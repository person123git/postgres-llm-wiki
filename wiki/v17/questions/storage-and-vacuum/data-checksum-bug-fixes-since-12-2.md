---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Data Checksum Bug Fixes Since PostgreSQL 12.2, Through PostgreSQL 17.11 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Scope and baseline](#scope-and-baseline)
  - [Fixes and mitigations](#fixes-and-mitigations)
  - [What these fixes do not imply](#what-these-fixes-do-not-imply)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, list bug fixes since version 12.2 related to data checksums.

Follow-up: Group the fixes by the oldest minor release to which each was back-patched.

## Answer

### Scope and baseline

This is a selected list of data-page [checksum](../../../glossary.md#data-checksums) fixes and closely related fixes to checksum checking, offline conversion, and recovery through the pinned PostgreSQL 17.11 commit. The first table groups fixes by the **oldest tagged minor release containing an effective [back-patch](../../../glossary.md#back-patch)**, including older branches where a fix was back-patched even though this question starts at 12.2. The second table separates fixes first shipped on a development branch or directly in 17.11, with no earlier minor-release backport at the pin. A release label does **not** mean the old defect affected every release between 12.2 and that label. PostgreSQL 17 normally initializes a cluster with checksums **off**; `initdb -k` turns them on. `pg_checksums` changes or checks the state offline, after a clean shutdown. An initialized page is checked on read when checksums are on, but an all-zero page is accepted without comparison ([initdb.c#checksums](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L167), [initdb.c#checksums-option](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L2511), [pg_checksums.c#cluster-state](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L563-L582), [bufpage.c#PageIsVerifiedExtended](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L98-L141)).

A checksum protects a page only if three separate stages all work. Each row below belongs to one stage, and a fix in one stage does not make the others infallible:

1. **Write.** The writer computes the checksum and stores it in the page header just before the page goes to disk, for pages written through the buffer manager and for pages written directly, such as by the [bulk writer](../../../glossary.md#bulk-writer) ([bufpage.c#PageSetChecksumInplace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L1542-L1549), [bulk_write.c#flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L278-L283)). `pg_combinebackup` also writes pages, but does not recompute their checksums ([pg_combinebackup.sgml#app-pgcombinebackup-limitations](../../../../raw/postgres-17/doc/src/sgml/ref/pg_combinebackup.sgml#L277-L283)). The GiST row fixes this stage, and the mixed-backup row warns about it.
2. **Select and compare.** A reader, `pg_checksums` or the base backup must pick only real relation pages, then compare each page's stored checksum with a recomputed one. The base backup also skips pages too new to compare safely ([pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L299-L420), [basebackup.c#verify_page_checksum](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1980-L2013)). The tablespace, `pg_internal.init`, end-of-file, `.DS_Store` and file-selection rows fix this stage; the progress row fixes only the progress report of the same scan.
3. **Recover.** WAL replay must keep the page LSN and the dirty state that [torn-page](../../../glossary.md#torn-page) protection relies on, including for the auxiliary map forks ([heapam.c#heap_xlog_visible](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9822-L9848), [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L206-L247)). The heap-LSN, FSM and VM rows fix this stage.

### Fixes and mitigations

The release classification comes from release tags fetched into this PostgreSQL 17 checkout's git history: match each fix to its backport commit, find the first `REL_NN_M` tag containing it, compare the preceding tag, and check for reverts before the release. A backport hash is *not* interchangeable with the original hash on the v17 development line. The current behavior beside each row is cited only from the pinned v17 implementation; the tag names and historical hashes identify the source-history evidence. The tables are sorted by earliest release *version*, not commit date.

#### Backported to an older release branch

| Oldest released backport (hash) | V17-history fix | Fault and correction | Evidence at the v17 pin |
|---|---|---|---|
| **11.8** (`da2a7180aa`); first in the v12 line: **12.3** (`a8beece956`) | `bf883b211e` | The relation cache writes its init file through a temporary file named `pg_internal.init.<pid>`, so a server that crashed at a bad moment could leave one behind. The exclusion lists of `pg_checksums` and the base backup matched `pg_internal.init` by exact name only, so such a leftover was not excluded, which the commit says could cause failures when verifying checksums. Both lists now match `pg_internal.init` as a name prefix. | [relcache.c#write_relcache_init_file](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L6533-L6541), [pg_checksums.c#skip](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L108-L117), [pg_checksums.c#skipfile](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L157-L173), [basebackup.c#excludeFiles](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L199-L203) |
| **11.8** (`8f9aba1874`); first in the v12 line: **12.3** (`f6e8e8b380`) | `428a2609ef` | Offline `pg_checksums` used to descend into *other PostgreSQL versions'* directories at a shared [tablespace](../../../glossary.md#tablespace) location; `--enable` could even rewrite their pages. It now descends into this cluster's `TABLESPACE_VERSION_DIRECTORY` only. | [pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L383-L420) |
| **11.19** (`5eaf3e375d`); first in the v12 line: **12.14** (`7dd39e9e80`) | `d6a3dbe14f` | When replay of a visibility-map record set a heap page's all-visible flag, `heap_xlog_visible` did not advance that page's [LSN](../../../glossary.md#lsn). The commit says this bypassed the `minRecoveryPoint` protection and left a theoretical torn-page hazard with checksums. The fix advances the heap page LSN on that redo path whenever `XLogHintBitIsNeeded()` is true, that is, when checksums or `wal_log_hints` are on. When the record's [full-page image](../../../glossary.md#full-page-image) of the heap page was restored instead, replay does nothing more to it. | [heapam.c#heap_xlog_visible](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9822-L9848), [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L118) |
| **12.7** (`24bcce3bd6`) | `2eb1fc8b1a` | `pg_checksums --progress` counted whole files in the total but omitted new, unchecksummed pages from progress, so the reported percentage could finish short of 100%. The byte counter now advances before skipping a new page. This fixes reporting, not verification. | [pg_checksums.c#scan_file](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L196-L226), [pg_checksums.c#progress_report](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L123-L154) |
| **14.23** (`f5d1038d92`); v17: **17.10** (`1cf010f216`) | `1cf010f216` | During [WAL](../../../glossary.md#wal) replay, `XLogRecordPageWithFreeSpace()` marked a changed [free space map](../../../glossary.md#free-space-map) page with `MarkBufferDirtyHint()`. During recovery that function returns without dirtying a permanent buffer whenever `XLogHintBitIsNeeded()` is true, that is, when checksums **or** `wal_log_hints` are on. So the updated free-space value could be lost when the page was evicted. Replay now calls `MarkBufferDirty()` when the map value changed. The commit describes stale free-space data that stalls inserts after a switchover, not user-data corruption. | [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L206-L247), [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5099-L5112), [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L118) |
| **15.3** (`349803b18f`) | `33352b9279` | In an online [base backup](../../../glossary.md#base-backup), a page whose checksum fails is reread once. If a concurrent truncation makes that reread hit end-of-file at the first block of a chunk, the reader shrinks the chunk to zero bytes. The caller tested for end-of-file only *before* checksum validation, so it missed that zero and passed a zero-length chunk to the backup sink, which fails an assertion. The fix moved the caller's `cnt == 0` test to *after* validation, just before the chunk is archived. The reread's own end-of-file handling was not changed. | [basebackup.c#read_file_data_into_buffer](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1891-L1904), [basebackup.c#sendFile-EOF](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1756-L1762) |
| **15.7** (`29f0052380`) | `c1fc502f59` | The macOS Finder can leave a `.DS_Store` file in a data directory. Tools that expect only PostgreSQL files there could trip over it. Without the skip, for example, `pg_checksums` would parse `.DS_Store` as a relation file name and stop with an "invalid segment number" error, because the text after its first `.` is not a number. `pg_checksums`, the server's base backup and `pg_rewind` now skip it. Backports to 12 (`76bb6dd2e5`), 13 (`d3fdfdcd1c`) and 14 (`aeee173d22`) were **reverted** by `588f370b16`, `077dec8b6c` and `99cd749ce2` before their next tagged releases; those releases lack the skip. | [pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L332-L334), [pg_checksums.c#segment-parse](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L355-L363), [basebackup.c#sendDir](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1263-L1265), [filemap.c#decide_file_action](../../../../raw/postgres-17/src/bin/pg_rewind/filemap.c#L711-L713) |

#### First released without an older minor-release backport

| First release | V17-history fix | Fault and correction | Evidence at the v17 pin |
|---|---|---|---|
| **13.0** | `addd034ae1` | Commit `0065174324` (2020-03-05, on the 13 development line) changed the final reduction from `% 65535` to `% 65536`. For most 32-bit inputs that gives a different result from the 12.2 formula, so it changed nearly every page's checksum. It could also wrap the `+ 1` result to zero in the 16-bit field. `addd034ae1` restored `% 65535` the next day. Both commits first appear in `REL_13_0`, so no tagged release ever shipped the regression, and 12.2 never had it. | [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-17/src/include/storage/checksum_impl.h#L210-L214) |
| **14.0** | `c47a240fe6` | The then-new sorted [GiST build](../../../glossary.md#gist-build-method) wrote pages outside the buffer manager without setting their checksums. The commit inserted checksum calculation before its direct writes. In v17 the sorted build instead uses the [bulk writer](../../../glossary.md#bulk-writer), since `8af2565248` (2024-02-23) moved it there; the bulk writer sets each page's checksum before writing it. | [gistbuild.c#gist_indexsortbuild](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L400-L453), [bulk_write.c#flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L278-L283) |
| **17.0** | `025584a168` | Base backup used to assume that nearly every plain file under a relation directory was checksummed, producing false checksum complaints for stray non-relation files. It now uses the parsed non-temporary relation filename both for relation-specific exclusions and to decide whether to verify a page checksum; it can still copy other files. | [basebackup.c#sendDir](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1303-L1343), [basebackup.c#sendFile](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1600-L1614) |
| **17.0** | `205db0114e` | An [incremental backup](../../../glossary.md#incremental-backup) chain can mix checksum settings. When the final backup has checksums enabled, `pg_combinebackup` now **warns** if a preceding backup differs; it does not recalculate checksums or make the output safe automatically. | [pg_combinebackup.c#check_control_files](../../../../raw/postgres-17/src/bin/pg_combinebackup/pg_combinebackup.c#L594-L658) |
| **17.11** | `c0d9864f5c` | Heap changes clearing [visibility map](../../../glossary.md#visibility-map) bits did not register their map buffers in the WAL record. This could omit changed map blocks from an incremental backup; it also omitted their full-page images, which mattered for torn map pages. The fix registers modified map buffers with the heap WAL record and keeps a replay path for old records. Checksums could detect/zero a torn map page, but did *not* prevent the incremental-backup omission. | [heapam.c#heap_insert](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L2160-L2182), [heapam.c#heap_insert-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L2260-L2278), [heapam.c#heap_update-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9433-L9451), [heapam.c#heap_xlog_update](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L10320-L10410) |

The 2026 rows are included because the **17.11 pinned branch** contains them; 17.0 had neither fix.

Most rows matter only on a cluster with data checksums on, which is not the default. `initdb` defaults `data_checksums` to false ([initdb.c#checksums](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L167)). With checksums off, the read path does not compare page checksums, the base backup does not verify them, and `pg_checksums --check` refuses to run ([bufpage.c#PageIsVerifiedExtended](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L98-L109), [basebackup.c#sendFile](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1607-L1614), [pg_checksums.c#cluster-state](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L572-L574)).

The heap-LSN and FSM replay rows depend on a wider condition, `XLogHintBitIsNeeded()`, which is true when checksums are on **or** `wal_log_hints` is on ([xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L118)). `wal_log_hints` also defaults off: its boot value is `false`, and its `postmaster` context means a change needs a restart ([guc_tables.c#wal_log_hints](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1171-L1177)). Turning either one on changes two replay paths:

| Condition | Heap all-visible replay (11.19 row) | FSM replay before the 14.23/17.10 fix |
|---|---|---|
| Checksums and `wal_log_hints` both off (default) | The heap page LSN is not advanced, as the comment beside the code explains ([heapam.c#heap_xlog_visible](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9825-L9837)) | `MarkBufferDirtyHint()` dirties the buffer, so the change is written out ([bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5147-L5168)) |
| Checksums or `wal_log_hints` on | The heap page LSN is advanced ([heapam.c:9836-9837](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9836-L9837)) | `MarkBufferDirtyHint()` returns early during recovery, so the change could be lost ([bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5099-L5112)); the fix calls `MarkBufferDirty()` instead ([freespace.c:244-245](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L244-L245)) |

The `pg_combinebackup` warning has its own condition: it fires only when the final backup has checksums enabled and an earlier one differs ([pg_combinebackup.c#check_control_files](../../../../raw/postgres-17/src/bin/pg_combinebackup/pg_combinebackup.c#L629-L637)).

#### Regression tests at the pin

| Row | Test at the pin | Notes |
|---|---|---|
| `pg_internal.init` prefix (11.8) | [002_actions.pl:115-116](../../../../raw/postgres-17/src/bin/pg_checksums/t/002_actions.pl#L115-L116), [010_pg_basebackup.pl:257](../../../../raw/postgres-17/src/bin/pg_basebackup/t/010_pg_basebackup.pl#L257) | `bf883b211e` updated both TAP tests. |
| Foreign tablespaces (11.8) | [002_actions.pl:241-246](../../../../raw/postgres-17/src/bin/pg_checksums/t/002_actions.pl#L241-L246) | Added by `428a2609ef`. |
| `.DS_Store` (15.7) | [002_actions.pl:118-122](../../../../raw/postgres-17/src/bin/pg_checksums/t/002_actions.pl#L118-L122), [010_pg_basebackup.pl:262-266](../../../../raw/postgres-17/src/bin/pg_basebackup/t/010_pg_basebackup.pl#L262-L266), [003_extrafiles.pl:57-58](../../../../raw/postgres-17/src/bin/pg_rewind/t/003_extrafiles.pl#L57-L58) | Skipped on macOS. |
| Checksum formula (13.0) | [pageinspect/sql/checksum.sql:23-31](../../../../raw/postgres-17/contrib/pageinspect/sql/checksum.sql#L23-L31), [checksum.out](../../../../raw/postgres-17/contrib/pageinspect/expected/checksum.out#L1-L40) | Added two days after the fix by `38ce06c37e`, whose message names `0065174324` and `addd034ae1`. |
| VM block registration (17.11) | [002_blocks.pl:88-96](../../../../raw/postgres-17/src/bin/pg_walsummary/t/002_blocks.pl#L88-L96) | `c0d9864f5c` now expects the VM block in the WAL summary. |
| Heap LSN, progress, FSM, end-of-file, GiST, file selection, mixed backups | None | The fix commits `d6a3dbe14f`, `2eb1fc8b1a`, `1cf010f216`, `33352b9279`, `c47a240fe6`, `025584a168` and `205db0114e` add or change no test file. |

### What these fixes do not imply

Do not conflate a data-page checksum with the [control file](../../../glossary.md#control-file)'s CRC, a backup manifest digest, or a [WAL](../../../glossary.md#wal) checksum. For example, `57cb806308` protects `pg_control` updates against a *bad control-file checksum*, not a data-page checksum. The data-page calculation covers the page contents, with `pd_checksum` treated as zero, plus the block number. `4c9f50d116` avoids needless rewrites on repeated offline `--enable` runs, but is an optimization, not a correction of a wrong checksum ([xlog.c#XLogReportParameters](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L8182-L8194), [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-17/src/include/storage/checksum_impl.h#L196-L208), [pg_checksums.c#scan_file](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L237-L258)).

Nor do checksums prove every page valid: a genuinely all-zero page passes the page verifier, while `pg_checksums` skips pages it recognizes as new. The backup verifier also skips new pages and pages whose LSN is at or after the start of the backup, since concurrent writes can make their read appear torn ([bufpage.c#PageIsVerifiedExtended](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L98-L141), [pg_checksums.c#scan_file](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L215-L226), [basebackup.c#verify_page_checksum](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1995-L2013)).

## Context Reviewed

- Pinned `REL_17_STABLE` checkout at `786db8dcf168bd9df8f55047337525ac19118b1c`; inspected its original commits, backport copies and fetched `REL_11_*` through `REL_17_*` release tags in the same Git repository. Checked the previous tag and reverts, including `.DS_Store`'s rejected v12-v14 backports. No other checkout is cited. Commits on `REL_17_STABLE` after the pin are not evidence for this page.
- Read the writer/reader and error boundaries in `checksum_impl.h`, `bufpage.c`, `bufmgr.c#MarkBufferDirtyHint`, `pg_checksums.c`, the base backup's exclusion list, file selection, reread and end-of-file test, `pg_rewind`'s file filter, sorted GiST build and the bulk writer, `pg_combinebackup`, `heap_xlog_visible`, VM-clearing WAL and replay, and FSM recovery. Read each fix commit's diff and file list to place its change and to establish which fixes changed tests.
- The 2026-11-06 review re-read every row against its commit diff and fixed five contradictions: the end-of-file fix, the heap LSN fix, the FSM condition, the checksum inputs and the `.DS_Store` scope. It added the `pg_internal.init` row (`bf883b211e`), the regression-test table and the non-default settings table. No server or measurement was run.
- The version's existing common concept pages cover B-tree, GIN and other index-bloat test protocols, not data checksums; none was changed.

## Evidence Map

| Claim group | Primary pinned implementation | History used to identify the change |
|---|---|---|
| Checksum computation, GiST writes, offline scanning | [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-17/src/include/storage/checksum_impl.h#L186-L214), [bulk_write.c#flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L278-L283), [pg_checksums.c#skip](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L108-L117), [pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L299-L420) | `bf883b211e`, `428a2609ef`, `0065174324`, `addd034ae1`, `38ce06c37e`, `c47a240fe6`, `8af2565248`, `2eb1fc8b1a`, `c1fc502f59` |
| Backup page selection, races and mixed settings | [basebackup.c#excludeFiles](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L199-L203), [basebackup.c#sendFile](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1600-L1614), [basebackup.c#sendFile-EOF](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1756-L1762), [basebackup.c#read_file_data_into_buffer](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1844-L1927), [pg_combinebackup.c#check_control_files](../../../../raw/postgres-17/src/bin/pg_combinebackup/pg_combinebackup.c#L594-L658) | `bf883b211e`, `33352b9279`, `025584a168`, `205db0114e` |
| Recovery and map pages | [heapam.c#heap_xlog_visible](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9822-L9848), [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5099-L5112), [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L206-L247), [heapam.c#heap_update-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9433-L9451) | `d6a3dbe14f`, `1cf010f216`, `c0d9864f5c` |
| Non-default settings | [initdb.c#checksums](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L167), [guc_tables.c#wal_log_hints](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1171-L1177), [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L118) | - |
| Earliest released backport versus a reverted candidate | [pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L332-L334) | Release-tag ancestry of `da2a7180aa`, `8f9aba1874`, `5eaf3e375d`, `24bcce3bd6`, `349803b18f`, `29f0052380` and `f5d1038d92`; the 12-14 `.DS_Store` reverts `588f370b16`, `077dec8b6c`, `99cd749ce2` before their respective release tags |

## Open Questions

- This is a selection of direct data-checksum failures and mitigations, not an audit of every WAL, storage or filesystem bug that a checksum might expose. The FSM and VM rows concern more than checksum verification: the former addresses lost recovery updates when checksums or `wal_log_hints` are on, while the latter also addresses an incremental-backup omission ([freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L230-L245), [heapam.c#heap_update-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9433-L9451)). For example, `242dfcbafa` (oldest released backport 11.9, 12.4 in the v12 line) reduced the repeated full-page images that B-tree scans wrote when they killed already-dead index items with checksums or `wal_log_hints` on, according to its commit message. It is a WAL-volume fix, not a checksum correctness fix, so it is not listed.
- The comment in `XLogRecordPageWithFreeSpace()` says `MarkBufferDirtyHint()` does nothing during recovery "if checksums are enabled". The code's condition is `XLogHintBitIsNeeded()`, which is also true with `wal_log_hints` on and checksums off. The page follows the code ([freespace.c:234-243](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L234-L243), [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5099-L5112), [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-17/src/include/access/xlog.h#L118)).
- Release-tag ancestry establishes *first released inclusion*, not whether a historical defect is reproducible on every intervening minor release. The 2024 `pg_combinebackup` warning explicitly leaves mixed-setting output requiring an operator workaround ([pg_combinebackup.c#check_control_files](../../../../raw/postgres-17/src/bin/pg_combinebackup/pg_combinebackup.c#L629-L658)).
- The page describes the pinned tree only. A repin past `786db8dcf16` would require re-checking the FSM and VM rows against the later branch history.

## Source References

- [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-17/src/include/storage/checksum_impl.h#L186-L214)
- [pg_checksums.c#scan_file](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L175-L275)
- [pg_checksums.c#skip](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L108-L117)
- [basebackup.c#excludeFiles](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L191-L225)
- [basebackup.c#read_file_data_into_buffer](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1844-L1927)
- [basebackup.c#sendFile-EOF](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1756-L1762)
- [heapam.c#heap_xlog_visible](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9788-L9848)
- [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5051-L5168)
- [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L206-L247)
- [heapam.c#heap_update-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9433-L9451)
- [filemap.c#decide_file_action](../../../../raw/postgres-17/src/bin/pg_rewind/filemap.c#L711-L713)
- [guc_tables.c#wal_log_hints](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1171-L1177)

## Navigation

- [v17/index](../../index.md)
- [wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
