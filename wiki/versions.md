# PostgreSQL Versions

This page indexes the PostgreSQL versions covered by the wiki.

## Supported Versions

| Version | Status | Wiki Home | Branch | Pinned Commit | Coverage |
|---|---|---|---|---|---|
| 19 | active | [v19/index](v19/index.md) | `REL_19_STABLE` | `67342a148632801d44c2fb9a7bf4231b6827c5d2` | Behavioral claims cite the matching pinned checkout under `raw/postgres-19/`; filed coverage includes a comprehensive walkthrough of the new `pg_plan_advice` contrib module (core `pgs_mask` strategy mask and planner hooks, per-index disabling, advice language including underscore-separated occurrence numbers, generation, enforcement, feedback/EXPLAIN, GUCs, tests) and its scoped source history (20 core planner foundation/enabling/fix commits, 27 direct module commits, and test/doc/build support commits), plus a comprehensive walkthrough of the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command (blocking new-heap rewrite/swap, lock-free multi-table candidate discovery followed by locked per-table open/recheck, partitioned-target discovery hardening, concurrent online rewrite via logical decoding with a decoding `bgworker`, the `pgrepack` output plugin, a temporary replication slot, race-safe on-demand decoding activation, change spill/replay, lock upgrade and swap, the `max_repack_replication_slots` GUC, `pg_stat_progress_repack`, generic `MAINTAIN` / `pg_maintain` documentation, and tests) with 55 feature-scope source-history commits; and a comprehensive, commit-backed walkthrough of three v19 autovacuum/VACUUM features (parallel autovacuum via the `autovacuum_max_parallel_workers` GUC and `autovacuum_parallel_workers` reloption, autovacuum table scoring via `AutoVacuumScores`/the five `*_score_weight` GUCs/`pg_stat_autovacuum_scores` plus the post-beta1 MXID-score division-by-zero fix `1f2297b5487`, and read-only query scans setting pages all-visible in the visibility map through on-access pruning, including the post-beta1 `e9eaeb04248` fix that records free space when a page becomes all-visible so its FSM entry does not go stale, plus the `b01c31eef9c` VM-clear WAL/incremental-backup correction, `9171f77db23` TAP test, and `3180ce3d7a8` wrong-buffer error-path hardening); all three v19 question histories are pinned to the `REL_19_STABLE` commit `67342a14`, stamped `19beta3`; the 2026-08-17 repin reviewed the 152 commits after `99e47536` and added five REPACK feature-scope commits, a 27th `pg_plan_advice` module commit, and the role-change plan-cache invalidation `567286b762b` (CVE-2026-14666). |
| 18 | primary | [v18/index](v18/index.md) | `REL_18_STABLE` | `baa7b142aace6821ce085906f314a75bcc4d95c8` | Behavioral claims cite the matching pinned checkout under `raw/postgres-18/`; filed coverage includes B-tree VACUUM density design, custom cumulative statistics, extension hooks for VACUUM/autovacuum, `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, `track_activity_query_size` activity text storage, `NUM_BUFFER_PARTITIONS` buffer mapping usage, GUC default-value changes since v12 through v18 (`effective_io_concurrency` and `log_connections` included), a walkthrough of bi-directional logical replication (mutual `origin = none` subscriptions, apply-worker origin tagging, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, limitations, and the new v18 origin-differs conflict logging) with a source-verified section on all logical replication features new since v17 (conflict detection/logging/statistics, generated-column replication, `streaming = parallel` default, alterable `two_phase`, `max_active_replication_origins`, `idle_replication_slot_timeout`, `pg_createsubscriber` and `pg_recvlogical` options, contrib `pg_logicalinspect`), and a fully reviewed Row-Level Security walkthrough covering rewrite-time command-layered security quals and WCOs, relcache/generated-catalog plumbing, policy recursion and permissions, WCO execution order, RI and whole-table boundaries, planner selectivity/cost/index/pruning/parallel effects, DDL locks, partitions/views/COPY/logical replication, direct and adjacent settings, and five PostgreSQL 14-derived performance follow-ups covering plan caching, MultiXact, aggregation, whether RLS filters must be repeated in the query, and scalar-InitPlan versus index-runtime-key behavior; exact-pin tests reproduce stale prepared results under the same effective-role OID after membership-edge `inherit_option`, `BYPASSRLS`, superuser, and current-database-owner changes until `DISCARD PLANS`; release-based history evidence covers 21 scoped RLS fixes/improvements not already present in PostgreSQL 14.0, with the 14.0-shipped pg_dump bulk-query improvement excluded, four previously omitted changes added, and the exact cross-branch first-release matrix omitted, and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation (four internal transactions, two heap scans, three waits, table/session `ShareUpdateExclusiveLock`, `indislive`/`indisready`/`indisvalid` progression, invalid-index failure outcomes, regression/isolation coverage, and v18 changes from v17: `pg_index` TOAST snapshot handling, parallel GIN build, virtual-generated-column rejection, and temporal `WITHOUT OVERLAPS` shared plumbing), and a declarative-partition-bound analysis answering whether the bound syntax or `pg_class.relpartbound` changed since partitioning was introduced (three syntax changes, all landed by v12; the column definition never changed; the stored `pg_node_tree` gained `is_default`/`modulus`/`remainder` in v11 and lost real parse locations in v17; the deparsed `pg_get_expr` text never changed; exposure surfaces only gained additions; the v18-only grammar commit `2d8bff603c9`; the reverted MERGE/SPLIT PARTITION; the write-once/clear-once lifecycle and three engine read paths; and measurements taken on the previous 18.3-line pin `6cb307251c5` of the stored node text, DDL-time bound evaluation, the quoted-sentinel asymmetry, and the no-TOAST size cliff). The 2026-08-17 repin from the 18.3 line `6cb307251c5` to 18.6 `baa7b142aac` reviewed all 294 commits in the range, which crosses the 18.4 and 18.6 releases; 18.5 was stamped (`17fae4fbdd6`) but never released, as the checkout's own release notes record. Claim-changing commits: `0b12f56bfac` (CVE-2026-14666) registers `PlanCacheRoleCallback` so role, role-attribute and database-ownership changes invalidate role-dependent saved plans; `2a29b607dbb` (CVE-2026-6471) adds the `output_plugin_libraries` GUC that every logical-replication publisher must list `pgoutput` in; `cb35d730689` (CVE-2026-6638) quotes the publisher-side origin-check identifiers and so answers the bidirectional page's open question; `8a31ffc2d4c` (CVE-2026-14676) rebuilds `pg_stat_statements` query normalization on an expansible `StringInfo` without adding a length cap; `2780538433f` (CVE-2026-6470) requires `USAGE` on types used by index, policy and partition-key expressions; `e4527519b77` propagates `INDEX_CREATE_DEFERRABLE` to the `REINDEX CONCURRENTLY` copy; `fe464e9e686`/`5cc59834b86` move the custom-statistics drop entry point to `pgstat_drop_entry_ext()`; and `585181e0774`/`7c25cdb1ebf` add the extension-visible `read_stream_clear_strategy()`. No GUC present in both v12 and 18 changed its default in the range, and measurements taken on the previous pin are labelled as such rather than re-run. |
| 17 | active | [v17/index](v17/index.md) | `REL_17_STABLE` | `786db8dcf168bd9df8f55047337525ac19118b1c` | Behavioral claims cite the matching pinned checkout under `raw/postgres-17/`; filed coverage includes the complete contrib extension inventory, explanations for all 53 control-file-backed contrib extensions, partial-index pros/cons with ordinary expression indexes as the baseline, partial expression indexes, planner predicate implication, expression-key matching, operational restrictions, `ANALYZE` row-count and expression-statistics behavior, planner costing, regression coverage, and a what-changed-from-v12 section covering the post-v12 documentation warning, concurrent-build safe-wait exclusion for partial/expression indexes, MERGE target rechecks, expression statistics, `NULLS NOT DISTINCT`, partial-unique proof restriction, and partial-hash planning fix, and a reviewed summary of the seven GUC default-value changes since v12 (old/new value, introducing major version, apply scope, exclusions, and test-coverage notes, grounded in `guc_tables.c`/`config.sgml` and the checkout's own commit history), and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation (four internal transactions, two heap scans, three waits, the `indislive`/`indisready`/`indisvalid` progression, a dedicated steps-and-locks section on `ShareUpdateExclusiveLock` + the `WaitForLockers`/`WaitForOlderSnapshots` waits, and a "what changed from v12" section centered on the PG14 `PROC_IN_SAFE_IC` snapshot-wait optimization, the reverted VACUUM-ignores-CIC attempt, and the `PGXACT`->`PGPROC` `statusFlags` move, anchored to the checkout's own commit history), two performance follow-ups covering `maintenance_work_mem` and the broader GUC matrix (build/parallel controls, AM and read-stream I/O boundaries, storage, WAL/checkpoint/commit costs, waits/timeouts, apply scopes, and commonly confused non-controls), including the separate AM-build and serial-validation budgets, B-tree/BRIN planned-versus-launched participants and mandatory worker tapes, hash/GiST/GIN/SP-GiST/BRIN/contrib-Bloom boundaries, conditional GIN pending-list cleanup, scoped observability, and the workload-specific plateau, a comprehensive walkthrough of the `REINDEX INDEX CONCURRENTLY` implementation with a source-backed GUC-performance follow-up covering build and validation memory, B-tree/BRIN worker request and availability, AM-specific GiST/GIN boundaries, v17 read streams, temporary and explicit output storage, WAL/checkpoint/commit costs across nine internal commits for one index, all five timeout-sensitive waits, exact apply scopes, and commonly confused non-controls, while the implementation covers (`ReindexRelationConcurrently`'s six phases and five waits, the atomic phase-4 `index_concurrently_swap` new-valid/old-invalid flip via the transactional `CatalogTupleUpdate`, the `_ccnew`/`_ccold` naming with a per-phase failure table showing a healthy `index_name` is never left invalid, a dedicated "can a failure leave an invalid index with the original index name?" section confirming a RIC failure never converts a bloated-but-valid `index_name` into no usable index (the phase-4 swap renames and flips validity in one transaction; only an already-invalid `index_name` stays invalid, the `concur_reindex_ind5` repair case), and a since-v12 section: partitioned REINDEX support via `ReindexPartitions` leaf recursion, `REINDEX (TABLESPACE ...)`, transactional `index_set_state_flags`, `PROC_IN_SAFE_IC` for RIC, the progress-view `WAIT_5` fix, the v15 shared-memory stats rework, and the v17 `MAINTAIN` privilege, anchored to the checkout's own commit history), and a fully reviewed analysis of whether `ALTER TABLE ... ATTACH PARTITION` can drop indexes (the built-in path cannot, but event triggers and `ProcessUtility_hook` can issue independent DDL; core coverage includes parser/utility dispatch, parent/child lock serialization with DROP, rollback, exact `CompareIndexInfo` matching—including reusable partial/expression indexes and, since 17.11 (`1d6c654c818`), element-wise exclusion-operator comparison instead of a blanket exclusion-index refusal—catalog/dependency re-parenting, physical and multilevel creation, generated/catalog/AM/contrib boundaries, tests and gaps, and meaningful changes since v12; exact-pin execution also exposed an open multilevel validity edge where recursive creation absorbs an invalid leaf while the pre-existing top parent stays valid), and a walkthrough of bi-directional logical replication (mutual `origin = none` subscriptions, apply-worker origin tagging into WAL, `pgoutput`/decode-time origin filtering, the `copy_data` initial-sync WARNING, setup pattern, and limitations) with its commit history grouped by minor version (16-cycle core commits `366283961a`/`8756930190`/`0324651573`, 17.0 `54ccfd6586`, 17.5 `0ae1245e04`, 17.10 CVE-2026-6638 `f0f59b658e`, plus foundations and adjacent origin-infrastructure commits), and a table-partitioning optimizations page covering v17 planning/execution optimizations per partitioning type (legacy inheritance: plan-time constraint exclusion only; declarative: plan-time + run-time pruning with per-HASH/LIST/RANGE bound matching, partitionwise join with merged non-identical bounds, partitionwise aggregate FULL/PARTIAL, and executor tuple routing/row movement/COPY batching, plus the four `PGC_USERSET` GUCs) with an "optimizations since v12" section attributing v13 advanced partitionwise join (`c8434d64ce0`), v14 run-time pruning for `UPDATE`/`DELETE` (`86dc90056df`) + `DETACH PARTITION CONCURRENTLY` (`71f4c8c6f74`), v15 `live_parts`/run-time-prune refactor/`MERGE` (`475dbd0b718`/`297daa9d435`/`7103ebb7aae`), v16 cached tuple routing (`3592e0ff98b`), and v17 `boolcol IS [NOT] UNKNOWN` pruning (`07c36c1333e`) + concurrent-detach robustness (`11f1218ce81`) to the checkout's own Stamp-bracketed commit history, plus a follow-up section on per-optimization locking reduction and random-I/O sensitivity (only plan-time pruning removes read locks; run-time pruning and constraint exclusion keep the locks `AcquireExecutorLocks`/`find_all_inheritors` already took; scan-eliminating optimizations gain most on slow random storage because avoided per-partition fetches are priced at `random_page_cost`; and, unlike v12, v17 partitionwise aggregate gains by avoiding a hash-agg spill whose writes `cost_agg` prices at `random_page_cost`, while hash-join spills stay `seq_page_cost`). Filed coverage also includes a planner-penalties-for-bloated-indexes walkthrough establishing that v17 penalizes a bloated index only through four inputs filled by `get_relation_info()` (`IndexOptInfo.pages` from a live `RelationGetNumberOfBlocks()` call for non-partial indexes or `estimate_rel_size()`'s `pg_class` density for partial ones, the fast-root `tree_height` charge of `DEFAULT_PAGE_CPU_MULTIPLIER` (50) `* cpu_operator_cost` per level, the `index_pages` share threaded into `index_pages_fetched()` and `compute_parallel_worker()`, and the v17-only clamp `num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333))`), that `pgstatindex`'s `avg_leaf_density` and `leaf_fragmentation` are read by nothing on the cost path, eight bloat shapes with a per-query-shape sensitivity matrix, and a since-v12 history in which `index_pages_fetched()`/`cost_index()` are byte-identical to `REL_12_0` while v13 deduplication, v14 bottom-up deletion/same-VACUUM recycling/index-vacuum bypass/failsafe/`reltuples = -1`, v16 macro extraction and partitioned-index zeroing, and the v17 SAOP clamp are attributed to their first release tags; exact-pin measurements reproduced the cost closed form to the cent, an identical `4.44` point lookup across a 9.62x size difference, a `1428.00` gap equal to exactly `357 * random_page_cost` between 49.73% and 0% `leaf_fragmentation`, two 2,745-block indexes at an identical `12730.42` with 9.27% versus 89.18% `avg_leaf_density`, a forged `pg_class.relpages = 1` that changed nothing for a non-partial index, a 25%-selectivity `Seq Scan` flip, an index dropped from a `BitmapAnd`, 4 versus 6 parallel workers, a SAOP descent plateau at 3 versus 19, 852 versus 2,749 blocks with `deduplicate_items = off`, and 583 versus 1,174 blocks of version-churn growth without and with a held `REPEATABLE READ` snapshot; a filed follow-up on that page answers when a GIN index is discarded and a B-tree used instead, establishing three gates (clause matching against the GIN opfamily with `btree_gin` as the escape hatch, plan shape from GIN's AM flags, then `gincostestimate()` cost plus `add_path()`/`choose_bitmap_and()` pruning), the 4X stale-metapage fallback and the `gin_clean_pending_list()` staleness trap, the keyless full-index path on a partial GIN index, the four `CREATE INDEX`/`CLUSTER` rejections, where GIN still wins, and the v17-specific fact that a bare boolean `Var` does match a `btree_gin` bool opclass; exact-pin measurements priced `n = 42` at `12.97` through GIN versus `4.52` through a B-tree on identical statistics with both closed forms reproduced to the cent, proved three no-GIN-path cases through `disable_cost`, moved a GIN scan from `13.80` to `4187.83` and out of a `BitmapAnd` with 982 pending pages, recovered the charged page count exactly in five states (including `985.00` and `1001.00` on a 10-block index), and measured a multicolumn `gin (tsv, cat)` beating the `BitmapAnd` at `21.55` versus `183.18`. Filed coverage also includes a test of the PostgreSQL 12 core-SQL B-tree bloat method on this pin: the v12 page's Methods A-D were executed verbatim on an isolated 17.11 server built from the pin and scored against `CREATE INDEX CONCURRENTLY` rebuilds plus `pgstatindex`, over the 15 named fixtures, a 9-bloat-type x 3-scale x partial/non-partial 54-cell matrix, and a six-point duplication-ratio sweep. Accuracy transfers unchanged for distinct-key indexes (catalog model exact on 11 of 15 fixtures and all 24 non-partial matrix cells outside the duplicate-key type; index-only-scan census `leaf_pages` exact on 12 of 12 fixtures and 36 of 36 eligible cells; `pg_column_size` moves the variable-width fixture from +105 blocks to -1; the stale-partial failure still clears with one `ANALYZE`, worst error 499 blocks to 1), while v13 deduplication breaks the page-fill model: +223.3% on an all-duplicate 1,000,000-row index, +92.5% at 5 rows per key with 0% reclaimable, +20.9% on a 25%-NULL index, and +217 density points (313.58% against 96.15%). A source-derived dedup-aware sweep (6 bytes per extra TID, gated on `n_distinct > 0`, `deduplicate_items`, non-unique, and the `pg_amproc` equal-image support function) cuts the worst duplicate-key error to -24 blocks (2.9%) with no regression on already-exact cells, and a `reltuples = -1` guard removes a 100.0% bloat reading on a healthy 21 MB index; the v17 `VACUUM VERBOSE` rewrite (four page classes, no index row count) and its 2%-of-pages index-vacuum bypass are documented as Method D regressions. A source-only follow-up then corrects two reporting defects that both of that page's statements shipped with, neither of which changes `expected_blocks` or any measured percentage: `fsm_bytes > 0 AS has_freed_pages` proves only that the index's free space map has been written at least once since its relfilenode was created (an index FSM records free-or-used only, the first recorded page extends the fork to a fixed 24576 bytes at `block_size` 8192, `GetFreeIndexPage` marks pages used without shortening it, no nbtree path reaches `FreeSpaceMapPrepareTruncateRel`, only `RelationSetNewRelfilenumber` resets it, and deleted-but-not-yet-recyclable pages, half-dead pages and partly-emptied leaves never enter the map), so it is renamed `fsm_written_since_build` and current page classes are sourced from `VACUUM VERBOSE`'s `newly deleted / currently deleted / reusable` line in core or from contrib `pgstatindex`, `pg_freespace` and `pageinspect`; and `wasted` was clamped at zero beside an unclamped `bloat_pct`, so a 3211-block index modelled at 3353 printed `0 bytes` next to -4.4% while the dedup-aware sweep hid 10,805,248 bytes of over-prediction next to -92.5%, both fixed by one signed convention across the two columns and the `ORDER BY` key; and a fifth source-only follow-up renaming the three reporting columns and both statement tags off the word "bloat" to `wasted_space`, `wasted_space_pct`, `wasted_space_pct_floor` and `wiki_btree_wasted_space_sweep_*`, established as an `AS`-label-only change (`TargetEntry.resname` carries `query_jumble_ignore` so `queryid` cannot move, both `ORDER BY` keys are expressions, and 22-byte names are far under `NAMEDATALEN` truncation) and justified from the docs' own vocabulary (the glossary's bloat is per-page space these statements cannot see; "wasted space" is what a maintenance command recovers; no core view, `pg_proc.dat` entry or contrib SQL script names a column `bloat`), with the consumer impact named - `ERRCODE_UNDEFINED_COLUMN` on an old name, `text`-typed `pg_size_pretty` output so ranking belongs on the byte expression via a suggested `wasted_space_bytes`, and an existing `pg_stat_statements` entry keeping the old tag because its text is written only at entry creation. Also filed: a measured answer to how to tell, after a `pg_upgrade` from v12 to 17, whether a B-tree index must be rebuilt to enable deduplication - deduplication safety lives only in the metapage field `btm_allequalimage`, which v12 never wrote and `pg_upgrade` never sets (`btm_version` is 4 on both sides, so version is not the discriminator), plus a core-SQL-only probe (byte 64 of block 0 through `pg_read_binary_file` + `get_byte`, no `pageinspect`), a catalog mirror of `_bt_allequalimage`, the exact `GRANT EXECUTE` required, a platform self-test, priority ordering and snapshot/twin fallbacks; measured on three isolated 12.2 -> 17.10 `pg_upgrade --copy` runs (21/21 carried-over indexes at `pd_lower = 64` with the flag false, 54/54 agreement with `bt_metap()`, all 21 index files MD5-identical across the upgrade, gate matching the engine's own `DEBUG1` verdict on 19 index shapes, rebuilds reclaiming 67.4% to 69.1% on eight duplicate-heavy indexes and 0.0% on five not-equal-image ones, a carried-over index equal to the byte to a fresh `deduplicate_items = off` twin, and a 0.0%/10.0%/48.1%/65.4%/69.5%/67.4% savings curve at 1/2/5/20/100/1000 rows per key), including the `deduplicate_items` trap, `VACUUM FULL`/`CLUSTER` versus plain `VACUUM`, the zeroed `relpages`/`reltuples` `pg_upgrade` leaves behind, and `pg_largeobject_loid_pn_index` as the only carried-over catalog index, and a re-measurement of the v12 COMMENT-stored bytes-per-table-tuple REINDEX threshold over the same 132 cells (22 workloads x 6 non-btree index shapes) with `REINDEX INDEX` as ground truth, in which only GiST moved: hash 9-11%, SP-GiST 29-32%, GIN `fastupdate = on` 56-100%, GIN `fastupdate = off` 40-62% and BRIN-never reproduce the v12 page's confusion matrices cell for cell while GiST's perfect window narrows from 14-33% to 25-33% because v17's default `GIST_SORTED_BUILD` path ignores fillfactor, plus the two v17-only hazards (`reltuples = -1` turning the original formula negative, and the v14 2% index-vacuum bypass deferring dead index entries), the 3 x 4 GiST fillfactor/`buffering` sweep, byte-identical parallel BRIN builds at 0/2/4 workers, a partial index reading +3.38% against 49.09% reclaimable, and a 17-row since-v12 change table. The 2026-08-17 repin from 17.10 `54eeefaedbee` to 17.11 `786db8dcf16` reviewed all 193 commits in the range: `1d6c654c818` made `CompareIndexInfo` compare exclusion operators instead of refusing every exclusion index, `f67b9dd8fc7` rewrote the ATTACH index documentation, `28269fed661` propagates `INDEX_CREATE_DEFERRABLE` to the `REINDEX CONCURRENTLY` `_ccnew` copy and adds the `reindex-conc-index-built` injection point, `d1c8aa0b09f` (CVE-2026-6470) requires `USAGE` on types used by index expressions and predicates, `31f2acde53d` stopped RANGE pruning from wrongly discarding the DEFAULT partition, `01992176e08` (CVE-2026-6471) added the `output_plugin_libraries` GUC, `14810cc0d96` rebuilt replication-command quoting, and four refint commits documented its PostgreSQL 20 removal and secure-schema requirements. No GUC default moved in 17.11, and measurements taken on the previous pin are labelled as such rather than re-run. A twelve-issue external review of the portable core-SQL B-tree wasted-space statement was then worked on a 17.11 server built from the current pin, the first measurements on that page taken on the pinned commit: four issues were already fixed, the "page packing" attribution is mischaracterized (48 of `i_q1000`'s 49-block gap is posting-tuple miscounting, 1 block is internal-fanout error above a deduplicated leaf level, and leaf packing is modelled correctly), five changes are applied — per-key-group tuple round-up, `pg_stats_ext` `ndistinct` for multicolumn keys, an invisible-statistics caveat, a randomly-inserted fixture, and a stated baseline — and `least(reltuples, n_live_tup)` plus `bt_metap().allequalimage` are rejected with source reasons. A seventh follow-up on the core-SQL B-tree wasted-space page files seventeen mandatory deduplication-gate tests and scores both proposed statements against them on a 17.11 ICU build of the pin: the corrected 12-through-17 statement fails 3 and the earlier v17 sweep 6, all because the gate tests for the existence of an equal-image support function while `_bt_allequalimage` calls it; change 6 (require `LANGUAGE internal` with `prosrc` in `btequalimage` / `btvarstrequalimage`) takes both to 0 over-credits over 28 fixtures at one deliberate under-credit, with the `prosrc`-versus-`proname` choice, the post-build-mutation/`bt_index_check` case, a stock-database census, a zero-fixture audit query and the runnable fixture harness all measured. An eighth follow-up adds a maintained "current recommended statement" section directly after that page's verdict, naming one text to run, ranking all six variants on accuracy, fix count and measured version coverage, stating the residual errors and what the recommendation does not replace, and fixing the rule for displacing it. A ninth follow-up folds change 6 into that statement — the section becomes "with all six changes", change 6 keeps its rationale but loses its duplicate SQL, and no assembly step remains — and re-measures every server-measured table on the page on 17.11 across four servers (an ICU-and-debug 17.11 install with four fixture databases, a second 17.11 cluster, and isolated 14.23 and 12.2 servers), with every scored text generated mechanically from the page's own Markdown; it reproduced the fixture, matrix, Method B, density and gate scoreboards, moved only `ANALYZE`-sampled cells (Method A exact on 11 of 15 fixtures rather than 9, the matrix at 33 rather than 30, `i_q1000` at 899 blocks on 8 stored most-common values), corrected the claim that change 6 under-credits an internal alias of `btequalimage` (it does not: both texts read 69.3% on a true 69.4%, so the cost is confined to non-internal support functions), and newly measured the nondeterministic-collation conjunct on ICU, change 1's −24.8% / −33.1% residuals on a two-column and a partial fixture, and a never-rebuilt 1.5M-row random twin at 25.9% beside its rebuilt sibling's 11.5%. Filed coverage also includes a pgstatindex-only B-tree bloat report that runs unchanged on 12 and 17: one statement whose only measurement function is `pgstatindex`, reporting wasted space (free leaf bytes plus every empty and deleted page, against perfect packing) beside estimated reclaimable space (against a rebuild at the index's own fillfactor, taken from `BLCKSZ * (100 - fillfactor) / 100` in `_bt_pagestate` rather than from the fillfactor itself), with every candidate filter justified by a shape `pgstatindex` raises on -- non-B-tree access methods, partitioned indexes, another session's temp index, and the majors' one behavioural difference, an invalid index that 17.11 refuses (`13503eb5905`, earliest containing tag `REL_17_0`) and 12.2 reports. Scored against `REINDEX INDEX` on isolated 12.2 and 17.11 servers over 24 index shapes: 91 of 94 and 89 of 93 indexes within 1.0 point, the same `+1.7` worst over-estimate on both, 24 report rows byte-identical across the two servers, and three named under-estimates (deleted-not-vacuumed `-90.0`, a rebuild that would deduplicate `-69.4` on v13+, 400-byte keys `-5.1`); plus the `NaN` trap for indexes with no leaf pages, the `pg_stat_scan_tables` grant and why the OID form needs no schema `USAGE`, ~830 MB read per run through a 256 kB bulk-read ring, a `lock_timeout` cancellation at 2000.9 ms, and the concurrent-drop race that aborts the whole statement. |
| 14 | active | [v14/index](v14/index.md) | `REL_14_STABLE` | `a92fbdfb830046e907813e9067b2c9de9708d600` | Behavioral claims cite the matching pinned checkout under `raw/postgres-14/`; source version 14.24, pinned to the `REL_14_STABLE` tip `a92fbdfb830` (`REL_14_24-6-ga92fbdfb830`, six commits past the 14.24 release stamp; repinned from 14.23 `5c00f4e2e3b` on 2026-08-17). Filed coverage so far is the mandatory codebase navigation guide: source-tree layout from the top-level and backend makefiles, the normal SQL statement path through `tcop`/rewrite/planner/executor (`exec_simple_query` -> `pg_parse_query` -> `pg_analyze_and_rewrite`/`QueryRewrite` -> `pg_plan_queries`/`planner` -> `PortalRun`/executor entry points), utility-command dispatch via `ProcessUtility`, generated catalog/header artifacts (backend `generated-headers` target, the backend catalog `generated-header-symlinks`/`genbki.pl` path, and the `pg_index.h` catalog header), key parser/planner/executor/relcache/AM data structures, and contrib and test/doc surfaces. Filed question coverage adds a Row-Level Security walkthrough: RLS as a rewrite-time feature (`pg_policy` storage, `relrowsecurity`/`relforcerowsecurity`, relcache `rd_rsdesc`, `check_enable_rls` bypass logic, `get_row_security_policies` security-barrier `USING` quals and `WITH CHECK` options, permissive-OR/restrictive-AND with default-deny, the planner `security_level`/leakproof ordering, executor `ExecWithCheckOptions`, plan-cache `dependsOnRLS` re-planning, partition/inheritance and view-owner handling, generated catalog/header implications, and COPY/CTAS/whole-table/RI/logical-replication/error-detail boundaries), its source-evident scalability/performance issues (including a dedicated RLS-and-the-plan-cache section: RLS-aware plan caching keyed on role + `row_security`, where 14.24's `PlanCacheRoleCallback` (CVE-2026-14666, `f4174aa84a3`) now invalidates role-dependent saved plans on `pg_auth_members`/`pg_authid`/`pg_database` changes, closing the former same-role membership, `INHERIT`, `BYPASSRLS`, `SUPERUSER`, intermediate-role, and database-owner staleness window for saved plans, with `DISCARD PLANS` retained as a blunt manual reset, the role-independent relcache `rd_rsdesc` policy cache, the scenarios where caching does not help — role/`row_security` mismatch at reuse, the never-applicable simple-validity fast path, un-cached simple-protocol and one-shot SPI queries, the conditional first-five custom-plan opportunities, DDL invalidation — and the no-cache re-analyze/rewrite/re-plan cost; a query-vs-policy planning subsection showing RLS policy conditions need not be restated in the query — `USING` quals are transferred into `baserestrictinfo` at ordered security levels (the first sublist is level 0; restrictive quals precede the permissive OR in mixed sets), safely promotable clauses can match indexes, partition pruning sees the full restriction list, and query-text copies land at the maximum security level; a sub-SELECT-wrapping subsection showing that wrapping an uncorrelated function call in a scalar `(SELECT ...)` (e.g. `(SELECT auth.uid())`) can make each generated InitPlan occurrence lazy, at most once per execution, and cached in a `PARAM_EXEC` parameter, avoiding repeated function-expression work when the unwrapped expression would remain a per-tuple filter, with caveats (only uncorrelated sub-SELECTs qualify, exact index runtime keys can avoid per-row calls but lossy paths recheck, and rewrite copies and separate occurrences are not generally memoized together); an RLS-and-aggregation mitigation section showing that `Agg` nodes consume RLS-filtered subplans and mitigations focus on indexable policy predicates, leakproof/indexable filters, partition pruning/partitionwise aggregation (`enable_partitionwise_aggregate` is session-scoped), stable plan-cache environments, or deliberately trusted/pre-aggregated reporting paths; plus the FK-validation/two MultiXact paths (policy-authored `FOR SHARE` locking subqueries and RLS-induced FK validation fallback) where RLS disables the bulk RI path and per-row `FOR KEY SHARE` checks can create/read MultiXacts on hot referenced rows), and the RLS control/inspection surface (`row_security`, general `plan_cache_mode` and aggregation-adjacent `enable_partitionwise_aggregate` scopes, table flags, `BYPASSRLS`, policy options, extension hooks, `row_security_active`, catalog/psql views, and pg_dump/pg_restore controls), with explicit core/hook test coverage. Filed question coverage also adds a MultiXact walkthrough: how MultiXact backs shared row locks (a `MultiXactId` in `xmax` for a set of `(xid, MultiXactStatus)` members, the offsets/members SLRUs under `pg_multixact/`, `MultiXactStateData` + per-backend oldest arrays, the `MultiXactIdCreate`/`Expand`/`CreateFromMembers` -> `GetNewMultiXactId`/`RecordNewMultiXact` write path, and the `GetMultiXactIdMembers` read path), the backend-local 256-entry per-transaction LRU cache whose tail eviction only frees memory rather than spilling an entry (`mXactCacheEnt`/`MAX_CACHE_ENTRIES`/`MXactContext`/`mXactCachePut`), and the separate miss path through the shared 8-offset/16-member SLRU pools to a possible filesystem `pg_pread`/`SLRURead` that may still hit the OS page cache; the distinction between system-wide MXID/member allocation and one backend's cache pressure; foreign keys (RI `FOR KEY SHARE` -> `heap_lock_tuple` -> `compute_new_xmax_infomask`, including null/partition/committed-updater edges), compatible explicit locks, UPDATE/DELETE/update chains, executor/trigger/logical-apply callers, and exact VACUUM replacement branches; three degradation paths (hot-row/full-member-array creation work, exclusive SLRU control/per-buffer/file-I/O waits after cache misses, and ID/member growth driving aggressive vacuum plus separate hard stops); observability (`pg_stat_slru`, exact generated wait names, `mxid_age`, targeted member decoding) with attribution limits; the four vacuum GUC scopes (three `PGC_USERSET` settings and restart-only `autovacuum_multixact_freeze_max_age`), per-table autovacuum MultiXact reloptions, build/generated/WAL/tool/contrib boundaries, verified diagnostic SQL, the seven MultiXact isolation specs passing on the previous 14.23 pin, and explicit cache/SLRU performance-test absence. Filed question coverage also adds a fully source-reviewed functions/procedures-in-a-WHERE-clause performance walkthrough: procedures (`prokind='p'`) are rejected in expressions and run through utility `CALL`, whose arguments still use standalone expression evaluation; set-returning, aggregate, and window functions are also rejected directly in `WHERE`; scalar residual quals scale with candidate scan tuples or join pairs, subject to short-circuiting, `STRICT` null skips, SQL inlining, constant folding, pseudoconstant gating and rescans, and index runtime-key placement; the volatility effects (`VOLATILE` per row where needed, `STABLE` single-eval as an index-scan runtime key or no-`Var` one-time qual, `IMMUTABLE` constant-folded with constant args), the cost model (`add_function_cost` = `procost * cpu_operator_cost`, with omitted `CREATE FUNCTION COST` defaults of 1 for C/internal functions and 100 otherwise) and bare-Boolean-function fallback selectivity (`function_selectivity` 0.3333333), the narrower no-statistics `boolvarsel` 0.5 fallback, and expression-index/extended-expression statistics, plain-index search-key limits, expression-index/generated-column matching, post-planning immutability checks, non-volatile runtime keys, and support-generated index conditions, the separate leakproof security-promotion/index-pushdown rule and same-node `< 10 * cpu_operator_cost` ordering cutoff, plus `PARALLEL UNSAFE`/`RESTRICTED` plan effects; plus the mitigations (reduce candidate tuples/pairs, expression indexes/generated columns or estimate-only expression statistics, uncorrelated scalar sub-SELECT -> lazy once-per-execution InitPlan, truthful volatility/`STRICT`/`COST`, SQL-function inlining, planner `SUPPORT`, truthful `LEAKPROOF`/`PARALLEL SAFE`, and measured JIT/function tracking), with `ROWS` explicitly excluded for direct WHERE functions; generated `pg_proc_d.h`, fmgr/contrib-`sepgsql`, extension-bitcode, GUC-scope, and regression/absence boundaries are included; the RLS page is agent-verified by `GPT-5` at `2026-07-20T18:13:13Z`, the functions/procedures page at `2026-07-13T19:41:59Z`, and the MultiXact page at `2026-07-13T20:08:07Z`; only the codebase navigation guide remains `not yet`. The 2026-08-17 repin from 14.23 `5c00f4e2e3b` to 14.24 `a92fbdfb830` reviewed all 133 commits in the range. Claim-changing commits: `f4174aa84a3` (CVE-2026-14666) registers `PlanCacheRoleCallback` on `pg_auth_members`/`pg_authid`/`pg_database`, which closes the former same-role RLS plan-cache staleness window for saved plans; `1a358b8f2a2` (CVE-2026-6470) requires `USAGE` on types used by policy, index, default, CHECK and partition-key expressions; `155dacbc547` (CVE-2026-14680) rejects calls to functions taking or returning type `internal`; `802dc79df63` removed replication-slot advice from the MultiXact wraparound hints and documented that slots do not hold back multixact cleanup; and `2bb60eb4fea` moved `RecordNewMultiXact()`'s SLRU lock acquisition later to fix a replay self-deadlock. Nothing was re-measured on 14.24, so previous-pin measurements are labelled as such. |
| 12 | legacy | [v12/index](v12/index.md) | `REL_12_STABLE` | `45b88269a353ad93744772791feb6d01bc7e1e42` | Behavioral claims cite the matching pinned checkout under `raw/postgres-12/`; the reviewed mandatory codebase navigation guide covers build ownership, backend/standalone entry, simple and extended protocol including Sync, parse/analyze/RIR rewrite/planner/portal/full executor flow, utility and error boundaries, generated catalog/parser/fmgr/LWLock artifacts, core hooks, packaged extensions, custom AM registration/dispatch/tests, key data structures, and test/doc surfaces; filed coverage includes two `CREATE INDEX CONCURRENTLY` performance follow-ups: a `maintenance_work_mem` analysis (AM-dependent first-build and common validation-sort uses, B-tree planned-versus-launched participant shares and required tape runs, conditional GIN cleanup, BRIN's empty enumeration, GiST's buffered temporary-file path, scoped observability, and the workload-specific plateau) plus a broader CIC GUC matrix (B-tree worker request and pool limits, AM-specific settings, v12 heap-scan and storage boundaries, WAL/checkpoint/commit costs, waits/timeouts, exact apply scopes, absent later-version controls, and commonly confused non-controls), foreign-key join selectivity, partial-index pros/cons with ordinary expression indexes as the baseline, partial expression indexes, planner predicate implication, expression-key matching, operational restrictions, `ANALYZE` row-count and expression-statistics behavior, planner costing, and regression coverage, a fully reviewed planner-statistics-source analysis (`pg_stat_all_tables`, `pg_stats`, and `pg_stats_ext` excluded as direct feedback; `pg_class` plus physical size, `pg_statistic`, built extended data, ordinary/partial index sizing, inherited/expression statistics including partial-index correlation, defaults/security/errors, indirect auto-analyze and planner endpoint counter increments, estimate/cost metadata, AM/FDW/hook/contrib/generated-cache boundaries, and direct tests included), `EXPLAIN (ANALYZE, BUFFERS)` output, `pg_stat_statements` mechanics and configuration, a production database-health checklist covering observability prerequisites and apply scopes, activity/connections/locks, cumulative database and table counters, vacuum and wraparound, checkpoints, WAL archiving, replication, checksums, query and relation diagnostics, and database-log issue interpretation, `psql` environment variables and session timeout behavior, reviewed full non-snapshot `pgstatindex` behavior (extension install/API and privileges, true-root metadata, one-time main-fork scan, half-dead/deleted/live classification, size/density/fragmentation formulas and two-decimal formatting, physical `LP_DEAD` occupancy, concurrency and integrity-check limits, generated-header boundary, and test gaps), planner penalties for bloated B-tree indexes through physical pages and tree height including the storage-manager path and I/O/CPU boundary for `RelationGetNumberOfBlocks()`, a proposed `avg_leaf_density` / `leaf_fragmentation` triage heuristic for REINDEX candidates, B-tree leaf density (60% vs 90%) impact on planner page-count costs and executor leaf-page walking, now fully reviewed at the pin with exact-pin measurements (2733 versus 4116 leaf pages, 2738 versus 4121 warm index-only scan buffers, a 5552.01 cost gap equal to 1388 extra blocks at `random_page_cost = 4`, an unchanged probe cost on narrow keys against a wide-key fixture where 90-to-60 raised `tree_level` from 2 to 3, and a random-insert index that reached 65.58% density with no deletions), a density-versus-fragmentation comparison for index scan I/O with level estimates and cache/storage sensitivity, and a three-stage REINDEX triage heuristic that shortlists candidates from `pg_class`/`pg_stat_*`/`pg_relation_size()` signals, confirms density with gated `pgstatindex` runs, prioritizes by configured-fillfactor reclaimable pages times paired-window scan rate with `REINDEX (CONCURRENTLY)` execution notes, measures post-REINDEX improvement via size/shape deltas, per-scan counter rates (per-index plain REINDEX keeps its OID while concurrent REINDEX copies the published collector entry, with pending-count continuity treated as an open boundary, including the v12 `index_concurrently_swap` stats copy), and `EXPLAIN (ANALYZE, BUFFERS)` / `pg_stat_statements` before/after windows, and documents the cited rationale for excluding `leaf_fragmentation` from the REINDEX priority score, and a comprehensive walkthrough of the `CREATE INDEX CONCURRENTLY` implementation in `DefineIndex` (four internal transactions, two heap scans, three deliberate transaction-set waits, the `indislive`/`indisready`/`indisvalid` progression set non-transactionally by `index_set_state_flags`; the reviewed CIC page now also covers parser/utility/generated-catalog artifacts, the core-vs-index-AM/contrib-Bloom boundary, four deliberate transaction barriers versus additional event-trigger/predicate/AM/unique-validation/parallel-worker waits, temp and `IF NOT EXISTS` early-exit ordering, a post-`SET_VALID` `ddl_command_end` failure that leaves a valid index, and crash durability scoped to reviewed in-tree AMs rather than arbitrary third-party callbacks; it also retains the preconditions/restrictions and full failure-state coverage, and a dedicated steps-and-locks section covering transaction- and session-level `ShareUpdateExclusiveLock`, the `WaitForLockers(ShareLock)` writer waits, `WaitForOlderSnapshots`, and the `LockConflicts` rationale for why DML proceeds while another CIC/`VACUUM`/`ANALYZE`/DDL is blocked, plus a reviewed blocker-by-blocker enumeration of the four deliberate core transaction barriers separately from additional event-trigger/predicate/AM/unique-validation/parallel-worker waits; the four barriers cover conflicting-lock holders at acquisition with the autovacuum-cancel and prepared-transaction nuances, open writers at the two lock waits, and same-database old-snapshot holders at the snapshot wait, with the `pg_stat_progress_create_index` wait phases and worked examples for a running `pg_dump` and an hour-long open / idle-in-transaction session, plus a failure-scenarios section mapping each CIC phase to the leftover on the table — no index, an invalid not-ready index, an invalid ready index that still takes writes and enforces uniqueness, or a valid index after a post-`SET_VALID` command-end error — with each leftover's planner/write/uniqueness/HOT cost, session-lock release on ordinary ERROR/cancel paths, a resolved crash/immediate-shutdown recovery outcome (xidless `heap_inplace_update` flag flips commit asynchronously and are redone by `heap_xlog_inplace`, so recovery lands on one of the four documented leftovers without losing completed in-tree build data while retaining a later durable `SET_VALID`; this crash-durability result does not repair the separate prepared-transaction gap or prove arbitrary third-party AM callbacks), `DROP`/`REINDEX` recovery), a companion comprehensive walkthrough of the v12 `REINDEX INDEX CONCURRENTLY` implementation with a source-backed GUC-performance follow-up covering AM-dependent build/validation memory, B-tree-only worker controls, v12 bulk-read/synchronized-scan boundaries, temporary and fixed output storage, B-tree immediate sync plus WAL/checkpoint/commit costs, all five timeout-sensitive waits, exact apply scopes, absent later-version controls, and commonly confused non-controls in `ReindexRelationConcurrently` (its six phases — create the `_ccnew` copy, build, validate, swap, set-dead, drop — and five wait points, namely the three shared CIC waits plus two extra `AccessExclusiveLock` "wait for readers" waits before marking the old index dead and before dropping it; the atomic `index_concurrently_swap` that marks the new index valid and the old invalid, renames `_ccnew`→original and original→`_ccold`, and moves constraints/triggers/comment/dependencies and per-index cumulative stats; `ShareUpdateExclusiveLock` held at transaction and session level on the old index, new index, and heap; the no-transaction-block/temp-fallback/system-catalog/partitioned/exclusion/invalid restrictions and toast rebuild; the two-index state-flag progression; a per-phase failure table leaving an invalid `_ccnew` before the swap or `_ccold` after, with regression evidence; multi-index batching; and the source-vs-view progress-phase discrepancy where phase 6 reports `WAIT_4` so `waiting for readers before dropping` is never emitted), and an analysis of whether `ALTER TABLE ... ATTACH PARTITION` can drop indexes from the attached table (it cannot: `AttachPartitionEnsureIndexes` is strictly match-or-create via `IndexSetParentIndex`/`DefineIndex`, incompatible/extra indexes are kept, no `performDeletion`/`RemoveInheritance` runs in the attach path and the pre-index `CreateInheritance` merges CHECK constraints only; v12's looser matching — no `indisvalid` skip, existence-only constraint check — still never drops; covering the three look-alikes, the `ALTER INDEX ... ATTACH PARTITION` re-parent/error path, and the docs' plain-`CREATE INDEX` pre-build workflow), and a comprehensive enumeration of all outcomes that leave an invalid index (`indisvalid=false`) derived from the full set of `indisvalid=false` writers in the checkout: a failed/cancelled/crashed `CREATE INDEX CONCURRENTLY`, a failed `REINDEX CONCURRENTLY` (`_ccnew`/`_ccold`), a failed/interrupted `DROP INDEX CONCURRENTLY` (retryable clear-valid step), an incomplete partitioned parent (`CREATE INDEX ON ONLY` / attached invalid child, validated by `validatePartitionedIndex` on `ATTACH PARTITION`), and `pg_upgrade` from <= 9.6 invalidating hash indexes in the new cluster, with the persistent state table, the single-transaction-DDL contrast, the planner/executor/VACUUM cost and the commands that reject invalid indexes, and per-outcome repair, and the partitioning optimizations and configuration during query planning and execution (partition pruning plan-time before child expansion in `expand_partitioned_rtentry` and run-time initial + per-scan pruning on `Append`/`MergeAppend`, with per-strategy HASH/LIST/RANGE bound-matching and tuple routing; constraint exclusion `relation_excluded_by_constraints` modes for legacy inheritance; partitionwise join `build_joinrel_partition_info`/`have_partkey_equi_join` and partitionwise aggregate `group_by_has_partkey`; executor tuple routing, cross-partition UPDATE row movement, and per-partition COPY multi-insert; the four session-scoped GUCs with defaults; and the v12 limitation that execution-time pruning covers only `Append`/`MergeAppend`, not `ModifyTable`; plus a follow-up locking/random-I/O analysis per optimization: only plan-time pruning removes read locks (it prunes before `expand_partitioned_rtentry` locks partitions, while `find_all_inheritors` locks all inheritance children before constraint exclusion runs and `AcquireExecutorLocks` locks every plan-time-surviving partition despite run-time pruning; write-path routing takes `RowExclusiveLock` lazily per touched partition), and the scan-eliminating optimizations gain most on slow random storage because avoided per-partition index heap fetches are priced at `random_page_cost` = 4× `seq_page_cost`, whereas partitionwise join/aggregate cut CPU/memory/locality (v12 costs hash-join spill as sequential and keeps hash aggregation in memory)). Filed coverage also includes a fully reviewed `plan_cache_mode` analysis: the enum/`PGC_USERSET`/`GUC_EXPLAIN` definition and its session/transaction apply scope, the eight ordered rules in `choose_custom_plan` with `force_generic_plan`/`force_custom_plan` sitting behind the one-shot, no-parameter, and transaction-statement rules, the `auto` cost comparison and its custom-only `1000 * cpu_operator_cost * (nrelations + 1)` planning charge, the `GetCachedPlan` correction step, `PARAM_FLAG_CONST` folding and the `var_eq_const`/`var_eq_non_const` selectivity split, the `CachedPlanSource` heuristic fields and every caller that reaches the decision, cost-history retention across invalidation and `DISCARD PLANS`, the per-execution `AcquireExecutorLocks` footprint of a generic plan, v12 observability limits and error paths, build/extension boundaries, the single direct regression test with explicit gaps, and five ancestor commits from `e6faf910d75` to `ca0b3828504`; fourteen exact-pin measurements reproduce the execution-6 switch, the cached-but-unexecuted generic plan, the ignored `force_*` overrides, PL/pgSQL static versus dynamic behavior, `pgbench -M prepared` versus `-M extended`, 2 versus 5 partition locks, and 0.65 ms versus 0.006 ms of planning per execution. B-tree coverage now also includes a core-SQL-only bloat measurement proposal that models the size a rebuild would produce from v12's own `_bt_buildadd` fillfactor rule and subtracts it from `pg_relation_size()`, ranked across four no-contrib methods (catalog sweep, `pg_column_size` slot measurement, index-only-scan `BUFFERS` census, `CREATE INDEX CONCURRENTLY` rebuild probe) with exact-pin accuracy against real rebuilds on 14 fixtures, plus a filed follow-up that scores those methods against `pgstatindex` over a 54-index bloat-type/scale/partial matrix and shows an `avg_leaf_density` rebuild-size predictor losing in 16 of 18 cells. Filed `storage-and-vacuum` coverage begins with a proposed and tested `fillfactor`-corrected `pgstattuple_approx` heap-bloat metric: the `statapprox_heap()` visibility-map skip and free-space-map estimate, why a skipped page's FSM entry is a VACUUM-written value in v12, the reloption-to-`saveFreeSpace` path shared by `RelationGetBufferForTuple()`, `heap_multi_insert()`, `raw_heap_insert()`, and `heap_page_prune_opt()`, a seven-point fillfactor sweep in which the uncorrected signal tracks `100 - fillfactor` (0.44 to 91.02 on unbloated tables) while the corrected one holds 0.02 to 1.79 except 10.16 at `fillfactor = 10`, a closed-form residual model matching all seven to within 0.05 points, the reloption-changed-after-load case where `VACUUM FULL` grows the table 44.93% and only the unclamped metric reports it, exact-pin cost (6.9 ms and 42 physical reads against 484.8 ms and 137,932 reads on a 1.08 GB table), accuracy of −3.11 to +1.79 points against `VACUUM FULL` over eight ordinary fixtures, the unusable-page-tail false positive the correction cannot reach, the dropped-column and retained-line-pointer false negatives, the refused TOAST relation, the `reltuples` dependency and integer-truncated `scanned_percent`, uncommitted-insert and old-snapshot behavior, the `pg_stat_scan_tables` privilege boundary, and a catalog-only estimator that is exact on nine well-behaved fixtures and wrong by 21 to 5,699 points on adversarial ones. Filed `indexing` coverage also includes a calibration of the COMMENT-stored bytes-per-table-tuple rebuild trigger for all five core non-btree access methods: `pg_relation_size(index) / table pg_class.reltuples` stamped into each index comment at build time, scored over 132 exact-pin cells (22 insert/update/delete workloads x 6 index shapes) against `REINDEX INDEX` ground truth, finding GiST the only access method that calibrates cleanly (any threshold in 20-30%, 14/0/0/8, with only -0.29% to +3.84% fresh-build drift across 25k-1.6M rows), hash (9-11%) and SP-GiST (29-32%) usable only at a stable row count because a fresh build alone spans -20.31% to +49.42% and up to +33.08%, GIN needing 50-75%, and BRIN unusable at any threshold (0.00% reclaimed in 22/22 cells against readings from -87.50% to +9900.00%); with the delete-only `1/(1-f)-1` identity, the GIN pending-list trap where a flush moves the reading from +261.73% to +388.78% while delivering the whole 3,779 -> 1,336 block win, hash's 10.3% baseline spread from `estimate_rel_size`, its splitpoint staircase and 87 filesystem-unallocated pages counted by `pg_relation_size`, BRIN's 6.3x `pages_per_range` baseline spread and precision-only degradation, the stale-`reltuples` +84.71%/-7.64% inversion, a partial-index false negative (-5.02% reading, 44.59% reclaimable), per-access-method rebuild costs and query-block payoffs (none for hash or BRIN), and `COMMENT ON INDEX` mechanics verified on the pin. Filed coverage also includes a no-regex extraction of declarative range partition bounds: v12 exposes no typed bound, so the deliverable decomposes the `pg_get_expr(relpartbound, oid)` text that `psql \d` and `pg_dump` consume, using the deparser's fixed grammar from `get_rule_expr`/`get_range_partbound_string`/`get_const_expr` (`FOR VALUES FROM (` … `) TO (`, `, ` separators, bare `MINVALUE`/`MAXVALUE`, `DEFAULT`, no `::type` or `COLLATE` at `showtype = -1`, `simple_quote_literal` quoting that follows `standard_conforming_strings`) plus a quote-masking pass over `string_to_array(bound_def, '''')` so `strpos`/`substr` see only real separators; with a gated `split_part` variant, the qualifying-relation rules (`relispartition` + `partstrat = 'r'`, `relkind` r/p/f, multi-level, cross-schema, DEFAULT rows, detached and index partitions excluded), cast-back semantics and the `boolean`/`char(3)` exceptions, rejected alternatives (`relpartbound::text` byte dump, `pg_get_partition_constraintdef`, `pg_partition_tree`), and exact-pin measurements (41/41 reconstructed deparse match, 95/95 recipe versus 9/41 partitions wrong for the obvious split, 45/45 for `pg_get_expr(…, 0)` and `pretty = true`, 78/81 exact round trips, six output GUCs swept, 14 catalog locks and zero partition locks at 1048 partitions in 39.9-44.9 ms, full visibility to an unprivileged role, the NULL-deparse drop race, and the untoastable-`pg_class` `row is too big` ceiling). The core-SQL B-tree bloat page's statements now report `wasted_space` and `wasted_space_pct` under the tag `wiki_btree_wasted_space_sweep`, with the Method C probe index renamed `wiki_wasted_space_probe`, filed as a source-only correction that moves no measured number: v12 defines a bloated index as one with "many empty or nearly-empty pages" (`ref/reindex.sgml`) — per-page state that file-size-minus-model arithmetic cannot see — while `ref/copy.sgml` supplies "wasted space" for what a maintenance command recovers, no v12 SQL interface names a column `bloat`, and this checkout has no glossary; `AS` labels cannot reach the expression `ORDER BY` key or the contrib-computed query ID (`resname` is not jumbled and the lexer drops the comment), whereas the probe rename is an object name that does re-key the two utility statements, which v12 hashes by text. |

## Coverage Notes

- 2026-08-20: filed [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12
  and 17 (unverified)](v17/questions/indexing/btree-bloat-with-pgstatindex.md) against
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). **One statement, one
  measurement function.** `pgstatindex` supplies every number; `pg_class`, `pg_index`,
  `pg_am`, `pg_namespace` and a `pg_relation_size` cost prefilter only choose and label
  indexes. It reports **two different percentages on purpose**: `wasted_pct`, free bytes in
  live leaf pages plus every empty and deleted page measured against perfect packing, and
  `est_reclaimable_pct`, the same file measured against a rebuild at its own fillfactor.
  The rebuild target is taken from the build code rather than from the reloption — a sorted
  build closes a leaf when free space falls below `BLCKSZ * (100 - fillfactor) / 100`, so at
  8192/90 the reachable density is 89.95%, not 90% — which makes the estimate conservative
  by construction. The two readings disagree loudly and correctly: a fresh `fillfactor = 10`
  index reads **89.6% wasted and −0.5% reclaimable**, while an index whose leaves are a
  textbook 89.94% dense is **69.9% reclaimable** because 1,918 of its pages are empty or
  deleted. **Every candidate filter is there because `pgstatindex` raises on that shape and
  one raised call aborts the whole report**: the five non-B-tree access methods and
  partitioned indexes (`relation "..." is not a btree index`), another session's temp index
  (a 4.4 MB one moved the candidate count 27 -> 28 on 17.11), a stale OID (`could not open
  relation with OID ...`), and **the majors' single behavioural difference — an invalid
  index, which 17.11 refuses with `index "..." is not valid` (commit `13503eb5905`, earliest
  containing tag `REL_17_0` in this checkout) while 12.2 returns a row for it**. **Scored
  against `REINDEX INDEX` on newly initialised isolated 12.2 and 17.11 servers** over one
  shared 24-shape fixture script plus one 17-only deduplication fixture: **91 of 94 and 89
  of 93 scored indexes within 1.0 point**, 92 and 90 within 2.0, the identical `+1.7` worst
  over-estimate on both (a 456 kB TOAST primary key; every other row over-estimates by at
  most `+0.1`), and **24 of the report's rows byte-identical across the two servers** — the
  three that differ are the two duplicate-key indexes (21 MB against 6800 kB, 20 MB against
  6368 kB: deduplication changes the input, not the arithmetic) and a non-deterministic
  churn fixture. Three under-estimates are documented rather than hidden: entries deleted
  but not vacuumed (`−0.1%` reported against 89.9% actual, on both servers), an index with
  `deduplicate_items = off` whose rebuild would deduplicate (`−0.3%` against 69.1%, v13+
  only), and 400-byte keys (`69.9%` against 75.0%, because wide tuples close a page fuller
  than the model's bound). Also measured on both: `index_size` equal to `pg_relation_size`
  for **218 of 218 and 212 of 212** candidates; `NaN` density on an index with no leaf pages,
  and `NaN > 20` true for both `float8` and `numeric`, with all 62 and 59 `NaN` rows scoring
  exactly 0.0 through the statement's guard; leaf capacity implied at **8151.6 and 8152.1**
  from two known-content pages against the hard-coded `8192 - 24 - 16`; a `pg_stat_scan_tables`
  member reading every row while `pgstatindex('bl.i_del90')` by name fails for the same role
  with `permission denied for schema bl`, which is why the statement passes an OID; ~830 MB
  of index pages read per run through a 256 kB `BAS_BULKREAD` ring (106,351 and 105,413
  buffers at the function scan, 124.2 ms and 132.6 ms execution); a `lock_timeout`
  cancellation at 2000.9 ms behind an uncommitted `DROP INDEX`; and a concurrent drop three
  seconds into a run killing the whole statement. Eight open questions record the gaps,
  including no standby run, one block size, and no roll-up for partitioned tables. The page
  is human-unverified and agent-unverified.

- 2026-08-20: **rebuilt the recommended B-tree wasted-space statement for readability and
  maintainability** in [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on
  PostgreSQL 17 (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md),
  on unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11), and **reran all 91
  mandatory tests plus fixtures 92-112 on newly initialised 17.11 and 12.2 clusters**. The
  rebuild is a **pure refactor and moves nothing the statement returns**: still sixteen
  CTEs, but `sized`/`fit`/`posting` collapse into one `page` CTE of three commented
  laterals while a new `env` and a new `gate` take the server constants and the whole
  deduplication decision out of `idx` (77 lines to 62); three statistics-state facts are
  named once in `modelled` so the caveat list and the `WHERE` clause stop spelling the same
  conditions twice; and a 35-line header maps each named change 1-6 and A-D to its stage.
  Every arithmetic expression is byte-identical and moved rather than rewritten, because
  the model mixes `numeric` and `float8`. **Identity is measured four ways on both
  servers**: `EXCEPT` in each direction between views generated from the two Markdown
  blocks returns **0 rows over 95 indexes on 17.11 and 92 on 12.2**, both exact texts give
  byte-identical `psql` output, and **0 of 129 and 0 of 106 scored fixtures disagree** on
  status, either percentage, caveats or the suppression flag. The reran suite scores
  identically under both texts: 30 deduplication-gate fixtures reproducing 421/460/1376/1931
  blocks, 0 gate over-credits and the single `i_ei_true` under-credit at `−226.4%`; 74
  partial-index tests at **66 PASS / 0 critical false positives / 0 false positives / 8
  false negatives**, with **74 of 74 live-block counts matching for the fourth consecutive
  run** and six floors moved by `ANALYZE` sampling but none across a verdict boundary; test
  16's `ei_null`/`ei_boom`/`i_squat` cases and the four DDL refusals verbatim; and the
  zero-fixture audit query at 6 rows in the fixture database against 0 in a stock one. The
  cost is stated rather than hidden — one extra `CTE Scan on idx` (5 nodes to 6), `+1.3 ms`
  of planning and `+0.8 ms` of execution at 123 indexes, `+3.2%` of execution at 600. And
  running the whole partial suite on a 12 server for the first time found the exclusions'
  portability limit: `dedup_applies` is false for all 106 indexes on 12.2, so change A's
  duplicates disjunct (25 of the 36 withheld rows on 17.11) cannot fire and **tests 30 and
  64 return as unsuppressed critical false positives at 87.6% and 93.5%** with `status = ok`
  and an empty caveat string, while the missing `pg_stat_force_next_flush()` leaves 17
  fixtures scored with `last_analyze` unset and fires change B nineteen times. Three
  fixtures are unconstructible on 12.2 (test 38's `deduplicate_items`, tests 51-52's ICU).
  Five open questions record the gaps, including no run at a block size other than 8192, no
  14.23 run of the rebuilt text, and the two new 12.2 false positives. The page remains
  human-unverified and agent-unverified.

- 2026-08-20: **applied change D** to the recommended B-tree wasted-space statement in
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), on unchanged
  pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). It is the **first exclusion
  term that reaches non-partial indexes**, so the flag is renamed `suppress_partial` ->
  `suppress_row`: with one new input column (`has_expressions`, i.e. `pg_index.indexprs
  IS NOT NULL`) and one disjunct, **a non-partial expression index with no statistics
  row of its own returns no row**. The source says why the shape cannot be priced —
  `ANALYZE` writes an index its own `pg_statistic` rows only for expression attributes
  (`analyze.c:450-478`, `:588-602`), and `index_drop` uses the same `indexprs` test when
  deciding whether an index has statistics to remove (`index.c:2341-2363`) — so the
  expression falls to the statement's 32-byte default width. Control `np97` read **64.9%
  waste on 5201 blocks a `REINDEX` reproduces exactly**: a 44-byte modelled tuple against
  a measured 120-byte item, on leaves `pgstatindex` puts at 91.31% density with zero
  fragmentation. **Two critical false positives go and no partial-index verdict moves**
  (66 PASS / 0 / 0 / 8 under both texts over tests 18-91, the disjunct carrying `NOT
  is_partial`); exactly four rows differ between the texts, all non-partial: `np97` 64.9%
  and a mixed `(k, upper(s))` key 60.5% withheld as false positives, a −614.9%
  over-prediction withheld at no cost, and one true detection lost. **Unlike change C the
  silence lifts** — one `ANALYZE` takes `np97` to a reported `−16.6%` — and an
  identical-DDL pair prices the whole trade: the same index that a `REINDEX` shrinks 5201
  blocks to 523 reads 96.4% with no statistics row and 89.2% with one, against a measured
  89.9%. Two wider forms were generated from the page's own Markdown and scored beside the
  filed one: dropping `has_expressions` also catches a plain index whose column carries
  `SET STATISTICS 0` (the one alertable hole change D leaves, 64.9% on a healthy index),
  and dropping `last_analyze IS NOT NULL` also catches a never-analysed table the alerting
  rule already covers; neither was applied. An unprivileged role reading through a
  `security_invoker` copy is untouched, because `any_stats_hidden` blocks the term.
  Report-level effect over 95 indexes and 73,867 blocks: 53 rows to 49, 35 to 31
  over the 1 MB filter, 6 readings above the 50% line to 3, 0 of 11 non-partial indexes
  withheld to 4; cost is inside the noise over eight interleaved pairs (40.1-48.2 ms as
  filed against 38.7-47.6 ms amended). Measured by restarting the change-C run's isolated
  17.11 cluster with a fresh scratch database — **74 of 74 live-block counts and all 74
  pre-change reported/withheld verdicts reproduce for the third consecutive run** — plus
  seven new non-partial fixtures numbered 106-112, four estimator texts generated
  mechanically from the page's Markdown, and `REINDEX INDEX` as ground truth; change D is
  the second term also run on the isolated 12.2 server, where the same expression fixture
  builds the same 5201 blocks and moves from a reported 64.9% to withheld. Four open
  questions record the gaps, including the discarded true detection, the `SET STATISTICS
  0` residual, the fixture-inflated above-50 count, and changes A and B still being
  17.11-only. The page remains human-unverified and agent-unverified.

- 2026-08-20: **applied change C** to the recommended B-tree wasted-space statement in
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) and
  re-scored all 74 partial-index tests against the amended text, on unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). The change is one more
  disjunct in `suppress_partial` — `bool_or(attnum > indnkeyatts AND attlen < 0)`
  in the `statvis` CTE — so **a partial index with a variable-width `INCLUDE`
  column returns no row**, and the suite reaches **0 critical false positives**
  (from 1). Exactly one of the 74 tests changes state, test 47; the four true
  detections still report at 74.7 / 74.3 / 89.1 / 94.2 against measured 74.9 /
  74.3 / 89.1 / 94.2, and the 8 false negatives are unchanged. The defect cannot
  be repaired in core SQL: a non-key column can never be an expression and
  `ANALYZE` writes index statistics only for expression attributes, so the width
  can only come from whole-table `avg_width` — 13 bytes against a stored 207 on
  test 47, a 36-byte modelled tuple against a measured 217.7, on a 787-block index
  `pgstatindex` reads at 89.71% leaf density and one `ANALYZE` moves only from
  84.0% to 83.6%. The term is scoped to `attlen < 0`, so an `INCLUDE (int)` index
  keeps its correct 0.0%; both wider forms were built from the page's own Markdown
  and priced rather than adopted — any `INCLUDE` column costs one more correct row,
  any variable-width column costs three more and is the only form that catches a
  wide **key** column, which still reads 84.1% on a fresh 792-block index with an
  empty `caveats` string. The price is measured and stated: a genuinely
  89.9%-reclaimable partial index with a `text` payload, estimated at 89.5%, is now
  withheld too, and the term emits **no caveat**, so this is the one silence a
  reader cannot see or lift. Whole-database effect over 88 indexes: 52 rows to 47,
  4 readings above the 50% line to 2, 36 partial indexes withheld to 41, 0 of 5
  non-partial indexes touched (including one with a variable-width `INCLUDE` column
  reading a correct −4.0%); cost is inside the noise over eight interleaved pairs
  (38.5-41.3 ms as filed against 39.0-42.3 ms amended). Measured by restarting the
  previous follow-up's isolated 17.11 cluster with a fresh scratch database and
  re-running its unchanged fixture scripts — **74 of 74 live-block counts and all
  74 pre-change reported/withheld verdicts reproduce exactly** — plus six new
  `INCLUDE` fixtures numbered 100-105, four estimator texts generated mechanically
  from the page's Markdown, and `REINDEX INDEX` as ground truth. Change C is also
  the only exclusion term run on an older server: a new isolated 12.2 build reports
  84.1% on the same 787-block fixture under the pre-change text and withholds it
  under the amended one. Six open questions record the gaps, including the discarded
  true detection, the wide-key family the term does not close, the caveat-free
  silence, and that changes A and B are still 17.11-only. The page remains
  human-unverified and agent-unverified.

- 2026-08-19: **applied changes A and B** to the recommended B-tree wasted-space
  statement in [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL
  17 (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) and
  re-scored all 74 partial-index tests against the amended text, on unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). The two changes are now a
  `suppress_partial` flag in the `modelled` CTE and one `AND NOT suppress_partial`
  conjunct in the `WHERE`, so a partial index missing an index-column statistics
  row, one crediting duplicates from table statistics, or one whose table has
  changed past its **auto-analyze trigger** returns no row at all. **The twelve
  critical false positives become one.** Change A withholds 8 (tests 30, 32, 49, 50,
  78, 79, 83, 85), change B withholds 3 more (64, 69, 84) plus the one plain false
  positive (66), and only test 47 — a wide `INCLUDE` column reading 84.0% on a
  787-block index a `REINDEX` reproduces block for block, with an empty `caveats`
  string — survives, exactly as the tenth follow-up predicted from the recorded
  caveat strings. No true detection is lost: tests 68, 74, 75 and 77 still report
  75.0 / 74.3 / 89.1 / 94.2 against measured 74.9 / 74.3 / 89.1 / 94.2, and the 8
  false negatives are unchanged. Change B's threshold is the trigger
  `relation_needs_vacanalyze` uses — `autovacuum_analyze_threshold +
  autovacuum_analyze_scale_factor * reltuples`, per-table reloption overrides
  included and a negative `reltuples` clamped to zero, measured at 100,050 / 50,050
  / 10,050 / 100 / 1,000,000 / 60 / 50 across seven fixtures — and not the
  `n_mod_since_analyze > 0` first sketched: `> 0` scores these 74 tests identically
  but withheld two genuinely 89.1%-reclaimable indexes on purpose-built calibration
  fixtures, and a GUC-only threshold got both reloption fixtures backwards. The
  exclusion carries `is_partial`, so 0 of 4 non-partial controls are touched, at the
  stated price of a non-partial expression index with no statistics row reading 64.9%
  unsuppressed on a freshly built 5201-block index. Whole-database effect on the
  final state: 82 rows returned to 46, 8 readings above the 50% alert line to 2, and
  9 of the pre-change top-20 triage rows withheld, 6 of them above 50%; cost is
  inside the noise (33.4 / 38.0 / 32.8 ms as filed against 37.5 / 33.6 / 35.2 ms
  amended, interleaved over 86 indexes), and no percentage moves because
  `expected_blocks` and `floor_blocks` are untouched. One new hazard is measured:
  without `pg_stat_force_next_flush()` before an `ANALYZE`, a same-session bulk load
  leaves `n_mod_since_analyze` at the full 200,000 rows and `n_live_tup` at 400,000
  for a 200,000-row table, which fires the new caveat spuriously in the safe
  direction. Measured on one isolated 17.11 server built out of tree from the pin
  `--with-icu --enable-debug`, `autovacuum`/`fsync` off, with three estimator texts
  generated mechanically from the page's own Markdown (the amended text, the
  pre-change text from `git show HEAD:`, and a `> 0` copy) and `REINDEX INDEX` as
  ground truth; the 74 fixtures are a **reconstruction** from the tenth follow-up's
  published shapes, since the original harness was deleted with its sandbox — 61 of
  the 74 reproduce their filed live-block count exactly and 13 differ, three of those
  because the reconstruction got the fixture shape wrong, none changing a verdict
  class. Five open questions record the gaps, including that the reconstruction is
  not the same population, that the `is_partial` scoping is a judgement rather than a
  measurement, that the borrowed trigger is not calibrated against a subset, and that
  the two new terms were never run on 12.2 or 14.23. The page remains
  human-unverified and agent-unverified.

- 2026-08-19: folded **74 partial-index mandatory tests** into the mandatory-tests
  section of [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), taking
  that suite to 91 tests in one numbering, against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). **The recommended statement
  fails the primary requirement.** Scored on `wasted_space_pct_floor`, as the
  requirement specifies: 52 PASS, **12 critical false positives**, 1 false positive,
  8 false negatives, 1 cell the pass/fail rule leaves undefined. The twelve report
  84.0% to 99.6% waste on freshly built or freshly grown partial indexes that a
  `REINDEX` reproduces block for block, and the page's filed alerting rule suppresses
  none of them; the eight false negatives are indexes a `REINDEX` shrinks by 73-94%
  reading under 20%. One structural fact explains 20 of the 22: `ANALYZE` builds
  per-column statistics for an index only when the index has expressions, so a
  partial index on a plain column carries a row count and nothing else and is priced
  from whole-table `avg_width`, `null_frac` and `n_distinct` — measured as 0
  `pg_stats` rows for six plain-column partial indexes against 3 for the expression
  ones. Materialising each predicate subset as its own table and pointing the same
  statement at it modelled all seven within 4.1 points and matched the partial twin's
  rebuilt size exactly in all seven, which locates the defect entirely in the
  statistics input rather than the arithmetic. The floor column is proved immune by
  construction to the duplication family (`n_distinct`, MCV and extended-statistics
  mismatches reach 70.6% on the point estimate and 0.0% on the floor) and fully
  exposed to width, NULL-fraction and row-count mismatch. One `ANALYZE` repairs 7 of
  the 12 (93.5% to 0.3%, 99.6% to 0.0%, 86.7% to 1.1%) and nothing in core SQL
  repairs the other 5, because no catalog holds a plain-column partial index's
  subset width or NULL fraction. Two measured alert-routing changes would take the 12
  to 1 at zero cost in true detections — suppress on two caveats the statement
  already emits, and add a partial-index staleness caveat keyed on
  `n_mod_since_analyze`, which separated 0 from 300,000 and 399,000 across a healthy
  index, two stale ones and a genuinely 89%-reclaimable one — and until they are
  applied the recommended-statement section excludes partial indexes from any alert.
  Three incidental findings were filed: a B-tree index datum over 510 bytes is
  pglz-compressed in place (142 blocks for 20,000 compressible 1001-byte keys against
  1560 for 20,000 incompressible 481-byte ones), a same-session `DELETE; VACUUM`
  leaves `n_dead_tup` non-zero because VACUUM and ANALYZE write the counters
  absolutely while pending per-backend deltas add on top, and an append-only partial
  index measured 880 blocks live against 881 rebuilt. Measured on one isolated 17.11
  server built from the pin `--with-icu --enable-debug`, 74 partial indexes over 60
  tables, with `REINDEX INDEX` as ground truth and the statement extracted
  mechanically from the page's own Markdown; the server was shut down cleanly
  afterwards and its sandbox retained under `.wiki-runtime/tmp/partial17/`. Nine
  open questions record what was not done, including that no 12.2 or 14.23 run backs
  the partial-index numbers and that neither proposed change was applied and
  re-scored end to end. The page remains human-unverified and agent-unverified.

- 2026-08-18: folded change 6 into the recommended B-tree wasted-space statement on
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) and
  **re-measured every server-measured table on that page**, against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). The section is retitled "with
  all six changes", change 6 keeps its rationale but loses its duplicate SQL block,
  and the recommendation is now one block of SQL with no assembly step. Nothing on
  the page is a 17.10 observation any more. Four servers: one isolated 17.11 install
  configured `--with-icu --enable-debug` carrying four scratch databases (the 15 v12
  fixtures plus the 54-cell matrix and Methods A/A-prime/B/C/D; the twelve-issue-review
  family with its unprivileged `probe` role; the 28 mandatory-test fixtures with their
  eight custom operator classes; a fresh database for the runnable harness), a second
  17.11 cluster plus isolated 14.23 and 12.2 servers for the 12-through-17 family.
  Every scored statement text, including the pre-change-6 form used for the "before"
  column of every comparison, was generated mechanically from the page's own Markdown,
  so the scored text is the filed text plus documented edits. Reproduced exactly: all
  15 `pgstatindex` fixture rows, Method C's rebuild sizes, Method B's 36-of-36 census
  and its 18 never-vacuumed `Heap Fetches` catches, the 313.58%-against-96.15% density
  blow-up, the gate scoreboard (5 over-credits before change 6, 0 after, 1 deliberate
  under-credit; 8 for the earlier v17 sweep), the audit query's 6 / 0 / 0 rows, the
  harness's 23 indexes with 11 "can" and 11 "cannot" `DEBUG1` verdicts, and every 12.2
  blocker (`invalid function number 4`, `unrecognized parameter "deduplicate_items"`,
  `ICU is not supported in this build`). Moved only where `ANALYZE` sampled
  differently: Method A exact on 11 of 15 fixtures rather than 9 (the partial fixture
  landed on 50,034 estimated rows), the 54-cell matrix at 33 exact rather than 30, the
  duplication sweep's 10-rows-per-key row flipping to a negative `n_distinct` and
  losing its credit, and `i_q1000` modelling 899 blocks rather than 901 on 8 stored
  most-common values instead of 11. One earlier claim is corrected: change 6 does not
  under-credit an opclass whose support function is a `LANGUAGE internal` alias of
  `btequalimage` — `prosrc` is what the whitelist matches, so both texts report 69.3%
  on a true 69.4% — and the cost is confined to non-internal support functions,
  re-measured with a SQL one. Newly measured, because this 17.11 build has ICU:
  the nondeterministic-collation conjunct (a healthy 3611-block index reads 0.1%
  against the earlier sweep's 87.6%), change 1's residual over-prediction on a
  two-column and a partial fixture (−24.8% and −33.1%), a never-rebuilt 1.5M-row
  random-insertion twin at 25.9% beside its rebuilt sibling's 11.5%, and interleaved
  timings that price the five earlier changes at roughly 30% while change 6 stays
  inside the noise. Twenty-two open questions were rewritten and three retired as
  resolved. All servers were stopped afterwards. The page remains human-unverified and
  agent-unverified.

- 2026-08-18: added a maintained "current recommended statement" section to
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), at
  the user's request and against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c`, together with the standing
  requirement that drives it. The recommendation is the corrected statement with
  all five changes, with change 6's `all_equalimage` subquery substituted for the
  existence test. It is assembled in two stated steps rather than copied, so no
  second SQL text can drift, and the 17.11 and 12.2 runs behind the seventeen
  mandatory-test verdicts were generated exactly that way — the assembled text is
  one the page has run, not one it only describes. The section ranks all six
  variants on the three criteria the prompt named. Accuracy: **0 over-credits**
  over 28 zero-waste fixtures against 5 for the same statement as filed and 8 for
  the earlier v17 sweep, 0 fixtures above 30% on either percentage column against
  3 and 4, plus the five changes' own gains (`i_q1000` +5.5% to −0.6%,
  `i_ext`/`i_sup` −206.4% to +0.1%, and a genuinely 49.8%-reclaimable correlated
  index from an unalertable −38.3% to +49.9%). Fixes: six numbered changes on top
  of the portable statement's gate conjuncts and both reporting corrections,
  ending in the catalog form of what `_bt_allequalimage` actually does — look the
  support function up and **call** it — with `prosrc` rather than `proname` as the
  identity, which is what `fmgr` resolves for a `LANGUAGE internal` function.
  Compatibility: 12 through 17 unchanged, measured on 12.2 and 17.11 for this
  exact text and on 14.23 and 17.10 for the pre-change-6 text, with 13, 15 and 16
  never run and `pg_language.lanname` / `pg_proc.prosrc` confirmed present on
  12.2. Four residual errors are stated with their direction, all safe for a
  floor-based alert except the last: change 1's −99.4% band just above a multiple
  of the 132-TID cap, change 2's −88.6% on independent columns, change 6's single
  custom-opclass under-credit (a real 69.4% rebuild win reported as 0.1%), and the
  27.1% a randomly inserted, never-deleted index reads on both columns while the
  model is exact to the block. The section also records what the recommendation
  does not replace — Method C as the only exact arbiter, Method B's leaf census,
  Method A-prime's sampled width — and the rule for displacing it: name the trade
  instead of switching silently, since an over-prediction and an over-credit are
  not interchangeable. Source-only, with no server run; two new open questions
  record that the ranking covers only this page's own statements and fixtures, and
  that the assembled text has two servers and one 28-fixture set behind it while
  every per-fixture number it inherits was measured without change 6.
  `raw/postgres-17/` is unchanged, and the page stays human-unverified and
  agent-unverified.

- 2026-08-18: filed a seventeen-test mandatory deduplication-gate suite as a
  seventh follow-up on [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on
  PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md),
  against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c`, and scored
  both deduplication-aware statements on the page against every test. The
  corrected 12-through-17 statement fails 3 of 17 and the earlier v17 sweep fails
  6; every failure is the same defect, a gate that asks whether an equal-image
  support function exists in `pg_amproc` while `_bt_allequalimage` looks it up and
  then **calls** it. Measured on a **17.11 ICU** build of the pin
  (`--with-icu --enable-debug`, `block_size` 8192, `autovacuum`/`fsync` off) in a
  scratch database with 28 fixtures that have nothing to reclaim, plus the
  isolated 12.2 server for version compatibility. Eight custom operator classes
  supply what no built-in type can — `amvalidate` returns true for SQL and
  PL/pgSQL support functions, since `btvalidate` and `assignProcTypes` check only
  arity, boolean return and non-cross-typedness — and a NULL-returning function
  aborts `CREATE INDEX` with `function 16575 returned NULL` while a raising one
  propagates its exception. As filed the gate over-credits 5 fixtures: a healthy
  1376-block index whose support function returns false reads **69.3%**, and two
  mixed TRUE/FALSE two-column indexes read **78.1%** once a `CREATE STATISTICS
  (ndistinct)` object gives change 2 something to compress; the floor column stays
  at 0.1-0.2% throughout, so the floor-based alert rule was immune and the point
  estimate was not. Change 6 requires the registered proc to be `LANGUAGE
  internal` with `prosrc` in (`btequalimage`, `btvarstrequalimage`): 0
  over-credits, one deliberate under-credit (a custom opclass that really does
  deduplicate, 421 blocks, reads −226.4%), 12.2 / 12.5 / 16.6 ms against the filed
  12.8 / 12.1 / 14.0 ms over 34,164 blocks, and no reading moves on a stock
  database, whose 29 B-tree support-function-4 rows are 26 `btequalimage` plus 3
  `btvarstrequalimage`, all `LANGUAGE internal` in `pg_catalog`. `prosrc` beats
  `proname` on measured fixtures: a `LANGUAGE internal` alias in `public` and a
  renamed-away built-in break the name test as under-credits while a same-named
  SQL impostor at a fresh OID breaks it as an over-credit, which `fmgr`'s built-in
  fast path and its `LANGUAGE internal` `prosrc` lookup explain exactly. Rewriting
  a support function to return false leaves the metapage true, makes
  `bt_index_check` raise `metapage incorrectly indicates that deduplication is
  safe`, and a `REINDEX` then takes the index from 421 to 1376 blocks — which the
  corrected reading predicted to within 0.4 points, closing the earlier
  metapage-versus-catalog question with a measurement. The reverse mutation is
  filed as the honest cost: adding a working `FUNCTION 4` to an opclass that had
  none makes a rebuild reclaim a true 69.4% that change 6 reports as 0.1%. On 12.2
  seven of the seventeen tests are unconstructible (`invalid function number 4`,
  `unrecognized parameter "deduplicate_items"`, `ICU is not supported in this
  build`) and all four statement variants agree with the engine on all 10
  buildable fixtures, with `i_multi_bad` reading 28.8% on both majors — an
  `avg_width` truncation to 4 on a 5-byte `numeric`, not a gate error. Ships a
  zero-fixture audit query (6 rows in the fixture database, 0 on a stock 17.11 and
  0 on 12.2) and a fixture harness that was re-run verbatim from the page text: 23
  indexes, 11 engine "can", 11 "cannot", and no verdict at all for the `INCLUDE`
  index. 13 new open questions record what was not measured, including that the
  earlier tables were not re-scored with change 6 and that the whitelist is a
  fixed two-name list. `raw/postgres-17/` is unchanged, and the page stays
  human-unverified and agent-unverified.

- 2026-08-18: worked a twelve-issue external review of the portable
  12-through-17 statement on [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat
  Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) as a
  sixth follow-up, measured on a new isolated **17.11** server built out of tree
  from the current pin `786db8dcf168bd9df8f55047337525ac19118b1c` — the first
  numbers on that page taken on the pinned commit rather than the previous 17.10
  one — plus an isolated 12.2 server for portability. The ledger is four issues
  already fixed (the negative `n_distinct` branch, the nondeterministic-collation
  and `indnatts = indnkeyatts` conjuncts, and the two reporting defects), one
  mischaracterized, five applied and two rejected. The mischaracterization is the
  page's own attribution of `i_q1000`'s 49-block gap to page packing: 48 blocks
  are a posting-tuple count error — the model divided a class's rows by its TID
  count, letting one tuple span two key groups, while `_bt_load` flushes the
  pending posting list at every key change — and 1 block is internal-fanout
  error, because pivots above a deduplicated leaf level carry a heap TID and
  measure 212 per page against the modelled 284; leaf packing was already right,
  confirmed by a leaf holding nine 808-byte posting tuples plus a 24-byte
  truncated high key. Applied, with directions stated: the prescribed key-group
  round-up removes every phantom-bloat reading above 264 rows per key (`i_q1000`
  +5.5% to −0.6%, `i_r500` and `i_r1000` +5.5% to −0.2%/−0.3%, `i_r265` +10.8% to
  −33.5%) and is **not** a strict improvement, because it prices each group's
  last, partial posting tuple at full size and so over-predicts to −99.4% at 133
  rows per key and −88.7% at 143, while `i_cd` does not move at all (its 38 blocks
  are an off-by-one leaf capacity: 12 modelled posting tuples where the build fits
  11 data items plus the high key); `pg_stats_ext.n_distinct` replaces the
  per-column product when an `ndistinct` object covers the whole key, which is the
  only change that recovers a missed alert — a correlated `(a, b)` index with 49.8%
  genuinely reclaimable read `−38.3%` with a `−53.3%` floor and now reads `+49.9%`,
  one block from its rebuild — with measured superset-covering, wrong-column-set
  fallback, reversed-column and expression-key behavior, and `pg_stats_ext`'s
  owner-membership condition (8 rows to the owner, 0 to an unprivileged role);
  a `statistics not visible to this role` caveat separates a withheld `pg_stats`
  row from an absent one and joins the alert-suppression set, because the 32-byte
  default `avg_width` makes a healthy 100-byte-key index read 62.5% on **both**
  columns to a role without `SELECT` (0.0% as owner), the same on 12.2; a
  randomly inserted, never-deleted index reports 27.1% on both columns at 65.68%
  leaf density and 49.87% fragmentation with zero deleted pages, its model
  exactly equal to the 2745-block rebuild, which is the one valid issue the
  floor-plus-status-plus-caveats rule cannot defend against; and the baseline is
  stated from `_bt_findsplitloc` — fillfactor only on rightmost splits, 50:50
  elsewhere — with a measured drift table in which the 1020 blocks a rebuild
  reclaimed came 10.9% back after 500,000 further random inserts and a fresh
  rebuild landed on the model's 4116 blocks exactly. Rejected: `least(reltuples,
  n_live_tup)` stays, since `reltuples` alone reports the 19%-drained `i_stale` as
  0.0% and loses 521 blocks on both columns, and `n_live_tup` is DML-maintained
  without `ANALYZE`; `bt_metap().allequalimage` stays out, since the build
  recomputes the flag from the catalog, a pg_upgrade'd v12 index reports false
  until a `REINDEX`, the function is superuser-only, and a `deduplicate_items =
  off` index reports true while holding no posting lists — the metapage answers
  "may this index deduplicate as it stands", not "would a fresh build
  deduplicate". The statement runs unchanged on 12.2, every added construct
  exists in 12 (`pg_stats_ext.n_distinct` renders `{"1, 2": 1000}` there too), and
  `inherited`/`exprs` are deliberately not referenced because 12 lacks them. 15
  new open questions record what was not measured, including the unapplied exact
  leaf-capacity and two-size-posting-tuple refinements, the untested RLS half of
  the new caveat, the un-recomputed 30%-threshold alert table, and that the rest
  of the page remains a 17.10 observation. `raw/postgres-17/` is unchanged, and
  the page stays human-unverified and agent-unverified.

- 2026-08-18: renamed the reporting identifiers of the core-SQL B-tree bloat
  statements on [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12
  (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md) off the
  word "bloat", at the user's request and against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2), matching the same correction
  made to the v17 page a day earlier. `wasted` is now `wasted_space`, `bloat_pct`
  is `wasted_space_pct`, the Method A tag is `wiki_btree_wasted_space_sweep`, and
  the Method C probe index is `wiki_wasted_space_probe` in all five places it
  appears; the two accuracy-table headers became `model wasted_space %` and
  `true wasted_space %`, while the page title, the conceptual use of "bloat" and
  the fixture-recipe `bloat type` column stay as filed. Source-only: nothing was
  executed, so "no number moves" is an argument about which expressions changed.
  The v12 evidence differs from v17's in three ways worth recording. There is no
  glossary in this checkout, so the normative definition comes from
  `ref/reindex.sgml` — a bloated index "contains many empty or nearly-empty pages"
  — reinforced by `maintenance.sgml`'s partly-emptied-page mechanism, both per-page
  state that a file-size-minus-model statement cannot see, while `ref/copy.sgml`
  supplies "wasted space" for what a maintenance command recovers; all 24
  doc-tree matches for the string are prose, `system_views.sql`, `pg_proc.dat` and
  every contrib SQL script have none, and `pgstattuple` says
  `free_space`/`free_percent`. The query ID is computed in contrib
  `pg_stat_statements` rather than core, and its `JumbleExpr` `T_TargetEntry` case
  hashes `resno` and `ressortgroupref` only, so `resname` cannot move it, and the
  lexer's `xc` state discards the tag before parsing. And the probe rename is not a
  label change at all: because v12 keys a utility statement by
  `pgss_hash_string` over its text, the renamed `CREATE`/`DROP INDEX CONCURRENTLY`
  get new `pg_stat_statements` entries, while inside the two `SELECT`s the same
  name is a constant that `RecordConstLocation` and `generate_normalized_query`
  reduce to `$n`. Five new open questions record what was not run, including that
  the byte column stays clamped at zero beside a signed percentage — `idx_dup` at
  −6.4% and `idx_var` at −4.6% are the two cells affected — and that v12's
  `half_rounded` rounds toward positive infinity rather than symmetrically. No
  server was started, `raw/postgres-12/` is unchanged, and the page remains
  human-unverified and agent-unverified.

- 2026-08-18: renamed the reporting columns of both bloat statements on
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on
  PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) off the
  word "bloat", at the user's request and against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). `wasted` is now
  `wasted_space`, `bloat_pct` is `wasted_space_pct`, `bloat_pct_floor` is
  `wasted_space_pct_floor`, and the two statement tags are
  `wiki_btree_wasted_space_sweep_v17` and `wiki_btree_wasted_space_sweep_12_17`;
  the page's prose follows the new names, with two artifacts left verbatim (the
  psql capture taken before the rename and the as-filed expression quotes in the
  reporting-defect table). Source-only: nothing was executed, so the claim that
  only `AS` labels moved rests on the two `SELECT` lists plus the pinned tree.
  `AS` assigns an output column label and nothing else, both `ORDER BY` keys are
  expressions rather than labels (a label is usable there only as a bare sort
  key), and the rename cannot move `queryid`, because `JumbleQuery` hashes the
  post-parse-analysis `Query` tree and `TargetEntry.resname` is declared
  `pg_node_attr(query_jumble_ignore)`, for which `gen_node_support.pl` emits no
  code into the generated `queryjumblefuncs.funcs.c`. The 22-byte names sit far
  below the `NAMEDATALEN` truncation point, where the lexer's
  `downcase_truncate_identifier` would raise a `NOTICE`. The rename is more than
  cosmetic: the glossary defines bloat as space in data pages holding no current
  row versions - per-page state that these statements, which subtract a modelled
  rebuild size from the file size, cannot see - while "wasted space" is the docs'
  own phrase for space a maintenance command recovers, and no core view,
  `pg_proc.dat` entry or contrib SQL script names a column `bloat`
  (`pgstattuple` calls the quantity `free_space`/`free_percent`; the only
  non-prose occurrence in the tree is an unrelated static in the imported IANA
  timezone compiler). Consumer impact is filed too: an old name now raises
  `ERRCODE_UNDEFINED_COLUMN`, `pg_size_pretty` returns `text` so ranking belongs
  on the byte expression through a suggested `wasted_space_bytes`, and an existing
  `pg_stat_statements` entry keeps showing the old tag because the query text is
  written only when the hash entry is created. Five new open questions record
  that no server ran the renamed statements. No server was started,
  `raw/postgres-17/` is unchanged, and the page remains human-unverified and
  agent-unverified.

- 2026-08-18: extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on
  PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with a
  source-only follow-up that confirms and corrects two reporting defects both of
  its bloat statements shipped with, against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). Neither defect touches
  `expected_blocks`, `floor_blocks`, `live_rows`, the deduplication gate, `status`
  or `caveats`, so no percentage on the page moves; the two statement blocks now
  carry the corrected columns and the measurements are labelled as taken with the
  defective ones. First, `fsm_bytes > 0 AS has_freed_pages` reports history, not
  state: an index's FSM stores only free-or-used (`RecordFreeIndexPage` writes
  `BLCKSZ - 1`, `RecordUsedIndexPage` writes 0), the first recorded page extends
  the fork to a fixed three blocks / 24576 bytes at `block_size` 8192
  (`SlotsPerFSMPage` 4069, `FSM_TREE_DEPTH` 3), `_bt_allocbuf`'s
  `GetFreeIndexPage` marks pages used without shortening the file, no nbtree path
  reaches `FreeSpaceMapPrepareTruncateRel` (its live callers are heap truncation
  and `RelationTruncateIndexes`; the SP-GiST call is `#ifdef NOT_USED`), and only a
  storage replacement through `RelationSetNewRelfilenumber` — `REINDEX`,
  transactional `TRUNCATE` — resets it, so the column reads false exactly when the
  index is fresh. In the other direction `btvacuumpage` records a deleted page only
  once `BTPageIsRecyclable` holds, `_bt_pendingfsm_finalize` stops at the first
  still-visible `safexid`, half-dead pages are never recorded, and partly-emptied
  leaves — the documented bloat mechanism — never enter the map at all, so a
  deleted-page-heavy index can read false. Renamed `fsm_written_since_build`, with
  the current empty/deleted/reusable counts sourced from `VACUUM VERBOSE`'s
  `pages: N in total, N newly deleted, N currently deleted, N reusable` line
  (`IndexBulkDeleteResult`, where nbtree treats `pages_free` as whole-index state)
  in core, or from contrib: `pgstatindex.deleted_pages` plus `empty_pages` (which
  is in fact the half-dead class), `pg_freespace` with the `avail > 0` idiom its own
  regression test uses on a B-tree index, and `pageinspect`'s per-page `type`. A
  `pg_read_binary_file` decode of the `_fsm` fork is noted as possible in core SQL
  and not attempted. Second, `wasted` was clamped at zero while `bloat_pct` was
  not, so `i_var` on 17.10 printed `0 bytes` beside -4.4% while the signed value is
  -1,163,264 bytes, and the earlier dedup-aware sweep printed `0 bytes` beside
  -92.5% while hiding 10,805,248 bytes; a negative percentage is shown to mean only
  that a rebuild is not predicted to shrink the index — usually model
  over-prediction, but the page's own `idx_dup` really did grow from 396 to 426
  blocks on rebuild — so both columns and the `ORDER BY` key now use one signed
  convention, which `pg_size_pretty` supports symmetrically and the `dbsize`
  regression test asserts. Seven new open questions record that nothing was
  re-measured: no server ran the corrected statement, the rendered strings
  `-1136 kB` / `-10 MB` / `-7824 kB` are derived by hand, and no fixture's FSM state
  was ever recorded, so neither direction of the boolean's failure was observed. No
  server was started, `raw/postgres-17/` is unchanged, and the page remains
  human-unverified and agent-unverified.

- 2026-08-17: removed the v17 question page Detecting Bloat in All Index Types by
  Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 17 (unverified) at the
  user's request, together with its root-index and v17-landing-page entries, the
  two inbound `## Navigation` links on
  [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 17 (unverified)](v17/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md)
  and
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md),
  and the index/heap-ratio clause of the v17 coverage cell above that described
  it: the screening-signal-only verdict, the 49-cell drift-versus-"index larger
  than the heap" scores, the `hashbuild`/`estimate_rel_size` bucket-sizing and
  same-VACUUM-recycling findings, the day-zero sweep, the GIN
  baseline-1.004900 fixture, and the comment durability matrix. No surviving v17
  page referred to the page in prose. The two historical coverage notes below
  that referenced it now name it as plain text. The v12 page of the same name was
  removed on 2026-08-13, so no supported version carries this index/heap-ratio
  proposal any more; COMMENT-stored index-bloat screening in v17 now runs through
  the bytes-per-table-tuple page named above, and core-SQL B-tree bloat
  measurement through the B-tree page named above. The pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` and every surviving page's citations
  and verification fields are unchanged.

- 2026-08-17: repinned the four non-legacy versions to their upstream branch tips
  and reviewed every commit in each range: v14 14.23 `5c00f4e2e3b` -> 14.24
  `a92fbdfb830` (133 commits), v17 17.10 `54eeefaedbe` -> 17.11 `786db8dcf16`
  (193), v18 18.3-line `6cb307251c5` -> 18.6 `baa7b142aac` (294, crossing the 18.4
  and 18.6 releases; 18.5 was stamped but never released), and v19 19beta2
  `99e47536bbf` -> 19beta3 `67342a14863` (152). v12 stays pinned to 12.2
  `45b88269a35` because its pages carry measurements taken on a 12.2 build.
  The v19 checkout had drifted 52 commits *behind* its declared pin, and that pin
  was not even present in the local clone, so every v19 citation line number
  resolved against the wrong tree; `scripts/wiki_lint` now checks the pinned commit
  against the checkout's HEAD so this cannot recur silently. 1,981 citation line
  fragments were re-anchored with the new `scripts/repin_citations`, 493 stale
  labels synced, 38 anchors repointed by hand, and 11 pre-existing citations that
  ran past end-of-file were fixed (including seven on v12 pages). Five claims that
  the ranges falsified were corrected: the v14 and v18 same-role RLS plan-cache
  staleness finding (closed by CVE-2026-14666), the v17 exclusion-index non-reuse
  claim (`1d6c654c818`), the v17 RANGE DEFAULT-partition pruning bug
  (`31f2acde53d`), the v17 "no related commits since 17.10" statement, and the v19
  pgrepack direct-use rejection site (`5d47df21e89`). The v18 bidirectional page's
  open question about unquoted publisher-side identifiers is answered by
  CVE-2026-6638. Measurements taken on the previous pins are kept and labelled as
  such rather than re-run, with a per-page source check of whether the underlying
  code moved; every affected page keeps `verified: false` and
  `verified_by_agent: not yet`.

- 2026-08-17: removed the v12 question page Can COMMENT-Stored Table DML
  Counters Trigger GIN REINDEX at 40% in PostgreSQL 12? (unverified) at the
  user's request, together with its root-index and v12-landing-page entries and
  the one inbound `## Navigation` link on
  [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 12 (unverified)](v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md).
  No surviving v12 page referred to it in prose, and the v12 coverage cell above
  never described it, so no coverage clause was dropped. The four historical
  coverage notes below that referenced the page now name it as plain text.
  COMMENT-stored index-bloat screening in v12 now runs through the
  bytes-per-table-tuple page named above, and core-SQL B-tree bloat measurement
  through
  [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md);
  no v12 page proposes a table-DML-counter GIN rebuild trigger any more. The pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` and every surviving page's
  citations and verification fields are unchanged.

- 2026-08-13: removed the v12 question page Detecting Bloat in All Index Types
  by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 12 (unverified)
  at the user's request, together with its root-index and v12-landing-page
  entries, the two inbound `## Navigation` links on
  [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 12 (unverified)](v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md)
  and Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in
  PostgreSQL 12? (unverified),
  and the index/heap-ratio clause of the v12 coverage cell above that described
  it, including its drift-versus-"index larger than the heap" and
  `REINDEX INDEX CONCURRENTLY` follow-ups. The six historical coverage notes
  below that referenced the page now name it as plain text. COMMENT-stored
  index-bloat screening in v12 now runs through the bytes-per-table-tuple and
  GIN DML-counter pages named above, and core-SQL B-tree bloat measurement
  through
  [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md).
  The separate v17 page Detecting Bloat in All Index Types by Storing an
  Index/Heap Size Ratio in COMMENT in PostgreSQL 17 (unverified) and its inbound
  links were untouched then, so v17 still carried this proposal on that date; it
  was removed on 2026-08-17, as the note above records. The
  pin `45b88269a353ad93744772791feb6d01bc7e1e42` and every surviving page's
  citations and verification fields are unchanged.

- 2026-08-13: filed [Changes to Declarative Partition Bound Syntax and
  pg_class.relpartbound Since Partitioning Was Introduced, as of PostgreSQL 18
  (unverified)](v18/questions/server-administration/declarative-partition-bound-syntax-and-relpartbound.md)
  against unchanged pin `6cb307251c5c6261286c1566496920976640108e` (18.3).
  Verdict: yes to both, but every substantive change landed early. The bound
  grammar changed three times and has been frozen since v12 — `d363d42bb9a`
  replaced `UNBOUNDED` with `MINVALUE`/`MAXVALUE` before 10.0 shipped,
  `6f6b99d1335` and `1aba8e651ac` added `DEFAULT` and hash
  `WITH (MODULUS m, REMAINDER r)` in v11 (with `9361f6f54e3` adding the
  sentinel-ordering rule), and `7c079d7417a` collapsed the literal enumeration
  into a general `partition_bound_expr` in v12. The complete `git log -L` over
  the `PartitionBoundSpec:` production for `REL_10_0..6cb30725` returns only six
  commits, the later ones being a typo fix, two indentation runs, and the v18
  commit `2d8bff603c9`, which adds `parser_errposition(@3)` to the two
  hash-bound errors and is v18's sole grammar change. The `pg_class.relpartbound`
  column definition never changed, while the stored `pg_node_tree` changed twice:
  v11 added `is_default`/`modulus`/`remainder`, and v17's `d20d8fbd3e4` made
  `nodeToString()` write `-1` for every `:location`. The v16 switch to generated
  node support (`964d01ae90c`) left the output byte-compatible, and `outDatum`
  and `_outConst` are byte-identical to `REL_10_0`. The deparsed
  `pg_get_expr(relpartbound, oid)` text never changed, and no typed bound
  accessor was ever added, so the three SQL forms remain the opaque node tree,
  the deparsed `FOR VALUES ...` string, and the derived
  `pg_get_partition_constraintdef()` boolean; exposure changes since v10 are
  additions only, the newest being v14's ` DETACH PENDING` psql annotation.
  Also documented: `MODULUS`/`REMAINDER` are `NonReservedWord` `DefElem`s rather
  than keywords, `MINVALUE`/`MAXVALUE` reach the transform as `ColumnRef`s,
  MERGE/SPLIT PARTITION were reverted by `3890d90c150` before 18.0, the
  write-once/clear-once `relpartbound` lifecycle including the two-transaction
  concurrent detach, and the three engine read paths in `partdesc.c`,
  `partcache.c` and `partbounds.c` with the v18 hardening `c899c6839f5`.
  Measured on an isolated 18.3 build: the literal stored node text for hash,
  list, range and default bounds with `:location -1` at every level in 14/14
  bounds; `FOR VALUES IN ((2+1), 5, NULL)` stored as `Const` 3 and deparsed as
  `FOR VALUES IN (3, 5, NULL)`; `WITH (REMAINDER 0, MODULUS 4)` accepted and
  canonicalized to `WITH (modulus 4, remainder 0)`; quoted `"minvalue"` accepted
  as the sentinel while `"MINVALUE"` is rejected; `FROM (-5)` deparsing as
  `FROM ('-5')`; `pretty = true` identical to the plain call on 14/14 bounds; a
  detach-then-reattach and a `pg_dump -s` round trip reproducing the bound
  exactly; `DETACH` clearing `relpartbound` and `relispartition`; and a no-TOAST
  size cliff where an 8,000-character hex bound stores in 7,627 bytes (25,720
  characters of node text) while 8,400 characters fails with
  `row is too big: size 8216, maximum size 8160`. Four open questions were
  filed, including the platform-specific size cliff, whether `d20d8fbd3e4`
  considered its `pg_node_tree` catalog side effects, the undocumented and
  untested quoted-sentinel asymmetry, and the unmeasured foreign-table partition
  case. Three test gaps were recorded: no test anywhere in the tree asserts the
  stored node text, the size cliff, or the quoted-sentinel behavior. The
  isolated server was stopped and its data directory removed. The page is
  human-unverified and agent-unverified.

- 2026-08-12: filed [Extracting Declarative Range Partition Bounds With SQL and
  No Regex in PostgreSQL 12
  (unverified)](v12/questions/server-administration/extract-range-partition-bounds-without-regex.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2).
  Verdict: v12 has no catalog column, view, or function that returns a range
  bound as a typed value, so extraction must decompose the
  `pg_get_expr(relpartbound, oid)` text — the same string `psql \d` prints and
  `pg_dump` pastes into `ATTACH PARTITION` — and no regex is needed because the
  deparser's grammar is fixed: `FOR VALUES FROM (` … `) TO (`, elements joined by
  `, `, bare `MINVALUE`/`MAXVALUE`, `DEFAULT` for a default partition, and
  literals with no `::type` and no `COLLATE` because `get_range_partbound_string`
  calls `get_const_expr` with `showtype = -1`. The filed recipe splits the text on
  the quote character with `string_to_array`, masks each literal's contents to an
  equal-length run so quote parity is decidable (the two quotes of an escaped
  quote are adjacent, so every spurious "outside" segment is empty), then finds
  the separators with `strpos`/`substr`; it uses `substr` rather than `substring`
  because only `substring(text, text)` resolves to the regex `textregexsubstr`.
  Measured on an isolated 12.2 server over 18 range parents, 45 partitions and 95
  creation-time control elements: a from-scratch reconstruction of the deparse
  rules matched `pg_get_expr` on 41/41 partitions, the recipe scored 95/95, and the
  obvious `split_part` + `btrim` version scored 81/94 elements and got 9/41
  partitions wrong on values containing `) TO (`, `, `, or an escaped quote.
  `pg_get_expr(relpartbound, 0)` and the `pretty = true` form were byte-identical
  to the canonical call on 45/45; no bound text contained `::`; two contained a
  newline inside a literal. Also measured: `DateStyle`, `TimeZone`,
  `IntervalStyle`, `extra_float_digits`, `bytea_output` and
  `standard_conforming_strings` all change the text (all `PGC_USERSET`), with
  `standard_conforming_strings = off` doubling backslashes and dropping the score
  to 91/95 until an `E''`-string `replace` restored 95/95 under both settings —
  while the plain `replace(x, '\\', '\')` spelling does not even parse under that
  setting; 14 catalog `AccessShareLock`s and zero partition locks at 1048
  partitions, 39.9-44.9 ms for the masked recipe versus 8.3-9.2 ms for the plain
  split; an unprivileged role read all 2109 rows including bounds for a schema it
  cannot use; `DETACH` erases `relpartbound`; a concurrently dropped partition
  makes `pg_get_expr(…, oid)` return NULL while `pg_get_expr(…, 0)` still
  deparses; the value round trip through the key type was exact in 78/81, the
  exceptions being `boolean`'s `true`/`false` keywords; and a 9600-character bound
  fails at DDL time with `row is too big: size 9304, maximum size 8160` because
  `pg_class` has no TOAST table. Seven open questions were filed, including the
  complete upstream absence of any test that deparses a quoted RANGE bound
  containing a comma, quote, or backslash, the untested foreign-table partition
  case, and expression-key type derivation. The test database and server were
  removed. The page is human-unverified and agent-unverified.

- 2026-08-12: filed [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX
  Threshold for Every Non-B-Tree Index in PostgreSQL 12
  (unverified)](v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2).
  Verdict: the scheme calibrates for exactly one of the five non-btree access
  methods. Measured on an isolated 12.2 server over 132 cells (22 insert/update/
  delete workloads x 6 index shapes) with `REINDEX INDEX` as ground truth and
  "worth rebuilding" defined as reclaiming >= 20% of current bytes: GiST
  separates perfectly at any threshold in 20-30% (14 TP / 0 FP / 0 FN / 8 TN)
  and a fresh zero-bloat GiST build drifts only -0.29% to +3.84% across
  25k-1.6M rows; hash (window 8.78-11.11%) and SP-GiST (28.24-32.17%) also score
  100% but only at a fixed row count, because a *freshly built* hash index reads
  -20.31% to +49.42% across the power-of-two bucket staircase (28.071 to 52.634
  bytes/row over 60k-250k rows) and SP-GiST up to +33.08%; GIN needs 50%
  (`fastupdate = off`, still 2 FP) or 75% (on, 18/0/0/4); BRIN has no working
  threshold at all, `REINDEX` reclaiming exactly 0.00% in 22/22 cells while the
  reading ranged -87.50% to +9900.00%. One all-method threshold of 30% scores
  90.2% over 132 cells and 93.6% over the 110 non-BRIN cells. Rank correlation
  with reclaimable bytes: GiST 0.992, hash 0.987, GIN-on 0.974, SP-GiST 0.958,
  GIN-off 0.936, BRIN undefined. Also measured: every access method reads exactly
  `1/(1-f)-1` after a delete of fraction f (11.11/33.33/100/300/900/9900%), so
  deletes carry no index information; `VACUUM` never shrank any index (byte-
  identical sizes in every delete cell); the GIN pending list reads +261.73% and
  `gin_clean_pending_list()` moves it to a *worse* +388.78% (23,232,512 ->
  31,391,744 bytes) while delivering the entire query win (3,779 -> 1,336 blocks,
  11.259 -> 1.393 ms) that the later rebuild did not, and `fastupdate` alone
  changes bytes-per-tuple 2.8x; three hash builds over identical 200k rows give
  34.5293 / 38.0928 / 37.1917 bytes/row because `hashbuild` sizes buckets from
  `estimate_rel_size`; a grown hash index carried 87 unwritten zero pages
  (`pg_relation_size` 5,341,184 vs 4,628,480 bytes allocated per `stat`) and
  `pgstattuple()` refused it; hash's del10 "reclaim" is re-bucketing (768 -> 640
  buckets); BRIN's `pages_per_range` spreads the baseline 6.3x (3/4/19 pages at
  128/8/1) and BRIN churn cost 130 -> 487 query blocks at +0.00% reading; stale
  `reltuples` flips one GiST index between +84.71% and -7.64%; a partial index
  that really grew 80.4% read -5.02% against 44.59% reclaimable; and rebuild cost
  ran GiST 650.8 ms, SP-GiST 292.9, GIN 156.7, hash 120.3, BRIN 19.4. `COMMENT ON
  INDEX` mechanics were verified on the pin (literal-only grammar so a computed
  baseline needs dynamic SQL, `ShareUpdateExclusiveLock` on the index, owner-only
  writes, world-readable, `IS NULL`/`IS ''` both delete the row, preserved by
  plain REINDEX and moved to the new OID by REINDEX CONCURRENTLY, copied onto an
  empty clone by `LIKE INCLUDING ALL`). Eleven open questions were filed,
  including single-platform constants, the fixed 1,000-key GIN fixture, warm
  single-client query measurement, and GiST's non-deterministic builds. The test
  server was stopped and its data directory removed. The page is human-unverified
  and agent-unverified.

- 2026-08-12: removed the v12 question page Calibrating a COMMENT-Stored
  Bytes-per-Index-Row REINDEX Threshold in PostgreSQL 12 (unverified) at the
  user's request, together with its root-index and v12-landing-page entries and
  the closing `indexing` clause of the v12 coverage cell above that described
  it. No surviving page linked or cited it, so no `## Navigation` list changed.
  The three historical coverage notes below that referenced the page now name it
  as plain text. COMMENT-stored index-bloat screening in v12 now runs through
  Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in
  COMMENT in PostgreSQL 12 (unverified)
  and Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in
  PostgreSQL 12? (unverified),
  and core-SQL B-tree bloat measurement through
  [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md).
  The pin `45b88269a353ad93744772791feb6d01bc7e1e42` and every surviving page's
  citations and verification fields are unchanged.

- 2026-08-12: filed [Checking Whether an Index Needs a Rebuild to Enable
  Deduplication After pg_upgrade From PostgreSQL 12 to 17
  (unverified)](v17/questions/indexing/btree-deduplication-after-pg-upgrade.md)
  against unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10).
  Verdict: after a binary upgrade, every B-tree index carried over from the v12
  cluster is unable to deduplicate, and only a rebuild changes it. Deduplication
  safety is recorded once at build time in the metapage's `btm_allequalimage`;
  `_bt_upgrademetapage` refuses to set it ("Only a REINDEX can set this field"),
  `_bt_metaversion` and `bt_metap` both rely on it reading zero for indexes
  upgraded from 12, and no commit under `src/bin/pg_upgrade` has ever touched
  `allequalimage`. `btm_version` is 4 before and after, so version is not the
  discriminator. The filed core-SQL check reads byte 64 of block 0 with
  `pg_read_binary_file` + `get_byte` (offsets confirmed by compiling `offsetof`
  against the pin's own installed headers: contents at 24, `btm_version` at 28,
  `btm_allequalimage` at 64, v17 metapage `pd_lower` 72) and pairs it with a
  catalog mirror of `_bt_allequalimage`. Measured on three isolated 12.2 -> 17.10
  `pg_upgrade --copy` runs plus one ICU-enabled 17.10 cluster: 21/21 carried-over
  indexes read `pd_lower = 64`, version 4 and the flag false while every 17-built
  index read 72/true; the probe agreed with `pageinspect`'s `bt_metap()` in 75/75
  comparisons; all 21 index files were MD5-identical between the old and new data
  directories; the gate matched the engine's own `DEBUG1` verdict on 19 index
  shapes, the silent case being the `INCLUDE` index whose early `return false`
  skips the `elog`. `REINDEX INDEX CONCURRENTLY` reclaimed 69.1%, 69.0%, 69.0%,
  68.7%, 68.5%, 67.4% and 67.4% on the seven duplicate-heavy indexes, 0.0% on the
  unique and the empty one, and left the five not-equal-image indexes identical to
  the byte; a fresh `deduplicate_items = off` twin measured 22,519,808 bytes,
  exactly the carried-over index's size, against 6,963,200 with deduplication on.
  The savings curve over rows per key was 0.0% / 10.0% / 48.1% / 65.4% / 69.5% /
  67.4% at 1 / 2 / 5 / 20 / 100 / 1000. Also measured: the `deduplicate_items`
  trap (flag true, full 22,519,808 bytes, engine still logging "can safely use
  deduplication"), a carried-over index growing to 3.1x a fresh twin over the same
  200,000 duplicate inserts, `VACUUM FULL`/`CLUSTER` setting the flag while plain
  `VACUUM` does not, `GRANT EXECUTE ON FUNCTION pg_read_binary_file(...)` being
  required and `pg_read_server_files` being neither sufficient nor necessary,
  `relpages`/`reltuples` left at 0 for all 21 indexes until the first `ANALYZE`,
  and `pg_largeobject_loid_pn_index` as the single `pg_catalog` B-tree index the
  upgrade carries over. Nine open questions were filed, including the single-platform
  byte offsets and the unobserved standby/replay case. Two open questions on the v17
  core-SQL bloat page were updated to point at these measurements. All four test
  servers were stopped and their data directories removed, leaving only the scripts
  and captured output under `.wiki-runtime/tmp/dedup-upgrade/`. The page is
  human-unverified and agent-unverified.

- 2026-08-12: extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on
  PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with a
  proposed single B-tree bloat statement for servers 12 through 17, against
  unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10). The statement
  runs unchanged on every major in that range and credits deduplication only
  where the catalog proves the engine would: non-unique, `deduplicate_items` on,
  `indnatts = indnkeyatts`, a `pg_amproc` equal-image row at `amprocnum = 4` for
  every key opclass, and no key collation with `collisdeterministic = false`. Its
  posting-list arithmetic is derived from `_bt_load`'s
  `MAXALIGN_DOWN(BLCKSZ * 10 / 100) - sizeof(ItemIdData)` cap (812 bytes),
  `_bt_dedup_save_htid`/`_bt_form_posting`'s `MAXALIGN(base + n * 6)` tuple size,
  and `_bt_buildadd`'s `pgspc + last_truncextra < btps_full` rule, which together
  put 9 posting tuples of 808 bytes on a leaf and model 843 leaves against an
  849-block rebuild for 1,000,000 rows under one key. Key groups come from a
  mixture — the NULL run (NULLs deduplicate because `_bt_keep_natts_fast` treats
  two of them as equal), each `most_common_freqs` entry, then the remaining
  distinct values — and a negative `n_distinct` is credited only for whole-table
  indexes because the fraction is over the table's rows, not a partial
  predicate's subset. Measured on isolated 12.2, 14.23 and 17.10 servers built
  from their own pins (17.10 configured `--with-icu`) with identical fixture DDL
  and `CREATE INDEX CONCURRENTLY` ground truth: 68/68/72 indexes swept over
  133,677/94,910/103,056 blocks in 17.7 to 32.9 ms warm, 0/34/36 credited,
  `expected_blocks` identical to the v12 page's Method A in all 34 scored cells
  on 12.2, and within 5% of a rebuild on 25 of 33 cells on 14.23 and 25 of 35 on
  17.10 against Method A's 14 and 15. `i_dupdel` (10 keys, 90% of 1,000,000 rows
  deleted and vacuumed) reads 89.8% on 14.23 and 17.10 against a true 89.8%, and
  90.0% on 12.2 against a true 89.9%. The statement emits a second
  `bloat_pct_floor` column (one tuple per row) plus a `caveats` column, and
  alerting on the floor scores 4 true positives, 0 false positives and 1 false
  negative on every server, against 2 to 3 false positives for the point estimate
  alone: a healthy 28 MB nondeterministic-collation ICU index reads 88.2% without
  the new conjunct and 0.1% with it, a hot-value skew index whose `n_distinct`
  came back 81,281 against a true 750,001 reads 53.4% point / −20.9% floor and
  0.1% after `SET (n_distinct = -0.75)`, and a table doubled since its last
  `ANALYZE` reads 49.9% with the caveat `row-count sources disagree`. Ten new open
  questions were filed, including that majors 13, 15 and 16 were never run and
  the 5.5% to 8.2% posting-list packing loss that is measured but not modelled.
  All three servers were stopped and their data directories removed. The page
  remains human-unverified and agent-unverified.

- 2026-08-12: extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on
  PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with
  three follow-up sections answering whether that page's deduplication-aware
  sweep works for indexes on a v12 server as well as a v17 one, against unchanged
  pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10). Verdict: yes on both,
  because each v17-specific term is gated on a catalog fact a 12 server cannot
  produce - a `pg_amproc` equal-image row at `amprocnum = 4` (`_bt_allequalimage`
  is the engine's own test, and support numbers are bounded by the access method's
  `amsupport`), the `deduplicate_items` reloption, and the `-1` `reltuples`
  sentinel; the three constructs first ship in `REL_13_0`, `REL_13_0` and
  `REL_14_0` per this checkout's own history (`612a1ab7672`, `0d861bbb702`,
  `3d351d916b2`), none of them an ancestor of `REL_12_2`. Measured by running the
  identical statement on isolated 12.2 and 17.10 servers over the same DDL and
  data: 0 of 32 indexes credited on 12.2 with `expected_blocks` identical to the
  v12 page's Method A in every cell and no miss worse than 4 blocks, against 15 of
  34 credited on 17.10; a 10-distinct-key index with 90% of rows deleted and
  vacuumed reads 90.0% on 12.2 (rebuild 278 blocks) and 90.1% on 17.10 (rebuild
  87) against true bloat of 89.9% and 89.8%, where the uncorrected model would
  have said 67.5% on 17.10. 12.2 refuses the gate even deliberately
  (`ERROR: invalid function number 4, must be between 1 and 3`,
  `ERROR: unrecognized parameter "deduplicate_items"`). Two new findings: an
  `INCLUDE` index with a low-cardinality included column read 78.1% bloat on a
  0%-reclaimable 3853-block index on 17.10 because `_bt_allequalimage` refuses
  INCLUDE indexes outright while the sweep probes only key columns - adding
  `indnatts = indnkeyatts` to the gate makes that cell exact and moves no other
  cell - and on 12.2 the `-1` guard is dead code, so a stale-zero `reltuples`
  after `TRUNCATE` and reload reports 99.9% bloat on a healthy 825-block index
  where 17.10 reports `unmeasured`; a zero-plus-size rule for 12 and 13 servers
  was verified as filed. Both servers were stopped, the test databases dropped,
  and the 17.10 data directory removed. The page remains human-unverified and
  agent-unverified.

- 2026-08-11: rebuilt the v12 COMMENT-stored index-bloat page around the index's
  own row count and renamed it to Calibrating a COMMENT-Stored
  Bytes-per-Index-Row REINDEX Threshold in PostgreSQL 12 (unverified),
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). The approved follow-up prompt is "replace live rows by the
  index `pg_class.reltuples`", so the metric is now
  `pg_relation_size(index) / index reltuples` everywhere and every measurement
  was re-run on a fresh isolated 12.2 server. New source model: three writers
  set an index's `reltuples` and they disagree about units - the AM's build
  result through `index_update_stats()`, `ANALYZE`'s
  `ceil(tupleFract * totalrows)`, and `lazy_cleanup_index()`'s
  `num_index_tuples` gated on `estimated_count`; `ginbuild()` counts extracted
  entries while `ginvacuumcleanup()` substitutes the heap count under an
  explicit XXX, `brinbuild()` counts range summary tuples and
  `brinvacuumcleanup()` recounts them with `include_partial = false`, and
  `btvacuumcleanup()`/`hashvacuumcleanup()` can return NULL so no update
  happens at all. Measured over a new 96-cell matrix (12 workloads x 8 indexes,
  seven capture phases each): the swap is a no-op on ordinary indexes (drift
  identical to four decimals in all 84 non-BRIN cells; index and table
  `reltuples` equal in all 192 plain-`ANALYZE` observations), fixes partial
  indexes (10 of 10 defined cells against the table denominator's 8 of 13 over
  13 partial cells), and disqualifies GIN and BRIN. `drift >= 1.30` scores
  42/1/4/49 (94.8%) over 96 cells and 42/0/4/38 (95.2%) excluding BRIN; the four
  misses are all in-domain churn, where the fork stops growing while a rebuild
  reclaims 26.28% to 56.15%. Named measurements: a 20-keys-per-row GIN index
  recorded 4,200,000 rows for a 400,000-row table, making a post-rebuild
  baseline read drift 10.50 with no bloat; one 24,576-byte BRIN index reported
  0.0123 and 279.2727 bytes per index row at different times; a
  `VACUUM (VERBOSE, ANALYZE)` printed no index line while a partial index's own
  population grew 50%, leaving a sampled 150,710 in place, and the
  `vacuum_cleanup_index_scale_factor = 0` reloption restored an exact 310,000;
  `reltuples::numeric` rounded 20,000,020 to 20,000,000 through
  `float4_numeric()`'s six significant digits; a GIN pending list moved the
  ratio down 32% and then up 74% while its probe ran 674x slower (4.043 ms and
  497 blocks against 0.006 ms and 6); deleted-page bloat gave 28.50% reclaim
  with no query change but 413.33 of planner cost, against low-density bloat at
  49.64% reclaim, half the scan blocks and no wall-clock gain; and a partial
  index's predicate column turned 1,800,000 HOT updates into 0 and quadrupled a
  sibling index while both partial indexes read drift 1.0000. New `wiki_bpr_v3`
  capture and detection SQL with an access-method allowlist, a stored-shape
  check and an empty-index rule was executed on the pin, along with a five-step
  runbook using `REINDEX INDEX CONCURRENTLY`. The isolated server was stopped
  and its data directory, build tree, and fixtures removed. The page is
  human-unverified and agent-unverified.

- 2026-08-11: extended the page then titled Calibrating a COMMENT-Stored
  Bytes-per-Live-Row REINDEX Threshold in PostgreSQL 12, since renamed to
  Calibrating a COMMENT-Stored Bytes-per-Index-Row REINDEX Threshold in
  PostgreSQL 12 (unverified),
  with a comprehensive B-tree partial-index section, against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2). The filed
  follow-up prompt asks for that section and for the earlier "the denominator is
  wrong by construction" material to be folded into it. Verdict: keep the +30%
  B-tree trigger but divide by the *index's* own `pg_class.reltuples`. On 13
  partial-index cells measured on a fresh isolated 12.2 server (1,000,000-row
  tables, a 10% `st = 'open'` predicate and a 10% `done_at IS NULL` predicate,
  `REINDEX` size delta as ground truth) the table denominator scored 8 of 13
  while the index denominator scored 11 of 11 cells where it is defined, with a
  clean band from 1.0243 benign to 1.3795 harmful; the two undefined cells are
  the two indexes whose own `reltuples` is 0, which a separate empty-index rule
  catches. The swap is free for non-partial indexes because
  `compute_index_stats()` leaves their `tupleFract` at 1.0, verified as an exact
  match to the table count in all 16 `ANALYZE` runs. Table-denominator failures
  measured: 5.2346 with 0% reclaimable on growth inside the predicate, 2.0000
  with 0% reclaimable on deletion outside it, 1.0000 with 99.64% reclaimable on
  a drained queue index, and 1.0526 with 49.64% reclaimable on scattered
  deletion. New source findings: a partial index's predicate columns enter the
  HOT-blocking bitmap, so two identical tables running identical updates
  recorded 0 versus 1,800,000 HOT updates and a 49.96%-reclaimable sibling
  index while both partial indexes reported drift 1.0000; `_bt_vacuum_needs_cleanup()`
  gates the cleanup scan on *heap* growth, so a `VACUUM` skipped an index whose
  own population had grown 50% because the heap grew 5%; `VACUUM (ANALYZE)`
  writes the exact index count that plain `ANALYZE` samples (102,634 and 99,900
  against exactly 100,000); and `get_relation_info()` estimates a partial
  index's row count instead of pinning it. Two bloat shapes were separated with
  `pgstatindex`: deleted-page bloat (66.59% reclaim, 546 deleted pages, density
  89.18, 1.1% fewer scan blocks, but a 2200.13 planner-cost drop equal to 550
  unread pages x `random_page_cost` 4.0) and low-density bloat (49.64% reclaim,
  density 45.06 to 89.83, 49.6% fewer scan blocks). The plain-`ANALYZE` noise
  floor runs from -3.87%/+2.97% at a 10% predicate to `reltuples = 0` on four of
  six runs at 0.001%. New `wiki_bpr_v2` capture and detection SQL, executed on
  the pin along with a five-step runbook using `REINDEX INDEX CONCURRENTLY`, is
  filed. The isolated server was stopped and its data directory, build tree, and
  fixtures removed. The page remains human-unverified and agent-unverified.

- 2026-08-11: filed the page then titled Calibrating a COMMENT-Stored
  Bytes-per-Live-Row REINDEX Threshold in PostgreSQL 12, since renamed to
  Calibrating a COMMENT-Stored Bytes-per-Index-Row REINDEX Threshold in
  PostgreSQL 12 (unverified),
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). The page stores `pg_relation_size(index)` divided by the
  table's `pg_class.reltuples` in the index `COMMENT` and calibrates the drift
  trigger against `REINDEX` ground truth on an isolated server built from the
  pin, over 88 cells (11 workloads x 8 indexes, one index per access method
  including a second GIN index with `fastupdate = off`) plus twelve targeted
  fixtures. Calibrated triggers are +30% for B-tree, GiST, and hash, +40% for
  GIN after the pending list is flushed, and +60% for contrib `bloom`; SP-GiST
  is not separable because its benign 10x-growth drift of 2.2041 exceeds its
  harmful minimum of 2.0000, and BRIN must not use the signal at all. A single
  +30% to +40% rule scored 43 true positives, 5 false positives, 0 false
  negatives, and 40 true negatives, and was exact on the 66 cells that exclude
  BRIN and SP-GiST. Source findings: no v12 index access method truncates its
  own main fork (SP-GiST's truncation is inside `#ifdef NOT_USED`), so VACUUM
  only records reusable pages through `RecordFreeIndexPage()` and index bytes
  are a high-water mark; hash allocates a whole splitpoint batch and counts the
  resulting file hole; `gistchoose()` breaks ties with `random()`. Measured
  failure modes: drift is exactly 2.0000 for all eight indexes in every delete
  workload while reclaim spans 0.00% to 54.19%; a GIN pending list reads 4.1231
  and `gin_clean_pending_list()` raises it to 4.6182 while removing a 12.0x
  query slowdown; 20-keys-per-row GIN rows read 6.3176 and a rebuild grows the
  index 16.85%; an unsummarized BRIN range set reads 0.1000 while a probe costs
  49,961 blocks against 1,411; a partial index reads 0.3640 with 65.66%
  reclaimable; `reltuples` and `n_live_tup` differ 2x after an unvacuumed delete
  and `pg_stat_reset()` zeroes the latter; 400,000 HOT updates moved no index
  byte; and a 49.96% B-tree space reclaim bought 50% fewer index-only-scan
  blocks but 6% to 10% less time and nothing on a point lookup. Capture and
  detection SQL plus a five-step runbook were executed on the pin; the isolated
  server was stopped and its data directory, build tree, and fixtures removed.
  The page is human-unverified and agent-unverified.

- 2026-08-11: removed the v12 question page Detecting Index Bloat with
  COMMENT-Stored Bytes per Tuple in PostgreSQL 12 (unverified) at the user's
  request, together with its root-index and v12-landing-page entries and the
  three inbound `## Navigation` links on
  Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in
  PostgreSQL 12? (unverified),
  [Physical Index Statistics, Tuple Counts, and Bytes per Tuple in PostgreSQL 12 (unverified)](v12/questions/indexing/physical-index-statistics-tuple-counts-and-bytes.md),
  and [Proposing and Testing a fillfactor-Corrected pgstattuple_approx Metric for Table Heap Bloat in PostgreSQL 12 (unverified)](v12/questions/storage-and-vacuum/pgstattuple-approx-heap-bloat.md).
  The v12 coverage cell above never described the page, so it is unchanged.
  COMMENT-stored index-bloat screening in v12 now runs through Detecting Bloat
  in All Index Types by Storing an Index/Heap Size Ratio in COMMENT in
  PostgreSQL 12 (unverified)
  and the GIN DML-counter page named above. The pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` and every surviving page's
  citations and verification fields are unchanged.

- 2026-08-11: reframed [Proposing and Testing a fillfactor-Corrected
  pgstattuple_approx Metric for Table Heap Bloat in PostgreSQL 12
  (unverified)](v12/questions/storage-and-vacuum/pgstattuple-approx-heap-bloat.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). The filed `## Question` now asks for the metric corrected
  for the table's `fillfactor`, and the page is rebuilt around that correction.
  New source derivation: the `RELOPT_KIND_HEAP` reloption with its 10-to-100
  bounds and `ShareUpdateExclusiveLock`, `RelationGetTargetPageFreeSpace()` =
  `BLCKSZ * (100 - fillfactor) / 100` with its integer truncation, the four
  heap callers of that reserve, `make_new_heap()` reloption inheritance, and
  the documented refusals on partitioned and TOAST tables. New measurements on
  a fresh isolated 12.2 server: a seven-point fillfactor sweep where the
  uncorrected signal reads 0.44 to 91.02 on unbloated tables while the
  corrected one reads 0.02 to 1.79 except 10.16 at `fillfactor = 10`; a
  closed-form residual model from the page-close condition that matches all
  seven within 0.05 points; three bloated fillfactor fixtures at −1.51, −0.78,
  and −1.42 corrected against +14.45 and +34.58 raw; `VACUUM FULL` and
  `CLUSTER` agreeing at 1,250 blocks and 40 rows per page while
  `ALTER COLUMN TYPE` gives 1,316 at a wider row; a reloption lowered after
  load that makes a rewrite grow the table 44.93%, reported as −42.23 only
  when the metric is not clamped at zero; the reloption bound, partitioned,
  TOAST, materialized-view, and lock-level boundary results; and three
  `ANALYZE` runs moving `approx_tuple_count` by −379, +697, and −279 rows,
  which replaces the earlier 18-row live-count claim. The nine-fixture
  accuracy matrix, the emptied-page reconciliation, and the step 1 to 3 worked
  example were re-run and reproduced their filed values, with `d_bloat`'s
  ground truth now recorded as 59.99 and its error as −1.86. The step 2 query
  drops its `greatest(..., 0)` clamp and step 1 gains a `fillfactor` column.
  The page is human-unverified and agent-unverified.

- 2026-08-11: filed [Proposing and Testing pgstattuple_approx for Table Heap
  Bloat in PostgreSQL 12
  (unverified)](v12/questions/storage-and-vacuum/pgstattuple-approx-heap-bloat.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2), the first page in the v12 `storage-and-vacuum` category.
  The page proposes a three-step procedure — catalog screen, then
  `pgstattuple_approx` with a `fillfactor` correction, then confirmation before
  a rewrite — and tests it on an isolated server built from the pinned
  checkout with `autovacuum = off`, using `VACUUM FULL` size reduction as
  ground truth. `pgstattuple_approx` cost 6.9 ms and 42 physical reads against
  `pgstattuple`'s 484.8 ms and 137,932 reads on a 1.08 GB all-visible table,
  and 216.7 ms against 470.8 ms with no visibility map at all. Over nine
  fixtures the corrected estimate ran −3.11 to +1.79 points against ground
  truth on the eight ordinary row shapes; the raw `free + dead` signal instead
  produced a 31.25-point `fillfactor` false positive. Two false negatives are
  quantified: dropped-column bytes (85.40% reclaimable, reported 1.18%) and
  v12's retained line-pointer arrays (2 to 7 points). `pageinspect` and
  `pg_freespacemap` reconciled `approx_tuple_len` to `sum(8192 - fsm)` byte for
  byte and showed emptied pages keeping 58 line pointers and charging 288
  phantom bytes each. A catalog-only estimator was exact on all nine
  well-behaved fixtures and wrong by 21.32, 79.99, and −5698.78 points on
  alignment-padding, stale-statistics, and never-analyzed tables. Also filed:
  the exact `reltuples` dependency of `approx_tuple_count` (a forged
  `reltuples = 7` is returned verbatim when every page is skipped), integer
  truncation in `scanned_percent`, a 15.69-point phantom dead-tuple reading
  during an uncommitted bulk insert, dead-but-not-reclaimable rows under an old
  snapshot, the relation-kind matrix including the refused TOAST relation, and
  the `pg_stat_scan_tables` privilege boundary. Test objects were dropped and
  the isolated server was stopped. The page is human-unverified and
  agent-unverified.

- 2026-08-11: reviewed [Physical Index Statistics, Tuple Counts, and Bytes per
  Tuple in PostgreSQL 12
  (unverified)](v12/questions/indexing/physical-index-statistics-tuple-counts-and-bytes.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Every claim and all 354 source citations were rechecked.
  Corrections: `pgstattuple.tuple_len` includes each item's `MAXALIGN` padding
  because B-tree and hash pass an already-rounded `IndexTupleSize()`, and the
  three `bigint` averages now cast to `numeric`; a bare `block_size` identifier
  became `current_setting('block_size')::bigint`;
  `btm_last_cleanup_num_heap_tuples` is written by `btbulkdelete()` as well as
  cleanup; the `ANALYZE` skip guard is `!inh && !(options & VACOPT_VACUUM)`;
  `PageIndexTupleDeleteNoCompact()` returns the tuple bytes to free space and
  retains only the line pointer; `get_raw_page()` requires superuser and
  rejects partitioned indexes; `brinbulkdelete()` still returns a zeroed
  result struct; `pg_am` has four columns; and the `pgstattuple` regression
  file does call generic `pgstattuple()` on a partitioned-index root as an
  expected failure. Two new Open Questions: the BRIN `revmapNumPages =
  lastRevmapPage - 1` off-by-one quantified against `brincostestimate()`, and
  `bufpage.h`'s "unused in index pages" comment versus GIN's `pd_prune_xid`
  reuse. Roughly 40 citations were repointed or rebounded, including the BRIN
  revmap reader, the GiST VACUUM leaf count, `pg_relpages_v1_5`, and the
  `pgstattuple` regression file's true 119-line extent. The page remains
  human-unverified and agent-unverified: this review re-read source only and
  did not re-execute the page's SQL on a live 12.2 server.

- 2026-08-11: filed [Physical Index Statistics, Tuple Counts, and Bytes per
  Tuple in PostgreSQL 12
  (unverified)](v12/questions/indexing/physical-index-statistics-tuple-counts-and-bytes.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). The page inventories shared catalog and relation-size
  values and every core index access method's persistent physical metadata,
  with contrib Bloom as the packaged-extension boundary. It distinguishes
  build-, `ANALYZE`-, and `VACUUM`-written `reltuples` meanings from physical
  page-item counts, gives guarded formulas for allocated bytes per catalog
  unit or scanned data item, and documents why GIN keys/postings and BRIN
  summaries cannot be treated as one tuple per heap row. The page remains
  human-unverified and agent-unverified.

- 2026-08-11: filed Can COMMENT-Stored Table DML Counters Trigger GIN
  REINDEX at 40% in PostgreSQL 12? (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). The source audit rejects an automatic 40% GIN rebuild
  trigger because the proposed counters are table-wide attempted actions, HOT
  updates require no index entries, partial predicates can skip writes, deletes
  reach GIN only through VACUUM, and GIN operator classes choose an arbitrary
  row-to-key cardinality. Exact-pin tests produced a 40% HOT-update false
  positive with 0.00% rebuild savings and a 1% high-cardinality-delete false
  negative with 99.13% savings. Tested capture and detection SQL stores a
  HOT-adjusted inspection baseline plus table-OID, fixed-row, and
  statistics-reset guards in a reserved index comment, and labels 40% as
  `inspect GIN; do not auto-reindex`. The page remains human-unverified and
  agent-unverified pending local workload calibration, a partial-index design,
  and resolution of a v12 GIN-pending-cleanup documentation discrepancy.

- 2026-08-07: filed Detecting Index Bloat with COMMENT-Stored Bytes per Tuple
  in PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). The proposal stores actual main-fork bytes per estimated
  indexed heap row in a versioned index comment, treats `1.0` as the normalized
  baseline, and reports `1.4` as an investigation threshold rather than an
  automatic rebuild verdict. Source review established that plain table
  `ANALYZE` must run after creation and before each comparison because GIN's
  immediate build count is extracted entries while BRIN's is range summaries;
  plain `ANALYZE` rewrites both to an estimated indexed-row denominator. Tested
  capture, detection, and guarded post-rebuild recapture SQL preserves human
  comments and filters non-physical or unsuitable index states. An isolated
  exact-pin run covered B-tree, hash, GiST, SP-GiST, GIN, BRIN, and contrib
  Bloom. BRIN and GIN false-positive fixtures reached drift `5.0` and `137.2`
  while `REINDEX` reclaimed `0.0%`, disproving a universal all-AM threshold.
  The page remains human-unverified and agent-unverified pending production
  calibration and third-party-AM review.

- 2026-08-06: filed Detecting Bloat in All Index Types by Storing an
  Index/Heap Size Ratio in COMMENT in PostgreSQL 17 (unverified)
  against unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa`
  (PostgreSQL 17.10), copying the v12 question of the same name and reviewing it
  for v17. Prompt hygiene applied first; the user chose silently corrected
  wording, the copy request itself as the filed `## Question`, a full
  re-measurement on v17, and a since-v12 history section. No v12 number was
  carried over: an isolated 17.10 server (previous pin) built from the pinned checkout with
  `autovacuum = off` reproduced every fixture. A 49-cell matrix of seven
  workloads times seven access methods with `REINDEX TABLE` ground truth scored
  `drift >= 1.40` at 13 true and 3 false positives against `index > heap` at 6
  and 2; the 19 cells in the drift 0.90-1.10 band span -8.4% to 80.0%
  reclaimable. Two findings are new relative to the v12 page and are traced to
  source. First, `hashbuild` sizes the initial bucket array from
  `estimate_rel_size`, so identical 200,000-row data produced 923 blocks on a
  narrow heap, 5,122 on a wide never-analyzed one, and 822 when `ANALYZE` ran
  first — a 6.2x inflated day-zero baseline, which added an `ANALYZE`-first rule
  to Capture discipline. Second, v14's same-VACUUM B-tree page recycling
  (`9dd963ae253`) is present but did not fire on an idle server: the
  delete-and-reload cycle still left B-tree +99.27%, GiST +89.47% and GIN
  +82.72% after one VACUUM, while repeating it against a concurrent
  `txid_current()` consumer populated the FSM and ended at 745 blocks instead of
  1,098, matching the condition stated in `_bt_pendingfsm_finalize`. SP-GiST is
  newly disqualified in these fixtures (fresh ratio drifts 2.2787 over 16x
  growth at 0.0% reclaimable). The since-v12 section attributes nine changes to
  their first release tags, including the 17.1 fix `fee8cb94734`, which stops
  parent index comments propagating onto partition children; the measured v17
  outcome is that an `ALTER COLUMN TYPE` on a partitioned table now leaves child
  baselines empty rather than silently overwriting them with the parent's, so
  the audit query detects the loss. Also filed: a day-zero sweep whose implied
  `index > heap` thresholds span 0.94 to 4444.67, the GIN baseline-1.004900
  fixture, RIC-versus-plain-`REINDEX` byte equality on all seven access methods,
  the four RIC refusal cases, an interrupted-RIC `_ccnew` with no baseline, a
  nine-operation comment durability matrix, lock/ownership/read-visibility
  probes, 300-index survey timings against the `relpages` variant, and three
  verified read-only queries. Test objects were dropped and the isolated server
  was stopped. The page is filed with human `verified: false`, the visible
  `(unverified)` title, and `verified_by_agent: not yet`.
- 2026-08-06: added a decision-rule comparison section to Detecting Bloat in
  All Index Types by Storing an Index/Heap Size Ratio in COMMENT in
  PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied first; the user chose corrected
  wording with no prompt note, the absolute reading `current ratio > 1.0` on
  main forks, full exact-pin measurements, and all seven access methods.
  A fresh isolated 12.2 server with `autovacuum = off` ran a 49-cell matrix of
  seven workloads times seven access methods with `REINDEX TABLE` ground truth:
  `drift >= 1.40` scored 13 true and 1 false positive, `index > heap` scored 6
  and 2. The section proves `index_bytes > heap_bytes` is the drift rule
  `drift > 1 / baseline_ratio`, whose per-index threshold measured 0.94 to 1282
  on one table, a 1367x spread set by heap row width. A day-zero sweep held
  every index's block count constant across heaps of 2,858, 3,847 and 13,334
  blocks while the GIN ratio fell 1.435 -> 1.066 -> 0.308; a fresh-B-tree sweep
  from 0 to 1,000,000 rows fell 2.000 -> 0.266. The question's own premise was
  reproduced on a GIN index with baseline 1.004900, which the absolute rule
  condemns at 0.0% reclaimable the moment it is built and at every later step,
  while drift stayed quiet until a rebuild was worth 38.9%. At the 1.40
  crossing a rebuild reclaimed 25.2-38.9%, GiST stalled at 1.399 with 35.4%
  reclaimable, and the 22 cells near drift 1.00 span -8.5% to 80.0%. Threshold
  sweeps from 1.10 to 1.50 score identically. Added a verified read-only
  two-rule comparison query, five Evidence Map rows on tuple layout, fill
  factors, the empty-index metapage and `reindex_index` locks, and six new
  Open Questions. A further section answers the maintenance process rebuilding
  with `REINDEX INDEX CONCURRENTLY`: on a second isolated server, two
  byte-identical churned 200,000-row tables produced identical rebuilt files
  under RIC and plain `REINDEX INDEX` on all seven access methods, so no score
  moves, and all seven stored baselines survived the swap while every index OID
  changed. The costs that remain are peak storage of old plus new until
  phase 6, a `ShareUpdateExclusiveLock` that conflicts with lazy VACUUM's own,
  four `WaitForLockersMultiple` points, and the transaction-block ban. Four
  measured loop-breakers: a direct RIC on an exclusion-constraint index errors,
  `REINDEX TABLE CONCURRENTLY` skips it with only a `WARNING` so its drift
  climbs forever, a temporary relation silently falls back to the
  non-concurrent path, and a `statement_timeout`-interrupted RIC left a
  baseline-less `_ccnew` that the page's audit query flags. Added six more
  Evidence Map rows, two Context Reviewed bullets, five Open Questions, and
  twelve Source References. Test objects were dropped and both isolated
  servers were stopped. Human `verified: false`, the visible `(unverified)`
  title, and
  `verified_by_agent: not yet` are unchanged because this was a scoped
  follow-up rather than a full page re-audit.
- 2026-08-06: reviewed Detecting Bloat in All Index Types by Storing an
  Index/Heap Size Ratio in COMMENT in PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`.
  Reclassified the ratio as an allocation-growth screen; separated index growth,
  rebuild-reclaimable fraction, and excess over rebuilt size; corrected ordinary
  VACUUM, BRIN revmap, SP-GiST redirect-XID, GIN pending-list, HOT, lock,
  `relpages`, dump/restore, and `CREATE TABLE ... LIKE INCLUDING ALL` claims;
  and added the catalog/generated-header boundary. Replaced the unsafe audit cast
  and zero-heap filter with exact-pin-tested queries. Reran the 200k-row cycle at
  one, two, and three post-delete VACUUM passes: GIN is 4,101 -> 7,482 blocks
  (+82.44%) with 4,100 entry pages, and SP-GiST is +0.83% in every arm. Updated
  both indexes and the v12 coverage row. Human `verified: false`, the visible
  `(unverified)` title, and `verified_by_agent: not yet` remain unchanged because
  the non-cycle historical measurement tables were not all reproduced.
- 2026-08-06: added a delete-and-reload cycle test section to Detecting Bloat
  in All Index Types by Storing an Index/Heap Size Ratio in COMMENT in
  PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied first; the user chose silently
  corrected wording, all seven access methods, fresh ascending reload keys, and
  results plus a runnable script. A second isolated `autovacuum = off` 12.2
  server carried one 200k-row table with all seven indexes through capture,
  `VACUUM`, full delete, `VACUUM`, and a 200k-row reload. The heap returned to
  exactly 3,847 blocks, so ratio drift equalled raw index growth digit for digit
  on all seven access methods and equalled `REINDEX` ground truth on six; hash's
  0.86% gap resolved to 7 overflow pages at an identical `maxbucket = 767`. One
  post-delete `VACUUM` left B-tree +99.27%, GiST +89.47% and GIN +82.72%;
  two- and three-`VACUUM` arms plus an independent replicate split the access
  methods along the `RecentGlobalXmin` recyclability gate, and `gin_metapage_info`
  showed 600,000 entries still on 4,101 entry pages after three passes. The
  published script was re-run standalone on a fresh database and reproduced every
  filed number. Test objects were dropped and the isolated server was stopped.
  `verified_by_agent` stays `not yet` because this was a scoped addition rather
  than a full page re-audit; human `verified: false` and the visible
  `(unverified)` title are unchanged.
- 2026-08-06: filed Detecting Bloat in All Index Types by Storing an
  Index/Heap Size Ratio in COMMENT in PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied first; the user chose corrected
  wording, a bare-ratio comment payload, source plus exact-pin tests, and
  read-only filed SQL. An isolated 12.2 server built from the pinned checkout
  measured seven access methods (btree, hash, gist, spgist, gin, brin, contrib
  bloom) across a three-scale fresh-ratio invariance sweep, a seven-point
  hash/B-tree sawtooth sweep, and six workload fixtures with `REINDEX` and
  `VACUUM FULL` ground truth, plus a nine-operation COMMENT durability matrix,
  lock/ownership/read-visibility probes, and 300-index survey timings. Test
  objects were dropped and the isolated server was stopped. The page is filed
  with human `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet`.
- 2026-08-06: removed the v17 question page How a GIN Index Becomes Bloated in
  PostgreSQL 17, and How to Measure It (unverified) at the user's request,
  together with its root-index, v17-landing-page, and coverage-cell entries and
  both inbound references on [Planner Penalties for Bloated Indexes in
  PostgreSQL 17 (unverified)](v17/questions/query-planning/bloated-indexes-query-planner.md):
  the `## Related Pages` entry, and one prose cross-reference that now points at
  that page's own follow-up section on when a GIN index is discarded and a
  B-tree used instead. With the v12 page of the same name removed earlier the
  same day, no version carries a dedicated GIN-bloat page; v17 GIN behavior at
  the planner level remains on that planner page, and the GIN contrib surface
  remains in [PostgreSQL 17 Contrib Extensions
  (unverified)](v17/questions/server-administration/contrib-extensions.md). The
  pin `54eeefaedbee0385529f3edf321bb99e49232aaa` and every surviving page's
  citations and verification fields are unchanged.
- 2026-08-06: removed every remaining `pgstatindex` sampling question page at
  the user's request: Proposing a Sampling pgstatindex Variant for PostgreSQL 17
  (unverified), Proposing a Sampling pgstatindex Variant for PostgreSQL 18
  (unverified), and Why pgstatindex Cannot Use pgstattuple_approx-Style
  Approximation in PostgreSQL 18 (unverified). Removed their root-index and
  version-landing-page entries, the one inbound `## Navigation` link on
  How a GIN Index Becomes Bloated in PostgreSQL 17, and How to Measure It
  (unverified), and the one inbound
  `## Related Pages` link on [Planner Penalties for Bloated Indexes in
  PostgreSQL 17 (unverified)](v17/questions/query-planning/bloated-indexes-query-planner.md).
  Also stripped the sampling clauses from the v17 and v18 coverage cells above,
  so no version now advertises a sampling `pgstatindex` page. With the v12 pages
  removed earlier the same day, the wiki carries no sampling `pgstatindex`
  coverage at all. v18 keeps [Computing and Storing avg_leaf_density During
  (Auto)VACUUM of a B-Tree Index
  (unverified)](v18/questions/indexing/avg-leaf-density-during-vacuum.md), and
  v12 keeps [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12
  (unverified)](v12/questions/indexing/how-pgstatindex-calculates-information.md).
  The pins `54eeefaedbee0385529f3edf321bb99e49232aaa` and
  `6cb307251c5c6261286c1566496920976640108e` and every surviving page's
  citations and verification fields are unchanged.
- 2026-08-06: removed the v12 question page Proposing a Sampling pgstatindex
  Variant for PostgreSQL 12 (unverified) at the user's request, together with
  its root-index, v12-landing-page, and coverage-summary entries and the one
  inbound `## Navigation` link on
  [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](v12/questions/indexing/how-pgstatindex-calculates-information.md).
  v12 now carries no sampling `pgstatindex` proposal. The exact full-scan
  function stays on that page, and contrib-free bloat estimation stays in
  [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md).
  The v17 and v18 sampling `pgstatindex` pages are unaffected. The pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` and every surviving page's
  citations and verification fields are unchanged.
- 2026-08-06: removed the v12 question page Proposing a Sampling pgstatginindex
  Variant for PostgreSQL 12 (unverified) at the user's request, together with
  its root-index, v12-landing-page, and coverage-summary entries. No surviving
  page linked to it, so no navigation section changed. v12 now has no GIN
  page-accounting page; GIN behavior at the planner level remains in
  [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/query-planning/bloated-indexes-query-planner.md).
  The B-tree page
  Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)
  and the v17/v18 sampling `pgstatindex` pages are unaffected. The pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` and every surviving page's
  citations and verification fields are unchanged.
- 2026-08-06: removed the v12 question page How a GIN Index Becomes Bloated in
  PostgreSQL 12, and How to Measure It (unverified) at the user's request,
  together with its root-index, v12-landing-page, coverage-summary, and
  cross-page navigation entries. v12 GIN coverage now runs through
  Proposing a Sampling pgstatginindex Variant for PostgreSQL 12 (unverified),
  with GIN-versus-B-tree planner behavior in
  [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/query-planning/bloated-indexes-query-planner.md).
  The v17 page How a GIN Index Becomes Bloated in PostgreSQL 17, and How to
  Measure It (unverified) is unaffected. The pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` and every surviving page's
  citations and verification fields are unchanged.
- 2026-08-06: filed every question page under a category directory. All 54
  question pages moved from `wiki/vNN/questions/<page>.md` to
  `wiki/vNN/questions/<category>/<page>.md`, across six closed,
  version-identical categories: `query-planning`, `indexing`,
  `storage-and-vacuum`, `replication-and-wal`, `observability`, and
  `server-administration`. The same question basename takes the same category
  in every version, so cross-version coverage comparison is a directory diff.
  Page-relative links on this page were rewritten to match. Coverage, pins,
  citation targets, and verification fields are unchanged; see
  `MANDATORY Question Categories` in `AGENTS.md`.
- 2026-08-06: removed the v12 question page A Heuristic to Detect B-Tree Index
  Bloat in PostgreSQL 12 (unverified) at the user's request, together with its
  root-index, v12-landing-page, and cross-page navigation entries. v12 B-tree
  bloat coverage now runs through [Measuring B-Tree Index Bloat With Core SQL
  Only in PostgreSQL 12
  (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md), [How
  pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12
  (unverified)](v12/questions/indexing/how-pgstatindex-calculates-information.md), and
  [Planner Penalties for Bloated Indexes in PostgreSQL 12
  (unverified)](v12/questions/query-planning/bloated-indexes-query-planner.md). The pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` and every surviving page's
  citations and verification fields are unchanged.
- 2026-08-06: answered a filed follow-up on [Measuring B-Tree Index Bloat With
  Core SQL Only in PostgreSQL 12
  (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
  Prompt hygiene applied first; the user chose corrected wording, the same page,
  bloat-relevant `pgstatindex` columns only, and a wide matrix with repeats.
  Built a second isolated 12.2 server carrying 9 bloat types (fresh, scattered
  delete, range delete, random-order insert, all-duplicate, vacuumed churn,
  unvacuumed churn, `LP_DEAD`, stale-catalog) at 200k/500k/1000k rows, each
  table carrying a non-partial and a partial index over the same key, for 54
  indexes; delete patterns use moduli 7 and 11 against a `id % 5 = 0` predicate
  so the partial index loses the same proportion of entries. Against a real
  `CREATE INDEX CONCURRENTLY` rebuild the filed catalog model was exact in 39 of
  54 cells, within 5 blocks in 47, within 0.8 percentage points in 48, and
  wrong only on 6 cells. The index-only-scan census reproduced
  `pgstatindex.leaf_pages` exactly in all 36 cells that passed the
  `Heap Fetches: 0` precondition (partial and non-partial alike),
  `avg_leaf_density` to within 0.04-0.15 points, and `blocks - leaf_est` matched
  `internal + deleted + half-dead + metapage` exactly in all 36, recovering 2330
  deleted pages within one page. Scored head to head as rebuild-size predictors,
  `ceil(leaf_pages * avg_leaf_density / fillfactor) + internal_pages + 1` lost to
  the core-SQL model in 16 of 18 type/kind cells, reaching 994.4% error on the
  `LP_DEAD` fixture where `pgstatindex` reports a healthy-looking 90.06 density
  on a 2745-block index that rebuilds to 251 blocks, because `PageGetFreeSpace`
  counts dead and `LP_DEAD` entries as used. Isolated the one large model
  failure to partial indexes on tables deleted without VACUUM or ANALYZE (510
  blocks, 92.6 points, reporting 0.0% to −2.0% bloat against a true 89.3-90.6%),
  because `pg_stat_all_tables.n_live_tup` counts the whole table and cannot
  substitute for a partial index's `reltuples`; a plain `ANALYZE` cut that error
  to at most one block, the residual being ANALYZE's sampled
  `tupleFract * totalrows` (18,337 against a true 18,181). All 97 citations were
  machine-audited with 0 broken ranges. Test objects were dropped, the isolated
  server was stopped, and human `verified: false`, the visible `(unverified)`
  title, and `verified_by_agent: not yet` are unchanged because this was a
  scoped follow-up.
- 2026-08-06: filed [Measuring B-Tree Index Bloat With Core SQL Only in
  PostgreSQL 12
  (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
  Prompt hygiene applied first; the user chose corrected wording, a new question
  page, source plus exact-pin tests, and permission to include write/DDL rebuild
  probes. Established from source that core v12 exposes no SQL-callable page or
  FSM reader at all and that `pgstattuple`, `pageinspect`, `pg_freespacemap`,
  and `amcheck` all default to `superuser = true` because their control files
  omit the field, then derived a closed form for the size a rebuild would
  produce: `_bt_pagestate` sets the leaf threshold to
  `BLCKSZ * (100 - fillfactor) / 100`, `_bt_blnewpage` pre-reserves the high-key
  line pointer, and `_bt_buildadd` closes a page when `PageGetFreeSpace()` falls
  below the threshold, giving
  `tuples_per_leaf = floor((BLCKSZ - 48 - floor(BLCKSZ * (100 - ff) / 100)) / (MAXALIGN(hoff + data) + 4))`.
  One isolated 12.2 server with 15 fixtures priced four core-only methods
  against an actual `CREATE INDEX CONCURRENTLY` rebuild: the 26.7 ms catalog
  sweep matched the rebuilt block count exactly on 10 of 14 indexes, within 2
  blocks on 3 more, and by −4.6% on a 2-to-81-character text key, which a
  `pg_column_size` measurement (57.000 bytes over a full scan, 56.893 from a 1%
  `BERNOULLI` sample, against the catalog-derived 60) corrected to +0.32%. The
  `EXPLAIN (ANALYZE, BUFFERS)` index-only-scan census reproduced
  `pgstatindex.leaf_pages` exactly on all 12 fixtures with `Heap Fetches: 0`
  (2736 − 3 = 2733 on the control) and rebuilt `avg_leaf_density` to within 0.03
  to 0.14 points on 11 of them, with no contrib installed. Recorded six measured
  failure modes: stale `pg_class.reltuples` reporting 0.0% on an index whose
  rebuild goes 2745 → 276 blocks until `pg_stat_all_tables.n_live_tup` is
  substituted; an all-duplicate index that a rebuild *grows* from 1291 to 1376
  blocks because splits used `BTREE_SINGLEVAL_FILLFACTOR` 96 while a sorted
  build packs to 90, correctly predicted as −6.4%; a random-insert index
  permanently at 27.0%; a range-deleted index whose `avg_leaf_density` reads a
  healthy 89.83 while 2330 of 2745 blocks are deleted pages; a variable-width
  key mispriced by one MAXALIGN of an average; and an unvacuumed index whose
  300,000 heap fetches inflated the census from 3279 to 8452 leaf pages. Also
  documented `VACUUM VERBOSE`'s exact per-index page census, the no-op-VACUUM
  gap from `_bt_vacuum_needs_cleanup`, and `pg_relation_size(idx,'fsm') > 0` as
  a free deleted-page flag that was non-zero for exactly the one fixture with
  deleted pages. All 95 citations were machine-audited against the pinned source
  with 0 broken ranges, and all five filed SQL blocks ran verbatim on the pinned
  build. New test objects were dropped, the isolated server was stopped, and
  human `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are unchanged pending a claim-by-claim review.
- 2026-08-04: answered a filed follow-up on Proposing a Sampling
  pgstatginindex Variant for PostgreSQL 12 (unverified) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied first; the user chose corrected
  wording, source plus exact-pin tests, and concrete new wasted-space output
  fields rather than a diagnostic-only answer. Established the central
  distinction from source: the function can measure free space but not wasted
  space, because GIN has no `fillfactor` (`ginoptions` parses only `fastupdate`
  and `gin_pending_list_limit`) and `entrySplitPage` equalizes entry pages by
  data size with no sorted-insert case, so ~50% free is the structural steady
  state. Four healthy, never-deleted fixtures measured 49.54-49.66% free, and
  a sibling index built over identical rows was byte-identical to the original
  (412 blocks, 50.46% density, same 1,688,200 used bytes), so a rebuild would
  return zero bytes against a reported 49.66% free. Proposed two exact fields
  (`pending_space`, `total_space`) and four estimates (`approx_free_space`,
  `approx_free_percent`, `approx_recyclable_pages`, `approx_recyclable_space`)
  on the `pgstathashindex` model of counting whole unused pages as free and
  excluding bookkeeping pages from the denominator, reading the delete XID out
  of `pd_prune_xid` to split recyclable from stuck pages. The census's
  recyclability verdict matched VACUUM's own `pages_free` and `pg_freespace`
  exactly on all seven fixtures (465, 168 and 40 blocks at avail 8160, zero
  elsewhere), and the two deletion paths were distinguishable on disk
  (`prune_xid = 0` for 393 drained pending pages against 522 and 518 for
  posting pages deleted by `ginDeletePage`). Recorded the hard limits with
  measurements: deleting 360,000 of 400,000 rows without VACUUM moved not one
  output field while `pgstattuple` reported 75.13% dead heap tuples; entry
  tuples for keys that lost every TID survive VACUUM, 4,000 retained where a
  fresh build has 2,000, with the fork never shrinking; and a 246-page pending
  list at 99.6% density was 30.98% of one index yet left `free_percent`
  indistinguishable from a healthy index's. 1,800 further seeded runs plus
  nine full-sample checks established full-sample equivalence on every new
  field, `free_percent` within 0.25 points of truth from 137 of 13,672 pages,
  a worst case of 9.72 points on a 308-page index, and a 60% alarm threshold
  classifying bloated versus healthy correctly in 800 of 800 runs inside a
  measured 51.22-to-62.02 gap, with an empty index as the one false positive.
  Four sibling-index rebuilds priced the fields honestly: of the bytes raw free
  space reported, only 97.7%, 91.4%, 70.0% and 0.0% materialized as recovered
  bytes, while a baseline-corrected form predicted 96.2-98.5% of them. All 292
  citations on the page were machine-audited against the pinned source with 0
  errors across 55 files, and the filed census prototype ran verbatim and
  reproduced the recorded numbers. New test objects were dropped, the isolated
  server was stopped, and human `verified: false`, the visible `(unverified)`
  title, and `verified_by_agent: not yet` are unchanged because this was a
  scoped follow-up. `scripts/wiki_lint` reports 0 errors and 0 warnings.
- 2026-08-04: filed Proposing a Sampling pgstatginindex Variant for PostgreSQL
  12 (unverified) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2), scoped to GIN only and derived from the existing v12 B-tree
  sampling proposal. Prompt hygiene applied first; the user chose corrected
  wording, a GIN-native page-class contract, exact-pin tests in addition to
  source, and a GIN-specific sample policy rather than the B-tree page's
  100 MiB / 10% floor. Established that v12 offers no physical GIN page
  accounting at all (`pgstattuple()` rejects GIN, `pgstatindex` and
  `pgstathashindex` reject it by access method, `pgstatginindex` reads only
  block 0, and every `pageinspect` GIN function takes one caller-supplied raw
  page), then documented the full flag taxonomy with the ordering trap that
  `GinPageSetDeleted` ORs `GIN_DELETED` onto a posting page's existing
  `GIN_DATA`/`GIN_LEAF` bits while `shiftList` overwrites a drained pending
  page's flags to exactly `GIN_DELETED`; the three per-class capacity
  denominators (8160 entry/list, 8152 data); the data-page `pd_lower`/`maxoff`
  traps; and that GIN never truncates the main fork. Designed three
  GIN-specific rules — an exact pending stratum from the real-time metapage
  counters, a post-stratified expansion over the exactly-known non-pending
  stratum, and a metapage-derived sample floor with an explicit cap — and
  recorded that nothing in v12 samples an index, the only `BlockSampler` caller
  being the table-AM-bound `acquire_sample_rows`. Exact-pin measurements on one
  isolated 12.2 server used seven fixtures (a 106.82 MiB entry-only control, a
  68.57 MiB mixed index carrying all six classes at once, a 246-page pending
  list, a 41.1%-deleted posting-tree index, a small healthy index, a partial
  index, and an empty index) and 6,100 seeded runs: full-sample equivalence held
  on all seven; post-stratification cut the `entry_leaf_pages` standard
  deviation from 46.6 to 6.8 at a 31% pending share and removed a low bias
  (544.0 versus 537.1 against a truth of 544), and from 227.8 to 149.0 at 4.5%;
  entry-leaf density at 1% coverage reproduced 50.44% with 0.00 maximum error
  over 100 seeds; and a one-page class returned `NaN` in 88 of 100 runs.
  Recorded an objective negative result: the page's own first proposal, a flat
  50-page absolute floor, lost to the B-tree size-based rule — an 88-page sample
  missed a 12-page posting-tree-internal class in 87 of 100 runs against 32 for
  the 878-page B-tree rule — so the floor was replaced by the metapage-derived
  one, which reached 18.52% of the index against a requested 1% and still missed
  the class 6 times in 100. Quantified four distinct metapage misreports
  (deleted-but-unrecyclable pages hidden inside `nDataPages`, recyclable ones
  vanishing from every class so 41.1% of one index's blocks became invisible,
  pending pages counted nowhere, and `nTotalPages` reading 548 against 794 live
  blocks), plus a VACUUM that flushed a 393-page pending list and *grew* the
  index from 8,777 to 9,325 blocks with 465 deleted pages left behind. Verified
  the exact pending-list walk and every prototype error path, including that a
  B-tree index is rejected only accidentally by `gin_metapage_info`'s
  `flags == GIN_META` check. All 219 citations were machine-audited against the
  pinned source with 0 errors across 48 files; human `verified: false`, the
  visible `(unverified)` title, and `verified_by_agent: not yet` are unchanged
  pending a claim-by-claim review. `scripts/wiki_lint` reports 0 errors and 0
  warnings.
- 2026-08-04: answered a filed follow-up on [Planner Penalties for Bloated
  Indexes in PostgreSQL 17
  (unverified)](v17/questions/query-planning/bloated-indexes-query-planner.md) against
  unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10).
  Prompt hygiene applied first; the user chose corrected wording, source plus
  exact-pin tests, and a `## Follow-Up` section on the existing v17 page rather
  than a new page. Established from source that v17 discards a GIN index at
  three separate gates, only the last of which is cost: clause matching
  (`op_in_opfamily()` in `match_opclause_to_indexcol()`, with
  `get_index_clause_from_support()` as the only escape hatch, and the four core
  `pg_amop.dat` GIN families carrying no `<`/`<=`/`>=`/`>`); plan shape from
  GIN's AM flags (`amgettuple = NULL`, `amcanorder`/`amcanorderbyop` false,
  `amcanreturn = NULL`, `amsearchnulls`/`amsearcharray`/`amcanparallel` false),
  which removes plain index scans, `ORDER BY` pathkeys, index-only scans,
  `IS NULL`, native `IN (...)`, and parallel index scans; and cost, where
  `gincostestimate()` charges both `random_page_cost` and
  `DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost` for every pending, entry and
  data page with no tree-height charge, after which `add_path()` fuzzy dominance
  and `choose_bitmap_and()` drop the loser. Also documented the 4X
  stale-metapage-statistics fallback and the trap that
  `gin_clean_pending_list()` leaves `nTotalPages` stale, the keyless
  full-index path a partial GIN index can still yield through `amoptionalkey`
  and the `indexQuals == NIL` branch, the four `CREATE INDEX`/`CLUSTER`
  rejections, and where GIN still wins. Recorded one v17-specific correction
  found by checking rather than carrying over the v12 answer: a bare boolean
  `Var` *does* match a `btree_gin` bool opclass, because `IsBooleanOpfamily()`
  falls back to `op_in_opfamily(BooleanEqualOperator, ...)` for non-built-in
  opfamilies and `match_boolean_index_clause()` rewrites the clause. Exact-pin
  measurements on one isolated 17.10 server (previous pin) priced the same `n = 42` predicate
  at `12.97` through a `btree_gin` GIN index and `4.52` through a B-tree on a
  single table with literally identical statistics (the other index dropped
  inside a rolled-back transaction), with both closed forms reproduced in SQL to
  the cent (`12.9725` and `4.5225`), even though the GIN index was 279 blocks
  against the B-tree's 280; reproduced B-tree wins on `BETWEEN`
  (`67.15` vs `44.31`), `n < 20` (`28.63` vs `8.89`) and `IN (1,2,3)`
  (`30.10` vs `13.57`); proved the three no-path cases with `disable_cost`-priced
  sequential scans; showed a 982-page `fastupdate` pending list moving GIN's
  index cost from `13.80` to `4187.83` so the planner dropped it from the
  `BitmapAnd` and demoted `tsv @@ …` to a `Filter`, with
  `gin_clean_pending_list()` returning exactly `982`; recovered the charged page
  count exactly in five separate states through a two-`random_page_cost` probe
  (`3.00`, `3.00`, `985.00` = 982 pending + 2 entry + 1 data page under the
  invented-statistics branch, `4.00`, and `1001.00` on a 10-block keyless
  partial index); reproduced all four AM rejection messages and the live
  `amutils` property matrix; and measured a multicolumn `gin (tsv, cat)` index
  beating the `BitmapAnd` at `21.55` versus `183.18`, confirming the
  `btree_gin` documentation's own claim. Two filed diagnostic SQL blocks ran
  verbatim against objects literally named `my_table`, `my_col` and
  `my_gin_index`, reporting `pending_pct_of_index = 72.57` and `303.00` charged
  pages of which 246 were pending. Recorded the explicit absence of any
  upstream GIN-versus-B-tree plan comparison and of any `gincostestimate()`
  test coverage, plus five new `## Open Questions` including a `BitmapAnd`
  outcome that was not stable across two runs. All 142 citations on the page
  were machine-audited against the pinned source; human `verified: false`, the
  visible `(unverified)` title, and `verified_by_agent: not yet` are unchanged
  because this was a scoped follow-up, not a fresh full-page claim audit.
  `scripts/wiki_lint` reports 0 errors and 0 warnings.
- 2026-08-04: filed [Planner Penalties for Bloated Indexes in PostgreSQL 17
  (unverified)](v17/questions/query-planning/bloated-indexes-query-planner.md) against
  unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10).
  Prompt hygiene applied first; the user chose corrected wording, exact-pin
  empirical tests in addition to source, and a new standalone v17 page
  cross-linked to the existing v12 page. Established from source that v17 has
  exactly four bloat-sensitive planner inputs, all filled by
  `get_relation_info()`: `IndexOptInfo.pages` (a live
  `RelationGetNumberOfBlocks()` answer for a non-partial index,
  `estimate_rel_size()`'s `pg_class` density for a partial one),
  `IndexOptInfo.tree_height` from `_bt_getrootheight()`'s `btm_fastlevel`, the
  `index_pages` argument threaded into `index_pages_fetched()` and
  `compute_parallel_worker()`, and the v17-only descent clamp
  `num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333))`. Recorded
  that `pg_class` carries no density or fragmentation column, that
  `pgstatindex`'s `avg_leaf_density` and `leaf_fragmentation` are computed from
  live leaf pages only and are read by nothing on the cost path, and that only
  B-tree, hash, GiST and SP-GiST route through `genericcostestimate()`.
  Catalogued eight bloat shapes with a per-query-shape sensitivity matrix.
  Built the since-v12 section from the checkout's own history: `git log -L`
  bounded by `REL_12_0..HEAD` plus full-function diffs showed
  `index_pages_fetched()` and `cost_index()` are byte-identical and that only
  five commits touched `btcostestimate` (three cosmetic), and
  `git tag --contains` attributed v13 `0d861bbb702`, v14 `d168b666823` /
  `9dd963ae253` / `5100010ee4d` / `1e55e7d1755` / `3499df0dee8` /
  `3d351d916b2`, v16 `eb5c4e953bb` / `cd9479af2af` / `3c569049b7b`, and v17
  `5bf748b86bc` / `9391f71523b` to their first release tags, with the in-tree
  docs and nbtree README supplying their own "prior to PostgreSQL 14" and
  "PostgreSQL 14 added" boundaries. Exact-pin measurements on one isolated
  17.10 server reproduced a closed-form cost prediction from
  `(pages, tuples, fastlevel)` to the cent (`123144.43` exact), an identical
  `4.44` point lookup across a 2,745-block/90.06% index and its
  26,411-block/9.62% twin, a `1428.00` cost gap equal to exactly
  `357 * random_page_cost` between 49.73% and 0% `leaf_fragmentation` with zero
  residual, two 2,745-block indexes over the same 100,000 rows both priced at
  `12730.42` while `avg_leaf_density` read 9.27% versus a healthy-looking
  89.18% (the latter hiding 2,465 deleted pages), a forged
  `pg_class.relpages = 1` that left a non-partial index's cost unchanged at
  `25528.42` while forging a partial index's `reltuples` moved it from
  `24140.12` to `23140.29`, a `cpu_operator_cost = 1` run isolating the height
  charge to exactly `50.00`, a `Seq Scan` plan flip at 25% selectivity, an
  index dropped from a `BitmapAnd` with its qual demoted to a `Filter`, 4
  versus 6 parallel workers, a SAOP descent plateau at exactly 3 versus 19,
  852 versus 2,749 blocks with `deduplicate_items = off` (cost gaps `68.00`
  and `7516.00` matching page arithmetic exactly), and 583 versus 1,174 blocks
  of version-churn growth without and with a held `REPEATABLE READ` snapshot.
  Recorded the explicit test absence (`src/test` has no reference to
  `tree_height`, `btcostestimate`, or `genericcostestimate`; the only in-tree
  `pgstatindex` test runs on an empty index and expects `NaN`). Human
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are unchanged pending a claim-by-claim review.
  `scripts/wiki_lint` reports 0 errors and 0 warnings.
- 2026-08-04: answered a filed follow-up on [Planner Penalties for Bloated
  Indexes in PostgreSQL 12
  (unverified)](v12/questions/query-planning/bloated-indexes-query-planner.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
  Prompt hygiene applied first; the user chose corrected wording ("When might a
  GIN index be discarded by the query planner and a B-tree used instead?"),
  source-plus-exact-pin-tests scope, and normalization of the page's
  `## Answer Up Front` heading to `## Answer` plus a new `## Contents` table of
  contents. Established from source that v12 discards a GIN index at three
  separate gates, only the last of which is cost: clause matching
  (`op_in_opfamily()` in `match_opclause_to_indexcol()`, with
  `get_index_clause_from_support()` as the only escape hatch, and the core
  `pg_amop.dat` GIN families carrying no `<`/`<=`/`>=`/`>`); plan shape from
  GIN's AM flags (`amgettuple = NULL`, `amcanorder`/`amcanorderbyop` false,
  `amcanreturn = NULL`, `amsearchnulls`/`amsearcharray`/`amcanparallel` false),
  which removes plain index scans, `ORDER BY` pathkeys, index-only scans,
  `IS NULL`, native `IN (...)`, and parallel index scans; and cost, where
  `gincostestimate()` charges `random_page_cost` for every pending, entry, and
  data page with no descent shortcut, then `add_path()` fuzzy dominance and
  `choose_bitmap_and()` drop the loser. Also documented the 4X
  stale-metapage-statistics fallback, the keyless full-index path a partial GIN
  index can still yield through `amoptionalkey` and
  `GIN_SEARCH_MODE_EVERYTHING`, the four `CREATE INDEX` AM rejections, and where
  GIN still wins. Exact-pin measurements on an isolated 12.2 server priced the
  same `n = 42` predicate at `12.22` through a `btree_gin` GIN index and `4.65`
  through a B-tree on one table with identical statistics, even though the GIN
  index was 279 pages against the B-tree's 826; reproduced B-tree wins on
  `BETWEEN` (`50.68` vs `15.10`), `n < 20` (`28.69` vs `13.11`), `IN (1,2,3)`
  (`28.67` vs `13.94`), `ORDER BY … LIMIT 10` (`8045.40` vs `1.09`),
  index-only scan (`123.74` vs `4.95`), and `IS NULL` (`5343.10` vs `8.43`);
  proved the three no-path cases with `disable_cost`-priced sequential scans at
  `10000000000.00`; showed a 1177-page `fastupdate` pending list moving GIN's
  index cost from `12.45` to `4720.55` so the planner dropped it from the
  `BitmapAnd` and demoted `tsv @@ …` to a `Filter`; and verified the page charge
  exactly, with `pgstatginindex` reporting 393 pending pages,
  `(1588.55 - 397.55) / 3 = 397` pages charged at two `random_page_cost`
  settings, and `gin_clean_pending_list()` returning `393`. A separate 50-page
  run of the three filed SQL blocks, executed verbatim against objects literally
  named `my_table` and `my_gin_index`, reported
  `pending_startup_cost = 200` and a GIN cost drop from `217.51` to `17.51`.
  Recorded the explicit absence of any upstream GIN-versus-B-tree plan
  comparison and of any `gincostestimate()` test coverage. Reset
  `verified_by_agent` to `not yet` because this was a scoped follow-up rather
  than a fresh full-page claim audit; human `verified: false` and the visible
  `(unverified)` title are unchanged. `scripts/wiki_lint` reports 0 errors and
  0 warnings.
- 2026-08-03: filed How a GIN Index Becomes Bloated in PostgreSQL 17, and How
  to Measure It (unverified) against
  unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10).
  Prompt hygiene applied first; the user approved corrected wording, asked for
  contrib and core-SQL-only measurement paths split cleanly, a full exact-pin
  empirical run, and a "what changed since PostgreSQL 12" section. Established
  seven bloat mechanisms from source and, contrary to the usual assumption,
  measured that a rebuild is not always smaller: because `entrySplitPage`
  equalizes by data size with no rightmost or build case and a build drains its
  accumulator in ascending key order, a fresh build settles at 50.31%-52.72%
  entry-leaf fill, so `REINDEX` grew a scattered-retail index (57.79% fill) from
  7,692,288 to 8,765,440 bytes while an ascending-retail index was byte-identical
  to its rebuild. Also measured a 16,801,792-byte entry tree surviving deletion
  of all 300,000 rows and two VACUUMs before `REINDEX` cut it to 16,384 bytes;
  posting leaves at 7.50% fill after 95% of rows were deleted; 76 deleted posting
  pages entering the FSM only after the visibility horizon advanced; FSM `avail`
  of exactly 8160 rather than the `BLCKSZ - 1` implied by the `indexfsm.c`
  comment; a 1471-page pending list moving a forced `Bitmap Index Scan` from
  17.63 to 6260.82 and buffers from 4 to 1473; and the three different meanings
  of `pg_class.reltuples` (300,000 extracted keys after `CREATE INDEX`, 100,000
  heap rows after `ANALYZE`, 90,000 after VACUUM, against a true entry count of
  2,988). Introduced a two-`random_page_cost` `EXPLAIN` probe that recovered
  pending-page counts exactly (591.00 and 101.00 against 589 and 99 pending pages
  plus two fixed pages) and a `CREATE INDEX CONCURRENTLY` rebuild probe whose
  `fresh_bytes` matched the later `REINDEX` size byte-for-byte. Attributed every
  since-v12 change to the checkout's own Stamp-bracketed history, including the
  v16 `DEFAULT_PAGE_CPU_MULTIPLIER` charges that changed the probe's scale factor
  and the v14 index-vacuum bypass and failsafe that can skip GIN entirely. All
  nine filed SQL blocks ran verbatim on an isolated exact-pin server, against
  objects literally named `my_table` / `my_col` / `my_gin_index`. Human
  `verified: false` and the visible `(unverified)` title are unchanged, and
  `verified_by_agent` stays `not yet` pending a separate claim-by-claim review.
  `scripts/wiki_lint` reports 0 errors and 0 warnings.
- 2026-08-03: answered a filed core-SQL-only follow-up on How a GIN Index
  Becomes Bloated in PostgreSQL 12, and How to Measure It (unverified) against
  unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2). The user chose
  corrected prompt wording, core-only scope, and a ranked presentation of all
  three candidate methods. Established from source that core SQL cannot inspect
  a GIN index at all: `pg_proc.dat` holds no page, page-header, metapage, or
  FSM-contents function, `GinStatsData` is reachable only from C, and no
  `pg_statistic` slot carries a distinct-element count, so
  `pg_stats.elem_count_histogram` yields postings per row for array columns only
  and nothing for `tsvector` or `jsonb`. Ranked three methods by measured
  accuracy on five deterministic fixtures across three churn rounds: a
  `CREATE INDEX CONCURRENTLY` rebuild probe whose `fresh_bytes` matched the
  later `REINDEX` size byte-for-byte in all five cases; a read-only recorded
  bytes-per-row baseline that stayed within 2.75 percentage points of probe
  truth over eight comparisons and returned exactly 0.00% on the untouched
  control; and a `TABLESAMPLE` extrapolation rejected with numbers, at +17% to
  +455% fresh-size error and −384% reported bloat on a 0%-bloated index, traced
  to GIN size being strongly sublinear in row count. Documented the three
  different meanings `pg_class.reltuples` carries for a GIN index (799,493
  extracted keys after `REINDEX` versus 266,667 heap rows after
  `VACUUM (ANALYZE)` on the same index), which fork each core size function
  counts (a measured 24 kB FSM fork made `pg_table_size` exceed
  `pg_relation_size` by 0.2%), and two core-only pending-list probes: the
  `Bitmap Index Scan` total cost divided by `random_page_cost` (887.5 and
  1,471.7 against 883 and 1,469 real pending pages) and `gin_clean_pending_list`
  returning 883. All eight filed SQL blocks ran verbatim on an isolated
  exact-pin server. Added seven follow-up entries under `## Open Questions`,
  including that no core-only method exists for a first measurement with no
  prior rebuild. Reset `verified_by_agent` to `not yet` because this was a
  scoped follow-up rather than a fresh full-page claim audit; human
  `verified: false` and the visible `(unverified)` title are unchanged.
  `scripts/wiki_lint` reports 0 errors and 0 warnings.
- 2026-08-03: completed a full corrective review of How a GIN Index Becomes
  Bloated in PostgreSQL 12, and How to Measure It (unverified) against
  unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2). Corrected four
  material overstatements: build and retail posting-page split policies are
  segment-based heuristics rather than exact fill percentages; an autovacuum
  partial clean can still empty the pre-existing list when there is no concurrent
  append; ordinary VACUUM does not truncate the GIN fork, but `VACUUM FULL`,
  `CLUSTER`, and `TRUNCATE` can rebuild or reset index storage; and a raw deleted
  page count mixes XID-delayed posting-tree pages with former pending-list pages
  that `shiftList` can record in the FSM immediately. Added the separate retained
  entry-page pattern caused when growing in-line posting tuples split entry pages
  before becoming small posting-tree pointers, qualified VACUUM's segment
  recompression and index-cleanup caller gates, and corrected the measurement and
  generated-header boundaries. All five production SQL recipes ran verbatim on an
  isolated exact-pin server. Deterministic fixtures reproduced pending-list growth
  and cleanup, a 16,801,792-byte entry tree surviving deletion of all 300,000 rows,
  an 8.2% retail entry-tree gap with identical posting-page counts after rebuild,
  sparse posting trees falling from 1,654,784 to 98,304 bytes only after rebuild,
  XID-delayed FSM publication for 12 deleted posting pages, and the planner cost
  shift from 5903.25 to 27.25 when a 1471-page pending list was cleaned. Recorded
  `verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-08-03T18:24:11Z`; human
  `verified: false` and the visible `(unverified)` title remain unchanged.
  `scripts/wiki_lint` reports 0 errors and 0 warnings.
- 2026-07-29: completed a full corrective review of [Comprehensive
  plan_cache_mode Analysis in PostgreSQL 12
  (unverified)](v12/questions/query-planning/plan-cache-mode.md) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2). The page was a
  thin 82-line draft whose 16 distinct citations were all single-line `#L`
  links carrying `file.c#Symbol` labels, with no `## Contents`, no
  `## Evidence Map`, no `## Navigation`, a slug title, and `## Open Questions`
  reading "Pending deep inquiry"; it is now a 125-citation walkthrough. Renamed
  the file from `plan_cache_mode_analysis.md` to `plan-cache-mode.md` to match
  the directory convention, and reworded the filed `## Question` from an
  imperative into a question on explicit user approval. Corrections to the old
  draft: the "decision hierarchy" omitted that `force_generic_plan` and
  `force_custom_plan` are checked *after* the one-shot, no-parameter, and
  transaction-statement rules and therefore cannot override them; the planning
  charge was described without noting it applies to custom plans only; and the
  claimed `GetCachedPlan` "correction" line pointed at `L1200` without the
  surrounding build-and-reprice logic. Added the material the draft lacked:
  `cursor_options` rule 6 and its SPI-only door, `generic_cost = -1` first-pass
  bias, the `INT_MAX` counter guard, `PARAM_FLAG_CONST` folding with the
  `var_eq_const`/`var_eq_non_const` split, retention of the cost history across
  invalidation and `DISCARD PLANS`, the per-execution `AcquireExecutorLocks`
  footprint, PL/pgSQL static/dynamic/simple-expression boundaries, named versus
  unnamed protocol statements, v12 observability limits, error paths,
  build/extension boundaries, test gaps, and five ancestor commits. Exact-pin
  measurements on an isolated 12.2 server with `contrib/auto_explain` installed
  reproduced the switch at execution 6 (charged custom 25.02 versus
  `generic_cost` 18.77), proved the rejected generic plan is cached by recovering
  a stale `Seq Scan` plan under `enable_seqscan = off` where a fresh generic plan
  costs 28.29, showed `force_custom_plan` unable to replan a parameterless
  statement, showed PL/pgSQL static SQL switching at call 6 while dynamic
  `EXECUTE ... USING` ignored `force_generic_plan`, contrasted
  `pgbench -M prepared` (5 custom then generic) with `-M extended` (always
  custom), measured 2 versus 5 relation locks on a four-partition table, and
  measured 0.65 ms versus 0.006 ms of planning per execution on a six-way join.
  Both fenced SQL blocks ran verbatim at the pin with `ON_ERROR_STOP=1`. The
  server was stopped and its disposable fixtures were left under
  `.wiki-runtime/`. Agent verification was recorded as
  `claude-opus-5-max 2026-07-29T21:23:49Z`; human verification remains false.
- 2026-07-29: completed a full corrective review of [Impact of B-Tree Leaf
  Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12
  (unverified)](v12/questions/indexing/leaf-density-60-vs-90-query-impact.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
  Re-resolved all 67 original citation ranges and re-cut, split, or replaced
  most of them; the page now carries 92 distinct ranges. Typical fixes were
  ranges that began mid-comment or ended mid-statement (`nbtree.h#fillfactor`,
  `plancat.c#get_relation_info`, `costsize.c#index_pages_fetched`,
  `nbtsplitloc.c` split selection, `selfuncs.c#genericcostestimate`), a
  `_bt_readpage` range that ran past the function into `_bt_saveitem`, a
  `_bt_readnextpage` backward range that started mid-comment, an
  `estimate_rel_size` range that spilled out of the index case, and
  multi-symbol "X and Y" labels replaced with one symbol per citation.
  Corrected the `EXPLAIN BUFFERS` claim (temp buffers have only read and
  written counters) and the planner-input claim (`index->pages` is the live
  `RelationGetNumberOfBlocks` count, not `pg_class.relpages`; the catalog
  density path applies to partial indexes). Restructured to the mandatory
  shape: added `## Contents`, folded `## Answer Up Front` and its sections into
  one `## Answer`, and renamed `## Related Pages` to `## Navigation`. Added
  source-backed material the page lacked: how splits rather than deletes reach
  60% density, `LP_DEAD` occupancy and in-page compaction, half-dead versus
  deleted page accounting, the absence of index-page prefetch in v12, the
  scan-path data structures, five `PGC_USERSET` settings with apply scope, and
  the contrib plus generated-`BTREE_AM_OID` build boundaries. Exact-pin
  measurements on an isolated 12.2 server reproduced the central claim: over
  1,000,000 `bigint` keys, 90.06 versus 59.90 `avg_leaf_density` gave 2733
  versus 4116 leaf pages and 2738 versus 4121 warm index-only scan buffers
  (50.5% more), with the entire 5552.01 serial-plan cost gap equal to 1388
  extra blocks at `random_page_cost = 4`; a 10,000-key range measured 32 versus
  45 index-only buffers and 31 versus 44 `Bitmap Index Scan` buffers at
  identical heap blocks; the equality probe was identical (cost 0.42..4.44, 4
  buffers) at unchanged `tree_level`; a 100,000-row wide-key fixture showed
  90-to-60 raising `tree_level` from 2 to 3, the probe cost to 0.54..4.56, and
  the probe to 5 buffers; and 1,000,000 random keys inserted retail settled at
  65.58% density with 49.71% `leaf_fragmentation` and no deletions. The filed
  `## Question` was reworded from an imperative with a question mark to a
  question on explicit user approval. The server was stopped and its
  disposable fixtures were left under `.wiki-runtime/`. Agent verification was
  recorded as `claude-opus-5-max 2026-07-29T20:41:00Z`; human verification
  remains false.
- 2026-07-29: completed a full corrective review of [B-Tree Leaf Density vs
  Fragmentation Impact on Index Scan I/O in PostgreSQL 12
  (unverified)](v12/questions/indexing/leaf-density-vs-fragmentation-index-scan-io.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Re-resolved all 52 citation ranges and re-cut or split 40
  of them, mostly ranges that started or ended mid-statement or mid-comment,
  including a `cost.h` range
  that ran into an unrelated enum, an `indexam.sgml` range attached to the
  "must be written in C" paragraph instead of the page-cost paragraph, a
  `_bt_walk_left` range that covered less than half the function, and
  `nbtree.c#btvacuumpage-recycle` / `nbtpage.c#_bt_getbuf-free-page` ranges
  that missed the `RecordFreeIndexPage` and `GetFreeIndexPage` code they were
  cited for. Corrected the `EXPLAIN BUFFERS` claim (temp buffers have only
  read and written counters) and one rounding cell (1.72 to 1.73). Added the
  mandatory `## Contents` block, folded the answer under a single `## Answer`
  section, and added source-backed material the page lacked: v12 issues no
  index-page prefetch, `LP_DEAD` entries still count as dense until an insert
  or VACUUM removes them, half-dead pages stay in the leaf chain while being
  excluded from both `pgstatindex` columns, deleted pages are unlinked yet
  still priced through `index->pages`, both columns can be `NaN`, and the
  three relevant GUCs are `PGC_USERSET`. Exact-pin measurements on an isolated
  12.2 server reproduced the central claim: 90.06 versus 59.90 `avg_leaf_density`
  raised warm index-only scan buffers from 2738 to 4121 (50.5 %, against a
  predicted 1.5035x), while removing 49.95 % `leaf_fragmentation` at matched
  density moved them only from 3697 to 3680; `pageinspect` confirmed 1845 of
  3694 backward right links, a 1848-block mean jump, and zero links to the next
  physical block; and a 90 %-deleted index reported `avg_leaf_density = 90.00`
  over the same 547 leaf pages until VACUUM dropped it to 9.26. The filed
  survey query was executed at the pin and then hardened into a
  `WITH ... AS MATERIALIZED` candidate list so an inlined plan cannot call
  `pgstatindex` on a hash or GiST index; it returned only the ordinary B-trees
  in a database that also held hash, GiST, GIN, BRIN, and partitioned-parent
  indexes. The server was stopped and
  its disposable fixtures were left under `.wiki-runtime/`. Agent verification
  was recorded as `claude-opus-5-max 2026-07-29T19:54:40Z`; human verification
  remains false.
- 2026-07-29: completed a full corrective review of [PostgreSQL 12 Database
  Health Checklist (unverified)](v12/questions/server-administration/database-health-checklist.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Re-resolved and label-checked every citation, now 345.
  Fixed a `checkpointer.c#ForwardFsyncRequest` label naming a symbol that does
  not exist in v12 (`ForwardSyncRequest`, whose range also ran past the
  function), an `analyze.c#do_analyze_rel` label whose range sits in
  `analyze_rel` and stopped before the partition recursion, two
  `relation_needs_vacanalyze` ranges that covered only part of the function's
  header comment, a threshold-decision range that omitted the
  statistics-unavailable branch, a `monitoring.sgml#pg_stat_all_tables` range
  cited under the `pg_stat_progress_vacuum` paragraph, a progress-vacuum doc
  range that stopped inside the `phase` row before the columns the query
  selects, swapped `auth.c` labels (`ClientAuthentication` pointing into
  `auth_failed`, `password-failures` pointing at the `no pg_hba.conf entry`
  block), a `high-availability.sgml#hot_standby_feedback` citation whose lines
  discuss `pg_last_wal_replay_lsn` instead, a `PGC_SU_BACKEND` range that cut
  off before the code that ignores a reload in existing backends, a
  `LogCheckpointEnd` range that cut off before the write/sync duration fields,
  an `initdb.c` citation that only set the `system_views.sql` path, and a
  `wait_for_stats` range that started before the function and dropped its
  terminator. Added the missing evidence for nine replication GUC apply
  scopes, `data_checksums`, `initdb --data-checksums`, `log_lock_waits`,
  `log_temp_files`, the `pg_monitor` role grants, the `varsup.c` XID
  wraparound messages, the `LogCheckpointStart`/`LogCheckpointEnd` message
  sites, the `exec_simple_query` duration-log site, and
  `pgstat_recv_recoveryconflict`, which also documents that a dropped-database
  conflict is deliberately not counted. Corrected "role ownership" to the
  actual `has_privs_of_role` membership test and the
  `pg_stat_statements` redaction from "null" to a nulled `queryid` plus an
  `<insufficient privilege>` text placeholder. An isolated exact-pin primary
  and streaming standby ran all eleven fenced SQL blocks without error and
  reproduced the `pg_monitor` progress gate (`pid` and `datname` only), a
  recovery conflict counted as `confl_snapshot` with its client cancellation,
  a PID-zero prepared-transaction blocker with its retained relation and
  transaction locks, lock-wait and deadlock logging with the `deadlocks`
  counter, temp-file logging with `temp_files`/`temp_bytes`, the v12
  autovacuum summary's absent WAL field, and the `pg_settings` superuser-only
  filter. Both servers were stopped and their fixtures left under
  `.wiki-runtime/`. Agent verification was recorded as
  `claude-opus-5-max 2026-07-29T16:53:15Z`; human verification remains false.
- 2026-07-29: completed a full corrective review of [How wal_sender_timeout Is
  Used and What It Impacts in PostgreSQL 12
  (unverified)](v12/questions/replication-and-wal/wal-sender-timeout.md) against unchanged pin
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
- 2026-07-29: completed a full corrective review of A Heuristic to Detect
  B-Tree Index Bloat in PostgreSQL 12 (unverified) against unchanged pin
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
  (unverified)](v14/questions/server-administration/row-level-security-rls.md) against unchanged pin
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
  (unverified)](v18/questions/server-administration/row-level-security-rls.md) against unchanged pin
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
  (unverified)](v18/questions/server-administration/row-level-security-rls.md) against unchanged pin
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
  PostgreSQL 12 (unverified)](v18/questions/server-administration/row-level-security-rls.md) against
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
  Health Checklist (unverified)](v12/questions/server-administration/database-health-checklist.md)
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
  Settings (unverified)](v14/questions/server-administration/row-level-security-rls.md) and all five
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
  (unverified)](v17/questions/indexing/attach-partition-index-drops.md) against unchanged
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
- 2026-07-17: completed a full claim-to-source review of Proposing a Sampling
  pgstatindex Variant for PostgreSQL 17
  (unverified) against
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
- 2026-07-17: extended Proposing a Sampling pgstatindex Variant for PostgreSQL
  12 (unverified) with a
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
- 2026-07-17: extended Proposing a Sampling pgstatindex Variant for PostgreSQL
  12 (unverified) with the
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
- 2026-07-17: completed a full claim-to-source review of Proposing a Sampling
  pgstatindex Variant for PostgreSQL 12
  (unverified) against
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
  PostgreSQL 12 (unverified)](v12/questions/replication-and-wal/wal-sender-timeout.md) with the
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
  (unverified)](v12/questions/replication-and-wal/wal-sender-timeout.md) against unchanged pin
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
  (unverified)](v12/questions/query-planning/query-planner-statistics-sources.md) against
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
  PostgreSQL 12 (unverified)](v12/questions/indexing/create-index-concurrently.md) with
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
  PostgreSQL 12 (unverified)](v12/questions/indexing/reindex-index-concurrently.md) with
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
  PostgreSQL 17 (unverified)](v17/questions/indexing/reindex-index-concurrently.md) with
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
  PostgreSQL 17 (unverified)](v17/questions/indexing/create-index-concurrently.md) with
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
  (unverified)](v14/questions/storage-and-vacuum/multixact-foreign-keys-cache-spill.md) against
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
  (unverified)](v14/questions/query-planning/functions-procedures-in-where-clause.md) against
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
  (unverified)](v18/questions/server-administration/row-level-security-rls.md) against unchanged pin
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
  PostgreSQL 17 (unverified)](v17/questions/indexing/create-index-concurrently.md) with
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
  (unverified)](v12/questions/indexing/create-index-concurrently.md) against unchanged pin
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
  (unverified)](v12/questions/indexing/create-index-concurrently.md) against unchanged pin
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
  (unverified)](v12/questions/indexing/create-index-concurrently.md) against unchanged pin
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
  PostgreSQL 12 (unverified)](v12/questions/indexing/create-index-concurrently.md) with
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
  (unverified)](v12/questions/indexing/how-pgstatindex-calculates-information.md) against
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
  (unverified)](v12/questions/indexing/create-index-concurrently.md) against unchanged pin
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
  Settings (unverified)](v14/questions/server-administration/row-level-security-rls.md) against the
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
  (unverified)](v17/questions/query-planning/partitioning-planning-execution-optimizations.md)
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
  (unverified)](v17/questions/query-planning/partitioning-planning-execution-optimizations.md)
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
  (unverified)](v12/questions/query-planning/partitioning-planning-execution-optimizations.md)
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
  (unverified)](v12/questions/query-planning/partitioning-planning-execution-optimizations.md)
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
  (unverified)](v14/questions/server-administration/row-level-security-rls.md) against the unchanged
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
  Overhead (unverified)](v14/questions/query-planning/functions-procedures-in-where-clause.md)
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
  (unverified)](v14/questions/query-planning/functions-procedures-in-where-clause.md) against
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
  (unverified)](v14/questions/server-administration/row-level-security-rls.md) with a `### Wrapping a
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
  (unverified)](v14/questions/server-administration/row-level-security-rls.md) for the approved,
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
  (unverified)](v14/questions/server-administration/row-level-security-rls.md) against the unchanged
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
  (unverified)](v14/questions/storage-and-vacuum/multixact-foreign-keys-cache-spill.md), against the
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
  (RLS) in PostgreSQL 14 (unverified)](v14/questions/server-administration/row-level-security-rls.md),
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
  (unverified)](v18/questions/server-administration/row-level-security-rls.md) against the unchanged
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
  Autovacuum and VACUUM (unverified)](v19/questions/storage-and-vacuum/autovacuum-parallel-scoring-visibility.md):
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
