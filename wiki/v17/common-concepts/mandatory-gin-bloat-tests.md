---
type: common-concept
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Mandatory GIN Bloat Tests (unverified)

## Contents

- [Definition](#definition)
- [Why It Exists](#why-it-exists)
- [How It Works](#how-it-works)
  - [The five phases of a run](#the-five-phases-of-a-run)
  - [The settle step, and proving it ran](#the-settle-step-and-proving-it-ran)
  - [What the run fixes, and the apply scope of each setting](#what-the-run-fixes-and-the-apply-scope-of-each-setting)
  - [The measurement lock](#the-measurement-lock)
  - [The oracle](#the-oracle)
  - [How a published quantity is declared and scored](#how-a-published-quantity-is-declared-and-scored)
  - [The four mandatory cross-checks](#the-four-mandatory-cross-checks)
  - [The concurrency rules](#the-concurrency-rules)
  - [The reading rules](#the-reading-rules)
  - [Coverage the protocol requires](#coverage-the-protocol-requires)
  - [What the protocol does not cover](#what-the-protocol-does-not-cover)
- [Where It Appears in Source](#where-it-appears-in-source)
  - [Callers and callees the protocol depends on](#callers-and-callees-the-protocol-depends-on)
  - [Shipped tests that cover the same behavior](#shipped-tests-that-cover-the-same-behavior)
- [Related Structures and Functions](#related-structures-and-functions)
- [Interactions with Other Concepts](#interactions-with-other-concepts)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Definition

The mandatory GIN bloat tests are this wiki's shared **measurement protocol** for any claim about how many bytes a PostgreSQL 17 GIN index is wasting, or how many a rebuild would return. The protocol fixes six things and nothing else: the five phases every scored fixture runs, the settle step that must be proven to have completed before a page is read, the `SHARE ROW EXCLUSIVE` measurement lock that makes a census comparable instead of merely describable, the single oracle - a measured `REINDEX INDEX` bracketed by `pg_relation_size(index, 'main')` - the way a published quantity must be declared as a lower bound, an upper bound or a level before it is scored, and the four cross-checks plus the reading rules that decide which numbers may be published as bloat at all. It deliberately does **not** define a fixture corpus: the numbered GIN fixtures, their recipes and their results stay on the question page that runs them, and this page is what that page must state it followed. A GIN waste claim is not "tested" in this wiki until it has been produced under these phases, against this oracle, with a declared kind per published column.

## Why It Exists

Seven properties of the v17 GIN implementation make a plausible GIN waste reading wrong, and each is the reason one protocol rule exists.

**No single instrument answers the question, so every method is a composite.** `pgstattuple` refuses GIN outright: `pgstat_relation` dispatches on `relam` and the GIN arm falls into the "not supported" error ([pgstattuple.c#pgstat_relation-gin](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L260-L296)). `pgstatginindex` accepts a GIN index but reads three fields from block 0 and returns, never touching a second page ([pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577), [pgstatindex.c#GinIndexStat](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L97-L108)). `contrib/amcheck` 1.4 ships B-tree and heap verifiers only, so there is no structural GIN verifier to borrow page counts from ([amcheck.control:3](../../../raw/postgres-17/contrib/amcheck/amcheck.control#L3), [amcheck--1.3--1.4.sql#bt-only](../../../raw/postgres-17/contrib/amcheck/amcheck--1.3--1.4.sql#L13-L25)). A composite instrument has no single refusal to hide behind, which is why the protocol scores the number rather than the tool.

**The entry tree never deletes a tuple.** VACUUM removes deletable TIDs from posting lists and deletes emptied posting-tree pages, but it deletes neither tuples nor pages from the entry tree, and the GIN README states that as the reason entry-tree leaves need no dedicated high key ([README#page-deletion](../../../raw/postgres-17/src/backend/access/gin/README#L389-L396), [README#concurrency-highkey](../../../raw/postgres-17/src/backend/access/gin/README#L309-L314)). An entry tuple for a key that no longer occurs therefore survives every VACUUM and dies only in a rebuild, so any quantity a method calls "payload" is not rebuild-invariant.

**A deleted page is not a reclaimable page until the horizon moves, and GIN never truncates.** `ginDeletePage` sets `GIN_DELETED` and stamps the next transaction id into `pd_prune_xid` ([ginvacuum.c#ginDeletePage-setdeleted](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192), [ginblock.h#GinPageGetDeleteXid](../../../raw/postgres-17/src/include/access/ginblock.h#L132-L138)); `GinPageIsRecyclable` accepts a page only if it is new, or deleted with an invalid delete xid, or deleted with one `GlobalVisCheckRemovableXid` has moved past ([ginvacuum.c#GinPageIsRecyclable](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)). `ginvacuumcleanup` records the recyclable ones in the FSM and then re-reads the relation length rather than shortening it ([ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789), [ginvacuum.c#ginvacuumcleanup-relength](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L794-L802)), and `GinNewBuffer` hands those pages back out inside the same index before extending the file ([ginutil.c#GinNewBuffer](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335)). Dead pages are therefore internal capacity, not returned bytes, which is why the oracle has to be a measured rebuild and not a page census.

**Free space means two different things on the two live page classes.** GIN's own free-space test on a data leaf is `GinDataLeafPageGetFreeSpace`, defined as `PageGetExactFreeSpace`, which is `pd_upper - pd_lower` with no allowance for a line pointer ([ginblock.h#GinDataLeafPageGetFreeSpace](../../../raw/postgres-17/src/include/access/ginblock.h#L287), [bufpage.c#PageGetExactFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L952-L973)). On an entry page the comparable engine test is `entryIsEnoughSpace`, which uses `PageGetFreeSpace` and adds `sizeof(ItemIdData)` to what it needs, because a new tuple also needs a line pointer ([ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)). The same subtraction is insertion capacity on one class and a physical gap on the other, so the protocol forbids publishing their sum as a bound.

**Pending-list pages are deferred work, and flushing them can allocate.** With `fastupdate` on, inserts land in a linked list of `GIN_LIST` pages whose head, tail and counters live in the metapage, and the documentation names the four events that merge them into the main structure ([ginblock.h#GinMetaPageData](../../../raw/postgres-17/src/include/access/ginblock.h#L55-L101), [gin.sgml#GIN-Fast-Update](../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537)). The merge is `ginInsertCleanup`, which drives the accumulated keys through `ginEntryInsert` ([ginfast.c#ginInsertCleanup](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L776-L783), [ginfast.c#flush-entry-insert](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L926-L933)), and a split on that path takes a new page from `GinNewBuffer` ([ginbtree.c#split-newbuffer](../../../raw/postgres-17/src/backend/access/gin/ginbtree.c#L463-L466), [gindatapage.c#dataSplit-newbuffer](../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L1822-L1826)). Settling the index is part of the measurement, and it is not free.

**The metapage's page counts are written by one caller only.** `nTotalPages`, `nEntryPages`, `nDataPages` and `nEntries` are documented in the struct as "accurate as of last VACUUM" ([ginblock.h#planner-stats](../../../raw/postgres-17/src/include/access/ginblock.h#L77-L83)), are written by `ginvacuumcleanup` through `ginUpdateStats` ([ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789), [ginutil.c#ginUpdateStats](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L645-L650)), and `pageinspect` republishes them with the same caveat in a comment ([ginfuncs.c#gin_metapage_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95)). An `ANALYZE`-only call into the GIN callback is a no-op except in an autovacuum worker ([ginvacuum.c#analyze_only](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L729)). A census that trusts those counters without proving a cleanup ran is reading a number from an unknown earlier state.

**A census holds no relation lock between page reads.** `get_raw_page_internal` opens the relation with `AccessShareLock`, copies one page, and closes the relation with that lock before the next call ([rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199)). A multi-page census of a busy index is therefore a sequence of instants, not a snapshot, which is why the protocol has both a lock rule and a bracket rule.

## How It Works

### The five phases of a run

Every scored fixture passes through the same five phases, in order, on one isolated cluster. The phase boundaries are what make a GIN number comparable to a rebuild at all.

| Phase | What happens | Why it is a separate phase |
|---|---|---|
| build | create the table, load it, create the index that is scored, settle it | the as-built state is the only state in which the index is known to be dense |
| baseline | record `pg_relation_size(index, 'main')`, the metapage row, and the page-class census | a before-and-after claim has nothing to compare against otherwise, and the entry-page slack of a fresh build is the level every later reading is judged against |
| churn | run the recipe's own writes, then the settle step | this is the state the method is asked about |
| decide | run the method under test, unmodified, under the measurement lock | the text that is scored must be the text that is published |
| oracle | `REINDEX INDEX`, then read the size again | the only ground truth for "how many bytes would a rebuild return" |

The settle step belongs to the build and churn phases, not to the decide phase: a method may not be handed an index whose pending list and deleted pages were never resolved, because that state is an artifact of the fixture rather than of the index.

### The settle step, and proving it ran

Settling means: the pending list has been merged, and the metapage's page counts describe the file as it now stands. A run must record evidence that it happened, because every failure mode here is silent.

1. **`VACUUM` on the table is the settle command.** GIN registers `ginbulkdelete` and `ginvacuumcleanup` as its `ambulkdelete` and `amvacuumcleanup` callbacks ([ginutil.c#ginhandler-vacuum](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L64-L72)), and `ginvacuumcleanup` is what walks every block from `GIN_ROOT_BLKNO`, records recyclable pages in the FSM and rewrites the metapage counters ([ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789)).
2. **`gin_clean_pending_list()` settles the pending list only.** It opens the *index* with `RowExclusiveLock`, requires ownership, refuses during recovery and on another session's temporary index, and is a logged no-op on an index that is not valid ([ginfast.c#gin_clean_pending_list](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091)). It does not rewrite the metapage's page counts, so it is not a substitute for the `VACUUM`.
3. **A successful `VACUUM` may never have touched the index.** `do_index_cleanup` starts true and is forced false when `INDEX_CLEANUP` is disabled ([vacuumlazy.c#do_index_cleanup-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L397)), the cleanup call is made only when it is still set ([vacuumlazy.c#lazy_cleanup_all_indexes-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066)), and the wraparound failsafe clears it mid-run ([vacuumlazy.c#failsafe-disables-cleanup](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2320-L2326)). The index-vacuum bypass is the one case that does *not* cost the settle step: it turns off index vacuuming and explicitly keeps index cleanup ([vacuumlazy.c#bypass-keeps-cleanup](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1949)). A run must therefore assert the settle step from the index's own state, not from the fact that a `VACUUM` returned.
4. **Expect more than one `VACUUM` before deleted pages count as free.** A page deleted by the first VACUUM carries a delete xid the cluster has not passed yet, so `GinPageIsRecyclable` refuses it and `ginvacuumcleanup` counts it as a data page instead of recording it free ([ginvacuum.c#GinPageIsRecyclable](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829), [ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789)).
5. **On a standby the settle step cannot run at all.** `VACUUM` is classified as not strictly read-only ([utility.c#vacuum-not-read-only](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L295)) and is refused during recovery by the utility gate ([utility.c#recovery-gate](../../../raw/postgres-17/src/backend/tcop/utility.c#L570-L583), [utility.c#PreventCommandDuringRecovery](../../../raw/postgres-17/src/backend/tcop/utility.c#L440-L449)); `gin_clean_pending_list` refuses for itself ([ginfast.c#recovery-refusal](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1037-L1041)). A standby census reads whatever state replay produced, and the protocol does not score a bound against it.

### What the run fixes, and the apply scope of each setting

A run states these settings and their scope, so a reader knows what was held still and what a reproduction has to change.

| Setting | Context | Apply scope | Role in a run |
|---|---|---|---|
| `autovacuum` | `PGC_SIGHUP` ([guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457)) | reload | off for the run, so no background worker settles or churns a fixture between phases |
| `statement_timeout` | `PGC_USERSET` ([guc_tables.c#statement_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)) | session/transaction | bounds the census and the rebuild |
| `lock_timeout` | `PGC_USERSET` ([guc_tables.c#lock_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)) | session/transaction | bounds the wait for the measurement lock |
| `maintenance_work_mem` | `PGC_USERSET` ([guc_tables.c#maintenance_work_mem](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)) | session/transaction | an input to the oracle, not a constant; see [The oracle](#the-oracle) |
| `gin_pending_list_limit` | `PGC_USERSET` ([guc_tables.c#gin_pending_list_limit](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)) | session/transaction | decides whether an insert stream flushes in the foreground mid-fixture ([ginfast.c#foreground-cleanup](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L452-L471)) |
| `fastupdate`, `gin_pending_list_limit` reloptions | `RELOPT_KIND_GIN`, `AccessExclusiveLock` ([reloptions.c#fastupdate](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131), [reloptions.c#gin_pending_list_limit](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347)) | per index, and the `ALTER` that sets them locks the table out | a fixture axis; the per-index form overrides the GUC |

`fastupdate` and `gin_pending_list_limit` are the *only* two GIN reloptions ([ginutil.c#ginoptions](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614)). There is no `fillfactor` for a GIN index, which is why the protocol never compares an index's density against a fixed target; see [The reading rules](#the-reading-rules).

### The measurement lock

When a census has to be compared with anything - an earlier census, a later rebuild, another index - it runs inside one transaction holding `SHARE ROW EXCLUSIVE` on the index's **table**. That mode conflicts with `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE`, itself, `EXCLUSIVE` and `ACCESS EXCLUSIVE`, and is compatible with `ACCESS SHARE` and `ROW SHARE` only ([lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104), [mvcc.sgml#SHARE-ROW-EXCLUSIVE](../../../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1023-L1043)). So it excludes writers, `VACUUM` and `ANALYZE` - both of which take `ShareUpdateExclusiveLock` ([vacuum.c#vacuum-lockmode](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2056), [analyze.c#analyze-lockmode](../../../raw/postgres-17/src/backend/commands/analyze.c#L135-L145)), `VACUUM FULL`'s `AccessExclusiveLock` ([vacuum.c#vacuum-lockmode](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2056)), and both `REINDEX` forms - while leaving ordinary readers alone.

Three rules follow from the source, and a run may not weaken them:

- **`SHARE` is one step weaker and not enough.** A plain `REINDEX INDEX` takes `ShareLock` on the table, which is self-compatible, so it would proceed and swap the index's storage under a running census ([indexcmds.c#RangeVarCallbackForReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2857-L2872), [indexcmds.c#ReindexIndex-locks-and-dispatch](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2824-L2849)).
- **The lock cannot be taken on the index, and one writer escapes it.** `LOCK TABLE` refuses anything but a plain table, a partitioned table or a view ([lockcmds.c#RangeVarCallbackForLockTable](../../../raw/postgres-17/src/backend/commands/lockcmds.c#L70-L107)), while `gin_clean_pending_list` locks only the index ([ginfast.c#gin_clean_pending_list](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091)). The protocol closes that hole operationally - the index's owner does not flush the pending list during a measurement - and a run states that it did so.
- **The privilege is not `SELECT`.** `LockTableAclCheck` accepts `MAINTAIN`, `UPDATE`, `DELETE` or `TRUNCATE` for every mode above `ROW EXCLUSIVE`, and adds `SELECT` only at `ACCESS SHARE` ([lockcmds.c#LockTableAclCheck](../../../raw/postgres-17/src/backend/commands/lockcmds.c#L279-L299)). A raw-page census already needs superuser ([rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199)), so this binds only a method built on `pgstatginindex`, whose execute privilege is granted to `pg_stat_scan_tables` ([pgstattuple--1.4--1.5.sql#pgstatginindex-grant](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57)).

The lock is a cost, not a default: it blocks every writer on the table for the duration of the census, so a run reports the interval it held and pairs it with `lock_timeout`.

### The oracle

The oracle is a measured `REINDEX INDEX` on the settled fixture, with `pg_relation_size(index, 'main')` read immediately before and after. `reindex_index` suppresses the target index, gives it a new relfilenode and calls `index_build`, so the after-reading is a new physical file rather than a compacted old one ([index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)). `pg_relation_size` with an explicit fork sums the segments of that one fork ([dbsize.c#pg_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371), [dbsize.c#calculate_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L302-L343), [func.sgml#pg_relation_size](../../../raw/postgres-17/doc/src/sgml/func.sgml#L29627-L29643)), which is the right denominator because a page census counts main-fork blocks and nothing else.

Two constraints on using it:

- **The oracle measures returned filesystem bytes, not waste.** The engine reclaims a GIN page into the index's own FSM and never truncates the relation, so a page census and the oracle answer different questions by construction ([ginvacuum.c#ginvacuumcleanup-relength](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L794-L802), [ginutil.c#GinNewBuffer](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335)).
- **A run must report the `maintenance_work_mem` the rebuild ran at.** The GIN build flushes its accumulator to the index whenever allocated memory reaches `maintenance_work_mem` ([gininsert.c#build-flush](../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L286-L292)), so the rebuilt size is a function of the budget as well as of the data. "What a rebuild would return" is not a property of the index alone, and an unlabelled oracle number is not reproducible.

### How a published quantity is declared and scored

Before the run, every column the method publishes is declared as exactly one of three kinds. The declaration is filed with the fixture plan and may not be rewritten afterwards.

| Declared kind | What it claims against the oracle | Verdicts |
|---|---|---|
| lower bound | the quantity never exceeds the bytes a rebuild returns | `HELD`, `VIOLATED` |
| upper bound | the quantity is never less than the bytes a rebuild returns | `HELD`, `VIOLATED` |
| level | the quantity has no relation to what a rebuild returns | `DECLARED` - never scored as a bound, and never published as reclaimable space |

Two columns are mandatory for a run to be readable: `truth_pct`, the oracle's own fraction of the churned file, computed from the two size readings and nothing else; and `declared_kind` per published quantity, filed before the run. A `VIOLATED` verdict is corrected in the method or moved to a `level`, not annotated away.

A method that also makes a **decision** - rebuild, or leave alone - declares its own threshold, because the protocol fixes the oracle and not the threshold. The decision is then scored `PASS`, `FALSE POSITIVE` (rebuilt, and the rebuild returned less than the method's own threshold) or `FALSE NEGATIVE` (not rebuilt, and a rebuild would have returned more). A run reports the threshold beside the score; a threshold chosen after the results is not a scored run.

### The four mandatory cross-checks

No GIN number is published from a single reading. All four checks run, and a run records which ones agreed.

| Check | What it compares | Direction and failure mode |
|---|---|---|
| FSM free pages | the census's recyclable-page count against the FSM's count of entirely-free pages, read per block by `pg_freespace` ([pg_freespacemap.c#pg_freespace](../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L50)) | one-directional. A page can be deleted and not yet recorded free, because `ginvacuumcleanup` records only the recyclable ones ([ginvacuum.c#pages_free](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L766-L794)). The index FSM tracks only free-versus-in-use, recording a free page as `BLCKSZ - 1` ([indexfsm.c#index-fsm-notes](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L20), [indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)), and the value read back is the category floor, so a free index page reads `MaxFSMRequestSize` rather than `BLCKSZ - 1` ([freespace.c#FSM_CATEGORIES](../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L37-L66), [freespace.c#fsm_space_cat_to_avail](../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L427-L435), [htup_details.h#MaxHeapTupleSize](../../../raw/postgres-17/src/include/access/htup_details.h#L563)). Derive that constant from the running build; never type it. |
| size bracket | `pg_relation_size(index, 'main')` re-read *after* the census, against the block count the census scanned | catches a file that grew or shrank mid-scan, which is exactly what the per-call lock release permits ([rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199)) |
| `VACUUM VERBOSE` | the fourth number of the index line, which is `pages_free` ([vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L720-L731)), against the census's recyclable count | agrees only for the VACUUM that produced the state; `pages_free` is `ginvacuumcleanup`'s own `totFreePages` ([ginvacuum.c#pages_free](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L766-L794)) |
| metapage page-type counts | `nEntryPages` and `nDataPages` against the SQL census of live pages | two caveats fall out of the code and must be applied, not discovered: a deleted page that is not yet recyclable is counted as a **data** page, because `GinPageIsData` is tested after `GinPageIsRecyclable`; and a live `list` page is counted in neither bucket, so `nTotalPages` can exceed `1 + nEntryPages + nDataPages` ([ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789)) |

### The concurrency rules

- **Flag what the progress views can see.** A census records whether `pg_stat_progress_vacuum`, `pg_stat_progress_analyze` or `pg_stat_progress_create_index` showed work on the relation at either end of its scan ([system_views.sql#pg_stat_progress_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1209-L1227), [system_views.sql#pg_stat_progress_analyze](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1188-L1207), [system_views.sql#pg_stat_progress_create_index](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1256-L1289)). The `create_index` view names `REINDEX` and `REINDEX CONCURRENTLY` explicitly, so a rebuild racing the census is visible by command.
- **The flag is not a guarantee.** A maintenance command that starts and finishes between the two progress reads leaves no trace in either, so a flagged census is unreliable and an unflagged one is only unrefuted. Where a comparison must be exact, take the lock instead of reading the flags; see [The measurement lock](#the-measurement-lock).
- **Read a busy index twice.** Two censuses that disagree on an unchanging file size are evidence of a mixed-instant reading, and that evidence exists because no relation lock is held across the page reads ([rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199)).

### The reading rules

These decide which numbers may be published, and as what.

- **Report the page classes separately.** Whole-page waste, live-page slack and pending-list bytes have three different relations to a rebuild; a single combined percentage is page accounting and must be declared a `level` unless the run scores it as a bound and it holds.
- **Entry-page slack is insertion capacity, not a defect.** The engine's own entry-page test charges a line pointer for the next tuple ([ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)), so a freshly built index carries a large, healthy entry slack.
- **Compare an index to its own history or to a rebuilt twin, never to another opclass.** GIN has no `fillfactor` reloption ([ginutil.c#ginoptions](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614)), so the engine defines no target density for a GIN build, and a cross-opclass density comparison has no fixed point.
- **Pending bytes are a `fastupdate` tuning signal, not waste.** The remedies are `VACUUM`, `gin_clean_pending_list()` or `gin_pending_list_limit` ([gin.sgml#GIN-Fast-Update](../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537)), and the flush itself can allocate ([ginfast.c#flush-entry-insert](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L926-L933), [ginbtree.c#split-newbuffer](../../../raw/postgres-17/src/backend/access/gin/ginbtree.c#L463-L466)).
- **Dead pages that will not go away may be waiting on a transaction, not on `VACUUM`.** The horizon test is in `GinPageIsRecyclable` ([ginvacuum.c#GinPageIsRecyclable](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)), so a run that reports stuck pages also reports whether a snapshot was held.
- **Suppress every slack-derived field when the format is not version 2.** `pd_lower` on a pre-9.4 posting-tree page cannot be trusted, and the struct says so ([ginblock.h#pd_lower-untrusted](../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309), [ginblock.h#GIN_CURRENT_VERSION](../../../raw/postgres-17/src/include/access/ginblock.h#L86-L103)). Publish the page counts, withhold the slack and everything computed from it.
- **Suppress slack-derived fields when any page could not be decoded or classified.** `gin_page_opaque_info` raises on a foreign special-area size and prints unrecognized flag bits as hex rather than guessing a class ([ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L172)); `gin_leafpage_items` refuses any page whose flags are not exactly `GIN_DATA | GIN_LEAF | GIN_COMPRESSED` ([ginfuncs.c#gin_leafpage_items-flags](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L211-L226)); and `get_page_from_raw` raises on a page of the wrong size ([rawpage.c#get_page_from_raw](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L214-L234)). One such page can cost a multi-index statement every row it would have returned, so a statement scored here reports a per-page status instead of failing.
- **Handle the new page in both readers.** `gin_metapage_info`, `gin_page_opaque_info` and `gin_leafpage_items` return NULL for a page `PageIsNew` accepts ([ginfuncs.c#gin_metapage_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95), [ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L172), [bufpage.h#PageIsNew](../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234)), while `page_header` has no such test and reports the zeros it finds ([rawpage.c#page_header](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L245-L266)). An all-zero page is recyclable whole-page waste, not a zero-slack entry page.

### Coverage the protocol requires

A conforming run exercises each behavior below at least once, and says which of its own fixtures did. The fixture names, recipes and SQL are page-local to the method being scored; this table is the behavior list, not a corpus.

| Behavior a run must reach | Why the protocol requires it |
|---|---|
| keys that no longer occur after churn | the entry tree never deletes a tuple, so this is where a payload model fails ([README#page-deletion](../../../raw/postgres-17/src/backend/access/gin/README#L389-L396)) |
| emptied posting-tree pages | `ginDeletePage` plus recycling is the only source of whole-page waste besides a new page ([ginvacuum.c#ginDeletePage-setdeleted](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192)) |
| half-empty posting-tree leaves with nothing deletable | `PageGetExactFreeSpace` slack on data pages is the signal a rebuild really repays ([ginblock.h#GinDataLeafPageGetFreeSpace](../../../raw/postgres-17/src/include/access/ginblock.h#L287)) |
| a populated pending list, and the same index after a flush | deferred work is not waste, and the flush can allocate ([ginfast.c#ginInsertCleanup](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L776-L783)) |
| an untouched index and an empty index | fresh entry-page slack must not read as reclaimable ([ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482)) |
| a snapshot held across the settling `VACUUM` | separates "deleted" from "recyclable" ([ginvacuum.c#GinPageIsRecyclable](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)) |
| a `VACUUM` whose index cleanup did not run | a command that succeeded and changed nothing in the index ([vacuumlazy.c#do_index_cleanup-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L397)) |
| an undecodable or unclassifiable page, and an all-zero page | the suppression rules, and the difference between a NULL row and a zero reading ([ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L172)) |
| a concurrent `VACUUM`, a concurrent rebuild and a writer stream | the mixed-instant reading the per-call lock release allows ([rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199)) |
| more than one operator class | no GIN fillfactor exists, so density has no cross-opclass reference ([ginutil.c#ginoptions](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614)) |
| one rebuild at more than one `maintenance_work_mem` | the oracle is budget-dependent ([gininsert.c#build-flush](../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L286-L292)) |

### What the protocol does not cover

Named limits, so a consumer page does not claim more than the protocol gives.

- **It scores the number, not the instrument.** Whether a method reads raw pages, `pgstatginindex`, the FSM or the catalogs is out of scope; the declared kind and the oracle are what is checked. A method that cannot run on a required behavior records that it skipped it.
- **There is no structural verifier to corroborate a page census.** `contrib/amcheck` has no GIN entry point, so a census's page classification is checked only against the metapage, the FSM and `VACUUM`'s own output ([amcheck--1.3--1.4.sql#bt-only](../../../raw/postgres-17/contrib/amcheck/amcheck--1.3--1.4.sql#L13-L25)).
- **One block size and one platform.** Every page constant derives from `BLCKSZ` and `MAXALIGN`; the protocol fixes no second build.
- **The pending-list hole stays open.** No lock available from SQL excludes a direct `gin_clean_pending_list()` by the index's owner ([lockcmds.c#RangeVarCallbackForLockTable](../../../raw/postgres-17/src/backend/commands/lockcmds.c#L70-L107), [ginfast.c#gin_clean_pending_list](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091)). The protocol handles it by operational rule only.
- **A standby is not a scored environment.** The settle step cannot run there ([utility.c#recovery-gate](../../../raw/postgres-17/src/backend/tcop/utility.c#L570-L583), [ginfast.c#recovery-refusal](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1037-L1041)).
- **The engine's own test suites validate the build, not the method.** The shipped GIN and `pageinspect` tests assert function behavior on tiny fixtures; none of them reads a bloat estimate ([gin.sql#pageinspect-gin](../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L1-L19)).

## Where It Appears in Source

The protocol has no code in the PostgreSQL tree. What it has in the tree is the behavior each rule targets, and these are the files a reviewer reads to judge a run.

| Area | Files | What the protocol takes from it |
|---|---|---|
| Page reclamation and the metapage census | `src/backend/access/gin/ginvacuum.c`, `src/backend/access/gin/ginutil.c` | `ginvacuumcleanup`'s every-block walk, `GinPageIsRecyclable`, `ginDeletePage`, `ginUpdateStats`, `GinNewBuffer`, and the absence of any truncation ([ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789)) |
| Page layout and flags | `src/include/access/ginblock.h`, `src/include/storage/bufpage.h`, `src/backend/storage/page/bufpage.c` | the flag bits, the metapage struct and its "as of last VACUUM" caveat, the delete-xid macros, `PageIsNew`, and the two free-space functions ([ginblock.h#GinMetaPageData](../../../raw/postgres-17/src/include/access/ginblock.h#L55-L101)) |
| Entry and data page geometry | `src/backend/access/gin/ginentrypage.c`, `src/backend/access/gin/gindatapage.c`, `src/backend/access/gin/ginbtree.c` | which page class charges a line pointer, and where a split allocates ([ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482)) |
| The pending list | `src/backend/access/gin/ginfast.c`, `doc/src/sgml/gin.sgml` | `ginInsertCleanup`, the foreground cleanup threshold, `shiftList`'s flag overwrite, and `gin_clean_pending_list`'s lock, ownership and recovery rules ([ginfast.c#gin_clean_pending_list](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091)) |
| VACUUM's index work | `src/backend/access/heap/vacuumlazy.c`, `src/backend/commands/vacuum.c` | the three ways index cleanup does not run, the bypass that keeps it, the `VERBOSE` index line, and the table lock a settle step takes ([vacuumlazy.c#do_index_cleanup-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L397)) |
| The free space map | `src/backend/storage/freespace/indexfsm.c`, `src/backend/storage/freespace/freespace.c`, `doc/src/sgml/pgfreespacemap.sgml` | what an index FSM stores, and what value a free page reads back ([indexfsm.c#index-fsm-notes](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L20)) |
| Locking | `src/backend/storage/lmgr/lock.c`, `src/backend/commands/lockcmds.c`, `src/backend/commands/indexcmds.c`, `doc/src/sgml/mvcc.sgml` | the conflict table, `LOCK TABLE`'s relkind and privilege rules, and the table lock each `REINDEX` form takes ([lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)) |
| The oracle | `src/backend/catalog/index.c`, `src/backend/utils/adt/dbsize.c`, `src/backend/access/gin/gininsert.c` | the rebuild's new relfilenode, the one-fork size function, and the build's memory-driven flush ([index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)) |
| Measurement interface | `contrib/pageinspect/rawpage.c`, `contrib/pageinspect/ginfuncs.c`, `contrib/pgstattuple/pgstatindex.c`, `contrib/pg_freespacemap/pg_freespacemap.c` | the superuser gate, the per-call lock release, the three GIN readers and their refusals, and what `pgstatginindex` can and cannot see ([ginfuncs.c#gin_metapage_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95)) |
| Observability of a racing command | `src/backend/catalog/system_views.sql` | the three progress views a census consults ([system_views.sql#pg_stat_progress_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1209-L1227)) |

### Callers and callees the protocol depends on

A citation that lands on a function's signature resolves its name; it does not support a claim about what the function does. These are the boundaries the protocol's rules actually cross.

| Protocol rule | Boundary |
|---|---|
| the settle step's index work | `vacuum_rel` -> `lazy_vacuum_rel` -> `lazy_cleanup_all_indexes` -> `ginvacuumcleanup` -> `RecordFreeIndexPage` and `ginUpdateStats`, reached only while `do_index_cleanup` is set ([vacuumlazy.c#lazy_cleanup_all_indexes-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066), [ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789), [ginutil.c#ginUpdateStats](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L645-L650)) |
| the pending-list settle | `gin_clean_pending_list` -> `ginInsertCleanup` -> `processPendingPage` -> `ginEntryInsert`, and `shiftList` for the freed pages ([ginfast.c#gin_clean_pending_list](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091), [ginfast.c#flush-entry-insert](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L926-L933), [ginfast.c#shiftList-flags](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L630-L635)) |
| a flush that grows the file | `ginEntryInsert` -> the entry/data btree insert -> a split -> `GinNewBuffer`, which extends only after the FSM is empty ([ginbtree.c#split-newbuffer](../../../raw/postgres-17/src/backend/access/gin/ginbtree.c#L463-L466), [gindatapage.c#dataSplit-newbuffer](../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L1822-L1826), [ginutil.c#GinNewBuffer](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335)) |
| the analyze no-op | `analyze_rel` -> the `amvacuumcleanup` callback with `analyze_only` set, which returns without a census unless the caller is an autovacuum worker ([ginvacuum.c#analyze_only](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L729)) |
| the oracle | `ReindexIndex` -> `reindex_index` -> `RelationSetNewRelfilenumber` then `index_build`, under `ShareLock` on the table ([indexcmds.c#ReindexIndex-locks-and-dispatch](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2824-L2849), [indexcmds.c#RangeVarCallbackForReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2857-L2872), [index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)) |
| one census page read | `get_raw_page` -> `get_raw_page_internal`, which opens, copies and closes per call, then `get_page_from_raw` in each reader ([rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199), [rawpage.c#get_page_from_raw](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L214-L234)) |
| the FSM cross-check | `pg_freespace` -> `GetRecordedFreeSpace` -> `fsm_space_cat_to_avail`, against `RecordFreeIndexPage` -> `RecordPageWithFreeSpace` -> `fsm_space_avail_to_cat` on the write side ([pg_freespacemap.c#pg_freespace](../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L50), [freespace.c#fsm_space_cat_to_avail](../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L427-L435), [indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)) |
| the standby refusal | `standard_ProcessUtility` -> `ClassifyUtilityCommandAsReadOnly` -> `PreventCommandDuringRecovery` for `VACUUM`, and `RecoveryInProgress` inside `gin_clean_pending_list` ([utility.c#vacuum-not-read-only](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L295), [utility.c#recovery-gate](../../../raw/postgres-17/src/backend/tcop/utility.c#L570-L583), [ginfast.c#recovery-refusal](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1037-L1041)) |

### Shipped tests that cover the same behavior

The protocol is not in the tree, and neither is anything that scores a GIN bloat number. Naming the absences matters as much as naming the coverage.

| Behavior | Shipped coverage |
|---|---|
| the three `pageinspect` GIN readers on a live index | [gin.sql#pageinspect-gin](../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L1-L19) |
| their refusals: wrong page size, foreign special-area size, and an all-zero page | [gin.sql#pageinspect-failures](../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L21-L39) |
| `pgstatginindex` output, and `pgstatindex` refusing a GIN index | [pgstattuple.sql#pgstatginindex](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L47-L63) |
| a pending list flushed by `gin_clean_pending_list()`, by `VACUUM`, and `fastupdate` turned off afterwards | [gin.sql#pending-list](../../../raw/postgres-17/src/test/regress/sql/gin.sql#L8-L30) |
| a structural verifier for the index being measured | **no coverage for GIN.** `contrib/amcheck` verifies B-tree indexes and heaps only ([check_btree.sql#bt_index_check](../../../raw/postgres-17/contrib/amcheck/sql/check_btree.sql#L25-L26), [amcheck--1.3--1.4.sql#bt-only](../../../raw/postgres-17/contrib/amcheck/amcheck--1.3--1.4.sql#L13-L25)) |
| a page census cross-checked against the metapage, the FSM or a rebuild | **no coverage.** No shipped test compares a GIN page classification with `nEntryPages`/`nDataPages`, with `pg_freespace`, or with a `REINDEX` |

## Related Structures and Functions

| Symbol | Role in the protocol |
|---|---|
| [ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789) | the settle step's census: what becomes FSM-free, what is counted as entry or data, and what the metapage ends up saying |
| [ginvacuum.c#GinPageIsRecyclable](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829) | the three-way test that separates a deleted page from a reusable one |
| [ginvacuum.c#ginDeletePage-setdeleted](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192) | where the `GIN_DELETED` flag and the delete xid are written |
| [ginutil.c#GinNewBuffer](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335) | why reclaimed pages stay inside the index |
| [ginutil.c#ginUpdateStats](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L645-L650) | the only writer of the metapage's page counts |
| [ginblock.h#GinMetaPageData](../../../raw/postgres-17/src/include/access/ginblock.h#L55-L101) | every field a census reads from block 0, with the version history that gates the slack columns |
| [ginblock.h#GinDataLeafPageGetFreeSpace](../../../raw/postgres-17/src/include/access/ginblock.h#L287) | GIN's own definition of free space on a data leaf |
| [ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482) | the entry-page counterpart, which charges a line pointer |
| [ginfast.c#ginInsertCleanup](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L776-L783) | the pending-list merge every settle step depends on |
| [ginfast.c#gin_clean_pending_list](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091) | the index-only settle command, its `RowExclusiveLock`, and the measurement hole it leaves |
| [ginutil.c#ginoptions](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614) | the two GIN reloptions, and the absence of a fillfactor |
| [vacuumlazy.c#do_index_cleanup-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L397) | the flag whose state decides whether a `VACUUM` settles the index at all |
| [vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L720-L731) | the `VACUUM VERBOSE` cross-check's four fields |
| [lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104) | the conflict table behind the measurement lock |
| [lockcmds.c#LockTableAclCheck](../../../raw/postgres-17/src/backend/commands/lockcmds.c#L279-L299) | the privilege the measurement lock needs, which is not `SELECT` |
| [index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789) | the oracle's rebuild |
| [dbsize.c#pg_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371) | the size reading on both sides of the oracle |
| [gininsert.c#build-flush](../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L286-L292) | why the oracle depends on `maintenance_work_mem` |
| [rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199) | the superuser gate and the per-call lock release that make a census non-atomic |
| [ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L172) | page classification, the NULL row for a new page, and the hex fallback for an unknown flag |
| [pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577) | the metapage-only reading, and its three refusals |

## Interactions with Other Concepts

- **VACUUM.** The protocol depends on `VACUUM` for the only state transition it cannot make itself: dead entries into recyclable pages and a refreshed metapage. It makes three boundaries explicit - it never reproduces the autovacuum launcher's verdicts, it does not assume a `VACUUM` that returned actually cleaned the index, and it treats more than one `VACUUM` as normal rather than as a fixture quirk.
- **The free space map.** The FSM is a cross-check, not a source of truth: for an index it records only free-versus-in-use, and the value read back is a category floor ([indexfsm.c#index-fsm-notes](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L20), [freespace.c#fsm_space_cat_to_avail](../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L427-L435)). Whether a page a rebuild would drop is in the FSM is a separate question from whether a rebuild would drop it.
- **The GIN pending list and `fastupdate`.** The protocol requires the pending list to be settled and reported separately, and treats a flush as an action with a cost. How to tune `fastupdate` is not a bloat question.
- **REINDEX.** The oracle is the blocking `REINDEX INDEX` form. Lock levels, the concurrent form's behavior and failure outcomes belong to the method or to a `REINDEX` question page, not here.
- **B-tree bloat measurement.** The B-tree suite is a different document with a different contract: it defines a fixture corpus and decision bands, where this page defines a protocol and leaves the corpus to the consumer. See [Mandatory B-Tree Bloat Tests](mandatory-btree-bloat-tests.md) for navigation; no claim on this page rests on it.
- **The measurement interface.** `pageinspect`, `pgstattuple` and `pg_freespacemap` are inputs whose refusals shape what a run can say ([pageinspect.control:3](../../../raw/postgres-17/contrib/pageinspect/pageinspect.control#L3), [pgstattuple.control:3](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L3), [pg_freespacemap.control:3](../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.control#L3)). The protocol scores the number a method publishes, not the extension it came from.

## Context Reviewed

- GIN vacuum end to end: `ginvacuumcleanup`'s `analyze_only` early return, its `ginInsertCleanup` call when `ginbulkdelete` did not run, the every-block loop from `GIN_ROOT_BLKNO`, the recyclable/data/entry classification order, `nTotalPages` assignment, `ginUpdateStats`, `IndexFreeSpaceMapVacuum`, `pages_free`, and the closing re-read of the relation length; `ginDeletePage`'s flag and delete-xid writes; and `GinPageIsRecyclable`'s new/invalid-xid/horizon arms.
- The GIN page format: the opaque struct and all eight flag bits, the metapage struct including the "accurate as of last VACUUM" comment and the version-0/1/2 history, `GIN_CURRENT_VERSION`, the delete-xid macros, `GinDataLeafPageGetFreeSpace`, the note on why `pd_lower` is untrustworthy on pre-9.4 data pages, and the non-leaf free-space macro.
- Entry and data page geometry: `entryIsEnoughSpace`'s line-pointer accounting against `PageGetFreeSpace`, `PageGetExactFreeSpace`, `PageIsNew`, `SizeOfPageHeaderData`, and the `PageHeaderData` fields a census reads.
- The pending list: the metapage head/tail/counters, `ginInsertCleanup`'s signature and its `processPendingPage`/`ginEntryInsert` loop, `shiftList`'s replacement of the flags word with `GIN_DELETED` and its metapage counter arithmetic, the foreground cleanup threshold against `GinGetPendingListCleanupSize`, and `gin_clean_pending_list`'s lock mode, recovery refusal, GIN check, other-session-temp refusal, ownership check and `!indisvalid` DEBUG1 no-op.
- Page allocation: `GinNewBuffer`'s FSM-then-extend loop and its `GinPageIsRecyclable` recheck, plus its four call sites - the build's meta and root pages, the btree split, the pending-list page and the posting-tree split.
- The GIN AM registration, and the two GIN reloptions with their `RELOPT_KIND_GIN` entries and `AccessExclusiveLock` level.
- VACUUM's index path: `do_index_cleanup`'s initialization, the `INDEX_CLEANUP` disable, the failsafe's clearing of it, the bypass conditions and the comment stating that the bypass keeps index cleanup, the `VERBOSE` index line's four fields, the `update_relstats_all_indexes` gate, and the `FULL`-versus-concurrent table lock mode; `ANALYZE`'s `ShareUpdateExclusiveLock`.
- The index FSM: what it stores, `RecordFreeIndexPage`'s `BLCKSZ - 1`, `GetFreeIndexPage`'s `BLCKSZ / 2` request, the 256 categories, `MaxFSMRequestSize` as `MaxHeapTupleSize`, `fsm_space_avail_to_cat`'s rounding down and the 255 reservation, `fsm_space_cat_to_avail`, `GetRecordedFreeSpace`, `pg_freespace`, and the documentation's statement that index FSM values mean only in-use versus empty.
- Locking: the full conflict table, the documentation of `SHARE` and `SHARE ROW EXCLUSIVE`, `RangeVarCallbackForLockTable`'s relkind restriction, `LockTableAclCheck`'s privilege mask, and the table lock each `REINDEX` form takes through `RangeVarCallbackForReindexIndex` and `ReindexIndex`'s dispatch - checked against `DefineIndex`'s own `ShareLock`, which is a different call site.
- The oracle: `reindex_index`'s suppress / new-relfilenode / `index_build` sequence, `pg_relation_size`'s per-fork behavior and `calculate_relation_size`'s segment walk, the documentation of the `fork` argument, and the GIN build's `maintenance_work_mem` flush.
- The measurement interface: `get_raw_page_internal`'s superuser gate, other-session-temp refusal, range check and per-call open/close; `get_page_from_raw`'s page-size error; `page_header`'s lack of a new-page test; and all three GIN `pageinspect` readers including their NULL returns, special-size errors, `GIN_META` check, unknown-flag hex fallback, and `gin_leafpage_items`' exact-flag requirement.
- `pgstattuple`'s GIN rejection in `pgstat_relation`, `pgstatginindex_internal`'s three fields and three refusals, the `GinIndexStat` struct, and the 1.5 `pg_stat_scan_tables` grant.
- Recovery gating: the read-only classification of `VACUUM`, `REINDEX` and `CLUSTER`, the utility-level gate, and `PreventCommandDuringRecovery`'s message.
- The three progress views a census consults, and the `CREATE INDEX` view's command labels for both `REINDEX` forms.
- The GUCs a run fixes, with their contexts: `autovacuum`, `statement_timeout`, `lock_timeout`, `maintenance_work_mem` and `gin_pending_list_limit`.
- The shipped tests that touch the same behavior - `contrib/pageinspect/sql/gin.sql`, `contrib/pgstattuple/sql/pgstattuple.sql`, `src/test/regress/sql/gin.sql` - and `contrib/amcheck`'s function inventory across its installation scripts, which is where the absence of a GIN verifier is visible.
- The GIN README's page-deletion and concurrency sections, and the `gin.sgml` fast-update section.

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstattuple` refuses a GIN index outright | [pgstattuple.c#pgstat_relation-gin](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L260-L296) |
| `pgstatginindex` reads three metapage fields and no second page | [pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577), [pgstatindex.c#GinIndexStat](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L97-L108) |
| `pgstatginindex` is executable by `pg_stat_scan_tables` and not by `PUBLIC` | [pgstattuple--1.4--1.5.sql#pgstatginindex-grant](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57) |
| v17 contrib ships no GIN verifier | [amcheck.control:3](../../../raw/postgres-17/contrib/amcheck/amcheck.control#L3), [amcheck--1.3--1.4.sql#bt-only](../../../raw/postgres-17/contrib/amcheck/amcheck--1.3--1.4.sql#L13-L25) |
| VACUUM never deletes a tuple or a page from the entry tree | [README#page-deletion](../../../raw/postgres-17/src/backend/access/gin/README#L389-L396), [README#concurrency-highkey](../../../raw/postgres-17/src/backend/access/gin/README#L309-L314) |
| A deleted page carries the next transaction id and becomes reusable only past the horizon | [ginvacuum.c#ginDeletePage-setdeleted](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192), [ginvacuum.c#GinPageIsRecyclable](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829), [ginblock.h#GinPageGetDeleteXid](../../../raw/postgres-17/src/include/access/ginblock.h#L132-L138) |
| `ginvacuumcleanup` records free pages, counts the rest, writes the metapage and does not truncate | [ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789), [ginvacuum.c#ginvacuumcleanup-relength](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L794-L802), [ginutil.c#ginUpdateStats](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L645-L650) |
| A deleted-but-not-recyclable page is counted as a data page, and a `list` page in neither bucket | [ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789) |
| Reclaimed pages are reused inside the index before the file is extended | [ginutil.c#GinNewBuffer](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335), [indexfsm.c#GetFreeIndexPage](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L32-L46) |
| The metapage page counts are documented as accurate only as of the last VACUUM | [ginblock.h#planner-stats](../../../raw/postgres-17/src/include/access/ginblock.h#L77-L83), [ginfuncs.c#gin_metapage_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95) |
| An `analyze_only` GIN cleanup call is a no-op outside an autovacuum worker | [ginvacuum.c#analyze_only](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L729) |
| Index cleanup can be disabled up front or by the failsafe, and the bypass keeps it | [vacuumlazy.c#do_index_cleanup-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L397), [vacuumlazy.c#lazy_cleanup_all_indexes-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066), [vacuumlazy.c#failsafe-disables-cleanup](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2320-L2326), [vacuumlazy.c#bypass-keeps-cleanup](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1949) |
| GIN's vacuum callbacks are `ginbulkdelete` and `ginvacuumcleanup` | [ginutil.c#ginhandler-vacuum](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L64-L72) |
| Data-leaf free space is `pd_upper - pd_lower`, while an entry page also needs a line pointer | [ginblock.h#GinDataLeafPageGetFreeSpace](../../../raw/postgres-17/src/include/access/ginblock.h#L287), [bufpage.c#PageGetExactFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L952-L973), [ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923) |
| `pd_lower` is untrustworthy on pre-9.4 data pages, and version 2 is current | [ginblock.h#pd_lower-untrusted](../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309), [ginblock.h#GIN_CURRENT_VERSION](../../../raw/postgres-17/src/include/access/ginblock.h#L86-L103) |
| A pending-list flush drives keys through `ginEntryInsert`, and a split allocates a page | [ginfast.c#ginInsertCleanup](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L776-L783), [ginfast.c#flush-entry-insert](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L926-L933), [ginbtree.c#split-newbuffer](../../../raw/postgres-17/src/backend/access/gin/ginbtree.c#L463-L466), [gindatapage.c#dataSplit-newbuffer](../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L1822-L1826) |
| A pending-list shift overwrites the flags word with `GIN_DELETED` | [ginfast.c#shiftList-flags](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L630-L635) |
| An insert stream flushes in the foreground once the pending list passes its limit | [ginfast.c#foreground-cleanup](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L452-L471), [gin.sgml#GIN-Fast-Update](../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537) |
| `gin_clean_pending_list` locks only the index, needs ownership, and refuses in recovery | [ginfast.c#gin_clean_pending_list](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091), [ginfast.c#recovery-refusal](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1037-L1041) |
| `LOCK TABLE` cannot name an index | [lockcmds.c#RangeVarCallbackForLockTable](../../../raw/postgres-17/src/backend/commands/lockcmds.c#L70-L107) |
| `SHARE ROW EXCLUSIVE` excludes writers, VACUUM, ANALYZE and both REINDEX forms, and `SHARE` does not exclude a plain REINDEX | [lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104), [mvcc.sgml#SHARE-ROW-EXCLUSIVE](../../../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1023-L1043), [indexcmds.c#RangeVarCallbackForReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2857-L2872), [vacuum.c#vacuum-lockmode](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2056), [analyze.c#analyze-lockmode](../../../raw/postgres-17/src/backend/commands/analyze.c#L135-L145) |
| The measurement lock's privilege is not `SELECT` | [lockcmds.c#LockTableAclCheck](../../../raw/postgres-17/src/backend/commands/lockcmds.c#L279-L299) |
| A census holds no relation lock between page reads, and needs superuser | [rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199) |
| The oracle's rebuild is a new physical file | [index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789), [indexcmds.c#ReindexIndex-locks-and-dispatch](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2824-L2849) |
| `pg_relation_size` with a fork argument measures one fork | [dbsize.c#pg_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371), [dbsize.c#calculate_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L302-L343), [func.sgml#pg_relation_size](../../../raw/postgres-17/doc/src/sgml/func.sgml#L29627-L29643) |
| The rebuilt size depends on `maintenance_work_mem` | [gininsert.c#build-flush](../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L286-L292), [guc_tables.c#maintenance_work_mem](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474) |
| An index FSM stores only free-versus-in-use, and a free page reads back `MaxFSMRequestSize` | [indexfsm.c#index-fsm-notes](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L20), [indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), [freespace.c#FSM_CATEGORIES](../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L37-L66), [freespace.c#fsm_space_cat_to_avail](../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L427-L435), [htup_details.h#MaxHeapTupleSize](../../../raw/postgres-17/src/include/access/htup_details.h#L563), [pgfreespacemap.sgml#index-pages](../../../raw/postgres-17/doc/src/sgml/pgfreespacemap.sgml#L61-L71) |
| The fourth field of the `VACUUM VERBOSE` index line is `pages_free` | [vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L720-L731), [ginvacuum.c#pages_free](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L766-L794) |
| A racing VACUUM, ANALYZE or rebuild is visible in the progress views | [system_views.sql#pg_stat_progress_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1209-L1227), [system_views.sql#pg_stat_progress_analyze](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1188-L1207), [system_views.sql#pg_stat_progress_create_index](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1256-L1289) |
| A GIN index has no fillfactor, so there is no engine-defined target density | [ginutil.c#ginoptions](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614), [reloptions.c#fastupdate](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131), [reloptions.c#gin_pending_list_limit](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347) |
| One undecodable or unclassifiable page can cost a whole statement, and a new page returns NULL | [ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L172), [ginfuncs.c#gin_leafpage_items-flags](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L211-L226), [rawpage.c#get_page_from_raw](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L214-L234), [rawpage.c#page_header](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L245-L266), [bufpage.h#PageIsNew](../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234) |
| The settle step cannot run on a standby | [utility.c#vacuum-not-read-only](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L295), [utility.c#recovery-gate](../../../raw/postgres-17/src/backend/tcop/utility.c#L570-L583), [utility.c#PreventCommandDuringRecovery](../../../raw/postgres-17/src/backend/tcop/utility.c#L440-L449) |
| The run's settings and their apply scopes | [guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457), [guc_tables.c#statement_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631), [guc_tables.c#gin_pending_list_limit](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585) |
| No shipped test scores a GIN page census against the metapage, the FSM or a rebuild | [gin.sql#pageinspect-gin](../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L1-L19), [gin.sql#pageinspect-failures](../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L21-L39), [gin.sql#pending-list](../../../raw/postgres-17/src/test/regress/sql/gin.sql#L8-L30), [pgstattuple.sql#pgstatginindex](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L47-L63) |

## Open Questions

- **The protocol fixes no fixture corpus, so coverage is self-reported.** [Coverage the protocol requires](#coverage-the-protocol-requires) lists behaviors, not fixtures, and a run says which of its own fixtures reached each one. Two consumer pages can therefore both conform while exercising very different shapes, and nothing here makes their scores comparable. Whether this page should grow a named corpus, or a second page should hold one, is undecided.
- **The pending-list hole cannot be closed from SQL.** A direct `gin_clean_pending_list()` by the index's owner is invisible to the measurement lock, and each census on either side of it is internally consistent, so no cross-check flags the pair. The protocol handles it by operational rule, which is not enforcement.
- **A maintenance command entirely between the two progress reads is undetectable.** The concurrency flag is a sampling of two instants; a `VACUUM` that starts and finishes inside the census leaves nothing for the flag, the size bracket or the metapage check to catch. Only the lock removes the case, and the lock is not always available.
- **No structural verifier corroborates the page classification.** With no GIN entry point in `contrib/amcheck`, a census's classification is checked only against three readings that share its assumptions. A misclassification consistent with the metapage would pass every cross-check here.
- **The decision threshold is the method's, not the protocol's.** Because a GIN waste reading has no engine-defined target density, this page deliberately fixes the oracle and leaves the rebuild threshold to the method. That makes a `FALSE NEGATIVE` on one page incomparable with a `FALSE NEGATIVE` on another, and there is no source basis in v17 for choosing a shared threshold.
- **The oracle is budget-dependent and the protocol only requires it to be labelled.** A run reports the `maintenance_work_mem` it rebuilt at and, where required, more than one budget; it does not define which budget the "true" reclaimable figure belongs to.

## Source References

- [pgstattuple.c#pgstat_relation-gin](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L260-L296)
- [pgstatindex.c#pgstatginindex_internal](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L506-L577)
- [pgstatindex.c#GinIndexStat](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L97-L108)
- [pgstattuple--1.4--1.5.sql#pgstatginindex-grant](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57)
- [pgstattuple.control:3](../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L3)
- [amcheck.control:3](../../../raw/postgres-17/contrib/amcheck/amcheck.control#L3)
- [amcheck--1.3--1.4.sql#bt-only](../../../raw/postgres-17/contrib/amcheck/amcheck--1.3--1.4.sql#L13-L25)
- [check_btree.sql#bt_index_check](../../../raw/postgres-17/contrib/amcheck/sql/check_btree.sql#L25-L26)
- [pageinspect.control:3](../../../raw/postgres-17/contrib/pageinspect/pageinspect.control#L3)
- [rawpage.c#get_raw_page_internal](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L199)
- [rawpage.c#get_page_from_raw](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L214-L234)
- [rawpage.c#page_header](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L245-L266)
- [ginfuncs.c#gin_metapage_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95)
- [ginfuncs.c#gin_page_opaque_info](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L172)
- [ginfuncs.c#gin_leafpage_items-flags](../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L211-L226)
- [gin.sql#pageinspect-gin](../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L1-L19)
- [gin.sql#pageinspect-failures](../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L21-L39)
- [pgstattuple.sql#pgstatginindex](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L47-L63)
- [pg_freespacemap.control:3](../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.control#L3)
- [pg_freespacemap.c#pg_freespace](../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L50)
- [README#concurrency-highkey](../../../raw/postgres-17/src/backend/access/gin/README#L309-L314)
- [README#page-deletion](../../../raw/postgres-17/src/backend/access/gin/README#L389-L396)
- [ginvacuum.c#ginDeletePage-setdeleted](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192)
- [ginvacuum.c#analyze_only](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L729)
- [ginvacuum.c#ginvacuumcleanup-census](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L789)
- [ginvacuum.c#pages_free](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L766-L794)
- [ginvacuum.c#ginvacuumcleanup-relength](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L794-L802)
- [ginvacuum.c#GinPageIsRecyclable](../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)
- [ginutil.c#ginhandler-vacuum](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L64-L72)
- [ginutil.c#GinNewBuffer](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L335)
- [ginutil.c#ginoptions](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L601-L614)
- [ginutil.c#ginUpdateStats](../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L645-L650)
- [ginfast.c#foreground-cleanup](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L452-L471)
- [ginfast.c#shiftList-flags](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L630-L635)
- [ginfast.c#ginInsertCleanup](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L776-L783)
- [ginfast.c#flush-entry-insert](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L926-L933)
- [ginfast.c#gin_clean_pending_list](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091)
- [ginfast.c#recovery-refusal](../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1037-L1041)
- [ginentrypage.c#entryIsEnoughSpace](../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482)
- [ginbtree.c#split-newbuffer](../../../raw/postgres-17/src/backend/access/gin/ginbtree.c#L463-L466)
- [gindatapage.c#dataSplit-newbuffer](../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L1822-L1826)
- [gininsert.c#build-flush](../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L286-L292)
- [ginblock.h#GinMetaPageData](../../../raw/postgres-17/src/include/access/ginblock.h#L55-L101)
- [ginblock.h#planner-stats](../../../raw/postgres-17/src/include/access/ginblock.h#L77-L83)
- [ginblock.h#GIN_CURRENT_VERSION](../../../raw/postgres-17/src/include/access/ginblock.h#L86-L103)
- [ginblock.h#GinPageGetDeleteXid](../../../raw/postgres-17/src/include/access/ginblock.h#L132-L138)
- [ginblock.h#GinDataLeafPageGetFreeSpace](../../../raw/postgres-17/src/include/access/ginblock.h#L287)
- [ginblock.h#pd_lower-untrusted](../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309)
- [bufpage.h#PageIsNew](../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234)
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)
- [bufpage.c#PageGetExactFreeSpace](../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L952-L973)
- [htup_details.h#MaxHeapTupleSize](../../../raw/postgres-17/src/include/access/htup_details.h#L563)
- [indexfsm.c#index-fsm-notes](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L20)
- [indexfsm.c#GetFreeIndexPage](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L32-L46)
- [indexfsm.c#RecordFreeIndexPage](../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)
- [freespace.c#FSM_CATEGORIES](../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L37-L66)
- [freespace.c#fsm_space_cat_to_avail](../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L427-L435)
- [vacuumlazy.c#do_index_cleanup-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L397)
- [vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L720-L731)
- [vacuumlazy.c#lazy_cleanup_all_indexes-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066)
- [vacuumlazy.c#bypass-keeps-cleanup](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1949)
- [vacuumlazy.c#failsafe-disables-cleanup](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2320-L2326)
- [vacuum.c#vacuum-lockmode](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2056)
- [analyze.c#analyze-lockmode](../../../raw/postgres-17/src/backend/commands/analyze.c#L135-L145)
- [index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)
- [indexcmds.c#ReindexIndex-locks-and-dispatch](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2824-L2849)
- [indexcmds.c#RangeVarCallbackForReindexIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2857-L2872)
- [lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L64-L104)
- [lockcmds.c#RangeVarCallbackForLockTable](../../../raw/postgres-17/src/backend/commands/lockcmds.c#L70-L107)
- [lockcmds.c#LockTableAclCheck](../../../raw/postgres-17/src/backend/commands/lockcmds.c#L279-L299)
- [dbsize.c#calculate_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L302-L343)
- [dbsize.c#pg_relation_size](../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L371)
- [reloptions.c#fastupdate](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131)
- [reloptions.c#gin_pending_list_limit](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347)
- [utility.c#vacuum-not-read-only](../../../raw/postgres-17/src/backend/tcop/utility.c#L280-L295)
- [utility.c#PreventCommandDuringRecovery](../../../raw/postgres-17/src/backend/tcop/utility.c#L440-L449)
- [utility.c#recovery-gate](../../../raw/postgres-17/src/backend/tcop/utility.c#L570-L583)
- [system_views.sql#pg_stat_progress_analyze](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1188-L1207)
- [system_views.sql#pg_stat_progress_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1209-L1227)
- [system_views.sql#pg_stat_progress_create_index](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1256-L1289)
- [guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457)
- [guc_tables.c#maintenance_work_mem](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2474)
- [guc_tables.c#statement_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)
- [guc_tables.c#lock_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)
- [guc_tables.c#gin_pending_list_limit](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)
- [gin.sql#pending-list](../../../raw/postgres-17/src/test/regress/sql/gin.sql#L8-L30)
- [gin.sgml#GIN-Fast-Update](../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537)
- [mvcc.sgml#SHARE-ROW-EXCLUSIVE](../../../raw/postgres-17/doc/src/sgml/mvcc.sgml#L1023-L1043)
- [func.sgml#pg_relation_size](../../../raw/postgres-17/doc/src/sgml/func.sgml#L29627-L29643)
- [pgfreespacemap.sgml#index-pages](../../../raw/postgres-17/doc/src/sgml/pgfreespacemap.sgml#L61-L71)

## Navigation

- [v17/index](../index.md)
- [wiki index](../../index.md)
- [versions](../../versions.md)
