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
  - [Alignment](#alignment)
  - [allequalimage](#allequalimage)
  - [amcheck](#amcheck)
  - [Apply worker](#apply-worker)
  - [Asynchronous I/O](#asynchronous-io)
  - [Autovacuum](#autovacuum)
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
  - [B-tree](#b-tree)
  - [B-tree page deletion](#b-tree-page-deletion)
  - [btree_gin and btree_gist](#btree_gin-and-btree_gist)
  - [Buffer manager](#buffer-manager)
  - [Catalog](#catalog)
  - [Checkpoint](#checkpoint)
  - [Clock sweep](#clock-sweep)
  - [CLUSTER](#cluster)
  - [Collation](#collation)
  - [COMMENT ON](#comment-on)
  - [Common table expression](#common-table-expression)
  - [CONCURRENTLY](#concurrently)
  - [Conflict detection](#conflict-detection)
  - [Constraint exclusion](#constraint-exclusion)
  - [Contrib](#contrib)
  - [Correlation](#correlation)
  - [Cost](#cost)
  - [Covering index](#covering-index)
  - [Crash recovery](#crash-recovery)
  - [Critical section](#critical-section)
  - [Cumulative statistics](#cumulative-statistics)
  - [Cumulative statistics kind](#cumulative-statistics-kind)
  - [Custom and generic plan](#custom-and-generic-plan)
  - [Data checksums](#data-checksums)
  - [Datum](#datum)
  - [Dead tuple](#dead-tuple)
  - [Deadlock](#deadlock)
  - [Declarative partitioning](#declarative-partitioning)
  - [Deduplication](#deduplication)
  - [Dirty buffer](#dirty-buffer)
  - [Dynamic shared memory](#dynamic-shared-memory)
  - [effective_cache_size](#effective_cache_size)
  - [effective_io_concurrency](#effective_io_concurrency)
  - [ereport](#ereport)
  - [Event trigger](#event-trigger)
  - [Executor](#executor)
  - [Executor state](#executor-state)
  - [EXPLAIN](#explain)
  - [EXPLAIN BUFFERS](#explain-buffers)
  - [Expression index](#expression-index)
  - [Extended statistics](#extended-statistics)
  - [Extension](#extension)
  - [Fast root](#fast-root)
  - [Fillfactor](#fillfactor)
  - [fmgr](#fmgr)
  - [Foreign data wrapper](#foreign-data-wrapper)
  - [Foreign key trigger](#foreign-key-trigger)
  - [Fork](#fork)
  - [Free space map](#free-space-map)
  - [Freezing](#freezing)
  - [fsync](#fsync)
  - [Full-page image](#full-page-image)
  - [Function volatility](#function-volatility)
  - [GIN](#gin)
  - [GiST](#gist)
  - [GiST build method](#gist-build-method)
  - [Grammar](#grammar)
  - [GUC](#guc)
  - [GUC context](#guc-context)
  - [Hash index](#hash-index)
  - [Heap](#heap)
  - [Heavyweight lock](#heavyweight-lock)
  - [Hook](#hook)
  - [HOT](#hot)
  - [Hot standby](#hot-standby)
  - [Huge pages](#huge-pages)
  - [Index scan](#index-scan)
  - [Index vacuuming](#index-vacuuming)
  - [INDEX_CLEANUP](#index_cleanup)
  - [Index-only scan](#index-only-scan)
  - [IndexOptInfo](#indexoptinfo)
  - [Inheritance](#inheritance)
  - [Injection point](#injection-point)
  - [Invalid index](#invalid-index)
  - [Invalidation message](#invalidation-message)
  - [io_combine_limit](#io_combine_limit)
  - [Isolation test](#isolation-test)
  - [JIT compilation](#jit-compilation)
  - [Leakproof function](#leakproof-function)
  - [Line pointer](#line-pointer)
  - [Lock mode](#lock-mode)
  - [Lock queue](#lock-queue)
  - [Logical decoding](#logical-decoding)
  - [Logical replication](#logical-replication)
  - [LSN](#lsn)
  - [LWLock](#lwlock)
  - [maintenance_work_mem](#maintenance_work_mem)
  - [Memoize](#memoize)
  - [Memory context](#memory-context)
  - [Metapage](#metapage)
  - [Most common values and histogram](#most-common-values-and-histogram)
  - [MultiXact](#multixact)
  - [MVCC](#mvcc)
  - [NOT VALID](#not-valid)
  - [OID](#oid)
  - [Operator class](#operator-class)
  - [Origin filter](#origin-filter)
  - [OS page cache](#os-page-cache)
  - [Page](#page)
  - [Page split](#page-split)
  - [pageinspect](#pageinspect)
  - [Parallel query](#parallel-query)
  - [Parallel vacuum](#parallel-vacuum)
  - [Parse tree](#parse-tree)
  - [Partial index](#partial-index)
  - [Partition bound](#partition-bound)
  - [Partition pruning](#partition-pruning)
  - [Partitioned index](#partitioned-index)
  - [Partitionwise join](#partitionwise-join)
  - [Path](#path)
  - [Pending list](#pending-list)
  - [pg_attribute](#pg_attribute)
  - [pg_cast](#pg_cast)
  - [pg_class](#pg_class)
  - [pg_freespacemap](#pg_freespacemap)
  - [pg_index](#pg_index)
  - [pg_node_tree](#pg_node_tree)
  - [pg_plan_advice](#pg_plan_advice)
  - [pg_proc](#pg_proc)
  - [pg_stat_activity](#pg_stat_activity)
  - [pg_stat_all_tables](#pg_stat_all_tables)
  - [pg_stat_io](#pg_stat_io)
  - [pg_stat_statements](#pg_stat_statements)
  - [pg_upgrade](#pg_upgrade)
  - [pgs_mask](#pgs_mask)
  - [pgstatindex](#pgstatindex)
  - [pgstattuple](#pgstattuple)
  - [Plan cache mode](#plan-cache-mode)
  - [PlannedStmt](#plannedstmt)
  - [Planner](#planner)
  - [PL/pgSQL](#plpgsql)
  - [Portal](#portal)
  - [Posting list](#posting-list)
  - [Posting tree](#posting-tree)
  - [Postmaster](#postmaster)
  - [Prepared statement](#prepared-statement)
  - [Progress reporting](#progress-reporting)
  - [Pruning](#pruning)
  - [Publication](#publication)
  - [Qual](#qual)
  - [Query jumbling](#query-jumbling)
  - [Range table](#range-table)
  - [Read stream](#read-stream)
  - [Regression test](#regression-test)
  - [REINDEX](#reindex)
  - [Relation size functions](#relation-size-functions)
  - [Relcache](#relcache)
  - [Relfilenumber](#relfilenumber)
  - [RelOptInfo](#reloptinfo)
  - [reltuples and relpages](#reltuples-and-relpages)
  - [Reorder buffer](#reorder-buffer)
  - [REPACK](#repack)
  - [Replication origin](#replication-origin)
  - [Replication slot](#replication-slot)
  - [Rewriter](#rewriter)
  - [Ring buffer](#ring-buffer)
  - [Row lock](#row-lock)
  - [Row-level security](#row-level-security)
  - [Security barrier](#security-barrier)
  - [SECURITY DEFINER](#security-definer)
  - [Selectivity](#selectivity)
  - [Sequential scan](#sequential-scan)
  - [shared_buffers](#shared_buffers)
  - [shared_preload_libraries](#shared_preload_libraries)
  - [Shared-memory statistics](#shared-memory-statistics)
  - [ShareUpdateExclusiveLock](#shareupdateexclusivelock)
  - [Simple index deletion](#simple-index-deletion)
  - [SLRU](#slru)
  - [Snapshot](#snapshot)
  - [SP-GiST](#sp-gist)
  - [SPI](#spi)
  - [SQL function inlining](#sql-function-inlining)
  - [statement_timeout and lock_timeout](#statement_timeout-and-lock_timeout)
  - [Statistics](#statistics)
  - [Storage manager](#storage-manager)
  - [Storage parameter](#storage-parameter)
  - [SubPlan](#subplan)
  - [Subscription](#subscription)
  - [Synchronous replication](#synchronous-replication)
  - [Syscache](#syscache)
  - [Table rewrite](#table-rewrite)
  - [Table synchronization](#table-synchronization)
  - [Tablespace](#tablespace)
  - [TAP test](#tap-test)
  - [TID](#tid)
  - [Timeline](#timeline)
  - [TOAST](#toast)
  - [track_activity_query_size](#track_activity_query_size)
  - [Transaction ID](#transaction-id)
  - [Truncation](#truncation)
  - [Tuple](#tuple)
  - [Tuplesort](#tuplesort)
  - [Two-phase commit](#two-phase-commit)
  - [Utility command](#utility-command)
  - [VACUUM](#vacuum)
  - [Vacuum cost delay](#vacuum-cost-delay)
  - [Vacuum failsafe](#vacuum-failsafe)
  - [VACUUM FULL](#vacuum-full)
  - [varlena](#varlena)
  - [Visibility map](#visibility-map)
  - [Wait event](#wait-event)
  - [WAL](#wal)
  - [WAL level](#wal-level)
  - [WAL receiver](#wal-receiver)
  - [WAL sender](#wal-sender)
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
- The main paragraph of an entry cites PostgreSQL 17 unless it opens by naming another version, which happens only when the concept does not exist in 17.
- The **Version notes:** list gives each other checked version its own evidence. Each note says whether the main paragraph holds for that version, names the concrete differences, or states that the concept is not present. Every note cites only its own version's checkout.
- As of 2026-09-23, every entry was checked against all five pinned checkouts below: PostgreSQL 12, 14, 17, 18 and 19.
- A glossary link supplies vocabulary, not proof. A page that links a term still needs its own matching-version source citations.
- Deeper, version-local explanations belong on `wiki/vNN/common-concepts/` pages, which entries link when one exists.

## Source Pins

| Version | Checkout | Branch | Pinned commit |
|---|---|---|---|
| 12 | `raw/postgres-12/` | `REL_12_STABLE` | `45b88269a353ad93744772791feb6d01bc7e1e42` |
| 14 | `raw/postgres-14/` | `REL_14_STABLE` | `a92fbdfb830046e907813e9067b2c9de9708d600` |
| 17 | `raw/postgres-17/` | `REL_17_STABLE` | `786db8dcf168bd9df8f55047337525ac19118b1c` |
| 18 | `raw/postgres-18/` | `REL_18_STABLE` | `baa7b142aace6821ce085906f314a75bcc4d95c8` |
| 19 | `raw/postgres-19/` | `REL_19_STABLE` | `135b867a530cac2e3796d87c852b53bef40f0077` |

## Terms

### Access method

**Aliases:** AM, index access method, table access method, `pg_am`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An access method is a pluggable implementation of how a relation stores and finds data. Each row of the `pg_am` catalog names one, gives its handler function, and marks it as an index (`i`) or table (`t`) method ([pg_am.h#FormData_pg_am](../raw/postgres-17/src/include/catalog/pg_am.h#L29-L41), [pg_am.h#AMTYPE_INDEX](../raw/postgres-17/src/include/catalog/pg_am.h#L59-L63)). PostgreSQL 17 ships `heap` as its only table AM, plus six index AMs: `btree`, `hash`, `gist`, `gin`, `spgist` and `brin` ([pg_am.dat](../raw/postgres-17/src/include/catalog/pg_am.dat#L14-L35)). When reading source, `GetIndexAmRoutine()` calls the handler, which returns an `IndexAmRoutine` of capability flags and callbacks such as `amgettuple`, `amgetbitmap` and `amcanreturn`. Core code checks those fields instead of testing the index type ([amapi.c#GetIndexAmRoutine](../raw/postgres-17/src/backend/access/index/amapi.c#L24-L46), [amapi.h#IndexAmRoutine](../raw/postgres-17/src/include/access/amapi.h#L270-L296)).

**Version notes:**
- PostgreSQL 12: Holds. `pg_am` has the same `amname`/`amhandler`/`amtype` columns and `i`/`t` types, and 12 ships `heap` as the only table AM plus the same six index AMs ([pg_am.h#FormData_pg_am](../raw/postgres-12/src/include/catalog/pg_am.h#L29-L41), [pg_am.h#AMTYPE_INDEX](../raw/postgres-12/src/include/catalog/pg_am.h#L53-L56), [pg_am.dat](../raw/postgres-12/src/include/catalog/pg_am.dat#L14-L35)). `GetIndexAmRoutine()` returns the same `IndexAmRoutine` with `amgettuple`, `amgetbitmap` and `amcanreturn` ([amapi.c#GetIndexAmRoutine](../raw/postgres-12/src/backend/access/index/amapi.c#L32-L47), [amapi.h#IndexAmRoutine](../raw/postgres-12/src/include/access/amapi.h#L163-L233)).
- PostgreSQL 14: Holds. `pg_am` has the same `amtype` column with `i` and `t` values, and 14 ships the same `heap` table AM and six index AMs ([pg_am.h#FormData_pg_am](../raw/postgres-14/src/include/catalog/pg_am.h#L29-L41), [pg_am.h#AMTYPE_INDEX](../raw/postgres-14/src/include/catalog/pg_am.h#L58-L61), [pg_am.dat](../raw/postgres-14/src/include/catalog/pg_am.dat#L15-L34)). `GetIndexAmRoutine()` and `IndexAmRoutine` work the same way ([amapi.c#GetIndexAmRoutine](../raw/postgres-14/src/backend/access/index/amapi.c#L33-L46), [amapi.h#IndexAmRoutine](../raw/postgres-14/src/include/access/amapi.h#L210-L286)).
- PostgreSQL 18: Holds. `pg_am` still has the same index/table rows (`heap` plus `btree`, `hash`, `gist`, `gin`, `spgist`, `brin`), and `GetIndexAmRoutine()` still calls the handler ([pg_am.dat](../raw/postgres-18/src/include/catalog/pg_am.dat#L14-L35), [amapi.c#GetIndexAmRoutine](../raw/postgres-18/src/backend/access/index/amapi.c#L24-L46)). `IndexAmRoutine` gains new capability flags (`amcanhash`, `amconsistentequality`, `amconsistentordering`) and callbacks (`amgettreeheight`, `amtranslatestrategy`, `amtranslatecmptype`) ([amapi.h#IndexAmRoutine](../raw/postgres-18/src/include/access/amapi.h#L230-L323)).
- PostgreSQL 19: Holds. `pg_am` still names each AM, its handler and its type `i` or `t`, and v19 still ships `heap` plus the same six index AMs ([pg_am.h#FormData_pg_am](../raw/postgres-19/src/include/catalog/pg_am.h#L31-L43), [pg_am.h#AMTYPE_INDEX](../raw/postgres-19/src/include/catalog/pg_am.h#L63-L66), [pg_am.dat](../raw/postgres-19/src/include/catalog/pg_am.dat#L14-L35)). One difference: `GetIndexAmRoutine()` now returns a `const IndexAmRoutine *`, and handlers such as `bthandler()` return a pointer to a statically allocated `const` struct ([amapi.c#GetIndexAmRoutine](../raw/postgres-19/src/backend/access/index/amapi.c#L24-L59), [nbtree.c#bthandler](../raw/postgres-19/src/backend/access/nbtree/nbtree.c#L118-L121), [amapi.h#IndexAmRoutine](../raw/postgres-19/src/include/access/amapi.h#L302-L313)).

Related: [B-tree](#b-tree), [GIN](#gin), [GiST](#gist), [SP-GiST](#sp-gist), [BRIN](#brin), [Hash index](#hash-index), [Heap](#heap)

### AccessExclusiveLock

**Aliases:** `ACCESS EXCLUSIVE` lock mode, lock mode 8. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The strongest table-level lock mode. It conflicts with every other mode, so its holder is the only transaction touching the table in any way, including plain `SELECT` ([lockdefs.h#AccessExclusiveLock](../raw/postgres-17/src/include/storage/lockdefs.h#L45-L47), [lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)). The docs list `DROP TABLE`, `TRUNCATE`, `REINDEX`, `CLUSTER`, `VACUUM FULL` and many `ALTER TABLE` forms as taking it ([mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1067-L1090)). In source, look for it in `vacuum_rel()`, which picks it only for `VACUUM FULL` ([vacuum.c:2054-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2054-L2055)). Plain VACUUM's end-of-table truncation also requests it, but only with `ConditionalLockRelation()`. It retries for a short time and then gives up rather than queue for the lock ([vacuumlazy.c#lazy_truncate_heap](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2572-L2600)).

**Version notes:**
- PostgreSQL 12: Holds. It is lock mode 8, and `vacuum_rel()` picks it only for `VACUUM FULL` ([lockdefs.h#AccessExclusiveLock](../raw/postgres-12/src/include/storage/lockdefs.h#L45-L46), [vacuum.c:1680-1681](../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1681)). Truncation in `lazy_truncate_heap()` also retries `ConditionalLockRelation()` every 50 ms for up to 5 s, then gives up ([vacuumlazy.c#lazy_truncate_heap](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1897-L1926), [vacuumlazy.c:82-83](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L82-L83)). The v12 lock-mode docs are at a different place in `mvcc.sgml` ([mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1002-L1028)).
- PostgreSQL 14: Holds. It is lock mode 8, and `vacuum_rel()` picks it only for `VACUUM FULL` ([lockdefs.h#AccessExclusiveLock](../raw/postgres-14/src/include/storage/lockdefs.h#L45-L47), [vacuum.c:1919-1920](../raw/postgres-14/src/backend/commands/vacuum.c#L1919-L1920)). Truncation retries `ConditionalLockRelation()` every 50 ms for up to 5 s, then gives up ([vacuumlazy.c:102-104](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L102-L104), [vacuumlazy.c#lazy_truncate_heap](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L3203-L3240)).
- PostgreSQL 18: Holds. The mode, its conflict row, the doc list, `vacuum_rel()`'s choice of it only for `VACUUM FULL`, and the conditional truncation lock are unchanged ([lockdefs.h#AccessExclusiveLock](../raw/postgres-18/src/include/storage/lockdefs.h#L45-L47), [lock.c#LockConflicts](../raw/postgres-18/src/backend/storage/lmgr/lock.c#L99-L103), [vacuum.c:2092-2093](../raw/postgres-18/src/backend/commands/vacuum.c#L2092-L2093), [vacuumlazy.c#lazy_truncate_heap](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L3246-L3274)).
- PostgreSQL 19: Holds. Mode 8 still conflicts with every mode, and `vacuum_rel()` still picks it only for `VACUUM FULL` ([lockdefs.h#AccessExclusiveLock](../raw/postgres-19/src/include/storage/lockdefs.h#L45-L46), [lock.c#LockConflicts](../raw/postgres-19/src/backend/storage/lmgr/lock.c#L102-L106), [vacuum.c:2084-2085](../raw/postgres-19/src/backend/commands/vacuum.c#L2084-L2085)). The docs now add `REPACK` to its takers, and `REPACK (CONCURRENTLY)` holds it only while it swaps the files ([mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-19/doc/src/sgml/mvcc.sgml#L1081-L1109)). Truncation still asks for it only with `ConditionalLockRelation()` ([vacuumlazy.c#lazy_truncate_heap](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L3166-L3225)).

Related: [Heavyweight lock](#heavyweight-lock), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [Truncation](#truncation), [VACUUM FULL](#vacuum-full)

### Alignment

**Aliases:** `MAXALIGN`, `TYPEALIGN`, `typalign`, `MAXIMUM_ALIGNOF`, alignment padding. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Alignment means starting a value at a byte offset that is a multiple of some power of two, and inserting unused padding bytes to get there. `TYPEALIGN()` rounds a length up to a multiple of its alignment, and `MAXALIGN()` rounds up to `MAXIMUM_ALIGNOF`, the platform's largest built-in alignment ([c.h#TYPEALIGN](../raw/postgres-17/src/include/c.h#L808-L828)). Each data type declares its own alignment in `pg_type.typalign` (`c`, `s`, `i` or `d`), and PostgreSQL pads before a value so it starts on that boundary, both on disk and in memory ([pg_type.h#typalign](../raw/postgres-17/src/include/catalog/pg_type.h#L150-L176)). It matters when reading size arithmetic: `PageAddItemExtended()` reserves `MAXALIGN(size)` bytes for every item it puts on a page, and a heap tuple pads its header so user data starts `MAXALIGN`'d ([bufpage.c:315](../raw/postgres-17/src/backend/storage/page/bufpage.c#L315), [htup_details.h:65-68](../raw/postgres-17/src/include/access/htup_details.h#L65-L68)). The control file records `maxAlign`, and the server refuses to start on a cluster built with a different `MAXALIGN` ([pg_control.h:199](../raw/postgres-17/src/include/catalog/pg_control.h#L199), [xlog.c#ReadControlFile](../raw/postgres-17/src/backend/access/transam/xlog.c#L4385-L4391)). `pg_controldata` prints it as "Maximum data alignment" ([pg_controldata.c:313](../raw/postgres-17/src/bin/pg_controldata/pg_controldata.c#L313)).

**Version notes:**
- PostgreSQL 12: The same macros and `typalign` column exist ([c.h:678-685](../raw/postgres-12/src/include/c.h#L678-L685), [pg_type.h#typalign](../raw/postgres-12/src/include/catalog/pg_type.h#L146-L170)), and `PageAddItemExtended()` also reserves `MAXALIGN(size)` ([bufpage.c:301](../raw/postgres-12/src/backend/storage/page/bufpage.c#L301)).
- PostgreSQL 14: Holds ([c.h:795-802](../raw/postgres-14/src/include/c.h#L795-L802), [bufpage.c:315](../raw/postgres-14/src/backend/storage/page/bufpage.c#L315)). `pg_type.h` names the letters with `TYPALIGN_CHAR` through `TYPALIGN_DOUBLE` ([pg_type.h:300-303](../raw/postgres-14/src/include/catalog/pg_type.h#L300-L303)).
- PostgreSQL 18: Holds ([c.h:790-797](../raw/postgres-18/src/include/c.h#L790-L797), [bufpage.c:314](../raw/postgres-18/src/backend/storage/page/bufpage.c#L314), [xlog.c#ReadControlFile](../raw/postgres-18/src/backend/access/transam/xlog.c#L4434)).
- PostgreSQL 19: Holds ([c.h:896-903](../raw/postgres-19/src/include/c.h#L896-L903), [bufpage.c:326](../raw/postgres-19/src/backend/storage/page/bufpage.c#L326), [xlog.c#ReadControlFile](../raw/postgres-19/src/backend/access/transam/xlog.c#L4505)).

Related: [Page](#page), [Tuple](#tuple), [Line pointer](#line-pointer), [BLCKSZ](#blcksz)

### allequalimage

**Aliases:** `btm_allequalimage`, "equality is image equality". **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`allequalimage` is a flag in a B-tree metapage. It records whether every key column's operator class promises that equal values are also bitwise-identical, which is the condition that makes deduplication safe. `_bt_allequalimage()` computes it from each opclass's `BTEQUALIMAGE_PROC` support function, and index builds store the result ([nbtree.h#BTMetaPageData](../raw/postgres-17/src/include/access/nbtree.h#L103-L119), [nbtutils.c#_bt_allequalimage](../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5140)). Only a build can set it to true. `_bt_upgrademetapage()` forces it to false. As `nbtree.h` notes, `pg_upgrade` never sets it, so indexes carried over from PostgreSQL 12 need `REINDEX` before they can deduplicate ([nbtpage.c#_bt_upgrademetapage](../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L106-L126), [nbtree.h:135-141](../raw/postgres-17/src/include/access/nbtree.h#L135-L141)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The v12 B-tree metapage has no `btm_allequalimage` field, and the current on-disk version is 4, before deduplication existed ([nbtree.h#BTMetaPageData](../raw/postgres-12/src/include/access/nbtree.h#L97-L110), [nbtree.h#BTREE_VERSION](../raw/postgres-12/src/include/access/nbtree.h#L133-L135)). A v12 index carried forward by `pg_upgrade` therefore never has the flag set.
- PostgreSQL 14: Holds. The metapage has `btm_allequalimage`, `_bt_allequalimage()` computes it, and `_bt_upgrademetapage()` forces it to false, so only a build or `REINDEX` can set it ([nbtree.h#BTMetaPageData](../raw/postgres-14/src/include/access/nbtree.h#L101-L117), [nbtutils.c#_bt_allequalimage](../raw/postgres-14/src/backend/access/nbtree/nbtutils.c#L2712-L2771), [nbtpage.c#_bt_upgrademetapage](../raw/postgres-14/src/backend/access/nbtree/nbtpage.c#L109-L133), [nbtree.h:133-141](../raw/postgres-14/src/include/access/nbtree.h#L133-L141)).
- PostgreSQL 18: Holds. The metapage field, `_bt_allequalimage()`, and the `_bt_upgrademetapage()` rule that forces it false are unchanged ([nbtree.h#BTMetaPageData](../raw/postgres-18/src/include/access/nbtree.h#L104-L120), [nbtutils.c#_bt_allequalimage](../raw/postgres-18/src/backend/access/nbtree/nbtutils.c#L4248-L4259), [nbtpage.c#_bt_upgrademetapage](../raw/postgres-18/src/backend/access/nbtree/nbtpage.c#L106-L126), [nbtree.h:136-142](../raw/postgres-18/src/include/access/nbtree.h#L136-L142)).
- PostgreSQL 19: Holds. `btm_allequalimage` is still a metapage field computed by `_bt_allequalimage()` at build time, `_bt_upgrademetapage()` still forces it false, and the header still says `pg_upgrade` cannot set it ([nbtree.h#BTMetaPageData](../raw/postgres-19/src/include/access/nbtree.h#L104-L120), [nbtutils.c#_bt_allequalimage](../raw/postgres-19/src/backend/access/nbtree/nbtutils.c#L1165-L1175), [nbtpage.c#_bt_upgrademetapage](../raw/postgres-19/src/backend/access/nbtree/nbtpage.c#L100-L127), [nbtree.h:136-142](../raw/postgres-19/src/include/access/nbtree.h#L136-L142)).

Related: [Deduplication](#deduplication), [Metapage](#metapage), [pg_upgrade](#pg_upgrade), [REINDEX](#reindex)

### amcheck

**Aliases:** `bt_index_check`, `bt_index_parent_check`, `verify_heapam`, `gin_index_check`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`amcheck` is a [contrib](#contrib) extension that checks whether a table or index is internally consistent. It raises no error when the structure looks valid ([amcheck.sgml:10-30](../raw/postgres-17/doc/src/sgml/amcheck.sgml#L10-L30), [amcheck.control](../raw/postgres-17/contrib/amcheck/amcheck.control#L1-L3)). For [B-tree](#b-tree) indexes it verifies invariants such as items being in logical order on each page, and it can optionally confirm that every visible heap tuple has a matching index entry ([verify_nbtree.c header](../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L1-L14)). `bt_index_check()` takes only `AccessShareLock` and skips parent/child checks. `bt_index_parent_check()` takes `ShareLock` and also checks that each parent downlink bounds its child page ([verify_nbtree.c#bt_index_check](../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L231-L268)). `verify_heapam()` scans heap pages and returns one row per corruption it finds ([verify_heapam.c#verify_heapam](../raw/postgres-17/contrib/amcheck/verify_heapam.c#L184-L217)).

**Version notes:**
- PostgreSQL 12: Only the two B-tree functions exist, and they take at most a `heapallindexed` argument ([verify_nbtree.c#bt_index_check](../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L204)). `verify_heapam.c` does not exist; `contrib/amcheck/` holds only `verify_nbtree.c` ([amcheck.control](../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L3)).
- PostgreSQL 14: Adds `verify_heapam()` ([verify_heapam.c:212](../raw/postgres-14/contrib/amcheck/verify_heapam.c#L212)) next to the B-tree checks ([verify_nbtree.c#bt_index_check](../raw/postgres-14/contrib/amcheck/verify_nbtree.c#L206-L229)).
- PostgreSQL 18: Adds `gin_index_check()` for [GIN](#gin) indexes; it takes `AccessShareLock` ([verify_gin.c#gin_index_check](../raw/postgres-18/contrib/amcheck/verify_gin.c#L1-L12), [verify_gin.c:70-79](../raw/postgres-18/contrib/amcheck/verify_gin.c#L70-L79)). The extension version is 1.5 ([amcheck.control](../raw/postgres-18/contrib/amcheck/amcheck.control#L1-L3)).
- PostgreSQL 19: Same function set as 18 ([verify_gin.c:79](../raw/postgres-19/contrib/amcheck/verify_gin.c#L79), [verify_nbtree.c:252](../raw/postgres-19/contrib/amcheck/verify_nbtree.c#L252), [verify_heapam.c:252](../raw/postgres-19/contrib/amcheck/verify_heapam.c#L252)).

Related: [Contrib](#contrib), [Extension](#extension), [B-tree](#b-tree), [GIN](#gin), [Heap](#heap)

### Apply worker

**Aliases:** logical replication apply worker. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An apply worker is the subscriber-side process that receives a subscription's change stream and replays it into local tables. The logical replication launcher starts one per enabled subscription, as a dynamic background worker whose entry point is `ApplyWorkerMain` ([worker.c header](../raw/postgres-17/src/backend/replication/logical/worker.c#L10-L16), [launcher.c#logicalrep_worker_launch](../raw/postgres-17/src/backend/replication/logical/launcher.c#L476-L485)). In source, `run_apply_worker` sets up the subscription's replication origin before it starts streaming, and `apply_dispatch` routes each protocol message ([worker.c#run_apply_worker](../raw/postgres-17/src/backend/replication/logical/worker.c#L4516-L4525), [worker.c#apply_dispatch](../raw/postgres-17/src/backend/replication/logical/worker.c#L3296-L3300)). Do not confuse it with the tablesync worker or the parallel apply worker. The launcher starts those two separately ([launcher.c#logicalrep_worker_launch](../raw/postgres-17/src/backend/replication/logical/launcher.c#L487-L503)).

**Version notes:**
- PostgreSQL 12: Holds, with two differences. The launcher starts one apply worker per enabled subscription, but table-sync workers run the same entry point, `ApplyWorkerMain`, and differ only by a valid `relid` ([worker.c header](../raw/postgres-12/src/backend/replication/logical/worker.c#L10-L16), [launcher.c#logicalrep_worker_launch](../raw/postgres-12/src/backend/replication/logical/launcher.c#L424-L431)). There is no `run_apply_worker` and no parallel apply worker; `ApplyWorkerMain` itself sets up the origin, and `apply_dispatch` routes messages ([worker.c#ApplyWorkerMain](../raw/postgres-12/src/backend/replication/logical/worker.c#L1721-L1731), [worker.c#apply_dispatch](../raw/postgres-12/src/backend/replication/logical/worker.c#L979-L983)).
- PostgreSQL 14: Holds, with two differences. The launcher starts every logical replication worker, tablesync included, with entry point `ApplyWorkerMain`; a table sync worker differs only by a valid `relid` ([launcher.c#logicalrep_worker_launch](../raw/postgres-14/src/backend/replication/logical/launcher.c#L393-L405), [worker.c#ApplyWorkerMain](../raw/postgres-14/src/backend/replication/logical/worker.c#L3094-L3100)). 14 has no parallel apply worker and no `run_apply_worker`; `ApplyWorkerMain` sets up the replication origin itself, and `apply_dispatch` routes messages ([worker.c:3231-3234](../raw/postgres-14/src/backend/replication/logical/worker.c#L3231-L3234), [worker.c#apply_dispatch](../raw/postgres-14/src/backend/replication/logical/worker.c#L2044-L2116)).
- PostgreSQL 18: Holds. The launcher still starts apply, tablesync and parallel apply workers as dynamic background workers, and `run_apply_worker` still sets up the origin ([worker.c header](../raw/postgres-18/src/backend/replication/logical/worker.c#L10-L16), [launcher.c#logicalrep_worker_launch](../raw/postgres-18/src/backend/replication/logical/launcher.c#L473-L500), [worker.c#run_apply_worker](../raw/postgres-18/src/backend/replication/logical/worker.c#L4584-L4593)).
- PostgreSQL 19: Holds. The launcher starts one apply worker per enabled subscription with entry point `ApplyWorkerMain`, `run_apply_worker()` sets up the origin before streaming, and `apply_dispatch()` routes messages ([worker.c header](../raw/postgres-19/src/backend/replication/logical/worker.c#L10-L16), [launcher.c#logicalrep_worker_launch](../raw/postgres-19/src/backend/replication/logical/launcher.c#L509-L517), [worker.c#run_apply_worker](../raw/postgres-19/src/backend/replication/logical/worker.c#L5701-L5730), [worker.c#apply_dispatch](../raw/postgres-19/src/backend/replication/logical/worker.c#L3802-L3805)). v19 adds a fourth worker type besides apply, parallel apply and tablesync: the sequencesync worker, entry point `SequenceSyncWorkerMain` ([worker_internal.h#LogicalRepWorkerType](../raw/postgres-19/src/include/replication/worker_internal.h#L27-L35), [launcher.c#logicalrep_worker_launch](../raw/postgres-19/src/backend/replication/logical/launcher.c#L529-L542)).

Related: [Background worker](#background-worker), [Logical replication](#logical-replication), [Replication origin](#replication-origin), [Subscription](#subscription)

### Asynchronous I/O

**Aliases:** AIO, `io_method`, `io_workers`, `io_min_workers`, `io_max_workers`, `io_uring`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

In PostgreSQL 18, asynchronous I/O (AIO) lets a backend start a read, keep working, and collect the result later, instead of blocking in each system call. The subsystem lives in `src/backend/storage/aio/`, and its README explains the goal: better prefetching and controlled writeback than the operating system gives synchronous I/O ([aio/README.md](../raw/postgres-18/src/backend/storage/aio/README.md#L1-L15), [aio/Makefile#OBJS](../raw/postgres-18/src/backend/storage/aio/Makefile#L11-L21)). The `io_method` setting picks how the I/O runs: `sync`, `worker` (dedicated I/O worker processes, the default) or `io_uring` when built with it ([aio.h#IoMethod](../raw/postgres-18/src/include/storage/aio.h#L31-L42)). `io_method` has context `postmaster`, so changing it needs a restart ([guc_tables.c#io_method](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5444-L5451)). `io_workers` sets the worker count, default 3, with context `sighup`, so a reload applies it ([guc_tables.c#io_workers](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3318-L3328)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The storage makefile has no `aio` subdirectory ([storage/Makefile#SUBDIRS](../raw/postgres-12/src/backend/storage/Makefile#L11)).
- PostgreSQL 14: Not present in PostgreSQL 14. The storage makefile has no `aio` subdirectory ([storage/Makefile#SUBDIRS](../raw/postgres-14/src/backend/storage/Makefile#L11)).
- PostgreSQL 17: Not present as a subsystem. `storage/aio/` builds only `read_stream.c`, a look-ahead layer that still reads through `StartReadBuffers()` and `WaitReadBuffers()` ([aio/Makefile#OBJS](../raw/postgres-17/src/backend/storage/aio/Makefile#L11-L12), [read_stream.c header](../raw/postgres-17/src/backend/storage/aio/read_stream.c#L1-L18)).
- PostgreSQL 19: Present. `io_method` is still `postmaster` context (restart). `io_workers` is replaced by `io_min_workers` (default 2) and `io_max_workers` (default 8), both `sighup` (reload) ([guc_parameters.dat#io_max_workers](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1393-L1415), [aio.h#DEFAULT_IO_METHOD](../raw/postgres-19/src/include/storage/aio.h#L34-L42)).

Related: [Buffer manager](#buffer-manager), [Read stream](#read-stream), [shared_buffers](#shared_buffers)

### Autovacuum

**Aliases:** autovacuum launcher, autovacuum worker. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Autovacuum is the set of background processes that run [VACUUM](#vacuum) and ANALYZE on their own schedule. One launcher process plans the work, and the [postmaster](#postmaster) forks the worker processes that do it ([autovacuum.c:7-27](../raw/postgres-17/src/backend/postmaster/autovacuum.c#L7-L27), [glossary.sgml#glossary-autovacuum](../raw/postgres-17/doc/src/sgml/glossary.sgml#L125-L142)). A worker vacuums a table when its dead-tuple count passes a base threshold plus a scale factor times `reltuples`. It also vacuums when inserts pass a similar insert threshold ([autovacuum.c#relation_needs_vacanalyze](../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3070-L3095)). Autovacuum runs only while both the `autovacuum` and `track_counts` settings are on ([autovacuum.c#AutoVacuumingActive](../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3236-L3241)). The `autovacuum` setting has context `sighup`, so changing it needs a reload ([guc_tables.c:1450-1456](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1456)).

**Version notes:**
- PostgreSQL 12: Holds, except that 12 has no insert-based trigger. The launcher plans and the postmaster forks the workers ([autovacuum.c:5-27](../raw/postgres-12/src/backend/postmaster/autovacuum.c#L5-L27)). A table is vacuumed only when dead tuples pass base threshold plus scale factor times `reltuples` ([autovacuum.c#relation_needs_vacanalyze](../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060-L3080)). Both `autovacuum` and `track_counts` must be on ([autovacuum.c#AutoVacuumingActive](../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3206-L3212)). `autovacuum` is `PGC_SIGHUP`, so a change needs a reload; it lives in `guc.c` ([guc.c#autovacuum](../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433)).
- PostgreSQL 14: Holds. The launcher/worker split and the postmaster forking workers are the same ([autovacuum.c:7-27](../raw/postgres-14/src/backend/postmaster/autovacuum.c#L7-L27)). Both the dead-tuple threshold and the insert threshold exist ([autovacuum.c#relation_needs_vacanalyze](../raw/postgres-14/src/backend/postmaster/autovacuum.c#L3232-L3236)). It runs only while `autovacuum` and `track_counts` are on ([autovacuum.c#AutoVacuumingActive](../raw/postgres-14/src/backend/postmaster/autovacuum.c#L3382-L3387)). `autovacuum` is `PGC_SIGHUP`, so a change needs a reload; the GUC table is in `guc.c` in 14, not `guc_tables.c` ([guc.c:1569](../raw/postgres-14/src/backend/utils/misc/guc.c#L1569)).
- PostgreSQL 18: Holds, with a changed trigger formula. The dead-tuple threshold is now capped by `autovacuum_vacuum_max_threshold` (default 100000000, -1 disables; context `sighup`, so a reload), and the insert threshold scales `reltuples` by the table's unfrozen fraction, from the new `relallfrozen` ([autovacuum.c#relation_needs_vacanalyze](../raw/postgres-18/src/backend/postmaster/autovacuum.c#L3092-L3127), [guc_tables.c#autovacuum_vacuum_max_threshold](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3549-L3556)). `autovacuum` is still `sighup` (reload) ([guc_tables.c:1542-1549](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1542-L1549)). `autovacuum_max_workers` is `sighup` (reload) up to the new `autovacuum_worker_slots`, which is `postmaster` (restart) ([guc_tables.c:3599-3615](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3599-L3615)).
- PostgreSQL 19: Holds for the launcher/worker split, the threshold formulas, and the `autovacuum` + `track_counts` gate ([autovacuum.c:5-27](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L5-L27), [glossary.sgml#glossary-autovacuum](../raw/postgres-19/doc/src/sgml/glossary.sgml#L165-L183), [autovacuum.c#relation_needs_vacanalyze](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3285-L3307), [autovacuum.c#AutoVacuumingActive](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3465-L3471)). New in v19: `relation_needs_vacanalyze()` also returns a score, the maximum of each value's ratio to its threshold, and workers process tables in score order. Five `autovacuum_*_score_weight` GUCs (default 1.0; 0.0 restores pre-v19 ordering) scale the components ([autovacuum.c#relation_needs_vacanalyze](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3028-L3072), [autovacuum.c#TableToProcessComparator](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1905-L1916), [autovacuum.c:2321](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2321)). `autovacuum_vacuum_score_weight` has context `PGC_SIGHUP` (reload). `autovacuum` stays `PGC_SIGHUP` (reload) ([guc_parameters.dat#autovacuum_vacuum_score_weight](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L275-L279), [guc_parameters.dat#autovacuum](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L124-L128)).

Related: [VACUUM](#vacuum), [reltuples and relpages](#reltuples-and-relpages), [Cumulative statistics](#cumulative-statistics), [GUC context](#guc-context)

### Backend

**Aliases:** backend process. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A backend is the server process that serves one client session and handles its requests ([glossary.sgml#glossary-backend](../raw/postgres-17/doc/src/sgml/glossary.sgml#L173-L187)). The postmaster forks one per accepted connection in `BackendStartup` ([postmaster.c#BackendStartup](../raw/postgres-17/src/backend/postmaster/postmaster.c#L3525-L3536)). The child enters `BackendMain`, which authenticates the client and then starts the main loop ([backend_startup.c#BackendMain](../raw/postgres-17/src/backend/tcop/backend_startup.c#L50-L58)). That loop is `PostgresMain`, where every backend runs its queries ([postgres.c#PostgresMain](../raw/postgres-17/src/backend/tcop/postgres.c#L4247-L4259)). The glossary itself warns against confusing a backend with a background worker or the background writer ([glossary.sgml#glossary-backend](../raw/postgres-17/doc/src/sgml/glossary.sgml#L180-L184)).

**Version notes:**
- PostgreSQL 12: Holds, but the startup path differs. There is no `backend_startup.c` and no `BackendMain`. `BackendStartup()` in `postmaster.c` forks the child, which runs `BackendInitialize()` and then `BackendRun()`, and that calls `PostgresMain()` ([postmaster.c#BackendStartup](../raw/postgres-12/src/backend/postmaster/postmaster.c#L4059-L4075), [postmaster.c#BackendInitialize](../raw/postgres-12/src/backend/postmaster/postmaster.c#L4209-L4211), [postmaster.c#BackendRun](../raw/postgres-12/src/backend/postmaster/postmaster.c#L4385-L4437), [postgres.c#PostgresMain](../raw/postgres-12/src/backend/tcop/postgres.c#L3706-L3709)).
- PostgreSQL 14: Holds, but the startup functions differ. 14 has no `backend_startup.c` or `BackendMain`. The postmaster forks in `BackendStartup`, the child runs `BackendInitialize` (authentication setup) and then `BackendRun`, which calls `PostgresMain` ([postmaster.c#BackendStartup](../raw/postgres-14/src/backend/postmaster/postmaster.c#L4216-L4220), [postmaster.c#BackendInitialize](../raw/postgres-14/src/backend/postmaster/postmaster.c#L4368-L4372), [postmaster.c#BackendRun](../raw/postgres-14/src/backend/postmaster/postmaster.c#L4541-L4548), [postgres.c#PostgresMain](../raw/postgres-14/src/backend/tcop/postgres.c#L4027)). The docs glossary has the same definition and warning ([glossary.sgml#glossary-backend](../raw/postgres-14/doc/src/sgml/glossary.sgml#L126-L140)).
- PostgreSQL 18: Holds. The postmaster forks one per connection in `BackendStartup`, which now first takes a `PMChild` slot and launches the child with `postmaster_child_launch()` ([postmaster.c#BackendStartup](../raw/postgres-18/src/backend/postmaster/postmaster.c#L3509-L3571), [pmchild.c header](../raw/postgres-18/src/backend/postmaster/pmchild.c#L1-L20)). The child still runs `BackendMain` and then `PostgresMain` ([backend_startup.c#BackendMain](../raw/postgres-18/src/backend/tcop/backend_startup.c#L69-L77), [postgres.c#PostgresMain](../raw/postgres-18/src/backend/tcop/postgres.c#L4176-L4188)). The docs' warning is at a new line range ([glossary.sgml#glossary-backend](../raw/postgres-18/doc/src/sgml/glossary.sgml#L207-L209)).
- PostgreSQL 19: Holds. `BackendStartup()` forks a backend per connection, `BackendMain()` authenticates and calls `PostgresMain()` ([glossary.sgml#glossary-backend](../raw/postgres-19/doc/src/sgml/glossary.sgml#L213-L227), [postmaster.c#BackendStartup](../raw/postgres-19/src/backend/postmaster/postmaster.c#L3591-L3610), [backend_startup.c#BackendMain](../raw/postgres-19/src/backend/tcop/backend_startup.c#L70-L77), [backend_startup.c:124](../raw/postgres-19/src/backend/tcop/backend_startup.c#L124), [postgres.c#PostgresMain](../raw/postgres-19/src/backend/tcop/postgres.c#L4273-L4275)).

Related: [Background worker](#background-worker), [Postmaster](#postmaster)

### Background worker

**Aliases:** bgworker. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A background worker is a server process that runs built-in or extension-supplied code instead of serving a client connection ([glossary.sgml#glossary-background-worker](../raw/postgres-17/doc/src/sgml/glossary.sgml#L189-L204)). A `BackgroundWorker` struct describes the worker: its name, its start time, its restart interval, and the library and function to run ([bgworker.h#BackgroundWorker](../raw/postgres-17/src/include/postmaster/bgworker.h#L89-L101)). Static workers register through `RegisterBackgroundWorker`, which only has an effect from the postmaster or from `shared_preload_libraries` initialization. Running backends use `RegisterDynamicBackgroundWorker` instead ([bgworker.c#RegisterBackgroundWorker](../raw/postgres-17/src/backend/postmaster/bgworker.c#L854-L862), [bgworker.c#RegisterDynamicBackgroundWorker](../raw/postgres-17/src/backend/postmaster/bgworker.c#L959-L971)). Do not confuse a background worker with a backend or with the background writer ([glossary.sgml#glossary-backend](../raw/postgres-17/doc/src/sgml/glossary.sgml#L180-L184)).

**Version notes:**
- PostgreSQL 12: Holds. `BackgroundWorker` has the same name, start time, restart interval, library and function fields ([bgworker.h#BackgroundWorker](../raw/postgres-12/src/include/postmaster/bgworker.h#L88-L100)). `RegisterBackgroundWorker` works only from the postmaster or `shared_preload_libraries`; running backends use `RegisterDynamicBackgroundWorker` ([bgworker.c#RegisterBackgroundWorker](../raw/postgres-12/src/backend/postmaster/bgworker.c#L840-L866), [bgworker.c#RegisterDynamicBackgroundWorker](../raw/postgres-12/src/backend/postmaster/bgworker.c#L932)).
- PostgreSQL 14: Holds. `BackgroundWorker` has the same fields, and `RegisterBackgroundWorker` refuses non-core workers outside `shared_preload_libraries` loading, while running backends use `RegisterDynamicBackgroundWorker` ([bgworker.h#BackgroundWorker](../raw/postgres-14/src/include/postmaster/bgworker.h#L88-L100), [bgworker.c#RegisterBackgroundWorker](../raw/postgres-14/src/backend/postmaster/bgworker.c#L889-L905), [bgworker.c#RegisterDynamicBackgroundWorker](../raw/postgres-14/src/backend/postmaster/bgworker.c#L973), [glossary.sgml#glossary-background-worker](../raw/postgres-14/doc/src/sgml/glossary.sgml#L142-L160)).
- PostgreSQL 18: Holds. The `BackgroundWorker` struct and the static and dynamic registration rules are unchanged ([bgworker.h#BackgroundWorker](../raw/postgres-18/src/include/postmaster/bgworker.h#L89-L101), [bgworker.c#RegisterBackgroundWorker](../raw/postgres-18/src/backend/postmaster/bgworker.c#L932-L940), [bgworker.c#RegisterDynamicBackgroundWorker](../raw/postgres-18/src/backend/postmaster/bgworker.c#L1035-L1047), [glossary.sgml#glossary-backend](../raw/postgres-18/doc/src/sgml/glossary.sgml#L207-L209)).
- PostgreSQL 19: Holds. `BackgroundWorker` has the same fields, `RegisterBackgroundWorker()` still works only from the postmaster or `shared_preload_libraries`, and running backends use `RegisterDynamicBackgroundWorker()` ([glossary.sgml#glossary-background-worker](../raw/postgres-19/doc/src/sgml/glossary.sgml#L229-L248), [bgworker.h#BackgroundWorker](../raw/postgres-19/src/include/postmaster/bgworker.h#L96-L108), [bgworker.c#RegisterBackgroundWorker](../raw/postgres-19/src/backend/postmaster/bgworker.c#L955-L962), [bgworker.c#RegisterDynamicBackgroundWorker](../raw/postgres-19/src/backend/postmaster/bgworker.c#L1060-L1069)).

Related: [Apply worker](#apply-worker), [Background writer](#background-writer), [Backend](#backend), [Postmaster](#postmaster)

### Background writer

**Aliases:** bgwriter. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An auxiliary server process that writes dirty pages from shared buffers to the file system a little at a time, so that ordinary backends rarely have to write a buffer themselves before reusing it ([bgwriter.c](../raw/postgres-17/src/backend/postmaster/bgwriter.c#L5-L13), [glossary.sgml#background-writer](../raw/postgres-17/doc/src/sgml/glossary.sgml#L209-L221)). Since 9.2 it no longer performs checkpoints; the checkpointer process does ([bgwriter.c:13](../raw/postgres-17/src/backend/postmaster/bgwriter.c#L13)). Its main loop calls `BgBufferSync()` in the buffer manager ([bgwriter.c:234](../raw/postgres-17/src/backend/postmaster/bgwriter.c#L234)). Its pace is set by `bgwriter_delay`, a `sighup` GUC, so changing it needs a reload ([guc_tables.c#bgwriter_delay](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3077-L3084)).

**Version notes:**
- PostgreSQL 12: Holds. The bgwriter writes dirty shared buffers so backends rarely must, and it has not done checkpoints since 9.2 ([bgwriter.c](../raw/postgres-12/src/backend/postmaster/bgwriter.c#L5-L13)). Its loop calls `BgBufferSync()` ([bgwriter.c:267](../raw/postgres-12/src/backend/postmaster/bgwriter.c#L267)). `bgwriter_delay` is `PGC_SIGHUP` (reload) and is defined in `guc.c` ([guc.c#bgwriter_delay](../raw/postgres-12/src/backend/utils/misc/guc.c#L2728-L2736)).
- PostgreSQL 14: Holds. The header describes the same job and says it stopped doing checkpoints in 9.2; its loop calls `BgBufferSync()` ([bgwriter.c:5-15](../raw/postgres-14/src/backend/postmaster/bgwriter.c#L5-L15), [bgwriter.c:244](../raw/postgres-14/src/backend/postmaster/bgwriter.c#L244)). `bgwriter_delay` is `PGC_SIGHUP`, so a change needs a reload; it is defined in `guc.c` ([guc.c:3013](../raw/postgres-14/src/backend/utils/misc/guc.c#L3013)).
- PostgreSQL 18: Holds. `bgwriter.c`, the `BgBufferSync()` call, and `bgwriter_delay` as a `sighup` GUC (reload) are unchanged ([bgwriter.c](../raw/postgres-18/src/backend/postmaster/bgwriter.c#L5-L13), [bgwriter.c:236](../raw/postgres-18/src/backend/postmaster/bgwriter.c#L236), [guc_tables.c#bgwriter_delay](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3217-L3224)).
- PostgreSQL 19: Holds. The header, the 9.2 checkpoint note and the `BgBufferSync()` call are unchanged, and `bgwriter_delay` is still `PGC_SIGHUP` (reload) ([bgwriter.c](../raw/postgres-19/src/backend/postmaster/bgwriter.c#L5-L13), [bgwriter.c:236](../raw/postgres-19/src/backend/postmaster/bgwriter.c#L236), [glossary.sgml#glossary-background-writer](../raw/postgres-19/doc/src/sgml/glossary.sgml#L250-L268), [guc_parameters.dat#bgwriter_delay](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L337-L344)).

Related: [Buffer manager](#buffer-manager), [Checkpoint](#checkpoint), [Dirty buffer](#dirty-buffer), [shared_buffers](#shared_buffers)

### Bitmap scan

**Aliases:** Bitmap Index Scan, Bitmap Heap Scan, TIDBitmap. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A bitmap scan runs in two steps. First, an index scan collects matching row addresses into an in-memory bitmap. Then a heap scan visits the table pages the bitmap names, and the bitmap's iterator hands them out from sorted page lists ([nodeBitmapHeapscan.c](../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L1-L17), [tidbitmap.c#tbm_begin_iterate](../raw/postgres-17/src/backend/nodes/tidbitmap.c#L710-L747)). `tidbitmap.c` stores the set, and it can turn "lossy": it keeps only whole pages, not tuple offsets, so the heap step has to recheck the conditions ([tidbitmap.c](../raw/postgres-17/src/backend/nodes/tidbitmap.c#L3-L30)). `nodeBitmapHeapscan.c` requires an MVCC snapshot because the index and heap passes are decoupled ([nodeBitmapHeapscan.c](../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L1-L17)). GIN and BRIN leave `amgettuple` NULL, so bitmap scans are the only way the executor can read them ([ginutil.c:79](../raw/postgres-17/src/backend/access/gin/ginutil.c#L79), [brin.c:289](../raw/postgres-17/src/backend/access/brin/brin.c#L289)).

**Version notes:**
- PostgreSQL 12: Holds. The bitmap heap scan needs an MVCC snapshot, `tidbitmap.c` supports lossy page-only storage that forces a recheck, and GIN and BRIN leave `amgettuple` NULL ([nodeBitmapHeapscan.c](../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L1-L17), [tidbitmap.c](../raw/postgres-12/src/backend/nodes/tidbitmap.c#L3-L30), [tidbitmap.c#tbm_begin_iterate](../raw/postgres-12/src/backend/nodes/tidbitmap.c#L688), [ginutil.c:71](../raw/postgres-12/src/backend/access/gin/ginutil.c#L71), [brin.c:119](../raw/postgres-12/src/backend/access/brin/brin.c#L119)).
- PostgreSQL 14: Holds. `nodeBitmapHeapscan.c` requires an MVCC snapshot, `tidbitmap.c` supports lossy pages that force a recheck, and `tbm_begin_iterate()` hands out sorted pages ([nodeBitmapHeapscan.c](../raw/postgres-14/src/backend/executor/nodeBitmapHeapscan.c#L1-L17), [tidbitmap.c](../raw/postgres-14/src/backend/nodes/tidbitmap.c#L3-L30), [tidbitmap.c#tbm_begin_iterate](../raw/postgres-14/src/backend/nodes/tidbitmap.c#L688)). GIN and BRIN leave `amgettuple` NULL ([ginutil.c:77](../raw/postgres-14/src/backend/access/gin/ginutil.c#L77), [brin.c:129](../raw/postgres-14/src/backend/access/brin/brin.c#L129)).
- PostgreSQL 18: Holds. `nodeBitmapHeapscan.c` and `tidbitmap.c` describe the same two-step, possibly lossy scan, and GIN and BRIN still leave `amgettuple` NULL ([nodeBitmapHeapscan.c](../raw/postgres-18/src/backend/executor/nodeBitmapHeapscan.c#L1-L17), [tidbitmap.c](../raw/postgres-18/src/backend/nodes/tidbitmap.c#L3-L30), [ginutil.c:84](../raw/postgres-18/src/backend/access/gin/ginutil.c#L84), [brin.c:296](../raw/postgres-18/src/backend/access/brin/brin.c#L296)).
- PostgreSQL 19: Holds. The MVCC-snapshot note, lossy storage and recheck are unchanged, and GIN and BRIN still leave `amgettuple` NULL ([nodeBitmapHeapscan.c](../raw/postgres-19/src/backend/executor/nodeBitmapHeapscan.c#L1-L16), [tidbitmap.c](../raw/postgres-19/src/backend/nodes/tidbitmap.c#L3-L30), [ginutil.c:85](../raw/postgres-19/src/backend/access/gin/ginutil.c#L85), [brin.c:300](../raw/postgres-19/src/backend/access/brin/brin.c#L300)). Iteration now splits into `tbm_begin_private_iterate()`, which builds the sorted page lists, and a `tbm_begin_iterate()` wrapper that picks a private or shared iterator ([tidbitmap.c#tbm_begin_private_iterate](../raw/postgres-19/src/backend/nodes/tidbitmap.c#L662-L700), [tidbitmap.c#tbm_begin_iterate](../raw/postgres-19/src/backend/nodes/tidbitmap.c#L1560-L1587)).

Related: [TID](#tid), [Index-only scan](#index-only-scan), [GIN](#gin), [BRIN](#brin), [work_mem](#work_mem)

### BKI

**Aliases:** Backend Interface, `postgres.bki`, `genbki.pl`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

BKI is the bootstrap file format that `initdb` uses to create the system catalogs and load their first rows. The build generates `postgres.bki` from the catalog headers and `.dat` files with the Perl script `genbki.pl` ([bki.sgml:40-52](../raw/postgres-17/doc/src/sgml/bki.sgml#L40-L52), [genbki.pl:1-7](../raw/postgres-17/src/backend/catalog/genbki.pl#L1-L7)). `genbki.pl` also writes a derived `pg_*_d.h` header for each catalog, so catalog OIDs and column-number macros come from generated files rather than from hand-written code ([bki.sgml:54-61](../raw/postgres-17/doc/src/sgml/bki.sgml#L54-L61), [genbki.pl:456-475](../raw/postgres-17/src/backend/catalog/genbki.pl#L456-L475)). A separate grammar, `bootparse.y`, reads the BKI file in bootstrap mode ([bootparse.y:4-5](../raw/postgres-17/src/backend/bootstrap/bootparse.y#L4-L5)).

**Version notes:**
- PostgreSQL 12: Holds. `genbki.pl` builds `postgres.bki` and a `pg_*_d.h` header per catalog, and `bootparse.y` parses BKI ([bki.sgml:44-64](../raw/postgres-12/doc/src/sgml/bki.sgml#L44-L64), [genbki.pl:1-8](../raw/postgres-12/src/backend/catalog/genbki.pl#L1-L8), [bootparse.y:1-5](../raw/postgres-12/src/backend/bootstrap/bootparse.y#L1-L5)).
- PostgreSQL 14: Holds. `genbki.pl` builds `postgres.bki` and the `pg_*_d.h` headers, and `bootparse.y` reads BKI in bootstrap mode ([bki.sgml:40-61](../raw/postgres-14/doc/src/sgml/bki.sgml#L40-L61), [genbki.pl:1-7](../raw/postgres-14/src/backend/catalog/genbki.pl#L1-L7), [genbki.pl:422-441](../raw/postgres-14/src/backend/catalog/genbki.pl#L422-L441), [bootparse.y:4-5](../raw/postgres-14/src/backend/bootstrap/bootparse.y#L4-L5)).
- PostgreSQL 18: Holds. `genbki.pl`, the derived headers, and `bootparse.y` are unchanged ([bki.sgml:40-52](../raw/postgres-18/doc/src/sgml/bki.sgml#L40-L52), [genbki.pl:456-475](../raw/postgres-18/src/backend/catalog/genbki.pl#L456-L475), [bootparse.y:4-5](../raw/postgres-18/src/backend/bootstrap/bootparse.y#L4-L5)).
- PostgreSQL 19: Holds. `genbki.pl` still builds `postgres.bki` and the `pg_*_d.h` headers, and `bootparse.y` still reads BKI ([bki.sgml:40-61](../raw/postgres-19/doc/src/sgml/bki.sgml#L40-L61), [genbki.pl:1-7](../raw/postgres-19/src/backend/catalog/genbki.pl#L1-L7), [genbki.pl:456-475](../raw/postgres-19/src/backend/catalog/genbki.pl#L456-L475), [bootparse.y:4-5](../raw/postgres-19/src/backend/bootstrap/bootparse.y#L4-L5)).

Related: [Catalog](#catalog), [OID](#oid), [Syscache](#syscache), [Grammar](#grammar)

### BLCKSZ

**Aliases:** block size, `block_size`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`BLCKSZ` is the compile-time size of one disk block and one shared buffer, 8192 bytes by default. Build configuration only accepts 1, 2, 4, 8, 16 or 32 kB, and changing it needs a new `initdb` ([configure.ac#blocksize](../raw/postgres-17/configure.ac#L258-L289)). Page-derived limits are written in terms of `BLCKSZ`. Two examples are the TOAST size limits, built by `MaximumBytesPerTuple()`, and the free space a B-tree leaves per page, `BTGetTargetPageFreeSpace()` ([heaptoast.h#MaximumBytesPerTuple](../raw/postgres-17/src/include/access/heaptoast.h#L20-L26), [nbtree.h#BTGetTargetPageFreeSpace](../raw/postgres-17/src/include/access/nbtree.h#L1144-L1145)). A running server reports the value through the read-only `block_size` setting. Its GUC context is `internal`, so no restart, reload or `SET` can change it ([guc_tables.c#block_size](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3267-L3276)).

**Version notes:**
- PostgreSQL 12: Holds, with renamed files. The build accepts 1 to 32 kB, defaults to 8 kB, and needs a new `initdb` to change; the source is `configure.in`, not `configure.ac`, and 12 has no Meson build ([configure.in#blocksize](../raw/postgres-12/configure.in#L250-L277)). `MaximumBytesPerTuple()` lives in `tuptoaster.h`, not `heaptoast.h`, and 12 has no `BTGetTargetPageFreeSpace()`; B-tree uses `RelationGetTargetPageFreeSpace()` ([tuptoaster.h#MaximumBytesPerTuple](../raw/postgres-12/src/include/access/tuptoaster.h#L27-L33), [nbtsort.c:728](../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L728)). `block_size` is `PGC_INTERNAL`, so nothing can change it ([guc.c#block_size](../raw/postgres-12/src/backend/utils/misc/guc.c#L2880-L2888)).
- PostgreSQL 14: Holds. `configure.ac` accepts 1 to 32 kB with a default of 8 ([configure.ac#blocksize](../raw/postgres-14/configure.ac#L255-L267)). `MaximumBytesPerTuple()` and `BTGetTargetPageFreeSpace()` are written in `BLCKSZ` ([heaptoast.h#MaximumBytesPerTuple](../raw/postgres-14/src/include/access/heaptoast.h#L23-L26), [nbtree.h#BTGetTargetPageFreeSpace](../raw/postgres-14/src/include/access/nbtree.h#L1101-L1102)). `block_size` is `PGC_INTERNAL` and cannot be changed; it is defined in `guc.c` ([guc.c#block_size](../raw/postgres-14/src/backend/utils/misc/guc.c#L3183-L3191)).
- PostgreSQL 18: Holds. The allowed sizes, the page-derived macros, and `block_size` as an `internal` GUC (no change possible) are unchanged ([configure.ac#blocksize](../raw/postgres-18/configure.ac#L248-L279), [nbtree.h#BTGetTargetPageFreeSpace](../raw/postgres-18/src/include/access/nbtree.h#L1164-L1165), [guc_tables.c#block_size](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3457-L3466)).
- PostgreSQL 19: Holds. configure accepts 1 to 32 kB with default 8 and says a change needs `initdb`, `MaximumBytesPerTuple()` and `BTGetTargetPageFreeSpace()` are still built from `BLCKSZ`, and `block_size` is still `PGC_INTERNAL` ([configure.ac#blocksize](../raw/postgres-19/configure.ac#L248-L274), [heaptoast.h#MaximumBytesPerTuple](../raw/postgres-19/src/include/access/heaptoast.h#L20-L26), [nbtree.h#BTGetTargetPageFreeSpace](../raw/postgres-19/src/include/access/nbtree.h#L1133-L1134), [guc_parameters.dat#block_size](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L374-L381)).

Related: [Block](#block), [Page](#page)

### Bloat

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Bloat is space in data pages that holds no current row versions, such as free space or outdated row versions ([glossary.sgml#glossary-bloat](../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)). For indexes, the `REINDEX` docs describe a bloated index as one with many empty or nearly-empty pages. A rebuild removes those pages by writing a new copy of the index ([ref/reindex.sgml#bloated](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L62)). The docs also say bloat in non-B-tree indexes "has not been well researched" and recommend watching their physical size ([maintenance.sgml#routine-reindex](../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046)). The wiki's v17 bloat test protocols are [Mandatory B-Tree Bloat Tests (unverified)](v17/common-concepts/mandatory-btree-bloat-tests.md), [Mandatory GIN Bloat Tests (unverified)](v17/common-concepts/mandatory-gin-bloat-tests.md) and [Mandatory Non-B-Tree, Non-GIN Bloat Tests (unverified)](v17/common-concepts/mandatory-non-btree-non-gin-bloat-tests.md).

**Version notes:**
- PostgreSQL 12: Holds for the docs cited, but 12 has no `glossary.sgml` definition. `REINDEX` describes a bloated index as one with many empty or nearly-empty pages, and the maintenance docs call non-B-tree bloat not well researched ([ref/reindex.sgml#bloated](../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L56), [maintenance.sgml#routine-reindex](../raw/postgres-12/doc/src/sgml/maintenance.sgml#L877-L879)). The wiki's bloat test pages are v17 pages and do not cover 12.
- PostgreSQL 14: Holds. The docs glossary, the `REINDEX` description of a bloated index, and the "not been well researched" warning for non-B-tree indexes all appear ([glossary.sgml#glossary-bloat](../raw/postgres-14/doc/src/sgml/glossary.sgml#L194-L202), [ref/reindex.sgml#bloated](../raw/postgres-14/doc/src/sgml/ref/reindex.sgml#L53-L62), [maintenance.sgml#routine-reindex](../raw/postgres-14/doc/src/sgml/maintenance.sgml#L1023-L1025)). The linked bloat test pages are v17 pages and do not cover 14.
- PostgreSQL 18: Holds. The glossary definition, the `REINDEX` wording, and the "not been well researched" note for non-B-tree indexes are unchanged ([glossary.sgml#glossary-bloat](../raw/postgres-18/doc/src/sgml/glossary.sgml#L267-L275), [ref/reindex.sgml#bloated](../raw/postgres-18/doc/src/sgml/ref/reindex.sgml#L54-L62), [maintenance.sgml#routine-reindex](../raw/postgres-18/doc/src/sgml/maintenance.sgml#L1065-L1068)). The linked bloat test protocol pages are v17 pages.
- PostgreSQL 19: Holds. The glossary definition, the `REINDEX` "bloated" note, and the "not been well researched" warning for non-B-tree indexes are unchanged ([glossary.sgml#glossary-bloat](../raw/postgres-19/doc/src/sgml/glossary.sgml#L283-L291), [ref/reindex.sgml#bloated](../raw/postgres-19/doc/src/sgml/ref/reindex.sgml#L54-L63), [maintenance.sgml#routine-reindex](../raw/postgres-19/doc/src/sgml/maintenance.sgml#L1253-L1257)). v19 adds `REPACK`, which removes table bloat by rewriting the table into a new file ([ref/repack.sgml](../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L42-L50)). The linked bloat-test pages are v17 pages.

Related: [Fillfactor](#fillfactor), [REINDEX](#reindex), [VACUUM](#vacuum), [VACUUM FULL](#vacuum-full)

### Block

**Aliases:** disk block, `BlockNumber`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A block is one fixed-size unit of a relation's data file, numbered from 0. It is also the unit of I/O: one shared buffer holds exactly one disk block ([block.h#BlockNumber](../raw/postgres-17/src/include/storage/block.h#L17-L35)). Source code addresses blocks with the 32-bit `BlockNumber` and uses `InvalidBlockNumber` (`0xFFFFFFFF`) as "no block" ([block.h#InvalidBlockNumber](../raw/postgres-17/src/include/storage/block.h#L31-L35)). A block is a raw unit of storage; a [page](#page) is the formatted layout that access methods write into a block ([bufpage.h#Page](../raw/postgres-17/src/include/storage/bufpage.h#L22-L28)).

**Version notes:**
- PostgreSQL 12: Holds. `BlockNumber` is a `uint32`, one buffer holds one disk block, and `InvalidBlockNumber` is `0xFFFFFFFF` ([block.h#BlockNumber](../raw/postgres-12/src/include/storage/block.h#L17-L33)).
- PostgreSQL 14: Holds. `BlockNumber` is a 32-bit type and `InvalidBlockNumber` is `0xFFFFFFFF`; `Page` is a pointer to the formatted block ([block.h#BlockNumber](../raw/postgres-14/src/include/storage/block.h#L31-L33), [bufpage.h#Page](../raw/postgres-14/src/include/storage/bufpage.h#L78)).
- PostgreSQL 18: Holds ([block.h#BlockNumber](../raw/postgres-18/src/include/storage/block.h#L17-L35), [bufpage.h#Page](../raw/postgres-18/src/include/storage/bufpage.h#L25-L31)).
- PostgreSQL 19: Holds. `BlockNumber` is still a 32-bit block index, `InvalidBlockNumber` is still `0xFFFFFFFF`, and a page is still the formatted layout on top of a block ([block.h#BlockNumber](../raw/postgres-19/src/include/storage/block.h#L17-L33), [bufpage.h#Page](../raw/postgres-19/src/include/storage/bufpage.h#L25-L28)). In v19, `Page` is declared as `PageData *`, where `PageData` is `char` ([bufpage.h:80-81](../raw/postgres-19/src/include/storage/bufpage.h#L80-L81)).

Related: [BLCKSZ](#blcksz), [Page](#page), [TID](#tid)

### Bottom-up index deletion

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Bottom-up index deletion is a B-tree step that runs just before a leaf page would split. It removes old row versions left behind by `UPDATE`s that did not change the indexed columns. The executor sends a hint that the incoming tuple is a "logically unchanged" duplicate. `_bt_bottomupdel_pass()` then asks the table AM which duplicates are safe to delete ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L557-L579), [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L285-L307)). In `_bt_delete_or_dedup_one_page()`, it runs after simple deletion and before deduplication. Unlike simple deletion, it guesses from heuristics and may find nothing to delete ([nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2781)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The v12 `nbtree` build has no `nbtdedup.o`, so neither bottom-up deletion nor deduplication exists ([nbtree/Makefile:15-16](../raw/postgres-12/src/backend/access/nbtree/Makefile#L15-L16)). Before a split, a full leaf page only gets `_bt_vacuum_one_page()`, which removes `LP_DEAD` items ([nbtinsert.c:753-759](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L753-L759), [nbtinsert.c#_bt_vacuum_one_page](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2245-L2251)).
- PostgreSQL 14: Holds. The README has a "Bottom-Up deletion" section ([nbtree/README](../raw/postgres-14/src/backend/access/nbtree/README#L528-L561)). `_bt_delete_or_dedup_one_page()` tries it on the executor's "unchanged" hint or on unique-index duplicates, after simple deletion and before deduplication ([nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-14/src/backend/access/nbtree/nbtinsert.c#L2746-L2771), [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-14/src/backend/access/nbtree/nbtdedup.c#L305)).
- PostgreSQL 18: Holds. The README, `_bt_bottomupdel_pass()`, and the order inside `_bt_delete_or_dedup_one_page()` are unchanged ([nbtree/README](../raw/postgres-18/src/backend/access/nbtree/README#L557-L579), [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-18/src/backend/access/nbtree/nbtdedup.c#L285-L307), [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-18/src/backend/access/nbtree/nbtinsert.c#L2757-L2781)).
- PostgreSQL 19: Holds. The README still describes the executor hint and heuristic nature, `_bt_bottomupdel_pass()` still asks the table AM, and `_bt_delete_or_dedup_one_page()` still runs simple deletion, then bottom-up deletion, then deduplication ([nbtree/README](../raw/postgres-19/src/backend/access/nbtree/README#L556-L583), [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-19/src/backend/access/nbtree/nbtdedup.c#L285-L310), [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-19/src/backend/access/nbtree/nbtinsert.c#L2803-L2828)).

Related: [Simple index deletion](#simple-index-deletion), [Deduplication](#deduplication), [Page split](#page-split), [HOT](#hot), [MVCC](#mvcc)

### BRIN

**Aliases:** Block Range Index. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A BRIN index stores one small summary per range of consecutive table pages, such as the minimum and maximum value in that range. A query can then skip ranges that cannot match ([brin/README](../raw/postgres-17/src/backend/access/brin/README#L1-L13)). By default one range covers 128 heap pages, set by `pages_per_range` ([brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-17/src/include/access/brin.h#L39)). BRIN stores no row addresses. It therefore supports only `amgetbitmap`, which returns a lossy bitmap of whole page ranges, and the heap scan must recheck every row ([brin/README](../raw/postgres-17/src/backend/access/brin/README#L16-L23), [brin.c:289](../raw/postgres-17/src/backend/access/brin/brin.c#L289)).

**Version notes:**
- PostgreSQL 12: Holds. BRIN keeps one summary per page range, defaults to 128 pages per range, and supports only a lossy `amgetbitmap` that forces a recheck ([brin/README](../raw/postgres-12/src/backend/access/brin/README#L1-L23), [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-12/src/include/access/brin.h#L39), [brin.c:119-120](../raw/postgres-12/src/backend/access/brin/brin.c#L119-L120)).
- PostgreSQL 14: Holds. The README describes the same range summaries and lossy `amgetbitmap`, the default range is 128 pages, and `amgettuple` is NULL ([brin/README](../raw/postgres-14/src/backend/access/brin/README#L1-L23), [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-14/src/include/access/brin.h#L38), [brin.c:129-130](../raw/postgres-14/src/backend/access/brin/brin.c#L129-L130)).
- PostgreSQL 18: Holds. The default of 128 pages per range and bitmap-only access are unchanged ([brin/README](../raw/postgres-18/src/backend/access/brin/README#L1-L13), [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-18/src/include/access/brin.h#L39), [brin.c:296](../raw/postgres-18/src/backend/access/brin/brin.c#L296)).
- PostgreSQL 19: Holds. The README, the 128-page default `pages_per_range`, and bitmap-only access (`amgettuple` NULL) are unchanged ([brin/README](../raw/postgres-19/src/backend/access/brin/README#L1-L24), [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-19/src/include/access/brin.h#L40), [brin.c:300-301](../raw/postgres-19/src/backend/access/brin/brin.c#L300-L301)).

Related: [Access method](#access-method), [Bitmap scan](#bitmap-scan), [Metapage](#metapage)

### B-tree

**Aliases:** btree, nbtree. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A B-tree is PostgreSQL's default index type: a balanced, sorted tree that answers equality and range searches and can return rows in order. `CREATE INDEX` without `USING` picks it ([index.h#DEFAULT_INDEX_TYPE](../raw/postgres-17/src/include/catalog/index.h#L21), [nbtree.c#bthandler](../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L96-L117)). The implementation lives in `src/backend/access/nbtree/`. It follows the Lehman and Yao algorithm, which adds a right-link and a "high key" to each page so that searches can detect a concurrent page split ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L1-L28)). Block 0 is always the metapage ([nbtree.h#BTREE_METAPAGE](../raw/postgres-17/src/include/access/nbtree.h#L148)).

**Version notes:**
- PostgreSQL 12: Holds. `btree` is the default index type, `bthandler` builds its routine, the README describes Lehman and Yao right-links and high keys, and block 0 is the metapage ([index.h#DEFAULT_INDEX_TYPE](../raw/postgres-12/src/include/catalog/index.h#L21), [nbtree.c#bthandler](../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L106-L108), [nbtree/README](../raw/postgres-12/src/backend/access/nbtree/README#L1-L24), [nbtree.h#BTREE_METAPAGE](../raw/postgres-12/src/include/access/nbtree.h#L131)).
- PostgreSQL 14: Holds. `btree` is the default index type, the code follows Lehman and Yao, and block 0 is the metapage ([index.h#DEFAULT_INDEX_TYPE](../raw/postgres-14/src/include/catalog/index.h#L21), [nbtree.c#bthandler](../raw/postgres-14/src/backend/access/nbtree/nbtree.c#L95-L144), [nbtree/README](../raw/postgres-14/src/backend/access/nbtree/README#L1-L12), [nbtree.h#BTREE_METAPAGE](../raw/postgres-14/src/include/access/nbtree.h#L146)).
- PostgreSQL 18: Holds. `btree` is still the default type, the README still describes Lehman and Yao, and block 0 is the metapage ([index.h#DEFAULT_INDEX_TYPE](../raw/postgres-18/src/include/catalog/index.h#L21), [nbtree/README](../raw/postgres-18/src/backend/access/nbtree/README#L1-L28), [nbtree.h#BTREE_METAPAGE](../raw/postgres-18/src/include/access/nbtree.h#L149)). New in 18: scan-key preprocessing can add a "skip array" on a leading column that has no `=` condition, so `WHERE y = 4` on an index on `(x, y)` no longer forces a full index scan ([nbtpreprocesskeys.c](../raw/postgres-18/src/backend/access/nbtree/nbtpreprocesskeys.c#L128-L135)).
- PostgreSQL 19: Holds. `DEFAULT_INDEX_TYPE` is still `"btree"`, the README still describes Lehman and Yao right-links and high keys, and block 0 is still the metapage ([index.h#DEFAULT_INDEX_TYPE](../raw/postgres-19/src/include/catalog/index.h#L27), [nbtree.c#bthandler](../raw/postgres-19/src/backend/access/nbtree/nbtree.c#L118-L148), [nbtree/README](../raw/postgres-19/src/backend/access/nbtree/README#L1-L28), [nbtree.h#BTREE_METAPAGE](../raw/postgres-19/src/include/access/nbtree.h#L149)).

Related: [Access method](#access-method), [Metapage](#metapage), [Page split](#page-split), [Deduplication](#deduplication), [Bottom-up index deletion](#bottom-up-index-deletion)

### B-tree page deletion

**Aliases:** half-dead page, deleted page, page recycling, `_bt_pagedel`, `safexid`, `btpo.xact`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

B-tree page deletion is how [VACUUM](#vacuum) removes a leaf page that has become completely empty, so the page can later be reused. The [B-tree](#b-tree) never merges partly full pages and never deletes the rightmost page on a level ([nbtree/README#page-deletion](../raw/postgres-17/src/backend/access/nbtree/README#L232-L245)). Deletion has two stages. `_bt_mark_page_halfdead()` first removes the page's downlink from its parent and marks it half-dead, so searches skip it. `_bt_unlink_halfdead_page()` then unlinks it from its siblings and marks it deleted ([nbtree/README#page-deletion](../raw/postgres-17/src/backend/access/nbtree/README#L247-L260), [nbtpage.c#_bt_pagedel](../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1802), [nbtree.h#BTP_DELETED](../raw/postgres-17/src/include/access/nbtree.h#L78-L80)). A deleted page is not free yet. It stays as a tombstone, stamped with a `safexid`, until no running scan could still hold a link to it. Only then may VACUUM put it in the [free space map](#free-space-map) for reuse ([nbtree/README#fsm](../raw/postgres-17/src/backend/access/nbtree/README#L383-L399), [nbtree.h#BTPageIsRecyclable](../raw/postgres-17/src/include/access/nbtree.h#L290-L318)).

**Version notes:**
- PostgreSQL 12: The same two-stage deletion exists ([nbtree/README#page-deletion](../raw/postgres-12/src/backend/access/nbtree/README#L200-L223), [nbtree.h#BTP_DELETED](../raw/postgres-12/src/include/access/nbtree.h#L73-L75)). A deleted page stores a 32-bit `btpo.xact` in its opaque area instead of a `safexid`. `_bt_page_recyclable()` recycles it once that XID precedes `RecentGlobalXmin` ([nbtree.h#BTPageOpaqueData](../raw/postgres-12/src/include/access/nbtree.h#L55-L68), [nbtpage.c#_bt_page_recyclable](../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L941-L963)).
- PostgreSQL 14: Deleted pages store a 64-bit `safexid` in `BTDeletedPageData`, replacing the old 32-bit field ([nbtree.h:55-59](../raw/postgres-14/src/include/access/nbtree.h#L55-L59), [nbtree.h#BTDeletedPageData](../raw/postgres-14/src/include/access/nbtree.h#L230-L238)). `BTPageIsRecyclable()` takes only the page ([nbtree.h:290](../raw/postgres-14/src/include/access/nbtree.h#L290)).
- PostgreSQL 18: Holds ([nbtree/README#page-deletion](../raw/postgres-18/src/backend/access/nbtree/README#L232-L260), [nbtree.h#BTPageIsRecyclable](../raw/postgres-18/src/include/access/nbtree.h#L292), [nbtpage.c#_bt_pagedel](../raw/postgres-18/src/backend/access/nbtree/nbtpage.c#L1802)).
- PostgreSQL 19: Holds ([nbtree/README#page-deletion](../raw/postgres-19/src/backend/access/nbtree/README#L232-L260), [nbtree.h#BTPageIsRecyclable](../raw/postgres-19/src/include/access/nbtree.h#L292), [nbtpage.c#_bt_pagedel](../raw/postgres-19/src/backend/access/nbtree/nbtpage.c#L1832)).

Related: [B-tree](#b-tree), [VACUUM](#vacuum), [Free space map](#free-space-map), [Bloat](#bloat), [xmin horizon](#xmin-horizon)

### btree_gin and btree_gist

**Aliases:** `btree_gin`, `btree_gist`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`btree_gin` and `btree_gist` are two [contrib](#contrib) extensions. They add [operator classes](#operator-class) that let [GIN](#gin) and [GiST](#gist) index ordinary scalar types such as `int4`, `text`, `date` and `uuid` with B-tree-like comparison behavior ([btree-gin.sgml:9-22](../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L9-L22), [btree-gist.sgml:9-21](../raw/postgres-17/doc/src/sgml/btree-gist.sgml#L9-L21)). They cannot enforce uniqueness and do not usually beat a plain [B-tree](#b-tree). Their use is a multicolumn GIN or GiST index that mixes a scalar column with a column only GIN or GiST can index ([btree-gin.sgml:24-32](../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L24-L32), [btree-gist.sgml:23-33](../raw/postgres-17/doc/src/sgml/btree-gist.sgml#L23-L33)). `btree_gist` also indexes `<>`, which exclusion constraints can use ([btree-gist.sgml:35-40](../raw/postgres-17/doc/src/sgml/btree-gist.sgml#L35-L40)). Both are marked `trusted`, so a non-superuser with `CREATE` on the database can install them ([btree_gin.control](../raw/postgres-17/contrib/btree_gin/btree_gin.control#L1-L6), [btree_gist.control](../raw/postgres-17/contrib/btree_gist/btree_gist.control#L1-L6)).

**Version notes:**
- PostgreSQL 12: Both modules exist, but their control files have no `trusted` line, so only a superuser can install them ([btree_gin.control](../raw/postgres-12/contrib/btree_gin/btree_gin.control#L1-L5), [btree_gist.control](../raw/postgres-12/contrib/btree_gist/btree_gist.control#L1-L5)).
- PostgreSQL 14: Both are `trusted` ([btree_gin.control](../raw/postgres-14/contrib/btree_gin/btree_gin.control#L1-L6), [btree_gist.control](../raw/postgres-14/contrib/btree_gist/btree_gist.control#L1-L6)).
- PostgreSQL 18: Holds; `btree_gist` is at version 1.8 ([btree_gin.control](../raw/postgres-18/contrib/btree_gin/btree_gin.control#L1-L6), [btree_gist.control](../raw/postgres-18/contrib/btree_gist/btree_gist.control#L1-L6)).
- PostgreSQL 19: `btree_gin` 1.4 adds cross-type operators, such as comparing an `int2` column with an `int4` value ([btree_gin.control](../raw/postgres-19/contrib/btree_gin/btree_gin.control#L1-L6), [btree_gin--1.3--1.4.sql:6-20](../raw/postgres-19/contrib/btree_gin/btree_gin--1.3--1.4.sql#L6-L20)).

Related: [GIN](#gin), [GiST](#gist), [Contrib](#contrib), [Extension](#extension), [Operator class](#operator-class)

### Buffer manager

**Aliases:** bufmgr. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The buffer manager keeps copies of disk blocks in shared memory. Backends read and change pages there instead of going to the file each time. Its main entry points are `ReadBuffer()` (find or load a page and pin it), `ReleaseBuffer()` (unpin it) and `MarkBufferDirty()`, which defers the disk write until buffer replacement or a checkpoint ([bufmgr.c#entry-points](../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L15-L29)). Each buffer has a `BufferDesc` header holding the page's identity (`tag`), an atomic state word with flags, pin count and usage count, and a content lock ([buf_internals.h#BufferDesc](../raw/postgres-17/src/include/storage/buf_internals.h#L245-L256)).

**Version notes:**
- PostgreSQL 12: Holds. `bufmgr.c` names the same `ReadBuffer()`, `ReleaseBuffer()` and `MarkBufferDirty()` entry points ([bufmgr.c#entry-points](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L15-L30)). `BufferDesc` has the same `tag`, atomic `state` word and `content_lock` ([buf_internals.h#BufferDesc](../raw/postgres-12/src/include/storage/buf_internals.h#L178-L190)).
- PostgreSQL 14: Holds. The same entry points are documented, and `BufferDesc` holds the tag, an atomic state word and a content lock ([bufmgr.c#entry-points](../raw/postgres-14/src/backend/storage/buffer/bufmgr.c#L15-L29), [buf_internals.h#BufferDesc](../raw/postgres-14/src/include/storage/buf_internals.h#L182-L193)).
- PostgreSQL 18: Holds, with asynchronous I/O added. The entry points are unchanged ([bufmgr.c#entry-points](../raw/postgres-18/src/backend/storage/buffer/bufmgr.c#L15-L29)). `BufferDesc` gains `io_wref`, set while an asynchronous read or write on the buffer is in progress ([buf_internals.h#BufferDesc](../raw/postgres-18/src/include/storage/buf_internals.h#L258-L271)). The new `io_method` GUC picks the I/O mechanism (`sync`, `worker`, or `io_uring` where built), defaults to `worker`, and has context `postmaster`, so a change needs a restart ([guc_tables.c#io_method](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5444-L5451), [aio.h:42](../raw/postgres-18/src/include/storage/aio.h#L42), [aio.c#io_method_options](../raw/postgres-18/src/backend/storage/aio/aio.c#L64-L71)).
- PostgreSQL 19: Holds for `ReadBuffer()`, `ReleaseBuffer()` and `MarkBufferDirty()`. The entry-point list adds `StartReadBuffer()`, `StartReadBuffers()` and `WaitReadBuffers()` ([bufmgr.c#entry-points](../raw/postgres-19/src/backend/storage/buffer/bufmgr.c#L15-L32)). `BufferDesc` differs: its state word is now 64-bit (`pg_atomic_uint64`), and the buffer content lock is part of that state word rather than an LWLock, with waiters in `lock_waiters` ([buf_internals.h#BufferDesc](../raw/postgres-19/src/include/storage/buf_internals.h#L303-L310), [buf_internals.h#BufferDesc](../raw/postgres-19/src/include/storage/buf_internals.h#L326-L359)).

Related: [Clock sweep](#clock-sweep), [Dirty buffer](#dirty-buffer), [Ring buffer](#ring-buffer), [shared_buffers](#shared_buffers)

### Catalog

**Aliases:** system catalog, `pg_catalog`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A system catalog is an ordinary table in which PostgreSQL records its own metadata: tables, indexes, types, functions, and so on ([glossary.sgml#glossary-system-catalog](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1772-L1788)). Backend C code knows each catalog's row layout and reads and writes it directly ([bki.sgml:6-15](../raw/postgres-17/doc/src/sgml/bki.sgml#L6-L15)). Each catalog is declared in a header under `src/include/catalog/` with the `CATALOG()` macro ([bki.sgml:6-24](../raw/postgres-17/doc/src/sgml/bki.sgml#L6-L24), [genbki.h:23](../raw/postgres-17/src/include/catalog/genbki.h#L23)). The SQL standard uses "catalog" for what PostgreSQL calls a database. The wiki always means the system-catalog sense ([glossary.sgml#glossary-catalog](../raw/postgres-17/doc/src/sgml/glossary.sgml#L314-L326)).

**Version notes:**
- PostgreSQL 12: Holds. Backend C code knows each catalog's layout, and each catalog is declared under `src/include/catalog/` with `CATALOG()` ([bki.sgml:6-24](../raw/postgres-12/doc/src/sgml/bki.sgml#L6-L24), [genbki.h:23](../raw/postgres-12/src/include/catalog/genbki.h#L23)). 12 has no `glossary.sgml`, so the "catalog versus database" doc note cited for 17 does not exist in 12.
- PostgreSQL 14: Holds. The docs glossary defines the system catalog and warns about the SQL-standard "catalog" sense, and catalogs are declared with `CATALOG()` ([glossary.sgml#glossary-system-catalog](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1519-L1535), [glossary.sgml#glossary-catalog](../raw/postgres-14/doc/src/sgml/glossary.sgml#L218-L230), [genbki.h:23](../raw/postgres-14/src/include/catalog/genbki.h#L23)).
- PostgreSQL 18: Holds ([glossary.sgml#glossary-system-catalog](../raw/postgres-18/doc/src/sgml/glossary.sgml#L1821-L1837), [bki.sgml:6-24](../raw/postgres-18/doc/src/sgml/bki.sgml#L6-L24), [genbki.h:23](../raw/postgres-18/src/include/catalog/genbki.h#L23)).
- PostgreSQL 19: Holds. The glossary and `bki.sgml` definitions are unchanged, and catalogs are still declared with `CATALOG()` in `src/include/catalog/` headers ([glossary.sgml#glossary-system-catalog](../raw/postgres-19/doc/src/sgml/glossary.sgml#L1872-L1896), [bki.sgml:6-24](../raw/postgres-19/doc/src/sgml/bki.sgml#L6-L24), [genbki.h:42](../raw/postgres-19/src/include/catalog/genbki.h#L42), [glossary.sgml#glossary-catalog](../raw/postgres-19/doc/src/sgml/glossary.sgml#L355-L373)).

Related: [pg_class](#pg_class), [pg_index](#pg_index), [BKI](#bki), [Syscache](#syscache), [Relcache](#relcache), [OID](#oid)

### Checkpoint

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A point in the WAL at which every data-file change made before it is guaranteed to be on disk. It is also the act of flushing dirty pages to reach that point ([glossary.sgml#checkpoint](../raw/postgres-17/doc/src/sgml/glossary.sgml#L353-L376)). Crash recovery starts replay from the latest checkpoint's redo record, so older WAL segments can be recycled ([wal.sgml](../raw/postgres-17/doc/src/sgml/wal.sgml#L495-L509)). In source, the work is `CreateCheckPoint()`. An online checkpoint writes an `XLOG_CHECKPOINT_REDO` record first and an `XLOG_CHECKPOINT_ONLINE` record when done. A shutdown checkpoint writes one `XLOG_CHECKPOINT_SHUTDOWN` record ([xlog.c#CreateCheckPoint](../raw/postgres-17/src/backend/access/transam/xlog.c#L6827-L6862)). `checkpoint_timeout` is a `sighup` GUC ([guc_tables.c#checkpoint_timeout](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2855-L2862)).

**Version notes:**
- PostgreSQL 12: Differs. Crash recovery starts from the latest checkpoint's redo point, and older segments can be recycled ([wal.sgml](../raw/postgres-12/doc/src/sgml/wal.sgml#L450-L462)). 12 has no `XLOG_CHECKPOINT_REDO` record: `CreateCheckPoint()` writes one record, either `XLOG_CHECKPOINT_SHUTDOWN` or `XLOG_CHECKPOINT_ONLINE` ([pg_control.h:66-68](../raw/postgres-12/src/include/catalog/pg_control.h#L66-L68), [xlog.c#CreateCheckPoint](../raw/postgres-12/src/backend/access/transam/xlog.c#L8810-L8819)). `checkpoint_timeout` is `PGC_SIGHUP` (reload), defined in `guc.c` ([guc.c#checkpoint_timeout](../raw/postgres-12/src/backend/utils/misc/guc.c#L2567-L2570)).
- PostgreSQL 14: Holds, but the WAL records differ. 14 has no `XLOG_CHECKPOINT_REDO` record: an online checkpoint writes one `XLOG_CHECKPOINT_ONLINE` record at the end and a shutdown checkpoint writes `XLOG_CHECKPOINT_SHUTDOWN` ([pg_control.h:67-68](../raw/postgres-14/src/include/catalog/pg_control.h#L67-L68), [xlog.c#CreateCheckPoint](../raw/postgres-14/src/backend/access/transam/xlog.c#L9440-L9448)). Recovery still starts from the checkpoint's redo record ([wal.sgml](../raw/postgres-14/doc/src/sgml/wal.sgml#L498-L509)). `checkpoint_timeout` is `PGC_SIGHUP`, so a change needs a reload ([guc.c#checkpoint_timeout](../raw/postgres-14/src/backend/utils/misc/guc.c#L2793)).
- PostgreSQL 18: Holds. `CreateCheckPoint()` writes the same `XLOG_CHECKPOINT_REDO`, `XLOG_CHECKPOINT_ONLINE` and `XLOG_CHECKPOINT_SHUTDOWN` records. It now returns `bool`, and false means an idle system skipped the checkpoint ([xlog.c#CreateCheckPoint](../raw/postgres-18/src/backend/access/transam/xlog.c#L6891-L6929)). `checkpoint_timeout` is still `sighup` (reload) ([guc_tables.c#checkpoint_timeout](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2983-L2990)).
- PostgreSQL 19: Holds. `CreateCheckPoint()` still writes `XLOG_CHECKPOINT_REDO` then `XLOG_CHECKPOINT_ONLINE` online, or one `XLOG_CHECKPOINT_SHUTDOWN` record, and recovery still starts at the redo record ([glossary.sgml#glossary-checkpoint](../raw/postgres-19/doc/src/sgml/glossary.sgml#L394-L419), [wal.sgml](../raw/postgres-19/doc/src/sgml/wal.sgml#L652-L668), [xlog.c#CreateCheckPoint](../raw/postgres-19/src/backend/access/transam/xlog.c#L7409-L7430)). `checkpoint_timeout` is still `PGC_SIGHUP` (reload) ([guc_parameters.dat#checkpoint_timeout](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L429-L436)).

Related: [Crash recovery](#crash-recovery), [Full-page image](#full-page-image), [WAL](#wal), [Background writer](#background-writer)

### Clock sweep

**Aliases:** clock-sweep, `usage_count`, `nextVictimBuffer`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Clock sweep is the algorithm that chooses which shared buffer to reuse when no free buffer is left. A "clock hand" (`nextVictimBuffer`) moves around all buffers. It skips a pinned buffer, lowers a nonzero usage count by one, and takes the first unpinned buffer whose count is zero ([buffer/README#clock-sweep](../raw/postgres-17/src/backend/storage/buffer/README#L166-L199)). Code: `ClockSweepTick()` advances the hand and `StrategyGetBuffer()` runs the loop ([freelist.c#ClockSweepTick](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L103-L125), [freelist.c#StrategyGetBuffer](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L316-L332)). Each pin raises the usage count up to `BM_MAX_USAGE_COUNT` (5) ([buffer/README#clock-sweep](../raw/postgres-17/src/backend/storage/buffer/README#L171-L174)). The cap keeps the sweep short at the cost of only approximating LRU ([buf_internals.h#BM_MAX_USAGE_COUNT](../raw/postgres-17/src/include/storage/buf_internals.h#L71-L79)).

**Version notes:**
- PostgreSQL 12: Holds. The README describes the `nextVictimBuffer` clock hand, usage counts raised on pin, and decrements on pass ([buffer/README#clock-sweep](../raw/postgres-12/src/backend/storage/buffer/README#L170-L200)). `ClockSweepTick()` and `StrategyGetBuffer()` carry it, and `BM_MAX_USAGE_COUNT` is 5 ([freelist.c#ClockSweepTick](../raw/postgres-12/src/backend/storage/buffer/freelist.c#L113), [freelist.c#StrategyGetBuffer](../raw/postgres-12/src/backend/storage/buffer/freelist.c#L201), [buf_internals.h#BM_MAX_USAGE_COUNT](../raw/postgres-12/src/include/storage/buf_internals.h#L73-L77)).
- PostgreSQL 14: Holds. The README describes the same `nextVictimBuffer` hand and usage-count decrement, `ClockSweepTick()` advances the hand, `StrategyGetBuffer()` runs the loop, and the cap is `BM_MAX_USAGE_COUNT` 5 ([buffer/README#clock-sweep](../raw/postgres-14/src/backend/storage/buffer/README#L166-L199), [freelist.c#ClockSweepTick](../raw/postgres-14/src/backend/storage/buffer/freelist.c#L113-L169), [freelist.c#StrategyGetBuffer](../raw/postgres-14/src/backend/storage/buffer/freelist.c#L315-L333), [buf_internals.h#BM_MAX_USAGE_COUNT](../raw/postgres-14/src/include/storage/buf_internals.h#L70-L77)).
- PostgreSQL 18: Holds. `ClockSweepTick()`, `StrategyGetBuffer()`, and `BM_MAX_USAGE_COUNT` (5) are unchanged ([freelist.c#ClockSweepTick](../raw/postgres-18/src/backend/storage/buffer/freelist.c#L103-L125), [freelist.c#StrategyGetBuffer](../raw/postgres-18/src/backend/storage/buffer/freelist.c#L316-L332), [buf_internals.h#BM_MAX_USAGE_COUNT](../raw/postgres-18/src/include/storage/buf_internals.h#L79-L87)).
- PostgreSQL 19: Holds for the hand `nextVictimBuffer`, `ClockSweepTick()`, the loop in `StrategyGetBuffer()` and the cap `BM_MAX_USAGE_COUNT` = 5 ([freelist.c#ClockSweepTick](../raw/postgres-19/src/backend/storage/buffer/freelist.c#L103-L120), [freelist.c#StrategyGetBuffer](../raw/postgres-19/src/backend/storage/buffer/freelist.c#L238-L260), [buf_internals.h#BM_MAX_USAGE_COUNT](../raw/postgres-19/src/include/storage/buf_internals.h#L136-L144)). v19 has no buffer free list, so the sweep is the only way to get a victim buffer, not a fallback for when "no free buffer is left" ([buffer/README#clock-sweep](../raw/postgres-19/src/backend/storage/buffer/README#L172-L203)). The loop now checks and pins a buffer with a compare-and-swap loop on the 64-bit state word, without locking the buffer header ([freelist.c#StrategyGetBuffer](../raw/postgres-19/src/backend/storage/buffer/freelist.c#L245-L253)).

Related: [Buffer manager](#buffer-manager), [Ring buffer](#ring-buffer), [shared_buffers](#shared_buffers)

### CLUSTER

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`CLUSTER` rewrites a table into a new physical file in the order of one of its indexes, then rebuilds the indexes. The table keeps its OID because only the relfilenumbers are swapped ([cluster.c#cluster_rel](../raw/postgres-17/src/backend/commands/cluster.c#L293-L311)). In source, `rebuild_relation` calls `make_new_heap`, `copy_table_data`, and `finish_heap_swap`, and holds `AccessExclusiveLock` on the table until commit ([cluster.c#rebuild_relation](../raw/postgres-17/src/backend/commands/cluster.c#L624-L674), [cluster.c:337](../raw/postgres-17/src/backend/commands/cluster.c#L337)). The same code path implements [VACUUM FULL](#vacuum-full) when no index is given ([cluster.c:1-4](../raw/postgres-17/src/backend/commands/cluster.c#L1-L4)).

**Version notes:**
- PostgreSQL 12: Holds, with older naming: 12 source says it swaps the "relfilenodes", not relfilenumbers ([cluster.c#cluster_rel](../raw/postgres-12/src/backend/commands/cluster.c#L248-L262)). `rebuild_relation` calls `make_new_heap`, `copy_table_data` and `finish_heap_swap`, `cluster_rel` opens the table with `AccessExclusiveLock`, and the same path implements `VACUUM FULL` ([cluster.c#rebuild_relation](../raw/postgres-12/src/backend/commands/cluster.c#L588-L625), [cluster.c:289](../raw/postgres-12/src/backend/commands/cluster.c#L289), [cluster.c:1-4](../raw/postgres-12/src/backend/commands/cluster.c#L1-L4)).
- PostgreSQL 14: Holds. `cluster_rel` swaps relfilenodes to keep the table OID, `rebuild_relation` calls `make_new_heap`, `copy_table_data` and `finish_heap_swap`, the table is opened with `AccessExclusiveLock`, and the same code serves `VACUUM FULL` ([cluster.c#cluster_rel](../raw/postgres-14/src/backend/commands/cluster.c#L259-L266), [cluster.c#rebuild_relation](../raw/postgres-14/src/backend/commands/cluster.c#L591-L630), [cluster.c:303](../raw/postgres-14/src/backend/commands/cluster.c#L303), [cluster.c:1-4](../raw/postgres-14/src/backend/commands/cluster.c#L1-L4)). 14 names the file number `relfilenode`, not relfilenumber.
- PostgreSQL 18: Holds, with a changed calling convention. `cluster_rel()` now takes an already-open `Relation` that the caller locked with `AccessExclusiveLock`, instead of a table OID ([cluster.c#cluster_rel](../raw/postgres-18/src/backend/commands/cluster.c#L283-L287), [cluster.c#cluster_rel](../raw/postgres-18/src/backend/commands/cluster.c#L311-L321)). `rebuild_relation` still calls `make_new_heap`, `copy_table_data` and `finish_heap_swap`. It now receives the index as an open `Relation`, and its header says locks are kept after it closes the relations ([cluster.c#rebuild_relation](../raw/postgres-18/src/backend/commands/cluster.c#L618-L690)). The file header still says it implements VACUUM FULL ([cluster.c:1-4](../raw/postgres-18/src/backend/commands/cluster.c#L1-L4)).
- PostgreSQL 19: Differs. The code moved from `cluster.c` to `repack.c`, whose header says "REPACK a table; formerly known as CLUSTER" and that VACUUM FULL uses parts of it ([repack.c:1-21](../raw/postgres-19/src/backend/commands/repack.c#L1-L21)). The docs now define `CLUSTER` as equivalent to `REPACK ... USING INDEX` ([ref/cluster.sgml#description](../raw/postgres-19/doc/src/sgml/ref/cluster.sgml#L32-L40)). The mechanism is the same: `cluster_rel()` swaps relfilenumbers so the OID survives, and `rebuild_relation()` calls `make_new_heap()`, `copy_table_data()` and then `finish_heap_swap()` ([repack.c#cluster_rel](../raw/postgres-19/src/backend/commands/repack.c#L465-L491), [repack.c#rebuild_relation](../raw/postgres-19/src/backend/commands/repack.c#L970-L978), [repack.c:1047-1066](../raw/postgres-19/src/backend/commands/repack.c#L1047-L1066), [repack.c:1105-1120](../raw/postgres-19/src/backend/commands/repack.c#L1105-L1120)). Non-concurrent runs take `AccessExclusiveLock`; `REPACK (CONCURRENTLY)` starts with `ShareUpdateExclusiveLock` and upgrades only for the swap ([repack.c#RepackLockLevel](../raw/postgres-19/src/backend/commands/repack.c#L456-L463), [repack.c:12-21](../raw/postgres-19/src/backend/commands/repack.c#L12-L21)).

Related: [VACUUM FULL](#vacuum-full), [Table rewrite](#table-rewrite), [Relfilenumber](#relfilenumber), [AccessExclusiveLock](#accessexclusivelock)

### Collation

**Aliases:** `pg_collation`, `COLLATE`, collation provider, ICU, libc provider. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A collation is a named set of rules for sorting and comparing text. Every expression of a collatable type, such as `text`, carries one, and it is either a column's declared collation or derived from the inputs ([charset.sgml#collation-concepts](../raw/postgres-17/doc/src/sgml/charset.sgml#L649-L672)). Each collation is a row in `pg_collation`. Its `collprovider` says which library implements it: `builtin`, `icu` or `libc` ([pg_collation.h#FormData_pg_collation](../raw/postgres-17/src/include/catalog/pg_collation.h#L29-L50), [pg_collation.h#COLLPROVIDER_DEFAULT](../raw/postgres-17/src/include/catalog/pg_collation.h#L70-L73)). It matters for indexes: `pg_index.indcollation` records each key column's collation, and the planner uses an index for a text comparison only when the collations match ([pg_index.h:53](../raw/postgres-17/src/include/catalog/pg_index.h#L53), [indxpath.c#IndexCollMatchesExprColl](../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L40-L41)).

**Version notes:**
- PostgreSQL 12: Holds, but only `icu` and `libc` providers exist, and `collcollate` and `collctype` are fixed-size `name` columns ([pg_collation.h#FormData_pg_collation](../raw/postgres-12/src/include/catalog/pg_collation.h#L29-L44), [pg_collation.h#COLLPROVIDER_DEFAULT](../raw/postgres-12/src/include/catalog/pg_collation.h#L55-L57)).
- PostgreSQL 14: Holds, with the same two providers and `name` columns as 12 ([pg_collation.h#FormData_pg_collation](../raw/postgres-14/src/include/catalog/pg_collation.h#L29-L49), [pg_collation.h#COLLPROVIDER_DEFAULT](../raw/postgres-14/src/include/catalog/pg_collation.h#L67-L69)).
- PostgreSQL 18: Holds as in 17, including the `builtin` provider ([pg_collation.h#COLLPROVIDER_DEFAULT](../raw/postgres-18/src/include/catalog/pg_collation.h#L70-L73)).
- PostgreSQL 19: Holds as in 17 ([pg_collation.h#FormData_pg_collation](../raw/postgres-19/src/include/catalog/pg_collation.h#L31-L53), [pg_collation.h#COLLPROVIDER_DEFAULT](../raw/postgres-19/src/include/catalog/pg_collation.h#L74-L77)).

Related: [Catalog](#catalog), [pg_index](#pg_index), [Operator class](#operator-class), [Planner](#planner)

### COMMENT ON

**Aliases:** `pg_description`, object comment. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`COMMENT ON` attaches a free-text description to a database object. `CreateComments` stores it as one `pg_description` row, and an empty string removes the comment ([comment.c#CreateComments](../raw/postgres-17/src/backend/commands/comment.c#L143-L170)). The row is keyed by the object's OID, the OID of the catalog that holds the object, and a sub-ID that is the column number for column comments and 0 otherwise ([pg_description.h:10-19](../raw/postgres-17/src/include/catalog/pg_description.h#L10-L19), [pg_description.h#FormData_pg_description](../raw/postgres-17/src/include/catalog/pg_description.h#L48-L57)). `REINDEX CONCURRENTLY` builds an index with a new OID, so `index_concurrently_swap` moves the old index's comment to it ([index.c:1740-1782](../raw/postgres-17/src/backend/catalog/index.c#L1740-L1782)).

**Version notes:**
- PostgreSQL 12: Holds. `CreateComments` stores one `pg_description` row keyed by object OID, class OID and sub-ID, and an empty string removes it ([comment.c#CreateComments](../raw/postgres-12/src/backend/commands/comment.c#L142-L156), [pg_description.h:10-21](../raw/postgres-12/src/include/catalog/pg_description.h#L10-L21)). `REINDEX CONCURRENTLY` exists in 12, and `index_concurrently_swap` moves the comment to the new index OID ([index.c#index_concurrently_swap](../raw/postgres-12/src/backend/catalog/index.c#L1441-L1448), [index.c:1612-1640](../raw/postgres-12/src/backend/catalog/index.c#L1612-L1640)).
- PostgreSQL 14: Holds. `CreateComments` stores one `pg_description` row and deletes it for an empty string; the row key is `objoid`, `classoid` and `objsubid` ([comment.c#CreateComments](../raw/postgres-14/src/backend/commands/comment.c#L142-L225), [pg_description.h#FormData_pg_description](../raw/postgres-14/src/include/catalog/pg_description.h#L48-L56)). `REINDEX CONCURRENTLY` moves the old index's comment to the new index ([index.c:1683-1724](../raw/postgres-14/src/backend/catalog/index.c#L1683-L1724)).
- PostgreSQL 18: Holds ([comment.c#CreateComments](../raw/postgres-18/src/backend/commands/comment.c#L143-L170), [pg_description.h#FormData_pg_description](../raw/postgres-18/src/include/catalog/pg_description.h#L48-L57), [index.c:1743-1785](../raw/postgres-18/src/backend/catalog/index.c#L1743-L1785)).
- PostgreSQL 19: Holds. `CreateComments()` still writes one `pg_description` row keyed by object OID, class OID and sub-ID, and an empty string still deletes it. `index_concurrently_swap()` still moves the old index's comment to the new OID ([comment.c#CreateComments](../raw/postgres-19/src/backend/commands/comment.c#L145-L166), [pg_description.h:10-21](../raw/postgres-19/src/include/catalog/pg_description.h#L10-L21), [pg_description.h#FormData_pg_description](../raw/postgres-19/src/include/catalog/pg_description.h#L50-L59), [index.c:1758-1790](../raw/postgres-19/src/backend/catalog/index.c#L1758-L1790)).

Related: [OID](#oid), [REINDEX](#reindex), [CONCURRENTLY](#concurrently), [Catalog](#catalog)

### Common table expression

**Aliases:** CTE, `WITH` query, `MATERIALIZED`, `NOT MATERIALIZED`, `CommonTableExpr`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A common table expression is a named auxiliary query written in a `WITH` clause and referenced by the main query like a table ([queries.sgml#queries-with](../raw/postgres-17/doc/src/sgml/queries.sgml#L2047-L2062)). The parser stores each one as a `CommonTableExpr` node, whose `ctematerialized` field records `MATERIALIZED`, `NOT MATERIALIZED` or no choice ([parsenodes.h#CTEMaterialize](../raw/postgres-17/src/include/nodes/parsenodes.h#L1636-L1641), [parsenodes.h#CommonTableExpr](../raw/postgres-17/src/include/nodes/parsenodes.h#L1668-L1680)). The choice matters to the planner. `SS_process_ctes()` inlines a non-recursive, side-effect-free `SELECT` CTE as a subquery when it is marked `NOT MATERIALIZED`, or when it has no marking and only one reference ([subselect.c#SS_process_ctes](../raw/postgres-17/src/backend/optimizer/plan/subselect.c#L934-L951)). Otherwise the planner plans the CTE separately as a `SubPlan` ([subselect.c#SS_process_ctes](../raw/postgres-17/src/backend/optimizer/plan/subselect.c#L991-L997)). A separately planned CTE is evaluated once per execution of the parent query, and the planner cannot push the parent's conditions down into a multiply-referenced one ([queries.sgml#queries-with-cte-materialization](../raw/postgres-17/doc/src/sgml/queries.sgml#L2500-L2518)).

**Version notes:**
- PostgreSQL 12: Holds, with the same three markings and inlining rule ([parsenodes.h#CTEMaterialize](../raw/postgres-12/src/include/nodes/parsenodes.h#L1417-L1422), [subselect.c#SS_process_ctes](../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L882-L899)). `CommonTableExpr` has no `SEARCH` or `CYCLE` clause fields ([parsenodes.h#CommonTableExpr](../raw/postgres-12/src/include/nodes/parsenodes.h#L1424-L1441)).
- PostgreSQL 14: Holds ([parsenodes.h#CTEMaterialize](../raw/postgres-14/src/include/nodes/parsenodes.h#L1464-L1469), [subselect.c#SS_process_ctes](../raw/postgres-14/src/backend/optimizer/plan/subselect.c#L947-L964)). `CommonTableExpr` gains `search_clause` and `cycle_clause` fields ([parsenodes.h:1504-1505](../raw/postgres-14/src/include/nodes/parsenodes.h#L1504)).
- PostgreSQL 18: Holds ([parsenodes.h#CTEMaterialize](../raw/postgres-18/src/include/nodes/parsenodes.h#L1665-L1670), [subselect.c#SS_process_ctes](../raw/postgres-18/src/backend/optimizer/plan/subselect.c#L934-L951)).
- PostgreSQL 19: Holds ([parsenodes.h#CTEMaterialize](../raw/postgres-19/src/include/nodes/parsenodes.h#L1741-L1746), [subselect.c#SS_process_ctes](../raw/postgres-19/src/backend/optimizer/plan/subselect.c#L942-L959)).

Related: [Planner](#planner), [Parse tree](#parse-tree), [Function volatility](#function-volatility)

### CONCURRENTLY

**Aliases:** CIC (`CREATE INDEX CONCURRENTLY`), RIC (`REINDEX CONCURRENTLY`), `DROP INDEX CONCURRENTLY`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`CONCURRENTLY` is an option that builds, rebuilds or drops an index without blocking writes to the table. It costs more total work: two table scans and waits for older transactions to finish ([ref/create_index.sgml](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L612-L643), [ref/reindex.sgml](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L367-L380)). In source, `DefineIndex()` takes `ShareUpdateExclusiveLock` instead of `ShareLock` when the option is set ([indexcmds.c:678](../raw/postgres-17/src/backend/commands/indexcmds.c#L678)). `index_set_state_flags()` moves the index through the `pg_index` states in separate transactions ([index.c#index_set_state_flags](../raw/postgres-17/src/backend/catalog/index.c#L3469-L3503)). `ReindexRelationConcurrently()` implements `REINDEX CONCURRENTLY`, which first adds a new transient index definition to `pg_index` to replace the old index ([indexcmds.c#ReindexRelationConcurrently](../raw/postgres-17/src/backend/commands/indexcmds.c#L3428-L3451), [ref/reindex.sgml](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L382-L394)). A failed concurrent build leaves an invalid index behind ([ref/create_index.sgml](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L650)).

**Version notes:**
- PostgreSQL 12: Holds. 12 is the first release with `REINDEX CONCURRENTLY`. The docs describe two table scans and waits for older transactions ([ref/create_index.sgml](../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L544-L556), [ref/reindex.sgml](../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L284-L313)). `DefineIndex()` takes `ShareUpdateExclusiveLock` instead of `ShareLock`, and `index_set_state_flags()` and `ReindexRelationConcurrently()` exist ([indexcmds.c:563](../raw/postgres-12/src/backend/commands/indexcmds.c#L563), [index.c#index_set_state_flags](../raw/postgres-12/src/backend/catalog/index.c#L3332), [indexcmds.c#ReindexRelationConcurrently](../raw/postgres-12/src/backend/commands/indexcmds.c#L2715-L2739)). In 12, `REINDEX CONCURRENTLY` on a partitioned table only warns that it is not yet supported ([indexcmds.c:2721-2723](../raw/postgres-12/src/backend/commands/indexcmds.c#L2721-L2723)). A failed build leaves an invalid index ([ref/create_index.sgml](../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L580)).
- PostgreSQL 14: Holds. `DefineIndex()` takes `ShareUpdateExclusiveLock` instead of `ShareLock`, `index_set_state_flags()` walks the `pg_index` states, and `ReindexRelationConcurrently()` adds a transient index ([indexcmds.c:663](../raw/postgres-14/src/backend/commands/indexcmds.c#L663), [index.c#index_set_state_flags](../raw/postgres-14/src/backend/catalog/index.c#L3467-L3539), [indexcmds.c#ReindexRelationConcurrently](../raw/postgres-14/src/backend/commands/indexcmds.c#L3384), [ref/reindex.sgml](../raw/postgres-14/doc/src/sgml/ref/reindex.sgml#L352-L374)). A failed build leaves an invalid index ([ref/create_index.sgml](../raw/postgres-14/doc/src/sgml/ref/create_index.sgml#L632-L636)).
- PostgreSQL 18: Holds. The docs, the `ShareUpdateExclusiveLock` choice in `DefineIndex()`, `index_set_state_flags()`, and `ReindexRelationConcurrently()` are unchanged ([ref/create_index.sgml](../raw/postgres-18/doc/src/sgml/ref/create_index.sgml#L618-L649), [indexcmds.c:681](../raw/postgres-18/src/backend/commands/indexcmds.c#L681), [index.c#index_set_state_flags](../raw/postgres-18/src/backend/catalog/index.c#L3523-L3557), [indexcmds.c#ReindexRelationConcurrently](../raw/postgres-18/src/backend/commands/indexcmds.c#L3561-L3584)).
- PostgreSQL 19: Holds. The docs still describe two table scans and waits, `DefineIndex()` still picks `ShareUpdateExclusiveLock` over `ShareLock`, `index_set_state_flags()` still steps the `pg_index` flags, and `ReindexRelationConcurrently()` still adds a transient index first ([ref/create_index.sgml](../raw/postgres-19/doc/src/sgml/ref/create_index.sgml#L618-L646), [indexcmds.c:685](../raw/postgres-19/src/backend/commands/indexcmds.c#L685), [index.c#index_set_state_flags](../raw/postgres-19/src/backend/catalog/index.c#L3541-L3551), [indexcmds.c#ReindexRelationConcurrently](../raw/postgres-19/src/backend/commands/indexcmds.c#L3590-L3612), [ref/reindex.sgml](../raw/postgres-19/doc/src/sgml/ref/reindex.sgml#L385-L396)). A failed build still leaves an invalid index ([ref/create_index.sgml](../raw/postgres-19/doc/src/sgml/ref/create_index.sgml#L650-L673)). v19 also has `REPACK (CONCURRENTLY)`, a table rewrite rather than an index option ([mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-19/doc/src/sgml/mvcc.sgml#L1101-L1103)).

Related: [Invalid index](#invalid-index), [REINDEX](#reindex), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [Snapshot](#snapshot), [pg_index](#pg_index)

### Conflict detection

**Aliases:** logical replication conflict, `ConflictType`, `insert_exists`, `update_origin_differs`, `ReportApplyConflict`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

In PostgreSQL 18, conflict detection is the subscriber-side code that classifies and logs cases where an incoming logical replication change does not fit the local data. It lives in `conflict.c` ([conflict.c header](../raw/postgres-18/src/backend/replication/logical/conflict.c#L1-L12)). `ConflictType` names seven cases, such as `CT_INSERT_EXISTS` for a unique-key clash and `CT_UPDATE_ORIGIN_DIFFERS` for a row last changed by a different origin ([conflict.h#ConflictType](../raw/postgres-18/src/include/replication/conflict.h#L31-L59)). `ReportApplyConflict()` logs the conflict, and each type is counted in `pg_stat_subscription_stats` ([conflict.c#ReportApplyConflict](../raw/postgres-18/src/backend/replication/logical/conflict.c#L103-L120), [logical-replication.sgml#logical-replication-conflicts](../raw/postgres-18/doc/src/sgml/logical-replication.sgml#L1746-L1775)). Origin-based types need the local row's commit origin. That data exists only when `track_commit_timestamp` is on ([conflict.c#GetTupleTransactionInfo](../raw/postgres-18/src/backend/replication/logical/conflict.c#L62-L85)). `track_commit_timestamp` has context `postmaster`, so turning it on needs a restart ([guc_tables.c#track_commit_timestamp](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1099-L1107)). Detection only reports. It does not resolve anything: `insert_exists` raises an error until someone fixes the row by hand ([logical-replication.sgml#conflict-insert-exists](../raw/postgres-18/doc/src/sgml/logical-replication.sgml#L1762-L1775)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The logical replication makefile builds no `conflict.o` ([logical/Makefile#OBJS](../raw/postgres-12/src/backend/replication/logical/Makefile#L17-L18)). The docs only say a constraint violation stops replication and missing rows are skipped ([logical-replication.sgml#logical-replication-conflicts](../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L308-L318)).
- PostgreSQL 14: Not present in PostgreSQL 14 ([logical/Makefile#OBJS](../raw/postgres-14/src/backend/replication/logical/Makefile#L17-L29), [logical-replication.sgml#logical-replication-conflicts](../raw/postgres-14/doc/src/sgml/logical-replication.sgml#L320-L330)).
- PostgreSQL 17: Not present in PostgreSQL 17 ([logical/Makefile#OBJS](../raw/postgres-17/src/backend/replication/logical/Makefile#L17-L31), [logical-replication.sgml#logical-replication-conflicts](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L1602-L1612)).
- PostgreSQL 19: Present, with an eighth type, `CT_UPDATE_DELETED`, for a row another origin deleted concurrently ([conflict.h#ConflictType](../raw/postgres-19/src/include/replication/conflict.h#L31-L62)). It is detected only when `track_commit_timestamp` and the subscription option `retain_dead_tuples` are both on ([logical-replication.sgml#conflict-update-deleted](../raw/postgres-19/doc/src/sgml/logical-replication.sgml#L2068-L2082), [pg_subscription.h:83-84](../raw/postgres-19/src/include/catalog/pg_subscription.h#L83-L84)). `track_commit_timestamp` is still `postmaster` context (restart) ([guc_parameters.dat#track_commit_timestamp](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3199)).

Related: [Logical replication](#logical-replication), [Apply worker](#apply-worker), [Replication origin](#replication-origin), [Subscription](#subscription), [Origin filter](#origin-filter)

### Constraint exclusion

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Constraint exclusion is a planner check that skips a table when its `CHECK` constraints prove that no row can match the query's `WHERE` clauses [plancat.c#relation_excluded_by_constraints](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1576-L1600). The `constraint_exclusion` GUC controls it. Its context is `user`, so a session or transaction can `SET` it with no reload or restart. Its default `partition` applies the test only to appendrel members, such as inheritance children [guc_tables.c#constraint_exclusion](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4798-L4807) [plancat.c:1625-1641](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1625-L1641). Do not confuse it with partition pruning. Declarative partitions are pruned first from partition bounds, so the `partition` mode does not re-test their partition constraints [plancat.c:1631-1638](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1631-L1638).

**Version notes:**
- PostgreSQL 12: Holds. `relation_excluded_by_constraints()` skips a relation whose constraints contradict its restrictions ([plancat.c#relation_excluded_by_constraints](../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1380-L1392)). `constraint_exclusion` is `PGC_USERSET` (session scope) with default `partition` ([guc.c#constraint_exclusion](../raw/postgres-12/src/backend/utils/misc/guc.c#L4243-L4251)). In 12, `partition` mode also treats an inherited `UPDATE`/`DELETE` target as an appendrel member, because 12 still plans those through `inheritance_planner`; pruned partitions are not re-tested ([plancat.c:1441-1458](../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1441-L1458)).
- PostgreSQL 14: Holds. `relation_excluded_by_constraints()` has the same `off`/`partition`/`on` switch, and `partition` mode tests only appendrel members because pruning already handled partition bounds ([plancat.c#relation_excluded_by_constraints](../raw/postgres-14/src/backend/optimizer/util/plancat.c#L1541-L1568)). `constraint_exclusion` is `PGC_USERSET` (session scope) with default `partition`, defined in `guc.c` ([guc.c#constraint_exclusion](../raw/postgres-14/src/backend/utils/misc/guc.c#L4679-L4688)).
- PostgreSQL 18: Holds. `constraint_exclusion` is still `user` context (session scope) with default `partition` ([plancat.c#relation_excluded_by_constraints](../raw/postgres-18/src/backend/optimizer/util/plancat.c#L1605-L1629), [guc_tables.c#constraint_exclusion](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5061-L5070), [plancat.c:1654-1670](../raw/postgres-18/src/backend/optimizer/util/plancat.c#L1654-L1670)).
- PostgreSQL 19: Holds. `relation_excluded_by_constraints()` still does the check, and the `partition` mode still handles only appendrel members because pruning has already run ([plancat.c#relation_excluded_by_constraints](../raw/postgres-19/src/backend/optimizer/util/plancat.c#L1840-L1913)). `constraint_exclusion` is still `PGC_USERSET` (session scope) with default `partition` ([guc_parameters.dat#constraint_exclusion](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L526-L533)).

Related: [Partition pruning](#partition-pruning), [Planner](#planner), [RelOptInfo](#reloptinfo)

### Contrib

**Aliases:** `contrib/`, additional supplied modules. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Contrib is the source directory of optional modules that ship with PostgreSQL but are not part of the core server ([contrib.sgml:7-15](../raw/postgres-17/doc/src/sgml/contrib.sgml#L7-L15), [README:1-9](../raw/postgres-17/contrib/README#L1-L9)). Most contrib modules are [extensions](#extension): they ship a `.control` file and are installed per database with `CREATE EXTENSION` ([README:19-25](../raw/postgres-17/contrib/README#L19-L25), [pageinspect.control](../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L5)). Some are only loadable libraries. For example, `auto_explain` is loaded with `LOAD` rather than `CREATE EXTENSION` ([auto-explain.sgml:18-23](../raw/postgres-17/doc/src/sgml/auto-explain.sgml#L18-L23)). A reader should not treat "contrib module" and "extension" as the same thing.

**Version notes:**
- PostgreSQL 12: Holds. `contrib/` holds optional modules outside the core server, most installed with `CREATE EXTENSION`, and `auto_explain` is loaded with `LOAD` ([contrib.sgml:6-15](../raw/postgres-12/doc/src/sgml/contrib.sgml#L6-L15), [README:1-25](../raw/postgres-12/contrib/README#L1-L25), [auto-explain.sgml:23](../raw/postgres-12/doc/src/sgml/auto-explain.sgml#L23)).
- PostgreSQL 14: Holds. The README and docs describe the same optional modules, most register with `CREATE EXTENSION`, and `auto_explain` is loaded with `LOAD` ([README:1-9](../raw/postgres-14/contrib/README#L1-L9), [README:18-25](../raw/postgres-14/contrib/README#L18-L25), [contrib.sgml:7-15](../raw/postgres-14/doc/src/sgml/contrib.sgml#L7-L15), [pageinspect.control](../raw/postgres-14/contrib/pageinspect/pageinspect.control#L1-L5), [auto-explain.sgml:23](../raw/postgres-14/doc/src/sgml/auto-explain.sgml#L23)).
- PostgreSQL 18: Holds ([contrib.sgml:7-15](../raw/postgres-18/doc/src/sgml/contrib.sgml#L7-L15), [README:1-9](../raw/postgres-18/contrib/README#L1-L9), [pageinspect.control](../raw/postgres-18/contrib/pageinspect/pageinspect.control#L1-L5), [auto-explain.sgml:18-23](../raw/postgres-18/doc/src/sgml/auto-explain.sgml#L18-L23)).
- PostgreSQL 19: Holds. The README and appendix wording and the `CREATE EXTENSION` path are unchanged, and `auto_explain` is still loaded with `LOAD` ([contrib.sgml:7-16](../raw/postgres-19/doc/src/sgml/contrib.sgml#L7-L16), [README:1-28](../raw/postgres-19/contrib/README#L1-L28), [pageinspect.control](../raw/postgres-19/contrib/pageinspect/pageinspect.control#L1-L5), [auto-explain.sgml:18-24](../raw/postgres-19/doc/src/sgml/auto-explain.sgml#L18-L24)). v19 adds contrib `pg_plan_advice`, another library that is loaded, not installed with `CREATE EXTENSION` ([pgplanadvice.sgml](../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml#L30-L39)).

Related: [Extension](#extension), [pageinspect](#pageinspect), [pgstattuple](#pgstattuple), [pg_freespacemap](#pg_freespacemap), [Hook](#hook)

### Correlation

**Aliases:** `pg_stats.correlation`, `STATISTIC_KIND_CORRELATION`, physical-order correlation, `indexCorrelation`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Correlation is a per-column statistic between −1 and +1. It measures how closely the table's physical row order follows the column's sorted order ([pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-17/src/include/catalog/pg_statistic.h#L212-L222)). ANALYZE computes it with the other scalar statistics, and `pg_stats` shows it as the `correlation` column ([analyze.c:2807-2834](../raw/postgres-17/src/backend/commands/analyze.c#L2807-L2834), [system_views.sql#pg_stats](../raw/postgres-17/src/backend/catalog/system_views.sql#L219-L225)). It matters for index scan costing. `cost_index()` computes a best case, where matching rows sit on consecutive heap pages, and a worst case, where every heap page fetch is random. It then blends the two by the square of the correlation ([costsize.c#cost_index](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L730-L746), [costsize.c:785-787](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L785-L787)). For a B-tree, `btcostestimate()` takes the value from the leading index column's statistics and scales it by 0.75 for a multicolumn index ([selfuncs.c#btcostestimate](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7123), [selfuncs.c#btcostestimate](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7182-L7199)).

**Version notes:**
- PostgreSQL 12: Holds ([pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-12/src/include/catalog/pg_statistic.h#L201-L210), [costsize.c:712-714](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L712-L714)).
- PostgreSQL 14: Holds ([pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-14/src/include/catalog/pg_statistic.h#L209-L218), [costsize.c:727-729](../raw/postgres-14/src/backend/optimizer/path/costsize.c#L727-L729)).
- PostgreSQL 18: Holds ([pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-18/src/include/catalog/pg_statistic.h#L213-L222), [costsize.c:795-797](../raw/postgres-18/src/backend/optimizer/path/costsize.c#L795-L797)).
- PostgreSQL 19: Holds ([pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-19/src/include/catalog/pg_statistic.h#L217-L226), [costsize.c:784-786](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L784-L786)).

Related: [Statistics](#statistics), [Most common values and histogram](#most-common-values-and-histogram), [Cost](#cost), [Index-only scan](#index-only-scan), [CLUSTER](#cluster)

### Cost

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A cost is the planner's estimate of how expensive a plan step is, in arbitrary units anchored to `seq_page_cost` (one sequential page read) [costsize.c header](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L6-L30). Every path carries a `startup_cost`, spent before the first row, and a `total_cost`, spent to fetch all rows [pathnodes.h#Path](../raw/postgres-17/src/include/nodes/pathnodes.h#L1662-L1665) [costsize.c:37-46](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L37-L46). The C type `Cost` is a `double` [nodes.h:251](../raw/postgres-17/src/include/nodes/nodes.h#L251). The base parameters (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, `cpu_index_tuple_cost`, `cpu_operator_cost`) are globals in `costsize.c` [costsize.c:119-128](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L119-L128). Each index access method prices its own scans through its `amcostestimate` callback [costsize.c#cost_index](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L612-L622).

**Version notes:**
- PostgreSQL 12: Holds. Costs use arbitrary units anchored to `seq_page_cost`, each path has `startup_cost` and `total_cost`, `Cost` is a `double`, and `cost_index()` calls the AM's `amcostestimate` ([costsize.c header](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L6-L14), [pathnodes.h#Path](../raw/postgres-12/src/include/nodes/pathnodes.h#L1121-L1122), [nodes.h:658](../raw/postgres-12/src/include/nodes/nodes.h#L658), [costsize.c:110-114](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L110-L114), [costsize.c#cost_index](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L476), [costsize.c:544](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L544)).
- PostgreSQL 14: Holds. Paths carry `startup_cost` and `total_cost`, `Cost` is a `double`, the base cost parameters are globals in `costsize.c`, and `cost_index()` calls the AM's `amcostestimate` ([pathnodes.h#Path](../raw/postgres-14/src/include/nodes/pathnodes.h#L1191-L1192), [nodes.h:673](../raw/postgres-14/src/include/nodes/nodes.h#L673), [costsize.c:119-123](../raw/postgres-14/src/backend/optimizer/path/costsize.c#L119-L123), [costsize.c#cost_index](../raw/postgres-14/src/backend/optimizer/path/costsize.c#L559-L561)).
- PostgreSQL 18: Holds, with a new comparison key. A `Path` now also carries `disabled_nodes`, a count of disabled plan nodes such as a seq scan under `enable_seqscan = off` ([pathnodes.h#Path](../raw/postgres-18/src/include/nodes/pathnodes.h#L1794-L1797)). `add_path()` treats that count as a higher-order part of the cost, so startup and total cost only break ties between paths with equal counts ([pathnode.c#add_path](../raw/postgres-18/src/backend/optimizer/util/pathnode.c#L391-L420)). The cost units, `Cost` type, base parameters and `amcostestimate` are unchanged ([costsize.c header](../raw/postgres-18/src/backend/optimizer/path/costsize.c#L6-L30), [nodes.h:257](../raw/postgres-18/src/include/nodes/nodes.h#L257), [costsize.c:130-139](../raw/postgres-18/src/backend/optimizer/path/costsize.c#L130-L139)).
- PostgreSQL 19: Holds. Costs are still arbitrary units anchored to `seq_page_cost`, every path still has `startup_cost` and `total_cost`, `Cost` is still a `double`, and `cost_index()` still calls the AM's `amcostestimate` ([costsize.c header](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L6-L40), [pathnodes.h#Path](../raw/postgres-19/src/include/nodes/pathnodes.h#L2005-L2008), [nodes.h:259](../raw/postgres-19/src/include/nodes/nodes.h#L259), [costsize.c:131-140](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L131-L140), [costsize.c#cost_index](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L545-L623)). In v19, a `Path` also counts `disabled_nodes`, and `add_path()` treats that count as a higher-order part of cost than `startup_cost` and `total_cost` ([pathnodes.h#Path](../raw/postgres-19/src/include/nodes/pathnodes.h#L2004-L2006), [pathnode.c#add_path](../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L395-L405)).

Related: [Path](#path), [Planner](#planner), [Selectivity](#selectivity), [effective_cache_size](#effective_cache_size)

### Covering index

**Aliases:** `INCLUDE` columns, non-key columns, `indnkeyatts`, `amcaninclude`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A covering index stores extra non-key columns, listed in `CREATE INDEX ... INCLUDE (...)`, next to its key columns. Searches cannot use non-key columns, and uniqueness ignores them. An [index-only scan](#index-only-scan) can return them without visiting the table ([ref/create_index.sgml#INCLUDE](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L153-L165)). `pg_index` records the split: `indnatts` counts all columns and `indnkeyatts` counts only the key columns ([pg_index.h#FormData_pg_index](../raw/postgres-17/src/include/catalog/pg_index.h#L30-L35)). An [access method](#access-method) opts in with `amcaninclude`. In PostgreSQL 17, B-tree, GiST and SP-GiST set it ([nbtree.c:121](../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L121), [gist.c:79](../raw/postgres-17/src/backend/access/gist/gist.c#L79), [spgutils.c:64](../raw/postgres-17/src/backend/access/spgist/spgutils.c#L64)). Non-key columns widen every index tuple. A B-tree with any `INCLUDE` column never uses [deduplication](#deduplication), because `_bt_allequalimage()` returns false for it ([ref/create_index.sgml:167-174](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L167-L174), [nbtutils.c#_bt_allequalimage](../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5140-L5147)).

**Version notes:**
- PostgreSQL 12: `INCLUDE` and `indnkeyatts` exist ([ref/create_index.sgml#INCLUDE](../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L153-L165), [pg_index.h:34](../raw/postgres-12/src/include/catalog/pg_index.h#L34)). Only B-tree and GiST support it; SP-GiST sets `amcaninclude = false` ([nbtree.c:125](../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L125), [gist.c:77](../raw/postgres-12/src/backend/access/gist/gist.c#L77), [spgutils.c:58](../raw/postgres-12/src/backend/access/spgist/spgutils.c#L58)). B-tree deduplication does not exist in 12: the metapage has no `allequalimage` field ([nbtree.h#BTMetaPageData](../raw/postgres-12/src/include/access/nbtree.h#L97-L110)), so the deduplication rule does not apply.
- PostgreSQL 14: SP-GiST also supports `INCLUDE` ([spgutils.c:63](../raw/postgres-14/src/backend/access/spgist/spgutils.c#L63)), and `_bt_allequalimage()` rejects `INCLUDE` indexes ([nbtutils.c:2717](../raw/postgres-14/src/backend/access/nbtree/nbtutils.c#L2717)).
- PostgreSQL 18: Holds ([nbtree.c:138](../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L138), [gist.c:82](../raw/postgres-18/src/backend/access/gist/gist.c#L82), [spgutils.c:67](../raw/postgres-18/src/backend/access/spgist/spgutils.c#L67), [nbtutils.c:4264](../raw/postgres-18/src/backend/access/nbtree/nbtutils.c#L4264)).
- PostgreSQL 19: Holds ([nbtree.c:141](../raw/postgres-19/src/backend/access/nbtree/nbtree.c#L141), [gist.c:82](../raw/postgres-19/src/backend/access/gist/gist.c#L82), [spgutils.c:67](../raw/postgres-19/src/backend/access/spgist/spgutils.c#L67), [nbtutils.c:1180](../raw/postgres-19/src/backend/access/nbtree/nbtutils.c#L1180)).

Related: [Index-only scan](#index-only-scan), [pg_index](#pg_index), [Deduplication](#deduplication), [B-tree](#b-tree)

### Crash recovery

**Aliases:** WAL replay, redo. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The startup step that replays WAL from the last checkpoint's redo point after an unclean shutdown. It re-applies every change that may not have reached the data files ([wal.sgml](../raw/postgres-17/doc/src/sgml/wal.sgml#L502-L505)). `PerformWalRecovery()` drives it, and it never runs after a clean shutdown ([xlogrecovery.c#PerformWalRecovery](../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L1656-L1662)). Each record goes to its resource manager's `rm_redo` callback ([xlogrecovery.c#ApplyWalRecord](../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L1997-L2001)). Redo compares the page LSN with the record's position to skip changes already applied ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L420-L422)).

**Version notes:**
- PostgreSQL 12: Holds, with a different home. Recovery replays WAL from the latest checkpoint's redo record ([wal.sgml](../raw/postgres-12/doc/src/sgml/wal.sgml#L454-L457)). 12 has no `xlogrecovery.c` and no `PerformWalRecovery()`; `StartupXLOG()` in `xlog.c` runs redo and calls each record's `rm_redo` ([xlog.c#StartupXLOG](../raw/postgres-12/src/backend/access/transam/xlog.c#L6198), [xlog.c:7183](../raw/postgres-12/src/backend/access/transam/xlog.c#L7183)). Redo uses the page LSN to skip changes already applied ([transam/README](../raw/postgres-12/src/backend/access/transam/README#L411-L413)).
- PostgreSQL 14: Holds, but the code lives elsewhere. 14 has no `xlogrecovery.c` or `PerformWalRecovery()`. `StartupXLOG()` in `xlog.c` sets `InRecovery` when the control file shows an unclean shutdown, and its replay loop calls each record's `rm_redo` ([xlog.c#StartupXLOG](../raw/postgres-14/src/backend/access/transam/xlog.c#L7105-L7118), [xlog.c:7575-7576](../raw/postgres-14/src/backend/access/transam/xlog.c#L7575-L7576)). Redo still skips changes by comparing the page LSN ([transam/README](../raw/postgres-14/src/backend/access/transam/README#L420-L422)).
- PostgreSQL 18: Holds ([wal.sgml](../raw/postgres-18/doc/src/sgml/wal.sgml#L500-L503), [xlogrecovery.c#PerformWalRecovery](../raw/postgres-18/src/backend/access/transam/xlogrecovery.c#L1674-L1680), [xlogrecovery.c#ApplyWalRecord](../raw/postgres-18/src/backend/access/transam/xlogrecovery.c#L2016-L2020)).
- PostgreSQL 19: Holds. Replay still starts from the latest checkpoint's redo record, `PerformWalRecovery()` still never runs after a clean shutdown, each record still goes to its resource manager's `rm_redo`, and redo still skips changes whose page LSN is already past the record ([wal.sgml](../raw/postgres-19/doc/src/sgml/wal.sgml#L660-L663), [xlogrecovery.c#PerformWalRecovery](../raw/postgres-19/src/backend/access/transam/xlogrecovery.c#L1615-L1621), [xlogrecovery.c#ApplyWalRecord](../raw/postgres-19/src/backend/access/transam/xlogrecovery.c#L1975), [transam/README](../raw/postgres-19/src/backend/access/transam/README#L420-L422)).

Related: [Checkpoint](#checkpoint), [Full-page image](#full-page-image), [LSN](#lsn), [WAL](#wal)

### Critical section

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A stretch of backend code in which any error must crash the server rather than be handled normally. `START_CRIT_SECTION()` and `END_CRIT_SECTION()` just raise and lower the counter `CritSectionCount` ([miscadmin.h#START_CRIT_SECTION](../raw/postgres-17/src/include/miscadmin.h#L150-L156)). While that counter is non-zero, `errstart()` promotes any `ERROR` to `PANIC` ([elog.c#errstart](../raw/postgres-17/src/backend/utils/error/elog.c#L356-L364)). WAL-logged page changes are wrapped in one. Shared buffers then hold changes that are not yet logged, and an ordinary error must not let those changes reach disk ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L442-L446)). It is not a lock. The macros only change a counter in the current backend, so other processes are not blocked ([miscadmin.h#START_CRIT_SECTION](../raw/postgres-17/src/include/miscadmin.h#L150-L156)).

**Version notes:**
- PostgreSQL 12: Holds. `START_CRIT_SECTION()` and `END_CRIT_SECTION()` only change `CritSectionCount`, and `errstart()` turns `ERROR` into `PANIC` while it is non-zero ([miscadmin.h#START_CRIT_SECTION](../raw/postgres-12/src/include/miscadmin.h#L132-L138), [elog.c#errstart](../raw/postgres-12/src/backend/utils/error/elog.c#L246-L249)). WAL-logged page changes use one ([transam/README](../raw/postgres-12/src/backend/access/transam/README#L433-L437)).
- PostgreSQL 14: Holds. `START_CRIT_SECTION()` increments `CritSectionCount`, `errstart()` promotes `ERROR` to `PANIC` while it is non-zero, and WAL-logged page changes are wrapped in one ([miscadmin.h#START_CRIT_SECTION](../raw/postgres-14/src/include/miscadmin.h#L148-L154), [elog.c#errstart](../raw/postgres-14/src/backend/utils/error/elog.c#L355-L358), [transam/README](../raw/postgres-14/src/backend/access/transam/README#L441-L446)).
- PostgreSQL 18: Holds ([miscadmin.h#START_CRIT_SECTION](../raw/postgres-18/src/include/miscadmin.h#L150-L156), [elog.c#errstart](../raw/postgres-18/src/backend/utils/error/elog.c#L353-L361), [transam/README](../raw/postgres-18/src/backend/access/transam/README#L442-L446)).
- PostgreSQL 19: Holds. `START_CRIT_SECTION()` and `END_CRIT_SECTION()` still only change `CritSectionCount`, `errstart()` still promotes `ERROR` to `PANIC` inside one, and the README still wraps WAL-logged page changes in one ([miscadmin.h#START_CRIT_SECTION](../raw/postgres-19/src/include/miscadmin.h#L152-L158), [elog.c#errstart](../raw/postgres-19/src/backend/utils/error/elog.c#L366-L373), [transam/README](../raw/postgres-19/src/backend/access/transam/README#L442-L446)).

Related: [WAL](#wal), [ereport](#ereport)

### Cumulative statistics

**Aliases:** cumulative statistics system, pgstat, `pg_stat_*` views. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The cumulative statistics system counts server activity: table and index accesses, row counts, and vacuum and analyze runs ([monitoring.sgml#monitoring-stats](../raw/postgres-17/doc/src/sgml/monitoring.sgml#L137-L145)). In `pgstat.c`, each process accumulates counters locally as pending entries and later flushes them to shared memory through `pgstat_report_stat()`. The startup process loads the stats from disk, and the checkpointer writes them at shutdown ([pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L1-L16), [pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L44-L50)). The `track_counts` GUC switches collection and has context `PGC_SUSET`, so a superuser can change it per session ([guc_tables.c#track_counts](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1412-L1419)). Do not confuse these activity counters with the planner's column statistics. The live view of what processes are doing right now is also a separate facility ([monitoring.sgml#monitoring-stats](../raw/postgres-17/doc/src/sgml/monitoring.sgml#L147-L152)).

**Version notes:**
- PostgreSQL 12: Differs. 12 has a separate statistics collector process, not a shared-memory system. Backends send counters over a UDP socket with `pgstat_report_stat()`, and the collector writes files under `stats_temp_directory` ([pgstat.c header](../raw/postgres-12/src/backend/postmaster/pgstat.c#L1-L16), [pgstat.c:420-424](../raw/postgres-12/src/backend/postmaster/pgstat.c#L420-L424), [pgstat.c#pgstat_report_stat](../raw/postgres-12/src/backend/postmaster/pgstat.c#L803-L813), [guc.c#stats_temp_directory](../raw/postgres-12/src/backend/utils/misc/guc.c#L4076-L4079)). The docs call it "the statistics collector" and keep the live activity view separate ([monitoring.sgml#monitoring-stats](../raw/postgres-12/doc/src/sgml/monitoring.sgml#L132-L155)). `track_counts` is `PGC_SUSET` (superuser, session scope) ([guc.c#track_counts](../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1400)). `stats_temp_directory` is `PGC_SIGHUP` (reload).
- PostgreSQL 14: Differs. 14 has the older statistics collector: a separate "stats collector" process that receives UDP messages from backends and writes stats files; the shared-memory cumulative statistics system of 15+ does not exist ([pgstat.c header](../raw/postgres-14/src/backend/postmaster/pgstat.c#L1-L17), [pgstat.c:404-406](../raw/postgres-14/src/backend/postmaster/pgstat.c#L404-L406), [monitoring.sgml#monitoring-stats](../raw/postgres-14/doc/src/sgml/monitoring.sgml#L138-L154)). `pgstat.c` is in `src/backend/postmaster/`, and `pgstat_report_stat()` sends table stats as messages ([pgstat.c#pgstat_report_stat](../raw/postgres-14/src/backend/postmaster/pgstat.c#L844)). `track_counts` is `PGC_SUSET` (superuser session scope) ([guc.c#track_counts](../raw/postgres-14/src/backend/utils/misc/guc.c#L1527)).
- PostgreSQL 18: Holds. Pending local counters are still flushed through `pgstat_report_stat()`, and `track_counts` is still `PGC_SUSET` (superuser, session scope) ([pgstat.c header](../raw/postgres-18/src/backend/utils/activity/pgstat.c#L1-L17), [guc_tables.c#track_counts](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1495-L1502)). New in 18: extensions can register custom statistics kinds at startup with `pgstat_register_kind()` ([pgstat.c header](../raw/postgres-18/src/backend/utils/activity/pgstat.c#L53-L62), [pgstat.c#pgstat_register_kind](../raw/postgres-18/src/backend/utils/activity/pgstat.c#L1465)).
- PostgreSQL 19: Holds. Processes still keep pending entries and flush them with `pgstat_report_stat()`, the startup process loads the stats and the checkpointer writes them at shutdown, and `track_counts` is still `PGC_SUSET` (session scope, superuser only) ([monitoring.sgml#monitoring-stats](../raw/postgres-19/doc/src/sgml/monitoring.sgml#L137-L152), [pgstat.c header](../raw/postgres-19/src/backend/utils/activity/pgstat.c#L1-L16), [pgstat.c header](../raw/postgres-19/src/backend/utils/activity/pgstat.c#L44-L51), [guc_parameters.dat#track_counts](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3211-L3215)). The v19 header also says external modules can define custom statistics kinds ([pgstat.c header](../raw/postgres-19/src/backend/utils/activity/pgstat.c#L53-L55)).

Related: [Statistics](#statistics), [Wait event](#wait-event), [GUC context](#guc-context)

### Cumulative statistics kind

**Aliases:** `PgStat_Kind`, `PgStat_KindInfo`, stats kind, custom cumulative statistics, `pgstat_register_kind`, fixed-numbered and variable-numbered stats. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A statistics kind is one category of cumulative statistics, such as per-table counters or the single set of checkpointer counters. PostgreSQL 17 lists its kinds in the `PgStat_Kind` enum and splits them into variable-numbered kinds, with one entry per object (database, relation, function, replication slot, subscription), and fixed-numbered kinds, with one global entry (archiver, bgwriter, checkpointer, I/O, SLRU, WAL) ([pgstat.h#PgStat_Kind](../raw/postgres-17/src/include/pgstat.h#L33-L54)). Each kind has a `PgStat_KindInfo` descriptor. Its `fixed_amount` flag records which group the kind is in, and the table `pgstat_kind_infos[]` holds one descriptor per kind ([pgstat_internal.h#PgStat_KindInfo](../raw/postgres-17/src/include/utils/pgstat_internal.h#L201-L219), [pgstat.c#pgstat_kind_infos](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L258)). In 17 the list is a closed C enum, with no registration call for extensions ([pgstat.h#PgStat_Kind](../raw/postgres-17/src/include/pgstat.h#L33-L54)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. Statistics flow through a separate statistics collector process in `postmaster/pgstat.c`, not through a kind registry ([pgstat.c header](../raw/postgres-12/src/backend/postmaster/pgstat.c#L2-L4)).
- PostgreSQL 14: Not present in PostgreSQL 14. It still uses the statistics collector process ([pgstat.c header](../raw/postgres-14/src/backend/postmaster/pgstat.c#L1-L6)).
- PostgreSQL 18: `PgStat_Kind` becomes a `uint32` ID. Built-in kinds use IDs 1 to 12, with a new per-backend kind, and IDs 24 to 32 are reserved for custom kinds ([pgstat_kind.h](../raw/postgres-18/src/include/utils/pgstat_kind.h#L16-L59)). An extension registers a custom kind with `pgstat_register_kind()`, which must run while `shared_preload_libraries` loads ([pgstat.c#pgstat_register_kind](../raw/postgres-18/src/backend/utils/activity/pgstat.c#L1455-L1483)). `shared_preload_libraries` has context `postmaster`, so adding such an extension needs a restart ([guc_tables.c#shared_preload_libraries](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L4487-L4488)).
- PostgreSQL 19: As in 18, with a new fixed-numbered lock kind, so built-in IDs run 1 to 13 ([pgstat_kind.h](../raw/postgres-19/src/include/utils/pgstat_kind.h#L26-L52), [pgstat.c#pgstat_register_kind](../raw/postgres-19/src/backend/utils/activity/pgstat.c#L1508-L1522)). `shared_preload_libraries` is still `postmaster` context (restart) ([guc_parameters.dat#shared_preload_libraries](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2746)).

Related: [Cumulative statistics](#cumulative-statistics), [Shared-memory statistics](#shared-memory-statistics), [shared_preload_libraries](#shared_preload_libraries), [Extension](#extension)

### Custom and generic plan

**Aliases:** plan cache, cached plan, `CachedPlanSource`, `CachedPlan`, `plancache.c`, `choose_custom_plan`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A statement with parameters, such as a prepared statement, can run with two kinds of plan. A custom plan is built for the actual parameter values of one execution. A generic plan is built once without those values and reused. The plan cache in `plancache.c` makes this choice and throws cached plans away when the objects they depend on change ([plancache.c header](../raw/postgres-17/src/backend/utils/cache/plancache.c#L6-L30)). Each cached statement is a `CachedPlanSource`, which holds the generic plan and running cost totals for the custom plans ([plancache.h#CachedPlanSource](../raw/postgres-17/src/include/utils/plancache.h#L121-L133)). `choose_custom_plan()` builds custom plans for the first five executions. After that it switches to the generic plan if the generic plan's cost is below the average custom-plan cost, which includes a planning charge ([plancache.c#choose_custom_plan](../raw/postgres-17/src/backend/utils/cache/plancache.c#L1054-L1101)). The `plan_cache_mode` setting can force either kind. Its context is `user`, so it applies per session or transaction ([guc_tables.c#plan_cache_mode](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5111-L5122)).

**Version notes:**
- PostgreSQL 12: Holds, including the five-custom-plan rule and `plan_cache_mode` with context `user` ([plancache.c#choose_custom_plan](../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016-L1063), [guc.c#plan_cache_mode](../raw/postgres-12/src/backend/utils/misc/guc.c#L4504)).
- PostgreSQL 14: Holds ([plancache.c#choose_custom_plan](../raw/postgres-14/src/backend/utils/cache/plancache.c#L1031-L1078), [guc.c#plan_cache_mode](../raw/postgres-14/src/backend/utils/misc/guc.c#L4949)).
- PostgreSQL 18: Holds ([plancache.c#choose_custom_plan](../raw/postgres-18/src/backend/utils/cache/plancache.c#L1166-L1213), [guc_tables.c#plan_cache_mode](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5385)).
- PostgreSQL 19: Holds ([plancache.c#choose_custom_plan](../raw/postgres-19/src/backend/utils/cache/plancache.c#L1184-L1231), [guc_parameters.dat#plan_cache_mode](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2363)).

Related: [Plan cache mode](#plan-cache-mode), [Prepared statement](#prepared-statement), [Planner](#planner), [Invalidation message](#invalidation-message), [SPI](#spi)

### Data checksums

**Aliases:** page checksum, `pd_checksum`, `data_checksums`, `initdb --data-checksums`, `pg_checksums`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Data checksums are a 16-bit value, stored in each data [page](#page) header, that lets PostgreSQL detect a page corrupted on disk. The field is `pd_checksum`, and it is left unset when checksums are off ([bufpage.h:127-134](../raw/postgres-17/src/include/storage/bufpage.h#L127-L134)). When checksums are on, `PageIsVerifiedExtended()` recomputes the checksum of every page read in and compares it with the stored value ([bufpage.c#PageIsVerifiedExtended](../raw/postgres-17/src/backend/storage/page/bufpage.c#L88-L109)). The choice is made per cluster. In PostgreSQL 17, `initdb -k` turns checksums on and the default is off. The offline `pg_checksums` tool can enable, disable or check them on a stopped cluster ([initdb.c:167](../raw/postgres-17/src/bin/initdb/initdb.c#L167), [initdb.c:2511](../raw/postgres-17/src/bin/initdb/initdb.c#L2511), [pg_checksums.c header](../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L1-L5)). The read-only `data_checksums` setting reports the state; its context is `internal`, so no configuration change can set it ([guc_tables.c:1883-1890](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1883-L1890)). Checksums cost WAL. A hint-bit change must be WAL-logged, with a [full-page image](#full-page-image) the first time the page changes after a [checkpoint](#checkpoint), because a torn write would otherwise break the checksum ([xlog.h#XLogHintBitIsNeeded](../raw/postgres-17/src/include/access/xlog.h#L110-L118), [xloginsert.c#XLogSaveBufferForHint](../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1055-L1065)). `wal_log_hints` forces the same logging without checksums. Its context is `postmaster`, so changing it needs a restart ([guc_tables.c:1171](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1171)).

**Version notes:**
- PostgreSQL 12: Holds, with checksums off by default ([initdb.c:144](../raw/postgres-12/src/bin/initdb/initdb.c#L144), [xlog.h:192](../raw/postgres-12/src/include/access/xlog.h#L192)). The verify function is `PageIsVerified()`, and `data_checksums` is defined in `guc.c` ([bufpage.c:82](../raw/postgres-12/src/backend/storage/page/bufpage.c#L82), [guc.c:1827](../raw/postgres-12/src/backend/utils/misc/guc.c#L1827)). `pg_checksums` already works offline ([pg_checksums.c header](../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L1-L5)).
- PostgreSQL 14: Holds, off by default ([initdb.c:146](../raw/postgres-14/src/bin/initdb/initdb.c#L146), [bufpage.c:88](../raw/postgres-14/src/backend/storage/page/bufpage.c#L88), [guc.c:1991](../raw/postgres-14/src/backend/utils/misc/guc.c#L1991)).
- PostgreSQL 18: `initdb` turns checksums on by default, and `--no-data-checksums` turns them off ([initdb.c:167](../raw/postgres-18/src/bin/initdb/initdb.c#L167), [initdb.c:2542](../raw/postgres-18/src/bin/initdb/initdb.c#L2542)). Verification is `PageIsVerified()` again ([bufpage.c:94](../raw/postgres-18/src/backend/storage/page/bufpage.c#L94)).
- PostgreSQL 19: Checksums can be turned on or off while the cluster runs. `pg_enable_data_checksums()` starts a background worker that rewrites every page with a checksum, passing through an "inprogress-on" state ([datachecksum_state.c header](../raw/postgres-19/src/backend/postmaster/datachecksum_state.c#L1-L40), [func-admin.sgml:3159-3186](../raw/postgres-19/doc/src/sgml/func/func-admin.sgml#L3159-L3186)). `data_checksums` becomes an enum, still with context `internal` ([guc_parameters.dat:581-588](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L581-L588)). Hint-bit logging now keys off `DataChecksumsNeedWrite()`, which is also true in the transition states ([xlog.h:136](../raw/postgres-19/src/include/access/xlog.h#L136), [xlog.c#DataChecksumsNeedWrite](../raw/postgres-19/src/backend/access/transam/xlog.c#L4682-L4688)). The initdb default stays on ([initdb.c:167](../raw/postgres-19/src/bin/initdb/initdb.c#L167)).

Related: [Page](#page), [Full-page image](#full-page-image), [WAL](#wal), [GUC context](#guc-context)

### Datum

**Aliases:** `Datum`, `DatumGetX`, `XGetDatum`, `NullableDatum`, pass-by-value, pass-by-reference, `typbyval`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A `Datum` is the generic C type that backend code uses to hold one SQL value. It holds the value itself for a small pass-by-value type, or a pointer to the value for a pass-by-reference type. It is exactly as wide as a pointer ([postgres.h#Datum](../raw/postgres-17/src/include/postgres.h#L54-L64)). Code converts with paired helpers such as `DatumGetInt32()` and `Int32GetDatum()` ([postgres.h#DatumGetInt32](../raw/postgres-17/src/include/postgres.h#L198-L216)). Whether a type is passed by value is recorded in `pg_type.typbyval`. Variable-length types are always passed by reference ([pg_type.h#typbyval](../raw/postgres-17/src/include/catalog/pg_type.h#L58-L66)). Function-call interfaces, tuple forming and deforming, and executor slots all pass values as `Datum` plus a separate null flag. `NullableDatum` bundles the two ([postgres.h#NullableDatum](../raw/postgres-17/src/include/postgres.h#L66-L79)).

**Version notes:**
- PostgreSQL 12: Holds; `Datum` is `uintptr_t`, and the conversions are macros rather than inline functions ([postgres.h#Datum](../raw/postgres-12/src/include/postgres.h#L357-L367), [postgres.h#DatumGetInt32](../raw/postgres-12/src/include/postgres.h#L472-L479)).
- PostgreSQL 14: Holds as in 12 ([postgres.h#Datum](../raw/postgres-14/src/include/postgres.h#L401-L411), [postgres.h#DatumGetInt32](../raw/postgres-14/src/include/postgres.h#L516-L523)).
- PostgreSQL 18: Holds as in 17 ([postgres.h#Datum](../raw/postgres-18/src/include/postgres.h#L59-L69)).
- PostgreSQL 19: `Datum` is always 8 bytes (`uint64_t`), even on 32-bit platforms ([postgres.h#Datum](../raw/postgres-19/src/include/postgres.h#L58-L76)). Built-in 8-byte types such as `float8`, `int8` and `timestamp` are therefore always passed by value ([pg_config_manual.h#USE_FLOAT8_BYVAL](../raw/postgres-19/src/include/pg_config_manual.h#L85-L91)).

Related: [fmgr](#fmgr), [Tuple](#tuple), [varlena](#varlena), [TOAST](#toast)

### Dead tuple

**Aliases:** dead row, dead row version, `n_dead_tup`, `HEAPTUPLE_DEAD`, `HEAPTUPLE_RECENTLY_DEAD`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A dead tuple is an old row version that an `UPDATE` or `DELETE` replaced and that no transaction still needs. It keeps using space until pruning or VACUUM removes it. VACUUM uses `HeapTupleSatisfiesVacuum()` to sort row versions. `HEAPTUPLE_DEAD` means the version is dead and removable. `HEAPTUPLE_RECENTLY_DEAD` means it is dead but some transaction may still see it, so it must stay for now ([heapam.h#HTSV_Result](../raw/postgres-17/src/include/access/heapam.h#L122-L130)). VACUUM turns removed versions into `LP_DEAD` line pointers and collects their TIDs in `dead_items`, so that index entries can be removed before the pointers are freed ([vacuumlazy.c#LVRelState](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L176-L185)). The `n_dead_tup` column of `pg_stat_all_tables` is an estimate. Each committed update or delete adds one, and VACUUM overwrites the count with what it found ([system_views.sql:688](../raw/postgres-17/src/backend/catalog/system_views.sql#L688), [pgstat_relation.c#AtEOXact_PgStat_Relations](../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L568-L570), [pgstat_relation.c#pgstat_report_vacuum](../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L232-L233)).

**Version notes:**
- PostgreSQL 12: Holds ([heapam.h#HTSV_Result](../raw/postgres-12/src/include/access/heapam.h#L85-L93), [system_views.sql:567](../raw/postgres-12/src/backend/catalog/system_views.sql#L567)). The dead-tuple counter lives in the statistics collector code ([pgstat.c:2150-2152](../raw/postgres-12/src/backend/postmaster/pgstat.c#L2150-L2152)).
- PostgreSQL 14: Holds as in 12 ([heapam.h#HTSV_Result](../raw/postgres-14/src/include/access/heapam.h#L92-L100), [system_views.sql:646](../raw/postgres-14/src/backend/catalog/system_views.sql#L646), [pgstat.c:2422-2424](../raw/postgres-14/src/backend/postmaster/pgstat.c#L2422-L2424)).
- PostgreSQL 18: Holds ([heapam.h#HTSV_Result](../raw/postgres-18/src/include/access/heapam.h#L121-L129), [system_views.sql:697](../raw/postgres-18/src/backend/catalog/system_views.sql#L697), [pgstat_relation.c:582-584](../raw/postgres-18/src/backend/utils/activity/pgstat_relation.c#L582-L584)).
- PostgreSQL 19: Holds ([heapam.h#HTSV_Result](../raw/postgres-19/src/include/access/heapam.h#L135-L143), [system_views.sql:735](../raw/postgres-19/src/backend/catalog/system_views.sql#L735), [pgstat_relation.c:583-585](../raw/postgres-19/src/backend/utils/activity/pgstat_relation.c#L583-L585)).

Related: [MVCC](#mvcc), [xmin horizon](#xmin-horizon), [Pruning](#pruning), [VACUUM](#vacuum), [Line pointer](#line-pointer), [Bloat](#bloat)

### Deadlock

**Aliases:** deadlock detection, `deadlock_timeout`, `DeadLockCheck`, "deadlock detected". **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A deadlock is a cycle of transactions, each waiting for a [heavyweight lock](#heavyweight-lock) another one holds, so none can proceed. PostgreSQL lets transactions request locks in any order, so deadlocks are possible, and it detects them lazily. A backend that must wait goes to sleep and arms a timer of `deadlock_timeout` milliseconds, 1000 by default. Only if the timer fires before the lock is granted does it run the detector ([lmgr/README#deadlock-detection](../raw/postgres-17/src/backend/storage/lmgr/README#L338-L359), [proc.c:1295-1307](../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1295-L1307)). `DeadLockCheck()` looks for a cycle through the waiting process. It first tries to fix the cycle by reordering wait queues. If that is impossible, it returns `DS_HARD_DEADLOCK` and the caller aborts that transaction ([deadlock.c#DeadLockCheck](../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L202-L217)). The victim gets `ERROR: deadlock detected`, and the event is counted in the cumulative statistics ([deadlock.c#DeadLockReport](../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L1127-L1135)). `deadlock_timeout` has context `superuser`, so a superuser can change it per session ([guc_tables.c:2149-2156](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2149-L2156)). [LWLocks](#lwlock) have no deadlock detection ([lmgr/README:19-24](../raw/postgres-17/src/backend/storage/lmgr/README#L19-L24)).

**Version notes:**
- PostgreSQL 12: Holds; `deadlock_timeout` is defined in `guc.c` with context `superuser` ([deadlock.c#DeadLockCheck](../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L217), [guc.c:2064](../raw/postgres-12/src/backend/utils/misc/guc.c#L2064), [lmgr/README:349](../raw/postgres-12/src/backend/storage/lmgr/README#L349)).
- PostgreSQL 14: Holds ([deadlock.c#DeadLockCheck](../raw/postgres-14/src/backend/storage/lmgr/deadlock.c#L217), [guc.c:2237](../raw/postgres-14/src/backend/utils/misc/guc.c#L2237)).
- PostgreSQL 18: Holds ([deadlock.c#DeadLockCheck](../raw/postgres-18/src/backend/storage/lmgr/deadlock.c#L220), [guc_tables.c:2266](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2266)).
- PostgreSQL 19: Holds; the setting is declared in `guc_parameters.dat`, still with context `superuser` ([deadlock.c#DeadLockCheck](../raw/postgres-19/src/backend/storage/lmgr/deadlock.c#L220), [guc_parameters.dat:627](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L627)).

Related: [Heavyweight lock](#heavyweight-lock), [Lock mode](#lock-mode), [Lock queue](#lock-queue), [LWLock](#lwlock)

### Declarative partitioning

**Aliases:** partitioned table, partition, `PARTITION BY`, partition key, default partition, `RELKIND_PARTITIONED_TABLE`, `pg_partitioned_table`, tuple routing. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Declarative partitioning splits one logical table into several ordinary tables, called partitions. `CREATE TABLE ... PARTITION BY` names the method (range, list or hash) and the partition key ([ddl.sgml#ddl-partitioning-declarative](../raw/postgres-17/doc/src/sgml/ddl.sgml#L3979-L4000), [parsenodes.h#PartitionStrategy](../raw/postgres-17/src/include/nodes/parsenodes.h#L870-L875)). The partitioned table itself has `relkind` `'p'` and no storage. Only its partitions hold rows ([pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-17/src/include/catalog/pg_class.h#L172), [ddl.sgml#ddl-partitioning-declarative](../raw/postgres-17/doc/src/sgml/ddl.sgml#L3991-L4000)). `pg_partitioned_table` stores the key: the strategy, key columns or expressions, operator classes, collations, and the default partition's OID ([pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-17/src/include/catalog/pg_partitioned_table.h#L30-L57)). On insert, `ExecFindPartition()` routes each row to its leaf partition and raises an error when none matches ([execPartition.c#ExecFindPartition](../raw/postgres-17/src/backend/executor/execPartition.c#L243-L262)).

**Version notes:**
- PostgreSQL 12: Holds, with the same three strategies as `#define` constants ([parsenodes.h:798-800](../raw/postgres-12/src/include/nodes/parsenodes.h#L798-L800), [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-12/src/include/catalog/pg_class.h#L162), [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-12/src/include/catalog/pg_partitioned_table.h#L30-L54), [ddl.sgml#ddl-partitioning-declarative](../raw/postgres-12/doc/src/sgml/ddl.sgml#L3608)).
- PostgreSQL 14: Holds as in 12 ([parsenodes.h:814-816](../raw/postgres-14/src/include/nodes/parsenodes.h#L814-L816), [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-14/src/include/catalog/pg_class.h#L172), [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-14/src/include/catalog/pg_partitioned_table.h#L30-L58), [execPartition.c#ExecFindPartition](../raw/postgres-14/src/backend/executor/execPartition.c#L257)).
- PostgreSQL 18: Holds ([parsenodes.h#PartitionStrategy](../raw/postgres-18/src/include/nodes/parsenodes.h#L896-L901), [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-18/src/include/catalog/pg_class.h#L175), [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-18/src/include/catalog/pg_partitioned_table.h#L30-L58), [execPartition.c#ExecFindPartition](../raw/postgres-18/src/backend/executor/execPartition.c#L265)).
- PostgreSQL 19: Holds ([parsenodes.h#PartitionStrategy](../raw/postgres-19/src/include/nodes/parsenodes.h#L916-L921), [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-19/src/include/catalog/pg_class.h#L179), [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-19/src/include/catalog/pg_partitioned_table.h#L32-L60), [execPartition.c#ExecFindPartition](../raw/postgres-19/src/backend/executor/execPartition.c#L268)).

Related: [Partition bound](#partition-bound), [Partitioned index](#partitioned-index), [Partition pruning](#partition-pruning), [Partitionwise join](#partitionwise-join), [Inheritance](#inheritance), [pg_class](#pg_class)

### Deduplication

**Aliases:** B-tree deduplication, `deduplicate_items`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Deduplication merges several B-tree leaf entries that have the same key into one posting-list tuple that holds many row addresses. This saves space and delays page splits ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L904-L920)). It runs lazily, only when a leaf page is about to split. It is the last step in `_bt_delete_or_dedup_one_page()`, and it runs only when the `deduplicate_items` storage parameter is on (the default) and the index's `allequalimage` flag is true ([nbtinsert.c:2778-2781](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2778-L2781), [nbtree.h#BTGetDeduplicateItems](../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150)). `_bt_dedup_pass()` in `nbtdedup.c` does the merging ([nbtdedup.c#_bt_dedup_pass](../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L36-L58)). Do not confuse it with bottom-up deletion, which removes entries instead of merging them.

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The v12 `nbtree` build has no `nbtdedup.o` ([nbtree/Makefile:15-16](../raw/postgres-12/src/backend/access/nbtree/Makefile#L15-L16)), and the B-tree reloptions offer `fillfactor` and `vacuum_cleanup_index_scale_factor` but no `deduplicate_items` ([reloptions.c:177-186](../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186)). v12 leaf pages store one tuple per heap TID, and version-4 indexes use the heap TID as a tiebreaker key column ([nbtree.h:236-238](../raw/postgres-12/src/include/access/nbtree.h#L236-L238)).
- PostgreSQL 14: Holds. The README describes merging duplicates into posting lists to delay splits, and it runs last in `_bt_delete_or_dedup_one_page()` only when `deduplicate_items` is on (default true) and `allequalimage` is set ([nbtree/README](../raw/postgres-14/src/backend/access/nbtree/README#L877-L884), [nbtinsert.c:2767-2770](../raw/postgres-14/src/backend/access/nbtree/nbtinsert.c#L2767-L2770), [nbtree.h#BTGetDeduplicateItems](../raw/postgres-14/src/include/access/nbtree.h#L1103-L1107), [nbtdedup.c#_bt_dedup_pass](../raw/postgres-14/src/backend/access/nbtree/nbtdedup.c#L57)).
- PostgreSQL 18: Holds ([nbtree/README](../raw/postgres-18/src/backend/access/nbtree/README#L904-L920), [nbtinsert.c:2778-2781](../raw/postgres-18/src/backend/access/nbtree/nbtinsert.c#L2778-L2781), [nbtree.h#BTGetDeduplicateItems](../raw/postgres-18/src/include/access/nbtree.h#L1166-L1170), [nbtdedup.c#_bt_dedup_pass](../raw/postgres-18/src/backend/access/nbtree/nbtdedup.c#L36-L58)).
- PostgreSQL 19: Holds. Deduplication is still lazy, still the last step before a leaf split, still gated by `deduplicate_items` and `allequalimage`, and still done by `_bt_dedup_pass()` ([nbtree/README](../raw/postgres-19/src/backend/access/nbtree/README#L903-L919), [nbtinsert.c:2825-2828](../raw/postgres-19/src/backend/access/nbtree/nbtinsert.c#L2825-L2828), [nbtree.h#BTGetDeduplicateItems](../raw/postgres-19/src/include/access/nbtree.h#L1135-L1139), [nbtdedup.c#_bt_dedup_pass](../raw/postgres-19/src/backend/access/nbtree/nbtdedup.c#L36-L60)).

Related: [Posting list](#posting-list), [allequalimage](#allequalimage), [Bottom-up index deletion](#bottom-up-index-deletion), [Page split](#page-split)

### Dirty buffer

**Aliases:** `BM_DIRTY`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A dirty buffer is a shared buffer whose page was changed in memory but has not yet been written to its data file. `MarkBufferDirty()` sets this state; the caller must hold a pin and an exclusive content lock, and the actual write happens later ([bufmgr.c#MarkBufferDirty](../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2540-L2551)). The flag is `BM_DIRTY` in the buffer state word ([buf_internals.h:61](../raw/postgres-17/src/include/storage/buf_internals.h#L61)). A dirty buffer must be written before clock sweep can recycle it ([buffer/README#clock-sweep](../raw/postgres-17/src/backend/storage/buffer/README#L198-L199)). The background writer looks ahead of the clock hand for dirty, unpinned, zero-usage buffers to write early ([buffer/README#bgwriter](../raw/postgres-17/src/backend/storage/buffer/README#L252-L257)).

**Version notes:**
- PostgreSQL 12: Holds. `MarkBufferDirty()` needs a pin and an exclusive content lock and defers the write, and `BM_DIRTY` is the flag ([bufmgr.c#MarkBufferDirty](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1446-L1457), [buf_internals.h:59](../raw/postgres-12/src/include/storage/buf_internals.h#L59)). A dirty victim must be written first, and the bgwriter scans ahead of `nextVictimBuffer` for dirty, unpinned, zero-usage buffers ([buffer/README#clock-sweep](../raw/postgres-12/src/backend/storage/buffer/README#L202-L204), [buffer/README#bgwriter](../raw/postgres-12/src/backend/storage/buffer/README#L255-L260)).
- PostgreSQL 14: Holds. `MarkBufferDirty()` requires a pin and exclusive lock and defers the write, the flag is `BM_DIRTY`, and the background writer scans ahead of the clock hand for dirty, unpinned, zero-usage buffers ([bufmgr.c#MarkBufferDirty](../raw/postgres-14/src/backend/storage/buffer/bufmgr.c#L1573-L1582), [buf_internals.h:59](../raw/postgres-14/src/include/storage/buf_internals.h#L59), [buffer/README#bgwriter](../raw/postgres-14/src/backend/storage/buffer/README#L252-L257)).
- PostgreSQL 18: Holds ([bufmgr.c#MarkBufferDirty](../raw/postgres-18/src/backend/storage/buffer/bufmgr.c#L2960-L2971), [buf_internals.h:69](../raw/postgres-18/src/include/storage/buf_internals.h#L69), [buffer/README#bgwriter](../raw/postgres-18/src/backend/storage/buffer/README#L255-L260)).
- PostgreSQL 19: Holds. `MarkBufferDirty()` still needs a pin and an exclusive content lock and defers the write, `BM_DIRTY` is still a state flag, and the bgwriter still scans ahead of the clock hand ([bufmgr.c#MarkBufferDirty](../raw/postgres-19/src/backend/storage/buffer/bufmgr.c#L3160-L3170), [buf_internals.h:107-108](../raw/postgres-19/src/include/storage/buf_internals.h#L107-L108), [buffer/README#clock-sweep](../raw/postgres-19/src/backend/storage/buffer/README#L200-L203), [buffer/README#bgwriter](../raw/postgres-19/src/backend/storage/buffer/README#L253-L258)). In v19, `BM_DIRTY` is a bit in the 64-bit state word ([buf_internals.h:100-108](../raw/postgres-19/src/include/storage/buf_internals.h#L100-L108)). A new content-lock mode, `BUFFER_LOCK_SHARE_EXCLUSIVE`, lets `MarkBufferDirtyHint()` mark hint-bit changes dirty without an exclusive lock ([bufmgr.h:217](../raw/postgres-19/src/include/storage/bufmgr.h#L217), [bufmgr.c#MarkBufferDirtyHint](../raw/postgres-19/src/backend/storage/buffer/bufmgr.c#L5828-L5845)).

Related: [Background writer](#background-writer), [Checkpoint](#checkpoint), [Clock sweep](#clock-sweep), [WAL](#wal)

### Dynamic shared memory

**Aliases:** DSM, DSA, `dsm_segment`, `dsa_area`, `dshash`, DSM registry, `dynamic_shared_memory_type`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Dynamic shared memory is shared memory that the server creates while running, not once at startup. It comes in three layers. `dsm.c` manages whole segments and removes each one when its last mapping goes away ([dsm.c header](../raw/postgres-17/src/backend/storage/ipc/dsm.c#L3-L15)). `dsa.c` builds a shared heap, a DSA area, on top of segments, with `dsa_allocate()` and `dsa_free()` returning pseudo-pointers that any backend can convert ([dsa.c header](../raw/postgres-17/src/backend/utils/mmgr/dsa.c#L3-L16)). `dshash.c` provides concurrent hash tables inside a DSA area ([dshash.c header](../raw/postgres-17/src/backend/lib/dshash.c#L3-L9)). Parallel query puts its shared state in a DSM segment, and cumulative statistics keep their entries in a DSA-backed `dshash` table ([parallel.c:323](../raw/postgres-17/src/backend/access/transam/parallel.c#L323), [pgstat_shmem.c#StatsShmemInit](../raw/postgres-17/src/backend/utils/activity/pgstat_shmem.c#L169-L184)). The DSM registry lets an extension get a named segment without reserving memory at startup ([dsm_registry.c header](../raw/postgres-17/src/backend/storage/ipc/dsm_registry.c#L3-L16)). `dynamic_shared_memory_type` and `min_dynamic_shared_memory` have context `postmaster`, so changing either needs a restart ([guc_tables.c#min_dynamic_shared_memory](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2247-L2255), [guc_tables.c#dynamic_shared_memory_type](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4997-L5004)).

**Version notes:**
- PostgreSQL 12: Segments, DSA and `dshash` exist, and parallel query uses a segment ([dsm.c header](../raw/postgres-12/src/backend/storage/ipc/dsm.c#L3-L4), [dsa.c header](../raw/postgres-12/src/backend/utils/mmgr/dsa.c#L3-L4), [parallel.c:294](../raw/postgres-12/src/backend/access/transam/parallel.c#L294)). There is no DSM registry and no `min_dynamic_shared_memory`. `dynamic_shared_memory_type` is `postmaster` context (restart) ([guc.c#dynamic_shared_memory_type](../raw/postgres-12/src/backend/utils/misc/guc.c#L4420-L4422)). Statistics still go through the collector process, not DSA ([pgstat.c header](../raw/postgres-12/src/backend/postmaster/pgstat.c#L2-L4)).
- PostgreSQL 14: As in 12, plus `min_dynamic_shared_memory`, `postmaster` context (restart) ([guc.c#min_dynamic_shared_memory](../raw/postgres-14/src/backend/utils/misc/guc.c#L2324-L2325), [parallel.c:318](../raw/postgres-14/src/backend/access/transam/parallel.c#L318)). No DSM registry. Statistics still use the collector ([pgstat.c header](../raw/postgres-14/src/backend/postmaster/pgstat.c#L1-L6)).
- PostgreSQL 18: Holds as in 17 ([dsm_registry.c#GetNamedDSMSegment](../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L132), [parallel.c:327](../raw/postgres-18/src/backend/access/transam/parallel.c#L327), [guc_tables.c#min_dynamic_shared_memory](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2364), [guc_tables.c#dynamic_shared_memory_type](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5260)).
- PostgreSQL 19: The registry also hands out named DSA areas and named `dshash` tables through `GetNamedDSA()` and `GetNamedDSHash()` ([dsm_registry.c#GetNamedDSA](../raw/postgres-19/src/backend/storage/ipc/dsm_registry.c#L279), [dsm_registry.c#GetNamedDSHash](../raw/postgres-19/src/backend/storage/ipc/dsm_registry.c#L360)). Both GUCs remain `postmaster` context (restart) ([guc_parameters.dat#dynamic_shared_memory_type](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L830), [guc_parameters.dat#min_dynamic_shared_memory](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2209)).

Related: [Parallel query](#parallel-query), [Shared-memory statistics](#shared-memory-statistics), [Extension](#extension), [LWLock](#lwlock)

### effective_cache_size

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`effective_cache_size` tells the planner how much data it may assume is cached, counting both shared buffers and the kernel cache. It is measured in pages, and it allocates no memory [costsize.c:22-25](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L22-L25) [guc_tables.c#effective_cache_size](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3509-L3518). Its context is `user`, so a session or transaction can `SET` it with no reload or restart. Its default is 524288 pages [cost.h:34](../raw/postgres-17/src/include/optimizer/cost.h#L34). Its main consumer is `index_pages_fetched()`. That function gives each table a pro-rated share of the cache and applies the Mackert and Lohman formula to estimate repeated page fetches [costsize.c#index_pages_fetched](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L897-L928).

**Version notes:**
- PostgreSQL 12: Holds. It is a page-count planner hint covering shared buffers and kernel cache, `PGC_USERSET` (session scope), default 524288 pages ([costsize.c:22-25](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L22-L25), [guc.c#effective_cache_size](../raw/postgres-12/src/backend/utils/misc/guc.c#L3108-L3117), [cost.h:32](../raw/postgres-12/src/include/optimizer/cost.h#L32)). `index_pages_fetched()` pro-rates it and uses Mackert and Lohman ([costsize.c#index_pages_fetched](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L791-L825)).
- PostgreSQL 14: Holds. It is a page-unit planner assumption with default 524288 pages and context `PGC_USERSET` (session scope), and `index_pages_fetched()` pro-rates it before applying the Mackert and Lohman formula ([guc.c#effective_cache_size](../raw/postgres-14/src/backend/utils/misc/guc.c#L3424-L3433), [cost.h:32](../raw/postgres-14/src/include/optimizer/cost.h#L32), [costsize.c#index_pages_fetched](../raw/postgres-14/src/backend/optimizer/path/costsize.c#L806-L865)).
- PostgreSQL 18: Holds. Still `user` context (session scope), default 524288 pages, consumed by `index_pages_fetched()` ([guc_tables.c#effective_cache_size](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3716-L3725), [cost.h:34](../raw/postgres-18/src/include/optimizer/cost.h#L34), [costsize.c#index_pages_fetched](../raw/postgres-18/src/backend/optimizer/path/costsize.c#L907-L938)).
- PostgreSQL 19: Holds. It is still a planner-only page count with default 524288 pages, it is still `PGC_USERSET` (session scope), and `index_pages_fetched()` still gives each table a pro-rated share of it ([costsize.c:22-25](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L22-L25), [guc_parameters.dat#effective_cache_size](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L837-L845), [cost.h:34](../raw/postgres-19/src/include/optimizer/cost.h#L34), [costsize.c#index_pages_fetched](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L858-L915)).

Related: [Cost](#cost), [shared_buffers](#shared_buffers), [Planner](#planner)

### effective_io_concurrency

**Aliases:** `maintenance_io_concurrency`, prefetch depth, I/O concurrency. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`effective_io_concurrency` tells PostgreSQL how many read requests the storage can usefully handle at once. PostgreSQL uses it to decide how far ahead to issue reads before it needs the data ([guc_tables.c:3109-3120](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3109-L3120)). `maintenance_io_concurrency` is the same knob for maintenance work ([guc_tables.c:3123-3135](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3123-L3135)). In PostgreSQL 17 the defaults are 1 and 10 on platforms with prefetch support, and 0 elsewhere ([bufmgr.h:156-163](../raw/postgres-17/src/include/storage/bufmgr.h#L156-L163)). A [read stream](#read-stream) takes its maximum number of in-flight I/Os from `effective_io_concurrency`, or from `maintenance_io_concurrency` when the caller passes `READ_STREAM_MAINTENANCE` ([read_stream.c:420-438](../raw/postgres-17/src/backend/storage/aio/read_stream.c#L420-L438)). Index-entry deletion in the heap uses the maintenance value as its prefetch distance ([heapam.c:8548-8558](../raw/postgres-17/src/backend/access/heap/heapam.c#L8548-L8558)). A tablespace can override either value, which `spccache.c` looks up ([spccache.c:219-236](../raw/postgres-17/src/backend/utils/cache/spccache.c#L219-L236)). Both have context `user`, so they can be set per session without a reload ([guc_tables.c:3109-3135](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3109-L3135)).

**Version notes:**
- PostgreSQL 12: Only `effective_io_concurrency` exists, with context `user` and default 1 where prefetch is supported ([guc.c:2759-2772](../raw/postgres-12/src/backend/utils/misc/guc.c#L2759-L2772)). Its main consumer is bitmap heap scan prefetching ([nodeBitmapHeapscan.c:803-810](../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L803-L810)).
- PostgreSQL 14: Both settings exist, context `user`, defaults 1 and 10 where prefetch is supported ([guc.c:3045-3076](../raw/postgres-14/src/backend/utils/misc/guc.c#L3045-L3076)).
- PostgreSQL 18: Both defaults rise to 16 ([bufmgr.h:161-162](../raw/postgres-18/src/include/storage/bufmgr.h#L161-L162)). Context stays `user` ([guc_tables.c:3250-3274](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3250-L3274)).
- PostgreSQL 19: Defaults are 16 and 16, context `user`; both are declared in `guc_parameters.dat` ([bufmgr.h:170-171](../raw/postgres-19/src/include/storage/bufmgr.h#L170-L171), [guc_parameters.dat:847](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L847), [guc_parameters.dat:1937](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1937)).

Related: [io_combine_limit](#io_combine_limit), [Read stream](#read-stream), [Bitmap scan](#bitmap-scan), [GUC context](#guc-context)

### ereport

**Aliases:** `elog`, error level, `ERROR`, `FATAL`, `PANIC`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`ereport` and the shorter `elog` are the macros that server code uses to log a message or raise an error. The first argument is a severity level that runs from `DEBUG5` through `LOG`, `WARNING`, `ERROR`, and `FATAL` up to `PANIC` ([elog.h:25-56](../raw/postgres-17/src/include/utils/elog.h#L25-L56), [elog.h#ereport](../raw/postgres-17/src/include/utils/elog.h#L163-L164), [elog.h#elog](../raw/postgres-17/src/include/utils/elog.h#L239-L240)). The level decides what happens next. `ERROR` long-jumps to the current error handler and aborts the transaction, `FATAL` ends the process, and `PANIC` calls `abort()`, and the postmaster then kills the other backends too ([elog.c#errfinish](../raw/postgres-17/src/backend/utils/error/elog.c#L515-L543), [elog.c:594-604](../raw/postgres-17/src/backend/utils/error/elog.c#L594-L604)). When source code shows an `ereport(ERROR, ...)`, execution never continues on the next line.

**Version notes:**
- PostgreSQL 12: Holds. The levels run `DEBUG5` to `PANIC`, and `ereport`/`elog` are macros ([elog.h:20-53](../raw/postgres-12/src/include/utils/elog.h#L20-L53), [elog.h#ereport](../raw/postgres-12/src/include/utils/elog.h#L141), [elog.h#elog](../raw/postgres-12/src/include/utils/elog.h#L217)). `errfinish()` exits the process for `FATAL` and calls `abort()` for `PANIC` ([elog.c#errfinish](../raw/postgres-12/src/backend/utils/error/elog.c#L513-L552)). The `ereport` macro takes the older two-argument form `ereport(elevel, (rest))` in 12 ([elog.h#ereport](../raw/postgres-12/src/include/utils/elog.h#L141)).
- PostgreSQL 14: Holds. The levels run from `DEBUG5` to `PANIC`, and `errfinish()` long-jumps for `ERROR`, exits for `FATAL`, and calls `abort()` for `PANIC` ([elog.h:20-50](../raw/postgres-14/src/include/utils/elog.h#L20-L50), [elog.h#ereport](../raw/postgres-14/src/include/utils/elog.h#L157), [elog.h#elog](../raw/postgres-14/src/include/utils/elog.h#L232), [elog.c#errfinish](../raw/postgres-14/src/backend/utils/error/elog.c#L513-L689)).
- PostgreSQL 18: Holds ([elog.h:25-56](../raw/postgres-18/src/include/utils/elog.h#L25-L56), [elog.c#errfinish](../raw/postgres-18/src/backend/utils/error/elog.c#L512-L540), [elog.c:591-601](../raw/postgres-18/src/backend/utils/error/elog.c#L591-L601)).
- PostgreSQL 19: Holds. The levels still run from `DEBUG5` to `PANIC`, `ERROR` still long-jumps with `PG_RE_THROW()`, `FATAL` still calls `proc_exit(1)`, and `PANIC` still calls `abort()` ([elog.h:27-58](../raw/postgres-19/src/include/utils/elog.h#L27-L58), [elog.h#ereport](../raw/postgres-19/src/include/utils/elog.h#L166), [elog.h#elog](../raw/postgres-19/src/include/utils/elog.h#L242), [elog.c#errfinish](../raw/postgres-19/src/backend/utils/error/elog.c#L528-L552), [elog.c:608-622](../raw/postgres-19/src/backend/utils/error/elog.c#L608-L622)). v19 adds a `FATAL_CLIENT_ONLY` level (23), which `errfinish()` handles like `FATAL` ([elog.h:57](../raw/postgres-19/src/include/utils/elog.h#L57), [elog.c:577](../raw/postgres-19/src/backend/utils/error/elog.c#L577)).

Related: [Critical section](#critical-section), [Memory context](#memory-context), [Backend](#backend)

### Event trigger

**Aliases:** `pg_event_trigger`, `ddl_command_start`, `ddl_command_end`, `sql_drop`, `table_rewrite`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An event trigger is a database-wide trigger that fires on DDL events rather than on row changes to one table ([event-trigger.sgml:10-16](../raw/postgres-17/doc/src/sgml/event-trigger.sgml#L10-L16)). Its events are `login`, `ddl_command_start`, `ddl_command_end`, `table_rewrite`, and `sql_drop` ([event-trigger.sgml:27-35](../raw/postgres-17/doc/src/sgml/event-trigger.sgml#L27-L35)). Each trigger is a `pg_event_trigger` row naming the event and the function to call ([pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-17/src/include/catalog/pg_event_trigger.h#L29-L42)). `ProcessUtilitySlow` fires the start event before it dispatches a command, and fires the drop and end events after the command runs ([utility.c:1106-1113](../raw/postgres-17/src/backend/tcop/utility.c#L1106-L1113), [utility.c:1943-1946](../raw/postgres-17/src/backend/tcop/utility.c#L1943-L1946)). An event trigger is a SQL-level object. A [hook](#hook), by contrast, is a C function pointer that a loaded library sets.

**Version notes:**
- PostgreSQL 12: Differs. 12 supports only `ddl_command_start`, `ddl_command_end`, `table_rewrite` and `sql_drop`; the `login` event does not exist ([event-trigger.sgml:28-34](../raw/postgres-12/doc/src/sgml/event-trigger.sgml#L28-L34)). The `pg_event_trigger` catalog and the start/drop/end calls in `ProcessUtilitySlow` match 17 ([pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-12/src/include/catalog/pg_event_trigger.h#L26-L42), [utility.c:972-973](../raw/postgres-12/src/backend/tcop/utility.c#L972-L973), [utility.c:1711-1712](../raw/postgres-12/src/backend/tcop/utility.c#L1711-L1712)).
- PostgreSQL 14: Differs. 14 has only four events: `ddl_command_start`, `ddl_command_end`, `table_rewrite` and `sql_drop`; there is no `login` event ([event-trigger.sgml:28-35](../raw/postgres-14/doc/src/sgml/event-trigger.sgml#L28-L35)). The `pg_event_trigger` catalog and the firing points in `ProcessUtilitySlow` match ([pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-14/src/include/catalog/pg_event_trigger.h#L29-L42), [utility.c:1112](../raw/postgres-14/src/backend/tcop/utility.c#L1112), [utility.c:1928-1929](../raw/postgres-14/src/backend/tcop/utility.c#L1928-L1929)).
- PostgreSQL 18: Holds. The same five events are listed, and `ProcessUtilitySlow` fires them at the same points ([event-trigger.sgml:27-36](../raw/postgres-18/doc/src/sgml/event-trigger.sgml#L27-L36), [pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-18/src/include/catalog/pg_event_trigger.h#L29-L42), [utility.c:1109-1116](../raw/postgres-18/src/backend/tcop/utility.c#L1109-L1116), [utility.c:1946-1949](../raw/postgres-18/src/backend/tcop/utility.c#L1946-L1949)).
- PostgreSQL 19: Holds. The same five events exist, triggers are still `pg_event_trigger` rows, and `ProcessUtilitySlow()` still fires the start event before dispatch and the drop and end events after ([event-trigger.sgml:10-16](../raw/postgres-19/doc/src/sgml/event-trigger.sgml#L10-L16), [event-trigger.sgml:27-37](../raw/postgres-19/doc/src/sgml/event-trigger.sgml#L27-L37), [pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-19/src/include/catalog/pg_event_trigger.h#L31-L45), [utility.c:1118](../raw/postgres-19/src/backend/tcop/utility.c#L1118), [utility.c:1960-1961](../raw/postgres-19/src/backend/tcop/utility.c#L1960-L1961)).

Related: [Hook](#hook), [Utility command](#utility-command), [Catalog](#catalog)

### Executor

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The executor runs a finished plan and produces its rows. Callers drive it through four entry points: `ExecutorStart`, `ExecutorRun`, `ExecutorFinish`, and `ExecutorEnd` [execMain.c header](../raw/postgres-17/src/backend/executor/execMain.c#L1-L28). It takes a `PlannedStmt` as input and never looks at the `Query` tree [plannodes.h#PlannedStmt](../raw/postgres-17/src/include/nodes/plannodes.h#L31-L35) [parsenodes.h#Query](../raw/postgres-17/src/include/nodes/parsenodes.h#L101-L110). Utility commands skip the executor and go to `ProcessUtility` instead [utility.c#ProcessUtility](../raw/postgres-17/src/backend/tcop/utility.c#L498-L525).

**Version notes:**
- PostgreSQL 12: Holds. The four entry points are the same, the executor takes a `PlannedStmt`, and utility commands go to `ProcessUtility` ([execMain.c header](../raw/postgres-12/src/backend/executor/execMain.c#L1-L28), [plannodes.h#PlannedStmt](../raw/postgres-12/src/include/nodes/plannodes.h#L42), [parsenodes.h#Query](../raw/postgres-12/src/include/nodes/parsenodes.h#L108), [utility.c#ProcessUtility](../raw/postgres-12/src/backend/tcop/utility.c#L338)).
- PostgreSQL 14: Holds. The four `Executor*` entry points, the `PlannedStmt` input and the `ProcessUtility` bypass are the same ([execMain.c header](../raw/postgres-14/src/backend/executor/execMain.c#L1-L28), [plannodes.h#PlannedStmt](../raw/postgres-14/src/include/nodes/plannodes.h#L42), [parsenodes.h#Query](../raw/postgres-14/src/include/nodes/parsenodes.h#L116), [utility.c#ProcessUtility](../raw/postgres-14/src/backend/tcop/utility.c#L503-L530)).
- PostgreSQL 18: Holds ([execMain.c header](../raw/postgres-18/src/backend/executor/execMain.c#L1-L28), [plannodes.h#PlannedStmt](../raw/postgres-18/src/include/nodes/plannodes.h#L31-L35), [utility.c#ProcessUtility](../raw/postgres-18/src/backend/tcop/utility.c#L498-L525)).
- PostgreSQL 19: Holds. The four entry points are unchanged, the executor still takes a `PlannedStmt` and never reads the `Query`, and utility commands still go to `ProcessUtility()` ([execMain.c header](../raw/postgres-19/src/backend/executor/execMain.c#L1-L28), [plannodes.h#PlannedStmt](../raw/postgres-19/src/include/nodes/plannodes.h#L44-L58), [parsenodes.h#Query](../raw/postgres-19/src/include/nodes/parsenodes.h#L100-L110), [utility.c#ProcessUtility](../raw/postgres-19/src/backend/tcop/utility.c#L504-L530)).

Related: [PlannedStmt](#plannedstmt), [Portal](#portal), [Planner](#planner), [Utility command](#utility-command)

### Executor state

**Aliases:** `EState`, `PlanState`, plan-state tree, `ExecInitNode`, `ExecProcNode`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Executor state is the run-time data the executor builds from a finished plan. All data that changes during execution lives in a state tree that mirrors the plan tree, so the plan itself stays read-only and can be cached and reused ([executor/README#Plan Trees and State Trees](../raw/postgres-17/src/backend/executor/README#L47-L59)). `ExecInitNode()` walks the plan and builds that tree of `PlanState` nodes. `ExecProcNode()` then pulls tuples from its top node, and each node pulls from its children ([execProcnode.c NOTES](../raw/postgres-17/src/backend/executor/execProcnode.c#L43-L61)). Every `PlanState` points to its `Plan`, to the shared `EState`, to the function that returns its next tuple, and to optional run-time instrumentation ([execnodes.h#PlanState](../raw/postgres-17/src/include/nodes/execnodes.h#L1113-L1130)). The single `EState` holds the per-query context: the scan direction, the snapshot, the range table, opened relations and the `PlannedStmt` ([execnodes.h#EState](../raw/postgres-17/src/include/nodes/execnodes.h#L615-L637)).

**Version notes:**
- PostgreSQL 12: Holds ([execnodes.h#EState](../raw/postgres-12/src/include/nodes/execnodes.h#L496), [execnodes.h#PlanState](../raw/postgres-12/src/include/nodes/execnodes.h#L936-L946), [execProcnode.c#ExecInitNode](../raw/postgres-12/src/backend/executor/execProcnode.c#L139)).
- PostgreSQL 14: Holds ([execnodes.h#EState](../raw/postgres-14/src/include/nodes/execnodes.h#L566), [execnodes.h#PlanState](../raw/postgres-14/src/include/nodes/execnodes.h#L986-L996), [execProcnode.c#ExecInitNode](../raw/postgres-14/src/backend/executor/execProcnode.c#L142)).
- PostgreSQL 18: Holds ([execnodes.h#EState](../raw/postgres-18/src/include/nodes/execnodes.h#L649), [execnodes.h#PlanState](../raw/postgres-18/src/include/nodes/execnodes.h#L1149-L1161), [execProcnode.c#ExecInitNode](../raw/postgres-18/src/backend/executor/execProcnode.c#L142)).
- PostgreSQL 19: Holds ([execnodes.h#EState](../raw/postgres-19/src/include/nodes/execnodes.h#L690), [execnodes.h#PlanState](../raw/postgres-19/src/include/nodes/execnodes.h#L1195-L1207), [execProcnode.c#ExecInitNode](../raw/postgres-19/src/backend/executor/execProcnode.c#L142)).

Related: [Executor](#executor), [PlannedStmt](#plannedstmt), [Snapshot](#snapshot), [Range table](#range-table), [EXPLAIN](#explain)

### EXPLAIN

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`EXPLAIN` shows the plan the planner chose, with its estimated costs and row counts. With `ANALYZE`, it also runs the statement and reports what actually happened [commands/explain.c#ExplainQuery](../raw/postgres-17/src/backend/commands/explain.c#L175-L207). `ExplainQuery()` parses the option list into an `ExplainState`. It rejects `TIMING` and `SERIALIZE` without `ANALYZE`, and it rejects `GENERIC_PLAN` combined with `ANALYZE` [commands/explain.c:284-305](../raw/postgres-17/src/backend/commands/explain.c#L284-L305). `NewExplainState()` turns only `COSTS` on by default [commands/explain.c#NewExplainState](../raw/postgres-17/src/backend/commands/explain.c#L370-L377).

**Version notes:**
- PostgreSQL 12: Differs in options. `ExplainQuery()` accepts `ANALYZE`, `VERBOSE`, `COSTS`, `BUFFERS`, `SETTINGS`, `TIMING`, `SUMMARY` and `FORMAT`; 12 has no `SERIALIZE`, `GENERIC_PLAN`, `WAL` or `MEMORY` ([explain.c#ExplainQuery](../raw/postgres-12/src/backend/commands/explain.c#L159-L201)). It rejects `BUFFERS` and `TIMING` without `ANALYZE` ([explain.c:206-218](../raw/postgres-12/src/backend/commands/explain.c#L206-L218)). `NewExplainState()` turns only `COSTS` on ([explain.c#NewExplainState](../raw/postgres-12/src/backend/commands/explain.c#L286-L296)).
- PostgreSQL 14: Differs. 14 has no `SERIALIZE` or `GENERIC_PLAN` option; `ExplainQuery()` rejects `WAL` and `TIMING` without `ANALYZE` ([explain.c#ExplainQuery](../raw/postgres-14/src/backend/commands/explain.c#L230-L242)). `NewExplainState()` turns only `COSTS` on by default ([explain.c#NewExplainState](../raw/postgres-14/src/backend/commands/explain.c#L311-L321)).
- PostgreSQL 18: Holds, with the option code moved. `ExplainQuery()` now calls `ParseExplainOptionList()`, and both it and `NewExplainState()` live in the new file `explain_state.c` ([explain.c#ExplainQuery](../raw/postgres-18/src/backend/commands/explain.c#L171-L186), [explain_state.c#NewExplainState](../raw/postgres-18/src/backend/commands/explain_state.c#L57-L71)). The same checks reject `TIMING` and `SERIALIZE` without `ANALYZE`, and `GENERIC_PLAN` with `ANALYZE`. `BUFFERS` now defaults to the `ANALYZE` setting, and unknown options go to extension-registered EXPLAIN options before an error ([explain_state.c#ParseExplainOptionList](../raw/postgres-18/src/backend/commands/explain_state.c#L162-L199)). `NewExplainState()` still sets only `costs` on ([explain_state.c:66](../raw/postgres-18/src/backend/commands/explain_state.c#L66)).
- PostgreSQL 19: Holds, but the code moved. `ExplainQuery()` still runs the statement only with `ANALYZE`. It now calls `ParseExplainOptionList()` in the new file `explain_state.c`, where `NewExplainState()` still turns only `COSTS` on ([explain.c#ExplainQuery](../raw/postgres-19/src/backend/commands/explain.c#L176-L191), [explain_state.c#NewExplainState](../raw/postgres-19/src/backend/commands/explain_state.c#L60-L74)). `TIMING`, `SERIALIZE`, `WAL` and `IO` now require `ANALYZE`, `GENERIC_PLAN` still rejects `ANALYZE`, and `TIMING`, `BUFFERS` and `SUMMARY` default to the `ANALYZE` value ([explain_state.c#ParseExplainOptionList](../raw/postgres-19/src/backend/commands/explain_state.c#L176-L214)). Extensions can register their own options with `RegisterExtensionExplainOption()` ([explain_state.c:12](../raw/postgres-19/src/backend/commands/explain_state.c#L12), [explain_state.c#RegisterExtensionExplainOption](../raw/postgres-19/src/backend/commands/explain_state.c#L332)). For example, `pg_plan_advice` adds `PLAN_ADVICE` ([pgplanadvice.sgml](../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml#L41-L44)).

Related: [EXPLAIN BUFFERS](#explain-buffers), [Planner](#planner), [Cost](#cost)

### EXPLAIN BUFFERS

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`BUFFERS` is an `EXPLAIN` option that counts the blocks each plan node touched: shared, local, and temporary blocks that were hit, read, dirtied, or written [ref/explain.sgml#BUFFERS](../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L180-L206). A hit means the block was already in cache. Read time and write time appear only when `track_io_timing` is on, and the option defaults to off in v17 [ref/explain.sgml#BUFFERS](../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L183-L206). The counters live in `BufferUsage`, and `show_buffer_usage()` prints only non-zero values in text format [instrument.h#BufferUsage](../raw/postgres-17/src/include/executor/instrument.h#L24-L42) [commands/explain.c#show_buffer_usage](../raw/postgres-17/src/backend/commands/explain.c#L3739-L3785).

**Version notes:**
- PostgreSQL 12: Differs. The counters are the same shared, local and temp blocks in `BufferUsage`, and text format prints only non-zero values ([ref/explain.sgml#BUFFERS](../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L166-L191), [instrument.h#BufferUsage](../raw/postgres-12/src/include/executor/instrument.h#L19-L33), [explain.c#show_buffer_usage](../raw/postgres-12/src/backend/commands/explain.c#L2867)). In 12, `BUFFERS` without `ANALYZE` is an error ([explain.c:206-209](../raw/postgres-12/src/backend/commands/explain.c#L206-L209)). I/O timings print only when the read or write time is non-zero ([explain.c:2881-2882](../raw/postgres-12/src/backend/commands/explain.c#L2881-L2882)).
- PostgreSQL 14: Holds, with smaller counters. It reports shared and local hit/read/dirtied/written, but temp blocks only read and written, and one pair of read/write times; defaults to off ([ref/explain.sgml#BUFFERS](../raw/postgres-14/doc/src/sgml/ref/explain.sgml#L171-L194), [instrument.h#BufferUsage](../raw/postgres-14/src/include/executor/instrument.h#L24-L38)). `show_buffer_usage()` prints only non-zero values in text format ([explain.c#show_buffer_usage](../raw/postgres-14/src/backend/commands/explain.c#L3497-L3627)).
- PostgreSQL 18: Differs in the default. `BUFFERS` is now on automatically whenever `ANALYZE` is used, unless it is set explicitly ([ref/explain.sgml#BUFFERS](../raw/postgres-18/doc/src/sgml/ref/explain.sgml#L180-L208), [explain_state.c:180](../raw/postgres-18/src/backend/commands/explain_state.c#L180)). `BufferUsage` and the non-zero-only text output are unchanged ([instrument.h#BufferUsage](../raw/postgres-18/src/include/executor/instrument.h#L24-L42), [explain.c#show_buffer_usage](../raw/postgres-18/src/backend/commands/explain.c#L4082-L4128)).
- PostgreSQL 19: Differs on the default. The option still counts shared, local and temp blocks hit, read, dirtied and written, with times only under `track_io_timing`, and counters still live in `BufferUsage` ([ref/explain.sgml#BUFFERS](../raw/postgres-19/doc/src/sgml/ref/explain.sgml#L183-L203), [instrument.h#BufferUsage](../raw/postgres-19/src/include/executor/instrument.h#L24-L42), [explain.c#show_buffer_usage](../raw/postgres-19/src/backend/commands/explain.c#L4292)). In v19, `BUFFERS` is on by default whenever `ANALYZE` is used, and off otherwise ([ref/explain.sgml#BUFFERS](../raw/postgres-19/doc/src/sgml/ref/explain.sgml#L204-L209), [explain_state.c:184-185](../raw/postgres-19/src/backend/commands/explain_state.c#L184-L185)). `track_io_timing` is `PGC_SUSET` (session scope, superuser only) ([guc_parameters.dat#track_io_timing](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3224-L3228)).

Related: [EXPLAIN](#explain), [Buffer manager](#buffer-manager), [shared_buffers](#shared_buffers), [Dirty buffer](#dirty-buffer)

### Expression index

**Aliases:** index on expressions, functional index. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An expression index stores the result of a function or expression, such as `lower(col1)`, instead of a plain column value. A query can use it when its `WHERE` clause uses the same expression ([indices.sgml#indexes-expressional](../raw/postgres-17/doc/src/sgml/indices.sgml#L732-L760)). In the catalog, each expression column appears as a zero in `pg_index.indkey`, and its expression tree is stored in `pg_index.indexprs` ([pg_index.h#FormData_pg_index](../raw/postgres-17/src/include/catalog/pg_index.h#L48-L61)). An expression index is not a partial index. The first changes what is indexed; the second changes which rows are indexed.

**Version notes:**
- PostgreSQL 12: Holds. The docs describe indexes on expressions, and `pg_index` stores 0 in `indkey` with the tree in `indexprs` ([indices.sgml#indexes-expressional](../raw/postgres-12/doc/src/sgml/indices.sgml#L679-L700), [pg_index.h#FormData_pg_index](../raw/postgres-12/src/include/catalog/pg_index.h#L47-L57)).
- PostgreSQL 14: Holds. The docs section is the same, and each expression column is a zero in `indkey` with its tree in `indexprs` ([indices.sgml#indexes-expressional](../raw/postgres-14/doc/src/sgml/indices.sgml#L730-L760), [pg_index.h#FormData_pg_index](../raw/postgres-14/src/include/catalog/pg_index.h#L47-L58)).
- PostgreSQL 18: Holds ([indices.sgml#indexes-expressional](../raw/postgres-18/doc/src/sgml/indices.sgml#L772-L800), [pg_index.h#FormData_pg_index](../raw/postgres-18/src/include/catalog/pg_index.h#L48-L61)).
- PostgreSQL 19: Holds. The docs are unchanged, and an expression column is still a zero in `pg_index.indkey` with its tree in `indexprs` ([indices.sgml#indexes-expressional](../raw/postgres-19/doc/src/sgml/indices.sgml#L772-L800), [pg_index.h#FormData_pg_index](../raw/postgres-19/src/include/catalog/pg_index.h#L50-L62)).

Related: [Partial index](#partial-index), [pg_index](#pg_index), [Planner](#planner)

### Extended statistics

**Aliases:** `CREATE STATISTICS`, statistics object, `pg_statistic_ext`, `pg_statistic_ext_data`, ndistinct, functional dependencies, multivariate MCV. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Extended statistics are optional statistics that cover several columns or expressions together. By default the planner assumes that conditions on different columns are independent. Extended statistics record the dependencies that break that assumption ([statistics/README](../raw/postgres-17/src/backend/statistics/README#L1-L21)). `CREATE STATISTICS` makes a statistics object, stored in `pg_statistic_ext` with its columns, expressions and requested kinds ([pg_statistic_ext.h#FormData_pg_statistic_ext](../raw/postgres-17/src/include/catalog/pg_statistic_ext.h#L33-L62)). There are four kinds: ndistinct, functional dependencies, multivariate MCV lists and per-expression statistics ([pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-17/src/include/catalog/pg_statistic_ext.h#L84-L87)). ANALYZE writes the built values into `pg_statistic_ext_data`, one row per object and inheritance flag ([pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../raw/postgres-17/src/include/catalog/pg_statistic_ext_data.h#L31-L45)). `clauselist_selectivity()` asks `statext_clauselist_selectivity()` to estimate the clauses it can first, then estimates the rest one by one ([clausesel.c#clauselist_selectivity_ext](../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L145-L156)).

**Version notes:**
- PostgreSQL 12: Only the ndistinct, dependencies and MCV kinds exist, with no expression statistics ([pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-12/src/include/catalog/pg_statistic_ext.h#L67-L69)). `pg_statistic_ext_data` has one row per object, with no inheritance flag ([pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../raw/postgres-12/src/include/catalog/pg_statistic_ext_data.h#L31-L43)).
- PostgreSQL 14: Adds the expressions kind ([pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-14/src/include/catalog/pg_statistic_ext.h#L84-L87)). `pg_statistic_ext_data` still has no inheritance flag ([pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../raw/postgres-14/src/include/catalog/pg_statistic_ext_data.h#L31-L44)).
- PostgreSQL 18: Holds as in 17 ([pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-18/src/include/catalog/pg_statistic_ext.h#L84-L87), [pg_statistic_ext_data.h:35](../raw/postgres-18/src/include/catalog/pg_statistic_ext_data.h#L35)).
- PostgreSQL 19: Holds as in 17 ([pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-19/src/include/catalog/pg_statistic_ext.h#L88-L91), [pg_statistic_ext_data.h:37](../raw/postgres-19/src/include/catalog/pg_statistic_ext_data.h#L37)). New functions `pg_restore_extended_stats()` and `pg_clear_extended_stats()` load and clear an object's data ([pg_proc.dat#pg_restore_extended_stats](../raw/postgres-19/src/include/catalog/pg_proc.dat#L12615-L12622)).

Related: [Statistics](#statistics), [Selectivity](#selectivity), [Most common values and histogram](#most-common-values-and-histogram), [Planner](#planner)

### Extension

**Aliases:** `CREATE EXTENSION`, control file. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An extension is a named package of SQL objects that PostgreSQL installs, tracks, and drops as a unit. It consists of a script file, a control file, and often a shared library of C code ([extend.sgml:525-540](../raw/postgres-17/doc/src/sgml/extend.sgml#L525-L540), [glossary.sgml#glossary-extension](../raw/postgres-17/doc/src/sgml/glossary.sgml#L693-L705)). `CreateExtension` parses the control file into an `ExtensionControlFile` struct. Fields such as `superuser` and `trusted` decide who may install it ([extension.c#ExtensionControlFile](../raw/postgres-17/src/backend/commands/extension.c#L79-L96), [extension.c#CreateExtension](../raw/postgres-17/src/backend/commands/extension.c#L1898)). When the library loads, its `_PG_init()` function runs, which is where extensions usually install [hooks](#hook) ([dfmgr.c:284-289](../raw/postgres-17/src/backend/utils/fmgr/dfmgr.c#L284-L289)).

**Version notes:**
- PostgreSQL 12: Differs. An extension is a script file plus control file, often with a shared library ([extend.sgml:328-341](../raw/postgres-12/doc/src/sgml/extend.sgml#L328-L341)). 12's `ExtensionControlFile` has `superuser` but no `trusted` field; trusted extensions do not exist in 12 ([extension.c#ExtensionControlFile](../raw/postgres-12/src/backend/commands/extension.c#L76-L89)). `_PG_init()` runs when the library loads ([dfmgr.c:285-287](../raw/postgres-12/src/backend/utils/fmgr/dfmgr.c#L285-L287)).
- PostgreSQL 14: Holds. An extension needs a script file and a control file, `ExtensionControlFile` carries `superuser` and `trusted`, and loading the library runs `_PG_init()` ([extend.sgml:532-535](../raw/postgres-14/doc/src/sgml/extend.sgml#L532-L535), [extension.c#ExtensionControlFile](../raw/postgres-14/src/backend/commands/extension.c#L80-L94), [extension.c#CreateExtension](../raw/postgres-14/src/backend/commands/extension.c#L1832), [dfmgr.c:285-287](../raw/postgres-14/src/backend/utils/fmgr/dfmgr.c#L285-L287)).
- PostgreSQL 18: Holds. `ExtensionControlFile` keeps `superuser` and `trusted`, and gains `basedir` and `control_dir` ([extension.c#ExtensionControlFile](../raw/postgres-18/src/backend/commands/extension.c#L85-L104)). New in 18: the `extension_control_path` GUC (default `$system`) lists directories to search for control files. Its context is `superuser`, so a superuser can set it per session ([guc_tables.c#extension_control_path](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L4403-L4411)). `_PG_init()` still runs at library load ([dfmgr.c:294-299](../raw/postgres-18/src/backend/utils/fmgr/dfmgr.c#L294-L299)).
- PostgreSQL 19: Holds. An extension is still a script file plus a control file plus an optional library, `ExtensionControlFile` still carries `superuser` and `trusted`, and loading a library still calls `_PG_init()` ([extend.sgml:527-540](../raw/postgres-19/doc/src/sgml/extend.sgml#L527-L540), [glossary.sgml#glossary-extension](../raw/postgres-19/doc/src/sgml/glossary.sgml#L749-L762), [extension.c#ExtensionControlFile](../raw/postgres-19/src/backend/commands/extension.c#L86-L105), [extension.c#CreateExtension](../raw/postgres-19/src/backend/commands/extension.c#L2149), [dfmgr.c:294-299](../raw/postgres-19/src/backend/utils/fmgr/dfmgr.c#L294-L299)). The struct gains `basedir` and `control_dir`, which record the directory where the control file was found ([extension.c#ExtensionControlFile](../raw/postgres-19/src/backend/commands/extension.c#L88-L91)).

Related: [Contrib](#contrib), [Hook](#hook), [fmgr](#fmgr)

### Fast root

**Aliases:** `btm_fastroot`, `btm_fastlevel`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The fast root is the lowest B-tree level that has only one page. Searches start there instead of at the true root, which skips useless single-page levels after mass deletions ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L366-L381)). The metapage stores both roots ([nbtree.h#BTMetaPageData](../raw/postgres-17/src/include/access/nbtree.h#L103-L110)). `_bt_getrootheight()` returns the fast root's level ([nbtpage.c#_bt_getrootheight](../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L664-L676)). The planner stores that value in `IndexOptInfo.tree_height` ([plancat.c:488-494](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L488-L494)). `btcostestimate()` then charges `(tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost` per descent, and that multiplier is 50.0 ([selfuncs.c:7093-7106](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106), [selfuncs.c:145](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)).

**Version notes:**
- PostgreSQL 12: Holds, except for the constant's name. The metapage keeps the fast root, `_bt_getrootheight()` returns its level, and the planner stores it in `tree_height` ([nbtree/README](../raw/postgres-12/src/backend/access/nbtree/README#L336-L350), [nbtree.h#BTMetaPageData](../raw/postgres-12/src/include/access/nbtree.h#L101-L104), [nbtpage.c#_bt_getrootheight](../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L575-L586), [plancat.c:409-418](../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L418)). 12 has no `DEFAULT_PAGE_CPU_MULTIPLIER`; `btcostestimate()` writes the literal `50.0` ([selfuncs.c:6105-6116](../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6105-L6116)).
- PostgreSQL 14: Holds, but the descent charge is written as a literal. The README, `btm_fastroot`/`btm_fastlevel`, `_bt_getrootheight()` and `IndexOptInfo.tree_height` are the same ([nbtree/README](../raw/postgres-14/src/backend/access/nbtree/README#L380-L392), [nbtree.h#BTMetaPageData](../raw/postgres-14/src/include/access/nbtree.h#L101-L108), [nbtpage.c#_bt_getrootheight](../raw/postgres-14/src/backend/access/nbtree/nbtpage.c#L672-L714), [plancat.c:433-437](../raw/postgres-14/src/backend/optimizer/util/plancat.c#L433-L437)). 14 has no `DEFAULT_PAGE_CPU_MULTIPLIER`: `btcostestimate()` charges `(tree_height + 1) * 50.0 * cpu_operator_cost` ([selfuncs.c:6888-6898](../raw/postgres-14/src/backend/utils/adt/selfuncs.c#L6888-L6898)).
- PostgreSQL 18: Holds, with a changed source path. The metapage and `_bt_getrootheight()` are unchanged ([nbtree.h#BTMetaPageData](../raw/postgres-18/src/include/access/nbtree.h#L104-L111), [nbtpage.c#_bt_getrootheight](../raw/postgres-18/src/backend/access/nbtree/nbtpage.c#L664-L676)). The planner now fills `tree_height` through the new `amgettreeheight` callback; B-tree's `btgettreeheight()` returns `_bt_getrootheight()` ([plancat.c:488-499](../raw/postgres-18/src/backend/optimizer/util/plancat.c#L488-L499), [nbtree.c#btgettreeheight](../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1753-L1760)). `btcostestimate()` still charges `(tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost` with the multiplier at 50.0, now per SAOP or skip-array descent ([selfuncs.c:7794-7807](../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L7794-L7807), [selfuncs.c:145](../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L145)).
- PostgreSQL 19: Holds. The README and metapage fields are unchanged, and `btcostestimate()` still charges `(tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost` with the multiplier at 50.0 ([nbtree/README](../raw/postgres-19/src/backend/access/nbtree/README#L366-L382), [nbtree.h#BTMetaPageData](../raw/postgres-19/src/include/access/nbtree.h#L104-L111), [nbtpage.c#_bt_getrootheight](../raw/postgres-19/src/backend/access/nbtree/nbtpage.c#L672-L680), [selfuncs.c:8150-8161](../raw/postgres-19/src/backend/utils/adt/selfuncs.c#L8150-L8161), [selfuncs.c:144](../raw/postgres-19/src/backend/utils/adt/selfuncs.c#L144)). The planner no longer calls `_bt_getrootheight()` directly. It calls the new AM callback `amgettreeheight`, and B-tree's `btgettreeheight()` returns `_bt_getrootheight()` ([plancat.c:489-501](../raw/postgres-19/src/backend/optimizer/util/plancat.c#L489-L501), [nbtree.c#btgettreeheight](../raw/postgres-19/src/backend/access/nbtree/nbtree.c#L1811-L1814)).

Related: [B-tree](#b-tree), [Metapage](#metapage), [IndexOptInfo](#indexoptinfo), [Cost](#cost)

### Fillfactor

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Fillfactor is a per-table or per-index storage parameter: the percentage of a page to fill before starting a new page. For a table, the leftover space gives `UPDATE` room to put the new row version on the same page, which makes HOT updates more likely ([ref/create_table.sgml#fillfactor](../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1468-L1476)). The heap default is 100, and the minimum is 10 ([rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-17/src/include/utils/rel.h#L348-L349)). Changing it takes `ShareUpdateExclusiveLock` and affects only later inserts ([reloptions.c#fillfactor](../raw/postgres-17/src/backend/access/common/reloptions.c#L174-L194)). B-tree leaf pages default to 90. B-tree applies fillfactor during index builds and rightmost page splits, uses a fixed 70 above the leaf level, and uses 96 when it splits a page full of one duplicate value ([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../raw/postgres-17/src/include/access/nbtree.h#L189-L202)).

**Version notes:**
- PostgreSQL 12: Holds. Heap default 100, minimum 10, and the setting takes `ShareUpdateExclusiveLock` and affects later inserts ([ref/create_table.sgml#fillfactor](../raw/postgres-12/doc/src/sgml/ref/create_table.sgml#L1319-L1337), [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-12/src/include/utils/rel.h#L278-L279), [reloptions.c#fillfactor](../raw/postgres-12/src/backend/access/common/reloptions.c#L166-L186)). B-tree uses 90 at the leaf, a fixed 70 above, and 96 for single-value splits ([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../raw/postgres-12/src/include/access/nbtree.h#L160-L171)).
- PostgreSQL 14: Holds. The heap default is 100 with minimum 10, a change takes `ShareUpdateExclusiveLock`, and B-tree uses 90 at the leaf, 70 above it, and 96 for single-value splits ([rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-14/src/include/utils/rel.h#L330-L331), [reloptions.c#fillfactor](../raw/postgres-14/src/backend/access/common/reloptions.c#L168-L177), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../raw/postgres-14/src/include/access/nbtree.h#L188-L201), [ref/create_table.sgml#fillfactor](../raw/postgres-14/doc/src/sgml/ref/create_table.sgml#L1397-L1412)).
- PostgreSQL 18: Holds ([ref/create_table.sgml#fillfactor](../raw/postgres-18/doc/src/sgml/ref/create_table.sgml#L1594-L1602), [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-18/src/include/utils/rel.h#L359-L360), [reloptions.c#fillfactor](../raw/postgres-18/src/backend/access/common/reloptions.c#L174-L194), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../raw/postgres-18/src/include/access/nbtree.h#L190-L203)).
- PostgreSQL 19: Holds. The heap default is 100 with minimum 10, a change takes `ShareUpdateExclusiveLock` and affects only later inserts, and B-tree keeps 90 for leaves, 70 above them and 96 for single-value splits ([ref/create_table.sgml#fillfactor](../raw/postgres-19/doc/src/sgml/ref/create_table.sgml#L1606-L1625), [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-19/src/include/utils/rel.h#L361-L362), [reloptions.c#fillfactor](../raw/postgres-19/src/backend/access/common/reloptions.c#L189-L198), [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../raw/postgres-19/src/include/access/nbtree.h#L190-L203)).

Related: [B-tree](#b-tree), [Bloat](#bloat), [HOT](#hot), [Page split](#page-split)

### fmgr

**Aliases:** function manager, `FmgrInfo`, `FunctionCallInfo`, `PG_FUNCTION_ARGS`, `Gen_fmgrtab.pl`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

fmgr, the function manager, is the calling convention through which the server calls SQL-callable C functions ([fmgr.h:3-8](../raw/postgres-17/src/include/fmgr.h#L3-L8)). Every such function takes one `FunctionCallInfo` argument, which the `PG_FUNCTION_ARGS` macro declares. It returns a `Datum` ([fmgr.h:38-40](../raw/postgres-17/src/include/fmgr.h#L38-L40), [fmgr.h#FunctionCallInfoBaseData](../raw/postgres-17/src/include/fmgr.h#L85-L95), [fmgr.h:193](../raw/postgres-17/src/include/fmgr.h#L193)). `fmgr_info` resolves a function OID into an `FmgrInfo` that holds the C address and flags such as strictness ([fmgr.c#fmgr_info](../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L113-L130), [fmgr.h#FmgrInfo](../raw/postgres-17/src/include/fmgr.h#L56-L67)). The build generates the built-in function table and the `F_*` OID macros from `pg_proc.dat` with `Gen_fmgrtab.pl` ([Gen_fmgrtab.pl:1-6](../raw/postgres-17/src/backend/utils/Gen_fmgrtab.pl#L1-L6)).

**Version notes:**
- PostgreSQL 12: Holds. Functions take one `FunctionCallInfo` declared by `PG_FUNCTION_ARGS`, and `fmgr_info` fills an `FmgrInfo` ([fmgr.h:1-8](../raw/postgres-12/src/include/fmgr.h#L1-L8), [fmgr.h:38](../raw/postgres-12/src/include/fmgr.h#L38), [fmgr.h#FunctionCallInfoBaseData](../raw/postgres-12/src/include/fmgr.h#L85), [fmgr.h:188](../raw/postgres-12/src/include/fmgr.h#L188), [fmgr.c#fmgr_info](../raw/postgres-12/src/backend/utils/fmgr/fmgr.c#L124), [fmgr.h#FmgrInfo](../raw/postgres-12/src/include/fmgr.h#L56)). `Gen_fmgrtab.pl` builds the tables from `pg_proc.dat` ([Gen_fmgrtab.pl:1-6](../raw/postgres-12/src/backend/utils/Gen_fmgrtab.pl#L1-L6)).
- PostgreSQL 14: Holds. Functions take one `FunctionCallInfo` declared by `PG_FUNCTION_ARGS` and return a `Datum`, `fmgr_info` fills an `FmgrInfo`, and `Gen_fmgrtab.pl` generates the built-in table from `pg_proc.dat` ([fmgr.h:38-40](../raw/postgres-14/src/include/fmgr.h#L38-L40), [fmgr.h#FmgrInfo](../raw/postgres-14/src/include/fmgr.h#L56), [fmgr.h#FunctionCallInfoBaseData](../raw/postgres-14/src/include/fmgr.h#L85), [fmgr.h:193](../raw/postgres-14/src/include/fmgr.h#L193), [fmgr.c#fmgr_info](../raw/postgres-14/src/backend/utils/fmgr/fmgr.c#L126-L129), [Gen_fmgrtab.pl:1-6](../raw/postgres-14/src/backend/utils/Gen_fmgrtab.pl#L1-L6)).
- PostgreSQL 18: Holds ([fmgr.h:3-8](../raw/postgres-18/src/include/fmgr.h#L3-L8), [fmgr.h#FunctionCallInfoBaseData](../raw/postgres-18/src/include/fmgr.h#L85-L95), [fmgr.c#fmgr_info](../raw/postgres-18/src/backend/utils/fmgr/fmgr.c#L113-L130), [Gen_fmgrtab.pl:1-6](../raw/postgres-18/src/backend/utils/Gen_fmgrtab.pl#L1-L6)).
- PostgreSQL 19: Holds. `PG_FUNCTION_ARGS` still declares one `FunctionCallInfo` argument, `fmgr_info()` still fills an `FmgrInfo`, and `Gen_fmgrtab.pl` still generates the built-in table and OID macros from `pg_proc.dat` ([fmgr.h:3-8](../raw/postgres-19/src/include/fmgr.h#L3-L8), [fmgr.h:38-40](../raw/postgres-19/src/include/fmgr.h#L38-L40), [fmgr.h#FunctionCallInfoBaseData](../raw/postgres-19/src/include/fmgr.h#L85-L96), [fmgr.h:193](../raw/postgres-19/src/include/fmgr.h#L193), [fmgr.c#fmgr_info](../raw/postgres-19/src/backend/utils/fmgr/fmgr.c#L128-L132), [fmgr.h#FmgrInfo](../raw/postgres-19/src/include/fmgr.h#L56-L67), [Gen_fmgrtab.pl:1-6](../raw/postgres-19/src/backend/utils/Gen_fmgrtab.pl#L1-L6)).

Related: [OID](#oid), [Extension](#extension), [BKI](#bki)

### Foreign data wrapper

**Aliases:** FDW, `FdwRoutine`, `fdwapi.h`, foreign table, foreign server, `postgres_fdw`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A foreign data wrapper is a plug-in that lets a foreign table read, and optionally write, data that lives outside the local server. The core server sends every operation on a foreign table to its wrapper ([fdwhandler.sgml#fdwhandler](../raw/postgres-17/doc/src/sgml/fdwhandler.sgml#L11-L19)). A foreign table is a relation with `relkind` `'f'` that has no local storage ([pg_class.h#RELKIND_FOREIGN_TABLE](../raw/postgres-17/src/include/catalog/pg_class.h#L171)). The wrapper's handler function returns an `FdwRoutine`, a table of callbacks for sizing, path generation, planning, scanning, modifying and asynchronous execution. `GetFdwRoutine()` calls the handler and checks the result ([fdwapi.h#FdwRoutine](../raw/postgres-17/src/include/foreign/fdwapi.h#L204-L281), [foreign.c#GetFdwRoutine](../raw/postgres-17/src/backend/foreign/foreign.c#L321-L345)). The contrib module `postgres_fdw` is the reference wrapper for other PostgreSQL servers ([postgres_fdw.c#postgres_fdw_handler](../raw/postgres-17/contrib/postgres_fdw/postgres_fdw.c#L553)).

**Version notes:**
- PostgreSQL 12: Holds, but `FdwRoutine` has no asynchronous-execution callbacks ([fdwapi.h#FdwRoutine](../raw/postgres-12/src/include/foreign/fdwapi.h#L183-L249), [pg_class.h#RELKIND_FOREIGN_TABLE](../raw/postgres-12/src/include/catalog/pg_class.h#L161)).
- PostgreSQL 14: Holds, and adds the asynchronous callbacks such as `IsForeignPathAsyncCapable` ([fdwapi.h#FdwRoutine](../raw/postgres-14/src/include/foreign/fdwapi.h#L204-L281), [fdwapi.h:277](../raw/postgres-14/src/include/foreign/fdwapi.h#L277)).
- PostgreSQL 18: Holds ([fdwapi.h#FdwRoutine](../raw/postgres-18/src/include/foreign/fdwapi.h#L204-L281), [pg_class.h#RELKIND_FOREIGN_TABLE](../raw/postgres-18/src/include/catalog/pg_class.h#L174)).
- PostgreSQL 19: Holds ([fdwapi.h#FdwRoutine](../raw/postgres-19/src/include/foreign/fdwapi.h#L208-L286), [pg_class.h#RELKIND_FOREIGN_TABLE](../raw/postgres-19/src/include/catalog/pg_class.h#L178)).

Related: [Contrib](#contrib), [Extension](#extension), [Path](#path), [pg_class](#pg_class)

### Foreign key trigger

**Aliases:** RI trigger, referential integrity trigger, `ri_triggers.c`, `RI_FKey_check`, `RI_FKey_check_ins`, `RI_FKey_noaction_del`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

PostgreSQL enforces a foreign key with internal triggers, not with special executor code. `ri_triggers.c` holds the trigger functions ([ri_triggers.c header](../raw/postgres-17/src/backend/utils/adt/ri_triggers.c#L3-L6)). Creating a foreign key adds "check" triggers on the referencing table for `INSERT` and `UPDATE`, and "action" triggers on the referenced table for `DELETE` and `UPDATE` ([tablecmds.c#CreateFKCheckTrigger](../raw/postgres-17/src/backend/commands/tablecmds.c#L12345-L12351), [tablecmds.c#createForeignKeyActionTriggers](../raw/postgres-17/src/backend/commands/tablecmds.c#L12407-L12414)). The check trigger calls `RI_FKey_check()`, which runs a cached SPI query of the form `SELECT 1 FROM <pktable> x WHERE pkatt1 = $1 ... FOR KEY SHARE OF x` ([ri_triggers.c#RI_FKey_check](../raw/postgres-17/src/backend/utils/adt/ri_triggers.c#L234-L239), [ri_triggers.c:359-366](../raw/postgres-17/src/backend/utils/adt/ri_triggers.c#L359-L366)). That query takes a shared row lock on the referenced row. When several transactions hold shared locks on one row at once, the row's lockers are recorded as a MultiXact ([multixact.c header](../raw/postgres-17/src/backend/access/transam/multixact.c#L6-L8)).

**Version notes:**
- PostgreSQL 12: Holds, with the same SPI lookup and `FOR KEY SHARE` lock ([ri_triggers.c#RI_FKey_check](../raw/postgres-12/src/backend/utils/adt/ri_triggers.c#L234), [ri_triggers.c:356-357](../raw/postgres-12/src/backend/utils/adt/ri_triggers.c#L356-L357)).
- PostgreSQL 14: Holds ([ri_triggers.c#RI_FKey_check](../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L236), [ri_triggers.c:358-359](../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L358-L359)).
- PostgreSQL 18: Holds, and adds temporal foreign keys that use `PERIOD` ([ri_triggers.c#RI_ConstraintInfo](../raw/postgres-18/src/backend/utils/adt/ri_triggers.c#L124), [ri_triggers.c:371-372](../raw/postgres-18/src/backend/utils/adt/ri_triggers.c#L371-L372)).
- PostgreSQL 19: Adds a fast path. For a non-partitioned, non-temporal key backed by a B-tree, `RI_FKey_check()` probes the referenced unique index directly and skips SPI. Inside an active after-trigger queue it batches rows, up to 64 at a time ([ri_triggers.c#RI_FKey_check](../raw/postgres-19/src/backend/utils/adt/ri_triggers.c#L509-L527), [ri_triggers.c#ri_fastpath_is_applicable](../raw/postgres-19/src/backend/utils/adt/ri_triggers.c#L3455-L3480), [ri_triggers.c#RI_FASTPATH_BATCH_SIZE](../raw/postgres-19/src/backend/utils/adt/ri_triggers.c#L224-L232)). Other keys still use the SPI query ([ri_triggers.c:569-570](../raw/postgres-19/src/backend/utils/adt/ri_triggers.c#L569-L570)).

Related: [MultiXact](#multixact), [Row lock](#row-lock), [SPI](#spi), [Heavyweight lock](#heavyweight-lock)

### Fork

**Aliases:** relation fork, `ForkNumber`, main fork. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A fork is one of the separate file sets that make up a single relation's storage. The main fork holds the data. The free space map and visibility map are secondary forks. Unlogged relations also have an init fork ([glossary.sgml#glossary-fork](../raw/postgres-17/doc/src/sgml/glossary.sgml#L799-L811)). In source, the `ForkNumber` enum names them `MAIN_FORKNUM`, `FSM_FORKNUM`, `VISIBILITYMAP_FORKNUM` and `INIT_FORKNUM`. Buffer and storage calls such as `ReadBufferExtended()` take one of these values ([relpath.h#ForkNumber](../raw/postgres-17/src/include/common/relpath.h#L47-L62), [bufmgr.c#ReadBufferExtended](../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L799-L800)).

**Version notes:**
- PostgreSQL 12: Holds. The main, FSM, visibility-map and init forks exist with the same `ForkNumber` names, and `ReadBufferExtended()` takes a fork ([relpath.h#ForkNumber](../raw/postgres-12/src/include/common/relpath.h#L40-L53), [bufmgr.c#ReadBufferExtended](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L641)). 12 has no `glossary.sgml`; `storage.sgml` describes the forks ([storage.sgml](../raw/postgres-12/doc/src/sgml/storage.sgml#L205-L216)).
- PostgreSQL 14: Holds. The same four forks exist in `ForkNumber`, and unlogged relations have an init fork ([relpath.h#ForkNumber](../raw/postgres-14/src/include/common/relpath.h#L40-L53), [glossary.sgml#glossary-fork](../raw/postgres-14/doc/src/sgml/glossary.sgml#L635-L648)).
- PostgreSQL 18: Holds ([glossary.sgml#glossary-fork](../raw/postgres-18/doc/src/sgml/glossary.sgml#L824-L836), [relpath.h#ForkNumber](../raw/postgres-18/src/include/common/relpath.h#L56-L71)).
- PostgreSQL 19: Holds. The glossary still lists main, FSM, VM and init forks, and `ForkNumber` has the same four members ([glossary.sgml#glossary-fork](../raw/postgres-19/doc/src/sgml/glossary.sgml#L855-L868), [relpath.h#ForkNumber](../raw/postgres-19/src/include/common/relpath.h#L56-L69), [bufmgr.c#ReadBufferExtended](../raw/postgres-19/src/backend/storage/buffer/bufmgr.c#L926)).

Related: [Free space map](#free-space-map), [Relfilenumber](#relfilenumber), [Visibility map](#visibility-map)

### Free space map

**Aliases:** FSM, `FSM_FORKNUM`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The free space map records roughly how much free space each page of a relation has, so an insert can find a page with room without scanning the table. It lives in its own fork ([glossary.sgml#glossary-fsm](../raw/postgres-17/doc/src/sgml/glossary.sgml#L814-L828)). It stores one byte per page, as one of 256 space categories ([freespace.c#NOTES](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L14-L50)). Code: `GetPageWithFreeSpace()` finds a page, `RecordPageWithFreeSpace()` updates one, and `FreeSpaceMapVacuum()` updates the upper levels of the map ([freespace.c#GetPageWithFreeSpace](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L137), [freespace.c#FreeSpaceMapVacuum](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L368)). Do not confuse it with the visibility map, a different fork that tracks tuple visibility per page ([glossary.sgml#glossary-vm](../raw/postgres-17/doc/src/sgml/glossary.sgml#L2093-L2105)).

**Version notes:**
- PostgreSQL 12: Holds. The FSM lives in its own fork and stores one byte per page as one of 256 categories ([freespace.c#NOTES](../raw/postgres-12/src/backend/storage/freespace/freespace.c#L14-L41)). `GetPageWithFreeSpace()`, `RecordPageWithFreeSpace()` and `FreeSpaceMapVacuum()` are the entry points ([freespace.c#GetPageWithFreeSpace](../raw/postgres-12/src/backend/storage/freespace/freespace.c#L132), [freespace.c#RecordPageWithFreeSpace](../raw/postgres-12/src/backend/storage/freespace/freespace.c#L181), [freespace.c#FreeSpaceMapVacuum](../raw/postgres-12/src/backend/storage/freespace/freespace.c#L349)). 12 has no `glossary.sgml`; `storage.sgml` describes the FSM and VM forks ([storage.sgml](../raw/postgres-12/doc/src/sgml/storage.sgml#L205-L213)).
- PostgreSQL 14: Holds. The FSM uses one byte per page in 256 categories, with the same three entry points ([glossary.sgml#glossary-fsm](../raw/postgres-14/doc/src/sgml/glossary.sgml#L650-L665), [freespace.c#NOTES](../raw/postgres-14/src/backend/storage/freespace/freespace.c#L36-L63), [freespace.c#GetPageWithFreeSpace](../raw/postgres-14/src/backend/storage/freespace/freespace.c#L136), [freespace.c#FreeSpaceMapVacuum](../raw/postgres-14/src/backend/storage/freespace/freespace.c#L366)).
- PostgreSQL 18: Holds ([glossary.sgml#glossary-fsm](../raw/postgres-18/doc/src/sgml/glossary.sgml#L839-L853), [freespace.c#NOTES](../raw/postgres-18/src/backend/storage/freespace/freespace.c#L14-L50), [freespace.c#GetPageWithFreeSpace](../raw/postgres-18/src/backend/storage/freespace/freespace.c#L137)).
- PostgreSQL 19: Holds. The FSM still stores one byte per page in 256 categories, and `GetPageWithFreeSpace()`, `RecordPageWithFreeSpace()` and `FreeSpaceMapVacuum()` are unchanged ([glossary.sgml#glossary-fsm](../raw/postgres-19/doc/src/sgml/glossary.sgml#L870-L885), [freespace.c#NOTES](../raw/postgres-19/src/backend/storage/freespace/freespace.c#L14-L50), [freespace.c#GetPageWithFreeSpace](../raw/postgres-19/src/backend/storage/freespace/freespace.c#L137), [freespace.c#FreeSpaceMapVacuum](../raw/postgres-19/src/backend/storage/freespace/freespace.c#L368), [glossary.sgml#glossary-vm](../raw/postgres-19/doc/src/sgml/glossary.sgml#L2240-L2253)).

Related: [Fork](#fork), [pg_freespacemap](#pg_freespacemap), [Visibility map](#visibility-map)

### Freezing

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

VACUUM's marking of old row versions as frozen. A frozen row version counts as older than every normal xid, so it stays "in the past" after the xid counter wraps ([maintenance.sgml#vacuum-for-wraparound](../raw/postgres-17/doc/src/sgml/maintenance.sgml#L440-L456)). The reserved `FrozenTransactionId` (2) marks very old tuples ([transam.h](../raw/postgres-17/src/include/access/transam.h#L20-L35)). Modern heap code sets the `HEAP_XMIN_FROZEN` infomask bits instead of overwriting `t_xmin` ([htup_details.h:206](../raw/postgres-17/src/include/access/htup_details.h#L206)). `heap_prepare_freeze_tuple()` decides which xmin, xmax and xvac fields of a tuple are old enough to freeze, and it builds a freeze plan for the page ([heapam.c#heap_prepare_freeze_tuple](../raw/postgres-17/src/backend/access/heap/heapam.c#L7371-L7415)). `vacuum_freeze_min_age` is a `user` GUC, so it can be set per session ([guc_tables.c#vacuum_freeze_min_age](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2666-L2672)).

**Version notes:**
- PostgreSQL 12: Holds. VACUUM marks old rows frozen so they stay in the past after wraparound ([maintenance.sgml#vacuum-for-wraparound](../raw/postgres-12/doc/src/sgml/maintenance.sgml#L400-L413)). `FrozenTransactionId` is reserved, and `HEAP_XMIN_FROZEN` is the infomask combination ([transam.h](../raw/postgres-12/src/include/access/transam.h#L20-L34), [htup_details.h:205](../raw/postgres-12/src/include/access/htup_details.h#L205)). `heap_prepare_freeze_tuple()` checks xmin, xmax and xvac, but in 12 it fills a per-tuple `*frz` entry rather than a page freeze plan ([heapam.c#heap_prepare_freeze_tuple](../raw/postgres-12/src/backend/access/heap/heapam.c#L6085-L6118)). `vacuum_freeze_min_age` is `PGC_USERSET` (session scope) ([guc.c#vacuum_freeze_min_age](../raw/postgres-12/src/backend/utils/misc/guc.c#L2411-L2413)).
- PostgreSQL 14: Holds, with an older freeze interface. `FrozenTransactionId` is 2, and heap code sets `HEAP_XMIN_FROZEN` ([maintenance.sgml#vacuum-for-wraparound](../raw/postgres-14/doc/src/sgml/maintenance.sgml#L435-L456), [transam.h](../raw/postgres-14/src/include/access/transam.h#L24-L33), [htup_details.h:205](../raw/postgres-14/src/include/access/htup_details.h#L205)). In 14, `heap_prepare_freeze_tuple()` fills one `xl_heap_freeze_tuple` per tuple, and `heap_execute_freeze_tuple()` applies it; there is no page-level freeze plan ([heapam.c#heap_prepare_freeze_tuple](../raw/postgres-14/src/backend/access/heap/heapam.c#L7009-L7012), [heapam.c#heap_execute_freeze_tuple](../raw/postgres-14/src/backend/access/heap/heapam.c#L7238)). `vacuum_freeze_min_age` is `PGC_USERSET` (session scope) ([guc.c#vacuum_freeze_min_age](../raw/postgres-14/src/backend/utils/misc/guc.c#L2618)).
- PostgreSQL 18: Holds, with eager scanning added. `FrozenTransactionId`, `HEAP_XMIN_FROZEN` and `heap_prepare_freeze_tuple()` are unchanged ([transam.h](../raw/postgres-18/src/include/access/transam.h#L20-L35), [htup_details.h:206](../raw/postgres-18/src/include/access/htup_details.h#L206), [heapam.c#heap_prepare_freeze_tuple](../raw/postgres-18/src/backend/access/heap/heapam.c#L7272-L7316)). `vacuum_freeze_min_age` is still `user` context (session scope) ([guc_tables.c#vacuum_freeze_min_age](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2794-L2801)). New in 18: a normal (non-aggressive) vacuum may also scan some all-visible pages "eagerly" to freeze them ahead of the next aggressive vacuum ([vacuumlazy.c](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L44-L58)). `vacuum_max_eager_freeze_failure_rate` (default 0.03, 0 disables) bounds this and has `user` context (session scope) ([guc_tables.c#vacuum_max_eager_freeze_failure_rate](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L4155-L4162)).
- PostgreSQL 19: Holds. `FrozenTransactionId` is still 2, frozen tuples still use the `HEAP_XMIN_FROZEN` bits, `heap_prepare_freeze_tuple()` still builds the freeze plan, and `vacuum_freeze_min_age` is still `PGC_USERSET` (session scope) ([maintenance.sgml#vacuum-for-wraparound](../raw/postgres-19/doc/src/sgml/maintenance.sgml#L412-L466), [transam.h](../raw/postgres-19/src/include/access/transam.h#L23-L33), [htup_details.h:206](../raw/postgres-19/src/include/access/htup_details.h#L206), [heapam.c#heap_prepare_freeze_tuple](../raw/postgres-19/src/backend/access/heap/heapam.c#L7217-L7270), [guc_parameters.dat#vacuum_freeze_min_age](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3380-L3386)).

Related: [Transaction ID](#transaction-id), [Wraparound](#wraparound), [xmin and xmax](#xmin-and-xmax), [VACUUM](#vacuum), [MultiXact](#multixact)

### fsync

**Aliases:** `fsync()`, `pg_fsync`, durable flush, `fsync` setting, `data_sync_retry`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`fsync` is the system call PostgreSQL issues to make data it has already written to a file durable on disk. The same name is used for the setting that enables those calls ([config.sgml#guc-fsync](../raw/postgres-17/doc/src/sgml/config.sgml#L3022-L3035)). Wrappers in `fd.c` make the calls: `pg_fsync()` flushes a file, and `pg_flush_data()` only advises the OS to start writing dirty data early, so that a later `fsync` costs less ([fd.c#pg_fsync](../raw/postgres-17/src/backend/storage/file/fd.c#L385-L389), [fd.c#pg_flush_data](../raw/postgres-17/src/backend/storage/file/fd.c#L518-L530)). A backend that writes a data file segment usually does not flush it. `register_dirty_segment()` queues an fsync request for the [checkpoint](#checkpoint) instead. `CheckPointGuts()` writes out dirty buffers and then calls `ProcessSyncRequests()` to flush every queued file ([md.c#register_dirty_segment](../raw/postgres-17/src/backend/storage/smgr/md.c#L1359-L1369), [xlog.c#CheckPointGuts](../raw/postgres-17/src/backend/access/transam/xlog.c#L7530-L7535)). A failed data-file fsync makes the server `PANIC` unless `data_sync_retry` is on, because the OS may already have dropped the dirty data ([fd.c#data_sync_elevel](../raw/postgres-17/src/backend/storage/file/fd.c#L3918-L3934)). The `fsync` setting has context `sighup`, so changing it needs a reload. Turning it off risks unrecoverable corruption after a crash ([guc_tables.c:1096-1107](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1096-L1107), [config.sgml#guc-fsync](../raw/postgres-17/doc/src/sgml/config.sgml#L3037-L3044)). `data_sync_retry` has context `postmaster` and needs a restart ([guc_tables.c:2003](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2003)).

**Version notes:**
- PostgreSQL 12: Holds. `fsync` (context `sighup`) and `data_sync_retry` (context `postmaster`) are defined in `guc.c` ([guc.c:1123](../raw/postgres-12/src/backend/utils/misc/guc.c#L1123), [guc.c:1947](../raw/postgres-12/src/backend/utils/misc/guc.c#L1947)), and the `fd.c` wrappers and checkpoint sync queue exist ([fd.c:333](../raw/postgres-12/src/backend/storage/file/fd.c#L333), [fd.c:405](../raw/postgres-12/src/backend/storage/file/fd.c#L405), [sync.c:236](../raw/postgres-12/src/backend/storage/sync/sync.c#L236)).
- PostgreSQL 14: Holds ([guc.c:1228](../raw/postgres-14/src/backend/utils/misc/guc.c#L1228), [fd.c:352](../raw/postgres-14/src/backend/storage/file/fd.c#L352), [sync.c:294](../raw/postgres-14/src/backend/storage/sync/sync.c#L294)).
- PostgreSQL 18: Holds ([guc_tables.c:1136](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1136), [fd.c:389](../raw/postgres-18/src/backend/storage/file/fd.c#L389), [sync.c:286](../raw/postgres-18/src/backend/storage/sync/sync.c#L286)).
- PostgreSQL 19: Holds; `fsync` is declared in `guc_parameters.dat`, context `sighup` ([guc_parameters.dat:1111](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1111), [fd.c:390](../raw/postgres-19/src/backend/storage/file/fd.c#L390), [sync.c:287](../raw/postgres-19/src/backend/storage/sync/sync.c#L287)).

Related: [Checkpoint](#checkpoint), [WAL](#wal), [OS page cache](#os-page-cache), [Dirty buffer](#dirty-buffer)

### Full-page image

**Aliases:** FPI, full-page write, backup block. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A copy of a whole data page stored inside a WAL record. Replay restores the copy instead of redoing a small change, which guards against pages left half-written by a crash ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L424-L435)). Only the first change to a page after a checkpoint carries one. `XLogRecordAssemble()` adds the image when the page LSN is at or before `RedoRecPtr`, unless `REGBUF_NO_IMAGE` is set or page writes are off ([xloginsert.c#XLogRecordAssemble](../raw/postgres-17/src/backend/access/transam/xloginsert.c#L604-L620)). `full_page_writes` is a `sighup` GUC ([guc_tables.c#full_page_writes](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1157-L1167)).

**Version notes:**
- PostgreSQL 12: Holds. The first change to a page after a checkpoint carries a full-page copy ([transam/README](../raw/postgres-12/src/backend/access/transam/README#L418-L426)). `XLogRecordAssemble()` takes the image when the page LSN is at or before `RedoRecPtr`, unless `REGBUF_NO_IMAGE` is set or page writes are off ([xloginsert.c#XLogRecordAssemble](../raw/postgres-12/src/backend/access/transam/xloginsert.c#L539-L555)). `full_page_writes` is `PGC_SIGHUP` (reload) ([guc.c#full_page_writes](../raw/postgres-12/src/backend/utils/misc/guc.c#L1164-L1166)).
- PostgreSQL 14: Holds. The first change to a page after a checkpoint carries a page copy, and `XLogRecordAssemble()` adds it when the page LSN is at or before `RedoRecPtr`, unless `REGBUF_NO_IMAGE` is set or page writes are off ([transam/README](../raw/postgres-14/src/backend/access/transam/README#L424-L435), [xloginsert.c#XLogRecordAssemble](../raw/postgres-14/src/backend/access/transam/xloginsert.c#L547-L563)). `full_page_writes` is `PGC_SIGHUP`, so a change needs a reload ([guc.c#full_page_writes](../raw/postgres-14/src/backend/utils/misc/guc.c#L1288)).
- PostgreSQL 18: Holds. `full_page_writes` is still `sighup` (reload) ([transam/README](../raw/postgres-18/src/backend/access/transam/README#L424-L435), [xloginsert.c#XLogRecordAssemble](../raw/postgres-18/src/backend/access/transam/xloginsert.c#L604-L620), [guc_tables.c#full_page_writes](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1196-L1206)).
- PostgreSQL 19: Holds. Only the first change after a checkpoint carries an image, `XLogRecordAssemble()` still adds it when the page LSN is at or before `RedoRecPtr` unless `REGBUF_NO_IMAGE` is set or page writes are off, and `full_page_writes` is still `PGC_SIGHUP` (reload) ([transam/README](../raw/postgres-19/src/backend/access/transam/README#L424-L435), [xloginsert.c#XLogRecordAssemble](../raw/postgres-19/src/backend/access/transam/xloginsert.c#L678-L693), [guc_parameters.dat#full_page_writes](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1118-L1123)).

Related: [Checkpoint](#checkpoint), [LSN](#lsn), [WAL](#wal)

### Function volatility

**Aliases:** volatility category, `IMMUTABLE`, `STABLE`, `VOLATILE`, `provolatile`, `PROVOLATILE_IMMUTABLE`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Function volatility is the promise a function makes to the planner about how its result can change. `VOLATILE`, the default, promises nothing and is re-evaluated for every row. `STABLE` promises the same result for the same arguments within one statement. `IMMUTABLE` promises the same result forever ([xfunc.sgml#xfunc-volatility](../raw/postgres-17/doc/src/sgml/xfunc.sgml#L1610-L1667)). The catalog stores it in `pg_proc.provolatile` as `'i'`, `'s'` or `'v'` ([pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-17/src/include/catalog/pg_proc.h#L158-L166)). The planner acts on it. `evaluate_function()` folds an immutable call with constant arguments to a constant, and it folds a stable call only when it is estimating ([clauses.c#evaluate_function](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L4495-L4507)). `contain_volatile_functions()` blocks optimizations such as CTE inlining ([clauses.c#contain_volatile_functions](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L535), [subselect.c#SS_process_ctes](../raw/postgres-17/src/backend/optimizer/plan/subselect.c#L941-L949)). `CREATE INDEX` rejects an index expression that calls any function that is not immutable ([indexcmds.c:1960-1966](../raw/postgres-17/src/backend/commands/indexcmds.c#L1960-L1966)).

**Version notes:**
- PostgreSQL 12: Holds ([pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-12/src/include/catalog/pg_proc.h#L154-L156), [xfunc.sgml#xfunc-volatility](../raw/postgres-12/doc/src/sgml/xfunc.sgml#L1481), [indexcmds.c:1660](../raw/postgres-12/src/backend/commands/indexcmds.c#L1660)).
- PostgreSQL 14: Holds ([pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-14/src/include/catalog/pg_proc.h#L163-L165), [xfunc.sgml#xfunc-volatility](../raw/postgres-14/doc/src/sgml/xfunc.sgml#L1607), [indexcmds.c:1918](../raw/postgres-14/src/backend/commands/indexcmds.c#L1918)).
- PostgreSQL 18: Holds ([pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-18/src/include/catalog/pg_proc.h#L164-L166), [xfunc.sgml#xfunc-volatility](../raw/postgres-18/doc/src/sgml/xfunc.sgml#L1595), [indexcmds.c:2037](../raw/postgres-18/src/backend/commands/indexcmds.c#L2037)).
- PostgreSQL 19: Holds ([pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-19/src/include/catalog/pg_proc.h#L168-L170), [xfunc.sgml#xfunc-volatility](../raw/postgres-19/doc/src/sgml/xfunc.sgml#L1595), [indexcmds.c:2050](../raw/postgres-19/src/backend/commands/indexcmds.c#L2050)).

Related: [Planner](#planner), [Expression index](#expression-index), [Common table expression](#common-table-expression), [Leakproof function](#leakproof-function), [SQL function inlining](#sql-function-inlining)

### GIN

**Aliases:** Generalized Inverted Index. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

GIN is an inverted index. It maps each key found inside a value, such as an array element or a text-search lexeme, to the list of rows that contain it ([gin/README](../raw/postgres-17/src/backend/access/gin/README#L8-L26)). A GIN index has a metapage and a B-tree of key entries. A key's row list is stored either inline as a posting list or as a separate posting tree. If `fastupdate` is on, the index can also have pending-list pages ([gin/README](../raw/postgres-17/src/backend/access/gin/README#L94-L105)). GIN sets `amgettuple` and `amcanreturn` to NULL, so it supports neither plain index scans nor index-only scans ([ginutil.c:70-79](../raw/postgres-17/src/backend/access/gin/ginutil.c#L70-L79)).

**Version notes:**
- PostgreSQL 12: Holds. GIN has a metapage, a B-tree of keys, posting trees, and fast-update pending pages, and it leaves `amgettuple` and `amcanreturn` NULL ([gin/README](../raw/postgres-12/src/backend/access/gin/README#L8-L12), [gin/README](../raw/postgres-12/src/backend/access/gin/README#L95-L105), [ginutil.c:63-71](../raw/postgres-12/src/backend/access/gin/ginutil.c#L63-L71)).
- PostgreSQL 14: Holds. The README describes the same inverted index, with a metapage, a B-tree of key entries, posting lists or posting trees, and pending-list pages under fast update; `amgettuple` and `amcanreturn` are NULL ([gin/README](../raw/postgres-14/src/backend/access/gin/README#L8-L26), [gin/README](../raw/postgres-14/src/backend/access/gin/README#L94-L103), [ginutil.c:68-77](../raw/postgres-14/src/backend/access/gin/ginutil.c#L68-L77)).
- PostgreSQL 18: Holds. GIN still sets `amcanreturn` and `amgettuple` to NULL ([gin/README](../raw/postgres-18/src/backend/access/gin/README#L8-L26), [ginutil.c:74-84](../raw/postgres-18/src/backend/access/gin/ginutil.c#L74-L84)). New in 18: GIN supports parallel index builds (`amcanbuildparallel = true`, false in 17) ([ginutil.c:60](../raw/postgres-18/src/backend/access/gin/ginutil.c#L60), [gininsert.c#GinBuildShared](../raw/postgres-18/src/backend/access/gin/gininsert.c#L49-L96)).
- PostgreSQL 19: Holds. GIN is still an inverted index of keys to posting lists or posting trees, with a metapage and optional pending-list pages, and it still sets `amgettuple` and `amcanreturn` to NULL ([gin/README](../raw/postgres-19/src/backend/access/gin/README#L8-L26), [gin/README](../raw/postgres-19/src/backend/access/gin/README#L95-L105), [ginutil.c:75](../raw/postgres-19/src/backend/access/gin/ginutil.c#L75), [ginutil.c:85](../raw/postgres-19/src/backend/access/gin/ginutil.c#L85)). In v19, `ginhandler()` sets `amcanbuildparallel`, so GIN indexes can be built in parallel ([ginutil.c:61](../raw/postgres-19/src/backend/access/gin/ginutil.c#L61)).

Related: [Posting list](#posting-list), [Posting tree](#posting-tree), [Pending list](#pending-list), [Bitmap scan](#bitmap-scan), [Access method](#access-method)

### GiST

**Aliases:** Generalized Search Tree. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

GiST is a balanced tree framework for building custom index types, such as R-trees for geometric data. The operator class decides what each key means ([gist.sgml#gist-intro](../raw/postgres-17/doc/src/sgml/gist.sgml#L11-L25), [gist/README](../raw/postgres-17/src/backend/access/gist/README#L1-L25)). In PostgreSQL 17, `gisthandler()` marks GiST as supporting ordered nearest-neighbor searches and multicolumn keys, but not unique indexes ([gist.c#gisthandler](../raw/postgres-17/src/backend/access/gist/gist.c#L59-L70)). GiST has no metapage: block 0 is the root ([gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-17/src/include/access/gist_private.h#L262)).

**Version notes:**
- PostgreSQL 12: Holds. `gisthandler()` sets `amcanorderbyop` and `amcanmulticol` true and `amcanunique` false, and block 0 is the root ([gist.c#gisthandler](../raw/postgres-12/src/backend/access/gist/gist.c#L59-L69), [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-12/src/include/access/gist_private.h#L263), [gist.sgml#gist-intro](../raw/postgres-12/doc/src/sgml/gist.sgml#L11)).
- PostgreSQL 14: Holds. `gisthandler()` sets ordered nearest-neighbour search and multicolumn support but not uniqueness, and block 0 is the root ([gist.sgml#gist-intro](../raw/postgres-14/doc/src/sgml/gist.sgml#L11-L25), [gist.c#gisthandler](../raw/postgres-14/src/backend/access/gist/gist.c#L67-L70), [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-14/src/include/access/gist_private.h#L262)).
- PostgreSQL 18: Holds. `gisthandler()` still sets ordered nearest-neighbor search and multicolumn support, but not uniqueness, and block 0 is still the root ([gist.c#gisthandler](../raw/postgres-18/src/backend/access/gist/gist.c#L59-L73), [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-18/src/include/access/gist_private.h#L262)).
- PostgreSQL 19: Holds. `gisthandler()` still marks ordered nearest-neighbor search and multicolumn support but no uniqueness, and block 0 is still the root ([gist.sgml#gist-intro](../raw/postgres-19/doc/src/sgml/gist.sgml#L11-L25), [gist/README](../raw/postgres-19/src/backend/access/gist/README#L1-L25), [gist.c#gisthandler](../raw/postgres-19/src/backend/access/gist/gist.c#L59-L73), [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-19/src/include/access/gist_private.h#L262)).

Related: [SP-GiST](#sp-gist), [Access method](#access-method), [Metapage](#metapage)

### GiST build method

**Aliases:** sorted build, buffering build, `buffering` storage parameter, `GIST_SORTSUPPORT_PROC`, `GistBuildMode`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A [GiST](#gist) index can be built in two ways, and the method chosen affects how full and well-ordered the finished pages are. A sorted build sorts all tuples and packs leaf pages bottom-up, as a B-tree build does. An insert build adds tuples one at a time and may switch to a buffering algorithm that queues tuples at inner levels to cut I/O ([gistbuild.c header](../raw/postgres-17/src/backend/access/gist/gistbuild.c#L6-L22)). `gistbuild()` uses the sorted method only when every key column's [operator class](#operator-class) supplies a `sortsupport` function (`GIST_SORTSUPPORT_PROC`) and the `buffering` [storage parameter](#storage-parameter) is not `on` ([gistbuild.c#gistbuild](../raw/postgres-17/src/backend/access/gist/gistbuild.c#L209-L247), [gist.h:40](../raw/postgres-17/src/include/access/gist.h#L40)). `buffering` accepts `on`, `off` and `auto`, and `auto` is the default ([reloptions.c:521-531](../raw/postgres-17/src/backend/access/common/reloptions.c#L521-L531)). Either way, the build leaves `fillfactor` free space on each page ([gistbuild.c:249-253](../raw/postgres-17/src/backend/access/gist/gistbuild.c#L249-L253)).

**Version notes:**
- PostgreSQL 12: Only the insert build exists, with optional buffering. `GistBufferingMode` has no sorted mode ([gistbuild.c#GistBufferingMode](../raw/postgres-12/src/backend/access/gist/gistbuild.c#L43-L54)). The v12 support-function numbers in `gist.h` stop at `GIST_FETCH_PROC` (9), with no sort-support entry ([gist.h:29-37](../raw/postgres-12/src/include/access/gist.h#L29-L37)). The `buffering` option is a string parsed with `strcmp` ([gistbuild.c:127-141](../raw/postgres-12/src/backend/access/gist/gistbuild.c#L127-L141)).
- PostgreSQL 14: Adds the sorted build with the same selection rule as 17 ([gistbuild.c header](../raw/postgres-14/src/backend/access/gist/gistbuild.c#L15-L16), [gistbuild.c:242](../raw/postgres-14/src/backend/access/gist/gistbuild.c#L242), [gist.h:40](../raw/postgres-14/src/include/access/gist.h#L40)).
- PostgreSQL 18: Holds ([gistbuild.c:246](../raw/postgres-18/src/backend/access/gist/gistbuild.c#L246), [reloptions.c:534](../raw/postgres-18/src/backend/access/common/reloptions.c#L534)).
- PostgreSQL 19: Holds ([gistbuild.c:246](../raw/postgres-19/src/backend/access/gist/gistbuild.c#L246), [reloptions.c:566](../raw/postgres-19/src/backend/access/common/reloptions.c#L566)).

Related: [GiST](#gist), [Operator class](#operator-class), [Storage parameter](#storage-parameter), [Fillfactor](#fillfactor)

### Grammar

**Aliases:** `gram.y`, `scan.l`, raw parser, bison, flex. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The grammar is the bison source `gram.y`. It turns the tokens produced by the flex scanner `scan.l` into a raw [parse tree](#parse-tree) ([parser/README:1-14](../raw/postgres-17/src/backend/parser/README#L1-L14), [gram.y:6-7](../raw/postgres-17/src/backend/parser/gram.y#L6-L7)). `raw_parser` is the entry point that runs this lexical and grammatical analysis ([parser.c#raw_parser](../raw/postgres-17/src/backend/parser/parser.c#L34-L42)). The build generates `scan.c`, `gram.c`, and `gram.h` from these files, so the checkout contains no `gram.c` ([parser/meson.build:30-41](../raw/postgres-17/src/backend/parser/meson.build#L30-L41), [meson.build#bison_kw](../raw/postgres-17/meson.build#L353-L356)). The grammar must not access the database. Name lookup and semantic checks happen later, during parse analysis ([gram.y:25-30](../raw/postgres-17/src/backend/parser/gram.y#L25-L30)).

**Version notes:**
- PostgreSQL 12: Holds, with a different build. `gram.y` turns `scan.l` tokens into a raw parse tree, `raw_parser` is the entry point, and the grammar must not access the database ([parser/README:1-14](../raw/postgres-12/src/backend/parser/README#L1-L14), [gram.y:25-27](../raw/postgres-12/src/backend/parser/gram.y#L25-L27), [parser.c#raw_parser](../raw/postgres-12/src/backend/parser/parser.c#L36)). 12 has no Meson: the `Makefile` generates `gram.c`, `gram.h` and `scan.c`, and it notes that those files ship in the distribution tarball ([parser/Makefile:30-52](../raw/postgres-12/src/backend/parser/Makefile#L30-L52)).
- PostgreSQL 14: Holds, but the build differs. `gram.y` and `scan.l` play the same roles and `raw_parser` is the entry point ([parser/README](../raw/postgres-14/src/backend/parser/README#L11-L13), [parser.c#raw_parser](../raw/postgres-14/src/backend/parser/parser.c#L42-L86)). 14 has no Meson build; `src/backend/parser/Makefile` generates `gram.c`, `gram.h` and `scan.c` ([parser/Makefile:51-60](../raw/postgres-14/src/backend/parser/Makefile#L51-L60)). The grammar still must not access the database ([gram.y:25-30](../raw/postgres-14/src/backend/parser/gram.y#L25-L30)).
- PostgreSQL 18: Holds ([parser/README:1-14](../raw/postgres-18/src/backend/parser/README#L1-L14), [parser.c#raw_parser](../raw/postgres-18/src/backend/parser/parser.c#L34-L42), [parser/meson.build:30-41](../raw/postgres-18/src/backend/parser/meson.build#L30-L41), [gram.y:25-30](../raw/postgres-18/src/backend/parser/gram.y#L25-L30)).
- PostgreSQL 19: Holds. `gram.y` and `scan.l` still produce raw parse trees through `raw_parser()`, the build still generates `scan.c`, `gram.c` and `gram.h`, and the grammar still must not access the database ([parser/README:1-16](../raw/postgres-19/src/backend/parser/README#L1-L16), [gram.y:6-7](../raw/postgres-19/src/backend/parser/gram.y#L6-L7), [parser.c#raw_parser](../raw/postgres-19/src/backend/parser/parser.c#L33-L42), [parser/meson.build:29-43](../raw/postgres-19/src/backend/parser/meson.build#L29-L43), [meson.build#bison_kw](../raw/postgres-19/meson.build#L430-L431), [gram.y:25-31](../raw/postgres-19/src/backend/parser/gram.y#L25-L31)).

Related: [Parse tree](#parse-tree), [Utility command](#utility-command), [BKI](#bki)

### GUC

**Aliases:** Grand Unified Configuration, configuration parameter, setting, `pg_settings`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A GUC is one server configuration parameter, such as `shared_buffers` or `work_mem`. The name comes from the "Grand Unified Configuration" scheme ([guc_tables.c:1-10](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1-L10)). Each parameter is one entry in a typed array: `ConfigureNamesBool`, `ConfigureNamesInt`, `ConfigureNamesReal`, `ConfigureNamesString`, or `ConfigureNamesEnum`. An entry records the name, the [GUC context](#guc-context), the C variable, the default, and the limits ([guc_tables.c:781](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L781), [guc_tables.c#work_mem](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2447-L2456)). The `pg_settings` view shows every GUC's current value and context ([system_views.sql#pg_settings](../raw/postgres-17/src/backend/catalog/system_views.sql#L608-L609)).

**Version notes:**
- PostgreSQL 12: Holds, but the tables live in `guc.c`. `guc_tables.c` does not exist in 12; the typed arrays such as `ConfigureNamesBool` and `ConfigureNamesInt` are static in `guc.c` ([guc.c:1-6](../raw/postgres-12/src/backend/utils/misc/guc.c#L1-L6), [guc.c:881](../raw/postgres-12/src/backend/utils/misc/guc.c#L881), [guc.c:1962](../raw/postgres-12/src/backend/utils/misc/guc.c#L1962), [guc.c#work_mem](../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2240)). `pg_settings` is a view ([system_views.sql#pg_settings](../raw/postgres-12/src/backend/catalog/system_views.sql#L512)).
- PostgreSQL 14: Holds, but the file differs. The typed `ConfigureNames*` arrays live in `src/backend/utils/misc/guc.c`; 14 has no `guc_tables.c` ([guc.c:1-6](../raw/postgres-14/src/backend/utils/misc/guc.c#L1-L6), [guc.c:957](../raw/postgres-14/src/backend/utils/misc/guc.c#L957), [guc.c:2135](../raw/postgres-14/src/backend/utils/misc/guc.c#L2135), [guc.c#work_mem](../raw/postgres-14/src/backend/utils/misc/guc.c#L2415)). `pg_settings` is a view defined in `system_views.sql` ([system_views.sql#pg_settings](../raw/postgres-14/src/backend/catalog/system_views.sql#L579)).
- PostgreSQL 18: Holds. Parameters are still entries in the typed `ConfigureNames*` arrays in `guc_tables.c` ([guc_tables.c:1-10](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1-L10), [guc_tables.c:800](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L800), [system_views.sql#pg_settings](../raw/postgres-18/src/backend/catalog/system_views.sql#L604-L605)).
- PostgreSQL 19: Differs in where GUCs are defined. The five typed `ConfigureNames*` arrays are gone. v19 has one `struct config_generic ConfigureNames[]` array, generated at build time from records in `guc_parameters.dat` by `gen_guc_tables.pl` and pulled into `guc_tables.c` as `guc_tables.inc.c` ([guc_tables.c:1-10](../raw/postgres-19/src/backend/utils/misc/guc_tables.c#L1-L10), [guc_tables.h:311](../raw/postgres-19/src/include/utils/guc_tables.h#L311), [gen_guc_tables.pl:1-10](../raw/postgres-19/src/backend/utils/misc/gen_guc_tables.pl#L1-L10), [guc_tables.c:811](../raw/postgres-19/src/backend/utils/misc/guc_tables.c#L811)). Each record still names the GUC, its context, its variable, its default and its limits, for example `work_mem` ([guc_parameters.dat#work_mem](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3632-L3640)). `pg_settings` still shows every GUC ([system_views.sql#pg_settings](../raw/postgres-19/src/backend/catalog/system_views.sql#L646)).

Related: [GUC context](#guc-context), [shared_buffers](#shared_buffers), [work_mem](#work_mem), [maintenance_work_mem](#maintenance_work_mem)

### GUC context

**Aliases:** `GucContext`, `PGC_POSTMASTER`, `PGC_SIGHUP`, `PGC_USERSET`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A GUC's context says when and by whom the setting can be changed. The seven values are `PGC_INTERNAL`, `PGC_POSTMASTER`, `PGC_SIGHUP`, `PGC_SU_BACKEND`, `PGC_BACKEND`, `PGC_SUSET`, and `PGC_USERSET` ([guc.h#GucContext](../raw/postgres-17/src/include/utils/guc.h#L36-L76)). `pg_settings.context` shows them as `internal`, `postmaster`, `sighup`, `superuser-backend`, `backend`, `superuser`, and `user` ([guc_tables.c#GucContext_Names](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L641-L650)). The wiki maps them to actions: `postmaster` needs a restart, `sighup` needs a reload, and the backend, superuser, and user contexts apply per session or per transaction ([system-views.sgml:3346-3438](../raw/postgres-17/doc/src/sgml/system-views.sgml#L3346-L3438)). For example, `shared_buffers` is `PGC_POSTMASTER` and `work_mem` is `PGC_USERSET` ([guc_tables.c:2262](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2262), [guc_tables.c:2448](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2448)).

**Version notes:**
- PostgreSQL 12: Holds. The same seven contexts exist, with the same `pg_settings` names ([guc.h#GucContext](../raw/postgres-12/src/include/utils/guc.h#L68-L77), [guc.c#GucContext_Names](../raw/postgres-12/src/backend/utils/misc/guc.c#L594-L603)). In 12 the `pg_settings` view is documented in `catalogs.sgml`, not `system-views.sgml` ([catalogs.sgml#view-pg-settings](../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10506-L10610)). `shared_buffers` is `PGC_POSTMASTER` and `work_mem` is `PGC_USERSET` ([guc.c:2155](../raw/postgres-12/src/backend/utils/misc/guc.c#L2155), [guc.c:2231](../raw/postgres-12/src/backend/utils/misc/guc.c#L2231)).
- PostgreSQL 14: Holds. The same seven `GucContext` values map to the same `pg_settings.context` names ([guc.h#GucContext](../raw/postgres-14/src/include/utils/guc.h#L68-L77), [guc.c#GucContext_Names](../raw/postgres-14/src/backend/utils/misc/guc.c#L676-L685)). `shared_buffers` is `PGC_POSTMASTER` and `work_mem` is `PGC_USERSET` ([guc.c:2339](../raw/postgres-14/src/backend/utils/misc/guc.c#L2339), [guc.c:2415](../raw/postgres-14/src/backend/utils/misc/guc.c#L2415)). The context list is documented in `catalogs.sgml` in 14; `system-views.sgml` does not exist ([catalogs.sgml#view-pg-settings](../raw/postgres-14/doc/src/sgml/catalogs.sgml#L12320-L12428)).
- PostgreSQL 18: Holds. The same seven contexts, names and `pg_settings` descriptions ([guc.h#GucContext](../raw/postgres-18/src/include/utils/guc.h#L40-L80), [guc_tables.c#GucContext_Names](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L657-L666), [system-views.sgml:3700-3792](../raw/postgres-18/doc/src/sgml/system-views.sgml#L3700-L3792)). `shared_buffers` is still `PGC_POSTMASTER` and `work_mem` still `PGC_USERSET` ([guc_tables.c:2379](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2379), [guc_tables.c:2576](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2576)).
- PostgreSQL 19: Holds. The seven `GucContext` values and their `pg_settings.context` names are unchanged, `shared_buffers` is still `PGC_POSTMASTER` (restart) and `work_mem` is still `PGC_USERSET` (session scope) ([guc.h#GucContext](../raw/postgres-19/src/include/utils/guc.h#L39-L80), [guc_tables.c#GucContext_Names](../raw/postgres-19/src/backend/utils/misc/guc_tables.c#L694-L703), [system-views.sgml](../raw/postgres-19/doc/src/sgml/system-views.sgml#L3895-L4005), [guc_parameters.dat:2711](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2711), [guc_parameters.dat:3632](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3632)). To find a v19 GUC's context, read its `context =>` field in `guc_parameters.dat`, not a C table.

Related: [GUC](#guc), [Postmaster](#postmaster)

### Hash index

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A hash index places each entry in a bucket chosen by hashing the key. It supports only equality lookups. Buckets are split one at a time as the index grows, and a bucket overflows into chained "overflow pages" ([hash/README](../raw/postgres-17/src/backend/access/hash/README#L13-L28)). Entries store only the 32-bit hash code, not the key value, and each page keeps them sorted by hash code ([hash/README](../raw/postgres-17/src/backend/access/hash/README#L36-L42)). `hashhandler()` marks hash indexes as single-column only, with no uniqueness and no ordering, and block 0 is the metapage ([hash.c#hashhandler](../raw/postgres-17/src/backend/access/hash/hash.c#L57-L68), [hash.h#HASH_METAPAGE](../raw/postgres-17/src/include/access/hash.h#L198)).

**Version notes:**
- PostgreSQL 12: Holds. Buckets split one at a time and use overflow pages, entries hold only the hash code sorted per page, `hashhandler()` sets single-column, non-unique, unordered, and block 0 is the metapage ([hash/README](../raw/postgres-12/src/backend/access/hash/README#L13-L42), [hash.c#hashhandler](../raw/postgres-12/src/backend/access/hash/hash.c#L58-L68), [hash.h#HASH_METAPAGE](../raw/postgres-12/src/include/access/hash.h#L196)).
- PostgreSQL 14: Holds. Buckets, overflow pages, hash-code-only entries sorted per page, single-column and non-unique handler flags, and metapage at block 0 all match ([hash/README](../raw/postgres-14/src/backend/access/hash/README#L13-L15), [hash/README](../raw/postgres-14/src/backend/access/hash/README#L36-L42), [hash.c#hashhandler](../raw/postgres-14/src/backend/access/hash/hash.c#L63-L67), [hash.h#HASH_METAPAGE](../raw/postgres-14/src/include/access/hash.h#L196)).
- PostgreSQL 18: Holds. `hashhandler()` still sets single-column, no uniqueness, no ordering, and block 0 is the metapage ([hash/README](../raw/postgres-18/src/backend/access/hash/README#L13-L28), [hash.c#hashhandler](../raw/postgres-18/src/backend/access/hash/hash.c#L58-L72), [hash.h#HASH_METAPAGE](../raw/postgres-18/src/include/access/hash.h#L198)).
- PostgreSQL 19: Holds. Buckets still split one at a time and chain overflow pages, entries still store only the sorted 32-bit hash code, `hashhandler()` still marks it single-column, non-unique and unordered, and block 0 is still the metapage ([hash/README](../raw/postgres-19/src/backend/access/hash/README#L13-L28), [hash/README](../raw/postgres-19/src/backend/access/hash/README#L35-L42), [hash.c#hashhandler](../raw/postgres-19/src/backend/access/hash/hash.c#L70-L84), [hash.h#HASH_METAPAGE](../raw/postgres-19/src/include/access/hash.h#L198)).

Related: [Access method](#access-method), [Metapage](#metapage)

### Heap

**Aliases:** heap table, heap access method, `heapam`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The heap is the storage that holds a table's rows. It lives in the main fork of the table's files ([glossary.sgml#glossary-heap](../raw/postgres-17/doc/src/sgml/glossary.sgml#L875-L886)). "Heap" is also the name of the built-in table access method. Its catalog row points to the handler `heap_tableam_handler`, which returns the `heapam_methods` callback table ([pg_am.dat#heap](../raw/postgres-17/src/include/catalog/pg_am.dat#L15-L17), [heapam_handler.c#heap_tableam_handler](../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L2652-L2662)). The heap keeps no row order: even after `CLUSTER`, new and updated rows are not placed in index order ([ref/cluster.sgml#description](../raw/postgres-17/doc/src/sgml/ref/cluster.sgml#L45-L49)). An index maps key values to the TIDs of row versions in the heap ([indexam.sgml#intro](../raw/postgres-17/doc/src/sgml/indexam.sgml#L38-L43)).

**Version notes:**
- PostgreSQL 12: Holds. 12 is the first release with table access methods; `heap` is the only one, with handler `heap_tableam_handler` ([pg_am.dat#heap](../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L17), [heapam_handler.c#heap_tableam_handler](../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2666)). `CLUSTER` is a one-time ordering, and an index maps keys to heap TIDs ([ref/cluster.sgml#description](../raw/postgres-12/doc/src/sgml/ref/cluster.sgml#L44-L48), [indexam.sgml#intro](../raw/postgres-12/doc/src/sgml/indexam.sgml#L38-L42)). 12 has no `glossary.sgml` definition.
- PostgreSQL 14: Holds. `heap` is the only table AM, its handler `heap_tableam_handler` returns `heapam_methods`, `CLUSTER` order is not maintained, and indexes store TIDs ([glossary.sgml#glossary-heap](../raw/postgres-14/doc/src/sgml/glossary.sgml#L711-L722), [pg_am.dat#heap](../raw/postgres-14/src/include/catalog/pg_am.dat#L15-L17), [heapam_handler.c#heap_tableam_handler](../raw/postgres-14/src/backend/access/heap/heapam_handler.c#L2605-L2609), [ref/cluster.sgml#description](../raw/postgres-14/doc/src/sgml/ref/cluster.sgml#L49-L52), [indexam.sgml#intro](../raw/postgres-14/doc/src/sgml/indexam.sgml#L39-L40)).
- PostgreSQL 18: Holds ([glossary.sgml#glossary-heap](../raw/postgres-18/doc/src/sgml/glossary.sgml#L900-L911), [pg_am.dat#heap](../raw/postgres-18/src/include/catalog/pg_am.dat#L15-L17), [heapam_handler.c#heap_tableam_handler](../raw/postgres-18/src/backend/access/heap/heapam_handler.c#L2675-L2685)).
- PostgreSQL 19: Holds. The heap still lives in the main fork, `heap` is still the table AM whose handler returns `heapam_methods`, and indexes still map keys to heap TIDs ([glossary.sgml#glossary-heap](../raw/postgres-19/doc/src/sgml/glossary.sgml#L948-L960), [pg_am.dat#heap](../raw/postgres-19/src/include/catalog/pg_am.dat#L15-L17), [heapam_handler.c#heap_tableam_handler](../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L2731-L2734), [indexam.sgml#intro](../raw/postgres-19/doc/src/sgml/indexam.sgml#L37-L44)). The "no row order after clustering" note now lives in the `REPACK` reference page ([ref/repack.sgml](../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L101-L108)).

Related: [Access method](#access-method), [Page](#page), [TID](#tid), [Tuple](#tuple)

### Heavyweight lock

**Aliases:** regular lock, lmgr lock, table-level lock. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The lock manager used for all user-driven locks, such as locks on tables and other database objects. It has eight modes with table-driven conflicts, detects deadlocks, and releases everything at transaction end ([lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L32-L35), [lockdefs.h](../raw/postgres-17/src/include/storage/lockdefs.h#L34-L47), [lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)). A `LOCKTAG` names the locked object ([lock.h#LOCKTAG](../raw/postgres-17/src/include/storage/lock.h#L164-L172)). Wrappers such as `LockRelationOid()` call `LockAcquireExtended()` ([lmgr.c#LockRelationOid](../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L102-L116)). Unlike an LWLock, it has deadlock detection and is held until transaction end ([lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L20-L35)).

**Version notes:**
- PostgreSQL 12: Holds. The README describes regular locks with table-driven modes, deadlock detection, and release at transaction end; LWLocks have none of that ([lmgr/README](../raw/postgres-12/src/backend/storage/lmgr/README#L20-L35)). `LockConflicts`, `LOCKTAG` and `LockRelationOid()` calling `LockAcquireExtended()` match ([lock.c#LockConflicts](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65), [lock.h#LOCKTAG](../raw/postgres-12/src/include/storage/lock.h#L164), [lmgr.c#LockRelationOid](../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L108-L116), [lockdefs.h](../raw/postgres-12/src/include/storage/lockdefs.h#L34-L46)).
- PostgreSQL 14: Holds. Regular locks have eight table-driven modes, deadlock detection and release at transaction end; `LOCKTAG` names the object and `LockRelationOid()` calls `LockAcquireExtended()` ([lmgr/README](../raw/postgres-14/src/backend/storage/lmgr/README#L20-L35), [lockdefs.h](../raw/postgres-14/src/include/storage/lockdefs.h#L34-L47), [lock.c#LockConflicts](../raw/postgres-14/src/backend/storage/lmgr/lock.c#L65-L105), [lock.h#LOCKTAG](../raw/postgres-14/src/include/storage/lock.h#L167-L175), [lmgr.c#LockRelationOid](../raw/postgres-14/src/backend/storage/lmgr/lmgr.c#L109-L117)).
- PostgreSQL 18: Holds. `LockRelationOid()` still calls `LockAcquireExtended()`, which takes one extra argument in 18 ([lmgr/README](../raw/postgres-18/src/backend/storage/lmgr/README#L20-L35), [lock.c#LockConflicts](../raw/postgres-18/src/backend/storage/lmgr/lock.c#L65-L105), [lmgr.c#LockRelationOid](../raw/postgres-18/src/backend/storage/lmgr/lmgr.c#L101-L116)).
- PostgreSQL 19: Holds. It still has eight table-driven modes, deadlock detection and release at transaction end, and `LockRelationOid()` still calls `LockAcquireExtended()` ([lmgr/README](../raw/postgres-19/src/backend/storage/lmgr/README#L32-L35), [lockdefs.h](../raw/postgres-19/src/include/storage/lockdefs.h#L34-L48), [lock.c#LockConflicts](../raw/postgres-19/src/backend/storage/lmgr/lock.c#L68-L108), [lmgr.c#LockRelationOid](../raw/postgres-19/src/backend/storage/lmgr/lmgr.c#L107-L116), [lmgr/README](../raw/postgres-19/src/backend/storage/lmgr/README#L18-L35)). `LOCKTAG` moved from `lock.h` to a new header, `locktag.h` ([locktag.h#LOCKTAG](../raw/postgres-19/src/include/storage/locktag.h#L64-L72)).

Related: [AccessExclusiveLock](#accessexclusivelock), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [LWLock](#lwlock)

### Hook

**Aliases:** `ProcessUtility_hook`, `planner_hook`, `_PG_init`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A hook is a global function-pointer variable that core code checks at a fixed point. When a loaded library has set the pointer, core calls the library's function in place of the standard routine. For example, `ProcessUtility` calls `ProcessUtility_hook` when it is set and `standard_ProcessUtility` otherwise ([utility.c:513-525](../raw/postgres-17/src/backend/tcop/utility.c#L513-L525), [utility.h:71-78](../raw/postgres-17/src/include/tcop/utility.h#L71-L78)). `planner` treats `planner_hook` the same way ([planner.c#planner](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L274-L285)). Libraries usually set hooks in `_PG_init()`, which runs when the library is loaded ([dfmgr.c:284-289](../raw/postgres-17/src/backend/utils/fmgr/dfmgr.c#L284-L289)). A hook function normally chains to the previous hook or to the `standard_` routine. `pg_stat_statements` saves the old `ProcessUtility_hook` and calls it, or `standard_ProcessUtility` when none was set ([pg_stat_statements.c:479-480](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L479-L480), [pg_stat_statements.c:1157-1164](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L1157-L1164)). Hooks are C-level and have no catalog row, unlike an [event trigger](#event-trigger).

**Version notes:**
- PostgreSQL 12: Holds. `ProcessUtility` calls `ProcessUtility_hook` or `standard_ProcessUtility`, `planner` treats `planner_hook` the same way, and `pg_stat_statements` chains to the previous hook ([utility.c:355-360](../raw/postgres-12/src/backend/tcop/utility.c#L355-L360), [utility.h:29-34](../raw/postgres-12/src/include/tcop/utility.h#L29-L34), [planner.c#planner](../raw/postgres-12/src/backend/optimizer/plan/planner.c#L268-L273), [pg_stat_statements.c:437](../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L437), [pg_stat_statements.c:1001-1006](../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1001-L1006)). `_PG_init()` runs at library load ([dfmgr.c:285-287](../raw/postgres-12/src/backend/utils/fmgr/dfmgr.c#L285-L287)).
- PostgreSQL 14: Holds. `ProcessUtility` calls `ProcessUtility_hook` or `standard_ProcessUtility`, `planner` treats `planner_hook` the same way, libraries set hooks in `_PG_init()`, and `pg_stat_statements` chains to the previous hook ([utility.c:520-530](../raw/postgres-14/src/backend/tcop/utility.c#L520-L530), [utility.h:71-78](../raw/postgres-14/src/include/tcop/utility.h#L71-L78), [planner.c#planner](../raw/postgres-14/src/backend/optimizer/plan/planner.c#L264-L274), [dfmgr.c:285-287](../raw/postgres-14/src/backend/utils/fmgr/dfmgr.c#L285-L287), [pg_stat_statements.c:467](../raw/postgres-14/contrib/pg_stat_statements/pg_stat_statements.c#L467), [pg_stat_statements.c:1132-1137](../raw/postgres-14/contrib/pg_stat_statements/pg_stat_statements.c#L1132-L1137)).
- PostgreSQL 18: Holds. `ProcessUtility_hook`, `planner_hook`, `_PG_init()`, and `pg_stat_statements`' chaining are unchanged ([utility.c:513-525](../raw/postgres-18/src/backend/tcop/utility.c#L513-L525), [planner.c#planner](../raw/postgres-18/src/backend/optimizer/plan/planner.c#L287-L310), [pg_stat_statements.c:1169-1176](../raw/postgres-18/contrib/pg_stat_statements/pg_stat_statements.c#L1169-L1176)).
- PostgreSQL 19: Holds. `ProcessUtility()` still calls `ProcessUtility_hook` or `standard_ProcessUtility()`, `planner()` still calls `planner_hook` or `standard_planner()`, `_PG_init()` still runs at load, and `pg_stat_statements` still chains to the previous hook ([utility.c#ProcessUtility](../raw/postgres-19/src/backend/tcop/utility.c#L518-L530), [utility.h:71-78](../raw/postgres-19/src/include/tcop/utility.h#L71-L78), [planner.c#planner](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L328-L342), [dfmgr.c:294-299](../raw/postgres-19/src/backend/utils/fmgr/dfmgr.c#L294-L299), [pg_stat_statements.c:492](../raw/postgres-19/contrib/pg_stat_statements/pg_stat_statements.c#L492), [pg_stat_statements.c:1153-1159](../raw/postgres-19/contrib/pg_stat_statements/pg_stat_statements.c#L1153-L1159)). v19 adds planner hooks `planner_setup_hook`, `planner_shutdown_hook`, `build_simple_rel_hook`, `joinrel_setup_hook` and `join_path_setup_hook` ([planner.c:73-83](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L73-L83), [relnode.c:50-54](../raw/postgres-19/src/backend/optimizer/util/relnode.c#L50-L54), [joinpath.c:31-32](../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L31-L32)). `get_relation_info_hook` is gone: `plancat.h` now opens its declarations with `get_relation_info()` itself and declares no hook type ([plancat.h:17-20](../raw/postgres-19/src/include/optimizer/plancat.h#L17-L20)).

Related: [Extension](#extension), [Event trigger](#event-trigger), [Utility command](#utility-command), [Planner](#planner)

### HOT

**Aliases:** heap-only tuple, HOT update, HOT chain, `HEAP_ONLY_TUPLE`, `HEAP_HOT_UPDATED`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A HOT update writes the new row version on the same heap page and adds no new index entries. The index keeps pointing at the chain's first version, and later versions are found by following the chain ([README.HOT#intro](../raw/postgres-17/src/backend/access/heap/README.HOT#L1-L60)). `heap_update()` chooses HOT only when the new version fits on the same page and no column of a HOT-blocking index changed. Summarizing indexes such as BRIN do not block HOT, but they are still updated when their columns change ([heapam.c#heap_update](../raw/postgres-17/src/backend/access/heap/heapam.c#L4138-L4160), [README.HOT#intro](../raw/postgres-17/src/backend/access/heap/README.HOT#L41-L42)). The tuple header flags `HEAP_HOT_UPDATED` (the old version) and `HEAP_ONLY_TUPLE` (the new version) mark the chain ([htup_details.h#HEAP_ONLY_TUPLE](../raw/postgres-17/src/include/access/htup_details.h#L274-L282)). Here "column used in an index" also includes columns that appear only in a partial-index predicate ([README.HOT#intro](../raw/postgres-17/src/backend/access/heap/README.HOT#L36-L39)).

**Version notes:**
- PostgreSQL 12: Differs. The chain mechanics and the `HEAP_HOT_UPDATED`/`HEAP_ONLY_TUPLE` flags are the same, and partial-index predicate columns count as indexed ([README.HOT](../raw/postgres-12/src/backend/access/heap/README.HOT#L34-L38), [htup_details.h#HEAP_ONLY_TUPLE](../raw/postgres-12/src/include/access/htup_details.h#L280-L281)). In 12 there is no summarizing-index exception: `heap_update()` takes `INDEX_ATTR_BITMAP_ALL`, so a change to a BRIN-indexed column also blocks HOT ([heapam.c#heap_update](../raw/postgres-12/src/backend/access/heap/heapam.c#L2960-L2965)).
- PostgreSQL 14: Differs on summarizing indexes. The HOT chain, the `HEAP_HOT_UPDATED`/`HEAP_ONLY_TUPLE` flags and the partial-predicate rule match ([README.HOT](../raw/postgres-14/src/backend/access/heap/README.HOT#L30-L38), [htup_details.h#HEAP_ONLY_TUPLE](../raw/postgres-14/src/include/access/htup_details.h#L280-L281)). But in 14, `heap_update()` builds its HOT-blocking set from every indexed column (`INDEX_ATTR_BITMAP_ALL`), so a change to a BRIN-indexed column also prevents HOT; the summarizing-index exception does not exist ([heapam.c#heap_update](../raw/postgres-14/src/backend/access/heap/heapam.c#L3287), [heapam.c#heap_update](../raw/postgres-14/src/backend/access/heap/heapam.c#L3980-L3986)).
- PostgreSQL 18: Holds ([README.HOT#intro](../raw/postgres-18/src/backend/access/heap/README.HOT#L1-L60), [heapam.c#heap_update](../raw/postgres-18/src/backend/access/heap/heapam.c#L4114-L4136), [htup_details.h#HEAP_ONLY_TUPLE](../raw/postgres-18/src/include/access/htup_details.h#L288-L296)).
- PostgreSQL 19: Holds. `heap_update()` still chooses HOT only on the same page when no HOT-blocking indexed column changed, summarizing indexes still do not block HOT but are still updated, and `HEAP_HOT_UPDATED` and `HEAP_ONLY_TUPLE` still mark the chain ([README.HOT#intro](../raw/postgres-19/src/backend/access/heap/README.HOT#L34-L45), [heapam.c#heap_update](../raw/postgres-19/src/backend/access/heap/heapam.c#L4063-L4083), [htup_details.h#HEAP_ONLY_TUPLE](../raw/postgres-19/src/include/access/htup_details.h#L295-L296)).

Related: [Fillfactor](#fillfactor), [Line pointer](#line-pointer), [Pruning](#pruning), [Tuple](#tuple)

### Hot standby

**Aliases:** standby server, read-only replica, `hot_standby`, recovery conflict, `max_standby_streaming_delay`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Hot standby means a server that is still replaying WAL, as a replica or during archive recovery, also accepts read-only queries ([high-availability.sgml#hot-standby](../raw/postgres-17/doc/src/sgml/high-availability.sgml#L1508-L1523)). The `hot_standby` setting turns it on. Its context is `postmaster`, so changing it needs a restart ([guc_tables.c#hot_standby](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1797-L1800)). Replay can conflict with those queries. One example is a lock taken on the primary. Another is a cleanup record that removes rows a standby query's snapshot can still see ([high-availability.sgml#hot-standby-conflict](../raw/postgres-17/doc/src/sgml/high-availability.sgml#L1747-L1775)). `ResolveRecoveryConflictWithSnapshot()` finds the conflicting snapshots ([standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-17/src/backend/storage/ipc/standby.c#L456-L470)). Replay waits for them up to `max_standby_streaming_delay` or `max_standby_archive_delay`, and then cancels them ([standby.c#GetStandbyLimitTime](../raw/postgres-17/src/backend/storage/ipc/standby.c#L200-L221)). `max_standby_streaming_delay` has context `sighup`, so a reload applies it ([guc_tables.c#max_standby_streaming_delay](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2171)).

**Version notes:**
- PostgreSQL 12: Holds; both settings keep the same contexts ([guc.c#hot_standby](../raw/postgres-12/src/backend/utils/misc/guc.c#L1753), [guc.c#max_standby_streaming_delay](../raw/postgres-12/src/backend/utils/misc/guc.c#L2086), [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-12/src/backend/storage/ipc/standby.c#L294), [high-availability.sgml#hot-standby](../raw/postgres-12/doc/src/sgml/high-availability.sgml#L1665)).
- PostgreSQL 14: Holds ([guc.c#hot_standby](../raw/postgres-14/src/backend/utils/misc/guc.c#L1905), [guc.c#max_standby_streaming_delay](../raw/postgres-14/src/backend/utils/misc/guc.c#L2259), [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-14/src/backend/storage/ipc/standby.c#L444), [high-availability.sgml#hot-standby-conflict](../raw/postgres-14/doc/src/sgml/high-availability.sgml#L1736)).
- PostgreSQL 18: Holds ([guc_tables.c#hot_standby](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1896), [guc_tables.c#max_standby_streaming_delay](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2288), [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-18/src/backend/storage/ipc/standby.c#L468)).
- PostgreSQL 19: Holds ([guc_parameters.dat#hot_standby](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1236), [guc_parameters.dat#max_standby_streaming_delay](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2159), [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-19/src/backend/storage/ipc/standby.c#L470)).

Related: [WAL](#wal), [Crash recovery](#crash-recovery), [Snapshot](#snapshot), [WAL receiver](#wal-receiver), [xmin horizon](#xmin-horizon)

### Huge pages

**Aliases:** `huge_pages`, `huge_page_size`, `MAP_HUGETLB`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Huge pages are operating-system memory pages larger than the normal size. Using them for the shared memory segment gives smaller page tables and less CPU time spent on memory management ([config.sgml#guc-huge-pages](../raw/postgres-17/doc/src/sgml/config.sgml#L1724-L1726)). The `huge_pages` setting accepts `off`, `on` or `try` and defaults to `try`. `huge_page_size` picks the size, and 0 means the system default. Both have GUC context `postmaster`, so a change needs a restart ([guc_tables.c#huge_pages](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5056-L5064), [guc_tables.c#huge_page_size](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3597-L3607), [guc_tables.c#huge_pages_options](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L363-L374)). On `mmap` platforms, `CreateAnonymousSegment()` rounds the request up to the huge page size and asks for `MAP_HUGETLB`. With `try`, it falls back to normal pages when that fails. It records the outcome in the read-only `huge_pages_status` ([sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-17/src/backend/port/sysv_shmem.c#L597-L636)).

**Version notes:**
- PostgreSQL 12: Differs. `huge_pages` accepts `off`, `on` or `try`, defaults to `try`, and is `PGC_POSTMASTER` (restart) ([guc.c#huge_pages](../raw/postgres-12/src/backend/utils/misc/guc.c#L4471-L4478), [guc.c#huge_pages_options](../raw/postgres-12/src/backend/utils/misc/guc.c#L403-L413)). 12 has neither `huge_page_size` nor `huge_pages_status`; `CreateAnonymousSegment()` asks for `MAP_HUGETLB` at the system huge page size and, with `try`, falls back after a DEBUG1 message ([sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-12/src/backend/port/sysv_shmem.c#L539-L570)). The docs give the same page-table rationale ([config.sgml#guc-huge-pages](../raw/postgres-12/doc/src/sgml/config.sgml#L1533-L1560)).
- PostgreSQL 14: Holds, except for the status setting. `huge_pages` (`off`/`on`/`try`, default `try`) and `huge_page_size` are both `PGC_POSTMASTER`, so a change needs a restart ([guc.c#huge_pages](../raw/postgres-14/src/backend/utils/misc/guc.c#L4918-L4925), [guc.c#huge_page_size](../raw/postgres-14/src/backend/utils/misc/guc.c#L3514-L3521), [guc.c#huge_pages_options](../raw/postgres-14/src/backend/utils/misc/guc.c#L466-L477), [config.sgml#guc-huge-pages](../raw/postgres-14/doc/src/sgml/config.sgml#L1685)). `CreateAnonymousSegment()` rounds up and falls back under `try` ([sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-14/src/backend/port/sysv_shmem.c#L578-L597)). 14 has no `huge_pages_status` setting: the name appears nowhere under `src/` or `doc/`.
- PostgreSQL 18: Holds. `huge_pages` and `huge_page_size` are still `postmaster` context (restart), and `CreateAnonymousSegment()` is unchanged ([config.sgml#guc-huge-pages](../raw/postgres-18/doc/src/sgml/config.sgml#L1801-L1803), [guc_tables.c#huge_pages](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5329-L5337), [guc_tables.c#huge_page_size](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3804-L3814), [sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-18/src/backend/port/sysv_shmem.c#L597-L636)).
- PostgreSQL 19: Holds. `huge_pages` still takes `off`, `on` or `try` with default `try`, `huge_page_size` 0 still means the system default, both are still `PGC_POSTMASTER` (restart), and `CreateAnonymousSegment()` still falls back and sets `huge_pages_status` ([config.sgml#guc-huge-pages](../raw/postgres-19/doc/src/sgml/config.sgml#L1844-L1863), [config.sgml:1874](../raw/postgres-19/doc/src/sgml/config.sgml#L1874), [guc_parameters.dat#huge_page_size](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1248-L1264), [guc_tables.c#huge_pages_options](../raw/postgres-19/src/backend/utils/misc/guc_tables.c#L365-L376), [sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-19/src/backend/port/sysv_shmem.c#L600-L640)).

Related: [shared_buffers](#shared_buffers), [GUC context](#guc-context)

### Index scan

**Aliases:** Index Scan node, `IndexScan`, `nodeIndexscan.c`, plain index scan. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An index scan is the plan node that walks an index in key order and, for each matching entry, fetches the row from the table. It differs from a [bitmap scan](#bitmap-scan), which collects all matching [TIDs](#tid) first, and from an [index-only scan](#index-only-scan), which can skip the table ([nodeIndexscan.c header](../raw/postgres-17/src/backend/executor/nodeIndexscan.c#L15-L28)). In source, `IndexNext()` loops on `index_getnext_slot()`. That asks the [access method](#access-method) for the next TID with `index_getnext_tid()` and then reads the heap tuple with `index_fetch_heap()` ([nodeIndexscan.c#IndexNext](../raw/postgres-17/src/backend/executor/nodeIndexscan.c#L80-L130), [indexam.c#index_getnext_slot](../raw/postgres-17/src/backend/access/index/indexam.c#L632-L698)). Because each fetch can land on a different heap page, the [planner](#planner) charges `random_page_cost` per page when index order is uncorrelated with table order. It charges mostly `seq_page_cost` when they are perfectly correlated, and interpolates by the square of the correlation in between ([costsize.c#cost_index](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L644-L660), [costsize.c:787](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L787)). Default page costs are 1.0 sequential and 4.0 random ([cost.h:24-25](../raw/postgres-17/src/include/optimizer/cost.h#L24-L25)).

**Version notes:**
- PostgreSQL 12: Holds ([nodeIndexscan.c#IndexNext](../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L81), [indexam.c#index_getnext_slot](../raw/postgres-12/src/backend/access/index/indexam.c#L607), [costsize.c#cost_index](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L476), [cost.h:24-25](../raw/postgres-12/src/include/optimizer/cost.h#L24-L25)).
- PostgreSQL 14: Holds ([nodeIndexscan.c#IndexNext](../raw/postgres-14/src/backend/executor/nodeIndexscan.c#L81), [indexam.c#index_getnext_slot](../raw/postgres-14/src/backend/access/index/indexam.c#L658), [costsize.c#cost_index](../raw/postgres-14/src/backend/optimizer/path/costsize.c#L491)).
- PostgreSQL 18: Holds ([nodeIndexscan.c#IndexNext](../raw/postgres-18/src/backend/executor/nodeIndexscan.c#L80), [indexam.c#index_getnext_slot](../raw/postgres-18/src/backend/access/index/indexam.c#L720), [costsize.c#cost_index](../raw/postgres-18/src/backend/optimizer/path/costsize.c#L560)).
- PostgreSQL 19: Holds ([nodeIndexscan.c#IndexNext](../raw/postgres-19/src/backend/executor/nodeIndexscan.c#L82), [indexam.c#index_getnext_slot](../raw/postgres-19/src/backend/access/index/indexam.c#L698), [costsize.c#cost_index](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L545)).

Related: [Bitmap scan](#bitmap-scan), [Index-only scan](#index-only-scan), [Sequential scan](#sequential-scan), [Cost](#cost), [TID](#tid)

### Index vacuuming

**Aliases:** `ambulkdelete`, `amvacuumcleanup`, bulk delete, vacuum cleanup, `IndexBulkDeleteResult`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Index vacuuming is the part of [VACUUM](#vacuum) that removes index entries pointing at dead heap tuples, so the heap slots can then be freed. Each index [access method](#access-method) provides two callbacks in its `IndexAmRoutine`: `ambulkdelete` and `amvacuumcleanup` ([amapi.h:275-276](../raw/postgres-17/src/include/access/amapi.h#L275-L276)). `ambulkdelete` scans the whole index and asks a callback, for each entry's [TID](#tid), whether to delete it. VACUUM may call it more than once if its dead-TID memory fills. `amvacuumcleanup` runs once at the end and may do further work, such as reclaiming empty index pages ([indexam.sgml#ambulkdelete](../raw/postgres-17/doc/src/sgml/indexam.sgml#L390-L434)). In PostgreSQL 17 the heap side drives both through `lazy_vacuum_all_indexes()` and `lazy_cleanup_all_indexes()`, which call `vac_bulkdel_one_index()` and `vac_cleanup_one_index()` for each index ([vacuumlazy.c#lazy_vacuum_all_indexes](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1990), [vacuumlazy.c#lazy_cleanup_all_indexes](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2353), [vacuum.c#vac_bulkdel_one_index](../raw/postgres-17/src/backend/commands/vacuum.c#L2544-L2565)). The AM reports results in `IndexBulkDeleteResult`, including tuples removed and pages deleted or free. VACUUM uses these to update `pg_class` ([genam.h#IndexBulkDeleteResult](../raw/postgres-17/src/include/access/genam.h#L75-L84)). The [INDEX_CLEANUP](#index_cleanup) option can skip this phase ([ref/vacuum.sgml#INDEX_CLEANUP](../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L187-L221)).

**Version notes:**
- PostgreSQL 12: The same two callbacks exist ([amapi.h:213-214](../raw/postgres-12/src/include/access/amapi.h#L213-L214)). The heap side calls them from `lazy_vacuum_index()` and `lazy_cleanup_index()` in `vacuumlazy.c` ([vacuumlazy.c#lazy_vacuum_index](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1742), [vacuumlazy.c#lazy_cleanup_index](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1775)). `IndexBulkDeleteResult` has `pages_removed` and no `pages_newly_deleted` ([genam.h#IndexBulkDeleteResult](../raw/postgres-12/src/include/access/genam.h#L72-L81)).
- PostgreSQL 14: `IndexBulkDeleteResult` gains `pages_newly_deleted` ([genam.h#IndexBulkDeleteResult](../raw/postgres-14/src/include/access/genam.h#L74-L83)), and `lazy_vacuum_all_indexes()` drives the phase ([vacuumlazy.c#lazy_vacuum_all_indexes](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L2218)).
- PostgreSQL 18: Holds ([amapi.h:297-298](../raw/postgres-18/src/include/access/amapi.h#L297-L298), [vacuum.c#vac_bulkdel_one_index](../raw/postgres-18/src/backend/commands/vacuum.c#L2651), [vacuumlazy.c#lazy_vacuum_all_indexes](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L2589)).
- PostgreSQL 19: Holds ([amapi.h:300-301](../raw/postgres-19/src/include/access/amapi.h#L300-L301), [vacuum.c#vac_bulkdel_one_index](../raw/postgres-19/src/backend/commands/vacuum.c#L2666), [vacuumlazy.c#lazy_vacuum_all_indexes](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L2508)).

Related: [VACUUM](#vacuum), [INDEX_CLEANUP](#index_cleanup), [Access method](#access-method), [B-tree page deletion](#b-tree-page-deletion), [maintenance_work_mem](#maintenance_work_mem)

### INDEX_CLEANUP

**Aliases:** `VACUUM (INDEX_CLEANUP ...)`, `vacuum_index_cleanup` storage parameter, `VACOPTVALUE_AUTO`, index vacuuming bypass. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`INDEX_CLEANUP` is a [VACUUM](#vacuum) option that decides whether VACUUM removes dead entries from a table's indexes. It takes `AUTO`, `ON` or `OFF`, and `AUTO` is the default ([ref/vacuum.sgml#INDEX_CLEANUP](../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L187-L221)). `OFF` skips [index vacuuming](#index-vacuuming) and index cleanup entirely. `ON` forces index vacuuming whenever there are dead tuples. `AUTO` lets VACUUM skip it when there is little to gain ([vacuumlazy.c#heap_vacuum_rel](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L388-L402)). In PostgreSQL 17 that bypass applies when fewer than 2% of heap pages hold dead items and the dead-item TIDs use under 32 MB ([vacuumlazy.c:89](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L89), [vacuumlazy.c#lazy_vacuum](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1909-L1933)). The per-table default comes from the `vacuum_index_cleanup` [storage parameter](#storage-parameter), which accepts the same three values ([reloptions.c:508-519](../raw/postgres-17/src/backend/access/common/reloptions.c#L508-L519)). Skipped index cleanup leaves dead entries in the indexes and dead [line pointers](#line-pointer) in the heap until a later VACUUM does it ([ref/vacuum.sgml#INDEX_CLEANUP](../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L205-L220)).

**Version notes:**
- PostgreSQL 12: The option is a plain boolean with no `AUTO`. Index cleanup is on unless turned off, and the `vacuum_index_cleanup` storage parameter is a boolean ([ref/vacuum.sgml#INDEX_CLEANUP](../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L187-L204), [vacuum.h#VacOptTernaryValue](../raw/postgres-12/src/include/commands/vacuum.h#L157-L162), [reloptions.c:1419](../raw/postgres-12/src/backend/access/common/reloptions.c#L1419)).
- PostgreSQL 14: Adds `AUTO` and the 2% bypass ([ref/vacuum.sgml:35](../raw/postgres-14/doc/src/sgml/ref/vacuum.sgml#L35), [vacuum.h:198](../raw/postgres-14/src/include/commands/vacuum.h#L198), [vacuumlazy.c:110](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L110)).
- PostgreSQL 18: Holds ([ref/vacuum.sgml:34](../raw/postgres-18/doc/src/sgml/ref/vacuum.sgml#L34), [vacuumlazy.c:187](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L187), [reloptions.c:523](../raw/postgres-18/src/backend/access/common/reloptions.c#L523)).
- PostgreSQL 19: Holds ([ref/vacuum.sgml:33](../raw/postgres-19/doc/src/sgml/ref/vacuum.sgml#L33), [vacuumlazy.c:187](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L187), [reloptions.c:555](../raw/postgres-19/src/backend/access/common/reloptions.c#L555)).

Related: [VACUUM](#vacuum), [Index vacuuming](#index-vacuuming), [Storage parameter](#storage-parameter), [Line pointer](#line-pointer)

### Index-only scan

**Aliases:** Index Only Scan, IOS. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An index-only scan answers a query from index entries alone. It visits the table only for pages it cannot prove are visible to every transaction. `ExecIndexOnlyScan()` checks the visibility map for each row address, and only when the page is not marked all-visible does it call `index_fetch_heap()` ([nodeIndexonlyscan.c](../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L127-L169)). `EXPLAIN ANALYZE` reports those heap visits as `Heap Fetches` ([explain.c:1992-1994](../raw/postgres-17/src/backend/commands/explain.c#L1992-L1994)). An index AM supports this only if it sets `amcanreturn`. In v17, B-tree, GiST and SP-GiST set it; GIN, hash and BRIN leave it NULL ([nbtree.c:134](../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L134), [gist.c:92](../raw/postgres-17/src/backend/access/gist/gist.c#L92), [spgutils.c:77](../raw/postgres-17/src/backend/access/spgist/spgutils.c#L77), [ginutil.c:70](../raw/postgres-17/src/backend/access/gin/ginutil.c#L70), [hash.c:90](../raw/postgres-17/src/backend/access/hash/hash.c#L90), [brin.c:280](../raw/postgres-17/src/backend/access/brin/brin.c#L280)).

**Version notes:**
- PostgreSQL 12: Holds. The node checks the visibility map and calls `index_fetch_heap()` only for pages not all-visible, and `EXPLAIN ANALYZE` reports `Heap Fetches` ([nodeIndexonlyscan.c](../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L169), [explain.c:1607](../raw/postgres-12/src/backend/commands/explain.c#L1607)). B-tree, GiST and SP-GiST set `amcanreturn`; GIN, hash and BRIN leave it NULL ([nbtree.c:133](../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L133), [gist.c:85](../raw/postgres-12/src/backend/access/gist/gist.c#L85), [spgutils.c:66](../raw/postgres-12/src/backend/access/spgist/spgutils.c#L66), [ginutil.c:63](../raw/postgres-12/src/backend/access/gin/ginutil.c#L63), [hash.c:84](../raw/postgres-12/src/backend/access/hash/hash.c#L84), [brin.c:111](../raw/postgres-12/src/backend/access/brin/brin.c#L111)).
- PostgreSQL 14: Holds. `IndexOnlyNext()` checks the visibility map and calls `index_fetch_heap()` only for pages not all-visible, `EXPLAIN` prints `Heap Fetches`, and B-tree, GiST and SP-GiST set `amcanreturn` while GIN, hash and BRIN leave it NULL ([nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-14/src/backend/executor/nodeIndexonlyscan.c#L164-L172), [explain.c:1765](../raw/postgres-14/src/backend/commands/explain.c#L1765), [nbtree.c:125](../raw/postgres-14/src/backend/access/nbtree/nbtree.c#L125), [gist.c:89](../raw/postgres-14/src/backend/access/gist/gist.c#L89), [spgutils.c:74](../raw/postgres-14/src/backend/access/spgist/spgutils.c#L74), [ginutil.c:68](../raw/postgres-14/src/backend/access/gin/ginutil.c#L68), [hash.c:86](../raw/postgres-14/src/backend/access/hash/hash.c#L86), [brin.c:120](../raw/postgres-14/src/backend/access/brin/brin.c#L120)).
- PostgreSQL 18: Holds. The same AMs set `amcanreturn`: B-tree, GiST and SP-GiST do; GIN, hash and BRIN leave it NULL ([nodeIndexonlyscan.c](../raw/postgres-18/src/backend/executor/nodeIndexonlyscan.c#L128-L170), [explain.c:1982-1984](../raw/postgres-18/src/backend/commands/explain.c#L1982-L1984), [nbtree.c:151](../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L151), [gist.c:95](../raw/postgres-18/src/backend/access/gist/gist.c#L95), [spgutils.c:80](../raw/postgres-18/src/backend/access/spgist/spgutils.c#L80), [ginutil.c:74](../raw/postgres-18/src/backend/access/gin/ginutil.c#L74), [hash.c:94](../raw/postgres-18/src/backend/access/hash/hash.c#L94), [brin.c:286](../raw/postgres-18/src/backend/access/brin/brin.c#L286)).
- PostgreSQL 19: Holds. `IndexOnlyNext()` still checks the visibility map and calls `index_fetch_heap()` only when the page is not all-visible, EXPLAIN still reports `Heap Fetches`, and B-tree, GiST and SP-GiST still set `amcanreturn` while GIN, hash and BRIN leave it NULL ([nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-19/src/backend/executor/nodeIndexonlyscan.c#L131-L176), [explain.c:1996](../raw/postgres-19/src/backend/commands/explain.c#L1996), [nbtree.c:154](../raw/postgres-19/src/backend/access/nbtree/nbtree.c#L154), [gist.c:95](../raw/postgres-19/src/backend/access/gist/gist.c#L95), [spgutils.c:80](../raw/postgres-19/src/backend/access/spgist/spgutils.c#L80), [ginutil.c:75](../raw/postgres-19/src/backend/access/gin/ginutil.c#L75), [hash.c:106](../raw/postgres-19/src/backend/access/hash/hash.c#L106), [brin.c:290](../raw/postgres-19/src/backend/access/brin/brin.c#L290)).

Related: [Visibility map](#visibility-map), [Bitmap scan](#bitmap-scan), [EXPLAIN](#explain), [Access method](#access-method)

### IndexOptInfo

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`IndexOptInfo` is the planner's summary of one index on a table being planned. It holds the index's size and height, its columns, and its access method's cost-estimate callback [pathnodes.h#IndexOptInfo](../raw/postgres-17/src/include/nodes/pathnodes.h#L1107-L1128) [pathnodes.h:1206](../raw/postgres-17/src/include/nodes/pathnodes.h#L1206). `get_relation_info()` builds it. For a non-partial index, `pages` is the index's current physical block count, and `tuples` is copied from the parent table. For B-tree only, `tree_height` comes from `_bt_getrootheight()`; every other access method gets -1 [plancat.c:463-500](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L500). `btcostestimate()` charges `(tree_height + 1) * DEFAULT_PAGE_CPU_MULTIPLIER * cpu_operator_cost` per descent, and that multiplier is 50.0. Its comment names this charge as what keeps bloated indexes from looking as cheap as unbloated ones [selfuncs.c#btcostestimate](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7093-L7106) [selfuncs.c:145](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145).

**Version notes:**
- PostgreSQL 12: Holds, except for the constant's name. The struct carries `pages`, `tuples`, `tree_height` and `amcostestimate` ([pathnodes.h#IndexOptInfo](../raw/postgres-12/src/include/nodes/pathnodes.h#L781-L793), [pathnodes.h:834](../raw/postgres-12/src/include/nodes/pathnodes.h#L834)). For a non-partial index, `pages` is the block count and `tuples` is the table's; only B-tree gets a real `tree_height` ([plancat.c:394-418](../raw/postgres-12/src/backend/optimizer/util/plancat.c#L394-L418)). `btcostestimate()` uses the literal `50.0`, with the same bloat comment ([selfuncs.c#btcostestimate](../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6105-L6116)).
- PostgreSQL 14: Holds, with the literal descent charge. The struct has `pages`, `tree_height` and `amcostestimate` ([pathnodes.h#IndexOptInfo](../raw/postgres-14/src/include/nodes/pathnodes.h#L832-L843), [pathnodes.h:887](../raw/postgres-14/src/include/nodes/pathnodes.h#L887)). `get_relation_info()` sets `pages` from the physical block count and `tuples` from the table for non-partial indexes, and `tree_height` only for B-tree ([plancat.c:418-441](../raw/postgres-14/src/backend/optimizer/util/plancat.c#L418-L441)). `btcostestimate()` writes the charge as `(tree_height + 1) * 50.0 * cpu_operator_cost`, with the same comment about bloated indexes ([selfuncs.c#btcostestimate](../raw/postgres-14/src/backend/utils/adt/selfuncs.c#L6888-L6898)).
- PostgreSQL 18: Holds, with one change. `tree_height` no longer comes from a B-tree-only test. `get_relation_info()` calls the AM's `amgettreeheight` callback when it is set and uses -1 otherwise. In core only B-tree sets it ([plancat.c:463-500](../raw/postgres-18/src/backend/optimizer/util/plancat.c#L463-L500), [nbtree.c:153](../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L153)). The struct and the `btcostestimate()` descent charge are unchanged ([pathnodes.h#IndexOptInfo](../raw/postgres-18/src/include/nodes/pathnodes.h#L1137-L1158), [selfuncs.c#btcostestimate](../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L7794-L7807)).
- PostgreSQL 19: Holds for the struct's role and for `pages` and `tuples` from `get_relation_info()`, and `btcostestimate()` still charges 50 × `cpu_operator_cost` per level descended ([pathnodes.h#IndexOptInfo](../raw/postgres-19/src/include/nodes/pathnodes.h#L1346-L1367), [pathnodes.h:1449](../raw/postgres-19/src/include/nodes/pathnodes.h#L1449), [plancat.c:465-487](../raw/postgres-19/src/backend/optimizer/util/plancat.c#L465-L487), [selfuncs.c#btcostestimate](../raw/postgres-19/src/backend/utils/adt/selfuncs.c#L8150-L8161), [selfuncs.c:144](../raw/postgres-19/src/backend/utils/adt/selfuncs.c#L144)). `tree_height` now comes from the AM's `amgettreeheight` callback, not a B-tree-only call. Only B-tree sets the callback in core, so other AMs still get -1 ([plancat.c:489-501](../raw/postgres-19/src/backend/optimizer/util/plancat.c#L489-L501), [nbtree.c:156](../raw/postgres-19/src/backend/access/nbtree/nbtree.c#L156), [amapi.h:304](../raw/postgres-19/src/include/access/amapi.h#L304)).

Related: [RelOptInfo](#reloptinfo), [Cost](#cost), [B-tree](#b-tree), [Bloat](#bloat), [reltuples and relpages](#reltuples-and-relpages)

### Inheritance

**Aliases:** table inheritance, `INHERITS`, `pg_inherits`, inheritance child, inheritance parent, `inh` flag, appendrel. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Table inheritance lets a child table take its parent's columns. A query on the parent then also reads every child, unless it says `ONLY` ([ddl.sgml#ddl-inherit](../raw/postgres-17/doc/src/sgml/ddl.sgml#L3489-L3501), [ddl.sgml:3582-3588](../raw/postgres-17/doc/src/sgml/ddl.sgml#L3582-L3588)). The `pg_inherits` catalog stores one row per parent-child link. Declarative partitions are recorded there as well ([pg_inherits.h#FormData_pg_inherits](../raw/postgres-17/src/include/catalog/pg_inherits.h#L32-L38)). In the planner, a range-table entry with the `inh` bit set stands for the whole set. `expand_inherited_rtentry()` adds an entry for each child and builds an "appendrel", which usually becomes an `Append` plan ([inherit.c#expand_inherited_rtentry](../raw/postgres-17/src/backend/optimizer/util/inherit.c#L60-L88)). So partitioned tables and traditional inheritance share much planner code. The difference is that a partitioned parent has no rows of its own to scan ([inherit.c:73-78](../raw/postgres-17/src/backend/optimizer/util/inherit.c#L73-L78)).

**Version notes:**
- PostgreSQL 12: Holds, but `pg_inherits` has no `inhdetachpending` column ([pg_inherits.h#FormData_pg_inherits](../raw/postgres-12/src/include/catalog/pg_inherits.h#L32-L37), [inherit.c#expand_inherited_rtentry](../raw/postgres-12/src/backend/optimizer/util/inherit.c#L79), [ddl.sgml#ddl-inherit](../raw/postgres-12/doc/src/sgml/ddl.sgml#L3119)).
- PostgreSQL 14: Holds, and `pg_inherits` gains `inhdetachpending`, which marks a partition that is being detached concurrently ([pg_inherits.h#FormData_pg_inherits](../raw/postgres-14/src/include/catalog/pg_inherits.h#L32-L38), [inherit.c#expand_inherited_rtentry](../raw/postgres-14/src/backend/optimizer/util/inherit.c#L84)).
- PostgreSQL 18: Holds ([pg_inherits.h#FormData_pg_inherits](../raw/postgres-18/src/include/catalog/pg_inherits.h#L32-L38), [inherit.c#expand_inherited_rtentry](../raw/postgres-18/src/backend/optimizer/util/inherit.c#L86)).
- PostgreSQL 19: Holds ([pg_inherits.h#FormData_pg_inherits](../raw/postgres-19/src/include/catalog/pg_inherits.h#L34-L40), [inherit.c#expand_inherited_rtentry](../raw/postgres-19/src/backend/optimizer/util/inherit.c#L88)).

Related: [Declarative partitioning](#declarative-partitioning), [Range table](#range-table), [Planner](#planner), [Partition pruning](#partition-pruning)

### Injection point

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An injection point is a named spot in server code where a test can attach a callback that runs when execution reaches it ([injection_point.c header](../raw/postgres-17/src/backend/utils/misc/injection_point.c#L3-L7)). The `INJECTION_POINT(name)` macro calls `InjectionPointRun` only in builds configured with `--enable-injection-points`. Otherwise it compiles to nothing ([injection_point.h](../raw/postgres-17/src/include/utils/injection_point.h#L14-L21), [installation.sgml#configure-option-enable-injection-points](../raw/postgres-17/doc/src/sgml/installation.sgml#L1657-L1670)). Real call sites sit in hot paths such as `heap_update` ([heapam.c:3448](../raw/postgres-17/src/backend/access/heap/heapam.c#L3448)). Tests attach callbacks there through the `src/test/modules/injection_points` module ([injection_points.control](../raw/postgres-17/src/test/modules/injection_points/injection_points.control#L1-L4)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The v12 test-module list has no `injection_points` module, so tests cannot attach callbacks to named points ([modules/Makefile:7-24](../raw/postgres-12/src/test/modules/Makefile#L7-L24)).
- PostgreSQL 14: Not present in PostgreSQL 14. The tree has no `injection_point.c`, and `src/test/modules` lists no `injection_points` module ([misc/Makefile](../raw/postgres-14/src/backend/utils/misc/Makefile#L17-L30), [modules/Makefile](../raw/postgres-14/src/test/modules/Makefile#L7-L33)).
- PostgreSQL 18: Differs in the macro signature. `INJECTION_POINT(name, arg)` now takes a second argument that is passed to the callback. 18 also adds `INJECTION_POINT_LOAD`, `INJECTION_POINT_CACHED` and `IS_INJECTION_POINT_ATTACHED`, and all of them still compile to nothing without `--enable-injection-points` ([injection_point.h](../raw/postgres-18/src/include/utils/injection_point.h#L14-L27), [installation.sgml#configure-option-enable-injection-points](../raw/postgres-18/doc/src/sgml/installation.sgml#L1689-L1702)). The `heap_update` call site is `INJECTION_POINT("heap_update-before-pin", NULL)` ([heapam.c:3423](../raw/postgres-18/src/backend/access/heap/heapam.c#L3423)).
- PostgreSQL 19: Holds. Injection points still run attached callbacks and compile to nothing without `--enable-injection-points`, and tests still use the `injection_points` module ([injection_point.c header](../raw/postgres-19/src/backend/utils/misc/injection_point.c#L3-L7), [injection_point.h](../raw/postgres-19/src/include/utils/injection_point.h#L27-L40), [installation.sgml#configure-option-enable-injection-points](../raw/postgres-19/doc/src/sgml/installation.sgml#L1684-L1696), [injection_points.control](../raw/postgres-19/src/test/modules/injection_points/injection_points.control#L1-L4)). The macro now takes two arguments, `INJECTION_POINT(name, arg)`, and v19 adds `INJECTION_POINT_LOAD`, `INJECTION_POINT_CACHED` and `IS_INJECTION_POINT_ATTACHED` ([injection_point.h](../raw/postgres-19/src/include/utils/injection_point.h#L30-L34), [heapam.c:3370](../raw/postgres-19/src/backend/access/heap/heapam.c#L3370)).

Related: [Isolation test](#isolation-test), [TAP test](#tap-test)

### Invalid index

**Aliases:** `indisvalid`, `indisready`, `indislive`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An invalid index has `pg_index.indisvalid = false`, so the planner will not use it for queries. Three `pg_index` flags track an index's state: `indislive` (it exists), `indisready` (inserts must maintain it) and `indisvalid` (queries may use it) ([pg_index.h:42-45](../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)). `get_relation_info()` skips any index without `indisvalid`. The executor still inserts into it when `indisready` is set ([plancat.c:256-267](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L256-L267)). A failed `CREATE INDEX CONCURRENTLY` leaves one behind. `psql` shows it as `INVALID`, and the documented fix is to drop it or run `REINDEX INDEX CONCURRENTLY` ([ref/create_index.sgml](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L646-L666)).

**Version notes:**
- PostgreSQL 12: Holds. `pg_index` has `indisvalid`, `indisready` and `indislive`, and `get_relation_info()` skips an invalid index ([pg_index.h:40-43](../raw/postgres-12/src/include/catalog/pg_index.h#L40-L43), [plancat.c:200-210](../raw/postgres-12/src/backend/optimizer/util/plancat.c#L200-L210)). A failed CIC leaves one; the fix is drop or `REINDEX INDEX CONCURRENTLY` ([ref/create_index.sgml](../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L596)).
- PostgreSQL 14: Holds. `pg_index` has `indisvalid`, `indisready` and `indislive`; `get_relation_info()` skips invalid indexes while the executor still inserts into ready ones; a failed concurrent build shows as `INVALID` and the fix is to drop it or `REINDEX INDEX CONCURRENTLY` ([pg_index.h:41-44](../raw/postgres-14/src/include/catalog/pg_index.h#L41-L44), [plancat.c:218-229](../raw/postgres-14/src/backend/optimizer/util/plancat.c#L218-L229), [ref/create_index.sgml](../raw/postgres-14/doc/src/sgml/ref/create_index.sgml#L640-L651)).
- PostgreSQL 18: Holds ([pg_index.h:42-45](../raw/postgres-18/src/include/catalog/pg_index.h#L42-L45), [plancat.c:259-270](../raw/postgres-18/src/backend/optimizer/util/plancat.c#L259-L270), [ref/create_index.sgml](../raw/postgres-18/doc/src/sgml/ref/create_index.sgml#L652-L672)).
- PostgreSQL 19: Holds. The `indislive`, `indisready` and `indisvalid` flags are unchanged, `get_relation_info()` still skips invalid indexes while the executor still inserts into ready ones, and a failed concurrent build still leaves an `INVALID` index ([pg_index.h:44-47](../raw/postgres-19/src/include/catalog/pg_index.h#L44-L47), [plancat.c:243-254](../raw/postgres-19/src/backend/optimizer/util/plancat.c#L243-L254), [ref/create_index.sgml](../raw/postgres-19/doc/src/sgml/ref/create_index.sgml#L650-L673)).

Related: [CONCURRENTLY](#concurrently), [pg_index](#pg_index), [REINDEX](#reindex), [Planner](#planner)

### Invalidation message

**Aliases:** cache invalidation, shared invalidation, sinval, SI message, `SharedInvalidationMessage`, `CacheInvalidateRelcache`, `AcceptInvalidationMessages`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An invalidation message tells backends that something they may have cached, such as a catalog row or a relcache entry, is now stale. Each backend keeps private caches of catalog data. When a transaction changes a catalog row, `inval.c` queues invalidation events, applies them locally at the next command boundary, and broadcasts them to other backends at commit through the shared invalidation queue ([inval.c header](../raw/postgres-17/src/backend/utils/cache/inval.c#L4-L35)). `SharedInvalidationMessage` is a union of message types: one catcache tuple, a whole catalog, one relcache entry, all relcache entries, an smgr entry, the relation map, or a snapshot ([sinval.h](../raw/postgres-17/src/include/storage/sinval.h#L20-L56), [sinval.h#SharedInvalidationMessage](../raw/postgres-17/src/include/storage/sinval.h#L113-L122)). Other backends read the queue in `AcceptInvalidationMessages()`. One place that call happens is when a backend first takes a lock on a relation, which is why a lock is enough to guarantee a fresh relcache entry ([lmgr.c#LockRelationOid](../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L117-L137)).

**Version notes:**
- PostgreSQL 12: Holds, with the same seven message types ([sinval.h#SharedInvalidationMessage](../raw/postgres-12/src/include/storage/sinval.h#L113-L122), [inval.c#AcceptInvalidationMessages](../raw/postgres-12/src/backend/utils/cache/inval.c#L681)).
- PostgreSQL 14: Holds ([sinval.h#SharedInvalidationMessage](../raw/postgres-14/src/include/storage/sinval.h#L113-L122), [inval.c#AcceptInvalidationMessages](../raw/postgres-14/src/backend/utils/cache/inval.c#L721)).
- PostgreSQL 18: Adds an eighth type that invalidates a logical replication `RelationSyncCache` entry ([sinval.h](../raw/postgres-18/src/include/storage/sinval.h#L21-L31), [sinval.h#SharedInvalidationMessage](../raw/postgres-18/src/include/storage/sinval.h#L124-L134)).
- PostgreSQL 19: As in 18 ([sinval.h#SharedInvalidationMessage](../raw/postgres-19/src/include/storage/sinval.h#L124-L134), [inval.c#AcceptInvalidationMessages](../raw/postgres-19/src/backend/utils/cache/inval.c#L930)).

Related: [Relcache](#relcache), [Syscache](#syscache), [Custom and generic plan](#custom-and-generic-plan), [Heavyweight lock](#heavyweight-lock)

### io_combine_limit

**Aliases:** I/O combining, vectored I/O size, `io_max_combine_limit`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`io_combine_limit` is the largest number of adjacent blocks PostgreSQL merges into one read or write system call. It first appears in PostgreSQL 17, with context `user` and unit blocks ([guc_tables.c:3138-3149](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3138-L3149)). The default is 128 kB worth of blocks, 16 at the default 8 kB [BLCKSZ](#blcksz). The maximum is `PG_IOV_MAX`, at most 32 blocks ([bufmgr.h:166-169](../raw/postgres-17/src/include/storage/bufmgr.h#L166-L169), [pg_iovec.h:43](../raw/postgres-17/src/include/port/pg_iovec.h#L43)). A [read stream](#read-stream) builds reads of up to this many blocks by merging neighbouring block requests ([read_stream.c header](../raw/postgres-17/src/backend/storage/aio/read_stream.c#L3-L20)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. `guc.c` defines only `effective_io_concurrency` among these I/O settings ([guc.c:2759-2772](../raw/postgres-12/src/backend/utils/misc/guc.c#L2759-L2772)).
- PostgreSQL 14: Not present in PostgreSQL 14. `guc.c` has `effective_io_concurrency` and `maintenance_io_concurrency` but no combine limit ([guc.c:3045-3076](../raw/postgres-14/src/backend/utils/misc/guc.c#L3045-L3076)).
- PostgreSQL 18: `PG_IOV_MAX` rises to at most 128 blocks ([pg_iovec.h:47](../raw/postgres-18/src/include/port/pg_iovec.h#L47)). A new `io_max_combine_limit`, context `postmaster` (restart), clamps the per-session `io_combine_limit` (context `user`) ([guc_tables.c:3278-3305](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3278-L3305), [bufmgr.c:164-172](../raw/postgres-18/src/backend/storage/buffer/bufmgr.c#L164-L172)).
- PostgreSQL 19: Same pair of settings as 18, declared in `guc_parameters.dat` ([guc_parameters.dat:1364-1382](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1364-L1382), [bufmgr.h:175-176](../raw/postgres-19/src/include/storage/bufmgr.h#L175-L176)).

Related: [effective_io_concurrency](#effective_io_concurrency), [Read stream](#read-stream), [Asynchronous I/O](#asynchronous-io), [BLCKSZ](#blcksz)

### Isolation test

**Aliases:** isolation tester, spec test. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An isolation test checks concurrent behavior by running several interleaved sessions against one server ([isolation/README](../raw/postgres-17/src/test/isolation/README#L3-L12)). A spec file under `specs/` defines SQL steps, groups them into sessions, and lists the permutations (step orders) to run. Each session gets its own connection ([isolation/README](../raw/postgres-17/src/test/isolation/README#L56-L63), [isolation/README](../raw/postgres-17/src/test/isolation/README#L83-L98)). The ordinary `pg_regress` driver cannot run these tests, because it uses one connection at a time ([isolation/README](../raw/postgres-17/src/test/isolation/README#L6-L9)).

**Version notes:**
- PostgreSQL 12: Holds. The README describes concurrent sessions, one connection each, and says `pg_regress` cannot run them ([isolation/README](../raw/postgres-12/src/test/isolation/README#L3-L12), [isolation/README](../raw/postgres-12/src/test/isolation/README#L78-L84)).
- PostgreSQL 14: Holds. The README describes spec files with setup, sessions, steps and permutations, run over multiple connections that `pg_regress` cannot manage ([isolation/README](../raw/postgres-14/src/test/isolation/README#L6-L11), [isolation/README](../raw/postgres-14/src/test/isolation/README#L60-L73)).
- PostgreSQL 18: Holds ([isolation/README](../raw/postgres-18/src/test/isolation/README#L3-L12), [isolation/README](../raw/postgres-18/src/test/isolation/README#L56-L63)).
- PostgreSQL 19: Holds. Specs under `specs/` still define steps, sessions and permutations, each session still gets its own connection, and `pg_regress` still cannot run them ([isolation/README](../raw/postgres-19/src/test/isolation/README#L3-L12), [isolation/README](../raw/postgres-19/src/test/isolation/README#L56-L63), [isolation/README](../raw/postgres-19/src/test/isolation/README#L83-L98)).

Related: [Regression test](#regression-test), [TAP test](#tap-test), [Injection point](#injection-point)

### JIT compilation

**Aliases:** JIT, just-in-time compilation, LLVM, `jit`, `jit_above_cost`, `jit_provider`, `PGJIT_PERFORM`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

JIT compilation turns generic, interpreted work, such as evaluating a `WHERE` expression or deforming tuples of one table, into native machine code at query run time ([jit/README](../raw/postgres-17/src/backend/jit/README#L1-L22)). The core JIT interface lives in `src/backend/jit/`, and the LLVM-based provider lives in `jit/llvm/`. The planner decides per query. It sets `PGJIT_PERFORM` in `PlannedStmt.jitFlags` when `jit` is on and the plan's total cost exceeds `jit_above_cost`. Higher cost thresholds add optimization and inlining ([planner.c#standard_planner](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L564-L586)). `jit` defaults to on and `jit_above_cost` to 100000. Both have context `user`, so they apply per session or transaction ([guc_tables.c#jit](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1925-L1932), [guc_tables.c#jit_above_cost](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3754-L3761)). `jit_provider` has context `postmaster`, so changing it needs a restart ([guc_tables.c#jit_provider](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4678)).

**Version notes:**
- PostgreSQL 12: Holds, with `jit` on by default and the same contexts ([guc.c#jit](../raw/postgres-12/src/backend/utils/misc/guc.c#L1869-L1876), [guc.c#jit_above_cost](../raw/postgres-12/src/backend/utils/misc/guc.c#L3285), [guc.c#jit_provider](../raw/postgres-12/src/backend/utils/misc/guc.c#L4192), [planner.c:539](../raw/postgres-12/src/backend/optimizer/plan/planner.c#L539)).
- PostgreSQL 14: Holds, with `jit` on by default ([guc.c#jit](../raw/postgres-14/src/backend/utils/misc/guc.c#L2033-L2040), [guc.c#jit_above_cost](../raw/postgres-14/src/backend/utils/misc/guc.c#L3646), [guc.c#jit_provider](../raw/postgres-14/src/backend/utils/misc/guc.c#L4585), [planner.c:534](../raw/postgres-14/src/backend/optimizer/plan/planner.c#L534)).
- PostgreSQL 18: Holds, with `jit` on by default ([guc_tables.c#jit](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2024-L2031), [guc_tables.c#jit_above_cost](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3961), [guc_tables.c#jit_provider](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L4918), [planner.c:603](../raw/postgres-18/src/backend/optimizer/plan/planner.c#L603)).
- PostgreSQL 19: `jit` now defaults to off. Contexts and the planner rule are unchanged ([guc_parameters.dat#jit](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1444-L1449), [guc_parameters.dat#jit_above_cost](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1451), [guc_parameters.dat#jit_provider](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1515), [planner.c:702](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L702)).

Related: [Executor](#executor), [PlannedStmt](#plannedstmt), [Cost](#cost), [GUC context](#guc-context)

### Leakproof function

**Aliases:** `LEAKPROOF`, `proleakproof`, `contain_leaked_vars`, `RestrictInfo.leakproof`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A leakproof function has no side effects and reveals nothing about its arguments except through its return value. A function that raises an error for some inputs, or puts argument values in error messages, is not leakproof ([create_function.sgml#LEAKPROOF](../raw/postgres-17/doc/src/sgml/ref/create_function.sgml#L357-L372)). The flag is `pg_proc.proleakproof` ([pg_proc.h:65](../raw/postgres-17/src/include/catalog/pg_proc.h#L65)). It matters for row-level security and `security_barrier` views. The planner must apply their security conditions before any user condition that passes row data to a non-leakproof function. `contain_leaked_vars()` finds such calls, and `make_restrictinfo_internal()` records the result in `RestrictInfo.leakproof` for any condition that security quals could delay ([clauses.c#contain_leaked_vars](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L1258-L1273), [restrictinfo.c#make_restrictinfo_internal](../raw/postgres-17/src/backend/optimizer/util/restrictinfo.c#L138-L146)). A leakproof function can see rows that a policy would later filter out, so only a superuser may mark a function `LEAKPROOF` ([functioncmds.c#CreateFunction](../raw/postgres-17/src/backend/commands/functioncmds.c#L1127-L1134)).

**Version notes:**
- PostgreSQL 12: Holds ([pg_proc.h:66](../raw/postgres-12/src/include/catalog/pg_proc.h#L66), [restrictinfo.c:130](../raw/postgres-12/src/backend/optimizer/util/restrictinfo.c#L130), [functioncmds.c:1030](../raw/postgres-12/src/backend/commands/functioncmds.c#L1030)).
- PostgreSQL 14: Holds ([pg_proc.h:65](../raw/postgres-14/src/include/catalog/pg_proc.h#L65), [restrictinfo.c:136](../raw/postgres-14/src/backend/optimizer/util/restrictinfo.c#L136), [functioncmds.c:1165](../raw/postgres-14/src/backend/commands/functioncmds.c#L1165)).
- PostgreSQL 18: Holds ([pg_proc.h:65](../raw/postgres-18/src/include/catalog/pg_proc.h#L65), [restrictinfo.c:135](../raw/postgres-18/src/backend/optimizer/util/restrictinfo.c#L135), [functioncmds.c:1150](../raw/postgres-18/src/backend/commands/functioncmds.c#L1150)).
- PostgreSQL 19: Holds ([pg_proc.h:67](../raw/postgres-19/src/include/catalog/pg_proc.h#L67), [restrictinfo.c:135](../raw/postgres-19/src/backend/optimizer/util/restrictinfo.c#L135), [functioncmds.c:1165](../raw/postgres-19/src/backend/commands/functioncmds.c#L1165)).

Related: [Row-level security](#row-level-security), [Security barrier](#security-barrier), [Function volatility](#function-volatility), [Qual](#qual)

### Line pointer

**Aliases:** item identifier, `ItemIdData`, `lp_flags`, `LP_UNUSED`, `LP_NORMAL`, `LP_REDIRECT`, `LP_DEAD`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A line pointer is a 4-byte slot in the array at the start of a page. It records where a tuple sits on the page (`lp_off`), how long it is (`lp_len`) and what state it is in (`lp_flags`) ([itemid.h#ItemIdData](../raw/postgres-17/src/include/storage/itemid.h#L17-L30)). A TID names a line pointer, not a byte offset, so a tuple can move inside its page without changing its TID ([bufpage.h#NOTES](../raw/postgres-17/src/include/storage/bufpage.h#L56-L64)). The four states are `LP_UNUSED` (free for reuse), `LP_NORMAL` (points to a tuple), `LP_REDIRECT` (a HOT link to another slot, with no storage) and `LP_DEAD` (dead; it may or may not still have storage) ([itemid.h#LP_UNUSED](../raw/postgres-17/src/include/storage/itemid.h#L34-L41)). VACUUM sets an `LP_DEAD` slot to `LP_UNUSED` only in its second heap pass, after the index entries that point to it are gone. No index entry may ever point to an `LP_UNUSED` slot ([vacuumlazy.c#lazy_vacuum](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L793-L800)).

**Version notes:**
- PostgreSQL 12: Holds. `ItemIdData` has `lp_off`, `lp_flags` and `lp_len`, with the same four states; TIDs name line pointers, not byte offsets ([itemid.h#ItemIdData](../raw/postgres-12/src/include/storage/itemid.h#L25-L41), [bufpage.h#NOTES](../raw/postgres-12/src/include/storage/bufpage.h#L54-L64)). In 12 the second heap pass is `lazy_vacuum_heap()`, which sets dead slots unused after index cleanup ([vacuumlazy.c#lazy_vacuum_heap](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1515-L1526), [vacuumlazy.c:1615](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1615)).
- PostgreSQL 14: Holds. `ItemIdData` has `lp_off`, `lp_flags` and `lp_len`, with the same four states, and TIDs name line pointers rather than byte offsets ([itemid.h#ItemIdData](../raw/postgres-14/src/include/storage/itemid.h#L25-L30), [itemid.h#LP_UNUSED](../raw/postgres-14/src/include/storage/itemid.h#L38-L41), [bufpage.h#NOTES](../raw/postgres-14/src/include/storage/bufpage.h#L56-L64)). VACUUM's second heap pass, `lazy_vacuum_heap_rel()`, turns `LP_DEAD` into `LP_UNUSED` ([vacuumlazy.c#lazy_vacuum_heap_rel](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L2296-L2301)).
- PostgreSQL 18: Holds ([itemid.h#ItemIdData](../raw/postgres-18/src/include/storage/itemid.h#L17-L30), [itemid.h#LP_UNUSED](../raw/postgres-18/src/include/storage/itemid.h#L34-L41), [vacuumlazy.c#lazy_vacuum](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L1177-L1184)).
- PostgreSQL 19: Holds. `ItemIdData` still holds `lp_off`, `lp_flags` and `lp_len` with the same four states, a TID still names a line pointer, and VACUUM still sets `LP_DEAD` to `LP_UNUSED` only in its final heap pass so no index entry points to an unused slot ([itemid.h#ItemIdData](../raw/postgres-19/src/include/storage/itemid.h#L17-L41), [bufpage.h#NOTES](../raw/postgres-19/src/include/storage/bufpage.h#L56-L70), [vacuumlazy.c#lazy_scan_heap](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L1255-L1262), [vacuumlazy.c#lazy_vacuum](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L2370-L2383)).

Related: [HOT](#hot), [Page](#page), [Pruning](#pruning), [TID](#tid)

### Lock mode

**Aliases:** table lock modes, `AccessShareLock`, `RowShareLock`, `RowExclusiveLock`, `ShareLock`, `ShareRowExclusiveLock`, `ExclusiveLock`, `LOCKMODE`, lock conflict table. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A lock mode is one of the eight strengths of [heavyweight lock](#heavyweight-lock) a transaction can hold on a table. They are numbered 1 to 8, from `AccessShareLock` (plain `SELECT`) to [AccessExclusiveLock](#accessexclusivelock). The comments in `lockdefs.h` name typical users: `RowExclusiveLock` for `INSERT`/`UPDATE`/`DELETE`, [ShareUpdateExclusiveLock](#shareupdateexclusivelock) for plain VACUUM and `CREATE INDEX CONCURRENTLY`, and `ShareLock` for plain `CREATE INDEX` ([lockdefs.h:33-48](../raw/postgres-17/src/include/storage/lockdefs.h#L33-L48)). All eight are table-level locks, even the ones whose names contain "row". What distinguishes them is only which other modes they conflict with ([mvcc.sgml#locking-tables](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L890-L900)). The `LockConflicts[]` array encodes that matrix as one bitmask per mode. For example, `AccessShareLock` conflicts only with `AccessExclusiveLock`, while `ShareLock` conflicts with `RowExclusiveLock`, so a plain `CREATE INDEX` blocks writes but not reads ([lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)).

**Version notes:**
- PostgreSQL 12: Holds; the same eight modes and conflict table ([lockdefs.h:36-45](../raw/postgres-12/src/include/storage/lockdefs.h#L36-L45), [lock.c#LockConflicts](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65)).
- PostgreSQL 14: Holds ([lockdefs.h:36-45](../raw/postgres-14/src/include/storage/lockdefs.h#L36-L45), [lock.c#LockConflicts](../raw/postgres-14/src/backend/storage/lmgr/lock.c#L65)).
- PostgreSQL 18: Holds ([lockdefs.h:36-45](../raw/postgres-18/src/include/storage/lockdefs.h#L36-L45), [lock.c#LockConflicts](../raw/postgres-18/src/backend/storage/lmgr/lock.c#L65)).
- PostgreSQL 19: Holds ([lockdefs.h:36-45](../raw/postgres-19/src/include/storage/lockdefs.h#L36-L45), [lock.c#LockConflicts](../raw/postgres-19/src/backend/storage/lmgr/lock.c#L68)).

Related: [Heavyweight lock](#heavyweight-lock), [AccessExclusiveLock](#accessexclusivelock), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [Lock queue](#lock-queue), [Deadlock](#deadlock)

### Lock queue

**Aliases:** lock wait queue, lock waiters, `waitProcs`, `pg_locks.granted`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The lock queue is the ordered list of processes waiting for a [heavyweight lock](#heavyweight-lock) on one object. It is kept in the lock's `waitProcs` list ([lock.h#LOCK](../raw/postgres-17/src/include/storage/lock.h#L297-L317)). A new request is granted at once only if it conflicts with neither the modes already held nor the requests already waiting. Otherwise the process joins the end of the queue, except that it may go ahead of waiters that conflict with locks it already holds ([lmgr/README:364-381](../raw/postgres-17/src/backend/storage/lmgr/README#L364-L381)). That rule is why a single queued strong request, such as an `AccessExclusiveLock` from `ALTER TABLE`, makes every later `SELECT` on the table wait behind it, even though the `SELECT`s do not conflict with each other ([lmgr/README:364-373](../raw/postgres-17/src/backend/storage/lmgr/README#L364-L373)). The [lock_timeout](#statement_timeout-and-lock_timeout) setting bounds how long a statement can wait in the queue ([config.sgml#guc-lock-timeout](../raw/postgres-17/doc/src/sgml/config.sgml#L9534-L9565)). In the `pg_locks` view, a waiting request shows `granted = false` ([system-views.sgml:1523-1527](../raw/postgres-17/doc/src/sgml/system-views.sgml#L1523-L1527)).

**Version notes:**
- PostgreSQL 12: Holds; `waitProcs` is a `PROC_QUEUE` ([lock.h:297](../raw/postgres-12/src/include/storage/lock.h#L297), [lmgr/README:364-381](../raw/postgres-12/src/backend/storage/lmgr/README#L364-L381), [catalogs.sgml:9239](../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9239)).
- PostgreSQL 14: Holds ([lock.h:309](../raw/postgres-14/src/include/storage/lock.h#L309), [lmgr/README:364-381](../raw/postgres-14/src/backend/storage/lmgr/README#L364-L381)).
- PostgreSQL 18: Holds; `waitProcs` is a `dclist_head`, as in 17 ([lock.h:318](../raw/postgres-18/src/include/storage/lock.h#L318), [lmgr/README:364-381](../raw/postgres-18/src/backend/storage/lmgr/README#L364-L381), [system-views.sgml:1863](../raw/postgres-18/doc/src/sgml/system-views.sgml#L1863)).
- PostgreSQL 19: Holds ([lock.h:148](../raw/postgres-19/src/include/storage/lock.h#L148), [lmgr/README:364-381](../raw/postgres-19/src/backend/storage/lmgr/README#L364-L381), [system-views.sgml:1970](../raw/postgres-19/doc/src/sgml/system-views.sgml#L1970)).

Related: [Heavyweight lock](#heavyweight-lock), [Lock mode](#lock-mode), [Deadlock](#deadlock), [statement_timeout and lock_timeout](#statement_timeout-and-lock_timeout)

### Logical decoding

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Logical decoding reads WAL records and turns them into a stream of changes that a consumer can read ([decode.c header](../raw/postgres-17/src/backend/replication/logical/decode.c#L3-L8), [logical.c header](../raw/postgres-17/src/backend/replication/logical/logical.c#L10-L18)). `LogicalDecodingProcessRecord` feeds each WAL record through the decoder ([decode.c#LogicalDecodingProcessRecord](../raw/postgres-17/src/backend/replication/logical/decode.c#L87-L88)). An output plugin then formats the changes through the callbacks in `OutputPluginCallbacks`. `pgoutput` is the plugin that logical replication uses ([output_plugin.h#OutputPluginCallbacks](../raw/postgres-17/src/include/replication/output_plugin.h#L216-L243), [pgoutput.c header](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L3-L4)). `CheckLogicalDecodingRequirements` refuses to decode unless `wal_level` is `logical` and the process is connected to a database ([logical.c#CheckLogicalDecodingRequirements](../raw/postgres-17/src/backend/replication/logical/logical.c#L116-L133)).

**Version notes:**
- PostgreSQL 12: Differs. The decoder, `LogicalDecodingProcessRecord`, `OutputPluginCallbacks` and `pgoutput` exist ([decode.c header](../raw/postgres-12/src/backend/replication/logical/decode.c#L3-L8), [decode.c#LogicalDecodingProcessRecord](../raw/postgres-12/src/backend/replication/logical/decode.c#L97), [output_plugin.h#OutputPluginCallbacks](../raw/postgres-12/src/include/replication/output_plugin.h#L105-L115), [pgoutput.c header](../raw/postgres-12/src/backend/replication/pgoutput/pgoutput.c#L3-L4)). `CheckLogicalDecodingRequirements` needs `wal_level = logical` and a database, and in 12 it also refuses to run during recovery, so decoding on a standby is impossible ([logical.c#CheckLogicalDecodingRequirements](../raw/postgres-12/src/backend/replication/logical/logical.c#L78-L113)).
- PostgreSQL 14: Differs on standbys. `decode.c` and `LogicalDecodingProcessRecord` feed changes to an output plugin through `OutputPluginCallbacks`, and `pgoutput` is the replication plugin ([decode.c header](../raw/postgres-14/src/backend/replication/logical/decode.c#L3-L8), [decode.c#LogicalDecodingProcessRecord](../raw/postgres-14/src/backend/replication/logical/decode.c#L106), [output_plugin.h#OutputPluginCallbacks](../raw/postgres-14/src/include/replication/output_plugin.h#L214), [pgoutput.c header](../raw/postgres-14/src/backend/replication/pgoutput/pgoutput.c#L3-L4)). `CheckLogicalDecodingRequirements` requires `wal_level` `logical` and a database connection, and in 14 it also refuses to run during recovery, so a standby cannot decode ([logical.c#CheckLogicalDecodingRequirements](../raw/postgres-14/src/backend/replication/logical/logical.c#L117-L144)).
- PostgreSQL 18: Holds ([decode.c header](../raw/postgres-18/src/backend/replication/logical/decode.c#L3-L8), [output_plugin.h#OutputPluginCallbacks](../raw/postgres-18/src/include/replication/output_plugin.h#L216-L243), [logical.c#CheckLogicalDecodingRequirements](../raw/postgres-18/src/backend/replication/logical/logical.c#L116-L133)).
- PostgreSQL 19: Differs on the `wal_level` requirement. `LogicalDecodingProcessRecord()` still feeds records to the decoder, output plugins still use `OutputPluginCallbacks`, and `pgoutput` is still the logical replication plugin ([decode.c header](../raw/postgres-19/src/backend/replication/logical/decode.c#L3-L8), [logical.c header](../raw/postgres-19/src/backend/replication/logical/logical.c#L10-L18), [decode.c#LogicalDecodingProcessRecord](../raw/postgres-19/src/backend/replication/logical/decode.c#L89), [output_plugin.h#OutputPluginCallbacks](../raw/postgres-19/src/include/replication/output_plugin.h#L216-L243), [pgoutput.c header](../raw/postgres-19/src/backend/replication/pgoutput/pgoutput.c#L3-L4)). `CheckLogicalDecodingRequirements()` no longer demands `wal_level = logical`. It needs `wal_level >= replica` and a database connection. With `replica`, logical decoding turns on right after the first logical slot is created, and the checkpointer turns it off, lazily, once no valid logical slot remains. The read-only `effective_wal_level` (`PGC_INTERNAL`) shows the level in force ([logical.c#CheckLogicalDecodingRequirements](../raw/postgres-19/src/backend/replication/logical/logical.c#L117-L139), [logicalctl.c header](../raw/postgres-19/src/backend/replication/logical/logicalctl.c#L3-L37), [guc_parameters.dat#effective_wal_level](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L857-L863)). `wal_level` is `PGC_POSTMASTER` (restart) ([guc_parameters.dat#wal_level](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3505-L3510)).

Related: [WAL](#wal), [Logical replication](#logical-replication), [Replication slot](#replication-slot), [Replication origin](#replication-origin)

### Logical replication

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Logical replication copies table rows and their changes by replication identity, not by block address as physical replication does ([logical-replication.sgml](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L6-L13)). It uses a publish-and-subscribe model. A subscriber first copies a snapshot of the data, then applies later changes in the publisher's order ([logical-replication.sgml](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L15-L33)). In source, the publisher side is logical decoding plus the `pgoutput` plugin. On the subscriber, the launcher starts apply and tablesync workers as dynamic background workers ([pgoutput.c header](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L3-L4), [launcher.c#logicalrep_worker_launch](../raw/postgres-17/src/backend/replication/logical/launcher.c#L470-L503)).

**Version notes:**
- PostgreSQL 12: Holds. It replicates by replication identity with publish and subscribe, starting from a snapshot ([logical-replication.sgml](../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L6-L33)). `pgoutput` is the publisher plugin, and the launcher starts apply and table-sync workers as background workers ([pgoutput.c header](../raw/postgres-12/src/backend/replication/pgoutput/pgoutput.c#L3-L4), [launcher.c#logicalrep_worker_launch](../raw/postgres-12/src/backend/replication/logical/launcher.c#L294-L295), [launcher.c:424-431](../raw/postgres-12/src/backend/replication/logical/launcher.c#L424-L431)).
- PostgreSQL 14: Holds. The docs describe replication by replication identity, publish/subscribe, and an initial snapshot copy, and the launcher starts workers as dynamic background workers ([logical-replication.sgml](../raw/postgres-14/doc/src/sgml/logical-replication.sgml#L8-L31), [launcher.c#logicalrep_worker_launch](../raw/postgres-14/src/backend/replication/logical/launcher.c#L393-L405)). In 14 apply and tablesync workers share the `ApplyWorkerMain` entry point.
- PostgreSQL 18: Holds. The docs reword the intro but keep the same model ([logical-replication.sgml](../raw/postgres-18/doc/src/sgml/logical-replication.sgml#L6-L28), [launcher.c#logicalrep_worker_launch](../raw/postgres-18/src/backend/replication/logical/launcher.c#L467-L500)).
- PostgreSQL 19: Holds for the replication-identity basis, the publish/subscribe model, the initial copy and the ordered apply ([logical-replication.sgml](../raw/postgres-19/doc/src/sgml/logical-replication.sgml#L6-L34)). In v19, publications can also carry sequences (`FOR ALL SEQUENCES`), and the launcher can start a sequencesync worker alongside apply and tablesync workers ([logical-replication.sgml#logical-replication-publication](../raw/postgres-19/doc/src/sgml/logical-replication.sgml#L104-L120), [launcher.c#logicalrep_worker_launch](../raw/postgres-19/src/backend/replication/logical/launcher.c#L509-L542)).

Related: [Publication](#publication), [Subscription](#subscription), [Apply worker](#apply-worker), [Logical decoding](#logical-decoding), [Replication slot](#replication-slot)

### LSN

**Aliases:** log sequence number, `XLogRecPtr`, `pg_lsn`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A 64-bit byte position in the WAL stream ([xlogdefs.h#XLogRecPtr](../raw/postgres-17/src/include/access/xlogdefs.h#L17-L21)). Every data page stores in `pd_lsn` the LSN just past the last WAL record that changed it ([bufpage.h#PageHeaderData](../raw/postgres-17/src/include/storage/bufpage.h#L155-L158)). The buffer manager must flush WAL up to a page's LSN before writing that page. This is the "write the log before the data" rule ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L409-L415), [xloginsert.c#XLogInsert](../raw/postgres-17/src/backend/access/transam/xloginsert.c#L466-L472)). Code prints an LSN as `%X/%X` with `LSN_FORMAT_ARGS()` ([xlogdefs.h:42-44](../raw/postgres-17/src/include/access/xlogdefs.h#L42-L44)).

**Version notes:**
- PostgreSQL 12: Holds, except for printing. `XLogRecPtr` is 64 bits, `pd_lsn` stores the page LSN, and WAL must be flushed up to it before the page is written ([xlogdefs.h#XLogRecPtr](../raw/postgres-12/src/include/access/xlogdefs.h#L17-L21), [bufpage.h#PageHeaderData](../raw/postgres-12/src/include/storage/bufpage.h#L154), [transam/README](../raw/postgres-12/src/backend/access/transam/README#L402-L409)). 12 has no `LSN_FORMAT_ARGS()`; code prints `%X/%X` with two explicit `uint32` casts ([xlog.c:1185-1186](../raw/postgres-12/src/backend/access/transam/xlog.c#L1185-L1186)).
- PostgreSQL 14: Holds. `XLogRecPtr` is a 64-bit position, pages store `pd_lsn`, WAL is flushed up to the page LSN before the page is written, and `LSN_FORMAT_ARGS()` exists ([xlogdefs.h#XLogRecPtr](../raw/postgres-14/src/include/access/xlogdefs.h#L21), [bufpage.h#PageHeaderData](../raw/postgres-14/src/include/storage/bufpage.h#L154), [transam/README](../raw/postgres-14/src/backend/access/transam/README#L409-L415), [xlogdefs.h:42-44](../raw/postgres-14/src/include/access/xlogdefs.h#L42-L44)).
- PostgreSQL 18: Holds. `LSN_FORMAT_ARGS()` and the `%X/%X` form are unchanged ([xlogdefs.h#XLogRecPtr](../raw/postgres-18/src/include/access/xlogdefs.h#L17-L21), [bufpage.h#PageHeaderData](../raw/postgres-18/src/include/storage/bufpage.h#L159-L162), [xlogdefs.h:42-44](../raw/postgres-18/src/include/access/xlogdefs.h#L42-L44)).
- PostgreSQL 19: Holds. `XLogRecPtr` is still a 64-bit WAL position, every page still stores `pd_lsn`, and WAL must still be flushed up to a page's LSN before the page is written ([xlogdefs.h#XLogRecPtr](../raw/postgres-19/src/include/access/xlogdefs.h#L21), [bufpage.h#PageHeaderData](../raw/postgres-19/src/include/storage/bufpage.h#L184-L197), [transam/README](../raw/postgres-19/src/backend/access/transam/README#L407-L416), [xloginsert.c#XLogInsert](../raw/postgres-19/src/backend/access/transam/xloginsert.c#L470-L482)). Two v19 details differ: `pd_lsn` has type `PageXLogRecPtr`, and the print format is `%X/%08X`, so the low half is zero-padded to eight digits ([bufpage.h#PageHeaderData](../raw/postgres-19/src/include/storage/bufpage.h#L187), [xlogdefs.h:39-47](../raw/postgres-19/src/include/access/xlogdefs.h#L39-L47)).

Related: [WAL](#wal), [Page](#page), [Full-page image](#full-page-image)

### LWLock

**Aliases:** lightweight lock. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A short-term lock that protects a shared-memory data structure, in shared or exclusive mode. It has no deadlock detection and no timeout. It is released automatically on error, and waiters are served in arrival order ([lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L20-L30)). The struct holds a tranche ID, an atomic state word and a waiter list ([lwlock.h#LWLock](../raw/postgres-17/src/include/storage/lwlock.h#L41-L50)). Code takes one with `LWLockAcquire()` ([lwlock.c#LWLockAcquire](../raw/postgres-17/src/backend/storage/lmgr/lwlock.c#L1170)). One example is `XidGenLock`, taken while the transaction ID limits are updated ([varsup.c:443](../raw/postgres-17/src/backend/access/transam/varsup.c#L443)). Unlike a heavyweight lock, it has no deadlock detection and is not held to transaction end ([lmgr/README](../raw/postgres-17/src/backend/storage/lmgr/README#L20-L35)).

**Version notes:**
- PostgreSQL 12: Holds. LWLocks are shared or exclusive, have no deadlock detection or timeout, are released on error, and serve waiters in arrival order ([lmgr/README](../raw/postgres-12/src/backend/storage/lmgr/README#L20-L30)). The struct has a tranche, an atomic state and a waiter list ([lwlock.h#LWLock](../raw/postgres-12/src/include/storage/lwlock.h#L32-L40)). `LWLockAcquire()` takes one, for example `XidGenLock` in `varsup.c` ([lwlock.c#LWLockAcquire](../raw/postgres-12/src/backend/storage/lmgr/lwlock.c#L1123), [varsup.c:76](../raw/postgres-12/src/backend/access/transam/varsup.c#L76)).
- PostgreSQL 14: Holds. The README describes the same shared/exclusive lock with no deadlock detection, the struct holds a tranche, atomic state and waiter list, and `XidGenLock` is taken with `LWLockAcquire()` ([lmgr/README](../raw/postgres-14/src/backend/storage/lmgr/README#L20-L30), [lwlock.h#LWLock](../raw/postgres-14/src/include/storage/lwlock.h#L39-L48), [lwlock.c#LWLockAcquire](../raw/postgres-14/src/backend/storage/lmgr/lwlock.c#L1206), [varsup.c:78](../raw/postgres-14/src/backend/access/transam/varsup.c#L78)).
- PostgreSQL 18: Holds ([lmgr/README](../raw/postgres-18/src/backend/storage/lmgr/README#L20-L30), [lwlock.h#LWLock](../raw/postgres-18/src/include/storage/lwlock.h#L41-L50), [varsup.c:443](../raw/postgres-18/src/backend/access/transam/varsup.c#L443)).
- PostgreSQL 19: Holds. LWLocks still offer shared and exclusive modes with no deadlock detection and no timeout, the struct still holds a tranche, an atomic state and a waiter list, and `XidGenLock` is still taken in `SetTransactionIdLimit()` ([lmgr/README](../raw/postgres-19/src/backend/storage/lmgr/README#L18-L30), [lwlock.h#LWLock](../raw/postgres-19/src/include/storage/lwlock.h#L41-L50), [lwlock.c#LWLockAcquire](../raw/postgres-19/src/backend/storage/lmgr/lwlock.c#L1150), [varsup.c:438](../raw/postgres-19/src/backend/access/transam/varsup.c#L438)). One boundary moved: v19 buffer content locks are no longer LWLocks ([buf_internals.h#BufferDesc](../raw/postgres-19/src/include/storage/buf_internals.h#L303-L310)).

Related: [Heavyweight lock](#heavyweight-lock), [Wait event](#wait-event)

### maintenance_work_mem

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`maintenance_work_mem` caps the memory used by maintenance operations such as `VACUUM` and `CREATE INDEX`. Its context is `user`, so a session or transaction can `SET` it with no reload or restart. Its default is 65536 kB (64 MB) [guc_tables.c#maintenance_work_mem](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2474). A B-tree build sizes its sort with it instead of `work_mem` [nbtsort.c:370-375](../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L370-L375). Lazy `VACUUM` uses it to bound the store of dead TIDs. When that store fills, `VACUUM` must pass over every index before it can continue. An autovacuum worker uses `autovacuum_work_mem` instead when that setting is not -1 [vacuumlazy.c:12-16](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L12-L16) [vacuumlazy.c#dead_items_alloc](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2823-L2828).

**Version notes:**
- PostgreSQL 12: Holds, with a different dead-TID store. It is `PGC_USERSET` (session scope), default 65536 kB ([guc.c#maintenance_work_mem](../raw/postgres-12/src/backend/utils/misc/guc.c#L2244-L2252)). A B-tree build sizes its sort with it ([nbtsort.c:387-390](../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L387-L390)). In 12, lazy VACUUM stores dead TIDs in a flat `ItemPointerData` array sized from it (or from `autovacuum_work_mem` for autovacuum when that is not -1) and capped at `MaxAllocSize` (1 GB), so more memory past that cap does not help ([vacuumlazy.c:12-17](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L12-L17), [vacuumlazy.c#lazy_space_alloc](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2145-L2153)).
- PostgreSQL 14: Holds, with a different dead-TID store. It is `PGC_USERSET` (session scope) with default 65536 kB, and B-tree builds size their sort with it ([guc.c#maintenance_work_mem](../raw/postgres-14/src/backend/utils/misc/guc.c#L2428-L2434), [nbtsort.c:375-378](../raw/postgres-14/src/backend/access/nbtree/nbtsort.c#L375-L378)). In 14, lazy VACUUM keeps dead tuples in a plain TID array sized from it (or `autovacuum_work_mem` for autovacuum workers when not -1), and that array is also capped at `MaxAllocSize` (1 GB) ([vacuumlazy.c:10-16](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L10-L16), [vacuumlazy.c#compute_max_dead_tuples](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L3452-L3464)).
- PostgreSQL 18: Holds. Still `user` context (session scope), default 65536 kB, used by B-tree builds and for the dead-TID store, with `autovacuum_work_mem` taking over in workers when not -1 ([guc_tables.c#maintenance_work_mem](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2594-L2602), [nbtsort.c:372-377](../raw/postgres-18/src/backend/access/nbtree/nbtsort.c#L372-L377), [vacuumlazy.c#dead_items_alloc](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L3497-L3502)).
- PostgreSQL 19: Holds. It is still `PGC_USERSET` (session scope) with default 65536 kB, B-tree builds still size their sort with it, and VACUUM still bounds its dead-TID store with it unless an autovacuum worker has `autovacuum_work_mem` set ([guc_parameters.dat#maintenance_work_mem](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1951-L1959), [nbtsort.c:375-380](../raw/postgres-19/src/backend/access/nbtree/nbtsort.c#L375-L380), [vacuumlazy.c:3-14](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L3-L14), [vacuumlazy.c#dead_items_alloc](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L3443-L3445)). `autovacuum_work_mem` is `PGC_SIGHUP` (reload) ([guc_parameters.dat#autovacuum_work_mem](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L291-L300)).

Related: [work_mem](#work_mem), [VACUUM](#vacuum), [TID](#tid), [GUC context](#guc-context)

### Memoize

**Aliases:** Memoize node, `nodeMemoize.c`, `enable_memoize`, `get_memoize_path`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Memoize is a plan node that caches the results of a parameterized inner scan, usually the inner side of a nested loop join. When the outer side repeats a parameter value, Memoize returns the cached rows instead of rescanning ([nodeMemoize.c header](../raw/postgres-17/src/backend/executor/nodeMemoize.c#L13-L19)). The cache is a hash table with least-recently-used eviction. It never spills to disk ([nodeMemoize.c header](../raw/postgres-17/src/backend/executor/nodeMemoize.c#L21-L26)). Its memory limit is `work_mem` times `hash_mem_multiplier` ([nodeMemoize.c#ExecInitMemoize](../raw/postgres-17/src/backend/executor/nodeMemoize.c#L1039), [nodeHash.c#get_hash_memory_limit](../raw/postgres-17/src/backend/executor/nodeHash.c#L3602-L3613)). The planner tries to put a Memoize path on top of a parameterized inner path in `get_memoize_path()` ([joinpath.c#get_memoize_path](../raw/postgres-17/src/backend/optimizer/path/joinpath.c#L577-L581)). `enable_memoize`, on by default, has context `user`, so it applies per session or transaction ([guc_tables.c#enable_memoize](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L874-L882)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The executor makefile builds no `nodeMemoize.o` ([executor/Makefile:24-25](../raw/postgres-12/src/backend/executor/Makefile#L24-L25)).
- PostgreSQL 14: Present, with the same GUC context ([nodeMemoize.c header](../raw/postgres-14/src/backend/executor/nodeMemoize.c#L1-L3), [guc.c#enable_memoize](../raw/postgres-14/src/backend/utils/misc/guc.c#L1049), [joinpath.c#get_memoize_path](../raw/postgres-14/src/backend/optimizer/path/joinpath.c#L507)).
- PostgreSQL 18: Holds ([guc_tables.c#enable_memoize](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L893), [joinpath.c#get_memoize_path](../raw/postgres-18/src/backend/optimizer/path/joinpath.c#L675)).
- PostgreSQL 19: Holds ([guc_parameters.dat#enable_memoize](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L950), [joinpath.c#get_memoize_path](../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L712)).

Related: [Executor](#executor), [Path](#path), [work_mem](#work_mem), [Planner](#planner)

### Memory context

**Aliases:** `palloc`, `pfree`, `CurrentMemoryContext`, `MemoryContextSwitchTo`, `TopMemoryContext`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A memory context is a named pool of memory with a defined lifetime, and the server does most of its allocation inside one. Freeing or resetting the context releases everything allocated in it at once ([mmgr/README:9-49](../raw/postgres-17/src/backend/utils/mmgr/README#L9-L49)). `palloc` allocates in `CurrentMemoryContext`, and `MemoryContextSwitchTo` changes that target ([mmgr/README:36-40](../raw/postgres-17/src/backend/utils/mmgr/README#L36-L40)). `palloc` never returns NULL: when it runs out of memory it raises `elog(ERROR)` ([mmgr/README:59-62](../raw/postgres-17/src/backend/utils/mmgr/README#L59-L62)). Contexts form a tree rooted at `TopMemoryContext`, with children such as `TopTransactionContext` and `MessageContext` ([mmgr/README:185-223](../raw/postgres-17/src/backend/utils/mmgr/README#L185-L223)). Knowing the current context tells a reader how long a pointer stays valid.

**Version notes:**
- PostgreSQL 12: Holds. `palloc` allocates in `CurrentMemoryContext`, never returns NULL, and contexts form a tree under `TopMemoryContext` with `MessageContext` and `TopTransactionContext` ([mmgr/README:33-40](../raw/postgres-12/src/backend/utils/mmgr/README#L33-L40), [mmgr/README:55-57](../raw/postgres-12/src/backend/utils/mmgr/README#L55-L57), [mmgr/README:175-215](../raw/postgres-12/src/backend/utils/mmgr/README#L175-L215)).
- PostgreSQL 14: Holds. `palloc` allocates in `CurrentMemoryContext`, `MemoryContextSwitchTo` changes it, `palloc` raises `elog(ERROR)` on out-of-memory, and the tree is rooted at `TopMemoryContext` ([mmgr/README:37-40](../raw/postgres-14/src/backend/utils/mmgr/README#L37-L40), [mmgr/README:59-62](../raw/postgres-14/src/backend/utils/mmgr/README#L59-L62), [mmgr/README:179-210](../raw/postgres-14/src/backend/utils/mmgr/README#L179-L210)).
- PostgreSQL 18: Holds ([mmgr/README:9-49](../raw/postgres-18/src/backend/utils/mmgr/README#L9-L49), [mmgr/README:185-223](../raw/postgres-18/src/backend/utils/mmgr/README#L185-L223)).
- PostgreSQL 19: Holds. `palloc` still allocates in `CurrentMemoryContext`, `MemoryContextSwitchTo` still changes it, `palloc` still raises `elog(ERROR)` instead of returning NULL, and the tree is still rooted at `TopMemoryContext` ([mmgr/README:9-49](../raw/postgres-19/src/backend/utils/mmgr/README#L9-L49), [mmgr/README:35-40](../raw/postgres-19/src/backend/utils/mmgr/README#L35-L40), [mmgr/README:59-62](../raw/postgres-19/src/backend/utils/mmgr/README#L59-L62), [mmgr/README:185-223](../raw/postgres-19/src/backend/utils/mmgr/README#L185-L223)).

Related: [ereport](#ereport), [work_mem](#work_mem), [Portal](#portal)

### Metapage

**Aliases:** meta page, metadata page. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A metapage is a special page, at block 0 of an index, that holds bookkeeping for the whole index instead of index entries. B-tree, hash, GIN, SP-GiST and BRIN each define one at block 0; GiST puts its root there instead ([nbtree.h:148](../raw/postgres-17/src/include/access/nbtree.h#L148), [hash.h:198](../raw/postgres-17/src/include/access/hash.h#L198), [ginblock.h:52](../raw/postgres-17/src/include/access/ginblock.h#L52), [spgist_private.h:47](../raw/postgres-17/src/include/access/spgist_private.h#L47), [brin_page.h:75](../raw/postgres-17/src/include/access/brin_page.h#L75), [gist_private.h:262](../raw/postgres-17/src/include/access/gist_private.h#L262)). The B-tree metapage (`BTMetaPageData`) records the root, the fast root, the on-disk version and `btm_allequalimage` ([nbtree.h#BTMetaPageData](../raw/postgres-17/src/include/access/nbtree.h#L103-L119)). The GIN metapage records where the pending list starts and ends and how many pages it has ([ginblock.h#GinMetaPageData](../raw/postgres-17/src/include/access/ginblock.h#L55-L75)).

**Version notes:**
- PostgreSQL 12: Holds, except for the B-tree fields. B-tree, hash, GIN, SP-GiST and BRIN put a metapage at block 0, and GiST puts its root there ([nbtree.h:131](../raw/postgres-12/src/include/access/nbtree.h#L131), [hash.h:196](../raw/postgres-12/src/include/access/hash.h#L196), [ginblock.h:51](../raw/postgres-12/src/include/access/ginblock.h#L51), [spgist_private.h:26](../raw/postgres-12/src/include/access/spgist_private.h#L26), [brin_page.h:75](../raw/postgres-12/src/include/access/brin_page.h#L75), [gist_private.h:263](../raw/postgres-12/src/include/access/gist_private.h#L263)). The v12 `BTMetaPageData` has the root, fast root and version, plus `btm_oldest_btpo_xact` and `btm_last_cleanup_num_heap_tuples`, but no `btm_allequalimage` ([nbtree.h#BTMetaPageData](../raw/postgres-12/src/include/access/nbtree.h#L97-L110)). The GIN metapage tracks the pending list's head, tail and page count ([ginblock.h#GinMetaPageData](../raw/postgres-12/src/include/access/ginblock.h#L54-L76)).
- PostgreSQL 14: Holds. B-tree, hash, GIN, SP-GiST and BRIN keep a metapage at block 0, GiST keeps its root there, and the GIN metapage records the pending list's head, tail and page count ([nbtree.h:146](../raw/postgres-14/src/include/access/nbtree.h#L146), [hash.h:196](../raw/postgres-14/src/include/access/hash.h#L196), [ginblock.h:52](../raw/postgres-14/src/include/access/ginblock.h#L52), [spgist_private.h:47](../raw/postgres-14/src/include/access/spgist_private.h#L47), [brin_page.h:75](../raw/postgres-14/src/include/access/brin_page.h#L75), [gist_private.h:262](../raw/postgres-14/src/include/access/gist_private.h#L262), [ginblock.h#GinMetaPageData](../raw/postgres-14/src/include/access/ginblock.h#L55-L75)).
- PostgreSQL 18: Holds ([nbtree.h:149](../raw/postgres-18/src/include/access/nbtree.h#L149), [hash.h:198](../raw/postgres-18/src/include/access/hash.h#L198), [ginblock.h:52](../raw/postgres-18/src/include/access/ginblock.h#L52), [spgist_private.h:47](../raw/postgres-18/src/include/access/spgist_private.h#L47), [brin_page.h:75](../raw/postgres-18/src/include/access/brin_page.h#L75), [gist_private.h:262](../raw/postgres-18/src/include/access/gist_private.h#L262), [ginblock.h#GinMetaPageData](../raw/postgres-18/src/include/access/ginblock.h#L55-L75)).
- PostgreSQL 19: Holds. B-tree, hash, GIN, SP-GiST and BRIN still put the metapage at block 0 and GiST still puts its root there, and the B-tree and GIN metapage fields are unchanged ([nbtree.h:149](../raw/postgres-19/src/include/access/nbtree.h#L149), [hash.h:198](../raw/postgres-19/src/include/access/hash.h#L198), [ginblock.h:52](../raw/postgres-19/src/include/access/ginblock.h#L52), [spgist_private.h:47](../raw/postgres-19/src/include/access/spgist_private.h#L47), [brin_page.h:75](../raw/postgres-19/src/include/access/brin_page.h#L75), [gist_private.h:262](../raw/postgres-19/src/include/access/gist_private.h#L262), [nbtree.h#BTMetaPageData](../raw/postgres-19/src/include/access/nbtree.h#L104-L120), [ginblock.h#GinMetaPageData](../raw/postgres-19/src/include/access/ginblock.h#L55-L75)).

Related: [Fast root](#fast-root), [allequalimage](#allequalimage), [Pending list](#pending-list), [B-tree](#b-tree)

### Most common values and histogram

**Aliases:** MCV list, `most_common_vals`, `most_common_freqs`, `histogram_bounds`, `n_distinct`, `stadistinct`, `STATISTIC_KIND_MCV`, `STATISTIC_KIND_HISTOGRAM`, `default_statistics_target`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

These are the main per-column statistics that ANALYZE stores in `pg_statistic` and `pg_stats` shows. The most-common-values (MCV) list holds the column's most frequent non-null values with the fraction of rows each covers ([pg_statistic.h#STATISTIC_KIND_MCV](../raw/postgres-17/src/include/catalog/pg_statistic.h#L180-L190)). The histogram holds boundary values that split the remaining values, those not in the MCV list, into buckets of roughly equal population ([pg_statistic.h#STATISTIC_KIND_HISTOGRAM](../raw/postgres-17/src/include/catalog/pg_statistic.h#L193-L210)). `stadistinct` estimates the number of distinct values. A negative value means a fraction of the row count, so −1 means unique ([pg_statistic.h#stadistinct](../raw/postgres-17/src/include/catalog/pg_statistic.h#L53-L69)). `pg_stats` exposes them as `n_distinct`, `most_common_vals`, `most_common_freqs` and `histogram_bounds` ([system_views.sql#pg_stats](../raw/postgres-17/src/backend/catalog/system_views.sql#L197-L218)). For `col = const`, `var_eq_const()` reads the MCV list first. For range comparisons, `ineq_histogram_selectivity()` reads the histogram ([selfuncs.c#var_eq_const](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L290-L296), [selfuncs.c#ineq_histogram_selectivity](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L1043)). `default_statistics_target`, default 100, sets how many entries each list can hold. Its context is `user`, so it applies per session or transaction ([guc_tables.c#default_statistics_target](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2071-L2079)).

**Version notes:**
- PostgreSQL 12: Holds ([pg_statistic.h#STATISTIC_KIND_MCV](../raw/postgres-12/src/include/catalog/pg_statistic.h#L168-L198), [system_views.sql#pg_stats](../raw/postgres-12/src/backend/catalog/system_views.sql#L197-L218), [guc.c#default_statistics_target](../raw/postgres-12/src/backend/utils/misc/guc.c#L1986)).
- PostgreSQL 14: Holds ([pg_statistic.h#STATISTIC_KIND_MCV](../raw/postgres-14/src/include/catalog/pg_statistic.h#L176-L206), [system_views.sql#pg_stats](../raw/postgres-14/src/backend/catalog/system_views.sql#L197-L218), [guc.c#default_statistics_target](../raw/postgres-14/src/backend/utils/misc/guc.c#L2159)).
- PostgreSQL 18: Holds ([pg_statistic.h#STATISTIC_KIND_MCV](../raw/postgres-18/src/include/catalog/pg_statistic.h#L180-L210), [system_views.sql#pg_stats](../raw/postgres-18/src/backend/catalog/system_views.sql#L193-L214), [guc_tables.c#default_statistics_target](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2188)).
- PostgreSQL 19: Holds ([pg_statistic.h#STATISTIC_KIND_MCV](../raw/postgres-19/src/include/catalog/pg_statistic.h#L184-L214), [system_views.sql#pg_stats](../raw/postgres-19/src/backend/catalog/system_views.sql#L200-L221), [guc_parameters.dat#default_statistics_target](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L750)).

Related: [Statistics](#statistics), [Selectivity](#selectivity), [Correlation](#correlation), [Extended statistics](#extended-statistics), [pg_class](#pg_class)

### MultiXact

**Aliases:** MultiXactId, MXID, `pg_multixact`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An ID that stands for a set of transactions, used when several transactions lock or update the same row at once and a single xmax cannot name them all ([multixact.c](../raw/postgres-17/src/backend/access/transam/multixact.c#L5-L16)). A `MultiXactId` has the same width as a `TransactionId` so that it fits in `t_xmax` ([c.h:678-679](../raw/postgres-17/src/include/c.h#L678-L679)). The `HEAP_XMAX_IS_MULTI` infomask bit says which one `t_xmax` holds ([htup_details.h:209](../raw/postgres-17/src/include/access/htup_details.h#L209)). MultiXactIds also age and must be frozen by VACUUM, as xids are ([heapam.c#heap_prepare_freeze_tuple](../raw/postgres-17/src/backend/access/heap/heapam.c#L7393-L7403)).

**Version notes:**
- PostgreSQL 12: Holds. A MultiXactId stands for a set of lockers or updaters, it is a `TransactionId`-width type, `HEAP_XMAX_IS_MULTI` flags it, and `heap_prepare_freeze_tuple()` also handles multis ([multixact.c](../raw/postgres-12/src/backend/access/transam/multixact.c#L5-L16), [c.h:517](../raw/postgres-12/src/include/c.h#L517), [htup_details.h:208](../raw/postgres-12/src/include/access/htup_details.h#L208), [heapam.c#heap_prepare_freeze_tuple](../raw/postgres-12/src/backend/access/heap/heapam.c#L6085-L6110)).
- PostgreSQL 14: Holds. `pg_multixact` stores member arrays, `MultiXactId` is a `TransactionId`-width type, `HEAP_XMAX_IS_MULTI` marks it, and freezing processes multixacts through `FreezeMultiXactId()` ([multixact.c](../raw/postgres-14/src/backend/access/transam/multixact.c#L5-L20), [c.h:642](../raw/postgres-14/src/include/c.h#L642), [htup_details.h:208](../raw/postgres-14/src/include/access/htup_details.h#L208), [heapam.c:7074](../raw/postgres-14/src/backend/access/heap/heapam.c#L7074)).
- PostgreSQL 18: Holds ([multixact.c](../raw/postgres-18/src/backend/access/transam/multixact.c#L5-L16), [c.h:647-648](../raw/postgres-18/src/include/c.h#L647-L648), [htup_details.h:209](../raw/postgres-18/src/include/access/htup_details.h#L209)).
- PostgreSQL 19: Holds. A MultiXactId still stands for a set of member transactions, it is still as wide as a `TransactionId` so it fits in `t_xmax`, and `HEAP_XMAX_IS_MULTI` still marks it ([multixact.c](../raw/postgres-19/src/backend/access/transam/multixact.c#L5-L17), [c.h:752-753](../raw/postgres-19/src/include/c.h#L752-L753), [htup_details.h:209](../raw/postgres-19/src/include/access/htup_details.h#L209)). In v19, `MultiXactOffset`, the position in the members area, is 64-bit (`uint64`) ([c.h:755](../raw/postgres-19/src/include/c.h#L755)).

Related: [xmin and xmax](#xmin-and-xmax), [Transaction ID](#transaction-id), [Freezing](#freezing), [Wraparound](#wraparound)

### MVCC

**Aliases:** multi-version concurrency control. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

PostgreSQL's way of letting readers and writers work on the same rows without blocking each other. An update creates a new version of a row instead of changing it in place, and old versions are removed only after no transaction can still see them ([glossary.sgml#mvcc](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1171-L1185)). Each heap tuple records its inserting and deleting transactions in `t_xmin` and `t_xmax` ([htup_details.h#HeapTupleFields](../raw/postgres-17/src/include/access/htup_details.h#L122-L132)). `HeapTupleSatisfiesMVCC()` checks those against a snapshot to decide visibility ([heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L937-L960)). Outdated row versions count as bloat until they are removed ([glossary.sgml#glossary-bloat](../raw/postgres-17/doc/src/sgml/glossary.sgml#L242-L250)).

**Version notes:**
- PostgreSQL 12: Holds. The docs describe multiversion concurrency, `HeapTupleFields` holds `t_xmin` and `t_xmax`, and `HeapTupleSatisfiesMVCC()` checks them against a snapshot ([mvcc.sgml#mvcc-intro](../raw/postgres-12/doc/src/sgml/mvcc.sgml#L20-L45), [htup_details.h#HeapTupleFields](../raw/postgres-12/src/include/access/htup_details.h#L121-L132), [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L962)).
- PostgreSQL 14: Holds. The docs glossary defines it the same way, tuples carry `t_xmin` and `t_xmax`, and `HeapTupleSatisfiesMVCC()` checks them against a snapshot ([glossary.sgml#glossary-mvcc](../raw/postgres-14/doc/src/sgml/glossary.sgml#L945-L960), [htup_details.h#HeapTupleFields](../raw/postgres-14/src/include/access/htup_details.h#L121-L131), [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-14/src/backend/access/heap/heapam_visibility.c#L959-L1144)).
- PostgreSQL 18: Holds ([glossary.sgml#mvcc](../raw/postgres-18/doc/src/sgml/glossary.sgml#L1220-L1234), [htup_details.h#HeapTupleFields](../raw/postgres-18/src/include/access/htup_details.h#L122-L132), [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-18/src/backend/access/heap/heapam_visibility.c#L937-L960)).
- PostgreSQL 19: Holds. The glossary definition, `t_xmin` and `t_xmax`, and `HeapTupleSatisfiesMVCC()` are unchanged ([glossary.sgml#glossary-mvcc](../raw/postgres-19/doc/src/sgml/glossary.sgml#L1268-L1283), [htup_details.h#HeapTupleFields](../raw/postgres-19/src/include/access/htup_details.h#L122-L132), [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-19/src/backend/access/heap/heapam_visibility.c#L939), [glossary.sgml#glossary-bloat](../raw/postgres-19/doc/src/sgml/glossary.sgml#L283-L291)). v19 adds a page-at-a-time variant, `HeapTupleSatisfiesMVCCBatch()` ([heapam_visibility.c:1690](../raw/postgres-19/src/backend/access/heap/heapam_visibility.c#L1690)).

Related: [Snapshot](#snapshot), [xmin and xmax](#xmin-and-xmax), [xmin horizon](#xmin-horizon), [Tuple](#tuple), [Bloat](#bloat), [VACUUM](#vacuum)

### NOT VALID

**Aliases:** `NOT VALID` constraint, `VALIDATE CONSTRAINT`, `pg_constraint.convalidated`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`NOT VALID` is an `ALTER TABLE ... ADD CONSTRAINT` option that adds a constraint without scanning the rows already in the table. In PostgreSQL 17 it applies to foreign-key and `CHECK` constraints. New inserts and updates are checked from then on, but the database does not assume old rows satisfy the constraint ([ref/alter_table.sgml#ADD-table-constraint](../raw/postgres-17/doc/src/sgml/ref/alter_table.sgml#L452-L475)). A later `ALTER TABLE ... VALIDATE CONSTRAINT` scans the table to check the old rows. It takes only a [ShareUpdateExclusiveLock](#shareupdateexclusivelock), plus a `ROW SHARE` lock on the referenced table for a foreign key, so concurrent writes can continue ([ref/alter_table.sgml#VALIDATE-CONSTRAINT](../raw/postgres-17/doc/src/sgml/ref/alter_table.sgml#L564-L576), [ref/alter_table.sgml#notes](../raw/postgres-17/doc/src/sgml/ref/alter_table.sgml#L1445-L1466)). The catalog records the state in `pg_constraint.convalidated` ([pg_constraint.h:54](../raw/postgres-17/src/include/catalog/pg_constraint.h#L54)), and `ATExecValidateConstraint()` performs the validation ([tablecmds.c#ATExecValidateConstraint](../raw/postgres-17/src/backend/commands/tablecmds.c#L11719)). Foreign keys on partitioned tables cannot be `NOT VALID` in 17 ([ref/alter_table.sgml:490-492](../raw/postgres-17/doc/src/sgml/ref/alter_table.sgml#L490-L492)).

**Version notes:**
- PostgreSQL 12: Holds for foreign-key and `CHECK` constraints ([ref/alter_table.sgml:368-370](../raw/postgres-12/doc/src/sgml/ref/alter_table.sgml#L368-L370), [pg_constraint.h:54](../raw/postgres-12/src/include/catalog/pg_constraint.h#L54), [tablecmds.c#ATExecValidateConstraint](../raw/postgres-12/src/backend/commands/tablecmds.c#L9216)). The lock level for validation is stated only in the notes ([ref/alter_table.sgml:1241-1243](../raw/postgres-12/doc/src/sgml/ref/alter_table.sgml#L1241-L1243)).
- PostgreSQL 14: Holds ([ref/alter_table.sgml:421-423](../raw/postgres-14/doc/src/sgml/ref/alter_table.sgml#L421-L423), [ref/alter_table.sgml:540](../raw/postgres-14/doc/src/sgml/ref/alter_table.sgml#L540), [tablecmds.c#ATExecValidateConstraint](../raw/postgres-14/src/backend/commands/tablecmds.c#L10807)).
- PostgreSQL 18: Not-null constraints can also be `NOT VALID` ([ref/alter_table.sgml:472-474](../raw/postgres-18/doc/src/sgml/ref/alter_table.sgml#L472-L474)). Foreign-key validation is queued through `QueueFKConstraintValidation()` ([tablecmds.c#QueueFKConstraintValidation](../raw/postgres-18/src/backend/commands/tablecmds.c#L13016)).
- PostgreSQL 19: Same as 18 ([ref/alter_table.sgml:483-485](../raw/postgres-19/doc/src/sgml/ref/alter_table.sgml#L483-L485), [pg_constraint.h:57](../raw/postgres-19/src/include/catalog/pg_constraint.h#L57), [tablecmds.c#ATExecValidateConstraint](../raw/postgres-19/src/backend/commands/tablecmds.c#L13337)).

Related: [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [Lock mode](#lock-mode), [Catalog](#catalog)

### OID

**Aliases:** object identifier, `Oid`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An OID is an unsigned 4-byte integer that serves as the primary key of most system catalogs ([postgres_ext.h:27-31](../raw/postgres-17/src/include/postgres_ext.h#L27-L31), [datatype.sgml:4767-4781](../raw/postgres-17/doc/src/sgml/datatype.sgml#L4767-L4781)). OIDs below 16384 are reserved for objects that `initdb` creates. User objects get OIDs from 16384 upward, and a wrapped counter skips back over the reserved range ([transam.h:173-197](../raw/postgres-17/src/include/access/transam.h#L173-L197)). OIDs are unique only within one catalog, so `GetNewOidWithIndex` checks the target catalog's index for a collision ([catalog.c#GetNewOidWithIndex](../raw/postgres-17/src/backend/catalog/catalog.c#L396-L421)). A relation's OID (`pg_class.oid`) is not its file name. The file is named by its [relfilenumber](#relfilenumber), and the two can diverge ([pg_upgrade.c:17-20](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L17-L20)).

**Version notes:**
- PostgreSQL 12: Holds. `Oid` is an unsigned 4-byte integer, user OIDs start at 16384, a wrapped counter skips the reserved range, and `GetNewOidWithIndex` checks the catalog index ([postgres_ext.h:27-31](../raw/postgres-12/src/include/postgres_ext.h#L27-L31), [datatype.sgml](../raw/postgres-12/doc/src/sgml/datatype.sgml#L4541-L4545), [transam.h:122-141](../raw/postgres-12/src/include/access/transam.h#L122-L141), [catalog.c#GetNewOidWithIndex](../raw/postgres-12/src/backend/catalog/catalog.c#L330)). In 12 the reserved sub-ranges differ (`FirstGenbkiObjectId` 10000, `FirstBootstrapObjectId` 12000) ([transam.h:139-141](../raw/postgres-12/src/include/access/transam.h#L139-L141)). The relation's file is named by its relfilenode, which can diverge from its OID ([pg_upgrade.c:14-23](../raw/postgres-12/src/bin/pg_upgrade/pg_upgrade.c#L14-L23)).
- PostgreSQL 14: Holds. `Oid` is an unsigned int, user OIDs start at 16384 and a wrapped counter skips the reserved range, and `GetNewOidWithIndex` checks the catalog index ([postgres_ext.h:27-31](../raw/postgres-14/src/include/postgres_ext.h#L27-L31), [transam.h:185-192](../raw/postgres-14/src/include/access/transam.h#L185-L192), [catalog.c#GetNewOidWithIndex](../raw/postgres-14/src/backend/catalog/catalog.c#L352-L442)). The file name is `relfilenode` in 14, and it can diverge from the OID after `CLUSTER`, `REINDEX` or `VACUUM FULL` ([pg_upgrade.c:18-23](../raw/postgres-14/src/bin/pg_upgrade/pg_upgrade.c#L18-L23)).
- PostgreSQL 18: Holds ([postgres_ext.h:28-32](../raw/postgres-18/src/include/postgres_ext.h#L28-L32), [transam.h:173-197](../raw/postgres-18/src/include/access/transam.h#L173-L197), [catalog.c#GetNewOidWithIndex](../raw/postgres-18/src/backend/catalog/catalog.c#L423-L448)).
- PostgreSQL 19: Holds. `Oid` is still an unsigned 4-byte integer, user OIDs still start at 16384, a wrapped counter still skips the reserved range, `GetNewOidWithIndex()` still checks for collisions, and `pg_upgrade` still notes that OID and relfilenode can diverge ([postgres_ext.h:29-32](../raw/postgres-19/src/include/postgres_ext.h#L29-L32), [datatype.sgml](../raw/postgres-19/doc/src/sgml/datatype.sgml#L4807-L4821), [transam.h:170-197](../raw/postgres-19/src/include/access/transam.h#L170-L197), [catalog.c#GetNewOidWithIndex](../raw/postgres-19/src/backend/catalog/catalog.c#L448), [pg_upgrade.c:9-21](../raw/postgres-19/src/bin/pg_upgrade/pg_upgrade.c#L9-L21)).

Related: [Catalog](#catalog), [pg_class](#pg_class), [Relfilenumber](#relfilenumber), [Transaction ID](#transaction-id)

### Operator class

**Aliases:** opclass, operator family, opfamily, `pg_opclass`, `pg_opfamily`, `pg_amop`, `pg_amproc`, strategy number, support function. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An operator class tells an index access method how to handle one data type: which operators the index can answer and which helper functions it calls. You need one before you can index a column of that type ([xindex.sgml#xindex](../raw/postgres-17/doc/src/sgml/xindex.sgml#L3-L32)). Each row of `pg_opclass` ties an access method and input type to an operator family, and says whether it is the default class for that type ([pg_opclass.h#FormData_pg_opclass](../raw/postgres-17/src/include/catalog/pg_opclass.h#L49-L76)). An operator family groups compatible classes so that cross-type comparisons, such as `int4` against `int8`, can use the index ([xindex.sgml#xindex-opfamily](../raw/postgres-17/doc/src/sgml/xindex.sgml#L987-L1000), [pg_opfamily.h#FormData_pg_opfamily](../raw/postgres-17/src/include/catalog/pg_opfamily.h#L29-L44)). The family's operators sit in `pg_amop` under strategy numbers, and its support functions sit in `pg_amproc` under support numbers ([pg_amop.h#FormData_pg_amop](../raw/postgres-17/src/include/catalog/pg_amop.h#L54-L66), [pg_amproc.h#FormData_pg_amproc](../raw/postgres-17/src/include/catalog/pg_amproc.h#L43-L57)). For B-tree, support function 1 is the comparison function and 4 is `BTEQUALIMAGE_PROC`, which decides whether deduplication is safe ([nbtree.h#BTORDER_PROC](../raw/postgres-17/src/include/access/nbtree.h#L702-L712)). An index records its per-column opclass in `pg_index.indclass` ([pg_index.h:54](../raw/postgres-17/src/include/catalog/pg_index.h#L54)).

**Version notes:**
- PostgreSQL 12: Holds for the catalogs ([pg_opclass.h#FormData_pg_opclass](../raw/postgres-12/src/include/catalog/pg_opclass.h#L49-L76), [pg_opfamily.h#FormData_pg_opfamily](../raw/postgres-12/src/include/catalog/pg_opfamily.h#L29-L44)). B-tree has only three support functions, with no `BTEQUALIMAGE_PROC` ([nbtree.h#BTORDER_PROC](../raw/postgres-12/src/include/access/nbtree.h#L391-L395)).
- PostgreSQL 14: Holds, with five B-tree support functions, as in 17 ([pg_opclass.h#FormData_pg_opclass](../raw/postgres-14/src/include/catalog/pg_opclass.h#L49-L76), [nbtree.h#BTEQUALIMAGE_PROC](../raw/postgres-14/src/include/access/nbtree.h#L703-L705)).
- PostgreSQL 18: Holds, and B-tree gains a sixth support function, `BTSKIPSUPPORT_PROC`, for skip scan ([pg_opclass.h#FormData_pg_opclass](../raw/postgres-18/src/include/catalog/pg_opclass.h#L49-L76), [nbtree.h#BTORDER_PROC](../raw/postgres-18/src/include/access/nbtree.h#L712-L723)).
- PostgreSQL 19: As in 18 ([pg_opclass.h#FormData_pg_opclass](../raw/postgres-19/src/include/catalog/pg_opclass.h#L51-L78), [nbtree.h#BTEQUALIMAGE_PROC](../raw/postgres-19/src/include/access/nbtree.h#L720-L723)).

Related: [Access method](#access-method), [B-tree](#b-tree), [allequalimage](#allequalimage), [Deduplication](#deduplication), [pg_index](#pg_index), [Collation](#collation)

### Origin filter

**Aliases:** subscription `origin` parameter, `origin = none`, `origin = any`, `suborigin`, `LOGICALREP_ORIGIN_NONE`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An origin filter lets a subscription ask the publisher to skip changes that were themselves replicated in from somewhere else. It is how two nodes can replicate to each other without looping changes back and forth. The subscription stores the choice in `pg_subscription.suborigin`, which is `any` by default; `none` asks only for changes that have no replication origin ([pg_subscription.h#LOGICALREP_ORIGIN_NONE](../raw/postgres-17/src/include/catalog/pg_subscription.h#L34-L44), [pg_subscription.h#suborigin](../raw/postgres-17/src/include/catalog/pg_subscription.h#L113-L115)). On the publisher, `pgoutput` parses the `origin` option into `publish_no_origin` and rejects any other value ([pgoutput.c#parse_output_parameters](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L395-L410)). Its `pgoutput_origin_filter()` callback then drops every change whose origin id is set ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L1725-L1737)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. `pg_subscription` has no `suborigin` column ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-12/src/include/catalog/pg_subscription.h#L39-L64)), and `pgoutput_origin_filter()` always returns false, so every change is forwarded ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-12/src/backend/replication/pgoutput/pgoutput.c#L430-L437)).
- PostgreSQL 14: Not present in PostgreSQL 14. `pg_subscription` has no `suborigin` column ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-14/src/include/catalog/pg_subscription.h#L42-L73)), and `pgoutput_origin_filter()` always returns false ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-14/src/backend/replication/pgoutput/pgoutput.c#L818-L823)).
- PostgreSQL 18: Holds. The same `none`/`any` values and filter exist ([pg_subscription.h#LOGICALREP_ORIGIN_NONE](../raw/postgres-18/src/include/catalog/pg_subscription.h#L152-L162), [pgoutput.c#pgoutput_origin_filter](../raw/postgres-18/src/backend/replication/pgoutput/pgoutput.c#L1765-L1776)).
- PostgreSQL 19: Holds. The filter now takes a `ReplOriginId` and compares it against `InvalidReplOriginId`; those are new spellings of the origin id type and its "no origin" value ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-19/src/backend/replication/pgoutput/pgoutput.c#L1768-L1778), [pg_subscription.h#suborigin](../raw/postgres-19/src/include/catalog/pg_subscription.h#L113-L115)).

Related: [Replication origin](#replication-origin), [Subscription](#subscription), [Logical replication](#logical-replication), [Logical decoding](#logical-decoding)

### OS page cache

**Aliases:** kernel page cache, kernel disk cache, operating system cache, double buffering. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The OS page cache is the operating system's own memory cache of file data. It sits under PostgreSQL's [shared_buffers](#shared_buffers). The docs say PostgreSQL relies on it, which is why a `shared_buffers` above about 40% of RAM is unlikely to help ([config.sgml#guc-shared-buffers](../raw/postgres-17/doc/src/sgml/config.sgml#L1672-L1684)). The planner accounts for both caches through [effective_cache_size](#effective_cache_size), which should cover shared buffers plus "the portion of the kernel's disk cache" holding PostgreSQL data files. It is only an estimate and reserves nothing ([config.sgml#guc-effective-cache-size](../raw/postgres-17/doc/src/sgml/config.sgml#L6034-L6052)). A write reaches durable storage only after [fsync](#fsync) ([config.sgml#guc-fsync](../raw/postgres-17/doc/src/sgml/config.sgml#L3022-L3035)). `pg_flush_data()` exists to ask the kernel to start writing dirty cached data early ([fd.c#pg_flush_data](../raw/postgres-17/src/backend/storage/file/fd.c#L518-L530)). PostgreSQL 17 can bypass the kernel cache only through the developer setting `debug_io_direct`. Its context is `postmaster` (restart), and the docs say it currently reduces performance ([guc_tables.c:4699-4708](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4699-L4708), [config.sgml#guc-debug-io-direct](../raw/postgres-17/doc/src/sgml/config.sgml#L11595-L11624)).

**Version notes:**
- PostgreSQL 12: The same `shared_buffers` and `effective_cache_size` guidance appears ([config.sgml:1510-1520](../raw/postgres-12/doc/src/sgml/config.sgml#L1510-L1520), [config.sgml:4895-4911](../raw/postgres-12/doc/src/sgml/config.sgml#L4895-L4911)), and `pg_flush_data()` exists ([fd.c:405](../raw/postgres-12/src/backend/storage/file/fd.c#L405)).
- PostgreSQL 14: Same as 12 ([config.sgml:1634-1644](../raw/postgres-14/doc/src/sgml/config.sgml#L1634-L1644), [config.sgml:5534-5550](../raw/postgres-14/doc/src/sgml/config.sgml#L5534-L5550)).
- PostgreSQL 18: Holds, with `debug_io_direct` still `postmaster` context ([config.sgml:1749-1759](../raw/postgres-18/doc/src/sgml/config.sgml#L1749-L1759), [guc_tables.c:4940](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L4940)).
- PostgreSQL 19: Holds ([config.sgml:1821-1831](../raw/postgres-19/doc/src/sgml/config.sgml#L1821-L1831), [guc_parameters.dat:676](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L676)).

Related: [shared_buffers](#shared_buffers), [effective_cache_size](#effective_cache_size), [fsync](#fsync), [Buffer manager](#buffer-manager)

### Page

**Aliases:** disk page, slotted page, `PageHeaderData`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A page is the formatted layout inside a block. It has a header, an array of line pointers growing forward, tuples growing backward from the end, and an optional "special space" for index metadata. The page is full when `pd_lower` meets `pd_upper` ([bufpage.h#layout](../raw/postgres-17/src/include/storage/bufpage.h#L22-L45)). `PageHeaderData` holds the page LSN (`pd_lsn`), checksum, flags, the free-space bounds, `pd_special`, and `pd_prune_xid`, which is a pruning hint ([bufpage.h#PageHeaderData](../raw/postgres-17/src/include/storage/bufpage.h#L155-L168)).

**Version notes:**
- PostgreSQL 12: Holds. The layout, `pd_lower`/`pd_upper`, special space, `pd_lsn`, checksum and `pd_prune_xid` are the same ([bufpage.h#layout](../raw/postgres-12/src/include/storage/bufpage.h#L22-L45), [bufpage.h#PageHeaderData](../raw/postgres-12/src/include/storage/bufpage.h#L151-L164)).
- PostgreSQL 14: Holds. The layout is the same, and `PageHeaderData` has `pd_lsn`, `pd_checksum`, `pd_flags`, `pd_lower`, `pd_upper`, `pd_special` and `pd_prune_xid` ([bufpage.h#layout](../raw/postgres-14/src/include/storage/bufpage.h#L22-L45), [bufpage.h#PageHeaderData](../raw/postgres-14/src/include/storage/bufpage.h#L152-L164)).
- PostgreSQL 18: Holds for the layout and `PageHeaderData` ([bufpage.h#layout](../raw/postgres-18/src/include/storage/bufpage.h#L25-L48), [bufpage.h#PageHeaderData](../raw/postgres-18/src/include/storage/bufpage.h#L159-L172)). One related change: `initdb` now enables data checksums by default (false in 17), so the header's checksum field is filled unless `--no-data-checksums` is given ([initdb.c:167](../raw/postgres-18/src/bin/initdb/initdb.c#L167), [initdb.c:2542](../raw/postgres-18/src/bin/initdb/initdb.c#L2542)).
- PostgreSQL 19: Holds. The slotted layout, `pd_lower`/`pd_upper`, special space and the `PageHeaderData` fields are unchanged ([bufpage.h#layout](../raw/postgres-19/src/include/storage/bufpage.h#L25-L47), [bufpage.h#PageHeaderData](../raw/postgres-19/src/include/storage/bufpage.h#L184-L197)). `pd_lsn` now has type `PageXLogRecPtr` ([bufpage.h:187](../raw/postgres-19/src/include/storage/bufpage.h#L187)).

Related: [Block](#block), [Line pointer](#line-pointer), [LSN](#lsn), [Metapage](#metapage)

### Page split

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A page split happens when a new entry does not fit on an index page. In a B-tree, `_bt_insertonpg()` calls `_bt_split()` when `PageGetFreeSpace()` is smaller than the new item. `_bt_split()` moves part of the entries to a new right sibling, and `_bt_insert_parent()` then adds a downlink for it in the parent page ([nbtinsert.c:1210-1241](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1210-L1241), [nbtinsert.c#_bt_split](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L1440-L1470), [nbtinsert.c#_bt_insert_parent](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2083-L2084)). `_bt_findsplitloc()` normally divides free space evenly between the two pages. On the rightmost page it leaves the left page `fillfactor`% full, so ascending inserts produce pages about that full ([nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L86-L102)). Before splitting a leaf, B-tree first tries simple deletion, then bottom-up deletion, then deduplication ([nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2683-L2781)).

**Version notes:**
- PostgreSQL 12: Differs in the pre-split steps. `_bt_insertonpg()` calls `_bt_split()` when free space is short, and `_bt_insert_parent()` adds the downlink ([nbtinsert.c:974-999](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L974-L999), [nbtinsert.c#_bt_split](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1216), [nbtinsert.c#_bt_insert_parent](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L1742)). `_bt_findsplitloc()` leaves a rightmost left page `fillfactor`% full ([nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L97-L127)). Before a split, 12 only removes `LP_DEAD` items with `_bt_vacuum_one_page()`; there is no bottom-up deletion or deduplication step ([nbtinsert.c:753-759](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L753-L759)).
- PostgreSQL 14: Holds. `_bt_insertonpg()` calls `_bt_split()` when free space is short, then `_bt_insert_parent()` adds the downlink ([nbtinsert.c:1207-1238](../raw/postgres-14/src/backend/access/nbtree/nbtinsert.c#L1207-L1238), [nbtinsert.c#_bt_split](../raw/postgres-14/src/backend/access/nbtree/nbtinsert.c#L1464), [nbtinsert.c#_bt_insert_parent](../raw/postgres-14/src/backend/access/nbtree/nbtinsert.c#L2096)). `_bt_findsplitloc()` leaves the left page fillfactor% full on the rightmost page, and the pre-split order is simple deletion, bottom-up deletion, deduplication ([nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-14/src/backend/access/nbtree/nbtsplitloc.c#L95-L98), [nbtsplitloc.c:284-291](../raw/postgres-14/src/backend/access/nbtree/nbtsplitloc.c#L284-L291), [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-14/src/backend/access/nbtree/nbtinsert.c#L2672-L2771)).
- PostgreSQL 18: Holds ([nbtinsert.c:1210-1241](../raw/postgres-18/src/backend/access/nbtree/nbtinsert.c#L1210-L1241), [nbtinsert.c#_bt_split](../raw/postgres-18/src/backend/access/nbtree/nbtinsert.c#L1440-L1470), [nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-18/src/backend/access/nbtree/nbtsplitloc.c#L86-L102)).
- PostgreSQL 19: Holds. `_bt_insertonpg()` still calls `_bt_split()` when `PageGetFreeSpace()` is too small, `_bt_insert_parent()` still adds the downlink, `_bt_findsplitloc()` still equalizes free space except on the rightmost page, and the pre-split order is unchanged ([nbtinsert.c:1220-1236](../raw/postgres-19/src/backend/access/nbtree/nbtinsert.c#L1220-L1236), [nbtinsert.c#_bt_split](../raw/postgres-19/src/backend/access/nbtree/nbtinsert.c#L1489), [nbtinsert.c#_bt_insert_parent](../raw/postgres-19/src/backend/access/nbtree/nbtinsert.c#L2130), [nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-19/src/backend/access/nbtree/nbtsplitloc.c#L90-L99), [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-19/src/backend/access/nbtree/nbtinsert.c#L2730-L2828)).

Related: [Fillfactor](#fillfactor), [Simple index deletion](#simple-index-deletion), [Bottom-up index deletion](#bottom-up-index-deletion), [Deduplication](#deduplication), [Bloat](#bloat)

### pageinspect

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pageinspect` is a contrib extension that shows the raw contents of individual database pages ([pageinspect.control](../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L2)). Its core function `get_raw_page` requires a superuser. It copies one block of any relation fork byte for byte into a `bytea` value ([rawpage.c#get_raw_page_internal](../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L153), [rawpage.c#get_raw_page_internal](../raw/postgres-17/contrib/pageinspect/rawpage.c#L185-L193)). Separate per-format decoders parse such a copy, for example `bt_page_items` for B-tree pages and the GIN and heap readers in their own source files ([pageinspect--1.5--1.6.sql](../raw/postgres-17/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88-L90), [ginfuncs.c header](../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L1-L3), [heapfuncs.c header](../raw/postgres-17/contrib/pageinspect/heapfuncs.c#L1-L4)).

**Version notes:**
- PostgreSQL 12: Holds. `get_raw_page` needs superuser and copies one block of a fork into `bytea`; `bt_page_items`, GIN and heap decoders exist ([pageinspect.control](../raw/postgres-12/contrib/pageinspect/pageinspect.control#L1-L5), [rawpage.c#get_raw_page_internal](../raw/postgres-12/contrib/pageinspect/rawpage.c#L95-L106), [pageinspect--1.5--1.6.sql](../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88), [ginfuncs.c header](../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L1-L3), [heapfuncs.c header](../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L1-L4)). The default version in 12 is 1.7.
- PostgreSQL 14: Holds. `get_raw_page` requires a superuser and copies one block, and per-format decoders such as `bt_page_items` and the GIN and heap readers exist ([pageinspect.control](../raw/postgres-14/contrib/pageinspect/pageinspect.control#L1-L2), [rawpage.c#get_raw_page_internal](../raw/postgres-14/contrib/pageinspect/rawpage.c#L150-L153), [pageinspect--1.5--1.6.sql](../raw/postgres-14/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88-L90), [ginfuncs.c header](../raw/postgres-14/contrib/pageinspect/ginfuncs.c#L1-L3), [heapfuncs.c header](../raw/postgres-14/contrib/pageinspect/heapfuncs.c#L1-L4)).
- PostgreSQL 18: Holds. The extension default version is 1.13 in 18 ([pageinspect.control](../raw/postgres-18/contrib/pageinspect/pageinspect.control#L1-L5), [rawpage.c#get_raw_page_internal](../raw/postgres-18/contrib/pageinspect/rawpage.c#L153-L156), [pageinspect--1.5--1.6.sql](../raw/postgres-18/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88-L90)).
- PostgreSQL 19: Holds. `get_raw_page` still requires a superuser and copies one block of a fork, and the per-format decoders are still separate ([pageinspect.control](../raw/postgres-19/contrib/pageinspect/pageinspect.control#L1-L2), [rawpage.c#get_raw_page_internal](../raw/postgres-19/contrib/pageinspect/rawpage.c#L145-L156), [pageinspect--1.5.sql:178](../raw/postgres-19/contrib/pageinspect/pageinspect--1.5.sql#L178)). The extension's default version is 1.13 ([pageinspect.control](../raw/postgres-19/contrib/pageinspect/pageinspect.control#L3)).

Related: [Page](#page), [Fork](#fork), [Contrib](#contrib), [Extension](#extension), [pgstattuple](#pgstattuple)

### Parallel query

**Aliases:** parallel worker, `Gather`, `Gather Merge`, parallel safety, `proparallel`, `max_parallel_workers_per_gather`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Parallel query lets one query use extra worker processes. A `Gather` or `Gather Merge` plan node launches the workers, each runs a copy of the plan below it, and the node merges their rows into one stream ([nodeGather.c header](../raw/postgres-17/src/backend/executor/nodeGather.c#L9-L15)). The workers are background worker processes. They share state with the leader through dynamic shared memory, set up through a `ParallelContext` ([README.parallel#Overview](../raw/postgres-17/src/backend/access/transam/README.parallel#L1-L12)). Each function carries a parallel-safety label in `pg_proc.proparallel`: safe (`s`), restricted to the leader (`r`), or unsafe (`u`) ([pg_proc.h#PROPARALLEL_SAFE](../raw/postgres-17/src/include/catalog/pg_proc.h#L173-L175)). `EXPLAIN` prints `Workers Planned` and, under `ANALYZE`, `Workers Launched` for a Gather node ([explain.c#Gather](../raw/postgres-17/src/backend/commands/explain.c#L2029-L2046)). `max_parallel_workers_per_gather` caps workers per Gather node. Its context is `user`, so it can be changed per session ([guc_tables.c:3420](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3420)).

**Version notes:**
- PostgreSQL 12: Holds. The Gather design and the three labels are the same ([nodeGather.c header](../raw/postgres-12/src/backend/executor/nodeGather.c#L9-L15), [pg_proc.h#PROPARALLEL_SAFE](../raw/postgres-12/src/include/catalog/pg_proc.h#L163-L165)). The GUC table is in `guc.c`; `max_parallel_workers_per_gather` is `user` context, so session scope ([guc.c:3008](../raw/postgres-12/src/backend/utils/misc/guc.c#L3008)).
- PostgreSQL 14: Holds ([nodeGather.c header](../raw/postgres-14/src/backend/executor/nodeGather.c#L9-L15), [pg_proc.h#PROPARALLEL_SAFE](../raw/postgres-14/src/include/catalog/pg_proc.h#L172-L174)). `max_parallel_workers_per_gather` is `user` context in `guc.c`, so session scope ([guc.c:3324](../raw/postgres-14/src/backend/utils/misc/guc.c#L3324)).
- PostgreSQL 18: Holds ([nodeGather.c header](../raw/postgres-18/src/backend/executor/nodeGather.c#L9-L15), [pg_proc.h#PROPARALLEL_SAFE](../raw/postgres-18/src/include/catalog/pg_proc.h#L173-L175)). `max_parallel_workers_per_gather` is still `user` context, so session scope ([guc_tables.c:3628](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3628)).
- PostgreSQL 19: Holds ([nodeGather.c header](../raw/postgres-19/src/backend/executor/nodeGather.c#L9-L15), [pg_proc.h#PROPARALLEL_SAFE](../raw/postgres-19/src/include/catalog/pg_proc.h#L177-L179)). GUC definitions now live in the data file `guc_parameters.dat`. There, `max_parallel_workers_per_gather` is still `PGC_USERSET`, so session scope ([guc_parameters.dat:2063](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2063)).

Related: [Background worker](#background-worker), [Executor](#executor), [Planner](#planner), [EXPLAIN](#explain), [Parallel vacuum](#parallel-vacuum)

### Parallel vacuum

**Aliases:** parallel index vacuum, `VACUUM (PARALLEL n)`, `vacuumparallel.c`, `ParallelVacuumState`, parallel autovacuum. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Parallel vacuum lets VACUUM hand whole indexes to parallel worker processes during index bulk deletion and index cleanup. Each index is processed by one process, and the workers exit once all indexes are done ([vacuumparallel.c header](../raw/postgres-17/src/backend/commands/vacuumparallel.c#L9-L17)). A user asks for it with the `PARALLEL` option, which accepts 0 up to `MAX_PARALLEL_WORKER_LIMIT` ([vacuum.c#ExecVacuum](../raw/postgres-17/src/backend/commands/vacuum.c#L255-L275)). Autovacuum never uses it in PostgreSQL 17; it sets `nworkers = -1` for every table ([autovacuum.c:2839-2840](../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2839-L2840)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. `ExecVacuum()` accepts no `parallel` option and rejects unknown options ([vacuum.c#ExecVacuum](../raw/postgres-12/src/backend/commands/vacuum.c#L104-L137)).
- PostgreSQL 14: Present, but the code lives in `vacuumlazy.c` rather than a separate `vacuumparallel.c` ([vacuumlazy.c:25-35](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L25-L35)). The `PARALLEL` option exists ([vacuum.c#ExecVacuum](../raw/postgres-14/src/backend/commands/vacuum.c#L166-L172)). Autovacuum does not use it ([autovacuum.c:2989-2990](../raw/postgres-14/src/backend/postmaster/autovacuum.c#L2989-L2990)).
- PostgreSQL 18: Holds as in 17 ([vacuumparallel.c header](../raw/postgres-18/src/backend/commands/vacuumparallel.c#L9-L17)). Autovacuum still does not use it ([autovacuum.c:2853-2854](../raw/postgres-18/src/backend/postmaster/autovacuum.c#L2853-L2854)).
- PostgreSQL 19: Differs. Autovacuum can run parallel vacuum ([vacuumparallel.c header](../raw/postgres-19/src/backend/commands/vacuumparallel.c#L1-L25)). A worker takes its parallel degree from the `autovacuum_parallel_workers` reloption, where 0 disables it ([autovacuum.c:2941-2960](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2941-L2960)). The new `autovacuum_max_parallel_workers` GUC caps it and defaults to 0. Its context is `sighup`, so changing it needs a reload ([guc_parameters.dat:172-176](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L172-L176)).

Related: [VACUUM](#vacuum), [Autovacuum](#autovacuum), [Parallel query](#parallel-query), [maintenance_work_mem](#maintenance_work_mem)

### Parse tree

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A parse tree is the in-memory tree of nodes that represents one SQL statement. `raw_parser()` runs the lexer and grammar and returns raw parse trees, each wrapped in a `RawStmt` [parser.c#raw_parser](../raw/postgres-17/src/backend/parser/parser.c#L34-L42) [parsenodes.h#RawStmt](../raw/postgres-17/src/include/nodes/parsenodes.h#L2005-L2025). Parse analysis turns each raw tree into a `Query` node. That step does real work only for optimizable statements. A utility statement's raw tree is copied into `Query.utilityStmt` [parsenodes.h:2005-2009](../raw/postgres-17/src/include/nodes/parsenodes.h#L2005-L2009). The rewriter and planner consume the `Query`, and the executor never sees it [parsenodes.h#Query](../raw/postgres-17/src/include/nodes/parsenodes.h#L101-L117).

**Version notes:**
- PostgreSQL 12: Holds. `raw_parser()` returns `RawStmt`-wrapped trees, a utility statement sits in `Query.utilityStmt`, and the executor does not see the `Query` ([parser.c#raw_parser](../raw/postgres-12/src/backend/parser/parser.c#L36), [parsenodes.h#RawStmt](../raw/postgres-12/src/include/nodes/parsenodes.h#L1482-L1488), [parsenodes.h#Query](../raw/postgres-12/src/include/nodes/parsenodes.h#L100-L120)).
- PostgreSQL 14: Holds. `raw_parser()` returns `RawStmt`-wrapped trees, a utility statement's raw tree goes into `Query.utilityStmt`, and the executor never sees the `Query` ([parser.c#raw_parser](../raw/postgres-14/src/backend/parser/parser.c#L42-L86), [parsenodes.h#RawStmt](../raw/postgres-14/src/include/nodes/parsenodes.h#L1544-L1562), [parsenodes.h#Query](../raw/postgres-14/src/include/nodes/parsenodes.h#L105-L128)).
- PostgreSQL 18: Holds ([parser.c#raw_parser](../raw/postgres-18/src/backend/parser/parser.c#L34-L42), [parsenodes.h#RawStmt](../raw/postgres-18/src/include/nodes/parsenodes.h#L2069-L2089), [parsenodes.h#Query](../raw/postgres-18/src/include/nodes/parsenodes.h#L101-L117)).
- PostgreSQL 19: Holds. `raw_parser()` still returns `RawStmt`-wrapped raw trees, parse analysis still turns each into a `Query`, a utility statement still travels in `utilityStmt`, and the executor still never sees the `Query` ([parser.c#raw_parser](../raw/postgres-19/src/backend/parser/parser.c#L33-L42), [parsenodes.h#RawStmt](../raw/postgres-19/src/include/nodes/parsenodes.h#L2141-L2165), [parsenodes.h#Query](../raw/postgres-19/src/include/nodes/parsenodes.h#L100-L110)).

Related: [Grammar](#grammar), [Planner](#planner), [Utility command](#utility-command), [PlannedStmt](#plannedstmt)

### Partial index

**Aliases:** predicate index, `indpred`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A partial index covers only the table rows that satisfy a `WHERE` condition, called the predicate. It is smaller, and it is cheaper to maintain than an index on every row ([indices.sgml#indexes-partial](../raw/postgres-17/doc/src/sgml/indices.sgml#L801-L827)). The predicate is stored as an expression tree in `pg_index.indpred` ([pg_index.h:60-61](../raw/postgres-17/src/include/catalog/pg_index.h#L60-L61)). The planner may use the index only when it can prove that the query's conditions imply the predicate. `check_index_predicates()` sets `IndexOptInfo.predOK` from `predicate_implied_by()` ([indxpath.c#check_index_predicates](../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3227-L3244), [indxpath.c:3349](../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L3349)).

**Version notes:**
- PostgreSQL 12: Holds. The predicate is in `pg_index.indpred`, and `check_index_predicates()` sets `predOK` with `predicate_implied_by()` ([indices.sgml#indexes-partial](../raw/postgres-12/doc/src/sgml/indices.sgml#L757-L782), [pg_index.h:56](../raw/postgres-12/src/include/catalog/pg_index.h#L56), [indxpath.c#check_index_predicates](../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L3414), [indxpath.c:3514](../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L3514)).
- PostgreSQL 14: Holds. The predicate is stored in `pg_index.indpred`, and `check_index_predicates()` sets `predOK` from `predicate_implied_by()` ([indices.sgml#indexes-partial](../raw/postgres-14/doc/src/sgml/indices.sgml#L799-L827), [pg_index.h:59](../raw/postgres-14/src/include/catalog/pg_index.h#L59), [indxpath.c#check_index_predicates](../raw/postgres-14/src/backend/optimizer/path/indxpath.c#L3420)).
- PostgreSQL 18: Holds ([indices.sgml#indexes-partial](../raw/postgres-18/doc/src/sgml/indices.sgml#L841-L867), [pg_index.h:60-61](../raw/postgres-18/src/include/catalog/pg_index.h#L60-L61), [indxpath.c#check_index_predicates](../raw/postgres-18/src/backend/optimizer/path/indxpath.c#L3925-L3942)).
- PostgreSQL 19: Holds. The predicate is still stored in `pg_index.indpred`, and `check_index_predicates()` still sets `predOK` from `predicate_implied_by()` ([indices.sgml#indexes-partial](../raw/postgres-19/doc/src/sgml/indices.sgml#L841-L870), [pg_index.h:62](../raw/postgres-19/src/include/catalog/pg_index.h#L62), [indxpath.c#check_index_predicates](../raw/postgres-19/src/backend/optimizer/path/indxpath.c#L3940), [indxpath.c:4045](../raw/postgres-19/src/backend/optimizer/path/indxpath.c#L4045)).

Related: [Expression index](#expression-index), [pg_index](#pg_index), [IndexOptInfo](#indexoptinfo), [Planner](#planner)

### Partition bound

**Aliases:** `relpartbound`, `PartitionBoundSpec`, `FOR VALUES`, `DEFAULT` partition. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A partition bound is the rule that says which rows belong in one partition: a hash modulus and remainder, a list of values, a range from a lower to an upper bound, or `DEFAULT`. The parser builds it as a `PartitionBoundSpec` node, which carries a strategy code, a default flag and fields for each strategy ([parsenodes.h#PartitionStrategy](../raw/postgres-17/src/include/nodes/parsenodes.h#L870-L875), [parsenodes.h#PartitionBoundSpec](../raw/postgres-17/src/include/nodes/parsenodes.h#L891-L915)). `StorePartitionBound()` serializes that node with `nodeToString()` into the partition's `pg_class.relpartbound` column, a `pg_node_tree` ([heap.c#StorePartitionBound](../raw/postgres-17/src/backend/catalog/heap.c#L3571-L3608), [pg_class.h#relpartbound](../raw/postgres-17/src/include/catalog/pg_class.h#L139-L140)). `pg_get_expr()` turns it back into SQL text: `DEFAULT`, `FOR VALUES WITH (...)`, `FOR VALUES IN (...)` or `FOR VALUES FROM ... TO ...` ([ruleutils.c#T_PartitionBoundSpec](../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L10121-L10170)).

**Version notes:**
- PostgreSQL 12: Holds. The strategies are `#define` constants rather than an enum ([parsenodes.h#PartitionBoundSpec](../raw/postgres-12/src/include/nodes/parsenodes.h#L798-L827)). Storage and deparse match ([heap.c#StorePartitionBound](../raw/postgres-12/src/backend/catalog/heap.c#L3726-L3761), [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L8945-L8993)).
- PostgreSQL 14: Holds ([parsenodes.h#PartitionBoundSpec](../raw/postgres-14/src/include/nodes/parsenodes.h#L814-L843), [heap.c#StorePartitionBound](../raw/postgres-14/src/backend/catalog/heap.c#L3907-L3942), [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-14/src/backend/utils/adt/ruleutils.c#L9542-L9590)).
- PostgreSQL 18: Holds ([parsenodes.h#PartitionBoundSpec](../raw/postgres-18/src/include/nodes/parsenodes.h#L917-L941), [heap.c#StorePartitionBound](../raw/postgres-18/src/backend/catalog/heap.c#L4092-L4127), [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-18/src/backend/utils/adt/ruleutils.c#L10431-L10480)).
- PostgreSQL 19: Holds ([parsenodes.h#PartitionBoundSpec](../raw/postgres-19/src/include/nodes/parsenodes.h#L937-L961), [heap.c#StorePartitionBound](../raw/postgres-19/src/backend/catalog/heap.c#L4096-L4131), [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-19/src/backend/utils/adt/ruleutils.c#L11010-L11059)).

Related: [pg_node_tree](#pg_node_tree), [pg_class](#pg_class), [Partition pruning](#partition-pruning), [Partitioned index](#partitioned-index)

### Partition pruning

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Partition pruning skips the partitions of a partitioned table that cannot hold matching rows. It compares the query's conditions with each partition's bounds [partprune.c header](../raw/postgres-17/src/backend/partitioning/partprune.c#L3-L25). It can run at plan time, in `prune_append_rel_partitions()`. It can also run during execution when a value becomes known only then, such as a parameter; the planner then attaches pruning steps to the plan with `make_partition_pruneinfo()` [partprune.c:15-18](../raw/postgres-17/src/backend/partitioning/partprune.c#L15-L18) [execPartition.c#ExecFindMatchingSubPlans](../raw/postgres-17/src/backend/executor/execPartition.c#L2308). The `enable_partition_pruning` GUC controls both phases. Its context is `user`, so a session or transaction can `SET` it with no reload or restart. It is on by default [guc_tables.c#enable_partition_pruning](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L964-L974).

**Version notes:**
- PostgreSQL 12: Holds. Pruning runs at plan time and, for values known only at run time, at execution ([partprune.c header](../raw/postgres-12/src/backend/partitioning/partprune.c#L3-L18), [execPartition.c#ExecFindMatchingSubPlans](../raw/postgres-12/src/backend/executor/execPartition.c#L1991)). `enable_partition_pruning` is `PGC_USERSET` (session scope), default on ([guc.c#enable_partition_pruning](../raw/postgres-12/src/backend/utils/misc/guc.c#L1044-L1052)).
- PostgreSQL 14: Holds. `partprune.c` prunes at plan time and hands execution-time steps to the executor, which runs them in `ExecFindMatchingSubPlans` ([partprune.c header](../raw/postgres-14/src/backend/partitioning/partprune.c#L3-L18), [execPartition.c#ExecFindMatchingSubPlans](../raw/postgres-14/src/backend/executor/execPartition.c#L2009)). `enable_partition_pruning` is `PGC_USERSET` (session scope) and on by default ([guc.c#enable_partition_pruning](../raw/postgres-14/src/backend/utils/misc/guc.c#L1139-L1147)).
- PostgreSQL 18: Holds. `enable_partition_pruning` is still `user` context (session scope), on by default ([partprune.c header](../raw/postgres-18/src/backend/partitioning/partprune.c#L3-L25), [execPartition.c#ExecFindMatchingSubPlans](../raw/postgres-18/src/backend/executor/execPartition.c#L2498), [guc_tables.c#enable_partition_pruning](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L983-L993)).
- PostgreSQL 19: Holds. Pruning still compares conditions with partition bounds at plan time in `prune_append_rel_partitions()` and at run time from steps built by `make_partition_pruneinfo()`, and `enable_partition_pruning` is still `PGC_USERSET` (session scope), default on ([partprune.c header](../raw/postgres-19/src/backend/partitioning/partprune.c#L3-L26), [partprune.c#prune_append_rel_partitions](../raw/postgres-19/src/backend/partitioning/partprune.c#L780), [execPartition.c#ExecFindMatchingSubPlans](../raw/postgres-19/src/backend/executor/execPartition.c#L2667), [guc_parameters.dat#enable_partition_pruning](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L985-L991)). In v19, the executor's "initial" pruning runs up front in `ExecDoInitialPruning()`, before `ExecInitNode()` builds the child plans ([execPartition.c#ExecDoInitialPruning](../raw/postgres-19/src/backend/executor/execPartition.c#L1971-L1995)).

Related: [Constraint exclusion](#constraint-exclusion), [Partitionwise join](#partitionwise-join), [Planner](#planner), [Executor](#executor)

### Partitioned index

**Aliases:** `RELKIND_PARTITIONED_INDEX`, relkind `I`, parent index, `ALTER INDEX ... ATTACH PARTITION`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A partitioned index is the index you create on a partitioned table. It has relkind `I` and holds no data itself; each partition gets its own ordinary index, and those child indexes do the real work ([pg_class.h#RELKIND_PARTITIONED_INDEX](../raw/postgres-17/src/include/catalog/pg_class.h#L164-L173), [pg_class.h#RELKIND_HAS_STORAGE](../raw/postgres-17/src/include/catalog/pg_class.h#L191-L207)). `index_create()` links a child index to its parent with a `pg_inherits` row ([index.c:1065-1069](../raw/postgres-17/src/backend/catalog/index.c#L1065-L1069)). `IndexSetParentIndex()` makes the same link when an existing index is attached ([indexcmds.c#IndexSetParentIndex](../raw/postgres-17/src/backend/commands/indexcmds.c#L4322-L4363)). `CREATE INDEX CONCURRENTLY` on a partitioned table fails with an error ([indexcmds.c:724-730](../raw/postgres-17/src/backend/commands/indexcmds.c#L724-L730)). `REINDEX` of a partitioned index goes through `ReindexPartitions()`, which reindexes each partition ([indexcmds.c#ReindexPartitions](../raw/postgres-17/src/backend/commands/indexcmds.c#L3227-L3233)).

**Version notes:**
- PostgreSQL 12: Differs. Relkind `I` exists and has no storage ([pg_class.h#RELKIND_HAS_STORAGE](../raw/postgres-12/src/include/catalog/pg_class.h#L163-L192)), and children link through `pg_inherits` ([index.c:1007](../raw/postgres-12/src/backend/catalog/index.c#L1007)). `REINDEX` of a partitioned index fails with "not yet implemented" ([indexcmds.c#ReindexPartitionedIndex](../raw/postgres-12/src/backend/commands/indexcmds.c#L3384-L3396)). `CREATE INDEX CONCURRENTLY` is also rejected ([indexcmds.c:612-616](../raw/postgres-12/src/backend/commands/indexcmds.c#L612-L616)).
- PostgreSQL 14: Holds as in 17. `ReindexPartitions()` exists ([indexcmds.c#ReindexPartitions](../raw/postgres-14/src/backend/commands/indexcmds.c#L3157-L3164)), and `CREATE INDEX CONCURRENTLY` is still rejected ([indexcmds.c:724](../raw/postgres-14/src/backend/commands/indexcmds.c#L724)).
- PostgreSQL 18: Holds ([pg_class.h#RELKIND_PARTITIONED_INDEX](../raw/postgres-18/src/include/catalog/pg_class.h#L176), [indexcmds.c:738](../raw/postgres-18/src/backend/commands/indexcmds.c#L738), [indexcmds.c#ReindexPartitions](../raw/postgres-18/src/backend/commands/indexcmds.c#L3366)).
- PostgreSQL 19: Holds ([pg_class.h#RELKIND_PARTITIONED_INDEX](../raw/postgres-19/src/include/catalog/pg_class.h#L180), [indexcmds.c:742](../raw/postgres-19/src/backend/commands/indexcmds.c#L742), [indexcmds.c#ReindexPartitions](../raw/postgres-19/src/backend/commands/indexcmds.c#L3392)).

Related: [pg_class](#pg_class), [pg_index](#pg_index), [Partition bound](#partition-bound), [REINDEX](#reindex), [CONCURRENTLY](#concurrently)

### Partitionwise join

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A partitionwise join splits one join of two partitioned tables into joins between matching partitions, then appends the results. It needs both tables to use the same partitioning scheme, with an equi-join on the partition keys [joinrels.c#try_partitionwise_join](../raw/postgres-17/src/backend/optimizer/path/joinrels.c#L1458-L1477). `build_joinrel_partition_info()` checks those conditions and returns early when the feature is off [relnode.c#build_joinrel_partition_info](../raw/postgres-17/src/backend/optimizer/util/relnode.c#L2016-L2045). The `enable_partitionwise_join` GUC has context `user`, so a session or transaction can `SET` it with no reload or restart. It is off by default [guc_tables.c#enable_partitionwise_join](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L924-L932).

**Version notes:**
- PostgreSQL 12: Holds. It needs the same partitioning scheme and an equi-join on the keys, and `build_joinrel_partition_info()` returns early when the feature is off ([joinrels.c#try_partitionwise_join](../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L1320-L1336), [relnode.c#build_joinrel_partition_info](../raw/postgres-12/src/backend/optimizer/util/relnode.c#L1613-L1625)). `enable_partitionwise_join` is `PGC_USERSET` (session scope), default off ([guc.c#enable_partitionwise_join](../raw/postgres-12/src/backend/utils/misc/guc.c#L1004-L1010)).
- PostgreSQL 14: Holds. `try_partitionwise_join()` joins matching partitions, and `build_joinrel_partition_info()` returns early when the feature is off ([joinrels.c#try_partitionwise_join](../raw/postgres-14/src/backend/optimizer/path/joinrels.c#L1358-L1552), [relnode.c#build_joinrel_partition_info](../raw/postgres-14/src/backend/optimizer/util/relnode.c#L1656-L1664)). `enable_partitionwise_join` is `PGC_USERSET` (session scope) and off by default ([guc.c#enable_partitionwise_join](../raw/postgres-14/src/backend/utils/misc/guc.c#L1099-L1107)).
- PostgreSQL 18: Holds. `enable_partitionwise_join` is still `user` context (session scope), off by default ([joinrels.c#try_partitionwise_join](../raw/postgres-18/src/backend/optimizer/path/joinrels.c#L1401-L1420), [relnode.c#build_joinrel_partition_info](../raw/postgres-18/src/backend/optimizer/util/relnode.c#L1990-L2019), [guc_tables.c#enable_partitionwise_join](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L943-L951)).
- PostgreSQL 19: Holds for the conditions (same partitioning scheme, equi-join on the keys), and `enable_partitionwise_join` is still `PGC_USERSET` (session scope), default off ([joinrels.c#try_partitionwise_join](../raw/postgres-19/src/backend/optimizer/path/joinrels.c#L1591-L1611), [guc_parameters.dat#enable_partitionwise_join](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1000-L1005)). The gate changed: `build_joinrel_partition_info()` now tests the joinrel's `pgs_mask` bit `PGS_CONSIDER_PARTITIONWISE`. The GUC only sets that bit in the planner's default mask, and extension hooks may change it per relation ([relnode.c#build_joinrel_partition_info](../raw/postgres-19/src/backend/optimizer/util/relnode.c#L2150-L2162), [planner.c:523-524](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L523-L524), [pathnodes.h:24-35](../raw/postgres-19/src/include/nodes/pathnodes.h#L24-L35)).

Related: [Partition pruning](#partition-pruning), [RelOptInfo](#reloptinfo), [Planner](#planner)

### Path

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A `Path` is one candidate way to produce a relation's rows, such as a sequential scan, an index scan, or a particular join method. It carries estimated rows, `startup_cost`, `total_cost`, and sort order (`pathkeys`) [pathnodes.h#Path](../raw/postgres-17/src/include/nodes/pathnodes.h#L1622-L1669). The planner collects paths in each `RelOptInfo.pathlist`. `add_path()` keeps a new path only if no existing path beats it on cost, sort order, row count, parameterization, and parallel safety, and it discards the old paths the new one beats [pathnode.c#add_path](../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L361-L420). A path is not yet executable. `create_plan()` turns the winning path into a `Plan` [planner.c:418-422](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L418-L422).

**Version notes:**
- PostgreSQL 12: Holds. `Path` has rows, costs and `pathkeys`; `add_path()` keeps a path that wins on cost, sort order, rows, parameterization or parallel safety; `create_plan()` turns the winner into a `Plan` ([pathnodes.h#Path](../raw/postgres-12/src/include/nodes/pathnodes.h#L1104-L1126), [pathnode.c#add_path](../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L368-L380), [planner.c:411-413](../raw/postgres-12/src/backend/optimizer/plan/planner.c#L411-L413)).
- PostgreSQL 14: Holds. `Path` carries rows, costs and `pathkeys`, `add_path()` keeps only non-dominated paths, and `standard_planner()` turns the cheapest final path into a plan with `create_plan()` ([pathnodes.h#Path](../raw/postgres-14/src/include/nodes/pathnodes.h#L1174-L1195), [pathnode.c#add_path](../raw/postgres-14/src/backend/optimizer/util/pathnode.c#L427-L629), [planner.c:408-410](../raw/postgres-14/src/backend/optimizer/plan/planner.c#L408-L410)).
- PostgreSQL 18: Holds, with one new field. `Path` also carries `disabled_nodes` ([pathnodes.h#Path](../raw/postgres-18/src/include/nodes/pathnodes.h#L1753-L1801)). `add_path()` now compares disabled-node counts before costs and keeps `pathlist` sorted by `disabled_nodes` and then `total_cost` ([pathnode.c#add_path](../raw/postgres-18/src/backend/optimizer/util/pathnode.c#L391-L440)). `create_plan()` still turns the winner into a `Plan` ([planner.c:450-454](../raw/postgres-18/src/backend/optimizer/plan/planner.c#L450-L454)).
- PostgreSQL 19: Holds. A `Path` still carries rows, `startup_cost`, `total_cost` and `pathkeys`, `add_path()` still keeps only undominated paths, and `create_plan()` still turns the winner into a `Plan` ([pathnodes.h#Path](../raw/postgres-19/src/include/nodes/pathnodes.h#L1964-L2012), [pathnode.c#add_path](../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L385-L411), [planner.c:537-539](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L537-L539)). In v19, `Path` also has `disabled_nodes`, and `add_path()` compares that count before cost. A path of a strategy that a GUC or a `pgs_mask` disables is marked disabled rather than always removed ([pathnodes.h#Path](../raw/postgres-19/src/include/nodes/pathnodes.h#L2006), [pathnode.c#add_path](../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L395-L405), [pathnodes.h:37-40](../raw/postgres-19/src/include/nodes/pathnodes.h#L37-L40)).

Related: [RelOptInfo](#reloptinfo), [Cost](#cost), [Planner](#planner), [PlannedStmt](#plannedstmt)

### Pending list

**Aliases:** GIN fast update, `fastupdate`, `gin_pending_list_limit`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The pending list is an unsorted list of GIN pages where new entries wait before being merged into the main GIN structure. This makes inserts faster, but every search must also scan the list ([gin.sgml](../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L528), [ginfast.c](../raw/postgres-17/src/backend/access/gin/ginfast.c#L1-L7)). The `fastupdate` storage parameter turns it on or off; the default is on ([reloptions.c:123-131](../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131)). An insert calls `ginInsertCleanup()` once the list grows past the index's own `gin_pending_list_limit` storage parameter, or, when that is unset, the `gin_pending_list_limit` GUC ([ginfast.c:458-471](../raw/postgres-17/src/backend/access/gin/ginfast.c#L458-L471), [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-17/src/include/access/gin_private.h#L39-L45)). The GUC defaults to 4096 kB. Its context is `user`, so a session or transaction can `SET` it with no reload or restart ([guc_tables.c:3577-3585](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3577-L3585)). VACUUM, autoanalyze and `gin_clean_pending_list()` also empty it ([gin.sgml](../raw/postgres-17/doc/src/sgml/gin.sgml#L508-L515)). The list's head, tail and size live in the GIN metapage ([ginblock.h#GinMetaPageData](../raw/postgres-17/src/include/access/ginblock.h#L55-L75)).

**Version notes:**
- PostgreSQL 12: Holds. `fastupdate` defaults on, and cleanup starts past the index's `gin_pending_list_limit` or the GUC ([reloptions.c:125-133](../raw/postgres-12/src/backend/access/common/reloptions.c#L125-L133), [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-12/src/include/access/gin_private.h#L35-L39), [ginfast.c:448-461](../raw/postgres-12/src/backend/access/gin/ginfast.c#L448-L461)). The GUC is `PGC_USERSET` (session scope), default 4096 kB ([guc.c#gin_pending_list_limit](../raw/postgres-12/src/backend/utils/misc/guc.c#L3176-L3182)). VACUUM, autoanalyze and `gin_clean_pending_list()` also flush it ([gin.sgml#gin-fast-update](../raw/postgres-12/doc/src/sgml/gin.sgml#L470-L496)). Changing `fastupdate` takes `AccessExclusiveLock` in 12 ([reloptions.c:125-133](../raw/postgres-12/src/backend/access/common/reloptions.c#L125-L133)).
- PostgreSQL 14: Holds. `fastupdate` defaults on, an insert runs `ginInsertCleanup()` once the list passes the index's `gin_pending_list_limit` or the GUC, and VACUUM, autoanalyze and `gin_clean_pending_list()` also empty it ([gin.sgml](../raw/postgres-14/doc/src/sgml/gin.sgml#L503-L530), [reloptions.c:127-132](../raw/postgres-14/src/backend/access/common/reloptions.c#L127-L132), [ginfast.c:457-470](../raw/postgres-14/src/backend/access/gin/ginfast.c#L457-L470), [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-14/src/include/access/gin_private.h#L38-L44)). The GUC defaults to 4096 kB and is `PGC_USERSET` (session scope) ([guc.c:3492-3498](../raw/postgres-14/src/backend/utils/misc/guc.c#L3492-L3498)).
- PostgreSQL 18: Holds. `fastupdate` defaults on, and `gin_pending_list_limit` is still `user` context (session scope) with default 4096 kB ([ginfast.c:458-471](../raw/postgres-18/src/backend/access/gin/ginfast.c#L458-L471), [reloptions.c:123-131](../raw/postgres-18/src/backend/access/common/reloptions.c#L123-L131), [guc_tables.c:3784-3792](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3784-L3792), [ginblock.h#GinMetaPageData](../raw/postgres-18/src/include/access/ginblock.h#L55-L75)).
- PostgreSQL 19: Holds. `fastupdate` still defaults on, inserts still call `ginInsertCleanup()` past the index or GUC limit, `gin_pending_list_limit` is still `PGC_USERSET` (session scope) with default 4096 kB, VACUUM, autoanalyze and `gin_clean_pending_list()` still empty the list, and the metapage still tracks head, tail and size ([gin.sgml](../raw/postgres-19/doc/src/sgml/gin.sgml#L504-L530), [ginfast.c](../raw/postgres-19/src/backend/access/gin/ginfast.c#L1-L7), [reloptions.c:130](../raw/postgres-19/src/backend/access/common/reloptions.c#L130), [ginfast.c:456-471](../raw/postgres-19/src/backend/access/gin/ginfast.c#L456-L471), [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-19/src/include/access/gin_private.h#L41-L47), [guc_parameters.dat#gin_pending_list_limit](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1198-L1205), [ginblock.h#GinMetaPageData](../raw/postgres-19/src/include/access/ginblock.h#L55-L75)).

Related: [GIN](#gin), [Metapage](#metapage), [VACUUM](#vacuum), [GUC context](#guc-context)

### pg_attribute

**Aliases:** `Form_pg_attribute`, `FormData_pg_attribute`, `attnum`, `atttypid`, `attlen`, `attalign`, `attisdropped`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_attribute` is the system [catalog](#catalog) with one row per column of every relation. That includes indexes and every other object that has a [pg_class](#pg_class) row. "Attribute" is the historical word for column ([catalogs.sgml#catalog-pg-attribute](../raw/postgres-17/doc/src/sgml/catalogs.sgml#L1101-L1121)). In C it is `FormData_pg_attribute`, reached through the pointer type `Form_pg_attribute` ([pg_attribute.h#FormData_pg_attribute](../raw/postgres-17/src/include/catalog/pg_attribute.h#L37), [pg_attribute.h:209](../raw/postgres-17/src/include/catalog/pg_attribute.h#L209)). Size and layout work reads it constantly. `attlen` and `attalign` copy the type's length and [alignment](#alignment) from `pg_type`, and they are kept even after a column is dropped, so old tuples can still be decoded. `attnum` numbers user columns from 1, and `attisdropped` marks dropped columns that still take space in old rows ([pg_attribute.h:44-74](../raw/postgres-17/src/include/catalog/pg_attribute.h#L44-L74), [pg_attribute.h:106-109](../raw/postgres-17/src/include/catalog/pg_attribute.h#L106-L109), [pg_attribute.h:144-145](../raw/postgres-17/src/include/catalog/pg_attribute.h#L144-L145)). In PostgreSQL 17, `attstattarget` is nullable and lives in the variable-length part of the row ([pg_attribute.h:168-176](../raw/postgres-17/src/include/catalog/pg_attribute.h#L168-L176)).

**Version notes:**
- PostgreSQL 12: Holds ([pg_attribute.h#FormData_pg_attribute](../raw/postgres-12/src/include/catalog/pg_attribute.h#L37), [pg_attribute.h:129](../raw/postgres-12/src/include/catalog/pg_attribute.h#L129), [pg_attribute.h:147](../raw/postgres-12/src/include/catalog/pg_attribute.h#L147)). `attstattarget` is a fixed `int32` near the start of the row, default -1 ([pg_attribute.h:52-58](../raw/postgres-12/src/include/catalog/pg_attribute.h#L52-L58)).
- PostgreSQL 14: Same layout as 12 for `attstattarget` ([pg_attribute.h:56-62](../raw/postgres-14/src/include/catalog/pg_attribute.h#L56-L62), [pg_attribute.h:118](../raw/postgres-14/src/include/catalog/pg_attribute.h#L118), [pg_attribute.h:154](../raw/postgres-14/src/include/catalog/pg_attribute.h#L154)).
- PostgreSQL 18: `attcacheoff` is no longer a catalog column. The cached tuple offset lives in the in-memory `CompactAttribute` of a tuple descriptor ([tupdesc.h#CompactAttribute](../raw/postgres-18/src/include/access/tupdesc.h#L60-L75), [pg_attribute.h:100](../raw/postgres-18/src/include/catalog/pg_attribute.h#L100), [pg_attribute.h:138](../raw/postgres-18/src/include/catalog/pg_attribute.h#L138)).
- PostgreSQL 19: Same as 18 ([pg_attribute.h#FormData_pg_attribute](../raw/postgres-19/src/include/catalog/pg_attribute.h#L39), [pg_attribute.h:102](../raw/postgres-19/src/include/catalog/pg_attribute.h#L102), [pg_attribute.h:206](../raw/postgres-19/src/include/catalog/pg_attribute.h#L206)).

Related: [Catalog](#catalog), [pg_class](#pg_class), [Alignment](#alignment), [Tuple](#tuple)

### pg_cast

**Aliases:** cast, binary-coercible, `castmethod`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_cast` is the catalog of data-type conversions. Each row names a source type, a target type, and a cast function, where function 0 means the two types are binary-coercible ([pg_cast.h#FormData_pg_cast](../raw/postgres-17/src/include/catalog/pg_cast.h#L32-L50)). `castmethod` records whether the cast uses a function (`f`), binary compatibility (`b`), or the types' text input and output functions (`i`) ([pg_cast.h#CoercionMethod](../raw/postgres-17/src/include/catalog/pg_cast.h#L87-L92)). The distinction matters for `ALTER COLUMN ... TYPE`. `ATColumnChangeRequiresRewrite` skips the table rewrite for a binary-coercible change and requires a rewrite for most function casts ([tablecmds.c#ATColumnChangeRequiresRewrite](../raw/postgres-17/src/backend/commands/tablecmds.c#L13100-L13152)). `int4` to `int8` is a function cast ([pg_cast.dat:41-42](../raw/postgres-17/src/include/catalog/pg_cast.dat#L41-L42)).

**Version notes:**
- PostgreSQL 12: Holds. `pg_cast` has `castfunc` and `castmethod` with `f`/`b`/`i`, `ATColumnChangeRequiresRewrite` exists, and `int4` to `int8` is a function cast ([pg_cast.h#FormData_pg_cast](../raw/postgres-12/src/include/catalog/pg_cast.h#L30-L49), [pg_cast.h#CoercionMethod](../raw/postgres-12/src/include/catalog/pg_cast.h#L83-L85), [tablecmds.c#ATColumnChangeRequiresRewrite](../raw/postgres-12/src/backend/commands/tablecmds.c#L10603), [pg_cast.dat:41](../raw/postgres-12/src/include/catalog/pg_cast.dat#L41)).
- PostgreSQL 14: Holds. `pg_cast` rows carry `castfunc` and `castmethod` (`f`, `b`, `i`), `ATColumnChangeRequiresRewrite` decides rewrites, and `int4` to `int8` is a function cast ([pg_cast.h#FormData_pg_cast](../raw/postgres-14/src/include/catalog/pg_cast.h#L32-L50), [pg_cast.h#CoercionMethod](../raw/postgres-14/src/include/catalog/pg_cast.h#L87-L92), [tablecmds.c#ATColumnChangeRequiresRewrite](../raw/postgres-14/src/backend/commands/tablecmds.c#L12183-L12222), [pg_cast.dat:41](../raw/postgres-14/src/include/catalog/pg_cast.dat#L41)).
- PostgreSQL 18: Holds ([pg_cast.h#FormData_pg_cast](../raw/postgres-18/src/include/catalog/pg_cast.h#L32-L50), [pg_cast.h#CoercionMethod](../raw/postgres-18/src/include/catalog/pg_cast.h#L87-L92), [tablecmds.c#ATColumnChangeRequiresRewrite](../raw/postgres-18/src/backend/commands/tablecmds.c#L14686-L14738), [pg_cast.dat:41-42](../raw/postgres-18/src/include/catalog/pg_cast.dat#L41-L42)).
- PostgreSQL 19: Holds. `pg_cast` still stores source, target, function and `castmethod` (`f`, `b`, `i`), `ATColumnChangeRequiresRewrite()` still decides whether `ALTER COLUMN ... TYPE` rewrites, and `int4` to `int8` is still a function cast ([pg_cast.h#FormData_pg_cast](../raw/postgres-19/src/include/catalog/pg_cast.h#L34-L52), [pg_cast.h#CoercionMethod](../raw/postgres-19/src/include/catalog/pg_cast.h#L91-L96), [tablecmds.c#ATColumnChangeRequiresRewrite](../raw/postgres-19/src/backend/commands/tablecmds.c#L15111), [pg_cast.dat:41-42](../raw/postgres-19/src/include/catalog/pg_cast.dat#L41-L42)).

Related: [Table rewrite](#table-rewrite), [Catalog](#catalog), [fmgr](#fmgr)

### pg_class

**Aliases:** relation, `relkind`, `Form_pg_class`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_class` is the catalog with one row for every relation: tables, indexes, sequences, views, materialized views, TOAST tables, composite types, foreign tables, and partitioned tables and indexes ([pg_class.h:32](../raw/postgres-17/src/include/catalog/pg_class.h#L32), [pg_class.h:164-173](../raw/postgres-17/src/include/catalog/pg_class.h#L164-L173)). Its columns include the access method `relam`, the file identifier `relfilenode`, the planner's size estimates `relpages` and `reltuples`, and `relkind` ([pg_class.h:52-84](../raw/postgres-17/src/include/catalog/pg_class.h#L52-L84)). In C, a row is a `FormData_pg_class` read through `Form_pg_class`. A relcache entry keeps a copy in `rd_rel` ([pg_class.h:142-153](../raw/postgres-17/src/include/catalog/pg_class.h#L142-L153), [rel.h:111](../raw/postgres-17/src/include/utils/rel.h#L111)).

**Version notes:**
- PostgreSQL 12: Holds. `pg_class` has `relam`, `relfilenode`, `relpages`, `reltuples` and `relkind`, the same relkinds including partitioned tables and indexes, and the relcache copy `rd_rel` ([pg_class.h:29](../raw/postgres-12/src/include/catalog/pg_class.h#L29), [pg_class.h:52-81](../raw/postgres-12/src/include/catalog/pg_class.h#L52-L81), [pg_class.h:154-163](../raw/postgres-12/src/include/catalog/pg_class.h#L154-L163), [pg_class.h:139-150](../raw/postgres-12/src/include/catalog/pg_class.h#L139-L150), [rel.h:84](../raw/postgres-12/src/include/utils/rel.h#L84)).
- PostgreSQL 14: Holds. `pg_class` has one row per relation with `relam`, `relfilenode`, `relpages`, `reltuples` and `relkind`; the relcache copy is `rd_rel` ([pg_class.h:32](../raw/postgres-14/src/include/catalog/pg_class.h#L32), [pg_class.h:52-84](../raw/postgres-14/src/include/catalog/pg_class.h#L52-L84), [pg_class.h:153-173](../raw/postgres-14/src/include/catalog/pg_class.h#L153-L173), [rel.h:109](../raw/postgres-14/src/include/utils/rel.h#L109)).
- PostgreSQL 18: Holds. `pg_class` gains `relallfrozen`, the estimated count of all-frozen pages, next to `relallvisible` ([pg_class.h:52-87](../raw/postgres-18/src/include/catalog/pg_class.h#L52-L87), [pg_class.h:145-156](../raw/postgres-18/src/include/catalog/pg_class.h#L145-L156), [rel.h:111](../raw/postgres-18/src/include/utils/rel.h#L111)).
- PostgreSQL 19: Holds. It still has one row per relation of the same kinds, with `relam`, `relfilenode`, `relpages`, `reltuples` and `relkind`, and the relcache still keeps it in `rd_rel` ([pg_class.h:34](../raw/postgres-19/src/include/catalog/pg_class.h#L34), [pg_class.h:55-89](../raw/postgres-19/src/include/catalog/pg_class.h#L55-L89), [pg_class.h:171-180](../raw/postgres-19/src/include/catalog/pg_class.h#L171-L180), [rel.h:111](../raw/postgres-19/src/include/utils/rel.h#L111)). v19 adds `relallfrozen`, an estimate of all-frozen blocks. Autovacuum's insert threshold uses it to scale by the unfrozen fraction of the table ([pg_class.h:72-73](../raw/postgres-19/src/include/catalog/pg_class.h#L72-L73), [autovacuum.c#relation_needs_vacanalyze](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3281-L3290)).

Related: [reltuples and relpages](#reltuples-and-relpages), [Relfilenumber](#relfilenumber), [pg_index](#pg_index), [Relcache](#relcache), [OID](#oid)

### pg_freespacemap

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_freespacemap` is a contrib extension that reads a relation's free space map ([pg_freespacemap.control](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.control#L1-L2)). `pg_freespace(rel, blkno)` returns `GetRecordedFreeSpace()` for one block. The one-argument form loops over every block of the main fork ([pg_freespacemap.c#pg_freespace](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L48), [pg_freespacemap--1.1.sql](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L20)). The value is what the FSM recorded, not a live page measurement. The FSM stores one byte per page, so free space is rounded to `BLCKSZ / 256` steps ([freespace.c](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L36-L65), [freespace.c#GetRecordedFreeSpace](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L249-L269)). The install script revokes both functions from `PUBLIC` ([pg_freespacemap--1.1.sql](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L23-L25)). The default version is 1.2, and its update script grants `EXECUTE` on both functions to the `pg_stat_scan_tables` role ([pg_freespacemap.control](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.control#L3), [pg_freespacemap--1.1--1.2.sql](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1--1.2.sql#L6-L7)).

**Version notes:**
- PostgreSQL 12: Holds. `pg_freespace` returns `GetRecordedFreeSpace()` for a block, the one-argument form loops the main fork, and both functions are revoked from `PUBLIC` ([pg_freespacemap.control](../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.control#L1-L5), [pg_freespacemap.c#pg_freespace](../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c#L24-L38), [pg_freespacemap--1.1.sql](../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L25)).
- PostgreSQL 14: Holds. `pg_freespace` reads the recorded FSM value per block, the one-argument form loops over the main fork, and the install script revokes both from `PUBLIC` ([pg_freespacemap.control](../raw/postgres-14/contrib/pg_freespacemap/pg_freespacemap.control#L1-L2), [pg_freespacemap.c#pg_freespace](../raw/postgres-14/contrib/pg_freespacemap/pg_freespacemap.c#L25-L49), [pg_freespacemap--1.1.sql](../raw/postgres-14/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L25), [freespace.c#GetRecordedFreeSpace](../raw/postgres-14/src/backend/storage/freespace/freespace.c#L252-L269)).
- PostgreSQL 18: Holds ([pg_freespacemap.c#pg_freespace](../raw/postgres-18/contrib/pg_freespacemap/pg_freespacemap.c#L27-L51), [pg_freespacemap--1.1.sql](../raw/postgres-18/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L25), [freespace.c#GetRecordedFreeSpace](../raw/postgres-18/src/backend/storage/freespace/freespace.c#L249-L269)).
- PostgreSQL 19: Holds. `pg_freespace(rel, blkno)` still returns `GetRecordedFreeSpace()`, values are still FSM-recorded rather than live, and the 1.1 script still revokes both functions from `PUBLIC` ([pg_freespacemap.control](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap.control#L1-L2), [pg_freespacemap.c#pg_freespace](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap.c#L28-L49), [pg_freespacemap--1.1.sql](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L23-L25)). Later scripts change access and the one-argument form: 1.2 grants both functions to `pg_stat_scan_tables`, and 1.3 rewrites the one-argument form as a SQL function over `pg_relation_size()` ([pg_freespacemap--1.1--1.2.sql](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap--1.1--1.2.sql#L6-L7), [pg_freespacemap--1.2--1.3.sql](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap--1.2--1.3.sql#L6-L13), [pg_freespacemap.control](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap.control#L3)).

Related: [Free space map](#free-space-map), [BLCKSZ](#blcksz), [Contrib](#contrib)

### pg_index

**Aliases:** `Form_pg_index`, `indisvalid`, `indisready`, `indislive`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_index` holds the index-specific facts for each index. Its row links the index's `pg_class` row (`indexrelid`) to the table it indexes (`indrelid`) ([pg_index.h#FormData_pg_index](../raw/postgres-17/src/include/catalog/pg_index.h#L29-L63)). Three flags track an index's lifecycle. `indislive` says whether it exists at all, `indisready` whether it receives inserts, and `indisvalid` whether queries may use it ([pg_index.h:42-45](../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)). The row also stores the key columns `indkey`, the operator classes, the expression trees `indexprs`, and a partial-index predicate `indpred` ([pg_index.h:49-61](../raw/postgres-17/src/include/catalog/pg_index.h#L49-L61)). The relcache exposes the row as `rd_index` ([rel.h:192](../raw/postgres-17/src/include/utils/rel.h#L192)).

**Version notes:**
- PostgreSQL 12: Holds. `indexrelid`, `indrelid`, the three state flags, `indkey`, `indclass`, `indexprs` and `indpred` are present, and the relcache exposes `rd_index` ([pg_index.h#FormData_pg_index](../raw/postgres-12/src/include/catalog/pg_index.h#L29-L59), [rel.h:144](../raw/postgres-12/src/include/utils/rel.h#L144)).
- PostgreSQL 14: Holds. `pg_index` links `indexrelid` to `indrelid`, has the three lifecycle flags and `indkey`, `indexprs` and `indpred`, and the relcache exposes it as `rd_index` ([pg_index.h#FormData_pg_index](../raw/postgres-14/src/include/catalog/pg_index.h#L29-L60), [pg_index.h:41-44](../raw/postgres-14/src/include/catalog/pg_index.h#L41-L44), [rel.h:187](../raw/postgres-14/src/include/utils/rel.h#L187)).
- PostgreSQL 18: Holds ([pg_index.h#FormData_pg_index](../raw/postgres-18/src/include/catalog/pg_index.h#L29-L63), [rel.h:192](../raw/postgres-18/src/include/utils/rel.h#L192)).
- PostgreSQL 19: Holds. `pg_index` still links `indexrelid` to `indrelid` and holds the three state flags, `indkey`, `indexprs` and `indpred`, and the relcache still exposes it as `rd_index` ([pg_index.h#FormData_pg_index](../raw/postgres-19/src/include/catalog/pg_index.h#L31-L65), [pg_index.h:44-47](../raw/postgres-19/src/include/catalog/pg_index.h#L44-L47), [rel.h:192](../raw/postgres-19/src/include/utils/rel.h#L192)).

Related: [Invalid index](#invalid-index), [CONCURRENTLY](#concurrently), [Partial index](#partial-index), [Expression index](#expression-index), [pg_class](#pg_class)

### pg_node_tree

**Aliases:** serialized node tree, `nodeToString()`, `stringToNode()`, `pg_get_expr()`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_node_tree` is the catalog column type that stores an internal expression or node tree as text. Columns such as `pg_class.relpartbound` use it ([pg_class.h#relpartbound](../raw/postgres-17/src/include/catalog/pg_class.h#L139-L140)). The server writes the text with `nodeToString()` and reads it back with `stringToNode()` ([outfuncs.c#nodeToString](../raw/postgres-17/src/backend/nodes/outfuncs.c#L786-L794), [read.c#stringToNode](../raw/postgres-17/src/backend/nodes/read.c#L86-L93)). The type accepts no input, because the functions that read it do not trust malformed text; output is allowed ([pseudotypes.c#pg_node_tree](../raw/postgres-17/src/backend/utils/adt/pseudotypes.c#L322-L335)). To read a value as SQL, call `pg_get_expr()`. It works for any node tree that can appear in a catalog `pg_node_tree` column, except query trees ([ruleutils.c#pg_get_expr](../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L2618-L2631)). The per-node read and write code is generated by `gen_node_support.pl` and included into `outfuncs.c` ([outfuncs.c:373](../raw/postgres-17/src/backend/nodes/outfuncs.c#L373)).

**Version notes:**
- PostgreSQL 12: Holds, but the per-node output functions are written by hand in `outfuncs.c`, which has no generated include ([outfuncs.c#nodeToString](../raw/postgres-12/src/backend/nodes/outfuncs.c#L4301)). Input is rejected by `pg_node_tree_in()` ([pseudotypes.c#pg_node_tree_in](../raw/postgres-12/src/backend/utils/adt/pseudotypes.c#L266-L284)).
- PostgreSQL 14: Holds, with hand-written output functions as in 12 ([outfuncs.c#nodeToString](../raw/postgres-14/src/backend/nodes/outfuncs.c#L4546), [read.c#stringToNode](../raw/postgres-14/src/backend/nodes/read.c#L89), [pseudotypes.c#pg_node_tree](../raw/postgres-14/src/backend/utils/adt/pseudotypes.c#L340)).
- PostgreSQL 18: Holds ([outfuncs.c:379](../raw/postgres-18/src/backend/nodes/outfuncs.c#L379), [outfuncs.c#nodeToString](../raw/postgres-18/src/backend/nodes/outfuncs.c#L805), [pseudotypes.c#pg_node_tree](../raw/postgres-18/src/backend/utils/adt/pseudotypes.c#L326)).
- PostgreSQL 19: Holds ([outfuncs.c:378](../raw/postgres-19/src/backend/nodes/outfuncs.c#L378), [outfuncs.c#nodeToString](../raw/postgres-19/src/backend/nodes/outfuncs.c#L811), [ruleutils.c#pg_get_expr](../raw/postgres-19/src/backend/utils/adt/ruleutils.c#L3036)).

Related: [Partition bound](#partition-bound), [pg_class](#pg_class), [pg_index](#pg_index), [Parse tree](#parse-tree)

### pg_plan_advice

**Aliases:** plan advice, advice string, advice tag, `EXPLAIN (PLAN_ADVICE)`, `pg_plan_advice.advice`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

In PostgreSQL 19, `pg_plan_advice` is a contrib module that describes key planner decisions in a small "plan advice" language, and can feed that advice back in to steer later planning ([pgplanadvice.sgml:10-15](../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml#L10-L15)). An advice string is a list of items; each applies a tag such as `SEQ_SCAN` or `HASH_JOIN` to one or more relation identifiers ([README:18-27](../raw/postgres-19/contrib/pg_plan_advice/README#L18-L27)). The module must be loaded through `shared_preload_libraries`, `session_preload_libraries` or `LOAD` ([pgplanadvice.sgml:31-40](../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml#L31-L40)). Loading it registers the `EXPLAIN (PLAN_ADVICE)` option and installs planner hooks ([pg_plan_advice.c#_PG_init](../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L127-L138), [pgpa_planner.c#pgpa_planner_install_hooks](../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L180-L194)). Those hooks enforce advice by adjusting each relation's [pgs_mask](#pgs_mask) ([pgpa_planner.c#pgpa_build_simple_rel](../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L436-L445)). Supplied advice lives in `pg_plan_advice.advice`, a `user`-context setting, so it is set per session ([pg_plan_advice.c#_PG_init](../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L70-L79)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The contrib build list has no `pg_plan_advice` ([contrib/Makefile:31-37](../raw/postgres-12/contrib/Makefile#L31-L37)).
- PostgreSQL 14: Not present in PostgreSQL 14 ([contrib/Makefile:32-38](../raw/postgres-14/contrib/Makefile#L32-L38)).
- PostgreSQL 17: Not present in PostgreSQL 17 ([contrib/Makefile:32-38](../raw/postgres-17/contrib/Makefile#L32-L38)).
- PostgreSQL 18: Not present in PostgreSQL 18 ([contrib/Makefile:32-40](../raw/postgres-18/contrib/Makefile#L32-L40)).
- PostgreSQL 19: Present, as described above. The build list also adds a sibling module, `pg_stash_advice` ([contrib/Makefile:36-40](../raw/postgres-19/contrib/Makefile#L36-L40)).

Related: [pgs_mask](#pgs_mask), [Planner](#planner), [Hook](#hook), [Contrib](#contrib), [EXPLAIN](#explain)

### pg_proc

**Aliases:** `Form_pg_proc`, function catalog, `prokind`, `provolatile`, `proleakproof`, `procost`, `prorows`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_proc` is the system catalog with one row per function, procedure, aggregate or window function ([pg_proc.h#FormData_pg_proc](../raw/postgres-17/src/include/catalog/pg_proc.h#L30-L129)). Its columns carry the properties the planner and executor read. They include the estimated cost (`procost`), estimated rows for set-returning functions (`prorows`), the object kind (`prokind`), `SECURITY DEFINER` (`prosecdef`), leakproofness (`proleakproof`), volatility (`provolatile`) and parallel safety (`proparallel`) ([pg_proc.h#FormData_pg_proc](../raw/postgres-17/src/include/catalog/pg_proc.h#L46-L77)). `prokind` is `f`, `a`, `w` or `p`. `provolatile` is `i` (immutable), `s` (stable) or `v` (volatile) ([pg_proc.h#PROKIND_FUNCTION](../raw/postgres-17/src/include/catalog/pg_proc.h#L148-L166)). SQL-standard function bodies are stored pre-parsed in `prosqlbody`, a `pg_node_tree` ([pg_proc.h:121](../raw/postgres-17/src/include/catalog/pg_proc.h#L121)). Source code looks rows up through the `PROCOID` syscache ([pg_proc.h:143](../raw/postgres-17/src/include/catalog/pg_proc.h#L143)).

**Version notes:**
- PostgreSQL 12: Differs. The same property columns exist ([pg_proc.h#FormData_pg_proc](../raw/postgres-12/src/include/catalog/pg_proc.h#L31-L127)). There is no `prosqlbody`; function source is only the `prosrc` text ([pg_proc.h:116](../raw/postgres-12/src/include/catalog/pg_proc.h#L116)).
- PostgreSQL 14: Holds, including `prosqlbody` ([pg_proc.h#FormData_pg_proc](../raw/postgres-14/src/include/catalog/pg_proc.h#L30-L129), [pg_proc.h:121](../raw/postgres-14/src/include/catalog/pg_proc.h#L121)).
- PostgreSQL 18: Holds ([pg_proc.h#FormData_pg_proc](../raw/postgres-18/src/include/catalog/pg_proc.h#L30-L129)).
- PostgreSQL 19: Holds ([pg_proc.h#FormData_pg_proc](../raw/postgres-19/src/include/catalog/pg_proc.h#L32-L131)).

Related: [Catalog](#catalog), [Syscache](#syscache), [fmgr](#fmgr), [pg_node_tree](#pg_node_tree), [Parallel query](#parallel-query)

### pg_stat_activity

**Aliases:** `pg_stat_activity` view, `PgBackendStatus`, `pg_stat_get_activity()`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_stat_activity` is the view with one row per server process, showing what each one is doing right now. That includes its state, its current wait event, its transaction and snapshot horizon (`backend_xid`, `backend_xmin`), its query ID, the query text and its `backend_type` ([system_views.sql#pg_stat_activity](../raw/postgres-17/src/backend/catalog/system_views.sql#L865-L891)). The view reads `pg_stat_get_activity()`, which copies each backend's `PgBackendStatus` entry from shared memory ([system_views.sql#pg_stat_activity](../raw/postgres-17/src/backend/catalog/system_views.sql#L889), [backend_status.h#PgBackendStatus](../raw/postgres-17/src/include/utils/backend_status.h#L97-L173)). The `query` text lives in a fixed-size buffer, `st_activity_raw`, that may cut a string in the middle of a multi-byte character ([backend_status.h#st_activity_raw](../raw/postgres-17/src/include/utils/backend_status.h#L148-L156)). `track_activity_query_size` sets that buffer's size. Its context is `postmaster`, so changing it needs a restart ([guc_tables.c:3566](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3566)).

**Version notes:**
- PostgreSQL 12: Differs. The view has no `leader_pid` and no `query_id` column ([system_views.sql#pg_stat_activity](../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L756)). `PgBackendStatus` is declared in `pgstat.h`, not `backend_status.h` ([pgstat.h#PgBackendStatus](../raw/postgres-12/src/include/pgstat.h#L1027)). `track_activity_query_size` is `postmaster` context, so restart ([guc.c:3165](../raw/postgres-12/src/backend/utils/misc/guc.c#L3165)).
- PostgreSQL 14: Holds, with `leader_pid` and `query_id` ([system_views.sql#pg_stat_activity](../raw/postgres-14/src/backend/catalog/system_views.sql#L812-L838)). `track_activity_query_size` is `postmaster` context, so restart ([guc.c:3481](../raw/postgres-14/src/backend/utils/misc/guc.c#L3481)).
- PostgreSQL 18: Holds ([system_views.sql#pg_stat_activity](../raw/postgres-18/src/backend/catalog/system_views.sql#L878-L904)). `track_activity_query_size` is `postmaster` context, so restart ([guc_tables.c:3773](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3773)).
- PostgreSQL 19: Holds ([system_views.sql#pg_stat_activity](../raw/postgres-19/src/backend/catalog/system_views.sql#L939-L965)). `track_activity_query_size` is `PGC_POSTMASTER`, so restart ([guc_parameters.dat:3190](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3190)).

Related: [Cumulative statistics](#cumulative-statistics), [Wait event](#wait-event), [Backend](#backend), [xmin horizon](#xmin-horizon)

### pg_stat_all_tables

**Aliases:** `pg_stat_user_tables`, `pg_stat_sys_tables`, `pg_stat_all_indexes`, `pg_stat_user_indexes`, `seq_scan`, `idx_scan`, `n_dead_tup`, `last_autovacuum`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_stat_all_tables` is the [cumulative statistics](#cumulative-statistics) view with one row per table: how often it was scanned, how many rows were inserted, updated or deleted, the live and dead tuple estimates, and when and how often it was vacuumed and analyzed. It is a plain SQL view over `pg_stat_get_*()` functions, defined in `system_views.sql`. It covers ordinary tables, TOAST tables, materialized views and partitioned tables ([system_views.sql#pg_stat_all_tables](../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L702)). `pg_stat_user_tables` and `pg_stat_sys_tables` are the same view filtered by schema ([system_views.sql#pg_stat_user_tables](../raw/postgres-17/src/backend/catalog/system_views.sql#L726-L739)). `pg_stat_all_indexes` gives one row per index, with `idx_scan`, `last_idx_scan`, `idx_tup_read` and `idx_tup_fetch`, and `pg_stat_user_indexes` filters it the same way ([system_views.sql#pg_stat_all_indexes](../raw/postgres-17/src/backend/catalog/system_views.sql#L790-L815)). Maintenance heuristics read `n_dead_tup`, `n_mod_since_analyze`, `n_ins_since_vacuum` and the last-vacuum times here ([system_views.sql:687-694](../raw/postgres-17/src/backend/catalog/system_views.sql#L687-L694)).

**Version notes:**
- PostgreSQL 12: The view exists without `last_seq_scan`, `last_idx_scan`, `n_tup_newpage_upd` and `n_ins_since_vacuum` ([system_views.sql#pg_stat_all_tables](../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L582), [system_views.sql#pg_stat_all_indexes](../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672)).
- PostgreSQL 14: Adds `n_ins_since_vacuum` ([system_views.sql:648](../raw/postgres-14/src/backend/catalog/system_views.sql#L648)); still no last-scan columns ([system_views.sql#pg_stat_all_tables](../raw/postgres-14/src/backend/catalog/system_views.sql#L631)).
- PostgreSQL 18: Adds `total_vacuum_time`, `total_autovacuum_time`, `total_analyze_time` and `total_autoanalyze_time` ([system_views.sql:708-711](../raw/postgres-18/src/backend/catalog/system_views.sql#L708-L711)).
- PostgreSQL 19: Also adds a per-table `stats_reset` column ([system_views.sql:746-750](../raw/postgres-19/src/backend/catalog/system_views.sql#L746-L750), [system_views.sql#pg_stat_all_tables](../raw/postgres-19/src/backend/catalog/system_views.sql#L717)).

Related: [Cumulative statistics](#cumulative-statistics), [Autovacuum](#autovacuum), [VACUUM](#vacuum), [Index scan](#index-scan), [Sequential scan](#sequential-scan)

### pg_stat_io

**Aliases:** I/O statistics view, `pg_stat_get_io()`, `pgstat_io.c`, `IOObject`, `IOContext`, `IOOp`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_stat_io` is a cumulative statistics view that counts I/O by backend type, object and context. Each row shows reads, writes, writebacks, extends, hits, evictions, reuses and fsyncs, with times where tracked ([system_views.sql#pg_stat_io](../raw/postgres-17/src/backend/catalog/system_views.sql#L1153-L1173)). In PostgreSQL 17 the objects are ordinary and temporary relations. The contexts are `bulkread`, `bulkwrite`, `normal` and `vacuum`, and the counters form a three-dimensional array ([pgstat.h#IOObject](../raw/postgres-17/src/include/pgstat.h#L278-L313)). The counting code lives in its own file, `pgstat_io.c` ([pgstat_io.c header](../raw/postgres-17/src/backend/utils/activity/pgstat_io.c#L1-L8)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. `system_views.sql` defines no `pg_stat_io`; the closest buffer-write counters are in `pg_stat_bgwriter` ([system_views.sql#pg_stat_bgwriter](../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947)).
- PostgreSQL 14: Not present in PostgreSQL 14; the closest counters are again in `pg_stat_bgwriter` ([system_views.sql#pg_stat_bgwriter](../raw/postgres-14/src/backend/catalog/system_views.sql#L1058-L1070)).
- PostgreSQL 18: Differs. The view reports bytes (`read_bytes`, `write_bytes`, `extend_bytes`) in place of the single `op_bytes` column ([system_views.sql#pg_stat_io](../raw/postgres-18/src/backend/catalog/system_views.sql#L1171-L1193)). WAL becomes an I/O object, and a new `init` context appears ([pgstat.h#IOOBJECT_WAL](../raw/postgres-18/src/include/pgstat.h#L277-L286)).
- PostgreSQL 19: Holds as in 18 ([system_views.sql#pg_stat_io](../raw/postgres-19/src/backend/catalog/system_views.sql#L1261-L1283), [pgstat.h#IOObject](../raw/postgres-19/src/include/pgstat.h#L281-L286)).

Related: [Cumulative statistics](#cumulative-statistics), [Buffer manager](#buffer-manager), [Ring buffer](#ring-buffer), [Background writer](#background-writer)

### pg_stat_statements

**Aliases:** pgss, `pg_stat_statements.max`, query texts file, `pgss_query_texts.stat`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_stat_statements` is a contrib module that totals planning and execution statistics per normalized statement across the whole cluster. It keeps the counters in a shared hash table of fixed size ([pg_stat_statements.c:1-9](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L1-L9)). Normalized query text is stored in an external file, not in shared memory, and shared memory keeps only offsets into it ([pg_stat_statements.c:15-20](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L15-L20), [pg_stat_statements.c#PGSS_TEXT_FILE](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L85)). The module works only when loaded through `shared_preload_libraries`. On load it calls `EnableQueryId()`, so `compute_query_id = auto` turns on core query jumbling ([pg_stat_statements.c#_PG_init](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L380-L395)). `pg_stat_statements.max` caps the number of tracked statements. It has context `postmaster`, so changing it needs a restart ([pg_stat_statements.c#_PG_init](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L400-L412)).

**Version notes:**
- PostgreSQL 12: Differs. The module computes its own query jumble in `JumbleQuery()` inside the module; core has no query ID ([pg_stat_statements.c:10-28](../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L10-L28), [pg_stat_statements.c#JumbleQuery](../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L334)). It still requires `shared_preload_libraries` ([pg_stat_statements.c#_PG_init](../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L359-L360)). `pg_stat_statements.max` is `postmaster` context, so restart ([pg_stat_statements.c#_PG_init](../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L365-L372)).
- PostgreSQL 14: Holds. Jumbling moved to core, and the module calls `EnableQueryId()` ([pg_stat_statements.c#_PG_init](../raw/postgres-14/contrib/pg_stat_statements/pg_stat_statements.c#L370-L377)). `pg_stat_statements.max` is `postmaster` context, so restart ([pg_stat_statements.c#_PG_init](../raw/postgres-14/contrib/pg_stat_statements/pg_stat_statements.c#L382-L389)).
- PostgreSQL 18: Holds ([pg_stat_statements.c#_PG_init](../raw/postgres-18/contrib/pg_stat_statements/pg_stat_statements.c#L395-L414)). `pg_stat_statements.max` is `postmaster` context, so restart.
- PostgreSQL 19: Holds ([pg_stat_statements.c#_PG_init](../raw/postgres-19/contrib/pg_stat_statements/pg_stat_statements.c#L400-L419)). `pg_stat_statements.max` is `postmaster` context, so restart.

Related: [Query jumbling](#query-jumbling), [Contrib](#contrib), [Extension](#extension), [Cumulative statistics](#cumulative-statistics)

### pg_upgrade

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pg_upgrade` moves a cluster to a newer major version without a dump and restore. It creates new system catalogs and reuses the old user data files ([pgupgrade.sgml:39-58](../raw/postgres-17/doc/src/sgml/ref/pgupgrade.sgml#L39-L58)). It carries the files over by clone, copy, `copy_file_range`, or hard link ([pg_upgrade.h#transferMode](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.h#L253-L260)). It forces table OIDs and relfilenumbers to match between the old and new clusters so that the file names and the stored TOAST pointers stay valid ([pg_upgrade.c:10-21](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L10-L21)). Because it reuses the old user data files, index pages keep the on-disk state the old version wrote ([pgupgrade.sgml:48-53](../raw/postgres-17/doc/src/sgml/ref/pgupgrade.sgml#L48-L53)).

**Version notes:**
- PostgreSQL 12: Differs. It reuses old user data files ([pgupgrade.sgml](../raw/postgres-12/doc/src/sgml/ref/pgupgrade.sgml#L39-L55)). Transfer modes are clone, copy and link only; there is no `copy_file_range` mode ([pg_upgrade.h#transferMode](../raw/postgres-12/src/bin/pg_upgrade/pg_upgrade.h#L235-L240)). 12 preserves `pg_class.oid` but not relfilenodes: in the new cluster the relfilenode equals the old OID, so old and new relfilenodes differ after `CLUSTER`, `REINDEX` or `VACUUM FULL` ([pg_upgrade.c:10-23](../raw/postgres-12/src/bin/pg_upgrade/pg_upgrade.c#L10-L23)).
- PostgreSQL 14: Differs. It still reuses old user data files ([pgupgrade.sgml:39-58](../raw/postgres-14/doc/src/sgml/ref/pgupgrade.sgml#L39-L58)). But 14 offers only clone, copy and link modes, with no `copy_file_range` mode ([pg_upgrade.h#transferMode](../raw/postgres-14/src/bin/pg_upgrade/pg_upgrade.h#L228-L233)). And 14 preserves table OIDs but not relfilenodes: they differ between clusters when the old one ran `CLUSTER`, `REINDEX` or `VACUUM FULL` ([pg_upgrade.c:10-23](../raw/postgres-14/src/bin/pg_upgrade/pg_upgrade.c#L10-L23)).
- PostgreSQL 18: Holds, with two additions. A new `--swap` transfer mode moves the old data directories into the new cluster ([pg_upgrade.h#transferMode](../raw/postgres-18/src/bin/pg_upgrade/pg_upgrade.h#L258-L265), [pgupgrade.sgml#swap](../raw/postgres-18/doc/src/sgml/ref/pgupgrade.sgml#L328-L336)). Unless `--no-statistics` is given, it now carries most optimizer statistics over; extended statistics and cumulative statistics are not transferred ([pgupgrade.sgml:833-840](../raw/postgres-18/doc/src/sgml/ref/pgupgrade.sgml#L833-L840)). The OID and relfilenumber preservation is unchanged ([pg_upgrade.c:10-21](../raw/postgres-18/src/bin/pg_upgrade/pg_upgrade.c#L10-L21)).
- PostgreSQL 19: Holds. It still creates new catalogs and reuses user data files, and it still forces matching OIDs and relfilenumbers ([pgupgrade.sgml:39-58](../raw/postgres-19/doc/src/sgml/ref/pgupgrade.sgml#L39-L58), [pg_upgrade.c:9-21](../raw/postgres-19/src/bin/pg_upgrade/pg_upgrade.c#L9-L21)). v19 adds a fifth transfer mode, `--swap`, which moves the old data directories into the new cluster and then replaces the catalog files ([pg_upgrade.h#transferMode](../raw/postgres-19/src/bin/pg_upgrade/pg_upgrade.h#L266-L273), [pgupgrade.sgml](../raw/postgres-19/doc/src/sgml/ref/pgupgrade.sgml#L328-L337)).

Related: [Relfilenumber](#relfilenumber), [OID](#oid), [allequalimage](#allequalimage), [Catalog](#catalog)

### pgs_mask

**Aliases:** path generation strategy mask, `PGS_*` flags, `default_pgs_mask`, `PGS_CONSIDER_PARTITIONWISE`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

In PostgreSQL 19, `pgs_mask` is a 64-bit set of `PGS_*` bits that tells the planner which path strategies it may use, such as sequential scan, index scan, hash join or nested loop with memoize ([pathnodes.h#PGS_SEQSCAN](../raw/postgres-19/src/include/nodes/pathnodes.h#L25-L84)). The planner builds `PlannerGlobal.default_pgs_mask` from the `enable_*` settings, then offers `planner_setup_hook` a chance to change it ([planner.c:493-529](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L493-L529)). Each new base and join `RelOptInfo` copies the default into its own `pgs_mask`, and `build_simple_rel_hook` can edit it per relation ([relnode.c:234](../raw/postgres-19/src/backend/optimizer/util/relnode.c#L234), [relnode.c:400-409](../raw/postgres-19/src/backend/optimizer/util/relnode.c#L400-L409)). When a path uses a cleared strategy, the cost functions mostly mark it disabled rather than skip it. For example, `cost_seqscan()` sets `disabled_nodes` ([costsize.c#cost_seqscan](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L330-L336)). Some bits, such as `PGS_CONSIDER_PARTITIONWISE`, stop path generation outright ([pathnodes.h:47-57](../raw/postgres-19/src/include/nodes/pathnodes.h#L47-L57)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. `RelOptInfo` has no strategy mask ([pathnodes.h#RelOptInfo](../raw/postgres-12/src/include/nodes/pathnodes.h#L645-L649)). An `enable_*` setting that is off adds `disable_cost` to the path instead ([costsize.c#cost_seqscan](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L231-L232)).
- PostgreSQL 14: Not present in PostgreSQL 14 ([pathnodes.h#RelOptInfo](../raw/postgres-14/src/include/nodes/pathnodes.h#L687-L691)). Off `enable_*` settings add `disable_cost` ([costsize.c#cost_seqscan](../raw/postgres-14/src/backend/optimizer/path/costsize.c#L246-L247)).
- PostgreSQL 17: Not present in PostgreSQL 17 ([pathnodes.h#RelOptInfo](../raw/postgres-17/src/include/nodes/pathnodes.h#L876-L881)). Off `enable_*` settings add `disable_cost` ([costsize.c#cost_seqscan](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L304-L305)).
- PostgreSQL 18: Not present in PostgreSQL 18 ([pathnodes.h#RelOptInfo](../raw/postgres-18/src/include/nodes/pathnodes.h#L906-L911)). Paths already count `disabled_nodes`, but `cost_seqscan()` sets it straight from `enable_seqscan` ([costsize.c#cost_seqscan](../raw/postgres-18/src/backend/optimizer/path/costsize.c#L357)).

Related: [pg_plan_advice](#pg_plan_advice), [Planner](#planner), [Path](#path), [RelOptInfo](#reloptinfo), [Partitionwise join](#partitionwise-join), [Hook](#hook)

### pgstatindex

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pgstatindex` is a `pgstattuple` function that summarizes a B-tree index's shape and space use. It rejects any other index type and any index that is not valid ([pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L250)). It reads the metapage, then every other block, and counts deleted, half-dead ("empty"), leaf and internal pages ([pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L300-L327)). `avg_leaf_density` is 100 minus leaf free space as a percentage of leaf capacity. `leaf_fragmentation` is the share of leaves whose right sibling sits at a lower block number ([pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L310-L322), [pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L349-L372)). The install scripts grant it to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql](../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)).

**Version notes:**
- PostgreSQL 12: Differs. It rejects non-B-tree indexes but has no `indisvalid` check, so 12 will read an invalid index ([pgstatindex.c#pgstatindex_impl](../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L238)). It counts deleted, half-dead, leaf and internal pages, and computes density and fragmentation the same way ([pgstatindex.c#pgstatindex_impl](../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L309), [pgstatindex.c:349-354](../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L349-L354)). It is granted to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql](../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L37)).
- PostgreSQL 14: Holds. It rejects non-B-tree and invalid indexes, counts deleted, half-dead, leaf and internal pages, and computes `avg_leaf_density` and `leaf_fragmentation` the same way; version 1.5 grants it to `pg_stat_scan_tables` ([pgstatindex.c#pgstatindex_impl](../raw/postgres-14/contrib/pgstattuple/pgstatindex.c#L224-L250), [pgstatindex.c#pgstatindex_impl](../raw/postgres-14/contrib/pgstattuple/pgstatindex.c#L300-L327), [pgstatindex.c#pgstatindex_impl](../raw/postgres-14/contrib/pgstattuple/pgstatindex.c#L355-L371), [pgstattuple--1.4--1.5.sql](../raw/postgres-14/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L37)).
- PostgreSQL 18: Holds, with one arithmetic change. Leaf free space is now summed with `PageGetExactFreeSpace()` instead of `PageGetFreeSpace()`, so it no longer subtracts one line pointer per page, and `avg_leaf_density` can read slightly lower than in 17 for the same page ([pgstatindex.c#pgstatindex_impl](../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L297-L324)). The rejection rules, the density formula, and the grant are unchanged ([pgstatindex.c#pgstatindex_impl](../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L213-L247), [pgstatindex.c#pgstatindex_impl](../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L346-L369), [pgstattuple--1.4--1.5.sql](../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)).
- PostgreSQL 19: Holds. It still rejects non-B-tree and invalid indexes, still counts deleted, half-dead, leaf and internal pages, still computes `avg_leaf_density` and `leaf_fragmentation` the same way, and is still granted to `pg_stat_scan_tables` ([pgstatindex.c#pgstatindex_impl](../raw/postgres-19/contrib/pgstattuple/pgstatindex.c#L214-L250), [pgstatindex.c#pgstatindex_impl](../raw/postgres-19/contrib/pgstattuple/pgstatindex.c#L301-L345), [pgstatindex.c#pgstatindex_impl](../raw/postgres-19/contrib/pgstattuple/pgstatindex.c#L381-L392), [pgstattuple--1.4--1.5.sql](../raw/postgres-19/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L37)).

Related: [pgstattuple](#pgstattuple), [B-tree](#b-tree), [Metapage](#metapage), [Bloat](#bloat), [Invalid index](#invalid-index)

### pgstattuple

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`pgstattuple` is a contrib extension that scans a relation and reports tuple-level space use: live and dead tuple counts and bytes, plus free space ([pgstattuple.control](../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L2), [pgstattuple.c#pgstattuple_type](../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L55-L63)). `pgstat_relation` sends tables and sequences to `pgstat_heap`. It handles B-tree, hash and GiST indexes itself and rejects GIN, SP-GiST and BRIN ([pgstattuple.c#pgstat_relation](../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L255-L298)). The same extension provides `pgstatindex`, `pgstatginindex`, `pgstathashindex` and the sampling `pgstattuple_approx` ([pgstatindex.c#pgstatginindex](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L485-L507), [pgstatindex.c:586](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L586), [pgstatapprox.c#pgstattuple_approx](../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L227)).

**Version notes:**
- PostgreSQL 12: Holds. It reports live/dead counts and free space, routes heaps to `pgstat_heap`, handles B-tree, hash and GiST indexes, and ships `pgstatginindex`, `pgstathashindex` and `pgstattuple_approx` ([pgstattuple.c#pgstattuple_type](../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L55-L63), [pgstattuple.c#pgstat_relation](../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L241-L300), [pgstatindex.c#pgstatginindex](../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L488), [pgstatindex.c:582](../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L582), [pgstatapprox.c#pgstattuple_approx](../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L224)).
- PostgreSQL 14: Holds. `pgstat_relation` handles B-tree, hash and GiST indexes and rejects GIN, SP-GiST and BRIN, and the extension also ships `pgstatginindex`, `pgstathashindex` and `pgstattuple_approx` ([pgstattuple.control](../raw/postgres-14/contrib/pgstattuple/pgstattuple.control#L1-L2), [pgstattuple.c#pgstat_relation](../raw/postgres-14/contrib/pgstattuple/pgstattuple.c#L270-L292), [pgstatindex.c#pgstatginindex](../raw/postgres-14/contrib/pgstattuple/pgstatindex.c#L504), [pgstatindex.c:605](../raw/postgres-14/contrib/pgstattuple/pgstatindex.c#L605), [pgstatapprox.c#pgstattuple_approx](../raw/postgres-14/contrib/pgstattuple/pgstatapprox.c#L227)).
- PostgreSQL 18: Holds ([pgstattuple.c#pgstattuple_type](../raw/postgres-18/contrib/pgstattuple/pgstattuple.c#L57-L65), [pgstattuple.c#pgstat_relation](../raw/postgres-18/contrib/pgstattuple/pgstattuple.c#L257-L300), [pgstatapprox.c#pgstattuple_approx](../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L214)).
- PostgreSQL 19: Holds. It still reports live and dead tuple counts and bytes plus free space, `pgstat_relation()` still sends tables to `pgstat_heap()`, handles B-tree, hash and GiST itself and rejects GIN, SP-GiST and BRIN, and the extension still ships `pgstatindex`, `pgstatginindex`, `pgstathashindex` and `pgstattuple_approx` ([pgstattuple.control](../raw/postgres-19/contrib/pgstattuple/pgstattuple.control#L1-L2), [pgstattuple.c#pgstattuple_type](../raw/postgres-19/contrib/pgstattuple/pgstattuple.c#L57-L65), [pgstattuple.c#pgstat_relation](../raw/postgres-19/contrib/pgstattuple/pgstattuple.c#L243-L293), [pgstatindex.c#pgstatginindex](../raw/postgres-19/contrib/pgstattuple/pgstatindex.c#L503), [pgstatindex.c#pgstathashindex](../raw/postgres-19/contrib/pgstattuple/pgstatindex.c#L608), [pgstatapprox.c#pgstattuple_approx](../raw/postgres-19/contrib/pgstattuple/pgstatapprox.c#L279)).

Related: [pgstatindex](#pgstatindex), [Tuple](#tuple), [Bloat](#bloat), [Contrib](#contrib)

### Plan cache mode

**Aliases:** `plan_cache_mode`, `auto`, `force_custom_plan`, `force_generic_plan`, `PlanCacheMode`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`plan_cache_mode` is the setting that decides whether a cached statement runs with a custom plan or a generic plan. It has three values: `auto`, `force_generic_plan` and `force_custom_plan` ([plancache.h#PlanCacheMode](../raw/postgres-17/src/include/utils/plancache.h#L29-L35)). `choose_custom_plan()` applies it only after two early exits: a one-shot plan is always custom, and a statement with no parameters is always generic. It then lets the setting force the choice ([plancache.c#choose_custom_plan](../raw/postgres-17/src/backend/utils/cache/plancache.c#L1053-L1073)). Under `auto`, the first five executions get custom plans. After that, the generic plan wins whenever its cost is lower than the average custom-plan cost, which includes planning cost ([plancache.c#choose_custom_plan](../raw/postgres-17/src/backend/utils/cache/plancache.c#L1081-L1100)). The setting has context `user`, so it can be set per session or transaction ([guc_tables.c:5112](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5112)).

**Version notes:**
- PostgreSQL 12: Holds. The same three values and the same five-custom-plan rule apply ([plancache.h#PlanCacheMode](../raw/postgres-12/src/include/utils/plancache.h#L26-L32), [plancache.c#choose_custom_plan](../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016-L1062)). The setting is `user` context, so session scope ([guc.c:4504](../raw/postgres-12/src/backend/utils/misc/guc.c#L4504)).
- PostgreSQL 14: Holds ([plancache.c#choose_custom_plan](../raw/postgres-14/src/backend/utils/cache/plancache.c#L1031-L1077)). The setting is `user` context, so session scope ([guc.c:4949](../raw/postgres-14/src/backend/utils/misc/guc.c#L4949)).
- PostgreSQL 18: Holds ([plancache.c#choose_custom_plan](../raw/postgres-18/src/backend/utils/cache/plancache.c#L1166-L1212)). The setting is `user` context, so session scope ([guc_tables.c:5385](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5385)).
- PostgreSQL 19: Holds ([plancache.c#choose_custom_plan](../raw/postgres-19/src/backend/utils/cache/plancache.c#L1184-L1230)). The setting is `PGC_USERSET`, so session scope ([guc_parameters.dat:2363](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2363)).

Related: [Custom and generic plan](#custom-and-generic-plan), [Prepared statement](#prepared-statement), [Planner](#planner), [GUC context](#guc-context)

### PlannedStmt

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`PlannedStmt` is the planner's output. It is the top node of a `Plan` tree and holds the one-time information the executor needs: the range table, result relations, subplans, and the OIDs the plan depends on [plannodes.h#PlannedStmt](../raw/postgres-17/src/include/nodes/plannodes.h#L31-L35) [plannodes.h:70-89](../raw/postgres-17/src/include/nodes/plannodes.h#L70-L89). A utility statement is also wrapped in a `PlannedStmt`. That wrapper has `commandType = CMD_UTILITY` and the statement in `utilityStmt`, and the rest of the struct is mostly unused [plannodes.h:37-40](../raw/postgres-17/src/include/nodes/plannodes.h#L37-L40). `pg_plan_queries()` builds such a wrapper without calling the planner [postgres.c#pg_plan_queries](../raw/postgres-17/src/backend/tcop/postgres.c#L976-L1008).

**Version notes:**
- PostgreSQL 12: Holds. A utility statement is wrapped with `CMD_UTILITY` and `utilityStmt`, and `pg_plan_queries()` builds that wrapper ([plannodes.h#PlannedStmt](../raw/postgres-12/src/include/nodes/plannodes.h#L30-L42), [postgres.c#pg_plan_queries](../raw/postgres-12/src/backend/tcop/postgres.c#L946)).
- PostgreSQL 14: Holds. It heads the plan tree with the range table, result relations, subplans and dependency OIDs, and utility statements are wrapped with `CMD_UTILITY`; `pg_plan_queries()` builds that wrapper ([plannodes.h#PlannedStmt](../raw/postgres-14/src/include/nodes/plannodes.h#L29-L40), [plannodes.h:66-80](../raw/postgres-14/src/include/nodes/plannodes.h#L66-L80), [postgres.c#pg_plan_queries](../raw/postgres-14/src/backend/tcop/postgres.c#L883-L915)).
- PostgreSQL 18: Holds. The struct still holds the plan tree, range table, result relations, subplans and dependency OIDs. It gains fields such as `planId` and `unprunableRelids` ([plannodes.h#PlannedStmt](../raw/postgres-18/src/include/nodes/plannodes.h#L31-L139)). The utility wrapper and `pg_plan_queries()` are unchanged ([plannodes.h:37-40](../raw/postgres-18/src/include/nodes/plannodes.h#L37-L40), [postgres.c#pg_plan_queries](../raw/postgres-18/src/backend/tcop/postgres.c#L970-L1002)).
- PostgreSQL 19: Holds. It still heads the plan tree with the executor's one-time data, a utility statement is still wrapped with `commandType = CMD_UTILITY`, and `pg_plan_queries()` still builds that wrapper without the planner ([plannodes.h#PlannedStmt](../raw/postgres-19/src/include/nodes/plannodes.h#L44-L58), [postgres.c#pg_plan_queries](../raw/postgres-19/src/backend/tcop/postgres.c#L987-L1017)). v19 adds fields including `planId`, `planOrigin` (standard, generic cached or custom cached), `unprunableRelids`, `elidedNodes` and `extension_state` ([plannodes.h#PlannedStmt](../raw/postgres-19/src/include/nodes/plannodes.h#L36-L42), [plannodes.h#PlannedStmt](../raw/postgres-19/src/include/nodes/plannodes.h#L71-L75), [plannodes.h:113](../raw/postgres-19/src/include/nodes/plannodes.h#L113), [plannodes.h:156](../raw/postgres-19/src/include/nodes/plannodes.h#L156), [plannodes.h:165](../raw/postgres-19/src/include/nodes/plannodes.h#L165)).

Related: [Planner](#planner), [Executor](#executor), [Utility command](#utility-command), [Parse tree](#parse-tree)

### Planner

**Aliases:** optimizer, query planner, query optimizer. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The planner decides how to run a query. It turns an analyzed `Query` into the cheapest plan it can find [glossary.sgml#glossary-planner](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1334-L1345). Its entry point, `planner()`, calls `planner_hook` if an extension installed one and `standard_planner()` otherwise [planner.c#planner](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L262-L285). `standard_planner()` runs `subquery_planner()` to build paths. It then picks the cheapest final path with `get_cheapest_fractional_path()` and turns it into a plan with `create_plan()` [planner.c:415-422](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L415-L422). The planner reads table and index sizes from the catalogs in `plancat.c` [plancat.c#get_relation_info](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L115-L117).

**Version notes:**
- PostgreSQL 12: Holds. `planner()` calls `planner_hook` or `standard_planner()`, which picks the cheapest fractional path and calls `create_plan()`; `get_relation_info()` reads sizes ([planner.c#planner](../raw/postgres-12/src/backend/optimizer/plan/planner.c#L268-L280), [planner.c:411-413](../raw/postgres-12/src/backend/optimizer/plan/planner.c#L411-L413), [plancat.c#get_relation_info](../raw/postgres-12/src/backend/optimizer/util/plancat.c#L111)).
- PostgreSQL 14: Holds. `planner()` calls `planner_hook` or `standard_planner()`, which picks the cheapest final path and calls `create_plan()`; `get_relation_info()` reads sizes from the catalogs ([glossary.sgml#glossary-planner](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1107-L1118), [planner.c#planner](../raw/postgres-14/src/backend/optimizer/plan/planner.c#L264-L274), [planner.c:408-410](../raw/postgres-14/src/backend/optimizer/plan/planner.c#L408-L410), [plancat.c#get_relation_info](../raw/postgres-14/src/backend/optimizer/util/plancat.c#L114)).
- PostgreSQL 18: Holds. `planner()` still dispatches to `planner_hook` or `standard_planner()`, and now also reports the plan's `planId` ([planner.c#planner](../raw/postgres-18/src/backend/optimizer/plan/planner.c#L287-L313), [planner.c:447-454](../raw/postgres-18/src/backend/optimizer/plan/planner.c#L447-L454), [plancat.c#get_relation_info](../raw/postgres-18/src/backend/optimizer/util/plancat.c#L116-L118)).
- PostgreSQL 19: Holds. `planner()` still calls `planner_hook` or `standard_planner()`, which still runs `subquery_planner()`, picks the cheapest path and calls `create_plan()`, and `plancat.c` still reads sizes ([glossary.sgml#glossary-planner](../raw/postgres-19/doc/src/sgml/glossary.sgml#L1431-L1442), [planner.c#planner](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L328-L342), [planner.c:532-539](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L532-L539), [plancat.c#get_relation_info](../raw/postgres-19/src/backend/optimizer/util/plancat.c#L121)). `planner()` now also receives the query string and an `ExplainState`. The `enable_*` GUCs now build a per-query strategy mask, `default_pgs_mask`, that extensions such as `pg_plan_advice` can narrow per relation ([planner.c#planner](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L328-L329), [pathnodes.h:24-35](../raw/postgres-19/src/include/nodes/pathnodes.h#L24-L35), [planner.c:493-524](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L493-L524)).

Related: [Path](#path), [RelOptInfo](#reloptinfo), [Cost](#cost), [PlannedStmt](#plannedstmt), [Hook](#hook)

### PL/pgSQL

**Aliases:** plpgsql, `DO` block. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

PL/pgSQL is PostgreSQL's built-in procedural language for writing functions, procedures, and anonymous `DO` blocks [plpgsql.sgml:4](../raw/postgres-17/doc/src/sgml/plpgsql.sgml#L4). It ships as the `plpgsql` extension, and `initdb` creates it in every new cluster [plpgsql--1.0.sql](../raw/postgres-17/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L15) [initdb.c#load_plpgsql](../raw/postgres-17/src/bin/initdb/initdb.c#L1974-L1976). A `DO` statement reaches `ExecuteDoStmt()`, which calls the language's inline handler, `plpgsql_inline_handler()` [functioncmds.c#ExecuteDoStmt](../raw/postgres-17/src/backend/commands/functioncmds.c#L2060-L2066) [functioncmds.c:2146-2156](../raw/postgres-17/src/backend/commands/functioncmds.c#L2146-L2156). The inline handler connects to SPI before running any SQL [pl_handler.c#plpgsql_inline_handler](../raw/postgres-17/src/pl/plpgsql/src/pl_handler.c#L313-L330).

**Version notes:**
- PostgreSQL 12: Holds. `initdb` runs `CREATE EXTENSION plpgsql`, `ExecuteDoStmt()` calls the language's inline handler, and `plpgsql_inline_handler()` connects to SPI ([initdb.c#load_plpgsql](../raw/postgres-12/src/bin/initdb/initdb.c#L1999-L2001), [plpgsql--1.0.sql](../raw/postgres-12/src/pl/plpgsql/src/plpgsql--1.0.sql#L1-L9), [functioncmds.c#ExecuteDoStmt](../raw/postgres-12/src/backend/commands/functioncmds.c#L2172), [functioncmds.c:2256-2266](../raw/postgres-12/src/backend/commands/functioncmds.c#L2256-L2266), [pl_handler.c#plpgsql_inline_handler](../raw/postgres-12/src/pl/plpgsql/src/pl_handler.c#L300-L314)).
- PostgreSQL 14: Holds. It ships as the `plpgsql` extension, `initdb` installs it, `ExecuteDoStmt()` calls the inline handler, and `plpgsql_inline_handler()` connects to SPI ([plpgsql.sgml:4](../raw/postgres-14/doc/src/sgml/plpgsql.sgml#L4), [plpgsql--1.0.sql](../raw/postgres-14/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L5), [initdb.c#load_plpgsql](../raw/postgres-14/src/bin/initdb/initdb.c#L1891), [functioncmds.c#ExecuteDoStmt](../raw/postgres-14/src/backend/commands/functioncmds.c#L2160-L2170), [pl_handler.c#plpgsql_inline_handler](../raw/postgres-14/src/pl/plpgsql/src/pl_handler.c#L329)).
- PostgreSQL 18: Holds ([plpgsql--1.0.sql](../raw/postgres-18/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L15), [initdb.c#load_plpgsql](../raw/postgres-18/src/bin/initdb/initdb.c#L1992-L1994), [functioncmds.c#ExecuteDoStmt](../raw/postgres-18/src/backend/commands/functioncmds.c#L2078-L2084), [pl_handler.c#plpgsql_inline_handler](../raw/postgres-18/src/pl/plpgsql/src/pl_handler.c#L314-L331)).
- PostgreSQL 19: Holds. It still ships as the `plpgsql` extension that `initdb` creates, and `ExecuteDoStmt()` still calls the inline handler, which connects to SPI first ([plpgsql.sgml:4](../raw/postgres-19/doc/src/sgml/plpgsql.sgml#L4), [plpgsql--1.0.sql](../raw/postgres-19/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L15), [initdb.c#load_plpgsql](../raw/postgres-19/src/bin/initdb/initdb.c#L2015-L2017), [functioncmds.c#ExecuteDoStmt](../raw/postgres-19/src/backend/commands/functioncmds.c#L2099), [functioncmds.c:2179-2189](../raw/postgres-19/src/backend/commands/functioncmds.c#L2179-L2189), [pl_handler.c#plpgsql_inline_handler](../raw/postgres-19/src/pl/plpgsql/src/pl_handler.c#L316-L330)).

Related: [SPI](#spi), [Extension](#extension), [Utility command](#utility-command)

### Portal

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A portal holds the execution state of a query that is running or ready to run. Both SQL cursors and the unnamed and named portals of the client protocol use it [portal.h header](../raw/postgres-17/src/include/utils/portal.h#L3-L9). It is a `PortalData` struct with its own memory context [portal.h#PortalData](../raw/postgres-17/src/include/utils/portal.h#L113-L120). `PortalRun()` runs a portal's statements and can stop after a row count, leaving the portal suspended [pquery.c#PortalRun](../raw/postgres-17/src/backend/tcop/pquery.c#L662-L689). For a utility statement, `PortalRunUtility()` calls `ProcessUtility()` [pquery.c#PortalRunUtility](../raw/postgres-17/src/backend/tcop/pquery.c#L1125-L1156).

**Version notes:**
- PostgreSQL 12: Holds. A portal holds a query's execution state for cursors and protocol portals; `PortalRun()` and `PortalRunUtility()` exist ([portal.h header](../raw/postgres-12/src/include/utils/portal.h#L3-L9), [portal.h#PortalData](../raw/postgres-12/src/include/utils/portal.h#L114), [pquery.c#PortalRun](../raw/postgres-12/src/backend/tcop/pquery.c#L686), [pquery.c#PortalRunUtility](../raw/postgres-12/src/backend/tcop/pquery.c#L1131)).
- PostgreSQL 14: Holds. A portal holds execution state for cursors and protocol portals, `PortalRun()` runs it, and `PortalRunUtility()` calls `ProcessUtility()` ([portal.h header](../raw/postgres-14/src/include/utils/portal.h#L3-L9), [portal.h#PortalData](../raw/postgres-14/src/include/utils/portal.h#L115), [pquery.c#PortalRun](../raw/postgres-14/src/backend/tcop/pquery.c#L683-L841), [pquery.c#PortalRunUtility](../raw/postgres-14/src/backend/tcop/pquery.c#L1153)).
- PostgreSQL 18: Holds. `PortalRun()` drops the unused `run_once` argument ([portal.h#PortalData](../raw/postgres-18/src/include/utils/portal.h#L113-L120), [pquery.c#PortalRun](../raw/postgres-18/src/backend/tcop/pquery.c#L663-L686), [pquery.c#PortalRunUtility](../raw/postgres-18/src/backend/tcop/pquery.c#L1122-L1153)).
- PostgreSQL 19: Holds. A portal is still the execution state for cursors and protocol portals, `PortalData` still has its own memory context, and `PortalRun()` and `PortalRunUtility()` are unchanged in role ([portal.h header](../raw/postgres-19/src/include/utils/portal.h#L3-L9), [portal.h#PortalData](../raw/postgres-19/src/include/utils/portal.h#L115-L120), [pquery.c#PortalRun](../raw/postgres-19/src/backend/tcop/pquery.c#L681), [pquery.c#PortalRunUtility](../raw/postgres-19/src/backend/tcop/pquery.c#L1118)).

Related: [Executor](#executor), [Utility command](#utility-command), [Memory context](#memory-context)

### Posting list

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A posting list is an array of row addresses (TIDs) stored under one key. B-tree and GIN both use the idea, in different formats. In B-tree, deduplication creates posting-list tuples: one key followed by a sorted TID array, marked with `INDEX_ALT_TID_MASK` and `BT_IS_POSTING` ([nbtree.h](../raw/postgres-17/src/include/access/nbtree.h#L432-L457)). In GIN, a leaf entry holds a compressed posting list when it fits in `GinMaxItemSize`. Otherwise the entry points to a posting tree ([gininsert.c#buildFreshLeafTuple](../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L165)). GIN compresses its lists with varbyte encoding; B-tree does not compress them ([ginpostinglist.c](../raw/postgres-17/src/backend/access/gin/ginpostinglist.c#L23-L42), [nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L931-L942)).

**Version notes:**
- PostgreSQL 12: Differs. GIN posting lists exist, compressed with varbyte encoding, and `buildFreshLeafTuple()` falls back to a posting tree past `GinMaxItemSize` ([gininsert.c#buildFreshLeafTuple](../raw/postgres-12/src/backend/access/gin/gininsert.c#L130-L166), [ginpostinglist.c](../raw/postgres-12/src/backend/access/gin/ginpostinglist.c#L31-L35)). B-tree posting-list tuples do not exist in 12: the `INDEX_ALT_TID_MASK` bit is never set on leaf tuples, and there is no `BT_IS_POSTING` ([nbtree.h:236-238](../raw/postgres-12/src/include/access/nbtree.h#L236-L238)).
- PostgreSQL 14: Holds. B-tree posting-list tuples are marked with `INDEX_ALT_TID_MASK` and `BT_IS_POSTING`, GIN stores a compressed list in the leaf or a posting tree, GIN uses varbyte encoding, and B-tree does not compress ([nbtree.h](../raw/postgres-14/src/include/access/nbtree.h#L428-L466), [gininsert.c#buildFreshLeafTuple](../raw/postgres-14/src/backend/access/gin/gininsert.c#L129-L169), [ginpostinglist.c](../raw/postgres-14/src/backend/access/gin/ginpostinglist.c#L31-L35), [nbtree/README](../raw/postgres-14/src/backend/access/nbtree/README#L906-L910)).
- PostgreSQL 18: Holds ([nbtree.h](../raw/postgres-18/src/include/access/nbtree.h#L433-L458), [gininsert.c#buildFreshLeafTuple](../raw/postgres-18/src/backend/access/gin/gininsert.c#L290-L330), [ginpostinglist.c](../raw/postgres-18/src/backend/access/gin/ginpostinglist.c#L23-L42)).
- PostgreSQL 19: Holds. B-tree posting-list tuples are still marked with `INDEX_ALT_TID_MASK` and `BT_IS_POSTING`, GIN still stores an inline compressed posting list when it fits `GinMaxItemSize`, GIN still uses varbyte compression, and B-tree still does not compress ([nbtree.h](../raw/postgres-19/src/include/access/nbtree.h#L440-L467), [gininsert.c#buildFreshLeafTuple](../raw/postgres-19/src/backend/access/gin/gininsert.c#L290-L341), [ginpostinglist.c](../raw/postgres-19/src/backend/access/gin/ginpostinglist.c#L23-L42), [nbtree/README](../raw/postgres-19/src/backend/access/nbtree/README#L932-L942)).

Related: [Deduplication](#deduplication), [Posting tree](#posting-tree), [GIN](#gin), [TID](#tid)

### Posting tree

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A posting tree is a separate B-tree of row addresses (TIDs) that GIN builds for one key when its posting list is too large to fit inline in the entry tuple ([gin/README](../raw/postgres-17/src/backend/access/gin/README#L22-L26)). `buildFreshLeafTuple()` first tries to compress the TIDs into a posting list that fits `GinMaxItemSize`. If that fails, it calls `createPostingTree()` and stores the tree's root block in the entry with `GinSetPostingTree()` ([gininsert.c#buildFreshLeafTuple](../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L165), [ginblock.h#GinMaxItemSize](../raw/postgres-17/src/include/access/ginblock.h#L249-L253)). Do not confuse it with GIN's main entry tree, which is indexed by key rather than by TID.

**Version notes:**
- PostgreSQL 12: Holds. GIN builds a B-tree of TIDs when the list will not fit `GinMaxItemSize`, via `createPostingTree()` and `GinSetPostingTree()` ([gin/README](../raw/postgres-12/src/backend/access/gin/README#L22-L26), [gininsert.c#buildFreshLeafTuple](../raw/postgres-12/src/backend/access/gin/gininsert.c#L139-L166), [ginblock.h#GinMaxItemSize](../raw/postgres-12/src/include/access/ginblock.h#L252)).
- PostgreSQL 14: Holds. `buildFreshLeafTuple()` tries a compressed list that fits `GinMaxItemSize`, and otherwise calls `createPostingTree()` and stores the root with `GinSetPostingTree()` ([gin/README](../raw/postgres-14/src/backend/access/gin/README#L22-L26), [gininsert.c#buildFreshLeafTuple](../raw/postgres-14/src/backend/access/gin/gininsert.c#L129-L169), [ginblock.h#GinMaxItemSize](../raw/postgres-14/src/include/access/ginblock.h#L249-L253)).
- PostgreSQL 18: Holds ([gin/README](../raw/postgres-18/src/backend/access/gin/README#L22-L26), [gininsert.c#buildFreshLeafTuple](../raw/postgres-18/src/backend/access/gin/gininsert.c#L290-L330), [ginblock.h#GinMaxItemSize](../raw/postgres-18/src/include/access/ginblock.h#L249-L253)).
- PostgreSQL 19: Holds. GIN still builds a separate TID B-tree with `createPostingTree()` when the list does not fit, and it stores the root with `GinSetPostingTree()` ([gin/README](../raw/postgres-19/src/backend/access/gin/README#L22-L26), [gininsert.c#buildFreshLeafTuple](../raw/postgres-19/src/backend/access/gin/gininsert.c#L290-L341), [ginblock.h#GinMaxItemSize](../raw/postgres-19/src/include/access/ginblock.h#L249-L253)).

Related: [Posting list](#posting-list), [GIN](#gin), [TID](#tid)

### Postmaster

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The postmaster is the first process of a server instance. It starts the auxiliary processes and creates a backend for each client connection ([glossary.sgml#glossary-postmaster](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1256-L1270)). It creates shared memory but avoids touching it, so a crashing backend cannot take it down. On a backend crash it resets the system ([postmaster.c header](../raw/postgres-17/src/backend/postmaster/postmaster.c#L3-L23)). Source entry points are `PostmasterMain`, the idle loop `ServerLoop`, and `BackendStartup` ([postmaster.c#PostmasterMain](../raw/postgres-17/src/backend/postmaster/postmaster.c#L486-L490), [postmaster.c#ServerLoop](../raw/postgres-17/src/backend/postmaster/postmaster.c#L1625-L1629), [postmaster.c#BackendStartup](../raw/postgres-17/src/backend/postmaster/postmaster.c#L3525-L3536)).

**Version notes:**
- PostgreSQL 12: Holds. The postmaster creates shared memory but avoids touching it and resets the system after a backend crash; `PostmasterMain`, `ServerLoop` and `BackendStartup` exist ([postmaster.c header](../raw/postgres-12/src/backend/postmaster/postmaster.c#L3-L23), [postmaster.c#PostmasterMain](../raw/postgres-12/src/backend/postmaster/postmaster.c#L567), [postmaster.c#ServerLoop](../raw/postgres-12/src/backend/postmaster/postmaster.c#L1620), [postmaster.c#BackendStartup](../raw/postgres-12/src/backend/postmaster/postmaster.c#L4059-L4060)).
- PostgreSQL 14: Holds. The postmaster creates shared memory but stays out of it, resets the system after a backend crash, and runs `PostmasterMain`, `ServerLoop` and `BackendStartup` ([glossary.sgml#glossary-postmaster](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1030-L1044), [postmaster.c header](../raw/postgres-14/src/backend/postmaster/postmaster.c#L12-L23), [postmaster.c#PostmasterMain](../raw/postgres-14/src/backend/postmaster/postmaster.c#L581), [postmaster.c#ServerLoop](../raw/postgres-14/src/backend/postmaster/postmaster.c#L1668), [postmaster.c#BackendStartup](../raw/postgres-14/src/backend/postmaster/postmaster.c#L4216)).
- PostgreSQL 18: Holds. It now tracks every child in a `PMChild` slot from `pmchild.c` ([postmaster.c header](../raw/postgres-18/src/backend/postmaster/postmaster.c#L3-L23), [postmaster.c#ServerLoop](../raw/postgres-18/src/backend/postmaster/postmaster.c#L1649-L1653), [postmaster.c#BackendStartup](../raw/postgres-18/src/backend/postmaster/postmaster.c#L3509-L3571), [pmchild.c header](../raw/postgres-18/src/backend/postmaster/pmchild.c#L1-L20)).
- PostgreSQL 19: Holds. The postmaster still forks backends, avoids touching shared memory and resets after a backend crash, and `PostmasterMain`, `ServerLoop` and `BackendStartup` are still the entry points ([glossary.sgml#glossary-postmaster](../raw/postgres-19/doc/src/sgml/glossary.sgml#L1353-L1368), [postmaster.c header](../raw/postgres-19/src/backend/postmaster/postmaster.c#L3-L23), [postmaster.c#PostmasterMain](../raw/postgres-19/src/backend/postmaster/postmaster.c#L498), [postmaster.c#ServerLoop](../raw/postgres-19/src/backend/postmaster/postmaster.c#L1679), [postmaster.c#BackendStartup](../raw/postgres-19/src/backend/postmaster/postmaster.c#L3592)).

Related: [Backend](#backend), [Background worker](#background-worker), [Crash recovery](#crash-recovery)

### Prepared statement

**Aliases:** `PREPARE`, `EXECUTE`, `DEALLOCATE`, extended query protocol, unnamed statement, `CachedPlanSource`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A prepared statement is a statement that has been parsed and analyzed once, saved under a name, and then executed many times with different parameter values. The SQL commands `PREPARE`, `EXECUTE` and `DEALLOCATE` are implemented in `commands/prepare.c` ([prepare.c header](../raw/postgres-17/src/backend/commands/prepare.c#L1-L5)). Prepared statements belong to one backend. They live in a per-backend hash table, so plans are not shared between sessions ([prepare.c#prepared_queries](../raw/postgres-17/src/backend/commands/prepare.c#L38-L44)). Protocol-level Parse and Bind messages do the same job. The unnamed statement is kept apart from that table to cut overhead for short-lived queries ([postgres.c#unnamed_stmt_psrc](../raw/postgres-17/src/backend/tcop/postgres.c#L158-L163)). Both paths store a `CachedPlanSource` in the plan cache. `EXECUTE` then asks `GetCachedPlan()` for a custom or generic plan ([prepare.c#ExecuteQuery](../raw/postgres-17/src/backend/commands/prepare.c#L147-L193), [plancache.c header](../raw/postgres-17/src/backend/utils/cache/plancache.c#L3-L12)).

**Version notes:**
- PostgreSQL 12: Holds ([prepare.c#prepared_queries](../raw/postgres-12/src/backend/commands/prepare.c#L46), [prepare.c#ExecuteQuery](../raw/postgres-12/src/backend/commands/prepare.c#L200-L246), [postgres.c#unnamed_stmt_psrc](../raw/postgres-12/src/backend/tcop/postgres.c#L158)).
- PostgreSQL 14: Holds ([prepare.c#prepared_queries](../raw/postgres-14/src/backend/commands/prepare.c#L46), [prepare.c#ExecuteQuery](../raw/postgres-14/src/backend/commands/prepare.c#L184-L230), [postgres.c#unnamed_stmt_psrc](../raw/postgres-14/src/backend/tcop/postgres.c#L172)).
- PostgreSQL 18: Holds ([prepare.c#prepared_queries](../raw/postgres-18/src/backend/commands/prepare.c#L47), [prepare.c#ExecuteQuery](../raw/postgres-18/src/backend/commands/prepare.c#L150-L196), [postgres.c#unnamed_stmt_psrc](../raw/postgres-18/src/backend/tcop/postgres.c#L151)).
- PostgreSQL 19: Holds ([prepare.c#prepared_queries](../raw/postgres-19/src/backend/commands/prepare.c#L49), [prepare.c#ExecuteQuery](../raw/postgres-19/src/backend/commands/prepare.c#L152-L198), [postgres.c#unnamed_stmt_psrc](../raw/postgres-19/src/backend/tcop/postgres.c#L165)).

Related: [Custom and generic plan](#custom-and-generic-plan), [Plan cache mode](#plan-cache-mode), [Portal](#portal), [Planner](#planner)

### Progress reporting

**Aliases:** `pg_stat_progress_*` views, `pgstat_progress_update_param()`, `PROGRESS_*` constants, `ProgressCommandType`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Progress reporting is how a long-running command publishes how far it has got, through views such as `pg_stat_progress_vacuum` ([system_views.sql#pg_stat_progress_vacuum](../raw/postgres-17/src/backend/catalog/system_views.sql#L1209-L1227)). Each backend owns an array of 20 integer slots in its `PgBackendStatus` entry. The running command writes a slot with `pgstat_progress_update_param()`, which does nothing when `track_activities` is off ([backend_progress.h#PGSTAT_NUM_PROGRESS_PARAM](../raw/postgres-17/src/include/utils/backend_progress.h#L22-L33), [backend_progress.c#pgstat_progress_update_param](../raw/postgres-17/src/backend/utils/activity/backend_progress.c#L42-L62)). The meaning of each slot per command is defined in `commands/progress.h` ([progress.h:1-25](../raw/postgres-17/src/include/commands/progress.h#L1-L25)). Each view reads the raw slots with `pg_stat_get_progress_info('<COMMAND>')` and names them, for example `param1` as `phase` ([system_views.sql#pg_stat_progress_vacuum](../raw/postgres-17/src/backend/catalog/system_views.sql#L1209-L1227)). In PostgreSQL 17, the covered commands are VACUUM, ANALYZE, CLUSTER, CREATE INDEX, base backup and COPY ([backend_progress.h#ProgressCommandType](../raw/postgres-17/src/include/utils/backend_progress.h#L22-L31)).

**Version notes:**
- PostgreSQL 12: Differs. Only VACUUM, CLUSTER and CREATE INDEX report progress ([pgstat.h#ProgressCommandType](../raw/postgres-12/src/include/pgstat.h#L955-L963), [system_views.sql:949-994](../raw/postgres-12/src/backend/catalog/system_views.sql#L949-L994)). The update function lives in `postmaster/pgstat.c` ([pgstat.c#pgstat_progress_update_param](../raw/postgres-12/src/backend/postmaster/pgstat.c#L3213-L3220)).
- PostgreSQL 14: Holds, with the same six commands as 17 ([backend_progress.h#ProgressCommandType](../raw/postgres-14/src/include/utils/backend_progress.h#L22-L33), [backend_progress.c#pgstat_progress_update_param](../raw/postgres-14/src/backend/utils/activity/backend_progress.c#L47)).
- PostgreSQL 18: Holds, with the same six commands ([backend_progress.h#ProgressCommandType](../raw/postgres-18/src/include/utils/backend_progress.h#L22-L33)).
- PostgreSQL 19: Differs. `PROGRESS_COMMAND_CLUSTER` is gone. `PROGRESS_COMMAND_REPACK` and `PROGRESS_COMMAND_DATACHECKSUMS` are added ([backend_progress.h#ProgressCommandType](../raw/postgres-19/src/include/utils/backend_progress.h#L22-L34)). `pg_stat_progress_cluster` survives as a view over the new `pg_stat_progress_repack` ([system_views.sql#pg_stat_progress_cluster](../raw/postgres-19/src/backend/catalog/system_views.sql#L1380-L1399)).

Related: [Cumulative statistics](#cumulative-statistics), [VACUUM](#vacuum), [REPACK](#repack), [CONCURRENTLY](#concurrently)

### Pruning

**Aliases:** heap page pruning, HOT pruning, page defragmentation, `heap_page_prune_opt`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Pruning removes dead tuples from one heap page and shortens HOT chains. It can run without a full VACUUM. Defragmentation then moves the surviving tuples together so the freed space becomes usable ([README.HOT#Pruning](../raw/postgres-17/src/backend/access/heap/README.HOT#L199-L219)). Ordinary reads call `heap_page_prune_opt()` opportunistically. It returns early during recovery, when the page has no `pd_prune_xid`, or when that XID is not yet removable. Otherwise it prunes only when the page is short of free space ([pruneheap.c#heap_page_prune_opt](../raw/postgres-17/src/backend/access/heap/pruneheap.c#L180-L230)). VACUUM uses `heap_page_prune_and_freeze()`, which can also freeze tuples in the same pass ([pruneheap.c#heap_page_prune_and_freeze](../raw/postgres-17/src/backend/access/heap/pruneheap.c#L330-L352)). Pruning cannot remove index entries. So a chain's root line pointer stays while any chain member is live, and becomes "dead" when none is. The next regular VACUUM removes the index entries and then that line pointer ([README.HOT#root-line-pointer](../raw/postgres-17/src/backend/access/heap/README.HOT#L122-L128)).

**Version notes:**
- PostgreSQL 12: Differs in the VACUUM path. `heap_page_prune_opt()` returns early during recovery or when the page is not prunable, and prunes only when free space is short ([pruneheap.c#heap_page_prune_opt](../raw/postgres-12/src/backend/access/heap/pruneheap.c#L84-L140)). 12 has no `heap_page_prune_and_freeze()`: VACUUM calls `heap_page_prune()` and freezes in a separate step ([pruneheap.c#heap_page_prune](../raw/postgres-12/src/backend/access/heap/pruneheap.c#L180), [vacuumlazy.c:977](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L977)). The README describes pruning, defragmentation and dead root line pointers ([README.HOT#Pruning](../raw/postgres-12/src/backend/access/heap/README.HOT#L178-L195), [README.HOT#root-line-pointer](../raw/postgres-12/src/backend/access/heap/README.HOT#L116-L121)).
- PostgreSQL 14: Holds for opportunistic pruning; VACUUM uses a different function. `heap_page_prune_opt()` returns early in recovery or when `pd_prune_xid` is not removable, and prunes only when free space is short ([README.HOT#Pruning](../raw/postgres-14/src/backend/access/heap/README.HOT#L178-L196), [pruneheap.c#heap_page_prune_opt](../raw/postgres-14/src/backend/access/heap/pruneheap.c#L121-L203)). 14 has no `heap_page_prune_and_freeze()`: VACUUM calls `heap_page_prune()`, and freezing is a separate step ([vacuumlazy.c:1741](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L1741), [pruneheap.c#heap_page_prune](../raw/postgres-14/src/backend/access/heap/pruneheap.c#L243)). The root line pointer rule is the same ([README.HOT](../raw/postgres-14/src/backend/access/heap/README.HOT#L114-L118)).
- PostgreSQL 18: Holds ([README.HOT#Pruning](../raw/postgres-18/src/backend/access/heap/README.HOT#L199-L219), [pruneheap.c#heap_page_prune_opt](../raw/postgres-18/src/backend/access/heap/pruneheap.c#L180-L230), [pruneheap.c#heap_page_prune_and_freeze](../raw/postgres-18/src/backend/access/heap/pruneheap.c#L330-L352)).
- PostgreSQL 19: Holds for the definition, the early returns (recovery, no `pd_prune_xid`, XID not removable) and the free-space trigger in `heap_page_prune_opt()`, and for VACUUM's `heap_page_prune_and_freeze()`. The root line pointer still turns "dead" until VACUUM removes it ([README.HOT#Pruning](../raw/postgres-19/src/backend/access/heap/README.HOT#L199-L219), [pruneheap.c#heap_page_prune_opt](../raw/postgres-19/src/backend/access/heap/pruneheap.c#L272-L325), [pruneheap.c#heap_page_prune_and_freeze](../raw/postgres-19/src/backend/access/heap/pruneheap.c#L1090-L1113), [README.HOT#root-line-pointer](../raw/postgres-19/src/backend/access/heap/README.HOT#L124-L128)). In v19, on-access pruning takes a visibility-map buffer and, for a scan the planner marked read-only, can also set the page all-visible in the VM ([pruneheap.c#heap_page_prune_opt](../raw/postgres-19/src/backend/access/heap/pruneheap.c#L255-L262), [pruneheap.c:359-361](../raw/postgres-19/src/backend/access/heap/pruneheap.c#L359-L361)).

Related: [Freezing](#freezing), [HOT](#hot), [Line pointer](#line-pointer), [VACUUM](#vacuum)

### Publication

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A publication is a named set of table changes that a publisher node offers to subscribers. It exists in one database and can limit which DML kinds it sends ([logical-replication.sgml#logical-replication-publication](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L100-L122)). The `pg_publication` catalog stores it, with flags for all tables, each operation kind, and publishing partition changes through the root ([pg_publication.h#FormData_pg_publication](../raw/postgres-17/src/include/catalog/pg_publication.h#L29-L57)). `CREATE PUBLICATION` runs `CreatePublication` ([publicationcmds.c#CreatePublication](../raw/postgres-17/src/backend/commands/publicationcmds.c#L733)).

**Version notes:**
- PostgreSQL 12: Differs. `pg_publication` has `puballtables` and per-operation flags, but no `pubviaroot`, so 12 cannot publish partition changes through the root ([pg_publication.h#FormData_pg_publication](../raw/postgres-12/src/include/catalog/pg_publication.h#L29-L55)). `CreatePublication` handles `CREATE PUBLICATION` ([publicationcmds.c#CreatePublication](../raw/postgres-12/src/backend/commands/publicationcmds.c#L140), [logical-replication.sgml#logical-replication-publication](../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L97-L120)).
- PostgreSQL 14: Holds. A publication lives in one database and can limit DML kinds, and `pg_publication` has `puballtables`, the per-operation flags and `pubviaroot`; `CREATE PUBLICATION` runs `CreatePublication` ([logical-replication.sgml#logical-replication-publication](../raw/postgres-14/doc/src/sgml/logical-replication.sgml#L97-L122), [pg_publication.h#FormData_pg_publication](../raw/postgres-14/src/include/catalog/pg_publication.h#L29-L57), [publicationcmds.c#CreatePublication](../raw/postgres-14/src/backend/commands/publicationcmds.c#L150)).
- PostgreSQL 18: Holds. `pg_publication` gains `pubgencols`, which says whether stored generated columns are published ([pg_publication.h#FormData_pg_publication](../raw/postgres-18/src/include/catalog/pg_publication.h#L29-L63), [publicationcmds.c#CreatePublication](../raw/postgres-18/src/backend/commands/publicationcmds.c#L832)).
- PostgreSQL 19: Holds. A publication is still a named change set in one database stored in `pg_publication`, and `CREATE PUBLICATION` still runs `CreatePublication()` ([logical-replication.sgml#logical-replication-publication](../raw/postgres-19/doc/src/sgml/logical-replication.sgml#L104-L108), [pg_publication.h#FormData_pg_publication](../raw/postgres-19/src/include/catalog/pg_publication.h#L31-L71), [publicationcmds.c#CreatePublication](../raw/postgres-19/src/backend/commands/publicationcmds.c#L836)). v19 publications can hold sequences (`FOR ALL SEQUENCES`, flag `puballsequences`), and `FOR ALL TABLES` can exclude tables with `EXCEPT` ([logical-replication.sgml#logical-replication-publication](../raw/postgres-19/doc/src/sgml/logical-replication.sgml#L114-L123), [pg_publication.h:49](../raw/postgres-19/src/include/catalog/pg_publication.h#L49)).

Related: [Logical replication](#logical-replication), [Subscription](#subscription), [Catalog](#catalog)

### Qual

**Aliases:** qualifier, qualification, `quals`, restriction clause, `RestrictInfo`, `baserestrictinfo`, one-time filter. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

"Qual" is source-code shorthand for a qualification: a boolean condition from `WHERE`, `JOIN ... ON` or a policy that rows must pass. The planner splits a condition into its ANDed pieces and wraps each in a `RestrictInfo` node ([pathnodes.h#RestrictInfo](../raw/postgres-17/src/include/nodes/pathnodes.h#L2412-L2421)). A piece that refers to one base relation goes on that relation's `baserestrictinfo` list ([pathnodes.h#RelOptInfo](../raw/postgres-17/src/include/nodes/pathnodes.h#L740-L742)). A "pseudoconstant" piece, with no Vars of the current query level and no volatile functions, can become a one-time qual in a gating `Result` node ([pathnodes.h:2531-2540](../raw/postgres-17/src/include/nodes/pathnodes.h#L2531-L2540)). In the finished plan, every `Plan` node carries a `qual` list of implicitly ANDed conditions ([plannodes.h#Plan](../raw/postgres-17/src/include/nodes/plannodes.h#L153)). The executor tests those conditions with `ExecQual()` ([executor.h#ExecQual](../raw/postgres-17/src/include/executor/executor.h#L407-L417)). `EXPLAIN` prints an index scan's index quals as `Index Cond` and the remaining quals as `Filter` ([explain.c:1969-1977](../raw/postgres-17/src/backend/commands/explain.c#L1969-L1977)).

**Version notes:**
- PostgreSQL 12: Holds ([pathnodes.h#RestrictInfo](../raw/postgres-12/src/include/nodes/pathnodes.h#L1796-L1805), [plannodes.h#Plan](../raw/postgres-12/src/include/nodes/plannodes.h#L141), [executor.h#ExecQual](../raw/postgres-12/src/include/executor/executor.h#L364), [explain.c:1583-1589](../raw/postgres-12/src/backend/commands/explain.c#L1583-L1589)).
- PostgreSQL 14: Holds ([pathnodes.h#RestrictInfo](../raw/postgres-14/src/include/nodes/pathnodes.h#L1913-L1922), [plannodes.h#Plan](../raw/postgres-14/src/include/nodes/plannodes.h#L142), [executor.h#ExecQual](../raw/postgres-14/src/include/executor/executor.h#L402)).
- PostgreSQL 18: Holds ([pathnodes.h#RestrictInfo](../raw/postgres-18/src/include/nodes/pathnodes.h#L2551-L2560), [plannodes.h#Plan](../raw/postgres-18/src/include/nodes/plannodes.h#L204), [executor.h#ExecQual](../raw/postgres-18/src/include/executor/executor.h#L515)).
- PostgreSQL 19: Holds ([pathnodes.h#RestrictInfo](../raw/postgres-19/src/include/nodes/pathnodes.h#L2746-L2755), [plannodes.h#Plan](../raw/postgres-19/src/include/nodes/plannodes.h#L237), [executor.h#ExecQual](../raw/postgres-19/src/include/executor/executor.h#L527)).

Related: [Planner](#planner), [Executor](#executor), [Selectivity](#selectivity), [RelOptInfo](#reloptinfo), [EXPLAIN](#explain)

### Query jumbling

**Aliases:** query ID, `queryid`, `query_id`, `compute_query_id`, `JumbleQuery()`, query fingerprint, `queryjumblefuncs.c`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Query jumbling computes a fingerprint of an analyzed query so that statements differing only in constants get the same query ID. It serializes only the fields judged essential, skipping details such as the values of constants. The 64-bit hash goes into `Query.queryId` at the end of parse analysis and travels into the plan ([queryjumblefuncs.c header](../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L6-L22)). The per-node jumbling code is generated from node definitions. Fields marked `query_jumble_ignore` are left out ([nodes.h:104-106](../raw/postgres-17/src/include/nodes/nodes.h#L104-L106), [queryjumblefuncs.c:233](../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L233)). `compute_query_id` switches it `off`, `on` or `auto`. Under `auto`, IDs are computed only after a module such as `pg_stat_statements` calls `EnableQueryId()` ([queryjumble.h#IsQueryIdEnabled](../raw/postgres-17/src/include/nodes/queryjumble.h#L72-L84), [queryjumblefuncs.c#EnableQueryId](../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L145-L154)). The setting has context `superuser`, so a superuser can change it per session ([guc_tables.c:4788](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4788)).

**Version notes:**
- PostgreSQL 12: Differs. Core computes no query ID and has no `compute_query_id`. `Query.queryId` exists for plugins to fill ([parsenodes.h#Query](../raw/postgres-12/src/include/nodes/parsenodes.h#L116)), and `pg_stat_statements` does its own jumbling ([pg_stat_statements.c:16-28](../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L16-L28)).
- PostgreSQL 14: Differs. Jumbling is in core, but in the hand-written `utils/misc/queryjumble.c` ([queryjumble.c header](../raw/postgres-14/src/backend/utils/misc/queryjumble.c#L1-L12), [queryjumble.c#JumbleQuery](../raw/postgres-14/src/backend/utils/misc/queryjumble.c#L101)). `compute_query_id` has the same values and is `superuser` context, so session scope for a superuser ([queryjumble.h#ComputeQueryIdType](../raw/postgres-14/src/include/utils/queryjumble.h#L55-L62), [guc.c:4669](../raw/postgres-14/src/backend/utils/misc/guc.c#L4669)).
- PostgreSQL 18: Differs slightly. `Query.queryId` is a signed `int64` rather than `uint64` ([parsenodes.h#Query](../raw/postgres-18/src/include/nodes/parsenodes.h#L136)). `compute_query_id` is still `superuser` context ([guc_tables.c:5051](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5051)).
- PostgreSQL 19: Holds as in 18 ([parsenodes.h#Query](../raw/postgres-19/src/include/nodes/parsenodes.h#L139), [queryjumblefuncs.c#JumbleQuery](../raw/postgres-19/src/backend/nodes/queryjumblefuncs.c#L139)). `compute_query_id` is `PGC_SUSET`, so session scope for a superuser ([guc_parameters.dat:512](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L512)).

Related: [pg_stat_statements](#pg_stat_statements), [Parse tree](#parse-tree), [Cumulative statistics](#cumulative-statistics)

### Range table

**Aliases:** `RangeTblEntry`, RTE, `rtable`, range-table index (RT index), `RTEKind`, `rt_fetch()`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The range table is the list of every relation-like thing a query reads from: tables, subqueries, joins, functions, `VALUES` lists, CTEs and more. Each element is a `RangeTblEntry` whose `rtekind` says which kind it is ([parsenodes.h#RangeTblEntry](../raw/postgres-17/src/include/nodes/parsenodes.h#L964-L976), [parsenodes.h#RTEKind](../raw/postgres-17/src/include/nodes/parsenodes.h#L1023-L1036)). The list hangs off `Query.rtable` and survives into `PlannedStmt.rtable` ([parsenodes.h#Query](../raw/postgres-17/src/include/nodes/parsenodes.h#L168), [plannodes.h#PlannedStmt](../raw/postgres-17/src/include/nodes/plannodes.h#L72)). Code refers to an entry by its 1-based position, the RT index. A `Var`'s `varno` is such an index, and `rt_fetch()` turns an index back into the entry ([primnodes.h#Var](../raw/postgres-17/src/include/nodes/primnodes.h#L251-L255), [parsetree.h#rt_fetch](../raw/postgres-17/src/include/parser/parsetree.h#L26-L32)). Permission checks do not live in the RTE; `perminfoindex` points to a separate `RTEPermissionInfo` ([parsenodes.h:1077-1079](../raw/postgres-17/src/include/nodes/parsenodes.h#L1077-L1079), [plannodes.h#PlannedStmt](../raw/postgres-17/src/include/nodes/plannodes.h#L74)).

**Version notes:**
- PostgreSQL 12: Differs. The RTE itself carries the permission fields such as `requiredPerms`, and there is no `RTEPermissionInfo` ([parsenodes.h#RangeTblEntry](../raw/postgres-12/src/include/nodes/parsenodes.h#L1096)). The kinds match 17 ([parsenodes.h#RTEKind](../raw/postgres-12/src/include/nodes/parsenodes.h#L955-L968), [parsetree.h#rt_fetch](../raw/postgres-12/src/include/parser/parsetree.h#L31-L32)).
- PostgreSQL 14: Differs as in 12: `requiredPerms` is still on the RTE ([parsenodes.h#RangeTblEntry](../raw/postgres-14/src/include/nodes/parsenodes.h#L1144), [parsenodes.h#RTEKind](../raw/postgres-14/src/include/nodes/parsenodes.h#L969-L982)).
- PostgreSQL 18: Differs. A new `RTE_GROUP` kind represents the grouping step ([parsenodes.h#RTEKind](../raw/postgres-18/src/include/nodes/parsenodes.h#L1039-L1053)).
- PostgreSQL 19: Differs. `RTE_GROUP` remains, and `RTE_GRAPH_TABLE` is added for the `GRAPH_TABLE` clause ([parsenodes.h#RTEKind](../raw/postgres-19/src/include/nodes/parsenodes.h#L1095-L1110)).

Related: [Parse tree](#parse-tree), [Planner](#planner), [RelOptInfo](#reloptinfo), [PlannedStmt](#plannedstmt)

### Read stream

**Aliases:** streaming read, `ReadStream`, `read_stream.c`, `read_stream_begin_relation()`, `read_stream_next_buffer()`, `io_combine_limit`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A read stream is a helper that reads a relation's blocks ahead of the code that uses them. The caller supplies a callback that yields block numbers. The stream merges neighbouring blocks into reads of up to `io_combine_limit` and starts them early through `StartReadBuffers()`, with a look-ahead distance that adapts to whether I/O is needed ([read_stream.c header](../raw/postgres-17/src/backend/storage/aio/read_stream.c#L3-L18)). This replaces calling `ReadBuffer()` one block at a time. The caller opens a stream with `read_stream_begin_relation()` and takes pinned buffers from it with `read_stream_next_buffer()`. The `READ_STREAM_MAINTENANCE` and `READ_STREAM_SEQUENTIAL` flags describe the access pattern ([read_stream.h](../raw/postgres-17/src/include/storage/read_stream.h#L27-L59)). In PostgreSQL 17, heap sequential scans and ANALYZE use it ([heapam.c:1252](../raw/postgres-17/src/backend/access/heap/heapam.c#L1252), [analyze.c:1199](../raw/postgres-17/src/backend/commands/analyze.c#L1199)). `io_combine_limit` has context `user`, so it can be set per session ([guc_tables.c:3139-3140](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3139-L3140)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. There is no `storage/aio/` directory, and heap scans read one block at a time with `ReadBufferExtended()` ([heapam.c#heapgetpage](../raw/postgres-12/src/backend/access/heap/heapam.c#L381)).
- PostgreSQL 14: Not present in PostgreSQL 14. Heap scans also use `ReadBufferExtended()` ([heapam.c#heapgetpage](../raw/postgres-14/src/backend/access/heap/heapam.c#L416)).
- PostgreSQL 18: Differs. The same API now sits beside the asynchronous I/O subsystem in `storage/aio/` ([read_stream.c header](../raw/postgres-18/src/backend/storage/aio/read_stream.c#L3-L18)). VACUUM's heap scan and B-tree vacuum also use streams ([vacuumlazy.c:1236](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L1236), [nbtree.c:1270](../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1270)). `io_combine_limit` is still `user` context, so session scope. It is capped by the new `io_max_combine_limit`, which is `postmaster` context, so changing it needs a restart ([guc_tables.c:3279-3295](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3279-L3295)).
- PostgreSQL 19: Differs. More callers use streams, including hash and GIN vacuum and `pgstatindex` ([hash.c:524](../raw/postgres-19/src/backend/access/hash/hash.c#L524), [ginvacuum.c:829](../raw/postgres-19/src/backend/access/gin/ginvacuum.c#L829), [pgstatindex.c:292](../raw/postgres-19/contrib/pgstattuple/pgstatindex.c#L292)). `io_combine_limit` is `PGC_USERSET`, so session scope ([guc_parameters.dat:1364](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1364)).

Related: [Buffer manager](#buffer-manager), [Asynchronous I/O](#asynchronous-io), [Ring buffer](#ring-buffer), [VACUUM](#vacuum)

### Regression test

**Aliases:** `pg_regress`, `make check`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A regression test runs SQL scripts and compares their output with stored expected output ([pg_regress.c header](../raw/postgres-17/src/test/regress/pg_regress.c#L3)). The driver writes results under `results/` and diffs them against `expected/`. Any difference is saved in `regression.diffs` ([regress.sgml](../raw/postgres-17/doc/src/sgml/regress.sgml#L483-L492)). `parallel_schedule` lists which core tests run together ([parallel_schedule](../raw/postgres-17/src/test/regress/parallel_schedule#L12-L17)).

**Version notes:**
- PostgreSQL 12: Holds. `pg_regress` diffs `results/` against `expected/` into `regression.diffs`, and `parallel_schedule` groups tests ([pg_regress.c header](../raw/postgres-12/src/test/regress/pg_regress.c#L3), [regress.sgml](../raw/postgres-12/doc/src/sgml/regress.sgml#L443-L450), [parallel_schedule](../raw/postgres-12/src/test/regress/parallel_schedule#L13-L16)).
- PostgreSQL 14: Holds. `pg_regress` compares `results/` with `expected/`, writes differences to `regression.diffs`, and `parallel_schedule` groups the tests ([pg_regress.c header](../raw/postgres-14/src/test/regress/pg_regress.c#L3), [regress.sgml](../raw/postgres-14/doc/src/sgml/regress.sgml#L464-L469), [parallel_schedule](../raw/postgres-14/src/test/regress/parallel_schedule#L12-L17)).
- PostgreSQL 18: Holds ([pg_regress.c header](../raw/postgres-18/src/test/regress/pg_regress.c#L3), [regress.sgml](../raw/postgres-18/doc/src/sgml/regress.sgml#L517-L526), [parallel_schedule](../raw/postgres-18/src/test/regress/parallel_schedule#L12-L17)).
- PostgreSQL 19: Holds. `pg_regress` still diffs `results/` against `expected/` into `regression.diffs`, and `parallel_schedule` still groups the core tests ([pg_regress.c header](../raw/postgres-19/src/test/regress/pg_regress.c#L3), [regress.sgml](../raw/postgres-19/doc/src/sgml/regress.sgml#L617-L623), [parallel_schedule](../raw/postgres-19/src/test/regress/parallel_schedule#L10-L18)).

Related: [Isolation test](#isolation-test), [TAP test](#tap-test)

### REINDEX

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`REINDEX` rebuilds an index from its table's data and replaces the old copy. Use it for a corrupt index, a bloated index, a changed storage parameter such as `fillfactor`, or an invalid index left by a failed concurrent build ([ref/reindex.sgml](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L37-L82)). Without `CONCURRENTLY`, `reindex_index()` locks the table with `ShareLock` and the index with `AccessExclusiveLock`. It then gives the index a new relfilenumber with `RelationSetNewRelfilenumber()` and calls `index_build()` ([index.c:3600-3613](../raw/postgres-17/src/backend/catalog/index.c#L3600-L3613), [index.c:3652-3654](../raw/postgres-17/src/backend/catalog/index.c#L3652-L3654), [index.c:3784-3789](../raw/postgres-17/src/backend/catalog/index.c#L3784-L3789)). `REINDEX CONCURRENTLY` takes a different path, `ReindexRelationConcurrently()` ([indexcmds.c#ReindexRelationConcurrently](../raw/postgres-17/src/backend/commands/indexcmds.c#L3428-L3451)).

**Version notes:**
- PostgreSQL 12: Holds, with older names. `reindex_index()` locks the table `ShareLock` and the index `AccessExclusiveLock`, then calls `RelationSetNewRelfilenode()` (not `...Relfilenumber`) and `index_build()` ([index.c#reindex_index](../raw/postgres-12/src/backend/catalog/index.c#L3450-L3470), [index.c:3526-3530](../raw/postgres-12/src/backend/catalog/index.c#L3526-L3530)). `REINDEX CONCURRENTLY` goes through `ReindexRelationConcurrently()` ([indexcmds.c#ReindexRelationConcurrently](../raw/postgres-12/src/backend/commands/indexcmds.c#L2715-L2739)).
- PostgreSQL 14: Holds, with older names. The reasons to rebuild are the same ([ref/reindex.sgml](../raw/postgres-14/doc/src/sgml/ref/reindex.sgml#L37-L82)). `reindex_index()` locks the table with `ShareLock` and the index with `AccessExclusiveLock`, then calls `RelationSetNewRelfilenode()` (not `RelationSetNewRelfilenumber()`) and `index_build()` ([index.c:3600-3602](../raw/postgres-14/src/backend/catalog/index.c#L3600-L3602), [index.c:3639-3641](../raw/postgres-14/src/backend/catalog/index.c#L3639-L3641), [index.c:3758-3762](../raw/postgres-14/src/backend/catalog/index.c#L3758-L3762)). `REINDEX CONCURRENTLY` uses `ReindexRelationConcurrently()` ([indexcmds.c#ReindexRelationConcurrently](../raw/postgres-14/src/backend/commands/indexcmds.c#L3384)).
- PostgreSQL 18: Holds ([ref/reindex.sgml](../raw/postgres-18/doc/src/sgml/ref/reindex.sgml#L37-L82), [index.c:3654-3667](../raw/postgres-18/src/backend/catalog/index.c#L3654-L3667), [index.c:3838-3843](../raw/postgres-18/src/backend/catalog/index.c#L3838-L3843)).
- PostgreSQL 19: Holds. The use cases are unchanged, and `reindex_index()` still takes `ShareLock` on the table and `AccessExclusiveLock` on the index, sets a new relfilenumber and calls `index_build()`, while `CONCURRENTLY` still goes through `ReindexRelationConcurrently()` ([ref/reindex.sgml](../raw/postgres-19/doc/src/sgml/ref/reindex.sgml#L37-L82), [index.c:3673-3686](../raw/postgres-19/src/backend/catalog/index.c#L3673-L3686), [index.c:3722-3727](../raw/postgres-19/src/backend/catalog/index.c#L3722-L3727), [index.c:3858-3862](../raw/postgres-19/src/backend/catalog/index.c#L3858-L3862), [indexcmds.c#ReindexRelationConcurrently](../raw/postgres-19/src/backend/commands/indexcmds.c#L3612)).

Related: [CONCURRENTLY](#concurrently), [Invalid index](#invalid-index), [Bloat](#bloat), [Relfilenumber](#relfilenumber), [AccessExclusiveLock](#accessexclusivelock)

### Relation size functions

**Aliases:** `pg_relation_size`, `pg_table_size`, `pg_indexes_size`, `pg_total_relation_size`, `dbsize.c`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The relation size functions report how many bytes a table or index occupies on disk. Each measures a different set of files. The shared helper `calculate_relation_size()` sums the sizes of one [fork](#fork)'s segment files, using `stat()` on each file until one is missing ([dbsize.c#calculate_relation_size](../raw/postgres-17/src/backend/utils/adt/dbsize.c#L300-L342)).

| Function | Counts | Source |
|---|---|---|
| `pg_relation_size(rel [, fork])` | one fork, `main` by default | [system_functions.sql:285-289](../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289), [dbsize.c#pg_relation_size](../raw/postgres-17/src/backend/utils/adt/dbsize.c#L346-L371) |
| `pg_table_size(rel)` | every fork, plus the [TOAST](#toast) table and its index, but not the table's own indexes | [dbsize.c#calculate_table_size](../raw/postgres-17/src/backend/utils/adt/dbsize.c#L415-L440) |
| `pg_indexes_size(rel)` | every fork of every index on the table | [dbsize.c#calculate_indexes_size](../raw/postgres-17/src/backend/utils/adt/dbsize.c#L445-L470) |
| `pg_total_relation_size(rel)` | `pg_table_size` plus `pg_indexes_size` | [dbsize.c#calculate_total_relation_size](../raw/postgres-17/src/backend/utils/adt/dbsize.c#L523-L540) |

`pg_relation_size()` returns NULL instead of an error for a relation dropped while the query runs. It takes only an `AccessShareLock` ([dbsize.c#pg_relation_size](../raw/postgres-17/src/backend/utils/adt/dbsize.c#L353-L369)).

**Version notes:**
- PostgreSQL 12: Holds ([dbsize.c#pg_relation_size](../raw/postgres-12/src/backend/utils/adt/dbsize.c#L311), [dbsize.c#calculate_table_size](../raw/postgres-12/src/backend/utils/adt/dbsize.c#L389), [dbsize.c#calculate_total_relation_size](../raw/postgres-12/src/backend/utils/adt/dbsize.c#L493)). The one-argument `pg_relation_size` is a SQL wrapper whose body is stored in `pg_proc.dat` rather than `system_functions.sql` ([pg_proc.dat:6883-6887](../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6887)).
- PostgreSQL 14: Holds; the wrapper moves to `system_functions.sql` ([system_functions.sql:278](../raw/postgres-14/src/backend/catalog/system_functions.sql#L278), [dbsize.c#pg_relation_size](../raw/postgres-14/src/backend/utils/adt/dbsize.c#L311)).
- PostgreSQL 18: Holds ([system_functions.sql:285](../raw/postgres-18/src/backend/catalog/system_functions.sql#L285), [dbsize.c#pg_relation_size](../raw/postgres-18/src/backend/utils/adt/dbsize.c#L364), [dbsize.c#calculate_table_size](../raw/postgres-18/src/backend/utils/adt/dbsize.c#L442)).
- PostgreSQL 19: Holds ([system_functions.sql:269](../raw/postgres-19/src/backend/catalog/system_functions.sql#L269), [dbsize.c#pg_relation_size](../raw/postgres-19/src/backend/utils/adt/dbsize.c#L364), [dbsize.c#calculate_table_size](../raw/postgres-19/src/backend/utils/adt/dbsize.c#L442)).

Related: [Fork](#fork), [TOAST](#toast), [Bloat](#bloat), [pgstattuple](#pgstattuple), [reltuples and relpages](#reltuples-and-relpages)

### Relcache

**Aliases:** relation cache, `RelationData`, `Relation`, `RelationIdGetRelation`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The relcache is each backend's private cache of relation descriptors. A descriptor is a `RelationData` struct built from the relation's catalog rows ([relcache.c:1-22](../raw/postgres-17/src/backend/utils/cache/relcache.c#L1-L22), [rel.h#RelationData](../raw/postgres-17/src/include/utils/rel.h#L51-L62)). `RelationIdGetRelation` returns the cached entry or builds one, and the caller must already hold a lock on the relation ([relcache.c#RelationIdGetRelation](../raw/postgres-17/src/backend/utils/cache/relcache.c#L2068-L2082)). An entry carries `rd_rel` (the `pg_class` row), `rd_index` for an index, and `rd_indam`, the index access method's routine table ([rel.h:111](../raw/postgres-17/src/include/utils/rel.h#L111), [rel.h:192](../raw/postgres-17/src/include/utils/rel.h#L192), [rel.h:206](../raw/postgres-17/src/include/utils/rel.h#L206)). Catalog changes invalidate entries at command boundaries and, after commit, in other backends ([inval.c:1-35](../raw/postgres-17/src/backend/utils/cache/inval.c#L1-L35)). The relcache caches whole relation descriptors. The [syscache](#syscache) caches individual catalog rows.

**Version notes:**
- PostgreSQL 12: Holds. `RelationIdGetRelation` returns or builds a `RelationData`, the caller must hold a lock, and entries carry `rd_rel`, `rd_index` and `rd_indam` ([relcache.c:1-22](../raw/postgres-12/src/backend/utils/cache/relcache.c#L1-L22), [rel.h#RelationData](../raw/postgres-12/src/include/utils/rel.h#L53), [relcache.c#RelationIdGetRelation](../raw/postgres-12/src/backend/utils/cache/relcache.c#L1983-L1991), [rel.h:84](../raw/postgres-12/src/include/utils/rel.h#L84), [rel.h:144](../raw/postgres-12/src/include/utils/rel.h#L144), [rel.h:158](../raw/postgres-12/src/include/utils/rel.h#L158)). Invalidation works at command boundaries and commit ([inval.c:1-35](../raw/postgres-12/src/backend/utils/cache/inval.c#L1-L35)).
- PostgreSQL 14: Holds. The relcache holds `RelationData` entries with `rd_rel`, `rd_index` and `rd_indam`, `RelationIdGetRelation` expects the caller to hold a lock, and `inval.c` handles invalidation at command boundaries and commit ([relcache.c:1-22](../raw/postgres-14/src/backend/utils/cache/relcache.c#L1-L22), [rel.h#RelationData](../raw/postgres-14/src/include/utils/rel.h#L54), [relcache.c#RelationIdGetRelation](../raw/postgres-14/src/backend/utils/cache/relcache.c#L2057-L2065), [rel.h:109](../raw/postgres-14/src/include/utils/rel.h#L109), [rel.h:187](../raw/postgres-14/src/include/utils/rel.h#L187), [rel.h:201](../raw/postgres-14/src/include/utils/rel.h#L201), [inval.c:1-35](../raw/postgres-14/src/backend/utils/cache/inval.c#L1-L35)).
- PostgreSQL 18: Holds ([relcache.c:1-22](../raw/postgres-18/src/backend/utils/cache/relcache.c#L1-L22), [rel.h#RelationData](../raw/postgres-18/src/include/utils/rel.h#L51-L62), [relcache.c#RelationIdGetRelation](../raw/postgres-18/src/backend/utils/cache/relcache.c#L2087-L2101), [inval.c:1-35](../raw/postgres-18/src/backend/utils/cache/inval.c#L1-L35)).
- PostgreSQL 19: Holds. The relcache still caches `RelationData` descriptors, `RelationIdGetRelation()` still expects the caller to hold a lock, and entries still carry `rd_rel`, `rd_index` and `rd_indam`, invalidated at command boundaries and after commit ([relcache.c:1-22](../raw/postgres-19/src/backend/utils/cache/relcache.c#L1-L22), [rel.h#RelationData](../raw/postgres-19/src/include/utils/rel.h#L55), [relcache.c#RelationIdGetRelation](../raw/postgres-19/src/backend/utils/cache/relcache.c#L2078-L2093), [rel.h:111](../raw/postgres-19/src/include/utils/rel.h#L111), [rel.h:192](../raw/postgres-19/src/include/utils/rel.h#L192), [inval.c:1-35](../raw/postgres-19/src/backend/utils/cache/inval.c#L1-L35)). `rd_indam` is now a pointer to a `const` routine struct ([rel.h:206](../raw/postgres-19/src/include/utils/rel.h#L206)).

Related: [Syscache](#syscache), [pg_class](#pg_class), [pg_index](#pg_index), [Access method](#access-method)

### Relfilenumber

**Aliases:** relfilenode, `RelFileNumber`, `RelFileLocator`, `pg_class.relfilenode`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A relfilenumber is the number in a relation's data file name. It is stored in `pg_class.relfilenode` and is separate from the relation's OID ([relpath.h#RelFileNumber](../raw/postgres-17/src/include/common/relpath.h#L22-L28), [pg_class.h#relfilenode](../raw/postgres-17/src/include/catalog/pg_class.h#L56-L57)). Keeping them separate lets a rewrite swap in new physical files while the relation keeps its OID ([cluster.c#swap_relation_files](../raw/postgres-17/src/backend/commands/cluster.c#L1035-L1040)). `RelFileLocator` combines tablespace OID, database OID and relfilenumber to identify the physical storage. A `relfilenode` of 0 marks a "mapped" catalog, whose file number is kept by `relmapper.c` ([relfilelocator.h#RelFileLocator](../raw/postgres-17/src/include/storage/relfilelocator.h#L33-L63)).

**Version notes:**
- PostgreSQL 12: Differs in naming only. 12 has no `RelFileNumber` type or `RelFileLocator`; the file number is an `Oid` called `relfilenode`, and `RelFileNode` (`spcNode`, `dbNode`, `relNode`) identifies storage ([pg_class.h:52-54](../raw/postgres-12/src/include/catalog/pg_class.h#L52-L54), [relfilenode.h#RelFileNode](../raw/postgres-12/src/include/storage/relfilenode.h#L50-L62)). A 0 still marks a mapped catalog, and rewrites swap files while the OID stays ([cluster.c#swap_relation_files](../raw/postgres-12/src/backend/commands/cluster.c#L1002)).
- PostgreSQL 14: Differs in naming. 14 has no `RelFileNumber` or `RelFileLocator`. The file number is an `Oid` in `pg_class.relfilenode`, and `RelFileNode` (`spcNode`, `dbNode`, `relNode`) identifies the storage; a `relfilenode` of 0 still marks a mapped catalog ([pg_class.h#relfilenode](../raw/postgres-14/src/include/catalog/pg_class.h#L55-L57), [relfilenode.h#RelFileNode](../raw/postgres-14/src/include/storage/relfilenode.h#L30-L62)). `swap_relation_files` swaps the relfilenodes while the OID stays ([cluster.c#swap_relation_files](../raw/postgres-14/src/backend/commands/cluster.c#L1005-L1010)).
- PostgreSQL 18: Holds ([relpath.h#RelFileNumber](../raw/postgres-18/src/include/common/relpath.h#L22-L28), [pg_class.h#relfilenode](../raw/postgres-18/src/include/catalog/pg_class.h#L56-L57), [relfilelocator.h#RelFileLocator](../raw/postgres-18/src/include/storage/relfilelocator.h#L33-L63)).
- PostgreSQL 19: Holds. `RelFileNumber` is still stored in `pg_class.relfilenode`, a rewrite still swaps it while the OID stays, `RelFileLocator` still combines tablespace, database and relfilenumber, and 0 still marks a mapped catalog ([relpath.h#RelFileNumber](../raw/postgres-19/src/include/common/relpath.h#L22-L25), [pg_class.h#relfilenode](../raw/postgres-19/src/include/catalog/pg_class.h#L58-L59), [repack.c#swap_relation_files](../raw/postgres-19/src/backend/commands/repack.c#L1480-L1486), [relfilelocator.h#RelFileLocator](../raw/postgres-19/src/include/storage/relfilelocator.h#L20-L63)). The swap code now lives in `repack.c` ([repack.c#swap_relation_files](../raw/postgres-19/src/backend/commands/repack.c#L1507)).

Related: [Fork](#fork), [OID](#oid), [pg_class](#pg_class), [Table rewrite](#table-rewrite)

### RelOptInfo

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`RelOptInfo` is the planner's working record for one relation. That relation can be a base table, a join, or an upper-level step such as grouping [pathnodes.h#RelOptKind](../raw/postgres-17/src/include/nodes/pathnodes.h#L819-L827). It holds the candidate paths (`pathlist`), the cheapest startup and total paths, and, for a base table, the list of `IndexOptInfo` nodes [pathnodes.h:663-697](../raw/postgres-17/src/include/nodes/pathnodes.h#L663-L697). It also holds size estimates taken from `pg_class`: `pages`, `tuples`, and `allvisfrac` [pathnodes.h:938-944](../raw/postgres-17/src/include/nodes/pathnodes.h#L938-L944).

**Version notes:**
- PostgreSQL 12: Holds. The same `RelOptKind` values, `pathlist`, cheapest paths, `indexlist`, and `pages`/`tuples`/`allvisfrac` exist ([pathnodes.h#RelOptKind](../raw/postgres-12/src/include/nodes/pathnodes.h#L597-L606), [pathnodes.h:653-680](../raw/postgres-12/src/include/nodes/pathnodes.h#L653-L680)).
- PostgreSQL 14: Holds. `RelOptKind` covers base, join and upper rels, and `RelOptInfo` holds `pathlist`, the cheapest paths, `indexlist`, and `pages`, `tuples` and `allvisfrac` ([pathnodes.h#RelOptKind](../raw/postgres-14/src/include/nodes/pathnodes.h#L639-L650), [pathnodes.h:695-722](../raw/postgres-14/src/include/nodes/pathnodes.h#L695-L722)).
- PostgreSQL 18: Holds ([pathnodes.h#RelOptKind](../raw/postgres-18/src/include/nodes/pathnodes.h#L849-L857), [pathnodes.h:693-727](../raw/postgres-18/src/include/nodes/pathnodes.h#L693-L727), [pathnodes.h:968-974](../raw/postgres-18/src/include/nodes/pathnodes.h#L968-L974)).
- PostgreSQL 19: Holds. `RelOptInfo` still covers base, join and upper relations and holds `pathlist`, the cheapest paths, `indexlist`, `pages`, `tuples` and `allvisfrac` ([pathnodes.h#RelOptKind](../raw/postgres-19/src/include/nodes/pathnodes.h#L975-L983), [pathnodes.h:1050-1054](../raw/postgres-19/src/include/nodes/pathnodes.h#L1050-L1054), [pathnodes.h:1091](../raw/postgres-19/src/include/nodes/pathnodes.h#L1091), [pathnodes.h:1095-1097](../raw/postgres-19/src/include/nodes/pathnodes.h#L1095-L1097)). v19 adds `pgs_mask`, the set of path strategies allowed for this relation ([pathnodes.h:1039](../raw/postgres-19/src/include/nodes/pathnodes.h#L1039), [pathnodes.h:24-35](../raw/postgres-19/src/include/nodes/pathnodes.h#L24-L35)).

Related: [Path](#path), [IndexOptInfo](#indexoptinfo), [Planner](#planner), [reltuples and relpages](#reltuples-and-relpages)

### reltuples and relpages

**Aliases:** `pg_class.reltuples`, `pg_class.relpages`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

These two `pg_class` columns hold a relation's estimated row count and page count. Neither is always up to date, and `reltuples = -1` means unknown [pg_class.h:61-66](../raw/postgres-17/src/include/catalog/pg_class.h#L61-L66). `VACUUM`, `ANALYZE`, and some DDL such as `CREATE INDEX` update them. For a table, `reltuples` counts live rows only [catalogs.sgml#relpages](../raw/postgres-17/doc/src/sgml/catalogs.sgml#L2020-L2046) [vacuum.c#vac_update_relstats](../raw/postgres-17/src/backend/commands/vacuum.c#L1404-L1415). The planner does not use `relpages` as the size. It takes the current block count and multiplies it by the density `reltuples / relpages` [tableam.c#table_block_relation_estimate_size](../raw/postgres-17/src/backend/access/table/tableam.c#L666-L667) [tableam.c:711-747](../raw/postgres-17/src/backend/access/table/tableam.c#L711-L747).

**Version notes:**
- PostgreSQL 12: Differs. The columns hold estimated rows and pages, updated by `VACUUM`, `ANALYZE` and some DDL ([catalogs.sgml#relpages](../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1760-L1785), [vacuum.c#vac_update_relstats](../raw/postgres-12/src/backend/commands/vacuum.c#L1157)). 12 has no `reltuples = -1` convention; the header says only "not always up-to-date", and "never vacuumed" is approximated by `relpages = 0` ([pg_class.h:59-63](../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63), [heapam_handler.c#heapam_estimate_rel_size](../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2106-L2118)). The planner scales the current block count by `reltuples / relpages` in `heapam_estimate_rel_size()`, not in `tableam.c` ([heapam_handler.c:2131-2134](../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2131-L2134)).
- PostgreSQL 14: Holds. The columns are estimates, `reltuples = -1` means unknown, VACUUM, ANALYZE and `CREATE INDEX` update them, and the planner scales the current block count by `reltuples / relpages` ([pg_class.h:61-66](../raw/postgres-14/src/include/catalog/pg_class.h#L61-L66), [catalogs.sgml#relpages](../raw/postgres-14/doc/src/sgml/catalogs.sgml#L1979-L2004), [vacuum.c#vac_update_relstats](../raw/postgres-14/src/backend/commands/vacuum.c#L1317-L1431), [tableam.c#table_block_relation_estimate_size](../raw/postgres-14/src/backend/access/table/tableam.c#L678-L748)).
- PostgreSQL 18: Holds. `vac_update_relstats()` now also takes a count of all-frozen pages for the new `relallfrozen` column ([pg_class.h:61-72](../raw/postgres-18/src/include/catalog/pg_class.h#L61-L72), [vacuum.c#vac_update_relstats](../raw/postgres-18/src/backend/commands/vacuum.c#L1436-L1447), [tableam.c:711-747](../raw/postgres-18/src/backend/access/table/tableam.c#L711-L747)).
- PostgreSQL 19: Holds. Both are estimates, `reltuples = -1` still means unknown, VACUUM, ANALYZE and some DDL update them, and the planner still scales the current block count by `reltuples / relpages` ([pg_class.h:63-68](../raw/postgres-19/src/include/catalog/pg_class.h#L63-L68), [catalogs.sgml#relpages](../raw/postgres-19/doc/src/sgml/catalogs.sgml#L2052-L2080), [vacuum.c#vac_update_relstats](../raw/postgres-19/src/backend/commands/vacuum.c#L1432), [tableam.c#table_block_relation_estimate_size](../raw/postgres-19/src/backend/access/table/tableam.c#L718-L778)).

Related: [pg_class](#pg_class), [Statistics](#statistics), [RelOptInfo](#reloptinfo), [IndexOptInfo](#indexoptinfo), [VACUUM](#vacuum)

### Reorder buffer

**Aliases:** `ReorderBuffer`, `reorderbuffer.c`, transaction reassembly, spill to disk, `logical_decoding_work_mem`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The reorder buffer is the part of logical decoding that collects decoded changes per transaction, in WAL order, and hands a transaction to the output plugin only when it commits. It also stitches subtransactions into their top-level transaction and reassembles TOAST chunks ([reorderbuffer.c NOTES](../raw/postgres-17/src/backend/replication/logical/reorderbuffer.c#L13-L45)). Large transactions can spill to disk. When total decoded-change memory passes `logical_decoding_work_mem`, the buffer picks the largest transaction and evicts it ([reorderbuffer.c NOTES](../raw/postgres-17/src/backend/replication/logical/reorderbuffer.c#L32-L57)). If the plugin supports streaming, `ReorderBufferCheckMemoryLimit()` streams an in-progress transaction instead of writing it to disk ([reorderbuffer.c#ReorderBufferCheckMemoryLimit](../raw/postgres-17/src/backend/replication/logical/reorderbuffer.c#L3770-L3848)). `logical_decoding_work_mem` has context `user`, so it can be set per session ([guc_tables.c:2477-2480](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2477-L2480)).

**Version notes:**
- PostgreSQL 12: Differs. There is no `logical_decoding_work_mem` and no streaming. A transaction spills to disk once it holds 4096 changes in memory ([reorderbuffer.c#max_changes_in_memory](../raw/postgres-12/src/backend/replication/logical/reorderbuffer.c#L153-L165), [reorderbuffer.c#ReorderBufferCheckSerializeTXN](../raw/postgres-12/src/backend/replication/logical/reorderbuffer.c#L2239-L2250)).
- PostgreSQL 14: Holds. The memory limit and streaming exist ([reorderbuffer.c#ReorderBufferCheckMemoryLimit](../raw/postgres-14/src/backend/replication/logical/reorderbuffer.c#L3641-L3672)). `logical_decoding_work_mem` is `user` context, so session scope ([guc.c:2439](../raw/postgres-14/src/backend/utils/misc/guc.c#L2439)).
- PostgreSQL 18: Holds ([reorderbuffer.c#ReorderBufferCheckMemoryLimit](../raw/postgres-18/src/backend/replication/logical/reorderbuffer.c#L3907-L3949)). `logical_decoding_work_mem` is `user` context, so session scope ([guc_tables.c:2605](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2605)).
- PostgreSQL 19: Holds ([reorderbuffer.c#ReorderBufferCheckMemoryLimit](../raw/postgres-19/src/backend/replication/logical/reorderbuffer.c#L3931-L3985)). `logical_decoding_work_mem` is `PGC_USERSET`, so session scope ([guc_parameters.dat:1927](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1927)). `REPACK (CONCURRENTLY)` also depends on logical decoding ([repack.c header](../raw/postgres-19/src/backend/commands/repack.c#L12-L21)).

Related: [Logical decoding](#logical-decoding), [Replication slot](#replication-slot), [TOAST](#toast), [Logical replication](#logical-replication)

### REPACK

**Aliases:** `REPACK`, `REPACK (CONCURRENTLY)`, `repack.c`, `pgrepack` output plugin, `pg_stat_progress_repack`, `max_repack_replication_slots`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

In PostgreSQL 19, `REPACK` is the command that rewrites a table into new files with no dead space, optionally in index order, and gives the freed space back to the operating system ([ref/repack.sgml#Description](../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L42-L73)). Its code in `repack.c` is the former CLUSTER code, and `VACUUM FULL` shares parts of it. By default it holds `AccessExclusiveLock` while it copies rows into a new relation and swaps the files ([repack.c header](../raw/postgres-19/src/backend/commands/repack.c#L3-L10)). With `CONCURRENTLY`, it holds only `ShareUpdateExclusiveLock` during the copy. A background worker uses logical decoding to capture concurrent changes; these are replayed on the new heap before a short `AccessExclusiveLock` for the swap ([repack.c header](../raw/postgres-19/src/backend/commands/repack.c#L12-L21)). The decoding uses a built-in output plugin, `pgrepack` ([pgrepack.c header](../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L1-L10)). Its replication slots come from `max_repack_replication_slots`, default 5. That setting has context `postmaster`, so changing it needs a restart ([guc_parameters.dat:2108-2114](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2108-L2114)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. `REPACK` is not a keyword ([kwlist.h:333-334](../raw/postgres-12/src/include/parser/kwlist.h#L333-L334)). Table rewrites for `CLUSTER` live in `cluster.c` ([cluster.c#cluster_rel](../raw/postgres-12/src/backend/commands/cluster.c#L266)).
- PostgreSQL 14: Not present in PostgreSQL 14 ([kwlist.h:346-347](../raw/postgres-14/src/include/parser/kwlist.h#L346-L347), [cluster.c#cluster_rel](../raw/postgres-14/src/backend/commands/cluster.c#L277)).
- PostgreSQL 17: Not present in PostgreSQL 17 ([kwlist.h:374-375](../raw/postgres-17/src/include/parser/kwlist.h#L374-L375), [cluster.c#cluster_rel](../raw/postgres-17/src/backend/commands/cluster.c#L311)).
- PostgreSQL 18: Not present in PostgreSQL 18 ([kwlist.h:376-377](../raw/postgres-18/src/include/parser/kwlist.h#L376-L377), [cluster.c#cluster_rel](../raw/postgres-18/src/backend/commands/cluster.c#L311)).
- PostgreSQL 19: Present, as described above ([kwlist.h:388](../raw/postgres-19/src/include/parser/kwlist.h#L388)).

Related: [CLUSTER](#cluster), [VACUUM FULL](#vacuum-full), [Table rewrite](#table-rewrite), [Logical decoding](#logical-decoding), [Reorder buffer](#reorder-buffer), [Progress reporting](#progress-reporting)

### Replication origin

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A replication origin names a source of replicated changes and records how far replay from it has progressed ([origin.c header](../raw/postgres-17/src/backend/replication/logical/origin.c#L13-L38)). Each origin has a two-byte internal id that is stored in WAL. A session that sets an origin marks the changes it produces with it, and output plugins can filter on it to prevent replication loops ([origin.c header](../raw/postgres-17/src/backend/replication/logical/origin.c#L18-L24), [origin.c header](../raw/postgres-17/src/backend/replication/logical/origin.c#L37-L43)). Each apply worker uses an origin named `pg_<suboid>`, and tablesync workers use `pg_<suboid>_<relid>` ([worker.c#ReplicationOriginNameForLogicalRep](../raw/postgres-17/src/backend/replication/logical/worker.c#L420-L442), [worker.c#run_apply_worker](../raw/postgres-17/src/backend/replication/logical/worker.c#L4516-L4525)). With `origin = none`, `pgoutput_origin_filter` drops every change that carries an origin ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L1722-L1735)).

**Version notes:**
- PostgreSQL 12: Differs. Origins exist with a 2-byte internal id and can be used to filter loops ([origin.c header](../raw/postgres-12/src/backend/replication/logical/origin.c#L13-L45)). In 12 only the main apply worker sets up an origin, named `pg_<suboid>`; table-sync workers do not use per-table origins ([worker.c#ApplyWorkerMain](../raw/postgres-12/src/backend/replication/logical/worker.c#L1703-L1731)). 12 has no subscription `origin` option, and `pgoutput_origin_filter` always returns false, so `pgoutput` never drops changes by origin ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-12/src/backend/replication/pgoutput/pgoutput.c#L433-L437)).
- PostgreSQL 14: Differs. Origins have a 2-byte internal id stored in WAL, and apply and tablesync workers use `pg_<suboid>` and `pg_<suboid>_<relid>` ([origin.c header](../raw/postgres-14/src/backend/replication/logical/origin.c#L13-L43), [worker.c:3230](../raw/postgres-14/src/backend/replication/logical/worker.c#L3230), [tablesync.c#ReplicationOriginNameForTablesync](../raw/postgres-14/src/backend/replication/logical/tablesync.c#L940-L943)). But 14 has no subscription `origin` option, and `pgoutput_origin_filter` always returns false, so `pgoutput` never drops changes by origin ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-14/src/backend/replication/pgoutput/pgoutput.c#L819-L823)).
- PostgreSQL 18: Holds ([origin.c header](../raw/postgres-18/src/backend/replication/logical/origin.c#L13-L38), [worker.c#ReplicationOriginNameForLogicalRep](../raw/postgres-18/src/backend/replication/logical/worker.c#L411-L433), [pgoutput.c#pgoutput_origin_filter](../raw/postgres-18/src/backend/replication/pgoutput/pgoutput.c#L1762-L1775)).
- PostgreSQL 19: Holds. Origins still have a 2-byte internal id stored in WAL, apply and tablesync workers still use `pg_<suboid>` and `pg_<suboid>_<relid>`, and `pgoutput_origin_filter()` still drops changes with an origin under `origin = none` ([origin.c header](../raw/postgres-19/src/backend/replication/logical/origin.c#L13-L43), [worker.c#ReplicationOriginNameForLogicalRep](../raw/postgres-19/src/backend/replication/logical/worker.c#L645-L663), [worker.c#run_apply_worker](../raw/postgres-19/src/backend/replication/logical/worker.c#L5725-L5730), [pgoutput.c#pgoutput_origin_filter](../raw/postgres-19/src/backend/replication/pgoutput/pgoutput.c#L1769-L1778)). The id type is now named `ReplOriginId` ([pgoutput.c#pgoutput_origin_filter](../raw/postgres-19/src/backend/replication/pgoutput/pgoutput.c#L1770)).

Related: [Apply worker](#apply-worker), [Subscription](#subscription), [Logical decoding](#logical-decoding)

### Replication slot

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A replication slot is persistent, crash-safe server state for one replication stream. Its main job is to stop the server from removing WAL or old row versions that the stream still needs ([slot.c header](../raw/postgres-17/src/backend/replication/slot.c#L15-L27)). `ReplicationSlotPersistentData` holds the slot's `xmin` and `catalog_xmin` horizons, its `restart_lsn`, and for logical slots its output plugin ([slot.h#ReplicationSlotPersistentData](../raw/postgres-17/src/include/replication/slot.h#L63-L96)). So a slot that stops advancing holds back both WAL removal and the removal of old row versions ([slot.c header](../raw/postgres-17/src/backend/replication/slot.c#L15-L18)). Each subscription receives its changes through one slot on the publisher ([logical-replication.sgml#logical-replication-subscription](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L197-L199)).

**Version notes:**
- PostgreSQL 12: Holds. Slots are persistent, crash-safe state that prevent early removal of WAL and old tuple versions, and `ReplicationSlotPersistentData` holds `xmin`, `catalog_xmin`, `restart_lsn` and the plugin ([slot.c header](../raw/postgres-12/src/backend/replication/slot.c#L13-L27), [slot.h#ReplicationSlotPersistentData](../raw/postgres-12/src/include/replication/slot.h#L43-L85)). Each subscription receives changes through one slot ([logical-replication.sgml#logical-replication-subscription](../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L188-L190)).
- PostgreSQL 14: Holds. A slot keeps WAL and old row versions its stream needs, with `xmin`, `catalog_xmin`, `restart_lsn` and `plugin` in `ReplicationSlotPersistentData`; each subscription uses one slot ([slot.c header](../raw/postgres-14/src/backend/replication/slot.c#L15-L18), [slot.h#ReplicationSlotPersistentData](../raw/postgres-14/src/include/replication/slot.h#L43-L100), [logical-replication.sgml#logical-replication-subscription](../raw/postgres-14/doc/src/sgml/logical-replication.sgml#L188-L190)).
- PostgreSQL 18: Holds ([slot.c header](../raw/postgres-18/src/backend/replication/slot.c#L15-L21), [slot.h#ReplicationSlotPersistentData](../raw/postgres-18/src/include/replication/slot.h#L70-L103), [logical-replication.sgml](../raw/postgres-18/doc/src/sgml/logical-replication.sgml#L223-L224)). New in 18: `idle_replication_slot_timeout` (default 0, off) invalidates slots idle longer than that. Its context is `sighup`, so a change needs a reload ([guc_tables.c#idle_replication_slot_timeout](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3109-L3117)).
- PostgreSQL 19: Holds. A slot still holds back WAL and old row versions through `xmin`, `catalog_xmin` and `restart_lsn`, and each subscription still uses one publisher slot ([slot.c header](../raw/postgres-19/src/backend/replication/slot.c#L13-L21), [slot.h#ReplicationSlotPersistentData](../raw/postgres-19/src/include/replication/slot.h#L95-L162), [logical-replication.sgml#logical-replication-subscription](../raw/postgres-19/doc/src/sgml/logical-replication.sgml#L232)). In v19, creating the first logical slot while `wal_level = replica` switches logical decoding on ([logicalctl.c header](../raw/postgres-19/src/backend/replication/logical/logicalctl.c#L5-L25)).

Related: [xmin horizon](#xmin-horizon), [WAL](#wal), [LSN](#lsn), [Logical decoding](#logical-decoding), [Subscription](#subscription)

### Rewriter

**Aliases:** query rewriter, rule system, rewrite rules, `QueryRewrite`, `rewriteHandler.c`, view expansion. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The rewriter is the stage between parse analysis and planning. It takes one analyzed `Query` tree and returns zero or more trees after applying rules ([rules.sgml:49-52](../raw/postgres-17/doc/src/sgml/rules.sgml#L49-L52)). `pg_rewrite_query()` sends every non-utility query to `QueryRewrite()`; utility statements skip it ([postgres.c#pg_rewrite_query](../raw/postgres-17/src/backend/tcop/postgres.c#L809-L829)). `QueryRewrite()` first applies non-`SELECT` rules in `RewriteQuery()`, then runs `fireRIRrules()` on each resulting query ([rewriteHandler.c#QueryRewrite](../raw/postgres-17/src/backend/rewrite/rewriteHandler.c#L4546-L4581)). `fireRIRrules()` expands views through `ApplyRetrieveRule()` and fetches row-level security quals through `get_row_security_policies()` ([rewriteHandler.c#fireRIRrules](../raw/postgres-17/src/backend/rewrite/rewriteHandler.c#L2155-L2172), [rewriteHandler.c#fireRIRrules](../raw/postgres-17/src/backend/rewrite/rewriteHandler.c#L2235-L2248)). "RIR" in the source means an `ON SELECT DO INSTEAD SELECT` rule, the kind every view has ([rewriteHandler.c:12-17](../raw/postgres-17/src/backend/rewrite/rewriteHandler.c#L12-L17)).

**Version notes:**
- PostgreSQL 12: Holds. `QueryRewrite()` and the row-security call in `fireRIRrules()` are present ([rewriteHandler.c#QueryRewrite](../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L3914-L3924), [rewriteHandler.c#fireRIRrules](../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L2041-L2045)).
- PostgreSQL 14: Holds ([rewriteHandler.c#QueryRewrite](../raw/postgres-14/src/backend/rewrite/rewriteHandler.c#L4357-L4367), [rewriteHandler.c#fireRIRrules](../raw/postgres-14/src/backend/rewrite/rewriteHandler.c#L2277-L2281)).
- PostgreSQL 18: Holds ([rewriteHandler.c#QueryRewrite](../raw/postgres-18/src/backend/rewrite/rewriteHandler.c#L4626-L4636), [rewriteHandler.c#fireRIRrules](../raw/postgres-18/src/backend/rewrite/rewriteHandler.c#L2251-L2255)).
- PostgreSQL 19: Holds ([rewriteHandler.c#QueryRewrite](../raw/postgres-19/src/backend/rewrite/rewriteHandler.c#L4786-L4796), [rewriteHandler.c#fireRIRrules](../raw/postgres-19/src/backend/rewrite/rewriteHandler.c#L2277-L2281)).

Related: [Parse tree](#parse-tree), [Planner](#planner), [Row-level security](#row-level-security), [Security barrier](#security-barrier), [Utility command](#utility-command)

### Ring buffer

**Aliases:** buffer access strategy, `BufferAccessStrategy`, `BAS_BULKREAD`, `BAS_BULKWRITE`, `BAS_VACUUM`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A ring buffer is a small, private set of shared buffers that one large operation reuses over and over. This stops a big scan, bulk write or VACUUM from pushing everything else out of the cache ([glossary.sgml#glossary-buffer-access-strategy](../raw/postgres-17/doc/src/sgml/glossary.sgml#L272-L297)). `GetAccessStrategy()` picks the default ring size: 256 kB for `BAS_BULKREAD`, 16 MB for `BAS_BULKWRITE` and 2 MB for `BAS_VACUUM`. `BAS_NORMAL` gets no ring ([freelist.c#GetAccessStrategy](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L535-L570)). VACUUM's ring size comes from `vacuum_buffer_usage_limit`, and VACUUM flushes WAL to reuse dirty ring buffers rather than dropping them from the ring ([buffer/README#ring](../raw/postgres-17/src/backend/storage/buffer/README#L205-L235)).

**Version notes:**
- PostgreSQL 12: Differs. `GetAccessStrategy()` gives 256 kB to `BAS_BULKREAD`, 16 MB to `BAS_BULKWRITE`, and only 256 kB to `BAS_VACUUM`, capped at `NBuffers / 8` ([freelist.c#GetAccessStrategy](../raw/postgres-12/src/backend/storage/buffer/freelist.c#L542-L576)). 12 has no `vacuum_buffer_usage_limit`; VACUUM flushes WAL to reuse dirty ring buffers ([buffer/README#ring](../raw/postgres-12/src/backend/storage/buffer/README#L235-L240)).
- PostgreSQL 14: Differs on VACUUM. `GetAccessStrategy()` gives 256 kB to `BAS_BULKREAD`, 16 MB to `BAS_BULKWRITE`, and a fixed 256 kB to `BAS_VACUUM` (not 2 MB); `BAS_NORMAL` gets no ring ([freelist.c#GetAccessStrategy](../raw/postgres-14/src/backend/storage/buffer/freelist.c#L555-L568)). 14 has no `vacuum_buffer_usage_limit` GUC and no docs-glossary entry for buffer access strategy; VACUUM still flushes WAL to reuse dirty ring buffers ([buffer/README#ring](../raw/postgres-14/src/backend/storage/buffer/README#L232-L236)).
- PostgreSQL 18: Differs for bulk reads. `BAS_BULKWRITE` (16 MB) and `BAS_VACUUM` (2 MB) are unchanged. `BAS_BULKREAD` now starts at 256 kB and grows by `BLCKSZ * io_combine_limit * effective_io_concurrency`, capped by the backend's pin limit ([freelist.c#GetAccessStrategy](../raw/postgres-18/src/backend/storage/buffer/freelist.c#L535-L615)). `effective_io_concurrency` defaults to 16 and has `user` context (session scope) ([guc_tables.c#effective_io_concurrency](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3250-L3258), [bufmgr.h:161](../raw/postgres-18/src/include/storage/bufmgr.h#L161)). The README's VACUUM ring rules are unchanged ([buffer/README#ring](../raw/postgres-18/src/backend/storage/buffer/README#L208-L238)).
- PostgreSQL 19: Holds for the purpose, `BAS_BULKWRITE` at 16 MB, `BAS_VACUUM` at 2 MB, no ring for `BAS_NORMAL`, and VACUUM flushing WAL to reuse dirty ring buffers ([glossary.sgml#glossary-buffer-access-strategy](../raw/postgres-19/doc/src/sgml/glossary.sgml#L313-L339), [freelist.c#GetAccessStrategy](../raw/postgres-19/src/backend/storage/buffer/freelist.c#L426-L495), [buffer/README#ring](../raw/postgres-19/src/backend/storage/buffer/README#L206-L240)). `BAS_BULKREAD` differs: it starts at 256 kB and grows by `io_combine_limit × effective_io_concurrency` blocks, capped by the backend's pin limit ([freelist.c#GetAccessStrategy](../raw/postgres-19/src/backend/storage/buffer/freelist.c#L442-L487)). `vacuum_buffer_usage_limit` is `PGC_USERSET` (session scope) ([guc_parameters.dat#vacuum_buffer_usage_limit](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3321-L3329)).

Related: [Buffer manager](#buffer-manager), [Clock sweep](#clock-sweep), [VACUUM](#vacuum)

### Row lock

**Aliases:** tuple lock, row-level lock, `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`, `LockTupleMode`, `heap_lock_tuple`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A row lock blocks other writers and lockers of one row, not readers, and lasts until transaction end or savepoint rollback ([mvcc.sgml#locking-rows](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1241-L1256)). There are four modes, carried in source by `LockTupleMode`: key share, share, no-key exclusive and exclusive. `UPDATE` takes no-key exclusive unless it changes key columns, and `DELETE` takes exclusive ([lockoptions.h#LockTupleMode](../raw/postgres-17/src/include/nodes/lockoptions.h#L46-L59)). Row locks do not live in the shared lock table. `heap_lock_tuple()` writes the locker's XID into the tuple's xmax and sets infomask bits such as `HEAP_XMAX_LOCK_ONLY` and `HEAP_XMAX_KEYSHR_LOCK` ([README.tuplock](../raw/postgres-17/src/backend/access/heap/README.tuplock#L1-L13), [htup_details.h:194-197](../raw/postgres-17/src/include/access/htup_details.h#L194-L197), [heapam.c#heap_lock_tuple](../raw/postgres-17/src/backend/access/heap/heapam.c#L4775-L4801)). When several transactions lock one row, xmax holds a MultiXactId instead ([README.tuplock](../raw/postgres-17/src/backend/access/heap/README.tuplock#L10-L13)). The heavyweight lock manager is used only to queue waiters fairly ([README.tuplock](../raw/postgres-17/src/backend/access/heap/README.tuplock#L15-L36)).

**Version notes:**
- PostgreSQL 12: Holds; same four modes and the same README ([lockoptions.h#LockTupleMode](../raw/postgres-12/src/include/nodes/lockoptions.h#L46-L59), [README.tuplock](../raw/postgres-12/src/backend/access/heap/README.tuplock#L1-L13), [heapam.c:3973](../raw/postgres-12/src/backend/access/heap/heapam.c#L3973)).
- PostgreSQL 14: Holds ([lockoptions.h#LockTupleMode](../raw/postgres-14/src/include/nodes/lockoptions.h#L46-L59), [README.tuplock](../raw/postgres-14/src/backend/access/heap/README.tuplock#L1-L13), [heapam.c:4523](../raw/postgres-14/src/backend/access/heap/heapam.c#L4523)).
- PostgreSQL 18: Holds ([lockoptions.h#LockTupleMode](../raw/postgres-18/src/include/nodes/lockoptions.h#L46-L59), [README.tuplock](../raw/postgres-18/src/backend/access/heap/README.tuplock#L1-L13), [heapam.c:4777](../raw/postgres-18/src/backend/access/heap/heapam.c#L4777)).
- PostgreSQL 19: Holds ([lockoptions.h#LockTupleMode](../raw/postgres-19/src/include/nodes/lockoptions.h#L47-L60), [README.tuplock](../raw/postgres-19/src/backend/access/heap/README.tuplock#L1-L13), [heapam.c:4728](../raw/postgres-19/src/backend/access/heap/heapam.c#L4728)).

Related: [MultiXact](#multixact), [xmin and xmax](#xmin-and-xmax), [Heavyweight lock](#heavyweight-lock), [Tuple](#tuple)

### Row-level security

**Aliases:** RLS, row security policy, `CREATE POLICY`, `pg_policy`, `relrowsecurity`, `relforcerowsecurity`, `BYPASSRLS`, `row_security`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Row-level security lets a table restrict, per user, which rows normal queries can return or change. With RLS enabled and no policy defined, the default is deny: no rows are visible ([ddl.sgml#ddl-rowsecurity](../raw/postgres-17/doc/src/sgml/ddl.sgml#L2519-L2540)). Each policy is a row of `pg_policy`, which stores its command, permissive or restrictive flag, roles, `USING` qual and `WITH CHECK` qual ([pg_policy.h#FormData_pg_policy](../raw/postgres-17/src/include/catalog/pg_policy.h#L29-L44)). `pg_class.relrowsecurity` and `relforcerowsecurity` switch it on per table ([pg_class.h:107-111](../raw/postgres-17/src/include/catalog/pg_class.h#L107-L111)). `check_enable_rls()` decides whether RLS applies: `BYPASSRLS` roles and superusers skip it, and owners skip it unless the table has `FORCE ROW LEVEL SECURITY` ([rls.c#check_enable_rls](../raw/postgres-17/src/backend/utils/misc/rls.c#L75-L117)). The rewriter adds the policy quals as security-barrier quals, so they run before user conditions unless those are leakproof ([rewriteHandler.c#fireRIRrules](../raw/postgres-17/src/backend/rewrite/rewriteHandler.c#L2243-L2248), [ddl.sgml#ddl-rowsecurity](../raw/postgres-17/doc/src/sgml/ddl.sgml#L2552-L2559)). The `row_security` GUC has context `user`, so it is set per session; turning it off makes a query that RLS would filter raise an error instead ([guc_tables.c#row_security](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1595-L1603), [rls.c#check_enable_rls](../raw/postgres-17/src/backend/utils/misc/rls.c#L119-L127)).

**Version notes:**
- PostgreSQL 12: Holds. `pg_policy`, the `pg_class` flags, `rolbypassrls` and `check_enable_rls()` all exist; `row_security` is defined in `guc.c` with context `user` (session scope) ([pg_policy.h#FormData_pg_policy](../raw/postgres-12/src/include/catalog/pg_policy.h#L29-L44), [pg_class.h:105-108](../raw/postgres-12/src/include/catalog/pg_class.h#L105-L108), [pg_authid.h:41](../raw/postgres-12/src/include/catalog/pg_authid.h#L41), [rls.c#check_enable_rls](../raw/postgres-12/src/backend/utils/misc/rls.c#L52), [guc.c#row_security](../raw/postgres-12/src/backend/utils/misc/guc.c#L1571-L1579)).
- PostgreSQL 14: Holds; `row_security` is in `guc.c`, context `user` ([pg_class.h:108-111](../raw/postgres-14/src/include/catalog/pg_class.h#L108-L111), [rls.c#check_enable_rls](../raw/postgres-14/src/backend/utils/misc/rls.c#L52), [guc.c#row_security](../raw/postgres-14/src/backend/utils/misc/guc.c#L1723-L1731)).
- PostgreSQL 18: Holds; `row_security` context `user` ([pg_class.h:111-114](../raw/postgres-18/src/include/catalog/pg_class.h#L111-L114), [rls.c#check_enable_rls](../raw/postgres-18/src/backend/utils/misc/rls.c#L52), [guc_tables.c#row_security](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1696-L1704)).
- PostgreSQL 19: Holds. The `row_security` definition moved to the `guc_parameters.dat` data file, still `PGC_USERSET` (session scope) ([pg_policy.h:31](../raw/postgres-19/src/include/catalog/pg_policy.h#L31), [pg_class.h:113-116](../raw/postgres-19/src/include/catalog/pg_class.h#L113-L116), [guc_parameters.dat#row_security](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2579-L2584)).

Related: [Rewriter](#rewriter), [Security barrier](#security-barrier), [Leakproof function](#leakproof-function), [SECURITY DEFINER](#security-definer), [pg_class](#pg_class), [GUC context](#guc-context)

### Security barrier

**Aliases:** `security_barrier` view option, security barrier quals, `securityQuals`, `security_level`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A security barrier is a rule that some filter conditions must run before any user-supplied condition that could leak the rows they hide. Without it, a user function in the outer query can see rows that a view's `WHERE` clause was meant to hide ([rules.sgml](../raw/postgres-17/doc/src/sgml/rules.sgml#L2082-L2091), [rules.sgml](../raw/postgres-17/doc/src/sgml/rules.sgml#L2140-L2150)). Views opt in with the `security_barrier` reloption, and row-level security policies always get this treatment ([reloptions.c#view_reloptions](../raw/postgres-17/src/backend/access/common/reloptions.c#L2006-L2016), [rewriteHandler.c#fireRIRrules](../raw/postgres-17/src/backend/rewrite/rewriteHandler.c#L2243-L2248)). In source the quals travel in `RangeTblEntry.securityQuals` ([parsenodes.h#RangeTblEntry](../raw/postgres-17/src/include/nodes/parsenodes.h#L1249-L1250)). The planner then tags each qual with a `security_level`. `order_qual_clauses()` runs lower levels first, and only leakproof quals may move ahead ([createplan.c#order_qual_clauses](../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L5283-L5293)). This ordering can make barrier views slower, so the option is off by default ([rules.sgml](../raw/postgres-17/doc/src/sgml/rules.sgml#L2146-L2150)).

**Version notes:**
- PostgreSQL 12: Holds; the view reloption, `securityQuals` and the same ordering rule exist ([reloptions.c#view_reloptions](../raw/postgres-12/src/backend/access/common/reloptions.c#L1443-L1455), [parsenodes.h:1102](../raw/postgres-12/src/include/nodes/parsenodes.h#L1102), [createplan.c#order_qual_clauses](../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4919-L4929)).
- PostgreSQL 14: Holds ([reloptions.c#view_reloptions](../raw/postgres-14/src/backend/access/common/reloptions.c#L2004-L2008), [parsenodes.h:1150](../raw/postgres-14/src/include/nodes/parsenodes.h#L1150), [createplan.c#order_qual_clauses](../raw/postgres-14/src/backend/optimizer/plan/createplan.c#L5192-L5202)).
- PostgreSQL 18: Holds ([reloptions.c#view_reloptions](../raw/postgres-18/src/backend/access/common/reloptions.c#L2034-L2038), [parsenodes.h:1278](../raw/postgres-18/src/include/nodes/parsenodes.h#L1278), [createplan.c#order_qual_clauses](../raw/postgres-18/src/backend/optimizer/plan/createplan.c#L5387-L5397)).
- PostgreSQL 19: Holds ([reloptions.c#view_reloptions](../raw/postgres-19/src/backend/access/common/reloptions.c#L2143-L2147), [parsenodes.h:1341](../raw/postgres-19/src/include/nodes/parsenodes.h#L1341), [createplan.c#order_qual_clauses](../raw/postgres-19/src/backend/optimizer/plan/createplan.c#L5234-L5244)).

Related: [Row-level security](#row-level-security), [Leakproof function](#leakproof-function), [Rewriter](#rewriter), [Planner](#planner)

### SECURITY DEFINER

**Aliases:** security definer function, `prosecdef`, `fmgr_security_definer`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A `SECURITY DEFINER` function runs with the privileges of its owner instead of its caller. The flag is `pg_proc.prosecdef` ([pg_proc.h:60-62](../raw/postgres-17/src/include/catalog/pg_proc.h#L60-L62)). When `fmgr_info` sees `prosecdef`, a `proconfig` setting, or a function-entry hook, it routes the call through `fmgr_security_definer()` ([fmgr.c#fmgr_info_cxt_security](../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L189-L213)). That wrapper switches the current user to the owner for the call and restores it afterwards ([fmgr.c#fmgr_security_definer](../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L700-L712)). The switch also changes which row-level security policies apply inside the body, because `check_enable_rls()` tests the current user ([rls.c#check_enable_rls](../raw/postgres-17/src/backend/utils/misc/rls.c#L52-L55)). The planner never inlines a SQL function marked `SECURITY DEFINER` ([clauses.c#inline_function](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L4587-L4597)).

**Version notes:**
- PostgreSQL 12: Holds ([pg_proc.h:63](../raw/postgres-12/src/include/catalog/pg_proc.h#L63), [fmgr.c:189-205](../raw/postgres-12/src/backend/utils/fmgr/fmgr.c#L189-L205), [fmgr.c#fmgr_security_definer](../raw/postgres-12/src/backend/utils/fmgr/fmgr.c#L660), [clauses.c#inline_function](../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4434-L4440)).
- PostgreSQL 14: Holds ([pg_proc.h:62](../raw/postgres-14/src/include/catalog/pg_proc.h#L62), [fmgr.c:191-205](../raw/postgres-14/src/backend/utils/fmgr/fmgr.c#L191-L205), [clauses.c#inline_function](../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4422-L4428)).
- PostgreSQL 18: Holds ([pg_proc.h:62](../raw/postgres-18/src/include/catalog/pg_proc.h#L62), [fmgr.c:191-205](../raw/postgres-18/src/backend/utils/fmgr/fmgr.c#L191-L205), [clauses.c#inline_function](../raw/postgres-18/src/backend/optimizer/util/clauses.c#L4611-L4617)).
- PostgreSQL 19: Holds ([pg_proc.h:64](../raw/postgres-19/src/include/catalog/pg_proc.h#L64), [fmgr.c:193-207](../raw/postgres-19/src/backend/utils/fmgr/fmgr.c#L193-L207), [clauses.c#inline_function](../raw/postgres-19/src/backend/optimizer/util/clauses.c#L5361-L5367)).

Related: [fmgr](#fmgr), [Row-level security](#row-level-security), [SQL function inlining](#sql-function-inlining), [Hook](#hook)

### Selectivity

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Selectivity is the estimated fraction of rows that pass a condition, from 0 to 1. The C type `Selectivity` is a `double` [nodes.h:250](../raw/postgres-17/src/include/nodes/nodes.h#L250). `clauselist_selectivity()` estimates a list of `AND`ed conditions. It applies extended statistics first and multiplies the remaining estimates, even though conditions are often not independent [clausesel.c#clauselist_selectivity](../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L57-L72). When no statistics apply, `selfuncs.h` supplies defaults, such as 0.005 for equality and 1/3 for an inequality [selfuncs.h:33-40](../raw/postgres-17/src/include/utils/selfuncs.h#L33-L40).

**Version notes:**
- PostgreSQL 12: Holds. `Selectivity` is a `double`, `clauselist_selectivity()` applies extended statistics first, and the defaults are 0.005 for equality and 1/3 for inequality ([nodes.h:657](../raw/postgres-12/src/include/nodes/nodes.h#L657), [clausesel.c#clauselist_selectivity](../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L60-L100), [selfuncs.h:33-36](../raw/postgres-12/src/include/utils/selfuncs.h#L33-L36)).
- PostgreSQL 14: Holds. `Selectivity` is a `double`, `clauselist_selectivity()` applies extended statistics before multiplying the rest, and `selfuncs.h` defines defaults such as `DEFAULT_EQ_SEL` 0.005 ([nodes.h:672](../raw/postgres-14/src/include/nodes/nodes.h#L672), [clausesel.c#clauselist_selectivity](../raw/postgres-14/src/backend/optimizer/path/clausesel.c#L102-L110), [clausesel.c:114-135](../raw/postgres-14/src/backend/optimizer/path/clausesel.c#L114-L135), [selfuncs.h:30-40](../raw/postgres-14/src/include/utils/selfuncs.h#L30-L40)).
- PostgreSQL 18: Holds ([nodes.h:256](../raw/postgres-18/src/include/nodes/nodes.h#L256), [clausesel.c#clauselist_selectivity](../raw/postgres-18/src/backend/optimizer/path/clausesel.c#L57-L72), [selfuncs.h:33-40](../raw/postgres-18/src/include/utils/selfuncs.h#L33-L40)).
- PostgreSQL 19: Holds. `Selectivity` is still a `double`, `clauselist_selectivity()` still applies extended statistics first and multiplies the rest, and the defaults are still 0.005 for equality and 1/3 for inequality ([nodes.h:258](../raw/postgres-19/src/include/nodes/nodes.h#L258), [clausesel.c#clauselist_selectivity](../raw/postgres-19/src/backend/optimizer/path/clausesel.c#L55-L100), [selfuncs.h:30-37](../raw/postgres-19/src/include/utils/selfuncs.h#L30-L37)).

Related: [Statistics](#statistics), [Cost](#cost), [Planner](#planner)

### Sequential scan

**Aliases:** Seq Scan node, `SeqScan`, `nodeSeqscan.c`, full table scan. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A sequential scan reads every page of a table in order and tests each row against the query's conditions. It is the baseline the [planner](#planner) compares every index path against ([nodeSeqscan.c header](../raw/postgres-17/src/backend/executor/nodeSeqscan.c#L15-L26)). In the executor, `SeqNext()` opens a table scan and pulls rows with `table_scan_getnextslot()`, so the table [access method](#access-method) does the page reading ([nodeSeqscan.c#SeqNext](../raw/postgres-17/src/backend/executor/nodeSeqscan.c#L49-L82)). For the heap in PostgreSQL 17, that reading goes through a [read stream](#read-stream) created in `heap_beginscan()`, which can prefetch and combine block reads ([heapam.c#heap_beginscan](../raw/postgres-17/src/backend/access/heap/heapam.c#L1242-L1258)). `cost_seqscan()` charges `seq_page_cost` for every table page plus `cpu_tuple_cost` and qual cost for every tuple. `enable_seqscan = off` adds `disable_cost` rather than removing the path ([costsize.c#cost_seqscan](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L284-L322)). Both `enable_seqscan` and `seq_page_cost` have context `user`, so they can be set per session ([guc_tables.c:784](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L784), [guc_tables.c:3676](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3676)).

**Version notes:**
- PostgreSQL 12: Holds, but the heap reads page by page through `heapgetpage()`, with no read stream ([nodeSeqscan.c#SeqNext](../raw/postgres-12/src/backend/executor/nodeSeqscan.c#L50), [heapam.c#heapgetpage](../raw/postgres-12/src/backend/access/heap/heapam.c#L352), [costsize.c#cost_seqscan](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L211)).
- PostgreSQL 14: Same as 12 ([nodeSeqscan.c#SeqNext](../raw/postgres-14/src/backend/executor/nodeSeqscan.c#L50), [heapam.c#heapgetpage](../raw/postgres-14/src/backend/access/heap/heapam.c#L387), [costsize.c#cost_seqscan](../raw/postgres-14/src/backend/optimizer/path/costsize.c#L226)).
- PostgreSQL 18: `enable_seqscan = off` now marks the path as disabled by setting `disabled_nodes`, instead of adding `disable_cost` ([costsize.c#cost_seqscan](../raw/postgres-18/src/backend/optimizer/path/costsize.c#L357)). The executor adds specialized variants such as `ExecSeqScanWithQual()` ([nodeSeqscan.c:130](../raw/postgres-18/src/backend/executor/nodeSeqscan.c#L130)), and the heap still reads through a read stream ([heapam.c:1221](../raw/postgres-18/src/backend/access/heap/heapam.c#L1221)).
- PostgreSQL 19: The disabled flag comes from the relation's `pgs_mask` planner strategy mask rather than directly from `enable_seqscan` ([costsize.c#cost_seqscan](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L279), [costsize.c:332-336](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L332-L336)). The heap still uses a read stream ([heapam.c:1297](../raw/postgres-19/src/backend/access/heap/heapam.c#L1297)).

Related: [Index scan](#index-scan), [Bitmap scan](#bitmap-scan), [Cost](#cost), [Read stream](#read-stream), [Ring buffer](#ring-buffer)

### shared_buffers

**Aliases:** shared buffer pool, `NBuffers`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`shared_buffers` sets how many buffers are in the shared buffer pool. Its unit is blocks. The built-in default is 16384 blocks, which is 128 MB at 8 kB `BLCKSZ`. The GUC context is `postmaster`, so a change needs a restart ([guc_tables.c#shared_buffers](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2260-L2270)). In source, the value is the global `NBuffers` ([globals.c:139](../raw/postgres-17/src/backend/utils/init/globals.c#L139)). The clock hand wraps around modulo `NBuffers` ([freelist.c#ClockSweepTick](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L103-L125)). It is PostgreSQL's own cache; the server also relies on the operating system's cache ([config.sgml#guc-shared-buffers](../raw/postgres-17/doc/src/sgml/config.sgml#L1673-L1680)).

**Version notes:**
- PostgreSQL 12: Differs in the built-in default. It is `PGC_POSTMASTER` (restart) and backs the global `NBuffers`, but the compiled-in default is 1024 blocks (8 MB at 8 kB); `initdb` normally writes 128MB into `postgresql.conf` ([guc.c#shared_buffers](../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2163), [globals.c:131](../raw/postgres-12/src/backend/utils/init/globals.c#L131), [config.sgml#guc-shared-buffers](../raw/postgres-12/doc/src/sgml/config.sgml#L1488-L1500)). The clock hand wraps modulo `NBuffers` ([freelist.c#ClockSweepTick](../raw/postgres-12/src/backend/storage/buffer/freelist.c#L113)).
- PostgreSQL 14: Differs on the built-in default. The GUC is `PGC_POSTMASTER`, so a change needs a restart, and its boot value in `guc.c` is 1024 blocks (8 MB at 8 kB), not 16384; the docs say the default is typically 128 MB because `initdb` chooses it ([guc.c#shared_buffers](../raw/postgres-14/src/backend/utils/misc/guc.c#L2339-L2347), [config.sgml#guc-shared-buffers](../raw/postgres-14/doc/src/sgml/config.sgml#L1621-L1623)). The value lives in `NBuffers` ([globals.c:135](../raw/postgres-14/src/backend/utils/init/globals.c#L135)).
- PostgreSQL 18: Holds. Still `postmaster` context (restart), default 16384 blocks ([guc_tables.c#shared_buffers](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2377-L2387), [globals.c:142](../raw/postgres-18/src/backend/utils/init/globals.c#L142), [config.sgml#guc-shared-buffers](../raw/postgres-18/doc/src/sgml/config.sgml#L1750-L1757)).
- PostgreSQL 19: Holds. The unit is blocks, the default is 16384, the context is `PGC_POSTMASTER` (restart), the variable is `NBuffers`, and the clock hand still wraps modulo `NBuffers` ([guc_parameters.dat#shared_buffers](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2711-L2718), [globals.c:144](../raw/postgres-19/src/backend/utils/init/globals.c#L144), [freelist.c:337-338](../raw/postgres-19/src/backend/storage/buffer/freelist.c#L337-L338), [config.sgml#guc-shared-buffers](../raw/postgres-19/doc/src/sgml/config.sgml#L1799-L1840)).

Related: [Buffer manager](#buffer-manager), [Clock sweep](#clock-sweep), [Huge pages](#huge-pages)

### shared_preload_libraries

**Aliases:** preload library, `process_shared_preload_libraries`, `_PG_init` at postmaster start. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`shared_preload_libraries` lists shared libraries that the postmaster loads at server start, before any backend exists. If a listed library is missing, the server fails to start ([config.sgml#guc-shared-preload-libraries](../raw/postgres-17/doc/src/sgml/config.sgml#L10412-L10421)). Libraries that allocate shared memory, reserve LWLocks or start background workers must be loaded this way ([config.sgml#guc-shared-preload-libraries](../raw/postgres-17/doc/src/sgml/config.sgml#L10423-L10429)). In source, `PostmasterMain` calls `process_shared_preload_libraries()`, which loads each library while `process_shared_preload_libraries_in_progress` is true, then calls `process_shmem_requests()` so the libraries' `shmem_request_hook` can reserve space ([postmaster.c:916-938](../raw/postgres-17/src/backend/postmaster/postmaster.c#L916-L938), [miscinit.c#process_shared_preload_libraries](../raw/postgres-17/src/backend/utils/init/miscinit.c#L1894-L1907), [miscinit.c#process_shmem_requests](../raw/postgres-17/src/backend/utils/init/miscinit.c#L1922-L1932)). The GUC has context `postmaster`, so changing it needs a restart ([guc_tables.c#shared_preload_libraries](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4257-L4266)).

**Version notes:**
- PostgreSQL 12: Holds; context `postmaster` (restart), defined in `guc.c` ([guc.c#shared_preload_libraries](../raw/postgres-12/src/backend/utils/misc/guc.c#L3767-L3776), [miscinit.c#process_shared_preload_libraries](../raw/postgres-12/src/backend/utils/init/miscinit.c#L1583-L1590)). Differs: a library reserves shared memory by calling `RequestAddinShmemSpace()` directly from `_PG_init`, which works only during preload ([ipci.c#RequestAddinShmemSpace](../raw/postgres-12/src/backend/storage/ipc/ipci.c#L58-L70)). The only shared-memory hook 12 declares is `shmem_startup_hook` in `ipc.h`; there is no `shmem_request_hook` ([ipc.h:22](../raw/postgres-12/src/include/storage/ipc.h#L22), [ipc.h:77](../raw/postgres-12/src/include/storage/ipc.h#L77)).
- PostgreSQL 14: Holds; context `postmaster` (restart) ([guc.c#shared_preload_libraries](../raw/postgres-14/src/backend/utils/misc/guc.c#L4149-L4158), [miscinit.c#process_shared_preload_libraries](../raw/postgres-14/src/backend/utils/init/miscinit.c#L1716-L1723)). Like 12, it uses `RequestAddinShmemSpace()` from `_PG_init` rather than a request hook ([ipci.c#RequestAddinShmemSpace](../raw/postgres-14/src/backend/storage/ipc/ipci.c#L59-L71)).
- PostgreSQL 18: Holds; context `postmaster` (restart) ([guc_tables.c#shared_preload_libraries](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L4486-L4495), [miscinit.c#process_shared_preload_libraries](../raw/postgres-18/src/backend/utils/init/miscinit.c#L1903-L1910), [postmaster.c:933](../raw/postgres-18/src/backend/postmaster/postmaster.c#L933)).
- PostgreSQL 19: Holds; `PGC_POSTMASTER` (restart), now defined in `guc_parameters.dat` ([guc_parameters.dat#shared_preload_libraries](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2746-L2751), [miscinit.c#process_shared_preload_libraries](../raw/postgres-19/src/backend/utils/init/miscinit.c#L1853-L1860), [postmaster.c:937](../raw/postgres-19/src/backend/postmaster/postmaster.c#L937)).

Related: [Extension](#extension), [Hook](#hook), [Background worker](#background-worker), [Postmaster](#postmaster), [GUC context](#guc-context)

### Shared-memory statistics

**Aliases:** shared memory stats, `pgstat_shmem.c`, `PgStatShared_*`, stats collector (PostgreSQL 12 and 14), `pg_stat` directory. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

This entry is about where the cumulative statistics counters live. In PostgreSQL 17 they live in shared memory. Fixed-numbered stats, such as checkpointer counters, use plain shared memory. Per-object stats use a `dshash` table in dynamic shared memory ([pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L17-L27), [pgstat_shmem.c](../raw/postgres-17/src/backend/utils/activity/pgstat_shmem.c#L1-L10)). Each process first adds to local pending counters and flushes them to shared memory later, which keeps contention low ([pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L44-L50)). The startup process loads the counters from `pg_stat/pgstat.stat`, and the checkpointer writes them at a clean shutdown ([pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L10-L15), [pgstat.h:28](../raw/postgres-17/src/include/pgstat.h#L28)). The `stats_fetch_consistency` GUC controls whether a reader sees a snapshot or live values ([pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L55-L59)).

**Version notes:**
- PostgreSQL 12: Differs. Counters live in a separate **statistics collector** process. Backends send it messages through `pgstat_send()`, and it writes stats files that backends read ([pgstat.c header](../raw/postgres-12/src/backend/postmaster/pgstat.c#L1-L17), [pgstat.c#pgstat_send](../raw/postgres-12/src/backend/postmaster/pgstat.c#L4334-L4341), [pgstat.c#PgstatCollectorMain](../raw/postgres-12/src/backend/postmaster/pgstat.c#L4418-L4428), [pgstat.h:29-30](../raw/postgres-12/src/include/pgstat.h#L29-L30)).
- PostgreSQL 14: Differs in the same way as 12: collector process, message sends, `pg_stat/global.stat` ([pgstat.c#pgstat_send](../raw/postgres-14/src/backend/postmaster/pgstat.c#L2951-L2958), [pgstat.c#PgstatCollectorMain](../raw/postgres-14/src/backend/postmaster/pgstat.c#L3167-L3177), [pgstat.h:28-29](../raw/postgres-14/src/include/pgstat.h#L28-L29)).
- PostgreSQL 18: Holds. The checkpointer writes a kind's stats only if that kind allows it. `pgstat_register_kind()` lets extensions add their own stats kinds ([pgstat.c header](../raw/postgres-18/src/backend/utils/activity/pgstat.c#L14-L25), [pgstat.c#pgstat_register_kind](../raw/postgres-18/src/backend/utils/activity/pgstat.c#L1465)).
- PostgreSQL 19: Holds, as in 18 ([pgstat.c header](../raw/postgres-19/src/backend/utils/activity/pgstat.c#L14-L25), [pgstat.c#pgstat_register_kind](../raw/postgres-19/src/backend/utils/activity/pgstat.c#L1508), [pgstat.h:36](../raw/postgres-19/src/include/pgstat.h#L36)).

Related: [Cumulative statistics](#cumulative-statistics), [Dynamic shared memory](#dynamic-shared-memory), [Checkpoint](#checkpoint)

### ShareUpdateExclusiveLock

**Aliases:** `SHARE UPDATE EXCLUSIVE` lock mode, lock mode 4. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A table lock mode that conflicts with itself and with the stronger modes, but not with ordinary reads and writes ([lockdefs.h#ShareUpdateExclusiveLock](../raw/postgres-17/src/include/storage/lockdefs.h#L39-L40), [lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L77-L80)). It protects a table against concurrent schema changes and `VACUUM` runs. Plain `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY`, `CREATE STATISTICS`, `COMMENT ON`, `REINDEX CONCURRENTLY` and some `ALTER INDEX` and `ALTER TABLE` forms take it ([mvcc.sgml#SHARE UPDATE EXCLUSIVE](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L977-L1000)). `vacuum_rel()` chooses it for every VACUUM except `VACUUM FULL` ([vacuum.c:2054-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2054-L2055)).

**Version notes:**
- PostgreSQL 12: Holds. It is mode 4, conflicts with itself and stronger modes, and plain VACUUM uses it ([lockdefs.h#ShareUpdateExclusiveLock](../raw/postgres-12/src/include/storage/lockdefs.h#L39-L40), [lock.c#LockConflicts](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81), [vacuum.c:1680-1681](../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1681)). The v12 docs list VACUUM, ANALYZE, CIC, `REINDEX CONCURRENTLY`, `CREATE STATISTICS` and some ALTER forms, but not `COMMENT ON` ([mvcc.sgml#SHARE UPDATE EXCLUSIVE](../raw/postgres-12/doc/src/sgml/mvcc.sgml#L912-L935)).
- PostgreSQL 14: Holds. Mode 4 conflicts with itself and stronger modes, is taken by plain VACUUM, ANALYZE, CIC, `CREATE STATISTICS`, `COMMENT ON` and `REINDEX CONCURRENTLY`, and `vacuum_rel()` picks it for non-FULL VACUUM ([lockdefs.h#ShareUpdateExclusiveLock](../raw/postgres-14/src/include/storage/lockdefs.h#L39), [lock.c#LockConflicts](../raw/postgres-14/src/backend/storage/lmgr/lock.c#L78-L81), [mvcc.sgml#SHARE UPDATE EXCLUSIVE](../raw/postgres-14/doc/src/sgml/mvcc.sgml#L939-L956), [vacuum.c:1919-1920](../raw/postgres-14/src/backend/commands/vacuum.c#L1919-L1920)).
- PostgreSQL 18: Holds ([lockdefs.h#ShareUpdateExclusiveLock](../raw/postgres-18/src/include/storage/lockdefs.h#L39-L40), [mvcc.sgml#SHARE UPDATE EXCLUSIVE](../raw/postgres-18/doc/src/sgml/mvcc.sgml#L977-L1000), [vacuum.c:2092-2093](../raw/postgres-18/src/backend/commands/vacuum.c#L2092-L2093)).
- PostgreSQL 19: Holds. Mode 4 still conflicts with itself and stronger modes, the documented takers are unchanged, and `vacuum_rel()` still picks it for every non-FULL VACUUM ([lockdefs.h#ShareUpdateExclusiveLock](../raw/postgres-19/src/include/storage/lockdefs.h#L39-L40), [lock.c#LockConflicts](../raw/postgres-19/src/backend/storage/lmgr/lock.c#L81-L84), [mvcc.sgml#SHARE UPDATE EXCLUSIVE](../raw/postgres-19/doc/src/sgml/mvcc.sgml#L991-L1010), [vacuum.c:2084-2085](../raw/postgres-19/src/backend/commands/vacuum.c#L2084-L2085)). `REPACK (CONCURRENTLY)` also holds it for most of its run ([repack.c:12-21](../raw/postgres-19/src/backend/commands/repack.c#L12-L21), [repack.c#RepackLockLevel](../raw/postgres-19/src/backend/commands/repack.c#L456-L463)).

Related: [Heavyweight lock](#heavyweight-lock), [AccessExclusiveLock](#accessexclusivelock), [CONCURRENTLY](#concurrently), [VACUUM](#vacuum)

### Simple index deletion

**Aliases:** LP_DEAD deletion, "known dead" index entries. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Simple index deletion removes index entries that are already marked `LP_DEAD`. An earlier scan sets that flag once it finds the row dead to every transaction ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L511-L534)). In B-tree, it is the first step `_bt_delete_or_dedup_one_page()` tries before a leaf page split. It can also delete nearby entries that turn out to be removable at no extra cost ([nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2703-L2731)). Hash and GiST have their own `LP_DEAD` removal routines ([hashinsert.c#_hash_vacuum_one_page](../raw/postgres-17/src/backend/access/hash/hashinsert.c#L364-L370), [gist.c#gistprunepage](../raw/postgres-17/src/backend/access/gist/gist.c#L1666-L1671)). Unlike bottom-up deletion, it only acts on entries that are already known dead ([nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L582-L585)).

**Version notes:**
- PostgreSQL 12: Differs. `LP_DEAD` marking and removal exist, and removal happens when a split looms ([nbtree/README](../raw/postgres-12/src/backend/access/nbtree/README#L400-L419)). In 12 it is `_bt_vacuum_one_page()`, which only removes items already marked dead and does not look for extra deletable neighbors ([nbtinsert.c#_bt_vacuum_one_page](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2245-L2251)). Hash and GiST have `_hash_vacuum_one_page` and `gistprunepage` ([hashinsert.c#_hash_vacuum_one_page](../raw/postgres-12/src/backend/access/hash/hashinsert.c#L338), [gist.c#gistprunepage](../raw/postgres-12/src/backend/access/gist/gist.c#L1644)).
- PostgreSQL 14: Holds. The README describes deleting `LP_DEAD` entries and nearby free-to-check entries, and `_bt_delete_or_dedup_one_page()` tries `_bt_simpledel_pass()` first; hash and GiST have their own routines ([nbtree/README](../raw/postgres-14/src/backend/access/nbtree/README#L475-L510), [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-14/src/backend/access/nbtree/nbtinsert.c#L2712-L2728), [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-14/src/backend/access/hash/hashinsert.c#L338), [gist.c#gistprunepage](../raw/postgres-14/src/backend/access/gist/gist.c#L1669)).
- PostgreSQL 18: Holds ([nbtree/README](../raw/postgres-18/src/backend/access/nbtree/README#L511-L534), [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-18/src/backend/access/nbtree/nbtinsert.c#L2703-L2731), [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-18/src/backend/access/hash/hashinsert.c#L364-L370), [gist.c#gistprunepage](../raw/postgres-18/src/backend/access/gist/gist.c#L1670-L1675)).
- PostgreSQL 19: Holds. It still removes `LP_DEAD`-marked entries first before a leaf split, may take nearby removable entries too, and hash and GiST still have their own routines ([nbtree/README](../raw/postgres-19/src/backend/access/nbtree/README#L510-L534), [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-19/src/backend/access/nbtree/nbtinsert.c#L2740-L2775), [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-19/src/backend/access/hash/hashinsert.c#L368), [gist.c#gistprunepage](../raw/postgres-19/src/backend/access/gist/gist.c#L1675), [nbtree/README](../raw/postgres-19/src/backend/access/nbtree/README#L582-L584)).

Related: [Bottom-up index deletion](#bottom-up-index-deletion), [Line pointer](#line-pointer), [Page split](#page-split), [VACUUM](#vacuum)

### SLRU

**Aliases:** simple LRU, SLRU buffers, `slru.c`, `pg_xact`, `pg_subtrans`, `pg_multixact`, `pg_stat_slru`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

An SLRU is a small, dedicated page cache for transaction metadata that is indexed by transaction ID, such as commit status, subtransaction parents, commit timestamps and MultiXact data ([slru.c](../raw/postgres-17/src/backend/access/transam/slru.c#L1-L10)). It is separate from the main buffer pool and uses a plain least-recently-used scheme. It never evicts the latest page ([slru.c](../raw/postgres-17/src/backend/access/transam/slru.c#L17-L23)). In PostgreSQL 17 each SLRU's buffers are split into banks, and each bank has its own control LWLock ([slru.c](../raw/postgres-17/src/backend/access/transam/slru.c#L17-L31)). Buffer counts are GUCs such as `multixact_offset_buffers`, `multixact_member_buffers` and `transaction_buffers`. All have context `postmaster`, so changing them needs a restart ([guc_tables.c#multixact_member_buffers](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2316-L2336), [guc_tables.c#transaction_buffers](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2371-L2380)). The `pg_stat_slru` view reports hits, reads and writes per SLRU ([system_views.sql#pg_stat_slru](../raw/postgres-17/src/backend/catalog/system_views.sql#L919)).

**Version notes:**
- PostgreSQL 12: Differs. The SLRU has one control lock and no banks, and the MultiXact buffer counts are compiled-in constants of 8 and 16 pages, not GUCs ([slru.c](../raw/postgres-12/src/backend/access/transam/slru.c#L3-L22), [multixact.h:32-33](../raw/postgres-12/src/include/access/multixact.h#L32-L33), [multixact.c:1829-1835](../raw/postgres-12/src/backend/access/transam/multixact.c#L1829-L1835)). 12 keeps no SLRU statistics: the stats collector's message types include no SLRU message, so there is no `pg_stat_slru` view ([pgstat.h#StatMsgType](../raw/postgres-12/src/include/pgstat.h#L48-L69)).
- PostgreSQL 14: Differs like 12: one control lock and fixed constants of 8 and 16 pages. `pg_stat_slru` exists ([slru.c](../raw/postgres-14/src/backend/access/transam/slru.c#L3-L10), [multixact.h:33-34](../raw/postgres-14/src/include/access/multixact.h#L33-L34), [system_views.sql#pg_stat_slru](../raw/postgres-14/src/backend/catalog/system_views.sql#L866)).
- PostgreSQL 18: Holds; banked buffers and the same `postmaster`-context GUCs ([slru.c](../raw/postgres-18/src/backend/access/transam/slru.c#L17-L23), [guc_tables.c#multixact_member_buffers](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2444-L2464)).
- PostgreSQL 19: Holds; the GUCs are defined in `guc_parameters.dat`, still `PGC_POSTMASTER` ([slru.c](../raw/postgres-19/src/backend/access/transam/slru.c#L17-L23), [guc_parameters.dat#multixact_member_buffers](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2256-L2274)).

Related: [MultiXact](#multixact), [Transaction ID](#transaction-id), [Buffer manager](#buffer-manager), [LWLock](#lwlock)

### Snapshot

**Aliases:** MVCC snapshot, `SnapshotData`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A record of which transactions count as finished for one statement or transaction. It decides which row versions that statement sees ([snapshot.h#SNAPSHOT_MVCC](../raw/postgres-17/src/include/utils/snapshot.h#L37-L50)). An MVCC snapshot holds `xmin` (every older xid is settled), `xmax` (every xid at or above it is invisible), and the in-progress list `xip` ([snapshot.h#SnapshotData](../raw/postgres-17/src/include/utils/snapshot.h#L142-L166)). `GetSnapshotData()` builds it from the proc array ([procarray.c#GetSnapshotData](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2177)). Other snapshot types exist, such as `SNAPSHOT_ANY` and `SNAPSHOT_NON_VACUUMABLE`, which obey different rules ([snapshot.h](../raw/postgres-17/src/include/utils/snapshot.h#L66-L69)). A snapshot's `xmin` is a visibility bound, not the per-row `t_xmin` stamp ([snapshot.h#SnapshotData](../raw/postgres-17/src/include/utils/snapshot.h#L156-L157), [htup_details.h#HeapTupleFields](../raw/postgres-17/src/include/access/htup_details.h#L122-L132)).

**Version notes:**
- PostgreSQL 12: Holds. `SnapshotData` has `xmin`, `xmax` and `xip`, `GetSnapshotData()` builds it, and `SNAPSHOT_ANY` and `SNAPSHOT_NON_VACUUMABLE` exist ([snapshot.h#SNAPSHOT_MVCC](../raw/postgres-12/src/include/utils/snapshot.h#L50), [snapshot.h#SnapshotData](../raw/postgres-12/src/include/utils/snapshot.h#L142-L168), [procarray.c#GetSnapshotData](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1505), [snapshot.h](../raw/postgres-12/src/include/utils/snapshot.h#L69-L118)).
- PostgreSQL 14: Holds. `SnapshotData` has `xmin`, `xmax` and `xip`, `GetSnapshotData()` builds it from the proc array, and `SNAPSHOT_ANY` and `SNAPSHOT_NON_VACUUMABLE` exist ([snapshot.h#SNAPSHOT_MVCC](../raw/postgres-14/src/include/utils/snapshot.h#L50), [snapshot.h#SnapshotData](../raw/postgres-14/src/include/utils/snapshot.h#L142-L168), [procarray.c#GetSnapshotData](../raw/postgres-14/src/backend/storage/ipc/procarray.c#L2251), [snapshot.h](../raw/postgres-14/src/include/utils/snapshot.h#L69), [snapshot.h](../raw/postgres-14/src/include/utils/snapshot.h#L118)).
- PostgreSQL 18: Holds ([snapshot.h#SNAPSHOT_MVCC](../raw/postgres-18/src/include/utils/snapshot.h#L33-L46), [snapshot.h#SnapshotData](../raw/postgres-18/src/include/utils/snapshot.h#L138-L162), [procarray.c#GetSnapshotData](../raw/postgres-18/src/backend/storage/ipc/procarray.c#L2175)).
- PostgreSQL 19: Holds. `SNAPSHOT_MVCC`, `SNAPSHOT_ANY` and `SNAPSHOT_NON_VACUUMABLE` exist, `SnapshotData` still holds `xmin`, `xmax` and `xip`, and `GetSnapshotData()` still builds it ([snapshot.h#SNAPSHOT_MVCC](../raw/postgres-19/src/include/utils/snapshot.h#L36-L46), [snapshot.h#SnapshotData](../raw/postgres-19/src/include/utils/snapshot.h#L138-L164), [snapshot.h](../raw/postgres-19/src/include/utils/snapshot.h#L65), [snapshot.h](../raw/postgres-19/src/include/utils/snapshot.h#L114), [procarray.c#GetSnapshotData](../raw/postgres-19/src/backend/storage/ipc/procarray.c#L2114)).

Related: [MVCC](#mvcc), [xmin and xmax](#xmin-and-xmax), [xmin horizon](#xmin-horizon), [Transaction ID](#transaction-id)

### SP-GiST

**Aliases:** space-partitioned GiST. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

SP-GiST is a framework for index structures that are not balanced and that split the key space into regions, such as quadtrees, k-d trees and radix trees ([spgist/README](../raw/postgres-17/src/backend/access/spgist/README#L1-L12)). An SP-GiST tree has inner tuples, which hold labeled child pointers, and leaf tuples. The two kinds are kept on separate pages, and branches may have different depths ([spgist/README](../raw/postgres-17/src/backend/access/spgist/README#L14-L24)). `spghandler()` marks it as supporting ordered searches, but not unique or multicolumn indexes. Block 0 is its metapage ([spgutils.c#spghandler](../raw/postgres-17/src/backend/access/spgist/spgutils.c#L44-L55), [spgist_private.h:47](../raw/postgres-17/src/include/access/spgist_private.h#L47)).

**Version notes:**
- PostgreSQL 12: Holds. It is an unbalanced space-partitioning framework with inner and leaf tuples on separate pages; `spghandler()` sets ordered search on, unique and multicolumn off; block 0 is the metapage ([spgist/README](../raw/postgres-12/src/backend/access/spgist/README#L1-L25), [spgutils.c#spghandler](../raw/postgres-12/src/backend/access/spgist/spgutils.c#L40-L50), [spgist_private.h:26](../raw/postgres-12/src/include/access/spgist_private.h#L26)).
- PostgreSQL 14: Holds. The README describes the same unbalanced, space-partitioning trees with inner and leaf tuples that are never mixed on one level, and `spghandler()` supports ordered searches but not unique or multicolumn indexes; block 0 is the metapage ([spgist/README](../raw/postgres-14/src/backend/access/spgist/README#L1-L27), [spgutils.c#spghandler](../raw/postgres-14/src/backend/access/spgist/spgutils.c#L52-L55), [spgist_private.h:47](../raw/postgres-14/src/include/access/spgist_private.h#L47)).
- PostgreSQL 18: Holds. `spghandler()` still sets ordered search but not uniqueness or multicolumn ([spgist/README](../raw/postgres-18/src/backend/access/spgist/README#L1-L12), [spgutils.c#spghandler](../raw/postgres-18/src/backend/access/spgist/spgutils.c#L44-L58), [spgist_private.h:47](../raw/postgres-18/src/include/access/spgist_private.h#L47)).
- PostgreSQL 19: Holds. Inner and leaf tuples on separate pages, unbalanced trees, ordered search without unique or multicolumn support, and a block-0 metapage are unchanged ([spgist/README](../raw/postgres-19/src/backend/access/spgist/README#L1-L24), [spgutils.c#spghandler](../raw/postgres-19/src/backend/access/spgist/spgutils.c#L44-L58), [spgist_private.h:47](../raw/postgres-19/src/include/access/spgist_private.h#L47)).

Related: [GiST](#gist), [Access method](#access-method), [Metapage](#metapage)

### SPI

**Aliases:** Server Programming Interface. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

SPI lets C code running inside the server run SQL through the parser, planner, and executor [spi.sgml](../raw/postgres-17/doc/src/sgml/spi.sgml#L10-L18). Most procedural languages, including PL/pgSQL, run their SQL through it [spi.sgml](../raw/postgres-17/doc/src/sgml/spi.sgml#L20-L27). Code calls `SPI_connect()`, runs statements with functions such as `SPI_execute()`, and finishes with `SPI_finish()` [spi.c#SPI_connect](../raw/postgres-17/src/backend/executor/spi.c#L94) [spi.c#SPI_execute](../raw/postgres-17/src/backend/executor/spi.c#L594-L614) [spi.c#SPI_finish](../raw/postgres-17/src/backend/executor/spi.c#L182).

**Version notes:**
- PostgreSQL 12: Holds. SPI gives C code access to the parser, planner and executor, and `SPI_connect()`, `SPI_execute()` and `SPI_finish()` exist ([spi.sgml](../raw/postgres-12/doc/src/sgml/spi.sgml#L10-L27), [spi.c#SPI_connect](../raw/postgres-12/src/backend/executor/spi.c#L89), [spi.c#SPI_execute](../raw/postgres-12/src/backend/executor/spi.c#L496), [spi.c#SPI_finish](../raw/postgres-12/src/backend/executor/spi.c#L176)).
- PostgreSQL 14: Holds. SPI gives C code access to the parser, planner and executor, procedural languages use it, and the entry points are `SPI_connect()`, `SPI_execute()` and `SPI_finish()` ([spi.sgml](../raw/postgres-14/doc/src/sgml/spi.sgml#L10-L27), [spi.c#SPI_connect](../raw/postgres-14/src/backend/executor/spi.c#L95), [spi.c#SPI_execute](../raw/postgres-14/src/backend/executor/spi.c#L607), [spi.c#SPI_finish](../raw/postgres-14/src/backend/executor/spi.c#L183)).
- PostgreSQL 18: Holds ([spi.sgml](../raw/postgres-18/doc/src/sgml/spi.sgml#L10-L18), [spi.c#SPI_connect](../raw/postgres-18/src/backend/executor/spi.c#L94), [spi.c#SPI_execute](../raw/postgres-18/src/backend/executor/spi.c#L594-L614), [spi.c#SPI_finish](../raw/postgres-18/src/backend/executor/spi.c#L182)).
- PostgreSQL 19: Holds. SPI still gives C code the parser, planner and executor, and `SPI_connect()`, `SPI_execute()` and `SPI_finish()` are unchanged ([spi.sgml](../raw/postgres-19/doc/src/sgml/spi.sgml#L8-L28), [spi.c#SPI_connect](../raw/postgres-19/src/backend/executor/spi.c#L95), [spi.c#SPI_execute](../raw/postgres-19/src/backend/executor/spi.c#L597), [spi.c#SPI_finish](../raw/postgres-19/src/backend/executor/spi.c#L183)).

Related: [PL/pgSQL](#plpgsql), [Executor](#executor), [Planner](#planner)

### SQL function inlining

**Aliases:** function inlining, `inline_function`, `inline_set_returning_function`, `inline_function_in_from`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

SQL function inlining is the planner replacing a call to a simple SQL-language function with the function's body. The planner can then fold constants, use indexes and prune partitions on the expanded expression ([clauses.c#inline_function](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L4529-L4535)). `simplify_function()` tries a support function first and then `inline_function()` during constant folding ([clauses.c#simplify_function](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L4141-L4157)). Inlining is refused for non-SQL functions, `SECURITY DEFINER` functions, set-returning or `RECORD`-returning functions, functions with `SET` clauses, and functions the caller cannot execute. It is also refused when a function-entry hook wants the call ([clauses.c#inline_function](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L4587-L4610)). Inlining must not change volatility or strictness ([clauses.c#inline_function](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L4548-L4549)). Set-returning SQL functions in `FROM` go through a separate path, `inline_set_returning_function()`, called from `prepjointree.c` ([clauses.c#inline_set_returning_function](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L5058-L5075), [prepjointree.c:906](../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L906)).

**Version notes:**
- PostgreSQL 12: Holds; same two paths ([clauses.c#inline_function](../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4434-L4440), [clauses.c:4040](../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4040), [prepjointree.c:633](../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L633)).
- PostgreSQL 14: Holds ([clauses.c#inline_function](../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4422-L4428), [prepjointree.c:660](../raw/postgres-14/src/backend/optimizer/prep/prepjointree.c#L660)).
- PostgreSQL 18: Holds ([clauses.c#inline_function](../raw/postgres-18/src/backend/optimizer/util/clauses.c#L4611-L4617), [prepjointree.c:933](../raw/postgres-18/src/backend/optimizer/prep/prepjointree.c#L933)).
- PostgreSQL 19: Differs for `FROM`-clause calls. `inline_function_in_from()` replaces `inline_set_returning_function()`. It first asks the function's support function through `SupportRequestInlineInFrom`, then falls back to built-in SQL inlining in `inline_sql_function_in_from()` ([clauses.c#inline_function_in_from](../raw/postgres-19/src/backend/optimizer/util/clauses.c#L5825-L5844), [clauses.c#inline_function_in_from](../raw/postgres-19/src/backend/optimizer/util/clauses.c#L5936-L5973), [supportnodes.h#SupportRequestInlineInFrom](../raw/postgres-19/src/include/nodes/supportnodes.h#L108-L125), [prepjointree.c:1192](../raw/postgres-19/src/backend/optimizer/prep/prepjointree.c#L1192)). Expression inlining keeps the same refusal list ([clauses.c#inline_function](../raw/postgres-19/src/backend/optimizer/util/clauses.c#L5361-L5368)).

Related: [Planner](#planner), [Function volatility](#function-volatility), [SECURITY DEFINER](#security-definer), [Partition pruning](#partition-pruning)

### statement_timeout and lock_timeout

**Aliases:** `statement_timeout`, `lock_timeout`, `StatementTimeout`, `LockTimeout`, session timeouts. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`statement_timeout` and `lock_timeout` are two settings that cancel a statement that runs or waits too long. `statement_timeout` aborts any statement that takes longer than the limit, measured from when the command reaches the server until it completes. In PostgreSQL 17, each statement of a multi-statement simple-query message gets its own timer ([config.sgml#guc-statement-timeout](../raw/postgres-17/doc/src/sgml/config.sgml#L9462-L9488)). `lock_timeout` aborts a statement only while it waits for a lock. The limit applies separately to each lock acquisition, and setting it at or above a nonzero `statement_timeout` is pointless ([config.sgml#guc-lock-timeout](../raw/postgres-17/doc/src/sgml/config.sgml#L9534-L9565)). Before a backend sleeps on a [heavyweight lock](#heavyweight-lock), it arms the lock timeout alongside the [deadlock](#deadlock) timer ([proc.c:1295-1307](../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1295-L1307)). When a timeout fires, the backend raises "canceling statement due to lock timeout" or "... due to statement timeout" ([postgres.c:3396-3408](../raw/postgres-17/src/backend/tcop/postgres.c#L3396-L3408)). Both default to 0 (off) and have context `user`, so a session can set them with `SET` or `SET LOCAL` without a reload ([guc_tables.c:2611-2630](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2630)). The docs advise against setting `lock_timeout` in `postgresql.conf`, because it would affect all sessions ([config.sgml#guc-lock-timeout](../raw/postgres-17/doc/src/sgml/config.sgml#L9564-L9568)).

**Version notes:**
- PostgreSQL 12: Both exist with context `user` ([guc.c:2378](../raw/postgres-12/src/backend/utils/misc/guc.c#L2378), [guc.c:2389](../raw/postgres-12/src/backend/utils/misc/guc.c#L2389)). The docs do not yet state that each statement in a simple-query message gets its own timer ([config.sgml#guc-statement-timeout](../raw/postgres-12/doc/src/sgml/config.sgml#L7669-L7691)).
- PostgreSQL 14: Holds ([guc.c:2574](../raw/postgres-14/src/backend/utils/misc/guc.c#L2574), [guc.c:2585](../raw/postgres-14/src/backend/utils/misc/guc.c#L2585)).
- PostgreSQL 18: Holds ([guc_tables.c:2740](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2740), [guc_tables.c:2751](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2751)).
- PostgreSQL 19: Holds; both are declared in `guc_parameters.dat` with context `user` ([guc_parameters.dat:1609](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1609), [guc_parameters.dat:2892](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2892)).

Related: [Lock queue](#lock-queue), [Deadlock](#deadlock), [Heavyweight lock](#heavyweight-lock), [GUC context](#guc-context)

### Statistics

**Aliases:** planner statistics, `pg_statistic`, `pg_stats`, `ANALYZE` statistics. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Planner statistics are per-column summaries of a table's data that `ANALYZE` samples. They include the null fraction, the average width, the number of distinct values, most-common values, and histograms [pg_statistic.h#pg_statistic](../raw/postgres-17/src/include/catalog/pg_statistic.h#L29-L69). `do_analyze_rel()` computes them and `update_attstats()` stores them in `pg_statistic`. The readable `pg_stats` view exposes them [analyze.c#do_analyze_rel](../raw/postgres-17/src/backend/commands/analyze.c#L272-L283) [analyze.c:593-601](../raw/postgres-17/src/backend/commands/analyze.c#L593-L601) [system_views.sql#pg_stats](../raw/postgres-17/src/backend/catalog/system_views.sql#L189-L200). The planner reads them through `examine_variable()` when it estimates selectivity [selfuncs.c#examine_variable](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L5037-L5040). Do not confuse these with cumulative statistics, the `pg_stat_*` activity counters.

**Version notes:**
- PostgreSQL 12: Holds. `pg_statistic` holds per-column stats, `do_analyze_rel()` and `update_attstats()` compute and store them, `pg_stats` exposes them, and `examine_variable()` reads them ([pg_statistic.h#pg_statistic](../raw/postgres-12/src/include/catalog/pg_statistic.h#L29), [analyze.c#do_analyze_rel](../raw/postgres-12/src/backend/commands/analyze.c#L295), [analyze.c:565-572](../raw/postgres-12/src/backend/commands/analyze.c#L565-L572), [system_views.sql#pg_stats](../raw/postgres-12/src/backend/catalog/system_views.sql#L189), [selfuncs.c#examine_variable](../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L4436)).
- PostgreSQL 14: Holds. `pg_statistic` holds the same per-column summaries, `do_analyze_rel()` computes them and `update_attstats()` stores them, `pg_stats` exposes them, and the planner reads them through `examine_variable()` ([pg_statistic.h#pg_statistic](../raw/postgres-14/src/include/catalog/pg_statistic.h#L29-L69), [analyze.c#do_analyze_rel](../raw/postgres-14/src/backend/commands/analyze.c#L290), [analyze.c:604-611](../raw/postgres-14/src/backend/commands/analyze.c#L604-L611), [system_views.sql#pg_stats](../raw/postgres-14/src/backend/catalog/system_views.sql#L189), [selfuncs.c#examine_variable](../raw/postgres-14/src/backend/utils/adt/selfuncs.c#L4969)).
- PostgreSQL 18: Holds ([pg_statistic.h#pg_statistic](../raw/postgres-18/src/include/catalog/pg_statistic.h#L29-L69), [analyze.c#do_analyze_rel](../raw/postgres-18/src/backend/commands/analyze.c#L270-L281), [system_views.sql#pg_stats](../raw/postgres-18/src/backend/catalog/system_views.sql#L185-L196), [selfuncs.c#examine_variable](../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L5312-L5315)). Related change: `pg_upgrade` now carries most of these statistics across an upgrade; see pg_upgrade ([pgupgrade.sgml:833-840](../raw/postgres-18/doc/src/sgml/ref/pgupgrade.sgml#L833-L840)).
- PostgreSQL 19: Holds. `ANALYZE` still samples per-column stats into `pg_statistic` via `do_analyze_rel()` and `update_attstats()`, `pg_stats` still exposes them, and the planner still reads them through `examine_variable()` ([pg_statistic.h#pg_statistic](../raw/postgres-19/src/include/catalog/pg_statistic.h#L31-L127), [analyze.c#do_analyze_rel](../raw/postgres-19/src/backend/commands/analyze.c#L306), [analyze.c:612](../raw/postgres-19/src/backend/commands/analyze.c#L612), [system_views.sql#pg_stats](../raw/postgres-19/src/backend/catalog/system_views.sql#L190), [selfuncs.c#examine_variable](../raw/postgres-19/src/backend/utils/adt/selfuncs.c#L5652)). v19 can also load statistics without `ANALYZE`, through `pg_restore_relation_stats()` and `pg_restore_attribute_stats()` ([relation_stats.c#pg_restore_relation_stats](../raw/postgres-19/src/backend/statistics/relation_stats.c#L248), [attribute_stats.c#pg_restore_attribute_stats](../raw/postgres-19/src/backend/statistics/attribute_stats.c#L699)).

Related: [Selectivity](#selectivity), [reltuples and relpages](#reltuples-and-relpages), [Cumulative statistics](#cumulative-statistics), [Catalog](#catalog)

### Storage manager

**Aliases:** smgr, `smgr.c`, `md.c`, magnetic disk manager, segment file, `RELSEG_SIZE`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The storage manager is the layer below the buffer manager that turns "block N of fork F of relation R" into file operations. All relation file operations dispatch through `smgr.c` ([smgr.c](../raw/postgres-17/src/backend/storage/smgr/smgr.c#L1-L12)). `smgr.c` calls through the `f_smgr` function table, which has one implementation: `md.c`, the historical "magnetic disk" manager ([smgr.c#smgrsw](../raw/postgres-17/src/backend/storage/smgr/smgr.c#L107-L120), [md.c](../raw/postgres-17/src/backend/storage/smgr/md.c#L1-L12)). `md.c` splits each relation fork into segment files of at most `RELSEG_SIZE` blocks ([md.c](../raw/postgres-17/src/backend/storage/smgr/md.c#L42-L56)). `RELSEG_SIZE` is set at build time from `--with-segsize`, which defaults to 1 GB, or from `--with-segsize-blocks` ([configure.ac](../raw/postgres-17/configure.ac#L294-L318)). In PostgreSQL 17 the read callback is vectored: `smgr_readv` reads several blocks in one call ([smgr.c#f_smgr](../raw/postgres-17/src/backend/storage/smgr/smgr.c#L91-L93)).

**Version notes:**
- PostgreSQL 12: Differs. The table has a single-block `smgr_read` callback ([smgr.c#f_smgr](../raw/postgres-12/src/backend/storage/smgr/smgr.c#L40-L64), [md.c](../raw/postgres-12/src/backend/storage/smgr/md.c#L44-L50)). `RELSEG_SIZE` is set in `configure.in` ([configure.in:288-293](../raw/postgres-12/configure.in#L288-L293)).
- PostgreSQL 14: Differs like 12: a single-block `smgr_read` ([smgr.c#f_smgr](../raw/postgres-14/src/backend/storage/smgr/smgr.c#L40-L65), [md.c](../raw/postgres-14/src/backend/storage/smgr/md.c#L45-L51), [configure.ac:292-297](../raw/postgres-14/configure.ac#L292-L297)).
- PostgreSQL 18: Differs. The table adds `smgr_maxcombine` and `smgr_startreadv`, which starts a read as an asynchronous I/O handle ([smgr.c#f_smgr](../raw/postgres-18/src/backend/storage/smgr/smgr.c#L105-L113), [md.c](../raw/postgres-18/src/backend/storage/smgr/md.c#L44-L50)).
- PostgreSQL 19: Same as 18 ([smgr.c#f_smgr](../raw/postgres-19/src/backend/storage/smgr/smgr.c#L105-L113), [md.c](../raw/postgres-19/src/backend/storage/smgr/md.c#L46-L52)).

Related: [Buffer manager](#buffer-manager), [Fork](#fork), [Relfilenumber](#relfilenumber), [Block](#block), [Asynchronous I/O](#asynchronous-io)

### Storage parameter

**Aliases:** reloption, reloptions, relation option, `pg_class.reloptions`, `WITH (...)`, `ALTER TABLE ... SET (...)`, `StdRdOptions`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A storage parameter is a per-table or per-index setting given with `WITH (...)` on `CREATE` or with `ALTER ... SET (...)`, such as [fillfactor](#fillfactor) or `autovacuum_vacuum_scale_factor`. Tables accept `toast.`-prefixed copies that apply to their [TOAST](#toast) table. Index parameters are documented with `CREATE INDEX` ([ref/create_table.sgml#sql-createtable-storage-parameters](../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1432-L1455)). The values are stored as a `text[]` in `pg_class.reloptions` ([pg_class.h:137](../raw/postgres-17/src/include/catalog/pg_class.h#L137)). `reloptions.c` validates them against built-in option tables. `default_reloptions()` parses table options into `StdRdOptions`, and `index_reloptions()` hands index options to the access method's own `amoptions` parser ([reloptions.c header](../raw/postgres-17/src/backend/access/common/reloptions.c#L1-L4), [reloptions.c#default_reloptions](../raw/postgres-17/src/backend/access/common/reloptions.c#L1847), [reloptions.c#index_reloptions](../raw/postgres-17/src/backend/access/common/reloptions.c#L2055-L2071), [rel.h#StdRdOptions](../raw/postgres-17/src/include/utils/rel.h#L336)). The [relcache](#relcache) keeps the parsed result in `rd_options` ([rel.h:171-175](../raw/postgres-17/src/include/utils/rel.h#L171-L175), [relcache.c#RelationParseRelOptions](../raw/postgres-17/src/backend/utils/cache/relcache.c#L464)). Each option declares the lock `ALTER ... SET` must take. The default is `AccessExclusiveLock`, and autovacuum options and `fillfactor` need only `ShareUpdateExclusiveLock` ([reloptions.c:53-75](../raw/postgres-17/src/backend/access/common/reloptions.c#L53-L75)).

**Version notes:**
- PostgreSQL 12: Holds, with the same `toast.` prefix ([ref/create_table.sgml#sql-createtable-storage-parameters](../raw/postgres-12/doc/src/sgml/ref/create_table.sgml#L1291-L1314), [pg_class.h:134](../raw/postgres-12/src/include/catalog/pg_class.h#L134), [reloptions.c#default_reloptions](../raw/postgres-12/src/backend/access/common/reloptions.c#L1376)).
- PostgreSQL 14: Adds `build_reloptions()` as the common parser ([reloptions.c#build_reloptions](../raw/postgres-14/src/backend/access/common/reloptions.c#L1913), [rel.h#StdRdOptions](../raw/postgres-14/src/include/utils/rel.h#L318)).
- PostgreSQL 18: Holds ([ref/create_table.sgml#sql-createtable-storage-parameters](../raw/postgres-18/doc/src/sgml/ref/create_table.sgml#L1558), [reloptions.c#default_reloptions](../raw/postgres-18/src/backend/access/common/reloptions.c#L1869), [pg_class.h:140](../raw/postgres-18/src/include/catalog/pg_class.h#L140)).
- PostgreSQL 19: Holds ([ref/create_table.sgml#sql-createtable-storage-parameters](../raw/postgres-19/doc/src/sgml/ref/create_table.sgml#L1578), [reloptions.c#default_reloptions](../raw/postgres-19/src/backend/access/common/reloptions.c#L1975), [pg_class.h:142](../raw/postgres-19/src/include/catalog/pg_class.h#L142)).

Related: [Fillfactor](#fillfactor), [pg_class](#pg_class), [Relcache](#relcache), [INDEX_CLEANUP](#index_cleanup), [GiST build method](#gist-build-method)

### SubPlan

**Aliases:** `SubPlan`, InitPlan, `initPlan`, correlated subquery, `nodeSubplan.c`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A `SubPlan` is the executable expression that replaces a sub-`SELECT` after the planner has planned it. It points to a separate plan tree in `PlannedStmt.subplans` ([primnodes.h#SubPlan](../raw/postgres-17/src/include/nodes/primnodes.h#L1021-L1028)). An **InitPlan** is a sub-select with no parameters from the outer query. The planner makes it one when `parParam` is empty, for example an uncorrelated `EXISTS` or scalar subquery ([subselect.c#build_subplan](../raw/postgres-17/src/backend/optimizer/plan/subselect.c#L385-L407)). An InitPlan hangs off the plan node's `initPlan` list and runs at most once, lazily, the first time its output parameter is needed ([plannodes.h#Plan](../raw/postgres-17/src/include/nodes/plannodes.h#L156-L157), [nodeSubplan.c#ExecSetParamPlan](../raw/postgres-17/src/backend/executor/nodeSubplan.c#L1051-L1059)). A correlated subplan is re-run through `ExecSubPlan()` with new parameter values ([nodeSubplan.c#ExecSubPlan](../raw/postgres-17/src/backend/executor/nodeSubplan.c#L55-L62), [primnodes.h#SubPlan](../raw/postgres-17/src/include/nodes/primnodes.h#L1044-L1052)). The planner names each one "InitPlan N" or "SubPlan N" in `plan_name`, a field kept for EXPLAIN and debugging ([subselect.c:563-564](../raw/postgres-17/src/backend/optimizer/plan/subselect.c#L563-L564), [primnodes.h#SubPlan](../raw/postgres-17/src/include/nodes/primnodes.h#L1070-L1071)).

**Version notes:**
- PostgreSQL 12: Holds ([primnodes.h#SubPlan](../raw/postgres-12/src/include/nodes/primnodes.h#L649-L720), [subselect.c:393](../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L393), [nodeSubplan.c#ExecSetParamPlan](../raw/postgres-12/src/backend/executor/nodeSubplan.c#L1051)).
- PostgreSQL 14: Holds ([primnodes.h#SubPlan](../raw/postgres-14/src/include/nodes/primnodes.h#L696-L767), [subselect.c:396](../raw/postgres-14/src/backend/optimizer/plan/subselect.c#L396)).
- PostgreSQL 18: Holds; the `SubPlan` struct matches 17 ([primnodes.h#SubPlan](../raw/postgres-18/src/include/nodes/primnodes.h#L1041-L1114), [subselect.c:397](../raw/postgres-18/src/backend/optimizer/plan/subselect.c#L397)).
- PostgreSQL 19: Differs in details. `SubPlan` gains an `isInitPlan` flag, and subplan names come from `choose_plan_name()` instead of the fixed "InitPlan N"/"SubPlan N" format ([primnodes.h#SubPlan](../raw/postgres-19/src/include/nodes/primnodes.h#L1095-L1097), [subselect.c:224-226](../raw/postgres-19/src/backend/optimizer/plan/subselect.c#L224-L226), [planner.c#choose_plan_name](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L9197-L9203)). The InitPlan rule for empty `parParam` is unchanged ([subselect.c:410](../raw/postgres-19/src/backend/optimizer/plan/subselect.c#L410)).

Related: [Planner](#planner), [Executor](#executor), [PlannedStmt](#plannedstmt), [EXPLAIN](#explain)

### Subscription

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A subscription is the downstream side of logical replication. It names a connection to a publisher and the publications to receive ([logical-replication.sgml#logical-replication-subscription](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L176-L183)). The shared catalog `pg_subscription` stores the connection string, the publisher-side slot name, the publication list, and `suborigin` ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-17/src/include/catalog/pg_subscription.h#L100-L117)). Its `origin` option chooses between receiving every change (`any`, the default) and only changes that carry no origin (`none`) ([create_subscription.sgml](../raw/postgres-17/doc/src/sgml/ref/create_subscription.sgml#L400-L411)). `CREATE SUBSCRIPTION` runs `CreateSubscription` ([subscriptioncmds.c#CreateSubscription](../raw/postgres-17/src/backend/commands/subscriptioncmds.c#L605)).

**Version notes:**
- PostgreSQL 12: Differs. `pg_subscription` stores the connection string, slot name and publication list, but has no `suborigin`, and 12 has no `origin` option ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-12/src/include/catalog/pg_subscription.h#L53-L64), [logical-replication.sgml#logical-replication-subscription](../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L163-L170)). `CreateSubscription` runs `CREATE SUBSCRIPTION` ([subscriptioncmds.c#CreateSubscription](../raw/postgres-12/src/backend/commands/subscriptioncmds.c#L315)).
- PostgreSQL 14: Differs. The subscription names a connection and publications, and `pg_subscription` stores the connection string, slot name and publication list ([logical-replication.sgml#logical-replication-subscription](../raw/postgres-14/doc/src/sgml/logical-replication.sgml#L163-L175), [pg_subscription.h#FormData_pg_subscription](../raw/postgres-14/src/include/catalog/pg_subscription.h#L42-L73), [subscriptioncmds.c#CreateSubscription](../raw/postgres-14/src/backend/commands/subscriptioncmds.c#L353)). 14 has no `suborigin` column and no `origin` option ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-14/src/include/catalog/pg_subscription.h#L42-L73)).
- PostgreSQL 18: Holds ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-18/src/include/catalog/pg_subscription.h#L80-L97), [create_subscription.sgml](../raw/postgres-18/doc/src/sgml/ref/create_subscription.sgml#L407-L418), [subscriptioncmds.c#CreateSubscription](../raw/postgres-18/src/backend/commands/subscriptioncmds.c#L565)).
- PostgreSQL 19: Holds. `pg_subscription` still stores the connection string, slot name, publications and `suborigin`, `origin` still defaults to `any`, and `CREATE SUBSCRIPTION` still runs `CreateSubscription()` ([logical-replication.sgml#logical-replication-subscription](../raw/postgres-19/doc/src/sgml/logical-replication.sgml#L207-L215), [pg_subscription.h#FormData_pg_subscription](../raw/postgres-19/src/include/catalog/pg_subscription.h#L45-L117), [create_subscription.sgml](../raw/postgres-19/doc/src/sgml/ref/create_subscription.sgml#L433-L443), [subscriptioncmds.c#CreateSubscription](../raw/postgres-19/src/backend/commands/subscriptioncmds.c#L647)). In v19, a subscription can connect through a foreign server (`subserver`) instead of a connection string, and `origin` has no effect for sequences ([pg_subscription.h:95-100](../raw/postgres-19/src/include/catalog/pg_subscription.h#L95-L100), [create_subscription.sgml](../raw/postgres-19/doc/src/sgml/ref/create_subscription.sgml#L443)).

Related: [Publication](#publication), [Apply worker](#apply-worker), [Replication origin](#replication-origin), [Replication slot](#replication-slot)

### Synchronous replication

**Aliases:** sync rep, `synchronous_commit`, `synchronous_standby_names`, `syncrep.c`, `SyncRepWaitForLSN`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Synchronous replication makes a committing transaction wait until chosen standbys confirm its commit record. All the waiting happens on the primary, and the standbys do not know about it ([syncrep.c](../raw/postgres-17/src/backend/replication/syncrep.c#L5-L20)). `synchronous_standby_names` names the standbys and picks `FIRST` (priority) or `ANY` (quorum) ([syncrep.c](../raw/postgres-17/src/backend/replication/syncrep.c#L31-L45)). `synchronous_commit` picks what to wait for: nothing (`off`), local flush, remote write, remote flush, or remote apply ([xact.h#SyncCommitLevel](../raw/postgres-17/src/include/access/xact.h#L68-L77)). Backends wait in `SyncRepWaitForLSN()` ([syncrep.c:148](../raw/postgres-17/src/backend/replication/syncrep.c#L148)). `synchronous_commit` has context `user` (session or transaction scope). `synchronous_standby_names` has context `sighup` (reload) ([guc_tables.c#synchronous_commit](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4925-L4933), [guc_tables.c#synchronous_standby_names](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4571-L4580)).

**Version notes:**
- PostgreSQL 12: Holds; the same five levels and GUC contexts ([xact.h#SyncCommitLevel](../raw/postgres-12/src/include/access/xact.h#L68-L76), [syncrep.c#SyncRepWaitForLSN](../raw/postgres-12/src/backend/replication/syncrep.c#L146), [guc.c#synchronous_commit](../raw/postgres-12/src/backend/utils/misc/guc.c#L4353-L4361), [guc.c#synchronous_standby_names](../raw/postgres-12/src/backend/utils/misc/guc.c#L4086-L4095)).
- PostgreSQL 14: Holds ([syncrep.c](../raw/postgres-14/src/backend/replication/syncrep.c#L7-L8), [guc.c#synchronous_commit](../raw/postgres-14/src/backend/utils/misc/guc.c#L4800-L4808), [guc.c#synchronous_standby_names](../raw/postgres-14/src/backend/utils/misc/guc.c#L4478-L4487)).
- PostgreSQL 18: Holds ([syncrep.c](../raw/postgres-18/src/backend/replication/syncrep.c#L7-L8), [guc_tables.c#synchronous_commit](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5188-L5196), [guc_tables.c#synchronous_standby_names](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L4800-L4809)).
- PostgreSQL 19: Holds; GUCs are in `guc_parameters.dat` with the same contexts ([syncrep.c](../raw/postgres-19/src/backend/replication/syncrep.c#L7-L8), [guc_parameters.dat#synchronous_commit](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2958-L2964), [guc_parameters.dat#synchronous_standby_names](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2966-L2973)).

Related: [WAL sender](#wal-sender), [WAL receiver](#wal-receiver), [LSN](#lsn), [GUC context](#guc-context)

### Syscache

**Aliases:** system cache, catcache, `SearchSysCache1`, `MAKE_SYSCACHE`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The syscache is a per-backend cache of individual catalog rows, looked up by key. The parser, planner, and executor use it for fast catalog access ([syscache.c:13-17](../raw/postgres-17/src/backend/utils/cache/syscache.c#L13-L17)). Each cache is declared next to its catalog with `MAKE_SYSCACHE`, for example `RELOID` on `pg_class_oid_index`. `genbki.pl` generates the cache ID list from those declarations ([pg_class.h:159-160](../raw/postgres-17/src/include/catalog/pg_class.h#L159-L160), [genbki.h:123-127](../raw/postgres-17/src/include/catalog/genbki.h#L123-L127), [genbki.pl:449-453](../raw/postgres-17/src/backend/catalog/genbki.pl#L449-L453)). Callers look up a row with `SearchSysCache1` and friends and must release it with `ReleaseSysCache` ([syscache.c#SearchSysCache](../raw/postgres-17/src/backend/utils/cache/syscache.c#L200-L230)).

**Version notes:**
- PostgreSQL 12: Differs in declaration. The syscache caches catalog rows by key, and callers use `SearchSysCache1` and `ReleaseSysCache` ([syscache.c:13-17](../raw/postgres-12/src/backend/utils/cache/syscache.c#L13-L17), [syscache.c#SearchSysCache1](../raw/postgres-12/src/backend/utils/cache/syscache.c#L1124), [syscache.c#ReleaseSysCache](../raw/postgres-12/src/backend/utils/cache/syscache.c#L1172)). 12 has no `MAKE_SYSCACHE`: caches are listed by hand in the `cacheinfo[]` array and the `SysCacheIdentifier` enum in `syscache.h` ([syscache.c:89](../raw/postgres-12/src/backend/utils/cache/syscache.c#L89), [syscache.c:124](../raw/postgres-12/src/backend/utils/cache/syscache.c#L124), [syscache.h:84](../raw/postgres-12/src/include/utils/syscache.h#L84)).
- PostgreSQL 14: Differs in declaration. The syscache gives fast row lookups, and callers use `SearchSysCache*` and `ReleaseSysCache` ([syscache.c:13-17](../raw/postgres-14/src/backend/utils/cache/syscache.c#L13-L17), [syscache.c#SearchSysCache](../raw/postgres-14/src/backend/utils/cache/syscache.c#L1130), [syscache.c#ReleaseSysCache](../raw/postgres-14/src/backend/utils/cache/syscache.c#L1191)). But 14 has no `MAKE_SYSCACHE`: cache IDs such as `RELOID` are a hand-written enum in `syscache.h`, and their definitions are the hand-written `cacheinfo[]` array in `syscache.c` ([syscache.h#SysCacheIdentifier](../raw/postgres-14/src/include/utils/syscache.h#L32-L85), [syscache.c#cacheinfo](../raw/postgres-14/src/backend/utils/cache/syscache.c#L127)).
- PostgreSQL 18: Holds ([syscache.c:13-17](../raw/postgres-18/src/backend/utils/cache/syscache.c#L13-L17), [pg_class.h:162-163](../raw/postgres-18/src/include/catalog/pg_class.h#L162-L163), [genbki.h:123-127](../raw/postgres-18/src/include/catalog/genbki.h#L123-L127), [syscache.c#SearchSysCache](../raw/postgres-18/src/backend/utils/cache/syscache.c#L200-L230)).
- PostgreSQL 19: Holds. Caches are still declared with `MAKE_SYSCACHE`, for example `RELOID`, `genbki.pl` still writes `syscache_ids.h`, and callers still pair `SearchSysCache1()` with `ReleaseSysCache()` ([syscache.c:13-17](../raw/postgres-19/src/backend/utils/cache/syscache.c#L13-L17), [pg_class.h:166](../raw/postgres-19/src/include/catalog/pg_class.h#L166), [genbki.h:146](../raw/postgres-19/src/include/catalog/genbki.h#L146), [genbki.pl:449-451](../raw/postgres-19/src/backend/catalog/genbki.pl#L449-L451), [syscache.c#SearchSysCache](../raw/postgres-19/src/backend/utils/cache/syscache.c#L209-L221), [syscache.c#ReleaseSysCache](../raw/postgres-19/src/backend/utils/cache/syscache.c#L265)). Cache IDs now have the type `SysCacheIdentifier` ([syscache.c#SearchSysCache](../raw/postgres-19/src/backend/utils/cache/syscache.c#L209)).

Related: [Relcache](#relcache), [Catalog](#catalog), [BKI](#bki)

### Table rewrite

**Aliases:** heap rewrite, `make_new_heap`, `finish_heap_swap`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A table rewrite copies every live row into a brand-new set of files and then swaps those files in for the old ones. `CLUSTER`, `VACUUM FULL` and some forms of `ALTER TABLE` use it. `make_new_heap()` creates the transient table, the caller fills it, and `finish_heap_swap()` swaps the files and rebuilds all indexes ([cluster.c#make_new_heap](../raw/postgres-17/src/backend/commands/cluster.c#L676-L690), [cluster.c#finish_heap_swap](../raw/postgres-17/src/backend/commands/cluster.c#L1433-L1440)). In `ALTER TABLE`, `ATRewriteTables()` fires the `table_rewrite` event trigger and then runs that same sequence ([tablecmds.c#ATRewriteTables](../raw/postgres-17/src/backend/commands/tablecmds.c#L5834-L5878)). Four reasons can cause an `ALTER TABLE` rewrite: persistence change, a default value, a column rewrite, or an access method change ([event_trigger.h#AT_REWRITE](../raw/postgres-17/src/include/commands/event_trigger.h#L36-L43)). The swap exchanges the two relations' relfilenumbers, so the table keeps its OID but gets new files ([cluster.c#swap_relation_files](../raw/postgres-17/src/backend/commands/cluster.c#L1035-L1040)).

**Version notes:**
- PostgreSQL 12: Differs. `make_new_heap()` and `finish_heap_swap()` do the rewrite, and `ATRewriteTables()` fires `table_rewrite` first ([cluster.c#make_new_heap](../raw/postgres-12/src/backend/commands/cluster.c#L641), [cluster.c#finish_heap_swap](../raw/postgres-12/src/backend/commands/cluster.c#L1341), [tablecmds.c#ATRewriteTables](../raw/postgres-12/src/backend/commands/tablecmds.c#L4635-L4672)). 12 has only three `AT_REWRITE` reasons; there is no access-method-change reason ([event_trigger.h#AT_REWRITE](../raw/postgres-12/src/include/commands/event_trigger.h#L31-L33)). The swap exchanges relfilenodes ([cluster.c#swap_relation_files](../raw/postgres-12/src/backend/commands/cluster.c#L1002)).
- PostgreSQL 14: Holds, with three rewrite reasons. `make_new_heap()` and `finish_heap_swap()` drive rewrites, and `ATRewriteTables()` fires `table_rewrite` before them ([cluster.c#make_new_heap](../raw/postgres-14/src/backend/commands/cluster.c#L644), [cluster.c#finish_heap_swap](../raw/postgres-14/src/backend/commands/cluster.c#L1363), [tablecmds.c#ATRewriteTables](../raw/postgres-14/src/backend/commands/tablecmds.c#L5525-L5562)). 14 defines only persistence change, default value and column rewrite as `AT_REWRITE` reasons; there is no access-method-change reason ([event_trigger.h#AT_REWRITE](../raw/postgres-14/src/include/commands/event_trigger.h#L38-L40)). The swap exchanges relfilenodes ([cluster.c#swap_relation_files](../raw/postgres-14/src/backend/commands/cluster.c#L1005)).
- PostgreSQL 18: Holds ([cluster.c#make_new_heap](../raw/postgres-18/src/backend/commands/cluster.c#L693-L707), [cluster.c#finish_heap_swap](../raw/postgres-18/src/backend/commands/cluster.c#L1440-L1447), [tablecmds.c#ATRewriteTables](../raw/postgres-18/src/backend/commands/tablecmds.c#L5970-L6014), [event_trigger.h#AT_REWRITE](../raw/postgres-18/src/include/commands/event_trigger.h#L36-L43)).
- PostgreSQL 19: Holds, but the functions moved to `repack.c`. `make_new_heap()` builds the transient table and `finish_heap_swap()` swaps files and rebuilds indexes ([repack.c#make_new_heap](../raw/postgres-19/src/backend/commands/repack.c#L1122-L1133), [repack.c#finish_heap_swap](../raw/postgres-19/src/backend/commands/repack.c#L1883-L1889)). `ATRewriteTables()` still fires `table_rewrite` and then runs the same sequence, the four `AT_REWRITE_*` reasons are unchanged, and the swap still keeps the OID ([tablecmds.c#ATRewriteTables](../raw/postgres-19/src/backend/commands/tablecmds.c#L5881), [tablecmds.c:6005](../raw/postgres-19/src/backend/commands/tablecmds.c#L6005), [tablecmds.c:6025](../raw/postgres-19/src/backend/commands/tablecmds.c#L6025), [tablecmds.c:6043](../raw/postgres-19/src/backend/commands/tablecmds.c#L6043), [event_trigger.h#AT_REWRITE](../raw/postgres-19/src/include/commands/event_trigger.h#L40-L43), [repack.c#swap_relation_files](../raw/postgres-19/src/backend/commands/repack.c#L1480-L1486)). `REPACK` joins `CLUSTER` and `VACUUM FULL` as a rewrite command ([repack.c:1-10](../raw/postgres-19/src/backend/commands/repack.c#L1-L10)).

Related: [CLUSTER](#cluster), [Event trigger](#event-trigger), [Relfilenumber](#relfilenumber), [VACUUM FULL](#vacuum-full)

### Table synchronization

**Aliases:** tablesync, tablesync worker, initial table copy, `copy_data`, `tablesync.c`, `pg_subscription_rel` states. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Table synchronization is the first copy of a table's existing rows when a logical replication subscription starts, followed by catch-up to the apply worker's position. Each table is copied by its own tablesync worker. The copy can run in parallel and does not hold back the xid and LSN of the whole database ([tablesync.c](../raw/postgres-17/src/backend/replication/logical/tablesync.c#L11-L27)). Progress is a per-table state in `pg_subscription_rel`: `INIT`, `DATASYNC`, `FINISHEDCOPY`, `SYNCDONE` and `READY` are stored, and `SYNCWAIT` and `CATCHUP` are memory-only handshakes with the apply worker ([pg_subscription_rel.h](../raw/postgres-17/src/include/catalog/pg_subscription_rel.h#L58-L74), [tablesync.c](../raw/postgres-17/src/backend/replication/logical/tablesync.c#L29-L50)). The subscription option `copy_data`, default true, decides whether the copy happens at all ([create_subscription.sgml#copy_data](../raw/postgres-17/doc/src/sgml/ref/create_subscription.sgml#L249-L254)). In PostgreSQL 17 the worker's entry point is `TablesyncWorkerMain` ([tablesync.c#TablesyncWorkerMain](../raw/postgres-17/src/backend/replication/logical/tablesync.c#L1734-L1745)).

**Version notes:**
- PostgreSQL 12: Differs. There is no `FINISHEDCOPY` state ([pg_subscription_rel.h](../raw/postgres-12/src/include/catalog/pg_subscription_rel.h#L45-L54)). A tablesync worker runs `ApplyWorkerMain` and branches into `LogicalRepSyncTableStart()` when `am_tablesync_worker()` is true ([worker.c](../raw/postgres-12/src/backend/replication/logical/worker.c#L1690-L1695)).
- PostgreSQL 14: Differs from 17 only in the entry point. `FINISHEDCOPY` exists, and the worker still starts from `ApplyWorkerMain` ([pg_subscription_rel.h:64](../raw/postgres-14/src/include/catalog/pg_subscription_rel.h#L64), [worker.c](../raw/postgres-14/src/backend/replication/logical/worker.c#L3197-L3202)).
- PostgreSQL 18: Holds ([tablesync.c#TablesyncWorkerMain](../raw/postgres-18/src/backend/replication/logical/tablesync.c#L1748), [pg_subscription_rel.h](../raw/postgres-18/src/include/catalog/pg_subscription_rel.h#L62-L69)).
- PostgreSQL 19: Holds. The entry point is renamed `TableSyncWorkerMain`, and a separate `sequencesync.c` worker syncs sequences through the same catalog ([tablesync.c#TableSyncWorkerMain](../raw/postgres-19/src/backend/replication/logical/tablesync.c#L1608-L1618), [sequencesync.c](../raw/postgres-19/src/backend/replication/logical/sequencesync.c#L9-L20)).

Related: [Logical replication](#logical-replication), [Apply worker](#apply-worker), [Subscription](#subscription), [Replication slot](#replication-slot)

### Tablespace

**Aliases:** `pg_tablespace`, `reltablespace`, `pg_default`, `pg_global`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A tablespace is a named file-system location where PostgreSQL stores relation files ([manage-ag.sgml#manage-ag-tablespaces](../raw/postgres-17/doc/src/sgml/manage-ag.sgml#L380-L392)). The shared catalog `pg_tablespace` lists them. Two exist from `initdb`: `pg_default` (OID 1663) and `pg_global` (OID 1664) ([pg_tablespace.h:29](../raw/postgres-17/src/include/catalog/pg_tablespace.h#L29), [pg_tablespace.dat](../raw/postgres-17/src/include/catalog/pg_tablespace.dat#L14-L18)). `pg_class.reltablespace` records a relation's tablespace, and 0 means the database's default ([pg_class.h:59-60](../raw/postgres-17/src/include/catalog/pg_class.h#L59-L60)). The planner reads per-tablespace `random_page_cost` and `seq_page_cost` through `get_tablespace_page_costs()`, and falls back to the GUCs when a tablespace sets none ([spccache.c#get_tablespace_page_costs](../raw/postgres-17/src/backend/utils/cache/spccache.c#L173-L200)).

**Version notes:**
- PostgreSQL 12: Holds ([pg_tablespace.dat](../raw/postgres-12/src/include/catalog/pg_tablespace.dat#L15-L20), [pg_class.h:57](../raw/postgres-12/src/include/catalog/pg_class.h#L57), [spccache.c:182](../raw/postgres-12/src/backend/utils/cache/spccache.c#L182)).
- PostgreSQL 14: Holds ([pg_tablespace.dat](../raw/postgres-14/src/include/catalog/pg_tablespace.dat#L15-L18), [pg_class.h:60](../raw/postgres-14/src/include/catalog/pg_class.h#L60), [spccache.c:181](../raw/postgres-14/src/backend/utils/cache/spccache.c#L181)).
- PostgreSQL 18: Holds ([pg_tablespace.dat](../raw/postgres-18/src/include/catalog/pg_tablespace.dat#L15-L18), [pg_class.h:60](../raw/postgres-18/src/include/catalog/pg_class.h#L60), [spccache.c:182](../raw/postgres-18/src/backend/utils/cache/spccache.c#L182)).
- PostgreSQL 19: Holds ([pg_tablespace.dat](../raw/postgres-19/src/include/catalog/pg_tablespace.dat#L15-L18), [pg_class.h:62](../raw/postgres-19/src/include/catalog/pg_class.h#L62), [spccache.c:183](../raw/postgres-19/src/backend/utils/cache/spccache.c#L183)).

Related: [pg_class](#pg_class), [Relfilenumber](#relfilenumber), [Cost](#cost), [OID](#oid)

### TAP test

**Aliases:** Perl TAP test, `prove` test. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A TAP test is a Perl script, usually `t/*.pl`, that the `prove` tool runs. Client-program tests under `src/bin` and many multi-node scenarios are written this way ([regress.sgml#regress-tap](../raw/postgres-17/doc/src/sgml/regress.sgml#L796-L820)). These tests run only when the build was configured with `--enable-tap-tests` ([regress.sgml#regress-tap](../raw/postgres-17/doc/src/sgml/regress.sgml#L827-L828)). Scripts drive throwaway servers through `PostgreSQL::Test::Cluster`, which runs `initdb`, starts and restarts nodes, and runs `psql` ([Cluster.pm](../raw/postgres-17/src/test/perl/PostgreSQL/Test/Cluster.pm#L8-L39)).

**Version notes:**
- PostgreSQL 12: Differs in module names. TAP tests are `t/*.pl` run by `prove` and need `--enable-tap-tests` ([regress.sgml#regress-tap](../raw/postgres-12/doc/src/sgml/regress.sgml#L757-L780), [regress.sgml:245-247](../raw/postgres-12/doc/src/sgml/regress.sgml#L245-L247)). 12 has no `PostgreSQL::Test::Cluster`; scripts use `PostgresNode.pm` and `TestLib.pm` ([PostgresNode.pm](../raw/postgres-12/src/test/perl/PostgresNode.pm#L1-L30)).
- PostgreSQL 14: Holds, with an older module name. TAP tests are `t/*.pl` scripts run by `prove` and need `--enable-tap-tests` ([regress.sgml#regress-tap](../raw/postgres-14/doc/src/sgml/regress.sgml#L776-L806)). In 14 most tests `use PostgresNode`; `PostgreSQL::Test::Cluster` exists only as a back-patched alias of `PostgresNode` ([PostgresNode.pm](../raw/postgres-14/src/test/perl/PostgresNode.pm#L8-L14), [Cluster.pm](../raw/postgres-14/src/test/perl/PostgreSQL/Test/Cluster.pm#L4-L17)).
- PostgreSQL 18: Holds ([regress.sgml#regress-tap](../raw/postgres-18/doc/src/sgml/regress.sgml#L830-L854), [Cluster.pm](../raw/postgres-18/src/test/perl/PostgreSQL/Test/Cluster.pm#L8-L39)).
- PostgreSQL 19: Holds. TAP tests are still `t/*.pl` scripts run by `prove`, still need `--enable-tap-tests`, and still drive servers through `PostgreSQL::Test::Cluster` ([regress.sgml#regress-tap](../raw/postgres-19/doc/src/sgml/regress.sgml#L930-L962), [Cluster.pm](../raw/postgres-19/src/test/perl/PostgreSQL/Test/Cluster.pm#L8-L12)).

Related: [Regression test](#regression-test), [Isolation test](#isolation-test), [Injection point](#injection-point)

### TID

**Aliases:** tuple identifier, item pointer, `ItemPointerData`, `ctid`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A TID is the physical address of a tuple: a block number plus a line-pointer number on that page. The struct is `ItemPointerData`, six bytes with `ip_blkid` and `ip_posid` ([itemptr.h#ItemPointerData](../raw/postgres-17/src/include/storage/itemptr.h#L20-L40)). Index entries store TIDs to point at heap rows. Each heap tuple header stores one in `t_ctid`, which points to the tuple itself or to its newer version after an update ([htup_details.h#t_ctid](../raw/postgres-17/src/include/access/htup_details.h#L86-L99)). SQL shows the TID as the `ctid` system column ([heap.c#ctid](../raw/postgres-17/src/backend/catalog/heap.c#L143-L147)). An update stores the new version at a new TID and links the old version to it, so a TID is not a stable row identity ([htup_details.h#t_ctid](../raw/postgres-17/src/include/access/htup_details.h#L86-L89)).

**Version notes:**
- PostgreSQL 12: Holds. `ItemPointerData` has `ip_blkid` and `ip_posid`, `t_ctid` points to the newer version, and `ctid` is the system column ([itemptr.h#ItemPointerData](../raw/postgres-12/src/include/storage/itemptr.h#L36-L45), [htup_details.h#t_ctid](../raw/postgres-12/src/include/access/htup_details.h#L85-L90), [heap.c#ctid](../raw/postgres-12/src/backend/catalog/heap.c#L153-L154)).
- PostgreSQL 14: Holds. `ItemPointerData` has `ip_blkid` and `ip_posid`, `t_ctid` points to the tuple or its newer version, and `ctid` is a system column ([itemptr.h#ItemPointerData](../raw/postgres-14/src/include/storage/itemptr.h#L36-L40), [htup_details.h#t_ctid](../raw/postgres-14/src/include/access/htup_details.h#L85-L92), [heap.c#ctid](../raw/postgres-14/src/backend/catalog/heap.c#L153-L154)).
- PostgreSQL 18: Holds ([itemptr.h#ItemPointerData](../raw/postgres-18/src/include/storage/itemptr.h#L20-L40), [htup_details.h#t_ctid](../raw/postgres-18/src/include/access/htup_details.h#L86-L99), [heap.c#ctid](../raw/postgres-18/src/backend/catalog/heap.c#L144-L148)).
- PostgreSQL 19: Holds. `ItemPointerData` is still a block id plus offset, `t_ctid` still points to itself or the newer version, and SQL still shows it as `ctid` ([itemptr.h#ItemPointerData](../raw/postgres-19/src/include/storage/itemptr.h#L17-L45), [htup_details.h#t_ctid](../raw/postgres-19/src/include/access/htup_details.h#L86-L100), [heap.c#ctid](../raw/postgres-19/src/backend/catalog/heap.c#L143-L147)).

Related: [Block](#block), [HOT](#hot), [Line pointer](#line-pointer), [Tuple](#tuple)

### Timeline

**Aliases:** timeline ID, TLI, `TimeLineID`, `.history` file, `timeline.c`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A timeline is a numbered branch of WAL history. The server starts a new one when an archive recovery or point-in-time recovery ends, so the new WAL cannot overwrite WAL from the old history. A normal restart or crash recovery keeps the same timeline ([xlogdefs.h#TimeLineID](../raw/postgres-17/src/include/access/xlogdefs.h#L51-L60), [backup.sgml#backup-timelines](../raw/postgres-17/doc/src/sgml/backup.sgml#L1409-L1418)). The timeline ID is the first 8 hex digits of every WAL segment file name ([xlog_internal.h#XLogFileName](../raw/postgres-17/src/include/access/xlog_internal.h#L164-L170)). Each timeline switch is recorded in a `<tli>.history` file that lists the parent timeline, the switch-point LSN and a reason. These files are archived with the WAL ([timeline.c](../raw/postgres-17/src/backend/access/transam/timeline.c#L1-L22)).

**Version notes:**
- PostgreSQL 12: Holds; `XLogFileName` is a macro rather than an inline function ([xlogdefs.h#TimeLineID](../raw/postgres-12/src/include/access/xlogdefs.h#L43-L52), [xlog_internal.h#XLogFileName](../raw/postgres-12/src/include/access/xlog_internal.h#L155-L158), [timeline.c](../raw/postgres-12/src/backend/access/transam/timeline.c#L4-L7)).
- PostgreSQL 14: Holds, also with the macro form ([xlogdefs.h#TimeLineID](../raw/postgres-14/src/include/access/xlogdefs.h#L51-L60), [xlog_internal.h#XLogFileName](../raw/postgres-14/src/include/access/xlog_internal.h#L165), [timeline.c](../raw/postgres-14/src/backend/access/transam/timeline.c#L4-L7)).
- PostgreSQL 18: Holds ([xlogdefs.h#TimeLineID](../raw/postgres-18/src/include/access/xlogdefs.h#L51-L60), [xlog_internal.h#XLogFileName](../raw/postgres-18/src/include/access/xlog_internal.h#L166), [timeline.c](../raw/postgres-18/src/backend/access/transam/timeline.c#L4-L7)).
- PostgreSQL 19: Holds ([xlogdefs.h#TimeLineID](../raw/postgres-19/src/include/access/xlogdefs.h#L54-L63), [xlog_internal.h#XLogFileName](../raw/postgres-19/src/include/access/xlog_internal.h#L165), [timeline.c](../raw/postgres-19/src/backend/access/transam/timeline.c#L4-L7)).

Related: [WAL](#wal), [LSN](#lsn), [Crash recovery](#crash-recovery), [Hot standby](#hot-standby)

### TOAST

**Aliases:** The Oversized-Attribute Storage Technique, TOAST table, `reltoastrelid`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

TOAST stores large column values outside the main row. It compresses them and splits them into chunks in a separate TOAST table that belongs to the main table ([glossary.sgml#glossary-toast](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1857-L1870)). `pg_class.reltoastrelid` links a table to its TOAST table ([pg_class.h:72](../raw/postgres-17/src/include/catalog/pg_class.h#L72)). On insert, `heap_prepare_insert()` calls `heap_toast_insert_or_update()` when a table or materialized-view tuple is longer than `TOAST_TUPLE_THRESHOLD` or already has external values ([heapam.c#heap_prepare_insert](../raw/postgres-17/src/backend/access/heap/heapam.c#L2348-L2363)). That threshold is the largest tuple size that lets four tuples fit on one heap page, so it scales with `BLCKSZ` ([heaptoast.h#TOAST_TUPLE_THRESHOLD](../raw/postgres-17/src/include/access/heaptoast.h#L38-L50)).

**Version notes:**
- PostgreSQL 12: Holds, with older names. `reltoastrelid` links the TOAST table, and `heap_prepare_insert()` calls `toast_insert_or_update()` (not `heap_toast_insert_or_update()`) above `TOAST_TUPLE_THRESHOLD` for tables and matviews ([pg_class.h:69](../raw/postgres-12/src/include/catalog/pg_class.h#L69), [heapam.c#heap_prepare_insert](../raw/postgres-12/src/backend/access/heap/heapam.c#L2049-L2088)). The threshold lives in `tuptoaster.h` and fits four tuples per page ([tuptoaster.h#TOAST_TUPLE_THRESHOLD](../raw/postgres-12/src/include/access/tuptoaster.h#L46-L55)).
- PostgreSQL 14: Holds. `reltoastrelid` links the TOAST table, `heap_prepare_insert()` calls `heap_toast_insert_or_update()` for tables and materialized views over `TOAST_TUPLE_THRESHOLD`, and the threshold targets four tuples per page ([glossary.sgml#glossary-toast](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1604-L1617), [pg_class.h:72](../raw/postgres-14/src/include/catalog/pg_class.h#L72), [heapam.c#heap_prepare_insert](../raw/postgres-14/src/backend/access/heap/heapam.c#L2319-L2327), [heaptoast.h#TOAST_TUPLE_THRESHOLD](../raw/postgres-14/src/include/access/heaptoast.h#L39-L48)).
- PostgreSQL 18: Holds ([glossary.sgml#glossary-toast](../raw/postgres-18/doc/src/sgml/glossary.sgml#L1906-L1919), [heapam.c#heap_prepare_insert](../raw/postgres-18/src/backend/access/heap/heapam.c#L2323-L2338), [heaptoast.h#TOAST_TUPLE_THRESHOLD](../raw/postgres-18/src/include/access/heaptoast.h#L38-L50)).
- PostgreSQL 19: Holds. `reltoastrelid` still links the TOAST table, `heap_prepare_insert()` still calls `heap_toast_insert_or_update()` above `TOAST_TUPLE_THRESHOLD`, and the threshold still fits four tuples per page ([glossary.sgml#glossary-toast](../raw/postgres-19/doc/src/sgml/glossary.sgml#L2004-L2017), [pg_class.h:77](../raw/postgres-19/src/include/catalog/pg_class.h#L77), [heapam.c#heap_prepare_insert](../raw/postgres-19/src/backend/access/heap/heapam.c#L2228-L2265), [heaptoast.h#TOAST_TUPLE_THRESHOLD](../raw/postgres-19/src/include/access/heaptoast.h#L38-L48)).

Related: [BLCKSZ](#blcksz), [Heap](#heap), [Tuple](#tuple)

### track_activity_query_size

**Aliases:** `pgstat_track_activity_query_size`, `st_activity_raw`, `BackendActivityBuffer`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`track_activity_query_size` sets how many bytes of each session's current query text the server keeps for `pg_stat_activity.query`. The default is 1024 ([config.sgml#guc-track-activity-query-size](../raw/postgres-17/doc/src/sgml/config.sgml#L8416-L8432)). The server reserves that many bytes per backend-status slot in shared memory at startup ([backend_status.c#BackendStatusShmemSize](../raw/postgres-17/src/backend/utils/activity/backend_status.c#L95-L97)). Longer text is cut to size − 1 bytes when stored. The cut ignores multibyte characters, and the reader fixes that on display ([backend_status.c:552](../raw/postgres-17/src/backend/utils/activity/backend_status.c#L552), [backend_status.h#PgBackendStatus](../raw/postgres-17/src/include/utils/backend_status.h#L148-L156)). The GUC has context `postmaster`, so a change needs a restart. It accepts 100 to 1048576 bytes ([guc_tables.c#track_activity_query_size](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3565-L3574)).

**Version notes:**
- PostgreSQL 12: Differs in the limit and location. The maximum is 102400 bytes, and the code lives in `postmaster/pgstat.c` and `pgstat.h`. Context is still `postmaster` (restart) ([guc.c#track_activity_query_size](../raw/postgres-12/src/backend/utils/misc/guc.c#L3164-L3173), [pgstat.c:2665-2666](../raw/postgres-12/src/backend/postmaster/pgstat.c#L2665-L2666), [pgstat.c:3169](../raw/postgres-12/src/backend/postmaster/pgstat.c#L3169), [pgstat.h:1086](../raw/postgres-12/src/include/pgstat.h#L1086)).
- PostgreSQL 14: Holds: maximum 1048576, context `postmaster` ([guc.c#track_activity_query_size](../raw/postgres-14/src/backend/utils/misc/guc.c#L3480-L3489), [backend_status.c:570](../raw/postgres-14/src/backend/utils/activity/backend_status.c#L570)).
- PostgreSQL 18: Holds ([guc_tables.c#track_activity_query_size](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3772-L3781), [backend_status.c:622](../raw/postgres-18/src/backend/utils/activity/backend_status.c#L622)).
- PostgreSQL 19: Holds; defined in `guc_parameters.dat` as `PGC_POSTMASTER`, 100 to 1048576. The buffer is now reserved through `ShmemRequestStruct` ([guc_parameters.dat#track_activity_query_size](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3190-L3197), [backend_status.c:109-112](../raw/postgres-19/src/backend/utils/activity/backend_status.c#L109-L112), [backend_status.c:587](../raw/postgres-19/src/backend/utils/activity/backend_status.c#L587)).

Related: [pg_stat_activity](#pg_stat_activity), [pg_stat_statements](#pg_stat_statements), [GUC context](#guc-context)

### Transaction ID

**Aliases:** xid, `TransactionId`, `FullTransactionId`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A 32-bit number a transaction receives the first time it needs one, typically when it first writes a row ([c.h:669](../raw/postgres-17/src/include/c.h#L669), [transam/README](../raw/postgres-17/src/backend/access/transam/README#L193-L195)). Values 0 to 2 are reserved: invalid, bootstrap and frozen. Normal xids start at 3 ([transam.h](../raw/postgres-17/src/include/access/transam.h#L31-L35)). Normal xids compare modulo 2^32, so "older" means within half the number space ([transam.c#TransactionIdPrecedes](../raw/postgres-17/src/backend/access/transam/transam.c#L276-L293)). `FullTransactionId` adds a 32-bit epoch, making a 64-bit value that never wraps ([transam.h#FullTransactionId](../raw/postgres-17/src/include/access/transam.h#L60-L68), [glossary.sgml#xid](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1890-L1905)).

**Version notes:**
- PostgreSQL 12: Holds. `TransactionId` is 32 bits, 0 to 2 are reserved and normal xids start at 3, comparison is modulo 2^32, and `FullTransactionId` exists as a 64-bit value ([c.h:507](../raw/postgres-12/src/include/c.h#L507), [transam.h](../raw/postgres-12/src/include/access/transam.h#L27-L34), [transam.c#TransactionIdPrecedes](../raw/postgres-12/src/backend/access/transam/transam.c#L300), [transam.h#FullTransactionId](../raw/postgres-12/src/include/access/transam.h#L47-L62)). XIDs are assigned on first write ([transam/README](../raw/postgres-12/src/backend/access/transam/README#L190-L195)).
- PostgreSQL 14: Holds. `TransactionId` is 32-bit, assigned on first write; 0 to 2 are reserved; normal xids compare modulo 2^32; and `FullTransactionId` is a 64-bit value ([c.h:632](../raw/postgres-14/src/include/c.h#L632), [transam/README](../raw/postgres-14/src/backend/access/transam/README#L193-L196), [transam.h](../raw/postgres-14/src/include/access/transam.h#L23-L34), [transam.c#TransactionIdPrecedes](../raw/postgres-14/src/backend/access/transam/transam.c#L305-L318), [transam.h#FullTransactionId](../raw/postgres-14/src/include/access/transam.h#L65-L68), [glossary.sgml#xid](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1637-L1659)).
- PostgreSQL 18: Holds ([c.h:638](../raw/postgres-18/src/include/c.h#L638), [transam.h](../raw/postgres-18/src/include/access/transam.h#L31-L35), [transam.c#TransactionIdPrecedes](../raw/postgres-18/src/backend/access/transam/transam.c#L276-L293), [transam.h#FullTransactionId](../raw/postgres-18/src/include/access/transam.h#L60-L68)).
- PostgreSQL 19: Holds. `TransactionId` is still 32-bit and assigned lazily, 0 to 2 are still reserved, normal XIDs still compare modulo 2^32, and `FullTransactionId` still adds an epoch ([c.h:743](../raw/postgres-19/src/include/c.h#L743), [transam/README](../raw/postgres-19/src/backend/access/transam/README#L193-L195), [transam.h](../raw/postgres-19/src/include/access/transam.h#L23-L33), [transam.h#FullTransactionId](../raw/postgres-19/src/include/access/transam.h#L65-L68), [glossary.sgml#glossary-xid](../raw/postgres-19/doc/src/sgml/glossary.sgml#L2037-L2060)). `TransactionIdPrecedes()` is now a `static inline` function in `transam.h`, not in `transam.c` ([transam.h#TransactionIdPrecedes](../raw/postgres-19/src/include/access/transam.h#L259-L275)).

Related: [xmin and xmax](#xmin-and-xmax), [Wraparound](#wraparound), [Freezing](#freezing), [MultiXact](#multixact)

### Truncation

**Aliases:** relation truncation, vacuum truncation, `lazy_truncate_heap`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

In this wiki, truncation means VACUUM cutting empty pages off the end of a table and giving that disk space back to the operating system. VACUUM tries it only when at least 1000 pages, or one-sixteenth of the table, at the end could be freed. It skips truncation when the wraparound failsafe is active ([vacuumlazy.c#should_attempt_truncation](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2510-L2544), [vacuumlazy.c#REL_TRUNCATE_MINIMUM](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L62-L72)). `lazy_truncate_heap()` needs `AccessExclusiveLock`. It never queues for it: it retries a conditional lock every 50 ms, and after 5 s it gives up truncating ([vacuumlazy.c#lazy_truncate_heap](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2572-L2600), [vacuumlazy.c#VACUUM_TRUNCATE_LOCK_TIMEOUT](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L82-L83)). `RelationTruncate()` does the physical cut and drops the buffers for removed blocks ([storage.c#RelationTruncate](../raw/postgres-17/src/backend/catalog/storage.c#L280-L289)). The `vacuum_truncate` storage option can turn it off per table ([rel.h#StdRdOptions](../raw/postgres-17/src/include/utils/rel.h#L345-L346)). This is different from the SQL `TRUNCATE` command, which removes all rows from a table ([ref/truncate.sgml#description](../raw/postgres-17/doc/src/sgml/ref/truncate.sgml#L33)).

**Version notes:**
- PostgreSQL 12: Differs. The 1000-page or one-sixteenth rule and the `vacuum_truncate` reloption exist ([vacuumlazy.c#should_attempt_truncation](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1852-L1866), [vacuumlazy.c#REL_TRUNCATE_MINIMUM](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L68-L72), [rel.h#StdRdOptions](../raw/postgres-12/src/include/utils/rel.h#L275)). Its check has no wraparound-failsafe test, but it skips truncation whenever `old_snapshot_threshold` is enabled ([vacuumlazy.c:1859-1863](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1859-L1863)). The 50 ms / 5 s conditional lock retry is the same, and `RelationTruncate()` does the cut ([vacuumlazy.c#lazy_truncate_heap](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1897-L1926), [storage.c#RelationTruncate](../raw/postgres-12/src/backend/catalog/storage.c#L230)).
- PostgreSQL 14: Holds, with one more skip condition. VACUUM tries it only when at least 1000 pages or 1/16 of the table can go, and skips it under the failsafe, when disabled, or when `old_snapshot_threshold` is enabled ([vacuumlazy.c#should_attempt_truncation](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L3164-L3179), [vacuumlazy.c#REL_TRUNCATE_MINIMUM](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L89-L93)). `lazy_truncate_heap()` retries a conditional `AccessExclusiveLock` every 50 ms for 5 s ([vacuumlazy.c#lazy_truncate_heap](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L3216-L3238), [vacuumlazy.c#VACUUM_TRUNCATE_LOCK_TIMEOUT](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L102-L104)). `RelationTruncate()` does the cut, and `vacuum_truncate` can disable it ([storage.c#RelationTruncate](../raw/postgres-14/src/backend/catalog/storage.c#L277-L431), [rel.h#StdRdOptions](../raw/postgres-14/src/include/utils/rel.h#L327), [ref/truncate.sgml#description](../raw/postgres-14/doc/src/sgml/ref/truncate.sgml#L33)).
- PostgreSQL 18: Holds, with a new server-wide switch. The thresholds, the 50 ms / 5 s lock retry, and `RelationTruncate()` are unchanged ([vacuumlazy.c#should_attempt_truncation](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L3184-L3218), [vacuumlazy.c#VACUUM_TRUNCATE_LOCK_TIMEOUT](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L180-L181), [storage.c#RelationTruncate](../raw/postgres-18/src/backend/catalog/storage.c#L281-L290)). `vacuum_truncate` is now also a GUC (default on, `user` context, so session scope). The storage option overrides it only when it was set on the table, which `vacuum_truncate_set` records ([guc_tables.c#vacuum_truncate](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2148-L2154), [rel.h#StdRdOptions](../raw/postgres-18/src/include/utils/rel.h#L349-L350)).
- PostgreSQL 19: Holds. The 1000-page or one-sixteenth rule, the failsafe skip, the conditional `AccessExclusiveLock` retried every 50 ms for up to 5 s, and `RelationTruncate()` are unchanged ([vacuumlazy.c#should_attempt_truncation](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L3146-L3160), [vacuumlazy.c#REL_TRUNCATE_MINIMUM](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L169-L170), [vacuumlazy.c#lazy_truncate_heap](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L3166-L3225), [vacuumlazy.c#VACUUM_TRUNCATE_LOCK_TIMEOUT](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L179-L181), [storage.c#RelationTruncate](../raw/postgres-19/src/backend/catalog/storage.c#L289), [ref/truncate.sgml#description](../raw/postgres-19/doc/src/sgml/ref/truncate.sgml#L33)). v19 adds a `vacuum_truncate` GUC (`PGC_USERSET`, session scope, default on), and the per-table option is now a three-state `pg_ternary` so "unset" can defer to the GUC ([guc_parameters.dat#vacuum_truncate](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3429-L3433), [rel.h#StdRdOptions](../raw/postgres-19/src/include/utils/rel.h#L352)).

Related: [AccessExclusiveLock](#accessexclusivelock), [VACUUM](#vacuum), [Wraparound](#wraparound)

### Tuple

**Aliases:** row, row version, heap tuple, `HeapTupleHeaderData`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A tuple is an ordered set of attribute values. When a table defines the order it is usually called a row ([glossary.sgml#glossary-tuple](../raw/postgres-17/doc/src/sgml/glossary.sgml#L1948-L1960)). An `UPDATE` writes a new tuple rather than changing the old one in place, and the old version's `t_ctid` points to the new one. So one logical row can have several tuple versions on disk ([htup_details.h#t_ctid](../raw/postgres-17/src/include/access/htup_details.h#L86-L94)). Every heap tuple starts with a 23-byte `HeapTupleHeaderData`. It holds transaction fields (`t_choice`), `t_ctid`, `t_infomask2`, `t_infomask`, the header length `t_hoff`, and a null bitmap ([htup_details.h#HeapTupleHeaderData](../raw/postgres-17/src/include/access/htup_details.h#L153-L181)). Index tuples use a different header, `IndexTupleData`: a heap TID plus a 16-bit `t_info` word ([itup.h#IndexTupleData](../raw/postgres-17/src/include/access/itup.h#L22-L50)).

**Version notes:**
- PostgreSQL 12: Holds. An update writes a new version linked by `t_ctid`, heap tuples start with `HeapTupleHeaderData`, and index tuples use `IndexTupleData` ([htup_details.h#t_ctid](../raw/postgres-12/src/include/access/htup_details.h#L85-L90), [htup_details.h#HeapTupleHeaderData](../raw/postgres-12/src/include/access/htup_details.h#L152-L180), [itup.h#IndexTupleData](../raw/postgres-12/src/include/access/itup.h#L35-L50)).
- PostgreSQL 14: Holds. The docs glossary defines a tuple, `t_ctid` links versions, `HeapTupleHeaderData` holds `t_choice`, `t_ctid`, the infomasks, `t_hoff` and the null bitmap, and index tuples use `IndexTupleData` ([glossary.sgml#glossary-tuple](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1694-L1707), [htup_details.h#t_ctid](../raw/postgres-14/src/include/access/htup_details.h#L85-L92), [htup_details.h#HeapTupleHeaderData](../raw/postgres-14/src/include/access/htup_details.h#L152-L178), [itup.h#IndexTupleData](../raw/postgres-14/src/include/access/itup.h#L35-L51)).
- PostgreSQL 18: Holds ([glossary.sgml#glossary-tuple](../raw/postgres-18/doc/src/sgml/glossary.sgml#L1997-L2009), [htup_details.h#HeapTupleHeaderData](../raw/postgres-18/src/include/access/htup_details.h#L153-L181), [itup.h#IndexTupleData](../raw/postgres-18/src/include/access/itup.h#L22-L50)).
- PostgreSQL 19: Holds. Updates still write new versions linked by `t_ctid`, `HeapTupleHeaderData` still holds `t_choice`, `t_ctid`, the infomasks, `t_hoff` and the null bitmap, and index tuples still use `IndexTupleData` ([glossary.sgml#glossary-tuple](../raw/postgres-19/doc/src/sgml/glossary.sgml#L2095-L2108), [htup_details.h#t_ctid](../raw/postgres-19/src/include/access/htup_details.h#L86-L95), [htup_details.h#HeapTupleHeaderData](../raw/postgres-19/src/include/access/htup_details.h#L153-L181), [itup.h#IndexTupleData](../raw/postgres-19/src/include/access/itup.h#L35-L51)).

Related: [Heap](#heap), [MVCC](#mvcc), [TID](#tid), [xmin and xmax](#xmin-and-xmax)

### Tuplesort

**Aliases:** `tuplesort.c`, `tuplesortvariants.c`, external sort, external merge, `tuplesort_begin_index_btree`, logical tapes. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Tuplesort is PostgreSQL's general sorting module, used by `ORDER BY`, merge joins, B-tree index builds and other callers. It sorts in memory with quicksort while the data fits in the caller's memory budget. Otherwise it writes quicksorted runs to temporary files, called "tapes", and merges them ([tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L6-L12), [tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L44)). The caller chooses the budget: most pass [work_mem](#work_mem), but a [B-tree](#b-tree) build uses [maintenance_work_mem](#maintenance_work_mem) through `tuplesort_begin_index_btree()` ([tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L32), [nbtsort.c:371-374](../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L371-L374), [tuplesortvariants.c#tuplesort_begin_index_btree](../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L352)). In PostgreSQL 17 the merge is a balanced k-way merge, and `logtape.c` reuses temp-file space as soon as each block is read ([tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L14-L24), [tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L66-L72)). EXPLAIN ANALYZE shows which path ran as the sort method: `quicksort`, `top-N heapsort`, `external sort` or `external merge` ([tuplesort.c#tuplesort_method_name](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L2580-L2595)).

**Version notes:**
- PostgreSQL 12: Runs are merged with Knuth's polyphase merge, and the per-kind sort setup, including `tuplesort_begin_index_btree()`, is still in `tuplesort.c` ([tuplesort.c header](../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L10-L20), [tuplesort.c#tuplesort_begin_index_btree](../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L975)).
- PostgreSQL 14: Same as 12, still polyphase merge in `tuplesort.c` ([tuplesort.c:17](../raw/postgres-14/src/backend/utils/sort/tuplesort.c#L17), [tuplesort.c#tuplesort_begin_index_btree](../raw/postgres-14/src/backend/utils/sort/tuplesort.c#L1068)).
- PostgreSQL 18: Holds: balanced k-way merge, with the variants in `tuplesortvariants.c` ([tuplesort.c:15-16](../raw/postgres-18/src/backend/utils/sort/tuplesort.c#L15-L16), [tuplesortvariants.c#tuplesort_begin_index_btree](../raw/postgres-18/src/backend/utils/sort/tuplesortvariants.c#L359)).
- PostgreSQL 19: Holds ([tuplesort.c:15-16](../raw/postgres-19/src/backend/utils/sort/tuplesort.c#L15-L16), [tuplesortvariants.c#tuplesort_begin_index_btree](../raw/postgres-19/src/backend/utils/sort/tuplesortvariants.c#L360)).

Related: [work_mem](#work_mem), [maintenance_work_mem](#maintenance_work_mem), [B-tree](#b-tree), [EXPLAIN](#explain)

### Two-phase commit

**Aliases:** 2PC, `PREPARE TRANSACTION`, prepared transaction, global transaction (gxact), `twophase.c`, `max_prepared_transactions`, `pg_twophase`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Two-phase commit splits a commit into `PREPARE TRANSACTION`, which makes the transaction durable under a client-chosen GID, and a later `COMMIT PREPARED` or `ROLLBACK PREPARED` ([twophase.c](../raw/postgres-17/src/backend/access/transam/twophase.c#L12-L22)). A prepared transaction keeps a dummy `PGPROC`, so its XID still counts as running and it keeps its locks ([twophase.c](../raw/postgres-17/src/backend/access/transam/twophase.c#L24-L26)). Its state goes to WAL at prepare, and a checkpoint copies it into files under `pg_twophase` ([twophase.c](../raw/postgres-17/src/backend/access/transam/twophase.c#L27-L45)). `max_prepared_transactions` defaults to 0, which disables the feature. Its context is `postmaster`, so a change needs a restart ([guc_tables.c#max_prepared_transactions](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2578-L2586), [config.sgml#guc-max-prepared-transactions](../raw/postgres-17/doc/src/sgml/config.sgml#L1831-L1833)). Logical replication subscriptions can also replicate prepared transactions; `pg_subscription.subtwophasestate` tracks this ([pg_subscription.h:86](../raw/postgres-17/src/include/catalog/pg_subscription.h#L86)).

**Version notes:**
- PostgreSQL 12: Holds for the core feature. The gxact has a dummy `PGXACT` as well as a `PGPROC` ([twophase.c](../raw/postgres-12/src/backend/access/transam/twophase.c#L24-L26), [guc.c#max_prepared_transactions](../raw/postgres-12/src/backend/utils/misc/guc.c#L2344-L2352)). Differs: subscriptions have no two-phase option; `FormData_pg_subscription` has no `subtwophasestate` field ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-12/src/include/catalog/pg_subscription.h#L39-L64)).
- PostgreSQL 14: Holds for the core feature, with context `postmaster` ([twophase.c](../raw/postgres-14/src/backend/access/transam/twophase.c#L24-L26), [guc.c#max_prepared_transactions](../raw/postgres-14/src/backend/utils/misc/guc.c#L2540-L2548)). Differs: `pg_subscription` has no `subtwophasestate` field ([pg_subscription.h#FormData_pg_subscription](../raw/postgres-14/src/include/catalog/pg_subscription.h#L42-L73)).
- PostgreSQL 18: Holds ([twophase.c](../raw/postgres-18/src/backend/access/transam/twophase.c#L24-L26), [guc_tables.c#max_prepared_transactions](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2706-L2714), [pg_subscription.h:66](../raw/postgres-18/src/include/catalog/pg_subscription.h#L66)).
- PostgreSQL 19: Holds; `PGC_POSTMASTER` in `guc_parameters.dat` ([twophase.c](../raw/postgres-19/src/backend/access/transam/twophase.c#L24-L26), [guc_parameters.dat#max_prepared_transactions](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2100-L2106), [pg_subscription.h:68](../raw/postgres-19/src/include/catalog/pg_subscription.h#L68)).

Related: [Transaction ID](#transaction-id), [xmin horizon](#xmin-horizon), [Subscription](#subscription), [Checkpoint](#checkpoint)

### Utility command

**Aliases:** utility statement, non-optimizable statement, `ProcessUtility`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A utility command is any statement the planner does not optimize [parsenodes.h#Query](../raw/postgres-17/src/include/nodes/parsenodes.h#L101-L107). Its parse tree travels in `Query.utilityStmt`. `pg_plan_queries()` wraps it in a `PlannedStmt` without calling the planner [postgres.c#pg_plan_queries](../raw/postgres-17/src/backend/tcop/postgres.c#L987-L997). Execution goes to `ProcessUtility()`, which calls `ProcessUtility_hook` if an extension set one and `standard_ProcessUtility()` otherwise [utility.c#ProcessUtility](../raw/postgres-17/src/backend/tcop/utility.c#L498-L525). Commands that support event triggers continue to `ProcessUtilitySlow` [utility.c:527-537](../raw/postgres-17/src/backend/tcop/utility.c#L527-L537).

**Version notes:**
- PostgreSQL 12: Holds. A utility statement sits in `Query.utilityStmt`, `pg_plan_queries()` wraps it without planning, `ProcessUtility()` calls the hook or `standard_ProcessUtility()`, and event-trigger commands go to `ProcessUtilitySlow` ([parsenodes.h#Query](../raw/postgres-12/src/include/nodes/parsenodes.h#L100-L106), [postgres.c#pg_plan_queries](../raw/postgres-12/src/backend/tcop/postgres.c#L956-L962), [utility.c#ProcessUtility](../raw/postgres-12/src/backend/tcop/utility.c#L338-L375), [utility.c:826](../raw/postgres-12/src/backend/tcop/utility.c#L826)).
- PostgreSQL 14: Holds. Utility statements set `Query.utilityStmt`, `pg_plan_queries()` wraps them without planning, `ProcessUtility()` honors `ProcessUtility_hook`, and event-trigger-aware commands go through `ProcessUtilitySlow` ([parsenodes.h#Query](../raw/postgres-14/src/include/nodes/parsenodes.h#L105-L111), [postgres.c#pg_plan_queries](../raw/postgres-14/src/backend/tcop/postgres.c#L894-L900), [utility.c#ProcessUtility](../raw/postgres-14/src/backend/tcop/utility.c#L503-L530), [utility.c:965](../raw/postgres-14/src/backend/tcop/utility.c#L965)).
- PostgreSQL 18: Holds ([postgres.c#pg_plan_queries](../raw/postgres-18/src/backend/tcop/postgres.c#L981-L991), [utility.c#ProcessUtility](../raw/postgres-18/src/backend/tcop/utility.c#L498-L525), [utility.c:527-537](../raw/postgres-18/src/backend/tcop/utility.c#L527-L537)).
- PostgreSQL 19: Holds. The parse tree still travels in `Query.utilityStmt`, `pg_plan_queries()` still wraps it without the planner, `ProcessUtility()` still dispatches through the hook or `standard_ProcessUtility()`, and event-trigger commands still reach `ProcessUtilitySlow()` ([parsenodes.h#Query](../raw/postgres-19/src/include/nodes/parsenodes.h#L100-L107), [postgres.c#pg_plan_queries](../raw/postgres-19/src/backend/tcop/postgres.c#L998-L1005), [utility.c#ProcessUtility](../raw/postgres-19/src/backend/tcop/utility.c#L504-L530), [utility.c:964](../raw/postgres-19/src/backend/tcop/utility.c#L964)).

Related: [Parse tree](#parse-tree), [PlannedStmt](#plannedstmt), [Hook](#hook), [Event trigger](#event-trigger), [Portal](#portal)

### VACUUM

**Aliases:** lazy vacuum, plain VACUUM, `heap_vacuum_rel`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

VACUUM removes dead row versions so their space can be reused, and does related [MVCC](#mvcc) upkeep such as [freezing](#freezing) ([glossary.sgml#glossary-vacuum](../raw/postgres-17/doc/src/sgml/glossary.sgml#L2056-L2069)). Plain VACUUM takes only `ShareUpdateExclusiveLock`, so normal reads and writes continue while it runs ([vacuum.c:2049-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2055)). For a heap table it runs `heap_vacuum_rel` ([heapam_handler.c:2632](../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L2632)). That function scans the heap to prune pages and collect dead TIDs. It then removes the matching index entries, and finally marks the heap's `LP_DEAD` slots unused. The dead TIDs are held in memory bounded by `maintenance_work_mem`, or `autovacuum_work_mem` in an autovacuum worker when that is set ([vacuumlazy.c:2826-2828](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2826-L2828), [vacuumlazy.c:1-21](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1-L21), [vacuumlazy.c#lazy_scan_heap](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L779-L816)).

**Version notes:**
- PostgreSQL 12: Holds, with a different dead-TID store. Plain VACUUM takes `ShareUpdateExclusiveLock` and runs `heap_vacuum_rel` for heap tables ([vacuum.c:1680-1681](../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1681), [heapam_handler.c:2641](../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2641)). It collects dead TIDs in a flat array bounded by `maintenance_work_mem` or `autovacuum_work_mem`, cleans indexes, then runs a second heap pass ([vacuumlazy.c:7-22](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L7-L22), [vacuumlazy.c#lazy_vacuum_heap](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1515-L1526)).
- PostgreSQL 14: Holds. Plain VACUUM takes `ShareUpdateExclusiveLock`, runs `heap_vacuum_rel` for heap tables, prunes and collects dead TIDs, cleans indexes, then marks `LP_DEAD` slots unused, with the dead-TID array bounded by `maintenance_work_mem` or `autovacuum_work_mem` ([glossary.sgml#glossary-vacuum](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1801-L1819), [vacuum.c:1919-1920](../raw/postgres-14/src/backend/commands/vacuum.c#L1919-L1920), [heapam_handler.c:2579](../raw/postgres-14/src/backend/access/heap/heapam_handler.c#L2579), [vacuumlazy.c#lazy_scan_heap](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L909), [vacuumlazy.c#compute_max_dead_tuples](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L3452-L3457)).
- PostgreSQL 18: Holds. The `vacuumlazy.c` header now describes the three phases, and the dead-TID store is still bounded by `maintenance_work_mem` or `autovacuum_work_mem` ([vacuum.c:2087-2093](../raw/postgres-18/src/backend/commands/vacuum.c#L2087-L2093), [heapam_handler.c:2656](../raw/postgres-18/src/backend/access/heap/heapam_handler.c#L2656), [vacuumlazy.c:1-39](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L1-L39), [vacuumlazy.c:3500-3502](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L3500-L3502)). New in 18: normal vacuums can eagerly scan all-visible pages to freeze them; see Freezing ([vacuumlazy.c](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L44-L58)).
- PostgreSQL 19: Holds. Plain VACUUM still takes `ShareUpdateExclusiveLock`, still runs `heap_vacuum_rel` for heap tables, still prunes and collects dead TIDs, vacuums indexes, then marks `LP_DEAD` slots unused, and still bounds the TID store by `maintenance_work_mem` or `autovacuum_work_mem` ([glossary.sgml#glossary-vacuum](../raw/postgres-19/doc/src/sgml/glossary.sgml#L2203-L2221), [vacuum.c:2084-2085](../raw/postgres-19/src/backend/commands/vacuum.c#L2084-L2085), [heapam_handler.c:2705](../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L2705), [vacuumlazy.c:3-14](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L3-L14), [vacuumlazy.c#lazy_scan_heap](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L1242-L1277), [vacuumlazy.c#dead_items_alloc](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L3443-L3445)).

Related: [Autovacuum](#autovacuum), [VACUUM FULL](#vacuum-full), [Pruning](#pruning), [Line pointer](#line-pointer), [ShareUpdateExclusiveLock](#shareupdateexclusivelock), [maintenance_work_mem](#maintenance_work_mem), [Visibility map](#visibility-map)

### Vacuum cost delay

**Aliases:** cost-based vacuum delay, `vacuum_cost_delay`, `vacuum_cost_limit`, `vacuum_cost_page_hit`/`miss`/`dirty`, `vacuum_delay_point`, `VacuumCostBalance`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

Cost-based vacuum delay throttles `VACUUM` and `ANALYZE`. The process adds an estimated cost for each page it hits, reads or dirties. When the total reaches `vacuum_cost_limit`, it sleeps for about `vacuum_cost_delay` and resets the counter ([config.sgml#runtime-config-resource-vacuum-cost](../raw/postgres-17/doc/src/sgml/config.sgml#L2395-L2409)). In source, loops call `vacuum_delay_point()` about once per page. It sleeps for `vacuum_cost_delay * balance / limit`, capped at four times the delay, and reports the `VacuumDelay` wait event ([vacuum.c#vacuum_delay_point](../raw/postgres-17/src/backend/commands/vacuum.c#L2383-L2437)). Parallel vacuum workers share one balance ([vacuum.c#vacuum_delay_point](../raw/postgres-17/src/backend/commands/vacuum.c#L2421-L2427)). `vacuum_cost_delay` (default 0, off) and `vacuum_cost_limit` have context `user`, so they apply per session. `autovacuum_vacuum_cost_delay` (default 2 ms) has context `sighup` and needs a reload ([guc_tables.c#vacuum_cost_delay](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3864-L3873), [guc_tables.c#vacuum_cost_limit](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2545-L2553), [guc_tables.c#autovacuum_vacuum_cost_delay](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3875-L3884)).

**Version notes:**
- PostgreSQL 12: Differs. `vacuum_delay_point()` has no shared parallel balance, and `vacuum_cost_page_miss` defaults to 10 instead of 2. Contexts match: `user` for the vacuum GUCs and `sighup` for the autovacuum delay ([vacuum.c#vacuum_delay_point](../raw/postgres-12/src/backend/commands/vacuum.c#L1940-L1971), [guc.c#vacuum_cost_page_miss](../raw/postgres-12/src/backend/utils/misc/guc.c#L2291-L2299), [guc.c#vacuum_cost_delay](../raw/postgres-12/src/backend/utils/misc/guc.c#L3372-L3381), [guc.c#autovacuum_vacuum_cost_delay](../raw/postgres-12/src/backend/utils/misc/guc.c#L3383-L3392)).
- PostgreSQL 14: Holds; `vacuum_cost_page_miss` defaults to 2 ([vacuum.c:2205](../raw/postgres-14/src/backend/commands/vacuum.c#L2205), [guc.c#vacuum_cost_page_miss](../raw/postgres-14/src/backend/utils/misc/guc.c#L2487-L2495), [guc.c#vacuum_cost_delay](../raw/postgres-14/src/backend/utils/misc/guc.c#L3744-L3753)).
- PostgreSQL 18: Differs slightly. `vacuum_delay_point()` takes an `is_analyze` argument. The new `track_cost_delay_timing` GUC (context `superuser`, session scope) records time spent sleeping ([vacuum.c#vacuum_delay_point](../raw/postgres-18/src/backend/commands/vacuum.c#L2437-L2449), [vacuum.c:2492-2499](../raw/postgres-18/src/backend/commands/vacuum.c#L2492-L2499), [guc_tables.c#track_cost_delay_timing](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L1503-L1511)).
- PostgreSQL 19: As 18; GUCs are in `guc_parameters.dat` ([vacuum.c:2438](../raw/postgres-19/src/backend/commands/vacuum.c#L2438), [guc_parameters.dat#vacuum_cost_delay](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3331-L3346), [guc_parameters.dat#track_cost_delay_timing](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3205-L3209)).

Related: [VACUUM](#vacuum), [Autovacuum](#autovacuum), [Wait event](#wait-event), [Parallel vacuum](#parallel-vacuum), [GUC context](#guc-context)

### Vacuum failsafe

**Aliases:** wraparound failsafe, `vacuum_failsafe_age`, `vacuum_multixact_failsafe_age`, `lazy_check_wraparound_failsafe`, `VacuumFailsafeActive`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The vacuum failsafe is VACUUM's last-resort mode when a table's `relfrozenxid` or `relminmxid` is dangerously old. VACUUM then skips everything it can in order to finish freezing sooner ([config.sgml#guc-vacuum-failsafe-age](../raw/postgres-17/doc/src/sgml/config.sgml#L9676-L9697)). `lazy_check_wraparound_failsafe()` fires when `vacuum_xid_failsafe_check()` finds the table older than `vacuum_failsafe_age`. The threshold is never below 1.05 × `autovacuum_freeze_max_age`, and the MultiXact limit follows the same rule ([vacuum.c#vacuum_xid_failsafe_check](../raw/postgres-17/src/backend/commands/vacuum.c#L1252-L1290)). Once it fires, VACUUM stops index vacuuming, index cleanup and truncation. It drops its buffer ring, turns off cost-based delay and logs a warning ([vacuumlazy.c#lazy_check_wraparound_failsafe](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2288-L2344)). VACUUM rechecks every 4 GB of scanned heap ([vacuumlazy.c:92](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L92)). Both age GUCs have context `user` (session scope) and default to 1.6 billion ([guc_tables.c#vacuum_failsafe_age](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2706-L2723)).

**Version notes:**
- PostgreSQL 12: Not present in PostgreSQL 12. The v12 GUC table groups the vacuum age settings as four freeze ages followed directly by `vacuum_defer_cleanup_age`; none of them is a failsafe age ([guc.c:2410-2453](../raw/postgres-12/src/backend/utils/misc/guc.c#L2410-L2453)).
- PostgreSQL 14: Holds; this is the first pinned version with it. GUCs are in `guc.c` with context `user` ([vacuumlazy.c#lazy_check_wraparound_failsafe](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L2588-L2628), [vacuum.c#vacuum_xid_failsafe_check](../raw/postgres-14/src/backend/commands/vacuum.c#L1174), [guc.c#vacuum_failsafe_age](../raw/postgres-14/src/backend/utils/misc/guc.c#L2666-L2683)).
- PostgreSQL 18: Holds ([vacuumlazy.c#lazy_check_wraparound_failsafe](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L2952-L3015), [guc_tables.c#vacuum_failsafe_age](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2834-L2851)).
- PostgreSQL 19: Holds; GUCs in `guc_parameters.dat`, `PGC_USERSET` ([vacuumlazy.c#lazy_check_wraparound_failsafe](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L2892-L2956), [guc_parameters.dat#vacuum_failsafe_age](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3372-L3378)).

Related: [Wraparound](#wraparound), [Freezing](#freezing), [VACUUM](#vacuum), [Vacuum cost delay](#vacuum-cost-delay), [MultiXact](#multixact)

### VACUUM FULL

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`VACUUM FULL` rewrites the whole table into a new file that contains only live rows, then rebuilds its indexes. It is a different operation from plain [VACUUM](#vacuum), not a stronger form of it. `vacuum_rel` takes `AccessExclusiveLock` for it and calls `cluster_rel` with no index ([vacuum.c:2049-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2055), [vacuum.c:2248-2260](../raw/postgres-17/src/backend/commands/vacuum.c#L2248-L2260)). `cluster_rel` then rewrites the table in physical order rather than index order ([cluster.c#cluster_rel](../raw/postgres-17/src/backend/commands/cluster.c#L306-L308)). The lock blocks readers and writers until commit ([mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1067-L1080), [vacuum.c:2252](../raw/postgres-17/src/backend/commands/vacuum.c#L2252)). The table keeps its OID but gets new files, because the rewrite swaps relfilenumbers ([cluster.c#cluster_rel](../raw/postgres-17/src/backend/commands/cluster.c#L293-L299)).

**Version notes:**
- PostgreSQL 12: Holds. `vacuum_rel()` takes `AccessExclusiveLock` and calls `cluster_rel()` with no index, which rewrites in physical order and swaps relfilenodes ([vacuum.c:1680-1681](../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1681), [vacuum.c:1827-1828](../raw/postgres-12/src/backend/commands/vacuum.c#L1827-L1828), [cluster.c#cluster_rel](../raw/postgres-12/src/backend/commands/cluster.c#L248-L263)).
- PostgreSQL 14: Holds. `vacuum_rel` takes `AccessExclusiveLock` for FULL and calls `cluster_rel` with no index, and the rewrite keeps the OID while swapping relfilenodes ([vacuum.c:1919-1920](../raw/postgres-14/src/backend/commands/vacuum.c#L1919-L1920), [vacuum.c:2075-2087](../raw/postgres-14/src/backend/commands/vacuum.c#L2075-L2087), [cluster.c#cluster_rel](../raw/postgres-14/src/backend/commands/cluster.c#L259-L264)).
- PostgreSQL 18: Holds. `vacuum_rel` still picks `AccessExclusiveLock` for FULL, and it now passes the open relation to `cluster_rel` with `InvalidOid`. `cluster_rel` closes the relation but keeps the lock ([vacuum.c:2087-2093](../raw/postgres-18/src/backend/commands/vacuum.c#L2087-L2093), [vacuum.c:2298-2313](../raw/postgres-18/src/backend/commands/vacuum.c#L2298-L2313), [cluster.c#cluster_rel](../raw/postgres-18/src/backend/commands/cluster.c#L294-L311), [mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-18/doc/src/sgml/mvcc.sgml#L1067-L1080)).
- PostgreSQL 19: Holds. `vacuum_rel()` still takes `AccessExclusiveLock` and calls `cluster_rel()` with no index, which rewrites in physical order and keeps the OID ([vacuum.c:2078-2085](../raw/postgres-19/src/backend/commands/vacuum.c#L2078-L2085), [vacuum.c:2300-2305](../raw/postgres-19/src/backend/commands/vacuum.c#L2300-L2305), [repack.c#cluster_rel](../raw/postgres-19/src/backend/commands/repack.c#L465-L478), [mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-19/doc/src/sgml/mvcc.sgml#L1081-L1100)). `cluster_rel()` now lives in `repack.c` and takes a command tag, `REPACK_COMMAND_VACUUMFULL` ([vacuum.c:2303](../raw/postgres-19/src/backend/commands/vacuum.c#L2303), [parsenodes.h#RepackCommand](../raw/postgres-19/src/include/nodes/parsenodes.h#L4063)).

Related: [CLUSTER](#cluster), [Table rewrite](#table-rewrite), [AccessExclusiveLock](#accessexclusivelock), [Relfilenumber](#relfilenumber), [Bloat](#bloat)

### varlena

**Aliases:** variable-length datum, `struct varlena`, `VARSIZE`, `VARDATA`, `VARHDRSZ`, 1-byte header, 4-byte header, `varatt.h`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A varlena is PostgreSQL's layout for variable-length values such as `text`, `bytea` and arrays: a length header followed by the data bytes ([c.h#varlena](../raw/postgres-17/src/include/c.h#L690-L717)). The header has two forms. A 4-byte header is aligned and allows values up to 1 GB, compressed or not. A 1-byte header is unaligned and allows up to 126 bytes of data, or marks a TOAST pointer ([varatt.h:142-155](../raw/postgres-17/src/include/varatt.h#L142-L155), [varatt.h#varattrib_1b](../raw/postgres-17/src/include/varatt.h#L127-L139)). Code must use macros such as `VARSIZE_ANY`, `VARDATA_ANY` and `VARSIZE_ANY_EXHDR` rather than read `vl_len_` directly ([c.h#varlena](../raw/postgres-17/src/include/c.h#L692-L700)).

**Version notes:**
- PostgreSQL 12: Holds; the header macros live in `postgres.h` because `varatt.h` does not exist yet ([c.h#varlena](../raw/postgres-12/src/include/c.h#L549-L563), [postgres.h:168-181](../raw/postgres-12/src/include/postgres.h#L168-L181), [postgres.h:341](../raw/postgres-12/src/include/postgres.h#L341)).
- PostgreSQL 14: Holds; also in `postgres.h` ([c.h#varlena](../raw/postgres-14/src/include/c.h#L666-L680), [postgres.h:179-192](../raw/postgres-14/src/include/postgres.h#L179-L192)).
- PostgreSQL 18: Holds; `varatt.h` as in 17 ([c.h#varlena](../raw/postgres-18/src/include/c.h#L672-L686), [varatt.h:142-155](../raw/postgres-18/src/include/varatt.h#L142-L155)).
- PostgreSQL 19: Differs in spelling. `varlena` is a typedef, not only `struct varlena`, and `VARSIZE_ANY_EXHDR` and similar helpers are inline functions in `varatt.h` ([c.h#varlena](../raw/postgres-19/src/include/c.h#L769-L796), [varatt.h:157-170](../raw/postgres-19/src/include/varatt.h#L157-L170), [varatt.h:472](../raw/postgres-19/src/include/varatt.h#L472)).

Related: [TOAST](#toast), [Tuple](#tuple), [Datum](#datum)

### Visibility map

**Aliases:** VM, `VISIBILITYMAP_FORKNUM`, all-visible, all-frozen. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The visibility map is a fork with two bits per heap page ([glossary.sgml#glossary-vm](../raw/postgres-17/doc/src/sgml/glossary.sgml#L2093-L2105), [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-17/src/include/access/visibilitymapdefs.h#L20-L21)):

- *all-visible* means every tuple on the page is visible to all transactions.
- *all-frozen* means every tuple on the page is frozen.

VACUUM uses the bits to skip pages. A set bit is always true. A clear bit only means "not known". Setting a bit is WAL-logged, and clearing one is replayed as part of the change that caused it ([visibilitymap.c#NOTES](../raw/postgres-17/src/backend/access/heap/visibilitymap.c#L22-L50)). An index-only scan visits the heap only when the page's all-visible bit is clear ([nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L161-L168)). Do not confuse it with the free space map, which tracks free space ([glossary.sgml#glossary-fsm](../raw/postgres-17/doc/src/sgml/glossary.sgml#L814-L828)).

**Version notes:**
- PostgreSQL 12: Holds. Two bits per heap page, all-visible and all-frozen; a set bit is reliable, setting is WAL-logged, and clearing rides on the causing change ([visibilitymap.h](../raw/postgres-12/src/include/access/visibilitymap.h#L26-L27), [visibilitymap.c#NOTES](../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L22-L50)). In 12 the bit macros live in `visibilitymap.h`; there is no `visibilitymapdefs.h`. Index-only scans skip the heap when the bit is set ([nodeIndexonlyscan.c](../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L169)).
- PostgreSQL 14: Holds. Two bits per heap page, all-visible and all-frozen; a set bit is always correct, setting is WAL-logged and clearing rides on the causing change's WAL; index-only scans consult it ([glossary.sgml#glossary-vm](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1838-L1851), [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-14/src/include/access/visibilitymapdefs.h#L20-L21), [visibilitymap.c#NOTES](../raw/postgres-14/src/backend/access/heap/visibilitymap.c#L23-L50), [nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-14/src/backend/executor/nodeIndexonlyscan.c#L164-L172)).
- PostgreSQL 18: Holds ([visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-18/src/include/access/visibilitymapdefs.h#L20-L21), [visibilitymap.c#NOTES](../raw/postgres-18/src/backend/access/heap/visibilitymap.c#L22-L50), [nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-18/src/backend/executor/nodeIndexonlyscan.c#L162-L169)).
- PostgreSQL 19: Holds. The VM still has all-visible and all-frozen bits per page with the same set/clear rules, and index-only scans still visit the heap only when the all-visible bit is clear ([glossary.sgml#glossary-vm](../raw/postgres-19/doc/src/sgml/glossary.sgml#L2240-L2253), [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-19/src/include/access/visibilitymapdefs.h#L20-L21), [visibilitymap.c#NOTES](../raw/postgres-19/src/backend/access/heap/visibilitymap.c#L23-L60), [nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-19/src/backend/executor/nodeIndexonlyscan.c#L165-L173)). In v19, on-access pruning by a read-only scan can also set VM bits, not only VACUUM ([pruneheap.c:359-361](../raw/postgres-19/src/backend/access/heap/pruneheap.c#L359-L361)).

Related: [Fork](#fork), [Freezing](#freezing), [Free space map](#free-space-map), [Index-only scan](#index-only-scan), [VACUUM](#vacuum)

### Wait event

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A wait event names what a server process is waiting for right now. Its class says what kind of wait it is, such as an LWLock, a heavyweight lock, a buffer pin, IPC, a timeout or I/O ([wait_event.h:18-27](../raw/postgres-17/src/include/utils/wait_event.h#L18-L27)). `pg_stat_activity` shows it in the `wait_event_type` and `wait_event` columns ([system_views.sql#pg_stat_activity](../raw/postgres-17/src/backend/catalog/system_views.sql#L865-L882)). Code brackets each wait with `pgstat_report_wait_start()` and `pgstat_report_wait_end()`. The start call writes one 4-byte value that encodes the class and the event ([wait_event.h#pgstat_report_wait_start](../raw/postgres-17/src/include/utils/wait_event.h#L65-L92)). `wait_event_names.txt` lists every event. The build generates the enum header, the name lookup code, and the documentation tables from it ([wait_event_names.txt](../raw/postgres-17/src/backend/utils/activity/wait_event_names.txt#L7-L24)).

**Version notes:**
- PostgreSQL 12: Differs in where events are defined. The wait classes (LWLock, lock, buffer pin, activity, client, extension, IPC, timeout, I/O) and `pgstat_report_wait_start()` are in `pgstat.h`, and `pg_stat_activity` shows `wait_event_type` and `wait_event` ([pgstat.h:750-763](../raw/postgres-12/src/include/pgstat.h#L750-L763), [pgstat.h#pgstat_report_wait_start](../raw/postgres-12/src/include/pgstat.h#L1319-L1330), [system_views.sql#pg_stat_activity](../raw/postgres-12/src/backend/catalog/system_views.sql#L747-L748)). 12 has no `wait_event.h` and no `wait_event_names.txt`; events are hand-written enums in `pgstat.h`, and `pgstat_get_wait_event()` in `pgstat.c` maps them to names ([pgstat.h:775](../raw/postgres-12/src/include/pgstat.h#L775), [pgstat.c#pgstat_get_wait_event](../raw/postgres-12/src/backend/postmaster/pgstat.c#L3557)).
- PostgreSQL 14: Differs in how events are defined. The classes, the `pg_stat_activity` columns and `pgstat_report_wait_start()` writing one 4-byte value match ([wait_event.h:18-26](../raw/postgres-14/src/include/utils/wait_event.h#L18-L26), [system_views.sql#pg_stat_activity](../raw/postgres-14/src/backend/catalog/system_views.sql#L828-L829), [wait_event.h#pgstat_report_wait_start](../raw/postgres-14/src/include/utils/wait_event.h#L245-L271)). But 14 has no `wait_event_names.txt`; the event enums are hand-written in `wait_event.h` and the names in `wait_event.c` ([wait_event.h:157](../raw/postgres-14/src/include/utils/wait_event.h#L157), [wait_event.c#pgstat_get_wait_event](../raw/postgres-14/src/backend/utils/activity/wait_event.c#L129)).
- PostgreSQL 18: Holds, with the class list moved. The `PG_WAIT_*` class constants now live in `wait_classes.h`, not `wait_event.h` ([wait_classes.h](../raw/postgres-18/src/include/utils/wait_classes.h#L18-L27)). `pgstat_report_wait_start()` still writes one 4-byte value, and `wait_event_names.txt` still drives the generated files ([wait_event.h#pgstat_report_wait_start](../raw/postgres-18/src/include/utils/wait_event.h#L48-L76), [wait_event_names.txt](../raw/postgres-18/src/backend/utils/activity/wait_event_names.txt#L7-L24), [system_views.sql#pg_stat_activity](../raw/postgres-18/src/backend/catalog/system_views.sql#L878-L895)).
- PostgreSQL 19: Holds. `pg_stat_activity` still shows `wait_event_type` and `wait_event`, `pgstat_report_wait_start()` still writes one 4-byte value, and `wait_event_names.txt` still generates the enum, lookup code and docs ([system_views.sql#pg_stat_activity](../raw/postgres-19/src/backend/catalog/system_views.sql#L939-L956), [wait_event.h#pgstat_report_wait_start](../raw/postgres-19/src/include/utils/wait_event.h#L67), [wait_event_names.txt](../raw/postgres-19/src/backend/utils/activity/wait_event_names.txt#L7-L24)). The class constants moved to a new header, `wait_classes.h`, which also lists an `InjectionPoint` class ([wait_classes.h:18-27](../raw/postgres-19/src/include/utils/wait_classes.h#L18-L27)).

Related: [Cumulative statistics](#cumulative-statistics), [LWLock](#lwlock), [Heavyweight lock](#heavyweight-lock)

### WAL

**Aliases:** write-ahead log, XLOG, xlog. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The sequential journal of every change to the cluster. It is written before the changed data pages, so replaying it after a crash restores a consistent state ([glossary.sgml#wal](../raw/postgres-17/doc/src/sgml/glossary.sgml#L2269-L2279), [transam/README](../raw/postgres-17/src/backend/access/transam/README#L402-L415)). Source code calls it XLOG ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L402)). A WAL-logged change follows a fixed recipe: lock the buffer, enter a critical section, change the page, `MarkBufferDirty()`, `XLogBeginInsert()` and `XLogRegister*()`, `XLogInsert()`, then `PageSetLSN()` ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L437-L466)). The same stream also serves point-in-time recovery and hot-standby replication through log shipping ([transam/README](../raw/postgres-17/src/backend/access/transam/README#L402-L404)).

**Version notes:**
- PostgreSQL 12: Holds. Source calls it XLOG, it serves crash recovery, PITR and hot standby, and the write-ahead rule and WAL-logging recipe are the same ([transam/README](../raw/postgres-12/src/backend/access/transam/README#L391-L409), [transam/README](../raw/postgres-12/src/backend/access/transam/README#L441-L456)).
- PostgreSQL 14: Holds. The WAL (XLOG in code) guarantees crash recovery and also serves PITR and hot standby, and a WAL-logged change follows the same recipe ([glossary.sgml#glossary-wal](../raw/postgres-14/doc/src/sgml/glossary.sgml#L1964-L1976), [transam/README](../raw/postgres-14/src/backend/access/transam/README#L402-L405), [transam/README](../raw/postgres-14/src/backend/access/transam/README#L438-L466)).
- PostgreSQL 18: Holds ([glossary.sgml#wal](../raw/postgres-18/doc/src/sgml/glossary.sgml#L2318-L2328), [transam/README](../raw/postgres-18/src/backend/access/transam/README#L402-L415), [transam/README](../raw/postgres-18/src/backend/access/transam/README#L437-L466)).
- PostgreSQL 19: Holds. The glossary and README definitions, the XLOG name, and the lock / critical section / change / `MarkBufferDirty()` / `XLogInsert()` / `PageSetLSN()` recipe are unchanged ([glossary.sgml#glossary-wal](../raw/postgres-19/doc/src/sgml/glossary.sgml#L2416-L2428), [transam/README](../raw/postgres-19/src/backend/access/transam/README#L399-L416), [transam/README](../raw/postgres-19/src/backend/access/transam/README#L437-L466)). v19 adds `effective_wal_level`, which can read `logical` while `wal_level` is `replica` ([logicalctl.c header](../raw/postgres-19/src/backend/replication/logical/logicalctl.c#L5-L22)).

Related: [LSN](#lsn), [Checkpoint](#checkpoint), [Full-page image](#full-page-image), [Crash recovery](#crash-recovery), [Critical section](#critical-section), [Logical decoding](#logical-decoding)

### WAL level

**Aliases:** `wal_level`, `minimal`, `replica`, `logical`, `WalLevel`, `effective_wal_level` (PostgreSQL 19). **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`wal_level` decides how much information the server writes to WAL. `minimal` keeps only what crash recovery needs. `replica`, the default, adds what archiving, replication and hot standby need. `logical` adds what logical decoding needs, and each level includes the ones below it ([config.sgml#guc-wal-level](../raw/postgres-17/doc/src/sgml/config.sgml#L2969-L2977)). In source the levels are the `WalLevel` enum, and code tests them through macros such as `XLogLogicalInfoActive()` ([xlog.h#WalLevel](../raw/postgres-17/src/include/access/xlog.h#L69-L75), [xlog.h:123-124](../raw/postgres-17/src/include/access/xlog.h#L123-L124)). Logical decoding raises an error below `logical` ([logical.c#CheckLogicalDecodingRequirements](../raw/postgres-17/src/backend/replication/logical/logical.c#L125-L128)). The GUC has context `postmaster`, so changing it needs a restart ([guc_tables.c#wal_level](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4986-L4994)).

**Version notes:**
- PostgreSQL 12: Holds; context `postmaster` (restart) ([xlog.h#WalLevel](../raw/postgres-12/src/include/access/xlog.h#L160-L165), [logical.c:87-90](../raw/postgres-12/src/backend/replication/logical/logical.c#L87-L90), [guc.c#wal_level](../raw/postgres-12/src/backend/utils/misc/guc.c#L4409-L4417)).
- PostgreSQL 14: Holds ([xlog.h#WalLevel](../raw/postgres-14/src/include/access/xlog.h#L163-L168), [logical.c:117-120](../raw/postgres-14/src/backend/replication/logical/logical.c#L117-L120), [guc.c#wal_level](../raw/postgres-14/src/backend/utils/misc/guc.c#L4856-L4864)).
- PostgreSQL 18: Holds ([xlog.h#WalLevel](../raw/postgres-18/src/include/access/xlog.h#L71-L77), [logical.c:125-128](../raw/postgres-18/src/backend/replication/logical/logical.c#L125-L128), [guc_tables.c#wal_level](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5249-L5257)).
- PostgreSQL 19: Differs. With `wal_level = replica`, the server can still write logical-level WAL on demand. `XLogLogicalInfoActive()` is also true when a process-local `XLogLogicalInfo` flag is set. The read-only `effective_wal_level` (`PGC_INTERNAL`) shows `logical` while logical replication slots exist ([xlog.h:141-151](../raw/postgres-19/src/include/access/xlog.h#L141-L151), [config.sgml#guc-effective-wal-level](../raw/postgres-19/doc/src/sgml/config.sgml#L12387-L12403), [guc_parameters.dat#effective_wal_level](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L857-L864)). `CheckLogicalDecodingRequirements()` now asserts only `wal_level >= replica`, which `CheckSlotRequirements()` has already checked ([logical.c#CheckLogicalDecodingRequirements](../raw/postgres-19/src/backend/replication/logical/logical.c#L116-L132)). `wal_level` itself is still `PGC_POSTMASTER` (restart) ([guc_parameters.dat#wal_level](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3505-L3510)).

Related: [WAL](#wal), [Logical decoding](#logical-decoding), [Replication slot](#replication-slot), [Hot standby](#hot-standby), [REPACK](#repack)

### WAL receiver

**Aliases:** walreceiver, `walreceiver.c`, `WalReceiverMain`, `pg_stat_wal_receiver`, `wal_receiver_timeout`, `wal_receiver_status_interval`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The WAL receiver is the standby-side process that streams WAL from the primary. The startup process asks the postmaster to start it. It connects to a WAL sender on the primary, writes the received WAL to disk, and advances `WalRcv->flushedUpto` so the startup process knows how far it may replay ([walreceiver.c](../raw/postgres-17/src/backend/replication/walreceiver.c#L5-L16)). If streaming ends but the connection stays up, it waits for new instructions instead of exiting ([walreceiver.c](../raw/postgres-17/src/backend/replication/walreceiver.c#L24-L30)). The `pg_stat_wal_receiver` view shows its state ([system_views.sql#pg_stat_wal_receiver](../raw/postgres-17/src/backend/catalog/system_views.sql#L932)). `wal_receiver_timeout` ends a silent connection and `wal_receiver_status_interval` paces status replies. Both have context `sighup` (reload) ([guc_tables.c#wal_receiver_status_interval](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2192-L2212)).

**Version notes:**
- PostgreSQL 12: Holds; both GUCs `sighup` ([walreceiver.c](../raw/postgres-12/src/backend/replication/walreceiver.c#L5-L7), [walreceiver.c#WalReceiverMain](../raw/postgres-12/src/backend/replication/walreceiver.c#L167), [guc.c#wal_receiver_status_interval](../raw/postgres-12/src/backend/utils/misc/guc.c#L2107-L2127)).
- PostgreSQL 14: Holds ([walreceiver.c#WalReceiverMain](../raw/postgres-14/src/backend/replication/walreceiver.c#L175), [guc.c#wal_receiver_status_interval](../raw/postgres-14/src/backend/utils/misc/guc.c#L2280-L2300)).
- PostgreSQL 18: Holds ([walreceiver.c#WalReceiverMain](../raw/postgres-18/src/backend/replication/walreceiver.c#L159), [guc_tables.c#wal_receiver_status_interval](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2309-L2329)).
- PostgreSQL 19: Differs. `wal_receiver_timeout` has context `PGC_USERSET` (session scope), not `sighup`. `wal_receiver_status_interval` stays `sighup` (reload) ([guc_parameters.dat#wal_receiver_timeout](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3533-L3541), [guc_parameters.dat#wal_receiver_status_interval](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3524-L3531), [walreceiver.c#WalReceiverMain](../raw/postgres-19/src/backend/replication/walreceiver.c#L155)).

Related: [WAL sender](#wal-sender), [Hot standby](#hot-standby), [Timeline](#timeline), [Synchronous replication](#synchronous-replication)

### WAL sender

**Aliases:** walsender, `walsender.c`, `max_wal_senders`, `wal_sender_timeout`, `pg_stat_replication`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

A WAL sender is a primary-side process that streams WAL to one client, either a standby's WAL receiver or a replication tool. Like a backend, it serves one connection, but it accepts only replication commands such as `START_REPLICATION` ([walsender.c](../raw/postgres-17/src/backend/replication/walsender.c#L5-L18)). For logical replication, it decodes WAL through a logical slot before sending it ([walsender.c#StartLogicalReplication](../raw/postgres-17/src/backend/replication/walsender.c#L1487-L1500)). `WalSndCheckTimeOut()` ends the connection when no reply arrives within `wal_sender_timeout` ([walsender.c#WalSndCheckTimeOut](../raw/postgres-17/src/backend/replication/walsender.c#L2803-L2826)). The `pg_stat_replication` view shows one row per WAL sender ([system_views.sql#pg_stat_replication](../raw/postgres-17/src/backend/catalog/system_views.sql#L893)). `max_wal_senders` has context `postmaster` (restart). `wal_sender_timeout` has context `user`, so it can be set per session ([guc_tables.c#max_wal_senders](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2935-L2943), [guc_tables.c#wal_sender_timeout](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2969-L2978)).

**Version notes:**
- PostgreSQL 12: Holds; `max_wal_senders` `postmaster`, `wal_sender_timeout` `user` ([walsender.c](../raw/postgres-12/src/backend/replication/walsender.c#L5-L9), [walsender.c#WalSndCheckTimeOut](../raw/postgres-12/src/backend/replication/walsender.c#L2126), [guc.c#max_wal_senders](../raw/postgres-12/src/backend/utils/misc/guc.c#L2635-L2643), [guc.c#wal_sender_timeout](../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2665)).
- PostgreSQL 14: Holds ([walsender.c#WalSndCheckTimeOut](../raw/postgres-14/src/backend/replication/walsender.c#L2296), [guc.c#max_wal_senders](../raw/postgres-14/src/backend/utils/misc/guc.c#L2872-L2880), [guc.c#wal_sender_timeout](../raw/postgres-14/src/backend/utils/misc/guc.c#L2906-L2915)).
- PostgreSQL 18: Holds ([walsender.c#WalSndCheckTimeOut](../raw/postgres-18/src/backend/replication/walsender.c#L2801), [guc_tables.c#max_wal_senders](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3063-L3071), [guc_tables.c#wal_sender_timeout](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3097-L3106)).
- PostgreSQL 19: Holds; GUCs in `guc_parameters.dat` with the same contexts ([walsender.c#WalSndCheckTimeOut](../raw/postgres-19/src/backend/replication/walsender.c#L2966), [guc_parameters.dat#max_wal_senders](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L2177-L2183), [guc_parameters.dat#wal_sender_timeout](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3578-L3585)).

Related: [WAL receiver](#wal-receiver), [Replication slot](#replication-slot), [Logical decoding](#logical-decoding), [Synchronous replication](#synchronous-replication), [Backend](#backend)

### work_mem

**Checked on:** PostgreSQL 12, 14, 17, 18, 19.

`work_mem` caps the memory each sort or hash table in a query may use before it spills to temporary files. Its context is `user`, so a session or transaction can `SET` it with no reload or restart. Its default is 4096 kB (4 MB). The limit applies to each sort operation or hash table, not to each query [guc_tables.c#work_mem](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2448-L2457). Sorts receive it as their `workMem` budget [tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L31-L37). Hash-based nodes may use `work_mem * hash_mem_multiplier` [nodeHash.c#get_hash_memory_limit](../raw/postgres-17/src/backend/executor/nodeHash.c#L3602-L3613).

**Version notes:**
- PostgreSQL 12: Differs. It is `PGC_USERSET` (session scope), default 4096 kB, per sort or hash table ([guc.c#work_mem](../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2240)). Sorts use it as `workMem` ([tuplesort.c header](../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L28-L40)). 12 has no `hash_mem_multiplier`: the hash join sizes its table from plain `work_mem` ([nodeHash.c:704](../raw/postgres-12/src/backend/executor/nodeHash.c#L704)).
- PostgreSQL 14: Holds. It is `PGC_USERSET` (session scope) with default 4096 kB, sorts use it as `workMem`, and hash nodes may use `work_mem * hash_mem_multiplier` ([guc.c#work_mem](../raw/postgres-14/src/backend/utils/misc/guc.c#L2415-L2424), [tuplesort.c header](../raw/postgres-14/src/backend/utils/sort/tuplesort.c#L25-L28), [nodeHash.c#get_hash_memory_limit](../raw/postgres-14/src/backend/executor/nodeHash.c#L3430-L3449)).
- PostgreSQL 18: Holds. Still `user` context (session scope), default 4096 kB ([guc_tables.c#work_mem](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L2576-L2585), [tuplesort.c header](../raw/postgres-18/src/backend/utils/sort/tuplesort.c#L31-L37), [nodeHash.c#get_hash_memory_limit](../raw/postgres-18/src/backend/executor/nodeHash.c#L3622-L3633)).
- PostgreSQL 19: Holds. It is still `PGC_USERSET` (session scope) with default 4096 kB per sort or hash table, sorts still get it as `workMem`, and hash nodes still multiply by `hash_mem_multiplier` ([guc_parameters.dat#work_mem](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L3632-L3640), [tuplesort.c header](../raw/postgres-19/src/backend/utils/sort/tuplesort.c#L31-L37), [nodeHash.c#get_hash_memory_limit](../raw/postgres-19/src/backend/executor/nodeHash.c#L3672-L3690)).

Related: [maintenance_work_mem](#maintenance_work_mem), [GUC context](#guc-context), [Executor](#executor)

### Wraparound

**Aliases:** transaction ID wraparound, XID wraparound. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The danger that 32-bit transaction IDs run all the way around. Normal xids compare modulo 2^32, so an unfrozen xid more than half the space old would start to compare as newer than current ones ([transam.c#TransactionIdPrecedes](../raw/postgres-17/src/backend/access/transam/transam.c#L276-L293)). `SetTransactionIdLimit()` puts the hard limit halfway around from the oldest unfrozen xid in any database ([varsup.c#SetTransactionIdLimit](../raw/postgres-17/src/backend/access/transam/varsup.c#L382-L391)). It then sets three thresholds below that limit ([varsup.c#SetTransactionIdLimit](../raw/postgres-17/src/backend/access/transam/varsup.c#L405-L440)):

- At `xidVacLimit`, forced autovacuums begin.
- 40 million xids before the limit, warnings begin.
- 3 million xids before the limit, the server refuses to assign new xids.

`GetNewTransactionId()` enforces the refusal ([varsup.c#GetNewTransactionId](../raw/postgres-17/src/backend/access/transam/varsup.c#L147-L159)). `autovacuum_freeze_max_age` is a `postmaster` GUC, so changing it needs a restart ([guc_tables.c#autovacuum_freeze_max_age](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3377-L3384)).

**Version notes:**
- PostgreSQL 12: Differs in thresholds. `SetTransactionIdLimit()` puts the wrap limit halfway around, stops new xids 1 million before it, and starts warnings 10 million before the stop (11 million before wrap), not 3 and 40 million ([varsup.c#SetTransactionIdLimit](../raw/postgres-12/src/backend/access/transam/varsup.c#L347-L392)). `GetNewTransactionId()` enforces it ([varsup.c#GetNewTransactionId](../raw/postgres-12/src/backend/access/transam/varsup.c#L49-L105)). `autovacuum_freeze_max_age` is `PGC_POSTMASTER` (restart) ([guc.c#autovacuum_freeze_max_age](../raw/postgres-12/src/backend/utils/misc/guc.c#L2967-L2968)).
- PostgreSQL 14: Holds. `SetTransactionIdLimit()` puts the wrap limit halfway from the oldest `datfrozenxid`, forces autovacuum at `xidVacLimit`, warns 40 million before and stops 3 million before, and `GetNewTransactionId()` enforces the stop ([varsup.c#SetTransactionIdLimit](../raw/postgres-14/src/backend/access/transam/varsup.c#L362-L421), [varsup.c#GetNewTransactionId](../raw/postgres-14/src/backend/access/transam/varsup.c#L106-L130)). `autovacuum_freeze_max_age` is `PGC_POSTMASTER`, so a change needs a restart ([guc.c#autovacuum_freeze_max_age](../raw/postgres-14/src/backend/utils/misc/guc.c#L3279)).
- PostgreSQL 18: Holds. `autovacuum_freeze_max_age` is still `postmaster` context (restart) ([transam.c#TransactionIdPrecedes](../raw/postgres-18/src/backend/access/transam/transam.c#L276-L293), [varsup.c#SetTransactionIdLimit](../raw/postgres-18/src/backend/access/transam/varsup.c#L405-L440), [varsup.c#GetNewTransactionId](../raw/postgres-18/src/backend/access/transam/varsup.c#L147-L159), [guc_tables.c#autovacuum_freeze_max_age](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L3576-L3584)).
- PostgreSQL 19: Differs in one threshold. XIDs still compare modulo 2^32, the hard limit is still halfway around from the oldest unfrozen XID, forced autovacuums still start at `xidVacLimit`, the stop limit is still 3 million XIDs before wrap, and `GetNewTransactionId()` still enforces it ([transam.h#TransactionIdPrecedes](../raw/postgres-19/src/include/access/transam.h#L259-L275), [varsup.c#SetTransactionIdLimit](../raw/postgres-19/src/backend/access/transam/varsup.c#L378-L402), [varsup.c#SetTransactionIdLimit](../raw/postgres-19/src/backend/access/transam/varsup.c#L418-L435), [varsup.c#GetNewTransactionId](../raw/postgres-19/src/backend/access/transam/varsup.c#L124-L160)). Warnings now start **100 million** XIDs before the limit, not 40 million ([varsup.c#SetTransactionIdLimit](../raw/postgres-19/src/backend/access/transam/varsup.c#L404-L416)). `autovacuum_freeze_max_age` is still `PGC_POSTMASTER` (restart) ([guc_parameters.dat#autovacuum_freeze_max_age](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L156-L162)).

Related: [Transaction ID](#transaction-id), [Freezing](#freezing), [Autovacuum](#autovacuum), [MultiXact](#multixact)

### xmin and xmax

**Aliases:** `t_xmin`, `t_xmax`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The two transaction IDs stored in every heap tuple header. `t_xmin` is the transaction that inserted the version; `t_xmax` is the one that deleted or locked it, or 0 if none ([htup_details.h#HeapTupleFields](../raw/postgres-17/src/include/access/htup_details.h#L122-L132)). Infomask bits change how `t_xmax` reads: it may be only a row locker (`HEAP_XMAX_LOCK_ONLY`) or a MultiXactId (`HEAP_XMAX_IS_MULTI`) ([htup_details.h](../raw/postgres-17/src/include/access/htup_details.h#L196-L209)). A snapshot also has fields named `xmin` and `xmax`, but they are visibility bounds, not per-row stamps ([snapshot.h#SnapshotData](../raw/postgres-17/src/include/utils/snapshot.h#L156-L157)).

**Version notes:**
- PostgreSQL 12: Holds. `t_xmin` and `t_xmax` are in `HeapTupleFields`, and `HEAP_XMAX_LOCK_ONLY` and `HEAP_XMAX_IS_MULTI` qualify `t_xmax` ([htup_details.h#HeapTupleFields](../raw/postgres-12/src/include/access/htup_details.h#L121-L132), [htup_details.h](../raw/postgres-12/src/include/access/htup_details.h#L196-L208)).
- PostgreSQL 14: Holds. `t_xmin` and `t_xmax` live in `HeapTupleFields`, `HEAP_XMAX_LOCK_ONLY` and `HEAP_XMAX_IS_MULTI` change how `t_xmax` reads, and snapshots have separate `xmin`/`xmax` bounds ([htup_details.h#HeapTupleFields](../raw/postgres-14/src/include/access/htup_details.h#L121-L131), [htup_details.h](../raw/postgres-14/src/include/access/htup_details.h#L196-L208), [snapshot.h#SnapshotData](../raw/postgres-14/src/include/utils/snapshot.h#L157-L158)).
- PostgreSQL 18: Holds ([htup_details.h#HeapTupleFields](../raw/postgres-18/src/include/access/htup_details.h#L122-L132), [htup_details.h](../raw/postgres-18/src/include/access/htup_details.h#L196-L209), [snapshot.h#SnapshotData](../raw/postgres-18/src/include/utils/snapshot.h#L152-L153)).
- PostgreSQL 19: Holds. `t_xmin` and `t_xmax` are unchanged, `HEAP_XMAX_LOCK_ONLY` and `HEAP_XMAX_IS_MULTI` still qualify `t_xmax`, and the snapshot's `xmin`/`xmax` are still visibility bounds ([htup_details.h#HeapTupleFields](../raw/postgres-19/src/include/access/htup_details.h#L122-L132), [htup_details.h](../raw/postgres-19/src/include/access/htup_details.h#L197-L209), [snapshot.h#SnapshotData](../raw/postgres-19/src/include/utils/snapshot.h#L153-L154)).

Related: [MVCC](#mvcc), [Snapshot](#snapshot), [Transaction ID](#transaction-id), [MultiXact](#multixact), [Tuple](#tuple)

### xmin horizon

**Aliases:** OldestXmin, removable cutoff, `GlobalVisState`. **Checked on:** PostgreSQL 12, 14, 17, 18, 19.

The oldest transaction ID that some session or replication slot might still need to see. Rows deleted by transactions older than it are dead to everyone and can be removed ([procarray.c#GlobalVisState](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L102-L122)). `ComputeXidHorizons()` computes separate horizons for shared, catalog, user-data and temporary relations. Replication-slot xmins hold back all but the temporary one ([procarray.c](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L128-L158), [procarray.c#ComputeXidHorizonsResult](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L234-L237)). VACUUM reads it as `cutoffs->OldestXmin` ([vacuum.c:1120](../raw/postgres-17/src/backend/commands/vacuum.c#L1120)). So an old snapshot in any session of the same database, or an old slot `xmin`, keeps VACUUM from removing rows deleted after it ([procarray.c](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L145-L151)).

**Version notes:**
- PostgreSQL 12: Differs in mechanism. 12 has no `GlobalVisState` or `ComputeXidHorizons()`; VACUUM calls `GetOldestXmin()`, which considers only the current database for non-shared relations and folds in replication-slot xmins ([procarray.c#GetOldestXmin](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1263-L1307), [procarray.c:1379-1380](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1379-L1380), [vacuum.c:910](../raw/postgres-12/src/backend/commands/vacuum.c#L910)). The effect is the same: an old snapshot in the database, or an old slot xmin, holds back removal.
- PostgreSQL 14: Holds, but VACUUM reads it differently. 14 has `GlobalVisState` and `ComputeXidHorizons()`, with the same shared, catalog, data and temp horizons and slot xmins ([procarray.c#GlobalVisState](../raw/postgres-14/src/backend/storage/ipc/procarray.c#L128-L168), [procarray.c#ComputeXidHorizons](../raw/postgres-14/src/backend/storage/ipc/procarray.c#L1750)). 14 has no `cutoffs` struct: `vacuum_set_xid_limits()` returns `oldestXmin` from `GetOldestNonRemovableTransactionId()` ([vacuum.c#vacuum_set_xid_limits](../raw/postgres-14/src/backend/commands/vacuum.c#L945-L997)).
- PostgreSQL 18: Holds ([procarray.c#GlobalVisState](../raw/postgres-18/src/backend/storage/ipc/procarray.c#L102-L122), [procarray.c#ComputeXidHorizonsResult](../raw/postgres-18/src/backend/storage/ipc/procarray.c#L234-L237), [vacuum.c:1152](../raw/postgres-18/src/backend/commands/vacuum.c#L1152)).
- PostgreSQL 19: Holds. `GlobalVisState` still decides removability, `ComputeXidHorizons()` still computes shared, catalog, data and temporary horizons with slot xmins held in, and VACUUM still reads `cutoffs->OldestXmin` ([procarray.c#GlobalVisState](../raw/postgres-19/src/backend/storage/ipc/procarray.c#L119-L183), [procarray.c#ComputeXidHorizonsResult](../raw/postgres-19/src/backend/storage/ipc/procarray.c#L196-L261), [procarray.c#ComputeXidHorizons](../raw/postgres-19/src/backend/storage/ipc/procarray.c#L1674), [vacuum.c:1142](../raw/postgres-19/src/backend/commands/vacuum.c#L1142)).

Related: [Snapshot](#snapshot), [MVCC](#mvcc), [Pruning](#pruning), [VACUUM](#vacuum), [Replication slot](#replication-slot), [Bloat](#bloat)

## Open Questions

- Verification depth: every citation was checked mechanically (file exists at the pin, line range in bounds, version notes cite only their own version). Reviewers re-opened about a third of the new-entry citations and spot-checked 15 to 28 version-note lines per version. The remaining citations have been read only by the agent that wrote them, so `verified_by_agent` stays `not yet`.
- COMMENT ON: the entry states only what `REINDEX CONCURRENTLY` does with an index comment. It does not state what plain `REINDEX` does, because no separate citation was gathered for that path.

## Source References

One representative citation per cited source file, grouped by version:

**PostgreSQL 12** (315 files):

- [configure.in#blocksize](../raw/postgres-12/configure.in#L250-L277)
- [contrib/Makefile:31-37](../raw/postgres-12/contrib/Makefile#L31-L37)
- [README:1-25](../raw/postgres-12/contrib/README#L1-L25)
- [amcheck.control](../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L3)
- [verify_nbtree.c#bt_index_check](../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L204)
- [btree_gin.control](../raw/postgres-12/contrib/btree_gin/btree_gin.control#L1-L5)
- [btree_gist.control](../raw/postgres-12/contrib/btree_gist/btree_gist.control#L1-L5)
- [ginfuncs.c header](../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L1-L3)
- [heapfuncs.c header](../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L1-L4)
- [pageinspect--1.5--1.6.sql](../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88)
- [pageinspect.control](../raw/postgres-12/contrib/pageinspect/pageinspect.control#L1-L5)
- [rawpage.c#get_raw_page_internal](../raw/postgres-12/contrib/pageinspect/rawpage.c#L95-L106)
- [pg_freespacemap--1.1.sql](../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L25)
- [pg_freespacemap.c#pg_freespace](../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c#L24-L38)
- [pg_freespacemap.control](../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.control#L1-L5)
- [pg_stat_statements.c:437](../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L437)
- [pgstatapprox.c#pgstattuple_approx](../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L224)
- [pgstatindex.c#pgstatindex_impl](../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L238)
- [pgstattuple--1.4--1.5.sql](../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L37)
- [pgstattuple.c#pgstattuple_type](../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L55-L63)
- [auto-explain.sgml:23](../raw/postgres-12/doc/src/sgml/auto-explain.sgml#L23)
- [bki.sgml:44-64](../raw/postgres-12/doc/src/sgml/bki.sgml#L44-L64)
- [catalogs.sgml#view-pg-settings](../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10506-L10610)
- [config.sgml#guc-huge-pages](../raw/postgres-12/doc/src/sgml/config.sgml#L1533-L1560)
- [contrib.sgml:6-15](../raw/postgres-12/doc/src/sgml/contrib.sgml#L6-L15)
- [datatype.sgml](../raw/postgres-12/doc/src/sgml/datatype.sgml#L4541-L4545)
- [ddl.sgml#ddl-partitioning-declarative](../raw/postgres-12/doc/src/sgml/ddl.sgml#L3608)
- [event-trigger.sgml:28-34](../raw/postgres-12/doc/src/sgml/event-trigger.sgml#L28-L34)
- [extend.sgml:328-341](../raw/postgres-12/doc/src/sgml/extend.sgml#L328-L341)
- [gin.sgml#gin-fast-update](../raw/postgres-12/doc/src/sgml/gin.sgml#L470-L496)
- [gist.sgml#gist-intro](../raw/postgres-12/doc/src/sgml/gist.sgml#L11)
- [high-availability.sgml#hot-standby](../raw/postgres-12/doc/src/sgml/high-availability.sgml#L1665)
- [indexam.sgml#intro](../raw/postgres-12/doc/src/sgml/indexam.sgml#L38-L42)
- [indices.sgml#indexes-expressional](../raw/postgres-12/doc/src/sgml/indices.sgml#L679-L700)
- [logical-replication.sgml#logical-replication-conflicts](../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L308-L318)
- [maintenance.sgml#routine-reindex](../raw/postgres-12/doc/src/sgml/maintenance.sgml#L877-L879)
- [monitoring.sgml#monitoring-stats](../raw/postgres-12/doc/src/sgml/monitoring.sgml#L132-L155)
- [mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1002-L1028)
- [ref/alter_table.sgml:368-370](../raw/postgres-12/doc/src/sgml/ref/alter_table.sgml#L368-L370)
- [ref/cluster.sgml#description](../raw/postgres-12/doc/src/sgml/ref/cluster.sgml#L44-L48)
- [ref/create_index.sgml](../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L544-L556)
- [ref/create_table.sgml#fillfactor](../raw/postgres-12/doc/src/sgml/ref/create_table.sgml#L1319-L1337)
- [ref/explain.sgml#BUFFERS](../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L166-L191)
- [pgupgrade.sgml](../raw/postgres-12/doc/src/sgml/ref/pgupgrade.sgml#L39-L55)
- [ref/reindex.sgml#bloated](../raw/postgres-12/doc/src/sgml/ref/reindex.sgml#L47-L56)
- [ref/vacuum.sgml#INDEX_CLEANUP](../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L187-L204)
- [regress.sgml](../raw/postgres-12/doc/src/sgml/regress.sgml#L443-L450)
- [spi.sgml](../raw/postgres-12/doc/src/sgml/spi.sgml#L10-L27)
- [storage.sgml](../raw/postgres-12/doc/src/sgml/storage.sgml#L205-L216)
- [wal.sgml](../raw/postgres-12/doc/src/sgml/wal.sgml#L450-L462)
- [xfunc.sgml#xfunc-volatility](../raw/postgres-12/doc/src/sgml/xfunc.sgml#L1481)
- [brin/README](../raw/postgres-12/src/backend/access/brin/README#L1-L23)
- [brin.c:119](../raw/postgres-12/src/backend/access/brin/brin.c#L119)
- [reloptions.c:177-186](../raw/postgres-12/src/backend/access/common/reloptions.c#L177-L186)
- [gin/README](../raw/postgres-12/src/backend/access/gin/README#L8-L12)
- [ginfast.c:448-461](../raw/postgres-12/src/backend/access/gin/ginfast.c#L448-L461)
- [gininsert.c#buildFreshLeafTuple](../raw/postgres-12/src/backend/access/gin/gininsert.c#L130-L166)
- [ginpostinglist.c](../raw/postgres-12/src/backend/access/gin/ginpostinglist.c#L31-L35)
- [ginutil.c:71](../raw/postgres-12/src/backend/access/gin/ginutil.c#L71)
- [gist.c:77](../raw/postgres-12/src/backend/access/gist/gist.c#L77)
- [gistbuild.c#GistBufferingMode](../raw/postgres-12/src/backend/access/gist/gistbuild.c#L43-L54)
- [hash/README](../raw/postgres-12/src/backend/access/hash/README#L13-L42)
- [hash.c#hashhandler](../raw/postgres-12/src/backend/access/hash/hash.c#L58-L68)
- [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-12/src/backend/access/hash/hashinsert.c#L338)
- [README.HOT](../raw/postgres-12/src/backend/access/heap/README.HOT#L34-L38)
- [README.tuplock](../raw/postgres-12/src/backend/access/heap/README.tuplock#L1-L13)
- [heapam.c#heap_prepare_freeze_tuple](../raw/postgres-12/src/backend/access/heap/heapam.c#L6085-L6118)
- [heapam_handler.c#heap_tableam_handler](../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2666)
- [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L962)
- [pruneheap.c#heap_page_prune_opt](../raw/postgres-12/src/backend/access/heap/pruneheap.c#L84-L140)
- [vacuumlazy.c#lazy_truncate_heap](../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1897-L1926)
- [visibilitymap.c#NOTES](../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L22-L50)
- [amapi.c#GetIndexAmRoutine](../raw/postgres-12/src/backend/access/index/amapi.c#L32-L47)
- [indexam.c#index_getnext_slot](../raw/postgres-12/src/backend/access/index/indexam.c#L607)
- [nbtree/Makefile:15-16](../raw/postgres-12/src/backend/access/nbtree/Makefile#L15-L16)
- [nbtree/README](../raw/postgres-12/src/backend/access/nbtree/README#L1-L24)
- [nbtinsert.c:753-759](../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L753-L759)
- [nbtpage.c#_bt_page_recyclable](../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L941-L963)
- [nbtree.c#bthandler](../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L106-L108)
- [nbtsort.c:728](../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L728)
- [nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-12/src/backend/access/nbtree/nbtsplitloc.c#L97-L127)
- [spgist/README](../raw/postgres-12/src/backend/access/spgist/README#L1-L25)
- [spgutils.c:58](../raw/postgres-12/src/backend/access/spgist/spgutils.c#L58)
- [transam/README](../raw/postgres-12/src/backend/access/transam/README#L411-L413)
- [multixact.c](../raw/postgres-12/src/backend/access/transam/multixact.c#L5-L16)
- [parallel.c:294](../raw/postgres-12/src/backend/access/transam/parallel.c#L294)
- [slru.c](../raw/postgres-12/src/backend/access/transam/slru.c#L3-L22)
- [timeline.c](../raw/postgres-12/src/backend/access/transam/timeline.c#L4-L7)
- [transam.c#TransactionIdPrecedes](../raw/postgres-12/src/backend/access/transam/transam.c#L300)
- [twophase.c](../raw/postgres-12/src/backend/access/transam/twophase.c#L24-L26)
- [varsup.c:76](../raw/postgres-12/src/backend/access/transam/varsup.c#L76)
- [xlog.c#CreateCheckPoint](../raw/postgres-12/src/backend/access/transam/xlog.c#L8810-L8819)
- [xloginsert.c#XLogRecordAssemble](../raw/postgres-12/src/backend/access/transam/xloginsert.c#L539-L555)
- [bootparse.y:1-5](../raw/postgres-12/src/backend/bootstrap/bootparse.y#L1-L5)
- [catalog.c#GetNewOidWithIndex](../raw/postgres-12/src/backend/catalog/catalog.c#L330)
- [genbki.pl:1-8](../raw/postgres-12/src/backend/catalog/genbki.pl#L1-L8)
- [heap.c#StorePartitionBound](../raw/postgres-12/src/backend/catalog/heap.c#L3726-L3761)
- [index.c#index_concurrently_swap](../raw/postgres-12/src/backend/catalog/index.c#L1441-L1448)
- [storage.c#RelationTruncate](../raw/postgres-12/src/backend/catalog/storage.c#L230)
- [system_views.sql:567](../raw/postgres-12/src/backend/catalog/system_views.sql#L567)
- [analyze.c#do_analyze_rel](../raw/postgres-12/src/backend/commands/analyze.c#L295)
- [cluster.c#cluster_rel](../raw/postgres-12/src/backend/commands/cluster.c#L248-L262)
- [comment.c#CreateComments](../raw/postgres-12/src/backend/commands/comment.c#L142-L156)
- [explain.c#ExplainQuery](../raw/postgres-12/src/backend/commands/explain.c#L159-L201)
- [extension.c#ExtensionControlFile](../raw/postgres-12/src/backend/commands/extension.c#L76-L89)
- [functioncmds.c:1030](../raw/postgres-12/src/backend/commands/functioncmds.c#L1030)
- [indexcmds.c:563](../raw/postgres-12/src/backend/commands/indexcmds.c#L563)
- [prepare.c#prepared_queries](../raw/postgres-12/src/backend/commands/prepare.c#L46)
- [publicationcmds.c#CreatePublication](../raw/postgres-12/src/backend/commands/publicationcmds.c#L140)
- [subscriptioncmds.c#CreateSubscription](../raw/postgres-12/src/backend/commands/subscriptioncmds.c#L315)
- [tablecmds.c#ATExecValidateConstraint](../raw/postgres-12/src/backend/commands/tablecmds.c#L9216)
- [vacuum.c:1680-1681](../raw/postgres-12/src/backend/commands/vacuum.c#L1680-L1681)
- [executor/Makefile:24-25](../raw/postgres-12/src/backend/executor/Makefile#L24-L25)
- [execMain.c header](../raw/postgres-12/src/backend/executor/execMain.c#L1-L28)
- [execPartition.c#ExecFindMatchingSubPlans](../raw/postgres-12/src/backend/executor/execPartition.c#L1991)
- [execProcnode.c#ExecInitNode](../raw/postgres-12/src/backend/executor/execProcnode.c#L139)
- [nodeBitmapHeapscan.c](../raw/postgres-12/src/backend/executor/nodeBitmapHeapscan.c#L1-L17)
- [nodeGather.c header](../raw/postgres-12/src/backend/executor/nodeGather.c#L9-L15)
- [nodeHash.c:704](../raw/postgres-12/src/backend/executor/nodeHash.c#L704)
- [nodeIndexonlyscan.c](../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L169)
- [nodeIndexscan.c#IndexNext](../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L81)
- [nodeSeqscan.c#SeqNext](../raw/postgres-12/src/backend/executor/nodeSeqscan.c#L50)
- [nodeSubplan.c#ExecSetParamPlan](../raw/postgres-12/src/backend/executor/nodeSubplan.c#L1051)
- [spi.c#SPI_connect](../raw/postgres-12/src/backend/executor/spi.c#L89)
- [outfuncs.c#nodeToString](../raw/postgres-12/src/backend/nodes/outfuncs.c#L4301)
- [tidbitmap.c](../raw/postgres-12/src/backend/nodes/tidbitmap.c#L3-L30)
- [clausesel.c#clauselist_selectivity](../raw/postgres-12/src/backend/optimizer/path/clausesel.c#L60-L100)
- [costsize.c:712-714](../raw/postgres-12/src/backend/optimizer/path/costsize.c#L712-L714)
- [indxpath.c#check_index_predicates](../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L3414)
- [joinrels.c#try_partitionwise_join](../raw/postgres-12/src/backend/optimizer/path/joinrels.c#L1320-L1336)
- [createplan.c#order_qual_clauses](../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L4919-L4929)
- [planner.c#planner](../raw/postgres-12/src/backend/optimizer/plan/planner.c#L268-L273)
- [subselect.c#SS_process_ctes](../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L882-L899)
- [prepjointree.c:633](../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L633)
- [clauses.c#inline_function](../raw/postgres-12/src/backend/optimizer/util/clauses.c#L4434-L4440)
- [inherit.c#expand_inherited_rtentry](../raw/postgres-12/src/backend/optimizer/util/inherit.c#L79)
- [pathnode.c#add_path](../raw/postgres-12/src/backend/optimizer/util/pathnode.c#L368-L380)
- [plancat.c#relation_excluded_by_constraints](../raw/postgres-12/src/backend/optimizer/util/plancat.c#L1380-L1392)
- [relnode.c#build_joinrel_partition_info](../raw/postgres-12/src/backend/optimizer/util/relnode.c#L1613-L1625)
- [restrictinfo.c:130](../raw/postgres-12/src/backend/optimizer/util/restrictinfo.c#L130)
- [parser/Makefile:30-52](../raw/postgres-12/src/backend/parser/Makefile#L30-L52)
- [parser/README:1-14](../raw/postgres-12/src/backend/parser/README#L1-L14)
- [gram.y:25-27](../raw/postgres-12/src/backend/parser/gram.y#L25-L27)
- [parser.c#raw_parser](../raw/postgres-12/src/backend/parser/parser.c#L36)
- [partprune.c header](../raw/postgres-12/src/backend/partitioning/partprune.c#L3-L18)
- [sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-12/src/backend/port/sysv_shmem.c#L539-L570)
- [autovacuum.c:5-27](../raw/postgres-12/src/backend/postmaster/autovacuum.c#L5-L27)
- [bgworker.c#RegisterBackgroundWorker](../raw/postgres-12/src/backend/postmaster/bgworker.c#L840-L866)
- [bgwriter.c](../raw/postgres-12/src/backend/postmaster/bgwriter.c#L5-L13)
- [pgstat.c header](../raw/postgres-12/src/backend/postmaster/pgstat.c#L1-L16)
- [postmaster.c#BackendStartup](../raw/postgres-12/src/backend/postmaster/postmaster.c#L4059-L4075)
- [logical/Makefile#OBJS](../raw/postgres-12/src/backend/replication/logical/Makefile#L17-L18)
- [decode.c header](../raw/postgres-12/src/backend/replication/logical/decode.c#L3-L8)
- [launcher.c#logicalrep_worker_launch](../raw/postgres-12/src/backend/replication/logical/launcher.c#L424-L431)
- [logical.c#CheckLogicalDecodingRequirements](../raw/postgres-12/src/backend/replication/logical/logical.c#L78-L113)
- [origin.c header](../raw/postgres-12/src/backend/replication/logical/origin.c#L13-L45)
- [reorderbuffer.c#max_changes_in_memory](../raw/postgres-12/src/backend/replication/logical/reorderbuffer.c#L153-L165)
- [worker.c header](../raw/postgres-12/src/backend/replication/logical/worker.c#L10-L16)
- [pgoutput.c header](../raw/postgres-12/src/backend/replication/pgoutput/pgoutput.c#L3-L4)
- [slot.c header](../raw/postgres-12/src/backend/replication/slot.c#L13-L27)
- [syncrep.c#SyncRepWaitForLSN](../raw/postgres-12/src/backend/replication/syncrep.c#L146)
- [walreceiver.c](../raw/postgres-12/src/backend/replication/walreceiver.c#L5-L7)
- [walsender.c](../raw/postgres-12/src/backend/replication/walsender.c#L5-L9)
- [rewriteHandler.c#QueryRewrite](../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L3914-L3924)
- [storage/Makefile#SUBDIRS](../raw/postgres-12/src/backend/storage/Makefile#L11)
- [buffer/README#clock-sweep](../raw/postgres-12/src/backend/storage/buffer/README#L170-L200)
- [bufmgr.c#entry-points](../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L15-L30)
- [freelist.c#ClockSweepTick](../raw/postgres-12/src/backend/storage/buffer/freelist.c#L113)
- [fd.c:333](../raw/postgres-12/src/backend/storage/file/fd.c#L333)
- [freespace.c#NOTES](../raw/postgres-12/src/backend/storage/freespace/freespace.c#L14-L41)
- [dsm.c header](../raw/postgres-12/src/backend/storage/ipc/dsm.c#L3-L4)
- [ipci.c#RequestAddinShmemSpace](../raw/postgres-12/src/backend/storage/ipc/ipci.c#L58-L70)
- [procarray.c#GetSnapshotData](../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1505)
- [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-12/src/backend/storage/ipc/standby.c#L294)
- [lmgr/README:349](../raw/postgres-12/src/backend/storage/lmgr/README#L349)
- [deadlock.c#DeadLockCheck](../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L217)
- [lmgr.c#LockRelationOid](../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L108-L116)
- [lock.c#LockConflicts](../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65)
- [lwlock.c#LWLockAcquire](../raw/postgres-12/src/backend/storage/lmgr/lwlock.c#L1123)
- [bufpage.c:301](../raw/postgres-12/src/backend/storage/page/bufpage.c#L301)
- [md.c](../raw/postgres-12/src/backend/storage/smgr/md.c#L44-L50)
- [smgr.c#f_smgr](../raw/postgres-12/src/backend/storage/smgr/smgr.c#L40-L64)
- [sync.c:236](../raw/postgres-12/src/backend/storage/sync/sync.c#L236)
- [postgres.c#PostgresMain](../raw/postgres-12/src/backend/tcop/postgres.c#L3706-L3709)
- [pquery.c#PortalRun](../raw/postgres-12/src/backend/tcop/pquery.c#L686)
- [utility.c:972-973](../raw/postgres-12/src/backend/tcop/utility.c#L972-L973)
- [Gen_fmgrtab.pl:1-6](../raw/postgres-12/src/backend/utils/Gen_fmgrtab.pl#L1-L6)
- [dbsize.c#pg_relation_size](../raw/postgres-12/src/backend/utils/adt/dbsize.c#L311)
- [pseudotypes.c#pg_node_tree_in](../raw/postgres-12/src/backend/utils/adt/pseudotypes.c#L266-L284)
- [ri_triggers.c#RI_FKey_check](../raw/postgres-12/src/backend/utils/adt/ri_triggers.c#L234)
- [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L8945-L8993)
- [selfuncs.c:6105-6116](../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6105-L6116)
- [inval.c#AcceptInvalidationMessages](../raw/postgres-12/src/backend/utils/cache/inval.c#L681)
- [plancache.c#choose_custom_plan](../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016-L1063)
- [relcache.c:1-22](../raw/postgres-12/src/backend/utils/cache/relcache.c#L1-L22)
- [spccache.c:182](../raw/postgres-12/src/backend/utils/cache/spccache.c#L182)
- [syscache.c:13-17](../raw/postgres-12/src/backend/utils/cache/syscache.c#L13-L17)
- [elog.c#errstart](../raw/postgres-12/src/backend/utils/error/elog.c#L246-L249)
- [dfmgr.c:285-287](../raw/postgres-12/src/backend/utils/fmgr/dfmgr.c#L285-L287)
- [fmgr.c#fmgr_info](../raw/postgres-12/src/backend/utils/fmgr/fmgr.c#L124)
- [globals.c:131](../raw/postgres-12/src/backend/utils/init/globals.c#L131)
- [miscinit.c#process_shared_preload_libraries](../raw/postgres-12/src/backend/utils/init/miscinit.c#L1583-L1590)
- [guc.c#autovacuum](../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433)
- [rls.c#check_enable_rls](../raw/postgres-12/src/backend/utils/misc/rls.c#L52)
- [mmgr/README:33-40](../raw/postgres-12/src/backend/utils/mmgr/README#L33-L40)
- [dsa.c header](../raw/postgres-12/src/backend/utils/mmgr/dsa.c#L3-L4)
- [tuplesort.c header](../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L10-L20)
- [initdb.c:144](../raw/postgres-12/src/bin/initdb/initdb.c#L144)
- [pg_checksums.c header](../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L1-L5)
- [pg_upgrade.c:14-23](../raw/postgres-12/src/bin/pg_upgrade/pg_upgrade.c#L14-L23)
- [pg_upgrade.h#transferMode](../raw/postgres-12/src/bin/pg_upgrade/pg_upgrade.h#L235-L240)
- [amapi.h#IndexAmRoutine](../raw/postgres-12/src/include/access/amapi.h#L163-L233)
- [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-12/src/include/access/brin.h#L39)
- [brin_page.h:75](../raw/postgres-12/src/include/access/brin_page.h#L75)
- [genam.h#IndexBulkDeleteResult](../raw/postgres-12/src/include/access/genam.h#L72-L81)
- [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-12/src/include/access/gin_private.h#L35-L39)
- [ginblock.h:51](../raw/postgres-12/src/include/access/ginblock.h#L51)
- [gist.h:29-37](../raw/postgres-12/src/include/access/gist.h#L29-L37)
- [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-12/src/include/access/gist_private.h#L263)
- [hash.h#HASH_METAPAGE](../raw/postgres-12/src/include/access/hash.h#L196)
- [heapam.h#HTSV_Result](../raw/postgres-12/src/include/access/heapam.h#L85-L93)
- [htup_details.h:205](../raw/postgres-12/src/include/access/htup_details.h#L205)
- [itup.h#IndexTupleData](../raw/postgres-12/src/include/access/itup.h#L35-L50)
- [multixact.h:32-33](../raw/postgres-12/src/include/access/multixact.h#L32-L33)
- [nbtree.h#BTMetaPageData](../raw/postgres-12/src/include/access/nbtree.h#L97-L110)
- [spgist_private.h:26](../raw/postgres-12/src/include/access/spgist_private.h#L26)
- [transam.h](../raw/postgres-12/src/include/access/transam.h#L20-L34)
- [tuptoaster.h#MaximumBytesPerTuple](../raw/postgres-12/src/include/access/tuptoaster.h#L27-L33)
- [visibilitymap.h](../raw/postgres-12/src/include/access/visibilitymap.h#L26-L27)
- [xact.h#SyncCommitLevel](../raw/postgres-12/src/include/access/xact.h#L68-L76)
- [xlog.h:192](../raw/postgres-12/src/include/access/xlog.h#L192)
- [xlog_internal.h#XLogFileName](../raw/postgres-12/src/include/access/xlog_internal.h#L155-L158)
- [xlogdefs.h#XLogRecPtr](../raw/postgres-12/src/include/access/xlogdefs.h#L17-L21)
- [c.h:678-685](../raw/postgres-12/src/include/c.h#L678-L685)
- [genbki.h:23](../raw/postgres-12/src/include/catalog/genbki.h#L23)
- [index.h#DEFAULT_INDEX_TYPE](../raw/postgres-12/src/include/catalog/index.h#L21)
- [pg_am.dat](../raw/postgres-12/src/include/catalog/pg_am.dat#L14-L35)
- [pg_am.h#FormData_pg_am](../raw/postgres-12/src/include/catalog/pg_am.h#L29-L41)
- [pg_attribute.h#FormData_pg_attribute](../raw/postgres-12/src/include/catalog/pg_attribute.h#L37)
- [pg_authid.h:41](../raw/postgres-12/src/include/catalog/pg_authid.h#L41)
- [pg_cast.dat:41](../raw/postgres-12/src/include/catalog/pg_cast.dat#L41)
- [pg_cast.h#FormData_pg_cast](../raw/postgres-12/src/include/catalog/pg_cast.h#L30-L49)
- [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-12/src/include/catalog/pg_class.h#L162)
- [pg_collation.h#FormData_pg_collation](../raw/postgres-12/src/include/catalog/pg_collation.h#L29-L44)
- [pg_constraint.h:54](../raw/postgres-12/src/include/catalog/pg_constraint.h#L54)
- [pg_control.h:66-68](../raw/postgres-12/src/include/catalog/pg_control.h#L66-L68)
- [pg_description.h:10-21](../raw/postgres-12/src/include/catalog/pg_description.h#L10-L21)
- [pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-12/src/include/catalog/pg_event_trigger.h#L26-L42)
- [pg_index.h:34](../raw/postgres-12/src/include/catalog/pg_index.h#L34)
- [pg_inherits.h#FormData_pg_inherits](../raw/postgres-12/src/include/catalog/pg_inherits.h#L32-L37)
- [pg_opclass.h#FormData_pg_opclass](../raw/postgres-12/src/include/catalog/pg_opclass.h#L49-L76)
- [pg_opfamily.h#FormData_pg_opfamily](../raw/postgres-12/src/include/catalog/pg_opfamily.h#L29-L44)
- [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-12/src/include/catalog/pg_partitioned_table.h#L30-L54)
- [pg_policy.h#FormData_pg_policy](../raw/postgres-12/src/include/catalog/pg_policy.h#L29-L44)
- [pg_proc.dat:6883-6887](../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6887)
- [pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-12/src/include/catalog/pg_proc.h#L154-L156)
- [pg_publication.h#FormData_pg_publication](../raw/postgres-12/src/include/catalog/pg_publication.h#L29-L55)
- [pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-12/src/include/catalog/pg_statistic.h#L201-L210)
- [pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-12/src/include/catalog/pg_statistic_ext.h#L67-L69)
- [pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../raw/postgres-12/src/include/catalog/pg_statistic_ext_data.h#L31-L43)
- [pg_subscription.h#FormData_pg_subscription](../raw/postgres-12/src/include/catalog/pg_subscription.h#L39-L64)
- [pg_subscription_rel.h](../raw/postgres-12/src/include/catalog/pg_subscription_rel.h#L45-L54)
- [pg_tablespace.dat](../raw/postgres-12/src/include/catalog/pg_tablespace.dat#L15-L20)
- [pg_type.h#typalign](../raw/postgres-12/src/include/catalog/pg_type.h#L146-L170)
- [event_trigger.h#AT_REWRITE](../raw/postgres-12/src/include/commands/event_trigger.h#L31-L33)
- [vacuum.h#VacOptTernaryValue](../raw/postgres-12/src/include/commands/vacuum.h#L157-L162)
- [relpath.h#ForkNumber](../raw/postgres-12/src/include/common/relpath.h#L40-L53)
- [executor.h#ExecQual](../raw/postgres-12/src/include/executor/executor.h#L364)
- [instrument.h#BufferUsage](../raw/postgres-12/src/include/executor/instrument.h#L19-L33)
- [fmgr.h:1-8](../raw/postgres-12/src/include/fmgr.h#L1-L8)
- [fdwapi.h#FdwRoutine](../raw/postgres-12/src/include/foreign/fdwapi.h#L183-L249)
- [miscadmin.h#START_CRIT_SECTION](../raw/postgres-12/src/include/miscadmin.h#L132-L138)
- [execnodes.h#EState](../raw/postgres-12/src/include/nodes/execnodes.h#L496)
- [lockoptions.h#LockTupleMode](../raw/postgres-12/src/include/nodes/lockoptions.h#L46-L59)
- [nodes.h:658](../raw/postgres-12/src/include/nodes/nodes.h#L658)
- [parsenodes.h#CTEMaterialize](../raw/postgres-12/src/include/nodes/parsenodes.h#L1417-L1422)
- [pathnodes.h#Path](../raw/postgres-12/src/include/nodes/pathnodes.h#L1121-L1122)
- [plannodes.h#PlannedStmt](../raw/postgres-12/src/include/nodes/plannodes.h#L42)
- [primnodes.h#SubPlan](../raw/postgres-12/src/include/nodes/primnodes.h#L649-L720)
- [cost.h:32](../raw/postgres-12/src/include/optimizer/cost.h#L32)
- [kwlist.h:333-334](../raw/postgres-12/src/include/parser/kwlist.h#L333-L334)
- [parsetree.h#rt_fetch](../raw/postgres-12/src/include/parser/parsetree.h#L31-L32)
- [pgstat.h#PgBackendStatus](../raw/postgres-12/src/include/pgstat.h#L1027)
- [postgres.h#Datum](../raw/postgres-12/src/include/postgres.h#L357-L367)
- [postgres_ext.h:27-31](../raw/postgres-12/src/include/postgres_ext.h#L27-L31)
- [bgworker.h#BackgroundWorker](../raw/postgres-12/src/include/postmaster/bgworker.h#L88-L100)
- [output_plugin.h#OutputPluginCallbacks](../raw/postgres-12/src/include/replication/output_plugin.h#L105-L115)
- [slot.h#ReplicationSlotPersistentData](../raw/postgres-12/src/include/replication/slot.h#L43-L85)
- [block.h#BlockNumber](../raw/postgres-12/src/include/storage/block.h#L17-L33)
- [buf_internals.h#BufferDesc](../raw/postgres-12/src/include/storage/buf_internals.h#L178-L190)
- [bufpage.h#NOTES](../raw/postgres-12/src/include/storage/bufpage.h#L54-L64)
- [ipc.h:22](../raw/postgres-12/src/include/storage/ipc.h#L22)
- [itemid.h#ItemIdData](../raw/postgres-12/src/include/storage/itemid.h#L25-L41)
- [itemptr.h#ItemPointerData](../raw/postgres-12/src/include/storage/itemptr.h#L36-L45)
- [lock.h#LOCKTAG](../raw/postgres-12/src/include/storage/lock.h#L164)
- [lockdefs.h#AccessExclusiveLock](../raw/postgres-12/src/include/storage/lockdefs.h#L45-L46)
- [lwlock.h#LWLock](../raw/postgres-12/src/include/storage/lwlock.h#L32-L40)
- [relfilenode.h#RelFileNode](../raw/postgres-12/src/include/storage/relfilenode.h#L50-L62)
- [sinval.h#SharedInvalidationMessage](../raw/postgres-12/src/include/storage/sinval.h#L113-L122)
- [utility.h:29-34](../raw/postgres-12/src/include/tcop/utility.h#L29-L34)
- [elog.h:20-53](../raw/postgres-12/src/include/utils/elog.h#L20-L53)
- [guc.h#GucContext](../raw/postgres-12/src/include/utils/guc.h#L68-L77)
- [plancache.h#PlanCacheMode](../raw/postgres-12/src/include/utils/plancache.h#L26-L32)
- [portal.h header](../raw/postgres-12/src/include/utils/portal.h#L3-L9)
- [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-12/src/include/utils/rel.h#L278-L279)
- [selfuncs.h:33-36](../raw/postgres-12/src/include/utils/selfuncs.h#L33-L36)
- [snapshot.h#SNAPSHOT_MVCC](../raw/postgres-12/src/include/utils/snapshot.h#L50)
- [syscache.h:84](../raw/postgres-12/src/include/utils/syscache.h#L84)
- [pl_handler.c#plpgsql_inline_handler](../raw/postgres-12/src/pl/plpgsql/src/pl_handler.c#L300-L314)
- [plpgsql--1.0.sql](../raw/postgres-12/src/pl/plpgsql/src/plpgsql--1.0.sql#L1-L9)
- [isolation/README](../raw/postgres-12/src/test/isolation/README#L3-L12)
- [modules/Makefile:7-24](../raw/postgres-12/src/test/modules/Makefile#L7-L24)
- [PostgresNode.pm](../raw/postgres-12/src/test/perl/PostgresNode.pm#L1-L30)
- [parallel_schedule](../raw/postgres-12/src/test/regress/parallel_schedule#L13-L16)
- [pg_regress.c header](../raw/postgres-12/src/test/regress/pg_regress.c#L3)

**PostgreSQL 14** (318 files):

- [configure.ac#blocksize](../raw/postgres-14/configure.ac#L255-L267)
- [contrib/Makefile:32-38](../raw/postgres-14/contrib/Makefile#L32-L38)
- [README:1-9](../raw/postgres-14/contrib/README#L1-L9)
- [verify_heapam.c:212](../raw/postgres-14/contrib/amcheck/verify_heapam.c#L212)
- [verify_nbtree.c#bt_index_check](../raw/postgres-14/contrib/amcheck/verify_nbtree.c#L206-L229)
- [btree_gin.control](../raw/postgres-14/contrib/btree_gin/btree_gin.control#L1-L6)
- [btree_gist.control](../raw/postgres-14/contrib/btree_gist/btree_gist.control#L1-L6)
- [ginfuncs.c header](../raw/postgres-14/contrib/pageinspect/ginfuncs.c#L1-L3)
- [heapfuncs.c header](../raw/postgres-14/contrib/pageinspect/heapfuncs.c#L1-L4)
- [pageinspect--1.5--1.6.sql](../raw/postgres-14/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88-L90)
- [pageinspect.control](../raw/postgres-14/contrib/pageinspect/pageinspect.control#L1-L5)
- [rawpage.c#get_raw_page_internal](../raw/postgres-14/contrib/pageinspect/rawpage.c#L150-L153)
- [pg_freespacemap--1.1.sql](../raw/postgres-14/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L25)
- [pg_freespacemap.c#pg_freespace](../raw/postgres-14/contrib/pg_freespacemap/pg_freespacemap.c#L25-L49)
- [pg_freespacemap.control](../raw/postgres-14/contrib/pg_freespacemap/pg_freespacemap.control#L1-L2)
- [pg_stat_statements.c:467](../raw/postgres-14/contrib/pg_stat_statements/pg_stat_statements.c#L467)
- [pgstatapprox.c#pgstattuple_approx](../raw/postgres-14/contrib/pgstattuple/pgstatapprox.c#L227)
- [pgstatindex.c#pgstatindex_impl](../raw/postgres-14/contrib/pgstattuple/pgstatindex.c#L224-L250)
- [pgstattuple--1.4--1.5.sql](../raw/postgres-14/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L37)
- [pgstattuple.c#pgstat_relation](../raw/postgres-14/contrib/pgstattuple/pgstattuple.c#L270-L292)
- [pgstattuple.control](../raw/postgres-14/contrib/pgstattuple/pgstattuple.control#L1-L2)
- [auto-explain.sgml:23](../raw/postgres-14/doc/src/sgml/auto-explain.sgml#L23)
- [bki.sgml:40-61](../raw/postgres-14/doc/src/sgml/bki.sgml#L40-L61)
- [catalogs.sgml#view-pg-settings](../raw/postgres-14/doc/src/sgml/catalogs.sgml#L12320-L12428)
- [config.sgml#guc-huge-pages](../raw/postgres-14/doc/src/sgml/config.sgml#L1685)
- [contrib.sgml:7-15](../raw/postgres-14/doc/src/sgml/contrib.sgml#L7-L15)
- [event-trigger.sgml:28-35](../raw/postgres-14/doc/src/sgml/event-trigger.sgml#L28-L35)
- [extend.sgml:532-535](../raw/postgres-14/doc/src/sgml/extend.sgml#L532-L535)
- [gin.sgml](../raw/postgres-14/doc/src/sgml/gin.sgml#L503-L530)
- [gist.sgml#gist-intro](../raw/postgres-14/doc/src/sgml/gist.sgml#L11-L25)
- [glossary.sgml#glossary-backend](../raw/postgres-14/doc/src/sgml/glossary.sgml#L126-L140)
- [high-availability.sgml#hot-standby-conflict](../raw/postgres-14/doc/src/sgml/high-availability.sgml#L1736)
- [indexam.sgml#intro](../raw/postgres-14/doc/src/sgml/indexam.sgml#L39-L40)
- [indices.sgml#indexes-expressional](../raw/postgres-14/doc/src/sgml/indices.sgml#L730-L760)
- [logical-replication.sgml#logical-replication-conflicts](../raw/postgres-14/doc/src/sgml/logical-replication.sgml#L320-L330)
- [maintenance.sgml#routine-reindex](../raw/postgres-14/doc/src/sgml/maintenance.sgml#L1023-L1025)
- [monitoring.sgml#monitoring-stats](../raw/postgres-14/doc/src/sgml/monitoring.sgml#L138-L154)
- [mvcc.sgml#SHARE UPDATE EXCLUSIVE](../raw/postgres-14/doc/src/sgml/mvcc.sgml#L939-L956)
- [plpgsql.sgml:4](../raw/postgres-14/doc/src/sgml/plpgsql.sgml#L4)
- [ref/alter_table.sgml:421-423](../raw/postgres-14/doc/src/sgml/ref/alter_table.sgml#L421-L423)
- [ref/cluster.sgml#description](../raw/postgres-14/doc/src/sgml/ref/cluster.sgml#L49-L52)
- [ref/create_index.sgml](../raw/postgres-14/doc/src/sgml/ref/create_index.sgml#L632-L636)
- [ref/create_table.sgml#fillfactor](../raw/postgres-14/doc/src/sgml/ref/create_table.sgml#L1397-L1412)
- [ref/explain.sgml#BUFFERS](../raw/postgres-14/doc/src/sgml/ref/explain.sgml#L171-L194)
- [pgupgrade.sgml:39-58](../raw/postgres-14/doc/src/sgml/ref/pgupgrade.sgml#L39-L58)
- [ref/reindex.sgml#bloated](../raw/postgres-14/doc/src/sgml/ref/reindex.sgml#L53-L62)
- [ref/truncate.sgml#description](../raw/postgres-14/doc/src/sgml/ref/truncate.sgml#L33)
- [ref/vacuum.sgml:35](../raw/postgres-14/doc/src/sgml/ref/vacuum.sgml#L35)
- [regress.sgml](../raw/postgres-14/doc/src/sgml/regress.sgml#L464-L469)
- [spi.sgml](../raw/postgres-14/doc/src/sgml/spi.sgml#L10-L27)
- [wal.sgml](../raw/postgres-14/doc/src/sgml/wal.sgml#L498-L509)
- [xfunc.sgml#xfunc-volatility](../raw/postgres-14/doc/src/sgml/xfunc.sgml#L1607)
- [brin/README](../raw/postgres-14/src/backend/access/brin/README#L1-L23)
- [brin.c:129](../raw/postgres-14/src/backend/access/brin/brin.c#L129)
- [reloptions.c#fillfactor](../raw/postgres-14/src/backend/access/common/reloptions.c#L168-L177)
- [gin/README](../raw/postgres-14/src/backend/access/gin/README#L8-L26)
- [ginfast.c:457-470](../raw/postgres-14/src/backend/access/gin/ginfast.c#L457-L470)
- [gininsert.c#buildFreshLeafTuple](../raw/postgres-14/src/backend/access/gin/gininsert.c#L129-L169)
- [ginpostinglist.c](../raw/postgres-14/src/backend/access/gin/ginpostinglist.c#L31-L35)
- [ginutil.c:77](../raw/postgres-14/src/backend/access/gin/ginutil.c#L77)
- [gist.c#gisthandler](../raw/postgres-14/src/backend/access/gist/gist.c#L67-L70)
- [gistbuild.c header](../raw/postgres-14/src/backend/access/gist/gistbuild.c#L15-L16)
- [hash/README](../raw/postgres-14/src/backend/access/hash/README#L13-L15)
- [hash.c#hashhandler](../raw/postgres-14/src/backend/access/hash/hash.c#L63-L67)
- [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-14/src/backend/access/hash/hashinsert.c#L338)
- [README.HOT](../raw/postgres-14/src/backend/access/heap/README.HOT#L30-L38)
- [README.tuplock](../raw/postgres-14/src/backend/access/heap/README.tuplock#L1-L13)
- [heapam.c#heap_prepare_freeze_tuple](../raw/postgres-14/src/backend/access/heap/heapam.c#L7009-L7012)
- [heapam_handler.c#heap_tableam_handler](../raw/postgres-14/src/backend/access/heap/heapam_handler.c#L2605-L2609)
- [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-14/src/backend/access/heap/heapam_visibility.c#L959-L1144)
- [pruneheap.c#heap_page_prune_opt](../raw/postgres-14/src/backend/access/heap/pruneheap.c#L121-L203)
- [vacuumlazy.c:102-104](../raw/postgres-14/src/backend/access/heap/vacuumlazy.c#L102-L104)
- [visibilitymap.c#NOTES](../raw/postgres-14/src/backend/access/heap/visibilitymap.c#L23-L50)
- [amapi.c#GetIndexAmRoutine](../raw/postgres-14/src/backend/access/index/amapi.c#L33-L46)
- [indexam.c#index_getnext_slot](../raw/postgres-14/src/backend/access/index/indexam.c#L658)
- [nbtree/README](../raw/postgres-14/src/backend/access/nbtree/README#L528-L561)
- [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-14/src/backend/access/nbtree/nbtdedup.c#L305)
- [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-14/src/backend/access/nbtree/nbtinsert.c#L2746-L2771)
- [nbtpage.c#_bt_upgrademetapage](../raw/postgres-14/src/backend/access/nbtree/nbtpage.c#L109-L133)
- [nbtree.c#bthandler](../raw/postgres-14/src/backend/access/nbtree/nbtree.c#L95-L144)
- [nbtsort.c:375-378](../raw/postgres-14/src/backend/access/nbtree/nbtsort.c#L375-L378)
- [nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-14/src/backend/access/nbtree/nbtsplitloc.c#L95-L98)
- [nbtutils.c#_bt_allequalimage](../raw/postgres-14/src/backend/access/nbtree/nbtutils.c#L2712-L2771)
- [spgist/README](../raw/postgres-14/src/backend/access/spgist/README#L1-L27)
- [spgutils.c:63](../raw/postgres-14/src/backend/access/spgist/spgutils.c#L63)
- [tableam.c#table_block_relation_estimate_size](../raw/postgres-14/src/backend/access/table/tableam.c#L678-L748)
- [transam/README](../raw/postgres-14/src/backend/access/transam/README#L420-L422)
- [multixact.c](../raw/postgres-14/src/backend/access/transam/multixact.c#L5-L20)
- [parallel.c:318](../raw/postgres-14/src/backend/access/transam/parallel.c#L318)
- [slru.c](../raw/postgres-14/src/backend/access/transam/slru.c#L3-L10)
- [timeline.c](../raw/postgres-14/src/backend/access/transam/timeline.c#L4-L7)
- [transam.c#TransactionIdPrecedes](../raw/postgres-14/src/backend/access/transam/transam.c#L305-L318)
- [twophase.c](../raw/postgres-14/src/backend/access/transam/twophase.c#L24-L26)
- [varsup.c:78](../raw/postgres-14/src/backend/access/transam/varsup.c#L78)
- [xlog.c#CreateCheckPoint](../raw/postgres-14/src/backend/access/transam/xlog.c#L9440-L9448)
- [xloginsert.c#XLogRecordAssemble](../raw/postgres-14/src/backend/access/transam/xloginsert.c#L547-L563)
- [bootparse.y:4-5](../raw/postgres-14/src/backend/bootstrap/bootparse.y#L4-L5)
- [catalog.c#GetNewOidWithIndex](../raw/postgres-14/src/backend/catalog/catalog.c#L352-L442)
- [genbki.pl:1-7](../raw/postgres-14/src/backend/catalog/genbki.pl#L1-L7)
- [heap.c#StorePartitionBound](../raw/postgres-14/src/backend/catalog/heap.c#L3907-L3942)
- [index.c:1683-1724](../raw/postgres-14/src/backend/catalog/index.c#L1683-L1724)
- [storage.c#RelationTruncate](../raw/postgres-14/src/backend/catalog/storage.c#L277-L431)
- [system_functions.sql:278](../raw/postgres-14/src/backend/catalog/system_functions.sql#L278)
- [system_views.sql:646](../raw/postgres-14/src/backend/catalog/system_views.sql#L646)
- [analyze.c#do_analyze_rel](../raw/postgres-14/src/backend/commands/analyze.c#L290)
- [cluster.c#cluster_rel](../raw/postgres-14/src/backend/commands/cluster.c#L259-L266)
- [comment.c#CreateComments](../raw/postgres-14/src/backend/commands/comment.c#L142-L225)
- [explain.c#ExplainQuery](../raw/postgres-14/src/backend/commands/explain.c#L230-L242)
- [extension.c#ExtensionControlFile](../raw/postgres-14/src/backend/commands/extension.c#L80-L94)
- [functioncmds.c:1165](../raw/postgres-14/src/backend/commands/functioncmds.c#L1165)
- [indexcmds.c:663](../raw/postgres-14/src/backend/commands/indexcmds.c#L663)
- [prepare.c#prepared_queries](../raw/postgres-14/src/backend/commands/prepare.c#L46)
- [publicationcmds.c#CreatePublication](../raw/postgres-14/src/backend/commands/publicationcmds.c#L150)
- [subscriptioncmds.c#CreateSubscription](../raw/postgres-14/src/backend/commands/subscriptioncmds.c#L353)
- [tablecmds.c#ATExecValidateConstraint](../raw/postgres-14/src/backend/commands/tablecmds.c#L10807)
- [vacuum.c:1919-1920](../raw/postgres-14/src/backend/commands/vacuum.c#L1919-L1920)
- [execMain.c header](../raw/postgres-14/src/backend/executor/execMain.c#L1-L28)
- [execPartition.c#ExecFindPartition](../raw/postgres-14/src/backend/executor/execPartition.c#L257)
- [execProcnode.c#ExecInitNode](../raw/postgres-14/src/backend/executor/execProcnode.c#L142)
- [nodeBitmapHeapscan.c](../raw/postgres-14/src/backend/executor/nodeBitmapHeapscan.c#L1-L17)
- [nodeGather.c header](../raw/postgres-14/src/backend/executor/nodeGather.c#L9-L15)
- [nodeHash.c#get_hash_memory_limit](../raw/postgres-14/src/backend/executor/nodeHash.c#L3430-L3449)
- [nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-14/src/backend/executor/nodeIndexonlyscan.c#L164-L172)
- [nodeIndexscan.c#IndexNext](../raw/postgres-14/src/backend/executor/nodeIndexscan.c#L81)
- [nodeMemoize.c header](../raw/postgres-14/src/backend/executor/nodeMemoize.c#L1-L3)
- [nodeSeqscan.c#SeqNext](../raw/postgres-14/src/backend/executor/nodeSeqscan.c#L50)
- [spi.c#SPI_connect](../raw/postgres-14/src/backend/executor/spi.c#L95)
- [outfuncs.c#nodeToString](../raw/postgres-14/src/backend/nodes/outfuncs.c#L4546)
- [read.c#stringToNode](../raw/postgres-14/src/backend/nodes/read.c#L89)
- [tidbitmap.c](../raw/postgres-14/src/backend/nodes/tidbitmap.c#L3-L30)
- [clausesel.c#clauselist_selectivity](../raw/postgres-14/src/backend/optimizer/path/clausesel.c#L102-L110)
- [costsize.c:727-729](../raw/postgres-14/src/backend/optimizer/path/costsize.c#L727-L729)
- [indxpath.c#check_index_predicates](../raw/postgres-14/src/backend/optimizer/path/indxpath.c#L3420)
- [joinpath.c#get_memoize_path](../raw/postgres-14/src/backend/optimizer/path/joinpath.c#L507)
- [joinrels.c#try_partitionwise_join](../raw/postgres-14/src/backend/optimizer/path/joinrels.c#L1358-L1552)
- [createplan.c#order_qual_clauses](../raw/postgres-14/src/backend/optimizer/plan/createplan.c#L5192-L5202)
- [planner.c#planner](../raw/postgres-14/src/backend/optimizer/plan/planner.c#L264-L274)
- [subselect.c#SS_process_ctes](../raw/postgres-14/src/backend/optimizer/plan/subselect.c#L947-L964)
- [prepjointree.c:660](../raw/postgres-14/src/backend/optimizer/prep/prepjointree.c#L660)
- [clauses.c#inline_function](../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4422-L4428)
- [inherit.c#expand_inherited_rtentry](../raw/postgres-14/src/backend/optimizer/util/inherit.c#L84)
- [pathnode.c#add_path](../raw/postgres-14/src/backend/optimizer/util/pathnode.c#L427-L629)
- [plancat.c#relation_excluded_by_constraints](../raw/postgres-14/src/backend/optimizer/util/plancat.c#L1541-L1568)
- [relnode.c#build_joinrel_partition_info](../raw/postgres-14/src/backend/optimizer/util/relnode.c#L1656-L1664)
- [restrictinfo.c:136](../raw/postgres-14/src/backend/optimizer/util/restrictinfo.c#L136)
- [parser/Makefile:51-60](../raw/postgres-14/src/backend/parser/Makefile#L51-L60)
- [parser/README](../raw/postgres-14/src/backend/parser/README#L11-L13)
- [gram.y:25-30](../raw/postgres-14/src/backend/parser/gram.y#L25-L30)
- [parser.c#raw_parser](../raw/postgres-14/src/backend/parser/parser.c#L42-L86)
- [partprune.c header](../raw/postgres-14/src/backend/partitioning/partprune.c#L3-L18)
- [sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-14/src/backend/port/sysv_shmem.c#L578-L597)
- [autovacuum.c:7-27](../raw/postgres-14/src/backend/postmaster/autovacuum.c#L7-L27)
- [bgworker.c#RegisterBackgroundWorker](../raw/postgres-14/src/backend/postmaster/bgworker.c#L889-L905)
- [bgwriter.c:5-15](../raw/postgres-14/src/backend/postmaster/bgwriter.c#L5-L15)
- [pgstat.c header](../raw/postgres-14/src/backend/postmaster/pgstat.c#L1-L17)
- [postmaster.c#BackendStartup](../raw/postgres-14/src/backend/postmaster/postmaster.c#L4216-L4220)
- [logical/Makefile#OBJS](../raw/postgres-14/src/backend/replication/logical/Makefile#L17-L29)
- [decode.c header](../raw/postgres-14/src/backend/replication/logical/decode.c#L3-L8)
- [launcher.c#logicalrep_worker_launch](../raw/postgres-14/src/backend/replication/logical/launcher.c#L393-L405)
- [logical.c#CheckLogicalDecodingRequirements](../raw/postgres-14/src/backend/replication/logical/logical.c#L117-L144)
- [origin.c header](../raw/postgres-14/src/backend/replication/logical/origin.c#L13-L43)
- [reorderbuffer.c#ReorderBufferCheckMemoryLimit](../raw/postgres-14/src/backend/replication/logical/reorderbuffer.c#L3641-L3672)
- [tablesync.c#ReplicationOriginNameForTablesync](../raw/postgres-14/src/backend/replication/logical/tablesync.c#L940-L943)
- [worker.c#ApplyWorkerMain](../raw/postgres-14/src/backend/replication/logical/worker.c#L3094-L3100)
- [pgoutput.c header](../raw/postgres-14/src/backend/replication/pgoutput/pgoutput.c#L3-L4)
- [slot.c header](../raw/postgres-14/src/backend/replication/slot.c#L15-L18)
- [syncrep.c](../raw/postgres-14/src/backend/replication/syncrep.c#L7-L8)
- [walreceiver.c#WalReceiverMain](../raw/postgres-14/src/backend/replication/walreceiver.c#L175)
- [walsender.c#WalSndCheckTimeOut](../raw/postgres-14/src/backend/replication/walsender.c#L2296)
- [rewriteHandler.c#QueryRewrite](../raw/postgres-14/src/backend/rewrite/rewriteHandler.c#L4357-L4367)
- [storage/Makefile#SUBDIRS](../raw/postgres-14/src/backend/storage/Makefile#L11)
- [buffer/README#clock-sweep](../raw/postgres-14/src/backend/storage/buffer/README#L166-L199)
- [bufmgr.c#entry-points](../raw/postgres-14/src/backend/storage/buffer/bufmgr.c#L15-L29)
- [freelist.c#ClockSweepTick](../raw/postgres-14/src/backend/storage/buffer/freelist.c#L113-L169)
- [fd.c:352](../raw/postgres-14/src/backend/storage/file/fd.c#L352)
- [freespace.c#NOTES](../raw/postgres-14/src/backend/storage/freespace/freespace.c#L36-L63)
- [ipci.c#RequestAddinShmemSpace](../raw/postgres-14/src/backend/storage/ipc/ipci.c#L59-L71)
- [procarray.c#GetSnapshotData](../raw/postgres-14/src/backend/storage/ipc/procarray.c#L2251)
- [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-14/src/backend/storage/ipc/standby.c#L444)
- [lmgr/README](../raw/postgres-14/src/backend/storage/lmgr/README#L20-L35)
- [deadlock.c#DeadLockCheck](../raw/postgres-14/src/backend/storage/lmgr/deadlock.c#L217)
- [lmgr.c#LockRelationOid](../raw/postgres-14/src/backend/storage/lmgr/lmgr.c#L109-L117)
- [lock.c#LockConflicts](../raw/postgres-14/src/backend/storage/lmgr/lock.c#L65-L105)
- [lwlock.c#LWLockAcquire](../raw/postgres-14/src/backend/storage/lmgr/lwlock.c#L1206)
- [bufpage.c:315](../raw/postgres-14/src/backend/storage/page/bufpage.c#L315)
- [md.c](../raw/postgres-14/src/backend/storage/smgr/md.c#L45-L51)
- [smgr.c#f_smgr](../raw/postgres-14/src/backend/storage/smgr/smgr.c#L40-L65)
- [sync.c:294](../raw/postgres-14/src/backend/storage/sync/sync.c#L294)
- [postgres.c#PostgresMain](../raw/postgres-14/src/backend/tcop/postgres.c#L4027)
- [pquery.c#PortalRun](../raw/postgres-14/src/backend/tcop/pquery.c#L683-L841)
- [utility.c:1112](../raw/postgres-14/src/backend/tcop/utility.c#L1112)
- [Gen_fmgrtab.pl:1-6](../raw/postgres-14/src/backend/utils/Gen_fmgrtab.pl#L1-L6)
- [backend_progress.c#pgstat_progress_update_param](../raw/postgres-14/src/backend/utils/activity/backend_progress.c#L47)
- [backend_status.c:570](../raw/postgres-14/src/backend/utils/activity/backend_status.c#L570)
- [wait_event.c#pgstat_get_wait_event](../raw/postgres-14/src/backend/utils/activity/wait_event.c#L129)
- [dbsize.c#pg_relation_size](../raw/postgres-14/src/backend/utils/adt/dbsize.c#L311)
- [pseudotypes.c#pg_node_tree](../raw/postgres-14/src/backend/utils/adt/pseudotypes.c#L340)
- [ri_triggers.c#RI_FKey_check](../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L236)
- [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-14/src/backend/utils/adt/ruleutils.c#L9542-L9590)
- [selfuncs.c:6888-6898](../raw/postgres-14/src/backend/utils/adt/selfuncs.c#L6888-L6898)
- [inval.c#AcceptInvalidationMessages](../raw/postgres-14/src/backend/utils/cache/inval.c#L721)
- [plancache.c#choose_custom_plan](../raw/postgres-14/src/backend/utils/cache/plancache.c#L1031-L1078)
- [relcache.c:1-22](../raw/postgres-14/src/backend/utils/cache/relcache.c#L1-L22)
- [spccache.c:181](../raw/postgres-14/src/backend/utils/cache/spccache.c#L181)
- [syscache.c:13-17](../raw/postgres-14/src/backend/utils/cache/syscache.c#L13-L17)
- [elog.c#errstart](../raw/postgres-14/src/backend/utils/error/elog.c#L355-L358)
- [dfmgr.c:285-287](../raw/postgres-14/src/backend/utils/fmgr/dfmgr.c#L285-L287)
- [fmgr.c#fmgr_info](../raw/postgres-14/src/backend/utils/fmgr/fmgr.c#L126-L129)
- [globals.c:135](../raw/postgres-14/src/backend/utils/init/globals.c#L135)
- [miscinit.c#process_shared_preload_libraries](../raw/postgres-14/src/backend/utils/init/miscinit.c#L1716-L1723)
- [misc/Makefile](../raw/postgres-14/src/backend/utils/misc/Makefile#L17-L30)
- [guc.c:1569](../raw/postgres-14/src/backend/utils/misc/guc.c#L1569)
- [queryjumble.c header](../raw/postgres-14/src/backend/utils/misc/queryjumble.c#L1-L12)
- [rls.c#check_enable_rls](../raw/postgres-14/src/backend/utils/misc/rls.c#L52)
- [mmgr/README:37-40](../raw/postgres-14/src/backend/utils/mmgr/README#L37-L40)
- [tuplesort.c:17](../raw/postgres-14/src/backend/utils/sort/tuplesort.c#L17)
- [initdb.c:146](../raw/postgres-14/src/bin/initdb/initdb.c#L146)
- [pg_upgrade.c:18-23](../raw/postgres-14/src/bin/pg_upgrade/pg_upgrade.c#L18-L23)
- [pg_upgrade.h#transferMode](../raw/postgres-14/src/bin/pg_upgrade/pg_upgrade.h#L228-L233)
- [amapi.h#IndexAmRoutine](../raw/postgres-14/src/include/access/amapi.h#L210-L286)
- [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-14/src/include/access/brin.h#L38)
- [brin_page.h:75](../raw/postgres-14/src/include/access/brin_page.h#L75)
- [genam.h#IndexBulkDeleteResult](../raw/postgres-14/src/include/access/genam.h#L74-L83)
- [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-14/src/include/access/gin_private.h#L38-L44)
- [ginblock.h:52](../raw/postgres-14/src/include/access/ginblock.h#L52)
- [gist.h:40](../raw/postgres-14/src/include/access/gist.h#L40)
- [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-14/src/include/access/gist_private.h#L262)
- [hash.h#HASH_METAPAGE](../raw/postgres-14/src/include/access/hash.h#L196)
- [heapam.h#HTSV_Result](../raw/postgres-14/src/include/access/heapam.h#L92-L100)
- [heaptoast.h#MaximumBytesPerTuple](../raw/postgres-14/src/include/access/heaptoast.h#L23-L26)
- [htup_details.h:205](../raw/postgres-14/src/include/access/htup_details.h#L205)
- [itup.h#IndexTupleData](../raw/postgres-14/src/include/access/itup.h#L35-L51)
- [multixact.h:33-34](../raw/postgres-14/src/include/access/multixact.h#L33-L34)
- [nbtree.h#BTMetaPageData](../raw/postgres-14/src/include/access/nbtree.h#L101-L117)
- [spgist_private.h:47](../raw/postgres-14/src/include/access/spgist_private.h#L47)
- [transam.h](../raw/postgres-14/src/include/access/transam.h#L24-L33)
- [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-14/src/include/access/visibilitymapdefs.h#L20-L21)
- [xlog.h#WalLevel](../raw/postgres-14/src/include/access/xlog.h#L163-L168)
- [xlog_internal.h#XLogFileName](../raw/postgres-14/src/include/access/xlog_internal.h#L165)
- [xlogdefs.h#XLogRecPtr](../raw/postgres-14/src/include/access/xlogdefs.h#L21)
- [c.h:795-802](../raw/postgres-14/src/include/c.h#L795-L802)
- [genbki.h:23](../raw/postgres-14/src/include/catalog/genbki.h#L23)
- [index.h#DEFAULT_INDEX_TYPE](../raw/postgres-14/src/include/catalog/index.h#L21)
- [pg_am.dat](../raw/postgres-14/src/include/catalog/pg_am.dat#L15-L34)
- [pg_am.h#FormData_pg_am](../raw/postgres-14/src/include/catalog/pg_am.h#L29-L41)
- [pg_attribute.h:56-62](../raw/postgres-14/src/include/catalog/pg_attribute.h#L56-L62)
- [pg_cast.dat:41](../raw/postgres-14/src/include/catalog/pg_cast.dat#L41)
- [pg_cast.h#FormData_pg_cast](../raw/postgres-14/src/include/catalog/pg_cast.h#L32-L50)
- [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-14/src/include/catalog/pg_class.h#L172)
- [pg_collation.h#FormData_pg_collation](../raw/postgres-14/src/include/catalog/pg_collation.h#L29-L49)
- [pg_control.h:67-68](../raw/postgres-14/src/include/catalog/pg_control.h#L67-L68)
- [pg_description.h#FormData_pg_description](../raw/postgres-14/src/include/catalog/pg_description.h#L48-L56)
- [pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-14/src/include/catalog/pg_event_trigger.h#L29-L42)
- [pg_index.h#FormData_pg_index](../raw/postgres-14/src/include/catalog/pg_index.h#L47-L58)
- [pg_inherits.h#FormData_pg_inherits](../raw/postgres-14/src/include/catalog/pg_inherits.h#L32-L38)
- [pg_opclass.h#FormData_pg_opclass](../raw/postgres-14/src/include/catalog/pg_opclass.h#L49-L76)
- [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-14/src/include/catalog/pg_partitioned_table.h#L30-L58)
- [pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-14/src/include/catalog/pg_proc.h#L163-L165)
- [pg_publication.h#FormData_pg_publication](../raw/postgres-14/src/include/catalog/pg_publication.h#L29-L57)
- [pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-14/src/include/catalog/pg_statistic.h#L209-L218)
- [pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-14/src/include/catalog/pg_statistic_ext.h#L84-L87)
- [pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../raw/postgres-14/src/include/catalog/pg_statistic_ext_data.h#L31-L44)
- [pg_subscription.h#FormData_pg_subscription](../raw/postgres-14/src/include/catalog/pg_subscription.h#L42-L73)
- [pg_subscription_rel.h:64](../raw/postgres-14/src/include/catalog/pg_subscription_rel.h#L64)
- [pg_tablespace.dat](../raw/postgres-14/src/include/catalog/pg_tablespace.dat#L15-L18)
- [pg_type.h:300-303](../raw/postgres-14/src/include/catalog/pg_type.h#L300-L303)
- [event_trigger.h#AT_REWRITE](../raw/postgres-14/src/include/commands/event_trigger.h#L38-L40)
- [vacuum.h:198](../raw/postgres-14/src/include/commands/vacuum.h#L198)
- [relpath.h#ForkNumber](../raw/postgres-14/src/include/common/relpath.h#L40-L53)
- [executor.h#ExecQual](../raw/postgres-14/src/include/executor/executor.h#L402)
- [instrument.h#BufferUsage](../raw/postgres-14/src/include/executor/instrument.h#L24-L38)
- [fmgr.h:38-40](../raw/postgres-14/src/include/fmgr.h#L38-L40)
- [fdwapi.h#FdwRoutine](../raw/postgres-14/src/include/foreign/fdwapi.h#L204-L281)
- [miscadmin.h#START_CRIT_SECTION](../raw/postgres-14/src/include/miscadmin.h#L148-L154)
- [execnodes.h#EState](../raw/postgres-14/src/include/nodes/execnodes.h#L566)
- [lockoptions.h#LockTupleMode](../raw/postgres-14/src/include/nodes/lockoptions.h#L46-L59)
- [nodes.h:673](../raw/postgres-14/src/include/nodes/nodes.h#L673)
- [parsenodes.h#CTEMaterialize](../raw/postgres-14/src/include/nodes/parsenodes.h#L1464-L1469)
- [pathnodes.h#Path](../raw/postgres-14/src/include/nodes/pathnodes.h#L1191-L1192)
- [plannodes.h#PlannedStmt](../raw/postgres-14/src/include/nodes/plannodes.h#L42)
- [primnodes.h#SubPlan](../raw/postgres-14/src/include/nodes/primnodes.h#L696-L767)
- [cost.h:32](../raw/postgres-14/src/include/optimizer/cost.h#L32)
- [kwlist.h:346-347](../raw/postgres-14/src/include/parser/kwlist.h#L346-L347)
- [pgstat.h:28-29](../raw/postgres-14/src/include/pgstat.h#L28-L29)
- [postgres.h#Datum](../raw/postgres-14/src/include/postgres.h#L401-L411)
- [postgres_ext.h:27-31](../raw/postgres-14/src/include/postgres_ext.h#L27-L31)
- [bgworker.h#BackgroundWorker](../raw/postgres-14/src/include/postmaster/bgworker.h#L88-L100)
- [output_plugin.h#OutputPluginCallbacks](../raw/postgres-14/src/include/replication/output_plugin.h#L214)
- [slot.h#ReplicationSlotPersistentData](../raw/postgres-14/src/include/replication/slot.h#L43-L100)
- [block.h#BlockNumber](../raw/postgres-14/src/include/storage/block.h#L31-L33)
- [buf_internals.h#BufferDesc](../raw/postgres-14/src/include/storage/buf_internals.h#L182-L193)
- [bufpage.h#Page](../raw/postgres-14/src/include/storage/bufpage.h#L78)
- [itemid.h#ItemIdData](../raw/postgres-14/src/include/storage/itemid.h#L25-L30)
- [itemptr.h#ItemPointerData](../raw/postgres-14/src/include/storage/itemptr.h#L36-L40)
- [lock.h#LOCKTAG](../raw/postgres-14/src/include/storage/lock.h#L167-L175)
- [lockdefs.h#AccessExclusiveLock](../raw/postgres-14/src/include/storage/lockdefs.h#L45-L47)
- [lwlock.h#LWLock](../raw/postgres-14/src/include/storage/lwlock.h#L39-L48)
- [relfilenode.h#RelFileNode](../raw/postgres-14/src/include/storage/relfilenode.h#L30-L62)
- [sinval.h#SharedInvalidationMessage](../raw/postgres-14/src/include/storage/sinval.h#L113-L122)
- [utility.h:71-78](../raw/postgres-14/src/include/tcop/utility.h#L71-L78)
- [backend_progress.h#ProgressCommandType](../raw/postgres-14/src/include/utils/backend_progress.h#L22-L33)
- [elog.h:20-50](../raw/postgres-14/src/include/utils/elog.h#L20-L50)
- [guc.h#GucContext](../raw/postgres-14/src/include/utils/guc.h#L68-L77)
- [portal.h header](../raw/postgres-14/src/include/utils/portal.h#L3-L9)
- [queryjumble.h#ComputeQueryIdType](../raw/postgres-14/src/include/utils/queryjumble.h#L55-L62)
- [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-14/src/include/utils/rel.h#L330-L331)
- [selfuncs.h:30-40](../raw/postgres-14/src/include/utils/selfuncs.h#L30-L40)
- [snapshot.h#SNAPSHOT_MVCC](../raw/postgres-14/src/include/utils/snapshot.h#L50)
- [syscache.h#SysCacheIdentifier](../raw/postgres-14/src/include/utils/syscache.h#L32-L85)
- [wait_event.h:18-26](../raw/postgres-14/src/include/utils/wait_event.h#L18-L26)
- [pl_handler.c#plpgsql_inline_handler](../raw/postgres-14/src/pl/plpgsql/src/pl_handler.c#L329)
- [plpgsql--1.0.sql](../raw/postgres-14/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L5)
- [isolation/README](../raw/postgres-14/src/test/isolation/README#L6-L11)
- [modules/Makefile](../raw/postgres-14/src/test/modules/Makefile#L7-L33)
- [Cluster.pm](../raw/postgres-14/src/test/perl/PostgreSQL/Test/Cluster.pm#L4-L17)
- [PostgresNode.pm](../raw/postgres-14/src/test/perl/PostgresNode.pm#L8-L14)
- [parallel_schedule](../raw/postgres-14/src/test/regress/parallel_schedule#L12-L17)
- [pg_regress.c header](../raw/postgres-14/src/test/regress/pg_regress.c#L3)

**PostgreSQL 17** (371 files):

- [configure.ac#blocksize](../raw/postgres-17/configure.ac#L258-L289)
- [contrib/Makefile:32-38](../raw/postgres-17/contrib/Makefile#L32-L38)
- [README:1-9](../raw/postgres-17/contrib/README#L1-L9)
- [amcheck.control](../raw/postgres-17/contrib/amcheck/amcheck.control#L1-L3)
- [verify_heapam.c#verify_heapam](../raw/postgres-17/contrib/amcheck/verify_heapam.c#L184-L217)
- [verify_nbtree.c header](../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L1-L14)
- [btree_gin.control](../raw/postgres-17/contrib/btree_gin/btree_gin.control#L1-L6)
- [btree_gist.control](../raw/postgres-17/contrib/btree_gist/btree_gist.control#L1-L6)
- [ginfuncs.c header](../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L1-L3)
- [heapfuncs.c header](../raw/postgres-17/contrib/pageinspect/heapfuncs.c#L1-L4)
- [pageinspect--1.5--1.6.sql](../raw/postgres-17/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88-L90)
- [pageinspect.control](../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L5)
- [rawpage.c#get_raw_page_internal](../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L153)
- [pg_freespacemap--1.1--1.2.sql](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1--1.2.sql#L6-L7)
- [pg_freespacemap--1.1.sql](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L20)
- [pg_freespacemap.c#pg_freespace](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L48)
- [pg_freespacemap.control](../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.control#L1-L2)
- [pg_stat_statements.c:479-480](../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L479-L480)
- [pgstatapprox.c#pgstattuple_approx](../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L227)
- [pgstatindex.c#pgstatindex_impl](../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L250)
- [pgstattuple--1.4--1.5.sql](../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)
- [pgstattuple.c#pgstattuple_type](../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L55-L63)
- [pgstattuple.control](../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L2)
- [postgres_fdw.c#postgres_fdw_handler](../raw/postgres-17/contrib/postgres_fdw/postgres_fdw.c#L553)
- [amcheck.sgml:10-30](../raw/postgres-17/doc/src/sgml/amcheck.sgml#L10-L30)
- [auto-explain.sgml:18-23](../raw/postgres-17/doc/src/sgml/auto-explain.sgml#L18-L23)
- [backup.sgml#backup-timelines](../raw/postgres-17/doc/src/sgml/backup.sgml#L1409-L1418)
- [bki.sgml:40-52](../raw/postgres-17/doc/src/sgml/bki.sgml#L40-L52)
- [btree-gin.sgml:9-22](../raw/postgres-17/doc/src/sgml/btree-gin.sgml#L9-L22)
- [btree-gist.sgml:9-21](../raw/postgres-17/doc/src/sgml/btree-gist.sgml#L9-L21)
- [catalogs.sgml#catalog-pg-attribute](../raw/postgres-17/doc/src/sgml/catalogs.sgml#L1101-L1121)
- [charset.sgml#collation-concepts](../raw/postgres-17/doc/src/sgml/charset.sgml#L649-L672)
- [config.sgml#guc-fsync](../raw/postgres-17/doc/src/sgml/config.sgml#L3022-L3035)
- [contrib.sgml:7-15](../raw/postgres-17/doc/src/sgml/contrib.sgml#L7-L15)
- [datatype.sgml:4767-4781](../raw/postgres-17/doc/src/sgml/datatype.sgml#L4767-L4781)
- [ddl.sgml#ddl-partitioning-declarative](../raw/postgres-17/doc/src/sgml/ddl.sgml#L3979-L4000)
- [event-trigger.sgml:10-16](../raw/postgres-17/doc/src/sgml/event-trigger.sgml#L10-L16)
- [extend.sgml:525-540](../raw/postgres-17/doc/src/sgml/extend.sgml#L525-L540)
- [fdwhandler.sgml#fdwhandler](../raw/postgres-17/doc/src/sgml/fdwhandler.sgml#L11-L19)
- [gin.sgml](../raw/postgres-17/doc/src/sgml/gin.sgml#L503-L528)
- [gist.sgml#gist-intro](../raw/postgres-17/doc/src/sgml/gist.sgml#L11-L25)
- [glossary.sgml#glossary-autovacuum](../raw/postgres-17/doc/src/sgml/glossary.sgml#L125-L142)
- [high-availability.sgml#hot-standby](../raw/postgres-17/doc/src/sgml/high-availability.sgml#L1508-L1523)
- [indexam.sgml#intro](../raw/postgres-17/doc/src/sgml/indexam.sgml#L38-L43)
- [indices.sgml#indexes-expressional](../raw/postgres-17/doc/src/sgml/indices.sgml#L732-L760)
- [installation.sgml#configure-option-enable-injection-points](../raw/postgres-17/doc/src/sgml/installation.sgml#L1657-L1670)
- [logical-replication.sgml#logical-replication-conflicts](../raw/postgres-17/doc/src/sgml/logical-replication.sgml#L1602-L1612)
- [maintenance.sgml#routine-reindex](../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1042-L1046)
- [manage-ag.sgml#manage-ag-tablespaces](../raw/postgres-17/doc/src/sgml/manage-ag.sgml#L380-L392)
- [monitoring.sgml#monitoring-stats](../raw/postgres-17/doc/src/sgml/monitoring.sgml#L137-L145)
- [mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1067-L1090)
- [plpgsql.sgml:4](../raw/postgres-17/doc/src/sgml/plpgsql.sgml#L4)
- [queries.sgml#queries-with](../raw/postgres-17/doc/src/sgml/queries.sgml#L2047-L2062)
- [ref/alter_table.sgml#ADD-table-constraint](../raw/postgres-17/doc/src/sgml/ref/alter_table.sgml#L452-L475)
- [ref/cluster.sgml#description](../raw/postgres-17/doc/src/sgml/ref/cluster.sgml#L45-L49)
- [create_function.sgml#LEAKPROOF](../raw/postgres-17/doc/src/sgml/ref/create_function.sgml#L357-L372)
- [ref/create_index.sgml](../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L612-L643)
- [create_subscription.sgml](../raw/postgres-17/doc/src/sgml/ref/create_subscription.sgml#L400-L411)
- [ref/create_table.sgml#fillfactor](../raw/postgres-17/doc/src/sgml/ref/create_table.sgml#L1468-L1476)
- [ref/explain.sgml#BUFFERS](../raw/postgres-17/doc/src/sgml/ref/explain.sgml#L180-L206)
- [pgupgrade.sgml:39-58](../raw/postgres-17/doc/src/sgml/ref/pgupgrade.sgml#L39-L58)
- [ref/reindex.sgml#bloated](../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L62)
- [ref/truncate.sgml#description](../raw/postgres-17/doc/src/sgml/ref/truncate.sgml#L33)
- [ref/vacuum.sgml#INDEX_CLEANUP](../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L187-L221)
- [regress.sgml](../raw/postgres-17/doc/src/sgml/regress.sgml#L483-L492)
- [rules.sgml:49-52](../raw/postgres-17/doc/src/sgml/rules.sgml#L49-L52)
- [spi.sgml](../raw/postgres-17/doc/src/sgml/spi.sgml#L10-L18)
- [system-views.sgml:3346-3438](../raw/postgres-17/doc/src/sgml/system-views.sgml#L3346-L3438)
- [wal.sgml](../raw/postgres-17/doc/src/sgml/wal.sgml#L495-L509)
- [xfunc.sgml#xfunc-volatility](../raw/postgres-17/doc/src/sgml/xfunc.sgml#L1610-L1667)
- [xindex.sgml#xindex](../raw/postgres-17/doc/src/sgml/xindex.sgml#L3-L32)
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
- [gist.c:79](../raw/postgres-17/src/backend/access/gist/gist.c#L79)
- [gistbuild.c header](../raw/postgres-17/src/backend/access/gist/gistbuild.c#L6-L22)
- [hash/README](../raw/postgres-17/src/backend/access/hash/README#L13-L28)
- [hash.c#hashhandler](../raw/postgres-17/src/backend/access/hash/hash.c#L57-L68)
- [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-17/src/backend/access/hash/hashinsert.c#L364-L370)
- [README.HOT#intro](../raw/postgres-17/src/backend/access/heap/README.HOT#L1-L60)
- [README.tuplock](../raw/postgres-17/src/backend/access/heap/README.tuplock#L1-L13)
- [heapam.c:8548-8558](../raw/postgres-17/src/backend/access/heap/heapam.c#L8548-L8558)
- [heapam_handler.c#heap_tableam_handler](../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L2652-L2662)
- [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L937-L960)
- [pruneheap.c#heap_page_prune_opt](../raw/postgres-17/src/backend/access/heap/pruneheap.c#L180-L230)
- [vacuumlazy.c#lazy_truncate_heap](../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2572-L2600)
- [visibilitymap.c#NOTES](../raw/postgres-17/src/backend/access/heap/visibilitymap.c#L22-L50)
- [amapi.c#GetIndexAmRoutine](../raw/postgres-17/src/backend/access/index/amapi.c#L24-L46)
- [indexam.c#index_getnext_slot](../raw/postgres-17/src/backend/access/index/indexam.c#L632-L698)
- [nbtree/README](../raw/postgres-17/src/backend/access/nbtree/README#L557-L579)
- [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L285-L307)
- [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2757-L2781)
- [nbtpage.c#_bt_upgrademetapage](../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L106-L126)
- [nbtree.c#bthandler](../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L96-L117)
- [nbtsort.c:370-375](../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L370-L375)
- [nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-17/src/backend/access/nbtree/nbtsplitloc.c#L86-L102)
- [nbtutils.c#_bt_allequalimage](../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5140)
- [spgist/README](../raw/postgres-17/src/backend/access/spgist/README#L1-L12)
- [spgutils.c:64](../raw/postgres-17/src/backend/access/spgist/spgutils.c#L64)
- [tableam.c#table_block_relation_estimate_size](../raw/postgres-17/src/backend/access/table/tableam.c#L666-L667)
- [transam/README](../raw/postgres-17/src/backend/access/transam/README#L420-L422)
- [README.parallel#Overview](../raw/postgres-17/src/backend/access/transam/README.parallel#L1-L12)
- [multixact.c header](../raw/postgres-17/src/backend/access/transam/multixact.c#L6-L8)
- [parallel.c:323](../raw/postgres-17/src/backend/access/transam/parallel.c#L323)
- [slru.c](../raw/postgres-17/src/backend/access/transam/slru.c#L1-L10)
- [timeline.c](../raw/postgres-17/src/backend/access/transam/timeline.c#L1-L22)
- [transam.c#TransactionIdPrecedes](../raw/postgres-17/src/backend/access/transam/transam.c#L276-L293)
- [twophase.c](../raw/postgres-17/src/backend/access/transam/twophase.c#L12-L22)
- [varsup.c:443](../raw/postgres-17/src/backend/access/transam/varsup.c#L443)
- [xlog.c#ReadControlFile](../raw/postgres-17/src/backend/access/transam/xlog.c#L4385-L4391)
- [xloginsert.c#XLogSaveBufferForHint](../raw/postgres-17/src/backend/access/transam/xloginsert.c#L1055-L1065)
- [xlogrecovery.c#PerformWalRecovery](../raw/postgres-17/src/backend/access/transam/xlogrecovery.c#L1656-L1662)
- [bootparse.y:4-5](../raw/postgres-17/src/backend/bootstrap/bootparse.y#L4-L5)
- [catalog.c#GetNewOidWithIndex](../raw/postgres-17/src/backend/catalog/catalog.c#L396-L421)
- [genbki.pl:1-7](../raw/postgres-17/src/backend/catalog/genbki.pl#L1-L7)
- [heap.c#StorePartitionBound](../raw/postgres-17/src/backend/catalog/heap.c#L3571-L3608)
- [index.c:1740-1782](../raw/postgres-17/src/backend/catalog/index.c#L1740-L1782)
- [storage.c#RelationTruncate](../raw/postgres-17/src/backend/catalog/storage.c#L280-L289)
- [system_functions.sql:285-289](../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289)
- [system_views.sql#pg_stats](../raw/postgres-17/src/backend/catalog/system_views.sql#L219-L225)
- [analyze.c:2807-2834](../raw/postgres-17/src/backend/commands/analyze.c#L2807-L2834)
- [cluster.c#cluster_rel](../raw/postgres-17/src/backend/commands/cluster.c#L293-L311)
- [comment.c#CreateComments](../raw/postgres-17/src/backend/commands/comment.c#L143-L170)
- [commands/explain.c#ExplainQuery](../raw/postgres-17/src/backend/commands/explain.c#L175-L207)
- [extension.c#ExtensionControlFile](../raw/postgres-17/src/backend/commands/extension.c#L79-L96)
- [functioncmds.c#CreateFunction](../raw/postgres-17/src/backend/commands/functioncmds.c#L1127-L1134)
- [indexcmds.c:678](../raw/postgres-17/src/backend/commands/indexcmds.c#L678)
- [prepare.c header](../raw/postgres-17/src/backend/commands/prepare.c#L1-L5)
- [publicationcmds.c#CreatePublication](../raw/postgres-17/src/backend/commands/publicationcmds.c#L733)
- [subscriptioncmds.c#CreateSubscription](../raw/postgres-17/src/backend/commands/subscriptioncmds.c#L605)
- [tablecmds.c#CreateFKCheckTrigger](../raw/postgres-17/src/backend/commands/tablecmds.c#L12345-L12351)
- [vacuum.c:2054-2055](../raw/postgres-17/src/backend/commands/vacuum.c#L2054-L2055)
- [vacuumparallel.c header](../raw/postgres-17/src/backend/commands/vacuumparallel.c#L9-L17)
- [executor/README#Plan Trees and State Trees](../raw/postgres-17/src/backend/executor/README#L47-L59)
- [execMain.c header](../raw/postgres-17/src/backend/executor/execMain.c#L1-L28)
- [execPartition.c#ExecFindPartition](../raw/postgres-17/src/backend/executor/execPartition.c#L243-L262)
- [execProcnode.c NOTES](../raw/postgres-17/src/backend/executor/execProcnode.c#L43-L61)
- [nodeBitmapHeapscan.c](../raw/postgres-17/src/backend/executor/nodeBitmapHeapscan.c#L1-L17)
- [nodeGather.c header](../raw/postgres-17/src/backend/executor/nodeGather.c#L9-L15)
- [nodeHash.c#get_hash_memory_limit](../raw/postgres-17/src/backend/executor/nodeHash.c#L3602-L3613)
- [nodeIndexonlyscan.c](../raw/postgres-17/src/backend/executor/nodeIndexonlyscan.c#L127-L169)
- [nodeIndexscan.c header](../raw/postgres-17/src/backend/executor/nodeIndexscan.c#L15-L28)
- [nodeMemoize.c header](../raw/postgres-17/src/backend/executor/nodeMemoize.c#L13-L19)
- [nodeSeqscan.c header](../raw/postgres-17/src/backend/executor/nodeSeqscan.c#L15-L26)
- [nodeSubplan.c#ExecSetParamPlan](../raw/postgres-17/src/backend/executor/nodeSubplan.c#L1051-L1059)
- [spi.c#SPI_connect](../raw/postgres-17/src/backend/executor/spi.c#L94)
- [foreign.c#GetFdwRoutine](../raw/postgres-17/src/backend/foreign/foreign.c#L321-L345)
- [jit/README](../raw/postgres-17/src/backend/jit/README#L1-L22)
- [dshash.c header](../raw/postgres-17/src/backend/lib/dshash.c#L3-L9)
- [outfuncs.c#nodeToString](../raw/postgres-17/src/backend/nodes/outfuncs.c#L786-L794)
- [queryjumblefuncs.c header](../raw/postgres-17/src/backend/nodes/queryjumblefuncs.c#L6-L22)
- [read.c#stringToNode](../raw/postgres-17/src/backend/nodes/read.c#L86-L93)
- [tidbitmap.c#tbm_begin_iterate](../raw/postgres-17/src/backend/nodes/tidbitmap.c#L710-L747)
- [clausesel.c#clauselist_selectivity_ext](../raw/postgres-17/src/backend/optimizer/path/clausesel.c#L145-L156)
- [costsize.c#cost_index](../raw/postgres-17/src/backend/optimizer/path/costsize.c#L730-L746)
- [indxpath.c#IndexCollMatchesExprColl](../raw/postgres-17/src/backend/optimizer/path/indxpath.c#L40-L41)
- [joinpath.c#get_memoize_path](../raw/postgres-17/src/backend/optimizer/path/joinpath.c#L577-L581)
- [joinrels.c#try_partitionwise_join](../raw/postgres-17/src/backend/optimizer/path/joinrels.c#L1458-L1477)
- [createplan.c#order_qual_clauses](../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L5283-L5293)
- [planner.c#planner](../raw/postgres-17/src/backend/optimizer/plan/planner.c#L274-L285)
- [subselect.c#SS_process_ctes](../raw/postgres-17/src/backend/optimizer/plan/subselect.c#L934-L951)
- [prepjointree.c:906](../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L906)
- [clauses.c#evaluate_function](../raw/postgres-17/src/backend/optimizer/util/clauses.c#L4495-L4507)
- [inherit.c#expand_inherited_rtentry](../raw/postgres-17/src/backend/optimizer/util/inherit.c#L60-L88)
- [pathnode.c#add_path](../raw/postgres-17/src/backend/optimizer/util/pathnode.c#L361-L420)
- [plancat.c#relation_excluded_by_constraints](../raw/postgres-17/src/backend/optimizer/util/plancat.c#L1576-L1600)
- [relnode.c#build_joinrel_partition_info](../raw/postgres-17/src/backend/optimizer/util/relnode.c#L2016-L2045)
- [restrictinfo.c#make_restrictinfo_internal](../raw/postgres-17/src/backend/optimizer/util/restrictinfo.c#L138-L146)
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
- [logical/Makefile#OBJS](../raw/postgres-17/src/backend/replication/logical/Makefile#L17-L31)
- [decode.c header](../raw/postgres-17/src/backend/replication/logical/decode.c#L3-L8)
- [launcher.c#logicalrep_worker_launch](../raw/postgres-17/src/backend/replication/logical/launcher.c#L476-L485)
- [logical.c header](../raw/postgres-17/src/backend/replication/logical/logical.c#L10-L18)
- [origin.c header](../raw/postgres-17/src/backend/replication/logical/origin.c#L13-L38)
- [reorderbuffer.c NOTES](../raw/postgres-17/src/backend/replication/logical/reorderbuffer.c#L13-L45)
- [tablesync.c](../raw/postgres-17/src/backend/replication/logical/tablesync.c#L11-L27)
- [worker.c header](../raw/postgres-17/src/backend/replication/logical/worker.c#L10-L16)
- [pgoutput.c header](../raw/postgres-17/src/backend/replication/pgoutput/pgoutput.c#L3-L4)
- [slot.c header](../raw/postgres-17/src/backend/replication/slot.c#L15-L27)
- [syncrep.c](../raw/postgres-17/src/backend/replication/syncrep.c#L5-L20)
- [walreceiver.c](../raw/postgres-17/src/backend/replication/walreceiver.c#L5-L16)
- [walsender.c](../raw/postgres-17/src/backend/replication/walsender.c#L5-L18)
- [rewriteHandler.c#QueryRewrite](../raw/postgres-17/src/backend/rewrite/rewriteHandler.c#L4546-L4581)
- [statistics/README](../raw/postgres-17/src/backend/statistics/README#L1-L21)
- [aio/Makefile#OBJS](../raw/postgres-17/src/backend/storage/aio/Makefile#L11-L12)
- [read_stream.c header](../raw/postgres-17/src/backend/storage/aio/read_stream.c#L1-L18)
- [buffer/README#clock-sweep](../raw/postgres-17/src/backend/storage/buffer/README#L166-L199)
- [bufmgr.c#entry-points](../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L15-L29)
- [freelist.c#ClockSweepTick](../raw/postgres-17/src/backend/storage/buffer/freelist.c#L103-L125)
- [fd.c#pg_fsync](../raw/postgres-17/src/backend/storage/file/fd.c#L385-L389)
- [freespace.c#NOTES](../raw/postgres-17/src/backend/storage/freespace/freespace.c#L14-L50)
- [dsm.c header](../raw/postgres-17/src/backend/storage/ipc/dsm.c#L3-L15)
- [dsm_registry.c header](../raw/postgres-17/src/backend/storage/ipc/dsm_registry.c#L3-L16)
- [procarray.c#GetSnapshotData](../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2177)
- [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-17/src/backend/storage/ipc/standby.c#L456-L470)
- [lmgr/README#deadlock-detection](../raw/postgres-17/src/backend/storage/lmgr/README#L338-L359)
- [deadlock.c#DeadLockCheck](../raw/postgres-17/src/backend/storage/lmgr/deadlock.c#L202-L217)
- [lmgr.c#LockRelationOid](../raw/postgres-17/src/backend/storage/lmgr/lmgr.c#L102-L116)
- [lock.c#LockConflicts](../raw/postgres-17/src/backend/storage/lmgr/lock.c#L98-L102)
- [lwlock.c#LWLockAcquire](../raw/postgres-17/src/backend/storage/lmgr/lwlock.c#L1170)
- [proc.c:1295-1307](../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1295-L1307)
- [bufpage.c:315](../raw/postgres-17/src/backend/storage/page/bufpage.c#L315)
- [md.c#register_dirty_segment](../raw/postgres-17/src/backend/storage/smgr/md.c#L1359-L1369)
- [smgr.c](../raw/postgres-17/src/backend/storage/smgr/smgr.c#L1-L12)
- [backend_startup.c#BackendMain](../raw/postgres-17/src/backend/tcop/backend_startup.c#L50-L58)
- [postgres.c#PostgresMain](../raw/postgres-17/src/backend/tcop/postgres.c#L4247-L4259)
- [pquery.c#PortalRun](../raw/postgres-17/src/backend/tcop/pquery.c#L662-L689)
- [utility.c:1106-1113](../raw/postgres-17/src/backend/tcop/utility.c#L1106-L1113)
- [Gen_fmgrtab.pl:1-6](../raw/postgres-17/src/backend/utils/Gen_fmgrtab.pl#L1-L6)
- [backend_progress.c#pgstat_progress_update_param](../raw/postgres-17/src/backend/utils/activity/backend_progress.c#L42-L62)
- [backend_status.c#BackendStatusShmemSize](../raw/postgres-17/src/backend/utils/activity/backend_status.c#L95-L97)
- [pgstat.c header](../raw/postgres-17/src/backend/utils/activity/pgstat.c#L1-L16)
- [pgstat_io.c header](../raw/postgres-17/src/backend/utils/activity/pgstat_io.c#L1-L8)
- [pgstat_relation.c#AtEOXact_PgStat_Relations](../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L568-L570)
- [pgstat_shmem.c#StatsShmemInit](../raw/postgres-17/src/backend/utils/activity/pgstat_shmem.c#L169-L184)
- [wait_event_names.txt](../raw/postgres-17/src/backend/utils/activity/wait_event_names.txt#L7-L24)
- [dbsize.c#calculate_relation_size](../raw/postgres-17/src/backend/utils/adt/dbsize.c#L300-L342)
- [pseudotypes.c#pg_node_tree](../raw/postgres-17/src/backend/utils/adt/pseudotypes.c#L322-L335)
- [ri_triggers.c header](../raw/postgres-17/src/backend/utils/adt/ri_triggers.c#L3-L6)
- [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-17/src/backend/utils/adt/ruleutils.c#L10121-L10170)
- [selfuncs.c#btcostestimate](../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7123)
- [inval.c header](../raw/postgres-17/src/backend/utils/cache/inval.c#L4-L35)
- [plancache.c header](../raw/postgres-17/src/backend/utils/cache/plancache.c#L6-L30)
- [relcache.c:1-22](../raw/postgres-17/src/backend/utils/cache/relcache.c#L1-L22)
- [spccache.c:219-236](../raw/postgres-17/src/backend/utils/cache/spccache.c#L219-L236)
- [syscache.c:13-17](../raw/postgres-17/src/backend/utils/cache/syscache.c#L13-L17)
- [elog.c#errstart](../raw/postgres-17/src/backend/utils/error/elog.c#L356-L364)
- [dfmgr.c:284-289](../raw/postgres-17/src/backend/utils/fmgr/dfmgr.c#L284-L289)
- [fmgr.c#fmgr_info](../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L113-L130)
- [globals.c:139](../raw/postgres-17/src/backend/utils/init/globals.c#L139)
- [miscinit.c#process_shared_preload_libraries](../raw/postgres-17/src/backend/utils/init/miscinit.c#L1894-L1907)
- [guc_tables.c:1450-1456](../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1456)
- [injection_point.c header](../raw/postgres-17/src/backend/utils/misc/injection_point.c#L3-L7)
- [rls.c#check_enable_rls](../raw/postgres-17/src/backend/utils/misc/rls.c#L75-L117)
- [mmgr/README:9-49](../raw/postgres-17/src/backend/utils/mmgr/README#L9-L49)
- [dsa.c header](../raw/postgres-17/src/backend/utils/mmgr/dsa.c#L3-L16)
- [tuplesort.c header](../raw/postgres-17/src/backend/utils/sort/tuplesort.c#L6-L12)
- [tuplesortvariants.c#tuplesort_begin_index_btree](../raw/postgres-17/src/backend/utils/sort/tuplesortvariants.c#L352)
- [initdb.c:167](../raw/postgres-17/src/bin/initdb/initdb.c#L167)
- [pg_checksums.c header](../raw/postgres-17/src/bin/pg_checksums/pg_checksums.c#L1-L5)
- [pg_controldata.c:313](../raw/postgres-17/src/bin/pg_controldata/pg_controldata.c#L313)
- [pg_upgrade.c:17-20](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L17-L20)
- [pg_upgrade.h#transferMode](../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.h#L253-L260)
- [amapi.h#IndexAmRoutine](../raw/postgres-17/src/include/access/amapi.h#L270-L296)
- [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-17/src/include/access/brin.h#L39)
- [brin_page.h:75](../raw/postgres-17/src/include/access/brin_page.h#L75)
- [genam.h#IndexBulkDeleteResult](../raw/postgres-17/src/include/access/genam.h#L75-L84)
- [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-17/src/include/access/gin_private.h#L39-L45)
- [ginblock.h:52](../raw/postgres-17/src/include/access/ginblock.h#L52)
- [gist.h:40](../raw/postgres-17/src/include/access/gist.h#L40)
- [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-17/src/include/access/gist_private.h#L262)
- [hash.h#HASH_METAPAGE](../raw/postgres-17/src/include/access/hash.h#L198)
- [heapam.h#HTSV_Result](../raw/postgres-17/src/include/access/heapam.h#L122-L130)
- [heaptoast.h#MaximumBytesPerTuple](../raw/postgres-17/src/include/access/heaptoast.h#L20-L26)
- [htup_details.h:65-68](../raw/postgres-17/src/include/access/htup_details.h#L65-L68)
- [itup.h#IndexTupleData](../raw/postgres-17/src/include/access/itup.h#L22-L50)
- [nbtree.h#BTMetaPageData](../raw/postgres-17/src/include/access/nbtree.h#L103-L119)
- [spgist_private.h:47](../raw/postgres-17/src/include/access/spgist_private.h#L47)
- [transam.h](../raw/postgres-17/src/include/access/transam.h#L20-L35)
- [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-17/src/include/access/visibilitymapdefs.h#L20-L21)
- [xact.h#SyncCommitLevel](../raw/postgres-17/src/include/access/xact.h#L68-L77)
- [xlog.h#XLogHintBitIsNeeded](../raw/postgres-17/src/include/access/xlog.h#L110-L118)
- [xlog_internal.h#XLogFileName](../raw/postgres-17/src/include/access/xlog_internal.h#L164-L170)
- [xlogdefs.h#XLogRecPtr](../raw/postgres-17/src/include/access/xlogdefs.h#L17-L21)
- [c.h#TYPEALIGN](../raw/postgres-17/src/include/c.h#L808-L828)
- [genbki.h:23](../raw/postgres-17/src/include/catalog/genbki.h#L23)
- [index.h#DEFAULT_INDEX_TYPE](../raw/postgres-17/src/include/catalog/index.h#L21)
- [pg_am.dat](../raw/postgres-17/src/include/catalog/pg_am.dat#L14-L35)
- [pg_am.h#FormData_pg_am](../raw/postgres-17/src/include/catalog/pg_am.h#L29-L41)
- [pg_amop.h#FormData_pg_amop](../raw/postgres-17/src/include/catalog/pg_amop.h#L54-L66)
- [pg_amproc.h#FormData_pg_amproc](../raw/postgres-17/src/include/catalog/pg_amproc.h#L43-L57)
- [pg_attribute.h#FormData_pg_attribute](../raw/postgres-17/src/include/catalog/pg_attribute.h#L37)
- [pg_cast.dat:41-42](../raw/postgres-17/src/include/catalog/pg_cast.dat#L41-L42)
- [pg_cast.h#FormData_pg_cast](../raw/postgres-17/src/include/catalog/pg_cast.h#L32-L50)
- [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-17/src/include/catalog/pg_class.h#L172)
- [pg_collation.h#FormData_pg_collation](../raw/postgres-17/src/include/catalog/pg_collation.h#L29-L50)
- [pg_constraint.h:54](../raw/postgres-17/src/include/catalog/pg_constraint.h#L54)
- [pg_control.h:199](../raw/postgres-17/src/include/catalog/pg_control.h#L199)
- [pg_description.h:10-19](../raw/postgres-17/src/include/catalog/pg_description.h#L10-L19)
- [pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-17/src/include/catalog/pg_event_trigger.h#L29-L42)
- [pg_index.h:53](../raw/postgres-17/src/include/catalog/pg_index.h#L53)
- [pg_inherits.h#FormData_pg_inherits](../raw/postgres-17/src/include/catalog/pg_inherits.h#L32-L38)
- [pg_opclass.h#FormData_pg_opclass](../raw/postgres-17/src/include/catalog/pg_opclass.h#L49-L76)
- [pg_opfamily.h#FormData_pg_opfamily](../raw/postgres-17/src/include/catalog/pg_opfamily.h#L29-L44)
- [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-17/src/include/catalog/pg_partitioned_table.h#L30-L57)
- [pg_policy.h#FormData_pg_policy](../raw/postgres-17/src/include/catalog/pg_policy.h#L29-L44)
- [pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-17/src/include/catalog/pg_proc.h#L158-L166)
- [pg_publication.h#FormData_pg_publication](../raw/postgres-17/src/include/catalog/pg_publication.h#L29-L57)
- [pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-17/src/include/catalog/pg_statistic.h#L212-L222)
- [pg_statistic_ext.h#FormData_pg_statistic_ext](../raw/postgres-17/src/include/catalog/pg_statistic_ext.h#L33-L62)
- [pg_statistic_ext_data.h#FormData_pg_statistic_ext_data](../raw/postgres-17/src/include/catalog/pg_statistic_ext_data.h#L31-L45)
- [pg_subscription.h#LOGICALREP_ORIGIN_NONE](../raw/postgres-17/src/include/catalog/pg_subscription.h#L34-L44)
- [pg_subscription_rel.h](../raw/postgres-17/src/include/catalog/pg_subscription_rel.h#L58-L74)
- [pg_tablespace.dat](../raw/postgres-17/src/include/catalog/pg_tablespace.dat#L14-L18)
- [pg_tablespace.h:29](../raw/postgres-17/src/include/catalog/pg_tablespace.h#L29)
- [pg_type.h#typalign](../raw/postgres-17/src/include/catalog/pg_type.h#L150-L176)
- [event_trigger.h#AT_REWRITE](../raw/postgres-17/src/include/commands/event_trigger.h#L36-L43)
- [progress.h:1-25](../raw/postgres-17/src/include/commands/progress.h#L1-L25)
- [relpath.h#ForkNumber](../raw/postgres-17/src/include/common/relpath.h#L47-L62)
- [executor.h#ExecQual](../raw/postgres-17/src/include/executor/executor.h#L407-L417)
- [instrument.h#BufferUsage](../raw/postgres-17/src/include/executor/instrument.h#L24-L42)
- [fmgr.h:3-8](../raw/postgres-17/src/include/fmgr.h#L3-L8)
- [fdwapi.h#FdwRoutine](../raw/postgres-17/src/include/foreign/fdwapi.h#L204-L281)
- [miscadmin.h#START_CRIT_SECTION](../raw/postgres-17/src/include/miscadmin.h#L150-L156)
- [execnodes.h#PlanState](../raw/postgres-17/src/include/nodes/execnodes.h#L1113-L1130)
- [lockoptions.h#LockTupleMode](../raw/postgres-17/src/include/nodes/lockoptions.h#L46-L59)
- [nodes.h:251](../raw/postgres-17/src/include/nodes/nodes.h#L251)
- [parsenodes.h#CTEMaterialize](../raw/postgres-17/src/include/nodes/parsenodes.h#L1636-L1641)
- [pathnodes.h#Path](../raw/postgres-17/src/include/nodes/pathnodes.h#L1662-L1665)
- [plannodes.h#PlannedStmt](../raw/postgres-17/src/include/nodes/plannodes.h#L31-L35)
- [primnodes.h#Var](../raw/postgres-17/src/include/nodes/primnodes.h#L251-L255)
- [queryjumble.h#IsQueryIdEnabled](../raw/postgres-17/src/include/nodes/queryjumble.h#L72-L84)
- [cost.h:34](../raw/postgres-17/src/include/optimizer/cost.h#L34)
- [kwlist.h:374-375](../raw/postgres-17/src/include/parser/kwlist.h#L374-L375)
- [parsetree.h#rt_fetch](../raw/postgres-17/src/include/parser/parsetree.h#L26-L32)
- [pgstat.h#PgStat_Kind](../raw/postgres-17/src/include/pgstat.h#L33-L54)
- [pg_iovec.h:43](../raw/postgres-17/src/include/port/pg_iovec.h#L43)
- [postgres.h#Datum](../raw/postgres-17/src/include/postgres.h#L54-L64)
- [postgres_ext.h:27-31](../raw/postgres-17/src/include/postgres_ext.h#L27-L31)
- [bgworker.h#BackgroundWorker](../raw/postgres-17/src/include/postmaster/bgworker.h#L89-L101)
- [output_plugin.h#OutputPluginCallbacks](../raw/postgres-17/src/include/replication/output_plugin.h#L216-L243)
- [slot.h#ReplicationSlotPersistentData](../raw/postgres-17/src/include/replication/slot.h#L63-L96)
- [block.h#BlockNumber](../raw/postgres-17/src/include/storage/block.h#L17-L35)
- [buf_internals.h#BufferDesc](../raw/postgres-17/src/include/storage/buf_internals.h#L245-L256)
- [bufmgr.h:156-163](../raw/postgres-17/src/include/storage/bufmgr.h#L156-L163)
- [bufpage.h#Page](../raw/postgres-17/src/include/storage/bufpage.h#L22-L28)
- [itemid.h#ItemIdData](../raw/postgres-17/src/include/storage/itemid.h#L17-L30)
- [itemptr.h#ItemPointerData](../raw/postgres-17/src/include/storage/itemptr.h#L20-L40)
- [lock.h#LOCKTAG](../raw/postgres-17/src/include/storage/lock.h#L164-L172)
- [lockdefs.h#AccessExclusiveLock](../raw/postgres-17/src/include/storage/lockdefs.h#L45-L47)
- [lwlock.h#LWLock](../raw/postgres-17/src/include/storage/lwlock.h#L41-L50)
- [read_stream.h](../raw/postgres-17/src/include/storage/read_stream.h#L27-L59)
- [relfilelocator.h#RelFileLocator](../raw/postgres-17/src/include/storage/relfilelocator.h#L33-L63)
- [sinval.h](../raw/postgres-17/src/include/storage/sinval.h#L20-L56)
- [utility.h:71-78](../raw/postgres-17/src/include/tcop/utility.h#L71-L78)
- [backend_progress.h#PGSTAT_NUM_PROGRESS_PARAM](../raw/postgres-17/src/include/utils/backend_progress.h#L22-L33)
- [backend_status.h#PgBackendStatus](../raw/postgres-17/src/include/utils/backend_status.h#L97-L173)
- [elog.h:25-56](../raw/postgres-17/src/include/utils/elog.h#L25-L56)
- [guc.h#GucContext](../raw/postgres-17/src/include/utils/guc.h#L36-L76)
- [injection_point.h](../raw/postgres-17/src/include/utils/injection_point.h#L14-L21)
- [pgstat_internal.h#PgStat_KindInfo](../raw/postgres-17/src/include/utils/pgstat_internal.h#L201-L219)
- [plancache.h#CachedPlanSource](../raw/postgres-17/src/include/utils/plancache.h#L121-L133)
- [portal.h header](../raw/postgres-17/src/include/utils/portal.h#L3-L9)
- [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-17/src/include/utils/rel.h#L348-L349)
- [selfuncs.h:33-40](../raw/postgres-17/src/include/utils/selfuncs.h#L33-L40)
- [snapshot.h#SNAPSHOT_MVCC](../raw/postgres-17/src/include/utils/snapshot.h#L37-L50)
- [wait_event.h:18-27](../raw/postgres-17/src/include/utils/wait_event.h#L18-L27)
- [varatt.h:142-155](../raw/postgres-17/src/include/varatt.h#L142-L155)
- [pl_handler.c#plpgsql_inline_handler](../raw/postgres-17/src/pl/plpgsql/src/pl_handler.c#L313-L330)
- [plpgsql--1.0.sql](../raw/postgres-17/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L15)
- [isolation/README](../raw/postgres-17/src/test/isolation/README#L3-L12)
- [injection_points.control](../raw/postgres-17/src/test/modules/injection_points/injection_points.control#L1-L4)
- [Cluster.pm](../raw/postgres-17/src/test/perl/PostgreSQL/Test/Cluster.pm#L8-L39)
- [parallel_schedule](../raw/postgres-17/src/test/regress/parallel_schedule#L12-L17)
- [pg_regress.c header](../raw/postgres-17/src/test/regress/pg_regress.c#L3)

**PostgreSQL 18** (308 files):

- [configure.ac#blocksize](../raw/postgres-18/configure.ac#L248-L279)
- [contrib/Makefile:32-40](../raw/postgres-18/contrib/Makefile#L32-L40)
- [README:1-9](../raw/postgres-18/contrib/README#L1-L9)
- [amcheck.control](../raw/postgres-18/contrib/amcheck/amcheck.control#L1-L3)
- [verify_gin.c#gin_index_check](../raw/postgres-18/contrib/amcheck/verify_gin.c#L1-L12)
- [btree_gin.control](../raw/postgres-18/contrib/btree_gin/btree_gin.control#L1-L6)
- [btree_gist.control](../raw/postgres-18/contrib/btree_gist/btree_gist.control#L1-L6)
- [pageinspect--1.5--1.6.sql](../raw/postgres-18/contrib/pageinspect/pageinspect--1.5--1.6.sql#L88-L90)
- [pageinspect.control](../raw/postgres-18/contrib/pageinspect/pageinspect.control#L1-L5)
- [rawpage.c#get_raw_page_internal](../raw/postgres-18/contrib/pageinspect/rawpage.c#L153-L156)
- [pg_freespacemap--1.1.sql](../raw/postgres-18/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L12-L25)
- [pg_freespacemap.c#pg_freespace](../raw/postgres-18/contrib/pg_freespacemap/pg_freespacemap.c#L27-L51)
- [pg_stat_statements.c:1169-1176](../raw/postgres-18/contrib/pg_stat_statements/pg_stat_statements.c#L1169-L1176)
- [pgstatapprox.c#pgstattuple_approx](../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L214)
- [pgstatindex.c#pgstatindex_impl](../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L297-L324)
- [pgstattuple--1.4--1.5.sql](../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)
- [pgstattuple.c#pgstattuple_type](../raw/postgres-18/contrib/pgstattuple/pgstattuple.c#L57-L65)
- [auto-explain.sgml:18-23](../raw/postgres-18/doc/src/sgml/auto-explain.sgml#L18-L23)
- [bki.sgml:40-52](../raw/postgres-18/doc/src/sgml/bki.sgml#L40-L52)
- [config.sgml#guc-huge-pages](../raw/postgres-18/doc/src/sgml/config.sgml#L1801-L1803)
- [contrib.sgml:7-15](../raw/postgres-18/doc/src/sgml/contrib.sgml#L7-L15)
- [event-trigger.sgml:27-36](../raw/postgres-18/doc/src/sgml/event-trigger.sgml#L27-L36)
- [glossary.sgml#glossary-backend](../raw/postgres-18/doc/src/sgml/glossary.sgml#L207-L209)
- [indices.sgml#indexes-expressional](../raw/postgres-18/doc/src/sgml/indices.sgml#L772-L800)
- [installation.sgml#configure-option-enable-injection-points](../raw/postgres-18/doc/src/sgml/installation.sgml#L1689-L1702)
- [logical-replication.sgml#logical-replication-conflicts](../raw/postgres-18/doc/src/sgml/logical-replication.sgml#L1746-L1775)
- [maintenance.sgml#routine-reindex](../raw/postgres-18/doc/src/sgml/maintenance.sgml#L1065-L1068)
- [mvcc.sgml#SHARE UPDATE EXCLUSIVE](../raw/postgres-18/doc/src/sgml/mvcc.sgml#L977-L1000)
- [ref/alter_table.sgml:472-474](../raw/postgres-18/doc/src/sgml/ref/alter_table.sgml#L472-L474)
- [ref/create_index.sgml](../raw/postgres-18/doc/src/sgml/ref/create_index.sgml#L618-L649)
- [create_subscription.sgml](../raw/postgres-18/doc/src/sgml/ref/create_subscription.sgml#L407-L418)
- [ref/create_table.sgml#fillfactor](../raw/postgres-18/doc/src/sgml/ref/create_table.sgml#L1594-L1602)
- [ref/explain.sgml#BUFFERS](../raw/postgres-18/doc/src/sgml/ref/explain.sgml#L180-L208)
- [pgupgrade.sgml#swap](../raw/postgres-18/doc/src/sgml/ref/pgupgrade.sgml#L328-L336)
- [ref/reindex.sgml#bloated](../raw/postgres-18/doc/src/sgml/ref/reindex.sgml#L54-L62)
- [ref/vacuum.sgml:34](../raw/postgres-18/doc/src/sgml/ref/vacuum.sgml#L34)
- [regress.sgml](../raw/postgres-18/doc/src/sgml/regress.sgml#L517-L526)
- [spi.sgml](../raw/postgres-18/doc/src/sgml/spi.sgml#L10-L18)
- [system-views.sgml:3700-3792](../raw/postgres-18/doc/src/sgml/system-views.sgml#L3700-L3792)
- [wal.sgml](../raw/postgres-18/doc/src/sgml/wal.sgml#L500-L503)
- [xfunc.sgml#xfunc-volatility](../raw/postgres-18/doc/src/sgml/xfunc.sgml#L1595)
- [brin/README](../raw/postgres-18/src/backend/access/brin/README#L1-L13)
- [brin.c:296](../raw/postgres-18/src/backend/access/brin/brin.c#L296)
- [reloptions.c#fillfactor](../raw/postgres-18/src/backend/access/common/reloptions.c#L174-L194)
- [gin/README](../raw/postgres-18/src/backend/access/gin/README#L8-L26)
- [ginfast.c:458-471](../raw/postgres-18/src/backend/access/gin/ginfast.c#L458-L471)
- [gininsert.c#GinBuildShared](../raw/postgres-18/src/backend/access/gin/gininsert.c#L49-L96)
- [ginpostinglist.c](../raw/postgres-18/src/backend/access/gin/ginpostinglist.c#L23-L42)
- [ginutil.c:84](../raw/postgres-18/src/backend/access/gin/ginutil.c#L84)
- [gist.c:82](../raw/postgres-18/src/backend/access/gist/gist.c#L82)
- [gistbuild.c:246](../raw/postgres-18/src/backend/access/gist/gistbuild.c#L246)
- [hash/README](../raw/postgres-18/src/backend/access/hash/README#L13-L28)
- [hash.c#hashhandler](../raw/postgres-18/src/backend/access/hash/hash.c#L58-L72)
- [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-18/src/backend/access/hash/hashinsert.c#L364-L370)
- [README.HOT#intro](../raw/postgres-18/src/backend/access/heap/README.HOT#L1-L60)
- [README.tuplock](../raw/postgres-18/src/backend/access/heap/README.tuplock#L1-L13)
- [heapam.c#heap_prepare_freeze_tuple](../raw/postgres-18/src/backend/access/heap/heapam.c#L7272-L7316)
- [heapam_handler.c#heap_tableam_handler](../raw/postgres-18/src/backend/access/heap/heapam_handler.c#L2675-L2685)
- [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-18/src/backend/access/heap/heapam_visibility.c#L937-L960)
- [pruneheap.c#heap_page_prune_opt](../raw/postgres-18/src/backend/access/heap/pruneheap.c#L180-L230)
- [vacuumlazy.c#lazy_truncate_heap](../raw/postgres-18/src/backend/access/heap/vacuumlazy.c#L3246-L3274)
- [visibilitymap.c#NOTES](../raw/postgres-18/src/backend/access/heap/visibilitymap.c#L22-L50)
- [amapi.c#GetIndexAmRoutine](../raw/postgres-18/src/backend/access/index/amapi.c#L24-L46)
- [indexam.c#index_getnext_slot](../raw/postgres-18/src/backend/access/index/indexam.c#L720)
- [nbtree/README](../raw/postgres-18/src/backend/access/nbtree/README#L557-L579)
- [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-18/src/backend/access/nbtree/nbtdedup.c#L285-L307)
- [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-18/src/backend/access/nbtree/nbtinsert.c#L2757-L2781)
- [nbtpage.c#_bt_upgrademetapage](../raw/postgres-18/src/backend/access/nbtree/nbtpage.c#L106-L126)
- [nbtpreprocesskeys.c](../raw/postgres-18/src/backend/access/nbtree/nbtpreprocesskeys.c#L128-L135)
- [nbtree.c:138](../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L138)
- [nbtsort.c:372-377](../raw/postgres-18/src/backend/access/nbtree/nbtsort.c#L372-L377)
- [nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-18/src/backend/access/nbtree/nbtsplitloc.c#L86-L102)
- [nbtutils.c#_bt_allequalimage](../raw/postgres-18/src/backend/access/nbtree/nbtutils.c#L4248-L4259)
- [spgist/README](../raw/postgres-18/src/backend/access/spgist/README#L1-L12)
- [spgutils.c:67](../raw/postgres-18/src/backend/access/spgist/spgutils.c#L67)
- [tableam.c:711-747](../raw/postgres-18/src/backend/access/table/tableam.c#L711-L747)
- [transam/README](../raw/postgres-18/src/backend/access/transam/README#L442-L446)
- [multixact.c](../raw/postgres-18/src/backend/access/transam/multixact.c#L5-L16)
- [parallel.c:327](../raw/postgres-18/src/backend/access/transam/parallel.c#L327)
- [slru.c](../raw/postgres-18/src/backend/access/transam/slru.c#L17-L23)
- [timeline.c](../raw/postgres-18/src/backend/access/transam/timeline.c#L4-L7)
- [transam.c#TransactionIdPrecedes](../raw/postgres-18/src/backend/access/transam/transam.c#L276-L293)
- [twophase.c](../raw/postgres-18/src/backend/access/transam/twophase.c#L24-L26)
- [varsup.c:443](../raw/postgres-18/src/backend/access/transam/varsup.c#L443)
- [xlog.c#ReadControlFile](../raw/postgres-18/src/backend/access/transam/xlog.c#L4434)
- [xloginsert.c#XLogRecordAssemble](../raw/postgres-18/src/backend/access/transam/xloginsert.c#L604-L620)
- [xlogrecovery.c#PerformWalRecovery](../raw/postgres-18/src/backend/access/transam/xlogrecovery.c#L1674-L1680)
- [bootparse.y:4-5](../raw/postgres-18/src/backend/bootstrap/bootparse.y#L4-L5)
- [catalog.c#GetNewOidWithIndex](../raw/postgres-18/src/backend/catalog/catalog.c#L423-L448)
- [genbki.pl:456-475](../raw/postgres-18/src/backend/catalog/genbki.pl#L456-L475)
- [heap.c#StorePartitionBound](../raw/postgres-18/src/backend/catalog/heap.c#L4092-L4127)
- [index.c:1743-1785](../raw/postgres-18/src/backend/catalog/index.c#L1743-L1785)
- [storage.c#RelationTruncate](../raw/postgres-18/src/backend/catalog/storage.c#L281-L290)
- [system_functions.sql:285](../raw/postgres-18/src/backend/catalog/system_functions.sql#L285)
- [system_views.sql:697](../raw/postgres-18/src/backend/catalog/system_views.sql#L697)
- [analyze.c#do_analyze_rel](../raw/postgres-18/src/backend/commands/analyze.c#L270-L281)
- [cluster.c#cluster_rel](../raw/postgres-18/src/backend/commands/cluster.c#L283-L287)
- [comment.c#CreateComments](../raw/postgres-18/src/backend/commands/comment.c#L143-L170)
- [explain.c#ExplainQuery](../raw/postgres-18/src/backend/commands/explain.c#L171-L186)
- [explain_state.c#NewExplainState](../raw/postgres-18/src/backend/commands/explain_state.c#L57-L71)
- [extension.c#ExtensionControlFile](../raw/postgres-18/src/backend/commands/extension.c#L85-L104)
- [functioncmds.c:1150](../raw/postgres-18/src/backend/commands/functioncmds.c#L1150)
- [indexcmds.c:681](../raw/postgres-18/src/backend/commands/indexcmds.c#L681)
- [prepare.c#prepared_queries](../raw/postgres-18/src/backend/commands/prepare.c#L47)
- [publicationcmds.c#CreatePublication](../raw/postgres-18/src/backend/commands/publicationcmds.c#L832)
- [subscriptioncmds.c#CreateSubscription](../raw/postgres-18/src/backend/commands/subscriptioncmds.c#L565)
- [tablecmds.c#QueueFKConstraintValidation](../raw/postgres-18/src/backend/commands/tablecmds.c#L13016)
- [vacuum.c:2092-2093](../raw/postgres-18/src/backend/commands/vacuum.c#L2092-L2093)
- [vacuumparallel.c header](../raw/postgres-18/src/backend/commands/vacuumparallel.c#L9-L17)
- [execMain.c header](../raw/postgres-18/src/backend/executor/execMain.c#L1-L28)
- [execPartition.c#ExecFindPartition](../raw/postgres-18/src/backend/executor/execPartition.c#L265)
- [execProcnode.c#ExecInitNode](../raw/postgres-18/src/backend/executor/execProcnode.c#L142)
- [nodeBitmapHeapscan.c](../raw/postgres-18/src/backend/executor/nodeBitmapHeapscan.c#L1-L17)
- [nodeGather.c header](../raw/postgres-18/src/backend/executor/nodeGather.c#L9-L15)
- [nodeHash.c#get_hash_memory_limit](../raw/postgres-18/src/backend/executor/nodeHash.c#L3622-L3633)
- [nodeIndexonlyscan.c](../raw/postgres-18/src/backend/executor/nodeIndexonlyscan.c#L128-L170)
- [nodeIndexscan.c#IndexNext](../raw/postgres-18/src/backend/executor/nodeIndexscan.c#L80)
- [nodeSeqscan.c:130](../raw/postgres-18/src/backend/executor/nodeSeqscan.c#L130)
- [spi.c#SPI_connect](../raw/postgres-18/src/backend/executor/spi.c#L94)
- [outfuncs.c:379](../raw/postgres-18/src/backend/nodes/outfuncs.c#L379)
- [tidbitmap.c](../raw/postgres-18/src/backend/nodes/tidbitmap.c#L3-L30)
- [clausesel.c#clauselist_selectivity](../raw/postgres-18/src/backend/optimizer/path/clausesel.c#L57-L72)
- [costsize.c:795-797](../raw/postgres-18/src/backend/optimizer/path/costsize.c#L795-L797)
- [indxpath.c#check_index_predicates](../raw/postgres-18/src/backend/optimizer/path/indxpath.c#L3925-L3942)
- [joinpath.c#get_memoize_path](../raw/postgres-18/src/backend/optimizer/path/joinpath.c#L675)
- [joinrels.c#try_partitionwise_join](../raw/postgres-18/src/backend/optimizer/path/joinrels.c#L1401-L1420)
- [createplan.c#order_qual_clauses](../raw/postgres-18/src/backend/optimizer/plan/createplan.c#L5387-L5397)
- [planner.c#planner](../raw/postgres-18/src/backend/optimizer/plan/planner.c#L287-L310)
- [subselect.c#SS_process_ctes](../raw/postgres-18/src/backend/optimizer/plan/subselect.c#L934-L951)
- [prepjointree.c:933](../raw/postgres-18/src/backend/optimizer/prep/prepjointree.c#L933)
- [clauses.c#inline_function](../raw/postgres-18/src/backend/optimizer/util/clauses.c#L4611-L4617)
- [inherit.c#expand_inherited_rtentry](../raw/postgres-18/src/backend/optimizer/util/inherit.c#L86)
- [pathnode.c#add_path](../raw/postgres-18/src/backend/optimizer/util/pathnode.c#L391-L420)
- [plancat.c#relation_excluded_by_constraints](../raw/postgres-18/src/backend/optimizer/util/plancat.c#L1605-L1629)
- [relnode.c#build_joinrel_partition_info](../raw/postgres-18/src/backend/optimizer/util/relnode.c#L1990-L2019)
- [restrictinfo.c:135](../raw/postgres-18/src/backend/optimizer/util/restrictinfo.c#L135)
- [parser/README:1-14](../raw/postgres-18/src/backend/parser/README#L1-L14)
- [gram.y:25-30](../raw/postgres-18/src/backend/parser/gram.y#L25-L30)
- [parser/meson.build:30-41](../raw/postgres-18/src/backend/parser/meson.build#L30-L41)
- [parser.c#raw_parser](../raw/postgres-18/src/backend/parser/parser.c#L34-L42)
- [partprune.c header](../raw/postgres-18/src/backend/partitioning/partprune.c#L3-L25)
- [sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-18/src/backend/port/sysv_shmem.c#L597-L636)
- [autovacuum.c#relation_needs_vacanalyze](../raw/postgres-18/src/backend/postmaster/autovacuum.c#L3092-L3127)
- [bgworker.c#RegisterBackgroundWorker](../raw/postgres-18/src/backend/postmaster/bgworker.c#L932-L940)
- [bgwriter.c](../raw/postgres-18/src/backend/postmaster/bgwriter.c#L5-L13)
- [pmchild.c header](../raw/postgres-18/src/backend/postmaster/pmchild.c#L1-L20)
- [postmaster.c#BackendStartup](../raw/postgres-18/src/backend/postmaster/postmaster.c#L3509-L3571)
- [conflict.c header](../raw/postgres-18/src/backend/replication/logical/conflict.c#L1-L12)
- [decode.c header](../raw/postgres-18/src/backend/replication/logical/decode.c#L3-L8)
- [launcher.c#logicalrep_worker_launch](../raw/postgres-18/src/backend/replication/logical/launcher.c#L473-L500)
- [logical.c#CheckLogicalDecodingRequirements](../raw/postgres-18/src/backend/replication/logical/logical.c#L116-L133)
- [origin.c header](../raw/postgres-18/src/backend/replication/logical/origin.c#L13-L38)
- [reorderbuffer.c#ReorderBufferCheckMemoryLimit](../raw/postgres-18/src/backend/replication/logical/reorderbuffer.c#L3907-L3949)
- [tablesync.c#TablesyncWorkerMain](../raw/postgres-18/src/backend/replication/logical/tablesync.c#L1748)
- [worker.c header](../raw/postgres-18/src/backend/replication/logical/worker.c#L10-L16)
- [pgoutput.c#pgoutput_origin_filter](../raw/postgres-18/src/backend/replication/pgoutput/pgoutput.c#L1765-L1776)
- [slot.c header](../raw/postgres-18/src/backend/replication/slot.c#L15-L21)
- [syncrep.c](../raw/postgres-18/src/backend/replication/syncrep.c#L7-L8)
- [walreceiver.c#WalReceiverMain](../raw/postgres-18/src/backend/replication/walreceiver.c#L159)
- [walsender.c#WalSndCheckTimeOut](../raw/postgres-18/src/backend/replication/walsender.c#L2801)
- [rewriteHandler.c#QueryRewrite](../raw/postgres-18/src/backend/rewrite/rewriteHandler.c#L4626-L4636)
- [aio/Makefile#OBJS](../raw/postgres-18/src/backend/storage/aio/Makefile#L11-L21)
- [aio/README.md](../raw/postgres-18/src/backend/storage/aio/README.md#L1-L15)
- [aio.c#io_method_options](../raw/postgres-18/src/backend/storage/aio/aio.c#L64-L71)
- [read_stream.c header](../raw/postgres-18/src/backend/storage/aio/read_stream.c#L3-L18)
- [buffer/README#bgwriter](../raw/postgres-18/src/backend/storage/buffer/README#L255-L260)
- [bufmgr.c#entry-points](../raw/postgres-18/src/backend/storage/buffer/bufmgr.c#L15-L29)
- [freelist.c#ClockSweepTick](../raw/postgres-18/src/backend/storage/buffer/freelist.c#L103-L125)
- [fd.c:389](../raw/postgres-18/src/backend/storage/file/fd.c#L389)
- [freespace.c#NOTES](../raw/postgres-18/src/backend/storage/freespace/freespace.c#L14-L50)
- [dsm_registry.c#GetNamedDSMSegment](../raw/postgres-18/src/backend/storage/ipc/dsm_registry.c#L132)
- [procarray.c#GetSnapshotData](../raw/postgres-18/src/backend/storage/ipc/procarray.c#L2175)
- [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-18/src/backend/storage/ipc/standby.c#L468)
- [lmgr/README](../raw/postgres-18/src/backend/storage/lmgr/README#L20-L35)
- [deadlock.c#DeadLockCheck](../raw/postgres-18/src/backend/storage/lmgr/deadlock.c#L220)
- [lmgr.c#LockRelationOid](../raw/postgres-18/src/backend/storage/lmgr/lmgr.c#L101-L116)
- [lock.c#LockConflicts](../raw/postgres-18/src/backend/storage/lmgr/lock.c#L99-L103)
- [bufpage.c:314](../raw/postgres-18/src/backend/storage/page/bufpage.c#L314)
- [md.c](../raw/postgres-18/src/backend/storage/smgr/md.c#L44-L50)
- [smgr.c#f_smgr](../raw/postgres-18/src/backend/storage/smgr/smgr.c#L105-L113)
- [sync.c:286](../raw/postgres-18/src/backend/storage/sync/sync.c#L286)
- [backend_startup.c#BackendMain](../raw/postgres-18/src/backend/tcop/backend_startup.c#L69-L77)
- [postgres.c#PostgresMain](../raw/postgres-18/src/backend/tcop/postgres.c#L4176-L4188)
- [pquery.c#PortalRun](../raw/postgres-18/src/backend/tcop/pquery.c#L663-L686)
- [utility.c:1109-1116](../raw/postgres-18/src/backend/tcop/utility.c#L1109-L1116)
- [Gen_fmgrtab.pl:1-6](../raw/postgres-18/src/backend/utils/Gen_fmgrtab.pl#L1-L6)
- [backend_status.c:622](../raw/postgres-18/src/backend/utils/activity/backend_status.c#L622)
- [pgstat.c header](../raw/postgres-18/src/backend/utils/activity/pgstat.c#L1-L17)
- [pgstat_relation.c:582-584](../raw/postgres-18/src/backend/utils/activity/pgstat_relation.c#L582-L584)
- [wait_event_names.txt](../raw/postgres-18/src/backend/utils/activity/wait_event_names.txt#L7-L24)
- [dbsize.c#pg_relation_size](../raw/postgres-18/src/backend/utils/adt/dbsize.c#L364)
- [pseudotypes.c#pg_node_tree](../raw/postgres-18/src/backend/utils/adt/pseudotypes.c#L326)
- [ri_triggers.c#RI_ConstraintInfo](../raw/postgres-18/src/backend/utils/adt/ri_triggers.c#L124)
- [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-18/src/backend/utils/adt/ruleutils.c#L10431-L10480)
- [selfuncs.c:7794-7807](../raw/postgres-18/src/backend/utils/adt/selfuncs.c#L7794-L7807)
- [inval.c:1-35](../raw/postgres-18/src/backend/utils/cache/inval.c#L1-L35)
- [plancache.c#choose_custom_plan](../raw/postgres-18/src/backend/utils/cache/plancache.c#L1166-L1213)
- [relcache.c:1-22](../raw/postgres-18/src/backend/utils/cache/relcache.c#L1-L22)
- [spccache.c:182](../raw/postgres-18/src/backend/utils/cache/spccache.c#L182)
- [syscache.c:13-17](../raw/postgres-18/src/backend/utils/cache/syscache.c#L13-L17)
- [elog.c#errstart](../raw/postgres-18/src/backend/utils/error/elog.c#L353-L361)
- [dfmgr.c:294-299](../raw/postgres-18/src/backend/utils/fmgr/dfmgr.c#L294-L299)
- [fmgr.c#fmgr_info](../raw/postgres-18/src/backend/utils/fmgr/fmgr.c#L113-L130)
- [globals.c:142](../raw/postgres-18/src/backend/utils/init/globals.c#L142)
- [miscinit.c#process_shared_preload_libraries](../raw/postgres-18/src/backend/utils/init/miscinit.c#L1903-L1910)
- [guc_tables.c#io_method](../raw/postgres-18/src/backend/utils/misc/guc_tables.c#L5444-L5451)
- [rls.c#check_enable_rls](../raw/postgres-18/src/backend/utils/misc/rls.c#L52)
- [mmgr/README:9-49](../raw/postgres-18/src/backend/utils/mmgr/README#L9-L49)
- [tuplesort.c:15-16](../raw/postgres-18/src/backend/utils/sort/tuplesort.c#L15-L16)
- [tuplesortvariants.c#tuplesort_begin_index_btree](../raw/postgres-18/src/backend/utils/sort/tuplesortvariants.c#L359)
- [initdb.c:167](../raw/postgres-18/src/bin/initdb/initdb.c#L167)
- [pg_upgrade.c:10-21](../raw/postgres-18/src/bin/pg_upgrade/pg_upgrade.c#L10-L21)
- [pg_upgrade.h#transferMode](../raw/postgres-18/src/bin/pg_upgrade/pg_upgrade.h#L258-L265)
- [amapi.h#IndexAmRoutine](../raw/postgres-18/src/include/access/amapi.h#L230-L323)
- [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-18/src/include/access/brin.h#L39)
- [brin_page.h:75](../raw/postgres-18/src/include/access/brin_page.h#L75)
- [ginblock.h:52](../raw/postgres-18/src/include/access/ginblock.h#L52)
- [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-18/src/include/access/gist_private.h#L262)
- [hash.h#HASH_METAPAGE](../raw/postgres-18/src/include/access/hash.h#L198)
- [heapam.h#HTSV_Result](../raw/postgres-18/src/include/access/heapam.h#L121-L129)
- [heaptoast.h#TOAST_TUPLE_THRESHOLD](../raw/postgres-18/src/include/access/heaptoast.h#L38-L50)
- [htup_details.h:206](../raw/postgres-18/src/include/access/htup_details.h#L206)
- [itup.h#IndexTupleData](../raw/postgres-18/src/include/access/itup.h#L22-L50)
- [nbtree.h#BTMetaPageData](../raw/postgres-18/src/include/access/nbtree.h#L104-L120)
- [spgist_private.h:47](../raw/postgres-18/src/include/access/spgist_private.h#L47)
- [transam.h](../raw/postgres-18/src/include/access/transam.h#L20-L35)
- [tupdesc.h#CompactAttribute](../raw/postgres-18/src/include/access/tupdesc.h#L60-L75)
- [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-18/src/include/access/visibilitymapdefs.h#L20-L21)
- [xlog.h#WalLevel](../raw/postgres-18/src/include/access/xlog.h#L71-L77)
- [xlog_internal.h#XLogFileName](../raw/postgres-18/src/include/access/xlog_internal.h#L166)
- [xlogdefs.h#XLogRecPtr](../raw/postgres-18/src/include/access/xlogdefs.h#L17-L21)
- [c.h:790-797](../raw/postgres-18/src/include/c.h#L790-L797)
- [genbki.h:23](../raw/postgres-18/src/include/catalog/genbki.h#L23)
- [index.h#DEFAULT_INDEX_TYPE](../raw/postgres-18/src/include/catalog/index.h#L21)
- [pg_am.dat](../raw/postgres-18/src/include/catalog/pg_am.dat#L14-L35)
- [pg_attribute.h:100](../raw/postgres-18/src/include/catalog/pg_attribute.h#L100)
- [pg_cast.dat:41-42](../raw/postgres-18/src/include/catalog/pg_cast.dat#L41-L42)
- [pg_cast.h#FormData_pg_cast](../raw/postgres-18/src/include/catalog/pg_cast.h#L32-L50)
- [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-18/src/include/catalog/pg_class.h#L175)
- [pg_collation.h#COLLPROVIDER_DEFAULT](../raw/postgres-18/src/include/catalog/pg_collation.h#L70-L73)
- [pg_description.h#FormData_pg_description](../raw/postgres-18/src/include/catalog/pg_description.h#L48-L57)
- [pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-18/src/include/catalog/pg_event_trigger.h#L29-L42)
- [pg_index.h#FormData_pg_index](../raw/postgres-18/src/include/catalog/pg_index.h#L48-L61)
- [pg_inherits.h#FormData_pg_inherits](../raw/postgres-18/src/include/catalog/pg_inherits.h#L32-L38)
- [pg_opclass.h#FormData_pg_opclass](../raw/postgres-18/src/include/catalog/pg_opclass.h#L49-L76)
- [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-18/src/include/catalog/pg_partitioned_table.h#L30-L58)
- [pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-18/src/include/catalog/pg_proc.h#L164-L166)
- [pg_publication.h#FormData_pg_publication](../raw/postgres-18/src/include/catalog/pg_publication.h#L29-L63)
- [pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-18/src/include/catalog/pg_statistic.h#L213-L222)
- [pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-18/src/include/catalog/pg_statistic_ext.h#L84-L87)
- [pg_statistic_ext_data.h:35](../raw/postgres-18/src/include/catalog/pg_statistic_ext_data.h#L35)
- [pg_subscription.h#LOGICALREP_ORIGIN_NONE](../raw/postgres-18/src/include/catalog/pg_subscription.h#L152-L162)
- [pg_subscription_rel.h](../raw/postgres-18/src/include/catalog/pg_subscription_rel.h#L62-L69)
- [pg_tablespace.dat](../raw/postgres-18/src/include/catalog/pg_tablespace.dat#L15-L18)
- [event_trigger.h#AT_REWRITE](../raw/postgres-18/src/include/commands/event_trigger.h#L36-L43)
- [relpath.h#ForkNumber](../raw/postgres-18/src/include/common/relpath.h#L56-L71)
- [executor.h#ExecQual](../raw/postgres-18/src/include/executor/executor.h#L515)
- [instrument.h#BufferUsage](../raw/postgres-18/src/include/executor/instrument.h#L24-L42)
- [fmgr.h:3-8](../raw/postgres-18/src/include/fmgr.h#L3-L8)
- [fdwapi.h#FdwRoutine](../raw/postgres-18/src/include/foreign/fdwapi.h#L204-L281)
- [miscadmin.h#START_CRIT_SECTION](../raw/postgres-18/src/include/miscadmin.h#L150-L156)
- [execnodes.h#EState](../raw/postgres-18/src/include/nodes/execnodes.h#L649)
- [lockoptions.h#LockTupleMode](../raw/postgres-18/src/include/nodes/lockoptions.h#L46-L59)
- [nodes.h:257](../raw/postgres-18/src/include/nodes/nodes.h#L257)
- [parsenodes.h#CTEMaterialize](../raw/postgres-18/src/include/nodes/parsenodes.h#L1665-L1670)
- [pathnodes.h#Path](../raw/postgres-18/src/include/nodes/pathnodes.h#L1794-L1797)
- [plannodes.h#PlannedStmt](../raw/postgres-18/src/include/nodes/plannodes.h#L31-L35)
- [primnodes.h#SubPlan](../raw/postgres-18/src/include/nodes/primnodes.h#L1041-L1114)
- [cost.h:34](../raw/postgres-18/src/include/optimizer/cost.h#L34)
- [kwlist.h:376-377](../raw/postgres-18/src/include/parser/kwlist.h#L376-L377)
- [pgstat.h#IOOBJECT_WAL](../raw/postgres-18/src/include/pgstat.h#L277-L286)
- [pg_iovec.h:47](../raw/postgres-18/src/include/port/pg_iovec.h#L47)
- [postgres.h#Datum](../raw/postgres-18/src/include/postgres.h#L59-L69)
- [postgres_ext.h:28-32](../raw/postgres-18/src/include/postgres_ext.h#L28-L32)
- [bgworker.h#BackgroundWorker](../raw/postgres-18/src/include/postmaster/bgworker.h#L89-L101)
- [conflict.h#ConflictType](../raw/postgres-18/src/include/replication/conflict.h#L31-L59)
- [output_plugin.h#OutputPluginCallbacks](../raw/postgres-18/src/include/replication/output_plugin.h#L216-L243)
- [slot.h#ReplicationSlotPersistentData](../raw/postgres-18/src/include/replication/slot.h#L70-L103)
- [aio.h#IoMethod](../raw/postgres-18/src/include/storage/aio.h#L31-L42)
- [block.h#BlockNumber](../raw/postgres-18/src/include/storage/block.h#L17-L35)
- [buf_internals.h#BufferDesc](../raw/postgres-18/src/include/storage/buf_internals.h#L258-L271)
- [bufmgr.h:161-162](../raw/postgres-18/src/include/storage/bufmgr.h#L161-L162)
- [bufpage.h#Page](../raw/postgres-18/src/include/storage/bufpage.h#L25-L31)
- [itemid.h#ItemIdData](../raw/postgres-18/src/include/storage/itemid.h#L17-L30)
- [itemptr.h#ItemPointerData](../raw/postgres-18/src/include/storage/itemptr.h#L20-L40)
- [lock.h:318](../raw/postgres-18/src/include/storage/lock.h#L318)
- [lockdefs.h#AccessExclusiveLock](../raw/postgres-18/src/include/storage/lockdefs.h#L45-L47)
- [lwlock.h#LWLock](../raw/postgres-18/src/include/storage/lwlock.h#L41-L50)
- [relfilelocator.h#RelFileLocator](../raw/postgres-18/src/include/storage/relfilelocator.h#L33-L63)
- [sinval.h](../raw/postgres-18/src/include/storage/sinval.h#L21-L31)
- [backend_progress.h#ProgressCommandType](../raw/postgres-18/src/include/utils/backend_progress.h#L22-L33)
- [elog.h:25-56](../raw/postgres-18/src/include/utils/elog.h#L25-L56)
- [guc.h#GucContext](../raw/postgres-18/src/include/utils/guc.h#L40-L80)
- [injection_point.h](../raw/postgres-18/src/include/utils/injection_point.h#L14-L27)
- [pgstat_kind.h](../raw/postgres-18/src/include/utils/pgstat_kind.h#L16-L59)
- [portal.h#PortalData](../raw/postgres-18/src/include/utils/portal.h#L113-L120)
- [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-18/src/include/utils/rel.h#L359-L360)
- [selfuncs.h:33-40](../raw/postgres-18/src/include/utils/selfuncs.h#L33-L40)
- [snapshot.h#SNAPSHOT_MVCC](../raw/postgres-18/src/include/utils/snapshot.h#L33-L46)
- [wait_classes.h](../raw/postgres-18/src/include/utils/wait_classes.h#L18-L27)
- [wait_event.h#pgstat_report_wait_start](../raw/postgres-18/src/include/utils/wait_event.h#L48-L76)
- [varatt.h:142-155](../raw/postgres-18/src/include/varatt.h#L142-L155)
- [pl_handler.c#plpgsql_inline_handler](../raw/postgres-18/src/pl/plpgsql/src/pl_handler.c#L314-L331)
- [plpgsql--1.0.sql](../raw/postgres-18/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L15)
- [isolation/README](../raw/postgres-18/src/test/isolation/README#L3-L12)
- [Cluster.pm](../raw/postgres-18/src/test/perl/PostgreSQL/Test/Cluster.pm#L8-L39)
- [parallel_schedule](../raw/postgres-18/src/test/regress/parallel_schedule#L12-L17)
- [pg_regress.c header](../raw/postgres-18/src/test/regress/pg_regress.c#L3)

**PostgreSQL 19** (341 files):

- [configure.ac#blocksize](../raw/postgres-19/configure.ac#L248-L274)
- [contrib/Makefile:36-40](../raw/postgres-19/contrib/Makefile#L36-L40)
- [README:1-28](../raw/postgres-19/contrib/README#L1-L28)
- [verify_gin.c:79](../raw/postgres-19/contrib/amcheck/verify_gin.c#L79)
- [verify_heapam.c:252](../raw/postgres-19/contrib/amcheck/verify_heapam.c#L252)
- [verify_nbtree.c:252](../raw/postgres-19/contrib/amcheck/verify_nbtree.c#L252)
- [btree_gin--1.3--1.4.sql:6-20](../raw/postgres-19/contrib/btree_gin/btree_gin--1.3--1.4.sql#L6-L20)
- [btree_gin.control](../raw/postgres-19/contrib/btree_gin/btree_gin.control#L1-L6)
- [pageinspect--1.5.sql:178](../raw/postgres-19/contrib/pageinspect/pageinspect--1.5.sql#L178)
- [pageinspect.control](../raw/postgres-19/contrib/pageinspect/pageinspect.control#L1-L5)
- [rawpage.c#get_raw_page_internal](../raw/postgres-19/contrib/pageinspect/rawpage.c#L145-L156)
- [pg_freespacemap--1.1--1.2.sql](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap--1.1--1.2.sql#L6-L7)
- [pg_freespacemap--1.1.sql](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L23-L25)
- [pg_freespacemap--1.2--1.3.sql](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap--1.2--1.3.sql#L6-L13)
- [pg_freespacemap.c#pg_freespace](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap.c#L28-L49)
- [pg_freespacemap.control](../raw/postgres-19/contrib/pg_freespacemap/pg_freespacemap.control#L1-L2)
- [README:18-27](../raw/postgres-19/contrib/pg_plan_advice/README#L18-L27)
- [pg_plan_advice.c#_PG_init](../raw/postgres-19/contrib/pg_plan_advice/pg_plan_advice.c#L127-L138)
- [pgpa_planner.c#pgpa_planner_install_hooks](../raw/postgres-19/contrib/pg_plan_advice/pgpa_planner.c#L180-L194)
- [pg_stat_statements.c:492](../raw/postgres-19/contrib/pg_stat_statements/pg_stat_statements.c#L492)
- [pgstatapprox.c#pgstattuple_approx](../raw/postgres-19/contrib/pgstattuple/pgstatapprox.c#L279)
- [pgstatindex.c#pgstatindex_impl](../raw/postgres-19/contrib/pgstattuple/pgstatindex.c#L214-L250)
- [pgstattuple--1.4--1.5.sql](../raw/postgres-19/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L37)
- [pgstattuple.c#pgstattuple_type](../raw/postgres-19/contrib/pgstattuple/pgstattuple.c#L57-L65)
- [pgstattuple.control](../raw/postgres-19/contrib/pgstattuple/pgstattuple.control#L1-L2)
- [auto-explain.sgml:18-24](../raw/postgres-19/doc/src/sgml/auto-explain.sgml#L18-L24)
- [bki.sgml:40-61](../raw/postgres-19/doc/src/sgml/bki.sgml#L40-L61)
- [catalogs.sgml#relpages](../raw/postgres-19/doc/src/sgml/catalogs.sgml#L2052-L2080)
- [config.sgml#guc-huge-pages](../raw/postgres-19/doc/src/sgml/config.sgml#L1844-L1863)
- [contrib.sgml:7-16](../raw/postgres-19/doc/src/sgml/contrib.sgml#L7-L16)
- [datatype.sgml](../raw/postgres-19/doc/src/sgml/datatype.sgml#L4807-L4821)
- [event-trigger.sgml:10-16](../raw/postgres-19/doc/src/sgml/event-trigger.sgml#L10-L16)
- [extend.sgml:527-540](../raw/postgres-19/doc/src/sgml/extend.sgml#L527-L540)
- [func-admin.sgml:3159-3186](../raw/postgres-19/doc/src/sgml/func/func-admin.sgml#L3159-L3186)
- [gin.sgml](../raw/postgres-19/doc/src/sgml/gin.sgml#L504-L530)
- [gist.sgml#gist-intro](../raw/postgres-19/doc/src/sgml/gist.sgml#L11-L25)
- [glossary.sgml#glossary-autovacuum](../raw/postgres-19/doc/src/sgml/glossary.sgml#L165-L183)
- [indexam.sgml#intro](../raw/postgres-19/doc/src/sgml/indexam.sgml#L37-L44)
- [indices.sgml#indexes-expressional](../raw/postgres-19/doc/src/sgml/indices.sgml#L772-L800)
- [installation.sgml#configure-option-enable-injection-points](../raw/postgres-19/doc/src/sgml/installation.sgml#L1684-L1696)
- [logical-replication.sgml#conflict-update-deleted](../raw/postgres-19/doc/src/sgml/logical-replication.sgml#L2068-L2082)
- [maintenance.sgml#routine-reindex](../raw/postgres-19/doc/src/sgml/maintenance.sgml#L1253-L1257)
- [monitoring.sgml#monitoring-stats](../raw/postgres-19/doc/src/sgml/monitoring.sgml#L137-L152)
- [mvcc.sgml#ACCESS EXCLUSIVE](../raw/postgres-19/doc/src/sgml/mvcc.sgml#L1081-L1109)
- [pgplanadvice.sgml](../raw/postgres-19/doc/src/sgml/pgplanadvice.sgml#L30-L39)
- [plpgsql.sgml:4](../raw/postgres-19/doc/src/sgml/plpgsql.sgml#L4)
- [ref/alter_table.sgml:483-485](../raw/postgres-19/doc/src/sgml/ref/alter_table.sgml#L483-L485)
- [ref/cluster.sgml#description](../raw/postgres-19/doc/src/sgml/ref/cluster.sgml#L32-L40)
- [ref/create_index.sgml](../raw/postgres-19/doc/src/sgml/ref/create_index.sgml#L618-L646)
- [create_subscription.sgml](../raw/postgres-19/doc/src/sgml/ref/create_subscription.sgml#L433-L443)
- [ref/create_table.sgml#fillfactor](../raw/postgres-19/doc/src/sgml/ref/create_table.sgml#L1606-L1625)
- [ref/explain.sgml#BUFFERS](../raw/postgres-19/doc/src/sgml/ref/explain.sgml#L183-L203)
- [pgupgrade.sgml:39-58](../raw/postgres-19/doc/src/sgml/ref/pgupgrade.sgml#L39-L58)
- [ref/reindex.sgml#bloated](../raw/postgres-19/doc/src/sgml/ref/reindex.sgml#L54-L63)
- [ref/repack.sgml](../raw/postgres-19/doc/src/sgml/ref/repack.sgml#L42-L50)
- [ref/truncate.sgml#description](../raw/postgres-19/doc/src/sgml/ref/truncate.sgml#L33)
- [ref/vacuum.sgml:33](../raw/postgres-19/doc/src/sgml/ref/vacuum.sgml#L33)
- [regress.sgml](../raw/postgres-19/doc/src/sgml/regress.sgml#L617-L623)
- [spi.sgml](../raw/postgres-19/doc/src/sgml/spi.sgml#L8-L28)
- [system-views.sgml](../raw/postgres-19/doc/src/sgml/system-views.sgml#L3895-L4005)
- [wal.sgml](../raw/postgres-19/doc/src/sgml/wal.sgml#L652-L668)
- [xfunc.sgml#xfunc-volatility](../raw/postgres-19/doc/src/sgml/xfunc.sgml#L1595)
- [meson.build#bison_kw](../raw/postgres-19/meson.build#L430-L431)
- [brin/README](../raw/postgres-19/src/backend/access/brin/README#L1-L24)
- [brin.c:300](../raw/postgres-19/src/backend/access/brin/brin.c#L300)
- [reloptions.c#fillfactor](../raw/postgres-19/src/backend/access/common/reloptions.c#L189-L198)
- [gin/README](../raw/postgres-19/src/backend/access/gin/README#L8-L26)
- [ginfast.c](../raw/postgres-19/src/backend/access/gin/ginfast.c#L1-L7)
- [gininsert.c#buildFreshLeafTuple](../raw/postgres-19/src/backend/access/gin/gininsert.c#L290-L341)
- [ginpostinglist.c](../raw/postgres-19/src/backend/access/gin/ginpostinglist.c#L23-L42)
- [ginutil.c:85](../raw/postgres-19/src/backend/access/gin/ginutil.c#L85)
- [ginvacuum.c:829](../raw/postgres-19/src/backend/access/gin/ginvacuum.c#L829)
- [gist/README](../raw/postgres-19/src/backend/access/gist/README#L1-L25)
- [gist.c:82](../raw/postgres-19/src/backend/access/gist/gist.c#L82)
- [gistbuild.c:246](../raw/postgres-19/src/backend/access/gist/gistbuild.c#L246)
- [hash/README](../raw/postgres-19/src/backend/access/hash/README#L13-L28)
- [hash.c#hashhandler](../raw/postgres-19/src/backend/access/hash/hash.c#L70-L84)
- [hashinsert.c#_hash_vacuum_one_page](../raw/postgres-19/src/backend/access/hash/hashinsert.c#L368)
- [README.HOT#intro](../raw/postgres-19/src/backend/access/heap/README.HOT#L34-L45)
- [README.tuplock](../raw/postgres-19/src/backend/access/heap/README.tuplock#L1-L13)
- [heapam.c#heap_prepare_freeze_tuple](../raw/postgres-19/src/backend/access/heap/heapam.c#L7217-L7270)
- [heapam_handler.c#heap_tableam_handler](../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L2731-L2734)
- [heapam_visibility.c#HeapTupleSatisfiesMVCC](../raw/postgres-19/src/backend/access/heap/heapam_visibility.c#L939)
- [pruneheap.c#heap_page_prune_opt](../raw/postgres-19/src/backend/access/heap/pruneheap.c#L272-L325)
- [vacuumlazy.c#lazy_truncate_heap](../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L3166-L3225)
- [visibilitymap.c#NOTES](../raw/postgres-19/src/backend/access/heap/visibilitymap.c#L23-L60)
- [amapi.c#GetIndexAmRoutine](../raw/postgres-19/src/backend/access/index/amapi.c#L24-L59)
- [indexam.c#index_getnext_slot](../raw/postgres-19/src/backend/access/index/indexam.c#L698)
- [nbtree/README](../raw/postgres-19/src/backend/access/nbtree/README#L556-L583)
- [nbtdedup.c#_bt_bottomupdel_pass](../raw/postgres-19/src/backend/access/nbtree/nbtdedup.c#L285-L310)
- [nbtinsert.c#_bt_delete_or_dedup_one_page](../raw/postgres-19/src/backend/access/nbtree/nbtinsert.c#L2803-L2828)
- [nbtpage.c#_bt_upgrademetapage](../raw/postgres-19/src/backend/access/nbtree/nbtpage.c#L100-L127)
- [nbtree.c#bthandler](../raw/postgres-19/src/backend/access/nbtree/nbtree.c#L118-L121)
- [nbtsort.c:375-380](../raw/postgres-19/src/backend/access/nbtree/nbtsort.c#L375-L380)
- [nbtsplitloc.c#_bt_findsplitloc](../raw/postgres-19/src/backend/access/nbtree/nbtsplitloc.c#L90-L99)
- [nbtutils.c#_bt_allequalimage](../raw/postgres-19/src/backend/access/nbtree/nbtutils.c#L1165-L1175)
- [spgist/README](../raw/postgres-19/src/backend/access/spgist/README#L1-L24)
- [spgutils.c:67](../raw/postgres-19/src/backend/access/spgist/spgutils.c#L67)
- [tableam.c#table_block_relation_estimate_size](../raw/postgres-19/src/backend/access/table/tableam.c#L718-L778)
- [transam/README](../raw/postgres-19/src/backend/access/transam/README#L420-L422)
- [multixact.c](../raw/postgres-19/src/backend/access/transam/multixact.c#L5-L17)
- [slru.c](../raw/postgres-19/src/backend/access/transam/slru.c#L17-L23)
- [timeline.c](../raw/postgres-19/src/backend/access/transam/timeline.c#L4-L7)
- [twophase.c](../raw/postgres-19/src/backend/access/transam/twophase.c#L24-L26)
- [varsup.c:438](../raw/postgres-19/src/backend/access/transam/varsup.c#L438)
- [xlog.c#ReadControlFile](../raw/postgres-19/src/backend/access/transam/xlog.c#L4505)
- [xloginsert.c#XLogRecordAssemble](../raw/postgres-19/src/backend/access/transam/xloginsert.c#L678-L693)
- [xlogrecovery.c#PerformWalRecovery](../raw/postgres-19/src/backend/access/transam/xlogrecovery.c#L1615-L1621)
- [bootparse.y:4-5](../raw/postgres-19/src/backend/bootstrap/bootparse.y#L4-L5)
- [catalog.c#GetNewOidWithIndex](../raw/postgres-19/src/backend/catalog/catalog.c#L448)
- [genbki.pl:1-7](../raw/postgres-19/src/backend/catalog/genbki.pl#L1-L7)
- [heap.c#StorePartitionBound](../raw/postgres-19/src/backend/catalog/heap.c#L4096-L4131)
- [index.c:1758-1790](../raw/postgres-19/src/backend/catalog/index.c#L1758-L1790)
- [storage.c#RelationTruncate](../raw/postgres-19/src/backend/catalog/storage.c#L289)
- [system_functions.sql:269](../raw/postgres-19/src/backend/catalog/system_functions.sql#L269)
- [system_views.sql:735](../raw/postgres-19/src/backend/catalog/system_views.sql#L735)
- [analyze.c#do_analyze_rel](../raw/postgres-19/src/backend/commands/analyze.c#L306)
- [comment.c#CreateComments](../raw/postgres-19/src/backend/commands/comment.c#L145-L166)
- [explain.c#ExplainQuery](../raw/postgres-19/src/backend/commands/explain.c#L176-L191)
- [explain_state.c#NewExplainState](../raw/postgres-19/src/backend/commands/explain_state.c#L60-L74)
- [extension.c#ExtensionControlFile](../raw/postgres-19/src/backend/commands/extension.c#L86-L105)
- [functioncmds.c:1165](../raw/postgres-19/src/backend/commands/functioncmds.c#L1165)
- [indexcmds.c:685](../raw/postgres-19/src/backend/commands/indexcmds.c#L685)
- [prepare.c#prepared_queries](../raw/postgres-19/src/backend/commands/prepare.c#L49)
- [publicationcmds.c#CreatePublication](../raw/postgres-19/src/backend/commands/publicationcmds.c#L836)
- [repack.c:1-21](../raw/postgres-19/src/backend/commands/repack.c#L1-L21)
- [subscriptioncmds.c#CreateSubscription](../raw/postgres-19/src/backend/commands/subscriptioncmds.c#L647)
- [tablecmds.c#ATExecValidateConstraint](../raw/postgres-19/src/backend/commands/tablecmds.c#L13337)
- [vacuum.c:2084-2085](../raw/postgres-19/src/backend/commands/vacuum.c#L2084-L2085)
- [vacuumparallel.c header](../raw/postgres-19/src/backend/commands/vacuumparallel.c#L1-L25)
- [execMain.c header](../raw/postgres-19/src/backend/executor/execMain.c#L1-L28)
- [execPartition.c#ExecFindPartition](../raw/postgres-19/src/backend/executor/execPartition.c#L268)
- [execProcnode.c#ExecInitNode](../raw/postgres-19/src/backend/executor/execProcnode.c#L142)
- [nodeBitmapHeapscan.c](../raw/postgres-19/src/backend/executor/nodeBitmapHeapscan.c#L1-L16)
- [nodeGather.c header](../raw/postgres-19/src/backend/executor/nodeGather.c#L9-L15)
- [nodeHash.c#get_hash_memory_limit](../raw/postgres-19/src/backend/executor/nodeHash.c#L3672-L3690)
- [nodeIndexonlyscan.c#IndexOnlyNext](../raw/postgres-19/src/backend/executor/nodeIndexonlyscan.c#L131-L176)
- [nodeIndexscan.c#IndexNext](../raw/postgres-19/src/backend/executor/nodeIndexscan.c#L82)
- [spi.c#SPI_connect](../raw/postgres-19/src/backend/executor/spi.c#L95)
- [outfuncs.c:378](../raw/postgres-19/src/backend/nodes/outfuncs.c#L378)
- [queryjumblefuncs.c#JumbleQuery](../raw/postgres-19/src/backend/nodes/queryjumblefuncs.c#L139)
- [tidbitmap.c](../raw/postgres-19/src/backend/nodes/tidbitmap.c#L3-L30)
- [clausesel.c#clauselist_selectivity](../raw/postgres-19/src/backend/optimizer/path/clausesel.c#L55-L100)
- [costsize.c:784-786](../raw/postgres-19/src/backend/optimizer/path/costsize.c#L784-L786)
- [indxpath.c#check_index_predicates](../raw/postgres-19/src/backend/optimizer/path/indxpath.c#L3940)
- [joinpath.c:31-32](../raw/postgres-19/src/backend/optimizer/path/joinpath.c#L31-L32)
- [joinrels.c#try_partitionwise_join](../raw/postgres-19/src/backend/optimizer/path/joinrels.c#L1591-L1611)
- [createplan.c#order_qual_clauses](../raw/postgres-19/src/backend/optimizer/plan/createplan.c#L5234-L5244)
- [planner.c#planner](../raw/postgres-19/src/backend/optimizer/plan/planner.c#L328-L342)
- [subselect.c#SS_process_ctes](../raw/postgres-19/src/backend/optimizer/plan/subselect.c#L942-L959)
- [prepjointree.c:1192](../raw/postgres-19/src/backend/optimizer/prep/prepjointree.c#L1192)
- [clauses.c#inline_function](../raw/postgres-19/src/backend/optimizer/util/clauses.c#L5361-L5367)
- [inherit.c#expand_inherited_rtentry](../raw/postgres-19/src/backend/optimizer/util/inherit.c#L88)
- [pathnode.c#add_path](../raw/postgres-19/src/backend/optimizer/util/pathnode.c#L395-L405)
- [plancat.c#relation_excluded_by_constraints](../raw/postgres-19/src/backend/optimizer/util/plancat.c#L1840-L1913)
- [relnode.c:50-54](../raw/postgres-19/src/backend/optimizer/util/relnode.c#L50-L54)
- [restrictinfo.c:135](../raw/postgres-19/src/backend/optimizer/util/restrictinfo.c#L135)
- [parser/README:1-16](../raw/postgres-19/src/backend/parser/README#L1-L16)
- [gram.y:6-7](../raw/postgres-19/src/backend/parser/gram.y#L6-L7)
- [parser/meson.build:29-43](../raw/postgres-19/src/backend/parser/meson.build#L29-L43)
- [parser.c#raw_parser](../raw/postgres-19/src/backend/parser/parser.c#L33-L42)
- [partprune.c header](../raw/postgres-19/src/backend/partitioning/partprune.c#L3-L26)
- [sysv_shmem.c#CreateAnonymousSegment](../raw/postgres-19/src/backend/port/sysv_shmem.c#L600-L640)
- [autovacuum.c:5-27](../raw/postgres-19/src/backend/postmaster/autovacuum.c#L5-L27)
- [bgworker.c#RegisterBackgroundWorker](../raw/postgres-19/src/backend/postmaster/bgworker.c#L955-L962)
- [bgwriter.c](../raw/postgres-19/src/backend/postmaster/bgwriter.c#L5-L13)
- [datachecksum_state.c header](../raw/postgres-19/src/backend/postmaster/datachecksum_state.c#L1-L40)
- [postmaster.c#BackendStartup](../raw/postgres-19/src/backend/postmaster/postmaster.c#L3591-L3610)
- [decode.c header](../raw/postgres-19/src/backend/replication/logical/decode.c#L3-L8)
- [launcher.c#logicalrep_worker_launch](../raw/postgres-19/src/backend/replication/logical/launcher.c#L509-L517)
- [logical.c header](../raw/postgres-19/src/backend/replication/logical/logical.c#L10-L18)
- [logicalctl.c header](../raw/postgres-19/src/backend/replication/logical/logicalctl.c#L3-L37)
- [origin.c header](../raw/postgres-19/src/backend/replication/logical/origin.c#L13-L43)
- [reorderbuffer.c#ReorderBufferCheckMemoryLimit](../raw/postgres-19/src/backend/replication/logical/reorderbuffer.c#L3931-L3985)
- [sequencesync.c](../raw/postgres-19/src/backend/replication/logical/sequencesync.c#L9-L20)
- [tablesync.c#TableSyncWorkerMain](../raw/postgres-19/src/backend/replication/logical/tablesync.c#L1608-L1618)
- [worker.c header](../raw/postgres-19/src/backend/replication/logical/worker.c#L10-L16)
- [pgoutput.c header](../raw/postgres-19/src/backend/replication/pgoutput/pgoutput.c#L3-L4)
- [pgrepack.c header](../raw/postgres-19/src/backend/replication/pgrepack/pgrepack.c#L1-L10)
- [slot.c header](../raw/postgres-19/src/backend/replication/slot.c#L13-L21)
- [syncrep.c](../raw/postgres-19/src/backend/replication/syncrep.c#L7-L8)
- [walreceiver.c#WalReceiverMain](../raw/postgres-19/src/backend/replication/walreceiver.c#L155)
- [walsender.c#WalSndCheckTimeOut](../raw/postgres-19/src/backend/replication/walsender.c#L2966)
- [rewriteHandler.c#QueryRewrite](../raw/postgres-19/src/backend/rewrite/rewriteHandler.c#L4786-L4796)
- [attribute_stats.c#pg_restore_attribute_stats](../raw/postgres-19/src/backend/statistics/attribute_stats.c#L699)
- [relation_stats.c#pg_restore_relation_stats](../raw/postgres-19/src/backend/statistics/relation_stats.c#L248)
- [buffer/README#clock-sweep](../raw/postgres-19/src/backend/storage/buffer/README#L172-L203)
- [bufmgr.c#entry-points](../raw/postgres-19/src/backend/storage/buffer/bufmgr.c#L15-L32)
- [freelist.c#ClockSweepTick](../raw/postgres-19/src/backend/storage/buffer/freelist.c#L103-L120)
- [fd.c:390](../raw/postgres-19/src/backend/storage/file/fd.c#L390)
- [freespace.c#NOTES](../raw/postgres-19/src/backend/storage/freespace/freespace.c#L14-L50)
- [dsm_registry.c#GetNamedDSA](../raw/postgres-19/src/backend/storage/ipc/dsm_registry.c#L279)
- [procarray.c#GetSnapshotData](../raw/postgres-19/src/backend/storage/ipc/procarray.c#L2114)
- [standby.c#ResolveRecoveryConflictWithSnapshot](../raw/postgres-19/src/backend/storage/ipc/standby.c#L470)
- [lmgr/README](../raw/postgres-19/src/backend/storage/lmgr/README#L32-L35)
- [deadlock.c#DeadLockCheck](../raw/postgres-19/src/backend/storage/lmgr/deadlock.c#L220)
- [lmgr.c#LockRelationOid](../raw/postgres-19/src/backend/storage/lmgr/lmgr.c#L107-L116)
- [lock.c#LockConflicts](../raw/postgres-19/src/backend/storage/lmgr/lock.c#L102-L106)
- [lwlock.c#LWLockAcquire](../raw/postgres-19/src/backend/storage/lmgr/lwlock.c#L1150)
- [bufpage.c:326](../raw/postgres-19/src/backend/storage/page/bufpage.c#L326)
- [md.c](../raw/postgres-19/src/backend/storage/smgr/md.c#L46-L52)
- [smgr.c#f_smgr](../raw/postgres-19/src/backend/storage/smgr/smgr.c#L105-L113)
- [sync.c:287](../raw/postgres-19/src/backend/storage/sync/sync.c#L287)
- [backend_startup.c#BackendMain](../raw/postgres-19/src/backend/tcop/backend_startup.c#L70-L77)
- [postgres.c#PostgresMain](../raw/postgres-19/src/backend/tcop/postgres.c#L4273-L4275)
- [pquery.c#PortalRun](../raw/postgres-19/src/backend/tcop/pquery.c#L681)
- [utility.c:1118](../raw/postgres-19/src/backend/tcop/utility.c#L1118)
- [Gen_fmgrtab.pl:1-6](../raw/postgres-19/src/backend/utils/Gen_fmgrtab.pl#L1-L6)
- [backend_status.c:109-112](../raw/postgres-19/src/backend/utils/activity/backend_status.c#L109-L112)
- [pgstat.c header](../raw/postgres-19/src/backend/utils/activity/pgstat.c#L1-L16)
- [pgstat_relation.c:583-585](../raw/postgres-19/src/backend/utils/activity/pgstat_relation.c#L583-L585)
- [wait_event_names.txt](../raw/postgres-19/src/backend/utils/activity/wait_event_names.txt#L7-L24)
- [dbsize.c#pg_relation_size](../raw/postgres-19/src/backend/utils/adt/dbsize.c#L364)
- [ri_triggers.c#RI_FKey_check](../raw/postgres-19/src/backend/utils/adt/ri_triggers.c#L509-L527)
- [ruleutils.c#T_PartitionBoundSpec](../raw/postgres-19/src/backend/utils/adt/ruleutils.c#L11010-L11059)
- [selfuncs.c:8150-8161](../raw/postgres-19/src/backend/utils/adt/selfuncs.c#L8150-L8161)
- [inval.c#AcceptInvalidationMessages](../raw/postgres-19/src/backend/utils/cache/inval.c#L930)
- [plancache.c#choose_custom_plan](../raw/postgres-19/src/backend/utils/cache/plancache.c#L1184-L1231)
- [relcache.c:1-22](../raw/postgres-19/src/backend/utils/cache/relcache.c#L1-L22)
- [spccache.c:183](../raw/postgres-19/src/backend/utils/cache/spccache.c#L183)
- [syscache.c:13-17](../raw/postgres-19/src/backend/utils/cache/syscache.c#L13-L17)
- [elog.c#errstart](../raw/postgres-19/src/backend/utils/error/elog.c#L366-L373)
- [dfmgr.c:294-299](../raw/postgres-19/src/backend/utils/fmgr/dfmgr.c#L294-L299)
- [fmgr.c#fmgr_info](../raw/postgres-19/src/backend/utils/fmgr/fmgr.c#L128-L132)
- [globals.c:144](../raw/postgres-19/src/backend/utils/init/globals.c#L144)
- [miscinit.c#process_shared_preload_libraries](../raw/postgres-19/src/backend/utils/init/miscinit.c#L1853-L1860)
- [gen_guc_tables.pl:1-10](../raw/postgres-19/src/backend/utils/misc/gen_guc_tables.pl#L1-L10)
- [guc_parameters.dat#io_max_workers](../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L1393-L1415)
- [guc_tables.c:1-10](../raw/postgres-19/src/backend/utils/misc/guc_tables.c#L1-L10)
- [injection_point.c header](../raw/postgres-19/src/backend/utils/misc/injection_point.c#L3-L7)
- [mmgr/README:9-49](../raw/postgres-19/src/backend/utils/mmgr/README#L9-L49)
- [tuplesort.c:15-16](../raw/postgres-19/src/backend/utils/sort/tuplesort.c#L15-L16)
- [tuplesortvariants.c#tuplesort_begin_index_btree](../raw/postgres-19/src/backend/utils/sort/tuplesortvariants.c#L360)
- [initdb.c:167](../raw/postgres-19/src/bin/initdb/initdb.c#L167)
- [pg_upgrade.c:9-21](../raw/postgres-19/src/bin/pg_upgrade/pg_upgrade.c#L9-L21)
- [pg_upgrade.h#transferMode](../raw/postgres-19/src/bin/pg_upgrade/pg_upgrade.h#L266-L273)
- [amapi.h#IndexAmRoutine](../raw/postgres-19/src/include/access/amapi.h#L302-L313)
- [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../raw/postgres-19/src/include/access/brin.h#L40)
- [brin_page.h:75](../raw/postgres-19/src/include/access/brin_page.h#L75)
- [gin_private.h#GinGetPendingListCleanupSize](../raw/postgres-19/src/include/access/gin_private.h#L41-L47)
- [ginblock.h:52](../raw/postgres-19/src/include/access/ginblock.h#L52)
- [gist_private.h#GIST_ROOT_BLKNO](../raw/postgres-19/src/include/access/gist_private.h#L262)
- [hash.h#HASH_METAPAGE](../raw/postgres-19/src/include/access/hash.h#L198)
- [heapam.h#HTSV_Result](../raw/postgres-19/src/include/access/heapam.h#L135-L143)
- [heaptoast.h#MaximumBytesPerTuple](../raw/postgres-19/src/include/access/heaptoast.h#L20-L26)
- [htup_details.h:206](../raw/postgres-19/src/include/access/htup_details.h#L206)
- [itup.h#IndexTupleData](../raw/postgres-19/src/include/access/itup.h#L35-L51)
- [nbtree.h#BTMetaPageData](../raw/postgres-19/src/include/access/nbtree.h#L104-L120)
- [spgist_private.h:47](../raw/postgres-19/src/include/access/spgist_private.h#L47)
- [transam.h](../raw/postgres-19/src/include/access/transam.h#L23-L33)
- [visibilitymapdefs.h#VISIBILITYMAP_ALL_VISIBLE](../raw/postgres-19/src/include/access/visibilitymapdefs.h#L20-L21)
- [xlog.h:136](../raw/postgres-19/src/include/access/xlog.h#L136)
- [xlog_internal.h#XLogFileName](../raw/postgres-19/src/include/access/xlog_internal.h#L165)
- [xlogdefs.h#XLogRecPtr](../raw/postgres-19/src/include/access/xlogdefs.h#L21)
- [c.h:896-903](../raw/postgres-19/src/include/c.h#L896-L903)
- [genbki.h:42](../raw/postgres-19/src/include/catalog/genbki.h#L42)
- [index.h#DEFAULT_INDEX_TYPE](../raw/postgres-19/src/include/catalog/index.h#L27)
- [pg_am.dat](../raw/postgres-19/src/include/catalog/pg_am.dat#L14-L35)
- [pg_am.h#FormData_pg_am](../raw/postgres-19/src/include/catalog/pg_am.h#L31-L43)
- [pg_attribute.h#FormData_pg_attribute](../raw/postgres-19/src/include/catalog/pg_attribute.h#L39)
- [pg_cast.dat:41-42](../raw/postgres-19/src/include/catalog/pg_cast.dat#L41-L42)
- [pg_cast.h#FormData_pg_cast](../raw/postgres-19/src/include/catalog/pg_cast.h#L34-L52)
- [pg_class.h#RELKIND_PARTITIONED_TABLE](../raw/postgres-19/src/include/catalog/pg_class.h#L179)
- [pg_collation.h#FormData_pg_collation](../raw/postgres-19/src/include/catalog/pg_collation.h#L31-L53)
- [pg_constraint.h:57](../raw/postgres-19/src/include/catalog/pg_constraint.h#L57)
- [pg_description.h:10-21](../raw/postgres-19/src/include/catalog/pg_description.h#L10-L21)
- [pg_event_trigger.h#FormData_pg_event_trigger](../raw/postgres-19/src/include/catalog/pg_event_trigger.h#L31-L45)
- [pg_index.h#FormData_pg_index](../raw/postgres-19/src/include/catalog/pg_index.h#L50-L62)
- [pg_inherits.h#FormData_pg_inherits](../raw/postgres-19/src/include/catalog/pg_inherits.h#L34-L40)
- [pg_opclass.h#FormData_pg_opclass](../raw/postgres-19/src/include/catalog/pg_opclass.h#L51-L78)
- [pg_partitioned_table.h#FormData_pg_partitioned_table](../raw/postgres-19/src/include/catalog/pg_partitioned_table.h#L32-L60)
- [pg_policy.h:31](../raw/postgres-19/src/include/catalog/pg_policy.h#L31)
- [pg_proc.dat#pg_restore_extended_stats](../raw/postgres-19/src/include/catalog/pg_proc.dat#L12615-L12622)
- [pg_proc.h#PROVOLATILE_IMMUTABLE](../raw/postgres-19/src/include/catalog/pg_proc.h#L168-L170)
- [pg_publication.h#FormData_pg_publication](../raw/postgres-19/src/include/catalog/pg_publication.h#L31-L71)
- [pg_statistic.h#STATISTIC_KIND_CORRELATION](../raw/postgres-19/src/include/catalog/pg_statistic.h#L217-L226)
- [pg_statistic_ext.h#STATS_EXT_NDISTINCT](../raw/postgres-19/src/include/catalog/pg_statistic_ext.h#L88-L91)
- [pg_statistic_ext_data.h:37](../raw/postgres-19/src/include/catalog/pg_statistic_ext_data.h#L37)
- [pg_subscription.h:83-84](../raw/postgres-19/src/include/catalog/pg_subscription.h#L83-L84)
- [pg_tablespace.dat](../raw/postgres-19/src/include/catalog/pg_tablespace.dat#L15-L18)
- [event_trigger.h#AT_REWRITE](../raw/postgres-19/src/include/commands/event_trigger.h#L40-L43)
- [relpath.h#ForkNumber](../raw/postgres-19/src/include/common/relpath.h#L56-L69)
- [executor.h#ExecQual](../raw/postgres-19/src/include/executor/executor.h#L527)
- [instrument.h#BufferUsage](../raw/postgres-19/src/include/executor/instrument.h#L24-L42)
- [fmgr.h:3-8](../raw/postgres-19/src/include/fmgr.h#L3-L8)
- [fdwapi.h#FdwRoutine](../raw/postgres-19/src/include/foreign/fdwapi.h#L208-L286)
- [miscadmin.h#START_CRIT_SECTION](../raw/postgres-19/src/include/miscadmin.h#L152-L158)
- [execnodes.h#EState](../raw/postgres-19/src/include/nodes/execnodes.h#L690)
- [lockoptions.h#LockTupleMode](../raw/postgres-19/src/include/nodes/lockoptions.h#L47-L60)
- [nodes.h:259](../raw/postgres-19/src/include/nodes/nodes.h#L259)
- [parsenodes.h#CTEMaterialize](../raw/postgres-19/src/include/nodes/parsenodes.h#L1741-L1746)
- [pathnodes.h#Path](../raw/postgres-19/src/include/nodes/pathnodes.h#L2005-L2008)
- [plannodes.h#PlannedStmt](../raw/postgres-19/src/include/nodes/plannodes.h#L44-L58)
- [primnodes.h#SubPlan](../raw/postgres-19/src/include/nodes/primnodes.h#L1095-L1097)
- [supportnodes.h#SupportRequestInlineInFrom](../raw/postgres-19/src/include/nodes/supportnodes.h#L108-L125)
- [cost.h:34](../raw/postgres-19/src/include/optimizer/cost.h#L34)
- [plancat.h:17-20](../raw/postgres-19/src/include/optimizer/plancat.h#L17-L20)
- [kwlist.h:388](../raw/postgres-19/src/include/parser/kwlist.h#L388)
- [pg_config_manual.h#USE_FLOAT8_BYVAL](../raw/postgres-19/src/include/pg_config_manual.h#L85-L91)
- [pgstat.h#IOObject](../raw/postgres-19/src/include/pgstat.h#L281-L286)
- [postgres.h#Datum](../raw/postgres-19/src/include/postgres.h#L58-L76)
- [postgres_ext.h:29-32](../raw/postgres-19/src/include/postgres_ext.h#L29-L32)
- [bgworker.h#BackgroundWorker](../raw/postgres-19/src/include/postmaster/bgworker.h#L96-L108)
- [conflict.h#ConflictType](../raw/postgres-19/src/include/replication/conflict.h#L31-L62)
- [output_plugin.h#OutputPluginCallbacks](../raw/postgres-19/src/include/replication/output_plugin.h#L216-L243)
- [slot.h#ReplicationSlotPersistentData](../raw/postgres-19/src/include/replication/slot.h#L95-L162)
- [worker_internal.h#LogicalRepWorkerType](../raw/postgres-19/src/include/replication/worker_internal.h#L27-L35)
- [aio.h#DEFAULT_IO_METHOD](../raw/postgres-19/src/include/storage/aio.h#L34-L42)
- [block.h#BlockNumber](../raw/postgres-19/src/include/storage/block.h#L17-L33)
- [buf_internals.h#BufferDesc](../raw/postgres-19/src/include/storage/buf_internals.h#L303-L310)
- [bufmgr.h:217](../raw/postgres-19/src/include/storage/bufmgr.h#L217)
- [bufpage.h#Page](../raw/postgres-19/src/include/storage/bufpage.h#L25-L28)
- [itemid.h#ItemIdData](../raw/postgres-19/src/include/storage/itemid.h#L17-L41)
- [itemptr.h#ItemPointerData](../raw/postgres-19/src/include/storage/itemptr.h#L17-L45)
- [lock.h:148](../raw/postgres-19/src/include/storage/lock.h#L148)
- [lockdefs.h#AccessExclusiveLock](../raw/postgres-19/src/include/storage/lockdefs.h#L45-L46)
- [locktag.h#LOCKTAG](../raw/postgres-19/src/include/storage/locktag.h#L64-L72)
- [lwlock.h#LWLock](../raw/postgres-19/src/include/storage/lwlock.h#L41-L50)
- [relfilelocator.h#RelFileLocator](../raw/postgres-19/src/include/storage/relfilelocator.h#L20-L63)
- [sinval.h#SharedInvalidationMessage](../raw/postgres-19/src/include/storage/sinval.h#L124-L134)
- [utility.h:71-78](../raw/postgres-19/src/include/tcop/utility.h#L71-L78)
- [backend_progress.h#ProgressCommandType](../raw/postgres-19/src/include/utils/backend_progress.h#L22-L34)
- [elog.h:27-58](../raw/postgres-19/src/include/utils/elog.h#L27-L58)
- [guc.h#GucContext](../raw/postgres-19/src/include/utils/guc.h#L39-L80)
- [guc_tables.h:311](../raw/postgres-19/src/include/utils/guc_tables.h#L311)
- [injection_point.h](../raw/postgres-19/src/include/utils/injection_point.h#L27-L40)
- [pgstat_kind.h](../raw/postgres-19/src/include/utils/pgstat_kind.h#L26-L52)
- [portal.h header](../raw/postgres-19/src/include/utils/portal.h#L3-L9)
- [rel.h#HEAP_DEFAULT_FILLFACTOR](../raw/postgres-19/src/include/utils/rel.h#L361-L362)
- [selfuncs.h:30-37](../raw/postgres-19/src/include/utils/selfuncs.h#L30-L37)
- [snapshot.h#SNAPSHOT_MVCC](../raw/postgres-19/src/include/utils/snapshot.h#L36-L46)
- [wait_classes.h:18-27](../raw/postgres-19/src/include/utils/wait_classes.h#L18-L27)
- [wait_event.h#pgstat_report_wait_start](../raw/postgres-19/src/include/utils/wait_event.h#L67)
- [varatt.h:157-170](../raw/postgres-19/src/include/varatt.h#L157-L170)
- [pl_handler.c#plpgsql_inline_handler](../raw/postgres-19/src/pl/plpgsql/src/pl_handler.c#L316-L330)
- [plpgsql--1.0.sql](../raw/postgres-19/src/pl/plpgsql/src/plpgsql--1.0.sql#L3-L15)
- [isolation/README](../raw/postgres-19/src/test/isolation/README#L3-L12)
- [injection_points.control](../raw/postgres-19/src/test/modules/injection_points/injection_points.control#L1-L4)
- [Cluster.pm](../raw/postgres-19/src/test/perl/PostgreSQL/Test/Cluster.pm#L8-L12)
- [parallel_schedule](../raw/postgres-19/src/test/regress/parallel_schedule#L10-L18)
- [pg_regress.c header](../raw/postgres-19/src/test/regress/pg_regress.c#L3)

## Navigation

- [Wiki index](index.md)
- [Versions](versions.md)
- [Overview](overview.md)
- [PostgreSQL 12 index](v12/index.md)
- [PostgreSQL 12 Codebase Navigation Guide (unverified)](v12/codebase-navigation-guide.md)
- [PostgreSQL 14 index](v14/index.md)
- [PostgreSQL 14 Codebase Navigation Guide (unverified)](v14/codebase-navigation-guide.md)
- [PostgreSQL 17 index](v17/index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](v17/codebase-navigation-guide.md)
- [PostgreSQL 18 index](v18/index.md)
- [PostgreSQL 18 Codebase Navigation Guide (unverified)](v18/codebase-navigation-guide.md)
- [PostgreSQL 19 index](v19/index.md)
- [PostgreSQL 19 Codebase Navigation Guide (unverified)](v19/codebase-navigation-guide.md)
