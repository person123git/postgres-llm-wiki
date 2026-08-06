---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: not yet
---

# How a GIN Index Becomes Bloated in PostgreSQL 17, and How to Measure It (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [The four page classes inside a GIN index](#the-four-page-classes-inside-a-gin-index)
  - [Mechanism 1: the pending list (fastupdate)](#mechanism-1-the-pending-list-fastupdate)
  - [Mechanism 2: entry-tree tuples and pages are never removed](#mechanism-2-entry-tree-tuples-and-pages-are-never-removed)
  - [Mechanism 3: entry leaves settle near half full, and a rebuild does not fix that](#mechanism-3-entry-leaves-settle-near-half-full-and-a-rebuild-does-not-fix-that)
  - [Mechanism 4: posting-tree splits leave headroom](#mechanism-4-posting-tree-splits-leave-headroom)
  - [Mechanism 5: VACUUM leaves slack inside posting-tree leaves](#mechanism-5-vacuum-leaves-slack-inside-posting-tree-leaves)
  - [Mechanism 6: deleted posting-tree pages wait for the visibility horizon](#mechanism-6-deleted-posting-tree-pages-wait-for-the-visibility-horizon)
  - [Mechanism 7: ordinary VACUUM never shortens the GIN file](#mechanism-7-ordinary-vacuum-never-shortens-the-gin-file)
  - [When VACUUM skips the index entirely](#when-vacuum-skips-the-index-entirely)
  - [What VACUUM can and cannot recover](#what-vacuum-can-and-cannot-recover)
  - [Measuring: which shipped tools accept a GIN index](#measuring-which-shipped-tools-accept-a-gin-index)
  - [Recipe 1: pending-list backlog for every GIN index](#recipe-1-pending-list-backlog-for-every-gin-index)
  - [Recipe 2: per-page census](#recipe-2-per-page-census)
  - [Recipe 3: leaf fill for entry and posting leaves](#recipe-3-leaf-fill-for-entry-and-posting-leaves)
  - [Recipe 4: metapage statistics versus live size](#recipe-4-metapage-statistics-versus-live-size)
  - [Recipe 5: pages the FSM already offers for reuse](#recipe-5-pages-the-fsm-already-offers-for-reuse)
  - [Measuring with core SQL only](#measuring-with-core-sql-only)
  - [Recipe 6: size, forks, and catalog staleness](#recipe-6-size-forks-and-catalog-staleness)
  - [The reltuples trap](#the-reltuples-trap)
  - [Recipe 7: VACUUM VERBOSE and the autovacuum log](#recipe-7-vacuum-verbose-and-the-autovacuum-log)
  - [Recipe 8: reading the pending-page count out of EXPLAIN](#recipe-8-reading-the-pending-page-count-out-of-explain)
  - [Recipe 9: gin_clean_pending_list as an exact pending-page count](#recipe-9-gin_clean_pending_list-as-an-exact-pending-page-count)
  - [Recipe 10: the rebuild probe, the only ground truth](#recipe-10-the-rebuild-probe-the-only-ground-truth)
  - [How bloat reaches the planner](#how-bloat-reaches-the-planner)
  - [Settings, and their apply scope](#settings-and-their-apply-scope)
  - [Exact-pin measurements](#exact-pin-measurements)
  - [Key data structures](#key-data-structures)
  - [Caller and callee boundary](#caller-and-callee-boundary)
  - [Build, generated-header, and extension boundary](#build-generated-header-and-extension-boundary)
  - [Tests and explicit test absence](#tests-and-explicit-test-absence)
  - [What changed since PostgreSQL 12](#what-changed-since-postgresql-12)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17: how can a GIN index become bloated? How do you measure whether a GIN index is bloated?

## Answer

### Short answer

A PostgreSQL 17 GIN index grows beyond the size of its live contents for seven distinct reasons. They are not all the same kind of bloat: a pending list is a live write buffer, split headroom is normal structure, and only retained entry tuples plus sparse posting pages are genuinely historical space.

| # | Source of excess size | What ordinary VACUUM does about it | Evidence |
|---|---|---|---|
| 1 | Pending-list (`fastupdate`) pages accumulate | Merges the list and hands the freed pages to the free space map | [ginfast.c#ginInsertCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L779-L1025), [ginfast.c#shiftList](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L553-L671) |
| 2 | Entry-tree tuples and pages survive after their posting lists empty | Removes dead TIDs, keeps the tuple and the page | [ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L455-L562), [README#Page deletion](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396) |
| 3 | Entry-page splits equalize by data size, with no rightmost or build special case | Nothing; VACUUM never merges entry pages | [ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L601-L696), [ginentrypage.c#entryIsEnoughSpace](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482) |
| 4 | Retail posting-tree splits leave balancing or append headroom | Does not rebalance populated pages | [gindatapage.c#dataBeginPlaceToPageLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L617-L667) |
| 5 | Partly emptied posting-tree leaves keep their page and segment slack | Recompresses changed segments inside the old byte budget, never re-encodes or merges | [gindatapage.c#ginVacuumPostingTreeLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L737-L864) |
| 6 | Deleted posting-tree pages wait before they can be reused | Records them in the free space map only once the delete XID is globally invisible | [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L798-L822), [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L747-L787) |
| 7 | Reusable pages stay inside the main fork at its high-water size | Reuses them; ordinary VACUUM does not shorten a GIN fork | [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335) |

`REINDEX` is the index-only repair: it replaces the physical file with a fresh build ([index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3752-L3760)). `REINDEX INDEX CONCURRENTLY` builds a replacement before swapping ([indexcmds.c#ReindexRelationConcurrently](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L3436-L3450)). Table-level commands rebuild indexes as a side effect: `VACUUM FULL` and `CLUSTER` through [cluster.c#rebuild_relation](../../../../raw/postgres-17/src/backend/commands/cluster.c#L632-L645), and `TRUNCATE` through [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3034-L3045). Those are not substitutes for an index-only repair; they take stronger locks and touch the table.

Measurement is harder than for B-tree, and PostgreSQL 17's own documentation says so: "The potential for bloat in non-B-tree indexes has not been well researched. It is a good idea to periodically monitor the index's physical size when using any non-B-tree index type." ([maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1036-L1040)). PostgreSQL 17 ships no GIN density function. `pgstatindex`, `pgstattuple`, `pgstattuple_approx`, and `amcheck` all refuse a GIN index. What works is `pgstattuple`'s metapage-only `pgstatginindex`, a `pageinspect` page walk, `pg_freespacemap`, core size functions, `VACUUM VERBOSE`, an `EXPLAIN`-cost probe, and comparison against a freshly built copy.

One correction to the common assumption: **a fresh build is not always smaller.** On a distinct-key fixture at this pin, `REINDEX` made the index 14.0% *larger*, because a build inserts entry tuples in ascending key order and entry splits have no rightmost-append optimization. See [Mechanism 3](#mechanism-3-entry-leaves-settle-near-half-full-and-a-rebuild-does-not-fix-that).

### The four page classes inside a GIN index

A GIN index is a metapage, a B-tree of key values (the *entry tree*), optional *posting trees* holding the heap TIDs of very common keys, and optional *pending-list* pages holding not-yet-merged entries ([README#Index structure](../../../../raw/postgres-17/src/backend/access/gin/README#L95-L105)). The metapage is block 0 and the entry-tree root is block 1 ([ginblock.h#GIN_METAPAGE_BLKNO](../../../../raw/postgres-17/src/include/access/ginblock.h#L51-L53)).

Every page carries a `flags` word naming its class ([ginblock.h#GIN_DATA](../../../../raw/postgres-17/src/include/access/ginblock.h#L41-L48)), which is what makes the census in [Recipe 2](#recipe-2-per-page-census) possible. Two flag combinations matter for bloat accounting:

- A trimmed pending page has its whole flags word overwritten with `GIN_DELETED`, so it reports `deleted` and nothing else ([ginfast.c#shiftList-flags](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L630-L635)).
- A deleted posting-tree page keeps its `data`/`leaf` bits and adds `deleted`, because `GinPageSetDeleted` ORs the bit in ([ginblock.h#GinPageSetDeleted](../../../../raw/postgres-17/src/include/access/ginblock.h#L125-L127), [ginvacuum.c#ginDeletePage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L196)).

### Mechanism 1: the pending list (fastupdate)

`fastupdate` defaults to on ([gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../../raw/postgres-17/src/include/access/gin_private.h#L33-L45)), so by default `gininsert` appends each new row's keys to an unsorted linear list of `GIN_LIST` pages instead of inserting them into the entry tree ([gininsert.c#gininsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L482-L536)).

The trim trigger compares *page capacity*, not bytes actually used, against `gin_pending_list_limit`:

```c
	cleanupSize = GinGetPendingListCleanupSize(index);
	if (metadata->nPendingPages * GIN_PAGE_FREESIZE > cleanupSize * 1024L)
		needCleanup = true;
```

([ginfast.c#ginHeapTupleFastInsert-trigger](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471), with `GIN_PAGE_FREESIZE` at [ginfast.c#GIN_PAGE_FREESIZE](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L41-L42)). A half-empty pending page therefore counts at full weight.

Three separate things keep the list longer than the limit suggests:

1. **Internal fragmentation.** A row whose entries do not fit on one pending page gets pages to itself. The README states it directly: "a heap tuple whose entries do not all fit on one pending-list page must have those pages to itself, even if this results in wasting much of the space on the preceding page and the last page for the tuple" ([README#GIN_LIST_FULLROW](../../../../raw/postgres-17/src/backend/access/gin/README#L201-L216)). The abandoned-tail path is the `separateList` branch in [ginfast.c#ginHeapTupleFastInsert-separateList](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L253-L287), and `makeSublist` allocates fresh pages per heap tuple ([ginfast.c#makeSublist](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L144-L209)).
2. **A backend gives up on contention.** An insert-driven cleanup takes `ConditionalLockPage` and returns immediately if another cleanup holds the lock; it also budgets only `work_mem`, not `maintenance_work_mem` ([ginfast.c#ginInsertCleanup-locking](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L800-L828)).
3. **Cleanup stops at the tail it remembered on entry** unless the caller asked for a full clean ([ginfast.c#ginInsertCleanup-blknoFinish](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L843-L888), [ginfast.c#ginInsertCleanup-cleanupFinish](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L982-L995)).

Autovacuum is asymmetric here. Both VACUUM entry points pass `full_clean = !AmAutoVacuumWorkerProcess()`, so an autovacuum worker stops at the tail it remembered while manual `VACUUM` continues until the list is empty ([ginvacuum.c#ginbulkdelete-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L584-L595), [ginvacuum.c#ginvacuumcleanup-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L712-L722)). With no concurrent append, the remembered tail can still be the end of the whole pre-existing list, so a partial clean does not imply a nonempty residue. An autovacuum *analyze* runs a partial clean; a plain manual `ANALYZE` is a no-op for the pending list ([ginvacuum.c#ginvacuumcleanup-analyze_only](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L698-L710), reached from [analyze.c#do_analyze_rel-cleanup](../../../../raw/postgres-17/src/backend/commands/analyze.c#L704-L720)).

`gin_clean_pending_list(regclass)` requests a full clean without vacuuming the heap. It takes `RowExclusiveLock`, refuses to run during recovery, rejects non-GIN and other sessions' temporary indexes, requires index ownership, treats a `!indisvalid` index as a `DEBUG1` no-op, and returns the number of pending pages removed ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091)).

Trimmed pending pages get the `GIN_DELETED` flag and, when `fill_fsm` is set, go straight into the free space map ([ginfast.c#shiftList-RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L662-L668)), followed by an FSM vacuum ([ginfast.c#ginInsertCleanup-fsm](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1014-L1020)). They become reusable pages inside the same file; the file does not shrink.

One operational trap: turning `fastupdate` off does not flush what is already pending. The documentation says so ([create_index.sgml#fastupdate](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L510-L535)), and the exact-pin run below confirms it.

### Mechanism 2: entry-tree tuples and pages are never removed

This is the mechanism with no VACUUM remedy, and it is by design. The README says it three times:

- "There is no delete operation in the key (entry) tree. The reason for this is that in our experience, the set of distinct words in a large corpus changes very slowly." ([README#no-entry-delete](../../../../raw/postgres-17/src/backend/access/gin/README#L28-L31))
- "That works because tuples are never deleted from the entry tree." ([README#entry-high-key](../../../../raw/postgres-17/src/backend/access/gin/README#L309-L316))
- "Vacuum never deletes tuples or pages from the entry tree." ([README#Page deletion](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396))

The code matches. `ginVacuumEntryPage` strips dead TIDs from an entry tuple's in-line posting list and rewrites the tuple in place. When the list becomes empty it builds the tuple with a null posting list rather than removing it:

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

([ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L506-L557)); `GinFormTuple` accepts a zero-length posting list and returns a valid tuple ([ginentrypage.c#GinFormTuple](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L43-L155)). `GinPageSetDeleted` is applied only to `GIN_DATA` pages, guarded by an assertion in the deletion scan ([ginvacuum.c#ginScanToDelete-assert](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L279-L282)).

Consequence: under ordinary DML plus VACUUM, a key that once existed keeps its entry-tuple slot, and its entry page is never reclaimed. A workload that cycles through distinct keys makes the entry-page count non-decreasing until a physical rebuild.

A stable key set can retain extra entry pages too. An in-line posting list grows inside its entry tuple; once the tuple can no longer hold it, `addItemPointersToLeafTuple` creates a posting tree and replaces the expanded tuple with a small tree pointer ([gininsert.c#addItemPointersToLeafTuple](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L45-L115)). If those expanded tuples already forced entry-page splits, the later shrink does not merge the pages back.

### Mechanism 3: entry leaves settle near half full, and a rebuild does not fix that

`entrySplitPage` copies all tuples into a workspace and moves the split point when the accumulated left size passes half of the total:

```c
		if (lsize > totalsize / 2)
		{
			if (separator == InvalidOffsetNumber)
				separator = i - 1;
			page = rpage;
		}
```

([ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691)). There is no rightmost-page case and no build-mode case, unlike the posting-tree code in [Mechanism 4](#mechanism-4-posting-tree-splits-leave-headroom). `entryIsEnoughSpace` reserves no fill factor ([ginentrypage.c#entryIsEnoughSpace](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482)), and GIN exposes no `fillfactor` reloption: its only two reloptions are `fastupdate` and `gin_pending_list_limit` ([ginutil.c#ginoptions](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614)).

Now combine that with how a build feeds the entry tree. `ginbuild` accumulates keys in a red-black tree and drains it with an in-order walk, so `ginEntryInsert` sees keys in ascending order ([gininsert.c#ginbuild](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L316-L428), [ginbulk.c#ginBeginBAScan](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L255-L293)). Ascending insertion plus a strict half-and-half split leaves every page except the rightmost at roughly 50% fill, because the left half is never revisited.

Measured at this pin over 400,000 rows with 100,000 distinct keys: a fresh build produced 1063 entry leaves at **50.73%** average fill, while retail insertion of the same rows in scattered key order produced 933 entry leaves at **57.79%**. `REINDEX` therefore grew that index from 7,692,288 to 8,765,440 bytes. Retail insertion in *ascending* key order matched the build exactly: 877 leaves at 50.31% both before and after `REINDEX`, byte-for-byte 7,233,536. Changing `maintenance_work_mem` from 64 MB to 1 MB did not change the built result.

Two consequences for measurement:

- Roughly 50% entry-leaf fill is **structure, not bloat**. Do not treat it as reclaimable.
- "Bytes a rebuild would reclaim" can be negative for an entry-tree-dominated index. The rebuild probe in [Recipe 10](#recipe-10-the-rebuild-probe-the-only-ground-truth) reports the truth, including when the truth is unfavourable.

### Mechanism 4: posting-tree splits leave headroom

When a key accumulates more TIDs than fit in an entry tuple, the TIDs move into a posting tree ([gininsert.c#buildFreshLeafTuple](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L125-L166)). The leaf split policy depends on build mode and, for ordinary DML, on whether the insertion is an append:

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

([gindatapage.c#dataBeginPlaceToPageLeaf-split](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L617-L667)). These are heuristics over discrete compressed segments, not exact occupancy promises. Internal posting-tree pages pack a rightmost build split as far as possible and otherwise halve the posting-item count ([gindatapage.c#dataSplitPageInternal](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L1290-L1300)).

`btree->isBuild` is set from a non-null build-statistics pointer on both insertion paths ([gininsert.c#ginEntryInsert-isBuild](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L187-L190), [gindatapage.c#ginInsertItemPointers](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L1908-L1934)). `CREATE INDEX` and `REINDEX` therefore produce denser posting trees than retail insertion of the same rows, which is why the documentation advises building the index after a bulk load ([gin.sgml#gin-tips](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L561-L590)).

### Mechanism 5: VACUUM leaves slack inside posting-tree leaves

`ginVacuumPostingTreeLeaf` removes dead TIDs and recompresses each changed segment within that segment's old byte budget ([gindatapage.c#ginVacuumPostingTreeLeaf-segments](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L747-L795)). It then reconstructs the page, but deliberately does not coalesce or redivide small surviving segments:

```c
	 * We don't try to re-encode the segments here, even though some of them
	 * might be really small now that we've removed some items from them. It
	 * seems like a waste of effort, ...
	 * ... You'll have to REINDEX anyway if you want the full gain of the
	 * new tighter index format.
```

([gindatapage.c#ginVacuumPostingTreeLeaf-no-reencode](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L797-L813))

Page reclamation deletes a posting-tree page only when it is *completely* empty, and never the left- or rightmost branch:

```c
	if (isempty)
	{
		/* we never delete the left- or rightmost branch */
		if (BufferIsValid(me->leftBuffer) && !GinPageRightMost(page))
```

([ginvacuum.c#ginScanToDelete](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L303-L318)). The second pass only runs at all when the first pass saw at least one empty page ([ginvacuum.c#ginVacuumPostingTree](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L408-L448)). Sibling pages are never merged anywhere in the access method.

Measured at this pin: after deleting 760,000 of 800,000 rows from a 20-distinct-key fixture and vacuuming, the surviving 44 posting leaves averaged **7.50%** fill and the index stayed at 1,163,264 bytes; a rebuild produced 73,728 bytes.

### Mechanism 6: deleted posting-tree pages wait for the visibility horizon

`ginDeletePage` marks the page deleted and stamps it with the next transaction ID ([ginvacuum.c#ginDeletePage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L196)), stored in the page header's `pd_prune_xid` ([ginblock.h#GinPageSetDeleteXid](../../../../raw/postgres-17/src/include/access/ginblock.h#L132-L138)). The page becomes reusable only when no backend could still view that XID as running:

```c
	/*
	 * If no backend still could view delete_xid as in running, all scans
	 * concurrent with ginDeletePage() must have finished.
	 */
	return GlobalVisCheckRemovableXid(NULL, delete_xid);
```

([ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L798-L822))

Only `ginvacuumcleanup`'s full page scan converts a recyclable page into a free-space-map entry ([ginvacuum.c#ginvacuumcleanup-page-loop](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L747-L787)), and `GinNewBuffer` re-checks recyclability before reusing a page the FSM handed back ([ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335)). So a freshly deleted page normally needs a *later* VACUUM before it is offered for reuse. Measured at this pin: the VACUUM that deleted 76 posting pages recorded 0 free pages; after three `txid_current()` calls advanced the counter, the next VACUUM recorded all 76.

### Mechanism 7: ordinary VACUUM never shortens the GIN file

`ginvacuumcleanup` reports `num_pages` from the live block count and never truncates ([ginvacuum.c#ginvacuumcleanup-num_pages](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L787-L795)). Reclaimed pages are recycled through the FSM by `GinNewBuffer`, inside the same file, at its high-water size ([ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335)).

Measured at this pin: a 131,072-byte GIN index stayed at 131,072 bytes after deleting every row and running VACUUM twice; `VACUUM FULL` then rebuilt it to 16,384 bytes.

### When VACUUM skips the index entirely

Two PostgreSQL 17 behaviours can leave GIN untouched even when VACUUM runs on the table:

- **The index-vacuum bypass.** If fewer than 2% of the table's pages hold dead items and the dead-item store is under 32 MB, VACUUM sets `do_index_vacuuming = false`, so `ginbulkdelete` never runs ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1946)). Index *cleanup* still runs, so the pending list is still flushed and the metapage statistics are still refreshed. `VACUUM (INDEX_CLEANUP ON)` clears `consider_bypass_optimization`, which disables this bypass but not the failsafe below; `INDEX_CLEANUP OFF` disables index vacuuming and cleanup outright ([vacuumlazy.c#index_cleanup](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L388-L407)).
- **The wraparound failsafe.** Once the table's age crosses `vacuum_failsafe_age`, VACUUM clears *both* `do_index_vacuuming` and `do_index_cleanup`, so GIN gets neither `ginbulkdelete` nor a pending-list flush, and the metapage statistics go stale ([vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2299-L2347)).

Both are visible in `VACUUM VERBOSE` and in the autovacuum log as `index scan bypassed:` or `index scan bypassed by failsafe:` ([vacuumlazy.c#verbose-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L717)).

### What VACUUM can and cannot recover

| Space | Recovered by ordinary VACUUM? | Evidence |
|---|---|---|
| Pending-list pages | Yes, merged into the entry tree and recorded free | [ginfast.c#ginInsertCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L779-L1025) |
| Dead TIDs in entry tuples | Yes, TIDs removed | [ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L455-L562) |
| Emptied entry tuples and their pages | No, never | [README#Page deletion](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396) |
| Half-full entry leaves | No, no merge and no fill factor | [ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L601-L696) |
| Slack inside posting-tree leaves | Partly; changed segments recompress, nothing re-encodes | [gindatapage.c#ginVacuumPostingTreeLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L797-L813) |
| Fully emptied posting-tree pages | Yes, deleted, then FSM-recorded a VACUUM later | [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L798-L822) |
| File length | No | [ginvacuum.c#ginvacuumcleanup-num_pages](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L787-L795) |

### Measuring: which shipped tools accept a GIN index

| Call | PostgreSQL 17 result | Evidence |
|---|---|---|
| `pgstatginindex(gin_index)` | Works. Returns only `version`, `pending_pages`, `pending_tuples`, read from the metapage | [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L507-L577) |
| `pg_relpages(gin_index)` | Works. Main-fork block count | [pgstatindex.c#pg_relpages_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L456-L474) |
| `pgstatindex(gin_index)` | `ERROR: relation "…" is not a btree index` | [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228) |
| `pgstattuple(gin_index)` | `ERROR: index "…" (gin index) is not supported` | [pgstattuple.c#pgstattuple_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L269-L297) |
| `pgstattuple_approx(gin_index)` | `ERROR: relation "…" is of wrong relation kind` | [pgstatapprox.c#pgstattuple_approx_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L283-L294) |
| `bt_index_check(gin_index)` | `ERROR: only B-Tree indexes are supported as targets for verification` | [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L423-L455) |
| `gin_metapage_info`, `gin_page_opaque_info`, `gin_leafpage_items` | Work, superuser only | [ginfuncs.c#gin_metapage_info](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95), [ginfuncs.c#gin_page_opaque_info](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L172), [ginfuncs.c#gin_leafpage_items](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L181-L285) |
| `pg_freespace(gin_index)` | Works, but binary: an index page is either offered or not | [pg_freespacemap.c#pg_freespace](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L18-L50), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55) |

Privileges differ by extension. `pgstatginindex` and `pg_freespace` are revoked from `PUBLIC` and granted to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57), [pg_freespacemap--1.1.sql#REVOKE](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L23-L25), [pg_freespacemap--1.1--1.2.sql#GRANT](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1--1.2.sql#L6-L7)). `pageinspect` has no such model: each function checks `superuser()` in C ([ginfuncs.c#superuser-check](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L42-L45), [pageinspect.sgml#superuser](../../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L10-L14)).

Two traps in the `pageinspect` path:

- `gin_page_opaque_info` validates only the special-area *size*, not the flags, and GIN, SP-GiST, and BRIN all use an 8-byte special area ([ginfuncs.c#gin_page_opaque_info-validate](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L122-L128), [ginblock.h#special-space-note](../../../../raw/postgres-17/src/include/access/ginblock.h#L19-L37)). Point it at the wrong index and you get plausible nonsense.
- `pg_freespace` on an index never returns `BLCKSZ - 1`, even though the `indexfsm.c` header comment says free index pages are recorded as `(BLCKSZ - 1)` ([indexfsm.c#header-comment](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L19)). `RecordPageWithFreeSpace` quantizes into an FSM category and reading back maps category 255 to `MaxFSMRequestSize` ([freespace.c#fsm_space_avail_to_cat](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L398-L435), [freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L249-L271)). At this pin the measured value is exactly **8160**. Test `avail > 0`, never `avail = 8191`. The documentation makes the same point qualitatively: "For indexes, what is tracked is entirely-unused pages … the values are not meaningful, just whether a page is in-use or empty." ([pgfreespacemap.sgml#indexes](../../../../raw/postgres-17/doc/src/sgml/pgfreespacemap.sgml#L67-L71))

### Recipe 1: pending-list backlog for every GIN index

The cheapest real signal. `pgstatginindex` reads only the metapage, and `nPendingPages` is the one field that is always current ([ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)).

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';
SELECT /* wiki_gin_pending_list_survey */
       c.relnamespace::regnamespace AS schema_name,
       t.relname                    AS table_name,
       c.relname                    AS index_name,
       pg_size_pretty(pg_relation_size(c.oid)) AS index_size,
       g.pending_pages,
       g.pending_tuples,
       pg_size_pretty(g.pending_pages::bigint * current_setting('block_size')::int) AS pending_bytes,
       coalesce((o.opt = 'fastupdate=off'), false) AS fastupdate_off
  FROM pg_class c
  JOIN pg_index i ON i.indexrelid = c.oid
  JOIN pg_class t ON t.oid = i.indrelid
  JOIN pg_am   a ON a.oid = c.relam
  LEFT JOIN LATERAL unnest(c.reloptions) AS o(opt)
         ON o.opt LIKE 'fastupdate=%'
 CROSS JOIN LATERAL pgstatginindex(c.oid) AS g
 WHERE a.amname = 'gin'
   AND i.indisvalid
   AND c.relpersistence <> 't'
 ORDER BY g.pending_pages DESC, c.relname;
RESET statement_timeout;
RESET lock_timeout;
```

The `indisvalid` filter is required: at this pin `pgstatginindex` raises `index "%s" is not valid` for an invalid index ([pgstatindex.c#pgstatginindex_internal-invalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L538-L543)). The temporary-relation filter avoids `cannot access temporary indexes of other sessions` ([pgstatindex.c#pgstatginindex_internal-temp](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L528-L536)).

### Recipe 2: per-page census

This is the only way to see how a GIN index actually spends its pages. It reads every block, so treat it as a maintenance-window query.

```sql
SET statement_timeout = '60s';
SELECT /* wiki_gin_page_census */
       CASE
         WHEN 'meta' = ANY (o.flags)                                  THEN 'meta'
         WHEN 'list' = ANY (o.flags)                                  THEN 'pending list (live)'
         WHEN 'data' = ANY (o.flags) AND 'deleted' = ANY (o.flags)     THEN 'posting tree page (deleted)'
         WHEN 'deleted' = ANY (o.flags)                                THEN 'former pending page (deleted)'
         WHEN 'data' = ANY (o.flags) AND 'leaf' = ANY (o.flags)        THEN 'posting tree leaf'
         WHEN 'data' = ANY (o.flags)                                   THEN 'posting tree internal'
         WHEN 'leaf' = ANY (o.flags)                                   THEN 'entry tree leaf'
         ELSE                                                              'entry tree internal'
       END AS page_class,
       count(*) AS pages,
       pg_size_pretty(count(*) * current_setting('block_size')::int) AS bytes,
       round(100.0 * count(*) / sum(count(*)) OVER (), 2) AS pct_of_index
  FROM generate_series(0, pg_relation_size('my_gin_index') / current_setting('block_size')::int - 1) AS blk,
       gin_page_opaque_info(get_raw_page('my_gin_index', blk)) AS o
 GROUP BY 1
 ORDER BY pages DESC;
RESET statement_timeout;
```

`get_raw_page` reads the main fork through shared buffers under a share lock, so it sees dirty in-memory pages ([rawpage.c#get_raw_page_internal](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199)). How to read the output:

- `pending list (live)` is a write buffer. Flush it ([Recipe 9](#recipe-9-gin_clean_pending_list-as-an-exact-pending-page-count)), do not reindex for it.
- `former pending page (deleted)` and `posting tree page (deleted)` are reusable-or-soon-reusable space inside the file. Cross-check with [Recipe 5](#recipe-5-pages-the-fsm-already-offers-for-reuse).
- A high `entry tree leaf` count with few `posting tree` pages points at [Mechanism 2](#mechanism-2-entry-tree-tuples-and-pages-are-never-removed) or [Mechanism 3](#mechanism-3-entry-leaves-settle-near-half-full-and-a-rebuild-does-not-fix-that), and only the first of those is reclaimable.

### Recipe 3: leaf fill for entry and posting leaves

PostgreSQL 17 has no GIN equivalent of `pgstatindex`'s `avg_leaf_density`, but `pageinspect`'s `page_header` exposes `pd_lower` and `pd_upper`, and their difference is the page's exact free space ([rawpage.c#page_header](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L246-L316), [bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L952-L970)). A freshly initialized GIN page has `pd_lower` at the 24-byte page header and `pd_upper` at `BLCKSZ` minus the 8-byte special area ([bufpage.c#PageInit](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L41-L60), [ginutil.c#GinInitPage](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L337-L347)), so `block_size - 32` (8160 at the default block size) is the exact denominator for an entry page. A posting-tree leaf additionally keeps an 8-byte page right bound below `pd_lower`, capping its data at `GinDataPageMaxDataSize` = 8152 ([ginblock.h#GinDataPageMaxDataSize](../../../../raw/postgres-17/src/include/access/ginblock.h#L320-L323)), so the query below understates posting-leaf fill by about 0.1 percentage point.

```sql
SET statement_timeout = '60s';
SELECT /* wiki_gin_leaf_fill */
       CASE WHEN 'data' = ANY (o.flags) THEN 'posting tree leaf' ELSE 'entry tree leaf' END AS leaf_class,
       count(*) AS leaf_pages,
       round(avg(h.upper - h.lower)::numeric, 1) AS avg_free_bytes,
       round((100.0 * (1 - avg(h.upper - h.lower)
              / (current_setting('block_size')::int - 32.0)))::numeric, 2) AS avg_leaf_fill_pct
  FROM generate_series(0, pg_relation_size('my_gin_index') / current_setting('block_size')::int - 1) AS blk,
       gin_page_opaque_info(get_raw_page('my_gin_index', blk)) AS o,
       page_header(get_raw_page('my_gin_index', blk)) AS h
 WHERE 'leaf' = ANY (o.flags)
   AND NOT ('deleted' = ANY (o.flags))
 GROUP BY 1 ORDER BY 1;
RESET statement_timeout;
```

Interpretation, from the mechanisms above:

- Entry-tree leaves near 50% are expected after any build. Values *above* that mean retail insertion densified them.
- Entry-tree leaves far *below* 50% mean the in-line posting lists inside the entry tuples shrank, because `ginVacuumEntryPage` rewrites a narrower tuple in place and never merges pages ([ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L506-L557)). Measured at this pin: a fixture that lost two thirds of its rows fell to 28.62% entry-leaf fill over 142 leaves. That space is reclaimable.
- Posting-tree leaves near 50%-75% are normal for retail churn. Single-digit fill, as in the 7.50% measurement above, is the signature of [Mechanism 5](#mechanism-5-vacuum-leaves-slack-inside-posting-tree-leaves) and is genuinely reclaimable.

### Recipe 4: metapage statistics versus live size

The metapage's page and entry counts are only as fresh as the last `ginvacuumcleanup` ([ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L55-L102), [ginutil.c#ginUpdateStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L644-L701)). Comparing them with the live block count tells you both how much the index grew since then and whether the planner still trusts them.

```sql
SELECT /* wiki_gin_metapage_staleness */
       m.n_total_pages AS meta_total_pages,
       pg_relation_size('my_gin_index') / current_setting('block_size')::int AS live_pages,
       m.n_entry_pages, m.n_data_pages, m.n_entries, m.n_pending_pages,
       round(( (pg_relation_size('my_gin_index') / current_setting('block_size')::int)::numeric
               / nullif(m.n_total_pages, 0) ), 2) AS growth_factor_since_vacuum,
       (pg_relation_size('my_gin_index') / current_setting('block_size')::int) >= 4 * m.n_total_pages
         AS planner_will_invent_stats
  FROM gin_metapage_info(get_raw_page('my_gin_index', 0)) AS m;
```

The `4 *` test mirrors `gincostestimate`'s own cutoff, described in [How bloat reaches the planner](#how-bloat-reaches-the-planner).

### Recipe 5: pages the FSM already offers for reuse

```sql
SET statement_timeout = '60s';
SELECT /* wiki_gin_fsm_reusable */
       count(*) FILTER (WHERE avail > 0) AS reusable_pages,
       pg_size_pretty((count(*) FILTER (WHERE avail > 0)) * current_setting('block_size')::int) AS reusable_bytes,
       count(*) AS pages_probed
  FROM pg_freespace('my_gin_index');
```

Read this as "space a future insert can consume without extending the file". On a quiescent index it should agree with the census: measured at this pin, 736 `former pending page (deleted)` entries in [Recipe 2](#recipe-2-per-page-census) and exactly 736 reusable pages here. It is not reclaimable-by-rebuild space, and it is populated only by `ginvacuumcleanup` and by pending-list trims ([ginvacuum.c#ginvacuumcleanup-page-loop](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L747-L787), [ginfast.c#shiftList-RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L662-L668)). The set-returning wrapper iterates main-fork blocks, so a never-vacuumed index reports all zeros ([pg_freespacemap--1.1.sql#pg_freespace](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L6-L25)).

### Measuring with core SQL only

Without any contrib extension, PostgreSQL 17 exposes no GIN internals at all. `GinStatsData` is reachable only from C, and its only two callers are the planner's `gincostestimate` and `ginNewScanKey`'s index-version check; neither is an SQL-callable inspection path ([gin.h#GinStatsData](../../../../raw/postgres-17/src/include/access/gin.h#L39-L50), [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642), [ginscan.c#ginNewScanKey-version](../../../../raw/postgres-17/src/backend/access/gin/ginscan.c#L471-L481)). The catalog contains no page-inspection function at all, and the only non-opclass `gin_*` entry is the mutator `gin_clean_pending_list` ([pg_proc.dat#gin_clean_pending_list](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L9513-L9516)).

What is left is still enough to answer "is this index growing faster than its data", in five steps: sizes and forks, catalog staleness, the VACUUM log, an `EXPLAIN` probe for the pending list, and a rebuild probe for ground truth.

### Recipe 6: size, forks, and catalog staleness

```sql
SELECT /* wiki_gin_core_size */
       c.relname AS index_name,
       pg_relation_size(c.oid) AS main_fork_bytes,
       pg_relation_size(c.oid, 'fsm') AS fsm_fork_bytes,
       pg_table_size(c.oid) AS all_forks_bytes,
       c.relpages AS recorded_pages,
       pg_relation_size(c.oid) / current_setting('block_size')::int AS live_pages,
       c.reltuples AS recorded_reltuples
  FROM pg_class c
 WHERE c.relname = 'my_gin_index';
```

Fork selection matters when you compare numbers across time:

- `pg_relation_size(idx)` is main-fork only; the one-argument form is defined as `pg_relation_size($1, 'main')` ([system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289), [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L343)). Pending-list pages live in the main fork, so this number includes them.
- `pg_table_size(idx)` and `pg_total_relation_size(idx)` add every fork, including the FSM ([dbsize.c#calculate_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L415-L443)). At this pin an FSM fork appeared only after the first VACUUM or pending-list trim, and then measured 24,576 bytes.
- `pg_indexes_size(idx)` on an *index* always returns 0, because it walks the relation's own index list and an index has none ([dbsize.c#calculate_indexes_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L445-L483)).

### The reltuples trap

`pg_class.reltuples` on a GIN index carries three incompatible meanings, and none of them is "number of index entries":

| After | Value | Evidence |
|---|---|---|
| `CREATE INDEX` / `REINDEX` | the number of *extracted keys*, summed with duplicates across rows | [gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L252-L274), [gininsert.c#ginbuild-result](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L419-L427), [index.c#index_update_stats](../../../../raw/postgres-17/src/backend/catalog/index.c#L2759-L2927) |
| `ANALYZE` | the *heap* row count, scaled by the partial-index fraction | [analyze.c#do_analyze_rel-index-stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663) |
| a full-scan `VACUUM` | the *heap* tuple count, which the source itself calls bogus | [ginvacuum.c#ginvacuumcleanup-num_index_tuples](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L726-L732) |
| a page-skipping `VACUUM` | unchanged, because `estimated_count` suppresses the catalog update | [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3098) |

Measured at this pin on one index over 100,000 rows with three keys per row and 2,988 distinct keys: `reltuples` was 300,000 after `CREATE INDEX`, 100,000 after `ANALYZE`, 300,000 again after `REINDEX`, and 90,000 after a VACUUM that removed 10,000 rows. The true entry count, 2,988, appeared only in the metapage. Use `relpages` (compared against the live block count) for staleness, and never build a bloat ratio out of a GIN index's `reltuples`.

A related measurement gap hits `pg_stat_all_indexes`: GIN sets `amgettuple = NULL` ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89)), so it is only ever driven through `index_getbitmap`, which bumps `tuples_returned` and never `tuples_fetched` ([indexam.c#index_getbitmap](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L717-L736), [indexam.c#index_fetch_heap](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L631-L655)). `idx_tup_fetch` is therefore structurally 0 for every GIN index; measured 0 against `idx_tup_read` of 268 on the same scans where a B-tree reported 74.

### Recipe 7: VACUUM VERBOSE and the autovacuum log

This is the only core channel that reports GIN page classes. For each index, VACUUM prints total pages, newly deleted pages, cumulative deleted pages, and pages recorded free ([vacuumlazy.c#verbose-per-index](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731)), fed from GIN's own counters ([ginvacuum.c#ginDeletePage-counters](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L232-L236), [ginvacuum.c#ginvacuumcleanup-pages_free](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L784-L795), [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L56-L84)).

```text
index "t_entry_gin": pages: 2051 in total, 0 newly deleted, 0 currently deleted, 0 reusable
index "t_posting_gin": pages: 9 in total, 0 newly deleted, 0 currently deleted, 0 reusable
index scan bypassed: 1 pages from table (0.06% of total) have 50 dead item identifiers
```

Those three lines are from the exact-pin run. The first is the fingerprint of [Mechanism 2](#mechanism-2-entry-tree-tuples-and-pages-are-never-removed): 2051 pages, nothing deleted, nothing reusable, after every row in the table had been deleted and vacuumed twice. To collect this continuously without running VACUUM by hand, lower `log_autovacuum_min_duration` (reload).

### Recipe 8: reading the pending-page count out of EXPLAIN

`gincostestimate` seeds its entry-page fetch count with the live pending-page count and charges each of those pages `random_page_cost` plus a fixed CPU component ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7881-L7885), [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7936-L7941), [selfuncs.c#gincostestimate-random](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7975-L7979)). A `Bitmap Index Scan` node's startup cost is forced to 0 and its total cost is set to exactly `indexTotalCost` ([createplan.c#create_bitmap_subplan](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3476-L3485)), so the plan exposes that number directly.

Two derivations, in increasing precision:

1. **Single plan.** Each pending page contributes `random_page_cost + 50 * cpu_operator_cost` (4.125 with default settings; the multiplier is [selfuncs.c#DEFAULT_PAGE_CPU_MULTIPLIER](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145)). Dividing the node's total cost by that gives an upper bound on the pending-page count.
2. **Two plans at different `random_page_cost`.** `random_page_cost` multiplies only page counts, so the cost difference divided by the `random_page_cost` difference is exactly `entryPagesFetched` plus both `dataPagesFetched` terms, which for a one-key `@>` probe is the pending-page count plus a small constant. Use a plan with one search entry, no `ScalarArrayOpExpr`, and no nested-loop repetition, because `counts.arrayScans > 1` or `loop_count > 1` routes the page counts through `index_pages_fetched` cache amortization and breaks the linearity ([selfuncs.c#gincostestimate-cache](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7956-L7973)).

```sql
SET statement_timeout = '10s';
SET enable_seqscan = off;
SET random_page_cost = 4;
EXPLAIN (COSTS ON, SUMMARY OFF) SELECT /* wiki_gin_pending_probe_lo */ count(*) FROM my_table WHERE my_col @> ARRAY[1];
SET random_page_cost = 104;
EXPLAIN (COSTS ON, SUMMARY OFF) SELECT /* wiki_gin_pending_probe_hi */ count(*) FROM my_table WHERE my_col @> ARRAY[1];
RESET random_page_cost;
RESET enable_seqscan;
RESET statement_timeout;
```

Take the two `Bitmap Index Scan` total costs and compute `(hi - lo) / 100`. Measured twice at this pin: 591.00 against 589 real pending pages in a 591-page index, and 101.00 against 99 real pending pages in a 101-page index. Both `random_page_cost` and `enable_seqscan` are `PGC_USERSET`, so this is a session-scoped, read-only probe. It is a probe of the *planner's* input, so it also silently confirms whether the metapage statistics are stale.

### Recipe 9: gin_clean_pending_list as an exact pending-page count

`gin_clean_pending_list` returns `stats.pages_deleted`, accumulated by `shiftList` as it unlinks pending pages ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091), [ginfast.c#shiftList-stats](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L590-L591)). That is an exact count, at the price of flushing the list.

```sql
SET statement_timeout = '15min';
SET lock_timeout = '5s';
SELECT /* wiki_gin_flush_pending_list */ gin_clean_pending_list('my_gin_index') AS pending_pages_removed;
RESET lock_timeout;
RESET statement_timeout;
```

Expect a long runtime on a large list: the flush inserts every pending entry into the entry tree, budgeted by `maintenance_work_mem` ([ginfast.c#ginInsertCleanup-workMemory](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L817)). A `fastupdate = off` index and an already-empty list both return 0 ([ginfast.c#ginInsertCleanup-empty](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L835-L841), [func.sgml#gin_clean_pending_list](../../../../raw/postgres-17/doc/src/sgml/func.sgml#L30118-L30137)). Measured at this pin: 1471, then 0 on the second call, and 0 on a `fastupdate = off` index. The flush *grew* the main fork from 12,066,816 to 15,581,184 bytes, because the entry tree had to be built while the 1471 emptied pending pages stayed in place.

### Recipe 10: the rebuild probe, the only ground truth

Because no shipped function reports GIN density, the only exact answer to "how many bytes would a rebuild reclaim" is to build a copy and compare. `CREATE INDEX CONCURRENTLY` keeps the table writable, at the cost of two heap scans and three waits ([indexcmds.c#DefineIndex-phase2](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1627-L1642), [indexcmds.c#DefineIndex-phase3](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1682-L1689), [indexcmds.c#DefineIndex-WaitForOlderSnapshots](../../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1750-L1752)).

```sql
SET lock_timeout = '5s';
CREATE INDEX CONCURRENTLY wiki_gin_probe ON my_table USING gin (my_col);
SELECT /* wiki_gin_rebuild_probe */
       pg_relation_size('my_gin_index') AS current_bytes,
       pg_relation_size('wiki_gin_probe') AS fresh_bytes,
       pg_relation_size('my_gin_index') - pg_relation_size('wiki_gin_probe') AS reclaimable_bytes,
       round(100.0 * (pg_relation_size('my_gin_index') - pg_relation_size('wiki_gin_probe'))
             / nullif(pg_relation_size('my_gin_index'), 0), 2) AS pct_reclaimable;
DROP INDEX CONCURRENTLY wiki_gin_probe;
RESET lock_timeout;
```

Rules for using it honestly:

- Reproduce the *exact* logical definition of the index being probed, including every column, the operator class, any `WHERE` predicate, and the `fastupdate` reloption. A probe with different reloptions measures a different index.
- Do not set `statement_timeout` for the build: a concurrent build runs in several internal transactions and can legitimately take a long time. Keep `lock_timeout` low so the probe cannot queue behind DDL.
- The probe doubles the write and WAL cost of a rebuild you may then perform anyway. If the answer is "reindex", `REINDEX INDEX CONCURRENTLY` alone is cheaper than probe-then-reindex.
- A negative `reclaimable_bytes` is a real answer, not an error. See [Mechanism 3](#mechanism-3-entry-leaves-settle-near-half-full-and-a-rebuild-does-not-fix-that).

Measured at this pin: on a churned fixture the probe reported `fresh_bytes` 131,072 against `current_bytes` 2,064,384, i.e. 93.65% reclaimable, and the subsequent `REINDEX` produced exactly 131,072 bytes. On an already-rebuilt index the probe reported 0.00%.

### How bloat reaches the planner

GIN bloat reaches the planner through three separate channels.

1. **Live block count.** For a non-partial index the planner uses the live `RelationGetNumberOfBlocks` value, not `pg_class.relpages` ([plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L486)). Growth is therefore visible immediately, with no ANALYZE.
2. **Metapage statistics, with a staleness cutoff.** `gincostestimate` trusts the metapage's page and entry counts only if they are at most the live size and more than a quarter of it, scaling them by the growth ratio; otherwise it invents 90% entry pages, 10% data pages, and 100 entries per entry page ([selfuncs.c#gincostestimate-scale](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7712-L7766)). A GIN index that has grown more than 4x since its last VACUUM is costed from invented numbers.
3. **Pending pages as pure startup cost.** Every pending page is charged as an entry page fetched before the scan starts ([selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7881-L7885)). This is the dominant plan-level effect of GIN bloat, and it is what [Recipe 8](#recipe-8-reading-the-pending-page-count-out-of-explain) measures.

Measured at this pin: a 1471-page pending list raised a forced `Bitmap Index Scan` estimate to 6260.82 with 1473 buffers, against 17.63 and 4 buffers for the same query once the list was flushed. Note the direction: bloat makes GIN scans look *expensive*, so a bloated GIN index can be planned away in favour of a sequential scan.

### Settings, and their apply scope

| Setting | Scope | Effect on GIN bloat |
|---|---|---|
| `gin_pending_list_limit` | `PGC_USERSET`; session/transaction, no restart or reload ([guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3575-L3584), measured `context = user`) | Capacity threshold at which an inserting backend trims the pending list; default 4 MB ([config.sgml#gin_pending_list_limit](../../../../raw/postgres-17/doc/src/sgml/config.sgml#L9793-L9814)) |
| `fastupdate` reloption | `ALTER INDEX`, takes `AccessExclusiveLock`; does not flush what is already pending | Turns the pending list on or off ([create_index.sgml#fastupdate](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L510-L535)) |
| `gin_pending_list_limit` reloption | `ALTER INDEX`, takes `AccessExclusiveLock` | Per-index override of the GUC ([ginutil.c#ginoptions](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614)) |
| `maintenance_work_mem` | `PGC_USERSET`; session/transaction | Budget for a forced pending-list flush and for the build accumulator ([ginfast.c#ginInsertCleanup-workMemory](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L817), [gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314)) |
| `autovacuum_work_mem` | `PGC_SIGHUP`; reload | Replaces `maintenance_work_mem` for an autovacuum worker's flush ([ginfast.c#ginInsertCleanup-workMemory](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L807-L817)) |
| `work_mem` | `PGC_USERSET`; session/transaction | Budget for the *insert-triggered* flush, which is the common case under `fastupdate` ([ginfast.c#ginInsertCleanup-locking](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L818-L828)) |
| `vacuum_failsafe_age` | `PGC_USERSET`; session/transaction (measured `context = user`) | Below this age VACUUM still cleans indexes; above it, GIN is skipped entirely ([vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2299-L2347)) |
| `autovacuum_vacuum_scale_factor`, `autovacuum_vacuum_threshold` | `PGC_SIGHUP`; reload (also per-table reloptions) | How often the pending list gets flushed and dead TIDs removed |
| `autovacuum_analyze_scale_factor` | `PGC_SIGHUP`; reload | Autoanalyze also performs a partial pending-list clean ([ginvacuum.c#ginvacuumcleanup-analyze_only](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L698-L710)) |
| `log_autovacuum_min_duration` | `PGC_SIGHUP`; reload | Makes [Recipe 7](#recipe-7-vacuum-verbose-and-the-autovacuum-log) continuous |
| `max_parallel_maintenance_workers`, `min_parallel_index_scan_size` | `PGC_USERSET`; session/transaction | GIN participates in parallel VACUUM for both bulkdelete and cleanup ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L58-L61)), and each `maintenance_work_mem`-using index gets a share ([vacuumparallel.c#parallel_vacuum_init](../../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L370-L375)) |
| `random_page_cost`, `cpu_operator_cost` | `PGC_USERSET`; session/transaction | Scale factors in [Recipe 8](#recipe-8-reading-the-pending-page-count-out-of-explain) |

There is no `fillfactor` for GIN. `CREATE INDEX … WITH (fillfactor = 70)` on a GIN index fails at this pin with `ERROR: unrecognized parameter "fillfactor"`.

### Exact-pin measurements

Built PostgreSQL 17.10 from the pinned checkout, ran an isolated server with `autovacuum = off` and `maintenance_work_mem = 64MB`, and installed `pgstattuple`, `pageinspect`, `pg_freespacemap`, `amcheck`, `btree_gin`, and `pg_trgm`. All nine SQL blocks filed above ran verbatim with `ON_ERROR_STOP=1`, against a fixture whose objects were literally named `my_table`, `my_col`, and `my_gin_index` so that no placeholder had to be edited.

**Pending list.** 200,000 rows with 3 keys each, `fastupdate` on, `gin_pending_list_limit` raised to 512 MB for the loading session: 1471 pending pages and 200,000 pending tuples in a 12,066,816-byte index whose census was 1 metapage, 1 entry leaf, and 1471 live list pages. `ANALYZE` left the count at 1471. `gin_clean_pending_list` returned 1471; afterwards the index held 1 metapage, 3 entry internal pages, 427 entry leaves, and 1471 deleted former pending pages, the main fork had grown to 15,581,184 bytes, a 24,576-byte FSM fork had appeared, and all 1471 pages were recorded free at `avail = 8160`. The forced `Bitmap Index Scan` estimate fell from 6260.82 to 17.63 and its buffers from 1473 to 4.

**Entry-tree retention.** 300,000 rows with 300,000 distinct keys built a 16,801,792-byte index. After deleting all 300,000 rows and vacuuming twice: still 16,801,792 bytes, metapage still reporting 2050 entry pages and 300,000 entries, census 2040 entry leaves plus 10 entry internal pages, zero deleted pages, zero FSM pages, and `VACUUM VERBOSE` reporting `0 newly deleted, 0 currently deleted, 0 reusable`. `REINDEX` produced 16,384 bytes, a 1025x reduction.

**Entry-leaf density.** 400,000 rows over 100,000 distinct keys. Scattered retail insertion: 933 entry leaves, 57.79% average fill, 7,692,288 bytes. `REINDEX` of the same rows: 1063 leaves, 50.73% fill, 8,765,440 bytes, i.e. 14.0% larger. Ascending retail insertion: 877 leaves, 50.31% fill, 7,233,536 bytes, byte-identical after `REINDEX`. `maintenance_work_mem` of 1 MB and 64 MB produced the same built index.

**Posting trees.** 800,000 rows over 20 distinct keys built 1,163,264 bytes as 120 posting leaves and 20 posting internal pages. Deleting 760,000 rows and vacuuming left the size unchanged with 44 posting leaves, 76 deleted posting pages, and zero FSM entries; three `txid_current()` calls followed by another VACUUM recorded all 76 pages free; `REINDEX` produced 73,728 bytes as 6 posting leaves and 1 internal page.

**Churn fixture and rebuild probe.** A 600,000-row load over 50 keys, 96% deleted, 20,000 rows re-inserted, then vacuumed twice: 2,064,384 bytes as 102 posting leaves at 7.50% fill, 50 posting internal pages, 50 former pending pages, 48 deleted posting pages, 1 metapage, and 1 entry leaf at 12.25% fill, with 98 pages (784 kB) offered by the FSM. The `CREATE INDEX CONCURRENTLY` probe reported 131,072 fresh bytes and 93.65% reclaimable; the subsequent `REINDEX` produced exactly 131,072 bytes. Ordinary VACUUM never shrank the file, including after every row was deleted; `VACUUM FULL` cut it to 16,384 bytes.

**Pending-page probe.** Fixture with 589 pending pages in a 591-page index: costs 2512.77 at `random_page_cost = 4` and 4876.77 at 8, so `(4876.77 - 2512.77) / 4 = 591.00`. Second fixture with 99 pending pages in a 101-page index: 430.00 at 4 and 10530.00 at 104, so `(10530.00 - 430.00) / 100 = 101.00`.

**Catalog and statistics.** One index over 100,000 rows, 3 keys per row, 2,988 distinct keys: `reltuples` 300,000 after `CREATE INDEX`, 100,000 after `ANALYZE`, 300,000 after `REINDEX`, 90,000 after a VACUUM that removed 10,000 rows; metapage `n_entries` 2,988 throughout. `idx_tup_fetch` was 0 for the GIN index with `idx_tup_read` 268, while a B-tree on the same table reported `idx_tup_fetch` 74. `pg_indexes_size` on a GIN index returned 0; `pg_table_size` exceeded `pg_relation_size` by exactly the FSM fork.

**Tool acceptance.** `pgstatginindex` returned `(2, 0, 0)` on a GIN index and failed on a B-tree with `relation "t_control_pkey" is not a GIN index`. `pgstatindex`, `pgstattuple`, `pgstattuple_approx`, and `bt_index_check` failed on the GIN index with the messages tabulated above. `pg_relpages` worked on both index types. `ALTER INDEX … SET (fastupdate = off)` left all 589 pending pages in place until `gin_clean_pending_list` removed them. Deleting 50 of 200,000 rows produced `index scan bypassed: 1 pages from table (0.06% of total) have 50 dead item identifiers`.

### Key data structures

| Structure | Role |
|---|---|
| [ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L55-L102) | Pending-list head/tail, pending page and tuple counts, and the VACUUM-time page/entry statistics |
| [ginblock.h#GinPageOpaqueData](../../../../raw/postgres-17/src/include/access/ginblock.h#L30-L39) | Per-page rightlink, `maxoff`, and the flags word that names the page class |
| [gin.h#GinStatsData](../../../../raw/postgres-17/src/include/access/gin.h#L39-L50) | C-only carrier for the metapage statistics, read by the planner |
| [gin_private.h#GinOptions](../../../../raw/postgres-17/src/include/access/gin_private.h#L23-L31) | The two GIN reloptions: `fastupdate` and `gin_pending_list_limit` |
| [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L56-L84) | `num_pages`, `pages_newly_deleted`, `pages_deleted`, `pages_free` as surfaced by `VACUUM VERBOSE` |
| [gindatapage.c#disassembledLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L45-L68) | In-memory segment list used for posting-leaf repacking and vacuuming |
| [gin_private.h#BuildAccumulator](../../../../raw/postgres-17/src/include/access/gin_private.h#L431-L439) | Red-black accumulator that makes a build insert keys in ascending order |

### Caller and callee boundary

- Insert path: `gininsert` -> `ginHeapTupleFastCollect` -> `ginHeapTupleFastInsert` -> conditional `ginInsertCleanup` ([gininsert.c#gininsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L482-L536), [ginfast.c#ginHeapTupleFastInsert](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L218-L472)).
- Non-fastupdate insert path: `gininsert` -> `ginHeapTupleInsert` -> `ginEntryInsert` -> `addItemPointersToLeafTuple` or `buildFreshLeafTuple` -> `ginInsertItemPointers` ([gininsert.c#ginHeapTupleInsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L464-L480), [gininsert.c#ginEntryInsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L175-L244)).
- VACUUM path: `vacuum_one_index` -> `ginbulkdelete` (entry-leaf walk, then deferred `ginVacuumPostingTree` per posting root) and `ginvacuumcleanup` (full page scan, FSM recording, `ginUpdateStats`) ([ginvacuum.c#ginbulkdelete](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L564-L685), [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L687-L796)).
- ANALYZE path: `do_analyze_rel` -> `index_vacuum_cleanup` with `analyze_only = true`, which GIN honours only for an autovacuum worker ([analyze.c#do_analyze_rel-cleanup](../../../../raw/postgres-17/src/backend/commands/analyze.c#L704-L720)).
- Planner path: `get_relation_info` records the live block count, then `gincostestimate` reads the metapage on every plan ([plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L486), [selfuncs.c#gincostestimate-stats](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7696-L7710)).
- Page allocation: every GIN page comes from `GinNewBuffer`, which drains the FSM first and extends the file only when nothing is recyclable ([ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335)).

### Build, generated-header, and extension boundary

- `GIN_AM_OID` comes from the generated catalog header produced by `genbki.pl` from `pg_am.dat`, and both `gin_clean_pending_list` and `pgstatginindex` compare `relam` against it ([pg_am.dat#gin](../../../../raw/postgres-17/src/include/catalog/pg_am.dat#L26-L29), [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1043-L1049)).
- `gin_clean_pending_list` is a core catalog function, so it needs no extension ([pg_proc.dat#gin_clean_pending_list](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L9513-L9516)).
- `pgstatginindex` exists in two C entry points: the pre-1.5 one enforces `superuser()`, the 1.5 one relies on the SQL-level `REVOKE`/`GRANT` ([pgstatindex.c#pgstatginindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L484-L504)).
- `pageinspect`'s GIN functions are declared in the base install script and unchanged by later upgrade scripts ([pageinspect--1.5.sql#gin](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.5.sql#L239-L279)).
- GIN declares `amcanbuildparallel = false` and `amcanparallel = false`, so no parallel build or parallel scan is available at this pin; `amusemaintenanceworkmem = true` and the parallel-vacuum options are what put GIN into parallel VACUUM ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L41-L62)).
- GIN sets `ambuildphasename = NULL`, so `pg_stat_progress_create_index` reports no named phase for a GIN build ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L64-L76)).

### Tests and explicit test absence

Present at this pin:

- Core GIN regression coverage exercises `gin_clean_pending_list`, including the "nothing to flush" case ([gin.sql#pending](../../../../raw/postgres-17/src/test/regress/sql/gin.sql#L1-L25), [gin.out#pending](../../../../raw/postgres-17/src/test/regress/expected/gin.out#L1-L25)).
- `pgstattuple` tests assert `pgstatginindex` on a GIN index and the rejection paths on other AMs, partitioned tables, views, and foreign tables ([pgstattuple.sql#gin](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L45-L115), [pgstattuple.out#gin](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L120-L160)).
- `pageinspect` tests cover all three GIN functions plus their validation errors and all-zero-page behaviour ([pageinspect/gin.sql](../../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L1-L41), [pageinspect/gin.out](../../../../raw/postgres-17/contrib/pageinspect/expected/gin.out#L1-L71)).
- `pg_freespacemap` tests assert only that a recorded index page has `avail > 0`, never a specific value ([pg_freespacemap.sql](../../../../raw/postgres-17/contrib/pg_freespacemap/sql/pg_freespacemap.sql#L1-L32)).

Absent at this pin:

- No test asserts any GIN density, occupancy, or bloat figure.
- No test asserts that an entry page survives deletion of all its rows.
- No test asserts the autovacuum partial pending-list clean, only the full clean through `gin_clean_pending_list`.
- No test asserts the delayed FSM publication of deleted posting-tree pages.

### What changed since PostgreSQL 12

GIN's own bloat mechanics are unchanged. Every behavioural difference sits around GIN: when VACUUM calls it, how much memory it gets, how the planner prices it, and what the measurement tools accept. Attributions below come from the pinned checkout's own history, bracketed by the release stamp commits `Stamp 12.0` (`ad1f2885b8c`) through `Stamp 17.0` (`d7ec59a63d7`); all listed commits are ancestors of the pin.

| First in | Commit | What it changes for GIN bloat or measurement |
|---|---|---|
| 13 | `4d8a8d0c738` "Introduce IndexAM fields for parallel vacuum" | GIN opts into parallel VACUUM for both bulkdelete and cleanup and declares `amusemaintenanceworkmem`, so a parallel VACUUM splits `maintenance_work_mem` across `maintenance_work_mem`-using indexes ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L58-L61), [vacuumparallel.c#parallel_vacuum_init](../../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L370-L375)) |
| 13 | `ec28808ba85` "Fix ginEntryInsert's counting of GIN leaf tuples" | Metapage `nEntries` after a build now counts leaf tuples rather than `ginEntryInsert` calls, so build-time and VACUUM-time entry counts finally agree ([gininsert.c#ginEntryInsert-nEntries](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L232-L238)) |
| 13 | `4b754d6c16e` "Avoid full scan of GIN indexes when possible" | Full-index-scan costing became per-attribute, changing GIN plan costs relative to v12 ([selfuncs.c#GinQualCounts](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7369-L7377), [selfuncs.c#gincostestimate-fullscan](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7851-L7876)) |
| 14 | `5100010ee4d` "Teach VACUUM to bypass unnecessary index vacuuming" and `3499df0dee8` "Support disabling index bypassing by VACUUM" | The 2%-of-pages bypass can skip `ginbulkdelete` entirely, leaving dead TIDs in the entry tree longer; `INDEX_CLEANUP` became a three-valued option so `ON` can force the work ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1946)) |
| 14 | `1e55e7d1755` "Add wraparound failsafe to VACUUM" and `c242baa4a83` "Consider triggering VACUUM failsafe during scan" | An aged table gets neither GIN bulkdelete nor pending-list cleanup, so a pending list can survive a completed VACUUM ([vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2299-L2347)) |
| 14 | `dc7420c2c92` "snapshot scalability: Don't compute global horizons while building snapshots" | The deleted-page recycling gate moved from a `RecentGlobalXmin` macro to the `GlobalVisCheckRemovableXid` function, so the horizon can be fresher ([ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L798-L822)) |
| 14 | `23763618390` "VACUUM VERBOSE: Count \"newly deleted\" index pages" | Added the `newly deleted` column that makes [Recipe 7](#recipe-7-vacuum-verbose-and-the-autovacuum-log) distinguish this VACUUM's deletions from the cumulative total ([ginvacuum.c#ginDeletePage-counters](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L232-L236)) |
| 16 | `eb5c4e953bb` "Extract the multiplier for CPU process cost of index page into a macro" and `cd9479af2af` "Improve GIN cost estimation" | Added the `DEFAULT_PAGE_CPU_MULTIPLIER` CPU charges per fetched page. This is why the v17 pending-page probe divides by `random_page_cost + 50 * cpu_operator_cost` rather than by `random_page_cost` alone ([selfuncs.c#DEFAULT_PAGE_CPU_MULTIPLIER](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L145), [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7936-L7941)) |
| 17 | `667e65aac35` "Use TidStore for dead tuple TIDs storage during lazy vacuum" | Dead-item storage is now a radix tree without the old array size cap, so a large VACUUM makes fewer full `ginbulkdelete` passes; `pg_stat_progress_vacuum` reports `max_dead_tuple_bytes` instead of `max_dead_tuples` ([vacuumlazy.c#dead_items_alloc](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2871-L2882)) |
| 17 | `b4375717147` "Allow parallel CREATE INDEX for BRIN indexes" | Introduced `amcanbuildparallel`; GIN sets it to `false`, which is the explicit statement that v17 has no parallel GIN build ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L55-L56)) |
| 17 (also back-patched to 12.x as `975ae05537f`) | `13503eb5905` "Diagnose !indisvalid in more SQL functions" | `pgstatginindex` now errors on an invalid index and `gin_clean_pending_list` became a `DEBUG1` no-op, which is why [Recipe 1](#recipe-1-pending-list-backlog-for-every-gin-index) filters on `indisvalid` ([pgstatindex.c#pgstatginindex_internal-invalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L538-L543), [ginfast.c#gin_clean_pending_list-invalid](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1068-L1086)) |

No policy inside the access method changed. Apart from the rows above, every commit touching `src/backend/access/gin` between `Stamp 12.0` and the pin is either a correctness fix that was also back-patched into the 12 branch (`b1071408048`, `d5ad7a09afd`, `e14641197a5`, `4093ff57370`, `bde7493d108`, `e41955faf06`, `1df14a56694`, and the v17-cycle `6a1ea02c491`, `5c62ecf6ec3`, `126552c85c1`), the v13 opclass-parameters refactor `911e7020770` that only reshapes `ginoptions`, or a comment, README, or copyright edit. So the pending-list trigger and trim algorithm, the autovacuum-versus-manual `full_clean` asymmetry, autoanalyze cleaning the list while manual `ANALYZE` does not, posting-tree and entry-page split policy, the VACUUM two-pass shape and its refusal to re-encode segments, the absence of entry-tree deletion, and `gin_pending_list_limit`'s 4 MB default and `PGC_USERSET` context all stand as described above. `pgstatginindex` still returns the same three columns and `pageinspect` still ships the same three superuser-only GIN functions. Still absent at this pin: parallel GIN build, GIN support in `amcheck`, a GIN `fillfactor`, and any documentation of GIN bloat mechanics.

## Context Reviewed

- GIN access method: `src/backend/access/gin/ginfast.c`, `ginvacuum.c`, `gindatapage.c`, `ginentrypage.c`, `gininsert.c`, `ginutil.c`, `ginbtree.c`, `ginbulk.c`, `ginscan.c`, and `README`.
- GIN headers: `src/include/access/gin.h`, `gin_private.h`, `ginblock.h`.
- VACUUM and ANALYZE: `src/backend/access/heap/vacuumlazy.c`, `src/backend/commands/vacuum.c`, `vacuumparallel.c`, `analyze.c`, `src/include/access/genam.h`.
- Planner: `src/backend/utils/adt/selfuncs.c`, `src/backend/optimizer/util/plancat.c`, `src/backend/optimizer/plan/createplan.c`.
- Storage and size: `src/backend/storage/freespace/indexfsm.c`, `freespace.c`, `src/backend/utils/adt/dbsize.c`, `src/backend/catalog/system_functions.sql`, `src/backend/catalog/index.c`.
- Statistics plumbing: `src/backend/access/index/indexam.c`, `src/include/pgstat.h`, `src/backend/catalog/system_views.sql`.
- Catalogs and GUCs: `src/include/catalog/pg_proc.dat`, `pg_am.dat`, `src/backend/utils/misc/guc_tables.c`.
- Contrib: `contrib/pgstattuple/`, `contrib/pageinspect/`, `contrib/pg_freespacemap/`, `contrib/amcheck/`.
- Documentation: `doc/src/sgml/gin.sgml`, `maintenance.sgml`, `ref/create_index.sgml`, `config.sgml`, `func.sgml`, `pgstattuple.sgml`, `pageinspect.sgml`, `pgfreespacemap.sgml`.
- Tests: `src/test/regress/sql/gin.sql`, `contrib/pgstattuple/sql/pgstattuple.sql`, `contrib/pageinspect/sql/gin.sql`, `contrib/pg_freespacemap/sql/pg_freespacemap.sql`.
- Source history of the pinned checkout, bracketed by the `Stamp 12.0` through `Stamp 17.0` commits, for every attribution in [What changed since PostgreSQL 12](#what-changed-since-postgresql-12).
- An isolated exact-pin PostgreSQL 17.10 server built from the pinned checkout, used for every measurement quoted above.

## Evidence Map

| Claim | Source |
|---|---|
| `fastupdate` defaults to on | [gin_private.h#GIN_DEFAULT_USE_FASTUPDATE](../../../../raw/postgres-17/src/include/access/gin_private.h#L33-L45) |
| The trim trigger counts page capacity, not bytes used | [ginfast.c#ginHeapTupleFastInsert-trigger](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L471) |
| A row whose entries span pages wastes the tail of the previous page | [README#GIN_LIST_FULLROW](../../../../raw/postgres-17/src/backend/access/gin/README#L201-L216) |
| An insert-driven flush bails out on contention and uses `work_mem` | [ginfast.c#ginInsertCleanup-locking](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L800-L828) |
| Autovacuum requests a partial clean, manual VACUUM a full one | [ginvacuum.c#ginbulkdelete-pending](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L584-L595) |
| Autoanalyze cleans the list; manual ANALYZE does not | [ginvacuum.c#ginvacuumcleanup-analyze_only](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L698-L710) |
| Trimmed pending pages get `GIN_DELETED` and enter the FSM | [ginfast.c#shiftList-RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L662-L668) |
| Turning `fastupdate` off does not flush the list | [create_index.sgml#fastupdate](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L510-L535) |
| Vacuum never deletes entry-tree tuples or pages | [README#Page deletion](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396) |
| An emptied entry tuple is rewritten with a null posting list | [ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L506-L557) |
| Entry splits equalize data size with no rightmost or build case | [ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691) |
| Entry insertion reserves no fill factor | [ginentrypage.c#entryIsEnoughSpace](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482) |
| A build drains its accumulator in ascending key order | [ginbulk.c#ginBeginBAScan](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L255-L293) |
| GIN has only two reloptions | [ginutil.c#ginoptions](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614) |
| Posting-leaf splits are build-, append-, and balance-sensitive | [gindatapage.c#dataBeginPlaceToPageLeaf-split](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L617-L667) |
| VACUUM does not re-encode posting segments | [gindatapage.c#ginVacuumPostingTreeLeaf-no-reencode](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L797-L813) |
| Only completely empty non-edge posting pages are deleted | [ginvacuum.c#ginScanToDelete](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L303-L318) |
| Recycling waits for global visibility of the delete XID | [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L798-L822) |
| Only `ginvacuumcleanup` publishes recyclable pages to the FSM | [ginvacuum.c#ginvacuumcleanup-page-loop](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L747-L787) |
| Ordinary VACUUM does not shorten the fork | [ginvacuum.c#ginvacuumcleanup-num_pages](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L787-L795) |
| Pages are reused through `GinNewBuffer` before the file grows | [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335) |
| VACUUM can bypass index vacuuming below 2% of pages | [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1946) |
| The failsafe skips index vacuuming and cleanup together | [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2299-L2347) |
| VACUUM prints per-index page classes | [vacuumlazy.c#verbose-per-index](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731) |
| `pgstatginindex` reads only the metapage and returns three columns | [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L507-L577) |
| `pgstatindex` rejects a GIN index | [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228) |
| `pgstattuple` rejects a GIN index | [pgstattuple.c#pgstattuple_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L269-L297) |
| `pgstattuple_approx` rejects any index | [pgstatapprox.c#pgstattuple_approx_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatapprox.c#L283-L294) |
| `amcheck` rejects a GIN index | [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-17/contrib/amcheck/verify_nbtree.c#L423-L455) |
| `gin_page_opaque_info` validates size but not flags | [ginfuncs.c#gin_page_opaque_info-validate](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L122-L128) |
| `pageinspect` GIN functions are superuser-only | [ginfuncs.c#superuser-check](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L42-L45) |
| Free index pages are FSM-quantized, not stored as `BLCKSZ - 1` | [freespace.c#fsm_space_avail_to_cat](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L398-L435) |
| `page_header` exposes the exact free space of any page | [rawpage.c#page_header](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L246-L316) |
| `pg_relation_size` is main-fork only | [system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289) |
| `pg_indexes_size` on an index returns zero | [dbsize.c#calculate_indexes_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L445-L483) |
| Build-time `reltuples` counts extracted keys | [gininsert.c#ginHeapTupleBulkInsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L252-L274) |
| VACUUM reports the heap tuple count as the index tuple count | [ginvacuum.c#ginvacuumcleanup-num_index_tuples](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L726-L732) |
| A page-skipping VACUUM does not update index `pg_class` rows | [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3098) |
| ANALYZE sets index `reltuples` from the heap row count | [analyze.c#do_analyze_rel-index-stats](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663) |
| GIN has no `amgettuple`, so `idx_tup_fetch` stays zero | [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89) |
| Pending pages are charged as startup cost | [selfuncs.c#gincostestimate-pending](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7881-L7885) |
| Each fetched page also carries a fixed CPU charge | [selfuncs.c#gincostestimate-page-cpu](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7936-L7941) |
| `random_page_cost` is applied to fetched pages | [selfuncs.c#gincostestimate-random](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7975-L7979) |
| A Bitmap Index Scan node's total cost equals `indexTotalCost` | [createplan.c#create_bitmap_subplan](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3476-L3485) |
| Metapage statistics are distrusted beyond 4x growth | [selfuncs.c#gincostestimate-scale](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7712-L7766) |
| The planner uses the live block count for a non-partial index | [plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L486) |
| `gin_clean_pending_list` returns removed pending pages | [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091) |
| `gin_pending_list_limit` is `PGC_USERSET` with a 4 MB default | [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3575-L3584) |
| GIN cannot build in parallel at this pin | [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L55-L56) |
| Non-B-tree bloat is documented as under-researched | [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1036-L1040) |

## Open Questions

- The autovacuum partial pending-list clean was not reproduced. The fixtures used manual VACUUM and `gin_clean_pending_list`, both full cleans, so the size of a residue left by an autovacuum worker under concurrent inserts is unmeasured at this pin.
- The measured entry-leaf fill figures (50.31%, 50.73%, 52.72%, 57.79%) come from four fixtures over `int[]` columns. Whether the same ~50% built-density result holds for `tsvector`, `jsonb`, and `pg_trgm` opclasses, whose entry tuples have very different width distributions, was not tested.
- No steady-state occupancy target or bloat threshold for GIN exists anywhere in the pinned tree, so this page offers no "N% means reindex" rule. The only defensible trigger remains the rebuild probe in [Recipe 10](#recipe-10-the-rebuild-probe-the-only-ground-truth) or sustained growth of physical size against row count.
- The `indexfsm.c` header comment states that unused index pages are recorded as `(BLCKSZ - 1)`, which is true of the value passed in but never of the value stored or read back. The exact-pin measurement is 8160. This is a source-comment inconsistency, not a behavioural claim; no in-tree test asserts either value.
- The pending-page probe in [Recipe 8](#recipe-8-reading-the-pending-page-count-out-of-explain) was validated on two fixtures whose metapage statistics were stale enough to take `gincostestimate`'s invented-statistics path. Its accuracy on an index with fresh metapage statistics and many entry pages, where `entryPagesFetched` has a larger non-pending component, is unmeasured.
- `gin_leafpage_items` was not used to quantify within-segment slack in posting leaves. [Recipe 3](#recipe-3-leaf-fill-for-entry-and-posting-leaves) measures page-level free space only, so segment-level waste inside a nominally full posting leaf is not covered.
- The `VACUUM VERBOSE` `reusable` column reports `pages_free` from the most recent `ginvacuumcleanup` only. Whether it can be reconciled exactly with `pg_freespace` on a busy index, where `GinNewBuffer` consumes FSM entries concurrently, was not tested.
- Whether v17's `TidStore` change measurably reduces repeated `ginbulkdelete` passes on a large GIN index was reasoned from source, not measured; no fixture here was large enough to exhaust the pre-v17 dead-item array.

## Source References

- [ginfast.c#ginHeapTupleFastInsert](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L218-L472)
- [ginfast.c#shiftList](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L553-L671)
- [ginfast.c#ginInsertCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L779-L1025)
- [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091)
- [ginvacuum.c#ginDeletePage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L129-L236)
- [ginvacuum.c#ginScanToDelete](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L246-L338)
- [ginvacuum.c#ginVacuumEntryPage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L455-L562)
- [ginvacuum.c#ginbulkdelete](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L564-L685)
- [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L687-L796)
- [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L798-L822)
- [gindatapage.c#dataBeginPlaceToPageLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L447-L707)
- [gindatapage.c#ginVacuumPostingTreeLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L737-L864)
- [gindatapage.c#dataSplitPageInternal](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L1251-L1327)
- [ginentrypage.c#entryIsEnoughSpace](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482)
- [ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L601-L696)
- [gininsert.c#ginEntryInsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L175-L244)
- [gininsert.c#ginbuild](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L316-L428)
- [gininsert.c#gininsert](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L482-L536)
- [ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L89)
- [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335)
- [ginutil.c#ginGetStats](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L616-L642)
- [ginbulk.c#ginBeginBAScan](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L255-L293)
- [README#Page deletion](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396)
- [ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L55-L102)
- [gin.h#GinStatsData](../../../../raw/postgres-17/src/include/access/gin.h#L39-L50)
- [genam.h#IndexBulkDeleteResult](../../../../raw/postgres-17/src/include/access/genam.h#L56-L84)
- [vacuumlazy.c#lazy_vacuum-bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1946)
- [vacuumlazy.c#verbose-per-index](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731)
- [selfuncs.c#gincostestimate](../../../../raw/postgres-17/src/backend/utils/adt/selfuncs.c#L7661-L8049)
- [plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L463-L486)
- [createplan.c#create_bitmap_subplan](../../../../raw/postgres-17/src/backend/optimizer/plan/createplan.c#L3476-L3485)
- [dbsize.c#calculate_table_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L415-L443)
- [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)
- [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L507-L577)
- [ginfuncs.c#gin_metapage_info](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95)
- [pg_freespacemap.c#pg_freespace](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L18-L50)
- [gin.sgml#gin-fast-update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L538)
- [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1036-L1040)
- [gin.sql#pending](../../../../raw/postgres-17/src/test/regress/sql/gin.sql#L1-L25)

## Navigation

- [v17/index](../../index.md)
- [PostgreSQL 17 Codebase Navigation Guide](../../codebase-navigation-guide.md)
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 17](pgstatindex-sample-variant-proposal.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](reindex-index-concurrently.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md)
- [PostgreSQL 17 Contrib Extensions](../server-administration/contrib-extensions.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
