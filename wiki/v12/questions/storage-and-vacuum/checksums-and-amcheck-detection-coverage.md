---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# What Data Checksums and amcheck Detect, and What They Miss, in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Four kinds of failure](#four-kinds-of-failure)
  - [Mental model](#mental-model)
  - [Logic map](#logic-map)
  - [Detected data corruption](#detected-data-corruption)
  - [Undetected data corruption and data loss](#undetected-data-corruption-and-data-loss)
  - [Coverage comparison table](#coverage-comparison-table)
  - [Non-default settings that change the result](#non-default-settings-that-change-the-result)
  - [Residual risks and additional protection](#residual-risks-and-additional-protection)
  - [Causal summary](#causal-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Question: PostgreSQL Data Corruption and Data Loss Detection Coverage
Analyze the effectiveness and limitations of PostgreSQL data checksums and amcheck / pg_amcheck when both are enabled and executed routinely.
Divide the analysis into two main categories:
1. Detected Data Corruption
Identify the types of corruption that can be detected by PostgreSQL data checksums, amcheck, or both.
Include:
* Disk-level bit flips and physical page corruption
* Torn pages and partial writes
* Unexpected page modifications
* B-tree index structural corruption
* Missing or inconsistent index entries
* Heap page and tuple corruption
* Invalid tuple metadata
* TOAST data corruption
* Corruption caused by memory, storage, or I/O failures
* Index ordering and consistency problems
2. Undetected Data Corruption and Data Loss
Identify corruption or data-loss scenarios that can go undetected even when checksums and amcheck routinely complete successfully.
Include:
* Silent loss of rows or records
* Logically incorrect but structurally valid data
* Accidental or unauthorized deletion or modification
* Missing data that leaves tables and indexes consistent
* Stale pages containing valid checksums
* Lost committed transactions
* WAL corruption or missing WAL segments
* Missing database files or relations
* Incomplete index-type coverage
* Application-level data integrity violations
* Storage failures producing internally consistent but incorrect database states
* Replication-related data loss
3. Coverage and Limitations
For each scenario, explain:
* Whether checksums detect it
* Whether amcheck detects it
* Whether detection depends on configuration or execution options
* Whether detection is guaranteed, conditional, or unavailable
* Whether corruption can remain undetected despite successful checks
* Whether the mechanism detects, prevents, or repairs the problem
Present the findings in a comparison table with columns:
Failure Scenario | Checksums | amcheck | Detection Status | Limitations
4. Residual Risks and Additional Protection
Explain which risks remain after both checks complete successfully and what additional mechanisms are needed, including backup verification, restore testing, WAL validation, and application-level reconciliation.
Objective: Clearly establish what PostgreSQL data checksums and routine amcheck executions can detect, what they cannot detect, and why successful checks do not guarantee complete data integrity or protection against data loss.
Ensure that the analysis distinguishes between physical corruption, structural corruption, logical corruption, and actual data loss. Avoid describing conditional detection as guaranteed coverage.

## Answer

### Short answer

**In PostgreSQL 12, `pg_amcheck` does not exist, and [amcheck](../../../glossary.md#amcheck) checks only [B-tree](../../../glossary.md#b-tree) indexes.** The `src/bin` build list has no `pg_amcheck` program, and the amcheck module builds one file, `verify_nbtree.c`, whose functions reject every relation that is not a B-tree index. The question's premise therefore holds only for amcheck's two SQL functions, `bt_index_check()` and `bt_index_parent_check()`. Tables are checked only indirectly, through the `heapallindexed` option. [bin/Makefile#SUBDIRS](../../../../raw/postgres-12/src/bin/Makefile#L16-L34) [amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4) [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323)

Neither mechanism guarantees detection of anything. Each one detects some failures **conditionally**:

- [Data checksums](../../../glossary.md#data-checksums) are off by default. When on, they detect damage to an initialized page only when the server reads that page from disk. Cached pages, all-zero pages and pages nobody reads are not tested, and a 16-bit checksum can match by chance. [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [checksum_impl.h#reduce](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214)
- amcheck detects broken B-tree structure on pages its walk reaches. With `heapallindexed = true` it also detects heap rows that lack a matching index entry, probabilistically. It never checks the reverse direction, other index types, or table columns outside the index. [verify_nbtree.c#bt_target_page_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10)

Both mechanisms only **detect**. Neither prevents nor repairs damage. The amcheck docs say a pass cannot prove that corruption is absent, and that there is no general repair method. [amcheck.sgml#cannot-prove-absence](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L316-L317) [amcheck.sgml#repair](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L321-L337)

Both are blind to whole classes of loss: correctly written but unwanted changes, lost commits, broken or missing [WAL](../../../glossary.md#wal), and missing files or segments that leave a consistent shorter relation. Those need backups, restore tests, WAL checks and application reconciliation.

This page is a source reading. It runs no measurement. The companion page [How amcheck Checks B-Tree Indexes and Their Tables (unverified)](../indexing/amcheck-indexes-tables-and-data-checksums.md) measures many of the amcheck and checksum boundaries cited here, and [amcheck on zeroed relation segments (unverified)](../indexing/amcheck-zeroed-relation-segments.md) covers missing segments.

### Four kinds of failure

The question asks to separate four kinds of failure. This page uses these definitions:

| Kind | Meaning | Example | Can a page-level or index-level check see it? |
|---|---|---|---|
| Physical corruption | Bytes on disk differ from the bytes PostgreSQL wrote. | Bit flip, [torn page](../../../glossary.md#torn-page), misdirected write | Checksums can, on read. amcheck can if the damage breaks an invariant it tests. |
| Structural corruption | Bytes are internally inconsistent with the format's rules. | Keys out of order, broken sibling links, bad line pointers | amcheck, for B-trees. Checksums only if the damage happened after the checksum was computed. |
| Logical corruption | Every structure is valid, but the content is wrong. | Wrong value written by an application or a bug, an index entry that disagrees with its heap row | amcheck only for the index/heap disagreement it probes. Checksums never. |
| Data loss | Data that was committed or existed is gone, and what remains is consistent. | Lost commit, deleted row, missing segment | Usually neither, because nothing left behind is inconsistent. |

### Mental model

Each check compares a value it reads with another value. A check can find only a disagreement between those two values. It cannot find an error that changed both, or that left nothing to compare.

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| `pd_checksum` | 16-bit checksum in the page header | Computed from the buffer at flush time when checksums are on and the page is initialized. [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) | Stored on disk | Compared with a recomputed value when the page is next read from disk. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) |
| Block number | The page's position in the relation | Mixed into the checksum. [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) | Live, from the read request | Makes a page written to the wrong block of the same relation fail. No relation identifier is mixed in. |
| Cached page | The copy in [shared buffers](../../../glossary.md#shared_buffers) | A previous read or write | Stored in memory | A buffer hit returns it without verification. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) |
| B-tree invariants | Rules that must hold on index pages, such as key order | Checked by amcheck on each reached page. [verify_nbtree.c#bt_target_page_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) | Live, per call | One `ERROR` on the first violation. |
| Index fingerprint | A [Bloom filter](../../../glossary.md#bloom-filter) of the index's leaf entries | Built during the walk when `heapallindexed` is true. [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083) | Live, per call | Probed for each heap row that belongs in the index. |
| WAL record CRC | CRC-32C over each WAL record | Computed at insert; checked by `ValidXLogRecord()` when WAL is read. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725) | Stored in WAL | A mismatch ends the readable WAL. Neither checksums nor amcheck read WAL. |

### Logic map

```text
A failure happens
  |
  +-- Did it change bytes after PostgreSQL computed their checksum? -- no --> checksums cannot see it   [1]
  |      | yes
  |      +-- checksums on (initdb -k / pg_checksums --enable)? -- no --> no checksum test           [2]
  |      +-- is the page ever read from disk (not cached, not skipped)? -- no --> not tested       [3]
  |      +-- is the page all zero? -- yes --> accepted without a checksum                         [4]
  |      +-- did the 16-bit value happen to match? -- yes --> accepted                             [5]
  |      +-- else: WARNING + ERROR "invalid page in block" (unless a superuser override is on)     [6]
  |
  +-- Is it in a B-tree index page the amcheck walk reaches? -- yes --> invariant checks            [7]
  +-- Is it a heap row that should have an index entry, with heapallindexed? -- yes --> Bloom probe [8]
  |      (probe asks "is this row in the index?", never "is this index entry in the heap?")
  |
  +-- Otherwise: WAL, missing files, other index types, non-indexed values, unwanted but valid
      writes, lost commits --> neither check looks at it                                           [9]
```

- **[1]** [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750)
- **[2]** [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826)
- **[3]** [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796)
- **[4]** [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142)
- **[5]** [checksum_impl.h#reduce](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214)
- **[6]** [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925)
- **[7]** [verify_nbtree.c#root-to-leaf-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L551)
- **[8]** [verify_nbtree.c#heap-phase](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L625), [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926)
- **[9]** [amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4), [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725)

### Detected data corruption

Every entry below is **conditional**. None is guaranteed.

1. **Disk-level bit flips and physical page corruption (physical).** With checksums on, a read from disk recomputes the checksum over the whole page and its block number. A mismatch logs `page verification failed` and raises `invalid page in block`. Without checksums, only damage that breaks the page-header sanity test, or an amcheck invariant, is caught. [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925)
2. **Torn pages and partial writes (physical).** A torn page mixes old and new halves, so its checksum normally no longer matches. Checksums therefore detect it on read, but do not repair it. Repair comes from WAL: with `full_page_writes` on (the default), recovery restores a [full-page image](../../../glossary.md#full-page-image) taken after the last checkpoint. With checksums on, hint-bit-only changes also log an image under the conditions in the companion page. A file shorter than a requested block fails with a short-read error. [guc.c#full_page_writes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1164-L1175) [xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192) [bufmgr.c#hint-wal-gates](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3443-L3510) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)
3. **Unexpected page modifications (physical).** A change made outside PostgreSQL after the write fails the checksum on the next disk read. A page placed at the wrong block of the same relation also fails, because the block number is in the checksum. A valid page from another relation at the same block number passes. [checksum_impl.h#blkno](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L207-L208)
4. **B-tree index structural corruption (structural).** amcheck checks page sanity, line pointers, levels, high keys and sibling links on every page its walk reaches. `bt_index_parent_check` adds parent/child checks and needs a `ShareLock`, which blocks writes and fails on a [hot standby](../../../glossary.md#hot-standby). [verify_nbtree.c#page-sanity](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2396-L2471) [verify_nbtree.c#bt_downlink_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1508-L1647) [verify_nbtree.c#lock-choice](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L232-L263) [lock.c#recovery-lock-limit](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766)
5. **Index ordering and consistency problems (structural/logical).** Ordering checks compare items within a page, against the high key and across siblings. They catch an index whose operator class or collation no longer sorts its keys the same way, such as after an operating-system collation change. [verify_nbtree.c#order-checks](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1130-L1258) [amcheck.sgml#collation](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L222-L250)
6. **Missing or inconsistent index entries (logical).** Only with `heapallindexed = true`, amcheck forms the index entry each qualifying heap row should have and probes the fingerprint. "Definitely absent" raises `lacks matching index tuple` (`XX001`). The probe is a Bloom filter, so a missing entry can escape with small probability. [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) [amcheck.sgml#bloom-memory](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L195-L207)
7. **Heap page and tuple corruption (physical/structural).** amcheck has no heap checker at this pin. With checksums on, the `heapallindexed` scan reads every heap page of the main fork, so each initialized page read from disk gets a checksum test. Without checksums, only damage to indexed values, HOT chains or detoasted values used by the index is found. [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L351-L379) [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1666)
8. **Invalid tuple metadata (structural).** No v12 code validates heap tuple headers such as `xmin`, `xmax` or infomask bits as a check. A broken HOT chain is found incidentally: the build scan raises `failed to find parent tuple for heap-only tuple`. Damaged header bytes otherwise surface only as a checksum failure, with checksums on. [heapam_handler.c#hot-root-error](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1634-L1652)
9. **[TOAST](../../../glossary.md#toast) data corruption (physical/structural).** amcheck fetches TOAST values only when an indexed column, index expression or partial-index predicate needs them. A missing chunk then raises the TOAST reader's error. Other TOAST values are not read. The TOAST table's own pages get checksum tests only when something reads them. [indextuple.c#external-fetch](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88) [tuptoaster.c:2013](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2013)
10. **Corruption caused by memory, storage or I/O failures.** Storage and I/O damage after the write is checksum territory. Memory damage **before** the write gets a valid checksum, because the checksum is computed from the buffer at flush time. amcheck can still catch memory damage that breaks an invariant or changes an indexed value; the docs say `heapallindexed` greatly increases the chance of catching single-bit memory errors. [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) [amcheck.sgml#ram](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L295-L313)

### Undetected data corruption and data loss

These cases can go undetected even when every routine check completes successfully.

1. **Silent loss of rows or records (data loss).** A zeroed heap page is accepted without a checksum test and contributes no rows. The `heapallindexed` probe asks only whether each found heap row is in the index, so index entries whose heap rows vanished are not reported. [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614)
2. **Logically incorrect but structurally valid data (logical).** A wrong value written through normal SQL or by a server bug gets a valid checksum and a matching index entry. Nothing compares it with what it should have been. [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750)
3. **Accidental or unauthorized deletion or modification (data loss/logical).** A committed `DELETE`, `UPDATE`, `TRUNCATE` or `DROP` goes through normal code paths. It produces correct pages, checksums and index entries, so both checks pass.
4. **Missing data that leaves tables and indexes consistent (data loss).** A heap and its indexes that both lost the same rows agree with each other. A missing trailing segment file is also invisible: `mdnblocks()` stops counting at the first segment it cannot open, so the relation looks shorter rather than broken. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750)
5. **Stale pages containing valid checksums (physical, looks valid).** A lost write leaves the older page image on disk. That image carries its own valid checksum and correct block number, so checksums pass. amcheck catches it only if the stale page breaks an invariant or lacks an entry for a heap row that the scan finds. [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214)
6. **Lost committed transactions (data loss).** A commit is durable locally only when its WAL is flushed. With `synchronous_commit = off` the commit returns before the flush, so a crash can lose it; the source comment says so. With `fsync = off`, flushes are not forced, so a system crash can lose any recent write. Both settings default to safe values. After either loss the remaining database is consistent, so both checks pass. [xact.c#commit-flush](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1367-L1400) [guc.c#synchronous_commit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4354-L4361) [guc.c#fsync](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1133)
7. **WAL corruption or missing WAL segments (data loss).** WAL records carry their own CRC, checked by `ValidXLogRecord()` while WAL is read, not by checksums or amcheck. During recovery, the redo loop stops at the first record `ReadRecord()` cannot return, and the server logs `redo done at`. Everything after a damaged record or a missing segment is lost, and the resulting database passes both checks. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725) [xlog.c#redo-loop-end](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7248-L7250) [xlog.c#redo-done](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7297-L7299)
8. **Missing database files or relations (data loss).** A missing first segment makes `mdopen()` raise `could not open file` when the relation is accessed. That error comes from the storage manager, not from either check; amcheck reports it only if it opens that relation. A dropped relation leaves no catalog entry, so nothing looks for it. [md.c#mdopen](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L450-L463)
9. **Incomplete index-type coverage (structural).** amcheck rejects GiST, GIN, hash, BRIN and SP-GiST indexes. Their structure is checked by nothing; checksums still test their pages on disk reads. [check_btree.sql#brin-rejection](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L49-L53)
10. **Application-level data integrity violations (logical).** Business rules, cross-table invariants and constraints that were never declared are outside both mechanisms. amcheck does not even check B-tree uniqueness: it clears `ii_Unique` before its scan. [verify_nbtree.c#unique-off](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L599-L607)
11. **Storage failures producing internally consistent but incorrect states (physical, looks valid).** A storage snapshot or restore that returns an older but complete set of pages has valid checksums and consistent indexes. Neither check compares the data with any external reference.
12. **Replication-related data loss (data loss).** Synchronous replication is off by default: `synchronous_standby_names` is empty, so a commit does not wait for any standby. A failover to an asynchronous standby can then lose commits that the primary acknowledged. The promoted standby is internally consistent and passes both checks. [guc.c#synchronous_standby_names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4087-L4095) [syncrep.c#SyncStandbysDefined](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L92-L93) [syncrep.h#SyncRepRequested](../../../../raw/postgres-12/src/include/replication/syncrep.h#L19-L20) [syncrep.c#fast-exit](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L158-L175)

### Coverage comparison table

"Detects" means "raises an error when the case is met". No row is guaranteed; the best status is **conditional**.

| Failure Scenario | Checksums | amcheck | Detection Status | Limitations |
|---|---|---|---|---|
| Bit flip on an initialized page on disk | Detects on disk read, if checksums are on | Detects only if an invariant or indexed value breaks | Conditional | Cached, unread and all-zero pages untested; 1-in-65,535 chance of a match; detects only, no repair |
| Torn page / partial write | Detects on read | Possibly, through invariants | Conditional | Repair needs `full_page_writes` (default on) and replay; truncated file errors on short read |
| Page at the wrong block of the same relation | Detects on read | Possibly | Conditional | Same-block page from another relation passes |
| B-tree structural damage | No, unless the page was also damaged on disk | Detects on reached pages | Conditional | Unreached pages unchecked; parent checks need `bt_index_parent_check` and a `ShareLock`; first error stops the call |
| B-tree ordering / collation drift | No: pages are validly written | Detects | Conditional | Only B-trees; standby collation can differ from the primary |
| Heap row lacking an index entry | No | Detects with `heapallindexed` | Conditional, probabilistic | Bloom filter can miss; partial-index and snapshot rules skip rows |
| Index entry whose heap row is gone | No | No | Unavailable | One-way probe only |
| Heap page bytes damaged on disk | Detects on read; amcheck's heap scan supplies the read | Only damage in indexed values | Conditional | Needs checksums on and `heapallindexed` to read the heap |
| Invalid tuple header metadata | Only if damaged on disk | Only a broken HOT chain | Conditional / mostly unavailable | No heap checker in v12 |
| TOAST damage | Only for TOAST pages that are read | Only for values the index needs | Conditional | Other TOAST values unread |
| Memory damage before the write | No: valid checksum is computed over wrong bytes | Possibly, via invariants and probe | Conditional | Non-indexed values unchecked |
| Zeroed heap page | No: accepted without test | No: rows simply vanish | Unavailable | Silent row loss |
| Stale page with valid checksum (lost write) | No | Possibly | Conditional | Old image is self-consistent |
| Valid but wrong data; unwanted DML | No | No | Unavailable | Logically wrong, structurally valid |
| Lost committed transaction | No | No | Unavailable | Prevented only by flush settings and sync replication |
| WAL corruption or missing WAL | No (WAL has its own CRC) | No | Unavailable to both | Recovery ends at the first bad record |
| Missing trailing segment | No | No: relation looks shorter | Unavailable | `mdnblocks()` stops at the gap |
| Missing first segment / dropped relation | No | Error only if amcheck opens it; dropped relation not seen | Conditional / unavailable | Error comes from the storage manager |
| Non-B-tree index damage | On disk reads only | No | Unavailable for structure | GiST, GIN, hash, BRIN, SP-GiST rejected |
| Application integrity violation; duplicate keys | No | No (uniqueness not checked) | Unavailable | Needs constraints or reconciliation |
| Consistent but outdated storage state | No | No | Unavailable | Needs an external reference |
| Replication loss on failover | No | No | Unavailable | Async by default |

### Non-default settings that change the result

| Setting | Default | Cause and mechanism | Effect |
|---|---|---|---|
| Data checksums | Off: `initdb` sets them only with `-k`, or `pg_checksums --enable` on a stopped cluster. [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [pg_checksums.c#shutdown-required](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L513-L521) | `DataChecksumsEnabled()` reads the control file and gates the checksum computation and test. [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826) | Every checksum row above assumes this non-default state. |
| `heapallindexed` | False when omitted. [verify_nbtree.c#bt_index_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L180-L192) | Only this option builds the fingerprint and scans the heap. | Without it amcheck never reads the heap, so heap checksums are not exercised by amcheck. |
| `ignore_checksum_failure`, `zero_damaged_pages` | Both off. [guc.c#ignore_checksum_failure](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161) | A superuser can `SET` them (`superuser` context, session scope). `PageIsVerified()` then accepts the page with a warning, or `ReadBuffer_common()` zeroes it. [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L908-L918) | A checksum failure stops being an error; with `zero_damaged_pages`, heap rows silently disappear. |
| `synchronous_commit` | `on` (`remote_flush`). [xact.h:79](../../../../raw/postgres-12/src/include/access/xact.h#L79) [guc.c#synchronous_commit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4354-L4361) | `off` takes the asynchronous branch in `RecordTransactionCommit()`. Context `user`: `SET` in the session. | Recent commits can be lost in a crash. |
| `fsync`, `full_page_writes` | Both on. [guc.c#fsync](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1133) [guc.c#full_page_writes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1164-L1175) | Context `sighup`: changed in the configuration file and applied on reload. | Off removes crash durability or torn-page repair; checksums still only detect. |
| `synchronous_standby_names` | Empty. [guc.c#synchronous_standby_names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4087-L4095) | Context `sighup`. When non-empty and `synchronous_commit` is above `local`, commits wait in `SyncRepWaitForLSN()`. [syncrep.h#SyncRepRequested](../../../../raw/postgres-12/src/include/replication/syncrep.h#L19-L20) | Reduces failover loss; does not detect corruption. |

### Residual risks and additional protection

After both checks succeed, these risks remain: every **Unavailable** row above, plus missed cases in the **Conditional** rows. The pinned tree offers these additional tools. Each also has limits.

| Need | Mechanism in v12 | What it adds | Limit |
|---|---|---|---|
| Check every file, not only pages that are read | Offline `pg_checksums --check`; [base backup](../../../glossary.md#base-backup) verification during `BASE_BACKUP` | Reads whole files rather than following links. [pg_checksums.c#directories](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L545-L561) [basebackup.c#verify-gate](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1413-L1430) | Skips `pd_upper == 0` pages; the backup skips unlogged and temporary relations and pages newer than its start. [basebackup.c#skip-and-retry](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1470-L1556) |
| Backup verification | No backup manifest or `pg_verifybackup` exists at this pin (absent from `src/bin`). [bin/Makefile#SUBDIRS](../../../../raw/postgres-12/src/bin/Makefile#L16-L34) | — | A backup's integrity is shown only by restoring it. |
| Restore testing | Restore a base backup, replay archived WAL, then run amcheck and application checks on the copy. | Exercises WAL availability and CRCs during replay. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725) | A test restore that stops early at a bad record still starts; compare the `redo done at` position with the expected end. [xlog.c#redo-done](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7297-L7299) |
| WAL validation | `pg_waldump` decodes WAL through `XLogReadRecord()`, the reader that runs the CRC check. [pg_waldump.c:1134](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L1134) | Finds damaged or missing archived segments before a restore needs them. | Shows only that records are readable, not that the archive is complete for your recovery target. |
| Prevent lost commits | Keep `fsync`, `full_page_writes` and `synchronous_commit` at their defaults; configure `synchronous_standby_names` for failover. | Prevents, rather than detects, loss. | Async standbys still lag. |
| Logical correctness | Declared constraints, plus application reconciliation such as row counts and totals compared with an external system of record. | The only defence against valid-but-wrong data and unwanted changes. | Must be designed per application. |
| Non-B-tree indexes | `REINDEX` rebuilds an index from the heap. | Repairs index-only damage. | Does not detect, and the amcheck docs warn it may not fix the cause. [amcheck.sgml#repair](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L321-L337) |

### Causal summary

A checksum is computed from a buffer when it is written → it is tested only when that page is next read from disk, not cached, not all zero → so it detects later physical damage to pages someone reads, and nothing else. amcheck reads B-tree pages it can reach and, with `heapallindexed`, every heap row of one table → so it detects broken B-tree structure and heap rows missing from the index, probabilistically. Neither reads WAL, compares data with an external reference, or notices data that is consistently absent → so lost commits, lost WAL, missing segments, unwanted valid writes and stale-but-valid pages pass both checks. Both only detect; durability settings prevent loss, and backups with tested restores are the only repair.

## Context Reviewed

- `contrib/amcheck/` (Makefile, `verify_nbtree.c`, tests) and `doc/src/sgml/amcheck.sgml`.
- Checksum path: `bufpage.c`, `bufmgr.c`, `checksum_impl.h`, `xlog.c#DataChecksumsEnabled`, `initdb.c`, `pg_checksums.c`, `basebackup.c`.
- Storage manager: `md.c` (`mdopen`, `mdnblocks`, short reads).
- WAL and commit: `xlogreader.c#ValidXLogRecord`, `xlog.c` redo loop, `xact.c#RecordTransactionCommit`, `guc.c` durability settings.
- Replication: `syncrep.c`, `syncrep.h`.
- `src/bin/Makefile` for the absence of `pg_amcheck` and `pg_verifybackup`.
- Related pages: the v12 amcheck, emptied-segment and zeroed-segment pages. No v12 common concept pages exist.

## Evidence Map

| Claim | Evidence |
|---|---|
| No `pg_amcheck`, no `pg_verifybackup` in v12 | [bin/Makefile#SUBDIRS](../../../../raw/postgres-12/src/bin/Makefile#L16-L34) |
| amcheck is B-tree only | [amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4), [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323) |
| Checksum computed at flush, verified on read, not on hit | [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750), [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925), [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) |
| All-zero pages accepted | [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) |
| One-way, probabilistic heap probe | [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926), [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10) |
| WAL CRC and redo stop | [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725), [xlog.c#redo-loop-end](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7248-L7250) |
| Async commit loss | [xact.c#commit-flush](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1367-L1400) |
| Missing trailing segment invisible | [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750) |
| Sync replication off by default | [guc.c#synchronous_standby_names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4087-L4095), [syncrep.c#SyncStandbysDefined](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L92-L93) |

## Open Questions

- The claim that a torn page "normally" fails its checksum depends on the checksum's distribution over mixed old/new halves; the pinned source gives only the 16-bit output range, not a collision rate for that pattern.
- Several undetected cases (lost writes, consistent-but-old storage snapshots, failover loss, unwanted DML) are deductions from what the checks read; this page runs no measurement for them.
- The exact `pg_waldump` output on a CRC failure was not traced here.

## Source References

- [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1-L40)
- [amcheck.sgml](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L1-L40)
- [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162)
- [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925)
- [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214)
- [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725)
- [xact.c#commit-flush](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1367-L1400)
- [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750)
- [syncrep.c#fast-exit](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L158-L175)

## Navigation

- [v12/index](../../index.md)
- [Wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [How amcheck Checks B-Tree Indexes and Their Tables (unverified)](../indexing/amcheck-indexes-tables-and-data-checksums.md)
