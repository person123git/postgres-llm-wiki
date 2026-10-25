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
  - [Options and state changes](#options-and-state-changes)
  - [Detected data corruption](#detected-data-corruption)
  - [Undetected data corruption and data loss](#undetected-data-corruption-and-data-loss)
  - [Coverage comparison table](#coverage-comparison-table)
  - [Non-default settings that change the result](#non-default-settings-that-change-the-result)
  - [Residual risks and additional protection](#residual-risks-and-additional-protection)
  - [Causal summary](#causal-summary)
- [Measurement Script](#measurement-script)
  - [Purpose and invocation](#purpose-and-invocation)
  - [Fixture conditions](#fixture-conditions)
  - [Last run and results](#last-run-and-results)
  - [Published script](#published-script)
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

A successful check is not a proof of complete integrity. Coverage depends on what was read and which checks ran:

- [Data checksums](../../../glossary.md#data-checksums) are off by default. When on, ordinary buffer reads can detect damage to an initialized page on a cache miss. A buffer hit does not test the disk image; all-zero pages are accepted and checksum collisions are possible. A sane-header mismatch can be accepted with `ignore_checksum_failure`, and `zero_damaged_pages` can replace a rejected page with zeros; both default off. Offline and backup verification have separate scan/skip rules, described below. [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [checksum_impl.h#reduce](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214)
- amcheck detects violations of the B-tree invariants it tests on pages its walk reaches. With `heapallindexed = true` it also detects [heap](../../../glossary.md#heap) rows that lack a matching index entry, probabilistically. It never checks the reverse direction or other index types. It does not comprehensively validate heap attributes outside the indexed values, expressions and [partial-index](../../../glossary.md#partial-index) predicate. [verify_nbtree.c#bt_target_page_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10)

Both mechanisms only **detect**. Neither prevents nor repairs damage. The amcheck docs say a pass cannot prove that corruption is absent, and that there is no general repair method. [amcheck.sgml#cannot-prove-absence](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L316-L317) [amcheck.sgml#repair](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L321-L337)

Both are blind to whole classes of loss: correctly written but unwanted changes, lost commits, broken or missing [WAL](../../../glossary.md#wal), and missing files or segments that leave a consistent shorter relation. Those need backups, restore tests, WAL checks and application reconciliation. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614) [xlogreader.c](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725)

The [measurement below](#measurement-script) exercises selected boundaries; the remaining cases are source deductions. For the complete amcheck walkthrough, see [How amcheck Checks B-Tree Indexes and Their Tables (unverified)](../indexing/amcheck-indexes-tables-and-data-checksums.md). [Emptied-segment measurements (unverified)](../indexing/amcheck-emptied-segments-with-checksums.md) cover truncation; this page also tests removed files.

### Four kinds of failure

The question asks to separate four kinds of failure. This page uses these definitions:

| Kind | Meaning | Example | Can a page-level or index-level check see it? |
|---|---|---|---|
| Physical corruption | Bytes on disk differ from the bytes PostgreSQL wrote. | Bit flip, [torn page](../../../glossary.md#torn-page), misdirected write | Checksums can, on read. amcheck can if the damage breaks an invariant it tests. [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) |
| Structural corruption | Bytes are internally inconsistent with the format's rules. | Keys out of order, broken sibling links, bad [line pointers](../../../glossary.md#line-pointer) | amcheck, for B-trees. Checksums only if the damage happened after the checksum was computed. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) |
| Logical corruption | Every structure is valid, but the content is wrong. | Wrong value written by an application or a bug, an index entry that disagrees with its heap row | amcheck only for the index/heap disagreement it probes. Checksums never. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| Data loss | Data that was committed or existed is gone; the remaining structures may or may not be consistent. | Lost commit, deleted row, missing segment | Consistent loss can pass; broken remaining structures or referenced missing files can still error. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614) |

### Mental model

Checksums compare page bytes and block position with a stored checksum. amcheck compares index structure with its invariants and, optionally, heap-derived tuples with index fingerprints. Consistently wrong inputs can satisfy these comparisons; the checks have no independent record of intended application contents. [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926)

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| `pd_checksum` | 16-bit checksum in the page header | Computed from the buffer at flush time when checksums are on and the page is initialized. [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) | Stored on disk | Compared with a recomputed value when the page is next read from disk. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) |
| Block number | The page's position in the relation | Mixed into the checksum. [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) | Live, from the read request | Can expose a page written to the wrong block, subject to checksum collisions. No relation identifier is mixed in. |
| Cached page | The copy in [shared buffers](../../../glossary.md#shared_buffers) | A previous read or write | Stored in memory | A buffer hit returns it without verification. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) |
| B-tree invariants | Rules that must hold on index pages, such as key order | Checked by amcheck on each reached page. [verify_nbtree.c#bt_target_page_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) | Live, per call | One `ERROR` on the first violation. |
| Index fingerprint | A [Bloom filter](../../../glossary.md#bloom-filter) of non-`LP_DEAD` index leaf entries | Built during the walk when `heapallindexed` is true. [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083) | Live, per call | Probed for each heap row that belongs in the index. |
| WAL record CRC | CRC-32C over each WAL record | Computed at insert; checked by `ValidXLogRecord()` when WAL is read. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725) | Stored in WAL | A mismatch rejects that record; recovery determines whether to retry, stop or fail. Neither checksums nor amcheck read WAL. |

### Logic map

```text
A page is requested through the buffer manager
  +-- buffer hit -> use cached bytes; no disk verification                         [1]
  +-- cache miss -> read requested block
         +-- missing file / short read -> access error (see recovery/zero override) [2]
         +-- complete page -> initialized (pd_upper != 0)?
                +-- no -> all bytes zero? yes: accept; no: reject                   [3]
                +-- yes -> check header sanity; compute checksum if enabled
                       +-- mismatch -> WARNING                                     [3]
                       +-- sane header and (no mismatch OR ignore override)?
                              yes -> accept
                              no -> zero_damaged_pages on? zero + WARNING
                                    otherwise ERROR                                [4]

amcheck reads reached B-tree pages through that path
  -> check selected structural/order invariants                                    [5]
  -> rootdescend enabled? independent root searches for leaf entries                [6]
  -> heapallindexed enabled? fingerprint index; scan qualifying heap tuples
       -> absent from Bloom filter? ERROR; possibly present? continue               [7]
       -> erased heap tuples supply no probe; reached missing index files error

WAL completeness and intended application contents have no comparison here.         [8]
```

- **[1]** [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796)
- **[2]** [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1209-L1225). Short-read zero filling is a non-default `zero_damaged_pages` branch or the WAL-replaying process's `InRecovery` branch; it is not enabled merely by a client connecting to a hot standby. [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L189-L200) [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161)
- **[3]** [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162). Checksums default off; `initdb -k` or offline `pg_checksums --enable` turns them on. [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [pg_checksums.c](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L513-L521)
- **[4]** [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925). Both overrides default off and have superuser session scope. [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161)
- **[5]** [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L551) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870)
- **[6]** [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2059-L2108). `rootdescend` defaults false and requires a version-4 B-tree. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L205-L218) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L486-L492)
- **[7]** [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L625) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926). `heapallindexed` defaults false. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L180-L192)
- **[8]** [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) [xlogreader.c](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725)

### Options and state changes

`bt_index_check()` normally takes `AccessShareLock`; `bt_index_parent_check()` takes `ShareLock` and adds parent/child checks. Both optional booleans default to false. `rootdescend = true`, available on the parent checker, independently searches from the root for leaf tuples; it requires B-tree format version 4 and does not fetch the corresponding heap rows. Parent checking with `heapallindexed = true` also builds a second Bloom filter for missing-downlink checks. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L263) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L438-L492) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2059-L2108)

The main state is `BtreeCheckState`: it holds the relations, option flags, current target page and fingerprint filters. The wrapper validates and locks the relations; the tree walk reads through the buffer manager, then the optional table access-method build scan calls `bt_tuple_present_callback()`. That callback forms the expected index tuple and probes its fingerprint. [Snapshot](../../../glossary.md#snapshot) and partial-index rules decide which heap tuples supply probes. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L54-L110) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L223-L300) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L624) [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1306-L1315) [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1593-L1666)

| Event | State before → after | Effect on checking |
|---|---|---|
| External disk change while a good copy is cached | Disk differs; buffer still holds earlier bytes | A hit bypasses disk verification; a later uncached read can expose damage. [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) |
| Clean stop, file removal, new heap scan | Old file extent → shorter fresh block count | Lost heap tuples supply no probes; index links can still request missing blocks. [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362) |
| Restore an old index file after inserting a heap row | Old checksummed index; newer heap | Structural checks may pass; `heapallindexed` can find the absent entry. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| [REINDEX](../../../glossary.md#reindex) with intact heap and working index semantics | Damaged/outdated index → newly built index | Rebuild can repair index-only damage; it cannot reconstruct erased heap rows. [index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L3515-L3535) |
| Replay an applicable full-page image | Torn target page → WAL image | Recovery restores that image; checksum verification itself does not repair. [xlogutils.c](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L352-L386) |

### Detected data corruption

Every entry below is **conditional**. None is guaranteed.

1. **Disk-level bit flips and physical page corruption (physical).** With checksums on, a read from disk recomputes the checksum over the whole page and its block number. By default, a mismatch logs `page verification failed` and raises `invalid page in block`. Both overrides default off: `ignore_checksum_failure = on` accepts only a sane-header mismatch, while `zero_damaged_pages = on` zeroes a rejected page instead; see [settings](#non-default-settings-that-change-the-result). Without checksums, page-header sanity, amcheck invariants and incidental access errors still provide limited detection. [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925)
2. **Torn pages and partial writes (physical).** A torn page mixes old and new bytes. Checksums detect it on a checked read if the computed value differs, but do not repair it. Repair can come from WAL: with `full_page_writes` on (the default), recovery can restore an applicable [full-page image](../../../glossary.md#full-page-image) logged for the page, followed by subsequent redo. [xlogutils.c](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L352-L386) With checksums on, hint-bit-only changes also log an image under the conditions in the companion page. A short read normally errors; the [logic map](#logic-map) names the recovery and `zero_damaged_pages` exceptions. [guc.c#full_page_writes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1164-L1175) [xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192) [bufmgr.c#hint-wal-gates](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3443-L3510) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)
3. **Unexpected page modifications (physical).** An external change can fail verification on the next checked disk read. Moving a page to a different block changes the checksum input because the block number is mixed in; a collision can still hide the move. A valid page from another relation at the same block number passes. [checksum_impl.h#blkno](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L207-L208)
4. **B-tree index structural corruption (structural).** amcheck checks page sanity, line pointers, levels, high keys and sibling links on every page its walk reaches. `bt_index_parent_check` adds parent/child checks and needs a `ShareLock`, which blocks writes and fails on a [hot standby](../../../glossary.md#hot-standby). [verify_nbtree.c#page-sanity](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2396-L2471) [verify_nbtree.c#bt_downlink_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1508-L1647) [verify_nbtree.c#lock-choice](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L232-L263) [lock.c#recovery-lock-limit](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766)
5. **Index ordering and consistency problems (structural/logical).** Ordering checks compare items within a page, against the high key and across siblings. They can catch an index whose [operator class](../../../glossary.md#operator-class) or [collation](../../../glossary.md#collation) no longer sorts its keys the same way, such as after an operating-system collation change. [verify_nbtree.c#order-checks](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1130-L1258) [amcheck.sgml#collation](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L222-L250)
6. **Missing or inconsistent index entries (logical).** With `heapallindexed = true`, amcheck compares heap rows with the index: it forms the index entry each qualifying heap row should have and probes the fingerprint. "Definitely absent" raises `lacks matching index tuple` (`XX001`). The probe is a Bloom filter, so a missing entry can escape with small probability. [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) [amcheck.sgml#bloom-memory](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L195-L207)
7. **Heap page and tuple corruption (physical/structural).** amcheck has no heap checker at this pin. With checksums on, a completed `heapallindexed` scan visits the heap main fork within the block bound captured at scan start, so initialized pages fetched from disk are checksum-tested. Missing tails outside that bound are not visited. Without checksums, the scan can still encounter ordinary page/tuple-access errors, broken [HOT](../../../glossary.md#hot) chains, and problems evaluating or detoasting indexed values or predicates; it does not validate every heap field. [heapam.c#scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231) [heapam.c#read-page](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L378-L382) [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1666)
8. **Invalid tuple metadata (structural).** These amcheck functions have no dedicated validator for heap tuple headers such as [xmin and xmax](../../../glossary.md#xmin-and-xmax) or infomask bits. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L614) A broken HOT chain is found incidentally: the build scan raises `failed to find parent tuple for heap-only tuple`. Tuple metadata also affects visibility decisions made by the scan. [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1356-L1385) [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1580-L1601) That use does not constitute comprehensive header validation. Checksums can also detect changed header bytes on a checked disk read. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [heapam_handler.c#hot-root-error](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1634-L1652)
9. **[TOAST](../../../glossary.md#toast) data corruption (physical/structural).** During the heap membership scan, amcheck fetches TOAST values only when an indexed column, index expression or partial-index predicate needs them. [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1603-L1626) A missing required chunk can then raise the TOAST reader's error. Other external values are not exhaustively fetched by that scan. A separately checked TOAST-table B-tree still has the usual index/heap checking scope. The TOAST table's own pages get checksum tests only when something reads them. [indextuple.c#external-fetch](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88) [tuptoaster.c](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2007-L2017)
10. **Corruption caused by memory, storage or I/O failures.** Storage and I/O damage after the write is checksum territory. Memory damage **before** the write can get a valid checksum on an initialized page, because the checksum is computed from the buffer at flush time. amcheck can still catch memory damage that breaks an invariant or changes an indexed value; the docs say `heapallindexed` greatly increases the chance of catching single-bit memory errors. [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) [amcheck.sgml#ram](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L295-L313)

### Undetected data corruption and data loss

These cases can go undetected even when every routine check completes successfully.

1. **Silent loss of rows or records (data loss).** A zeroed heap page is accepted without a checksum test and contributes no rows. The `heapallindexed` probe asks only whether each found heap row is in the index, so index entries whose heap rows vanished are not reported. [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614)
2. **Logically incorrect but structurally valid data (logical).** A wrong value that normal SQL or a bug writes consistently to the heap and indexes can have a valid checksum and matching index entry. Nothing compares it with what it should have been. [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750)
3. **Accidental or unauthorized deletion or modification (data loss/logical).** If an unwanted committed `DELETE`, `UPDATE`, `TRUNCATE` or `DROP` leaves otherwise consistent objects, neither check knows the intended prior contents. This is a deduction from their page and tuple comparisons, not an audit of authorization. [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926)
4. **Missing data that leaves tables and indexes consistent (data loss).** A newly sized heap scan can stop before a removed trailing heap segment: `mdnblocks()` discovers a shorter relation, so no lost tuple supplies a membership probe. This does not apply to every missing index segment: amcheck follows stored tree links, and a reached missing file errors. Cached pages and already-open segment descriptors can change what a later check observes. [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L614) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362) [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1209-L1225)
5. **Stale pages containing valid checksums (physical, looks valid).** A lost write can leave the older page image on disk. That image carries its own valid checksum and correct block number, so checksums pass. amcheck can catch it if the stale page breaks an invariant or lacks an entry for a heap row that the scan finds. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214)
6. **Lost committed transactions (data loss).** `synchronous_commit` defaults to `on`. Setting it to `off` in the session permits the asynchronous commit branch to acknowledge a commit before WAL is flushed; a crash can lose those recent commits without causing inconsistency. Forced synchronous cases still flush. `fsync` also defaults to `on`, but disabling it in configuration (reload) removes durable synchronization and can cause unrecoverable corruption, not merely clean transaction loss. Neither checker proves that an acknowledged transaction survived. [xact.c](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1363-L1409) [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4354-L4361) [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1133) [fd.c](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L345-L355) [config.sgml](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2515-L2519) [config.sgml](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2567-L2584)
7. **WAL corruption or missing WAL segments (data loss or recovery failure).** WAL records have a separate CRC, checked by `ValidXLogRecord()`, outside page checksums and amcheck. A failed record read can end a redo pass, but that does not prove a usable database. Archive recovery refuses to finish before its required consistency point or backup end, and standby recovery can retry another source or wait. A successful recovery also does not prove it reached an operator’s intended endpoint. [xlogreader.c](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725) [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7248-L7250) [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7363-L7396) [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L11526-L11536) [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L11858-L11908)
8. **Missing database files or relations (data loss).** A missing first segment makes `mdopen()` raise `could not open file` when the relation is accessed. That is a storage-manager access error. amcheck also explicitly checks whether a B-tree main fork exists and can raise `index ... lacks a main relation fork` before reading it. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L281-L293) These functions check the supplied index; they do not compare the current relation inventory with an older expected inventory. [md.c#mdopen](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L450-L463)
9. **Incomplete index-type coverage (structural).** amcheck rejects GiST, GIN, hash, BRIN and SP-GiST indexes. These amcheck functions do not validate their structure. Ordinary access methods still have checks, such as hash page-type/special-area/metapage checks and GiST page sanity checks. [hashutil.c](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L219-L289) [gistutil.c](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L768-L799) Checksums still test eligible pages on disk reads. [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [check_btree.sql#brin-rejection](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L49-L53)
10. **Application-level data integrity violations (logical).** Business rules, cross-table invariants and constraints that were never declared are outside both mechanisms. amcheck does not even check B-tree uniqueness: it clears `ii_Unique` before its scan. [verify_nbtree.c#unique-off](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L599-L607)
11. **Storage failures producing internally consistent but incorrect states (physical, looks valid).** An otherwise internally consistent, older storage state can have valid checksums and consistent indexes. Neither check compares the data with any external reference. [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926)
12. **Replication-related data loss (data loss).** Synchronous replication is off by default: `synchronous_standby_names` is empty, so a commit does not wait for any standby. A failover to an asynchronous standby can then lose commits that the primary acknowledged. If the promoted standby is otherwise consistent, these checks do not reveal the missing primary-only commits. [guc.c#synchronous_standby_names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4087-L4095) [syncrep.c#SyncStandbysDefined](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L92-L93) [syncrep.h#SyncRepRequested](../../../../raw/postgres-12/src/include/replication/syncrep.h#L19-L20) [syncrep.c#fast-exit](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L158-L175)

### Coverage comparison table

"Detects" means "raises an error when the case is met". Statuses describe these mechanisms, not every error PostgreSQL can raise during ordinary access. Rows that say “detects” retain their stated conditions and the [override settings](#non-default-settings-that-change-the-result).

| Failure Scenario | Checksums | amcheck | Detection Status | Limitations |
|---|---|---|---|---|
| Bit flip on an initialized page on disk | Detects on disk read, if checksums are on | Detects only if an invariant or indexed value breaks | Conditional | Cached, unread and all-zero pages untested; Collision possible; probability requires a damage model; detects only, no repair [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| Torn page / partial write | Detects on read | Possibly, through invariants | Conditional | Repair needs `full_page_writes` (default on) and replay; truncated file errors on short read [xlogutils.c](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L352-L386) [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) |
| Page at the wrong block of the same relation | Detects on read | Possibly | Conditional | Same-block page from another relation passes [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) |
| B-tree structural damage | No, unless the page was also damaged on disk | Detects on reached pages | Conditional | Unreached pages unchecked; parent checks need `bt_index_parent_check` and a `ShareLock`; first error stops the call [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L232-L263) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) |
| B-tree ordering / collation drift | No: pages are validly written | Detects | Conditional | Only B-trees; standby collation can differ from the primary [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1130-L1258) [amcheck.sgml](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L222-L250) |
| Heap row lacking an index entry | No | Detects with `heapallindexed` | Conditional, probabilistic | Bloom filter can miss; partial-index and snapshot rules skip rows [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1593-L1666) |
| Index entry whose heap row is gone | No | No | Unavailable | One-way probe only [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614) |
| Heap page bytes damaged on disk | Detects on read; amcheck's heap scan supplies the read | Incidental access errors and indexed-value comparisons | Conditional | Needs checksums on and `heapallindexed` to read the heap [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1593-L1666) |
| Invalid tuple header metadata | Only if damaged on disk | Incidental errors, including broken HOT chains | Conditional / mostly unavailable | No heap checker in v12 [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1634-L1652) [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) |
| TOAST damage | Only for TOAST pages that are read | Only for values the index needs | Conditional | Parent-table probe does not fetch all external values; a TOAST-table index can be checked separately [indextuple.c](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88) [tuptoaster.c](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2007-L2017) |
| Memory damage before the write | No for initialized bytes checksummed after the damage | Possibly, via invariants and probe | Conditional | Non-indexed values unchecked [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| Zeroed heap page | No: accepted without test | No: rows simply vanish | Unavailable | Silent row loss [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614) |
| Stale page with valid checksum (lost write) | No | Possibly | Conditional | Valid checksum does not imply valid index/heap relationships [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| Valid but wrong data; unwanted DML | No | No | Unavailable | Logically wrong, structurally valid [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| Lost committed transaction | No | No | Unavailable | Checks do not establish durability or acknowledged-transaction completeness [xact.c](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1363-L1409) |
| WAL corruption or missing WAL | No (WAL has its own CRC) | No | Unavailable to both | Recovery can retry, fail, or end; startup is not guaranteed [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7363-L7396) [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L11526-L11536) |
| Missing trailing segment | No checksum for absent bytes | Heap loss can pass; reached index file can error | Conditional | Fresh heap size and index block references differ [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362) |
| Missing first segment / dropped relation | No | Error only if amcheck opens it; dropped relation not seen | Conditional / unavailable | Storage access errors or explicit main-fork refusal; depends on what is opened [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L450-L463) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L281-L293) |
| Non-B-tree index damage | On disk reads only | No | Unavailable through amcheck | Ordinary access methods can still detect structural faults [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323) [hashutil.c](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L219-L289) |
| Application integrity violation; duplicate keys | No | No (uniqueness not checked) | Unavailable | Needs constraints or reconciliation [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L599-L614) |
| Consistent but outdated storage state | No | No | Unavailable | Needs an external reference [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| Replication loss on failover | No | No | Unavailable | Async by default [syncrep.c](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L158-L185) |

**Worked calculation (not measured):** the checksum returns one of 65,535 values. Only if a damage model makes the recomputed value uniformly distributed over that range is the match probability `1 / 65,535 ≈ 0.001526%`. This assumption is not established for a single-bit flip, a moved block or a torn write. [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214)

### Non-default settings that change the result

| Setting | Default | Cause and mechanism | Effect |
|---|---|---|---|
| Data checksums | Off: `initdb` sets them only with `-k`, or `pg_checksums --enable` on a stopped cluster. [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [pg_checksums.c#shutdown-required](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L513-L521) | `DataChecksumsEnabled()` reads the control file and gates the checksum computation and test. [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826) | Every checksum row above assumes this non-default state. |
| `heapallindexed` | False when omitted. [verify_nbtree.c#bt_index_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L180-L192) | Only this option builds the membership fingerprint and requests the heap scan. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L419-L432) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L625) | Without it there is no heap verification scan. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L625) |
| `ignore_checksum_failure`, `zero_damaged_pages` | Both off. [guc.c#ignore_checksum_failure](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161) | A superuser can `SET` them (`superuser` context, session scope). `PageIsVerified()` accepts a checksum mismatch only when `ignore_checksum_failure` is on AND the header is sane. `ReadBuffer_common()` instead zeroes a rejected page when `zero_damaged_pages` is on. [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L149-L162) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L908-L918) | A checksum failure stops being an error; with `zero_damaged_pages`, rows on the zeroed heap page disappear and a warning is emitted. Short reads also return zeros under that option or in the WAL-replaying process. [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) |
| `synchronous_commit` | `on` (`remote_flush`). [xact.h:79](../../../../raw/postgres-12/src/include/access/xact.h#L79) [guc.c#synchronous_commit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4354-L4361) | `off` permits the asynchronous branch unless the command forces a synchronous flush. [xact.c](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1363-L1409) Context `user`: `SET` in the session. | Recent commits can be lost in a crash. |
| `fsync`, `full_page_writes` | Both on. [guc.c#fsync](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1133) [guc.c#full_page_writes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1164-L1175) | Context `sighup`: changed in the configuration file and applied on reload. | Disabling `fsync` risks unrecoverable corruption; disabling `full_page_writes` removes its usual torn-page protection (forced-image paths still exist); checksums still only detect. |
| `synchronous_standby_names` | Empty. [guc.c#synchronous_standby_names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4087-L4095) | Context `sighup`. When synchronous replication is requested, `SyncRepWaitForLSN()` tests whether standbys are configured and whether the required position is already acknowledged; only outstanding waits are queued. [syncrep.c](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L142-L190) [syncrep.h#SyncRepRequested](../../../../raw/postgres-12/src/include/replication/syncrep.h#L19-L20) | Reduces failover loss; does not detect corruption. |

### Residual risks and additional protection

After both checks succeed, these risks remain: every **Unavailable** row above, plus missed cases in the **Conditional** rows. The pinned tree offers these additional tools. Each also has limits.

| Need | Mechanism in v12 | What it adds | Limit |
|---|---|---|---|
| Scan eligible files beyond query/tree reach | Offline `pg_checksums --check`; checksum verification during [base backup](../../../glossary.md#base-backup) | Both read selected on-disk files, independently of shared buffers. [pg_checksums.c](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L170-L225) [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1413-L1430) | `pg_checksums` excludes its skip list and temporary-directory/file prefixes; `-f` can further narrow it. Backup excludes temporary relations and non-init [forks](../../../glossary.md#fork) of [unlogged relations](../../../glossary.md#unlogged-relation), but keeps init forks. Both skip `pd_upper == 0`; backup additionally skips pages at/after its start LSN and disables verification for the remainder of a file after an unaligned read. [pg_checksums.c](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L95-L110) [pg_checksums.c](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L301-L352) [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1162) [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1454-L1556) |
| Backup verification | No backup manifest or `pg_verifybackup` exists at this pin (absent from `src/bin`). [bin/Makefile#SUBDIRS](../../../../raw/postgres-12/src/bin/Makefile#L16-L34) | — | Restore tests add evidence of recoverability, not a proof of every logical value or intended recovery endpoint. |
| Restore testing | Restore a base backup, replay archived WAL, then run amcheck and application checks on the copy. | Exercises WAL availability and CRCs during replay. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725) | Startup can fail for missing required WAL. If it succeeds, still compare the recovered endpoint with the intended endpoint; `redo done at` alone proves neither completeness nor application correctness. [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7363-L7396) [xlog.c#redo-done](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7297-L7299) |
| WAL validation | `pg_waldump` decodes WAL through `XLogReadRecord()`, the reader that runs the CRC check. [pg_waldump.c:1134](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L1134) | Finds damaged or missing archived segments before a restore needs them. | Shows only that records are readable, not that the archive is complete for your recovery target. |
| Prevent lost commits | Keep `fsync`, `full_page_writes` and `synchronous_commit` at their defaults; configure `synchronous_standby_names` for failover. | Supports durability and replication acknowledgements; does not detect corruption. | Async standbys still lag. [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1133) [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1164-L1175) [xact.c](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1363-L1409) [syncrep.c](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L158-L190) |
| Logical correctness | Declared constraints, plus application reconciliation such as row counts and totals compared with an external system of record. | Checks intended business relationships and reconciles contents against an independent reference. | Must be designed per application. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L599-L614) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| Non-B-tree indexes | `REINDEX` rebuilds an index from the heap. | Can repair index-only damage when the heap and index semantics are sound. [index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L3515-L3535) | A rebuild is not an integrity certificate and can itself encounter errors; it may not fix the underlying cause. [amcheck.sgml#repair](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L321-L337) |

### Causal summary

Initialized page written with checksums enabled → stored checksum → later checked disk read → mismatch detected unless an override changes handling. Cached pages, all-zero pages and collisions remain gaps. [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162) [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214)

B-tree walk → structural checks on reached pages → optional root searches and heap membership probes. A valid old checksum does not stop these checks finding a stale index/heap mismatch. A shortened heap scan can hide lost rows, while a tree link to a missing index file can error. Neither mechanism audits acknowledged transactions or WAL completeness. Repairs depend on the defect: index rebuild, applicable WAL images, or recovery from a suitable backup are distinct possibilities. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L624) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2059-L2108) [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362) [index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L3515-L3535) [xlogutils.c](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L352-L386)

## Measurement Script

### Purpose and invocation

This script tests removed heap/index tails, stale index membership, checksum overrides, ordinary hash-page validation, `REINDEX`, unlogged backup exclusions and a backup missing required WAL. It creates only disposable objects. It does not simulate power loss or claim a failure probability.

From the repository root:

```sh
mkdir -p .wiki-runtime/tmp
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/storage-and-vacuum/checksums-and-amcheck-detection-coverage.md | sed '1d;$d' > .wiki-runtime/tmp/cov12-run.sh
bash -n .wiki-runtime/tmp/cov12-run.sh
bash .wiki-runtime/tmp/cov12-run.sh
```

| Item | Contract |
|---|---|
| Stages | Default: `build verify cleanup`. `build` reuses the owned installation; `verify` first stops owned servers and recreates only this script's data, backups and outputs; `stop` checks teardown without deleting results; `cleanup` stops and deletes the entire sandbox. |
| Re-run | `bash .wiki-runtime/tmp/cov12-run.sh build verify` retains the build and results but stops every server. Inspect results, then run `bash .wiki-runtime/tmp/cov12-run.sh cleanup`. |
| Environment | Run from the repository root (`PWD`). Inputs: `JOBS=8`, `PORT12=55449`; host `PATH` supplies the build toolchain and POSIX utilities. Script sets `TMPDIR` inside the sandbox, `PGHOST`, `PGPORT`, `PGDATABASE=postgres`, `PGOPTIONS`, `LC_ALL=C` and `TZ=UTC`, and clears `PGUSER`, `PGSERVICE`, `PGSERVICEFILE`. No external PostgreSQL service is used. |
| Prerequisites | Bash, Git, make, a C compiler, the pinned tree's Git-build prerequisites and zlib. Local shared-memory/socket permission is needed. Configure: `--without-readline --with-zlib`, `CFLAGS='-O2 -g -std=gnu99'`; installs amcheck. |
| Output | `.wiki-runtime/tmp/cov12/out/`: read `platform.txt`, `results.txt`, then individual probe files. Expected errors are asserted. Full logs remain only until cleanup. |
| Runtime | Approximately one minute including a fresh build on this host; verification with the build present takes several seconds. |
| Cleanup | The EXIT/INT/TERM paths stop owned servers, including failed/interrupted startup attempts. `cleanup` also deletes `.wiki-runtime/tmp/cov12/`. Remove the extracted runner afterwards. Process-list inspection must succeed before teardown is accepted. No service or sandbox is retained in the filed run. |

### Fixture conditions

| Setting or state | Default | Script cause, mechanism and scope |
|---|---|---|
| Relation segments | 131,072 blocks (1 GB at 8 kB) | Generated `pg_config.h` uses 128 blocks (1 MB) so the tail-removal fixtures stay small. All reported segment/block positions depend on this build change. Size discovery and block routing use `RELSEG_SIZE`; the default build needs larger files to reach the same branches. [configure.in](../../../../raw/postgres-12/configure.in#L279-L308) [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750) [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1100-L1127) |
| Data checksums | Off | `initdb -k` enables them; no checksums-off leg was run. `DataChecksumsEnabled()` gates page verification. [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826) [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L103) |
| `ignore_checksum_failure`, `zero_damaged_pages` | Both off | Separate session `SET`s, superuser scope, test acceptance of a sane-header mismatch and replacement of a rejected heap page. Their default leg errors. The checksum is damaged again after stopping the accepting leg because hint-bit writes may refresh it. [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161) [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L149-L162) [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925) [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) |
| `autovacuum` | On | Off in the private configuration, reload scope, to keep background workers away from the damaged fixtures. [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1424-L1434) |
| Socket/port | `localhost`, compiled socket directory, 5432 | `listen_addresses=''`, private socket directory and port 55449; all restart settings. The fixtures run sequentially on that private socket. [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936-L3963) [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177-L2188) |
| Timeouts | Both 0 | `PGOPTIONS` sets `statement_timeout=10min`, `lock_timeout=5s`, user/session scope, to bound SQL waits. [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2401) |
| Scan enables | On | Session scope: disable index/bitmap/index-only scans for the shortened-heap count; disable sequential/bitmap scans for the hash lookup. The sequential/index/bitmap switches add cost penalties; the index-only switch removes eligibility. The error and count probes exercise the intended fixture paths; the cost switches are not universal plan-forcing guarantees. [costsize.c](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L964-L971) [indxpath.c](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L1905-L1908) [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L884-L922) [costsize.c](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L225-L233) [costsize.c](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L525-L535) |
| Backup and unlogged fixture | Ordinary tables are permanent; basebackup defaults to streaming WAL and a spread checkpoint | Explicit `UNLOGGED` creates an init fork. `pg_basebackup -X fetch --checkpoint=fast` requests WAL collection at backup end and an immediate checkpoint, so the small fixture completes promptly. [pg_basebackup.c](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L89-L100) [pg_basebackup.c](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L1870-L1878) [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L10330-L10338) One untouched copy starts; another has its copied WAL files removed before startup. [gram.y](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283) [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624) [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1162) [pg_basebackup.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/pg_basebackup.sgml#L290-L365) |
| File damage | No external modifications | While stopped: remove the final heap/index segment, restore an older index file, zero the two-byte checksum field at offset 8, and zero the hash metapage. Clean restarts remove cached pages/descriptors. The stale fixture intentionally leaves a newer heap and an older index. [bufpage.h#PageXLogRecPtr](../../../../raw/postgres-12/src/include/storage/bufpage.h#L90-L98) [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L162) [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214) [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750) [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) |

The locale and encoding are explicitly C/UTF-8 for reproducible text comparisons; without these options `initdb` derives locale and encoding from its environment. [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2300-L2329) [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2555-L2576)

`fsync`, `full_page_writes` and `synchronous_commit` stay at their defaults (`on`). The script records `shared_buffers`, which `initdb` selects for the host rather than assuming the compiled boot value; restarts separate disk-damage probes from cached state. [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1133) [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1164-L1175) [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4354-L4361) [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1018-L1102)

### Last run and results

Last successful run: **2026-10-25T20:02:16Z**, PostgreSQL **12.2**, pin `45b88269a353ad93744772791feb6d01bc7e1e42`, Darwin 27.0.0 arm64, Apple clang 21.0.0. Block size 8192, maximum alignment 8, 128 blocks per relation segment, C locale, UTF-8, UTC, `shared_buffers=128MB`, data checksums on. The 17 recorded outcome assertions passed. The published script is identical to the executed script. All three data directories were stopped; cleanup removed the build, clusters and results.

| Fixture | Observed result | Source explanation |
|---|---|---|
| Intact B-tree | Parent check with both options passed | [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L205-L218) |
| Last heap segment removed | `.6` removed; count fell from 16,000 to 14,592; parent check with both options passed | Fresh size omits erased rows; no reverse index-to-heap probe. [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614) |
| Last index segment removed | `.3` removed; parent check with both options failed `58P01`, target block 395 | Tree references requested a missing file. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362) [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1209-L1225) |
| Old index restored after inserting heap row 101 | Targeted offline checksum check passed with 0 bad blocks; parent structural/rootdescend check passed; adding `heapallindexed` failed `XX001` at heap TID `(0,101)` | Old valid checksum does not ensure heap membership. [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1926) |
| Rebuild stale index | `REINDEX` followed by parent check with both options passed | Rebuild takes its input from the current heap. [index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L3515-L3535) |
| Only `pd_checksum` set to zero on a heap page | Full offline scan found 1 bad block; default amcheck heap scan failed `XX001` | Computed checksums cannot be zero; mismatch is rejected by default. [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214) [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L162) |
| `ignore_checksum_failure=on`, same damage | Count 100; warning `01000`; no error | Sane header permits acceptance. The warning's code is the default because the source omits `errcode()`. [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L149-L158) [elog.c](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L381-L388) |
| `zero_damaged_pages=on`, checksum damaged again after restart | Count 0; checksum warning and `XX001` zeroing warning; no error | Rejecting read is replaced with a zero page. [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925) |
| Hash metapage overwritten with zeros | amcheck rejected hash target (`0A000`); ordinary hash lookup failed `XX002` | All-zero allowance does not bypass the hash access method's sanity check. [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323) [hashutil.c](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L219-L241) |
| Unlogged relation in backup | Init fork present; live main fork absent | Backup retains init forks and excludes other unlogged forks. [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1162) |
| Backup restore | Intact backup started and returned 101 rows from `stale`; copy with required WAL removed refused startup with `FATAL: could not locate required checkpoint record` | Required checkpoint lookup fails before redo can produce a usable restored database. [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6407-L6434) |

The heap fixture lost `16,000 - 14,592 = 1,408` rows from the scan. That subtraction describes this fixture, not a general amount of loss per segment. All error codes here are [SQLSTATEs](../../../glossary.md#sqlstate); a passing probe means only that the stated call completed successfully.

### Published script

```bash
#!/usr/bin/env bash
# Disposable fixtures only. Run from the repository root.
set -uo pipefail
ROOT="$PWD"
SRC="$ROOT/raw/postgres-12"
BOX="$ROOT/.wiki-runtime/tmp/cov12"
PIN=45b88269a353ad93744772791feb6d01bc7e1e42
JOBS="${JOBS:-8}"
PORT12="${PORT12:-55449}"
BUILD="$BOX/build"
BIN="$BOX/install/bin"
DATA="$BOX/data"
RESTORE="$BOX/restore"
BAD="$BOX/bad"
OUT="$BOX/out"
SOCK="$BOX/sock"
export TMPDIR="$BOX/tmp"
export PGHOST="$SOCK" PGPORT="$PORT12" PGDATABASE=postgres
export PGOPTIONS='-c statement_timeout=10min -c lock_timeout=5s'
export LC_ALL=C TZ=UTC
unset PGUSER PGSERVICE PGSERVICEFILE
fail() { printf 'FAIL: %s\n' "$*" >&2; exit 1; }
owned() {
  [ -f "$BOX/owner" ] && [ "$(cat "$BOX/owner")" = "$ROOT $PIN" ] || fail 'unowned sandbox'
}
sql() { "$BIN/psql" -X -Atq -v ON_ERROR_STOP=1 -v VERBOSITY=verbose -c "$1"; }
check() { "$@" || fail "assertion: $*"; }
start() {
  owned
  [ ! -e "$1/postmaster.pid" ] || fail "existing PID file: $1"
  # Record ownership BEFORE pg_ctl; an interrupted startup still gets stopped.
  : > "$1/.cov-started"
  "$BIN/pg_ctl" -D "$1" -l "$2" -w -t 30 start > "$OUT/start.txt" 2>&1
}
stop_one() {
  local process_list
  [ -d "$1" ] || return 0
  if [ -e "$1/postmaster.pid" ]; then
    [ -f "$1/.cov-started" ] || fail "refusing to stop unowned server: $1"
    "$BIN/pg_ctl" -D "$1" -m fast -w -t 30 stop > /dev/null || fail "stop: $1"
  fi
  [ ! -e "$1/postmaster.pid" ] || fail "PID survived: $1"
  process_list=$(ps -eo args=) || fail 'cannot inspect processes; teardown unconfirmed'
  if printf '%s\n' "$process_list" | grep -F "$1" | grep -v 'grep' > /dev/null; then
    fail "process still names $1"
  fi
}
stop_all() {
  [ -d "$BOX" ] || return 0
  owned
  stop_one "$DATA"
  stop_one "$RESTORE"
  stop_one "$BAD"
  [ ! -d "$SOCK" ] || [ -z "$(ls -A "$SOCK")" ] || fail 'socket not free'
  if [ -x "$BIN/pg_isready" ] && "$BIN/pg_isready" -q -h "$SOCK" -p "$PORT12"; then
    fail 'private port still answers'
  fi
}
on_exit() { local rc=$?; trap - EXIT; stop_all; exit "$rc"; }
trap on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
build() {
  check test "$(git --no-optional-locks -C "$SRC" rev-parse HEAD)" = "$PIN"
  if [ -e "$BOX" ]; then owned; else
    check mkdir -p "$BOX"
    printf '%s %s\n' "$ROOT" "$PIN" > "$BOX/owner"
  fi
  check mkdir -p "$BUILD" "$OUT" "$TMPDIR"
  [ ! -x "$BIN/postgres" ] || return 0
  (cd "$BUILD" && "$SRC/configure" --prefix="$BOX/install" --without-readline \
    --with-zlib CFLAGS='-O2 -g -std=gnu99' > configure.log 2>&1) || fail configure
  # Shorten relation segments from 1 GB to 1 MB; raw checkout stays read-only.
  sed 's/^#define RELSEG_SIZE .*/#define RELSEG_SIZE 128/' \
    "$BUILD/src/include/pg_config.h" > "$BUILD/src/include/pg_config.h.new"
  check mv "$BUILD/src/include/pg_config.h.new" "$BUILD/src/include/pg_config.h"
  check grep -q '^#define RELSEG_SIZE 128$' "$BUILD/src/include/pg_config.h"
  (cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 \
    && make -C contrib/amcheck install > amcheck.log 2>&1) || fail build
}
probe() {
  local name="$1" expected="$2" pattern="$3" query="$4" rc
  sql "$query" > "$OUT/$name.txt" 2>&1
  rc=$?
  if [ "$expected" = ok ]; then check test "$rc" -eq 0; else check test "$rc" -ne 0; fi
  check grep -E -q "$pattern" "$OUT/$name.txt"
  printf '%s|%s\n' "$name" "$expected" >> "$OUT/results.txt"
}
path_of() { sql "SELECT /* wiki_cov12 */ pg_relation_filepath('$1')"; }
verify() {
  owned
  check test "$(git --no-optional-locks -C "$SRC" rev-parse HEAD)" = "$PIN"
  stop_all
  # A verification run recreates only this script's disposable data and output.
  check rm -rf "$DATA" "$RESTORE" "$BAD" "$OUT" "$SOCK"
  check mkdir -p "$OUT" "$SOCK"
  check "$BIN/initdb" -k -D "$DATA" -A trust --locale=C --encoding=UTF8 > "$OUT/initdb.txt" 2>&1
  {
    printf "listen_addresses = ''\nunix_socket_directories = '%s'\nport = %s\n" "$SOCK" "$PORT12"
    printf 'autovacuum = off\n'
  } >> "$DATA/postgresql.conf"
  start "$DATA" "$OUT/server.log" || fail startup
  {
    date -u '+%Y-%m-%dT%H:%M:%SZ'
    uname -srm
    "$BIN/postgres" --version
    "$BIN/pg_config" --configure
    "$BIN/pg_controldata" "$DATA" | grep -E '^(Maximum data alignment|Database block size|Blocks per segment of large relation|Data page checksum version):'
    for setting in shared_buffers data_checksums fsync full_page_writes synchronous_commit; do
      printf '%s=%s\n' "$setting" "$(sql "SHOW /* wiki_cov12 */ $setting")"
    done
  } > "$OUT/platform.txt"
  check sql 'CREATE /* wiki_cov12 */ EXTENSION amcheck'
  local t hpath ipath spath cpath xpath upath hlast ilast original remaining
  for t in hmissing imissing; do
    check sql "CREATE /* wiki_cov12 */ TABLE $t (id int, k text, pad text)"
    check sql "INSERT /* wiki_cov12 */ INTO $t SELECT g, lpad(g::text,180,'0'), repeat('x',200) FROM generate_series(1,16000) g"
    check sql "CREATE /* wiki_cov12 */ INDEX ${t}_k ON $t(k)"
    check sql "VACUUM /* wiki_cov12 */ ANALYZE $t"
  done
  check sql 'CREATE /* wiki_cov12 */ TABLE stale (id int); INSERT /* wiki_cov12 */ INTO stale SELECT generate_series(1,100); CREATE /* wiki_cov12 */ INDEX stale_i ON stale(id)'
  check sql 'CREATE /* wiki_cov12 */ TABLE csum (id int); INSERT /* wiki_cov12 */ INTO csum SELECT generate_series(1,100); CREATE /* wiki_cov12 */ INDEX csum_i ON csum(id)'
  check sql 'CREATE /* wiki_cov12 */ TABLE hh (id int); INSERT /* wiki_cov12 */ INTO hh SELECT generate_series(1,100); CREATE /* wiki_cov12 */ INDEX hh_i ON hh USING hash(id)'
  check sql 'CREATE /* wiki_cov12 */ UNLOGGED TABLE uu (id int); INSERT /* wiki_cov12 */ INTO uu VALUES (1)'
  hpath=$(path_of hmissing); ipath=$(path_of imissing_k); spath=$(path_of stale_i)
  cpath=$(path_of csum); xpath=$(path_of hh_i); upath=$(path_of uu)
  hlast=$(sql "SELECT /* wiki_cov12 */ (pg_relation_size('hmissing')/8192-1)/128")
  ilast=$(sql "SELECT /* wiki_cov12 */ (pg_relation_size('imissing_k')/8192-1)/128")
  check test "$hlast" -gt 0
  check test "$ilast" -gt 0
  printf 'heap_last_segment=%s\nindex_last_segment=%s\n' "$hlast" "$ilast" > "$OUT/fixtures.txt"
  probe baseline ok '^$' "SELECT /* wiki_cov12 */ bt_index_parent_check('stale_i',true,true)"
  stop_one "$DATA"
  check cp "$DATA/$spath" "$OUT/stale-index.saved"
  start "$DATA" "$OUT/server.log" || fail startup
  check sql 'INSERT /* wiki_cov12 */ INTO stale VALUES (101)'
  # Back up an intact database first. FETCH supplies the required WAL.
  check "$BIN/pg_basebackup" -D "$RESTORE" -X fetch --checkpoint=fast > "$OUT/backup.txt" 2>&1
  check test -f "$RESTORE/${upath}_init"
  check test ! -f "$RESTORE/$upath"
  printf 'backup_unlogged_init_only|ok\n' >> "$OUT/results.txt"
  check cp -R "$RESTORE" "$BAD"
  stop_one "$DATA"
  start "$RESTORE" "$OUT/restore.log" || fail 'intact backup restore'
  probe restore_good ok '^101$' 'SELECT /* wiki_cov12 */ count(*) FROM stale'
  stop_one "$RESTORE"
  # Remove WAL only from a private, never-started backup copy.
  check rm -f "$BAD"/pg_wal/0*
  if start "$BAD" "$OUT/restore-bad.log"; then fail 'backup without required WAL unexpectedly started'; fi
  check grep -q 'FATAL:  could not locate required checkpoint record' "$OUT/restore-bad.log"
  printf 'restore_missing_wal|startup_rejected\n' >> "$OUT/results.txt"
  stop_one "$BAD"
  # All damage is applied after clean shutdown, with no old buffers or FDs.
  check rm "$DATA/$hpath.$hlast" "$DATA/$ipath.$ilast"
  check cp "$OUT/stale-index.saved" "$DATA/$spath"
  check "$BIN/pg_checksums" --check -D "$DATA" -f "${spath##*/}" > "$OUT/stale-checksums.txt" 2>&1
  check grep -E -q 'Bad checksums: *0$' "$OUT/stale-checksums.txt"
  printf 'stale_index_checksums|ok\n' >> "$OUT/results.txt"
  # pd_checksum is the two-byte field after the 8-byte LSN. Zero cannot be a valid checksum.
  check dd if=/dev/zero of="$DATA/$cpath" bs=1 seek=8 count=2 conv=notrunc 2> "$OUT/dd-checksum.txt"
  check dd if=/dev/zero of="$DATA/$xpath" bs=8192 count=1 conv=notrunc 2> "$OUT/dd-hash.txt"
  "$BIN/pg_checksums" --check -D "$DATA" > "$OUT/checksums.txt" 2>&1
  check test "$?" -ne 0
  check grep -E -q 'Bad checksums: *1$' "$OUT/checksums.txt"
  printf 'offline_checksum_bad_blocks|1\n' >> "$OUT/results.txt"
  start "$DATA" "$OUT/server.log" || fail startup
  probe missing_heap ok '^$' "SELECT /* wiki_cov12 */ bt_index_parent_check('hmissing_k',true,true)"
  remaining=$(sql 'SET /* wiki_cov12 */ enable_indexscan=off; SET /* wiki_cov12 */ enable_bitmapscan=off; SET /* wiki_cov12 */ enable_indexonlyscan=off; SELECT /* wiki_cov12 */ count(*) FROM hmissing')
  check test "$remaining" -lt 16000
  printf 'missing_heap_count|%s\n' "$remaining" >> "$OUT/results.txt"
  probe missing_index error 'could not open file' "SELECT /* wiki_cov12 */ bt_index_parent_check('imissing_k',true,true)"
  probe stale_structure ok '^$' "SELECT /* wiki_cov12 */ bt_index_parent_check('stale_i',false,true)"
  probe stale_membership error 'XX001:.*lacks matching index tuple' "SELECT /* wiki_cov12 */ bt_index_parent_check('stale_i',true,true)"
  check sql 'REINDEX /* wiki_cov12 */ INDEX stale_i'
  probe stale_repaired ok '^$' "SELECT /* wiki_cov12 */ bt_index_parent_check('stale_i',true,true)"
  probe checksum_default error 'XX001:.*invalid page in block' "SELECT /* wiki_cov12 */ bt_index_check('csum_i',true)"
  probe checksum_ignore ok '^100$' 'SET /* wiki_cov12 */ ignore_checksum_failure=on; SELECT /* wiki_cov12 */ count(*) FROM csum'
  check grep -q 'page verification failed' "$OUT/checksum_ignore.txt"
  # Evict the accepted page before testing the second override.
  stop_one "$DATA"
  # The accepting read can set hints and a shutdown write can refresh its checksum.
  check dd if=/dev/zero of="$DATA/$cpath" bs=1 seek=8 count=2 conv=notrunc 2> "$OUT/dd-checksum-again.txt"
  start "$DATA" "$OUT/server.log" || fail startup
  probe checksum_zero ok '^0$' 'SET /* wiki_cov12 */ zero_damaged_pages=on; SELECT /* wiki_cov12 */ count(*) FROM csum'
  check grep -q 'zeroing out page' "$OUT/checksum_zero.txt"
  probe hash_amcheck error '0A000:.*only B-Tree' "SELECT /* wiki_cov12 */ bt_index_check('hh_i')"
  probe hash_access error 'XX002:.*unexpected zero page' 'SET /* wiki_cov12 */ enable_seqscan=off; SET /* wiki_cov12 */ enable_bitmapscan=off; SELECT /* wiki_cov12 */ id FROM hh WHERE id=1'
  stop_all
  cat "$OUT/platform.txt" "$OUT/results.txt"
}
cleanup() { stop_all; [ ! -d "$BOX" ] || { owned; check rm -rf "$BOX"; }; }
[ $# -gt 0 ] || set -- build verify cleanup
for stage in "$@"; do
  case "$stage" in build) build;; verify) verify;; stop) stop_all;; cleanup) cleanup;; *) fail "unknown stage: $stage";; esac
done
```

## Context Reviewed

- `contrib/amcheck/` (Makefile, `verify_nbtree.c`, tests) and `doc/src/sgml/amcheck.sgml`.
- Checksum path: `bufpage.c`, `bufmgr.c`, `checksum_impl.h`, `xlog.c#DataChecksumsEnabled`, `initdb.c`, `pg_checksums.c`, `basebackup.c`.
- Storage manager: `md.c` (`mdopen`, `mdnblocks`, short reads).
- WAL and commit: `xlogreader.c#ValidXLogRecord`, `xlog.c` redo loop, `xact.c#RecordTransactionCommit`, `guc.c` durability settings.
- Replication: `syncrep.c`, `syncrep.h`.
- Adjacent repair and validation paths: `fd.c`, `xlogutils.c`, `catalog/index.c`, hash `_hash_checkpage()` and GiST `gistcheckpage()`.
- `src/bin/Makefile` for the absence of `pg_amcheck` and `pg_verifybackup`.
- Related pages: the v12 amcheck, emptied-segment and zeroed-segment pages, plus the schema-empty-files durability discussion. No v12 common concept pages exist.

The pinned regression test exercises successful basic/heap checks, permission failures and non-B-tree rejection; it is not evidence of the failure scenarios measured here. [check_btree.sql](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L22-L60) The module builds `verify_nbtree.c`; the file includes the table-AM and B-tree headers used at the call boundary. The measurement changes only the generated build header's `RELSEG_SIZE`, not the raw checkout. [Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L16) [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L23-L40) [pg_config.h.in](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864)

## Evidence Map

| Claim | Evidence |
|---|---|
| No `pg_amcheck`, no `pg_verifybackup` in v12 | [bin/Makefile#SUBDIRS](../../../../raw/postgres-12/src/bin/Makefile#L16-L34) |
| amcheck is B-tree only | [amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4), [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323) |
| Checksum computed at flush, verified on read, not on hit | [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750), [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925), [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) |
| All-zero pages accepted | [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) |
| One-way, probabilistic heap probe | [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926), [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10) |
| WAL CRC rejection; recovery may retry or refuse startup | [xlogreader.c](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725), [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7363-L7396), [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L11526-L11536) |
| Async commit loss | [xact.c#commit-flush](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1367-L1400) |
| Fresh heap size can hide a removed tail; reached index blocks still read | [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L750), [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362), [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1209-L1225) |
| Sync replication off by default | [guc.c#synchronous_standby_names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4087-L4095), [syncrep.c#SyncStandbysDefined](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L92-L93) |

## Open Questions

- No measured collision rate is claimed. The checksum output range alone does not establish a probability for a particular damage pattern. The header comment calls zero a valid checksum, but the implementation returns `(checksum % 65535) + 1`; implementation wins. [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L123-L127) [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214)
- Power-loss behavior with `fsync = off`, actual lost asynchronous commits, standby failover, torn-page repair and WAL CRC corruption are not injected here. Their coverage statements are source readings. The restore measurement removes required WAL entirely; it does not measure the later minimum-consistency/end-of-backup refusal branch.
- The exact `pg_waldump` output on a CRC failure was not traced here.

## Source References

- [configure.in](../../../../raw/postgres-12/configure.in#L279-L308)
- [amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4)
- [check_btree.sql#brin-rejection](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L49-L53)
- [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323)
- [amcheck.sgml#cannot-prove-absence](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L316-L317)
- [config.sgml](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2515-L2519)
- [pg_basebackup.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/pg_basebackup.sgml#L290-L365)
- [indextuple.c#external-fetch](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88)
- [gistutil.c](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L768-L799)
- [hashutil.c](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L219-L289)
- [heapam.c#scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231)
- [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1306-L1315)
- [tuptoaster.c](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2007-L2017)
- [xact.c](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1363-L1409)
- [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L189-L200)
- [xlogreader.c](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L707-L725)
- [xlogutils.c](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L352-L386)
- [index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L3515-L3535)
- [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10)
- [costsize.c](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L964-L971)
- [indxpath.c](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L1905-L1908)
- [gram.y](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283)
- [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1413-L1430)
- [syncrep.c#SyncStandbysDefined](../../../../raw/postgres-12/src/backend/replication/syncrep.c#L92-L93)
- [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925)
- [fd.c](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L345-L355)
- [lock.c#recovery-lock-limit](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766)
- [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162)
- [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)
- [elog.c](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L381-L388)
- [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161)
- [bin/Makefile#SUBDIRS](../../../../raw/postgres-12/src/bin/Makefile#L16-L34)
- [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144)
- [pg_basebackup.c](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L89-L100)
- [pg_checksums.c](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L513-L521)
- [pg_waldump.c:1134](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L1134)
- [xact.h:79](../../../../raw/postgres-12/src/include/access/xact.h#L79)
- [xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192)
- [pg_config.h.in](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864)
- [syncrep.h#SyncRepRequested](../../../../raw/postgres-12/src/include/replication/syncrep.h#L19-L20)
- [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L145-L162)
- [checksum_impl.h#reduce](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214)

## Navigation

- [v12/index](../../index.md)
- [Wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [How amcheck Checks B-Tree Indexes and Their Tables (unverified)](../indexing/amcheck-indexes-tables-and-data-checksums.md)
