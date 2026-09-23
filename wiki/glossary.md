---
type: glossary
verified: false
verified_by_agent: not yet
---

# Wiki Glossary (unverified)

## Contents

- [Scope](#scope)
- [Source Pins](#source-pins)
- [Terms](#terms)
  - [Access method](#access-method)
  - [AccessExclusiveLock](#accessexclusivelock)
  - [allequalimage](#allequalimage)
  - [Apply worker](#apply-worker)
  - [Autovacuum](#autovacuum)
  - [B-tree](#b-tree)
  - [Backend](#backend)
  - [Background worker](#background-worker)
  - [Background writer](#background-writer)
  - [Bitmap scan](#bitmap-scan)
  - [BKI](#bki)
  - [BLCKSZ](#blcksz)
  - [Bloat](#bloat)
  - [Block](#block)
  - [Bottom-up index deletion](#bottom-up-index-deletion)
  - [BRIN](#brin)
  - [Buffer manager](#buffer-manager)
  - [Catalog](#catalog)
  - [Checkpoint](#checkpoint)
  - [Clock sweep](#clock-sweep)
  - [CLUSTER](#cluster)
  - [COMMENT ON](#comment-on)
  - [CONCURRENTLY](#concurrently)
  - [Constraint exclusion](#constraint-exclusion)
  - [Contrib](#contrib)
  - [Cost](#cost)
  - [Crash recovery](#crash-recovery)
  - [Critical section](#critical-section)
  - [Cumulative statistics](#cumulative-statistics)
  - [Deduplication](#deduplication)
  - [Dirty buffer](#dirty-buffer)
  - [effective_cache_size](#effective_cache_size)
  - [ereport](#ereport)
  - [Event trigger](#event-trigger)
  - [Executor](#executor)
  - [EXPLAIN BUFFERS](#explain-buffers)
  - [EXPLAIN](#explain)
  - [Expression index](#expression-index)
  - [Extension](#extension)
  - [Fast root](#fast-root)
  - [Fillfactor](#fillfactor)
  - [fmgr](#fmgr)
  - [Fork](#fork)
  - [Free space map](#free-space-map)
  - [Freezing](#freezing)
  - [Full-page image](#full-page-image)
  - [GIN](#gin)
  - [GiST](#gist)
  - [Grammar](#grammar)
  - [GUC context](#guc-context)
  - [GUC](#guc)
  - [Hash index](#hash-index)
  - [Heap](#heap)
  - [Heavyweight lock](#heavyweight-lock)
  - [Hook](#hook)
  - [HOT](#hot)
  - [Huge pages](#huge-pages)
  - [Index-only scan](#index-only-scan)
  - [IndexOptInfo](#indexoptinfo)
  - [Injection point](#injection-point)
  - [Invalid index](#invalid-index)
  - [Isolation test](#isolation-test)
  - [Line pointer](#line-pointer)
  - [Logical decoding](#logical-decoding)
  - [Logical replication](#logical-replication)
  - [LSN](#lsn)
  - [LWLock](#lwlock)
  - [maintenance_work_mem](#maintenance_work_mem)
  - [Memory context](#memory-context)
  - [Metapage](#metapage)
  - [MultiXact](#multixact)
  - [MVCC](#mvcc)
  - [OID](#oid)
  - [Page split](#page-split)
  - [Page](#page)
  - [pageinspect](#pageinspect)
  - [Parse tree](#parse-tree)
  - [Partial index](#partial-index)
  - [Partition pruning](#partition-pruning)
  - [Partitionwise join](#partitionwise-join)
  - [Path](#path)
  - [Pending list](#pending-list)
  - [pgstatindex](#pgstatindex)
  - [pgstattuple](#pgstattuple)
  - [pg_cast](#pg_cast)
  - [pg_class](#pg_class)
  - [pg_freespacemap](#pg_freespacemap)
  - [pg_index](#pg_index)
  - [pg_upgrade](#pg_upgrade)
  - [PlannedStmt](#plannedstmt)
  - [Planner](#planner)
  - [PL/pgSQL](#plpgsql)
  - [Portal](#portal)
  - [Posting list](#posting-list)
  - [Posting tree](#posting-tree)
  - [Postmaster](#postmaster)
  - [Pruning](#pruning)
  - [Publication](#publication)
  - [Regression test](#regression-test)
  - [REINDEX](#reindex)
  - [Relcache](#relcache)
  - [Relfilenumber](#relfilenumber)
  - [RelOptInfo](#reloptinfo)
  - [reltuples and relpages](#reltuples-and-relpages)
  - [Replication origin](#replication-origin)
  - [Replication slot](#replication-slot)
  - [Ring buffer](#ring-buffer)
  - [Selectivity](#selectivity)
  - [shared_buffers](#shared_buffers)
  - [ShareUpdateExclusiveLock](#shareupdateexclusivelock)
  - [Simple index deletion](#simple-index-deletion)
  - [Snapshot](#snapshot)
  - [SP-GiST](#sp-gist)
  - [SPI](#spi)
  - [Statistics](#statistics)
  - [Subscription](#subscription)
  - [Syscache](#syscache)
  - [Table rewrite](#table-rewrite)
  - [TAP test](#tap-test)
  - [TID](#tid)
  - [TOAST](#toast)
  - [Transaction ID](#transaction-id)
  - [Truncation](#truncation)
  - [Tuple](#tuple)
  - [Utility command](#utility-command)
  - [VACUUM FULL](#vacuum-full)
  - [VACUUM](#vacuum)
  - [Visibility map](#visibility-map)
  - [Wait event](#wait-event)
  - [WAL](#wal)
  - [work_mem](#work_mem)
  - [Wraparound](#wraparound)
  - [xmin and xmax](#xmin-and-xmax)
  - [xmin horizon](#xmin-horizon)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Scope

This is the one glossary for the whole wiki, shared by every PostgreSQL version. It is written for readers who are not PostgreSQL source developers. It explains the jargon, acronyms, source-code names and advanced concepts that wiki pages and the pinned source trees use.

- Each entry leads with a plain-language definition, then names the source symbols that carry the concept, then cites the evidence.
- Each entry states the versions it was checked on in its **Checked on:** line. A definition applies only to those versions. The shared page does not imply that a term means the same thing in every version.
- As of 2026-09-23, every entry was checked against PostgreSQL 17 only (`raw/postgres-17/`, pinned below). Before relying on an entry for PostgreSQL 12, 14, 18 or 19, check that version's own checkout and add a version qualification to the entry.
- A glossary link supplies vocabulary, not proof. A page that links a term still needs its own matching-version source citations.
- Deeper, version-local explanations belong on `wiki/vNN/common-concepts/` pages, which entries link when one exists.

## Source Pins

| Version | Checkout | Branch | Pinned commit |
|---|---|---|---|
| 17 | `raw/postgres-17/` | `REL_17_STABLE` | `786db8dcf168bd9df8f55047337525ac19118b1c` |

## Terms

### Access method

**Aliases:** AM, index access method, table access method, `pg_am`. **Checked on:** PostgreSQL 17.

An access method is a pluggable implementation of how a relation stores and finds data. Each row of the `pg_am` catalog names one, gives its handler function, and marks it as an index (`i`) or table (`t`) method ([pg_am.h#FormData_pg_am](../raw/postgres-17/src/include/catalog/pg_am.h#L29-L41), [pg_am.h#AMTYPE_INDEX](../raw/postgres-17/src/include/catalog/pg_am.h#L59-L63)). PostgreSQL 17 ships `heap` as its only table AM, plus six index AMs: `btree`, `hash`, `gist`, `gin`, `spgist` and `brin` ([pg_am.dat](../raw/postgres-17/src/include/catalog/pg_am.dat#L14-L35)). When reading source, `GetIndexAmRoutine()` calls the handler, which returns an `IndexAmRoutine` of capability flags and callbacks such as `amgettuple`, `amgetbitmap` and `amcanreturn`. Core code checks those fields instead of testing the index type ([amapi.c#GetIndexAmRoutine](../raw/postgres-17/src/backend/access/index/amapi.c#L24-L46), [amapi.h#IndexAmRoutine](../raw/postgres-17/src/include/access/amapi.h#L270-L296)).

Related: [B-tree](#b-tree), [GIN](#gin), [GiST](#gist), [SP-GiST](#sp-gist), [BRIN](#brin), [Hash index](#hash-index), [Heap](#heap)

### AccessExclusiveLock

**Aliases:** `ACCESS EXCLUSIVE` lock mode, lock mode 8. **Checked on:** PostgreSQL 17.

The strongest table-level lock mode. It conflicts with every other mode, so its holder is the only transaction touching the table in any way, including plain `SELECT` ([lockdefs.h#AccessExclusiveLock](../raw/postgres-17/src/include/storage/lockdefs.h#L45-L47), [lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)). The docs list `DROP TABLE`, `TRUNCATE`, `REINDEX`, `CLUSTER`, `VACUUM FULL` and many `ALTER TABLE` forms as taking it ([mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1067-L1090)). In source, look for it in `vacuum_rel()`, which picks it only for `VACUUM FULL` ([vacuum.c:2054-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2054-L2055)). Plain VACUUM's end-of-table truncation also requests it, but only with `ConditionalLockRelation()`. It retries for a short time and then gives up rather than queue for the lock ([vacuumlazy.c#lazy_truncate_heap](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2572-L2600)).

Related: [Heavyweight lock](#heavyweight-lock), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [Truncation](#truncation), [VACUUM FULL](#vacuum-full)

### allequalimage

**Aliases:** `btm_allequalimage`, "equality is image equality". **Checked on:** PostgreSQL 17.

`allequalimage` is a flag in a B-tree metapage. It records whether every key column's operator class promises that equal values are also bitwise-identical, which is the condition that makes deduplication safe. `_bt_allequalimage()` computes it from each opclass's `BTEQUALIMAGE_PROC` support function, and index builds store the result ([nbtree.h#BTMetaPageData](../raw/postgres-17/src/include/access/nbtree.h#L103-L119), [nbtutils.c#_bt_allequalimage](../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5140)). Only a build can set it to true. `_bt_upgrademetapage()` forces it to false. As `nbtree.h` notes, `pg_upgrade` never sets it, so indexes carried over from PostgreSQL 12 need `REINDEX` before they can deduplicate ([nbtpage.c#_bt_upgrademetapage](../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L106-L126), [nbtree.h:135-141](../raw/postgres-17/src/include/access/nbtree.h#L135-L141)).

Related: [Deduplication](#deduplication), [Metapage](#metapage), [pg_upgrade](#pg_upgrade), [REINDEX](#reindex)

### Apply worker

**Aliases:** logical replication apply worker. **Checked on:** PostgreSQL 17.

An apply worker is the subscriber-side process that receives a subscription's change stream and replays it into local tables. The logical replication launcher starts one per enabled subscription, as a dynamic background worker whose entry point is `ApplyWorkerMain` ([worker.c header](../raw/postgres-17/src/backend/replication/logical/worker.c#L10-L16), [launcher.c#logicalrep_worker_launch](../raw/postgres-17/src/backend/replication/logical/launcher.c#L476-L485)). In source, `run_apply_worker` sets up the subscription's replication origin before it starts streaming, and `apply_dispatch` routes each protocol message ([worker.c#run_apply_worker](../raw/postgres-17/src/backend/replication/logical/worker.c#L4516-L4525), [worker.c#apply_dispatch](../raw/postgres-17/src/backend/replication/logical/worker.c#L3296-L3300)). Do not confuse it with the tablesync worker or the parallel apply worker. The launcher starts those two separately ([launcher.c#logicalrep_worker_launch](../raw/postgres-17/src/backend/replication/logical/launcher.c#L487-L503)).

Related: [Background worker](#background-worker), [Logical replication](#logical-replication), [Replication origin](#replication-origin), [Subscription](#subscription)

### Autovacuum

**Aliases:** autovacuum launcher, autovacuum worker. **Checked on:** PostgreSQL 17.

Autovacuum is the set of background processes that run [VACUUM](#vacuum) and ANALYZE on their own schedule. One launcher process plans the work, and the [postmaster](#postmaster) forks the worker processes that do it ([autovacuum.c:7-27](../raw/postgres-17/src/backend/postmaster/autovacuum.c#L7-L27), [glossary.sgml#glossary-autovacuum](../raw/postgres-17/doc/src/sgml/glossary.sgml#L125-L142)). A worker vacuums a table when its dead-tuple count passes a base threshold plus a scale factor times `reltuples`. It also vacuums when inserts pass a similar insert threshold ([autovacuum.c#relation_needs_vacanalyze](../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3070-L3095)). Autovacuum runs only while both the `autovacuum` and `track_counts` settings are on ([autovacuum.c#AutoVacuumingActive](../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3236-L3241)). The `autovacuum` setting has context `sighup`, so changing it needs a reload ([guc_tables.c:1450-1456](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1456)).

Related: [VACUUM](#vacuum), [reltuples and relpages](#reltuples-and-relpages), [Cumulative statistics](#cumulative-statistics), [GUC context](#guc-context)

### B-tree

**Aliases:** btree, nbtree. **Checked on:** PostgreSQL 17.

A B-tree is PostgreSQL's default index type: a balanced, sorted tree that answers equality and range searches and can return rows in order. `CREATE INDEX` without `USING` picks it ([index.h#DEFAULT_INDEX_TYPE](../raw/postgres-17/src/include/catalog/index.h#L21), [nbtree.c#bthandler](../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L96-L117)). The implementation lives in `src/backend/access/nbtree/`. It follows the Lehman and Yao algorithm, which adds a right-link and a "high key" to each page so that searches can detect a concurrent page split ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L1-L28)). Block 0 is always the metapage ([nbtree.h#BTREE_METAPAGE](../raw/postgres-17/src/include/access/nbtree.h#L148)).

Related: [Access method](#access-method), [Metapage](#metapage), [Page split](#page-split), [Deduplication](#deduplication), [Bottom-up index deletion](#bottom-up-index-deletion)

### Backend

**Aliases:** backend process. **Checked on:** PostgreSQL 17.

A backend is the server process that serves one client session and handles its requests ([glossary.sgml#glossary-backend](../raw/postgres-17/doc/src/sgml/glossary.sgml#L173-L187)). The postmaster forks one per accepted connection in `BackendStartup` ([postmaster.c#BackendStartup](../raw/postgres-17/src/backend/postmaster/postmaster.c#L3525-L3536)). The child enters `BackendMain`, which authenticates the client and then starts the main loop ([backend_startup.c#BackendMain](../raw/postgres-17/src/backend/tcop/backend_startup.c#L50-L58)). That loop is `PostgresMain`, where every backend runs its queries ([postgres.c#PostgresMain](../raw/postgres-17/src/backend/tcop/postgres.c#L4247-L4259)). The glossary itself warns against confusing a backend with a background worker or the background writer ([glossary.sgml#glossary-backend](../raw/postgres-17/doc/src/sgml/glossary.sgml#L180-L184)).

Related: [Background worker](#background-worker), [Postmaster](#postmaster)

### Background worker

**Aliases:** bgworker. **Checked on:** PostgreSQL 17.

A background worker is a server process that runs built-in or extension-supplied code instead of serving a client connection ([glossary.sgml#glossary-background-worker](../raw/postgres-17/doc/src/sgml/glossary.sgml#L189-L204)). A `BackgroundWorker` struct describes the worker: its name, its start time, its restart interval, and the library and function to run ([bgworker.h#BackgroundWorker](../raw/postgres-17/src/include/postmaster/bgworker.h#L89-L101)). Static workers register through `RegisterBackgroundWorker`, which only has an effect from the postmaster or from `shared_preload_libraries` initialization. Running backends use `RegisterDynamicBackgroundWorker` instead ([bgworker.c#RegisterBackgroundWorker](../raw/postgres-17/src/backend/postmaster/bgworker.c#L854-L862), [bgworker.c#RegisterDynamicBackgroundWorker](../raw/postgres-17/src/backend/postmaster/bgworker.c#L959-L971)). Do not confuse a background worker with a backend or with the background writer ([glossary.sgml#glossary-backend](../raw/postgres-17/doc/src/sgml/glossary.sgml#L180-L184)).

Related: [Apply worker](#apply-worker), [Background writer](#background-writer), [Backend](#backend), [Postmaster](#postmaster)

### Background writer

**Aliases:** bgwriter. **Checked on:** PostgreSQL 17.

An auxiliary server process that writes dirty pages from shared buffers to the file system a little at a time, so that ordinary backends rarely have to write a buffer themselves before reusing it ([bgwriter.c](../raw/postgres-17/src/backend/postmaster/bgwriter.c#L5-L13), [glossary.sgml#background-writer](../raw/postgres-17/doc/src/sgml/glossary.sgml#L209-L221)). Since 9.2 it no longer performs checkpoints; the checkpointer process does ([bgwriter.c:13](../raw/postgres-17/src/backend/postmaster/bgwriter.c#L13)). Its main loop calls `BgBufferSync()` in the buffer manager ([bgwriter.c:234](../raw/postgres-17/src/backend/postmaster/bgwriter.c#L234)). Its pace is set by `bgwriter_delay`, a `sighup` GUC, so changing it needs a reload ([guc_tables.c#bgwriter_delay](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3077-L3084)).

Related: [Buffer manager](#buffer-manager), [Checkpoint](#checkpoint), [Dirty buffer](#dirty-buffer), [shared_buffers](#shared_buffers)

### Bitmap scan

**Aliases:** Bitmap Index Scan, Bitmap Heap Scan, TIDBitmap. **Checked on:** PostgreSQL 17.

A bitmap scan runs in two steps. First, an index scan collects matching row addresses into an in-memory bitmap. Then a heap scan visits the table pages the bitmap names, and the bitmap's iterator hands them out from sorted page lists ([nodeBitmapHeapscan.c](../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L1-L17), [tidbitmap.c#tbm_begin_iterate](../raw/postgres-17/src/backend/nodes/tidbitmap.c#L710-L747)). `tidbitmap.c` stores the set, and it can turn "lossy": it keeps only whole pages, not tuple offsets, so the heap step has to recheck the conditions ([tidbitmap.c](../raw/postgres-17/src/backend/nodes/tidbitmap.c#L3-L30)). `nodeBitmapHeapscan.c` requires an MVCC snapshot because the index and heap passes are decoupled ([nodeBitmapHeapscan.c](../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L1-L17)). GIN and BRIN leave `amgettuple` NULL, so bitmap scans are the only way the executor can read them ([ginutil.c:79](../raw/postgres-17/src/backend/access/gin/ginutil.c#L79), [brin.c:289](../raw/postgres-17/src/backend/access/brin/brin.c#L289)).

Related: [TID](#tid), [Index-only scan](#index-only-scan), [GIN](#gin), [BRIN](#brin), [work_mem](#work_mem)

### BKI

**Aliases:** Backend Interface, `postgres.bki`, `genbki.pl`. **Checked on:** PostgreSQL 17.

BKI is the bootstrap file format that `initdb` uses to create the system catalogs and load their first rows. The build generates `postgres.bki` from the catalog headers and `.dat` files with the Perl script `genbki.pl` ([bki.sgml:40-52](../raw/postgres-17/doc/src/sgml/bki.sgml#L40-L52), [genbki.pl:1-7](../raw/postgres-17/src/backend/catalog/genbki.pl#L1-L7)). `genbki.pl` also writes a derived `pg_*_d.h` header for each catalog, so catalog OIDs and column-number macros come from generated files rather than from hand-written code ([bki.sgml:54-61](../raw/postgres-17/doc/src/sgml/bki.sgml#L54-L61), [genbki.pl:456-475](../raw/postgres-17/src/backend/catalog/genbki.pl#L456-L475)). A separate grammar, `bootparse.y`, reads the BKI file in bootstrap mode ([bootparse.y:4-5](../raw/postgres-17/src/backend/bootstrap/bootparse.y#L4-L5)).

Related: [Catalog](#catalog), [OID](#oid), [Syscache](#syscache), [Grammar](#grammar)

### BLCKSZ

**Aliases:** block size, `block_size`. **Checked on:** PostgreSQL 17.

`BLCKSZ` is the compile-time size of one disk block and one shared buffer, 8192 bytes by default. Build configuration only accepts 1, 2, 4, 8, 16 or 32 kB, and changing it needs a new `initdb` ([configure.ac#blocksize](../raw/postgres-17/configure.ac#L258-L289)). Page-derived limits are written in terms of `BLCKSZ`. Two examples are the TOAST size limits, built by `MaximumBytesPerTuple()`, and the free space a B-tree leaves per page, `BTGetTargetPageFreeSpace()` ([heaptoast.h#MaximumBytesPerTuple](../raw/postgres-17/src/include/access/heaptoast.h#L20-L26), [nbtree.h#BTGetTargetPageFreeSpace](../raw/postgres-17/src/include/access/nbtree.h#L1144-L1145)). A running server reports the value through the read-only `block_size` setting. Its GUC context is `internal`, so no restart, reload or `SET` can change it ([guc_tables.c#block_size](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3267-L3276)).

Related: [Block](#block), [Page](#page)

### Bloat

**Checked on:** PostgreSQL 17.

Bloat is space in data pages that holds no current row versions, such as free space or outdated row versions ([glossary.sgml#glossary-bloat](../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)). For indexes, the `REINDEX` docs describe a bloated index as one with many empty or nearly-empty pages. A rebuild removes those pages by writing a new copy of the index ([ref/reindex.sgml#bloated](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L62)). The docs also say bloat in non-B-tree indexes "has not been well researched" and recommend watching their physical size ([maintenance.sgml#routine-reindex](../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046)). The wiki's v17 bloat test protocols are [Mandatory B-Tree Bloat Tests (unverified)](v17/common-concepts/mandatory-btree-bloat-tests.md), [Mandatory GIN Bloat Tests (unverified)](v17/common-concepts/mandatory-gin-bloat-tests.md) and [Mandatory Non-B-Tree, Non-GIN Bloat Tests (unverified)](v17/common-concepts/mandatory-non-btree-non-gin-bloat-tests.md).

Related: [Fillfactor](#fillfactor), [REINDEX](#reindex), [VACUUM](#vacuum), [VACUUM FULL](#vacuum-full)

### Block

**Aliases:** disk block, `BlockNumber`. **Checked on:** PostgreSQL 17.

A block is one fixed-size unit of a relation's data file, numbered from 0. It is also the unit of I/O: one shared buffer holds exactly one disk block ([block.h#BlockNumber](../raw/postgres-17/src/include/storage/block.h#L17-L35)). Source code addresses blocks with the 32-bit `BlockNumber` and uses `InvalidBlockNumber` (`0xFFFFFFFF`) as "no block" ([block.h#InvalidBlockNumber](../raw/postgres-17/src/include/storage/block.h#L31-L35)). A block is a raw unit of storage; a [page](#page) is the formatted layout that access methods write into a block ([bufpage.h#Page](../raw/postgres-17/src/include/storage/bufpage.h#L22-L28)).

Related: [BLCKSZ](#blcksz), [Page](#page), [TID](#tid)

### Bottom-up index deletion

**Checked on:** PostgreSQL 17.

Bottom-up index deletion is a B-tree step that runs just before a leaf page would split. It removes old row versions left behind by `UPDATE`s that did not change the indexed columns. The executor sends a hint that the incoming tuple is a "logically unchanged" duplicate. `_bt_bottomupdel_pass()` then asks the table AM which duplicates are safe to delete ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L557-L579), [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L285-L307)). In `_bt_delete_or_dedup_one_page()`, it runs after simple deletion and before deduplication. Unlike simple deletion, it guesses from heuristics and may find nothing to delete ([nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2781)).

Related: [Simple index deletion](#simple-index-deletion), [Deduplication](#deduplication), [Page split](#page-split), [HOT](#hot), [MVCC](#mvcc)

### BRIN

**Aliases:** Block Range Index. **Checked on:** PostgreSQL 17.

A BRIN index stores one small summary per range of consecutive table pages, such as the minimum and maximum value in that range. A query can then skip ranges that cannot match ([brin/README](../raw/postgres-17/src/backend/access/brin/README#L1-L13)). By default one range covers 128 heap pages, set by `pages_per_range` ([brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-17/src/include/access/brin.h#L39)). BRIN stores no row addresses. It therefore supports only `amgetbitmap`, which returns a lossy bitmap of whole page ranges, and the heap scan must recheck every row ([brin/README](../raw/postgres-17/src/backend/access/brin/README#L16-L23), [brin.c:289](../raw/postgres-17/src/backend/access/brin/brin.c#L289)).

Related: [Access method](#access-method), [Bitmap scan](#bitmap-scan), [Metapage](#metapage)

### Buffer manager

**Aliases:** bufmgr. **Checked on:** PostgreSQL 17.

The buffer manager keeps copies of disk blocks in shared memory. Backends read and change pages there instead of going to the file each time. Its main entry points are `ReadBuffer()` (find or load a page and pin it), `ReleaseBuffer()` (unpin it) and `MarkBufferDirty()`, which defers the disk write until buffer replacement or a checkpoint ([bufmgr.c#entry-points](../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L15-L29)). Each buffer has a `BufferDesc` header holding the page's identity (`tag`), an atomic state word with flags, pin count and usage count, and a content lock ([buf_internals.h#BufferDesc](../raw/postgres-17/src/include/storage/buf_internals.h#L245-L256)).

Related: [Clock sweep](#clock-sweep), [Dirty buffer](#dirty-buffer), [Ring buffer](#ring-buffer), [shared_buffers](#shared_buffers)

### Catalog

**Aliases:** system catalog, `pg_catalog`. **Checked on:** PostgreSQL 17.

A system catalog is an ordinary table in which PostgreSQL records its own metadata: tables, indexes, types, functions, and so on ([glossary.sgml#glossary-system-catalog](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1772-L1788)). Backend C code knows each catalog's row layout and reads and writes it directly ([bki.sgml:6-15](../raw/postgres-17/doc/src/sgml/bki.sgml#L6-L15)). Each catalog is declared in a header under `src/include/catalog/` with the `CATALOG()` macro ([bki.sgml:6-24](../raw/postgres-17/doc/src/sgml/bki.sgml#L6-L24), [genbki.h:23](../raw/postgres-17/src/include/catalog/genbki.h#L23)). The SQL standard uses "catalog" for what PostgreSQL calls a database. The wiki always means the system-catalog sense ([glossary.sgml#glossary-catalog](../raw/postgres-17/doc/src/sgml/glossary.sgml#L314-L326)).

Related: [pg_class](#pg_class), [pg_index](#pg_index), [BKI](#bki), [Syscache](#syscache), [Relcache](#relcache), [OID](#oid)

### Checkpoint

**Checked on:** PostgreSQL 17.

A point in the WAL at which every data-file change made before it is guaranteed to be on disk. It is also the act of flushing dirty pages to reach that point ([glossary.sgml#checkpoint](../raw/postgres-17/doc/src/sgml/glossary.sgml#L353-L376)). Crash recovery starts replay from the latest checkpoint's redo record, so older WAL segments can be recycled ([wal.sgml](../raw/postgres-17/doc/src/sgml/wal.sgml#L495-L509)). In source, the work is `CreateCheckPoint()`. An online checkpoint writes an `XLOG_CHECKPOINT_REDO` record first and an `XLOG_CHECKPOINT_ONLINE` record when done. A shutdown checkpoint writes one `XLOG_CHECKPOINT_SHUTDOWN` record ([xlog.c#CreateCheckPoint](../raw/postgres-17/src/backend/access/transam/xlog.c#L6827-L6862)). `checkpoint_timeout` is a `sighup` GUC ([guc_tables.c#checkpoint_timeout](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2855-L2862)).

Related: [Crash recovery](#crash-recovery), [Full-page image](#full-page-image), [WAL](#wal), [Background writer](#background-writer)

### Clock sweep

**Aliases:** clock-sweep, `usage_count`, `nextVictimBuffer`. **Checked on:** PostgreSQL 17.

Clock sweep is the algorithm that chooses which shared buffer to reuse when no free buffer is left. A "clock hand" (`nextVictimBuffer`) moves around all buffers. It skips a pinned buffer, lowers a nonzero usage count by one, and takes the first unpinned buffer whose count is zero ([buffer/README#clock-sweep](../raw/postgres-17/src/backend/storage/buffer/README#L166-L199)). Code: `ClockSweepTick()` advances the hand and `StrategyGetBuffer()` runs the loop ([freelist.c#ClockSweepTick](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L103-L125), [freelist.c#StrategyGetBuffer](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L316-L332)). Each pin raises the usage count up to `BM_MAX_USAGE_COUNT` (5) ([buffer/README#clock-sweep](../raw/postgres-17/src/backend/storage/buffer/README#L171-L174)). The cap keeps the sweep short at the cost of only approximating LRU ([buf_internals.h#BM_MAX_USAGE_COUNT](../raw/postgres-17/src/include/storage/buf_internals.h#L71-L79)).

Related: [Buffer manager](#buffer-manager), [Ring buffer](#ring-buffer), [shared_buffers](#shared_buffers)

### CLUSTER

**Checked on:** PostgreSQL 17.

`CLUSTER` rewrites a table into a new physical file in the order of one of its indexes, then rebuilds the indexes. The table keeps its OID because only the relfilenumbers are swapped ([cluster.c#cluster_rel](../raw/postgres-17/src/backend/commands/cluster.c#L293-L311)). In source, `rebuild_relation` calls `make_new_heap`, `copy_table_data`, and `finish_heap_swap`, and holds `AccessExclusiveLock` on the table until commit ([cluster.c#rebuild_relation](../raw/postgres-17/src/backend/commands/cluster.c#L624-L674), [cluster.c:337](../raw/postgres-17/src/backend/commands/cluster.c#L337)). The same code path implements [VACUUM FULL](#vacuum-full) when no index is given ([cluster.c:1-4](../raw/postgres-17/src/backend/commands/cluster.c#L1-L4)).

Related: [VACUUM FULL](#vacuum-full), [Table rewrite](#table-rewrite), [Relfilenumber](#relfilenumber), [AccessExclusiveLock](#accessexclusivelock)

### COMMENT ON

**Aliases:** `pg_description`, object comment. **Checked on:** PostgreSQL 17.

`COMMENT ON` attaches a free-text description to a database object. `CreateComments` stores it as one `pg_description` row, and an empty string removes the comment ([comment.c#CreateComments](../raw/postgres-17/src/backend/commands/comment.c#L143-L170)). The row is keyed by the object's OID, the OID of the catalog that holds the object, and a sub-ID that is the column number for column comments and 0 otherwise ([pg_description.h:10-19](../raw/postgres-17/src/include/catalog/pg_description.h#L10-L19), [pg_description.h#FormData_pg_description](../raw/postgres-17/src/include/catalog/pg_description.h#L48-L57)). `REINDEX CONCURRENTLY` builds an index with a new OID, so `index_concurrently_swap` moves the old index's comment to it ([index.c:1740-1782](../raw/postgres-17/src/backend/catalog/index.c#L1740-L1782)).

Related: [OID](#oid), [REINDEX](#reindex), [CONCURRENTLY](#concurrently), [Catalog](#catalog)

### CONCURRENTLY

**Aliases:** CIC (`CREATE INDEX CONCURRENTLY`), RIC (`REINDEX CONCURRENTLY`), `DROP INDEX CONCURRENTLY`. **Checked on:** PostgreSQL 17.

`CONCURRENTLY` is an option that builds, rebuilds or drops an index without blocking writes to the table. It costs more total work: two table scans and waits for older transactions to finish ([ref/create_index.sgml](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L612-L643), [ref/reindex.sgml](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L367-L380)). In source, `DefineIndex()` takes `ShareUpdateExclusiveLock` instead of `ShareLock` when the option is set ([indexcmds.c:678](../raw/postgres-17/src/backend/commands/indexcmds.c#L678)). `index_set_state_flags()` moves the index through the `pg_index` states in separate transactions ([index.c#index_set_state_flags](../raw/postgres-17/src/backend/catalog/index.c#L3469-L3503)). `ReindexRelationConcurrently()` implements `REINDEX CONCURRENTLY`, which first adds a new transient index definition to `pg_index` to replace the old index ([indexcmds.c#ReindexRelationConcurrently](../raw/postgres-17/src/backend/commands/indexcmds.c#L3428-L3451), [ref/reindex.sgml](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L382-L394)). A failed concurrent build leaves an invalid index behind ([ref/create_index.sgml](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L650)).

Related: [Invalid index](#invalid-index), [REINDEX](#reindex), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [Snapshot](#snapshot), [pg_index](#pg_index)

### Constraint exclusion

**Checked on:** PostgreSQL 17.

Constraint exclusion is a planner check that skips a table when its `CHECK` constraints prove that no row can match the query's `WHERE` clauses [plancat.c#relation_excluded_by_constraints](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1576-L1600). The `constraint_exclusion` GUC controls it. Its context is `user`, so a session or transaction can `SET` it with no reload or restart. Its default `partition` applies the test only to appendrel members, such as inheritance children [guc_tables.c#constraint_exclusion](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4798-L4807) [plancat.c:1625-1641](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1625-L1641). Do not confuse it with partition pruning. Declarative partitions are pruned first from partition bounds, so the `partition` mode does not re-test their partition constraints [plancat.c:1631-1638](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1631-L1638).

Related: [Partition pruning](#partition-pruning), [Planner](#planner), [RelOptInfo](#reloptinfo)

### Contrib

**Aliases:** `contrib/`, additional supplied modules. **Checked on:** PostgreSQL 17.

Contrib is the source directory of optional modules that ship with PostgreSQL but are not part of the core server ([contrib.sgml:7-15](../raw/postgres-17/doc/src/sgml/contrib.sgml#L7-L15), [README:1-9](../raw/postgres-17/contrib/README#L1-L9)). Most contrib modules are [extensions](#extension): they ship a `.control` file and are installed per database with `CREATE EXTENSION` ([README:19-25](../raw/postgres-17/contrib/README#L19-L25), [pageinspect.control](../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L5)). Some are only loadable libraries. For example, `auto_explain` is loaded with `LOAD` rather than `CREATE EXTENSION` ([auto-explain.sgml:18-23](../raw/postgres-17/doc/src/sgml/auto-explain.sgml#L18-L23)). A reader should not treat "contrib module" and "extension" as the same thing.

Related: [Extension](#extension), [pageinspect](#pageinspect), [pgstattuple](#pgstattuple), [pg_freespacemap](#pg_freespacemap), [Hook](#hook)

### Cost

**Checked on:** PostgreSQL 17.

A cost is the planner's estimate of how expensive a plan step is, in arbitrary units anchored to `seq_page_cost` (one sequential page read) [costsize.c header](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6-L30). Every path carries a `startup_cost`, spent before the first row, and a `total_cost`, spent to fetch all rows [pathnodes.h#Path](../raw/postgres-17/src/include/nodes/pathnodes.h#L1662-L1665) [costsize.c:37-46](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L37-L46). The C type `Cost` is a `double` [nodes.h:251](../raw/postgres-17/src/include/nodes/nodes.h#L251). The base parameters (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, `cpu_index_tuple_cost`, `cpu_operator_cost`) are globals in `costsize.c` [costsize.c:119-128](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L119-L128). Each index access method prices its own scans through its `amcostestimate` callback [costsize.c#cost_index](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L612-L622).

Related: [Path](#path), [Planner](#planner), [Selectivity](#selectivity), [effective_cache_size](#effective_cache_size)

### Crash recovery

**Aliases:** WAL replay, redo. **Checked on:** PostgreSQL 17.

The startup step that replays WAL from the last checkpoint's redo point after an unclean shutdown. It re-applies every change that may not have reached the data files ([wal.sgml](../raw/postgres-17/doc/src/sgml/wal.sgml#L502-L505)). `PerformWalRecovery()` drives it, and it never runs after a clean shutdown ([xlogrecovery.c#PerformWalRecovery](../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L1656-L1662)). Each record goes to its resource manager's `rm_redo` callback ([xlogrecovery.c#ApplyWalRecord](../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L1997-L2001)). Redo compares the page LSN with the record's position to skip changes already applied ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L420-L422)).

Related: [Checkpoint](#checkpoint), [Full-page image](#full-page-image), [LSN](#lsn), [WAL](#wal)

### Critical section

**Checked on:** PostgreSQL 17.

A stretch of backend code in which any error must crash the server rather than be handled normally. `START_CRIT_SECTION()` and `END_CRIT_SECTION()` just raise and lower the counter `CritSectionCount` ([miscadmin.h#START_CRIT_SECTION](../raw/postgres-17/src/include/miscadmin.h#L150-L156)). While that counter is non-zero, `errstart()` promotes any `ERROR` to `PANIC` ([elog.c#errstart](../raw/postgres-17/src/backend/utils/error/elog.c#L356-L364)). WAL-logged page changes are wrapped in one. Shared buffers then hold changes that are not yet logged, and an ordinary error must not let those changes reach disk ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L442-L446)). It is not a lock. The macros only change a counter in the current backend, so other processes are not blocked ([miscadmin.h#START_CRIT_SECTION](../raw/postgres-17/src/include/miscadmin.h#L150-L156)).

Related: [WAL](#wal), [ereport](#ereport)

### Cumulative statistics

**Aliases:** cumulative statistics system, pgstat, `pg_stat_*` views. **Checked on:** PostgreSQL 17.

The cumulative statistics system counts server activity: table and index accesses, row counts, and vacuum and analyze runs ([monitoring.sgml#monitoring-stats](../raw/postgres-17/doc/src/sgml/monitoring.sgml#L137-L145)). In `pgstat.c`, each process accumulates counters locally as pending entries and later flushes them to shared memory through `pgstat_report_stat()`. The startup process loads the stats from disk, and the checkpointer writes them at shutdown ([pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L1-L16), [pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L44-L50)). The `track_counts` GUC switches collection and has context `PGC_SUSET`, so a superuser can change it per session ([guc_tables.c#track_counts](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1412-L1419)). Do not confuse these activity counters with the planner's column statistics. The live view of what processes are doing right now is also a separate facility ([monitoring.sgml#monitoring-stats](../raw/postgres-17/doc/src/sgml/monitoring.sgml#L147-L152)).

Related: [Statistics](#statistics), [Wait event](#wait-event), [GUC context](#guc-context)

### Deduplication

**Aliases:** B-tree deduplication, `deduplicate_items`. **Checked on:** PostgreSQL 17.

Deduplication merges several B-tree leaf entries that have the same key into one posting-list tuple that holds many row addresses. This saves space and delays page splits ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L904-L920)). It runs lazily, only when a leaf page is about to split. It is the last step in `_bt_delete_or_dedup_one_page()`, and it runs only when the `deduplicate_items` storage parameter is on (the default) and the index's `allequalimage` flag is true ([nbtinsert.c:2778-2781](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2778-L2781), [nbtree.h#BTGetDeduplicateItems](../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150)). `_bt_dedup_pass()` in `nbtdedup.c` does the merging ([nbtdedup.c#_bt_dedup_pass](../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L36-L58)). Do not confuse it with bottom-up deletion, which removes entries instead of merging them.

Related: [Posting list](#posting-list), [allequalimage](#allequalimage), [Bottom-up index deletion](#bottom-up-index-deletion), [Page split](#page-split)

### Dirty buffer

**Aliases:** `BM_DIRTY`. **Checked on:** PostgreSQL 17.

A dirty buffer is a shared buffer whose page was changed in memory but has not yet been written to its data file. `MarkBufferDirty()` sets this state; the caller must hold a pin and an exclusive content lock, and the actual write happens later ([bufmgr.c#MarkBufferDirty](../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2540-L2551)). The flag is `BM_DIRTY` in the buffer state word ([buf_internals.h:61](../raw/postgres-17/src/include/storage/buf_internals.h#L61)). A dirty buffer must be written before clock sweep can recycle it ([buffer/README#clock-sweep](../raw/postgres-17/src/backend/storage/buffer/README#L198-L199)). The background writer looks ahead of the clock hand for dirty, unpinned, zero-usage buffers to write early ([buffer/README#bgwriter](../raw/postgres-17/src/backend/storage/buffer/README#L252-L257)).

Related: [Background writer](#background-writer), [Checkpoint](#checkpoint), [Clock sweep](#clock-sweep), [WAL](#wal)

### effective_cache_size

**Checked on:** PostgreSQL 17.

`effective_cache_size` tells the planner how much data it may assume is cached, counting both shared buffers and the kernel cache. It is measured in pages, and it allocates no memory [costsize.c:22-25](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L22-L25) [guc_tables.c#effective_cache_size](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3509-L3518). Its context is `user`, so a session or transaction can `SET` it with no reload or restart. Its default is 524288 pages [cost.h:34](../raw/postgres-17/src/include/optimizer/cost.h#L34). Its main consumer is `index_pages_fetched()`. That function gives each table a pro-rated share of the cache and applies the Mackert and Lohman formula to estimate repeated page fetches [costsize.c#index_pages_fetched](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L897-L928).

Related: [Cost](#cost), [shared_buffers](#shared_buffers), [Planner](#planner)

### ereport

**Aliases:** `elog`, error level, `ERROR`, `FATAL`, `PANIC`. **Checked on:** PostgreSQL 17.

`ereport` and the shorter `elog` are the macros that server code uses to log a message or raise an error. The first argument is a severity level that runs from `DEBUG5` through `LOG`, `WARNING`, `ERROR`, and `FATAL` up to `PANIC` ([elog.h:25-56](../raw/postgres-17/src/include/utils/elog.h#L25-L56), [elog.h#ereport](../raw/postgres-17/src/include/utils/elog.h#L163-L164), [elog.h#elog](../raw/postgres-17/src/include/utils/elog.h#L239-L240)). The level decides what happens next. `ERROR` long-jumps to the current error handler and aborts the transaction, `FATAL` ends the process, and `PANIC` calls `abort()`, and the postmaster then kills the other backends too ([elog.c#errfinish](../raw/postgres-17/src/backend/utils/error/elog.c#L515-L543), [elog.c:594-604](../raw/postgres-17/src/backend/utils/error/elog.c#L594-L604)). When source code shows an `ereport(ERROR, ...)`, execution never continues on the next line.

Related: [Critical section](#critical-section), [Memory context](#memory-context), [Backend](#backend)

### Event trigger

**Aliases:** `pg_event_trigger`, `ddl_command_start`, `ddl_command_end`, `sql_drop`, `table_rewrite`. **Checked on:** PostgreSQL 17.

An event trigger is a database-wide trigger that fires on DDL events rather than on row changes to one table ([event-trigger.sgml:10-16](../raw/postgres-17/doc/src/sgml/event-trigger.sgml#L10-L16)). Its events are `login`, `ddl_command_start`, `ddl_command_end`, `table_rewrite`, and `sql_drop` ([event-trigger.sgml:27-35](../raw/postgres-17/doc/src/sgml/event-trigger.sgml#L27-L35)). Each trigger is a `pg_event_trigger` row naming the event and the function to call ([pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-17/src/include/catalog/pg_event_trigger.h#L29-L42)). `ProcessUtilitySlow` fires the start event before it dispatches a command, and fires the drop and end events after the command runs ([utility.c:1106-1113](../raw/postgres-17/src/backend/tcop/utility.c#L1106-L1113), [utility.c:1943-1946](../raw/postgres-17/src/backend/tcop/utility.c#L1943-L1946)). An event trigger is a SQL-level object. A [hook](#hook), by contrast, is a C function pointer that a loaded library sets.

Related: [Hook](#hook), [Utility command](#utility-command), [Catalog](#catalog)

### Executor

**Checked on:** PostgreSQL 17.

The executor runs a finished plan and produces its rows. Callers drive it through four entry points: `ExecutorStart`, `ExecutorRun`, `ExecutorFinish`, and `ExecutorEnd` [execMain.c header](../raw/postgres-17/src/backend/executor/execMain.c#L1-L28). It takes a `PlannedStmt` as input and never looks at the `Query` tree [plannodes.h#PlannedStmt](../raw/postgres-17/src/include/nodes/plannodes.h#L31-L35) [parsenodes.h#Query](../raw/postgres-17/src/include/nodes/parsenodes.h#L101-L110). Utility commands skip the executor and go to `ProcessUtility` instead [utility.c#ProcessUtility](../raw/postgres-17/src/backend/tcop/utility.c#L498-L525).

Related: [PlannedStmt](#plannedstmt), [Portal](#portal), [Planner](#planner), [Utility command](#utility-command)

### EXPLAIN BUFFERS

**Checked on:** PostgreSQL 17.

`BUFFERS` is an `EXPLAIN` option that counts the blocks each plan node touched: shared, local, and temporary blocks that were hit, read, dirtied, or written [ref/explain.sgml#BUFFERS](../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L180-L206). A hit means the block was already in cache. Read time and write time appear only when `track_io_timing` is on, and the option defaults to off in v17 [ref/explain.sgml#BUFFERS](../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L183-L206). The counters live in `BufferUsage`, and `show_buffer_usage()` prints only non-zero values in text format [instrument.h#BufferUsage](../raw/postgres-17/src/include/executor/instrument.h#L24-L42) [commands/explain.c#show_buffer_usage](../raw/postgres-17/src/backend/commands/explain.c#L3739-L3785).

Related: [EXPLAIN](#explain), [Buffer manager](#buffer-manager), [shared_buffers](#shared_buffers), [Dirty buffer](#dirty-buffer)

### EXPLAIN

**Checked on:** PostgreSQL 17.

`EXPLAIN` shows the plan the planner chose, with its estimated costs and row counts. With `ANALYZE`, it also runs the statement and reports what actually happened [commands/explain.c#ExplainQuery](../raw/postgres-17/src/backend/commands/explain.c#L175-L207). `ExplainQuery()` parses the option list into an `ExplainState`. It rejects `TIMING` and `SERIALIZE` without `ANALYZE`, and it rejects `GENERIC_PLAN` combined with `ANALYZE` [commands/explain.c:284-305](../raw/postgres-17/src/backend/commands/explain.c#L284-L305). `NewExplainState()` turns only `COSTS` on by default [commands/explain.c#NewExplainState](../raw/postgres-17/src/backend/commands/explain.c#L370-L377).

Related: [EXPLAIN BUFFERS](#explain-buffers), [Planner](#planner), [Cost](#cost)

### Expression index

**Aliases:** index on expressions, functional index. **Checked on:** PostgreSQL 17.

An expression index stores the result of a function or expression, such as `lower(col1)`, instead of a plain column value. A query can use it when its `WHERE` clause uses the same expression ([indices.sgml#indexes-expressional](../raw/postgres-17/doc/src/sgml/indices.sgml#L732-L760)). In the catalog, each expression column appears as a zero in `pg_index.indkey`, and its expression tree is stored in `pg_index.indexprs` ([pg_index.h#FormData_pg_index](../raw/postgres-17/src/include/catalog/pg_index.h#L48-L61)). An expression index is not a partial index. The first changes what is indexed; the second changes which rows are indexed.

Related: [Partial index](#partial-index), [pg_index](#pg_index), [Planner](#planner)

### Extension

**Aliases:** `CREATE EXTENSION`, control file. **Checked on:** PostgreSQL 17.

An extension is a named package of SQL objects that PostgreSQL installs, tracks, and drops as a unit. It consists of a script file, a control file, and often a shared library of C code ([extend.sgml:525-540](../raw/postgres-17/doc/src/sgml/extend.sgml#L525-L540), [glossary.sgml#glossary-extension](../raw/postgres-17/doc/src/sgml/glossary.sgml#L693-L705)). `CreateExtension` parses the control file into an `ExtensionControlFile` struct. Fields such as `superuser` and `trusted` decide who may install it ([extension.c#ExtensionControlFile](../raw/postgres-17/src/backend/commands/extension.c#L79-L96), [extension.c#CreateExtension](../raw/postgres-17/src/backend/commands/extension.c#L1898)). When the library loads, its `_PG_init()` function runs, which is where extensions usually install [hooks](#hook) ([dfmgr.c:284-289](../raw/postgres-17/src/backend/utils/fmgr/dfmgr.c#L284-L289)).

Related: [Contrib](#contrib), [Hook](#hook), [fmgr](#fmgr)

### Fast root

**Aliases:** `btm_fastroot`, `btm_fastlevel`. **Checked on:** PostgreSQL 17.

The fast root is the lowest B-tree level that has only one page. Searches start there instead of at the true root, which skips useless single-page levels after mass deletions ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L366-L381)). The metapage stores both roots ([nbtree.h#BTMetaPageData](../raw/postgres-17/src/include/access/nbtree.h#L103-L110)). `_bt_getrootheight()` returns the fast root's level ([nbtpage.c#_bt_getrootheight](../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L664-L676)). The planner stores that value in `IndexOptInfo.tree_height` ([plancat.c:488-494](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L494)). `btcostestimate()` then charges `(tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost` per descent, and that multiplier is 50.0 ([selfuncs.c:7093-7106](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106), [selfuncs.c:145](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).

Related: [B-tree](#b-tree), [Metapage](#metapage), [IndexOptInfo](#indexoptinfo), [Cost](#cost)

### Fillfactor

**Checked on:** PostgreSQL 17.

Fillfactor is a per-table or per-index storage parameter: the percentage of a page to fill before starting a new page. For a table, the leftover space gives `UPDATE` room to put the new row version on the same page, which makes HOT updates more likely ([ref/create_table.sgml#fillfactor](../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1468-L1476)). The heap default is 100, and the minimum is 10 ([rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-17/src/include/utils/rel.h#L348-L349)). Changing it takes `ShareUpdateExclusiveLock` and affects only later inserts ([reloptions.c#fillfactor](../raw/postgres-17/src/backend/access/common/reloptions.c#L174-L194)). B-tree leaf pages default to 90. B-tree applies fillfactor during index builds and rightmost page splits, uses a fixed 70 above the leaf level, and uses 96 when it splits a page full of one duplicate value ([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../raw/postgres-17/src/include/access/nbtree.h#L189-L202)).

Related: [B-tree](#b-tree), [Bloat](#bloat), [HOT](#hot), [Page split](#page-split)

### fmgr

**Aliases:** function manager, `FmgrInfo`, `FunctionCallInfo`, `PG_FUNCTION_ARGS`, `Gen_fmgrtab.pl`. **Checked on:** PostgreSQL 17.

fmgr, the function manager, is the calling convention through which the server calls SQL-callable C functions ([fmgr.h:3-8](../raw/postgres-17/src/include/fmgr.h#L3-L8)). Every such function takes one `FunctionCallInfo` argument, which the `PG_FUNCTION_ARGS` macro declares. It returns a `Datum` ([fmgr.h:38-40](../raw/postgres-17/src/include/fmgr.h#L38-L40), [fmgr.h#FunctionCallInfoBaseData](../raw/postgres-17/src/include/fmgr.h#L85-L95), [fmgr.h:193](../raw/postgres-17/src/include/fmgr.h#L193)). `fmgr_info` resolves a function OID into an `FmgrInfo` that holds the C address and flags such as strictness ([fmgr.c#fmgr_info](../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L113-L130), [fmgr.h#FmgrInfo](../raw/postgres-17/src/include/fmgr.h#L56-L67)). The build generates the built-in function table and the `F_*` OID macros from `pg_proc.dat` with `Gen_fmgrtab.pl` ([Gen_fmgrtab.pl:1-6](../raw/postgres-17/src/backend/utils/Gen_fmgrtab.pl#L1-L6)).

Related: [OID](#oid), [Extension](#extension), [BKI](#bki)

### Fork

**Aliases:** relation fork, `ForkNumber`, main fork. **Checked on:** PostgreSQL 17.

A fork is one of the separate file sets that make up a single relation's storage. The main fork holds the data. The free space map and visibility map are secondary forks. Unlogged relations also have an init fork ([glossary.sgml#glossary-fork](../raw/postgres-17/doc/src/sgml/glossary.sgml#L799-L811)). In source, the `ForkNumber` enum names them `MAIN_FORKNUM`, `FSM_FORKNUM`, `VISIBILITYMAP_FORKNUM` and `INIT_FORKNUM`. Buffer and storage calls such as `ReadBufferExtended()` take one of these values ([relpath.h#ForkNumber](../raw/postgres-17/src/include/common/relpath.h#L47-L62), [bufmgr.c#ReadBufferExtended](../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L799-L800)).

Related: [Free space map](#free-space-map), [Relfilenumber](#relfilenumber), [Visibility map](#visibility-map)

### Free space map

**Aliases:** FSM, `FSM_FORKNUM`. **Checked on:** PostgreSQL 17.

The free space map records roughly how much free space each page of a relation has, so an insert can find a page with room without scanning the table. It lives in its own fork ([glossary.sgml#glossary-fsm](../raw/postgres-17/doc/src/sgml/glossary.sgml#L814-L828)). It stores one byte per page, as one of 256 space categories ([freespace.c#NOTES](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L14-L50)). Code: `GetPageWithFreeSpace()` finds a page, `RecordPageWithFreeSpace()` updates one, and `FreeSpaceMapVacuum()` updates the upper levels of the map ([freespace.c#GetPageWithFreeSpace](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L137), [freespace.c#FreeSpaceMapVacuum](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L368)). Do not confuse it with the visibility map, a different fork that tracks tuple visibility per page ([glossary.sgml#glossary-vm](../raw/postgres-17/doc/src/sgml/glossary.sgml#L2093-L2105)).

Related: [Fork](#fork), [pg_freespacemap](#pg_freespacemap), [Visibility map](#visibility-map)

### Freezing

**Checked on:** PostgreSQL 17.

VACUUM's marking of old row versions as frozen. A frozen row version counts as older than every normal xid, so it stays "in the past" after the xid counter wraps ([maintenance.sgml#vacuum-for-wraparound](../raw/postgres-17/doc/src/sgml/maintenance.sgml#L440-L456)). The reserved `FrozenTransactionId` (2) marks very old tuples ([transam.h](../raw/postgres-17/src/include/access/transam.h#L20-L35)). Modern heap code sets the `HEAP_XMIN_FROZEN` infomask bits instead of overwriting `t_xmin` ([htup_details.h:206](../raw/postgres-17/src/include/access/htup_details.h#L206)). `heap_prepare_freeze_tuple()` decides which xmin, xmax and xvac fields of a tuple are old enough to freeze, and it builds a freeze plan for the page ([heapam.c#heap_prepare_freeze_tuple](../raw/postgres-17/src/backend/access/heap/heapam.c#L7371-L7415)). `vacuum_freeze_min_age` is a `user` GUC, so it can be set per session ([guc_tables.c#vacuum_freeze_min_age](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2666-L2672)).

Related: [Transaction ID](#transaction-id), [Wraparound](#wraparound), [xmin and xmax](#xmin-and-xmax), [VACUUM](#vacuum), [MultiXact](#multixact)

### Full-page image

**Aliases:** FPI, full-page write, backup block. **Checked on:** PostgreSQL 17.

A copy of a whole data page stored inside a WAL record. Replay restores the copy instead of redoing a small change, which guards against pages left half-written by a crash ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L424-L435)). Only the first change to a page after a checkpoint carries one. `XLogRecordAssemble()` adds the image when the page LSN is at or before `RedoRecPtr`, unless `REGBUF_NO_IMAGE` is set or page writes are off ([xloginsert.c#XLogRecordAssemble](../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L620)). `full_page_writes` is a `sighup` GUC ([guc_tables.c#full_page_writes](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1157-L1167)).

Related: [Checkpoint](#checkpoint), [LSN](#lsn), [WAL](#wal)

### GIN

**Aliases:** Generalized Inverted Index. **Checked on:** PostgreSQL 17.

GIN is an inverted index. It maps each key found inside a value, such as an array element or a text-search lexeme, to the list of rows that contain it ([gin/README](../raw/postgres-17/src/backend/access/gin/README#L8-L26)). A GIN index has a metapage and a B-tree of key entries. A key's row list is stored either inline as a posting list or as a separate posting tree. If `fastupdate` is on, the index can also have pending-list pages ([gin/README](../raw/postgres-17/src/backend/access/gin/README#L94-L105)). GIN sets `amgettuple` and `amcanreturn` to NULL, so it supports neither plain index scans nor index-only scans ([ginutil.c:70-79](../raw/postgres-17/src/backend/access/gin/ginutil.c#L70-L79)).

Related: [Posting list](#posting-list), [Posting tree](#posting-tree), [Pending list](#pending-list), [Bitmap scan](#bitmap-scan), [Access method](#access-method)

### GiST

**Aliases:** Generalized Search Tree. **Checked on:** PostgreSQL 17.

GiST is a balanced tree framework for building custom index types, such as R-trees for geometric data. The operator class decides what each key means ([gist.sgml#gist-intro](../raw/postgres-17/doc/src/sgml/gist.sgml#L11-L25), [gist/README](../raw/postgres-17/src/backend/access/gist/README#L1-L25)). In PostgreSQL 17, `gisthandler()` marks GiST as supporting ordered nearest-neighbor searches and multicolumn keys, but not unique indexes ([gist.c#gisthandler](../raw/postgres-17/src/backend/access/gist/gist.c#L59-L70)). GiST has no metapage: block 0 is the root ([gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-17/src/include/access/gist_private.h#L262)).

Related: [SP-GiST](#sp-gist), [Access method](#access-method), [Metapage](#metapage)

### Grammar

**Aliases:** `gram.y`, `scan.l`, raw parser, bison, flex. **Checked on:** PostgreSQL 17.

The grammar is the bison source `gram.y`. It turns the tokens produced by the flex scanner `scan.l` into a raw [parse tree](#parse-tree) ([parser/README:1-14](../raw/postgres-17/src/backend/parser/README#L1-L14), [gram.y:6-7](../raw/postgres-17/src/backend/parser/gram.y#L6-L7)). `raw_parser` is the entry point that runs this lexical and grammatical analysis ([parser.c#raw_parser](../raw/postgres-17/src/backend/parser/parser.c#L34-L42)). The build generates `scan.c`, `gram.c`, and `gram.h` from these files, so the checkout contains no `gram.c` ([parser/meson.build:30-41](../raw/postgres-17/src/backend/parser/meson.build#L30-L41), [meson.build#bison_kw](../raw/postgres-17/meson.build#L353-L356)). The grammar must not access the database. Name lookup and semantic checks happen later, during parse analysis ([gram.y:25-30](../raw/postgres-17/src/backend/parser/gram.y#L25-L30)).

Related: [Parse tree](#parse-tree), [Utility command](#utility-command), [BKI](#bki)

### GUC context

**Aliases:** `GucContext`, `PGC_POSTMASTER`, `PGC_SIGHUP`, `PGC_USERSET`. **Checked on:** PostgreSQL 17.

A GUC's context says when and by whom the setting can be changed. The seven values are `PGC_INTERNAL`, `PGC_POSTMASTER`, `PGC_SIGHUP`, `PGC_SU_BACKEND`, `PGC_BACKEND`, `PGC_SUSET`, and `PGC_USERSET` ([guc.h#GucContext](../raw/postgres-17/src/include/utils/guc.h#L36-L76)). `pg_settings.context` shows them as `internal`, `postmaster`, `sighup`, `superuser-backend`, `backend`, `superuser`, and `user` ([guc_tables.c#GucContext_Names](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L641-L650)). The wiki maps them to actions: `postmaster` needs a restart, `sighup` needs a reload, and the backend, superuser, and user contexts apply per session or per transaction ([system-views.sgml:3346-3438](../raw/postgres-17/doc/src/sgml/system-views.sgml#L3346-L3438)). For example, `shared_buffers` is `PGC_POSTMASTER` and `work_mem` is `PGC_USERSET` ([guc_tables.c:2262](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2262), [guc_tables.c:2448](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2448)).

Related: [GUC](#guc), [Postmaster](#postmaster)

### GUC

**Aliases:** Grand Unified Configuration, configuration parameter, setting, `pg_settings`. **Checked on:** PostgreSQL 17.

A GUC is one server configuration parameter, such as `shared_buffers` or `work_mem`. The name comes from the "Grand Unified Configuration" scheme ([guc_tables.c:1-10](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1-L10)). Each parameter is one entry in a typed array: `ConfigureNamesBool`, `ConfigureNamesInt`, `ConfigureNamesReal`, `ConfigureNamesString`, or `ConfigureNamesEnum`. An entry records the name, the [GUC context](#guc-context), the C variable, the default, and the limits ([guc_tables.c:781](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L781), [guc_tables.c#work_mem](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2456)). The `pg_settings` view shows every GUC's current value and context ([system_views.sql#pg_settings](../raw/postgres-17/src/backend/catalog/system_views.sql#L608-L609)).

Related: [GUC context](#guc-context), [shared_buffers](#shared_buffers), [work_mem](#work_mem), [maintenance_work_mem](#maintenance_work_mem)

### Hash index

**Checked on:** PostgreSQL 17.

A hash index places each entry in a bucket chosen by hashing the key. It supports only equality lookups. Buckets are split one at a time as the index grows, and a bucket overflows into chained "overflow pages" ([hash/README](../raw/postgres-17/src/backend/access/hash/README#L13-L28)). Entries store only the 32-bit hash code, not the key value, and each page keeps them sorted by hash code ([hash/README](../raw/postgres-17/src/backend/access/hash/README#L36-L42)). `hashhandler()` marks hash indexes as single-column only, with no uniqueness and no ordering, and block 0 is the metapage ([hash.c#hashhandler](../raw/postgres-17/src/backend/access/hash/hash.c#L57-L68), [hash.h#HASH_METAPAGE](../raw/postgres-17/src/include/access/hash.h#L198)).

Related: [Access method](#access-method), [Metapage](#metapage)

### Heap

**Aliases:** heap table, heap access method, `heapam`. **Checked on:** PostgreSQL 17.

The heap is the storage that holds a table's rows. It lives in the main fork of the table's files ([glossary.sgml#glossary-heap](../raw/postgres-17/doc/src/sgml/glossary.sgml#L875-L886)). "Heap" is also the name of the built-in table access method. Its catalog row points to the handler `heap_tableam_handler`, which returns the `heapam_methods` callback table ([pg_am.dat#heap](../raw/postgres-17/src/include/catalog/pg_am.dat#L15-L17), [heapam_handler.c#heap_tableam_handler](../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L2652-L2662)). The heap keeps no row order: even after `CLUSTER`, new and updated rows are not placed in index order ([ref/cluster.sgml#description](../raw/postgres-17/doc/src/sgml/ref/cluster.sgml#L45-L49)). An index maps key values to the TIDs of row versions in the heap ([indexam.sgml#intro](../raw/postgres-17/doc/src/sgml/indexam.sgml#L38-L43)).

Related: [Access method](#access-method), [Page](#page), [TID](#tid), [Tuple](#tuple)

### Heavyweight lock

**Aliases:** regular lock, lmgr lock, table-level lock. **Checked on:** PostgreSQL 17.

The lock manager used for all user-driven locks, such as locks on tables and other database objects. It has eight modes with table-driven conflicts, detects deadlocks, and releases everything at transaction end ([lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L32-L35), [lockdefs.h](../raw/postgres-17/src/include/storage/lockdefs.h#L34-L47), [lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)). A `LOCKTAG` names the locked object ([lock.h#LOCKTAG](../raw/postgres-17/src/include/storage/lock.h#L164-L172)). Wrappers such as `LockRelationOid()` call `LockAcquireExtended()` ([lmgr.c#LockRelationOid](../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L102-L116)). Unlike an LWLock, it has deadlock detection and is held until transaction end ([lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L20-L35)).

Related: [AccessExclusiveLock](#accessexclusivelock), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [LWLock](#lwlock)

### Hook

**Aliases:** `ProcessUtility_hook`, `planner_hook`, `_PG_init`. **Checked on:** PostgreSQL 17.

A hook is a global function-pointer variable that core code checks at a fixed point. When a loaded library has set the pointer, core calls the library's function in place of the standard routine. For example, `ProcessUtility` calls `ProcessUtility_hook` when it is set and `standard_ProcessUtility` otherwise ([utility.c:513-525](../raw/postgres-17/src/backend/tcop/utility.c#L513-L525), [utility.h:71-78](../raw/postgres-17/src/include/tcop/utility.h#L71-L78)). `planner` treats `planner_hook` the same way ([planner.c#planner](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L274-L285)). Libraries usually set hooks in `_PG_init()`, which runs when the library is loaded ([dfmgr.c:284-289](../raw/postgres-17/src/backend/utils/fmgr/dfmgr.c#L284-L289)). A hook function normally chains to the previous hook or to the `standard_` routine. `pg_stat_statements` saves the old `ProcessUtility_hook` and calls it, or `standard_ProcessUtility` when none was set ([pg_stat_statements.c:479-480](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L479-L480), [pg_stat_statements.c:1157-1164](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L1157-L1164)). Hooks are C-level and have no catalog row, unlike an [event trigger](#event-trigger).

Related: [Extension](#extension), [Event trigger](#event-trigger), [Utility command](#utility-command), [Planner](#planner)

### HOT

**Aliases:** heap-only tuple, HOT update, HOT chain, `HEAP_ONLY_TUPLE`, `HEAP_HOT_UPDATED`. **Checked on:** PostgreSQL 17.

A HOT update writes the new row version on the same heap page and adds no new index entries. The index keeps pointing at the chain's first version, and later versions are found by following the chain ([README.HOT#intro](../raw/postgres-17/src/backend/access/heap/README.HOT#L1-L60)). `heap_update()` chooses HOT only when the new version fits on the same page and no column of a HOT-blocking index changed. Summarizing indexes such as BRIN do not block HOT, but they are still updated when their columns change ([heapam.c#heap_update](../raw/postgres-17/src/backend/access/heap/heapam.c#L4138-L4160), [README.HOT#intro](../raw/postgres-17/src/backend/access/heap/README.HOT#L41-L42)). The tuple header flags `HEAP_HOT_UPDATED` (the old version) and `HEAP_ONLY_TUPLE` (the new version) mark the chain ([htup_details.h#HEAP_ONLY_TUPLE](../raw/postgres-17/src/include/access/htup_details.h#L274-L282)). Here "column used in an index" also includes columns that appear only in a partial-index predicate ([README.HOT#intro](../raw/postgres-17/src/backend/access/heap/README.HOT#L36-L39)).

Related: [Fillfactor](#fillfactor), [Line pointer](#line-pointer), [Pruning](#pruning), [Tuple](#tuple)

### Huge pages

**Aliases:** `huge_pages`, `huge_page_size`, `MAP_HUGETLB`. **Checked on:** PostgreSQL 17.

Huge pages are operating-system memory pages larger than the normal size. Using them for the shared memory segment gives smaller page tables and less CPU time spent on memory management ([config.sgml#guc-huge-pages](../raw/postgres-17/doc/src/sgml/config.sgml#L1724-L1726)). The `huge_pages` setting accepts `off`, `on` or `try` and defaults to `try`. `huge_page_size` picks the size, and 0 means the system default. Both have GUC context `postmaster`, so a change needs a restart ([guc_tables.c#huge_pages](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5056-L5064), [guc_tables.c#huge_page_size](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3597-L3607), [guc_tables.c#huge_pages_options](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L363-L374)). On `mmap` platforms, `CreateAnonymousSegment()` rounds the request up to the huge page size and asks for `MAP_HUGETLB`. With `try`, it falls back to normal pages when that fails. It records the outcome in the read-only `huge_pages_status` ([sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-17/src/backend/port/sysv_shmem.c#L597-L636)).

Related: [shared_buffers](#shared_buffers), [GUC context](#guc-context)

### Index-only scan

**Aliases:** Index Only Scan, IOS. **Checked on:** PostgreSQL 17.

An index-only scan answers a query from index entries alone. It visits the table only for pages it cannot prove are visible to every transaction. `ExecIndexOnlyScan()` checks the visibility map for each row address, and only when the page is not marked all-visible does it call `index_fetch_heap()` ([nodeIndexonlyscan.c](../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L127-L169)). `EXPLAIN ANALYZE` reports those heap visits as `Heap Fetches` ([explain.c:1992-1994](../raw/postgres-17/src/backend/commands/explain.c#L1992-L1994)). An index AM supports this only if it sets `amcanreturn`. In v17, B-tree, GiST and SP-GiST set it; GIN, hash and BRIN leave it NULL ([nbtree.c:134](../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L134), [gist.c:92](../raw/postgres-17/src/backend/access/gist/gist.c#L92), [spgutils.c:77](../raw/postgres-17/src/backend/access/spgist/spgutils.c#L77), [ginutil.c:70](../raw/postgres-17/src/backend/access/gin/ginutil.c#L70), [hash.c:90](../raw/postgres-17/src/backend/access/hash/hash.c#L90), [brin.c:280](../raw/postgres-17/src/backend/access/brin/brin.c#L280)).

Related: [Visibility map](#visibility-map), [Bitmap scan](#bitmap-scan), [EXPLAIN](#explain), [Access method](#access-method)

### IndexOptInfo

**Checked on:** PostgreSQL 17.

`IndexOptInfo` is the planner's summary of one index on a table being planned. It holds the index's size and height, its columns, and its access method's cost-estimate callback [pathnodes.h#IndexOptInfo](../raw/postgres-17/src/include/nodes/pathnodes.h#L1107-L1128) [pathnodes.h:1206](../raw/postgres-17/src/include/nodes/pathnodes.h#L1206). `get_relation_info()` builds it. For a non-partial index, `pages` is the index's current physical block count, and `tuples` is copied from the parent table. For B-tree only, `tree_height` comes from `_bt_getrootheight()`; every other access method gets -1 [plancat.c:463-500](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L500). `btcostestimate()` charges `(tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost` per descent, and that multiplier is 50.0. Its comment names this charge as what keeps bloated indexes from looking as cheap as unbloated ones [selfuncs.c#btcostestimate](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106) [selfuncs.c:145](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145).

Related: [RelOptInfo](#reloptinfo), [Cost](#cost), [B-tree](#b-tree), [Bloat](#bloat), [reltuples and relpages](#reltuples-and-relpages)

### Injection point

**Checked on:** PostgreSQL 17.

An injection point is a named spot in server code where a test can attach a callback that runs when execution reaches it ([injection_point.c header](../raw/postgres-17/src/backend/utils/misc/injection_point.c#L3-L7)). The `INJECTION_POINT(name)` macro calls `InjectionPointRun` only in builds configured with `--enable-injection-points`. Otherwise it compiles to nothing ([injection_point.h](../raw/postgres-17/src/include/utils/injection_point.h#L14-L21), [installation.sgml#configure-option-enable-injection-points](../raw/postgres-17/doc/src/sgml/installation.sgml#L1657-L1670)). Real call sites sit in hot paths such as `heap_update` ([heapam.c:3448](../raw/postgres-17/src/backend/access/heap/heapam.c#L3448)). Tests attach callbacks there through the `src/test/modules/injection_points` module ([injection_points.control](../raw/postgres-17/src/test/modules/injection_points/injection_points.control#L1-L4)).

Related: [Isolation test](#isolation-test), [TAP test](#tap-test)

### Invalid index

**Aliases:** `indisvalid`, `indisready`, `indislive`. **Checked on:** PostgreSQL 17.

An invalid index has `pg_index.indisvalid = false`, so the planner will not use it for queries. Three `pg_index` flags track an index's state: `indislive` (it exists), `indisready` (inserts must maintain it) and `indisvalid` (queries may use it) ([pg_index.h:42-45](../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)). `get_relation_info()` skips any index without `indisvalid`. The executor still inserts into it when `indisready` is set ([plancat.c:256-267](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267)). A failed `CREATE INDEX CONCURRENTLY` leaves one behind. `psql` shows it as `INVALID`, and the documented fix is to drop it or run `REINDEX INDEX CONCURRENTLY` ([ref/create_index.sgml](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L666)).

Related: [CONCURRENTLY](#concurrently), [pg_index](#pg_index), [REINDEX](#reindex), [Planner](#planner)

### Isolation test

**Aliases:** isolation tester, spec test. **Checked on:** PostgreSQL 17.

An isolation test checks concurrent behavior by running several interleaved sessions against one server ([isolation/README](../raw/postgres-17/src/test/isolation/README#L3-L12)). A spec file under `specs/` defines SQL steps, groups them into sessions, and lists the permutations (step orders) to run. Each session gets its own connection ([isolation/README](../raw/postgres-17/src/test/isolation/README#L56-L63), [isolation/README](../raw/postgres-17/src/test/isolation/README#L83-L98)). The ordinary `pg_regress` driver cannot run these tests, because it uses one connection at a time ([isolation/README](../raw/postgres-17/src/test/isolation/README#L6-L9)).

Related: [Regression test](#regression-test), [TAP test](#tap-test), [Injection point](#injection-point)

### Line pointer

**Aliases:** item identifier, `ItemIdData`, `lp_flags`, `LP_UNUSED`, `LP_NORMAL`, `LP_REDIRECT`, `LP_DEAD`. **Checked on:** PostgreSQL 17.

A line pointer is a 4-byte slot in the array at the start of a page. It records where a tuple sits on the page (`lp_off`), how long it is (`lp_len`) and what state it is in (`lp_flags`) ([itemid.h#ItemIdData](../raw/postgres-17/src/include/storage/itemid.h#L17-L30)). A TID names a line pointer, not a byte offset, so a tuple can move inside its page without changing its TID ([bufpage.h#NOTES](../raw/postgres-17/src/include/storage/bufpage.h#L56-L64)). The four states are `LP_UNUSED` (free for reuse), `LP_NORMAL` (points to a tuple), `LP_REDIRECT` (a HOT link to another slot, with no storage) and `LP_DEAD` (dead; it may or may not still have storage) ([itemid.h#LP_UNUSED](../raw/postgres-17/src/include/storage/itemid.h#L34-L41)). VACUUM sets an `LP_DEAD` slot to `LP_UNUSED` only in its second heap pass, after the index entries that point to it are gone. No index entry may ever point to an `LP_UNUSED` slot ([vacuumlazy.c#lazy_vacuum](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L793-L800)).

Related: [HOT](#hot), [Page](#page), [Pruning](#pruning), [TID](#tid)

### Logical decoding

**Checked on:** PostgreSQL 17.

Logical decoding reads WAL records and turns them into a stream of changes that a consumer can read ([decode.c header](../raw/postgres-17/src/backend/replication/logical/decode.c#L3-L8), [logical.c header](../raw/postgres-17/src/backend/replication/logical/logical.c#L10-L18)). `LogicalDecodingProcessRecord` feeds each WAL record through the decoder ([decode.c#LogicalDecodingProcessRecord](../raw/postgres-17/src/backend/replication/logical/decode.c#L87-L88)). An output plugin then formats the changes through the callbacks in `OutputPluginCallbacks`. `pgoutput` is the plugin that logical replication uses ([output_plugin.h#OutputPluginCallbacks](../raw/postgres-17/src/include/replication/output_plugin.h#L216-L243), [pgoutput.c header](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L3-L4)). `CheckLogicalDecodingRequirements` refuses to decode unless `wal_level` is `logical` and the process is connected to a database ([logical.c#CheckLogicalDecodingRequirements](../raw/postgres-17/src/backend/replication/logical/logical.c#L116-L133)).

Related: [WAL](#wal), [Logical replication](#logical-replication), [Replication slot](#replication-slot), [Replication origin](#replication-origin)

### Logical replication

**Checked on:** PostgreSQL 17.

Logical replication copies table rows and their changes by replication identity, not by block address as physical replication does ([logical-replication.sgml](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L6-L13)). It uses a publish-and-subscribe model. A subscriber first copies a snapshot of the data, then applies later changes in the publisher's order ([logical-replication.sgml](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L15-L33)). In source, the publisher side is logical decoding plus the `pgoutput` plugin. On the subscriber, the launcher starts apply and tablesync workers as dynamic background workers ([pgoutput.c header](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L3-L4), [launcher.c#logicalrep_worker_launch](../raw/postgres-17/src/backend/replication/logical/launcher.c#L470-L503)).

Related: [Publication](#publication), [Subscription](#subscription), [Apply worker](#apply-worker), [Logical decoding](#logical-decoding), [Replication slot](#replication-slot)

### LSN

**Aliases:** log sequence number, `XLogRecPtr`, `pg_lsn`. **Checked on:** PostgreSQL 17.

A 64-bit byte position in the WAL stream ([xlogdefs.h#XLogRecPtr](../raw/postgres-17/src/include/access/xlogdefs.h#L17-L21)). Every data page stores in `pd_lsn` the LSN just past the last WAL record that changed it ([bufpage.h#PageHeaderData](../raw/postgres-17/src/include/storage/bufpage.h#L155-L158)). The buffer manager must flush WAL up to a page's LSN before writing that page. This is the "write the log before the data" rule ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L409-L415), [xloginsert.c#XLogInsert](../raw/postgres-17/src/backend/access/transam/xloginsert.c#L466-L472)). Code prints an LSN as `%X/%X` with `LSN_FORMAT_ARGS()` ([xlogdefs.h:42-44](../raw/postgres-17/src/include/access/xlogdefs.h#L42-L44)).

Related: [WAL](#wal), [Page](#page), [Full-page image](#full-page-image)

### LWLock

**Aliases:** lightweight lock. **Checked on:** PostgreSQL 17.

A short-term lock that protects a shared-memory data structure, in shared or exclusive mode. It has no deadlock detection and no timeout. It is released automatically on error, and waiters are served in arrival order ([lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L20-L30)). The struct holds a tranche ID, an atomic state word and a waiter list ([lwlock.h#LWLock](../raw/postgres-17/src/include/storage/lwlock.h#L41-L50)). Code takes one with `LWLockAcquire()` ([lwlock.c#LWLockAcquire](../raw/postgres-17/src/backend/storage/lmgr/lwlock.c#L1170)). One example is `XidGenLock`, taken while the transaction ID limits are updated ([varsup.c:443](../raw/postgres-17/src/backend/access/transam/varsup.c#L443)). Unlike a heavyweight lock, it has no deadlock detection and is not held to transaction end ([lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L20-L35)).

Related: [Heavyweight lock](#heavyweight-lock), [Wait event](#wait-event)

### maintenance_work_mem

**Checked on:** PostgreSQL 17.

`maintenance_work_mem` caps the memory used by maintenance operations such as `VACUUM` and `CREATE INDEX`. Its context is `user`, so a session or transaction can `SET` it with no reload or restart. Its default is 65536 kB (64 MB) [guc_tables.c#maintenance_work_mem](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2474). A B-tree build sizes its sort with it instead of `work_mem` [nbtsort.c:370-375](../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L370-L375). Lazy `VACUUM` uses it to bound the store of dead TIDs. When that store fills, `VACUUM` must pass over every index before it can continue. An autovacuum worker uses `autovacuum_work_mem` instead when that setting is not -1 [vacuumlazy.c:12-16](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L12-L16) [vacuumlazy.c#dead_items_alloc](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2823-L2828).

Related: [work_mem](#work_mem), [VACUUM](#vacuum), [TID](#tid), [GUC context](#guc-context)

### Memory context

**Aliases:** `palloc`, `pfree`, `CurrentMemoryContext`, `MemoryContextSwitchTo`, `TopMemoryContext`. **Checked on:** PostgreSQL 17.

A memory context is a named pool of memory with a defined lifetime, and the server does most of its allocation inside one. Freeing or resetting the context releases everything allocated in it at once ([mmgr/README:9-49](../raw/postgres-17/src/backend/utils/mmgr/README#L9-L49)). `palloc` allocates in `CurrentMemoryContext`, and `MemoryContextSwitchTo` changes that target ([mmgr/README:36-40](../raw/postgres-17/src/backend/utils/mmgr/README#L36-L40)). `palloc` never returns NULL: when it runs out of memory it raises `elog(ERROR)` ([mmgr/README:59-62](../raw/postgres-17/src/backend/utils/mmgr/README#L59-L62)). Contexts form a tree rooted at `TopMemoryContext`, with children such as `TopTransactionContext` and `MessageContext` ([mmgr/README:185-223](../raw/postgres-17/src/backend/utils/mmgr/README#L185-L223)). Knowing the current context tells a reader how long a pointer stays valid.

Related: [ereport](#ereport), [work_mem](#work_mem), [Portal](#portal)

### Metapage

**Aliases:** meta page, metadata page. **Checked on:** PostgreSQL 17.

A metapage is a special page, at block 0 of an index, that holds bookkeeping for the whole index instead of index entries. B-tree, hash, GIN, SP-GiST and BRIN each define one at block 0; GiST puts its root there instead ([nbtree.h:148](../raw/postgres-17/src/include/access/nbtree.h#L148), [hash.h:198](../raw/postgres-17/src/include/access/hash.h#L198), [ginblock.h:52](../raw/postgres-17/src/include/access/ginblock.h#L52), [spgist_private.h:47](../raw/postgres-17/src/include/access/spgist_private.h#L47), [brin_page.h:75](../raw/postgres-17/src/include/access/brin_page.h#L75), [gist_private.h:262](../raw/postgres-17/src/include/access/gist_private.h#L262)). The B-tree metapage (`BTMetaPageData`) records the root, the fast root, the on-disk version and `btm_allequalimage` ([nbtree.h#BTMetaPageData](../raw/postgres-17/src/include/access/nbtree.h#L103-L119)). The GIN metapage records where the pending list starts and ends and how many pages it has ([ginblock.h#GinMetaPageData](../raw/postgres-17/src/include/access/ginblock.h#L55-L75)).

Related: [Fast root](#fast-root), [allequalimage](#allequalimage), [Pending list](#pending-list), [B-tree](#b-tree)

### MultiXact

**Aliases:** MultiXactId, MXID, `pg_multixact`. **Checked on:** PostgreSQL 17.

An ID that stands for a set of transactions, used when several transactions lock or update the same row at once and a single xmax cannot name them all ([multixact.c](../raw/postgres-17/src/backend/access/transam/multixact.c#L5-L16)). A `MultiXactId` has the same width as a `TransactionId` so that it fits in `t_xmax` ([c.h:678-679](../raw/postgres-17/src/include/c.h#L678-L679)). The `HEAP_XMAX_IS_MULTI` infomask bit says which one `t_xmax` holds ([htup_details.h:209](../raw/postgres-17/src/include/access/htup_details.h#L209)). MultiXactIds also age and must be frozen by VACUUM, as xids are ([heapam.c#heap_prepare_freeze_tuple](../raw/postgres-17/src/backend/access/heap/heapam.c#L7393-L7403)).

Related: [xmin and xmax](#xmin-and-xmax), [Transaction ID](#transaction-id), [Freezing](#freezing), [Wraparound](#wraparound)

### MVCC

**Aliases:** multi-version concurrency control. **Checked on:** PostgreSQL 17.

PostgreSQL's way of letting readers and writers work on the same rows without blocking each other. An update creates a new version of a row instead of changing it in place, and old versions are removed only after no transaction can still see them ([glossary.sgml#mvcc](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1171-L1185)). Each heap tuple records its inserting and deleting transactions in `t_xmin` and `t_xmax` ([htup_details.h#HeapTupleFields](../raw/postgres-17/src/include/access/htup_details.h#L122-L132)). `HeapTupleSatisfiesMVCC()` checks those against a snapshot to decide visibility ([heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L937-L960)). Outdated row versions count as bloat until they are removed ([glossary.sgml#glossary-bloat](../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)).

Related: [Snapshot](#snapshot), [xmin and xmax](#xmin-and-xmax), [xmin horizon](#xmin-horizon), [Tuple](#tuple), [Bloat](#bloat), [VACUUM](#vacuum)

### OID

**Aliases:** object identifier, `Oid`. **Checked on:** PostgreSQL 17.

An OID is an unsigned 4-byte integer that serves as the primary key of most system catalogs ([postgres_ext.h:27-31](../raw/postgres-17/src/include/postgres_ext.h#L27-L31), [datatype.sgml:4767-4781](../raw/postgres-17/doc/src/sgml/datatype.sgml#L4767-L4781)). OIDs below 16384 are reserved for objects that `initdb` creates. User objects get OIDs from 16384 upward, and a wrapped counter skips back over the reserved range ([transam.h:173-197](../raw/postgres-17/src/include/access/transam.h#L173-L197)). OIDs are unique only within one catalog, so `GetNewOidWithIndex` checks the target catalog's index for a collision ([catalog.c#GetNewOidWithIndex](../raw/postgres-17/src/backend/catalog/catalog.c#L396-L421)). A relation's OID (`pg_class.oid`) is not its file name. The file is named by its [relfilenumber](#relfilenumber), and the two can diverge ([pg_upgrade.c:17-20](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L17-L20)).

Related: [Catalog](#catalog), [pg_class](#pg_class), [Relfilenumber](#relfilenumber), [Transaction ID](#transaction-id)

### Page split

**Checked on:** PostgreSQL 17.

A page split happens when a new entry does not fit on an index page. In a B-tree, `_bt_insertonpg()` calls `_bt_split()` when `PageGetFreeSpace()` is smaller than the new item. `_bt_split()` moves part of the entries to a new right sibling, and `_bt_insert_parent()` then adds a downlink for it in the parent page ([nbtinsert.c:1210-1241](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1210-L1241), [nbtinsert.c#_bt_split](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1440-L1470), [nbtinsert.c#_bt_insert_parent](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2083-L2084)). `_bt_findsplitloc()` normally divides free space evenly between the two pages. On the rightmost page it leaves the left page `fillfactor`% full, so ascending inserts produce pages about that full ([nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L86-L102)). Before splitting a leaf, B-tree first tries simple deletion, then bottom-up deletion, then deduplication ([nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2683-L2781)).

Related: [Fillfactor](#fillfactor), [Simple index deletion](#simple-index-deletion), [Bottom-up index deletion](#bottom-up-index-deletion), [Deduplication](#deduplication), [Bloat](#bloat)

### Page

**Aliases:** disk page, slotted page, `PageHeaderData`. **Checked on:** PostgreSQL 17.

A page is the formatted layout inside a block. It has a header, an array of line pointers growing forward, tuples growing backward from the end, and an optional "special space" for index metadata. The page is full when `pd_lower` meets `pd_upper` ([bufpage.h#layout](../raw/postgres-17/src/include/storage/bufpage.h#L22-L45)). `PageHeaderData` holds the page LSN (`pd_lsn`), checksum, flags, the free-space bounds, `pd_special`, and `pd_prune_xid`, which is a pruning hint ([bufpage.h#PageHeaderData](../raw/postgres-17/src/include/storage/bufpage.h#L155-L168)).

Related: [Block](#block), [Line pointer](#line-pointer), [LSN](#lsn), [Metapage](#metapage)

### pageinspect

**Checked on:** PostgreSQL 17.

`pageinspect` is a contrib extension that shows the raw contents of individual database pages ([pageinspect.control](../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L2)). Its core function `get_raw_page` requires a superuser. It copies one block of any relation fork byte for byte into a `bytea` value ([rawpage.c#get_raw_page_internal](../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L153), [rawpage.c#get_raw_page_internal](../raw/postgres-17/contrib/pageinspect/rawpage.c#L185-L193)). Separate per-format decoders parse such a copy, for example `bt_page_items` for B-tree pages and the GIN and heap readers in their own source files ([pageinspect--1.5--1.6.sql](../raw/postgres-17/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88-L90), [ginfuncs.c header](../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L1-L3), [heapfuncs.c header](../raw/postgres-17/contrib/pageinspect/heapfuncs.c#L1-L4)).

Related: [Page](#page), [Fork](#fork), [Contrib](#contrib), [Extension](#extension), [pgstattuple](#pgstattuple)

### Parse tree

**Checked on:** PostgreSQL 17.

A parse tree is the in-memory tree of nodes that represents one SQL statement. `raw_parser()` runs the lexer and grammar and returns raw parse trees, each wrapped in a `RawStmt` [parser.c#raw_parser](../raw/postgres-17/src/backend/parser/parser.c#L34-L42) [parsenodes.h#RawStmt](../raw/postgres-17/src/include/nodes/parsenodes.h#L2005-L2025). Parse analysis turns each raw tree into a `Query` node. That step does real work only for optimizable statements. A utility statement's raw tree is copied into `Query.utilityStmt` [parsenodes.h:2005-2009](../raw/postgres-17/src/include/nodes/parsenodes.h#L2005-L2009). The rewriter and planner consume the `Query`, and the executor never sees it [parsenodes.h#Query](../raw/postgres-17/src/include/nodes/parsenodes.h#L101-L117).

Related: [Grammar](#grammar), [Planner](#planner), [Utility command](#utility-command), [PlannedStmt](#plannedstmt)

### Partial index

**Aliases:** predicate index, `indpred`. **Checked on:** PostgreSQL 17.

A partial index covers only the table rows that satisfy a `WHERE` condition, called the predicate. It is smaller, and it is cheaper to maintain than an index on every row ([indices.sgml#indexes-partial](../raw/postgres-17/doc/src/sgml/indices.sgml#L801-L827)). The predicate is stored as an expression tree in `pg_index.indpred` ([pg_index.h:60-61](../raw/postgres-17/src/include/catalog/pg_index.h#L60-L61)). The planner may use the index only when it can prove that the query's conditions imply the predicate. `check_index_predicates()` sets `IndexOptInfo.predOK` from `predicate_implied_by()` ([indxpath.c#check_index_predicates](../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3227-L3244), [indxpath.c:3349](../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3349)).

Related: [Expression index](#expression-index), [pg_index](#pg_index), [IndexOptInfo](#indexoptinfo), [Planner](#planner)

### Partition pruning

**Checked on:** PostgreSQL 17.

Partition pruning skips the partitions of a partitioned table that cannot hold matching rows. It compares the query's conditions with each partition's bounds [partprune.c header](../raw/postgres-17/src/backend/partitioning/partprune.c#L3-L25). It can run at plan time, in `prune_append_rel_partitions()`. It can also run during execution when a value becomes known only then, such as a parameter; the planner then attaches pruning steps to the plan with `make_partition_pruneinfo()` [partprune.c:15-18](../raw/postgres-17/src/backend/partitioning/partprune.c#L15-L18) [execPartition.c#ExecFindMatchingSubPlans](../raw/postgres-17/src/backend/executor/execPartition.c#L2308). The `enable_partition_pruning` GUC controls both phases. Its context is `user`, so a session or transaction can `SET` it with no reload or restart. It is on by default [guc_tables.c#enable_partition_pruning](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L964-L974).

Related: [Constraint exclusion](#constraint-exclusion), [Partitionwise join](#partitionwise-join), [Planner](#planner), [Executor](#executor)

### Partitionwise join

**Checked on:** PostgreSQL 17.

A partitionwise join splits one join of two partitioned tables into joins between matching partitions, then appends the results. It needs both tables to use the same partitioning scheme, with an equi-join on the partition keys [joinrels.c#try_partitionwise_join](../raw/postgres-17/src/backend/optimizer/path/joinrels.c#L1458-L1477). `build_joinrel_partition_info()` checks those conditions and returns early when the feature is off [relnode.c#build_joinrel_partition_info](../raw/postgres-17/src/backend/optimizer/util/relnode.c#L2016-L2045). The `enable_partitionwise_join` GUC has context `user`, so a session or transaction can `SET` it with no reload or restart. It is off by default [guc_tables.c#enable_partitionwise_join](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L924-L932).

Related: [Partition pruning](#partition-pruning), [RelOptInfo](#reloptinfo), [Planner](#planner)

### Path

**Checked on:** PostgreSQL 17.

A `Path` is one candidate way to produce a relation's rows, such as a sequential scan, an index scan, or a particular join method. It carries estimated rows, `startup_cost`, `total_cost`, and sort order (`pathkeys`) [pathnodes.h#Path](../raw/postgres-17/src/include/nodes/pathnodes.h#L1622-L1669). The planner collects paths in each `RelOptInfo.pathlist`. `add_path()` keeps a new path only if no existing path beats it on cost, sort order, row count, parameterization, and parallel safety, and it discards the old paths the new one beats [pathnode.c#add_path](../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L361-L420). A path is not yet executable. `create_plan()` turns the winning path into a `Plan` [planner.c:418-422](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L418-L422).

Related: [RelOptInfo](#reloptinfo), [Cost](#cost), [Planner](#planner), [PlannedStmt](#plannedstmt)

### Pending list

**Aliases:** GIN fast update, `fastupdate`, `gin_pending_list_limit`. **Checked on:** PostgreSQL 17.

The pending list is an unsorted list of GIN pages where new entries wait before being merged into the main GIN structure. This makes inserts faster, but every search must also scan the list ([gin.sgml](../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L528), [ginfast.c](../raw/postgres-17/src/backend/access/gin/ginfast.c#L1-L7)). The `fastupdate` storage parameter turns it on or off; the default is on ([reloptions.c:123-131](../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131)). An insert calls `ginInsertCleanup()` once the list grows past the index's own `gin_pending_list_limit` storage parameter, or, when that is unset, the `gin_pending_list_limit` GUC ([ginfast.c:458-471](../raw/postgres-17/src/backend/access/gin/ginfast.c#L458-L471), [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-17/src/include/access/gin_private.h#L39-L45)). The GUC defaults to 4096 kB. Its context is `user`, so a session or transaction can `SET` it with no reload or restart ([guc_tables.c:3577-3585](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3577-L3585)). VACUUM, autoanalyze and `gin_clean_pending_list()` also empty it ([gin.sgml](../raw/postgres-17/doc/src/sgml/gin.sgml#L508-L515)). The list's head, tail and size live in the GIN metapage ([ginblock.h#GinMetaPageData](../raw/postgres-17/src/include/access/ginblock.h#L55-L75)).

Related: [GIN](#gin), [Metapage](#metapage), [VACUUM](#vacuum), [GUC context](#guc-context)

### pgstatindex

**Checked on:** PostgreSQL 17.

`pgstatindex` is a `pgstattuple` function that summarizes a B-tree index's shape and space use. It rejects any other index type and any index that is not valid ([pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L250)). It reads the metapage, then every other block, and counts deleted, half-dead ("empty"), leaf and internal pages ([pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L300-L327)). `avg_leaf_density` is 100 minus leaf free space as a percentage of leaf capacity. `leaf_fragmentation` is the share of leaves whose right sibling sits at a lower block number ([pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L310-L322), [pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L349-L372)). The install scripts grant it to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql](../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)).

Related: [pgstattuple](#pgstattuple), [B-tree](#b-tree), [Metapage](#metapage), [Bloat](#bloat), [Invalid index](#invalid-index)

### pgstattuple

**Checked on:** PostgreSQL 17.

`pgstattuple` is a contrib extension that scans a relation and reports tuple-level space use: live and dead tuple counts and bytes, plus free space ([pgstattuple.control](../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L2), [pgstattuple.c#pgstattuple_type](../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L55-L63)). `pgstat_relation` sends tables and sequences to `pgstat_heap`. It handles B-tree, hash and GiST indexes itself and rejects GIN, SP-GiST and BRIN ([pgstattuple.c#pgstat_relation](../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L255-L298)). The same extension provides `pgstatindex`, `pgstatginindex`, `pgstathashindex` and the sampling `pgstattuple_approx` ([pgstatindex.c#pgstatginindex](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L485-L507), [pgstatindex.c:586](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L586), [pgstatapprox.c#pgstattuple_approx](../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L227)).

Related: [pgstatindex](#pgstatindex), [Tuple](#tuple), [Bloat](#bloat), [Contrib](#contrib)

### pg_cast

**Aliases:** cast, binary-coercible, `castmethod`. **Checked on:** PostgreSQL 17.

`pg_cast` is the catalog of data-type conversions. Each row names a source type, a target type, and a cast function, where function 0 means the two types are binary-coercible ([pg_cast.h#FormData_pg_cast](../raw/postgres-17/src/include/catalog/pg_cast.h#L32-L50)). `castmethod` records whether the cast uses a function (`f`), binary compatibility (`b`), or the types' text input and output functions (`i`) ([pg_cast.h#CoercionMethod](../raw/postgres-17/src/include/catalog/pg_cast.h#L87-L92)). The distinction matters for `ALTER COLUMN ... TYPE`. `ATColumnChangeRequiresRewrite` skips the table rewrite for a binary-coercible change and requires a rewrite for most function casts ([tablecmds.c#ATColumnChangeRequiresRewrite](../raw/postgres-17/src/backend/commands/tablecmds.c#L13100-L13152)). `int4` to `int8` is a function cast ([pg_cast.dat:41-42](../raw/postgres-17/src/include/catalog/pg_cast.dat#L41-L42)).

Related: [Table rewrite](#table-rewrite), [Catalog](#catalog), [fmgr](#fmgr)

### pg_class

**Aliases:** relation, `relkind`, `Form_pg_class`. **Checked on:** PostgreSQL 17.

`pg_class` is the catalog with one row for every relation: tables, indexes, sequences, views, materialized views, TOAST tables, composite types, foreign tables, and partitioned tables and indexes ([pg_class.h:32](../raw/postgres-17/src/include/catalog/pg_class.h#L32), [pg_class.h:164-173](../raw/postgres-17/src/include/catalog/pg_class.h#L164-L173)). Its columns include the access method `relam`, the file identifier `relfilenode`, the planner's size estimates `relpages` and `reltuples`, and `relkind` ([pg_class.h:52-84](../raw/postgres-17/src/include/catalog/pg_class.h#L52-L84)). In C, a row is a `FormData_pg_class` read through `Form_pg_class`. A relcache entry keeps a copy in `rd_rel` ([pg_class.h:142-153](../raw/postgres-17/src/include/catalog/pg_class.h#L142-L153), [rel.h:111](../raw/postgres-17/src/include/utils/rel.h#L111)).

Related: [reltuples and relpages](#reltuples-and-relpages), [Relfilenumber](#relfilenumber), [pg_index](#pg_index), [Relcache](#relcache), [OID](#oid)

### pg_freespacemap

**Checked on:** PostgreSQL 17.

`pg_freespacemap` is a contrib extension that reads a relation's free space map ([pg_freespacemap.control](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.control#L1-L2)). `pg_freespace(rel, blkno)` returns `GetRecordedFreeSpace()` for one block. The one-argument form loops over every block of the main fork ([pg_freespacemap.c#pg_freespace](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L48), [pg_freespacemap--1.1.sql](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L20)). The value is what the FSM recorded, not a live page measurement. The FSM stores one byte per page, so free space is rounded to `BLCKSZ / 256` steps ([freespace.c](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L36-L65), [freespace.c#GetRecordedFreeSpace](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L249-L269)). The install script revokes both functions from `PUBLIC` ([pg_freespacemap--1.1.sql](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L23-L25)).

Related: [Free space map](#free-space-map), [BLCKSZ](#blcksz), [Contrib](#contrib)

### pg_index

**Aliases:** `Form_pg_index`, `indisvalid`, `indisready`, `indislive`. **Checked on:** PostgreSQL 17.

`pg_index` holds the index-specific facts for each index. Its row links the index's `pg_class` row (`indexrelid`) to the table it indexes (`indrelid`) ([pg_index.h#FormData_pg_index](../raw/postgres-17/src/include/catalog/pg_index.h#L29-L63)). Three flags track an index's lifecycle. `indislive` says whether it exists at all, `indisready` whether it receives inserts, and `indisvalid` whether queries may use it ([pg_index.h:42-45](../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)). The row also stores the key columns `indkey`, the operator classes, the expression trees `indexprs`, and a partial-index predicate `indpred` ([pg_index.h:49-61](../raw/postgres-17/src/include/catalog/pg_index.h#L49-L61)). The relcache exposes the row as `rd_index` ([rel.h:192](../raw/postgres-17/src/include/utils/rel.h#L192)).

Related: [Invalid index](#invalid-index), [CONCURRENTLY](#concurrently), [Partial index](#partial-index), [Expression index](#expression-index), [pg_class](#pg_class)

### pg_upgrade

**Checked on:** PostgreSQL 17.

`pg_upgrade` moves a cluster to a newer major version without a dump and restore. It creates new system catalogs and reuses the old user data files ([pgupgrade.sgml:39-58](../raw/postgres-17/doc/src/sgml/ref/pgupgrade.sgml#L39-L58)). It carries the files over by clone, copy, `copy_file_range`, or hard link ([pg_upgrade.h#transferMode](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.h#L253-L260)). It forces table OIDs and relfilenumbers to match between the old and new clusters so that the file names and the stored TOAST pointers stay valid ([pg_upgrade.c:10-21](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L10-L21)). Because it reuses the old user data files, index pages keep the on-disk state the old version wrote ([pgupgrade.sgml:48-53](../raw/postgres-17/doc/src/sgml/ref/pgupgrade.sgml#L48-L53)).

Related: [Relfilenumber](#relfilenumber), [OID](#oid), [allequalimage](#allequalimage), [Catalog](#catalog)

### PlannedStmt

**Checked on:** PostgreSQL 17.

`PlannedStmt` is the planner's output. It is the top node of a `Plan` tree and holds the one-time information the executor needs: the range table, result relations, subplans, and the OIDs the plan depends on [plannodes.h#PlannedStmt](../raw/postgres-17/src/include/nodes/plannodes.h#L31-L35) [plannodes.h:70-89](../raw/postgres-17/src/include/nodes/plannodes.h#L70-L89). A utility statement is also wrapped in a `PlannedStmt`. That wrapper has `commandType = CMD_UTILITY` and the statement in `utilityStmt`, and the rest of the struct is mostly unused [plannodes.h:37-40](../raw/postgres-17/src/include/nodes/plannodes.h#L37-L40). `pg_plan_queries()` builds such a wrapper without calling the planner [postgres.c#pg_plan_queries](../raw/postgres-17/src/backend/tcop/postgres.c#L976-L1008).

Related: [Planner](#planner), [Executor](#executor), [Utility command](#utility-command), [Parse tree](#parse-tree)

### Planner

**Aliases:** optimizer, query planner, query optimizer. **Checked on:** PostgreSQL 17.

The planner decides how to run a query. It turns an analyzed `Query` into the cheapest plan it can find [glossary.sgml#glossary-planner](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1334-L1345). Its entry point, `planner()`, calls `planner_hook` if an extension installed one and `standard_planner()` otherwise [planner.c#planner](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L262-L285). `standard_planner()` runs `subquery_planner()` to build paths. It then picks the cheapest final path with `get_cheapest_fractional_path()` and turns it into a plan with `create_plan()` [planner.c:415-422](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L415-L422). The planner reads table and index sizes from the catalogs in `plancat.c` [plancat.c#get_relation_info](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L115-L117).

Related: [Path](#path), [RelOptInfo](#reloptinfo), [Cost](#cost), [PlannedStmt](#plannedstmt), [Hook](#hook)

### PL/pgSQL

**Aliases:** plpgsql, `DO` block. **Checked on:** PostgreSQL 17.

PL/pgSQL is PostgreSQL's built-in procedural language for writing functions, procedures, and anonymous `DO` blocks [plpgsql.sgml:4](../raw/postgres-17/doc/src/sgml/plpgsql.sgml#L4). It ships as the `plpgsql` extension, and `initdb` creates it in every new cluster [plpgsql--1.0.sql](../raw/postgres-17/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L15) [initdb.c#load_plpgsql](../raw/postgres-17/src/bin/initdb/initdb.c#L1974-L1976). A `DO` statement reaches `ExecuteDoStmt()`, which calls the language's inline handler, `plpgsql_inline_handler()` [functioncmds.c#ExecuteDoStmt](../raw/postgres-17/src/backend/commands/functioncmds.c#L2060-L2066) [functioncmds.c:2146-2156](../raw/postgres-17/src/backend/commands/functioncmds.c#L2146-L2156). The inline handler connects to SPI before running any SQL [pl_handler.c#plpgsql_inline_handler](../raw/postgres-17/src/pl/plpgsql/src/pl_handler.c#L313-L330).

Related: [SPI](#spi), [Extension](#extension), [Utility command](#utility-command)

### Portal

**Checked on:** PostgreSQL 17.

A portal holds the execution state of a query that is running or ready to run. Both SQL cursors and the unnamed and named portals of the client protocol use it [portal.h header](../raw/postgres-17/src/include/utils/portal.h#L3-L9). It is a `PortalData` struct with its own memory context [portal.h#PortalData](../raw/postgres-17/src/include/utils/portal.h#L113-L120). `PortalRun()` runs a portal's statements and can stop after a row count, leaving the portal suspended [pquery.c#PortalRun](../raw/postgres-17/src/backend/tcop/pquery.c#L662-L689). For a utility statement, `PortalRunUtility()` calls `ProcessUtility()` [pquery.c#PortalRunUtility](../raw/postgres-17/src/backend/tcop/pquery.c#L1125-L1156).

Related: [Executor](#executor), [Utility command](#utility-command), [Memory context](#memory-context)

### Posting list

**Checked on:** PostgreSQL 17.

A posting list is an array of row addresses (TIDs) stored under one key. B-tree and GIN both use the idea, in different formats. In B-tree, deduplication creates posting-list tuples: one key followed by a sorted TID array, marked with `INDEX_ALT_TID_MASK` and `BT_IS_POSTING` ([nbtree.h](../raw/postgres-17/src/include/access/nbtree.h#L432-L457)). In GIN, a leaf entry holds a compressed posting list when it fits in `GinMaxItemSize`. Otherwise the entry points to a posting tree ([gininsert.c#buildFreshLeafTuple](../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L165)). GIN compresses its lists with varbyte encoding; B-tree does not compress them ([ginpostinglist.c](../raw/postgres-17/src/backend/access/gin/ginpostinglist.c#L23-L42), [nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L931-L942)).

Related: [Deduplication](#deduplication), [Posting tree](#posting-tree), [GIN](#gin), [TID](#tid)

### Posting tree

**Checked on:** PostgreSQL 17.

A posting tree is a separate B-tree of row addresses (TIDs) that GIN builds for one key when its posting list is too large to fit inline in the entry tuple ([gin/README](../raw/postgres-17/src/backend/access/gin/README#L22-L26)). `buildFreshLeafTuple()` first tries to compress the TIDs into a posting list that fits `GinMaxItemSize`. If that fails, it calls `createPostingTree()` and stores the tree's root block in the entry with `GinSetPostingTree()` ([gininsert.c#buildFreshLeafTuple](../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L165), [ginblock.h#GinMaxItemSize](../raw/postgres-17/src/include/access/ginblock.h#L249-L253)). Do not confuse it with GIN's main entry tree, which is indexed by key rather than by TID.

Related: [Posting list](#posting-list), [GIN](#gin), [TID](#tid)

### Postmaster

**Checked on:** PostgreSQL 17.

The postmaster is the first process of a server instance. It starts the auxiliary processes and creates a backend for each client connection ([glossary.sgml#glossary-postmaster](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1256-L1270)). It creates shared memory but avoids touching it, so a crashing backend cannot take it down. On a backend crash it resets the system ([postmaster.c header](../raw/postgres-17/src/backend/postmaster/postmaster.c#L3-L23)). Source entry points are `PostmasterMain`, the idle loop `ServerLoop`, and `BackendStartup` ([postmaster.c#PostmasterMain](../raw/postgres-17/src/backend/postmaster/postmaster.c#L486-L490), [postmaster.c#ServerLoop](../raw/postgres-17/src/backend/postmaster/postmaster.c#L1625-L1629), [postmaster.c#BackendStartup](../raw/postgres-17/src/backend/postmaster/postmaster.c#L3525-L3536)).

Related: [Backend](#backend), [Background worker](#background-worker), [Crash recovery](#crash-recovery)

### Pruning

**Aliases:** heap page pruning, HOT pruning, page defragmentation, `heap_page_prune_opt`. **Checked on:** PostgreSQL 17.

Pruning removes dead tuples from one heap page and shortens HOT chains. It can run without a full VACUUM. Defragmentation then moves the surviving tuples together so the freed space becomes usable ([README.HOT#Pruning](../raw/postgres-17/src/backend/access/heap/README.HOT#L199-L219)). Ordinary reads call `heap_page_prune_opt()` opportunistically. It returns early during recovery, when the page has no `pd_prune_xid`, or when that XID is not yet removable. Otherwise it prunes only when the page is short of free space ([pruneheap.c#heap_page_prune_opt](../raw/postgres-17/src/backend/access/heap/pruneheap.c#L180-L230)). VACUUM uses `heap_page_prune_and_freeze()`, which can also freeze tuples in the same pass ([pruneheap.c#heap_page_prune_and_freeze](../raw/postgres-17/src/backend/access/heap/pruneheap.c#L330-L352)). Pruning cannot remove index entries. So a chain's root line pointer stays while any chain member is live, and becomes "dead" when none is. The next regular VACUUM removes the index entries and then that line pointer ([README.HOT#root-line-pointer](../raw/postgres-17/src/backend/access/heap/README.HOT#L122-L128)).

Related: [Freezing](#freezing), [HOT](#hot), [Line pointer](#line-pointer), [VACUUM](#vacuum)

### Publication

**Checked on:** PostgreSQL 17.

A publication is a named set of table changes that a publisher node offers to subscribers. It exists in one database and can limit which DML kinds it sends ([logical-replication.sgml#logical-replication-publication](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L100-L122)). The `pg_publication` catalog stores it, with flags for all tables, each operation kind, and publishing partition changes through the root ([pg_publication.h#FormData_pg_publication](../raw/postgres-17/src/include/catalog/pg_publication.h#L29-L57)). `CREATE PUBLICATION` runs `CreatePublication` ([publicationcmds.c#CreatePublication](../raw/postgres-17/src/backend/commands/publicationcmds.c#L733)).

Related: [Logical replication](#logical-replication), [Subscription](#subscription), [Catalog](#catalog)

### Regression test

**Aliases:** `pg_regress`, `make check`. **Checked on:** PostgreSQL 17.

A regression test runs SQL scripts and compares their output with stored expected output ([pg_regress.c header](../raw/postgres-17/src/test/regress/pg_regress.c#L3)). The driver writes results under `results/` and diffs them against `expected/`. Any difference is saved in `regression.diffs` ([regress.sgml](../raw/postgres-17/doc/src/sgml/regress.sgml#L483-L492)). `parallel_schedule` lists which core tests run together ([parallel_schedule](../raw/postgres-17/src/test/regress/parallel_schedule#L12-L17)).

Related: [Isolation test](#isolation-test), [TAP test](#tap-test)

### REINDEX

**Checked on:** PostgreSQL 17.

`REINDEX` rebuilds an index from its table's data and replaces the old copy. Use it for a corrupt index, a bloated index, a changed storage parameter such as `fillfactor`, or an invalid index left by a failed concurrent build ([ref/reindex.sgml](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L37-L82)). Without `CONCURRENTLY`, `reindex_index()` locks the table with `ShareLock` and the index with `AccessExclusiveLock`. It then gives the index a new relfilenumber with `RelationSetNewRelfilenumber()` and calls `index_build()` ([index.c:3600-3613](../raw/postgres-17/src/backend/catalog/index.c#L3600-L3613), [index.c:3652-3654](../raw/postgres-17/src/backend/catalog/index.c#L3652-L3654), [index.c:3784-3789](../raw/postgres-17/src/backend/catalog/index.c#L3784-L3789)). `REINDEX CONCURRENTLY` takes a different path, `ReindexRelationConcurrently()` ([indexcmds.c#ReindexRelationConcurrently](../raw/postgres-17/src/backend/commands/indexcmds.c#L3428-L3451)).

Related: [CONCURRENTLY](#concurrently), [Invalid index](#invalid-index), [Bloat](#bloat), [Relfilenumber](#relfilenumber), [AccessExclusiveLock](#accessexclusivelock)

### Relcache

**Aliases:** relation cache, `RelationData`, `Relation`, `RelationIdGetRelation`. **Checked on:** PostgreSQL 17.

The relcache is each backend's private cache of relation descriptors. A descriptor is a `RelationData` struct built from the relation's catalog rows ([relcache.c:1-22](../raw/postgres-17/src/backend/utils/cache/relcache.c#L1-L22), [rel.h#RelationData](../raw/postgres-17/src/include/utils/rel.h#L51-L62)). `RelationIdGetRelation` returns the cached entry or builds one, and the caller must already hold a lock on the relation ([relcache.c#RelationIdGetRelation](../raw/postgres-17/src/backend/utils/cache/relcache.c#L2068-L2082)). An entry carries `rd_rel` (the `pg_class` row), `rd_index` for an index, and `rd_indam`, the index access method's routine table ([rel.h:111](../raw/postgres-17/src/include/utils/rel.h#L111), [rel.h:192](../raw/postgres-17/src/include/utils/rel.h#L192), [rel.h:206](../raw/postgres-17/src/include/utils/rel.h#L206)). Catalog changes invalidate entries at command boundaries and, after commit, in other backends ([inval.c:1-35](../raw/postgres-17/src/backend/utils/cache/inval.c#L1-L35)). The relcache caches whole relation descriptors. The [syscache](#syscache) caches individual catalog rows.

Related: [Syscache](#syscache), [pg_class](#pg_class), [pg_index](#pg_index), [Access method](#access-method)

### Relfilenumber

**Aliases:** relfilenode, `RelFileNumber`, `RelFileLocator`, `pg_class.relfilenode`. **Checked on:** PostgreSQL 17.

A relfilenumber is the number in a relation's data file name. It is stored in `pg_class.relfilenode` and is separate from the relation's OID ([relpath.h#RelFileNumber](../raw/postgres-17/src/include/common/relpath.h#L22-L28), [pg_class.h#relfilenode](../raw/postgres-17/src/include/catalog/pg_class.h#L56-L57)). Keeping them separate lets a rewrite swap in new physical files while the relation keeps its OID ([cluster.c#swap_relation_files](../raw/postgres-17/src/backend/commands/cluster.c#L1035-L1040)). `RelFileLocator` combines tablespace OID, database OID and relfilenumber to identify the physical storage. A `relfilenode` of 0 marks a "mapped" catalog, whose file number is kept by `relmapper.c` ([relfilelocator.h#RelFileLocator](../raw/postgres-17/src/include/storage/relfilelocator.h#L33-L63)).

Related: [Fork](#fork), [OID](#oid), [pg_class](#pg_class), [Table rewrite](#table-rewrite)

### RelOptInfo

**Checked on:** PostgreSQL 17.

`RelOptInfo` is the planner's working record for one relation. That relation can be a base table, a join, or an upper-level step such as grouping [pathnodes.h#RelOptKind](../raw/postgres-17/src/include/nodes/pathnodes.h#L819-L827). It holds the candidate paths (`pathlist`), the cheapest startup and total paths, and, for a base table, the list of `IndexOptInfo` nodes [pathnodes.h:663-697](../raw/postgres-17/src/include/nodes/pathnodes.h#L663-L697). It also holds size estimates taken from `pg_class`: `pages`, `tuples`, and `allvisfrac` [pathnodes.h:938-944](../raw/postgres-17/src/include/nodes/pathnodes.h#L938-L944).

Related: [Path](#path), [IndexOptInfo](#indexoptinfo), [Planner](#planner), [reltuples and relpages](#reltuples-and-relpages)

### reltuples and relpages

**Aliases:** `pg_class.reltuples`, `pg_class.relpages`. **Checked on:** PostgreSQL 17.

These two `pg_class` columns hold a relation's estimated row count and page count. Neither is always up to date, and `reltuples = -1` means unknown [pg_class.h:61-66](../raw/postgres-17/src/include/catalog/pg_class.h#L61-L66). `VACUUM`, `ANALYZE`, and some DDL such as `CREATE INDEX` update them. For a table, `reltuples` counts live rows only [catalogs.sgml#relpages](../raw/postgres-17/doc/src/sgml/catalogs.sgml#L2020-L2046) [vacuum.c#vac_update_relstats](../raw/postgres-17/src/backend/commands/vacuum.c#L1404-L1415). The planner does not use `relpages` as the size. It takes the current block count and multiplies it by the density `reltuples / relpages` [tableam.c#table_block_relation_estimate_size](../raw/postgres-17/src/backend/access/table/tableam.c#L666-L667) [tableam.c:711-747](../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747).

Related: [pg_class](#pg_class), [Statistics](#statistics), [RelOptInfo](#reloptinfo), [IndexOptInfo](#indexoptinfo), [VACUUM](#vacuum)

### Replication origin

**Checked on:** PostgreSQL 17.

A replication origin names a source of replicated changes and records how far replay from it has progressed ([origin.c header](../raw/postgres-17/src/backend/replication/logical/origin.c#L13-L38)). Each origin has a two-byte internal id that is stored in WAL. A session that sets an origin marks the changes it produces with it, and output plugins can filter on it to prevent replication loops ([origin.c header](../raw/postgres-17/src/backend/replication/logical/origin.c#L18-L24), [origin.c header](../raw/postgres-17/src/backend/replication/logical/origin.c#L37-L43)). Each apply worker uses an origin named `pg_<suboid>`, and tablesync workers use `pg_<suboid>_<relid>` ([worker.c#ReplicationOriginNameForLogicalRep](../raw/postgres-17/src/backend/replication/logical/worker.c#L420-L442), [worker.c#run_apply_worker](../raw/postgres-17/src/backend/replication/logical/worker.c#L4516-L4525)). With `origin = none`, `pgoutput_origin_filter` drops every change that carries an origin ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L1722-L1735)).

Related: [Apply worker](#apply-worker), [Subscription](#subscription), [Logical decoding](#logical-decoding)

### Replication slot

**Checked on:** PostgreSQL 17.

A replication slot is persistent, crash-safe server state for one replication stream. Its main job is to stop the server from removing WAL or old row versions that the stream still needs ([slot.c header](../raw/postgres-17/src/backend/replication/slot.c#L15-L27)). `ReplicationSlotPersistentData` holds the slot's `xmin` and `catalog_xmin` horizons, its `restart_lsn`, and for logical slots its output plugin ([slot.h#ReplicationSlotPersistentData](../raw/postgres-17/src/include/replication/slot.h#L63-L96)). So a slot that stops advancing holds back both WAL removal and the removal of old row versions ([slot.c header](../raw/postgres-17/src/backend/replication/slot.c#L15-L18)). Each subscription receives its changes through one slot on the publisher ([logical-replication.sgml#logical-replication-subscription](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L197-L199)).

Related: [xmin horizon](#xmin-horizon), [WAL](#wal), [LSN](#lsn), [Logical decoding](#logical-decoding), [Subscription](#subscription)

### Ring buffer

**Aliases:** buffer access strategy, `BufferAccessStrategy`, `BAS_BULKREAD`, `BAS_BULKWRITE`, `BAS_VACUUM`. **Checked on:** PostgreSQL 17.

A ring buffer is a small, private set of shared buffers that one large operation reuses over and over. This stops a big scan, bulk write or VACUUM from pushing everything else out of the cache ([glossary.sgml#glossary-buffer-access-strategy](../raw/postgres-17/doc/src/sgml/glossary.sgml#L272-L297)). `GetAccessStrategy()` picks the default ring size: 256 kB for `BAS_BULKREAD`, 16 MB for `BAS_BULKWRITE` and 2 MB for `BAS_VACUUM`. `BAS_NORMAL` gets no ring ([freelist.c#GetAccessStrategy](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L535-L570)). VACUUM's ring size comes from `vacuum_buffer_usage_limit`, and VACUUM flushes WAL to reuse dirty ring buffers rather than dropping them from the ring ([buffer/README#ring](../raw/postgres-17/src/backend/storage/buffer/README#L205-L235)).

Related: [Buffer manager](#buffer-manager), [Clock sweep](#clock-sweep), [VACUUM](#vacuum)

### Selectivity

**Checked on:** PostgreSQL 17.

Selectivity is the estimated fraction of rows that pass a condition, from 0 to 1. The C type `Selectivity` is a `double` [nodes.h:250](../raw/postgres-17/src/include/nodes/nodes.h#L250). `clauselist_selectivity()` estimates a list of `AND`ed conditions. It applies extended statistics first and multiplies the remaining estimates, even though conditions are often not independent [clausesel.c#clauselist_selectivity](../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L57-L72). When no statistics apply, `selfuncs.h` supplies defaults, such as 0.005 for equality and 1/3 for an inequality [selfuncs.h:33-40](../raw/postgres-17/src/include/utils/selfuncs.h#L33-L40).

Related: [Statistics](#statistics), [Cost](#cost), [Planner](#planner)

### shared_buffers

**Aliases:** shared buffer pool, `NBuffers`. **Checked on:** PostgreSQL 17.

`shared_buffers` sets how many buffers are in the shared buffer pool. Its unit is blocks. The built-in default is 16384 blocks, which is 128 MB at 8 kB `BLCKSZ`. The GUC context is `postmaster`, so a change needs a restart ([guc_tables.c#shared_buffers](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2260-L2270)). In source, the value is the global `NBuffers` ([globals.c:139](../raw/postgres-17/src/backend/utils/init/globals.c#L139)). The clock hand wraps around modulo `NBuffers` ([freelist.c#ClockSweepTick](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L103-L125)). It is PostgreSQL's own cache; the server also relies on the operating system's cache ([config.sgml#guc-shared-buffers](../raw/postgres-17/doc/src/sgml/config.sgml#L1673-L1680)).

Related: [Buffer manager](#buffer-manager), [Clock sweep](#clock-sweep), [Huge pages](#huge-pages)

### ShareUpdateExclusiveLock

**Aliases:** `SHARE UPDATE EXCLUSIVE` lock mode, lock mode 4. **Checked on:** PostgreSQL 17.

A table lock mode that conflicts with itself and with the stronger modes, but not with ordinary reads and writes ([lockdefs.h#ShareUpdateExclusiveLock](../raw/postgres-17/src/include/storage/lockdefs.h#L39-L40), [lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L80)). It protects a table against concurrent schema changes and `VACUUM` runs. Plain `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY`, `CREATE STATISTICS`, `COMMENT ON`, `REINDEX CONCURRENTLY` and some `ALTER INDEX` and `ALTER TABLE` forms take it ([mvcc.sgml#SHARE UPDATE EXCLUSIVE](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L977-L1000)). `vacuum_rel()` chooses it for every VACUUM except `VACUUM FULL` ([vacuum.c:2054-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2054-L2055)).

Related: [Heavyweight lock](#heavyweight-lock), [AccessExclusiveLock](#accessexclusivelock), [CONCURRENTLY](#concurrently), [VACUUM](#vacuum)

### Simple index deletion

**Aliases:** LP_DEAD deletion, "known dead" index entries. **Checked on:** PostgreSQL 17.

Simple index deletion removes index entries that are already marked `LP_DEAD`. An earlier scan sets that flag once it finds the row dead to every transaction ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L511-L534)). In B-tree, it is the first step `_bt_delete_or_dedup_one_page()` tries before a leaf page split. It can also delete nearby entries that turn out to be removable at no extra cost ([nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2703-L2731)). Hash and GiST have their own `LP_DEAD` removal routines ([hashinsert.c#_hash_vacuum_one_page](../raw/postgres-17/src/backend/access/hash/hashinsert.c#L364-L370), [gist.c#gistprunepage](../raw/postgres-17/src/backend/access/gist/gist.c#L1666-L1671)). Unlike bottom-up deletion, it only acts on entries that are already known dead ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L582-L585)).

Related: [Bottom-up index deletion](#bottom-up-index-deletion), [Line pointer](#line-pointer), [Page split](#page-split), [VACUUM](#vacuum)

### Snapshot

**Aliases:** MVCC snapshot, `SnapshotData`. **Checked on:** PostgreSQL 17.

A record of which transactions count as finished for one statement or transaction. It decides which row versions that statement sees ([snapshot.h#SNAPSHOT_MVCC](../raw/postgres-17/src/include/utils/snapshot.h#L37-L50)). An MVCC snapshot holds `xmin` (every older xid is settled), `xmax` (every xid at or above it is invisible), and the in-progress list `xip` ([snapshot.h#SnapshotData](../raw/postgres-17/src/include/utils/snapshot.h#L142-L166)). `GetSnapshotData()` builds it from the proc array ([procarray.c#GetSnapshotData](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2177)). Other snapshot types exist, such as `SNAPSHOT_ANY` and `SNAPSHOT_NON_VACUUMABLE`, which obey different rules ([snapshot.h](../raw/postgres-17/src/include/utils/snapshot.h#L66-L69)). A snapshot's `xmin` is a visibility bound, not the per-row `t_xmin` stamp ([snapshot.h#SnapshotData](../raw/postgres-17/src/include/utils/snapshot.h#L156-L157), [htup_details.h#HeapTupleFields](../raw/postgres-17/src/include/access/htup_details.h#L122-L132)).

Related: [MVCC](#mvcc), [xmin and xmax](#xmin-and-xmax), [xmin horizon](#xmin-horizon), [Transaction ID](#transaction-id)

### SP-GiST

**Aliases:** space-partitioned GiST. **Checked on:** PostgreSQL 17.

SP-GiST is a framework for index structures that are not balanced and that split the key space into regions, such as quadtrees, k-d trees and radix trees ([spgist/README](../raw/postgres-17/src/backend/access/spgist/README#L1-L12)). An SP-GiST tree has inner tuples, which hold labeled child pointers, and leaf tuples. The two kinds are kept on separate pages, and branches may have different depths ([spgist/README](../raw/postgres-17/src/backend/access/spgist/README#L14-L24)). `spghandler()` marks it as supporting ordered searches, but not unique or multicolumn indexes. Block 0 is its metapage ([spgutils.c#spghandler](../raw/postgres-17/src/backend/access/spgist/spgutils.c#L44-L55), [spgist_private.h:47](../raw/postgres-17/src/include/access/spgist_private.h#L47)).

Related: [GiST](#gist), [Access method](#access-method), [Metapage](#metapage)

### SPI

**Aliases:** Server Programming Interface. **Checked on:** PostgreSQL 17.

SPI lets C code running inside the server run SQL through the parser, planner, and executor [spi.sgml](../raw/postgres-17/doc/src/sgml/spi.sgml#L10-L18). Most procedural languages, including PL/pgSQL, run their SQL through it [spi.sgml](../raw/postgres-17/doc/src/sgml/spi.sgml#L20-L27). Code calls `SPI_connect()`, runs statements with functions such as `SPI_execute()`, and finishes with `SPI_finish()` [spi.c#SPI_connect](../raw/postgres-17/src/backend/executor/spi.c#L94) [spi.c#SPI_execute](../raw/postgres-17/src/backend/executor/spi.c#L594-L614) [spi.c#SPI_finish](../raw/postgres-17/src/backend/executor/spi.c#L182).

Related: [PL/pgSQL](#plpgsql), [Executor](#executor), [Planner](#planner)

### Statistics

**Aliases:** planner statistics, `pg_statistic`, `pg_stats`, `ANALYZE` statistics. **Checked on:** PostgreSQL 17.

Planner statistics are per-column summaries of a table's data that `ANALYZE` samples. They include the null fraction, the average width, the number of distinct values, most-common values, and histograms [pg_statistic.h#pg_statistic](../raw/postgres-17/src/include/catalog/pg_statistic.h#L29-L69). `do_analyze_rel()` computes them and `update_attstats()` stores them in `pg_statistic`. The readable `pg_stats` view exposes them [analyze.c#do_analyze_rel](../raw/postgres-17/src/backend/commands/analyze.c#L272-L283) [analyze.c:593-601](../raw/postgres-17/src/backend/commands/analyze.c#L593-L601) [system_views.sql#pg_stats](../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L200). The planner reads them through `examine_variable()` when it estimates selectivity [selfuncs.c#examine_variable](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5037-L5040). Do not confuse these with cumulative statistics, the `pg_stat_*` activity counters.

Related: [Selectivity](#selectivity), [reltuples and relpages](#reltuples-and-relpages), [Cumulative statistics](#cumulative-statistics), [Catalog](#catalog)

### Subscription

**Checked on:** PostgreSQL 17.

A subscription is the downstream side of logical replication. It names a connection to a publisher and the publications to receive ([logical-replication.sgml#logical-replication-subscription](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L176-L183)). The shared catalog `pg_subscription` stores the connection string, the publisher-side slot name, the publication list, and `suborigin` ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-17/src/include/catalog/pg_subscription.h#L100-L117)). Its `origin` option chooses between receiving every change (`any`, the default) and only changes that carry no origin (`none`) ([create_subscription.sgml](../raw/postgres-17/doc/src/sgml/ref/create_subscription.sgml#L400-L411)). `CREATE SUBSCRIPTION` runs `CreateSubscription` ([subscriptioncmds.c#CreateSubscription](../raw/postgres-17/src/backend/commands/subscriptioncmds.c#L605)).

Related: [Publication](#publication), [Apply worker](#apply-worker), [Replication origin](#replication-origin), [Replication slot](#replication-slot)

### Syscache

**Aliases:** system cache, catcache, `SearchSysCache1`, `MAKE_SYSCACHE`. **Checked on:** PostgreSQL 17.

The syscache is a per-backend cache of individual catalog rows, looked up by key. The parser, planner, and executor use it for fast catalog access ([syscache.c:13-17](../raw/postgres-17/src/backend/utils/cache/syscache.c#L13-L17)). Each cache is declared next to its catalog with `MAKE_SYSCACHE`, for example `RELOID` on `pg_class_oid_index`. `genbki.pl` generates the cache ID list from those declarations ([pg_class.h:159-160](../raw/postgres-17/src/include/catalog/pg_class.h#L159-L160), [genbki.h:123-127](../raw/postgres-17/src/include/catalog/genbki.h#L123-L127), [genbki.pl:449-453](../raw/postgres-17/src/backend/catalog/genbki.pl#L449-L453)). Callers look up a row with `SearchSysCache1` and friends and must release it with `ReleaseSysCache` ([syscache.c#SearchSysCache](../raw/postgres-17/src/backend/utils/cache/syscache.c#L200-L230)).

Related: [Relcache](#relcache), [Catalog](#catalog), [BKI](#bki)

### Table rewrite

**Aliases:** heap rewrite, `make_new_heap`, `finish_heap_swap`. **Checked on:** PostgreSQL 17.

A table rewrite copies every live row into a brand-new set of files and then swaps those files in for the old ones. `CLUSTER`, `VACUUM FULL` and some forms of `ALTER TABLE` use it. `make_new_heap()` creates the transient table, the caller fills it, and `finish_heap_swap()` swaps the files and rebuilds all indexes ([cluster.c#make_new_heap](../raw/postgres-17/src/backend/commands/cluster.c#L676-L690), [cluster.c#finish_heap_swap](../raw/postgres-17/src/backend/commands/cluster.c#L1433-L1440)). In `ALTER TABLE`, `ATRewriteTables()` fires the `table_rewrite` event trigger and then runs that same sequence ([tablecmds.c#ATRewriteTables](../raw/postgres-17/src/backend/commands/tablecmds.c#L5834-L5878)). Four reasons can cause an `ALTER TABLE` rewrite: persistence change, a default value, a column rewrite, or an access method change ([event_trigger.h#AT_REWRITE](../raw/postgres-17/src/include/commands/event_trigger.h#L36-L43)). The swap exchanges the two relations' relfilenumbers, so the table keeps its OID but gets new files ([cluster.c#swap_relation_files](../raw/postgres-17/src/backend/commands/cluster.c#L1035-L1040)).

Related: [CLUSTER](#cluster), [Event trigger](#event-trigger), [Relfilenumber](#relfilenumber), [VACUUM FULL](#vacuum-full)

### TAP test

**Aliases:** Perl TAP test, `prove` test. **Checked on:** PostgreSQL 17.

A TAP test is a Perl script, usually `t/*.pl`, that the `prove` tool runs. Client-program tests under `src/bin` and many multi-node scenarios are written this way ([regress.sgml#regress-tap](../raw/postgres-17/doc/src/sgml/regress.sgml#L796-L820)). These tests run only when the build was configured with `--enable-tap-tests` ([regress.sgml#regress-tap](../raw/postgres-17/doc/src/sgml/regress.sgml#L827-L828)). Scripts drive throwaway servers through `PostgreSQL::Test::Cluster`, which runs `initdb`, starts and restarts nodes, and runs `psql` ([Cluster.pm](../raw/postgres-17/src/test/perl/PostgreSQL/Test/Cluster.pm#L8-L39)).

Related: [Regression test](#regression-test), [Isolation test](#isolation-test), [Injection point](#injection-point)

### TID

**Aliases:** tuple identifier, item pointer, `ItemPointerData`, `ctid`. **Checked on:** PostgreSQL 17.

A TID is the physical address of a tuple: a block number plus a line-pointer number on that page. The struct is `ItemPointerData`, six bytes with `ip_blkid` and `ip_posid` ([itemptr.h#ItemPointerData](../raw/postgres-17/src/include/storage/itemptr.h#L20-L40)). Index entries store TIDs to point at heap rows. Each heap tuple header stores one in `t_ctid`, which points to the tuple itself or to its newer version after an update ([htup_details.h#t_ctid](../raw/postgres-17/src/include/access/htup_details.h#L86-L99)). SQL shows the TID as the `ctid` system column ([heap.c#ctid](../raw/postgres-17/src/backend/catalog/heap.c#L143-L147)). An update stores the new version at a new TID and links the old version to it, so a TID is not a stable row identity ([htup_details.h#t_ctid](../raw/postgres-17/src/include/access/htup_details.h#L86-L89)).

Related: [Block](#block), [HOT](#hot), [Line pointer](#line-pointer), [Tuple](#tuple)

### TOAST

**Aliases:** The Oversized-Attribute Storage Technique, TOAST table, `reltoastrelid`. **Checked on:** PostgreSQL 17.

TOAST stores large column values outside the main row. It compresses them and splits them into chunks in a separate TOAST table that belongs to the main table ([glossary.sgml#glossary-toast](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1857-L1870)). `pg_class.reltoastrelid` links a table to its TOAST table ([pg_class.h:72](../raw/postgres-17/src/include/catalog/pg_class.h#L72)). On insert, `heap_prepare_insert()` calls `heap_toast_insert_or_update()` when a table or materialized-view tuple is longer than `TOAST_TUPLE_THRESHOLD` or already has external values ([heapam.c#heap_prepare_insert](../raw/postgres-17/src/backend/access/heap/heapam.c#L2348-L2363)). That threshold is the largest tuple size that lets four tuples fit on one heap page, so it scales with `BLCKSZ` ([heaptoast.h#TOAST_TUPLE_THRESHOLD](../raw/postgres-17/src/include/access/heaptoast.h#L38-L50)).

Related: [BLCKSZ](#blcksz), [Heap](#heap), [Tuple](#tuple)

### Transaction ID

**Aliases:** xid, `TransactionId`, `FullTransactionId`. **Checked on:** PostgreSQL 17.

A 32-bit number a transaction receives the first time it needs one, typically when it first writes a row ([c.h:669](../raw/postgres-17/src/include/c.h#L669), [transam/README](../raw/postgres-17/src/backend/access/transam/README#L193-L195)). Values 0 to 2 are reserved: invalid, bootstrap and frozen. Normal xids start at 3 ([transam.h](../raw/postgres-17/src/include/access/transam.h#L31-L35)). Normal xids compare modulo 2^32, so "older" means within half the number space ([transam.c#TransactionIdPrecedes](../raw/postgres-17/src/backend/access/transam/transam.c#L276-L293)). `FullTransactionId` adds a 32-bit epoch, making a 64-bit value that never wraps ([transam.h#FullTransactionId](../raw/postgres-17/src/include/access/transam.h#L60-L68), [glossary.sgml#xid](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1890-L1905)).

Related: [xmin and xmax](#xmin-and-xmax), [Wraparound](#wraparound), [Freezing](#freezing), [MultiXact](#multixact)

### Truncation

**Aliases:** relation truncation, vacuum truncation, `lazy_truncate_heap`. **Checked on:** PostgreSQL 17.

In this wiki, truncation means VACUUM cutting empty pages off the end of a table and giving that disk space back to the operating system. VACUUM tries it only when at least 1000 pages, or one-sixteenth of the table, at the end could be freed. It skips truncation when the wraparound failsafe is active ([vacuumlazy.c#should_attempt_truncation](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2510-L2544), [vacuumlazy.c#REL_TRUNCATE_MINIMUM](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L62-L72)). `lazy_truncate_heap()` needs `AccessExclusiveLock`. It never queues for it: it retries a conditional lock every 50 ms, and after 5 s it gives up truncating ([vacuumlazy.c#lazy_truncate_heap](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2572-L2600), [vacuumlazy.c#VACUUM_TRUNCATE_LOCK_TIMEOUT](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L82-L83)). `RelationTruncate()` does the physical cut and drops the buffers for removed blocks ([storage.c#RelationTruncate](../raw/postgres-17/src/backend/catalog/storage.c#L280-L289)). The `vacuum_truncate` storage option can turn it off per table ([rel.h#StdRdOptions](../raw/postgres-17/src/include/utils/rel.h#L345-L346)). This is different from the SQL `TRUNCATE` command, which removes all rows from a table ([ref/truncate.sgml#description](../raw/postgres-17/doc/src/sgml/ref/truncate.sgml#L33)).

Related: [AccessExclusiveLock](#accessexclusivelock), [VACUUM](#vacuum), [Wraparound](#wraparound)

### Tuple

**Aliases:** row, row version, heap tuple, `HeapTupleHeaderData`. **Checked on:** PostgreSQL 17.

A tuple is an ordered set of attribute values. When a table defines the order it is usually called a row ([glossary.sgml#glossary-tuple](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1948-L1960)). An `UPDATE` writes a new tuple rather than changing the old one in place, and the old version's `t_ctid` points to the new one. So one logical row can have several tuple versions on disk ([htup_details.h#t_ctid](../raw/postgres-17/src/include/access/htup_details.h#L86-L94)). Every heap tuple starts with a 23-byte `HeapTupleHeaderData`. It holds transaction fields (`t_choice`), `t_ctid`, `t_infomask2`, `t_infomask`, the header length `t_hoff`, and a null bitmap ([htup_details.h#HeapTupleHeaderData](../raw/postgres-17/src/include/access/htup_details.h#L153-L181)). Index tuples use a different header, `IndexTupleData`: a heap TID plus a 16-bit `t_info` word ([itup.h#IndexTupleData](../raw/postgres-17/src/include/access/itup.h#L22-L50)).

Related: [Heap](#heap), [MVCC](#mvcc), [TID](#tid), [xmin and xmax](#xmin-and-xmax)

### Utility command

**Aliases:** utility statement, non-optimizable statement, `ProcessUtility`. **Checked on:** PostgreSQL 17.

A utility command is any statement the planner does not optimize [parsenodes.h#Query](../raw/postgres-17/src/include/nodes/parsenodes.h#L101-L107). Its parse tree travels in `Query.utilityStmt`. `pg_plan_queries()` wraps it in a `PlannedStmt` without calling the planner [postgres.c#pg_plan_queries](../raw/postgres-17/src/backend/tcop/postgres.c#L987-L997). Execution goes to `ProcessUtility()`, which calls `ProcessUtility_hook` if an extension set one and `standard_ProcessUtility()` otherwise [utility.c#ProcessUtility](../raw/postgres-17/src/backend/tcop/utility.c#L498-L525). Commands that support event triggers continue to `ProcessUtilitySlow` [utility.c:527-537](../raw/postgres-17/src/backend/tcop/utility.c#L527-L537).

Related: [Parse tree](#parse-tree), [PlannedStmt](#plannedstmt), [Hook](#hook), [Event trigger](#event-trigger), [Portal](#portal)

### VACUUM FULL

**Checked on:** PostgreSQL 17.

`VACUUM FULL` rewrites the whole table into a new file that contains only live rows, then rebuilds its indexes. It is a different operation from plain [VACUUM](#vacuum), not a stronger form of it. `vacuum_rel` takes `AccessExclusiveLock` for it and calls `cluster_rel` with no index ([vacuum.c:2049-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2055), [vacuum.c:2248-2260](../raw/postgres-17/src/backend/commands/vacuum.c#L2248-L2260)). `cluster_rel` then rewrites the table in physical order rather than index order ([cluster.c#cluster_rel](../raw/postgres-17/src/backend/commands/cluster.c#L306-L308)). The lock blocks readers and writers until commit ([mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1067-L1080), [vacuum.c:2252](../raw/postgres-17/src/backend/commands/vacuum.c#L2252)). The table keeps its OID but gets new files, because the rewrite swaps relfilenumbers ([cluster.c#cluster_rel](../raw/postgres-17/src/backend/commands/cluster.c#L293-L299)).

Related: [CLUSTER](#cluster), [Table rewrite](#table-rewrite), [AccessExclusiveLock](#accessexclusivelock), [Relfilenumber](#relfilenumber), [Bloat](#bloat)

### VACUUM

**Aliases:** lazy vacuum, plain VACUUM, `heap_vacuum_rel`. **Checked on:** PostgreSQL 17.

VACUUM removes dead row versions so their space can be reused, and does related [MVCC](#mvcc) upkeep such as [freezing](#freezing) ([glossary.sgml#glossary-vacuum](../raw/postgres-17/doc/src/sgml/glossary.sgml#L2056-L2069)). Plain VACUUM takes only `ShareUpdateExclusiveLock`, so normal reads and writes continue while it runs ([vacuum.c:2049-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2055)). For a heap table it runs `heap_vacuum_rel` ([heapam_handler.c:2632](../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L2632)). That function scans the heap to prune pages and collect dead TIDs. It then removes the matching index entries, and finally marks the heap's `LP_DEAD` slots unused. The dead TIDs are held in memory bounded by `maintenance_work_mem`, or `autovacuum_work_mem` in an autovacuum worker when that is set ([vacuumlazy.c:2826-2828](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2826-L2828), [vacuumlazy.c:1-21](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1-L21), [vacuumlazy.c#lazy_scan_heap](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L779-L816)).

Related: [Autovacuum](#autovacuum), [VACUUM FULL](#vacuum-full), [Pruning](#pruning), [Line pointer](#line-pointer), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [maintenance_work_mem](#maintenance_work_mem), [Visibility map](#visibility-map)

### Visibility map

**Aliases:** VM, `VISIBILITYMAP_FORKNUM`, all-visible, all-frozen. **Checked on:** PostgreSQL 17.

The visibility map is a fork with two bits per heap page ([glossary.sgml#glossary-vm](../raw/postgres-17/doc/src/sgml/glossary.sgml#L2093-L2105), [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-17/src/include/access/visibilitymapdefs.h#L20-L21)):

- *all-visible* means every tuple on the page is visible to all transactions.
- *all-frozen* means every tuple on the page is frozen.

VACUUM uses the bits to skip pages. A set bit is always true. A clear bit only means "not known". Setting a bit is WAL-logged, and clearing one is replayed as part of the change that caused it ([visibilitymap.c#NOTES](../raw/postgres-17/src/backend/access/heap/visibilitymap.c#L22-L50)). An index-only scan visits the heap only when the page's all-visible bit is clear ([nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L161-L168)). Do not confuse it with the free space map, which tracks free space ([glossary.sgml#glossary-fsm](../raw/postgres-17/doc/src/sgml/glossary.sgml#L814-L828)).

Related: [Fork](#fork), [Freezing](#freezing), [Free space map](#free-space-map), [Index-only scan](#index-only-scan), [VACUUM](#vacuum)

### Wait event

**Checked on:** PostgreSQL 17.

A wait event names what a server process is waiting for right now. Its class says what kind of wait it is, such as an LWLock, a heavyweight lock, a buffer pin, IPC, a timeout or I/O ([wait_event.h:18-27](../raw/postgres-17/src/include/utils/wait_event.h#L18-L27)). `pg_stat_activity` shows it in the `wait_event_type` and `wait_event` columns ([system_views.sql#pg_stat_activity](../raw/postgres-17/src/backend/catalog/system_views.sql#L865-L882)). Code brackets each wait with `pgstat_report_wait_start()` and `pgstat_report_wait_end()`. The start call writes one 4-byte value that encodes the class and the event ([wait_event.h#pgstat_report_wait_start](../raw/postgres-17/src/include/utils/wait_event.h#L65-L92)). `wait_event_names.txt` lists every event. The build generates the enum header, the name lookup code, and the documentation tables from it ([wait_event_names.txt](../raw/postgres-17/src/backend/utils/activity/wait_event_names.txt#L7-L24)).

Related: [Cumulative statistics](#cumulative-statistics), [LWLock](#lwlock), [Heavyweight lock](#heavyweight-lock)

### WAL

**Aliases:** write-ahead log, XLOG, xlog. **Checked on:** PostgreSQL 17.

The sequential journal of every change to the cluster. It is written before the changed data pages, so replaying it after a crash restores a consistent state ([glossary.sgml#wal](../raw/postgres-17/doc/src/sgml/glossary.sgml#L2269-L2279), [transam/README](../raw/postgres-17/src/backend/access/transam/README#L402-L415)). Source code calls it XLOG ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L402)). A WAL-logged change follows a fixed recipe: lock the buffer, enter a critical section, change the page, `MarkBufferDirty()`, `XLogBeginInsert()` and `XLogRegister*()`, `XLogInsert()`, then `PageSetLSN()` ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L437-L466)). The same stream also serves point-in-time recovery and hot-standby replication through log shipping ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L402-L404)).

Related: [LSN](#lsn), [Checkpoint](#checkpoint), [Full-page image](#full-page-image), [Crash recovery](#crash-recovery), [Critical section](#critical-section), [Logical decoding](#logical-decoding)

### work_mem

**Checked on:** PostgreSQL 17.

`work_mem` caps the memory each sort or hash table in a query may use before it spills to temporary files. Its context is `user`, so a session or transaction can `SET` it with no reload or restart. Its default is 4096 kB (4 MB). The limit applies to each sort operation or hash table, not to each query [guc_tables.c#work_mem](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2448-L2457). Sorts receive it as their `workMem` budget [tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L37). Hash-based nodes may use `work_mem * hash_mem_multiplier` [nodeHash.c#get_hash_memory_limit](../raw/postgres-17/src/backend/executor/nodeHash.c#L3602-L3613).

Related: [maintenance_work_mem](#maintenance_work_mem), [GUC context](#guc-context), [Executor](#executor)

### Wraparound

**Aliases:** transaction ID wraparound, XID wraparound. **Checked on:** PostgreSQL 17.

The danger that 32-bit transaction IDs run all the way around. Normal xids compare modulo 2^32, so an unfrozen xid more than half the space old would start to compare as newer than current ones ([transam.c#TransactionIdPrecedes](../raw/postgres-17/src/backend/access/transam/transam.c#L276-L293)). `SetTransactionIdLimit()` puts the hard limit halfway around from the oldest unfrozen xid in any database ([varsup.c#SetTransactionIdLimit](../raw/postgres-17/src/backend/access/transam/varsup.c#L382-L391)). It then sets three thresholds below that limit ([varsup.c#SetTransactionIdLimit](../raw/postgres-17/src/backend/access/transam/varsup.c#L405-L440)):

- At `xidVacLimit`, forced autovacuums begin.
- 40 million xids before the limit, warnings begin.
- 3 million xids before the limit, the server refuses to assign new xids.

`GetNewTransactionId()` enforces the refusal ([varsup.c#GetNewTransactionId](../raw/postgres-17/src/backend/access/transam/varsup.c#L147-L159)). `autovacuum_freeze_max_age` is a `postmaster` GUC, so changing it needs a restart ([guc_tables.c#autovacuum_freeze_max_age](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3377-L3384)).

Related: [Transaction ID](#transaction-id), [Freezing](#freezing), [Autovacuum](#autovacuum), [MultiXact](#multixact)

### xmin and xmax

**Aliases:** `t_xmin`, `t_xmax`. **Checked on:** PostgreSQL 17.

The two transaction IDs stored in every heap tuple header. `t_xmin` is the transaction that inserted the version; `t_xmax` is the one that deleted or locked it, or 0 if none ([htup_details.h#HeapTupleFields](../raw/postgres-17/src/include/access/htup_details.h#L122-L132)). Infomask bits change how `t_xmax` reads: it may be only a row locker (`HEAP_XMAX_LOCK_ONLY`) or a MultiXactId (`HEAP_XMAX_IS_MULTI`) ([htup_details.h](../raw/postgres-17/src/include/access/htup_details.h#L196-L209)). A snapshot also has fields named `xmin` and `xmax`, but they are visibility bounds, not per-row stamps ([snapshot.h#SnapshotData](../raw/postgres-17/src/include/utils/snapshot.h#L156-L157)).

Related: [MVCC](#mvcc), [Snapshot](#snapshot), [Transaction ID](#transaction-id), [MultiXact](#multixact), [Tuple](#tuple)

### xmin horizon

**Aliases:** OldestXmin, removable cutoff, `GlobalVisState`. **Checked on:** PostgreSQL 17.

The oldest transaction ID that some session or replication slot might still need to see. Rows deleted by transactions older than it are dead to everyone and can be removed ([procarray.c#GlobalVisState](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L102-L122)). `ComputeXidHorizons()` computes separate horizons for shared, catalog, user-data and temporary relations. Replication-slot xmins hold back all but the temporary one ([procarray.c](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L128-L158), [procarray.c#ComputeXidHorizonsResult](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L234-L237)). VACUUM reads it as `cutoffs->OldestXmin` ([vacuum.c:1120](../raw/postgres-17/src/backend/commands/vacuum.c#L1120)). So an old snapshot in any session of the same database, or an old slot `xmin`, keeps VACUUM from removing rows deleted after it ([procarray.c](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L145-L151)).

Related: [Snapshot](#snapshot), [MVCC](#mvcc), [Pruning](#pruning), [VACUUM](#vacuum), [Replication slot](#replication-slot), [Bloat](#bloat)

## Open Questions

- Version coverage: no entry has been checked against PostgreSQL 12, 14, 18 or 19 yet. Those checkouts need their own review before an entry gains a version qualification for them.
- COMMENT ON: the entry states only what `REINDEX CONCURRENTLY` does with an index comment. It does not state what plain `REINDEX` does, because no separate citation was gathered for that path.

## Source References

One representative citation per cited source file, all from PostgreSQL 17:

- [configure.ac#blocksize](../raw/postgres-17/configure.ac#L258-L289)
- [README:1-9](../raw/postgres-17/contrib/README#L1-L9)
- [ginfuncs.c header](../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L1-L3)
- [heapfuncs.c header](../raw/postgres-17/contrib/pageinspect/heapfuncs.c#L1-L4)
- [pageinspect--1.5--1.6.sql](../raw/postgres-17/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88-L90)
- [pageinspect.control](../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L5)
- [rawpage.c#get_raw_page_internal](../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L153)
- [pg_freespacemap--1.1.sql](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L20)
- [pg_freespacemap.c#pg_freespace](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L48)
- [pg_freespacemap.control](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.control#L1-L2)
- [pg_stat_statements.c:479-480](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L479-L480)
- [pgstatapprox.c#pgstattuple_approx](../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L227)
- [pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L250)
- [pgstattuple--1.4--1.5.sql](../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)
- [pgstattuple.c#pgstattuple_type](../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L55-L63)
- [pgstattuple.control](../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L2)
- [auto-explain.sgml:18-23](../raw/postgres-17/doc/src/sgml/auto-explain.sgml#L18-L23)
- [bki.sgml:40-52](../raw/postgres-17/doc/src/sgml/bki.sgml#L40-L52)
- [catalogs.sgml#relpages](../raw/postgres-17/doc/src/sgml/catalogs.sgml#L2020-L2046)
- [config.sgml#guc-huge-pages](../raw/postgres-17/doc/src/sgml/config.sgml#L1724-L1726)
- [contrib.sgml:7-15](../raw/postgres-17/doc/src/sgml/contrib.sgml#L7-L15)
- [datatype.sgml:4767-4781](../raw/postgres-17/doc/src/sgml/datatype.sgml#L4767-L4781)
- [event-trigger.sgml:10-16](../raw/postgres-17/doc/src/sgml/event-trigger.sgml#L10-L16)
- [extend.sgml:525-540](../raw/postgres-17/doc/src/sgml/extend.sgml#L525-L540)
- [gin.sgml](../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L528)
- [gist.sgml#gist-intro](../raw/postgres-17/doc/src/sgml/gist.sgml#L11-L25)
- [glossary.sgml#glossary-bloat](../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)
- [indexam.sgml#intro](../raw/postgres-17/doc/src/sgml/indexam.sgml#L38-L43)
- [indices.sgml#indexes-expressional](../raw/postgres-17/doc/src/sgml/indices.sgml#L732-L760)
- [installation.sgml#configure-option-enable-injection-points](../raw/postgres-17/doc/src/sgml/installation.sgml#L1657-L1670)
- [logical-replication.sgml](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L6-L13)
- [maintenance.sgml#routine-reindex](../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046)
- [monitoring.sgml#monitoring-stats](../raw/postgres-17/doc/src/sgml/monitoring.sgml#L137-L145)
- [mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1067-L1090)
- [plpgsql.sgml:4](../raw/postgres-17/doc/src/sgml/plpgsql.sgml#L4)
- [ref/cluster.sgml#description](../raw/postgres-17/doc/src/sgml/ref/cluster.sgml#L45-L49)
- [ref/create_index.sgml](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L612-L643)
- [create_subscription.sgml](../raw/postgres-17/doc/src/sgml/ref/create_subscription.sgml#L400-L411)
- [ref/create_table.sgml#fillfactor](../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1468-L1476)
- [ref/explain.sgml#BUFFERS](../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L180-L206)
- [pgupgrade.sgml:39-58](../raw/postgres-17/doc/src/sgml/ref/pgupgrade.sgml#L39-L58)
- [ref/reindex.sgml#bloated](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L62)
- [ref/truncate.sgml#description](../raw/postgres-17/doc/src/sgml/ref/truncate.sgml#L33)
- [regress.sgml](../raw/postgres-17/doc/src/sgml/regress.sgml#L483-L492)
- [spi.sgml](../raw/postgres-17/doc/src/sgml/spi.sgml#L10-L18)
- [system-views.sgml:3346-3438](../raw/postgres-17/doc/src/sgml/system-views.sgml#L3346-L3438)
- [wal.sgml](../raw/postgres-17/doc/src/sgml/wal.sgml#L495-L509)
- [meson.build#bison_kw](../raw/postgres-17/meson.build#L353-L356)
- [brin/README](../raw/postgres-17/src/backend/access/brin/README#L1-L13)
- [brin.c:289](../raw/postgres-17/src/backend/access/brin/brin.c#L289)
- [reloptions.c#fillfactor](../raw/postgres-17/src/backend/access/common/reloptions.c#L174-L194)
- [gin/README](../raw/postgres-17/src/backend/access/gin/README#L8-L26)
- [ginfast.c](../raw/postgres-17/src/backend/access/gin/ginfast.c#L1-L7)
- [gininsert.c#buildFreshLeafTuple](../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L165)
- [ginpostinglist.c](../raw/postgres-17/src/backend/access/gin/ginpostinglist.c#L23-L42)
- [ginutil.c:79](../raw/postgres-17/src/backend/access/gin/ginutil.c#L79)
- [gist/README](../raw/postgres-17/src/backend/access/gist/README#L1-L25)
- [gist.c#gisthandler](../raw/postgres-17/src/backend/access/gist/gist.c#L59-L70)
- [hash/README](../raw/postgres-17/src/backend/access/hash/README#L13-L28)
- [hash.c#hashhandler](../raw/postgres-17/src/backend/access/hash/hash.c#L57-L68)
- [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-17/src/backend/access/hash/hashinsert.c#L364-L370)
- [README.HOT#intro](../raw/postgres-17/src/backend/access/heap/README.HOT#L1-L60)
- [heapam.c#heap_update](../raw/postgres-17/src/backend/access/heap/heapam.c#L4138-L4160)
- [heapam_handler.c#heap_tableam_handler](../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L2652-L2662)
- [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L937-L960)
- [pruneheap.c#heap_page_prune_opt](../raw/postgres-17/src/backend/access/heap/pruneheap.c#L180-L230)
- [vacuumlazy.c#lazy_vacuum](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L793-L800)
- [visibilitymap.c#NOTES](../raw/postgres-17/src/backend/access/heap/visibilitymap.c#L22-L50)
- [amapi.c#GetIndexAmRoutine](../raw/postgres-17/src/backend/access/index/amapi.c#L24-L46)
- [nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L1-L28)
- [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L285-L307)
- [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2781)
- [nbtpage.c#_bt_upgrademetapage](../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L106-L126)
- [nbtree.c#bthandler](../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L96-L117)
- [nbtsort.c:370-375](../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L370-L375)
- [nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L86-L102)
- [nbtutils.c#_bt_allequalimage](../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5140)
- [spgist/README](../raw/postgres-17/src/backend/access/spgist/README#L1-L12)
- [spgutils.c:77](../raw/postgres-17/src/backend/access/spgist/spgutils.c#L77)
- [tableam.c#table_block_relation_estimate_size](../raw/postgres-17/src/backend/access/table/tableam.c#L666-L667)
- [transam/README](../raw/postgres-17/src/backend/access/transam/README#L420-L422)
- [multixact.c](../raw/postgres-17/src/backend/access/transam/multixact.c#L5-L16)
- [transam.c#TransactionIdPrecedes](../raw/postgres-17/src/backend/access/transam/transam.c#L276-L293)
- [varsup.c:443](../raw/postgres-17/src/backend/access/transam/varsup.c#L443)
- [xlog.c#CreateCheckPoint](../raw/postgres-17/src/backend/access/transam/xlog.c#L6827-L6862)
- [xloginsert.c#XLogRecordAssemble](../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L620)
- [xlogrecovery.c#PerformWalRecovery](../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L1656-L1662)
- [bootparse.y:4-5](../raw/postgres-17/src/backend/bootstrap/bootparse.y#L4-L5)
- [catalog.c#GetNewOidWithIndex](../raw/postgres-17/src/backend/catalog/catalog.c#L396-L421)
- [genbki.pl:1-7](../raw/postgres-17/src/backend/catalog/genbki.pl#L1-L7)
- [heap.c#ctid](../raw/postgres-17/src/backend/catalog/heap.c#L143-L147)
- [index.c#index_set_state_flags](../raw/postgres-17/src/backend/catalog/index.c#L3469-L3503)
- [storage.c#RelationTruncate](../raw/postgres-17/src/backend/catalog/storage.c#L280-L289)
- [system_views.sql#pg_stats](../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L200)
- [analyze.c#do_analyze_rel](../raw/postgres-17/src/backend/commands/analyze.c#L272-L283)
- [cluster.c#swap_relation_files](../raw/postgres-17/src/backend/commands/cluster.c#L1035-L1040)
- [comment.c#CreateComments](../raw/postgres-17/src/backend/commands/comment.c#L143-L170)
- [explain.c:1992-1994](../raw/postgres-17/src/backend/commands/explain.c#L1992-L1994)
- [extension.c#ExtensionControlFile](../raw/postgres-17/src/backend/commands/extension.c#L79-L96)
- [functioncmds.c#ExecuteDoStmt](../raw/postgres-17/src/backend/commands/functioncmds.c#L2060-L2066)
- [indexcmds.c:678](../raw/postgres-17/src/backend/commands/indexcmds.c#L678)
- [publicationcmds.c#CreatePublication](../raw/postgres-17/src/backend/commands/publicationcmds.c#L733)
- [subscriptioncmds.c#CreateSubscription](../raw/postgres-17/src/backend/commands/subscriptioncmds.c#L605)
- [tablecmds.c#ATRewriteTables](../raw/postgres-17/src/backend/commands/tablecmds.c#L5834-L5878)
- [vacuum.c:2054-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2054-L2055)
- [execMain.c header](../raw/postgres-17/src/backend/executor/execMain.c#L1-L28)
- [execPartition.c#ExecFindMatchingSubPlans](../raw/postgres-17/src/backend/executor/execPartition.c#L2308)
- [nodeBitmapHeapscan.c](../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L1-L17)
- [nodeHash.c#get_hash_memory_limit](../raw/postgres-17/src/backend/executor/nodeHash.c#L3602-L3613)
- [nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L161-L168)
- [spi.c#SPI_connect](../raw/postgres-17/src/backend/executor/spi.c#L94)
- [tidbitmap.c#tbm_begin_iterate](../raw/postgres-17/src/backend/nodes/tidbitmap.c#L710-L747)
- [clausesel.c#clauselist_selectivity](../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L57-L72)
- [costsize.c header](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6-L30)
- [indxpath.c#check_index_predicates](../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3227-L3244)
- [joinrels.c#try_partitionwise_join](../raw/postgres-17/src/backend/optimizer/path/joinrels.c#L1458-L1477)
- [planner.c:418-422](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L418-L422)
- [pathnode.c#add_path](../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L361-L420)
- [plancat.c:488-494](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L494)
- [relnode.c#build_joinrel_partition_info](../raw/postgres-17/src/backend/optimizer/util/relnode.c#L2016-L2045)
- [parser/README:1-14](../raw/postgres-17/src/backend/parser/README#L1-L14)
- [gram.y:6-7](../raw/postgres-17/src/backend/parser/gram.y#L6-L7)
- [parser/meson.build:30-41](../raw/postgres-17/src/backend/parser/meson.build#L30-L41)
- [parser.c#raw_parser](../raw/postgres-17/src/backend/parser/parser.c#L34-L42)
- [partprune.c header](../raw/postgres-17/src/backend/partitioning/partprune.c#L3-L25)
- [sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-17/src/backend/port/sysv_shmem.c#L597-L636)
- [autovacuum.c:7-27](../raw/postgres-17/src/backend/postmaster/autovacuum.c#L7-L27)
- [bgworker.c#RegisterBackgroundWorker](../raw/postgres-17/src/backend/postmaster/bgworker.c#L854-L862)
- [bgwriter.c](../raw/postgres-17/src/backend/postmaster/bgwriter.c#L5-L13)
- [postmaster.c#BackendStartup](../raw/postgres-17/src/backend/postmaster/postmaster.c#L3525-L3536)
- [decode.c header](../raw/postgres-17/src/backend/replication/logical/decode.c#L3-L8)
- [launcher.c#logicalrep_worker_launch](../raw/postgres-17/src/backend/replication/logical/launcher.c#L476-L485)
- [logical.c header](../raw/postgres-17/src/backend/replication/logical/logical.c#L10-L18)
- [origin.c header](../raw/postgres-17/src/backend/replication/logical/origin.c#L13-L38)
- [worker.c header](../raw/postgres-17/src/backend/replication/logical/worker.c#L10-L16)
- [pgoutput.c header](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L3-L4)
- [slot.c header](../raw/postgres-17/src/backend/replication/slot.c#L15-L27)
- [buffer/README#clock-sweep](../raw/postgres-17/src/backend/storage/buffer/README#L166-L199)
- [bufmgr.c#entry-points](../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L15-L29)
- [freelist.c#ClockSweepTick](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L103-L125)
- [freespace.c#NOTES](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L14-L50)
- [procarray.c#GetSnapshotData](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2177)
- [lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L32-L35)
- [lmgr.c#LockRelationOid](../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L102-L116)
- [lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)
- [lwlock.c#LWLockAcquire](../raw/postgres-17/src/backend/storage/lmgr/lwlock.c#L1170)
- [backend_startup.c#BackendMain](../raw/postgres-17/src/backend/tcop/backend_startup.c#L50-L58)
- [postgres.c#pg_plan_queries](../raw/postgres-17/src/backend/tcop/postgres.c#L976-L1008)
- [pquery.c#PortalRun](../raw/postgres-17/src/backend/tcop/pquery.c#L662-L689)
- [utility.c#ProcessUtility](../raw/postgres-17/src/backend/tcop/utility.c#L498-L525)
- [Gen_fmgrtab.pl:1-6](../raw/postgres-17/src/backend/utils/Gen_fmgrtab.pl#L1-L6)
- [pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L1-L16)
- [wait_event_names.txt](../raw/postgres-17/src/backend/utils/activity/wait_event_names.txt#L7-L24)
- [selfuncs.c:7093-7106](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106)
- [inval.c:1-35](../raw/postgres-17/src/backend/utils/cache/inval.c#L1-L35)
- [relcache.c:1-22](../raw/postgres-17/src/backend/utils/cache/relcache.c#L1-L22)
- [syscache.c:13-17](../raw/postgres-17/src/backend/utils/cache/syscache.c#L13-L17)
- [elog.c#errstart](../raw/postgres-17/src/backend/utils/error/elog.c#L356-L364)
- [dfmgr.c:284-289](../raw/postgres-17/src/backend/utils/fmgr/dfmgr.c#L284-L289)
- [fmgr.c#fmgr_info](../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L113-L130)
- [globals.c:139](../raw/postgres-17/src/backend/utils/init/globals.c#L139)
- [guc_tables.c#block_size](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3267-L3276)
- [injection_point.c header](../raw/postgres-17/src/backend/utils/misc/injection_point.c#L3-L7)
- [mmgr/README:9-49](../raw/postgres-17/src/backend/utils/mmgr/README#L9-L49)
- [tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L37)
- [initdb.c#load_plpgsql](../raw/postgres-17/src/bin/initdb/initdb.c#L1974-L1976)
- [pg_upgrade.c:17-20](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L17-L20)
- [pg_upgrade.h#transferMode](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.h#L253-L260)
- [amapi.h#IndexAmRoutine](../raw/postgres-17/src/include/access/amapi.h#L270-L296)
- [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-17/src/include/access/brin.h#L39)
- [brin_page.h:75](../raw/postgres-17/src/include/access/brin_page.h#L75)
- [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-17/src/include/access/gin_private.h#L39-L45)
- [ginblock.h:52](../raw/postgres-17/src/include/access/ginblock.h#L52)
- [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-17/src/include/access/gist_private.h#L262)
- [hash.h#HASH_METAPAGE](../raw/postgres-17/src/include/access/hash.h#L198)
- [heaptoast.h#MaximumBytesPerTuple](../raw/postgres-17/src/include/access/heaptoast.h#L20-L26)
- [htup_details.h#HEAP_ONLY_TUPLE](../raw/postgres-17/src/include/access/htup_details.h#L274-L282)
- [itup.h#IndexTupleData](../raw/postgres-17/src/include/access/itup.h#L22-L50)
- [nbtree.h#BTGetTargetPageFreeSpace](../raw/postgres-17/src/include/access/nbtree.h#L1144-L1145)
- [spgist_private.h:47](../raw/postgres-17/src/include/access/spgist_private.h#L47)
- [transam.h](../raw/postgres-17/src/include/access/transam.h#L20-L35)
- [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-17/src/include/access/visibilitymapdefs.h#L20-L21)
- [xlogdefs.h#XLogRecPtr](../raw/postgres-17/src/include/access/xlogdefs.h#L17-L21)
- [c.h:678-679](../raw/postgres-17/src/include/c.h#L678-L679)
- [genbki.h:23](../raw/postgres-17/src/include/catalog/genbki.h#L23)
- [index.h#DEFAULT_INDEX_TYPE](../raw/postgres-17/src/include/catalog/index.h#L21)
- [pg_am.dat#heap](../raw/postgres-17/src/include/catalog/pg_am.dat#L15-L17)
- [pg_am.h#FormData_pg_am](../raw/postgres-17/src/include/catalog/pg_am.h#L29-L41)
- [pg_cast.dat:41-42](../raw/postgres-17/src/include/catalog/pg_cast.dat#L41-L42)
- [pg_cast.h#FormData_pg_cast](../raw/postgres-17/src/include/catalog/pg_cast.h#L32-L50)
- [pg_class.h#relfilenode](../raw/postgres-17/src/include/catalog/pg_class.h#L56-L57)
- [pg_description.h:10-19](../raw/postgres-17/src/include/catalog/pg_description.h#L10-L19)
- [pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-17/src/include/catalog/pg_event_trigger.h#L29-L42)
- [pg_index.h#FormData_pg_index](../raw/postgres-17/src/include/catalog/pg_index.h#L48-L61)
- [pg_publication.h#FormData_pg_publication](../raw/postgres-17/src/include/catalog/pg_publication.h#L29-L57)
- [pg_statistic.h#pg_statistic](../raw/postgres-17/src/include/catalog/pg_statistic.h#L29-L69)
- [pg_subscription.h#FormData_pg_subscription](../raw/postgres-17/src/include/catalog/pg_subscription.h#L100-L117)
- [event_trigger.h#AT_REWRITE](../raw/postgres-17/src/include/commands/event_trigger.h#L36-L43)
- [relpath.h#ForkNumber](../raw/postgres-17/src/include/common/relpath.h#L47-L62)
- [instrument.h#BufferUsage](../raw/postgres-17/src/include/executor/instrument.h#L24-L42)
- [fmgr.h:3-8](../raw/postgres-17/src/include/fmgr.h#L3-L8)
- [miscadmin.h#START_CRIT_SECTION](../raw/postgres-17/src/include/miscadmin.h#L150-L156)
- [nodes.h:251](../raw/postgres-17/src/include/nodes/nodes.h#L251)
- [parsenodes.h#Query](../raw/postgres-17/src/include/nodes/parsenodes.h#L101-L110)
- [pathnodes.h#Path](../raw/postgres-17/src/include/nodes/pathnodes.h#L1662-L1665)
- [plannodes.h#PlannedStmt](../raw/postgres-17/src/include/nodes/plannodes.h#L31-L35)
- [cost.h:34](../raw/postgres-17/src/include/optimizer/cost.h#L34)
- [postgres_ext.h:27-31](../raw/postgres-17/src/include/postgres_ext.h#L27-L31)
- [bgworker.h#BackgroundWorker](../raw/postgres-17/src/include/postmaster/bgworker.h#L89-L101)
- [output_plugin.h#OutputPluginCallbacks](../raw/postgres-17/src/include/replication/output_plugin.h#L216-L243)
- [slot.h#ReplicationSlotPersistentData](../raw/postgres-17/src/include/replication/slot.h#L63-L96)
- [block.h#BlockNumber](../raw/postgres-17/src/include/storage/block.h#L17-L35)
- [buf_internals.h#BufferDesc](../raw/postgres-17/src/include/storage/buf_internals.h#L245-L256)
- [bufpage.h#Page](../raw/postgres-17/src/include/storage/bufpage.h#L22-L28)
- [itemid.h#ItemIdData](../raw/postgres-17/src/include/storage/itemid.h#L17-L30)
- [itemptr.h#ItemPointerData](../raw/postgres-17/src/include/storage/itemptr.h#L20-L40)
- [lock.h#LOCKTAG](../raw/postgres-17/src/include/storage/lock.h#L164-L172)
- [lockdefs.h#AccessExclusiveLock](../raw/postgres-17/src/include/storage/lockdefs.h#L45-L47)
- [lwlock.h#LWLock](../raw/postgres-17/src/include/storage/lwlock.h#L41-L50)
- [relfilelocator.h#RelFileLocator](../raw/postgres-17/src/include/storage/relfilelocator.h#L33-L63)
- [utility.h:71-78](../raw/postgres-17/src/include/tcop/utility.h#L71-L78)
- [elog.h:25-56](../raw/postgres-17/src/include/utils/elog.h#L25-L56)
- [guc.h#GucContext](../raw/postgres-17/src/include/utils/guc.h#L36-L76)
- [injection_point.h](../raw/postgres-17/src/include/utils/injection_point.h#L14-L21)
- [portal.h header](../raw/postgres-17/src/include/utils/portal.h#L3-L9)
- [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-17/src/include/utils/rel.h#L348-L349)
- [selfuncs.h:33-40](../raw/postgres-17/src/include/utils/selfuncs.h#L33-L40)
- [snapshot.h#SNAPSHOT_MVCC](../raw/postgres-17/src/include/utils/snapshot.h#L37-L50)
- [wait_event.h:18-27](../raw/postgres-17/src/include/utils/wait_event.h#L18-L27)
- [pl_handler.c#plpgsql_inline_handler](../raw/postgres-17/src/pl/plpgsql/src/pl_handler.c#L313-L330)
- [plpgsql--1.0.sql](../raw/postgres-17/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L15)
- [isolation/README](../raw/postgres-17/src/test/isolation/README#L3-L12)
- [injection_points.control](../raw/postgres-17/src/test/modules/injection_points/injection_points.control#L1-L4)
- [Cluster.pm](../raw/postgres-17/src/test/perl/PostgreSQL/Test/Cluster.pm#L8-L39)
- [parallel_schedule](../raw/postgres-17/src/test/regress/parallel_schedule#L12-L17)
- [pg_regress.c header](../raw/postgres-17/src/test/regress/pg_regress.c#L3)

## Navigation

- [Wiki index](index.md)
- [Versions](versions.md)
- [Overview](overview.md)
- [PostgreSQL 17 index](v17/index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](v17/codebase-navigation-guide.md)
