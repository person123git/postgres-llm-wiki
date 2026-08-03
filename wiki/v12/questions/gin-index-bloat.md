---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# How a GIN Index Becomes Bloated in PostgreSQL 12, and How to Measure It (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [The four page budgets inside a GIN index](#the-four-page-budgets-inside-a-gin-index)
  - [Mechanism 1: the pending list (fastupdate)](#mechanism-1-the-pending-list-fastupdate)
  - [Mechanism 2: the entry tree never gives a page back](#mechanism-2-the-entry-tree-never-gives-a-page-back)
  - [Mechanism 3: posting-tree pages split at 50% or 75%, not 100%](#mechanism-3-posting-tree-pages-split-at-50-or-75-not-100)
  - [Mechanism 4: VACUUM leaves slack inside posting-tree leaves](#mechanism-4-vacuum-leaves-slack-inside-posting-tree-leaves)
  - [Mechanism 5: deleted pages wait for an XID, not for a VACUUM](#mechanism-5-deleted-pages-wait-for-an-xid-not-for-a-vacuum)
  - [Mechanism 6: the file is never truncated](#mechanism-6-the-file-is-never-truncated)
  - [Write amplification is the driver](#write-amplification-is-the-driver)
  - [What VACUUM can and cannot recover](#what-vacuum-can-and-cannot-recover)
  - [Measuring: which tools accept a GIN index](#measuring-which-tools-accept-a-gin-index)
  - [Measurement recipe 1: pending-list backlog](#measurement-recipe-1-pending-list-backlog)
  - [Measurement recipe 2: per-page census](#measurement-recipe-2-per-page-census)
  - [Measurement recipe 3: metapage picture versus live size](#measurement-recipe-3-metapage-picture-versus-live-size)
  - [Measurement recipe 4: recyclable pages in the FSM](#measurement-recipe-4-recyclable-pages-in-the-fsm)
  - [Measurement recipe 5: ground truth by rebuilding](#measurement-recipe-5-ground-truth-by-rebuilding)
  - [How bloat reaches the planner](#how-bloat-reaches-the-planner)
  - [Settings, and their apply scope](#settings-and-their-apply-scope)
  - [Exact-pin measurements](#exact-pin-measurements)
  - [Key data structures](#key-data-structures)
  - [Caller and callee boundary](#caller-and-callee-boundary)
  - [Build, generated-header, and extension boundary](#build-generated-header-and-extension-boundary)
  - [Tests and explicit test absence](#tests-and-explicit-test-absence)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, how can a GIN index become bloated? How do you measure whether a GIN index is bloated?

## Answer

### Short answer

A GIN index bloats in six distinct ways. VACUUM can free pages for two of them; none of the six ever returns disk to the filesystem, so only `REINDEX` shrinks the file:

| # | Mechanism | Can VACUUM make the pages reusable? |
|---|---|---|
| 1 | Pending-list (`fastupdate`) pages accumulate | Yes |
| 2 | Entry-tree pages are never deleted, merged, or emptied | **Never** |
| 3 | Posting-tree leaves split to 50% (or 75% when appending), not 100% | No |
| 4 | VACUUM does not re-pack posting-tree leaves or merge siblings | No |
| 5 | Fully empty, non-edge posting-tree pages are deleted | Yes, but only once the XID counter passes their delete XID |
| 6 | The index file is never truncated | **Never** |

Measurement is harder than for B-tree, because v12 ships no GIN density function. `pgstatindex` and `pgstattuple` both refuse a GIN index. The tools that do work are `pgstatginindex` (pending list only), `pageinspect`'s three GIN functions (superuser, one page at a time), `pg_freespacemap`, and `pg_relation_size` compared against a freshly built copy. The v12 documentation itself concedes the gap: "The potential for bloat in non-B-tree indexes has not been well researched. It is a good idea to periodically monitor the index's physical size when using any non-B-tree index type." ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L876-L880))

### The four page budgets inside a GIN index

A GIN index is a metapage, a B-tree of key values (the *entry tree*), optional *posting trees* holding the heap TIDs for very common keys, and optional *pending-list* pages holding not-yet-merged entries ([README#Index structure](../../../raw/postgres-12/src/backend/access/gin/README#L95-L105)). Every page carries a `flags` word that names its class ([ginblock.h#GIN_DATA](../../../raw/postgres-12/src/include/access/ginblock.h#L40-L48)), which is what makes the census in [Measurement recipe 2](#measurement-recipe-2-per-page-census) possible.

The metapage is block 0 and the entry-tree root is block 1 ([ginblock.h#GIN_METAPAGE_BLKNO](../../../raw/postgres-12/src/include/access/ginblock.h#L50-L52)).

### Mechanism 1: the pending list (fastupdate)

`fastupdate` defaults to on ([gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../raw/postgres-12/src/include/access/gin_private.h#L31-L39)), so by default `gininsert` appends every new row's keys to an unsorted linear list of `GIN_LIST` pages instead of inserting them into the entry tree ([gininsert.c#gininsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L511-L531)).

Those pages accumulate until a trim fires. The trigger compares *page capacity*, not bytes actually used, against `gin_pending_list_limit`:

```c
	cleanupSize = GinGetPendingListCleanupSize(index);
	if (metadata->nPendingPages * GIN_PAGE_FREESIZE > cleanupSize * 1024L)
		needCleanup = true;
```

([ginfast.c#ginHeapTupleFastInsert-cleanup-trigger](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L461)) with `GIN_PAGE_FREESIZE` defined at [ginfast.c#GIN_PAGE_FREESIZE](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L40-L41). A half-empty pending page therefore counts at full weight toward the limit.

Three separate things keep the list longer than the limit suggests:

1. **Internal fragmentation.** A row whose entries do not fit on one pending page gets pages to itself, wasting the tail of the previous page. The README states it directly: "a heap tuple whose entries do not all fit on one pending-list page must have those pages to itself, even if this results in wasting much of the space on the preceding page and the last page for the tuple" ([README#GIN_LIST_FULLROW](../../../raw/postgres-12/src/backend/access/gin/README#L206-L216)). The abandoned-tail path is the `separateList` branch in [ginfast.c#ginHeapTupleFastInsert-separateList](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L251-L275), and `makeSublist` allocates fresh pages per heap tuple ([ginfast.c#makeSublist](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L143-L208)).
2. **A backend gives up on contention.** An insert-driven cleanup uses `ConditionalLockPage` and simply returns if another cleanup holds the lock ([ginfast.c#ginInsertCleanup-locking](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L800-L828)).
3. **Cleanup stops at the tail it remembered on entry** unless the caller asked for a full clean ([ginfast.c#ginInsertCleanup-blknoFinish](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L843-L847), [ginfast.c#ginInsertCleanup-cleanupFinish](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L881-L888)).

Autovacuum is asymmetric here. Both VACUUM entry points pass `full_clean = !IsAutoVacuumWorkerProcess()`, so an autovacuum worker never does a full clean, while a manual `VACUUM` does ([ginvacuum.c#ginbulkdelete-pending-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L583-L594), [ginvacuum.c#ginvacuumcleanup-pending-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L711-L721)). An autovacuum *analyze* also cleans the pending list, while a plain manual `ANALYZE` is a no-op ([ginvacuum.c#ginvacuumcleanup-analyze_only](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L697-L709)).

`gin_clean_pending_list(regclass)` is the only way to force a full clean on demand. It requires index ownership, refuses to run during recovery, and returns the number of pending pages deleted ([ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1030-L1074)).

Trimmed pending pages get the `GIN_DELETED` flag and, when `fill_fsm` is set, go straight to the free space map ([ginfast.c#shiftList-flags](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L632), [ginfast.c#shiftList-RecordFreeIndexPage](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L659-L665)). They become reusable pages inside the same file; the file does not shrink.

One operational trap: turning `fastupdate` off does not flush what is already pending. The documentation says so ([create_index.sgml#fastupdate](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L459-L467)), and the exact-pin run below confirms it.

### Mechanism 2: the entry tree never gives a page back

This is the mechanism with no VACUUM remedy, and it is by design. The README says it three times:

- "There is no delete operation in the key (entry) tree. The reason for this is that in our experience, the set of distinct words in a large corpus changes very slowly." ([README#no-entry-delete](../../../raw/postgres-12/src/backend/access/gin/README#L28-L31))
- "That works because tuples are never deleted from the entry tree." ([README#entry-high-key](../../../raw/postgres-12/src/backend/access/gin/README#L311-L314))
- "Vacuum never deletes tuples or pages from the entry tree." ([README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396))

The code matches. `ginVacuumEntryPage` strips dead TIDs out of an entry tuple's in-line posting list and rewrites the tuple in place. When the posting list becomes empty it builds the tuple with a null posting list rather than removing it:

```c
				if (nitems > 0)
				{
					plist = ginCompressPostingList(items, nitems, GinMaxItemSize, NULL);
					plistsize = SizeOfGinPostingList(plist);
				}
				else
				{
					plist = NULL;
					plistsize = 0;
				}
```

([ginvacuum.c#ginVacuumEntryPage](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556)); `GinFormTuple` accepts `nipd = 0` and returns a valid tuple ([ginentrypage.c#GinFormTuple](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L44-L154)). `GinPageSetDeleted` is only ever applied to `GIN_DATA` pages, guarded by an assertion in the deletion scan ([ginvacuum.c#ginDeletePage-SetDeleted](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L182-L192), [ginvacuum.c:280](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L280)).

Consequence: a key that once existed keeps an entry-tuple slot forever, and the entry-tree page holding it is never freed. A workload that cycles through distinct keys — UUIDs, timestamps rendered as text, per-request trigram sets — grows the entry tree monotonically until `REINDEX`.

Entry-tree splits also never pack tighter than half. `entrySplitPage` equalizes byte size with no build-time or rightmost-append special case ([ginentrypage.c#entrySplitPage](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L666-L689)), and `entryIsEnoughSpace` reserves no fillfactor ([ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L459-L483)). GIN exposes no `fillfactor` reloption at all; its only two are `fastupdate` and `gin_pending_list_limit` ([ginutil.c#ginoptions](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L629)).

### Mechanism 3: posting-tree pages split at 50% or 75%, not 100%

When a key accumulates more TIDs than fit in an entry tuple, the TIDs move to a posting tree ([gininsert.c#addItemPointersToLeafTuple](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L42-L119)). How full a new posting-tree leaf starts depends entirely on whether the insert came from a build or from ordinary DML:

```c
		 * leafRepackItems already divided the segments between the left and
		 * the right page. It filled the left page as full as possible, and
		 * put the rest to the right page. When building a new index, that's
		 * good, ... But otherwise, split
		 * 50/50, by moving segments from the left page to the right page
		 * until they're balanced.
		 *
		 * As a further heuristic, when appending items to the end of the
		 * page, try to make the left page 75% full, ...
```

([gindatapage.c#dataBeginPlaceToPageLeaf-split](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L617-L667)). Internal posting-tree pages behave the same way: full packing only when `btree->isBuild` and the page is rightmost, otherwise an exact 50/50 split ([gindatapage.c#dataSplitPageInternal](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L1284-L1294)).

`btree->isBuild` is true only for build-time inserts, which is why `CREATE INDEX` and `REINDEX` produce a materially denser index than the same rows inserted retail ([gininsert.c#ginbuild](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L378-L384)). The v12 documentation gives the same advice without the mechanism: "for bulk insertions into a table it is advisable to drop the GIN index and recreate it after finishing bulk insertion" ([gin.sgml#gin-tips](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L535-L550)).

### Mechanism 4: VACUUM leaves slack inside posting-tree leaves

`ginVacuumPostingTreeLeaf` removes dead TIDs from each compressed segment and rewrites the page, but deliberately does not re-encode the now-sparse segments:

```c
	 * We don't try to re-encode the segments here, even though some of them
	 * might be really small now that we've removed some items from them. It
	 * seems like a waste of effort, ...
	 * ... You'll have to REINDEX anyway if you want the full gain of the
	 * new tighter index format.
```

([gindatapage.c#ginVacuumPostingTreeLeaf-no-reencode](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L794-L810))

There is also no sibling merging anywhere in GIN. A posting-tree page is deleted only when it is *completely* empty, and never if it is the left- or rightmost branch:

```c
	if (isempty)
	{
		/* we never delete the left- or rightmost branch */
		if (BufferIsValid(me->leftBuffer) && !GinPageRightMost(page))
```

([ginvacuum.c#ginScanToDelete](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L302-L317)). A leaf holding one surviving TID out of 20,000 keeps its whole page. The second deletion pass only runs at all if the first pass found at least one fully empty leaf ([ginvacuum.c#ginVacuumPostingTree](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L407-L447)).

Finally, a key promoted to a posting tree is never demoted back to an in-line posting list, so its posting-tree root page survives even when nearly all its TIDs are gone.

### Mechanism 5: deleted pages wait for an XID, not for a VACUUM

`ginDeletePage` stamps the page with the *next* XID at deletion time:

```c
	GinPageSetDeleted(page);
	GinPageSetDeleteXid(page, ReadNewTransactionId());
```

([ginvacuum.c#ginDeletePage-deleteXid](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L182-L192)), and recyclability requires that XID to be strictly older than `RecentGlobalXmin`:

```c
#define GinPageIsRecyclable(page) ( PageIsNew(page) || (GinPageIsDeleted(page) \
	&& TransactionIdPrecedes(GinPageGetDeleteXid(page), RecentGlobalXmin)))
```

([ginblock.h#GinPageIsRecyclable](../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138)). The README explains why the wait exists: a concurrent reader may already hold a downlink to the page, so "deleted page cannot be reclaimed immediately" ([README#downlink-reclaim](../../../raw/postgres-12/src/backend/access/gin/README#L427-L433)).

`ginvacuumcleanup` is the only place that hands a GIN page to the free space map, and it does so only for recyclable pages ([ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L777)). The practical effect, measured below, is *not* "a second VACUUM frees them": with an idle XID counter, VACUUM #2 and #3 freed nothing. What is required is a VACUUM that runs after the global xmin has moved past the delete XID.

### Mechanism 6: the file is never truncated

No GIN code path truncates the relation. The only storage calls in `src/backend/access/gin/` are `RecordFreeIndexPage` ([ginvacuum.c:761](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L761), [ginfast.c:665](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L665)), `IndexFreeSpaceMapVacuum` ([ginvacuum.c:784](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L784), [ginfast.c#ginInsertCleanup-fsm](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1014-L1020)), and `GetFreeIndexPage` ([ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)). `ginvacuumcleanup` merely *reports* the current length ([ginvacuum.c#ginvacuumcleanup-num_pages](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L788-L792)). VACUUM's truncation logic is heap-only ([vacuumlazy.c#lazy_truncate_heap-RelationTruncate](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1962-L1965)).

So a GIN index's high-water mark is permanent for the life of that relfilenode. Freed pages are reused through `GinNewBuffer`, which pulls from the FSM before extending ([ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)) — and because it is one shared pool, pending-list churn and posting-tree churn compete for the same free pages.

### Write amplification is the driver

GIN inserts one index entry per *distinct extracted key* per row ([gininsert.c#ginHeapTupleInsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L462-L482), dedup at [ginutil.c#ginExtractEntries](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L535-L590)). Even a NULL or zero-key item gets a placeholder entry ([ginutil.c#ginExtractEntries-placeholders](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L492-L526)).

An `UPDATE` that touches the indexed column cannot be HOT, so it re-inserts every key of the new row version while the old version's keys stay in the index until VACUUM removes the dead TIDs ([heapam.c#heap_update-hot-check](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3593-L3600), [heapam_handler.c#update_indexes](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L336-L344)). The documentation states the amplification: "inserting or updating one heap row can cause many inserts into the index (one for each key extracted from the indexed item)" ([gin.sgml#gin-fast-update](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L473-L489)).

### What VACUUM can and cannot recover

| Page class | VACUUM deletes it? | Returns to FSM? | File shrinks? |
|---|---|---|---|
| Pending list (`GIN_LIST`) | Yes, on full/partial clean | Yes | No |
| Posting-tree leaf, fully empty, not left/rightmost | Yes | Only once XID passes | No |
| Posting-tree leaf, partly empty | No | No | No |
| Posting-tree leaf, leftmost or rightmost | No | No | No |
| Posting-tree internal, fully empty, not left/rightmost | Yes | Only once XID passes | No |
| Posting-tree root | No | No | No |
| Entry-tree leaf or internal | **No** | **No** | No |
| Metapage | No | No | No |

Only `REINDEX` (or `REINDEX CONCURRENTLY`) resets the file, because it builds a fresh relfilenode with `btree->isBuild` packing.

### Measuring: which tools accept a GIN index

| Tool | GIN? | What you get |
|---|---|---|
| `pg_relation_size` / `pg_indexes_size` | Yes | on-disk bytes ([dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)) |
| `pgstattuple.pg_relpages` | Yes | live block count ([pgstatindex.c#check_relation_relkind](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L754-L770)) |
| `pgstattuple.pgstatginindex` | Yes | **only** `version`, `pending_pages`, `pending_tuples` ([pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573)) |
| `pgstattuple.pgstatindex` | **No** | `relation "%s" is not a btree index` ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L228)) |
| `pgstattuple.pgstattuple` | **No** | `"%s" (gin index) is not supported` ([pgstattuple.c#pgstat_relation](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276), [pgstattuple.c#pgstat_relation-ereport](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L308-L311)) |
| `pgstattuple.pgstattuple_approx` | **No** | `"%s" is not a table or materialized view` ([pgstatapprox.c#pgstattuple_approx_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L280-L290)) |
| `pageinspect.gin_metapage_info` | Yes, superuser | all ten metapage fields ([ginfuncs.c#gin_metapage_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L34-L87)) |
| `pageinspect.gin_page_opaque_info` | Yes, superuser | `rightlink`, `maxoff`, decoded `flags[]` ([ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L90-L154)) |
| `pageinspect.gin_leafpage_items` | Compressed data leaves only | posting-list segments ([ginfuncs.c#gin_leafpage_items](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L163-L265)) |
| `pg_freespacemap.pg_freespace` | Yes | free/used, effectively binary ([pg_freespacemap--1.1.sql:7-20](../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L7-L20)) |
| `contrib/amcheck` | **No** | `only B-Tree indexes are supported as targets for verification` ([verify_nbtree.c#btree_index_checkable](../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323)) |

Two consequences worth stating plainly. First, `pgstatginindex` reads exactly one buffer — the metapage — so it can never report size, density, or free space, even though the metapage carries `nTotalPages`/`nEntryPages`/`nDataPages`/`nEntries` that it declines to expose ([pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L541-L553)). Second, `pg_freespace` on an index is binary by construction: `RecordFreeIndexPage` writes `BLCKSZ - 1`, which saturates the FSM category ([indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55)), and the documentation warns "For indexes, what is tracked is entirely-unused pages ... the values are not meaningful, just whether a page is full or empty" ([pgfreespacemap.sgml#index-caveat](../../../raw/postgres-12/doc/src/sgml/pgfreespacemap.sgml#L66-L70)).

`pg_stat_all_indexes` is an access counter, not a size tool, and one of its columns is structurally dead for GIN: `idx_tup_fetch` is only incremented on the `amgettuple` path ([indexam.c#index_fetch_heap](../../../raw/postgres-12/src/backend/access/index/indexam.c#L565-L589)), and GIN sets `amgettuple = NULL` ([ginutil.c#ginhandler](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L71-L72)). `idx_scan` and `idx_tup_read` do work ([ginscan.c:409](../../../raw/postgres-12/src/backend/access/gin/ginscan.c#L409), [indexam.c#index_getbitmap](../../../raw/postgres-12/src/backend/access/index/indexam.c#L651-L670)). The documentation describes the same exclusion ([monitoring.sgml#idx_tup_fetch](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2914-L2929)).

### Measurement recipe 1: pending-list backlog

Cheap, non-superuser (`pg_stat_scan_tables` suffices, granted in [pgstattuple--1.4--1.5.sql:56-57](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L56-L57)), and always current. Run this first: a large pending list is the one form of GIN bloat you can fix in seconds.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';

WITH gin_indexes AS MATERIALIZED (
  SELECT c.oid AS idxoid,
         n.nspname AS schema_name,
         c.relname AS index_name
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'i'
    AND c.relam = (SELECT oid FROM pg_am WHERE amname = 'gin')
    AND c.relpersistence <> 't'
)
SELECT /* wiki_gin_pending_list_backlog */
       g.schema_name,
       g.index_name,
       pg_size_pretty(pg_relation_size(g.idxoid)) AS index_size,
       s.pending_pages,
       s.pending_tuples,
       pg_size_pretty(s.pending_pages::bigint * current_setting('block_size')::bigint)
         AS pending_bytes
FROM gin_indexes g
CROSS JOIN LATERAL pgstatginindex(g.idxoid) AS s
ORDER BY s.pending_pages DESC, g.index_name;
```

`AS MATERIALIZED` is load-bearing: it keeps the planner from inlining the CTE and calling `pgstatginindex` on a non-GIN index, which aborts the whole query with the error quoted above. `relpersistence <> 't'` avoids the other-sessions temp-index error ([pgstatindex.c#pgstatginindex_internal-temp](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L531-L539)). At the pin this returned only the nine GIN indexes in a database that also held B-tree, hash, GiST, and BRIN indexes.

Remedy when `pending_pages` is large: `SELECT gin_clean_pending_list('idx');` (index owner, not during recovery).

### Measurement recipe 2: per-page census

This is the only way in v12 to see where a GIN index's bytes actually went. It needs superuser, because `get_raw_page` gates on `superuser()` rather than a role ([rawpage.c#get_raw_page_internal](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106)), and it takes an `AccessShareLock` per call.

```sql
SET statement_timeout = '60s';
SET lock_timeout = '5s';

SELECT /* wiki_gin_page_census */
       CASE
         WHEN 'deleted' = ANY (o.flags) THEN 'deleted (awaiting recycle)'
         WHEN 'meta'    = ANY (o.flags) THEN 'metapage'
         WHEN 'list'    = ANY (o.flags) THEN 'pending list'
         WHEN 'data'    = ANY (o.flags) AND 'leaf' = ANY (o.flags) THEN 'posting-tree leaf'
         WHEN 'data'    = ANY (o.flags) THEN 'posting-tree internal'
         WHEN 'leaf'    = ANY (o.flags) THEN 'entry-tree leaf'
         ELSE 'entry-tree internal'
       END AS page_class,
       count(*) AS pages,
       pg_size_pretty(count(*) * current_setting('block_size')::bigint) AS bytes
FROM generate_series(0,
       (pg_relation_size('mixed_gin_b') / current_setting('block_size')::bigint) - 1
     ) AS b(blkno)
CROSS JOIN LATERAL gin_page_opaque_info(get_raw_page('mixed_gin_b', b.blkno::int)) AS o
GROUP BY 1
ORDER BY 2 DESC;
```

Read it this way:

- A large **pending list** share means recipe 1's remedy applies.
- A large **deleted** share means pages are waiting on `RecentGlobalXmin`, so a later VACUUM will make them reusable — but the file still will not shrink.
- A large **entry-tree** share against a small live key count is unrecoverable bloat; only `REINDEX` fixes it.
- **`deleted` is checked first on purpose.** `GinPageSetDeleted` uses `|=`, so a deleted posting-tree page still has `GIN_DATA` set ([ginblock.h#GinPageSetDeleted](../../../raw/postgres-12/src/include/access/ginblock.h#L124-L126)); testing `data` first would misclassify it.

Two caveats on this recipe. `gin_page_opaque_info` performs no page-type validation whatsoever — it casts the special pointer unconditionally ([ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L109-L111)) — so point it only at a GIN index. And `maxoff` is not an entry-tuple count: the struct comment scopes it to `GIN_DATA & ~GIN_LEAF` pages and `GIN_LIST` pages ([ginblock.h#GinPageOpaqueData](../../../raw/postgres-12/src/include/access/ginblock.h#L29-L36)), so it reads 0 on entry-tree leaves. Use the metapage `n_entries` for entry counts instead.

### Measurement recipe 3: metapage picture versus live size

The metapage's four planner counters are refreshed only by `CREATE INDEX`/`REINDEX` and by VACUUM's cleanup phase. `ginGetStats` says so: "nPendingPages can be trusted to be up-to-date, as can ginVersion; but the other fields are as of the last VACUUM" ([ginutil.c#ginGetStats](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L657); the only two writers are [gininsert.c#ginbuild-stats](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L402-L406) and [ginvacuum.c#ginvacuumcleanup-stats](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L781)).

Comparing the stale figure against the live block count is therefore itself a bloat signal — and a planner-accuracy signal, per [How bloat reaches the planner](#how-bloat-reaches-the-planner).

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';

SELECT /* wiki_gin_metapage_vs_live */
       m.n_total_pages   AS pages_at_last_vacuum,
       (pg_relation_size('mixed_gin_b') / current_setting('block_size')::bigint) AS pages_now,
       m.n_entry_pages,
       m.n_data_pages,
       m.n_entries,
       m.n_pending_pages,
       m.version
FROM gin_metapage_info(get_raw_page('mixed_gin_b', 0)) AS m;
```

`n_entry_pages + n_data_pages` need not equal `n_total_pages - 1`: the cleanup loop counts neither recyclable pages nor live `GIN_LIST` pages into either bucket, while `nTotalPages` is the whole file ([ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L777)). A `version` below 2 means a pre-9.4 index that still holds uncompressed posting lists and needs `REINDEX` for the tighter format ([ginblock.h#GinMetaPageData-version](../../../raw/postgres-12/src/include/access/ginblock.h#L84-L102)).

### Measurement recipe 4: recyclable pages in the FSM

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';

SELECT /* wiki_gin_free_pages */
       count(*) FILTER (WHERE f.avail > 0) AS free_pages,
       count(*)                            AS total_pages,
       round(100.0 * count(*) FILTER (WHERE f.avail > 0) / greatest(count(*), 1), 2)
         AS free_pct
FROM pg_freespace('mixed_gin_b') AS f;
```

`free_pages` counts pages a future insert can reuse without extending the file. A high `free_pct` means the index has already shed the churn and is sized for its high-water mark, not its current contents — `REINDEX` will shrink it. `VACUUM VERBOSE` reports the same number as "`%u` are currently reusable" ([vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827)).

### Measurement recipe 5: ground truth by rebuilding

Everything above is indirect. The only unambiguous answer to "how much is reclaimable" is to build a second index and compare.

```sql
SET statement_timeout = '10min';
SET lock_timeout = '5s';

CREATE INDEX CONCURRENTLY mixed_gin_b_probe ON mixed USING gin (b);

SELECT /* wiki_gin_rebuild_delta */
       pg_size_pretty(pg_relation_size('mixed_gin_b'))       AS current_size,
       pg_size_pretty(pg_relation_size('mixed_gin_b_probe')) AS rebuilt_size,
       round(100.0 * (pg_relation_size('mixed_gin_b') - pg_relation_size('mixed_gin_b_probe'))
             / greatest(pg_relation_size('mixed_gin_b'), 1), 2) AS reclaimable_pct;

DROP INDEX mixed_gin_b_probe;
```

This costs a full build and doubles the index's disk footprint while it runs. It is the measurement to reach for when recipes 1–4 disagree or when you need a number to justify a maintenance window.

### How bloat reaches the planner

GIN bloat is not only a disk problem; `gincostestimate` reads it twice.

**Pending pages are charged as startup I/O.** Every pending page is assumed read at scan start:

```c
	/*
	 * Compute cost to begin scan, first of all, pay attention to pending
	 * list.
	 */
	entryPagesFetched = numPendingPages;
```

([selfuncs.c#gincostestimate-pending](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6869-L6873)), then priced at `random_page_cost` ([selfuncs.c#gincostestimate-startup](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6922-L6926)). That charge is honest: every scan really does walk the pending list before the main index ([ginget.c#gingetbitmap-scanPendingInsert](../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1868-L1878)), which the documentation calls out as fast update's "main disadvantage" ([gin.sgml#gin-fast-update-disadvantage](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L491-L499)).

**Stale metapage stats can push the planner onto an invented model.** The stats are trusted only if the index has not grown more than 4x since the last VACUUM:

```c
	if (numPages > 0 && ginStats.nTotalPages <= numPages &&
		ginStats.nTotalPages > numPages / 4 &&
		ginStats.nEntryPages > 0 && ginStats.nEntries > 0)
```

([selfuncs.c#gincostestimate-stats-trust](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6716-L6750)). Otherwise it assumes 90% entry pages, 10% data pages, and 100 entries per entry page ([selfuncs.c#gincostestimate-fallback](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6751-L6770)). A never-vacuumed GIN index falls into that branch, as the exact-pin run below shows.

The block count itself is always live, never `pg_class.relpages`: `get_relation_info` calls `RelationGetNumberOfBlocks` for a non-partial index ([plancat.c#get_relation_info-index-size](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407)), and `estimate_rel_size` also uses the live count for a partial one ([plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L971)). `tree_height` stays `-1` for GIN ([plancat.c#get_relation_info-tree_height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L418)), so unlike B-tree there is no height charge.

### Settings, and their apply scope

| Setting | Kind | Apply scope |
|---|---|---|
| `gin_pending_list_limit` | GUC, `PGC_USERSET`, default 4096 kB, min 64 kB ([guc.c#gin_pending_list_limit](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184)) | **Session/transaction** — no restart, no reload |
| `fastupdate` | index reloption, default true ([reloptions.c#fastupdate](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L125-L133)) | `ALTER INDEX`, takes **`AccessExclusiveLock`** on the index |
| `gin_pending_list_limit` | index reloption, default −1 = inherit the GUC ([reloptions.c#gin_pending_list_limit](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L322-L330)) | `ALTER INDEX`, takes **`AccessExclusiveLock`** on the index |
| `maintenance_work_mem` / `autovacuum_work_mem` | GUC | Used by forced pending cleanup ([ginfast.c#ginInsertCleanup-workMemory](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L807-L817)) |
| `work_mem` | GUC | Used by insert-driven pending cleanup ([ginfast.c:827](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L827)) |

The reloption locks matter operationally: `ALTER INDEX ... SET (fastupdate = off)` blocks reads and writes on that index for its duration, and it does not flush the existing pending list.

`pg_settings` at the pin confirms the GUC scope:

```text
          name          | setting | unit | context | boot_val | min_val |  max_val
------------------------+---------+------+---------+----------+---------+------------
 gin_fuzzy_search_limit | 0       |      | user    | 0        | 0       | 2147483647
 gin_pending_list_limit | 4096    | kB   | user    | 4096     | 64      | 2147483647
```

`context = user` maps to session/transaction scope.

### Exact-pin measurements

All figures below come from an isolated PostgreSQL 12.2 server built from the pinned checkout, with `autovacuum = off`, `maintenance_work_mem = 64MB`, and `pgstattuple` / `pageinspect` / `pg_freespacemap` installed. `SELECT version()` reported `PostgreSQL 12.2 on x86_64-pc-linux-gnu`.

**Pending list, `fastupdate = on`, default limit, 200,000 rows of `ARRAY[g, g%7, g%13]`:**

| Measurement | Value |
|---|---|
| Index size | 12,722,176 bytes (1553 pages) |
| `pgstatginindex` | `version 2`, `pending_pages 411`, `pending_tuples 57022` |
| Census | 411 pending, 104 deleted, 972 entry leaf, 5 entry internal, 47 posting leaf, 13 posting internal, 1 meta |
| Metapage planner stats | `n_total_pages 2`, `n_entry_pages 1`, `n_data_pages 0`, `n_entries 0` |
| `gin_clean_pending_list` | returned 411 |
| Size after clean | 15,228,928 bytes, `pending_pages 0`, 411 deleted pages, FSM 411 free pages |
| Size after `REINDEX` | 11,804,672 bytes |

The metapage reporting 2 total pages for a 12 MB index is the stale-stats branch in the flesh: nothing had vacuumed the index, so the counters were still the build-time values. Note also that flushing the pending list *grew* the index by 2.5 MB, because the entries moved into the entry tree while the freed pending pages stayed allocated; `REINDEX` then cut 22.5% off the flushed size.

Every one of the 411 freed pages reported `avail = 8160`, the saturated FSM category, matching `RecordFreeIndexPage`'s `BLCKSZ - 1`.

**Entry tree, `fastupdate = off`, 300,000 rows each with one distinct key:**

| Step | Index size | Census | Metapage `n_entries` | FSM free |
|---|---|---|---|---|
| Populated + VACUUM | 16,801,792 B | 2040 entry leaf, 10 entry internal, 1 meta | 300000 | 0 |
| `DELETE` all 300,000 rows, VACUUM | 16,801,792 B | unchanged | 300000 | 0 |
| VACUUM again | 16,801,792 B | unchanged | 300000 | 0 |
| `REINDEX` | 16,384 B | 1 entry leaf, 1 meta | — | — |

An empty table kept a 16 MB index across two VACUUMs, and the metapage still counted 300,000 entries because the entry tuples were rewritten with empty posting lists rather than removed. `REINDEX` reclaimed 1025x.

**Posting trees, `fastupdate = off`, 800,000 rows over 40 keys:**

| Step | Index size | Deleted pages | Posting leaves | FSM free |
|---|---|---|---|---|
| Populated + VACUUM | 1,785,856 B | 0 | 160 | 0 |
| `DELETE` 760,000 (95%), VACUUM #1 | 1,785,856 B | 80 | 80 | 0 |
| VACUUM #2 | 1,785,856 B | 80 | 80 | 0 |
| VACUUM #3 | 1,785,856 B | 80 | 80 | 0 |
| `REINDEX` | 98,304 B | 0 | 0 | — |

Deleting 95% of the rows freed no bytes and no reusable pages. `REINDEX` shrank the index 18.2x, and the 40,000 surviving rows needed no posting trees at all.

**The recycle gate is the XID counter, not the VACUUM count.** Repeating the above on a fresh fixture:

| Step | Deleted pages | FSM free pages |
|---|---|---|
| VACUUM #1 (snapshot showed next XID 511) | 80 | 0 |
| VACUUM #2, no XIDs consumed in between | 80 | **0** |
| `txid_current()` x3 -> 511, 512, 513; VACUUM #3 | 80 | **80** (`max(avail) = 8160`) |

The delete XID stamped at VACUUM #1 was 511; only once the global xmin moved past it did `GinPageIsRecyclable` accept the pages. This corrects the intuitive "a second VACUUM frees them" reading: a second VACUUM on an otherwise idle cluster freed nothing.

Those freed pages are genuinely reused. Inserting 60,000 rows with new distinct keys consumed all 80 (FSM free 80 -> 0, `deleted` disappeared from the census) while the file grew from 218 to 547 pages — 329 by extension plus the 80 recycled.

**Retail insert versus build packing**, 800,000 rows over 40 keys inserted in random key order:

| | Index size | Posting-tree leaves |
|---|---|---|
| Grown by `INSERT` | 2,072,576 B | 194 |
| After `REINDEX` | 1,622,016 B | 156 |

27.8% larger before the rebuild, 24.4% more posting-tree leaves — the 50/50 versus build-time packing difference, with no dead rows involved at all.

**Planner effect of a pending list**, 300,000 rows, `gin_pending_list_limit = 1GB` on the index so the list was never trimmed:

| State | Live blocks | Pending pages | `Bitmap Index Scan` cost | Buffers | Exec time |
|---|---|---|---|---|---|
| Pending list intact | 1473 | 1471 | `0.00..5903.25` | 1473 | 9.793 ms |
| After `gin_clean_pending_list` | 3577 | 0 | `0.00..27.25` | 4 | 0.027 ms |
| After `VACUUM` | 3577 | 0 | `0.00..27.25` | 4 | 0.040 ms |

Both costs reproduce exactly from the source formula, in the invented-statistics branch (`n_total_pages` was 2 against 1473 and then 3577 live blocks, so `nTotalPages > numPages / 4` failed both times):

- Bloated: `numEntryPages = floor((1473-1471) * 0.90) = 1`, `numDataPages = 1`, `numEntries = 100`; `entryPagesFetched = 1471 + ceil(1 * rint(1^0.15)) = 1472`; startup `1472 * 4 = 5888`; `dataPagesFetched = 1`; total `5888 + 4 + 1500 * (0.005 + 0.0025) = 5903.25`.
- Clean: `numEntryPages = floor(3577 * 0.90) = 3219`; `entryPagesFetched = ceil(rint(3219^0.15)) = 3`; startup `12`; `dataPagesFetched = 1`; total `12 + 4 + 11.25 = 27.25`.

The VACUUM then refreshed the metapage to `n_total_pages 3577`, `n_entry_pages 2050`, `n_data_pages 55`, `n_entries 300001`, and the trusted-stats branch produced the same 27.25.

**`ALTER INDEX ... SET (fastupdate = off)` does not flush**, 100,000 rows:

| Step | `pending_pages` |
|---|---|
| After insert | 246 |
| After `ALTER INDEX ... SET (fastupdate = off)` | 246 |
| After `gin_clean_pending_list` (returned 246) | 0 |

**Tool acceptance**, on the 218-page GIN index:

```text
pgstatginindex('t_xid_gin')   -> version 2, pending_pages 0, pending_tuples 0
pg_relpages('t_xid_gin')      -> 218
pgstatindex('t_xid_gin')      -> ERROR:  relation "t_xid_gin" is not a btree index
pgstattuple('t_xid_gin')      -> ERROR:  "t_xid_gin" (gin index) is not supported
pgstattuple_approx(...)       -> ERROR:  "t_xid_gin" is not a table or materialized view
```

`pg_class` reported `relpages 218`, `reltuples 800000` — the heap tuple count, matching `ginvacuumcleanup`'s own admission that it "always report[s] the heap tuple count as the number of index entries" ([ginvacuum.c#ginvacuumcleanup-num_index_tuples](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L725-L731)). After a bitmap scan returning 2000 rows, `pg_stat_all_indexes` showed `idx_scan 1`, `idx_tup_read 2000`, `idx_tup_fetch 0`, confirming the structurally dead column.

All five filed SQL snippets ran verbatim at the pin with `ON_ERROR_STOP` set. The recipe-1 survey returned only GIN indexes in a database that also contained B-tree, hash, GiST, and BRIN indexes. The server was stopped afterwards and its disposable fixtures were left under `.wiki-runtime/`.

### Key data structures

| Structure | Where | Role in bloat |
|---|---|---|
| `GinMetaPageData` | [ginblock.h#GinMetaPageData](../../../raw/postgres-12/src/include/access/ginblock.h#L54-L102) | pending head/tail plus the four last-VACUUM counters |
| `GinPageOpaqueData` | [ginblock.h#GinPageOpaqueData](../../../raw/postgres-12/src/include/access/ginblock.h#L29-L36) | per-page `flags` that the census reads; `maxoff` is class-specific |
| `GinStatsData` | [gin.h#GinStatsData](../../../raw/postgres-12/src/include/access/gin.h#L38-L49) | the planner's view; deliberately omits `nPendingHeapTuples` |
| `GinOptions` | [gin_private.h#GinOptions](../../../raw/postgres-12/src/include/access/gin_private.h#L21-L39) | `fastupdate` and the per-index pending limit |
| `disassembledLeaf` / `leafSegmentInfo` | [gindatapage.c#disassembledLeaf](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L45-L103) | split bookkeeping (`lastleft`, `lsize`, `rsize`) that decides leaf fill |
| `GinPostingList` | [ginblock.h#GinPostingList](../../../raw/postgres-12/src/include/access/ginblock.h#L334-L347) | varbyte-compressed segment; min/target/max 128/256/384 bytes ([gindatapage.c#GinPostingListSegmentMaxSize](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L25-L36)) |
| `GinIndexStat` | [pgstatindex.c#GinIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L97-L108) | the three fields `pgstatginindex` will return |
| `IndexBulkDeleteResult` | consumed at [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827) | `pages_deleted` / `pages_free`, visible only in `VACUUM VERBOSE` |

### Caller and callee boundary

Insert side: `ExecInsertIndexTuples` -> `index_insert` -> `gininsert` ([ginutil.c#ginhandler](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L58-L62)) -> either `ginHeapTupleFastCollect` + `ginHeapTupleFastInsert` (fastupdate) or `ginHeapTupleInsert` -> `ginEntryInsert` ([gininsert.c#gininsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L484-L537)). Page allocation always funnels through `GinNewBuffer` ([ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)).

VACUUM side: `lazy_vacuum_index` runs `ginbulkdelete` only when the heap scan collected dead TIDs ([vacuumlazy.c#lazy_vacuum_all_indexes](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1419-L1440)), whereas `lazy_cleanup_index` -> `ginvacuumcleanup` runs on every indexed VACUUM ([vacuumlazy.c#lazy_cleanup_index-loop](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1466-L1471)). Inside `ginbulkdelete`: pending cleanup, then a left-to-right walk of entry leaves calling `ginVacuumEntryPage`, then `ginVacuumPostingTree` per collected posting-tree root ([ginvacuum.c#ginbulkdelete](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L563-L684)).

`lazy_cleanup_index` also gates the catalog update: when VACUUM skipped any heap page, `estimated_count` is true and `pg_class.relpages`/`reltuples` for the index are not refreshed at all ([vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1784-L1815)).

Planner side: `cost_index` -> `IndexOptInfo.amcostestimate` -> `gincostestimate` -> `ginGetStats` ([costsize.c#cost_index-amcostestimate](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L537-L548), [selfuncs.c#gincostestimate-ginGetStats](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6700-L6714)).

### Build, generated-header, and extension boundary

Every GIN-versus-not-GIN gate in the measurement path compares `relam` against `GIN_AM_OID`, which is not a hand-written constant. It is declared as an `oid_symbol` in the catalog data file ([pg_am.dat:27-29](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L27-L29)) and emitted into the generated `pg_am_d.h` by `genbki.pl` ([genbki.pl#oid_symbol-emit](../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603)), which the backend `generated-headers` target drives ([Makefile#generated-headers](../../../raw/postgres-12/src/backend/Makefile#L160-L162), [catalog/Makefile#generated-header-symlinks](../../../raw/postgres-12/src/backend/catalog/Makefile#L71-L90)). `pg_am_d.h` is a build artifact and is absent from the checkout, being listed in [catalog/.gitignore](../../../raw/postgres-12/src/include/catalog/.gitignore#L1-L3) — so the `#define GIN_AM_OID 2742` line itself is not citable here.

Consumers of that symbol: `gin_clean_pending_list` ([ginfast.c#gin_clean_pending_list-am-check](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1044-L1050)), `pgstatginindex`'s `IS_GIN` macro ([pgstatindex.c#IS_GIN](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L73)), and `pgstattuple`'s rejection switch ([pgstattuple.c#pgstat_relation](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276)).

Extension boundary: the whole measurement story lives in three contrib modules that must be installed separately — `pgstattuple` (default version 1.5, GIN function added in 1.1 and re-created as `pgstatginindex_v1_5` in [pgstattuple--1.4--1.5.sql:49-57](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57)), `pageinspect` (GIN trio added in [pageinspect--1.2--1.3.sql:46-82](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.2--1.3.sql#L46-L82), marked parallel-safe in [pageinspect--1.4--1.5.sql:22-24](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.4--1.5.sql#L22-L24)), and `pg_freespacemap`. `amcheck` ships no GIN support at all.

### Tests and explicit test absence

What the pinned tree covers:

- `src/test/regress/sql/gin.sql` exercises `fastupdate` on and off, a per-index `gin_pending_list_limit = 4096`, `gin_clean_pending_list` returning `>10` and then exactly `0`, and VACUUM after deletes ([gin.sql:1-36](../../../raw/postgres-12/src/test/regress/sql/gin.sql#L1-L36), asserted output at [gin.out:12-24](../../../raw/postgres-12/src/test/regress/expected/gin.out#L12-L24)).
- `contrib/pageinspect/sql/gin.sql` asserts one metapage snapshot and the two page-type error messages ([gin.sql:1-19](../../../raw/postgres-12/contrib/pageinspect/sql/gin.sql#L1-L19), [gin.out:5-37](../../../raw/postgres-12/contrib/pageinspect/expected/gin.out#L5-L36)).
- `contrib/pgstattuple` asserts `pgstatginindex` on an empty index and several wrong-AM errors ([pgstattuple.out:126-152](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L126-L152)).
- `src/test/isolation/specs/predicate-gin.spec` covers SSI predicate locking with `fastupdate` toggled, not space ([predicate-gin.spec:12-19](../../../raw/postgres-12/src/test/isolation/specs/predicate-gin.spec#L12-L19)).

What is explicitly not covered:

- **No test asserts a GIN index's size or page count after churn.** `contrib/pageinspect/sql/gin.sql` uses `pg_relation_size` only to address the last block ([gin.sql:16-19](../../../raw/postgres-12/contrib/pageinspect/sql/gin.sql#L16-L19)).
- **No test asserts any GIN plan cost.** Both GIN `EXPLAIN`s in `create_index.sql` use `(costs off)` ([create_index.sql:283-284](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L283-L284)), so the stale-stats scaling branch and the invented-statistics fallback have no direct coverage.
- **`pgstattuple('gin_index')` is never called**, so the `(gin index) is not supported` path is untested; the `pgstattuple` fixture table is never populated, so a non-empty pending list is never measured.
- **`gin_pending_list_limit` is never exercised as a GUC**, only as a reloption.
- **`pageinspect`'s GIN test uses `fastupdate = off`**, so no `GIN_LIST` page is ever inspected.
- **`contrib/pg_freespacemap` has no regression tests** — its `Makefile` has no `REGRESS` line ([pg_freespacemap/Makefile](../../../raw/postgres-12/contrib/pg_freespacemap/Makefile#L1-L20)).
- **`src/test/regress/sql/vacuum.sql` has no GIN index**, and `src/test/modules/` has no GIN coverage.

## Context Reviewed

- GIN access method: `ginfast.c`, `ginvacuum.c`, `gindatapage.c`, `ginentrypage.c`, `ginutil.c`, `gininsert.c`, `ginscan.c`, `ginget.c`, `ginpostinglist.c`, `ginxlog.c`, plus `gin.h`, `gin_private.h`, `ginblock.h`, `ginxlog.h`, and `src/backend/access/gin/README`.
- VACUUM integration: `vacuumlazy.c`, `vacuum.c`, `analyze.c`, `indexfsm.c`, `freespace.c`.
- Planner: `selfuncs.c` (`gincostestimate`), `plancat.c`, `costsize.c`, `pathnodes.h`.
- Settings: `guc.c`, `reloptions.c`, `postgresql.conf.sample`.
- Measurement tooling: `contrib/pgstattuple` (`pgstatindex.c`, `pgstattuple.c`, `pgstatapprox.c`, install/upgrade scripts, tests), `contrib/pageinspect` (`ginfuncs.c`, `rawpage.c`, scripts, tests), `contrib/pg_freespacemap`, `contrib/amcheck` (`verify_nbtree.c`).
- Catalog/generated-header boundary: `pg_am.dat`, `pg_am.h`, `genbki.pl`, `src/backend/Makefile`, `src/backend/catalog/Makefile`, `src/include/catalog/.gitignore`.
- Documentation: `gin.sgml`, `maintenance.sgml`, `ref/create_index.sgml`, `ref/reindex.sgml`, `pgstattuple.sgml`, `pageinspect.sgml`, `pgfreespacemap.sgml`, `monitoring.sgml`, `func.sgml`, `catalogs.sgml`, `storage.sgml`.
- Tests: `src/test/regress/sql/gin.sql`, `create_index.sql`, `tsearch.sql`, `vacuum.sql`, `src/test/isolation/specs/predicate-gin.spec`, `contrib/pgstattuple/sql`, `contrib/pageinspect/sql/gin.sql`, `contrib/amcheck/sql`.
- Exact-pin execution on an isolated 12.2 server built from `raw/postgres-12/` into `.wiki-runtime/pg12-install`.

## Evidence Map

| Claim | Evidence |
|---|---|
| `fastupdate` defaults on; inserts go to the pending list | [gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../raw/postgres-12/src/include/access/gin_private.h#L31-L39), [gininsert.c#gininsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L511-L531) |
| Trim threshold uses page capacity, not bytes used | [ginfast.c#ginHeapTupleFastInsert-cleanup-trigger](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L461), [ginfast.c#GIN_PAGE_FREESIZE](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L40-L41) |
| Wide rows get pending pages to themselves, wasting space | [README#GIN_LIST_FULLROW](../../../raw/postgres-12/src/backend/access/gin/README#L206-L216), [ginfast.c#makeSublist](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L143-L208) |
| Insert-driven cleanup abandons on contention | [ginfast.c#ginInsertCleanup-locking](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L800-L828) |
| Cleanup stops at the remembered tail unless full | [ginfast.c#ginInsertCleanup-cleanupFinish](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L881-L888) |
| Autovacuum never does a full pending clean | [ginvacuum.c#ginbulkdelete-pending-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L583-L594), [ginvacuum.c#ginvacuumcleanup-pending-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L711-L721) |
| Autoanalyze cleans the pending list; manual ANALYZE does not | [ginvacuum.c#ginvacuumcleanup-analyze_only](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L697-L709) |
| `gin_clean_pending_list` requires ownership, refuses recovery | [ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1030-L1074) |
| Turning off `fastupdate` does not flush the list | [create_index.sgml#fastupdate](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L459-L467) |
| Entry tree is never deleted from | [README#no-entry-delete](../../../raw/postgres-12/src/backend/access/gin/README#L28-L31), [README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396) |
| Emptied entry tuples are retained with a null posting list | [ginvacuum.c#ginVacuumEntryPage](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556), [ginentrypage.c#GinFormTuple](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L44-L154) |
| Entry splits equalize bytes, no append/build case | [ginentrypage.c#entrySplitPage](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L666-L689) |
| GIN has only two reloptions, no fillfactor | [ginutil.c#ginoptions](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L629) |
| Data leaves split 50/50, or 75% when appending, 100% on build | [gindatapage.c#dataBeginPlaceToPageLeaf-split](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L617-L667) |
| Internal data pages split 50/50 outside a build | [gindatapage.c#dataSplitPageInternal](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L1284-L1294) |
| VACUUM does not re-encode sparse segments | [gindatapage.c#ginVacuumPostingTreeLeaf-no-reencode](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L794-L810) |
| Only fully empty, non-edge posting pages are deleted | [ginvacuum.c#ginScanToDelete](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L302-L317) |
| Deleted pages carry the next XID and wait for `RecentGlobalXmin` | [ginvacuum.c#ginDeletePage-deleteXid](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L182-L192), [ginblock.h#GinPageIsRecyclable](../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138) |
| Only `ginvacuumcleanup` records GIN free pages | [ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L777) |
| GIN never truncates the relation | [ginvacuum.c#ginvacuumcleanup-num_pages](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L788-L792), [vacuumlazy.c#lazy_truncate_heap-RelationTruncate](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1962-L1965) |
| Freed pages are reused before extension | [ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335) |
| One entry per distinct key per row; non-HOT update re-inserts all | [gininsert.c#ginHeapTupleInsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L462-L482), [heapam.c#heap_update-hot-check](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3593-L3600) |
| `pgstatginindex` returns only three metapage fields | [pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573) |
| `pgstatindex` / `pgstattuple` / `pgstattuple_approx` reject GIN | [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L228), [pgstattuple.c#pgstat_relation](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276), [pgstatapprox.c#pgstattuple_approx_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L280-L290) |
| `pageinspect` GIN functions and their gates | [ginfuncs.c#gin_metapage_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L34-L87), [ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L90-L154), [ginfuncs.c#gin_leafpage_items](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L163-L265), [rawpage.c#get_raw_page_internal](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106) |
| Index FSM is binary, saturating at 8160 bytes | [indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55), [pgfreespacemap.sgml#index-caveat](../../../raw/postgres-12/doc/src/sgml/pgfreespacemap.sgml#L66-L70) |
| Metapage stats are as of the last VACUUM | [ginutil.c#ginGetStats](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L657) |
| Planner charges every pending page at `random_page_cost` | [selfuncs.c#gincostestimate-pending](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6869-L6873), [selfuncs.c#gincostestimate-startup](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6922-L6926) |
| 4x growth cutoff, then invented statistics | [selfuncs.c#gincostestimate-stats-trust](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6716-L6750), [selfuncs.c#gincostestimate-fallback](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6751-L6770) |
| Scans always walk the pending list first | [ginget.c#gingetbitmap-scanPendingInsert](../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1871-L1878), [gin.sgml#gin-fast-update-disadvantage](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L491-L499) |
| Planner uses the live block count, and no tree height for GIN | [plancat.c#get_relation_info-index-size](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [plancat.c#get_relation_info-tree_height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L418) |
| `gin_pending_list_limit` is `PGC_USERSET`; reloptions take `AccessExclusiveLock` | [guc.c#gin_pending_list_limit](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184), [reloptions.c#fastupdate](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L125-L133), [reloptions.c#gin_pending_list_limit](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L322-L330) |
| `idx_tup_fetch` is structurally 0 for GIN | [ginutil.c#ginhandler](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L71-L72), [indexam.c#index_fetch_heap](../../../raw/postgres-12/src/backend/access/index/indexam.c#L565-L589), [monitoring.sgml#idx_tup_fetch](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2914-L2929) |
| `pg_class` index stats skipped when VACUUM skipped heap pages | [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1784-L1815) |
| GIN reports heap tuple count as index entries | [ginvacuum.c#ginvacuumcleanup-num_index_tuples](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L725-L731) |
| amcheck has no GIN support | [verify_nbtree.c#btree_index_checkable](../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323) |
| Documentation concedes non-B-tree bloat is under-researched | [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L876-L880) |
| `GIN_AM_OID` comes from a generated header | [pg_am.dat:27-29](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L27-L29), [genbki.pl#oid_symbol-emit](../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603), [catalog/.gitignore](../../../raw/postgres-12/src/include/catalog/.gitignore#L1-L3) |

## Open Questions

- **`indexfsm.c`'s header comment contradicts its own code.** The comment says the FSM uses "`BLCKSZ - 1` to denote used pages, and 0 for unused" ([indexfsm.c#NOTES](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L14-L19)), but `RecordFreeIndexPage` writes `BLCKSZ - 1` and `RecordUsedIndexPage` writes 0 ([indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L65)). Source wins, and the exact-pin run agrees: free GIN pages read back as `avail = 8160`. The comment appears to be inverted.
- **No steady-state occupancy figure is derivable from source.** The split rules give 50%, 75%, and build-time-full targets, but the pinned tree states no expected long-run fill for entry or posting pages under a mixed workload. The 27.8% retail-versus-rebuild gap measured here is one fixture, not a general constant.
- **No threshold for "bloated" is defined anywhere in the pinned tree.** Unlike B-tree, where `fillfactor` gives a reference density, GIN has no target to compare against. The recipes here report facts; choosing an action threshold remains a judgement call, and recipe 5 is the only way to convert it into a byte count.
- **Autovacuum's partial pending clean was not measured.** The `full_clean = !IsAutoVacuumWorkerProcess()` asymmetry is established from source, but the exact-pin server ran with `autovacuum = off`, so the size of the residue an autovacuum worker leaves behind is unquantified here.
- **`gin_leafpage_items` was not used to quantify within-leaf slack.** It would give exact per-segment byte counts for posting-tree leaves, which is the missing piece for a true density metric. Its restriction to compressed data leaves ([ginfuncs.c#gin_leafpage_items](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L196-L202)) means it cannot cover entry-tree leaves, so it could only ever measure part of the index.
- **Cross-version behavior is out of scope.** Whether any of these mechanisms changed after v12 was not investigated, and per the evidence rules cannot be answered from this checkout.

## Source References

- GIN implementation: [ginfast.c](../../../raw/postgres-12/src/backend/access/gin/ginfast.c), [ginvacuum.c](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c), [gindatapage.c](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c), [ginentrypage.c](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c), [ginutil.c](../../../raw/postgres-12/src/backend/access/gin/ginutil.c), [gininsert.c](../../../raw/postgres-12/src/backend/access/gin/gininsert.c), [ginget.c](../../../raw/postgres-12/src/backend/access/gin/ginget.c), [ginscan.c](../../../raw/postgres-12/src/backend/access/gin/ginscan.c), [ginpostinglist.c](../../../raw/postgres-12/src/backend/access/gin/ginpostinglist.c), [README](../../../raw/postgres-12/src/backend/access/gin/README)
- GIN headers: [ginblock.h](../../../raw/postgres-12/src/include/access/ginblock.h), [gin_private.h](../../../raw/postgres-12/src/include/access/gin_private.h), [gin.h](../../../raw/postgres-12/src/include/access/gin.h)
- VACUUM and FSM: [vacuumlazy.c](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c), [indexfsm.c](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c), [freespace.c](../../../raw/postgres-12/src/backend/storage/freespace/freespace.c)
- Planner: [selfuncs.c](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c), [plancat.c](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c), [costsize.c](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c)
- Settings: [guc.c](../../../raw/postgres-12/src/backend/utils/misc/guc.c), [reloptions.c](../../../raw/postgres-12/src/backend/access/common/reloptions.c)
- Measurement tooling: [pgstatindex.c](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c), [pgstattuple.c](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c), [pgstatapprox.c](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c), [ginfuncs.c](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c), [rawpage.c](../../../raw/postgres-12/contrib/pageinspect/rawpage.c), [pg_freespacemap.c](../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c), [verify_nbtree.c](../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c)
- Documentation: [gin.sgml](../../../raw/postgres-12/doc/src/sgml/gin.sgml), [maintenance.sgml](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml), [create_index.sgml](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml), [pgstattuple.sgml](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml), [pageinspect.sgml](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml), [pgfreespacemap.sgml](../../../raw/postgres-12/doc/src/sgml/pgfreespacemap.sgml), [monitoring.sgml](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml)
- Tests: [regress/sql/gin.sql](../../../raw/postgres-12/src/test/regress/sql/gin.sql), [pageinspect/sql/gin.sql](../../../raw/postgres-12/contrib/pageinspect/sql/gin.sql), [pgstattuple/expected/pgstattuple.out](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out), [predicate-gin.spec](../../../raw/postgres-12/src/test/isolation/specs/predicate-gin.spec)

## Navigation

- [v12 index](../index.md)
- [PostgreSQL 12 Codebase Navigation Guide (unverified)](../codebase-navigation-guide.md)
- [A Heuristic to Detect B-Tree Index Bloat in PostgreSQL 12 (unverified)](index-bloat-heuristic.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [How NULL Values Are Handled in PostgreSQL 12 Indexes (unverified)](null-values-in-indexes.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
