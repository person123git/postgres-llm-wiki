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
  - [Mechanism 2: entry-tree tuples and pages are retained](#mechanism-2-entry-tree-tuples-and-pages-are-retained)
  - [Mechanism 3: posting-tree split policy leaves headroom](#mechanism-3-posting-tree-split-policy-leaves-headroom)
  - [Mechanism 4: VACUUM leaves slack inside posting-tree leaves](#mechanism-4-vacuum-leaves-slack-inside-posting-tree-leaves)
  - [Mechanism 5: deleted posting-tree pages wait for global xmin](#mechanism-5-deleted-posting-tree-pages-wait-for-global-xmin)
  - [Mechanism 6: ordinary VACUUM never truncates the GIN file](#mechanism-6-ordinary-vacuum-never-truncates-the-gin-file)
  - [Write amplification is the driver](#write-amplification-is-the-driver)
  - [What VACUUM can and cannot recover](#what-vacuum-can-and-cannot-recover)
  - [Measuring: which tools accept a GIN index](#measuring-which-tools-accept-a-gin-index)
  - [Measurement recipe 1: pending-list backlog](#measurement-recipe-1-pending-list-backlog)
  - [Measurement recipe 2: per-page census](#measurement-recipe-2-per-page-census)
  - [Measurement recipe 3: metapage picture versus live size](#measurement-recipe-3-metapage-picture-versus-live-size)
  - [Measurement recipe 4: pages recorded free in the FSM](#measurement-recipe-4-pages-recorded-free-in-the-fsm)
  - [Measurement recipe 5: ground truth by rebuilding](#measurement-recipe-5-ground-truth-by-rebuilding)
  - [How bloat reaches the planner](#how-bloat-reaches-the-planner)
  - [Settings, and their apply scope](#settings-and-their-apply-scope)
  - [Exact-pin measurements](#exact-pin-measurements)
  - [Key data structures](#key-data-structures)
  - [Caller and callee boundary](#caller-and-callee-boundary)
  - [Build, generated-header, and extension boundary](#build-generated-header-and-extension-boundary)
  - [Tests and explicit test absence](#tests-and-explicit-test-absence)
  - [Follow-up: measuring wasted bytes with core SQL only](#follow-up-measuring-wasted-bytes-with-core-sql-only)
  - [Why core SQL cannot see inside a GIN index](#why-core-sql-cannot-see-inside-a-gin-index)
  - [What core SQL does expose](#what-core-sql-does-expose)
  - [Core method A: rebuild probe, exact](#core-method-a-rebuild-probe-exact)
  - [Core method B: recorded baseline bytes per row](#core-method-b-recorded-baseline-bytes-per-row)
  - [The reltuples trap](#the-reltuples-trap)
  - [Core method C: sampled probe, rejected](#core-method-c-sampled-probe-rejected)
  - [Which size to divide by](#which-size-to-divide-by)
  - [Core-only pending-list probes](#core-only-pending-list-probes)
  - [Accuracy measured at the pin](#accuracy-measured-at-the-pin)
  - [Settings for the core-only recipes](#settings-for-the-core-only-recipes)
  - [Test coverage for the core-only path](#test-coverage-for-the-core-only-path)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, how can a GIN index become bloated? How do you measure whether a GIN index is bloated?

Follow-up:

How, using SQL and no extra contrib extension, can I measure the wasted bytes of a GIN index and provide a bloat percentage based on wasted bytes?

## Answer

### Short answer

A PostgreSQL 12 GIN index can be larger than a fresh build for six related reasons. They are not all the same kind of bloat: a pending list is a live write buffer, split headroom is normal structure, while retained empty entry tuples and sparse posting pages are historical space.

| # | Source of excess size relative to a fresh build | What ordinary VACUUM can do | Evidence |
|---|---|---|---|
| 1 | Pending-list (`fastupdate`) pages accumulate | Merge some or all of the list and make removed list pages reusable | [ginfast.c#ginInsertCleanup](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L780-L888), [ginfast.c#shiftList](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L560-L667) |
| 2 | Entry-tree tuples and pages remain after their posting lists empty | Remove dead TIDs, but retain the entry tuple and page | [ginvacuum.c#ginVacuumEntryPage](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556), [README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396) |
| 3 | Retail posting-tree splits leave balancing or append headroom | Does not rebalance populated pages | [gindatapage.c#dataBeginPlaceToPageLeaf-split](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L617-L667) |
| 4 | Partly emptied posting-tree leaves retain page and segment slack | Recompress changed segments, but does not coalesce them or merge sibling pages | [gindatapage.c#ginVacuumPostingTreeLeaf](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L744-L810), [ginvacuum.c#ginScanToDelete](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L302-L317) |
| 5 | Deleted posting-tree pages wait before reuse | Record them in the free space map (FSM) after the delete XID precedes `RecentGlobalXmin` | [ginblock.h#GinPageIsRecyclable](../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L777) |
| 6 | Reusable pages remain inside the main fork at its high-water size | Reuse them; ordinary VACUUM does not shorten the GIN fork | [ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335), [ginvacuum.c#ginvacuumcleanup-num_pages](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L792) |

`REINDEX` is the direct index-only operation that replaces the physical file and repacks the live contents ([index.c#reindex_index-new-relfilenode](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3530)). `REINDEX CONCURRENTLY` builds a replacement index before swapping and dropping the old one ([indexcmds.c#ReindexRelationConcurrently-phases](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2941-L2952)). It is therefore wrong to say that no other command can shrink the file: `VACUUM FULL` and `CLUSTER` rebuild every index after rewriting the table ([cluster.c#rebuild_relation](../../../raw/postgres-12/src/backend/commands/cluster.c#L610-L626), [cluster.c#finish_heap_swap-reindex](../../../raw/postgres-12/src/backend/commands/cluster.c#L1378-L1410)), and `TRUNCATE` reconstructs indexes after removing all table rows ([heap.c#RelationTruncateIndexes](../../../raw/postgres-12/src/backend/catalog/heap.c#L3163-L3205), [tablecmds.c#ExecuteTruncate-reindex](../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1813-L1840)). Those table-level commands have materially different data and locking semantics; they are not substitutes for an index-only bloat repair.

Measurement is harder than for B-tree, because v12 ships no GIN density function. `pgstatindex` and `pgstattuple` both refuse a GIN index. The usable shipped tools are core size functions, `pgstattuple`'s `pg_relpages` and pending-list-only `pgstatginindex`, `pageinspect`'s three GIN functions, and `pg_freespacemap`; the closest byte answer is comparison with a freshly built copy. The v12 documentation itself says non-B-tree bloat has not been well researched and recommends monitoring physical size ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L876-L880)).

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

Autovacuum is asymmetric here. Both VACUUM entry points pass `full_clean = !IsAutoVacuumWorkerProcess()`: an autovacuum worker stops after processing the tail remembered at entry, while manual `VACUUM` continues until the list is empty ([ginvacuum.c#ginbulkdelete-pending-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L583-L594), [ginvacuum.c#ginvacuumcleanup-pending-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L711-L721), [ginfast.c#ginInsertCleanup-cleanupFinish](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L843-L888)). With no concurrent append, the remembered tail can still be the end of the whole pre-existing list, so `full_clean = false` does not imply a nonempty residue. An autovacuum *analyze* also invokes a partial clean, while a plain manual `ANALYZE` is a no-op ([ginvacuum.c#ginvacuumcleanup-analyze_only](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L697-L709)).

`gin_clean_pending_list(regclass)` is the dedicated way to request a full clean without vacuuming the heap; manual `VACUUM` is another full-clean caller. The function requires index ownership, refuses to run during recovery, and returns the number of pending pages deleted ([ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1030-L1074)).

Trimmed pending pages get the `GIN_DELETED` flag and, when `fill_fsm` is set, go straight to the free space map ([ginfast.c#shiftList-flags](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L632), [ginfast.c#shiftList-RecordFreeIndexPage](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L659-L665)). They become reusable pages inside the same file; the file does not shrink.

One operational trap: turning `fastupdate` off does not flush what is already pending. The documentation says so ([create_index.sgml#fastupdate](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L459-L467)), and the exact-pin run below confirms it.

### Mechanism 2: entry-tree tuples and pages are retained

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

Consequence: under ordinary DML and VACUUM, a key that once existed keeps an entry-tuple slot and its entry-tree page is not reclaimed. A workload that cycles through distinct keys therefore makes the entry-tree page count nondecreasing until a physical rebuild ([README#no-entry-delete](../../../raw/postgres-12/src/backend/access/gin/README#L28-L31), [README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396)).

A stable key set can retain extra entry pages too. An in-line posting list grows inside its entry tuple; when the tuple can no longer hold it, `addItemPointersToLeafTuple` creates a posting tree and replaces the expanded tuple with a small tree pointer ([gininsert.c#addItemPointersToLeafTuple](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L42-L118)). If those expanded tuples already forced entry-page splits, their later shrink does not merge the entry pages because VACUUM never deletes entry-tree tuples or pages ([README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396)).

Entry-tree splits try to balance total tuple bytes between the two output pages; tuple granularity can make the result uneven. There is no build-time or rightmost-append special case ([ginentrypage.c#entrySplitPage](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L654-L689)), and `entryIsEnoughSpace` reserves no fillfactor ([ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L459-L483)). GIN exposes no `fillfactor` reloption; its two reloptions are `fastupdate` and `gin_pending_list_limit` ([ginutil.c#ginoptions](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L629)).

### Mechanism 3: posting-tree split policy leaves headroom

When a key accumulates more TIDs than fit in an entry tuple, the TIDs move to a posting tree ([gininsert.c#addItemPointersToLeafTuple](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L42-L119)). The leaf split policy depends on build mode and, for ordinary DML, whether the insertion is an append:

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

([gindatapage.c#dataBeginPlaceToPageLeaf-split](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L617-L667)). These are heuristics over discrete compressed segments, not exact occupancy promises: ordinary splits balance the two sides, the append case tries to leave the left side at least about 75% full, and a build packs the split's left page before placing the remainder on the right. Internal posting-tree pages pack a rightmost build split's left page as far as possible; other internal splits divide posting-item counts in half ([gindatapage.c#dataSplitPageInternal](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L1284-L1294)).

`btree->isBuild` is set from a non-null build-statistics pointer on both entry and posting-tree insertion paths ([gininsert.c#ginEntryInsert-isBuild](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L180-L200), [gindatapage.c#ginInsertItemPointers-isBuild](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L1901-L1912)). `CREATE INDEX` and `REINDEX` can therefore produce denser posting trees than retail insertion of the same rows. The v12 documentation likewise advises recreating a GIN index after bulk insertion ([gin.sgml#gin-tips](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L535-L550)).

### Mechanism 4: VACUUM leaves slack inside posting-tree leaves

`ginVacuumPostingTreeLeaf` removes dead TIDs and recompresses each changed segment within that segment's old byte budget ([gindatapage.c#ginVacuumPostingTreeLeaf-segments](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L744-L790)). It then reconstructs the page, but deliberately does not coalesce or redivide small surviving segments:

```c
	 * We don't try to re-encode the segments here, even though some of them
	 * might be really small now that we've removed some items from them. It
	 * seems like a waste of effort, ...
	 * ... You'll have to REINDEX anyway if you want the full gain of the
	 * new tighter index format.
```

([gindatapage.c#ginVacuumPostingTreeLeaf-no-reencode](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L794-L810))

The implemented page-reclamation rule deletes a posting-tree page only when it is *completely* empty, and never when it is the left- or rightmost branch:

```c
	if (isempty)
	{
		/* we never delete the left- or rightmost branch */
		if (BufferIsValid(me->leftBuffer) && !GinPageRightMost(page))
```

([ginvacuum.c#ginScanToDelete](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L302-L317)). A partly populated leaf therefore keeps its page. The second deletion pass runs only if the first pass found at least one empty leaf ([ginvacuum.c#ginVacuumPostingTree](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L407-L447)).

Finally, a key promoted to a posting tree is never demoted back to an in-line posting list, so its posting-tree root survives even when few TIDs remain ([ginget.c#entryGetItem-no-demotion](../../../raw/postgres-12/src/backend/access/gin/ginget.c#L405-L410)).

### Mechanism 5: deleted posting-tree pages wait for global xmin

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

`ginvacuumcleanup` is the path that publishes XID-deferred posting-tree pages to the FSM: it scans the fork and records only pages for which `GinPageIsRecyclable` is true ([ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L777)). A second VACUUM is not sufficient by itself; its `RecentGlobalXmin` must have moved past the delete XID. Long-running transactions can delay that movement because `RecentGlobalXmin` is based on the oldest `TransactionXmin` across running non-lazy-VACUUM transactions ([ginblock.h#GinPageIsRecyclable](../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [procarray.c#RecentGlobalXmin](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1488-L1503)).

Pending-list pages follow a different path. `shiftList` replaces their flags with `GIN_DELETED` and can call `RecordFreeIndexPage` immediately when `fill_fsm` is true ([ginfast.c#shiftList-delete-and-record](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L665)). Insert-triggered cleanup and `gin_clean_pending_list` both request that immediate FSM recording ([ginfast.c#ginHeapTupleFastInsert-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L456-L461), [ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1067-L1070)). A raw `deleted` page count therefore mixes two states and cannot by itself mean “waiting for global xmin.”

### Mechanism 6: ordinary VACUUM never truncates the GIN file

The GIN cleanup callback scans the existing fork, records recyclable pages, vacuums the FSM, and reports `RelationGetNumberOfBlocks`; it contains no relation-truncation step ([ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L738-L794)). Generic index cleanup delegates to the access method's callback ([indexam.c#index_vacuum_cleanup](../../../raw/postgres-12/src/backend/access/index/indexam.c#L702-L712)), while lazy VACUUM's truncation path truncates the heap relation ([vacuumlazy.c#lazy_truncate_heap-RelationTruncate](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1962-L1965)). Ordinary VACUUM therefore does not shorten a GIN main fork.

Its high-water size persists until a physical rebuild or table truncation. `GinNewBuffer` asks the FSM before extending the file and validates that a returned candidate is actually recyclable ([ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)). All ordinary main-fork page allocation uses that shared path ([ginfast.c#makeSublist](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L143-L169), [ginbtree.c#ginPlaceToPage](../../../raw/postgres-12/src/backend/access/gin/ginbtree.c#L450-L498), [gindatapage.c#createPostingTree](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L1807-L1831)), so pending-list and tree growth can reuse the same recorded pages.

### Write amplification is the driver

GIN inserts one index entry per *distinct extracted key* per row ([gininsert.c#ginHeapTupleInsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L462-L482), dedup at [ginutil.c#ginExtractEntries](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L535-L590)). Even a NULL or zero-key item gets a placeholder entry ([ginutil.c#ginExtractEntries-placeholders](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L492-L526)).

An `UPDATE` that touches the indexed column cannot be HOT, so it re-inserts every key of the new row version while the old version's keys stay in the index until VACUUM removes the dead TIDs ([heapam.c#heap_update-hot-check](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3593-L3600), [heapam_handler.c#update_indexes](../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L336-L344)). The documentation states the amplification: "inserting or updating one heap row can cause many inserts into the index (one for each key extracted from the indexed item)" ([gin.sgml#gin-fast-update](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L473-L489)).

### What VACUUM can and cannot recover

| Page class | VACUUM deletes it? | Returns to FSM? | File shrinks? |
|---|---|---|---|
| Pending list (`GIN_LIST`) | Yes, on full/partial clean | Yes; `shiftList` can record it immediately | No |
| Posting-tree leaf, fully empty, not left/rightmost | Yes | Only once XID passes | No |
| Posting-tree leaf, partly empty | No | No | No |
| Posting-tree leaf, leftmost or rightmost | No | No | No |
| Posting-tree internal, fully empty, not left/rightmost | Yes | Only once XID passes | No |
| Posting-tree root | No | No | No |
| Entry-tree leaf or internal | **No** | **No** | No |
| Metapage | No | No | No |

Ordinary VACUUM never shrinks these files. Use `REINDEX` or `REINDEX CONCURRENTLY` for an index-only rebuild; table-rewrite or truncation commands can also rebuild or reset the index, but with different data and locking effects ([index.c#reindex_index-new-relfilenode](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3530), [cluster.c#finish_heap_swap-reindex](../../../raw/postgres-12/src/backend/commands/cluster.c#L1378-L1410), [heap.c#RelationTruncateIndexes](../../../raw/postgres-12/src/backend/catalog/heap.c#L3163-L3205)).

### Measuring: which tools accept a GIN index

| Tool | GIN? | What you get |
|---|---|---|
| `pg_relation_size` | Yes | one index fork's on-disk bytes; the one-argument SQL form defaults to the main fork ([dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336), [pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6891)) |
| `pgstattuple.pg_relpages` | Yes | live block count from `RelationGetNumberOfBlocks` ([pgstatindex.c#pg_relpagesbyid_v1_5](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L457-L477)) |
| `pgstattuple.pgstatginindex` | Yes | **only** `version`, `pending_pages`, `pending_tuples` ([pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573)) |
| `pgstattuple.pgstatindex` | **No** | `relation "%s" is not a btree index` ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L228)) |
| `pgstattuple.pgstattuple` | **No** | `"%s" (gin index) is not supported` ([pgstattuple.c#pgstat_relation](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276), [pgstattuple.c#pgstat_relation-ereport](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L308-L311)) |
| `pgstattuple.pgstattuple_approx` | **No** | `"%s" is not a table or materialized view` ([pgstatapprox.c#pgstattuple_approx_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L280-L290)) |
| `pageinspect.gin_metapage_info` | Yes, superuser | all ten metapage fields ([ginfuncs.c#gin_metapage_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L34-L87)) |
| `pageinspect.gin_page_opaque_info` | Yes, superuser | `rightlink`, `maxoff`, decoded `flags[]` ([ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L90-L154)) |
| `pageinspect.gin_leafpage_items` | Compressed data leaves only | posting-list segments ([ginfuncs.c#gin_leafpage_items](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L163-L265)) |
| `pg_freespacemap.pg_freespace` | Yes | recorded FSM value, effectively binary for an index ([pg_freespacemap.c#pg_freespace](../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c#L17-L41)) |
| `contrib/amcheck` | **No** | `only B-Tree indexes are supported as targets for verification` ([verify_nbtree.c#btree_index_checkable](../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323)) |

Two consequences matter. First, `pgstatginindex` reads exactly one buffer — the metapage — so it cannot report size, density, or free space, even though that metapage contains `nTotalPages`, `nEntryPages`, `nDataPages`, and `nEntries` fields that the function does not return ([pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L541-L564)). Second, `pg_freespace` returns what the FSM records, not a fresh inspection of each page ([pg_freespacemap.c#pg_freespace](../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c#L17-L41), [freespace.c#GetRecordedFreeSpace](../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L225-L247)). For an index the value is effectively binary: `RecordFreeIndexPage` writes `BLCKSZ - 1` ([indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55)), and the documentation says only whether a page is entirely unused is meaningful ([pgfreespacemap.sgml#index-caveat](../../../raw/postgres-12/doc/src/sgml/pgfreespacemap.sgml#L66-L70)).

`pg_stat_all_indexes` is an access counter, not a size tool, and one of its columns is structurally dead for GIN: `idx_tup_fetch` is only incremented on the `amgettuple` path ([indexam.c#index_fetch_heap](../../../raw/postgres-12/src/backend/access/index/indexam.c#L565-L589)), and GIN sets `amgettuple = NULL` ([ginutil.c#ginhandler](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L71-L72)). `idx_scan` and `idx_tup_read` do work ([ginscan.c:409](../../../raw/postgres-12/src/backend/access/gin/ginscan.c#L409), [indexam.c#index_getbitmap](../../../raw/postgres-12/src/backend/access/index/indexam.c#L651-L670)). The documentation describes the same exclusion ([monitoring.sgml#idx_tup_fetch](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2914-L2929)).

### Measurement recipe 1: pending-list backlog

Run this first. `pgstatginindex` reads one share-locked metapage buffer and returns the pending counters stored there ([pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573)); a non-superuser member of `pg_stat_scan_tables` may execute it ([pgstattuple--1.4--1.5.sql:49-57](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57)). Pending-list cleanup has a direct remedy, although its duration depends on list size, memory, storage, and contention ([ginfast.c#ginInsertCleanup-workMemory](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L807-L828)).

```sql
SET /* wiki_gin_pending_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_pending_lock_timeout */ lock_timeout = '5s';

WITH /* wiki_gin_pending_list_backlog */ gin_indexes AS MATERIALIZED (
  SELECT c.oid AS idxoid,
         n.nspname AS schema_name,
         c.relname AS index_name
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'i'
    AND c.relam = (SELECT oid FROM pg_am WHERE amname = 'gin')
    AND c.relpersistence <> 't'
)
SELECT g.schema_name,
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

`AS MATERIALIZED` is load-bearing: the v12 planner treats that option as an instruction not to inline the CTE ([subselect.c#CTEMaterializeAlways](../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L857-L892)). That prevents outer-query reordering from applying `pgstatginindex` to a non-GIN relation, which would abort the query with the error quoted above. `relpersistence <> 't'` avoids the other-sessions temp-index error ([pgstatindex.c#pgstatginindex_internal-temp](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L531-L539)). At the pin this returned only the nine GIN indexes in a database that also held B-tree, hash, GiST, and BRIN indexes.

Remedy when `pending_pages` is large: `SELECT /* wiki_gin_clean_pending_list */ gin_clean_pending_list('public.mixed_gin_b');` (index owner, not during recovery). This merges pending entries and makes former list pages reusable; it does not promise to reduce total file size and can allocate tree pages while it runs ([ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1030-L1074), [ginfast.c#shiftList-delete-and-record](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L665)).

### Measurement recipe 2: per-page census

Among the SQL tools shipped with v12, this is the way to classify every main-fork page. It needs superuser, because `get_raw_page` gates on `superuser()` rather than a role, and each call opens the relation with `AccessShareLock` ([rawpage.c#get_raw_page_internal](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L94-L110)). Run it during a quiet interval: each page is copied under a buffer lock, but the loop is not one atomic physical snapshot ([rawpage.c#get_raw_page_internal-copy](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L148-L169)).

```sql
SET /* wiki_gin_census_statement_timeout */ statement_timeout = '60s';
SET /* wiki_gin_census_lock_timeout */ lock_timeout = '5s';

SELECT /* wiki_gin_page_census */
       CASE
         WHEN 'deleted' = ANY (o.flags) AND 'data' = ANY (o.flags)
           THEN 'deleted posting-tree page'
         WHEN 'deleted' = ANY (o.flags)
           THEN 'deleted pending-list page'
         WHEN 'meta'    = ANY (o.flags) THEN 'metapage'
         WHEN 'list'    = ANY (o.flags) THEN 'pending list'
         WHEN 'data'    = ANY (o.flags) AND 'leaf' = ANY (o.flags) THEN 'posting-tree leaf'
         WHEN 'data'    = ANY (o.flags) THEN 'posting-tree internal'
         WHEN 'leaf'    = ANY (o.flags) THEN 'entry-tree leaf'
         ELSE 'entry-tree internal'
       END AS page_class,
       count(*) AS pages,
       count(*) FILTER (
         WHERE pg_freespace('public.mixed_gin_b', b.blkno) > 0
       ) AS fsm_recorded_pages,
       pg_size_pretty(count(*) * current_setting('block_size')::bigint) AS bytes
FROM generate_series(0,
       (pg_relation_size('public.mixed_gin_b') / current_setting('block_size')::bigint) - 1
     ) AS b(blkno)
CROSS JOIN LATERAL gin_page_opaque_info(
  get_raw_page('public.mixed_gin_b', b.blkno::int)
) AS o
GROUP BY 1
ORDER BY 2 DESC;
```

Read it this way:

- A large **pending list** share means recipe 1's remedy applies.
- **Deleted posting-tree pages** with `fsm_recorded_pages = 0` can be waiting for `RecentGlobalXmin`. If the FSM column is nonzero, VACUUM has already recorded those pages; the `GIN_DELETED` flag remains until reuse ([ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L777), [ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L297-L321)).
- **Deleted pending-list pages** normally enter the FSM immediately when insert-driven cleanup, autoanalyze, or `gin_clean_pending_list` requests `fill_fsm = true` ([ginfast.c#shiftList-delete-and-record](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L665)).
- A large **entry-tree** share against a small live key count is not recoverable by ordinary VACUUM; an index or table rebuild is needed to repack it ([README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396), [index.c#reindex_index-new-relfilenode](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3530)).
- **The two `deleted` cases are checked first on purpose.** `GinPageSetDeleted` uses `|=`, so a deleted posting-tree page retains `GIN_DATA`; pending-list deletion instead replaces all flags with `GIN_DELETED` ([ginblock.h#GinPageSetDeleted](../../../raw/postgres-12/src/include/access/ginblock.h#L124-L126), [ginfast.c#shiftList-delete-flag](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L632)).

Two caveats on this recipe. `gin_page_opaque_info` performs no page-type validation whatsoever — it casts the special pointer unconditionally ([ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L109-L111)) — so point it only at a GIN index. And `maxoff` is not an entry-tuple count: the struct comment scopes it to `GIN_DATA & ~GIN_LEAF` pages and `GIN_LIST` pages ([ginblock.h#GinPageOpaqueData](../../../raw/postgres-12/src/include/access/ginblock.h#L29-L36)), so it reads 0 on entry-tree leaves. Use the metapage `n_entries` for entry counts instead.

### Measurement recipe 3: metapage picture versus live size

The metapage's four planner counters are refreshed only by a build/rebuild and by VACUUM's GIN cleanup phase. `ginGetStats` marks `nPendingPages` and `ginVersion` as current but the other fields as last-VACUUM data ([ginutil.c#ginGetStats](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L657)); the only update call sites are the build and cleanup paths ([gininsert.c#ginbuild-stats](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L402-L406), [ginvacuum.c#ginvacuumcleanup-stats](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L779-L781)).

Comparing that saved figure with the live block count measures growth or counter staleness, not bloat by itself: legitimate inserts can create the same gap. It is also a planner-accuracy signal, per [How bloat reaches the planner](#how-bloat-reaches-the-planner).

```sql
SET /* wiki_gin_meta_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_meta_lock_timeout */ lock_timeout = '5s';

SELECT /* wiki_gin_metapage_vs_live */
       m.n_total_pages   AS pages_at_last_vacuum,
       (pg_relation_size('public.mixed_gin_b') / current_setting('block_size')::bigint) AS pages_now,
       m.n_entry_pages,
       m.n_data_pages,
       m.n_entries,
       m.n_pending_pages,
       m.version
FROM gin_metapage_info(get_raw_page('public.mixed_gin_b', 0)) AS m;
```

`n_entry_pages + n_data_pages` need not equal `n_total_pages - 1`: the cleanup loop counts neither recyclable pages nor live `GIN_LIST` pages into either bucket, while `nTotalPages` is the whole file ([ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L781)). A `version` below 2 denotes a pre-9.4 index that may contain legacy uncompressed posting data; version 0 may also lack null entries and placeholders ([ginblock.h#GinMetaPageData-version](../../../raw/postgres-12/src/include/access/ginblock.h#L84-L102)). A rebuild initializes the metapage to `GIN_CURRENT_VERSION` ([ginutil.c#GinInitMetabuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L349-L376)).

### Measurement recipe 4: pages recorded free in the FSM

```sql
SET /* wiki_gin_fsm_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_fsm_lock_timeout */ lock_timeout = '5s';

SELECT /* wiki_gin_free_pages */
       count(*) FILTER (WHERE f.avail > 0) AS free_pages,
       count(*)                            AS total_pages,
       round(100.0 * count(*) FILTER (WHERE f.avail > 0) / greatest(count(*), 1), 2)
         AS free_pct
FROM pg_freespace('public.mixed_gin_b') AS f;
```

`free_pages` counts positive entries recorded in the index FSM at the instant of the query. It is strong evidence of reusable high-water space, but not a guarantee: FSM entries are hints, and `GinNewBuffer` rechecks the page after obtaining and clearing a candidate ([indexfsm.c#GetFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L32-L45), [ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L297-L321)). A high `free_pct` establishes that many pages are recorded free; it does not by itself predict the exact rebuild size. A GIN `VACUUM VERBOSE` reports `stats->pages_free` from its own physical cleanup scan as “currently reusable,” which is comparable but need not be the same observation at a different time ([ginvacuum.c#ginvacuumcleanup-pages_free](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L786), [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827)).

### Measurement recipe 5: ground truth by rebuilding

Everything above is indirect. The closest direct measurement of rebuild-reclaimable bytes is a second index built from the same table contents with the same logical definition.

```sql
SET /* wiki_gin_rebuild_statement_timeout */ statement_timeout = '10min';
SET /* wiki_gin_rebuild_lock_timeout */ lock_timeout = '5s';

SELECT /* wiki_gin_index_definition */
       pg_get_indexdef('public.mixed_gin_b'::regclass);

CREATE /* wiki_gin_rebuild_probe */ INDEX CONCURRENTLY mixed_gin_b_probe
  ON public.mixed USING gin (b);

SELECT /* wiki_gin_rebuild_delta */
       pg_size_pretty(pg_relation_size('public.mixed_gin_b'))       AS current_size,
       pg_size_pretty(pg_relation_size('public.mixed_gin_b_probe')) AS rebuilt_size,
       round(100.0 * (pg_relation_size('public.mixed_gin_b') -
                      pg_relation_size('public.mixed_gin_b_probe'))
             / greatest(pg_relation_size('public.mixed_gin_b'), 1), 2)
         AS reclaimable_pct;

DROP /* wiki_gin_drop_rebuild_probe */ INDEX CONCURRENTLY public.mixed_gin_b_probe;
```

Replace the fixture names and reproduce the original definition shown by `pg_get_indexdef`: columns or expressions, operator classes, collations, predicate, and GIN reloptions all affect size. Choose a probe name that does not already exist. `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block ([utility.c#CREATE-INDEX-CONCURRENTLY](../../../raw/postgres-12/src/backend/tcop/utility.c#L1299-L1311)) and permits concurrent writes, so the comparison is an operational sample rather than one multi-relation snapshot. It builds a second index with two table scans, does more total work than a standard build, and adds CPU and I/O load while normal operations continue ([create_index.sgml#Building-Indexes-Concurrently](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L527-L558)); the old and probe indexes both occupy storage until the probe is dropped. Use it when the diagnostic value justifies that work and temporary disk capacity.

### How bloat reaches the planner

GIN bloat is not only a disk problem; `gincostestimate` reads it twice.

**Pending pages are charged as startup I/O.** When the metapage count is less than the live index size, every pending page is assumed read at scan start; an impossible count at least as large as the index is discarded as zero ([selfuncs.c#gincostestimate-pending-sanity](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6716-L6730)):

```c
	/*
	 * Compute cost to begin scan, first of all, pay attention to pending
	 * list.
	 */
	entryPagesFetched = numPendingPages;
```

([selfuncs.c#gincostestimate-pending](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6869-L6873)), then priced at `random_page_cost` ([selfuncs.c#gincostestimate-startup](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6922-L6926)). That charge is honest: every scan really does walk the pending list before the main index ([ginget.c#gingetbitmap-scanPendingInsert](../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1868-L1878)), which the documentation calls out as fast update's "main disadvantage" ([gin.sgml#gin-fast-update-disadvantage](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L491-L499)).

**Stale metapage stats can push the planner onto an invented model.** The four saved counters are used only when `nTotalPages` is positive, no larger than the live size, strictly greater than one quarter of the live size, and both `nEntryPages` and `nEntries` are positive:

```c
	if (numPages > 0 && ginStats.nTotalPages <= numPages &&
		ginStats.nTotalPages > numPages / 4 &&
		ginStats.nEntryPages > 0 && ginStats.nEntries > 0)
```

([selfuncs.c#gincostestimate-stats-trust](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6716-L6750)). The intended cutoff is 4x growth, and the strict comparison rejects the exact 4x boundary, subject to integer division. Otherwise the planner assumes 90% entry pages, 10% data pages, and 100 entries per entry page ([selfuncs.c#gincostestimate-fallback](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6751-L6770)). A never-vacuumed GIN index can fall into that branch, as the exact-pin run below shows.

The block count itself is always live, never `pg_class.relpages`: `get_relation_info` calls `RelationGetNumberOfBlocks` for a non-partial index ([plancat.c#get_relation_info-index-size](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407)), and `estimate_rel_size` also uses the live count for a partial one ([plancat.c#estimate_rel_size-index](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L955-L971)). `tree_height` stays `-1` for GIN ([plancat.c#get_relation_info-tree_height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L418)), so unlike B-tree there is no height charge.

### Settings, and their apply scope

| Setting | Kind | Apply scope |
|---|---|---|
| `gin_pending_list_limit` | GUC, `PGC_USERSET`, default 4096 kB, min 64 kB ([guc.c#gin_pending_list_limit](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184)) | **Session/transaction** — no restart, no reload |
| `fastupdate` | index reloption, default true ([reloptions.c#fastupdate](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L125-L133)) | `ALTER INDEX`, takes **`AccessExclusiveLock`** on the index |
| `gin_pending_list_limit` | index reloption, default −1 = inherit the GUC ([reloptions.c#gin_pending_list_limit](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L322-L330)) | `ALTER INDEX`, takes **`AccessExclusiveLock`** on the index |
| `maintenance_work_mem` | GUC, `PGC_USERSET` ([guc.c#maintenance_work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2243-L2251)) | **Session/transaction** — no restart, no reload; used by forced cleanup outside an autovacuum worker ([ginfast.c#ginInsertCleanup-workMemory](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L807-L817)) |
| `autovacuum_work_mem` | GUC, `PGC_SIGHUP` ([guc.c#autovacuum_work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3029-L3037)) | **Reload** — no restart; an autovacuum worker uses it for forced cleanup when it is not `-1` ([ginfast.c#ginInsertCleanup-workMemory](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L807-L817)) |
| `work_mem` | GUC, `PGC_USERSET` ([guc.c#work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2230-L2240)) | **Session/transaction** — no restart, no reload; used by insert-driven cleanup ([ginfast.c:827](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L827)) |
| `random_page_cost` | GUC, `PGC_USERSET` ([guc.c#random_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3217-L3226)) | **Session/transaction** — no restart, no reload; prices GIN startup page reads ([selfuncs.c#gincostestimate-startup](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6922-L6926)) |
| `enable_seqscan` | GUC, `PGC_USERSET` ([guc.c#enable_seqscan](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L883-L890)) | **Session/transaction** — no restart, no reload; used only in the exact-pin fixture to expose the GIN path, not recommended as a bloat remedy |

The reloption locks matter operationally: `ALTER INDEX ... SET (fastupdate = off)` requests `AccessExclusiveLock` and does not flush the existing pending list ([reloptions.c#fastupdate](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L125-L133), [create_index.sgml#fastupdate](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L459-L467)). The `statement_timeout` and `lock_timeout` settings in the measurement recipes are both `PGC_USERSET` ([guc.c#statement-and-lock-timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2398)); each `SET` applies at session scope, while `SET LOCAL` would apply only to the current transaction ([set.sgml#SET-scope](../../../raw/postgres-12/doc/src/sgml/ref/set.sgml#L38-L58)).

`pg_settings` at the pin confirms the GUC scope:

```text
          name          | setting | unit | context | boot_val | min_val |  max_val
------------------------+---------+------+---------+----------+---------+------------
 gin_fuzzy_search_limit | 0       |      | user    | 0        | 0       | 2147483647
 gin_pending_list_limit | 4096    | kB   | user    | 4096     | 64      | 2147483647
```

`context = user` maps to session/transaction scope.

### Exact-pin measurements

The review reran deterministic fixtures on an isolated PostgreSQL 12.2 server built from the pinned checkout, with 8 kB blocks, `autovacuum = off`, `maintenance_work_mem = 64MB`, and `pgstattuple`, `pageinspect`, and `pg_freespacemap` installed. Timings are intentionally omitted because they are machine-dependent; page counts, flags, FSM records, and planner costs are the useful results.

**Pending list: `fastupdate = on`, default limit, 200,000 rows of `ARRAY[g, g % 7, g % 13]`:**

| Measurement | Value |
|---|---|
| Before explicit cleanup | 12,722,176 B (1553 pages); 411 pending pages / 57,022 pending tuples |
| Census | 411 pending; 104 deleted former-list pages already FSM-recorded; 972 entry leaves; 5 entry internals; 47 posting leaves; 13 posting internals; 1 meta |
| Metapage counters | `n_total_pages 2`, `n_entry_pages 1`, `n_data_pages 0`, `n_entries 0` |
| `gin_clean_pending_list` | returned 411; size became 15,228,928 B; 411 former-list pages were FSM-recorded |
| `REINDEX` | 11,804,672 B |

Cleanup grew the file while moving entries into the trees, then left the removed list pages available for reuse; that is why “clean pending list” does not mean “smaller file” ([ginfast.c#shiftList-delete-and-record](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L665), [ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)). Each positive FSM value read back as 8160 bytes, the saturated category produced from `BLCKSZ - 1` on this 8 kB build ([indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55)).

**Entry-key retention: `fastupdate = off`, 300,000 rows of `ARRAY[g]`:**

| Step | Index size | Entry pages | Metapage `n_entries` | FSM recorded |
|---|---|---|---|---|
| Populated, then VACUUM | 16,801,792 B | 2040 leaf + 10 internal | 300,000 | 0 |
| Delete every row, then VACUUM twice | 16,801,792 B | unchanged | 300,000 | 0 |
| `REINDEX` empty table | 16,384 B | 1 leaf | — | — |

The empty table retained every entry tuple and page, matching the source rule that VACUUM does not delete from the entry tree ([README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396), [ginvacuum.c#ginVacuumEntryPage](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556)).

**Transient entry-tuple growth: `fastupdate = off`, an index created empty, then 800,000 rows of `ARRAY[(g * 17) % 40]`:**

| State | Index size | Entry pages | Posting pages |
|---|---|---|---|
| Grown by retail `INSERT` | 1,802,240 B | 18 leaf + 1 internal | 160 leaf + 40 internal |
| After `REINDEX` | 1,654,784 B | 1 leaf | 160 leaf + 40 internal |

This fixture reclaimed 8.2% even though the posting-tree page count did not change. While a key still uses an in-line posting list, the entry tuple grows; when it no longer fits, GIN creates a posting tree and replaces the tuple with a small tree pointer ([gininsert.c#addItemPointersToLeafTuple](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L42-L118)). Entry pages split during the growth phase but are never merged after the tuples shrink, so this is a second entry-tree retention pattern.

**Sparse posting trees: build an index over 800,000 rows of `ARRAY[g % 40]`, then delete 760,000 rows:**

| Step | Index size | Deleted posting pages | Live posting leaves | FSM recorded |
|---|---|---|---|---|
| Populated, then VACUUM | 1,654,784 B | 0 | 160 | 0 |
| Delete 95%, then VACUUM | 1,654,784 B | 80 | 80 | 0 |
| Two more VACUUMs without horizon advancement | 1,654,784 B | 80 | 80 | 0 |
| `REINDEX` | 98,304 B | 0 | 0 | — |

The 40,000 survivors fit in entry tuples after rebuild, but ordinary VACUUM neither demoted the existing posting trees nor shortened the fork ([ginget.c#entryGetItem-no-demotion](../../../raw/postgres-12/src/backend/access/gin/ginget.c#L405-L410), [ginvacuum.c#ginvacuumcleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L738-L794)). A separate 120,000-row one-key fixture produced 12 deleted posting pages with zero FSM records; after three new transactions advanced the isolated cluster's horizon, the next VACUUM recorded all 12 while their physical flags remained `deleted`. The general gate is `RecentGlobalXmin`, not a fixed VACUUM or transaction count ([ginblock.h#GinPageIsRecyclable](../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138)).

**Planner effect: 300,000 rows of `ARRAY[g, 1]`, a 1 GB per-index pending limit, and `enable_seqscan = off` only to expose the index path:**

| State | Live blocks | Pending pages | `Bitmap Index Scan` cost | Index buffers |
|---|---|---|---|---|
| Pending list intact | 1473 | 1471 | `0.00..5903.25` | 1473 |
| After `gin_clean_pending_list` | 3563 | 0 | `0.00..27.25` | 4 |
| After VACUUM | 3563 | 0 | `0.00..27.25` | 4 |

Both costs reproduce from `gincostestimate` in the invented-statistics branch. Before cleanup: `numEntryPages = floor((1473 - 1471) * 0.90) = 1`; `entryPagesFetched = 1471 + 1 = 1472`; startup I/O is `1472 * 4 = 5888`; one data page costs 4; and tuple CPU is `1500 * (0.005 + 0.0025) = 11.25`, totaling 5903.25. After cleanup: `floor(3563 * 0.90) = 3206` estimated entry pages, the power formula rounds to 3 fetched entry pages, so `12 + 4 + 11.25 = 27.25` ([selfuncs.c#gincostestimate-stats-trust](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6716-L6770), [selfuncs.c#gincostestimate-cost-formula](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6869-L6978)). VACUUM then stored `n_total_pages 3563`, `n_entry_pages 2050`, `n_data_pages 41`, and `n_entries 300000`; the trusted-stats path produced the same index cost.

**Turning fast update off does not flush:** after 100,000 rows of `ARRAY[g]`, `pending_pages` stayed at 246 across `ALTER INDEX ... SET (fastupdate = off)`, and `gin_clean_pending_list` returned 246 before the count became zero. This matches the documented reloption behavior ([create_index.sgml#fastupdate](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L459-L467)).

All five measurement snippets ran verbatim with `ON_ERROR_STOP` against the pin. The concurrent probe built and dropped successfully. The page census separately confirmed that a former pending-list page is `deleted` without `data` and immediately FSM-recorded, whereas a deleted posting-tree page retains `data` and may be either absent from or present in the FSM. The isolated server was stopped after validation; disposable SQL, logs, clusters, and databases remain under `.wiki-runtime/`.

### Key data structures

| Structure | Where | Role in bloat |
|---|---|---|
| `GinMetaPageData` | [ginblock.h#GinMetaPageData](../../../raw/postgres-12/src/include/access/ginblock.h#L54-L102) | pending head/tail plus the four last-VACUUM counters |
| `GinPageOpaqueData` | [ginblock.h#GinPageOpaqueData](../../../raw/postgres-12/src/include/access/ginblock.h#L29-L36) | per-page `flags` that the census reads; `maxoff` is class-specific |
| `GinStatsData` | [gin.h#GinStatsData](../../../raw/postgres-12/src/include/access/gin.h#L38-L49) | the planner's view; it has `nPendingPages` but no `nPendingHeapTuples` field |
| `GinOptions` | [gin_private.h#GinOptions](../../../raw/postgres-12/src/include/access/gin_private.h#L21-L39) | `fastupdate` and the per-index pending limit |
| `disassembledLeaf` / `leafSegmentInfo` | [gindatapage.c#disassembledLeaf](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L45-L103) | split bookkeeping (`lastleft`, `lsize`, `rsize`) that decides leaf fill |
| `GinPostingList` | [ginblock.h#GinPostingList](../../../raw/postgres-12/src/include/access/ginblock.h#L334-L347) | varbyte-compressed segment; min/target/max 128/256/384 bytes ([gindatapage.c#GinPostingListSegmentMaxSize](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L25-L36)) |
| `GinIndexStat` | [pgstatindex.c#GinIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L97-L108) | the three fields `pgstatginindex` will return |
| `IndexBulkDeleteResult` | consumed at [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827) | `pages_deleted` / `pages_free`, visible only in `VACUUM VERBOSE` |

### Caller and callee boundary

Insert side: `ExecInsertIndexTuples` forms the index values and calls `index_insert` ([execIndexing.c#ExecInsertIndexTuples](../../../raw/postgres-12/src/backend/executor/execIndexing.c#L273-L400)); `index_insert` dispatches through `aminsert` ([indexam.c#index_insert](../../../raw/postgres-12/src/backend/access/index/indexam.c#L169-L188)), which GIN registers as `gininsert` ([ginutil.c#ginhandler](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L58-L62)). `gininsert` then chooses `ginHeapTupleFastCollect` plus `ginHeapTupleFastInsert` for fast update, or `ginHeapTupleInsert` plus `ginEntryInsert` for direct insertion ([gininsert.c#gininsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L484-L537)). Main-fork allocation goes through `GinNewBuffer`, which validates FSM candidates before extending ([ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335)).

VACUUM side: when index cleanup is enabled and the heap scan collected dead TIDs, `lazy_vacuum_index` dispatches to GIN's `ginbulkdelete`; the final `lazy_cleanup_index` dispatches to `ginvacuumcleanup` ([vacuumlazy.c#lazy_vacuum_all_indexes](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1419-L1440), [vacuumlazy.c#lazy_cleanup_index-loop](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1466-L1471), [indexam.c#index_vacuum_cleanup](../../../raw/postgres-12/src/backend/access/index/indexam.c#L702-L712), [ginutil.c#ginhandler](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L58-L63)). The `useindex` gate requires at least one index and enabled cleanup; the table reloption defaults to enabled but `VACUUM (INDEX_CLEANUP FALSE)` can disable it ([vacuumlazy.c#lazy_vacuum_rel-useindex](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L289), [vacuum.c#vacuum-index-cleanup-default](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1770-L1780)). Inside `ginbulkdelete`: pending cleanup, a left-to-right walk of entry leaves through `ginVacuumEntryPage`, then `ginVacuumPostingTree` for each collected posting-tree root ([ginvacuum.c#ginbulkdelete](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L563-L684)).

`lazy_cleanup_index` also gates the catalog update: when VACUUM skipped any heap page, `estimated_count` is true and `pg_class.relpages`/`reltuples` for the index are not refreshed at all ([vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1784-L1815)).

Planner side: `cost_index` -> `IndexOptInfo.amcostestimate` -> `gincostestimate` -> `ginGetStats` ([costsize.c#cost_index-amcostestimate](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L537-L548), [selfuncs.c#gincostestimate-ginGetStats](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6700-L6714)).

### Build, generated-header, and extension boundary

GIN-specific type gates in `gin_clean_pending_list`, `pgstatginindex`, and `pgstattuple` compare `relam` with `GIN_AM_OID`. That symbol is declared as an `oid_symbol` in the catalog data file ([pg_am.dat:27-29](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L27-L29)) and emitted into generated `pg_am_d.h` by `genbki.pl` ([genbki.pl#oid_symbol-emit](../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603)); the backend and catalog makefiles drive that generated-header step ([Makefile#generated-headers](../../../raw/postgres-12/src/backend/Makefile#L160-L162), [catalog/Makefile#generated-header-symlinks](../../../raw/postgres-12/src/backend/catalog/Makefile#L71-L90)). `pg_am_d.h` is a build artifact excluded by [catalog/.gitignore](../../../raw/postgres-12/src/include/catalog/.gitignore#L1-L3), so the generated `#define` is not present in the pinned checkout.

Consumers of that symbol: `gin_clean_pending_list` ([ginfast.c#gin_clean_pending_list-am-check](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1044-L1050)), `pgstatginindex`'s `IS_GIN` macro ([pgstatindex.c#IS_GIN](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L73)), and `pgstattuple`'s rejection switch ([pgstattuple.c#pgstat_relation](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276)).

The other measurement paths are different. Core size functions do not need a GIN type gate. `pageinspect` receives raw `bytea`; `gin_metapage_info` validates a metapage flag, but `gin_page_opaque_info` decodes the special area without checking the source relation or page type ([ginfuncs.c#gin_metapage_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L34-L59), [ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L90-L111)). `pg_freespacemap` reads recorded FSM state without checking the access method ([pg_freespacemap.c#pg_freespace](../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c#L17-L41)).

Extension boundary: detailed GIN inspection uses three separately installed contrib modules. `pgstattuple` added `pgstatginindex` in extension version 1.1 ([pgstattuple--1.0--1.1.sql:6-11](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.0--1.1.sql#L6-L11)) and its default 1.5 definition grants execution to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql:49-57](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57)). `pageinspect` added its GIN trio in version 1.3 ([pageinspect--1.2--1.3.sql:46-82](../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.2--1.3.sql#L46-L82)). `pg_freespacemap` exposes the FSM. `amcheck` has no GIN verifier ([verify_nbtree.c#btree_index_checkable](../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323)).

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

### Follow-up: measuring wasted bytes with core SQL only

Define **wasted bytes** as the bytes the index occupies beyond what rebuilding the same index over the same table contents right now would occupy. That is the number `REINDEX` can hand back, and it is the only definition core SQL can be held to, because nothing in core reports free space *inside* a GIN page.

With no contrib extension installed, v12 gives you three usable methods. Only the first is exact.

| # | Method | Result | Writes? | Cost | Measured accuracy at the pin |
|---|---|---|---|---|---|
| A | Build a duplicate index and subtract | **Exact** | Creates and drops a second index | One full index build, plus the probe's disk | `fresh_bytes` matched the later `REINDEX` size byte-for-byte on all five fixtures |
| B | Compare against a bytes-per-row baseline you recorded after the last rebuild | Estimate | Reads only; you maintain one small table | Two catalog lookups | Within **2.75 percentage points** of truth on all four fixtures across two churn rounds; exactly 0.00 on the untouched control |
| C | Build a GIN index over a `TABLESAMPLE` subset and extrapolate | **Unusable** | Creates a temp table and index | A fraction of one build | Off by **+17% to +455%** on fresh size; reported −384% bloat on a 0%-bloated index |

Use A when you can afford a build and need a number you will act on. Use B for routine monitoring. Do not use C; the measurements are below so the failure is on the record rather than assumed.

The v12 documentation frames the same limitation: "The potential for bloat in non-B-tree indexes has not been well researched. It is a good idea to periodically monitor the index's physical size when using any non-B-tree index type." ([maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L876-L880))

### Why core SQL cannot see inside a GIN index

Three separate gaps, each verifiable in the pinned tree.

1. **No core function reads an index page.** `pg_proc.dat` contains no raw-page, page-header, metapage, or FSM-contents function; the GIN inspectors are contrib (`gin_metapage_info`, `gin_page_opaque_info`, `gin_leafpage_items` at [ginfuncs.c#gin_metapage_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L34-L87), [ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L90-L154), [ginfuncs.c#gin_leafpage_items](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L163-L265)), as is `pg_freespace` ([pg_freespacemap.c#pg_freespace](../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c#L17-L41)). The one general-purpose escape hatch, `pg_read_binary_file` ([genfile.c#pg_read_binary_file](../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L288-L300)), has no page-format awareness, bypasses shared buffers, and needs elevated privileges; it is a boundary, not a recipe.
2. **The GIN metapage counters are C-only.** `GinStatsData` carries `nTotalPages`, `nEntryPages`, `nDataPages`, and `nEntries` ([gin.h#GinStatsData](../../../raw/postgres-12/src/include/access/gin.h#L38-L49)); `ginGetStats` reads them ([ginutil.c#ginGetStats](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L657)), and its only callers are the planner and GIN's own scan setup ([selfuncs.c#gincostestimate-ginGetStats](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6700-L6714)). No core SQL wrapper exists.
3. **No statistic counts GIN keys.** `pg_statistic` has five generic slots ([pg_statistic.h#STATISTIC_NUM_SLOTS](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L120-L126)) and none holds a column-wide distinct-*element* cardinality. `array_typanalyze` gets closest: the last member of the distinct-element-count histogram is the average number of distinct non-null elements per row ([array_typanalyze.c#compute_array_stats-avg-count](../../../raw/postgres-12/src/backend/utils/adt/array_typanalyze.c#L605-L614), specified at [pg_statistic.h#STATISTIC_KIND_DECHIST](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L236-L249), surfaced as `pg_stats.elem_count_histogram` at [system_views.sql#pg_stats-elem_count_histogram](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L240-L246) and documented at [catalogs.sgml#elem_count_histogram](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10921-L10930)). That gives *postings per row*, not distinct keys, and it is array-only: `ts_typanalyze` fills a most-common-elements slot and nothing else ([ts_typanalyze.c#compute_tsvector_stats-mcelem](../../../raw/postgres-12/src/backend/tsearch/ts_typanalyze.c#L415-L426)), and `jsonb` declares no `typanalyze` at all ([pg_type.dat:438-442](../../../raw/postgres-12/src/include/catalog/pg_type.dat#L438-L442)), so it falls through to `std_typanalyze` ([analyze.c#typanalyze-dispatch](../../../raw/postgres-12/src/backend/commands/analyze.c#L955-L963)).

So a self-contained absolute estimate is out of reach. Every core-only number below is either a measured difference against a real fresh build, or a difference against a fresh build you measured earlier.

Two structural facts explain why a modeled "ideal" size would not help even if the key count were known. A GIN entry-tree split equalizes bytes across the two output pages with no build-time or append special case ([ginentrypage.c#entrySplitPage-balance](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L666-L689)), so even a fresh entry tree is not densely packed; and posting-list bytes per heap TID are data-dependent, ranging from 1 to 7 bytes of varbyte delta plus an 8-byte per-segment header ([ginpostinglist.c#varbyte-format](../../../raw/postgres-12/src/backend/access/gin/ginpostinglist.c#L23-L72), [ginpostinglist.c#MaxBytesPerInteger](../../../raw/postgres-12/src/backend/access/gin/ginpostinglist.c#L83-L84), [ginblock.h#GinPostingList](../../../raw/postgres-12/src/include/access/ginblock.h#L334-L346)), with segments held between 128 and 384 bytes ([gindatapage.c#GinPostingListSegmentMaxSize](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L25-L36)).

### What core SQL does expose

| Core object | GIN? | What it gives you |
|---|---|---|
| `pg_relation_size(regclass)` | Yes | main-fork bytes; the one-argument form is SQL over the two-argument form with `'main'` ([pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6891)) |
| `pg_relation_size(regclass, text)` | Yes | one named fork: `main`, `fsm`, `vm`, or `init` ([relpath.c#forkNames](../../../raw/postgres-12/src/common/relpath.c#L33-L38)), sized by `stat()` over every segment file ([dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308)) |
| `pg_table_size` / `pg_total_relation_size` | Yes | all forks of the index; the sizing code notes it "behaves sanely if applied to an index" ([dbsize.c#calculate_table_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L380-L408)) |
| `pg_indexes_size` on an index | Yes | always 0 — "Can be applied safely to an index, but you'll just get zero" ([dbsize.c#calculate_indexes_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L410-L414)) |
| `pg_class.relpages` (index) | Yes | block count as of the last stats refresh ([pg_class.h#relpages](../../../raw/postgres-12/src/include/catalog/pg_class.h#L57-L63), [catalogs.sgml#pg_class.relpages](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1759-L1770)) |
| `pg_class.reltuples` (index) | Yes, but | three different meanings — see [The reltuples trap](#the-reltuples-trap) |
| `pg_stats.elem_count_histogram` | Array columns only | average distinct elements per row in its last element ([catalogs.sgml#elem_count_histogram](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10921-L10930)) |
| `gin_clean_pending_list(regclass)` | Yes | the number of pending-list pages it removed ([pg_proc.dat#gin_clean_pending_list](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L8639-L8642), [ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1030-L1074)) — the one GIN internal core SQL will report, and it mutates the index |
| `EXPLAIN` on a GIN scan | Yes | a pending-page proxy, via the cost model — see [Core-only pending-list probes](#core-only-pending-list-probes) |
| `VACUUM (VERBOSE)` | Yes | `pages_deleted` and `pages_free` as server messages, not queryable rows ([vacuumlazy.c#lazy_cleanup_index-ereport](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1817-L1827)) |

`pg_relation_size` is exact, not an estimate: it `stat()`s the files. It also works on temporary relations, which method C relies on — "we can safely apply this to temp tables of other sessions, so there is no check here" ([dbsize.c#calculate_relation_size-temp](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L271)).

### Core method A: rebuild probe, exact

Build a second index with the same definition, subtract, drop it. This is the same rebuild comparison as [Measurement recipe 5](#measurement-recipe-5-ground-truth-by-rebuilding), expressed as wasted bytes and a percentage.

```sql
SET /* wiki_gin_waste_probe_statement_timeout */ statement_timeout = '30min';
SET /* wiki_gin_waste_probe_lock_timeout */ lock_timeout = '5s';

SELECT /* wiki_gin_waste_probe_definition */
       pg_get_indexdef('public.arr_churn_gin'::regclass);

CREATE /* wiki_gin_waste_probe_build */ INDEX CONCURRENTLY arr_churn_gin_probe
  ON public.arr_churn USING gin (tags) WITH (fastupdate = off);

SELECT /* wiki_gin_waste_probe_result */
       'arr_churn_gin'                                          AS index_name,
       pg_relation_size('public.arr_churn_gin')                 AS current_bytes,
       pg_relation_size('public.arr_churn_gin_probe')            AS fresh_bytes,
       pg_relation_size('public.arr_churn_gin')
         - pg_relation_size('public.arr_churn_gin_probe')        AS wasted_bytes,
       pg_size_pretty(pg_relation_size('public.arr_churn_gin')
         - pg_relation_size('public.arr_churn_gin_probe'))       AS wasted_pretty,
       round(100.0 * (pg_relation_size('public.arr_churn_gin')
                      - pg_relation_size('public.arr_churn_gin_probe'))
             / greatest(pg_relation_size('public.arr_churn_gin'), 1), 2)
                                                                AS bloat_pct;

DROP /* wiki_gin_waste_probe_drop */ INDEX CONCURRENTLY public.arr_churn_gin_probe;
```

Reproduce the original definition from `pg_get_indexdef`: columns or expressions, operator class, collation, predicate, and both GIN reloptions all move the size. `CREATE INDEX CONCURRENTLY` cannot run inside a transaction block ([utility.c#CREATE-INDEX-CONCURRENTLY](../../../raw/postgres-12/src/backend/tcop/utility.c#L1299-L1311)) and does more total work than a plain build while permitting concurrent writes ([create_index.sgml#Building-Indexes-Concurrently](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L527-L558)), so the two sizes are read at slightly different times rather than from one snapshot. Both indexes occupy storage until the probe is dropped. A plain `CREATE INDEX` is cheaper but takes `ShareLock` on the table instead of `ShareUpdateExclusiveLock`, so only `SELECT ... FOR UPDATE/SHARE` proceeds and ordinary writers block ([indexcmds.c#DefineIndex-lockmode](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L548-L564)).

Set `maintenance_work_mem` the same way for the probe as for your eventual real rebuild: the build accumulator flushes whenever it reaches that budget ([gininsert.c#ginBuildCallback-flush](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L290-L311)), and the documentation calls GIN build time "very sensitive" to it ([gin.sgml#gin-tips-maintenance_work_mem](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L554-L563)).

At the pin, the probe's `fresh_bytes` equalled the size the index actually had after a later `REINDEX`, for all five fixtures, to the byte. That is the strongest statement available: the probe does not approximate the rebuild, it reproduces it.

### Core method B: recorded baseline bytes per row

The read-only method. Record `pg_relation_size` and the table's live row count once, right after a rebuild; afterwards, scale the baseline by the current row count and subtract.

```sql
CREATE /* wiki_gin_baseline_table */ TABLE public.wiki_gin_baseline (
  index_oid       oid PRIMARY KEY,
  index_name      text NOT NULL,
  recorded_at     timestamptz NOT NULL DEFAULT now(),
  baseline_bytes  bigint NOT NULL,
  baseline_rows   double precision NOT NULL
);
```

Refresh a row immediately after every `REINDEX` or `REINDEX CONCURRENTLY` of that index, with the table's statistics current:

```sql
SET /* wiki_gin_baseline_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_baseline_lock_timeout */ lock_timeout = '5s';

INSERT /* wiki_gin_record_baseline */ INTO public.wiki_gin_baseline
       (index_oid, index_name, baseline_bytes, baseline_rows)
SELECT c.oid, c.relname, pg_relation_size(c.oid), t.reltuples
FROM pg_class c
JOIN pg_index i ON i.indexrelid = c.oid
JOIN pg_class t ON t.oid = i.indrelid
WHERE c.relname = 'arr_churn_gin'
ON CONFLICT (index_oid) DO UPDATE
   SET index_name     = excluded.index_name,
       recorded_at    = now(),
       baseline_bytes = excluded.baseline_bytes,
       baseline_rows  = excluded.baseline_rows;
```

Then read wasted bytes and the percentage at any time, with no writes to the index:

```sql
SET /* wiki_gin_waste_ro_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_waste_ro_lock_timeout */ lock_timeout = '5s';

SELECT /* wiki_gin_waste_from_baseline */
       b.index_name,
       pg_size_pretty(pg_relation_size(b.index_oid))                    AS current_size,
       pg_size_pretty(est.fresh_bytes)                                  AS modeled_fresh_size,
       pg_relation_size(b.index_oid) - est.fresh_bytes                  AS wasted_bytes,
       round(100.0 * (pg_relation_size(b.index_oid) - est.fresh_bytes)
             / greatest(pg_relation_size(b.index_oid), 1), 2)           AS bloat_pct,
       b.recorded_at                                                    AS baseline_recorded_at
FROM public.wiki_gin_baseline b
JOIN pg_index i ON i.indexrelid = b.index_oid
JOIN pg_class t ON t.oid = i.indrelid
CROSS JOIN LATERAL (
  SELECT round(b.baseline_bytes / b.baseline_rows * greatest(t.reltuples, 0))::bigint
) AS est(fresh_bytes)
WHERE b.baseline_rows > 0
ORDER BY 5 DESC, 1;
```

Why it works, and where it breaks:

- The denominator is the **table's** `reltuples`, which has one stable meaning: live rows, refreshed by `VACUUM`, `ANALYZE`, and a few DDL commands ([catalogs.sgml#pg_class.reltuples](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1772-L1782), written by [vacuum.c#vac_update_relstats](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1180-L1191)). Using the *index's* `reltuples` would be wrong; see the next section.
- `WHERE b.baseline_rows > 0` is load-bearing: a baseline taken on an empty table divides by zero. `greatest(t.reltuples, 0)` keeps a never-analyzed table from producing a negative modeled size.
- It assumes the number of extracted keys per row and their cardinality have not changed since the baseline. Rewriting the indexed column with a different key distribution invalidates the baseline, not the method; re-record after the next rebuild.
- It is only as fresh as `t.reltuples`. VACUUM skips its `pg_class` update for the *index* when it skipped any heap page ([vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1803-L1815), gate at [vacuumlazy.c:1787](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1787)), and plain `ANALYZE` samples rather than counts. At the pin the post-VACUUM table estimate was 171,433 against 171,429 real live rows, and `ANALYZE` corrected it to 171,429.
- `pg_size_pretty` accepts a negative argument, so an over-estimated baseline prints legibly rather than failing; at the pin `pg_size_pretty((-123456)::bigint)` returned `-121 kB`. A negative `wasted_bytes` means the baseline is stale, not that the index shrank below a fresh build.

A useful companion signal, also read-only, is growth since the last statistics refresh. `relpages` is `RelationGetNumberOfBlocks` at that moment, so the gap against the live size is new space:

```sql
SET /* wiki_gin_growth_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_growth_lock_timeout */ lock_timeout = '5s';

SELECT /* wiki_gin_pages_since_stats */
       c.relname AS index_name,
       c.relpages AS pages_at_last_stats_update,
       (pg_relation_size(c.oid) / current_setting('block_size')::bigint) AS pages_now,
       (pg_relation_size(c.oid) / current_setting('block_size')::bigint) - c.relpages
         AS pages_added,
       pg_size_pretty(pg_relation_size(c.oid, 'fsm')) AS fsm_fork_size
FROM pg_class c
WHERE c.relkind = 'i'
  AND c.relam = (SELECT oid FROM pg_am WHERE amname = 'gin')
  AND c.relpersistence <> 't'
ORDER BY 4 DESC, 1;
```

This is a growth detector, not a waste measurement: legitimate inserts move it too. `relpersistence <> 't'` keeps other sessions' temporary indexes out. A nonzero `fsm_fork_size` means VACUUM or a pending-list clean has already recorded reusable pages inside the existing file.

### The reltuples trap

Do not normalize a GIN index by its own `pg_class.reltuples`. The value means three different things depending on which command last touched it.

| Last operation | Index `reltuples` holds | Evidence |
|---|---|---|
| `CREATE INDEX` / `REINDEX` | total **extracted keys**, counted with duplicates across rows | [gininsert.c#ginBuildCallback-indtuples](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L262-L270), returned at [gininsert.c#ginbuild-index_tuples](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L419-L427), stored by [index.c#index_build-index_update_stats](../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2986) |
| `VACUUM` | the **heap** live tuple count | [ginvacuum.c#ginvacuumcleanup-num_index_tuples](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L725-L731) |
| `ANALYZE` (without VACUUM) | `ceil(tupleFract * totalrows)`, and `tupleFract` is 1.0 for a non-partial index | [analyze.c#do_analyze_rel-index-stats](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629), [analyze.c:438](../../../raw/postgres-12/src/backend/commands/analyze.c#L438) |
| `VACUUM` that skipped heap pages | unchanged, because `estimated_count` is true | [vacuumlazy.c:1787](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1787), [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1803-L1815) |

GIN's own comment explains why the VACUUM number is a heap count: "we always report the heap tuple count as the number of index entries. This is bogus if the index is partial, but it's real hard to tell how many distinct heap entries are referenced by a GIN index." ([ginvacuum.c#ginvacuumcleanup-num_index_tuples](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L725-L731))

Measured at the pin on the same four indexes, immediately after `REINDEX` and then after `VACUUM (ANALYZE)`:

| Index | after `REINDEX` | after `VACUUM (ANALYZE)` |
|---|---|---|
| `arr_churn_gin` | 799,493 | 266,667 |
| `arr_fresh_gin` | 1,198,764 | 400,000 |
| `arr_pending_gin` | 598,885 | 200,000 |
| `ts_docs_gin` | 1,050,000 | 150,000 |

A bytes-per-index-tuple metric therefore jumps by the keys-per-row factor at every rebuild. The table's `reltuples` does not.

### Core method C: sampled probe, rejected

The idea is to build a GIN index over a `TABLESAMPLE` subset in a temporary table and scale bytes per row up to the full table. It is legal, cheap, and wrong.

```sql
SET /* wiki_gin_sample_statement_timeout */ statement_timeout = '10min';
SET /* wiki_gin_sample_lock_timeout */ lock_timeout = '5s';

CREATE /* wiki_gin_sample_copy */ TEMP TABLE gin_probe_sample AS
  SELECT * FROM public.arr_churn TABLESAMPLE SYSTEM (5) REPEATABLE (42);

CREATE /* wiki_gin_sample_build */ INDEX gin_probe_sample_idx
  ON gin_probe_sample USING gin (tags);

SELECT /* wiki_gin_sample_extrapolate */
       n.sample_rows,
       pg_relation_size('gin_probe_sample_idx')                      AS sample_bytes,
       t.reltuples                                                   AS table_rows,
       round(pg_relation_size('gin_probe_sample_idx')::numeric
             * t.reltuples / n.sample_rows)                          AS extrapolated_fresh_bytes
FROM (SELECT count(*) AS sample_rows FROM gin_probe_sample) n
CROSS JOIN pg_class t
WHERE t.oid = 'public.arr_churn'::regclass
  AND n.sample_rows > 0;

DROP /* wiki_gin_sample_drop */ TABLE gin_probe_sample;
```

The mechanics are sound. `TABLESAMPLE` applies to plain tables, matviews, and partitioned tables ([parse_clause.c#transformRangeTableSample-relkind](../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1160-L1169)); `SYSTEM` picks whole blocks and `BERNOULLI` flips a coin per row, with `SYSTEM` faster but "less-random ... as a result of clustering effects" ([select.sgml#TABLESAMPLE-methods](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L404-L423), block-level behaviour at [system.c#system_nextsampletuple](../../../raw/postgres-12/src/backend/access/tablesample/system.c#L224-L235), per-row at [bernoulli.c#bernoulli_nextsampletuple](../../../raw/postgres-12/src/backend/access/tablesample/bernoulli.c#L169-L179)). An index on a temporary table is automatically temporary ([create_table.sgml#TEMPORARY-indexes](../../../raw/postgres-12/doc/src/sgml/ref/create_table.sgml#L173-L178)), and no GIN-specific restriction applies; `CONCURRENTLY` would simply be downgraded ([indexcmds.c#DefineIndex-temp-nonconcurrent](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L489-L499)).

The estimator is what fails: **GIN size is strongly sublinear in row count**, so bytes per row measured on a sample is far above the full-table value. Three source-level reasons:

1. Below the promotion threshold, a key's TIDs live in-line in its entry tuple; once the compressed list no longer fits `GinMaxItemSize` (2712 bytes at an 8 kB block size), GIN moves them to a posting tree and leaves a small pointer behind ([ginblock.h#GinMaxItemSize](../../../raw/postgres-12/src/include/access/ginblock.h#L245-L256), [gininsert.c#addItemPointersToLeafTuple](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L42-L119)). A small sample keeps most keys on the expensive side of that transition.
2. Distinct keys saturate while rows grow, so the entry tree's share shrinks with scale — and the entry tree is the part that a 50/50 split leaves half-empty ([ginentrypage.c#entrySplitPage-balance](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L666-L689)).
3. Segment overhead amortizes: every compressed segment pays an 8-byte header and is kept at least 128 bytes ([ginblock.h#GinPostingList](../../../raw/postgres-12/src/include/access/ginblock.h#L334-L346), [gindatapage.c#GinPostingListSegmentMaxSize](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L25-L36)), which a short posting list cannot fill efficiently.

Measured on `arr_fresh_gin`, an index with **0.00% true bloat**, bytes per row fell monotonically with sample size: 64.5 at 3,430 rows, 29.4 at 20,090, 21.5 at 40,320, 16.6 at 100,730, and 13.3 at the full 400,000. Extrapolating from any of those overstates the fresh size, so the method reports large negative bloat on a perfectly packed index:

| Sample | Sample rows | Sample bytes | Extrapolated fresh | True fresh | Fresh error | Reported bloat (true 0.00%) |
|---|---|---|---|---|---|---|
| 1% | 3,430 | 221,184 | 25,794,052 | 5,332,992 | +383.67% | −383.67% |
| 5% | 20,090 | 589,824 | 11,743,634 | 5,332,992 | +120.21% | −120.21% |
| 10% | 40,320 | 868,352 | 8,614,603 | 5,332,992 | +61.53% | −61.53% |
| 25% | 100,730 | 1,671,168 | 6,636,228 | 5,332,992 | +24.44% | −24.44% |

On the genuinely bloated fixtures the same bias masks real waste. `arr_churn_gin` (true 72.14%) read −54.64% / 29.39% / 50.75% / 64.10% at 1/5/10/25%; `arr_pending_gin` (true 75.73%) read −34.47% / 33.00% / 52.59% / 66.71%; `ts_docs_gin` (true 24.17%) read −246.53% / −67.19% / −27.31% / 11.02%. Copying only the indexed column instead of `SELECT *` changed `arr_churn_gin`'s answer by roughly two points (31.87% / 52.55% / 66.07%), which is noise next to the sublinearity error. A 5% sample of the all-rows-deleted `arr_uniq` returned zero rows, leaving the estimate undefined.

Sampling 25% of the table to land 24 points wrong is worse than method A, which samples 100% and is exact.

### Which size to divide by

Pick one size function and use it on both sides of the subtraction.

- `pg_relation_size(idx)` counts the **main fork only** ([pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6891)). This is the right denominator for "bytes a rebuild reclaims", because the main fork is what GIN grows and never truncates ([ginvacuum.c#ginvacuumcleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L738-L794)).
- `pg_table_size(idx)` and `pg_total_relation_size(idx)` add the FSM fork ([dbsize.c#calculate_table_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L380-L408)). Every index except hash has one ([storage.sgml#Free-Space-Map](../../../raw/postgres-12/doc/src/sgml/storage.sgml#L600-L607)), and an index has no visibility map ([storage.sgml#relation-forks](../../../raw/postgres-12/doc/src/sgml/storage.sgml#L204-L216)).
- `pg_indexes_size(idx)` is always 0 and is not a mistake worth making twice ([dbsize.c#calculate_indexes_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L410-L414)).

The forks are not interchangeable in practice. At the pin, four of the five GIN indexes had a 0-byte FSM fork, so `pg_relation_size`, `pg_table_size`, and `pg_total_relation_size` returned identical values. After `gin_clean_pending_list` recorded 883 removed pending pages, `arr_pending_gin` reported main 11,247,616 bytes, FSM 24,576 bytes, and `pg_table_size` 11,272,192 bytes — a 0.2% difference that would silently show up as extra "waste" if the two sides of the subtraction used different functions.

### Core-only pending-list probes

When `fastupdate` is on, most of a GIN index's excess can be pending-list pages, and core SQL can size them two ways without contrib.

**Read-only, via the planner.** `gincostestimate` starts its entry-page fetch count at the metapage's pending-page count and prices those pages at `random_page_cost` ([selfuncs.c#gincostestimate-pending](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6869-L6873), [selfuncs.c#gincostestimate-startup](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6922-L6926)), then folds that into the index total cost ([selfuncs.c#gincostestimate-total](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6964-L6977)). GIN has no `amgettuple` ([ginutil.c#ginhandler](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L71-L72)), so it always plans as a `Bitmap Index Scan`, whose displayed startup cost is forced to zero and whose total cost is exactly `indextotalcost` ([createplan.c#create_bitmap_subplan-cost](../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L3214-L3216)). So the node's total cost divided by `random_page_cost` is an upper-bound proxy for pending pages, and it is tight when the pending list dominates:

```sql
SET /* wiki_gin_pending_probe_statement_timeout */ statement_timeout = '30s';
SET /* wiki_gin_pending_probe_lock_timeout */ lock_timeout = '5s';
SET /* wiki_gin_pending_probe_seqscan */ enable_seqscan = off;

EXPLAIN /* wiki_gin_pending_probe */ (COSTS ON)
SELECT count(*) FROM public.arr_pending WHERE tags @> ARRAY[7];

RESET /* wiki_gin_pending_probe_reset */ enable_seqscan;
```

Measured at the pin, with `random_page_cost = 4`:

| Index state | Live pages | `Bitmap Index Scan` total cost | cost ÷ `random_page_cost` | Actual pending pages |
|---|---|---|---|---|
| `arr_pending_gin`, list intact | 1,471 | 5,886.95 | 1,471.7 | 1,469 |
| `arr_pending_gin`, after more inserts | 1,240 | 3,550.00 | 887.5 | 883 |
| `arr_pending_gin`, after `gin_clean_pending_list` | 1,373 | 21.04 | 5.3 | 0 |
| `arr_fresh_gin`, no pending list | 651 | 22.20 | 5.6 | 0 |
| `arr_churn_gin`, no pending list | 590 | 20.11 | 5.0 | 0 |

Read it as a ratio against the index's page count: 71.6% and 71.2% of pages implied pending on the two loaded `arr_pending_gin` states, against 0.4% for the same index once its list was cleaned and 0.9% and 0.8% on the two indexes that never had one. `enable_seqscan = off` was needed because with the list intact the planner correctly preferred a sequential scan. The proxy is not a measurement: it also carries data-page fetches and per-tuple CPU terms, and it moves with the query's selectivity.

**Destructive, but exact.** `gin_clean_pending_list` returns the number of pending pages it deleted ([ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1030-L1074)). At the pin it returned 883, matching the actual count, and the index then grew from 10,158,080 to 11,247,616 bytes as those entries moved into the trees — so this reports pending pages and reduces future scan cost, but does not shrink the file.

### Accuracy measured at the pin

All numbers come from one isolated PostgreSQL 12.2 server built from the pinned checkout, 8 kB blocks, `autovacuum = off`, `maintenance_work_mem = 64MB`, `random_page_cost = 4`. `pgstattuple` was installed **only** to cross-check the core-only pending-page proxy; no recipe above uses it. Timings are omitted as machine-dependent.

Five fixtures:

| Index | Table contents | Definition | Churn applied |
|---|---|---|---|
| `arr_churn_gin` | 400,000 rows of `ARRAY[g % 1000, g % 997, (g * 7) % 5000]` | `gin (tags) WITH (fastupdate = off)` | 320,000 non-HOT updates, then a third deleted |
| `arr_uniq_gin` | 300,000 rows of `ARRAY[g]` | same | every row deleted |
| `ts_docs_gin` | 200,000 seven-lexeme `tsvector` documents | `gin (body) WITH (fastupdate = off)` | half rewritten, a quarter deleted |
| `arr_fresh_gin` | same generator as `arr_churn`, 400,000 rows | same | none — control |
| `arr_pending_gin` | 200,000 rows inserted after index creation | `gin (tags) WITH (fastupdate = on, gin_pending_list_limit = 1048576)` | pending list never cleaned |

**Method A against a real `REINDEX`.** The probe size and the post-`REINDEX` size were identical in every case:

| Index | Current bytes | Probe `fresh_bytes` | Post-`REINDEX` bytes | Wasted bytes | Bloat % |
|---|---|---|---|---|---|
| `arr_churn_gin` | 13,524,992 | 3,768,320 | 3,768,320 | 9,756,672 | 72.14 |
| `arr_uniq_gin` | 16,801,792 | 16,384 | 16,384 | 16,785,408 | 99.90 |
| `ts_docs_gin` | 4,947,968 | 3,751,936 | 3,751,936 | 1,196,032 | 24.17 |
| `arr_pending_gin` | 12,050,432 | 2,924,544 | 2,924,544 | 9,125,888 | 75.73 |
| `arr_fresh_gin` | 5,332,992 | 5,332,992 | 5,332,992 | 0 | 0.00 |

The control returning exactly 0.00% matters: the method has no systematic offset to subtract.

**Method B against method A.** Baselines were recorded right after those rebuilds, dividing by the table's `reltuples`: 14.1312 bytes/row for `arr_churn_gin`, 13.3325 for `arr_fresh_gin`, 14.6227 for `arr_pending_gin`, 25.0129 for `ts_docs_gin`. Two further churn rounds followed, each ending in `VACUUM (ANALYZE)` or `ANALYZE`, with a fresh probe for truth:

| Round | Index | Current bytes | Est. fresh | True fresh | Est. bloat % | True bloat % | Error (points) |
|---|---|---|---|---|---|---|---|
| 2 | `arr_churn_gin` | 4,833,280 | 3,229,993 | 3,293,184 | 33.17 | 31.86 | +1.31 |
| 2 | `ts_docs_gin` | 4,243,456 | 3,751,936 | 3,768,320 | 11.58 | 11.20 | +0.38 |
| 2 | `arr_pending_gin` | 10,158,080 | 4,679,270 | 4,399,104 | 53.94 | 56.69 | −2.75 |
| 2 | `arr_fresh_gin` | 5,332,992 | 5,332,992 | 5,332,992 | 0.00 | 0.00 | 0.00 |
| 3 | `arr_churn_gin` | 4,833,280 | 2,422,494 | 2,473,984 | 49.88 | 48.81 | +1.07 |
| 3 | `ts_docs_gin` | 4,243,456 | 3,751,936 | 3,768,320 | 11.58 | 11.20 | +0.38 |
| 3 | `arr_pending_gin` | 11,247,616 | 4,679,270 | 4,399,104 | 58.40 | 60.89 | −2.49 |
| 3 | `arr_fresh_gin` | 5,332,992 | 5,332,992 | 5,332,992 | 0.00 | 0.00 | 0.00 |

Worst error across eight comparisons: 2.75 percentage points, on the fixture whose live row count grew 60% past its baseline. The untouched control stayed at exactly 0.00%.

**What `pg_stats` offered.** Only the array columns carried a distinct-element-count histogram, and its last element matched the three-element arrays:

| Column | `n_distinct` | `most_common_elems` length | `elem_count_histogram` length | Average distinct elements per row |
|---|---|---|---|---|
| `arr_churn.tags` | −1 | 1000 | 101 | 2.9983332 |
| `arr_fresh.tags` | −1 | 1000 | 101 | 2.9972334 |
| `arr_pending.tags` | −1 | 1000 | 101 | 2.9947667 |
| `ts_docs.body` | −1 | 818 | (null) | (null) |

The `tsvector` column had no histogram at all, which is the array-only limit from [Why core SQL cannot see inside a GIN index](#why-core-sql-cannot-see-inside-a-gin-index) reproduced.

**The filed SQL, run verbatim.** All eight fenced blocks in this follow-up executed against the pin with `ON_ERROR_STOP` set, in one pass, on the round-3 state. Method A reproduced `arr_churn_gin` at current 4,833,280 bytes, fresh 2,473,984, wasted 2,359,296 (`2304 kB`), 48.81% — the same figures as the round-3 truth table. Method B returned 0.00% because the baseline had just been recorded. The sampled probe drew 8,358 rows into 294,912 bytes and extrapolated 6,048,872 bytes against a true fresh 2,473,984, a further +144.5% overstatement. The isolated server was stopped afterwards; disposable SQL, logs, and clusters remain under `.wiki-runtime/`.

### Settings for the core-only recipes

| Setting | Kind | Apply scope |
|---|---|---|
| `statement_timeout`, `lock_timeout` | GUC, `PGC_USERSET` ([guc.c#statement-and-lock-timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2398)) | **Session/transaction** — no restart, no reload; `SET LOCAL` narrows to the current transaction ([set.sgml#SET-scope](../../../raw/postgres-12/doc/src/sgml/ref/set.sgml#L38-L58)) |
| `maintenance_work_mem` | GUC, `PGC_USERSET` ([guc.c#maintenance_work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2243-L2251)) | **Session/transaction**; keep it identical between the probe build and the eventual real rebuild ([gininsert.c#ginBuildCallback-flush](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L290-L311)) |
| `temp_buffers` | GUC, `PGC_USERSET` ([guc.c#temp_buffers](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2165-L2174)) | **Session/transaction**, but only before the session's first temporary-table access; method C's temp build reads and writes local buffers |
| `random_page_cost` | GUC, `PGC_USERSET` ([guc.c#random_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3217-L3226)) | **Session/transaction**; it is the divisor in the pending-page proxy |
| `enable_seqscan` | GUC, `PGC_USERSET` ([guc.c#enable_seqscan](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L883-L890)) | **Session/transaction**; used only to force the GIN path in the `EXPLAIN` probe, never as a remedy |

None of the recipes needs a restart or a reload, and none needs a `postgresql.conf` change.

### Test coverage for the core-only path

- **`pg_relation_size` is never asserted against a GIN index.** The only GIN use of it in the tree addresses the last block of a page-inspection fixture ([pageinspect/sql/gin.sql:16-19](../../../raw/postgres-12/contrib/pageinspect/sql/gin.sql#L16-L19)), and that file is contrib.
- **No test compares a GIN index's size before and after `REINDEX`,** so method A's core claim has no in-tree regression coverage; the byte-for-byte match above is exact-pin measurement, not a shipped test.
- **No test asserts `pg_class.relpages` or `reltuples` for a GIN index,** so the three-meanings behaviour in [The reltuples trap](#the-reltuples-trap) is established from source plus measurement only.
- **No test builds a GIN index on a temporary table.** The ten `USING gin` sites in `src/test/regress/sql/` are all permanent relations, and `src/test/regress/sql/gin.sql` creates no temporary table ([regress/sql/gin.sql](../../../raw/postgres-12/src/test/regress/sql/gin.sql#L1-L36)). Method C's temp-index step rests on index persistence being inherited from the table rather than on a test.
- **No test asserts a GIN plan cost,** because both GIN `EXPLAIN`s in the regression suite use `(costs off)` ([create_index.sql:283-284](../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L283-L284)). The pending-page proxy therefore has no shipped coverage either.
- **`pg_stats.elem_count_histogram` is exercised for arrays** in the statistics regression tests, but no test connects it to GIN sizing.

## Context Reviewed

- GIN access method: `ginfast.c`, `ginvacuum.c`, `gindatapage.c`, `ginentrypage.c`, `ginutil.c`, `gininsert.c`, `ginscan.c`, `ginget.c`, `ginpostinglist.c`, `ginxlog.c`, plus `gin.h`, `gin_private.h`, `ginblock.h`, `ginxlog.h`, and `src/backend/access/gin/README`.
- Executor, utility, index-AM, VACUUM, and horizon integration: `execIndexing.c`, `utility.c`, `indexam.c`, `vacuumlazy.c`, `vacuum.c`, `analyze.c`, `procarray.c`, `indexfsm.c`, `freespace.c`.
- Physical rebuild and truncation paths: `index.c`, `indexcmds.c`, `cluster.c`, `tablecmds.c`, `heap.c`.
- Planner: `selfuncs.c` (`gincostestimate`), `plancat.c`, `costsize.c`, `subselect.c`, `pathnodes.h`.
- Settings: `guc.c`, `reloptions.c`, `postgresql.conf.sample`.
- Measurement tooling: `contrib/pgstattuple` (`pgstatindex.c`, `pgstattuple.c`, `pgstatapprox.c`, install/upgrade scripts, tests), `contrib/pageinspect` (`ginfuncs.c`, `rawpage.c`, scripts, tests), `contrib/pg_freespacemap`, `contrib/amcheck` (`verify_nbtree.c`).
- Catalog/generated-header boundary: `pg_am.dat`, `pg_proc.dat`, `pg_am.h`, `genbki.pl`, `src/backend/Makefile`, `src/backend/catalog/Makefile`, `src/include/catalog/.gitignore`.
- Documentation: `gin.sgml`, `maintenance.sgml`, `ref/create_index.sgml`, `ref/reindex.sgml`, `pgstattuple.sgml`, `pageinspect.sgml`, `pgfreespacemap.sgml`, `monitoring.sgml`, `func.sgml`, `catalogs.sgml`, `storage.sgml`.
- Tests: `src/test/regress/sql/gin.sql`, `create_index.sql`, `tsearch.sql`, `vacuum.sql`, `src/test/isolation/specs/predicate-gin.spec`, `contrib/pgstattuple/sql`, `contrib/pageinspect/sql/gin.sql`, `contrib/amcheck/sql`.
- Exact-pin execution on isolated 12.2 servers built from `raw/postgres-12/`, including the focused review cluster under `.wiki-runtime/pg12-gin-review-data-20260803`.
- Core-only measurement follow-up: core size functions and their fork coverage (`dbsize.c`, `relpath.c`, `pg_proc.dat`), `pg_class` size/row statistics and every writer of them (`pg_class.h`, `vacuum.c`, `vacuumlazy.c`, `analyze.c`, `index.c`, `gininsert.c`, `ginvacuum.c`), column statistics for GIN-indexable types (`pg_statistic.h`, `system_views.sql`, `array_typanalyze.c`, `ts_typanalyze.c`, `pg_type.dat`), the absence of any core page/FSM reader (`pg_proc.dat`, `genfile.c`), the GIN fresh-build size model (`ginblock.h`, `ginpostinglist.c`, `gindatapage.c`, `ginentrypage.c`, `gininsert.c`), the bitmap-index-scan cost surface (`selfuncs.c`, `createplan.c`), `TABLESAMPLE` semantics and restrictions (`select.sgml`, `system.c`, `bernoulli.c`, `parse_clause.c`), temporary-relation index behaviour (`create_table.sgml`, `indexcmds.c`), and `storage.sgml` fork layout.
- A second isolated 12.2 cluster under `.wiki-runtime/ginwaste-data` with five GIN fixtures, three churn rounds, `CREATE INDEX CONCURRENTLY` probes, `REINDEX` ground truth, and verbatim execution of every fenced SQL block in the follow-up.

## Evidence Map

| Claim | Evidence |
|---|---|
| `fastupdate` defaults on; inserts go to the pending list | [gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../raw/postgres-12/src/include/access/gin_private.h#L31-L39), [gininsert.c#gininsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L511-L531) |
| Trim threshold uses page capacity, not bytes used | [ginfast.c#ginHeapTupleFastInsert-cleanup-trigger](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L461), [ginfast.c#GIN_PAGE_FREESIZE](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L40-L41) |
| Wide rows get pending pages to themselves, wasting space | [README#GIN_LIST_FULLROW](../../../raw/postgres-12/src/backend/access/gin/README#L206-L216), [ginfast.c#makeSublist](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L143-L208) |
| Insert-driven cleanup abandons on contention | [ginfast.c#ginInsertCleanup-locking](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L800-L828) |
| Cleanup stops at the remembered tail unless full | [ginfast.c#ginInsertCleanup-cleanupFinish](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L881-L888) |
| Autovacuum passes `full_clean = false`; manual VACUUM passes true | [ginvacuum.c#ginbulkdelete-pending-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L583-L594), [ginvacuum.c#ginvacuumcleanup-pending-cleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L711-L721) |
| Autoanalyze cleans the pending list; manual ANALYZE does not | [ginvacuum.c#ginvacuumcleanup-analyze_only](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L697-L709) |
| `gin_clean_pending_list` requires ownership, refuses recovery | [ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1030-L1074) |
| Turning off `fastupdate` does not flush the list | [create_index.sgml#fastupdate](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L459-L467) |
| Entry tree is never deleted from | [README#no-entry-delete](../../../raw/postgres-12/src/backend/access/gin/README#L28-L31), [README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396) |
| Emptied entry tuples are retained with a null posting list | [ginvacuum.c#ginVacuumEntryPage](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556), [ginentrypage.c#GinFormTuple](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L44-L154) |
| An in-line posting list can grow, force entry splits, then become a smaller posting-tree pointer | [gininsert.c#addItemPointersToLeafTuple](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L42-L118), [README#Page deletion](../../../raw/postgres-12/src/backend/access/gin/README#L391-L396) |
| Entry splits try to equalize bytes, with no append/build special case | [ginentrypage.c#entrySplitPage](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L654-L689) |
| GIN has only two reloptions, no fillfactor | [ginutil.c#ginoptions](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L629) |
| Data-leaf splits balance segments, target roughly 75% on append, and pack the build split's left page | [gindatapage.c#dataBeginPlaceToPageLeaf-split](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L617-L667) |
| Internal data pages split counts in half outside a rightmost build split | [gindatapage.c#dataSplitPageInternal](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L1284-L1294) |
| VACUUM recompresses changed segments but does not coalesce/redivide small survivors | [gindatapage.c#ginVacuumPostingTreeLeaf](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c#L744-L810) |
| Only fully empty, non-edge posting pages are deleted | [ginvacuum.c#ginScanToDelete](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L302-L317) |
| Deleted pages carry the next XID and wait for the oldest running-transaction horizon | [ginvacuum.c#ginDeletePage-deleteXid](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L182-L192), [ginblock.h#GinPageIsRecyclable](../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [procarray.c#RecentGlobalXmin](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1488-L1503) |
| `ginvacuumcleanup` records XID-safe posting-tree pages; `shiftList` can record pending pages immediately | [ginvacuum.c#ginvacuumcleanup-page-loop](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L777), [ginfast.c#shiftList-delete-and-record](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L627-L665) |
| Ordinary VACUUM does not truncate the GIN fork | [ginvacuum.c#ginvacuumcleanup](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L738-L794), [vacuumlazy.c#lazy_truncate_heap-RelationTruncate](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1962-L1965) |
| REINDEX, table rewrite, and TRUNCATE can replace or reset index storage | [index.c#reindex_index-new-relfilenode](../../../raw/postgres-12/src/backend/catalog/index.c#L3519-L3530), [cluster.c#finish_heap_swap-reindex](../../../raw/postgres-12/src/backend/commands/cluster.c#L1378-L1410), [heap.c#RelationTruncateIndexes](../../../raw/postgres-12/src/backend/catalog/heap.c#L3163-L3205) |
| FSM candidates are validated before reuse or extension | [indexfsm.c#GetFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L32-L45), [ginutil.c#GinNewBuffer](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L286-L335) |
| One entry per distinct key per row; non-HOT update re-inserts all | [gininsert.c#ginHeapTupleInsert](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L462-L482), [heapam.c#heap_update-hot-check](../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3593-L3600) |
| `pgstatginindex` returns only three metapage fields | [pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L573) |
| `MATERIALIZED` prevents v12 CTE inlining in the mixed-AM survey | [subselect.c#CTEMaterializeAlways](../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c#L857-L892) |
| `pg_relation_size(regclass)` defaults to the main fork | [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336), [pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6891) |
| `pg_relpages` returns the live relation block count | [pgstatindex.c#pg_relpagesbyid_v1_5](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L457-L477) |
| `pgstatindex` / `pgstattuple` / `pgstattuple_approx` reject GIN | [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L228), [pgstattuple.c#pgstat_relation](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L274-L276), [pgstatapprox.c#pgstattuple_approx_internal](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c#L280-L290) |
| `pageinspect` GIN functions and their gates | [ginfuncs.c#gin_metapage_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L34-L87), [ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L90-L154), [ginfuncs.c#gin_leafpage_items](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L163-L265), [rawpage.c#get_raw_page_internal](../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106) |
| `pg_freespace` reports FSM records; index records are binary and saturate at 8160 bytes for an 8 kB build | [pg_freespacemap.c#pg_freespace](../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c#L17-L41), [indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L55), [pgfreespacemap.sgml#index-caveat](../../../raw/postgres-12/doc/src/sgml/pgfreespacemap.sgml#L66-L70) |
| Metapage stats are as of the last VACUUM | [ginutil.c#ginGetStats](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L631-L657) |
| Planner charges every pending page at `random_page_cost` | [selfuncs.c#gincostestimate-pending](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6869-L6873), [selfuncs.c#gincostestimate-startup](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6922-L6926) |
| Strict one-quarter/4x growth cutoff, then invented statistics | [selfuncs.c#gincostestimate-stats-trust](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6716-L6750), [selfuncs.c#gincostestimate-fallback](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6751-L6770) |
| Scans always walk the pending list first | [ginget.c#gingetbitmap-scanPendingInsert](../../../raw/postgres-12/src/backend/access/gin/ginget.c#L1871-L1878), [gin.sgml#gin-fast-update-disadvantage](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L491-L499) |
| Planner uses the live block count, and no tree height for GIN | [plancat.c#get_relation_info-index-size](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L387-L407), [plancat.c#get_relation_info-tree_height](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L409-L418) |
| The rebuild probe is transaction-block-forbidden and does extra concurrent-build work | [utility.c#CREATE-INDEX-CONCURRENTLY](../../../raw/postgres-12/src/backend/tcop/utility.c#L1299-L1311), [create_index.sgml#Building-Indexes-Concurrently](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L527-L558) |
| GIN-related GUC contexts and reloption locks | [guc.c#enable_seqscan](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L883-L890), [guc.c#work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2230-L2240), [guc.c#maintenance_work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2243-L2251), [guc.c#statement-and-lock-timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2398), [guc.c#autovacuum_work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3029-L3037), [guc.c#gin_pending_list_limit](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184), [guc.c#random_page_cost](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3217-L3226), [reloptions.c#fastupdate](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L125-L133), [reloptions.c#gin_pending_list_limit](../../../raw/postgres-12/src/backend/access/common/reloptions.c#L322-L330) |
| `idx_tup_fetch` is structurally 0 for GIN | [ginutil.c#ginhandler](../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L71-L72), [indexam.c#index_fetch_heap](../../../raw/postgres-12/src/backend/access/index/indexam.c#L565-L589), [monitoring.sgml#idx_tup_fetch](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2914-L2929) |
| `pg_class` index stats skipped when VACUUM skipped heap pages | [vacuumlazy.c#lazy_cleanup_index](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1784-L1815) |
| GIN reports heap tuple count as index entries | [ginvacuum.c#ginvacuumcleanup-num_index_tuples](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L725-L731) |
| amcheck has no GIN support | [verify_nbtree.c#btree_index_checkable](../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323) |
| Documentation concedes non-B-tree bloat is under-researched | [maintenance.sgml#routine-reindex](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L876-L880) |
| `GIN_AM_OID` comes from a generated header | [pg_am.dat:27-29](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L27-L29), [genbki.pl#oid_symbol-emit](../../../raw/postgres-12/src/backend/catalog/genbki.pl#L598-L603), [catalog/.gitignore](../../../raw/postgres-12/src/include/catalog/.gitignore#L1-L3) |
| `pg_relation_size` covers one fork and is a `stat()` of the segment files | [dbsize.c#calculate_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308), [dbsize.c#pg_relation_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336) |
| The one-argument form defaults to `'main'`; a two-argument form names the fork | [pg_proc.dat#pg_relation_size](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6891), [relpath.c#forkNames](../../../raw/postgres-12/src/common/relpath.c#L33-L38) |
| `pg_table_size` sums every fork and is safe on an index | [dbsize.c#calculate_table_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L380-L408) |
| `pg_indexes_size` on an index returns zero | [dbsize.c#calculate_indexes_size](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L410-L414) |
| `pg_relation_size` works on temporary relations | [dbsize.c#calculate_relation_size-temp](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L271) |
| Indexes have an FSM fork but no visibility map | [storage.sgml#relation-forks](../../../raw/postgres-12/doc/src/sgml/storage.sgml#L204-L216), [storage.sgml#Free-Space-Map](../../../raw/postgres-12/doc/src/sgml/storage.sgml#L600-L607) |
| `relpages`/`reltuples` are planner estimates refreshed by VACUUM, ANALYZE, and some DDL | [pg_class.h#relpages](../../../raw/postgres-12/src/include/catalog/pg_class.h#L57-L63), [catalogs.sgml#pg_class.relpages](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1759-L1770), [catalogs.sgml#pg_class.reltuples](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L1772-L1782) |
| `vac_update_relstats` is the writer of both fields | [vacuum.c#vac_update_relstats-signature](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1156-L1162), [vacuum.c#vac_update_relstats](../../../raw/postgres-12/src/backend/commands/vacuum.c#L1180-L1191) |
| After `CREATE INDEX`/`REINDEX`, a GIN index's `reltuples` is the extracted-key count | [gininsert.c#ginBuildCallback-indtuples](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L262-L270), [gininsert.c#ginbuild-index_tuples](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L419-L427), [index.c#index_build-index_update_stats](../../../raw/postgres-12/src/backend/catalog/index.c#L2977-L2986) |
| ANALYZE without VACUUM sets index `reltuples` from a sampled fraction, 1.0 when non-partial | [analyze.c#do_analyze_rel-index-stats](../../../raw/postgres-12/src/backend/commands/analyze.c#L607-L629), [analyze.c:438](../../../raw/postgres-12/src/backend/commands/analyze.c#L438) |
| No `pg_statistic` slot holds a column-wide distinct-element count | [pg_statistic.h#STATISTIC_NUM_SLOTS](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L120-L126) |
| The last element of the distinct-element-count histogram is the per-row average | [array_typanalyze.c#compute_array_stats-avg-count](../../../raw/postgres-12/src/backend/utils/adt/array_typanalyze.c#L605-L614), [pg_statistic.h#STATISTIC_KIND_DECHIST](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L236-L249), [catalogs.sgml#elem_count_histogram](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10921-L10930) |
| `pg_stats` exposes that histogram from statistic kind 5 | [system_views.sql#pg_stats-elem_count_histogram](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L240-L246) |
| `tsvector` gets a most-common-elements slot only; `jsonb` gets no `typanalyze` | [ts_typanalyze.c#compute_tsvector_stats-mcelem](../../../raw/postgres-12/src/backend/tsearch/ts_typanalyze.c#L415-L426), [pg_type.dat:438-442](../../../raw/postgres-12/src/include/catalog/pg_type.dat#L438-L442), [analyze.c#typanalyze-dispatch](../../../raw/postgres-12/src/backend/commands/analyze.c#L955-L963) |
| `gin_clean_pending_list` is a core SQL function returning pages deleted | [pg_proc.dat#gin_clean_pending_list](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L8639-L8642), [ginfast.c#gin_clean_pending_list](../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L1030-L1074) |
| `pg_read_binary_file` is the only core path to raw relation bytes | [genfile.c#pg_read_binary_file](../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L288-L300) |
| A `Bitmap Index Scan` node shows zero startup and `indextotalcost` as its total | [createplan.c#create_bitmap_subplan-cost](../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c#L3214-L3216) |
| `gincostestimate` folds the pending-page charge into the index total cost | [selfuncs.c#gincostestimate-startup](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6922-L6926), [selfuncs.c#gincostestimate-total](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6964-L6977) |
| Entry-tree splits equalize bytes with no build or append special case | [ginentrypage.c#entrySplitPage-balance](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c#L666-L689) |
| An entry tuple's in-line posting list is capped by `GinMaxItemSize` before promotion | [ginblock.h#GinMaxItemSize](../../../raw/postgres-12/src/include/access/ginblock.h#L245-L256), [gininsert.c#addItemPointersToLeafTuple](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L42-L119) |
| A posting-list segment costs an 8-byte header plus 1 to 7 varbyte bytes per later TID | [ginblock.h#GinPostingList](../../../raw/postgres-12/src/include/access/ginblock.h#L334-L346), [ginpostinglist.c#varbyte-format](../../../raw/postgres-12/src/backend/access/gin/ginpostinglist.c#L23-L72), [ginpostinglist.c#MaxBytesPerInteger](../../../raw/postgres-12/src/backend/access/gin/ginpostinglist.c#L83-L84) |
| A GIN build allocates a metapage and root before scanning | [gininsert.c#ginbuild-meta-root](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L337-L357) |
| The build accumulator flushes at `maintenance_work_mem`, and the docs call build time sensitive to it | [gininsert.c#ginBuildCallback-flush](../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L290-L311), [gin.sgml#gin-tips-maintenance_work_mem](../../../raw/postgres-12/doc/src/sgml/gin.sgml#L554-L563) |
| `TABLESAMPLE` accepts only tables, matviews, and partitioned tables | [parse_clause.c#transformRangeTableSample-relkind](../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1160-L1169) |
| `SYSTEM` samples whole blocks, `BERNOULLI` samples rows | [select.sgml#TABLESAMPLE-methods](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L404-L423), [system.c#system_nextsampletuple](../../../raw/postgres-12/src/backend/access/tablesample/system.c#L224-L235), [bernoulli.c#bernoulli_nextsampletuple](../../../raw/postgres-12/src/backend/access/tablesample/bernoulli.c#L169-L179) |
| An index on a temporary table is automatically temporary, and CONCURRENTLY is downgraded | [create_table.sgml#TEMPORARY-indexes](../../../raw/postgres-12/doc/src/sgml/ref/create_table.sgml#L173-L178), [indexcmds.c#DefineIndex-temp-nonconcurrent](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L489-L499) |
| `temp_buffers` is session-scoped | [guc.c#temp_buffers](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2165-L2174) |

## Open Questions

- **`indexfsm.c`'s header comment contradicts its own code.** The comment says the FSM uses "`BLCKSZ - 1` to denote used pages, and 0 for unused" ([indexfsm.c#NOTES](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L14-L19)), but `RecordFreeIndexPage` writes `BLCKSZ - 1` and `RecordUsedIndexPage` writes 0 ([indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L48-L65)). Source wins, and the exact-pin run agrees: free GIN pages read back as `avail = 8160`. The comment appears to be inverted.
- **No steady-state occupancy figure is derivable from source.** The split code balances discrete segments, has an approximately 75%-left append heuristic, and packs a build split's left page, but the pinned tree states no expected long-run fill for entry or posting pages under a mixed workload. The reviewed retail fixture's 8.2% reclaimable share came from retained entry pages, while its posting-page count did not change; it does not quantify the posting-split policy.
- **No threshold for "bloated" is defined anywhere in the pinned tree.** Unlike B-tree, where `fillfactor` gives a reference density, GIN has no target to compare against. The recipes report observable facts; choosing an action threshold remains a judgement call, and the controlled rebuild comparison is the closest direct byte estimate.
- **Autovacuum's `full_clean = false` behavior was not measured under concurrent append.** The caller asymmetry is established from source, but the exact-pin server ran with `autovacuum = off`. Without concurrent appends, a partial clean can still reach the end of the entire pre-existing list, so no fixed residue can be inferred.
- **`gin_leafpage_items` was not used to quantify within-leaf slack.** It would give exact per-segment byte counts for posting-tree leaves, which is the missing piece for a true density metric. Its restriction to compressed data leaves ([ginfuncs.c#gin_leafpage_items](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c#L196-L202)) means it cannot cover entry-tree leaves, so it could only ever measure part of the index.
- **Cross-version behavior is out of scope.** Whether any of these mechanisms changed after v12 was not investigated, and per the evidence rules cannot be answered from this checkout.
- **Method B's accuracy bound is fixture-specific, not a general guarantee.** The worst error observed was 2.75 percentage points over eight comparisons on four fixtures, and the largest error came from the fixture whose live row count grew 60% past its baseline. Nothing in the pinned tree bounds the error, and no shipped test exercises it. Treat the number as evidence that the method is usable, not as a tolerance.
- **The bytes-per-row baseline was only tested with a stable key distribution.** All fixtures kept the same keys-per-row and roughly the same key cardinality across churn. A workload that changes the number of extracted keys per row, or moves keys across the `GinMaxItemSize` promotion boundary ([ginblock.h#GinMaxItemSize](../../../raw/postgres-12/src/include/access/ginblock.h#L245-L256)), would invalidate a baseline by an amount this review did not quantify.
- **No core-only method was found for a first measurement with no prior rebuild.** Method A needs a build; method B needs a baseline recorded after one. An operator inheriting an un-rebuilt GIN index has no read-only way to get an absolute wasted-byte figure from core SQL, because of the three gaps in [Why core SQL cannot see inside a GIN index](#why-core-sql-cannot-see-inside-a-gin-index).
- **The pending-page proxy has no derived error bound.** The two loaded observations landed 0.18% and 0.51% above the true pending-page count, but the `Bitmap Index Scan` total cost also carries data-page fetches and per-tuple CPU terms ([selfuncs.c#gincostestimate-total](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c#L6964-L6977)), so its accuracy depends on the probe query's selectivity. Only an upper-bound reading is defensible.
- **Whether a smarter sampled estimator could work was not established.** This review only tested linear extrapolation of bytes per row, which failed. A model that separates saturating entry-tree bytes from linear posting bytes is conceivable, but it needs the distinct-key count that no core statistic provides ([pg_statistic.h#STATISTIC_NUM_SLOTS](../../../raw/postgres-12/src/include/catalog/pg_statistic.h#L120-L126)), so it was not pursued.
- **`pg_read_binary_file` was not tested as a page reader.** It is the only core path to raw relation bytes ([genfile.c#pg_read_binary_file](../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L288-L300)), and decoding GIN page headers from `bytea` in SQL is arguably "core SQL only", but it bypasses shared buffers and needs elevated privileges. No attempt was made to build or validate such a decoder.

## Source References

- GIN implementation: [ginfast.c](../../../raw/postgres-12/src/backend/access/gin/ginfast.c), [ginvacuum.c](../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c), [gindatapage.c](../../../raw/postgres-12/src/backend/access/gin/gindatapage.c), [ginentrypage.c](../../../raw/postgres-12/src/backend/access/gin/ginentrypage.c), [ginutil.c](../../../raw/postgres-12/src/backend/access/gin/ginutil.c), [gininsert.c](../../../raw/postgres-12/src/backend/access/gin/gininsert.c), [ginget.c](../../../raw/postgres-12/src/backend/access/gin/ginget.c), [ginscan.c](../../../raw/postgres-12/src/backend/access/gin/ginscan.c), [ginpostinglist.c](../../../raw/postgres-12/src/backend/access/gin/ginpostinglist.c), [README](../../../raw/postgres-12/src/backend/access/gin/README)
- GIN headers: [ginblock.h](../../../raw/postgres-12/src/include/access/ginblock.h), [gin_private.h](../../../raw/postgres-12/src/include/access/gin_private.h), [gin.h](../../../raw/postgres-12/src/include/access/gin.h)
- Executor, utility, and index API: [execIndexing.c](../../../raw/postgres-12/src/backend/executor/execIndexing.c), [utility.c](../../../raw/postgres-12/src/backend/tcop/utility.c), [indexam.c](../../../raw/postgres-12/src/backend/access/index/indexam.c)
- VACUUM, horizons, and FSM: [vacuumlazy.c](../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c), [vacuum.c](../../../raw/postgres-12/src/backend/commands/vacuum.c), [procarray.c](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c), [indexfsm.c](../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c), [freespace.c](../../../raw/postgres-12/src/backend/storage/freespace/freespace.c)
- Rebuild and truncation: [index.c](../../../raw/postgres-12/src/backend/catalog/index.c), [indexcmds.c](../../../raw/postgres-12/src/backend/commands/indexcmds.c), [cluster.c](../../../raw/postgres-12/src/backend/commands/cluster.c), [tablecmds.c](../../../raw/postgres-12/src/backend/commands/tablecmds.c), [heap.c](../../../raw/postgres-12/src/backend/catalog/heap.c)
- Planner: [selfuncs.c](../../../raw/postgres-12/src/backend/utils/adt/selfuncs.c), [plancat.c](../../../raw/postgres-12/src/backend/optimizer/util/plancat.c), [costsize.c](../../../raw/postgres-12/src/backend/optimizer/path/costsize.c), [subselect.c](../../../raw/postgres-12/src/backend/optimizer/plan/subselect.c), [createplan.c](../../../raw/postgres-12/src/backend/optimizer/plan/createplan.c)
- Core size functions and relation paths: [dbsize.c](../../../raw/postgres-12/src/backend/utils/adt/dbsize.c), [relpath.c](../../../raw/postgres-12/src/common/relpath.c), [genfile.c](../../../raw/postgres-12/src/backend/utils/adt/genfile.c)
- Catalog size and column statistics: [pg_class.h](../../../raw/postgres-12/src/include/catalog/pg_class.h), [pg_statistic.h](../../../raw/postgres-12/src/include/catalog/pg_statistic.h), [system_views.sql](../../../raw/postgres-12/src/backend/catalog/system_views.sql), [analyze.c](../../../raw/postgres-12/src/backend/commands/analyze.c), [array_typanalyze.c](../../../raw/postgres-12/src/backend/utils/adt/array_typanalyze.c), [ts_typanalyze.c](../../../raw/postgres-12/src/backend/tsearch/ts_typanalyze.c), [pg_type.dat](../../../raw/postgres-12/src/include/catalog/pg_type.dat)
- Table sampling and temporary relations: [system.c](../../../raw/postgres-12/src/backend/access/tablesample/system.c), [bernoulli.c](../../../raw/postgres-12/src/backend/access/tablesample/bernoulli.c), [parse_clause.c](../../../raw/postgres-12/src/backend/parser/parse_clause.c)
- Settings: [guc.c](../../../raw/postgres-12/src/backend/utils/misc/guc.c), [reloptions.c](../../../raw/postgres-12/src/backend/access/common/reloptions.c)
- Catalog and generated-header inputs: [pg_am.dat](../../../raw/postgres-12/src/include/catalog/pg_am.dat), [pg_proc.dat](../../../raw/postgres-12/src/include/catalog/pg_proc.dat), [genbki.pl](../../../raw/postgres-12/src/backend/catalog/genbki.pl), [catalog/Makefile](../../../raw/postgres-12/src/backend/catalog/Makefile)
- Measurement tooling: [pgstatindex.c](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c), [pgstattuple.c](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c), [pgstatapprox.c](../../../raw/postgres-12/contrib/pgstattuple/pgstatapprox.c), [ginfuncs.c](../../../raw/postgres-12/contrib/pageinspect/ginfuncs.c), [rawpage.c](../../../raw/postgres-12/contrib/pageinspect/rawpage.c), [pg_freespacemap.c](../../../raw/postgres-12/contrib/pg_freespacemap/pg_freespacemap.c), [verify_nbtree.c](../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c)
- Documentation: [gin.sgml](../../../raw/postgres-12/doc/src/sgml/gin.sgml), [maintenance.sgml](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml), [create_index.sgml](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml), [create_table.sgml](../../../raw/postgres-12/doc/src/sgml/ref/create_table.sgml), [select.sgml](../../../raw/postgres-12/doc/src/sgml/ref/select.sgml), [set.sgml](../../../raw/postgres-12/doc/src/sgml/ref/set.sgml), [catalogs.sgml](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml), [storage.sgml](../../../raw/postgres-12/doc/src/sgml/storage.sgml), [pgstattuple.sgml](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml), [pageinspect.sgml](../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml), [pgfreespacemap.sgml](../../../raw/postgres-12/doc/src/sgml/pgfreespacemap.sgml), [monitoring.sgml](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml)
- Tests: [regress/sql/gin.sql](../../../raw/postgres-12/src/test/regress/sql/gin.sql), [pageinspect/sql/gin.sql](../../../raw/postgres-12/contrib/pageinspect/sql/gin.sql), [pgstattuple/expected/pgstattuple.out](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out), [predicate-gin.spec](../../../raw/postgres-12/src/test/isolation/specs/predicate-gin.spec)

## Navigation

- [v12 index](../index.md)
- [PostgreSQL 12 Codebase Navigation Guide (unverified)](../codebase-navigation-guide.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](bloated-indexes-query-planner.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [How NULL Values Are Handled in PostgreSQL 12 Indexes (unverified)](null-values-in-indexes.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
