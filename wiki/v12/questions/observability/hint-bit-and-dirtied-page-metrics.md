---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Metrics for Hint Bits Set and Pages Dirtied as Hints in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Main idea](#main-idea)
  - [Values that decide what a metric shows](#values-that-decide-what-a-metric-shows)
  - [Logic map: which counters a hint reaches](#logic-map-which-counters-a-hint-reaches)
  - [Comparison of the available metrics](#comparison-of-the-available-metrics)
  - [How the dirtied counter is incremented](#how-the-dirtied-counter-is-incremented)
  - [Where each reporter takes its numbers from](#where-each-reporter-takes-its-numbers-from)
  - [Cases that change the counts](#cases-that-change-the-counts)
  - [Lifecycle events](#lifecycle-events)
  - [Key interaction: pages, bits and images are three different units](#key-interaction-pages-bits-and-images-are-three-different-units)
  - [Example queries](#example-queries)
  - [Causal summary](#causal-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, what metrics or stats views measure, or provide stats on, the number of set hint bits and of pages dirtied as hints?

## Answer

### Main idea

**PostgreSQL 12 has no metric that counts [hint bits](../../../glossary.md#hint-bits) being set. It also has no statistics view that counts pages dirtied *by hints*.** When `SetHintBits()` sets a bit, it ORs the bit into the tuple header and calls `MarkBufferDirtyHint()`. It can also return without setting anything; see [Cases that change the counts](#cases-that-change-the-counts). Either way it increments no counter of its own ([heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131)). None of the cumulative statistics views has a "dirtied" column: `pg_stat_database`, `pg_statio_*_tables` and `pg_stat_bgwriter` count block reads, hits and writes only ([system_views.sql#pg_statio_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L622-L639), [system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L880), [system_views.sql#pg_stat_bgwriter](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947)).

What exists are indirect measures, in three groups:

1. **Dirtied-block counters.** Every clean-to-[dirty](../../../glossary.md#dirty-buffer) transition of a buffer increments `pgBufferUsage.shared_blks_dirtied` (or `local_blks_dirtied` for temporary tables). This happens both for [WAL](../../../glossary.md#wal)-logged changes and for hint changes ([bufmgr.c:1497-1502](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1497-L1502), [bufmgr.c:3541-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3541-L3546), [localbuf.c:300-301](../../../../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L300-L301)). Three reporters expose it: [`EXPLAIN (ANALYZE, BUFFERS)`](../../../glossary.md#explain-buffers) as `dirtied=`, [`pg_stat_statements`](../../../glossary.md#pg_stat_statements) as `shared_blks_dirtied` and `local_blks_dirtied`, and [`auto_explain`](../../../glossary.md#auto_explain) with `log_analyze` and `log_buffers`. A fourth counter, `VacuumPageDirty`, is printed as "dirtied" in the [autovacuum](../../../glossary.md#autovacuum) log line ([vacuumlazy.c:430-434](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L430-L434)). None of them tells a hint apart from any other change.
2. **Hint WAL records.** When [data checksums](../../../glossary.md#data-checksums) or `wal_log_hints` are on, a hint on a permanent page that has had no WAL-logged change since the latest [checkpoint](../../../glossary.md#checkpoint)'s redo start writes an `XLOG_FPI_FOR_HINT` [full-page image](../../../glossary.md#full-page-image), unless the buffer is already dirty and unchanged since any write began, or the server is in recovery ([bufmgr.c:3470-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3507), [xloginsert.c:917](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L917), [xloginsert.c:953](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L953)). [`pg_waldump`](../../../glossary.md#pg_waldump) `--stats=record` counts these records as `XLOG/FPI_FOR_HINT` ([pg_waldump.c:971-983](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L971-L983), [pg_waldump.c:729-751](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L729-L751), [xlogdesc.c:185-186](../../../../raw/postgres-12/src/backend/access/rmgrdesc/xlogdesc.c#L185-L186)). This is the only hint-specific count in 12, and it is not a SQL view. By default both settings are off ([initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185)), so no such record exists.
3. **Point-in-time inspection.** [`pageinspect`](../../../glossary.md#pageinspect)'s `heap_page_items()` returns each tuple's `t_infomask`, so a superuser can count the tuples that currently carry each hint bit ([pageinspect--1.5.sql:38-55](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L38-L55), [htup_details.h:203-207](../../../../raw/postgres-12/src/include/access/htup_details.h#L203-L207)). A bit being present does not prove a hint writer set it: every insert sets `HEAP_XMAX_INVALID`, for example; see [Example queries](#example-queries). [`pg_buffercache`](../../../glossary.md#pg_buffercache) shows whether each shared buffer is dirty right now, but not why ([pg_buffercache_pages.c:164-167](../../../../raw/postgres-12/contrib/pg_buffercache/pg_buffercache_pages.c#L164-L167)). Both describe the current state, not how many hints were set over time.

The existing page [Which Client Statements Set Hint Bits and Dirty Pages as Hints in PostgreSQL 12 (unverified)](../storage-and-vacuum/client-statements-that-set-hint-bits.md) explains which statements reach `SetHintBits()` and `MarkBufferDirtyHint()`. This page covers how to observe them.

### Values that decide what a metric shows

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| Hint bits `HEAP_XMIN_COMMITTED` (0x0100), `HEAP_XMIN_INVALID` (0x0200), `HEAP_XMAX_COMMITTED` (0x0400), `HEAP_XMAX_INVALID` (0x0800) | Cached transaction outcome in each tuple header; both [xmin](../../../glossary.md#xmin-and-xmax) bits together mean frozen | `t_infomask` on the page ([htup_details.h:203-207](../../../../raw/postgres-12/src/include/access/htup_details.h#L203-L207)) | Stored on the page | `pageinspect` reads them; no counter records when they are set |
| `BM_DIRTY`, `BM_JUST_DIRTIED` | Buffer has unwritten changes; buffer was changed since its current write started | Buffer header state word ([buf_internals.h:59](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L59), [buf_internals.h:64](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L64)) | Live, shared memory | `BM_DIRTY` decides whether a dirtying call increments the counter; `pg_buffercache.isdirty` reports it |
| `pgBufferUsage` | Per-[backend](../../../glossary.md#backend) running totals of block hits, reads, dirtied and written | `BufferUsage` struct ([instrument.h#BufferUsage](../../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33)) | Live, backend-local memory | `EXPLAIN`, `pg_stat_statements` and `auto_explain` take differences of it |
| `VacuumPageDirty` | Count of buffers dirtied since the start of the current `vacuum()` call | Global incremented beside `shared_blks_dirtied`, reset in `vacuum()` ([bufmgr.c:1499](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1499), [bufmgr.c:3543](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3543), [vacuum.c:385](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L385)) | Live, backend-local | Autovacuum log line: the `dirtied` count and the average write rate ([vacuumlazy.c:392](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L392)). Cost accounting does not read it: it adds the separate `VacuumCostPageDirty` to `VacuumCostBalance` ([bufmgr.c:3545-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3545-L3546)) |
| `XLogHintBitIsNeeded()` | `DataChecksumsEnabled() \|\| wal_log_hints` | `pg_control` and the GUC ([xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192)) | Checksums: set by `initdb`, and changed only on a shut-down cluster with `pg_checksums` ([pg_checksums.c:383-384](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L383-L384), [pg_checksums.c:516-519](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L516-L519), [pg_checksums.c:599](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L599)). `wal_log_hints`: server start | Decides whether a hint can produce an `FPI_FOR_HINT` record |
| Page [LSN](../../../glossary.md#lsn) vs `RedoRecPtr` | Whether the page has had a WAL-protected change since the latest checkpoint's redo start | Page header; `GetRedoRecPtr()` ([xloginsert.c:907-917](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L907-L917)) | Page LSN stored, redo pointer live | Decides whether a hint on a WAL-branch cluster writes an image |

Two of these values have similar names but different units. The dirtied counters count **buffers**, once per clean-to-dirty transition. The hint bits are **per tuple**: one page can gain hundreds of bits in one scan, yet normally adds one to the dirtied counter. It adds again only if the buffer is written clean between two hints. The `FPI_FOR_HINT` count is a third unit: **page images**, normally one per page per checkpoint cycle. That is not a hard limit; see [Key interaction](#key-interaction-pages-bits-and-images-are-three-different-units).

### Logic map: which counters a hint reaches

```text
SetHintBits() / other hint writer           (no counter of its own)    [A]
  │
  └─ MarkBufferDirtyHint(buffer)                                       [B]
       │
       ├─ local buffer (temporary table)
       │     └─ MarkLocalBufferDirty: if not BM_DIRTY → local_blks_dirtied++   [C]
       │
       ├─ BM_DIRTY and BM_JUST_DIRTIED both set → return, nothing counted      [D]
       │
       ├─ XLogHintBitIsNeeded() and BM_PERMANENT
       │     ├─ RecoveryInProgress() → return: bit set, page NOT dirtied,
       │     │                         nothing counted                         [E]
       │     └─ XLogSaveBufferForHint
       │           ├─ page LSN ≤ RedoRecPtr → XLOG_FPI_FOR_HINT record in WAL   [F]
       │           │                          (visible to pg_waldump --stats)
       │           └─ page LSN > RedoRecPtr → no record
       │
       └─ lock header; if not BM_DIRTY → dirtied = true                         [G]
             set BM_DIRTY | BM_JUST_DIRTIED
             if dirtied → VacuumPageDirty++, pgBufferUsage.shared_blks_dirtied++
                    │
                    ├─ EXPLAIN (ANALYZE, BUFFERS)  "dirtied=" per plan node     [H]
                    ├─ pg_stat_statements           shared_blks_dirtied         [I]
                    ├─ auto_explain (log_buffers)   "dirtied=" in the log       [J]
                    └─ autovacuum log               "N dirtied"                 [K]

Not reached by any path above: pg_stat_database, pg_statio_*, pg_stat_bgwriter [L]
Current-state only:  pageinspect t_infomask, pg_buffercache isdirty           [M]
```

Key to the map:

- [A] [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131). The other hint writers include index `LP_DEAD` marks, called [killed index tuples](../../../glossary.md#killed-index-tuple) ([nbtutils.c:1810](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1810)), `pd_prune_xid` ([pruneheap.c:290](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L290)) and the [free space map](../../../glossary.md#free-space-map) ([freespace.c:221](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L221), [fsmpage.c:287](../../../../raw/postgres-12/src/backend/storage/freespace/fsmpage.c#L287)). The [statements page](../storage-and-vacuum/client-statements-that-set-hint-bits.md#logic-map) lists them all.
- [B] [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3422-L3548).
- [C] [bufmgr.c:3430-3434](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3430-L3434), [localbuf.c#MarkLocalBufferDirty](../../../../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L280-L305).
- [D] [bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454).
- [E] [bufmgr.c:3480-3481](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3480-L3481).
- [F] [bufmgr.c:3506-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3506-L3507), [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L893-L957).
- [G] [bufmgr.c:3509-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3509-L3546). The page LSN takes an image's LSN only in the clean-buffer branch ([bufmgr.c:3514-3533](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3514-L3533)).
- [H] [explain.c:1864-1866](../../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c:2899-2901](../../../../raw/postgres-12/src/backend/commands/explain.c#L2899-L2901).
- [I] [pg_stat_statements.c:1283](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1283), [pg_stat_statements.c:1641](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1641).
- [J] [auto_explain.c:281-282](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L281-L282), [auto_explain.c:380](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L380).
- [K] [vacuumlazy.c:430-434](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L430-L434).
- [L] [system_views.sql#pg_statio_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L622-L639), [system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L880), [system_views.sql#pg_stat_bgwriter](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947).
- [M] [rawpage.c:161-166](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L161-L166), [pg_buffercache_pages.c:164-167](../../../../raw/postgres-12/contrib/pg_buffercache/pg_buffercache_pages.c#L164-L167).

### Comparison of the available metrics

| Metric | Where | What it counts | Includes hint dirtying? | Separates hints from other changes? | Needs |
|---|---|---|---|---|---|
| `dirtied=` | `EXPLAIN (ANALYZE, BUFFERS)` | Shared and local buffers that went clean → dirty while each plan node ran, children included ([explain.c:2890-2902](../../../../raw/postgres-12/src/backend/commands/explain.c#L2890-L2902), [explain.c:2908-2919](../../../../raw/postgres-12/src/backend/commands/explain.c#L2908-L2919), [explain.sgml:181-188](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L181-L188)) | Yes | No | Runs the statement. `BUFFERS` requires `ANALYZE` in 12 ([explain.c:206-209](../../../../raw/postgres-12/src/backend/commands/explain.c#L206-L209)) |
| `shared_blks_dirtied`, `local_blks_dirtied` | `pg_stat_statements` view | Cumulative per normalized statement ([pg_stat_statements--1.4.sql:26-30](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.4.sql#L26-L30), [pgstatstatements.sgml:149-180](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L149-L180)) | Yes | No | Module in `shared_preload_libraries` ([pg_stat_statements.c:359-360](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L359-L360)) and `CREATE EXTENSION` |
| `dirtied=` in logged plans | `auto_explain` | Same as `EXPLAIN`, for statements over the duration threshold | Yes | No | Off by default; see [Cases that change the counts](#cases-that-change-the-counts) |
| `buffer usage: … N dirtied` | Autovacuum log line | Buffers dirtied during one autovacuum `vacuum()` call ([vacuumlazy.c:373-379](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L373-L379), [vacuumlazy.c:430-434](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L430-L434)) | Yes, including VACUUM's own hints | No; [pruning](../../../glossary.md#pruning) ([pruneheap.c:256](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L256)) and [freezing](../../../glossary.md#freezing) ([vacuumlazy.c:1211](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1211)) dirty pages too | `log_autovacuum_min_duration ≥ 0`, default `-1` |
| `XLOG/FPI_FOR_HINT` row | `pg_waldump --stats=record` | Hint images written to WAL ([pg_waldump.c:729-751](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L729-L751)) | Only on clusters with checksums or `wal_log_hints` on | **Yes**, but counts images, not bits or every hint-dirtied page | File access to `pg_wal`, and a non-default cluster setting |
| `t_infomask` bits | `pageinspect` `heap_page_items(get_raw_page(...))` | Hint bits present on tuples now | n/a, a snapshot of page state | Shows the bits themselves | Superuser ([rawpage.c:103-106](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106)) |
| `prune_xid` | `pageinspect` `page_header()` | The page's `pd_prune_xid` hint field now ([pageinspect--1.5.sql:22-33](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L22-L33)) | n/a | Shows one non-tuple hint | Superuser |
| `isdirty` | `pg_buffercache` view | Whether each shared buffer is dirty now ([pgbuffercache.sgml:97-100](../../../../raw/postgres-12/doc/src/sgml/pgbuffercache.sgml#L97-L100)) | Yes, as current state | No | Superuser or `pg_monitor` ([pg_buffercache--1.2--1.3.sql:6-7](../../../../raw/postgres-12/contrib/pg_buffercache/pg_buffercache--1.2--1.3.sql#L6-L7)) |
| `blks_read`, `blks_hit`, `heap_blks_*`, `buffers_checkpoint`, `buffers_clean`, `buffers_backend` | `pg_stat_database`, `pg_statio_*`, `pg_stat_bgwriter` | Reads and hits in `pg_stat_database` and `pg_statio_*`; writes in `pg_stat_bgwriter` ([system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L880), [system_views.sql#pg_statio_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L622-L639), [monitoring.sgml:2440-2458](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2440-L2458)) | No dirtying counter at all | No | — |

Hint-dirtied shared buffers written by the [background writer](../../../glossary.md#background-writer), the checkpointer, or a backend add to `pg_stat_bgwriter`'s write counters ([bufmgr.c:1991-1994](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1991-L1994), [bufmgr.c:2300](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2300), [monitoring.sgml:2440-2458](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2440-L2458)). Those counters cannot tell a hint-only page from any other dirty page. Two kinds of page can stay out of them. A temporary table's local buffers are written with only `local_blks_written` counted ([localbuf.c:206-227](../../../../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L206-L227)). An unlogged table's buffers are skipped by a normal checkpoint, though not by a shutdown or end-of-recovery checkpoint ([bufmgr.c:1805-1810](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1805-L1810)).

### How the dirtied counter is incremented

**Input.** A hint writer holds a pin and at least a share lock on a buffer, has changed the page in memory, and calls `MarkBufferDirtyHint()` ([bufmgr.c:3422-3438](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3422-L3438)).

**Decision 1: is it a temporary-table buffer?** If so, `MarkLocalBufferDirty()` increments `local_blks_dirtied` only when the buffer was not already dirty. It never writes WAL ([bufmgr.c:3430-3434](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3430-L3434), [localbuf.c:298-303](../../../../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L298-L303)). Therefore temporary-table hints show up under `local_blks_dirtied`, never `shared_blks_dirtied`.

**Decision 2: is the buffer already dirty and unchanged since its write started?** An unlocked test of `BM_DIRTY | BM_JUST_DIRTIED` returns at once when both are set ([bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454)). So the second and later hints on an already dirty page add nothing while the buffer stays dirty. This is why one page normally counts once, however many bits a scan sets on it. A scan checks visibility under a share content lock ([heapam.c:401-457](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L401-L457)), and a buffer write takes a share lock too ([bufmgr.c:2399](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2399)), so a write can finish between two hints. The next hint then counts the page again.

**Decision 3: does the cluster WAL-log hints?** When `XLogHintBitIsNeeded()` is true and the buffer is permanent, two things can happen. During recovery the function returns before dirtying, so nothing is counted ([bufmgr.c:3470-3481](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3481)). Otherwise `XLogSaveBufferForHint()` may write an `FPI_FOR_HINT` record ([bufmgr.c:3506-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3506-L3507)). That record is the only place a hint leaves a hint-specific trace.

**Calculation.** Under the buffer-header lock the function sets `dirtied = true` only if `BM_DIRTY` was clear. It then sets both flags and, when `dirtied` is true, increments `VacuumPageDirty` and `pgBufferUsage.shared_blks_dirtied` ([bufmgr.c:3509-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3509-L3546)).

**Output.** One increment per clean-to-dirty transition. `MarkBufferDirty()`, which WAL-logged changes use, increments the same two counters under the same condition ([bufmgr.c:1494-1502](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1494-L1502)). **Consequence:** the counter cannot say which function dirtied a page. A plain read-only `SELECT` that reports `dirtied=` above zero changed pages through some hint writer or through a WAL-logged prune ([pruneheap.c:256](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L256)); the [statements page](../storage-and-vacuum/client-statements-that-set-hint-bits.md#how-to-see-it-happen) lists those writers. A `SELECT` that locks rows or calls a function that writes also dirties pages through `MarkBufferDirty()`, as `heap_lock_tuple()` does ([heapam.c:4589](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L4589)).

### Where each reporter takes its numbers from

`pgBufferUsage` is a running total per backend. Each reporter takes a difference over its own window, so the windows decide what is included.

| Reporter | Window | Included | Not included |
|---|---|---|---|
| `EXPLAIN (ANALYZE, BUFFERS)` per node | Each `ExecProcNode()` call on that node, via `InstrStartNode()`/`InstrStopNode()` ([execProcnode.c#ExecProcNodeInstr](../../../../raw/postgres-12/src/backend/executor/execProcnode.c#L454-L466), [instrument.c:61-100](../../../../raw/postgres-12/src/backend/executor/instrument.c#L61-L100)) | Scans and modifications done inside plan nodes; [parallel](../../../glossary.md#parallel-query) workers' usage added to the leader ([execParallel.c:1114](../../../../raw/postgres-12/src/backend/executor/execParallel.c#L1114), [instrument.c:181-185](../../../../raw/postgres-12/src/backend/executor/instrument.c#L181-L185)) | Planning, executor startup, and after-trigger processing at `ExecutorFinish`, such as [foreign-key](../../../glossary.md#foreign-key-trigger) checks, because those run outside `ExecProcNode()` ([execMain.c:431-442](../../../../raw/postgres-12/src/backend/executor/execMain.c#L431-L442)). 12 prints buffer usage only per plan node ([explain.c:1864-1866](../../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866)) |
| `pg_stat_statements`, optimizable statement | `queryDesc->totaltime`, started and stopped around `ExecutorRun` and `ExecutorFinish` ([pg_stat_statements.c:864-876](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L864-L876), [execMain.c:335-336](../../../../raw/postgres-12/src/backend/executor/execMain.c#L335-L336), [execMain.c:381-382](../../../../raw/postgres-12/src/backend/executor/execMain.c#L381-L382), [execMain.c:431-442](../../../../raw/postgres-12/src/backend/executor/execMain.c#L431-L442)) | Execution and after-trigger work, stored at `ExecutorEnd` ([pg_stat_statements.c:943-950](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L943-L950)) | Parse, analysis and planning catalog lookups, and `ExecutorStart` |
| `pg_stat_statements`, [utility statement](../../../glossary.md#utility-command) | Whole `ProcessUtility` call, when `pg_stat_statements.track_utility` is on (its default) ([pg_stat_statements.c:984-996](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L984-L996), [pg_stat_statements.c:1027-1042](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1027-L1042), [pg_stat_statements.c:390-399](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L390-L399)) | Manual `VACUUM`, `ANALYZE`, `CREATE INDEX`, `CLUSTER` and other utility work | `EXECUTE`, `PREPARE` and `DEALLOCATE` are excluded from this path |
| `auto_explain` | Same per-node instrumentation as `EXPLAIN`, added in its `ExecutorStart` hook ([auto_explain.c:270-284](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L270-L284)) | Same as `EXPLAIN` | Same as `EXPLAIN` |
| Autovacuum log | From the reset at the start of `vacuum()` to the log call ([vacuum.c:380-385](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L380-L385), [vacuumlazy.c:373-434](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L373-L434)) | All dirtying in that autovacuum run | Autovacuum calls `vacuum()` directly, not through `ProcessUtility`, so `pg_stat_statements` never sees it ([autovacuum.c:3112-3117](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3112-L3117)) |

### Cases that change the counts

| Condition / state | Behavior | Consequence for the metrics |
|---|---|---|
| Default cluster: checksums off, `wal_log_hints = off` | `XLogHintBitIsNeeded()` is false, so hints only dirty buffers ([xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192), [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185)) | Dirtied counters move; `pg_waldump` finds no `FPI_FOR_HINT` |
| Checksums on (`initdb -k`, or `pg_checksums --enable` on a shut-down cluster) or `wal_log_hints = on` | The WAL branch runs for permanent buffers ([bufmgr.c:3470-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3507)) | `FPI_FOR_HINT` records appear, normally one per page per checkpoint cycle; see [Key interaction](#key-interaction-pages-bits-and-images-are-three-different-units) |
| Buffer already `BM_DIRTY` and `BM_JUST_DIRTIED` | Fast return ([bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454)) | No increment, no image |
| Commit not yet flushed, permanent buffer, page LSN older than the commit LSN | Default: `SetHintBits()` sets the bit. Cause: the tuple's transaction used [asynchronous commit](../../../glossary.md#asynchronous-commit) and its commit record is still unflushed. Mechanism: the `XLogNeedsFlush()` test returns before the OR and before `MarkBufferDirtyHint()` ([heapam_visibility.c:117-127](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L117-L127)) | No bit, no dirtying, no count; a later check can set the hint |
| Hint arrives while the buffer is being written | `FlushBuffer()` clears `BM_JUST_DIRTIED` before writing ([bufmgr.c:2715](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2715)); the hint finds `BM_DIRTY` still set, so `dirtied` stays false, but it sets `BM_JUST_DIRTIED`, which stops the write from clearing `BM_DIRTY` ([bufmgr.c:3513-3535](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3513-L3535), [bufmgr.c:3996-3997](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3996-L3997)) | The page stays dirty but the counter does not count it again |
| [Hot standby](../../../glossary.md#hot-standby) with the WAL branch active | Returns before dirtying ([bufmgr.c:3480-3481](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3480-L3481)) | No increment. If the buffer is clean, the hint is lost at eviction. A buffer already dirty with both flags takes the fast return instead ([bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454)), so the hint is written with the page |
| Hot standby, default cluster | `XLogHintBitIsNeeded()` is false, so the recovery return is skipped and the normal dirtying path runs ([bufmgr.c:3470-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3507), [bufmgr.c:3509-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3509-L3546)) | Increments as on a primary; no WAL |
| [Unlogged](../../../glossary.md#unlogged-relation) table | Not `BM_PERMANENT`, skips the WAL branch ([bufmgr.c:3470-3471](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3471)) | `shared_blks_dirtied` moves; no image even with checksums |
| Temporary table | Local buffer path ([bufmgr.c:3430-3434](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3430-L3434)) | Counted under `local_blks_dirtied` |
| Racing unlocked fast-path test | The header comment says the function may fail to mark the buffer dirty ([bufmgr.c:3417-3420](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3417-L3420), [bufmgr.c:3442-3452](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3442-L3452)) | A hint can, rarely, be neither dirtied nor counted |

**Non-default reporters.** Three of the reporters log or collect nothing until a setting is changed:

| Reporter | Default | Cause | Mechanism |
|---|---|---|---|
| `auto_explain` buffer output | `auto_explain.log_min_duration = -1` (disabled), `log_analyze = false`, `log_buffers = false` ([auto_explain.c:95-115](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L95-L115), [auto_explain.c:140-149](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L140-L149)) | Load the module with `LOAD` as a superuser or with a preload setting ([auto-explain.sgml:20-30](../../../../raw/postgres-12/doc/src/sgml/auto-explain.sgml#L20-L30)), and set all three: `log_min_duration ≥ 0`, `log_analyze = on`, `log_buffers = on` | `auto_explain_enabled()` needs `log_min_duration ≥ 0` ([auto_explain.c:66-69](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L66-L69)); buffer instrumentation is requested only when `log_analyze` and `log_buffers` are both on ([auto_explain.c:273-282](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L273-L282)) |
| Autovacuum "dirtied" | `log_autovacuum_min_duration = -1` ([guc.c:2715-2724](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2715-L2724)) | Set the GUC, or the table's `log_autovacuum_min_duration` storage parameter (see the note after the GUC scope table), to `0` or a positive threshold in milliseconds | The log line is emitted only for an autovacuum worker with `log_min_duration ≥ 0` and a run longer than the threshold ([vacuumlazy.c:373-379](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L373-L379), [autovacuum.c:2861-2864](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2861-L2864)) |
| `pg_stat_statements` | Not loaded: `shared_preload_libraries` defaults to empty ([guc.c:3773-3774](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3773-L3774)) | Add it to `shared_preload_libraries` and restart, then `CREATE EXTENSION pg_stat_statements` | `_PG_init()` returns without installing hooks unless loaded at preload time ([pg_stat_statements.c:359-360](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L359-L360)) |

[GUC context](../../../glossary.md#guc-context) for these settings, from their definitions:

| Setting | Context | What a change needs |
|---|---|---|
| `shared_preload_libraries` | `PGC_POSTMASTER` ([guc.c:3768](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3768)) | Restart |
| `log_autovacuum_min_duration` | `PGC_SIGHUP` ([guc.c:2716](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2716)) | Reload |
| `auto_explain.log_min_duration`, `log_analyze`, `log_buffers` | `PGC_SUSET` ([auto_explain.c:101](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L101), [auto_explain.c:112](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L112), [auto_explain.c:145](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L145)) | Session or transaction scope with `SET`, for a superuser only |
| `pg_stat_statements.track_utility` | `PGC_SUSET` ([pg_stat_statements.c:395](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L395)) | Session or transaction scope with `SET`, for a superuser only |
| `wal_log_hints` | `PGC_POSTMASTER` ([guc.c:1178](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1178)) | Restart |

The per-table override is the [storage parameter](../../../glossary.md#storage-parameter) `log_autovacuum_min_duration`. It is set with `ALTER TABLE`, which needs no restart or reload, and it defaults to `-1` ([reloptions.c:296-304](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L296-L304), [reloptions.c:1403-1404](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L1403-L1404)). Its value replaces the GUC when it is `≥ 0` ([autovacuum.c:2861-2864](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2861-L2864)).

Turning on `wal_log_hints` only to count hints has a cost: a hint that reaches the WAL branch on a page with no WAL-logged change since the redo start then writes a full-page image ([bufmgr.c:3470-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3507), [xloginsert.c:917](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L917)). The [statements page](../storage-and-vacuum/client-statements-that-set-hint-bits.md#measured-results) publishes measurements of that cost with its script.

### Lifecycle events

| Event | State before | State after | Effect on later counts |
|---|---|---|---|
| Buffer written out | `BM_DIRTY` set | `BM_DIRTY` cleared, unless a change set `BM_JUST_DIRTIED` during the write ([bufmgr.c:3996-3997](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3996-L3997)) | Once cleared, the next hint on that page increments the dirtied counter again |
| Checkpoint starts | Page LSN may be past the old redo pointer | `RedoRecPtr` advances | On a WAL-branch cluster, the next hint on each page writes a new `FPI_FOR_HINT` ([xloginsert.c:907-917](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L907-L917)), except that a page still dirty with both flags set takes the fast return ([bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454)) until it has been written |
| Hint already set on a tuple | Bit absent | Bit present | Later checks skip `SetHintBits()`, so repeated reads add nothing ([heapam_visibility.c:970](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L970)) |
| New statement or new plan node loop | Counter window open | Window closed and summed | `EXPLAIN` and `pg_stat_statements` report the difference for that window only ([instrument.c:61-100](../../../../raw/postgres-12/src/backend/executor/instrument.c#L61-L100)) |
| `vacuum()` call starts | `VacuumPageDirty` from an earlier run | Reset to 0 ([vacuum.c:385](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L385)) | The autovacuum log line covers this run only |
| `pg_stat_statements_reset()` | Accumulated totals | Entries removed; the 1.7 default version redefines the function with optional arguments ([pg_stat_statements--1.6--1.7.sql:13-19](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements--1.6--1.7.sql#L13-L19)) | Totals start again from 0 |
| Restart with `wal_log_hints` changed | Old predicate | New predicate | Decides whether `FPI_FOR_HINT` records can appear ([xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192)) |
| `pg_checksums --enable` or `--disable` on a shut-down cluster | Old checksum state in `pg_control` | New state ([pg_checksums.c:599](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L599)) | Same decision, from the next start ([xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192)) |

### Key interaction: pages, bits and images are three different units

`SetHintBits()` works per tuple, `MarkBufferDirtyHint()` counts per buffer transition, and `XLogSaveBufferForHint()` normally writes once per page per checkpoint cycle. Because each step collapses repeats from the step before, the three numbers diverge:

- Suppose, as a hypothetical illustration, that a scan sets 15 hint bits on each of 94 clean pages, and that no write of those buffers finishes and no other backend hints them during the scan. It makes 1,410 `SetHintBits()` calls. It adds at most 94 to `shared_blks_dirtied`: only the first call per page passes the `BM_DIRTY` test ([bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454), [bufmgr.c:3513-3516](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3513-L3516)).
- On a WAL-branch cluster it writes at most 94 `FPI_FOR_HINT` records, and fewer for pages already changed since the checkpoint's redo start ([xloginsert.c:917](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L917)).
- If some of those pages were already dirty from an earlier `INSERT`, the scan's `dirtied=` is lower still, even though it set the same 1,410 bits.

The `FPI_FOR_HINT` count and the dirtied count are not nested either. A hint on a page that is mid-write keeps `BM_DIRTY` set and so is not counted as dirtied, yet it can still write an image when the page LSN is at or before the redo pointer ([bufmgr.c:3470-3516](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3516)). That image also leaves the page LSN unchanged, because the function sets the LSN only when the buffer was clean ([bufmgr.c:3514-3533](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3514-L3533)). So a later hint can write another image for the same page in the same checkpoint cycle. Concurrent backends can do the same: the source says multiple copies of one block can be recorded in separate records ([xloginsert.c:887-890](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L887-L890)). The `FPI_FOR_HINT` count also includes every non-tuple hint writer, such as free space map updates ([freespace.c:221](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L221)) and index `LP_DEAD` marks ([nbtutils.c:1810](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1810)). Therefore no metric in 12 gives the number of hint bits set; `pageinspect` can only count the bits present now.

### Example queries

These are diagnostic examples. They set timeouts with `SET LOCAL`, which lasts until the transaction ends; `statement_timeout` and `lock_timeout` are `PGC_USERSET`, so any user can do this ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)).

**Statements that dirty the most buffers** (`pg_stat_statements` installed). A read-only statement with a high `shared_blks_dirtied` is a likely hint writer, because a pure read has no WAL-logged change of its own, though it can still prune ([pruneheap.c:256](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L256)):

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';
SELECT /* wiki_top_dirtying_statements */ calls, shared_blks_dirtied, local_blks_dirtied,
       shared_blks_written, left(query, 80) AS query
FROM pg_stat_statements
ORDER BY shared_blks_dirtied DESC
LIMIT 20;
COMMIT;
```

**Hint bits present now on one table** (superuser, `pageinspect` installed). This reads every block of the main fork into shared buffers and copies it; it calls no visibility routine, so it sets no hints itself ([rawpage.c:161-166](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L161-L166)). Replace `my_table`; on a large table the scan is expensive:

```sql
BEGIN;
SET LOCAL statement_timeout = '60s';
SET LOCAL lock_timeout = '5s';
SELECT /* wiki_count_hint_bits */
       count(*) FILTER (WHERE lp_flags = 1)                                   AS normal_tuples,
       count(*) FILTER (WHERE (t_infomask & 768) = 256)                       AS xmin_committed,
       count(*) FILTER (WHERE (t_infomask & 768) = 512)                       AS xmin_invalid,
       count(*) FILTER (WHERE (t_infomask & 768) = 768)                       AS xmin_frozen,
       count(*) FILTER (WHERE lp_flags = 1 AND (t_infomask & 768) = 0)        AS xmin_unhinted,
       count(*) FILTER (WHERE (t_infomask & 1024) <> 0)                       AS xmax_committed,
       count(*) FILTER (WHERE (t_infomask & 2048) <> 0)                       AS xmax_invalid
FROM generate_series(0, (pg_relation_size('my_table') / current_setting('block_size')::int)::int - 1) AS blk,
     LATERAL heap_page_items(get_raw_page('my_table', blk));
COMMIT;
```

The masks are the `htup_details.h` values: 256 = `HEAP_XMIN_COMMITTED`, 512 = `HEAP_XMIN_INVALID`, 768 = both (frozen), 1024 = `HEAP_XMAX_COMMITTED`, 2048 = `HEAP_XMAX_INVALID` ([htup_details.h:203-207](../../../../raw/postgres-12/src/include/access/htup_details.h#L203-L207)). `lp_flags = 1` is `LP_NORMAL`, a [line pointer](../../../glossary.md#line-pointer) state ([itemid.h:38-41](../../../../raw/postgres-12/src/include/storage/itemid.h#L38-L41)). Running the query twice, before and after a workload, gives the net change in present bits, not the number of hint writes. Some of those bits come from no hint writer: every insert sets `HEAP_XMAX_INVALID` ([heapam.c:2067](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2067)), and a frozen insert sets both xmin bits ([heapam.c:2069-2070](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2069-L2070)). So `xmax_invalid` and `xmin_frozen` overstate what hints set.

**Dirty buffers of one table now** (`pg_buffercache` installed, superuser or `pg_monitor`):

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';
SELECT /* wiki_dirty_buffers_for_table */ b.relforknumber, count(*) FILTER (WHERE b.isdirty) AS dirty, count(*) AS cached
FROM pg_buffercache b
WHERE b.relfilenode = pg_relation_filenode('my_table')
  AND b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
  AND b.reltablespace = (SELECT COALESCE(NULLIF(c.reltablespace, 0), d.dattablespace)
                         FROM pg_class c, pg_database d
                         WHERE c.oid = 'my_table'::regclass AND d.datname = current_database())
GROUP BY b.relforknumber;
COMMIT;
```

**Hint images in a WAL range** (shell, cluster with checksums or `wal_log_hints` on). Take `pg_current_wal_lsn()` before and after the workload, then:

```text
pg_waldump --stats=record -p "$PGDATA/pg_wal" -s <start-lsn> -e <end-lsn>
```

Read the `XLOG/FPI_FOR_HINT` row: `N` is the number of hint images ([pg_waldump.c:707](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L707), [pg_waldump.c:826-838](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L826-L838), [pg_waldump.sgml:183-191](../../../../raw/postgres-12/doc/src/sgml/ref/pg_waldump.sgml#L183-L191)). Other writers can add WAL in the same range, but only hint writers produce `FPI_FOR_HINT`: its one insert site is in `XLogSaveBufferForHint()`, whose one caller is `MarkBufferDirtyHint()` ([xloginsert.c:953](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L953), [bufmgr.c:3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3507)).

### Causal summary

**Initial state:** a tuple's inserting or deleting transaction has finished, no hint bit records that outcome yet, and the buffer is clean.

**Event:** a statement's visibility check, or another hint writer, sets a bit and calls `MarkBufferDirtyHint()`. A commit that is not yet flushed can make `SetHintBits()` skip both.

**Transformation:** `SetHintBits()` counts nothing. `MarkBufferDirtyHint()` increments `shared_blks_dirtied` and `VacuumPageDirty` (for a temporary table, only `local_blks_dirtied`) once per clean-to-dirty transition. It counts nothing if the buffer was already dirty, or if a standby with checksums or `wal_log_hints` declines to dirty it.

**Interaction:** WAL-logged changes increment the same counters, so `EXPLAIN (ANALYZE, BUFFERS)`, `pg_stat_statements`, `auto_explain` and the autovacuum log report dirtied pages from all causes. Only on a cluster with checksums or `wal_log_hints` does a hint leave a hint-specific trace: an `XLOG_FPI_FOR_HINT` record, normally once per page per checkpoint cycle, countable with `pg_waldump --stats=record`.

**Observed result:** no view counts hint bits set or pages dirtied by hints. The closest measures are the all-cause dirtied counters, the `FPI_FOR_HINT` record count, and point-in-time inspection with `pageinspect` and `pg_buffercache`.

**Refresh:** a buffer write clears `BM_DIRTY` and re-arms the dirtied count for that page, unless a change arrived during the write. A checkpoint re-arms the hint image for a page, which takes effect once a hint gets past the fast return.

## Context Reviewed

- Version: PostgreSQL 12, pinned checkout `raw/postgres-12/` at `45b88269a353ad93744772791feb6d01bc7e1e42` (`git rev-parse HEAD` matched `wiki/versions.md`).
- Counting code: `bufmgr.c` (`MarkBufferDirty`, `MarkBufferDirtyHint`, `FlushBuffer`, `TerminateBufferIO`), `localbuf.c#MarkLocalBufferDirty`, `buf_internals.h`, `instrument.h`, `instrument.c`, `execProcnode.c#ExecProcNodeInstr`, `execMain.c` (`standard_ExecutorRun`, `standard_ExecutorFinish`), `execParallel.c`, `globals.c`/`miscadmin.h` (`VacuumPageDirty`), `vacuum.c`, `vacuumlazy.c`, `autovacuum.c`.
- Hint writers: `heapam_visibility.c#SetHintBits`, `pruneheap.c`, `nbtutils.c`, `freespace.c`, `fsmpage.c`; the full caller list was reviewed for the [statements page](../storage-and-vacuum/client-statements-that-set-hint-bits.md) and is linked, not repeated.
- WAL side: `xloginsert.c#XLogSaveBufferForHint`, `xlog.h#XLogHintBitIsNeeded`, `pg_control.h`, `xlogdesc.c`, `pg_waldump.c`, `pg_checksums.c`, `initdb.c`.
- Bit origins and locks: `heapam.c` (`heapgetpage`, `heap_prepare_insert`, `heap_lock_tuple`), `SyncOneBuffer` in `bufmgr.c`.
- Reporters: `explain.c`, `doc/src/sgml/ref/explain.sgml`; `contrib/pg_stat_statements` (C source, `--1.4.sql`, control file version 1.7, `pgstatstatements.sgml`); `contrib/auto_explain` (C source, `auto-explain.sgml`); `contrib/pageinspect` (`rawpage.c`, `heapfuncs.c`, `--1.5.sql`, `--1.5--1.6.sql`, `pageinspect.sgml`); `contrib/pg_buffercache` (C source, `--1.2.sql`, `--1.2--1.3.sql`, `pgbuffercache.sgml`); `doc/src/sgml/ref/pg_waldump.sgml`.
- Absence checks: a search of `pgstat.c`, `pgstat.h`, `pgstatfuncs.c` and `system_views.sql` for `dirt` found nothing, and for `hint` found only `getaddrinfo` hints and a hash-table size comment in `pgstat.c`. `probes.d` has buffer read, flush, write-dirty and checkpoint probes, but none for dirtying or hints ([probes.d:58-70](../../../../raw/postgres-12/src/backend/utils/probes.d#L58-L70)). `pageinspect` 1.7 has no infomask-decoding function, and `bt_page_items` returns no line-pointer flag ([pageinspect--1.5--1.6.sql:90-98](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5--1.6.sql#L90-L98)).
- GUC definitions: `guc.c` (`wal_log_hints`, `log_autovacuum_min_duration`, `shared_preload_libraries`, `statement_timeout`, `lock_timeout`); the `log_autovacuum_min_duration` storage parameter in `reloptions.c`; custom GUCs in `auto_explain.c` and `pg_stat_statements.c`.
- Build and generated headers: none of the cited symbols comes from a generated header. `pg_stat_statements`, `auto_explain`, `pageinspect` and `pg_buffercache` are [contrib](../../../glossary.md#contrib) modules built separately from the core server.
- Tests: `contrib/pg_stat_statements/sql/` and `src/test/regress/` were not found to assert a `shared_blks_dirtied` value, and no test asserts `FPI_FOR_HINT` counts; see Open Questions.
- Common concept pages: `wiki/v12/common-concepts/` does not exist, so none was linked.
- No measurement was run for this page, and it reports no measured number. The linked statements page publishes its own measurements and script.

## Evidence Map

| Claim | Evidence |
|---|---|
| `SetHintBits()` counts nothing | [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131) |
| No dirtied column in the cumulative statistics views | [system_views.sql#pg_statio_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L622-L639), [system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L880), [system_views.sql#pg_stat_bgwriter](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947) |
| Both dirtying functions increment the same counters on clean → dirty | [bufmgr.c:1494-1502](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1494-L1502), [bufmgr.c:3509-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3509-L3546) |
| Local buffers count under `local_blks_dirtied` | [bufmgr.c:3430-3434](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3430-L3434), [localbuf.c:298-303](../../../../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L298-L303) |
| Already-dirty fast return; recovery return | [bufmgr.c:3453-3454](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3453-L3454), [bufmgr.c:3480-3481](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3480-L3481) |
| Mid-write hint keeps the page dirty without counting | [bufmgr.c:2715](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2715), [bufmgr.c:3996-3997](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3996-L3997) |
| One page can get more than one image in a checkpoint cycle | [bufmgr.c:3514-3533](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3514-L3533), [xloginsert.c:887-890](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L887-L890) |
| A page can count as dirtied more than once in one scan | [heapam.c:401-457](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L401-L457), [bufmgr.c:2399](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2399), [bufmgr.c:3996-3997](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3996-L3997) |
| `SetHintBits()` can return without setting a bit | [heapam_visibility.c:117-127](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L117-L127) |
| Not every bit present came from a hint writer | [heapam.c:2067](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2067), [heapam.c:2069-2070](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2069-L2070) |
| Checksums can change on a shut-down cluster | [pg_checksums.c:383-384](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L383-L384), [pg_checksums.c:516-519](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L516-L519), [pg_checksums.c:599](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L599) |
| `VacuumPageDirty` is not read by cost accounting | [bufmgr.c:3545-3546](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3545-L3546), [vacuumlazy.c:392](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L392) |
| The per-table log setting is a storage parameter | [reloptions.c:296-304](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L296-L304), [reloptions.c:1403-1404](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L1403-L1404) |
| `FPI_FOR_HINT` written only on the WAL branch and when page LSN ≤ redo pointer | [bufmgr.c:3470-3507](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3507), [xloginsert.c:907-917](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L907-L917), [xloginsert.c:953](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L953) |
| `pg_waldump --stats=record` counts records per type | [pg_waldump.c:971-983](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L971-L983), [pg_waldump.c:729-751](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L729-L751), [xlogdesc.c:185-186](../../../../raw/postgres-12/src/backend/access/rmgrdesc/xlogdesc.c#L185-L186) |
| EXPLAIN node windows and output | [execProcnode.c#ExecProcNodeInstr](../../../../raw/postgres-12/src/backend/executor/execProcnode.c#L454-L466), [instrument.c:61-100](../../../../raw/postgres-12/src/backend/executor/instrument.c#L61-L100), [explain.c:1864-1866](../../../../raw/postgres-12/src/backend/commands/explain.c#L1864-L1866), [explain.c:2890-2902](../../../../raw/postgres-12/src/backend/commands/explain.c#L2890-L2902) |
| `pg_stat_statements` windows | [pg_stat_statements.c:864-876](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L864-L876), [pg_stat_statements.c:943-950](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L943-L950), [pg_stat_statements.c:984-996](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L984-L996), [execMain.c:431-442](../../../../raw/postgres-12/src/backend/executor/execMain.c#L431-L442) |
| Autovacuum log and its gate | [vacuumlazy.c:373-379](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L373-L379), [vacuumlazy.c:430-434](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L430-L434), [guc.c:2715-2724](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2715-L2724) |
| `auto_explain` defaults and gate | [auto_explain.c:66-69](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L66-L69), [auto_explain.c:95-115](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L95-L115), [auto_explain.c:140-149](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L140-L149), [auto_explain.c:273-282](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L273-L282) |
| `pageinspect` exposes `t_infomask`, needs superuser, copies without visibility checks | [pageinspect--1.5.sql:38-55](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L38-L55), [rawpage.c:103-106](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106), [rawpage.c:161-166](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L161-L166) |
| `pg_buffercache.isdirty` reads `BM_DIRTY` | [pg_buffercache_pages.c:164-167](../../../../raw/postgres-12/contrib/pg_buffercache/pg_buffercache_pages.c#L164-L167), [pg_buffercache--1.2--1.3.sql:6-7](../../../../raw/postgres-12/contrib/pg_buffercache/pg_buffercache--1.2--1.3.sql#L6-L7) |

## Open Questions

- No test in the pinned checkout was found that asserts a `shared_blks_dirtied` or `dirtied=` value, or an `FPI_FOR_HINT` count. The counting behavior on this page is a source reading. The linked statements page measured `dirtied=` and `FPI_FOR_HINT` for a single `SELECT`, but no measurement here checks `pg_stat_statements` or the autovacuum log line.
- The source comment calls the `MarkBufferDirtyHint()` fast-path race harmless and rare ([bufmgr.c:3442-3452](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3442-L3452)). How often it drops a count in practice is not established. Neither is how often one page gets a second `FPI_FOR_HINT` in a checkpoint cycle, or a second dirtied count in one scan.

## Source References

- [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L112-L131)
- [bufmgr.c#MarkBufferDirty](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1457-L1503)
- [bufmgr.c#MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3422-L3548)
- [localbuf.c#MarkLocalBufferDirty](../../../../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L280-L305)
- [instrument.h#BufferUsage](../../../../raw/postgres-12/src/include/executor/instrument.h#L19-L33)
- [instrument.c#InstrStartNode](../../../../raw/postgres-12/src/backend/executor/instrument.c#L61-L100)
- [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L893-L957)
- [heapam.c:2067](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2067)
- [pruneheap.c:256](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L256)
- [pg_checksums.c:383-384](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L383-L384)
- [reloptions.c:296-304](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L296-L304)
- [explain.c:2890-2902](../../../../raw/postgres-12/src/backend/commands/explain.c#L2890-L2902)
- [pg_stat_statements.c:943-950](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L943-L950)
- [auto_explain.c:270-284](../../../../raw/postgres-12/contrib/auto_explain/auto_explain.c#L270-L284)
- [vacuumlazy.c:430-434](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L430-L434)
- [pg_waldump.c:729-751](../../../../raw/postgres-12/src/bin/pg_waldump/pg_waldump.c#L729-L751)
- [pageinspect--1.5.sql:38-55](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L38-L55)
- [pg_buffercache_pages.c:164-167](../../../../raw/postgres-12/contrib/pg_buffercache/pg_buffercache_pages.c#L164-L167)
- [system_views.sql#pg_stat_bgwriter](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947)

## Navigation

- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [Which Client Statements Set Hint Bits and Dirty Pages as Hints in PostgreSQL 12 (unverified)](../storage-and-vacuum/client-statements-that-set-hint-bits.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](explain-analyze-buffers-output.md)
- [How pg_stat_statements Works and Which Settings Affect It in PostgreSQL 12 (unverified)](pg-stat-statements.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
