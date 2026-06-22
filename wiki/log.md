# Wiki Log

Append one entry after every scaffold change, version lifecycle event, ingest, trace, lint pass, or filed answer.

## [2026-06-22] review-fix v12 | invalid-index outcomes Source References attach-validate entry

- Continued the section-by-section review of [All Outcomes That Leave an Invalid
  Index in PostgreSQL 12, Including a Failed CREATE INDEX CONCURRENTLY
  (unverified)](v12/questions/invalid-index-outcomes.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- `## Source References`: verified every existing entry resolves to its symbol,
  then added the `ATExecAttachPartitionIdx` driver
  ([tablecmds.c:16636-16643](../raw/postgres-12/src/backend/commands/tablecmds.c#L16636-L16643))
  — the one inline citation added during this review (the
  `### 4. An incomplete partitioned index` validation-driver correction) not
  already covered by a broader function-level entry. The two other new inline
  cites (`swap-names` L1490-1492 and `dic-overview` L2058-2067) are already
  covered by the `index_concurrently_swap` (L1447-1716) and `index_drop`
  (L2007-2276) entries.
- `verified_by_agent` stays `not yet`: scoped review/fix, not a full-page
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] review-fix v12 | invalid-index outcomes Open Questions completeness wording

- Continued the section-by-section review of [All Outcomes That Leave an Invalid
  Index in PostgreSQL 12, Including a Failed CREATE INDEX CONCURRENTLY
  (unverified)](v12/questions/invalid-index-outcomes.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- `## Open Questions`: tightened the closed-set completeness paragraph on two
  points (the five-producer conclusion is unchanged and re-confirmed):
  - Same `indisvalid =` scan-method fix as Context Reviewed: the scan now also
    covers the `UpdateIndexRelation` tuple build
    (`Anum_pg_index_indisvalid` / `BoolGetDatum(isvalid)`,
    [index.c:612](../raw/postgres-12/src/backend/catalog/index.c#L612)) that the
    literal `indisvalid =` pattern skips for the born-invalid case.
  - Completed the true-writer list: it omitted the `REINDEX CONCURRENTLY` swap's
    new-index validation `newIndexForm->indisvalid = true`
    ([index.c:1532](../raw/postgres-12/src/backend/catalog/index.c#L1532)). Added
    `index_concurrently_swap`'s new index alongside `validatePartitionedIndex`,
    the CIC set-valid, and the non-concurrent reindex paths, so the scan's
    `true`-writers are fully enumerated.
- Verified the Open Questions citations resolve: `index_set_state_flags`
  ([index.c:3331-3403](../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403))
  and `heap_xlog_inplace`
  ([heapam.c:8797-8835](../raw/postgres-12/src/backend/access/heap/heapam.c#L8797-L8835)),
  and the CIC/RIC crash-recovery delegations are intact.
- `verified_by_agent` stays `not yet`: scoped review/fix, not a full-page
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] review-fix v12 | invalid-index outcomes closed-set scan methodology

- Continued the section-by-section review of [All Outcomes That Leave an Invalid
  Index in PostgreSQL 12, Including a Failed CREATE INDEX CONCURRENTLY
  (unverified)](v12/questions/invalid-index-outcomes.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- `## Context Reviewed`: tightened the closed-set methodology note. It said the
  complete set of `indisvalid` writers was "found by grepping `indisvalid =`",
  but that literal pattern does not match the born-invalid writer it lists —
  `UpdateIndexRelation` writes `values[Anum_pg_index_indisvalid - 1] =
  BoolGetDatum(isvalid)`
  ([index.c:612](../raw/postgres-12/src/backend/catalog/index.c#L612)), where
  `isvalid = !concurrent && !invalid`, so `indisvalid =` skips it. Reworded to
  scan for both direct `indisvalid =` assignments and the `UpdateIndexRelation`
  tuple build (`Anum_pg_index_indisvalid` / `BoolGetDatum(isvalid)`). The
  conclusion (five producers) is unchanged and was independently re-confirmed.
- All other Context Reviewed citations re-verified against pinned source.
- The matching "scan for `indisvalid =`" wording in `## Open Questions` will get
  the same fix when that section is reviewed.
- `verified_by_agent` stays `not yet`: scoped review/fix, not a full-page
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] review-fix v12 | invalid-index outcomes partitioned validation driver correction

- Continued the section-by-section review of [All Outcomes That Leave an Invalid
  Index in PostgreSQL 12, Including a Failed CREATE INDEX CONCURRENTLY
  (unverified)](v12/questions/invalid-index-outcomes.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- `### 4. An incomplete partitioned index`: corrected a factual error. The page
  said partitioned-parent validation "is driven by `ALTER INDEX ... ATTACH
  PARTITION` (or by `ALTER TABLE ... ATTACH PARTITION`)". The `ALTER TABLE`
  parenthetical is wrong in v12: `validatePartitionedIndex` has exactly one
  external caller, `ATExecAttachPartitionIdx` (the `ALTER INDEX ... ATTACH
  PARTITION` executor)
  ([tablecmds.c:16643](../raw/postgres-12/src/backend/commands/tablecmds.c#L16643)),
  plus its own upward recursion
  ([tablecmds.c:16764](../raw/postgres-12/src/backend/commands/tablecmds.c#L16764));
  `ALTER TABLE ... ATTACH PARTITION` runs `AttachPartitionEnsureIndexes`
  ([tablecmds.c:15771](../raw/postgres-12/src/backend/commands/tablecmds.c#L15771),
  [tablecmds.c:15872](../raw/postgres-12/src/backend/commands/tablecmds.c#L15872)),
  which never calls it. The sole `indisvalid = true` writer for a partitioned
  parent is `validatePartitionedIndex`
  ([tablecmds.c:16736](../raw/postgres-12/src/backend/commands/tablecmds.c#L16736)),
  and the docs agree only `ALTER INDEX ... ATTACH PARTITION` marks the parent
  valid
  ([create_index.sgml:684-685](../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L684-L685)).
  Removed the parenthetical and added the driver's caller citation
  ([tablecmds.c:16636-16643](../raw/postgres-12/src/backend/commands/tablecmds.c#L16636-L16643)).
  This also resolves an internal inconsistency: the page's "How to clear" section
  already named only `ALTER INDEX ... ATTACH PARTITION`.
- Rest of the partitioned-index section re-verified and unchanged: the ON ONLY
  `INDEX_CREATE_INVALID` path, the invalid-child `invalidate_parent` path, the
  `tuples == nparts` validity gate with upward recursion, the planner's
  ignore-all-partitioned-indexes skip, and the two `indexing.out` lifecycles.
- `verified_by_agent` stays `not yet`: scoped review/fix, not a full-page
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] review-fix v12 | invalid-index outcomes DROP INDEX CONCURRENTLY quote citation

- Continued the section-by-section review of [All Outcomes That Leave an Invalid
  Index in PostgreSQL 12, Including a Failed CREATE INDEX CONCURRENTLY
  (unverified)](v12/questions/invalid-index-outcomes.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- `### 3. A failed or interrupted DROP INDEX CONCURRENTLY`: the section quotes
  "more or less the reverse" and paraphrases the keep-`indisready` rationale, but
  its only inline citation
  ([index.c:2089-2192](../raw/postgres-12/src/backend/catalog/index.c#L2089-L2192))
  starts after both. Added the `index_drop` overview-comment citation
  ([index.c:2058-2067](../raw/postgres-12/src/backend/catalog/index.c#L2058-L2067)),
  which holds the verbatim "more or less the reverse process of Create Index
  Concurrently" line and the "keep indisready = true so transactions that are
  still scanning the index can continue to see valid index contents" rationale.
  No claim changed.
- Rest of the DROP INDEX CONCURRENTLY section re-verified and unchanged: the
  4-step `index_drop` concurrent sequence (clear-valid + commit, WaitForLockers,
  `index_concurrently_set_dead`/`INDEX_DROP_SET_DEAD` + commit, WaitForLockers +
  delete), the (t,t,f)->(f,f,f) states, the `RelationGetIndexList` not-live skip,
  the retryable clear-valid, and the single-transaction non-concurrent contrast.
- `verified_by_agent` stays `not yet`: scoped review/fix, not a full-page
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] review-fix v12 | invalid-index outcomes RIC swap rename citation

- Section-by-section review of [All Outcomes That Leave an Invalid Index in
  PostgreSQL 12, Including a Failed CREATE INDEX CONCURRENTLY
  (unverified)](v12/questions/invalid-index-outcomes.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`, per user
  request (review and commit one section at a time).
- `### 2. A failed or cancelled REINDEX CONCURRENTLY`: the swap sentence asserts
  `index_concurrently_swap` renames `_ccnew`/`_ccold` *and* flips `indisvalid` in
  one transaction, but cited only the validity flip
  ([index.c:1531-1537](../raw/postgres-12/src/backend/catalog/index.c#L1531-L1537)).
  Added a citation to the actual name swap
  ([index.c:1490-1492](../raw/postgres-12/src/backend/catalog/index.c#L1490-L1492),
  the two `namestrcpy` calls), so both halves of the one-transaction atomicity
  claim are cited from inside `index_concurrently_swap`. No claim changed.
- Earlier sections in this pass (Question, the five-outcome overview, the flag /
  producers / state machinery, and the CIC outcome) were re-verified against
  pinned source and needed no edit.
- `verified_by_agent` stays `not yet`: scoped review/fix, not a full-page
  re-verification; CIC/RIC crash-recovery flag state stays delegated to the
  sibling pages.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] review-fix v12 | invalid-index outcomes RIC crash-scope and citation pass

- Reviewed [All Outcomes That Leave an Invalid Index in PostgreSQL 12, Including
  a Failed CREATE INDEX CONCURRENTLY (unverified)](v12/questions/invalid-index-outcomes.md)
  against pinned `raw/postgres-12/` commit
  `45b88269a353ad93744772791feb6d01bc7e1e42`, focusing on the closed set of
  `indisvalid=false` writers and invalid-index consumers.
- Reconfirmed the closed producer set with whole-`src` scans: born-invalid
  `index_create` paths (`INDEX_CREATE_CONCURRENT` and `INDEX_CREATE_INVALID`),
  the partitioned-parent invalid-child update, RIC's `index_concurrently_swap`,
  `DROP INDEX CONCURRENTLY`'s `INDEX_DROP_CLEAR_VALID`, and `pg_upgrade`'s
  old-hash-index invalidation. No extra normal-operation producer was found.
- Fixed a scope overclaim in the RIC section: ordinary ERROR/cancel/timeout
  outcomes remain covered by the RIC failure table, but exact crash or
  `immediate`-shutdown recovered RIC flag state stays delegated to the sibling
  RIC page's Open Questions. Added supporting RIC source/doc citations.
- Tightened citation ranges for `validatePartitionedIndex`, partitioned-index
  recursion, and `index_drop`; added a missing `The five outcomes at a glance`
  heading so the Contents block matches the page; added `heap_xlog_inplace` to
  Source References for the CIC crash-recovery scope note.
- `verified_by_agent` remains `not yet`: this was a targeted review/fix pass,
  and the RIC crash-boundary state remains intentionally scoped under the RIC
  page's Open Questions rather than independently closed here.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] review v12 | all outcomes that leave an invalid index — full re-verification

- Reviewed [All Outcomes That Leave an Invalid Index in PostgreSQL 12, Including
  a Failed CREATE INDEX CONCURRENTLY (unverified)](v12/questions/invalid-index-outcomes.md)
  against pinned `raw/postgres-12/` commit
  `45b88269a353ad93744772791feb6d01bc7e1e42` — the full claim-by-claim
  re-verification the filing entry left pending.
- Re-checked every source citation's line range against the pin: `index.c`
  (`UpdateIndexRelation` flag store, `index_concurrently_swap` L1531-1537,
  `index_concurrently_set_dead`, `index_drop` concurrent branch L2089-2192,
  `validate_index` overview L3114-3168, `index_set_state_flags` L3331-3403),
  `indexcmds.c` (CIC first commit L1318-1320, partitioned `INDEX_CREATE_INVALID`
  L988-998, attach-invalid-child L1243-1265, bulk-REINDEX skip L2819-2824),
  `tablecmds.c` `validatePartitionedIndex` L16682-16768 and replica-identity
  reject, `plancat.c` invalid/partitioned skips, `pg_index.h` flags, `index.h`
  `INDEX_CREATE_INVALID`, `execIndexing.c`, `vacuum.c`, `cluster.c`, `matview.c`,
  `parse_utilcmd.c`, `relcache.c`, `check.c`, `version.c`; plus the doc
  (`create_index.sgml`) and regression (`create_index.out`, `indexing.out`)
  evidence. All ranges resolve and support their claims.
- Re-confirmed the page's central thesis (the closed set of `indisvalid = false`
  producers) with a whole-source-tree scan for `indisvalid =`: the only catalog
  `false`-writers anywhere are `index.c:612` born-invalid (CIC + partitioned
  `ON ONLY`), `index.c:1533` (RIC swap), `index.c:3381`
  (`INDEX_DROP_CLEAR_VALID`), `indexcmds.c:1260` (partitioned invalid child), and
  `version.c:369` (`pg_upgrade` hash). `index_concurrently_swap` and
  `INDEX_DROP_CLEAR_VALID` have exactly one caller each, confirming the RIC-only
  and DROP-INDEX-CONCURRENTLY-only attributions. Verified the partitioned invalid
  parent is born `(indislive,indisready,indisvalid) = (t,t,f)` from the
  `index_create` flag derivation (`invalid` true, `concurrent` false).
- Only edits were two evidence-note tightenings (no claim changed): corrected the
  Open Questions completeness note (the prior "whole-tree grep over `src/backend`
  and `src/bin/pg_upgrade`" wording) to describe the whole-source-tree scan and
  name the non-write `indisvalid =` sites, and split `validatePartitionedIndex`
  (sets `true`) out of the Context Reviewed `false`-writer list.
- `verified_by_agent` stays `not yet`: this page's body crash/`immediate`-shutdown
  claim for the CIC flag flips delegates its full WAL-redo proof to the sibling
  CIC page, whose RIC crash-boundary state is still an Open Question, so not every
  claim is independently closed on this page.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] question v12 | all outcomes that leave an invalid index

- Filed [All Outcomes That Leave an Invalid Index in PostgreSQL 12, Including a
  Failed CREATE INDEX CONCURRENTLY (unverified)](v12/questions/invalid-index-outcomes.md)
  against pinned `raw/postgres-12/` commit
  `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2).
- Prompt hygiene: the user's prompt was filed lowercase
  (`all possible outcomes that leave an invalid index, including a failed create
  index concurrently`). Per `MANDATORY Prompt Hygiene`, asked before drafting and
  the user approved a light cleanup (capitalize `CREATE INDEX CONCURRENTLY` and
  the leading word, add a period); the corrected text is restated verbatim under
  `## Question` with a prompt note.
- Scope established by whole-tree grep of `indisvalid` over `src/backend` and
  `src/bin/pg_upgrade`: the complete set of code paths that persist
  `indisvalid = false` is `UpdateIndexRelation` (CIC + partitioned-invalid),
  `index_set_state_flags(INDEX_DROP_CLEAR_VALID)`, `index_concurrently_swap`, the
  `DefineIndex` partitioned-parent invalidation, and `pg_upgrade`'s
  `old_9_6_invalidate_hash_indexes`. So five operational outcomes leave an invalid
  index: (1) a failed/cancelled/crashed `CREATE INDEX CONCURRENTLY`
  (invalid-not-ready vs invalid-ready); (2) a failed `REINDEX CONCURRENTLY`
  (invalid `_ccnew` pre-swap or `_ccold` post-swap; healthy original name never
  left invalid); (3) a failed/interrupted `DROP INDEX CONCURRENTLY` (clear-valid →
  set-dead across commits; clear-valid is retryable by design,
  `index.c:3367-3383`); (4) an incomplete partitioned parent
  (`CREATE INDEX ON ONLY` with partitions sets `INDEX_CREATE_INVALID`
  `indexcmds.c:988-998`, or attaching an invalid child `indexcmds.c:1243-1265`;
  `validatePartitionedIndex` `tablecmds.c:16682-16768` flips valid once every
  partition has a valid match); (5) `pg_upgrade` from <= 9.6 marking new-cluster
  hash indexes invalid (`version.c:367-376`, gated `check.c:218-220`, writes
  `reindex_hash.sql`).
- Added the persistent `(indislive, indisready, indisvalid)` state table (and why
  `(f,t)` is impossible via the state-flag asserts), a contrast that
  single-transaction `CREATE INDEX`/`REINDEX`/`DROP INDEX` roll back cleanly with
  no leftover, the cost of an invalid index (planner skips it `plancat.c:199-210`,
  executor still maintains it when `indisready` `execIndexing.c:330-332`, VACUUM
  still processes it `vacuum.c:1869-1880`) and the commands that reject one
  (`CLUSTER`, `REFRESH MATERIALIZED VIEW CONCURRENTLY`, `ADD CONSTRAINT ... USING
  INDEX`, `REPLICA IDENTITY`, bulk `REINDEX CONCURRENTLY`), and per-outcome
  repair. Cross-links the dedicated CIC and RIC pages instead of re-deriving them;
  regression evidence cited from `create_index.out` (CIC/RIC) and `indexing.out`
  (partitioned invalid lifecycle).
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
  `verified_by_agent: not yet` pending a full claim-by-claim re-verification pass.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] repin v19 | fetch latest master and review all questions

- Per user request ("check for new commits"), fetched upstream PostgreSQL
  `master` for `raw/postgres-19/` and repinned from
  `ff8bec8c460a13bedbb416d8697f4675a0709ce8` to
  `9a60f295bcb186a729d04e76377b7f122b2a1dd9` (16 new commits; new tip
  `9a60f295`, 2026-06-22, `Strip removed-relation references from PlaceHolderVars
  at join removal`).
- Reviewed all three v19 question pages and the mandatory codebase navigation
  guide against the changed-file range. Intersecting the 16 commits' changed
  files with every v19 citation, the only cited file that changed is
  `src/backend/postmaster/autovacuum.c` (autovacuum scoring page). The only
  changed file relevant to filed coverage is the post-`REL_19_BETA1` autovacuum
  MXID-score division-by-zero fix `1f2297b5487` (touches `autovacuum.c` and
  `maintenance.sgml`).
- Updated [PostgreSQL 19 Autovacuum and VACUUM
  (unverified)](v19/questions/autovacuum-parallel-scoring-visibility.md):
  documented the `scores->mxid = mxid_age / Max(1, multixact_freeze_max_age)`
  divisor guard (the `mxid` component degrades to the raw multixact age when
  `effective_multixact_freeze_max_age` from `MultiXactMemberFreezeThreshold()`
  reaches 0 under high multixact member-space usage) and the new member-space
  scaling note; added `1f2297b5487` to the scoring commit-history table;
  corrected the now-stale "no autovacuum feature files changed between
  REL_19_BETA1 and this pin" sentence; and shifted the four `autovacuum.c`
  citations past line 3048 (`L3064-L3078`->`L3067-L3081`,
  `L3195-L3238`->`L3198-L3243`, `L3239-L3240`->`L3244-L3245`,
  `L3287-L3315`->`L3292-L3320`). Verified the unchanged anchors below line 3048
  still resolve.
- Confirmed by `git diff` that no `pg_plan_advice`, `REPACK`, or
  visibility-map/pruning feature files changed since `REL_19_BETA1`/the old pin,
  so those pages and the navigation guide took only the `pinned_commit` bump
  (plus the in-prose checkout-hash references on the pg_plan_advice and REPACK
  pages).
- Updated `wiki/index.md`, `wiki/v19/index.md` (incl. `Repinned: 2026-06-22`),
  and `wiki/versions.md` (pin column, prose hash, and a new Coverage Notes
  entry). `verified_by_agent` remains `not yet` on all v19 pages because this
  was a repin and changed-file review, not a full claim-by-claim
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-22] review-fix v12 | EXPLAIN ANALYZE BUFFERS full-page citation pass + Contents TOC

- Reviewed [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12
  (unverified)](v12/questions/explain-analyze-buffers-output.md) section by
  section against pinned `raw/postgres-12/` commit
  `45b88269a353ad93744772791feb6d01bc7e1e42`, per user request. Verified every
  citation target and every behavioral claim against source.
- Fixed three citation defects:
  - `explain.c#ExplainOnePlan` pointed at `#L374`, which is the call site inside
    `ExplainOneQuery`, not the function. Repointed to the definition `#L466`
    (the `if (es->buffers) instrument_option |= INSTRUMENT_BUFFERS;` mapping is
    at [explain.c:484-485](../raw/postgres-12/src/backend/commands/explain.c#L484-L485)).
    4 occurrences (Short Answer, Context Reviewed, Evidence Map, Source References).
  - `execMain.c#InitResultRelInfo` pointed at `#L850`; the definition is `#L1277`
    and the trigger-instrumentation alloc is
    [execMain.c:1302-1303](../raw/postgres-12/src/backend/executor/execMain.c#L1302-L1303).
    1 occurrence (Where It Appears In The Plan).
  - `ref/explain.sgml#BUFFERS` at `#L43` is only the synopsis token
    `BUFFERS [ boolean ]`; the descriptive prose (shared/local/temp meaning,
    dirtied/written, upper-node-includes-children) is the varlistentry at
    [ref/explain.sgml:168-192](../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L168-L192).
    Moved the 6 prose claims to the existing `#BUFFERS-def`/`#L168-L192` anchor
    (What Each Field Means, the two How-The-Counters paragraphs, Evidence Map,
    Source References, Open Questions). Kept the synopsis `#BUFFERS`/`#L43` entry
    in Context Reviewed as distinct reviewed context.
- Added the mandatory `## Contents` table of contents (the page predated the TOC
  rule; this review is a substantial revision, so the migration grace no longer
  applies).
- Confirmed-correct and left unchanged: all `bufmgr.c`/`localbuf.c`/`buffile.c`
  counter increments and read/write timing
  ([bufmgr.c:736-757](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L736-L757),
  [bufmgr.c:904](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L904),
  [bufmgr.c:2769-2772](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2769-L2772),
  [bufmgr.c:1497-1500](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1497-L1500),
  [localbuf.c:227](../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L227),
  [localbuf.c:300-301](../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L300-L301),
  [buffile.c:440-441](../raw/postgres-12/src/backend/storage/file/buffile.c#L440-L441),
  [buffile.c:497](../raw/postgres-12/src/backend/storage/file/buffile.c#L497));
  `show_buffer_usage` text/non-text formatting
  ([explain.c:2867-2984](../raw/postgres-12/src/backend/commands/explain.c#L2867-L2984));
  the snapshot/diff and wrapper instrumentation
  ([instrument.c:71](../raw/postgres-12/src/backend/executor/instrument.c#L71),
  [instrument.c:97-98](../raw/postgres-12/src/backend/executor/instrument.c#L97-L98),
  [execProcnode.c:392-393](../raw/postgres-12/src/backend/executor/execProcnode.c#L392-L393),
  [execProcnode.c:455-466](../raw/postgres-12/src/backend/executor/execProcnode.c#L455-L466));
  the parallel and per-worker paths
  ([explain.c:1864-1931](../raw/postgres-12/src/backend/commands/explain.c#L1864-L1931));
  the trigger path ([execMain.c:1302-1303](../raw/postgres-12/src/backend/executor/execMain.c#L1302-L1303),
  [trigger.c:2410-2461](../raw/postgres-12/src/backend/commands/trigger.c#L2410-L2461),
  [explain.c:951-955](../raw/postgres-12/src/backend/commands/explain.c#L951-L955));
  `track_io_timing` PGC_SUSET/default-false
  ([guc.c:1402-1407](../raw/postgres-12/src/backend/utils/misc/guc.c#L1402-L1407));
  and all Tests-And-Examples claims, including the verified absence of any
  literal `Buffers:` EXPLAIN output in `expected/*.out` and isolation specs.
- `verified_by_agent` stays `not yet` and the title keeps `(unverified)`
  (`verified:` is human-only). Every claim and citation was re-checked against
  pinned source in this pass.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] open-question v18 | RLS minor-release first-appearance labels assigned

- Resolved the remaining `## Open Questions` item on [Row-Level Security (RLS) in
  PostgreSQL 18: Implementation, Performance, Settings, and Fixes Since PostgreSQL 12
  (unverified)](v18/questions/row-level-security-rls.md): the precise first minor
  release that shipped each of the 20 `### Fixes Since PostgreSQL 12` fixes. Worked
  against the unchanged pin `6cb307251c5c6261286c1566496920976640108e` over the
  unshallowed `raw/postgres-18/` checkout (62,317 commits, 685 tags).
- Method: for each fix, listed every same-subject commit across all refs
  (`git log --all --fixed-strings --grep=<subject>`), mapped each distinct
  per-branch cherry-pick to its earliest containing release tag
  (`git tag --contains <c> | grep '^REL_NN_M$' | sort -V | head -1`), and read tag
  stamp dates with `git log -1 --format=%cs <tag>`. Confirmed the back-patch reach
  in each commit message matched the same-subject commit count.
- Added a `### Minor-Release First Appearance` section and table giving, per fix,
  the master development-cycle major (a `.0`), the first minor release on every
  tracked v12+ branch grouped by stamp date, and the back-patch reach. Because
  PostgreSQL ships coordinated same-day minors across branches, a back-patched
  fix first appears in those back-branch minors (earlier than the master `.0`):
  e.g. CVE-2023-2455 (`ca73753b09`) shipped 2023-05-08 in 15.3/14.8/13.11/12.15
  and only reached the v16 line in 16.0 (2023-09-11).
- Correction surfaced and applied: the pin is a `REL_18_STABLE` commit and that
  branch forked from master at `9c5b9a280c` (2025-06-29), so two of the table's
  listed hashes are REL_18_STABLE back-patch commits, not master commits —
  `64f77c6a65` (CVE-2025-8713; master `22424953cde`) and `749f4ce4d9` (doc fix;
  master `7dc4fa91413`), both masters on the unreleased v19 line. Reworded the
  Fixes-table intro (was "the hash shown is the master-branch commit").
- Four fixes were not back-patched and first shipped in one major: `a2ab9c06ea`
  (15.0), `6572bd55b0` (17.0), `0dca5d68d7` (18.0), `cd3c45125d` (18.0).
- Updated Contents, removed the resolved open-question bullet (kept the
  curated-list-completeness item), and added Context Reviewed / Evidence Map
  entries. Refreshed the `wiki/index.md`, `wiki/v18/index.md`, and
  `wiki/versions.md` (row + new Coverage Note) summaries.
- `verified_by_agent` stays `not yet`: this was a targeted open-question
  investigation, not a full-page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] open-question v18 | RLS commit-provenance open question resolved (checkout unshallowed)

- Re-investigated and resolved the first/main `## Open Questions` item on [Row-Level
  Security (RLS) in PostgreSQL 18: Implementation, Performance, Settings, and Fixes
  Since PostgreSQL 12 (unverified)](v18/questions/row-level-security-rls.md) — the
  follow-up the prior unshallow entry (below) flagged as "unblocked but not done."
  Now that `raw/postgres-18/` carries full history (62,317 commits, 685 tags, pin
  unchanged at `6cb307251c5c6261286c1566496920976640108e` =
  `REL_18_3-113-g6cb307251c5`), local `git` is valid provenance evidence.
- Verified all 20 commit hashes in the page's `### Fixes Since PostgreSQL 12` table:
  each exists in the checkout, each commit subject matches the page's summary, each
  is an ancestor of the pinned commit (so it is present in v18), and each is a
  descendant of the v13 development stamp `615cebc94b5` (2019-07-01) — i.e. a genuine
  post-PostgreSQL-12 change. Confirmed the five RLS CVEs from the commit messages:
  CVE-2023-2455 (`ca73753b09`), CVE-2023-39418 (`c2e08b04c9`), CVE-2024-4317
  (`521a7156ab`), CVE-2024-10976 (`cd7ab57532`), CVE-2025-8713 (`64f77c6a65`).
- Corrected one inaccurate row: `d907bd0543` was described as "allowed users with
  BYPASSRLS to alter BYPASSRLS-bearing roles," but its message shows it fixed an
  over-broad check (from the original RLS commit `491c029dbc4`) that required
  superuser to change *any* property of a BYPASSRLS role; the requirement is now
  scoped to changing the BYPASSRLS attribute itself (current `AlterRole()` gates on
  `dbypassRLS`, [user.c:812-817](../raw/postgres-18/src/backend/commands/user.c#L812-L817)).
  Also relabeled the table's `Unverified fix description` column to `Fix description`.
- Completeness: searched `615cebc94b5..6cb307251c` over the core RLS files
  (`rowsecurity.c`/`.h`, `rls.c`/`.h`, `policy.c`, `pg_policy.h`) plus an
  RLS/policy/WCO/CVE keyword scan of subjects and messages. Found no significant
  standalone RLS behavioral or security fix missing from the table. The extra RLS
  commits surfaced are companion/follow-up fixes to already-listed areas
  (`2ddbfede0c6`, `3e6e86abca0`, `5102f39440f` for pg_dump policy dumping/ordering;
  `66ddac6982c` defense-in-depth paired with CVE-2025-8713), docs/message-wording/
  psql-tab-completion/cosmetic-refactor commits, and the v15 security-invoker-views
  feature (`7faa5fc84bf`, REL_15_0) the page already discusses. Absolute completeness
  of a curated "fixes" list stays a judgment call.
- Edited the RLS page: rewrote the `### Fixes Since PostgreSQL 12` intro (provenance
  now verified; back-patch caveat that the master hash's development-cycle major is
  not the first shipping release), the `d907bd0543` row, the `## Context Reviewed`
  provenance bullet, the `## Evidence Map` hash row, and the two `## Open Questions`
  bullets (removed the shallow-checkout framing; kept a calibrated completeness note
  and a refined minor-release-label note). Refreshed the shallow-caveat wording in
  the `wiki/versions.md` v18 row + a new Coverage Note, and the `wiki/index.md` and
  `wiki/v18/index.md` summaries.
- Minor-release first-appearance labels remain open: most fixes were back-patched to
  all supported branches, so the first shipping minor release would require tracing
  each back-branch cherry-pick (distinct hashes) and their tags. Did not touch the
  v18 `CREATE INDEX CONCURRENTLY` page's separate "shallow" note (different
  v17->v18-diff task).
- `verified_by_agent` stays `not yet` and the title keeps `(unverified)`: this was a
  targeted open-question investigation, not a full-page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] checkout v18 | unshallowed raw/postgres-18 source checkout (pin unchanged)

- Per user request, unshallowed the `raw/postgres-18/` evidence checkout. It was
  a depth-1, single-commit clone (1 commit, 0 tags, `.git/shallow` present),
  unlike the full `raw/postgres-12/`, `raw/postgres-17/`, and `raw/postgres-19/`
  clones. Ran `git -C raw/postgres-18 fetch --unshallow --tags origin` against
  `origin` (`https://git.postgresql.org/git/postgresql.git`); the standard
  `+refs/heads/*:refs/remotes/origin/*` refspec was already in place.
- Pin preserved (no repin): HEAD stays detached at
  `6cb307251c5c6261286c1566496920976640108e`, which `git describe --tags` now
  resolves as `REL_18_3-113-g6cb307251c5` and which is contained in
  `origin/REL_18_STABLE`. Working tree clean.
- Post-fetch state: no longer shallow (`is-shallow-repository` = `false`,
  `.git/shallow` removed); 62,317 commits reachable from HEAD (was 1); 685 tags
  (was 0), at parity with the v17/v12 checkouts.
- No wiki content changed: all 15 `wiki/v18/**` pages already carry
  `pinned_commit: 6cb307251c5c6261286c1566496920976640108e`, the commit is
  unchanged, and no source-citation line numbers shift. `raw/postgres-12/`,
  `raw/postgres-17/`, and `raw/postgres-19/` were left untouched (clean, at their
  pins) per the user's "leave as-is" choice.
- Follow-up unblocked but not done: the shallow-checkout limitation that was
  cited as the reason v18 commit provenance was "left open" (the RLS page Open
  Questions, plus the `versions.md` v18 row and the `index.md` / `v18/index.md`
  summaries) is now lifted — local `git log`/tags can verify it. That provenance
  has NOT been re-investigated here, so those "shallow" notes remain to be
  refreshed when it is.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] open-question v18 | RLS first open question — no benchmark in pinned checkout

- Investigated the first open question on [Row-Level Security (RLS) in
  PostgreSQL 18: Implementation, Performance, Settings, and Fixes Since
  PostgreSQL 12 (unverified)](v18/questions/row-level-security-rls.md) against
  pinned `raw/postgres-18/` commit
  `6cb307251c5c6261286c1566496920976640108e`: whether the checkout contains a
  benchmark that quantifies RLS overhead by policy/role/partition count or
  predicate shape.
- Conclusion: it does not. The tree's only benchmark/performance surfaces
  (`src/bin/pgbench/`, `contrib/intarray/bench/`,
  `src/interfaces/ecpg/test/performance/`, `src/test/modules/test_json_parser/`)
  have no row-security or policy references; every whole-tree `benchmark` match
  is unrelated to RLS (the `prepqual.c` hit is about TPC benchmarks needing
  OR-normalization); there are no RLS isolation specs; and `rowsecurity.sql` and
  `test_rls_hooks` verify only plan/qual shape with
  `EXPLAIN (COSTS OFF)`/`(VERBOSE, COSTS OFF)` — no `\timing`,
  `EXPLAIN (ANALYZE)`, or `pg_sleep`, and `generate_series` loads at most ~100
  data rows. The only RLS performance statements in the checkout are qualitative
  docs guidance in `ddl.sgml` (the "best-performing case"; the row-share-lock
  "performance problem").
- Rewrote the close of `### Scalability and Performance Issues` to state the
  qualitative guidance and the verified absence of a benchmark, added a
  `## Context Reviewed` search-method bullet and an `## Evidence Map` row, and
  removed the resolved first `## Open Questions` bullet (the provenance and
  minor-release-label questions remain open).
- `verified_by_agent` stays `not yet` because this was a targeted open-question
  investigation, not a full-page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review v12 | CREATE INDEX CONCURRENTLY failure-scenarios and test-coverage sections

- Reviewed the `### Failure scenarios and the outcome on the table` section
  (intro plus the eight subsections, including `#### Server crash or immediate
  shutdown` and `#### Recovery`) and the `### Test coverage` section of [How
  CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`, per user
  request.
- Verified the full citation set across both sections (~30 citations) and found
  them all accurate; no changes to the page. Spot-highlights: commit-1 boundary
  ([indexcmds.c:1318-1320](../raw/postgres-12/src/backend/commands/indexcmds.c#L1318-L1320));
  the `index_set_state_flags` SET_READY/SET_VALID assert ladder + INDEX_DROP_SET_DEAD
  ([index.c:3353-3396](../raw/postgres-12/src/backend/catalog/index.c#L3353-L3396));
  the not-ready leftover cost map — opened+`RowExclusiveLock`ed but no inserts,
  unique check skipped, still HOT-counted
  ([execIndexing.c:185-192](../raw/postgres-12/src/backend/executor/execIndexing.c#L185-L192),
  [execIndexing.c:330-332](../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L332),
  [execIndexing.c:537-539](../raw/postgres-12/src/backend/executor/execIndexing.c#L537-L539),
  [relcache.c:4861-4870](../raw/postgres-12/src/backend/utils/cache/relcache.c#L4861-L4870));
  the build vs second-scan duplicate errors
  ([tuplesort.c:4048-4056](../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4048-L4056),
  [nbtinsert.c:563-568](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L568));
  the failed-CIC session-lock release
  ([lmgr.c:356-363](../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L356-L363),
  [proc.c:772-798](../raw/postgres-12/src/backend/storage/lmgr/proc.c#L772-L798));
  and the regression saga (concur_index3 INVALID, VACUUM FULL, REINDEX repair;
  the `_ccnew` RIC repair; concur_index1/2 built concurrently)
  ([create_index.out:1369-1436](../raw/postgres-12/src/test/regress/expected/create_index.out#L1369-L1436),
  [create_index.out:2323-2358](../raw/postgres-12/src/test/regress/expected/create_index.out#L2323-L2358)).
- Crash/immediate-shutdown subsection re-verified end to end (consistent with the
  prior 2026-06-19 crash-recovery resolution): DB_SHUTDOWNED -> InRecovery
  ([xlog.c:6740-6766](../raw/postgres-12/src/backend/access/transam/xlog.c#L6740-L6766));
  `heap_inplace_update` one `XLOG_HEAP_INPLACE` + PageSetLSN, no flush, and its
  physical redo
  ([heapam.c:5746-5774](../raw/postgres-12/src/backend/access/heap/heapam.c#L5746-L5774),
  [heapam.c:8797-8835](../raw/postgres-12/src/backend/access/heap/heapam.c#L8797-L8835));
  the xidless asynchronous-commit branch even under `synchronous_commit=on`
  ([xact.c:1342-1391](../raw/postgres-12/src/backend/access/transam/xact.c#L1342-L1391),
  [xlog.c:2630-2645](../raw/postgres-12/src/backend/access/transam/xlog.c#L2630-L2645));
  `XLogFlush` through-position monotonicity
  ([xlog.c:2791-2798](../raw/postgres-12/src/backend/access/transam/xlog.c#L2791-L2798));
  per-AM build durability (B-tree `smgrimmedsync`/WAL-only-when-archiving, the
  GiST/SP-GiST/GIN build-end `log_newpage_range`, BRIN/hash buffered build)
  ([nbtsort.c:1288-1307](../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1288-L1307),
  [nbtsort.c:580](../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L580),
  [gistbuild.c:217-226](../raw/postgres-12/src/backend/access/gist/gistbuild.c#L217-L226),
  [spginsert.c:134-143](../raw/postgres-12/src/backend/access/spgist/spginsert.c#L134-L143),
  [gininsert.c:408-417](../raw/postgres-12/src/backend/access/gin/gininsert.c#L408-L417),
  [brin.c:683-709](../raw/postgres-12/src/backend/access/brin/brin.c#L683-L709)); and the
  unlogged-relation reset
  ([reinit.c:36-46](../raw/postgres-12/src/backend/storage/file/reinit.c#L36-L46),
  [xlog.c:6878-6884](../raw/postgres-12/src/backend/access/transam/xlog.c#L6878-L6884)).
- `verified_by_agent` stays `not yet`; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review v12 | CREATE INDEX CONCURRENTLY why-two-scans-three-waits section

- Reviewed the `### Why two scans and three waits` section of [How CREATE INDEX
  CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`, per user
  request.
- Verified all four citations against source and found the section accurate; no
  changes to the page. The three `validate_index`-header sub-ranges match: Wait 1
  -> no incompatible HOT update so the first build scan is safe
  ([index.c:3118-3132](../raw/postgres-12/src/backend/catalog/index.c#L3118-L3132));
  `indisready` + Wait 2 -> live writers self-insert, so `validate_index` only
  backfills earlier-unindexed tuples
  ([index.c:3134-3152](../raw/postgres-12/src/backend/catalog/index.c#L3134-L3152));
  Wait 3 -> no surviving older snapshot, so SET_VALID is safe
  ([index.c:3163-3168](../raw/postgres-12/src/backend/catalog/index.c#L3163-L3168)).
  The unique-index caveat matches the docs verbatim in substance — uniqueness
  enforced from the second scan, violations visible before availability, failed
  second scan leaves a still-enforcing invalid index
  ([ref/create_index.sgml:598-606](../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L598-L606)).
- The existing Evidence Map rows (correctness narrative; invalid-index/unique
  caveat; MVCC-backfill) already back these claims. `verified_by_agent` stays
  `not yet`; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY walsender / replication-slot Wait 3 section

- Reviewed the `### Can walsenders or replication-slot xmin holders appear in
  the Wait 3 set?` section of [How CREATE INDEX CONCURRENTLY Is Implemented in
  PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md)
  against pinned `raw/postgres-12/` commit
  `45b88269a353ad93744772791feb6d01bc7e1e42`, per user request.
- Re-verified every claim/citation across the three points and the exception:
  Point 1 — slot xmin is a global aggregated by
  `ReplicationSlotsComputeRequiredXmin`
  ([slot.c:701-742](../raw/postgres-12/src/backend/replication/slot.c#L701-L742))
  into `procArray->replication_slot_xmin`/`replication_slot_catalog_xmin`
  ([procarray.c:90-93](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L90-L93),
  [procarray.c:2982-2992](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2982-L2992)),
  which `GetCurrentVirtualXIDs` never reads (loop reads only `pgxact->xmin`,
  [procarray.c:2520-2523](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520-L2523))
  whereas `GetOldestXmin`
  ([procarray.c:1425-1441](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1425-L1441))
  and `GetSnapshotData`'s `RecentGlobalXmin`
  ([procarray.c:1727-1741](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1727-L1741))
  do. Point 2 — physical walsender writes `MyPgXact->xmin` from feedback
  ([walsender.c:2026-2065](../raw/postgres-12/src/backend/replication/walsender.c#L2026-L2065),
  comment "so that the xmin will be taken into account by GetOldestXmin") or
  clears it under a slot
  ([walsender.c:1872-1909](../raw/postgres-12/src/backend/replication/walsender.c#L1872-L1909)),
  but connects to no database (`InitPostgres` early return for
  `am_walsender && !am_db_walsender`,
  [postinit.c:841-867](../raw/postgres-12/src/backend/utils/init/postinit.c#L841-L867);
  `databaseId = InvalidOid` from
  [proc.c:394-396](../raw/postgres-12/src/backend/storage/lmgr/proc.c#L394-L396)),
  and `GetCurrentVirtualXIDs`'s same-db test
  ([procarray.c:2520](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520))
  lacks `GetOldestXmin`'s `|| proc->databaseId == 0 /* always include WalSender
  */` clause
  ([procarray.c:1348-1350](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1348-L1350)).
  Point 3 — the valid-VXID gate
  ([procarray.c:2537-2539](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2537-L2539),
  [lock.h:69-82](../raw/postgres-12/src/include/storage/lock.h#L69-L82)) plus the
  `lxid`/`xmin` clear in `ProcArrayEndTransaction`
  ([procarray.c:433-456](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L433-L456)).
  Exception — logical walsender's `SnapBuildInitialSnapshot` runs inside a
  `REPEATABLE READ` txn (asserts `XACT_REPEATABLE_READ`, sets
  `MyPgXact->xmin = snap->xmin`,
  [snapbuild.c:543-583](../raw/postgres-12/src/backend/replication/logical/snapbuild.c#L543-L583))
  and connects to a real database (`replication=database` sets
  `am_db_walsender`,
  [postmaster.c:2103-2124](../raw/postgres-12/src/backend/postmaster/postmaster.c#L2103-L2124)),
  so it is an ordinary in-database snapshot holder. Confirmed exhaustiveness by
  grepping all `(MyPgXact|pgxact)->xmin =` setters: every meaningful-value
  setter outside the physical-walsender feedback path is in-transaction
  (`GetSnapshotData`/imported/restored snapshot installs and `SnapshotResetXmin`
  in `procarray.c`/`snapmgr.c`, and `SnapBuildInitialSnapshot`'s REPEATABLE READ
  txn). The "No — not through replication's xmin-holdback machinery" conclusion
  holds.
- Fixed one precision error shared by the body and Evidence Map: both said
  `lxid` and `xmin` "are set ... as a pair" / "are set in `StartTransaction`,"
  but `StartTransaction` ([xact.c:1981-1994](../raw/postgres-12/src/backend/access/transam/xact.c#L1981-L1994))
  sets only `lxid`; `xmin` is set when a snapshot is taken (`GetSnapshotData`),
  not in `StartTransaction`. Reworded to "an ordinary backend only advertises an
  xmin from within a transaction, and that xmin is cleared together with the
  local transaction id when the transaction ends," and the Evidence Map row to
  "`lxid` is set in `StartTransaction`, and `lxid` and `xmin` are cleared
  together in `ProcArrayEndTransaction`." The load-bearing clear-together fact is
  unchanged.
- Updated `wiki/versions.md` Coverage Notes. `verified_by_agent` stays `not yet`
  because this was a scoped section review, not a full-page re-verification;
  title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY prepared-transaction writer-wait section

- Reviewed the `### Is skipping prepared transactions in the writer waits safe?`
  section of [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`, per user
  request.
- Re-verified every claim/citation in the section: the `WaitForLockersMultiple`
  "prepared xacts ... certainly aren't going to do anything anymore" comment
  ([lmgr.c:890-894](../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L890-L894),
  with `WaitForLockers` wrapping it at
  [lmgr.c:942-949](../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L942-L949));
  the `validate_index` "wait for all transactions that could have been modifying
  the table to terminate" (twice) / "Any tuples committed live after the snap
  will be inserted into the index by their originating transaction" invariant
  ([index.c:3117-3144](../raw/postgres-12/src/backend/catalog/index.c#L3117-L3144))
  and the Wait 1 HOT-safety comment
  ([indexcmds.c:1348-1364](../raw/postgres-12/src/backend/commands/indexcmds.c#L1348-L1364));
  `MarkAsPreparingGuts` giving the dummy proc the real `xid` but invalid
  `xmin`/`backendId`
  ([twophase.c:465-472](../raw/postgres-12/src/backend/access/transam/twophase.c#L465-L472));
  the `GetLockConflicts` invalid-VXID drops
  ([lock.c:2930-2936](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2930-L2936),
  [lock.c:2995-3001](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2995-L3001)),
  the prepared-lock transfer to the primary lock table
  ([lock.c:2873-2876](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2873-L2876)),
  the "debatable" note
  ([lock.c:2815-2818](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2815-L2818)),
  and the `VirtualTransactionIdIsValid` `backendId != InvalidBackendId`
  requirement
  ([lock.h:71-73](../raw/postgres-12/src/include/storage/lock.h#L71-L73)); the
  Wait 3 `xmin`/VXID filter
  ([procarray.c:2525-2539](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2525-L2539))
  and prepared-xacts-in-array note
  ([procarray.c:15-18](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L15-L18));
  the `SnapshotAny` `INSERT_IN_PROGRESS` indexing branch vs. the MVCC `else`
  branch
  ([heapam_handler.c:1429-1494](../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1429-L1494),
  [heapam_handler.c:1595-1600](../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1595-L1600));
  `FinishPreparedTransaction` doing no index work
  ([twophase.c:1455-1534](../raw/postgres-12/src/backend/access/transam/twophase.c#L1455-L1534));
  the not-ready-index insert skip
  ([execIndexing.c:330-332](../raw/postgres-12/src/backend/executor/execIndexing.c#L330-L332));
  and the non-concurrent `ShareLock`-vs-`RowExclusiveLock` contrast
  ([utility.c:1320-1321](../raw/postgres-12/src/backend/tcop/utility.c#L1320-L1321),
  [lock.c:83-86](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86)).
- Confirmed the negative claims: no "prepared" mention in
  `doc/src/sgml/ref/create_index.sgml` and no prepared/two-phase handling in
  `src/backend/commands/indexcmds.c`. The section's "Not on its own [safe]"
  conclusion holds: a write by a prepared transaction that commits after the
  build is left unindexed.
- Fixed one precision error: the writer-wait bullet read as if the prepared
  xact's invalid VXID is dropped in "both the fast-path and primary-table
  scans," but at prepare its `RowExclusiveLock` is transferred to the primary
  lock table, so it is encountered and dropped only in the primary-table scan.
  Reworded so both `GetLockConflicts` scan phases drop invalid VXIDs in general
  while noting the prepared xact hits only the primary-table one (invalid
  `backendId`). Evidence Map rows 1408-1412 were already accurate and unchanged.
- Updated `wiki/versions.md` Coverage Notes. `verified_by_agent` stays `not yet`
  because this was a scoped section review, not a full-page re-verification;
  title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY blocker matrix

- Reviewed the `### All operations that can block CREATE INDEX CONCURRENTLY`
  section of [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Re-checked the four blocking points: initial
  `ShareUpdateExclusiveLock` acquisition, the two `WaitForLockers(ShareLock)`
  writer waits, and the final same-database old-snapshot
  `WaitForOlderSnapshots` wait.
- Fixed the Wait 3 boundary wording: it collects an initial same-database VXID
  list filtered by advertised `xmin` and vacuum flags; later rechecks remove
  entries from that list but do not add new wait targets.
- Corrected the `pg_dump` worked example: a same-database dump can block Wait 3
  only when it is included by that VXID/`xmin` filter. Starting after the
  reference snapshot is not itself the source-level skip condition.
- Added portal/cursor snapshot citations for the READ COMMITTED
  idle-in-transaction case and updated Context Reviewed, Evidence Map, and
  Source References. `verified_by_agent` remains `not yet` because this was a
  scoped section review, not a full-page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review v12 | CREATE INDEX CONCURRENTLY open-question section

- Reviewed the `## Open Questions` section of [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`, per user request.
- Re-verified every crash/immediate-shutdown citation the "None for source behavior" conclusion rests on, all confirmed: `index_set_state_flags`' xidless `Assert` + `heap_inplace_update` ([index.c:3338-3403](../raw/postgres-12/src/backend/catalog/index.c#L3338-L3403)); `heap_inplace_update`'s single `XLOG_HEAP_INPLACE` + `PageSetLSN` + no `XLogFlush` ([heapam.c:5746-5774](../raw/postgres-12/src/backend/access/heap/heapam.c#L5746-L5774)) and its physical redo `heap_xlog_inplace` ([heapam.c:8797-8835](../raw/postgres-12/src/backend/access/heap/heapam.c#L8797-L8835)); the xidless async-commit branch of `RecordTransactionCommit` taken regardless of `synchronous_commit` ([xact.c:1232-1392](../raw/postgres-12/src/backend/access/transam/xact.c#L1232-L1392)) with `XLogSetAsyncXactLSN` ([xlog.c:2630-2670](../raw/postgres-12/src/backend/access/transam/xlog.c#L2630-L2670)); `XLogFlush`-through-position + SET_READY(txn2)<SET_VALID(txn4) ordering ([xlog.c:2791-2798](../raw/postgres-12/src/backend/access/transam/xlog.c#L2791-L2798), [index.c:1426-1438](../raw/postgres-12/src/backend/catalog/index.c#L1426-L1438), [indexcmds.c:1448-1463](../raw/postgres-12/src/backend/commands/indexcmds.c#L1448-L1463)); and the `state != DB_SHUTDOWNED` -> `InRecovery` trigger ([xlog.c:6740-6766](../raw/postgres-12/src/backend/access/transam/xlog.c#L6740-L6766)). The "None" conclusion holds.
- Found and closed one precision gap: the "A recovered valid index is always complete" paragraph evidenced build durability only for B-tree (outside-buffers `smgrimmedsync`). Verified the other five core AMs build through shared buffers and WAL-log their pages so the build WAL precedes SET_VALID — GiST/SP-GiST/GIN via a build-end `log_newpage_range` ([gistbuild.c:217-226](../raw/postgres-12/src/backend/access/gist/gistbuild.c#L217-L226), [spginsert.c:134-143](../raw/postgres-12/src/backend/access/spgist/spginsert.c#L134-L143), [gininsert.c:408-417](../raw/postgres-12/src/backend/access/gin/gininsert.c#L408-L417)), hash/BRIN via their buffered build path ([hash.c:129-168](../raw/postgres-12/src/backend/access/hash/hash.c#L129-L168), [hashpage.c:346-402](../raw/postgres-12/src/backend/access/hash/hashpage.c#L346-L402), [brin.c:683-709](../raw/postgres-12/src/backend/access/brin/brin.c#L683-L709)). Also noted that an **unlogged** table's heap and indexes reset together to the empty init fork on crash recovery, so a recovered valid index on one is never incomplete ([xlog.c:6878-6884](../raw/postgres-12/src/backend/access/transam/xlog.c#L6878-L6884), [xlog.c:7323-7331](../raw/postgres-12/src/backend/access/transam/xlog.c#L7323-L7331), [reinit.c:36-46](../raw/postgres-12/src/backend/storage/file/reinit.c#L36-L46)).
- Rewrote the completeness paragraph to be AM-general (general `XLogFlush`-through-position argument plus the two per-AM build-durability mechanisms and the unlogged-reset clause), split the single completeness Evidence Map row into an AM-general row plus an unlogged-reset row, added nine Source References and a Context Reviewed extension, and tightened the `## Open Questions` text accordingly while keeping the "None for source behavior" marker.
- Updated `wiki/versions.md`. `verified_by_agent` stays `not yet` because this was a scoped review/expansion, not a full-page re-verification; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] expand v12 | CREATE INDEX CONCURRENTLY crash / immediate-shutdown recovery

- Resolved the open question in [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) about the exact `pg_index` flag state after a crash / `immediate` shutdown around `INDEX_CREATE_SET_READY`, `INDEX_CREATE_SET_VALID`, and their commit boundaries, against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Source-level conclusion: a crash leaves the **same four states as an ERROR/cancel — and never a valid-but-incomplete index.** The flag flips run in XID-less transactions (`index_set_state_flags` asserts `GetTopTransactionIdIfAny() == InvalidTransactionId`) and are written by `heap_inplace_update`, which logs one `XLOG_HEAP_INPLACE` record, sets the page LSN, and does **not** `XLogFlush` ([index.c:3331-3403](../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403), [heapam.c:5746-5774](../raw/postgres-12/src/backend/access/heap/heapam.c#L5746-L5774)).
- Because the phase transaction has no XID, `RecordTransactionCommit` writes no commit record and takes the asynchronous branch — `XLogSetAsyncXactLSN`, not `XLogFlush` — regardless of `synchronous_commit`; CIC never calls `ForceSyncCommit` ([xact.c:1232-1392](../raw/postgres-12/src/backend/access/transam/xact.c#L1232-L1392), [xlog.c:2630-2670](../raw/postgres-12/src/backend/access/transam/xlog.c#L2630-L2670)). So a flip becomes durable only when the WAL writer, a checkpoint, or a later synchronous commit flushes WAL past its LSN.
- On recovery `heap_xlog_inplace` redoes the byte overwrite physically and unconditionally ([heapam.c:8797-8835](../raw/postgres-12/src/backend/access/heap/heapam.c#L8797-L8835)), so a flip can survive a phase that wrote no commit record, or be lost although `CommitTransactionCommand` returned. It is never inconsistent: `XLogFlush` flushes WAL through a position ([xlog.c:2791-2798](../raw/postgres-12/src/backend/access/transam/xlog.c#L2791-L2798)), so durable SET_VALID implies durable SET_READY (recovered `(indisready, indisvalid)` monotone, never `(f,t)`). A recovered valid index is complete because the B-tree build `smgrimmedsync`s its file before the build transaction may commit and WAL-logs pages only when `wal_level >= replica` ([nbtsort.c:1288-1307](../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1288-L1307), [nbtsort.c:580](../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L580), [rel.h:519-520](../raw/postgres-12/src/include/utils/rel.h#L519-L520)), and the validate scan's WAL precedes SET_VALID under the `FlushBuffer` WAL-before-data rule ([bufmgr.c:2712-2736](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2712-L2736)). An unclean stop runs this WAL replay automatically ([xlog.c:6740-6766](../raw/postgres-12/src/backend/access/transam/xlog.c#L6740-L6766)).
- Rewrote the `#### Server crash or immediate shutdown` subsection (added a recovered-state table), replaced the Evidence Map's single "left open" row with six rows, added a Context Reviewed entry and eight Source References, and replaced the resolved `## Open Questions` bullet with the "none for source behavior" marker.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. `verified_by_agent` stays `not yet` because this was a targeted expansion, not a full-page re-verification; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] expand v12 | CREATE INDEX CONCURRENTLY walsender / replication-slot Wait 3 safety

- Resolved the open question in [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) about whether walsenders or other non-transaction backends holding an old xmin (via `hot_standby_feedback` or replication slots) can appear in the Wait 3 (`WaitForOlderSnapshots`) set, against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Source-level conclusion: **no**, not through replication's xmin-holdback machinery. (1) A replication slot's reserved xmin/catalog_xmin is aggregated by `ReplicationSlotsComputeRequiredXmin` into the *global* `procArray->replication_slot_xmin`/`replication_slot_catalog_xmin` via `ProcArraySetReplicationSlotXmin` ([slot.c:701-742](../raw/postgres-12/src/backend/replication/slot.c#L701-L742), [procarray.c:90-93](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L90-L93), [procarray.c:2982-2992](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2982-L2992)); `GetCurrentVirtualXIDs` reads only each backend's own `pgxact->xmin` and never the slot globals ([procarray.c:2520-2523](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520-L2523)), whereas `GetOldestXmin` and `GetSnapshotData` (`RecentGlobalXmin`) do fold them in ([procarray.c:1425-1441](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1425-L1441), [procarray.c:1727-1741](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1727-L1741)).
- (2) A physical walsender that sets its own `MyPgXact->xmin` from `hot_standby_feedback` (no slot, [walsender.c:2026-2065](../raw/postgres-12/src/backend/replication/walsender.c#L2026-L2065)) is filtered out by database: `InitPostgres` returns early for `am_walsender && !am_db_walsender` without setting `MyDatabaseId`, leaving `databaseId = InvalidOid` ([postinit.c:841-867](../raw/postgres-12/src/backend/utils/init/postinit.c#L841-L867)), and `GetCurrentVirtualXIDs`'s same-db test lacks `GetOldestXmin`'s `|| proc->databaseId == 0 /* always include WalSender */` clause ([procarray.c:2520](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2520), [procarray.c:1348-1350](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1348-L1350)). With a slot it clears `MyPgXact->xmin` entirely ([walsender.c:1872-1909](../raw/postgres-12/src/backend/replication/walsender.c#L1872-L1909)).
- (3) The valid-VXID gate ([procarray.c:2537-2539](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2537-L2539), [lock.h:69-82](../raw/postgres-12/src/include/storage/lock.h#L69-L82)) requires a valid `lxid`; `lxid` and `xmin` are set in `StartTransaction` and cleared together in `ProcArrayEndTransaction` ([xact.c:1981-1994](../raw/postgres-12/src/backend/access/transam/xact.c#L1981-L1994), [procarray.c:433-456](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L433-L456)), so a non-transaction backend has neither. The only walsender that can appear is a logical one mid-`REPEATABLE READ` snapshot export (`SnapBuildInitialSnapshot`, [snapbuild.c:543-583](../raw/postgres-12/src/backend/replication/logical/snapbuild.c#L543-L583)), i.e. an ordinary in-database transaction the page already covers.
- Added a new `### Can walsenders or replication-slot xmin holders appear in the Wait 3 set?` section, a Contents entry, a cross-referenced Point 4 bullet, six Evidence Map rows, a Context Reviewed entry, and twelve Source References; removed the resolved `## Open Questions` bullet.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. `verified_by_agent` stays `not yet` because this was a targeted expansion, not a full-page re-verification; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-18] expand v12 | CREATE INDEX CONCURRENTLY prepared-transaction writer-wait safety

- Resolved the open question in [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) about whether skipping prepared transactions in Waits 1 and 2 is safe for index correctness, against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Source-level conclusion: it is **not** sufficient on its own. The `validate_index` header and the Wait 1 comment in `DefineIndex` state the invariant the writer waits enforce — index-modifying transactions must either terminate before the scans or insert their own tuples ([index.c:3117-3144](../raw/postgres-12/src/backend/catalog/index.c#L3117-L3144), [indexcmds.c:1348-1364](../raw/postgres-12/src/backend/commands/indexcmds.c#L1348-L1364)). A prepared transaction satisfies neither half.
- Traced the mechanism: `MarkAsPreparingGuts` gives the dummy proc the real in-progress `xid` but an invalid `xmin`/VXID ([twophase.c:465-472](../raw/postgres-12/src/backend/access/transam/twophase.c#L465-L472)), so `GetLockConflicts` drops it from Waits 1/2 ([lock.c:2930-2936](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2930-L2936), [lock.c:2995-3001](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2995-L3001)), `WaitForOlderSnapshots` skips it at Wait 3, the MVCC build/validate scans never see its in-progress tuples (the concurrent path never runs the `SnapshotAny` `INSERT_IN_PROGRESS` logic), and `COMMIT PREPARED` records the commit and releases locks but performs no index maintenance ([twophase.c:1455-1534](../raw/postgres-12/src/backend/access/transam/twophase.c#L1455-L1534)). Net: a write committed after the build is left unindexed.
- Added a new `### Is skipping prepared transactions in the writer waits safe?` section (with a concrete `INSERT` walkthrough table and the non-concurrent `CREATE INDEX` lock-blocking contrast), a Contents entry, five Evidence Map rows, a Context Reviewed entry, two Source References entries, and updated the Points 2-3 cross-reference. Removed the resolved `## Open Questions` bullet.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. `verified_by_agent` stays `not yet` because this was a targeted expansion, not a full-page re-verification; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-18] expand v12 | CREATE INDEX CONCURRENTLY first-build-scan tuple visibility

- Resolved the open question in [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) about the first build scan's exact tuple-visibility rule, against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Traced line-by-line into the table AM: `index_concurrently_build` sets `indexInfo->ii_Concurrent = true` and calls `index_build` -> the AM `ambuild` -> `table_index_build_scan` (B-tree/hash/GiST/SP-GiST/GIN/BRIN), which dispatches to `heapam_index_build_range_scan` with `anyvisible = false`.
- Documented the MVCC-vs-`SnapshotAny` fork: a concurrent build leaves `OldestXmin` invalid (guarded by `!ii_Concurrent`) and scans with a fresh MVCC snapshot, so `heap_getnext`/`heapgetpage` applies `HeapTupleSatisfiesVisibility` -> `HeapTupleSatisfiesMVCC` and the build loop takes the `else` ("heap_getnext did the time qual check") branch. The concurrent path therefore never enters the `HeapTupleSatisfiesVacuum` switch, never indexes `RECENTLY_DEAD` tuples, and never sets `ii_BrokenHotChain`, unlike a normal build. Noted the parallel B-tree path inherits the same MVCC snapshot from the parallel scan.
- Added a new `### The first build scan's tuple-visibility rule` section plus Contents entry, seven Evidence Map rows, Context Reviewed and Source References entries, and removed the resolved `## Open Questions` bullet.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. `verified_by_agent` stays `not yet` because this was a targeted expansion, not a full-page re-verification; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-18] expand v12 | CREATE INDEX CONCURRENTLY inter-builder interactions

- Expanded [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Resolved the open question about other concurrent index builds in the v12 snapshot wait: same-table CICs serialize at the self-conflicting `ShareUpdateExclusiveLock`; different-table CIC/RIC can still meet in the database-wide `WaitForOlderSnapshots` / `GetCurrentVirtualXIDs` old-snapshot wait; and CIC avoids mutual final-phase deadlock by dropping its reference snapshot, committing into a no-`xmin` transaction, and only then waiting.
- Added source evidence for `ReindexRelationConcurrently`'s analogous wait boundary, the `multiple-cic` isolation expected output, and same-checkout history review (`c3d09b3bd23`, `54eff5311d7`, `1dec82068b3`). Removed the resolved `## Open Questions` bullet.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. Reset `verified_by_agent` to `not yet` because this was a targeted expansion rather than a full-page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-18] question v12 | wal_sender_timeout usage and logical replication impact

- Filed [How wal_sender_timeout Is Used and What It Impacts in PostgreSQL 12 (unverified)](v12/questions/wal-sender-timeout.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Prompt hygiene: the original request had typos and grammar issues; the user approved corrected wording before filing. The corrected prompt is restated under `## Question` with a prompt note.
- Coverage includes the `PGC_USERSET` GUC definition, default and `0` disable behavior, sender-side reply tracking via `last_reply_timestamp`, `WalSndLoop` timeout activation, half-time keepalive reply requests, full-time `WalSndCheckTimeOut` disconnect, physical and logical `START_REPLICATION` paths, logical subscription apply-worker feedback, `pg_recvlogical` feedback, subscriber-side `wal_receiver_*` distinction, monitoring, synchronous replication, replication-slot WAL-retention effects, and the direct test-coverage gap.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. `verified_by_agent` remains `not yet`; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-18] question v18 | CREATE INDEX CONCURRENTLY implementation

- Filed [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 18 (unverified)](v18/questions/create-index-concurrently.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Prompt hygiene: the user's request had capitalization, spacing, and grammar issues; the user approved corrected wording before filing the page. The corrected prompt is restated under `## Question` with a prompt note.
- Coverage includes parser/utility dispatch, restrictions, four internal transactions, two heap scans, three waits, table/session `ShareUpdateExclusiveLock`, `indislive`/`indisready`/`indisvalid` state progression, invalid-index failure outcomes, and regression/isolation/progress-view coverage.
- The PostgreSQL 17 comparison section records that the core choreography is unchanged and lists v18-side deltas: `pg_index` now has TOAST storage so the final mark-valid update pushes an active snapshot, GIN can use parallel build machinery during CIC's build phase, virtual generated columns are rejected before CIC starts, and temporal `WITHOUT OVERLAPS` changed shared `DefineIndex()` plumbing without adding standalone CIC syntax.
- Updated `wiki/index.md`, `wiki/v18/index.md`, and `wiki/versions.md`. `verified_by_agent` remains `not yet`; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-18] review-fix v17 | CREATE INDEX CONCURRENTLY implementation

- Reviewed [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](v17/questions/create-index-concurrently.md) against pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa` (`REL_17_STABLE`, 17.10).
- Re-checked the `DefineIndex` concurrent branch, `WaitForOlderSnapshots`, `set_indexsafe_procflags`, `UpdateIndexRelation`/`index_create`, `index_concurrently_build`, `validate_index`, `index_set_state_flags`, `WaitForLockersMultiple`, lock conflict tables, docs, tests, and same-checkout commit history for the v12-to-v17 deltas.
- Fixed review findings: added the mandatory `## Contents` block; corrected the v12-era claim that v17 `index_set_state_flags` uses non-transactional in-place updates. v17 uses transactional `CatalogTupleUpdate` for the CIC `indisready` and `indisvalid` flips (`index.c:3440-3518`), with the PG14 change anchored to commit `83158f74d3a`; and added direct `WaitForLockersMultiple` evidence for the wait-without-taking-the-relation-lock explanation.
- Updated `wiki/index.md`, `wiki/v17/index.md`, and `wiki/versions.md`. Advanced `verified_by_agent` to `GPT-5-5-XHigh-Thinking 2026-06-18T14:10:05Z`; `verified:` stays human-only `false`, so the page title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-17] review-fix v17 | partial indexes pros/cons

- Reviewed [Pros and Cons of Partial Indexes in PostgreSQL 17 (unverified)](v17/questions/partial-indexes-pros-cons.md) against pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa` (`REL_17_STABLE`, 17.10).
- Re-checked the core partial-index, expression-index, planner predicate implication, expression-key matching, `ANALYZE`, selectivity/costing, operational-restriction, regression-test, and v12-to-v17 history claims against the pinned checkout.
- Fixed review findings: widened the partial-hash regression expected-output citation to include the `Index Scan using hash_i4_partial_index` line, and filled in missing `## Source References` entries for body citations that were already otherwise valid.
- Advanced `verified_by_agent` to `GPT-5 2026-06-17T20:10:26Z`; `verified:` stays human-only `false`, so the page title keeps `(unverified)`. Existing index and version summaries remain accurate.

## [2026-06-17] expand v17 | partial indexes — what changed from PostgreSQL 12

- Per user follow-up, added `### What changed from PostgreSQL 12` to [Pros and Cons of Partial Indexes in PostgreSQL 17 (unverified)](v17/questions/partial-indexes-pros-cons.md), against pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- Prompt hygiene: the follow-up had capitalization/grammar issues; the user approved correcting “add a section on what changed from postgresql 12” to “Add a section on what changed from PostgreSQL 12.” before updating the page.
- The new section states that the core partial-index mechanics remain recognizable from the v12 baseline (predicate in `pg_index.indpred`, build/DML predicate filtering, planner `predOK`/`predicate_implied_by()`, and `ANALYZE` predicate-filtered row-count estimates) and then lists v17-side deltas from source/history: post-v12 documentation warning against partial-index-as-partitioning, PG14 `PROC_IN_SAFE_IC` safe-wait optimization with expression/partial indexes excluded, PG15 MERGE target-relation rechecks, PG14 expression statistics without expression-index maintenance, PG15 `NULLS NOT DISTINCT`, PG16 skip of partial unique indexes for join uniqueness proofs, and the pinned v17 stable partial-hash planning fix.
- Updated `wiki/index.md`, `wiki/v17/index.md`, and `wiki/versions.md`. `verified_by_agent` remains `not yet`; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-17] question v17 | partial indexes pros, cons, statistics, and planner use

- Filed [Pros and Cons of Partial Indexes in PostgreSQL 17 (unverified)](v17/questions/partial-indexes-pros-cons.md) against pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- Covers partial-index definition and storage (`pg_index.indpred`, `IndexInfo.ii_Predicate`, `IndexOptInfo.indpred`/`predOK`), predicate validation and build/DML filtering, planner implication matching and bitmap recheck behavior, pros and cons, operational restrictions (constraint adoption, replica identity, `CLUSTER`, materialized-view concurrent refresh, `ON CONFLICT` inference), and regression coverage.
- Includes ordinary expression indexes as the baseline and partial indexes on expressions: `pg_index.indkey` zeroes plus `indexprs`, `FormIndexDatum()` expression evaluation after predicate filtering, separate planner gates for predicate implication and expression-key matching, partial expression-index statistics, partitioned-index attach coverage, and `ON CONFLICT` expression/partial inference coverage.
- Adds a dedicated statistics section: `ANALYZE` filters sampled rows through the predicate, estimates `tupleFract` and partial-index `pg_class.reltuples`, stores expression-index statistics in `pg_statistic`, and planner costing adds non-redundant predicate clauses to index selectivity while excluding partial expression-index stats from whole-relation expression estimates.
- Prompt hygiene: the original request had grammar/capitalization issues; the user approved correcting the wording before filing this page.
- Updated `wiki/index.md`, `wiki/v17/index.md`, and `wiki/versions.md`. `verified_by_agent` remains `not yet`; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-17] question v12 | partial indexes pros, cons, statistics, and planner use

- Filed [Pros and Cons of Partial Indexes in PostgreSQL 12 (unverified)](v12/questions/partial-indexes-pros-cons.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Covers partial-index definition and storage (`pg_index.indpred`, `IndexInfo.ii_Predicate`, `IndexOptInfo.indpred`/`predOK`), predicate validation and build/DML filtering, planner implication matching and bitmap recheck behavior, pros and cons, operational restrictions (constraint adoption, replica identity, `CLUSTER`, `ON CONFLICT` inference), and regression coverage.
- Follow-up coverage added partial indexes on expressions: `pg_index.indkey` zeroes plus `indexprs`, `FormIndexDatum()` expression evaluation after predicate filtering, separate planner gates for predicate implication and expression-key matching, partial expression-index statistics, and partitioned-index regression coverage for matching/non-matching expression-plus-predicate definitions.
- Second follow-up coverage added ordinary non-partial expression indexes as the baseline: syntax and docs examples, catalog storage with `indpred` null, expression-key maintenance via `FormIndexDatum()`, planner expression-key matching without predicate implication, whole-relation expression statistics from non-partial expression indexes, and unique functional/expression-index regression coverage.
- Adds a dedicated statistics section: `ANALYZE` filters sampled rows through the predicate, estimates `tupleFract` and partial-index `pg_class.reltuples`, stores expression-index statistics in `pg_statistic`, and planner costing adds non-redundant predicate clauses to index selectivity while excluding partial expression-index stats from whole-relation expression estimates.
- Prompt hygiene: the original request had grammar/capitalization issues, and both follow-ups were phrased as fragments; the user approved correcting the wording before filing/updating this page.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`. `verified_by_agent` remains `not yet`; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-14] review-fix v12 | codebase navigation guide coverage gaps

- Fixed review findings in [PostgreSQL 12 Codebase Navigation Guide (unverified)](v12/codebase-navigation-guide.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Added the extended-query protocol entry path: `PostgresMain()` Parse/Bind/Execute dispatch into `exec_parse_message()`, `exec_bind_message()`, and `exec_execute_message()`, including the `CachedPlanSource`/`GetCachedPlan()`/portal handoff and matching abort-state checks.
- Added the core extension-command boundary for contrib work: `src/backend/commands/extension.c` control-file parsing, `CreateExtensionInternal()` version/script selection and `pg_extension` insertion, and `execute_sql_string()` parsing, rewriting, planning, and executing extension script commands.
- Tightened the utility evidence map by splitting simple-query vs extended-query evidence and citing direct `standard_ProcessUtility()` `EXPLAIN`/`SET` dispatch separately from `ProcessUtilitySlow()` DDL dispatch.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md` Coverage Notes. `verified_by_agent` remains `not yet` because this was a targeted coverage fix, not a full page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-14] scaffold | codebase guide follows question document rules

- Updated `AGENTS.md` so `type: codebase-navigation-guide` keeps its root-level `wiki/vNN/codebase-navigation-guide.md` path but otherwise follows all question-document rules.
- Updated `scripts/wiki_lint` to treat `type: codebase-navigation-guide` as a question-style type: `verified: false`, ordered front matter, non-empty `## Question`, non-empty inline `## Answer`, and source references are now enforced.
- Converted all supported-version guides ([v12](v12/codebase-navigation-guide.md), [v17](v17/codebase-navigation-guide.md), [v18](v18/codebase-navigation-guide.md), [v19](v19/codebase-navigation-guide.md)) from `## Guide` to `## Question` plus inline `## Answer`; the v17/v18/v19 pages use the canonical mandatory guide prompt for their version.
- Updated `wiki/index.md`, all supported version landing pages, and `wiki/versions.md` Coverage Notes.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-14] scaffold | mandatory codebase navigation guides

- Updated `AGENTS.md` to make `wiki/vNN/codebase-navigation-guide.md` mandatory for every supported version, with `type: codebase-navigation-guide`, root-level placement under `wiki/vNN/`, its own front matter shape, root-level citation depth, and required links from the global and version indexes.
- Updated `scripts/wiki_lint` to recognize the root-level guide as a version-local managed page, require source references for it, require one guide per supported version, validate its path/type pairing, and allow only `index.md` plus `codebase-navigation-guide.md` at each version root.
- Converted [PostgreSQL 12 Codebase Navigation Guide (unverified)](v12/codebase-navigation-guide.md) from a question-path page into the root-level guide type, adjusted citation depth from `../../../raw/postgres-12/...` to `../../raw/postgres-12/...`, and removed the `## Question` / `## Answer` wrapper.
- Added [PostgreSQL 17 Codebase Navigation Guide (unverified)](v17/codebase-navigation-guide.md), [PostgreSQL 18 Codebase Navigation Guide (unverified)](v18/codebase-navigation-guide.md), and [PostgreSQL 19 Codebase Navigation Guide (unverified)](v19/codebase-navigation-guide.md), each citing only its matching pinned checkout.
- Updated `wiki/index.md`, all supported version landing pages, and `wiki/versions.md` Coverage Notes.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-14] question v12 | codebase navigation guide

- Filed [PostgreSQL 12 Codebase Navigation Guide (unverified)](v12/codebase-navigation-guide.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_STABLE`, 12.2).
- Prompt hygiene: the user approved correcting "create a codebase navigation guide document" to "Create a codebase navigation guide document."; the corrected prompt is restated under `## Question` with an inline prompt note.
- The guide maps the v12 source tree from source evidence only: top-level and backend makefile layout, SQL statement flow through `PostgresMain` / `exec_simple_query` / parser / analyzer / rewrite / planner / portal / executor, utility-command dispatch through `ProcessUtility` and `ProcessUtilitySlow`, generated catalog/parser/GUC/header artifacts, key structs (`RawStmt`, `Query`, `PlannedStmt`, `RelOptInfo`, `Path`, `QueryDesc`, `EState`, `PlanState`, `RelationData`, `TableAmRoutine`, `IndexAmRoutine`, `MemoryContextData`), contrib extension boundaries, and regression/isolation/TAP/docs surfaces.
- Filed as `verified_by_agent: not yet`; title keeps `(unverified)`. Added a full `## Contents` table of contents, `## Context Reviewed`, `## Evidence Map`, `## Open Questions`, `## Source References`, and `## Navigation`.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md` Coverage Notes.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] expand v17 | REINDEX INDEX CONCURRENTLY — can a failure leave the original index name invalid?

- Per a user follow-up, added a `### Can a failure leave an invalid index with the original index name?` section to [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](v17/questions/reindex-index-concurrently.md), placed right after `### Failure scenarios and the outcome on the table`. The v17 companion to the same v12 RIC section. All evidence cites pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa` (`REL_17_STABLE`, 17.10).
- Source-confirmed answer: for a **healthy** index a RIC failure never leaves the original `index_name` invalid, so a bloated-but-valid index never degrades to no usable index. Phase 4's swap is the only step that touches the original's name or `indisvalid`; `index_concurrently_swap` renames `_ccnew`→`index_name` / original→`index_name_ccold` (`index.c:1591-1593`) and flips `new.indisvalid=true`/`old.indisvalid=false` in the same call via transactional `CatalogTupleUpdate` (`index.c:1642-1648`), inside one phase-4 transaction (`indexcmds.c:4102-4163`; intent comment `:4096-4100`). Pre-swap failure → untouched still-valid (possibly bloated) original + invalid `_ccnew`; post-swap failure → rebuilt valid `index_name` + invalid `_ccold`. The invalid leftover is always the differently-named copy, never the bare `index_name`; the planner skips only `!indisvalid` indexes (`plancat.c:260-266`).
- Recorded the v17-specific point (vs the v12 section): **all** `index_set_state_flags` writes are transactional `CatalogTupleUpdate` in v17 (`index.c:3449-3521`), not v12's in-place `heap_inplace_update`, so each phase's catalog effect commits/rolls back atomically; cross-linked to the existing `## Open Questions` crash note rather than re-asserting crash-recovery state.
- Documented the sole exception: `index_name` stays invalid after a RIC failure only when it was already invalid before RIC ran (the repair case), staged by the regression suite — `concur_reindex_ind5` made invalid by a failed CIC, then `REINDEX INDEX CONCURRENTLY` failing its `_ccnew` build leaves both `INVALID`, dropped + repaired afterward (`create_index.out:2716-2734` both-invalid, `:2736-2760` repair).
- Prompt hygiene: the user's follow-up had grammar/phrasing issues; the user chose to correct, so the filed `## Question` follow-up uses corrected wording (meaning preserved) with an inline correction note.
- Added the new subsection to the `## Contents` TOC (slug `#can-a-failure-leave-an-invalid-index-with-the-original-index-name`, hand-checked to resolve, along with the new in-section `#failure-scenarios-and-the-outcome-on-the-table`, `#what-changed-from-postgresql-12`, and `#open-questions` links) and one `## Evidence Map` row.
- Re-verified this session against the pinned checkout: the swap rename + validity flip and the single phase-4 transaction with its intent comment, `index_set_state_flags` ending in `CatalogTupleUpdate`, `plancat` skip-invalid, and the `concur_reindex_ind5` both-invalid / repair regression output. Advanced `verified_by_agent` to `claude-opus-4-8 2026-06-12T20:57:06Z` (was `claude-opus-4-8 2026-06-12T20:49:47Z`). `verified:` stays human-only `false`; title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/v17/index.md`, and `wiki/versions.md` (v17 row clause + new Coverage Note).
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] question v17 | REINDEX INDEX CONCURRENTLY — implementation, steps, locks, failure scenarios, and changes since v12

- Filed [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](v17/questions/reindex-index-concurrently.md) against pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa` (`REL_17_STABLE`, 17.10). It is the v17 companion to the v12 RIC page and the v17 CIC page.
- Prompt hygiene: the user's prompt had grammar/capitalization issues ("what change since postgresql 12"); per the user's choice the filed `## Question` uses corrected wording ("what changed since PostgreSQL 12", capitalized PostgreSQL) with a one-line correction note. Meaning preserved.
- Mechanics covered with v17 citations: dispatch via `ExecReindex`/`ReindexParams` (`indexcmds.c:2693-2781`), recovery prevention via `ClassifyUtilityCommandAsReadOnly` returning only `COMMAND_OK_IN_READ_ONLY_TXN` for `ReindexStmt` (`utility.c:280-296`, `:581-582`); `ReindexIndex`/`ReindexTable` three-way dispatch (partitioned→`ReindexPartitions`, concurrent→`ReindexRelationConcurrently`, temp→non-concurrent); the six phases and five waits of `ReindexRelationConcurrently` (`indexcmds.c:3437-4295`); the atomic phase-4 `index_concurrently_swap` flipping new-valid/old-invalid via transactional `CatalogTupleUpdate` (`index.c:1642-1648`) plus name/constraint/dep moves and `pgstat_copy_relation_stats`+`CopyStatistics` (`index.c:1794-1798`); session+transaction `ShareUpdateExclusiveLock` on heap/old/new index; the `LockConflicts` rationale (`ShareLock` waits out writers, `AccessExclusiveLock` waits out readers); the restriction set; the `indislive`/`indisready`/`indisvalid` progression on both indexes; per-phase failure outcomes with planner/relcache/executor handling of invalid-but-ready leftovers (`plancat.c:260-266`, `relcache.c:4800-4869`, `execIndexing.c:356-359`); and the progress view (`WAIT_5` now emitted in phase 6).
- "What changed from PostgreSQL 12" section, each attributed to a commit present in the checkout and first-release-tagged via `git tag --contains REL_NN_0`: partitioned REINDEX (incl. CONCURRENTLY) support via `ReindexPartitions`→`ReindexMultipleInternal` leaf recursion (`a6642b3ae06`, REL_14_0; v12 errored "not yet implemented for partitioned indexes"), `REINDEX (TABLESPACE ...)` (`c5b286047cd`, REL_14_0), `index_set_state_flags` made transactional replacing v12's `heap_inplace_update` (`83158f74d3a`, REL_14_0), `PROC_IN_SAFE_IC` extended to RIC (`f9900df5`, REL_14_0), the progress `WAIT_5` "waiting for readers before dropping" fix (`e66bcfb4c66`, REL_14_0; v12 wrongly reused `WAIT_4`), the v15 shared-memory stats rework (`8ea7963fc74`, REL_15_0), and the v17 `MAINTAIN` privilege (`ecb0fd33720`, REL_17_0); plus the v13 swap fixes `d80be6f2f6c`/`68ac9cf2499`. All deltas re-derived directly from `raw/postgres-17/` source (v12 baseline confirmed in `raw/postgres-12/`, e.g. `index.c:3400` `heap_inplace_update`).
- Recorded under `## Open Questions` that the sibling v17 CIC page still describes `index_set_state_flags` as non-transactional (v12 behavior); v17 source uses `CatalogTupleUpdate` (`index.c:3445-3518`). Flagged for reconciliation when the CIC page is next revised; did not edit that page in this session.
- Page opens with a `## Contents` TOC (H2+H3 only; the six `####` phase headings excluded, matching the v12 RIC/CIC precedent); all 16 anchors hand-checked under the VS Code slugifier. Escaped a literal pipe in one Evidence Map cell to protect the table.
- A background exploration subagent misreported `raw/postgres-17/` as missing (sandbox/permission denial); verified directly that the checkout exists and HEAD matches the pin, then did the full deep inquiry myself.
- Updated `wiki/index.md`, `wiki/v17/index.md`, and `wiki/versions.md` (v17 row clause + new Coverage Note). `verified:` stays human-only `false`; title keeps `(unverified)`.

## [2026-06-12] expand v12 | REINDEX INDEX CONCURRENTLY — can a failure leave the original index name invalid?

- Per a user follow-up, added a `### Can a failure leave an invalid index with the original index name?` section to [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/reindex-index-concurrently.md), placed right after `### Failure scenarios and the outcome on the table`. All evidence cites pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Source-confirmed answer: for a **healthy** index a RIC failure never leaves the original `index_name` invalid. Phase 4's swap is the only step that touches the original's name or `indisvalid`; `index_concurrently_swap` renames `_ccnew`→`index_name` / original→`index_name_ccold` and flips new-valid/old-invalid in the same call via transactional `CatalogTupleUpdate` (`index.c:1490-1492`, `:1531-1537`), inside a single phase-4 transaction (`indexcmds.c:3212-3261`; intent comment `:3207-3209`). So a pre-swap failure leaves the untouched (still-valid, possibly bloated) original plus an invalid `_ccnew`; a post-swap failure leaves the rebuilt valid `index_name` plus an invalid `_ccold`. The invalid leftover is always the differently-named copy, never the bare `index_name`, so a bloated-but-valid index never degrades to no usable index. The planner skips only `!indisvalid` indexes (`plancat.c:199-210`).
- Documented the sole exception: `index_name` is left invalid after a RIC failure only when it was already invalid before RIC ran (the repair case), staged by the regression suite — `concur_reindex_ind5` made invalid by a failed CIC, then `REINDEX INDEX CONCURRENTLY` failing its `_ccnew` build leaves both `INVALID`, dropped + repaired afterward (`create_index.out:2317-2358`). Scoped crash/immediate-shutdown flag state to the existing Open Question (the swap flip is transactional, but phase-2 set-ready and phase-5 set-dead use the non-transactional `index_set_state_flags`/`heap_inplace_update` path).
- Prompt hygiene: the user's follow-up had grammar/phrasing issues; per the user's choice the filed `## Question` follow-up uses corrected wording (meaning preserved), with an inline note recording the correction.
- Added the new subsection to the `## Contents` TOC (slug `#can-a-failure-leave-an-invalid-index-with-the-original-index-name`, hand-checked to resolve, as are the new in-section `#failure-scenarios-...` and `#open-questions` links) and two `## Evidence Map` rows.
- Re-verified this session against the pinned checkout: the swap rename/validity and single phase-4 transaction, phases 2/3/5/6, `index_concurrently_set_dead` + `index_set_state_flags`, `RelationGetIndexList` `indislive`, `plancat` skip-invalid, the `LockConflicts` table, dispatch/`ReindexIndex`, the restriction cluster (system/invalid/exclusion/toast/index-allowed), session locks, the drop concurrent-lock-mode dispatch, and the stats copy; the remaining peripheral citations retain the same-day verification against the same immutable pin.
- Advanced `verified_by_agent` to `claude-opus-4-8 2026-06-12T20:15:00Z` (was `claude-opus-4-8 2026-06-12T17:41:08Z`). `verified:` stays human-only `false`; title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md` (new Coverage Note).
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] review-fix v12 | CREATE INDEX CONCURRENTLY — regression-repair citation correction

- Independently re-reviewed [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) claim by claim against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Fixed one correctness error in the "Worked regression examples" → "Repairing the leftover" bullet: it claimed the invalid index "is fixed with a non-concurrent `REINDEX INDEX`" and cited `create_index.out:2335-2350`, but that range is the `REINDEX INDEX CONCURRENTLY concur_reindex_ind5` example (repair command at `:2351`, valid at `:2358`) — there is no non-concurrent `REINDEX INDEX` repair there. The genuine non-concurrent repair in the suite is `REINDEX TABLE concur_heap` (`:1422`), which clears `concur_index3`'s `INVALID` marker (`:1431`). Reworded to cite `REINDEX TABLE` (`:1422-1436`) for the non-concurrent path and `:2323-2358` for the concurrent path.
- Tightened two "repaired"/"fixed by REINDEX" citations so the repair outcome falls inside the cited range: the Test coverage bullet's `concurrent-invalid` reference now spans `:1382-1436` (was `:1382-1417`, which stopped before the `:1422` repair), and the matching Evidence Map row now also cites `:1422-1436` and is reworded from "fixed by REINDEX" to "made valid by non-concurrent `REINDEX TABLE` once the duplicate is deleted". Updated the `create_index.out#concurrent-invalid` Source Reference range to match.
- Re-verified every other claim against source and found it accurate: lockmode/dispatch (`utility.c:1307-1326`, `indexcmds.c:563-564`), the four-transaction concurrent path (`indexcmds.c:974-1014`, `:1307-1472`), the three waits (`WaitForLockers`/`ShareLock` `lock.c:83-86`, `WaitForOlderSnapshots` `indexcmds.c:339-402`), the `LockConflicts` table incl. `ShareLock` not self-conflicting and SUE self-conflict (`lock.c:65-105`, `:194-196`), state flags + `index_set_state_flags` non-transactional in-place WAL overwrite (`index.c:612-615`, `:3316-3403`, `heapam.c:5692-5774`), the failure-state machine and per-leftover costs (`plancat.c:200-210`, `execIndexing.c:185-539`, `relcache.c:4388-4395`/`4861-4870`, `tuplesort.c:4048-4056`, `nbtinsert.c:563-568`), the blocking-points evidence (`GetLockConflicts`/prepared `lock.c:2804-2818`/`2873-2876`, `lmgr.c:850-949`, autovacuum-cancel `proc.c:1308-1375`, `procarray.c` `GetCurrentVirtualXIDs` filters `:2467-2548`, `pg_dump.c:1140-1194`/`6646-6671`), and the docs (`mvcc.sgml:890-1030`, `monitoring.sgml:3630-3709`, `create_index.sgml:545-631`).
- Advanced `verified_by_agent` to `claude-opus-4-8 2026-06-12T17:55:45Z` (was `not yet`): every asserted claim was re-checked against the pinned checkout, and the genuinely-uncertain items (crash-recovery flag state, prepared-xact writer-wait safety, walsender old-xmin appearance, inter-CIC interaction, first-scan visibility) remain scoped under `## Open Questions`. `verified:` stays human-only `false`; title keeps `(unverified)`.
- No `##`/`###` heading was added/removed/renamed/reordered, so the `## Contents` table of contents is unchanged. Coverage is unchanged (the documented `DROP INDEX` / `REINDEX INDEX CONCURRENTLY` recovery summary already matches), so `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` were left as-is.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] review-fix v12 | REINDEX INDEX CONCURRENTLY — ShareLock conflict-set correction

- Independently re-reviewed [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/reindex-index-concurrently.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`, claim by claim.
- Fixed one correctness error in the Evidence Map: the `ShareLock` (Waits 1/2) row listed `ShareLock` itself in the conflict set, but `LockConflicts[ShareLock]` (`lock.c:83-86`) is `{RowExclusiveLock, ShareUpdateExclusiveLock, ShareRowExclusiveLock, ExclusiveLock, AccessExclusiveLock}` — `ShareLock` is not self-conflicting, so a held `ShareLock` is not waited out. Removed `ShareLock` from the list and added a parenthetical noting it is not self-conflicting. The body text (Wait 1/2 prose and the steps-and-locks table) never made this error.
- Re-verified everything else against source and found it accurate: the six phases and phase-overview comment (`indexcmds.c:2941-2955`); dispatch/restrictions (`utility.c:773-807`, `ReindexIndex`/`ReindexTable`/`ReindexMultipleTables`/`ReindexPartitionedIndex`, system-catalog direct error vs schema/database skip, partitioned-table skip vs partitioned-index error, exclusion error-vs-skip, invalid skip-vs-allow, temp fallback); the `index.c` helpers with exact line numbers (`index_concurrently_create_copy` 1240-1388, `index_concurrently_build` 1399-1439, `index_concurrently_swap` 1447-1716 incl. name swap 1490-1492 / constraint flags 1524-1529 / valid+invalid 1531-1534 / deps 1542-1680 / stats copy 1683-1705, `index_concurrently_set_dead` 1727-1761, `index_set_state_flags` 3331-3403); `validate_index` taking heap SUE + index `RowExclusiveLock` (3204-3206); the drop path (`dependency.c:1345-1352` mapping `PERFORM_DELETION_CONCURRENT_LOCK`→`concurrent_lock_mode`, `index_drop` SUE lockmode and skipped `if (concurrent)` branch); the lock/wait infrastructure (`WaitForOlderSnapshots` 339-402, `WaitForLockersMultiple` 850-949, `AccessExclusiveLock`-conflicts-`AccessShareLock` 99-103, SUE self-conflict 194-196); planner/executor invalid-index handling (`plancat.c:199-210`, `relcache.c` 4327-4329/4864-4869, `execIndexing.c:330-400`, `nbtinsert.c:563-568`); and the regression/isolation/doc evidence.
- Re-confirmed the headline Open Question: phase 6 sets `PROGRESS_CREATEIDX_PHASE_WAIT_4` (`indexcmds.c:3303`), so the `WAIT_5`=9 → `waiting for readers before dropping` text (`progress.h:82`, `system_views.sql:1015`, `monitoring.sgml:3722`) is never emitted in v12. Source wins; claim stands.
- Advanced `verified_by_agent` to `claude-opus-4-8 2026-06-12T17:41:08Z` (was `gpt-5 2026-06-12T17:29:25Z`) since every claim was re-checked against the pinned checkout. `verified:` stays human-only `false`; title keeps `(unverified)`.
- No section was added/removed/renamed/reordered, so the `## Contents` table of contents is unchanged; its 16 anchors still resolve. Coverage is unchanged, so `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` summaries were left as-is.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] review-fix v12 | REINDEX INDEX CONCURRENTLY implementation page

- Reviewed [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/reindex-index-concurrently.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Fixed correctness issues found in the review: CIC's old-snapshot wait can block a plain `SELECT` with an old snapshot, so the RIC comparison now distinguishes CIC's two lock-based `ShareLock` waits from the old-snapshot wait; the RIC `ShareLock` waits now describe the full `ShareLock` conflict set rather than only `RowExclusiveLock` writers; the `AccessExclusiveLock` waits are described as conservative heap-lock-tag waits that can include readers even if they would not actually use the old index.
- Split restriction details that were previously collapsed: direct system-catalog targets / `REINDEX SYSTEM CONCURRENTLY` error while concurrent schema/database sweeps skip system catalogs with a warning; partitioned tables warn/skip, while partitioned indexes named directly error via `ReindexPartitionedIndex`.
- Corrected the test-coverage section: v12 does have a dedicated `src/test/isolation/specs/reindex-concurrently.spec` with expected output covering concurrent read/write transactions around `REINDEX TABLE CONCURRENTLY`; removed the false "no RIC isolation spec" claim.
- Tightened failure and cleanup evidence: pre-swap failures now say the original index state is unchanged (valid only if it started valid), invalid leftovers are backed with planner (`plancat`) and executor/relcache citations, and the `PERFORM_DELETION_CONCURRENT_LOCK` / `index_drop` citation now covers both the lock-mode path and the skipped concurrent set-dead/two-wait branch.
- Added `heapam.c#heap_inplace_update` to the crash-recovery open question, updated Source References and Evidence Map, and advanced `verified_by_agent` to `gpt-5 2026-06-12T17:29:25Z`; `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] split v12 | REINDEX INDEX CONCURRENTLY — new page split out of the CIC page

- Per user request, split the combined index-concurrency page into one page per command, adding [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/reindex-index-concurrently.md) alongside the existing [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md). All evidence cites pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- New RIC page is a full deep-inquiry walkthrough of `ReindexRelationConcurrently`: the six phases (create the `_ccnew` copy via `index_concurrently_create_copy`, build, validate, swap, set-dead, drop) and the five wait points — the three shared CIC waits plus two extra `WaitForLockers(AccessExclusiveLock)` "wait for readers" waits before set-dead and drop (`AccessExclusiveLock` conflicts with `AccessShareLock`); the atomic `index_concurrently_swap` (new valid + old invalid, `_ccnew`→original / original→`_ccold` rename, constraints/triggers/comment/dependencies and per-index cumulative stats moved); `ShareUpdateExclusiveLock` scope on old index, new index, and heap; dispatch and restrictions; two-index state-flag progression; a per-phase failure table; multi-index batching; and test coverage (plus the test-absence of a RIC isolation spec).
- Recorded a source-vs-view discrepancy under `## Open Questions`: `progress.h` defines `PROGRESS_CREATEIDX_PHASE_WAIT_5 = 9` ("waiting for readers before dropping"), but phase 6 sets `WAIT_4` (8), so that view/`monitoring.sgml` phase text is never emitted in v12. Source wins per `AGENTS.md`.
- Edited the CIC page: replaced its in-page RIC subsection with a one-paragraph cross-link; trimmed the RIC-specific Evidence Map row, the `ReindexRelationConcurrently`/`index_concurrently_swap` Source References, and the Context Reviewed phrasing; reworded the recovery bullet; added the RIC page to Navigation. CIC content otherwise unchanged.
- Bookkeeping: trimmed the RIC `_ccnew`/`_ccold` example from the CIC summaries and added a RIC entry/coverage clause in `wiki/index.md`, `wiki/v12/index.md`, and `wiki/versions.md`.
- Prompt hygiene: the user's prompt spelled it "CONCURENTLY"; per the user's choice the filed `## Question` uses corrected "REINDEX INDEX CONCURRENTLY".
- Both pages keep `verified: false`, `verified_by_agent: not yet`, and `(unverified)` titles. RIC page Contents lists only `##`/`###` (the six `####` phase headings are excluded, matching the CIC precedent); all 16 RIC and 15 CIC TOC anchors were checked to resolve under the VS Code slugifier.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] review-fix v12 | CIC crash scope and citation precision

- Fixed review findings in [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Replaced the top-level `DefineIndex` signature-only citation with citations to the actual concurrent path: `INDEX_CREATE_CONCURRENT` / `INDEX_CREATE_SKIP_BUILD` setup and the post-commit build/validate/mark-valid phase sequence.
- Narrowed the failure/session-lock wording to ordinary ERROR/cancel paths: failure before commit 1 leaves no index; failure after commit 1 and before `SET_VALID` leaves an invalid index; either path releases the session-level `ShareUpdateExclusiveLock`.
- Removed the definitive crash/immediate-shutdown outcome claim. The page now states that crash recovery was not traced, cites `index_set_state_flags` as non-transactional and `heap_inplace_update` as a WAL-logged in-place overwrite, and keeps the exact crash-window flag state under `## Open Questions`.
- Added `heapam.c#heap_inplace_update` to Context Reviewed, Evidence Map, and Source References; updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` so their summaries describe crash/immediate-shutdown as an open recovery question rather than covered behavior.
- Verification fields untouched (`verified: false`, `verified_by_agent: not yet`); title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] rules | AGENTS.md — require a Contents table of contents on all content pages

- Added a `## MANDATORY Table of Contents` section to `AGENTS.md`: every content page (`type: question`, `type: concept`, legacy `type: answer`) must open with a `## Contents` table of contents, regardless of length. Navigation pages (`wiki/index.md`, `wiki/versions.md`, `wiki/log.md`, `wiki/overview.md`, `wiki/vNN/index.md`) are exempt.
- Format mandated (matches the v12 CIC page precedent): H2 + H3 only, nested bullet list, placed between the title and the first content section; `####` and deeper excluded; the `## Contents` section does not list itself; keep it in sync on section add/remove/rename/reorder.
- Anchors use the VS Code Markdown-preview slugifier (trim, lowercase, whitespace→`-`, strip punctuation incl. backticks/underscores/em dashes), with the `pg_index` -> `pgindex` gotcha and the duplicate `-1`/`-2` suffix rule spelled out; noted that `scripts/wiki_lint` does not validate `#`-anchor links, so they must be checked by hand in VS Code preview.
- Wired the requirement into the `MANDATORY Answer And File` workflow as new step 9 (add the TOC), renumbering "Update indexes and log" to step 10.
- Added a migration note: existing TOC-less content pages stay valid until next substantially revised. Did not backfill existing pages and did not change `scripts/wiki_lint` (no anchor enforcement added).
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] navigation v12 | CIC question — add Contents table-of-contents

- Per user request, added a `## Contents` table of contents to [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md), placed between the title and `## Question`.
- Depth is H2 + H3 (user's choice): the seven `##` sections, with the eight `###` Answer subsections nested under `Answer`. The `####` sub-subsections (the per-blocking-point and per-failure-state headings) are intentionally excluded.
- Anchors follow VS Code's Markdown-preview slugifier (the wiki's link target per `AGENTS.md`): lowercase, whitespace→`-`, punctuation stripped including `_`, so `### The three pg_index state flags` → `#the-three-pgindex-state-flags`. Generated mechanically via the project venv, not by hand.
- Lint-safe: `scripts/wiki_lint`'s `extract_markdown_wiki_links` skips URLs starting with `#`, so in-page anchor links are not validated as wiki links and are not flagged broken.
- Self-checked: all 15 TOC links map 1:1 onto the 15 H2/H3 headings (excluding `Contents` itself) — no missing targets, no uncovered headings, no duplicate slugs needing `-1` suffixes.
- No claims or content changed; this is navigation only. Verification fields untouched (`verified: false`, `verified_by_agent: not yet`); title keeps `(unverified)`. Left `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` coverage summaries unchanged (page coverage is unchanged).
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] expand v12 | CIC failure-states — REINDEX _ccnew/_ccold example

- Per user follow-up, added a `#### The same states under REINDEX INDEX CONCURRENTLY (the _ccnew / _ccold names)` subsection to [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md), right after the CREATE "three persistent pg_index states" table (kept as-is per the user's choice of "Both tables").
- Added a setup table (what gets reindexed: `REINDEX INDEX CONCURRENTLY concur_reindex_ind5` builds `concur_reindex_ind5_ccnew`) and a result-names table showing the `\d` names after each persistent state of the `_ccnew` copy's build: no copy -> only the original; either invalid state -> original + `concur_reindex_ind5_ccnew` INVALID; valid+swap -> single rebuilt `concur_reindex_ind5` with the old index renamed `_ccold`, set dead, and dropped.
- Source-verified the REINDEX naming/flow: six phases (`indexcmds.c:2941-2955`), the `<orig>_ccnew` copy name (`indexcmds.c:2993-2998`), the swap renaming the rebuilt copy to the original and the old to `<orig>_ccold` while marking new valid / old invalid (`indexcmds.c:3201-3241`, `index.c:1490-1492`); regression leftover (`concur_reindex_ind5` + `_ccnew` both INVALID, then `_ccnew` dropped, `create_index.out:2323-2350`).
- Flagged a correctness nuance: a reindex whose `_ccnew` build fails never invalidates a healthy original (the swap is the only step that marks the old index invalid, and runs only after the copy is built+validated); the regression's original is INVALID only because that example built it from an already-failed CIC.
- Added a matching Evidence Map row and two Source References (`ReindexRelationConcurrently`, `index_concurrently_swap`); extended Context Reviewed. `verified_by_agent` stays `not yet`; title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] expand v12 | CIC failure-states table — named example column

- Per user follow-up, added an `Example (v12 regression suite)` column to the "three persistent pg_index states" table in [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md), naming a concrete index for each state.
- Examples cited from `create_index.out`: `concur_index7` for **no index** (rejected inside a `BEGIN; ... COMMIT;` block, never listed in `\d`, `1391-1395`); `concur_index3` for **invalid, not ready** (unique build over duplicate `f2` values fails in the build scan, shown `INVALID`, `1383-1385`/`1415`); `concur_index1`/`concur_index2` for **valid** (built concurrently, listed without `INVALID`, `1413-1420`).
- **invalid, ready** has no named v12 example: it needs a duplicate appearing during the second scan, which the non-concurrent regression test cannot stage (the test file itself notes it exercises "about half the code paths"); cell points to the docs' second-scan caveat (`create_index.sgml:598-606`).
- Added a matching Evidence Map row; `verified_by_agent` stays `not yet`; title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-12] expand v12 | CIC failure-scenarios section

- Added `### Failure scenarios and the outcome on the table` to [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md), per a user follow-up.
- Core result: the leftover on the table depends on which internal transaction was running. A failure before commit 1 leaves no index (`indexcmds.c:1318-1320`); after that the leftover is always invalid (`indisvalid = false`, planner skips it, `plancat.c:200-210`), and whether it is also ready depends only on whether the build had set `indisready` — which is `index_concurrently_build`'s last action (`index.c:1426-1438`).
- Documented the three persistent `pg_index` states (none / invalid+not-ready / invalid+ready / valid) with the `index_set_state_flags` assert ladder (`index.c:3353-3396`), and the per-leftover cost on the table: not-ready indexes are opened + `RowExclusiveLock`ed but receive no entries and skip the unique check (`execIndexing.c:185-192`, `330-332`, `537-539`) yet still count for HOT-safety so updates to their columns are forced non-HOT (`relcache.c:4388-4395`, `4861-4870`); ready-but-invalid indexes take full write overhead and enforce uniqueness (`create_index.sgml:574-606`, `nbtinsert.c:563-568`).
- Pinned the build-scan-vs-validation-scan split to the two distinct unique-violation errors: "could not create unique index ... is duplicated" from the build sort (`tuplesort.c:4048-4056`, before `indisready`) vs "duplicate key ... already exists" from `_bt_check_unique` (`nbtinsert.c:563-568`, second scan, after `indisready`).
- Added source-verified facts: a failed CIC does NOT leak its session lock (removed on `ereport(ERROR)`; main-transaction abort releases session locks via `ProcReleaseLocks` -> `LockReleaseAll(DEFAULT_LOCKMETHOD, !isCommit)`, `lmgr.c:356-363`, `proc.c:772-798`); `DROP INDEX CONCURRENTLY` is retryable because `INDEX_DROP_CLEAR_VALID` does not assert its starting flags (`index.c:3367-3383`); crash behavior follows the per-commit rule (Open Questions notes the crash flag-state was reasoned, not recovery-traced).
- Regression evidence cited from `create_index.out`: a pre-existing duplicate fails the build and leaves `concur_index3` INVALID and retained through `VACUUM FULL` (`1383-1417`, `1400-1406`), and a retried `REINDEX INDEX CONCURRENTLY` stacks an invalid `_ccnew` index (`2317-2350`).
- Prompt hygiene: the follow-up had two typos ("scenations" -> "scenarios", "comprensive" -> "comprehensive"); the user chose to correct both, and the corrected follow-up is restated verbatim under `## Question`.
- Set `verified_by_agent: not yet` (was `claude-fable-5 2026-06-10T18:35:24Z`): the new section was verified against the pinned `45b88269` checkout, but the full pre-existing page (the blocking-operations and worked-example sections) was not re-verified claim-by-claim in this pass, so the page-level timestamp cannot honestly stand. `verified:` stays human-only `false`; title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] remove v12 | CIC safety runbook

- Reverted commit `dcafd70` ("Add v12 CIC safety runbook") at the user's request.
- Removed `wiki/v12/runbooks/create-index-concurrently-safety-guardrails.md` ("Safe CREATE INDEX CONCURRENTLY Runbook for PostgreSQL 12 (unverified)") and its entries in `wiki/index.md`, `wiki/v12/index.md`, and the v12 coverage sentence in `wiki/versions.md`.
- Restored the pre-runbook `AGENTS.md` and `scripts/wiki_lint` (the `wiki/vNN/runbooks/` page-type support those files gained in that commit is reverted with it).
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] answer v12 | ATTACH PARTITION index-drop scenarios

- Filed [Can ALTER TABLE ... ATTACH PARTITION Drop Indexes in PostgreSQL 12? (unverified)](v12/questions/attach-partition-index-drops.md).
- Answer leads with: there is **no** scenario — `ALTER TABLE ... ATTACH PARTITION` never drops an index from the table being attached. v12's `AttachPartitionEnsureIndexes` (`tablecmds.c:15872-16040`, called from `ATExecAttachPartition` at `15771`) is strictly match-or-create: per partitioned parent index it re-parents a compatible existing partition index via `IndexSetParentIndex` (`16000-16008`; `IndexSetParentIndex` only edits `pg_inherits`/`pg_depend`, `indexcmds.c:3398-3405`) or creates a clone via `generateClonedIndexStmt`+`DefineIndex` (`16016-16029`; v12 `DefineIndex` is the 10-param signature with no `total_parts`, `indexcmds.c:430-439`). Incompatible/extra indexes are left in place.
- Confirmed no drop path: a whole-file scan shows every `performDeletion`/`RemoveInheritance` lives in DETACH (`ATExecDetachPartition`, `16256`/`16407-16421`), DROP CONSTRAINT, trigger, or sequence paths; the pre-index `CreateInheritance` step only merges CHECK constraints (`MergeConstraintsIntoExisting` skips `contype != CONSTRAINT_CHECK`, `13219-13252`).
- Noted v12's matching is looser than later minors but still never drops: the candidate loop applies only the `relispartition` guard (`15972-15974`) and an existence-only constraint check (`15990-15998`); it has **no** `indisvalid` skip and **no** constraint-type comparison (the later `fc55c7ff` / `cee8db3f` refinements documented on the v17 page; not re-cited here per one-version-per-page).
- Distinguished the three look-alikes (absorbed/re-parented index losing independent droppability — "cannot drop index ... requires it" `indexing.out:157-159`; the docs' manual redundant-CHECK drop `ddl.sgml:3960-3971`; a later cascading parent `DROP INDEX` `indexing.out:160-177`); showed `ALTER INDEX ... ATTACH PARTITION` (`ATExecAttachPartitionIdx`, `16504-16652`) also only re-parents or errors (`refuseDupeIndexAttach`, "index definitions do not match"); and gave the docs' pre-build workflow with plain `CREATE INDEX` (v12 bans partitioned CIC, `indexing.out:56-57`).
- Regression proof: attaching a table with four incompatible indexes (hash/partial/expression/`(a,a)`) keeps all four and adds one matching `idxpart1_a_idx2` (`indexing.out:222-243`); auto-create-on-attach (`78-122`) and reuse-no-duplicate (`125-150`) also cited.
- Prompt hygiene: the prompt is grammatically clean (no typos), so no correction was needed; the false premise ("in what scenarios *can*") is answered head-on with "None". Question restated verbatim under `## Question`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] answer v17 | ATTACH PARTITION index-drop scenarios + since-v12

- Filed [Can ALTER TABLE ... ATTACH PARTITION Drop Indexes in PostgreSQL 17? (unverified)](v17/questions/attach-partition-index-drops.md).
- Answer leads with: there is **no** scenario — `ALTER TABLE ... ATTACH PARTITION` never drops an index from the table being attached. `AttachPartitionEnsureIndexes` is strictly match-or-create: per partitioned parent index it re-parents a compatible existing partition index via `IndexSetParentIndex` (kept, not rebuilt) or creates a cloned one via `DefineIndex`; incompatible/extra/invalid child indexes are left in place. Confirmed there is no `index_drop`/`performDeletion` in the attach path (all such calls live in the DETACH/DROP CONSTRAINT/trigger paths), and `MergeConstraintsIntoExisting` touches CHECK constraints only. Regression proof: after attaching a table with four incompatible indexes, `\d` shows all four retained plus one added (`indexing.out:277-281`).
- Distinguished three look-alikes that are not ATTACH drops: an absorbed/re-parented index gains `relispartition` and can no longer be dropped on its own ("cannot drop index ... because index ... requires it"); the docs' manual drop of the now-redundant CHECK constraint (not an index); and a later cascading `DROP INDEX` on the parent.
- "Changes since PostgreSQL 12" section: the no-drop invariant is unchanged from v12's `AttachPartitionEnsureIndexes`; two matching refinements landed after the pinned 12.2 — the invalid-index skip (`indisvalid` guard, commit `fc55c7ff`, 2023, back-patched to v11+) and the v17-only same-constraint-type match that stops a PRIMARY KEY matching a plain UNIQUE (commit `cee8db3f`, 2024) — plus mechanical attmap / `DefineIndex` `total_parts` / `foreach_oid` refactors. Both commits verified as ancestors of the pinned HEAD (`git merge-base --is-ancestor`) with release reach via `git tag --contains`; v12 deltas anchored to the v17 checkout's history per the one-version-per-page citation rule.
- Per MANDATORY Prompt Hygiene, flagged the prompt's punctuation/capitalization slips and the false premise; the user chose to clean up the wording and document the actual behavior. Corrected question text restated under `## Question`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v17/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-09] research v12 | leaf_fragmentation exclusion rationale

- Added `### Why leaf_fragmentation Is Not in the Priority Score` to [Finding and Prioritizing Bloated B-Tree Indexes for REINDEX in PostgreSQL 12 (unverified)](v12/questions/index-bloat-reindex-heuristic.md), per user follow-up.
- Five cited reasons: (1) planner adjacency-blindness — `genericcostestimate` prices index page fetches at the tablespace `random_page_cost`, so fragmentation neither raises nor lowers estimated cost; (2) no byte value, incommensurable with the wasted-bytes-times-usage score; (3) metric coarseness — per-page backward-right-link test with no distance/run-length/cache information; (4) narrow, workload-conditional runtime impact via the `_bt_steppage` / `_bt_readnextpage` leaf walk, with the docs claiming only "slightly faster" adjacency; (5) density-triggered sorted rebuilds reset fragmentation for free while split-time `_bt_getbuf(P_NEW)` FSM reuse re-fragments churning indexes.
- Spot-checked all newly cited ranges against the pinned checkout (`pgstatindex.c` fragment test, `nbtinsert.c` split right-page allocation, `nbtsearch.c` leaf walk, `selfuncs.c` tablespace page-cost fetch, `config.sgml` page-cost constants).
- Added an `## Open Questions` bullet: no v12-derivable break-even for fragmentation-only rebuilds; page-cost constants are global/per-tablespace, never per index.
- Extended Evidence Map, Context Reviewed, and Source References; `verified_by_agent` remains `not yet`; title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` summaries.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-09] research v12 | post-REINDEX improvement measurement

- Added `### Measuring the Improvement After a REINDEX` to [Finding and Prioritizing Bloated B-Tree Indexes for REINDEX in PostgreSQL 12 (unverified)](v12/questions/index-bloat-reindex-heuristic.md), per user follow-up.
- Three measurement layers: physical shape (`pg_relation_size` delta, one post-rebuild `pgstatindex` run, plain `EXPLAIN` cost drop), per-index counter rates (`blocks_per_scan` falls while `tuples_per_scan` stays flat as the density-win control), and query level (`EXPLAIN (ANALYZE, BUFFERS)`, `pg_stat_statements` 1.7 windows with selective reset, `track_io_timing` PGC_SUSET scope note).
- Key v12 trace: per-index cumulative counters survive both REINDEX forms — plain `reindex_index` keeps the index relation and only swaps relfilenode; `index_concurrently_swap` copies `numscans`/tuple/block counters from the old index's collector entry into the new relation's pending stats, flushed at the next `pgstat_report_stat()`. Pending-counter loss at swap time filed under `## Open Questions` as an inference.
- Added the `wiki_index_reindex_baseline_v12` capture snippet with session-scoped timeouts; extended Evidence Map, Context Reviewed, and Source References.
- `verified_by_agent` remains `not yet`; title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md` summaries.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-09] answer v12 | bloated-index REINDEX triage heuristic

- Filed [Finding and Prioritizing Bloated B-Tree Indexes for REINDEX in PostgreSQL 12 (unverified)](v12/questions/index-bloat-reindex-heuristic.md).
- Proposed a three-stage heuristic: Stage 1 shortlists from always-available statistics (`pg_class` size/tuple estimates, `pg_stat_user_tables` churn incl. non-HOT updates, `pg_stat_user_indexes.idx_scan`, `pg_statio_user_indexes`, `pg_relation_size()` `stat()` probes); Stage 2 confirms with gated `pgstatindex` runs (full block walk, `AccessShareLock`, `BAS_BULKREAD` ring); Stage 3 ranks by `est_wasted_bytes * ln(1 + idx_scan)` and executes `REINDEX (CONCURRENTLY)`.
- Traced `idx_scan` counting to `_bt_first` only (insert-time uniqueness checks via `_bt_doinsert`/`_bt_check_unique` do not count), and the index `pg_class` staleness path through `_bt_vacuum_needs_cleanup` / `btvacuumcleanup` `NULL` return / `lazy_cleanup_index` skip, governed by `vacuum_cleanup_index_scale_factor`.
- Verified both production SQL snippets' catalogs, views, functions, and GUC scopes against the pinned checkout; tagged them `wiki_index_bloat_shortlist_v12` and `wiki_index_bloat_pgstatindex_v12` with session-scoped timeouts.
- The user approved correcting the prompt grammar before filing; the corrected text is restated under `## Question`.
- Filed as `verified_by_agent: not yet`; title carries `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

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

## [2026-06-05] ingest v12 | plan_cache_mode analysis


## [2026-06-06] answer v12 | plan_cache_mode analysis
- Filed [plan_cache_mode analysis](../answers/plan_cache_mode.md).
- Analyzed  implementation in PostgreSQL 12, covering  (heuristic-based), , and  modes.
- Traced decision logic in  and cost estimation in  within .
- Updated , , and .
## [2026-06-06] answer v12 | plan_cache_mode analysis
- Filed [plan_cache_mode analysis](../answers/plan_cache_mode.md).
- Analyzed plan_cache_mode implementation in PostgreSQL 12, covering auto (heuristic-based), force_generic_plan, and force_custom_plan modes.
- Traced decision logic in choose_custom_plan and cost estimation in cached_plan_cost within src/backend/utils/cache/plancache.c.
- Updated wiki/index.md, wiki/v12/index.md, and wiki/v12/questions/plan_cache_mode_analysis.md.
## [2026-06-06] review-fix v12 | comprehensive plan_cache_mode analysis

- Expanded [plan_cache_mode analysis](../answers/plan_cache_mode.md) with a comprehensive review against the pinned `45b88269` checkout.
- Added GUC context (`PGC_USERSET`), detailed decision hierarchy (oneshots, no-params, transaction stmts, cursor options), and the `GetCachedPlan` correction step.
- Normalized all citations to the relative Markdown format.
- Updated `verified_by_agent` and removed redundant entries in `wiki/v12/index.md`.
- Updated `wiki/index.md`, `wiki/v12/index.md`, and `wiki/log.md`.

## [2026-06-06] cleanup v12 | folded plan_cache_mode answer into its question page

- Adopted the single-question-document model from `AGENTS.md`: each question page now carries its answer inline.
- Folded the full plan_cache_mode answer into `## Answer` in `wiki/v12/questions/plan_cache_mode_analysis.md`, preserving all citations. Reset `verified_by_agent` to `not yet` because the page was restructured and its claims were not re-checked against the pinned checkout in this pass.
- Removed `wiki/v12/answers/plan_cache_mode.md` and the now-empty `wiki/v12/answers/` directory.
- Removed the `## Answers` section from `wiki/v12/index.md`, which also cleared the pre-existing broken `answers/plan_cache_mode` link.
- Updated `wiki/v12/questions/plan_cache_mode_analysis.md`, `wiki/v12/index.md`, and `wiki/log.md`.

## [2026-06-09] review-fix v12 | query planner statistics sources

- Reviewed [Query Planner Statistics Sources in PostgreSQL 12 (unverified)](v12/questions/query-planner-statistics-sources.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed that `pg_stat_all_tables` is a cumulative monitoring view and is not used as a core planner input; clarified that `pg_stats` and `pg_stats_ext` are also user-facing views over the underlying statistics catalogs, not direct planner inputs.
- Tightened relation-size wording for heap and index fallback paths, corrected index-list wording around `indislive` versus planner-usable `indisvalid` filtering, and narrowed constraint wording to relation exclusion.
- Added regression coverage notes for extended statistics, FK join estimation, and cumulative monitoring counters.
- Advanced `verified_by_agent`; `verified:` stays human-only `false`, so the title keeps `(unverified)`.

## [2026-06-09] review-fix v12 | query planner statistics sources citation ranges

- Re-reviewed [Query Planner Statistics Sources in PostgreSQL 12 (unverified)](v12/questions/query-planner-statistics-sources.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed the core planner does not reference `pg_stat_all_tables`, `PgStat_StatTabEntry`, or `pg_stat_get_*()` in `src/backend/optimizer/`; planner-source claims still match the cited v12 source.
- Corrected citation ranges for `pg_stats`, `pg_stats_ext`, extended-statistics MCV coverage, and the statistics-collector test setup.
- Refreshed `verified_by_agent`; `verified:` remains human-only `false`, so the title keeps `(unverified)`.

## [2026-06-10] review-fix v12 | bloated-index planner descent-charge scope

- Re-checked every behavioral claim in [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](v12/questions/bloated-indexes-query-planner.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed: `get_relation_info` pages/tuples/`tree_height` capture and partial-index `estimate_rel_size` path, the full `RelationGetNumberOfBlocks` -> `RelationGetNumberOfBlocksInFork` -> `smgrnblocks` -> `mdnblocks` -> `_mdnblocks` -> `FileSize`/`lseek(SEEK_END)` chain and its I/O/CPU boundaries, `genericcostestimate` pro-rata page formula with guard and single-vs-repeated-scan costing, `btcostestimate` log2 comparison and `(tree_height + 1) * 50 * cpu_operator_cost` bloat-charge comment, `_bt_getrootheight`/`btm_fastlevel` vs `pgstatindex` `btm_level`, `index_pages_fetched` `total_table_pages + index_pages` cache prorating and `cost_index` heap-side reuse, all `pgstatindex_impl` classification/density/fragmentation/`index_size` formulas, the `_bt_first`/`_bt_next`/`_bt_steppage`/`_bt_readnextpage`/`_bt_getbuf` leaf walk, `_bt_split` right-page allocation and FSM reuse, `btgetbitmap` and index-only VM checks, GUC contexts, docs (`maintenance.sgml`, `reindex.sgml`, `create_index.sgml`, `config.sgml`), and `pgstattuple`/`btree_index` regression coverage (no populated-index density assertions; no density-level plan comparisons).
- Corrected the cross-AM descent-charge scope: only the height measured from the index (`_bt_getrootheight()`) is B-tree-only; `gistcostestimate()` and `spgcostestimate()` apply the same `(tree_height + 1) * 50.0 * cpu_operator_cost` charge with a synthetic `log100(index->pages)` height, and `hashcostestimate()` charges no descent cost. Updated Answer Up Front, Planner Mechanisms, the Evidence Map row, and the `wiki/index.md` / `wiki/v12/index.md` summaries to match.
- Attributed the partial-index tuple clamp to `get_relation_info()` (not `estimate_rel_size()`), extended the `estimate_rel_size` index-branch citation to include the tuple estimate (L955-L1026), extended the `FileSize` citation to its `lseek` return (L2039-L2053), and added the `pgstattuple` 1.4->1.5 `pgstatindex(regclass)` re-creation citation.
- Advanced `verified_by_agent` to the timestamp form; `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] review-fix v12 | density vs fragmentation citation ranges

- Re-checked every behavioral claim in [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](v12/questions/leaf-density-vs-fragmentation-index-scan-io.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed: `pgstatindex_impl` live-leaf-only `avg_leaf_density` (`100 - free_space / max_avail * 100`) and `leaf_fragmentation` (`fragments / leaf_pages * 100` with the `btpo_next != P_NONE && btpo_next < blkno` test), `PageGetFreeSpace` line-pointer subtraction, `nbtree.h` fillfactor constants (90 leaf / 70 non-leaf / 96 single-value) and `nbtsplitloc.c` rightmost-vs-50:50 selection, the `_bt_first`/`_bt_binsrch`/`_bt_readpage`/`_bt_next`/`_bt_steppage`/`_bt_readnextpage`/`_bt_walk_left`/`_bt_endpoint` scan paths, `_bt_insertonpg`/`_bt_split`/`_bt_getbuf(P_NEW)` FSM-reuse-or-extend allocation with `btvacuumpage` `RecordFreeIndexPage`, `get_relation_info` pages/tuples/tree-height capture and the `estimate_rel_size` index branch, `genericcostestimate` pro-rata page formula, the `(tree_height + 1) * 50.0 * cpu_operator_cost` bloat descent charge, `cost.h` 1.0/4.0 defaults and the `config.sgml` mechanical/cached/SSD `random_page_cost` guidance, `show_buffer_usage`/`BufferUsage` counter shapes, `btgetbitmap` and index-only-scan VM checks, docs (`maintenance.sgml` physical-adjacency claim, `reindex.sgml` bloat wording, `indexam.sgml`, `pgstattuple.sgml`), and the `pgstattuple`/`btree_index` regression coverage gaps. Grepped that `leaf_fragmentation` exists only under `contrib/pgstattuple/`, so the planner-blindness claims hold. Re-derived all density/fragmentation/combined multiplier tables from the stated formulas.
- No factual corrections needed. Widened two citation ranges: the `_bt_first` scan-key-preprocessing sentence now cites the full function (`L746-L1328`, covering `_bt_preprocess_keys`), and the Operational Reading `pgstatindex` result citation now starts at `L336` so it covers the cited `index_size` and `leaf_pages` outputs.
- Advanced `verified_by_agent` to the timestamp form (`claude-fable-5 2026-06-10T10:43:46Z`); `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] review-fix v12 | leaf density 60 vs 90 citation precision

- Re-checked every behavioral claim in [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](v12/questions/leaf-density-60-vs-90-query-impact.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed: the `pgstatindex_impl` live-leaf-only density formula and `PageGetFreeSpace` line-pointer reservation, `nbtree.h` fillfactor constants/comment and `nbtsplitloc.c` rightmost-vs-50:50 selection, `get_relation_info` pages/tuples/tree-height capture and the partial-index `estimate_rel_size` branch, the `genericcostestimate` pro-rata `numIndexPages` formula feeding `index_pages_fetched` and tablespace page costs, the `(tree_height + 1) * 50.0 * cpu_operator_cost` bloat descent charge per scan and per SA expansion, `bthandler` callback wiring, the `_bt_first`/`_bt_binsrch`/`_bt_readpage`/`_bt_next`/`_bt_steppage`/`_bt_readnextpage` (forward `btpo_next` + `_bt_getbuf`, `P_FIRSTDATAKEY` restart)/`_bt_walk_left`/`_bt_endpoint` scan paths, `btgetbitmap` and index-only-scan `VM_ALL_VISIBLE` checks, `BufferAlloc` hit-path partition-lock/`BufTableLookup`/pin sequence with `NUM_BUFFER_PARTITIONS = 128`, `btvacuumcleanup`/`btvacuumscan`/`lazy_cleanup_index` statistics flow, `maintenance.sgml`/`reindex.sgml` wording, and the `pgstattuple`/`btree_index` regression coverage gaps.
- Tightened the leaf-density decay bullet: deletes do not directly create `LP_DEAD` items; later index scans mark dead tuples via `_bt_killitems`, and `_bt_vacuum_one_page` or VACUUM removes them. Added `nbtutils.c#_bt_killitems` and `nbtinsert.c#_bt_vacuum_one_page` citations and listed `nbtutils.c` under Context Reviewed.
- Corrected citation ranges to the pinned checkout: `estimate_rel_size` index branch extended to its tuple estimate (`L955-L1026`, three places), `index_pages_fetched` normalized to `L787-L877` (was `L754-L860`, which started inside `extract_nonindex_conditions`), `_bt_walk_left` extended to its function end `L2053` (three places, covering the deleted-page handling the claim cites), `btvacuumscan` stats extended to `L1099` (covers the `stats->num_pages`/`pages_free` assignments), `show_buffer_usage` extended to `L2985` (three places), `cost_index` corrected to `L475-L748`, combined selfuncs reference corrected to `L5679-L6223` (the old `L5919-L6267` missed `genericcostestimate` and included `hashcostestimate`), and the malformed `nbtree.c#L215-L330, L897-L996` link split into valid `btgettuple`/`btgetbitmap` (`L213-L342`) and `btvacuumcleanup`/`btvacuumscan` (`L890-L1099`) links. Split the mislabeled `bufmgr.c` reference (`L3400-L3607`) into `ReleaseAndReadBuffer` (`L1506-L1553`) and `LockBuffer` (`L3585-L3607`).
- Advanced `verified_by_agent` to the timestamp form (`claude-fable-5 2026-06-10T11:04:20Z`); `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] remove v19 | autovacuum parallel workers, scoring, and visibility-map question

- Removed `wiki/v19/questions/autovacuum-parallel-scoring-visibility.md` at the user's request.
- The page was an orphan and untracked (never committed): no links from `wiki/index.md`, `wiki/v19/index.md`, or `wiki/log.md`, and no references to its slug anywhere else in the repo, so no index edits were needed.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] answer v19 | autovacuum parallel workers, table scoring, set-VM-on-read

- Filed [PostgreSQL 19 Autovacuum and VACUUM: Parallel Workers, Table Scoring, and Setting Pages All-Visible During Reads (unverified)](v19/questions/autovacuum-parallel-scoring-visibility.md), refiling the topic that was removed as an orphan draft earlier today (this time tracked and linked from all indexes).
- Verified all three user-supplied claims against the pinned `REL_19_BETA1` checkout (`4b0bf0788b066a4ca1d4f959566678e44ec93422`) before drafting; the v19 release notes pointed to the feature commits, but every behavioral claim is cited to implementation source.
- Parallel autovacuum: `parallel_vacuum_compute_workers()` caps autovacuum at `autovacuum_max_parallel_workers` (PGC_SIGHUP, default 0 = serial) vs `max_parallel_maintenance_workers` for manual VACUUM; per-table `autovacuum_parallel_workers` reloption maps to `VacuumParams.nworkers` (0→-1 disable, >0→that many, -1→auto); index phases only; new `PVSharedCostParams` generation-counter cost-delay propagation. Core commit `1ff3180ca`, TAP test `001_parallel_autovacuum.pl`.
- Table scoring: `relation_needs_vacanalyze()` fills `AutoVacuumScores` (xid/mxid/vac/vac_ins/anl, each × a `*_score_weight` GUC, all PGC_SIGHUP, 1.0, 0.0–10.0; failsafe-age `pow()` scaling); `do_autovacuum()` sorts `TableToProcess` descending by `scores.max` with an all-weights-0.0 escape hatch; eligibility stays threshold-based. Distinguished from the launcher's separate database-level `adl_score`. Monitoring view `pg_stat_autovacuum_scores`. Core commit `d7965d65f`; algorithm has no dedicated test (view covered by `rules.out`).
- Set-VM-on-read: `ScanRelIsReadOnly()` → `SO_HINT_REL_READ_ONLY` → `heap_prepare_pagescan()` → `heap_page_prune_opt(..., rel_read_only)` adds `HEAP_PAGE_PRUNE_SET_VM`; `PruneState.attempt_set_vm`/`set_all_visible`; `heap_page_will_set_vm()` safety valve avoids newly dirtying the page or forcing an FPI; read scans set all-visible (not all-frozen). Reduces future ordinary VACUUM work via the VM-skip path in `vacuumlazy.c`. Headline commit `b46e1e54d` (no dedicated test of its own); listed the v19-cycle commit cluster with a scoping caveat under Open Questions.
- Prompt hygiene: user approved a light copyedit of the prompt (code-font GUC, "VACUUM" the command, "all-visible in the visibility map", terminal period); the copyedited prompt is restated under `## Question` with a note. Filed `verified_by_agent: not yet`; title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v19/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] repin v19 | repin to post-beta1 master e18b0cb and re-verify all questions

- Repinned `raw/postgres-19/` from the `REL_19_BETA1` tag (`4b0bf0788b066a4ca1d4f959566678e44ec93422`) to the post-beta1 `master` commit `e18b0cb7344cb4bd28468f6c0aeeb9b9241d30aa` (63 commits later, dated 2026-06-10). PostgreSQL 19 has no `REL_19_STABLE` branch yet, so `master` is the v19 development line; `configure.ac` still reads `19beta1` at this commit. User-requested source fetch (`git fetch origin master`).
- Scoped re-verification by intersecting each page's cited files with the beta1..e18b0cb diff. Most cited files were byte-identical, so only changed-file citations needed line re-derivation.
- `pg-plan-advice.md`: no citation drift (`relnode.c` `pgs_mask` lines L234/L840 byte-identical); no `pg_plan_advice`/`test_plan_advice`/doc commits in range, so the "22 direct module commits" history and newest module commit `b1901e28` are unchanged. Updated `pinned_commit` and the Context-Reviewed pin reference.
- `autovacuum-parallel-scoring-visibility.md`: no autovacuum/parallel-vacuum/pruning feature-file changes in range. `executor.h#ScanRelIsReadOnly` shifted L700→L698 (two unrelated single-line deletions above it); `config.sgml` and `system_views.sql` cited ranges unaffected (net-zero / below the hunks). Updated `pinned_commit`, Short Answer pin, and Source-Commit-History pin note.
- `repack-command.md`: commit `cd7b204b` ("Disallow direct use of the pgrepack logical decoding plugin", 2026-06-09) rewrote `repack_setup_logical_decoding()` and the `pgrepack` output plugin. Added it as the 41st feature-scope commit (count 40→41 in the title, the page intro, the table-intro, and Context Reviewed; date range now 2026-03-10..2026-06-09). Recorded the macro rename `REPL_PLUGIN_NAME`→`PGREPACK_PLUGIN` and the slot-name change `repack_<pid>`→`pg_repack_<pid>`, and added a citation for the new `repack_startup()` direct-use guard plus the new `repack.sql` rejection test (now L1-L84). Re-derived every drifted citation: `repack_setup_logical_decoding` L194-L309→L194-L291, slot L215-L225→L217-L223, enable-decoding L211-L225→L211-L224, cleanup L311-L322→L293-L304, WAL-recycling L392-L421→L374-L403, `change_useless_for_repack` L517-L554→L499-L536, `pgrepack.c` `repack_process_change` L92-L161→L110-L179 and `repack_store_change` L172-L287→L190-L305, `repack_internal.h#DecodingWorkerShared` L63-L119→L61-L117; whole-file bounds updated (`repack_worker.c` →L536, `pgrepack.c` →L305, `repack_internal.h` →L122). Unchanged: `RepackWorkerMain` (L59-L164), `decode_loop`, `connect`, `RepackWorkerShutdown`, `ConcurrentChangeKind`, and all `repack.c`/`slot.c`/`system_views.sql` citations.
- All re-derived ranges spot-checked to bracket their intended symbols in the new checkout.
- `verified:` stays human-only `false` and `verified_by_agent:` stays `not yet` on all three pages: this pass repins and re-verifies citations, not a full claim-by-claim verification. Titles keep `(unverified)`.
- Updated `wiki/versions.md` (branch label `master` (post-19beta1), commit, REPACK 40→41, pin sentence), `wiki/v19/index.md` (branch/tag, commit, Repinned 2026-06-10, coverage, REPACK link 40→41), and `wiki/index.md` (v19 pin line, REPACK 40→41).
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] answer v12 | CREATE INDEX CONCURRENTLY implementation and table locks

- Filed [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md).
- Traced the concurrent branch of `DefineIndex` end-to-end: four internal transactions (catalog create -> build -> validate -> mark valid), two heap scans, and three waits, with the `indislive`/`indisready`/`indisvalid` progression created not-ready/not-valid by `index_create`/`UpdateIndexRelation` and flipped non-transactionally (`heap_inplace_update`) by `index_set_state_flags`.
- Added the requested "All steps and locks required on the table" section: transaction-level `ShareUpdateExclusiveLock` (re-taken in `index_concurrently_build` and `validate_index`) plus a session-level `ShareUpdateExclusiveLock` spanning the commits; clarified that the three `WaitForLockers`/`WaitForOlderSnapshots` waits take no table lock (they sleep on VXIDs from `GetLockConflicts`); and used the `lock.c` `LockConflicts` table to show `ShareUpdateExclusiveLock` admits SELECT/DML (`RowExclusiveLock`) but self-conflicts and blocks plain `CREATE INDEX`/`VACUUM`/`ANALYZE`/DDL.
- Documented preconditions/restrictions (no transaction block via `PreventInTransactionBlock`, temp fallback, partitioned/system-catalog/exclusion bans), the invalid-index failure path, and regression (`create_index.sql`) plus `multiple-cic` isolation test coverage.
- User approved correcting the prompt typo ("explaination" -> "explanation"); the corrected text is restated verbatim under `## Question`.
- Filed as `verified_by_agent: not yet`; title carries `(unverified)`. Two `## Open Questions` items: the first-scan tuple-visibility rule inside `index_build`/`table_index_build_scan` was summarized from the `validate_index` header comment rather than traced line-by-line, and v12 `WaitForOlderSnapshots` does not exclude other CIC builds from its wait set.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] answer v17 | CREATE INDEX CONCURRENTLY implementation, table locks, and v12 diff

- Filed [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](v17/questions/create-index-concurrently.md), a cross-version page (v17 implementation + "what changed from v12").
- Verified against the pinned `raw/postgres-17/` checkout (`54eeefaedbee0385529f3edf321bb99e49232aaa`) that the CIC algorithm is structurally identical to v12: same four internal transactions, two heap scans, three waits, and `ShareUpdateExclusiveLock` (transaction- + session-level) footprint; `lockmode = concurrent ? ShareUpdateExclusiveLock : ShareLock` and the `index_concurrently_build`/`validate_index` heap-SUE/index-RowExclusive locks all match.
- Headline v12->v17 change: the PG14 `PROC_IN_SAFE_IC` optimization. A CIC on a non-partial, non-expression index advertises `PROC_IN_SAFE_IC` in `MyProc->statusFlags` via `set_indexsafe_procflags()` (called after each phase commit), and `WaitForOlderSnapshots` adds it to its ignore set. Scoped precisely: this only shortens Wait 3 (the snapshot wait) and only for *other* safe builds; the lock-based `WaitForLockers` Waits 1/2 are unaffected and still wait out writers. The v17 docs document this ("...unless the indexes involved are partial or have columns that are not simple column references").
- Per AGENTS.md one-version-per-page citation rule, all source citations are to `raw/postgres-17/`; the v12-diff claims are anchored to the v17 checkout's own commit history. Verified each cited commit with `git show`/`git tag --contains`: `c98763bf` (2020-11-25, CIC PROC_IN_SAFE_IC; first in REL_14_0), `f9900df5` (2021-01-15, RIC), `5788e258` (2020-08-14, PGXACT->ProcGlobal statusFlags), and `e28bb885` (2022-05-31, "Revert changes to CONCURRENTLY that 'sped up' Xmin advance" — present in v17, so VACUUM still holds CIC's xmin horizon).
- A background subagent mined candidate commits; I re-verified every hash against the checkout and discarded its fabricated/approximate hash for the (RIC-only) PG17 fix `cd6b2ae3`, which I left in Open Questions as out-of-scope.
- User approved correcting two prompt issues ("explaination" -> "explanation"; "what have changed" -> "what has changed"); the corrected prompt is restated verbatim under `## Question`.
- Filed `verified_by_agent: not yet`; title carries `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v17/index.md`.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-10] review-fix v17 | ATTACH PARTITION index drops: per-version since-v12 breakdown

- Reviewed [Can ALTER TABLE ... ATTACH PARTITION Drop Indexes in PostgreSQL 17? (unverified)](v17/questions/attach-partition-index-drops.md) against pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa` and restructured "What changed from PostgreSQL 12" into per-version subsections (v13/v14/v15/v16/v17.0/17.x), per user request.
- Re-confirmed the no-drop invariant: a fresh `tablecmds.c`-wide sweep shows no `index_drop`/`performDeletion`/`performMultipleDeletions` inside `ATExecAttachPartition`, `AttachPartitionEnsureIndexes`, or `ATExecAttachPartitionIdx` (nearest calls, L19537/L19789, are DETACH-path). A line-by-line diff of `AttachPartitionEnsureIndexes` between the pinned v12 and v17 checkouts confirmed every textual delta maps to a listed commit (citations stay v17-only).
- New per-version entries beyond the previously filed `fc55c7ff`/`cee8db3f`: v13 `e1551f96e6` attmap refactor and `5b1c61e8b8` error-code fix; v14 `e1ae40f381` error wording and `afc7e0ad55` dedicated `DROP INDEX CONCURRENTLY`-on-partitioned-index error; v15 `94aa7cc5f7` `NULLS NOT DISTINCT` joins `CompareIndexInfo` matching and `7b6ec86532` partitioned-index DROP deadlock fix; v16 `27f5c712b2` `DefineIndex` `total_parts`; v17.0 `9f71e10d65` `CompareIndexInfo` expression-vs-column fix, `38ea6aa90e` `validatePartitionedIndex` syscache-copy fix, `0cecc908e9` `relhassubclass` locking, and the `b0e96f3119`/`6f8bb7c1e9` `wqueue`/`foreach_oid` residual refactors; 17.x stable `becf6d2696` (2026-04-22, back-patched to v14+) letting a re-attach of an already-attached index validate a still-invalid parent, with its new `indexing.sql` L225-L306 tests.
- Recategorized `fc55c7ff` under v16: it is an ancestor of the 17devel stamp commit, so 16.0 is its first major release (back-patched to v11+ minors). Corrected the prior "git tag --contains" verification wording: the checkout carries no release tags, so first-release attribution now uses the "Stamp HEAD as NNdevel" cycle brackets, stated explicitly in Context Reviewed and Open Questions.
- Updated the ALTER INDEX ... ATTACH PARTITION section for `becf6d2696` (re-attach is accepted and attempts one round of parent validation), added the v17 CIC-still-banned-on-partitioned-tables citation (indexcmds.c:729) to the workflow section, extended the incompatible-index table row with the v15 `NULLS DISTINCT` dimension, and linked the now-existing v12 companion page from Open Questions and Navigation (replacing the stale "does not yet include" note).
- Spot-verified every carried-over citation range (tablecmds.c, indexcmds.c, index.c, ddl.sgml, indexing.sql/.out) against the pinned checkout; extended `CompareIndexInfo` (L2510-L2627) and `IndexSetParentIndex` (L4304-L4429) bounds; widened Source References (`ATExecAttachPartitionIdx` to L20022, added `validatePartitionedIndex` L20051-L20146).
- Advanced `verified_by_agent` to the timestamp form (`claude-fable-5 2026-06-10T18:01:25Z`); `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v17/index.md`.

## [2026-06-10] review-fix v12 | CREATE INDEX CONCURRENTLY: full claim review + blocking-operations section

- Reviewed [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2). Every existing claim and citation range re-checked against the pinned source (DefineIndex concurrent branch, WaitForOlderSnapshots, index_create/UpdateIndexRelation, index_concurrently_build, validate_index, index_set_state_flags, WaitForLockers/WaitForLockersMultiple, LockConflicts, lockdefs.h, utility.c dispatch, create_index.sgml, create_index.sql, multiple-cic.spec); no corrections were needed.
- Per user follow-up, added a new section "All operations that can block CREATE INDEX CONCURRENTLY", organized by the four blocking points: (1) the initial `ShareUpdateExclusiveLock` acquisition, with the five conflicting lock modes mapped to v12 commands via the `mvcc.sgml` lock table, the autovacuum SIGINT-cancel after `deadlock_timeout` (proc.c:1308-1375) with the anti-wraparound exception, and prepared transactions retaining locks until COMMIT/ROLLBACK PREPARED; (2)/(3) the two `WaitForLockers(ShareLock)` waits, which in practice wait only on open `RowExclusiveLock` writer transactions (idle-in-transaction included) — lock waiters, later writers, reads, `SELECT FOR UPDATE/SHARE`, and prepared transactions are skipped; (4) `WaitForOlderSnapshots`, which waits on same-database backends with xmin <= limitXmin (long-running queries on unrelated tables, REPEATABLE READ/SERIALIZABLE transactions per snapmgr.c:336-356, cursor-holding idle-in-transaction sessions, manual ANALYZE, other CICs) and skips autovacuum/lazy-VACUUM flags, other databases, xmin-0 backends, and prepared transactions (twophase.c:465-472). Closed with the `pg_stat_progress_create_index` wait phases.
- New Open Questions: the lmgr.c:890-894 assertion that ignoring prepared transactions in Waits 1/2 is fine, versus a prepared writer committing later at COMMIT PREPARED (safety not assessed from this checkout); walsender/slot xmin holders versus the Wait-3 valid-VXID requirement.
- User approved grammar-correcting the follow-up prompt ("and could block" -> "that could block"); the corrected follow-up is restated under `## Question`.
- Advanced `verified_by_agent` to the timestamp form (`claude-fable-5 2026-06-10T18:22:06Z`); `verified:` stays human-only `false`, so the title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.

## [2026-06-10] review-fix v12 | CREATE INDEX CONCURRENTLY: pg_dump and long-open-transaction worked examples

- Extended the blocking-operations section of [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](v12/questions/create-index-concurrently.md) with two worked examples answering user follow-ups, verified against the pinned `raw/postgres-12/` checkout.
- `pg_dump` (same database): blocks CIC at Wait 3 only, for the dump's entire remaining duration — it wraps the dump in one `REPEATABLE READ, READ ONLY` (or `SERIALIZABLE, READ ONLY, DEFERRABLE`) transaction (pg_dump.c:1166-1194) so its xmin stays pinned, and it disables `statement_timeout`/`lock_timeout`/`idle_in_transaction_session_timeout` (pg_dump.c:1140-1147). Its only table locks are `ACCESS SHARE` (pg_dump.c:6646-6671), which no other CIC blocking point checks; other-database dumps and dumps started after the reference snapshot are skipped; the blocking is one-way.
- Hour-long open / idle-in-transaction session: blocks per what it did, never per age — case table covering BEGIN-only (nothing), target-table writer (Waits 1/2 until transaction end, no recheck), pinned `REPEATABLE READ`/`SERIALIZABLE` snapshot (Wait 3), reads-only `READ COMMITTED` with no cursors (nothing: `SnapshotResetXmin` clears the advertised xmin, snapmgr.c:989-1028), open cursor / mid-statement old snapshot (Wait 3), and conflicting target-table DDL lock (initial acquisition). Noted that writes to other tables are irrelevant at Waits 1/2 (per-locktag) and that transactions starting after a wait begins never extend it.
- Restated the grammar-corrected follow-ups under `## Question` (per the user's earlier correction approval), including the idle-in-transaction variant.
- Updated Evidence Map, Context Reviewed, and Source References (pg_dump.c, snapmgr.c#SnapshotResetXmin); advanced `verified_by_agent` to `claude-fable-5 2026-06-10T18:35:24Z`; `verified:` stays `false`, title keeps `(unverified)`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`.

## [2026-06-10] review-fix v12 | Foreign-key join optimization for two-table joins

- Reviewed [Foreign-Key Join Optimization for Two-Table Joins (unverified)](v12/questions/fk-join-optimization-two-tables.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2). Every claim and citation re-checked against the pinned source.
- Confirmed the central thesis with a whole-source `fkey_list` sweep: the field appears only in pathnodes.h (declaration), planmain.c:78 (init), plancat.c (collection), initsplan.c (matching/pruning), outfuncs.c (debug output), costsize.c:4721 (selectivity), and the analyzejoins.c:441-444 comment — no joinpath.c/joinrels.c/GEQO reader, so FKs influence only cardinality estimation, never join order or algorithm choice.
- Re-verified the selectivity mechanics: straddle + semi/anti-singleton gating (costsize.c:4734-4757), parent_ec / `list_member_ptr(rinfos[i])` clause matching (4780-4812), the `removedlist != nmatched_ec + nmatched_ri` double-count guard with clause restore (4842-4847), `fkselec *= 1.0 / Max(ref_tuples, 1.0)` for inner/left/full (4890-4901) and `ref_rel->rows / ref_tuples` for semi/anti (4885-4888), `JOIN_INNER` `nrows = outer * inner * fkselec * jselec` with `clamp_row_est` (4655-4690), and both XXX caveats (no NULL derating, inheritance disregard, 4854-4870).
- Re-verified matching/pruning: EC-then-loose-qual scan with `outerjoin_delayed` skip (initsplan.c:2465-2546), both-sides-`RELOPT_BASEREL` drop (2445-2447), full-match retention `(nmatched_ec + nmatched_rcols) == nkeys` (2558), opfamily-checked EC match (equivclass.c:2086-2094), and the query_planner ordering (planmain.c:200/223/229/250/271).
- Re-verified the no-FK-in-join-removal claims: `join_is_removable` (analyzejoins.c:160) and `reduce_unique_semijoins` (511, via `innerrel_is_unique`) prove uniqueness through unique indexes (`rel_supports_distinctness`:608-615, `relation_has_unique_index_for`) or subquery distinctness — FK metadata is never consulted.
- One fix: the `get_relation_foreign_keys` short-circuit list omitted the referenced-side skip of inheritance-parent RTEs (`rte->inh`, plancat.c:541-543); added the bullet with citation and extended the matching Evidence Map row. All other claims and line anchors were already exact.
- Both existing Open Questions stand: later-minor FK-uniqueness shortcuts cannot be checked from this 12.2-pinned checkout, and the multi-column NULL overestimate is unquantified.
- Advanced `verified_by_agent` to the timestamp form (`claude-fable-5 2026-06-10T18:48:46Z`); `verified:` stays human-only `false`, so the title keeps `(unverified)`.

## [2026-06-10] review-fix v12 | pg_stat_statements mechanics and configuration

- Reviewed [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 12 (unverified)](v12/questions/pg-stat-statements.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2). Every claim and citation re-checked against the pinned source.
- Confirmed the core mechanics end to end: in-extension jumbling and `hash_any_extended()` queryId at `pgss_post_parse_analyze` (778-844, zero forced to 1), utility text hashing via `pgss_hash_string` (1081-1086, zero forced to 2 at 1151-1162), the `(userid, dbid, queryid)` key (135-140), the four `DefineCustom*` GUCs with exact defaults/contexts (`max` 5000/100/INT_MAX `PGC_POSTMASTER`, `track` top `PGC_SUSET`, `track_utility` on `PGC_SUSET`, `save` on `PGC_SIGHUP`, 365-410), `pgss_enabled()` (275-277), all seven hook installs (425-438), the EXECUTE/PREPARE/DEALLOCATE exclusion and COPY-tag row parse (984-1026), Welford counter updates (1238-1296), sticky entries via `cur_median_usage` (1726) skipped at readout (1613-1615), `entry_dealloc` 5%-min-10 victims (1820), external text file with `query_len = -1` drop on gc failure (2138-2226), `<insufficient privilege>`/null-queryid privacy filtering (1551-1602), the 1.7 reset signature and superuser-only REVOKE, clean-shutdown dump / crash no-dump (692-694), and the 23-column 1.3 API (1336-1341, 1.4 SQL OUT list).
- Verified the supporting absences by whole-tree grep: no `compute_query_id`, no `track_planning`, no `pg_stat_statements_info` anywhere in the v12 checkout.
- Fixes applied: (1) cited the previously uncited `track_activity_query_size` claim (guc.c:3164-3173 says "Sets the size reserved for pg_stat_activity.query"; config.sgml:6820-6836); (2) extended the regression-test citation from L1-L181 to L1-L254 so it covers the utility-command and reset sections the prose describes; (3) reworded three forward-version attributions ("unlike later releases", "added in later versions", "added in PostgreSQL 14") into v12-absence claims, since REL_12_STABLE history predates those changes and cross-version citations are banned; retitled the diff section to "v12 Specifics" and added an Open Questions entry pointing at the v18 sibling page; (4) backed the no-`pg_stat_statements_info`/no-`dealloc` bullet with Makefile DATA, 1.4-SQL view, and `Counters` citations.
- Advanced `verified_by_agent` to the timestamp form (`claude-fable-5 2026-06-10T18:58:30Z`); `verified:` stays human-only `false`, so the title keeps `(unverified)`.

## [2026-06-10] question v17 | bi-directional logical replication + commit history by minor version

- Filed [How Bi-Directional Logical Replication Works in PostgreSQL 17, and All Related Commits by Minor Version (unverified)](v17/questions/bidirectional-logical-replication.md) against pinned `raw/postgres-17/` commit `54eeefaedbee0385529f3edf321bb99e49232aaa` (`REL_17_STABLE`, 17.10).
- User approved correcting the prompt wording ("how the bi-directional logical replication works" -> "how bi-directional logical replication works") before filing; the corrected prompt is restated under `## Question` with a prompt note.
- Mechanics covered: `origin = none/any` parsing and `pg_subscription.suborigin`; apply-worker/tablesync session origins (`pg_<suboid>`, `pg_<suboid>_<relid>`); WAL origin tagging via `XLOG_INCLUDE_ORIGIN` call sites (heapam, xact, twophase, truncate, logical messages) and `XLogRecordAssemble`; `START_REPLICATION ... origin` gated on publisher >= 16; `pgoutput` `publish_no_origin` and `pgoutput_origin_filter`; decode-time per-change and per-transaction (`DecodeTXNNeedSkip`) filtering; reserved origin names; `check_publications_origin` WARNING for `copy_data = true` + `origin = none`; two-node loop walkthrough matching `030_origin.pl`; limitations (manual conflicts, no DDL/sequence replication, no docs how-to in v17).
- Commit history scoped by `git log` sweeps (`030_origin.pl` path, `-S LOGICALREP_ORIGIN_NONE`, `-S check_publications_origin`, `-S publish_no_origin`, subject search) and attributed by `git merge-base --is-ancestor` against `Stamp` commits: 16-cycle `366283961a`/`8756930190`/`0324651573`; 17.0 `54ccfd6586`; 17.5 `0ae1245e04`; 17.10 `f0f59b658e` (CVE-2026-6638); none in 17.1-17.4, 17.6-17.9, or after Stamp 17.10; foundations `5aa2350426` (9.5), `be65eddd80` (9.6), `665d1fad99` (10); incidental and adjacent origin-infrastructure commits listed separately.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v17/index.md`. `verified_by_agent: not yet` pending a full claim re-check pass.

## [2026-06-10] question v18 | bi-directional logical replication + new features since v17

- Filed [How Bi-Directional Logical Replication Works in PostgreSQL 18, and New Logical Replication Features Since PostgreSQL 17 (unverified)](v18/questions/bidirectional-logical-replication.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e` (`REL_18_STABLE`).
- User approved correcting the prompt wording ("explain how the bi-directional logical replication works, add section with all new features on logical replication since version 17") before filing; the corrected prompt is restated under `## Question` with a prompt note.
- Mechanics covered with v18 citations: `origin = none/any` parsing and `pg_subscription.suborigin`; apply-worker/tablesync session origins (`pg_<suboid>`, `pg_<suboid>_<relid>`); WAL origin tagging via `XLOG_INCLUDE_ORIGIN` call sites and `XLogRecordAssemble`; `START_REPLICATION ... origin` gated on publisher >= 16; `pgoutput` `publish_no_origin` and `pgoutput_origin_filter`; decode-time per-change and per-transaction (`DecodeTXNNeedSkip`) filtering; reserved origin names; `check_publications_origin` WARNING (now via `pg_get_publication_tables` with partition handling inline); two-node loop walkthrough matching `030_origin.pl`; and the new-in-18 conflict layer in the bidirectional context (`update_origin_differs`/`delete_origin_differs` need `track_commit_timestamp`, tested at `030_origin.pl` L147-L188).
- New-features-since-17 section scoped by the checkout's own `release-18.sgml` (Logical Replication, Streaming Replication and Recovery, Logical Replication Applications sections) and verified in source; all 20 feature commits ancestry-checked into the v18 cycle (between `e26810d01d` 18devel and `3d6a828938` Stamp 18.0): conflict detection/logging/statistics (`9758174e2`, `edcb71258`, `640178c92`, `6c2b5edec`, `73eba5004`), generated-column replication (`745217a05`, `7054186c4`, `87ce27de6`, `6252b1eaf`), `streaming = parallel` default (`1bf1140be`), alterable `two_phase` (`1462aad2e`, `4868c96bc`), `max_active_replication_origins` (`04ff636cb`, postmaster), `idle_replication_slot_timeout` (`ac0e33136`, sighup), `pg_createsubscriber` `--all`/`--clean`/`--enable-two-phase` (`fb2ea12f4`, `e5aeed4b8`+`60dda7bbc`, `e117cfb2f`), `pg_recvlogical` `--enable-failover`/`--enable-two-phase`/`--drop-slot` (`cf2655a90`, `c68100aa4`), contrib `pg_logicalinspect` (`7cdfeee32`); `pg_upgrade` subscription-check parallelization (`7baa36de5`) noted as related-only.
- Open question filed: the pinned checkout (2026-04-30) still interpolates names unquoted in `check_publications_origin` (subscriptioncmds.c:2142); the v17 sibling page records a 17.10 quoting fix dated 2026-05-11 that postdates this pin — repin to confirm the `REL_18_STABLE` equivalent.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`. `verified_by_agent: not yet` pending a full claim re-check pass.

## [2026-06-16] question v18 | row-level security implementation, settings, performance, and fixes since v12

- Filed [Row-Level Security (RLS) in PostgreSQL 18: Implementation, Performance, Settings, and Fixes Since PostgreSQL 12 (unverified)](v18/questions/row-level-security-rls.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e` (`REL_18_STABLE`).
- User approved correcting the prompt wording before filing; the corrected prompt is restated under `## Question` with a prompt note.
- Covered RLS implementation as rewrite-time `securityQuals` and `WithCheckOption` entries; permissive/restrictive policy combination; policy storage in `pg_policy`; table flags in `pg_class`; BYPASSRLS, owner, FORCE RLS, and `row_security` decisions; planner security levels and plan-cache invalidation; executor WCO checks; partitions/views/COPY/CTAS/logical replication boundaries; and error-detail suppression.
- Added a performance/scalability section for source-visible mechanisms: policy expression evaluation, OR/AND qual growth, security-barrier ordering, sublink recursion/locking, per-role plan-cache invalidation, policy role matching, AccessExclusiveLock policy/table-RLS DDL, and COPY/query-path effects.
- Listed RLS-related settings and surfaces with scope: `row_security` (`PGC_USERSET`, session/transaction), `ALTER TABLE` RLS flags, policy clauses, `BYPASSRLS`, `row_security_active()`, catalog views, `pg_dump --enable-row-security`, and `--no-policies`.
- Listed same-checkout RLS-scoped fixes since PostgreSQL 12, including whole-row/WCO handling, pg_dump policy comments and policy loading scalability, relcache memory leaks, BYPASSRLS role changes, DROP OWNED/policy-role fixes, logical replication RLS checks, COPY TO inheritance, MERGE/CVE fixes, statistics-view CVE checks, WHERE CURRENT OF/ctid, plan-cache role-dependency CVEs, SQL-function plan-cache coverage, selectivity-estimation checks, and `--no-policies`.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md`. `verified_by_agent: not yet` pending a full claim re-check pass.

## [2026-06-16] review-fix v18 | Row-Level Security provenance and citation review

- Reviewed [Row-Level Security (RLS) in PostgreSQL 18: Implementation, Performance, Settings, and Fixes Since PostgreSQL 12 (unverified)](v18/questions/row-level-security-rls.md) against pinned `raw/postgres-18/` commit `6cb307251c5c6261286c1566496920976640108e`.
- Confirmed the core implementation claims against source: rewrite-time `securityQuals` and WCO injection, permissive OR/restrictive AND/default-deny behavior, WCO fallback to `USING`, MERGE and `ON CONFLICT` checks, relcache policy storage, policy hooks, bypass logic, planner security levels, plan-cache RLS dependency, executor WCO enforcement, partition/COPY/CTAS/logical replication boundaries, settings, and regression coverage.
- Fixed review findings: widened the `ATExecForceNoForceRowSecurity` citation to include the closing brace, changed logical replication wording from "reject writes" to reject all replicated commands including `TRUNCATE`, corrected the MERGE/partition evidence to cite `execPartition.c#ExecInitPartitionInfo`, and widened the pg_dump extension-member policy citation.
- Marked the fix-history table's commit provenance as unverified because the local `raw/postgres-18/` checkout is shallow and contains only the pinned commit; current v18 code/test/doc evidence remains cited, but commit hashes, subjects, ancestry after the v13 development stamp, and all-fixes completeness are now listed under Open Questions.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v18/index.md` summaries to match the provenance caveat. `verified_by_agent` remains `not yet`.

## [2026-06-17] question v12 | NULL handling by index type

- Filed [How NULL Values Are Handled in PostgreSQL 12 Indexes (unverified)](v12/questions/null-values-in-indexes.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- User approved correcting the prompt wording before filing; the corrected prompt is restated under `## Question` with a prompt note.
- Covered the shared index path (`FormIndexDatum`/executor `isnull[]`/`index_insert`/`aminsert`), `IndexTuple` NULL bitmaps, `ScanKey` NULL flags, and the planner's `amsearchnulls` gate for generic `IS NULL` and `IS NOT NULL` index quals.
- Added per-index-type sections for B-tree, hash, GiST, SP-GiST, GIN, BRIN, and contrib Bloom, including storage/summary behavior, plain `NullTest` support, uniqueness or opclass boundaries where relevant, and direct regression-test coverage or explicit test absence.
- Updated `wiki/index.md`, `wiki/versions.md`, and `wiki/v12/index.md`; `.wiki-runtime/venv/bin/python scripts/wiki_lint` reported 0 errors and 0 warnings.

## [2026-06-17] review-fix v12 | NULL handling by index type

- Reviewed [How NULL Values Are Handled in PostgreSQL 12 Indexes (unverified)](v12/questions/null-values-in-indexes.md) against pinned `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42` (12.2).
- Re-checked the shared executor/index AM path, `amsearchnulls` planner gate, B-tree/hash/GiST/SP-GiST/GIN/BRIN/Bloom insertion and scan behavior, docs, and regression coverage against the pinned checkout.
- Fixed review findings: removed the `_hash_checkqual` runtime claim because its NULL-checking body is under `#ifdef NOT_USED`; corrected stale GiST `gistindex_keytest` citation ranges; corrected SP-GiST scan citations from stale `resetSpGistScanOpaque`/`spgWalk` line ranges to `spgPrepareScanKeys`, `resetSpGistScanOpaque`, and `spgAddStartItem`; and reworded the B-tree regression-test paragraph because the cited block shows result coverage, not `EXPLAIN` plan-shape coverage.
- Advanced `verified_by_agent` to `GPT-5-5-XHigh-Thinking 2026-06-17T17:30:21Z`; `verified:` stays human-only `false`, so the page title keeps `(unverified)`. Existing index and landing-page summaries remain accurate.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-18] repin v19 | fetch latest master and review all questions

- Fetched upstream PostgreSQL `master` for `raw/postgres-19/` and repinned from `e18b0cb7344cb4bd28468f6c0aeeb9b9241d30aa` to `ff8bec8c460a13bedbb416d8697f4675a0709ce8` (59 new commits; latest commit `ff8bec8c`, 2026-06-18, `Create TOAST table for partitions made by MERGE/SPLIT PARTITION`).
- Reviewed all v19 question pages and the mandatory v19 codebase navigation guide against the changed-file range. No `pg_plan_advice` module/doc/test files changed in the range; no autovacuum, parallel-vacuum, or visibility-map/pruning feature files changed in the range.
- Updated [How the REPACK Command Works in PostgreSQL 19, and Its 42 Feature-Scope Commits (unverified)](v19/questions/repack-command.md) for the new explicit REPACK commit `e2a8cabc` (`concurrent repack: check there are no leftover toast attribs`), adding the `restore_tuple()` leftover separately-stored-attribute check and refreshing affected `repack.c` / `parsenodes.h` line ranges.
- Added `## Contents` blocks to the three v19 question pages while touching them for the repin, and refreshed the v19 codebase guide's shifted `parsenodes.h` citations.
- Updated `wiki/index.md`, `wiki/v19/index.md`, and `wiki/versions.md`. `verified_by_agent` remains `not yet` on v19 pages because this was a repin and changed-file review, not a full claim-by-claim re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY opening and preconditions

- Reviewed the opening `## Answer` prose and `### Preconditions and restrictions`
  section of [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Tightened the opening lock claim to cite the actual `LockConflicts` table and
  the utility/`DefineIndex` `ShareUpdateExclusiveLock` selection, instead of
  relying only on `lockdefs.h` comments; added the `validate_index` overview
  citation for the correctness rationale.
- Fixed the preconditions lead-in so temporary tables are described accurately as
  disabling concurrency rather than being rejected.
- `verified_by_agent` remains `not yet` because this was a scoped section review,
  not a full-page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY pg_index state flags

- Reviewed the `### The three pg_index state flags` section of [How CREATE
  INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Tightened the `indislive` wording: in v12 it means backends may touch the
  index at all and include it in HOT-safety decisions, not that entries are
  already inserted into it. `indisready` is the executor insertion gate, and
  `indisvalid` is the planner-use gate.
- Added direct citations for the `pg_index` flag declarations, `RelationGetIndexList`
  live-index filtering / HOT-safety treatment, `BuildIndexInfo` /
  `ExecInsertIndexTuples` ready-for-inserts handling, and `plancat.c` invalid-index
  skip; updated the Evidence Map and Source References.
- `verified_by_agent` remains `not yet` because this was a scoped section review,
  not a full-page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY step-by-step implementation

- Reviewed the `### Step-by-step implementation` section of [How CREATE INDEX
  CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed the four-transaction outline, two heap scans, and three waits against
  the `DefineIndex` concurrent branch, `index_concurrently_build`,
  `validate_index`, `WaitForOlderSnapshots`, and the command-end
  `finish_xact_command` commit path.
- Tightened wording and citations: Transaction 1 now says the index is cataloged
  but not built yet (`INDEX_CREATE_SKIP_BUILD`), the post-commit HOT-safety
  sentence cites the direct comments, Wait 2 mirrors the "read-only for updates"
  source wording, and Wait 3 describes the actual advertised-`xmin` / VXID filter
  used by `GetCurrentVirtualXIDs`.
- Updated the Evidence Map and Source References for the step-by-step phase
  boundaries, skip-build path, HOT-safety comments, Wait 3 filter, and final
  command-end commit.
- `verified_by_agent` remains `not yet` because this was a scoped section review,
  not a full-page re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY first build scan visibility

- Reviewed the `### The first build scan's tuple-visibility rule` section of
  [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed the core rule: `index_concurrently_build` sets `ii_Concurrent`, the
  heap table AM keeps `OldestXmin` invalid and uses an MVCC snapshot, and the
  build loop relies on the heap scan's visibility filtering instead of the
  `SnapshotAny` / `HeapTupleSatisfiesVacuum` path.
- Tightened wording around the candidate tuple set: the heap scan feeds only
  MVCC-visible tuples into predicate / AM callback filtering, so "indexes exactly
  the live heap tuples" was too broad for partial indexes and AM callbacks.
- Added the B-tree parallel-build path to the section and Evidence Map: B-tree is
  the only v12 AM with parallel build support, and parallel workers still call
  `table_index_build_scan` with the shared concurrent flag and parallel scan
  descriptor.
- Reworded the recently-dead claim: concurrent build does not use the
  normal-build `HEAPTUPLE_RECENTLY_DEAD` inclusion branch, but a tuple visible to
  the MVCC snapshot can still be indexed if another transaction deletes it while
  the scan is running.
- Updated Context Reviewed, Evidence Map, and Source References. `verified_by_agent`
  remains `not yet` because this was a scoped section review, not a full-page
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY table lock timeline

- Reviewed the `### All steps and locks required on the table` section of [How
  CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed the lock model: the heap table uses transaction-level
  `ShareUpdateExclusiveLock` at command start/build/validate, plus a
  session-level `ShareUpdateExclusiveLock` across transaction gaps; standard
  transaction locks release at commit while the session lock persists until
  `UnlockRelationIdForSession`.
- Tightened the Wait 1/Wait 2 wording: `WaitForLockers` does not acquire a
  table lock, reads current conflicting holders through `GetLockConflicts`, skips
  lock waiters, and waits on VXIDs; the source comments describe the target as
  transactions with the table open for write / with the index still read-only for
  updates.
- Fixed the timeline table so Wait 1 and Wait 2 happen before the build/validate
  heap `ShareUpdateExclusiveLock` reacquisition, separated target-index
  `RowExclusiveLock` from heap locks, and noted that Txn 4 does not open a heap
  transaction lock before setting `indisvalid`.
- Split the stronger-lock examples (`ShareRowExclusiveLock`, `ExclusiveLock`,
  `AccessExclusiveLock`) and added the v12 lock docs citation for command
  examples.
- Updated the Evidence Map and Source References. `verified_by_agent` remains
  `not yet` because this was a scoped section review, not a full-page
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.

## [2026-06-19] review-fix v12 | CREATE INDEX CONCURRENTLY inter-builder interactions

- Reviewed the `### How concurrent index builds interact with each other`
  section of [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12
  (unverified)](v12/questions/create-index-concurrently.md) against pinned
  `raw/postgres-12/` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- Confirmed the current-source behavior: same-table CIC commands serialize on
  self-conflicting `ShareUpdateExclusiveLock`; different-table CIC/RIC commands
  can interact only through the same-database old-snapshot
  `WaitForOlderSnapshots` path; the builder drops its reference snapshot and
  commits into a no-`xmin` transaction before Wait 3.
- Tightened the section's evidence: widened `WaitForOlderSnapshots` citations to
  include its explanatory comments, recorded that v12 has no `PROC_IN_SAFE_IC`
  symbol, and removed the stale Context Reviewed claim that this scoped pass
  had rechecked the earlier history commits.
- Clarified the `multiple-cic` isolation test: it proves different-table CIC
  commands can overlap, but the visible wait is caused by the advisory-lock
  predicate used by the test, not by a CIC table-lock conflict.
- Updated the Evidence Map and Context Reviewed. `verified_by_agent` remains
  `not yet` because this was a scoped section review, not a full-page
  re-verification.
- `.wiki-runtime/venv/bin/python scripts/wiki_lint`: 0 errors, 0 warnings.
