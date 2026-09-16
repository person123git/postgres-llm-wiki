---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Pros and Cons of a Very Large shared_buffers Such as 256 GB on a 1 TB RAM System in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [What 256 GB of shared_buffers actually allocates](#what-256-gb-of-shared_buffers-actually-allocates)
  - [Pros](#pros)
  - [Cons](#cons)
  - [Thresholds that move when NBuffers is huge](#thresholds-that-move-when-nbuffers-is-huge)
  - [What a big pool does not buy](#what-a-big-pool-does-not-buy)
  - [Settings that move with it, and their apply scope](#settings-that-move-with-it-and-their-apply-scope)
  - [What to look at before and after the change](#what-to-look-at-before-and-after-the-change)
  - [Decision guide](#decision-guide)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, what are the pros and cons of having a very large `shared_buffers`, like 256 GB or more, on a system with more than 1 TB of RAM?

Prompt note, per `MANDATORY Prompt Hygiene`: the original request read `follow agents.md, in postgresql 12, question : what are the pros and cons of having a very large shared buffers like 256GB or more on a system with more than 1TB of RAM.` The defects were `agents.md` for AGENTS.md, lowercase `postgresql`, a space before the colon, `a very large shared buffers` for the GUC `shared_buffers`, `256GB`/`1TB` missing the space before the unit, a lowercase sentence start, and a terminating period on a question. The asker chose **correct and restate**, chose **source-only evidence with no measured numbers**, and chose **v12-only framing** with no reference to controls that exist in other major versions. All three choices are recorded here.

## Answer

### Short answer

256 GB is a legal setting and not an absurd one: v12 accepts up to `INT_MAX / 2` blocks, and the shipped documentation names 25 % of RAM as a starting point and 40 % as the point past which a larger pool is unlikely to beat a smaller one ([guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2150-L2163), [config.sgml#shared_buffers](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1495-L1528)). The real question is not whether the engine will start; it is that several v12 code paths are O(`NBuffers`), so their cost rises exactly in proportion to the pool while the work they do stays the same.

At the default 8 kB block size, 256 GB is 33,554,432 buffers: 2,048 times the 128 MB `initdb` default and 32 times an 8 GB pool. That multiplier lands on the checkpoint scan, the clock sweep, and every "find all buffers of this relation" walk.

| | Mechanism | Where it comes from |
|---|---|---|
| Pro | A hit costs a hash probe and a pin, no syscall | [bufmgr.c#BufferAlloc](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1020-L1057) |
| Pro | Fewer evictions means fewer foreground writes and WAL flushes | [bufmgr.c#BufferAlloc](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1088-L1157) |
| Pro | Repeatedly dirtied pages are written once per checkpoint | [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1812-L1857) |
| Con | Checkpoint scan and sort walk every buffer | [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1828-L1871) |
| Con | Drop, truncate and flush paths walk every buffer | [bufmgr.c#DropRelFileNodeBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2919-L2971) |
| Con | Victim search can need six full passes of the pool | [buf_internals.h#BM_MAX_USAGE_COUNT](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L69-L77) |
| Con | The anti-cache-flooding threshold is `NBuffers / 4` | [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L233-L252) |
| Con | Background writer output is capped independently of pool size | [bufmgr.c#BgBufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2274-L2300) |

### What 256 GB of shared_buffers actually allocates

`shared_buffers` is `PGC_POSTMASTER`, measured in `BLCKSZ` blocks, with a floor of 16 blocks and a ceiling of `INT_MAX / 2` blocks, because the code "sometimes multiplies the number of shared buffers by two without checking for overflow" ([guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2150-L2163)). Changing it needs a **restart**. The shipped default is 1024 blocks in the C table; `initdb` overwrites it by probing downward from 16384 blocks, that is 128 MB, and writing the largest value a trial postmaster accepts ([initdb.c#test_config_settings](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L947-L967), [initdb.c#test_config_settings](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1018-L1050)).

The number you write is the page bytes only. `BufferShmemSize()` adds four more per-buffer arrays on top of `NBuffers * BLCKSZ`: the descriptors, the freelist and buffer-table structures, the I/O locks, and the checkpoint sort array ([buf_init.c#BufferShmemSize](../../../../raw/postgres-12/src/backend/storage/buffer/buf_init.c#L160-L193)). The checkpointer separately sizes its fsync request queue at `NBuffers` entries ([checkpointer.c#CheckpointerShmemSize](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L889-L902), [checkpointer.c:928](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L928)).

| Per buffer | Structure | Bytes on a 64-bit build | Source |
|---|---|---|---|
| Page | `BLCKSZ` | 8192 | [buf_init.c#InitBufferPool](../../../../raw/postgres-12/src/backend/storage/buffer/buf_init.c#L81-L83) |
| Descriptor | `BufferDescPadded` | 64 | [buf_internals.h#BufferDescPadded](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L192-L218) |
| I/O lock | `LWLockMinimallyPadded` | 32 | [lwlock.h#LWLOCK_MINIMAL_SIZE](../../../../raw/postgres-12/src/include/storage/lwlock.h#L61-L88) |
| Checkpoint sort slot | `CkptSortItem` | 20 | [buf_internals.h#CkptSortItem](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L283-L298) |
| Fsync queue slot | `CheckpointerRequest` | 24 | [sync.h#FileTag](../../../../raw/postgres-12/src/include/storage/sync.h#L45-L51), [checkpointer.c#CheckpointerRequest](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L108-L113) |
| Mapping entry | `BufferLookupEnt`, sized for `NBuffers + 128` entries | 24 plus dynahash overhead | [buf_table.c#BufTableShmemSize](../../../../raw/postgres-12/src/backend/storage/buffer/buf_table.c#L28-L45), [freelist.c#StrategyShmemSize](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L453-L465) |

Multiplying those fixed struct sizes by 33,554,432 buffers gives roughly 2 GiB of descriptors, 1 GiB of I/O locks, 640 MiB of checkpoint sort slots and 768 MiB of fsync queue, that is about **4.4 GiB above the nominal 256 GiB before the mapping hash table is counted**. Those totals are arithmetic on the struct definitions, not a reading from a running server; see [Open Questions](#open-questions).

The whole request is mapped in one call. With the default `shared_memory_type = mmap`, `CreateAnonymousSegment()` issues a single `mmap`, retries without `MAP_HUGETLB` when `huge_pages = try` fails, and raises a FATAL error whose hint names `shared_buffers` when the kernel refuses ([sysv_shmem.c#CreateAnonymousSegment](../../../../raw/postgres-12/src/backend/port/sysv_shmem.c#L528-L590), [sysv_shmem.c#PGSharedMemoryCreate](../../../../raw/postgres-12/src/backend/port/sysv_shmem.c#L624-L657), [guc.c#shared_memory_type](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4429-L4437)). Then every descriptor is initialized in a serial loop before the postmaster accepts connections ([buf_init.c#InitBufferPool](../../../../raw/postgres-12/src/backend/storage/buffer/buf_init.c#L111-L144)).

### Pros

**A hit is a hash probe and a pin.** `BufferAlloc()` computes the tag hash, takes the mapping partition lock in shared mode, finds the buffer id, pins it, and returns; `ReadBuffer_common()` counts `shared_blks_hit` and returns before it reaches the storage manager ([bufmgr.c#BufferAlloc](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1020-L1057), [bufmgr.c#ReadBuffer_common](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L743-L796)). A miss instead calls `smgrread()` ([bufmgr.c:897](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897)), which reaches the kernel through `FileRead()` and `pg_pread()` ([md.c:596](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L596), [fd.c:1881](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1881)). A bigger pool converts more of the second shape into the first, including the OS-cache hits that still cost a syscall and a page copy.

**Fewer evictions, so fewer writes on the query path.** When the clock sweep hands back a dirty victim, the *requesting backend* writes it, and `FlushBuffer()` first forces WAL up to the page LSN for permanent relations ([bufmgr.c#BufferAlloc](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1088-L1157), [bufmgr.c#FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2717-L2762)). Those events are exactly `buffers_backend` and `buffers_backend_fsync` in `pg_stat_bgwriter` ([system_views.sql#pg_stat_bgwriter](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947)). A pool large enough to hold the write working set removes that work from user queries.

**Write coalescing across a checkpoint interval.** `BufferSync()` marks the buffers that were dirty when the checkpoint began and writes only those; a page dirtied a thousand times between two checkpoints is written once ([bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1812-L1857)). The larger the pool, the longer a hot page can stay dirty in memory rather than being evicted and re-read.

**Hot pages resist eviction.** Each pin raises `usage_count` up to `BM_MAX_USAGE_COUNT`, which is 5, and the sweep must decrement it to zero before the buffer can be taken ([buf_internals.h#BM_MAX_USAGE_COUNT](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L69-L77), [freelist.c#StrategyGetBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L315-L357)).

**Access-strategy rings reach their intended size.** Ring sizes are clamped by `Min(NBuffers / 8, ring_size)`, so a small pool shrinks the 256 kB bulk-read and vacuum rings and the 16 MB bulk-write ring; at 33.5 M buffers the clamp never binds ([freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L541-L588)).

**The fsync request queue effectively never fills.** Its capacity is `NBuffers`, and a full queue is what forces a backend to perform its own fsync ([checkpointer.c#ForwardSyncRequest](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L1107-L1160)).

**Large scans can actually be cached.** The bulk-read strategy and synchronized scanning engage only above `NBuffers / 4` blocks, so with a very large pool, mid-sized tables are read with the default strategy and stay resident ([heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L233-L252)). This is the same fact as the fourth con below; whether it helps or hurts depends on whether you want those pages kept.

### Cons

**Double buffering.** v12 reads and writes data files through the kernel page cache: `mdread()`/`mdwrite()` go to `FileRead()`/`FileWrite()` and then to `pg_pread()`/`pg_pwrite()` ([md.c:596](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L596), [md.c:666](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L666), [fd.c:1881](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1881), [fd.c:1963](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1963)). The only `PG_O_DIRECT` use in the tree is for WAL files, gated on `!XLogIsNeeded()`, that is `wal_level = minimal`, and never in the walreceiver ([xlogdefs.h#PG_O_DIRECT](../../../../raw/postgres-12/src/include/access/xlogdefs.h#L60-L74), [xlog.c#get_sync_bit](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L10019-L10035), [xlog.h#XLogIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L181)). A page that is hot in `shared_buffers` therefore tends to occupy RAM twice, which is the mechanism behind the documentation's warning that more than 40 % of RAM is unlikely to beat a smaller setting ([config.sgml#shared_buffers](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1510-L1523)).

**Every checkpoint scans the whole pool.** `BufferSync()` takes the header spinlock on each of `NBuffers` buffers to test `BM_DIRTY`, then `qsort`s the collected entries before writing ([bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1828-L1871)). At 33.5 M buffers that is 33.5 M spinlock acquisitions per checkpoint even if nothing is dirty. `CheckPointBuffers()` then runs the fsync phase ([bufmgr.c#CheckPointBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2580-L2593)), and the write phase is paced against `checkpoint_completion_target` ([checkpointer.c#CheckpointWriteDelay](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L648-L715), [checkpointer.c#IsCheckpointOnSchedule](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L717-L745)). A larger pool means a larger maximum dirty set to flush inside one interval, which is why the documentation ties a larger `shared_buffers` to a larger `max_wal_size` ([config.sgml#shared_buffers](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1518-L1522)).

**Drop, truncate and flush are linear scans, and the source says so.** `DropRelFileNodeBuffers()` carries the comment "XXX currently it sequentially searches the buffer pool", and so does `FlushRelationBuffers()` ([bufmgr.c#DropRelFileNodeBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2919-L2971), [bufmgr.c#FlushRelationBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3183-L3271)). The same shape appears in `DropRelFileNodesAllBuffers()`, `DropDatabaseBuffers()` and `FlushDatabaseBuffers()` ([bufmgr.c#DropRelFileNodesAllBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2982-L3072), [bufmgr.c#DropDatabaseBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3085-L3113), [bufmgr.c#FlushDatabaseBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3288-L3325)). The user-visible operations that reach them:

| Operation | Path | Source |
|---|---|---|
| Relation or fork unlink, for example `DROP TABLE` | `smgrdounlink`, `smgrdounlinkall`, `smgrdounlinkfork` | [smgr.c:390](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L390), [smgr.c:463](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L463), [smgr.c:523](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L523) |
| Heap truncation by VACUUM, and index truncation | `RelationTruncate` to `smgrtruncate` | [storage.c:294](../../../../raw/postgres-12/src/backend/catalog/storage.c#L294), [smgr.c:652](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L652), [vacuumlazy.c:1965](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1965) |
| Truncation replay on a standby or after a crash | `smgr_redo` | [storage.c:621](../../../../raw/postgres-12/src/backend/catalog/storage.c#L621) |
| `ALTER TABLE ... SET TABLESPACE` | `FlushRelationBuffers` before the file copy | [tablecmds.c:12778](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L12778) |
| `heap_sync` after a WAL-skipping bulk load | `FlushRelationBuffers` for heap and TOAST | [heapam.c#heap_sync](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8937-L8960) |
| `CREATE DATABASE`, `DROP DATABASE`, and their redo | `FlushDatabaseBuffers`, `DropDatabaseBuffers` | [dbcommands.c:942](../../../../raw/postgres-12/src/backend/commands/dbcommands.c#L942), [dbcommands.c:1228](../../../../raw/postgres-12/src/backend/commands/dbcommands.c#L1228), [dbcommands.c:2134](../../../../raw/postgres-12/src/backend/commands/dbcommands.c#L2134) |

A schema with many partitions makes this worse in the obvious way: dropping or truncating *n* relations one at a time is *n* walks of 33.5 M buffers, and only the batched `DropRelFileNodesAllBuffers()` amortizes them into one walk ([bufmgr.c#DropRelFileNodesAllBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3017-L3069)).

**Victim search degrades when the pool finally fills.** Until the freelist empties, allocation is cheap ([freelist.c#StrategyGetBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L268-L313)). After that, every allocation runs the clock sweep, and the header comment states the bound plainly: "it can take as many as `BM_MAX_USAGE_COUNT` + 1 complete cycles of clock sweeps to find a free buffer" ([buf_internals.h#BM_MAX_USAGE_COUNT](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L69-L77)). With 33.5 M buffers that worst case is about 201 M buffer-header lock-and-test operations for one allocation, and the `trycounter` that guards the loop is reset on every usage-count decrement, so only a run of `NBuffers` consecutively pinned buffers ends it with `no unpinned buffers available` ([freelist.c#StrategyGetBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L315-L357)). A very large pool also delays the first sweep, so usage counts have longer to saturate at 5 before any decay begins.

**The background writer does not scale with the pool.** Per round it writes at most `bgwriter_lru_maxpages`, default 100, and sleeps `bgwriter_delay`, default 200 ms ([bufmgr.c#BgBufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2274-L2300), [guc.c#bgwriter_lru_maxpages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2738-L2746), [guc.c#bgwriter_delay](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2727-L2736)). Its idle "cover the pool in 120 s" floor, `min_scan_buffers = NBuffers / (120000 / bgwriter_delay)`, is added to the reusable-buffer estimate that ends the scan loop, so at 33.5 M buffers and the default delay a round keeps scanning until it has found about 55,900 further reusable buffers, unless it laps the sweep first or stops at the 100-page write cap ([bufmgr.c#BgBufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2239-L2258)). The defaults were not chosen for a pool this size; leaving them alone pushes cleaning onto backends and onto the checkpoint.

**Memory committed here is not available elsewhere.** The pool is reserved at startup for the life of the postmaster. `work_mem` is per operation, `maintenance_work_mem` is per maintenance operation, and both are `PGC_USERSET`, so their true peak scales with concurrency ([guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2230-L2241), [guc.c#maintenance_work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2243-L2252)). The documentation's OOM guidance names lowering `shared_buffers` and `work_mem`, or reducing `max_connections` in favour of external pooling, as the responses to memory pressure ([runtime.sgml#linux-memory-overcommit](../../../../raw/postgres-12/doc/src/sgml/runtime.sgml#L1448-L1476)).

**Page-table overhead and huge-page operations.** The documentation states that huge pages reduce page tables and memory-management CPU time, "particularly when using large values of `shared_buffers`" ([runtime.sgml#linux-huge-pages](../../../../raw/postgres-12/doc/src/sgml/runtime.sgml#L1535-L1541)). Getting them is an operational task: size `vm.nr_hugepages` from the segment size, possibly set `vm.hugetlb_shm_group` and `ulimit -l`, and accept that `huge_pages = on` refuses to start when they are unavailable ([runtime.sgml#linux-huge-pages](../../../../raw/postgres-12/doc/src/sgml/runtime.sgml#L1542-L1594), [config.sgml#huge_pages](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1540-L1561)). The failure paths are explicit in the allocator ([sysv_shmem.c#CreateAnonymousSegment](../../../../raw/postgres-12/src/backend/port/sysv_shmem.c#L535-L586), [sysv_shmem.c#PGSharedMemoryCreate](../../../../raw/postgres-12/src/backend/port/sysv_shmem.c#L634-L640)).

**Restart and failover cost.** Every descriptor is initialized serially at startup ([buf_init.c#InitBufferPool](../../../../raw/postgres-12/src/backend/storage/buffer/buf_init.c#L111-L144)), a shutdown checkpoint must flush all dirty buffers including unlogged ones ([bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1803-L1810)), and after restart the pool is cold. The contrib `pg_prewarm` module can reload it, either explicitly or from a periodic dump of the buffer contents ([pg_prewarm.c#pg_prewarm](../../../../raw/postgres-12/contrib/pg_prewarm/pg_prewarm.c#L185-L200), [autoprewarm.c](../../../../raw/postgres-12/contrib/pg_prewarm/autoprewarm.c#L1-L24)), but re-reading 256 GB is itself the restart cost.

**Introspection gets expensive.** `pg_buffercache` allocates `sizeof(BufferCachePagesRec) * NBuffers` in backend-local memory with `MemoryContextAllocHuge()` and takes a header lock on every buffer ([pg_buffercache_pages.c](../../../../raw/postgres-12/contrib/pg_buffercache/pg_buffercache_pages.c#L126-L176)). At 33.5 M buffers that is a multi-hundred-megabyte local allocation and 33.5 M lock-unlock pairs per call.

**A full fsync queue becomes expensive to compact.** The queue rarely fills, but when it does, `CompactCheckpointerRequestQueue()` pallocs one `bool` per request and builds a hash over all of them while `CheckpointerCommLock` is held exclusively; the array is `NBuffers` long ([checkpointer.c#CompactCheckpointerRequestQueue](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L1178-L1268)).

**Contention is not reduced by size.** The mapping table is split into a fixed `NUM_BUFFER_PARTITIONS = 128` partitions regardless of `NBuffers` ([lwlock.h:107-126](../../../../raw/postgres-12/src/include/storage/lwlock.h#L107-L126), [buf_internals.h#BufMappingPartitionLock](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L121-L133)), an eviction takes two partition locks in exclusive mode ([bufmgr.c#BufferAlloc](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1169-L1212)), and `buffer_strategy_lock` remains one system-wide spinlock ([freelist.c#BufferStrategyControl](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L26-L61)). A workload bottlenecked on one hot page or on mapping-lock contention gains nothing from more buffers.

**No NUMA placement.** The segment is created with a single plain `mmap` and the tree contains no memory-policy call, so page placement across sockets is left entirely to the kernel ([sysv_shmem.c#CreateAnonymousSegment](../../../../raw/postgres-12/src/backend/port/sysv_shmem.c#L528-L590)). A 1 TB host is almost certainly multi-socket, and v12 has no lever for this.

### Thresholds that move when NBuffers is huge

Several unrelated subsystems derive their sizing from `NBuffers`. Setting 256 GB changes all of them at once, which is the part most easily missed.

| Derived value | Formula | At 33,554,432 buffers | Source |
|---|---|---|---|
| Bulk-read strategy and synchronized scan threshold | `rs_nblocks > NBuffers / 4` | 64 GiB; smaller tables use the default strategy | [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L233-L252) |
| Parallel seq-scan sync-scan decision | `phs_nblocks > NBuffers / 4` | same 64 GiB threshold | [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-12/src/backend/access/table/tableam.c#L370-L385) |
| Access-strategy ring clamp | `Min(NBuffers / 8, ring_size)` | never binds | [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L547-L577) |
| Auto-tuned `wal_buffers` | `NBuffers / 32`, capped at one WAL segment, floor 8 | capped, so 16 MB at the default segment size | [xlog.c#XLOGChooseNumBuffers](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4861-L4873) |
| CLOG SLRU buffers | `Min(128, Max(4, NBuffers / 512))` | capped at 128, already reached at 512 MB | [clog.c#CLOGShmemBuffers](../../../../raw/postgres-12/src/backend/access/transam/clog.c#L670-L679) |
| commit timestamp SLRU buffers | `Min(16, Max(4, NBuffers / 1024))` | capped at 16, already reached at 128 MB | [commit_ts.c#CommitTsShmemBuffers](../../../../raw/postgres-12/src/backend/access/transam/commit_ts.c#L465-L473) |
| Checkpointer fsync queue length | `NBuffers` | 33.5 M slots, about 768 MiB | [checkpointer.c#CheckpointerShmemSize](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L889-L902) |
| Hash index build sort threshold | `Min(maintenance_work_mem / BLCKSZ, NBuffers)` | `maintenance_work_mem` decides | [hash.c#hashbuild](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L140-L158) |
| Checkpoint log percentage | `ckpt_bufs_written * 100 / NBuffers` | small percentages even for huge write volumes | [xlog.c#LogCheckpointEnd](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8435-L8442) |

The first row deserves emphasis. The buffer-ring strategy exists precisely so that a large one-pass scan does not blow out the cache ([README#Buffer Ring Replacement Strategy](../../../../raw/postgres-12/src/backend/storage/buffer/README#L208-L249)). Raising `shared_buffers` to 256 GB raises the bar for that protection to 64 GiB, so a one-off sequential scan of, say, a 50 GB table now runs with the default strategy and can evict a large part of the working set. It also stops qualifying for synchronized scans, so two concurrent scans of the same table no longer share their reads ([heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L233-L252)).

### What a big pool does not buy

- **Temporary tables.** They use session-local buffers governed by `temp_buffers`, not the shared pool ([config.sgml#temp_buffers](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1592-L1612), [guc.c#temp_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2165-L2174), [bufmgr.c#ReadBuffer_common](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L732-L742)).
- **VACUUM reads.** Both manual and auto vacuum allocate a `BAS_VACUUM` ring and reuse it; the ring is 256 kB and does not grow with the pool, since `NBuffers` only ever clamps it downward ([vacuum.c:296](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L296), [autovacuum.c:2288](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2288), [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L559-L577)).
- **Planner cost estimates.** The planner never reads `NBuffers`; the comment at the top of `costsize.c` says so explicitly, and the cache assumption comes from `effective_cache_size`, a `PGC_USERSET` GUC defaulting to 524288 blocks ([costsize.c](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L22-L25), [guc.c#effective_cache_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3107-L3117), [cost.h#DEFAULT_EFFECTIVE_CACHE_SIZE](../../../../raw/postgres-12/src/include/optimizer/cost.h#L32)). Growing the pool without revisiting `effective_cache_size` leaves plans unchanged.
- **SLRU caches beyond modest sizes.** CLOG reaches its 128-buffer cap at 512 MB of `shared_buffers`, and commit timestamps reach their 16-buffer cap at 128 MB ([clog.c#CLOGShmemBuffers](../../../../raw/postgres-12/src/backend/access/transam/clog.c#L670-L679), [commit_ts.c#CommitTsShmemBuffers](../../../../raw/postgres-12/src/backend/access/transam/commit_ts.c#L465-L473)).
- **WAL buffering.** `wal_buffers = -1` is capped at one WAL segment, so it stops growing far below 256 GB ([xlog.c#XLOGChooseNumBuffers](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4861-L4873)).

### Settings that move with it, and their apply scope

Apply scope follows the GUC context: `postmaster` needs a restart, `sighup` needs a reload, `userset` applies per session or transaction.

| Setting | Context | Apply scope | Why it matters here | Source |
|---|---|---|---|---|
| `shared_buffers` | `PGC_POSTMASTER` | restart | the setting itself | [guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2150-L2163) |
| `huge_pages` | `PGC_POSTMASTER` | restart | page-table overhead for a 256 GB mapping | [guc.c#huge_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4470-L4478) |
| `shared_memory_type` | `PGC_POSTMASTER` | restart | selects the `mmap` or SysV path used to obtain the segment | [guc.c#shared_memory_type](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4429-L4437) |
| `wal_buffers` | `PGC_POSTMASTER` | restart | auto-tuning is capped, so set it explicitly if you want more | [guc.c#wal_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2602-L2611) |
| `max_wal_size` | `PGC_SIGHUP` | reload | the documentation ties it to a larger `shared_buffers` | [guc.c#max_wal_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2554-L2564) |
| `checkpoint_timeout` | `PGC_SIGHUP` | reload | sets how much dirty data can accumulate per checkpoint | [guc.c#checkpoint_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2566-L2575) |
| `checkpoint_completion_target` | `PGC_SIGHUP` | reload | spreads the write phase | [guc.c#checkpoint_completion_target](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3413-L3421) |
| `checkpoint_flush_after` | `PGC_SIGHUP` | reload | paces writeback requests during the checkpoint | [guc.c#checkpoint_flush_after](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2591-L2600) |
| `bgwriter_lru_maxpages` | `PGC_SIGHUP` | reload | the 100-page cap is the main brake on background cleaning | [guc.c#bgwriter_lru_maxpages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2738-L2746) |
| `bgwriter_lru_multiplier` | `PGC_SIGHUP` | reload | scales the lookahead estimate | [guc.c#bgwriter_lru_multiplier](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3351-L3359) |
| `bgwriter_delay` | `PGC_SIGHUP` | reload | sets rounds per second and the idle pool-coverage pace | [guc.c#bgwriter_delay](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2727-L2736) |
| `bgwriter_flush_after` | `PGC_SIGHUP` | reload | writeback pacing for background writes | [guc.c#bgwriter_flush_after](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2748-L2757) |
| `log_checkpoints` | `PGC_SIGHUP` | reload | the only built-in per-checkpoint timing record | [guc.c#log_checkpoints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1217-L1225) |
| `backend_flush_after` | `PGC_USERSET` | session or transaction | writeback pacing for backend-issued writes | [guc.c#backend_flush_after](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2777-L2786) |
| `effective_cache_size` | `PGC_USERSET` | session or transaction | the planner's cache assumption, independent of `NBuffers` | [guc.c#effective_cache_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3107-L3117) |
| `synchronize_seqscans` | `PGC_USERSET` | session or transaction | already disabled below the `NBuffers / 4` table-size threshold | [guc.c#synchronize_seqscans](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1732-L1740) |
| `temp_buffers` | `PGC_USERSET` | session, before first temp-table use | temp tables never use the shared pool | [guc.c#temp_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2165-L2174) |
| `work_mem` | `PGC_USERSET` | session or transaction | competes for the RAM the pool did not take | [guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2230-L2241) |
| `maintenance_work_mem` | `PGC_USERSET` | session or transaction | same, and it decides the hash-build sort threshold | [guc.c#maintenance_work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2243-L2252) |

### What to look at before and after the change

v12 exposes the relevant counters in `pg_stat_bgwriter` and `pg_stat_database`, and the per-checkpoint timings in the log ([system_views.sql#pg_stat_bgwriter](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947), [system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L882), [xlog.c#LogCheckpointEnd](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8435-L8442)). Note the boundary: `blks_hit` counts shared-buffer hits only, so a page served from the OS cache is counted as `blks_read` even though no disk head moved.

All three statements below are verified against this checkout's catalogs and functions, and both timeouts are session-scoped, so they expire with the session.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';
```

Who is doing the writing, and how often backends had to write or fsync for themselves:

```sql
SELECT /* wiki_v12_shared_buffers_writers */
       buffers_checkpoint,
       buffers_clean,
       maxwritten_clean,
       buffers_backend,
       buffers_backend_fsync,
       buffers_alloc,
       checkpoints_timed,
       checkpoints_req,
       checkpoint_write_time,
       checkpoint_sync_time,
       stats_reset
FROM pg_stat_bgwriter;
```

Shared-buffer hit ratio per database, remembering that misses may still be OS-cache hits:

```sql
SELECT /* wiki_v12_shared_buffer_hit_ratio */
       datname,
       blks_hit,
       blks_read,
       round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS hit_pct,
       blk_read_time,
       blk_write_time
FROM pg_stat_database
WHERE datname IS NOT NULL
ORDER BY blks_hit + blks_read DESC;
```

The derived thresholds this page describes, computed from the running configuration:

```sql
SELECT /* wiki_v12_nbuffers_derived_thresholds */
       sb.setting::bigint AS shared_buffer_blocks,
       pg_size_pretty(sb.setting::bigint * bs.setting::bigint) AS shared_buffers_total,
       pg_size_pretty((sb.setting::bigint / 4) * bs.setting::bigint) AS bulkread_strategy_threshold,
       pg_size_pretty((sb.setting::bigint / 8) * bs.setting::bigint) AS ring_clamp,
       least(128, greatest(4, sb.setting::bigint / 512)) AS clog_slru_buffers,
       least(16, greatest(4, sb.setting::bigint / 1024)) AS commit_ts_slru_buffers
FROM pg_settings sb, pg_settings bs
WHERE sb.name = 'shared_buffers'
  AND bs.name = 'block_size';
```

`pg_settings` is a view over `pg_show_all_settings()`, whose output columns include `name`, `setting`, `unit` and `context`, and `shared_buffers` reports in blocks because of its `GUC_UNIT_BLOCKS` flag ([system_views.sql#pg_settings](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L512-L513), [pg_proc.dat#pg_show_all_settings](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5770-L5775), [guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2159)). `block_size` is a `PGC_INTERNAL` preset that reports `BLCKSZ` in bytes ([guc.c#block_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2879-L2888)), and `pg_size_pretty(bigint)` exists in this checkout ([pg_proc.dat#pg_size_pretty](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6897-L6899)).

There is no catalog view of shared-memory allocations in this tree, so the only way to see what the postmaster actually mapped is from the operating system; the huge-pages procedure in the documentation does exactly that with `pmap` on the pid in `postmaster.pid` ([runtime.sgml#linux-huge-pages](../../../../raw/postgres-12/doc/src/sgml/runtime.sgml#L1542-L1566)). Use `pg_buffercache` sparingly at this size, for the reasons in [Cons](#cons).

### Decision guide

Reasoning from the mechanisms above, not from measurements on this hardware class:

**A very large pool is defensible when** the hot working set is larger than a moderate pool but fits in 256 GB, the workload re-dirties the same pages many times per checkpoint interval so write coalescing pays, the write path is otherwise dominated by `buffers_backend`, and the operational profile is stable: few `DROP`/`TRUNCATE`/`SET TABLESPACE` operations, infrequent restarts, huge pages configured, and checkpoint and background-writer settings tuned away from their defaults.

**It is a poor trade when** any of the following hold, each for a reason established above: the working set is far larger than any pool you can afford, so you are paying double buffering for a low hit rate; the schema has many partitions or the workload creates and drops many relations, so the O(`NBuffers`) walks run constantly; mid-sized table scans are common and you were relying on the bulk-read ring to protect the cache, which now needs a 64 GiB table to engage; the bottleneck is contention on a hot page or on mapping partitions, which more buffers do not relieve; or the server also needs the RAM for `work_mem`-hungry queries and the OS cache.

**Whatever you choose**, move `shared_buffers` in steps rather than to 256 GB in one jump, since each step needs a restart anyway, and re-check the `pg_stat_bgwriter` counters and `log_checkpoints` lines at each step. The engine gives no feedback that a pool is too large; it only shows up as checkpoint and eviction behaviour in those counters.

## Context Reviewed

- Buffer manager core: `src/backend/storage/buffer/bufmgr.c`, `buf_init.c`, `freelist.c`, `buf_table.c`, `localbuf.c` headers, and `src/backend/storage/buffer/README`.
- Buffer structures and partitioning: `src/include/storage/buf_internals.h`, `src/include/storage/buf.h`, `src/include/storage/lwlock.h`.
- Shared memory sizing and creation: `src/backend/storage/ipc/ipci.c`, `src/backend/port/sysv_shmem.c`.
- Checkpointer and background writer: `src/backend/postmaster/checkpointer.c`, `src/backend/access/transam/xlog.c` checkpoint logging and `XLOGChooseNumBuffers`.
- Every non-buffer reader of `NBuffers` in the backend: `clog.c`, `commit_ts.c`, `hash.c`, `heapam.c`, `tableam.c`, `costsize.c`, `postmaster.c`, `globals.c`, `guc.c`.
- Full-pool scan callers: `src/backend/storage/smgr/smgr.c`, `src/backend/catalog/storage.c`, `src/backend/commands/tablecmds.c`, `src/backend/commands/dbcommands.c`, `src/backend/access/heap/heapam.c`, `src/backend/access/heap/vacuumlazy.c`, `src/backend/access/heap/heapam_handler.c`.
- Storage manager I/O path: `src/backend/storage/smgr/md.c`, `src/backend/storage/file/fd.c`, `src/include/access/xlogdefs.h`.
- GUC definitions and defaults: `src/backend/utils/misc/guc.c`, `src/include/optimizer/cost.h`, `src/bin/initdb/initdb.c`.
- Catalogs and views: `src/backend/catalog/system_views.sql`, `src/include/catalog/pg_proc.dat`.
- Contrib boundary: `contrib/pg_buffercache/pg_buffercache_pages.c`, `contrib/pg_prewarm/pg_prewarm.c`, `contrib/pg_prewarm/autoprewarm.c`.
- Documentation in the same checkout: `doc/src/sgml/config.sgml`, `doc/src/sgml/runtime.sgml`.
- Tests: searched `src/test` and `contrib` for `shared_buffers`. The only uses are a 128 kB setting in one recovery TAP test and the 1 MB default in the TAP harness; there is no test that exercises a large pool. See [Open Questions](#open-questions).

## Evidence Map

| Claim | Evidence |
|---|---|
| Restart required; range is 16 to `INT_MAX / 2` blocks | [guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2150-L2163) |
| 25 % starting point, 40 % caution, larger pool wants larger `max_wal_size` | [config.sgml#shared_buffers](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1510-L1528) |
| `initdb` probes down from 16384 blocks | [initdb.c#test_config_settings](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L947-L967) |
| Four per-buffer arrays beyond the page bytes, plus the `NBuffers`-sized fsync queue | [buf_init.c#BufferShmemSize](../../../../raw/postgres-12/src/backend/storage/buffer/buf_init.c#L160-L193), [checkpointer.c#CheckpointerShmemSize](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L889-L902) |
| Descriptor is 64 bytes, I/O lock 32, sort slot 20, request slot 24 | [buf_internals.h#BufferDescPadded](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L192-L218), [lwlock.h#LWLOCK_MINIMAL_SIZE](../../../../raw/postgres-12/src/include/storage/lwlock.h#L61-L88), [buf_internals.h#CkptSortItem](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L283-L298), [sync.h#FileTag](../../../../raw/postgres-12/src/include/storage/sync.h#L45-L51) |
| Single `mmap`, huge-page fallback, FATAL hint names `shared_buffers` | [sysv_shmem.c#CreateAnonymousSegment](../../../../raw/postgres-12/src/backend/port/sysv_shmem.c#L528-L590) |
| Hit path returns before the storage manager | [bufmgr.c#ReadBuffer_common](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L743-L796), [bufmgr.c#BufferAlloc](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1020-L1057) |
| Miss path reaches the kernel through buffered `pread` | [bufmgr.c:897](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897), [md.c:596](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L596), [fd.c:1881](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1881) |
| Backends write dirty victims and flush WAL first | [bufmgr.c#BufferAlloc](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1088-L1157), [bufmgr.c#FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2717-L2762) |
| Checkpoint marks, sorts, then writes the dirty set | [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1812-L1871) |
| Up to `BM_MAX_USAGE_COUNT + 1` full sweeps for one victim | [buf_internals.h#BM_MAX_USAGE_COUNT](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L69-L77), [freelist.c#StrategyGetBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L315-L357) |
| Background writer cap and pool-coverage pace | [bufmgr.c#BgBufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2239-L2300) |
| Five full-pool scan functions and their callers | [bufmgr.c#DropRelFileNodeBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2919-L2971), [smgr.c:390](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L390), [storage.c:294](../../../../raw/postgres-12/src/backend/catalog/storage.c#L294), [tablecmds.c:12778](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L12778), [dbcommands.c:942](../../../../raw/postgres-12/src/backend/commands/dbcommands.c#L942) |
| `NBuffers / 4` governs bulk-read strategy and sync scans | [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L233-L252), [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-12/src/backend/access/table/tableam.c#L370-L385) |
| Ring purpose and sizes | [README#Buffer Ring Replacement Strategy](../../../../raw/postgres-12/src/backend/storage/buffer/README#L208-L249), [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L541-L588) |
| Partition count is fixed at 128 | [lwlock.h:107-126](../../../../raw/postgres-12/src/include/storage/lwlock.h#L107-L126) |
| Planner uses `effective_cache_size`, not `NBuffers` | [costsize.c](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L22-L25), [guc.c#effective_cache_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3107-L3117) |
| Observability counters and checkpoint log line | [system_views.sql#pg_stat_bgwriter](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947), [xlog.c#LogCheckpointEnd](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8435-L8442) |

## Open Questions

- The per-buffer byte figures and the derived totals of roughly 2 GiB, 1 GiB, 640 MiB and 768 MiB are arithmetic on the struct definitions for a 64-bit build with 8 kB blocks and 4-byte enums. They were not confirmed against a running server, and this page reports no measured number by design, per the scope the asker chose. `CkptSortItem` and `CheckpointerRequest` in particular depend on the compiler's enum width and padding.
- `LWLOCK_MINIMAL_SIZE` is `sizeof(LWLock) <= 32 ? 32 : 64`, and the header comment says 32 "on basically all common platforms" ([lwlock.h#LWLOCK_MINIMAL_SIZE](../../../../raw/postgres-12/src/include/storage/lwlock.h#L61-L88)). A `LOCK_DEBUG` build adds fields to `LWLock` and could change it.
- The buffer-mapping hash table size goes through `hash_estimate_size()` ([buf_table.c#BufTableShmemSize](../../../../raw/postgres-12/src/backend/storage/buffer/buf_table.c#L38-L45)); its dynahash bucket and segment overhead at 33.5 M entries was not derived here.
- No test in this checkout exercises a large `shared_buffers`. The only test settings found are `shared_buffers = 128kB` in [016_min_consistency.pl:52](../../../../raw/postgres-12/src/test/recovery/t/016_min_consistency.pl#L52) and `shared_buffers = 1MB` in [PostgresNode.pm:475](../../../../raw/postgres-12/src/test/perl/PostgresNode.pm#L475). The O(`NBuffers`) behaviour described here is therefore read from the code, with no upstream regression coverage at scale.
- Whether a 256 GB pool is a net win for a given workload cannot be settled from source. The mechanisms are cited; the outcome depends on working-set size, write locality, and storage, none of which this page measures.
- The clock-sweep worst case of six full passes is the bound the header comment states; how often it is approached in practice on a pool this size is not established by any code path or test here.
- NUMA placement is left to the kernel, since no memory-policy call exists in this tree. What that costs on a multi-socket 1 TB host is outside what the source can answer.
- v12 has no catalog-level view of shared-memory allocations, so the "what did the postmaster actually map" check is an operating-system procedure ([runtime.sgml#linux-huge-pages](../../../../raw/postgres-12/doc/src/sgml/runtime.sgml#L1542-L1566)), not a SQL one.
- This version has no `wiki/v12/common-concepts/` page for the buffer manager, the clock sweep, or buffer access strategies, so those concepts are explained inline here rather than linked. The concept pages should be filed as their own task.

## Source References

- [guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2150-L2163)
- [buf_init.c#InitBufferPool](../../../../raw/postgres-12/src/backend/storage/buffer/buf_init.c#L67-L152)
- [buf_init.c#BufferShmemSize](../../../../raw/postgres-12/src/backend/storage/buffer/buf_init.c#L160-L193)
- [freelist.c#StrategyGetBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L200-L358)
- [freelist.c#StrategyShmemSize](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L453-L465)
- [freelist.c#GetAccessStrategy](../../../../raw/postgres-12/src/backend/storage/buffer/freelist.c#L541-L588)
- [buf_table.c#BufTableShmemSize](../../../../raw/postgres-12/src/backend/storage/buffer/buf_table.c#L28-L45)
- [buf_internals.h#BM_MAX_USAGE_COUNT](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L69-L77)
- [buf_internals.h#BufMappingPartitionLock](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L121-L133)
- [buf_internals.h#BufferDescPadded](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L192-L218)
- [buf_internals.h#CkptSortItem](../../../../raw/postgres-12/src/include/storage/buf_internals.h#L283-L298)
- [lwlock.h#LWLOCK_MINIMAL_SIZE](../../../../raw/postgres-12/src/include/storage/lwlock.h#L61-L88)
- [lwlock.h:107-126](../../../../raw/postgres-12/src/include/storage/lwlock.h#L107-L126)
- [bufmgr.c#ReadBuffer_common](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L704-L915)
- [bufmgr.c#BufferAlloc](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L994-L1212)
- [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1774-L2038)
- [bufmgr.c#BgBufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2040-L2336)
- [bufmgr.c#CheckPointBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2574-L2593)
- [bufmgr.c#FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2668-L2785)
- [bufmgr.c#DropRelFileNodeBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2899-L2971)
- [bufmgr.c#DropRelFileNodesAllBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2973-L3072)
- [bufmgr.c#DropDatabaseBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3074-L3113)
- [bufmgr.c#FlushRelationBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3183-L3271)
- [bufmgr.c#FlushDatabaseBuffers](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3273-L3325)
- [README#Buffer Ring Replacement Strategy](../../../../raw/postgres-12/src/backend/storage/buffer/README#L208-L249)
- [checkpointer.c#CheckpointerShmemSize](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L889-L932)
- [checkpointer.c#CheckpointWriteDelay](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L648-L715)
- [checkpointer.c#ForwardSyncRequest](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L1086-L1160)
- [checkpointer.c#CompactCheckpointerRequestQueue](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L1162-L1268)
- [sysv_shmem.c#CreateAnonymousSegment](../../../../raw/postgres-12/src/backend/port/sysv_shmem.c#L521-L590)
- [sysv_shmem.c#PGSharedMemoryCreate](../../../../raw/postgres-12/src/backend/port/sysv_shmem.c#L609-L657)
- [ipci.c#CreateSharedMemoryAndSemaphores](../../../../raw/postgres-12/src/backend/storage/ipc/ipci.c#L94-L160)
- [xlog.c#XLOGChooseNumBuffers](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4850-L4873)
- [xlog.c#LogCheckpointEnd](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8435-L8442)
- [xlog.c#get_sync_bit](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L10019-L10035)
- [xlogdefs.h#PG_O_DIRECT](../../../../raw/postgres-12/src/include/access/xlogdefs.h#L60-L74)
- [xlog.h#XLogIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L181)
- [clog.c#CLOGShmemBuffers](../../../../raw/postgres-12/src/backend/access/transam/clog.c#L670-L679)
- [commit_ts.c#CommitTsShmemBuffers](../../../../raw/postgres-12/src/backend/access/transam/commit_ts.c#L465-L473)
- [hash.c#hashbuild](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L140-L158)
- [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L225-L260)
- [heapam.c#heap_sync](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8937-L8960)
- [heapam_handler.c:649](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L649)
- [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-12/src/backend/access/table/tableam.c#L370-L385)
- [vacuumlazy.c:1965](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1965)
- [vacuum.c:296](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L296)
- [autovacuum.c:2288](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2288)
- [smgr.c#smgrdounlink](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L375-L395)
- [smgr.c#smgrtruncate](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L640-L660)
- [storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L223-L300)
- [storage.c#smgr_redo](../../../../raw/postgres-12/src/backend/catalog/storage.c#L610-L625)
- [tablecmds.c:12778](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L12770-L12780)
- [dbcommands.c:942](../../../../raw/postgres-12/src/backend/commands/dbcommands.c#L942)
- [dbcommands.c:1228](../../../../raw/postgres-12/src/backend/commands/dbcommands.c#L1228)
- [dbcommands.c:2134](../../../../raw/postgres-12/src/backend/commands/dbcommands.c#L2134)
- [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L590-L600)
- [md.c#mdwrite](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L660-L670)
- [fd.c#FileRead](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1875-L1885)
- [fd.c#FileWrite](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1958-L1968)
- [costsize.c](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L22-L25)
- [cost.h#DEFAULT_EFFECTIVE_CACHE_SIZE](../../../../raw/postgres-12/src/include/optimizer/cost.h#L32)
- [system_views.sql#pg_settings](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L512-L513)
- [system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L882)
- [system_views.sql#pg_stat_bgwriter](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L935-L947)
- [pg_proc.dat#pg_show_all_settings](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5770-L5775)
- [pg_proc.dat#pg_size_pretty](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6897-L6899)
- [initdb.c#test_config_settings](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L947-L967)
- [initdb.c#test_config_settings](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1018-L1050)
- [pg_buffercache_pages.c](../../../../raw/postgres-12/contrib/pg_buffercache/pg_buffercache_pages.c#L126-L176)
- [pg_prewarm.c#pg_prewarm](../../../../raw/postgres-12/contrib/pg_prewarm/pg_prewarm.c#L185-L200)
- [autoprewarm.c](../../../../raw/postgres-12/contrib/pg_prewarm/autoprewarm.c#L1-L24)
- [config.sgml#shared_buffers](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1488-L1531)
- [config.sgml#huge_pages](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1533-L1590)
- [config.sgml#temp_buffers](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L1592-L1612)
- [runtime.sgml#linux-memory-overcommit](../../../../raw/postgres-12/doc/src/sgml/runtime.sgml#L1448-L1476)
- [runtime.sgml#linux-huge-pages](../../../../raw/postgres-12/doc/src/sgml/runtime.sgml#L1532-L1602)
- [016_min_consistency.pl:52](../../../../raw/postgres-12/src/test/recovery/t/016_min_consistency.pl#L52)
- [PostgresNode.pm:475](../../../../raw/postgres-12/src/test/perl/PostgresNode.pm#L475)

## Navigation

- [v12/index](../../index.md)
- [PostgreSQL 12 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
