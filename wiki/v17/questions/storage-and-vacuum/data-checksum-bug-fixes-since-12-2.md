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

The causal distinction is: a page must first *receive* the correct checksum when written, then the reader or backup tool must *select the right pages* and compare their checksums; recovery must also preserve changes to auxiliary pages. A fix in one stage does not make the others infallible ([bufpage.c#PageSetChecksumInplace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L1542-L1549), [basebackup.c#verify_page_checksum](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1980-L2013), [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L206-L246)).

### Fixes and mitigations

The release classification comes from release tags fetched into this PostgreSQL 17 checkout's git history: match each fix to its backport commit, find the first `REL_NN_M` tag containing it, compare the preceding tag, and check for reverts before the release. A backport hash is *not* interchangeable with the original hash on the v17 development line. The current behavior beside each row is cited only from the pinned v17 implementation; the tag names and historical hashes identify the source-history evidence. The tables are sorted by earliest release *version*, not commit date.

#### Backported to an older release branch

| Oldest released backport (hash) | V17-history fix | Fault and correction | Evidence at the v17 pin |
|---|---|---|---|
| **11.8** (`8f9aba1874`); first in the v12 line: **12.3** (`f6e8e8b380`) | `428a2609ef` | Offline `pg_checksums` used to descend into *other PostgreSQL versions'* directories at a shared [tablespace](../../../glossary.md#tablespace) location; `--enable` could even rewrite their pages. It now descends into this cluster's `TABLESPACE_VERSION_DIRECTORY` only. | [pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L383-L420) |
| **11.19** (`5eaf3e375d`); first in the v12 line: **12.14** (`7dd39e9e80`) | `d6a3dbe14f` | A theoretical torn-page hazard arose when replay set the heap page's all-visible flag without advancing its page [LSN](../../../glossary.md#lsn) under checksum/hint logging. The fix advances the LSN when the heap page is protected by a [full-page image](../../../glossary.md#full-page-image); without that protection it must *not* advance the LSN. | [visibilitymap.c#visibilitymap_set](../../../../raw/postgres-17/src/backend/access/heap/visibilitymap.c#L314-L337) |
| **12.7** (`24bcce3bd6`) | `2eb1fc8b1a` | `pg_checksums --progress` counted whole files in the total but omitted new, unchecksummed pages from progress, so the reported percentage could finish short of 100%. The byte counter now advances before skipping a new page. This fixes reporting, not verification. | [pg_checksums.c#scan_file](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L196-L226), [pg_checksums.c#progress_report](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L123-L154) |
| **14.23** (`f5d1038d92`); v17: **17.10** (`1cf010f216`) | `1cf010f216` | During [WAL](../../../glossary.md#wal) replay with checksums enabled, `MarkBufferDirtyHint()` did not mark a modified [free space map](../../../glossary.md#free-space-map) page dirty, so its updated free-space hint could be lost after eviction. Replay now uses `MarkBufferDirty()` for a changed map page. It is a recovery/performance fix, not a proof of user-data corruption. | [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L206-L246) |
| **15.3** (`349803b18f`) | `33352b9279` | In an online [base backup](../../../glossary.md#base-backup), a checksum failure triggers a reread. If a concurrent truncation makes that reread hit EOF at the first block of a chunk, checking EOF too late could pass a zero-length chunk to the backup sink and hit an assertion. The reread now checks EOF before revalidating or sending it. | [basebackup.c#read_file_data_into_buffer](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1870-L1927) |
| **15.7** (`29f0052380`) | `c1fc502f59` | Server-side checksum scanning could mistake a macOS `.DS_Store` file for a data file. The offline scanner now skips it. Backports to 12 (`76bb6dd2e5`), 13 (`d3fdfdcd1c`) and 14 (`aeee173d22`) were **reverted** by `588f370b16`, `077dec8b6c` and `99cd749ce2` before their next tagged releases; those releases lack the skip. | [pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L310-L338) |

#### First released without an older minor-release backport

| First release | V17-history fix | Fault and correction | Evidence at the v17 pin |
|---|---|---|---|
| **13.0** | `addd034ae1` | Corrected a newly introduced checksum calculation regression: reduction by `65536` followed by `+1` could wrap a 16-bit result to zero. The calculation now reduces by `65535` then adds one. This repaired a post-12.2 intermediate change, not a claim that 12.2 had that regression. | [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-17/src/include/storage/checksum_impl.h#L186-L214) |
| **14.0** | `c47a240fe6` | The then-new sorted [GiST build](../../../glossary.md#gist-build-method) wrote pages outside the buffer manager without setting their checksums. The commit inserted checksum calculation before its direct writes. In v17 the sorted build instead uses the bulk writer, which sets each page's checksum before writing it. | [gistbuild.c#gist_indexsortbuild](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L400-L453), [bulk_write.c#flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L278-L283) |
| **17.0** | `025584a168` | Base backup used to assume that nearly every plain file under a relation directory was checksummed, producing false checksum complaints for stray non-relation files. It now uses the parsed non-temporary relation filename both for relation-specific exclusions and to decide whether to verify a page checksum; it can still copy other files. | [basebackup.c#sendDir](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1303-L1343), [basebackup.c#sendFile](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1600-L1614) |
| **17.0** | `205db0114e` | An incremental-backup chain can mix checksum settings. When the final backup has checksums enabled, `pg_combinebackup` now **warns** if a preceding backup differs; it does not recalculate checksums or make the output safe automatically. | [pg_combinebackup.c#check_control_files](../../../../raw/postgres-17/src/bin/pg_combinebackup/pg_combinebackup.c#L594-L658) |
| **17.11** | `c0d9864f5c` | Heap changes clearing [visibility map](../../../glossary.md#visibility-map) bits did not register their map buffers in the WAL record. This could omit changed map blocks from an incremental backup; it also omitted their full-page images, which mattered for torn map pages. The fix registers modified map buffers with the heap WAL record and keeps a replay path for old records. Checksums could detect/zero a torn map page, but did *not* prevent the incremental-backup omission. | [heapam.c#heap_insert](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L2160-L2182), [heapam.c#heap_insert-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L2260-L2278), [heapam.c#heap_update-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9433-L9451), [heapam.c#heap_xlog_update](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L10320-L10410) |

The 2026 rows are included because the **17.11 pinned branch** contains them; 17.0 had neither fix. In particular, with checksums off, the normal path does not compare page checksums; with them on, the page reader and recovery's hint-dirtying behavior change, which explains why the FSM fault was checksum-dependent ([bufpage.c#PageIsVerifiedExtended](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L98-L109), [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L230-L245)).

### What these fixes do not imply

Do not conflate a data-page checksum with the [control file](../../../glossary.md#control-file)'s CRC, a backup manifest digest, or a [WAL](../../../glossary.md#wal) checksum. For example, `57cb806308` protects `pg_control` updates against a *bad control-file checksum*, not a data-page checksum; the data-page calculation uses `pd_checksum` and the block number. `4c9f50d116` avoids needless rewrites on repeated offline `--enable` runs, but is an optimization, not a correction of a wrong checksum ([xlog.c#XLogReportParameters](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L8182-L8194), [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-17/src/include/storage/checksum_impl.h#L193-L214), [pg_checksums.c#scan_file](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L237-L258)).

Nor do checksums prove every page valid: a genuinely all-zero page passes the page verifier, while `pg_checksums` skips pages it recognizes as new. The backup verifier also skips new pages and pages whose LSN is at or after the start of the backup, since concurrent writes can make their read appear torn ([bufpage.c#PageIsVerifiedExtended](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L98-L141), [pg_checksums.c#scan_file](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L215-L226), [basebackup.c#verify_page_checksum](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1995-L2013)).

## Context Reviewed

- Pinned `REL_17_STABLE` checkout at `786db8dcf168bd9df8f55047337525ac19118b1c`; inspected its original commits, backport copies and fetched `REL_11_*` through `REL_17_*` release tags in the same Git repository. Checked the previous tag and reverts, including `.DS_Store`'s rejected v12-v14 backports. No other checkout is cited.
- Read the writer/reader and error boundaries in `checksum_impl.h`, `bufpage.c`, `pg_checksums.c`, sorted GiST build and the bulk writer, base-backup selection and reread, `pg_combinebackup`, WAL/VM replay, and FSM recovery; checked relevant `pg_checksums` TAP tests and the same-checkout release notes. No server or measurement was run.
- The version's existing common concept pages cover B-tree, GIN and other index-bloat test protocols, not data checksums; none was changed.

## Evidence Map

| Claim group | Primary pinned implementation | History used to identify the change |
|---|---|---|
| Checksum computation, GiST writes, offline scanning | [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-17/src/include/storage/checksum_impl.h#L186-L214), [bulk_write.c#flush](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L278-L283), [pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L299-L420) | `428a2609ef`, `addd034ae1`, `c47a240fe6`, `2eb1fc8b1a`, `c1fc502f59` |
| Backup page selection, races and mixed settings | [basebackup.c#sendFile](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1600-L1614), [basebackup.c#read_file_data_into_buffer](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1844-L1927), [pg_combinebackup.c#check_control_files](../../../../raw/postgres-17/src/bin/pg_combinebackup/pg_combinebackup.c#L594-L658) | `33352b9279`, `025584a168`, `205db0114e` |
| Recovery and map pages | [visibilitymap.c#visibilitymap_set](../../../../raw/postgres-17/src/backend/access/heap/visibilitymap.c#L314-L337), [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L206-L246), [heapam.c#heap_update-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9433-L9451) | `d6a3dbe14f`, `1cf010f216`, `c0d9864f5c` |
| Earliest released backport versus a reverted candidate | [pg_checksums.c#scan_directory](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L310-L338) | Release-tag ancestry of `8f9aba1874`, `5eaf3e375d`, `24bcce3bd6`, `349803b18f`, `29f0052380` and `f5d1038d92`; the 12-14 `.DS_Store` reverts `588f370b16`, `077dec8b6c`, `99cd749ce2` before their respective release tags |

## Open Questions

- This is a selection of direct data-checksum failures and mitigations, not an audit of every WAL, storage or filesystem bug that a checksum might expose. The FSM and VM rows concern more than checksum verification: the former addresses lost recovery updates when checksums are enabled, while the latter also addresses an incremental-backup omission ([freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L230-L245), [heapam.c#heap_update-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9433-L9451)).
- Release-tag ancestry establishes *first released inclusion*, not whether a historical defect is reproducible on every intervening minor release. The 2024 `pg_combinebackup` warning explicitly leaves mixed-setting output requiring an operator workaround ([pg_combinebackup.c#check_control_files](../../../../raw/postgres-17/src/bin/pg_combinebackup/pg_combinebackup.c#L629-L658)).

## Source References

- [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-17/src/include/storage/checksum_impl.h#L186-L214)
- [pg_checksums.c#scan_file](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L175-L275)
- [basebackup.c#read_file_data_into_buffer](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L1844-L1927)
- [freespace.c#XLogRecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L206-L246)
- [heapam.c#heap_update-WAL](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9433-L9451)

## Navigation

- [v17/index](../../index.md)
- [wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
