---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Corruption Indicators in PostgreSQL 12 Logs With Data Checksums Disabled (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [What remains detectable](#what-remains-detectable)
  - [Values and structures that the checks compare](#values-and-structures-that-the-checks-compare)
  - [Detection and logging decision tree](#detection-and-logging-decision-tree)
  - [Data pages and short reads](#data-pages-and-short-reads)
  - [Index structure](#index-structure)
  - [TOAST and compressed values](#toast-and-compressed-values)
  - [Heap freezing and catalogs](#heap-freezing-and-catalogs)
  - [Transaction metadata and storage failures](#transaction-metadata-and-storage-failures)
  - [WAL, recovery, and the control file](#wal-recovery-and-the-control-file)
  - [Messages from explicit amcheck calls](#messages-from-explicit-amcheck-calls)
  - [Logging defaults and changes that affect visibility](#logging-defaults-and-changes-that-affect-visibility)
  - [When detection and relevant state change](#when-detection-and-relevant-state-change)
  - [Interpreting an incident](#interpreting-an-incident)
  - [Build boundaries and tests](#build-boundaries-and-tests)
  - [Final causal summary](#final-causal-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, when data checksums are disabled, which entries in the PostgreSQL server logs may indicate database corruption?

## Answer

**Look for failed [page](../../../glossary.md#page), index, [TOAST](../../../glossary.md#toast), [tuple freezing](../../../glossary.md#freezing), [catalog](../../../glossary.md#catalog), and [recovery](../../../glossary.md#crash-recovery) checks, even when [data checksums](../../../glossary.md#data-checksums) are off.** PostgreSQL still checks page-header bounds and several higher-level structural relationships. Those checks can reject damaged data without comparing a data-page checksum. They do not prove that every byte of an accepted page is correct. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L119) [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L719) [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L2016) [heapam.c#heap_prepare_freeze_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6142-L6163) [relcache.c#RelationBuildTupleDesc](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L639-L652) [xlogutils.c#XLogCheckInvalidPages](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L218-L243)

The tables below are a source-based reference for the pinned **PostgreSQL 12.2** tree, not an exhaustive list of every possible corruption symptom. Message text is shown as English source templates: `%u`, `%d`, `%s`, and similar fields stand for values inserted at runtime; `%m` inserts the operating-system error. These are not captured logs or measurements. Message translation depends on `lc_messages`. [configure.in:20](../../../../raw/postgres-12/configure.in#L20) [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L584-L590) [guc.c#lc_messages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3715-L3724) [snprintf.c#percent-m](../../../../raw/postgres-12/src/port/snprintf.c#L65-L71)

Here, **strong indicator** means that a consistency check rejected the structure it examined. **Context-dependent** means that the message also has a cause that does not establish damaged persistent data. This is an interpretation aid, not a claim that a log line identifies the failing hardware or software component. The same-checkout integrity-check documentation explicitly calls for investigating the cause of invariant violations. [amcheck.sgml#diagnosis](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L300-L336)

### What remains detectable

[Data checksums](../../../glossary.md#data-checksums) are off by default in this pin's `initdb`. The `-k` option enables them when creating a cluster. In a running server, `DataChecksumsEnabled()` reads `ControlFileData.data_checksum_version`; the read-only `data_checksums` setting reports the cluster's state. This page assumes that state is off. [initdb.c#data_checksums](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L139-L145) [initdb.c#checksum-option](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3143-L3148) [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826) [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835)

Disabling that check does not disable every integrity check:

| Check | What happens with data checksums off | Limit |
|---|---|---|
| Generic [page](../../../glossary.md#page) verification on a disk read | Tests header flags, free-space bounds, and special-space alignment; accepts a fully zero-filled page | A plausible header can accompany bad contents. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142) |
| Index and value interpretation | The consuming code can reject inconsistent index pages, chunk sequences, and compressed values | Detection depends on reaching that code and examining the affected data. [nbtpage.c#_bt_getbuf](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L748-L768) [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1923-L1935) [tuptoaster.c#toast_decompress_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2261-L2281) |
| [WAL](../../../glossary.md#wal) records and the [control file](../../../glossary.md#control-file) | Their separate checksum comparisons still run | The data-page setting does not gate these comparisons. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L696-L726) [xlog.c#control-file-checksum](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4638-L4647) |

Consequently, the absence of `page verification failed, calculated checksum ...` is expected when data checksums are disabled. That warning is reached only after `DataChecksumsEnabled()` allowed a checksum comparison and that comparison failed. Its absence is not a successful corruption scan. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L103) [bufpage.c#checksum-warning](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L144-L158)

### Values and structures that the checks compare

| Value or structure | Origin and state | What later code checks |
|---|---|---|
| `data_checksum_version` | Persistent cluster state loaded into `ControlFileData` | Whether to compare data-page checksums. [pg_control.h#data_checksum_version](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L221-L232) [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826) |
| `PageHeaderData` | Stored page fields, examined in a buffer | Flags and the order `pd_lower <= pd_upper <= pd_special <= BLCKSZ`, with alignment of `pd_special`. [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164) [bufpage.c#header-checks](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L111-L119) |
| Index special area and links | Stored index metadata | Expected access-method layout or relationships between connected pages. [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L710-L719) [ginbtree.c#ginStepRight](../../../../raw/postgres-12/src/backend/access/gin/ginbtree.c#L165-L190) |
| TOAST external pointer versus chunks returned | Stored `varatt_external` value size and identifier, compared with the chunks observed by an ordered index scan | Expected sequence and sizes, and whether all expected chunks were returned. A missing result does not by itself locate the fault in the TOAST heap versus its index. [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1890-L1935) [tuptoaster.c#chunk-checks](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L2016) [postgres.h#varatt_external](../../../../raw/postgres-12/src/include/postgres.h#L55-L73) |
| Tuple `xmin`, relation `relfrozenxid`, and freeze cutoff | Stored row and [catalog](../../../glossary.md#catalog) state, plus the cutoff supplied to [VACUUM](../../../glossary.md#vacuum)'s current pass | Whether the row's transaction information is consistent with the relation's frozen horizon and can be frozen. Stored does not imply stale; the error requires the specific contradictory values. [vacuumlazy.c#freeze-caller](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1186-L1195) [heapam.c#heap_prepare_freeze_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6142-L6163) |
| `XLogRecord` length, previous-record link, and checksum | Stored WAL fields compared with reader state and a freshly calculated checksum | Record layout, continuity, and checksum agreement. [xlogreader.c#record-validation](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L638-L726) [xlogrecord.h#XLogRecord](../../../../raw/postgres-12/src/include/access/xlogrecord.h#L41-L55) |
| `ErrorData` and logging settings | Process-local reporting state; output flags are computed for the current report | Severity, [SQLSTATE](../../../glossary.md#sqlstate), filtering, formatting, and destination. [elog.c#errstart](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L283-L305) [elog.c#default-sqlstate](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388) [elog.c#send_message_to_server_log](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L2860-L2867) [elog.h#ErrorData](../../../../raw/postgres-12/src/include/utils/elog.h#L346-L365) |

### Detection and logging decision tree

```text
Requested data or operation
|
+-- Relation page requested [A]
|   +-- Valid buffer already cached -> return buffer; no new disk-page verification
|   +-- Disk read needed
|       +-- OS read fails -> file-access ERROR
|       +-- Short read -> normally ERROR; recovery/zero_damaged_pages has a zero-fill branch
|       +-- Full page read -> checksum off: skip checksum comparison
|           +-- Header sane OR entire page zero -> generic verifier accepts
|           +-- Otherwise -> normally invalid-page ERROR
|               +-- zero_damaged_pages OR caller's RBM_ZERO_ON_ERROR -> WARNING + zero-fill
|   +-- Consumer interprets page/value -> its structural checks can still raise an error [B]
|
+-- WAL or startup control file read [C]
|   +-- Separate checksum/layout checks pass -> continue
|   +-- Check fails -> report according to startup/recovery caller and WAL source
|
+-- Explicit amcheck function invoked [D]
    +-- Requested B-tree invariants pass -> return normally
    +-- Invariant fails -> integrity-check error

Every report -> severity/default SQLSTATE -> server-log filter
            -> optional logging hook -> formatter and configured destination [E]
```

- **A:** `ReadBuffer()` supplies `RBM_NORMAL`; the buffer-hit return precedes `smgrread()` and `PageIsVerified()`. The short-read and invalid-header branches have separate handling. [bufmgr.c#ReadBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L590-L598) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L760-L795) [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L629) [bufmgr.c#verification-result](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142) `smgrread()` dispatches through `smgrsw` to `mdread()`. [smgr.c#smgrread](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L579-L591) [smgr.c#smgrsw](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L66-L82)
- **B:** `_bt_getbuf()` calls `_bt_checkpage()` after obtaining the buffer; fetching a TOAST value independently examines the returned chunks. [nbtpage.c#_bt_getbuf](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L763-L768) [tuptoaster.c#chunk-checks](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L2016)
- **C:** `ReadRecord()` receives WAL-reader diagnostics and decides how to report them. `ReadControlFile()` performs its own comparison. [xlog.c#ReadRecord](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4240-L4284) [xlog.c#control-file-checksum](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4638-L4647)
- **D:** The two `amcheck` entry points select their requested checks; the extension is a separate contrib module. [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L218) [Makefile#module](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10)
- **E:** Report construction selects the output flags and default code. An installed [hook](../../../glossary.md#hook) can suppress server output; the default hook pointer is null. The log formatter then adds the message fields. [elog.c:101](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L101) [elog.c#errstart](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L283-L305) [elog.c#default-sqlstate](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388) [elog.c#EmitErrorReport](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L1444-L1467) [elog.c#send_message_to_server_log](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L2860-L2933)

The catalog uses the reporting level requested at the source site. PostgreSQL can promote an `ERROR` to `FATAL` in startup or other contexts without an error handler, and to `PANIC` inside a [critical section](../../../glossary.md#critical-section). Therefore the message and its context matter as well as its displayed [log severity](../../../glossary.md#log-severity). [elog.c#severity-promotion](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L238-L268)

### Data pages and short reads

| Exact message template | Source level / SQLSTATE | Trigger and interpretation |
|---|---|---|
| `invalid page in block %u of relation %s` | `ERROR` / `XX001` | With checksums off, a full read failed the header/all-zero checks. Strong indicator of a malformed page as read; the message names its block and relation path. Normal `RBM_NORMAL` access rejects it. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142) [bufmgr.c#invalid-page](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) |
| `invalid page in block %u of relation %s; zeroing out page` | `WARNING` / `XX001` | The same failure reached the zero-fill branch. Normally an invalid page is an error because `zero_damaged_pages` defaults to off. The branch instead fires when that setting is on **or the caller selected `RBM_ZERO_ON_ERROR`**; it zeroes the buffer. This warning does not prove that somebody enabled the setting. [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1149-L1162) [bufmgr.c#zero-on-error](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) |
| `could not read block %u in file "%s": read only %d of %d bytes` | `ERROR` / `XX001` | A request for a full block returned fewer bytes, including possibly no bytes. The code identifies EOF or a partial block at EOF. Strong evidence that the requested block could not be supplied; it does not identify why the request and file size disagree. By default this is an error; `InRecovery` or `zero_damaged_pages = on` instead returns zeroes for this short-read branch. [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L629) |

The zero-on-error mode exists for callers prepared to repair noncritical data. Both the [free space map](../../../glossary.md#free-space-map) and [visibility map](../../../glossary.md#visibility-map) select it explicitly. Preserve the relation [fork](../../../glossary.md#fork) in the path: a warning for one of those auxiliary forks has different scope from loss of a [heap](../../../glossary.md#heap) or index data page. [bufmgr.c#read-modes](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L614-L625) [freespace.c:598](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L598) [visibilitymap.c#vm_readbuf](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L607-L609)

An [all-zero page](../../../glossary.md#all-zero-page) is a separate case. The generic verifier accepts it because a crash between extending and initializing a relation can legitimately leave one. A B-tree reader accessing an existing index page rejects a `PageIsNew()` page with the index-specific message below. The generic acceptance rule and the index rule serve different purposes. [bufpage.c#zero-page-rationale](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L71-L79) [bufpage.c#all-zero-check](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L696-L708)

### Index structure

These checks belong to the indicated [access method](../../../glossary.md#access-method), so match the index type as well as the text. They are checks of the structure being used, independently of the data-checksum setting. [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L719) [gistutil.c#gistcheckpage](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L767-L798) [hashutil.c#_hash_checkpage](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L219-L285)

| Message template | Source level / SQLSTATE | Trigger and interpretation |
|---|---|---|
| `index "%s" contains unexpected zero page at block %u` | `ERROR` / `XX002` | [B-tree](../../../glossary.md#b-tree), [GiST](../../../glossary.md#gist), and [hash](../../../glossary.md#hash-index) readers found `PageIsNew()` where they expected an initialized index page. Each source site supplies `HINT: Please REINDEX it.` Strong structural indicator. [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L696-L708) [gistutil.c#gistcheckpage](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L775-L787) [hashutil.c#_hash_checkpage](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L229-L241) |
| `index "%s" contains corrupted page at block %u` | `ERROR` / `XX002` | B-tree/GiST/hash special-space size disagrees with that method's layout; hash also uses this text for an unacceptable page-type flag. Strong structural indicator. [nbtpage.c#special-space](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L710-L719) [gistutil.c#special-space](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L789-L798) [hashutil.c#special-space-and-flags](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L243-L264) |
| `corrupted BRIN index: inconsistent range map` | `ERROR` / `XX002` | [BRIN](../../../glossary.md#brin) found a repeated unusable map target, a target offset beyond the page's items, or an unused target item on the shown paths. Strong inconsistency between the range map and its target. [brin_revmap.c#brinGetTupleForHeapBlock](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L248-L288) [brin_revmap.c#brinRevmapDesummarizeRange](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L378-L388) |
| `unexpected page type 0x%04X in BRIN index "%s" block %u` | `ERROR` / `XX002` | Reverse-map extension encountered a page that is neither new nor a regular BRIN page. Strong page-type mismatch. [brin_revmap.c#revmap_physical_extend](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L597-L604) |
| `right sibling of GIN page is of different type` | `ERROR` / `XX000` | [GIN](../../../glossary.md#gin) followed a right link and found different leaf/data flags on the next page. Strong relationship mismatch, with a generic internal-error code. [ginbtree.c#ginStepRight](../../../../raw/postgres-12/src/backend/access/gin/ginbtree.c#L165-L190) |
| `unexpected SPGiST tuple state: %d` | `ERROR` / `XX000` | An [SP-GiST](../../../glossary.md#sp-gist) scan reached a tuple state not valid for that position. Normal live tuples proceed; supported redirect/dead cases take their own branches before the error. The state/position combination is the inconsistency. [spgscan.c#spgTestLeafTuple](../../../../raw/postgres-12/src/backend/access/spgist/spgscan.c#L715-L752) [spgscan.c#inner-tuple-state](../../../../raw/postgres-12/src/backend/access/spgist/spgscan.c#L848-L865) |

The `REINDEX` hint is source-provided advice for those particular checks, not a guarantee that a rebuild fixes the underlying cause or that the heap is intact. The integrity-check documentation explicitly says there is no general repair method and a rebuild may be ineffective. [amcheck.sgml#repair-limits](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L324-L336)

### TOAST and compressed values

[TOAST](../../../glossary.md#toast) stores large values in chunks. When a value is fetched, the code reads its external pointer, calculates the expected chunk count from its stored size, and uses an ordered index scan to retrieve chunks. That is why these errors can appear only when a particular value is accessed, after simpler page checks have passed. [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1890-L1935)

1. The external pointer supplies the value identifier and stored byte size. [tuptoaster.c#external-pointer](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1890-L1899)
2. The scan supplies each chunk's sequence number and data. Each sequence number must match the next expected number, and each chunk must have the expected size. [tuptoaster.c#chunk-checks](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1935-L1947) [tuptoaster.c#chunk-checks](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L1997)
3. After scanning, the number of returned chunks must match the expected count. If the value is compressed, decompression has a further validity check. [tuptoaster.c#missing-chunk](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2006-L2016) [tuptoaster.c#toast_decompress_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2261-L2281)

| Message template | Source level / SQLSTATE | What failed |
|---|---|---|
| `missing chunk number %d for toast value %u in %s` | `ERROR` / `XX000` | The full-value fetch finished before returning every expected chunk. Strong fetch inconsistency; because retrieval uses the TOAST index, this does not prove that a chunk is physically absent from the heap. [tuptoaster.c#ordered-fetch](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1909-L1935) [tuptoaster.c#missing-chunk](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2009-L2016) |
| `unexpected chunk number %d (expected %d) for toast value %u in %s` | `ERROR` / `XX000` | Observed sequence differs from expected sequence. [tuptoaster.c#chunk-order](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L1972) |
| `unexpected chunk size %d (expected %d) in chunk %d of %d for toast value %u in %s` | `ERROR` / `XX000` | A nonfinal chunk does not have the required chunk size. [tuptoaster.c#chunk-size](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1973-L1980) |
| `unexpected chunk size %d (expected %d) in final chunk %d for toast value %u in %s` | `ERROR` / `XX000` | The final chunk does not reconcile with the external pointer's stored size. [tuptoaster.c#final-chunk-size](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1982-L1990) |
| `found toasted toast chunk for toast value %u in %s` | `ERROR` / `XX000` | A returned chunk itself has an extended representation that this path does not permit. [tuptoaster.c#chunk-representation](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1944-L1962) |
| `compressed data is corrupted` | `ERROR` / `XX000` | The compressed-value decoder rejected the supplied representation. Strong value-validation failure, without a data-page checksum comparison. [tuptoaster.c#toast_decompress_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2261-L2281) |

Slice fetches have related variants, including `when fetching slice`, out-of-range chunk numbers, and the same missing-chunk text. Keep the whole message rather than matching only `chunk`. [tuptoaster.c#slice-checks](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2193-L2220) [tuptoaster.c#slice-missing-chunk](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2242-L2249)

### Heap freezing and catalogs

[Freezing](../../../glossary.md#freezing) checks compare a tuple's [transaction ID](../../../glossary.md#transaction-id) with relation metadata and the current cutoff. VACUUM passes these values into `heap_prepare_freeze_tuple()`. Valid older committed transaction information can be frozen; a contradiction takes an error branch instead. [vacuumlazy.c#freeze-caller](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1186-L1195) [heapam.c#heap_prepare_freeze_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6142-L6163)

| Message template | Source level / SQLSTATE | Trigger and interpretation |
|---|---|---|
| `found xmin %u from before relfrozenxid %u` | `ERROR` / `XX001` | A normal tuple `xmin` precedes the relation's recorded frozen horizon. Strong contradiction between row and relation metadata. [heapam.c#xmin-horizon-check](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6142-L6151) |
| `uncommitted xmin %u from before xid cutoff %u needs to be frozen` | `ERROR` / `XX001` | `xmin` precedes the freeze cutoff but the commit-status check does not report it committed. Preserve any preceding transaction-status file errors as part of the diagnosis. [heapam.c#xmin-commit-check](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6153-L6163) |
| `catalog is missing %d attribute(s) for relid %u` | `ERROR` / `XX000` | Building the relation descriptor did not obtain its expected attribute count from `pg_attribute`. Strong inconsistency in the metadata observed by the lookup; it does not locate the fault in the catalog heap versus its index. [relcache.c#attribute-scan](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L520-L551) [relcache.c#attribute-count](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L639-L652) |
| `invalid attribute number %d for %s` | `ERROR` / `XX000` | That descriptor-building scan obtained an attribute number outside the relation's allowed range. Strong catalog/descriptor mismatch on this path. [relcache.c#attribute-number-check](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L549-L561) |

The attribute lookup normally allows its catalog index after critical relation-cache entries exist; during initial bootstrap it forces a heap scan. Thus even a message that says `catalog is missing` describes what that scan returned, rather than proving which underlying file is damaged. [relcache.c#attribute-scan](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L534-L544)

### Transaction metadata and storage failures

[SLRU](../../../glossary.md#slru) metadata uses a different reporting path from ordinary relation-page verification. Read its `DETAIL` line: the same primary message covers open, seek, read, write, synchronization, and close failures. [slru.c#SlruReportIOError](../../../../raw/postgres-12/src/backend/access/transam/slru.c#L893-L959)

| Message template or pair | Source level / SQLSTATE | Trigger and interpretation |
|---|---|---|
| `could not access status of transaction %u` plus `Could not open file "%s": %m.` or `Could not read from file "%s" at offset %u: %m.` | `ERROR` / OS-error-dependent | A metadata file could not be opened/read. Context-dependent: the operating-system error distinguishes missing files, permissions, and I/O failure; the primary text alone does not. [slru.c#SlruReportIOError](../../../../raw/postgres-12/src/backend/access/transam/slru.c#L909-L932) [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L600-L648) |
| `could not access status of transaction %u` plus `Could not read from file "%s" at offset %u: read too few bytes.` | `ERROR` / `XX000` | The metadata reader could not obtain the expected page. Strong short-read symptom; preserve path and offset. [slru.c#short-read](../../../../raw/postgres-12/src/backend/access/transam/slru.c#L922-L932) |
| `MultiXactId %u does no longer exist -- apparent wraparound` | `ERROR` / `XX000` | The requested [MultiXact](../../../glossary.md#multixact) ID precedes the current oldest retained ID. Serious when normal tuple access requests it, but the public diagnostic function can also be called with an old ID. [multixact.c#GetMultiXactIdMembers](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L1250-L1279) [multixact.c#pg_get_multixact_members](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L3336-L3356) |
| `MultiXactId %u has not been created yet -- apparent wraparound` | `ERROR` / `XX000` | The requested ID is not earlier than the next ID to allocate. Compare the tuple-driven call path with a caller explicitly supplying an ID before attributing this to stored corruption. [multixact.c#GetMultiXactIdMembers](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L1250-L1286) [heapam.c#MultiXactIdGetUpdateXid](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6470-L6493) [multixact.c#pg_get_multixact_members](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L3336-L3356) |
| `could not read block %u in file "%s": %m` | `ERROR` / OS-error-dependent | An operating-system read failed, as distinct from a completed short read. An I/O failure is an urgent storage symptom; permissions can reach the same template. [md.c#read-failure](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L612) [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L600-L648) |
| `could not fsync file "%s": %m` | Normally `PANIC` / OS-error-dependent | On this checkpoint synchronization path, a non-ignorable failure uses `data_sync_elevel(ERROR)`. Default `data_sync_retry = off` promotes it to `PANIC`; setting it on would retain `ERROR`. It reports a durability failure, not a completed comparison proving a damaged page. [sync.c#ProcessSyncRequests](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L387-L401) [fd.c#data_sync_elevel](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L3475-L3490) [guc.c#data_sync_retry](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1946-L1953) |

For OS-error-dependent SQLSTATEs, `EIO` maps to `58030`, a missing file to `58P01`, and permission failures to an insufficient-privilege code. Do not require `XX001` or `XX002` before considering a report relevant. [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L600-L648) [errcodes.txt#system-errors](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L431-L437)

### WAL, recovery, and the control file

Data-page checksums and WAL/control-file checksums cover different bytes. `ValidXLogRecord()` calculates its checksum over the WAL record. `ReadControlFile()` checks the control structure. Neither comparison tests `DataChecksumsEnabled()`. Therefore checksum-related messages in this section are compatible with `data_checksums = off`. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L696-L726) [xlog.c#control-file-checksum](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4638-L4647)

| Exact message template | Source level / SQLSTATE | Trigger and interpretation |
|---|---|---|
| `incorrect resource manager data checksum in record at %X/%X` | WAL caller chooses `LOG` / `00000` or `PANIC` / `XX000`; repeated local `LOG` complaints can become `DEBUG1` | The stored record checksum differs from the recalculated value. Strong failed record validation; establish WAL source and whether this is an incomplete local tail before concluding that a required complete record is damaged. [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L711-L723) [xlog.c#ReadRecord](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4240-L4284) [xlog.c#emode_for_corrupt_record](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L12126-L12156) |
| `invalid record length at %X/%X: wanted %u, got %u` | Same WAL-caller reporting rules | The header length is below the minimum. **Context-dependent:** the local end of available WAL also reaches invalid-record reporting; this line by itself is not proof of corruption. Complete archived/streamed records have a different expectation. [xlogreader.c#ValidXLogRecordHeader](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L638-L648) [xlog.c#emode_for_corrupt_record](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L12126-L12156) |
| `invalid resource manager ID %u at %X/%X` | Same WAL-caller reporting rules | Record header names a resource manager beyond the reader's maximum. Failed header validation; correlate with position and WAL source. [xlogreader.c#ValidXLogRecordHeader](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L650-L656) |
| `record with incorrect prev-link %X/%X at %X/%X` | Same WAL-caller reporting rules | The previous-record link is invalid for random access or disagrees with the previous position during sequential reading. The sequential check is explicitly meant to catch torn WAL pages. [xlogreader.c#previous-record-link](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L658-L688) |
| `could not locate a valid checkpoint record` | `PANIC` / `XX000` | Startup could not read the required checkpoint. Strong startup/recovery failure; read preceding WAL-reader messages to distinguish failed validation from unavailable WAL. [xlog.c#checkpoint-failure](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6536-L6545) |
| `incorrect checksum in control file` | `FATAL` / `XX000` | `ControlFileData` checksum comparison failed at startup. Strong control-file validation failure, independent of page-checksum state. [xlog.c#control-file-checksum](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4638-L4647) |
| `page %u of relation %s is uninitialized` or `page %u of relation %s does not exist`, followed by `WAL contains references to invalid pages` | Page lines `WARNING` / `01000`, final line `PANIC` / `XX000` | Recovery has an unresolved reference when it checks for consistency, or finds one after consistency was reached. Preserve the warning lines: they identify the pages that the final panic does not name. [xlogutils.c#report_invalid_page](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L60-L73) [xlogutils.c#log_invalid_page](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L85-L105) [xlogutils.c#XLogCheckInvalidPages](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L218-L243) |

Before reaching a consistent state, recovery normally records an invalid-page reference in an in-memory table and reports it only at `DEBUG1`. Later drop/truncate replay can remove a reference. The strong warning-plus-panic case is the unresolved reference at a consistency check, or a newly encountered one after consistency, not every debug mention of a missing page. [xlog.c#CheckRecoveryConsistency](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7859-L7879) [xlogutils.c#log_invalid_page](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L85-L138) [xlogutils.c#forget_invalid_pages](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L140-L171) [xlogutils.c#drop-and-truncate-callers](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L599-L640) [xlogutils.c#XLogCheckInvalidPages](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L218-L243)

For repeated WAL-reader complaints, the usual caller-selected level is retained for the first report. If the source is local `pg_wal`, the requested level is `LOG`, and the same record position was already complained about, `emode_for_corrupt_record()` substitutes `DEBUG1`. This intentional suppression means the number of visible lines is not the number of failed attempts. [xlog.c#emode_for_corrupt_record](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L12126-L12156)

### Messages from explicit amcheck calls

[`amcheck`](../../../glossary.md#amcheck) is a separate [contrib](../../../glossary.md#contrib) extension. These messages arise when an installed function is explicitly invoked, rather than from an automatic background corruption scan. At this pin the module builds `verify_nbtree.o` and exposes B-tree-checking entry points. [Makefile#module](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10) [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L218) [amcheck--1.0.sql#functions](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L1-L22)

| Message template | Source level / SQLSTATE | Check that produced it |
|---|---|---|
| `item order invariant violated for index "%s"` | `ERROR` / `XX002` | Adjacent index entries violate the ordering test. Retain the detailed index/heap tuple locations and page [LSN](../../../glossary.md#lsn). [verify_nbtree.c#item-order](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1184-L1198) |
| `heap tuple (%u,%u) from table "%s" lacks matching index tuple within index "%s"` | `ERROR` / `XX001` | An explicitly requested `heapallindexed` check failed to find the heap tuple in its index fingerprint filter. The argument defaults to false in the entry points; passing true enables this additional cross-check. [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L181-L215) [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1896-L1919) |
| `index "%s" meta page is corrupt` | `ERROR` / `XX002` | The expected B-tree [metapage](../../../glossary.md#metapage) lacks the meta flag or expected magic value. [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2370-L2380) |

`bt_index_check()` takes [AccessShareLock](../../../glossary.md#lock-mode) on heap and index. `bt_index_parent_check()` takes [ShareLock](../../../glossary.md#lock-mode) and additionally checks parent/child relationships. The latter's `rootdescend` option is also false by default and becomes active only when explicitly supplied true. These are different scopes of verification, not interchangeable promises that all data is sound. [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L218) [amcheck.sgml#detection-limits](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L314-L336)

### Logging defaults and changes that affect visibility

An error returned to a client and an error written to the server log have separate output decisions. `log_min_messages`, not `client_min_messages`, controls the server decision. By default the server threshold is `WARNING`; its ordering retains `WARNING`, `ERROR`, `LOG`, `FATAL`, and `PANIC`, while excluding `DEBUG1`. `LOG` has special ordering for this test, so raising the threshold to `LOG` would suppress ordinary warnings and errors. [elog.c#errstart](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L283-L305) [guc.c#log_min_messages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4296-L4305) [elog.c#is_log_level_output](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L3430-L3456)

| Setting | Pinned default | Meaning for this reference and scope of a change |
|---|---|---|
| `log_min_messages` | `warning` | Default retains the principal warning/error/panic signals above. A changed threshold can hide them; `is_log_level_output()` makes that decision. `PGC_SUSET`: session/transaction `SET` for a role allowed to change it. [guc.c#log_min_messages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4296-L4305) [elog.c#is_log_level_output](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L3438-L3456) |
| `log_error_verbosity` | `default` | Default text output keeps `DETAIL`, `HINT`, and `CONTEXT`, but does not add SQLSTATE beside the severity or source `LOCATION`. Optional `verbose` adds those fields; `terse` omits the supporting fields. `PGC_SUSET`: session/transaction scope for an allowed role. [guc.c#verbosity-options](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L281-L286) [guc.c#log_error_verbosity](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4286-L4294) [elog.c#text-log-fields](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L2860-L2933) |
| `log_line_prefix` | `%m [%p] ` | Default provides timestamp/PID fields without SQLSTATE. For ongoing log correlation, optionally add `%e`, which the formatter expands to SQLSTATE. `PGC_SIGHUP`: configuration reload. [guc.c#log_line_prefix](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3592-L3600) [elog.c#sqlstate-prefix](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L2591-L2596) |
| `log_destination` | `stderr` | Look at the configured output, rather than assuming a file under the data directory. Optional `csvlog` includes severity and SQLSTATE in separate fields, and requires the collector for normal CSV output. `PGC_SIGHUP`: reload. [guc.c#log_destination](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3848-L3859) [elog.c#csv-fields](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L2743-L2764) [elog.c#csv-destination](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L3027-L3047) |
| `logging_collector` | `off` | Default does not start PostgreSQL's log-file collector. Enabling it starts a capture subprocess; it is `PGC_POSTMASTER`, so that change requires restart. [guc.c#logging_collector](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1616-L1624) |
| `data_checksums` | Off at default `initdb` | Reports cluster state; `PGC_INTERNAL`, so it cannot be enabled with `SET` or a configuration reload. [initdb.c#checksum-default](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L139-L145) [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835) |
| `zero_damaged_pages` | `off` | Default rejects malformed pages. Turning it on selects the destructive warning-plus-zero-fill path; this page does not recommend that as a diagnostic action. `PGC_SUSET`: session/transaction scope for an allowed role. Internal `RBM_ZERO_ON_ERROR` can select the same branch independently. [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1149-L1162) [bufmgr.c#zero-on-error](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) |
| `ignore_checksum_failure` | `off` | Default does not bypass a failed checksum. Turning it on can accept a checksum-failing page whose header is sane, but has no effect when data checksums are off. `PGC_SUSET`: session/transaction scope for an allowed role. [guc.c#ignore_checksum_failure](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1134-L1148) [bufpage.c#checksum-bypass](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L148-L158) |
| `data_sync_retry` | `off` | Default makes covered synchronization failures panic. On retains the caller's lower level through `data_sync_elevel()`; this changes failure handling, not the correctness of the data. `PGC_POSTMASTER`: restart. [guc.c#data_sync_retry](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1946-L1953) [fd.c#data_sync_elevel](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L3475-L3490) |

The [GUC context](../../../glossary.md#guc-context) names above express where a setting can take effect; the `GucContext` definition distinguishes reload, restart, superuser, and user scope. The optional logging changes enrich future reports; they cannot recreate fields discarded from an earlier log. [guc.h#GucContext](../../../../raw/postgres-12/src/include/utils/guc.h#L39-L80) [elog.c#text-log-fields](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L2860-L2933)

SQLSTATE is useful as another search key, but not a complete corruption filter:

| Code | Meaning and limit |
|---|---|
| `XX001` | `data_corrupted`, explicitly assigned at several page/tuple checks. [errcodes.txt#internal-errors](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L489-L494) [bufmgr.c#invalid-page](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) |
| `XX002` | `index_corrupted`, explicitly assigned at many index checks. [errcodes.txt#internal-errors](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L489-L494) [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L702-L719) |
| `XX000` | Generic internal error. An `ERROR` or stronger report without an explicit code gets this default, so TOAST and several structural errors also use it. It does not itself prove corruption. [errcodes.txt#internal-errors](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L489-L494) [elog.c#default-sqlstate](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388) [tuptoaster.c#missing-chunk](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2012-L2016) |
| OS-error-dependent codes | File failures distinguish I/O, missing-file, permission, resource, and other cases. Use the message and operating-system detail too. [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L600-L648) |

`ReadRecord()`-reported WAL errors do not assign a corruption SQLSTATE themselves. A `LOG`/`DEBUG1` report therefore uses `00000`, a bare warning uses `01000`, and a bare panic uses `XX000`, according to the reporter's defaults. Matching only the two explicit corruption codes misses these paths. [xlog.c#ReadRecord](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4281-L4284) [elog.c#default-sqlstate](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388) [errcodes.txt#successful_completion](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L76-L83) [errcodes.txt#warning](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L81-L91)

### When detection and relevant state change

| Event | State before → state after | Effect on later detection |
|---|---|---|
| Read satisfied by an existing valid buffer | Page already cached → same buffer returned | Generic disk-page verification is not rerun; consumer checks can still run, as `_bt_getbuf()` shows. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L760-L795) [nbtpage.c#_bt_getbuf](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L763-L768) |
| Page must be loaded | No usable cached page → disk read and verification | Header checks run; checksum comparison is skipped in the assumed off state. [bufmgr.c#verification-result](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L119) |
| A large value is fetched | Stored pointer/chunks → ordered retrieval and reconstruction | TOAST errors can first surface at value access. [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1890-L1935) [tuptoaster.c#chunk-checks](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L2016) |
| VACUUM examines a retained tuple for freezing | Tuple/catalog horizons → current freeze decision | Contradictions surface through the explicit `xmin` checks; ordinary old committed `xmin` can become frozen. [vacuumlazy.c#freeze-caller](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1186-L1195) [heapam.c#heap_prepare_freeze_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6142-L6163) |
| Explicit `amcheck` call | Unchecked requested invariants → checked invariants or error | Only requested checks run; `heapallindexed` and `rootdescend` default false. Success cannot prove absence of all corruption. [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L218) [amcheck.sgml#detection-limits](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L314-L336) |
| Recovery records an invalid-page reference | No entry → entry keyed by relation/fork/block | Before consistency the report is normally debug-only; later drop/truncate can remove the entry. [xlogutils.c#invalid-page-state](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L85-L138) [xlogutils.c#forget_invalid_pages](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L140-L171) [xlogutils.c#drop-and-truncate-callers](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L599-L640) |
| Recovery checks unresolved entries or finds a new one after consistency | Inconsistent reference remains → warning plus panic | The page warnings and the final panic belong together. [xlogutils.c#log_invalid_page](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L85-L105) [xlogutils.c#XLogCheckInvalidPages](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L218-L243) |
| Reader retries the same failed local WAL position | First complaint remembered → later same-position `LOG` demoted to `DEBUG1` | Default logs can show only the first complaint. [xlog.c#emode_for_corrupt_record](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L12126-L12156) |

### Interpreting an incident

Use this sequence to interpret the evidence:

1. **Preserve the complete report and adjacent lines.** Keep the primary message, `DETAIL`, `HINT`, `CONTEXT`, timestamp, PID, and any supplied object path, block, tuple location, or LSN. These fields distinguish the failure paths in the catalog; the standard text formatter emits supporting fields at default verbosity. [elog.c#text-log-fields](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L2860-L2933) [slru.c#SlruReportIOError](../../../../raw/postgres-12/src/backend/access/transam/slru.c#L909-L952) [xlogutils.c#report_invalid_page](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L60-L73)
2. **Determine the operation that reached the check.** A disk-page read, an index traversal, a TOAST fetch, a VACUUM freeze check, WAL replay, and an explicit integrity-check call test different structures. Use the caller boundaries above instead of inferring that the entire database has been scanned. [bufmgr.c#verification-result](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [nbtpage.c#_bt_getbuf](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L763-L768) [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1923-L1935) [vacuumlazy.c#freeze-caller](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1186-L1195) [xlog.c#ReadRecord](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4240-L4284) [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L218)
3. **Apply the branch qualifications before declaring corruption.** Match the exact message and source code, not just words such as `invalid`, `checksum`, or `ERROR`. The cases below show why. [xlog.c#emode_for_corrupt_record](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L12126-L12156) [multixact.c#pg_get_multixact_members](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L3340-L3356) [nbtinsert.c#unique-violation](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L570)
4. **Choose the next check for the affected structure.** For an index, consider the scope, permissions, and locks of the pinned `amcheck` functions; for a storage failure, retain the actual OS error. Investigate the cause before treating a rebuild hint as a complete remedy. [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L218) [amcheck--1.1--1.2.sql#restricted-function](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.1--1.2.sql#L12-L19) [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L600-L648) [amcheck.sgml#repair-limits](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L324-L336)

| Message or condition | Why it is insufficient by itself |
|---|---|
| Local-tail `invalid record length ...` | Local end-of-WAL can produce invalid-record reporting; record position, WAL source, and recovery outcome supply the needed context. [xlog.c#emode_for_corrupt_record](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L12126-L12156) |
| `invalid MultiXactId: %u` | The SQL-facing function rejects an input below `FirstMultiXactId` with `invalid_parameter_value`. This can be a bad diagnostic argument rather than stored corruption. [multixact.c#pg_get_multixact_members](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L3336-L3343) |
| `invalid memory alloc request size %zu` | The allocator rejected an invalid size supplied by a caller. It does not inspect or identify a damaged relation page. A bad stored length is a hypothesis requiring the caller/context, not a conclusion from this line. [mcxt.c#MemoryContextAlloc](../../../../raw/postgres-12/src/backend/utils/mmgr/mcxt.c#L764-L779) |
| `duplicate key value violates unique constraint "%s"` | The normal unique-index insertion path emits this when it finds a conflicting key; it is not itself a structural-integrity test. [nbtinsert.c#unique-violation](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L570) |
| `%s (PID %d) was terminated by signal %d: %s` or `all server processes terminated; reinitializing` | These report process termination and the crash-reinitialization path. They contain no test proving damaged persistent data; use the preceding failure and its context. [postmaster.c#LogChildExit](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3656-L3686) [postmaster.c#crash-reinitialization](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3911-L3926) |
| `leftover placeholder tuple detected in BRIN index "%s", deleting` | Normal summarized tuples do not take this warning branch. A leftover placeholder from crashed summarization does; the desummarization code warns and proceeds. This differs from its range-map corruption errors. [brin_revmap.c#placeholder-cleanup](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L378-L405) |
| No matching corruption messages | Untouched pages, cached pages, unrequested invariants, plausible but wrong contents, and log filtering leave gaps in detection. Silence cannot certify all data. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L760-L795) [bufpage.c#header-check-limits](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L105-L119) [elog.c#is_log_level_output](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L3438-L3456) [amcheck.sgml#detection-limits](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L314-L336) |

The [database health checklist](../server-administration/database-health-checklist.md) provides the broader operational context for logging, crashes, storage pressure, and monitoring. This page narrows that work to interpreting possible corruption signals.

### Build boundaries and tests

Core reporting consumes generated `utils/errcodes.h`; its source of truth is `src/backend/utils/errcodes.txt`. The build generates the header with `generate-errcodes.pl` and exposes it in the build include tree. Use those source definitions, rather than assuming every integrity error has a dedicated corruption code. [elog.h#generated-error-codes](../../../../raw/postgres-12/src/include/utils/elog.h#L58-L71) [Makefile#errcodes-generation](../../../../raw/postgres-12/src/backend/utils/Makefile#L41-L42) [Makefile#header-stamp](../../../../raw/postgres-12/src/backend/utils/Makefile#L65-L71) [errcodes.txt#internal-errors](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L489-L494)

The index layout checks are normal runtime `ereport`/`elog` branches, not just assertions enabled by a special build. For example, `_bt_checkpage()` tests the special-area size against the compiled `BTPageOpaqueData` layout. `amcheck` is separately built as a contrib module and registers its SQL-callable functions through its extension scripts. [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L719) [Makefile#module](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10) [amcheck--1.0.sql#functions](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L1-L22) [amcheck--1.1--1.2.sql#function](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.1--1.2.sql#L12-L19)

| Test source inspected | What it supports, and its boundary |
|---|---|
| `contrib/amcheck/sql/check_btree.sql` | Exercises function permissions, parent/root checks after page deletion, and a TOAST-representation false-positive regression case. This is test-input evidence, not a new run or a reproduction of every error above. [check_btree.sql#permissions](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L21-L27) [check_btree.sql#structural-checks](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L80-L105) |
| `src/bin/pg_checksums/t/002_actions.pl` | Includes a default checksum-disabled cluster and file corruption checked through the offline checksum utility. Its utility-output checks are not PostgreSQL server-log checks and do not establish coverage of the checksum-disabled runtime paths above. [002_actions.pl#corruption-test](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L51-L76) [002_actions.pl#default-state](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L89-L98) |

No server was built or run for this page, and no measured numbers or fabricated log captures are published. Runtime reproduction of each catalog entry remains outside this source-only verification.

### Final causal summary

**Checksums off → a page read skips the data-page checksum comparison → header checks still run → a consumer can additionally reject an inconsistent index, tuple, catalog, or value → the reporting path supplies a message and code → current logging rules determine whether it is retained.** WAL and the control file follow separate validation paths. A later read, value access, VACUUM pass, or explicit integrity check can expose a previously unexamined inconsistency; a quiet log cannot prove that every such check has run or every byte is sound. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L119) [nbtpage.c#_bt_getbuf](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L763-L768) [tuptoaster.c#chunk-checks](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L2016) [heapam.c#heap_prepare_freeze_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6142-L6163) [relcache.c#attribute-count](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L639-L652) [elog.c#errstart](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L283-L305) [xlogreader.c#ValidXLogRecord](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L696-L726) [xlog.c#control-file-checksum](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4638-L4647) [amcheck.sgml#detection-limits](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L314-L336)

## Context Reviewed

- Exact source pin and `configure.in` version; version/global indexes and the existing health checklist for navigation only. No evidence was taken from another PostgreSQL version.
- Disk-read boundary (`ReadBuffer`, buffer hit/miss, `smgrread`/`mdread`), `PageHeaderData`, checksum gate, all-zero acceptance, explicit zero-on-error callers, and their exceptional behavior.
- Normal access-method readers: B-tree, GiST, hash, BRIN, GIN, and SP-GiST; their structural error branches and relevant adjacent normal branches.
- TOAST external pointers, ordered index retrieval, chunk validation, slice variants, and compressed-value decoding; VACUUM's freeze caller and tuple/catalog horizon checks; relation descriptor construction and its attribute scan.
- SLRU I/O reporting, MultiXact retained/allocation bounds, tuple-driven and SQL-facing callers, allocator checks, unique-index rejection, checkpoint synchronization, and child-process/crash reporting.
- WAL record structures and validation, `ReadRecord` reporting, checkpoint failure, separate control-file checksum, recovery invalid-page tracking/removal, and repeated-report suppression.
- Error-report fields, severity promotion, default SQLSTATEs, generated header ownership, file-error classification, server/client filtering, logging-hook boundary, text/CSV formats, GUC defaults and contexts.
- Separate contrib `amcheck` build and SQL registration, function options and locks, diagnostic limits, regression inputs, and the offline checksum utility's test boundary.
- Relevant shared glossary entries; PostgreSQL 12 has no common-concept pages to link for these mechanisms. No common-concept document was created or changed.

## Evidence Map

| Claim group | Primary evidence |
|---|---|
| Disabled checksums leave structural checks active; silence is not proof | [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L161) |
| Buffer lifecycle, caller mode, and invalid-page report | [bufmgr.c#read-modes](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L590-L635) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L760-L795) [bufmgr.c#verification-result](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) |
| Full block versus OS failure/short read | [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L629) |
| Shared message text across B-tree/GiST/hash, distinct checks in other methods | [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L719) [gistutil.c#gistcheckpage](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L767-L798) [hashutil.c#_hash_checkpage](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L219-L285) [brin_revmap.c#range-map-checks](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L248-L288) [ginbtree.c#ginStepRight](../../../../raw/postgres-12/src/backend/access/gin/ginbtree.c#L165-L190) [spgscan.c#spgTestLeafTuple](../../../../raw/postgres-12/src/backend/access/spgist/spgscan.c#L715-L752) |
| TOAST fetch errors describe returned chunks, with an additional decompression check | [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1890-L2016) [tuptoaster.c#toast_decompress_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2261-L2281) |
| Tuple horizons and catalog mismatch | [heapam.c#heap_prepare_freeze_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6142-L6163) [relcache.c#RelationBuildTupleDesc](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L520-L561) [relcache.c#attribute-count](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L639-L652) |
| Transaction file, bounds, and generic I/O interpretation | [slru.c#SlruReportIOError](../../../../raw/postgres-12/src/backend/access/transam/slru.c#L893-L959) [multixact.c#GetMultiXactIdMembers](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L1250-L1286) [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L584-L648) |
| Independent WAL/control checks and contextual reporting | [xlogreader.c#record-validation](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L638-L726) [xlog.c#ReadRecord](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4240-L4284) [xlog.c#control-file-checksum](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4638-L4647) [xlog.c#emode_for_corrupt_record](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L12126-L12156) |
| Recovery references have a lifecycle before the panic | [xlogutils.c#log_invalid_page](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L85-L138) [xlogutils.c#forget_invalid_pages](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L140-L171) [xlogutils.c#drop-and-truncate-callers](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L599-L640) [xlogutils.c#XLogCheckInvalidPages](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L218-L243) |
| Options and limits of explicit integrity checking | [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L218) [verify_nbtree.c#item-order](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1184-L1198) [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1896-L1919) [amcheck.sgml#detection-limits](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L314-L336) |
| Report defaults, formatting, and filters | [elog.c#default-sqlstate](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388) [elog.c#text-log-fields](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L2860-L2933) [elog.c#is_log_level_output](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L3430-L3456) [guc.c#logging-levels](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4286-L4315) |

## Open Questions

- The source catalog is deliberately bounded. It does not enumerate every error in extensions, user-defined types/operator classes, procedural code, or every core corruption path. A concrete incident outside these templates needs its own caller/source trace.
- No fault-injection or runtime log-reproduction matrix was run. The message templates, levels, codes, and conditions are established from source; environment-specific formatting, translation, hooks, and log collection need confirmation on the affected installation.
- A failing check establishes the inconsistency stated in its row. Determining whether the cause was storage, memory, a software defect, manual file changes, or an inconsistent restore needs incident evidence that this page does not have. The same-checkout diagnostic discussion warns against assuming a universal repair. [amcheck.sgml#diagnosis](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L300-L336)
- PostgreSQL 12 has no matching common-concept pages for page validation, TOAST integrity, or recovery invalid-page tracking. Those would require separate authorized work; this page carries the necessary local explanation.

## Source References

- [configure.in:20](../../../../raw/postgres-12/configure.in#L20)
- [contrib/amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10)
- [contrib/amcheck/amcheck--1.0.sql](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L1-L22)
- [contrib/amcheck/amcheck--1.1--1.2.sql](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.1--1.2.sql#L12-L19)
- [contrib/amcheck/sql/check_btree.sql](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L21-L27)
- [contrib/amcheck/verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L218)
- [doc/src/sgml/amcheck.sgml](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L300-L336)
- [src/backend/access/brin/brin_revmap.c](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L248-L288)
- [src/backend/access/gin/ginbtree.c](../../../../raw/postgres-12/src/backend/access/gin/ginbtree.c#L165-L190)
- [src/backend/access/gist/gistutil.c](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L767-L798)
- [src/backend/access/hash/hashutil.c](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L219-L285)
- [src/backend/access/heap/heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6142-L6163)
- [src/backend/access/heap/tuptoaster.c](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L2016)
- [src/backend/access/heap/vacuumlazy.c](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1186-L1195)
- [src/backend/access/heap/visibilitymap.c](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L607-L609)
- [src/backend/access/nbtree/nbtinsert.c](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L563-L570)
- [src/backend/access/nbtree/nbtpage.c](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L719)
- [src/backend/access/spgist/spgscan.c](../../../../raw/postgres-12/src/backend/access/spgist/spgscan.c#L715-L752)
- [src/backend/access/transam/multixact.c](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L1250-L1279)
- [src/backend/access/transam/slru.c](../../../../raw/postgres-12/src/backend/access/transam/slru.c#L893-L959)
- [src/backend/access/transam/xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826)
- [src/backend/access/transam/xlogreader.c](../../../../raw/postgres-12/src/backend/access/transam/xlogreader.c#L696-L726)
- [src/backend/access/transam/xlogutils.c](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L218-L243)
- [src/backend/postmaster/postmaster.c](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3656-L3686)
- [src/backend/storage/buffer/bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L590-L598)
- [src/backend/storage/file/fd.c](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L3475-L3490)
- [src/backend/storage/freespace/freespace.c:598](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L598)
- [src/backend/storage/page/bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L119)
- [src/backend/storage/smgr/md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L629)
- [src/backend/storage/smgr/smgr.c](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L579-L591)
- [src/backend/storage/sync/sync.c](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L387-L401)
- [src/backend/utils/Makefile](../../../../raw/postgres-12/src/backend/utils/Makefile#L41-L42)
- [src/backend/utils/cache/relcache.c](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L639-L652)
- [src/backend/utils/errcodes.txt](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L431-L437)
- [src/backend/utils/error/elog.c](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L584-L590)
- [src/backend/utils/misc/guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3715-L3724)
- [src/backend/utils/mmgr/mcxt.c](../../../../raw/postgres-12/src/backend/utils/mmgr/mcxt.c#L764-L779)
- [src/bin/initdb/initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L139-L145)
- [src/bin/pg_checksums/t/002_actions.pl](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L51-L76)
- [src/include/access/xlogrecord.h](../../../../raw/postgres-12/src/include/access/xlogrecord.h#L41-L55)
- [src/include/catalog/pg_control.h](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L221-L232)
- [src/include/postgres.h](../../../../raw/postgres-12/src/include/postgres.h#L55-L73)
- [src/include/storage/bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164)
- [src/include/utils/elog.h](../../../../raw/postgres-12/src/include/utils/elog.h#L346-L365)
- [src/include/utils/guc.h](../../../../raw/postgres-12/src/include/utils/guc.h#L39-L80)
- [src/port/snprintf.c](../../../../raw/postgres-12/src/port/snprintf.c#L65-L71)

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Wiki index](../../../index.md)
- [Version pins](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [PostgreSQL 12 Database Health Checklist (unverified)](../server-administration/database-health-checklist.md)
