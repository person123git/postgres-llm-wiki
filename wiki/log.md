# Wiki Log

Append one entry after every scaffold change, version lifecycle event, ingest, trace, lint pass, or filed answer.

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
