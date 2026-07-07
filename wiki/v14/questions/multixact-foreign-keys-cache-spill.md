---
type: question
version: 14
pinned_commit: 5c00f4e2e3bcee6931ae93429d53f7c2a4f46156
verified: false
verified_by_agent: not yet
---

# How MultiXact Works in PostgreSQL 14, and How Foreign Keys and Other Operations Degrade Performance When the Local Cache Spills to Secondary Storage (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [How MultiXact works](#how-multixact-works)
  - [The backend-local MultiXact cache and its spill](#the-backend-local-multixact-cache-and-its-spill)
  - [How foreign keys create MultiXacts](#how-foreign-keys-create-multixacts)
  - [Other operations that create MultiXacts](#other-operations-that-create-multixacts)
  - [How cache spill and MultiXact traffic degrade performance](#how-cache-spill-and-multixact-traffic-degrade-performance)
  - [Observability in v14](#observability-in-v14)
  - [Mitigations and settings](#mitigations-and-settings)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Explain how MultiXact works, and how foreign keys and other operations could lead to database engine performance degradation when the MultiXact local cache fills up and spills to secondary storage.

> Prompt note: this is a lightly grammar-corrected restatement of the original request, applied with the asker's approval per the wiki's prompt-hygiene rule.

## Answer

### Short answer

A **MultiXact** (multiple-transaction ID, or MXID) is how PostgreSQL records that *more than one* transaction holds a lock on the same row. A tuple header has only one `xmax` slot, so when a second locker or a locker-plus-updater must be recorded on one row, PostgreSQL allocates a MultiXactId and stores the actual member list — each `(xid, lock-mode)` pair — outside the heap, in two SLRU (simple least-recently-used) logs under `pg_multixact/` [multixact.c overview](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L6-L56).

Foreign keys are the most common way to reach this path: an insert or update on a child table takes a `FOR KEY SHARE` lock on the referenced parent row [ri_triggers.c#RI_FKey_check](../../../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L358-L386). When several transactions hold key-share locks on the *same* parent row, or a key-share locker coexists with an updater, PostgreSQL turns that row's `xmax` into a MultiXactId [heapam.c#compute_new_xmax_infomask](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L5333-L5495).

Each backend keeps a small **local cache** of MultiXacts it has recently created or read, capped at 256 entries and living only for the current transaction [multixact.c#MAX_CACHE_ENTRIES](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L313-L324). When that cache cannot answer a lookup — because it filled past 256 entries and evicted the oldest, because the transaction is new, or because the MultiXact was created by *another* backend — the lookup falls back to the SLRU, whose in-memory pool in v14 is only 8 offset pages and 16 member pages [multixact.h#buffers](../../../raw/postgres-14/src/include/access/multixact.h#L32-L34). On an SLRU miss the page is read from the on-disk `pg_multixact/` files [slru.c#SlruPhysicalReadPage](../../../raw/postgres-14/src/backend/access/transam/slru.c#L684-L739). The MultiXact read path enters `SimpleLruReadPage` while holding the relevant SLRU control lock in **exclusive** mode, then the SLRU code marks the slot read-busy, takes the per-buffer lock, releases the control lock during physical I/O, and reacquires it to publish the result [multixact.c#GetMultiXactIdMembers](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1472-L1529) [slru.c#SimpleLruReadPage-IO](../../../raw/postgres-14/src/backend/access/transam/slru.c#L439-L457). Heavy MultiXact traffic therefore contends on `MultiXactOffsetSLRULock`/`MultiXactMemberSLRULock`, per-buffer locks, and `SLRURead` I/O waits. Separately, member-space growth drives more aggressive (anti-wraparound) vacuuming and, at the extreme, refuses new MultiXacts [multixact.c#members-limit](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1246-L1264).

### How MultiXact works

#### What a MultiXact is

A tuple stores exactly one `xmax`. That is enough for a single locker or a single deleter/updater, but not for the common case where two or more live transactions lock the same row. To represent "several transactions lock this row," PostgreSQL allocates a MultiXactId and writes it into `xmax` with the `HEAP_XMAX_IS_MULTI` infomask bit. The MultiXactId is a 32-bit counter [multixact.h#Ids](../../../raw/postgres-14/src/include/access/multixact.h#L24-L30); the list of members it stands for is kept separately, and each member is a `(TransactionId, MultiXactStatus)` pair [multixact.h#MultiXactMember](../../../raw/postgres-14/src/include/access/multixact.h#L60-L64). The status encodes the member's lock strength: `ForKeyShare`, `ForShare`, `ForNoKeyUpdate`, `ForUpdate`, plus the two update/delete modes `NoKeyUpdate` and `Update` [multixact.h#MultiXactStatus](../../../raw/postgres-14/src/include/access/multixact.h#L41-L51). The name is historical — a MultiXact may legitimately contain a single member today [multixact.c#naming](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L6-L12).

The PostgreSQL documentation says the same thing: multixact IDs support row locking by multiple transactions, only the multixact ID appears in `xmax`, and the member list lives in `pg_multixact` [maintenance.sgml#Multixacts](../../../raw/postgres-14/doc/src/sgml/maintenance.sgml#L729-L748).

#### The two SLRUs and the pg_multixact files

Because a MultiXact holds a *variable-length* array of members, PostgreSQL uses two SLRU areas: an **offsets** log that maps each MultiXactId to the starting position of its members, and a **members** log that stores the packed `(xid, flags)` arrays [multixact.c#two-SLRUs](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L19-L25). Both are initialized as SLRUs backed by the directories `pg_multixact/offsets` and `pg_multixact/members` [multixact.c#SimpleLruInit](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1982-L1993). SLRU is a small in-memory page pool with straight LRU replacement, sized for the assumption that most traffic hits the newest page while reads touch a wider span [slru.c#overview](../../../raw/postgres-14/src/backend/access/transam/slru.c#L6-L14). In v14 the pools are fixed at compile time: **8** offset buffers and **16** member buffers [multixact.h#buffers](../../../raw/postgres-14/src/include/access/multixact.h#L32-L34), allocated from shared memory at startup [multixact.c#ShmemSize](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1965-L1967).

#### Key data structures

Shared MultiXact state lives in `MultiXactStateData`: the next MultiXactId to assign (`nextMXact`), the next members offset (`nextOffset`), the oldest values still referenced by any relation, and the anti-wraparound limits (`multiVacLimit`, `multiWarnLimit`, `multiStopLimit`, `multiWrapLimit`, and the member-space `offsetStopLimit`) [multixact.c#MultiXactStateData](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L200-L234). Two per-backend arrays follow it: `OldestMemberMXactId[]` and `OldestVisibleMXactId[]`, which bound how far back each backend could still be a member of, or need to read, a MultiXact — vacuum uses their minimum to decide what is safe to truncate [multixact.c#perBackend](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L236-L281). All of this is protected by `MultiXactGenLock`, with the two SLRU control locks guarding the buffer pools [multixact.c#locks](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L193-L198).

#### Creating a MultiXact (write path)

Row-lock code calls `MultiXactIdCreate` (two members) or `MultiXactIdExpand` (add a member to an existing MultiXact) [multixact.c#MultiXactIdCreate](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L398-L430). Expansion never mutates an existing MultiXact in place — it always builds a *new* one, so that code waiting on the old MultiXact is not disturbed [multixact.c#MultiXactIdExpand](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L451-L495). Both funnel into `MultiXactIdCreateFromMembers`, which:

1. checks the local cache for an identical member set and reuses that MultiXactId if found [multixact.c#reuse](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L789-L804);
2. otherwise calls `GetNewMultiXactId` to reserve an MXID and a members-offset range [multixact.c#GetNew-call](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L823-L835);
3. writes a WAL record, then records the offset and member bytes into the two SLRUs via `RecordNewMultiXact` [multixact.c#xlog-record](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L837-L855);
4. finally stores the new MultiXact in the local cache [multixact.c#cachePut-create](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L860-L861).

`RecordNewMultiXact` writes the offsets page under `MultiXactOffsetSLRULock` (exclusive) and then each member's `(xid, flags)` under `MultiXactMemberSLRULock` (exclusive) [multixact.c#RecordNewMultiXact-members](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1029-L1071). `nextOffset` advances by the number of members, so **member space is consumed monotonically** and never reused until vacuum truncates old segments [multixact.c#advance](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1322-L1324).

#### Reading a MultiXact (read path)

To resolve a MultiXactId back into its members, code calls `GetMultiXactIdMembers` [multixact.c#GetMultiXactIdMembers](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1359-L1362). It first consults the local cache and returns immediately on a hit [multixact.c#cache-lookup](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1386-L1393). On a miss it reads the offsets SLRU page to find the member range [multixact.c#offset-read](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1472-L1480), then walks the members SLRU pages to collect each `(xid, status)` [multixact.c#member-read](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1529-L1572), and finally copies the result into the local cache [multixact.c#cachePut-read](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1580). Crucially, the MultiXact path enters both SLRU reads with the control lock in **exclusive** mode — `GetMultiXactIdMembers` acquires `MultiXactOffsetSLRULock`/`MultiXactMemberSLRULock` in `LW_EXCLUSIVE` and calls `SimpleLruReadPage`, rather than using the shared-lock `SimpleLruReadPage_ReadOnly` fast path that CLOG uses [slru.c#ReadOnly](../../../raw/postgres-14/src/backend/access/transam/slru.c#L494-L525). `SimpleLruReadPage` can release that control lock around physical I/O after marking a slot read-busy, so the contention boundary is SLRU state lookup/update plus per-buffer I/O waits, not a control lock held across the whole `pg_pread` [slru.c#SimpleLruReadPage-IO](../../../raw/postgres-14/src/backend/access/transam/slru.c#L439-L457).

### The backend-local MultiXact cache and its spill

#### What it stores and how long it lives

Each backend keeps a private cache of MultiXacts so it does not have to hit the SLRU on every lookup [multixact.c#cache-purpose](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L296-L300). Entries are `mXactCacheEnt` records (the MultiXactId, its member count, an intrusive list node, and the member array), held on a doubly-linked list `MXactCache` [multixact.c#mXactCacheEnt](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L313-L322). The cache lives in `MXactContext`, a memory context created under `TopTransactionContext`, so **the entire cache is thrown away at transaction end** [multixact.c#MXactContext](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1721-L1728). Two lookups exist: `mXactCacheGetBySet` (find a MultiXactId by its exact member set, used to reuse an MXID when locking many rows) [multixact.c#GetBySet](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1612-L1657), and `mXactCacheGetById` (find the members for a given MultiXactId) [multixact.c#GetById](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1659-L1707). Both move a hit to the list head, so the list is ordered most-recently-used to least.

#### The 256-entry cap and LRU eviction

`mXactCachePut` pushes each new entry at the list head, then — once the count reaches `MAX_CACHE_ENTRIES` (256) — deletes the tail (least-recently-used) entry [multixact.c#mXactCachePut](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1713-L1758). This is the "fill up and spill" the question asks about: the local cache is a per-backend, per-transaction LRU capped at 256 MultiXacts. Once a transaction touches more than 256 distinct MultiXacts, every further one evicts the oldest, and any later reference to an evicted MultiXact **misses the cache and must go to the SLRU** (shared buffers, then the on-disk `pg_multixact/` files). Note the cache offers no cross-backend sharing: a MultiXact created by another backend is never in *this* backend's cache until this backend reads it from the SLRU at least once, and even a self-created MultiXact is gone as soon as the creating transaction ends.

### How foreign keys create MultiXacts

A foreign-key constraint is enforced by referential-integrity (RI) triggers. When a row is inserted or updated in the referencing (child) table, the check trigger runs a query of the form `SELECT 1 FROM <pktable> x WHERE pkatt = $1 ... FOR KEY SHARE OF x` against the referenced (parent) table [ri_triggers.c#RI_FKey_check](../../../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L358-L389). The parent relation is opened in `RowShareLock` mode precisely because the eventual row lock is `FOR KEY SHARE` [ri_triggers.c#RowShareLock](../../../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L266-L268). The restrict/no-action triggers on the parent side likewise open the child table in `RowShareLock` and query it with `FOR KEY SHARE OF x` [ri_triggers.c#ri_restrict](../../../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L617-L709).

`FOR KEY SHARE` is the weakest row lock: it exists so that FK checks do not block, and are not blocked by, ordinary non-key updates of the parent. But it is still a *lock recorded on the parent row*. The row lock flows through the executor's `table_tuple_lock` into the heap AM handler and then `heap_lock_tuple` [heapam_handler.c#heapam_tuple_lock](../../../raw/postgres-14/src/backend/access/heap/heapam_handler.c#L365-L367). `heap_lock_tuple` marks the backend's oldest-member bookkeeping and calls `compute_new_xmax_infomask` to decide the new `xmax` [heapam.c#heap_lock_tuple](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L5102-L5111). That function is where a MultiXact is born:

- If the parent row's `xmax` already **is a MultiXact**, the new locker is added with `MultiXactIdExpand` [heapam.c#expand](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L5385-L5389).
- If the parent row's `xmax` is a **still-running** single locker or updater, the two are combined into a brand-new MultiXact with `MultiXactIdCreate` [heapam.c#create-inprogress](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L5491-L5495).
- If the parent row carries a **committed updater**, a MultiXact preserves that updater alongside the new locker [heapam.c#create-committed](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L5391-L5413).

So the FK hot spot is a **"hot" parent row referenced by many concurrent child DML statements**: transaction A inserts a child → key-share lock on the parent row; transaction B inserts another child referencing the same parent → the parent's `xmax` becomes `MultiXactId{A, B}`; C, D, ... each expand it into a new, larger MultiXact. The in-tree isolation test `fk-contention` models exactly this — one session inserts into a child that references `foo(42)` while another updates `foo`, forcing the key-share lock and the update to coexist on one parent row [fk-contention.spec](../../../raw/postgres-14/src/test/isolation/specs/fk-contention.spec#L1-L19).

### Other operations that create MultiXacts

MultiXacts are not FK-specific. Anything that puts a second live locker (or a locker plus an updater) on a row goes through the same `compute_new_xmax_infomask` logic:

- **Explicit compatible row locks**: `SELECT ... FOR KEY SHARE / FOR SHARE / FOR NO KEY UPDATE / FOR UPDATE` routes row-mark locking through `table_tuple_lock` [nodeLockRows.c#ExecLockRows](../../../raw/postgres-14/src/backend/executor/nodeLockRows.c#L160-L190). A MultiXact forms when the new lock can coexist with an existing live locker or updater and therefore must be recorded in the same tuple `xmax`; compatible examples include many sessions taking `FOR KEY SHARE`, many sessions taking `FOR SHARE`, or `FOR KEY SHARE` coexisting with `FOR NO KEY UPDATE` on one row [mvcc.sgml#row-lock-compatibility](../../../raw/postgres-14/doc/src/sgml/mvcc.sgml#L1260-L1307). Conflicting row-lock combinations wait instead of immediately becoming co-members of one MultiXact [mvcc.sgml#row-lock-table](../../../raw/postgres-14/doc/src/sgml/mvcc.sgml#L1321-L1355).
- **UPDATE/DELETE against a locked row**: `heap_update` and `heap_delete` both call `compute_new_xmax_infomask`, so an update of a row that FK key-share lockers are holding must fold those lockers into a MultiXact [heapam.c#heap_delete](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L2998-L3000) [heapam.c#heap_update](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L3694-L3811).
- **`VACUUM` freezing of MultiXacts**: when a MultiXact still has some running members, freezing rebuilds it with the surviving members via `MultiXactIdCreateFromMembers`, which itself consumes a fresh MXID and member-space [heapam.c#FreezeMultiXactId](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L6967-L6969).

### How cache spill and MultiXact traffic degrade performance

Three distinct effects stack up. The first is the direct "cache spilled to secondary storage" path; the other two are what make that path expensive at scale.

#### 1. Local-cache miss falls back to the SLRU and its files

When `GetMultiXactIdMembers` misses the 256-entry local cache, it reads the offsets and members pages through the SLRU [multixact.c#miss-to-slru](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1472-L1572). If the needed page is not among the 8 offset / 16 member in-memory buffers, `SimpleLruReadPage` selects a victim slot and issues a physical read [slru.c#SimpleLruReadPage](../../../raw/postgres-14/src/backend/access/transam/slru.c#L434-L451). That physical read is a `pg_pread` from the `pg_multixact/` segment file on disk, reported as the `SLRURead` wait event [slru.c#SlruPhysicalReadPage](../../../raw/postgres-14/src/backend/access/transam/slru.c#L719-L729). This is the concrete meaning of "spill to secondary storage": once the local cache no longer answers the lookup and the tiny SLRU pool no longer holds the page, resolving a row's lockers becomes a disk read.

This gets worse the older and more scattered the referenced MultiXacts are. A workload that keeps reading rows whose `xmax` is an old MultiXact (for example, long-lived hot parent rows) references MultiXactIds spread across many SLRU pages. With only 8 offset and 16 member pages cached, those references thrash the pool.

#### 2. SLRU buffer contention and exclusive locking

Resolving a MultiXactId to its members in `GetMultiXactIdMembers` enters the offsets and members SLRU reads with the SLRU control lock held **exclusively** (see the read path above), so — unlike CLOG's `SimpleLruReadPage_ReadOnly` shared-lock fast path [slru.c#ReadOnly](../../../raw/postgres-14/src/backend/access/transam/slru.c#L494-L525) — MultiXact lookups contend on `MultiXactOffsetSLRULock` and `MultiXactMemberSLRULock` while checking the SLRU state [lwlocknames.txt](../../../raw/postgres-14/src/backend/storage/lmgr/lwlocknames.txt#L21-L23). The physical I/O itself is not done while holding the control lock: `SimpleLruReadPage` marks the slot read-busy, takes the per-buffer lock, releases the control lock for `SlruPhysicalReadPage`, then reacquires it to mark the page valid [slru.c#SimpleLruReadPage-IO](../../../raw/postgres-14/src/backend/access/transam/slru.c#L439-L457). Dirty-victim writes follow the same pattern through `SlruInternalWritePage`: mark write-busy under the control lock, take the per-buffer lock, release the control lock for physical write I/O, then reacquire it [slru.c#SlruInternalWritePage](../../../raw/postgres-14/src/backend/access/transam/slru.c#L539-L590). Under buffer pressure the victim search still stalls when every page is I/O-busy [slru.c#all-busy](../../../raw/postgres-14/src/backend/access/transam/slru.c#L1117-L1128), and concurrent backends needing a page that is mid-I/O block in `SimpleLruWaitIO` [slru.c#SimpleLruWaitIO](../../../raw/postgres-14/src/backend/access/transam/slru.c#L341-L349). The visible symptom is many sessions in `LWLock` waits on the MultiXact SLRU locks and per-buffer tranches `MultiXactOffsetBuffer`/`MultiXactMemberBuffer` [lwlock.c#tranches](../../../raw/postgres-14/src/backend/storage/lmgr/lwlock.c#L138-L140), plus `SLRURead` I/O waits — throughput can collapse even though CPU is idle.

#### 3. Member-space growth, aggressive vacuum, and wraparound

Because member space is consumed monotonically, sustained MultiXact creation grows the members SLRU on disk. `GetNewMultiXactId` guards this space:

- It signals the autovacuum launcher once member usage passes half the address space (`MULTIXACT_MEMBER_SAFE_THRESHOLD`), on each segment boundary [multixact.c#autovac-trigger](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1266-L1285).
- It warns when close to `offsetStopLimit` [multixact.c#warn](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1287-L1298).
- It **refuses to create new MultiXacts** with `ERROR: multixact "members" limit exceeded` once the limit is reached — blocking exactly the write transactions that need an MXID [multixact.c#members-limit](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1246-L1264).

As member usage rises, `MultiXactMemberFreezeThreshold` clamps the effective `autovacuum_multixact_freeze_max_age` downward — in the worst case to zero — which makes both autovacuum and manual `VACUUM` freeze MultiXacts far more aggressively (whole-table scans) to reclaim member space [multixact.c#MemberFreezeThreshold](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L2964-L3000). The documentation quantifies the same thresholds: once member storage exceeds about 10 GB aggressive scans run more often, the members area can reach about 20 GB before wraparound, warnings start 40 million from the wraparound point, and new MXIDs are refused with fewer than 3 million left [maintenance.sgml#members-storage](../../../raw/postgres-14/doc/src/sgml/maintenance.sgml#L774-L791). So a MultiXact-heavy FK workload pays twice: SLRU read/lock contention now, and a rising background (and eventually foreground-blocking) vacuum burden over time.

### Observability in v14

- **`pg_stat_slru`** exposes per-SLRU counters, including `blks_hit`, `blks_read` (disk reads), and `blks_written`, with rows named `MultiXactOffset` and `MultiXactMember` [system_views.sql#pg_stat_slru](../../../raw/postgres-14/src/backend/catalog/system_views.sql#L866-L877) [pgstat.c#slru_names](../../../raw/postgres-14/src/backend/postmaster/pgstat.c#L146-L157). A rising `blks_read` on the MultiXact SLRUs is the direct signal that lookups are spilling to disk.
- **`pg_stat_activity.wait_event_type` / `wait_event`** [system_views.sql#pg_stat_activity](../../../raw/postgres-14/src/backend/catalog/system_views.sql#L828-L829) show the contention: `LWLock` / `MultiXactOffsetSLRULock` | `MultiXactMemberSLRULock` | `MultiXactOffsetBuffer` | `MultiXactMemberBuffer`, and `IO` / `SLRURead` [wait_event.c#SLRURead](../../../raw/postgres-14/src/backend/utils/activity/wait_event.c#L645-L646).
- **`mxid_age(relminmxid)`** measures how far a table's oldest MultiXact is from the current counter, to find wraparound risk [pg_proc.dat#mxid_age](../../../raw/postgres-14/src/include/catalog/pg_proc.dat#L2377-L2378); **`pg_get_multixact_members()`** decodes a MultiXactId into its members [multixact.c#pg_get_multixact_members](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L3524-L3553).

The following read-only queries were checked against the pinned checkout (session-scoped timeouts recommended):

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';

-- MultiXact SLRU hit/miss and disk I/O
SELECT /* wiki_multixact_slru_io */ name, blks_zeroed, blks_hit, blks_read, blks_written
FROM   pg_stat_slru
WHERE  name IN ('MultiXactOffset', 'MultiXactMember');

-- Backends currently waiting on MultiXact SLRU locks or SLRU reads
SELECT /* wiki_multixact_waits */ wait_event_type, wait_event, count(*)
FROM   pg_stat_activity
WHERE  wait_event IN ('MultiXactOffsetSLRULock', 'MultiXactMemberSLRULock',
                      'MultiXactOffsetBuffer', 'MultiXactMemberBuffer', 'SLRURead')
GROUP  BY 1, 2
ORDER  BY 3 DESC;
```

### Mitigations and settings

There is **no v14 GUC to enlarge the MultiXact SLRU buffers** — they are the compile-time constants `NUM_MULTIXACTOFFSET_BUFFERS` (8) and `NUM_MULTIXACTMEMBER_BUFFERS` (16) [multixact.h#buffers](../../../raw/postgres-14/src/include/access/multixact.h#L32-L34), and the 256-entry local cache `MAX_CACHE_ENTRIES` is likewise a constant [multixact.c#MAX_CACHE_ENTRIES](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L321). So in v14 the practical levers are workload design and vacuum tuning, not buffer sizing:

- **Reduce MultiXact generation.** Avoid many concurrent transactions locking one hot parent row: batch child DML, shorten transactions holding key-share locks, or restructure so the shared parent row is not the contention point. Fewer concurrent lockers per row means fewer/shorter MultiXacts.
- **Keep member space small with vacuum.** The GUCs that govern MultiXact freezing and their apply scopes:

| Setting | Context | Apply scope | Default |
|---|---|---|---|
| `vacuum_multixact_freeze_min_age` | `PGC_USERSET` | session/transaction | 5000000 [guc.c#min_age](../../../raw/postgres-14/src/backend/utils/misc/guc.c#L2636-L2644) |
| `vacuum_multixact_freeze_table_age` | `PGC_USERSET` | session/transaction | 150000000 [guc.c#table_age](../../../raw/postgres-14/src/backend/utils/misc/guc.c#L2646-L2654) |
| `autovacuum_multixact_freeze_max_age` | `PGC_POSTMASTER` | restart | 400000000 [guc.c#max_age](../../../raw/postgres-14/src/backend/utils/misc/guc.c#L3291-L3300) |

Lowering `vacuum_multixact_freeze_min_age` / `vacuum_multixact_freeze_table_age` (session-scoped: set in a session or via `ALTER SYSTEM`/reload; no restart) makes vacuum freeze and reclaim old MultiXacts sooner. `autovacuum_multixact_freeze_max_age` is `PGC_POSTMASTER`, so **changing it requires a server restart** — the source marks it `PGC_POSTMASTER` deliberately [guc.c#max_age](../../../raw/postgres-14/src/backend/utils/misc/guc.c#L3291-L3300). Note that when member space is already high, `MultiXactMemberFreezeThreshold` overrides this value downward anyway [multixact.c#MemberFreezeThreshold](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L2964-L3000).

Autovacuum also has per-table storage parameters for the same MultiXact freeze family: `autovacuum_multixact_freeze_min_age`, `autovacuum_multixact_freeze_max_age`, and `autovacuum_multixact_freeze_table_age`, including `toast.` variants [create_table.sgml#autovacuum-multixact-reloptions](../../../raw/postgres-14/doc/src/sgml/ref/create_table.sgml#L1704-L1751). These are relation options, not GUCs; `reloptions.c` defines them for heap and TOAST relations with `ShareUpdateExclusiveLock`, so changing them is an `ALTER TABLE ... SET (...)` metadata change, not a server restart or reload [reloptions.c#autovacuum-multixact-reloptions](../../../raw/postgres-14/src/backend/access/common/reloptions.c#L263-L304). Autovacuum extracts those reloptions from `pg_class.reloptions` and uses them instead of the defaults when they are set [autovacuum.c#extract_autovac_opts](../../../raw/postgres-14/src/backend/postmaster/autovacuum.c#L2787-L2812) [autovacuum.c#multixact-relopts](../../../raw/postgres-14/src/backend/postmaster/autovacuum.c#L2954-L2971). The per-table max age can only lower the effective systemwide maximum [autovacuum.c#multixact-max-relopt](../../../raw/postgres-14/src/backend/postmaster/autovacuum.c#L3181-L3187).

## Context Reviewed

- `src/backend/access/transam/multixact.c` — the whole MultiXact manager: design overview, page/segment macros, member thresholds, `MultiXactStateData` and per-backend arrays, the local cache (`mXactCacheEnt`, `MAX_CACHE_ENTRIES`, get/put/evict), `MultiXactIdCreate`/`Expand`/`CreateFromMembers`, `GetNewMultiXactId`, `RecordNewMultiXact`, `GetMultiXactIdMembers`, `MultiXactMemberFreezeThreshold`, `pg_get_multixact_members`, and `MultiXactShmemSize`/`Init`.
- `src/include/access/multixact.h` — `MultiXactId`/`MultiXactOffset` constants, SLRU buffer counts, `MultiXactStatus`, `MultiXactMember`.
- `src/backend/access/transam/slru.c` and its header comment — the SLRU pool model, `SimpleLruReadPage`, `SimpleLruReadPage_ReadOnly`, `SlruSelectLRUPage`, `SlruPhysicalReadPage`, `SlruInternalWritePage`, `SimpleLruWaitIO`.
- `src/backend/access/heap/heapam.c` — `compute_new_xmax_infomask` (the create/expand decision), `heap_lock_tuple`, `heap_update`, `heap_delete`, and `FreezeMultiXactId`.
- `src/backend/access/heap/heapam_handler.c` — `heapam_tuple_lock` routing to `heap_lock_tuple`.
- `src/backend/utils/adt/ri_triggers.c` — the FK check/restrict queries that take `FOR KEY SHARE`.
- `src/backend/utils/misc/guc.c` — the three MultiXact vacuum GUCs and their contexts.
- `src/backend/access/common/reloptions.c`, `src/backend/postmaster/autovacuum.c`, `doc/src/sgml/ref/create_table.sgml` — per-table autovacuum MultiXact freeze storage parameters and their use.
- `doc/src/sgml/mvcc.sgml` and `src/backend/executor/nodeLockRows.c` — row-lock compatibility and explicit row-lock execution.
- `src/backend/storage/lmgr/lwlocknames.txt`, `src/backend/storage/lmgr/lwlock.c` — the MultiXact lock and buffer-tranche wait-event names.
- `src/backend/postmaster/pgstat.c`, `src/backend/catalog/system_views.sql`, `src/backend/utils/activity/wait_event.c`, `src/include/catalog/pg_proc.dat` — `pg_stat_slru`, SLRU names, `pg_stat_activity` wait columns, the `SLRURead` name, and `mxid_age`.
- `src/test/isolation/specs/fk-contention.spec` — a FK-vs-update row-lock contention scenario.
- `doc/src/sgml/maintenance.sgml` — the "Multixacts and Wraparound" section.

## Evidence Map

| Claim | Citation |
|---|---|
| MultiXact stores a per-MXID array of members; fundamental to shared row locks; two SLRUs | [multixact.c#overview](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L6-L56) |
| MultiXactId is a 32-bit counter; member = `(xid, status)`; six lock statuses | [multixact.h#Ids](../../../raw/postgres-14/src/include/access/multixact.h#L24-L30), [multixact.h#MultiXactMember](../../../raw/postgres-14/src/include/access/multixact.h#L60-L64), [multixact.h#MultiXactStatus](../../../raw/postgres-14/src/include/access/multixact.h#L41-L51) |
| Only the MXID is in `xmax`; members kept in `pg_multixact` | [maintenance.sgml#Multixacts](../../../raw/postgres-14/doc/src/sgml/maintenance.sgml#L729-L748) |
| Two SLRUs (offsets, members) backed by `pg_multixact/offsets` and `pg_multixact/members` | [multixact.c#two-SLRUs](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L19-L25), [multixact.c#SimpleLruInit](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1982-L1993) |
| v14 SLRU pools are 8 offset / 16 member buffers, allocated at startup | [multixact.h#buffers](../../../raw/postgres-14/src/include/access/multixact.h#L32-L34), [multixact.c#ShmemSize](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1965-L1967) |
| Shared state + anti-wraparound limits + per-backend oldest arrays | [multixact.c#MultiXactStateData](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L200-L234), [multixact.c#perBackend](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L236-L281) |
| Create/expand path; expand builds a new MXID, never mutates | [multixact.c#MultiXactIdCreate](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L398-L430), [multixact.c#MultiXactIdExpand](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L451-L495) |
| CreateFromMembers: cache reuse, GetNew, WAL, RecordNew, cachePut | [multixact.c#reuse](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L789-L804), [multixact.c#xlog-record](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L837-L861) |
| RecordNewMultiXact writes offsets then members under exclusive SLRU locks | [multixact.c#RecordNewMultiXact-members](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1029-L1071) |
| Member space (`nextOffset`) advances monotonically | [multixact.c#advance](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1322-L1324) |
| Read path: cache hit shortcut; offsets then members SLRU reads under exclusive lock; cachePut | [multixact.c#GetMultiXactIdMembers](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1386-L1580) |
| MultiXact reads enter `SimpleLruReadPage` under exclusive control locks (no ReadOnly shared fast path), while physical I/O releases that control lock | [slru.c#ReadOnly](../../../raw/postgres-14/src/backend/access/transam/slru.c#L494-L525), [multixact.c#offset-read](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1472-L1529), [slru.c#SimpleLruReadPage-IO](../../../raw/postgres-14/src/backend/access/transam/slru.c#L439-L457) |
| Local cache purpose, entry struct, list | [multixact.c#cache-purpose](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L296-L322) |
| Cache lives in a TopTransactionContext child → dropped at xact end | [multixact.c#MXactContext](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1721-L1728) |
| 256-entry cap; LRU tail eviction on overflow | [multixact.c#mXactCachePut](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1713-L1758) |
| FK check runs `SELECT ... FOR KEY SHARE OF x`; parent opened RowShareLock | [ri_triggers.c#RI_FKey_check](../../../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L358-L389), [ri_triggers.c#RowShareLock](../../../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L266-L268) |
| RI restrict/no-action checks query the child table with `FOR KEY SHARE` | [ri_triggers.c#ri_restrict](../../../raw/postgres-14/src/backend/utils/adt/ri_triggers.c#L617-L709) |
| Row lock routes executor → `heapam_tuple_lock` → `heap_lock_tuple` → `compute_new_xmax_infomask` | [heapam_handler.c#heapam_tuple_lock](../../../raw/postgres-14/src/backend/access/heap/heapam_handler.c#L365-L367), [heapam.c#heap_lock_tuple](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L5102-L5111) |
| MXID created/expanded when a second live locker or locker+updater hits one row | [heapam.c#compute_new_xmax_infomask](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L5333-L5495) |
| FK contention scenario in-tree | [fk-contention.spec](../../../raw/postgres-14/src/test/isolation/specs/fk-contention.spec#L1-L19) |
| Explicit row-lock clauses route through `LockRows`; compatible locks can coexist, conflicting locks wait | [nodeLockRows.c#ExecLockRows](../../../raw/postgres-14/src/backend/executor/nodeLockRows.c#L160-L190), [mvcc.sgml#row-lock-compatibility](../../../raw/postgres-14/doc/src/sgml/mvcc.sgml#L1260-L1307), [mvcc.sgml#row-lock-table](../../../raw/postgres-14/doc/src/sgml/mvcc.sgml#L1321-L1355) |
| UPDATE/DELETE against locked rows also create MXIDs | [heapam.c#heap_delete](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L2998-L3000), [heapam.c#heap_update](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L3694-L3811) |
| VACUUM freezing rebuilds partially-running MXIDs (consumes member space) | [heapam.c#FreezeMultiXactId](../../../raw/postgres-14/src/backend/access/heap/heapam.c#L6967-L6969) |
| Cache miss → SLRU buffer miss → physical `pg_pread` reported as `SLRURead` | [slru.c#SimpleLruReadPage](../../../raw/postgres-14/src/backend/access/transam/slru.c#L434-L451), [slru.c#SlruPhysicalReadPage](../../../raw/postgres-14/src/backend/access/transam/slru.c#L719-L729) |
| MultiXact SLRU lock names; buffer tranche names | [lwlocknames.txt](../../../raw/postgres-14/src/backend/storage/lmgr/lwlocknames.txt#L21-L23), [lwlock.c#tranches](../../../raw/postgres-14/src/backend/storage/lmgr/lwlock.c#L138-L140) |
| SLRU physical read/write I/O releases the control lock and uses per-buffer locks | [slru.c#SimpleLruReadPage-IO](../../../raw/postgres-14/src/backend/access/transam/slru.c#L439-L457), [slru.c#SlruInternalWritePage](../../../raw/postgres-14/src/backend/access/transam/slru.c#L539-L590) |
| Victim search stalls when all pages are I/O-busy; dirty victims go through `SlruInternalWritePage` | [slru.c#all-busy](../../../raw/postgres-14/src/backend/access/transam/slru.c#L1117-L1128), [slru.c#write-victim](../../../raw/postgres-14/src/backend/access/transam/slru.c#L1130-L1139) |
| Waiters block in `SimpleLruWaitIO` | [slru.c#SimpleLruWaitIO](../../../raw/postgres-14/src/backend/access/transam/slru.c#L341-L349) |
| Member-space guards: autovac trigger, warning, `members limit exceeded` ERROR | [multixact.c#autovac-trigger](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1266-L1285), [multixact.c#members-limit](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L1246-L1264) |
| Member pressure clamps effective freeze max age downward | [multixact.c#MemberFreezeThreshold](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L2964-L3000) |
| Member-storage thresholds (10 GB / 20 GB / 40M warn / 3M stop) | [maintenance.sgml#members-storage](../../../raw/postgres-14/doc/src/sgml/maintenance.sgml#L774-L791) |
| `pg_stat_slru` columns and MultiXact SLRU names | [system_views.sql#pg_stat_slru](../../../raw/postgres-14/src/backend/catalog/system_views.sql#L866-L877), [pgstat.c#slru_names](../../../raw/postgres-14/src/backend/postmaster/pgstat.c#L146-L157) |
| `pg_stat_activity` wait columns; `SLRURead` name | [system_views.sql#pg_stat_activity](../../../raw/postgres-14/src/backend/catalog/system_views.sql#L828-L829), [wait_event.c#SLRURead](../../../raw/postgres-14/src/backend/utils/activity/wait_event.c#L645-L646) |
| `mxid_age()` and `pg_get_multixact_members()` | [pg_proc.dat#mxid_age](../../../raw/postgres-14/src/include/catalog/pg_proc.dat#L2377-L2378), [multixact.c#pg_get_multixact_members](../../../raw/postgres-14/src/backend/access/transam/multixact.c#L3524-L3553) |
| GUC contexts: two `PGC_USERSET`, one `PGC_POSTMASTER` (restart) | [guc.c#min_age](../../../raw/postgres-14/src/backend/utils/misc/guc.c#L2636-L2644), [guc.c#table_age](../../../raw/postgres-14/src/backend/utils/misc/guc.c#L2646-L2654), [guc.c#max_age](../../../raw/postgres-14/src/backend/utils/misc/guc.c#L3291-L3300) |
| Per-table autovacuum MultiXact storage parameters exist and are used by autovacuum | [create_table.sgml#autovacuum-multixact-reloptions](../../../raw/postgres-14/doc/src/sgml/ref/create_table.sgml#L1704-L1751), [reloptions.c#autovacuum-multixact-reloptions](../../../raw/postgres-14/src/backend/access/common/reloptions.c#L263-L304), [autovacuum.c#multixact-relopts](../../../raw/postgres-14/src/backend/postmaster/autovacuum.c#L2954-L2971), [autovacuum.c#multixact-max-relopt](../../../raw/postgres-14/src/backend/postmaster/autovacuum.c#L3181-L3187) |

## Open Questions

- This page documents the mechanisms and the source-evident performance consequences of MultiXact traffic in PostgreSQL 14, but it does **not** include a quantified benchmark (MultiXacts/sec, `SLRURead` latency, or lock-wait time as a function of concurrency). The in-tree `fk-contention` isolation test exercises correctness of the FK/update interaction, not performance [fk-contention.spec](../../../raw/postgres-14/src/test/isolation/specs/fk-contention.spec); no in-tree benchmark quantifying MultiXact SLRU overhead was located.
- The exact set and minor-release provenance of MultiXact/SLRU fixes within the 14.x series (the checkout is pinned at `REL_14_23-3-g5c00f4e2e3b`) has not been traced here. This page describes behavior at the pin, not its change history within v14.

## Source References

- [src/backend/access/transam/multixact.c](../../../raw/postgres-14/src/backend/access/transam/multixact.c) — MultiXact manager: design overview, `MultiXactStateData`, the local cache (`mXactCacheEnt`/`MAX_CACHE_ENTRIES`/`mXactCacheGetById`/`mXactCacheGetBySet`/`mXactCachePut`), `MultiXactIdCreate`/`Expand`/`CreateFromMembers`, `GetNewMultiXactId`, `RecordNewMultiXact`, `GetMultiXactIdMembers`, `MultiXactMemberFreezeThreshold`, `pg_get_multixact_members`, `MultiXactShmemSize`/`Init`.
- [src/include/access/multixact.h](../../../raw/postgres-14/src/include/access/multixact.h) — ID/offset constants, `NUM_MULTIXACTOFFSET_BUFFERS`/`NUM_MULTIXACTMEMBER_BUFFERS`, `MultiXactStatus`, `MultiXactMember`.
- [src/backend/access/transam/slru.c](../../../raw/postgres-14/src/backend/access/transam/slru.c) — SLRU pool model, `SimpleLruReadPage`, `SimpleLruReadPage_ReadOnly`, `SlruSelectLRUPage`, `SlruPhysicalReadPage`, `SlruInternalWritePage`, `SimpleLruWaitIO`.
- [src/backend/access/heap/heapam.c](../../../raw/postgres-14/src/backend/access/heap/heapam.c) — `compute_new_xmax_infomask`, `heap_lock_tuple`, `heap_update`, `heap_delete`, `FreezeMultiXactId`.
- [src/backend/access/heap/heapam_handler.c](../../../raw/postgres-14/src/backend/access/heap/heapam_handler.c) — `heapam_tuple_lock` → `heap_lock_tuple`.
- [src/backend/utils/adt/ri_triggers.c](../../../raw/postgres-14/src/backend/utils/adt/ri_triggers.c) — FK check/restrict queries taking `FOR KEY SHARE`.
- [src/backend/utils/misc/guc.c](../../../raw/postgres-14/src/backend/utils/misc/guc.c) — `vacuum_multixact_freeze_min_age`, `vacuum_multixact_freeze_table_age`, `autovacuum_multixact_freeze_max_age`.
- [src/backend/access/common/reloptions.c](../../../raw/postgres-14/src/backend/access/common/reloptions.c), [src/backend/postmaster/autovacuum.c](../../../raw/postgres-14/src/backend/postmaster/autovacuum.c), [doc/src/sgml/ref/create_table.sgml](../../../raw/postgres-14/doc/src/sgml/ref/create_table.sgml) — per-table autovacuum MultiXact freeze storage parameters and their use.
- [doc/src/sgml/mvcc.sgml](../../../raw/postgres-14/doc/src/sgml/mvcc.sgml), [src/backend/executor/nodeLockRows.c](../../../raw/postgres-14/src/backend/executor/nodeLockRows.c) — row-lock compatibility and explicit row-lock execution.
- [src/backend/storage/lmgr/lwlocknames.txt](../../../raw/postgres-14/src/backend/storage/lmgr/lwlocknames.txt), [src/backend/storage/lmgr/lwlock.c](../../../raw/postgres-14/src/backend/storage/lmgr/lwlock.c) — MultiXact lock and buffer-tranche names.
- [src/backend/postmaster/pgstat.c](../../../raw/postgres-14/src/backend/postmaster/pgstat.c), [src/backend/catalog/system_views.sql](../../../raw/postgres-14/src/backend/catalog/system_views.sql), [src/backend/utils/activity/wait_event.c](../../../raw/postgres-14/src/backend/utils/activity/wait_event.c), [src/include/catalog/pg_proc.dat](../../../raw/postgres-14/src/include/catalog/pg_proc.dat) — `pg_stat_slru`, SLRU names, `pg_stat_activity` wait columns, `SLRURead`, `mxid_age`.
- [src/test/isolation/specs/fk-contention.spec](../../../raw/postgres-14/src/test/isolation/specs/fk-contention.spec) — FK/update row-lock contention scenario.
- [doc/src/sgml/maintenance.sgml](../../../raw/postgres-14/doc/src/sgml/maintenance.sgml) — "Multixacts and Wraparound".

## Navigation

- [v14/index](../index.md)
- [PostgreSQL 14 Codebase Navigation Guide](../codebase-navigation-guide.md)
- [Wiki Index](../../index.md)
- [Versions](../../versions.md)
