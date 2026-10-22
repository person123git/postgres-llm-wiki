---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Checking Every B-Tree Index TID Against Its Heap From Disk, Bypassing Shared Buffers, in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Answer up front](#answer-up-front)
  - [Mental model: what the check compares](#mental-model-what-the-check-compares)
  - [Why the usual page readers go through shared buffers](#why-the-usual-page-readers-go-through-shared-buffers)
  - [Logic map](#logic-map)
  - [When an index TID is valid](#when-an-index-tid-is-valid)
  - [Making the files agree with memory: the lock and two checkpoints](#making-the-files-agree-with-memory-the-lock-and-two-checkpoints)
  - [Never handing a damaged page to pageinspect](#never-handing-a-damaged-page-to-pageinspect)
  - [The check](#the-check)
  - [Reading the report](#reading-the-report)
  - [Measured results](#measured-results)
  - [Branches, limits and false positives](#branches-limits-and-false-positives)
  - [Final causal summary](#final-causal-summary)
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

In PostgreSQL 12, propose SQL that uses a contrib extension to check every TID in an index against its table's heap, to verify that the table has no data loss. The check must read only from disk, bypassing shared buffers. Do not use amcheck.

## Answer

### Answer up front

**No single contrib function in PostgreSQL 12 does this, but contrib [pageinspect](../../../glossary.md#pageinspect) can decode pages that core `pg_read_binary_file()` reads straight from the relation files.** The script below reads every page of the table, its [TOAST](../../../glossary.md#toast) table and their [B-tree](../../../glossary.md#b-tree) indexes from the segment files. It never reads those pages through [shared buffers](../../../glossary.md#shared_buffers). It then checks that every index entry's [TID](../../../glossary.md#tid) names a heap [line pointer](../../../glossary.md#line-pointer) that an index may legitimately point to.

pageinspect's own page readers do use shared buffers. `get_raw_page()` copies a page out of a shared buffer it pinned with `ReadBufferExtended()`, and `bt_page_items(text, int)` calls `ReadBuffer()` [rawpage.c#get_raw_page_internal](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L161-L166) [btreefuncs.c#bt_page_items](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L357). Its decoders also accept a page image as `bytea`, from any source [pageinspect--1.5--1.6.sql](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L82-L99). `pg_read_binary_file()` supplies that image: it opens the file with `AllocateFile()` and reads it with `fread()`, outside the buffer manager [genfile.c#read_binary_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L146-L176). amcheck is not used, as the question requires.

Three conditions make an on-disk check meaningful:

1. **The files must be current.** Shared buffers can hold newer page versions than the files. The script therefore takes a lock that stops writers and runs `CHECKPOINT`, which writes every [dirty buffer](../../../glossary.md#dirty-buffer) of a permanent relation that is dirty when the checkpoint starts [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1803-L1846).
2. **The heap must be read before the index, with a second checkpoint in between.** A [VACUUM](../../../glossary.md#vacuum) of the TOAST table can still run, because `LOCK TABLE` refuses TOAST tables. This order keeps it from producing a false finding; see [the lock and two checkpoints](#making-the-files-agree-with-memory-the-lock-and-two-checkpoints).
3. **No damaged page may reach a pageinspect decoder.** In PostgreSQL 12, `bt_page_items(bytea)` and `heap_page_items()` trust parts of the page they are given. The script validates every page in SQL and PL/pgSQL first; see [Never handing a damaged page to pageinspect](#never-handing-a-damaged-page-to-pageinspect).

On an isolated 12.2 build, the check passed every healthy table. With [data checksums](../../../glossary.md#data-checksums) off, the default, and with them on, it caught each of nine kinds of damage injected into the files: a page zeroed while still cached, an emptied last segment, a short middle segment, a lost heap write, a lost index write, a slot reused by a heap-only tuple, a corrupt line pointer, a zeroed TOAST page, and damaged index pages. A tenth kind, one changed byte inside a tuple, it caught only with checksums on. After a cold restart, the check left zero pages of the checked relations in shared buffers. See [Measured results](#measured-results).

The check runs in one direction only: from index entries to heap line pointers. It cannot find a heap row that lost its index entries, and it cannot tell whether a tuple's contents are right. [Branches, limits and false positives](#branches-limits-and-false-positives) lists those limits and the others.

### Mental model: what the check compares

A B-tree [leaf page](../../../glossary.md#leaf-page) item holds a key and a heap TID: a block number and a line pointer number [itup.h#IndexTupleData](../../../../raw/postgres-12/src/include/access/itup.h#L35-L51) [itemptr.h#ItemPointerData](../../../../raw/postgres-12/src/include/storage/itemptr.h#L36-L47) [block.h#BlockIdData](../../../../raw/postgres-12/src/include/storage/block.h#L53-L57). The check reads both sides from the files and compares them. It keeps every value in session-local temporary tables.

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| Index TID | Heap block and line pointer that one live leaf item names | `bt_page_items(bytea)` on a leaf page read from the index file | Stored on disk; current once the checkpoint has written the index's dirty buffers | Joined with the heap page of the same block |
| Line pointer state | `LP_UNUSED`, `LP_NORMAL`, `LP_REDIRECT` or `LP_DEAD`, plus whether a normal item is a heap-only tuple | The heap page read from the table file, decoded by the script | Stored on disk; current under the same condition | Decides whether a TID is valid |
| Reachable heap blocks | Blocks a backend can address: full segments up to and including the first short one | `pg_stat_file()` on each segment file | Live file sizes | A TID at or past this block is `heap_block_missing` |
| Page state | Whether the page is all zeros, has a sane header, and matches its checksum | The script's validation, plus `page_checksum()` when checksums are on | Computed from the bytes read | Decides whether a page may be decoded at all |
| Byte order and geometry | Block size, segment size, alignment, checksum setting, and how this build stores integers | `pg_control_init()`, `pg_control_system()` and bytes 8–11 of `global/pg_control` | Stored in the [control file](../../../glossary.md#control-file) | Every byte offset the script decodes |

Shared buffers are not in this table, and that is the point. A page still cached in memory may differ from the file. A normal query sees the cached copy. This check sees the file, so it reports what a fresh read after a restart would find.

### Why the usual page readers go through shared buffers

pageinspect's relation-aware readers go through the [buffer manager](../../../glossary.md#buffer-manager). `get_raw_page()` opens the relation, calls `ReadBufferExtended()`, and copies the buffer's page [rawpage.c#get_raw_page_internal](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L95-L172). `bt_page_items(text, int)` and `bt_page_stats()` call `ReadBuffer()` the same way [btreefuncs.c#bt_page_items](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L311-L363) [btreefuncs.c:203](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L203). Reading a page through the buffer manager returns the cached version when one exists, and loads the page into shared buffers when none does. Contrib `pg_prewarm` in `read` mode does bypass shared buffers: it reads each block into backend memory with `smgrread()`. But it returns only the number of blocks read, so it cannot show what a page contains [pg_prewarm.c:171-184](../../../../raw/postgres-12/contrib/pg_prewarm/pg_prewarm.c#L171-L184).

`pg_read_binary_file(filename, offset, length, missing_ok)` knows nothing about relations. It reads bytes from a file under the data directory [pg_proc.dat:6065-6073](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6065-L6073) [genfile.c#pg_read_binary_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L292-L323):

- It reads with stdio. `read_binary_file()` calls `AllocateFile()`, `fseeko()` and `fread()` [genfile.c#read_binary_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L110-L177).
- A read past the end of the file returns fewer bytes, not an error, because the result's length is whatever `fread()` returned [genfile.c:165-172](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L165-L172). With `missing_ok` true, a missing file returns NULL [genfile.c:146-155](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L146-L155). The script treats both as a `short_read` page.
- `EXECUTE` on it is revoked from `PUBLIC` [system_views.sql:1330-1335](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1330-L1335). The check needs a superuser anyway, because every pageinspect function the script calls tests `superuser()` [heapfuncs.c:135-138](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L135-L138) [btreefuncs.c:430-433](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L430-L433) [rawpage.c:306-309](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L306-L309), and so does `CHECKPOINT` [utility.c#CheckPointStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L756-L770).

"From disk" here means "from the file, as the kernel returns it". PostgreSQL 12 opens data files without `O_DIRECT`, so both the server and `pg_read_binary_file()` read through the [OS page cache](../../../glossary.md#os-page-cache) [md.c:450](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L450) [md.c#_mdfd_openseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1071). PostgreSQL 12 has no setting that opens data files with `O_DIRECT`, so the check cannot bypass that cache. See [Open Questions](#open-questions) for what that means for storage-level faults.

The relation's files come from `pg_relation_filepath()`, which also resolves mapped catalogs and tablespaces [dbsize.c#pg_relation_filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1022). A [fork](../../../glossary.md#fork) larger than `RELSEG_SIZE` blocks continues in files `.1`, `.2` and so on, managed by the [storage manager](../../../glossary.md#storage-manager). `mdnblocks()` stops at the first segment shorter than `RELSEG_SIZE`, so blocks in later files are unreachable [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755). An attempt to read such a block fails with "previous segment is only N blocks" [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1180-L1205). The script applies the same rule: it reads only reachable blocks and reports data in later files as a layout problem.

### Logic map

```text
psql -v table_name=... -f tid_check.sql
 │
 ├─[1] superuser? pageinspect >= 1.6? permanent ordinary table? index_name belongs to it? primary? ──no──► ERROR, exit 3
 ├─[2] geometry from pg_control_init(); byte order from bytes 8-11 of global/pg_control ──unknown──► ERROR
 ├─[3] self-test: decode pg_class block 0 from disk, compare with heap_page_items() ──mismatch──► ERROR
 ├─[4] CHECKPOINT (no lock yet: shrinks the locked checkpoint)
 ├─[5] BEGIN; LOCK table IN SHARE ROW EXCLUSIVE MODE; PREPARE on the TOAST table (ACCESS SHARE); CHECKPOINT
 ├─[6] list the table, its TOAST table and their live, ready B-tree indexes; stat every segment file
 ├─[7] HEAP PASS: read every reachable heap page in chunks
 │        page state ──not ok──► keep the state, decode nothing
 │                   ──ok──────► one code per line pointer (unused / normal / redirect / dead / heap-only / corrupt)
 ├─[8] CHECKPOINT
 ├─[9] INDEX PASS: read every index page in chunks
 │        page state, including item bounds ──not ok──► keep the state, decode nothing
 │        live leaf page ──► bt_page_items(bytea) ──► heap TID of every item except the high key
 ├─[10] re-read once every page whose state was not ok or all_zero
 ├─[11] verdict for every TID against the heap codes; print the report; COMMIT
 └─[12] VERDICT, in a new transaction
          any file path changed since [6]? ──yes──► INCONCLUSIVE
          confirmed problem (layout, bad page repeated on re-read, dangling TID)? ──yes──► FAIL
          a page that failed once but passed on re-read? ──yes──► INCONCLUSIVE
          no TID checked? ──yes──► NOTHING CHECKED
          otherwise ──► PASS
```

Keyed evidence for the map:

- [1] [utility.c#CheckPointStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L756-L770), [lock.c#recovery-lock-limit](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766), [pageinspect--1.5--1.6.sql](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L82-L99).
- [2] [pg_controldata.c#pg_control_init](../../../../raw/postgres-12/src/backend/utils/misc/pg_controldata.c#L265-L350), [pg_control.h#ControlFileData](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L100-L122), [pg_control.h:25](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L25).
- [3] [itemid.h#ItemIdData](../../../../raw/postgres-12/src/include/storage/itemid.h#L17-L41), [heapfuncs.c#heap_page_items](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L128-L283).
- [4], [5], [8] [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1785-L1846), [utility.c#LockStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L740-L749), [parse_relation.c#addRangeTableEntry](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1229-L1254).
- [6] [vacuum.c#vac_open_indexes](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1869-L1918), [relcache.c:4394](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4394), [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755).
- [7], [9] [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162), [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-12/src/include/access/nbtree.h#L55-L78), [nbtree.h#P_HIKEY](../../../../raw/postgres-12/src/include/access/nbtree.h#L181-L219).
- [10] [basebackup.c#torn-block-retry](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1470-L1493).

### When an index TID is valid

The check rests on one invariant: **a B-tree entry on a ready index never names an `LP_UNUSED` line pointer, a heap-only tuple, or a line pointer or block that does not exist.** Four mechanisms in PostgreSQL 12 maintain it.

1. **Index entries point at chain roots.** With [HOT](../../../glossary.md#hot), a row version that changes no indexed column and fits on the same page gets no index entry. It is marked `HEAP_ONLY_TUPLE`, and the index keeps pointing at the first version, the root [README.HOT:45-55](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L45-L55) [htup_details.h:280-281](../../../../raw/postgres-12/src/include/access/htup_details.h#L280-L281). An index built later also points at roots [README.HOT:296-301](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L296-L301).
2. **[Pruning](../../../glossary.md#pruning) never frees a root.** A dead root becomes `LP_DEAD`, or `LP_REDIRECT` pointing at the live chain member. Only heap-only tuples become `LP_UNUSED` [pruneheap.c#heap_prune_chain](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L384-L411) [pruneheap.c:570-615](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L570-L615). The README states the reason: the root "has to remain" until VACUUM has removed the index entries [README.HOT:116-122](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L116-L122).
3. **VACUUM removes index entries before it frees line pointers.** `lazy_scan_heap()` calls `lazy_vacuum_index()` for every index, then `lazy_vacuum_heap()`, which sets the dead items `LP_UNUSED` [vacuumlazy.c:760-785](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L760-L785) [vacuumlazy.c#lazy_vacuum_page](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1591-L1615). With `INDEX_CLEANUP` off, it frees nothing and leaves the items dead [vacuumlazy.c:1238-1265](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1238-L1265). The [Line pointer](../../../glossary.md#line-pointer) glossary entry records the same rule.
4. **A heap page's line-pointer array never shrinks in 12.** Compaction by `PageRepairFragmentation()` moves tuples but never stores a new `pd_lower` [bufpage.c#PageRepairFragmentation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L482-L570). So a TID's offset stays inside the array for as long as the page exists.

Two kinds of index fall outside the invariant, so the script skips them and says so. VACUUM processes only indexes marked `indisready`. The source explains that a not-ready index probably comes from a `CREATE INDEX CONCURRENTLY` that "failed early in execution", and that what it holds is "too corrupt to be processable" [vacuum.c#vac_open_indexes](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1869-L1918). An index with `indislive = false` is being dropped and is missing from the relation's index list [relcache.c:4394](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4394). Invalid but ready indexes are vacuumed and are checked. The script checks B-tree indexes only, and skips the others with a note. PostgreSQL 12's pageinspect has `bytea` item decoders for hash pages and for GIN's compressed posting lists, but none for GiST or SP-GiST [pageinspect--1.5--1.6.sql:35-43](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L35-L43) [pageinspect--1.5.sql:271-279](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L271-L279); hash support is an [open question](#open-questions).

The heap pass turns each line pointer into a one-byte code. The final join turns each TID into a verdict:

| Heap slot the TID names | Valid for a ready index? | Reason | Verdict |
|---|---|---|---|
| Block at or past the reachable end | No | Every block an entry names was written when the heap grew, and VACUUM truncates only trailing pages whose line pointers are all unused, "because we'd not have cleaned out their index entries" otherwise [vacuumlazy.c#count_nondeletable_pages](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2108-L2119) | `heap_block_missing` |
| Block whose page is all zeros | No, once the checkpoint has run | A new block is written as zeros first [bufmgr.c#ReadBuffer_common](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L866-L872), but its tuple was in a dirty buffer that the checkpoint wrote | `heap_page_all_zero` |
| Block whose page fails validation | Unknown | The page cannot be decoded safely | `heap_page_bad_header`, `heap_page_bad_checksum`, `heap_page_short_read` |
| Offset past the last line pointer | No | Mechanism 4 | `offset_beyond_line_pointers` |
| `LP_UNUSED` | No | Mechanisms 2 and 3 | `line_pointer_unused` |
| `LP_NORMAL`, flagged `HEAP_ONLY_TUPLE` | No | Mechanism 1 | `points_to_heap_only_tuple` |
| `LP_NORMAL` whose storage lies outside the page | No | The line pointer itself is corrupt | `line_pointer_corrupt` |
| `LP_NORMAL`, not heap-only | Yes | An ordinary row version | `ok` |
| `LP_REDIRECT` | Yes | A pruned HOT root (mechanism 2) | `ok` |
| `LP_DEAD` | Yes | A dead root waiting for VACUUM (mechanisms 2 and 3) | `ok` |

The line-pointer states and their meanings come from `itemid.h` [itemid.h#ItemIdData](../../../../raw/postgres-12/src/include/storage/itemid.h#L17-L41). The measured healthy table had all of the accepted kinds at once: 18,077 normal line pointers, 325 redirect roots that the indexes point at, 101 dead line pointers, 1,634 heap-only tuples and 2,720 unused line pointers.

### Making the files agree with memory: the lock and two checkpoints

**Why the files lag.** A change goes into a shared buffer first. The [checkpointer](../../../glossary.md#checkpointer), the [background writer](../../../glossary.md#background-writer) or a backend writes the page later, and nothing orders the writes of two different pages. An `INSERT` adds the heap tuple before its index entries [nodeModifyTable.c#ExecInsert](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L586-L593), yet the index page may reach the file first. Read at that moment, the files show an index entry for a tuple the heap file does not yet hold. That is a false finding, not data loss.

**What a checkpoint guarantees.** `BufferSync()` marks every buffer that is dirty when the [checkpoint](../../../glossary.md#checkpoint) starts and writes it before the checkpoint ends. It skips buffers of non-permanent relations unless the checkpoint is a shutdown or end-of-recovery checkpoint or was asked to flush everything [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1785-L1846). The SQL `CHECKPOINT` command requests an immediate checkpoint without that request, and waits for it [utility.c#CheckPointStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L756-L770). Therefore a change made in memory before `CHECKPOINT` starts is in the file when `CHECKPOINT` returns, but only for permanent tables. An [unlogged](../../../glossary.md#unlogged-relation) table's dirty buffers stay unwritten, and a temporary table's pages live in backend-local buffers [rel.h#RelationUsesLocalBuffers](../../../../raw/postgres-12/src/include/utils/rel.h#L523-L527), so the script refuses both.

**What the lock blocks.** `LOCK TABLE` must run inside a transaction block [utility.c#LockStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L740-L749). The script takes `SHARE ROW EXCLUSIVE`, which conflicts with every [lock mode](../../../glossary.md#lock-mode) from `ROW EXCLUSIVE` upward [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L104) [mvcc.sgml#SHARE-ROW-EXCLUSIVE](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L958-L978):

| Command on the table | Lock it needs | Blocked during the check? |
|---|---|---|
| `INSERT`, `UPDATE`, `DELETE` | `ROW EXCLUSIVE` [mvcc.sgml#ROW-EXCLUSIVE](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L910) | Yes |
| `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY`, `REINDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE` [mvcc.sgml#SHARE-UPDATE-EXCLUSIVE](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L912-L936) | Yes |
| `CREATE INDEX`; `REINDEX`, which also takes `ACCESS EXCLUSIVE` on the index | `SHARE` on the table [mvcc.sgml#SHARE](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L938-L956) [indexcmds.c:2397-2402](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2397-L2402) | Yes |
| `DROP INDEX` | `ACCESS EXCLUSIVE` on the table, or `SHARE UPDATE EXCLUSIVE` with `CONCURRENTLY` [tablecmds.c:1379-1380](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1379-L1380) | Yes |
| `VACUUM FULL`, `CLUSTER`, `TRUNCATE` | `ACCESS EXCLUSIVE` [mvcc.sgml#ACCESS-EXCLUSIVE](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1002-L1030) | Yes |
| `SELECT` | `ACCESS SHARE` | No |

The lock is self-exclusive, so two checks of one table cannot overlap. A standby cannot take it, because a backend in recovery may not take a relation lock stronger than `ROW EXCLUSIVE` [lock.c#recovery-lock-limit](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766). [Autovacuum](../../../glossary.md#autovacuum) runs `vacuum_rel()`, which takes `SHARE UPDATE EXCLUSIVE` unless the VACUUM is `FULL` [vacuum.c:1680-1681](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1681), so no worker can start on the table during the check. A worker that already holds that lock is cancelled once `LOCK TABLE` has waited `deadlock_timeout`, unless the worker is preventing wraparound [proc.c:1312-1349](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1312-L1349). An anti-wraparound worker makes `LOCK TABLE` wait until `lock_wait` expires, and the script stops.

**The TOAST table is the exception.** `LOCK TABLE` accepts only tables, partitioned tables and views, so it refuses a TOAST table [lockcmds.c#RangeVarCallbackForLockTable](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L82-L104). Autovacuum vacuums TOAST tables on their own, not together with their parent [autovacuum.c:2108-2113](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2108-L2113). So a TOAST VACUUM can run during the check. The script handles it in two ways:

- **No truncation.** `PREPARE tc_toast_lock AS SELECT 1 FROM <toast table>` parses a query without planning it [prepare.c#PrepareQuery](../../../../raw/postgres-12/src/backend/commands/prepare.c#L128-L180). Parsing opens the table with `ACCESS SHARE` and keeps the lock until commit [parse_relation.c#addRangeTableEntry](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1229-L1254). VACUUM truncates a heap only when its conditional `ACCESS EXCLUSIVE` request succeeds [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1902), and this lock makes that request fail.
- **Heap first, then a checkpoint, then the index.** Suppose the heap pass saw a TOAST line pointer as `LP_UNUSED`. VACUUM sets it unused only after it has removed every index entry for that TID (mechanism 3), so the removal happened in memory before the heap pass read the page. The checkpoint in step [8] starts after that and writes the index page. The index pass, which follows, therefore cannot see the removed entry. No other process writes TOAST rows during the check, because the table lock stops every writer of the parent table.

The script does not plan a query on the table to lock its indexes. Planning would read each B-tree's [metapage](../../../glossary.md#metapage) into shared buffers, because the planner asks `_bt_getrootheight()` for the tree height [plancat.c:409-413](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L413) [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L586-L626). The table lock already blocks every command that rewrites the table's indexes. Two rare commands remain: `REINDEX` of the TOAST index and `ALTER INDEX ... SET TABLESPACE`. The verdict step catches them, because it compares every file path with a fresh catalog lookup after commit.

Readers still run during the check, and they can still change pages:

| Event during the check | State before | State after | Effect on the check |
|---|---|---|---|
| A `SELECT` prunes a heap page | Root `LP_NORMAL`, heap-only members | Root `LP_REDIRECT` or `LP_DEAD`; dead heap-only members `LP_UNUSED` | None: both states are valid targets |
| A query sets [hint bits](../../../glossary.md#hint-bits), or an index scan marks index items `LP_DEAD` [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1793) | Page clean | Page dirty, written later | None on TIDs; the page may be written while it is read |
| That write overlaps the script's read | One version on disk | Two versions on disk, briefly | A [torn page](../../../glossary.md#torn-page) can fail validation once; step [10] re-reads it |
| TOAST VACUUM removes entries and frees line pointers | Entry present, line pointer dead | Entry gone, line pointer unused | Handled by the heap-first order above |
| `REINDEX` of the TOAST index commits | Old file path | New file path | Verdict `INCONCLUSIVE` |

Base backups face the same torn reads. `basebackup.c` re-reads a block once when its checksum fails, because the first read may have caught half of a concurrent write [basebackup.c#torn-block-retry](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1470-L1493). The script uses the same rule.

### Never handing a damaged page to pageinspect

Data loss usually comes with damaged pages, and PostgreSQL 12's `bytea` decoders do not defend against them:

- `bt_page_items(bytea)` checks only that the input holds a page header and is not a metapage [btreefuncs.c#bt_page_items_bytea](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L422-L466). For each item it then reads the tuple at the line pointer's offset and dumps as many bytes as the tuple's own size field claims, with no bounds check [btreefuncs.c#bt_page_print_tuples](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L256-L299). `ItemIdIsValid()` only tests that the pointer is non-null [itemid.h:86](../../../../raw/postgres-12/src/include/storage/itemid.h#L86), and `PageGetItem()` checks storage only with an assertion [bufpage.h#PageGetItem](../../../../raw/postgres-12/src/include/storage/bufpage.h#L340-L345).
- `heap_page_items()` takes its item count from `pd_lower` without bounding it by the page size [heapfuncs.c:166](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L166). It does check that each tuple lies inside the page [heapfuncs.c:200-210](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L200-L210). It then copies `lp_len - t_hoff` bytes, without first checking that `t_hoff` is at most `lp_len` [heapfuncs.c:229-234](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L229-L234).

A garbage size could make either function read past the end of its input. The script therefore never calls a decoder on a page it has not validated. It also never calls `heap_page_items()` on table data, except in the one-page self-test.

**Byte order.** The script decodes fixed-offset fields with `get_byte()`. That works only if it knows how this server stores integers. `pg_control_version` sits 8 bytes into `global/pg_control` [pg_control.h#ControlFileData](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L100-L122) and is 1201 [pg_control.h:25](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L25). `pg_control_system()` returns the same number [pg_controldata.c#pg_control_system](../../../../raw/postgres-12/src/backend/utils/misc/pg_controldata.c#L32-L77). Comparing the raw bytes with that number shows the byte order.

**Page validation**, in `pg_temp.tc_state()`:

| Check | Why | Source |
|---|---|---|
| Exactly `BLCKSZ` bytes were read | A shorter result means the file shrank or is missing | [genfile.c:165-172](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L165-L172) |
| All zeros → `all_zero` | The server accepts such pages as new | [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162) |
| With checksums on: `pd_checksum` equals `page_checksum(page, block)` | The checksum mixes in the absolute block number across segments, as base backups compute it | [rawpage.c#page_checksum](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L299-L324), [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L187-L208), [basebackup.c:1480](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1480) |
| `PageIsVerified()`'s header test: valid flag bits, `pd_lower <= pd_upper <= pd_special <= BLCKSZ`, aligned `pd_special` | Same as the server's own read check | [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162), [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L152-L185) |
| Stricter: `pd_lower >= 24`, the page size and layout version 4 in `pd_pagesize_version`, and `pd_special` equal to `BLCKSZ` for heap pages or `BLCKSZ - 16` for B-tree pages | `PageInit()` sets `pd_special` to the page size minus the special space, which is `BTPageOpaqueData` on B-tree pages | [bufpage.c#PageInit](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L42-L58), [nbtpage.c#_bt_pageinit](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L926-L929), [bufpage.h:199](../../../../raw/postgres-12/src/include/storage/bufpage.h#L199) |
| B-tree block 0 must carry `BTP_META`; no other block may | Block 0 is the metapage | [nbtree.h:131](../../../../raw/postgres-12/src/include/access/nbtree.h#L131), [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L50-L68) |
| B-tree items: state `LP_NORMAL`, or `LP_DEAD` as index scans set it; inside `[pd_upper, pd_special)`; aligned; and an `IndexTupleData` size of at least the header, null bitmap included, and at most `lp_len` | Exactly what `bt_page_print_tuples()` would otherwise trust | [itup.h#IndexTupleData](../../../../raw/postgres-12/src/include/access/itup.h#L35-L90), [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1793) |

**Line pointers.** `ItemIdData` is a C bitfield: `lp_off:15`, `lp_flags:2`, `lp_len:15` [itemid.h#ItemIdData](../../../../raw/postgres-12/src/include/storage/itemid.h#L25-L41). The C compiler decides how bitfields are laid out, so the script tests its own decoding first. It reads `pg_class` block 0 from disk, checks that every item is safe for `heap_page_items()`, and compares its own `lp_off`, `lp_flags`, `lp_len` and `t_infomask2` with that function's output. If any value differs, the script stops. On a heap page it then reads `t_infomask2` at offset 18 of each normal tuple to find `HEAP_ONLY_TUPLE` [htup_details.h#HeapTupleHeaderData](../../../../raw/postgres-12/src/include/access/htup_details.h#L152-L175).

**Which B-tree items carry heap TIDs.** Only items on live leaf pages: `BTP_LEAF` set, and `BTP_DELETED`, `BTP_HALF_DEAD` and `BTP_META` clear [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-12/src/include/access/nbtree.h#L55-L78). A half-dead leaf keeps one stub item whose TID field stores a block number, not a heap TID [nbtpage.c:1648-1665](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1648-L1665). On a page with a right sibling, item 1 is the high key, which points at no heap tuple [nbtree.h#P_HIKEY](../../../../raw/postgres-12/src/include/access/nbtree.h#L181-L219). An all-zero index page is legitimate, because a crash after extending the index can leave one, and VACUUM recycles it [nbtpage.c#_bt_page_recyclable](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L936-L952).

**Speed.** The per-page loops are [PL/pgSQL](../../../glossary.md#plpgsql), which `initdb` installs in every database [initdb.c#load_plpgsql](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1999-L2001). The byte readers `tc_u16()` and `tc_u32()` are SQL functions that are deliberately not `STRICT`. The planner does not inline a strict SQL function whose body contains a non-strict construct such as `CASE` [clauses.c#inline_function](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4600-L4602), and an inlined reader avoids a function call for every two bytes decoded. The PL/pgSQL code keeps `CASE` out of `IF` conditions, because PL/pgSQL reads an `IF` condition only up to its first `THEN` [pl_gram.y:2430-2431](../../../../raw/postgres-12/src/pl/plpgsql/src/pl_gram.y#L2430-L2431).

### The check

Save the block below as `tid_check.sql` and run it with psql 12 as a superuser, on the primary, in the database that holds the table:

```sh
psql -X -d mydb -v table_name=public.orders -f tid_check.sql
```

| psql variable | Default | Effect |
|---|---|---|
| `table_name` | required | The table to check, schema-qualified if needed. It must be a permanent ordinary table; for a partitioned table, run the check on each partition. |
| `index_name` | `''`, every B-tree index | Check one index of the table or of its TOAST table. The heap is read in full either way. |
| `include_toast` | `on` | Also check the TOAST table's index against the TOAST heap. |
| `chunk_blocks` | `128` | Blocks per `pg_read_binary_file()` call, so 1 MB per read at 8 kB blocks. |
| `max_findings` | `50` | Detail rows printed. |
| `lock_wait` | `10s` | `lock_timeout` while waiting for the table lock. |
| `statement_wait` | `2h` | `statement_timeout` for every statement. Raise it for very large tables; see [Measured results](#measured-results) for the cost per page and per entry. |

The script changes three settings with `SET`, so they last until the psql session ends; all three are `PGC_USERSET` and need no reload or restart [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2387) [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2389-L2398) [guc.c#client_min_messages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4232-L4241). `statement_timeout` and `lock_timeout` default to 0, which disables them; the script sets them so that neither a long read nor a lock queue runs unbounded. `client_min_messages` defaults to `notice`; the script sets `warning` so that notices from its own `DROP TABLE IF EXISTS` do not clutter the report.

The table stays locked against writes from step [5] until `COMMIT` after the report. The lock time is two checkpoints plus one read of the table, its TOAST table and the checked indexes. The unlocked `CHECKPOINT` in step [4] exists to shorten the first locked one.

```sql
-- tid_check.sql: PostgreSQL 12 on-disk index-to-heap TID check (contrib pageinspect).
-- Reads relation files with pg_read_binary_file(), never through shared buffers.
-- Run as a superuser on the primary:
--   psql -X -d mydb -v table_name=public.orders -f tid_check.sql
-- Optional psql variables (defaults in brackets):
--   index_name      check only this B-tree index of the table ['' = every B-tree index]
--   include_toast   also check the TOAST table's index [on]
--   chunk_blocks    blocks read per pg_read_binary_file() call [128]
--   max_findings    detail rows to print [50]
--   lock_wait       lock_timeout for the table lock [10s]
--   statement_wait  statement_timeout for each statement [2h]
\set ON_ERROR_STOP on
\if :{?table_name}
\else
\echo 'set table_name first, for example: psql -v table_name=public.orders -f tid_check.sql'
\quit
\endif
\if :{?index_name}
\else
\set index_name ''
\endif
\if :{?include_toast}
\else
\set include_toast on
\endif
\if :{?chunk_blocks}
\else
\set chunk_blocks 128
\endif
\if :{?max_findings}
\else
\set max_findings 50
\endif
\if :{?lock_wait}
\else
\set lock_wait 10s
\endif
\if :{?statement_wait}
\else
\set statement_wait 2h
\endif
SET /* wiki_tid_check */ statement_timeout = :'statement_wait';
SET /* wiki_tid_check */ lock_timeout = :'lock_wait';
SET /* wiki_tid_check */ client_min_messages = warning;

-- 1. Preconditions.  Every later step relies on them.  Each query returns
-- exactly one row, so \gset always sets or unsets its variables.
SELECT /* wiki_tid_check */
       coalesce((SELECT rolsuper FROM pg_roles WHERE rolname = current_user), false) AS tc_superuser,
       n.nspname AS tc_pi,
       coalesce(to_regprocedure(quote_ident(n.nspname) || '.bt_page_items(bytea)') IS NOT NULL
                AND to_regprocedure(quote_ident(n.nspname) || '.heap_page_items(bytea)') IS NOT NULL
                AND to_regprocedure(quote_ident(n.nspname) || '.page_checksum(bytea,integer)') IS NOT NULL,
                false) AS tc_pageinspect_ok
FROM (VALUES (1)) AS one
LEFT JOIN pg_extension AS e ON e.extname = 'pageinspect'
LEFT JOIN pg_namespace AS n ON n.oid = e.extnamespace \gset
\if :tc_superuser
\else
DO /* wiki_tid_check */ $$ BEGIN RAISE EXCEPTION 'tid_check: run as a superuser'; END $$;
\endif
\if :{?tc_pi}
\else
DO /* wiki_tid_check */ $$ BEGIN RAISE EXCEPTION 'tid_check: CREATE EXTENSION pageinspect in this database first'; END $$;
\endif
\if :tc_pageinspect_ok
\else
DO /* wiki_tid_check */ $$ BEGIN RAISE EXCEPTION 'tid_check: pageinspect 1.6 or later is required (ALTER EXTENSION pageinspect UPDATE)'; END $$;
\endif
SELECT /* wiki_tid_check */
       c.relkind = 'r' AND c.relpersistence = 'p' AS tc_table_ok,
       CASE WHEN :'index_name' = '' THEN true
            ELSE EXISTS (SELECT 1 FROM pg_index AS i
                         WHERE i.indexrelid = to_regclass(:'index_name')
                           AND i.indrelid IN (c.oid, c.reltoastrelid)) END AS tc_index_ok,
       NOT pg_is_in_recovery() AS tc_primary
FROM pg_class AS c
WHERE c.oid = :'table_name'::regclass \gset
\if :tc_table_ok
\else
DO /* wiki_tid_check */ $$ BEGIN RAISE EXCEPTION 'tid_check: table_name must be a permanent ordinary table (not unlogged, temporary, partitioned, a view or a materialized view)'; END $$;
\endif
\if :tc_index_ok
\else
DO /* wiki_tid_check */ $$ BEGIN RAISE EXCEPTION 'tid_check: index_name must name an index of table_name or of its TOAST table'; END $$;
\endif
\if :tc_primary
\else
DO /* wiki_tid_check */ $$ BEGIN RAISE EXCEPTION 'tid_check: run on the primary; LOCK TABLE ... SHARE ROW EXCLUSIVE is refused during recovery'; END $$;
\endif

-- 2. Page geometry and byte order.  pg_control_version sits 8 bytes into
-- global/pg_control; reading it raw shows how this server stores integers.
SELECT /* wiki_tid_check */
       ci.database_block_size AS tc_blcksz,
       ci.blocks_per_segment AS tc_relseg,
       ci.max_data_alignment AS tc_maxalign,
       CASE WHEN ci.data_page_checksum_version <> 0 THEN 'true' ELSE 'false' END AS tc_checksums,
       CASE cs.pg_control_version
         WHEN get_byte(b.v, 0) + 256 * get_byte(b.v, 1) + 65536 * get_byte(b.v, 2)
              + 16777216::bigint * get_byte(b.v, 3) THEN 'true'
         WHEN get_byte(b.v, 3) + 256 * get_byte(b.v, 2) + 65536 * get_byte(b.v, 1)
              + 16777216::bigint * get_byte(b.v, 0) THEN 'false'
       END AS tc_le
FROM pg_control_init() AS ci
CROSS JOIN pg_control_system() AS cs
CROSS JOIN (SELECT pg_read_binary_file('global/pg_control', 8, 4) AS v) AS b \gset
\if :{?tc_le}
\else
DO /* wiki_tid_check */ $$ BEGIN RAISE EXCEPTION 'tid_check: could not determine the byte order from global/pg_control'; END $$;
\endif

-- 3. Session-local helpers.  They decode PageHeaderData, ItemIdData and
-- IndexTupleData with get_byte(), so a damaged page is rejected before any
-- pageinspect decoder sees it.  tc_u16 and tc_u32 are plain SQL so the
-- planner can inline them.
CREATE /* wiki_tid_check */ OR REPLACE FUNCTION pg_temp.tc_u16(p bytea, o int, le boolean) RETURNS int
LANGUAGE sql IMMUTABLE AS $$
  SELECT CASE WHEN le THEN get_byte(p, o) + 256 * get_byte(p, o + 1)
              ELSE 256 * get_byte(p, o) + get_byte(p, o + 1) END $$;
CREATE /* wiki_tid_check */ OR REPLACE FUNCTION pg_temp.tc_u32(p bytea, o int, le boolean) RETURNS bigint
LANGUAGE sql IMMUTABLE AS $$
  SELECT CASE WHEN le THEN pg_temp.tc_u16(p, o, le) + 65536::bigint * pg_temp.tc_u16(p, o + 2, le)
              ELSE 65536::bigint * pg_temp.tc_u16(p, o, le) + pg_temp.tc_u16(p, o + 2, le) END $$;
-- Line pointer i: lp_off:15, lp_flags:2, lp_len:15, allocated from the low
-- bit on a little-endian build.  Step 4 checks this against heap_page_items().
CREATE /* wiki_tid_check */ OR REPLACE FUNCTION pg_temp.tc_lp(p bytea, i int, le boolean,
                                         OUT off int, OUT flags int, OUT len int)
LANGUAGE sql IMMUTABLE AS $$
  SELECT CASE WHEN le THEN (w & 32767)::int ELSE (w >> 17)::int END,
         ((w >> 15) & 3)::int,
         CASE WHEN le THEN (w >> 17)::int ELSE (w & 32767)::int END
  FROM (SELECT pg_temp.tc_u32(p, 24 + 4 * (i - 1), le) AS w) AS s $$;
-- Page state: 'ok', 'all_zero', 'short_read', 'bad_checksum', 'bad_header',
-- and for B-tree pages also 'bad_meta' and 'bad_items'.  kind is 'h' (heap)
-- or 'b' (B-tree); cks is page_checksum() masked to 16 bits, or NULL when
-- data checksums are off.  The header test is PageIsVerified()'s, plus
-- pd_lower >= 24, the page type's pd_special and pd_pagesize_version.
CREATE /* wiki_tid_check */ OR REPLACE FUNCTION pg_temp.tc_state(p bytea, blk bigint, kind text, cks int,
                                            blcksz int, maxalign int, le boolean) RETURNS text
LANGUAGE plpgsql IMMUTABLE AS $$
DECLARE
  lower int; upper int; special int; btflags int; w bigint;
  off int; flags int; len int; info int; size int; minsize int;
  -- PL/pgSQL ends an IF condition at its first THEN, so no CASE inside one.
  special_want int := CASE kind WHEN 'h' THEN blcksz ELSE blcksz - 16 END;
  nullsize int := (12 + maxalign - 1) / maxalign * maxalign;   -- MAXALIGN(8 + 4)
BEGIN
  IF p IS NULL OR length(p) <> blcksz THEN RETURN 'short_read'; END IF;
  IF btrim(p, '\x00'::bytea) = ''::bytea THEN RETURN 'all_zero'; END IF;
  upper := pg_temp.tc_u16(p, 14, le);
  IF upper = 0 THEN RETURN 'bad_header'; END IF;
  IF cks IS NOT NULL AND cks <> pg_temp.tc_u16(p, 8, le) THEN RETURN 'bad_checksum'; END IF;
  lower := pg_temp.tc_u16(p, 12, le);
  special := pg_temp.tc_u16(p, 16, le);
  IF (pg_temp.tc_u16(p, 10, le) & 65528) <> 0 OR lower < 24 OR lower > upper
     OR upper > special OR special > blcksz OR special % maxalign <> 0
     OR special <> special_want OR pg_temp.tc_u16(p, 18, le) <> blcksz + 4 THEN
    RETURN 'bad_header';
  END IF;
  IF kind = 'h' THEN RETURN 'ok'; END IF;
  btflags := pg_temp.tc_u16(p, blcksz - 4, le);           -- BTPageOpaqueData.btpo_flags
  IF blk = 0 THEN
    RETURN CASE WHEN btflags & 8 <> 0 THEN 'ok' ELSE 'bad_meta' END;   -- BTP_META
  END IF;
  IF btflags & 8 <> 0 THEN RETURN 'bad_items'; END IF;
  -- Every item must lie inside the tuple area, and its IndexTupleData size
  -- must fit in its line pointer, before bt_page_items() may read it.
  FOR i IN 1 .. (lower - 24) / 4 LOOP
    w := pg_temp.tc_u32(p, 24 + 4 * (i - 1), le);
    off := CASE WHEN le THEN w & 32767 ELSE w >> 17 END;
    flags := (w >> 15) & 3;
    len := CASE WHEN le THEN w >> 17 ELSE w & 32767 END;
    IF flags NOT IN (1, 3) OR len < 8 OR off < upper OR off + len > special
       OR off % maxalign <> 0 THEN
      RETURN 'bad_items';
    END IF;
    info := pg_temp.tc_u16(p, off + 6, le);
    size := info & 8191;                                   -- INDEX_SIZE_MASK
    minsize := CASE WHEN info & 32768 <> 0 THEN nullsize ELSE 8 END;   -- INDEX_NULL_MASK
    IF size > len OR size < minsize THEN
      RETURN 'bad_items';
    END IF;
  END LOOP;
  RETURN 'ok';
END $$;
-- One byte per heap line pointer: 0 unused, 1 normal, 2 redirect, 3 dead,
-- 4 normal but a heap-only tuple, 5 normal with storage outside the page.
CREATE /* wiki_tid_check */ OR REPLACE FUNCTION pg_temp.tc_heap_codes(p bytea, maxalign int, le boolean) RETURNS bytea
LANGUAGE plpgsql IMMUTABLE STRICT AS $$
DECLARE
  upper int := pg_temp.tc_u16(p, 14, le);
  special int := pg_temp.tc_u16(p, 16, le);
  n int := (pg_temp.tc_u16(p, 12, le) - 24) / 4;
  codes bytea; w bigint; off int; flags int; len int; code int;
BEGIN
  IF n <= 0 THEN RETURN ''::bytea; END IF;
  codes := decode(repeat('00', n), 'hex');
  FOR i IN 1 .. n LOOP
    w := pg_temp.tc_u32(p, 24 + 4 * (i - 1), le);
    off := CASE WHEN le THEN w & 32767 ELSE w >> 17 END;
    flags := (w >> 15) & 3;
    len := CASE WHEN le THEN w >> 17 ELSE w & 32767 END;
    IF flags <> 1 THEN
      code := flags;
    ELSIF len < 23 OR off < upper OR off + len > special OR off % maxalign <> 0 THEN
      code := 5;
    ELSIF pg_temp.tc_u16(p, off + 18, le) & 32768 <> 0 THEN  -- t_infomask2 & HEAP_ONLY_TUPLE
      code := 4;
    ELSE
      code := 1;
    END IF;
    codes := set_byte(codes, i - 1, code);
  END LOOP;
  RETURN codes;
END $$;
CREATE /* wiki_tid_check */ OR REPLACE FUNCTION pg_temp.tc_verdict(hoff int, hstate text, codes bytea) RETURNS text
LANGUAGE sql IMMUTABLE AS $$
  SELECT CASE
    WHEN hoff IS NULL THEN NULL
    WHEN hoff < 1 THEN 'invalid_tid'
    WHEN hstate IS NULL THEN 'heap_block_missing'
    WHEN hstate <> 'ok' THEN 'heap_page_' || hstate
    WHEN hoff > length(codes) THEN 'offset_beyond_line_pointers'
    ELSE CASE get_byte(codes, hoff - 1)
           WHEN 0 THEN 'line_pointer_unused'
           WHEN 4 THEN 'points_to_heap_only_tuple'
           WHEN 5 THEN 'line_pointer_corrupt'
           ELSE 'ok' END END $$;

-- 4. Self-test of the decoding on pg_class block 0, read from disk.
-- heap_page_items() runs only if every item passes the checks above, and
-- only if every normal item's t_hoff fits in its length.
WITH /* wiki_tid_check */ pg AS (
  SELECT p, pg_temp.tc_state(p, 0, 'h',
           CASE WHEN :tc_checksums AND length(p) = :tc_blcksz
                THEN :"tc_pi".page_checksum(p, 0)::int & 65535 END,
           :tc_blcksz, :tc_maxalign, :tc_le) AS state
  FROM (SELECT pg_read_binary_file(pg_relation_filepath('pg_class'), 0, :tc_blcksz) AS p) AS r
), ours AS (
  SELECT i, lp.off, lp.flags, lp.len,
         CASE WHEN lp.flags = 1 THEN pg_temp.tc_u16(pg.p, lp.off + 18, :tc_le) END AS infomask2,
         CASE WHEN lp.flags = 1 THEN get_byte(pg.p, lp.off + 22) <= lp.len END AS hoff_fits
  FROM pg
  CROSS JOIN LATERAL generate_series(1, (pg_temp.tc_u16(pg.p, 12, :tc_le) - 24) / 4) AS i
  CROSS JOIN LATERAL pg_temp.tc_lp(pg.p, i, :tc_le) AS lp
  WHERE pg.state = 'ok'
    AND position('\x05'::bytea IN pg_temp.tc_heap_codes(pg.p, :tc_maxalign, :tc_le)) = 0
), safe AS (
  SELECT count(*) > 0 AND bool_and(coalesce(hoff_fits, true)) AS ok FROM ours
), theirs AS (
  SELECT h.lp::int AS i, h.lp_off::int AS off, h.lp_flags::int AS flags, h.lp_len::int AS len,
         h.t_infomask2 AS infomask2
  FROM pg
  CROSS JOIN LATERAL :"tc_pi".heap_page_items(CASE WHEN (SELECT ok FROM safe) THEN pg.p END) AS h
)
SELECT /* wiki_tid_check */
       (SELECT state FROM pg) = 'ok'
       AND coalesce((SELECT ok FROM safe), false)
       AND (SELECT count(*) FROM ours) = (SELECT count(*) FROM theirs)
       AND NOT EXISTS (
         SELECT 1 FROM ours AS o FULL JOIN theirs AS t USING (i)
         WHERE o.off IS DISTINCT FROM t.off OR o.flags IS DISTINCT FROM t.flags
            OR o.len IS DISTINCT FROM t.len
            OR (o.flags = 1 AND o.infomask2 IS DISTINCT FROM t.infomask2)) AS tc_selftest_ok,
       (SELECT count(*) FROM ours) AS tc_selftest_items
\gset
\if :tc_selftest_ok
\echo 'self-test: line-pointer decoding matches heap_page_items() on' :tc_selftest_items 'items of pg_class block 0'
\else
DO /* wiki_tid_check */ $$ BEGIN RAISE EXCEPTION 'tid_check: decoding self-test failed; do not use this script on this platform'; END $$;
\endif

-- 5. Flush first without the lock, so the locked checkpoint has little to write.
CHECKPOINT /* wiki_tid_check */;

BEGIN /* wiki_tid_check */;
-- Blocks INSERT/UPDATE/DELETE, VACUUM, ANALYZE, CREATE INDEX and REINDEX on
-- the table; plain SELECT keeps working.
SELECT /* wiki_tid_check */ format('LOCK /* wiki_tid_check */ TABLE %s IN SHARE ROW EXCLUSIVE MODE', c.oid::regclass)
FROM pg_class AS c WHERE c.oid = :'table_name'::regclass \gexec
-- LOCK TABLE refuses TOAST tables.  PREPARE parses a query on the TOAST table
-- without planning it, and parsing keeps ACCESS SHARE until commit, so VACUUM
-- cannot truncate the TOAST table while it is read.
SELECT /* wiki_tid_check */ format('PREPARE /* wiki_tid_check */ tc_toast_lock AS SELECT 1 FROM %s', c.reltoastrelid::regclass)
FROM pg_class AS c
WHERE c.oid = :'table_name'::regclass AND c.reltoastrelid <> 0 \gexec
-- Everything written before the lock is now on its way to disk.
CHECKPOINT /* wiki_tid_check */;

-- 6. Relations, their files and their segment layout.
DROP /* wiki_tid_check */ TABLE IF EXISTS pg_temp.tc_rel, pg_temp.tc_seg, pg_temp.tc_heap_page,
                     pg_temp.tc_item, pg_temp.tc_recheck;
CREATE /* wiki_tid_check */ TEMP TABLE tc_rel AS
WITH /* wiki_tid_check */ heaps AS (
  SELECT c.oid AS relid, 'table'::text AS kind FROM pg_class AS c
  WHERE c.oid = :'table_name'::regclass
  UNION ALL
  SELECT c.reltoastrelid, 'toast' FROM pg_class AS c
  WHERE c.oid = :'table_name'::regclass AND c.reltoastrelid <> 0
    AND (:'include_toast'::boolean OR :'index_name' <> '')
), idx AS (
  SELECT i.indexrelid AS relid, h.kind || '_index' AS kind, i.indrelid AS heaprel,
         CASE WHEN am.amname <> 'btree' THEN 'skipped: not a B-tree index'
              WHEN NOT i.indislive THEN 'skipped: index is being dropped (indislive = false)'
              WHEN NOT i.indisready THEN 'skipped: VACUUM does not process it (indisready = false)'
         END AS skip
  FROM heaps AS h
  JOIN pg_index AS i ON i.indrelid = h.relid
  JOIN pg_class AS ic ON ic.oid = i.indexrelid
  JOIN pg_am AS am ON am.oid = ic.relam
  WHERE CASE WHEN :'index_name' = '' THEN true
              ELSE i.indexrelid = to_regclass(:'index_name') END
)
SELECT h.relid, h.kind, NULL::oid AS heaprel, NULL::text AS skip,
       pg_relation_filepath(h.relid) AS path
FROM heaps AS h
WHERE h.relid IN (SELECT heaprel FROM idx WHERE skip IS NULL)
UNION ALL
SELECT relid, kind, heaprel, skip, pg_relation_filepath(relid) FROM idx;

CREATE /* wiki_tid_check */ TEMP TABLE tc_seg AS
WITH /* wiki_tid_check */ RECURSIVE seg(relid, segno, path, bytes) AS (
  SELECT r.relid, 0::bigint, r.path, (pg_stat_file(r.path, true)).size
  FROM tc_rel AS r WHERE r.skip IS NULL
  UNION ALL
  SELECT s.relid, s.segno + 1, r.path || '.' || (s.segno + 1),
         (pg_stat_file(r.path || '.' || (s.segno + 1), true)).size
  FROM seg AS s JOIN tc_rel AS r ON r.relid = s.relid
  WHERE s.bytes IS NOT NULL
), shaped AS (
  SELECT s.*,
         min(s.segno) FILTER (WHERE s.bytes < :tc_relseg::bigint * :tc_blcksz)
           OVER (PARTITION BY s.relid) AS first_short
  FROM seg AS s WHERE s.bytes IS NOT NULL
)
SELECT relid, segno, path, bytes,
       -- blocks a fresh backend can reach: mdnblocks() stops at the first short segment
       CASE WHEN first_short IS NULL OR segno < first_short THEN :tc_relseg::bigint
            WHEN segno = first_short THEN bytes / :tc_blcksz
            ELSE 0 END AS read_blocks,
       concat_ws(', ',
         CASE WHEN bytes > :tc_relseg::bigint * :tc_blcksz THEN 'segment longer than RELSEG_SIZE' END,
         CASE WHEN bytes % :tc_blcksz <> 0 THEN 'partial trailing block' END,
         CASE WHEN segno > first_short AND bytes > 0 THEN 'data after a short segment' END) AS layout
FROM shaped;

-- 7. Read every reachable heap page (table and TOAST) from disk.
CREATE /* wiki_tid_check */ TEMP TABLE tc_heap_page (relid oid, blk bigint, state text, codes bytea,
                                PRIMARY KEY (relid, blk));
INSERT /* wiki_tid_check */ INTO tc_heap_page
SELECT pg.relid, pg.blk, pg.state,
       pg_temp.tc_heap_codes(CASE WHEN pg.state = 'ok' THEN pg.page END, :tc_maxalign, :tc_le)
FROM tc_rel AS r
JOIN tc_seg AS s ON s.relid = r.relid AND s.read_blocks > 0
CROSS JOIN LATERAL generate_series(0, s.read_blocks - 1, :chunk_blocks) AS c(first)
CROSS JOIN LATERAL (
  SELECT pg_read_binary_file(s.path, c.first * :tc_blcksz,
                             least(:chunk_blocks, s.read_blocks - c.first) * :tc_blcksz, true) AS buf,
         least(:chunk_blocks, s.read_blocks - c.first) AS n
  OFFSET 0) AS rb
CROSS JOIN LATERAL generate_series(0, rb.n - 1) AS k(i)
CROSS JOIN LATERAL (
  SELECT r.relid, s.segno * :tc_relseg + c.first + k.i AS blk,
         substring(rb.buf FROM (k.i * :tc_blcksz + 1)::int FOR :tc_blcksz) AS page
  OFFSET 0) AS p0
CROSS JOIN LATERAL (
  SELECT p0.relid, p0.blk, p0.page,
         pg_temp.tc_state(p0.page, p0.blk, 'h',
           CASE WHEN :tc_checksums AND length(p0.page) = :tc_blcksz
                THEN :"tc_pi".page_checksum(p0.page,
                       (CASE WHEN p0.blk > 2147483647 THEN p0.blk - 4294967296 ELSE p0.blk END)::int)::int & 65535
           END, :tc_blcksz, :tc_maxalign, :tc_le) AS state
  OFFSET 0) AS pg
WHERE r.kind IN ('table', 'toast');

-- 8. A second checkpoint before the indexes are read.  A concurrent VACUUM of
-- the TOAST table removes index entries before it frees heap line pointers,
-- so anything freed before step 7 read it is gone from disk after this.
CHECKPOINT /* wiki_tid_check */;

-- 9. Read every index page; collect the heap TID of every live leaf item.
CREATE /* wiki_tid_check */ TEMP TABLE tc_item (indexrelid oid, heaprel oid, iblk bigint, state text,
                           btflags int, ioff int, hblk bigint, hoff int);
INSERT /* wiki_tid_check */ INTO tc_item
SELECT pg.relid, r.heaprel, pg.blk, pg.state, pg.btflags, it.ioff, it.hblk, it.hoff
FROM tc_rel AS r
JOIN tc_seg AS s ON s.relid = r.relid AND s.read_blocks > 0
CROSS JOIN LATERAL generate_series(0, s.read_blocks - 1, :chunk_blocks) AS c(first)
CROSS JOIN LATERAL (
  SELECT pg_read_binary_file(s.path, c.first * :tc_blcksz,
                             least(:chunk_blocks, s.read_blocks - c.first) * :tc_blcksz, true) AS buf,
         least(:chunk_blocks, s.read_blocks - c.first) AS n
  OFFSET 0) AS rb
CROSS JOIN LATERAL generate_series(0, rb.n - 1) AS k(i)
CROSS JOIN LATERAL (
  SELECT r.relid, s.segno * :tc_relseg + c.first + k.i AS blk,
         substring(rb.buf FROM (k.i * :tc_blcksz + 1)::int FOR :tc_blcksz) AS page
  OFFSET 0) AS p0
CROSS JOIN LATERAL (
  SELECT p0.relid, p0.blk, p0.page, st.state,
         CASE WHEN st.state = 'ok' THEN pg_temp.tc_u16(p0.page, :tc_blcksz - 4, :tc_le) END AS btflags,
         CASE WHEN st.state = 'ok' THEN pg_temp.tc_u32(p0.page, :tc_blcksz - 12, :tc_le) END AS btpo_next
  FROM (SELECT pg_temp.tc_state(p0.page, p0.blk, 'b',
                 CASE WHEN :tc_checksums AND length(p0.page) = :tc_blcksz
                      THEN :"tc_pi".page_checksum(p0.page,
                             (CASE WHEN p0.blk > 2147483647 THEN p0.blk - 4294967296 ELSE p0.blk END)::int)::int & 65535
                 END, :tc_blcksz, :tc_maxalign, :tc_le) AS state) AS st
  OFFSET 0) AS pg
LEFT JOIN LATERAL (
  -- Live leaf pages only (BTP_LEAF set; BTP_DELETED, BTP_META, BTP_HALF_DEAD clear).
  -- bt_page_items() is STRICT, so a NULL page is never decoded.  Item 1 of a
  -- page with a right sibling is its high key, which points at no heap tuple.
  SELECT b.itemoffset::int AS ioff,
         (b.ctid::text::point)[0]::bigint AS hblk,
         (b.ctid::text::point)[1]::int AS hoff
  FROM :"tc_pi".bt_page_items(
         CASE WHEN pg.state = 'ok' AND pg.blk > 0 AND pg.btflags & 1 = 1 AND pg.btflags & 28 = 0
              THEN pg.page END) AS b
  WHERE NOT (b.itemoffset = 1 AND pg.btpo_next <> 0)
) AS it ON true
WHERE r.kind IN ('table_index', 'toast_index') AND r.skip IS NULL;

ANALYZE /* wiki_tid_check */ tc_heap_page;
ANALYZE /* wiki_tid_check */ tc_item;

-- 10. Re-read each page that failed validation once.  A page that a
-- concurrent write tore can fail once and pass the second time.
CREATE /* wiki_tid_check */ TEMP TABLE tc_recheck AS
WITH /* wiki_tid_check */ bad AS (
  SELECT relid, blk, state, 'h' AS kind FROM tc_heap_page WHERE state NOT IN ('ok', 'all_zero')
  UNION ALL
  SELECT DISTINCT indexrelid, iblk, state, 'b' FROM tc_item WHERE state NOT IN ('ok', 'all_zero')
)
SELECT b.relid, b.blk, b.kind, b.state AS first_state,
       pg_temp.tc_state(rr.page, b.blk, b.kind,
         CASE WHEN :tc_checksums AND length(rr.page) = :tc_blcksz
              THEN :"tc_pi".page_checksum(rr.page,
                     (CASE WHEN b.blk > 2147483647 THEN b.blk - 4294967296 ELSE b.blk END)::int)::int & 65535
         END, :tc_blcksz, :tc_maxalign, :tc_le) AS second_state
FROM bad AS b
JOIN tc_seg AS s ON s.relid = b.relid AND s.segno = b.blk / :tc_relseg
CROSS JOIN LATERAL (
  SELECT pg_read_binary_file(s.path, (b.blk % :tc_relseg) * :tc_blcksz, :tc_blcksz, true) AS page
  OFFSET 0) AS rr;

-- 11. Report.
\echo
\echo '== relations and segment files'
SELECT /* wiki_tid_check */ r.relid::regclass AS relation, r.kind, r.path,
       coalesce(r.skip, count(s.segno) || ' segment file(s), ' || coalesce(sum(s.read_blocks), 0)
                || ' reachable blocks') AS status,
       nullif(string_agg(nullif(s.layout, ''), '; ' ORDER BY s.segno), '') AS layout_problems
FROM tc_rel AS r
LEFT JOIN tc_seg AS s ON s.relid = r.relid
GROUP BY r.relid, r.kind, r.path, r.skip
ORDER BY r.kind, 1;

\echo '== pages read from disk, by state'
SELECT /* wiki_tid_check */ x.relid::regclass AS relation, x.state, count(*) AS pages,
       count(rc.relid) FILTER (WHERE rc.second_state = rc.first_state) AS confirmed_on_reread
FROM (SELECT relid, blk, state FROM tc_heap_page
      UNION ALL
      SELECT DISTINCT indexrelid, iblk, state FROM tc_item) AS x
LEFT JOIN tc_recheck AS rc ON rc.relid = x.relid AND rc.blk = x.blk
GROUP BY 1, 2
ORDER BY 1, 2;

\echo '== heap TIDs of live B-tree leaf items, by verdict'
SELECT /* wiki_tid_check */ i.indexrelid::regclass AS index,
       pg_temp.tc_verdict(i.hoff, h.state, h.codes) AS verdict, count(*) AS tids
FROM tc_item AS i
LEFT JOIN tc_heap_page AS h ON h.relid = i.heaprel AND h.blk = i.hblk
WHERE i.ioff IS NOT NULL
GROUP BY 1, 2
ORDER BY 1, 2;

\echo '== first findings (index page and item -> heap TID)'
SELECT /* wiki_tid_check */ i.indexrelid::regclass AS index, i.iblk AS index_block,
       i.ioff AS index_item, format('(%s,%s)', i.hblk, i.hoff)::tid AS heap_tid,
       pg_temp.tc_verdict(i.hoff, h.state, h.codes) AS verdict
FROM tc_item AS i
LEFT JOIN tc_heap_page AS h ON h.relid = i.heaprel AND h.blk = i.hblk
WHERE i.ioff IS NOT NULL AND pg_temp.tc_verdict(i.hoff, h.state, h.codes) <> 'ok'
ORDER BY 1, 2, 3
LIMIT :max_findings;

\echo '== table locks this check holds until COMMIT'
SELECT /* wiki_tid_check */ l.relation::regclass AS relation, l.mode, l.granted
FROM pg_locks AS l
WHERE l.pid = pg_backend_pid() AND l.locktype = 'relation'
  AND l.relation IN (SELECT relid FROM tc_rel)
ORDER BY 1, 2;

COMMIT /* wiki_tid_check */;
SELECT /* wiki_tid_check */ 'DEALLOCATE /* wiki_tid_check */ tc_toast_lock'
WHERE EXISTS (SELECT 1 FROM pg_prepared_statements WHERE name = 'tc_toast_lock') \gexec

-- 12. Verdict, after commit, with a fresh look at the catalog: if any file
-- path changed while the check ran, the result describes files that are gone.
\echo '== verdict'
SELECT /* wiki_tid_check */
       CASE WHEN f.paths_changed > 0 THEN 'INCONCLUSIVE: a relation was rewritten during the check; run it again'
            WHEN f.confirmed > 0 THEN 'FAIL: ' || f.confirmed || ' confirmed problem(s)'
            WHEN f.unconfirmed > 0 THEN 'INCONCLUSIVE: ' || f.unconfirmed
                 || ' page(s) failed validation once but not on re-read; run it again'
            WHEN f.tids = 0 THEN 'NOTHING CHECKED: no live B-tree leaf item was found'
            ELSE 'PASS: ' || f.tids || ' heap TIDs checked, none dangling' END AS verdict
FROM (
  SELECT (SELECT count(*) FROM tc_rel AS r
          WHERE r.path IS DISTINCT FROM pg_relation_filepath(r.relid)) AS paths_changed,
         (SELECT count(*) FROM tc_item WHERE ioff IS NOT NULL) AS tids,
         (SELECT count(*) FROM tc_seg WHERE layout <> '')
         + (SELECT count(*) FROM tc_recheck WHERE second_state = first_state)
         + (SELECT count(*) FROM tc_item AS i
            LEFT JOIN tc_heap_page AS h ON h.relid = i.heaprel AND h.blk = i.hblk
            LEFT JOIN tc_recheck AS rc ON rc.relid = i.heaprel AND rc.blk = i.hblk
            WHERE i.ioff IS NOT NULL
              AND pg_temp.tc_verdict(i.hoff, h.state, h.codes) <> 'ok'
              AND (rc.relid IS NULL OR rc.second_state = rc.first_state)) AS confirmed,
         (SELECT count(*) FROM tc_recheck WHERE second_state <> first_state) AS unconfirmed
) AS f;
```

### Reading the report

The report has six parts: relations and segment files, pages by state, TIDs by verdict, the first findings, the locks held, and the verdict. The page states are:

| Page state | Meaning | Counts as a problem? |
|---|---|---|
| `ok` | Header sane, checksum matches when checksums are on, items in bounds | No |
| `all_zero` | Every byte is zero | No by itself: bulk heap extension and a crash after index extension both leave such pages ([All-zero page](../../../glossary.md#all-zero-page)). Yes for every TID that names one. |
| `short_read` | Fewer than `BLCKSZ` bytes came back | Yes, if a re-read repeats it |
| `bad_checksum` | `pd_checksum` differs from the recomputed checksum | Yes, if a re-read repeats it |
| `bad_header` | The header fails the validation above | Yes, if a re-read repeats it |
| `bad_meta` | B-tree block 0 lacks `BTP_META` | Yes, if a re-read repeats it |
| `bad_items` | A B-tree line pointer or tuple size is out of bounds | Yes, if a re-read repeats it; that page's TIDs were not read |

The verdict line is one of `PASS`, `FAIL`, `INCONCLUSIVE` or `NOTHING CHECKED`, decided in the order shown in the [logic map](#logic-map). A `FAIL` counts every dangling TID, every repeated bad page and every segment layout problem.

A finding names the index block and item, the heap TID, and the verdict from [When an index TID is valid](#when-an-index-tid-is-valid). To see which rows are affected, look at that index item's key bytes with `bt_page_items(get_raw_page(index, block))`. That lookup reads through shared buffers, so it shows the cached version of the page.

### Measured results

The [measurement script](#measurement-script) built 12.2 from the pin, with 1 MB segments so that small tables span several segment files. It ran one suite with data checksums off, the default, and one with them on. Each suite created the fixtures, damaged them, and ran the check exactly as extracted from this page. Unless the table says otherwise, the damage was done with `dd`, `cp` or a shell redirection while the server was stopped.

| Fixture | Damage | Checksums off | Checksums on |
|---|---|---|---|
| `t_ok`: 20,000 rows after updates, deletes, VACUUM and on-access pruning | None | `PASS`: 38,854 TIDs | `PASS`: 38,854 TIDs |
| `t_big`: 1,000,000 rows, 34,483 heap blocks | None | `PASS`: 2,000,000 TIDs | `PASS`: 2,000,000 TIDs |
| `t_cached`: 4,000 rows | Block 5 zeroed **while the server ran**, its page clean in shared buffers | `FAIL`: 29 `heap_page_all_zero` in each index | Same |
| `t_trunc`: 4,000 rows, two segments | Last segment emptied | `FAIL`: 288 `heap_block_missing` in each index | Same |
| `t_midseg`: 12,000 rows, four segments | Segment 1 cut to 64 of 128 blocks | `FAIL`: "data after a short segment" for segments 2 and 3, 192 reachable blocks, 6,432 `heap_block_missing` in the primary key | Same |
| `t_stale_heap` | Last heap block put back to its image from before 2 rows were added to it (a lost heap write) | `FAIL`: 2 `offset_beyond_line_pointers` in each index | Same |
| `t_stale_index` | Primary key file put back to its image from before 400 rows were deleted and vacuumed (a lost index write) | `FAIL`: 400 `line_pointer_unused` in the primary key, none in the other index | Same |
| `t_hot` | Primary key put back to its image from before row (0,5) was deleted, vacuumed, and its slot reused by a HOT update | `FAIL`: 1 `points_to_heap_only_tuple` | Same |
| `t_lpcorrupt` | Line pointer (3,2) overwritten: `LP_NORMAL`, offset 8000, length 400 | `FAIL`: 1 `line_pointer_corrupt` in each index | `FAIL`: block 3 `bad_checksum`, confirmed on re-read; 29 `heap_page_bad_checksum` in the primary key |
| `t_flip` | One byte of tuple data in block 3 overwritten | **`PASS`: not detected** | `FAIL`: 29 `heap_page_bad_checksum` in the primary key |
| `t_badidx` | Primary key leaf 1: `pd_lower` = 65535; leaf 2: item 2's size field = 7,936 | `FAIL`: one `bad_header` and one `bad_items` page, both confirmed | `FAIL`: two `bad_checksum` pages, both confirmed |
| `t_toast` | TOAST heap block 2 zeroed | `FAIL`: 4 `heap_page_all_zero` in the TOAST index; all 4,000 primary-key TIDs `ok` | Same |

What the runs show beyond the table:

- **The check reads around shared buffers.** In `t_cached`, before the check ran, block 5 was a clean page in shared buffers. After the zeroing, `count(pad)`, which must read every heap tuple, still returned 4,000 of 4,000 rows. An index scan for `id = 146`, a row on block 5, still returned it. The check, run at that moment, reported all 29 rows of block 5 in both indexes, and block 5 was still a clean shared buffer afterwards. After a restart, `count(pad)` returned 3,971 and the index scan returned no row, without any error. The loss became visible to queries only after the cached page left memory. The silence matches the server's read check, which accepts an all-zero page [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L123-L142), and the heap fetch, which finds no line pointer on it [heapam.c#heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1556).
- **The check adds nothing to shared buffers.** After a restart, the check of `t_ok` left 0 buffers of `t_ok`, its two indexes, its TOAST table and the TOAST index in shared buffers. A plain `SELECT count(*) FROM t_ok` afterwards occupied 901.
- **TOAST damage is invisible to the table's own indexes.** In `t_toast`, every primary-key TID was valid, yet reading the values failed with `missing chunk number 0 for toast value … in pg_toast_…` [tuptoaster.c:2013](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2013). Only the TOAST-index pass found it. With `include_toast=off`, the same table passed.
- **The options and refusals work.** With `index_name` set to the primary key, `t_stale_index` reported the same 400 findings and read no other index. An unlogged table, a non-superuser, a database without pageinspect, and an index of a different table were each refused with psql exit status 3 before the lock was taken.

Cost: for `t_big`, the check's statements took 10,340 ms in total with checksums off and 10,475 ms with them on, for 34,483 heap blocks, 12,093 index blocks and 2,000,000 index entries. The heap pass took about 32 µs per heap page and the index pass about 2.7 µs per index entry; the report queries, which join all 2,000,000 TIDs again, took most of the rest. The [Last run](#last-run) record gives each step. The files had just been written, so they were probably served from the OS page cache; the run does not separate CPU time from I/O. Cold reads from storage add the time to read the table, its TOAST table and the checked indexes once.

### Branches, limits and false positives

| Condition or state | Behavior | Consequence |
|---|---|---|
| Data checksums off: `initdb` leaves them off unless `-k` is given [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) | A changed byte in tuple data or in an index key passes validation | Damage that keeps every line pointer valid is not detected; measured with `t_flip` |
| A heap row has lost its index entries | The check reads index entries only | Not detected; this direction needs a heap scan that probes the index, which is what the excluded amcheck option does ([amcheck](../../../glossary.md#amcheck)) |
| A lost heap write that kept every indexed line pointer in use, for example a lost HOT update | The stale page still has the same roots | Not detected; the rows hold stale contents |
| The index is not a B-tree | Listed as skipped | Not checked |
| `indisready` or `indislive` is false | Listed as skipped | Not checked, because VACUUM does not process it and its entries may legitimately dangle |
| Unlogged, temporary, partitioned or materialized-view relation | Refused before any lock | Check each partition separately; unlogged and temporary pages cannot be made current on disk by `CHECKPOINT` |
| Server in recovery | Refused | Run it on the primary |
| All-zero heap page that no TID names | Reported as a page state, not a failure | Normal after bulk extension |
| Page fails validation once and passes on re-read | `INCONCLUSIVE` | A concurrent write tore the first read; run the check again |
| A file path changes during the check | `INCONCLUSIVE` | A rewrite, such as `REINDEX` of the TOAST index, replaced the file being read |
| Data in a segment after a short segment | Layout problem; TIDs into those blocks are `heap_block_missing` | A backend cannot reach those blocks either; measured with `t_midseg` |
| A page is zeroed on disk while a clean copy is cached | `FAIL` | Measured with `t_cached`; normal queries still return the rows until the page leaves memory |
| A page is zeroed and the [visibility map](../../../glossary.md#visibility-map) still marks it all-visible | [Index-only scans](../../../glossary.md#index-only-scan) return its rows without visiting the heap [nodeIndexonlyscan.c#IndexOnlyNext](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L169) | A query that counts through such a scan can hide the loss; this check is not affected |
| The storage loses a write that the kernel page cache still holds | Both the server and the check read the cached data | Not detected while the kernel keeps the page; see [Open Questions](#open-questions) |
| The lock removed from the script | Inserts can add index entries whose heap tuples the heap pass did not read | False findings; a lock-free variant would need a confirming re-read of each finding, which this script does not implement |
| Very large table | One `statement_timeout` covers each pass | Raise `statement_wait`; the temporary tables hold one row per heap page and one per index entry |

### Final causal summary

Shared buffers can hold newer page versions than the files → the script blocks writers with `SHARE ROW EXCLUSIVE` and checkpoints, so every change made before the lock is in the files → it reads every reachable heap page with `pg_read_binary_file()`, validates it, and records each line pointer's state → a second checkpoint makes any concurrent TOAST VACUUM's index deletions durable before the index is read → it reads every index page, validates it, and lets `bt_page_items(bytea)` extract the heap TID of every live leaf item except the high key → a TID is valid only if it names an existing, initialized heap page and a line pointer that is normal and not heap-only, a redirect, or dead → anything else is a dangling TID, which means data loss or corruption on disk → a page that fails validation is re-read once, and a changed file path voids the result. No page of the checked relations passes through shared buffers, but every read still passes through the kernel's page cache.

## Measurement Script

### Usage

| Item | Value |
|---|---|
| Purpose | Builds the pin as 12.2 with 1 MB segments, creates two disposable clusters (data checksums off, then on), damages fixture tables in ten ways, and runs the check published on this page, extracted from the page itself, against each one. Backs every number under [Measured results](#measured-results), the line-pointer counts under [When an index TID is valid](#when-an-index-tid-is-valid), and the self-test. |
| Invocation | From the repository root, extract the Bash fence below to `.wiki-runtime/tmp/tc12-run.sh`, then run `bash .wiki-runtime/tmp/tc12-run.sh [stage ...]`. The script checks the pinned checkout, clears inherited `PG*` connection settings, uses its own socket directory and port, and checks the data directory of the server it reaches before creating fixtures. |
| Stages | Default order: `build extract suite_off suite_on report`. `stop` and `clean` tear down and are not in the default order. |
| Environment | `PORT12` (default `55446`) is the port label on the private socket; `JOBS` (default `8`) sets parallel build jobs. The checkout, sandbox path and `RELSEG_SIZE` are fixed in the script. |
| Prerequisites | A C toolchain, `make`, Perl, bison and flex, plus readline and zlib headers, because `build` configures with `--with-readline --with-zlib`; the pinned checkout at `raw/postgres-12/`; a little-endian machine, because the fixtures write little-endian bytes; locale `C`, encoding `UTF8`; about 1.5 GB of free disk for the build, the installation and the two clusters. PostgreSQL's own build runs Perl for code generation; the script itself uses only Bash, SQL and POSIX tools. |
| Output | Files under `.wiki-runtime/tmp/tc12/out/off/` and `.wiki-runtime/tmp/tc12/out/on/`. Read `platform.txt` first, then `results.txt`, then the full check reports such as `cached_online.txt`, `midseg.txt` and `big.txt`. |
| Runtime | On the test host (`Darwin 27.0.0 arm64`, `JOBS=8`): about 2.5 minutes from an empty sandbox (136 s on the last run), most of it the build; about 50 seconds with the build still present (48 s), about 24 s per suite. |
| Cleanup | `bash .wiki-runtime/tmp/tc12-run.sh stop clean` stops any owned cluster, checks that no PID file, sandbox server process or socket remains, and deletes `.wiki-runtime/tmp/tc12/`, including its build. Remove the extracted `tc12-run.sh` afterwards. |

To extract the published script without editing it, run this from the repository root:

```sh
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/indexing/index-heap-tid-check-on-disk.md | sed '1d;$d' > .wiki-runtime/tmp/tc12-run.sh
bash -n .wiki-runtime/tmp/tc12-run.sh
bash .wiki-runtime/tmp/tc12-run.sh
bash .wiki-runtime/tmp/tc12-run.sh stop clean
```

### Last run

On 2026-10-22 the published script was extracted with the command above and run on `Darwin 27.0.0 arm64` against PostgreSQL 12.2 at pin `45b88269a353ad93744772791feb6d01bc7e1e42`: first from an empty sandbox (136 s, build included), then again with the build present (48 s). Every assertion passed both times, and both runs gave the same results apart from timings. The platform facts both suites recorded in `platform.txt`:

- [`block_size`](../../../glossary.md#blcksz) 8192 and `segment_size` 1 MB, which is `blocks_per_segment` 128 from the patched `RELSEG_SIZE`; [maximum data alignment](../../../glossary.md#alignment) 8; little-endian;
- `shared_buffers` 128 MB, the value `initdb` wrote; `autovacuum` off;
- data checksums off in `suite_off` (`data_page_checksum_version` 0) and on in `suite_on` (1).

The `t_big` check, step by step, from `big.txt` of the first run with checksums off: the heap pass took 1,100 ms for 34,483 pages; the index pass took 5,467 ms for 12,093 index pages and 2,000,000 entries; the three report queries and the verdict took 342, 819, 572 and 1,663 ms; the two `ANALYZE` statements took 12 and 152 ms and building the re-read table took 200 ms; each `CHECKPOINT` took under 4 ms, because the script had just checkpointed. With checksums on, the heap and index passes took 1,131 and 5,444 ms. The whole check took 10,340 ms with checksums off and 10,475 ms with them on; the second run took 10,324 and 10,521 ms.

`stop clean` then stopped nothing, because each suite had already stopped its server, found no PID file, sandbox process or socket, and deleted the sandbox.

### Stages

| Stage | What it does | Needs |
|---|---|---|
| `build` | Configures 12.2 out of tree under `$SANDBOX/build` with `--enable-debug --with-readline --with-zlib`, patches the generated `pg_config.h` to `RELSEG_SIZE 128`, builds and installs the server, `pageinspect` and `pg_buffercache`. Skipped when a matching build exists. | nothing |
| `extract` | Copies this page's one `sql` fence to `$SANDBOX/sql/tid_check.sql` and checks its tag | `build` |
| `suite_off` | Recreates `data_off` with `initdb` (checksums off), starts it, builds the fixtures, runs every check and assertion in the table above plus the option and refusal tests, and stops the server | `extract` |
| `suite_on` | The same with `initdb --data-checksums` in `data_on`, without the option and refusal tests | `extract` |
| `report` | Prints `platform.txt` and `results.txt` of both suites | any |
| `stop` | Stops an owned cluster and checks that no PID file, process started from the sandbox's binaries or carrying the `postgres: tc12:` title, or socket remains | none |
| `clean` | Runs `stop`, then deletes `$SANDBOX`, refusing a sandbox it does not own | none |

Each suite starts and stops its own server in the same shell, and an exit trap stops it if a step fails. A suite recreates its cluster from scratch, so it can be re-run alone. Every fixture object is disposable.

### Non-default settings in the measurement

| Setting or state | Default at the pin | Value in the script, and why |
|---|---|---|
| `RELSEG_SIZE` | 131,072 blocks (1 GB at 8 kB) [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864) | 128 blocks (1 MB), patched only in the out-of-tree generated header, because `configure --with-segsize` takes whole gigabytes. It makes `t_trunc`, `t_midseg` and the TOAST tables span several segment files. The check reads the segment size from `pg_control_init()`, so its SQL is the same as at the default; with the default, those fixtures would fit in one segment and the segment branches would not be exercised. |
| Data checksums | Off: `initdb` sets them only with `-k` / `--data-checksums` [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [initdb.c:3054](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3054) | Off in `suite_off`, on in `suite_on`. The setting cannot change after `initdb`; `data_checksums` is `PGC_INTERNAL` [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1827-L1835). The `t_flip`, `t_lpcorrupt` and `t_badidx` results depend on it, as the table shows; every other result was the same in both suites. |
| `autovacuum` | `on` [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433) | `off` (`PGC_SIGHUP`, written to `postgresql.conf` before the first start), so that no worker vacuums a fixture between its damage and its check, which would change the findings. With the default, a worker could also make `LOCK TABLE` wait. |
| `statement_timeout`, `lock_timeout` | 0, disabled [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2387) [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2389-L2398) | 10 min and 30 s for every fixture session through `PGOPTIONS` (`PGC_USERSET`, session scope), to bound runaway fixture statements. The check sets its own values with `SET`. No result depends on them. |
| `enable_seqscan`, `enable_bitmapscan` | `on` [guc.c#enable_seqscan](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L884-L892) [guc.c#enable_bitmapscan](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L914-L922) | `off` with `SET` in the one psql session that looks up `id = 146` in `t_cached` (`PGC_USERSET`), so that the lookup is an index scan that fetches the heap tuple. The "by index" counts depend on it; with the defaults the planner may choose a sequential or bitmap scan instead. |
| `cluster_name`, `listen_addresses`, `unix_socket_directories`, `port` | `''` and `localhost` for the first two [guc.c#cluster_name](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4170-L4179) [guc.c#listen_addresses](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3951-L3957); the build's defaults for the other two [guc.c#unix_socket_directories](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936) [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177) | `tc12`, `''`, the sandbox socket directory and 55446 (all `PGC_POSTMASTER`, written before the first start), to isolate the cluster and tag its processes. No result depends on them. |
| Fixture state | — | Every table gets a primary key, an index on `k` and, on every 40th row, a 6,400-byte value of md5 hex digits, which is stored in the TOAST table. `t_ok` is updated, deleted from, vacuumed, updated, deleted from and scanned, so that it holds redirect, dead, heap-only and unused line pointers. Each damaged table is described in the results table. |

### The script

```bash
#!/usr/bin/env bash
#
# tc12-run.sh - the measurement script for the wiki page
# wiki/v12/questions/indexing/index-heap-tid-check-on-disk.md
#
# It builds the pinned PostgreSQL 12 checkout out of tree with a shortened
# RELSEG_SIZE, creates two disposable clusters (data checksums off, the
# default, and on), damages disposable fixture tables in ten ways, and runs
# the check published on the page against each table.  The check's SQL is
# extracted from the page itself, so the measured SQL is the filed SQL.
#
# NON-DEFAULT build knob: RELSEG_SIZE is patched from the pin's default 131072
# blocks (1 GB) to 128 blocks (1 MB) in the generated pg_config.h of the
# out-of-tree build, because configure's --with-segsize accepts only whole
# gigabytes in PostgreSQL 12.  The check reads the segment size from the
# control file (pg_control_init), so it adapts; the patch only makes the
# multi-segment fixtures small.
#
# Every fixture object is disposable.  The script creates and drops its own
# clusters and its database tc; it never connects to any other cluster.
# Damage is done with dd and printf on the sandbox's own data files, while
# the server is stopped, except for the t_cached leg, which zeroes one page
# while the server runs and holds a clean copy of that page in shared buffers.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/tc12-run.sh                 # default stages
#   bash .wiki-runtime/tmp/tc12-run.sh suite_on        # one suite again
#   bash .wiki-runtime/tmp/tc12-run.sh stop clean      # tear down
#
# Stages: build extract suite_off suite_on report stop clean
# Environment: PORT12 (default 55446), JOBS (default 8)
set -uo pipefail

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
PAGE="$WIKI_ROOT/wiki/v12/questions/indexing/index-heap-tid-check-on-disk.md"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/tc12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55446}"
JOBS="${JOBS:-8}"
RELSEG_BLOCKS=128
SPEC="$PIN12 $RELSEG_BLOCKS pageinspect pg_buffercache"

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; BIN="$INST/bin"
SOCK="$SANDBOX/sock"; SQLD="$SANDBOX/sql"; OUTROOT="$SANDBOX/out"
DATA=""; OUT=""

# Never inherit a connection target from the caller's environment.
unset PGDATABASE PGUSER PGSERVICE PGSERVICEFILE PGHOSTADDR PGPASSFILE PGSSLMODE
export PGHOST="$SOCK" PGPORT="$PORT12"
# statement_timeout and lock_timeout are user-settable (PGC_USERSET); through
# PGOPTIONS they bound every fixture statement of every session.  The
# published check sets its own values with SET.
export PGOPTIONS="-c statement_timeout=10min -c lock_timeout=30s"

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# A failing stage must not leave its cluster running.
on_exit() {
  local rc=$?
  trap - EXIT
  if [ -n "$DATA" ] && [ -f "$SANDBOX/.started" ] && [ -f "$SANDBOX/.tc12-owned" ]; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || true
    rm -f "$SANDBOX/.started"
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
  [ -f "$SANDBOX/.tc12-owned" ] || die "sandbox is not owned by this script: $SANDBOX"
  [ "$(cat "$SANDBOX/.tc12-owned")" = "$PIN12 $WIKI_ROOT" ] || die "sandbox owner/pin mismatch"
  export TMPDIR="$SANDBOX/tmp"
  mkdir -p "$TMPDIR"
}
source_pin() {
  [ "$(git --no-optional-locks -C "$SRC12" rev-parse HEAD)" = "$PIN12" ] \
    || die "PostgreSQL 12 checkout is not at $PIN12"
  [ -z "$(git --no-optional-locks -C "$SRC12" status --porcelain --untracked-files=no)" ] \
    || die "PostgreSQL 12 checkout has tracked changes"
}

# -X ignores ~/.psqlrc; every psql call sets ON_ERROR_STOP.
# tag_sql adds the wiki tag after each leading verb that lacks one.
tag_sql() {
  case "$1" in
    *wiki_tc12_fixture*) printf '%s\n' "$1" ;;
    *) printf '%s\n' "$1" | sed -E 's/(^|;[[:space:]]*)(SELECT|INSERT|UPDATE|DELETE|CREATE|DROP|VACUUM|CHECKPOINT|SET)([[:space:]])/\1\2 \/\* wiki_tc12_fixture \*\/\3/g' ;;
  esac
}
q() { # q <sql>: run fixture SQL in database tc, discard output
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d tc -c "$(tag_sql "$1")" > /dev/null || die "failed: $1"
}
s() { # s <sql>: one unaligned value from database tc
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d tc -c "$(tag_sql "$1")" || die "failed: $1"
}
check() { # check <table> <out-name>: the published check, unaligned output
  "$BIN/psql" -X -q -A -F '|' -v ON_ERROR_STOP=1 -d tc -v table_name="public.$1" \
      -f "$SQLD/tid_check.sql" > "$OUT/$2.txt" 2> "$OUT/$2.err" \
    || { cat "$OUT/$2.err" >&2; die "the check failed to run on $1"; }
  note "$2: $(grep -E '^(PASS|FAIL|INCONCLUSIVE|NOTHING)' "$OUT/$2.txt")"
}
check_opt() { # check_opt <table> <out-name> <psql -v options...>: the check with options
  local t=$1 name=$2
  shift 2
  "$BIN/psql" -X -q -A -F '|' -v ON_ERROR_STOP=1 -d tc -v table_name="public.$t" "$@" \
      -f "$SQLD/tid_check.sql" > "$OUT/$name.txt" 2> "$OUT/$name.err" \
    || { cat "$OUT/$name.err" >&2; die "the check failed to run on $t"; }
  note "$name: $(grep -E '^(PASS|FAIL|INCONCLUSIVE|NOTHING)' "$OUT/$name.txt")"
}
expect_refusal() { # expect_refusal <out-name> <message> <psql options...>: exit 3 with message
  local name=$1 msg=$2 rc
  shift 2
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 "$@" -f "$SQLD/tid_check.sql" \
      > "$OUT/$name.txt" 2> "$OUT/$name.err"
  rc=$?
  [ "$rc" = 3 ] || die "$name: psql exit status $rc, expected 3"
  grep -F -q -- "$msg" "$OUT/$name.err" || die "$name: no \"$msg\" in the error"
  grep -q 'LOCK' "$OUT/$name.txt" && die "$name: the check went past its preconditions"
  record "$name: refused with exit 3: $(grep -F -- "$msg" "$OUT/$name.err" | head -n 1 | sed 's/^.*ERROR: *//')"
}
verdict_is() { # verdict_is <out-name> <PASS|FAIL|...>
  grep -E -q "^$2" "$OUT/$1.txt" || die "$1: verdict is not $2"
  printf '%s verdict %s\n' "$1" "$2" >> "$OUT/results.txt"
}
tids() { # tids <out-name> <index> <verdict>: TID count in the verdict table, 0 if absent
  local n
  n=$(grep -E "^$2\|$3\|[0-9]+\$" "$OUT/$1.txt" | cut -d'|' -f3)
  printf '%s\n' "${n:-0}"
}
expect_tids() { # expect_tids <out-name> <index> <verdict> <count>
  local got
  got=$(tids "$1" "$2" "$3")
  [ "$got" = "$4" ] || die "$1: $2 $3 = $got, expected $4"
  printf '%s %s %s = %s\n' "$1" "$2" "$3" "$got" >> "$OUT/results.txt"
}
expect_page() { # expect_page <out-name> <relation> <state>: a page-state row exists
  grep -E -q "^$2\|$3\|[0-9]+\|[0-9]+\$" "$OUT/$1.txt" || die "$1: no $3 page row for $2"
  printf '%s %s page state %s: %s\n' "$1" "$2" "$3" \
    "$(grep -E "^$2\|$3\|" "$OUT/$1.txt" | cut -d'|' -f3-4 | tr '|' ' ')" >> "$OUT/results.txt"
}
record() { printf '%s\n' "$*" >> "$OUT/results.txt"; note "$*"; }

relfile() { # relfile <relation> <segno>: absolute path of one main-fork segment
  local rel
  rel=$(s "SELECT pg_relation_filepath('$1')")
  [ -n "$rel" ] || die "no file path for $1"
  if [ "$2" = 0 ]; then printf '%s/%s\n' "$DATA" "$rel"; else printf '%s/%s.%s\n' "$DATA" "$rel" "$2"; fi
}
blockfile() { relfile "$1" $(($2 / RELSEG_BLOCKS)); }  # file holding block $2
blockseek() { printf '%s\n' $(($1 % RELSEG_BLOCKS)); }   # block offset in that file

server_start() {
  printf '%s\n' "$DATA" > "$SANDBOX/.started"
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w -t 60 start > /dev/null 2>&1 \
    || die "pg_ctl start failed, see $OUT/server.log"
}
server_stop() {
  "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  rm -f "$SANDBOX/.started"
}

# --------------------------------------------------------------- build -------
stage_build() {
  say "build 12.2 out of tree from $SRC12, RELSEG_SIZE=$RELSEG_BLOCKS"
  source_pin
  if [ -e "$SANDBOX" ]; then
    own_sandbox
  else
    mkdir -p "$SANDBOX" || die "could not create sandbox"
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.tc12-owned"
    own_sandbox
  fi
  mkdir -p "$BUILD" "$OUTROOT"
  if [ -x "$BIN/postgres" ]; then
    [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$SPEC" ] \
      || die "existing build has a different specification"
    note "matching build already present"
    return 0
  fi
  [ -x "$SRC12/configure" ] || die "no pinned checkout at $SRC12"
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" --enable-debug \
      --with-readline --with-zlib CFLAGS="-O2 -g" > configure.log 2>&1 ) \
    || die "configure failed, see $BUILD/configure.log"
  # The non-default RELSEG_SIZE described in the header.
  sed "s/^#define RELSEG_SIZE .*/#define RELSEG_SIZE $RELSEG_BLOCKS/" \
      "$BUILD/src/include/pg_config.h" > "$BUILD/src/include/pg_config.h.new" \
    || die "could not patch RELSEG_SIZE"
  mv "$BUILD/src/include/pg_config.h.new" "$BUILD/src/include/pg_config.h"
  [ "$(sed -n 's/^#define RELSEG_SIZE[[:space:]]*//p' "$BUILD/src/include/pg_config.h")" = "$RELSEG_BLOCKS" ] \
    || die "RELSEG_SIZE patch did not take"
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 \
      && make -C contrib/pageinspect install >> install.log 2>&1 \
      && make -C contrib/pg_buffercache install >> install.log 2>&1 ) \
    || { grep 'error:' "$BUILD/make.log" | head -n 3 >&2; die "make failed, see $BUILD/make.log"; }
  printf '%s\n' "$SPEC" > "$SANDBOX/.build-spec"
  note "$("$BIN/postgres" --version)"
}

# ------------------------------------------------------------- extract -------
# Copy the page's one sql fence to $SQLD/tid_check.sql.  The fence string is
# assembled at run time so this script contains no literal Markdown fence.
stage_extract() {
  say "extract the published check from the page"
  own_sandbox
  mkdir -p "$SQLD"
  local fence n=0 inblock=0 line
  fence=$(printf '\140\140\140')
  : > "$SQLD/tid_check.sql"
  while IFS= read -r line; do
    if [ "$inblock" -eq 0 ] && [ "$line" = "${fence}sql" ]; then
      n=$((n + 1)); inblock=1; continue
    fi
    if [ "$inblock" -eq 1 ] && [ "$line" = "$fence" ]; then inblock=0; continue; fi
    [ "$inblock" -eq 1 ] && printf '%s\n' "$line" >> "$SQLD/tid_check.sql"
  done < "$PAGE"
  [ "$n" -eq 1 ] || die "expected 1 sql block in the page, found $n"
  grep -q 'wiki_tid_check' "$SQLD/tid_check.sql" || die "the sql block is not the check"
  note "tid_check.sql: $(wc -l < "$SQLD/tid_check.sql" | tr -d ' ') lines"
}

# --------------------------------------------------------------- suite -------
# Cluster settings, written before the first start:
#   listen_addresses, unix_socket_directories, port, cluster_name  PGC_POSTMASTER
#   autovacuum                                                     PGC_SIGHUP
# autovacuum is off so that no worker vacuums a fixture between its damage
# and its check.  shared_buffers keeps the 128MB that initdb writes.
cluster_up() { # cluster_up <off|on>
  [ -x "$BIN/postgres" ] && [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$SPEC" ] \
    || die "run the build stage first"
  [ -s "$SQLD/tid_check.sql" ] || die "run the extract stage first"
  mkdir -p "$SOCK" "$OUT"
  [ -z "$(ls -A "$SOCK")" ] || die "socket directory is not free"
  "$BIN/pg_isready" -q -h "$SOCK" -p "$PORT12" && die "a server already answers on $SOCK"
  rm -rf "$DATA"
  if [ "$1" = on ]; then
    "$BIN/initdb" -D "$DATA" -A trust --locale=C --encoding=UTF8 --data-checksums \
      > "$OUT/initdb.log" 2>&1 || die "initdb failed"
  else
    "$BIN/initdb" -D "$DATA" -A trust --locale=C --encoding=UTF8 \
      > "$OUT/initdb.log" 2>&1 || die "initdb failed"
  fi
  {
    printf "listen_addresses = ''\n"
    printf "unix_socket_directories = '%s'\n" "$SOCK"
    printf "port = %s\n" "$PORT12"
    printf "cluster_name = 'tc12'\n"
    printf "autovacuum = off\n"
  } >> "$DATA/postgresql.conf"
  server_start
  [ "$("$BIN/psql" -X -At -d postgres -c 'SHOW data_directory')" = "$DATA" ] \
    || die "connected to the wrong server"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d postgres -c 'CREATE /* wiki_tc12_fixture */ DATABASE tc' \
    > /dev/null || die "createdb failed"
  q "CREATE /* wiki_tc12_fixture */ EXTENSION pageinspect"
  q "CREATE /* wiki_tc12_fixture */ EXTENSION pg_buffercache"
  {
    uname -srm
    "$BIN/postgres" --version
    s "SELECT 'block_size=' || current_setting('block_size') || ' segment_size=' || current_setting('segment_size')"
    s "SELECT 'max_data_alignment=' || max_data_alignment || ' blocks_per_segment=' || blocks_per_segment || ' data_page_checksum_version=' || data_page_checksum_version FROM pg_control_init()"
    s "SELECT 'data_checksums=' || current_setting('data_checksums') || ' shared_buffers=' || current_setting('shared_buffers') || ' autovacuum=' || current_setting('autovacuum')"
    s "SELECT 'little_endian=' || (get_byte(pg_read_binary_file('global/pg_control', 8, 1), 0) = (SELECT pg_control_version % 256 FROM pg_control_system()))"
  } > "$OUT/platform.txt"
  grep -q '^block_size=8192 ' "$OUT/platform.txt" || die "fixtures assume 8 kB blocks"
  grep -q "blocks_per_segment=$RELSEG_BLOCKS " "$OUT/platform.txt" || die "segment size mismatch"
  # The byte patches below write little-endian values.
  grep -q '^little_endian=true$' "$OUT/platform.txt" || die "fixtures assume a little-endian server"
}

mk() { # mk <table> <rows>: disposable table, B-tree PK and text index, TOAST values
  q "DROP /* wiki_tc12_fixture */ TABLE IF EXISTS $1"
  q "CREATE /* wiki_tc12_fixture */ TABLE $1 (id int PRIMARY KEY, k text NOT NULL, pad text NOT NULL, big text)"
  q "CREATE /* wiki_tc12_fixture */ INDEX $1_k ON $1 (k)"
  q "INSERT /* wiki_tc12_fixture */ INTO $1
     SELECT g, md5(g::text), repeat('p', 200),
            CASE WHEN g % 40 = 0 THEN (SELECT string_agg(md5(g::text || x::text), '')
                                       FROM generate_series(1, 200) AS x) END
     FROM generate_series(1, $2) AS g"
  q "VACUUM /* wiki_tc12_fixture */ $1"
}
on_block() { # on_block <relation> <block>: live tuples on one heap block
  s "SELECT count(*) FROM $1 WHERE ctid >= '($2,0)'::tid AND ctid < '($(($2 + 1)),0)'::tid"
}
from_block() { # from_block <table> <block>: live rows on that block or later
  s "SELECT count(*) FROM $1 WHERE ctid >= '($2,0)'::tid"
}
copy_block() { # copy_block <relation> <block> <image>
  dd if="$(blockfile "$1" "$2")" of="$3" bs=8192 skip="$(blockseek "$2")" count=1 2> /dev/null \
    || die "copy of $1 block $2 failed"
  [ "$(wc -c < "$3" | tr -d ' ')" = 8192 ] || die "short copy of $1 block $2"
}
put_block() { # put_block <relation-file> <seek> <image>
  dd if="$3" of="$1" bs=8192 seek="$2" count=1 conv=notrunc 2> /dev/null || die "write to $1 failed"
}
zero_block() { # zero_block <relation-file> <seek>
  dd if=/dev/zero of="$1" bs=8192 seek="$2" count=1 conv=notrunc 2> /dev/null || die "zeroing $1 failed"
}
put_bytes() { # put_bytes <file> <byte-offset> <printf-octal-string>
  printf "$3" | dd of="$1" bs=1 seek="$2" conv=notrunc 2> /dev/null || die "patching $1 failed"
}

suite() { # suite <off|on>
  local mode=$1
  DATA="$SANDBOX/data_$mode"; OUT="$OUTROOT/$mode"
  say "suite with data checksums $mode"
  own_sandbox
  source_pin
  rm -rf "$OUT"; mkdir -p "$OUT"
  : > "$OUT/results.txt"
  cluster_up "$mode"
  cat "$OUT/platform.txt" >&2

  say "fixtures"
  mk t_ok 20000
  q "UPDATE /* wiki_tc12_fixture */ t_ok SET pad = repeat('q', 200) WHERE id % 7 = 0"
  q "DELETE /* wiki_tc12_fixture */ FROM t_ok WHERE id % 13 = 0"
  q "VACUUM /* wiki_tc12_fixture */ t_ok"
  q "UPDATE /* wiki_tc12_fixture */ t_ok SET pad = repeat('r', 200) WHERE id % 11 = 0"
  q "DELETE /* wiki_tc12_fixture */ FROM t_ok WHERE id % 19 = 0"
  # A sequential scan prunes pages that are nearly full: dead HOT chains
  # leave redirect line pointers, deleted rows leave dead ones.
  q "SELECT /* wiki_tc12_fixture */ count(*) FROM t_ok"
  local t
  for t in t_cached t_trunc t_stale_heap t_stale_index t_hot t_lpcorrupt t_flip t_badidx t_toast; do
    mk "$t" 4000
  done
  mk t_midseg 12000
  q "CHECKPOINT /* wiki_tc12_fixture */"
  s "SELECT relname || ' heap_blocks=' || pg_relation_size(oid) / 8192 FROM pg_class
     WHERE relname ~ '^t_' AND relkind = 'r' ORDER BY relname" > "$OUT/fixtures.txt"

  say "baseline: healthy table"
  check t_ok ok_baseline
  verdict_is ok_baseline PASS
  # Line-pointer states the check accepted, from the same session's temp tables.
  "$BIN/psql" -X -q -A -F '|' -v ON_ERROR_STOP=1 -d tc -v table_name=public.t_ok \
      -f "$SQLD/tid_check.sql" -c "
    SELECT /* wiki_tc12_fixture */ 'heap line pointers', x.code, count(*)
    FROM tc_heap_page AS h
    CROSS JOIN LATERAL generate_series(0, length(h.codes) - 1) AS i
    CROSS JOIN LATERAL (SELECT get_byte(h.codes, i) AS code) AS x
    WHERE h.relid = 't_ok'::regclass GROUP BY x.code ORDER BY x.code" -c "
    SELECT /* wiki_tc12_fixture */ 'index TIDs on redirect roots', i.indexrelid::regclass, count(*)
    FROM tc_item AS i JOIN tc_heap_page AS h ON h.relid = i.heaprel AND h.blk = i.hblk
    WHERE i.ioff IS NOT NULL AND get_byte(h.codes, i.hoff - 1) = 2
    GROUP BY 2 ORDER BY 2;" > "$OUT/ok_codes.txt" 2>&1 || die "codes query failed"
  grep -E '^(heap line pointers|index TIDs on redirect roots)\|' "$OUT/ok_codes.txt" >> "$OUT/results.txt"

  say "scale: one larger healthy table, timed"
  q "DROP /* wiki_tc12_fixture */ TABLE IF EXISTS t_big"
  q "CREATE /* wiki_tc12_fixture */ TABLE t_big (id int PRIMARY KEY, k text NOT NULL, pad text NOT NULL, big text)"
  q "CREATE /* wiki_tc12_fixture */ INDEX t_big_k ON t_big (k)"
  q "INSERT /* wiki_tc12_fixture */ INTO t_big SELECT g, md5(g::text), repeat('p', 200), NULL
     FROM generate_series(1, 1000000) AS g"
  q "VACUUM /* wiki_tc12_fixture */ t_big"
  q "CHECKPOINT /* wiki_tc12_fixture */"
  # \timing prints one "Time: N ms" line per statement; their integer parts
  # are summed.  The published SQL itself is unchanged.
  "$BIN/psql" -X -q -A -F '|' -v ON_ERROR_STOP=1 -d tc -v table_name=public.t_big \
      -c '\timing on' -f "$SQLD/tid_check.sql" > "$OUT/big.txt" 2> "$OUT/big.err" \
    || { cat "$OUT/big.err" >&2; die "the check failed to run on t_big"; }
  verdict_is big PASS
  local ms=0 tline
  for tline in $(grep -E '^Time: [0-9]+' "$OUT/big.txt" | cut -d' ' -f2 | cut -d. -f1); do
    ms=$((ms + tline))
  done
  record "t_big: $(s "SELECT pg_relation_size('t_big') / 8192") heap blocks, $(s "SELECT (pg_relation_size('t_big_pkey') + pg_relation_size('t_big_k')) / 8192") index blocks; $(grep -E '^PASS' "$OUT/big.txt"); statement time $ms ms"
  q "DROP /* wiki_tc12_fixture */ TABLE t_big"
  q "CHECKPOINT /* wiki_tc12_fixture */"
  grep -E -q '^heap line pointers\|2\|[1-9]' "$OUT/ok_codes.txt" || die "t_ok has no redirect line pointers"
  grep -E -q '^index TIDs on redirect roots\|t_ok_pkey\|[1-9]' "$OUT/ok_codes.txt" \
    || die "no index TID of t_ok points at a redirect root"

  say "online leg: zero a page that shared buffers still hold"
  local n5 id5 total
  total=$(s "SELECT count(pad) FROM t_cached")
  n5=$(on_block t_cached 5)
  id5=$(s "SELECT min(id) FROM t_cached WHERE ctid >= '(5,0)'::tid AND ctid < '(6,0)'::tid")
  [ "$(s "SELECT count(*) FROM pg_buffercache
          WHERE relfilenode = pg_relation_filenode('t_cached') AND relforknumber = 0
            AND relblocknumber = 5 AND NOT isdirty
            AND reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())")" = 1 ] \
    || die "t_cached block 5 is not a clean shared buffer"
  zero_block "$(blockfile t_cached 5)" "$(blockseek 5)"
  record "t_cached: block 5 zeroed on disk while cached; rows on block 5 = $n5"
  local seen byidx
  seen=$(s "SELECT count(pad) FROM t_cached")
  # enable_seqscan and enable_bitmapscan are PGC_USERSET; turned off for this
  # one psql session, so the lookup is an index scan that fetches the heap
  # tuple (pad is not in any index, so no index-only scan can answer it).
  byidx=$(s "SET enable_seqscan = off; SET enable_bitmapscan = off; SELECT count(pad) FROM t_cached WHERE id = $id5")
  record "t_cached while cached: count(pad) = $seen of $total; id $id5 by index = $byidx"
  [ "$seen" = "$total" ] && [ "$byidx" = 1 ] || die "normal reads already miss the zeroed page"
  check t_cached cached_online
  verdict_is cached_online FAIL
  expect_tids cached_online t_cached_pkey heap_page_all_zero "$n5"
  expect_tids cached_online t_cached_k heap_page_all_zero "$n5"
  [ "$(s "SELECT count(*) FROM pg_buffercache
          WHERE relfilenode = pg_relation_filenode('t_cached') AND relblocknumber = 5 AND NOT isdirty
            AND reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())")" = 1 ] \
    || die "t_cached block 5 left shared buffers or became dirty"

  say "online preparation for the offline damage"
  # Lost heap write: keep t_stale_heap's last block, then add rows to it.
  local lastb n_last ins_last
  lastb=$(s "SELECT pg_relation_size('t_stale_heap') / 8192 - 1")
  n_last=$(on_block t_stale_heap "$lastb")
  copy_block t_stale_heap "$lastb" "$OUT/stale_heap_block.img"
  q "INSERT /* wiki_tc12_fixture */ INTO t_stale_heap
     SELECT g, md5(g::text), repeat('p', 200), NULL FROM generate_series(4001, 4005) AS g"
  ins_last=$(( $(on_block t_stale_heap "$lastb") - n_last ))
  [ "$ins_last" -gt 0 ] || die "no inserted row landed on t_stale_heap block $lastb"
  q "CHECKPOINT /* wiki_tc12_fixture */"
  # Lost index write: keep the primary key file, then delete and vacuum.
  [ "$(s "SELECT pg_relation_size('t_stale_index_pkey') / 8192")" -le "$RELSEG_BLOCKS" ] \
    || die "t_stale_index_pkey spans segments"
  cp "$(relfile t_stale_index_pkey 0)" "$OUT/stale_index_pkey.img" || die "copy failed"
  q "DELETE /* wiki_tc12_fixture */ FROM t_stale_index WHERE id % 10 = 0"
  q "VACUUM /* wiki_tc12_fixture */ t_stale_index"
  local heap_after
  heap_after=$(s "SELECT pg_relation_size('t_stale_index') / 8192")
  q "CHECKPOINT /* wiki_tc12_fixture */"
  # Slot reuse by a heap-only tuple: delete (0,5), vacuum, HOT-update a row
  # of page 0 so its new version takes the freed slot.
  local hot_id upd_id
  cp "$(relfile t_hot_pkey 0)" "$OUT/hot_pkey.img" || die "copy failed"
  hot_id=$(s "SELECT id FROM t_hot WHERE ctid = '(0,5)'::tid")
  upd_id=$(s "SELECT id FROM t_hot WHERE ctid = '(0,7)'::tid")
  q "DELETE /* wiki_tc12_fixture */ FROM t_hot WHERE id = $hot_id"
  q "VACUUM /* wiki_tc12_fixture */ t_hot"
  q "UPDATE /* wiki_tc12_fixture */ t_hot SET pad = repeat('h', 200) WHERE id = $upd_id"
  [ "$(s "SELECT ctid FROM t_hot WHERE id = $upd_id")" = "(0,5)" ] \
    || die "the HOT update did not reuse slot (0,5)"
  q "CHECKPOINT /* wiki_tc12_fixture */"
  # Expected counts and file positions for the offline damage.
  local trunc_lost mid_lost lp3 n3 n3flip toastrel toast2 idx_leaf1 idx_leaf2 lpoff
  trunc_lost=$(from_block t_trunc "$RELSEG_BLOCKS")
  mid_lost=$(from_block t_midseg $((RELSEG_BLOCKS + 64)))
  [ "$(s "SELECT pg_relation_size('t_trunc') / 8192")" -gt "$RELSEG_BLOCKS" ] || die "t_trunc has one segment"
  [ "$(s "SELECT pg_relation_size('t_midseg') / 8192")" -gt $((RELSEG_BLOCKS * 3)) ] || die "t_midseg too small"
  n3=$(on_block t_lpcorrupt 3)
  lp3=$(s "SELECT count(*) FROM t_lpcorrupt WHERE ctid = '(3,2)'::tid")
  [ "$lp3" = 1 ] || die "t_lpcorrupt (3,2) is not a live row"
  n3flip=$(on_block t_flip 3)
  toastrel=$(s "SELECT reltoastrelid::regclass FROM pg_class WHERE oid = 't_toast'::regclass")
  toast2=$(on_block "$toastrel" 2)
  [ "$toast2" -gt 0 ] || die "TOAST block 2 holds no chunks"
  idx_leaf1=$(s "SELECT b FROM generate_series(1, pg_relation_size('t_badidx_pkey') / 8192 - 1) AS b,
                 LATERAL bt_page_stats('t_badidx_pkey', b::int) AS st WHERE st.type = 'l' ORDER BY b LIMIT 1")
  idx_leaf2=$(s "SELECT b FROM generate_series(1, pg_relation_size('t_badidx_pkey') / 8192 - 1) AS b,
                 LATERAL bt_page_stats('t_badidx_pkey', b::int) AS st WHERE st.type = 'l' ORDER BY b OFFSET 1 LIMIT 1")
  # The primary key's 16-byte items are shorter than a heap tuple header, so
  # heap_page_items() reports only their line pointers here.
  lpoff=$(s "SELECT lp_off FROM heap_page_items(get_raw_page('t_badidx_pkey', $idx_leaf2)) WHERE lp = 2")
  local f_trunc1 f_mid1 f_stale_heap f_stale_idx f_hot_idx f_lp f_flip f_idx f_toast
  f_trunc1=$(relfile t_trunc 1); f_mid1=$(relfile t_midseg 1)
  f_stale_heap=$(blockfile t_stale_heap "$lastb"); f_stale_idx=$(relfile t_stale_index_pkey 0)
  f_hot_idx=$(relfile t_hot_pkey 0); f_lp=$(blockfile t_lpcorrupt 3); f_flip=$(blockfile t_flip 3)
  f_idx=$(relfile t_badidx_pkey 0); f_toast=$(blockfile "$toastrel" 2)
  # The online leg's zeroed page survives only while its buffer stays clean.
  local relnode_cached
  relnode_cached=$(blockfile t_cached 5)

  say "offline damage"
  server_stop
  cat /dev/null > "$f_trunc1"                                  # last segment emptied
  dd if=/dev/null of="$f_mid1" bs=8192 seek=64 2> /dev/null || die "truncate failed"  # segment 1 cut to 64 blocks
  put_block "$f_stale_heap" "$(blockseek "$lastb")" "$OUT/stale_heap_block.img"   # lost heap write
  cp "$OUT/stale_index_pkey.img" "$f_stale_idx" || die "restore failed"           # lost index write
  cp "$OUT/hot_pkey.img" "$f_hot_idx" || die "restore failed"
  # Line pointer 2 of block 3: LP_NORMAL, lp_off 8000, lp_len 400, which ends
  # past the page.  Little-endian word 0x03209F40.
  put_bytes "$f_lp" $(( $(blockseek 3) * 8192 + 24 + 4 )) '\100\237\040\003'
  cp "$f_flip" "$OUT/flip_before.img"
  put_bytes "$f_flip" $(( $(blockseek 3) * 8192 + 8100 )) '\132'                 # one byte of tuple data
  cmp -s "$f_flip" "$OUT/flip_before.img" && die "the flipped byte did not change"
  rm -f "$OUT/flip_before.img"
  put_bytes "$f_idx" $(( idx_leaf1 * 8192 + 12 )) '\377\377'                     # pd_lower = 65535
  put_bytes "$f_idx" $(( idx_leaf2 * 8192 + lpoff + 6 )) '\000\037'             # t_info size 7936
  zero_block "$f_toast" "$(blockseek 2)"                                          # one TOAST page
  record "damage: t_trunc segment 1 emptied (lost rows $trunc_lost); t_midseg segment 1 cut to 64 blocks (unreachable rows $mid_lost)"
  record "damage: t_stale_heap block $lastb restored to its image before $ins_last rows were added to it"
  record "damage: t_stale_index_pkey restored to its image before 400 rows were deleted and vacuumed (heap blocks $heap_after)"
  record "damage: t_hot_pkey restored to its image before id $hot_id at (0,5) was deleted and the slot reused by a heap-only tuple"
  record "damage: t_lpcorrupt (3,2) line pointer overwritten; rows on block 3 = $n3; t_flip byte 8100 of block 3 overwritten; rows on block 3 = $n3flip"
  record "damage: t_badidx_pkey block $idx_leaf1 pd_lower = 65535; block $idx_leaf2 item 2 t_info size = 7936"
  record "damage: $toastrel block 2 zeroed; chunks on it = $toast2"
  [ -s "$relnode_cached" ] || die "t_cached file vanished"
  server_start

  say "bypass check: the check leaves shared buffers untouched"
  check t_ok ok_cold
  verdict_is ok_cold PASS
  local relset
  relset="SELECT pg_relation_filenode(x) FROM unnest(ARRAY['t_ok'::regclass, 't_ok_pkey'::regclass, 't_ok_k'::regclass,
            (SELECT reltoastrelid FROM pg_class WHERE oid = 't_ok'::regclass)::regclass,
            (SELECT indexrelid FROM pg_index WHERE indrelid = (SELECT reltoastrelid FROM pg_class WHERE oid = 't_ok'::regclass))::regclass]) AS x"
  record "shared buffers of t_ok and its indexes and TOAST after the cold check: $(s "SELECT count(*) FROM pg_buffercache WHERE relfilenode IN ($relset) AND reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())")"
  [ "$(s "SELECT count(*) FROM pg_buffercache WHERE relfilenode IN ($relset) AND reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())")" = 0 ] \
    || die "the check put pages of t_ok into shared buffers"
  q "SELECT /* wiki_tc12_fixture */ count(*) FROM t_ok"
  record "shared buffers of the same relations after SELECT count(*) FROM t_ok: $(s "SELECT count(*) FROM pg_buffercache WHERE relfilenode IN ($relset) AND reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())")"

  say "checks of the damaged tables"
  check t_cached cached_restart
  verdict_is cached_restart FAIL
  expect_tids cached_restart t_cached_pkey heap_page_all_zero "$n5"
  seen=$(s "SELECT count(pad) FROM t_cached")
  byidx=$(s "SET enable_seqscan = off; SET enable_bitmapscan = off; SELECT count(pad) FROM t_cached WHERE id = $id5")
  record "t_cached after restart: count(pad) = $seen of $total; id $id5 by index = $byidx"
  [ "$seen" = $((total - n5)) ] && [ "$byidx" = 0 ] || die "normal reads after restart do not show the loss"

  check t_trunc trunc
  verdict_is trunc FAIL
  expect_tids trunc t_trunc_pkey heap_block_missing "$trunc_lost"
  expect_tids trunc t_trunc_k heap_block_missing "$trunc_lost"

  check t_midseg midseg
  verdict_is midseg FAIL
  grep -q 'data after a short segment' "$OUT/midseg.txt" || die "midseg: no layout finding"
  record "midseg layout: $(grep -E '^t_midseg\|table\|' "$OUT/midseg.txt" | cut -d'|' -f4-5)"
  expect_tids midseg t_midseg_pkey heap_block_missing "$mid_lost"

  check t_stale_heap stale_heap
  verdict_is stale_heap FAIL
  expect_tids stale_heap t_stale_heap_pkey offset_beyond_line_pointers "$ins_last"
  expect_tids stale_heap t_stale_heap_k offset_beyond_line_pointers "$ins_last"

  check t_stale_index stale_index
  verdict_is stale_index FAIL
  expect_tids stale_index t_stale_index_pkey line_pointer_unused 400
  expect_tids stale_index t_stale_index_k line_pointer_unused 0

  check t_hot hot
  verdict_is hot FAIL
  expect_tids hot t_hot_pkey points_to_heap_only_tuple 1

  check t_lpcorrupt lpcorrupt
  verdict_is lpcorrupt FAIL
  check t_flip flip
  check t_badidx badidx
  verdict_is badidx FAIL
  if [ "$mode" = on ]; then
    expect_tids lpcorrupt t_lpcorrupt_pkey heap_page_bad_checksum "$n3"
    expect_page lpcorrupt t_lpcorrupt bad_checksum
    verdict_is flip FAIL
    expect_tids flip t_flip_pkey heap_page_bad_checksum "$n3flip"
    expect_page badidx t_badidx_pkey bad_checksum
  else
    expect_tids lpcorrupt t_lpcorrupt_pkey line_pointer_corrupt 1
    expect_tids lpcorrupt t_lpcorrupt_k line_pointer_corrupt 1
    verdict_is flip PASS
    expect_page badidx t_badidx_pkey bad_header
    expect_page badidx t_badidx_pkey bad_items
  fi

  check t_toast toast
  verdict_is toast FAIL
  expect_tids toast "$toastrel"_index heap_page_all_zero "$toast2"
  expect_tids toast t_toast_pkey ok 4000
  "$BIN/psql" -X -q -At -d tc -c "SELECT /* wiki_tc12_fixture */ sum(length(big)) FROM t_toast" \
    > "$OUT/toast_read.txt" 2>&1 && die "reading t_toast's TOAST values did not fail"
  record "t_toast detoast: $(grep -E 'ERROR' "$OUT/toast_read.txt" | head -n 1)"

  if [ "$mode" = off ]; then
    say "options and refusals"
    check_opt t_stale_index one_index -v index_name=public.t_stale_index_pkey
    verdict_is one_index FAIL
    expect_tids one_index t_stale_index_pkey line_pointer_unused 400
    grep -q '^t_stale_index_k|' "$OUT/one_index.txt" && die "one_index: t_stale_index_k was checked"
    record "one_index: only t_stale_index_pkey was read"
    check_opt t_toast no_toast -v include_toast=off
    verdict_is no_toast PASS
    q "CREATE UNLOGGED TABLE t_unlogged (id int PRIMARY KEY)"
    q "CREATE ROLE tc_plain LOGIN"
    "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d postgres -c 'CREATE /* wiki_tc12_fixture */ DATABASE tc_nopi' \
      > /dev/null || die "createdb failed"
    "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d tc_nopi \
      -c 'CREATE /* wiki_tc12_fixture */ TABLE t (id int PRIMARY KEY)' > /dev/null || die "fixture failed"
    expect_refusal refuse_unlogged 'permanent ordinary table' -d tc -v table_name=public.t_unlogged
    expect_refusal refuse_user 'run as a superuser' -d tc -U tc_plain -v table_name=public.t_ok
    expect_refusal refuse_nopi 'CREATE EXTENSION pageinspect' -d tc_nopi -v table_name=public.t
    expect_refusal refuse_index 'must name an index of table_name' -d tc \
      -v table_name=public.t_ok -v index_name=public.t_hot_pkey
  fi

  server_stop
  note "suite $mode passed; results in $OUT/results.txt"
}
stage_suite_off() { suite off; }
stage_suite_on() { suite on; }

stage_report() {
  own_sandbox
  local m
  for m in off on; do
    [ -f "$OUTROOT/$m/results.txt" ] || continue
    say "checksums $m: $OUTROOT/$m"
    cat "$OUTROOT/$m/platform.txt" "$OUTROOT/$m/results.txt" >&2
  done
}

# --------------------------------------------------------------- teardown ----
stage_stop() {
  say "stop clusters"
  [ -e "$SANDBOX" ] || { note "sandbox absent"; return 0; }
  own_sandbox
  local d
  for d in "$SANDBOX/data_off" "$SANDBOX/data_on"; do
    [ -d "$d" ] || continue
    if "$BIN/pg_ctl" -D "$d" status > /dev/null 2>&1; then
      [ -f "$SANDBOX/.started" ] || die "refusing to stop an unowned server in $d"
      "$BIN/pg_ctl" -D "$d" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed for $d"
    fi
    [ ! -f "$d/postmaster.pid" ] || die "postmaster.pid survived in $d"
  done
  rm -f "$SANDBOX/.started"
  # Match only processes started from the sandbox's binaries or carrying the
  # cluster_name title, so a shell whose command line names the path does not count.
  if pgrep -f -- "^$BIN/" > /dev/null 2>&1 || pgrep -f -- '^postgres: tc12:' > /dev/null 2>&1; then
    die "a sandbox server process is still running"
  fi
  [ ! -d "$SOCK" ] || [ -z "$(ls -A "$SOCK")" ] || die "socket directory is not free"
  if [ -x "$BIN/pg_isready" ]; then
    "$BIN/pg_isready" -q -h "$SOCK" -p "$PORT12" && die "socket is still active"
  fi
  note "stopped: no pid file, sandbox process or socket"
}
stage_clean() {
  say "delete sandbox $SANDBOX"
  stage_stop
  [ -e "$SANDBOX" ] || return 0
  own_sandbox
  rm -rf "$SANDBOX"
  note "deleted $SANDBOX"
}

# ------------------------------------------------------------- driver --------
stages="$*"
[ $# -eq 0 ] && stages="build extract suite_off suite_on report"
for st in $stages; do
  case "$(type -t "stage_$st" 2>/dev/null)" in
    function) "stage_$st" ;;
    *) die "unknown stage: $st" ;;
  esac
done
```

## Context Reviewed

- Request: the prompt and its follow-up "don't use amcheck" were combined and corrected under `## Question`; the original wording is in the `wiki/log.md` entry for this page.
- pageinspect 1.7 at the pin: the SQL definitions of `get_raw_page`, `page_header`, `heap_page_items`, `bt_page_items` (both forms), `hash_page_items`, `gin_leafpage_items` and `page_checksum`, and their C code, including every superuser test and every bounds check or missing one (`rawpage.c`, `btreefuncs.c`, `heapfuncs.c`, the `pageinspect--*.sql` scripts); and `pg_prewarm`'s `read` mode, which bypasses shared buffers but returns only a block count (`pg_prewarm.c`).
- Core file and catalog functions: `pg_read_binary_file`, `read_binary_file`, `convert_and_check_filename`, `pg_stat_file`, the `PUBLIC` revokes, `pg_relation_filepath`, `calculate_relation_size`, `pg_control_init`, `pg_control_system` and the control-file layout (`genfile.c`, `system_views.sql`, `dbsize.c`, `pg_controldata.c`, `pg_control.h`, `pg_proc.dat`).
- Page and tuple formats: `PageHeaderData`, `PageInit`, `PageIsVerified`, `PageRepairFragmentation`, `ItemIdData`, `IndexTupleData`, `ItemPointerData`, `HeapTupleHeaderData`, `BTPageOpaqueData`, the B-tree page flags, high key, metapage, half-dead and deleted pages, and the checksum algorithm (`bufpage.h`, `bufpage.c`, `itemid.h`, `itup.h`, `itemptr.h`, `block.h`, `htup_details.h`, `nbtree.h`, `nbtpage.c`, `checksum_impl.h`).
- Why index TIDs stay valid: README.HOT, pruning, VACUUM's index-then-heap order, `INDEX_CLEANUP` off, truncation of trailing pages, `vac_open_indexes` and `indisready`, `RelationGetIndexList` and `indislive`, and `_bt_killitems` (`README.HOT`, `pruneheap.c`, `vacuumlazy.c`, `vacuum.c`, `relcache.c`, `nbtutils.c`).
- Memory against disk: `BufferSync`, `ReadBuffer_common`'s zero-filled extension, `FlushBuffer`'s checksum, the `CHECKPOINT` statement, `ExecInsert`'s heap-then-index order, local buffers for temporary tables, and the base backup torn-block retry (`bufmgr.c`, `utility.c`, `nodeModifyTable.c`, `rel.h`, `basebackup.c`).
- Locks: the conflict table, the lock-mode documentation, `LOCK TABLE`'s transaction-block rule and relkind check, the recovery limit, the cancellation of a blocking autovacuum worker, the `REINDEX`, `DROP INDEX` and VACUUM lock levels, autovacuum's separate TOAST pass, `PREPARE`'s parse-only path, `addRangeTableEntry`'s retained lock, conditional truncation, and the planner's B-tree metapage read (`lock.c`, `proc.c`, `mvcc.sgml`, `utility.c`, `lockcmds.c`, `indexcmds.c`, `tablecmds.c`, `vacuum.c`, `autovacuum.c`, `prepare.c`, `parse_relation.c`, `vacuumlazy.c`, `plancat.c`, `nbtpage.c`).
- Storage manager: `mdnblocks`, `_mdfd_getseg`, `_mdfd_openseg` and the open flags (`md.c`).
- SQL and PL/pgSQL behavior the script relies on: SQL-function inlining rules, PL/pgSQL `IF ... THEN` parsing, `initdb` installing PL/pgSQL, and the GUC definitions for every setting the check or the measurement changes (`clauses.c`, `pl_gram.y`, `initdb.c`, `guc.c`).
- Consequences used in the results: the all-zero page path in `PageIsVerified`, `heap_hot_search_buffer`, index-only scans and the visibility map, and the TOAST "missing chunk" error (`bufpage.c`, `heapam.c`, `nodeIndexonlyscan.c`, `tuptoaster.c`).
- Measurement: the script in this page was run against the pin on 2026-10-22, as recorded under [Last run](#last-run).
- Comparison set for the contradiction check: the [relation-tail check](../storage-and-vacuum/zeroed-last-segment-check-all-tables.md), whose index TID witness reads through shared buffers and also skips the high key; the [amcheck follow-up](amcheck-zeroed-relation-segments.md); the [zeroed-file page](../storage-and-vacuum/zeroed-relation-file.md); the [corruption-log reference](../observability/corruption-log-messages-with-checksums-disabled.md); the [invalid-index page](invalid-index-outcomes.md), which cites the same `vac_open_indexes` rule; and the glossary entries this page links.
- Common concept pages: PostgreSQL 12 has none, so this page explains line-pointer states and buffer-versus-file freshness itself.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| pageinspect's page readers use shared buffers; its `bytea` decoders take any page image | [rawpage.c#get_raw_page_internal](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L161-L166), [btreefuncs.c:357](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L357), [pageinspect--1.5--1.6.sql](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L82-L99) |
| `pg_read_binary_file()` reads with stdio, returns short or NULL results instead of erroring, and is revoked from `PUBLIC` | [genfile.c#read_binary_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L110-L177), [system_views.sql:1330-1335](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1330-L1335) |
| Data files are opened without `O_DIRECT` | [md.c:450](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L450), [md.c#_mdfd_openseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1071) |
| Blocks after the first short segment are unreachable | [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755), [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1180-L1205); measured `t_midseg` |
| Index entries point at HOT roots; heap-only tuples have none | [README.HOT:45-55](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L45-L55), [README.HOT:296-301](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L296-L301); measured `t_hot` |
| Pruning never frees a root | [pruneheap.c#heap_prune_chain](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L384-L411), [pruneheap.c:570-615](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L570-L615), [README.HOT:116-122](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L116-L122) |
| VACUUM frees line pointers only after index cleanup, and truncates only all-unused trailing pages | [vacuumlazy.c:760-785](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L760-L785), [vacuumlazy.c#count_nondeletable_pages](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2108-L2119); measured `t_stale_index` |
| A heap page's line-pointer array never shrinks | [bufpage.c#PageRepairFragmentation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L482-L570); measured `t_stale_heap` |
| Not-ready and not-live indexes are not vacuumed | [vacuum.c#vac_open_indexes](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1869-L1918), [relcache.c:4394](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4394) |
| A checkpoint writes every permanent dirty buffer that existed when it began | [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1785-L1846), [utility.c#CheckPointStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L756-L770) |
| `SHARE ROW EXCLUSIVE` blocks writers, VACUUM, index builds and rewrites but not `SELECT` | [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L104), [mvcc.sgml#SHARE-ROW-EXCLUSIVE](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L958-L978); measured lock report |
| `LOCK TABLE` refuses TOAST tables; autovacuum vacuums them separately | [lockcmds.c#RangeVarCallbackForLockTable](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L82-L104), [autovacuum.c:2108-2113](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2108-L2113) |
| `PREPARE` keeps `ACCESS SHARE` on the TOAST table without planning, which blocks VACUUM truncation | [prepare.c#PrepareQuery](../../../../raw/postgres-12/src/backend/commands/prepare.c#L128-L180), [parse_relation.c#addRangeTableEntry](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1229-L1254), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1902); measured lock report |
| Planning a query on the table would read B-tree metapages through shared buffers | [plancat.c:409-413](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L413), [nbtpage.c#_bt_getrootheight](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L586-L626) |
| `bt_page_items(bytea)` and `heap_page_items()` trust parts of the page | [btreefuncs.c#bt_page_print_tuples](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L256-L299), [btreefuncs.c#bt_page_items_bytea](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L422-L466), [heapfuncs.c:166](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L166), [heapfuncs.c:229-234](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L229-L234) |
| Page validation and checksum rules | [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162), [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L187-L208), [basebackup.c:1480](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1480); measured `t_flip`, `t_lpcorrupt`, `t_badidx` |
| A torn read can fail once and pass on re-read | [basebackup.c#torn-block-retry](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1470-L1493) |
| Line-pointer decoding matches the compiler's bitfield layout on the test host | [itemid.h#ItemIdData](../../../../raw/postgres-12/src/include/storage/itemid.h#L25-L41); measured self-test, 47 items of `pg_class` block 0 |
| Live leaf pages, the high key and half-dead stubs | [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-12/src/include/access/nbtree.h#L55-L78), [nbtree.h#P_HIKEY](../../../../raw/postgres-12/src/include/access/nbtree.h#L181-L219), [nbtpage.c:1648-1665](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1648-L1665) |
| A zeroed page stays visible through a clean cached copy, then disappears silently | [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L123-L142), [heapam.c#heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1556); measured `t_cached` |
| The check leaves the checked relations out of shared buffers | Measured: 0 buffers after the cold check, 901 after a plain `count(*)` |

## Open Questions

- **The kernel page cache.** PostgreSQL 12 has no SQL-level way to read a data file past the OS page cache. If storage loses a write that the kernel still caches, both the server and this check read the cached copy. How to evict a file from the kernel cache before the check depends on the operating system, which is outside this wiki's evidence base.
- **Big-endian platforms.** The script decodes line pointers most-significant-bit first when the control file shows a big-endian server. That branch was never run. The self-test should stop the script if the guess is wrong, but no big-endian build was available to confirm it.
- **Branches not exercised.** The measurement did not produce a torn read, a file-path change during the check, a concurrent TOAST VACUUM, a not-ready index, or a hash, GiST or GIN index. Their handling on this page is derived from the source only.
- **Hash indexes.** pageinspect 1.7's `hash_page_items(bytea)` returns a TID per item [pageinspect--1.5--1.6.sql:35-43](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L35-L43), so hash indexes could be checked the same way. The script does not implement it.
- **A lock-free variant.** Without the table lock, a confirming re-read of each finding after one more checkpoint should separate concurrent inserts from real loss, but neither that design nor its interaction with concurrent VACUUM has been written or measured.
- **Cost on cold storage.** The measured cost comes from one host, with files that had just been written. The time on storage-bound reads is not measured.
- **Documentation wording.** The lock-mode documentation lists `REINDEX` among the commands that acquire `ACCESS EXCLUSIVE` [mvcc.sgml#ACCESS-EXCLUSIVE](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1002-L1030), without saying on which object. The source takes `SHARE` on the table and `ACCESS EXCLUSIVE` on the index [indexcmds.c:2397-2402](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2397-L2402); this page follows the source.

## Source References

- pageinspect and pg_prewarm: [pg_prewarm.c](../../../../raw/postgres-12/contrib/pg_prewarm/pg_prewarm.c#L171-L184), [rawpage.c](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L95-L324), [btreefuncs.c](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L203-L466), [heapfuncs.c](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L128-L283), [pageinspect--1.5.sql](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L22-L279), [pageinspect--1.5--1.6.sql](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L35-L99).
- File and catalog functions: [genfile.c](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L110-L323), [system_views.sql](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1330-L1335), [dbsize.c](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1022), [pg_controldata.c](../../../../raw/postgres-12/src/backend/utils/misc/pg_controldata.c#L32-L350), [pg_control.h](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L25-L122).
- Page and tuple formats: [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L152-L345), [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L42-L570), [itemid.h](../../../../raw/postgres-12/src/include/storage/itemid.h#L17-L86), [itup.h](../../../../raw/postgres-12/src/include/access/itup.h#L35-L90), [itemptr.h](../../../../raw/postgres-12/src/include/storage/itemptr.h#L36-L47), [block.h](../../../../raw/postgres-12/src/include/storage/block.h#L53-L57), [htup_details.h](../../../../raw/postgres-12/src/include/access/htup_details.h#L152-L281), [nbtree.h](../../../../raw/postgres-12/src/include/access/nbtree.h#L55-L219), [nbtpage.c](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L50-L1665), [nbtutils.c](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1718-L1793), [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L187-L208).
- Heap and VACUUM: [README.HOT](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L45-L301), [pruneheap.c](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L384-L615), [vacuumlazy.c](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L760-L2119), [vacuum.c](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1918), [relcache.c](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4327-L4394), [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1526-L1556), [tuptoaster.c](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1865-L2013), [nodeIndexonlyscan.c](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L169), [nodeModifyTable.c](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L586-L593).
- Buffers, checkpoints and storage: [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L866-L2750), [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L450-L1205), [rel.h](../../../../raw/postgres-12/src/include/utils/rel.h#L523-L527), [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1470-L1493), [pg_config.h.in](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864).
- Locks and commands: [lock.c](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L766), [proc.c](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1312-L1349), [mvcc.sgml](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L1030), [utility.c](../../../../raw/postgres-12/src/backend/tcop/utility.c#L740-L770), [lockcmds.c](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L82-L104), [indexcmds.c](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2397-L2402), [tablecmds.c](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1379-L1380), [autovacuum.c](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2108-L2113), [prepare.c](../../../../raw/postgres-12/src/backend/commands/prepare.c#L128-L180), [parse_relation.c](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1229-L1254), [plancat.c](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L413).
- SQL, PL/pgSQL and settings: [pg_proc.dat](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6065-L6073), [pl_gram.y](../../../../raw/postgres-12/src/pl/plpgsql/src/pl_gram.y#L2430-L2431), [clauses.c](../../../../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4600-L4602), [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144-L3054), [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L884-L4241).

## Navigation

- [Checking every table for a zeroed last segment (unverified)](../storage-and-vacuum/zeroed-last-segment-check-all-tables.md)
- [Can amcheck detect zeroed relation segments? (unverified)](amcheck-zeroed-relation-segments.md)
- [A manually zeroed relation file (unverified)](../storage-and-vacuum/zeroed-relation-file.md)
- [Corruption indicators with checksums disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md)
- [v12/index](../../index.md) · [wiki index](../../../index.md) · [versions](../../../versions.md) · [Wiki Glossary (unverified)](../../../glossary.md)
