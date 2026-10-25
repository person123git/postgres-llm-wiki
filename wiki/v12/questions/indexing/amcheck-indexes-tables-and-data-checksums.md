---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# How amcheck Checks B-Tree Indexes and Their Tables in PostgreSQL 12, What It Finds, Its Limits, and How Data Checksums Change Them (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Mental model](#mental-model)
  - [Logic map](#logic-map)
  - [Calling the functions](#calling-the-functions)
  - [The index walk](#the-index-walk)
  - [Checks on every page](#checks-on-every-page)
  - [What the parent check adds](#what-the-parent-check-adds)
  - [How the table is checked through heapallindexed](#how-the-table-is-checked-through-heapallindexed)
  - [What amcheck finds](#what-amcheck-finds)
  - [What amcheck cannot find](#what-amcheck-cannot-find)
  - [Bloom filter sizing worked example](#bloom-filter-sizing-worked-example)
  - [How data checksums work in PostgreSQL 12](#how-data-checksums-work-in-postgresql-12)
  - [How checksums change amcheck's limits](#how-checksums-change-amchecks-limits)
  - [Non-default settings and special cases](#non-default-settings-and-special-cases)
  - [Lifecycle events](#lifecycle-events)
  - [Checking a non-temporary physical table](#checking-a-non-temporary-physical-table)
  - [Build boundary and tests](#build-boundary-and-tests)
  - [Causal summary](#causal-summary)
- [Measurement Script](#measurement-script)
  - [Results and last run](#results-and-last-run)
  - [Usage](#usage)
  - [Settings and fixtures](#settings-and-fixtures)
  - [Script](#script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Follow AGENTS.md. In PostgreSQL 12, explain how amcheck works for indexes and tables. Also explain in detail what issues it finds, what its limitations are, and how having data checksums enabled or not reduces amcheck's limitations.

## Answer

### Short answer

**PostgreSQL 12's [`amcheck`](../../../glossary.md#amcheck) checks [B-tree](../../../glossary.md#b-tree) indexes only. It has no function that takes a table.** The module builds one source file, `verify_nbtree.c`, and its two functions reject any relation that is not a B-tree index. [Makefile#amcheck](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4) [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323) [check_btree.sql#table-rejection](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L41-L43)

A table is checked only indirectly. When you pass `heapallindexed = true`, the checker scans the index's table and asks, for each row that should be in the index, whether the index holds a matching entry. So the question's premise holds only in this narrow sense: amcheck tests a table through the indexed columns of the rows its scan finds. [verify_nbtree.c#heap-phase](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L625) [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926)

It finds three kinds of problems:

- **Broken index structure:** zero or malformed pages, bad line pointers, wrong levels, broken sibling links, and keys out of order within a page, across pages, or between a parent and its child. [verify_nbtree.c#bt_target_page_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870)
- **Index entries that the operator class no longer orders the same way**, for example after an operating-system collation change. [amcheck.sgml#collation](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L222-L250)
- **[Heap](../../../glossary.md#heap) rows that lack a matching index entry**, including rows whose indexed bytes were damaged, and some HOT-chain and TOAST damage met along the way. [verify_nbtree.c#callback-rationale](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1837-L1891)

Its main limits follow from that design. It reads only the index pages its tree walk reaches, and only the heap rows its scan returns. It never asks the reverse question, whether every index entry still has a heap row. Its row test is probabilistic. It sees pages as they sit in [shared buffers](../../../glossary.md#shared_buffers), and it stops at the first error. [verify_nbtree.c#root-to-leaf-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L551) [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10) [amcheck.sgml#shared-buffers](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L280-L293)

[Data checksums](../../../glossary.md#data-checksums) cover a different gap. They are off by default. When a cluster has them, an initialized page that amcheck causes to be **read from disk** gets a checksum comparison before amcheck sees it. A mismatch raises an error unless the superuser-only `ignore_checksum_failure` is on; random damage can also match the 16-bit checksum by chance (see [the checksum section](#how-data-checksums-work-in-postgresql-12)). That can catch damage in bytes amcheck never interprets, such as non-indexed columns, row headers and free space. Checksums do not help with pages already in shared buffers, pages amcheck never reads, [all-zero pages](../../../glossary.md#all-zero-page), truncated files, or damage that existed in memory before an initialized page was written with a valid checksum. [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162)

The explanation follows the pinned source. The [measurement script](#measurement-script) also tests selected boundaries with checksums off and on: valid dead index entries, target selection, TOAST reads from predicates, free-space damage, statistics reporting and base-backup verification. Other cases remain source deductions.

### Mental model

amcheck checks **invariants**: conditions that must always hold if the index is sound. An example is that items on a B-tree page are in key order. A failed invariant raises an error. A pass means only that no checked invariant failed. [amcheck.sgml#invariants](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L10-L28) [amcheck.sgml#cannot-prove-absence](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L316-L317)

These values decide what a call checks:

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| `parentcheck` / `readonly` | Which function was called. `bt_index_parent_check` sets it. | The entry point passes it to the worker; the state keeps it as `readonly`. [verify_nbtree.c#bt_index_parent_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L203-L218) [verify_nbtree.c#state-init](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L405-L411) | Live, per call | Chooses the [lock mode](../../../glossary.md#lock-mode) and prevents structural changes needed to make the extra checks safe. Concurrent scans can still set `LP_DEAD` hints. [verify_nbtree.c#sharelock-boundary](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L373-L384) |
| `heapallindexed` | Whether to compare the table with the index. | Second argument; false when omitted. [verify_nbtree.c#bt_index_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L180-L192) | Live, per call | Builds the index fingerprint and runs the table scan. |
| `rootdescend` | Whether to re-find each leaf entry from the root. | Third argument of the parent check only; false when omitted. [verify_nbtree.c#bt_index_parent_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L206-L213) | Live, per call | Adds one root-to-leaf search per leaf entry. |
| Root block and level | Where the walk starts. | The [metapage](../../../glossary.md#metapage), block 0 of the index. [verify_nbtree.c#metapage](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L500-L529) | Stored on disk | Starting point of the level-by-level walk. |
| Page copy | The bytes amcheck examines. | A private copy taken from a shared buffer, which holds either a cached copy or a fresh disk read. [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2342-L2360) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) | Stored; can be older than the file when the buffer was cached | Input to every page check. |
| Index fingerprint | A [Bloom filter](../../../glossary.md#bloom-filter) of leaf data entries not marked `LP_DEAD` when examined. | Built during the walk. [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083) | Live, in memory, freed at the end | Probed once per table row that belongs in the index. |
| Scan snapshot | Which table rows count. | `bt_index_check` registers an [MVCC](../../../glossary.md#mvcc) [snapshot](../../../glossary.md#snapshot); the parent check uses `SnapshotAny` plus a [vacuum](../../../glossary.md#vacuum) horizon. [verify_nbtree.c#snapshot](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L394) [verify_nbtree.c#snapshot](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L445-L447) [heapam_handler.c#snapshot-choice](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1219-L1223) | Live, per call | Decides which rows are probed. |
| [`pg_class.reltuples`](../../../glossary.md#reltuples-and-relpages), index size, [`maintenance_work_mem`](../../../glossary.md#maintenance_work_mem) | Inputs to the fingerprint's size. | Catalog estimate, current block count, setting. [verify_nbtree.c#bloom-sizing](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L419-L432) | `reltuples` is stored and can be stale; the other two are live | Decide the chance of missing a missing entry. |
| `pd_checksum` | 16-bit checksum in each page header. | Written when an initialized page is flushed with checksums on. [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L150-L163) [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) | Stored on disk | Compared on complete reads of initialized pages when checksums are on. [bufpage.c#verify-gate](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L103) |
| `data_checksum_version` | Whether the cluster has checksums. | The [control file](../../../glossary.md#control-file). [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826) | Stored; changes only at `initdb` or with offline `pg_checksums` | Gates checksum work on reads and writes. |

Two of these values are easy to confuse. The **index fingerprint** is built from the index pages amcheck reads during this call. The **table rows** come from a separate heap scan. The comparison runs in one direction only: each heap row asks whether the fingerprint holds its entry. It can therefore find heap rows that the index lacks, but never index entries whose heap row is gone. [verify_nbtree.c#root-to-leaf-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L551) [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614)

### Logic map

```text
bt_index_check(idx [, heapallindexed])      bt_index_parent_check(idx [, heapallindexed [, rootdescend]])
   AccessShareLock on table, then index        ShareLock on table, then index (refused on a hot standby)   [A]
                     \                         /
                      v                       v
        Target checks: B-tree? valid? not another session's temp index? main fork present?             [B]
                                 |
                                 v
        Read metapage (block 0): meta flag, magic number, version 2..4                                   [C]
                                 |
                                 v
        For each level, root down to leaf: start at leftmost page, follow right links                    [D]
          each page: buffer read (checksum verified only on a disk read) -> zero/special check -> copy   [E]
          per page: level, item count, high key, line pointers, tuple size and attributes                [F]
          order: compare real items, high key and comparable right-sibling item; legacy rules relax <   [G]
          parent check only: sibling links agree; comparable child keys > downlink in version 4          [H]
            versions 2/3 allow equality; negative-infinity items are excluded; reject visited deleted    [H]
          heapallindexed: normalize each non-dead leaf entry (reject external varlena) -> Bloom filter   [I]
          parent check + heapallindexed: Bloom-probe downlinks; allow interrupted split/deletion states   [J]
          rootdescend: search the tree from the root for every leaf entry                                [K]
                                 |
              heapallindexed false? --yes--> return; no error means no checked invariant failed
                                 | no
                                 v
        Table scan: bt_index_check -> MVCC snapshot; parent check -> SnapshotAny + vacuum horizon        [L]
          each heap page: buffer read (checksum verified only on a disk read)                            [E]
          each row that belongs in the index: partial-index predicate, HOT root TID, TOAST fetch         [M]
            build normalized index entry -> probe Bloom filter -> "definitely absent"? ERROR             [N]
                                 |
                                 v
                              return
```

- **[A]** [verify_nbtree.c#lock-choice](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L232-L263), [lock.c#recovery-lock-limit](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766)
- **[B]** [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L338), [verify_nbtree.c#main-fork](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L279-L294)
- **[C]** [verify_nbtree.c#metapage-checks](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2370-L2394), [nbtree.h#BTREE_VERSION](../../../../raw/postgres-12/src/include/access/nbtree.h#L131-L135)
- **[D]** [verify_nbtree.c#root-to-leaf-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L551), [verify_nbtree.c#bt_check_level_from_leftmost](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L649-L835)
- **[E]** [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2348-L2360), [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L377-L379), [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925), [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L719)
- **[F]** [verify_nbtree.c#page-sanity](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2396-L2471), [verify_nbtree.c#tuple-checks](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L884-L976), [verify_nbtree.c#PageGetItemIdCareful](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2514-L2548)
- **[G]** [verify_nbtree.c#order-checks](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1130-L1258)
- **[H]** [verify_nbtree.c#sibling-agreement](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L783-L793), [verify_nbtree.c#bt_downlink_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1508-L1647), [verify_nbtree.c#lower-bound-version-rules](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2273-L2319)
- **[I]** [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083)
- **[J]** [verify_nbtree.c#bt_downlink_missing_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1662-L1835)
- **[K]** [verify_nbtree.c#rootdescend-call](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L995-L1018), [verify_nbtree.c#bt_rootdescend](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2058-L2107)
- **[L]** [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614), [heapam_handler.c#snapshot-choice](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1212-L1260)
- **[M]** [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1666), [indextuple.c#external-fetch](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88)
- **[N]** [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926), [verify_nbtree.c#bt_normalize_tuple](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1961-L2032)

### Calling the functions

The extension installs five SQL overloads backed by two C entry points, through three version scripts:

| SQL signature | Script | Behavior of omitted arguments |
|---|---|---|
| `bt_index_check(regclass)` | 1.0 | `heapallindexed` is false. |
| `bt_index_parent_check(regclass)` | 1.0 | `heapallindexed` and `rootdescend` are false. |
| `bt_index_check(regclass, boolean)` | 1.1 | — |
| `bt_index_parent_check(regclass, boolean)` | 1.1 | `rootdescend` is false. |
| `bt_index_parent_check(regclass, boolean, boolean)` | 1.2 | — |

[amcheck--1.0.sql#functions](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L6-L24) [amcheck--1.0--1.1.sql#heapallindexed](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0--1.1.sql#L9-L29) [amcheck--1.1--1.2.sql#rootdescend](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.1--1.2.sql#L9-L19) [amcheck.control](../../../../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L5) [verify_nbtree.c#entry-points](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L180-L218)

Every overload is declared `STRICT`. A strict function is not executed when any argument is null, so `bt_index_check(idx, NULL)` returns null **without checking anything**. [amcheck--1.0--1.1.sql#heapallindexed](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0--1.1.sql#L12-L16) [create_function.sgml#STRICT](../../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L385-L390)

Each script revokes `EXECUTE` from `PUBLIC`. The C code deliberately skips any permission check on the relation, so a role that has been granted `EXECUTE` can check every index in the cluster. The regression test grants `EXECUTE` to an ordinary role and runs both checks as that role. [amcheck--1.0.sql#revoke](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L21-L24) [verify_nbtree.c#no-permission-check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L306-L313) [check_btree.sql#permissions](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L23-L39) Installing the extension is different: `amcheck.control` sets no `superuser` line, and the control-file parser defaults it to true, so `CREATE EXTENSION amcheck` needs a superuser. [amcheck.control](../../../../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L5) [extension.c#superuser-default](../../../../raw/postgres-12/src/backend/commands/extension.c#L616) [extension.c#superuser-check](../../../../raw/postgres-12/src/backend/commands/extension.c#L803)

The worker locks the table before the index, to avoid deadlocks. `bt_index_check` takes `AccessShareLock`, the mode plain `SELECT` takes. `bt_index_parent_check` takes `ShareLock`, which conflicts with `RowExclusiveLock`, so it blocks `INSERT`, `UPDATE` and `DELETE` on the table while it runs. It releases both locks when it returns rather than at commit. [verify_nbtree.c#lock-choice](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L232-L263) [lock.c#ShareLock-conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86) [verify_nbtree.c#early-release](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L296-L303) [amcheck.sgml#parent-locks](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L144-L154)

Before the explicit page walk, relation opening and the target checks can reject the call with a usage error rather than a corruption report:

| Condition | Result |
|---|---|
| Not an index | `"..." is not an index` (`42809`); `index_open()` rejects it before the amcheck target test |
| An index that is not a supported physical B-tree | `only B-Tree indexes are supported as targets for verification` (`0A000`) |
| Index of another session's temporary table | `cannot access temporary tables of other sessions` |
| [`pg_index.indisvalid`](../../../glossary.md#invalid-index) is false | `cannot check index ...`, detail `Index is not valid.` |
| Table lookup raced with a drop or re-create | `could not open parent table of index ...` |

[indexam.c#index_open](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L126-L140) [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L338) [verify_nbtree.c#parent-table](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L265-L274) [check_btree.out#target-errors](../../../../raw/postgres-12/contrib/amcheck/expected/check_btree.out#L46-L61) [errcodes.txt:119](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L119)

The worker then confirms that the index's main [fork](../../../glossary.md#fork) file exists, and reports `index "..." lacks a main relation fork` if it does not. [verify_nbtree.c#main-fork](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L279-L294)

### The index walk

1. **Start at the true root.** amcheck reads the metapage and checks its meta flag, its magic number and that its version lies between 2 and 4. It starts from the true root recorded there, not from the [fast root](../../../glossary.md#fast-root) that ordinary searches use, because the comment calls that more thorough and immune to a stale fast-root pointer. [verify_nbtree.c#metapage-checks](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2370-L2394) [verify_nbtree.c#true-root](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L504-L529)
2. **Walk one level at a time, left to right.** For each level it starts at the leftmost page and follows each page's right [sibling link](../../../glossary.md#sibling-link) until the rightmost page. The first downlink on the leftmost internal page gives the leftmost page of the next level down. The walk ends after the [leaf](../../../glossary.md#leaf-page) level. [verify_nbtree.c#bt_check_level_from_leftmost](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L649-L835) [verify_nbtree.c#next-level](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L751-L774)
3. **Read each page through the [buffer manager](../../../glossary.md#buffer-manager), then copy it.** `palloc_btree_page()` reads the block with `ReadBufferExtended()` and a bulk-read [ring buffer](../../../glossary.md#ring-buffer) strategy, runs B-tree's own `_bt_checkpage()`, copies the page into private memory and releases the buffer at once. Working on a copy means a misbehaving operator class cannot leave the backend stuck while it holds a buffer lock. [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2322-L2360) [verify_nbtree.c#bulkread](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L498)
4. **Skip pages that are being deleted.** A half-dead or deleted page is logged at `DEBUG1` and skipped. Under the parent check a deleted page reached by a level's leftmost downlink or by a sibling link is an error, because nothing may delete pages while `ShareLock` is held. A skipped page that claims to be rightmost is always an error (`block ... fell off the end of index ...`). [verify_nbtree.c#ignorable-pages](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L685-L717)
5. **Check the page, then move right.** Each page goes through `bt_target_page_check()`. A page whose right link or left link points to itself raises `circular link chain found`. [verify_nbtree.c#check-and-move](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L804-L829)

Because the walk follows links, it reads only pages that some link reaches. A block that no metapage, downlink or sibling link points to is never read. That includes fully deleted pages, which by design have no links from siblings or parents. [verify_nbtree.c#deleted-pages-unlinked](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L686-L697)

### Checks on every page

Both functions perform the applicable checks below. Basic page-copy checks also run on ignored pages, but the walk skips their tuple and ordering checks; fully deleted pages have no usable level field. [verify_nbtree.c#ignored-pages](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L683-L717) [verify_nbtree.c#deleted-level-exception](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2396-L2411)

| Check | Error when it fails | Source |
|---|---|---|
| Page is not all zero, and its special area has the B-tree size | `index "..." contains unexpected zero page at block ...` / `contains corrupted page at block ...` | [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L719) |
| A page other than block 0 is not a metapage | `invalid meta page found at block ...` | [verify_nbtree.c:2364-2368](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2364-L2368) |
| Non-deleted leaf pages are level 0; non-deleted internal pages are not | `invalid leaf page level ...` / `invalid internal page level 0 ...` | [verify_nbtree.c:2400-2411](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2400-L2411) |
| Item count is plausible; internal pages have a downlink; non-rightmost leaves have a high key | `Number of items ... exceeds MaxIndexTuplesPerPage`, `... lacks high key and/or at least one downlink`, `... lacks high key item` | [verify_nbtree.c:2431-2449](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2431-L2449) |
| Internal pages are never half-dead and never have garbage items | `internal page block ... is half-dead` / `... has garbage items` | [verify_nbtree.c:2460-2471](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2460-L2471) |
| Each page's level matches the level being walked | `leftmost down link for level points to block ... whose level is not one level down` | [verify_nbtree.c:795-802](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L795-L802) |
| Each checked [line pointer](../../../glossary.md#line-pointer) has nonzero storage ending before the B-tree special area, and is `LP_NORMAL` or `LP_DEAD`, not unused or redirected | `line pointer points past end of tuple space` / `invalid line pointer storage` | [verify_nbtree.c#PageGetItemIdCareful](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2514-L2548) |
| The tuple's own length equals the line pointer's length | `index tuple size does not equal lp_len`, hint `This could be a torn page problem.` | [verify_nbtree.c:941-951](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L941-L951) |
| High key and items have the expected attribute count, including that leaf data items do not use the pivot `INDEX_ALT_TID_MASK` flag | `wrong number of high key index tuple attributes` / `wrong number of index tuple attributes` | [verify_nbtree.c:884-976](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L884-L976), [nbtutils.c#_bt_check_natts](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L2382-L2474) |
| Comparable non-high-key items (excluding the internal negative-infinity item) do not exceed the version's size limit | `index row size ... exceeds maximum` | [verify_nbtree.c:988-993](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993), [verify_nbtree.c:1047-1070](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1047-L1070) |
| On an equal-prefix tiebreaker in a version 4 index, the compared upper-bound leaf item (next item on the page, or child item under the parent check) carries a heap [TID](../../../glossary.md#tid) | `... contains non-pivot tuple that lacks a heap TID` | [verify_nbtree.c:2181-2202](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2181-L2202), [verify_nbtree.c:2296-2317](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2296-L2317), [verify_nbtree.c#BTreeTupleGetHeapTIDCareful](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2559-L2573), [nbtree.h#BTreeTupleGetHeapTID](../../../../raw/postgres-12/src/include/access/nbtree.h#L346-L355) |
| Every comparable non-high-key item (excluding the internal negative-infinity item) is at most the page's high key (strictly less on internal pages) | `high key invariant violated` | [verify_nbtree.c:988-993](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993), [verify_nbtree.c:1130-1152](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1130-L1152) |
| Every comparable non-high-key item (excluding the internal negative-infinity item) is strictly less than the next item | `item order invariant violated` | [verify_nbtree.c:988-993](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993), [verify_nbtree.c:1160-1199](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1160-L1199) |
| The last comparable item is less than the first comparable item of the next live page to the right, when that item exists; skip the internal negative-infinity item | `cross page item order invariant violated` | [verify_nbtree.c:1218-1258](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1218-L1258), [verify_nbtree.c#bt_right_page_check_scankey](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1301-L1497) |
| A level has at least one live page | `index "..." has no valid pages on level below ...` | [verify_nbtree.c:544-548](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L544-L548) |

The ordering checks compare keys with the index's own B-tree comparison support function, the same code index scans use. That is why amcheck can detect an [operator class](../../../glossary.md#operator-class) or [collation](../../../glossary.md#collation) whose ordering changed after the index was built. [amcheck.sgml#opclass-code](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L29-L36) [verify_nbtree.c#invariant_l_offset](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2154-L2205)

`bt_index_check` can run while other sessions change the index, so the cross-page check has one race to rule out. When the check seems to fail, it re-reads the target page. If the page has meanwhile been deleted, the apparent violation was caused by that concurrent deletion and is not reported. [verify_nbtree.c#cross-page-recheck](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1228-L1247) [verify_nbtree.c#race-explanation](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1365-L1454)

An index carried over by [`pg_upgrade`](../../../glossary.md#pg_upgrade) from a release before on-disk version 4 may legally hold equal neighbouring keys. For such indexes the "strictly less" checks relax to "less than or equal", which is a weaker check. [verify_nbtree.c#pg_upgrade-relaxation](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2166-L2168) [nbtpage.c#_bt_heapkeyspace](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L638-L686)

### What the parent check adds

`bt_index_parent_check` holds `ShareLock`, so no page split, page deletion or `VACUUM` can run during the call. That stability is what makes these extra checks safe from false alarms. [verify_nbtree.c#sharelock-rationale](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L373-L384)

| Extra check | Needs | Error when it fails |
|---|---|---|
| Each level's first page is leftmost; the top level's page is the true root | parent check | `block ... is not leftmost` / `block ... is not true root` [verify_nbtree.c:727-740](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L727-L740) |
| Each page's left link names the page the walk came from | parent check | `left link/right link pair ... not in agreement` [verify_nbtree.c:783-793](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L783-L793) |
| No level-walk link or non-negative-infinity downlink reaches a deleted page | parent check | `downlink or sibling link points to deleted block` / `downlink to deleted page found` [verify_nbtree.c:698-704](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L698-L704) [verify_nbtree.c:988-993](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993) [verify_nbtree.c:1268-1273](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1268-L1273) [verify_nbtree.c:1593-1601](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1593-L1601) |
| Comparable child keys are strictly greater than the parent's separator in version 4; versions 2/3 allow equality. Negative-infinity parent and child items are excluded | parent check | `down-link lower bound invariant violated` [verify_nbtree.c#downlink-call](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993) [verify_nbtree.c#bt_downlink_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1603-L1644) [verify_nbtree.c#version-rule](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2273-L2319) |
| A reached non-root, non-ignored page's block number is not definitely absent from the downlink filter, unless an interrupted split/deletion explains the missing link; when descending a missing-downlink internal subtree via negative-infinity downlinks, each child level is one level down and the leaf is not deleted | parent check **and** `heapallindexed = true` | `leaf index block lacks downlink` / `downlink points to block ... whose level is not one level down` / `downlink to deleted leaf page found` / `internal index block lacks downlink` [verify_nbtree.c#missing-downlink-call](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1276-L1282) [verify_nbtree.c#bt_downlink_missing_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1662-L1834) |
| Each leaf entry can be found by a new search from the root | parent check, `rootdescend = true`, on-disk version 4 | `could not find tuple using search from root page` [verify_nbtree.c:995-1018](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L995-L1018) [verify_nbtree.c#version-gate](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L486-L492) |

The code enables the missing-downlink check only when both the parent check and `heapallindexed` are selected. It allocates a separate Bloom filter with [`work_mem`](../../../glossary.md#work_mem) as its budget (default `4096` kB, context `user`, settable per session or transaction with `SET`); it does not reuse the tuple filter's allocation or its `maintenance_work_mem` budget. Internal entries supply child block numbers to this filter, and reached pages probe their own block numbers. A Bloom false positive can therefore hide a missing downlink. That same check is also the only place that follows non-leftmost negative-infinity (`P_FIRSTDATAKEY`) downlinks to test whether they point to a deleted leaf page bypassed by sibling links, because `bt_downlink_check()` skips negative-infinity items. The source establishes this option gate, but gives no reason to describe it as memory reuse. [verify_nbtree.c#tuple-filter](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L413-L432) [verify_nbtree.c#downlinkfilter](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L470-L483) [guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2241) [verify_nbtree.c#downlink-fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L978-L986) [verify_nbtree.c#downlink-probe](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1714-L1807) [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L16)

The check also tolerates interrupted [page splits](../../../glossary.md#page-split) and interrupted multi-level [page deletion](../../../glossary.md#b-tree-page-deletion), where a missing downlink is legitimate. [verify_nbtree.c#tolerated-cases](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1649-L1719) [verify_nbtree.c#halfdead-chain](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1809-L1825)

`rootdescend` exists because downlink and high-key checks cannot see every key-space error. The first item of an internal page carries no key, so some inconsistencies between "cousin" leaf pages slip past. A fresh search starting at the fast root for every leaf entry adds coverage of these errors. The search stays inside the index: the heap TID is a search key, not an instruction to fetch the row. It is still incomplete: `_bt_search()` can move right without requiring an incomplete-split marker, which the source explicitly says can conceal an inconsistency. [verify_nbtree.c#rootdescend-rationale](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2034-L2057) [verify_nbtree.c#bt_rootdescend](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2058-L2107)

### How the table is checked through heapallindexed

The principle is that a fresh `CREATE INDEX` with the same definition must produce only entries that the existing index already holds. amcheck runs a "dummy" build: it computes the entries a new index would get and checks each one against a fingerprint of the existing index. [amcheck.sgml#heapallindexed](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L171-L187) [verify_nbtree.c#callback-rationale](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1843-L1849)

1. **Fingerprint the index during the walk.** Each leaf entry not marked dead (`LP_DEAD`, a [killed index tuple](../../../glossary.md#killed-index-tuple)) is normalized and added to the Bloom filter. Normalization rejects any index attribute carrying an external [TOAST](../../../glossary.md#toast) pointer (`external varlena datum in tuple that references heap row ...`, `XX002`), and decompresses compressed values before re-forming the tuple, because the heap and the index can compress the same value differently and a bitwise comparison would then report a false mismatch. [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083) [verify_nbtree.c#bt_normalize_tuple](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1928-L2032)
2. **Choose which rows count.** `bt_index_check` registers an MVCC snapshot **before** the walk starts, so every row that snapshot can see must already have its entry in the index. The parent check instead scans with `SnapshotAny` and judges each row with `HeapTupleSatisfiesVacuum()`, like a normal index build. That also covers recently deleted rows, except HOT-updated ones, whose entry belongs to the end of the chain. [verify_nbtree.c#snapshot](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L435-L469) [verify_nbtree.c#ii_Concurrent](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L586-L597) [heapam_handler.c#snapshot-choice](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1212-L1260) [heapam_handler.c#vacuum-visibility](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1364-L1600)
3. **Scan the heap.** amcheck opens its own [sequential scan](../../../glossary.md#sequential-scan) and hands it to `table_index_build_scan()`, which reads every heap page up to the block count measured when the scan starts. [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614) [tableam.h#scan-and-build-dispatch](../../../../raw/postgres-12/src/include/access/tableam.h#L761-L773) [tableam.h#table_index_build_scan](../../../../raw/postgres-12/src/include/access/tableam.h#L1512-L1533) [heapam_handler.c#scan-loop](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1306-L1309) [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L351-L379)
4. **Compute each row's index entry.** For a [partial index](../../../glossary.md#partial-index), rows that fail the predicate are skipped. Index expressions are evaluated. A [HOT](../../../glossary.md#hot) row is given the TID of its chain's root, as an index entry would be. [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1666)
5. **Probe the filter.** The callback forms an index entry from the computed values and TID, normalizes it and probes the filter. "Definitely absent" raises `heap tuple (b,o) from table "..." lacks matching index tuple within index "..."` with [SQLSTATE](../../../glossary.md#sqlstate) `XX001`. Under `bt_index_check` the error carries a hint to retry with the parent check. [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) [errcodes.txt:493-494](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L493-L494)

The scan also finds some heap damage on the way, through code it shares with index builds:

- **Broken HOT chain.** A heap-only row whose chain root cannot be found raises `failed to find parent tuple for heap-only tuple at (b,o)` (`XX001`). The source comment notes that in this case sequential scans can return correct results and every index can look structurally sound, while index scans give wrong answers. [heapam_handler.c#hot-root-error](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1634-L1652) [verify_nbtree.c#hot-chain-rationale](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1863-L1878)
- **Wrong indexed values in a HOT chain.** Because the newest row's values are paired with the root TID, an index entry with the same TID but stale values fails the probe. This catches the effect of a wrong HOT-safety decision, such as past `CREATE INDEX CONCURRENTLY` bugs. [verify_nbtree.c#wrong-values](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1879-L1890)
- **Missing TOAST data for indexed columns.** `index_form_tuple()` fetches every externally stored value it is given, so a damaged TOAST value in an indexed column raises the TOAST reader's error, such as `missing chunk number ...`. [indextuple.c#external-fetch](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88) [tuptoaster.c#heap_tuple_fetch_attr](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L101-L110) [tuptoaster.c:2013](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2013)
- **TOAST data used by predicates or expressions.** A column need not be stored in the index to be read. The scan evaluates the partial-index predicate before forming the entry, and evaluates index expressions through `FormIndexDatum()`. For example, a predicate `payload LIKE 'a%'` detoasts `payload` through `textlike()` even when the index stores only `id`. Damage encountered during that evaluation can error before the membership probe; this does not provide an independent audit of every source column's bytes. [heapam_handler.c#predicate-and-expressions](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) [like.c#textlike](../../../../raw/postgres-12/src/backend/utils/adt/like.c#L284-L300) [fmgr.h#detoast-macros](../../../../raw/postgres-12/src/include/fmgr.h#L283-L303)
- **Flipped bits in indexed heap bytes.** The probe compares the whole normalized entry bitwise, so a damaged indexed value in the heap no longer matches its index entry. The docs say this gives a greatly increased chance of catching single-bit memory errors. [amcheck.sgml#ram](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L295-L313)

The table phase does **not** check uniqueness. amcheck clears `ii_Unique` and the exclusion fields before the scan, so the build-scan code neither waits on concurrent inserters nor compares keys. Duplicate live keys in a unique index are not reported by any check in this module. [verify_nbtree.c#unique-off](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L599-L607) [heapam_handler.c#checking_uniqueness](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1188-L1190)

### What amcheck finds

amcheck's own corruption errors use one of two SQLSTATEs: `XX002` (`index_corrupted`) for damage inside the index, and `XX001` (`data_corrupted`) for a heap row the index lacks. The build-scan HOT error and the buffer manager's `invalid page` error also use `XX001`. The TOAST reader's errors are plain `elog(ERROR)` calls, which get the default `XX000` (`internal_error`). [errcodes.txt:492-494](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L492-L494) [heapam_handler.c#hot-root-error](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1646-L1652) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L920-L924) [tuptoaster.c:2013](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2013) [elog.c#default-errcode](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388)

| Problem class | Example cause | Detected by | Option needed |
|---|---|---|---|
| Zero page, wrong special-area size, impossible item count, bad line pointers, item lengths or levels | Storage fault or bad write on a page the walk reaches | Page checks above | none |
| Keys out of order inside a page, against the high key, or across siblings | Collation or operator-class change; AM bug | Ordering checks | none |
| Parent and child disagree; sibling links disagree; visited links to deleted pages | Lost or misdirected index write; AM bug | Parent check | `bt_index_parent_check` |
| A reached page with an unjustified missing downlink, or a missing-downlink internal subtree whose negative-infinity chain has a bad level or deleted leaf | Lost parent update or premature page deletion | Probabilistic missing-downlink check | parent check + `heapallindexed` |
| Upper-level key-space errors between cousin pages | AM bug | Root search | parent check + `rootdescend` |
| External varlena pointer stored in an index tuple | Corrupt index tuple header or AM bug | Normalization check | `heapallindexed` |
| Heap row with no index entry | Lost index insert; heap row written without index maintenance | Bloom probe | `heapallindexed` |
| Indexed heap bytes changed | Bit flip in heap or memory | Bloom probe | `heapallindexed` |
| Broken HOT chain; stale values in a HOT chain | Heap or commit-log damage; HOT-safety bug | Build-scan error; Bloom probe | `heapallindexed` |
| Damaged TOAST value needed for index values, a predicate or an expression | Lost TOAST chunk | TOAST reader error during entry formation or expression evaluation | `heapallindexed` |

[amcheck.sgml#effective-use](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L212-L318) [verify_nbtree.c#bt_target_page_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870) [verify_nbtree.c#external-varlena](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1990-L2001)

The docs state that no corruption error from amcheck should ever be a false positive, because each one reports a condition that by definition should never happen. They also warn that there is no general repair method and that [`REINDEX`](../../../glossary.md#reindex) may not fix the cause. [amcheck.sgml#repair](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L321-L337)

### What amcheck cannot find

| Limitation | Why | Consequence |
|---|---|---|
| No dedicated table checker | Only `verify_nbtree.c` is built; tables are rejected. [Makefile#amcheck](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4) [indexam.c#index_open](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L126-L140) | No comprehensive validation of heap row headers, [line pointers](../../../glossary.md#line-pointer) or arbitrary column values. The heap/TOAST checks described above can still find errors incidentally. |
| B-tree only | The same target test rejects GiST, GIN, hash, BRIN and SP-GiST. [check_btree.sql#brin-rejection](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L49-L53) | Other index types get no check at all. |
| One-way table comparison | The callback probes only for rows the scan returns. [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) | An index entry whose heap row vanished is not reported. Rows lost from the heap go unnoticed; see [the zeroed-segment page](amcheck-zeroed-relation-segments.md). |
| Probabilistic row test | A Bloom filter can say "possibly present" for an absent entry. [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10) | Each missing entry escapes with a small probability; the docs give no more than 2% at about 2 bytes per row. See the [worked example](#bloom-filter-sizing-worked-example). [amcheck.sgml#bloom-memory](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L195-L207) |
| Additional parent checks are incomplete | Downlink presence uses a second Bloom filter; `rootdescend` can recover by moving right. [verify_nbtree.c#downlink-probe](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1714-L1719) [verify_nbtree.c#rootdescend-limitation](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2070-L2079) | A successful stronger check still does not prove every downlink is present or correct. |
| Only reachable index pages | The walk follows links. [verify_nbtree.c#root-to-leaf-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L551) | Unlinked blocks, such as deleted pages, are never read. |
| Only rows that belong in the index | Partial-index predicate; snapshot rules. [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) | Rows outside a partial index, and rows invisible to the chosen snapshot, are not probed. |
| No uniqueness check | `ii_Unique` is cleared. [verify_nbtree.c#unique-off](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L599-L607) | Duplicate keys in a unique index pass. |
| Cached pages | A buffer hit returns the cached copy without reading the file. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) [amcheck.sgml#shared-buffers](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L286-L290) | Damage on disk under a clean cached copy is invisible until the page is read again. |
| First error stops the call | Every check uses `ereport(ERROR)`. [verify_nbtree.c:1142-1151](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1142-L1151) [amcheck.sgml#raises-error](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L140-L142) | One call reports one problem; the rest of the index is not checked. |
| All-zero heap pages | The page verifier accepts them; they have no line pointers. [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) | Rows on a zeroed heap page simply disappear from the scan. |
| Forks and other files | The explicit traversal covers the index and heap main forks. Entry formation, predicates and expressions can also fetch TOAST values. [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2348-L2349) [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L377-L378) [heapam_handler.c#predicate-and-expressions](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) [indextuple.c#external-fetch](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88) | No dedicated audit of the [visibility map](../../../glossary.md#visibility-map), [free space map](../../../glossary.md#free-space-map), all TOAST values, commit log or catalogs. Incidental accesses are not complete validation of those structures. |
| Cannot prove absence | Stated by the docs. [amcheck.sgml#cannot-prove-absence](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L316-L317) | A pass is evidence of no detected violation, not of a sound database. |

### Bloom filter sizing worked example

This example shows why `maintenance_work_mem` matters for `heapallindexed`. It is a calculation from the source formulas, not a measurement.

**Inputs and their sources:**

- `MaxIndexTuplesPerPage = (BLCKSZ - SizeOfPageHeaderData) / (MAXALIGN(sizeof(IndexTupleData) + 1) + sizeof(ItemIdData))`. [itup.h#MaxIndexTuplesPerPage](../../../../raw/postgres-12/src/include/access/itup.h#L130-L147)
- [`BLCKSZ`](../../../glossary.md#blcksz) is 8192 by default. The page header before the line-pointer array is 24 bytes, an index tuple header is 8 bytes (a 6-byte TID plus 2 bytes of `t_info`), and a line pointer is 4 bytes. [configure.in#blocksize](../../../../raw/postgres-12/configure.in#L253-L265) [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L150-L163) [itup.h#IndexTupleData](../../../../raw/postgres-12/src/include/access/itup.h#L35-L50) [itemptr.h#ItemPointerData](../../../../raw/postgres-12/src/include/storage/itemptr.h#L28-L40) [itemid.h#ItemIdData](../../../../raw/postgres-12/src/include/storage/itemid.h#L25-L30)
- This example assumes a platform where `MAXIMUM_ALIGNOF` is 8, so `MAXALIGN(9) = 16`. `configure` determines that value per platform. [c.h#MAXALIGN](../../../../raw/postgres-12/src/include/c.h#L685) [configure.in#MAXIMUM_ALIGNOF](../../../../raw/postgres-12/configure.in#L2010)
- `total_elems = max(index_blocks * (MaxIndexTuplesPerPage / 5), reltuples)`. [verify_nbtree.c#bloom-sizing](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L419-L432)
- `bitset_bytes = max(1 MiB, min(maintenance_work_mem_kB * 1024, total_elems * 2))`, then rounded **down** to a power of two in bits, capped at 2^32 bits. [bloomfilter.c#bloom_create](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L95-L111) [bloomfilter.c#my_bloom_power](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L196-L222)
- `k = clamp(rint(ln 2 * bits / total_elems), 1, 10)` hash functions. [bloomfilter.c#optimal_k](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L224-L235)
- `maintenance_work_mem` defaults to 65536 kB (64 MiB). [guc.c#maintenance_work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2244-L2252)

**Calculation A: a 10,000-block index with `reltuples` = 1,000,000, default memory.**

1. `MaxIndexTuplesPerPage = (8192 - 24) / (16 + 4) = 8168 / 20 = 408` (integer division).
2. `408 / 5 = 81`; `10,000 * 81 = 810,000`; `total_elems = max(810,000, 1,000,000) = 1,000,000`.
3. `min(65,536 * 1024, 2,000,000) = 2,000,000` bytes; `max(1,048,576, 2,000,000) = 2,000,000` bytes = 16,000,000 bits.
4. The largest power of two not above 16,000,000 is 2^23 = 8,388,608 bits = 1,048,576 bytes.
5. `k = rint(0.6931 * 8,388,608 / 1,000,000) = rint(5.81) = 6`.

**Result A:** the filter gets about 1.05 bytes per expected entry, not the 2 bytes the code aims for, because of the power-of-two rounding. The source comment says that even after such a drop the false-positive rate "won't exceed 2% in almost all cases". [bloomfilter.c#rounding-comment](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L95-L101)

**Calculation B: a 100,000,000-row index, default memory.** Assume the index has fewer than 1,234,568 blocks, so `reltuples` decides `total_elems` = 100,000,000.

1. `min(67,108,864, 200,000,000) = 67,108,864` bytes = 536,870,912 bits = 2^29 exactly.
2. Bytes per expected entry: `67,108,864 / 100,000,000 = 0.67`.
3. `k = rint(0.6931 * 5.369) = rint(3.72) = 4`.

**Result B:** memory, not the 2-bytes target, now limits the filter, at a third of the target. The docs say the miss probability "slowly increases" as memory per row falls below about 2 bytes. The pinned source does not compute the rate for this case. [amcheck.sgml#bloom-memory](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L195-L207)

**What raising the setting buys in case B.** The bitset can never exceed `total_elems * 2` = 200,000,000 bytes, and rounding down makes the largest reachable size 2^30 bits = 134,217,728 bytes (1.34 bytes per entry). That size is reached once `maintenance_work_mem * 1024 * 8 >= 2^30`, that is, at 131,072 kB (128 MiB). Any larger value changes nothing for this index. `maintenance_work_mem` has context `user`, so `SET maintenance_work_mem = '128MB'` in the checking session is enough; it needs no reload or restart. [guc.c#maintenance_work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2244-L2252)

`reltuples` is a stored estimate. If it is far too low and the index is small in blocks, `total_elems` underestimates the real entry count, so the real bytes per entry are lower than computed. The source comment says the filter copes with estimates off "by perhaps a factor of five or more". [bloomfilter.c#estimate-tolerance](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L65-L69)

The seed comes from `random()`, so each run hashes differently. A missing entry that escaped one run therefore gets a fresh chance in the next. [verify_nbtree.c#seed](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L429-L430) [bloomfilter.c#seed](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L80-L85) [amcheck.sgml#retry-chance](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L205-L207)

### How data checksums work in PostgreSQL 12

**Default, cause and mechanism.** By default a new cluster has no checksums: `initdb`'s flag starts false and only `-k` / `--data-checksums` sets it. `pg_checksums --enable` can also turn them on, but only for a cleanly shut-down cluster. The choice is recorded as `data_checksum_version` in the control file, and `DataChecksumsEnabled()` reads it. The read-only `data_checksums` setting reports the state and has context `internal`, so no configuration change sets it. [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [initdb.c:3147](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3147) [initdb.sgml#data-checksums](../../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L225) [pg_checksums.c#shutdown-required](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L513-L521) [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826) [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1827-L1835)

```text
WRITE  dirty buffer -> FlushBuffer -> PageSetChecksumCopy:
         PageIsNew (pd_upper == 0) or checksums off? write as is
         else copy page, compute pg_checksum_page(copy, blockno), store in pd_checksum, write copy
READ   cache miss -> mdread (short read -> ERROR) -> PageIsVerified(page, blockno):
         page not "new" (pd_upper != 0):
           checksums on? recompute; mismatch -> checksum_failure
           header sane (flags, pd_lower <= pd_upper <= pd_special <= BLCKSZ, aligned)?
           sane and no failure -> accept
         every byte zero? -> accept (no checksum compared)
         checksum_failure -> WARNING "page verification failed ..." and count it
           header sane and ignore_checksum_failure on? -> accept
         -> reject: ERROR "invalid page in block ..." (or zero the page if zero_damaged_pages is on)
HIT    page already in shared buffers -> returned without any verification
```

- Write path: [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750), [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189).
- Read path: [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162), [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925).
- Hit path: [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796).

Three properties of the checksum matter for amcheck:

1. **It covers the whole page and its block number.** `pg_checksum_page()` hashes the header, apart from the checksum field, and all page data, then mixes in the block number "to detect transposed pages". No relation identifier is mixed in. A page written to the wrong block of its own relation therefore fails, but a valid page from another relation at the same block number would pass. [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214)
2. **It is a 16-bit value.** The result is `(checksum % 65535) + 1`, one of 65,535 values. Random damage can therefore produce a matching checksum by chance. [checksum_impl.h#reduce](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214)
3. **It vouches for what was written, not for what should have been written.** The checksum is computed from the buffer's contents at flush time. If those contents were already wrong in memory, the wrong page gets a valid checksum. [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750)

Checksums also change [WAL](../../../glossary.md#wal). With checksums on, `XLogHintBitIsNeeded()` is true. For a permanent shared buffer outside recovery, `MarkBufferDirtyHint()` can then log a [full-page image](../../../glossary.md#full-page-image) when a [hint-bit](../../../glossary.md#hint-bits) update dirties the page and its [LSN](../../../glossary.md#lsn) is at or before the current redo point. That image protects the subsequent write against a [torn page](../../../glossary.md#torn-page). The LSN and dirty-buffer gates avoid generating an image for every hint update. [xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192) [bufmgr.c#hint-wal-gates](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3443-L3510) [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L895-L957)

This is not a promise for every relation with checksums. Local buffers return through `MarkLocalBufferDirty()`. A shared buffer without `BM_PERMANENT` bypasses this WAL path, and recovery returns without dirtying a permanent page merely for its hint update. [bufmgr.c#local-hints](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3422-L3434) [bufmgr.c#permanent-and-recovery-gates](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3481)

### How checksums change amcheck's limits

amcheck does not compute checksums itself. The only link is that amcheck reads pages through the buffer manager, and the buffer manager verifies a page whenever it reads it from disk. Each amcheck read therefore doubles as a checksum test, if the page was not already cached. The docs say so directly. [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2348-L2349) [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L377-L379) [amcheck.sgml#checksum-error](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L280-L293)

| Damage | Checksums off (default) | Checksums on |
|---|---|---|
| Bytes amcheck does not interpret on a **heap** page read from disk: non-indexed columns, row headers, free space | Not detected, unless the page header itself fails the sanity test. [bufpage.c#header-sanity](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L105-L119) | Detected **only with `heapallindexed = true`**, which is the only option that reads the heap. The read fails with `invalid page in block ...`. [verify_nbtree.c#heap-phase](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L556-L614) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) |
| Bytes amcheck does not interpret on a reached **index** page read from disk: free space, page LSN, unused header fields | Not detected. | Detected when the page is read. |
| Bytes amcheck does interpret, such as keys, links and lengths | Detected when an invariant breaks; not when the damage leaves every invariant true. | Detected at read, before any invariant runs; the error names the block instead of the invariant. |
| Damage under a clean copy in shared buffers | Not seen. | Not seen either: a buffer hit skips verification. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) |
| Pages outside the explicit index/heap traversal and any incidental accesses, including TOAST values unused by entry formation, predicates or expressions | No check from this call. | Enabling checksums does not add reads. Offline `pg_checksums --check`, or a [base backup](../../../glossary.md#base-backup), offers broader file coverage, with the exceptions below. |
| All-zero page | Index: reached page fails `_bt_checkpage()`. Heap: rows vanish silently. [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L698-L705) | Same. An all-zero page is accepted without a checksum comparison. [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) |
| File truncated below a requested block | Short-read `ERROR`. [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) | Same; the short read fails before verification. A truncated heap tail can still escape the scan. |
| Page written to the wrong block of its relation | Index: may fail level, link or order checks. Heap: only through the Bloom probe. | Detected at read, because the block number is in the checksum. [checksum_impl.h#blkno](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L207-L208) |
| Stale but valid old page image (lost write), or a valid page of another relation at the same block number | Index: may break invariants. Heap: rows that the index lacks can fail the probe. | Same as off: the old image carries its own valid checksum, and no relation identity is checked. |
| Damage made in memory before the write, or a software bug writing wrong content | Detectable by invariants and the probe. | Same: the checksum is computed over the wrong content. [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) |
| Collation or operator-class change | Detected by ordering checks. | Same; checksums are irrelevant here. [amcheck.sgml#collation](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L222-L250) |

The table shows the main gain. Without checksums, the membership test compares normalized index entries, with incidental checks while reading the heap and evaluating predicates or expressions. With checksums, each initialized heap page read from disk also gets a whole-page checksum comparison. A completed `heapallindexed` scan therefore adds checksum coverage of the table's main fork, excluding cached and all-zero pages and subject to checksum collisions and the override settings below. A first error prevents completion. [heapam_handler.c#predicate-and-expressions](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L351-L379) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162)

The amcheck docs make the complementary point: amcheck catches failure modes that checksums "will always fail to catch", and it helps with storage faults precisely when checksums are not enabled. [amcheck.sgml#effective-use](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L215-L219) [amcheck.sgml#checksum-error](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L280-L285)

**Worked calculation: chance that random damage keeps a valid checksum.** The checksum has 65,535 possible values. If damage changed a page in a way that makes the recomputed value behave as a uniformly random choice, the chance of matching the stored value is 1 in 65,535, about 0.0015%. This is a calculation from the output range, not a measured rate. [checksum_impl.h#reduce](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L210-L214)

**Reading the pages amcheck skips.** Two tools in the pinned tree read whole files instead of following links:

| Tool | When it runs | What it reads | What it skips |
|---|---|---|---|
| `pg_checksums --check` | Only on a cleanly shut-down cluster that has checksums. [pg_checksums.c#preconditions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L513-L535) | Eligible regular files under `global`, `base` and `pg_tblspc` (or the single `-f` / `--filenode` match when requested). [pg_checksums.c#directories](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L545-L561) [pg_checksums.c#filenode-filter](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L353-L355) | The `skip[]` file list (`pg_control`, `pg_filenode.map`, `pg_internal.init`, `PG_VERSION`, and on `EXEC_BACKEND` builds `config_exec_params*`), temporary files and directories prefixed `pgsql_tmp`, and within scanned files any page whose `pd_upper` is 0. [pg_checksums.c#skip-list](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L95-L110) [pg_checksums.c#temporary-exclusions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L301-L311) [pg_checksums.c#scan_file](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L192-L225) |
| Base backup (`BASE_BACKUP`, used by `pg_basebackup`) | On a running server with checksums, unless `NOVERIFY_CHECKSUMS` is given. [basebackup.c#verify-gate](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1413-L1430) [protocol.sgml#NOVERIFY_CHECKSUMS](../../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2569-L2577) | Eligible relation files under `global`, `base` and tablespaces, after file-selection and checksum exclusions. [basebackup.c#is_checksummed_file](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1343-L1362) [basebackup.c#noChecksumFiles](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L208-L218) | Temporary relations and every non-init fork of unlogged relations are excluded before their contents are read. [basebackup.c#relation-exclusions](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1162) In included files, it skips pages whose `pd_upper` is 0 and pages whose LSN is at or after the backup start. A read length not divisible by `BLCKSZ` warns and disables verification for that read and the rest of that file. Otherwise a mismatch is retried, and confirmed mismatches fail the backup. [basebackup.c#unaligned-read](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1454-L1466) [basebackup.c#skip-and-retry](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1470-L1556) [basebackup.c#final-error](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L621-L633) |

Ordinary tables default to permanent storage. Explicit `UNLOGGED` selects an [unlogged relation](../../../glossary.md#unlogged-relation), whose initialization fork supplies the backup's empty starting contents. `sendDir()` finds that init file and skips the relation's other forks; a successful backup therefore does not check the unlogged table's live heap or index contents. Temporary relations are excluded separately by filename. These file exclusions are independent of the page-level checksum exceptions. [gram.y#persistence-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283) [basebackup.c#relation-exclusions](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1162) [heapam_handler.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624)

Both tools skip a page by testing only `PageIsNew()`, which is `pd_upper == 0`. The server's `PageIsVerified()` rejects a page whose `pd_upper` is 0 but whose other bytes are not all zero. Such a page therefore passes both tools and fails when the server reads it. This is a reading of the three code paths, not a measured result. [bufpage.h#PageIsNew](../../../../raw/postgres-12/src/include/storage/bufpage.h#L229) [pg_checksums.c#scan_file](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L210-L213) [basebackup.c#skip-and-retry](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1478) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L162)

`pg_stat_database.checksum_failures` is not a complete ledger of errors. `PageIsVerified()` requests a counter update, but `pgstat_report_checksum_failures_in_db()` returns without sending it if the statistics socket is unavailable or `track_counts` is off. `track_counts` defaults on and has superuser session scope. At this pin the base-backup path has another exception: it reports a file's count only when that count is greater than one. One confirmed mismatch still fails the backup, but is not sent to the collector. A “could not verify checksum” warning from an unaligned read is not a confirmed mismatch either. Read the error and backup outcome as well as the counter. [bufpage.c:155](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L155) [pgstat.c#checksum-reporting](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556) [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1399) [basebackup.c#per-file-report](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1611-L1622) [basebackup.c#unaligned-read](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1454-L1466)

### Non-default settings and special cases

| Condition | Default | Cause and mechanism | Effect on amcheck |
|---|---|---|---|
| `zero_damaged_pages = on` | Off. [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161) | A superuser sets it (`PGC_SUSET`, session or transaction scope with `SET`). `ReadBuffer_common()` then zeroes a page that fails verification and only warns, and `mdread()` returns zeros for a short read. [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L908-L918) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L615-L624) | A damaged **index** page becomes a zero page, which amcheck still rejects. A damaged **heap** page loses all its rows from the scan, so `heapallindexed` can pass with only a `WARNING` in the log. The zeroed buffer is then marked valid, so later reads hit it in shared buffers. [bufmgr.c#mark-valid](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L945-L955) [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L698-L705) |
| `ignore_checksum_failure = on` | Off. [guc.c#ignore_checksum_failure](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1148) | A superuser sets it (`PGC_SUSET`, session or transaction scope with `SET`). `PageIsVerified()` still emits `WARNING: page verification failed ...` (`SQLSTATE` `01000`) and reports the failure to the statistics collector, then accepts the page if its header looks sane. [bufpage.c:148-158](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L148-L158) [elog.c#default-errcode](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388) | The read succeeds after the warning, and amcheck examines the damaged page, so the call raises an error only if an invariant or probe fails. |
| [Hot standby](../../../glossary.md#hot-standby) | A primary is not in recovery (`RecoveryInProgress()` is false). [xlog.c#RecoveryInProgress](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7913-L7957) | During recovery a backend cannot take a relation lock above `RowExclusiveLock`. [lock.c#recovery-lock-limit](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766) | `bt_index_parent_check` fails to lock; `bt_index_check` works. The docs note that collation mismatches between primary and standby can usually be seen only on the standby. [amcheck.sgml#standby](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L161-L164) [amcheck.sgml#collation](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L234-L239) |
| Unlogged index during recovery | By default, tables and their inherited indexes are permanent (`RELPERSISTENCE_PERMANENT`, `'p'`) and a primary is not in recovery, so `btree_index_mainfork_expected()` returns true and the main fork is required. [gram.y#persistence-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283) [index.c#relpersistence](../../../../raw/postgres-12/src/backend/catalog/index.c#L744-L749) [verify_nbtree.c:279-294](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L279-L294) | Explicit `UNLOGGED` sets `RELPERSISTENCE_UNLOGGED` (`'u'`) on the table and its indexes, and a standby has `RecoveryInProgress()` true, where such an index has no main-fork contents. [verify_nbtree.c#btree_index_mainfork_expected](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L340-L360) | `NOTICE` and skip; no check runs. |
| `REPEATABLE READ` or `SERIALIZABLE` transaction | `READ COMMITTED`. [guc.c#default_transaction_isolation](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4255-L4262) | `GetTransactionSnapshot()` can return the transaction's older snapshot, which may predate an index whose [`indcheckxmin`](../../../glossary.md#indcheckxmin) is set. [xact.h#IsolationUsesXactSnapshot](../../../../raw/postgres-12/src/include/access/xact.h#L51) | `bt_index_check(idx, true)` may fail with `cannot be verified using transaction snapshot` (`40001`). [verify_nbtree.c#indcheckxmin](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L454-L468) |
| Index from a pre-version-4 B-tree (via `pg_upgrade`) | A v12 build writes version 4. [nbtree.h#BTREE_VERSION](../../../../raw/postgres-12/src/include/access/nbtree.h#L133-L135) | `_bt_heapkeyspace()` reports false. | Weaker ordering checks; `rootdescend` errors with `Only B-Tree version 4 indexes support rootdescend verification.` [verify_nbtree.c#version-gate](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L486-L492) |

### Lifecycle events

| Event | State before | State after | Effect on a later amcheck call |
|---|---|---|---|
| A page is read into shared buffers | On disk only | Cached, verified once at read time | Later calls see the cached copy; disk changes beneath it are invisible until eviction. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) |
| A dirty page is flushed | Cached, modified | With checksums on and `pd_upper != 0`, on disk with a checksum computed now. `PageIsNew()` pages (`pd_upper == 0`) are written unchanged | A later complete disk read compares the checksum only for initialized pages. A wholly zero page is accepted; a nonzero page with `pd_upper == 0` fails generic verification under default settings. [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750) [bufpage.c#write-gate](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [bufpage.c#read-gate](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L162) |
| `initdb -k`, or `pg_checksums --enable` on a stopped cluster | `data_checksum_version` = 0 | > 0; `--enable` writes a checksum into each scanned page whose `pd_upper` is not 0 | Later complete reads of initialized pages compare checksums; cached and all-zero pages keep their exceptions. [pg_checksums.c#scan_file](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L215-L240) [pg_checksums.c#control-update](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L587-L590) [bufpage.c#verify-gate](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L142) |
| A crash during the write of a permanent page protected by the hint-update FPI path above | Page partly written | Recovery can restore the WAL image | Protection requires the permanent-buffer, non-recovery and LSN conditions described above. [bufmgr.c#hint-wal-gates](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3470-L3510) [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L895-L957) |
| `VACUUM` deletes B-tree pages | Linked pages | Unlinked deleted pages | The walk no longer reaches them, so their contents are never checked. [verify_nbtree.c#vacuum-deletion](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1327-L1333) [verify_nbtree.c#deleted-pages-unlinked](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L686-L697) |
| The index's `pg_class.reltuples` estimate or block count changes | Old values | New values | Changes the Bloom filter's size on the next call. [verify_nbtree.c#bloom-sizing](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L426-L428) |

### Checking a non-temporary physical table

Because one call checks one index, checking a physical table calls amcheck once per eligible B-tree index. Each `heapallindexed` call scans that table again. The query below is a template for one non-temporary physical table or [materialized view](../../../glossary.md#materialized-view). It does not enumerate inheritance or partition descendants; supply each physical leaf separately. A [partitioned index](../../../glossary.md#partitioned-index) has `relkind = 'I'` and is excluded by the `relkind = 'i'` filter. A temporary target is also excluded even though amcheck accepts this session's temporary indexes. Zero result rows mean that this query checked nothing. [amcheck.sgml#example](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L57-L71) [pg_index.h#indrelid](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L42) [pg_class.h#relation-kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) [verify_nbtree.c#temporary-target](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L325-L330)

```sql
SET /* wiki_amcheck_session_statement_timeout */ statement_timeout = '30min';
SET /* wiki_amcheck_session_lock_timeout */ lock_timeout = '5s';
SET /* wiki_amcheck_session_bloom_memory */ maintenance_work_mem = '128MB';
SELECT /* wiki_amcheck_table_btree_indexes */
       c.relname,
       bt_index_check(c.oid, true)
FROM pg_index i
JOIN pg_class c ON c.oid = i.indexrelid
JOIN pg_am am ON am.oid = c.relam
WHERE i.indrelid = 'public.my_table'::regclass
  AND am.amname = 'btree'
  AND c.relkind = 'i'
  AND c.relpersistence <> 't'
  AND i.indisvalid
  AND i.indisready
ORDER BY c.relname;
```

- [`statement_timeout`, `lock_timeout`](../../../glossary.md#statement_timeout-and-lock_timeout) and `maintenance_work_mem` all have context `user`, so `SET` applies them to this session only. [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396) [guc.c#maintenance_work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2244-L2252)
- The columns exist at the pin: `pg_index.indexrelid`, `indrelid`, `indisvalid`, `indisready`; `pg_class.relam`, `relpersistence`, `relkind`; `pg_am.amname`. `oid` casts implicitly to `regclass`. [pg_index.h#columns](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L42) [pg_class.h#columns](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L50-L81) [pg_am.h#amname](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L34) [pg_cast.dat#oid-regclass](../../../../raw/postgres-12/src/include/catalog/pg_cast.dat#L178-L179)
- The first error aborts the whole statement, so later indexes in the list go unchecked. Run one statement per index to see every failure.
- The membership comparison covers the normalized entry formed for each qualifying row; predicates and expressions can access additional values. Checksums can add whole-page verification for initialized heap pages fetched from disk, with the limitations above. [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) [heapam_handler.c#predicate-and-expressions](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162)

### Build boundary and tests

amcheck is a [contrib](../../../glossary.md#contrib) extension built from `contrib/amcheck`, default version 1.2. The checksum code is in core: `checksum_impl.h` is a header that external programs may include, and both the server and `pg_checksums` call `pg_checksum_page()`. [amcheck.control](../../../../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L5) [checksum_impl.h#header](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L1-L17) [pg_checksums.c#scan_file](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L215)

The module's regression test covers permissions, table and BRIN rejection, both functions with and without `heapallindexed`, an `INCLUDE` index with `rootdescend`, multi-level page deletion and the TOAST normalization bug #15597. It injects no corruption, so no upstream test at this pin exercises a corruption error path or checksum interaction through amcheck. [check_btree.sql#tests](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L1-L114) [Makefile#REGRESS](../../../../raw/postgres-12/contrib/amcheck/Makefile#L10)

### Causal summary

**Index pages reached by links → read through shared buffers (checksum-verified only on a disk read, when the cluster has checksums) → structural and ordering invariants, plus parent/child checks under `ShareLock` → optional fingerprint of leaf entries → optional heap scan that probes the fingerprint for each row that belongs in the index → first violation raises `ERROR`, otherwise silence.** Checksums widen what that same read path catches to every byte of every initialized page read from disk, unless the damage happens to match the 16-bit checksum or `ignore_checksum_failure` is on. They cannot reach cached pages, unread pages, all-zero pages, truncated files or damage written with a valid checksum. amcheck in turn catches ordering, cross-structure and in-memory damage that checksums can never see.

## Measurement Script

### Results and last run

**Last successful run:** 2026-10-25, PostgreSQL 12.2, pin `45b88269a353ad93744772791feb6d01bc7e1e42`, Linux 6.18.54.1-microsoft-standard-WSL2 x86_64, GCC 15.2.0 with `CFLAGS='-O2 -std=gnu99'`. Both clusters used 8192-byte blocks, maximum alignment 8, UTF-8, C locale, session TimeZone UTC, `shared_buffers = 128MB` and `maintenance_work_mem = 64MB`. One cluster used the default checksums-off state; the other used `initdb -k`. The successful `verify` run completed in 4 seconds and passed all **39 recorded checks**. The fresh core/extension build took 34 seconds. The separate `interrupt` stage passed both startup-shutdown tests in 1 second. These are measurements of this host and these small fixtures, not performance estimates for production indexes.

| Fixture / condition | Measured outcome | Source interpretation |
|---|---|---|
| Delete one of 20 rows, then force an ordinary index scan of the deleted key | `bt_page_stats()` found 1 dead index item; `bt_index_parent_check(..., true, true)` passed with checksums off and on | `LP_DEAD` with storage is valid; it is excluded from the tuple fingerprint. [nbtutils.c#killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1781-L1811) [verify_nbtree.c#line-pointer](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2531-L2548) [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083) |
| Apply the example's inventory filter to a partitioned parent and this session's temporary table | Parent: 0 matching physical indexes, although its leaf had 1. Temporary table: 0 matches, although a direct amcheck call passed. Same results in both clusters | The query's scope restrictions are real exclusions, not evidence that the excluded objects were checked. [pg_index.h#indrelid](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L42) [pg_class.h#kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) [verify_nbtree.c#temp-target](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L325-L330) |
| Pass a table as the target | `42809`, `"toast_probe" is not an index`, in both clusters | `index_open()` fails before amcheck's B-tree-specific test. [indexam.c#index_open](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L126-L140) |
| One row has a 20,000-byte external `payload`; both its plain `id` index and its partial `id` index initially work. Stop the server and truncate only its TOAST heap file to zero bytes | With `heapallindexed = true`, the plain index still passed. The partial index with `WHERE payload LIKE 'a%'` failed with `XX001`, `could not read block 0 ... read only 0 of 8192 bytes`, in both clusters | The predicate fetches TOAST data before forming the entry. This particular fixture fails in `mdread()` before the TOAST reader can report a missing chunk. It is not a measured `XX000` missing-chunk case. [heapam_handler.c#predicate](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) [like.c#textlike](../../../../raw/postgres-12/src/backend/utils/adt/like.c#L284-L300) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) |
| Stop the server and change byte 512, verified to lie in free space, on a heap page containing 10 rows | An index-only amcheck call passed in both clusters. The heap-checking call passed with checksums off and failed with `XX001`, `invalid page`, with checksums on | The non-indexed free byte changes no membership entry. A cold heap read with checksums on rejects its checksum mismatch. [verify_nbtree.c#heap-gate](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L614) [bufpage.c#verification](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L161) |
| Read that bad checksum page with `track_counts = off`, then with its default on state | Both reads failed. Counter: initially 0, still 0 after the off leg, then 1 after the on leg | Reporting is gated independently of page rejection. The harness waits for the collector after the enabled report. [pgstat.c#report-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556) |
| Base backup with exactly that one checksum-mismatching page | Backup failed; the client reported `checksum error occurred`. The database counter remained 1, its value from the earlier backend read | This pin sends a file's backup mismatch count to the collector only when it exceeds 1. The backend still errors for one mismatch; the client translates that checksum failure. [basebackup.c#per-file-report](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1611-L1622) [basebackup.c#final-error](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L621-L635) [pg_basebackup.c#checksum-error](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L2025-L2036) |
| Restore that page, then damage the free byte of another heap's first page and append one byte while stopped | The 8,193-byte file was copied by a successful base backup, with `could not verify checksum ... read buffer size 8193 and page size 8192 differ` | The unaligned read disables verification before testing even the complete first page in that read. Success with this warning does not establish checksum coverage or a sound backup. The fixture is not a restore test. [basebackup.c#unaligned-read](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1454-L1466) |
| Include a populated `UNLOGGED` table in that successful backup's source cluster | Its live main fork was absent from the backup; its init fork was present | File selection excludes non-init unlogged forks before checksum verification. This establishes the file exclusion, not detection of damage in the excluded heap. [basebackup.c#unlogged-exclusion](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1152) |
| Signal the runner after `pg_ctl start` returns but before `owned=1`, once with `TERM` and once with `INT` | Child exit codes were 143 and 130; both left no PID file, data-directory process, private socket/lock file or occupied port | The deterministic signal hook exercises the pending-ownership boundary. The EXIT trap reconciles that state and stops the newly launched server; this does not test every possible startup failure |

The final verification leg restores the second file and confirms that the original checksum fixture passes again. Each fault is applied only while the owned server is stopped; subsequent starts discard shared-buffer state, so these results do not measure cached-page masking. The script stops each cluster before returning from `verify`. The following `interrupt` stage replaces the disposable cluster for each signal test and stops each launched server through its EXIT trap. The task then ran `cleanup`, confirmed no PID file, data-directory process or socket and a free port, and deleted the sandbox, including the build.

### Usage

Run from the repository root. Extract the single Bash block below, then choose stages:

```sh
mkdir -p .wiki-runtime/tmp
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/indexing/amcheck-indexes-tables-and-data-checksums.md |
    sed '1d;$d' > .wiki-runtime/tmp/amr12-run.sh
JOBS=16 bash .wiki-runtime/tmp/amr12-run.sh build verify interrupt
cat .wiki-runtime/tmp/amr12/results.tsv
cat .wiki-runtime/tmp/amr12/startup-interruptions.tsv
bash .wiki-runtime/tmp/amr12-run.sh cleanup
rm .wiki-runtime/tmp/amr12-run.sh
```

| Item | Contract |
|---|---|
| Purpose | Reproduce the measured cases above on disposable fixtures; no production target is accepted |
| Stages | `build`: verify the pin and build/install core, amcheck and [`pageinspect`](../../../glossary.md#pageinspect) out of tree; reuse an existing owned build. `verify`: replace stopped fixtures, run the checksums-off and checksums-on legs, and stop each server. `interrupt`: run two child invocations that signal themselves after server launch but before confirming ownership, then assert exit status and complete shutdown. `cleanup`: refuse a running sandbox, confirm teardown, then delete it. With no arguments, run `build verify interrupt cleanup` |
| Environment | `JOBS`, default `8`, is the make parallelism. `AMR12_TEST_STARTUP_SIGNAL`, default empty, accepts `INT` or `TERM` only; `interrupt` sets it for its child invocations to exercise pending-startup cleanup. Leave it empty for measurements. Run from the repository root with Bash supporting `/dev/tcp`; `LC_ALL` is set to `C`. The script resolves its own path from `$0` for `interrupt` child invocations. Paths, role `amr12`, database `postgres`, private socket and port label `55448` are fixed by the script. It clears inherited `PGHOSTADDR`, `PGSERVICE`, `PGSERVICEFILE` and `PGOPTIONS`, and supplies connection arguments explicitly |
| Prerequisites | Git, Bash, GNU make, C compiler with GNU C99 mode, the Git-source build prerequisites including Bison/Flex and Perl, and standard POSIX command-line utilities (`ps`, `grep`, `sed`, `wc`, `tr`, `head`, `dd`). No network fetch or separately installed PostgreSQL is used. [installation.sgml#build-tools](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L80-L97) [installation.sgml#git-build](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L300-L318) |
| Build | `--without-readline --without-zlib`, private prefix, `CFLAGS='-O2 -std=gnu99'`. Optional editing/compression libraries are omitted; C99 mode avoids modern compilers treating this pin's `bool` typedef as a C23 keyword conflict. [installation.sgml#optional-libraries](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L118-L151) [configure.in#default-cflags](../../../../raw/postgres-12/configure.in#L408-L415) [c.h#bool](../../../../raw/postgres-12/src/include/c.h#L302-L309) |
| Output | Read `.wiki-runtime/tmp/amr12/results.tsv` first; `startup-interruptions.tsv` records the separate shutdown tests. Per-check `.out`/`.err` files, `platform-off.txt`, `platform-on.txt`, `host.txt`, build logs and `server.log` give detail. `verify` replaces the current result summary and fixtures. `interrupt` saves the measurement summary while its child invocations replace the disposable cluster; it also removes the prior backup copy. `cleanup` deletes all artifacts |
| Runtime | About a minute for a fresh build and this small suite on the measured host; a few seconds for `verify` with the build retained. A run after `cleanup` rebuilds |
| Cleanup | Every successful `verify` stops its servers. A pending-startup flag is set before `pg_ctl` runs, so the EXIT trap reconciles both completed and interrupted launches. It waits briefly for a PID file if an owned startup process still exists, then stops the server and checks teardown. `cleanup` removes `.wiki-runtime/tmp/amr12/` only after ownership and teardown checks. If teardown fails, it preserves the sandbox and reports the error. It never stops a pre-existing server |

The pending flag matters because `pg_ctl` starts a separate postmaster and, where supported, detaches it from the launching process group. A startup timeout also exits the launcher without proving that the postmaster exited. The harness therefore marks the launch pending before calling `pg_ctl`, and clears that state only after ownership is confirmed or shutdown is verified. [pg_ctl.c#detached-launch](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L449-L486) [pg_ctl.c#startup-timeout](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L887-L924)

### Settings and fixtures

| Change from the normal/default state | Scope and reason | Results affected |
|---|---|---|
| Checksums off → on with `initdb -k` in the second fresh cluster | Cluster creation; `data_checksums` is internal, not settable with `SET`. The first leg measures the default. [initdb.c#default](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1827-L1835) | Free-space-damage rejection, counters and checksum-verifying backups |
| No installed extension → amcheck and pageinspect in each disposable database | The fixture explicitly installs both; amcheck provides checks, pageinspect observes dead items and validates the fault's free-space offset. [amcheck.control](../../../../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L5) [pageinspect--1.5.sql#page_header](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L20-L33) [pageinspect--1.5.sql#bt_page_stats](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L160-L175) | All amcheck/inspection results |
| `autovacuum` on → off | `sighup` setting supplied in the fresh server configuration; keeps the deleted entry available for inspection. Manual deletion and the index scan create the measured `LP_DEAD` state. [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433) [nbtutils.c#killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1781-L1811) | Dead-item count and valid-dead-item check |
| `enable_seqscan`, `enable_bitmapscan`, `enable_indexonlyscan` on → off in the killed-entry probe session | All `user` settings; session `SET` directs the probe through an ordinary index scan so heap liveness can request a killed-entry hint. The other sessions use defaults. [guc.c#scan-switches](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L884-L921) [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L588) | Dead-item fixture only |
| `track_counts` on → off for one failing read | `superuser` session setting; disables reporting without suppressing the checksum error. [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1399) [pgstat.c#report-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556) | Counter remains unchanged in the off leg |
| Text's normal extended storage → `STORAGE EXTERNAL` for `toast_probe.payload` | Column storage option; forces this large, compressible fixture out of line instead of allowing inline compression. The predicate must then fetch its chunks. [tuptoaster.c#compression-and-external-storage](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L718-L800) | TOAST-predicate result |
| Permanent table → `CREATE UNLOGGED TABLE unlogged_probe` | Default persistence is permanent. The explicit option creates an init fork; backup file selection excludes the other forks when that init file exists. [gram.y#persistence-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283) [heapam_handler.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624) [basebackup.c#unlogged-exclusion](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1152) | Successful backup contains the init fork but no live main fork for this table |
| Intact files → stopped-server truncation, free-byte overwrite or appended partial block | Deliberate corruption fixtures; ordinary SQL maintains the index/heap and page-format invariants. Restart makes subsequent reads cold. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) [basebackup.c#unaligned-read](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1454-L1466) | TOAST, checksum and backup fault cases; these are not normal table states |
| `statement_timeout = 0`, `lock_timeout = 0` → `2min`, `5s` | `user` session settings supplied on every SQL connection; bound individual statements and lock waits. No measured check hit either limit. [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396) | SQL failure bounds |
| `listen_addresses = 'localhost'` → `''`, `unix_socket_directories = '/tmp'` → private `$work/s`, `unix_socket_permissions = 0777` → `0700`, `port = 5432` → `55448`, `TimeZone` (boot `'GMT'`; `initdb` writes the host zone into `postgresql.conf`) → `'UTC'`, plus `initdb`'s `128MB` `shared_buffers` override (boot default `1024` blocks = 8 MB) | Listener, socket, port and `shared_buffers` settings are `postmaster` settings chosen before start; `TimeZone` is a `user` setting appended to `postgresql.conf` for new sessions. They isolate the fixtures and format time consistently. [guc.c#shared_buffers-and-port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2199) [guc.c#timezone-and-listeners](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3905-L3960) [pg_config_manual.h#DEFAULT_PGSOCKET_DIR](../../../../raw/postgres-12/src/include/pg_config_manual.h#L182) [configure.in#default_port](../../../../raw/postgres-12/configure.in#L149-L164) [initdb.c#shared_buffers-and-timezone](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1018-L1102) [initdb.c#timezone-config](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1165-L1173) | Connection isolation and platform record; not detection semantics |
| Base-backup command uses `-X none -c fast -N` | No WAL transfer, fast [checkpoint](../../../glossary.md#checkpoint), no destination sync wait. These speed a disposable verification probe; the result is not a restore-ready backup. None requests `NOVERIFY_CHECKSUMS`. [pg_basebackup.c#options](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L330-L353) | Both backup tests |

### Script

```bash
#!/usr/bin/env bash
set -uo pipefail
umask 077
export LC_ALL=C
unset PGHOSTADDR PGSERVICE PGSERVICEFILE PGOPTIONS

root=$(pwd -P)
case "$0" in
    /*) runner=$0 ;;
    *) runner="$root/$0" ;;
esac
src="$root/raw/postgres-12"
work="$root/.wiki-runtime/tmp/amr12"
data="$work/data"
sock="$work/s"
bin="$work/install/bin"
port=55448
pin=45b88269a353ad93744772791feb6d01bc7e1e42
jobs=${JOBS:-8}
owned=0
start_pending=0
startup_signal=${AMR12_TEST_STARTUP_SIGNAL:-}
checks=0

die() { printf '%s\n' "$*" >&2; exit 1; }
case "$jobs" in ''|*[!0-9]*|0) die "JOBS must be a positive integer";; esac
case "$startup_signal" in ''|INT|TERM) ;; *) die "Invalid startup test signal";; esac
test -f "$root/AGENTS.md" && test -f "$src/configure" ||
    die "Run from the repository root"
test "$(git -C "$src" rev-parse HEAD)" = "$pin" || die "Wrong source pin"

port_free() {
    ! ( : > "/dev/tcp/127.0.0.1/$port" ) 2>/dev/null
}
stopped() {
    test ! -e "$data/postmaster.pid" || return 1
    ps -A -o pid= -o args= > "$work/process-check.txt" || return 1
    if grep -F -- "$data" "$work/process-check.txt" >/dev/null; then return 1; fi
    test ! -e "$sock/.s.PGSQL.$port" || return 1
    test ! -e "$sock/.s.PGSQL.$port.lock" || return 1
    port_free
}
stop_owned() {
    if test "$owned" -eq 1 || test "$start_pending" -eq 1; then
        local i
        # A failed/interrupted launcher can leave a postmaster still starting.
        for i in {1..10}; do
            test -f "$data/postmaster.pid" && break
            stopped && break
            sleep 1
        done
        if test -f "$data/postmaster.pid"; then
            "$bin/pg_ctl" -D "$data" -m fast -w stop >> "$work/control.log" 2>&1 ||
                return 1
        fi
        stopped || return 1
        owned=0
        start_pending=0
    fi
    stopped
}
on_exit() {
    local rc=$?
    trap - EXIT
    trap '' INT TERM
    if test "$owned" -eq 1 || test "$start_pending" -eq 1; then
        stop_owned || { printf 'Owned server teardown failed: %s\n' "$data" >&2; rc=1; }
    fi
    exit "$rc"
}
trap on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

prepare() {
    if test -e "$work"; then
        test -f "$work/owner" && test "$(cat "$work/owner")" = "$root $pin" ||
            die "Refusing an unowned sandbox"
    else
        mkdir -p "$work" || die "mkdir failed"
        printf '%s %s\n' "$root" "$pin" > "$work/owner" || die "owner write failed"
    fi
    stopped || die "Sandbox or port is already in use; no service was stopped"
}
build() {
    prepare
    mkdir -p "$work/build" "$sock" || die "mkdir failed"
    if test ! -f "$work/build/Makefile"; then
        (cd "$work/build" &&
            "$src/configure" --prefix="$work/install" --without-readline --without-zlib \
            CFLAGS='-O2 -std=gnu99') > "$work/configure.log" 2>&1 ||
            die "Configure failed; see configure.log"
    fi
    make -C "$work/build" -j "$jobs" > "$work/build.log" 2>&1 &&
    make -C "$work/build" install > "$work/install.log" 2>&1 ||
        die "Core build/install failed"
    local ext
    for ext in amcheck pageinspect; do
        make -C "$work/build/contrib/$ext" -j "$jobs" >> "$work/build.log" 2>&1 &&
        make -C "$work/build/contrib/$ext" install >> "$work/install.log" 2>&1 ||
            die "Extension build/install failed: $ext"
    done
}
start() {
    stopped || die "Refusing to start over an existing service"
    start_pending=1
    "$bin/pg_ctl" -D "$data" -l "$work/server.log" -w start >> "$work/control.log" 2>&1
    local rc=$?
    # Deterministically exercise the gap before ownership confirmation.
    if test -n "$startup_signal"; then
        test "$rc" -eq 0 && test -f "$data/postmaster.pid" ||
            die "Startup must succeed before testing interruption cleanup"
        kill "-$startup_signal" "$$" || die "Cannot deliver startup test signal"
    fi
    # Even a timed-out pg_ctl can have started our postmaster.
    if test -f "$data/postmaster.pid"; then owned=1; fi
    test "$rc" -eq 0 && test "$owned" -eq 1 || die "Startup failed"
    start_pending=0
}
sql() {
    if test "$#" -eq 0; then set -- -f -; fi
    PGCONNECT_TIMEOUT=5 PGAPPNAME=wiki_amr12 "$bin/psql" \
        -X -qAt -h "$sock" -p "$port" -U amr12 -d postgres \
        -v ON_ERROR_STOP=1 -v VERBOSITY=verbose \
        -c "SET /* wiki_amr_statement_timeout */ statement_timeout = '2min'" \
        -c "SET /* wiki_amr_lock_timeout */ lock_timeout = '5s'" "$@"
}
record() {
    checks=$((checks + 1))
    printf 'PASS\t%s\n' "$1" | tee -a "$work/results.tsv" || die "Result write failed"
}
expect() {
    test "$2" = "$3" || die "$1: expected [$2], got [$3]"
    record "$1=$3"
}
ok() {
    sql -c "$2" > "$work/$1.out" 2> "$work/$1.err" ||
        die "$1 failed; see its .err file"
    record "$1"
}
bad() {
    if sql -c "$3" > "$work/$1.out" 2> "$work/$1.err"; then
        die "$1 unexpectedly succeeded"
    fi
    grep -E "$2" "$work/$1.err" >/dev/null || die "$1 returned the wrong error"
    record "$1"
}
count_checksums() {
    sql -c "SELECT /* wiki_amr_checksum_count */ coalesce(checksum_failures, 0)
            FROM pg_stat_database WHERE datname = current_database()"
}
wait_count() {
    local i value
    for i in {1..50}; do
        value=$(count_checksums) || die "Counter query failed"
        if test "$value" = "$1"; then return; fi
        sleep 0.1
    done
    die "Counter did not reach $1 (last: $value)"
}
relation_path() {
    local relpath
    relpath=$(sql -c "SELECT /* wiki_amr_relation_path */ pg_relation_filepath('$1')") ||
        die "Relation path lookup failed"
    case "$relpath" in base/*) printf '%s/%s\n' "$data" "$relpath";;
        *) die "Unexpected fixture path";; esac
}
flip_free_byte() {
    printf '\125' | dd of="$1" bs=1 seek=512 conv=notrunc 2>/dev/null ||
        die "Fault injection failed"
}
backup() {
    PGCONNECT_TIMEOUT=5 PGAPPNAME=wiki_amr12 "$bin/pg_basebackup" \
        -h "$sock" -p "$port" -U amr12 -D "$work/backup" -X none -c fast -N \
        > "$work/$1.out" 2> "$work/$1.err"
}
init_cluster() {
    local mode=$1 escaped
    stopped || die "Cannot replace a running cluster"
    rm -rf "$data" "$work/backup" || die "Cannot reset fixtures"
    local args=()
    if test "$mode" = on; then args=(-k); fi
    "$bin/initdb" -D "$data" -U amr12 -A trust --no-locale --encoding=UTF8 \
        "${args[@]}" > "$work/init-$mode.log" 2>&1 || die "initdb failed"
    mkdir -p "$sock" || die "Socket directory failed"
    escaped=${sock//\'/\'\'}
    cat >> "$data/postgresql.conf" <<EOF
listen_addresses = ''
port = $port
unix_socket_directories = '$escaped'
unix_socket_permissions = 0700
autovacuum = off
timezone = 'UTC'
EOF
    start
    sql > "$work/fixture-$mode.out" 2> "$work/fixture-$mode.err" <<'SQL'
CREATE /* wiki_amr_fixture_extension */ EXTENSION amcheck;
CREATE /* wiki_amr_fixture_inspection */ EXTENSION pageinspect;
CREATE /* wiki_amr_fixture_dead */ TABLE dead_probe(id integer PRIMARY KEY);
INSERT /* wiki_amr_fixture_dead_rows */ INTO dead_probe SELECT generate_series(1,20);
CREATE /* wiki_amr_fixture_checksum */ TABLE checksum_probe(id integer PRIMARY KEY, payload text);
INSERT /* wiki_amr_fixture_checksum_rows */ INTO checksum_probe
SELECT g, repeat('x',100) FROM generate_series(1,10) g;
CREATE /* wiki_amr_fixture_partial */ TABLE partial_probe(id integer PRIMARY KEY, payload text);
INSERT /* wiki_amr_fixture_partial_rows */ INTO partial_probe
SELECT g, repeat('x',100) FROM generate_series(1,10) g;
CREATE /* wiki_amr_fixture_unlogged */ UNLOGGED TABLE unlogged_probe(id integer);
INSERT /* wiki_amr_fixture_unlogged_rows */ INTO unlogged_probe SELECT generate_series(1,10);
CREATE /* wiki_amr_fixture_toast */ TABLE toast_probe(id integer PRIMARY KEY, payload text);
ALTER /* wiki_amr_fixture_external */ TABLE toast_probe ALTER COLUMN payload SET STORAGE EXTERNAL;
INSERT /* wiki_amr_fixture_toast_row */ INTO toast_probe VALUES (1, repeat('a',20000));
CREATE /* wiki_amr_fixture_predicate */ INDEX toast_predicate ON toast_probe(id)
WHERE payload LIKE 'a%';
CREATE /* wiki_amr_fixture_parent */ TABLE parent_probe(id integer) PARTITION BY RANGE(id);
CREATE /* wiki_amr_fixture_leaf */ TABLE leaf_probe PARTITION OF parent_probe
FOR VALUES FROM (0) TO (100);
CREATE /* wiki_amr_fixture_parent_index */ INDEX parent_probe_idx ON parent_probe(id);
SQL
    test "$?" -eq 0 || die "Fixture creation failed"
    sql -c "SELECT /* wiki_amr_platform */ version(), current_setting('block_size'),
        current_setting('data_checksums'), current_setting('shared_buffers'),
        current_setting('maintenance_work_mem'), current_setting('track_counts'),
        current_setting('server_encoding'), current_setting('lc_collate'),
        current_setting('TimeZone')" > "$work/platform-$mode.txt" || die "Platform query failed"
    "$bin/pg_controldata" "$data" >> "$work/platform-$mode.txt" || die "Control read failed"
}
common_checks() {
    local mode=$1 value
    ok "$mode-delete" "DELETE /* wiki_amr_delete */ FROM dead_probe WHERE id=1"
    sql -c "SET /* wiki_amr_force_index */ enable_seqscan=off" \
        -c "SET /* wiki_amr_no_bitmap */ enable_bitmapscan=off" \
        -c "SET /* wiki_amr_no_indexonly */ enable_indexonlyscan=off" \
        -c "SELECT /* wiki_amr_kill_entry */ count(*) FROM dead_probe WHERE id=1" \
        > "$work/$mode-kill.out" || die "Index scan failed"
    value=$(sql -c "SELECT /* wiki_amr_dead_items */ dead_items FROM bt_page_stats('dead_probe_pkey',1)") ||
        die "Page inspection failed"
    expect "$mode-LP_DEAD-items" 1 "$value"
    ok "$mode-LP_DEAD-amcheck" "SELECT /* wiki_amr_dead_check */ bt_index_parent_check('dead_probe_pkey',true,true)"
    value=$(sql -c "SELECT /* wiki_amr_parent_inventory */ count(*) FROM pg_index i
        JOIN pg_class c ON c.oid=i.indexrelid JOIN pg_am a ON a.oid=c.relam
        WHERE i.indrelid='parent_probe'::regclass AND a.amname='btree'
        AND c.relkind='i' AND c.relpersistence<>'t' AND i.indisvalid AND i.indisready") ||
        die "Inventory query failed"
    expect "$mode-parent-inventory" 0 "$value"
    value=$(sql -c "SELECT /* wiki_amr_leaf_inventory */ count(*) FROM pg_index i
        JOIN pg_class c ON c.oid=i.indexrelid WHERE i.indrelid='leaf_probe'::regclass
        AND c.relkind='i' AND i.indisvalid") || die "Leaf inventory failed"
    expect "$mode-leaf-indexes" 1 "$value"
    sql > "$work/$mode-temp.out" 2> "$work/$mode-temp.err" <<'SQL'
CREATE /* wiki_amr_temp_fixture */ TEMP TABLE temp_probe(id integer PRIMARY KEY);
SELECT /* wiki_amr_temp_inventory */ count(*) FROM pg_index i
JOIN pg_class c ON c.oid=i.indexrelid
WHERE i.indrelid='temp_probe'::regclass AND c.relkind='i' AND c.relpersistence<>'t';
SELECT /* wiki_amr_temp_direct */ bt_index_check('temp_probe_pkey',true);
SQL
    test "$?" -eq 0 || die "Temporary index check failed"
    value=$(head -n 1 "$work/$mode-temp.out") || die "Cannot read temp result"
    expect "$mode-temp-inventory" 0 "$value"
    record "$mode-temp-direct-check"
    ok "$mode-predicate-before-damage" "SELECT /* wiki_amr_predicate_control */ bt_index_check('toast_predicate',true)"
    bad "$mode-table-rejected" '42809:.*is not an index' \
        "SELECT /* wiki_amr_table_target */ bt_index_check('toast_probe')"
    value=$(sql -c "SELECT /* wiki_amr_free_offset */ bool_and(lower<=512 AND upper>512)
        FROM (SELECT lower,upper FROM page_header(get_raw_page('checksum_probe',0))
        UNION ALL SELECT lower,upper FROM page_header(get_raw_page('partial_probe',0))) p") ||
        die "Cannot verify fault offset"
    expect "$mode-fault-offset-is-free-space" t "$value"
}
verify_mode() {
    local mode=$1 checksumfile partialfile unloggedfile toastrel toastfile value
    init_cluster "$mode"
    common_checks "$mode"
    checksumfile=$(relation_path checksum_probe) || die "Missing checksum path"
    partialfile=$(relation_path partial_probe) || die "Missing partial path"
    unloggedfile=$(relation_path unlogged_probe) || die "Missing unlogged path"
    toastrel=$(sql -c "SELECT /* wiki_amr_toast_name */ reltoastrelid::regclass
        FROM pg_class WHERE oid='toast_probe'::regclass") || die "Missing toast relation"
    toastfile=$(relation_path "$toastrel") || die "Missing toast path"
    stop_owned || die "Stop before fault injection failed"
    cp "$checksumfile" "$work/checksum.good" &&
    cp "$partialfile" "$work/partial.good" || die "Cannot save clean images"
    flip_free_byte "$checksumfile"
    : > "$toastfile" || die "Cannot truncate fixture TOAST heap"
    start
    ok "$mode-index-without-heap" "SELECT /* wiki_amr_index_only_check */ bt_index_check('checksum_probe_pkey')"
    ok "$mode-toast-unrelated-index" "SELECT /* wiki_amr_plain_check */ bt_index_check('toast_probe_pkey',true)"
    bad "$mode-toast-predicate-error" 'XX001:.*could not read block' \
        "SELECT /* wiki_amr_predicate_fault */ bt_index_check('toast_predicate',true)"
    if test "$mode" = off; then
        ok off-free-space-damage "SELECT /* wiki_amr_off_fault */ bt_index_check('checksum_probe_pkey',true)"
    else
        value=$(count_checksums) || die "Counter query failed"
        expect on-initial-checksum-counter 0 "$value"
        bad on-track-counts-off 'XX001:.*invalid page' \
            "SET /* wiki_amr_disable_counts */ track_counts=off;
             SELECT /* wiki_amr_unreported_fault */ bt_index_check('checksum_probe_pkey',true)"
        value=$(count_checksums) || die "Counter query failed"
        expect on-counter-after-disabled-reporting 0 "$value"
        bad on-track-counts-on 'XX001:.*invalid page' \
            "SELECT /* wiki_amr_reported_fault */ bt_index_check('checksum_probe_pkey',true)"
        wait_count 1
        record on-counter-after-enabled-reporting=1
        if backup on-one-mismatch-backup; then die "Corrupt backup unexpectedly succeeded"; fi
        grep -E 'checksum error occurred' \
            "$work/on-one-mismatch-backup.err" >/dev/null || die "Wrong backup error"
        record on-one-mismatch-backup-fails
        sleep 1
        value=$(count_checksums) || die "Counter query failed"
        expect on-counter-after-one-backup-mismatch 1 "$value"
        stop_owned || die "Stop before partial-file fixture failed"
        cp "$work/checksum.good" "$checksumfile" || die "Cannot restore checksum fixture"
        flip_free_byte "$partialfile"
        printf '\001' >> "$partialfile" || die "Cannot append partial block"
        rm -rf "$work/backup" || die "Cannot reset owned backup directory"
        start
        backup on-partial-file-backup || die "Partial-file backup failed"
        grep -E 'could not verify checksum.*read buffer size' \
            "$work/on-partial-file-backup.err" >/dev/null || die "Missing partial-file warning"
        record on-partial-file-backup-succeeds-with-warning
        value=$(wc -c < "$work/backup/${partialfile#"$data/"}" | tr -d ' ') ||
            die "Cannot read backup size"
        expect on-partial-file-backup-bytes 8193 "$value"
        test ! -e "$work/backup/${unloggedfile#"$data/"}" ||
            die "Backup unexpectedly contains unlogged main fork"
        record on-backup-excludes-unlogged-main
        test -f "$work/backup/${unloggedfile#"$data/"}_init" ||
            die "Backup lacks unlogged init fork"
        record on-backup-includes-unlogged-init
        stop_owned || die "Stop before restoring fixture failed"
        cp "$work/partial.good" "$partialfile" || die "Cannot restore partial fixture"
        start
        ok on-restored-check "SELECT /* wiki_amr_restored */ bt_index_check('checksum_probe_pkey',true)"
    fi
    stop_owned || die "Mode teardown failed"
}
verify() {
    prepare
    test -x "$bin/postgres" || die "Run build first"
    : > "$work/results.tsv" || die "Cannot create results"
    checks=0
    uname -srm > "$work/host.txt" || die "Cannot record host"
    "$bin/pg_config" --configure >> "$work/host.txt" || die "Cannot record build"
    local mode
    for mode in off on; do verify_mode "$mode"; done
    printf 'TOTAL\t%s\n' "$checks" | tee -a "$work/results.tsv" || die "Cannot record total"
}
interrupt() {
    prepare
    test -x "$bin/postgres" && test -f "$runner" || die "Extract the runner and build first"
    test -z "$startup_signal" || die "Do not set the test signal on the interrupt parent"
    local signal expected rc saved=0
    if test -f "$work/results.tsv"; then
        cp "$work/results.tsv" "$work/results.saved" || die "Cannot save measurement summary"
        saved=1
    fi
    : > "$work/startup-interruptions.tsv" || die "Cannot create interruption results"
    for signal in TERM INT; do
        if test "$signal" = TERM; then expected=143; else expected=130; fi
        AMR12_TEST_STARTUP_SIGNAL="$signal" bash "$runner" verify \
            > "$work/startup-$signal.log" 2>&1
        rc=$?
        test "$rc" -eq "$expected" || die "$signal startup exit: expected $expected, got $rc"
        stopped || die "$signal left a server, PID file, socket or occupied port"
        printf 'PASS\tstartup-%s-cleanup\texit=%s\n' "$signal" "$rc" |
            tee -a "$work/startup-interruptions.tsv" || die "Cannot record interruption result"
    done
    if test "$saved" -eq 1; then
        mv "$work/results.saved" "$work/results.tsv" || die "Cannot restore measurement summary"
    else
        rm -f "$work/results.tsv" || die "Cannot remove interrupted measurement summary"
    fi
}
cleanup() {
    prepare
    stopped || die "Refusing cleanup while a service may be running"
    rm -rf "$work" || die "Sandbox deletion failed"
    printf 'Teardown confirmed; deleted %s\n' "$work"
}

if test "$#" -eq 0; then set -- build verify interrupt cleanup; fi
for stage in "$@"; do
    began=$SECONDS
    case "$stage" in build) build;; verify) verify;; interrupt) interrupt;; cleanup) cleanup;;
        *) die "Stages: build verify interrupt cleanup";; esac
    printf '%s completed in %s seconds\n' "$stage" "$((SECONDS-began))"
done
```

## Context Reviewed

- amcheck module: `verify_nbtree.c` in full (entry points, `BtreeCheckState`, locks, target checks, the level walk, per-page checks, downlink and missing-downlink checks, the heap callback, normalization, root descent, the invariant helpers, page copying, line-pointer checks), the three SQL scripts, the control file, the Makefile, the regression SQL and `amcheck.sgml`.
- Core boundary: `index_open()`, `table_beginscan_strat()`, `table_index_build_scan()`, `heapam_index_build_range_scan()`, `heapgetpage()`, `index_form_tuple()`, `heap_tuple_fetch_attr()`, `textlike()` and detoast macros, `_bt_checkpage()`, `_bt_heapkeyspace()`, `_bt_killitems()`, the lock conflict table and recovery lock limit, `bloomfilter.c`.
- Checksums: `ReadBuffer_common()` read and hit paths, `FlushBuffer()`, `PageIsVerified()`, `PageSetChecksumCopy()`, `pg_checksum_page()`, `mdread()` and process-local `InRecovery`, `DataChecksumsEnabled()`, `XLogHintBitIsNeeded()`, `MarkBufferDirtyHint()` and its caller conditions, `pgstat_report_checksum_failures_in_db()`, the relevant GUC definitions, `initdb -k`, `pg_checksums`, base backup file selection and verification including unlogged/temporary exclusions, unaligned reads and per-file counters, client error translation, and `checksum_failures` docs.
- Settings and SQL: `maintenance_work_mem`, `statement_timeout`, `lock_timeout`, `default_transaction_isolation`, `STRICT` semantics, extension `superuser` default, catalog columns and the `oid`-to-`regclass` cast.
- Build/generated headers: `BLCKSZ` and `MAXIMUM_ALIGNOF` from `configure.in`; no generated header is specific to amcheck.
- History: the checkout is a shallow clone, so no history claim is made.
- Measurement: the complete script above, its 39-check result record, two startup-interruption results and both platform records; expected API errors in the upstream regression output; pageinspect's header and B-tree statistics fields; `pg_ctl` launch, detachment and startup-timeout paths.
- Comparison set for the contradiction check: the whole page including the script and results; the [zeroed-segment amcheck page](amcheck-zeroed-relation-segments.md), the [corruption-log page](../observability/corruption-log-messages-with-checksums-disabled.md), the [post-drop health page](../server-administration/health-after-zeroed-schema-drop.md), the [database-health checklist](../server-administration/database-health-checklist.md), related heap/visible-count and zeroed-file claims found by symbol searches, and the B-tree bloat page's contrib-boundary statement; the glossary entries linked here; this page's index entries and the v12 manifest row. The hint-bit comparison also found the separate leaf-density issue recorded below. PostgreSQL 12 has no common-concept pages.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| B-tree only; no table checker | [Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4), [target test](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L338) |
| Overloads, `STRICT`, `PUBLIC` revoked, superuser to install | [SQL scripts](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L6-L24), [extension.c:616](../../../../raw/postgres-12/src/backend/commands/extension.c#L616) |
| Lock modes and early release | [verify_nbtree.c:232-303](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L232-L303) |
| Walk follows links from the true root | [verify_nbtree.c:500-551](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L500-L551), [level walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L649-L835) |
| Per-page and ordering checks | [page sanity](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2334-L2474), [bt_target_page_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L871-L1283) |
| Parent-only checks | [downlink check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1508-L1647), [missing downlink](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1662-L1835), [root descent](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2058-L2107) |
| One-way, probabilistic table comparison | [heap phase](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L625), [callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926), [Bloom contract](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10) |
| Snapshot rules, predicate, HOT root, TOAST fetch | [heapam_handler.c:1212-1666](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1212-L1666), [indextuple.c:76-88](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88) |
| No uniqueness check | [verify_nbtree.c:599-607](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L599-L607) |
| Bloom sizing formulas | [verify_nbtree.c:419-432](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L419-L432), [bloom_create](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L87-L121), [optimal_k](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L224-L235) |
| Checksum verify on read, not on hit; computed on write | [bufmgr.c:762-925](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L925), [bufpage.c:81-162](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162), [bufmgr.c:2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2750) |
| Checksum includes block number, not relation; 16-bit | [checksum_impl.h:176-214](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214) |
| Checksums off by default; offline enable | [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [pg_checksums.c:513-521](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L513-L521) |
| Offline and backup verification scope | [pg_checksums.c:170-240](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L170-L240), [basebackup.c:1413-1556](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1413-L1556) |
| Predicates can fetch TOAST values not stored in the index | [predicate and expression evaluation](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626), [textlike](../../../../raw/postgres-12/src/backend/utils/adt/like.c#L284-L300), [detoast macros](../../../../raw/postgres-12/src/include/fmgr.h#L283-L303); measured by both TOAST-predicate legs |
| Counter exclusions and backup verification bypass | [report gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556), [backup per-file counter](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1611-L1622), [unaligned read](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1454-L1466); measured in the checksums-on leg |
| Hint-update FPI conditions | [MarkBufferDirtyHint](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3422-L3510), [XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L895-L957); source-only |
| Backup excludes unlogged non-init forks and temporary relations before checking pages | [sendDir](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1162); unlogged main/init selection measured in the checksums-on backup leg |
| Startup cleanup must account for a server before ownership confirmation | [pg_ctl launch](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L449-L486), [startup timeout](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L887-L924); two deterministic interruption tests in the published script |
| `zero_damaged_pages`, `ignore_checksum_failure`, hot standby behavior | [guc.c:1135-1161](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161), [bufmgr.c:908-924](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L908-L924), [lock.c:758-766](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766) |

## Open Questions

- **Docs versus source on who may run the checks.** `amcheck.sgml` says the functions "may only be used by superusers". The source revokes `EXECUTE` from `PUBLIC` and skips relation permission checks on purpose, and the regression test runs both checks as an ordinary role after `GRANT EXECUTE`. Source wins: a superuser can grant the right. [amcheck.sgml#superuser](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L37-L39) [verify_nbtree.c#no-permission-check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L306-L313) [check_btree.sql#permissions](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L29-L39)
- **Docs wording on which heap tuples are checked.** `amcheck.sgml` says `bt_index_check` verifies "the presence of all heap tuples as index tuples". The source checks only rows visible to its MVCC snapshot under `bt_index_check`, only rows satisfying a partial index's predicate, and under the parent check excludes HOT-updated recently dead rows. Source wins; the docs sentence is broader than the code. [amcheck.sgml#bt_index_check](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L100-L104) [heapam_handler.c#vacuum-visibility](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1402-L1428) [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1611-L1615)
- **Exact miss probability below 2 bytes per row.** The docs and the source comment bound the rate at about 2% for about 2 bytes per row, but the pinned tree does not compute it for less memory, as in calculation B. The rate there is left unstated.
- **Docs versus source on checksum counters.** The monitoring description does not name the `track_counts` gate or this pin's single-mismatch base-backup exception. The source wins, and the measurement reproduces both exclusions. [monitoring.sgml#checksum_failures](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2600-L2612) [pgstat.c#report-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556) [basebackup.c#per-file-report](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1611-L1622)
- **Limits of this measurement.** The script does not inject downlink corruption, estimate Bloom miss rates, exercise malformed `pd_upper == 0` pages, measure hint-update FPIs, test `zero_damaged_pages`, test standby short reads, measure temporary-relation backup exclusion, or restore the malformed backup. Those explanations remain source deductions, not measured outcomes. The shutdown tests inject signals at the pending-ownership boundary after launch; they do not reproduce a real startup timeout or every signal timing. No source/measurement disagreement remains: truncating the TOAST heap fails at the earlier short-read check, and the backup client translates the server's checksum error as documented by their separate source paths.
- **Contradiction outside this task's edit scope.** The [leaf-density/fragmentation page](leaf-density-vs-fragmentation-index-scan-io.md#reproduction) says `synchronous_commit = off` prevents the first VACUUM from setting committed hint bits. The [Hint bits glossary entry](../../../glossary.md#hint-bits) correctly makes this conditional. In this pin, `SetHintBits()` refuses only when the buffer is permanent, the commit LSN still needs flushing, and the page LSN is older. The source supports the glossary; the other page's measured outcome does not establish its unconditional explanation. That page was reported but not edited. [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L113-L131)
- **Another contradiction outside this task's edit scope.** The [database-health checklist](../server-administration/database-health-checklist.md#observability-prerequisites) says offline `pg_checksums` “verifies every file,” following `pg_checksums.sgml`'s statement that every file in the cluster is scanned or rewritten. This page's tool table instead names its exclusions. The source supports the bounded scope: the `skip[]` list, `pgsql_tmp` temporary-entry exclusions and optional `-f` filenode filtering limit files, and `PageIsNew()` pages bypass comparison. The checklist was reported but not edited, and the documentation's unqualified "every file" wording is recorded here as a docs-versus-source discrepancy. [pg_checksums.sgml#operation](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L48-L52) [pg_checksums.c#skip-list](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L95-L110) [pg_checksums.c#temporary-exclusions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L301-L311) [pg_checksums.c#filenode-filter](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L353-L355) [pg_checksums.c#new-page-skip](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L210-L213)
- **No common-concept pages.** PostgreSQL 12 has no common-concept page for B-tree integrity checking or for data checksums and page verification. Both would be useful separate pages; neither was created as part of this answer.

## Source References

- [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1-L2573) — detailed ranges accompany each claim.
- [Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L21), [amcheck.control](../../../../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L5), [amcheck--1.0.sql](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L1-L24), [amcheck--1.0--1.1.sql](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0--1.1.sql#L1-L29), [amcheck--1.1--1.2.sql](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.1--1.2.sql#L1-L19), [check_btree.sql](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L1-L114), [amcheck.sgml](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L1-L341).
- [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1150-L1699), [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L351-L394), [tableam.h](../../../../raw/postgres-12/src/include/access/tableam.h#L761-L773), [tableam.h:1512-1533](../../../../raw/postgres-12/src/include/access/tableam.h#L1512-L1533), [indextuple.c](../../../../raw/postgres-12/src/backend/access/common/indextuple.c#L76-L88), [tuptoaster.c](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L101-L110), [tuptoaster.c:2013](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L2013).
- [nbtpage.c](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L638-L719), [nbtree.h](../../../../raw/postgres-12/src/include/access/nbtree.h#L131-L135), [nbtree.h:346-355](../../../../raw/postgres-12/src/include/access/nbtree.h#L346-L355), [itup.h](../../../../raw/postgres-12/src/include/access/itup.h#L35-L147), [itemptr.h](../../../../raw/postgres-12/src/include/storage/itemptr.h#L28-L40), [itemid.h](../../../../raw/postgres-12/src/include/storage/itemid.h#L25-L30), [c.h](../../../../raw/postgres-12/src/include/c.h#L685), [configure.in](../../../../raw/postgres-12/configure.in#L149-L164), [configure.in:253-265](../../../../raw/postgres-12/configure.in#L253-L265), [configure.in:2010](../../../../raw/postgres-12/configure.in#L2010), [pg_config_manual.h:182](../../../../raw/postgres-12/src/include/pg_config_manual.h#L182).
- [bloomfilter.c](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L1-L235).
- [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L925), [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750), [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L63-L162), [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189), [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L150-L163), [checksum_impl.h](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L1-L214), [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630), [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4821-L4826), [xlog.c:7913-7957](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7913-L7957), [xlog.h:192](../../../../raw/postgres-12/src/include/access/xlog.h#L192).
- [lock.c](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86), [lock.c:758-766](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766), [xact.h:51](../../../../raw/postgres-12/src/include/access/xact.h#L51), [errcodes.txt](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L493-L494).
- [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161), [guc.c:1827-1835](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1827-L1835), [guc.c:2155-2199](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2199), [guc.c:2231-2252](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2252), [guc.c:2378-2396](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396), [guc.c:3905-3960](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3905-L3960), [guc.c:4255-4262](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4255-L4262).
- [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [initdb.c:1018-1173](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1018-L1173), [initdb.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L225), [pg_checksums.c](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L1-L5), [pg_checksums.c:95-355](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L95-L355), [pg_checksums.c:513-590](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L513-L590), [pg_checksums.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L48-L52), [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L208-L218), [basebackup.c:1343-1556](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1343-L1556), [protocol.sgml](../../../../raw/postgres-12/doc/src/sgml/protocol.sgml#L2569-L2577), [monitoring.sgml](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2600-L2612).
- [extension.c](../../../../raw/postgres-12/src/backend/commands/extension.c#L616), [create_function.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L385-L390), [index.c#relpersistence](../../../../raw/postgres-12/src/backend/catalog/index.c#L744-L749), [pg_index.h](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L42), [pg_class.h](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L50-L81), [pg_am.h](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L34), [pg_cast.dat](../../../../raw/postgres-12/src/include/catalog/pg_cast.dat#L178-L179).
- [indexam.c#index_open](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L126-L140), [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L588), [nbtutils.c#killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1781-L1811), [nbtutils.c#_bt_check_natts](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L2382-L2474), [heapam_visibility.c#SetHintBits](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L113-L131).
- [xloginsert.c#XLogSaveBufferForHint](../../../../raw/postgres-12/src/backend/access/transam/xloginsert.c#L895-L957), [elog.c#default-errcode](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388), [pgstat.c#checksum-reporting](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556).
- [like.c#textlike](../../../../raw/postgres-12/src/backend/utils/adt/like.c#L284-L300), [fmgr.h#detoast-macros](../../../../raw/postgres-12/src/include/fmgr.h#L283-L303), [check_btree.out#target-errors](../../../../raw/postgres-12/contrib/amcheck/expected/check_btree.out#L46-L61), [pageinspect--1.5.sql#page_header](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L20-L33), [pageinspect--1.5.sql#bt_page_stats](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L160-L175).
- [pg_basebackup.c#options](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L330-L353), [pg_basebackup.c#checksum-error](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L2025-L2036), [installation.sgml#build-tools](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L80-L151), [installation.sgml#git-build](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L300-L318).
- [gram.y#persistence-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283), [heapam_handler.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624), [basebackup.c#relation-exclusions](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1162), [xlog.c#InRecovery](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L189-L200).
- [pg_ctl.c#detached-launch](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L449-L486), [pg_ctl.c#startup-timeout](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L887-L924).

## Navigation

- [Can amcheck Detect Zeroed Relation Segments in PostgreSQL 12? (unverified)](amcheck-zeroed-relation-segments.md)
- [Corruption indicators with checksums disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md)
- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
