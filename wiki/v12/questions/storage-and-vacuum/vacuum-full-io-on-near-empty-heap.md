---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: claude-opus-5-max 2026-08-27T21:44:14Z
---

# How Much I/O a VACUUM FULL Performs on a Multi-GB, Near-Empty Heap in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Where VACUUM FULL goes in v12](#where-vacuum-full-goes-in-v12)
  - [Read side: the whole heap, every block, no skip path](#read-side-the-whole-heap-every-block-no-skip-path)
  - [Heap size means the physical file, not pg_class.relpages](#heap-size-means-the-physical-file-not-pg_classrelpages)
  - [Write side: the new heap is sized by live rows](#write-side-the-new-heap-is-sized-by-live-rows)
  - [Page arithmetic for the new heap](#page-arithmetic-for-the-new-heap)
  - [WAL volume](#wal-volume)
  - [The fsync](#the-fsync)
  - [The old file is unlinked, not rewritten](#the-old-file-is-unlinked-not-rewritten)
  - [Index rebuild](#index-rebuild)
  - [TOAST](#toast)
  - [Worked example for the shape in the question](#worked-example-for-the-shape-in-the-question)
  - [Where the reads land in the buffer pool](#where-the-reads-land-in-the-buffer-pool)
  - [Second-order writes to the old heap: hint bits](#second-order-writes-to-the-old-heap-hint-bits)
  - [No cost-based throttling applies to the rewrite](#no-cost-based-throttling-applies-to-the-rewrite)
  - [What VACUUM FULL never does here](#what-vacuum-full-never-does-here)
  - [How to observe the read volume on a v12 server](#how-to-observe-the-read-volume-on-a-v12-server)
  - [What the command leaves behind](#what-the-command-leaves-behind)
  - [Test coverage in the pinned tree](#test-coverage-in-the-pinned-tree)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, if a table heap is multiple GBs in size, however it has only a few thousand rows, so the table heap is essentially empty or filled with empty pages, how much I/O will a VACUUM FULL perform in relation to heap size?

Prompt hygiene note: the prompt as typed read "in postgresql 12, question: if a table heap is in multiple GBs however is has how few thousand rows so the table heap essentially empty or filled with empty pages ,  how much i/o a vacuum full will perform in relation to heap size?". Per `MANDATORY Prompt Hygiene` the asker was offered the choice and selected "correct and restate", so the restated text above fixes: `postgresql` -> `PostgreSQL`, `i/o` -> `I/O`, `vacuum full` -> `VACUUM FULL`, the garbled `is has how few thousand rows` -> `it has only a few thousand rows`, the missing verb in `the table heap essentially empty`, the space before the comma, and the doubled space. The asker also scoped this to source-only evidence (no measured run) and to `VACUUM FULL` alone (no comparison with plain `VACUUM`, `CLUSTER`, or `pg_repack`).

## Answer

### Short answer

Reads scale with the **full physical size of the heap**. Writes scale with the **live data only**. Emptiness buys you nothing on the read side.

| Component | Volume, relative to old heap size `S` | Why |
|---|---|---|
| Old heap reads | `≈ S` (every block, once, sequentially) | plain seq scan over `rs_nblocks` with no visibility-map skip |
| New heap writes | `≈ live bytes` (a few pages here), not `S` | one `smgrextend` per filled page |
| WAL | `0` at `wal_level = minimal`; `≈ new heap size` if `XLogIsNeeded()` | one full-page `log_newpage` per new page |
| fsync | one `smgrimmedsync` per new relation: the (tiny) heap, plus its TOAST heap if it has one | `heap_sync` at end of rewrite |
| Old heap deletion | metadata only (first segment `ftruncate`d to 0 with its unlink deferred past the next checkpoint, later 1 GB segments unlinked at once) | `mdunlinkfork` |
| Index rebuilds | reads the **new** small heap, writes small indexes | `reindex_relation` after the swap |
| TOAST | proportional to **live** toasted bytes, via index lookups | external values fetched back and re-saved |

So on a 4 GB heap holding a few thousand rows, expect roughly **4 GB of sequential reads** and, depending on row width, **184 kB to 2.5 MB of writes** — a read:write ratio of about **1,700:1 to 22,800:1** over the stored tuple widths in the [worked example](#worked-example-for-the-shape-in-the-question). The command's duration is set by how fast the storage can stream the old file, not by how much data survives.

The one thing that *is* proportional to emptiness is the per-tuple CPU work, and even that is bounded by *line pointers* rather than by surviving rows. v12's `PageRepairFragmentation` never removes unused line pointers — "It doesn't remove unused line pointers! Please don't change this." [bufpage.c#PageRepairFragmentation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L471-L482) — so a page emptied by `DELETE` plus `VACUUM` keeps its line pointer array and `PageGetMaxOffsetNumber` stays at that page's historical high-water mark. `heapgettup` then walks the array and skips every entry that fails `ItemIdIsNormal`, with no visibility test, no tuple reform and no rewrite call [heapam.c#heapgettup](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L651-L700). Only a page that never held a line pointer gives `PageGetMaxOffsetNumber(dp) == 0`, and only such a page costs nothing beyond its read.

### Where VACUUM FULL goes in v12

`VACUUM FULL` is not a separate code path in v12; it is `CLUSTER` without a cluster index. `vacuum_rel` takes `AccessExclusiveLock` for the `FULL` case and hands off to `cluster_rel` with `InvalidOid` as the index: [vacuum.c#vacuum_rel](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1681), [vacuum.c:1816-1828](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1816-L1828). `cluster.c`'s header comment states the contract: "If indexOid is InvalidOid, the table will be rewritten in physical order instead of index order. This is the new implementation of VACUUM FULL" [cluster.c#cluster_rel](../../../../raw/postgres-12/src/backend/commands/cluster.c#L261-L266).

`cluster_rel` tags progress reporting as `PROGRESS_CLUSTER_COMMAND_VACUUM_FULL` precisely because `indexOid` is invalid [cluster.c:276-281](../../../../raw/postgres-12/src/backend/commands/cluster.c#L276-L281), then calls `rebuild_relation` [cluster.c:428-429](../../../../raw/postgres-12/src/backend/commands/cluster.c#L428-L429), which does three things in order: create a transient heap, copy the data, swap and reindex [cluster.c#rebuild_relation](../../../../raw/postgres-12/src/backend/commands/cluster.c#L610-L626).

### Read side: the whole heap, every block, no skip path

Because `OIDOldIndex` is `InvalidOid`, `copy_table_data` leaves `OldIndex = NULL` and sets `use_sort = false` [cluster.c:797-800](../../../../raw/postgres-12/src/backend/commands/cluster.c#L797-L800), [cluster.c:891-894](../../../../raw/postgres-12/src/backend/commands/cluster.c#L891-L894). That selects the sequential-scan branch of the heap AM's copy callback, which opens a plain `table_beginscan` with `SnapshotAny` and **no scan keys**, and publishes the relation's total block count as `heap_blks_total`:

[heapam_handler.c#heapam_relation_copy_for_cluster](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L769-L782)

There is no visibility-map consultation anywhere in this path: `grep` for `visibilitymap` returns zero matches in all three files that make up the rewrite — [cluster.c](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1-L76), [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L20-L50) (which holds the scan-and-copy loop), and [rewriteheap.c](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L1-L120). That is a correctness requirement, not an oversight: the rewrite must copy every live *and* recently-dead tuple, and an all-visible page still contains tuples that must move to the new relfilenode. Contrast plain `VACUUM`, whose `heap_blks_scanned` documentation explicitly says "the visibility map is used to optimize scans, some blocks will be skipped without inspection" [monitoring.sgml#heap_blks_scanned](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3803-L3813); the `pg_stat_progress_cluster` counterpart carries no such sentence [monitoring.sgml#pg_stat_progress_cluster](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L4026-L4032).

The scan visits every block exactly once. `initscan` fixes `rs_nblocks = RelationGetNumberOfBlocks()` [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L225-L231), and `heapgettup` walks forward with wraparound, terminating only when it returns to `rs_startblock` [heapam.c#heapgettup](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L728-L764). Each block costs exactly one `ReadBufferExtended` [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L380-L383). An empty page is not special-cased: it is read, then `linesleft` is set from `PageGetMaxOffsetNumber` and the loop moves on [heapam.c:766-784](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L766-L784). A page with no line pointers at all gives `linesleft = 0` and is abandoned immediately; a page whose tuples were deleted and vacuumed still carries its line pointer array, so the loop walks it and rejects each entry on `ItemIdIsNormal` [heapam.c:651-700](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L651-L700). Either way the block is read once, and nothing about the page's emptiness removes that read.

There is also no prefetching to soften it. `PrefetchBuffer` appears nowhere in [cluster.c](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1-L76), [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L20-L50) or [rewriteheap.c](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L1-L120), and the one call site in `heapam.c` sits in `heap_compute_xid_horizon_for_tuples`, an index-vacuum helper outside this path [heapam.c:6952-6958](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L6952-L6958). So v12 issues one 8 kB `ReadBufferExtended` at a time and relies on OS readahead for sequential throughput.

One subtlety worth knowing when watching progress: `SnapshotAny` is not an MVCC snapshot, so page-at-a-time mode is switched off [heapam.c:1160-1164](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1160-L1164) and `heapgetpage` returns right after the read [heapam.c:385-386](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L385-L386) — which also means `heap_page_prune_opt` is *not* called on the old heap during the rewrite. Synchronized scanning can still start the scan mid-relation when the relation exceeds `NBuffers / 4` [heapam.c:245-296](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L245-L296), and because `heap_blks_scanned` is reported as `heapScan->rs_cblock + 1` [heapam_handler.c:815-821](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L815-L821), the progress counter can start high and wrap to 1. That counter is also updated per returned tuple rather than per block, which matters a great deal on this shape; see [How to observe the read volume on a v12 server](#how-to-observe-the-read-volume-on-a-v12-server). The block *count* read is unchanged.

### Heap size means the physical file, not pg_class.relpages

The read volume is set by `RelationGetNumberOfBlocks`, which for a table goes through `table_relation_size` and rounds up to whole blocks [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2799-L2826), [bufmgr.h:198-199](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L198-L199). That is a live `smgr` answer about the file on disk, not the possibly stale `pg_class.relpages` estimate. A heap whose `relpages` says 100 but whose file is 4 GB will be read as 4 GB.

### Write side: the new heap is sized by live rows

Every surviving tuple is reformed and handed to the rewrite module [heapam_handler.c#reform_and_rewrite_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2523-L2548), which fills one private page buffer and flushes it with `smgrextend` when the next tuple no longer fits:

[rewriteheap.c#raw_heap_insert](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L688-L728)

Three consequences for I/O accounting:

1. **The new heap bypasses shared buffers entirely.** `state->rs_buffer` is a `palloc(BLCKSZ)` scratch page [rewriteheap.c:270-273](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L270-L273) and each finished page goes straight to storage via `smgrextend(..., true)` — the `true` being `skipFsync`, because the rewrite fsyncs once at the end instead [rewriteheap.c:705-716](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L705-L716).
2. **Writes are strictly `ceil(live bytes / usable page bytes)` pages**, the last of which is the possibly-partial page that `end_heap_rewrite` flushes after the scan ends [rewriteheap.c:330-345](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L330-L345). Dead tuples never reach `raw_heap_insert`: `HEAPTUPLE_DEAD` short-circuits to `rewrite_heap_dead_tuple` and `continue` [heapam_handler.c:879-890](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L879-L890).
3. **The free space map is never consulted or written for the main fork**, because the rewrite does not go through `RelationGetBufferForTuple` at all: neither that function nor any free-space-map call appears in [rewriteheap.c](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L1-L120), whose only FSM reference is the `HEAP_INSERT_SKIP_FSM` flag it passes for TOAST inserts [rewriteheap.c:653-658](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L653-L658).

Tuples are also frozen on the way through, which costs no extra I/O beyond the page write [rewriteheap.c#rewrite_heap_tuple](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L405-L413); the cutoffs are computed aggressively because the table is being rewritten anyway [cluster.c:860-867](../../../../raw/postgres-12/src/backend/commands/cluster.c#L860-L867). This is why the docs call `FREEZE` redundant under `FULL` [vacuum.sgml#FREEZE](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L113-L126).

### Page arithmetic for the new heap

The packing rule is exactly the `raw_heap_insert` test `len + saveFreeSpace > pageFreeSpace`, with:

- `len = MAXALIGN(heaptup->t_len)` [rewriteheap.c:673](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L673)
- `saveFreeSpace = BLCKSZ * (100 - fillfactor) / 100`, defaulting to fillfactor 100 and therefore 0 [rewriteheap.c:684-686](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L684-L686), [rel.h#RelationGetTargetPageFreeSpace](../../../../raw/postgres-12/src/include/utils/rel.h#L304-L309), [rel.h:278-279](../../../../raw/postgres-12/src/include/utils/rel.h#L278-L279)
- `pageFreeSpace = PageGetHeapFreeSpace(page)`, which is `pd_upper - pd_lower` minus one `ItemIdData` for the new line pointer, and zero once `MaxHeapTuplesPerPage` line pointers exist with none free [bufpage.c#PageGetHeapFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L664-L714), [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L580-L597)

The new heap's fillfactor is inherited, because `make_new_heap` copies the old heap's reloptions [cluster.c:664-673](../../../../raw/postgres-12/src/backend/commands/cluster.c#L664-L673), [cluster.c:694-714](../../../../raw/postgres-12/src/backend/commands/cluster.c#L694-L714). With the constants `SizeOfPageHeaderData = 24` (from `PageHeaderData`: 8 + 2 + 2 + 2 + 2 + 2 + 2 + 4 before `pd_linp`) [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164), [bufpage.h:216](../../../../raw/postgres-12/src/include/storage/bufpage.h#L216) and `sizeof(ItemIdData) = 4` [itemid.h#ItemIdData](../../../../raw/postgres-12/src/include/storage/itemid.h#L25-L30), the derived capacity of a default-fillfactor page is

```text
tuples_per_page = floor((BLCKSZ - SizeOfPageHeaderData) / (MAXALIGN(t_len) + 4))
                = floor(8168 / (MAXALIGN(t_len) + 4))
```

capped by `MaxHeapTuplesPerPage = floor(8168 / (MAXALIGN(SizeofHeapTupleHeader) + 4)) = 291` [htup_details.h#MaxHeapTuplesPerPage](../../../../raw/postgres-12/src/include/access/htup_details.h#L564-L576), [htup_details.h:184](../../../../raw/postgres-12/src/include/access/htup_details.h#L184). This arithmetic is derived from the cited rules, not measured on a server.

### WAL volume

The rewrite is WAL-logged only when WAL is genuinely needed for it:

```c
use_wal = XLogIsNeeded() && RelationNeedsWAL(NewHeap);
```

[heapam_handler.c:719-723](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L719-L723)

`XLogIsNeeded()` is the `wal_level >= WAL_LEVEL_REPLICA` test [xlog.h:177-181](../../../../raw/postgres-12/src/include/access/xlog.h#L177-L181), [xlog.h#WalLevel](../../../../raw/postgres-12/src/include/access/xlog.h#L159-L165), and `RelationNeedsWAL` is true only for permanent relations [rel.h#RelationNeedsWAL](../../../../raw/postgres-12/src/include/utils/rel.h#L515-L520). So at `wal_level = minimal`, the new heap generates **no WAL for its data at all** — the transient relfilenode is created and dropped inside the one transaction, so crash recovery only has to not find it. When WAL is needed, each new page is logged as one full-page image via `log_newpage` [rewriteheap.c:697-703](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L697-L703) plus the last page [rewriteheap.c:331-338](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L331-L338).

The critical point for this question: **WAL is proportional to the new heap, not the old one.** A 4 GB heap with a few thousand rows produces kilobytes of WAL for the heap, not gigabytes. The remaining WAL comes from catalog updates (`pg_class` rows for both relations, the relfilenode swap, `RelationClearMissing`) and from the index rebuilds.

TOAST inserts made during the rewrite follow the same decision, via `HEAP_INSERT_SKIP_WAL` when `rs_use_wal` is false [rewriteheap.c:653-668](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L653-L668).

### The fsync

Regardless of the WAL decision, the rewrite ends with an unconditional flush-and-fsync of the new relation for any WAL-logged table [rewriteheap.c:347-359](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L347-L359). `heap_sync` flushes buffers and calls `smgrimmedsync` for the main fork and, if present, the TOAST heap; the FSM is deliberately not synced [heapam.c#heap_sync](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8937-L8961). On the shape in the question that is an fsync over a handful of pages.

### The old file is unlinked, not rewritten

`finish_heap_swap` exchanges relfilenodes rather than copying anything back [cluster.c#swap_relation_files](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1035-L1062), then drops the transient relation — which by then owns the *old*, multi-GB relfilenode — with `performDeletion` [cluster.c:1449-1458](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1449-L1458). Deletion is scheduled for commit, not executed inline [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L160), which is why the code comment notes there is "no chance to reclaim disk space before commit" [cluster.c:1378-1392](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1378-L1392).

Two I/O facts follow:

- Any dirty old-heap buffers still resident are **discarded, not written**: `smgrdounlinkall` states that "bufmgr will just drop them without bothering to write the contents" [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L459-L463).
- The main fork's first segment is `ftruncate`d to zero and its real unlink deferred to after the next checkpoint, to protect the relfilenode number from reuse; additional 1 GB segments are unlinked on the spot in the `for (segno = 1;; segno++)` loop that runs until `ENOENT` [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L284-L359), with the rationale spelled out for exactly this "CLUSTER and CREATE INDEX" fsync-not-WAL case [md.c:225-254](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L225-L254). Freeing multiple GB is therefore filesystem metadata work, not data movement.

### Index rebuild

After the swap, every index is rebuilt from scratch [cluster.c:1393-1410](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1393-L1410). `reindex_index` assigns a new relfilenode and calls `index_build` [index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3526-L3530), and each AM's build does a full `table_index_build_scan` — for example B-tree [nbtsort.c:487-491](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L487-L491), hash [hash.c:165-168](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L165-L168), GiST [gistbuild.c:193-198](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L193-L198), GIN [gininsert.c:378-384](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L378-L384), SP-GiST [spginsert.c:126-128](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L126-L128), BRIN [brin.c:719-724](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L719-L724).

That scan is over the **new** heap, which is already tiny, so index-rebuild I/O is proportional to the surviving rows and not to the old heap. The comment in `finish_heap_swap` gives the design reason: rebuilding beats incremental index maintenance because the rewrite is effectively a bulk load [cluster.c:1378-1392](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1378-L1392). Note that this is where `maintenance_work_mem` matters for a `VACUUM FULL` — a B-tree build sizes its sort area from it [nbtsort.c:442-445](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L442-L445) — while the heap phase itself never sorts, because `use_sort` is false and the tuplesort is never created [heapam_handler.c:738-744](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L738-L744).

### TOAST

`vacuum_rel` deliberately does **not** recurse into the TOAST relation for a `FULL` vacuum: "In VACUUM FULL, though, the toast table is automatically rebuilt by cluster_rel so we shouldn't recurse to it" [vacuum.c:1792-1800](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1792-L1800). `make_new_heap` creates a fresh TOAST relation for the transient heap [cluster.c:725-751](../../../../raw/postgres-12/src/backend/commands/cluster.c#L725-L751), and because both heaps have one, the swap is done by content with `rd_toastoid` set so pointers and value OIDs are preserved [cluster.c:826-856](../../../../raw/postgres-12/src/backend/commands/cluster.c#L826-L856).

The data still moves. Any tuple carrying an external pointer goes through the toaster [rewriteheap.c:640-669](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L640-L669), and `toast_insert_or_update` fetches each external value back — "any external value we find still in the tuple must be someone else's that we cannot reuse ... Fetch it back" [tuptoaster.c:680-699](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L680-L699) — via `heap_tuple_fetch_attr` -> `toast_fetch_datum` [tuptoaster.c#heap_tuple_fetch_attr](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L100-L111), then re-saves it with `toast_save_datum` [tuptoaster.c:863-870](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L863-L870).

So TOAST I/O is:

- **Reads**: index-driven chunk fetches for live toasted values only. `toast_fetch_datum` opens the TOAST index and reads the chunks through `systable_beginscan_ordered` on `(valueid, chunkidx)` [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1904-L1935), so a bloated TOAST relation is never sequentially scanned and its dead chunks cost nothing. Those reads are random-ish rather than sequential.
- **Writes**: the live chunks re-inserted into the new TOAST relation, with `HEAP_INSERT_SKIP_FSM` so the FSM is not read or written [rewriteheap.c:653-658](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L653-L658), [hio.c:325](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L325), [hio.c:520-527](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L520-L527).

A near-empty main heap with a near-empty-but-huge TOAST relation therefore shrinks both, and still only reads the main heap in full.

### Worked example for the shape in the question

Derived from the cited packing rule, for a 4 GB heap (`524,288` blocks) holding 5,000 live rows and no TOAST, default fillfactor:

| Stored tuple length `t_len` | `MAXALIGN(t_len) + 4` | tuples/page | new heap pages | new heap bytes | read:write |
|---|---|---|---|---|---|
| 32 | 36 | 226 | 23 | 188,416 | ~22,800 : 1 |
| 48 | 52 | 157 | 32 | 262,144 | ~16,400 : 1 |
| 100 | 108 | 75 | 67 | 548,864 | ~7,800 : 1 |
| 500 | 508 | 16 | 313 | 2,564,096 | ~1,675 : 1 |

Reads stay at 4,294,967,296 bytes in every row of that table. Only the right-hand columns move. The table is arithmetic over cited definitions, not a measurement; see [Open Questions](#open-questions).

### Where the reads land in the buffer pool

Because the relation is multiple GB and therefore certainly larger than `NBuffers / 4`, `initscan` attaches a `BAS_BULKREAD` strategy [heapam.c:245-258](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L245-L258) — `table_beginscan` passes `SO_ALLOW_STRAT` [tableam.h#table_beginscan](../../../../raw/postgres-12/src/include/access/tableam.h#L736-L744). That ring is `256 * 1024 / BLCKSZ` = 32 buffers, further capped at `NBuffers / 8` [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L541-L588).

Practical effect: streaming 4 GB through the pool recycles the same 32 buffers, so a `VACUUM FULL` on a bloated table largely leaves the rest of the cache alone. The reads still hit the storage device once per block. The ring is not airtight, though: a ring buffer that is dirty and whose write would need a WAL flush is dropped out of the ring by `StrategyRejectBuffer`, which sets that slot to `InvalidBuffer` so the next allocation comes from the normal clock sweep and evicts a non-ring buffer [freelist.c#StrategyRejectBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L685-L704), [bufmgr.c:1122-1139](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1122-L1139). On this shape that is bounded by the hint-bit dirtying described below.

### Second-order writes to the old heap: hint bits

There is one way the old, doomed heap can generate writes. Every surviving-candidate tuple is visibility-tested with `HeapTupleSatisfiesVacuum` [heapam_handler.c:823-828](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L823-L828), and that function sets commit/abort hint bits, each of which calls `MarkBufferDirtyHint` [heapam_visibility.c#HeapTupleSatisfiesVacuum](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1188-L1213), [heapam_visibility.c:1244-1253](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1244-L1253), [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131). A dirtied ring buffer is normally written when its slot is reused; only a buffer whose write would require a WAL flush is rejected and replaced [freelist.c#StrategyRejectBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L685-L696).

For the shape in the question this term is small and bounded by the number of *`LP_NORMAL` tuples* present, not by heap size: a page whose line pointers are all unused or dead never reaches `HeapTupleSatisfiesVacuum` at all, and a heap that reached this state through `DELETE` plus routine `VACUUM` generally has its remaining hint bits already set. Any such page still dirty at commit time is discarded rather than written [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L459-L463).

### No cost-based throttling applies to the rewrite

`vacuum_delay_point` is the only place a vacuum sleeps [vacuum.c#vacuum_delay_point](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1945-L1971), and it is **not called** anywhere in `cluster.c`, `rewriteheap.c`, or `heapam_handler.c` — the pinned tree's callers are `vacuumlazy.c`, `analyze.c`, the statistics type-analyze functions, and the index AMs' bulkdelete/cleanup routines. Buffer accesses accrue `VacuumCostBalance` only while `VacuumCostActive` is set, which requires `vacuum_cost_delay > 0` [bufmgr.c:767-772](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L767-L772), [bufmgr.c:959-961](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L959-L961), [vacuum.c:375-385](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L375-L385) — so at the v12 default of `0` the counters `VacuumPageHit`/`VacuumPageMiss` advance but the balance stays at zero. Either way, with no delay point in the path nothing sleeps.

So the gigabytes are read as fast as the device allows. Relevant settings and apply scopes:

| Setting | v12 default | Context | Apply scope | Effect on VACUUM FULL |
|---|---|---|---|---|
| `vacuum_cost_delay` | `0` | `PGC_USERSET` | session/transaction | none during the rewrite; if set above `0` the balance accrues there and can be paid off later by a `VACUUM (FULL, ANALYZE)`'s analyze phase, which does call `vacuum_delay_point` [analyze.c:1046](../../../../raw/postgres-12/src/backend/commands/analyze.c#L1046) and is never reset in between [vacuum.c:375-385](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L375-L385) |
| `vacuum_cost_limit` | `200` | `PGC_USERSET` | session/transaction | same |
| `vacuum_cost_page_hit` / `_miss` / `_dirty` | `1` / `10` / `20` | `PGC_USERSET` | session/transaction | same |

Sources: [guc.c:3372-3381](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3372-L3381), [guc.c:2311-2319](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2311-L2319), [guc.c:2281-2309](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2281-L2309), plus `VacuumCostActive = (VacuumCostDelay > 0)` [vacuum.c:375-385](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L375-L385). `PGC_USERSET` maps to session/transaction scope: no restart and no reload is needed to change any of them.

### What VACUUM FULL never does here

- It does not truncate the old file in place. `TRUNCATE` and `INDEX_CLEANUP` are explicitly ignored under `FULL` [vacuum.sgml#INDEX_CLEANUP](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L193-L203), [vacuum.sgml#TRUNCATE](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L209-L224), and `RelationTruncate` appears nowhere in [rewriteheap.c](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L1-L120).
- It cannot be combined with `DISABLE_PAGE_SKIPPING`, which errors out [vacuum.c:262-268](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L262-L268) — consistent with there being no page skipping to disable.
- It does not read the old heap twice. One scan, one pass.

### How to observe the read volume on a v12 server

Use `pg_statio_all_tables.heap_blks_read`, not the progress view. `heap_blks_total` is honest — it is `rs_nblocks` published once at scan start [heapam_handler.c:779-781](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L779-L781), [monitoring.sgml:4018-4024](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L4018-L4024) — but `heap_blks_scanned` is **not a per-block counter**, and on a near-empty heap it badly under-reports the reads:

- The only write to it is inside the per-tuple loop, immediately after `table_scan_getnextslot` returns a tuple, and its value is that tuple's block number plus one [heapam_handler.c:810-821](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L810-L821).
- Blocks that return no tuple therefore produce no update at all. Under `SnapshotAny` a block returns a tuple only if it holds at least one `LP_NORMAL` line pointer [heapam.c:651-700](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L651-L700), so a page whose line pointers are all `LP_UNUSED` or `LP_DEAD` — the normal state after `DELETE` plus `VACUUM` — is invisible to the counter.
- Nothing sets the counter to `heap_blks_total` when the scan finishes; the loop simply ends and `end_heap_rewrite` runs [heapam_handler.c:923-974](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L923-L974).

So on a 4 GB heap holding a few thousand rows the counter sits unchanged across every run of tupleless blocks while those gigabytes stream past, and it ends below `heap_blks_total` — its final value is one past the block of the last tuple the scan returned, not the number of blocks read. The v12 documentation says only that the counter "advances when the phase is `seq scanning heap`" [monitoring.sgml:4026-4032](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L4026-L4032); it does not promise one increment per block, and the plain-`VACUUM` column's "skipped blocks are included in this total, so that this number will eventually become equal to `heap_blks_total`" sentence has no counterpart here [monitoring.sgml#heap_blks_scanned](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3803-L3813).

The progress view is still worth watching for `phase`, `heap_tuples_written` and `index_rebuild_count`. Note that in the `VACUUM FULL` path `heap_tuples_scanned` and `heap_tuples_written` are set from the same `*num_tuples` counter in one `pgstat_progress_update_multi_param` call, so they are always equal and neither counts dead tuples [heapam_handler.c:892-922](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L892-L922). All columns used below exist in v12's `pg_stat_progress_cluster` [system_views.sql#pg_stat_progress_cluster](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L967-L992).

```sql
BEGIN;
SET LOCAL statement_timeout = '5s';
SET LOCAL lock_timeout = '2s';

SELECT /* wiki_vacuum_full_read_progress */
       p.pid,
       c.relname,
       p.command,
       p.phase,
       p.heap_blks_total,
       p.heap_blks_scanned,
       p.heap_tuples_scanned,
       p.heap_tuples_written,
       p.index_rebuild_count
  FROM pg_stat_progress_cluster p
  JOIN pg_class c ON c.oid = p.relid;
COMMIT;
```

The read count itself comes from `pg_statio_all_tables.heap_blks_read` / `heap_blks_hit`, taken before and after the command. Those are fed by the `pgstat_count_buffer_read` / `pgstat_count_buffer_hit` calls inside `ReadBufferExtended`, so every block the rewrite reads is counted exactly once [bufmgr.c:660-669](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L660-L669), [system_views.sql#pg_statio_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L623-L646):

```sql
BEGIN;
SET LOCAL statement_timeout = '5s';
SET LOCAL lock_timeout = '2s';

SELECT /* wiki_vacuum_full_io_counters */
       relid,
       schemaname,
       relname,
       heap_blks_read,
       heap_blks_hit,
       toast_blks_read,
       toast_blks_hit,
       idx_blks_read,
       idx_blks_hit
  FROM pg_statio_all_tables
 WHERE schemaname = 'public'
   AND relname = 'your_table';
COMMIT;
```

Both statements were checked against the pinned checkout's view definitions and column names; neither was executed, since this answer is source-only by request. The `BEGIN` / `COMMIT` wrapper is required for the timeouts to apply: at top level outside a transaction block, `SET LOCAL` calls `WarnNoTransactionBlock` [guc.c:8115-8123](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L8115-L8123), which raises `WARNING: SET LOCAL can only be used in transaction blocks` and leaves the following statement untimed [xact.c#CheckTransactionBlock](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L3404-L3430).

### What the command leaves behind

`swap_relation_files` swaps `relpages`, `reltuples`, and `relallvisible` between the two `pg_class` rows [cluster.c:1131-1148](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1131-L1148), and `copy_table_data` had already written the new heap's real page and tuple counts — `relpages` and `reltuples` only, never `relallvisible` [cluster.c:950-972](../../../../raw/postgres-12/src/backend/commands/cluster.c#L950-L972). Since the rewrite never touches the visibility map, the new heap has no VM fork and `relallvisible` lands at 0. The first plain `VACUUM` afterwards therefore reads the whole (now tiny) heap: with no VM fork, `visibilitymap_get_status` returns 0 for every block [visibilitymap.c#visibilitymap_get_status](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L329-L356), [visibilitymap.c#vm_readbuf](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L553-L587), so `lazy_scan_heap` finds nothing skippable [vacuumlazy.c:609-636](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L609-L636). `VACUUM (FULL, VERBOSE)` reports the old block count in its own message, using `RelationGetNumberOfBlocks(OldHeap)` [cluster.c:934-943](../../../../raw/postgres-12/src/backend/commands/cluster.c#L934-L943).

### Test coverage in the pinned tree

`VACUUM FULL` is exercised for correctness in [vacuum.sql:22](../../../../raw/postgres-12/src/test/regress/sql/vacuum.sql#L22), [vacuum.sql:70-74](../../../../raw/postgres-12/src/test/regress/sql/vacuum.sql#L70-L74), [vacuum.sql:144-145](../../../../raw/postgres-12/src/test/regress/sql/vacuum.sql#L144-L145), and for invalid-index behavior in [create_index.sql:495-499](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L495-L499). There is **no test of I/O volume, block counts, or progress-view output**: `pg_stat_progress_cluster` appears in the test tree only as a view-definition dump [rules.out:1849](../../../../raw/postgres-12/src/test/regress/expected/rules.out#L1849). The read-everything property is therefore guaranteed by the code path, not by a regression test.

## Context Reviewed

- Dispatch and locking: [vacuum.c](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1620-L1700), including the `FULL` lock mode, the TOAST-recursion suppression, and the `DISABLE_PAGE_SKIPPING` rejection.
- Rewrite driver: all of [cluster.c](../../../../raw/postgres-12/src/backend/commands/cluster.c#L255-L434) `cluster_rel`, `rebuild_relation`, `make_new_heap`, `copy_table_data`, `swap_relation_files`, `finish_heap_swap`.
- Heap AM copy callback and its registration: [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L690-L979), [heapam_handler.c:2640](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2640).
- Rewrite module: [rewriteheap.c](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L240-L757) `begin_heap_rewrite`, `end_heap_rewrite`, `rewrite_heap_tuple`, `raw_heap_insert`.
- Sequential-scan mechanics: [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L205-L320) `initscan`, [heapam.c:355-386](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L355-L386) `heapgetpage`, all of [heapam.c:486-785](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L486-L785) `heapgettup` including the `ItemIdIsNormal` skip and the per-page `linesleft` reset.
- Buffer strategy and stats: [freelist.c](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L537-L704), [bufmgr.c:640-970](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L970), [bufmgr.c:1090-1160](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1090-L1160) the ring-reject path, [bufmgr.c:2790-2837](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2790-L2837).
- Storage teardown: [storage.c](../../../../raw/postgres-12/src/backend/catalog/storage.c#L143-L170), [smgr.c](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L419-L473), [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L220-L362).
- Page and tuple limits, and line pointer retention: [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L439-L720) including `PageRepairFragmentation`, [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L110-L220), [htup_details.h](../../../../raw/postgres-12/src/include/access/htup_details.h#L550-L580), [itemid.h](../../../../raw/postgres-12/src/include/storage/itemid.h#L17-L33), [rel.h](../../../../raw/postgres-12/src/include/utils/rel.h#L270-L312).
- Visibility map after the swap: [visibilitymap.c](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L309-L367), [visibilitymap.c:547-587](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L547-L587), [vacuumlazy.c:560-736](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L560-L736).
- WAL-level and persistence predicates: [xlog.h](../../../../raw/postgres-12/src/include/access/xlog.h#L159-L199), [rel.h:515-520](../../../../raw/postgres-12/src/include/utils/rel.h#L515-L520).
- `SET LOCAL` scope for the observability statements: [guc.c:8110-8125](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L8110-L8125), [xact.c:3367-3430](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L3367-L3430).
- TOAST: [tuptoaster.c](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L640-L880), [tuptoaster.c:1890-1990](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1890-L1990) `toast_fetch_datum`, [hio.c](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L286-L330).
- Index rebuild entry points: [index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3540), plus each AM's `table_index_build_scan` call.
- GUCs: [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2280-L2320), [guc.c:3372-3392](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3372-L3392).
- Docs: [vacuum.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L60-L230), [maintenance.sgml](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L120-L240), [monitoring.sgml](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3780-L4090).
- Catalog views: [system_views.sql](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L590-L995).
- Tests: [vacuum.sql](../../../../raw/postgres-12/src/test/regress/sql/vacuum.sql#L18-L150), [create_index.sql](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L490-L505), [rules.out](../../../../raw/postgres-12/src/test/regress/expected/rules.out#L1845-L1855).

## Evidence Map

| Claim | Evidence |
|---|---|
| `VACUUM FULL` is `CLUSTER` with no index | [vacuum.c:1816-1828](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1816-L1828), [cluster.c:261-266](../../../../raw/postgres-12/src/backend/commands/cluster.c#L261-L266) |
| It takes `AccessExclusiveLock` | [vacuum.c:1680-1681](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1681) |
| No sort, so no temp-file I/O in the heap phase | [cluster.c:891-894](../../../../raw/postgres-12/src/backend/commands/cluster.c#L891-L894), [heapam_handler.c:738-744](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L738-L744) |
| Plain seq scan with `SnapshotAny`, no scan keys | [heapam_handler.c:769-782](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L769-L782) |
| `heap_blks_total` is the whole relation | [heapam_handler.c:779-781](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L779-L781), [monitoring.sgml:4018-4024](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L4018-L4024) |
| No visibility-map skipping in the rewrite | zero `visibilitymap` matches in [cluster.c](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1-L76), [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L20-L50) and [rewriteheap.c](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L1-L120); contrast [monitoring.sgml:3803-3813](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3803-L3813) |
| Every block is read exactly once | [heapam.c:225-231](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L225-L231), [heapam.c:728-764](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L728-L764), [heapam.c:380-383](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L380-L383) |
| Vacuumed-empty pages keep their line pointers, so the scan walks the array and skips on `ItemIdIsNormal` | [bufpage.c:471-482](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L471-L482), [heapam.c:651-700](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L651-L700), [heapam.c:766-784](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L766-L784) |
| Page-mode off under `SnapshotAny`, so no opportunistic pruning | [heapam.c:1160-1164](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1160-L1164), [heapam.c:385-386](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L385-L386) |
| Read volume is the physical file size | [bufmgr.c:2799-2826](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2799-L2826), [bufmgr.h:198-199](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L198-L199) |
| New pages written by `smgrextend`, outside shared buffers | [rewriteheap.c:270-273](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L270-L273), [rewriteheap.c:705-716](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L705-L716) |
| Dead tuples are never written | [heapam_handler.c:879-890](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L879-L890) |
| Packing rule and fillfactor inheritance | [rewriteheap.c:673](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L673), [rewriteheap.c:684-693](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L684-L693), [rel.h:304-309](../../../../raw/postgres-12/src/include/utils/rel.h#L304-L309), [cluster.c:664-673](../../../../raw/postgres-12/src/backend/commands/cluster.c#L664-L673) |
| Page free-space accounting | [bufpage.c:580-597](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L580-L597), [bufpage.c:664-714](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L664-L714), [bufpage.h:151-164](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164), [itemid.h:25-30](../../../../raw/postgres-12/src/include/storage/itemid.h#L25-L30) |
| `MaxHeapTuplesPerPage` cap | [htup_details.h:564-576](../../../../raw/postgres-12/src/include/access/htup_details.h#L564-L576) |
| WAL only when `XLogIsNeeded()`, one FPI per new page | [heapam_handler.c:719-723](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L719-L723), [rewriteheap.c:697-703](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L697-L703), [xlog.h:177-181](../../../../raw/postgres-12/src/include/access/xlog.h#L177-L181), [rel.h:515-520](../../../../raw/postgres-12/src/include/utils/rel.h#L515-L520) |
| Unconditional fsync of the new heap and its TOAST | [rewriteheap.c:347-359](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L347-L359), [heapam.c:8937-8961](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8937-L8961) |
| Relfilenodes are swapped, not copied | [cluster.c:1035-1062](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1035-L1062) |
| Old file deleted at commit; dirty buffers dropped unwritten | [cluster.c:1449-1458](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1449-L1458), [storage.c:144-160](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L160), [smgr.c:459-463](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L459-L463) |
| First segment truncated with its unlink deferred to the next checkpoint; later segments unlinked at once | [md.c:284-359](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L284-L359), [md.c:225-254](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L225-L254) |
| Indexes rebuilt from the new heap | [cluster.c:1393-1410](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1393-L1410), [index.c:3526-3530](../../../../raw/postgres-12/src/backend/catalog/index.c#L3526-L3530), [nbtsort.c:487-491](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L487-L491) |
| TOAST not recursed into; rebuilt via the main rewrite | [vacuum.c:1792-1800](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1792-L1800), [cluster.c:725-751](../../../../raw/postgres-12/src/backend/commands/cluster.c#L725-L751), [cluster.c:826-856](../../../../raw/postgres-12/src/backend/commands/cluster.c#L826-L856) |
| External values fetched back and re-saved | [rewriteheap.c:640-669](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L640-L669), [tuptoaster.c:680-699](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L680-L699), [tuptoaster.c:100-111](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L100-L111) |
| TOAST inserts skip the FSM | [rewriteheap.c:653-658](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L653-L658), [hio.c:325](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L325), [hio.c:520-527](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L520-L527) |
| 32-buffer `BAS_BULKREAD` ring | [heapam.c:245-258](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L245-L258), [tableam.h:736-744](../../../../raw/postgres-12/src/include/access/tableam.h#L736-L744), [freelist.c:541-588](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L541-L588) |
| Hint bits can dirty old-heap pages | [heapam_handler.c:823-828](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L823-L828), [heapam_visibility.c:112-131](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131), [heapam_visibility.c:1188-1213](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L1188-L1213), [freelist.c:685-696](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L685-L696) |
| No `vacuum_delay_point` in the rewrite path | [vacuum.c:1945-1971](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1945-L1971) plus zero matches in `cluster.c`/`rewriteheap.c`/`heapam_handler.c` |
| Cost GUC defaults and `PGC_USERSET` scope | [guc.c:3372-3381](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3372-L3381), [guc.c:2281-2319](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2281-L2319), [vacuum.c:375-385](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L375-L385) |
| `TRUNCATE`/`INDEX_CLEANUP` ignored, `DISABLE_PAGE_SKIPPING` rejected | [vacuum.sgml:193-224](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L193-L224), [vacuum.c:262-268](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L262-L268) |
| Progress and statistics observability | [system_views.sql:967-992](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L967-L992), [system_views.sql:623-646](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L623-L646), [bufmgr.c:660-669](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L660-L669) |
| `heap_blks_scanned` is updated per returned tuple, not per block, and is never squared up at end of scan | [heapam_handler.c:810-821](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L810-L821), [heapam_handler.c:923-974](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L923-L974), [monitoring.sgml:4026-4032](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L4026-L4032) |
| `heap_tuples_scanned` equals `heap_tuples_written` in this path | [heapam_handler.c:892-922](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L892-L922) |
| `SET LOCAL` needs an enclosing transaction block | [guc.c:8115-8123](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L8115-L8123), [xact.c:3404-3430](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L3404-L3430) |
| `relallvisible` swapped in, VM never built, so the next plain `VACUUM` cannot skip | [cluster.c:1131-1148](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1131-L1148), [cluster.c:950-972](../../../../raw/postgres-12/src/backend/commands/cluster.c#L950-L972), [visibilitymap.c:329-356](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L329-L356), [vacuumlazy.c:609-636](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L609-L636) |
| Correctness-only test coverage | [vacuum.sql:70-74](../../../../raw/postgres-12/src/test/regress/sql/vacuum.sql#L70-L74), [create_index.sql:495-499](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L495-L499), [rules.out:1849](../../../../raw/postgres-12/src/test/regress/expected/rules.out#L1849) |

## Open Questions

1. **Nothing here was measured.** The asker scoped this to source-only evidence, and the 2026-08-27 review kept that scope, so no 12.2 server was built and no read/write/WAL byte counts were captured. The [worked example](#worked-example-for-the-shape-in-the-question) is arithmetic over cited definitions. Confirming it means building 12.2 out of tree from `raw/postgres-12/`, creating a genuinely multi-GB near-empty heap, and comparing `pg_stat_progress_cluster.heap_blks_total`, `pg_statio_all_tables.heap_blks_read`, `pg_current_wal_lsn()` deltas, and process-level I/O against the predictions. Two specific unmeasured quantities: how far `heap_blks_scanned` ends below `heap_blks_total` on a given bloated heap, and how many line pointers per page survive `DELETE` plus `VACUUM` in practice.
2. **A documentation claim is wrong for this shape, and source wins.** [maintenance.sgml:233-237](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L233-L237) says these rewriting commands "temporarily use extra disk space approximately equal to the size of the table", and [vacuum.sgml:100-108](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L100-L108) says `FULL` "requires extra disk space, since it writes a new copy of the table". The transient heap is sized by surviving tuples in `raw_heap_insert`, not by the old relation, so on a near-empty multi-GB heap the extra space is negligible rather than ~`S`. The prose is a reasonable approximation for a mildly bloated table and misleading for a severely bloated one; no same-version doc text states the near-empty case.
3. **The hint-bit write term is unquantified.** `HeapTupleSatisfiesVacuum` can dirty old-heap pages, and whether a given dirty ring buffer is written or rejected depends on `XLogNeedsFlush(lsn)`, which in turn depends on whether `data_checksums` or `wal_log_hints` made the hint-bit update WAL-logged and so advanced the page LSN — that is the `XLogHintBitIsNeeded()` test [xlog.h:183-192](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192), consumed by `MarkBufferDirtyHint` [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3465-L3500). Neither that interaction nor its magnitude on a real bloated heap was traced to a conclusion here.
4. **Multi-segment behavior was read but not exercised.** For a multi-GB heap, `RELSEG_SIZE` means several 1 GB segment files. `mdunlinkfork` unlinks the non-first segments immediately and truncates the first, but the wall-clock cost of freeing many gigabytes of extents is filesystem-dependent and outside the pinned source.
5. **Concurrent-activity effects on the swap were not examined.** `DropRelFileNodesAllBuffers` scans the buffer pool, so the teardown cost scales with `shared_buffers`; that path was cited for its no-write property, not analyzed for cost.
6. **Partitioned tables were not considered.** `VACUUM FULL` on a partitioned parent expands to its leaves, and the per-leaf accounting above should apply once each, but the expansion path in `vacuum.c` was not traced.

## Source References

- [vacuum.c](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1620-L1860)
- [cluster.c](../../../../raw/postgres-12/src/backend/commands/cluster.c#L255-L1520)
- [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L690-L979)
- [rewriteheap.c](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L240-L757)
- [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L205-L785)
- [heapam_visibility.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L80-L145)
- [visibilitymap.c](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L309-L367)
- [vacuumlazy.c](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L560-L736)
- [tuptoaster.c](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L640-L880)
- [hio.c](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L286-L330)
- [nbtsort.c](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L380-L500)
- [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L970)
- [freelist.c](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L537-L704)
- [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L439-L720)
- [smgr.c](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L419-L473)
- [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L220-L362)
- [storage.c](../../../../raw/postgres-12/src/backend/catalog/storage.c#L143-L170)
- [index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L3433-L3540)
- [xact.c](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L3367-L3430)
- [system_views.sql](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L590-L995)
- [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2280-L2320)
- [xlog.h](../../../../raw/postgres-12/src/include/access/xlog.h#L159-L199)
- [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L110-L220)
- [htup_details.h](../../../../raw/postgres-12/src/include/access/htup_details.h#L550-L580)
- [itemid.h](../../../../raw/postgres-12/src/include/storage/itemid.h#L17-L33)
- [rel.h](../../../../raw/postgres-12/src/include/utils/rel.h#L270-L312)
- [bufmgr.h](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L176-L210)
- [tableam.h](../../../../raw/postgres-12/src/include/access/tableam.h#L723-L751)
- [ref/vacuum.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L60-L230)
- [maintenance.sgml](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L120-L240)
- [monitoring.sgml](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3780-L4090)
- [vacuum.sql](../../../../raw/postgres-12/src/test/regress/sql/vacuum.sql#L18-L150)
- [create_index.sql](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L490-L505)
- [rules.out](../../../../raw/postgres-12/src/test/regress/expected/rules.out#L1845-L1855)

## Navigation

- [v12/index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [PostgreSQL 12 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Proposing and Testing a fillfactor-Corrected pgstattuple_approx Metric for Table Heap Bloat in PostgreSQL 12 (unverified)](pgstattuple-approx-heap-bloat.md)
- [PostgreSQL 12 Database Health Checklist (unverified)](../server-administration/database-health-checklist.md)
