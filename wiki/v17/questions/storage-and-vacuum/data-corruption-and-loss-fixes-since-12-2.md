---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Data Corruption and Data Loss Bug Fixes Since PostgreSQL 12.2, Grouped by Oldest Back-Patched Minor Release (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Scope and method](#scope-and-method)
  - [How to read the tables](#how-to-read-the-tables)
  - [Settings and conditions that gate some fixes](#settings-and-conditions-that-gate-some-fixes)
  - [Oldest back-patch in the 9.5 branch](#oldest-back-patch-in-the-95-branch)
  - [Oldest back-patch in the 9.6 branch](#oldest-back-patch-in-the-96-branch)
  - [Oldest back-patch in the 10 branch](#oldest-back-patch-in-the-10-branch)
  - [Oldest back-patch in the 11 branch](#oldest-back-patch-in-the-11-branch)
  - [Oldest back-patch in the 12 branch](#oldest-back-patch-in-the-12-branch)
  - [Oldest back-patch in the 13 branch](#oldest-back-patch-in-the-13-branch)
  - [Oldest back-patch in the 14 branch](#oldest-back-patch-in-the-14-branch)
  - [Oldest back-patch in the 15 branch](#oldest-back-patch-in-the-15-branch)
  - [Oldest back-patch in the 16 branch](#oldest-back-patch-in-the-16-branch)
  - [Oldest back-patch in the 17 branch](#oldest-back-patch-in-the-17-branch)
  - [What was left out](#what-was-left-out)
  - [Final summary](#final-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, list bug fixes since version 12.2 for bugs that could lead to data corruption or data loss. Group the fixes by the oldest minor release to which each was back-patched.

## Answer

### Short answer

The minor-release notes from 12.3 through 17.11 describe **109 fixes** for bugs that could corrupt stored data or lose it. Each row restates its release note, and any fact a row adds carries its own evidence. A few borderline inclusions are listed under [Open Questions](#open-questions). The table below groups them by the **oldest minor release that shipped each fix**. Because many fixes were [back-patched](../../../glossary.md#back-patch) to every supported branch, that oldest release is often a 9.5, 9.6, 10 or 11 release that came out on the same day as a 12.x release. The full lists follow, one section per oldest branch.

| Oldest branch | Fixes | Oldest minor releases (fixes) |
|---|---|---|
| [9.5](#oldest-back-patch-in-the-95-branch) | 12 | 9.5.22 (2), 9.5.23 (3), 9.5.24 (4), 9.5.25 (3) |
| [9.6](#oldest-back-patch-in-the-96-branch) | 9 | 9.6.22 (1), 9.6.23 (2), 9.6.24 (6) |
| [10](#oldest-back-patch-in-the-10-branch) | 9 | 10.20 (3), 10.21 (3), 10.22 (1), 10.23 (2) |
| [11](#oldest-back-patch-in-the-11-branch) | 12 | 11.8 (1), 11.19 (2), 11.20 (3), 11.21 (6) |
| [12](#oldest-back-patch-in-the-12-branch) | 14 | 12.3 (3), 12.4 (1), 12.6 (1), 12.7 (1), 12.10 (1), 12.15 (1), 12.17 (2), 12.18 (1), 12.20 (1), 12.21 (2) |
| [13](#oldest-back-patch-in-the-13-branch) | 16 | 13.1 (1), 13.5 (1), 13.7 (1), 13.13 (1), 13.17 (1), 13.19 (5), 13.21 (4), 13.22 (1), 13.23 (1) |
| [14](#oldest-back-patch-in-the-14-branch) | 17 | 14.1 (1), 14.2 (1), 14.3 (1), 14.4 (1), 14.7 (1), 14.9 (1), 14.11 (1), 14.15 (1), 14.18 (1), 14.21 (4), 14.23 (1), 14.24 (3) |
| [15](#oldest-back-patch-in-the-15-branch) | 10 | 15.2 (1), 15.3 (1), 15.5 (1), 15.6 (2), 15.8 (1), 15.14 (1), 15.15 (1), 15.18 (1), 15.19 (1) |
| [16](#oldest-back-patch-in-the-16-branch) | 5 | 16.5 (1), 16.9 (2), 16.12 (1), 16.14 (1) |
| [17](#oldest-back-patch-in-the-17-branch) | 5 | 17.1 (1), 17.3 (1), 17.6 (1), 17.8 (1), 17.11 (1) |
| **Total** | **109** | |

Three facts matter most for a PostgreSQL 17 reader:

1. **All but one fix is in the pinned 17.11 code.** 73 fixes were already in 17.0, because the `REL_17_0` tag contains their v17-line commit. 35 more arrived in 17.1 through 17.11. The one exception is the 10.23 fix for `UPDATE ... SET (cols) = (SELECT ...)` on inherited tables, which has no commit with the same subject on the v17 line.
2. **Some damage outlives the fix.** A fixed server stops creating new damage, but it does not repair what an older release already wrote. The notes recommend `REINDEX` after several index fixes. These are the GiST split fixes (12.6, 12.17), the `CONCURRENTLY` build fixes (9.5.25, 9.6.24, 14.4), BRIN (9.5.24, 11.21), hash indexes on floats (9.6.24), B-tree on `interval` (13.13), the C-locale test (17.1) and several contrib opclasses. The notes also recommend dropping and re-adding the foreign keys broken by the partition fixes (13.17, 13.21).
3. **Some fixes apply only under a non-default setting or an installed extension.** The [settings table](#settings-and-conditions-that-gate-some-fixes) names each condition, its default at the v17 pin, and the code that tests it.

### Scope and method

This page answers from source history in the pinned PostgreSQL 17 checkout. The release tags of the 9.5 through 17 branches were fetched into it; its working tree stays at the pin. The selection runs in seven steps:

1. **Read the minor-release notes.** Each stable branch keeps its own release notes, built from the commits on that branch ([RELEASE_CHANGES:128-148](../../../../raw/postgres-17/src/tools/RELEASE_CHANGES#L128-L148)). The notes for 12.3-12.22, 13.1-13.23, 14.1-14.24, 15.1-15.19 and 16.1-16.15 come from those branches' last release tags. The 17.1-17.11 notes are in the pinned file `release-17.sgml`, starting at [release-17.sgml:4-10](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L4-L10). Major-release (N.0) sections are skipped because they describe features, not individual fixes.
2. **Merge copies of the same fix.** Every note item carries a comment listing each branch's commit and hash, which `git_changelog` prints ([git_changelog:8-19](../../../../raw/postgres-17/src/tools/git_changelog#L8-L19)). Items that share any hash are one fix. A keyword pass over the note text (corruption, loss, lost, overwrite, fsync, torn, reindex, wraparound, WAL, replay, backup, replication and similar words) left 491 merged candidates.
3. **Keep a fix only when its note describes damage to data.** That means wrong or missing data in a table, index, catalog, sequence, SLRU, WAL or control file, lost durability across an operating-system crash, wrong rows modified, or data lost or corrupted while it passes through replication, a backup, or a dump and restore. Every candidate was read by hand. [What was left out](#what-was-left-out) lists the classes that were rejected.
4. **Find the oldest released back-patch.** For each fix, take the oldest branch named in the note's commit comment. Find the first release tag on that branch that contains the branch's commit(s), and check that the preceding tag does not. Then search that branch's history up to the tag for a revert of the commit. No fix on this page was reverted before its oldest release.
5. **Find the v17-line commit.** Use the `master` or `REL_17_STABLE` commit from the comment if it is an ancestor of the pin. Otherwise find a pin-ancestor commit with the identical subject line; the table marks those "matched by subject". The first `REL_17_*` tag containing it gives the first 17.x release with the fix.
6. **Locate the fix's code at the pin.** The evidence column links lines of the pinned files that are identical, apart from trailing whitespace, to lines the v17-line commit added, and that occur only once in their file. Only clusters of at least two such lines are linked. Each link is therefore the fix's own code as it stands at the pin. The label names the file and lines rather than a function, because the lookup was by line content.
7. **Link the pinned release notes** for fixes that appear in a 17.x section. That text is in the pinned checkout itself.

A release label does not mean the bug existed in every release between 12.2 and that label. A fix that appears only from 13.x or later may have repaired a bug that 12 never had. It may also have arrived after 12.22, the last 12.x tag in the checkout. The notes alone do not tell these cases apart, so the "First 12.x" column only shows where the fix is listed. 53 fixes have no 12.x release.

### How to read the tables

| Column | Meaning |
|---|---|
| Oldest release (backport) | The first tagged release on the oldest branch that contains the fix, then that branch's commit hash or hashes. |
| Area | What the bug damaged. "Wrong rows" means a data-modifying command could act on the wrong rows or versions; "Metadata" means a user-set catalog property was silently dropped; "Read path" means stored data was intact but was returned corrupted. |
| Fix and risk | A short restatement of the release note: what could go wrong, and under which condition. |
| First 12.x | The first 12.x release whose notes list the fix, or "—". |
| V17-line commit; first 17.x | The commit on the PostgreSQL 17 line, and the first 17.x release containing it. |
| Evidence at the v17 pin | Pinned lines identical to lines the fix added, and the pinned 17.x note when one exists. "History only" means no cluster of two or more such lines was found, for example because the fix only removed code, changed a single line, or was later rewritten; the commit hashes are then the evidence. |

The area labels use these concepts: [WAL](../../../glossary.md#wal), [WAL archiving](../../../glossary.md#wal-archiving), [crash recovery](../../../glossary.md#crash-recovery), [checkpoints](../../../glossary.md#checkpoint) and [fsync](../../../glossary.md#fsync), the [control file](../../../glossary.md#control-file), the [SLRU](../../../glossary.md#slru) logs and [MultiXact](../../../glossary.md#multixact), the system [catalog](../../../glossary.md#catalog) and its [in-place updates](../../../glossary.md#in-place-update), [TOAST](../../../glossary.md#toast), [HOT](../../../glossary.md#hot) chains, the [visibility map](../../../glossary.md#visibility-map) and [free space map](../../../glossary.md#free-space-map), [CONCURRENTLY](../../../glossary.md#concurrently) index builds, [B-tree](../../../glossary.md#b-tree) [deduplication](../../../glossary.md#deduplication), [GiST](../../../glossary.md#gist), [GIN](../../../glossary.md#gin), [BRIN](../../../glossary.md#brin), [hash indexes](../../../glossary.md#hash-index), [index-only scans](../../../glossary.md#index-only-scan), [parallel vacuum](../../../glossary.md#parallel-vacuum), [EvalPlanQual](../../../glossary.md#evalplanqual), [two-phase commit](../../../glossary.md#two-phase-commit), [unlogged relations](../../../glossary.md#unlogged-relation), [logical decoding](../../../glossary.md#logical-decoding), [replication slots](../../../glossary.md#replication-slot), [replication origins](../../../glossary.md#replication-origin), [apply workers](../../../glossary.md#apply-worker), [promotion](../../../glossary.md#promotion), [timelines](../../../glossary.md#timeline), [base backups](../../../glossary.md#base-backup), [incremental backups](../../../glossary.md#incremental-backup), the [bulk writer](../../../glossary.md#bulk-writer), [contrib](../../../glossary.md#contrib) modules and [data checksums](../../../glossary.md#data-checksums).

### Settings and conditions that gate some fixes

Most rows describe the default configuration. The rows below need a setting or state that differs from the default, so on a default PostgreSQL 17 server their bug could not occur. Each row gives the default at the pin, what makes the case different, and the code that tests the condition.

| Condition (rows) | Default at the v17 pin | What differs, and the mechanism |
|---|---|---|
| WAL archiving (9.5.22, 9.5.24) | `archive_mode` boot value `off`; context `postmaster`, so a change needs a restart ([guc_tables.c:4936-4943](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4936-L4943)) | The archive status that both bugs mishandled exists only when archiving is on, which `XLogArchivingActive()` tests ([xlog.h:96-98](../../../../raw/postgres-17/src/include/access/xlog.h#L96-L98)). The 9.5.24 bug needs `archive_mode = always` on a standby. |
| `wal_level = minimal` (9.6.24 tablespace) | Boot value `replica`; `postmaster` ([guc_tables.c:4987-4994](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4987-L4994)) | `RelationNeedsWAL()` skips WAL for a relation created in the current transaction only when `XLogIsNeeded()` is false, that is, below `replica` ([rel.h:628-631](../../../../raw/postgres-17/src/include/utils/rel.h#L628-L631), [xlog.h:107](../../../../raw/postgres-17/src/include/access/xlog.h#L107)). Only that skipped data was lost. |
| Logical decoding or logical replication (every "Logical replication" row, and the 12.4 replica-identity row) | `wal_level` boot value `replica` | Logical decoding refuses to start below `logical` ([logical.c:125-128](../../../../raw/postgres-17/src/backend/replication/logical/logical.c#L125-L128)), so these bugs need `wal_level = logical` and a slot or subscription. |
| Prepared transactions (9.5.25, 9.6.24 promotion and CIC rows) | `max_prepared_transactions` boot value `0`; `postmaster` ([guc_tables.c:2579-2586](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2579-L2586)) | `PREPARE TRANSACTION` fails while the setting is 0 ([twophase.c:371-376](../../../../raw/postgres-17/src/backend/access/transam/twophase.c#L371-L376)), so no prepared transaction can exist. |
| `track_commit_timestamp` (10.20) | Boot value `false`; `postmaster` ([guc_tables.c:1061-1068](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1061-L1068)) | `TransactionTreeSetCommitTsData()` returns at once when the module is inactive ([commit_ts.c:157-158](../../../../raw/postgres-17/src/backend/access/transam/commit_ts.c#L157-L158)). |
| Data checksums or `wal_log_hints` (11.19, 14.23) | `initdb` defaults checksums off ([initdb.c:167](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L167)); `wal_log_hints` boot value `false`, `postmaster` ([guc_tables.c:1171-1177](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1171-L1177)) | Both replay paths branch on `XLogHintBitIsNeeded()`, which is true when either is on ([xlog.h:118](../../../../raw/postgres-17/src/include/access/xlog.h#L118), [heapam.c:9836-9837](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9836-L9837)). |
| `streaming = parallel` subscription (16.12) | New subscriptions default to `streaming = off` ([subscriptioncmds.c:150-151](../../../../raw/postgres-17/src/backend/commands/subscriptioncmds.c#L150-L151)) | The apply worker requests parallel streaming only for `LOGICALREP_STREAM_PARALLEL` ([worker.c:4386-4394](../../../../raw/postgres-17/src/backend/replication/logical/worker.c#L4386-L4394)). |
| WAL summarization (17.8; the incremental-backup half of 17.11) | `summarize_wal` boot value `false`; context `sighup`, so a reload applies it ([guc_tables.c:1787-1794](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1787-L1794)) | The server refuses an incremental backup without it ([basebackup.c:788-791](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L788-L791)). The 17.11 missing full-page-image half applies under the default. |
| `vacuum_defer_cleanup_age` (12.15) | The setting does not exist at the pin. Commit `1118cd37eb` removed it before 16.0; the removed definition had boot value `0` | The bug needed a non-zero value larger than the current 64-bit XID. History is the only evidence, because no code remains to cite. |
| `LC_CTYPE = C` with another `LC_COLLATE` (17.1) | `initdb` fills an unset `LC_CTYPE` and `LC_COLLATE` from the same `--locale` value ([initdb.c:2412-2417](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L2412-L2417)) | The fixed test reads `LC_COLLATE` for the libc provider ([pg_locale.c:1378-1382](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1378-L1382)); the old one read `LC_CTYPE`, which mattered only when the two differ. |
| ecpg `-C ORACLE` (11.20) | Not the default compatibility mode | The preprocessor sets `ECPG_COMPAT_ORACLE` only for that option ([ecpg.c:189-192](../../../../raw/postgres-17/src/interfaces/ecpg/preproc/ecpg.c#L189-L192)). |
| Contrib extension installed (`btree_gist`, `ltree`, `hstore`, `intarray`) | No extension is installed by default | The faulty opclass or input function runs only after `CREATE EXTENSION`; the row's evidence links that module's code. |
| Extension-only code paths (11.21 init fork, 17.3 bulk write) | Core never takes these paths | The notes say no core index access method creates an empty init fork, and core never starts a bulk write on a fork that holds data. |

### Oldest back-patch in the 9.5 branch

12 fixes, first released in 9.5.22, 9.5.23, 9.5.24, 9.5.25.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **9.5.22** (`3f77a6702`) | Metadata | `ALTER TABLE` that rebuilt an index forgot which index was marked for `CLUSTER` (`indisclustered`). | 12.3 | `a40caf5f8`; in 17.0 | [tablecmds.c:13719-13727](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L13719-L13727), [lsyscache.c:3714-3725](../../../../raw/postgres-17/src/backend/utils/cache/lsyscache.c#L3714-L3725) |
| **9.5.22** (`9eff11653`, `a4726998d`) | WAL archive | Crash recovery could recycle WAL segments that were ready for archiving but not yet archived, so the archive never received them. Needs WAL archiving ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.3 | `4e87c4836`, `f9c1b8dba`; in 17.0 | [xlog.c:6348-6353](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L6348-L6353), [xlogarchive.c:571-579](../../../../raw/postgres-17/src/backend/access/transam/xlogarchive.c#L571-L579) |
| **9.5.23** (`db820b9a9`) | Dump/restore | Parallel `pg_restore` could apply per-column privileges before the table privileges, so some column grants disappeared. A dump taken with the fixed `pg_dump` is needed. | 12.4 | `ea9125304`; in 17.0 | [pg_dump.c:15337-15343](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L15337-L15343) |
| **9.5.23** (`09dc17486`) | Control file | Some code paths changed `pg_control` without the correct lock, so it could be written with a bad CRC and prevent a restart after a crash. | 12.4 | `57cb80630`; in 17.0 | History only |
| **9.5.23** (`595b4115c`) | Logical replication | `pg_recvlogical` could exit before its final slot-position update, so the server re-sent data on the next connection. Needs logical decoding ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.4 | `cee9cadb59` (matched by subject); in 17.0 | History only |
| **9.5.24** (`3fbbf64ce`, `d8a22a387`) | SLRU | Concurrent truncations of SLRU logs could remove needed data, which later showed up as apparent wraparound or `could not access status of transaction` errors. | 12.5 | `566372b3d`; in 17.0 | [lmgr.c:487-491](../../../../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L487-L491) |
| **9.5.24** (`23d8b3577`) | Durability | Checkpoints did not fsync the SLRU directories, so an operating-system crash could lose SLRU files. | 12.5 | `aca74843e`; in 17.0 | History only |
| **9.5.24** (`d2d3a4bd3`) | Index (BRIN) | BRIN stored out-of-line TOAST pointers in index entries, which later failed with `missing chunk number 0`. `REINDEX` repairs an affected index. | 12.5 | `7577dd848`; in 17.0 | [brin_tuple.c:208-222](../../../../raw/postgres-17/src/backend/access/brin/brin_tuple.c#L208-L222) |
| **9.5.24** (`40a8fb1e0`) | WAL archive | A standby with `archive_mode = always` did not archive timeline history files, so a later point-in-time recovery could fail ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.5 | `fd26f7823`; in 17.0 | [walreceiver.c:864-866](../../../../raw/postgres-17/src/backend/replication/walreceiver.c#L864-L866) |
| **9.5.25** (`c424c75f5`) | SLRU | The SLRU wraparound point was not rounded to a page boundary correctly, so a cluster within one page of SLRU overflow could lose data. | 12.6 | `6db992833`; in 17.0 | [slru.c:1605-1657](../../../../raw/postgres-17/src/backend/access/transam/slru.c#L1605-L1657), [predicate.c:736-777](../../../../raw/postgres-17/src/backend/storage/lmgr/predicate.c#L736-L777) |
| **9.5.25** (`77a0e14f0`) | Index (CIC) | `CREATE INDEX CONCURRENTLY` did not wait for prepared transactions, so their rows could be missing from the new index. Needs prepared transactions ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.6 | `8a54e12a3`; in 17.0 | [lock.c:2934-2940](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L2934-L2940) |
| **9.5.25** (`0ab177bec`) | Wrong rows | `WHERE CURRENT OF` on a cursor whose plan contained a MergeAppend could silently give false matches. Support for that plan shape was removed. | 12.6 | `a0efda88a`; in 17.0 | History only |

### Oldest back-patch in the 9.6 branch

9 fixes, first released in 9.6.22, 9.6.23, 9.6.24.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **9.6.22** (`0c1caa48d`) | Table data | An array lower bound plus length could overflow an integer, which confused later assignments and could overwrite memory and modify data unexpectedly (CVE-2021-32027). | 12.7 | `f02b9085a`; in 17.0 | [arrayutils.c:135-140](../../../../raw/postgres-17/src/backend/utils/adt/arrayutils.c#L135-L140), [arrayfuncs.c:1352](../../../../raw/postgres-17/src/backend/utils/adt/arrayfuncs.c#L1352) |
| **9.6.23** (`734be249d`) | Catalog | `ALTER EXTENSION ADD/DROP` could run concurrently with `DROP EXTENSION` and leave corrupt catalog entries. | 12.8 | `626731db2`; in 17.0 | [extension.c:3456-3458](../../../../raw/postgres-17/src/backend/commands/extension.c#L3456-L3458) |
| **9.6.23** (`32d182dd0`) | Recovery | Replay of an abort record that truncated files did not advance the minimum recovery point, so recovery could stop at an earlier, inconsistent point. | 12.8 | `a00c138b7`; in 17.0 | History only |
| **9.6.24** (`148c6ee3b`, `5ec87619b`, `99197701e`) | WAL / standby | After a crash with a partial WAL record at a segment end, the primary overwrote WAL that standbys already had, leaving them unable to continue. The fix writes an `OVERWRITE_CONTRECORD` record instead. | 12.9 | `010e52337`, `d03bca4d7`, `d186d233d` +1; in 17.0 | [xlogdesc.c:154-161](../../../../raw/postgres-17/src/backend/access/rmgrdesc/xlogdesc.c#L154-L161), [xlog.c:7490-7493](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L7490-L7493) |
| **9.6.24** (`978998dbd`) | Durability | Crash recovery of `CREATE TABLESPACE` emptied the new directory, losing data that had been written without WAL. Needs `wal_level = minimal` ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.9 | `97ddda8a8`; in 17.0 | [tablespace.c:628-649](../../../../raw/postgres-17/src/backend/commands/tablespace.c#L628-L649) |
| **9.6.24** (`e2b2a9e1c`) | Table data | During standby promotion, a snapshot could miss a prepared transaction, and updates made with that snapshot could corrupt data. Needs prepared transactions ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.9 | `8a4237908`; in 17.0 | [xlog.c:6163-6164](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L6163-L6164) |
| **9.6.24** (`518493243`) | Index (CIC) | `CREATE INDEX CONCURRENTLY` could miss a `PREPARE TRANSACTION` still in progress, so its rows could be missing from the index. Needs prepared transactions ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.9 | `3cd9c3b92`; in 17.0 | [lock.c:4509-4544](../../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L4509-L4544), [twophase.c:852-884](../../../../raw/postgres-17/src/backend/access/transam/twophase.c#L852-L884) |
| **9.6.24** (`e428699cb`) | Index (CIC) | A race let backends skip adding new rows to an index being built or rebuilt `CONCURRENTLY`. Reindexing such indexes is recommended. | 12.9 | `fdd965d07`; in 17.0 | [relcache.c:1079-1091](../../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L1079-L1091), [inval.c:733-739](../../../../raw/postgres-17/src/backend/utils/cache/inval.c#L733-L739) |
| **9.6.24** (`a5e8f7b37`, `dea212e24`) | Index (hash) | `float4`/`float8` hash functions gave different codes for different NaN bit patterns, so hash indexes could return wrong results. Reindexing hash indexes on float columns is advised. | 12.9 | `b30cc0fd6`, `ce773f230`, `fd549145d`; in 17.0 | History only |

### Oldest back-patch in the 10 branch

9 fixes, first released in 10.20, 10.21, 10.22, 10.23.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **10.20** (`f862cc09f`) | Durability | Checkpoints did not fsync `pg_logical/mappings`, so a system crash could lose logical rewrite files. | 12.10 | `1fabec7d7`; in 17.0 | History only |
| **10.20** (`919be95c6`) | SLRU | The commit timestamp of a transaction's last subtransaction could be lost. Needs `track_commit_timestamp` ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.10 | `237d1f317`; in 17.0 | History only |
| **10.20** (`b21986908`) | Index (contrib) | `btree_gist` stored `char(N)` values without padding, so index-only scans returned space-stripped values. Needs the extension; `REINDEX` applies the fix at once. | 12.10 | `54b1cb7eb`; in 17.0 | [btree_text.c:96-149](../../../../raw/postgres-17/contrib/btree_gist/btree_text.c#L96-L149) |
| **10.21** (`409bcfda6`) | Metadata | `ALTER FUNCTION` that changed both `PARALLEL` and a `SET` clause lost the `PARALLEL` change. | 12.11 | `344a225cb`; in 17.0 | History only |
| **10.21** (`48b398943`, `57f618310`) | Recovery | A `TRUNCATE` that overlapped a checkpoint could let replay find data in pages that should have been removed. | 12.11 | `412ad7a55`; in 17.0 | [xlog.c:7168-7185](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L7168-L7185) |
| **10.21** (`b5e060473`) | Logical replication | Unchanged replica identity key columns stored out of line were left out of the update's WAL, so subscribers could not replicate the update ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.11 | `5e01001ff`; in 17.0 | [heapam.c:4673-4695](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4673-L4695) |
| **10.22** (`60e956eb8`) | Control file | A restartpoint that finished after promotion could write wrong last-checkpoint fields into `pg_control`, causing a PANIC and failed restart after a crash. | 12.12 | `7863ee4def` (matched by subject); in 17.0 | [xlog.c:7757-7769](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L7757-L7769) |
| **10.23** (`0b3ff539d`) | WAL | Promoting an archive-recovery (non-standby) instance whose last WAL segment ended in a partial record wrote an invalid WAL segment on the new timeline. | 12.13 | `6672d7913`; in 17.0 | History only |
| **10.23** (`174c929e3`, `42d0d46f9`, `e1ea6f374`) | Wrong rows | `UPDATE ... SET (cols) = (SELECT ...)` on an inherited or partitioned table could make incorrect data updates. No same-subject commit exists on the v17 line. | 12.13 | — | No v17-line commit. |

### Oldest back-patch in the 11 branch

12 fixes, first released in 11.8, 11.19, 11.20, 11.21.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **11.8** (`8f9aba187`) | Checksum tool | Offline `pg_checksums` processed other major versions' tablespace subdirectories at a shared location; in enable mode processing means writing checksums into pages. | 12.3 | `428a2609e`; in 17.0 | [pg_checksums.c:391-424](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L391-L424), [pg_checksums.c:237-258](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L237-L258) |
| **11.19** (`d95dcc9ab`) | Temporary data | Parallel hash join corrupted a large tuple's final chunk when it was exactly 32,760 bytes; the query typically failed later. | 12.14 | `ffcf6f4cf`; in 17.0 | History only |
| **11.19** (`5eaf3e375`) | WAL / standby | Replay that set a heap page's all-visible bit did not advance its LSN; the release note calls this a theoretical hazard. Applies with checksums or `wal_log_hints` ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.14 | `d6a3dbe14`; in 17.0 | [heapam.c:9836-9837](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L9836-L9837) |
| **11.20** (`9eb44bb04`, `dbd25dd0b`) | Client (ecpg) | ecpg programs built with `-C ORACLE` could write a terminating zero into the preceding field when `varcharsize` was 0, truncating that field. | 12.15 | `f18029084`; in 17.0 | [data.c:648-666](../../../../raw/postgres-17/src/interfaces/ecpg/ecpglib/data.c#L648-L666) |
| **11.20** (`012ffb365`) | Dump/restore | `pg_restore` of tables hash-partitioned on an enum could not route rows, and in `--load-via-partition-root` mode it could `TRUNCATE` targets with a hazard of lost data. | 12.15 | `bc8cd50fe`; in 17.0 | [pg_dump.c:2676-2699](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L2676-L2699), [pg_backup_archiver.c:1170-1188](../../../../raw/postgres-17/src/bin/pg_dump/pg_backup_archiver.c#L1170-L1188) |
| **11.20** (`ffec64ba8`) | Wrong rows | `INSERT ... ON CONFLICT DO UPDATE SET (cols) = (SELECT ...)` on a partitioned table could make incorrect data updates. | 12.15 | `87f3667ec`; in 17.0 | [execExpr.c:2799-2818](../../../../raw/postgres-17/src/backend/executor/execExpr.c#L2799-L2818), [nodeSubplan.c:351-359](../../../../raw/postgres-17/src/backend/executor/nodeSubplan.c#L351-L359) |
| **11.21** (`1c38e7ae1`) | Catalog | An interrupted `DROP DATABASE` could leave an accessible but corrupt database. The database is now marked invalid before irreversible steps. | 12.16 | `c66a7d75e`; in 17.0 | [dbcommands.c:3198-3221](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L3198-L3221), [postinit.c:1108-1113](../../../../raw/postgres-17/src/backend/utils/init/postinit.c#L1108-L1113) |
| **11.21** (`6377f705c`) | Durability | A newly created, still-empty table file was not fsynced at the next checkpoint, so an operating-system crash could lose the file. | 12.16 | `4b4798e13`; in 17.0 | [md.c:241-242](../../../../raw/postgres-17/src/backend/storage/smgr/md.c#L241-L242) |
| **11.21** (`b511d7323`, `fc7dc728d`) | Index (BRIN) | BRIN confused empty ranges with all-NULL ranges and merged all-NULL summaries wrongly, so searches for NULLs could miss rows. Reindexing such BRIN indexes is recommended. | 12.16 | `3ec8a3bfb5`, `3581cbdcd6` (matched by subject); in 17.0 | [brin.c:2063-2087](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2063-L2087), [brin_tuple.c:375-376](../../../../raw/postgres-17/src/backend/access/brin/brin_tuple.c#L375-L376) |
| **11.21** (`988719b88`) | Crash safety | Creating an empty init fork for an unlogged index was not WAL-logged. No core index access method creates an empty init fork; an extension might. | 12.16 | `3142a8845`; in 17.0 | History only |
| **11.21** (`bbfc26d86`) | Stored values (contrib) | `hstore` input treated some characters as whitespace and discarded them. Needs the extension. | 12.16 | `d522b05c8c` (matched by subject); in 17.0 | History only |
| **11.21** (`ab40b0395`) | Index (contrib) | `intarray`'s `gist__int_ops` accepted oversize arrays and created invalid index entries. Needs the extension. | 12.16 | `c00fbe89dc` (matched by subject); in 17.0 | [_int_gist.c:186-187](../../../../raw/postgres-17/contrib/intarray/_int_gist.c#L186-L187) |

### Oldest back-patch in the 12 branch

14 fixes, first released in 12.3, 12.4, 12.6, 12.7, 12.10, 12.15, 12.17, 12.18, 12.20, 12.21.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **12.3** (`a375f11c4`) | Table data | A `GENERATED` column whose value was an exact copy of a pass-by-reference column could store corrupted data. | 12.3 | `3cb02e307`; in 17.0 | History only |
| **12.3** (`f087d63a4`) | Metadata | `REINDEX CONCURRENTLY` lost `indisclustered`. | 12.3 | `d79fb88ac`; in 17.0 | History only |
| **12.3** (`26876127b`) | Catalog | A failed `REINDEX CONCURRENTLY` could leave the old index with no `pg_depend` links. | 12.3 | `fbcf08711`; in 17.0 | [index.c:1805-1806](../../../../raw/postgres-17/src/backend/catalog/index.c#L1805-L1806) |
| **12.4** (`75f147924`) | Metadata | `REINDEX CONCURRENTLY` lost the replica identity flag, so logical decoding stopped including old tuple values. | 12.4 | `1127f0e392` (matched by subject); in 17.0 | [index.c:1650-1662](../../../../raw/postgres-17/src/backend/catalog/index.c#L1650-L1662) |
| **12.6** (`0326635dd`) | Index (GiST) | GiST misdetected concurrent page splits, so concurrent inserts could place entries on the wrong pages. Reindexing affected GiST indexes is recommended. | 12.6 | `6b4d3046f`; in 17.0 | History only |
| **12.7** (`5b2945ec0`, `85edb1f26`) | Metadata | `REINDEX CONCURRENTLY` lost a per-column statistics target set on the index. | 12.7 | `4f4726005`, `bd1208098`; in 17.0 | History only |
| **12.10** (`5ed74d874`) | Index (TOAST) | TOAST updates released their table lock early, so `REINDEX CONCURRENTLY` on a TOAST table or index tended to produce a corrupted index. | 12.10 | `f99870dd8`; in 17.0 | History only |
| **12.15** (`3c92f7e9d`) | Table data | A `vacuum_defer_cleanup_age` larger than the current 64-bit XID made VACUUM remove live rows (14+) or recycle GiST pages early (12-13). The setting no longer exists at the v17 pin ([condition](#settings-and-conditions-that-gate-some-fixes)). | 12.15 | `be504a3e9`; in 17.0 | Removed by `1118cd37eb`, first in 16.0. |
| **12.17** (`26917ebea`) | Index (GiST) | A recursive page split during a GiST build tracked a downlink wrongly, so later searches could give incorrect answers. Reindexing GiST indexes may be advisable. | 12.17 | `28d3c2ddc`; in 17.0 | [gist.c:1069-1136](../../../../raw/postgres-17/src/backend/access/gist/gist.c#L1069-L1136) |
| **12.17** (`22b2e6e9d`) | Recovery | An out-of-memory failure while reading WAL was treated as end of WAL, which could make WAL replay inconsistent. | 12.17 | `6b18b3fe2`; in 17.0 | History only |
| **12.18** (`e6511fe64`) | Index (GIN) | Cleanup of an incomplete GIN internal-page split took a shared instead of an exclusive buffer lock, so two concurrent cleanups could corrupt the index. | 12.18 | `6a1ea02c4`; in 17.0 | [ginbtree.c:779-800](../../../../raw/postgres-17/src/backend/access/gin/ginbtree.c#L779-L800) |
| **12.20** (`067cb6c5d`, `d0054432d`) | Catalog | `DETACH PARTITION` with mismatched index-based constraints left a wrong `coninhcount`. | 12.20 | `0340eefd9`, `30ca4e1ab`; in 17.0 | [tablecmds.c:19589-19595](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19589-L19595) |
| **12.21** (`5a33a39a8`, `7354b680a`, `766809db3` +4) | Catalog / index | Races could lose an in-place catalog update; for example a lost `relhasindex = true` stopped updates of a new index and so corrupted it. | 12.21 | `0bcb9d079`, `3b7a689e1`, `54bc22fbf` +4; in 17.1 | [heapam.c:4450-4562](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4450-L4562), [genam.c:795-857](../../../../raw/postgres-17/src/backend/access/index/genam.c#L795-L857), [release-17.sgml (17.1)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L13471-L13547) |
| **12.21** (`da9950456`) | Catalog | Catalog caches were not reset at the end of recovery, so stale cache data could make an in-place catalog update get lost. | 12.21 | `a4668c99f`; in 17.1 | [sinvaladt.c:713-722](../../../../raw/postgres-17/src/backend/storage/ipc/sinvaladt.c#L713-L722), [release-17.sgml (17.1)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L13549-L13568) |

### Oldest back-patch in the 13 branch

16 fixes, first released in 13.1, 13.5, 13.7, 13.13, 13.17, 13.19, 13.21, 13.22, 13.23.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **13.1** (`2330f4d3a`, `7957e75c5`) | Read path | TOAST (pglz) decompression could loop forever or produce corrupted output data. | — | `dfc797730`, `fd2997565`; in 17.0 | [pg_lzcompress.c:859-886](../../../../raw/postgres-17/src/common/pg_lzcompress.c#L859-L886) |
| **13.5** (`77f7909a4`) | Metadata | `REINDEX CONCURRENTLY` lost operator-class parameters attached to the index. | — | `add5cf28d`; in 17.0 | History only |
| **13.7** (`bad202c61`) | Index (contrib) | `gist_ltree_ops`' default signature length changed by accident, so indexes used before upgrading the extension to 1.2 are very likely corrupt. Needs the extension. | — | `7e74aafc4`; in 17.0 | History only |
| **13.13** (`6fd1dbdb2`) | Index (B-tree) | B-tree deduplication was applied to `interval` columns, whose equal values can differ, giving wrong index-only scan results. Reindexing them is recommended. | — | `5f27b5f84`; in 17.0 | [verify_nbtree.c:388-398](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L388-L398) |
| **13.17** (`d20194cea`) | Integrity | `ATTACH`/`DETACH PARTITION` did not convert foreign-key catalog entries correctly, so a detached table could lack enforcement triggers and accept violating rows. | — | `5914a22f6`, `936ab6de9`; in 17.1 | [tablecmds.c:11224-11291](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L11224-L11291), [release-17.sgml (17.1)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L13004-L13076) |
| **13.19** (`8bfd2ceba`) | Catalog | Vacuuming a system catalog concurrently with an update could corrupt the catalog. | — | `1587f7b9f`, `f4af4515b`; in 17.3 | [heapam.c:3490-3498](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L3490-L3498), [xid8funcs.c:143-146](../../../../raw/postgres-17/src/backend/utils/adt/xid8funcs.c#L143-L146), [release-17.sgml (17.3)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L11209-L11227) |
| **13.19** (`228091216`, `417d41c65`, `a1d17a894`) | Table data | A failed filesystem truncation could leave inconsistent files, for example reviving deleted data. Such failures now PANIC so WAL replay restores consistency. | — | `0350b876b`, `45aef9f6b`, `66aaabe7a`; in 17.3 | [pg_visibility.c:383-404](../../../../raw/postgres-17/contrib/pg_visibility/pg_visibility.c#L383-L404), [smgr.c:701-707](../../../../raw/postgres-17/src/backend/storage/smgr/smgr.c#L701-L707), [release-17.sgml (17.3)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L11229-L11265) |
| **13.19** (`1168acbca`) | Durability | A checkpoint could start during a relation truncation, so the modified file might not be fsynced before the checkpoint completed. | — | `d4ffbf47b`; in 17.3 | [storage.c:369-370](../../../../raw/postgres-17/src/backend/catalog/storage.c#L369-L370), [release-17.sgml (17.3)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L11267-L11288) |
| **13.19** (`26a79cbbd`) | Durability | File renames used `link()`/`unlink()`, and a failure could leave two links to one file, risking corruption. | — | `dac1ff3090` (matched by subject); in 17.0 | [xlog.c:3577-3578](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L3577-L3578) |
| **13.19** (`102546322`) | Catalog | An update of `pg_database.datfrozenxid` could be lost when VACUUM ran concurrently with a `REASSIGN OWNED` of that database. | — | `fa6131377`; in 17.3 | [objectaddress.c:2784-2810](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L2784-L2810), [release-17.sgml (17.3)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L11290-L11308) |
| **13.21** (`20e5ef3ca`, `69a498eb6`, `a5b0c06da` +4) | WAL | WAL recycling and preallocation during archive recovery could corrupt WAL files restored from the archive. | — | `cc2c7d65fc`, `85656bc305`, `2b3e4672f7` +3 (matched by subject); in 17.0 | [xlog.c:9545-9556](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L9545-L9556) |
| **13.21** (`247ee9415`, `4164d6976`) | Logical replication | DDL that does not take a strong lock was not reflected in WAL decoding, which then decoded later changes with stale catalog data ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `cadaf0ac4`, `d96206f25`; in 17.5 | [snapbuild.c:943-953](../../../../raw/postgres-17/src/backend/replication/logical/snapbuild.c#L943-L953), [reorderbuffer.c:5494-5507](../../../../raw/postgres-17/src/backend/replication/logical/reorderbuffer.c#L5494-L5507), [release-17.sgml (17.5)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L10308-L10338) |
| **13.21** (`b3a9c536d`) | Integrity | Self-referential foreign keys on partitioned tables lacked catalog entries for new or attached partitions, so the constraint was not fully enforced. | — | `f51ae3187`; in 17.5 | [tablecmds.c:10628-10633](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L10628-L10633), [release-17.sgml (17.5)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L9560-L9591) |
| **13.21** (`d65485b02`) | Logical replication | Fast-forward logical decoding advanced the catalog xmin too far, so catalog rows it still needed could be vacuumed away ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `36148b22e`; in 17.5 | [release-17.sgml (17.5)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L10285-L10306) |
| **13.22** (`dd3df0b85`) | WAL | A slot restart point advanced during a checkpoint could let WAL be removed too soon, so a crash right afterwards could make recovery fail. | — | `2090edc6f`; in 17.6 | [xlog.c:670-671](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L670-L671), [release-17.sgml (17.6)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L8418-L8439) |
| **13.23** (`0610b843c`, `dbef9cbaa`) | Logical replication | `MERGE` and `INSERT ... ON CONFLICT DO UPDATE` did not require a replica identity on published tables, which can silently break replication ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `0b934d399`, `57dfb64ec`, `76f45be93`; in 17.7 | [execMain.c:1041-1045](../../../../raw/postgres-17/src/backend/executor/execMain.c#L1041-L1045), [execPartition.c:360-364](../../../../raw/postgres-17/src/backend/executor/execPartition.c#L360-L364), [release-17.sgml (17.7)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L6809-L6849) |

### Oldest back-patch in the 14 branch

17 fixes, first released in 14.1, 14.2, 14.3, 14.4, 14.7, 14.9, 14.11, 14.15, 14.18, 14.21, 14.23, 14.24.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **14.1** (`61a86ed55`) | Index | Parallel VACUUM skipped indexes below `min_parallel_index_scan_size` when the table had at least two larger ones, leaving index entries for removed rows. Autovacuum was not affected. | — | `9bacec15b`; in 17.0 | History only |
| **14.2** (`dad1539ae`) | Index / heap | Pruning could remove a recently-dead tuple but leave a redirect to it, so a later tuple in that slot joined the old HOT chain. | — | `18b87b201`; in 17.0 | [pruneheap.c:492-494](../../../../raw/postgres-17/src/backend/access/heap/pruneheap.c#L492-L494) |
| **14.3** (`7d80e93fb`) | Durability | Sorted GiST builds did not fsync the new index, so an operating-system crash shortly after could corrupt it. | — | `6c46e8a5d`; in 17.0 | [gistbuild.c:409](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L409), [gistbuild.c:453](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L453), [bulk_write.c:131-223](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L223), Since `8af2565248` the sorted build writes through the bulk writer, whose finish step fsyncs or registers the sync. |
| **14.4** (`042b584c7`) | Index (CIC) | A v14 optimization made `CREATE INDEX CONCURRENTLY` and `REINDEX CONCURRENTLY` sometimes miss rows updated during the build. The optimization was reverted; rebuilding such indexes is recommended. | — | `e28bb8851`; in 17.0 | [procarray.c:1862-1863](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1862-L1863) |
| **14.7** (`a18328bb3`) | Stored values | Assigning through a `jsonb` subscript taken from a toasted text value could store the value under a garbage key. | — | `b0feda79f`; in 17.0 | [jsonfuncs.c:1579-1583](../../../../raw/postgres-17/src/backend/utils/adt/jsonfuncs.c#L1579-L1583), [jsonb_gin.c:899-900](../../../../raw/postgres-17/src/backend/utils/adt/jsonb_gin.c#L899-L900) |
| **14.9** (`f8320cc72`) | Wrong rows | EvalPlanQual rechecks on inherited or partitioned targets could perform `UPDATE`/`DELETE` actions that a concurrent update should have skipped (READ COMMITTED). | — | `70b42f2790` (matched by subject); in 17.0 | [execMain.c:2991-3003](../../../../raw/postgres-17/src/backend/executor/execMain.c#L2991-L3003), [execScan.c:325-326](../../../../raw/postgres-17/src/backend/executor/execScan.c#L325-L326) |
| **14.11** (`c7edaeec5`) | Table data | A `BEFORE ROW UPDATE` trigger could read a new tuple version from an unpinned buffer, so garbage could appear in non-updated columns. | — | `6545ba96c`; in 17.0 | [trigger.c:3050-3062](../../../../raw/postgres-17/src/backend/commands/trigger.c#L3050-L3062) |
| **14.15** (`ba2535841`) | WAL (tool) | `pg_rewind` could delete WAL files that the rewound node still needed for recovery. | — | `cb844d66b`; in 17.2 | [parsexlog.c:223-238](../../../../raw/postgres-17/src/bin/pg_rewind/parsexlog.c#L223-L238), [filemap.c:245-268](../../../../raw/postgres-17/src/bin/pg_rewind/filemap.c#L245-L268), [release-17.sgml (17.2)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L12667-L12695) |
| **14.18** (`6be02bbc8`) | Index (BRIN) | `brin_bloom_union()` dropped data when merging compressed summaries, so index searches could miss rows; v17 parallel BRIN builds use this path. | — | `cb0ad70b8`; in 17.5 | [brin_bloom.c:697-704](../../../../raw/postgres-17/src/backend/access/brin/brin_bloom.c#L697-L704), [release-17.sgml (17.5)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L9593-L9621) |
| **14.21** (`7cb05dd2d`) | Notifications | An error while absorbing a `NOTIFY` message skipped that notification silently; such errors now end the session. | — | `b821c9292`; in 17.8 | [async.c:1930-1933](../../../../raw/postgres-17/src/backend/commands/async.c#L1930-L1933), [release-17.sgml (17.8)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L5118-L5155) |
| **14.21** (`14161bca4`) | Recovery | Recovery started even when WAL back to the checkpoint's redo point was missing, and could crash or corrupt the database. It now fails up front. | — | `f5927da4f`; in 17.8 | [xlogrecovery.c:802-808](../../../../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L802-L808), [release-17.sgml (17.8)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L5442-L5463) |
| **14.21** (`0dfbd191a`, `811422471`, `e4b1986b9`) | Catalog / SLRU | In-place updates were not invalidated non-transactionally or WAL-logged before becoming visible, so frozen-XID fields could become inconsistent and allow premature CLOG truncation. | — | `0f69bedde`, `bcb784e7d`, `d3e5d8950`; in 17.8 | [heapam.c:6840-6894](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6840-L6894), [bufmgr.c:3699-3735](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3699-L3735), [release-17.sgml (17.8)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L5204-L5236) |
| **14.21** (`46be454ca`) | SLRU | MultiXact log truncation after a crash could be incorrect. | — | `d3ad4cef6`; in 17.8 | [multixact.c:3386-3389](../../../../raw/postgres-17/src/backend/access/transam/multixact.c#L3386-L3389), [release-17.sgml (17.8)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L5279-L5294) |
| **14.23** (`f5d1038d9`) | Free space map | FSM changes made during replay were not marked dirty with checksums or `wal_log_hints` on, so they could be lost. The map is a hint, so the effect was degraded performance ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `1cf010f21`; in 17.10 | [freespace.c:244-245](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L244-L245), [release-17.sgml (17.10)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L4277-L4304) |
| **14.24** (`e4ad22eb0`) | Read path | GiST and SP-GiST index-only scans could mis-decode index tuples and emit corrupted data; in core only `range_ops` as a non-first column was affected. | — | `355faed5a`; in 17.11 | [nodeIndexonlyscan.c:256-316](../../../../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L256-L316), [release-17.sgml (17.11)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L1746-L1769) |
| **14.24** (`f528a5606`) | Index (contrib) | `ltree` comparisons overflowed for values with more than about 14,653 labels, so B-tree indexes on such values are probably corrupt. Needs the extension. | — | `c391c00d9`; in 17.11 | [ltree_op.c:97-113](../../../../raw/postgres-17/contrib/ltree/ltree_op.c#L97-L113), [ltree_gist.c:270-271](../../../../raw/postgres-17/contrib/ltree/ltree_gist.c#L270-L271), [release-17.sgml (17.11)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L2754-L2776) |
| **14.24** (`255bce448`) | Index (contrib) | `btree_gist` float opclasses mishandled NaN, so indexes containing NaN may need reindexing. Needs the extension. | — | `d215d2cc2`; in 17.11 | [btree_float4.c:37-87](../../../../raw/postgres-17/contrib/btree_gist/btree_float4.c#L37-L87), [btree_float8.c:38-71](../../../../raw/postgres-17/contrib/btree_gist/btree_float8.c#L38-L71), [release-17.sgml (17.11)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L2646-L2672) |

### Oldest back-patch in the 15 branch

10 fixes, first released in 15.2, 15.3, 15.5, 15.6, 15.8, 15.14, 15.15, 15.18, 15.19.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **15.2** (`8b5ba2f3f`) | Backup | `pg_basebackup` could corrupt very large tablespace map files. | — | `45f5c81ad`; in 17.0 | History only |
| **15.3** (`fa5dd460c`) | Catalog | `CREATE DATABASE ... STRATEGY WAL_LOG`, the v17 default strategy, handled the template's buffers wrongly, so later changes to the template's `pg_class` could be lost. | — | `8a8661828`; in 17.0 | History only |
| **15.5** (`3c1a1af91`) | Wrong rows | `MERGE` rechecks of concurrently updated rows could use the wrong rows of joined tables and make incorrect update decisions (READ COMMITTED). | — | `1d5caec22`; in 17.0 | [planner.c:2321-2322](../../../../raw/postgres-17/src/backend/optimizer/plan/planner.c#L2321-L2322) |
| **15.6** (`8fa4a1ac6`, `d493bed28`) | Durability | `CREATE DATABASE` was not fully durable: after an OS crash the database directory, `PG_VERSION` or `pg_filenode.map` could be missing or empty, also in a base backup taken then. | — | `0b6517a3b`, `df220714e`; in 17.0 | [dbcommands.c:524-535](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L524-L535) |
| **15.6** (`a77fb8c68`) | Logical replication | `pgoutput` sent NULL instead of the fast default for columns added with a constant default, so subscribers stored wrong values ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `360392fa2`; in 17.0 | [pgoutput.c:1185-1186](../../../../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L1185-L1186) |
| **15.8** (`24561b498`) | Sequences | `ALTER TABLE SET LOGGED/UNLOGGED` did not lock owned sequences, so concurrent `nextval()` effects could be lost. | — | `f88cdb36c4` (matched by subject); in 17.0 | History only |
| **15.14** (`d44efe87e`) | Backup | The LZ4 archive streamer used by `pg_basebackup` and `pg_verifybackup` could decompress poorly compressible data incorrectly. | — | `074003431`; in 17.6 | [release-17.sgml (17.6)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L8328-L8347) |
| **15.15** (`f871fbae9`) | Wrong rows | `MERGE ... UPDATE` with more than one concurrent update of the target row could pick the wrong latest tuple version. | — | `6195afbe5`; in 17.7 | [nodeModifyTable.c:3273-3275](../../../../raw/postgres-17/src/backend/executor/nodeModifyTable.c#L3273-L3275), [release-17.sgml (17.7)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L6786-L6807) |
| **15.18** (`4548e8746`, `9a42888a3`, `d3bb7841b`) | Backup | Backup decompression and tar parsing in `pg_basebackup` and `pg_verifybackup` could corrupt LZ4-compressed data in edge cases, among other bugs. | — | `2640c5ba7`, `8b198b093`, `f1298a4c2`; in 17.10 | [bbstreamer_gzip.c:295-297](../../../../raw/postgres-17/src/bin/pg_basebackup/bbstreamer_gzip.c#L295-L297), [bbstreamer_lz4.c:362-363](../../../../raw/postgres-17/src/bin/pg_basebackup/bbstreamer_lz4.c#L362-L363), [release-17.sgml (17.10)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L4323-L4361) |
| **15.19** (`d2980067b`) | Sequences | An unlogged sequence replicated to a standby was corrupt after promotion (`bad magic number in sequence`). | — | `a1ed6a9a0`; in 17.11 | [xlogutils.c:345-354](../../../../raw/postgres-17/src/backend/access/transam/xlogutils.c#L345-L354), [release-17.sgml (17.11)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L2120-L2142) |

### Oldest back-patch in the 16 branch

5 fixes, first released in 16.5, 16.9, 16.12, 16.14.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **16.5** (`b39c5272c`) | Logical replication | An apply worker could advance replication origin progress on error or shutdown, so the publisher would not resend that transaction ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `915aafe82a` (matched by subject); in 17.0 | [worker.c:4707-4718](../../../../raw/postgres-17/src/backend/replication/logical/worker.c#L4707-L4718) |
| **16.9** (`0af3ae468`, `514d47dfb`) | Dump/restore | Views and rules using `XMLSERIALIZE(... INDENT)` were dumped without `INDENT`, so they behaved differently after restore. | — | `2e0f93d7c`, `310907aaf`; in 17.5 | [ruleutils.c:9918-9920](../../../../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L9918-L9920), [release-17.sgml (17.5)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L9873-L9896) |
| **16.9** (`0de091a4b`) | Logical replication | An apply error that was caught without worker exit reset the replication origin wrongly, which could apply duplicate data ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `05676d87e`; in 17.5 | [release-17.sgml (17.5)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L10340-L10357) |
| **16.12** (`63a65adf4`) | Logical replication | After a parallel apply worker failed, progress could still advance, so the subscriber lost transactions. Needs `streaming = parallel` ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `0ed8f1afb`; in 17.8 | [release-17.sgml (17.8)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L5593-L5611) |
| **16.14** (`a4649c50a`) | Dump/restore | `pg_dumpall` skipped role grants whose grantor OID dangled, so those grants were lost. | — | `1cd783d20`; in 17.10 | [pg_dumpall.c:1141-1176](../../../../raw/postgres-17/src/bin/pg_dump/pg_dumpall.c#L1141-L1176), [release-17.sgml (17.10)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L4363-L4386) |

### Oldest back-patch in the 17 branch

5 fixes, first released in 17.1, 17.3, 17.6, 17.8, 17.11.

| Oldest release (backport) | Area | Fix and risk | First 12.x | V17-line commit; first 17.x | Evidence at the v17 pin |
|---|---|---|---|---|---|
| **17.1** (`8148e7124`) | Index | The C-locale test checked `LC_CTYPE` instead of `LC_COLLATE`, so with `LC_CTYPE = C` and another `LC_COLLATE`, indexes on strings could be corrupted ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `8148e7124`; in 17.1 | [pg_locale.c:1380-1382](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1380-L1382), [release-17.sgml (17.1)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L13078-L13107) |
| **17.3** (`969583553`) | Extension storage | Starting a bulk write on a relation fork that already held data overwrote it with zeros. Core never does that; some extensions might. | — | `969583553`; in 17.3 | [bulk_write.c:285-305](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L285-L305), [release-17.sgml (17.3)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L12030-L12047) |
| **17.6** (`967309116`) | Logical replication | `pg_createsubscriber` could send the last transaction applied during subscriber recovery again ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `967309116`; in 17.6 | [release-17.sgml (17.6)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L8908-L8925) |
| **17.8** (`ad569b54a`) | Backup | After VACUUM truncated a table larger than one segment between a full and an incremental backup, `pg_combinebackup` failed, so the backup could not be restored. Needs WAL summarization ([condition](#settings-and-conditions-that-gate-some-fixes)). | — | `ad569b54a`; in 17.8 | [release-17.sgml (17.8)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L5238-L5259) |
| **17.11** (`067213430`, `7edec8b57`, `c0d9864f5`) | Visibility map / backup | Heap operations that cleared visibility-map bits did not register the map buffer in WAL, so the WAL summarizer missed them and map full-page images were missing; torn map pages could give wrong index-only scan results. | — | `067213430`, `7edec8b57`, `c0d9864f5`; in 17.11 | [heapam.c:4180-4240](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L4180-L4240), [heap_surgery.c:331-349](../../../../raw/postgres-17/contrib/pg_surgery/heap_surgery.c#L331-L349), [release-17.sgml (17.11)](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L1937-L1968) |

### What was left out

Many candidates mention corruption or loss but do not describe damage to stored data. They were left out by class. Each example names the oldest release.

| Class left out | Why | Examples |
|---|---|---|
| Wrong query results with no wrong stored data | The answer was wrong, but nothing was written | Hash aggregation spill returning NULL grouping values (13.2), boolean partition pruning (12.19), hash join rescans with outer Params (12.16), skip-fetch bitmap scans returning dead tuples (13.21) |
| Crashes, PANICs and errors only | The notes describe a failure, not damage | PANIC when VACUUM sets all-visible during an update (12.13), replay failure for unlogged BRIN WAL (14.21), multixid truncation replay (14.22) |
| Hardening against corruption that already exists, or false corruption reports | The bug did not cause the corruption | `amcheck` false positives (11.22), VACUUM continuing past B-tree damage (10.23, 11.21), `pg_controldata` on a bad file (13.19), `pageinspect` and `pgstattuple` hardening |
| In-memory state only | Nothing persistent was affected | Plan cache entries (9.6.23), parse trees (11.20, 14.1), local buffer state (16.2), PL/Python dictionaries (12.20) |
| Statistics and planner estimates | Rebuilt by `ANALYZE` or by the statistics system | Lost `reltuples` update (14.13), index statistics across `REINDEX CONCURRENTLY` (12.5), shared statistics entries (15.10, 15.15) |
| Attacks by a malicious peer | A security fix, not a data-integrity bug in normal operation | `pg_basebackup`/`pg_rewind` path traversal (14.23), `PQfn()` buffer (14.23), `psql \gset` (9.5.24) |
| Consequence not stated as damage | The note does not say data was lost or corrupted | Frozen-XID race in VACUUM (12.19), stale catcache entry while detoasting (12.18), oldest-MultiXact array indexing (17.10), prepared-transaction subtransactions on a hot standby (12.20), locking an unrelated tuple through an update chain (14.21) |
| Availability or disk space, without loss | Replication or recovery stopped with an error, or space leaked | Slot `restart_lsn` moving backwards (13.18), WAL removed after a slot copy (12.3), leaked non-first segments (15.1) |
| Major-release-only fixes | Never listed in a minor-release section | For example the 14.0 GiST checksum fix on [the checksum page](data-checksum-bug-fixes-since-12-2.md) |

### Final summary

A bug fix lands on `master` → it is back-patched to every supported branch where the bug exists → each branch lists it in its next minor-release notes with all branch hashes → the oldest of those branches dates the fix (often a 9.5, 9.6, 10 or 11 release on the same day as a 12.x release) → the v17-line commit carries the fix into PostgreSQL 17, already in 17.0 for 73 fixes and in 17.1-17.11 for 35 → damage an older release already wrote, such as a bad index or a missing foreign-key trigger, stays until the reader runs `REINDEX` or re-adds the constraint, as the notes recommend.

## Context Reviewed

- Release notes: `release-12.sgml` at `REL_12_22`, `release-13.sgml` at `REL_13_23`, `release-14.sgml` at `REL_14_24`, `release-15.sgml` at `REL_15_19`, `release-16.sgml` at `REL_16_15`, all read from the v17 checkout's git objects, and the pinned [release-17.sgml](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L4-L10).
- Release tags `REL9_5_*`, `REL9_6_*`, `REL_10_*` (fetched for this page), `REL_11_*` through `REL_16_*` and `REL_17_*` in the v17 repository; the pinned HEAD was not moved.
- Release process and back-patch tooling: [RELEASE_CHANGES:86-99](../../../../raw/postgres-17/src/tools/RELEASE_CHANGES#L86-L99), [RELEASE_CHANGES:128-148](../../../../raw/postgres-17/src/tools/RELEASE_CHANGES#L128-L148), [git_changelog:8-19](../../../../raw/postgres-17/src/tools/git_changelog#L8-L19), [git_changelog:43-49](../../../../raw/postgres-17/src/tools/git_changelog#L43-L49).
- The diffs of each selected v17-line commit, compared line by line with the pinned files.
- Settings and the code that tests them: `guc_tables.c`, `xlog.h`, `rel.h`, `logical.c`, `twophase.c`, `commit_ts.c`, `initdb.c`, `subscriptioncmds.c`, `worker.c`, `basebackup.c`, `pg_locale.c`, `ecpg.c`.
- The in-place update protocol for the new glossary entry: [heapam.c:6600-6640](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6600-L6640), [genam.c:768-794](../../../../raw/postgres-17/src/backend/access/index/genam.c#L768-L794).
- The related v17 page [Data Checksum Bug Fixes Since PostgreSQL 12.2](data-checksum-bug-fixes-since-12-2.md), for overlap only; it is not evidence.

## Evidence Map

| Claim | Evidence |
|---|---|
| Each branch's minor-release notes are built from that branch's commits | [RELEASE_CHANGES:128-148](../../../../raw/postgres-17/src/tools/RELEASE_CHANGES#L128-L148) |
| A note item's comment lists every branch copy and hash, and copies are merged by message and time | [git_changelog:8-19](../../../../raw/postgres-17/src/tools/git_changelog#L8-L19), [git_changelog:43-49](../../../../raw/postgres-17/src/tools/git_changelog#L43-L49) |
| Oldest release, backport hash and v17-line commit for each row | Release tags and commits in the v17 repository's history, as listed in each row |
| Each fix's code still exists at the pin | The "Evidence at the v17 pin" links: pinned lines identical to lines the fix added |
| Fixes in 17.x | The linked sections of the pinned `release-17.sgml` |
| Defaults and gating conditions | The [settings table](#settings-and-conditions-that-gate-some-fixes) citations |
| `vacuum_defer_cleanup_age` was removed with boot value 0 | Commit `1118cd37eb` in the v17 history (first contained in `REL_16_BETA1`) |
| Sorted GiST builds now use the bulk writer, which syncs on finish | [gistbuild.c:409](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L409), [gistbuild.c:453](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L453), [bulk_write.c:131-223](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L223); commit `8af2565248` |

## Open Questions

- **The list is release-note-based, not a full audit of every commit.** A fix is listed only when a minor-release note describes damage to data. A fix whose note uses none of the search words, or describes the damage only vaguely, can be missing; [What was left out](#what-was-left-out) records the judgment calls. Fixes that never appeared in a minor-release section, such as fixes made only during a major development cycle, are out of scope.
- **Some inclusions are judgment calls.** The 9.5.25 `WHERE CURRENT OF` row is listed because the false matches feed `UPDATE`/`DELETE ... WHERE CURRENT OF`, which the note does not spell out. The 11.19 heap-LSN row is called a theoretical hazard by its note. The 14.23 FSM row lost hint data only. The 14.21 `NOTIFY` row lost a notification, not stored rows. The 13.1 and 14.24 read-path rows returned corrupted values without changing stored data. The 11.19 parallel hash row damaged temporary data. The six "Metadata" rows lost a user-set catalog property. A stricter reading would drop them.
- **Fixes that only an 11.x branch received are not covered.** The candidates come from the 12-17 notes. A fix that went to 11 and not to 12 or later would be missed.
- **The code links were located by line identity, not by reading each fix.** Each linked range holds lines identical to lines the fix commit added, and unique in their file. That shows the fix's code is present, but not that the range is the whole fix. Fixes that removed code, or whose code was rewritten later, show "History only".
- **The 10.23 inherited-UPDATE fix has no v17-line commit.** Whether the PostgreSQL 14 rewrite of inherited `UPDATE` planning removed the faulty path was not checked.
- **The 11.8 `pg_checksums` row describes the effect from the pinned code**, not from the release note, which says only that such subdirectories must not be processed. The claim that enable mode writes pages rests on [pg_checksums.c:237-258](../../../../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L237-L258).
- **The 14.23 release note and the pinned source disagree on the FSM condition.** The note says the map buffer was not marked dirty "if checksums are enabled". The pinned code shows the condition is `XLogHintBitIsNeeded()`, true with checksums or `wal_log_hints` ([xlog.h:118](../../../../raw/postgres-17/src/include/access/xlog.h#L118), [freespace.c:236-245](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L236-L245)). The row follows the source.
- **Re-check the 14.23 FSM and 17.11 VM rows on the next repin.** The local `REL_17_STABLE` branch beyond the pin was reported, during the review of the checksum page, to revert the FSM fix and to add a later VM fix. That is outside this page's evidence base.
- **The oldest release depends on the branches the note comment names.** A copy of a fix committed to an older branch under a different message would not be merged with it (`git_changelog` warns about this, [git_changelog:43-49](../../../../raw/postgres-17/src/tools/git_changelog#L43-L49)), so its oldest release could be older than shown.

## Source References

- [doc/src/sgml/release-17.sgml](../../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L4-L10)
- [src/tools/RELEASE_CHANGES](../../../../raw/postgres-17/src/tools/RELEASE_CHANGES#L86-L148)
- [src/tools/git_changelog](../../../../raw/postgres-17/src/tools/git_changelog#L8-L49)
- [src/backend/utils/misc/guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1061-L1068)
- [src/include/access/xlog.h](../../../../raw/postgres-17/src/include/access/xlog.h#L96-L118)
- [src/include/utils/rel.h](../../../../raw/postgres-17/src/include/utils/rel.h#L628-L631)
- [src/backend/replication/logical/logical.c](../../../../raw/postgres-17/src/backend/replication/logical/logical.c#L125-L128)
- [src/backend/access/transam/twophase.c](../../../../raw/postgres-17/src/backend/access/transam/twophase.c#L371-L376)
- [src/backend/access/transam/commit_ts.c](../../../../raw/postgres-17/src/backend/access/transam/commit_ts.c#L157-L158)
- [src/bin/initdb/initdb.c](../../../../raw/postgres-17/src/bin/initdb/initdb.c#L2412-L2417)
- [src/backend/commands/subscriptioncmds.c](../../../../raw/postgres-17/src/backend/commands/subscriptioncmds.c#L150-L151)
- [src/backend/replication/logical/worker.c](../../../../raw/postgres-17/src/backend/replication/logical/worker.c#L4386-L4394)
- [src/backend/backup/basebackup.c](../../../../raw/postgres-17/src/backend/backup/basebackup.c#L788-L791)
- [src/backend/utils/adt/pg_locale.c](../../../../raw/postgres-17/src/backend/utils/adt/pg_locale.c#L1378-L1382)
- [src/interfaces/ecpg/preproc/ecpg.c](../../../../raw/postgres-17/src/interfaces/ecpg/preproc/ecpg.c#L189-L192)
- [src/backend/access/heap/heapam.c](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L6600-L6640)
- [src/backend/access/index/genam.c](../../../../raw/postgres-17/src/backend/access/index/genam.c#L768-L794)
- [src/backend/storage/smgr/bulk_write.c](../../../../raw/postgres-17/src/backend/storage/smgr/bulk_write.c#L131-L223)
- Per-fix code links in the branch tables.

## Navigation

- [PostgreSQL 17 index](../../index.md)
- [Wiki index](../../../index.md)
- [Versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Data Checksum Bug Fixes Since PostgreSQL 12.2 (unverified)](data-checksum-bug-fixes-since-12-2.md)
