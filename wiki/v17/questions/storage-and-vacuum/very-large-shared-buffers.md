---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Pros and Cons of a Very Large shared_buffers Such as 256 GB on a 1 TB RAM System in PostgreSQL 17, and What Changed Since PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [What 256 GiB of shared_buffers actually allocates](#what-256-gib-of-shared_buffers-actually-allocates)
  - [Pros](#pros)
  - [Cons](#cons)
  - [The edges: the floor, the ceiling, and a pool larger than the machine](#the-edges-the-floor-the-ceiling-and-a-pool-larger-than-the-machine)
  - [Thresholds that move when NBuffers is huge](#thresholds-that-move-when-nbuffers-is-huge)
  - [What a big pool does not buy](#what-a-big-pool-does-not-buy)
  - [What changed since PostgreSQL 12](#what-changed-since-postgresql-12)
  - [What did not change since PostgreSQL 12](#what-did-not-change-since-postgresql-12)
  - [Settings that move with it, and their apply scope](#settings-that-move-with-it-and-their-apply-scope)
  - [What to look at before and after the change](#what-to-look-at-before-and-after-the-change)
  - [Decision guide](#decision-guide)
  - [How this was measured, and what the measurements cannot say](#how-this-was-measured-and-what-the-measurements-cannot-say)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, what are the pros and cons of having a very large `shared_buffers`, like 256 GB or more, on a system with more than 1 TB of RAM? What has changed since PostgreSQL 12?

Prompt note, per `MANDATORY Prompt Hygiene`: the request read `follow agents.md, In PostgreSQL 17, what are the pros and cons of having a very large \`shared_buffers\`, like 256 GB or more, on a system with more than 1 TB of RAM?, what have changed since version 12.` The defects were `agents.md` for AGENTS.md, the lowercase sentence opening `follow`, a comma splice joining the instruction to the question with `In` capitalised mid-sentence, `?,` splicing two questions together, `what have changed` for `what has changed`, a terminating period on a question, and `version 12` for `PostgreSQL 12`. The asker chose **correct and restate**, chose to **build 17.11 and measure** rather than a source-only answer, chose to evidence the since-v12 part from **`raw/postgres-17/` citations plus that checkout's own commit history** rather than citing the v12 checkout, and scoped the delta to **large-pool mechanisms only**. All four choices are recorded here.

The PostgreSQL 12 answer to the same question is a separate page: [Pros and Cons of a Very Large shared_buffers Such as 256 GB on a 1 TB RAM System in PostgreSQL 12 (unverified)](../../../v12/questions/storage-and-vacuum/very-large-shared-buffers.md). This page does not cite it as evidence.

## Answer

### Short answer

**256 GiB is a legal setting that v17 will allocate, and the engine has more pool-size-dependent machinery than v12 did - but three things it must do per operation still walk every buffer, and source alone cannot say whether the trade is a win for a given workload.**

- `shared_buffers` is `PGC_POSTMASTER`, so any change needs a **restart**. The range is 16 blocks through `INT_MAX / 2` blocks, because the code "sometimes multiplies the number of shared buffers by two without checking for overflow" ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2257-L2270)).
- The shipped documentation still recommends 25 % of memory as a starting point and cautions that more than 40 % is unlikely to beat a smaller setting, and it warns that a larger pool usually needs a matching `max_wal_size` increase ([config.sgml#shared_buffers](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L1650-L1692)). 256 GiB is 25 % of 1 TiB.
- At the default 8 kB block size, `shared_buffers = '256GB'` is **33,554,432 buffers**, and the pinned 17.11 binary computes the whole shared-memory request as **267,499 MB**: 262,144 MB of pages plus **5,355 MB** of bookkeeping. That was measured without allocating anything, because `postgres -C shared_memory_size` prints the sizing answer and exits before `CreateSharedMemoryAndSemaphores()` runs ([postmaster.c:955-971](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L955-L971), [ipci.c#InitializeShmemGUCs](../../../../raw/postgres-17/src/backend/storage/ipc/ipci.c#L369-L398)).
- The measured costs that scale with the pool are a **checkpoint's full-header scan** (10.7 ns per buffer on the test host), **server start** (90 ns per buffer), and every **`DROP`/`TRUNCATE`** (5.9 ns per buffer, per pass). On a primary these last are not optimised away: the targeted-lookup shortcut added in v14 is reachable only in recovery.
- The single most surprising number is unchanged from v12: a sequential scan asks for the bulk-read ring and synchronised scanning only when the relation exceeds **`NBuffers / 4`** ([heapam.c#initscan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L408-L527)). At 256 GiB that line sits at **64 GiB**, so every smaller scan is free to evict the pool.

### What 256 GiB of shared_buffers actually allocates

Four arrays and one hash table are sized directly from `NBuffers` ([buf_init.c#InitBufferPool](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L67-L151), [buf_init.c#BufferShmemSize](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L159-L186), [freelist.c#StrategyShmemSize](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L453-L464)), and the checkpointer's sync-request queue is sized from it too ([checkpointer.c#CheckpointerShmemSize](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L882-L897)). The per-buffer column below is measured from `pg_shmem_allocations` on a running 16 GiB pool, not derived from the struct definitions.

| Shared memory region | Bytes per buffer (measured) | At 33,554,432 buffers | Source |
|---|---|---|---|
| `Buffer Blocks` | 8192.002 | 256 GiB + 4 kB alignment | [buf_init.c:82-86](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L82-L86) |
| `Buffer Descriptors` | 64.000 | 2 GiB | [buf_init.c:76-79](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L76-L79) |
| `Checkpointer Data` (sync-request slots) | 32.000, until the queue caps | 320 MB at the 10,000,000-slot cap | [checkpointer.c:886-895](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L886-L895) |
| `Checkpoint BufferIds` (sort array) | 20.000 | 640 MiB | [buf_init.c:101-103](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L101-L103) |
| `Buffer IO Condition Variables` | 16.000 | 512 MiB | [buf_init.c:88-92](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L88-L92) |
| `Shared Buffer Lookup Table` | 0.063 named, plus its elements and buckets outside the named row | see the marginal figure below | [freelist.c:478-488](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L478-L488), [buf_table.c#BufTableShmemSize](../../../../raw/postgres-17/src/backend/storage/buffer/buf_table.c#L41-L44) |

Because the mapping table's elements and bucket directory are allocated outside its named `pg_shmem_allocations` row, the honest total comes from differencing two `-C shared_memory_size` answers. Measured marginal cost, from 1,000,000-block pairs:

| Block-count pair | Marginal bytes per buffer | What is happening |
|---|---|---|
| 20,000,000 -> 21,000,000 | **8331** | no boundary crossed, request queue already capped |
| 32,000,000 -> 33,000,000 | **8331** | same |
| 9,000,000 -> 10,000,000 | **8364** | request queue still growing, so +32 bytes per buffer |
| 10,000,000 -> 11,000,000 | **8331** | the queue stops growing at `MAX_CHECKPOINT_REQUESTS` |
| 33,000,000 -> 34,000,000 | **8601** | the mapping table's bucket array doubles inside this range |

**256 GiB lands on the wrong side of a power-of-two step, and the step is 257 MB.** The mapping table is sized `NBuffers + NUM_BUFFER_PARTITIONS` ([freelist.c:478-488](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L478-L488), [lwlock.h:87-93](../../../../raw/postgres-17/src/include/storage/lwlock.h#L87-L93)), 128 is added to a 2^25 pool, and dynahash rounds the bucket count up to a power of two. Measured, with nothing allocated:

| `shared_buffers` in blocks | `shared_memory_size` | Delta |
|---|---|---|
| 33,554,304 (2^25 - 128) | 267,241 MB | - |
| 33,554,305 | 267,498 MB | **+257 MB** |
| 33,554,432 (256 GiB exactly) | 267,499 MB | +1 MB |

So `shared_buffers = '255GB'` (33,423,360 blocks) buys back that 257 MB. Whether 257 MB matters on a 1 TiB host is a judgement, not a source claim.

Huge pages: the same binary reports **133,750** pages for 256 GiB at this host's 2 MiB `Hugepagesize`, or **262** pages with `huge_page_size = '1GB'` ([guc_tables.c#huge_page_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3598-L3607), [ipci.c#InitializeShmemGUCs](../../../../raw/postgres-17/src/backend/storage/ipc/ipci.c#L369-L398)). Both are `PGC_POSTMASTER`: **restart**.

### Pros

**A hit costs a partition lock and a pin, never an `smgr` call.** `ReadBuffer_common()` reaches `PinBufferForBlock()` and returns when the page is already valid; only a miss reaches `WaitReadBuffers()` -> `smgrreadv()` ([bufmgr.c#ReadBuffer_common](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1196-L1264), [bufmgr.c#WaitReadBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1411-L1588)). More resident pages means more of this path, though source cannot say how many more for a given workload.

**A dirty victim is written by whichever backend needs the buffer, and that write forces WAL first.** `GetVictimBuffer()` flushes the page it wants to reuse, and `FlushBuffer()` calls `XLogFlush()` up to the page LSN before `smgrwrite()` ([bufmgr.c#GetVictimBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1954-L2105), [bufmgr.c:3927](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3927), [bufmgr.c:3948](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3948)). A pool that holds the working set makes that foreground write rarer; since v16 it is also countable, as `pg_stat_io`'s `evictions` and `reuses` ([bufmgr.c:2061-2081](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2061-L2081), [system_views.sql#pg_stat_io](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1153-L1173)).

**A checkpoint writes only what was dirty when it started.** `BufferSync()` marks the dirty set with `BM_CHECKPOINT_NEEDED` in one pass and writes that set, so a page dirtied repeatedly between checkpoints is not written once per dirtying ([bufmgr.c#BufferSync](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2931-L3193)). A larger pool can absorb more re-dirtying before a write is forced, at the cost of a larger dirty set to write when the checkpoint comes.

**Hot pages survive the sweep.** `usage_count` rises to `BM_MAX_USAGE_COUNT` (5) and the clock sweep must decrement it to zero before the buffer can be taken ([buf_internals.h:71-79](../../../../raw/postgres-17/src/include/storage/buf_internals.h#L71-L79), [freelist.c#StrategyGetBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L196-L357)).

**The strategy rings scale with the pool instead of shrinking it.** Every ring is capped at `NBuffers / 8`, so at 256 GiB a `VACUUM` may be told to use up to 16 GiB of ring and actually get it ([freelist.c#GetAccessStrategyWithSize](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L584-L614)). Measured at a 1 GiB pool on a 37,384-block table, `VACUUM (BUFFER_USAGE_LIMIT ...)` left this many of the table's blocks resident:

| `BUFFER_USAGE_LIMIT` | Blocks of the table left in the pool |
|---|---|
| `'128kB'` | 31 |
| `'2MB'` (the default) | 271 |
| `'256MB'` | 16,399 - the `NBuffers / 8` cap is 16,384 |
| `'0'` (unlimited) | 41,137, the whole table |

**Per-backend pin budgets grow with the pool.** Batched pin limits are `NBuffers / (MaxBackends + NUM_AUXILIARY_PROCS)`, which read streams and relation extension both consult; at a small pool this is the binding limit, at 256 GiB it is not ([bufmgr.c#LimitAdditionalPins](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2120-L2144), [read_stream.c:452-478](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c#L452-L478)).

**Three SLRU caches grow with the pool, up to a cap.** `transaction_buffers`, `commit_timestamp_buffers` and `subtransaction_buffers` default to `NBuffers / 512` rounded down to a 16-slot bank and capped at 1024 blocks ([slru.c#SimpleLruAutotuneBuffers](../../../../raw/postgres-17/src/backend/access/transam/slru.c#L232-L237), [clog.c#CLOGShmemBuffers](../../../../raw/postgres-17/src/backend/access/transam/clog.c#L768-L775)). Measured: 32 blocks at a 128 MB pool, 256 at 1 GB, and 1024 - the cap - from 4 GB upward, so at 256 GiB each of the three is 8 MiB and no larger unless set explicitly.

**The sync-request queue effectively cannot fill.** It is `Min(NBuffers, 10000000)` slots; a backend only fsyncs for itself when the queue is full and compaction finds no duplicates ([checkpointer.c#ForwardSyncRequest](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L1099-L1141)).

### Cons

**Every checkpoint locks and inspects all 33.5 million buffer headers before writing anything.** `BufferSync()`'s first loop takes the header spinlock on each buffer, then it sorts only the dirty set ([bufmgr.c#BufferSync](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2931-L3193)). Measured over an idle, clean pool, six checkpoints each:

| Pool | Buffers | Median `CHECKPOINT` |
|---|---|---|
| 128 MB | 16,384 | 7.1 ms |
| 1 GB | 131,072 | 9.5 ms |
| 4 GB | 524,288 | 13.2 ms |
| 16 GB | 2,097,152 | 30.5 ms |

The slope between the last two rows is **10.7 ns per buffer**, which extrapolates to about **0.36 s of pure scan per checkpoint at 33,554,432 buffers**. The extrapolation is arithmetic on this host's slope, not a measurement of a 256 GiB pool.

**`DROP` and `TRUNCATE` scan the whole pool on a primary, once per pass.** `DropRelationBuffers()` and `DropRelationsAllBuffers()` will use targeted `BufMapping` lookups when the relation's size is cached and the work is below `BUF_DROP_FULL_SCAN_THRESHOLD` (`NBuffers / 32`), but `smgrnblocks_cached()` returns `InvalidBlockNumber` unless `InRecovery`, so on a primary the shortcut is unreachable ([bufmgr.c:84-89](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L84-L89), [bufmgr.c#DropRelationBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4111-L4223), [bufmgr.c#FindAndDropRelationBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4405-L4452), [smgr.c#smgrnblocks_cached](../../../../raw/postgres-17/src/backend/storage/smgr/smgr.c#L679-L690)). Measured on a primary, medians over 30 runs of a one-block table, with the commit inside the timed interval because the pass happens in `smgrDoPendingDeletes()` at commit ([storage.c#smgrDoPendingDeletes](../../../../raw/postgres-17/src/backend/catalog/storage.c#L657-L719)):

| Pool | Buffers | `DROP TABLE`, own transaction | `TRUNCATE`, main fork only | `TRUNCATE`, main + vm + fsm |
|---|---|---|---|---|
| 128 MB | 16,384 | 0.65 ms | 0.91 ms | 1.44 ms |
| 1 GB | 131,072 | 1.29 ms | 1.60 ms | 2.14 ms |
| 4 GB | 524,288 | 3.93 ms | 5.68 ms | 6.38 ms |
| 16 GB | 2,097,152 | 12.93 ms | 18.47 ms | 17.56 ms |

That is **5.9 ns per buffer per `DROP`**, or about **0.2 s per `DROP` at 33,554,432 buffers** by the same arithmetic. Two details matter operationally. First, batching helps: dropping 30 one-block tables in one transaction cost **35.7 ms** at 2,097,152 buffers against 30 x 12.93 ms = 388 ms one at a time, because `smgrdounlinkall()` hands the whole pending set to one pass ([smgr.c#smgrdounlinkall](../../../../raw/postgres-17/src/backend/storage/smgr/smgr.c#L462-L522)). Second, `TRUNCATE`'s cost did **not** multiply with the fork count, because it replaces the relfilenode and the old one is dropped once at commit; the per-fork `smgrtruncate()` path that does run a pass per fork is `VACUUM`'s truncation and its replay ([storage.c#RelationTruncate](../../../../raw/postgres-17/src/backend/catalog/storage.c#L288-L439), [storage.c#smgr_redo](../../../../raw/postgres-17/src/backend/catalog/storage.c#L965-L1079)).

**Six maintenance routines still walk every buffer, and the tree says so itself.** `FlushRelationBuffers()` carries the comment **`XXX currently it sequentially searches the buffer pool, should be changed to more clever ways of searching`**, with the rationale that these paths are not performance-critical - a judgement made for pools far smaller than this one ([bufmgr.c:4553-4568](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4553-L4568)).

| Routine | Reached by | Source |
|---|---|---|
| `DropRelationBuffers` | `smgrtruncate()`, so `VACUUM` truncation and its replay | [bufmgr.c#DropRelationBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4111-L4223) |
| `DropRelationsAllBuffers` | pending deletes at commit: `DROP`, `TRUNCATE`, `REINDEX`, rewriting DDL | [bufmgr.c#DropRelationsAllBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4234-L4393) |
| `DropDatabaseBuffers` | `DROP DATABASE`, `ALTER DATABASE ... SET TABLESPACE`, the end of a `STRATEGY wal_log` `CREATE DATABASE`, and replay of a drop | [bufmgr.c#DropDatabaseBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4466-L4493), [dbcommands.c:1600-1611](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L1600-L1611), [dbcommands.c#dropdb](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L1634-L1856), [dbcommands.c#movedb](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L1964-L2282) |
| `FlushRelationBuffers` | heap and index copies for `ALTER TABLE`/`ALTER INDEX ... SET TABLESPACE`, and an unlogged sequence's init fork | [bufmgr.c#FlushRelationBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4572-L4658), [heapam_handler.c:637-645](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L637-L645), [tablecmds.c:15586-15595](../../../../raw/postgres-17/src/backend/commands/tablecmds.c#L15586-L15595), [sequence.c:344-352](../../../../raw/postgres-17/src/backend/commands/sequence.c#L344-L352) |
| `FlushRelationsAllBuffers` | `smgrdosyncall()`, the `wal_level = minimal` sync path | [bufmgr.c#FlushRelationsAllBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4670-L4757) |
| `FlushDatabaseBuffers` | `CREATE DATABASE` replay of the file-copy strategy | [bufmgr.c#FlushDatabaseBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4925-L4960), [dbcommands.c#dbase_redo](../../../../raw/postgres-17/src/backend/commands/dbcommands.c#L3270-L3432) |

**Server start pays for initialising every header, serially.** `InitBufferPool()` clears the tag, initialises the atomic state, the content `LWLock` and the condition variable for each buffer in one loop ([buf_init.c#InitBufferPool](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L67-L151)). The postmaster logs `starting PostgreSQL` only *after* `CreateSharedMemoryAndSemaphores()` ([postmaster.c:980](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L980), [postmaster.c:1083-1084](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L1083-L1084)), so the interval that contains this work is the one before that line:

| Pool | Buffers | Shell clock to `starting PostgreSQL` | `starting` to `ready to accept connections` |
|---|---|---|---|
| 128 MB | 16,384 | 15 ms | 8 ms |
| 1 GB | 131,072 | 28 ms | 11 ms |
| 4 GB | 524,288 | 49 ms | 10 ms |
| 16 GB | 2,097,152 | 206 ms | 9 ms |

That is **90 ns per buffer**, about **3 s at 33,554,432 buffers** by extrapolation, and it is paid on every restart - including the restart that `shared_buffers` itself requires. The second column is flat, so this is pool initialisation and mapping, not startup-process work.

**The pool is not a substitute for the operating system cache, and by default the data is in both.** Relation data is read with ordinary buffered `preadv()` unless `debug_io_direct` is set, which is a `PGC_POSTMASTER` developer option ([guc_tables.c#debug_io_direct](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4699-L4708), [config.sgml#debug_io_direct](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L11595-L11620)). Turning it on removes the second copy, and with it the second cache. Two 37,384-block bulk-read scans on this host, each from an empty pool, bracket the difference: buffered, with the file in the page cache, `pg_stat_io` recorded **43.7 ms** of `read_time`; with `debug_io_direct = data`, it recorded **522.2 ms** for the same block count. The two scans are different fixtures at different `io_combine_limit` settings (`combine` at 8 kB, `above` at the default 128 kB), so treat the pair as the order of magnitude the page cache was worth here, not as a controlled A/B.

**`pg_buffercache`'s row-per-buffer reader costs one `BufferCachePagesRec` per buffer, collected before the first row is returned.** Measured at 2,097,152 buffers, median of three: **445.8 ms** for `SELECT count(*) FROM pg_buffercache`, against **8.0 ms** for `pg_buffercache_summary()` and **7.5 ms** for `pg_buffercache_usage_counts()` ([pg_buffercache_pages.c#pg_buffercache_pages](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache_pages.c#L69-L246), [pg_buffercache_pages.c#pg_buffercache_summary](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache_pages.c#L249-L313)). Scaled to 33,554,432 buffers that is a multi-second query, and - as arithmetic on the record layout for a 64-bit build, where the struct's ten fields pack into 32 bytes - a 1 GiB temporary array. A monitoring job that selects from `pg_buffercache` every minute is therefore a different proposition at 256 GiB than at 1 GiB.

**Autoprewarm's dump repeats an `NBuffers`-sized allocation on a timer.** `apw_dump_now()` allocates `sizeof(BlockInfoRecord) * NBuffers` with `MCXT_ALLOC_HUGE` and walks every header; `pg_prewarm.autoprewarm_interval` defaults to 300 s and is `PGC_SIGHUP`, so with the module preloaded this repeats every five minutes until it is set to 0 ([autoprewarm.c#apw_dump_now](../../../../raw/postgres-17/contrib/pg_prewarm/autoprewarm.c#L570-L713), [autoprewarm.c:104-120](../../../../raw/postgres-17/contrib/pg_prewarm/autoprewarm.c#L104-L120)).

**Replacement work is unbounded in wall-clock terms.** The clock sweep's stated bound is `BM_MAX_USAGE_COUNT + 1` complete cycles, and the `trycounter` resets on every usage-count decrement, so one allocation can examine far more than `NBuffers` headers under concurrency ([freelist.c#StrategyGetBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L196-L357), [freelist.c#ClockSweepTick](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L108-L164)). One sweep pass over 33.5 M buffers is 33.5 M atomic increments and header locks.

**Background cleaning does not scale itself up.** `bgwriter_lru_maxpages` defaults to 100 pages per round and `bgwriter_delay` to 200 ms, neither derived from `NBuffers`, while the round's own floor target is `NBuffers / (120000 / bgwriter_delay)` - **55,924 buffers** at this size and the default delay ([guc_tables.c#bgwriter_lru_maxpages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3088-L3096), [guc_tables.c#bgwriter_delay](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3077-L3086), [bufmgr.c#BgBufferSync](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3207-L3488)). The writer aims at a target three orders of magnitude above what it is allowed to write per round, so at this pool size the cleaning that matters is done by backends and the checkpointer.

**Fixed partitioning does not widen with the pool.** The mapping table has `NUM_BUFFER_PARTITIONS` = 128 partition locks regardless of size, and the strategy control block is guarded by a single `buffer_strategy_lock` spinlock ([lwlock.h:87-93](../../../../raw/postgres-17/src/include/storage/lwlock.h#L87-L93), [freelist.c:30-62](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L30-L62)).

**Memory committed here is memory `work_mem`, `maintenance_work_mem` and the page cache cannot use**, and the pool is not sized against them by any code in the tree. The planner never reads `NBuffers`; it reads `effective_cache_size`, which is a separate `PGC_USERSET` estimate of Postgres plus OS cache ([costsize.c:22-26](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L22-L26), [guc_tables.c#effective_cache_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3508-L3518)).

**There is no NUMA awareness anywhere in the tree.** A grep for `NUMA`, `libnuma` or `numa_available` across `src/`, `contrib/` and `configure.ac` in this checkout returns nothing, so a 256 GiB segment's placement across sockets is entirely the kernel's business.

**Nothing in the shipped tests exercises a large pool.** The only `shared_buffers` settings in the test tree are `128kB` in [016_min_consistency.pl:55](../../../../raw/postgres-17/src/test/recovery/t/016_min_consistency.pl#L55), `1MB` in [032_relfilenode_reuse.pl:21](../../../../raw/postgres-17/src/test/recovery/t/032_relfilenode_reuse.pl#L21) and `256kB` in [004_io_direct.pl:47](../../../../raw/postgres-17/src/test/modules/test_misc/t/004_io_direct.pl#L47). `contrib/pg_buffercache`'s own test only checks that its views agree with the setting ([pg_buffercache.sql](../../../../raw/postgres-17/contrib/pg_buffercache/sql/pg_buffercache.sql#L1-L12)).

### The edges: the floor, the ceiling, and a pool larger than the machine

All four probes below are measured on the pinned build.

| Probe | Result |
|---|---|
| `shared_buffers = 15` | `FATAL: 15 8kB is outside the valid range for parameter "shared_buffers" (16 8kB .. 1073741823 8kB)` |
| `shared_buffers = 16` | starts; `shared_memory_size` = 8 MB |
| `shared_buffers = 1073741823` (the ceiling, just under 8 TiB) | accepted; `shared_memory_size` = 8,548,777 MB |
| `shared_buffers = 1073741824` | `FATAL: 1073741824 8kB is outside the valid range ...` |
| `shared_buffers = '200GB'` on a 31 GiB host | `FATAL: could not map anonymous shared memory: Cannot allocate memory`, with the hint naming the request as 219,065,999,360 bytes |
| `shared_buffers = '1GB'`, `huge_pages = on`, no huge pages configured | the same `could not map anonymous shared memory` FATAL; the hint is the only mention of huge pages |

The range comes from the GUC definition ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2257-L2270)) and the mapping failure from `CreateAnonymousSegment()`, which retries without `MAP_HUGETLB` only when `huge_pages` is not `on`, and otherwise reports the error with that hint ([sysv_shmem.c#CreateAnonymousSegment](../../../../raw/postgres-17/src/backend/port/sysv_shmem.c#L599-L668)). The practical consequence for a 1 TiB host is that a mistyped `2560GB` is a failure to start, not a degraded start, and `huge_pages = on` converts a missing huge-page reservation into the same failure - which is why v15's `shared_memory_size_in_huge_pages` exists ([guc_tables.c#shared_memory_size_in_huge_pages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2294-L2303)).

### Thresholds that move when NBuffers is huge

| Derived value | Formula | At 33,554,432 buffers | Source |
|---|---|---|---|
| Bulk-read ring and synchronised scanning kick in | relation > `NBuffers / 4` | 8,388,608 blocks = **64 GiB** | [heapam.c#initscan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L408-L527), [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L389-L404) |
| Any strategy ring's cap | `NBuffers / 8` | 4,194,304 buffers = **32 GiB** | [freelist.c#GetAccessStrategyWithSize](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L584-L614) |
| Targeted drop instead of a full scan (recovery only) | blocks < `NBuffers / 32` | 1,048,576 blocks = 8 GiB | [bufmgr.c:84-89](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L84-L89) |
| Batched pin budget per backend | `NBuffers / (MaxBackends + NUM_AUXILIARY_PROCS)` | 260,111 at `max_connections = 100` and default worker and sender counts, where `MaxBackends` is 123 and `NUM_AUXILIARY_PROCS` 6 | [bufmgr.c#LimitAdditionalPins](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2120-L2144), [postinit.c:579-584](../../../../raw/postgres-17/src/backend/utils/init/postinit.c#L579-L584), [proc.h:431-443](../../../../raw/postgres-17/src/include/storage/proc.h#L431-L443) |
| `wal_buffers` when `-1` | `NBuffers / 32`, capped at one WAL segment | 16 MiB | [xlog.c#XLOGChooseNumBuffers](../../../../raw/postgres-17/src/backend/access/transam/xlog.c#L4575-L4585) |
| `transaction_buffers`, `commit_timestamp_buffers`, `subtransaction_buffers` when `0` | `NBuffers / 512`, bank-aligned, capped at 1024 blocks | 8 MiB each | [slru.c#SimpleLruAutotuneBuffers](../../../../raw/postgres-17/src/backend/access/transam/slru.c#L232-L237) |
| Background writer's per-round floor target | `NBuffers / (120000 / bgwriter_delay)` | 55,924 buffers, against a 100-page write cap | [bufmgr.c#BgBufferSync](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3207-L3488) |
| Checkpointer sync-request slots | `Min(NBuffers, 10000000)` | 10,000,000 | [checkpointer.c:133-134](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L133-L134) |

The `NBuffers / 4` line is the one to internalise. Measured at a 1 GiB pool, where the line is 32,768 blocks:

| Fixture | Blocks | Context used | Blocks of it left in the pool after one seq scan |
|---|---|---|---|
| `below` | 18,692 (under the line) | `normal` | **18,692** - all of it |
| `above` | 37,384 (over the line) | `bulkread`, 37,352 `reuses` | **32** - exactly the 256 kB ring |

### What a big pool does not buy

- **Temporary tables.** They live in per-backend local buffers sized by `temp_buffers`, a `PGC_USERSET` setting with a 1024-block default and its own pin limit ([guc_tables.c#temp_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2382-L2391), [localbuf.c#LimitAdditionalLocalPins](../../../../raw/postgres-17/src/backend/storage/buffer/localbuf.c#L290-L306)).
- **`VACUUM`'s default footprint.** Its ring is 2 MB unless raised ([guc_tables.c#vacuum_buffer_usage_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2272-L2281)).
- **Planner cost estimates.** The planner reads `effective_cache_size`, never `NBuffers` ([costsize.c:22-26](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c#L22-L26)).
- **Read combining you can see.** `pg_stat_io.reads` counts *blocks*: `WaitReadBuffers()` passes `io_buffers_len` to `pgstat_count_io_op_time()`, and `op_bytes` is hard-coded to `BLCKSZ` ([bufmgr.c:1519-1521](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1519-L1521), [pgstatfuncs.c:1425-1431](../../../../raw/postgres-17/src/backend/utils/adt/pgstatfuncs.c#L1425-L1431)). Measured, the same scan reported **37,489 reads at every one** of `io_combine_limit` = 8 kB, 32 kB, 128 kB and 256 kB.
- **A warm pool after a restart.** `pg_prewarm` loads pages on demand and autoprewarm stores block identifiers, not contents ([pg_prewarm.c#pg_prewarm](../../../../raw/postgres-17/contrib/pg_prewarm/pg_prewarm.c#L76-L291), [autoprewarm.c#autoprewarm_database_main](../../../../raw/postgres-17/contrib/pg_prewarm/autoprewarm.c#L437-L561)).

### What changed since PostgreSQL 12

Scoped to mechanisms that change the pros and cons at 256 GiB. Each row is attributed to the commit in this checkout's own history and to the first release tag that contains it.

| First in | Change | Why it matters at 256 GiB | Commit |
|---|---|---|---|
| 13 | `pg_shmem_allocations` view | the pool's own bookkeeping became inspectable, which is how the per-buffer table above was measured | `ed10f32e37e` |
| 13 | `FlushRelationsAllBuffers()` added | a sixth full-pool scan exists that v12 did not have | `cb2fd7eac28` |
| 14 | `BUF_DROP_FULL_SCAN_THRESHOLD` plus `smgrnblocks_cached()` | drop and truncate can use targeted lookups **in recovery only**; a standby's replay of a drop no longer scans 33.5 M headers, a primary's still does | `d6ad34f3410`, `bea449c635c` |
| 14 | per-buffer I/O locks replaced by condition variables | per-buffer overhead for that array is the measured 16 bytes, not an `LWLock` | `d87251048a0` |
| 14 | `huge_page_size` GUC | 1 GiB pages become selectable, taking the 256 GiB mapping from 133,750 page reservations to 262 | `d2bddc2500f` |
| 14 | `checkpoint_completion_target` default 0.5 -> 0.9 | the larger dirty set a big pool permits is spread over more of the interval by default | `bbcc4eb2e08` |
| 15 | `shared_memory_size` and `shared_memory_size_in_huge_pages` | the sizing of an unallocatable pool can be measured before committing to it; every 256 GiB figure on this page depends on it | `bd1788051b0`, `43c1c4f65ea` |
| 15 | built-in `shared_buffers` default 8 MB -> 128 MB | the baseline this decision is compared against moved | `f7bda63a487` |
| 15 | `CREATE DATABASE ... STRATEGY wal_log`, and the default | the default path copies block by block through the pool instead of forcing the file-copy strategy's checkpoints | `9c08aea6a30` |
| 15 | cumulative statistics moved into shared memory | prerequisite for the per-context I/O view below | `5891c7a8ed8` |
| 16 | `pg_stat_io` | evictions, reuses, hits and per-context reads became countable, so "is the pool too small" stops being a guess | `a9c70b46dbe` |
| 16 | `vacuum_buffer_usage_limit` and `GetAccessStrategyWithSize()` | `VACUUM`'s ring is configurable and capped at `NBuffers / 8`, so a big pool can be spent on maintenance deliberately; the default was raised to 2 MB in 17 | `1cbbee03385`, `98f320eb2ef` |
| 16 | `pg_buffercache_summary()` and `pg_buffercache_usage_counts()` | a pool census without the `NBuffers`-sized allocation: measured 8.0 ms against 445.8 ms at 2 M buffers | `2589434ae0f`, `f3fa31327ec` |
| 16 | `PG_IO_ALIGN_SIZE` | `Buffer Blocks` carries 4 kB of alignment padding, and direct I/O becomes possible | `faeedbcefd4` |
| 16 | `debug_io_direct` | double buffering becomes a choice rather than a fact; measured at 522.2 ms against 43.7 ms of `read_time` for the same scan | `319bae9a8da` |
| 16 | `LimitAdditionalPins()` | per-backend batch pin budget derived from `NBuffers`, so a large pool raises the ceiling on batched work | `31966b151e6` |
| 17 | read streams, vectored reads, `io_combine_limit`, and strategy pin limits | misses are issued as combined reads with adaptive look-ahead, and ring users cannot pin their way out of the ring | `b5a9b18cd0b`, `210622c60e1`, `b7b0f3f2724`, `041b96802ef`, `3bd8439ed62` |
| 17 | SLRU auto-sizing GUCs and bank-partitioned SLRU locks | `transaction_buffers` and two others now scale with `shared_buffers` to a 1024-block cap and can be raised to 1 GiB | `53c2a97a926` |
| 17 | `pg_stat_checkpointer`, and `buffers_backend`/`buffers_backend_fsync` removed from `pg_stat_bgwriter` | monitoring built on v12's `pg_stat_bgwriter` columns breaks; the replacement is `pg_stat_io` | `96f052613f3`, `74604a37f2f` |
| 17 | `pg_buffercache_evict()` | a pool can be perturbed deliberately under test | `13453eedd3f` |
| 17 | `huge_pages_status` | whether the running server actually got huge pages is now readable | `a14354cac0e` |
| 17.6 | `MAX_CHECKPOINT_REQUESTS` caps the sync-request queue at 10,000,000 slots | above a 76.3 GiB pool the queue stops growing; visible as the measured marginal drop from 8364 to 8331 bytes per buffer | `13559de9538`, `60589003480` |

### What did not change since PostgreSQL 12

Verified against this checkout's history over the range `REL_12_2..786db8dcf16`; every claim below is cited to the v17 source, and the v12 side is a history statement, not a citation.

- The GUC itself: `PGC_POSTMASTER`, floor 16, ceiling `INT_MAX / 2`, and the same overflow comment ([guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2257-L2270)).
- The `NBuffers / 4` bulk-read and synchronised-scan threshold ([heapam.c#initscan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L408-L527)).
- The 256 kB bulk-read ring and the 128 kB sync-scan report interval ([freelist.c#GetAccessStrategy](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L541-L574), [syncscan.c:73-83](../../../../raw/postgres-17/src/backend/access/common/syncscan.c#L73-L83)).
- `BM_MAX_USAGE_COUNT` = 5 and the shape of the clock sweep ([buf_internals.h:71-79](../../../../raw/postgres-17/src/include/storage/buf_internals.h#L71-L79), [freelist.c#StrategyGetBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L196-L357)).
- `NUM_BUFFER_PARTITIONS` = 128 and the single `buffer_strategy_lock` ([lwlock.h:87-93](../../../../raw/postgres-17/src/include/storage/lwlock.h#L87-L93), [freelist.c:30-62](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L30-L62)).
- `BufferSync()`'s full-header scan per checkpoint ([bufmgr.c#BufferSync](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2931-L3193)).
- `bgwriter_delay` 200 ms and `bgwriter_lru_maxpages` 100, neither derived from `NBuffers` ([guc_tables.c#bgwriter_delay](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3077-L3086), [guc_tables.c#bgwriter_lru_maxpages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3088-L3096)).
- The documentation's 25 % starting point and 40 % caution ([config.sgml#shared_buffers](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L1650-L1692)).
- No NUMA awareness, and no test that exercises a large pool.

### Settings that move with it, and their apply scope

| Setting | v17 default | Context | Apply scope | Relevance at 256 GiB |
|---|---|---|---|---|
| `shared_buffers` | 16384 blocks (128 MB) | `PGC_POSTMASTER` | **restart** | the decision itself |
| `huge_pages` | `try` | `PGC_POSTMASTER` | **restart** | `on` turns a missing reservation into a failure to start |
| `huge_page_size` | 0 (platform default) | `PGC_POSTMASTER` | **restart** | 1 GiB pages cut the reservation count from 133,750 to 262 |
| `huge_pages_status` | n/a, read-only | `PGC_INTERNAL` | read-only | whether the running server got them |
| `shared_memory_size`, `shared_memory_size_in_huge_pages` | n/a, read-only | `PGC_INTERNAL` | read-only, and readable via `postgres -C` before starting | plan the allocation |
| `max_wal_size` | 1024 MB | `PGC_SIGHUP` | **reload** | the docs tie a larger pool to raising this |
| `checkpoint_timeout`, `checkpoint_completion_target` | 300 s, 0.9 | `PGC_SIGHUP` | **reload** | spread the larger dirty set |
| `checkpoint_flush_after` | 32 blocks where the platform supports it, else 0 ([pg_config_manual.h:170-180](../../../../raw/postgres-17/src/include/pg_config_manual.h#L170-L180)) | `PGC_SIGHUP` | **reload** | writeback pacing during the checkpoint |
| `bgwriter_delay`, `bgwriter_lru_maxpages`, `bgwriter_lru_multiplier` | 200 ms, 100, 2.0 | `PGC_SIGHUP` | **reload** | the only way to make background cleaning scale with the pool |
| `backend_flush_after` | 0 (off) | `PGC_SIGHUP` | **reload** | pacing for the writes backends do themselves |
| `vacuum_buffer_usage_limit` | 2048 kB | `PGC_USERSET` | session/transaction, or per command via `BUFFER_USAGE_LIMIT` | spend pool on maintenance, up to `NBuffers / 8` |
| `io_combine_limit` | 16 blocks (128 kB) | `PGC_USERSET` | session/transaction | size of a combined read on a miss |
| `effective_io_concurrency`, `maintenance_io_concurrency` | 1 and 10 where the build supports prefetching, 0 and 0 otherwise | `PGC_USERSET` | session/transaction | look-ahead depth of the read streams ([bufmgr.h:157-166](../../../../raw/postgres-17/src/include/storage/bufmgr.h#L157-L166)) |
| `effective_cache_size` | 524288 blocks (4 GB) | `PGC_USERSET` | session/transaction | the planner's cache estimate; it never reads `NBuffers` |
| `temp_buffers` | 1024 blocks | `PGC_USERSET` | session, before first temp use | temp tables do not use the pool |
| `transaction_buffers`, `commit_timestamp_buffers`, `subtransaction_buffers` | 0 = auto | `PGC_POSTMASTER` | **restart** | auto-size to `NBuffers / 512`, capped at 1024 blocks |
| `multixact_offset_buffers`, `multixact_member_buffers`, `notify_buffers`, `serializable_buffers` | 16, 32, 16, 32 | `PGC_POSTMASTER` | **restart** | fixed; a big pool does not raise them |
| `debug_io_direct` | empty | `PGC_POSTMASTER` | **restart** | removes the OS cache from the data path; a developer option |
| `track_io_timing` | off | `PGC_SUSET` | reload, or session for a superuser | needed for `read_time`/`write_time` in `pg_stat_io` |
| `pg_prewarm.autoprewarm` | on | `PGC_POSTMASTER` | **restart** | whether the `NBuffers`-sized dump worker runs at all |
| `pg_prewarm.autoprewarm_interval` | 300 s | `PGC_SIGHUP` | **reload** | how often it repeats |

Contexts are read from the pinned GUC table: [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2257-L2270), [guc_tables.c#vacuum_buffer_usage_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2272-L2281), [guc_tables.c#shared_memory_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2283-L2292), [guc_tables.c#commit_timestamp_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2305-L2314), [guc_tables.c#subtransaction_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2360-L2369), [guc_tables.c#transaction_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2371-L2380), [guc_tables.c#max_wal_size](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2842-L2852), [guc_tables.c#checkpoint_flush_after](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2880-L2889), [guc_tables.c#io_combine_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3138-L3150), [guc_tables.c#backend_flush_after](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3152-L3161), [guc_tables.c#checkpoint_completion_target](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3916-L3924), [guc_tables.c#huge_pages](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5057-L5065), [guc_tables.c#huge_pages_status](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5067-L5076).

### What to look at before and after the change

Verified against the pinned catalogs. Each statement sets session-scoped timeouts first; both are `PGC_USERSET`.

```sql
SET /* wiki_shbuf_session */ statement_timeout = '30s';
SET /* wiki_shbuf_session */ lock_timeout = '5s';

-- 1. What the current pool costs, and what the proposed one would.
SELECT /* wiki_shbuf_sizing */ name, setting, unit
FROM pg_settings
WHERE name IN ('shared_buffers', 'shared_memory_size',
               'shared_memory_size_in_huge_pages', 'huge_pages',
               'huge_pages_status', 'huge_page_size');

-- 2. Where the pool's shared memory actually goes.
SELECT /* wiki_shbuf_shmem */ coalesce(name, '<anonymous>') AS name,
       pg_size_pretty(size) AS size
FROM pg_shmem_allocations
ORDER BY size DESC
LIMIT 15;

-- 3. Is the pool too small?  Evictions and reuses are the signal, and this
--    view replaces pg_stat_bgwriter.buffers_backend, which v17 removed.
SELECT /* wiki_shbuf_io */ backend_type, object, context,
       reads, hits, evictions, reuses, writes, fsyncs
FROM pg_stat_io
WHERE reads > 0 OR writes > 0 OR evictions > 0
ORDER BY backend_type, context;

-- 4. A pool census that does not allocate one record per buffer.
SELECT /* wiki_shbuf_census */ * FROM pg_buffercache_summary();
SELECT /* wiki_shbuf_usage */ * FROM pg_buffercache_usage_counts();

-- 5. Checkpoint pressure: requested checkpoints are the ones max_wal_size
--    forced, and buffers_written is the dirty set the pool handed over.
SELECT /* wiki_shbuf_ckpt */ num_timed, num_requested, buffers_written,
       write_time, sync_time, stats_reset
FROM pg_stat_checkpointer;
```

Before starting a server with the new value, size it without allocating it:

```bash
postgres -D "$PGDATA" -c shared_buffers=256GB -C shared_memory_size
postgres -D "$PGDATA" -c shared_buffers=256GB -C shared_memory_size_in_huge_pages
```

Privileges: `pg_shmem_allocations` is revoked from `PUBLIC` and granted to `pg_read_all_stats` ([system_views.sql:652-658](../../../../raw/postgres-17/src/backend/catalog/system_views.sql#L652-L658)); the `pg_buffercache` view and `pg_buffercache_pages()` are granted to `pg_monitor` ([pg_buffercache--1.2--1.3.sql:1-7](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache--1.2--1.3.sql#L1-L7)), as are `pg_buffercache_summary()` and `pg_buffercache_usage_counts()` ([pg_buffercache--1.3--1.4.sql:24-28](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache--1.3--1.4.sql#L24-L28)); and `pg_buffercache_evict()` checks `superuser()` in C, with no grant of its own ([pg_buffercache_pages.c#pg_buffercache_evict](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache_pages.c#L362-L375), [pg_buffercache--1.4--1.5.sql:1-6](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache--1.4--1.5.sql#L1-L6)).

### Decision guide

Reasoning only from the mechanisms filed above.

1. **If the hot working set fits in 256 GiB and the rest of the database is much larger**, the pool is doing what it is for: hits avoid `smgr` entirely, and hot pages hold `usage_count` against the sweep. The costs you accept are the checkpoint scan, the `DROP`/`TRUNCATE` scans, and a ~3 s restart.
2. **If the whole database fits in 256 GiB**, consider whether `debug_io_direct = data` belongs in the plan: the second copy in the OS cache is buying nothing, and the measured 522 ms versus 43.7 ms shows how much the OS cache is doing when it is left in place.
3. **If the workload drops or truncates relations frequently** - partition rotation, `REINDEX`, temp-to-permanent churn, an ETL that recreates tables - price it: 5.9 ns per buffer per pass is 0.2 s per `DROP` at this size, and batching many drops into one transaction was measured 10x cheaper than one transaction each.
4. **If sequential scans of tables under 64 GiB matter**, remember the `NBuffers / 4` line: below it, one scan can evict everything, and two concurrent scans of the same table stop sharing reads because synchronised scanning is off too.
5. **If a monitoring job selects from `pg_buffercache`**, move it to `pg_buffercache_summary()` before the pool grows.
6. **If autoprewarm is preloaded**, decide the interval deliberately; the dump repeats an `NBuffers`-sized allocation and a full header walk.
7. **Before the restart**, reserve huge pages against `shared_memory_size_in_huge_pages`, not against `shared_buffers`, and remember that `huge_pages = on` fails to start rather than falling back.
8. **What source cannot tell you**: whether 256 GiB beats 128 GiB for your queries. Nothing in the tree models that, and the documentation's own guidance stops at 25 % with a caution at 40 %.

### How this was measured, and what the measurements cannot say

- Every number above comes from [the script filed below](#measurement-script), run in one invocation on 2026-09-17 against server version **17.11**, built from pin `786db8dcf168bd9df8f55047337525ac19118b1c` (`REL_17_11-7-g786db8dcf16`).
- Platform: Linux 6.18.33.2-microsoft-standard-WSL2, x86_64, `MemTotal` 32,583,848 kB, `Hugepagesize` 2048 kB, `HugePages_Total` 0, `vm.overcommit_memory` 0. Build: `block_size` 8192, `wal_block_size` 8192, `segment_size` 131072 blocks, `MAXIMUM_ALIGNOF` 8, `data_checksums` off, `max_connections` 100.
- The build passed its own suites in the same run: core regression **All 225 tests passed**, `contrib/pg_buffercache` **All 1 tests passed**. The measurement cluster's log contains **0** `ERROR` or `FATAL` lines outside the deliberate probes, which run against a separate data directory.
- **This host cannot allocate 256 GiB.** Every 256 GiB figure is either computed by the pinned binary itself (`postgres -C`) or an explicitly labelled extrapolation of a measured slope. No timing at 33,554,432 buffers was measured.
- Timings are taken inside the server with `clock_timestamp()` around each statement, so no `psql` start-up cost is included; the `DROP`/`TRUNCATE` numbers include the commit, because that is where the buffer-pool pass happens.
- The cluster ran with one connection at a time and no concurrent load, so every figure is a single-backend, uncontended figure. Contention on the 128 mapping partitions and on `buffer_strategy_lock` is exactly what this setup cannot measure.
- The server was stopped and the sandbox deleted before this page was filed; see the `stop` and `clean` stages.

## Measurement Script

### Usage

| Item | Detail |
|---|---|
| Purpose | produces every measured number on this page: the `-C` sizing sweep and the 256 GiB figures, the `pg_shmem_allocations` per-buffer table and SLRU auto-sizing, server-start and checkpoint scan cost per pool size, `DROP`/`TRUNCATE` full-pool passes, the `NBuffers / 4` threshold, the `VACUUM` ring, `io_combine_limit`, the three `pg_buffercache` readers, and the startup error paths |
| Invocation | `bash shbuf.sh` from the repository root, with the script saved anywhere; it resolves everything from `WIKI_ROOT`, which defaults to `$PWD` |
| Stages | `build check sizing cluster shmem startup ckpt drop strategy vacring iocombine bcache errors summary stop` in that default order, plus `clean` on request. Select stages as arguments: `bash shbuf.sh sizing shmem`. Each stage is idempotent: `build` skips when the binary exists, `cluster` skips `initdb` when the data directory exists, and the timing stages delete their own previous rows before re-measuring |
| Environment | `WIKI_ROOT` (`$PWD`), `SRC` (`$WIKI_ROOT/raw/postgres-17`), `SANDBOX` (`$WIKI_ROOT/.wiki-runtime/tmp/shbuf`), `PORT` (`55418`), `JOBS` (`12`), `BASE_SB` (`1GB`), `POOLS` (`128MB 1GB 4GB 16GB`), `ROWS_BELOW` (`2000000`), `ROWS_ABOVE` (`4000000`), `TRIALS` (`3`), `DROPS` (`30`) |
| Prerequisites | a C toolchain, `make`, `flex`, `bison`, `perl`, readline and zlib headers, and ICU discoverable through `pkg-config` because the tree is configured `--with-icu`; `initdb` runs with `--locale=C --encoding=UTF8`; at least 20 GiB of free RAM for the 16 GiB pool stages and about 8 GiB of disk for the build, install and data directories |
| Output | one file per stage under `$SANDBOX/out/`, and `$SANDBOX/out/summary.txt`, which concatenates all of them plus the test-suite result lines and an `ERROR`/`FATAL` audit of the cluster log. Read `summary.txt` first |
| Runtime | about 13 minutes for a full run on 22 cores, of which roughly 1 minute is the build and 4 minutes `make check`; a re-run from a built tree with `sizing cluster shmem startup ckpt drop strategy vacring iocombine bcache errors summary` is about 8 minutes |
| Cleanup | `bash shbuf.sh clean` stops the server and deletes `$SANDBOX`. The `stop` stage, which runs by default, stops the server and asserts that no `postmaster.pid`, no matching `postgres` process and no listener on `$PORT` is left |

Isolation: the pinned checkout is read only, the build is out of tree, the cluster has its own data directory, socket directory and port `55418`, and every fixture table is disposable. GUC apply scopes are named in the script's own header, and every `psql` call runs `-X -v ON_ERROR_STOP=1` with session-scoped `statement_timeout` and `lock_timeout`.

### The script

```bash
#!/usr/bin/env bash
# Measurements for the wiki page
#   wiki/v17/questions/storage-and-vacuum/very-large-shared-buffers.md
#
# What this measures, and why it is safe to run
# ---------------------------------------------
# Every number that page reports about a running v17 server comes from this
# script.  It builds the pinned PostgreSQL 17 checkout out of tree, then
# answers two kinds of question:
#
#   1. What a 256 GiB (or larger) shared_buffers costs, without allocating it.
#      "postgres -C <runtime-computed GUC>" runs the shared-memory sizing code
#      and prints the answer before any segment is mapped, so a 31 GiB host
#      can report the exact shared-memory total, the huge-page count and the
#      marginal per-buffer overhead of a 256 GiB pool.
#   2. What the pool-size-dependent code paths do at sizes this host can run:
#      shared-memory structure sizes, SLRU auto-sizing, server start time,
#      checkpoint scan time, DROP/TRUNCATE full-pool scans, the NBuffers/4
#      bulk-read threshold, the VACUUM ring, io_combine_limit, the three
#      pg_buffercache readers, and the startup error paths.
#
# Timings are taken inside the server with clock_timestamp() around each
# statement, so no psql start-up cost is counted.  Server start time is read
# from the server's own log timestamps, which carry milliseconds.
#
# The pinned checkout under raw/postgres-17 is read only.  Everything this
# script writes lives under $SANDBOX, and the clean stage deletes it.  The
# cluster it starts has its own data directory, its own socket directory and a
# non-default port; it is never a cluster anyone else named.  Every fixture is
# disposable: the script creates and drops its own tables in its own database.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/shbuf.sh                 # every stage, in order
#   bash .wiki-runtime/tmp/shbuf.sh sizing shmem    # selected stages
#   bash .wiki-runtime/tmp/shbuf.sh clean           # stop, delete the sandbox
#
# Stages, in default order:
#   build check sizing cluster shmem startup ckpt drop strategy vacring
#   iocombine bcache errors summary stop
# and, on request only: clean
#
# Environment: WIKI_ROOT SRC SANDBOX PORT JOBS BASE_SB POOLS ROWS_BELOW
#              ROWS_ABOVE TRIALS DROPS
#
# GUC apply scopes, from the pinned v17 GUC table:
#   shared_buffers, huge_pages, huge_page_size, debug_io_direct,
#   transaction_buffers, commit_timestamp_buffers, subtransaction_buffers
#       -> PGC_POSTMASTER, restart.  This script restarts for every value.
#   bgwriter_delay, bgwriter_lru_maxpages -> PGC_SIGHUP, reload.  Read only.
#   vacuum_buffer_usage_limit, io_combine_limit, track_io_timing,
#   statement_timeout, lock_timeout -> PGC_USERSET, session/transaction scope.
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
SRC="${SRC:-$WIKI_ROOT/raw/postgres-17}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/shbuf}"
PORT="${PORT:-55418}"
JOBS="${JOBS:-12}"
BASE_SB="${BASE_SB:-1GB}"
POOLS="${POOLS:-128MB 1GB 4GB 16GB}"
ROWS_BELOW="${ROWS_BELOW:-2000000}"
ROWS_ABOVE="${ROWS_ABOVE:-4000000}"
TRIALS="${TRIALS:-3}"
DROPS="${DROPS:-30}"

BUILD="$SANDBOX/build17"; INST="$SANDBOX/install17"; DATA="$SANDBOX/data17"
EDATA="$SANDBOX/dataerr"; OUT="$SANDBOX/out"; SOCK="$SANDBOX/sock17"
BIN="$INST/bin"; DB=shbuf
export PGPORT="$PORT" PGHOST="$SOCK" PGDATABASE="$DB" PGUSER=postgres

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# -X ignores ~/.psqlrc so a stray file cannot change a result; ON_ERROR_STOP
# means no failed statement passes silently.  statement_timeout and
# lock_timeout are PGC_USERSET, so passing them through libpq applies them at
# session scope, with no reload and no restart.
SESSION_OPTS="-c statement_timeout=10min -c lock_timeout=60s"
pg()  { PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -v ON_ERROR_STOP=1 "$@"; }
pgq() { pg -At "$@"; }

running() { "$BIN/pg_ctl" -D "$DATA" status >/dev/null 2>&1; }
stop_server() { running && "$BIN/pg_ctl" -D "$DATA" -m fast -w stop >/dev/null 2>&1; return 0; }

# Restart the measurement cluster with one shared_buffers value plus any extra
# "name=value" options.  shared_buffers is PGC_POSTMASTER, so every size
# change below is a restart, not a reload.
start_with() {
  local sb="$1"; shift
  local log="${SERVER_LOG:-$OUT/server.log}"
  local o="-p $PORT -k $SOCK -c shared_buffers=$sb -c listen_addresses='' -c logging_collector=off"
  for extra in "$@"; do o="$o -c $extra"; done
  stop_server
  "$BIN/pg_ctl" -D "$DATA" -l "$log" -w -o "$o" start >/dev/null 2>&1 \
    || die "server did not start with shared_buffers=$sb (see $log)"
}

nbuffers() { pgq -c "SELECT /* wiki_shbuf_nbuffers */ setting::bigint FROM pg_settings WHERE name = 'shared_buffers';"; }

# The server-side stopwatch.  timeit() runs p_stmt p_n times and records the
# elapsed microseconds of each run; with p_fmt it substitutes the iteration
# number into the statement, which is how the DROP loop drops a different
# table each time.  Recording the pool size with every row is what makes the
# per-pool-size comparison possible.
install_timer() {
  pg -q <<'SQL' || die "could not install the timer"
CREATE /* wiki_shbuf_fixture */ TABLE IF NOT EXISTS meas
  (stage text, pool bigint, label text, i int, us bigint);
CREATE OR REPLACE /* wiki_shbuf_fixture */ FUNCTION timeit
  (p_stage text, p_label text, p_stmt text, p_n int, p_fmt boolean DEFAULT false)
RETURNS void LANGUAGE plpgsql AS $fn$
DECLARE
  i  int;
  t0 timestamptz;
  t1 timestamptz;
  s  text;
  nb bigint := (SELECT setting::bigint FROM pg_settings WHERE name = 'shared_buffers');
BEGIN
  FOR i IN 1..p_n LOOP
    s := CASE WHEN p_fmt THEN format(p_stmt, i) ELSE p_stmt END;
    t0 := clock_timestamp();
    EXECUTE s;
    t1 := clock_timestamp();
    INSERT INTO meas
      VALUES (p_stage, nb, p_label, i,
              (extract(epoch FROM (t1 - t0)) * 1000000)::bigint);
  END LOOP;
END
$fn$;
-- timeit() cannot time DROP or TRUNCATE: the buffer-pool work for a dropped
-- or replaced relfilenode is queued as a pending delete and performed by
-- smgrDoPendingDeletes() at commit, which is after timeit()'s stopwatch
-- stops.  timeit_tx() is a procedure, so it can COMMIT inside the loop and
-- put the commit inside the measured interval.
CREATE OR REPLACE /* wiki_shbuf_fixture */ PROCEDURE timeit_tx
  (p_stage text, p_label text, p_stmt text, p_n int, p_fmt boolean DEFAULT false)
LANGUAGE plpgsql AS $pr$
DECLARE
  i  int;
  t0 timestamptz;
  t1 timestamptz;
  s  text;
  nb bigint := (SELECT setting::bigint FROM pg_settings WHERE name = 'shared_buffers');
BEGIN
  FOR i IN 1..p_n LOOP
    s := CASE WHEN p_fmt THEN format(p_stmt, i) ELSE p_stmt END;
    t0 := clock_timestamp();
    EXECUTE s;
    COMMIT;
    t1 := clock_timestamp();
    INSERT INTO meas
      VALUES (p_stage, nb, p_label, i,
              (extract(epoch FROM (t1 - t0)) * 1000000)::bigint);
    COMMIT;
  END LOOP;
END
$pr$;
SQL
}

report_meas() {   # report_meas <stage> <outfile>
  pg -c "SELECT /* wiki_shbuf_report */ pool AS pool_buffers, label,
                count(*) AS n, min(us) AS min_us,
                round(avg(us))::bigint AS avg_us,
                (percentile_cont(0.5) WITHIN GROUP (ORDER BY us))::bigint AS median_us,
                max(us) AS max_us
         FROM meas WHERE stage = '$1'
         GROUP BY pool, label ORDER BY label, pool;" >> "$2" 2>&1
}

# below/above are the two seq-scan fixtures: one smaller and one larger than
# NBuffers/4 at BASE_SB.  Disposable, like every table here.
ensure_fixtures() {
  local n
  n=$(pgq -c "SELECT /* wiki_shbuf_fixture */ count(*) FROM pg_class WHERE relname IN ('below','above');")
  [ "$n" = 2 ] && return 0
  # Each VACUUM gets its own -c: several statements inside one -c travel as a
  # single query, which puts them in an implicit transaction block, and VACUUM
  # cannot run there.
  pg -q -c "DROP /* wiki_shbuf_fixture */ TABLE IF EXISTS below, above;" \
        -c "CREATE /* wiki_shbuf_fixture */ TABLE below (id int, pad text);
            INSERT /* wiki_shbuf_fixture */ INTO below
              SELECT g, repeat('x', 40) FROM generate_series(1, $ROWS_BELOW) g;
            CREATE /* wiki_shbuf_fixture */ TABLE above (id int, pad text);
            INSERT /* wiki_shbuf_fixture */ INTO above
              SELECT g, repeat('x', 40) FROM generate_series(1, $ROWS_ABOVE) g;" \
        -c "VACUUM /* wiki_shbuf_fixture */ (ANALYZE) below, above;" >/dev/null \
    || die "could not build the below/above fixtures"
}

# --------------------------------------------------------------------------
stage_build() {
  say "build: configure the pinned checkout out of tree, install it and two contrib modules"
  mkdir -p "$BUILD" "$OUT" "$SOCK"
  if [ -x "$BIN/postgres" ]; then note "already built, skipping"; return 0; fi
  [ -x "$SRC/configure" ] || die "no pinned checkout at $SRC"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" --enable-debug \
      --with-icu --with-readline --with-zlib > configure.log 2>&1 ) \
    || { cp "$BUILD/configure.log" "$OUT/" 2>/dev/null; die "configure failed, see $OUT/configure.log"; }
  ( cd "$BUILD" && make -s -j "$JOBS" > make.log 2>&1 \
      && make -s install > install.log 2>&1 \
      && make -s -C contrib/pg_buffercache install >> install.log 2>&1 \
      && make -s -C contrib/pg_prewarm install >> install.log 2>&1 ) \
    || { cp "$BUILD"/{make,install}.log "$OUT/" 2>/dev/null; die "make failed, see $OUT/make.log"; }
  cp "$BUILD"/{configure,make,install}.log "$OUT/" 2>/dev/null
  "$BIN/postgres" --version > "$OUT/version.txt" 2>&1
  note "$(cat "$OUT/version.txt")"
}

stage_check() {
  say "check: the build's own test suites"
  : > "$OUT/check_summary.txt"
  ( cd "$BUILD" && make -s check > "$OUT/check.log" 2>&1 )
  grep -E 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests failed' "$OUT/check.log" | tail -1 \
    | { read -r l; printf 'core regression: %s\n' "${l:-no result line}"; } >> "$OUT/check_summary.txt"
  ( cd "$BUILD" && make -s -C contrib/pg_buffercache check > "$OUT/check_bcache.log" 2>&1 )
  grep -E 'All [0-9]+ tests passed|[0-9]+ of [0-9]+ tests failed' "$OUT/check_bcache.log" | tail -1 \
    | { read -r l; printf 'pg_buffercache:   %s\n' "${l:-no result line}"; } >> "$OUT/check_summary.txt"
  cat "$OUT/check_summary.txt" >&2
}

# --------------------------------------------------------------------------
# No server runs in this stage.  "postgres -C" on a runtime-computed GUC
# reaches the shared-memory sizing code and exits before the segment is
# mapped, so these are real answers for pools this host cannot allocate.
stage_sizing() {
  say "sizing: shared_memory_size for pools up to 4 TiB, with nothing allocated"
  mkdir -p "$OUT"
  [ -f "$EDATA/PG_VERSION" ] || "$BIN/initdb" -D "$EDATA" -U postgres --locale=C --encoding=UTF8 \
    > "$OUT/initdb_err.log" 2>&1 || die "initdb for the probe directory failed"
  local sb nb smb hp
  : > "$OUT/sizing.txt"
  printf '%-10s %12s %12s %14s %14s\n' setting blocks shmem_mb huge_2mb blocks_bytes >> "$OUT/sizing.txt"
  for sb in 128MB 1GB 4GB 8GB 16GB 32GB 64GB 128GB 192GB 256GB 512GB 1TB 2TB 4TB; do
    nb=$("$BIN/postgres"  -D "$EDATA" -c shared_buffers="$sb" -C shared_buffers 2>/dev/null | tail -1)
    smb=$("$BIN/postgres" -D "$EDATA" -c shared_buffers="$sb" -C shared_memory_size 2>/dev/null | tail -1)
    hp=$("$BIN/postgres"  -D "$EDATA" -c shared_buffers="$sb" -C shared_memory_size_in_huge_pages 2>/dev/null | tail -1)
    printf '%-10s %12s %12s %14s %14s\n' "$sb" "$nb" "$smb" "$hp" "$(( nb * 8192 ))" >> "$OUT/sizing.txt"
  done

  # Marginal shared memory per buffer, from pairs of block counts.  The pair
  # form cancels every fixed cost, and a 1,000,000-block step keeps the 1 MB
  # rounding of shared_memory_size under 0.02 % of the difference.
  # 10,000,000 blocks is MAX_CHECKPOINT_REQUESTS: at and above it the
  # checkpointer request queue stops growing, so the marginal cost falls.
  local pair lo hi lomb himb
  printf '\nmarginal bytes of shared memory per buffer, from 1,000,000-block pairs\n' >> "$OUT/sizing.txt"
  printf '%-24s %12s %12s %16s %s\n' pair lo_mb hi_mb bytes_per_buffer note >> "$OUT/sizing.txt"
  for pair in 1000000:2000000:'crosses 2^20 buckets' \
              8000000:9000000:'crosses 2^23 buckets' \
              9000000:10000000:'queue still growing' \
              10000000:11000000:'queue capped at 10,000,000' \
              11000000:12000000:'queue capped' \
              20000000:21000000:'no boundary crossed' \
              32000000:33000000:'no boundary crossed' \
              33000000:34000000:'crosses 2^25 buckets'; do
    lo=${pair%%:*}; hi=$(printf '%s' "$pair" | cut -d: -f2)
    lomb=$("$BIN/postgres" -D "$EDATA" -c shared_buffers="$lo" -C shared_memory_size 2>/dev/null | tail -1)
    himb=$("$BIN/postgres" -D "$EDATA" -c shared_buffers="$hi" -C shared_memory_size 2>/dev/null | tail -1)
    printf '%-24s %12s %12s %16s %s\n' "$lo->$hi" "$lomb" "$himb" \
      "$(( ( (himb - lomb) * 1048576 ) / (hi - lo) ))" "${pair##*:}" >> "$OUT/sizing.txt"
  done

  # The buffer mapping table is sized NBuffers + NUM_BUFFER_PARTITIONS and its
  # bucket array is a power of two, so one buffer past a power of two doubles
  # the bucket array.  256 GiB is 33,554,432 blocks, which is 2^25 exactly, so
  # the default-block-size 256 GiB pool lands on the wrong side of the step by
  # NUM_BUFFER_PARTITIONS buffers.
  local b
  printf '\nthe buffer mapping table step around 2^25 blocks\n' >> "$OUT/sizing.txt"
  printf '%-14s %12s %14s %s\n' blocks shmem_mb delta_mb note >> "$OUT/sizing.txt"
  local prev=0
  for b in 33554176 33554304 33554305 33554432 33554433; do
    lomb=$("$BIN/postgres" -D "$EDATA" -c shared_buffers="$b" -C shared_memory_size 2>/dev/null | tail -1)
    printf '%-14s %12s %14s %s\n' "$b" "$lomb" "$(( prev == 0 ? 0 : lomb - prev ))" \
      "$( [ "$b" = 33554304 ] && printf '2^25 - 128, last size with 2^25 buckets'; \
          [ "$b" = 33554305 ] && printf 'first size needing 2^26 buckets'; \
          [ "$b" = 33554432 ] && printf '256 GiB exactly'; )" >> "$OUT/sizing.txt"
    prev=$lomb
  done

  # 33,554,432 blocks is exactly 256 GiB at the default 8 kB block size.
  printf '\n256 GiB, stated exactly\n' >> "$OUT/sizing.txt"
  {
    printf 'blocks                     %s\n' \
      "$("$BIN/postgres" -D "$EDATA" -c shared_buffers=256GB -C shared_buffers 2>/dev/null | tail -1)"
    printf 'shared_memory_size         %s MB\n' \
      "$("$BIN/postgres" -D "$EDATA" -c shared_buffers=256GB -C shared_memory_size 2>/dev/null | tail -1)"
    printf 'huge pages, 2 MB default   %s\n' \
      "$("$BIN/postgres" -D "$EDATA" -c shared_buffers=256GB -C shared_memory_size_in_huge_pages 2>/dev/null | tail -1)"
    printf 'huge pages, huge_page_size=1GB %s\n' \
      "$("$BIN/postgres" -D "$EDATA" -c shared_buffers=256GB -c huge_page_size=1GB \
           -C shared_memory_size_in_huge_pages 2>/dev/null | tail -1)"
  } >> "$OUT/sizing.txt"
  cat "$OUT/sizing.txt" >&2
}

# --------------------------------------------------------------------------
stage_cluster() {
  say "cluster: initdb, start at shared_buffers=$BASE_SB, install contrib and the timer"
  mkdir -p "$OUT" "$SOCK"
  [ -f "$DATA/PG_VERSION" ] || "$BIN/initdb" -D "$DATA" -U postgres --locale=C --encoding=UTF8 \
    > "$OUT/initdb.log" 2>&1 || die "initdb failed, see $OUT/initdb.log"
  start_with "$BASE_SB"
  if [ "$(PGDATABASE=postgres pgq -c "SELECT /* wiki_shbuf_fixture */ count(*) FROM pg_database WHERE datname = '$DB';")" = 0 ]; then
    PGDATABASE=postgres pg -q -c "CREATE /* wiki_shbuf_fixture */ DATABASE $DB;" || die "CREATE DATABASE failed"
  fi
  pg -q -c "CREATE /* wiki_shbuf_fixture */ EXTENSION IF NOT EXISTS pg_buffercache;
            CREATE /* wiki_shbuf_fixture */ EXTENSION IF NOT EXISTS pg_prewarm;" || die "extensions failed"
  install_timer
  ensure_fixtures
  {
    pgq -c "SELECT /* wiki_shbuf_facts */ version();"
    pgq -c "SELECT /* wiki_shbuf_facts */ name || ' = ' || setting || coalesce(' ' || unit, '')
            FROM pg_settings WHERE name IN
              ('block_size','wal_block_size','segment_size','data_checksums','shared_buffers','huge_pages',
               'huge_pages_status','huge_page_size','io_combine_limit','vacuum_buffer_usage_limit',
               'bgwriter_delay','bgwriter_lru_maxpages','bgwriter_lru_multiplier','checkpoint_timeout',
               'checkpoint_completion_target','max_wal_size','max_connections','effective_cache_size',
               'wal_level','track_io_timing','debug_io_direct','shared_memory_size',
               'shared_memory_size_in_huge_pages','temp_buffers','wal_buffers')
            ORDER BY name;"
    printf 'uname                  %s\n' "$(uname -srm)"
    printf 'MemTotal               %s\n' "$(grep -F MemTotal /proc/meminfo)"
    printf 'Hugepagesize           %s\n' "$(grep -F Hugepagesize /proc/meminfo)"
    printf 'HugePages_Total        %s\n' "$(grep -F HugePages_Total /proc/meminfo)"
    printf 'overcommit_memory      %s\n' "$(cat /proc/sys/vm/overcommit_memory)"
    printf 'pinned commit          %s\n' "$(cd "$SRC" && git log -1 --format=%H)"
    printf 'pinned describe        %s\n' "$(cd "$SRC" && git describe --tags)"
    printf 'MAX_DATA_ALIGNMENT     %s\n' "$(grep -F 'define MAXIMUM_ALIGNOF' "$BUILD/src/include/pg_config.h" 2>/dev/null)"
  } > "$OUT/facts.txt" 2>&1
  cat "$OUT/facts.txt" >&2
}

# --------------------------------------------------------------------------
# Every pool-size-dependent shared structure, read from the server itself.
stage_shmem() {
  say "shmem: pg_shmem_allocations and SLRU auto-sizing, per pool size"
  local sb nb
  : > "$OUT/shmem.txt"
  for sb in $POOLS; do
    start_with "$sb"; nb=$(nbuffers)
    { printf '\n--- shared_buffers = %s (%s buffers)\n' "$sb" "$nb"
      pg -c "SELECT /* wiki_shbuf_shmem */ name, size,
                    round(size::numeric / $nb, 3) AS bytes_per_buffer
             FROM pg_shmem_allocations
             WHERE name IN ('Buffer Blocks','Buffer Descriptors','Buffer IO Condition Variables',
                            'Checkpoint BufferIds','Shared Buffer Lookup Table','Buffer Strategy Status',
                            'Checkpointer Data','transaction','commit_timestamp','subtransaction')
             ORDER BY size DESC;"
      pg -c "SELECT /* wiki_shbuf_shmem_total */ coalesce(name, '<anonymous>') AS name, size,
                    round(size::numeric / $nb, 3) AS bytes_per_buffer
             FROM pg_shmem_allocations WHERE name IS NULL
             UNION ALL
             SELECT 'sum of all rows', sum(size), round(sum(size)::numeric / $nb, 3)
             FROM pg_shmem_allocations;"
      pg -c "SELECT /* wiki_shbuf_slru */ name, setting AS blocks, setting::bigint * 8192 AS bytes
             FROM pg_settings
             WHERE name IN ('transaction_buffers','commit_timestamp_buffers','subtransaction_buffers',
                            'multixact_offset_buffers','multixact_member_buffers','notify_buffers',
                            'serializable_buffers','wal_buffers')
             ORDER BY name;"
    } >> "$OUT/shmem.txt" 2>&1
  done
  cat "$OUT/shmem.txt" >&2
}

# --------------------------------------------------------------------------
# InitBufferPool() initializes every buffer header serially at startup, inside
# CreateSharedMemoryAndSemaphores().  The postmaster logs "starting
# PostgreSQL" *after* that call, so the interval that contains the pool's
# initialization is the one from the shell's pre-start clock reading to that
# first log line; it also contains exec, configuration load and the whole
# shared-memory mapping.  The second interval, from that line to "ready to
# accept connections", is startup-process work and contains no pool
# initialization at all.  Both are reported.
stage_startup() {
  say "startup: shared-memory creation and post-creation startup, per pool size"
  local sb nb i d t tz t0 first ready pre post
  : > "$OUT/startup.txt"
  printf '%-10s %12s %16s %16s\n' shared_buffers blocks to_starting_ms starting_to_ready_ms >> "$OUT/startup.txt"
  for sb in $POOLS; do
    pre=999999; post=999999
    for i in $(seq 1 "$TRIALS"); do
      stop_server
      SERVER_LOG="$OUT/start_$sb.log"; : > "$SERVER_LOG"
      t0=$(date +%s%3N)
      start_with "$sb"
      read -r d t tz _ < <(grep -F 'starting PostgreSQL' "$SERVER_LOG" | head -1)
      first=$(date -d "$d $t $tz" +%s%3N 2>/dev/null)
      read -r d t tz _ < <(grep -F 'ready to accept connections' "$SERVER_LOG" | head -1)
      ready=$(date -d "$d $t $tz" +%s%3N 2>/dev/null)
      if [ -n "$first" ] && [ -n "$ready" ]; then
        [ $(( first - t0 )) -lt "$pre" ] && pre=$(( first - t0 ))
        [ $(( ready - first )) -lt "$post" ] && post=$(( ready - first ))
      fi
    done
    unset SERVER_LOG
    nb=$(nbuffers)
    printf '%-10s %12s %16s %16s\n' "$sb" "$nb" "$pre" "$post" >> "$OUT/startup.txt"
  done
  cat "$OUT/startup.txt" >&2
}

# --------------------------------------------------------------------------
# BufferSync() locks and examines every buffer header before writing
# anything, so repeated checkpoints over an idle, clean pool price that scan
# on its own.
stage_ckpt() {
  say "ckpt: CHECKPOINT over an idle, clean pool, per pool size"
  local sb
  : > "$OUT/ckpt.txt"
  for sb in $POOLS; do
    start_with "$sb"; install_timer
    pg -q -c "DELETE /* wiki_shbuf_reset */ FROM meas WHERE stage = 'ckpt' AND pool =
                (SELECT setting::bigint FROM pg_settings WHERE name = 'shared_buffers');" >/dev/null
    pg -q -c "CHECKPOINT /* wiki_shbuf_ckpt_warm */;" >/dev/null
    pg -q -c "SELECT /* wiki_shbuf_ckpt */ timeit('ckpt', 'CHECKPOINT', 'CHECKPOINT', 6);" >/dev/null \
      || die "checkpoint timing failed at $sb"
  done
  report_meas ckpt "$OUT/ckpt.txt"
  cat "$OUT/ckpt.txt" >&2
}

# --------------------------------------------------------------------------
# DROP and TRUNCATE reach DropRelationsAllBuffers() and
# DropRelationBuffers().  Outside recovery smgrnblocks_cached() returns
# InvalidBlockNumber, so the BUF_DROP_FULL_SCAN_THRESHOLD shortcut cannot be
# taken and each call scans the whole pool; TRUNCATE runs one pass per fork.
stage_drop() {
  say "drop: DROP and TRUNCATE of tiny relations, per pool size"
  local sb i
  : > "$OUT/drop.txt"
  for sb in $POOLS; do
    start_with "$sb"; install_timer
    pg -q -c "DELETE /* wiki_shbuf_reset */ FROM meas WHERE stage = 'drop' AND pool =
                (SELECT setting::bigint FROM pg_settings WHERE name = 'shared_buffers');" >/dev/null

    # $DROPS disposable one-block tables, dropped one transaction at a time,
    # and then $DROPS more dropped in a single transaction.  The serial form
    # runs one full-pool pass per drop; the batched form runs one pass for the
    # whole set, because smgrdounlinkall() hands every pending delete to
    # DropRelationsAllBuffers() together.
    for i in $(seq 1 "$DROPS"); do
      pg -q -c "CREATE /* wiki_shbuf_fixture */ TABLE d$i (id int);
                INSERT /* wiki_shbuf_fixture */ INTO d$i VALUES (1);" >/dev/null
    done
    pg -q -c "CALL /* wiki_shbuf_drop */ timeit_tx('drop', 'DROP TABLE, 1 block, own transaction',
                'DROP TABLE d%s', $DROPS, true);" >/dev/null || die "drop timing failed at $sb"
    local list=""
    for i in $(seq 1 "$DROPS"); do
      pg -q -c "CREATE /* wiki_shbuf_fixture */ TABLE b$i (id int);
                INSERT /* wiki_shbuf_fixture */ INTO b$i VALUES (1);" >/dev/null
      list="$list${list:+, }b$i"
    done
    pg -q -c "CALL /* wiki_shbuf_drop_batch */ timeit_tx('drop', 'DROP TABLE, $DROPS tables, one transaction',
                'DROP TABLE $list', 1);" >/dev/null || die "batched drop timing failed at $sb"

    # TRUNCATE of a one-block table that has neither a visibility map nor a
    # free space map, then of one that has both, whose forks smgrtruncate()
    # drops in separate passes.
    pg -q -c "DROP /* wiki_shbuf_fixture */ TABLE IF EXISTS t1, t3;" >/dev/null
    pg -q -c "CREATE /* wiki_shbuf_fixture */ TABLE t1 (id int);
              CREATE /* wiki_shbuf_fixture */ TABLE t3 (id int);" >/dev/null
    for i in $(seq 1 "$DROPS"); do
      pg -q -c "INSERT /* wiki_shbuf_fixture */ INTO t1 VALUES (1);" \
            -c "CALL /* wiki_shbuf_trunc */ timeit_tx('drop', 'TRUNCATE, main fork only', 'TRUNCATE t1', 1);" >/dev/null
      pg -q -c "INSERT /* wiki_shbuf_fixture */ INTO t3 SELECT g FROM generate_series(1, 50000) g;" \
            -c "VACUUM /* wiki_shbuf_fixture */ t3;" \
            -c "CALL /* wiki_shbuf_trunc3 */ timeit_tx('drop', 'TRUNCATE, main + vm + fsm', 'TRUNCATE t3', 1);" >/dev/null
    done
    pg -q -c "DROP /* wiki_shbuf_fixture */ TABLE t1, t3;" >/dev/null
  done
  report_meas drop "$OUT/drop.txt"
  cat "$OUT/drop.txt" >&2
}

# --------------------------------------------------------------------------
# initscan() asks for BAS_BULKREAD and synchronized scanning only when the
# relation is larger than NBuffers/4.  Below that line a seq scan uses the
# default strategy and one table can fill the pool.
stage_strategy() {
  say "strategy: the NBuffers/4 bulk-read threshold, read from pg_buffercache"
  start_with "$BASE_SB"; install_timer; ensure_fixtures
  local nb thr t
  nb=$(nbuffers); thr=$(( nb / 4 ))
  : > "$OUT/strategy.txt"
  printf 'pool_blocks %s   NBuffers/4 %s blocks (%s MiB)\n' "$nb" "$thr" "$(( thr * 8192 / 1048576 ))" >> "$OUT/strategy.txt"
  pg -c "SELECT /* wiki_shbuf_relsize */ relname, pg_relation_size(oid) / 8192 AS blocks,
                pg_size_pretty(pg_relation_size(oid)) AS size,
                (pg_relation_size(oid) / 8192) > $thr AS above_threshold
         FROM pg_class WHERE relname IN ('below','above') ORDER BY relname;" >> "$OUT/strategy.txt" 2>&1
  for t in below above; do
    start_with "$BASE_SB"                       # a restart empties the pool
    pg -q -c "SELECT /* wiki_shbuf_reset_io */ pg_stat_reset_shared('io');" >/dev/null
    # max_parallel_workers_per_gather is PGC_USERSET.  Zero keeps the whole
    # scan in one backend, so pg_stat_io's client-backend rows account for all
    # of it instead of leaving part under 'parallel worker'.
    pg -q -c "SET /* wiki_shbuf_noparallel */ max_parallel_workers_per_gather = 0;
              SELECT /* wiki_shbuf_seqscan */ count(*) FROM $t;" >/dev/null
    { printf '\n--- after one seq scan of %s\n' "$t"
      pg -c "SELECT /* wiki_shbuf_cached */ '$t' AS rel,
                    count(*) FILTER (WHERE relfilenode = pg_relation_filenode('$t'::regclass)) AS blocks_of_rel,
                    count(*) FILTER (WHERE relfilenode IS NOT NULL) AS blocks_used,
                    count(*) AS pool_blocks
             FROM pg_buffercache;"
      pg -c "SELECT /* wiki_shbuf_io_ctx */ context, reads, hits, evictions, reuses
             FROM pg_stat_io
             WHERE backend_type = 'client backend' AND object = 'relation' AND (reads > 0 OR hits > 0)
             ORDER BY context;"
    } >> "$OUT/strategy.txt" 2>&1
  done
  cat "$OUT/strategy.txt" >&2
}

# --------------------------------------------------------------------------
# vacuum_buffer_usage_limit is PGC_USERSET and BUFFER_USAGE_LIMIT overrides it
# per command; 0 means "use as much of the pool as you like", and whatever the
# setting says the ring is capped at NBuffers/8.
stage_vacring() {
  say "vacring: VACUUM's buffer ring, from 128 kB to unlimited"
  local lim a b
  : > "$OUT/vacring.txt"
  printf '%-14s %14s %14s %14s\n' buffer_usage_limit blocks_of_rel blocks_used pool_blocks >> "$OUT/vacring.txt"
  # Its own fixture, so the stage neither depends on nor disturbs the
  # below/above pair the strategy stage measures.
  start_with "$BASE_SB"
  pg -q -c "DROP /* wiki_shbuf_fixture */ TABLE IF EXISTS vacrel;" >/dev/null
  pg -q -c "CREATE /* wiki_shbuf_fixture */ TABLE vacrel (id int, pad text);
            INSERT /* wiki_shbuf_fixture */ INTO vacrel
              SELECT g, repeat('x', 40) FROM generate_series(1, $ROWS_ABOVE) g;" >/dev/null
  pg -q -c "VACUUM /* wiki_shbuf_fixture */ (ANALYZE) vacrel;" >/dev/null
  printf 'vacrel is %s blocks; the ring cap is NBuffers/8 = %s blocks\n' \
    "$(pgq -c "SELECT /* wiki_shbuf_relsize */ pg_relation_size('vacrel'::regclass) / 8192;")" \
    "$(( $(nbuffers) / 8 ))" >> "$OUT/vacring.txt"
  for lim in '128kB' '2MB' '256MB' '0'; do
    start_with "$BASE_SB"
    pg -q -c "UPDATE /* wiki_shbuf_fixture */ vacrel SET pad = pad WHERE id % 10 = 0;" >/dev/null
    start_with "$BASE_SB"                       # a restart empties the pool again
    pg -q -c "VACUUM /* wiki_shbuf_vacring */ (BUFFER_USAGE_LIMIT '$lim') vacrel;" >/dev/null \
      || die "VACUUM (BUFFER_USAGE_LIMIT '$lim') failed"
    a=$(pgq -c "SELECT /* wiki_shbuf_cached */ count(*) FROM pg_buffercache
                WHERE relfilenode = pg_relation_filenode('vacrel'::regclass);")
    b=$(pgq -c "SELECT /* wiki_shbuf_cached */ count(*) FROM pg_buffercache WHERE relfilenode IS NOT NULL;")
    printf '%-14s %14s %14s %14s\n' "$lim" "$a" "$b" "$(nbuffers)" >> "$OUT/vacring.txt"
  done
  cat "$OUT/vacring.txt" >&2
}

# --------------------------------------------------------------------------
# io_combine_limit is PGC_USERSET and caps how many blocks one read-stream
# read may cover.  pg_stat_io's reads column counts *blocks*, not read calls:
# WaitReadBuffers() passes io_buffers_len, the number of blocks the vectored
# read covered, to pgstat_count_io_op_time().  So the same scan reports the
# same read count at every setting, and the only visible effect is elapsed
# time.  Both are reported, because the unchanged count is itself the finding.
stage_iocombine() {
  say "iocombine: io_combine_limit against scan time and pg_stat_io counts"
  local lim row blocks
  start_with "$BASE_SB" 'track_io_timing=on'; install_timer
  pg -q -c "DROP /* wiki_shbuf_fixture */ TABLE IF EXISTS combine;" >/dev/null
  pg -q -c "CREATE /* wiki_shbuf_fixture */ TABLE combine (id int, pad text);
            INSERT /* wiki_shbuf_fixture */ INTO combine
              SELECT g, repeat('x', 40) FROM generate_series(1, $ROWS_ABOVE) g;" >/dev/null
  pg -q -c "VACUUM /* wiki_shbuf_fixture */ (ANALYZE) combine;" >/dev/null
  blocks=$(pgq -c "SELECT /* wiki_shbuf_relsize */ pg_relation_size('combine'::regclass) / 8192;")
  pg -q -c "DELETE /* wiki_shbuf_reset */ FROM meas WHERE stage = 'iocombine';" >/dev/null
  : > "$OUT/iocombine.txt"
  printf 'combine is %s blocks; NBuffers/4 is %s blocks, so the scan is a bulk read\n' \
    "$blocks" "$(( $(nbuffers) / 4 ))" >> "$OUT/iocombine.txt"
  printf '%-18s %12s %12s %14s\n' io_combine_limit reads hits read_time_ms >> "$OUT/iocombine.txt"
  for lim in '8kB' '32kB' '128kB' '256kB'; do
    start_with "$BASE_SB" 'track_io_timing=on'   # empty pool, warm OS cache
    pg -q -c "SELECT /* wiki_shbuf_reset_io */ pg_stat_reset_shared('io');" >/dev/null
    pg -q -c "SET /* wiki_shbuf_iocombine */ io_combine_limit = '$lim';
              SET /* wiki_shbuf_noparallel */ max_parallel_workers_per_gather = 0;
              SELECT /* wiki_shbuf_seqscan */ timeit('iocombine', 'seq scan at io_combine_limit $lim',
                'SELECT count(*) FROM combine', 1);" >/dev/null \
      || die "scan with io_combine_limit $lim failed"
    row=$(pgq -F' ' -c "SELECT /* wiki_shbuf_io_reads */ coalesce(sum(reads), 0), coalesce(sum(hits), 0),
                               round(coalesce(sum(read_time), 0)::numeric, 1)
                        FROM pg_stat_io WHERE object = 'relation';")
    printf '%-18s %12s %12s %14s\n' "$lim" $row >> "$OUT/iocombine.txt"
  done
  report_meas iocombine "$OUT/iocombine.txt"
  cat "$OUT/iocombine.txt" >&2
}

# --------------------------------------------------------------------------
# pg_buffercache's row-per-buffer reader collects NBuffers records before it
# returns a row; the two summary readers added in 16 walk the same headers
# without allocating.
stage_bcache() {
  say "bcache: the three pg_buffercache readers at a 16 GiB pool"
  start_with 16GB; install_timer
  : > "$OUT/bcache.txt"
  printf 'pool_blocks %s\n' "$(nbuffers)" >> "$OUT/bcache.txt"
  pg -q -c "DELETE /* wiki_shbuf_reset */ FROM meas WHERE stage = 'bcache';" >/dev/null
  pg -q -c "SELECT /* wiki_shbuf_bcache */ timeit('bcache', 'pg_buffercache rows',
              'SELECT count(*) FROM pg_buffercache', 3);
            SELECT /* wiki_shbuf_bcache */ timeit('bcache', 'pg_buffercache_summary()',
              'SELECT * FROM pg_buffercache_summary()', 3);
            SELECT /* wiki_shbuf_bcache */ timeit('bcache', 'pg_buffercache_usage_counts()',
              'SELECT * FROM pg_buffercache_usage_counts()', 3);" >/dev/null || die "bcache timing failed"
  report_meas bcache "$OUT/bcache.txt"
  { printf '\n'; pg -c "SELECT /* wiki_shbuf_bcache */ * FROM pg_buffercache_summary();"; } >> "$OUT/bcache.txt" 2>&1
  cat "$OUT/bcache.txt" >&2
}

# --------------------------------------------------------------------------
# The edges: the GUC range, and what a pool larger than the machine does.
# These probes use their own data directory, so the measurement cluster is
# never touched.
stage_errors() {
  say "errors: range checks, an over-sized pool, huge pages, direct I/O"
  mkdir -p "$OUT"
  [ -f "$EDATA/PG_VERSION" ] || "$BIN/initdb" -D "$EDATA" -U postgres --locale=C --encoding=UTF8 \
    > "$OUT/initdb_err.log" 2>&1
  : > "$OUT/errors.txt"
  local probe_n
  probe() {
    printf '\n--- %s\n' "$1" >> "$OUT/errors.txt"; shift
    timeout 180 "$BIN/postgres" -D "$EDATA" -c "port=$(( PORT + 1 ))" \
      -c "unix_socket_directories=$SOCK" -c 'listen_addresses=' "$@" 2>&1 \
      | head -4 >> "$OUT/errors.txt"
  }
  for probe_n in 15:'one block below the floor' 1073741824:'one block above the ceiling' \
                 1073741823:'the ceiling itself' 16:'the floor itself'; do
    printf '\n--- shared_buffers = %s blocks, %s\n' "${probe_n%%:*}" "${probe_n##*:}" >> "$OUT/errors.txt"
    "$BIN/postgres" -D "$EDATA" -c shared_buffers="${probe_n%%:*}" -C shared_memory_size 2>&1 \
      | tail -1 >> "$OUT/errors.txt"
  done
  probe "shared_buffers = 200GB on this 31 GiB host" -c shared_buffers=200GB
  probe "shared_buffers = 1GB, huge_pages = on, no huge pages configured" -c shared_buffers=1GB -c huge_pages=on
  # debug_io_direct is PGC_POSTMASTER and a developer option: it takes the
  # operating system cache out of the data path, leaving the pool as the only
  # cache for relation data.
  start_with 1GB 'debug_io_direct=data' 'track_io_timing=on'
  { printf '\n--- debug_io_direct = data, one seq scan of above\n'
    pg -c "SELECT /* wiki_shbuf_directio */ current_setting('debug_io_direct') AS debug_io_direct,
                  current_setting('huge_pages_status') AS huge_pages_status,
                  current_setting('io_combine_limit') AS io_combine_limit;"
    pg -q -c "SELECT /* wiki_shbuf_reset_io */ pg_stat_reset_shared('io');" >/dev/null
    pg -c "SET /* wiki_shbuf_noparallel */ max_parallel_workers_per_gather = 0;
           SELECT /* wiki_shbuf_seqscan */ count(*) FROM above;"
    pg -c "SELECT /* wiki_shbuf_io_ctx */ context, reads, hits, round(read_time::numeric, 1) AS read_time_ms
           FROM pg_stat_io
           WHERE backend_type = 'client backend' AND object = 'relation' AND reads > 0;"
  } >> "$OUT/errors.txt" 2>&1
  start_with "$BASE_SB"
  cat "$OUT/errors.txt" >&2
}

# --------------------------------------------------------------------------
stage_summary() {
  say "summary: every result file in one place"
  local f
  { printf '===== facts\n'; cat "$OUT/facts.txt" 2>/dev/null
    for f in sizing shmem startup ckpt drop strategy vacring iocombine bcache errors; do
      printf '\n===== %s\n' "$f"; cat "$OUT/$f.txt" 2>/dev/null
    done
    printf '\n===== test suites\n'; cat "$OUT/check_summary.txt" 2>/dev/null
    printf '\n===== ERROR/FATAL audit of the measurement cluster log\n'
    printf 'matching lines: %s\n' "$(grep -cE 'ERROR|FATAL' "$OUT/server.log" 2>/dev/null)"
    grep -E 'ERROR|FATAL' "$OUT/server.log" 2>/dev/null | tail -10
  } > "$OUT/summary.txt" 2>&1
  cat "$OUT/summary.txt" >&2
  note "results in $OUT/summary.txt"
}

stage_stop() {
  say "stop: shut the measurement cluster down, and prove it is down"
  stop_server
  [ -f "$DATA/postmaster.pid" ] && die "postmaster.pid is still present in $DATA"
  pgrep -a postgres | grep -F "$SANDBOX" && die "a postgres process still refers to $SANDBOX"
  note "no postmaster.pid, no matching postgres process"
  note "listeners on port $PORT: $(ss -ltn 2>/dev/null | grep -c ":$PORT ")"
}

stage_clean() {
  say "clean: stop the server and delete the sandbox"
  stage_stop
  rm -rf "$SANDBOX"
  note "deleted $SANDBOX"
}

ALL="build check sizing cluster shmem startup ckpt drop strategy vacring iocombine bcache errors summary stop"
for s in ${@:-$ALL}; do
  case "$s" in
    build|check|sizing|cluster|shmem|startup|ckpt|drop|strategy|vacring|iocombine|bcache|errors|summary|stop|clean)
      "stage_$s" ;;
    *) die "unknown stage: $s" ;;
  esac
done
```

## Context Reviewed

- Buffer manager: [bufmgr.c](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c), [freelist.c](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c), [buf_init.c](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c), [buf_table.c](../../../../raw/postgres-17/src/backend/storage/buffer/buf_table.c), [localbuf.c](../../../../raw/postgres-17/src/backend/storage/buffer/localbuf.c), [buf_internals.h](../../../../raw/postgres-17/src/include/storage/buf_internals.h), [bufmgr.h](../../../../raw/postgres-17/src/include/storage/bufmgr.h).
- Read streams and I/O: [read_stream.c](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c), [smgr.c](../../../../raw/postgres-17/src/backend/storage/smgr/smgr.c), [md.c](../../../../raw/postgres-17/src/backend/storage/smgr/md.c).
- Shared memory and startup: [ipci.c](../../../../raw/postgres-17/src/backend/storage/ipc/ipci.c), [sysv_shmem.c](../../../../raw/postgres-17/src/backend/port/sysv_shmem.c), [postmaster.c](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c).
- Writers and checkpoints: [checkpointer.c](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c), [bgwriter.c](../../../../raw/postgres-17/src/backend/postmaster/bgwriter.c), [xlog.c](../../../../raw/postgres-17/src/backend/access/transam/xlog.c).
- SLRU sizing: [slru.c](../../../../raw/postgres-17/src/backend/access/transam/slru.c), [slru.h](../../../../raw/postgres-17/src/include/access/slru.h), [clog.c](../../../../raw/postgres-17/src/backend/access/transam/clog.c), [commit_ts.c](../../../../raw/postgres-17/src/backend/access/transam/commit_ts.c), [subtrans.c](../../../../raw/postgres-17/src/backend/access/transam/subtrans.c).
- Callers of the full-pool routines: [storage.c](../../../../raw/postgres-17/src/backend/catalog/storage.c), [dbcommands.c](../../../../raw/postgres-17/src/backend/commands/dbcommands.c), [tablecmds.c](../../../../raw/postgres-17/src/backend/commands/tablecmds.c), [heapam_handler.c](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c).
- Scan strategy: [heapam.c](../../../../raw/postgres-17/src/backend/access/heap/heapam.c), [tableam.c](../../../../raw/postgres-17/src/backend/access/table/tableam.c), [syncscan.c](../../../../raw/postgres-17/src/backend/access/common/syncscan.c).
- GUCs, views and docs: [guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c), [system_views.sql](../../../../raw/postgres-17/src/backend/catalog/system_views.sql), [pgstatfuncs.c](../../../../raw/postgres-17/src/backend/utils/adt/pgstatfuncs.c), [config.sgml](../../../../raw/postgres-17/doc/src/sgml/config.sgml).
- Contrib: [pg_buffercache_pages.c](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache_pages.c), [pg_prewarm.c](../../../../raw/postgres-17/contrib/pg_prewarm/pg_prewarm.c), [autoprewarm.c](../../../../raw/postgres-17/contrib/pg_prewarm/autoprewarm.c).
- Tests that mention the setting at all: [016_min_consistency.pl](../../../../raw/postgres-17/src/test/recovery/t/016_min_consistency.pl), [032_relfilenode_reuse.pl](../../../../raw/postgres-17/src/test/recovery/t/032_relfilenode_reuse.pl), [004_io_direct.pl](../../../../raw/postgres-17/src/test/modules/test_misc/t/004_io_direct.pl), [pg_buffercache.sql](../../../../raw/postgres-17/contrib/pg_buffercache/sql/pg_buffercache.sql).
- Source history in the same checkout, for the since-v12 section: `git log`, `git tag --contains` and `git diff REL_12_2..786db8dcf16` over the files above.

## Evidence Map

| Claim | Evidence |
|---|---|
| `shared_buffers` is restart-only, floor 16, ceiling `INT_MAX / 2` | [guc_tables.c#shared_buffers](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2257-L2270); measured floor/ceiling probes in the `errors` stage |
| 256 GiB is 33,554,432 buffers and 267,499 MB of shared memory | `sizing` stage, `postgres -C shared_memory_size`; the sizing path is [ipci.c#CalculateShmemSize](../../../../raw/postgres-17/src/backend/storage/ipc/ipci.c#L90-L166) and [ipci.c#InitializeShmemGUCs](../../../../raw/postgres-17/src/backend/storage/ipc/ipci.c#L369-L398) |
| `-C` prints sizing before allocating | [postmaster.c:955-971](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L955-L971) precedes [postmaster.c:980](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L980) |
| Per-buffer overhead 64 + 32 + 20 + 16 bytes, plus mapping-table elements | `shmem` stage against [buf_init.c#BufferShmemSize](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L159-L186), [checkpointer.c#CheckpointerShmemSize](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L882-L897); marginal 8331/8364 from the `sizing` pairs |
| 257 MB power-of-two step at 2^25 blocks | `sizing` stage boundary probe; sizing rule in [freelist.c:478-488](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L478-L488) and [lwlock.h:87-93](../../../../raw/postgres-17/src/include/storage/lwlock.h#L87-L93) |
| Request queue caps at 10,000,000 slots | [checkpointer.c:133-134](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L133-L134), [checkpointer.c#CheckpointerShmemSize](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c#L882-L897); visible as the 8364 -> 8331 marginal drop |
| Checkpoint scans every header; 10.7 ns per buffer measured | [bufmgr.c#BufferSync](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2931-L3193); `ckpt` stage |
| `DROP`/`TRUNCATE` scan the pool on a primary; 5.9 ns per buffer measured | [bufmgr.c#DropRelationBuffers](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L4111-L4223), [smgr.c#smgrnblocks_cached](../../../../raw/postgres-17/src/backend/storage/smgr/smgr.c#L679-L690), [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-17/src/backend/catalog/storage.c#L657-L719); `drop` stage |
| Batching drops into one transaction costs one pass | [smgr.c#smgrdounlinkall](../../../../raw/postgres-17/src/backend/storage/smgr/smgr.c#L462-L522); `drop` stage batched row |
| Server start is 90 ns per buffer before the first log line | [buf_init.c#InitBufferPool](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c#L67-L151), [postmaster.c:1083-1084](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c#L1083-L1084); `startup` stage |
| Bulk-read strategy and sync scan only above `NBuffers / 4` | [heapam.c#initscan](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L408-L527), [tableam.c#table_block_parallelscan_initialize](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L389-L404); `strategy` stage, 18,692 vs 32 resident blocks |
| Ring cap is `NBuffers / 8`; default `VACUUM` ring 2 MB | [freelist.c#GetAccessStrategyWithSize](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L584-L614), [guc_tables.c#vacuum_buffer_usage_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2272-L2281); `vacring` stage, 16,399 blocks at the 16,384 cap |
| `pg_stat_io.reads` counts blocks, so combining is invisible | [bufmgr.c:1519-1521](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L1519-L1521), [pgstatfuncs.c:1425-1431](../../../../raw/postgres-17/src/backend/utils/adt/pgstatfuncs.c#L1425-L1431); `iocombine` stage, 37,489 reads at all four settings |
| `pg_buffercache` row reader is 55x the summary readers | [pg_buffercache_pages.c#pg_buffercache_pages](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache_pages.c#L69-L246); `bcache` stage |
| Direct I/O removes the OS cache from the data path | [guc_tables.c#debug_io_direct](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4699-L4708); `errors` stage, 522.2 ms vs 43.7 ms `read_time` |
| SLRU auto-sizing to a 1024-block cap | [slru.c#SimpleLruAutotuneBuffers](../../../../raw/postgres-17/src/backend/access/transam/slru.c#L232-L237), [clog.c#CLOGShmemBuffers](../../../../raw/postgres-17/src/backend/access/transam/clog.c#L768-L775); `shmem` stage, 32 -> 256 -> 1024 blocks |
| Since-v12 attributions | `git log -S<symbol>` plus `git tag --contains` in `raw/postgres-17`, commit ids in the delta table |
| No NUMA support | grep for `NUMA`, `libnuma`, `numa_available` over `src/`, `contrib/`, `configure.ac` returns nothing |

## Open Questions

1. **Whether 256 GiB is the right size for a given 1 TiB host is not answerable from this evidence.** Source fixes the mechanisms and this run fixes their slopes on one machine; neither models a workload. The documentation's 25 % / 40 % guidance is guidance, not a measurement ([config.sgml#shared_buffers](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L1650-L1692)).
2. **No figure was measured at 33,554,432 buffers.** The 0.36 s checkpoint scan, 0.2 s `DROP` and 3 s startup at 256 GiB are linear extrapolations of slopes measured between 131,072 and 2,097,152 buffers. Whether those slopes stay linear across a 16x larger array, with TLB and NUMA effects in play, is untested here.
3. **Every timing is single-backend and uncontended.** The 128 mapping partitions, `buffer_strategy_lock`, and the clock sweep's behaviour under many concurrent allocators are exactly what a large pool is supposed to stress, and this run does not stress them.
4. **The `io_combine_limit` result is inconclusive, not negative.** Scan times were 157.7, 141.5, 156.8 and 146.9 ms at 8 kB, 32 kB, 128 kB and 256 kB, with the file in the host page cache and only one run each; that ordering is noise, not evidence that combining does nothing. A cold-storage test with repeated runs is needed, and v17's counters cannot help because `reads` is a block count.
5. **The `drop` stage measures a primary only.** The v14 targeted-lookup path is reachable only when `InRecovery` ([smgr.c#smgrnblocks_cached](../../../../raw/postgres-17/src/backend/storage/smgr/smgr.c#L679-L690)), so the claim that a standby's replay avoids the full scan is source-verified and unmeasured; it needs a standby leg.
6. **`TRUNCATE` did not scale with fork count, which contradicts the script's own comment.** The measured main-only and main+vm+fsm medians are within noise of each other, consistent with relfilenode replacement dropping the old file once at commit rather than per fork. The per-fork `smgrtruncate()` path was not isolated; a `VACUUM`-truncation leg would settle it.
7. **The 257 MB mapping-table step is measured through `shared_memory_size`, not through dynahash's own accounting.** The bucket-doubling explanation is consistent with [freelist.c:478-488](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L478-L488) and the observed 2^n boundaries, but no per-structure figure confirms the 257 MB is buckets alone.
8. **Huge pages were never actually used.** `HugePages_Total` was 0 on the test host, so `shared_memory_size_in_huge_pages` is a computation and `huge_pages = on` was only measured failing. The 1 GiB-page figure of 262 pages is likewise a computation.
9. **No shipped test covers a large pool**, so there is no upstream regression coverage behind any of the pool-size-dependent paths above; the largest setting anywhere in the tree is 1 MB.

## Source References

- [src/backend/storage/buffer/bufmgr.c](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c) - `BUF_DROP_FULL_SCAN_THRESHOLD` (84-89), `ReadBuffer_common` (1196-1264), `WaitReadBuffers` (1411-1588) including the block-count I/O statistics at 1519-1521, `GetVictimBuffer` (1954-2105) including eviction/reuse counting at 2061-2081, `LimitAdditionalPins` (2120-2144), `BufferSync` (2931-3193), `BgBufferSync` (3207-3488), `FlushBuffer` (3863-3991) with `XLogFlush` at 3927 and `smgrwrite` at 3948, `DropRelationBuffers` (4111-4223), `DropRelationsAllBuffers` (4234-4393), `FindAndDropRelationBuffers` (4405-4452), `DropDatabaseBuffers` (4466-4493), `FlushRelationBuffers` (4553-4658), `FlushRelationsAllBuffers` (4670-4757), `FlushDatabaseBuffers` (4925-4960), `EvictUnpinnedBuffer` (6160-6204).
- [src/backend/storage/buffer/freelist.c](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c) - `BufferStrategyControl` (30-62), `ClockSweepTick` (108-164), `StrategyGetBuffer` (196-357), `StrategyShmemSize` (453-464), `StrategyInitialize` (474-526), `GetAccessStrategy` (541-574), `GetAccessStrategyWithSize` (584-614), `GetAccessStrategyPinLimit` (647-672), `StrategyRejectBuffer` (798-816).
- [src/backend/storage/buffer/buf_init.c](../../../../raw/postgres-17/src/backend/storage/buffer/buf_init.c) - `InitBufferPool` (67-151), `BufferShmemSize` (159-186).
- [src/backend/storage/buffer/buf_table.c](../../../../raw/postgres-17/src/backend/storage/buffer/buf_table.c) - `BufTableShmemSize` (41-44), `InitBufTable` (51-66).
- [src/backend/storage/buffer/localbuf.c](../../../../raw/postgres-17/src/backend/storage/buffer/localbuf.c) - `LimitAdditionalLocalPins` (290-306).
- [src/include/storage/buf_internals.h](../../../../raw/postgres-17/src/include/storage/buf_internals.h) - `BM_MAX_USAGE_COUNT` (71-79), `BufferDesc` (246-258).
- [src/include/storage/bufmgr.h](../../../../raw/postgres-17/src/include/storage/bufmgr.h) - `NBuffers` declaration (149), prefetch-dependent I/O concurrency defaults (157-166).
- [src/include/storage/lwlock.h](../../../../raw/postgres-17/src/include/storage/lwlock.h) - `NUM_BUFFER_PARTITIONS` (87-93).
- [src/include/storage/proc.h](../../../../raw/postgres-17/src/include/storage/proc.h) - `NUM_SPECIAL_WORKER_PROCS` and `NUM_AUXILIARY_PROCS` (431-443).
- [src/include/pg_config_manual.h](../../../../raw/postgres-17/src/include/pg_config_manual.h) - `DEFAULT_CHECKPOINT_FLUSH_AFTER` (170-180).
- [src/backend/utils/init/postinit.c](../../../../raw/postgres-17/src/backend/utils/init/postinit.c) - the `MaxBackends` formula (579-584).
- [src/backend/storage/aio/read_stream.c](../../../../raw/postgres-17/src/backend/storage/aio/read_stream.c) - look-ahead design (1-45), pin limits in `read_stream_begin_relation` (452-478).
- [src/backend/storage/ipc/ipci.c](../../../../raw/postgres-17/src/backend/storage/ipc/ipci.c) - `CalculateShmemSize` (90-166), `InitializeShmemGUCs` (369-398).
- [src/backend/port/sysv_shmem.c](../../../../raw/postgres-17/src/backend/port/sysv_shmem.c) - `GetHugePageSize` (479-572), `CreateAnonymousSegment` (599-668).
- [src/backend/postmaster/postmaster.c](../../../../raw/postgres-17/src/backend/postmaster/postmaster.c) - runtime-computed `-C` exit (955-971), `CreateSharedMemoryAndSemaphores` (980), `starting %s` (1083-1084).
- [src/backend/postmaster/checkpointer.c](../../../../raw/postgres-17/src/backend/postmaster/checkpointer.c) - `MAX_CHECKPOINT_REQUESTS` (133-134), `CheckpointerShmemSize` (882-897), `ForwardSyncRequest` (1099-1141), `CompactCheckpointerRequestQueue` (1160-1258), `AbsorbSyncRequests` (1270-1311).
- [src/backend/access/transam/slru.c](../../../../raw/postgres-17/src/backend/access/transam/slru.c) - `SimpleLruShmemSize` (199-222), `SimpleLruAutotuneBuffers` (232-237).
- [src/include/access/slru.h](../../../../raw/postgres-17/src/include/access/slru.h) - `SLRU_MAX_ALLOWED_BUFFERS` (24).
- [src/backend/access/transam/clog.c](../../../../raw/postgres-17/src/backend/access/transam/clog.c) - `CLOGShmemBuffers` (768-775).
- [src/backend/access/transam/commit_ts.c](../../../../raw/postgres-17/src/backend/access/transam/commit_ts.c) - `CommitTsShmemBuffers` (506-513).
- [src/backend/access/transam/subtrans.c](../../../../raw/postgres-17/src/backend/access/transam/subtrans.c) - `SUBTRANSShmemBuffers` (201-208).
- [src/backend/access/transam/xlog.c](../../../../raw/postgres-17/src/backend/access/transam/xlog.c) - `XLOGChooseNumBuffers` (4575-4585).
- [src/backend/access/heap/heapam.c](../../../../raw/postgres-17/src/backend/access/heap/heapam.c) - `initscan` (408-527), sequential-scan read stream (1252-1258).
- [src/backend/access/table/tableam.c](../../../../raw/postgres-17/src/backend/access/table/tableam.c) - `table_block_parallelscan_initialize` (389-404).
- [src/backend/access/heap/syncscan.c](../../../../raw/postgres-17/src/backend/access/common/syncscan.c) - `SYNC_SCAN_REPORT_INTERVAL` (73-83).
- [src/backend/catalog/storage.c](../../../../raw/postgres-17/src/backend/catalog/storage.c) - `RelationTruncate` (288-439), `smgrDoPendingDeletes` (657-719), `smgr_redo` (965-1079).
- [src/backend/storage/smgr/smgr.c](../../../../raw/postgres-17/src/backend/storage/smgr/smgr.c) - `smgrdounlinkall` (462-522), `smgrnblocks_cached` (679-690).
- [src/backend/commands/dbcommands.c](../../../../raw/postgres-17/src/backend/commands/dbcommands.c) - `createdb` (670-1532), `dropdb` (1634-1856), `movedb` (1964-2282), `dbase_redo` (3270-3432).
- [src/backend/optimizer/path/costsize.c](../../../../raw/postgres-17/src/backend/optimizer/path/costsize.c) - `effective_cache_size` rationale (22-26).
- [src/backend/utils/misc/guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c) - `shared_buffers` (2257-2270), `vacuum_buffer_usage_limit` (2272-2281), `shared_memory_size` (2283-2292), `shared_memory_size_in_huge_pages` (2294-2303), `commit_timestamp_buffers` (2305-2314), `subtransaction_buffers` (2360-2369), `transaction_buffers` (2371-2380), `temp_buffers` (2382-2391), `max_wal_size` (2842-2852), `checkpoint_flush_after` (2880-2889), `bgwriter_delay` (3077-3086), `bgwriter_lru_maxpages` (3088-3096), `io_combine_limit` (3138-3150), `backend_flush_after` (3152-3161), `effective_cache_size` (3508-3518), `huge_page_size` (3598-3607), `checkpoint_completion_target` (3916-3924), `debug_io_direct` (4699-4708), `huge_pages` (5057-5065), `huge_pages_status` (5067-5076).
- [src/backend/utils/init/globals.c](../../../../raw/postgres-17/src/backend/utils/init/globals.c) - `NBuffers` default (139).
- [src/backend/catalog/system_views.sql](../../../../raw/postgres-17/src/backend/catalog/system_views.sql) - `pg_shmem_allocations` and its grants (652-658), `pg_stat_bgwriter` (1134-1139), `pg_stat_checkpointer` (1141-1151), `pg_stat_io` (1153-1173).
- [src/backend/access/heap/heapam_handler.c](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c) - heap copy for `SET TABLESPACE` (637-645).
- [src/backend/commands/tablecmds.c](../../../../raw/postgres-17/src/backend/commands/tablecmds.c) - index copy for `SET TABLESPACE` (15586-15595).
- [src/backend/commands/sequence.c](../../../../raw/postgres-17/src/backend/commands/sequence.c) - an unlogged sequence's init fork (344-352).
- [src/backend/utils/adt/pgstatfuncs.c](../../../../raw/postgres-17/src/backend/utils/adt/pgstatfuncs.c) - `op_bytes` hard-coded to `BLCKSZ` (1425-1431).
- [doc/src/sgml/config.sgml](../../../../raw/postgres-17/doc/src/sgml/config.sgml) - `shared_buffers` (1650-1692), `vacuum_buffer_usage_limit` (1974-2002), `transaction_buffers` (2143-2160), `io_combine_limit` (2778-2800), `huge_pages_status` (11204-11215), `shared_memory_size` (11365-11378), `shared_memory_size_in_huge_pages` (11379-11395), `debug_io_direct` (11595-11620).
- [contrib/pg_buffercache/pg_buffercache_pages.c](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache_pages.c) - `pg_buffercache_pages` (69-246), `pg_buffercache_summary` (249-313), `pg_buffercache_usage_counts` (316-356), `pg_buffercache_evict` (362-375).
- [contrib/pg_buffercache/pg_buffercache.control](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache.control#L1-L5) - default version 1.5.
- [contrib/pg_buffercache/pg_buffercache--1.2--1.3.sql](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache--1.2--1.3.sql#L1-L7), [pg_buffercache--1.3--1.4.sql](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache--1.3--1.4.sql#L24-L28), [pg_buffercache--1.4--1.5.sql](../../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache--1.4--1.5.sql#L1-L6) - the `pg_monitor` grants, and the ungranted `pg_buffercache_evict()`.
- [contrib/pg_prewarm/pg_prewarm.c](../../../../raw/postgres-17/contrib/pg_prewarm/pg_prewarm.c) - `pg_prewarm` (76-291).
- [contrib/pg_prewarm/autoprewarm.c](../../../../raw/postgres-17/contrib/pg_prewarm/autoprewarm.c) - GUC definitions (104-120), `autoprewarm_database_main` (437-561), `apw_dump_now` (570-713).
- [src/test/recovery/t/016_min_consistency.pl](../../../../raw/postgres-17/src/test/recovery/t/016_min_consistency.pl#L49-L56), [src/test/recovery/t/032_relfilenode_reuse.pl](../../../../raw/postgres-17/src/test/recovery/t/032_relfilenode_reuse.pl#L18-L24), [src/test/modules/test_misc/t/004_io_direct.pl](../../../../raw/postgres-17/src/test/modules/test_misc/t/004_io_direct.pl#L44-L50), [contrib/pg_buffercache/sql/pg_buffercache.sql](../../../../raw/postgres-17/contrib/pg_buffercache/sql/pg_buffercache.sql#L1-L12) - the only `shared_buffers` settings and pool assertions in the test tree.

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- Same question for PostgreSQL 12: [Pros and Cons of a Very Large shared_buffers Such as 256 GB on a 1 TB RAM System in PostgreSQL 12 (unverified)](../../../v12/questions/storage-and-vacuum/very-large-shared-buffers.md)
