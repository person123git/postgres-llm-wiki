---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Physical Replication With Different Data Checksum Settings in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Consequences in each direction](#consequences-in-each-direction)
  - [Why ordinary WAL replay can work](#why-ordinary-wal-replay-can-work)
  - [Detection, errors and overhead](#detection-errors-and-overhead)
  - [Backups, promotion and pg_rewind](#backups-promotion-and-pg_rewind)
  - [Checking and aligning the settings](#checking-and-aligning-the-settings)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, what are the consequences of having two database servers using physical replication when one has data checksums enabled and the other does not?

The user confirmed that “data checkpoint” meant “data checksums” and that the question concerns a primary and a standby.

## Answer

**Different checksum settings do not, by themselves, prevent ordinary physical WAL replication in this PostgreSQL 12 pin. Each server verifies and computes data-page checksums according to its own local setting. However, keep the settings consistent operationally: the node with checksums off loses that detection, and tools that copy page bytes directly can leave invalid checksums when the nodes disagree.** This conclusion assumes the standby was cloned correctly and any local checksum conversion completed on a cleanly stopped cluster; it does not apply to merely changing a control-file flag. [walreceiver.c#WalReceiverMain](../../../../raw/postgres-12/src/backend/replication/walreceiver.c#L311-L334) [xlogutils.c#XLogReadBufferForRedoExtended](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L345-L407) [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [pg_checksums.c#offline-conversion](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L541) [pg_checksums.sgml#replication-warning](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L210-L219).

Here “two databases” means two server clusters. [Data checksums](../../../glossary.md#data-checksums) are a cluster-wide property, not a separate setting for each SQL database. The read-only `data_checksums` setting exposes `ControlFileData.data_checksum_version` from the local [control file](../../../glossary.md#control-file). PostgreSQL 12 `initdb` defaults to checksums off; `initdb --data-checksums` or a completed offline `pg_checksums --enable` operation enables them. [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835) [xlog.c#data_checksums-report](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4783-L4785) [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826) [initdb.c#default](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [initdb.c#checksum-option](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3146-L3148) [pg_checksums.c#publish-state](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L606).

### Consequences in each direction

The table concerns normal replay into a valid local data directory. It does not assume that copying arbitrary files between these servers is safe.

| Primary | Standby | Consequence |
|---|---|---|
| On | Off | WAL replay can continue. The standby does not validate data-page checksums on reads or recompute them on writes, so it does not inherit the primary's checksum protection. The primary's extra WAL for eligible hint changes still enters the shared WAL stream. [PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L122) [PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L185-L192) [XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L915-L953). |
| Off | On | WAL replay can continue after proper offline conversion of the standby. The standby computes checksums when it writes reconstructed pages and checks its own stored pages when it reads them. This does not make the primary validate its pages or detect every bad value arriving from the primary. [offline page conversion](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L211-L242) [replay](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L356-L407) [write](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2743-L2762) [read](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L122). |
| On | On | Each node checks its own eligible disk reads and creates checksums on its own page writes. Neither node's successful checks prove that the other node's disk is healthy or that all stored data is semantically correct. [read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L924) [write](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2743-L2762) [checksum inputs](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214). |
| Off | Off | This is the `initdb` default. Neither node performs the optional data-page checksum comparison; page-header sanity checks and WAL-record CRC validation still operate. [initdb.c#default](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L122) [ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L726). |

### Why ordinary WAL replay can work

Three different values matter. The local control file decides whether to use checksums. With checksums enabled, an initialized [page](../../../glossary.md#page) receives a `pd_checksum` describing its bytes at write time; with checksums disabled, that field is not maintained. A [WAL](../../../glossary.md#wal) record has its own CRC protecting the record independently of `data_checksums`. An in-memory page, including one copied into a [full-page image](../../../glossary.md#full-page-image), need not have an up-to-date `pd_checksum`. [pg_control.h#data_checksum_version](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L221-L222) [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L160) [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L726) [page/README#WAL-images](../../../../raw/postgres-12/src/backend/storage/page/README#L14-L26).

```text
Primary WAL record
  -> WAL record CRC validation
  -> replay on standby
       full-page image to apply?
         yes -> restore image into a buffer; mark it dirty
         no  -> read/initialize local page; apply required change; mark dirty
  -> standby writes an initialized page
       local data checksums on?
         yes -> compute pd_checksum from resulting page and block number
         no  -> write without recomputing pd_checksum
  -> later disk read
       local data checksums on?
         yes -> compare stored checksum with recomputed checksum
         no  -> skip checksum comparison, retain page-header checks
```

Map evidence: WAL CRC validation is in `ValidXLogRecord`; full-image restoration and the ordinary page/LSN branch are in `XLogReadBufferForRedoExtended`; heap insert redo illustrates initialization or modification followed by dirtying; `FlushBuffer` calls `PageSetChecksumCopy`; `PageIsVerified` controls later reads. The map's “apply required change” excludes records already reflected in the page's LSN. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L726) [xlogutils.c#replay-branches](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L345-L407) [heapam.c#heap_xlog_insert](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8142-L8197) [bufmgr.c#FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2743-L2762) [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L122).

The [WAL receiver](../../../glossary.md#wal-receiver) checks the primary's system identifier and timeline. The physical `START_REPLICATION` request carries the requested WAL location, optional slot and timeline; it does not negotiate `data_checksums`. Thus a different checksum setting is not itself a connection rejection. A matching system identifier alone is not enough to build a standby: the premise remains a valid base copy and the required WAL. [walreceiver.c#identity-check](../../../../raw/postgres-12/src/backend/replication/walreceiver.c#L311-L334) [libpqwalreceiver.c#start-command](../../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L370-L398) [libpqwalreceiver.c#physical-timeline](../../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L427-L448) [basebackup.c#backup-copy](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L337-L357).

On the full-image branch, replay restores the image into memory and marks it dirty. It does not require the primary's `pd_checksum` to match that image. On the ordinary modification branch, replay uses the standby's local page and later writes the resulting contents through its local checksum path. This is why “the primary has checksums off” does not imply “every replayed page on a checksummed standby fails verification.” [xlogutils.c#image-versus-page](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L356-L407) [heapam.c#heap_xlog_insert](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8142-L8197) [bufmgr.c#FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2743-L2762).

### Detection, errors and overhead

**Protection is local and limited.** An enabled standby can detect a mismatch on an eligible page read from its own storage. But computing a checksum over already-wrong bytes produces a checksum for those bytes; it does not establish that the primary supplied the right rows. Conversely, a disabled standby skips this comparison even if an old checksum field happens to remain in the page. Disabling checksums does not clear that field on every write. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L122) [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214).

With the default `ignore_checksum_failure = off` and `zero_damaged_pages = off`, a checksum mismatch emits `page verification failed, calculated checksum ... but expected ...`; an ordinary buffer read then raises `invalid page in block ... of relation ...`. A query can fail, or replay can fail when it needs that bad local page. This is not an automatic failure just because the settings differ. Full-image restoration uses a zero-and-lock buffer path, and some redo callers explicitly request `RBM_ZERO_ON_ERROR`, so not every replay access has the ordinary read outcome. [guc.c#failure-defaults](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161) [bufpage.c#checksum-warning](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L144-L161) [bufmgr.c#read-error](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) [xlogutils.c#image-buffer](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L356-L386) [heapam.c#heap_xlog_visible](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L7906-L7911).

Checksums also do not constitute a continuous full-database audit. A buffer-cache hit bypasses the disk-read verification path, and a completely zero page is accepted by `PageIsVerified`. For broader detection limits, see [Data Checksums and amcheck Detection Coverage in PostgreSQL 12 (unverified)](../storage-and-vacuum/checksums-and-amcheck-detection-coverage.md). [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) [bufpage.c#zero-page](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L125-L142).

**The costs differ by role.** An enabled node computes page checksums for writes and eligible reads. On the primary, enabling checksums also makes `XLogHintBitIsNeeded()` true even with the default `wal_log_hints = off`. For permanent shared buffers that reach the hint-dirtying path, `XLogSaveBufferForHint` can emit a full-page image when the page LSN is not newer than the redo location. Those extra records must also be transported and replayed. This is a source-level cost mechanism, not a measured throughput or latency estimate. [bufpage.c#read-computation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L103) [bufpage.c#write-computation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185) [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L185-L192) [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3452-L3510) [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L915-L953).

On a checksummed [hot standby](../../../glossary.md#hot-standby), [hint bits](../../../glossary.md#hint-bits) do not cause the server to originate equivalent WAL during recovery. When the permanent-buffer hint path needs WAL protection, `MarkBufferDirtyHint` returns during recovery instead of dirtying the page for that hint. Consequently, primary-off/standby-on does not require turning `wal_log_hints` on at the primary merely to make checksum handling during normal replay work. Rewind eligibility is a separate issue below. [bufmgr.c#recovery-hint-branch](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3452-L3481) [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L185-L192).

### Backups, promotion and pg_rewind

WAL replay reconstructs pages through the local buffer machinery. A direct file/block copy can bypass that machinery. This is the boundary that makes unequal settings operationally risky. [xlogutils.c#replay](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L356-L407) [pg_rewind/filemap.c#copy-decisions](../../../../raw/postgres-12/src/bin/pg_rewind/filemap.c#L242-L291) [pg_checksums.sgml#replication-warning](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L210-L219).

| Event | State change and consequence |
|---|---|
| Create a standby from a complete base backup | A [base backup](../../../glossary.md#base-backup) copies `pg_control` as well as data files, so the new copy initially has the source's checksum state. A completed offline conversion on that copy can change its local state afterwards. [basebackup.c#control-copy](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L337-L357) [pg_checksums.c#publish-state](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L606). |
| Enable checksums offline | The tool scans eligible relation files, writes checksums for initialized pages, makes the data durable by default, then updates the control file. Changing only the flag omits the page conversion and is not equivalent. [pg_checksums.c#page-write](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L211-L242) [pg_checksums.c#scan-and-publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L606). |
| Disable checksums offline | The tool changes the control-file state without rewriting every page. Future writes bypass checksum recomputation, so old checksum bytes must not be assumed valid. [pg_checksums.c#scan-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L561) [pg_checksums.c#publish-state](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L606) [bufpage.c#write-bypass](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189). |
| Promote the standby | Promotion does not convert the cluster's checksum state. A promoted checksums-off standby remains off; an enabled one retains local checking. The checksum decision still reads the local control value; the end-of-recovery transition changes recovery/production state without converting checksums. [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826) [xlog.c#end-of-recovery](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7763-L7803). |
| Copy raw blocks into an enabled node from a disabled node | The copied page can carry a stale or absent checksum. If it is subsequently read as an initialized page before being correctly reconstructed or rewritten, checksum validation can fail. A byte copy is not the replay/flush path. [bufpage.c#read-check](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L122) [bufpage.c#disabled-write](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [pg_checksums.sgml#replication-warning](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L210-L219). |
| Rejoin a former primary with `pg_rewind` | Rewind copies selected relation blocks and replaces the target's control state from the source. It does not perform a full checksum conversion of every retained target page. Mismatched states therefore require special care even when the command's initial eligibility checks pass. [filemap.c#retained-blocks](../../../../raw/postgres-12/src/bin/pg_rewind/filemap.c#L261-L291) [pg_rewind.c#source-control-copy](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L368-L392) [pg_rewind.c#sanityChecks](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L408-L429). |

A concrete rewind hazard follows from that last row. This is a source-derived scenario, not a runtime reproduction:

1. The rewind source has checksums on. The target has checksums off but has maintained `wal_log_hints = on`, so it can pass the hint-logging eligibility gate. [pg_rewind.c#eligibility](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429).
2. Same-size relation files retain blocks that the WAL-derived copy map does not select. [filemap.c#selective-copy](../../../../raw/postgres-12/src/bin/pg_rewind/filemap.c#L261-L291).
3. Rewind gives the target the source's checksum-enabled control state without converting all retained pages. [pg_rewind.c#copy-control](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L368-L392).
4. A retained page not reconstructed by subsequent replay can still have an invalid checksum when read. Rewind success is therefore not proof of a valid checksum conversion. [bufpage.c#disabled-write](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [bufpage.c#read-check](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L122).

There is a second, distinct rejoin consequence: a former primary with checksums off and the default `wal_log_hints = off` fails `pg_rewind`'s target eligibility check. The target is the server being rewound, which need not be the server currently called the primary. If planning to use `wal_log_hints = on` instead of checksums for rewind eligibility, it requires a server restart (`PGC_POSTMASTER`) and must cover the relevant history; changing it afterwards does not create missing earlier hint WAL. The rewind documentation also requires `full_page_writes = on`, which is the default. Eligibility alone still does not solve unequal checksum states. [pg_rewind.c#target-gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429) [guc.c#WAL-defaults-and-contexts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1163-L1185) [xlog.h#hint-condition](../../../../raw/postgres-12/src/include/access/xlog.h#L185-L192) [pg_rewind.sgml#requirements](../../../../raw/postgres-12/doc/src/sgml/ref/pg_rewind.sgml#L61-L76) [pg_rewind.sgml#hint-and-full-page-requirements](../../../../raw/postgres-12/doc/src/sgml/ref/pg_rewind.sgml#L92-L99).

### Checking and aligning the settings

Inspect each server in a dedicated SQL session:

```sql
SET /* wiki_checksum_replication_statement_timeout */ statement_timeout = '5s';
SET /* wiki_checksum_replication_lock_timeout */ lock_timeout = '1s';
SHOW /* wiki_checksum_replication_state */ data_checksums;
```

The timeout values above are suggested session limits, not PostgreSQL defaults: both default to zero (disabled) and have `PGC_USERSET` scope. `data_checksums` is `PGC_INTERNAL` and disallowed in configuration files; `SET`, reload and an ordinary configuration-file restart cannot enable it. Its displayed value comes from the local control file at startup. [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2397) [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835) [xlog.c#reported-state](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4783-L4785).

**Recommended maintenance plan:** choose a consistent state for the replication topology. PostgreSQL 12's documentation recommends stopping all clusters before switching them consistently with the offline tool. Its alternative is to change the primary and recreate the standbys from fresh base backups. The tool requires a clean shutdown, and no process may write into a data directory while conversion runs. These are planned maintenance operations, not changes performed by this answer. [pg_checksums.c#shutdown-check](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L521) [pg_checksums.sgml#maintenance](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L204-L219).

Enabling a checksum over existing bytes does not establish that those bytes were correct before conversion. If mismatched nodes have already exchanged raw relation blocks or experienced corruption, preserve that distinction when investigating; a new checksum is not a repair of the logical contents. [pg_checksums.c#enable-writes-current-checksum](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L211-L242) [checksum_impl.h#inputs](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214).

## Context Reviewed

- Target: PostgreSQL 12.2, exact pin `45b88269a353ad93744772791feb6d01bc7e1e42`; the answer is a source reading, with no runtime measurement or server started.
- Caller/callee and structures: receiver identity and start command; WAL record validation; redo buffer selection and heap insert redo; buffer reads and flushes; `PageHeaderData.pd_checksum`, `ControlFileData.data_checksum_version`, and hint-dirtying gates. Citations appear beside each claim above.
- Lifecycle boundaries: base backup's control-file copy; offline checksum scan and publication; rewind's target eligibility, selective copying and source-derived control state.
- Build boundary: these are core backend and bundled utility paths, not an extension. Backend page code builds `bufpage.o` and `checksum.o`; the utility includes the shared checksum implementation. Generated `pg_config.h` supplies build configuration through `c.h`; the checksum tool explicitly rejects a cluster with an incompatible block size. No generated header was edited or assumed to exist in the raw tree. [page/Makefile](../../../../raw/postgres-12/src/backend/storage/page/Makefile#L10-L18) [pg_checksums/Makefile](../../../../raw/postgres-12/src/bin/pg_checksums/Makefile#L13-L23) [pg_checksums.c#includes](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L20-L31) [c.h#config-headers](../../../../raw/postgres-12/src/include/c.h#L51-L55) [configure.in#generated-header](../../../../raw/postgres-12/configure.in#L2473-L2477) [pg_checksums.c#block-size-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L503-L508).
- Tests reviewed: `pg_checksums` TAP coverage enables, disables, verifies and rejects online operation. The basic `pg_rewind` test uses the common streaming setup, which overrides the default `wal_log_hints = off` with `on`, allowing rewind's target gate to pass without checksums. No test of the two mixed-state streaming directions was located in `src/test/recovery/t/`, `src/bin/pg_checksums/t/` or `src/bin/pg_rewind/t/`. These tests were inspected, not run. [002_actions.pl#actions](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L89-L177) [001_basic.pl#setup](../../../../raw/postgres-12/src/bin/pg_rewind/t/001_basic.pl#L11-L16) [RewindTest.pm#setup_cluster](../../../../raw/postgres-12/src/bin/pg_rewind/t/RewindTest.pm#L120-L135) [PostgresNode.pm#streaming-settings](../../../../raw/postgres-12/src/test/perl/PostgresNode.pm#L462-L478) [guc.c#wal_log_hints-default](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185) [pg_rewind.c#target-gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429).
- Glossary reviewed: Data checksums, Control file, Page, WAL, Full-page image, WAL receiver, Hot standby, Hint bits and Base backup, including PostgreSQL 12 notes. No definition correction needed. PostgreSQL 12 has no matching common-concept page; the required mechanism is explained here.
- Contradiction comparison: complete page, linked glossary, both navigation entries, the v12 manifest row, and matching-version checksum detection, amcheck and database-health discussions. The stale README restriction, test-helper comment and an overbroad lifecycle claim on the amcheck page are recorded below. The current answer follows the implementation.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Local cluster state, not a per-database switch | [DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826), [data_checksums GUC](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835) |
| No checksum-state negotiation in physical streaming startup | [receiver identity checks](../../../../raw/postgres-12/src/backend/replication/walreceiver.c#L311-L334), [request fields](../../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L370-L398), [physical timeline](../../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L427-L448) |
| Replayed page gets the local write-time checksum | [redo branches](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L356-L407), [FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2743-L2762), [PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) |
| Read detection, warning and ordinary error | [PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L161), [ReadBuffer_common](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L924) |
| Hint WAL on primary and recovery's suppression of hint-only dirtying | [XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L185-L192), [MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3452-L3510), [XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L915-L953) |
| Offline conversion differs from changing a flag | [page rewrite](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L211-L242), [shutdown gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L521), [scan and control publication](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L606) |
| Rewind eligibility does not prove checksum compatibility | [target gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429), [selective retention](../../../../raw/postgres-12/src/bin/pg_rewind/filemap.c#L261-L291), [source control copy](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L368-L392) |

## Open Questions

- No mixed-state replication or rewind scenario was executed for this answer. The direction table and rewind example are implementation deductions; workload-specific CPU, WAL volume and latency effects remain unmeasured.
- Same-checkout documentation discrepancy: `src/backend/storage/page/README` says checksums must be enabled at `initdb` time. The pinned `pg_checksums` implementation can enable them later on a cleanly stopped cluster by rewriting eligible pages before publishing the control state. The implementation wins; the README's creation-only statement is stale. [page/README#stale-initdb-restriction](../../../../raw/postgres-12/src/backend/storage/page/README#L12) [pg_checksums.c#enable](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L211-L242) [pg_checksums.c#publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L606).
- Same-checkout test-comment discrepancy: `RewindTest.pm` says “data checksums are mandatory,” but its streaming setup supplies `wal_log_hints = on`, and `pg_rewind` accepts either condition. The basic test does not request `initdb -k`. The actual setup and eligibility code win over the comment. [RewindTest.pm#comment-and-setup](../../../../raw/postgres-12/src/bin/pg_rewind/t/RewindTest.pm#L120-L135) [001_basic.pl#setup](../../../../raw/postgres-12/src/bin/pg_rewind/t/001_basic.pl#L11-L16) [PostgresNode.pm#initdb](../../../../raw/postgres-12/src/test/perl/PostgresNode.pm#L437-L440) [PostgresNode.pm#wal_log_hints](../../../../raw/postgres-12/src/test/perl/PostgresNode.pm#L462-L478) [pg_rewind.c#target-gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429).
- Outside this page's edit scope: the `data_checksum_version` row in [amcheck's values table](../indexing/amcheck-indexes-tables-and-data-checksums.md#mental-model) says it “changes only at `initdb` or with offline `pg_checksums`.” This page's rewind lifecycle row identifies another way the target acquires a different value: `pg_rewind` copies the source's control structure and publishes it. The source supports the latter; the sibling's “only” needs qualification for replacement by physical-copy tools. The sibling was not edited. [pg_rewind.c#source-control-copy](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L368-L392).
- No matching PostgreSQL 12 common-concept page exists for physical replication and checksum boundaries. A separate concept page could cover that reusable mechanism; none was created as part of this answer.

## Source References

- [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826)
- [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L726)
- [xlogutils.c#XLogReadBufferForRedoExtended](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L345-L407)
- [heapam.c#heap_xlog_insert](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8142-L8197)
- [bufmgr.c#FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2743-L2762)
- [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3452-L3510)
- [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L161)
- [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189)
- [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214)
- [walreceiver.c#identity-check](../../../../raw/postgres-12/src/backend/replication/walreceiver.c#L311-L334)
- [libpqwalreceiver.c#libpqrcv_startstreaming](../../../../raw/postgres-12/src/backend/replication/libpqwalreceiver/libpqwalreceiver.c#L370-L398)
- [basebackup.c#control-copy](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L337-L357)
- [pg_checksums.c#offline-conversion](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L489-L606)
- [pg_rewind.c#control-copy-and-target-gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L368-L429)
- [filemap.c#selective-copy](../../../../raw/postgres-12/src/bin/pg_rewind/filemap.c#L261-L291)
- [pg_checksums.sgml#replication-warning](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L204-L219)
- [002_actions.pl#actions](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L89-L177)

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Version manifest](../../../versions.md)
