# Wiki Log

Append one entry after every scaffold change, version lifecycle event, ingest, trace, lint pass, or filed answer.

## [2026-09-02] answer v17 | improvements since v12 in storage, planning, index bloat and vacuum, measured on 17.11 and 12.2

- Filed [Improvements Since PostgreSQL 12 in Storage Performance, Query Planning, Index Bloat, and
  Vacuum, as of PostgreSQL 17
  (unverified)](v17/questions/storage-and-vacuum/improvements-since-v12.md) at the unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11).
- **Prompt hygiene first.** The asker chose "correct and restate". The original read
  `follow agents.md, in postgresql 17, question: what are the improvement since version 12 in terms of
  storage performance, query planning, index bloat , vacuum.`; the page records the six corrections
  (`agents.md` -> `AGENTS.md`, `postgresql` -> `PostgreSQL`, `improvement` -> `improvements`,
  `version 12` -> `PostgreSQL 12`, the stray space before a comma, and the missing `and`). Three
  scoping answers were taken before drafting: **build and measure** rather than source-only, **one
  combined page** filed under `storage-and-vacuum` rather than four split pages, and **comprehensive**
  per-area coverage rather than a curated subset.
- **Two servers, both built out of tree from this repo's own pins**, so `raw/` was never written to:
  17.11 from `raw/postgres-17` at the pin (port 55717) and 12.2 from `raw/postgres-12` at
  `45b88269a35` / `REL_12_2` (port 55712), both `configure --without-readline --without-zlib
  --without-icu --enable-debug`, both with identical non-default settings, `autovacuum = off`, and
  per-table `autovacuum_enabled = false` on every fixture. The retained `.wiki-runtime/tmp/pta12`
  sandbox was **not** touched; a fresh 12.2 install was built instead.
- **Release dating deliberately does not use `git tag --contains`.** That test is wrong for any
  reverted-then-reintroduced feature, and two in scope are exactly that: it puts `MAINTAIN` in v16
  (`60684dd834a` is an ancestor of `REL_16_0`, but `151c22deee6` reverted it on 2023-07-07 and
  `ecb0fd33720` re-landed it for **v17**) and `recovery_prefetch` in v14 (`1d257577e08`, reverted,
  re-landed as `5dc0418fab2` for **v15**). Every date on the page instead comes from a presence test of
  the defining symbol at `REL_12_0` through `REL_17_0` and the pin: `MAINTAIN` reads 0 hits at
  `REL_16_0` and 9 at `REL_17_0`. The same method excluded the eager/lazy freezing strategies
  (`4d417992613`, reverted the same day by `6c6b4972664`, never shipped), showed v13's dedup entry
  point was `_bt_dedup_one_page` rather than today's `_bt_dedup_pass`, and showed Memoize shipped in
  v14 already renamed because the master rename `83f4fcc6550` was back-patched.
- **Inventory was driven by the checkout, not by recall**: the working tree was diffed against
  `REL_12_0` for GUCs (76 added, 12 removed), reloptions (5 added) and system views (15 added), and
  commit ranges were enumerated per subsystem (333 vacuum, 491 index-AM, 676 planner, 643
  storage/WAL).
- **Index bloat, measured.** Deduplication (v13, `0d861bbb702`): 22,519,808 -> 6,979,584 bytes
  (**-69.0%**) on 100 distinct keys, 66,486,272 -> 4,030,464 (**-93.9%**) on a single-value text index,
  and **byte-identical** results on two all-distinct indexes (22,487,040 and 11,255,808, same 2,733
  leaf pages, same 90.06% density). Bottom-up deletion (v14, `d168b666823`) was **separated from
  dedup** by rebuilding the churn fixture with `deduplicate_items = off` - legitimate because
  `_bt_delete_or_dedup_one_page` says it "deliberately omit[s] an index-is-allequalimage test" - which
  gives **-64.3% from bottom-up alone** where the key never changes, a further -28.7% from dedup, and
  **byte-identical to 12.2** (58,449,920) where the keys do change, proving the 61.6% saving there is
  entirely dedup acting on incidental duplicates. Heap growth was identical (79,683,584 bytes) on both
  servers, confirming identical churn.
- **Vacuum, measured.** The v17 TidStore (`30e144287a7` plus radix tree `ee1b30f128d`) cuts index
  passes **18 -> 2** over 3,000,000 dead tuples at `maintenance_work_mem = 1MB` on an identical
  28,038-page heap, 12.2 capping at 174,517 TIDs per pass; `dead_items_alloc`'s byte budget
  (`max_bytes = vac_work_mem * 1024L`) is why the progress view swapped `max_dead_tuples`/
  `num_dead_tuples` for `max_dead_tuple_bytes`/`dead_tuple_bytes`/`num_dead_item_ids` plus
  `indexes_total`/`indexes_processed`, both column lists captured from the two servers. The v14 bypass
  (`5100010ee4d`) - gated on 2% of `rel_pages` **and** under 32 MB of TidStore in v17 - turned two full
  index scans into `index scan bypassed: 1 pages from table (0.01% of total)` at 22 buffer hits, 0
  misses and 391 bytes of WAL.
- **A source-derived prediction was tested and confirmed.** v16 page-level freezing (`1de58df4fec`)
  appeared to do nothing: after a 300,000-row bulk load a plain `VACUUM` left `all_frozen = 0` on
  *both* majors. Reading `heap_page_prune_and_freeze` explained why - the opportunistic branch needs
  `all_visible && all_frozen && nfrozen > 0` **and** an FPI to be written anyway, via `hint_bit_fpi`,
  which is only set when the visibility checks themselves forced one. Turning on the
  `postmaster`-context `wal_log_hints` (both clusters restarted) flipped 17.11 to **3,704 of 3,704
  pages and 300,000 tuples frozen**, `relfrozenxid` age 0, for `11116 records, 3709 full page images,
  31380997 bytes` of WAL, while 12.2 still froze **none**. Both the negative and positive runs are on
  the page.
- **Query planning, measured.** Memoize (v14) 572,305 -> 586 buffer hits (**977x**, `Hits: 99950
  Misses: 50`, inner loops 100,000 -> 50); incremental sort (v13) 7,447 -> 11 buffers and 31.867 ->
  0.601 ms while reading 1,000 rows instead of 1,000,000; GROUP BY key reordering (v17, `0452b461bc4`)
  rewrote `GROUP BY b, a` to `a, b` and removed the sort entirely; the v16 presorted aggregate removed
  a 2,947-block internal sort spill (144.799 -> 96.176 ms).
- **The nbtree SAOP rework was measured with its own control.** 500 **clustered** keys: 1,504 -> **5**
  buffers (0.296 -> 0.095 ms). 500 keys spread 2,000 apart: 1,504 versus 1,501 - **no benefit** - which
  pins the v17 change (`5bf748b86bc`) to the removal of redundant descents rather than to arrays in
  general.
- **Storage, measured with a decisive control.** Read-syscall counts came from
  `/proc/<backend-pid>/io` (`syscr`), no ptrace needed, with `max_parallel_workers_per_gather = 0`
  after a first run mis-attributed only 18,602 of 53,192 blocks to the leader. On one 53,192-block
  table: 12.2 issues **53,210** calls at exactly 1.000 blocks per call; 17.11 issues 53,210 / 13,318 /
  3,346 / 1,685 at `io_combine_limit` 8/32/128/256 kB (1.000 / 3.994 / 15.897 / 31.568 blocks per
  call). The 8 kB row **reproduces 12.2 exactly**, isolating block combining as the entire gain.
- **Three results contradict the obvious expectation**, and are given their own section: v13
  disk-based hash aggregation is *slower* than 17.11's own sort path (840.533 ms / 61,208 kB versus
  786.466 ms / 23,504 kB, against 12.2's 992.216 ms / 27,416 kB), so the honest v12->v17 gain on that
  query is the **sort** getting 20.7% faster and 14.3% smaller; the SAOP rework is worth nothing on
  spread arrays; and page-level freezing changes nothing at default settings.
- **Eight cross-version boundary checks** captured exact error text, including `unrecognized VACUUM
  option "parallel"`, `relation "pg_stat_io" does not exist`, `unrecognized privilege type "maintain"`
  and `unrecognized configuration parameter` for `io_combine_limit`, `default_toast_compression`,
  `vacuum_failsafe_age` and `vacuum_buffer_usage_limit` on 12.2, against successful `GRANT MAINTAIN`,
  `VACUUM (PARALLEL 4)` and a 35-row `pg_stat_io` on 17.11.
- **Honest gaps, all filed as open questions.** This host has no `lz4.h` or `zstd.h`, so both builds
  are pglz-only and the v14 TOAST / v15 WAL compression ratios are **not** measured; instead the build
  gate itself is measured (`invalid value for parameter "default_toast_compression": "lz4"`) and
  explained from `default_toast_compression_options`' `#ifdef USE_LZ4`. The pre-v17 1 GB dead-TID
  ceiling is asserted only from measurement, since proving it would need v12 source citations the page
  may not make. `read_bytes` was 0 on every read-stream run, so no physical device I/O was measured
  anywhere.
- Apply scopes for all 16 named settings were read from `pg_settings.context` on the 17.11 server and
  mapped explicitly: `transaction_buffers` and `wal_log_hints` **restart**;
  `autovacuum_vacuum_insert_threshold` and `recovery_prefetch` **reload**; the other twelve
  session/transaction.
- Bookkeeping: `wiki/index.md`, `wiki/v17/index.md`, and both the v17 coverage cell and a dated
  coverage note in `wiki/versions.md`. `verified_by_agent` stays `not yet` - the page is newly drafted,
  not re-verified claim by claim - and `verified:` was not touched.
- **Verification**: 109 citations across 27 files resolve, sit in range and cite only
  `raw/postgres-17/`; 87 identifier labels name a symbol inside their cited hunk and the 12 remaining
  labels are explicit line ranges, each read by hand; all 16 `## Contents` anchors resolve in document
  order.
- `raw/` untouched: all five checkouts sit at their pinned commits, and both builds were made out of
  tree.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-09-01] answer v17 | VACUUM/autovacuum relation truncation, TOAST, and what changed since v12

- Filed [How VACUUM and Autovacuum Truncation Works in PostgreSQL 17, Whether It Covers TOAST,
  and What Changed Since PostgreSQL 12
  (unverified)](v17/questions/storage-and-vacuum/vacuum-truncation-and-toast.md) at the unchanged
  pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11, `REL_17_11-7-g786db8dcf16`).
- **Prompt hygiene first.** The prompt had subject-verb disagreement twice ("how vacuum and
  autovacuum truncate works", "does it also handles toast?"), spaces before commas, a missing
  terminal question mark and lowercase `postgresql`/`toast`. The asker chose **correct and
  restate**, so `## Question` reads: *Follow AGENTS.md. In PostgreSQL 17, question: how does VACUUM
  and autovacuum truncation work? Does it also handle TOAST? What changed since version 12?*
  The asker then chose **source-and-history scope**, so no 17.11 or 12.2 server was built and the
  page contains **no measurements** - every number is a constant or a formula read from source.
- **Category**: `storage-and-vacuum`. Most cited symbols are `vacuumlazy.c`, `vacuum.c`,
  `pruneheap.c` and `storage.c`, i.e. heap pages and their reclamation.
- **Core answer.** Truncation runs between index cleanup and the `pg_class` update
  (`vacuumlazy.c:518-520`), is the only part of a lazy `VACUUM` that takes `AccessExclusiveLock`,
  and removes a **suffix** only. Autovacuum reaches the same `heap_vacuum_rel` through the same
  `vacuum_rel`; the code is identical.
- **The `LP_DEAD` subtlety is the key to the design.** `heap_page_prune_and_freeze` deliberately
  does not set `hastup` for `LP_DEAD` items, on the optimistic assumption the second heap pass
  turns them into `LP_UNUSED` - without it "rel truncation will only happen every other VACUUM, at
  most" (`pruneheap.c:1513-1521`). That makes `nonempty_pages` provisional, which is why
  `count_nondeletable_pages` is documented as *necessary, not optional* and why it stops at any
  `ItemIdIsUsed` line pointer rather than at a live tuple.
- **Thresholds are OR-ed, and that has a consequence nothing in the tree states**: `>= 1000`
  trailing pages **or** `>= rel_pages / 16`, so the fraction rule governs below 16,000 blocks and
  the flat 1000-page rule above it - a 1 TB heap with 999 trailing empty pages is never truncated.
  Filed as an open question.
- **Lock behavior**: ~100 `ConditionalLockRelation` retries of 50 ms (5000/50), then
  *"stopping truncate"*; self-suspension within 20 ms of a detected waiter, checked every 32
  blocks, committing a partial truncation and looping; no `vacuum_delay_point` while the lock is
  held, so `vacuum_cost_delay` does not slow the backwards scan; and, for autovacuum only, an
  external `SIGINT` from `ProcSleep`'s `DS_BLOCKED_BY_AUTOVACUUM` branch unless
  `PROC_VACUUM_FOR_WRAPAROUND` is set.
- **TOAST: yes, with a version-sensitive twist.** A TOAST table is a heap reached by its own
  `vacuum_rel` call after the parent commits, under the parent's session lock. Since **17.6**,
  `2e0b5d252b1` (Michael Paquier, author Nathan Bossart, 2025-06-25, "Avoid scribbling of VACUUM
  options", backpatched through 13) copies `VacuumParams` at the top of `vacuum_rel` and per
  relation in `vacuum()`'s loop, so the TOAST table resolves its **own** `toast.vacuum_truncate`
  and the second relation of `VACUUM a, b` no longer inherits the first's reloption. Before that -
  and in v12, which passes the caller's pointer straight down with no copy at all - TOAST silently
  used its parent's resolved value. The same commit added the six `vacuum-truncate-*` /
  `vacuum-index-cleanup-*` injection points; their test emits four notices per `VACUUM a, b` in
  enabled/disabled/disabled/enabled order, which is the direct proof of per-relation isolation.
- **Two TOAST facts worth separating.** A main-table `vacuum_truncate` does **not** propagate:
  `extract_autovac_opts`'s TOAST-to-main fallback covers only the `AutoVacOpts` sub-struct, and
  `vacuum_truncate` lives in `StdRdOptions` outside it. And autovacuum never recurses into TOAST -
  `do_autovacuum` says so explicitly ("we don't automatically vacuum toast tables along the parent
  table") and schedules them in a second `pg_class` pass keyed on `RELKIND_TOASTVALUE`.
- **Since v12: 16 changes**, each attributed to a commit in this checkout and to its first
  containing release tag. Behavioral: the `old_snapshot_threshold < 0` condition disappeared with
  the "snapshot too old" feature (v17 `f691f5b80a8`) - v12 refuses to truncate *at all* when that
  feature is enabled; the wraparound failsafe became a new veto (v14 `1e55e7d1755` +
  `60f1f09ff44`, globalized v16 `71a825194fd`); `pg_usleep` became `WaitLatch` with the
  `VacuumTruncate` wait event (v15 `70685385d70`); `PROCESS_TOAST` (v14 `7cb3048f38e`) and
  `PROCESS_MAIN` (v16 `4211fbd8413`) arrived; `toast_parent` moved TOAST privilege checks to the
  parent (v17 `ecb0fd33720`); and 17.6 fixed the scribbling. `RelationTruncate` separately gained
  batched three-fork truncation (v13 `6d05086c0a7`), `RelationPreTruncate` (v13 `c6b92041d38`) and
  a critical section plus checkpoint interlock (v15 `412ad7a5563`, extended 17.3 `d4ffbf47b2d`,
  with `66aaabe7a18` explaining the `smgrtruncate2` spelling). Reporting/refactor: `VacOptValue`
  (v14 `3499df0dee8`), `LVRelState` (v14 `b4af70cb210`), the unconditional last-page rule (v15
  `44fa84881ff`), the truncate error context (v13 `b61d161c146`, v14 `7e453634bb6`), log text (v15
  `b175b9cde72`, `872770fd6cc`) and line-pointer-array truncation as a separate meaning of the
  word (v14 `3c3b8a4b268`, v15 `10a8d138235`). Both `TRUNCATE` the option and the
  `vacuum_truncate`/`toast.vacuum_truncate` reloptions already existed in v12 (`b84dbc8eb80b`,
  `119dcfad988`).
- **Unchanged since v12**: `REL_TRUNCATE_MINIMUM` 1000, `REL_TRUNCATE_FRACTION` 16, the 20/50/5000
  ms timing constants, `ConditionalLockRelation` rather than a blocking upgrade, the relation-grew
  bailout, the `ItemIdIsUsed` test, the 32-block prefetch window, the `truncating heap` progress
  phase, and leaving `reltuples` alone while rewriting `relpages`.
- **Test-coverage gap recorded explicitly**: nothing in `src/test/` asserts the *"stopping
  truncate"* or *"suspending truncate"* messages, so the lock-retry, self-suspension and
  relation-grew paths are uncovered, and none of the four `vacuum-*.spec` isolation tests touches
  truncation.
- Also confirmed absent in this branch: any `vacuum_truncate` entry in `guc_tables.c`, so the
  reloption is the only lever and needs neither restart nor reload -
  `ALTER TABLE ... SET (vacuum_truncate = ...)` takes `ShareUpdateExclusiveLock` and applies at the
  next `VACUUM`.
- Bookkeeping: `wiki/index.md`, `wiki/v17/index.md`, and the v17 coverage cell plus a dated note in
  `wiki/versions.md`. `verified_by_agent` stays `not yet` - the page is a first draft, not a
  claim-by-claim re-verification - and `verified:` was not touched.
- 177 citations across 20 files verified to resolve, sit in range, match their labels' basenames
  and cite only `raw/postgres-17/`; all 21 `## Contents` anchors resolve and match document order.
- `raw/` untouched: `raw/postgres-17/` and `raw/postgres-12/` both sit at their pinned commits with
  a zero-length `git status --porcelain`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. The first run flagged a
  bare `raw/postgres-12/...` path written inside backticks in an open question; it was reworded to
  name the v12 files without a raw path, since a v17 page may not carry a v12 source reference.

## [2026-08-28] cleanup | removed the vfio VACUUM FULL measurement sandbox

- Deleted `.wiki-runtime/tmp/vfio/` at the asker's up-front instruction, taking `.wiki-runtime`
  from **23,356,794,699 to 10,672,372 bytes** and reclaiming **23,346,122,327 bytes** (21.7 GiB).
  What remains is `venv/` plus the empty `cache/`, `indexes/`, `logs/` and `tmp/` scaffold that
  `ensure_runtime_dirs()` recreates anyway.
- **A live server had to be stopped first.** `postgres` PID 69468 was serving
  `-D .wiki-runtime/tmp/vfio/data` on port 55412 with logger, checkpointer, background writer and
  walwriter attached. `pg_ctl -m fast stop` returned `server stopped` and no `postgres` process
  remained; nothing was killed mid-write.
- What went: the cluster `data/` (**11,578,576,021 bytes**) and the post-`VACUUM FREEZE`
  restore snapshot `data.snap/` (**11,578,721,973**) that let the same fixture be replayed
  seven times without a rebuild; the out-of-tree `git archive` build tree `src12/`
  (160,787,308) and the exact-pin 12.2 install (26,621,827), so **no compiled 12.2 install now
  remains**; and the harness, 475,202 bytes of `results/` (before/after snapshots, `/proc` io
  CSVs and progress samples for 13 measured commands), 924,889 bytes of `logs/`, and 6,370 bytes
  of `sql/`.
- **Every published number is in the page**, and the fixture SQL is published there in full
  under "How to reproduce", so re-measuring means rebuilding 12.2 out of tree from
  `raw/postgres-12/` and re-running it. The `/proc` poller and the `\watch` progress sampler are
  described but were not published verbatim; open question 8 on the page says so.
- `raw/` untouched: all five checkouts sit at their pinned commits with a zero-length
  `git status --porcelain` (`45b88269a35`, `a92fbdfb830`, `786db8dcf16`, `baa7b142aac`,
  `67342a14863`), because the build was made out of tree.
- No wiki page, index, version pin or verification field changed in this step.
  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-28] answer v12 | measured VACUUM FULL I/O on a 6.75 GB near-empty heap

- The asker followed up on
  [How Much I/O a VACUUM FULL Performs on a Multi-GB, Near-Empty Heap in PostgreSQL 12
  (unverified)](v12/questions/storage-and-vacuum/vacuum-full-io-on-near-empty-heap.md)
  and asked for the test to be run. The page had been **source-only by the original asker's
  scoping**, so 12.2 was built out of tree from `raw/postgres-12/` at unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (tag `REL_12_2`) and the answer now carries
  measurements. `raw/` untouched: `git status --porcelain` empty before and after.
- Prompt hygiene: the request wrote `mesure`, `hypotesis`, `agents.md`, lowercase `postgresql`,
  `i/o`, `db`, `200mil`, dropped the `so` in `to vacuum doesn't truncate the heap`, and put a
  space before a comma. The asker chose "correct and restate", so `## Question` carries a
  corrected follow-up paragraph naming every correction. Two scoping answers were taken up
  front: a narrow `bigint` row shape (it reproduces the page's own `t_len = 32` worked-example
  row) and deletion of the sandbox afterwards, with the fixture SQL published in the page.
- **The hypothesis is confirmed, exactly.** Fixture as specified: 200,000,000 rows into
  **884,956 blocks / 7,249,559,552 bytes** at **226 tuples per page** (the arithmetic the page
  predicted from `floor(8168 / (MAXALIGN(32) + 4))`, and `pageinspect` confirms `lp_len = 32`),
  `DELETE` of 199,800,000 rows keeping the physically last 200,000 so `VACUUM` could not
  truncate, then `VACUUM` (27.0 s, no truncation) and `VACUUM FREEZE` (50 ms, skipping 884,070
  already-frozen pages) leaving **all 884,956 pages all-visible and all-frozen** at unchanged
  size. `VACUUM FULL` then read **884,956 blocks with `heap_blks_hit` 0**, and `/proc/<pid>/io`
  `read_bytes` came to **7,249,559,552 - the file size to the byte** - in 2.15 s.
- **The control makes the point sharper than the measurement alone.** A plain `VACUUM` on that
  identical heap read **0 blocks in 17 ms**, skipping 884,955 frozen pages. The same visibility
  map that let `VACUUM` read nothing let the rewrite read 6.75 GB.
- **Seven `VACUUM FULL` runs.** The four comparable ones at `wal_level = minimal` agree
  byte-for-byte: `heap_blks_read` 884,956, `rchar` 7,250,034,354, `syscr` 885,019, WAL 84,720.
  `syscr` decomposes as **884,956 heap-block reads of exactly 8192 bytes** plus 63 other calls
  carrying the residual 474,802 bytes - one `pread` per block. A **warm page cache** changed
  only `read_bytes` (0 against 7,249,559,552) and the clock (1.25 s against 2.15 s); it did not
  remove one of the 884,956 reads. Cache state was controlled with
  `posix_fadvise(POSIX_FADV_DONTNEED)`, no privileges needed.
- **The extreme case.** With `vacuum_truncate = off` and every row deleted, the same 6.75 GB
  heap held **zero rows** and `VACUUM FULL` still read all 884,956 blocks - producing a
  **0-byte** new heap, because `end_heap_rewrite` writes its last page only
  `if (state->rs_buffer_valid)`.
- **WAL, decomposed.** `pg_waldump --stats=record` over the exact LSN range at
  `wal_level = replica`: 954 records, 7,349,944 bytes, of which **885 `XLOG/FPI` records** -
  exactly one per new-heap page - are 98.84%. Nothing describes the 884,956 blocks that were
  read. At `minimal` the same command logged **84,720 bytes**, 87x less.
- **Four more page claims measured.** The index rebuild read only the new 885-block heap
  (789 misses + 96 hits) with `idx_blks_read` 0, while the `CREATE INDEX` that preceded it read
  884,984 blocks - the extra 28 being the visibility map, which `index_update_stats` reads
  through `visibilitymap_count`. `VACUUM FULL` under `vacuum_cost_delay = 1ms` /
  `vacuum_cost_limit = 200` ran **2.05 s** against a `VACUUM (DISABLE_PAGE_SKIPPING)` at
  **57.27 s** over identical bytes. Immediately after commit the old relfilenode sat at
  **0 bytes with segments `.1`-`.6` already unlinked**, and vanished at `CHECKPOINT`. Every
  emptied page kept **226 of 226 line pointers** with `pd_lower` 928, which is why `pgstattuple`
  reported **88.53%** free rather than ~100%.
- **Two claims corrected on evidence.** `heap_blks_scanned` does **not** necessarily end below
  `heap_blks_total`: it ends at the last returned tuple's block plus one, measured at exactly
  884,956 where the last block holds live rows and at **0** on the rowless heap. During the scan
  it is as bad as the page said - **0 of 884,956 across 49 of 51 samples**, and 0 across all 94
  samples on the rowless heap. And the doc's "extra disk space approximately equal to the size
  of the table" is off by a factor of **0.0001** here: 7,249,920 bytes against 7,249,559,552.
- Open questions rewritten 6 -> 8: the three the run settled are gone, and TOAST (never built),
  unfrozen hint bits, the single row width and scale, the fast local storage, and the deleted
  sandbox are named as what remains. Citations **307 -> 362**, all re-checked by a throwaway
  checker for path existence, range bounds, version match, and label-versus-range agreement
  (including labels that name an enclosing function whose body contains the cited slice):
  **zero problems**. All 40 `## Contents` anchors were checked to match the headings in order,
  and every page-internal link resolves.
- `wiki/index.md`, `wiki/v12/index.md` and `wiki/versions.md` (v12 coverage cell plus a dated
  note) updated. The page keeps `verified: false` (human-only) and sets
  `verified_by_agent: claude-opus-5-max 2026-08-28T12:11:54Z`.

## [2026-08-27] review v12 | VACUUM FULL I/O on a near-empty heap, plus the bookkeeping it never got

- Reviewed
  [How Much I/O a VACUUM FULL Performs on a Multi-GB, Near-Empty Heap in PostgreSQL 12
  (unverified)](v12/questions/storage-and-vacuum/vacuum-full-io-on-near-empty-heap.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2, tag `REL_12_2`), with
  corrections made **in place**. Scope confirmed with the user up front: **source-only
  re-verification**, matching the original asker's scoping, so no 12.2 server was built.
- **The page was an orphan.** It existed as an untracked file, linked from nothing and absent
  from the log; `scripts/wiki_lint` was already reporting two warnings for it. This review
  finishes the `MANDATORY Bookkeeping` steps: entries added to `wiki/index.md` (under
  `#### Storage and Vacuum` for PostgreSQL 12.2) and `wiki/v12/index.md`, the v12 coverage cell
  and a dated coverage note added to `wiki/versions.md`, and this entry.
- **Mechanical checks first.** A throwaway checker under `.wiki-runtime/tmp/` extracted every
  citation and validated path existence, range bounds, version match, and label-versus-range
  agreement: **246 citations, 152 distinct ranges, zero problems** - no dead paths, no
  out-of-range line numbers, no cross-version citation, and every `file.ext:NN-MM` label
  agreeing with its own URL fragment. A second checker rebuilt the `## Contents` block from the
  headings: **25 entries, all anchors resolve, correct nesting, document order**. Front matter
  order, the `(unverified)` title suffix, the `storage-and-vacuum` category, and the
  `../../../../raw/postgres-12/` prefix depth were all already correct.
- **Finding 1, the observability advice was backwards.** The page said
  `pg_stat_progress_cluster.heap_blks_scanned` "advances once per block with no skipping" and
  called it "the honest number". It is not a per-block counter: the only write is inside the
  per-tuple loop of `heapam_relation_copy_for_cluster`, immediately after
  `table_scan_getnextslot` returns a tuple, as `heapScan->rs_cblock + 1`
  (`heapam_handler.c:810-821`), and **no code sets it to `heap_blks_total` when the scan ends**
  (the loop just exits into `end_heap_rewrite`, `heapam_handler.c:923-974`). Under `SnapshotAny`
  a block yields a tuple only when it holds at least one `LP_NORMAL` line pointer, so on exactly
  the shape this page is about the counter stands still across every run of tupleless blocks and
  finishes below `heap_blks_total` - its last value is one past the block of the last tuple
  returned. The section now leads with `pg_statio_all_tables.heap_blks_read`, and the read-side
  section cross-links the caveat.
- **Finding 2, "empty page" was too strong.** `PageGetMaxOffsetNumber(dp) == 0` holds only for a
  page that never held a line pointer. v12's `PageRepairFragmentation` states "It doesn't remove
  unused line pointers! Please don't change this." (`bufpage.c:471-482`), and nothing under
  `src/backend/access/heap/` touches `pd_lower`, so a page emptied by `DELETE` plus `VACUUM`
  keeps its array at its historical high-water mark; `heapgettup` walks it and rejects each
  entry on `ItemIdIsNormal` (`heapam.c:651-700`), with no visibility test, no reform and no
  rewrite call. Corrected in both the short answer and the read-side section, and added to the
  evidence map.
- **Finding 3, the published SQL could not apply its own timeouts.** Both statements opened with
  `SET LOCAL statement_timeout` / `SET LOCAL lock_timeout` and no `BEGIN`. At top level outside a
  transaction block, `ExecSetVariableStmt` calls `WarnNoTransactionBlock` (`guc.c:8115-8123`) and
  `CheckTransactionBlock` raises `WARNING: SET LOCAL can only be used in transaction blocks`
  (`xact.c:3404-3430`), leaving the following `SELECT` untimed. Both snippets are now wrapped in
  `BEGIN` / `COMMIT`, with the reason cited.
- **One arithmetic error.** The worked example's `t_len = 500` row printed
  `MAXALIGN(t_len) + 4` as 504; `MAXALIGN(500)` is 504, so the column is **508**. No derived
  column moves, since `floor(8168 / 508) = floor(8168 / 504) = 16`. The other three rows (36, 52,
  108) were correct, and the whole capacity derivation
  `floor(8168 / (MAXALIGN(t_len) + 4))` was re-derived from the `raw_heap_insert` test and
  `PageGetHeapFreeSpace`'s one-`ItemIdData` deduction and holds exactly; `MaxHeapTuplesPerPage`
  = `floor(8168 / 28)` = **291** was confirmed from `offsetof(HeapTupleHeaderData, t_bits)` = 23.
- **Two citation ranges were too narrow for the claim they carried.**
  `md.c#mdunlinkfork` 284-322 -> **284-359**: the deferred `register_unlink_segment` is line 323
  and the immediate later-segment unlink loop is 326-359, both outside the old range.
  `bufpage.c#PageGetHeapFreeSpace` 664-695 -> **664-714**: both `space = 0` returns are at 701
  and 710. Fixed in the body and in the evidence map.
- **Three precision fixes.** Writes are `ceil(live bytes / usable page bytes)` pages *including*
  the final partial page, not "plus" it (the old wording double-counted). `VacuumCostBalance`
  accrues only while `VacuumCostActive`, which needs `vacuum_cost_delay > 0`, so at the v12
  default of `0` only `VacuumPageHit`/`VacuumPageMiss` move. And the `BAS_BULKREAD` ring does not
  strictly spare the cache: `StrategyRejectBuffer` sets the rejected slot to `InvalidBuffer`
  (`freelist.c:685-704`) so the retry in `BufferAlloc` (`bufmgr.c:1122-1139`) takes a buffer from
  the clock sweep.
- **Six uncited claims now carry evidence**: `XLogIsNeeded()` as `wal_level >= WAL_LEVEL_REPLICA`
  plus `RelationNeedsWAL` (`xlog.h:177-181`, `xlog.h:159-165`, `rel.h:515-520`); the absence of
  any free-space-map or `RelationGetBufferForTuple` call in `rewriteheap.c`, whose sole FSM
  reference is the TOAST `HEAP_INSERT_SKIP_FSM` flag; the index-driven
  `systable_beginscan_ordered` chunk read in `toast_fetch_datum` (`tuptoaster.c:1904-1935`); the
  B-tree build's `maintenance_work_mem` sort budget (`nbtsort.c:442-445`); why the first plain
  `VACUUM` after the rewrite cannot skip a block (`visibilitymap_get_status` returns 0 with no VM
  fork, `visibilitymap.c:329-356` / `553-587`, feeding `vacuumlazy.c:609-636`); and
  `XLogHintBitIsNeeded()` behind open question 3 (`xlog.h:183-192`,
  `bufmgr.c#MarkBufferDirtyHint`).
- **Evidence widened where the old wording was incomplete**: the "no `visibilitymap`" and "no
  `PrefetchBuffer`" greps now also name `heapam_handler.c`, the file that actually holds the
  scan-and-copy loop (both still return zero matches there), and the one `PrefetchBuffer` call in
  `heapam.c` is placed in `heap_compute_xid_horizon_for_tuples`, outside this path. Three new
  evidence-map rows and a `heap_tuples_scanned`-equals-`heap_tuples_written` note were added, the
  latter because both come from one `pgstat_progress_update_multi_param` on `*num_tuples`.
- Everything else re-derived and left standing: `VACUUM FULL` as `cluster_rel` with `InvalidOid`
  and `AccessExclusiveLock`; `use_sort = false` so no tuplesort; one `ReadBufferExtended` per
  block with page mode off under `SnapshotAny`; `smgrextend(..., skipFsync = true)` outside shared
  buffers; the single closing `heap_sync`; relfilenode swap with dirty old buffers dropped
  unwritten; index rebuilds over the new heap; TOAST swapped by content with `rd_toastoid`; the
  32-buffer ring (`256 * 1024 / BLCKSZ`, capped at `NBuffers / 8`); cost GUC defaults
  `0`/`200`/`1`/`10`/`20`, all `PGC_USERSET`; the eleven files that call `vacuum_delay_point`
  (`analyze.c`, `vacuumlazy.c`, the three type-analyze functions, and the B-tree/hash/GiST/GIN
  (including `ginfast.c`)/SP-GiST vacuum routines), none of them in the rewrite path; and
  `pg_stat_progress_cluster` appearing in the test tree only as a `rules.out` view dump.
- Citations **246 -> 307**, all re-checked clean; open questions stay at **6**, with open
  question 1 now naming two newly identified unmeasured quantities (how far `heap_blks_scanned`
  ends below `heap_blks_total`, and surviving line pointers per page).
  `verified_by_agent` set to `claude-opus-5-max 2026-08-27T21:44:14Z`; `verified:` left `false`
  for a human.
- `raw/postgres-12/` untouched at `45b88269a35` with a zero-length `git status --porcelain`; no
  server built, no `.wiki-runtime` sandbox created beyond two throwaway checker scripts under
  `.wiki-runtime/tmp/`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-27] review v17 | GIN REINDEX COMMENT-baseline page re-verified on a second 17.11 build

- Reviewed
  [A COMMENT-Stored Baseline and Normalized Index Growth for Finding GIN Indexes That Need
  REINDEX CONCURRENTLY in PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-reindex-normalized-growth-comment-baseline.md)
  against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11), with corrections
  made **in place** at the user's instruction. Scope chosen up front: rebuild and re-run the
  published SQL plus the three named spot checks and a full citation pass, then delete the
  sandbox.
- **The page's own sandbox was gone**, deleted the same day it was filed, so 17.11 was built a
  **second** time out of tree from `raw/postgres-17/` via `git archive` into
  `.wiki-runtime/tmp/ginrev/src17/`, same `configure --without-icu --without-readline
  --with-zlib --enable-debug`, contrib included, and a fresh cluster at the page's settings
  (`shared_buffers` 512MB, `maintenance_work_mem` 256MB, `work_mem` 64MB, `autovacuum = off`).
  `postgres (PostgreSQL) 17.11`, `gin_pending_list_limit` 4096, `block_size` 8192.
- **Citations: 137, zero problems.** A purpose-built checker verified every Markdown inline link
  into `raw/postgres-17/` resolves, sits inside its file, cites only this version, and carries a
  label consistent with its own line range; 50 (now 53) distinct ranged citations were then read
  in full and checked against the claim each supports. Two initial flags were false positives of
  the checker, not the page - `maintenance.sgml#routine-reindex` names the enclosing
  `<sect1 id="routine-reindex">` opened at line 1018, and the quoted sentence at L1042-L1046 is
  verbatim - so the checker now accepts an sgml section id declared at or above the range. The
  30 `## Contents` anchors all resolve, cover every heading and match document order; front
  matter order and the `(unverified)` title rule are correct.
- **The published statements are still the tested ones.** The extractor found **exactly three**
  `wiki_gin` blocks at **41 / 18 / 76 lines**, and a re-extraction after every edit produced
  byte-identical files, so nothing in this review touched them. All three **ran verbatim**.
- **No absolute byte size could be reproduced, because the fixture SQL was never published.**
  A 600,000-row table re-derived from the page's prose built a **14,737,408**-byte index against
  the filed 16,474,112, and everything downstream moved with it. This is exactly the cost open
  question 13 predicted, and it is now stated on the page.
- **Every fixture-independent claim reproduced**: the nine-field capture with `bac` 0, `bti`
  600000, `btu` 0, `btd` 0 and the two-line human comment (`{do not drop}`, `@ 100%`) preserved
  byte for byte; the nine-field read round-trip returning only the two human lines;
  `churn_ratio` **2.0000**; `heap_tuple_ratio` **1.0000**; verdict
  `candidate: disproportionate growth`; `btu` **1200000** after re-capture; the
  `insufficient churn: not evaluated` re-evaluation; and exactly one payload after re-capture,
  confirming idempotence. The prediction identity landed at **60.90 predicted against 60.88
  measured** - 0.02 off rather than 0.00 because the rebuild came back **one page** larger than
  the baseline (14,745,600 against 14,737,408), the same one-page effect the filed run recorded
  on c10 rather than the exact return it recorded on c02 and c12.
- **The three requested spot checks.** (1) No file under `src/backend/access/gin/` calls
  `RelationTruncate`, while `spgvacuum.c:896` does - the premise holds. (2) The `reltuples`
  4-way swing reproduced **exactly**: **1000000 / 200000 / 100000 / 500000** over
  `CREATE INDEX` / `ANALYZE` / `DELETE 50% + VACUUM` / `REINDEX` on a 200,000-row table with
  1,000,000 entries. (3) The `fastupdate` pairing reproduced **exactly** where it counts:
  **310** pending pages, **515** free pages in the FSM, **4,218,880 bytes** of growth on the
  `fastupdate = on` index and **not one byte** on the `fastupdate = off` one, from equal
  14,581,760-byte bases. Those three figures are governed by `gin_pending_list_limit`, not by
  the fixture, which is the page's own claim about them. Six edge cases also returned
  byte-identical output: E2, E3 (comment follows to a new index OID with a stale `bfn`), E4
  (`rebuilt since baseline: re-capture`, firing ahead of the churn gate at `churn_ratio`
  0.0000), E6 (one lock, `ShareUpdateExclusiveLock` on the index, none on the table), E11
  (`not a GIN index: fresh_btree`), E13, E14
  (`refusing baseline for t0_gin: table reltuples is -1 (run ANALYZE first)`).
- **Finding 1, fixed: the comment-size arithmetic was wrong and self-inconsistent.** Measured on
  the server, a matrix-shaped payload is **133** bytes of JSON and the page's own printed
  example is **130**; the `@ginbase:` tag adds 9, the newline 1, the two-line human note 56. So
  the whole comment is **142** bytes bare and **199** with the note, not the filed 143 and 192 -
  and 192 contradicted the page's own E1 row of 196, which is right for a 130-byte payload. The
  sentence now derives all four figures and reconciles them with E1.
- **Finding 2, fixed: `VACUUM` is only a *conditional* writer of an index's `reltuples`.** The
  page cited `vac_cleanup_one_index`, which merely *produces* the count; the `pg_class` write is
  `update_relstats_all_indexes` (`vacuumlazy.c:3072-3099`) and is **skipped** by
  `if (istat == NULL || istat->estimated_count) continue;`. `estimated_count` is
  `vacrel->scanned_pages < vacrel->rel_pages`, passed to the AM through `ivinfo` and copied
  straight back out by GIN. Measured: carrying the same 100,000-row table one step further from
  the `500000` its `REINDEX` had written, a second plain `VACUUM` left it at **500000** and only
  `VACUUM (DISABLE_PAGE_SKIPPING)` wrote **100000**. That is a fifth value the index's own
  `reltuples` can hold - the one it held before - and a further argument for the brief's
  table-`reltuples` denominator. Three citations and two Evidence Map rows added.
- **Finding 3, filed as open question 14: one `maintenance_work_mem` row is off by one page.**
  A fresh GIN build satisfies `total_pages = n_entry_pages + n_data_pages + n_pending_pages + 1`,
  the `+ 1` being the metapage `ginvacuumcleanup` skips by starting at `GIN_ROOT_BLKNO`. Every
  other measured table on the page satisfies it (7,495,680, 82,329,600 and 246,169,600 all
  decompose exactly), and the review confirmed it with **residual 0 on six fresh builds**, in
  both the all-inline and the posting-tree regime (500 data pages) and at both 64MB and 256MB.
  The 64MB row does not: 109,371,392 bytes is 13,350 pages against 3,350 + 10,000 + 1 = 13,351.
  One of the three numbers is off by one - most likely `n_entry_pages` 3,349 - and the original
  fixture is gone, so it is flagged rather than guessed. The 32.8% conclusion is unaffected
  either way, and the page now says so.
- **Everything else checked out.** All internal arithmetic was re-derived by hand: every `pred`,
  `reclaimed`, `isr`, `htr` and `norm` in the 12-cell table, all five accuracy bands, all six
  rows of the growth sweep, and the page-count decomposition of the linearity table (915 = 914 +
  1, 2514 = 2513 + 1, 10050 = 49 + 10000 + 1, 30050 = 49 + 30000 + 1). c10's 57.14% is exactly
  4/7. The `pgstatginindex` column name asserted in open question 3 is real. `pgstat_report_analyze`
  does increment `analyze_count` inside the same `pgstat_get_entry_ref_locked` /
  `pgstat_unlock_entry` section as the `mod_since_analyze` reset. `index.c:1740-1784` is inside
  `index_concurrently_swap` (1566-1836). All four GUCs are `PGC_USERSET`.
- Page updates: a new `### Re-verification on a second 17.11 build` section with a
  filed-against-re-measured table, the two fixed claims, the new open question 14, a
  `## Contents` entry, a `## Context Reviewed` bullet, a `## Test methodology` bullet, three new
  Evidence Map rows, `vacuumlazy.c` added to `## Source References`, and open question 13
  updated to record that this review's sandbox was deleted too. Open questions 13 -> 14.
  `verified: false` untouched (human-only); `verified_by_agent` refreshed to
  `claude-opus-5-max 2026-08-27T15:31:08Z`.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated. `raw/postgres-17/` is
  unchanged with a zero-length `git status --porcelain` at
  `786db8dcf168bd9df8f55047337525ac19118b1c`, because the build was made out of tree.
- Sandbox `.wiki-runtime/tmp/ginrev/` **deleted** after the review at the user's instruction,
  reclaiming **2,097,872,850 bytes** (1.95 GiB) and taking `.wiki-runtime` from 2.11 GB back to
  **10,659,287 bytes**, the venv alone. The cluster was shut down first with `pg_ctl -m fast
  stop` (`server stopped`, PID 80475 plus checkpointer, background writer, walwriter and logical
  replication launcher; no `postgres` process remained and `data/postmaster.pid` was gone), so
  no `rm` touched a running data directory. Breakdown: `data/` 1,593,208,319 (the `pubsql` and
  `spot` databases), the out-of-tree build tree `src17/` 407,271,774, the exact-pin install
  97,292,483, and 92 KB of harness - `check_citations.py`, `check_anchors.py`,
  `dump_citations.py`, `extract_published_sql.py`, the extracted statements and the result sets.
  **No compiled 17.11 install remains again**, so a third rebuild from `raw/postgres-17/` would
  be needed to re-measure. Unlike the fixture SQL, the four review scripts are generic and
  cheap to rewrite; what stays unpublished is the page's own fixture builder, which is what
  blocked byte-exact reproduction this time and is now named in open question 13.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-27] review v17 | index entry count from the catalogs, re-measured on a rebuilt cluster

- **Reviewed** [Reading an Index's Entry Count From the Catalogs, for Every Index Type, in
  PostgreSQL 17 (unverified)](v17/questions/indexing/index-entry-count-from-catalogs.md)
  against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11), the page filed
  earlier the same day. Corrections were made **in place**.
- Prompt hygiene: the request read `follow agents.md , in postgresql 17 , review question:
  ...` — `agents.md` for AGENTS.md, lowercase `postgresql`, and a space before each comma.
  The asker chose "correct and restate", so `## Question` now carries a
  `### Review prompt, 2026-08-27` block with the corrected prompt and the corrections named.
  Three scoping answers were taken up front and are filed there: audit the citations **and**
  re-measure on a server, correct **in place**, and recreate the test cluster then **delete
  it** when the review is filed.
- **The environment had to be half-rebuilt.** `.wiki-runtime/tmp/idxent/` still held the
  full harness (11 SQL files plus the citation checker) and a working exact-pin install
  (`postgres (PostgreSQL) 17.11`, `server_version_num` 170011), but the data directory and
  build tree were gone, so the cluster was re-`initdb`'d and the fixtures were rebuilt in
  two fresh databases.
- **Source: 125 of 125 citations clean.** Every distinct `(file, start, end)` resolves,
  sits in bounds, and contains the code, comment or documentation text the claim attributes
  to it — re-read individually for the catalog headers, the seven AMs' build callbacks and
  `amvacuumcleanup` paths, `index_update_stats`, `update_relstats_all_indexes`, `analyze.c`'s
  three writers, the sentinel sites in `relcache.c`/`heap.c`/`index.c`, and all three SGML
  files. **Three citation labels disagreed with their own line ranges** and were fixed:
  `pg_class.h:62-70` -> `62-69`, and `nbtsort.c:585-600` -> `576-600` in both the body and
  the Evidence Map. `scripts/wiki_lint` cannot see either class of defect.
- **The published statement is byte-identical to the tested one.** 68 lines, compared line
  by line against `32_statement_v3.sql` after stripping that file's header comment.
- **Reproduction: every measured figure came back byte for byte except three that cannot.**
  14 of 14 exact in one database and 14 of 14 again in a second built from scratch, with
  every value identical (580766 / 597901 / 1178667 / 30000 / 150000 / 116 / 200000, plus the
  on-disk truths 2000 and 150379); 15 of 16 at 180,000 rows with the same partial-BRIN miss
  at `12` against `116`, delta `-104`, error `-89.655%`; the 522874 / 538110 / 1060984 GIN
  truths; the 100 / 98 / 198 metapage `n_entries` against 200000 `reltuples`, with 100
  distinct array elements and 98 distinct lexemes measured directly; the seven forgery
  verdicts and the three `VACUUM (DISABLE_PAGE_SKIPPING)` repairs to 180000 while forged
  btree and hash stayed at 777777; `115` against 116 summaries on a **3702**-block heap whose
  range 115 starts at block **3680**; 116 on disk for the partial BRIN with **104** empty
  summaries; 999 / 1000 on the partition leaves; `0` on an empty-table build while the table
  read `-1`, `1000` after a rebuild, `-1` after `TRUNCATE` plus `REINDEX`, `0` on the
  partitioned index and `0` on the invalid index left by a failed
  `CREATE UNIQUE INDEX CONCURRENTLY`; and **20,000,001** GIN entries reading back as
  `2e+07` / `20000000`, delta `-1`, with 20,000,000 exact.
- **The three that moved are now labelled as run-specific, which is a correction to how the
  page read.** The parallel BRIN count is nondeterministic by construction: a second
  five-point `max_parallel_maintenance_workers` sweep returned **116, 221, 328, 305, 326**
  against the filed 116, 222, 325, 328, 311, and three four-worker builds returned **326,
  330, 325** against 317, 319, 328, so the page now gives the range 305-330 and notes the
  on-disk count read 116 in all ten sweep builds. The two `ANALYZE` cells for *partial*
  indexes are sample-derived through `tupleFract = numindexrows / numrows`: measured
  **140142** and **18444** against the filed 139494 and 17736 (and 140478 / 17958 on a later
  pass), so both cells are marked `(sampled)`. The provenance section's clock times
  (`08:02:47` against `07:58:38`) were replaced by the invariant the review reproduced —
  `vacuum_looks_newer = true` twice, once with BRIN unchanged at 180000 and once with btree
  and hash left at 180000 while BRIN moved to 115.
- **Four findings were added because the review measured them.** (1) A BRIN index on an
  **empty** table reads `reltuples = 1`: the serial build's final `form_and_insert_tuple`
  has no empty-range guard, unlike the parallel `form_and_spill_tuple`, so it counts an
  empty final range — measured against one empty-range summary on disk and a zero-block
  heap. The page's "serial builds exclude empty ranges" heading was wrong on attribution and
  is now "empty ranges are backfilled without being counted". (2) Every access method
  allocates at least one page during `ambuild`, measured on an empty table as `relpages` of
  1 (btree, GiST, bloom), 2 (GIN), 3 (SP-GiST, BRIN) and 4 (hash), so the statement's
  `zero: no storage yet` branch is **unreachable** for a valid non-partitioned index — the
  two shapes that do read `relpages = 0` are caught by the earlier `relkind = 'I'` and
  `NOT indisvalid` branches, and an index on an empty table therefore reports `plausible`
  with `entry_count = 0`. (3) A **non-partial** BRIN undercount exists: a heap whose rows
  were all deleted without a VACUUM read `1` against **8** summaries on disk over 256 blocks
  at `pages_per_range = 32`. (4) That same fixture left `pg_class.relpages = 0` on a table
  holding 256 real blocks, so the `ct.relpages > 0` guard silently disabled the BRIN bound
  and `brin_max_ranges` came back NULL.
- **A verdict census was added**, because the page described the statement's bounds without
  ever saying which branches fire. Run over 20 indexes in one database and 16 in another,
  eight of nine branches fired, and only two *prove* an overwrite: the BRIN bound caught all
  three BRIN indexes after `ANALYZE` (180000, 180000 and 17790 against a bound of 116), and
  the GIN bound caught `i_gin_multi` (180000 against a 360000 floor). The GIN bound needs
  `indnatts >= 2` — a single-column GIN index's floor is one entry per row, exactly what an
  overwrite writes, so `suspect` is the strongest verdict available there — and hash at
  150000 against a 200,000-row table was correctly left `plausible`.
- **GUC apply scopes were added** for every setting the page sets, per `MANDATORY GUC
  Changes`: `statement_timeout` and `lock_timeout` are `PGC_USERSET` (session/transaction),
  `max_parallel_maintenance_workers` is `PGC_USERSET`, and `autovacuum` is `PGC_SIGHUP`
  (reload), each cited to `guc_tables.c` and added to Source References.
- **Open question 2 narrowed with history**: the parallel BRIN count is written by
  `b4375717147` ("Allow parallel CREATE INDEX for BRIN indexes", first tag `REL_17_BETA1`),
  whose own commit message describes the leader merge that makes the number wrong, and
  `bs_numtuples` is touched by no other commit reachable from this pin except the original
  BRIN commit `7516f525941`. Open question 3 now carries the two-run 305-330 bracket, 8
  carries the `relpages >= 1` caveat for out-of-core AMs, and a new question 9 records that
  the two `impossible` verdicts were exercised on one fixture set with no false positive
  constructed. Open questions 8 -> 9.
- Page mechanics re-checked mechanically: 26 of 26 `## Contents` entries resolve and match
  document order, all 27 page-internal anchors resolve, no citation label disagrees with its
  URL, no Obsidian wikilinks, front-matter field order correct, and the title still carries
  `(unverified)` with `verified: false`. Category placement is unchanged and correct
  (`questions/indexing/`).
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. `wiki/index.md`,
  `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is unchanged and clean; the page
  keeps `verified: false` with `verified_by_agent` refreshed to
  `claude-opus-5-max 2026-08-27T13:07:45Z`.
- Environment: the recreated cluster under `.wiki-runtime/tmp/idxent/data` was **deleted**
  at the asker's instruction once the review was filed, so the exact-pin install and the
  harness remain but re-verification means another `initdb` plus a fixture rebuild.

## [2026-08-27] answer v17 | reading an index's entry count from the catalogs, for every index type

- Filed `wiki/v17/questions/indexing/index-entry-count-from-catalogs.md` against unchanged
  pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11), answering how a SQL query reads
  an index's entry count from the catalogs for all index types. 125 distinct source
  citations, all machine-checked to resolve to a non-blank line range in the pinned
  checkout.
- Prompt hygiene: the request wrote `agents.md` for AGENTS.md, lowercase `postgresql`,
  `sql` for SQL, spaces before commas, `all indexes types` for `all index types`, omitted
  the main verb, and phrased the object as `the number of tuples from the heap are indexed
  by the one index`. The asker chose "correct and restate", so `## Question` carries the
  corrected text and `### Prompt corrections` names every change. Three scoping answers
  were taken up front and are filed: report the **AM entry count** rather than the heap-row
  count and explain the divergence; **build and measure** rather than answer from source
  alone; and restrict the statement to **catalogs plus `pg_stat_*` views**, with
  `pageinspect` used only as ground truth.
- **The answer is one column.** `pg_class.reltuples` on the index's own row is the only
  catalog column in 17.11 that can hold this count. Ruled out with line numbers:
  `pg_class.relpages`/`relallvisible` (page counts), `pg_index.indnatts`/`indnkeyatts`
  (column counts), all four `pg_am` columns, `pg_statistic.stadistinct` (a per-column
  distinct estimate that can be a negative multiplier, even on the rows `ANALYZE` writes
  keyed on an index OID), and both extended-statistics catalogs. No system view selects
  `reltuples` at all, and `pg_stat_all_indexes` has no stored-entry column -
  `idx_tup_read` is documented as entries *returned by scans*.
- **The unit is set by the access method, and the units disagree by orders of magnitude.**
  B-tree counts heap TIDs before deduplication (200000 catalog against **2000** on-disk
  index tuples on a 200-duplicates-per-key index, a factor of 100; 200000 against 150379 on
  a 25%-NULL column); hash omits every NULL-keyed row via `_hash_convert_tuple`'s early
  return (150000 against B-tree's 200000 on the same column); GIN sums extracted entries
  over rows **and** indexed columns (1178667 = 580766 + 597901 exactly), with one
  placeholder entry for NULL and one for `'{}'` (both measured at 30000); BRIN counts
  summarized page ranges (116 against 200000 rows); GiST, SP-GiST and contrib bloom count
  one entry per row.
- **Accuracy: 14 of 14 byte-exact** against `pageinspect` and plain-SQL ground truth
  immediately after a serial rebuild, across all seven access methods, then **reproduced
  byte for byte from scratch in a second database** - 580766, 597901, 1178667, 150000, 116,
  200000 and 30000 all identical. A second pass at 180,000 rows scored 15 of 16, the one
  miss being a partial BRIN index.
- **Three writers disagree and `ANALYZE` wins.** `CREATE INDEX` writes
  `IndexBuildResult.index_tuples`; VACUUM writes `num_index_tuples` only when the AM
  returned a struct and did not set `estimated_count`; `ANALYZE` writes
  `ceil(tupleFract * totalrows)` for **every** index unconditionally, because it is the only
  one of the three that never consults the access method. Measured over six lifecycle
  events: the first VACUUM takes GIN from 580766 to 200000 (the heap row count, per
  `ginvacuum.c:738`'s own `XXX` comment), and one `ANALYZE` takes BRIN from 116 to 180000
  and re-adds to hash the 40,000 NULL rows it never indexed. `DELETE` alone and
  `VACUUM (INDEX_CLEANUP OFF)` change nothing.
- **A forgery probe decides provenance per access method.** Every index's `reltuples` was
  set to 777777 and a plain VACUUM with nothing dead was run: btree, hash and GIN kept the
  forgery; GiST, SP-GiST, BRIN and bloom repaired it. Re-forging and running
  `VACUUM (DISABLE_PAGE_SKIPPING)` then repaired GIN as well, pinning the behaviour exactly
  to `estimated_count = vacrel->scanned_pages < vacrel->rel_pages`. So one VACUUM can leave
  a B-tree stale, overwrite a GIN with the wrong quantity, and recompute a BRIN, in one
  command - provenance is per index, never per table.
- **Four different numbers for one BRIN index**, against 116 range summaries on disk:
  **116** serial, **319** parallel, **12** for a partial index, **115** after a VACUUM. Each
  traced to a line: the leader copies `brinshared->indtuples` into `bs_numtuples` and then
  inserts merged ranges with `brin_doinsert` without incrementing, so the parallel figure is
  the number of pre-merge worker slices - nondeterministic, measured at 116/222/325/328/311
  over 0/1/2/4/8 workers and 317/319/328 across three four-worker builds of the same index;
  `brin_fill_empty_ranges` never touches the counter, so empty ranges are uncounted; and
  `brinvacuumcleanup` passes `include_partial = false`, so the trailing short range is
  skipped. This refines the mechanism already noted on
  `non-btree-index-inflation-comment-baseline.md` without contradicting it.
- **The published statement is byte-identical to the one tested** (68 lines, verified
  mechanically) and carries only catalog-decidable verdicts: two hard bounds - BRIN cannot
  exceed `ceil(table relpages / pages_per_range)`, GIN cannot fall below `rows x indnatts` -
  plus a `suspect` flag for equality with the table row estimate, and `n/a` branches for
  partitioned and invalid indexes. It correctly labelled all 22 indexes in the fixture set,
  with 2 documented false positives (single-column GIN indexes whose entry count
  legitimately equals their row count) and one undetectable case (the partial BRIN
  undercount, which is below the bound).
- **A timestamp-ordering design was built, disproved and dropped.** `VACUUM
  (INDEX_CLEANUP OFF)` advanced `last_vacuum` to 08:02:47 against `last_analyze` at
  07:58:38 while BRIN kept the `ANALYZE` value, and a following plain VACUUM left btree and
  hash at 180000 while moving BRIN to 115. The filed statement reports `analyzes` and
  `vacuums` as context only.
- Four further measured catalog facts: GIN's metapage holds an accurate **distinct-key**
  count (100 / 98 / 198, the third defensible GIN entry count) that never reaches
  `pg_class`; a newly created index reads **0** while its table reads **-1**, because
  `RelationBuildLocalRelation` `palloc0`s `rd_rel` and `index_create` skips the
  `reltuples = -1` step tables get in `AddNewRelationTuple` - measured on an empty-table
  index, a partitioned index and the invalid index left by a failed
  `CREATE INDEX CONCURRENTLY`, so `0` is ambiguous between empty and never-built; `-1`
  appears only after a storage reset (`TRUNCATE` plus `REINDEX`); and `float4` loses the low
  bit above 2^24, reading 20000000 for a measured **20,000,001** GIN entries.
- Eight open questions filed, including a doc-versus-source discrepancy resolved in favour
  of source per AGENTS.md: `indexam.sgml` says `amvacuumcleanup` statistics "will be
  reported by `VACUUM` if `VERBOSE` is given", but the `ereport` carrying
  `num_index_tuples` sits in `vac_cleanup_one_index` at `DEBUG2` at every reaching call
  site, and v17's `VACUUM VERBOSE` per-index line is page-only. The page therefore makes no
  VERBOSE claim. Also open: no upstream commit was identified for the parallel BRIN
  overcount, only one `pages_per_range` and one table size were tested, GIN opclasses beyond
  `array_ops`/`tsvector_ops` were not scored, SP-GiST has no `pageinspect` support so its
  ground truth is weaker than the other six AMs, and nothing was tested under concurrency.
- Environment: an isolated 17.11 server built VPATH out of tree from `raw/postgres-17/`
  (`raw/` left untouched, verified clean) under `.wiki-runtime/tmp/idxent/`, with
  `autovacuum = off`, 22 indexes over seven access methods, and two databases. The sandbox
  is retained, holding the fixtures, the truth views, the stage/forgery/edge probes, both
  statement revisions and the citation checker.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. All 23
  `## Contents` anchors verified to resolve, in document order, with none missing.
  `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is unchanged;
  the page keeps `verified: false` with
  `verified_by_agent: claude-opus-5-max 2026-08-27T12:13:11Z`.

## [2026-08-26] follow-up v17 | the GIN waste statements on PostgreSQL 12, and the whole corpus on both majors

- Extended [Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib
  Extensions on PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-index-wasted-space-contrib.md) with a third
  follow-up against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11):
  make all four published statements run on v12, and run every test on both majors.
- Prompt hygiene: the asker chose "correct and restate". The original read
  `follow agents.md , in postgresql 17 , for question : ... , add follow up: make sure
  that statement run on v12 , run all test on v12 and v17` - `agents.md` for AGENTS.md,
  lowercase `postgresql`, spaces before commas and the colon, `for question :`,
  `add follow up:`, `that statement run`, `all test`. Three scoping answers were taken
  up front: all four statements (not just the census), the whole corpus (not a
  portable subset), and both sandboxes deleted on filing.
- New provenance: **two** isolated clusters built out of tree from this repo's own
  pins - 12.2 (`server_version_num` 120002, port 55412, `pageinspect` 1.7) and 17.11
  (170011, port 55417, `pageinspect` 1.12) - `--without-readline --without-zlib`,
  `block_size` 8192, `autovacuum = off`, `fsync = off`, `shared_buffers = 256MB`. The
  evidence trees were untouched; both sandboxes were deleted when the page was filed.
- **Three edits make the statements portable.** `get_raw_page` takes `int4` on 12.2
  (`f18aa1b2039` widened it to `int8`, earliest tag `REL_14_0`), so the block number
  needs `::int`; the census needs a `pagesize = 0` class ahead of the flags tests; the
  probe needs the same guard on its `'{}'` filter. The FSM cross-check and size bracket
  run unchanged and derive 8160 on both. `page_header`'s widths are `smallint` on 12.2
  and `int` on 17.11 (`127404fbe28`, `REL_15_0`), which is harmless at 8 kB pages.
- **The decisive difference is all-zero pages.** 12.2 returns `flags = {}` where 17.11
  returns a NULL row; the fix `cd4868a5700` is `Backpatch-through: 10` and its 12-branch
  backport `5378d55cb2f` is first tagged `REL_12_11`, after the 12.2 pin. With two
  zeroed blocks appended to an index file, the filed text reads 56 entry / 0 new /
  0 waste bytes on 12.2 against 54 / 2 / 16,384 on 17.11, and the probe's downlinks read
  41 against 53. A `dd`-zeroed metapage aborts the entire census on 12.2. `-m immediate`
  mid-insert produced no zeroed pages on either server, so the earlier 4-and-7 result
  did not reproduce.
- **26 of 27 scored fixtures are byte-identical across the majors**, including all seven
  published fixtures reproducing every filed figure, with identical tallies: lower bound
  24 of 27, upper 25 of 27, upper-plus-pending 26 of 27. Identical as well: the `f2`
  lifecycle in all four states, the three-VACUUM horizon sequence (`prune_xid` = the
  holder's `backend_xmin`, 768 recycled on the next VACUUM after release with no new
  xids), the five flush rounds and their FSM readings, the privilege matrix, both
  timeouts, the standby refusals, the single-read proof (1125 against 2230 buffers on
  17.11, 1122 against 2227 on 12.2) and rebuild equivalence.
- **Two headline claims corrected.** `waste + slack` is not an upper bound with a large
  dead-key population (an 800k-row `jsonb_path_ops` index with 819,770 dead entry tuples
  read 43.43% against 50.00% reclaimed) nor with a live pending list (13.73% against
  40.00%, repaired by adding `pending_pct`; the settled twin behaves). And a census does
  evict a hot working set once the target exceeds the cache - at 8,192 buffers a
  9,616-block census took a 1,862-page hot set from `usagecount` 3 to 80 pages on 17.11
  and to 0 on 12.2, where a 16k-block seq scan left it intact.
- **A rebuild is not one number.** The same index rebuilt to 34,611,200 bytes at
  `maintenance_work_mem` 64MB and 50,814,976 at 96MB and above, on both majors, 47%
  apart, with payload constant near 25.6 MB and all the movement in `entry_slack`;
  `ginBuildCallback` spends a budget measured with `GetMemoryChunkSpace`, and 17.11 fits
  about 7% more per flush (v12 at 68MB and 70MB bracket v17 at 64MB), which is the sole
  reason the 27th fixture differs. Not parallelism: 0, 2 and 4 workers give the same
  bytes.
- **Open question 7 closed**: the round-five entry-tree explosion is slack exhaustion -
  entry slack falls by an almost constant 395 kB a round (2,126,732 -> 538,900) with the
  tree pinned at 572 pages, then 255 splits arrive at once.
- Other differences filed: 12.2's `pgstatginindex` answers for an invalid index where
  17.11 refuses (`13503eb5905`, `REL_17_0`), 12.2 prints the VACUUM cross-check as one
  deleted-pages number inside a `DETAIL` block with no "newly deleted" split, three
  refusal messages are worded differently, and v13+ picks up an insert-only table through
  `autovacuum_vacuum_insert_threshold`, which 12.2 does not have. The race-detector
  ranking is corrected: it depends on the writer, not the version (metapage check 23 of
  25 under four `fastupdate` writers where the size bracket caught 4).
- Page updates: Contents gained eleven entries; the Short answer gained a qualifying
  paragraph; the census and probe now carry the portability edits inline; eleven new
  `###` sections carry the port, the all-zero-page analysis, the corpus comparison with
  the reconstructed fixture SQL, the two new bound failures, the `maintenance_work_mem`
  result, the VACUUM wording, the refusal differences, the identical-behaviour list, the
  concurrency correction, the flush cascade and the eviction re-measurement. Reading
  rules gained seven bullets, Context Reviewed three, Evidence Map eighteen rows,
  Open Questions went 10 -> 15 (one closed, five new), and Source References gained
  seven entries.
- All 302 markdown source citations re-checked for in-bounds line ranges against the
  pinned checkout: zero problems, no wrong-version references, no wikilinks.
  `verified_by_agent` refreshed to `claude-opus-5-max 2026-08-26T16:30:12Z`; the page
  keeps `verified: false`. Root index, `wiki/v17/index.md`, the v17 coverage cell and a
  new `## Coverage Notes` entry all describe the follow-up.
- Cleanup: both servers were stopped and `.wiki-runtime/tmp/ginv/` removed at the
  user's instruction, reclaiming **13,569,785,002 bytes** (13 GiB) and leaving
  `.wiki-runtime/tmp/` empty. Both pinned checkouts are unmodified
  (`45b88269a35`, `786db8dcf16`, zero-length `git status --porcelain`), because both
  builds were made out of tree.

## [2026-08-26] cleanup | removed the ginw2 GIN sandbox from .wiki-runtime/tmp

- Removed `.wiki-runtime/tmp/ginw2/` at the user's request, reclaiming exactly
  **4,248,710,531 bytes** (4.0 GiB) and leaving `.wiki-runtime/tmp/` empty.
  `.wiki-runtime/` is now 13 MB, all of it `venv/`. This is the sandbox behind
  [Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib Extensions on
  PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-index-wasted-space-contrib.md), which the
  entry below had retained for re-verification.
- Disposable cluster data accounted for nearly all of it: `data/` 3.6 GB (the six
  databases `ginw`, `ginw3`, `ginw4`, `ginw5`, `ginw6`, `ginw7`) and the
  `pg_basebackup` standby `data_sb/` 319 MB. The rest was the out-of-tree VPATH build
  tree at 74 MB and the exact-pin 17.11 install at 35 MB, so **no compiled 17.11
  install now remains** and the next exact-pin GIN experiment must rebuild from
  `raw/postgres-17/`.
- The 860 KB of harness, results and server logs went with it, on the user's
  instruction — the 44 harness files under `work/` and 45 result sets under `results/`
  inventoried in the entry below, plus 33 run logs under `logs/`. The cost of this is low
  because that pass published its **fixture SQL inside the page** and then proved the
  published SQL reproduces every filed number byte for byte in two virgin databases,
  so the page is now its own reproduction recipe. That was not true of the earlier
  `ginw` sandbox, whose deletion forced a rebuild *and* a re-derivation of unpublished
  fixtures.
- Nothing was running. No postgres, `initdb` or `pg_ctl` process existed, `sock/` held
  no socket, and neither `data/` nor `data_sb/` had a `postmaster.pid` — the previous
  pass had already stopped the server and returned `shared_buffers` to 256MB. No
  `pg_ctl stop` was needed and no cluster was killed mid-write.
- Also cleared three stale tooling files: `logs/recent_log.log`, `logs/wiki_lint.log`
  and `cache/wiki_lint/last-run.txt`. Kept `venv/`, and kept the empty `cache/`,
  `indexes/{ctags,search,tree-sitter}`, `logs/` and `tmp/` scaffold, which
  `ensure_runtime_dirs()` recreates on every script run anyway.
- **Three stale sandbox pointers corrected**, the same failure this page already hit
  once: `wiki/versions.md` said `.wiki-runtime/tmp/ginw2/` "is retained" in the
  2026-08-26 note and the 2026-08-25 note, and the page's own `## Context Reviewed`
  described it in the present tense. All now say it was deleted and that reproduction
  means rebuilding from `raw/postgres-17/`. No source citation, pin, measurement, or
  verification field on any page changed; the page keeps
  `verified_by_agent: claude-opus-5-max 2026-08-26T15:02:41Z`, because every claim it
  makes rests on pinned source or on measurements it publishes, not on the sandbox
  still existing.
- `raw/` untouched. `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors,
  0 warnings.

## [2026-08-26] follow-up v17 | the GIN waste page's open questions attacked with measurements, 11 -> 10

- Third pass over [Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib
  Extensions on PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-index-wasted-space-contrib.md), unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11), on the retained
  `.wiki-runtime/tmp/ginw2/` sandbox (restarted, no rebuild needed). Open questions went
  **11 -> 10**: two closed outright, one new, six materially narrowed. Fourteen new
  fixtures, 251 source citations re-checked for in-bounds line ranges, `scripts/wiki_lint`
  clean.
- Prompt hygiene: the request read `follow agents.md , in postgresql 17 , for question :
  ... , review open questions` — `agents.md` for AGENTS.md, lowercase `postgresql`, a
  space before each comma and the colon. The asker chose "correct and restate", chose to
  attack the questions with measurements rather than audit them, and **ruled out the two
  expensive probes**: no second build at another `BLCKSZ` (open question 3) and no
  repeated crash runs (open question 10). Both are recorded as out of scope by request.
- **Reproducibility closed.** The page's published fixture SQL, re-run unmodified in two
  freshly created databases, returned every filed number byte for byte: the seven sizes
  (17,571,840 / 6,209,536 / 10,117,120 / 16,384 / 7,356,416 / 7,356,416 / 11,927,552),
  every page-class count, `entry_slack` and `data_slack` to the byte, the `f2` lifecycle
  states, the three-VACUUM `f5` sequence including its B-tree sibling line, and all seven
  `REINDEX` results (42.42 / 58.05 / 0.00 / 0.00 / 89.09 / 46.33 / 13.26%). The only
  figure that moved is the one the page said would: `f5`'s `prune_xid` read 953 against
  791 and 870. The churn sweep also re-ran identically.
- **`REINDEX CONCURRENTLY` closed.** Two identical fixtures under the same 100,000-row
  insert stream: plain `REINDEX` 3,276,800 -> **2,662,400** in 1901 ms, `REINDEX
  CONCURRENTLY` 3,276,800 -> **2,662,400** in 2013 ms, both ending
  `indisvalid`/`indisready`/`indislive` true with both tables at 200,000 rows; with no
  load both gave 1,548,288.
- **New probe, and the answer to the payload model's detection gap.** On a GIN entry-leaf
  page `(pd_lower - 24) / 4` is the entry-tuple count, because entry pages are ordinary
  `PageAddItem` pages. Measured against `count(DISTINCT ...)`: 50,028 = 50,028 lexemes, 32
  = 32 tags, and 53 = 50 tags plus GIN's three null categories (`GIN_CAT_NULL_KEY`,
  `GIN_CAT_EMPTY_ITEM`, `GIN_CAT_NULL_ITEM`). It separates the two churn shapes at
  **50,028 dead entry tuples against 1**, and the internal-downlink count is a free
  structural check (1,198 downlinks over 6 internal pages against 1,193 leaves = a root of
  5 plus 1,193). The probe is filed as a second published statement so the census's own
  byte-identical output claims stay intact; it ran a 26,195-block database in 146-241 ms.
- **The failure boundary is linear.** A five-point sweep holding churn volume constant
  (all 200k rows updated every time) while varying the share of the key population
  replaced: +0.09 / +12.62 / +25.16 / +37.69 / +50.19% at 0 / 25 / 50 / 75 / 100%, i.e.
  increments of +12.53, +12.54, +12.53, +12.50. **The correction still fails**: dead
  tuples are 16 bytes against live 32 (the average falls 32.0 -> 24.0 across the sweep),
  so subtracting at the average over-corrects by -7.48% to -18.44%, and -36.80% on `f1`.
  The two estimates bracket the truth on the 9 fixtures with dead tuples and coincide on
  the 11 without.
- **A second lower-bound violation, built on purpose.** Three fixtures grown entirely
  through the pending list: `fh2_gin` reads **21.66% dead against 19.25% reclaimed**,
  while `fh1_gin` holds by six blocks and `fh3_gin` holds. The bound is an identity —
  `waste <= reclaimed` exactly when the aged in-use core is at least as big as its own
  rebuild — and the density form of the test (aged-core fill against fresh fill) predicted
  the direction 3 of 3, including the sign of the near miss. `f13_btgin_gin`'s filed
  0.13-point margin is restated as what it is: one block.
- **Flush growth has a mechanism now.** Five identical 50k-row insert-and-flush rounds:
  the pending list is built from the FSM's free stock every time (0 free -> the file grew
  by exactly its 736 pending pages; 736 free -> zero growth, four rounds running), and
  every round reached its flush with `pg_freespace` reading 0, because
  `RecordPageWithFreeSpace` writes only the bottom FSM level while `fsm_search` starts at
  `FSM_ROOT_ADDRESS` and `ginInsertCleanup` vacuums the FSM only at the end — **a flush
  can never reuse the pages it is freeing**. What the merge itself wants is still
  unpredictable and is *not* a function of total slack: +194 blocks at 2,326,056 bytes of
  slack against +0 at 3,197,144 and +649 at 1,885,966. Round 5's 649-page entry-tree jump
  is filed as the one new open question.
- **A concurrent VACUUM defeats every cross-check**, which is the worst case on the page:
  14 censuses of an unchanging 2,594-block file reported 0, 0, 0, 74, 222, 444, 666, 888,
  1110, 1406, 1628, 1850, 2072, 2294 dead pages against a truth of 2,368 — 91% of the file
  — with the self-check, the metapage check and the size bracket all passing every time.
  Under four writers the ranking inverted: the **size bracket caught 13 of 14** where the
  metapage cross-check caught **0 of 14** (it caught 19 of 25 in the earlier
  insert-and-flush test only because that writer kept moving the metapage). The page's
  race advice was corrected accordingly, and "Three cross-checks" became "Four
  cross-checks". A concurrent `REINDEX INDEX CONCURRENTLY` swap landed between censuses in
  11 of 11 attempts, so the mid-census out-of-range trap is described but not reproduced.
- **The eviction conclusion is corrected, not extended.** At 16,384 buffers a 26,195-block
  census left a hot 3,704-page working set **fully resident** while destroying a
  once-read set of the same size; the hot set's `usagecount` went 5 -> 1 and a second
  census then took it. The mechanism is `PinBuffer`'s capped increment,
  `BM_MAX_USAGE_COUNT = 5` and `StrategyGetBuffer`'s decrement-and-take clock sweep. So a
  census spends usage count rather than pages, is safe for a genuinely hot set, and is
  fatal to a read-once one. The 16,303-block seq-scan control left 96 pages, as before.
- **Four more opclasses and a scale check.** `jsonb_ops` (+0.79% model error, 2 dead
  tuples), `text[]` `array_ops` (+29.23%), weighted `tsvector` (+42.79%), and
  `jsonb_path_ops` at 50,000 and 800,000 rows (+33.18% and +33.60%) — a 16x scale change
  moving the error 0.42 points, which retires scale as a suspect. Tally: **26 fixtures, 26
  of 26 upper bound, 24 of 26 lower**. Fresh-build fill now spans 50.16% to 72.11%.
- Statement changes: the FSM cross-check derives `MaxFSMRequestSize` from
  `pg_control_init()` (returning 8160, equal to the largest `avail` over 3,478 free GIN
  pages, and executable by `PUBLIC`) instead of hardcoding 8160; the size bracket is added
  as a cross-check; the entry-tuple probe is added. The main census statement is
  unchanged, so every previously filed census number still stands.
- Published SQL was re-tested verbatim: the printed churn-sweep recipe rebuilt `g100_gin`
  at exactly 15,261,696 bytes in a sixth database, and the printed FSM cross-check, size
  bracket and entry-tuple probe all ran as printed, the probe reproducing 1,713 entry leaf
  pages / 200,020 entry tuples / 4,800,336 tuple bytes / 1,721 downlinks. All 141 distinct
  file-and-range citations across 43 files were checked for in-bounds line ranges (0
  broken, 0 wikilinks), and the `## Contents` block was verified to match the heading
  order exactly.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` untouched;
  `verified:` left `false` (human-only) and `verified_by_agent` refreshed to
  `claude-opus-5-max 2026-08-26T15:02:41Z`. The sandbox is retained at
  `.wiki-runtime/tmp/ginw2/` (4.2 GB): the exact-pin install, the primary and standby data
  directories, 44 harness files under `work/`, 45 result sets under `results/`, and six
  databases — `ginw` (the review's corpus), `ginw3` and `ginw5` (the two virgin-database
  reproductions), `ginw4` (churn sweep and flush rounds), `ginw6` (violators, opclasses,
  races, eviction) and `ginw7` (published-SQL verification). `shared_buffers` was returned
  to 256MB after the eviction test and the server was stopped, leaving no process running.

## [2026-08-26] follow-up v17 | eight open questions on the GIN waste page closed by measurement

- Kept the review's rebuilt 17.11 sandbox running and answered eight of the thirteen
  open questions on [Measuring Wasted and Reclaimable Bytes in a GIN Index With
  Contrib Extensions on PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-index-wasted-space-contrib.md) with
  experiments rather than argument, on unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). Nine new fixtures, `f8` through
  `f16`, are published in the page's fixtures section; `pg_trgm` 1.6, `btree_gin` 1.3
  and `pg_buffercache` 1.5 were installed from the same pinned tree.
- **A single idle snapshot, not the transaction counter, is what pins recyclability.**
  With a `REPEATABLE READ` snapshot held in another session, `f8_horizon_gin` reported
  `768 newly deleted, 0 reusable` and then stayed at **0 reusable across two more
  VACUUMs and six consumed xids**. The instant the holder was terminated, the very next
  VACUUM — **with no new transaction ids at all** — reported `768 reusable` and the FSM
  filled. The numbers close the loop: all 768 pages carry `prune_xid = 870`, which is
  exactly the value the holder's `pg_stat_activity.backend_xmin` showed, and the
  cluster's snapshot xmax was already 882 at release. So the earlier "three xids then
  reusable" reading was a special case of the shared horizon moving, not a rule about
  xid consumption. First attempt at this test was invalid and is filed as such: killing
  `psql` leaves the backend in `pg_sleep`, so the "release" step released nothing until
  `pg_terminate_backend` was used.
- **The census's own self-check cannot see a concurrent writer.** 25 censuses of the new
  1,000,000-row `f10_race_gin` while another session pushed 5,000 rows through the
  pending list and flushed, in a loop: `census_total_pages = blocks` held **25 of 25**,
  while the census's `pending_pages` disagreed with `meta_pending_pages` in **19 of 25**
  (74 against 2, 12 against 74, 8 against 74 with 66 already `deleted`) and the block
  count went stale against the file size read immediately afterwards in **25 of 25**, by
  up to **664 blocks**. The self-check is blind by construction — both sides come from
  the single `pg_relation_size` reading in the first CTE — so the page now names the
  metapage cross-check as the thing that actually catches the race.
- **Autoanalyze alone flushed a 491-page pending list in under 10 seconds.**
  `autovacuum = on` is `PGC_SIGHUP` (`guc_tables.c:1449-1457`), so a reload sufficed;
  with only *analyze* made eager, `pending_pages` went 491 -> 0 with `last_autoanalyze`
  set and `last_autovacuum` still null, which is `ginvacuumcleanup`'s `analyze_only`
  branch. The file grew 836 -> 958 blocks and the waste reading jumped to 51.25%, so two
  censuses seconds apart can differ by 15% of the file with nobody touching the table.
  The insert before it also demonstrated reuse again: 491 pending pages landed almost
  entirely in the 490 dead pages left by the previous flush, for +1 block.
- **`REINDEX INDEX CONCURRENTLY` is the same ground truth as the plain form**, at least
  on this shape: `f9_ric_gin`, built by the identical recipe to `f6_slack_gin`, went
  7,356,416 -> **3,948,544** bytes, the same byte count plain `REINDEX` gave `f6`, in
  379 ms against 151 ms, with `indisvalid`/`indisready`/`indislive` all true, no
  `_ccnew`/`_ccold` leftovers, and a plain `REINDEX` straight afterwards returning the
  same number.
- **Five more opclasses and shapes were built, churned and rebuilt**, and both bounds
  held on all five, taking the totals to **upper bound 12 of 12, lower bound 11 of 12**:
  `jsonb_path_ops` 32.69% dead / 39.74% slack / 50.90% reclaimed, `gin_trgm_ops` 29.63 /
  47.87 / 68.98, `btree_gin` 62.71 / 18.22 / **62.84**, multicolumn 26.89 / 47.27 /
  58.83, partial 83.53 / 10.30 / 89.60. The `btree_gin` row is a **0.13-point** margin
  on the lower bound, which is now an open question in its own right. Fresh-build fill
  is firmly opclass-dependent: 51.30% (`btree_gin`) to 72.11% (`pg_trgm`), with the
  1,000,000-row six-key `int[]` at 68.98%, so "a fresh GIN build is about N% full" has
  no single value.
- **The payload-and-fill model's failure mode reproduced independently.** Four of the
  five new fixtures are predicted to +0.65%, +1.49%, +0.00% and +0.63%; the multicolumn
  one, whose churn replaced *both* key columns, misses by **+18.12%** — the same
  dead-entry-tuple mechanism as `f1`'s +19.94%. The error tracks how much of the key
  population died, not the opclass.
- **A real all-zero page was produced, twice.** `pg_ctl stop -m immediate` during a
  600,000-row insert left **4** all-zero pages in `f16_zero_gin` on the first attempt and
  **7** on the second, out of 623 and 1043 blocks, so the census's NULL-row branch fires
  on real pages and not only on a synthetic `bytea`. GIN extends through `GinNewBuffer`
  and initializes afterwards, so a crash in between leaves a block that `PageIsNew` and
  therefore `GinPageIsRecyclable` accept.
- **Eviction was measured directly**, after a first attempt at `shared_buffers = 1MB`
  proved self-defeating (the `pg_buffercache` query itself evicted the warmed table). At
  64MB / 8,192 buffers: a census of the 6,665-block `f10_race_gin` left **all 6,665
  pages resident** at `usagecount = 1` — 81% of the cache — while a plain
  `SELECT count(*)` over the 4,092-block `t1_churn` left only **98**, because a seq scan
  past `NBuffers / 4` takes a `BAS_BULKREAD` ring (`heapam.c:434-458`) and the census has
  no such limiter. Censusing four more indexes (4,238 blocks) then dropped
  `f10_race_gin` to **3,723** resident pages with 0 free buffers: the census evicts, and
  the first victims are its own earlier pages.
- Page edits: a follow-up fixture table in `### The fixtures`, a five-row opclass scoring
  table and the RIC paragraph in `### What the census meant against REINDEX`, the
  second model failure, the held-snapshot table in the horizon section, a new
  `### Concurrency: a census of a busy index is a mixed-instant reading` section, three
  new bullets in `### Cost of the census`, the crash finding in the all-zero-page design
  point, three new reading rules, two `## Context Reviewed` bullets, seven new
  `## Evidence Map` rows, and `## Open Questions` rewritten from 13 to 11 — eight
  answered, five new ones raised by the answers (the 0.13-point margin, one-workload
  race coverage, one-shape RIC, one-fixture-per-opclass, and the incidental zero-page
  count). Two new citations: `heapam.c:434-458` and `guc_tables.c:1449-1457`, taking the
  page to 123.
- `verified_by_agent` refreshed to `claude-opus-5-max 2026-08-26T14:01:59Z`; `verified:`
  is unchanged at `false`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings, plain and with
  `--warnings-as-errors`; Contents matches all 27 headings and every page-internal anchor
  resolves; all 123 citations resolve. Sandbox `.wiki-runtime/tmp/ginw2/` retained,
  `shared_buffers` restored to 256MB and `autovacuum` back to `off`; `raw/postgres-17`
  clean on `786db8dcf16`.

## [2026-08-26] review v17 | GIN wasted-space page re-verified on a rebuilt 17.11 server

- **Reviewed** [Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib
  Extensions on PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-index-wasted-space-contrib.md) on unchanged
  pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). Corrections were made **in
  place**, so the page now reads as one coherent answer whose numbers all come from
  one reproducible run.
- Prompt hygiene: the request read `follow agents.md , in postgresql 17 , review
  question : ...` — `agents.md` for AGENTS.md, lowercase `postgresql`, and a space
  before each comma and before the colon. The asker chose "correct and restate", so
  `## Question` carries the corrected review prompt and names the corrections. Three
  scoping answers were taken up front: a full review rather than source-only,
  corrections in place, and permission to correct the stale sandbox pointer and
  rebuild the sandbox.
- **The page's sandbox was gone.** `.wiki-runtime/tmp/` was empty: no
  `.wiki-runtime/tmp/ginw/` fixtures and no 17.11 install, so nothing could be
  re-measured in place. PostgreSQL 17.11 was rebuilt from `raw/postgres-17/` with a
  **VPATH build** (`configure` run from `.wiki-runtime/tmp/ginw2/build`), which leaves
  the read-only evidence tree untouched — `git -C raw/postgres-17 status` stayed clean
  — and `pageinspect` 1.12, `pgstattuple` 1.5 and `pg_freespacemap` 1.2 were installed
  from the same tree.
- **Source: all 119 citations re-read, zero broken.** Every unique
  `(file, start, end)` on the page resolves, sits inside the file, and contains the
  claimed code. Three repo-wide checks reproduced too: `RelationTruncate` and
  `smgrtruncate` have **zero** hits under `src/backend/access/gin/`, `pages_deleted` is
  incremented in exactly two places (`ginvacuum.c:235`, `ginfast.c:591`), and
  `contrib/amcheck` declares only `bt_index_check`, `bt_index_parent_check` and
  `verify_heapam`.
- Two source-side additions were filed. `freespace.c`'s own category-table comment
  works its example "assuming ... `MaxFSMRequestSize` is 8164 bytes", a value the macro
  cannot produce on this build, so the page now warns against reading 8164 out of the
  file it cites for 8160. And the `pd_lower`-is-maintained claim now cites
  `GinDataPageSetDataSize` (ginblock.h:310-314), the macro that actually writes
  `pd_lower`, rather than only its two call sites. One precision fix: "every
  `pageinspect` entry point checks `superuser()` in C" is true in effect but not
  per-function — 4 of 7 entry points in `rawpage.c` and 4 of 7 in `btreefuncs.c`
  inherit the check from a shared internal — so the sentence now says so.
- **The fixtures had never been published, so they were reconstructed and are now
  printed in the page as SQL.** Two of the reconstructions are provably right: `f5`
  (32 keys per row, first 95% of the heap deleted) and `f6` (same shape, every other
  row deleted) came out at **7,356,416 bytes / 898 blocks** with page-class censuses of
  1 / 96 / 32 / 768 and 1 / 864 / 32 / 0, `waste` 85.52% and 0.00%, slack 9.70%
  (7,520 / 706,048) and 51.34% (7,520 / 3,769,344), payload 351,392 and 3,579,552, and
  `REINDEX` results of **802,816** and **3,948,544** bytes — every one of those numbers
  byte-identical to the filed page, together with rebuilt payload fractions of 43.28%
  and 87.96% and model errors of +1.12% and +3.07%. `f4` (empty) reproduced 16,384 / 2
  / 8,160 / 49.80% / 50.20% as well. The original `f5`/`f6` were therefore
  all-32-keys-per-row fixtures, which is what the first run's "32 hot keys" meant.
- **The horizon sequence reproduced line for line**: `898 in total, 768 newly deleted,
  768 currently deleted, 0 reusable` with 0 FSM pages, then `0 / 0 / 0 reusable` on a
  second VACUUM, then `0 / 0 / 768 reusable` with 768 FSM pages at `avail = 8160` after
  three `pg_current_xact_id()` calls — and the B-tree sibling's `551 in total, 0 newly
  deleted, 520 currently deleted, 520 reusable` in the same VACUUM. 768 pages carried
  flags `{data,leaf,deleted,compressed}`, 130 blocks read `avail = 0`, and max `avail`
  over every GIN index was 8160. The one number that cannot travel between clusters is
  the delete xid: `prune_xid` read **791** here where the page said 768, which happened
  to equal its own page count, so the page now says what the value is instead of
  quoting it as a constant.
- `f1`, `f2`, `f3` and `f7` are this run's own fixtures and their figures replaced the
  first run's: `f1` 17,571,840 / 2145 blocks / 0.65% waste / 63.47% slack / 42.42%
  reclaimed, `f2` 6,209,536 / 758 / 64.64% / 10.65% / 58.05%, `f3` 10,117,120 / 1235 /
  0.00% / 48.05% / 0.00%, and the new `f7` 11,927,552 / 1456 / 9.89% / 44.26% / 13.26%.
  `f1` rebuilt to exactly `f3`'s size and page-class census, which is a self-check on
  the pair.
- **Both headline conclusions survive.** `waste + slack` bounded the reclaimed bytes
  from above in **7 of 7** fixtures, and `waste` alone was not a lower bound in 1 of 7:
  `f2` held 64.64% dead pages while `REINDEX` returned 58.05%, its 268-block in-use core
  rebuilding to 318 blocks (18.7% larger). The first run's figures for the same claim
  were 65.49% against 50.07% and 44.7%.
- **One filed claim did not reproduce and was rewritten.** The first run's pending-list
  flush grew the file by exactly one block; this run's flush left it at **6,209,536
  bytes, unchanged**, while still turning 490 of 758 blocks into dead pages and cutting
  entry slack from 575,940 to 318,084 with the entry tree fixed at 170 pages. The page
  now states the mechanism (the merge allocates through `GinNewBuffer` and extends only
  when the FSM is empty) and files the unpredictability as an open question.
- **A new failure mode was found in the payload-and-fill model**, which the first run
  had filed as accurate to 3.07% on six fixtures. It predicts within 3.1% on six of
  seven fixtures here but misses `f1` by **+19.94%**, and the mechanism is that GIN's
  entry tree never deletes a tuple: after a churn that replaces every term, the index
  still carries an entry tuple per dead term, which the census counts as payload and a
  rebuild discards. `f7` was added specifically to isolate this — same corpus, three
  rewrite passes that keep the term set — and it is predicted to **+0.07%**.
- Three claims the first run argued only from source are now measured. Dead pages are
  reused: with `f2` at 809 blocks of which 491 were dead, another 50,000 rows' pending
  list grew the file by **26 blocks**, not ~490. A deleted page that is not yet
  recyclable is counted as a *data* page by `ginvacuumcleanup`: `f1` read
  `meta_data_pages 77` against 49 leaf + 14 internal live data pages, the difference
  being its 14 deleted pages. And the recovery refusal was reproduced on a real
  **physical standby** built with `pg_basebackup`: `ERROR: recovery is in progress` with
  its hint, plus `cannot execute VACUUM during recovery`, while `pgstatginindex`,
  `get_raw_page` and `gin_page_opaque_info` all answered — so the page's own
  "settle the state first" step is impossible on a replica, which is now documented.
- Everything else in the traps table reproduced verbatim: `index "f5_deleted_gin" (gin
  index) is not supported`, `is not valid` for an invalid index from both `pgstattuple`
  and `pgstatginindex` (whose raw pages still read `version 2` over 2 blocks),
  `relation "..." is not a GIN index` for a B-tree and for a partitioned index,
  `cannot get raw page from relation "..."` with the partitioned-index detail,
  `pg_freespace` returning **0 rows** on the partitioned index and **answering anyway**
  on another session's temp index where the other two refuse, `block number 100000 is
  out of range`, `invalid page size` on a 100-byte `bytea`, the all-zero page's row of
  NULLs and `lower 0 | upper 0 | pagesize 0`, `gin_clean_pending_list` returning 0 with
  a `DEBUG1` on an invalid index, `must be superuser to use raw page functions` for a
  `pg_stat_scan_tables`-only role that could still run `pgstatginindex`, `pg_freespace`
  and `pgstattuple`, and `must be owner of index f6_slack_gin`. Cancellations came in at
  **2001.272 ms** (`lock_timeout = '2s'`) and **1500.477 ms**
  (`statement_timeout = '1500ms'`) against 2000.846 / 1500.252 filed, with the index
  intact afterwards. `gin_leafpage_items` reports the page's own flags, so the filed
  `Flags 0000` became `Flags 0002` when handed an entry leaf; the page now explains the
  value instead of fixing it.
- Cost and self-checks: `census_total_pages = blocks` on all 8 GIN indexes the statement
  selected, the `OFFSET 0` form read **1234** buffers on a 1235-block index against
  **2468** for the naive form (exactly twice), a cold census of a 1263-block index read
  1262 blocks in 16.311 ms and warm runs took 11.330 / 11.669 / 11.656 ms, and two
  consecutive whole-database runs were byte-identical at 1,244 bytes of CSV.
- Page edits: the review prompt and its corrections in `## Question`, the short answer,
  a new `### The fixtures` section carrying the published SQL, the two scoring tables,
  the lower-bound section, the entry-slack section, the lifecycle table, the horizon
  note, the cost paragraph, the privileges verification, two traps rows, the timeout
  paragraph, a reading rule, `## Context Reviewed`, six `## Evidence Map` rows, and
  `## Open Questions` rewritten from 11 to 13. The `## Contents` list gained one entry
  and was checked mechanically against all 26 headings; all page-internal anchors
  resolve.
- Sandbox: `.wiki-runtime/tmp/ginw2/` is **retained** at 1.7 GB — `data/` 1.3 GB,
  `data_sb/` (the standby) 319 MB, `build/` 74 MB, `install/` 35 MB, plus `work/` and
  `logs/` — with both postmasters **stopped**, no `postmaster.pid` and no postgres
  process left running; restart with
  `install/bin/pg_ctl -D data -l logs/server.log start`. `work/` holds the fixture SQL,
  the census statement as published, the horizon/lifecycle/reindex/probe scripts, the
  session and standby shell scripts, and the citation, anchor and scoring checkers;
  `results/` holds the censuses, probe transcripts and the final scoring table.
  `wiki/versions.md`'s claim that `.wiki-runtime/tmp/ginw/` is retained was corrected,
  since that directory no longer exists.
- **`verified_by_agent` was set** to `claude-opus-5-max 2026-08-26T13:30:26Z`, because
  every claim the page now makes is either re-read in the pinned source during this
  review or measured on the rebuilt server, with the residual gaps declared under
  `## Open Questions`. Two attributions are deliberately historical rather than
  re-verifiable — the first run's one-block flush growth and its 38.88% fresh-control
  slack — and both are named as the first run's, not as engine behaviour. `verified:`
  stays `false`, it being human-only, so the title keeps its `(unverified)` suffix.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings, plain and
  with `--warnings-as-errors`. `wiki/index.md`, `wiki/v17/index.md` and
  `wiki/versions.md` updated; `raw/postgres-17` is clean and still on
  `786db8dcf16`, the build having been run out-of-tree.

## [2026-08-25] answer v17 | GIN wasted-space procedure from contrib extensions

- Filed [Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib Extensions
  on PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-index-wasted-space-contrib.md) against
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11): a designed and
  measured `pageinspect` page census, with `pgstattuple` and `pg_freespacemap` as
  supporting checks, plus `REINDEX INDEX` as ground truth.
- Prompt hygiene: the request's first line read `Follow AGENTS.md. in PostgreSQL 17.`,
  whose stray period left a lowercase fragment, and item 5 wrote "the pg_stat_scan_tables
  grant for pgstatginindex 1.5" where 1.5 is the *pgstattuple extension* version. The
  asker chose "correct and restate", so `## Question` carries the corrected text and
  names both corrections. Two scoping answers were taken up front: ~200k-row fixtures
  rather than ~1M, and permission to delete the 8.3 GB `.wiki-runtime/tmp/idxm/` sandbox
  first.
- **Two premises of the brief are wrong and the page says so.** `pg_freespace()` cannot
  return 8191: `RecordFreeIndexPage` does write `BLCKSZ - 1`, but the FSM stores one byte
  per page, `fsm_space_avail_to_cat` collapses everything at or above `MaxFSMRequestSize`
  into category 255, and `fsm_space_cat_to_avail` converts 255 back to exactly
  `MaxFSMRequestSize` = `MaxHeapTupleSize` = **8160** on this build. Measured: 768 and 981
  free pages all read `avail = 8160`, and the maximum `avail` over every GIN index in the
  sandbox was 8160. And there is no `pages_free` field in `VACUUM VERBOSE` output — v17
  prints `pages: %u in total, %u newly deleted, %u currently deleted, %u reusable`, where
  only the fourth number is GIN's whole-index census; `pages_deleted` is incremented
  per-run by `ginDeletePage` and `shiftList`, which the server confirmed by reporting
  `0 currently deleted, 768 reusable` for the GIN index beside `520 currently deleted` for
  the B-tree on the same table.
- **Design.** Three numbers, never summed: whole-page waste (pages whose flags contain
  `deleted`, plus pages where `gin_page_opaque_info` returns a NULL row, i.e. all-zero),
  live-page slack (`page_header.upper - lower`, which is exactly GIN's own
  `GinDataLeafPageGetFreeSpace` = `PageGetExactFreeSpace`) split into entry-tree and
  posting-tree bytes, and pending-list bytes. `deleted` is tested before `data`/`list`
  because `ginDeletePage` only ORs the bit in, leaving `{data,leaf,deleted,compressed}`;
  `shiftList` by contrast *replaces* the flags word, so no stale `list` page can survive a
  flush. The published statement reads each page exactly once through a
  `(SELECT get_raw_page(...) OFFSET 0)` lateral — measured at **897** buffer hits on an
  898-block index against **1794** for the naive form that calls `get_raw_page` inside
  both functions — and carries its own self-check, `census_total_pages = blocks`, which
  held on all 8 GIN indexes.
- **Result against `REINDEX INDEX`, six ~200k-row fixtures.** `waste + slack` bounded the
  reclaimed bytes from above in **6 of 6**; `waste` alone did **not** bound them from
  below. The counterexample is a flushed pending list: **65.49% dead pages, 50.07%
  reclaimed**, because a fresh GIN build is less dense than an aged one —
  `entrySplitPage` equalizes data size, so key-ordered builds leave pages ~60% full
  (measured fills after rebuild: 60.84 / 58.80 / 61.12 / 50.20 / 43.28 / 87.96%), and that
  index's **517-block in-use core rebuilt to 748 blocks, 44.7% larger**.
- **Entry-page slack is growth room, not waste**, proven three ways: the untouched control
  reports 38.88% slack and `REINDEX` reclaims **0 bytes**; merging 50,000 rows of pending
  entries dropped entry slack 1,493,880 -> 644,348 while the entry tree grew 515 -> 516
  pages; and repeating that on a freshly rebuilt index (41.20% slack) absorbed another
  50,000 rows with **zero** page growth. Posting-tree slack does track the truth: 51.34%
  measured against 46.33% reclaimed.
- **Pending-list lifecycle, byte-exact:** 4,227,072/516 blocks -> 12,263,424/1497 with 981
  pending pages -> 12,271,616/1498 after `gin_clean_pending_list()` (which **grew the file
  by one block** and turned 65.49% of it into dead pages, 981 of them, confirmed by the
  FSM) -> 6,127,616/748 after `REINDEX`. `pgstatginindex` reads `0 / 0` in three of the
  four states. A `VACUUM` instead of the SQL flush reports those pages reusable **in the
  same run**, because `shiftList` leaves `pd_prune_xid` at 0.
- **The horizon result.** Posting-tree deletions behave the opposite way:
  `GinPageIsRecyclable` passes `NULL` to `GlobalVisCheckRemovableXid`, which selects
  `VISHORIZON_SHARED`, the most conservative horizon. On the idle cluster, VACUUM 1
  reported `768 newly deleted, 768 currently deleted, 0 reusable`, VACUUM 2 reported all
  zeroes, and only after three `pg_current_xact_id()` calls did VACUUM 3 report **768
  reusable** with 768 FSM entries at 8160. The census saw all 768 pages the whole time
  (`prune_xid = 768`), which is the practical argument for reading flags over reading the
  FSM. The file never shrank at any step, and there is no `RelationTruncate`/`smgrtruncate`
  call anywhere under the GIN access-method directory.
- Caveats measured, not asserted: superuser-only raw-page functions (a
  `pg_stat_scan_tables`-only role ran `pgstatginindex`, `pg_freespace` and table
  `pgstattuple`, but got `must be superuser to use raw page functions` and `must be owner
  of index`); `pgstattuple` refusing an *invalid* GIN index as `is not valid` before it
  ever reaches the AM switch, while `get_raw_page` + `gin_metapage_info` read it happily;
  partitioned GIN indexes refused by two functions and answered with 0 rows by
  `pg_freespace`; another session's temp GIN index refused by two functions and **answered
  anyway** by `pg_freespace`, which has no such guard; `gin_leafpage_items` needing flags
  to equal `{data,leaf,compressed}` exactly (`Flags 0000, expected 0083`); the page-at-a-
  time, non-atomic scan; entry-page slack over-stating usable space by one `ItemIdData`
  because `entryIsEnoughSpace` uses `PageGetFreeSpace`; no GIN verifier in v17 contrib
  amcheck; and the census reading through the default buffer strategy where
  `pgstattuple`/`pgstatindex` use `BAS_BULKREAD` (cold run: 1728 reads for a 1729-block
  index, 13.781 ms; warm: 1728 hits, 9.039 ms). `lock_timeout = '2s'` cancelled at
  2000.846 ms behind an uncommitted `DROP INDEX`; `statement_timeout = '1500ms'` at
  1500.252 ms. Two consecutive runs of the published statement were byte-identical.
- Eleven open questions, including no fixture producing an all-zero page, `ginVersion <> 2`
  being untestable on a v17-only cluster, autovacuum being off throughout (autoanalyze
  alone flushes a pending list, so the pending reading is inherently racy), only two index
  shapes, and the payload-and-fill model that predicted every rebuild size within 3.07%
  taking its fill fraction from the rebuild it predicts.
- Environment: `.wiki-runtime/tmp/idxm/` (8.3 GB) was deleted first at the user's explicit
  instruction, which destroys the recorded runs behind the non-B-tree inflation page; that
  page's two "sandbox/artifacts under" pointers in `## Context Reviewed` were corrected to
  say so. The new sandbox `.wiki-runtime/tmp/ginw/` is retained at 2.7 GB (cluster shut
  down cleanly, 11 SQL scripts, `work/analyze.py`, `work/checkpage.py`, 11 result files
  including `results/SUMMARY.md`), and the exact-pin 17.11 install under
  `.wiki-runtime/tmp/pstate/install/` was reused rather than rebuilt.
- Checked mechanically before filing: all 189 raw citations resolve to existing files with
  in-range line numbers, the `## Contents` list matches all 24 headings in order, and every
  page-internal anchor resolves. `wiki/index.md`, `wiki/v17/index.md` and
  `wiki/versions.md` updated; `raw/` is unchanged; the page keeps `verified: false` and
  `verified_by_agent: not yet`.

## [2026-08-25] review v17 | re-ran every test on the non-B-tree inflation heuristic

- Second review of [Detecting Inflated Non-B-Tree Indexes From Catalogs and a
  COMMENT-Stored Baseline in PostgreSQL 17
  (unverified)](v17/questions/indexing/non-btree-index-inflation-comment-baseline.md) on
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11), answering the gap the
  first review left: **the tests were actually re-run**, which closes its open question
  12.
- Prompt hygiene: the request wrote `agents.md` for AGENTS.md, lowercase `postgresql`, a
  space before a comma, and "run again all tests" for "run all tests again". The asker
  chose "correct and restate", so `### Review prompts, 2026-08-25` now carries both
  corrected prompts and names the corrections. Four scoping answers were taken up front:
  all three test groups, **three** matrix runs, and scoring by the statement the page
  publishes rather than the harness's copy.
- Scope executed: the 13-cell matrix three times (**runs 5, 6, 7**; 10m24s / 10m11s /
  10m09s wall, 622 / 611 / 608 s of measured phases), the six original probes P1-P6, the
  13 review probes R1-R12, and one new probe R13. The evaluate and capture blocks were
  extracted from the page itself, so what ran is what a reader would copy; the harness
  copy differs from the filed one only in output paths and that switch, verified by diff.
  Nothing under `results*/`, `sql/`, `cells/` or `logs/` was touched; all new artifacts
  are in `.wiki-runtime/tmp/idxm/review2/`.
- **The heuristic holds and the physical measurements are exactly reproducible.** All 13
  `(B, post-churn, C, R)` quadruples in runs 5-7 are byte-identical to filed run 4, hence
  to runs 2 and 3: **six runs, two days, one server restart**. So are all 13
  `reclaimed_bytes` and `reclaimed_pct`. **All 13 recommendations match in all three
  runs**, which is simultaneously the direct proof that yesterday's access-method guard
  is inert on matrix data, since runs 5-7 used the guarded text and run 4 the unguarded
  one.
- The `ANALYZE` gate held in all 39 new pre-`VACUUM` evaluations (78 over six runs), and
  `c05_gin_pending`'s pre-`VACUUM` inflation was 1.372 every time.
- **Nine of 13 `size_inflation` figures are bit-identical across all six runs.** The four
  that drift — `c04` 6.877-6.998, `c06` 4.801-4.907, `c08` 7.228-7.419, `c12`
  0.178-0.181 — are exactly the cells whose population term is an `ANALYZE` sample. Worst
  relative drift 2.6%, and no cell sits within 2% of its threshold. A full 13-cell,
  six-run table replaces the old 4-cell, three-run one.
- **The two figures corrected yesterday are sample-dependent, so they are now ranges.**
  `pf_shift` on `c12` read 9.152 / 9.024 / 9.124 / 8.976 / 9.148 / 9.074 against a **true
  9.00x** (200,000 of 2,000,000 rows to 1,800,000), and `c10`'s churn ratio 5.311 /
  5.468 / 5.418 / 5.396 / 5.479 / 5.364. Yesterday's fix was right about internal
  consistency and wrong about precision; quoting three decimals from one run is the
  defect, not which run it came from.
- **Every probe reproduced, and one reproduced its own bug.** P5 re-run unmodified again
  failed with `VACUUM cannot run inside a transaction block` and again printed its
  post-`ANALYZE` column twice — direct proof the filed third column never came from it —
  while the corrected form returned **43 / 200000 / 22 for BRIN and 180000 for the other
  four**, the filed table exactly, 43 included. P4's GIN pending list was byte-identical
  (491 pages / 16,654,336 -> 0 / 21,905,408). P3's reset block reproduced character for
  character with `d_ins` at `-200000`. P2's filenode/OID progression reproduced in shape
  (17275/17275 -> 17275/17276 -> 17277/17277) with identical eval readings. P1 came to
  **340/243** where the filed capture is 309/212, and both deltas are exactly 31 bytes —
  the length of `"dbr": "...", ` — which closes that figure arithmetically. P6 wrote
  **346** where the page says 348.
- **Two corrections.** The invalid-index arm was demonstrated yesterday on a *B-tree*
  fixture, which the guard now pre-empts with `unsupported access method`; re-tested on a
  target AM — an invalid **hash** index left by a `CREATE INDEX CONCURRENTLY` whose
  expression divided by zero — it fires as designed and reports `skip: index not valid`
  (new probe R13). And a parallel BRIN build's `reltuples` is **not reproducible**: eight
  observations of the default build span 43 to 46, the four-worker build gave 102 then
  90, while the serial build wrote 23 on both days.
- Re-confirmed from the review probes: the BRIN desummarize/summarize sequence byte for
  byte (114688 -> 114688 -> 196608, `REINDEX` back to 114688); two captures in the same
  second identical and one two seconds later differing only in `ts`; and the bloated
  B-tree at inflation 2.978 now reading `unsupported access method` where it read `none`.
- Page edits: `### Review prompt` became `### Review prompts` with both prompts, a new
  `### The three-run re-execution` section with a 13-cell six-run inflation table and a
  six-row probe table, 2 Contents entries, a Verdict paragraph, the Results
  reproducibility paragraph rewritten from three runs to six, the old spread table
  removed, three single-run figures turned into ranges, the BRIN build table gained a
  re-run column, the `dbr` arithmetic, the pre-`VACUUM` count, the invalid-index
  cross-reference, the GIN pending-list limitation, the hot-key aside softened to what it
  is, 1 Context Reviewed bullet, and Open Questions rewritten from 14 to 15 (dropping the
  now-answered "matrix not re-run", adding one-cluster scope, guard precedence, parallel
  non-determinism and the one unreproduced aside). All 29 Contents links resolve against
  the 30 `##`/`###` headings, in order, and so do both in-body anchors.
- Server: the same cluster restarted, `block_size` 8192, `autovacuum = off`,
  `fsync = off`, `shared_buffers` 512MB, `maintenance_work_mem` 256MB; shut down cleanly
  afterwards, sandbox 8.3 GB, 53 GB free. `raw/` untouched and clean on the manifest pin.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated.
  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. The page keeps
  `verified: false` and `verified_by_agent: not yet`: one measured aside — a single hot
  key at 1 row per 90 seconds — was not reproduced, and it is filed as open question 15
  rather than quietly dropped.

## [2026-08-25] review v17 | the COMMENT-stored non-B-tree inflation heuristic

- Reviewed [Detecting Inflated Non-B-Tree Indexes From Catalogs and a COMMENT-Stored
  Baseline in PostgreSQL 17
  (unverified)](v17/questions/indexing/non-btree-index-inflation-comment-baseline.md)
  against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11).
- Prompt hygiene: the request wrote `agents.md` for AGENTS.md, lowercase `postgresql`,
  and a space before a comma. The asker chose "correct and restate", so the page carries
  a `### Review prompt, 2026-08-25` block naming the corrections. Two scoping answers
  are recorded there: re-read every citation and exercise the SQL on a server but do
  **not** re-run the 13-cell matrix, and fix defects in place rather than only report
  them.
- **The measurement sandbox survived**, which changed what the review could do. The
  2026-08-24 cleanup removed `.wiki-runtime/tmp/idxmaint/` (the earlier, misfiring
  harness for a deleted page); this page's sandbox is `.wiki-runtime/tmp/idxm/`, built
  later the same day, and it still holds 7.6 GB: the cluster, all 13 fixture tables,
  `cells/`, `sql/`, `logs/probes.log` and four result sets. The 17.11 install under
  `pstate/install/` matches the configure line the page states. So the review restarted
  that cluster instead of rebuilding, and audited the filed numbers against the
  harness's own CSVs.
- **Citations: 101 links over 57 distinct ranges before this review, all re-read; none
  missing, none past EOF,
  none misattributed.** Three were imprecise and were corrected or extended:
  `index_update_stats` is not "skipped entirely" under `IsBinaryUpgrade` (only the
  `relpages`/`reltuples` write is), `pg_relation_size`'s VOLATILE marking now cites
  `pg_proc.dat`, and the `values_per_range` default of 32 now cites the macro rather
  than only the reloption call that passes it.
- **Three figures were run-3 values sitting in a run-4 table**: `pf_shift` 9.024, the
  derived 9.02x, `size_inflation` 0.180 and `c10_brin_minmax`'s churn ratio 5.468 now
  read 9.124, 9.12x, 0.178 and 5.418. No verdict moves; this is the page's own
  documented 1-2% `ANALYZE`-sample drift surfacing as an internal inconsistency.
- **Two "measured" claims had no recorded evidence. Both re-measure correctly.** The
  `reltuples` progression's probe ran `DELETE ...; VACUUM ...` in a single `psql -c`,
  which aborts with `VACUUM cannot run inside a transaction block` and rolls the
  `DELETE` back — the recorded third column is just the second one repeated. Re-run with
  the statements separated: **180000 for hash, GIN, GiST and SP-GiST, and 22 for BRIN**,
  exactly as filed. The statistics-reset output block was produced by an older statement
  (`churn_known` column, fabricated `-200000.000`); re-run with the filed text it
  **reproduces character for character**.
- **The BRIN parallel over-count mechanism was wrong and is now source-correct.** The
  merge does not "increment it again per merged range": each participant counts its own
  *partial* summaries in `form_and_spill_tuple`, adds them to `brinshared->indtuples`,
  the leader adopts that sum in `_brin_parallel_heapscan`, and `_brin_parallel_merge`
  unions the duplicates with `brin_doinsert` **without recounting**
  (`brin_fill_empty_ranges` does not count either). So the stored number is
  (participant, range) pairs. Overlap is the norm because a parallel scan's chunk is
  `nblocks / 2048` rounded up to a power of two, smaller than a 128-page BRIN range for
  any table under ~262,000 pages. Measured on one 23-range index: **23 serial, 44 then
  45 on repeats at the default, 102 with four workers**, against 23 real summary tuples
  by `brin_page_items` — which also explains the filed 348 on a 70-range index as five
  participants times seventy. The filed `43` is flagged as the artifact it is, and open
  question 14 records that only the serial number is reproducible.
- **`brin_summarize_range()` had never been run.** The filed table measured
  `brin_summarize_new_values()` while the prose and the prompt name
  `brin_desummarize_range()` + `brin_summarize_range()`. Re-run with 318 explicit
  desummarize calls then 318 summarize calls: **114688 -> 114688 -> 196608**, and
  `REINDEX` back to 114688 — identical, so the page's headline BRIN result now rests on
  the function pair the prompt asked about. `fresh` 49152/24576/11977 pages and
  `after churn + VACUUM` 114688/32768/40701 reproduced exactly.
- **Two captures are byte-identical only inside one clock second.** Measured md5s: two
  captures in the same second match, one two seconds later differs, in `ts` and nothing
  else. Also, the filed 309-byte/212-byte comment is the `dbr`-absent form — the field is
  only dropped because that database had never had a statistics reset — and the identical
  human comment re-captured after a reset is **340 bytes with a 243-byte payload**.
- **One statement change, regression-tested.** A B-tree index at `size_inflation` 2.978
  and `churn_ratio` 2.000 scored **`none`**, because the threshold table has no `btree`
  row, so the `LEFT JOIN` left every comparison NULL and the `CASE` fell to its `ELSE` —
  a refusal that reads as a verdict. Added
  `WHEN t.amname IS NULL THEN 'unsupported access method'` as the first arm. Over all 19
  indexes carrying a baseline on that cluster the filed and amended texts differ on
  **exactly 2 rows, both `btree`**; all 13 cells and the four non-B-tree review fixtures
  are byte-identical. The page's published block, extracted and run verbatim, matches the
  tested text on all 19 rows, and the published capture block is line-identical to
  `sql/capture.sql`.
- Two behaviours were measured for the first time and filed as evidence: a failed
  `CREATE UNIQUE INDEX CONCURRENTLY` leaves `indisvalid = false` and the statement
  correctly reports `skip: index not valid`; and a baseline-less index prints
  `churn_state = unknown: counters reset`, a mislabel (limitation 12) whose
  `recommendation` is still the correct `capture new baseline`.
- **What held.** Runs 2, 3 and 4 carry identical `(B, post-churn, C, R)` quadruples on
  all 13 cells and run 1 differs on exactly the two cells the page calls defective; all
  13 published cell scripts match the scripts that ran; the 13 `size_inflation` values
  are run 4's recorded output; every kB in the results table is the recorded byte count
  over 1024; the `1 - 1/inflation` column, the 7-of-13 and 12-of-13 counts, the seven
  `strong` flags and their 37.7-85.7% range, the pre-`VACUUM` `inconclusive` sweep with
  `c05`'s 1.372, the P1/P2/P4/P6 probe outputs, and the reset timestamp
  `2026-08-24 16:19:17.78836-04` all check out.
- Page edits: a `### Review prompt, 2026-08-25` block, a new
  `### What the 2026-08-25 review changed` section, 2 Contents entries, the corrected
  `reltuples` table plus a three-build comparison table, the rewritten parallel-build
  paragraph, the corrected BRIN maintenance table, the qualified comment-format and
  human-comment claims, the expanded reset paragraph, one new SQL arm with two comment
  blocks, 2 new limitations (11-12), 3 new open questions (12-14), 2 Context Reviewed
  bullets, 7 Evidence Map rows, and 3 Source References. All 28 Contents links resolve
  against the 29 `##`/`###` headings, in document order, and so does the one new
  in-body anchor.
- Measured on the page's own restarted cluster (17.11, `block_size` 8192,
  `autovacuum = off`, `fsync = off`, `shared_buffers` 512MB,
  `maintenance_work_mem` 256MB), then shut down cleanly again; the data directory is
  retained and the review's scripts, logs and CSVs are under
  `.wiki-runtime/tmp/idxm/review/`. Review fixtures `r1`, `r3`, `r5`, `r11` and `r12`
  were added to that database; the `c00`-`c12` fixtures, `results/` and
  `results_run{1,2,3}/` were not touched. `raw/` is untouched and clean on the manifest
  pin.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated. The page keeps
  `verified: false` and `verified_by_agent: not yet`: the review re-derived and re-ran a
  great deal, but it did not execute the 13-cell matrix again, so the results table is
  still run 4's output rather than an independently reproduced run (open question 12).

## [2026-08-24] follow-up v17 | mandatory test 113 and change E in the core-SQL bloat statement

- Added a 92nd mandatory test and one correction to [Testing the PostgreSQL 12
  Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), on
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). Test 113 is
  the asker's shape: 1,000,000 rows inserted as `state = 'pending'`, a partial
  index `WHERE state = 'pending'`, then every row updated to `'done'`. A
  `REINDEX` takes that index from **2745 blocks to 1** — the largest true reading
  on the page — and the recommended statement **failed all three states**.
- Prompt hygiene: the request had "a additional", "after change the state to
  done", spaces before commas and a run-on final clause; the asker approved a
  corrected restatement, which `## Question` carries with the original quoted in
  its prompt note. Four scoping answers are recorded there: both post-`UPDATE`
  states are measured, a failing statement is corrected rather than only
  reported, the test is numbered **113** so nothing filed is renumbered, and
  `state = 'pending'` is read as the predicate with a distinct `bigint` key.
- **The failure was never arithmetic.** `expected_blocks` was already 1 in all
  three states. The guard `reltuples = 0 AND actual_bytes > bs AND n_live_tup > 0`
  turned that into `unmeasured: reltuples 0, table has live rows`, and change B
  withheld the unmaintained state. The guard is right for 12 and 13 and wrong
  from 14: commit `3d351d916b2` in this checkout says in its own message that the
  point of the `-1` sentinel is that "it's impossible to distinguish 'never yet
  vacuumed' from 'vacuumed and seen to be empty'", earliest containing tag
  `REL_14_0`.
- **Change E** is two lines in that branch: the guard now also requires
  `server_version_num < 140000 OR n_mod_since_analyze > 0`. Both maintained
  states then read **100.0% / 100.0% and 21 MB against a measured 100.0%**. It is
  the first lettered change on the page that adds rows rather than withholding
  them, it adds no CTE, no join and no construct 12 lacks, and the header map,
  the intro paragraphs, the recommended-statement section, the reading rule and
  the residual-error table were all updated for it.
- **The threshold is deliberately not change B's.** A third text substituting the
  auto-analyze trigger was built and measured: fixture 118 — a subset measured
  empty, then 50,000 rows arriving under a 100,050 trigger — reports **99.3% on a
  139-block index a `REINDEX` reproduces exactly**, `status = ok`, no caveat. The
  `> 0` form withholds it and one `ANALYZE` gives the correct 2.9%.
- **Change E's own cost is filed, not hidden.** Fixture 120 is a tail-clustered
  2,000-row subset read at `default_statistics_target = 1`: the 300-row sample
  wrote `reltuples = 0` in **8 of 10** consecutive `ANALYZE`s, and change E then
  reads **87.5% on a live 8-block index**. It is recorded as a critical false
  positive in the residual table, identifiable by `modelled_rows = 0`, with the
  size bound that keeps it under the 1 MB triage filter marked as derived rather
  than measured.
- **Regression evidence is an `EXCEPT`, both directions, over 119 indexes**: the
  page's own runnable deduplication-gate harness verbatim, a shape rebuild of
  partial requirements 18-91, the 92-112 controls, and the eight change-E
  fixtures. **Exactly 4 rows move**, all in the branch: `p113b` and `p113c`
  `unmeasured` -> 100.0 (PASS), `p120` `unmeasured` -> 87.5 (the false positive),
  and `p118` unmoved. Verdicts go 50 PASS / 2 critical false positives / 4
  unmeasured to 52 / 3 / 1; the trigger form would be 52 / 4 / 0. Seven indexes
  reach the branch at all, and the page now carries an audit query that lists
  them on any database.
- Fidelity check on the rebuilt fixtures: `x109` reproduces the filed **64.9% on
  5201 blocks** exactly, and `p74`/`p75`/`p77`/`f91` land within 0.4 points of
  their filed values. The page states plainly that the 18-112 fixtures are shape
  rebuilds — the original scripts went with the sandbox deleted earlier today —
  so this run does not claim to re-verify the filed per-test table.
- Cost is unchanged: 58.0-68.4 ms filed against 57.4-65.1 ms amended over six
  interleaved pairs of the exact texts on the 119-index database; on the
  post-`REINDEX` state both exact texts produce byte-identical `psql` output.
- Measured on **two newly initialised clusters**, both built for this run because
  `.wiki-runtime/tmp/` was emptied earlier today: 17.11 from this page's pin
  configured `--without-readline --without-zlib --with-icu --enable-debug`, and
  12.2 from `45b88269a353ad93744772791feb6d01bc7e1e42` configured
  `--without-readline --without-zlib --enable-debug`; both `block_size` 8192,
  `autovacuum = off`, `fsync = off`. On 12.2 the same fixture reads `unmeasured`
  under **both** texts with `n_mod_since_analyze` at 0, so the version gate alone
  holds it — the change is a measured no-op there — while a `REINDEX` reclaims the
  same 2745 blocks to 1. `EXCEPT` returns 0 rows either way on that server. Both
  servers were shut down cleanly and both data directories are retained under
  `.wiki-runtime/tmp/pstate/`.
- Page edits: the corrected prompt and its note in `## Question`, one header-map
  line and four logic lines in the SQL block, a new paragraph in the statement's
  intro, a `### Verdict` paragraph, a new `### The current recommended statement`
  paragraph plus a `modelled_rows = 0` reading rule and two residual-error rows,
  a third row in the mandatory-suite group table, nine new `###` sections, 9
  Contents entries, 2 Context Reviewed bullets, 5 Evidence Map rows, 5 Open
  Questions and 12 Source References.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is
  unchanged and both checkouts are clean on their manifest pins; the page keeps
  `verified: false` and `verified_by_agent: not yet`.

## [2026-08-12] answer v12 | extract declarative range partition bounds with SQL and no regex

- Filed [Extracting Declarative Range Partition Bounds With SQL and No Regex in
  PostgreSQL 12
  (unverified)](v12/questions/server-administration/extract-range-partition-bounds-without-regex.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2). The
  prompt contained "postgreql 12", a space before its comma, lowercase "sql", and
  "declarative range partition" for "declarative range partitioning"; the user
  chose the corrected wording, which is what `## Question` now restates.
- Filed under `server-administration` because the deliverable is a catalog
  introspection runbook and every citation is catalog, deparse, client-tool, or
  documentation code rather than planner code.
- Verdict: PostgreSQL 12 exposes no typed partition bound. `pg_class.relpartbound`
  is a `pg_node_tree` whose text form dumps each datum as bytes via `outDatum`, so
  the only usable SQL form is the `pg_get_expr(relpartbound, oid)` deparse that
  `psql \d` prints and `pg_dump` pastes into `ATTACH PARTITION`. No regex is needed
  because `get_rule_expr`'s `T_PartitionBoundSpec` arm plus
  `get_range_partbound_string` and `get_const_expr(..., -1)` emit a fixed grammar:
  `FOR VALUES FROM (` … `) TO (`, elements joined by `, `, bare `MINVALUE`/
  `MAXVALUE`, `DEFAULT` for a default partition, and literals with no `::type` and
  no `COLLATE`.
- The filed recipe splits the text on the quote character with the two-argument
  `string_to_array`, masks each literal's contents to an equal-length `x` run so
  quote parity is decidable, then locates `) TO (` and `, ` with `strpos`/`substr`.
  It uses `substr`, never `substring`, because only `substring(text, text)`
  resolves to the regex `textregexsubstr`. Output is one row per partition, side,
  and key position, with the key column and type from `pg_attribute` through the
  zero-based `partattrs` vector.
- Tested on an isolated exact-pin 12.2 server built under `.wiki-runtime/pg12`.
  Correctness matrix: 18 range parents, 45 partitions, 95 creation-time control
  elements over `int4`/`int8`/`numeric`/`date`/`timestamptz`/`text`/`text COLLATE
  "C"`/two-column/expression/`boolean`/`float8`/`bytea`/`interval`/`char(3)`/
  `uuid` keys plus adversarial `text` bounds containing `) TO (`, `, `, escaped
  quotes, a newline, and the literal word `MINVALUE`.
- Results: a from-scratch reconstruction of the deparse rules matched
  `pg_get_expr` on 41/41 partitions; the recipe scored 95/95; the obvious
  `split_part` + `btrim` version scored 81/94 elements and got 9/41 partitions
  wrong; `pg_get_expr(relpartbound, 0)` and the `pretty = true` form were
  identical to the canonical call on 45/45; 0 bound texts contained `::` and 2
  contained a newline inside a literal; the value round trip through the key type
  was exact in 78/81, the three exceptions being `boolean`'s `true`/`false`
  keywords.
- Also measured: `DateStyle`, `TimeZone`, `IntervalStyle`, `extra_float_digits`,
  `bytea_output` and `standard_conforming_strings` all change the text, all
  `PGC_USERSET`; `standard_conforming_strings = off` dropped the score to 91/95
  until an `E''`-string `replace` restored 95/95 under both settings, while the
  plain `replace(x, '\\', '\')` spelling does not parse at all under that setting;
  14 catalog `AccessShareLock`s and zero partition locks; 39.9-44.9 ms masked
  versus 8.3-9.2 ms plain-split at 1048 partitions and 2109 rows; an unprivileged
  role read all 2109 rows including a partition whose schema it cannot use;
  `DETACH` erases `relpartbound`; a `REPEATABLE READ` snapshot plus a concurrent
  `DROP` made `pg_get_expr(…, oid)` NULL while `pg_get_expr(…, 0)` still
  deparsed; a 9600-character bound failed with `row is too big: size 9304,
  maximum size 8160` because `pg_class` has no TOAST table; and
  `cannot use subquery in partition bound` bounds the expression grammar.
- Recorded seven open questions, including the complete absence of any upstream
  test that deparses a quoted RANGE bound containing a comma, quote, or
  backslash, the untested `relkind = 'f'` case, expression-key type derivation,
  and the unobserved `READ COMMITTED` variant of the NULL-deparse race.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Kept
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet`. The temporary database and fixtures were dropped
  and the test server stopped; only the SQL scripts under
  `.wiki-runtime/tmp/partbounds/` remain.

## [2026-08-12] answer v12 | calibrate a COMMENT-stored bytes-per-table-tuple REINDEX threshold for every non-btree index

- Filed [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for
  Every Non-B-Tree Index in PostgreSQL 12
  (unverified)](v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2). The
  prompt contained "postgreql 12", "bytes per tuples", a comma splice before
  "For each index type", "documment", and "how it was tested"; the user chose
  "Fix all typos/grammar", picked the five core non-btree access methods
  (hash, GiST, GIN, SP-GiST, BRIN; no contrib `bloom`), and approved running real
  workloads rather than a source-only answer. The corrected prompt is restated
  verbatim under `## Question` with a note recording the correction.
- Verdict: the scheme calibrates for GiST, is row-count-dependent for hash and
  SP-GiST, is screening-only for GIN, and must never fire for BRIN. Per-access-
  method thresholds, perfect windows and confusion counts (positive class =
  `REINDEX` reclaims >= 20% of current bytes): GiST 25% (window 13.97-33.33,
  14/0/0/8), SP-GiST 30% (28.24-32.17, 15/0/0/7), hash 10% (8.78-11.11,
  16/0/0/6), GIN `fastupdate = off` 50% (no perfect window, 16/2/0/4), GIN
  `fastupdate = on` 75% (55.84-100.00, 18/0/0/4), BRIN never (0/4/0/18 at its
  least-bad setting). One all-method threshold of 30% scores 90.2% over all 132
  cells and 93.6% over the 110 non-BRIN cells.
- Method: built the exact pin out of tree under `.wiki-runtime/pg12`, installed
  its own `pageinspect`, `pgstattuple` and `pg_freespacemap`, and ran an isolated
  12.2 cluster on port 55442 with `autovacuum = off` so each cell's
  `VACUUM`/`ANALYZE` boundary is explicit. One deterministic id-derived fixture
  (200,000 rows, 5,406 heap pages) carried six indexes covering all five access
  methods plus both GIN `fastupdate` settings on separate columns. 22 workloads:
  a do-nothing control, six delete fractions, a delete-without-vacuum, five
  churn intensities, a partial-HOT update pass (measured 241,920 of 400,000
  updates HOT), three insert-growth multiples, a row-turnover cell, a mixed
  insert/update/delete cell, and a combined grow-2x-plus-churn cell. Each cell
  stamps the baseline through `COMMENT ON INDEX`, reaches its maintenance point,
  reads the ratio and access-method internals, measures the representative query
  with `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` (median of 7, warm), runs
  `REINDEX INDEX`, and measures both again. Page density came from
  `page_header(get_raw_page(...))` because `pgstattuple()` refuses GIN, SP-GiST
  and BRIN indexes on this version.
- Measured findings recorded on the page: `VACUUM` never shrank any index (byte-
  identical sizes in every delete cell, matching the source fact that the only
  index-side `RelationTruncate` call is `TRUNCATE`'s rebuild path and SP-GiST's
  is `#ifdef NOT_USED`); every access method reads exactly `1/(1-f)-1` after a
  delete of fraction f, so deletes carry no index information; a GIN pending list
  reads +261.73% and `gin_clean_pending_list()` moves it to a worse +388.78%
  (23,232,512 -> 31,391,744 bytes) while delivering the entire query improvement
  (3,779 -> 1,336 blocks, 11.259 -> 1.393 ms) that the subsequent rebuild did
  not, and `fastupdate` alone changes bytes-per-tuple 2.8x; three hash builds
  over identical 200,000 rows give 34.5293 / 38.0928 / 37.1917 bytes per row
  because `hashbuild` sizes buckets from `estimate_rel_size`; fresh hash builds
  from 60,000 to 250,000 rows span 28.071 to 52.634 bytes per row; a grown hash
  index carried 87 unwritten zero pages, with `pg_relation_size` reporting
  5,341,184 bytes against 4,628,480 allocated per `stat`, and `pgstattuple()`
  erroring on it; a 10% delete "reclaims" 22.66% by re-bucketing 768 -> 640;
  BRIN's `pages_per_range` spreads the baseline 6.3x and BRIN churn cost
  130 -> 487 query blocks at a +0.00% reading with 0.00% reclaimed; stale
  `reltuples` flips one GiST index between +84.71% and -7.64%; a partial index
  that grew 80.4% read -5.02% against 44.59% reclaimable; rebuild costs were
  GiST 650.8 ms, SP-GiST 292.9, GIN 156.7/155.7, hash 120.3, BRIN 19.4; and at a
  30% threshold the rebuild reduced query blocks in 12/13 GiST and 13/14 SP-GiST
  cells but 0/14 hash and 0/6 BRIN cells.
- `COMMENT ON INDEX` mechanics were executed on the pin: the grammar takes only
  a literal or `NULL` (both `IS (SELECT ...)` and `IS 12.5` are syntax errors, so
  a computed baseline needs dynamic SQL), one `ShareUpdateExclusiveLock` on the
  index and none on the table, owner-only writes but world-readable values,
  `IS NULL` and `IS ''` both deleting the `pg_description` row, plain `REINDEX`
  keeping the OID (18061 -> 18061) and `REINDEX INDEX CONCURRENTLY` moving the
  single row to a new OID (18061 -> 18063), and `LIKE ... INCLUDING ALL` copying
  the numeric baseline onto an 81,920-byte empty clone.
- Both fenced SQL blocks (the stamping function and the maintenance check) were
  extracted from the filed page and executed on the pin with zero errors and
  zero warnings; the check correctly returned `ignore` for BRIN, partial indexes,
  zero `reltuples`, and baselines whose recorded `relid` no longer matches.
- Reproducibility: two cells re-run end to end matched to the digit for 10 of 12
  index-level results, including byte-identical sizes; only GiST differed
  (84.62% vs 84.22% reading), and the two duplicated matrix labels agreed exactly
  for five of six index shapes. Eleven open questions were filed, covering the
  single-platform constants, the fixed 1,000-key GIN fixture, the 20%-reclaimable
  judgement call, warm single-client query measurement, autovacuum being off, the
  unmeasured SP-GiST held-snapshot case, and GiST's non-determinism.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and the v12 coverage cell plus a
  new coverage note in `wiki/versions.md`. Kept `verified: false`, the visible
  `(unverified)` title, and `verified_by_agent: not yet`: every cited line range
  was re-read in the pinned checkout and every number was produced on the pin,
  but the constants rest on one fixture family and one platform, so this is not
  a full claim-by-claim audit. The test server was stopped and its data
  directory removed; the harness and captured output remain under
  `.wiki-runtime/tmp/bpt/`, and `raw/postgres-12/` was not modified.

## [2026-08-12] remove v12 | delete Calibrating a COMMENT-Stored Bytes-per-Index-Row REINDEX Threshold

- Removed `wiki/v12/questions/indexing/comment-stored-bytes-per-index-row-bloat.md`
  per user request, which named the page by its exact visible title, so no
  prompt text is restated on any wiki page and no prompt-hygiene correction was
  needed.
- Removed the active page entry from `wiki/index.md` and `wiki/v12/index.md`. No
  surviving page linked, cited, or discussed the page, so no `## Navigation`
  list and no page prose changed; its own outbound links to
  `comment-stored-index-heap-ratio-bloat.md`,
  `comment-stored-table-dml-counters-gin-reindex.md`,
  `physical-index-statistics-tuple-counts-and-bytes.md`,
  `btree-index-bloat-core-sql-only.md`,
  `how-pgstatindex-calculates-information.md`, and
  `reindex-index-concurrently.md` went away with it, and all six keep inbound
  links from `wiki/index.md` and `wiki/v12/index.md`.
- Dropped the closing `indexing` clause of the v12 coverage cell in
  `wiki/versions.md`, which described this page's index-`reltuples` denominator,
  96-cell matrix, `drift >= 1.30` scores, GIN/BRIN disqualification, and
  measured counter-examples. The cell's other COMMENT-related text covers the
  surviving index/heap-ratio page and was left alone.
- Neutralized the remaining historical Markdown links in `wiki/log.md` (3) and
  `wiki/versions.md` (3) so they name the page title as plain text, and added a
  removal note to `## Coverage Notes`.
- v12 COMMENT-stored index-bloat coverage now runs through
  Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in
  COMMENT in PostgreSQL 12 (unverified)
  and Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in
  PostgreSQL 12? (unverified),
  and core-SQL bloat measurement through
  [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md).
- No source citation, pin, or verification field on any surviving page changed.
  `raw/postgres-12/` is unchanged and no server was started.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-12] answer v17 | which indexes need a rebuild for deduplication after pg_upgrade from 12

- Filed [Checking Whether an Index Needs a Rebuild to Enable Deduplication After
  pg_upgrade From PostgreSQL 12 to 17
  (unverified)](v17/questions/indexing/btree-deduplication-after-pg-upgrade.md)
  against unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa`
  (PostgreSQL 17.10).
- Prompt hygiene applied before drafting. The asker approved the corrected wording
  of "in postgreql 17 , question: after an database is upgraded from v12 to v17,
  how to check if an index needs to be rebuild to enable de-duplication.", chose a
  live `pg_upgrade` test over a source-only answer, and restricted the check itself
  to core SQL with no extensions.
- Verdict: every B-tree index `pg_upgrade` carries over from a v12 cluster is
  unable to deduplicate, and only a rebuild changes that. Source side:
  `BTMetaPageData`'s own comment ("Even version 4 indexes created on PostgreSQL v12
  will need a REINDEX ... pg_upgrade hasn't been taught to set the metapage
  field"), `_bt_metaversion`'s "we rely on the assumption that btm_allequalimage
  will be zero'ed", `_bt_upgrademetapage`'s "Only a REINDEX can set this field",
  `_bt_initmetapage` storing `_bt_allequalimage()`'s answer at build time, the
  build and insert deduplication gates in `_bt_load` and
  `_bt_delete_or_dedup_one_page`, and `pageinspect`'s identical assumption.
  `btm_version` is 4 both before and after, so version is not the discriminator.
- Deliverable is a core-SQL-only check with three parts: read byte 64 of block 0
  through `pg_read_binary_file(..., 0, 72, true)` + `get_byte`, mirror
  `_bt_allequalimage` in the catalogs (`indnatts = indnkeyatts`, a `pg_amproc`
  support function 4 per key opclass, `collisdeterministic` for
  `btvarstrequalimage` columns), and read the `deduplicate_items` reloption. Byte
  offsets were confirmed by compiling `offsetof` against the pin's own installed
  headers (contents at 24, `btm_version` 28, `btm_allequalimage` 64, v17 metapage
  `pd_lower` 72); the 12.2 headers have no such struct member at all.
- Measured on three isolated 12.2 -> 17.10 `pg_upgrade --copy` runs plus one
  ICU-enabled 17.10 cluster: 21/21 carried-over indexes read `pd_lower = 64`,
  version 4 and the flag false, while every 17-built index read 72/true; the probe
  agreed with `bt_metap()` (installed only as ground truth) in 75/75 row comparisons,
  57 before the rebuilds and 18 after;
  all 21 index files were MD5-identical between old and new data directories with
  OID and relfilenode preserved; the gate matched the engine's own `DEBUG1` verdict
  on 19 shapes, the one silent case being the `INCLUDE` index whose early
  `return false` skips the `elog`.
- Rebuild results: `REINDEX INDEX CONCURRENTLY` reclaimed 69.1% / 69.0% / 69.0% /
  68.7% / 68.5% / 67.4% / 67.4% on the seven duplicate-heavy indexes, 0.0% on the
  unique and empty ones, and left the five not-equal-image indexes identical to the
  byte. A fresh `deduplicate_items = off` twin measured 22,519,808 bytes, exactly
  the carried-over index's size, against 6,963,200 with deduplication on. The
  savings curve over rows per key ran 0.0% / 10.0% / 48.1% / 65.4% / 69.5% / 67.4%
  at 1 / 2 / 5 / 20 / 100 / 1000, and a carried-over index grew to 3.1x a fresh
  twin over the same 200,000 duplicate inserts.
- Also measured and filed: the `deduplicate_items` trap (metapage flag true, full
  22,519,808 bytes, engine still logging "can safely use deduplication"),
  `VACUUM FULL` and `CLUSTER` setting the flag while plain `VACUUM` does not, that
  `GRANT EXECUTE ON FUNCTION pg_read_binary_file(text,bigint,bigint,boolean)` is
  the requirement and `pg_read_server_files` is neither sufficient nor necessary,
  `relpages`/`reltuples` left at 0 (not the v14 `-1`) for all 21 indexes until the
  first `ANALYZE`, a single `pg_class.xmin` band of 534 that makes xmin useless as
  a durable marker, and `pg_largeobject_loid_pn_index` plus a TOAST index as the
  carried-over catalog cases. Both filed statements plus the priority query and the
  platform self-test were executed on the pin.
- Updated two open questions on
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md): the
  `pg_upgrade` reading it scoped out and the nondeterministic-collation false
  positive it could not measure are now both measured, and it also inherits the
  zeroed-`reltuples` blind spot right after an upgrade.
- Refreshed `wiki/index.md`, `wiki/v17/index.md`, and the v17 coverage cell plus a
  new coverage note in `wiki/versions.md`. `verified` stays `false` and
  `verified_by_agent` stays `not yet`: nine open questions remain, including
  single-platform byte offsets and an unobserved standby/replay case.
- Cleanup: all four test servers stopped and their data directories, tablespace
  directory and socket directory removed (5.2 GB reclaimed); the fixture and
  measurement scripts with their captured output remain under
  `.wiki-runtime/tmp/dedup-upgrade/` for re-verification. `raw/postgres-12/` and
  `raw/postgres-17/` were not modified.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-12] follow-up v17 | one B-tree bloat statement for servers 12 through 17

- Added eight follow-up sections to [Testing the PostgreSQL 12 Core-SQL B-Tree
  Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md)
  against unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa`
  (PostgreSQL 17.10), proposing one statement that runs unchanged on any server
  from 12 through 17 and credits deduplication only where the catalog proves the
  engine would deduplicate.
- Prompt hygiene applied before drafting. The asker approved the grammar
  correction of "propose a sql to measure bloat on postgresql v12 and later
  versions with support of deduplication", scoped "later versions" to servers 12
  through 17, and asked for the proposal to be built and measured on servers
  rather than derived from source alone.
- Five changes over the deduplication-aware sweep already on the page, each with
  a measured reason: posting tuples sized `MAXALIGN(base + tids * 6) + 4` under
  the 1/10-page cap with the high-key truncation credit on leaf capacity
  (`i_q5` −92.5% -> −2.5%, `i_qall` +2.8% -> +0.2%); key groups from a NULL-run
  plus most-common-value plus remainder mixture (`i_null` −27.7% -> −0.4%); a
  negative `n_distinct` credited only for whole-table indexes (`i_q5_part`
  49.0% -> 1.1%); a nondeterministic-collation conjunct in the gate; and both
  `reltuples` eras plus a `bloat_pct_floor` column and a `caveats` column.
- Source derivation, `raw/postgres-17/` only: `_bt_load`'s `maxpostingsize` of
  `MAXALIGN_DOWN(BLCKSZ * 10 / 100) - sizeof(ItemIdData)` (812 bytes),
  `_bt_dedup_save_htid`'s `MAXALIGN(basetupsize + n * sizeof(ItemPointerData))`
  test and `_bt_form_posting`'s `newsize`, `index_form_tuple`'s MAXALIGNed base,
  `_bt_buildadd`'s `pgspc + last_truncextra < btps_full` rule (which is why a
  leaf holds 9 x 808-byte posting tuples, giving 843 modelled leaves against an
  849-block rebuild for 1,000,000 rows under one key), `_bt_keep_natts_fast`
  treating two NULLs as equal, `_bt_allequalimage`'s INCLUDE refusal and support
  function 4 lookup, `btvarstrequalimage`'s nondeterministic-collation false,
  `pg_collation.collisdeterministic`, `analyze.c`'s 10%-of-rows `stadistinct`
  sign rule and MCV frequency storage.
- Exact-pin execution on three isolated servers built out of tree under
  `.wiki-runtime/` — 12.2, 14.23 and 17.10, the last configured `--with-icu` —
  with identical fixture DDL (a seven-point duplication band with partial
  siblings, NULL and hot-value skew, text and variable-width keys, unique,
  `INCLUDE` and multi-column shapes, `fillfactor = 50`,
  `deduplicate_items = off`, five real-bloat fixtures, a drained partial
  predicate, `TRUNCATE`-and-reload, grown-since-ANALYZE, empty, and two
  collation-only variants) and `CREATE INDEX CONCURRENTLY` rebuilds as ground
  truth. Same statement text everywhere: 68 / 68 / 72 indexes swept over
  133,677 / 94,910 / 103,056 blocks in 17.7 to 32.9 ms warm; 0 / 34 / 36
  indexes credited; `expected_blocks` identical to v12 Method A in 34 of 34
  scored cells on 12.2 and differing in 17 of 34 and 18 of 36 on 14.23 and
  17.10; one `unmeasured` row on each server, reported as
  `reltuples 0, table has live rows` on 12.2 and `reltuples unknown` on
  14.23/17.10.
- Accuracy against the rebuilds, excluding the unmeasured row: within 5% on
  31/31/31 cells on 12.2 (Method A / v17 sweep / proposal identical there),
  14 / 22 / 25 of 33 on 14.23, and 15 / 23 / 25 of 35 on 17.10. `i_dupdel`
  (10 keys, 90% deleted and vacuumed) reads 89.8% on 14.23 and 17.10 against a
  true 89.8% and 90.0% on 12.2 against a true 89.9%, from one statement.
- Alerting on `bloat_pct_floor` with `status = ok` and no `never analyzed` or
  `row-count sources disagree` caveat scores 4 true positives, 0 false
  positives and 1 false negative (`i_stale_part`, 94.6% true) on every server,
  against 2 to 3 false positives for the point estimate alone. The measured
  false-positive shapes are a healthy 28 MB nondeterministic-collation index
  (88.2% without the new conjunct, 0.1% with it), a hot-value skew index whose
  `n_distinct` came back 81,281 against a true 750,001 (53.4% point estimate,
  −20.9% floor, and 0.1% after `SET (n_distinct = -0.75)`), and a table doubled
  since its last `ANALYZE` (49.9%, caveat `row-count sources disagree`).
- Recorded ten new `## Open Questions`, including that majors 13, 15 and 16 were
  never run (13 combines deduplication with the pre-14 stale-zero hazard), the
  5.5% to 8.2% posting-list packing loss that is measured but not modelled, the
  caveats-over-suppression trade that keeps the `i_dupdelp` detection and the
  `i_stale_part` false negative, and two cumulative-statistics counter artifacts
  observed after `VACUUM (ANALYZE)`.
- Updated `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md`. Kept
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet`, because this is a new follow-up rather than a
  claim-by-claim re-verification of the whole page. All three servers were
  stopped, their data directories dropped and the two new out-of-tree build
  trees removed; the 14.23 and ICU-enabled 17.10 installs and the harness SQL
  stay under `.wiki-runtime/`, which is runtime state and not wiki content.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-12] follow-up v17 | does the dedup-aware bloat sweep work on v12 and v17?

- Added three follow-up sections to [Testing the PostgreSQL 12 Core-SQL B-Tree
  Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md)
  against unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa`
  (PostgreSQL 17.10).
- Prompt hygiene applied before drafting. The asker approved a grammar-corrected
  restatement of "does A deduplication-aware sweep for v17 works for indexes from
  v12 and v7?", confirmed that "v7" meant v17, and chose the reading "indexes on a
  live v12 server", i.e. does the statement parse, run and stay correct when
  pointed at a 12 server. The corrected prompt is restated under `## Question`
  with a prompt note; the `pg_upgrade` reading is scoped out and recorded in
  `## Open Questions`.
- Verdict: yes on both, because every v17-specific term is gated on a catalog
  fact a 12 server cannot produce. Source side (`raw/postgres-17/` only, as the
  page's version rule requires): `_bt_load`'s `deduplicate` condition is
  `allequalimage AND NOT isunique AND BTGetDeduplicateItems`, `allequalimage`
  comes from `_bt_allequalimage`, which needs a `BTEQUALIMAGE_PROC` (support
  number 4) per key column, `deduplicate_items` is a `RELOPT_KIND_BTREE` option,
  and `pg_class.reltuples` documents `-1` as unknown. Support numbers are bounded
  by the AM's `amsupport` (`bthandler`, `AlterOpFamily`'s `maxProcNumber`,
  `btvalidate`), so an older major cannot be made to advertise number 4. The
  three constructs first ship in `REL_13_0`, `REL_13_0` and `REL_14_0`
  (`612a1ab7672`, `0d861bbb702`, `3d351d916b2`), none an ancestor of `REL_12_2`,
  read from the same checkout's history.
- Exact-pin execution on two servers: the same DDL and generated data on an
  isolated 12.2 server and an isolated 17.10 server, each built from its own pin
  under `.wiki-runtime/`, no contrib installed, ground truth
  `CREATE INDEX CONCURRENTLY`. The statement ran unchanged on both: 32 indexes /
  10.9 ms on 12.2 and 34 / 10.5 ms on 17.10. On 12.2, 0 indexes credited,
  `all_equalimage` false on all 20 fixtures, `expected_blocks` identical to the
  v12 page's Method A in all 32 cells, and no miss worse than 4 blocks; on 17.10,
  15 of 34 credited and 15 differing. `idx_dupdel` (10 keys, 90% of 1,000,000
  rows deleted and vacuumed) reads 90.0% on 12.2 (model 276, rebuild 278, true
  89.9%) and 90.1% on 17.10 (model 84, rebuild 87, true 89.8%), where the
  uncorrected model says 67.5% on 17.10. Catalog probes: btree `max(amprocnum)`
  is 3 on 12.2 with zero rows at 4, `ALTER OPERATOR FAMILY ... ADD FUNCTION 4`
  fails with `invalid function number 4, must be between 1 and 3`,
  `ALTER INDEX ... SET (deduplicate_items = off)` fails with
  `unrecognized parameter`, and `array_lower(indclass, 1)` is 0 on both, so the
  `k < indnkeyatts` probe really does cover every key column.
- Two new findings. (1) An `INCLUDE` index with a low-cardinality included column
  is a v17 false positive of the sweep as filed: `_bt_allequalimage` refuses
  INCLUDE indexes outright while the SQL probe inspects only key columns, so
  `(a) INCLUDE (d)` with 1000 and 5 distinct values read 78.1% bloat on a
  0%-reclaimable 3853-block index. Adding `x.indnatts = x.indnkeyatts` to the gate
  makes that cell exact (model 3853 against a rebuild of 3853) and moves none of
  the other 33 indexes. (2) On 12.2 the `reltuples = -1` guard is dead code and the
  stale-*zero* hazard it replaced is still live: after `TRUNCATE` and reload a
  healthy 825-block index reads 99.9% bloat (collector `n_live_tup` 300,000,
  `pg_class` 0, so `least()` takes the zero) where 17.10 reports
  `unmeasured: reltuples unknown`. A zero-plus-size rule for 12 and 13 servers was
  executed as filed and returns exactly the one truncated index.
- Also recorded: the `n_distinct` sign boundary is unstable at ten rows per key
  (this run got the negative form where the page's earlier sweep got a positive
  97311), the posting-list-cap under-correction is 24 blocks at 1,000,000 rows per
  key and 69 at 1000, and one multi-column duplicate-key cell now exercises the
  per-column `n_distinct` product rule (842 modelled against a 897 rebuild).
- Updated `## Contents`, `## Question`, `## Context Reviewed`, `## Evidence Map`
  (7 new rows), `## Open Questions` (6 new items, plus corrections to the
  "no PostgreSQL 12 server was run" and multi-column notes), and
  `## Source References` (8 new entries). Refreshed `wiki/index.md`,
  `wiki/v17/index.md`, and `wiki/versions.md`. `verified` stays `false` and
  `verified_by_agent` stays `not yet`: this was a scoped follow-up, not a
  claim-by-claim re-verification of the whole page.
- Cleanup: both servers stopped, the two test databases dropped, the 17.10 data
  directory removed, and the temporary SQL scripts deleted.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-12] answer v17 | test the v12 core-SQL B-tree bloat method on 17

- Filed [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md)
  against unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa`
  (PostgreSQL 17.10). Prompt hygiene: the user approved correcting the prompt's
  "postgreql" and "acurace" typos and its trailing "from version 12" phrasing,
  and chose full-parity testing plus citing the v12 page's own numbers rather
  than re-running a PostgreSQL 12 server.
- Built the pin out of tree under `.wiki-runtime/pg17`, ran an isolated 17.10
  server (`autovacuum = off`, `pgstattuple` installed only as ground truth), and
  executed the v12 page's Methods A-D verbatim over three populations: the 15
  named fixtures, a 9-bloat-type x 3-scale x partial/non-partial 54-cell matrix
  (54 indexes over 27 tables), and a six-point duplication-ratio sweep. Ground
  truth per index is a `CREATE INDEX CONCURRENTLY` rebuild plus `pgstatindex`;
  Method B was driven through `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` so the
  plan's index name, `Heap Fetches` and buffer counts could be scored
  programmatically.
- Verdict: the SQL runs unchanged and is as accurate as the v12 page reports
  wherever keys are distinct - exact on 9 of 14 fixtures and all 24 non-partial
  matrix cells outside the duplicate-key type, `leaf_pages` exact on 12 of 12 fixtures and 36 of 36 eligible
  cells, `pg_column_size` moving the variable-width fixture from +105 blocks to
  -1, Method C exact by construction, and the stale-partial-index failure
  clearing with one `ANALYZE` (worst error 499 blocks to 1, against the v12
  page's 510 to 1). Nine of fifteen fixtures reproduce the v12 page's block
  count exactly, which is the control for the comparison.
- The v13 deduplication feature is the one break: `_bt_load` deduplicates
  non-unique all-equal-image indexes by default, so the per-row page-fill model
  overestimates a rebuild by +223.3% on an all-duplicate 1,000,000-row index,
  +92.5% at 5 rows per key on an index with 0% reclaimable, and +20.9% on a
  25%-NULL index; Method B's density formula reads 313.58% against
  `pgstatindex`'s 96.15%. Filed a dedup-aware sweep charging 6 bytes per extra
  TID, gated on `n_distinct > 0` (because `analyze.c` switches to a negative
  fraction above 10% distinct, and crediting that form turned a +105-block
  estimate into -271), on `deduplicate_items`, on non-uniqueness, and on the
  `pg_amproc` equal-image support function. It cuts the worst duplicate-key
  error to -24 blocks (2.9%) and changes no already-exact cell.
- Also measured and filed three smaller v17 differences: `pg_class.reltuples`
  now uses -1 for unknown and `TRUNCATE` restores that state, so the v12 SQL
  reports 100.0% bloat on an index a rebuild proves healthy (2745 against 2745)
  while the collector's `n_live_tup` read 2,000,000 for 1,000,000 rows, hence a
  guard that reports "unmeasured"; `VACUUM VERBOSE` prints four page classes and
  no index row count; and the v14 2%-of-pages index-vacuum bypass prints no
  index line at all (measured at 0.41% of pages, restored with
  `INDEX_CLEANUP ON`).
- Updated `wiki/index.md`, `wiki/v17/index.md`, and the v17 coverage note in
  `wiki/versions.md`. Kept `verified: false`, the visible `(unverified)` title,
  and `verified_by_agent: not yet`: the fixture recipes are reconstructions from
  the v12 page's descriptions, and the v12 comparison figures are attributions
  to that page rather than evidence from the v12 checkout.
- The isolated server was stopped and its 7.5 GB data directory, build tree, and
  socket directory were removed; the installed binaries and the executed SQL
  scripts remain under `.wiki-runtime/pg17/` for re-verification.
  `raw/postgres-17/` was not modified.

## [2026-08-11] follow-up v12 | swap the bloat denominator to the index's own reltuples

- Rebuilt Calibrating a COMMENT-Stored Bytes-per-Index-Row REINDEX Threshold in
  PostgreSQL 12 (unverified)
  around `pg_relation_size(index) / index reltuples`, against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2). Prompt hygiene:
  the user approved correcting the follow-up to "replace live rows by the index
  `pg_class.reltuples`", chose a whole-page re-measurement rather than a new
  section, and chose to rename both the title and the file, so
  `comment-stored-bytes-per-live-row-bloat.md` was deleted and its root-index,
  v12-landing-page, and `wiki/versions.md` entries were rewritten in place.
- Source model established first: three writers set an index's `reltuples` and
  they disagree about units. `index_update_stats()` stores the AM's own
  `IndexBuildResult.index_tuples`; plain `ANALYZE` stores
  `ceil(tupleFract * totalrows)`; `lazy_cleanup_index()` stores
  `IndexBulkDeleteResult.num_index_tuples` only when `estimated_count` is false.
  Per AM: B-tree, hash, GiST, SP-GiST and contrib `bloom` count one per indexed
  heap tuple; `ginbuild()` counts extracted entries (`indtuples += nentries`)
  while `ginvacuumcleanup()` substitutes the heap count under an explicit XXX;
  `brinbuild()` counts range summary tuples and `brinvacuumcleanup()` recounts
  them with `include_partial = false`; `btvacuumcleanup()` and
  `hashvacuumcleanup()` can return NULL, in which case nothing is written.
- Built the exact pin out of tree under `.wiki-runtime/tmp/bpr2/build`,
  installed to `.wiki-runtime/tmp/bpr2/inst` with contrib `bloom`,
  `pgstattuple` and `pageinspect`, and ran one isolated cluster on port 55432
  with `autovacuum = off`, `shared_buffers = 512MB`,
  `maintenance_work_mem = 256MB`.
- New 96-cell matrix (12 workloads x 8 indexes, seven capture phases each) with
  `REINDEX` ground truth. The swap is a no-op on ordinary indexes: drift under
  the index denominator equalled drift under the table denominator to four
  decimals in all 84 non-BRIN cells, and an index's `reltuples` equalled its
  table's in all 192 plain-`ANALYZE` observations. `drift >= 1.30` scores
  42/1/4/49 (94.8%) over 96 cells and 42/0/4/38 (95.2%) excluding BRIN; the four
  misses are all `w_churn2`, in-domain churn, where the fork stops growing while
  a rebuild still reclaims 26.28% to 56.15%.
- Partial indexes re-measured over 13 cells: the table denominator scores 8 of
  13, the index denominator 10 of 10 defined cells, and the three undefined
  cells are drained indexes with `reltuples = 0` that the explicit empty-index
  rule catches (2,260,992 bytes, 273 deleted pages, `avg_leaf_density` 0.05).
  Band 1.0163 benign to 1.3986 harmful.
- Disqualifying measurements: a 20-keys-per-row GIN index recorded 4,200,000
  rows for a 400,000-row table, so a post-rebuild baseline made the next reading
  drift 10.50 with no bloat; one 24,576-byte BRIN index reported 0.0123 and
  279.2727 bytes per index row at different times, a factor of 22,705, without
  moving a byte.
- Further new findings: `VACUUM (VERBOSE, ANALYZE)` printed no index line while
  a partial index's own population grew 50%, leaving a sampled 150,710 in place,
  and the `vacuum_cleanup_index_scale_factor = 0` reloption restored an exact
  310,000; `reltuples::numeric` rounds 20,000,020 to 20,000,000 through
  `float4_numeric()`'s `FLT_DIG` of 6, so the shipped SQL casts through
  `float8`; a GIN pending list moved the ratio down 32% and then up 74% while
  its probe ran 674x slower (4.043 ms / 497 blocks against 0.006 ms / 6);
  deleted-page bloat gave 28.50% reclaim with no query change but 413.33 of
  planner cost, against low-density bloat at 49.64% reclaim, half the scan
  blocks and no wall-clock gain; a partial index's predicate column turned
  1,800,000 HOT updates into 0 and quadrupled a sibling index while both partial
  indexes read drift 1.0000; and the plain-`ANALYZE` noise floor ran from
  -2.00%/+4.03% at a 10% predicate to `reltuples = 0` on four of six runs at
  0.001%.
- New `wiki_bpr_v3` capture and detection SQL (access-method allowlist,
  `am=`/`pred=` shape check, empty-index rule, `float8` cast, parameterized size
  floor) was executed on the pin, together with a five-step runbook using
  `REINDEX INDEX CONCURRENTLY`, which preserved the comment and left the exact
  build count of 50,000 on the replacement index. Comment durability was
  re-checked across `REINDEX`, `REINDEX INDEX CONCURRENTLY`, `RENAME`,
  `SET (fillfactor = 80)`, `VACUUM` and `ANALYZE`.
- The isolated server was stopped and its data directory, build tree, and
  fixtures were removed. Updated `wiki/index.md`, `wiki/v12/index.md`, and
  `wiki/versions.md`, and rewrote the two older log entries that linked the old
  filename. `verified` stays `false` and `verified_by_agent` stays `not yet`,
  so the visible title keeps its `(unverified)` suffix.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-11] follow-up v12 | B-tree partial indexes in the bytes-per-live-row page

- Extended the page then titled Calibrating a COMMENT-Stored Bytes-per-Live-Row
  REINDEX Threshold in PostgreSQL 12, since renamed to Calibrating a
  COMMENT-Stored Bytes-per-Index-Row REINDEX Threshold in PostgreSQL 12
  (unverified),
  with a new `### B-tree partial indexes: a comprehensive analysis` section,
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene: the user approved correcting the follow-up
  to "Add a section with a comprehensive analysis for B-tree partial indexes.
  Also include this in the analysis: Partial indexes: the denominator is wrong
  by construction." and chose to fold the existing short partial-index section
  into the new one rather than keep both.
- Rebuilt the exact pin out of tree under `.wiki-runtime/tmp/pg12/build`,
  installed to `.wiki-runtime/tmp/pg12/install`, and ran one isolated cluster on
  port 55432 with `autovacuum = off`, `shared_buffers = 512MB`,
  `maintenance_work_mem = 256MB`, plus contrib `pgstattuple`.
- Source work: the three predicate filters (`heapam_index_build_range_scan()`
  counting `reltuples` before the predicate test, `ExecInsertIndexTuples()`,
  `compute_index_stats()`), the four writers of an index's own
  `pg_class.reltuples`, `compute_index_stats()`'s skip of non-partial
  non-expression indexes (why `tupleFract` stays 1.0), `analyze.c`'s deliberate
  index skip under `VACOPT_VACUUM`, `lazy_cleanup_index()`,
  `btvacuumcleanup()`/`_bt_vacuum_needs_cleanup()` and the
  `vacuum_cleanup_index_scale_factor` GUC (`PGC_USERSET`) and B-tree reloption
  (`ShareUpdateExclusiveLock`), `_bt_initmetapage()`'s `-1.0` sentinel,
  `_bt_pagedel()` and `btvacuumpage()`, `RelationGetIndexAttrBitmap()`'s
  predicate `pull_varattnos()` and `heap_update()`'s HOT gate,
  `get_relation_info()`'s partial-index `estimate_rel_size()` branch,
  `genericcostestimate()`'s page charge, and `check_index_predicates()`.
- Measurements: 13 partial-index cells on 1,000,000-row tables with a 10%
  `st = 'open'` predicate and a 10% `done_at IS NULL` predicate, `REINDEX` size
  delta as ground truth. The table denominator scored 8 of 13 (3 TP, 2 FP, 3 FN,
  5 TN); the index's own `reltuples` scored 11 of 11 defined cells (4 TP, 0 FP,
  0 FN, 7 TN) with a 1.0243-to-1.3795 band, and the two undefined cells are the
  indexes at `reltuples = 0`, caught by an explicit empty-index rule.
- Also measured: `pgstatindex` separating deleted-page bloat (66.59% reclaim,
  546 deleted pages, density 89.18, 279 to 276 scan blocks, cost 5069.43 to
  2869.30 = 550 unread pages x `random_page_cost` 4.0) from low-density bloat
  (49.64% reclaim, density 45.06 to 89.83, 276 to 139 blocks); a fully drained
  index at 273 of 276 pages deleted that `VACUUM` still scans in full; the
  plain-`ANALYZE` noise floor at five predicate selectivities, from
  -3.87%/+2.97% at 10% to `reltuples = 0` on four of six runs at 0.001%;
  `VACUUM (ANALYZE)` writing exactly 100,000 three times where plain `ANALYZE`
  wrote 102,634 and 99,900; a `VACUUM` that printed no index line at all after
  the heap grew 5% while the partial index's population grew 50%; and two
  identical tables where the same 1,800,000 updates produced 0 versus 1,800,000
  HOT updates and a 49.96%-reclaimable sibling index, purely because a partial
  index's predicate names the updated column.
- New `wiki_bpr_v2` capture and detection SQL (denominator chosen from
  `indpred`, stored `pred=` guard, empty-partial-index rule) was executed on the
  pin, plus a five-step runbook with `REINDEX INDEX CONCURRENTLY` in which the
  table denominator would have reported 1.0526 for a 49.64%-reclaimable index
  and 1.0000 for a 99.64%-reclaimable one.
- Updated the page's Contents, Verdict, denominator-choice and operating-rules
  sections, `## Context Reviewed`, `## Evidence Map` (9 new rows),
  `## Open Questions` (8 new entries), `## Source References` (41 new
  citations), plus the root index, the v12 landing page, and version coverage.
  Kept `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet`.
- Verified every citation range against the pinned files, all 23 Contents
  anchors and every in-page anchor link, then stopped the isolated server and
  removed its data directory, build tree, and fixture scripts.
  `scripts/wiki_lint` reports 0 errors and 0 warnings.

## [2026-08-11] answer v12 | calibrate a COMMENT-stored bytes-per-live-row REINDEX threshold

- Filed the page then titled Calibrating a COMMENT-Stored Bytes-per-Live-Row
  REINDEX Threshold in PostgreSQL 12, since renamed to Calibrating a
  COMMENT-Stored Bytes-per-Index-Row REINDEX Threshold in PostgreSQL 12
  (unverified),
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene: the user approved fixing "by record" to
  "Record" and reading "postgreql 12" as PostgreSQL 12; the rest of the prompt
  is restated verbatim under `## Question`.
- Built the exact pin out of tree under `.wiki-runtime/tmp/pg12-build`, installed
  it to `.wiki-runtime/tmp/pg12-install`, and ran one isolated cluster on port
  55432 with `autovacuum = off`, `shared_buffers = 512MB`.
- Calibration matrix: 11 workloads x 8 indexes (btree, hash, gist, spgist, gin
  with `fastupdate` on and off, brin, contrib `bloom`), 200,000-row fixtures,
  `REINDEX TABLE` as ground truth. Best single threshold is 1.30 to 1.40 at
  43 TP / 5 FP / 0 FN / 40 TN (94.3%); on the 66 cells excluding BRIN and
  SP-GiST, 1.30 and 1.40 are exact. Per-access-method bands give +30% for
  B-tree, GiST, and hash, +40% for GIN, +60% for `bloom`; SP-GiST's benign
  2.2041 exceeds its harmful 2.0000, and BRIN had no harmful cell at all.
- Source basis: no v12 index AM truncates its own main fork (SP-GiST's
  `RelationTruncate()` is inside `#ifdef NOT_USED`), so VACUUM only records
  reusable pages via `RecordFreeIndexPage()` and bytes are a high-water mark;
  hash has no such call and reuses overflow pages through its own bitmap, and
  `_hash_alloc_buckets` extends the fork by a whole splitpoint; `gistchoose()`
  breaks ties with `random()`; GIN's `fastupdate` defaults to on and every scan
  reads the pending list first; `brininsert()` skips unsummarized ranges and
  `bringetbitmap()` returns all their pages; `brinbulkdelete()` is a no-op.
- Measured counter-examples: drift exactly 2.0000 for all eight indexes in each
  of the three delete workloads while reclaim spanned 0.00% to 54.19%; a GIN
  pending list at 4.1231 whose `gin_clean_pending_list()` fix removed a 12.0x
  query slowdown while raising the ratio to 4.6182 (1,471 pending pages,
  200,000 pending tuples); 20-keys-per-row GIN at 6.3176 with a 16.85% larger
  rebuild; an unsummarized BRIN range set at 0.1000 costing 49,961 blocks
  against 1,411 and 150.088 ms against 6.144 ms; a partial index at 0.3640 with
  65.66% reclaimable; `reltuples` versus `n_live_tup` differing 2x after an
  unvacuumed delete, with `pg_stat_reset()` zeroing the latter; 400,000 HOT
  updates (`fillfactor = 40`) moving no index byte against 14 HOT updates at
  the default fillfactor; a 1.322% GiST baseline spread over ten identical
  builds; and a 0.173% `reltuples` spread over six identical `ANALYZE` runs on
  a 76,097-page table.
- Rebuild payoff, measured with the visibility map held constant: 49.96% space,
  -49.9% index-only-scan blocks, -9.9% time, 0% on a point lookup, -4.4% blocks
  on a heap-fetching query. Also recorded the v12 behavior that a first VACUUM
  after churn leaves pages not-all-visible, which inflated an earlier run 10x
  until a second VACUUM was added.
- Capture SQL, detection SQL, and a five-step end-to-end runbook were executed
  on the pin. The first detection run failed with `ERROR: division by zero` on
  an empty partition child index; the filed SQL uses `nullif(table_rows, 0)`.
  Comment durability was confirmed across `REINDEX INDEX`,
  `REINDEX INDEX CONCURRENTLY`, `ALTER INDEX ... RENAME`,
  `ALTER INDEX ... SET (fillfactor)`, `VACUUM`, and `ANALYZE`.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md` (v12
  coverage cell plus a coverage note). `verified: false` and
  `verified_by_agent: not yet` because the page was not re-audited claim by
  claim after drafting.
- Cleanup: the isolated server was stopped and its data directory, build tree,
  install tree, and scratch SQL under `.wiki-runtime/tmp/` were removed;
  `raw/postgres-12/` is unchanged.

## [2026-08-11] remove v12 | delete Detecting Index Bloat with COMMENT-Stored Bytes per Tuple

- Removed `wiki/v12/questions/indexing/comment-stored-bytes-per-tuple-bloat.md`
  per user request.
- Removed the active page entry from `wiki/index.md` and `wiki/v12/index.md`,
  and the three inbound `## Navigation` links in
  `comment-stored-table-dml-counters-gin-reindex.md`,
  `physical-index-statistics-tuple-counts-and-bytes.md`, and
  `../storage-and-vacuum/pgstattuple-approx-heap-bloat.md`. No surviving page
  referred to the deleted page in prose.
- The v12 coverage cell in `wiki/versions.md` never described this page, so no
  coverage clause was dropped; its COMMENT-related text covers the surviving
  index/heap-ratio page.
- Neutralized the remaining historical Markdown links in `wiki/log.md` (1) and
  `wiki/versions.md` (1) so they refer to the page title as plain text, and
  added a removal note to `## Coverage Notes`.
- v12 COMMENT-stored index-bloat coverage now runs through
  Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in
  COMMENT in PostgreSQL 12 (unverified)
  and Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in
  PostgreSQL 12? (unverified).
- No source citation, pin, or verification field on any surviving page changed.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-11] cleanup | removed all scratch sandboxes from .wiki-runtime/tmp

- Removed every scratch sandbox under `.wiki-runtime/tmp/` at the user's
  request, reclaiming 12 GB: `bpt2/` (2.6 GB), `hbloat/` (4.4 GB), `ioguix/`
  (4.0 GB), and `ffcorr/` (1.1 GB). Disposable cluster data directories
  accounted for nearly all of it, with `pg_wal` alone at 2.2 GB, 2.2 GB,
  3.5 GB, and 1.1 GB respectively. The 316 KB of scratch SQL, shell drivers,
  and `.out` transcripts across 61 files was removed with them, on the user's
  instruction, because the filed pages already carry the results.
- This resolves the outstanding state recorded in the fillfactor-corrected
  `pgstattuple_approx` entry below, which noted three postgres servers left
  running from earlier sessions. All three were stopped cleanly with
  `pg_ctl -D <datadir> -m fast stop` before any deletion; no `postmaster.pid`
  and no postgres process remained afterwards.
- `bpt2/` also held the only out-of-tree PostgreSQL 12.2 build (184 MB) and
  install (67 MB), which every one of those servers ran from, with `bloom`,
  `pageinspect`, `pg_freespacemap`, `pg_visibility`, and `pgstattuple`. No
  compiled install now remains, so the next exact-pin experiment must rebuild
  from `raw/postgres-12/`.
- Kept `venv/`, `logs/`, and `cache/wiki_lint/last-run.txt`, which the tooling
  reads and appends to, plus the empty `indexes/` scaffold that
  `ensure_runtime_dirs()` recreates. `.wiki-runtime/` is now 13 MB.
- No wiki page, index, version pin, or `raw/` checkout changed; this entry is
  the only edit. `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors,
  0 warnings.

## [2026-08-11] review-fix v12 | fillfactor-corrected pgstattuple_approx metric

- Reframed Proposing and Testing a fillfactor-Corrected pgstattuple_approx
  Metric for Table Heap Bloat in PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied first: the user chose silently
  corrected wording, a rewritten `## Question`, and fresh measurements. The
  filed question is now "Propose and test the use of `pgstattuple_approx`,
  corrected for the table's `fillfactor`, to measure table heap bloat", and
  the title, verdict, section order, and both index entries follow it.
- Added a source derivation the page did not have: the `RELOPT_KIND_HEAP`
  `fillfactor` entry with default 100, bounds 10 and 100, and
  `ShareUpdateExclusiveLock`; `RelationGetFillFactor()` /
  `RelationGetTargetPageUsage()` / `RelationGetTargetPageFreeSpace()` and the
  integer truncation that makes the 70% reserve 2,457 bytes; the four heap
  callers of that reserve (`RelationGetBufferForTuple()`,
  `heap_multi_insert()`, `raw_heap_insert()`, `heap_page_prune_opt()`);
  `make_new_heap()` reloption inheritance; the `ATRewriteTable()`
  `table_tuple_insert()` path; and the documented partitioned-table and TOAST
  refusals.
- Re-measured on a fresh isolated 12.2 cluster built from the pin
  (`autovacuum = off`, `shared_buffers = 256MB`). The nine-fixture accuracy
  matrix, the approx-versus-exact table, the 35-page emptied-page
  reconciliation (123,968 / 162,752 / 111,720 / 163,500), and the step 1 to 3
  worked example all reproduced their filed values; `d_bloat` ground truth is
  now recorded as 59.99 with error −1.86.
- New results: a seven-point fillfactor sweep (uncorrected 0.44 to 91.02 on
  unbloated tables, corrected 0.02 to 1.79 except 10.16 at `fillfactor = 10`);
  a closed-form residual model from the page-close condition matching all
  seven within 0.05 points, with per-page geometry from `pageinspect` and
  `pg_freespacemap`; three bloated fillfactor fixtures at −1.51, −0.78, −1.42
  corrected against +14.45 and +34.58 raw; `VACUUM FULL` and `CLUSTER` both at
  1,250 blocks / 40 rows per page where `ALTER COLUMN TYPE` gives 1,316 / 38;
  `h_up`, whose rewrite grows the table 44.93% and is reported only as the
  unclamped −42.23; `h_down` at +0.25; `h_wide70` where the declared reserve
  absorbs the emergent tail (0.11 against 30.08 raw); and the reloption
  boundary matrix (9 and 101 rejected, partitioned and `toast.fillfactor`
  rejected, materialized view accepted, `ShareUpdateExclusiveLock` observed).
- Corrected one filed claim: the `m_hot` live-count delta is not 18 rows. With
  every page skipped, `approx_tuple_count` returns `pg_class.reltuples`
  verbatim, and three consecutive `ANALYZE` runs moved it by −379, +697, and
  −279 rows on the same 500,000-row table. Also added that plain VACUUM
  truncates trailing empty pages, so the retained-line-pointer surcharge is an
  interior-page effect.
- Changed the recommended SQL: step 1 now reports `fillfactor`, and step 2
  drops its `greatest(..., 0)` clamp so a rewrite that would enlarge the table
  is reportable. Added four Open Questions (single row width in the residual
  model, the unswept 10-to-30 fillfactor interval, index fillfactors out of
  scope, and the existing threshold-calibration gap).
- Test objects were dropped and the isolated server was stopped. Three
  postgres servers from earlier sessions were left running under
  `.wiki-runtime/tmp/`; they were not touched. Kept `verified: false`, the
  visible `(unverified)` title, and `verified_by_agent: not yet`: the
  fillfactor claims are freshly measured, but the cost, TOAST, snapshot,
  privilege, and catalog-estimator numbers were carried over from the original
  run rather than re-executed. Updated `wiki/index.md`, `wiki/v12/index.md`,
  and `wiki/versions.md`. `scripts/wiki_lint` reports 0 errors and 0 warnings.

## [2026-08-11] answer v12 | pgstattuple_approx for table heap bloat

- Filed Proposing and Testing pgstattuple_approx for Table Heap Bloat in
  PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied first; the user chose silently
  corrected wording, full exact-pin measurements, and a comparison against both
  exact `pgstattuple` and a catalog-only estimate. This is the first page in
  the new `wiki/v12/questions/storage-and-vacuum/` category directory.
- Source review covered `pgstatapprox.c` end to end plus `pgstattuple.c`,
  `vac_estimate_reltuples()`, `lazy_scan_heap()`'s all-visible and free-space
  paths, every `visibilitymap_set`/`visibilitymap_clear` call site,
  `GetRecordedFreeSpace()` and the free-space-map categories,
  `PageGetHeapFreeSpace()`, `PageRepairFragmentation()`, `raw_heap_insert()`,
  `reform_and_rewrite_tuple()`, `HeapTupleSatisfiesVacuum`/`Dirty`, ANALYZE's
  width and `relallvisible` accounting, the extension SQL scripts and grants,
  the documentation chapter, and the module's regression coverage.
- Built the exact pin out of tree under `.wiki-runtime/` and ran an isolated
  server with `autovacuum = off`, installing `pgstattuple`, `pageinspect`,
  `pg_freespacemap`, and `pg_visibility` from the same checkout. Ground truth
  is `VACUUM FULL` main-fork size reduction.
- Cost: 6.9 ms and 42 physical reads versus `pgstattuple`'s 484.8 ms and
  137,932 reads on a 1.08 GB all-visible table; 9.5 ms with 1% of pages dirty;
  216.7 ms versus 470.8 ms with no visibility map. The relation's free space
  map fork is 36 pages and its visibility map fork 5.
- Accuracy: over nine fixtures the `fillfactor`-corrected metric ran −3.11 to
  +1.79 points on the eight ordinary row shapes. The uncorrected
  `free + dead` signal produced a 31.25-point `fillfactor` false positive and a
  14.45-point overstatement. Wide 2,832-byte rows produced a 30.08-point false
  positive that no correction from this function's output can remove. Dropped
  columns produced an 84.22-point false negative; v12's retained line-pointer
  arrays cost 2 to 7 points, reconciled page by page with `pageinspect` and
  `pg_freespacemap` (`approx_tuple_len` equals `sum(8192 - fsm)` byte for
  byte).
- Also filed: the exact `reltuples` dependency of `approx_tuple_count` (a
  forged `reltuples = 7` returned verbatim when every page is skipped, then 73
  through the documented density blend), integer-truncated `scanned_percent`, a
  15.69-point phantom dead-tuple reading during an uncommitted bulk insert,
  47.65% "bloat" that `VACUUM FULL` could not reclaim under an old snapshot,
  the relation-kind and error matrix including the refused TOAST relation and
  the v12 sequence quirk, the `pg_stat_scan_tables` privilege boundary, and a
  catalog-only estimator that was exact on all nine well-behaved fixtures and
  wrong by 21.32, 79.99, and −5698.78 points on alignment-padding,
  stale-statistics, and never-analyzed tables.
- Six Open Questions recorded, including the untested 32-bit `scanned_percent`
  overflow above roughly 328 GiB scanned and untested standby behaviour.
- All test relations and the test role were dropped, the isolated server was
  stopped, and its 3.1 GB scratch build, install, and data directories under
  `.wiki-runtime/tmp/approx12/` were removed. `raw/postgres-12/` was unchanged
  throughout.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. The
  page is filed with human `verified: false`, the visible `(unverified)` title,
  and `verified_by_agent: not yet`.

## [2026-08-11] lint v12 | physical index statistics review

`wiki_lint` passed with 0 errors and 0 warnings after the review corrections to
the PostgreSQL 12 physical-index-statistics page and the updates to the global
index, v12 landing page, version coverage notes, and wiki log. A separate
citation-resolution pass confirmed all 354 source links resolve to existing
files with in-range line fragments.

## [2026-08-11] review-fix v12 | physical index statistics, tuple counts, and bytes

- Rechecked the complete [Physical Index Statistics, Tuple Counts, and Bytes
  per Tuple in PostgreSQL 12
  (unverified)](v12/questions/indexing/physical-index-statistics-tuple-counts-and-bytes.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Claim-by-claim source audit of all sections plus every
  source citation.
- Corrected factual claims. `pgstattuple.tuple_len` sums `lp_len`, and both
  B-tree and hash pass an already-`MAXALIGN`ed `IndexTupleSize()` to
  `PageAddItem`, so the payload average *includes* each item's alignment
  padding; the page previously said it excluded alignment. The three
  `pgstattuple` averages now cast to `numeric` because every count and length
  column is `bigint`. A bare `block_size` identifier in the formula table
  became `current_setting('block_size')::bigint`.
  `btm_last_cleanup_num_heap_tuples` is written by `btbulkdelete()` as well as
  `btvacuumcleanup()`. The `ANALYZE` skip is the full guard
  `!inh && !(params->options & VACOPT_VACUUM)`.
  `PageIndexTupleDeleteNoCompact()` returns the tuple's bytes to free space
  and retains only the line-pointer slot. `get_raw_page()` requires superuser
  and rejects partitioned indexes. `brinbulkdelete()` still allocates and
  returns a zeroed result struct. `pg_am` stores four columns. `pgstathashindex()`
  returns `version` first. The `pd_prune_xid` sentence now quotes `bufpage.h`
  instead of softening it, and the GIN posting-tree sentence admits the
  uncompressed pre-9.4 leaf layout. The `estimate_rel_size()` paragraph quotes
  the source's own "kluge" wording and adds the zero-`relpages` fallback.
- Corrected the test-coverage claim: `contrib/pgstattuple/sql/pgstattuple.sql`
  is 119 lines, not 63, and it *does* call generic `pgstattuple()` on an index
  — the partitioned-index root `test_partitioned_index`, as an expected
  failure. The absence claim is now scoped to indexes with storage and is
  backed by the `contrib/bloom` regression file and the heap-only
  `vacuum-reltuples` isolation spec.
- Added two Open Questions. The BRIN `revmapNumPages = lastRevmapPage - 1`
  entry now shows the off-by-one arithmetic (revmap pages occupy blocks
  `1..lastRevmapPage`) and its consumer in `brincostestimate()`, whose
  hypothetical branch computes a rounded-up count. The new entry records
  `bufpage.h`'s "currently unused in index pages" comment against GIN's
  `pd_prune_xid` reuse, GIN being the only index AM in the checkout that
  touches the field.
- Repointed or rebounded roughly 40 citations, including the BRIN revmap
  reader (`brinSetHeapBlockItemptr` -> `brinGetTupleForHeapBlock`), the GiST
  VACUUM leaf count (`gistvacuumscan` setup -> `gistvacuumpage`), the
  SP-GiST leaf count (leaf-root special case -> `vacuumLeafPage`),
  `pg_relpages` -> `pg_relpages_v1_5`, `blbuildCallback` ->
  `bloomBuildCallback`, and the `pgstattuple(text)` result declaration ->
  the `regclass` overload the page's own query calls. Added missing support
  for the normal insert path (`ExecInsertIndexTuples`), `ambulkdelete`
  (`lazy_vacuum_index`), fork-name strings (`relpath.c#forkNames`), GIN
  pending-list accounting (`ginfast.c`), Bloom's line-pointer-free layout
  (`BloomPageAddItem`), and the `pg_freespace` SQL signature.
- Added a cost note to the catalog inventory SQL: five volatile size calls run
  per row, so narrow the query or drop the redundant `pg_table_size()` column.
  The SQL itself was re-verified against the pinned catalogs, `pg_proc.dat`,
  `relpath.c` fork names, and the grammar, and is unchanged.
- Human `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are unchanged. The residual gap is stated on the
  page and here: this review re-read pinned source only and did not re-execute
  the page's SQL against a live 12.2 server.

## [2026-08-11] lint v12 | physical index statistics, tuple counts, and bytes

`wiki_lint` passed with 0 errors and 0 warnings after filing the PostgreSQL 12
physical-index-statistics page and updating the global index, v12 landing page,
version coverage notes, and wiki log.

## [2026-08-11] answer v12 | physical index statistics, tuple counts, and bytes

- Filed [Physical Index Statistics, Tuple Counts, and Bytes per Tuple in
  PostgreSQL 12
  (unverified)](v12/questions/indexing/physical-index-statistics-tuple-counts-and-bytes.md)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied before drafting; the user approved
  the corrected wording, which is restated verbatim under `## Question`.
- Mapped shared `pg_class`/`pg_index` fields, relation forks, FSM encoding,
  generic page overhead, and persistent B-tree, hash, GiST, GIN, SP-GiST,
  BRIN, and Bloom metadata. The page separates catalog estimates, stored
  counters, per-page observations, serialized item bytes, and whole-file byte
  shares.
- Traced every `reltuples` writer through build, standalone `ANALYZE`, and
  `VACUUM`; documented the valid row/item/summary denominators; and preserved
  the pinned GiST `pgstattuple`, BRIN revmap/planner/counting, GiST
  `pages_removed`, index-FSM, and live/dead reporting discrepancies under
  explicit limits and open questions.
- An isolated server built from the exact pin executed the catalog inventory
  SQL. A seven-AM fixture reproduced AM-specific build values, the
  standalone-`ANALYZE` row-estimate normalization, and AM-specific post-VACUUM
  values. The B-tree/hash `pgstattuple` block was instead verified against the
  pinned extension declaration and implementation because that runtime build
  did not install the module; the server was stopped after testing.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`.
  Human `verified: false` and `verified_by_agent: not yet` remain.

## [2026-08-11] lint v12 | COMMENT-stored table DML counters for GIN

`wiki_lint` passed with 0 errors and 0 warnings after filing the PostgreSQL 12
GIN table-counter inspection page and updating the global index, v12 landing
page, version coverage notes, and wiki log.

## [2026-08-11] answer v12 | COMMENT-stored table DML counters for GIN

- Filed Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in
  PostgreSQL 12? (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied before drafting; the user approved
  the corrected wording, which is restated verbatim under `## Question`.
- The source review establishes that the three proposed counters are table-wide
  attempted heap actions rather than per-index writes. It adds
  `n_tup_hot_upd` because HOT updates create no index entries, traces non-HOT
  updates through the executor into `gininsert`, traces deletes through VACUUM,
  and maps operator-class key extraction, the pending list, entry-tree key
  retention, posting-tree cleanup, page reuse, and fresh GIN builds.
- Repeated isolated exact-pin tests produced both decisive counterexamples:
  4,000 HOT updates on a 10,000-row table reached the original 40.00% score
  while the 131,072-byte GIN index remained unchanged and `REINDEX` reclaimed
  0.00%; deleting 100 high-cardinality rows reached only 1.00%, but the
  5,652,480-byte index remained at that size after VACUUM and rebuilt to 49,152
  bytes, reclaiming 99.13%. Both comments survived ordinary reindexing.
- Added tested production-shaped capture and detector SQL with session-scoped
  timeouts, schema allowlisting, physical/live/ready/valid/full-index filters,
  HOT subtraction, table-OID and `stats_reset` guards, strict parsing,
  human-comment preservation, and `inspect GIN; do not auto-reindex` at 40%.
  A second exact-pin fixture parsed the stored payload and reported 215
  non-HOT updates, or 21.50%, from 400 total updates.
- Documented mandatory post-success recapture, ordinary and concurrent
  reindex comment preservation, collector lag/reset/crash/`track_counts` and
  TRUNCATE boundaries, dump/restore and `CREATE TABLE LIKE` hazards,
  pending-list inspection, COMMENT locks and visibility, generated catalogs,
  extension/operator-class boundaries, upstream tests, and the source/docs
  discrepancy for `VACUUM (INDEX_CLEANUP false)`. Updated `wiki/index.md`,
  `wiki/v12/index.md`, and `wiki/versions.md`. Human `verified: false` and
  `verified_by_agent: not yet` remain pending local calibration and the open
  boundaries recorded on the page.

## [2026-08-07] follow-up v12 | COMMENT-stored bytes-per-tuple bloat calibration

Added the follow-up question and answer to the PostgreSQL 12 COMMENT-stored bytes-per-tuple bloat page. It establishes that the pinned source cannot derive a universal per-access-method increase rate, identifies operation-dependent allocation boundaries across the AMs, and specifies cohort calibration against measured rebuild yield before any automatic reindex decision.

## [2026-08-07] lint v12 | COMMENT-stored bytes-per-tuple bloat calibration

`wiki_lint` passed with 0 errors and 0 warnings after the follow-up update.

## [2026-08-07] answer v12 | COMMENT-stored bytes per tuple index-bloat screen

- Filed Detecting Index Bloat with COMMENT-Stored Bytes per Tuple in
  PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2). Prompt hygiene applied before drafting; the user approved
  the corrected wording, which is restated verbatim under `## Question`.
- The design stores actual index-main-fork bytes per estimated indexed heap row
  in a versioned index comment. The value `1.0` is the normalized starting
  drift, not the literal stored BPT. A later `1.4` result is labeled
  `investigate`, not an unconditional `REINDEX` decision.
- Source review found an all-AM denominator discontinuity immediately after
  index creation: common index code writes each AM's build count to
  `pg_class.reltuples`, GIN counts extracted entries, and BRIN counts range
  summaries. The workflow therefore runs plain table `ANALYZE` after creation
  and before each comparison; v12 then writes an estimated indexed-row count,
  including sampled partial-index selectivity. `VACUUM (ANALYZE)` is not used
  for this normalization because that plain-ANALYZE index update is skipped on
  the combined path.
- Added production-bound capture, detection, and guarded post-rebuild recapture
  SQL with session-scoped `statement_timeout` and `lock_timeout`, schema
  allowlisting, physical/live/ready/valid filters, one-time size measurement,
  guarded metadata parsing, human-comment preservation, and a material-byte
  column that is explicitly not called reclaimable bloat.
- Built and installed the exact 12.2 pin under `.wiki-runtime/`, installed
  contrib Bloom, and ran an isolated server with autovacuum disabled. All final
  SQL executed successfully. The seven-AM baseline covered B-tree, hash, GiST,
  SP-GiST, GIN, BRIN, and Bloom. An 80% scattered-delete BRIN fixture reached
  drift `5.0000` while rebuilding reclaimed `0.0%`; a legitimate GIN key-shape
  change reached drift `137.2000`, and rebuilding reproduced the same
  22,478,848-byte file. These cases disprove a universal all-AM threshold.
- Documented per-AM interpretation, the absence of a generic bloat callback,
  COMMENT ownership/lock and single-row semantics, generated-catalog and
  extension boundaries, dump/restore loss with `--no-comments`, plain versus
  concurrent reindex cost and restrictions, invalid concurrent-reindex
  leftovers, comment survival across the swap, and the required successful
  post-rebuild recapture step.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Human
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` remain because production thresholds,
  partial-index sampling variance, and third-party AMs require local
  calibration. The isolated server was stopped. Full `scripts/wiki_lint`
  reports 0 errors and 0 warnings; `git diff --check` passes.

## [2026-08-06] answer v17 | index/heap ratio in COMMENT, copied from v12 and reviewed

- Filed Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio
  in COMMENT in PostgreSQL 17 (unverified)
  against unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa`
  (PostgreSQL 17.10), copying the v12 question of the same name.
- Prompt hygiene applied before drafting. The user chose silently corrected
  wording, the copy request itself as the filed `## Question`, a full
  re-measurement on v17, and a since-v12 history section.
- No v12 measurement was carried over. Built the exact pin out of tree under
  `.wiki-runtime/tmp/pg17-build`, installed contrib `bloom`, `pgstattuple` and
  `pageinspect`, and ran an isolated 17.10 server with `autovacuum = off`.
- Reproduced every fixture family on v17: a three-scale fresh-ratio invariance
  sweep, a day-zero sweep over three heap row widths, a seven-point hash/B-tree
  sawtooth, a fresh-B-tree sweep from 0 to 1,000,000 rows, a 49-cell matrix of
  seven workloads times seven access methods with `REINDEX TABLE` ground truth,
  drift-threshold and ground-truth-threshold sweeps, a GIN
  baseline-1.004900 stepwise churn fixture, a delete-and-reload cycle in one-,
  two- and three-VACUUM arms with `pgstatindex`/`gin_metapage_info`/
  `hash_metapage_info` probes, a TOAST denominator fixture, 300-index survey
  timings against the `relpages` variant, a nine-operation comment durability
  matrix, lock/ownership/read-visibility probes, RIC-versus-plain-`REINDEX`
  size equality, the four RIC refusal cases, and a `statement_timeout`-
  interrupted RIC on a 3,000,000-row table.
- Scores: `drift >= 1.40` at 13 true / 3 false positives, `index > heap` at
  6 / 2, with 19 cells in the drift 0.90-1.10 band spanning -8.4% to 80.0%
  reclaimable. Verdict changed from the v12 page's: SP-GiST is now disqualified
  in these fixtures (fresh ratio drifts 2.2787 over 16x growth at 0.0%
  reclaimable).
- Two new source-traced findings. `hashbuild` sizes the initial bucket array
  from `estimate_rel_size`, so identical 200,000-row data produced 923 blocks on
  a narrow heap, 5,122 on a wide never-analyzed one, and 822 with `ANALYZE`
  first; this added an `ANALYZE`-first rule to Capture discipline. And v14's
  same-VACUUM B-tree recycling is present but did not fire on an idle server —
  the cycle still left B-tree +99.27%, GiST +89.47% and GIN +82.72% after one
  VACUUM, while a repeat against a concurrent `txid_current()` consumer
  populated the FSM and ended at 745 blocks instead of 1,098, matching the
  condition stated in `_bt_pendingfsm_finalize`.
- The since-v12 section attributes nine changes to their first release tags,
  each verified as an ancestor of the pin. The 17.1 fix `fee8cb94734` stops
  parent index comments propagating onto partition children; measured, a
  partitioned `ALTER COLUMN TYPE` now empties child baselines instead of
  silently overwriting them with the parent's, so the audit query detects it.
- All 127 source citations were machine-checked to resolve to existing v17 files
  with in-range line numbers; the BKI-generation citation was corrected from
  `src/backend/catalog/Makefile` to `src/include/catalog/Makefile`, where the
  `CATALOG_HEADERS` list and `bki-stamp` rule live in v17.
- Test objects were dropped and the isolated server was stopped. Updated
  `wiki/index.md`, `wiki/v17/index.md`, and `wiki/versions.md`. Human
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are as filed.

## [2026-08-06] follow-up v12 | rebuild with REINDEX INDEX CONCURRENTLY

- The user stated the maintenance process rebuilds with `REINDEX INDEX
  CONCURRENTLY`, so the decision-rule comparison in Detecting Bloat in All
  Index Types by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 12
  (unverified)
  was extended against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
- No score changes. On a second isolated 12.2 server with `autovacuum = off`,
  two byte-identical churned 200,000-row tables were rebuilt with RIC and with
  plain `REINDEX INDEX`; the resulting files matched block for block on all
  seven access methods (btree 551, spgist 1247, gin 4102, gist 1536, hash 824,
  bloom 394, brin 3), so every reclaimable fraction on the page applies to a
  concurrent rebuild.
- All seven stored baselines survived the swap while every index OID changed
  (16410 -> 16430 and so on), which `index_concurrently_swap()` explains. Added
  the operational consequence: the surviving comment is the *pre*-rebuild
  baseline, so the loop must overwrite it as its last step.
- Documented the cost profile that remains under RIC: `ShareUpdateExclusiveLock`
  instead of `ShareLock`/`AccessExclusiveLock`, peak storage of old plus new
  between phase 1's `_ccnew` copy and phase 6's drop, a lock that conflicts with
  lazy VACUUM's own `ShareUpdateExclusiveLock`, four `WaitForLockersMultiple`
  points, and the `PreventInTransactionBlock` ban.
- Measured four loop-breakers: a direct RIC on an exclusion-constraint index
  gives `ERROR: concurrent index creation for exclusion constraints is not
  supported`; `REINDEX TABLE CONCURRENTLY` skips it with only a `WARNING`, so
  its drift climbs forever; a temporary index silently takes the non-concurrent
  path with an unchanged OID; and system catalogs error. A
  `statement_timeout = '90ms'` interruption on a 3,000,000-row table left
  `f_k_ccnew` invalid, not-ready, live, 0 blocks and with no comment, which the
  page's baseline audit query reports as `no baseline stored`.
- Added six Evidence Map rows, two Context Reviewed bullets, five Open
  Questions, twelve Source References, and two rewritten recommendation
  bullets. Updated `wiki/index.md`, `wiki/v12/index.md`, and
  `wiki/versions.md`. No new `##`/`###` heading, so the Contents list is
  unchanged.
- All 16 new citations were re-resolved against the pinned checkout and every
  in-page anchor was re-checked. Test schemas were dropped and both isolated
  servers were stopped. Human `verified: false`, the visible `(unverified)`
  title, and `verified_by_agent: not yet` are unchanged.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-06] follow-up v12 | ratio drift versus "reindex when index > heap"

- Added a decision-rule comparison section to Detecting Bloat in All Index
  Types by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 12
  (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`
  (PostgreSQL 12.2).
- Prompt hygiene applied before drafting. The user chose corrected wording with
  no prompt note, the absolute reading `current ratio > 1.0` measured on main
  forks, full exact-pin measurements, and all seven access methods.
- Established the arithmetic identity that `index_bytes > heap_bytes` is
  equivalent to `drift > 1 / baseline_ratio`, so the proposed alternative is
  itself a drift rule whose per-index threshold nobody chose. Measured that
  threshold at 0.94 (gin) to 1282 (brin) on one table, a 1367x spread, and at
  0.70 to 4444 across three heap row widths.
- Built a fresh isolated 12.2 server from the pinned checkout with
  `autovacuum = off`. A day-zero sweep at 200,000 rows held every index's block
  count identical across heaps of 2,858, 3,847 and 13,334 blocks while the GIN
  ratio fell 1.435269 -> 1.066285 -> 0.307635, which grounds the ratio in
  `IndexTupleData` versus `HeapTupleHeaderData` layout. A fresh-B-tree sweep
  from 0 to 1,000,000 rows fell 2.000000 -> 0.266246, so a baseline captured at
  1,000 rows shows drift 0.586 at a million with no bloat.
- Scored both rules over a 49-cell matrix, seven workloads times seven access
  methods, with `REINDEX TABLE` ground truth and the label "a rebuild reclaims
  >= 25%": `drift >= 1.40` returned 13 true and 1 false positive against
  `index > heap`'s 6 and 2. The absolute rule flagged 6 of GIN's 7 cells, 1
  GiST, 1 SP-GiST and none of the other 28; four of its six true positives are
  GIN cells that were over the line at build time. Drift-threshold sweeps from
  1.10 to 1.50 score identically, and ground-truth sweeps at 10/25/40/50% leave
  the ranking unchanged.
- Reproduced the question's own premise: a GIN index tuned to baseline
  1.004900 that `index > heap` condemns at 0.0% reclaimable space the moment it
  is built and at every later step, while drift stayed quiet until a rebuild
  was worth 38.9%. First 1.40 crossings reclaimed 25.2% (spgist, 30% churn),
  37.4% (btree, 60%) and 38.9% (gin, 60%); GiST stalled at 1.399 with 35.4%
  reclaimable. Ground truth per step came from a second index of the same
  definition built on the live table and dropped.
- Filed one verified read-only comparison query that prints both rules plus the
  drift the absolute rule silently demands, exercised against an index carrying
  a human comment so the `CASE` guard is proven.
- Added five Evidence Map rows (index versus heap tuple layout, rebuilt-B-tree
  fillfactor 90 against heap 100, the empty-index metapage, and `reindex_index`
  locks), three Context Reviewed bullets, six Open Questions, one Contents
  entry, and eight Source References. Updated `wiki/index.md`,
  `wiki/v12/index.md`, and `wiki/versions.md`.
- Test schemas were dropped and the isolated server was stopped. Human
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are unchanged because this was a scoped
  follow-up, not a full page re-audit.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-06] review v12 | COMMENT-stored index/heap ratio bloat screen

- Re-reviewed Detecting Bloat in All Index Types by Storing an Index/Heap Size
  Ratio in COMMENT in PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Corrected the central interpretation: ratio drift is index allocation growth
  divided by heap allocation growth, not an access-method-neutral bloat measure.
  Distinguished the current-file rebuild-reclaimable fraction from excess bytes
  relative to the rebuilt file, and relabeled the fixture thresholds.
- Corrected source claims around ordinary VACUUM's conditional heap-tail
  truncation, BRIN's stepwise revmap extension, the SP-GiST redirect
  `RecentGlobalXmin` gate, GIN's cleanup trigger, indexed-key HOT eligibility,
  `pg_class.relpages` writers, COMMENT/catalog locks, default versus
  `--no-comments` dump/restore, `psql` visibility, and generated catalog headers.
- Added the `CREATE TABLE ... LIKE INCLUDING ALL` hazard: v12 clones index
  comments into new indexes, so a numeric baseline can silently describe the
  source heap. Updated capture discipline to reject ordinary VACUUM as proof of
  a compact heap and to forbid normalizing unexplained drift by re-capture.
- Replaced the baseline audit's unsafe cast-in-`OR` with a materialized,
  CASE-guarded parse. Reworked detection to measure each relation once and report
  zero-byte or unavailable relations explicitly rather than dropping them. An
  exact-pin 12.2 fixture confirmed the human-comment, zero-heap, zero-baseline,
  missing-baseline, and cloned-comment cases.
- Reran the 200k-row all-seven-AM cycle at one, two, and three post-delete VACUUM
  passes, repeating the one-pass arm. Corrected GIN from 4,102/7,495 blocks
  (+82.72%) to 4,101/7,482 (+82.44%); `gin_metapage_info` reports 600,000 entries
  on 4,100 entry pages. Confirmed B-tree +99.27%, GiST +89.47%, and SP-GiST
  +0.83%, including SP-GiST's retained nine blocks in every arm. Dropped the
  review schemas and the two extensions created for the tests; left the
  pre-existing repo-local server running.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Kept
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` because the historical non-cycle measurement
  tables were not all independently reproduced.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-06] follow-up v12 | delete-and-reload cycle test for the COMMENT-stored ratio

- Added a new `### A 200k-row delete-and-reload cycle test on all seven index
  types` section to Detecting Bloat in All Index Types by Storing an Index/Heap
  Size Ratio in COMMENT in PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Prompt hygiene applied before drafting. The user chose silently corrected
  wording ("postgreql" -> "PostgreSQL", "diferences" -> "differences",
  "add again 200k rows" -> "add 200k rows again", run-on steps punctuated), all
  seven access methods, fresh ascending reload keys `200001`-`400000`, and
  results plus a runnable script.
- Built a second isolated 12.2 server from the pinned checkout
  (`autovacuum = off`, private socket, port 54312) carrying one table with all
  seven access methods at 200,000 rows, and ran the user's exact sequence:
  load, build all seven indexes, capture the ratio into each comment, `VACUUM`,
  `DELETE` every row, `VACUUM`, reload 200,000 rows with fresh ascending keys,
  then `REINDEX TABLE` for ground truth.
- Headline result: the heap returned to exactly 3,847 blocks, so the heap growth
  factor is exactly 1.000 and `ratio_pct` equals raw `index_size_pct` digit for
  digit on all seven access methods, and equals `REINDEX` ground truth on six.
  B-tree +99.27%, GiST +89.47%, GIN +82.72%, SP-GiST +0.83%, and hash, BRIN and
  bloom at 0.00%.
- Ran the identical cycle at two and three post-`DELETE` `VACUUM` passes, plus an
  independent replicate of the one-pass arm that reproduced every byte. The arms
  split the access methods along the `RecentGlobalXmin` recyclability gate:
  B-tree and GiST returned to exactly 551 and 1,538 blocks with two passes
  (`_bt_page_recyclable`, `gistPageRecyclable`, and the key-order-agnostic
  `_bt_getbuf`/`GetFreeIndexPage` reuse path); SP-GiST, bloom, hash and BRIN
  never grew (no xid gate in `spgvacuumpage`/`blvacuumcleanup`, hash frees and
  reuses overflow pages through its own bitmap rather than the FSM, and
  `brinbulkdelete` is a documented no-op); GIN never recovered at any pass count
  because `ginVacuumEntryPage` retains entry tuples whose posting list empties.
- Supporting probes: `pgstatindex` showed the final 1,098-block B-tree is exactly
  `1 metapage + 3 internal + 547 leaf + 547 still-deleted`; `gin_metapage_info`
  showed 600,000 entries still on 4,101 entry pages after three `VACUUM` passes
  with a 0-byte FSM fork; `hash_metapage_info` resolved hash's 0.86% ground-truth
  gap to 7 overflow pages (53 versus 46) at an identical `maxbucket = 767`; and
  mid-cycle the zero-byte heap made the raw ratio `NULL` and the filed detection
  query return zero rows.
- Re-ran the published fixture script standalone on a fresh database; it
  reproduced every number in the filed results table. Test objects were dropped
  and the isolated server was stopped.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Kept
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet`, because this was a scoped addition rather than a
  full claim-by-claim page re-audit.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-06] question v12 | COMMENT-stored index/heap ratio as a bloat detector

- Filed Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio
  in COMMENT in PostgreSQL 12 (unverified)
  against unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Prompt hygiene applied before drafting: the user approved correcting
  "postgreql", the preposition, and the `COMMENT` capitalization, and chose a
  bare-ratio payload, source plus exact-pin tests, and read-only filed SQL. The
  correction is recorded as a prompt note under `## Question`.
- Source side: the full `COMMENT` path (grammar, `CommentObject`'s
  `ShareUpdateExclusiveLock` plus ownership check, `CreateComments`,
  `pg_description`, `obj_description`), comment lifecycle across DDL
  (`index_concurrently_swap` moving the `pg_description` row, `pg_dump`'s
  index comment, the `create_index` and `alter_table` regression suites),
  the size functions (`pg_relation_size` main-fork wrapper,
  `calculate_relation_size`, `calculate_table_size`,
  `calculate_toast_table_size`, `vac_update_relstats`), per-AM storage
  behavior (fill factors, `_hash_init`/`_hash_expandtable` splitpoints, BRIN
  pages-per-range and revmap capacity, the four index-vacuum FSM reclaim
  paths, and every `RelationTruncate`/`smgrtruncate` call site), and the
  contrib boundary (`pgstattuple`'s AM support matrix and the
  `superuser = true` control-file default).
- Corrected one mid-inquiry error: the SP-GiST `RelationTruncate` call found by
  grep is compiled out under `#ifdef NOT_USED`, so no v12 index AM shrinks its
  main fork during VACUUM.
- Built an isolated 12.2 server (VPATH build under `.wiki-runtime/`, port 5492,
  autovacuum off) and ran six experiments over seven access methods: a
  three-scale fresh-ratio invariance sweep (B-tree 0.9942, SP-GiST 0.9934,
  bloom 0.9909, GiST 1.0312, hash 0.7963, GIN 0.6361, BRIN 0.0625 over 16x
  growth), a seven-point hash/B-tree sawtooth sweep (healthy hash spans
  0.3314-0.4985 while B-tree moves 0.5%), six workload fixtures with `REINDEX`
  and `VACUUM FULL` ground truth, a nine-operation COMMENT durability matrix, a
  lock/ownership/read-visibility probe, and 300-index survey timings
  (5.3-7.1 ms size-based, 3.7-3.8 ms catalog-only, with `relpages` measured
  stale at 8 blocks against a live 281).
- Key measured results now on the page: `drift = index growth factor / heap
  growth factor`; a mass delete plus VACUUM leaves drift at exactly 1.000 on all
  seven AMs against 0.667-0.797 real index bloat; row widening masks 49.9%
  B-tree bloat as drift 0.312; heap truncation fakes drift 3.996 on a
  zero-bloat BRIN; a `pg_table_size` denominator collapses drift from 1.746 to
  0.095 under TOAST traffic; a single 20%-of-rows key update grows a
  fillfactor-90 B-tree by 1.995x; and `ALTER TABLE ... ALTER COLUMN TYPE` on a
  partitioned table overwrote both child baselines with the parent index's
  comment, matching the upstream test's own "wrong behavior for comments"
  note.
- Both filed queries (detection and baseline audit) were executed verbatim on
  the pinned server; both are read-only, tagged, and wrapped in transaction-
  scoped `statement_timeout`/`lock_timeout`.
- Test objects were dropped, the isolated server was stopped, and the build
  tree, data directory, and scratch SQL stay under `.wiki-runtime/tmp/`.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Human
  `verified` stays `false`, the visible title keeps `(unverified)`, and
  `verified_by_agent` stays `not yet`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-06] remove v17 | delete How a GIN Index Becomes Bloated

- Removed `wiki/v17/questions/indexing/gin-index-bloat.md` per user request.
  The v12 page of the same name was removed earlier the same day, so no version
  now carries a dedicated GIN-bloat page.
- Removed the active page entry from `wiki/index.md` and `wiki/v17/index.md`,
  and the `## Related Pages` entry in
  `wiki/v17/questions/query-planning/bloated-indexes-query-planner.md`.
- Reworded the one prose cross-reference on that same planner page so the
  non-B-tree-bloat paragraph points at the page's own
  `## Follow-Up: When A GIN Index Is Discarded And A B-Tree Is Used Instead`
  section instead of the deleted page. The paragraph's claims and its
  `maintenance.sgml` citation are unchanged.
- Dropped the GIN-bloat clause from the v17 coverage cell in `wiki/versions.md`
  and reopened the following sentence with "Filed coverage also includes", which
  the removed clause previously carried.
- Neutralized the remaining historical Markdown links in `wiki/log.md` (1) and
  `wiki/versions.md` (2) so they refer to the page title as plain text.
- v17 GIN coverage now runs through that follow-up section of [Planner Penalties
  for Bloated Indexes in PostgreSQL 17
  (unverified)](v17/questions/query-planning/bloated-indexes-query-planner.md),
  which prices GIN against B-tree on the same column, plus the contrib inventory
  in [PostgreSQL 17 Contrib Extensions
  (unverified)](v17/questions/server-administration/contrib-extensions.md).
- No source citation, pin, or verification field on any surviving page changed.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-06] remove | delete every pgstatindex sampling question page

- Removed all three surviving `pgstatindex` sampling question pages per user
  request, confirmed page by page before deleting:
  `wiki/v17/questions/indexing/pgstatindex-sample-variant-proposal.md`,
  `wiki/v18/questions/indexing/pgstatindex-sample-variant-proposal.md`, and
  `wiki/v18/questions/indexing/pgstatindex-approx-sampling.md`. The third page
  is an explanation rather than a proposal, but its filed question asks why
  `pgstatindex` cannot sample the index like `pgstattuple_approx`, so the user
  included it.
- Removed the active page entries from `wiki/index.md`, `wiki/v17/index.md`,
  and `wiki/v18/index.md`, the inbound `## Navigation` link in
  `wiki/v17/questions/indexing/gin-index-bloat.md`, and the inbound
  `## Related Pages` link in
  `wiki/v17/questions/query-planning/bloated-indexes-query-planner.md`.
- Stripped the sampling clauses from the v17 and v18 coverage cells in
  `wiki/versions.md` at the user's request, so the coverage summary no longer
  advertises the removed pages.
- Neutralized the remaining historical Markdown links in `wiki/log.md` (9) and
  `wiki/versions.md` (1) so they refer to the page titles as plain text.
- No sampling `pgstatindex` page remains in any version; the v12 pages were
  removed earlier the same day. Adjacent coverage is untouched: v18 keeps
  [Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index
  (unverified)](v18/questions/indexing/avg-leaf-density-during-vacuum.md), and
  v12 keeps [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12
  (unverified)](v12/questions/indexing/how-pgstatindex-calculates-information.md)
  and [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12
  (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md).
- No source citation, pin, or verification field on any surviving page changed.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-06] remove v12 | delete Proposing a Sampling pgstatindex Variant

- Removed `wiki/v12/questions/indexing/pgstatindex-sample-variant-proposal.md`
  per user request.
- Removed the active page entry from `wiki/index.md` and `wiki/v12/index.md`,
  the sampling-proposal clause from the v12 coverage cell in
  `wiki/versions.md`, and the one inbound `## Navigation` link in
  `how-pgstatindex-calculates-information.md`.
- Neutralized the remaining historical Markdown links in `wiki/log.md` (10) and
  `wiki/versions.md` (4) so they refer to the page title as plain text.
- Left the v17 and v18 sampling `pgstatindex` pages and their inbound links
  untouched. v12 keeps the exact full-scan explanation in
  `how-pgstatindex-calculates-information.md` and the contrib-free estimate in
  `btree-index-bloat-core-sql-only.md`.
- No source citation, pin, or verification field on any surviving page changed.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-06] remove v12 | delete Proposing a Sampling pgstatginindex Variant

- Removed `wiki/v12/questions/indexing/pgstatginindex-sample-variant-proposal.md`
  per user request.
- Removed the active page entry from `wiki/index.md` and `wiki/v12/index.md`,
  and the GIN sampling-proposal sentence from the v12 coverage cell in
  `wiki/versions.md`. No surviving wiki page linked to the deleted page, so no
  `## Navigation` section changed.
- Neutralized the remaining historical Markdown links in `wiki/log.md` and
  `wiki/versions.md` so they refer to the page title as plain text.
- Left the v12 B-tree page
  `wiki/v12/questions/indexing/pgstatindex-sample-variant-proposal.md` and the
  v17/v18 sampling `pgstatindex` pages untouched.
- No source citation, pin, or verification field on any surviving page changed.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-06] remove v12 | delete How a GIN Index Becomes Bloated

- Removed `wiki/v12/questions/indexing/gin-index-bloat.md` per user request.
- Removed the active page entry from `wiki/index.md` and `wiki/v12/index.md`,
  the GIN-bloat sentence from the v12 coverage cell in `wiki/versions.md`, and
  the navigation links in `btree-index-bloat-core-sql-only.md`,
  `pgstatginindex-sample-variant-proposal.md`, and
  `bloated-indexes-query-planner.md`.
- Neutralized the remaining historical Markdown links in `wiki/log.md` and
  `wiki/versions.md` so they refer to the page title as plain text.
- Left the separate v17 page `wiki/v17/questions/indexing/gin-index-bloat.md`
  and its inbound links untouched.
- No source citation, pin, or verification field on any surviving page changed.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-06] restructure | categorize all question documents

- Added `MANDATORY Question Categories` to `AGENTS.md`. Every `type: question`
  page now lives at `wiki/vNN/questions/<category>/<page>.md`. Filing directly
  under `wiki/vNN/questions/` is misfiled; there is no uncategorized location.
- The category set is closed at six subsystem-aligned directories, identical in
  every version: `query-planning`, `indexing`, `storage-and-vacuum`,
  `replication-and-wal`, `observability`, and `server-administration`. The rule
  carries a scope table, three ordered tie-breaks (subject over observing tool,
  then citation majority, then table order), one-page-one-category, and the
  repo-wide procedure required to add, rename, or remove a category.
- Moved all 54 existing question pages with `git mv`: indexing 23,
  query-planning 9, observability 7, server-administration 7,
  storage-and-vacuum 5, replication-and-wal 3. The same basename takes the same
  category in every version, so cross-version comparison is a directory diff.
- Rewrote 14,861 page-relative links by resolving each URL against the page's
  old directory and re-emitting it against the new one. Question-page source
  citations moved from `../../../raw/postgres-NN/...` to
  `../../../../raw/postgres-NN/...`; page-to-page navigation, cross-version
  question links, and inbound links from `wiki/index.md`, `wiki/versions.md`,
  and `wiki/log.md` were rewritten by the same rule. No citation target or line
  range changed.
- Fixed 185 line-wrapped Markdown links in `wiki/log.md` (124) and
  `wiki/versions.md` (61) that `scripts/wiki_lint` cannot see: its `MD_LINK_RE`
  forbids newlines in link text, so a link whose label wraps across lines is
  excluded from link validation. Such links still render in VS Code, so the
  path change left them broken until this pass.
- Corrected one citation in [How the REPACK Command Works in PostgreSQL 19, and
  Its 50 Feature-Scope Commits
  (unverified)](v19/questions/storage-and-vacuum/repack-command.md) that the
  same regex had hidden: an inline code span containing `[` swallowed the
  following link. Only the `../` depth changed; the target file and the
  `#L372-L463` range are unchanged.
- Regrouped `wiki/index.md` and all five `wiki/vNN/index.md` question lists
  under category headings in the canonical category order, `####` at the root
  index and `###` on landing pages. Bullet text is unchanged. Normalized the
  v12, v14, and v17 landing pages onto the `## Coverage` plus `## Questions`
  shape that v18 and v19 already used.
- Updated the `AGENTS.md` sections that named the old flat path: the citation
  depth prose and examples, `MANDATORY Question Documents`, `MANDATORY Wiki
  Structure`, and step 7 of `MANDATORY Answer And File`.
- No page prose, claim, citation target, `pinned_commit`, `verified`, or
  `verified_by_agent` value changed. `scripts/wiki_lint`: 0 errors, 0 warnings.
  An independent newline-tolerant pass resolves 15,930 relative links with 4
  pre-existing exceptions: one `raw/postgres-NN` placeholder inside a code span
  and three historical `../answers/plan_cache_mode.md` references to the answer
  page deleted on 2026-06-06.

## [2026-08-06] remove v12 | delete A Heuristic to Detect B-Tree Index Bloat

- Removed `wiki/v12/questions/index-bloat-heuristic.md` per user request.
- Removed the active page entry from `wiki/index.md` and `wiki/v12/index.md`,
  and the navigation links in `btree-index-bloat-core-sql-only.md`,
  `gin-index-bloat.md`, `leaf-density-60-vs-90-query-impact.md`, and
  `leaf-density-vs-fragmentation-index-scan-io.md`.
- Reworded the one prose cross-reference in
  `btree-index-bloat-core-sql-only.md` (`### Follow-up: an avg_leaf_density
  predictor head to head`) so its `pgstatindex` density-versus-fillfactor
  comparison no longer points at the deleted page; the section's own claims and
  citations are unchanged.
- Neutralized the remaining historical Markdown links in `wiki/log.md` and
  `wiki/versions.md` so they refer to the page title as plain text.
- No source citation, pin, or verification field on any surviving page changed.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-06] follow-up v12 | core-SQL bloat results versus pgstatindex

- Answered a filed follow-up on [Measuring B-Tree Index Bloat With Core SQL Only
  in PostgreSQL 12
  (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
- Prompt hygiene applied first. The user approved the corrected follow-up, "Add
  sections comparing the SQL bloat results to pgstatindex results, and measure
  what the error is with different types of index bloat and with partial and
  non-partial indexes.", and chose the same page, bloat-relevant `pgstatindex`
  columns only, and a wide matrix with repeats.
- Built a second isolated 12.2 server from the same pin with a 9-bloat-type x
  3-scale x partial/non-partial matrix: 54 indexes over 27 tables, each table
  carrying a non-partial and a partial index over the same key so both see
  identical churn. The predicate is `id % 5 = 0` and every delete pattern uses
  modulus 7 or 11, coprime with 5, so the partial index loses the same
  proportion of entries rather than all or none. Bloat types: fresh, scattered
  delete, contiguous range delete, random-order insert, all-duplicate keys,
  vacuumed churn, unvacuumed churn, `LP_DEAD`-killed entries, and stale
  catalogs. Scales 200k/500k/1000k rows are the repeat dimension.
- Method A was run from the page's own CTE chain, with only the 1 MB triage
  filter removed, so the measured error applies to the filed SQL. Against the
  Method C rebuild it was exact in 39 of 54 cells, within 5 blocks in 47, within
  0.8 percentage points in 48, and wrong on 6.
- Method B reproduced `pgstatindex.leaf_pages` exactly in all 36 cells that pass
  the `Heap Fetches: 0` precondition, partial and non-partial alike, and
  `avg_leaf_density` to within 0.04-0.05 points (0.14-0.15 on the all-duplicate
  cells, where the pivot absorbs a heap TID). `blocks - leaf_est` matched
  `internal + deleted + half-dead + metapage` exactly in all 36, and subtracting
  the modelled internal pages recovered 2330 deleted pages within one page. The
  18 ineligible cells are exactly the three never-vacuumed fixtures, and the
  precondition caught every one.
- Added a head-to-head against an `avg_leaf_density` rebuild-size predictor,
  `ceil(leaf_pages * avg_leaf_density / fillfactor) + internal_pages + 1`. The
  core-SQL model wins or ties in 16 of 18 type/kind cells. On the `LP_DEAD`
  fixture the density predictor is 994.4% wrong because `pgstatindex` reports a
  healthy-looking 90.06 density on a 2745-block index that rebuilds to 251
  blocks: `PageGetFreeSpace` counts dead and killed entries as used, so physical
  occupancy is not reclaimable space. Recorded explicitly that this scores
  `pgstatindex` as a predictor it never claimed to be, and listed what it still
  uniquely provides (deleted-versus-half-dead split, `leaf_fragmentation`,
  `tree_level`, no dependence on the visibility map or a live row count).
- Isolated the single large model failure: partial indexes on tables deleted
  without VACUUM or ANALYZE, 510 blocks and 92.6 points off, reporting 0.0% to
  −2.0% bloat against a true 89.3-90.6%, because `n_live_tup` counts the whole
  table and cannot stand in for a partial index's `reltuples`. The non-partial
  index on the same table was exact in every one of those cells. A plain
  `ANALYZE` cut the worst error from 510 blocks to 1; the residual is ANALYZE's
  sampled `tupleFract * totalrows` recording 18,337 index rows against a true
  18,181.
- Added five `### Follow-up:` sections, refreshed the table of contents, added
  the follow-up prompt under `## Question`, and recorded four new open
  questions: half-dead pages were never produced, the density-predictor formula
  is this page's construction, the matrix varies scale rather than seed, and the
  post-`ANALYZE` one-block residual was not chased.
- All 97 citations were machine-audited against the pinned source with 0 broken
  ranges. Test objects were dropped, the isolated server was stopped, and
  `raw/postgres-12/` was left unchanged.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Human
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are unchanged because this was a scoped
  follow-up.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-06] answer v12 | core-SQL-only B-tree index bloat measurement

- Filed [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12
  (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
- Prompt hygiene applied first. The user chose the corrected wording, "Propose a
  SQL-only method, using no contrib extensions, to measure B-tree index bloat in
  PostgreSQL 12.", a new question page rather than a follow-up on the existing
  `pgstatindex` heuristic page, source plus exact-pin tests, and permission to
  include write/DDL rebuild probes.
- Established the negative result first: core v12 has no SQL-callable page
  reader, FSM reader, or per-page fill function, and `pgstattuple`,
  `pageinspect`, `pg_freespacemap`, and `amcheck` all require superuser to
  install because their control files omit `superuser =` and
  `read_extension_control_file` defaults it to true.
- Derived the answer from v12's own build rule rather than from a folklore
  formula: `_bt_pagestate` sets the leaf threshold to
  `BLCKSZ * (100 - fillfactor) / 100` (70 for internal levels), `_bt_blnewpage`
  pre-reserves the high-key line pointer, `PageGetFreeSpace` already subtracts
  one line pointer, and `_bt_buildadd` starts a new page once free space falls
  below the threshold, so
  `tuples_per_leaf = floor((BLCKSZ - 48 - floor(BLCKSZ * (100 - ff) / 100)) / (MAXALIGN(hoff + data) + 4))`,
  with `hoff` 8 or 16 from `IndexInfoFindDataOffset` and `data` from
  `heap_compute_data_size` (NULL columns contribute nothing, short varlenas take
  no alignment padding).
- Ranked four core-only methods and measured each against an actual
  `CREATE INDEX CONCURRENTLY` rebuild on one isolated 12.2 server with 15
  fixtures. The 26.7 ms catalog-only sweep matched the rebuilt block count
  exactly on 10 of 14 indexes (`idx_seq` 2745, `idx_del` 276, `idx_range` 414,
  `idx_churn` 825, `idx_rand` 2745), within 2 blocks on 3 more, and by −4.6% on
  a 2-to-81-character text key. A `pg_column_size` slot measurement (57.000
  bytes full scan, 56.893 from a 1% `BERNOULLI` sample, against the
  catalog-derived 60) corrected that case to +0.32%.
- The `EXPLAIN (ANALYZE, BUFFERS)` index-only-scan census reproduced
  `pgstatindex.leaf_pages` exactly on all 12 fixtures with `Heap Fetches: 0`
  using `full_scan_blocks - descent_blocks` (2736 − 3 = 2733 on the control),
  matched `internal + deleted + half-dead + metapage` exactly as well, and
  rebuilt `avg_leaf_density` to within 0.03-0.14 points on 11 of them with no
  contrib installed.
- Recorded six measured failure modes: stale `pg_class.reltuples` reporting
  0.0% bloat on an index whose rebuild goes 2745 → 276 blocks until
  `pg_stat_all_tables.n_live_tup` is substituted; an all-duplicate index that a
  rebuild *grows* from 1291 to 1376 blocks (`BTREE_SINGLEVAL_FILLFACTOR` 96
  versus a sorted build's 90), predicted correctly as −6.4% negative bloat; a
  random-insert index permanently at 27.0% from 50/50 non-rightmost splits; a
  range-deleted index reading a healthy 89.83 `avg_leaf_density` while 2330 of
  2745 blocks are deleted pages; the variable-width MAXALIGN-of-an-average bias;
  and an unvacuumed index whose 300,000 heap fetches inflated the census from
  3279 to 8452 leaf pages.
- Also documented `VACUUM VERBOSE`'s exact per-index page census with its
  `lazy_cleanup_index` message text, the observed gap where a no-op VACUUM
  prints no index line because `btvacuumcleanup` skips the scan, and
  `pg_relation_size(idx,'fsm') > 0` as a free deleted-page flag that was
  non-zero for exactly the one fixture with deleted pages (24576 bytes) and zero
  for the other 13.
- All 95 Markdown citations were machine-audited against the pinned source with
  0 broken ranges and label-checked, and all five filed SQL blocks were executed
  verbatim on the pinned build with exit status 0, reproducing the recorded
  numbers. Test objects were dropped, the isolated server was stopped, and
  `raw/postgres-12/` was left unchanged.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Human
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are unchanged pending a claim-by-claim review.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-04] follow-up v12 | pgstatginindex sampling bloat and wasted space

- Answered a filed follow-up on Proposing a Sampling pgstatginindex Variant for
  PostgreSQL 12 (unverified) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
- Prompt hygiene applied first. The user approved the corrected follow-up, "Can
  this function measure bloat? Can it measure wasted space?", and chose source
  plus exact-pin tests and concrete new wasted-space output fields rather than a
  diagnostic-only answer.
- Established the central distinction from source: the function can measure free
  space but not wasted space. GIN has no `fillfactor` (`ginoptions` parses only
  `fastupdate` and `gin_pending_list_limit`), and `entrySplitPage` equalizes
  entry pages by total data size with no rightmost or sorted-insert case, so
  roughly half free is the structural steady state. `leafRepackItems` packs
  posting-tree leaves tight only when `btree->isBuild`, which is why a low
  data-leaf density is a real signal and a ~50% entry-leaf density is not.
- Proposed two exact fields, `pending_space` and `total_space`, and four
  estimates, `approx_free_space`, `approx_free_percent`,
  `approx_recyclable_pages` and `approx_recyclable_space`, on the
  `pgstathashindex` model of counting whole unused pages as free space and
  excluding bookkeeping pages from the denominator; `total_space` is exact
  because `nPendingPages` is. Recyclability comes from the delete XID GIN keeps
  in the page header's `pd_prune_xid`, with `GetOldestXmin`/`RecentGlobalXmin`
  precedents in `pgstatapprox.c` and `contrib/amcheck`.
- Recorded the limits with measurements, not hedges. Deleting 360,000 of 400,000
  rows without VACUUM moved not one output field, while `pgstattuple` reported
  75.13% dead heap tuples; a full scan would be equally blind because GIN pages
  carry no `LP_DEAD` line pointers, which is why `pgstathashindex` can report
  `dead_items` and a GIN function cannot. `ginVacuumEntryPage` rebuilds an
  emptied entry tuple with `nitems = 0` instead of deleting it, so 4,000 key
  tuples survived VACUUM where a fresh build has 2,000, and the fork never
  shrank. A 246-page pending list at 99.6% density was 30.98% of one index yet
  left `free_percent` indistinguishable from a healthy index's.
- Exact-pin measurements reused the seven existing fixtures plus two new ones on
  the same isolated 12.2 server. Four healthy, never-deleted indexes measured
  49.54-49.66% free, and a sibling index built over identical rows was
  byte-identical to the original (412 blocks, 50.46% density, the same 1,688,200
  used bytes), so a rebuild returns zero bytes against a reported 49.66% free.
  The census's recyclability verdict matched VACUUM's own `pages_free` and
  `pg_freespace` exactly on all seven fixtures (465, 168 and 40 blocks at avail
  8160, zero elsewhere), and the two deletion paths were distinguishable on disk
  (`prune_xid = 0` for 393 drained pending pages against 522 and 518 for posting
  pages). One `f_waste` case reproduced VACUUM's own "0 are currently reusable"
  followed by "40 are currently reusable" on a second run.
- 1,800 further seeded runs plus nine full-sample checks established full-sample
  equivalence on every new field, `free_percent` within 0.25 points of truth
  from 137 of 13,672 pages, a worst case of 9.72 points on a 308-page index, and
  a 60% alarm threshold classifying bloated versus healthy correctly in 800 of
  800 runs inside a measured 51.22-to-62.02 gap, with a two-block empty index as
  the one false positive. Four sibling-index rebuilds priced the fields
  honestly: of the bytes raw free space reported, only 97.7%, 91.4%, 70.0% and
  0.0% materialized as recovered bytes, while subtracting the structural
  baseline predicted 96.2-98.5% of them.
- Added eight new `## Open Questions`, including that `bufpage.h` calls
  `pd_prune_xid` "currently unused in index pages" while GIN stores its delete
  XID there, that no fixture mixed fresh and aged delete XIDs, that the 60%
  threshold is fitted rather than derived, and that the baseline correction
  needs a baseline.
- All 292 citations on the page were machine-audited against the pinned source
  with 0 errors across 55 files, and the filed census prototype ran verbatim and
  reproduced the recorded numbers. New test objects (`f_waste`, `f_keys` and the
  four sibling indexes) were dropped, the original seven fixtures are unchanged,
  and the isolated server is stopped.
- Refreshed `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Human
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are unchanged because this was a scoped
  follow-up, not a fresh claim-by-claim audit of the full page.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-04] answer v12 | sampling pgstatginindex variant for GIN indexes

- Filed Proposing a Sampling pgstatginindex Variant for PostgreSQL 12
  (unverified) against
  unchanged pin `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2),
  derived from the existing v12 B-tree sampling proposal but scoped to GIN only.
- Prompt hygiene applied first. The user chose corrected wording, a GIN-native
  page-class output contract, exact-pin empirical tests in addition to source,
  and a GIN-specific sample policy instead of carrying over the B-tree page's
  100 MiB / 10% floor.
- Established from source that v12 has no physical GIN page accounting at all:
  `pgstattuple()` rejects GIN with `"%s" (gin index) is not supported`,
  `pgstatindex`/`pgstathashindex` reject it by access method, `pgstatginindex`
  reads only block 0 for three columns, and every `pageinspect` GIN function
  takes one caller-supplied raw page. Documented the complete v12 flag taxonomy
  from `ginblock.h` with the ordering trap that `GinPageSetDeleted` ORs
  `GIN_DELETED` onto a posting page's existing `GIN_DATA`/`GIN_LEAF` bits while
  `shiftList` overwrites a drained pending page's flags to exactly
  `GIN_DELETED`; the three distinct per-class capacity denominators (8160 for
  entry and list pages, 8152 for data pages after the right-bound reservation);
  and the data-page traps (`pd_lower` is a byte count so
  `PageGetMaxOffsetNumber` is invalid, `maxoff` forced to
  `InvalidOffsetNumber` on compressed leaves, untrustworthy `pd_lower` on
  pre-9.4 uncompressed leaves). Traced `ginvacuumcleanup`'s three-way
  classification, `ginUpdateStats`, `ginGetStats`, `GinPageIsRecyclable`,
  `GinNewBuffer`, the absence of any `RelationTruncate` under
  `src/backend/access/gin/`, and `gincostestimate`'s 4X scaling cutoff.
- Designed three GIN-specific sampling rules: R1 an exact pending stratum, since
  `nPendingPages`/`nPendingHeapTuples` are maintained in real time rather than
  at VACUUM; R2 a post-stratified expansion `n_np / k_np` over the
  exactly-known non-pending stratum; R3 a metapage-derived floor
  `ceil(target * nTotalPages / nDataPages)` with a `max_sample_pages` cap.
  Recorded that no other v12 index AM has an exactly-known page-class count in
  its metapage, and that nothing in v12 samples an index — the only
  `BlockSampler` call site is the table-AM-bound `acquire_sample_rows`, and
  `IndexAmRoutine` has no sample callback.
- Built the pinned 12.2 source out of tree under `.wiki-runtime/`, ran one
  isolated server with `pgstattuple` 1.5 and `pageinspect` 1.7, and built seven
  GIN fixtures spanning a 106.82 MiB entry-only control, a mixed index with all
  six classes at once, a 246-page pending list, a 41.1%-deleted posting-tree
  index, a small healthy index, a partial index, and an empty index. Confirmed
  the derived capacity constants against live pages (`pagesize` 8192,
  `special` 8184). Forcing posting trees required 8 distinct keys over 900,000
  rows; a first attempt with 1,500 keys produced entry pages only.
- Ran 6,100 seeded prototype runs (5,600 in the main grid, 200 for the floor
  comparison, 300 for the derived floor). Full-sample equivalence held on all seven
  fixtures. Post-stratification cut the `entry_leaf_pages` standard deviation
  from 46.6 to 6.8 at a 31% pending share (mean 544.0 against a truth of 544,
  versus a biased 537.1) and from 227.8 to 149.0 at 4.5%. Entry-leaf density at
  1% coverage reproduced 50.44% with 0.00 maximum error over 100 seeds, while a
  one-page class returned `NaN` in 88 of 100 runs.
- Recorded an objective negative result rather than defending the design: the
  page's own first proposal, a flat 50-page absolute floor, was worse than the
  B-tree page's size-based rule. On the 68.57 MiB mixed fixture an 88-page
  sample missed the 12-page posting-tree-internal class in 87 of 100 runs and
  produced 36 `NaN` data densities, where the B-tree 10% rule (878 pages)
  missed 32 and produced none. The floor was therefore replaced by the
  metapage-derived one, which reached 1,727 pages (18.52% of the index against a
  requested 1%) and still missed the class in 6 of 100 runs.
- Quantified four separate ways the metapage misreports GIN structure:
  `nDataPages = 180` conflating 108 data pages with 72 deleted ones; a second
  VACUUM moving 168 deleted pages out of every metapage class so 41.1% of one
  index's blocks became invisible; pending pages counted in no class
  (`nEntryPages = 547` against 793 ordinary blocks); and `nTotalPages` reading
  548 against 794 live blocks. Also measured a VACUUM that flushed a 393-page
  pending list *growing* the index from 8,777 to 9,325 blocks and leaving 465
  deleted pages.
- Verified the exact pending-list walk (393/40,000 and 246/50,000 matching the
  metapage exactly at `nPendingPages + 1` reads) and all prototype error paths,
  including that a B-tree index is rejected only accidentally, by
  `gin_metapage_info`'s `flags == GIN_META` check, which is why the proposed C
  function must test `relam` itself.
- All 219 source citations were machine-audited against the pinned checkout with
  `.wiki-runtime/audit_gin12_sample.py`; 0 errors across 48 distinct files, and
  ten range boundaries were tightened after review. Human `verified: false`, the
  visible `(unverified)` title, and `verified_by_agent: not yet` are unchanged
  because no claim-by-claim full-page re-verification was performed.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`.
  `scripts/wiki_lint` run recorded below.

## [2026-08-04] follow-up v17 | GIN discarded in favour of a B-tree

- Answered a filed follow-up on [Planner Penalties for Bloated Indexes in
  PostgreSQL 17 (unverified)](v17/questions/query-planning/bloated-indexes-query-planner.md)
  against the unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa`
  (PostgreSQL 17.10). Prompt hygiene was applied before drafting: the user's
  instruction misspelled "PostgreSQL" as "postgreql", and the user chose the
  corrected spelling, source plus exact-pin tests, and a `## Follow-Up` section
  on the existing v17 page rather than a new standalone page. The follow-up
  question itself, "When might a GIN index be discarded by the query planner and
  a B-tree used instead?", is restated verbatim under `## Question`.
- Established from source that v17 discards a GIN index at three separate gates,
  only the last of which is cost. Gate 1 is clause matching: `create_index_paths`
  runs `match_restriction_clauses_to_index` before any cost model, and
  `match_opclause_to_indexcol` requires a collation match plus
  `op_in_opfamily()`, with `get_index_clause_from_support()` as the only escape
  hatch; the four core `pg_amop.dat` GIN families declare no `<`/`<=`/`>=`/`>`,
  and `contrib/btree_gin` closes the gate deliberately while documenting that it
  will not outperform B-tree. Gate 2 is plan shape from the AM flags
  `ginhandler` sets (`amgettuple = NULL`, `amcanorder`/`amcanorderbyop` false,
  `amcanreturn = NULL`, `amsearchnulls`/`amsearcharray`/`amcanparallel` false),
  traced through `get_index_paths`, `build_index_paths`, `check_index_only`,
  `index_can_return`, and the `NullTest` branch of `match_clause_to_indexcol`.
  Gate 3 is cost: `gincostestimate()` seeds its startup page count with the
  entire pending list and charges every pending, entry and data page at both
  `random_page_cost` and `DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost`, with
  no tree-height charge, after which `add_path()` fuzzy dominance at
  `STD_FUZZ_FACTOR` and `choose_bitmap_and()` drop the loser.
- Also documented the 4X stale-metapage-statistics fallback and the trap that
  `gin_clean_pending_list()` drains the list without refreshing `nTotalPages`,
  the keyless full-index path a partial GIN index can still yield through
  `amoptionalkey` plus the `indexQuals == NIL` branch, the four
  `CREATE INDEX`/`CLUSTER` AM rejections, and where GIN still wins.
- Corrected one claim rather than carrying it over from the v12 answer: in v17 a
  bare boolean `Var` *does* match a `btree_gin` bool opclass, because
  `IsBooleanOpfamily()` falls back to
  `op_in_opfamily(BooleanEqualOperator, ...)` for non-built-in opfamilies and
  `match_boolean_index_clause()` rewrites the clause to `indexkey = true`. The
  in-tree `bool.out` expected output and an exact-pin run both show
  `Index Cond: (i = true)`.
- Exact-pin measurements on one isolated 17.10 server, built from the pin with
  `btree_gin`, `pgstattuple` and `pageinspect`: the same `n = 42` predicate cost
  `12.97` through a `btree_gin` GIN index and `4.52` through a B-tree on a
  single table with literally identical statistics, obtained by dropping the
  other index inside a rolled-back transaction, even though the GIN index was
  279 blocks against the B-tree's 280. Both closed forms were reproduced in SQL
  from the catalog and the GIN metapage and matched to the cent (`12.9725` and
  `4.5225`). B-tree also won `BETWEEN` (`67.15` vs `44.31`), `n < 20`
  (`28.63` vs `8.89`) and `IN (1,2,3)` (`30.10` vs `13.57`), while
  `ORDER BY … LIMIT 10`, an index-only scan and `IS NULL` produced no GIN path
  at all, visible as `disable_cost`-priced sequential scans.
- A 982-page `fastupdate` pending list moved the GIN scan from `13.80` to
  `4187.83`, and the planner dropped the index from the `BitmapAnd` and demoted
  `tsv @@ …` to a `Filter`; `gin_clean_pending_list()` then returned exactly
  `982`, matching `pgstatginindex`. A two-`random_page_cost` probe recovered the
  charged page count exactly in five separate states: `3.00`, `3.00`, `985.00`
  (982 pending + 2 entry + 1 data page, under the invented-statistics branch
  because `1306 > 324 * 4`), `4.00`, and `1001.00` on a 10-block keyless partial
  index. All four AM rejection messages and the live `amutils` property matrix
  were reproduced, and a multicolumn `gin (tsv, cat)` index beat the `BitmapAnd`
  at `21.55` versus `183.18`, confirming the `btree_gin` documentation's own
  claim.
- Two filed diagnostic SQL blocks were executed verbatim against objects
  literally named `my_table`, `my_col` and `my_gin_index`, reporting
  `pending_pct_of_index = 72.57` and `303.00` charged pages of which 246 were
  pending. Both carry `/* wiki_... */` tags and session-scoped
  `statement_timeout` / `lock_timeout`.
- Recorded the explicit absence of any upstream GIN-versus-B-tree plan
  comparison and of any `gincostestimate()` test coverage, and added five new
  `## Open Questions`, including a `BitmapAnd` outcome that was not stable across
  two runs of the same fixture family.
- Test objects were dropped, the isolated server was stopped, and its data
  directory was removed; `raw/postgres-17/` was untouched. A machine audit
  checked all 142 citations on the page against the pinned source and fixed
  fourteen line ranges whose boundaries started or ended outside the intended
  symbol.
- Updated `wiki/index.md`, `wiki/v17/index.md`, and the `wiki/versions.md` v17
  row plus a new coverage note. `verified_by_agent` stays `not yet` because this
  was a scoped follow-up rather than a fresh full-page claim audit; human
  `verified: false` and the visible `(unverified)` title are unchanged.
  `scripts/wiki_lint` reports 0 errors and 0 warnings.

## [2026-08-04] answer v17 | planner penalties for bloated indexes

- Filed [Planner Penalties for Bloated Indexes in PostgreSQL 17
  (unverified)](v17/questions/query-planning/bloated-indexes-query-planner.md) against the
  unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10).
  Prompt hygiene was applied before drafting: the user chose the corrected
  wording ("In PostgreSQL 17, are there mechanisms to penalize bloated indexes
  in the query planner? ... and what changed since PostgreSQL 12."), asked for
  exact-pin empirical tests in addition to source, and asked for a new
  standalone v17 page cross-linked to the existing v12 page rather than a diff
  against it.
- Established from source that v17 has exactly four bloat-sensitive planner
  inputs, all filled by `get_relation_info()`: `IndexOptInfo.pages` (a live
  `RelationGetNumberOfBlocks()` answer for a non-partial index, or
  `estimate_rel_size()`'s `pg_class` density for a partial one),
  `IndexOptInfo.tree_height` from `_bt_getrootheight()`'s `btm_fastlevel`, the
  `index_pages` argument threaded into `index_pages_fetched()` and
  `compute_parallel_worker()`, and the v17-only descent clamp
  `num_sa_scans = Min(num_sa_scans, ceil(index->pages * 0.3333333))`. Documented
  that `pgstatindex`'s `avg_leaf_density` and `leaf_fragmentation` are computed
  from live leaf pages only and are read by nothing on the cost path, that
  `pg_class` has no density or fragmentation column, and that only B-tree, hash,
  GiST and SP-GiST route through `genericcostestimate()` while GIN and BRIN do
  not.
- Catalogued eight bloat shapes with per-query-shape sensitivity: low-density
  live leaf pages, deleted/half-dead tombstone pages, extra tree levels,
  fragmented leaf chains, non-HOT version churn, deduplication disabled,
  VACUUM's 2% `BYPASS_THRESHOLD_PAGES` bypass and wraparound failsafe, and
  non-B-tree bloat.
- Built the since-v12 section from the checkout's own history. `git log -L` on
  `genericcostestimate`, `btcostestimate`, `get_relation_info`,
  `index_pages_fetched` and `cost_index` bounded by `REL_12_0..HEAD`, plus
  full-function diffs against `REL_12_0`, showed `index_pages_fetched()` and
  `cost_index()` are byte-identical and that only five commits touched
  `btcostestimate` (three of them cosmetic). Attributed each real change to its
  first release tag via `git tag --contains`: v13 `0d861bbb702` deduplication;
  v14 `d168b666823` bottom-up index deletion, `9dd963ae253` same-VACUUM page
  recycling, `5100010ee4d` index-vacuum bypass, `1e55e7d1755` wraparound
  failsafe, `3499df0dee8` `INDEX_CLEANUP` auto, `3d351d916b2` `reltuples = -1`;
  v16 `eb5c4e953bb` `DEFAULT_PAGE_CPU_MULTIPLIER`, `cd9479af2af` GIN page CPU
  charges, `3c569049b7b` partitioned-index zeroing; v17 `5bf748b86bc` SAOP
  descent clamp and `9391f71523b` `estimate_array_length()` statistics. The
  in-tree docs and nbtree README supplied their own version boundaries ("Prior
  to PostgreSQL 14, the only category of B-Tree deletion was simple deletion";
  "PostgreSQL 14 added the ability for VACUUM to consider if it's possible to
  recycle newly deleted pages").
- Measured everything on one isolated server built from the exact pin
  (`PostgreSQL 17.10`, autovacuum off), using `pgstattuple`, `pageinspect`
  `bt_metap()` for the planner's fast-root height, and `pg_visibility`. Key
  results: a closed-form cost prediction from `(pages, tuples, fastlevel)`
  reproduced `EXPLAIN` to the cent (`123144.43` exact); a point lookup priced at
  exactly `4.44` on both a 2,745-block/90.06%-density index and its
  26,411-block/9.62% twin; a `1428.00` cost gap that is exactly
  `357 * random_page_cost` between 49.73% and 0% `leaf_fragmentation`, leaving
  zero residual for fragmentation; two 2,745-block indexes over the same 100,000
  rows both priced at `12730.42` while `avg_leaf_density` read 9.27% for the
  scattered-delete case and a healthy-looking 89.18% for the contiguous-delete
  case that hides 2,465 deleted pages; a forged `pg_class.relpages = 1` that
  left cost unchanged at `25528.42` for a non-partial index while forging a
  partial index's `reltuples` moved it from `24140.12` to `23140.29`; a
  `cpu_operator_cost = 1` run isolating the height charge to exactly `50.00`; a
  plan flip to `Seq Scan` at 25% selectivity; an index dropped from a
  `BitmapAnd` with `c = 7` demoted to a `Filter`; 4 versus 6 parallel workers;
  a SAOP descent plateau at exactly 3 versus 19 descents; 852 versus 2,749
  blocks with `deduplicate_items` off (gaps `68.00` and `7516.00` matching page
  arithmetic exactly); and 583 versus 1,174 blocks of version-churn growth
  without and with a held `REPEATABLE READ` snapshot.
- Recorded the explicit test absence: `src/test` contains no reference to
  `tree_height`, `btcostestimate`, or `genericcostestimate`, and the only
  in-tree `pgstatindex` test runs against an empty index and expects `NaN` for
  both density and fragmentation.
- Six follow-ups recorded under `## Open Questions`, including the one-cent
  rounding gaps in the verification SQL, the single-workload nature of the
  churn measurement, and the un-isolated heap-versus-index share of the
  `effective_cache_size` effect.
- The isolated server was stopped, the blocking snapshot session terminated,
  and disposable fixtures and SQL scripts remain under `.wiki-runtime/`.
  Human `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` are all unchanged pending a separate
  claim-by-claim review. Updated `wiki/index.md`, `wiki/v17/index.md`, and
  `wiki/versions.md`. `scripts/wiki_lint` reports 0 errors and 0 warnings.

## [2026-08-04] follow-up v12 | when the planner discards a GIN index for a B-tree

- Extended [Planner Penalties for Bloated Indexes in PostgreSQL 12
  (unverified)](v12/questions/query-planning/bloated-indexes-query-planner.md) against the
  unchanged `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`) pin. Prompt
  hygiene was applied before drafting: the user approved the corrected follow-up
  "When might a GIN index be discarded by the query planner and a B-tree used
  instead?", asked for source plus exact-pin tests, and approved normalizing the
  page's `## Answer Up Front` heading to the required `## Answer` and adding a
  `## Contents` table of contents.
- Structured the answer around three independent gates and cited each: clause
  matching in `match_clause_to_indexcol()` / `match_opclause_to_indexcol()`
  (`op_in_opfamily()`, the `get_index_clause_from_support()` escape hatch, and
  the core `pg_amop.dat` GIN operator families, none of which carry `<`, `<=`,
  `>=`, or `>`); plan shape from `ginhandler` versus `bthandler`
  (`amgettuple = NULL` so bitmap-only, `amcanorder`/`amcanorderbyop` false,
  `amcanreturn = NULL`, `amsearchnulls`/`amsearcharray`/`amcanparallel` false)
  routed through `get_relation_info()`, `get_index_paths()`,
  `build_index_paths()`, and `check_index_only()`; and cost, contrasting
  `gincostestimate()`'s all-`random_page_cost` page charges — pending list
  included and no descent shortcut — with `genericcostestimate()` plus
  `btcostestimate()`'s pro-rata page share and cheap descent, then
  `add_path()`/`STD_FUZZ_FACTOR` and `choose_bitmap_and()` pruning.
- Added the 4X stale-metapage-statistics branch, the keyless full-index GIN path
  a proven partial predicate can still produce via `amoptionalkey` and
  `GIN_SEARCH_MODE_EVERYTHING` (with the unreachable pre-9.1
  `ginVersion < 1` error), the four `CREATE INDEX`-time AM rejections, the
  `btree_gin` partial-match range behaviour, where GIN still wins, key data
  structures, the caller/callee chain, build/generated-catalog and contrib
  boundaries, eight settings with exact apply scopes (all `PGC_USERSET` plus the
  `fastupdate` reloption), and a three-query production SQL recipe.
- Exact-pin measurements on an isolated 12.2 server built from the checkout:
  `n = 42` cost `12.22` through GIN and `4.65` through a B-tree on one table
  with identical statistics (each alternative isolated by dropping the other
  index inside a rolled-back transaction), despite the GIN index being 279 pages
  against the B-tree's 826; B-tree also won `BETWEEN`, `n < 20`, `IN (1,2,3)`,
  `ORDER BY … LIMIT 10` (7381x), index-only scan (25x), and `IS NULL` (634x).
  `enable_seqscan = off` produced `disable_cost`-priced sequential scans at
  `10000000000.00` for the three no-path cases, proving no GIN path existed. A
  1177-page pending list moved GIN's index cost from `12.45` to `4720.55` and
  the planner dropped GIN from the `BitmapAnd`, demoting `tsv @@ …` to a
  `Filter`; cleaning restored it. The pending-page charge was verified exactly:
  `pgstatginindex` reported 393 pending pages, a two-`random_page_cost` probe
  recovered `(1588.55 - 397.55) / 3 = 397` charged pages, and
  `gin_clean_pending_list()` returned `393`. A keyless partial-GIN path charged
  187 pages for 186 index entries and still beat the `1834.00` sequential scan.
  The 4X fallback under-estimated by only 1.7% (`17.58` versus `17.89`), which
  is filed under `## Open Questions`.
- All three filed SQL blocks ran verbatim against a fixture literally named
  `my_table` with `my_gin_index` and `my_btree_index`, returning
  `pending_startup_cost = 200` for 50 pending pages and a GIN cost drop from
  `217.51` to `17.51` after cleanup. The isolated server was stopped and its
  disposable fixtures were left under `.wiki-runtime/`.
- Recorded explicit test absence: no core or contrib test compares a GIN plan
  with a B-tree plan, `gincostestimate()` has no coverage, every GIN `EXPLAIN`
  in the tree uses `COSTS OFF`, and the one asserted GIN-rejected-for-`Seq Scan`
  case (`contrib/btree_gin/expected/bool.out`) is caused by boolean clause
  simplification rather than costing.
- Refreshed `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Reset
  `verified_by_agent` to `not yet` because this was a scoped follow-up rather
  than a fresh full-page claim audit; human `verified: false` and the visible
  `(unverified)` title are unchanged.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. All
  201 distinct source citations on the page were re-checked for existence and
  in-range line numbers, and every `## Contents` anchor was checked against the
  page's headings.

## [2026-08-03] answer v17 | GIN index bloat and how to measure it

- Filed How a GIN Index Becomes Bloated in PostgreSQL 17, and How to Measure It
  (unverified) against unchanged pin
  `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10).
- Applied `MANDATORY Prompt Hygiene` first: the prompt contained the typo
  "postgreql", a space before a comma, a double space, and a lowercase sentence
  start. The user approved the corrected wording, chose contrib and core-SQL-only
  measurement paths presented as separate sections, asked for a full exact-pin
  empirical run, and asked for a "what changed since PostgreSQL 12" section.
- Established seven bloat mechanisms from v17 source: pending-list accumulation
  with the capacity-based trigger plus internal fragmentation, the
  `ConditionalLockPage` bail-out with a `work_mem` budget, and the remembered
  tail; an entry tree that is never deleted from; entry splits that equalize by
  data size with no rightmost or build case; posting-tree split headroom; VACUUM's
  refusal to re-encode posting segments or merge siblings; XID-gated posting-page
  reuse through `GlobalVisCheckRemovableXid`; and a fork ordinary VACUUM never
  shortens. Added the two v17 paths that skip GIN entirely, the 2%-of-pages
  index-vacuum bypass and the wraparound failsafe.
- Corrected a widespread assumption with measurements. Because `entrySplitPage`
  equalizes total data size with no rightmost or build case, and `ginbuild` drains
  its red-black accumulator in ascending key order, a fresh build settles at
  50.31%-52.72% entry-leaf fill. `REINDEX` therefore grew a scattered-retail index
  from 7,692,288 to 8,765,440 bytes (57.79% -> 50.73% fill, 933 -> 1063 leaves),
  while an ascending-retail index was byte-identical to its rebuild at 7,233,536
  bytes; `maintenance_work_mem` of 1 MB and 64 MB produced identical builds. The
  page now states that ~50% entry-leaf fill is structure, not bloat, and that a
  rebuild probe can legitimately report negative reclaimable bytes.
- Filed ten measurement recipes, five contrib and five core-only:
  `pgstatginindex` pending-list survey, a `pageinspect` page-class census that
  separates deleted posting pages from former pending pages, a `page_header`
  leaf-fill probe standing in for the absent GIN `avg_leaf_density`, a
  metapage-versus-live-size staleness check mirroring `gincostestimate`'s 4x
  cutoff, `pg_freespacemap` reusable pages, core size/fork/catalog staleness,
  `VACUUM VERBOSE` per-index page classes, a two-`random_page_cost` `EXPLAIN`
  probe, `gin_clean_pending_list` as an exact count, and a
  `CREATE INDEX CONCURRENTLY` rebuild probe.
- The `EXPLAIN` probe is new work for this pin. Because the only per-page term
  that scales with `random_page_cost` is the page count, the cost difference
  divided by the `random_page_cost` difference is exactly
  `entryPagesFetched + dataPagesFetched`. Measured 591.00 against 589 pending
  pages in a 591-page index and 101.00 against 99 pending pages in a 101-page
  index. Also documented the single-plan form and its 4.125-per-page scale from
  `random_page_cost + 50 * cpu_operator_cost`.
- Exact-pin measurements: 1471 pending pages in a 12,066,816-byte index whose
  forced `Bitmap Index Scan` cost 6260.82 with 1473 buffers, falling to 17.63 and
  4 after `gin_clean_pending_list` returned 1471 and the main fork *grew* to
  15,581,184 bytes with a new 24,576-byte FSM fork and 1471 pages recorded free at
  `avail = 8160`; a 16,801,792-byte entry tree unchanged after deleting all
  300,000 rows and two VACUUMs, then 16,384 bytes after `REINDEX`; posting leaves
  at 7.50% fill with 76 deleted pages entering the FSM only after three
  `txid_current()` calls advanced the horizon; a churn fixture at 93.65%
  reclaimable whose probe `fresh_bytes` of 131,072 matched the later `REINDEX`
  byte-for-byte; `reltuples` of 300,000 / 100,000 / 300,000 / 90,000 across
  `CREATE INDEX`, `ANALYZE`, `REINDEX`, and VACUUM against a true entry count of
  2,988; `idx_tup_fetch` structurally 0 with `idx_tup_read` 268; and the exact
  rejection messages from `pgstatindex`, `pgstattuple`, `pgstattuple_approx`, and
  `bt_index_check`.
- Attributed every since-v12 change to the checkout's own history, bracketed by
  the `Stamp 12.0` through `Stamp 17.0` commits and checked with
  `git merge-base --is-ancestor`: v13 `4d8a8d0c738`, `ec28808ba85`,
  `4b754d6c16e`; v14 `5100010ee4d`, `3499df0dee8`, `1e55e7d1755`, `c242baa4a83`,
  `dc7420c2c92`, `23763618390`; v16 `eb5c4e953bb`, `cd9479af2af`; v17
  `667e65aac35`, `b4375717147`, and `13503eb5905` (noted as back-patched to
  `REL_12_STABLE` as `975ae05537f`, so a minor-release rather than major-version
  difference). Listed what is unchanged, including the absence of parallel GIN
  build, GIN `amcheck` support, and a GIN `fillfactor`.
- All nine filed SQL blocks ran verbatim with `ON_ERROR_STOP=1` on an isolated
  exact-pin 17.10 server built from the pinned checkout under `.wiki-runtime/`,
  with `pgstattuple`, `pageinspect`, `pg_freespacemap`, `amcheck`, `btree_gin`,
  and `pg_trgm` installed. The server was stopped; disposable SQL, logs, and the
  cluster remain under `.wiki-runtime/`.
- Audited all 177 distinct citation ranges programmatically for existence and
  bounds, then re-cut every range whose first line fell mid-comment or on the
  wrong symbol; also verified that all 39 in-page anchors resolve and that
  `## Contents` lists every `##`/`###` heading in document order.
- Filed eight `## Open Questions`, including the unmeasured autovacuum partial
  clean, the `int[]`-only density fixtures, the absence of any bloat threshold in
  the pinned tree, the `indexfsm.c` `BLCKSZ - 1` comment versus the measured 8160,
  and the probe's untested accuracy on an index with fresh metapage statistics.
- Updated `wiki/index.md`, `wiki/v17/index.md`, and `wiki/versions.md`. Kept
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet`, because the per-claim re-verification pass has not
  been done as a separate review.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-03] follow-up v12 | GIN wasted bytes from core SQL only

- Filed a core-SQL-only follow-up on How a GIN Index Becomes Bloated in
  PostgreSQL 12, and How to Measure It (unverified) against the unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
- Applied prompt hygiene before drafting. The user approved the corrected
  wording, "How, using SQL and no extra contrib extension, can I measure the
  wasted bytes of a GIN index and provide a bloat percentage based on wasted
  bytes?", chose core-only scope (no `pgstattuple`, `pageinspect`,
  `pg_freespacemap`, or `amcheck`), and asked for all three candidate methods
  ranked.
- Defined wasted bytes as rebuild-reclaimable bytes, then established from
  source why nothing weaker is measurable: `pg_proc.dat` exposes no page,
  page-header, metapage, or FSM-contents function; `GinStatsData` is reachable
  only from `ginGetStats`' C callers; and no `pg_statistic` slot holds a
  distinct-element count, so `pg_stats.elem_count_histogram` gives postings per
  row for array columns only (`array_typanalyze.c`) and nothing for `tsvector`
  (`ts_typanalyze.c`, MCELEM only) or `jsonb` (no `typanalyze`).
- Ranked three methods. A: a `CREATE INDEX CONCURRENTLY` rebuild probe, exact.
  B: a recorded bytes-per-row baseline, read-only, with a baseline table, an
  `ON CONFLICT` refresh statement, and a growth-since-last-stats companion
  query. C: a `TABLESAMPLE` extrapolation, rejected.
- Measured on five deterministic fixtures (mid-cardinality arrays with churn,
  unique-key arrays fully deleted, churned `tsvector`, an untouched control, and
  a `fastupdate = on` index whose pending list was never cleaned). Method A's
  `fresh_bytes` equalled the later `REINDEX` size byte-for-byte in all five
  cases: 3,768,320 / 16,384 / 3,751,936 / 5,332,992 / 2,924,544. Method B stayed
  within 2.75 percentage points of probe truth across eight comparisons over two
  further churn rounds and returned exactly 0.00% on the control. Method C erred
  by +17% to +455% on fresh size and reported −383.67% bloat on the 0%-bloated
  control; the cause is traced to GIN size being strongly sublinear in row count
  (bytes per row fell 64.5 -> 29.4 -> 21.5 -> 16.6 -> 13.3 as the sample grew to
  the full 400,000 rows) through the `GinMaxItemSize` in-line-to-posting-tree
  transition, saturating distinct keys against 50/50 entry splits, and
  amortizing 128-to-384-byte segment overhead.
- Documented the `pg_class.reltuples` trap for GIN: extracted keys after
  `CREATE INDEX`/`REINDEX` (799,493 on `arr_churn_gin`), heap rows after
  `VACUUM` (266,667 on the same index), a sampled fraction after `ANALYZE`, and
  unchanged when VACUUM skipped heap pages. Method B therefore divides by the
  table's `reltuples`, not the index's.
- Added a fork-selection section: `pg_relation_size` is main-fork only,
  `pg_table_size`/`pg_total_relation_size` add the FSM fork, and
  `pg_indexes_size` on an index is always 0. Four of five fixtures had a 0-byte
  FSM fork; after `gin_clean_pending_list` the fifth reported main 11,247,616
  bytes against `pg_table_size` 11,272,192.
- Added two core-only pending-list probes. The `Bitmap Index Scan` node's total
  cost is exactly `indextotalcost` (`createplan.c` forces its startup to 0), and
  `gincostestimate` seeds the entry-page fetch count from the pending-page count
  at `random_page_cost`, so cost ÷ `random_page_cost` measured 1,471.7 and 887.5
  against 1,469 and 883 real pending pages, and 5.3 after the list was cleaned.
  `gin_clean_pending_list` returned 883 exactly and grew the file from
  10,158,080 to 11,247,616 bytes.
- All eight fenced SQL blocks ran verbatim with `ON_ERROR_STOP=1` on an isolated
  exact-pin 12.2 server under `.wiki-runtime/ginwaste-data`. `pgstattuple` was
  installed only to cross-check the pending-page proxy and is used by no filed
  recipe. The server was stopped; disposable SQL, logs, and the cluster remain
  under `.wiki-runtime/`.
- Verified every new citation range by direct read before drafting, and checked
  that all 42 `## Contents` entries plus the six in-body anchors resolve, in
  document order and at the right nesting level.
- Added seven entries under `## Open Questions`, including that no core-only
  method produces an absolute figure for an index that has never been rebuilt,
  that method B's 2.75-point bound is fixture-specific rather than a tolerance,
  and that `pg_read_binary_file` was not tested as a page decoder.
- Extended `## Context Reviewed`, `## Evidence Map` (27 new rows), and
  `## Source References`, and refreshed `wiki/index.md`, `wiki/v12/index.md`,
  and `wiki/versions.md`.
- Reset `verified_by_agent` to `not yet` because this was a scoped follow-up
  rather than a fresh full-page claim audit; human `verified: false` and the
  visible `(unverified)` title are unchanged.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-03] review-fix v12 | GIN index bloat and how to measure it

- Re-audited How a GIN Index Becomes Bloated in PostgreSQL 12, and How to
  Measure It (unverified) claim by claim against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2).
- Corrected four material overstatements. Posting-page split behavior is expressed
  as segment-balancing and append/build heuristics, not exact 50%/75%/100%
  occupancy. An autovacuum partial clean stops at its remembered tail but can
  still empty the pre-existing list without concurrent append. Ordinary VACUUM
  does not truncate a GIN main fork, but `VACUUM FULL`, `CLUSTER`, and `TRUNCATE`
  can rebuild or reset index storage. Deleted posting-tree pages and deleted
  former pending-list pages now have separate census classes, and FSM-recorded
  reuse is reported independently from the persistent `GIN_DELETED` flag.
- Added a second entry-tree retention mechanism: in-line posting tuples can grow
  enough to split entry pages, then shrink to posting-tree pointers without those
  entry pages ever merging. Qualified posting-leaf VACUUM as per-segment
  recompression within the former byte budget, the index-cleanup caller gate, FSM
  hints and candidate validation, the planner's strict one-quarter statistics
  test, and the core/generated-header/extension boundaries.
- Reworked all five production measurement recipes: every leading SQL verb has a
  wiki tag, session-scoped `statement_timeout` and `lock_timeout` bounds are stated,
  the census distinguishes both deleted page origins and FSM state, and the
  rebuild probe reproduces the exact logical index definition and calls out its
  concurrent-build overhead and snapshot limitation. All five snippets ran
  verbatim with `ON_ERROR_STOP` against the exact pin, including concurrent probe
  creation and removal.
- Reran deterministic exact-pin fixtures. Results include 411 pending pages
  cleaned into reusable high-water space; a 16,801,792-byte, 300,000-key entry
  tree surviving deletion of every row; 8.2% reclaimed from retained entry pages
  while posting-page counts stayed equal; a sparse posting tree shrinking from
  1,654,784 to 98,304 bytes only on `REINDEX`; 12 deleted posting pages moving
  from zero to 12 FSM records after the horizon advanced; and a 1471-page pending
  list moving the forced bitmap-index cost from 27.25 to 5903.25.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Recorded
  `verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-08-03T18:24:11Z`; human
  `verified: false` and the visible `(unverified)` title remain unchanged. The
  isolated server was stopped; disposable evidence remains under `.wiki-runtime/`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-03] answer v12 | GIN index bloat and how to measure it

- Filed How a GIN Index Becomes Bloated in PostgreSQL 12, and How to Measure It
  (unverified) against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (PostgreSQL 12.2). Applied
  `MANDATORY Prompt Hygiene` first: the prompt contained the typo "postgreql" and
  phrased its first clause as an imperative with a question mark, and the user
  approved the corrected wording before drafting.
- Separated six bloat mechanisms from source: pending-list (`fastupdate`)
  accumulation with a capacity-based trigger plus three ways the trim falls short
  (per-row page dedication, `ConditionalLockPage` bail-out, and stopping at the
  remembered tail); an entry tree that is never deleted from, where
  `ginVacuumEntryPage` rewrites an emptied entry tuple with a null posting list;
  posting-tree leaf splits at 50/50 or 75% outside a build against full packing
  when `btree->isBuild`; VACUUM's documented refusal to re-encode sparse segments,
  with no sibling merging anywhere and deletion restricted to fully empty
  non-edge pages; deleted pages gated on `RecentGlobalXmin` rather than on a
  VACUUM count; and a relation GIN never truncates.
- Recorded the autovacuum asymmetry (`full_clean = !IsAutoVacuumWorkerProcess()`
  in both VACUUM entry points, autoanalyze cleaning the pending list while manual
  `ANALYZE` does not), the `gincostestimate` pending-page charge at
  `random_page_cost`, the 4x `nTotalPages` staleness cutoff and its
  invented-statistics fallback, the live-block-count planner input with
  `tree_height = -1` for GIN, and apply scopes: `gin_pending_list_limit` is
  `PGC_USERSET` (session/transaction, no restart or reload, confirmed as
  `context = user` in `pg_settings`), while the `fastupdate` and per-index
  `gin_pending_list_limit` reloptions take `AccessExclusiveLock`.
- Documented the measurement surface, including the negative findings:
  `pgstatginindex` reads only the metapage and returns `version`,
  `pending_pages`, `pending_tuples`; `pgstatindex`, `pgstattuple`,
  `pgstattuple_approx` and `amcheck` all reject a GIN index; `pg_freespace` is
  binary for indexes because `RecordFreeIndexPage` writes `BLCKSZ - 1`;
  `idx_tup_fetch` is structurally 0 because GIN sets `amgettuple = NULL`; and
  `pageinspect`'s `gin_page_opaque_info` validates nothing while `maxoff` is not
  an entry-tuple count. Corrected a research claim that v12 `pageinspect` lacks
  GIN functions: `contrib/pageinspect/ginfuncs.c` defines all three.
- Built the exact pin out of tree under `.wiki-runtime/` and added
  `pg_freespacemap`, `amcheck`, `btree_gin`, and `pg_trgm` to the existing
  install. All five filed SQL recipes ran verbatim with `ON_ERROR_STOP=1`, and
  the pending-list survey returned only GIN indexes in a database that also held
  B-tree, hash, GiST and BRIN indexes.
- Exact-pin measurements: a 300,000-distinct-key index held 16,801,792 bytes and
  still reported 300,000 metapage entries after every row was deleted and two
  VACUUMs ran, then `REINDEX` cut it to 16,384 bytes; deleting 760,000 of 800,000
  rows freed no bytes, and VACUUM #1/#2/#3 recorded no free pages until three
  `txid_current()` calls advanced the counter past delete XID 511, after which one
  VACUUM recorded all 80 pages at `avail = 8160`; 60,000 subsequent rows consumed
  those 80 recycled pages; 800,000 rows inserted in random key order left the
  index 27.8% larger with 24.4% more posting-tree leaves than after `REINDEX`,
  with no dead rows; a 1471-page pending list raised a `Bitmap Index Scan`
  estimate from 27.25 to 5903.25 and buffers from 4 to 1473 (0.027 ms versus
  9.793 ms), both costs reproduced exactly from the source formula; and
  `ALTER INDEX ... SET (fastupdate = off)` left all 246 pending pages in place.
- Filed six `## Open Questions`, including the inverted `indexfsm.c` header
  comment (it describes `BLCKSZ - 1` as marking used pages, while
  `RecordFreeIndexPage` uses it for free pages and the exact-pin run agrees with
  the code), the absence of any steady-state occupancy figure or bloat threshold
  in the pinned tree, the unmeasured autovacuum partial clean, and within-leaf
  slack not quantified via `gin_leafpage_items`.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Kept
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet`, because the page's per-claim re-verification pass
  has not been done as a separate review. The isolated 12.2 server was stopped
  and its disposable fixtures were left under `.wiki-runtime/`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-13] answer v18 | declarative partition bound syntax and relpartbound since PG 10

- Filed [Changes to Declarative Partition Bound Syntax and pg_class.relpartbound
  Since Partitioning Was Introduced, as of PostgreSQL 18
  (unverified)](v18/questions/server-administration/declarative-partition-bound-syntax-and-relpartbound.md)
  against unchanged pin `6cb307251c5c6261286c1566496920976640108e`. Applied
  `MANDATORY Prompt Hygiene`: the user chose to correct the prompt's typos
  (`postgreql`, `is there any changes`, `changes on syntax`, the stray space
  before the comma, lower-case `postgresql`), chose the pinned v18 checkout's own
  git history as the pre-v12 baseline, and scoped "syntax" to the bound spec only.
- Verdict: yes on both, but every substantive change landed by v12/v17.
  `git log -L '/^PartitionBoundSpec:/,/^hash_partbound:/:src/backend/parser/gram.y'`
  over `REL_10_0..6cb30725` returns exactly six commits: `6f6b99d1335`
  (`DEFAULT`, 11.0), `1aba8e651ac` (hash `WITH (MODULUS, REMAINDER)`, 11.0),
  `95931133a95` (typo, 12.0), `7c079d7417a` (general `partition_bound_expr`,
  12.0), `30ed71e423e` (indent, 15.0) and `2d8bff603c9` (18.0, error cursors on
  the two hash-bound errors). `d363d42bb9a` replaced `UNBOUNDED` with
  `MINVALUE`/`MAXVALUE` inside the v10 cycle, and `9361f6f54e3` added the
  sentinel-ordering rule in 11.0. The documented synopsis is textually identical
  from 12.0 through the pin.
- `pg_class.relpartbound` itself never changed; the stored `pg_node_tree` changed
  twice (v11 added `is_default`/`modulus`/`remainder`; v17's `d20d8fbd3e4` writes
  `-1` for every `:location`). Confirmed the v16 generated-node-support switch
  `964d01ae90c` is output-compatible, and that `outDatum`, `_outConst` and the
  four `WRITE_*` macros used by the bound are byte-identical to `REL_10_0`.
  Exposure surfaces only gained additions, newest being v14's ` DETACH PENDING`;
  the `pg_get_expr` deparse path has no functional commit since 10.0.
- Built the exact pin out of tree under `.wiki-runtime/pg18/` and ran an isolated
  18.3 server. Captured the literal stored node text for hash/list/range/default
  bounds (`:location -1` in 14/14 bounds), `IN ((2+1), 5, NULL)` stored as `Const`
  3, reversed hash options canonicalized, quoted `"minvalue"` accepted while
  `"MINVALUE"` is rejected, `FROM (-5)` deparsing as `FROM ('-5')`,
  `pretty = true` identical on 14/14, detach/reattach and `pg_dump -s` round
  trips, `DETACH` clearing the column, absent MERGE/SPLIT syntax, the new v18
  error cursor, and a no-TOAST size cliff between an 8,000-character bound
  (7,627 stored bytes) and an 8,400-character one (`row is too big: size 8216,
  maximum size 8160`). The server was stopped and its data directory removed;
  `raw/postgres-18/` was left unmodified.
- Recorded three test gaps (no test asserts the stored node text, the size
  cliff, or the quoted-sentinel behavior) and four open questions. Kept
  `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` because this is a first filing rather than a
  separate full-page claim audit.
- Updated `wiki/index.md`, `wiki/v18/index.md`, and `wiki/versions.md`.
  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-13] remove v12 | delete Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in COMMENT

- Removed `wiki/v12/questions/indexing/comment-stored-index-heap-ratio-bloat.md`
  per user request, which named the page by its exact visible title. The
  request's `postgreql` typo drives no document generation and is restated on no
  wiki page, so no prompt-hygiene correction was needed.
- Removed the active page entry from `wiki/index.md` and `wiki/v12/index.md`, and
  the two inbound `## Navigation` links in
  `comment-stored-bytes-per-table-tuple-non-btree.md` and
  `comment-stored-table-dml-counters-gin-reindex.md`. No surviving v12 page
  referred to the deleted page in prose.
- Its own outbound links to `btree-index-bloat-core-sql-only.md`,
  `how-pgstatindex-calculates-information.md`,
  `leaf-density-vs-fragmentation-index-scan-io.md`,
  `reindex-index-concurrently.md`, and
  `../query-planning/bloated-indexes-query-planner.md` went away with it, and all
  five keep inbound links from `wiki/index.md` and `wiki/v12/index.md`.
- Dropped the index/heap-ratio clause of the v12 coverage cell in
  `wiki/versions.md`: the allocation-growth reclassification, the source-audit
  list, the one-, two- and three-pass delete/reload numbers, the 49-cell
  drift-versus-"index larger than the heap" scores, and the
  `REINDEX INDEX CONCURRENTLY` section. The cell's remaining COMMENT text
  describes the surviving bytes-per-table-tuple page.
- Neutralized the remaining historical Markdown links in `wiki/log.md` (7) and
  `wiki/versions.md` (6) so they name the page title as plain text, and added a
  removal note to `## Coverage Notes`.
- Left the separate v17 page
  `wiki/v17/questions/indexing/comment-stored-index-heap-ratio-bloat.md`, its
  inbound links, and the provenance prose that describes it as copied from the
  v12 question untouched, so v17 still carries this proposal.
- v12 COMMENT-stored index-bloat coverage now runs through
  [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 12 (unverified)](v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md)
  and Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40% in
  PostgreSQL 12? (unverified),
  and core-SQL bloat measurement through
  [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md).
- No source citation, pin, or verification field on any surviving page changed.
  `raw/postgres-12/` is unchanged and no server was started.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-13] answer v17 | copy and review the v12 COMMENT-stored bytes-per-table-tuple REINDEX threshold

- Filed Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 17 (unverified)
  against unchanged pin `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10,
  `REL_17_10-3-g54eeefaedbe`). Applied `MANDATORY Prompt Hygiene`: the request
  said `follow agents.md, In PostgreSQL 17, ...` and quoted the v12 title with
  its ` (unverified)` state marker; the user chose to correct all three issues
  (the `AGENTS.md` sentence, the comma splice with a mid-sentence capital, and
  the state marker), so `## Question` restates the corrected prompt plus the
  copied v12 question text. The user also chose full re-measurement over a
  source-only review.
- Verdict: the v12 calibration transfers almost unchanged and **only GiST moved**.
  Re-ran the entire v12 protocol on an isolated 17.10 server over 132 cells
  (22 workloads x 6 index shapes) with `REINDEX INDEX` as ground truth and
  "worth rebuilding" defined as reclaiming >= 20% of current bytes. hash 9-11%
  (16/0/0/6), SP-GiST 29-32% (15/0/0/7), GIN `fastupdate = on` 56-100%
  (18/0/0/4), GIN `fastupdate = off` 40-62% (16/2/0/4) and BRIN-unusable
  (0/4/0/18) reproduce the v12 page's confusion matrices cell for cell, and the
  hash, GIN, SP-GiST and BRIN result columns are identical cell for cell. GiST's
  perfect window narrows from 14-33% to 25-33%, with its lower edge set by one
  `churn25` cell at +24.46% against 19.65% reclaimable, so 30% is recommended
  instead of 25%. Single all-AM thresholds are unchanged: 30% -> 90.2% over 132
  cells and 93.6% over the 110 non-BRIN cells.
- Root cause of the GiST shift, traced in source and measured: when every key
  column has a `GIST_SORTSUPPORT_PROC`, v17 forces `GIST_SORTED_BUILD`
  (`gistbuild.c#L231-L248`) and that path carries
  `/* fillfactor ignored */` (`gistbuild.c#L468`). Measured a 3 x 4 sweep in
  which fillfactor 100/90/50/10 all yield 1,660 pages under
  `buffering = auto`/`off` but 1,633/1,775/3,158/18,824 under `buffering = on`,
  with an `int4range` control (whose `range_ops` has no sortsupport) honouring
  fillfactor throughout at 1,379/1,538/2,816/15,383. `point_ops` is the only
  sortsupport-capable GiST opclass among the four probed. GiST rebuild also fell
  to 244.4 ms, below SP-GiST's 310.4 ms, and rebuilt sizes are now
  byte-deterministic while the bloated size still varies by a few pages.
- Two hazards v12 could not have. `pg_class.reltuples = -1` for "unknown" makes
  the original formula return a negative bytes-per-tuple: measured
  -7,438,336.00 on a never-analyzed 200,000-row table, with `TRUNCATE`
  resetting the heap and both indexes to `-1` and `REINDEX` of an empty table
  leaving it. And the v14 2%-of-pages index-vacuum bypass fired as
  `index scan bypassed: 41 pages from table (0.38% of total) have 1500 dead item
  identifiers`, with the next pass removing 3,000 rather than 1,500. Every guard
  on the filed page and in the filed SQL now tests `reltuples <= 0`.
- Also measured on the pin: parallel BRIN build (new in v17) leaving the index
  byte-identical at 0/2/4 workers (139 pages at `pages_per_range = 1`, 3 at the
  default) with 4 workers observed live in `pg_stat_progress_create_index` and
  70.2 ms against 180.6 ms; the GIN pending-list four-stage table where a flush
  moves the reading from +261.73% to a worse +388.78% while delivering the whole
  2.8x block win; the hash triple of build-path spread
  (34.5293/38.0928/37.1917 bytes per row), splitpoint staircase (28.071 to
  52.634) and an 87-page filesystem hole confirmed by `stat` after `CHECKPOINT`;
  a partial index growing +96.4% but reading +3.38% against 49.09% reclaimable
  (+3.07%/48.94% on an independent re-run); `REINDEX INDEX CONCURRENTLY`
  matching plain `REINDEX` byte for byte on all six shapes with the surviving
  baseline reading -48.86% to -79.36%; a partitioned parent at
  `relpages = -1, reltuples = 200000` whose parent index has
  `pg_relation_size = 0` and whose comment reached 0 of 3 children; and
  `pgstattuple` no longer erroring on a hash index with a splitpoint hole
  (0 of 21 hash cells errored), which `036decbba2a` shipped in 17.7.
- The since-v12 section attributes 17 changes to their first release tags from
  the checkout's own history, and records what did not change: `indexfsm.c` has
  zero executable-code changes since `REL_12_0`, `ginfast.c`'s flush threshold
  and `shiftList` FSM handling are untouched, `_hash_alloc_buckets` is
  functionally identical, and GIN builds are still serial. Parallel GIN build
  (`8492feb98f6`, 18.0) is not an ancestor of the pin, so the page states the
  constants must not be reused for v18.
- All v12 comparisons are explicitly framed: behavioral differences are
  attributed to commits in this checkout's history, while every numeric
  "v12 measured X" is labelled a cross-reference to the v12 page rather than
  evidence from this pin.
- Built `contrib/pg_freespacemap` out of tree with `USE_PGXS=1` into
  `.wiki-runtime/pg17/install`; `raw/postgres-17/` verified unmodified with
  `git status --porcelain`. The isolated 17.10 server was stopped and its data
  directory removed; the harness and captured output remain under
  `.wiki-runtime/tmp/bpt17/`. Fifteen open questions filed, including the
  knife-edge GiST window, the one fired GiST cell that needed 7% more blocks
  after the rebuild, and the unmeasured cumulative cost of the bypass.
- Kept `verified: false`, the visible `(unverified)` title, and
  `verified_by_agent: not yet` because this is a first filing rather than a
  separate full-page claim audit. Updated `wiki/index.md`, `wiki/v17/index.md`,
  and `wiki/versions.md`.

## [2026-08-17] remove v12 | delete Can COMMENT-Stored Table DML Counters Trigger GIN REINDEX at 40%

- Removed `wiki/v12/questions/indexing/comment-stored-table-dml-counters-gin-reindex.md`
  per user request, which named the page by its exact visible title. The
  request's `remove question:` phrasing drives no document generation and is
  restated on no wiki page, so no prompt-hygiene correction was needed.
- Removed the active page entry from `wiki/index.md` and `wiki/v12/index.md`, and
  the one inbound `## Navigation` link in
  `comment-stored-bytes-per-table-tuple-non-btree.md`. No surviving v12 page
  referred to the deleted page in prose.
- Its own outbound link to `reindex-index-concurrently.md` went away with it,
  along with its three navigation links to `wiki/v12/index.md`,
  `wiki/index.md`, and `wiki/versions.md`; `reindex-index-concurrently.md` keeps
  inbound links from `wiki/index.md`, `wiki/v12/index.md`, and
  `comment-stored-bytes-per-table-tuple-non-btree.md`.
- The v12 coverage cell in `wiki/versions.md` never described this page, so no
  coverage clause was dropped; its COMMENT-related text covers the surviving
  bytes-per-table-tuple page.
- Neutralized the remaining historical Markdown links in `wiki/log.md` (4) and
  `wiki/versions.md` (4) so they name the page title as plain text, and added a
  removal note to `## Coverage Notes`.
- v12 COMMENT-stored index-bloat coverage now runs through
  [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 12 (unverified)](v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md)
  and core-SQL B-tree bloat measurement through
  [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md).
  No v12 page proposes a table-DML-counter GIN rebuild trigger any more, and no
  other version ever carried a copy of this question.
- No source citation, pin, or verification field on any surviving page changed.
  `raw/postgres-12/` is unchanged and no server was started.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-17] tooling | repin citation re-anchoring, label sync, and a pin-vs-checkout lint check

- Added `scripts/repin_citations`, which re-anchors `#Lstart-Lend` fragments when a
  `raw/postgres-NN/` checkout is repinned. It reads the cited block at the old
  commit and locates it at the new commit, reporting one of `unchanged`, `exact`,
  `context`, `endpoints`, `nearest`, `changed`, `missing-new`, `missing-old`, or
  `out-of-range`. `--apply` rewrites only mechanically resolvable fragments,
  `--sync-labels` also updates labels that embed their own line numbers such as
  `[index.c:2622-2626]`, and `--update-pinned-commit` sets front-matter pins.
  Reports land in `.wiki-runtime/cache/repin_citations/vNN.json`.
- Verified the tool before trusting it: 8 of 8 sampled moved citations had
  byte-identical old and new blocks, and after the run the label-symbol hit rate
  for re-anchored citations (`exact` 81.6%, `endpoints` 96.4%) matched or beat the
  untouched `unchanged` baseline of 77.5%, so the pass introduced no systematic
  drift. All 135 non-trivial re-anchors were checked individually: 28 `context`
  and 10 `nearest` blocks byte-identical, 97 `endpoints` boundary lines identical.
- Added a source-pin integrity check to `scripts/wiki_lint`. For every version in
  `wiki/versions.md` it now errors when the checkout is missing, when the pinned
  commit is not a full 40-character sha, when the pinned commit is absent from the
  checkout, or when the checkout HEAD is a different commit, and warns on an
  unclean checkout. Exercised all five cases against a throwaway repository.
- This closes the gap that hid the v19 drift below: lint compared page
  `pinned_commit:` values against the manifest but never against the checkout.

## [2026-08-17] repin v19 | 19beta2 99e47536 -> 19beta3 67342a14863, 152 commits reviewed

- Found and fixed a pin/checkout mismatch first: `wiki/versions.md` and all four
  v19 pages declared `99e47536bbf1a165f5dc8d504f928821ebc8df6a`, but
  `raw/postgres-19/` sat on `3aa54433b0cdce48facb610a5b720208cc760654`, 52 commits
  earlier, and the declared commit was not even present in the local clone. Every
  v19 citation line number therefore resolved against the wrong tree. Fetching
  `REL_19_STABLE` restored the object, and the repin moved the checkout forward
  from the declared pin.
- Repinned to `67342a148632801d44c2fb9a7bf4231b6827c5d2`
  (`REL_19_BETA3-26-g67342a14863`), stamped `19beta3` in `configure.ac` and
  `meson.build`. The range crosses the `REL_19_BETA3` stamp `3638289fb57` and
  contains a 35-commit security batch dated 2026-08-10.
- Reviewed all 152 commits: 14 change a filed claim, 29 touch covered files
  without changing a claim, 109 are unrelated.
- Page updates. `5d47df21e89` (CVE-2026-6471) moved the pgrepack direct-use
  rejection out of `repack_startup()` into `StartupDecodingContext()` and added the
  `output_plugin_libraries` GUC; `d7db169fa13` rejects `CLUSTER (ANALYZE)`;
  `6a80179f6b0` closed the decoding-deactivation versus slot-creation races;
  `fd56954c9fe` documented the `NO_LOGICAL` table-AM flags. REPACK feature-scope
  commits went 50 -> 55, including the page title. `3ab3f33281f` renamed
  EXISTS-to-ANY subplans to `exists_to_any_*`, changing `pg_plan_advice` target
  names and raising direct module commits 26 -> 27. `567286b762b` (CVE-2026-14666)
  added role/database plan-cache invalidation. `b3331578b58` restored the
  autovacuum launcher's descending `adl_score` sort, which had been ascending.
- 109 citation fragments re-anchored, 19 stale labels synced, three anchors
  repointed by hand (`plancache.c#L14-L42`, `autovacuum.c#L1118-L1124`,
  `slot.c#L372-L465`). `verified_by_agent` stays `not yet` on all four pages.

## [2026-08-17] repin v18 | 18.3 6cb307251c5 -> 18.6 baa7b142aac, 294 commits reviewed

- Repinned the primary version to `baa7b142aace6821ce085906f314a75bcc4d95c8`
  (`REL_18_6-10-gbaa7b142aac`, stamped 18.6). The range crosses the 18.4 and 18.6
  releases. 18.5 was stamped (`17fae4fbdd6`) but never released; the checkout's own
  release notes say so, and no `REL_18_5` tag exists, so the wiki does not call it
  a release.
- Reviewed all 294 commits: 17 change a filed claim, 82 touch covered files
  without changing a claim, 195 are unrelated.
- Page updates. `0b12f56bfac` (CVE-2026-14666) inverted the RLS page's central
  finding: `InitPlanCache()` now registers `PlanCacheRoleCallback` on
  `AUTHMEMROLEMEM`/`AUTHOID`/`DATABASEOID`, so same-role membership,
  `BYPASSRLS`, superuser and current-database-owner changes no longer leave a
  stale RLS decision, and its 18.3-line reproduction is now recorded as
  superseded. The RLS release-based history went 21 -> 24 changes.
  `2a29b607dbb` (CVE-2026-6471) added `output_plugin_libraries`, a new mandatory
  precondition for logical replication. `cb35d730689` (CVE-2026-6638) answered the
  bidirectional page's open question about unquoted publisher-side identifiers.
  `8a31ffc2d4c` (CVE-2026-14676) rebuilt `pg_stat_statements` normalization on an
  expansible `StringInfo`, without adding the length cap that page is about.
  `2780538433f`, `e4527519b77`, `fe464e9e686`/`5cc59834b86`, `2fd8d45ecf7`,
  `585181e0774`/`7c25cdb1ebf`, `ed8050370b7` and `d4420a97206` updated the CIC,
  custom-statistics, buffer-partition and vacuum-hook pages.
- No GUC present in both v12 and 18 changed its default in the range, so the
  GUC-defaults answer is unchanged; the page now says so explicitly.
- 633 citation fragments re-anchored, 145 stale labels synced, 13 anchors
  repointed by hand, including the four `bufmgr.c#BufferAlloc` ranges and five
  `heapam.c`/`libpqwalreceiver.c`/`subscriptioncmds.c` anchors whose cited text
  had moved. `verified_by_agent` reset to `not yet` where it carried a timestamp.

## [2026-08-17] repin v17 | 17.10 54eeefaedbe -> 17.11 786db8dcf16, 193 commits reviewed

- Repinned to `786db8dcf168bd9df8f55047337525ac19118b1c`
  (`REL_17_11-7-g786db8dcf16`, stamped 17.11), crossing the `REL_17_11` stamp
  `083ac033419`.
- Reviewed all 193 commits: 12 change a filed claim, 57 touch covered files
  without changing a claim, 124 are unrelated.
- Page updates. `1d6c654c818` replaced `CompareIndexInfo`'s blanket refusal of
  exclusion indexes with an element-wise comparison of exclusion operators,
  procedures and strategies, so the ATTACH page's exclusion-index non-reuse claim
  and its 17.10 measurement of that behavior are corrected and the new regression
  case cited. `31f2acde53d` fixed RANGE pruning wrongly discarding the DEFAULT
  partition. `28269fed661` propagates `INDEX_CREATE_DEFERRABLE` to the
  `REINDEX CONCURRENTLY` `_ccnew` copy and adds a third injection point.
  `d1c8aa0b09f` (CVE-2026-6470) requires `USAGE` on index-expression and
  predicate types. `01992176e08` (CVE-2026-6471) added `output_plugin_libraries`,
  and `14810cc0d96` rebuilt replication-command quoting, so the bidirectional
  page's "no related commits since 17.10" statement became a 17.11 section. Four
  refint commits added the PostgreSQL 20 removal notice and secure-schema rules to
  the contrib inventory.
- Measured pages keep every number, now labelled as measured on the previous
  17.10 pin with a per-page source check of whether the underlying code moved.
  No GUC default changed in 17.11.
- 929 citation fragments re-anchored, 326 stale labels synced, 11 anchors
  repointed by hand (four `index.c` exclusion anchors, two `read_stream.c`,
  `index.c#apply-reloptions`, four `libpqwalreceiver.c`, five `contrib-spi.sgml`
  sections). `verified_by_agent` reset to `not yet` on the three pages that
  carried a timestamp.

## [2026-08-17] repin v14 | 14.23 5c00f4e2e3b -> 14.24 a92fbdfb830, 133 commits reviewed

- Repinned to `a92fbdfb830046e907813e9067b2c9de9708d600`
  (`REL_14_24-6-ga92fbdfb830`, stamped 14.24), crossing the `REL_14_24` stamp
  `6b3806732b7`. PostgreSQL 14 has no meson build system, so `configure.ac` and
  the generated `configure` are the only stamp files; the navigation guide now
  records that, and that release tags on this checkout are unannotated, so
  `git describe --tags` is required.
- Reviewed all 133 commits: 6 change a filed claim, 30 touch covered files
  without changing a claim, 97 are unrelated.
- Page updates. `f4174aa84a3` (CVE-2026-14666) inverted the RLS page's central
  same-role plan-cache finding, and its 14.23 reproduction is recorded as
  superseded with the fix's two residual scopes (saved plans only; other
  databases' `pg_database` rows ignored) and its complete absence of in-tree
  tests. `1a358b8f2a2` (CVE-2026-6470) added the policy, index, default, CHECK and
  partition-key type-`USAGE` requirement. `155dacbc547` (CVE-2026-14680) added a
  fourth "cannot be called explicitly" class for `internal`-typed arguments and
  results. `802dc79df63` and `2bb60eb4fea` changed MultiXact wraparound hints and
  `RecordNewMultiXact()`'s SLRU locking, and `5100bdbd3ba` added an isolation spec,
  so the MultiXact page's "full isolation suite passed" claim is now scoped to the
  previous pin.
- 310 citation fragments re-anchored, 3 stale labels synced, one broken anchor
  repointed (`multixact.c#L1141-L1168`) and five duplicate labels given distinct
  names. `verified_by_agent` reset to `not yet` on the two pages that carried a
  timestamp.

## [2026-08-17] review-fix | 11 citations pointing past end-of-file, found by the new bounds check

- The new `repin_citations` bounds check surfaced 11 citations whose line range
  ran past the end of the cited file. All predate this repin and none was caught by
  `scripts/wiki_lint`, which checks only that the file exists.
- Fixed in v17 and v18: `execdesc.h#L33-L80` -> `#L33-L56` in a 70-line file and
  `memnodes.h#L117-L170` -> `#L117-L134` in a 152-line file, on both codebase
  navigation guides.
- Fixed in v12, whose pin `45b88269a353ad93744772791feb6d01bc7e1e42` did not move:
  `pg_stat_statements--1.6--1.7.sql#L13-L23` and `#L1-L23` -> `L13-L22`/`L1-L22`
  in a 22-line file, and five `timeouts.out` citations ending at `L76` in a
  73-line file -> `L73`.
- No prose changed on those pages; only the out-of-range endpoints.

## [2026-08-17] remove v17 | delete Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in COMMENT

- Removed `wiki/v17/questions/indexing/comment-stored-index-heap-ratio-bloat.md`
  per user request, which named the page by its exact visible title including the
  ` (unverified)` state marker. The request's `remove question:` phrasing and
  leading `#` drive no document generation and are restated on no wiki page, so
  no prompt-hygiene correction was needed.
- Removed the active page entry from `wiki/index.md` and `wiki/v17/index.md`, and
  the two inbound `## Navigation` links in
  `comment-stored-bytes-per-table-tuple-non-btree.md` and
  `btree-index-bloat-core-sql-only.md`. No surviving v17 page referred to the
  deleted page in prose.
- Its own outbound links to `reindex-index-concurrently.md`,
  `create-index-concurrently.md`,
  `../query-planning/bloated-indexes-query-planner.md` and
  `../server-administration/contrib-extensions.md` went away with it, along with
  its three navigation links to `wiki/v17/index.md`, `wiki/index.md` and
  `wiki/versions.md`; all four keep inbound links from `wiki/index.md` and
  `wiki/v17/index.md`.
- Dropped the index/heap-ratio clause of the v17 coverage cell in
  `wiki/versions.md`: the screening-signal-only verdict, the 49-cell
  drift-versus-"index larger than the heap" scores, the
  `hashbuild`/`estimate_rel_size` bucket-sizing and same-VACUUM-recycling
  (`9dd963ae253`) findings, the day-zero sweep, the GIN baseline-1.004900
  fixture, and the nine-operation comment durability matrix. The cell's remaining
  COMMENT text describes the surviving bytes-per-table-tuple page.
- Neutralized the remaining historical Markdown links in `wiki/log.md` (1) and
  `wiki/versions.md` (2) so they name the page title as plain text, and added a
  removal note to `## Coverage Notes`. The 2026-08-13 v12 removal note said the
  v17 page and its inbound links were untouched "so v17 still carries this
  proposal"; that sentence is now scoped to its date and points at this removal.
- The v12 page of the same name was removed on 2026-08-13, so no supported
  version carries this index/heap-ratio proposal any more. COMMENT-stored
  index-bloat screening in v17 now runs through
  Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 17 (unverified)
  and core-SQL B-tree bloat measurement through
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md).
- No source citation, pin, or verification field on any surviving page changed.
  `raw/postgres-17/` is unchanged and no server was started.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings;
  `git diff --check` passed.

## [2026-08-18] answer v17 | correct two reporting defects in the 12-through-17 B-tree bloat statement

- Extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with a
  fourth follow-up, against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). The follow-up prompt is
  grammatical, so it is restated verbatim under `## Question` with no
  prompt-hygiene note.
- Source-only follow-up: no server was started, `raw/postgres-17/` is unchanged,
  and both defects are confirmed from the pinned checkout alone. The user's
  framing is correct — neither column feeds `expected_blocks`, `floor_blocks`,
  `live_rows`, `key_groups`, `tids`, the deduplication gate, `status` or
  `caveats`, so no percentage on the page moves.
- Defect 1, `fsm_bytes > 0 AS has_freed_pages`, is a history bit. An index FSM
  records only free-or-used (`RecordFreeIndexPage` writes `BLCKSZ - 1`,
  `RecordUsedIndexPage` writes 0); the first recorded page extends the fork to
  three blocks / 24576 bytes at `block_size` 8192 (`SlotsPerFSMPage` 4069,
  `FSM_TREE_DEPTH` 3, leaf address maps to physical block 2), so its length is
  fixed rather than proportional. `_bt_allocbuf`'s `GetFreeIndexPage` marks pages
  used without shortening the file; no nbtree path reaches
  `FreeSpaceMapPrepareTruncateRel` (live `RelationTruncate` callers are
  `lazy_truncate_heap`, `heapam_relation_nontransactional_truncate` and
  `RelationTruncateIndexes`; the SP-GiST call sits in `#ifdef NOT_USED`); and only
  `RelationSetNewRelfilenumber` — `REINDEX` via `reindex_index`, transactional
  `TRUNCATE` via `ExecuteTruncateGuts` — resets it, creating the main fork alone.
  In the other direction `btvacuumpage` records a deleted page only once
  `BTPageIsRecyclable` holds, `_bt_pendingfsm_finalize` breaks at the first
  still-visible `safexid`, `btvacuumscan` vacuums the FSM only when `pages_free`
  is non-zero, half-dead pages are never recorded, and nbtree deletes a page only
  when it is "completely empty of items", so partly-emptied leaves never enter the
  map. Renamed `fsm_written_since_build`.
- Page-class replacements, from source: `VACUUM VERBOSE`'s
  `pages: N in total, N newly deleted, N currently deleted, N reusable` line is
  the only core source (`IndexBulkDeleteResult`, where nbtree documents
  `pages_free` as whole-index state), and contrib supplies the rest —
  `pgstatindex.deleted_pages` (every `P_ISDELETED` page) plus `empty_pages` (in
  fact the half-dead class), `pg_freespace` with the `avail > 0` idiom its own
  regression test uses on `freespace_btree`, and `pageinspect`'s per-page `type`
  (`d`/`D`/`e`/`l`/`i`/`r`). `pg_read_binary_file` on the `_fsm` fork is noted as a
  possible core-SQL decode and not attempted.
- Defect 2, the mixed clamp: `wasted` used `greatest(..., 0)` while `bloat_pct`
  did not. Derived from block counts already on the page, `i_var` on 17.10 printed
  `0 bytes` beside -4.4% where the signed value is -1,163,264 bytes (-142 blocks),
  and the earlier dedup-aware sweep printed `0 bytes` beside -92.5% while hiding
  -10,805,248 bytes (-1319 blocks). A negative `bloat_pct` is shown to mean only
  that a rebuild is not predicted to shrink the index: usually model
  over-prediction, but the page's own `idx_dup` grew 396 -> 426 blocks on rebuild
  under `BTREE_SINGLEVAL_FILLFACTOR`. One signed convention now applies to both
  columns and to the `ORDER BY` key, so a `LIMIT 20` no longer ties every
  over-predicted row at zero (`queries.sgml` documents that as an unpredictable
  subset). `pg_size_pretty` renders negatives symmetrically and
  `src/test/regress/expected/dbsize.out` asserts it.
- Both SQL blocks on the page were corrected in place, each with a pointer note
  saying the measurements below were taken with the defective columns; the
  Contents, Verdict, Context Reviewed, Evidence Map (13 new rows), Open Questions
  (7 new) and Source References (22 new) were updated to match.
- Root index, `wiki/v17/index.md` and the v17 coverage cell plus a new
  `## Coverage Notes` entry in `wiki/versions.md` all describe the correction.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. An
  ad-hoc bounds check under `.wiki-runtime/` confirms all 354 source citations on
  the page resolve in-range, and every one of its 71 in-page anchors resolves to a
  heading. The page keeps `verified: false` and
  `verified_by_agent: not yet`.

## [2026-08-18] follow-up v17 | rename the B-tree bloat statements' output columns to wasted_space

- Extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with a
  fifth follow-up, against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11).
- Prompt hygiene: the request ("add correction: on the sql don't use bloat as the
  output but use wasted_space") had lowercase "sql", a space before its comma, and
  telegraphic phrasing. The user chose the corrected restatement, which is what
  `## Question` now carries, with a prompt note recording the original wording and
  the three scoping answers (rename all three reporting columns, follow the new
  names in prose, rename the statement tags too).
- The rename, applied to both statements: `wasted` -> `wasted_space`,
  `bloat_pct` -> `wasted_space_pct`, `bloat_pct_floor` -> `wasted_space_pct_floor`,
  `/* wiki_btree_bloat_sweep_v17 */` -> `/* wiki_btree_wasted_space_sweep_v17 */`,
  and `/* wiki_btree_bloat_sweep_12_17 */` ->
  `/* wiki_btree_wasted_space_sweep_12_17 */`. `wasted_space_bytes` is named as the
  label for the raw `::bigint` a parsing consumer should read instead.
- Source-only: no server was started, `raw/postgres-17/` is unchanged, and no
  number on the page moves. `AS` sets an output column label and nothing else
  (`queries.sgml` Column Labels); both `ORDER BY` keys are expressions, and a label
  is usable there only as a bare sort key; and `queryid` cannot move, because
  `JumbleQuery` hashes the post-parse-analysis `Query` tree while
  `TargetEntry.resname` carries `pg_node_attr(query_jumble_ignore)`, for which
  `gen_node_support.pl` emits no `JUMBLE_STRING` into the generated
  `queryjumblefuncs.funcs.c` that `_jumbleNode` dispatches through. The longest new
  label is 22 bytes against the `NAMEDATALEN` limit of 63, where
  `downcase_truncate_identifier` -> `truncate_identifier` would raise a `NOTICE`.
- Why it is not only cosmetic: `glossary.sgml` defines bloat as "Space in data
  pages which does not contain current row versions", i.e. per-page state including
  free space in used pages, which neither statement can see — both subtract a
  modelled rebuild size from the file size, which is also why the number can go
  negative. `ref/copy.sgml` uses "wasted space" for space a maintenance command
  recovers. No SQL-visible interface in the tree says `bloat`: zero occurrences in
  `system_views.sql`, in `pg_proc.dat` and in every contrib SQL script, while
  `pgstattuple` names the quantity `free_space`/`free_percent`; the one non-prose
  occurrence is an unrelated static in `src/timezone/zic.c`.
- Consumer impact filed: reading an old name raises `ERRCODE_UNDEFINED_COLUMN`
  through `errorMissingColumn`; `pg_size_pretty` returns `text` for both its `int8`
  and `numeric` forms, so ordering by `wasted_space` is a collated `bttextcmp`
  comparison that puts `9 bytes` above `10 MB`; and because the query ID does not
  change while `pgss_store` writes the text only when the hash entry is created, an
  existing `pg_stat_statements` row keeps showing the old tag until eviction or
  `pg_stat_statements_reset()`.
- Page updates: both SQL blocks and both corrected-column snippets, the Verdict,
  and every prose mention of the three columns, with two artifacts left verbatim on
  purpose — the psql capture taken before the rename (now labelled as such) and the
  as-filed expression quotes in the reporting-defect table. `### Defect 2: wasted
  is clamped and bloat_pct is not` became `### Defect 2: the byte column is clamped
  and the percentage is not` so no heading names a column that no longer exists.
  Contents gained three entries and one renamed one; Context Reviewed, Evidence Map
  (7 new rows), Open Questions (5 new) and Source References (18 new) were updated
  to match.
- Root index, `wiki/v17/index.md` and the v17 coverage cell plus a new
  `## Coverage Notes` entry in `wiki/versions.md` all describe the rename.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. An
  ad-hoc bounds check under `.wiki-runtime/` confirms all 418 source citations on
  the page resolve in-range against the pinned checkout, and every one of its 82
  in-page anchors resolves to a heading. The page keeps `verified: false` and
  `verified_by_agent: not yet`.

## [2026-08-18] follow-up v12 | rename the core-SQL B-tree bloat statements' output to wasted_space

- Extended [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12
  (unverified)](v12/questions/indexing/btree-index-bloat-core-sql-only.md) with a
  third follow-up, against unchanged pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2). This is the v12 counterpart of
  the rename filed on the v17 page a day earlier, argued from v12 evidence only.
- Prompt hygiene: the request ("add correction: on the sql don't use bloat as the
  output but use wasted_space") had lowercase "sql", a space before its comma, and
  telegraphic phrasing. The user chose the corrected restatement, which is what
  `## Question` now carries, with a prompt note recording the original wording and
  the three scoping answers (rename both reporting columns, rename the statement
  tag and the probe index too, and follow the new names in prose plus the two
  accuracy-table headers).
- The rename: `wasted` -> `wasted_space`, `bloat_pct` -> `wasted_space_pct`,
  `/* wiki_btree_bloat_sweep */` -> `/* wiki_btree_wasted_space_sweep */`, and
  `wiki_bloat_probe` -> `wiki_wasted_space_probe` in all five sites it appears
  (the `pg_get_indexdef` replacement string, the `CREATE INDEX CONCURRENTLY`, both
  `pg_relation_size` arguments, the `DROP INDEX CONCURRENTLY`) plus the
  leftover-check sentence. Table headers `model bloat %` / `true bloat %` became
  `model wasted_space %` / `true wasted_space %`, and the one prose sentence that
  reads the percentage now names `wasted_space_pct`. Left as filed on purpose: the
  page title, conceptual "bloat" wording, and the `bloat type` fixture-recipe
  column. `wasted_space_bytes` is named as the label for the raw `::bigint` a
  parsing consumer should read instead.
- Source-only: no server was started, `raw/postgres-12/` is unchanged, and no
  number on the page moves. `AS` names a select-list entry for display and later
  reference (`queries.sgml` Column Labels); the sweep's `ORDER BY` is an
  expression, and a label is usable there only as a bare sort key; and the query ID
  cannot move because v12 computes it in contrib `pg_stat_statements`
  (`pgss_post_parse_analyze` -> `JumbleQuery`, whose rule of thumb ignores alias
  names) and its `JumbleExpr` `T_TargetEntry` case hashes `resno` and
  `ressortgroupref` only, never `resname`, while the lexer's `xc` state discards
  the `/* ... */` tag before parsing. Longest new identifier is 23 bytes against
  `NAMEDATALEN` 64, where `downcase_truncate_identifier` -> `truncate_identifier`
  would raise a `NOTICE`.
- Why it is not only cosmetic, on v12 terms: this checkout has no
  `glossary.sgml`, so the definition comes from `ref/reindex.sgml` — a bloated
  index "contains many empty or nearly-empty pages" — plus `maintenance.sgml`'s
  partly-emptied-page mechanism, both per-page state that a statement subtracting
  a modelled rebuild size from `pg_relation_size` cannot see, which is also why
  the percentage can go negative. `ref/copy.sgml` supplies "wasted space" for what
  a maintenance command recovers. All 24 case-insensitive doc-tree matches for
  `bloat` are prose; `system_views.sql`, `pg_proc.dat` and every contrib SQL
  script have zero; `pgstattuple` names the quantity
  `free_space`/`free_percent` and `approx_free_space`/`approx_free_percent`, and
  the only two contrib occurrences are C comments, one being
  `pgstatapprox.c`'s own header.
- Consumer impact filed, including one v12-specific asymmetry the v17 page does
  not have: reading an old label raises `ERRCODE_UNDEFINED_COLUMN` through
  `errorMissingColumn`; `pg_size_pretty` returns `text`, so ordering by the byte
  label is a `bttextcmp` comparison that puts `9 bytes` above `10 MB`; the tag
  survives into `log_statement`/duration lines and into `pg_stat_statements`,
  whose text is written only when the hash entry is created, so an existing row
  keeps the old tag until eviction or reset; and the probe rename, being an object
  name, does re-key the two utility statements, because `pgss_ProcessUtility`
  passes query ID 0 and `pgss_store` then hashes the statement text, while inside
  the two `SELECT`s the same name is a constant that `RecordConstLocation` and
  `generate_normalized_query` reduce to `$n`.
- Five new open questions record that nothing was executed: no server ran the
  renamed statements or the renamed Method C sequence, the `pg_stat_statements`
  consequences are derived, the vocabulary survey is a string search, and the byte
  column stays clamped at zero beside a signed percentage (`idx_dup` −6.4%,
  `idx_var` −4.6%), which the rename does not address and which v12's
  `half_rounded` — documented as rounding toward positive infinity, with no
  `dbsize` regression test in this checkout — would not render symmetrically
  anyway.
- Page updates: Contents gained three entries; the Method A block carries a
  pointer note; Context Reviewed gained a source-coverage bullet, Evidence Map ten
  rows, Open Questions five, and Source References sixteen.
- Root index, `wiki/v12/index.md`, the v12 coverage cell and a new
  `## Coverage Notes` entry in `wiki/versions.md` all describe the rename. The
  page keeps `verified: false` and `verified_by_agent: not yet`.

## [2026-08-18] follow-up v17 | five changes from a twelve-issue review of the portable B-tree wasted-space statement

- Extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with a
  sixth follow-up that works a twelve-issue external review of the portable
  12-through-17 statement, against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). Earlier follow-ups were not
  renumbered or rewritten, and the two-column contract
  (`wasted_space_pct`/`wasted_space_pct_floor`) plus the `wasted_space` naming are
  unchanged.
- Prompt hygiene: filed verbatim at the asker's direction, with two counts recorded
  in a prompt note rather than silently corrected. The parenthesis lists five items
  for "four of them are already fixed" (the asker confirmed the two reporting
  defects count as one review issue, making the ledger 4 + 1 + 5 + 2 = 12), and the
  second reporting defect became the signed `wasted_space` byte column, not
  `wasted_space_pct`, which was never defective. The asker also named the
  mischaracterized issue.
- New measurement provenance: an isolated **17.11** server built out of tree from
  the current pin (`--without-readline --without-zlib --without-icu`, `block_size`
  8192, `autovacuum = off`, `fsync = off`) — the first numbers on this page taken
  on the pinned commit rather than the previous 17.10 one — plus an isolated 12.2
  server for portability. `pgstattuple` and `pageinspect` were installed as ground
  truth only; `CREATE INDEX CONCURRENTLY` copies are the reclaimable-size arbiter.
  The filed statement, the corrected statement, a `security_invoker` copy for the
  unprivileged-role run, and a variant with only the `least()` guard removed were
  installed as views and scored in one query.
- Reproduction check first: the new server reproduces the earlier follow-up's cells
  exactly — `i_q1000` 847 modelled against 896 live and 896 rebuilt, `i_cd` 424
  against 462, `i_dupdel` 89.8% against a true 89.8%, `i_stale` 19.0% against a
  true 19.0% — which is what makes the rest of the comparison meaningful.
- Four issues confirmed already fixed, three of them by running the filed text:
  the negative-`n_distinct` branch (four fixtures read `n_distinct = -1`), the
  `indnatts = indnkeyatts` conjunct (`i_inc_lowcard` reads `dedup_applies = false`
  and 0.1%, so the 78.1% false positive of the third follow-up is gone), the two
  reporting defects, and — by text plus source, since this build has no ICU — the
  nondeterministic-collation conjunct. The `deduplicate_items` conjunct was
  measured too: `i_dupoff` reads 0.1% on 2749 blocks.
- Mischaracterization established: the page's "the last posting tuple is partial
  and pages no longer pack evenly" is the wrong mechanism for `i_q1000`. Of the
  49-block gap, 48 blocks are a posting-tuple count error (the model divided a
  class's rows by its TID count, letting one tuple span two key groups, while
  `_bt_load` flushes the pending list at every key change) and 1 block is
  internal-fanout error (pivots above a deduplicated leaf level carry a heap TID,
  measured 212 items per level-1 page against the modelled 284). Leaf packing was
  already right: a leaf holds nine 808-byte posting tuples plus a 24-byte truncated
  high key, which is the model's 9 data items.
- Change 1 applied as prescribed, and reported as not a strict improvement. It
  removes every phantom-bloat reading above 264 rows per key (`i_q1000` +5.5% to
  −0.6%, `i_r500`/`i_r1000` +5.5% to −0.2%/−0.3%, `i_r265` +10.8% to −33.5%) and
  introduces over-prediction to −99.4% at 133 rows per key and −88.7% at 143,
  because each group's last, partial posting tuple is still priced at the full
  `nmax` size; the count itself is exact (14,008 modelled against 14,000 built).
  `i_cd` does not move: at 100.1 rows per group the round-up multiplier is 1, and
  its 38 blocks are an off-by-one leaf capacity — 12 modelled posting tuples where
  `_bt_buildadd` fits 11 data items plus the high key. Both directions are confined
  to the point estimate, so the floor-based rule was immune before and after.
- Change 2 applied: an `ndistinct` extended-statistics estimate replaces the
  per-column product for a multicolumn key, read as
  `((n_distinct::text)::json ->> '<ascending attnums>')::numeric`. It is the only
  change that recovers a missed alert: `i_ext50`, a correlated `(a, b)` index with
  49.8% genuinely reclaimable, read `−38.3%` with a `−53.3%` floor and now reads
  `+49.9%`, one block from its rebuild — so the floor was *not* immune. Measured
  superset covering, wrong-column-set fallback, reversed column order, expression
  keys, and `pg_stats_ext`'s owner-membership condition (8 rows to the owner, 0 to
  an unprivileged role).
- Change 3 applied: a `statistics not visible to this role` caveat, plus a narrower
  `no statistics row for an index column`, from the view's own filter evaluated in
  SQL. It belongs in the suppression set for a reason the review did not give — the
  32-byte default `avg_width` moves the **floor**, so a healthy 100-byte-key index
  reads 62.5% on both columns to a role without `SELECT` against 0.0% as owner,
  reproduced identically on 12.2 — while the `n_distinct` half only collapses the
  point estimate onto the floor.
- Change 4 applied and measured: `i_rand`, 1,000,000 distinct keys inserted in
  random order into an existing index and never deleted, is 3765 blocks live at
  65.68% leaf density and 49.87% fragmentation with zero deleted, half-dead or
  empty pages, rebuilds to 2745, and reports 27.1% on both columns with the model
  exactly equal to the rebuild. Stated plainly as the one valid issue the
  floor-plus-status-plus-caveats rule cannot defend against.
- Change 5 applied: the baseline is a sorted build at `fillfactor`, while
  `_bt_findsplitloc` applies `fillfactor` only to rightmost leaf splits and splits
  every other leaf 50:50. Measured drift: 500,000 further random inserts took the
  never-rebuilt copy to 5590 blocks at 66.36% (26.4%), gave 10.9% of the reclaimed
  space back on the rebuilt copy (4622 blocks at 80.21%), and a fresh rebuild
  landed on the model's 4116 blocks exactly at 90.07%. A three-step decision rule
  separates "a rebuild would reclaim this" from "a rebuild is worth doing".
- Two fixes rejected with source reasons. `least(reltuples, n_live_tup)` stays:
  `n_live_tup` is DML-maintained at commit and re-based by `ANALYZE`, live rows are
  the rebuild baseline, and `reltuples` alone reports the 19%-drained `i_stale` as
  0.0% — 521 blocks and 4168 kB lost on both columns — while three other fixtures
  are bit-identical under both forms. `bt_metap().allequalimage` stays out: the
  build recomputes the flag from the catalog and only then writes the metapage, a
  pg_upgrade'd v12 index reports false until a `REINDEX`, the function raises
  `must be superuser to use pageinspect functions`, and a `deduplicate_items = off`
  index reports `allequalimage = t` on 2749 blocks holding no posting lists. The
  metapage answers "may this index deduplicate as it stands", which is the
  rebuild-decision question the pg_upgrade page uses it for.
- Portability verified by running the same file on 12.2 (`server_version_num`
  120002): every added construct exists there, `pg_stats_ext` has exactly the
  columns referenced, `n_distinct::text` renders `{"1, 2": 1000}`, and the
  unprivileged-role caveat fires the same way. `inherited` and `exprs` are
  deliberately not referenced because 12 lacks them; the stated fallback is
  `max()` over duplicate rows, which under-credits deduplication rather than
  over-crediting. On 12.2 the gate stays closed (`pg_amproc` count at
  `amprocnum = 4` is 0), so all 12 rows have `tids_per_tuple` 1.0 and every point
  estimate equals its floor; change 1 is dead code there, change 2 fills in
  `key_groups` on two indexes without moving a percentage, and change 3 does move
  percentages.
- Cost: 41.7 / 34.4 / 37.6 ms against the filed statement's 30.2 / 31.9 / 36.6 ms
  over 54 indexes and 75,847 blocks on 17.11, and 30.5 / 19.4 / 21.7 ms over 12
  indexes and 37,668 blocks on 12.2.
- Fifteen new open questions record what was not measured: the unapplied exact
  leaf-capacity rule and two-size posting-tuple mixture, the uncorrected
  internal-fanout term, the untested RLS half of the new caveat, the un-recomputed
  30%-threshold alert table, the single-seed `i_rand` result, the fixture-mutating
  drift experiment, extended statistics on plain columns only, no 13/14/15/16 run,
  and that everything above this follow-up remains a 17.10 observation.
- Page updates: Contents gained thirteen entries; the Verdict gained a paragraph;
  thirteen new `###` sections carry the ledger, the four already-fixed issues, the
  mischaracterization, the five changes, the full corrected statement, the fixture
  and duplication-band tables, the direction/magnitude/immunity table, the two
  rejected fixes, and the 12.2 run. Context Reviewed gained two bullets, Evidence
  Map thirteen rows, Open Questions fifteen, and Source References twenty-four.
- Root index, `wiki/v17/index.md`, the v17 coverage cell and a new
  `## Coverage Notes` entry in `wiki/versions.md` all describe the follow-up. The
  page keeps `verified: false` and `verified_by_agent: not yet`.

## [2026-08-18] follow-up v17 | seventeen mandatory deduplication-gate tests, and change 6

- Extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with a
  seventh follow-up: a seventeen-test mandatory suite for the deduplication gate,
  against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11).
  Earlier follow-ups were not renumbered or rewritten.
- Prompt hygiene: the request ("follow agents.md, in postgresql 17 , for question:
  ... create a section with mandatory tests to question, make sure that all
  proposed sql executes all mandatory tests and add these tests requirements for
  deduplication: ...") had a space before a comma, lowercase "postgresql"/"sql",
  "tests requirements", telegraphic phrasing and a requirement list that mixed
  fragments with imperatives. The user chose the corrected restatement, which is
  what `## Question` now carries, with a prompt note recording the original
  wording and the three scoping answers: both deduplication-aware statements are
  in scope, a statement that fails a mandatory test must be corrected rather than
  only reported, and "the original bug" is the existence-only equal-image gate.
- Verdict: the corrected 12-through-17 statement fails 3 of 17 tests and the
  earlier v17 sweep fails 6. Every failure is one defect — the gate asks whether
  an equal-image support function exists in `pg_amproc`, while `_bt_allequalimage`
  looks it up and then calls it, treating a missing OID and a false return
  identically.
- Measurement provenance: an isolated **17.11** server built out of tree from the
  current pin with `--with-icu --enable-debug`, `block_size` 8192, `autovacuum` and
  `fsync` off, in a scratch database; plus the isolated 12.2 server for test 17.
  `pageinspect` and `amcheck` installed as ground truth only. Both statements were
  installed as views generated mechanically from the page's own SQL text, so the
  scored text is the filed text plus the documented view edits.
- Fixtures: `t`, 500,000 rows with 5,000 distinct keys, 24 indexes covering every
  requirement (two deterministic-collation text indexes, one nondeterministic ICU
  index, four expression indexes whose expression type or collation differs from
  the column's, an `INCLUDE` index, `deduplicate_items = off` twins, a unique
  index, six custom-opclass indexes); and `t2`, the same shape on `(int4, int8)`
  with a `CREATE STATISTICS (ndistinct)` object so the mixed-key over-credit shows
  up as a percentage rather than only as a flag.
- Eight custom operator classes made tests 12-16 constructible in pure SQL:
  `amvalidate` returns true for SQL and PL/pgSQL support functions, because
  `btvalidate` and `assignProcTypes` check only arity, boolean return and
  non-cross-typedness. A NULL-returning function aborts `CREATE INDEX` with
  `function 16575 returned NULL`; a raising one propagates.
- Results as filed: 5 over-credits of 28 and 0 under-credits, with three fixtures
  above a 30% threshold on the point estimate and none on the floor. `i_ei_false`
  (support function returns false, 1376 blocks, nothing to reclaim) reads
  **69.3%**; `i2_tf` and `i2_ft` read **78.1%**. The v17 sweep adds `i_inc`
  (65.0%), `i_text_nondet` (77.7%) and `i_expr_lower_ci` (77.7%).
- Change 6 applied: the registered proc must be `LANGUAGE internal` with `prosrc`
  in (`btequalimage`, `btvarstrequalimage`). 0 over-credits, one deliberate
  under-credit — `i_ei_true`, genuinely deduplicated at 421 blocks, reads −226.4% —
  which is what test 16 ("never TRUE") costs against test 14 ("TRUE if safely
  determinable"). 12.2 / 12.5 / 16.6 ms against the filed 12.8 / 12.1 / 14.0 ms
  over 34,164 blocks; 10.3 ms against 14.0 ms on 12.2.
- `prosrc` beats `proname`, measured on four probes: a `LANGUAGE internal` alias in
  `public` and a renamed-away built-in break the name test as under-credits, while
  a same-named SQL impostor at a fresh OID breaks it as an over-credit. `fmgr`'s
  built-in fast path (which never reads `pg_proc`) and its `LANGUAGE internal`
  branch (which resolves by `prosrc`) explain all four.
- The metapage question is now closed by measurement, not only by source:
  rewriting `ei_true` to return false leaves `allequalimage` true, makes
  `bt_index_check` raise `metapage incorrectly indicates that deduplication is
  safe`, and a `REINDEX` takes the index from 421 to 1376 blocks — which the
  corrected reading (−226.4%) predicted to within 0.4 points.
- The honest cost is filed as its own measurement: adding a working `FUNCTION 4`
  to the opclass that had none makes a rebuild reclaim a true 69.4%, which the
  filed gate reported correctly and change 6 reports as 0.1%. Bounded by a
  stock-database census — 29 B-tree support-function-4 rows, 26 `btequalimage` plus
  3 `btvarstrequalimage`, all `LANGUAGE internal` in `pg_catalog` — so no reading
  moves where no custom opclass exists.
- Test 17 on 12.2: seven of seventeen tests are unconstructible (`invalid function
  number 4, must be between 1 and 3`, `unrecognized parameter
  "deduplicate_items"`, `ICU is not supported in this build`), no deduplication
  verdict is ever logged, and all four variants agree with the engine on all 10
  buildable fixtures. `i_multi_bad` reads 28.8% on both majors, which is an
  `avg_width` truncation to 4 on a 5-byte `numeric` rather than a gate error.
- Two runnable artifacts, both executed: a zero-fixture audit query that lists the
  indexes whose credit depends on the gate change (6 rows in the fixture database,
  0 on a stock 17.11 database, 0 on 12.2) and the fixture harness, re-run verbatim
  from the page text — 23 indexes, 11 engine "can", 11 "cannot", and no verdict at
  all for the `INCLUDE` index, because the early return precedes the debug message.
- Page updates: Contents gained twelve entries, `## Question` gained the
  seventeen-bullet follow-up and its prompt note, and twelve new `###` sections
  carry the verdict table, the per-test verdicts, the harness provenance, the
  engine walkthrough with the four DDL refusals, change 6, the
  `prosrc`-versus-`proname` evidence, both cost sections, the sweep's three
  conjuncts, the post-build mutation, the 12.2 run and the runnable harness.
  Context Reviewed gained two bullets, Evidence Map twelve rows, Open Questions
  thirteen, and Source References twenty-three.
- Root index, `wiki/v17/index.md`, the v17 coverage cell and a new
  `## Coverage Notes` entry in `wiki/versions.md` all describe the follow-up. The
  page keeps `verified: false` and `verified_by_agent: not yet`.

## [2026-08-18] follow-up v17 | a maintained pointer to the current recommended B-tree wasted-space statement

- Extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with an
  eighth follow-up: a standing `## Question` requirement plus one new section,
  [The current recommended
  statement](v17/questions/indexing/btree-index-bloat-core-sql-only.md#the-current-recommended-statement),
  against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11).
  Earlier follow-ups were not renumbered or rewritten.
- Prompt hygiene: the request ("follow agents.md, in postgresql 17 , for question:
  ... add to question the requirement to keep a section that point to the current
  recommended sql, select it based on the most acurate and with more fixes and more
  compatible") had a space before a comma, lowercase "postgresql"/"sql", "acurate",
  "a section that point to", and a trailing comparative fragment. The user chose
  the corrected restatement, which is what `## Question` now carries, with a prompt
  note recording the original wording and the three scoping answers: the section
  points at a statement already on the page rather than restating its SQL, it names
  the exact substitution needed to assemble it, and it sits directly after
  `### Verdict` rather than at the end of the page.
- The recommendation: the corrected statement with all five changes, with change
  6's `all_equalimage` subquery substituted for the existence test. Assembly is two
  stated steps — take the five-change SQL block, then replace the ten lines from
  the `-- deduplication gate` comment through `AS all_equalimage,` — so there is no
  second SQL copy to drift. The 17.11 and 12.2 runs behind the seventeen
  mandatory-test verdicts were generated exactly that way, so the assembled text is
  one the page has executed.
- Selected on the three criteria the prompt named, with a six-row ranking table.
  Accuracy: 0 over-credits over the 28 zero-waste fixtures against 5 as filed and 8
  for the earlier v17 sweep, and 0 fixtures above 30% on either percentage column
  against 3 and 4; the five changes add `i_q1000` +5.5% to −0.6%, `i_ext`/`i_sup`
  −206.4% to +0.1%, and a genuinely 49.8%-reclaimable index from an unalertable
  −38.3% to +49.9%. Fixes: six numbered changes on top of the portable statement's
  gate conjuncts and both reporting corrections, ending in the catalog form of what
  `_bt_allequalimage` does — look the support function up and call it — with
  `prosrc` as the identity `fmgr` resolves. Compatibility: 12 through 17 unchanged,
  measured on 12.2 and 17.11 for this exact text and on 14.23 and 17.10 for the
  pre-change-6 text; 13, 15 and 16 were never run.
- Four residual errors are named with their direction rather than buried: change 1's
  −99.4% band just above a multiple of the 132-TID cap, change 2's −88.6% on
  independent columns, change 6's one custom-opclass under-credit (a real 69.4%
  rebuild win reported as 0.1%), and the 27.1% a randomly inserted, never-deleted
  index reads on both columns while the model is exact to the block — the only one
  of the four that a floor-based alert cannot absorb.
- The section also fixes what the recommendation does not replace (Method C as the
  only exact arbiter, Method B's leaf census, Method A-prime's sampled width; the
  earlier v17 sweep needs its three conjuncts if it is kept), how to read the
  output (floor plus `status = 'ok'` plus the three-string suppression set), the
  `PGC_USERSET` scope of the two `SET`s the statement issues, the audit query to
  run before adopting the gate swap, and the displacement rule: state the trade
  instead of switching silently, because an over-prediction and an over-credit are
  not interchangeable.
- Source-only; no server was run. Page updates: one Contents entry, one `## Question`
  follow-up with its prompt note, one new `###` section, one Context Reviewed
  bullet, one Evidence Map row and two Open Questions (the ranking covers only this
  page's own statements and fixtures; the assembled text has two servers and one
  28-fixture set behind it, while every per-fixture number it inherits was measured
  without change 6).
- Root index, `wiki/v17/index.md`, the v17 coverage cell and a new
  `## Coverage Notes` entry in `wiki/versions.md` all describe the follow-up. The
  page keeps `verified: false` and `verified_by_agent: not yet`.

## [2026-08-18] follow-up v17 | fold change 6 into the B-tree wasted-space statement and re-measure the whole page

- Extended [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) with a
  ninth follow-up against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11): change 6 is now inside
  [The corrected statement, with all six
  changes](v17/questions/indexing/btree-index-bloat-core-sql-only.md#the-corrected-statement-with-all-six-changes),
  the section is retitled from "five changes", and **every server-measured table on
  the page was re-run on 17.11**. Nothing on the page is a 17.10 observation any
  more.
- Prompt hygiene: the request ("follow agents.md, in postgresql 17 , for question:
  ... correct and replace "The corrected statement, with all five changes" by
  adding the change 6 and re-run all tests") had a space before a comma, lowercase
  "postgresql", and "the change 6"; the asker approved the corrected restatement
  and confirmed three scoping decisions — "all tests" means every server-measured
  table, not just the mandatory suite; the section is renamed to "six changes" with
  every anchor followed; and change 6 keeps its rationale but loses its duplicate
  SQL block.
- Servers: one isolated 17.11 install built from the pin with `--with-icu
  --enable-debug` carrying four databases (the 15 v12 fixtures plus the 54-cell
  matrix and Methods A/A-prime/B/C/D; the twelve-issue-review family with its
  `probe` role; the 28 mandatory-test fixtures; a fresh database for the runnable
  harness), a second 17.11 cluster plus isolated 14.23 and 12.2 servers for the
  12-through-17 family. Every scored statement text was generated mechanically from
  the page's own Markdown, including the pre-change-6 form, so both columns of every
  comparison come from one source.
- Reproduced exactly: all 15 `pgstatindex` fixture rows, Method C's rebuilds, Method
  B's 36-of-36 census and its 18 never-vacuumed `Heap Fetches` catches, the 313.58%
  against 96.15% density blow-up, 5 gate over-credits before change 6 and 0 after
  with 1 under-credit, 8 for the earlier sweep, the audit query's 6 / 0 / 0 rows,
  the harness's 23 indexes and 11-versus-11 `DEBUG1` verdicts, and every 12.2
  blocker.
- Moved by `ANALYZE` sampling: Method A is exact on 11 of 15 fixtures rather than 9;
  the matrix scoreboard reads 33 exact rather than 30; the 10-rows-per-key sweep row
  flipped to a negative `n_distinct` and lost its credit; `i_q1000` models 899 rather
  than 901 because 8 most-common values were stored instead of 11.
- Corrected one wrong claim: [What change 6
  costs](v17/questions/indexing/btree-index-bloat-core-sql-only.md#what-change-6-costs)
  had reported the gate under-crediting an opclass that gains `ei_alias(oid)`. It
  does not — `ei_alias` is `LANGUAGE internal` with `prosrc = 'btequalimage'`, so the
  whitelist credits it and both texts report 69.3% on a true 69.4%. The cost applies
  only to a non-internal support function, re-measured with `ei_true(oid)`.
- Newly measured: the nondeterministic-collation conjunct on an ICU 17.11 build
  (`i_ci` 0.1% against the earlier sweep's 87.6% on a healthy 3611-block index),
  change 1's residual over-prediction on a two-column and a partial fixture
  (`i_inc_bothkeys` −24.8%, `i_q1000_part` −33.1%), a never-rebuilt 1.5M-row random
  twin for change 5 (25.9% against the rebuilt index's 11.5%), and interleaved
  timings that price the five earlier changes at roughly 30% while change 6 stays
  inside the noise.
- Page updates: the Contents, the verdict's follow-up list, the recommended-statement
  section and its residual-error table, the change-1 through change-5 sections, the
  six-change fixture tables, the 12.2 portability tables, the mandatory-test tables,
  the `prosrc` and mixed-key sections, one new `###` section recording the re-run,
  one Context Reviewed bullet, two Evidence Map rows, and twenty-two Open Questions
  rewritten of which three are retired as resolved (the "did not re-measure the rest
  of the page" gap, the un-measured collation conjunct, and the un-recomputed alert
  table). The page keeps `verified: false` and `verified_by_agent: not yet`.

## [2026-08-19] cleanup | removed every disposable build, install, and cluster from .wiki-runtime

- Nothing was reverted: the working tree was already clean at `ab9b295`, with no
  modified, staged, or untracked file under `git status --porcelain -uall`, so the
  user's revert request was a no-op.
- Stopped the one server still running from an earlier session, a 17.11 postmaster
  (pid 148760) launched from `.wiki-runtime/pg17/install/bin/postgres` on
  `.wiki-runtime/tmp/review17/data`, with `pg_ctl -D data -m fast stop`. It shut
  down cleanly and no `postmaster.pid` and no postgres process remained before any
  deletion began.
- Reclaimed 27 GB at the user's request; `.wiki-runtime/` went from 27 GB to 18 MB.
  Eight disposable cluster data directories accounted for 26 GB of it:
  `tmp/rerun6/data` (5.9 GB), `portable-sweep/data12` (4.9 GB),
  `portable-sweep/data1711` (3.9 GB), `tmp/review1711/data` (3.9 GB),
  `portable-sweep/data14` (3.7 GB), `pg12/data` (1.5 GB), `tmp/review17/data`
  (945 MB), and `tmp/dedup-gate-data` (623 MB).
- Removed the four out-of-tree build trees (580 MB): `tmp/dedup-gate-1711icu`
  (252 MB), `pg12/build` (183 MB), and `pg18/build` and `tmp/review1711/build`
  (73 MB each). Sources are untouched in the pinned `raw/postgres-NN/` checkouts.
- Removed all seven compiled install trees (482 MB) on the user's instruction:
  `pg17` and `pg1711icu` (91 MB each), `pg17icu` (90 MB), `pg14` (77 MB), `pg12`
  (66 MB), and `pg17_11` and `pg18` (35 MB each). No compiled install now remains,
  so the next exact-pin experiment must configure and build from
  `raw/postgres-NN/` again, including any contrib module and `--with-icu`.
- Removed the scratch SQL, harness drivers, `.out` transcripts, server logs, and
  draft Markdown fragments that went with them, because the filed pages already
  carry the results: `pg17/sql/`, all of `portable-sweep/` (`run.sh`,
  `rescore.sh`, `analyze.py`, `section_head.md`, `section_tail.md`, `out/`,
  `sql/`), and every remaining `tmp/` sandbox (`partbounds`, `bpt`, `bpt17`,
  `mandtests`, `dedup-gate`, `dedup-upgrade`, `repin-2026-08-17`). The ad-hoc
  helper `.wiki-runtime/check_page.py` went with them; durable tooling belongs
  under `scripts/`.
- One consequence to note: the pg_upgrade deduplication entry in `wiki/versions.md`
  points a re-verifier at captured output under `.wiki-runtime/tmp/dedup-upgrade/`,
  which no longer exists. That pointer is now dangling, and the run behind it must
  be redone from the pinned checkout if the numbers are ever challenged.
- Kept `venv/`, `cache/`, `indexes/`, and `logs/`, which the tooling reads and
  appends to; `scripts/wiki_lint` recreated the empty `tmp/` scaffold through
  `ensure_runtime_dirs()` on its next run.
- No wiki page, index, version pin, verification field, or `raw/` checkout changed;
  this entry is the only edit. All five pinned checkouts (`postgres-12`, `-14`,
  `-17`, `-18`, `-19`) are intact. `.wiki-runtime/venv/bin/python scripts/wiki_lint`:
  0 errors, 0 warnings.

## [2026-08-19] cleanup | pruned every wiki/log.md entry older than 2026-08-01

- Removed 262 of the 328 entries in `wiki/log.md` at the user's request, cutting the
  file from 8,579 to 3,238 lines. The user chose the calendar cutoff `2026-08-01`
  over a rolling one-month or a "before July 1" reading, so all 62 May, 131 June,
  and 69 July entries are gone and the 66 August entries survive, dated 2026-08-03
  through this one. Nothing is lost permanently: the pruned text is in git history
  through commit `db0a5c1`.
- No truncation marker was left behind, by the user's choice. The log opens on its
  `# Wiki Log` header and its append instruction, then goes straight to the oldest
  surviving entry with nothing recording that a cut happened.
- The prune ran per entry rather than as a line-range cut, because the file is not
  chronological. Entries ran newest-first from the top down to 2026-05-13, then
  oldest-first from what was line 6632 to the end, so July and August entries sat in
  both blocks. The 66 survivors keep that inherited order, which is worth knowing
  when reading `scripts/recent_log`: it returns the tail of the file, so
  `--limit 20` now walks 2026-08-19 back to 2026-08-13 and then jumps to the
  2026-08-03 entry that ends the newest-first block.
- Fixed the one dangling pointer the cut created. The `## Context Reviewed` bullet in
  `wiki/v19/questions/storage-and-vacuum/autovacuum-parallel-scoring-visibility.md`
  credited a 2026-06-10 log note with recording an earlier orphan draft of that
  question; the sentence is dropped at the user's request. The rest of the bullet is
  unchanged, and no claim, citation, `pinned_commit`, `verified`, or
  `verified_by_agent` value on that page moved.
- One in-log date reference survives on purpose: a kept 2026-08-06 entry mentions an
  answer page "deleted on 2026-06-06". It states when a page went away rather than
  pointing at a log entry, so it still reads correctly.
- Links inside `wiki/log.md` count as inbound links for the `wiki_lint` orphan check,
  so the prune could have orphaned pages that only pre-August entries linked. It did
  not. `.wiki-runtime/venv/bin/python scripts/wiki_lint` reports 0 errors and 0
  warnings, matching the pre-cut baseline, and `scripts/recent_log --limit 20` still
  parses the file.
- Nothing else changed: no other wiki page, index, version pin, or verification
  field, and no `raw/` checkout. All five pinned checkouts (`postgres-12`, `-14`,
  `-17`, `-18`, `-19`) are intact.

## [2026-08-19] cleanup | emptied .wiki-runtime of the partial-index sandbox, the 17.11 install, and all caches and logs

- Reclaimed 3.6 GB at the user's request; `.wiki-runtime/` went from 3,667 MB to
  13 MB. The user chose the widest of four offered scopes ("everything
  disposable") and, unlike the earlier cleanup today, also chose to clear the
  tooling caches and logs.
- Nothing had to be stopped. `tmp/partial17/data/postmaster.pid` still named pid
  488985 and `tmp/partial17/server.log` ended on `received smart shutdown request`
  at 09:20 EDT with no shutdown-complete line, but that process was already gone
  and no postgres, `pg_ctl`, `initdb`, or `psql` process existed before any
  deletion began. The pid file and the `.s.PGSQL.54417` socket were stale traces.
- Removed `tmp/partial17/` (3,552 MB): the cluster data directory (3,283 MB), the
  out-of-tree build of the v17 pin (270 MB), and 152 KB of harness files in 24
  pieces (`00_setup.sql` through `80_bloat.sql` with their `.out` transcripts,
  `extract_stmt.py`, `make_view.py`, `filed_stmt.sql`, `est_view.sql`,
  `initdb.log`, `server.log`).
- What that sandbox was, since nothing was filed from it: it re-ran the estimator
  statement already filed in
  [Estimating B-tree Index Bloat With Core SQL Only in PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md) against
  partial-index fixtures on a 17.11 ICU build of the pin, with `pageinspect` and
  `amcheck` installed and per-fixture output collected in a `res` table.
  `make_view.py` turned the filed text into a view `est` mechanically, dropping
  only the two leading `SET` lines, the `WHERE actual_bytes > 1024 * 1024` triage
  filter, and `LIMIT 20` — the same two harness edits the page already documents.
  The run never finished: the harness was still being authored at 09:05 (the
  server logged a syntax error on `CREATE INDEX ... WHERE active WITH (fillfactor
  = 100)`), `80_bloat.out` stops mid-script after its last `CALL`, and the newest
  wiki edit (08:49) predates the whole sandbox (08:55-09:20). No page, table,
  number, or verification field ever took anything from it, so those
  partial-index measurements must be redone from the pinned checkout if they are
  wanted.
- Removed the one compiled install, `pg1711icu/` (98 MB: `bin` 65 MB, `lib` 22 MB,
  `include` 8 MB, `share` 4 MB). It was 17.11 built `--with-icu` from the current
  v17 pin `786db8dcf168bd9df8f55047337525ac19118b1c` and it is what ran the
  sandbox. As after the earlier cleanup, no compiled install remains, so the next
  exact-pin experiment must configure and build from `raw/postgres-NN/` again,
  including any contrib module and `--with-icu`.
- Cleared the caches and logs the previous cleanup deliberately kept.
  `cache/repin_citations/` held 4.3 MB of per-version JSON written by the
  2026-08-17 repin (`v12.json` 2.0 MB, `v17.json` 978 KB, `v18.json` 724 KB,
  `v14.json` 452 KB, `v19.json` 200 KB); `scripts/repin_citations` rebuilds it
  from the checkouts, so the next repin re-reads raw source instead of a cache.
  Also gone: `cache/wiki_lint/last-run.txt` and all three tool logs
  (`wiki_lint.log` 18,879 bytes, `recent_log.log` 9,192 bytes,
  `repin_citations.log` 4,645 bytes), so tool run history now starts from this
  cleanup. The three `indexes/` subdirectories held 0 files already and were left
  alone.
- Kept `venv/` (13 MB), which every script runs from. `scripts/wiki_lint`
  recreated the rest of the scaffold through `ensure_runtime_dirs()` —
  `cache/wiki_lint`, `indexes/{ctags,search,tree-sitter}`, `logs`, `tmp` — and
  wrote a fresh `logs/wiki_lint.log` and `cache/wiki_lint/last-run.txt`.
- No new dangling pointer. Every `wiki/**/*.md` except `log.md` was scanned for
  `partial17`, `pg1711icu`, and `wiki-runtime`: no page named either deleted
  directory. The generic `.wiki-runtime/` harness pointers that pages do carry —
  including v17 and v12 `comment-stored-bytes-per-table-tuple-non-btree.md`, v17
  `btree-index-bloat-core-sql-only.md`, and the `tmp/dedup-upgrade/` pointer in
  `wiki/versions.md` — were already dangling before this cleanup and are no worse
  for it.
- No wiki page, index, version pin, verification field, or `raw/` checkout
  changed; this entry is the only edit. All five pinned checkouts are intact at
  the commits `wiki/versions.md` names (`postgres-12` `45b88269a35`,
  `postgres-14` `a92fbdfb830`, `postgres-17` `786db8dcf16`, `postgres-18`
  `baa7b142aac`, `postgres-19` `67342a14863`).
  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-19] follow-up v17 | seventy-four partial-index mandatory tests, and the twelve critical false positives they found

- Folded 74 partial-index tests into the mandatory-tests section of [Testing the
  PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), against
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). The section is
  retitled [Follow-up: ninety-one mandatory
  tests](v17/questions/indexing/btree-index-bloat-core-sql-only.md#follow-up-ninety-one-mandatory-tests):
  tests 1-17 are the deduplication-gate group, tests 18-91 the partial-index group,
  one continuous numbering as the asker chose.
- Prompt hygiene: the request ("follow agents.md, in postgresql 17 , for question:
  ... on the section with mandatory tests to question,  add these tests only for
  partial B-tree indexes.") had a space before a comma, lowercase "postgresql", a
  double space, and an ungrammatical header clause; the asker approved a corrected
  restatement of the header only. The 60 test cases, the two adversarial lists, the
  pass/fail rule and the primary requirement are the asker's own text, reflowed to
  one bullet each. Four scoping answers are recorded: fold into the existing section
  rather than add a new one; score only the recommended six-change statement; decide
  each verdict on `wasted_space_pct_floor` with `wasted_space_pct` recorded beside
  it; and read the pass/fail rule's "Estimator" as that same floor column.
- **The recommended statement fails the primary requirement.** Scored on the floor:
  52 PASS, **12 critical false positives**, 1 false positive, 8 false negatives, and
  1 cell the rule as written does not classify (test 73, 50.7% against 49.6%). The
  twelve read 84.0% to 99.6% on freshly built or freshly grown partial indexes that a
  `REINDEX` reproduces block for block, and the page's filed alerting rule suppresses
  **none** of the 21 failures — every one has `status = ok` and none carries a
  suppressing caveat. Scored on the point estimate instead it is 43 / 21 / 1 / 8 / 1.
- One structural fact explains 20 of the 22 failures: `ANALYZE` builds per-column
  statistics for an index only when the index has expressions
  (`analyze.c:448-478`, `analyze.c:861-863`), so a partial index on a plain column
  gets a row count and nothing else — measured as 0 `pg_stats` rows for the six
  plain-column partial indexes in the failing set against 3 for the expression ones —
  and the statement falls through to whole-table `avg_width`, `null_frac`,
  `n_distinct` and MCVs. Expression keys are the exception and the proof:
  `compute_index_stats` evaluates the predicate and skips every sample row that fails
  it, so `lower(name)` reported `n_distinct` 20 inside `WHERE active` against 100
  across the table.
- The defect is in the inputs, not the arithmetic. Materialising each of seven
  predicate subsets as its own table, analysing it, indexing it and pointing the
  **same statement** at the result modelled every one within 4.1 points and matched
  the partial twin's rebuilt size exactly in all seven (793/792, 641/636, 819/787,
  1575/1560, 465/464, 98/98, 22/22).
- Proved the floor column's immunity boundary and measured both sides: `n_distinct`,
  MCV and extended-statistics mismatch move only the point estimate (worst 70.6%
  against 0.0% on the floor, tests 80/81/82), because `leaf_pages_floor` contains no
  `key_groups` term; width, NULL-fraction and row-count mismatch move both (87.6%,
  90.1%, 84.1%, 92.1%, 91.2%, 94.0%, 93.5%, 99.5%, 99.6%).
- One `ANALYZE` repairs 7 of the 12 — 93.5% to 0.3%, 99.5% to 0.0%, 99.6% to 0.0%,
  80.4% to −0.4%, 86.7% to 1.1%, 92.2% to 1.6%, 98.1% to 0.9% — and **nothing in core
  SQL repairs the other 5**, which stayed at 87.9%, 90.6%, 84.1%, 91.9% and 91.2%
  after a second `ANALYZE`, because no catalog holds a plain-column partial index's
  subset width or NULL fraction and `SET (n_distinct = ...)` only moves the point
  estimate.
- Two alert-routing changes are filed with measurements and neither is applied to the
  statement: suppressing on two caveats it already emits catches 8 of the 12 at zero
  cost in true detections (every genuinely reclaimable reading above 50 has an empty
  `caveats` string), and a partial-index staleness caveat keyed on
  `n_mod_since_analyze` catches 3 more, measured as 0 against 300,000 and 399,000
  across a healthy fresh index, two stale ones and a genuinely 89%-reclaimable one.
  Only test 47, a wide INCLUDE column, resists both.
- Three incidental findings, all measured: a B-tree index datum above 510 bytes is
  pglz-compressed in place (`indextuple.c:116-133`, `TOAST_INDEX_TARGET`), so 20,000
  compressible 1001-byte keys built 142 blocks where 20,000 incompressible 481-byte
  keys built 1560; a same-session `DELETE; VACUUM` leaves `n_dead_tup` at exactly the
  delete count because VACUUM and ANALYZE write the counters absolutely while pending
  per-backend deltas add on top (`pgstat_relation.c:326-337` against `:847-867`), which
  raised a false caveat on six correct readings until the harness flushed first; and
  an append-only partial index measured 880 blocks live against 881 rebuilt, because
  the rightmost leaf splits at fillfactor (`nbtsplitloc.c:286-291`).
- Page edits: the new prompt and its note in `## Question`, a tenth-follow-up bullet
  in `### Verdict`, a partial-index exclusion plus two new residual-error rows and an
  `is_partial` term in the alerting rule in [The current recommended
  statement](v17/questions/indexing/btree-index-bloat-core-sql-only.md#the-current-recommended-statement)
  (the standing requirement to keep that section current), four renamed
  deduplication-group headings with every reference followed, ten new `###`
  subsections, 22 new Contents entries, two Context Reviewed bullets, nine Evidence
  Map rows, nine Open Questions, and 13 Source References. All 85 Contents links and
  all 215 page-internal anchors were checked to resolve.
- Measured on one isolated 17.11 server built out of tree from the pin
  `--without-readline --without-zlib --with-icu --enable-debug`, `block_size` 8192,
  `autovacuum`/`fsync` off, `maintenance_work_mem` 256MB: 74 partial indexes over 60
  tables of 200,000 to 1,000,000 rows, `REINDEX INDEX` as ground truth, the statement
  extracted mechanically from the page's own Markdown with the documented harness
  edits, and `pgstattuple`/`pageinspect`/`amcheck` as ground truth only. The sweep
  costs 35.2 ms cold then 22.6 and 21.7 ms over the 86 indexes and 41,576 blocks the
  scored run left. The server was shut down cleanly afterwards (`pg_ctl -m fast
  stop`, no `postmaster.pid` and no postgres process left); the sandbox under
  `.wiki-runtime/tmp/partial17/` is retained, 4.0 GB in a 3.7 GB data directory, a
  252 MB build tree, a 91 MB install and 116 KB of harness SQL, transcripts and the
  scoreboard. No wiki page points at that path, so removing it breaks no link.
- Nine open questions record the gaps: one statement, one server and one scale; no
  12.2 or 14.23 run for this group; neither proposed change applied and re-scored end
  to end; no threshold calibrated for `n_mod_since_analyze`; the subset probe named
  but not implemented in SQL; "0% reclaimable" argued from freshly built; the
  compression finding from one column pair; test 47 undetectable; and the borderline
  cell resolved by judgement.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is
  unchanged; the page keeps `verified: false` and `verified_by_agent: not yet`.
  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-19] follow-up v17 | changes A and B applied as a WHERE exclusion, and all 74 partial-index tests re-scored

- Applied both changes from [Two changes the partial-index tests
  justify](v17/questions/indexing/btree-index-bloat-core-sql-only.md#two-changes-the-partial-index-tests-justify)
  to the recommended statement in [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat
  Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), against
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). The asker chose a
  hard `WHERE` exclusion over a warning column, so the statement gains a
  `suppress_partial` flag in `modelled` and one `AND NOT suppress_partial` conjunct in
  the final `WHERE`: an untrustworthy partial-index reading is withheld, not annotated.
- Prompt hygiene: the request had a space before commas, lowercase "postgresql", double
  spaces, "by filter out", and a sentence opening on "and"; the asker approved a
  corrected restatement, and the prompt note records that the quoted `m.` qualifiers are
  the asker's while the page's own change-B sketch carries no alias. Four scoping answers
  are filed: hard exclusion; the trigger computed with per-table reloption overrides
  rather than from the GUCs alone; the whole 74-fixture suite rebuilt and re-scored
  rather than argued arithmetically; and `autovacuum = off` deliberately not suppressing
  the caveat.
- **12 critical false positives become 1.** Scored on `wasted_space_pct_floor`: change A
  withholds 8 (tests 30, 32, 49, 50, 78, 79, 83, 85), change B 3 more (64, 69, 84) plus
  the single plain false positive (66), and only test 47 survives — a wide `INCLUDE`
  column reading 84.0% on a 787-block index a `REINDEX` reproduces block for block, with
  an empty `caveats` string and `status = ok`. That split is exactly what the tenth
  follow-up predicted from the recorded caveat strings, test for test.
- **No true detection lost.** Tests 68, 74, 75 and 77 still report 75.0 / 74.3 / 89.1 /
  94.2 against measured 74.9 / 74.3 / 89.1 / 94.2, and the 8 false negatives are
  unchanged (4 of them now additionally withheld, which costs nothing because they read
  0.0, 0.0, 16.1 and 18.4 against 89.1, 89.1, 73.6 and 73.6 reclaims).
- Change B's threshold is the auto-analyze trigger, not `> 0`: `anl_base_thresh +
  anl_scale_factor * reltuples` with the table's reloption overriding either GUC and a
  negative `reltuples` clamped to zero, exactly as `relation_needs_vacanalyze` computes
  it (`autovacuum.c:3011-3017`, `:3063-3096`, `:3068`, `:3070-3072`; `reloptions.c`
  entries and parse table; both GUCs `PGC_SIGHUP` at 50 and 0.1). Measured trigger values
  100,050 / 50,050 / 10,050 / 100 / 1,000,000 / 60 / 50 across seven fixtures, the last
  being the `reltuples = -1` clamp after `TRUNCATE`. The `> 0` form scores these 74 tests
  identically but withheld two genuinely 89.1%-reclaimable calibration indexes, and a
  GUC-only threshold got both reloption fixtures backwards.
- The exclusion carries `is_partial` on judgement rather than measurement, so 0 of 4
  non-partial controls are touched; the price is stated — control `np97`, a freshly built
  non-partial expression index with no statistics row, reads 64.9% on 5201 blocks and is
  still reported — and the widening option is left as an open question.
- Report-level effect on the final database state (all 86 indexes freshly rebuilt, so
  every true reclaim is 0%): 82 rows returned to 46, 8 readings above the 50% alert line
  to 2, 41 indexes over the 1 MB triage filter to 28, and 9 of the pre-change top-20
  triage rows withheld, 6 of them above 50%. Cost is inside the noise: 33.4 / 38.0 / 32.8
  ms as filed against 37.5 / 33.6 / 35.2 ms amended, interleaved. No percentage moves —
  `expected_blocks` and `floor_blocks` are untouched.
- One new hazard measured: without `pg_stat_force_next_flush()` before an `ANALYZE`, a
  same-session bulk load leaves `n_mod_since_analyze` at the full 200,000 rows and
  `n_live_tup` at 400,000 for a 200,000-row table (`pgstat_relation.c:326-337` against
  `:847-867`), which fires the new caveat spuriously — in the safe direction. The
  harness rule on the page is tightened to flush before every `ANALYZE`, not only before
  `VACUUM`.
- Page edits: the new prompt and its note in `## Question`, an eleventh-follow-up bullet
  in `### Verdict`, the recommended-statement section rewritten so the carve-out is the
  statement's job rather than the reader's (blanket `NOT is_partial` dropped from the
  alerting rule, two residual-error rows replaced and two added), the statement text
  itself (two input columns, one caveat, one flag, one `WHERE` conjunct), the "Two
  changes" section marked applied with the `> 0` correction recorded, seven new `###`
  sections including a 74-row re-scored table, 8 new Contents entries, 2 Context Reviewed
  bullets, 8 Evidence Map rows, 5 rewritten/new Open Questions, and 12 Source References.
  All 239 page-internal anchors were checked to resolve.
- Measured on one isolated 17.11 server built out of tree from the pin
  `--without-readline --without-zlib --with-icu --enable-debug`, `block_size` 8192,
  `autovacuum`/`fsync` off, `maintenance_work_mem` 256MB. Three estimator texts were
  generated mechanically from the page's own Markdown — the amended text, the pre-change
  text from `git show HEAD:`, and a `> 0` copy — with one new documented harness edit
  (`AND NOT suppress_partial` removed, `suppress_partial` exposed) so a withheld index
  can still be scored; `REINDEX INDEX` is ground truth. The 74 fixtures are a
  **reconstruction** from the tenth follow-up's published shapes, because the original
  harness was deleted with its sandbox: 61 of the 74 reproduce their filed live-block
  count exactly and 13 differ (three of them because the reconstruction got the fixture
  shape wrong), none changing a verdict class. The server was shut down cleanly and its
  sandbox retained under `.wiki-runtime/tmp/partial17b/`.
- Five open questions record the gaps: the reconstruction is not the same population,
  the `is_partial` scoping is unmeasured, the borrowed trigger is not calibrated against
  a predicate subset, the two new terms were never run on 12.2 or 14.23, and test 47
  still has no detection signal.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is
  unchanged; the page keeps `verified: false` and `verified_by_agent: not yet`.
  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-20] follow-up v17 | change C: partial indexes with a variable-width INCLUDE column excluded, suite re-scored to zero critical false positives

- Applied change C to the recommended statement in [Testing the PostgreSQL 12
  Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), against
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). One more
  disjunct in `suppress_partial` — `bool_or(c.attnum > i.indnkeyatts AND c.attlen <
  0)` added to the `statvis` CTE with the `idx` join it needs, carried through
  `tuple` — so a partial index with a variable-width `INCLUDE` column returns no
  row.
- Prompt hygiene: the request had lowercase "postgresql", a space before a comma,
  two double spaces, and "exclude Partial B-tree with a wide INCLUDE column"
  missing its noun; the asker approved a corrected restatement. Four scoping
  answers are filed in the prompt note: narrow the term to variable-width non-key
  columns rather than any `INCLUDE` column, rebuild and re-score the whole
  74-fixture suite, emit **no caveat**, and read "wide" as a property of the type
  rather than a byte threshold so no tunable constant enters the statement.
- **1 critical false positive becomes 0.** Scored on `wasted_space_pct_floor` over
  tests 18-91: PASS 65 -> 66 (34 reported, 32 withheld), CRITICAL FALSE POSITIVE 1
  -> 0, FALSE POSITIVE 0 -> 0, FALSE NEGATIVE 8 -> 8 with the same 4-reported /
  4-withheld split. Exactly one test changes state, 47; the four true detections
  still report at 74.7 / 74.3 / 89.1 / 94.2 against measured 74.9 / 74.3 / 89.1 /
  94.2.
- **The run reproduces the eleventh follow-up cell for cell**: 74 of 74 live-block
  counts match the filed table and all 74 pre-change reported/withheld verdicts
  agree. Seven floors moved by more than a point from `ANALYZE` sampling (72, 73,
  86, 87, 89, 90, 91), none across a verdict boundary.
- The defect is not repairable in core SQL and the source says why: a non-key
  column can never be an expression (`create_index.sgml:185-188`), `ANALYZE`
  examines only expression attributes when building index statistics
  (`analyze.c:450-478`), an index's non-key attribute copies `attlen` from the
  table (`index.c:336-360`), and `attlen` is `pg_type.typlen`, negative exactly for
  variable-length types. Measured on test 47: `attlen` −1, 0 `pg_stats` rows for
  the index, table-wide `avg_width` 13 against a subset mean `pg_column_size` of
  207, a modelled 36-byte slot and 126 blocks against 787 live, `bt_page_items`
  mean item length 217.7 and `pgstatindex` `avg_leaf_density` 89.71% with zero
  fragmentation. One `ANALYZE` moves the reading only from 84.0% to 83.6%. A
  purpose-built partial expression index with an `INCLUDE` column got exactly one
  `pg_stats` row, for the expression, and none for the payload.
- Both wider forms were generated from the page's own Markdown by substituting the
  `bool_or` argument and scored beside the filed one: any `INCLUDE` column withholds
  37 of the 74 rather than 36 and costs test 46's correct 0.0%; any variable-width
  column withholds 41, costs tests 31, 33 and 52, and is the only form that catches
  the wide-**key** fixture `i103`. None of the three loses a true detection. The
  trade is recorded, not taken.
- The price is measured and stated three ways: fixture 100, a genuinely
  89.9%-reclaimable partial index with a `text` payload estimated at 89.5%, is now
  withheld — the first correct answer any exclusion on this page throws away;
  fixtures 101 and 104 lose correct readings of −1.3% and −709.2%; and the term
  emits no caveat, so the flag and the caveat list are no longer one-to-one and a
  reader cannot tell that this silence never lifts.
- Report-level effect over the final 88-index, 54,351-block database: 52 rows to
  47 (33 to 29 over the 1 MB triage filter), 4 readings above the 50% line to 2, 36
  partial indexes withheld to 41, and 0 of 5 non-partial indexes touched —
  including `i102`, which carries a variable-width `INCLUDE` column and reads a
  correct −4.0%. Cost is inside the noise over eight interleaved pairs: 38.5-41.3
  ms as filed against 39.0-42.3 ms amended.
- Change C is the only exclusion term run on a server older than 17. A new
  isolated 12.2 build from this repo's 12 pin builds the same fixtures at the same
  sizes (787 / 388 / 276 blocks), reports 84.1% on the `INCLUDE` fixture under the
  pre-change text and withholds it under the amended one; it also reproduced change
  B's counter hazard from the other side, since 12 has no
  `pg_stat_force_next_flush()` and the collector had to settle before
  `n_mod_since_analyze` fell to 0.
- Page edits: the new prompt and its note in `## Question`, a twelfth-follow-up
  paragraph in `### Verdict`, the recommended-statement section (carve-out
  paragraph, compatibility bullet, two new residual-error rows, and the reading
  rule's fourth term), the statement text itself, a current-state note under the
  ninety-one-tests summary, forward pointers from two eleventh-follow-up tables,
  seven new `###` sections, 7 new Contents entries, 2 Context Reviewed bullets, 5
  Evidence Map rows, 6 rewritten or new Open Questions, and 7 Source References.
  All 265 page-internal anchors were checked to resolve.
- Measured by restarting the eleventh follow-up's isolated 17.11 cluster
  (`.wiki-runtime/tmp/partial17b/data`, from the `--with-icu --enable-debug` install
  of the pin) with a **fresh scratch database**, re-running its unchanged fixture
  scripts plus six new `INCLUDE` fixtures numbered 100-105, and installing four
  estimator views generated mechanically from the page's Markdown; `REINDEX INDEX`
  is ground truth and `pageinspect`/`pgstattuple`/`amcheck` are ground truth only.
  Both exact statement texts were also run as filed, on both servers. Both servers
  were shut down cleanly; the new sandbox is retained under
  `.wiki-runtime/tmp/partial17c/` (harness SQL, the four view texts, `res.tsv`, and
  the 12.2 build, install and data directory).
- Six open questions record the gaps: the discarded true detection and the absence
  of any signal that would separate it, the wide-**key** family the term does not
  close and the measured-but-unapplied wider form, the caveat-free silence, the
  six new fixtures being one shape each with only `text` payloads, and changes A
  and B still being 17.11-only.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is
  unchanged; the page keeps `verified: false` and `verified_by_agent: not yet`.
  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-20] follow-up v17 | change D: non-partial expression indexes with no statistics row excluded, and the flag renamed suppress_row

- Applied change D to the recommended statement in [Testing the PostgreSQL 12
  Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), against
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). It is the first
  exclusion term that reaches **non-partial** indexes, so the flag is renamed
  `suppress_partial` -> `suppress_row`: one new input column in `idx`
  (`has_expressions`, i.e. `x.indexprs IS NOT NULL`) and one disjunct — `NOT
  is_partial AND has_expressions AND any_no_stats AND NOT any_stats_hidden AND
  last_analyze IS NOT NULL` — so a non-partial expression index with no statistics
  row of its own returns no row.
- Prompt hygiene: the request had lowercase "postgresql", a space before two
  commas, three double spaces, and "Non-partial expression B-tree" missing its
  noun; the asker approved a corrected restatement. Four scoping answers are filed
  in the prompt note: a hard `WHERE` exclusion with the flag renamed, the term
  narrowed to indexes that actually carry expressions rather than any index missing
  a statistics row, the whole 74-fixture suite rebuilt and re-scored, and a 12.2 run
  so change D matches change C in having been executed on a server older than 17.
- **Two critical false positives go and no partial-index verdict moves.** Over tests
  18-91 the two texts score identically — PASS 66 (34 reported, 32 withheld),
  CRITICAL FALSE POSITIVE 0, FALSE POSITIVE 0, FALSE NEGATIVE 8 (4 reported, 4
  withheld) — because the disjunct carries `NOT is_partial`. Exactly four rows differ
  between the texts over the 95 indexes in the database, all non-partial: `np97`
  64.9% on 5201 blocks a `REINDEX` reproduces exactly and a mixed `(k, upper(s))` key
  60.5% on 5477, both withheld; a −614.9% over-prediction withheld at no cost; and
  one true detection lost.
- **The run reproduces the eleventh and twelfth follow-ups cell for cell**: 74 of 74
  live-block counts match the filed table for the third consecutive run, and all 74
  pre-change reported/withheld states agree, test 47 included. Eight floors moved by
  more than a point from `ANALYZE` sampling (72, 73, 86, 87, 88, 89, 90, 91), none
  across a verdict boundary; test 73 is the borderline cell again at 50.4 against
  49.6.
- The source says why the shape cannot be priced: `ANALYZE` builds per-column
  statistics for an index only for its expression attributes (`analyze.c:450-478`)
  and writes them under the index's own relid (`analyze.c:588-602`), and `index_drop`
  states the same equivalence from the other end, removing an index's statistics only
  when `pg_index.indexprs` is non-null (`index.c:2341-2363`, with the comment "if it
  has any expression columns, we might have stored statistics about them");
  `indexprs` is documented as null exactly when every attribute is a simple reference
  (`pg_index.h:57-59`, `catalogs.sgml:4553-4564`). Measured on `np97`: 0 `pg_stats`
  rows for the index against 3 for the table, `slot` 44 and 1825 modelled blocks
  against 5201 live, `bt_page_items` item length 120.0 on the identical never-analysed
  twin, and `pgstatindex` `avg_leaf_density` 91.31% with zero fragmentation — denser
  than the fillfactor-90 build the model predicts.
- **Unlike change C, the silence lifts.** One `ANALYZE` takes `np97` from a withheld
  66.4% to a reported `−16.6%` (one `pg_stats` row, `avg_width` 59) and `x106` from a
  withheld 64.6% to `−7.3%`. The identical-DDL pair 106/107 prices the whole trade: the
  same index that a `REINDEX` shrinks 5201 blocks to 523 reads 96.4% with no statistics
  row and 89.2% with one, against a measured 89.9% reclaim.
- Two wider forms were generated from the page's own Markdown by substituting only the
  new disjunct and scored beside the filed one. Dropping `has_expressions` also catches
  fixture `x109`, a plain index whose column carries `SET STATISTICS 0` — the one
  alertable hole change D leaves, 64.9% on a healthy 5201-block index, and the only way
  a plain column loses its row while its table keeps one (`analyze.c:1015-1030`).
  Dropping `last_analyze IS NOT NULL` also catches `x108`, an expression index on a
  never-analysed table the alerting rule already suppresses through `never analyzed`.
  Neither was applied. An unprivileged role reading through a `security_invoker` copy is
  untouched: `any_stats_hidden` is true, so the term does not fire and the row is
  returned with `statistics not visible to this role`.
- Report-level effect over 95 B-tree indexes and 73,867 blocks, 84 partial and 11
  non-partial: 53 rows to 49, 35 to 31 over the 1 MB triage filter, 6 readings above the
  50% line to 3, 42 withheld to 46, and 0 of 11 non-partial indexes withheld to 4. Three
  of the six above-50 rows are fixtures built for this follow-up, which the page says
  plainly. Cost is inside the noise over eight interleaved pairs: 42.0-48.2 ms as filed
  against 38.7-47.6 ms amended; the term adds no CTE and no join.
- Change D is the second exclusion term executed on a server older than 17. The 12.2
  server (`server_version_num` 120002) builds the same fixtures at the same sizes (5201 /
  5201 / 5201 / 825), reports `np97` at 64.9% under the pre-change text and withholds it
  under the amended one, and leaves `x107`, `x109` and `np96` untouched. Its counters
  reproduced the known 12-era artifact — `n_live_tup` 600,000 for a 300,000-row table
  after `CREATE TABLE AS` plus `ANALYZE`, since 12 has no `pg_stat_force_next_flush()`.
- Page edits: the new prompt and its note in `## Question`, a thirteenth-follow-up
  paragraph in `### Verdict`, the recommended-statement section (the carve-out paragraph
  split so one paragraph covers the non-partial term, a rewritten reading rule naming all
  five conditions, the accuracy and compatibility bullets, one rewritten and one new
  residual-error row), the statement text itself, a current-state note under the
  ninety-one-tests summary, a superseded-in-part note on [Why the exclusion carries
  is_partial](v17/questions/indexing/btree-index-bloat-core-sql-only.md#why-the-exclusion-carries-is_partial),
  six new `###` sections, 6 new Contents entries, 2 Context Reviewed bullets, 6 Evidence
  Map rows, 4 rewritten or new Open Questions, and 10 Source References. All 292
  page-internal anchors were checked to resolve.
- Measured by restarting the twelfth follow-up's isolated 17.11 cluster
  (`.wiki-runtime/tmp/partial17b/data`, from the `--with-icu --enable-debug` install of
  the pin) with a **fresh scratch database**, re-running its unchanged fixture scripts
  plus seven new non-partial fixtures numbered 106-112, and installing five estimator
  views generated mechanically from the page's Markdown (amended, pre-change from `git
  show HEAD:`, two rejected variants, and a `security_invoker` copy); `REINDEX INDEX` is
  ground truth and `pageinspect`/`pgstattuple` are ground truth only. Both exact statement
  texts were also run as filed, on both servers. Both servers were shut down cleanly; the
  new sandbox is retained under `.wiki-runtime/tmp/partial17d/`.
- Four open questions record the gaps: the discarded true detection and the absence of a
  marker that would report it instead, the `SET STATISTICS 0` residual and the unapplied
  wider form, the seven new fixtures being one shape each with only `text` expressions and
  the above-50 count being fixture-inflated, and changes A and B still being 17.11-only.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is
  unchanged; the page keeps `verified: false` and `verified_by_agent: not yet`.
  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-20] follow-up v17 | recommended B-tree statement rebuilt for readability, and all 91 mandatory tests plus fixtures 92-112 rerun on 12.2 and 17.11

- Rebuilt the SQL block in [The corrected statement, with all six
  changes](v17/questions/indexing/btree-index-bloat-core-sql-only.md#the-corrected-statement-with-all-six-changes)
  for readability and maintainability, against unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). It is a **pure refactor**: still
  sixteen CTEs, but `sized`/`fit`/`posting` collapse into one `page` CTE of three commented
  laterals, while a new `env` and a new `gate` take the server constants and the whole
  deduplication decision out of `idx`, which drops from 77 lines to 62. Three
  statistics-state facts — `stats_row_missing`, `dedup_credited`, `stats_stale` — are named
  once in `modelled` and read by both the `WHERE` clause and the caveat list, so the drift
  the eleventh follow-up's comment warned about is now structurally impossible. A 35-line
  header lists the stages and maps each named change 1-6 and A-D to the stage carrying it.
  Text grows 357 -> 433 lines: +17 code, +58 comment.
- Prompt hygiene: the request had "agents.md" for AGENTS.md, lowercase "postgresql",
  "reability", "maintenanbility", a space before two commas and three double spaces; the
  asker approved a corrected restatement. Three scoping answers are filed in the prompt
  note: a pure refactor with byte-identical output as the hard requirement, "all tests"
  meaning the 91 mandatory tests plus fixtures 92-112, and the rebuilt SQL replacing the
  block in place so the recommended-statement pointer keeps naming the same section.
- **Every arithmetic expression is byte-identical and moved rather than rewritten**, because
  the model mixes `numeric` and `float8` and a re-association could move a percentage
  invisibly. Every CTE and output-column name the rest of the page cites is unchanged; the
  only prose that went stale is the opening sentence of `### Change 6`, which named `idx` as
  the CTE holding the gate and now names `gate`.
- **Identity is measured four ways on two servers, not asserted.** `SELECT * FROM est EXCEPT
  SELECT * FROM est_filed` and its reverse return **0 rows over 95 indexes on 17.11 and 92
  on 12.2**, across every exposed column including internals (`slot`, `leaf_cap`, `nmax`,
  `expected_blocks`, `floor_blocks`, `key_groups`, `tids`, `suppress_row`); both exact texts
  produce byte-identical `psql` output on both servers; and **0 of 129 and 0 of 106 scored
  fixtures** disagree on status, either percentage, caveats or the suppression flag. Both
  views come from one generator, one reading the working copy of the page and the other
  `git show HEAD:` of it.
- **The rerun scores identically to the filed tables.** Deduplication gate: 30 fixtures,
  421/460/1376/1931 blocks and `−0.2`/`8.0`/`0.1`/`0.2`% exactly as filed, `28.8%` on
  `i_multi_bad`, `−320.0%` on `i_multi_ok` against `8.1%` on `i2_ok`, 0 over-credits and the
  single `i_ei_true` under-credit at `−226.4%`, 0 fixtures above 30% on either column, and
  the engine's `DEBUG1` verdicts 14 "can safely" / 13 "cannot" / nothing for `i_inc`. Test
  16 reproduces in full: `function 17076 returned NULL`, `equalimage probe exploded`, the
  three `ALTER OPERATOR FAMILY` refusals, the `text_pattern_ops` refusal, `i_squat` at 1931
  blocks with the gate false and `i_text_det2` at 460 with it true during the same
  `pg_catalog.btvarstrequalimage` rename (restored afterwards), and the audit query at 6
  rows in the fixture database against 0 in a stock one. Partial indexes: **66 PASS (34
  reported, 32 withheld), 0 critical false positives, 0 false positives, 8 false negatives**,
  **74 of 74 live-block counts matching for the fourth consecutive run**, all 74
  reported/withheld states agreeing, and six floors moved by `ANALYZE` sampling (86-91) with
  none across a verdict boundary.
- **The cost is stated rather than hidden.** The inlined `gate` reads the materialized `idx`
  CTE once more — 5 `CTE Scan on idx` nodes against 6 — worth `+1.3 ms` of planning
  (7.87 -> 9.17 ms) and `+0.8 ms` of execution (36.17 -> 37.01 ms) on the 123-index fixture
  database, and `+3.2%` of execution on a 600-index one (192.33 -> 198.49 ms). End to end
  over 24 interleaved runs each: filed 41.8/43.5/51.5 ms min/median/max against rebuilt
  44.7/46.2/73.0.
- **Running the whole partial suite on a 12 server for the first time found the exclusions'
  portability limit.** `dedup_applies` is false for all 106 indexes on 12.2, so change A's
  duplicates disjunct — 25 of the 36 withheld rows on 17.11 — cannot fire, and **tests 30
  and 64 return as unsuppressed critical false positives at 87.6% and 93.5%** with
  `status = ok` and an empty caveat string on indexes a `REINDEX` reproduces block for
  block. Three of the five 12.2 critical false positives carry `never analyzed`, which the
  reading rule already discards. 12 has no `pg_stat_force_next_flush()`, so 17 fixtures were
  scored with `last_analyze` unset and change B fired 19 times against 7 on 17.11; that half
  of the swing measures the harness, not the statement. Three fixtures are unconstructible
  on 12.2 (test 38's `deduplicate_items`, tests 51-52's ICU), and the ten constructible
  deduplication-gate fixtures reproduce their filed 12.2 table including `i_int4` at 1376
  blocks against 421 on 17.11 and the four refusals verbatim.
- Page edits: the new prompt and its note in `## Question`, a fourteenth-follow-up paragraph
  in `### Verdict`, a rebuild paragraph in `### The current recommended statement`, a
  fourteenth paragraph above the SQL block, the rebuilt SQL itself, the corrected `Change 6`
  sentence, an updated most-compatible bullet, six new `###` sections, 6 new Contents
  entries, 2 Context Reviewed bullets, 5 Evidence Map rows, 5 new Open Questions, and 9
  Source References. All 318 page-internal anchors were checked to resolve.
- Measured on **two newly initialised clusters** rather than restarted ones: 17.11 from the
  `--with-icu --enable-debug` install of the pin, and 12.2 from this repo's pinned checkout
  at `45b88269a353ad93744772791feb6d01bc7e1e42`, both `block_size` 8192, `autovacuum = off`,
  `fsync = off`. Fixture scripts for tests 18-112 are the eleventh to thirteenth follow-ups'
  unchanged; the deduplication-gate fixtures were rebuilt from the page's own runnable
  harness; the 12.2 variants were produced by a script that drops only
  `pg_stat_force_next_flush()`, the `deduplicate_items` reloption and the two ICU
  collations, and prints every dropped line. `REINDEX INDEX` is ground truth;
  `pageinspect`/`pgstattuple`/`amcheck` are ground truth only on 17.11 and 12.2 has no
  contrib extension at all. Both servers were shut down cleanly; the sandbox is retained
  under `.wiki-runtime/tmp/refactor17/`, including both 9 GB data directories, so the run
  can be repeated without rebuilding the fixtures.
- Five open questions record the gaps: identity is proven at `block_size` 8192 only and the
  rebuilt text has not been run on 14.23, the two new 12.2 false positives have no proposed
  term, the 12.2 verdict distribution partly measures the harness, the readability claim
  itself is a judgement no second reader has checked, and the cost figures are one machine.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is unchanged;
  the page keeps `verified: false` and `verified_by_agent: not yet`.

  `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-20] answer v17 | B-tree bloat and wasted space from pgstatindex alone, on 12 and 17

- Filed [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and
  17 (unverified)](v17/questions/indexing/btree-bloat-with-pgstatindex.md) against
  unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). Filed under
  `indexing`, because the subject is B-tree page density and rebuild sizing even
  though a statistics interface supplies the numbers.
- Prompt hygiene: the request had lowercase `postgresql` and `sql`, a space before a
  comma, a double space, `that using` for `that uses`, `btree` for `B-tree`, and two
  comma splices; the asker approved a corrected restatement, which `## Question`
  carries with a note. Four scoping answers are recorded there: `pgstatindex` is the
  only measurement function while catalogs may be read to select and label indexes,
  one text must run unchanged on both majors, both servers must actually be run and
  scored against `REINDEX INDEX`, and it is a new page rather than a follow-up on the
  deliberately no-contrib
  [core-SQL bloat page](v17/questions/indexing/btree-index-bloat-core-sql-only.md).
- **The deliverable is one 125-line statement, eight stages, tagged
  `wiki_btree_bloat_pgstatindex_12_17`.** It reports two percentages on purpose:
  `wasted_pct`, free bytes in live leaf pages plus every empty and deleted page against
  perfect packing, and `est_reclaimable_pct`, the same file against a rebuild at its own
  fillfactor. The rebuild target comes from the build code, not the reloption — a sorted
  build closes a leaf when free space drops below `BLCKSZ * (100 - fillfactor) / 100`
  (`nbtsort.c:645-671`, `nbtree.h:1138-1145`), so at 8192/90 the reachable density is
  89.95% and the estimate is conservative by construction. The two columns disagree
  correctly: a fresh `fillfactor = 10` index reads **89.6% wasted, −0.5% reclaimable**,
  and an index whose leaves are 89.94% dense is **69.9% reclaimable** on 1,918 empty or
  deleted pages that `avg_leaf_density` cannot see (`pgstatindex.c:304-324`).
- **Every candidate filter guards a shape `pgstatindex` raises on**, and one raised call
  aborts the whole report. Reproduced on both servers: the five non-B-tree access methods
  and a partitioned index (`relation "..." is not a btree index`), another session's temp
  index (a 4.4 MB one moved the candidate count 27 -> 28 on 17.11 and 26 -> 27 on 12.2),
  a stale OID (`could not open relation with OID 2147483647`), and **the one behavioural
  difference between the majors**: 17.11 refuses an invalid index with `index "..." is
  not valid` while 12.2 returns a full row for the same fixture (version 4, 820 leaves,
  90.05%). The check is `13503eb5905` (2023-10-30), whose earliest containing release tag
  in this checkout is `REL_17_0`; `git tag --contains` lists no 12-16 tag, and the same
  command on an older commit does list 28 `REL_12_*` tags, so it was not back-patched.
- **Scored against `REINDEX INDEX` on two newly initialised isolated servers**, 12.2 built
  from this repo's v12 pin with `contrib/pgstattuple` compiled through PGXS from a copy of
  that checkout (never into `raw/`), and 17.11 from the `--with-icu --enable-debug` install
  of this pin. One shared 24-shape fixture script plus one 17-only `deduplicate_items = off`
  fixture: **91 of 94 and 89 of 93 scored indexes within 1.0 point**, 92 and 90 within 2.0,
  and the identical `+1.7` worst over-estimate on both (a 456 kB TOAST primary key; every
  other row over-estimates by at most `+0.1`). **24 of the report's rows are byte-identical
  across the two servers**; the three that differ are the two duplicate-key indexes
  (21 MB against 6800 kB, 20 MB against 6368 kB — deduplication changes the input, not the
  arithmetic, and the estimator is right on both) and a non-deterministic churn fixture.
- Three under-estimates are documented rather than hidden: entries deleted but not vacuumed
  (`−0.1%` against 89.9% actual, both servers), an index whose rebuild would deduplicate
  (`−0.3%` against 69.1%, 17-only because 12.2 rejects the reloption), and 400-byte keys
  (69.9% against 75.0%, because wide tuples close a page fuller than the model's bound and
  the rebuilt index measured 92.77% density). All three are in the safe direction.
- Also measured on both servers: `index_size` equal to `pg_relation_size` for **218 of 218
  and 212 of 212** candidates; `NaN` density for an index with no leaf pages, with
  `NaN > 20` true for `float8` **and** `numeric`, and all 62 and 59 `NaN` rows scoring
  exactly 0.0 through the statement's `leaf_pages > 0` guard; leaf capacity implied at
  **8151.6 and 8152.1** from a one-tuple page and an empty leaf page against the hard-coded
  `8192 - 24 - 16`; fresh-build densities 99.82 / 90.00 / 49.81 / 9.62 at fillfactor
  100/90/50/10, identical on both, against a model predicting 100.00 / 89.95 / 49.75 / 9.57;
  a `pg_stat_scan_tables` member reading every row while the same role's
  `pgstatindex('bl.i_del90')` by **name** fails with `permission denied for schema bl`,
  which is why the statement passes `c.idx_oid::regclass`; ~830 MB read per run through a
  256 kB `BAS_BULKREAD` ring (106,351 and 105,413 buffers at the function scan, 124.2 ms and
  132.6 ms execution); `lock_timeout` cancelling at 2000.9 ms and 2001.0 ms behind an
  uncommitted `DROP INDEX`; and a drop committed three seconds into a run aborting the whole
  statement.
- The scoring view was generated mechanically from the page's own statement text with two
  printed edits (the two `SET` lines dropped, `min_index_bytes` set to 0), and the filed
  Markdown block was verified byte-identical to the executed file (6,002 bytes, 125 lines).
  All 105 page links were checked to resolve, and every source citation's line range was
  checked to fall inside its file and to start and end on the intended lines.
- Eight open questions record the gaps: no standby run (the unlogged-in-recovery filter is
  reasoning, not measurement), only two minor versions, one block size, an untested
  internal-page term, an unexplained `+1.7`, two silent under-estimates with no in-statement
  warning, a non-deterministic churn fixture, and no roll-up for partitioned tables.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is unchanged;
  the page keeps `verified: false` and `verified_by_agent: not yet`. Both servers were shut
  down cleanly; the sandbox is retained under `.wiki-runtime/tmp/pgsi/`.

## [2026-08-20] follow-up v17 | alert_pct and the status column removed from the pgstatindex bloat report

- Removed the threshold and the verdict column from the statement in
  [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17
  (unverified)](v17/questions/indexing/btree-bloat-with-pgstatindex.md), against unchanged
  pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). Five edits: the
  `20::numeric AS alert_pct` entry in `params`, the `p.alert_pct` pass-through in
  `modelled`, the `CASE ... >= f.alert_pct THEN 'rebuild candidate' ELSE 'ok' END AS
  status` column, the header comment that told the reader to alert, and the stage-list
  line that called `params` a set of tunables. The text drops from 125 lines and 6,002
  bytes to 122 lines and 5,839 bytes, and the output from 15 columns to 14. `notes` is
  untouched.
- Prompt hygiene: the request had `agents.md` for AGENTS.md, lowercase `postgresql`, three
  spaces before commas, and a comma splice; the asker approved a corrected restatement,
  which `## Question` carries as a follow-up with a note. Three scoping answers are filed
  there: `status` goes with `alert_pct` because the parameter existed only to drive it,
  `notes` stays exactly as it was, and both retained servers are restarted and re-run
  rather than reasoned about.
- **The fixtures had to be rebuilt before anything could be compared.** The accuracy pass
  that produced the page's scoring tables ends by running `REINDEX INDEX` over every index
  it scores, so the retained databases no longer reproduced the filed report — the first
  run came back with 26 rows, all `ok`. After re-running both fixture scripts the filed
  text returned the archived 17.11 table **character for character**, and the 12.2 one as
  the same 27 rows with two swapped: `i_expr` and `i_text_del` are both 27 MB reclaimable
  at 79.4%, and `ORDER BY index_size - est_rebuilt_bytes DESC` has no tie-break. That is a
  property of the filed statement and is now an open question.
- **Identity is measured two ways on both servers, not asserted.** At the presentation
  level, the amended output equals the filed output with field 14 cut, byte for byte
  (2,448 bytes over 27 rows on 12.2, 2,531 over 28 on 17.11). At the internal level, one
  view per text over the `final` stage — generated mechanically from each text with the
  two `SET` lines dropped and `min_index_bytes` set to 0 — exposes **29 columns against 28
  with `alert_pct` the only loss**, and `EXCEPT` in both directions over the 28 shared
  columns returns **0 rows across 214 indexes on 12.2 and 220 on 17.11**. The amended text
  itself was derived by a script that asserts each of the five edits appears exactly once
  and prints it, and the page's SQL block was verified byte-identical to the executed file
  (5,839 bytes, 122 lines).
- **Cost is unchanged**, as a removed `CASE` over an already-computed expression should be:
  the same plan shape on both servers (4 `CTE Scan` nodes; 68 plan lines on 17.11, 60 on
  12.2), `EXPLAIN (ANALYZE, BUFFERS)` execution of 136.3 against 135.7 ms on 17.11 and
  134.7 against 127.9 ms on 12.2, and six interleaved end-to-end runs of each text
  spanning 131.5-143.8 against 132.2-137.6 ms on 17.11 and 132.6-140.2 against
  128.7-139.4 ms on 12.2.
- What the reader loses is the label, not the ranking: on this suite the 15 rows per server
  the old column called `rebuild candidate` are exactly the first 15 rows of the amended
  output, in the same order. The page says plainly that this is a coincidence of the
  fixtures — the sort is on reclaimable bytes and the label was on reclaimable percent, so
  a small badly bloated index can sort below a large healthy one.
- Page edits: the follow-up prompt and its note in `## Question`, the five SQL edits, a
  rewritten opening to `### How to read the output` and the `status` row deleted from its
  column table, one new `### Follow-up: no threshold, no verdict column` section, a new
  paragraph in `### How this was measured`, 1 Contents entry, 1 Context Reviewed bullet
  citing the ten-column result tuple `pgstatindex` builds, 2 Evidence Map rows, 1 rewritten
  and 2 new Open Questions, and 1 Source Reference. All 107 citations were re-checked to
  resolve and to fall inside their files, all 28 page-internal anchors to resolve, and the
  Contents list to match the page's 25 sections in order.
- Measured by restarting the two clusters retained under `.wiki-runtime/tmp/pgsi/` — 12.2
  built from this repo's v12 pin with `contrib/pgstattuple` compiled through PGXS, and
  17.11 from the `--with-icu --enable-debug` install of this pin — both at `block_size`
  8192, `autovacuum = off`, `fsync = off`. `pgstatindex` remains the only measurement
  function; no `pageinspect`, no `pgstattuple()`, no `amcheck`. Both servers were shut down
  cleanly and the sandbox is retained.
- Three open questions were added or rewritten: no replacement alerting rule was measured
  and the removed 20% was never derived from anything but convention, the `ORDER BY` has no
  tie-break, and the `i_novac`/`i_dedup_off` bullet now names the near-zero estimates
  (`−0.1%` and `−0.3%`) instead of the deleted `status = ok`.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is unchanged;
  the page keeps `verified: false` and `verified_by_agent: not yet`.

## [2026-08-20] follow-up v17 | wasted space rebased on the index fillfactor in the pgstatindex bloat report

- Rebased every wasted-space calculation on the index's own fillfactor in
  [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17
  (unverified)](v17/questions/indexing/btree-bloat-with-pgstatindex.md), against unchanged
  pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). Four edits: the `est` CTE's
  `leaf_bytes - live_leaf_bytes + dead_bytes` becomes
  `GREATEST(round(leaf_bytes * target_density) - live_leaf_bytes, 0) + dead_bytes`, the two
  presentation columns become `wasted_vs_fillfactor` and `wasted_ff_pct`, and the header
  comment plus the `sized` stage line state the new baseline. 122 lines and 5,839 bytes to
  126 and 6,154; still 14 output columns; `notes` untouched.
- Prompt hygiene: the request had `agents.md` for AGENTS.md, lowercase `postgresql`, two
  spaces before commas, and unhyphenated `wasted space related`; the asker approved a
  corrected restatement, which `## Question` carries as a second follow-up with a note.
  Four scoping answers are filed there: the baseline is the build-code target density the
  rebuild estimate already uses rather than the literal `fillfactor / 100`, the leaf term
  clamps at zero rather than going negative, the columns are renamed so no archived output
  is silently reinterpreted, and both retained servers are restarted and both texts run.
- **No fixture rebuild was needed this time.** Both clusters reproduced their filed output
  byte for byte on the first run (2,448 bytes over 27 rows on 12.2, 2,531 over 28 on
  17.11), because the previous follow-up ended with the fixtures rebuilt rather than
  reindexed. So this is a comparison of two texts, not of two database states.
- **Only the two renamed fields moved, measured two ways on both servers.** The 12
  untouched presentation fields are byte-identical (2,109 and 2,180 bytes); one view per
  text over the internal `final` stage exposes 28 columns either way with `wasted_space`
  swapped one-for-one for `wasted_vs_fillfactor`, and `EXCEPT` in both directions over the
  27 shared columns returns **0 rows across 214 indexes on 12.2 and 220 on 17.11**. Over
  every index: 0 negative values, 0 values above the old column, 121 and 117 now exactly
  zero, and 110 and 110 unchanged — the two `fillfactor = 100` fixtures, where
  `target_density` is exactly 1, plus the 108 and 104 indexes with no leaf pages. 24 of 27
  rows remain byte-identical across the two servers, the same three exceptions as before.
- **Cost is unchanged**: identical plan shape (4 `CTE Scan` nodes; 68 plan lines on 17.11,
  60 on 12.2) and **identical total buffers, 108,021 and 108,327**, differing only in the
  hit/read split; `EXPLAIN (ANALYZE, BUFFERS)` 131.1 against 120.9 ms on 17.11 and 123.6
  against 117.5 ms on 12.2, over six interleaved end-to-end runs of each text per server.
- **Ground truth is a rebuild, and it was run last because it is destructive.** After
  `REINDEX INDEX` over every scored index the new column reads exactly 0 for 81 of 97 and
  76 of 96, ≤ 0.1% for 87 and 85, and **≤ 0.4% for every index the report actually prints**.
  The worst residual anywhere is the same fixture on both servers: `c_one_idx`, one tuple
  on one leaf page at 0.29% density, 7,309 bytes and 44.6% — a real limit of the
  definition, excluded from the report by the 1 MB `min_index_bytes` prefilter and worse
  under the old baseline (8,128 bytes, 49.6%). A freshly rebuilt 416 kB `fillfactor = 100`
  index leaves 1.5% because its rightmost page holds the remainder.
- **The two percentage columns are related by a closed form that was predicted and then
  measured**: for in-page waste, `wasted_vs_fillfactor / est_reclaimable` tends to
  `(leaf_capacity - target_free) / block_size`, 0.8951 at 8192/90, measured 0.8868-0.8921
  over the 13 dead-page-free indexes with more than 1 MB of reclaim; `1.0001` on the
  1,918-dead-page fixture and `0.8321` on the wide-key one that mixes both.
- **Four new fixtures were built because every previous fixture with real waste had
  fillfactor 90.** Bloated at 100/50/10, they reproduce the predicted ratio to four
  decimals on both servers (0.9951/0.9895, 0.4951/0.4931, 0.0952/0.0948) and expose the
  consequence of the user's request: **`i_ff10_del90` is 89.9% reclaimable and reports 8.3%
  wasted**, because at fillfactor 10 nine tenths of the file is free space by instruction.
  The page states plainly that `est_reclaimable_pct` remains the column to read for "how
  much disk will `REINDEX` return". A `fillfactor = 50` fixture whose pages are all dead
  reads 89.7% on both columns, confirming dead pages stay 100% waste at any fillfactor.
- Those fixtures also found the reclaim estimate's first vacuumed miss over one point on
  this page (`87.1` against `89.9` on `i_ff10_del90`), attributed with source citations to
  the per-page high key that `avg_leaf_density` counts as payload but a rebuild writes ten
  times less often; the attribution is labelled plausible-not-proven and filed as an open
  question, along with the note that the page's accuracy figures are a fillfactor-90 result.
- Page edits: the second follow-up prompt and its note in `## Question`, the four SQL edits,
  a rewritten `wasted_space` row and example block in `### How to read the output`, one new
  `### Follow-up: wasted space measured against the fillfactor` section, the renamed and
  rewritten `### wasted_vs_fillfactor is not est_reclaimable`, a new paragraph in
  `### How this was measured`, an updated `### Everything the two servers agreed on`, 2
  Contents entries changed, 1 Context Reviewed bullet, 7 Evidence Map rows, 5 new or
  rewritten Open Questions, and 3 Source References. Checked mechanically: 122 citations
  resolve with in-file line ranges, 32 page-internal anchors resolve, and the Contents list
  matches the page's 26 sections in order.
- Measured by restarting the two clusters retained under `.wiki-runtime/tmp/pgsi/` — 12.2
  built from this repo's v12 pin with `contrib/pgstattuple` compiled through PGXS, and
  17.11 from the `--with-icu --enable-debug` install of this pin — both at `block_size`
  8192, `autovacuum = off`, `fsync = off`. `pgstatindex` remains the only measurement
  function; no `pageinspect`, no `pgstattuple()`, no `amcheck`. Both servers were shut down
  cleanly and the sandbox is retained; note that the residual pass reindexed the fixtures,
  so a future run must re-run `10_fixtures.sql`/`11_fixtures17.sql` first.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is unchanged;
  the page keeps `verified: false` and `verified_by_agent: not yet`.

## [2026-08-23] remove v17 | delete COMMENT-stored bytes-per-table-tuple REINDEX threshold question

- Removed
  `wiki/v17/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md`
  at the user's request. The user named its exact visible title, including the
  ` (unverified)` state marker. The request's casing and punctuation issues drive
  no document generation and are restated on no wiki page, so no prompt-hygiene
  correction was needed.
- Removed the active page entry from `wiki/index.md` and `wiki/v17/index.md`.
  No surviving v17 content page linked to the deleted page, so no `## Navigation`
  section required an edit.
- Removed the page's 132-cell re-measurement clause from the PostgreSQL 17
  coverage cell in `wiki/versions.md`. The PostgreSQL 12 page with the same
  basename remains filed and indexed.
- Neutralized the two historical Markdown links in `wiki/log.md` and the one in
  `wiki/versions.md`; they now retain the deleted title as plain text without a
  dangling link. Historical filename mentions remain historical records.
- No source citation, version pin, or verification field on a surviving page
  changed. `raw/postgres-17/` is unchanged and no server was started.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint` completed with 9 errors and
  0 warnings, all outside this change set: six missing v18 source-citation targets
  under `src/test/modules/injection_points/`, plus unavailable pinned commits in
  the v14, v18, and v19 raw checkouts. The requested deletion leaves no Markdown
  link to the removed v17 page. `git diff --check` passed.

## [2026-08-24] cleanup | removed the idxmaint sandbox and purged .wiki-runtime caches and logs

- Removed `.wiki-runtime/tmp/idxmaint/` at the user's request, reclaiming 11 GB
  and leaving `.wiki-runtime/tmp/` empty. The cluster data directory was nearly
  all of it at 11 GB, with `pg_wal` alone at 8.1 GB and `base` at 2.3 GB; the
  rest was a 412 MB in-tree PostgreSQL 17 build under `src17/`, a 98 MB
  `install/`, and 1.1 MB of build transcripts (`make.log` 908 KB, `install.log`
  188 KB, `configure.log` 16 KB).
- The 128 KB harness was removed with the bulk, on the user's instruction:
  `cells.sh`, `probes.sh`, `harness.sh`, `analyze.py`, five files under `sql/`,
  three under `work/`, and both result sets, `results/eval.csv` (12 KB) and
  `results/metrics.csv` (21 KB).
- The sandbox was unfiled work, not the reproduction base for any page.
  `harness.sh` describes it as the harness for the "@idxmaint COMMENT-stored
  index-inflation heuristic (PostgreSQL 17.11)", the successor to the v17
  question deleted in the entry above. It ran 11:12-11:47 on 2026-08-24, after
  the 10:56 bookkeeping writes that were the last wiki edits of that session, so
  no page was ever written from it. `run_all.log` also shows all 17 cells,
  `c0_control` through `c16_partial_churn`, reporting `table "tN" does not
  exist, skipping`, so the run was already misfiring before it stopped.
- No PostgreSQL instance was running at cleanup time and none had to be stopped.
  The host restarted at roughly 11:51, which killed postmaster PID 508251
  mid-shutdown: `server.log` records `received smart shutdown request` at
  11:50:37 and no completion, and the tree still held a stale
  `data/postmaster.pid` reading `stopping` plus a stale
  `sock/.s.PGSQL.55417`. PID 508251 was confirmed absent from `/proc`, and no
  postgres, `pg_ctl`, or `initdb` process remained.
- Purged the runtime scaffold as well: `cache/wiki_lint/last-run.txt`,
  `logs/recent_log.log`, `logs/wiki_lint.log`, and the three empty `indexes/`
  subdirectories. `venv/` was kept, as the tooling requires it. The cache is
  write-only — `write_cache()` never feeds back into `lint()` — so removing it
  suppresses no check, and `ensure_runtime_dirs()` recreated `cache/wiki_lint`,
  `indexes/ctags`, `indexes/search`, `indexes/tree-sitter`, `logs`, and `tmp` on
  the next tooling run. `.wiki-runtime/` is back to 13 MB and the repo to
  3.3 GB, down from 15 GB.
- No compiled install remains again, so the next exact-pin PostgreSQL 17
  experiment must rebuild from `raw/postgres-17/`.
- No wiki page, index, version pin, or `raw/` checkout changed; this entry is the
  only edit. All five checkouts remain clean with zero modified files and sit on
  their manifest pins: `45b88269` (12), `a92fbdfb` (14), `786db8dc` (17),
  `baa7b142` (18), and `67342a14` (19).
- Stale pointer left uncorrected: [B-Tree Bloat and Wasted Space From
  pgstatindex Alone, on PostgreSQL 12 and 17
  (unverified)](v17/questions/indexing/btree-bloat-with-pgstatindex.md) still
  states its sandbox is retained at `.wiki-runtime/tmp/pgsi/`. That directory was
  already absent before this cleanup — `tmp/` held only `idxmaint/` — so the
  claim was stale beforehand and this cleanup did not cause it.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. The
  nine errors recorded in the entry above are gone, and this cleanup did not mask
  them: all nine were `raw/` availability errors, the `raw/*/.git` directories
  were refreshed at 10:12 on 2026-08-24 before any deletion here, the six
  `src/test/modules/injection_points/` citation targets now exist in
  `raw/postgres-18/`, and each of the v14, v18, and v19 pins now resolves.

## [2026-08-24] lint-fix | repaired eight broken ## Contents anchors across five pages

- `scripts/wiki_lint` reported 0 errors and 0 warnings, plain and with
  `--warnings-as-errors`, and no other linter is configured in the repo. The
  defects fixed here are in the two areas `MANDATORY Table of Contents` and
  `MANDATORY Question Categories` state that lint does not cover: page-internal
  `#`-anchor links and question category placement. Both were checked directly.
- Category placement is clean: every `type: question` page sits in one of the six
  closed category directories and none is filed directly under
  `wiki/vNN/questions/`.
- Eight `## Contents` links resolved to no heading, all one root cause — the
  anchor dropped an underscore or a hyphen that the heading keeps:

  | Page | Was | Now |
  |---|---|---|
  | `v12/questions/indexing/how-pgstatindex-calculates-information.md` | `#how-indexsize-is-calculated` | `#how-index_size-is-calculated` |
  | same | `#how-avgleafdensity-is-calculated` | `#how-avg_leaf_density-is-calculated` |
  | same | `#how-leaffragmentation-is-calculated` | `#how-leaf_fragmentation-is-calculated` |
  | `v12/questions/indexing/invalid-index-outcomes.md` | `#5-pgupgrade-from-postgresql-96-or-earlier` | `#5-pg_upgrade-from-postgresql-96-or-earlier` |
  | `v12/questions/indexing/null-values-in-indexes.md` | `#btree` | `#b-tree` |
  | same | `#spgist` | `#sp-gist` |
  | `v12/questions/observability/explain-analyze-buffers-output.md` | `#io-timing-and-trackiotiming` | `#io-timing-and-track_io_timing` |
  | `v18/questions/indexing/create-index-concurrently.md` | `#the-pgindex-state-machine` | `#the-pg_index-state-machine` |

- Targets were generated with the `MANDATORY Table of Contents` slug rule — trim,
  lowercase, whitespace runs to `-`, strip punctuation such as backticks, `/` and
  `.`, preserve underscores and hyphens — after checking that implementation
  reproduces all three worked examples in that section
  (`the-three-pg_index-state-flags`, `how-maintenance_work_mem-is-used`,
  `open-questions`). `GiST` -> `#gist` was already correct and was left alone.
- Only the eight anchor targets changed. No heading, body, citation, front matter,
  or verification field was touched, so the timestamped `verified_by_agent` values
  on the `how-pgstatindex-calculates-information` and `null-values-in-indexes`
  pages still stand; a Contents typo is not a claim change. `raw/` is unchanged.
- Re-checked after the edits: 0 anchor problems, 0 category problems, and
  `.wiki-runtime/venv/bin/python scripts/wiki_lint --warnings-as-errors`: 0 errors,
  0 warnings. `git diff --check` passed.

## [2026-08-24] follow-up v17 | removed the reltuples = 0 guard from the recommended bloat statement

- Removed the five-line `live_rows` branch the asker quoted from
  [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17
  (unverified)](v17/questions/indexing/btree-index-bloat-core-sql-only.md), against unchanged
  pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11). `live_rows` is a two-arm `CASE`
  again — only the v14+ `-1` sentinel stops the model — so **change E is now unconditional**:
  every `reltuples = 0` is priced as an empty index on 12 through 17 and the row reports its
  whole file as reclaimable. The header comment above the branch and the stage-map line for
  change E were rewritten to match; `server_version_num` is still read and reported but **no
  expression in the statement tests it any more**.
- Prompt hygiene: the request had `agents.md` for AGENTS.md, lowercase `postgresql`, spaces
  before commas and `statatement` for statement; the asker approved a corrected restatement,
  which `## Question` carries with the quoted SQL character for character. Three scoping
  answers are filed there: the removal is **literal** and its false positives are accepted,
  rather than moving the condition into `suppress_row` or turning it into a caveat; **both
  servers are rebuilt and the whole suite re-scored**; and change E is **rewritten in place**
  rather than left as history beside a new follow-up.
- One consequence was not requested and is filed as part of the change: with the branch gone,
  `live_rows IS NULL` means `reltuples < 0` and nothing else, so the `status` arm reporting
  `unmeasured: reltuples 0, table has live rows` is unreachable and was deleted. A **third
  statement text** that keeps the arm returns **0 rows from `EXCEPT` in both directions over
  all 52 exposed columns** on both servers and scores an identical verdict distribution, so
  the deletion is proven output-identical rather than argued.
- **The two majors move in opposite directions, which is the whole result.** On 17.11 the
  change buys nothing and costs one critical false positive: `EXCEPT` moves **3 of 118 rows**,
  `p115` and `p117` are withheld by change B either way, and `p118` — a queue that refilled
  after `ANALYZE` measured its subset empty — goes from `unmeasured` to **99.3% waste on a
  1.1 MB index a `REINDEX` reproduces block for block**, with `status = ok` and an empty
  `caveats`. That is the same fixture and the same reading that got change B's auto-analyze
  trigger rejected as change E's test in the previous follow-up, so the failure is adopted
  knowingly. Verdicts go 43 PASS / 70 WITHHELD / 1 UNMEASURED / 3 critical / 1 borderline /
  1 false negative to 43 / 70 / **0** / **4** / 1 / 1, and the as-filed report changes by
  exactly one line.
- On 12.2 it is a real trade, because change E's counter test began with
  `server_version_num < 140000` and withheld every zero there: **7 of 103 rows** move, three
  of them genuinely reclaimable (`p113b`, `p113c`, `p75` at 100.0 / 100.0 / 99.6 against
  measured 100.0 / 100.0 / 99.6) and two of them false positives (`p118` at 99.3, `p120` at
  87.5). Verdicts go 54 PASS / 5 UNMEASURED / 5 critical to **57 / 0 / 7**, and the drained
  job-queue demo reads `unmeasured` before and `ok | 100.0 | 21 MB` after, on an index a
  `REINDEX` takes 2745 blocks to 1.
- **A new non-partial fixture family (121) prices the majors' difference from source.** An
  index whose table is emptied, `VACUUM`ed and reloaded reads 100.0 on both servers against a
  measured 50.0; add a `REINDEX` while the table is empty, or use a `TRUNCATE`, and 17.11
  reads `unmeasured: reltuples unknown` while **12.2 reads 100.0 and 99.9 on dense indexes a
  `REINDEX` cannot shrink**. The mechanism is `RelationSetNewRelfilenumber` writing
  `reltuples = -1` with a new relfilenode plus `index_update_stats` leaving that `-1` alone
  when the build counts zero rows; traced live on 17.11 as `1e+06` after the build, `0` after
  `VACUUM`, `-1` after `REINDEX`. All three non-partial fixtures carry `row-count sources
  disagree: analyze first`, which the page's reading rule already excludes; the partial
  `p118` carries nothing, because that caveat is written `WHEN NOT is_partial`.
- Timing is unchanged: six interleaved pairs read 43.4-51.3 ms filed against 43.5-53.1 ms
  amended on 17.11 and 36.0-42.0 against 35.3-63.3 on 12.2, the two outliers being first runs.
- Measured on the two servers the previous follow-up kept under `.wiki-runtime/tmp/pstate/`,
  restarted rather than rebuilt, with a **fresh database on each** so that run's post-`REINDEX`
  databases survive as a reproduction check. Fixture fidelity against its stored result table:
  same 119 index names, **119 of 119 identical pre-rebuild block counts**, 119 of 119 identical
  `status` values, 118 of 119 identical post-rebuild block counts, 107 of 119 identical floor
  readings. The verdict totals differ from that run (43/70 against 50/62) because nine tables'
  `CREATE TABLE AS` counts reached the cumulative statistics after their `ANALYZE` rather than
  before, firing change B. Two fixture defects were found in passing and are filed: `p75`
  deletes 100% of its predicate subset where its comment says 90%, and
  `CREATE INDEX np99 ON np99 (v)` cannot run at all because an index and a table share a
  namespace.
- Page edits: the new follow-up prompt and its note in `## Question`, the SQL block (branch,
  comment, stage map, dead status arm), 4 Contents entries renamed, the fifteenth-follow-up
  Verdict paragraph, the change-E paragraph in [The current recommended
  statement](v17/questions/indexing/btree-index-bloat-core-sql-only.md), 2 new residual-error
  rows, the `modelled_rows = 0` reading rule in two places, the whole change-E section block
  rewritten in place (3 sections retitled), 2 Context Reviewed bullets, 6 Evidence Map rows,
  and 8 rewritten or new Open Questions. Checked mechanically: the Contents list matches all
  120 headings in order and all 365 page-internal anchors resolve.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings. `wiki/index.md`,
  `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is unchanged; the page keeps
  `verified: false` and `verified_by_agent: not yet`.

## [2026-08-24] answer v17 | COMMENT-stored inflation heuristic for the five non-B-tree AMs

- Filed `wiki/v17/questions/indexing/non-btree-index-inflation-comment-baseline.md`
  against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11): a designed
  and measured `REINDEX`-candidate heuristic for HASH, GIN, GiST, SP-GiST and BRIN
  whose only persistent state is an `@idxmaint:` JSON payload appended to the index's
  own `COMMENT ON INDEX`. No table, no extension, no external store.
- Prompt hygiene: the request wrote `agents.md` for AGENTS.md, lowercase `postgresql`,
  spaces before commas, `minmax-multi` for the `minmax_multi` opclass family, and
  `DESUMMARIZE + SUMMARIZE` for `brin_desummarize_range()` / `brin_summarize_range()`.
  The asker chose "correct and restate", so `## Question` carries the corrected text
  and names the corrections. Three further scoping answers were taken up front: a full
  measured run rather than a design-only page, ~1M-row fixtures, and permission to
  delete unused `.wiki-runtime` data before testing.
- **Model.** `expected_fresh = base_size * cur_pop / base_pop`, with the population
  unit chosen per access method: indexed tuples for hash/GiST/SP-GiST, index tuples x
  summed `pg_stats.avg_width` for GIN, and `ceil(table relpages / pages_per_range)`
  for BRIN. Two deliberate deviations from the brief are argued in the page: the churn
  ratio is normalized by `n_live_tup` rather than by the logical population (dividing
  BRIN churn by summarized ranges gives ratios in the thousands), and baseline
  `n_live_tup` / `n_dead_tup` were dropped from the payload as having no decision
  power, while `ppr`, `iw`, `anl` and `dbr` were added. Final payload is 19 fields,
  measured at 212 bytes inside a 309-byte comment.
- **Result.** 13-cell matrix on an isolated 17.11 server built from the pin,
  `REINDEX INDEX` as ground truth, `VACUUM FULL` never used. Seven cells flagged
  `strong REINDEX candidate` reclaimed 37.7% / 80.2% / 85.5% / 79.4% / 73.0% / 85.7% /
  74.8%; four controls (no-churn hash, BRIN minmax, BRIN minmax_multi, partial hash)
  were correctly left alone. **`1 - 1/size_inflation` predicted the actual reclaimed
  fraction to 0.0 points on 7 of 13 cells and within 2.5 points on 12 of 13.** The
  single large miss is `c01_hash_dup` at +19.7 points, where the key *distribution*
  changed and a fresh build of the new data is legitimately larger.
- **Reproducibility.** The matrix ran three times end to end. All three produced
  identical `(B, post-churn, C, R)` quadruples on all 13 cells. Only the inflation
  figures drift, by 1-2%, because the population term comes from an `ANALYZE` sample;
  the per-run spread is filed as a table.
- **Three catalog facts the design turns on**, each measured and cited: an index's
  `pg_class.reltuples` means the AM's own `index_tuples` after `CREATE INDEX`, the
  *table's* row estimate after `ANALYZE` (`analyze.c:449` sets `tupleFract = 1.0` and
  `compute_index_stats` skips plain non-partial indexes), and the AM's
  `num_index_tuples` after `VACUUM` - measured 300000 / 200000 / 180000 for GIN and
  43 / 200000 / 22 for BRIN, so a baseline is only valid if captured after `ANALYZE`
  and the evaluation refuses to score without one; **v17 has no per-table statistics
  reset timestamp** (`PgStat_StatTabEntry` has no such field and the relation kind
  registers no `reset_timestamp_cb`), so resets are caught only by counter
  monotonicity, which is why the raw counters are stored; and a parallel BRIN build
  writes `reltuples = 348` where the true range count is 70, because
  `_brin_parallel_heapscan` copies the workers' partial count into `bs_numtuples` and
  `form_and_insert_tuple` then increments it again.
- **Two measured results contradict the brief and changed the design.**
  `brin_desummarize_range()` over all 318 ranges freed **nothing**, and the following
  `brin_summarize_new_values()` grew the index **71%**, from 114,688 to 196,608 bytes,
  where `REINDEX` returned it to 114,688 - so the BRIN arm never recommends it and
  uses the highest thresholds of any AM. And flushing a GIN pending list **grew** the
  index from 16,654,336 to 21,905,408 bytes (`pageinspect` as ground truth, 491
  pending pages to 0), so a pre-`VACUUM` GIN reading understates the eventual size
  rather than overstating it.
- Edge cases proven on the server: a manual `REINDEX` flips `baseline_state` to
  `rebuilt since baseline` via the stored filenode, and so does
  `REINDEX CONCURRENTLY`, which also moves the index OID 16649 -> 16651 while
  `index_concurrently_swap` carries the `pg_description` row across; after
  `pg_stat_reset_single_table_counters()` the statement reports
  `churn_state = unknown: counters reset` with a NULL churn ratio and a `weak`
  recommendation while still publishing the 3.167 size reading; the partial cell
  suppressed at `pf_shift` 9.024 against 2.5% actually reclaimable; and a two-line
  human comment containing `@` and `}` survived capture, re-capture and both `REINDEX`
  forms unchanged.
- Two revisions were made *because* the probes failed: the reset case originally
  reported `inconclusive: no VACUUM since baseline` (a reset zeroes `vacuum_count`
  too) and printed a fabricated `churn_ratio` of `-200000.000`; and `c01_hash_dup`
  originally used a single hot key, which is O(n^2) because the bucket is a pure
  function of the hash code - it completed 1 row in 90 seconds before being abandoned
  for 100 hot keys.
- Eleven open questions are filed, including that the BRIN arm is unvalidated against
  any true positive (both BRIN cells reclaimed 0 bytes), that the multiplicative model
  has no intercept and therefore reads 0.209 on a BRIN index whose truth is about 1.0,
  that the `[0.7, 1.43]` partial suppression window is an unmeasured guess, that only
  one fixture scale was tested, and that expression and multi-column indexes were not
  exercised at all.
- Environment: `.wiki-runtime/tmp/pstate/data`, `data12`, `src17`, `src12`,
  `install12` and scratch files were deleted at the user's explicit instruction before
  testing, reclaiming 17.3 GB; this makes stale the "the sandbox is retained" claim in
  `wiki/v17/questions/indexing/btree-index-bloat-core-sql-only.md`. The exact-pin
  17.11 install under `.wiki-runtime/tmp/pstate/install/` was kept and reused. The new
  sandbox is `.wiki-runtime/tmp/idxm/` and is retained, holding the harness, the 13
  cells, the probes and four result sets.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is
  unchanged; the page keeps `verified: false` and `verified_by_agent: not yet`.

## [2026-08-27] cleanup | emptied .wiki-runtime to the venv alone before the GIN work

- At the user's explicit instruction ("everything except venv"), removed
  `.wiki-runtime/tmp/idxent/` (98 MB, of which the exact-pin 17.11 `install/` was
  98 MB and the harness SQL, `check_citations.py` and server logs 116 KB), plus
  `cache/`, `indexes/` and `logs/`. `.wiki-runtime/` went from **111 MB to 13 MB**
  and now holds only `venv/` and an empty `tmp/`. The repo is unchanged at 3.3 GB.
- `tmp/idxent/` was the sandbox of the 2026-08-27 review of
  [Reading an Index's Entry Count From the Catalogs, for Every Index Type, in
  PostgreSQL 17 (unverified)](v17/questions/indexing/index-entry-count-from-catalogs.md).
  That review's own coverage note already recorded the data directory as deleted and
  the install as the only survivor, so this cleanup removed the install and the
  harness; re-verifying that page now means rebuilding 17.11 from `raw/postgres-17/`
  and re-running its published fixture SQL, which its note already says.
- No PostgreSQL process was running at cleanup time (`pgrep -a postgres` empty) and
  none had to be stopped. No data directory was present to remove.
- No wiki page, index, version pin, or `raw/` checkout changed in this step.
  `raw/postgres-17/` is clean at `786db8dcf168bd9df8f55047337525ac19118b1c`.

## [2026-08-27] answer v17 | GIN REINDEX heuristic from a COMMENT baseline and normalized index growth

- Filed `wiki/v17/questions/indexing/gin-reindex-normalized-growth-comment-baseline.md`
  against unchanged pin `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11): a designed
  and measured GIN-only `REINDEX CONCURRENTLY` candidate heuristic whose entire persistent
  state is a **133-byte `@ginbase:` JSON payload** appended to the index's own
  `COMMENT ON INDEX`. Nine flat fields, no table, no extension, four SQL statements.
- Prompt hygiene: the request wrote `agents.md` for `AGENTS.md` and lowercase
  `postgresql`, put a space before the comma after the last requirement, spliced the
  `.wiki-runtime` cleanup instruction into the question text, and wrote `75–80%` with an
  en dash. The asker chose "correct and restate", so `## Question` carries the corrected
  text and names the corrections; the cleanup instruction is carried out (entry above) but
  excluded from the filed question. Three scoping answers were taken up front: full purge
  of `.wiki-runtime` except the venv, a measured run rather than a design-only page, and a
  new GIN-specific page rather than a revision of the five-AM sibling.
- **Premise, from source.** No file under `src/backend/access/gin/` calls
  `RelationTruncate`; `ginvacuumcleanup` hands recyclable pages to the free space map and
  `ginInsertCleanup` does the same for flushed pending pages, so `pg_relation_size` on a
  GIN index is a high-water mark that only `REINDEX` lowers.
- **Why the brief forbids the index's own `reltuples`, measured.** One 200,000-row table's
  GIN index reads **1000000 / 200000 / 100000 / 500000** over `CREATE INDEX` / `ANALYZE` /
  `DELETE 50% + VACUUM` / `REINDEX` - a 10x swing decided by whichever command wrote last,
  because the build writes extracted entries, `ANALYZE` writes `ceil(1.0 * totalrows)`, and
  GIN's `amvacuumcleanup` writes the heap tuple count under its own `XXX ... bogus` comment.
- **Result.** 12-cell matrix, three full runs in three fresh databases, `REINDEX INDEX` as
  ground truth, `VACUUM FULL` never used. **Right on 10 of 12 cells.**
  `1 - 1/normalized_index_growth` predicted the reclaimed fraction to **within 0.05 points
  on 5 of the 9 evaluable cells** (83.82/83.82, 54.63/54.63, 57.18/57.14, 11.91/11.95,
  0.00/0.00). Runs 2 and 3 produced byte-identical ground-truth tables; baseline payloads
  differ only in `ts` and `bfn`.
- **The published statements are the tested ones.** A script extracts every fenced `sql`
  block carrying a `wiki_gin` tag from the filed page, asserts there are exactly three, and
  runs them verbatim (41 / 18 / 76 lines) in a fourth database against `orders_tags_gin`.
  The whole capture -> churn -> evaluate -> rebuild -> re-capture lifecycle worked and the
  prediction came out at **55.66% against a measured 55.66%** (37,150,720 -> 16,474,112).
- **Four brief premises corrected on evidence.** (1) The
  `index size remains >= 75-80% of baseline` clause is **vacuous** on GIN, so the shrinkage
  rule reduces to `heap_tuple_ratio <= 0.50` and thereby forces `normalized >= 2.0` - a
  strict subset of the growth rule that can never fire on its own; the only sub-1
  `index_size_ratio` seen anywhere (0.5334) appeared after the ground-truth `REINDEX`, where
  the `relfilenode` detector had already diverted the row. (2) "Both grew 50%, so
  `normalized = 1.0`, so do nothing" is false: +50% rows grew the index **+120%**
  (20,594,688 -> 45,391,872) for `normalized` 1.4694, and `REINDEX` reclaimed **39.51%**.
  (3) A fresh GIN build is **not linear** in heap tuples - 7,495,680 / 13,729,792 /
  20,594,688 / 82,329,600 / 82,329,600 / 246,169,600 bytes at 250k / 500k / 1M / 2M / 4M /
  8M rows over one 10,000-key universe, quadrupling between 1M and 2M as every posting list
  converts to a posting tree (`n_data_pages` 0 -> 10,000, `n_entry_pages` 2,513 -> 49) and
  then adding zero bytes to 4M; this is exactly why the same-keys doubling over-predicted
  67.65 against an actual 35.33 while the new-keys doubling, whose lists stayed inline, hit
  11.91 against 11.95. (4) The blind spot is sized by `gin_pending_list_limit`, not by any
  ratio: a paired probe added 150,000 rows to two identical 600,000-row tables and the
  `fastupdate = on` index grew **515 pages / 4,218,880 bytes** while the `fastupdate = off`
  index grew **nothing at all**; `VACUUM` flushed the pending list, the 515 pages showed up
  in `pg_freespace` and stayed in the file, and `REINDEX` returned both to 16,474,112. A
  six-point growth sweep therefore reads `normalized` 1.1419 / 1.0049 / 0.8374 against
  20.39 / 20.39 / 0.48% reclaimable.
- **Two design changes were forced by failed probes.** The first stale-statistics guard used
  `n_mod_since_analyze > 0.10 * reltuples` and misfired on the mass-delete cell in every run,
  reading 600000 while the same `VACUUM (ANALYZE)` had set `reltuples` correctly to 400000:
  `pgstat_report_analyze` zeroes the counter, but a backend's pending counts are applied
  **additively** afterwards and non-forced flushes are rate-limited to `PGSTAT_MIN_INTERVAL`
  (1000 ms). Reproduced independently at **1 of 4** back-to-back `DELETE` +
  `VACUUM (ANALYZE)` pairs, with `n_live_tup` clamped to 0 and `n_dead_tup` at 210000 while
  `pg_class.reltuples` read 90000; a 2 s pause was clean every time. The counter is
  demoted to an advisory `stats_lag` column and the veto replaced by an `analyze_count`
  comparison - which is what caught the one cell whose ratios were silently meaningless
  (60% deleted, no `ANALYZE`, `normalized` 1.0000 over a real 46.66% reclaim). Second, the
  churn gate and the `ANALYZE` gate were originally the other way round, which made the
  no-churn control report "no ANALYZE since baseline"; swapping them is the only difference
  between run 2 and run 3 output.
- **The baseline is a function of `maintenance_work_mem`**: the same 2M-row index rebuilt to
  **109,371,392 bytes at 64MB against 82,329,600 at 256MB and 1GB**, 32.8% apart, because
  `ginBuildCallback` dumps its accumulator at `maintenance_work_mem` and the 64MB build ends
  up with 3,350 entry pages against 49 for the same 10,000 keys.
- **Eleven edge cases proven on the server**, including `REINDEX CONCURRENTLY` moving the
  `pg_description` row as the index OID goes 16900 -> 16904 (so the payload survives with a
  stale `bfn`, and the detector fires), plain `REINDEX` keeping OID 16900 while the filenode
  moves 16900 -> 16903, a two-line human comment containing `{do not drop}` and `@ 100%`
  surviving capture and both rebuild forms, `COMMENT ON INDEX` holding
  `ShareUpdateExclusiveLock` on the **index only** and none on the table, a non-owner reading
  the baseline but refused a write at `SQLSTATE 42501`, `CREATE INDEX` itself setting the
  table's `reltuples` (-1 -> 80000) so a `t0` capture needs no `ANALYZE`, `TRUNCATE` resetting
  both to -1 and the capture refusing loudly, and `pg_dump --section=post-data` carrying the
  whole payload.
- **Citation check found and fixed a real error.** A purpose-built checker verified all
  **119 citations** resolve, sit in range, cite only `raw/postgres-17/`, and carry labels
  consistent with their own line ranges. It caught that the entry increment is in
  `ginHeapTupleBulkInsert`, called once per indexed column by `ginBuildCallback`, not in
  `ginBuildCallback` itself - which sharpened the claim to "summed over rows *and* columns" -
  plus twelve label/range mismatches and one `analyze.c:952-953` label against an
  `L948-L953` range. All fixed; the re-run reports zero problems. All 30 `## Contents`
  anchors were separately verified to resolve, cover every heading, and match document order.
- Thirteen open questions filed, including that **partial and multi-column GIN indexes were
  never tested** - where the table `reltuples` the heuristic divides by is not the indexed
  population at all, since `ginvacuumcleanup` says its own count is "bogus if the index is
  partial" and `ANALYZE` computes a real `tupleFract` for one - that `normalized_index_growth`
  is not monotone in reclaimable space (it ranks 3.0909/35.33% above 2.4733/56.24%), that
  autovacuum was off throughout, and that `est_reclaimable` is still printed even though it
  read 82 MB against a true 45 MB on one cell.
- Environment: `.wiki-runtime` was emptied to the venv alone first (entry above), so 17.11
  was rebuilt out of tree from `raw/postgres-17/` via `git archive` into
  `.wiki-runtime/tmp/ginorm/src17/`. The sandbox is **retained** at
  `.wiki-runtime/tmp/ginorm/`, holding the install, the data directory, the harness
  (`matrix.sh`, `sweep.sh`, `verify_published_sql.sh`, `check_citations.py`), 12 SQL files
  and all result sets. The cluster is left running on socket
  `.wiki-runtime/tmp/ginorm/sock` port 55461.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; `raw/` is unchanged;
  the page keeps `verified: false` (human-only) and sets
  `verified_by_agent: claude-opus-5-max 2026-08-27T14:51:41Z` after the 126-citation and
  30-anchor checks above.

## [2026-08-27] cleanup | purged .wiki-runtime to the venv alone, removing the ginorm GIN sandbox

- Removed `.wiki-runtime/tmp/ginorm/` at the user's request, taking `.wiki-runtime` from
  **9,417,254,911 to 10,659,161 bytes** and reclaiming **9,406,595,750 bytes** (8.76 GiB).
  What remains is `venv/` alone. This is the sandbox behind [A COMMENT-Stored Baseline and
  Normalized Index Growth for Finding GIN Indexes That Need REINDEX CONCURRENTLY in
  PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-reindex-normalized-growth-comment-baseline.md),
  which the two entries above had retained. **This entry supersedes the "Environment"
  bullet of the entry directly above**, which says the sandbox is retained and the cluster
  is left running on port 55461; both statements were true when written and are now false.
- **A live server had to be shut down first**, unlike the 2026-08-26 ginw2 cleanup where
  nothing was running. `postgres` PID 43546 was still serving
  `-D .wiki-runtime/tmp/ginorm/data` on socket `.wiki-runtime/tmp/ginorm/sock` port 55461,
  with the checkpointer, background writer, walwriter and logical replication launcher
  attached and one client backend (the inventory `psql`). `pg_ctl -m fast stop` returned
  `server stopped`, after which no `postgres` process remained, `data/postmaster.pid` was
  gone and the postmaster had removed its own socket. No cluster was killed mid-write and
  no `rm` touched a running data directory.
- Disposable cluster data was almost all of it: `data/` **8,900,731,994 bytes** holding the
  four databases the page describes - `gin1` 3744 MB (the 12-cell matrix), `gin2` and
  `gin3` 1790 MB each (the two repeat runs), and `pubsql` 120 MB (the verbatim replay of the
  published statements). The out-of-tree `git archive` build tree `src17/` was 407,271,774
  bytes and the exact-pin 17.11 install (`postgres (PostgreSQL) 17.11`) was 97,292,483, so
  **no compiled 17.11 install now remains** and the next exact-pin GIN experiment must
  re-`configure` and re-`make` from `raw/postgres-17/`.
- **The harness went with it, and this one was not self-reproducing.** On the user's
  instruction the 224 KB of `sql/` (16 files), `results/` (9 sets) and `logs/` (19 runs)
  were deleted along with `matrix.sh`, `sweep.sh`, `verify_published_sql.sh`,
  `check_citations.py` and the 1.1 MB of configure/make/install logs. The cost is higher
  than for ginw2, which had published its fixture SQL inside the page: this page publishes
  only its **three** operational statements, so the 12-cell fixture builder
  (`00_fixtures.sql`), the churn script (`40_churn.sql`) and the probe, sweep and edge-case
  files are gone and would have to be re-derived from the page's prose methodology. The
  three published statements remain recoverable verbatim, and the page's measured figures
  were already published in full.
- Also cleared three stale tooling files: `logs/recent_log.log`, `logs/wiki_lint.log` and
  `cache/wiki_lint/last-run.txt`. Kept `venv/` and the empty `cache/`,
  `indexes/{ctags,search,tree-sitter}`, `logs/` and `tmp/` scaffold, which
  `ensure_runtime_dirs()` recreates on every script run anyway. `venv/bin/python` still
  reports Python 3.12.3.
- **Two stale sandbox pointers corrected**, the same failure the 2026-08-26 cleanup had to
  fix on the sibling page. `wiki/versions.md`'s 2026-08-27 coverage note said the sandbox
  "is retained"; it now records the same-day deletion and says reproduction means rebuilding
  out of tree from `raw/postgres-17/`. The page's open question 13 said "The sandbox is
  retained but the cluster is not guaranteed"; it now states what was lost, points at
  [Test methodology](v17/questions/indexing/gin-reindex-normalized-growth-comment-baseline.md#test-methodology)
  for the unpublished fixtures, and names the three statements that survive verbatim. Both
  new anchor targets were checked against the page's own `## Contents` block. No source
  citation, pin, measurement, heading or verification field changed, so the page keeps
  `verified: false` and `verified_by_agent: claude-opus-5-max 2026-08-27T14:51:41Z` - every
  claim it makes rests on pinned source or on measurements it publishes, not on the sandbox
  still existing.
- `raw/` untouched: all five checkouts sit at their pinned commits with a zero-length
  `git status --porcelain` (`45b88269a35`, `a92fbdfb830`, `786db8dcf16`, `baa7b142aac`,
  `67342a14863`), because the build was made out of tree.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-31] answer v17 | how much I/O a VACUUM FULL performs on a multi-GB, near-empty heap, measured on 17.11

- Filed [How Much I/O a VACUUM FULL Performs on a Multi-GB, Near-Empty Heap in PostgreSQL 17
  (unverified)](v17/questions/storage-and-vacuum/vacuum-full-io-on-near-empty-heap.md) at pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11, `REL_17_11-7-g786db8dcf16`). This is the
  PostgreSQL 17 counterpart of the same-basename v12 page, answered independently on v17 evidence and
  with the asker's test run on a purpose-built 17.11 server. It is the **first `storage-and-vacuum`
  page for v17**, so that category group was created in both `wiki/index.md` and `wiki/v17/index.md`.
- **Prompt hygiene applied first.** The prompt carried a comma splice (`is multiple GBs in size,
  however it has only a few thousand rows`), a comma after a question mark (`in relation to heap
  size?, Run tests`), and a lowercased `postgresql 17`. The asker was offered the choice and selected
  "correct and restate"; the restatement and every change are recorded under `## Question`. The asker
  also chose the v12 page's `bigint` row shape (for comparability), a "what changed since PostgreSQL
  12" section, and deletion of the sandbox afterwards.
- **The hypothesis holds exactly.** Four byte-identical fixtures were built - 200,000,000 rows in
  **884,956 blocks / 7,249,559,552 bytes** at 226 tuples per page, 99.9% deleted with the physically
  last rows kept so `VACUUM` could not truncate, then `VACUUM` and `VACUUM FREEZE` leaving **all
  884,956 pages all-visible and all-frozen** - and `VACUUM FULL` read **884,956 blocks with 0 buffer
  hits**, with the block layer delivering **7,249,559,552 bytes, the file size to the byte**, in
  3.02 s. `pg_stat_io` agreed with `pg_statio_all_tables` to the block in the `bulkread` context. A
  plain `VACUUM` on the identical heap read **0 blocks** in 3.17 ms, skipping 884,955 pages: the same
  map that let `VACUUM` read nothing let the rewrite read 6.75 GB.
- **The headline v17 result is the syscall count, not the byte count.** Those 884,956 blocks arrived
  in **55,361 read system calls**, 15.985 blocks each, against a theoretical floor of 55,310 for
  16-block reads over seven segments. `b7b0f3f2724` (`REL_17_0`) routed sequential scans through the
  read stream, `210622c60e1` added `io_combine_limit` (default 128 kB = 16 blocks), and
  `4908c587205` added `smgrreadv`, so the rewrite's scan - an ordinary `table_beginscan`, which sets
  `SO_TYPE_SEQSCAN` - inherits read combining. A **six-point sweep** on a 44,248-block fixture moved
  the call count **28,179 / 14,122 / 7,088 / 3,586 / 1,823 / 976** across `8kB` / `16kB` / `32kB` /
  `64kB` / `128kB` / `256kB`, i.e. 0.998 to 29.732 blocks per call, while
  `heap_blks_read + heap_blks_hit` stayed at **exactly 44,248 in all six runs** and the device
  delivered 231.3-232.2 MB every time. The syscall count is tunable; the block count is not.
- **Nothing skips a page.** Searching all three rewrite files for `visibilitymap`, `VM_ALL_` and
  `all_visible` returns **zero** matches in `cluster.c` and `rewriteheap.c` and exactly two in
  `heapam_handler.c` - an `#include` and the bitmap-scan callback's `VM_ALL_VISIBLE` - and the
  subagent's history sweep confirms **no commit has ever added VM skipping to this path**.
- **Also measured**: the new heap at **885 blocks / 7,249,920 bytes** (`ceil(200000 / 226)`, a
  999.95:1 read-to-write ratio); **28 `XLOG/FPI` records** carrying the 885 full-page images, because
  `log_newpages` batches `XLR_MAX_BLOCK_ID` = 32 per record; WAL collapsing **7,353,992 -> 110,008
  bytes** at `wal_level = minimal` with all 884,956 reads and 55,361 calls unchanged, and the
  backend's writes there reducing to **889** calls of which 885 are the heap's 8 kB extends; a
  **warm-cache** run identical on every read counter with **`read_bytes = 0`** and 1.20 s against
  3.02 s; the **zero-row** extreme (`vacuum_truncate = off`, all rows deleted) reading all 6.75 GB to
  produce a **0-byte** new heap in **three** write calls; an **indexed** run decomposing exactly to
  **885,841 = 884,956 old heap + 885 new heap** with `idx_blks_read` +1 and a 4,513,792-byte index
  rebuilt to the same size; `CREATE INDEX` on the small bloated heap reading 44,250 of 44,248 blocks;
  and the old relfilenode left at **0 bytes with `.1`-`.6`, `_vm` and `_fsm` already unlinked** before
  any checkpoint.
- **Four differences from the older major, each traced to a commit in this checkout's history.**
  (1) Emptied pages keep **zero** line pointers at `pd_lower = 24`, so `pgstattuple` reports
  **99.56%** free rather than being inflated by a retained array: `PageRepairFragmentation` truncates
  trailing unused line pointers, and because the fixture had no indexes `lazy_scan_prune` passed
  `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`, sending dead items straight to `LP_UNUSED`. (2)
  `heap_blks_scanned` still reports **0 of 884,956 for 427 of 434 samples** during the scan, but now
  **ends at exactly 884,956** - observed on 14 of 14 samples in the `rebuilding index` phase -
  because `3df51ca8b39` added the end-of-scan assignment; **it was back-patched to `REL_12_6`**, so
  this is a 12.0-12.5 distinction, not a 12-vs-17 one, which corrects the framing the research brief
  started from. (3) The write side goes through v17's bulk-write facility (`8af25652489`), which
  batches 32 pages, hides the new heap's extends from `pg_stat_io` entirely, and hands the fsync to
  the checkpointer via `smgrregistersync` instead of calling `smgrimmedsync`. (4) `reltuples` survived
  the aggressive freeze at the correct 200,000, because `vac_estimate_reltuples` keeps the old value
  when under 2% of an unchanged-size relation is scanned.
- **Two conflicts filed as open questions.** `pg_stat_io.reads` is documented as a count of read
  operations of `op_bytes` each, but `WaitReadBuffers` increments it by `io_buffers_len`, so it read
  884,956 where the backend made 55,361 calls - source wins, and no counter in the server exposes the
  call count. And `maintenance.sgml` still says a rewrite "temporarily use[s] extra disk space
  approximately equal to the size of the table", which is off by a factor of **0.0001** here
  (7,249,920 against 7,249,559,552).
- **Verification.** A purpose-built checker confirmed all **343** citations resolve, sit in range,
  cite only `raw/postgres-17/`, and carry labels consistent with their own line ranges; a second pass
  confirmed all 39 `file#Symbol` labels appear inside their cited ranges. It caught one real error - a
  `bufpage.h#PageGetHeapFreeSpace` citation pointing at `bufpage.h`'s `PageHeaderData` struct, now
  `bufpage.c#PageGetHeapFreeSpace` - plus three `freelist.c` ranges running past the file's 816 lines
  and eight labels naming an enclosing symbol the cited hunk did not contain. All 41 `## Contents`
  anchors resolve and match document order. 8 open questions filed. The checker is kept at
  `.wiki-runtime/tmp/check_citations.py`.
- Environment: 17.11 built out of tree from `raw/postgres-17/` via `git archive` into
  `.wiki-runtime/tmp/vf17/src17/`, `--without-icu --without-readline --without-zlib --enable-debug`,
  contrib installed; cluster on socket `.wiki-runtime/tmp/vf17/sock` port 55617 with
  `shared_buffers = 128MB`, `autovacuum = off`, `track_io_timing = on`, `max_wal_size = 4GB`,
  `maintenance_work_mem = 256MB`. `io_combine_limit`, `effective_io_concurrency` and
  `maintenance_io_concurrency` were left at their defaults except in the sweep. `raw/` untouched.
- `wiki/index.md`, `wiki/v17/index.md` and `wiki/versions.md` updated; the page keeps
  `verified: false` (human-only) and sets `verified_by_agent: claude-opus-5-max` with the run's
  timestamp.

## [2026-08-31] cleanup | removed the vf17 VACUUM FULL sandbox after filing the v17 page

- Removed `.wiki-runtime/tmp/vf17/` at the asker's instruction, taking `.wiki-runtime` from
  **5,218,136,443 to 10,673,516 bytes** and reclaiming **5,207,462,927 bytes** (4.85 GiB). This is
  the sandbox behind [How Much I/O a VACUUM FULL Performs on a Multi-GB, Near-Empty Heap in
  PostgreSQL 17
  (unverified)](v17/questions/storage-and-vacuum/vacuum-full-io-on-near-empty-heap.md), filed in the
  entry above. What remains under `.wiki-runtime` is `venv/`, the empty `cache/`, `indexes/`,
  `logs/` scaffold, and four small Python checkers in `tmp/`.
- **The cluster was shut down cleanly first.** `postgres` PID 160741 was still serving
  `-D .wiki-runtime/tmp/vf17/data` on socket `.wiki-runtime/tmp/vf17/sock` port 55617.
  `pg_ctl -m fast stop` returned `server stopped`, after which no `postgres` process remained and
  `data/postmaster.pid` was gone. No cluster was killed mid-write and no `rm` touched a running data
  directory.
- Disposable cluster data was 90% of it: `data/` **4,702,638,876 bytes**, holding `vf1` at 14 MB
  (the last 200-million-row fixture, already rewritten down to 885 blocks by the final `VACUUM
  FULL`) and `vf2` at 353 MB (the `io_combine_limit` sweep's small fixture). The out-of-tree
  `git archive` build tree `src17/` was 407,105,439 bytes and the exact-pin 17.11 install was
  97,212,312, so **no compiled 17.11 install now remains** and the next exact-pin experiment must
  re-`configure` and re-`make` from `raw/postgres-17/`. Peak disk use during the run was higher: four
  200-million-row fixtures were built and destroyed in sequence, each one 7,249,559,552 bytes of heap
  plus WAL.
- **The harness went with it, and this one is partly self-reproducing.** The 470,376 bytes of
  `results/` (240 files: per-run `summary.txt`, `procio.before`/`after`, `progress.samples`,
  `statio*`, `segments*`, `cmd.out`), the 20,524 bytes of `logs/`, and `bin/run_vfull.sh`,
  `bin/batch.sh`, `bin/sweep.sh`, `bin/evict.py`, `sql/00_fixture.sql`, `sql/small_fixture.sql` and
  `sql/probes.sql` are gone. The cost is lower than for some earlier sandboxes because the page
  publishes its fixture SQL in full, its counter statements, and the `posix_fadvise` eviction
  snippet under
  [How to reproduce](v17/questions/storage-and-vacuum/vacuum-full-io-on-near-empty-heap.md#how-to-reproduce);
  what would have to be re-derived is the `/proc` and progress-view sampling loop, which the page
  describes but does not publish. Every measured figure was already published in the page, and open
  question 8 records the loss.
- Kept `.wiki-runtime/tmp/check_citations.py`, the 343-citation / 41-anchor checker written for this
  page, alongside the three earlier `check_page.py`, `check_toc.py` and `dump_citations.py` helpers.
- `raw/` untouched: all five checkouts sit at their pinned commits with a zero-length
  `git status --porcelain` (`45b88269a35`, `a92fbdfb830`, `786db8dcf16`, `baa7b142aac`,
  `67342a14863`), because the build was made out of tree.
- No wiki page changed in this entry, so no verification field moved.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-31] answer v12 | heap and TOAST bloat with pgstattuple_approx, one statement, tested on 12.2

- Filed [Measuring Heap and TOAST Bloat With pgstattuple_approx in PostgreSQL 12
  (unverified)](v12/questions/storage-and-vacuum/pgstattuple-approx-heap-and-toast-bloat.md) at pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2, tag `REL_12_2`). The asker specified a
  fourteen-item requirement list for one runnable statement; this is the third `storage-and-vacuum`
  page for v12 and is linked from `wiki/index.md` and `wiki/v12/index.md`.
- Prompt hygiene: the prompt was read for typos and grammatical errors before drafting and none were
  found, so it is restated verbatim under `## Question` and no clarification was needed.
- **One requirement cannot be met as literally written on this major, and the page leads with that.**
  `pgstattuple_approx` accepts only `RELKIND_RELATION` and `RELKIND_MATVIEW`
  (`pgstatapprox.c#L280-L290`), so a TOAST relation returns
  `ERROR: "pg_toast_16405" is not a table or materialized view` - reproduced on the build. The exact
  `pgstattuple` does route `RELKIND_TOASTVALUE` to `pgstat_heap()`, so the filed statement measures
  the main heap with `pgstattuple_approx`, the TOAST heap with `pgstattuple`, and carries a
  `measured_by` column naming the function and its exactness on each row. Both functions ship in the
  same extension, so "runnable after `CREATE EXTENSION pgstattuple`" still holds.
- **"No main-table fillfactor adjustment for the TOAST heap" is source-backed, not stylistic.**
  `heap_reloptions()` overwrites `fillfactor` with 100 for every `RELKIND_TOASTVALUE` relation,
  `ALTER TABLE ... SET (toast.fillfactor = 70)` returns `ERROR: unrecognized parameter "fillfactor"`,
  the TOAST relation's `reloptions` stayed `NULL` under a parent at `{fillfactor=70}`, and
  `toast_save_datum()` inserts chunks through plain `heap_insert()`, so `saveFreeSpace` is zero.
- **Scored against `VACUUM FULL` on an isolated 12.2 server, ten fixtures, 19 measured rows.** The
  requested reserve-subtraction formula is algebraically identical to the rewrite-matching ratio form
  at `fillfactor = 100` and the two agreed to the digit on 12 of 12 such rows, landing −4.02 to +8.98
  points from the truth; on the one bloated `fillfactor = 70` table it read **34.46 against a 50.00
  truth**, a 15.54-point understatement, because it charges the reserve against the pre-rewrite size
  while `raw_heap_insert()` reserves against the new heap. The ratio form read 49.22 on the same row.
  The page keeps the asked-for column and publishes the one-line alternative beside it.
- **The new finding is on the TOAST side: free space there is mostly chunk geometry.** A
  never-deleted TOAST relation reported **19.58% free** and `VACUUM FULL` reclaimed **0.00%**. It
  reconciles to the byte from source constants: `TOAST_MAX_CHUNK_SIZE` = 1,996, measured chunk lengths
  1,996/1,996/1,996/412, a full chunk row of 2,032 bytes and a tail row of 448, so 6,544 occupied +
  40 (header + 4 line pointers) leaves `PageGetHeapFreeSpace` = 8,192 − 6,584 − 4 = **1,604** per
  page, exactly the 1,604,000 over 1,000 pages that was measured. A payload sized to an exact chunk
  multiple (3,992 bytes) read **0.24%** = 20 bytes per page, and `VACUUM FULL` reclaimed 0.00% there
  too. Live length also reconciled exactly: 3,000 x 2,032 + 1,000 x 448 = 6,544,000.
- **Also measured**: `f_toastonly` hiding 24 MB of reclaimable TOAST (75.00%) behind a healthy 200 kB
  heap; the cost split - 45 buffer accesses and **zero heap reads** for the all-visible heap branch
  against **10,006 for a 5,000-page TOAST scan**, exactly two per page because `pgstat_heap()`'s
  free-space loop re-reads each block the sequential scan just brought in, and 20.706 ms / 3,449
  physical reads against 0.404 ms / 5 for the two functions on one 27 MB heap (51x, 690x);
  `scanned_percent` truncating to **0** with 1 page of 5,000 scanned; **10,000 dead TOAST chunks
  present before any VACUUM**, because `heap_delete()` calls `toast_delete()` in the same transaction;
  `pgstattuple(0)` raising `could not open relation with OID 0`, which is why the `reltoastrelid <> 0`
  filter sits one level below the lateral call; the naive filter placement nonetheless returning 0
  rows on 12.2 because the qual is a restriction on the outer relation and is pushed to its scan;
  `pg_stat_scan_tables` sufficing for both rows while `SELECT` on the table stays denied; and the
  VACUUM warning in one table - a trailing delete let plain `VACUUM` truncate a TOAST relation
  16,384,000 -> 8,192,000 bytes, while an interior delete of the same row count changed nothing until
  `VACUUM FULL` halved both forks.
- **Integrity check**: the published SQL was extracted mechanically from the page's own Markdown with
  a script and re-run against the fixtures; it reproduced the published output including the TOAST
  relation OIDs. Every fixture was built at least twice with identical byte counts.
- **Verification**: 136 citations, 32 files, all resolving, in range, `raw/postgres-12/` only, with
  every `file#Symbol` label's symbol present inside its cited range (13 labels naming an enclosing
  function whose header sat outside the hunk were fixed by widening the range or relabelling); 27
  `## Contents` anchors resolve and match document order; 8 open questions filed, including why the
  relkind gate excludes TOAST relations at all, given that they have both maps the gate cites.
  `verified_by_agent` set to `claude-opus-5-max 2026-08-31T14:13:19Z`; `verified: false` untouched.
- Environment: 12.2 built out of tree from `raw/postgres-12/` via `git archive` into
  `.wiki-runtime/tmp/pta12/src12/`, `--without-readline --without-zlib --without-icu --enable-debug`,
  `contrib/pgstattuple` installed; cluster on socket `.wiki-runtime/tmp/pta12/sock` port 55612 with
  `shared_buffers = 256MB`, `maintenance_work_mem = 256MB`, `autovacuum = off`, `fsync = off`,
  `track_io_timing = on`. `raw/` untouched: `git status --porcelain` empty before and after. Sandbox
  retained at `.wiki-runtime/tmp/pta12/`, with the fixture and statement scripts under its `sql/`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-31] remove v12 | delete Proposing and Testing a fillfactor-Corrected pgstattuple_approx Metric for Table Heap Bloat

- Removed `wiki/v12/questions/storage-and-vacuum/pgstattuple-approx-heap-bloat.md` at the user's
  request, which named the page by its exact visible title including the ` (unverified)` state
  marker. Prompt hygiene: the request's lowercased `postgresql` and uncapitalized opening drive no
  document generation and are restated on no wiki page, so no correction was needed.
- What went with it: 83,569 bytes over 776 lines, 237 Markdown citations across 33
  `raw/postgres-12/` files, 92 `## Source References` bullets, and 9 open questions. The page was
  human-unverified and agent-unverified (`verified: false`, `verified_by_agent: not yet`).
- Removed the active page entry from `wiki/index.md` (a 2,207-character bullet) and
  `wiki/v12/index.md` (1,985 characters). Both `Storage and Vacuum` groups still carry two pages, so
  neither heading was dropped.
- Removed the two inbound `## Navigation` links, in
  `pgstattuple-approx-heap-and-toast-bloat.md` and `vacuum-full-io-on-near-empty-heap.md`.
- **One surviving page referred to the deleted page in prose, and that claim is now
  self-supporting.** The heap-and-TOAST page's "heap rows with real bloat come in 1.8 to 4.0 points
  low" item deferred the detail to the deleted page; it now cites `pgstatapprox.c:97` and
  `bufpage.h#SizeOfPageHeaderData` - the two citations that page already carries in its
  exact-versus-approximate section - plus an in-page link to that section, whose anchor matches its
  own `## Contents` entry. Both lines were re-read in `raw/postgres-12/` before filing: line 97 is
  `stat->tuple_len += BLCKSZ - freespace;` and line 216 defines `SizeOfPageHeaderData`. That page
  keeps its existing `verified_by_agent: claude-opus-5-max 2026-08-31T14:13:19Z`, because the edit
  adds verified citations and changes no claim.
- Dropped the 1,450-character clause describing the page from the v12 coverage cell in
  `wiki/versions.md` (21,469 -> 20,019 characters): the `statapprox_heap()` visibility-map skip and
  free-space-map estimate, the reloption-to-`saveFreeSpace` path, the seven-point sweep, the
  closed-form residual model, the 44.93% growth case, the cost and accuracy numbers, the
  false-positive and false-negative catalogue, and the catalog-only estimator. The cell's surviving
  storage text describes the `VACUUM FULL` I/O page and its heap-and-TOAST companion.
- Neutralized the historical Markdown links that pointed at the deleted file: 3 in
  `wiki/versions.md` (the 2026-08-11 removal, review-fix, and filing notes) and 2 in `wiki/log.md`
  (the 2026-08-11 review-fix and filing entries). They keep the page title as plain text. The one
  backticked filename mention in the 2026-08-11 removal entry stays as a historical record.
- Added a removal note to `## Coverage Notes` in `wiki/versions.md` naming what is no longer filed
  anywhere in the wiki: the three-step screen/measure/confirm procedure, the seven-point fillfactor
  sweep (uncorrected 0.44 to 91.02 on unbloated tables against corrected 0.02 to 1.79, except 10.16
  at `fillfactor = 10`), the closed-form residual model, the reloption-lowered-after-load rewrite
  that grows a table 44.93%, the 30.08-point unusable-page-tail false positive, the 84.22-point
  dropped-column blind spot, the 6.9 ms / 42-read against 484.8 ms / 137,932-read cost comparison on
  a 1.08 GB table, and the catalog-only-estimator scoring. v12 `pgstattuple_approx` coverage now runs
  through [Measuring Heap and TOAST Bloat With pgstattuple_approx in PostgreSQL 12
  (unverified)](v12/questions/storage-and-vacuum/pgstattuple-approx-heap-and-toast-bloat.md), which
  keeps the `fillfactor` reserve derivation and the reserve-subtraction-versus-ratio comparison.
- Its six outbound links went with it (`wiki/v12/index.md`, `wiki/versions.md`, `wiki/index.md`, the
  v12 codebase navigation guide, `../indexing/btree-index-bloat-core-sql-only.md`, and
  `../server-administration/database-health-checklist.md`); every one of those targets keeps inbound
  links from other pages, so the deletion created no orphan.
- No version pin, `verified:` field, or source checkout changed. `raw/postgres-12/` is still on
  `45b88269a353ad93744772791feb6d01bc7e1e42` with an empty `git status --porcelain`, and no server
  was started.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-31] answer v12 | reliability of the catalog-statistics table and TOAST bloat query, measured on 12.2

- Filed [How Reliable the Catalog-Statistics Table and TOAST Bloat Query Is in PostgreSQL 12
  (unverified)](v12/questions/storage-and-vacuum/catalog-statistics-bloat-query-reliability.md) at pin
  `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2, tag `REL_12_2`). The asker supplied the widely
  circulated `pg_stats`/`pg_class` bloat-estimate statement - the `constants`/`no_stats`/
  `null_headers`/`data_headers`/`table_estimates`/`estimates_with_toast` chain - and asked how
  reliable it is. This is the fourth `storage-and-vacuum` page for v12 and is linked from
  `wiki/index.md` and `wiki/v12/index.md`.
- **Prompt hygiene applied first.** Four issues were raised before any drafting: sentence-initial
  `in postgresql 12`, the inverted embedded question `how reliable is this sql`, the lowercased
  `sql` and `toast`, and a stray space in `bloat :`. The asker chose "correct and restate"; the
  restatement and a four-row table of exactly what changed sit under `## Question`, and the SQL body
  is reproduced byte for byte, comments and typos included, because it is the object under review.
  The asker also chose score-only (no replacement query), `pgstattuple` as ground truth, and
  deletion of the sandbox afterwards.
- **Verdict: a usable screen and a bad measurement.** Three independent readings were taken for
  every fixture - the statement itself, exact `pgstattuple` (`dead + free` over main plus TOAST
  heap), and a real `VACUUM FULL` as arbiter. Over the 21 rows where the statement and the rewrite
  both produced a number: mean absolute error **9.47 points**, **13 within 1 point**, min
  **-50.00**, max **+33.00**, 5 worse than 20 points. The asker's chosen ground truth scored
  **10.79** against the same rewrites, with a **-86.41** miss, so the arbiter column was added and
  reported beside both rather than replacing the requested one.
- **As a triage screen** against "a rewrite returns at least 10 MB and at least 25%", the published
  `WHERE` clause scored **9 true positives, 2 false positives, 5 false negatives, 9 true
  negatives** over 25 measurable fixtures.
- **Every large error is attributed to a line of the statement, with the byte arithmetic shown.**
  **-50.00** on a table deleted but neither vacuumed nor re-analyzed, against a byte-identical
  fixture that was and read 50 against 50.00. **-25.05** because
  `nullhdr2 = MAX(null_frac) * MAXALIGN(23 + 1 + n/8)` charges a whole second aligned header: 68.02
  modelled bytes per tuple against a real `t_len` of 40 (on-page 44), where `heap_form_tuple`'s real
  bitmap cost is `MAXALIGN(23 + BITMAPLEN(12)) - MAXALIGN(23)` = 8 bytes and only for rows that have
  a null. **-11.98** because `ceil(toast.reltuples / 4)` is byte-exact at 6,400-byte payloads (three
  1,996-byte chunks plus a 412-byte tail: predicted 8,192,000 bytes, rewrite delivered 8,192,000)
  but over-predicts by exactly 1.5x when a 2,100-byte payload puts 6 chunks on a page. **-9.00** on
  an inheritance parent, where `analyze_rel`'s second `relhassubclass` pass gives **4 `pg_stats`
  rows for 2 live attributes** and `pg_stats` exposes `stainherit` without filtering it, doubling
  `datawidth` to 24.00 against 12.00 on the child. **+21.00** from per-column alignment padding that
  `sum((1 - null_frac) * avg_width)` cannot see (60.00 modelled against 72.00 real, chain verified:
  `ceil(400000 * 60 / 8172) = 2937` pages = 22.945 MB against 29.211 MB actual). **+33.00** from
  dividing by `bs - 20` with no per-page floor, implying 2.982 tuples per page where 2 fit; the
  rewrite reclaimed 0.00% and, re-measured after it, `pgstattuple` still read the same 10,736,000
  free bytes. **+31.00** from having no `fillfactor` term at all, with the rewrite preserving the
  reserve through `raw_heap_insert`.
- **Five relation classes never get a number**, three of them holding 25 MB to 32 MB with 50.00% to
  74.99% reclaimable: materialized views (absent from `information_schema.columns`, which is
  `relkind IN ('r','v','f','p')`), partitioned parents (absent from `pg_stat_user_tables`, which is
  `relkind IN ('r','t','m')`), any table with one `STATISTICS 0` column, any never-analyzed table
  (`do_analyze_rel` stores column statistics only inside `if (numrows > 0)`), and empty tables. And
  **no `can_estimate = false` row can ever pass the published `WHERE` clause** - all four evaluated
  it to `NULL` - so the `UNION ALL` that appends them is dead weight in the default configuration.
- **The `no_stats` CTE's stated purpose does not fire on this major.** Its comment cites JSON, but a
  `json` column does get a `pg_statistic` row: `json` has no default `=` operator and no opclass
  (measured 0 and 0, against 1 for `jsonb`), so `std_typanalyze` falls through to
  `compute_trivial_stats`, which still records `stanullfrac` and `stawidth`. Measured
  `avg_width = 54`, `can_estimate = true`, error **+1.00**.
- **One shape where the query beats the exact function.** A dropped 500-byte column reads **92**
  against a **92.42%** rewrite (35,110,912 -> 2,662,400 bytes) while `pgstattuple` reads **6.01**,
  because `reform_and_rewrite_tuple` nulls dropped attributes and the estimator only sees live
  columns in `pg_stats`.
- **The TOAST half has a bigger problem than its divisor.** A never-vacuumed TOAST relation keeps
  `relpages = 0`, so a 31 MB table reports `table_mb = 0.203` - a factor of 155 - and because
  `expected_bytes` omits the same relation it also reports `pct_bloat = 0`, which happens to match
  the 0.00% rewrite for entirely the wrong reason. Source path: `ANALYZE` never calls
  `vac_update_relstats` on the TOAST relation, `vacuum_rel`'s own comment says so, and autovacuum's
  TOAST pass only fires on `n_dead_tuples` or wraparound, which an insert-only table never reaches.
- **Also measured**: three incompatible size bases in one report (`relpages * bs` excludes FSM, VM
  and the TOAST index, a 442,368-byte gap on one fixture, while the null branch's `pg_table_size`
  includes them, giving 31.875 against 31.453 MB for the same shape); a stale `relpages` reporting
  4.227 MB for a 25.34 MB table; the `bloat_data` duplicate-column `ambiguous` error and the fact
  that `CREATE TABLE AS` over the whole statement still works because the outer list is unique; the
  comment/code mismatch in the final `WHERE`; one `AccessExclusiveLock` on any `pg_table_size`-branch
  table killing the whole statement at `lock_timeout` while the same lock on an estimated-branch
  table does not delay it (and a 24-relation `pg_table_size` sweep leaving 0 locks held, since the
  lock is taken and released per relation); a role with `SELECT` on 1 of 26 tables getting a
  clean-looking one-row report; and run time of 36 ms at 27 tables against 224 ms at 427, where
  analyzing everything made it **slower** because the cost is `information_schema.columns`, not
  `pg_table_size` (319-line plan, 9.0 ms planning, 28.0 ms execution).
- **Integrity checks**: the unfiltered scoring variant was generated from the verbatim statement by
  script, with the shared 4,962-byte CTE prefix asserted byte-identical (SHA-256 `e523590d...`), so
  every published number comes out of the asker's own arithmetic; and both SQL snippets published
  under "How to reproduce" were extracted mechanically from the page's Markdown and re-run against
  the same server.
- **Verification**: 138 citations across 26 files, all resolving, in range, `raw/postgres-12/` only,
  with every `file#Symbol` label's symbol present inside its cited range under word-boundary
  matching (13 labels naming an enclosing function whose header sat outside the hunk were fixed by
  widening the range or relabelling; two further false passes on substring matches -
  `analyze_rel` inside `do_analyze_rel`, and `heap_multi_insert` matching a comment - were caught by
  tightening the checker and fixed by widening to the definition line). 25 `## Contents` anchors
  resolve and match document order; 8 open questions. A self-audit against the result files caught
  13 numeric or wording errors in the draft, including a page-count slip (`2938` for `2937`), an
  "eight fixtures" claim where the measurement says thirteen, a partitioned-parent size quoted in
  bytes as MB, and three run-time triples printed sorted rather than as measured.
  `verified_by_agent` set to `claude-opus-5-max 2026-08-31T16:02:19Z`; `verified: false` untouched.
- Environment: 12.2 reused the exact-pin out-of-tree install at `.wiki-runtime/tmp/pta12/pg` (built
  from `raw/postgres-12/` via `git archive`, `contrib/pgstattuple` installed) with a fresh cluster in
  a separate sandbox at `.wiki-runtime/tmp/bsql12/`, socket `.wiki-runtime/tmp/bsql12/sock` port
  55613, `shared_buffers = 256MB`, `autovacuum = off`, `fsync = off`, `track_io_timing = on`,
  `default_statistics_target = 100`. `raw/` untouched: `git status --porcelain` empty before and
  after.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-31] cleanup | removed the bsql12 bloat-query measurement sandbox

- Removed `.wiki-runtime/tmp/bsql12/` at the asker's up-front instruction, taking `.wiki-runtime`
  from **3,280,717,341 to 1,753,225,220 bytes** and reclaiming **1,527,492,121 bytes** (1.42 GiB).
  This is the sandbox behind [How Reliable the Catalog-Statistics Table and TOAST Bloat Query Is in
  PostgreSQL 12
  (unverified)](v12/questions/storage-and-vacuum/catalog-statistics-bloat-query-reliability.md),
  filed in the entry above.
- **The cluster was shut down cleanly first.** `postgres` PID 232676 was still serving
  `-D .wiki-runtime/tmp/bsql12/data` on socket `.wiki-runtime/tmp/bsql12/sock` port 55613.
  `pg_ctl -m fast stop` returned `server stopped`, after which `data/postmaster.pid` was gone and no
  matching process remained. No `rm` touched a running data directory.
- Almost all of it was the data directory: `data/` was **1,527,248,038 bytes**, holding the 26
  fixture relations (roughly 500 MB of heaps and TOAST heaps at peak, plus WAL and the 400 throwaway
  tables the cost probe created and dropped). The harness itself was small: `results/` 135,537 bytes
  across 13 files and `sql/` 45,818 bytes across 15 files.
- **This sandbox borrowed its binaries, so nothing had to be rebuilt or is now missing.** The 12.2
  install at `.wiki-runtime/tmp/pta12/pg`, built out of tree from `raw/postgres-12/` for an earlier
  page with `contrib/pgstattuple` installed, was reused as-is; `bsql12` held only its own
  `initdb`-created cluster, SQL, and results. `pta12` is untouched and still present (1.7 GiB, of
  which the reusable install is a small part).
- **What the deletion costs, and what it does not.** The page publishes the three fixture
  definitions that carry the headline structural errors (alignment padding, page flooring, and the
  two-chunk TOAST geometry) and the full ground-truth statement, both verified to run. What is gone
  is the rest of the fixture matrix DDL, the mutation script, the `mkvariant.py` derivation of the
  scoring variant from the verbatim statement, and the lock, privilege and cost probe scripts; every
  number they produced is already published in the page, and open question 8 records the harness's
  one measurable artefact (its own scratch tables appearing in the report).
- `raw/` untouched: all five checkouts sit at their pinned commits with a zero-length
  `git status --porcelain` (`45b88269a35`, `a92fbdfb830`, `786db8dcf16`, `baa7b142aac`,
  `67342a14863`), because the build was made out of tree.
- No wiki page changed in this entry, so no verification field moved.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-31] answer v17 | a bloat percentage column for the GIN census statement, measured on 17.11 and 12.2

- Fourth follow-up on [Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib
  Extensions on PostgreSQL 17
  (unverified)](v17/questions/indexing/gin-index-wasted-space-contrib.md), unchanged pin
  `786db8dcf168bd9df8f55047337525ac19118b1c` (17.11): add a bloat percentage column to the census
  statement, built from information the statement already computes.
- **Prompt hygiene first.** The asker chose "correct and restate". The original read
  `follow agents.md, in postgresql 17, for question: ... , add to the census statement a bloat
  percentage column based on the already calculated information on the statement`; the page records
  the seven corrections. Three scoping answers were taken before drafting: the column reports
  `waste + slack` rather than the honest-signal subset (dead pages + `data_slack`) or the
  everything-but-payload form; it had to be verified on 12.2 as well as 17.11; and both sandboxes
  were to be deleted after filing.
- **The change to the statement is four lines**, generated as a one-hunk diff against the text
  extracted from the page's own Markdown:
  `CASE WHEN m.version = 2 THEN round(100.0 * ((c.deleted_pages + c.new_pages) * m.bs + c.entry_slack
  + c.data_slack) / nullif(m.main_bytes, 0), 2) END AS bloat_pct`, placed after
  `live_page_slack_pct` and gated on `gin_version = 2` exactly like the slack columns.
- **Nothing else in the statement moved.** The filed text and the amended text were run against the
  same seven-index database and compared cell by cell: all **175 pre-existing cells identical**, the
  amended text adding only `bloat_pct`. `EXPLAIN (ANALYZE, BUFFERS)` read `shared hit=7568` for both,
  so the column costs no page read.
- **Two servers, both built from this repo's own pins and both deleted after filing.** 17.11 at
  `.wiki-runtime/tmp/ginpct17/` (port 55437) carried the seven published fixtures rebuilt from the
  page's own fixture SQL plus its documented three-VACUUM sequence for `f5`, in an aged database and
  a rebuilt one; 12.2 at `.wiki-runtime/tmp/ginpct12/` (port 55412) ran the same fixtures with
  `txid_current()` for `pg_current_xact_id()`, on binaries from the retained
  `.wiki-runtime/tmp/pta12/pg` install with `pageinspect` 1.7 and `pg_freespacemap` 1.2 built from
  the same tree.
- **The column reproduces the page's own scoring column exactly**: 64.12 / 75.30 / 48.05 / 49.80 /
  95.22 / 51.34 / 54.15 on `f1`-`f7`, which is the `waste+slack %` column the REINDEX tables already
  print. Scored against `REINDEX INDEX` on both servers it bounded the truth on 7 of 7, loosely:
  over-read **+5.01 to +49.80 points**.
- **It runs on 12.2 with no further edit** - the `::int` cast and the `pagesize = 0` arm the text
  already carries are what it needs there - and **178 of 182 cells matched** across the two majors,
  the four differences being `f7`'s already-documented 128-and-10-byte slack gap, which rounds away.
  Both majors' rebuilds landed on the same seven byte counts (10,117,120 / 2,605,056 / 10,117,120 /
  16,384 / 802,816 / 3,948,544 / 10,346,496), so the reclaimed column is the same on each.
- **Two of the page's own rules had to be rewritten rather than quietly contradicted.** "Report the
  three classes separately. Never publish their sum as 'bloat'." now says to keep the three classes
  beside `bloat_pct` and to publish it as an upper bound on a rebuild, never as reclaimable space;
  and the entry-slack section's closing claim that summing is "misleading" now explains what the
  single number carries. The evidence for keeping that warning loud is in the numbers:
  `f3_fresh_gin`, never churned, reads **48.05** while `REINDEX` returns nothing, and
  `f4_empty_gin`, an index over an empty table, reads **49.80** over one entry page holding 8,160
  free bytes - a floor that comes from `entrySplitPage` halving what it splits.
- **One new fixture** measures the pending-list hole the column inherits: censused with a live
  246-page pending list an index read `bloat_pct` **8.66** beside `pending_pct` **81.73** while
  `REINDEX` reclaimed **67.77%** (2,465,792 -> 794,624 bytes), byte-identical on 12.2, and read
  **44.96** straight after its own rebuild. Its SQL is published on the page.
- Rounding is documented as measured: the column rounds the summed bytes once, so `f2_pending_gin`
  prints 64.64 and 10.65 against a `bloat_pct` of **75.30**, not 75.29.
- **Verification.** The amended statement was extracted from the page's own Markdown
  (SHA-256 `0caff083a317f308b159ae8dc2092e0aeca4e41a80d29e8a19d5a97f7a4a5af3`), confirmed
  byte-identical to the tested prototype, and re-run on both servers; on 17.11 it returned the filed
  table byte for byte. All **307 citations across 48 files** resolve, sit in range and cite only
  `raw/postgres-17/`; of 96 `file#Symbol` labels, 71 name a symbol inside their hunk and 25 name the
  function that encloses it, checked mechanically. One stale label was corrected:
  `gindatapage.c#dataPlaceToPageLeaf` names no function in this checkout, and the hunk it cites is
  inside `dataBeginPlaceToPageLeaf`. All 41 `## Contents` anchors resolve and match document order.
- Open questions went **15 -> 16**: `bloat_pct` has no actionable threshold, because its floor is a
  fresh build's fill (50.16% to 72.11% across opclasses, itself a function of
  `maintenance_work_mem`), and it was scored on eight fixtures rather than the page's 27-fixture
  corpus.
- Bookkeeping: `wiki/index.md`, `wiki/v17/index.md` and the v17 coverage cell in `wiki/versions.md`
  all had their "never their sum" / "three numbers that are never summed" wording corrected and a
  paragraph added; a coverage note was appended. `verified_by_agent` moved to
  `claude-opus-5-max 2026-08-31T18:49:20Z`; `verified: false` untouched.
- `raw/` untouched: all five checkouts sit at their pinned commits with a zero-length
  `git status --porcelain`, because both builds were made out of tree.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-08-31] cleanup | removed the two GIN bloat_pct measurement sandboxes

- Removed `.wiki-runtime/tmp/ginpct17/` and `.wiki-runtime/tmp/ginpct12/` at the asker's up-front
  instruction, taking `.wiki-runtime` from **5,614,889,150 to 1,754,367,778 bytes** and reclaiming
  **3,860,521,372 bytes** (3.60 GiB): 2,003,464,146 for the 17.11 sandbox and 1,856,711,870 for the
  12.2 one. These are the sandboxes behind the `bloat_pct` entry above.
- **Both clusters were shut down cleanly first.** `pg_ctl -m fast stop` returned `server stopped`
  for the 17.11 cluster on port 55437 and for the 12.2 cluster on port 55412; afterwards neither
  `data/postmaster.pid` existed and no `postgres` process remained. No `rm` touched a running data
  directory.
- **The retained `pta12` sandbox was restored to how it was found.** The 12.2 run borrowed its
  binaries from `.wiki-runtime/tmp/pta12/pg`, which ships only `pgstattuple`, so `pageinspect` and
  `pg_freespacemap` were built from `pta12/src12/contrib` and installed there for the duration;
  `make uninstall` removed both, and that install now lists exactly the `pgstattuple` and `plpgsql`
  extension files it had before, with no `pageinspect.so` or `pg_freespacemap.so` in its `lib/`.
  `pta12` is otherwise untouched and still present.
- **What the deletion costs, and what it does not.** The page publishes the amended census statement,
  the seven fixtures it was scored on, and the new pending-list fixture, all verified to run on both
  majors, so re-measuring means rebuilding 17.11 from `raw/postgres-17/`, reusing or rebuilding a
  12.2 install, and re-running that SQL. What is gone is the harness around it: the filed-versus-
  amended cell comparison script, the `EXPLAIN` variants, and the CSV captures - every number they
  produced is already in the page.
- `raw/` untouched: all five checkouts sit at their pinned commits with a zero-length
  `git status --porcelain` (`45b88269a35`, `a92fbdfb830`, `786db8dcf16`, `baa7b142aac`,
  `67342a14863`), because both builds were made out of tree.
- No wiki page changed in this entry, so no verification field moved.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-09-01] repin v19 | 19beta3 67342a14863 -> 19beta3 135b867a530, 104 commits reviewed

- Repinned `raw/postgres-19/` to `135b867a530cac2e3796d87c852b53bef40f0077`
  (`REL_19_BETA3-130-g135b867a530`, 2026-09-01) at the asker's request to move v19 to
  the latest `REL_19_STABLE` commit. Prompt hygiene was applied first and the asker
  chose "correct and restate": *Follow AGENTS.md. For PostgreSQL 19, repin the source
  checkout to the latest `REL_19_STABLE` commit.*
- **No new release stamp.** `configure.ac` and `meson.build` still read `19beta3`, and
  upstream has no `REL_19_RC1` or `REL_19_0` tag, so the branch tip is the only
  "latest" there is. The old pin remains an ancestor: a clean forward move.
- Reviewed all 104 commits: 6 change a filed claim, 7 touch a cited file without
  changing one, 91 are unrelated.
- **REPACK page 55 -> 58 feature-scope commits** (title, Short Answer, tables, Evidence
  Map). `9bb8e16bd53` adds an `already_locked` argument to
  `repack_is_permitted_for_relation()`: the two unlocked discovery loops still skip a
  concurrently dropped OID, while `cluster_rel_recheck()` passes `true` and the function
  asserts both the held lock and that the relation cannot be missing. `f23de46e15b`
  swaps `vacuum_relation` for `qualified_name opt_name_list` in `RepackStmt`, so
  `REPACK ONLY t` and `REPACK t *` — accepted but never read by `repack.c`, which
  touches no `->inh` — are now syntax errors; `VACUUM (FULL) ONLY t` is unaffected.
  `dc8628e55d4` aligns `ddl.sgml`, `maintenance.sgml`, `monitoring.sgml` and `mvcc.sgml`
  with REPACK's existence: `ctid` moves under any table-rewriting operation,
  `pg_stat_progress_repack` is documented for all three commands, and REPACK joins the
  `ACCESS EXCLUSIVE` list with its CONCURRENTLY-only-during-swap exception.
- **Autovacuum page: two post-beta3 fixes.** `4af0528a0e4` calls
  `parallel_vacuum_propagate_shared_delay_params()` right after
  `AutoVacuumUpdateCostLimit()` in `vacuum_delay_point()`, gated on
  `AmAutoVacuumWorkerProcess()`, so a rebalanced cost limit reaches the parallel workers
  instead of waiting for a `SIGHUP` — until now they could keep spending up to twice the
  configured budget; it also adds a third TAP case and the
  `autovacuum-worker-cost-balanced` injection point. `7ddb9c41a13` gives
  `pg_stat_get_autovacuum_scores()` the TOAST-to-main reloption fallback
  `do_autovacuum()` already applied, through a pre-pass over `pg_class`, so the view
  again matches its documented promise to compute scores the way a worker does.
- **`pg_plan_advice` page 27 -> 28 module commits.** `400c810ddbd` tightens the
  `FOREIGN_JOIN` sublist test from `== 1` to `< 2`, rejecting `FOREIGN_JOIN(())`.
- **Navigation guide**: no subsystem-ownership change; every makefile, catalog rule and
  routing entry point is byte-identical. Three anchors moved: `0ab90a5c941` (join removal
  now edits `root->parse->jointree` and re-derives planner state) shortened
  `subquery_planner()` by a line, `3e8bcc8644f` (revert of `ALTER TABLE ... MERGE/SPLIT
  PARTITION(S)`) dropped `SinglePartitionSpec`, `PartitionCmd.partlist` and two
  `AlterTableType` values and moved `RawStmt` up 26 lines, and `0ec36f4ba32` added
  `test_wait_lsn` to `src/test/modules`.
- **Files verified byte-identical across the range** for the claim scopes above:
  `repack_worker.c`, `pgrepack.c`, `repack.h`, `repack_internal.h`, `ref/repack.sgml`,
  `logical.c`, `logicalctl.c`, `slot.c`, the `test_decoding` repack test, `pruneheap.c`,
  `visibilitymap.c`, `vacuumlazy.c`, `heapam*.c`, `vacuumparallel.c`, `execUtils.c`, the
  six scan nodes, `reloptions.c`, `guc_parameters.dat`, `system_views.sql`,
  `012_vm_consistency.pl`, all of `contrib/pg_plan_advice/` except the three syntax-test
  files, `pg_stash_advice`, `pgplanadvice.sgml`, `test_plan_advice`, and `plancache.c`.
- 102 citation fragments re-anchored by `scripts/repin_citations --apply --sync-labels
  --update-pinned-commit`; the three `repack.c#repack_is_permitted_for_relation`
  anchors were repointed by hand, since the old L2353-L2392 range started at the
  signature and overran into the next function's comment (now L2344-L2395, comment plus
  body). Two labels on new citations were corrected after a symbol check
  (`tablecmds.c#RangeVarCallbackMaintainsTable`, `monitoring.sgml#pg_stat_autovacuum_scores`).
  All 739 v19 citations resolve, sit in range and cite only `raw/postgres-19/`.
- Bookkeeping: `wiki/versions.md` (pin, coverage cell, coverage note), `wiki/index.md`
  and `wiki/v19/index.md` (pin, counts, new findings). `verified_by_agent` stays
  `not yet` on all four v19 pages — a repin is not a claim-by-claim re-verification —
  and `verified:` was not touched.
- `raw/` otherwise untouched: the other four checkouts sit at their pinned commits, and
  `raw/postgres-19/` has a zero-length `git status --porcelain` at the new pin.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-09-01] review v19 | pg_plan_advice page re-verified claim by claim, six corrections

- Reviewed [How pg_plan_advice Works in PostgreSQL 19, and All Its Commits
  (unverified)](v19/questions/query-planning/pg-plan-advice.md) against the unchanged pin
  `135b867a530cac2e3796d87c852b53bef40f0077` (`REL_19_BETA3-130-g135b867a530`), with corrections made
  **in place**. Prompt hygiene was applied first and the asker chose "correct and restate": *Follow
  AGENTS.md. In PostgreSQL 19, review the question page: How pg_plan_advice Works in PostgreSQL 19,
  and All Its Commits (unverified).* Scope was confirmed up front: **full claim-by-claim
  re-verification, source-only**, so no v19 server was built.
- **Mechanical checks first.** A throwaway checker under `.wiki-runtime/tmp/pparev/` plus the existing
  `.wiki-runtime/tmp/check_citations.py` validated every citation for path existence, range bounds,
  version prefix and label-versus-file agreement, and rebuilt the `## Contents` block from the
  headings: **237 citations over 42 files and 56 anchors, zero problems** before the edits, **278
  citations over 48 files** after. Front matter order, the `(unverified)` title suffix, the
  `query-planning` category and the `../../../../raw/postgres-19/` prefix depth were already correct.
- **All 56 listed commits re-verified** for hash, subject, author, author date and ancestry against
  the pinned checkout: 20 core, 28 module, 8 support, every one an ancestor of the pin.
  `git log -- contrib/pg_plan_advice/` returns **exactly the 28 module commits the page lists**, and
  the support list is complete for the other three surfaces (`pgplanadvice.sgml` has 4 commits, 2 of
  them module commits; `src/test/modules/test_plan_advice` has 5; plus the `headerscheck` exclusion).
- **Finding 1, `PGS_CONSIDER_PARTITIONWISE` is never cleared.** The page said partitionwise
  consideration "is preserved unless explicitly removed". All eight `pgs_mask` writes in the module
  are `&= ~`, and none of their masks contains that bit — it appears in `contrib/pg_plan_advice/` only
  inside a comment saying it must not be unset, because a higher-level joinrel builds partitionwise
  paths from this level's paths (`pathnodes.h:52-58`). The section now names the two moves actually
  used: clear `PGS_JOIN_ANY` to induce a partitionwise join, clear `PGS_APPEND | PGS_MERGE_APPEND` to
  prevent one, plus the scan-level form where a single-rel `PARTITIONWISE` target leaves those two as
  the only surviving scan strategies.
- **Finding 2, the prepared-statement explanation had the wrong mechanism.** The page said
  `EXPLAIN (PLAN_ADVICE) EXECUTE` shows nothing because "the plan was built earlier without that
  trigger". `PREPARE` does not plan: `PrepareQuery` parse-analyzes, rewrites, calls
  `CompleteCachedPlan` and stores the statement (`prepare.c:121-139`), so on the first `EXECUTE` the
  planner runs *inside* the EXPLAIN. It still generates nothing because the plan cache never passes an
  `ExplainState` to the planner: `ExplainExecuteQuery` calls `GetCachedPlan` without one
  (`prepare.c:634-636`), `BuildCachedPlan` goes through `pg_plan_queries` (`plancache.c:1097-1101`),
  and `pg_plan_queries` hardcodes `NULL` (`postgres.c:1012-1013`); `ExplainState` appears nowhere in
  `plancache.c`/`plancache.h`. Core's own `c83ac02e` message says the argument "won't help with
  EXPLAIN EXECUTE is used" (sic). The same `es == NULL` also suppresses feedback unless
  `feedback_warnings` or `always_store_advice_details` is on. `pt2` in the `prepared` test is the
  proof the planner does run there, and that is now stated.
- **Finding 3, the regression example was silently truncated.** The star-schema block quoted three
  generated lines from `join_order.out#L43-L46`; the block is four lines, and the fourth is
  `NO_GATHER(f d1 d2)` at L47. Extended, and used to illustrate the documented emission order, with
  the test's `max_parallel_workers_per_gather = 0` cited for why `NO_GATHER` is there.
- **Finding 4, `3ab3f33281f` conflated supplied with generated advice.** The page said the
  `alternatives` test "advises `DO_NOT_SCAN(alt_t2@exists_to_any_1)` where it previously said
  `exists_2`". The commit's diff shows the renamed *generated* targets are
  `SEQ_SCAN`/`NO_GATHER(alt_t1 alt_t2@exists_to_any_1)`, the generated
  `DO_NOT_SCAN(alt_t2@exists_1)` line is unchanged because it names the *other* alternative, and the
  `exists_to_any_1` spelling of `DO_NOT_SCAN` is *supplied* advice at `sql/alternatives.sql:25`.
- **Finding 5, an author misattribution.** `ea203d371de` is authored and committed by **Richard Guo**,
  not Robert Haas. Heading, `## Contents` entry and anchor updated. Its `53e6f51ee` master-counterpart
  claim checks out: same subject and author date, resolves in this checkout's object store, not an
  ancestor of the pin and on no branch here.
- **Finding 6, the module's TAP test was missing.** `89f5f860cc5` is mostly test: 6 added and 2
  removed lines in `pgpa_scan.c` against 83 new lines of `t/001_foreign_scan.pl`, plus
  `TAP_TESTS = 1` and `postgres_fdw` in `EXTRA_INSTALL`. Round-Trip Testing is now three layers
  instead of two and names the two exact-advice assertions (`NO_GATHER(ftab)` for a pushed-down
  single-table aggregate, `FOREIGN_JOIN((ftab ftab2)) NO_GATHER(ftab ftab2)` for a real foreign join),
  the `tsm_system_time` dependency behind the `scan` TABLESAMPLE cases, and `001_replan_regress.pl`.
- **Smaller fixes.** The Open Questions bullet attributed "no control over aggregation strategy or
  sort order" to the README's Future Work, which never mentions sort order — that is `5883ff30`'s
  commit message, and the two are now attributed separately. The joinrel hook is documented as reading
  only the trove's `REL` slice (Gather plus partitionwise), which is all
  `pgpa_planner_apply_joinrel_advice` looks at. `dc47beac` and `47c110f7` were swapped so the core list
  is chronological. `4321dcad`'s "factored out" became "filtered out" and now names
  `pgpa_filter_out_join_relids`. `4f888d0f` is identified as touching only the two `prepared` test
  files, and Álvaro Herrera's name carries its accent.
- **Confirmed unchanged**, among much else: the 20-bit `PGS_*` list and the two enforcement styles,
  `default_pgs_mask` seeded from the `enable_*` GUCs in `standard_planner`, the five hooks and where
  core calls them, the full-join merge/hash exceptions, `IndexOptInfo.disabled` and the
  disable-every-other-index loop, the 20-tag enum, the underscore occurrence-number rules and their
  three regression cases, the fixed output order and column-76 wrap, the five `PGC_USERSET` GUCs with
  their defaults, `choose_custom_plan`'s policy including `567286b762b`'s role callbacks, the nine
  `5883ff30` reviewers, `wal_level=replica` from `5dbb63fc`, `f4a4f1a7`'s `FOREIGN_SCAN`->`FOREIGN_JOIN`
  and `NESTED_LOOP_MEMOIZE` doc fixes, and the `3d2f2eb1664`/`c9bd90242db` follow-ups touching no
  module file.
- Bookkeeping: `wiki/index.md`, `wiki/v19/index.md` and the v19 coverage cell plus a dated note in
  `wiki/versions.md`. `verified_by_agent` moved from `not yet` to
  `claude-opus-5-max 2026-09-01T14:07:51Z`, since every claim on the page was re-checked against
  pinned raw source; `verified: false` was not touched.
- `raw/` untouched: all five checkouts sit at their pinned commits, and `raw/postgres-19/` has a
  zero-length `git status --porcelain` at `135b867a530`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.
