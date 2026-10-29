---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Which Client Statements Set Hint Bits and Dirty Pages as Hints in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Main idea](#main-idea)
  - [Values that decide the outcome](#values-that-decide-the-outcome)
  - [Logic map](#logic-map)
  - [Statements and the paths that reach a hint](#statements-and-the-paths-that-reach-a-hint)
  - [When a visibility check sets a tuple hint](#when-a-visibility-check-sets-a-tuple-hint)
  - [Reads that do not set tuple hints](#reads-that-do-not-set-tuple-hints)
  - [What the hint does to the buffer and WAL](#what-the-hint-does-to-the-buffer-and-wal)
  - [Lifecycle events that change the outcome](#lifecycle-events-that-change-the-outcome)
  - [How to see it happen](#how-to-see-it-happen)
  - [Measured results](#measured-results)
  - [Causal summary](#causal-summary)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Last run](#last-run)
  - [Stages](#stages)
  - [Non-default settings in the measurement](#non-default-settings-in-the-measurement)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, what client operation or statement leads to the following?

> [Hint bits](../../../glossary.md#hint-bits) cache facts such as a tuple's committed status. A hint change can modify page bytes even when the user only reads rows. With both checksums and `wal_log_hints` off, that does not take the checksum-protection WAL branch. Enabling checksums makes `XLogHintBitIsNeeded()` true regardless of `wal_log_hints`, whose default remains off. It does not turn the displayed `wal_log_hints` setting on. [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L111-L135) [xlog.h#predicate](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192) [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185).

## Answer

### Main idea

**No single statement type triggers a hint. A hint is set by the first statement that runs a visibility check on a [tuple](../../../glossary.md#tuple) after the transaction that inserted or deleted it has finished, and only once the finished transaction is outside the checker's [snapshot](../../../glossary.md#snapshot). A plain `SELECT` does this, which is why the quoted paragraph says a hint can change page bytes "even when the user only reads rows".**

Every heap access path in the pinned source calls a `HeapTupleSatisfies*` routine. When one of these routines learns that a transaction committed or aborted, it calls `SetHintBits()`. `SetHintBits()` ORs the bit into the tuple header's `t_infomask` and calls `MarkBufferDirtyHint()` ([heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131), [heapam_visibility.c#HeapTupleSatisfiesVisibility](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1679-L1718)). So the statements that set hints are the ones that read heap tuples: `SELECT`, `COPY ... TO`, `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE/SHARE`, foreign-key checks, unique-index checks during `INSERT`, `VACUUM`, `ANALYZE`, `CREATE INDEX`, `CLUSTER` and `VACUUM FULL`, plus the catalog lookups that any statement makes. The table in [Statements and the paths that reach a hint](#statements-and-the-paths-that-reach-a-hint) gives the path for each one.

`MarkBufferDirtyHint()` is also called by other "hint" writes that are not tuple commit bits: index scans marking dead index entries `LP_DEAD`, opportunistic pruning updating `pd_prune_xid`, free-space-map updates, and a few rarer cases. These take the same WAL branch ([bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3405-L3553)).

The quoted WAL branch is a separate, later step. By default, with [data checksums](../../../glossary.md#data-checksums) off and `wal_log_hints` off, the hint only makes the buffer [dirty](../../../glossary.md#dirty-buffer), and no [WAL](../../../glossary.md#wal) is written for it. When checksums are on (or `wal_log_hints` is on), the first hint on a clean, permanent page since the last [checkpoint](../../../glossary.md#checkpoint) writes an `XLOG_FPI_FOR_HINT` [full-page image](../../../glossary.md#full-page-image) ([bufmgr.c:3470-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3507), [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L893-L957)). That branch is non-default behavior, and [What the hint does to the buffer and WAL](#what-the-hint-does-to-the-buffer-and-wal) gives its default, cause and mechanism. On a 12.2 build from the pin, the first `SELECT` over a freshly loaded 94-page table dirtied all 94 pages in every configuration. It wrote 0 bytes of WAL by default, but 94 hint images (765,400 bytes) with checksums on; see [Measured results](#measured-results).

### Values that decide the outcome

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| `HEAP_XMIN_COMMITTED`, `HEAP_XMIN_INVALID`, `HEAP_XMAX_COMMITTED`, `HEAP_XMAX_INVALID` | The hint bits: cached outcome of the inserting ([xmin](../../../glossary.md#xmin-and-xmax)) and deleting (xmax) transactions | `t_infomask` in the tuple header on the page ([htup_details.h:203-207](../../../../raw/postgres-12/src/include/access/htup_details.h#L203-L207)) | Stored on the page | A set bit lets later checks skip the commit-log lookup; it also means no new hint is needed ([heapam_visibility.c:970](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L970)) |
| Transaction outcome | Whether xmin or xmax committed, aborted, or is still running | `TransactionIdDidCommit()` and related lookups inside the visibility routine ([heapam_visibility.c:1056-1067](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1056-L1067)) | Live | Decides which bit, if any, can be set |
| The checker's snapshot | Which transactions the statement treats as still running | The executor's snapshot, or a catalog, dirty, or vacuum horizon depending on the caller | Live, per statement or scan | An MVCC reader sets no hint for a transaction its snapshot still treats as running ([heapam_visibility.c:1056-1057](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1056-L1057)) |
| Commit LSN | The [WAL](../../../glossary.md#wal) position of the transaction's commit record | `TransactionIdGetCommitLSN()` | Live | `SetHintBits()` skips a committed hint on a permanent buffer while that LSN is not flushed and the page LSN is older ([heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131)) |
| `PD_ALL_VISIBLE` | Page-level flag saying every tuple is visible to everyone | Page header | Stored | A non-recovery sequential scan skips per-tuple checks on such a page ([heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L427-L447)) |
| `BM_DIRTY`, `BM_JUST_DIRTIED` | The buffer already holds unwritten changes | Shared buffer header | Live | If both are set, `MarkBufferDirtyHint()` returns at once ([bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454)) |
| `BM_PERMANENT` | The buffer belongs to a permanent relation, or to any relation's init fork | Set when the buffer is assigned ([bufmgr.c:1303-1313](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1303-L1313)) | Stored in the buffer header | Only permanent buffers take the hint-WAL branch ([bufmgr.c:3470-3471](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3471)) |
| `XLogHintBitIsNeeded()` | `DataChecksumsEnabled() \|\| wal_log_hints` | `pg_control`'s `data_checksum_version`, and the GUC ([xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192), [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826)) | Checksums: stored in `pg_control`. GUC: fixed at server start | Selects the WAL branch |
| Page LSN and `RedoRecPtr` | The page's last WAL position, and the redo start of the latest checkpoint | Page header; `GetRedoRecPtr()` | Page LSN stored on page, redo pointer live | An image is written only if page LSN ≤ `RedoRecPtr` ([xloginsert.c:907-917](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L907-L917)) |

Two values with similar roles come from different places. The tuple's hint bits are stored on the page, while the transaction outcome they cache is a live lookup in the commit log. A hint is only a cached copy of that live outcome. That is why losing a hint is harmless and why the code may skip it.

### Logic map

```text
Client statement (SELECT, UPDATE, DELETE, FOR UPDATE, INSERT with unique index,
                  VACUUM, ANALYZE, CREATE INDEX, CLUSTER, catalog lookup, ...)
  │
  ├─[A] heap access path reads a page into a buffer
  │     ├─ heap page access by seq / index / bitmap scan → heap_page_prune_opt  ── [P]
  │     ├─ seq scan, page PD_ALL_VISIBLE and snapshot not taken in recovery
  │     │     → no per-tuple check, no tuple hint
  │     └─ otherwise → HeapTupleSatisfies<kind>(tuple, snapshot, buffer)
  │
  ├─[B] visibility routine, per tuple
  │     ├─ hint bit already set, or tuple frozen        → no hint needed
  │     ├─ xid is the current transaction / in progress  → no hint
  │     ├─ MVCC: xid still in the reader's snapshot      → no hint
  │     ├─ Any / HistoricMVCC / Toast (normal case)      → never sets a hint
  │     └─ outcome known (committed or aborted)          → SetHintBits
  │
  ├─[C] SetHintBits
  │     ├─ committed hint, permanent buffer, commit LSN not flushed,
  │     │  page LSN < commit LSN                         → skip (asynchronous commit)
  │     └─ else t_infomask |= bit; MarkBufferDirtyHint(buffer, true)
  │
  │  Other hint writers also call MarkBufferDirtyHint:
  │     [I] index scan kills dead entries (LP_DEAD)   [P] pd_prune_xid update
  │     [F] free space map update                     [S] sequence / BRIN / btree cycle id
  │
  └─[D] MarkBufferDirtyHint
        ├─ local (temporary-table) buffer      → mark local buffer dirty, no WAL
        ├─ BM_DIRTY and BM_JUST_DIRTIED set    → return, nothing else
        ├─ XLogHintBitIsNeeded() and BM_PERMANENT
        │     ├─ RecoveryInProgress()          → return: bit set in memory, page NOT dirtied
        │     └─ normal running → delayChkpt; XLogSaveBufferForHint
        │            ├─ page LSN ≤ RedoRecPtr → XLOG_FPI_FOR_HINT image, page LSN := image LSN
        │            └─ page LSN > RedoRecPtr → no WAL record
        │         then mark dirty
        └─ default (checksums off, wal_log_hints off), or unlogged buffer
                                               → mark dirty, no WAL
```

Key to the map:

- [A] [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L352-L460), [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L117-L165), [heapam_handler.c#heapam_scan_bitmap_next_block](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2180-L2290).
- [B] [heapam_visibility.c#HeapTupleSatisfiesMVCC](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L962-L1147), [heapam_visibility.c#HeapTupleSatisfiesAny](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L338-L342), [heapam_visibility.c#HeapTupleSatisfiesToast](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L360-L425), [heapam_visibility.c#HeapTupleSatisfiesHistoricMVCC](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1541-L1677).
- [C] [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131).
- [D] [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3422-L3553), [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L893-L957).
- [I] [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1718-L1814), [hashutil.c:612-622](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L612-L622), [gistget.c:75-91](../../../../raw/postgres-12/src/backend/access/gist/gistget.c#L75-L91).
- [P] [pruneheap.c:276-291](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L276-L291).
- [F] [freespace.c#fsm_set_and_search](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L671-L688).
- [S] [sequence.c:1194-1209](../../../../raw/postgres-12/src/backend/commands/sequence.c#L1194-L1209), [brin_pageops.c:540-548](../../../../raw/postgres-12/src/backend/access/brin/brin_pageops.c#L540-L548), [nbtree.c:1310-1323](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1310-L1323).

### Statements and the paths that reach a hint

Each row names a client operation, the code path it takes, and which hint it can set. "Tuple hint" means a `t_infomask` commit or abort bit set through `SetHintBits()`.

| Client operation | Path in the pinned source | Hint it can set |
|---|---|---|
| `SELECT` with a sequential scan (also `COPY table TO`, which reads through `table_beginscan()` and `table_scan_getnextslot()` ([copy.c:2119-2123](../../../../raw/postgres-12/src/backend/commands/copy.c#L2119-L2123))) | `SeqNext()` → `table_scan_getnextslot()` → `heap_getnextslot()` → `heapgettup_pagemode()` → `heapgetpage()`, which calls `HeapTupleSatisfiesVisibility()` per tuple unless the page is all-visible ([nodeSeqscan.c#SeqNext](../../../../raw/postgres-12/src/backend/executor/nodeSeqscan.c#L50-L85), [heapam.c#heap_getnextslot](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1349-L1373), [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L427-L450)) | Tuple hints via `HeapTupleSatisfiesMVCC()`. Also `pd_prune_xid` through `heap_page_prune_opt()` ([heapam.c:394](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L394)) |
| `SELECT` with an index scan | `IndexNext()` → `index_getnext_slot()` → `index_fetch_heap()` → `heapam_index_fetch_tuple()` → `heap_hot_search_buffer()` ([nodeIndexscan.c#IndexNext](../../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L81-L133), [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L589), [heapam.c:1612](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1612)) | Tuple hints; `pd_prune_xid`; and an `LP_DEAD` mark on the index entry when the whole HOT chain is dead to everyone ([heapam.c:1635-1637](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1635-L1637), [indexam.c:578-586](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L578-L586), [nbtree.c:253-266](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L253-L266)) |
| `SELECT` with an index-only scan | Visits the heap only when the [visibility map](../../../glossary.md#visibility-map) bit for the page is clear ([nodeIndexonlyscan.c:159-170](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L159-L170)) | Same as an index scan, but only for those heap visits |
| `SELECT` with a bitmap heap scan | `heapam_scan_bitmap_next_block()`: exact pages use `heap_hot_search_buffer()` with no `all_dead` output; lossy pages call `HeapTupleSatisfiesVisibility()` per tuple ([heapam_handler.c:2228-2275](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2228-L2275)) | Tuple hints and `pd_prune_xid` ([heapam_handler.c:2216](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2216)); no index `LP_DEAD` marks |
| `UPDATE`, `DELETE` | First the scan that finds the rows (rows above). Then `heap_update()` / `heap_delete()` call `HeapTupleSatisfiesUpdate()` on the target, and `UpdateXmaxHintBits()` after waiting for a concurrent updater ([nodeModifyTable.c:768](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L768), [nodeModifyTable.c:1311](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L1311), [heapam.c:2512](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2512), [heapam.c:2610](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2610), [heapam.c:3068](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3068), [heapam.c:3244](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3244)) | Tuple hints on the old row version, including `HEAP_XMAX_INVALID` for a lock-only or aborted xmax ([heapam.c#UpdateXmaxHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1780-L1808)) |
| `SELECT ... FOR UPDATE / NO KEY UPDATE / SHARE / KEY SHARE`, and foreign-key checks | `ExecLockRows()` → `table_tuple_lock()` → `heap_lock_tuple()` → `HeapTupleSatisfiesUpdate()` and `UpdateXmaxHintBits()`. The referential-integrity check query uses `FOR KEY SHARE`, so FK checks on `INSERT`/`UPDATE` of the referencing table take this path ([nodeLockRows.c:182](../../../../raw/postgres-12/src/backend/executor/nodeLockRows.c#L182), [heapam_handler.c:371](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L371), [heapam.c:4017](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L4017), [heapam.c:4473](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L4473), [ri_triggers.c:384](../../../../raw/postgres-12/src/backend/utils/adt/ri_triggers.c#L384)) | Tuple hints on the locked row |
| `INSERT` or `UPDATE` into a table with a unique B-tree index | `_bt_check_unique()` fetches conflicting heap tuples with a dirty snapshot and marks an index entry `LP_DEAD` when its HOT chain is dead to everyone ([nbtinsert.c:252](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L252), [nbtinsert.c:349-360](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L349-L360), [nbtinsert.c:575-590](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L575-L590)) | Tuple hints via `HeapTupleSatisfiesDirty()` ([heapam_visibility.c#HeapTupleSatisfiesDirty](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L744-L937)); index `LP_DEAD` marks |
| `INSERT`, `UPDATE`, `COPY FROM` choosing a target page | `RelationGetBufferForTuple()` consults the [free space map](../../../glossary.md#free-space-map); `RecordAndGetPageWithFreeSpace()` → `fsm_set_and_search()` marks the FSM page with a hint ([hio.c:533](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L533), [freespace.c:149-161](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L149-L161), [freespace.c:671-688](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L671-L688)) | FSM page hint (no tuple hint) |
| `VACUUM` | `lazy_scan_heap()` and the index-cleanup pass call `HeapTupleSatisfiesVacuum()`; heap pruning calls it too. VACUUM also updates the FSM and clears B-tree cycle IDs as hints ([vacuumlazy.c:1051](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1051), [pruneheap.c:488](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L488), [nbtree.c:1310-1323](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1310-L1323)) | Tuple hints via [HeapTupleSatisfiesVacuum](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1163-L1383); FSM and B-tree page hints |
| `ANALYZE` | `heapam_scan_analyze_next_tuple()` → `HeapTupleSatisfiesVacuum()` on sampled pages ([heapam_handler.c:1050](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1050)) | Tuple hints on sampled pages |
| `CREATE INDEX`, `REINDEX` | `heapam_index_build_range_scan()` → `HeapTupleSatisfiesVacuum()` ([heapam_handler.c:1387](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1387)) | Tuple hints on the old heap |
| `CLUSTER`, `VACUUM FULL` | `heapam_relation_copy_for_cluster()` → `HeapTupleSatisfiesVacuum()` on the old heap ([heapam_handler.c:828](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L828)) | Tuple hints on the old heap |
| Any statement that misses the [syscache](../../../glossary.md#syscache) | `systable_beginscan()` scans the [catalog](../../../glossary.md#catalog) with a catalog snapshot ([genam.c:352-378](../../../../raw/postgres-12/src/backend/access/index/genam.c#L352-L378)) | Tuple hints on catalog heaps (MVCC path) |
| `nextval()` on a sequence whose tuple carries a stale xmax | `read_seq_tuple()` clears the xmax and marks the buffer with a hint; the comment calls this cleanup after an old-release `SELECT FOR UPDATE` on a sequence ([sequence.c:1194-1209](../../../../raw/postgres-12/src/backend/commands/sequence.c#L1194-L1209)) | Sequence page hint (rare) |
| A BRIN insert or summarization that must extend the range map | `revmap_physical_extend()` → `brin_start_evacuating_page()` sets `BRIN_EVACUATE_PAGE` with a hint ([brin_revmap.c:607](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L607), [brin_pageops.c:540-548](../../../../raw/postgres-12/src/backend/access/brin/brin_pageops.c#L540-L548)) | BRIN page hint (rare) |

The `LP_DEAD` index mark is described in the glossary as a [killed index tuple](../../../glossary.md#killed-index-tuple). Opportunistic [pruning](../../../glossary.md#pruning) is two different writes. A real prune is WAL-logged as its own record, so it does not use the hint branch. Only the "nothing pruned, but `pd_prune_xid` changed or the page-full flag was set" case is treated as a non-WAL-logged hint ([pruneheap.c:276-291](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L276-L291)). `heap_page_prune_opt()` does nothing during recovery ([pruneheap.c:82-89](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L82-L89)).

### When a visibility check sets a tuple hint

A statement reaching a visibility routine does not always set a hint. The routine for the reader's snapshot type must reach a branch that calls `SetHintBits()`, and `SetHintBits()` must accept it.

**Input.** One tuple, one snapshot, one buffer that the caller holds with at least a share lock ([heapam_visibility.c:1679-1688](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1679-L1688)).

**Decision 1: is the bit already there?** `HeapTupleSatisfiesMVCC()` first tests `HeapTupleHeaderXminCommitted()`. If the xmin hint is already set, it skips the xmin lookup entirely ([heapam_visibility.c:970](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L970)). So only the first qualifying check after a transaction ends pays for a hint. Later readers find the bit and do nothing.

**Decision 2: does the reader's snapshot still see the transaction as running?** For an MVCC snapshot, a transaction ID that is in the snapshot returns "not visible" before any commit-log lookup, so no hint is set ([heapam_visibility.c:1056-1057](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1056-L1057)). The same holds for xmax ([heapam_visibility.c:1122-1123](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1122-L1123)). Therefore a long-running reader whose snapshot predates a commit does not hint that commit. A statement that starts after the commit does.

**Decision 3: what is the outcome?** A committed xmin gets `HEAP_XMIN_COMMITTED`, an aborted or crashed one gets `HEAP_XMIN_INVALID`. For xmax, an aborted deleter gets `HEAP_XMAX_INVALID` and a committed one gets `HEAP_XMAX_COMMITTED` ([heapam_visibility.c:1058-1067](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1058-L1067), [heapam_visibility.c:1125-1135](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1125-L1135)). A multixact xmax gets no hint in this routine ([heapam_visibility.c:1085-1110](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1085-L1110)).

**Decision 4: is the commit durable yet?** For a committed hint, `SetHintBits()` reads the commit LSN. If the buffer is permanent, that LSN still needs a WAL flush, and the page LSN is older, it returns without setting the bit. This happens with [asynchronous commit](../../../glossary.md#asynchronous-commit). Abort hints never wait ([heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131), comment at [heapam_visibility.c:81-111](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L81-L111)).

**Output.** The bit is ORed into `t_infomask` under the caller's share lock, and `MarkBufferDirtyHint(buffer, true)` runs ([heapam_visibility.c:129-130](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L129-L130)). **Consequence:** the page bytes in shared memory have changed, even though the statement only read rows. What happens to the buffer and WAL next is decided in `MarkBufferDirtyHint()`.

`HeapTupleSatisfiesUpdate()`, `HeapTupleSatisfiesDirty()`, `HeapTupleSatisfiesSelf()` and `HeapTupleSatisfiesVacuum()` follow the same pattern for their own rules. They have no snapshot test, so they hint any finished transaction they look up ([heapam_visibility.c#HeapTupleSatisfiesUpdate](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L457-L722), [heapam_visibility.c#HeapTupleSatisfiesSelf](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L169-L332)). `HeapTupleSatisfiesNonVacuumable()` calls `HeapTupleSatisfiesVacuum()`, so it hints too ([heapam_visibility.c#HeapTupleSatisfiesNonVacuumable](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1398-L1404)).

### Reads that do not set tuple hints

| Condition | Behavior | Consequence |
|---|---|---|
| Sequential scan of a page with `PD_ALL_VISIBLE`, snapshot not taken during recovery | `heapgetpage()` sets `valid = true` without calling a visibility routine ([heapam.c:427-447](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L427-L447)) | No tuple hint. `pd_prune_xid` can still change through `heap_page_prune_opt()`, which runs before this test ([heapam.c:394](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L394)) |
| Index-only scan, page's visibility-map bit set | The heap is not visited ([nodeIndexonlyscan.c:159-170](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L159-L170)) | No heap access, no heap hint |
| Hint already set, or tuple frozen | The routine tests the bit first ([heapam_visibility.c:970](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L970), [heapam_visibility.c:1070-1074](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1070-L1074)) | No change |
| Transaction still running, or in the MVCC reader's snapshot | Returns early ([heapam_visibility.c:1014-1057](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1014-L1057)) | No hint until a later check |
| Committed but unflushed asynchronous commit on a permanent page | `SetHintBits()` returns early ([heapam_visibility.c:117-126](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L117-L126)) | No hint until the WAL is flushed and a later check runs |
| `SnapshotAny` reads | `HeapTupleSatisfiesAny()` just returns true ([heapam_visibility.c#HeapTupleSatisfiesAny](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L338-L342)) | No hint |
| Logical decoding catalog reads | `HeapTupleSatisfiesHistoricMVCC()` contains no `SetHintBits()` call ([heapam_visibility.c#HeapTupleSatisfiesHistoricMVCC](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1541-L1677)) | No tuple hint |
| Detoasting a value | `toast_fetch_datum()` reads chunks with a TOAST snapshot ([tuptoaster.c:1730-1764](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1730-L1764), [snapmgr.h:92-95](../../../../raw/postgres-12/src/include/utils/snapmgr.h#L92-L95)); `HeapTupleSatisfiesToast()` sets hints only on the pre-9.0 `HEAP_MOVED_OFF`/`HEAP_MOVED_IN` branches ([heapam_visibility.c#HeapTupleSatisfiesToast](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L360-L425)) | No tuple hint on TOAST chunks in the normal case. The TOAST index scan can still prune the TOAST heap page or kill a dead index entry, because it goes through `index_fetch_heap()` |

### What the hint does to the buffer and WAL

**Default.** With checksums off (the `initdb` default: `data_checksums = false` unless `-k`/`--data-checksums` is given) and `wal_log_hints = off` (its boot value), `XLogHintBitIsNeeded()` is false ([initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [initdb.c:3146-3148](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3146-L3148), [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185), [xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192)). `MarkBufferDirtyHint()` then only sets `BM_DIRTY | BM_JUST_DIRTIED` and counts the page as dirtied. No WAL record is written for the hint ([bufmgr.c:3509-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3509-L3546)). The background writer, the checkpointer, or a backend evicting the buffer later writes the page with the hint.

**Cause of the WAL branch.** The cluster was initialized with data checksums, so `pg_control` has a non-zero `data_checksum_version`, or the server was started with `wal_log_hints = on`. `wal_log_hints` has context `PGC_POSTMASTER`, so it needs a restart ([guc.c:1178](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1178)). The checksum state comes from `pg_control`, not from a GUC ([xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826)).

**Mechanism.** `MarkBufferDirtyHint()` evaluates `XLogHintBitIsNeeded() && BM_PERMANENT` ([bufmgr.c:3470-3471](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3471)). When true and not in recovery, it sets `delayChkpt` and calls `XLogSaveBufferForHint()` ([bufmgr.c:3506-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3506-L3507)). That function copies the page and inserts an `XLOG_FPI_FOR_HINT` record only if the page LSN is at or before the current `RedoRecPtr`, meaning this is the page's first WAL-protected change since the checkpoint began ([xloginsert.c:907-954](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L907-L954)). The block is registered with `REGBUF_FORCE_IMAGE`, and `XLogRecordAssemble()` tests that flag before `doPageWrites`. So the image is written even when `full_page_writes` is off ([xloginsert.c:944-947](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L944-L947), [xloginsert.c:539-545](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L539-L545)). The reason is the torn-page problem: a hint changes a few bits, and any mix of old and new bits is a valid page, but the checksum covers the whole page and would no longer match ([xlog.h:183-192](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192), [storage/page/README:36-45](../../../../raw/postgres-12/src/backend/storage/page/README#L36-L45)).

Enabling checksums makes the predicate true through `DataChecksumsEnabled()`. It does not change the `wal_log_hints` variable, so `SHOW wal_log_hints` still reports `off`. The documentation says the setting "is ignored" when checksums are on ([config.sgml:2748-2753](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2748-L2753)).

| Condition / state | Behavior | Consequence |
|---|---|---|
| Temporary table (local buffer) | `MarkLocalBufferDirty()` and return ([bufmgr.c:3430-3434](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3430-L3434)) | Never hint WAL |
| Unlogged table (not `BM_PERMANENT`), main fork | Skips the WAL branch, marks dirty ([bufmgr.c:1312-1313](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1312-L1313), [bufmgr.c:3470-3471](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3471)) | No hint WAL even with checksums |
| Buffer already `BM_DIRTY` and `BM_JUST_DIRTIED` | Fast return ([bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454)) | No extra WAL; the earlier change already carried the image or the page is already dirty |
| Checksums or `wal_log_hints` on, page LSN > `RedoRecPtr` | No record; marks dirty ([xloginsert.c:917](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L917)) | A logged change since the checkpoint already protects the page |
| Checksums or `wal_log_hints` on, page LSN ≤ `RedoRecPtr`, normal running | `XLOG_FPI_FOR_HINT` written, page LSN advanced, then dirtied | A read-only `SELECT` generates WAL |
| Checksums or `wal_log_hints` on, [hot standby](../../../glossary.md#hot-standby) reader | Returns before dirtying ([bufmgr.c:3480-3481](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3480-L3481)) | Bit is set in memory but the page is not dirtied, so the hint is lost at eviction |
| Both off, hot standby reader | Takes the default dirtying path | The standby writes the hinted page to its own files; no WAL |

### Lifecycle events that change the outcome

| Event | State before | State after | Effect on later calculation |
|---|---|---|---|
| Inserting or deleting transaction commits or aborts | No hint possible | Outcome known in the commit log | The next qualifying check sets the hint |
| Commit record flushed (asynchronous commit) | Committed hint refused on permanent pages | `XLogNeedsFlush()` false | Committed hint allowed ([heapam_visibility.c:117-126](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L117-L126)) |
| Checkpoint starts | Page LSN may be > old redo pointer | `RedoRecPtr` advances | The first hint on each clean permanent page again writes an image when the WAL branch is active |
| Buffer written out | `BM_DIRTY` set | `BM_DIRTY` cleared | The next hint on that buffer runs the full `MarkBufferDirtyHint()` path again |
| VACUUM sets the page all-visible and the visibility-map bit | Readers check each tuple | Sequential scans and index-only scans skip checks or the heap | Fewer hint opportunities. VACUUM marks a page all-visible only if every live tuple has `HEAP_XMIN_COMMITTED` ([vacuumlazy.c:1097-1114](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1097-L1114)) |
| Freezing | xmin needs a lookup | Both xmin bits set (frozen) | No further xmin hint ([htup_details.h:203-207](../../../../raw/postgres-12/src/include/access/htup_details.h#L203-L207)) |
| Restart with `wal_log_hints` changed | Old predicate | New predicate | Changes whether later hints write images |

### How to see it happen

A hint dirties the buffer through `MarkBufferDirtyHint()`, which increments `pgBufferUsage.shared_blks_dirtied` ([bufmgr.c:3539-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3539-L3546)). `EXPLAIN (ANALYZE, BUFFERS)` prints that counter as `dirtied=` on the shared-buffer line ([explain.c:2890-2902](../../../../raw/postgres-12/src/backend/commands/explain.c#L2890-L2902)). So a `SELECT` that reports `dirtied=` above zero changed pages, by hints, by `pd_prune_xid`, or by an index `LP_DEAD` mark. It can also be a WAL-logged prune, which counts as dirtied through the normal `MarkBufferDirty()` path ([bufmgr.c:1500](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1500)). With the WAL branch active, `pg_waldump` names the records `FPI_FOR_HINT` ([xlogdesc.c:185-186](../../../../raw/postgres-12/src/backend/access/rmgrdesc/xlogdesc.c#L185-L186)).

`EXPLAIN ANALYZE` runs the query. This example uses session-scoped timeouts and a tag comment:

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';
EXPLAIN /* wiki_hint_bit_dirtied_check */ (ANALYZE, BUFFERS) SELECT count(*) FROM my_table;
COMMIT;
```

`statement_timeout` and `lock_timeout` have context `PGC_USERSET`, so `SET LOCAL` applies them to this transaction only ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)). Replace `my_table` with your table name. [Measured results](#measured-results) shows this output for each case.

### Measured results

The [Measurement Script](#measurement-script) built 12.2 from the pin and ran the same six cases on three clusters. **`df`** is the default cluster: checksums off, `wal_log_hints` off. **`ck`** was initialized with `initdb -k`, so data checksums are on. **`wl`** has checksums off and `wal_log_hints = on`. Each case runs one read-only `EXPLAIN (ANALYZE, BUFFERS) SELECT count(*)` on a 94-page table. It then counts the `FPI_FOR_HINT` records that `pg_waldump` finds between the WAL insert positions taken just before and just after that statement. Every leg also disables autovacuum, and case F turns `full_page_writes` off; [Non-default settings in the measurement](#non-default-settings-in-the-measurement) explains why each setting is needed and which results depend on it.

| Case | What happened before the measured `SELECT` | `dirtied` (all legs) | Hint images for the table: `df` | `ck` | `wl` |
|---|---|---|---|---|---|
| A, first read | `INSERT` of 1,400 rows committed, then a clean restart | 94 | 0 | 94 | 94 |
| B, second read | Case A's read | 0 | 0 | 0 | 0 |
| C, after `CHECKPOINT` | Case B, then `CHECKPOINT` | 0 | 0 | 0 | 0 |
| D, unlogged table, first read | Same insert into an `UNLOGGED` table, same restart | 94 | 0 | 0 | 0 |
| E, after a rolled-back `DELETE` | `BEGIN; DELETE FROM t; ROLLBACK;` then `CHECKPOINT` | 94 | 0 | 94 | 94 |
| F, `full_page_writes = off` | Reload with `full_page_writes = off`, new 94-page table, `CHECKPOINT` | 94 | 0 | 94 | 94 |

In every case, the count of all `FPI_FOR_HINT` records in the statement's window equals the table-only count. So a read-only statement wrote no hint image for any other relation. In `df`, the WAL insert position did not move at all during cases A, E and F (0 bytes). The 60 assertions in the script passed on every run.

What each case shows, against the source reading:

- **A: a plain `SELECT` changes page bytes in every leg.** It dirtied all 94 pages, because each page held committed rows without `HEAP_XMIN_COMMITTED` ([heapam_visibility.c:1058-1060](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1058-L1060)). Only `ck` and `wl` turned that into WAL, one image per page, because only there is `XLogHintBitIsNeeded()` true ([xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192)). The `ck` leg's platform record shows `wal_log_hints=off`, so checksums alone took the branch while the displayed setting stayed off.
- **B and C: a hint is set once.** The bits were already set, so no visibility routine called `SetHintBits()` ([heapam_visibility.c:970](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L970)). C shows that a checkpoint alone does not cause new images: the buffers were clean and past the redo pointer, but no hint changed.
- **D: an [unlogged](../../../glossary.md#unlogged-relation) table dirties pages but writes no image, even with checksums on.** Its buffers are not `BM_PERMANENT` ([bufmgr.c:1312-1313](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1312-L1313), [bufmgr.c:3470-3471](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3471)).
- **E: abort hints behave the same.** The aborted deleter left an xmax on every row, and the next read set `HEAP_XMAX_INVALID` on them, producing a new image per page in `ck` and `wl`.
- **F: the hint image does not depend on `full_page_writes`.** The wide WAL window contains an `FPW_CHANGE false` record in all three legs, so the setting really was off, yet `ck` and `wl` still wrote 94 images. This matches `REGBUF_FORCE_IMAGE` ([xloginsert.c:944-947](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L944-L947), [xloginsert.c:539-545](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L539-L545)), not the README wording in [Open Questions](#open-questions).

The same widened F window also caught writing statements producing hint images. In `ck` and `wl` it contained 5 `FPI_FOR_HINT` records for free-space-map forks: `pg_type`'s, `pg_attribute`'s (3), and the new table's. All carried the transaction ID of the `CREATE TABLE`/`INSERT` setup, not of the measured `SELECT`. This is the FSM-update path listed for `INSERT` in [Statements and the paths that reach a hint](#statements-and-the-paths-that-reach-a-hint). `df` had 0 such records.

**Worked calculation: the WAL a hinted `SELECT` writes.** These inputs come from the `ck` leg's case A dump and platform record:

| Input | Value | Source |
|---|---|---|
| Records in the window | 94 `FPI_FOR_HINT` | `ck-A_first_select.wal` |
| Total length of 93 of them | 8,173 bytes each (49-byte record plus the page image without its hole) | `len (rec/tot)` in the dump |
| Total length of the last one | 2,773 bytes | the table's partly filled last page has a larger free-space hole, which the image omits ([xloginsert.c:925-937](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L930-L947)) |
| Record alignment | each record's space is rounded up with `MAXALIGN` (8 bytes here) | [xlog.c:1255](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L1255), `max_data_alignment=8` |
| WAL page header | 24 bytes per 8 kB WAL page | [xlog_internal.h#XLogPageHeaderData](../../../../raw/postgres-12/src/include/access/xlog_internal.h#L36-L53), `wal_block_size=8192` |
| Measured insert-position difference | 765,400 bytes (765,376 for `ck` case E and `wl` cases A and E) | `pg_wal_lsn_diff` in the script; the same in all three runs |

Calculation:

1. Aligned record bytes: 93 × 8,176 + 2,776 = 760,368 + 2,776 = **763,144** bytes.
2. Remainder: 765,400 − 763,144 = **2,256** bytes = 94 × 24, the short headers of the 94 WAL pages the window spans. A 765,376-byte window leaves 2,232 bytes, which is 93 × 24. This is what one fewer page boundary gives, assuming the same record sizes, because the window started at a different offset within a WAL page. The script did not tabulate record sizes for those windows.
3. Result: a `SELECT` that reads 94 freshly committed pages writes about **765 kB of WAL** with checksums on, about 8.1 kB per page, which is close to one heap page per page read. With the defaults, the same `SELECT` writes **0 bytes**.

This is measured for this fixture only: 1,400 rows of about 530 bytes, all committed by one transaction, with every page clean and older than the last checkpoint. A table whose pages are already dirty, already hinted, all-visible, or newer than the redo pointer writes fewer images; see [What the hint does to the buffer and WAL](#what-the-hint-does-to-the-buffer-and-wal).

### Causal summary

**Initial state:** a transaction inserted, updated or deleted rows and then finished, and the tuples carry no hint bits yet.

**Event:** a later statement reads those tuples through a heap visibility check. Examples are a `SELECT` on a page that is not all-visible, an `UPDATE`/`DELETE`/`FOR UPDATE`, a unique or foreign-key check, `VACUUM`, `ANALYZE`, `CREATE INDEX`, `CLUSTER`, or a catalog lookup.

**Transformation:** the routine looks up the outcome. If the transaction is outside the reader's snapshot and, for a commit, its commit record is flushed, `SetHintBits()` ORs the bit into `t_infomask` and calls `MarkBufferDirtyHint()`. Index scans killing dead entries, pruning's `pd_prune_xid` update, and FSM updates call the same function.

**Interaction:** by default, with checksums and `wal_log_hints` both off, the buffer is just marked dirty. With checksums on, or `wal_log_hints` on after a restart, a permanent, clean page whose LSN predates the current checkpoint's redo pointer first gets an `XLOG_FPI_FOR_HINT` image, even with `full_page_writes` off. On a standby with that branch active, the page is not dirtied at all.

**Observed result:** a read-only statement changes page bytes, shows `dirtied=` in `EXPLAIN (BUFFERS)`, and with checksums on can generate WAL.

**Refresh:** the next checkpoint re-arms the image for each page, and VACUUM's all-visible marking and freezing remove the need for further tuple hints.

## Measurement Script

### Usage

| Item | Value |
|---|---|
| Purpose | Measures which reads set hint bits and when that writes WAL. It backs every number under [Measured results](#measured-results): pages dirtied per `SELECT`, `FPI_FOR_HINT` record counts, WAL bytes, the `FPW_CHANGE false` check, and the free-space-map hint images in the widened case-F window. |
| Invocation | From the repository root, extract the Bash fence below to `.wiki-runtime/tmp/hb12-run.sh` with the command after this table, then run `bash .wiki-runtime/tmp/hb12-run.sh [stage ...]`. Every build, data, socket and output file stays under `.wiki-runtime/tmp/hb12/`. |
| Stages | Default order: `build df ck wl report`. `stop` and `clean` tear down and are not in the default order. See [Stages](#stages). |
| Environment | `PORT12` (default `55443`) is the isolated port, `JOBS` (default `8`) the parallel build jobs, and `ROWS` (default `1400`) the rows per fixture table. The checkout, pin and sandbox paths are fixed in the script. |
| Prerequisites | A C toolchain, GNU make, bison and flex; readline and zlib development headers, because 12's `configure` enables both by default; the pinned checkout at `raw/postgres-12/`; about 500 MB of free disk. Locale `C`, encoding `UTF8`, set by the script. No extension is needed. |
| Output | Files under `.wiki-runtime/tmp/hb12/out/`. Read `results.txt` first: one line per leg and case with `leg|case|table|pages|dirtied|fpi_for_hint_table|fpi_for_hint_all|wal_bytes|fpw_change_false_wide|fsm_fpi_for_hint_wide`. Then `<leg>-platform.txt`, `<leg>-<case>.explain`, `<leg>-<case>.wal` (the statement's own WAL window), `<leg>-<case>.wide.wal` (widened for case F, otherwise the same window), and `<leg>-server.log`. |
| Runtime | On the 2026-10-29 host (Apple arm64, 10 cores, `JOBS=10`): about 100 seconds from an empty sandbox, most of it the build; about 14 seconds to re-run `df ck wl report` while the build is present. |
| Cleanup | Run `bash .wiki-runtime/tmp/hb12-run.sh stop clean` from the repository root. It stops any of the three clusters still running, checks that no `postmaster.pid`, process naming a data directory, socket or listener on `PORT12` remains, and deletes `.wiki-runtime/tmp/hb12/` with its build. Remove the extracted `.wiki-runtime/tmp/hb12-run.sh` afterwards. Each leg already stops its own cluster at the end of its stage. |

To extract the published script without editing it, run this from the repository root:

```sh
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/storage-and-vacuum/client-statements-that-set-hint-bits.md | sed '1d;$d' > .wiki-runtime/tmp/hb12-run.sh
bash -n .wiki-runtime/tmp/hb12-run.sh
bash .wiki-runtime/tmp/hb12-run.sh
```

### Last run

On 2026-10-29 the published script was extracted from this page with the command above and run from an empty sandbox (91 s with `JOBS=10`), then `df ck wl report` was re-run with the build present (14 s). This was on Darwin 27.0.0 arm64, against PostgreSQL 12.2 at pin `45b88269a353ad93744772791feb6d01bc7e1e42`, built with default configure options. Both runs passed all 60 assertions, and every `results.txt` line matched the two development runs made earlier that day. The platform records show `block_size=8192`, `wal_block_size=8192`, `max_data_alignment=8`, `shared_buffers=128MB` (chosen by `initdb`), `wal_level=replica` and `synchronous_commit=on` in all legs. They show `data_checksums=on`, `wal_log_hints=off` in `ck`; `data_checksums=off`, `wal_log_hints=on` in `wl`; and both off in `df`. The script's own `stop clean` then stopped nothing (each leg had already stopped its cluster), confirmed that no PID file, process, socket or listener remained, and deleted `.wiki-runtime/tmp/hb12/` with its build. The extracted script was removed.

### Stages

| Stage | What it does | Needs |
|---|---|---|
| `build` | Checks the pin, then configures 12.2 out of tree with default options under `$SANDBOX/build`, builds, and installs the server, `psql` and `pg_waldump`. Skipped when a matching build exists. | nothing |
| `df` | The default leg: `initdb` with checksums off, starts the cluster, creates fixtures, runs cases A to F, stops the cluster, then checks its assertions | `build` |
| `ck` | The same with `initdb -k` (data checksums on) | `build` |
| `wl` | The same with checksums off and `wal_log_hints = on` in `postgresql.conf` | `build` |
| `report` | Prints `results.txt` | any leg |
| `stop` | Stops any leg's cluster still running and checks that no PID file, process, socket or listener remains | none |
| `clean` | Runs `stop`, then deletes `$SANDBOX` | none |

Each leg stage is idempotent: it deletes and re-creates its own data directory and replaces its own lines in `results.txt`. Inside a leg, the cases run in a fixed order because each depends on the state the previous one left:

1. Create table `t` and unlogged table `u`, each with 1,400 rows; read the new catalog rows in another session; restart the server cleanly.
2. **A** reads `t`. **B** reads it again.
3. `CHECKPOINT`, then **C** reads `t`.
4. **D** reads `u` for the first time.
5. Delete all rows of `t` and roll back, `CHECKPOINT`, then **E** reads `t`.
6. Turn `full_page_writes` off, create and fill table `f`, read its catalog rows, `CHECKPOINT`, then **F** reads `f`. Turn `full_page_writes` back on.

Every table and database is disposable and exists only in the sandbox cluster.

### Non-default settings in the measurement

| Setting or state | Default at the pin | Value in the script, why, and which results depend on it |
|---|---|---|
| Data checksums (`ck` leg) | Off: `initdb` leaves `data_checksums = false` unless `-k`/`--data-checksums` is given ([initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [initdb.c:3146-3148](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3146-L3148)) | `initdb -k`, written into `pg_control` at cluster creation. This is the cause the question asks about. The `ck` column's image counts and WAL bytes depend on it, through `DataChecksumsEnabled()` in `XLogHintBitIsNeeded()` ([xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826), [xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192)). The `df` leg measures the default: 0 images and 0 WAL bytes. |
| `wal_log_hints` (`wl` leg) | `off`, context `PGC_POSTMASTER` ([guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185)) | `on`, written to `postgresql.conf` before the first start, so it needs a restart to change. It shows that the GUC alone takes the same branch as checksums. The `wl` column's image counts and WAL bytes depend on it; `df` measures the default. |
| `full_page_writes` (case F, all legs) | `on`, context `PGC_SIGHUP` ([guc.c#full_page_writes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1163-L1175)) | `off` through `ALTER SYSTEM` and `pg_reload_conf()` (a reload), then reset the same way after the case. It tests the claim that the hint image ignores this setting. Case F depends on it; cases A to E ran with the default. With `wal_level` at its default `replica`, the checkpointer logs the change as an `FPW_CHANGE` record, which the script requires in the window ([xlog.c#UpdateFullPageWrites](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9494-L9550)). |
| `autovacuum` | `on`, context `PGC_SIGHUP` ([guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1424-L1434)) | `off` in `postgresql.conf` before the first start. 1,400 inserted rows exceed the auto-analyze threshold of 50 plus 0.1 × `reltuples` ([autovacuum.c:3066-3067](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3066-L3067), [guc.c:2957-2962](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2957-L2962), [guc.c:3404-3409](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3404-L3409)). An auto-analyze reads sampled pages through `HeapTupleSatisfiesVacuum()`, which sets hints ([heapam_handler.c:1050](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1050)). With the default, a worker could hint the pages before case A, and A, D, E and F could then show fewer dirtied pages and images. |
| `initdb -A trust --locale=C --encoding=UTF8` | Without these, `initdb` takes the locale and encoding from the environment, and uses `trust` with a warning when no method is given ([initdb.c#setlocales](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2300-L2330), [initdb.c#check_authmethod_unspecified](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2393-L2400)) | Fixed so the run does not depend on the host. No result depends on them: the fixture data is ASCII and no comparison involves collation. |
| `listen_addresses = ''`, `unix_socket_directories`, `port = 55443` | `localhost`, `/tmp`, `5432`; all `PGC_POSTMASTER` ([guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936-L3963), [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177-L2188)) | Socket-only access in the sandbox on a non-default port, so no other cluster is touched. No result depends on them. |
| `statement_timeout`, `lock_timeout` | `0`, disabled; `PGC_USERSET` ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)) | 5 minutes and 5 seconds for every session, through `PGOPTIONS`. They only bound a runaway statement. |
| Clean restart before cases A and D | — | A fast shutdown writes every dirty buffer, unlogged ones included, which ordinary checkpoints skip ([bufmgr.c:1303-1307](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1303-L1307)), and moves the redo pointer past every fixture page's LSN. So A and D start with clean, uncached pages that are eligible for an image. Without the restart, the unlogged table's buffers would stay dirty, and D would take the "already dirty" fast return instead of the non-permanent branch. |
| Catalog warm-up session | — | `SELECT count(*) FROM <table> WHERE false` in a separate session, before the restart or `CHECKPOINT`, hints the catalog rows that `CREATE TABLE` added. The measured statement therefore meets only the fixture table's unhinted rows, which is why `fpi_for_hint_all` equals `fpi_for_hint_table`. |
| Rolled-back `DELETE` before case E | — | Leaves an aborted xmax on every row, so E measures abort hints (`HEAP_XMAX_INVALID`) on pages whose xmin hints were already set by A. |
| Measured statement | — | `EXPLAIN (ANALYZE, BUFFERS, COSTS OFF, TIMING OFF) SELECT count(*)`: it executes a sequential scan and reports `dirtied=`. The 94-page tables are below `min_parallel_table_scan_size`, 8 MB by default ([guc.c#min_parallel_table_scan_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3120-L3128)), so the plan has no parallel workers. |

`synchronous_commit` stays at its default `on`, so every fixture commit is flushed before a later read, and `SetHintBits()` never takes its unflushed-commit branch in this run ([heapam_visibility.c:117-126](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L117-L126)). `shared_buffers` is left at the value `initdb` chose, which `<leg>-platform.txt` records.

### The script

```bash
#!/usr/bin/env bash
#
# hb12-run.sh - the measurement script for the wiki page
# wiki/v12/questions/storage-and-vacuum/client-statements-that-set-hint-bits.md
#
# Builds the pinned PostgreSQL 12 checkout out of tree with default configure
# options, then runs three isolated clusters one after another, each started
# and stopped inside its own stage:
#   df  default cluster: data checksums off, wal_log_hints off
#   ck  initdb -k (data checksums on), wal_log_hints left at its default off
#   wl  data checksums off, wal_log_hints = on
# In each cluster the same six cases run.  Each case brackets one read-only
# EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) with two WAL insert positions,
# reads that WAL window with pg_waldump, and counts XLOG_FPI_FOR_HINT records
# whose block reference names the scanned table.  EXPLAIN's "dirtied=" value
# is the number of shared buffers the scan dirtied.
#
# Every table and database is disposable and lives only in this sandbox.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/hb12-run.sh [stage ...]
# Stages: build df ck wl report stop clean
# Default: build df ck wl report

set -uo pipefail
set -e

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/hb12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55443}"
JOBS="${JOBS:-8}"
ROWS="${ROWS:-1400}"

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; BIN="$INST/bin"
OUT="$SANDBOX/out"; SOCK="$SANDBOX/sock"
export PGPORT="$PORT12" PGHOST="$SOCK"
# Session-scoped bounds for every statement this script sends (user context).
export PGOPTIONS="-c statement_timeout=5min -c lock_timeout=5s"

DATA=""          # data directory of the leg that is running
LEG=""

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

on_exit() {
  local rc=$?
  trap - EXIT
  if [ "$rc" -ne 0 ] && [ -n "$DATA" ] && [ -f "$DATA/postmaster.pid" ] &&
     [ -f "$SANDBOX/.hb12-owned" ]; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop >&2 || true
  fi
  exit "$rc"
}
trap on_exit EXIT

own_sandbox() {
  case "$WIKI_ROOT" in
    */postgres-llm-wiki) ;;
    *) die "run from the postgres-llm-wiki repository root" ;;
  esac
  [ -f "$WIKI_ROOT/AGENTS.md" ] || die "repository root not found"
  [ -f "$SANDBOX/.hb12-owned" ] || die "sandbox is not owned by this script: $SANDBOX"
  [ "$(cat "$SANDBOX/.hb12-owned")" = "$PIN12 $WIKI_ROOT" ] || die "sandbox owner/pin mismatch"
  export TMPDIR="$SANDBOX/tmp"
  mkdir -p "$TMPDIR" "$OUT" "$SOCK"
}
source_pin() {
  [ "$(git --no-optional-locks -C "$SRC12" rev-parse HEAD)" = "$PIN12" ] \
    || die "PostgreSQL 12 checkout is not at $PIN12"
  [ -z "$(git --no-optional-locks -C "$SRC12" status --porcelain --untracked-files=no)" ] \
    || die "PostgreSQL 12 checkout has tracked changes"
}

# Add the inline tag comment after each statement's leading verb.
tag_sql() {
  printf '%s\n' "$1" | sed -E 's/(^|;[[:space:]]*)(SELECT|INSERT|DELETE|CREATE|DROP|CHECKPOINT|EXPLAIN|ALTER|BEGIN|ROLLBACK|SHOW)([[:space:]]|;|$)/\1\2 \/\* wiki_hb12_fixture \*\/\3/g'
}
q() { # q <sql>: run, discard output
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d hb -c "$(tag_sql "$1")" > /dev/null \
    || die "failed: $1"
}
s() { # s <db> <sql>: one unaligned result
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$1" -c "$(tag_sql "$2")" \
    || die "failed on $1: $2"
}

server_start() {
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/$LEG-server.log" -w -t 60 start > /dev/null 2>&1 \
    || die "pg_ctl start failed, see $OUT/$LEG-server.log"
}
server_stop() {
  "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed"
}
server_restart() {
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/$LEG-server.log" -m fast -w -t 60 restart > /dev/null 2>&1 \
    || die "pg_ctl restart failed"
}
verify_stopped() { # verify_stopped <datadir>
  [ ! -f "$1/postmaster.pid" ] || die "postmaster.pid still present in $1"
  if pgrep -f -- "$1" > /dev/null 2>&1; then die "a process still names $1"; fi
}
verify_free() {
  [ -z "$(ls -A "$SOCK" 2>/dev/null)" ] || die "socket directory $SOCK is not empty"
  if "$BIN/pg_isready" -q -h 127.0.0.1 -p "$PORT12"; then die "port $PORT12 is occupied"; fi
}

stage_build() {
  say "build 12.2 out of tree from $SRC12 (default configure options)"
  source_pin
  if [ ! -e "$SANDBOX" ]; then
    mkdir -p "$SANDBOX" || die "could not create sandbox"
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.hb12-owned"
  fi
  own_sandbox
  mkdir -p "$BUILD"
  if [ -x "$BIN/postgres" ] && [ -x "$BIN/pg_waldump" ]; then
    [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 default" ] \
      || die "existing build has a different specification"
    note "matching build already present"
    return 0
  fi
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" > configure.log 2>&1 ) \
    || { cp "$BUILD/configure.log" "$OUT/"; die "configure failed, see $OUT/configure.log"; }
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { cp "$BUILD/make.log" "$OUT/" 2>/dev/null; grep 'error:' "$BUILD/make.log" | head -n 3 >&2
         die "make failed, see $OUT/make.log"; }
  printf '%s default\n' "$PIN12" > "$SANDBOX/.build-spec"
  note "$("$BIN/postgres" --version)"
}

# measure <case> <table> [<window-start-lsn>]
# One read-only statement, bracketed by WAL insert positions.  The optional
# third argument widens the WAL window backwards (used by case F to include
# the FPW_CHANGE record).
measure() {
  local case=$1 tbl=$2 wstart=${3:-} pages path db rfn start end bytes dirtied fpi_t fpi_all fpw_off fsm_wide base
  base="$OUT/$LEG-$case"
  pages=$(s hb "SELECT pg_relation_size('$tbl') / current_setting('block_size')::int")
  path=$(s hb "SELECT pg_relation_filepath('$tbl')")
  db=$(printf '%s\n' "$path" | cut -d/ -f2)
  rfn=$(printf '%s\n' "$path" | cut -d/ -f3)
  start=$(s hb "SELECT pg_current_wal_insert_lsn()")
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d hb \
      -c "$(tag_sql "EXPLAIN (ANALYZE, BUFFERS, COSTS OFF, TIMING OFF) SELECT count(*) FROM $tbl")" \
      > "$base.explain" 2>&1 || { cat "$base.explain" >&2; die "EXPLAIN failed for $case"; }
  end=$(s hb "SELECT pg_current_wal_insert_lsn()")
  [ -n "$wstart" ] || wstart=$start
  bytes=$(s hb "SELECT pg_wal_lsn_diff('$end', '$start')")
  # Force the window's WAL to disk so pg_waldump can read all of it.
  s hb "SELECT pg_switch_wal()" > /dev/null
  # Narrow dump: only the measured statement.  Wide dump: from wstart (case F
  # widens it back to before the full_page_writes reload), else the same.
  dump "$start" "$end" "$base.wal"
  dump "$wstart" "$end" "$base.wide.wal"
  fpi_t=$(grep -c -E "desc: FPI_FOR_HINT .*rel [0-9]+/$db/$rfn blk " "$base.wal" || true)
  fpi_all=$(grep -c -E "desc: FPI_FOR_HINT" "$base.wal" || true)
  fpw_off=$(grep -c -E "desc: FPW_CHANGE false" "$base.wide.wal" || true)
  fsm_wide=$(grep -c -E "desc: FPI_FOR_HINT .* fork fsm blk " "$base.wide.wal" || true)
  dirtied=$(sed -n 's/.*Buffers: shared.* dirtied=\([0-9]*\).*/\1/p' "$base.explain" | head -n 1)
  [ -n "$dirtied" ] || dirtied=0
  printf '%s|%s|%s|%s|%s|%s|%s|%s|%s|%s\n' "$LEG" "$case" "$tbl" "$pages" "$dirtied" \
    "$fpi_t" "$fpi_all" "$bytes" "$fpw_off" "$fsm_wide" >> "$OUT/results.txt"
  note "$LEG $case $tbl: pages=$pages dirtied=$dirtied fpi_for_hint(table)=$fpi_t fpi_for_hint(all)=$fpi_all wal_bytes=$bytes fpw_change_false(wide)=$fpw_off fsm_fpi_for_hint(wide)=$fsm_wide"
}
dump() { # dump <from-lsn> <to-lsn> <file>
  if [ "$(s hb "SELECT pg_wal_lsn_diff('$2', '$1')")" != "0" ]; then
    "$BIN/pg_waldump" -p "$DATA/pg_wal" -s "$1" -e "$2" > "$3" 2> "$3.err" \
      || { cat "$3.err" >&2; die "pg_waldump failed: $1 to $2"; }
  else
    : > "$3"
  fi
}

expect() { # expect <case> <field-name> <field-number> <value>
  local got
  got=$(grep -E "^$LEG\|$1\|" "$OUT/results.txt" | tail -n 1 | cut -d'|' -f"$3")
  [ "$got" = "$4" ] || die "$LEG $1: $2 = $got, expected $4"
  note "ok: $LEG $1: $2 = $got"
}
pages_of() { grep -E "^$LEG\|$1\|" "$OUT/results.txt" | tail -n 1 | cut -d'|' -f4; }

run_leg() { # run_leg <leg> <initdb options> <extra postgresql.conf line or empty>
  LEG=$1
  local initopts=$2 confline=$3 hintwal p
  say "leg $LEG: initdb $initopts ${confline:+, $confline}"
  own_sandbox
  source_pin
  [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 default" ] || die "run the build stage first"
  DATA="$SANDBOX/$LEG"
  if [ -f "$DATA/postmaster.pid" ]; then server_stop; fi
  verify_free
  rm -rf "$DATA"
  if [ -f "$OUT/results.txt" ]; then
    grep -v -E "^$LEG\|" "$OUT/results.txt" > "$OUT/results.tmp" || true
    mv "$OUT/results.tmp" "$OUT/results.txt"
  fi
  # shellcheck disable=SC2086
  "$BIN/initdb" -D "$DATA" -A trust --locale=C --encoding=UTF8 $initopts \
    > "$OUT/$LEG-initdb.log" 2>&1 || die "initdb failed, see $OUT/$LEG-initdb.log"
  {
    printf "listen_addresses = ''\n"
    printf "unix_socket_directories = '%s'\n" "$SOCK"
    printf "port = %s\n" "$PORT12"
    printf "autovacuum = off\n"
    [ -z "$confline" ] || printf '%s\n' "$confline"
  } >> "$DATA/postgresql.conf"
  server_start
  s postgres "CREATE DATABASE hb" > /dev/null
  {
    printf 'leg=%s\n' "$LEG"
    uname -srm
    "$BIN/postgres" --version
    for g in data_checksums wal_log_hints full_page_writes wal_level synchronous_commit \
             autovacuum block_size wal_block_size shared_buffers; do
      s hb "SELECT '$g=' || current_setting('$g')"
    done
    s hb "SELECT 'max_data_alignment=' || max_data_alignment FROM pg_control_init()"
  } > "$OUT/$LEG-platform.txt"
  cat "$OUT/$LEG-platform.txt" >&2
  grep -q '^block_size=8192$' "$OUT/$LEG-platform.txt" || die "fixtures assume 8 kB blocks"

  # Fixtures (disposable).  Rows of ~530 bytes, about 14 per page.
  q "CREATE TABLE t (id int, pad text); INSERT INTO t SELECT g, repeat('x', 500) FROM generate_series(1, $ROWS) g"
  q "CREATE UNLOGGED TABLE u (id int, pad text); INSERT INTO u SELECT g, repeat('x', 500) FROM generate_series(1, $ROWS) g"
  # Hint the new catalog rows for t and u now, in a separate session, so the
  # measured statements below only meet the fixture tables' own unhinted rows.
  q "SELECT count(*) FROM t WHERE false; SELECT count(*) FROM u WHERE false"
  # A clean restart ends with a shutdown checkpoint: every shared buffer is
  # written (unlogged ones too, which ordinary checkpoints skip) and evicted,
  # and the redo pointer moves past every fixture page's LSN.
  server_restart

  # A: first read of t after its inserting transaction committed.
  measure A_first_select t
  # B: the same read again, hints already set and buffers already dirty.
  measure B_second_select t
  # C: after a CHECKPOINT the buffers are clean again, but the hints are set.
  q "CHECKPOINT"
  measure C_after_checkpoint t
  # D: first read of the unlogged table (clean buffers, not BM_PERMANENT).
  measure D_unlogged_first_select u
  # E: a rolled-back DELETE leaves an aborted xmax on every row of t.
  q "BEGIN; DELETE FROM t; ROLLBACK"
  q "CHECKPOINT"
  measure E_after_rolled_back_delete t
  # F: full_page_writes = off (sighup: ALTER SYSTEM plus reload), new table.
  p=$(s hb "SELECT pg_current_wal_insert_lsn()")
  q "ALTER SYSTEM SET full_page_writes = off"
  s hb "SELECT pg_reload_conf()" > /dev/null
  n=0
  until [ "$(s hb "SHOW full_page_writes")" = "off" ]; do
    n=$((n + 1)); [ "$n" -le 30 ] || die "full_page_writes did not change"; sleep 1
  done
  sleep 2   # let the checkpointer process the reload as well
  q "CREATE TABLE f (id int, pad text); INSERT INTO f SELECT g, repeat('x', 500) FROM generate_series(1, $ROWS) g"
  q "SELECT count(*) FROM f WHERE false"
  q "CHECKPOINT"
  measure F_full_page_writes_off f "$p"
  q "ALTER SYSTEM RESET full_page_writes"
  s hb "SELECT pg_reload_conf()" > /dev/null

  server_stop
  verify_stopped "$DATA"
  DATA=""

  # Assertions.  hintwal: does this leg take the hint-WAL branch?
  case "$LEG" in df) hintwal=no ;; *) hintwal=yes ;; esac
  p=$(pages_of A_first_select)
  [ "$p" -gt 50 ] || die "fixture too small: $p pages"
  expect A_first_select dirtied 5 "$p"
  expect B_second_select dirtied 5 0
  expect B_second_select fpi_table 6 0
  expect B_second_select fpi_all 7 0
  expect C_after_checkpoint dirtied 5 0
  expect C_after_checkpoint fpi_table 6 0
  expect C_after_checkpoint fpi_all 7 0
  expect D_unlogged_first_select dirtied 5 "$(pages_of D_unlogged_first_select)"
  expect D_unlogged_first_select fpi_table 6 0
  expect D_unlogged_first_select fpi_all 7 0
  expect E_after_rolled_back_delete dirtied 5 "$p"
  expect F_full_page_writes_off dirtied 5 "$(pages_of F_full_page_writes_off)"
  expect F_full_page_writes_off fpw_change_false 9 1
  if [ "$hintwal" = yes ]; then
    expect A_first_select fpi_table 6 "$p"
    expect A_first_select fpi_all 7 "$p"
    expect E_after_rolled_back_delete fpi_table 6 "$p"
    expect E_after_rolled_back_delete fpi_all 7 "$p"
    expect F_full_page_writes_off fpi_table 6 "$(pages_of F_full_page_writes_off)"
    expect F_full_page_writes_off fpi_all 7 "$(pages_of F_full_page_writes_off)"
  else
    for c in A_first_select E_after_rolled_back_delete F_full_page_writes_off; do
      expect "$c" fpi_table 6 0
      expect "$c" fpi_all 7 0
      expect "$c" wal_bytes 8 0
    done
  fi
}

stage_df() { run_leg df "" ""; }
stage_ck() { run_leg ck "-k" ""; }
stage_wl() { run_leg wl "" "wal_log_hints = on"; }

stage_report() {
  own_sandbox
  say "results ($OUT/results.txt)"
  printf 'leg|case|table|pages|dirtied|fpi_for_hint_table|fpi_for_hint_all|wal_bytes|fpw_change_false_wide|fsm_fpi_for_hint_wide\n' >&2
  cat "$OUT/results.txt" >&2
}

stage_stop() {
  local d
  [ -d "$SANDBOX" ] || { note "no sandbox"; return 0; }
  own_sandbox
  for d in df ck wl; do
    if [ -f "$SANDBOX/$d/postmaster.pid" ]; then
      "$BIN/pg_ctl" -D "$SANDBOX/$d" -m fast -w -t 60 stop > /dev/null 2>&1 || die "could not stop $d"
    fi
    verify_stopped "$SANDBOX/$d"
  done
  verify_free
  note "no server, socket or listener left"
}
stage_clean() {
  [ -d "$SANDBOX" ] || { note "no sandbox"; return 0; }
  stage_stop
  rm -rf "$SANDBOX"
  note "deleted $SANDBOX"
}

[ "$#" -gt 0 ] || set -- build df ck wl report
for st in "$@"; do
  case "$st" in
    build|df|ck|wl|report|stop|clean) "stage_$st" ;;
    *) die "unknown stage: $st" ;;
  esac
done
```

## Context Reviewed

- Version: PostgreSQL 12, pinned checkout `raw/postgres-12/` at `45b88269a353ad93744772791feb6d01bc7e1e42` (`git rev-parse HEAD` matched `wiki/versions.md`).
- Quoted paragraph: taken from [Consequences of Enabling Data Checksums](enabling-data-checksums-consequences.md#runtime-cost-and-additional-wal); used for navigation only.
- Tuple hints: `heapam_visibility.c` (all `HeapTupleSatisfies*` routines, `SetHintBits`, `HeapTupleSetHintBits`, the dispatcher), `heapam.c` (`heapgetpage`, `heapgettup`, `heap_getnextslot`, `heap_fetch`, `heap_hot_search_buffer`, `heap_delete`, `heap_update`, `heap_lock_tuple`, `UpdateXmaxHintBits`), `heapam_handler.c` (index fetch, bitmap, analyze, index build, cluster copy, `heapam_tuple_lock`), `vacuumlazy.c`, `pruneheap.c`, `tuptoaster.c`, `snapmgr.h`, `htup_details.h`.
- Other hint writers: tree-wide search for `MarkBufferDirtyHint` and `HeapTupleSetHintBits` callers in `src/` and `contrib/` (nbtree, hash, GiST, BRIN, FSM, sequence, pruning).
- Callers and executor boundary: `nodeSeqscan.c`, `nodeIndexscan.c`, `nodeIndexonlyscan.c`, `nodeBitmapHeapscan.c`, `nodeLockRows.c`, `nodeModifyTable.c`, `indexam.c` (`index_fetch_heap`), `genam.c` (`RelationGetIndexScan`, `systable_beginscan`), `nbtinsert.c` (`_bt_check_unique`), `hio.c`, `ri_triggers.c`.
- Buffer and WAL: `bufmgr.c` (`MarkBufferDirtyHint`, `BM_PERMANENT` assignment), `xloginsert.c` (`XLogSaveBufferForHint`, `XLogRecordAssemble`), `xlog.h`, `xlog.c#DataChecksumsEnabled`, `xlogdesc.c`, `guc.c` (`wal_log_hints`, timeouts), `initdb.c`.
- Docs and READMEs: `doc/src/sgml/config.sgml` (`wal_log_hints`), `src/backend/access/transam/README` ("Writing Hints"), `src/backend/storage/page/README`, `src/backend/storage/freespace/README`.
- Build and generated headers: none of the cited symbols come from generated headers; `XLogHintBitIsNeeded()` is a macro in `xlog.h`, so every caller re-evaluates it at run time.
- Tests: a search of `src/test/` for `FPI_FOR_HINT`, `hint bit` and `wal_log_hints` found only recovery TAP tests that set `wal_log_hints` as a precondition ([008_fsm_truncation.pl:18](../../../../raw/postgres-12/src/test/recovery/t/008_fsm_truncation.pl#L18), [015_promotion_pages.pl:14-17](../../../../raw/postgres-12/src/test/recovery/t/015_promotion_pages.pl#L14-L17), [PostgresNode.pm:476](../../../../raw/postgres-12/src/test/perl/PostgresNode.pm#L476)). None checks which statements set hints; see Open Questions.
- Common concept pages: `wiki/v12/common-concepts/` does not exist, so none was linked.
- Measurement: a 12.2 server built from the pin with default configure options, three clusters (default, `initdb -k`, `wal_log_hints = on`), `pg_waldump` record dumps per statement window, and `EXPLAIN (ANALYZE, BUFFERS)` output; see [Measurement Script](#measurement-script). For the setting defaults it relies on, also `guc.c` (`full_page_writes`, `autovacuum`, the analyze thresholds, `min_parallel_table_scan_size`, socket settings), `autovacuum.c`, `xlog.c` (`UpdateFullPageWrites`, `ReserveXLogInsertLocation`) and `xlog_internal.h`.

## Evidence Map

| Claim | Evidence |
|---|---|
| Visibility routines set hints and dirty the buffer | [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131), [heapam_visibility.c#HeapTupleSatisfiesVisibility](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1679-L1718) |
| Committed hint skipped while commit is unflushed | [heapam_visibility.c:117-126](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L117-L126) |
| MVCC reader does not hint a transaction in its snapshot | [heapam_visibility.c:1056-1057](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1056-L1057), [heapam_visibility.c:1122-1123](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1122-L1123) |
| All-visible pages skip per-tuple checks in sequential scans | [heapam.c:427-447](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L427-L447) |
| Any, HistoricMVCC and (normally) Toast set no hints | [heapam_visibility.c#HeapTupleSatisfiesAny](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L338-L342), [heapam_visibility.c#HeapTupleSatisfiesHistoricMVCC](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1541-L1677), [heapam_visibility.c#HeapTupleSatisfiesToast](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L360-L425) |
| Writes: `UPDATE`/`DELETE`/row locks use `HeapTupleSatisfiesUpdate` and `UpdateXmaxHintBits` | [heapam.c:2512](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2512), [heapam.c:3068](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3068), [heapam.c:4017](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L4017), [heapam.c#UpdateXmaxHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1780-L1808) |
| FK checks use `FOR KEY SHARE` | [ri_triggers.c:384](../../../../raw/postgres-12/src/backend/utils/adt/ri_triggers.c#L384) |
| Unique checks use a dirty snapshot and mark dead entries | [nbtinsert.c:349-360](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L349-L360), [nbtinsert.c:575-590](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L575-L590) |
| Maintenance commands use `HeapTupleSatisfiesVacuum` | [vacuumlazy.c:1051](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1051), [heapam_handler.c:828](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L828), [heapam_handler.c:1050](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1050), [heapam_handler.c:1387](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1387) |
| Index scans kill dead entries as hints, not in recovery | [indexam.c:578-586](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L578-L586), [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1718-L1814) |
| `pd_prune_xid` update is a hint; real pruning is WAL-logged | [pruneheap.c:276-291](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L276-L291) |
| FSM updates are hints | [freespace.c:671-688](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L671-L688), [hio.c:533](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L533) |
| Catalog lookups use a catalog snapshot | [genam.c:352-378](../../../../raw/postgres-12/src/backend/access/index/genam.c#L352-L378) |
| WAL branch predicate and defaults | [xlog.h:183-192](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192), [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185), [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826) |
| Local, permanent, already-dirty and recovery branches | [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3422-L3553), [bufmgr.c:1303-1313](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1303-L1313) |
| Image only when page LSN ≤ redo pointer; forced even with `full_page_writes` off | [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L893-L957), [xloginsert.c:539-545](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L539-L545) |
| Measured: a `SELECT` dirties every freshly committed page in all legs, and writes one `FPI_FOR_HINT` per page only with checksums or `wal_log_hints`; none on repeat, after a `CHECKPOINT` on hinted pages, or for an unlogged table; images even with `full_page_writes` off | [Measured results](#measured-results), produced by the [Measurement Script](#measurement-script); source for each outcome cited there |
| Observability | [bufmgr.c:3539-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3539-L3546), [explain.c:2890-2902](../../../../raw/postgres-12/src/backend/commands/explain.c#L2890-L2902), [xlogdesc.c:185-186](../../../../raw/postgres-12/src/backend/access/rmgrdesc/xlogdesc.c#L185-L186) |

## Open Questions

- **README versus source on `full_page_writes`.** [storage/page/README:36-45](../../../../raw/postgres-12/src/backend/storage/page/README#L36-L45) says the hint WAL record is needed "if we have full_page_writes = on and checksums enabled", and the comment at [bufmgr.c:3467-3468](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3467-L3468) says `full_page_writes` is handled inside `XLogInsert()`. The source registers the hint block with `REGBUF_FORCE_IMAGE`, and `XLogRecordAssemble()` tests that flag before `doPageWrites` ([xloginsert.c:944-947](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L944-L947), [xloginsert.c:539-545](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L539-L545)). Source wins: this page states the image is written with `full_page_writes` off. Case F in [Measured results](#measured-results) agrees with the source: 94 images with an `FPW_CHANGE false` record in the window. The README and comment wording remain a same-checkout discrepancy. The same discrepancy is already filed on the checksum-consequences page.
- **No test coverage found.** No regression or TAP test in the pinned checkout was found that checks which statements set hint bits or emit `XLOG_FPI_FOR_HINT`.
- **Measured scope is narrower than the statement table.** The script measures sequential-scan `SELECT`s on a permanent and an unlogged table, abort hints after a rolled-back `DELETE`, and, incidentally, FSM hint images from `CREATE TABLE`/`INSERT`. The other rows of [Statements and the paths that reach a hint](#statements-and-the-paths-that-reach-a-hint) (index scans and `LP_DEAD` marks, `UPDATE`/`DELETE`, row locks, unique and FK checks, maintenance commands, catalog lookups, sequence and BRIN cases), the snapshot and asynchronous-commit conditions, temporary tables, and the hot-standby branch remain source readings, not measured results.
- **Statement coverage is by access path, not exhaustive.** Any extension, PL function or contrib module that scans a heap through the table AM inherits the same visibility routines. This page does not list each one.

## Source References

- [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131)
- [heapam_visibility.c#HeapTupleSatisfiesMVCC](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L962-L1147)
- [heapam_visibility.c#HeapTupleSatisfiesVacuum](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1163-L1383)
- [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L352-L460)
- [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L117-L165)
- [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L589)
- [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1718-L1814)
- [pruneheap.c#heap_page_prune_opt](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L73-L135)
- [freespace.c#fsm_set_and_search](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L671-L688)
- [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3405-L3553)
- [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L893-L957)
- [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192)
- [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185)
- [transam/README#Writing-Hints](../../../../raw/postgres-12/src/backend/access/transam/README#L620-L639)
- [config.sgml#wal_log_hints](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2738-L2756)

## Navigation

- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [Consequences of Enabling Data Checksums in PostgreSQL 12 (unverified)](enabling-data-checksums-consequences.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
