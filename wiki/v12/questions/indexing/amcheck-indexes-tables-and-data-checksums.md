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
  - [Follow-up: why vanished heap rows go unreported](#follow-up-why-vanished-heap-rows-go-unreported)
  - [Follow-up: checking for dangling index entries](#follow-up-checking-for-dangling-index-entries)
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

Follow-up (2026-10-26):

Using amcheck, are index entries whose heap rows vanished not reported? Why not? Is there a way to perform this check?

> Prompt note: corrected from "using amcheck index entries whose heap rows vanished are not reported? why not? is there a way to perform this check?" Corrections: capitalized each sentence, added the comma after the introductory phrase, and turned the first statement-with-question-mark into a question.

## Answer

### Short answer

**PostgreSQL 12's [`amcheck`](../../../glossary.md#amcheck) checks [B-tree](../../../glossary.md#b-tree) indexes only. It has no function that takes a table.** The module builds one source file, `verify_nbtree.c`, and its two functions reject any relation that is not a B-tree index. [Makefile#amcheck](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4) [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L323) [check_btree.sql#table-rejection](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L41-L43)

A table is checked only indirectly. When you pass `heapallindexed = true`, the checker scans the index's table and asks, for each row that should be in the index, whether the index holds a matching entry. So the question's premise holds only in this narrow sense: amcheck tests a table through the indexed columns of the rows its scan finds. [verify_nbtree.c#heap-phase](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L625) [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926)

It finds three kinds of problems:

- **Broken index structure:** zero or malformed pages, bad line pointers, wrong levels, broken sibling links, and keys out of order within a page, across pages, or between a parent and its child. [verify_nbtree.c#bt_target_page_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L837-L870)
- **Index entries that the operator class no longer orders the same way**, for example after an operating-system collation change. [amcheck.sgml#collation](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L222-L250)
- **[Heap](../../../glossary.md#heap) rows that lack a matching index entry**, including rows whose indexed bytes were damaged, and some HOT-chain and TOAST damage met along the way. [verify_nbtree.c#callback-rationale](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1837-L1891)

Its main limits follow from that design. It reads only the index pages its tree walk reaches, and only the heap rows its scan returns. It never asks the reverse question, whether every index entry still has a heap row; see [why](#follow-up-why-vanished-heap-rows-go-unreported) and [a `pageinspect` check that does](#follow-up-checking-for-dangling-index-entries). Its row test is probabilistic. It sees pages as they sit in [shared buffers](../../../glossary.md#shared_buffers), and it stops at the first error. [verify_nbtree.c#root-to-leaf-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L551) [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10) [amcheck.sgml#shared-buffers](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L280-L293)

[Data checksums](../../../glossary.md#data-checksums) cover a different gap. They are off by default. When a cluster has them, an initialized page that amcheck causes to be **read from disk** gets a checksum comparison before amcheck sees it. A mismatch raises an error unless a superuser has turned on `ignore_checksum_failure` (the page is accepted only if its header looks sane) or `zero_damaged_pages` (the page is zeroed with only a warning); both are off by default (see [Non-default settings](#non-default-settings-and-special-cases)). Random damage can also match the 16-bit checksum by chance (see [the checksum section](#how-data-checksums-work-in-postgresql-12)). That can catch damage in bytes amcheck never interprets, such as non-indexed columns, row headers and free space. Checksums do not help with pages already in shared buffers, pages amcheck never reads, [all-zero pages](../../../glossary.md#all-zero-page), truncated files, or damage that existed in memory before an initialized page was written with a valid checksum. [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162)

The explanation follows the pinned source. The [measurement script](#measurement-script) also tests selected boundaries with checksums off and on: valid dead index entries, target selection, TOAST reads from predicates, free-space damage, statistics reporting and base-backup verification. Other cases remain source deductions.

### Mental model

amcheck checks **invariants**: conditions that must always hold if the index is sound. An example is that items on a B-tree page are in key order. A failed invariant raises an error. A pass means only that no checked invariant failed. [amcheck.sgml#invariants](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L10-L28) [amcheck.sgml#cannot-prove-absence](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L316-L317)

The tree-navigation terms matter for these checks. A [pivot tuple](../../../glossary.md#pivot-tuple) guides searches rather than pointing to a heap row. It can hold a separator key, a downlink to a child page, or both. A page's [high key](../../../glossary.md#leaf-page) is its upper key bound. An internal page's first data item is the [negative-infinity pivot](../../../glossary.md#pivot-tuple): it has a downlink but no key attributes, so ordinary key comparisons cannot check it. [nbtree/README#pivot-tuples](../../../../raw/postgres-12/src/backend/access/nbtree/README#L17-L40) [verify_nbtree.c#offset_is_negative_infinity](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2119-L2143)

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

Each script revokes `EXECUTE` from `PUBLIC`. The C code deliberately skips relation permission checks, so a role granted `EXECUTE` on an overload does not also need privileges on the target relation. The regression test grants those function privileges to an ordinary role and runs both checks as that role. [amcheck--1.0.sql#revoke](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L21-L24) [verify_nbtree.c#no-permission-check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L306-L313) [check_btree.sql#permissions](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L23-L39) This covers only otherwise eligible B-tree indexes in the connected database: the grant does not bypass the target exclusions below, and relation lookup rejects cross-database references. A cluster-wide checking process must connect to each database and have the installed function available and executable there. [verify_nbtree.c#target-exclusions](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L338) [namespace.c#database-boundary](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L241-L251) [contrib/README#per-database-installation](../../../../raw/postgres-12/contrib/README#L19-L25) Installing the extension is different: `amcheck.control` sets no `superuser` line, and the control-file parser defaults it to true, so `CREATE EXTENSION amcheck` needs a superuser. [amcheck.control](../../../../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L5) [extension.c#superuser-default](../../../../raw/postgres-12/src/backend/commands/extension.c#L616) [extension.c#superuser-check](../../../../raw/postgres-12/src/backend/commands/extension.c#L803)

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

The walk reads pages reached through the metapage, downlinks and sibling links, including links already copied by this call. Fully deleted pages have been unlinked from their siblings and parents, so a later walk does not reach them through those links. [verify_nbtree.c#deleted-pages-unlinked](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L686-L697) However, `bt_index_check` permits concurrent VACUUM: a link copied before deletion can still lead to the now-deleted page. Recycling waits until such readers can no longer reach it, so the checker can read the page and follow its retained right link. This is why the deleted-page handling above is needed even though deletion removes incoming links. [verify_nbtree.c#copied-links-and-deletion](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1327-L1359)

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
| Each non-high-key tuple's own length equals its line pointer's length; this includes the internal negative-infinity item | `index tuple size does not equal lp_len`, hint `This could be a torn page problem.` | [verify_nbtree.c#non-high-key-loop](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L912-L951) |
| High key and items have the expected attribute count, including that leaf data items do not use the pivot `INDEX_ALT_TID_MASK` flag | `wrong number of high key index tuple attributes` / `wrong number of index tuple attributes` | [verify_nbtree.c:884-976](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L884-L976), [nbtutils.c#_bt_check_natts](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L2382-L2474) |
| Comparable non-high-key items (excluding the internal negative-infinity item) do not exceed the version's size limit | `index row size ... exceeds maximum` | [verify_nbtree.c:988-993](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993), [verify_nbtree.c:1047-1070](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1047-L1070) |
| On an equal-prefix tiebreaker in a version 4 index, the compared upper-bound leaf item (next item on the page, or child item under the parent check) carries a heap [TID](../../../glossary.md#tid) | `... contains non-pivot tuple that lacks a heap TID` | [verify_nbtree.c:2181-2202](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2181-L2202), [verify_nbtree.c:2296-2317](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2296-L2317), [verify_nbtree.c#BTreeTupleGetHeapTIDCareful](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2559-L2573), [nbtree.h#BTreeTupleGetHeapTID](../../../../raw/postgres-12/src/include/access/nbtree.h#L346-L355) |
| Every comparable non-high-key item (excluding the internal negative-infinity item) is at most the page's high key (strictly less on internal pages) | `high key invariant violated` | [verify_nbtree.c:988-993](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993), [verify_nbtree.c:1130-1152](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1130-L1152) |
| Every comparable non-high-key item (excluding the internal negative-infinity item) is strictly less than the next item | `item order invariant violated` | [verify_nbtree.c:988-993](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993), [verify_nbtree.c:1160-1199](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1160-L1199) |
| The last comparable item is less than the first comparable item of the next live page to the right, when that item exists; skip the internal negative-infinity item | `cross page item order invariant violated` | [verify_nbtree.c:1218-1258](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1218-L1258), [verify_nbtree.c#bt_right_page_check_scankey](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1301-L1497) |
| A level has at least one live page | `index "..." has no valid pages on level below ...` | [verify_nbtree.c:544-548](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L544-L548) |

The ordering checks compare keys with the index's own B-tree comparison support function, the same code index scans use. That is why amcheck can detect an [operator class](../../../glossary.md#operator-class) or [collation](../../../glossary.md#collation) whose ordering changed after the index was built. [amcheck.sgml#opclass-code](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L29-L36) [verify_nbtree.c#invariant_l_offset](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2154-L2205)

`bt_index_check` can run while other sessions change the index, so the cross-page check has one race to rule out. When the check seems to fail, it re-reads the target page. If the page has meanwhile become deleted or half-dead (`P_IGNORE()`), the apparent violation is attributed to that concurrent deletion. This branch returns silently; it emits neither a corruption error nor a `DEBUG1` message for the race. [nbtree.h#P_IGNORE](../../../../raw/postgres-12/src/include/access/nbtree.h#L194) [verify_nbtree.c#cross-page-recheck](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1228-L1247) [verify_nbtree.c#race-explanation](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1365-L1454)

An index carried over by [`pg_upgrade`](../../../glossary.md#pg_upgrade) from a release before on-disk version 4 may legally hold equal neighbouring keys. For such indexes the "strictly less" checks relax to "less than or equal", which is a weaker check. [verify_nbtree.c#pg_upgrade-relaxation](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2166-L2168) [nbtpage.c#_bt_heapkeyspace](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L638-L686)

### What the parent check adds

`bt_index_parent_check` holds `ShareLock`, so no page split, page deletion or `VACUUM` can run during the call. That stability is what makes these extra checks safe from false alarms. [verify_nbtree.c#sharelock-rationale](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L373-L384)

| Extra check | Needs | Error when it fails |
|---|---|---|
| Each level's first page is leftmost; the top level's page is the true root | parent check | `block ... is not leftmost` / `block ... is not true root` [verify_nbtree.c:727-740](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L727-L740) |
| Each page's left link names the page the walk came from | parent check | `left link/right link pair ... not in agreement` [verify_nbtree.c:783-793](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L783-L793) |
| No level-walk link or non-negative-infinity downlink reaches a deleted page | parent check | `downlink or sibling link points to deleted block` / `downlink to deleted page found` [verify_nbtree.c:698-704](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L698-L704) [verify_nbtree.c:988-993](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993) [verify_nbtree.c:1268-1273](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1268-L1273) [verify_nbtree.c:1593-1601](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1593-L1601) |
| Comparable child keys are strictly greater than the parent's separator in version 4; versions 2/3 allow equality. Negative-infinity parent and child items are excluded | parent check | `down-link lower bound invariant violated` [verify_nbtree.c#downlink-call](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L988-L993) [verify_nbtree.c#bt_downlink_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1603-L1644) [verify_nbtree.c#version-rule](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2273-L2319) |
| A reached non-root, non-ignored page's block number is not definitely absent from the downlink filter, unless an interrupted split/deletion explains the missing link; when descending a missing-downlink internal subtree via negative-infinity downlinks, each internal child is one level down and the final leaf is not deleted. The descent breaks on a leaf before testing its level against its parent's | parent check **and** `heapallindexed = true` | `leaf index block lacks downlink` / `downlink points to block ... whose level is not one level down` / `downlink to deleted leaf page found` / `internal index block lacks downlink` [verify_nbtree.c#missing-downlink-call](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1276-L1282) [verify_nbtree.c#bt_downlink_missing_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1662-L1834) |
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
| A reached page with an unjustified missing downlink, or a missing-downlink internal subtree whose negative-infinity chain has an internal child at the wrong relative level or a deleted leaf | Lost parent update or premature page deletion | Probabilistic missing-downlink check | parent check + `heapallindexed` |
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
| One-way table comparison | The callback probes only for rows the scan returns. [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) | An index entry whose heap row vanished is not reported. Rows lost from the heap go unnoticed; see [the zeroed-segment page](amcheck-zeroed-relation-segments.md), [why](#follow-up-why-vanished-heap-rows-go-unreported) and [a separate check](#follow-up-checking-for-dangling-index-entries). |
| Probabilistic row test | A Bloom filter can say "possibly present" for an absent entry. [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L10) | Each missing entry escapes with a small probability; the docs give no more than 2% at about 2 bytes per row. See the [worked example](#bloom-filter-sizing-worked-example). [amcheck.sgml#bloom-memory](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L195-L207) |
| Additional parent checks are incomplete | Downlink presence uses a second Bloom filter; `rootdescend` can recover by moving right. [verify_nbtree.c#downlink-probe](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1714-L1719) [verify_nbtree.c#rootdescend-limitation](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2070-L2079) | A successful stronger check still does not prove every downlink is present or correct. |
| Only reachable index pages | The walk follows links, including previously copied links. [verify_nbtree.c#root-to-leaf-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L521-L551) [verify_nbtree.c#copied-links-and-deletion](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1327-L1359) | Blocks unreachable through those links are not read. Concurrent deletion can leave a copied link pointing to a deleted page that the call still reads; there is no physical scan of every allocated block. |
| Only rows that belong in the index | Partial-index predicate; snapshot rules. [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) | Rows outside a partial index, and rows invisible to the chosen snapshot, are not probed. |
| No uniqueness check | `ii_Unique` is cleared. [verify_nbtree.c#unique-off](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L599-L607) | Duplicate keys in a unique index pass. |
| Cached pages | A buffer hit returns the cached copy without reading the file. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) [amcheck.sgml#shared-buffers](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L286-L290) | Damage on disk under a clean cached copy is invisible until the page is read again. |
| First error stops the call | Every reported violation uses `ereport(ERROR)`. Some tolerated states, such as ignored pages and interrupted splits, emit `DEBUG1` messages; the cross-page concurrent-deletion race instead returns silently. [verify_nbtree.c#ignored-page-debug](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L706-L717) [verify_nbtree.c#interrupted-split-debug](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1701-L1713) [verify_nbtree.c#silent-race-return](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1235-L1246) [amcheck.sgml#raises-error](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L140-L142) | One call reports one problem; the rest of the index is not checked. |
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
READ   cache miss -> mdread:
         short read?
           zero_damaged_pages on (default off), or caller's InRecovery true?
             yes -> fill buffer with zeros, without warning; continue to PageIsVerified
             no  -> ERROR
         complete read or zero-filled short read -> PageIsVerified(page, blockno):
           page not "new" (pd_upper != 0):
             checksums on? recompute; mismatch -> checksum_failure
             header sane (flags, pd_lower <= pd_upper <= pd_special <= BLCKSZ, aligned)?
             sane and no failure -> accept
           every byte zero? -> accept (no checksum compared)
           checksum_failure -> WARNING "page verification failed ..." and request a counter report
             track_counts on (default) and statistics socket available? send report; otherwise skip it
             header sane and ignore_checksum_failure on? -> accept
           -> reject: ERROR "invalid page in block ..." (or zero with WARNING if zero_damaged_pages is on)
HIT    page already in shared buffers -> returned without any verification
```

- Write path: [bufmgr.c:2745-2750](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2745-L2750), [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189).
- Read path: [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L81-L162), [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925). `zero_damaged_pages` defaults off and has superuser session/transaction scope; its short-read branch bypasses the later invalid-page warning because the verifier accepts the resulting all-zero buffer. [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161) [bufpage.c#all-zero-acceptance](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L123-L142) `InRecovery` here belongs to the startup process replaying WAL; a client backend on a hot standby does not get this exemption merely because the server is recovering. [xlog.c#InRecovery](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L189-L200)
- Counter-report path: `track_counts` defaults on and has superuser session/transaction scope. Turning it off, or having no statistics socket, suppresses the report without suppressing the checksum warning or changing page acceptance. [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1399) [pgstat.c#checksum-reporting](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556) [bufpage.c#warning-and-acceptance](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L148-L162)
- Hit path: [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796).

Three properties of the checksum matter for amcheck:

1. **It covers the whole page and its block number.** `pg_checksum_page()` hashes the header, apart from the checksum field, and all page data, then mixes in the block number "to detect transposed pages". Moving a page to another block can therefore expose the misplaced write, but detection still depends on the reduced checksum differing: distinct block numbers can produce the same 16-bit result. No relation identifier is mixed in, so a valid page from another relation at the same block number would pass. [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L176-L214)
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
| Page written to the wrong block of its relation | Index: may fail level, link or order checks. Heap: only through the Bloom probe. | Detected on an initialized disk read if mixing in the new block number changes the reduced checksum; a collision can conceal the relocation. [checksum_impl.h#blkno-and-reduction](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L207-L214) |
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
| `zero_damaged_pages = on` | Off. [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161) | A superuser sets it (`PGC_SUSET`, session or transaction scope with `SET`). `ReadBuffer_common()` zeroes a page that fails verification and warns. A short read takes a different path: `mdread()` returns zeros without warning, and those zeros pass verification. [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L908-L918) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L615-L624) [bufpage.c#all-zero-acceptance](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L123-L142) | A zeroed **index** page is still rejected by amcheck. A zeroed **heap** page supplies no rows to the scan, so `heapallindexed` can pass; the verification-failure path warns, while the short-read zero fill itself is silent. The zeroed buffer is marked valid, so later reads hit it in shared buffers. [bufmgr.c#mark-valid](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L945-L955) [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L357-L361) [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L698-L705) |
| `ignore_checksum_failure = on` | Off. [guc.c#ignore_checksum_failure](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1148) | A superuser sets it (`PGC_SUSET`, session or transaction scope with `SET`). `PageIsVerified()` still emits `WARNING: page verification failed ...` (`SQLSTATE` `01000`: the call names `ERRCODE_DATA_CORRUPTED` without wrapping it in `errcode()`, so the code is discarded and the `WARNING` default applies) and requests a failure-counter update, subject to the `track_counts` and statistics-socket gates above, then accepts the page if its header looks sane. [bufpage.c:148-158](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L148-L158) [elog.c#default-errcode](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L382-L388) [pgstat.c#checksum-reporting](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1545) | The read succeeds after the warning, and amcheck examines the damaged page, so the call raises an error only if an invariant or probe fails. |
| [Hot standby](../../../glossary.md#hot-standby) | A primary is not in recovery (`RecoveryInProgress()` is false). [xlog.c#RecoveryInProgress](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7913-L7957) | During recovery a backend cannot take a relation lock above `RowExclusiveLock`. [lock.c#recovery-lock-limit](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766) | `bt_index_parent_check` fails to lock; `bt_index_check` works. The docs note that collation mismatches between primary and standby can usually be seen only on the standby. [amcheck.sgml#standby](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L161-L164) [amcheck.sgml#collation](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L234-L239) |
| Unlogged index during recovery | By default, tables and their inherited indexes are permanent (`RELPERSISTENCE_PERMANENT`, `'p'`) and a primary is not in recovery, so `btree_index_mainfork_expected()` returns true and the main fork is required. [gram.y#persistence-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283) [index.c#relpersistence](../../../../raw/postgres-12/src/backend/catalog/index.c#L744-L749) [verify_nbtree.c:279-294](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L279-L294) | Explicit `UNLOGGED` sets `RELPERSISTENCE_UNLOGGED` (`'u'`) on the table and its indexes, and a standby has `RecoveryInProgress()` true, where such an index has no main-fork contents. [verify_nbtree.c#btree_index_mainfork_expected](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L340-L360) | `NOTICE` and skip; no check runs. |
| `REPEATABLE READ` or `SERIALIZABLE` transaction | `READ COMMITTED`. [guc.c#default_transaction_isolation](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4255-L4262) | `GetTransactionSnapshot()` can return the transaction's older snapshot, which may predate an index whose [`indcheckxmin`](../../../glossary.md#indcheckxmin) is set. [xact.h#IsolationUsesXactSnapshot](../../../../raw/postgres-12/src/include/access/xact.h#L51) | `bt_index_check(idx, true)` may fail with `cannot be verified using transaction snapshot` (`40001`). [verify_nbtree.c#indcheckxmin](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L454-L468) |
| Index from a pre-version-4 B-tree (via `pg_upgrade`) | A v12 build writes version 4. [nbtree.h#BTREE_VERSION](../../../../raw/postgres-12/src/include/access/nbtree.h#L133-L135) | `_bt_heapkeyspace()` reports false. | Weaker ordering checks; `rootdescend` fails with `cannot verify that tuples from index "..." can each be found by an independent index search` (`0A000`), hint `Only B-Tree version 4 indexes support rootdescend verification.` [verify_nbtree.c#version-gate](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L486-L492) |

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

**Index pages reached by links → read through shared buffers (checksum-verified only on a disk read, when the cluster has checksums) → structural and ordering invariants, plus parent/child checks under `ShareLock` → optional fingerprint of leaf entries → optional heap scan that probes the fingerprint for each row that belongs in the index → first violation raises `ERROR`, otherwise silence.** Checksums widen what that same read path catches to every byte of every initialized page read from disk, unless the damage happens to match the 16-bit checksum or a superuser has turned on `ignore_checksum_failure` or `zero_damaged_pages`, both off by default. [bufmgr.c#read-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) [bufpage.c:148-158](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L148-L158) They cannot reach cached pages, unread pages, all-zero pages, truncated files or damage written with a valid checksum. amcheck in turn catches ordering, cross-structure and in-memory damage that checksums can never see.

### Follow-up: why vanished heap rows go unreported

**Yes: in PostgreSQL 12, amcheck never reports an index entry whose heap row has vanished, with any function or option.** This section calls such an entry a *dangling index entry*: a B-tree leaf entry whose heap [TID](../../../glossary.md#tid) names a block past the end of the table, a [line pointer](../../../glossary.md#line-pointer) number the heap page does not have, or an unused line pointer. Three design facts cause the blind spot. The comparison runs only from heap to index. The fingerprint cannot be asked which of its entries went unmatched. And an index entry without a matching scanned row is often legitimate, so "unmatched" would not mean "corrupt". [verify_nbtree.c#header](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L11-L14) [bloomfilter.h#API](../../../../raw/postgres-12/src/include/lib/bloomfilter.h#L18-L25) [README.HOT#dead-root](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L116-L120)

The [measurement script](#measurement-script) shows the result. After a stopped-server fault zeroed one heap page (26 rows) and cut the heap file to 3 blocks (522 more rows), both `bt_index_check(idx, true)` and `bt_index_parent_check(idx, true, true)` passed with checksums off and on, while a sequential count fell from 598 to 50 rows. The index still held 548 dangling entries. See [Results and last run](#results-and-last-run).

**Values that matter:**

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| Leaf entry heap TID | The (block, line pointer) of the heap row the entry indexes | Stored in the index tuple's `t_tid` [itup.h#IndexTupleData](../../../../raw/postgres-12/src/include/access/itup.h#L35-L50) | Stored in the index | amcheck uses it as an ordering tiebreaker, in error details and as bytes hashed into the fingerprint. It never fetches the heap row it names. [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083) [verify_nbtree.c#BTreeTupleGetHeapTIDCareful](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2559-L2573) |
| Fingerprint bits | A Bloom filter of normalized leaf entries | `bloom_add_element()` during the index walk [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083) | Live, in memory for one call | Probed only by rows the heap scan finds. [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) |
| Heap block count | How far the heap scan reads | Measured when the scan starts [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L351-L379) [heapam_handler.c#scan-loop](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1306-L1309) | Live file size | Blocks past it are never read, so their lost rows probe nothing. |
| Heap line pointer state | `LP_UNUSED`, `LP_NORMAL`, `LP_REDIRECT` or `LP_DEAD` | The heap page's line pointer array [itemid.h#LP_states](../../../../raw/postgres-12/src/include/storage/itemid.h#L38-L41) | Stored on the heap page | Decides whether an index entry's TID still names a slot that may legitimately hold its row, a link or a dead stub. |

**Logic map of what `heapallindexed` compares:**

```text
INDEX WALK   each leaf data entry not marked LP_DEAD
               -> normalize -> bloom_add_element(bits)          (the entry's identity is not kept)   [1]
HEAP SCAN    blocks 0 .. nblocks-1, nblocks measured at scan start                                  [2]
               block past end of file ......... never read -> its lost rows probe nothing
               all-zero page .................. 0 line pointers -> no rows -> no probe                [3]
               each qualifying row -> form its index entry -> bloom_lacks_element()?                [4]
                    yes -> ERROR XX001 "heap tuple ... lacks matching index tuple"
                    no  -> next row
END          return. No step asks which fingerprinted entries no row probed;
             the Bloom API has no enumeration or removal.                                           [5]
```

- **[1]** [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083), [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L12)
- **[2]** [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614), [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L351-L379)
- **[3]** [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142), [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L357-L360)
- **[4]** [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926)
- **[5]** [bloomfilter.h#API](../../../../raw/postgres-12/src/include/lib/bloomfilter.h#L18-L25)

**Why the reverse question is not asked:**

1. **The check is defined as a subset test in one direction.** The docs state the principle: a new index equivalent to the target "must only have entries that can be found in the existing structure". The header comment says the heap scan verifies "that every visible heap tuple has a matching index tuple". Nothing in that principle requires every existing entry to be produced again. [amcheck.sgml#heapallindexed](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L171-L187) [verify_nbtree.c#header](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L11-L14)
2. **The fingerprint cannot answer the reverse question.** A [Bloom filter](../../../glossary.md#bloom-filter) stores only bits. Its whole interface is create, add, test for absence, report the fraction of set bits, and free. Elements "can be added to the set, but not removed". A matching heap row therefore cannot tick an entry off, and after the scan nothing can list the entries that no row matched. [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L6-L16) [bloomfilter.h#API](../../../../raw/postgres-12/src/include/lib/bloomfilter.h#L18-L25)
3. **An unmatched index entry is normally legitimate.** Many entries have no row in amcheck's scan in a healthy table, as the table below shows. An exact reverse count would therefore have to fetch each entry's heap TID and inspect the line pointer it names. amcheck's heap phase never fetches a TID; it is a sequential build scan. [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L579-L614)
4. **amcheck checks a TID's format, not its target.** Leaf data items must not carry the pivot `INDEX_ALT_TID_MASK` flag, and version-4 tiebreaker comparisons require a heap TID to be present. No check compares the block number with the table's size or the offset with a heap page. [nbtutils.c#_bt_check_natts](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L2408-L2427) [verify_nbtree.c#BTreeTupleGetHeapTIDCareful](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2559-L2573)

**Index entries with no row in amcheck's scan, in a healthy table:**

| Index entry | What its heap TID names | Why it is legitimate |
|---|---|---|
| Entry for a deleted or updated-away row not yet pruned | An `LP_NORMAL` [dead tuple](../../../glossary.md#dead-tuple) | [VACUUM](../../../glossary.md#vacuum) removes index entries in its own pass, before its second heap pass frees the slots, "because we cannot remove the tuples until we've removed their index entries". The parent check's scan skips `HEAPTUPLE_DEAD` rows; `bt_index_check`'s [MVCC](../../../glossary.md#mvcc) scan returns only rows its snapshot sees. [vacuumlazy.c#index-then-heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L764-L785) [vacuumlazy.c#lazy_vacuum_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1515-L1524) [heapam_handler.c#dead-skip](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1389-L1393) [verify_nbtree.c#snapshot](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L435-L469) |
| Entry for a row that [pruning](../../../glossary.md#pruning) removed | An `LP_DEAD` stub with no tuple | Pruning marks the root line pointer dead when the whole chain is dead; "the next regular VACUUM pass can reclaim the index entries pointing at the line pointer and then the line pointer itself". [pruneheap.c#heap_prune_chain](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L335-L338) [README.HOT#dead-root](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L116-L120) |
| Entry for a [HOT](../../../glossary.md#hot) chain whose root version was pruned | An `LP_REDIRECT` link to a later chain member | The root line pointer stays "since the index still links to that line pointer". The scan reaches the live member, and the build scan pairs it with the root TID. [README.HOT#redirect](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L74-L79) [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1666) |
| Entry kept by `VACUUM (INDEX_CLEANUP FALSE)` or `vacuum_index_cleanup = false` | An `LP_DEAD` stub that VACUUM leaves in place | **Default:** index cleanup is on; the storage parameter defaults to true and the option's documented default follows it. **Cause:** the option or storage parameter set to false. **Mechanism:** `useindex` becomes false, so `lazy_scan_heap()` forgets the dead items instead of vacuuming the heap page, and the index entries remain. [reloptions.c#vacuum_index_cleanup](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L143-L150) [vacuum.sgml#INDEX_CLEANUP](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L187-L204) [vacuumlazy.c#useindex](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L285-L286) [vacuumlazy.c#forget-dead](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1243-L1265) |

The measurement's control fixture builds the last three states: two `LP_REDIRECT` and two `LP_DEAD` line pointers whose 600 index entries all remain. amcheck passed and the dangling-entry check below reported nothing.

**A further obstacle, by inference.** `bt_index_check` holds only `AccessShareLock`, which does not conflict with the `ShareUpdateExclusiveLock` that VACUUM takes. VACUUM could therefore delete an entry and free its heap slot between amcheck's copy of a leaf page and a later heap fetch, so a reverse probe under that lock could raise a false alarm. The source does not give this as amcheck's reason; it follows from the lock table and VACUUM's index-then-heap order. [verify_nbtree.c#lock-choice](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L232-L263) [lock.c#ShareUpdateExclusiveLock](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L81) [vacuum.c#vacuum_rel-lock](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1676-L1681)

### Follow-up: checking for dangling index entries

**There is no amcheck option for it in PostgreSQL 12, but a superuser can run the check with [`pageinspect`](../../../glossary.md#pageinspect).** The idea is to read every leaf entry's heap TID from the index pages and look up the line pointer it names on the heap page, while a table lock keeps VACUUM and writers out. An ordinary index scan finds only part of the problem.

| Method | Finds | Misses | Needs |
|---|---|---|---|
| amcheck, any function or option | None of these | Every dangling entry; measured 548 of 548 | — |
| Ordinary index scan forced over the whole key range | An entry whose block is past the end of the file: `ERROR` `XX001`, `could not read block ... read only 0 of 8192 bytes` | Entries naming a missing or unused line pointer, or a heap-only tuple: skipped without a message; measured 0 rows and no error for the 26 keys on the zeroed page | Only the ordinary right to read the table; the scan stops at the first short read |
| Per-index counters `tuples_returned` minus `tuples_fetched` | A count of TIDs that reached no visible row | Cannot tell dangling entries from legitimate dead entries | See the [visible-count checker's unresolved-TID signal](sql-heap-index-visible-count-checker-plan.md#heap-side-loss-and-the-unresolved-tid-signal) |
| The `pageinspect` statement below | Block past end, no such line pointer, unused line pointer, an uninspectable normal item, and an entry that names a heap-only tuple | An entry that names a valid slot holding a different row; wrong key values; invalid redirect targets or other heap-header defects; non-B-tree indexes | Superuser, and a `SHARE ROW EXCLUSIVE` lock on the table for the whole run |

Why the index scan silently skips most cases: `heap_hot_search_buffer()` stops at a "bogus TID" whose offset exceeds the page's line pointers, at an unused or dead line pointer, and at a heap-only tuple at chain start, and returns "not found". `index_getnext_slot()` then takes the next TID. A block past the end fails earlier, in `mdread()`'s short-read check, because the buffer manager must read the block before the search. The index scan also asks B-tree to mark such entries [killed](../../../glossary.md#killed-index-tuple), so later scans may skip them; the `pageinspect` statement still reads them, because it reads raw items. [heapam.c#heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1555-L1590) [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L624-L632) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L588) [btreefuncs.c#bt_page_print_tuples](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L269-L280)

**Logic map of the `pageinspect` check:**

```text
BEGIN; LOCK TABLE ONLY t IN SHARE ROW EXCLUSIVE MODE                                          [A]
   -> no writers, VACUUM or ordinary/concurrent REINDEX on this table until COMMIT             [A]
guard: valid physical B-tree of this heap table? no -> ERROR; yes -> parent-check preflight     [B]
target: index blocks; heap path, block bytes, segment bytes                                     [B]
heap extent: stat segments from 0 through first short/missing file; count complete blocks      [B]
for index blocks 1 .. N-1:  bt_page_stats -> keep type 'l' (live leaf; skips 'd','e','i','r')     [C]
   bt_page_items -> drop item 1 when btpo_next <> 0 (the high key) -> heap TID (block, offset)   [D]
   heap block >= heap blocks? ------------------------------------------> 'block past end of heap'
   else get_raw_page(heap block) -> heap_page_items -> line pointer lp = offset?                  [E]
        no such lp (includes every lp of an all-zero page) -----------> 'no such line pointer'
        lp_flags 0 (LP_UNUSED) ---------------------------------------> 'unused line pointer'
        lp_flags 1 with NULL t_infomask2 ------------------------------> 'uninspectable normal item'
        lp_flags 1 with HEAP_ONLY_TUPLE ------------------------------> 'heap-only tuple'
        lp_flags 1 otherwise, 2 (LP_REDIRECT), 3 (LP_DEAD) -----------> not reported by these tests [F]
COMMIT
```

- **[A]** `ShareRowExclusiveLock` conflicts with `RowExclusiveLock`, `ShareUpdateExclusiveLock` and `ShareLock`. These cover writers, VACUUM and both REINDEX paths; unlike plain `ShareLock`, it also excludes ordinary REINDEX, which takes `ShareLock` on the heap. `LOCK TABLE` holds its lock until the transaction ends. [lock.c#ShareRowExclusiveLock](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L91) [vacuum.c#vacuum_rel-lock](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1676-L1681) [index.c#reindex-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L3450-L3470) [indexcmds.c#concurrent-reindex-heap-lock](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3038-L3045) [lock.sgml#lock-duration](../../../../raw/postgres-12/doc/src/sgml/ref/lock.sgml#L61-L67)
- **[B]** The guard requires a valid, ready physical B-tree attached to the locked ordinary heap table, then calls `bt_index_parent_check()` without heap comparison or root descent. Its target and main-fork tests run even when there are no leaf pages. [pg_index.h#state-and-parent](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L49) [pg_class.h#relam-and-kind](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L54-L85) [verify_nbtree.c#target-and-main-fork](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L276-L338) `get_raw_page()` rejects a block at or past `RelationGetNumberOfBlocksInFork()`. For the heap, the statement computes the complete-block extent through the first short or missing segment, matching a fresh `mdnblocks()` lookup rather than summing all later files. [rawpage.c#block-range](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L148-L152) [bufmgr.c#relation-block-count](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2799-L2835) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755)
- **[C]** `bt_page_stats()` reports `d` deleted, `e` half-dead, then `l` leaf before `r` root, so a one-page index's root leaf counts as `l`; block 0 is the metapage and is rejected. [btreefuncs.c#page-type](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L108-L122) [btreefuncs.c#metapage-reject](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L197-L199)
- **[D]** On a page that is not rightmost (`btpo_next` is not `P_NONE`, 0), item 1 is the high key, which points at no heap row. `bt_page_items()` prints each item's `t_tid` as `(block,offset)` without validating it. [nbtree.h#P_RIGHTMOST](../../../../raw/postgres-12/src/include/access/nbtree.h#L181-L188) [nbtree.h#P_HIKEY](../../../../raw/postgres-12/src/include/access/nbtree.h#L217-L219) [btreefuncs.c#bt_page_print_tuples](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L269-L280)
- **[E]** `heap_page_items()` returns one row per line pointer up to `PageGetMaxOffsetNumber()`, which is 0 for an all-zero page. It reports `t_infomask2` only for a line pointer with plausible storage. If that storage test fails, tuple-header fields are NULL, even when the line pointer claims `LP_NORMAL`. The query reports that inability to inspect instead of letting a NULL bit-test silently exclude the item. [heapfuncs.c#heap_page_items](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L166-L227) [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L357-L360) [heapfuncs.c#uninspectable-item](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L270-L279) [execExprInterp.c#NULL-qualification](../../../../raw/postgres-12/src/backend/executor/execExprInterp.c#L820-L831)
- **[F]** Line pointer values and the heap-only flag `0x8000` (32768): [itemid.h#LP_states](../../../../raw/postgres-12/src/include/storage/itemid.h#L38-L41), [htup_details.h#HEAP_ONLY_TUPLE](../../../../raw/postgres-12/src/include/access/htup_details.h#L281). An index entry points to a chain's root, never to a heap-only member. [README.HOT#heap-only](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L64-L66)

**The statement.** Replace every occurrence of `public.my_table` and `public.my_table_pkey`. Run the whole transaction as a superuser in a fresh session, with amcheck and pageinspect installed and visible on its search path. Supply an ordinary heap table or physical leaf partition, not a partitioned parent, materialized view or TOAST relation: this template uses `LOCK TABLE`, whose v12 target check rejects materialized views and TOAST relations. `ONLY` avoids recursively locking inheritance descendants; check their indexes separately. [lockcmds.c#lock-target-and-recursion](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L63-L108) The [measurement script](#script) runs the same text with fixture names substituted. An error at any stage means the check did not complete; roll back the transaction before retrying.

```sql
BEGIN;
SET LOCAL /* wiki_dangling_statement_timeout */ statement_timeout = '30min';
SET LOCAL /* wiki_dangling_lock_timeout */ lock_timeout = '5s';
LOCK /* wiki_dangling_block_writers_vacuum_and_reindex */ TABLE ONLY public.my_table IN SHARE ROW EXCLUSIVE MODE;
DO /* wiki_dangling_validate_target */ $guard$
BEGIN
    IF NOT EXISTS (
        SELECT /* wiki_dangling_target_kind */ 1
        FROM pg_index i
        JOIN pg_class c ON c.oid = i.indexrelid
        JOIN pg_am a ON a.oid = c.relam
        JOIN pg_class h ON h.oid = i.indrelid
        JOIN pg_am ha ON ha.oid = h.relam
        WHERE i.indexrelid = 'public.my_table_pkey'::regclass
          AND i.indrelid = 'public.my_table'::regclass
          AND c.relkind = 'i' AND a.amname = 'btree'
          AND h.relkind = 'r' AND ha.amname = 'heap'
          AND i.indisvalid AND i.indisready
    ) THEN
        RAISE EXCEPTION 'target must be a valid physical B-tree index of the locked heap table'
            USING ERRCODE = '42809';
    END IF;
    PERFORM /* wiki_dangling_index_preflight */
        bt_index_parent_check('public.my_table_pkey'::regclass, false, false);
END
$guard$;
WITH RECURSIVE /* wiki_dangling_index_entries */ target AS MATERIALIZED (
    SELECT i.indexrelid::regclass AS ix, i.indrelid::regclass AS tbl,
           pg_relation_size(i.indexrelid) / current_setting('block_size')::bigint AS index_blocks,
           pg_relation_filepath(i.indrelid) AS heap_path,
           current_setting('block_size')::bigint AS block_bytes,
           pg_size_bytes(current_setting('segment_size')) AS segment_bytes
    FROM pg_index i
    WHERE i.indexrelid = 'public.my_table_pkey'::regclass
), heap_segments(segno, bytes) AS (
    SELECT 0::bigint, (pg_stat_file(t.heap_path, true)).size
    FROM target t
    UNION ALL
    SELECT s.segno + 1,
           (pg_stat_file(t.heap_path || '.' || (s.segno + 1), true)).size
    FROM heap_segments s CROSS JOIN target t
    WHERE s.bytes = t.segment_bytes
), heap_extent AS MATERIALIZED (
    SELECT coalesce(sum(s.bytes / t.block_bytes), 0)::bigint AS heap_blocks
    FROM heap_segments s CROSS JOIN target t
), leaf AS MATERIALIZED (
    SELECT b.blk, s.btpo_next
    FROM target t
    CROSS JOIN LATERAL generate_series(1, t.index_blocks - 1) AS b(blk)
    CROSS JOIN LATERAL bt_page_stats(t.ix::text,
        (CASE WHEN b.blk > 2147483647 THEN b.blk - 4294967296 ELSE b.blk END)::integer) AS s
    WHERE s.type = 'l'
), entries AS MATERIALIZED (
    SELECT l.blk AS index_block, e.itemoffset, e.ctid AS heap_tid,
           split_part(btrim(e.ctid::text, '()'), ',', 1)::bigint AS heap_block,
           split_part(btrim(e.ctid::text, '()'), ',', 2)::integer AS heap_offset
    FROM target t
    CROSS JOIN leaf l
    CROSS JOIN LATERAL bt_page_items(t.ix::text,
        (CASE WHEN l.blk > 2147483647 THEN l.blk - 4294967296 ELSE l.blk END)::integer) AS e
    WHERE NOT (l.btpo_next <> 0 AND e.itemoffset = 1)
), heap_blocks AS MATERIALIZED (
    SELECT DISTINCT e.heap_block
    FROM entries e
    CROSS JOIN heap_extent t
    WHERE e.heap_block < t.heap_blocks
), heap_items AS MATERIALIZED (
    SELECT hb.heap_block, h.lp, h.lp_flags, h.t_infomask2
    FROM target t
    CROSS JOIN heap_blocks hb
    CROSS JOIN LATERAL heap_page_items(get_raw_page(t.tbl::text,
        (CASE WHEN hb.heap_block > 2147483647
              THEN hb.heap_block - 4294967296 ELSE hb.heap_block END)::integer)) AS h
)
SELECT e.index_block, e.itemoffset, e.heap_tid,
       CASE WHEN e.heap_block >= t.heap_blocks THEN 'block past end of heap'
            WHEN h.lp IS NULL THEN 'no such line pointer'
            WHEN h.lp_flags = 0 THEN 'unused line pointer'
            WHEN h.lp_flags = 1 AND h.t_infomask2 IS NULL THEN 'uninspectable normal item'
            ELSE 'heap-only tuple'
       END AS problem
FROM entries e
CROSS JOIN heap_extent t
LEFT JOIN heap_items h ON h.heap_block = e.heap_block AND h.lp = e.heap_offset
WHERE e.heap_block >= t.heap_blocks
   OR h.lp IS NULL
   OR h.lp_flags = 0
   OR (h.lp_flags = 1 AND h.t_infomask2 IS NULL)
   OR (h.lp_flags = 1 AND (h.t_infomask2 & 32768) <> 0)
ORDER BY e.index_block, e.itemoffset;
COMMIT;
```

Only after the guard, preflight and final query all succeed does an empty result mean that no examined leaf entry names a block beyond the measured heap extent, a missing or unused line pointer, an uninspectable normal item, or a heap-only tuple identifiable from the decoded header. A valid empty physical B-tree also returns no rows; a table supplied as the index, a partitioned index, a non-B-tree index, an invalid/not-ready index or a parent mismatch instead fails the guard. Each output row names the index block and item, the heap TID and the problem. These are the explicit branches of the statement above, not a claim that an empty result proves the database sound.

`LP_NORMAL` normally has storage (`lp_len > 0`). A damaged line pointer can retain that flag while losing its length, alignment or valid bounds. `heap_page_items()` then supplies the line-pointer fields but no tuple header. `uninspectable normal item` means the heap-only test could not be performed; it does not claim that a heap-only tuple was found. Conversely, a decoded header is not a complete validation: pageinspect explicitly limits its checks, and this query does not follow redirects or compare surviving row values with index keys. [itemid.h#normal-storage](../../../../raw/postgres-12/src/include/storage/itemid.h#L18-L42) [heapfuncs.c#storage-test](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L202-L227) [heapfuncs.c#NULL-fields](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L270-L279)

How the statement's parts work:

- **Settings.** `statement_timeout` and `lock_timeout` have context `user`; `SET LOCAL` limits them to this transaction. [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396)
- **Target guard.** The catalog predicate checks the index kind, access method, state and parent before any page enumeration. The [PL/pgSQL](../../../glossary.md#plpgsql) `DO` block raises `42809` on failure and discards the preflight function's void result with `PERFORM`. `DO` is executed once, not once per generated block. [do.sgml#execution](../../../../raw/postgres-12/doc/src/sgml/ref/do.sgml#L34-L44) [plpgsql.sgml#PERFORM](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L945-L973) [verify_nbtree.c#target-check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L338)
- **The lock.** Without a lock, VACUUM could free a slot after its index entry was read, and inserts could split pages mid-run. Plain `SHARE` blocks those operations but allows another `ShareLock`, which ordinary REINDEX takes on the heap before replacing the index's physical file. The preflight releases its index lock when it returns, and each pageinspect call releases its lock after copying a page. Consequently, `SHARE` alone would allow the statement to mix an old block count or leaf metadata with a rebuilt index. This transaction instead holds `SHARE ROW EXCLUSIVE` on the heap until `COMMIT`, excluding REINDEX as well as writers and VACUUM. Ordinary reads remain allowed. [index.c#reindex-locks-and-rebuild](../../../../raw/postgres-12/src/backend/catalog/index.c#L3450-L3530) [verify_nbtree.c#early-release](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L296-L303) [btreefuncs.c#stats-release](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L181-L213) [btreefuncs.c#items-release](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L369-L373) [lock.c#conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L68-L91)

  On-access [pruning](../../../glossary.md#pruning) can still run, but it only turns roots into `LP_DEAD` or `LP_REDIRECT` and frees heap-only members, which no index entry names. [pruneheap.c#heap_prune_chain](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L335-L347) [README.HOT#heap-only-reclaim](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L99-L102)
- **Heap extent, not summed file size.** Normally every segment before the last is full. External truncation can leave a short middle segment followed by intact files. `pg_relation_size()` still sums those files, but a fresh `mdnblocks()` stops at the first short segment; using the sum as a read bound can therefore make `get_raw_page()` error. The [recursive CTE](../../../glossary.md#recursive-cte) follows `.1`, `.2`, and later names only while the previous file has exactly `segment_size` bytes. `pg_stat_file(..., true)` returns null for a missing name, so missing and short files both end the recursion. Each file contributes only complete blocks. `block_size` and `segment_size` are read-only build parameters; `pg_size_bytes()` decodes the latter's displayed unit. [dbsize.c#summed-file-size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308) [md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L44-L66) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [genfile.c#pg_stat_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L362-L427) [guc.c#block-and-segment-size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2878-L2900) [dbsize.c#pg_size_bytes](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L708-L841)
- **`MATERIALIZED`.** It is accepted at this pin and keeps each `WITH` query from being folded into the main query, so `get_raw_page()` is called only for the in-range blocks that `heap_blocks` kept. [gram.y#opt_materialized](../../../../raw/postgres-12/src/backend/parser/gram.y#L11425-L11428) [select.sgml#materialized](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L320-L325)
- **TID decoding.** `btrim()` removes the parentheses and `split_part()` takes the block and the offset. [func.sgml#btrim](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L1726-L1737) [func.sgml#split_part](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L2336-L2346)
- **Block-number arguments.** PostgreSQL block numbers range from 0 through 4,294,967,294, but these pageinspect SQL arguments are signed `int4`. A direct `bigint::integer` cast fails above 2,147,483,647. The `CASE` expressions subtract 2^32 only for the upper half, preserving the 32-bit pattern that each C entry point reads with `PG_GETARG_UINT32()`. Thus block 2,147,483,648 is passed as −2,147,483,648 and block 4,294,967,294 as −2; comparisons, joins and output retain the original positive `bigint` values. This removes an artificial 16 TiB boundary at 8 KiB per block; it does not avoid the scan's I/O or lock cost. [block.h#BlockNumber](../../../../raw/postgres-12/src/include/storage/block.h#L17-L35) [pageinspect--1.5.sql#block-arguments](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L8-L26) [pageinspect--1.5.sql#btree-arguments](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L162-L189) [int8.c#int84](../../../../raw/postgres-12/src/backend/utils/adt/int8.c#L1160-L1169) [rawpage.c#get_raw_page](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L46-L63) [btreefuncs.c#bt_page_stats](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L162-L166) [btreefuncs.c#bt_page_items](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L310-L314)
- **Catalog columns.** `pg_index.indexrelid` and `indrelid` exist at the pin, and `pageinspect` 1.7 provides the four functions with the columns used. [pg_index.h#columns](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L42) [pageinspect.control](../../../../raw/postgres-12/contrib/pageinspect/pageinspect.control#L1-L5) [pageinspect--1.5.sql#heap_page_items](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L38-L55) [pageinspect--1.5.sql#bt_page_stats](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L162-L175) [pageinspect--1.5.sql#bt_page_items](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L180-L189)

**Cases and limits:**

| Condition | Behavior | Consequence |
|---|---|---|
| Caller is not a superuser | Each of the four functions raises `must be superuser ...` | Grant nothing; run as a superuser. [btreefuncs.c#stats-superuser](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L176-L179) [btreefuncs.c#items-superuser](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L320-L323) [rawpage.c#superuser](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106) [heapfuncs.c#superuser](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L135-L138) |
| Another session holds a conflicting lock for longer than 5 s | `LOCK` fails with a lock timeout | Nothing was checked; retry later. |
| Index pages are themselves damaged | The parent-check preflight can error; damage it misses may still make `bt_page_stats()` or `bt_page_items()` error or report garbage TIDs | The statement includes the preflight, but still assumes a structurally sound index; amcheck cannot prove that assumption. [verify_nbtree.c#target-and-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L276-L294) [btreefuncs.c#bt_page_print_tuples](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L271-L272) |
| An entry names an `LP_NORMAL` slot that holds some other row | Not reported | The statement checks that the slot exists, not that its row matches the key. amcheck's `heapallindexed` probe checks the opposite direction only. |
| An entry names an `LP_NORMAL` item whose storage pageinspect cannot inspect | Reported as `uninspectable normal item` rather than silently passing a NULL header test | The row's heap-only status is unknown. A non-NULL decoded header still does not prove the entire tuple valid. [heapfuncs.c#storage-test](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L202-L227) [heapfuncs.c#NULL-fields](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L270-L279) |
| Other session's temporary table | Rejected with `cannot access temporary tables of other sessions` | Run it from the owning session. [btreefuncs.c#other-temp](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L342-L350) |
| Wrong kind, partitioned parent/index, invalid/not-ready index, or index of another table | The explicit catalog guard raises `42809` before page enumeration, even if enumeration would have produced no pages | Run once per valid physical B-tree of the named ordinary heap table; no silent skip is treated as success. [pg_class.h#kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) [pg_index.h#state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L49) |
| Short middle heap segment with later files still present | The statement ends its heap extent at the short segment and reports entries at or beyond it without passing them to `get_raw_page()` | Such entries can name still-present bytes in later files, not only erased rows. The result is a boundary mismatch, not a count of physically destroyed rows. A fresh storage-manager lookup has the same boundary. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) |
| External file changes during the check, or old cached pages | SQL locks do not prevent external file writes; cached pages can hide disk changes | Stop external modification first. A fresh session avoids old per-backend segment state, but does not discard shared buffers; the statement is not a forced disk audit. [md.c#open-segment-state](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L717-L732) [rawpage.c#buffer-copy](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L159-L169) |
| Large index | Beyond the preflight's traversal, `bt_page_stats()` reads each non-metapage block once, `bt_page_items()` reads each selected leaf again, and `get_raw_page()` reads each distinct in-range referenced heap page once; `entries` materializes one row per leaf entry | Expect an index pass and much of the heap while writes wait. These are buffer accesses, not necessarily disk reads. [btreefuncs.c#stats-read](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L201-L215) [rawpage.c#buffer-copy](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L159-L169) |

**Causal summary.** A heap row vanishes (zeroed page, cut file, freed slot) → its index entry still holds the old TID → amcheck's heap scan finds no row there, so nothing probes the fingerprint for it → the Bloom filter can only answer probes and cannot list unmatched entries → this loss alone raises no amcheck error. The separate statement locks and validates the target, runs its index preflight, measures the heap through the first short or missing segment, and inspects leaf TIDs. It reports references beyond that boundary and the listed line-pointer problems, not whether every surviving slot holds the right row. The source branches and limits are mapped above.

## Measurement Script

### Results and last run

**Last successful run:** 2026-10-26, PostgreSQL 12.2, pin `45b88269a353ad93744772791feb6d01bc7e1e42`, Linux 6.18.54.1-microsoft-standard-WSL2 x86_64, GCC 13.3.0 with `CFLAGS='-O2 -std=gnu99'`. Both clusters used 8192-byte blocks, maximum alignment 8, UTF-8, C locale, session TimeZone UTC, `shared_buffers = 128MB` and `maintenance_work_mem = 64MB`. The generated build header sets the non-default `RELSEG_SIZE = 128`, so `segment_size = 1MB`; raw source is unchanged. One cluster used the default checksums-off state; the other used `initdb -k`. The fresh core/extension build took 51 seconds with `JOBS=8`. The subsequent `verify interrupt` run passed all **127 recorded checks** and both startup-shutdown tests: verification took 8 seconds, and interruption testing took 2 seconds. All fixture outcomes below reproduced. The unchanged 127-check harness includes the ten malformed-item checks added in an earlier review: intact control, NULL header, legacy query's false negative, corrected result count and exact reported item, in both checksum configurations. The fixture checks the little-endian page-header and line-pointer layout before constructing its fault image. These are measurements of this host and these small fixtures, not performance estimates for production indexes.

| Fixture / condition | Measured outcome | Source interpretation |
|---|---|---|
| `malformed_probe`: one intact row; prepare a page image with its line pointer still `LP_NORMAL` but `lp_len = 0`, then replace the heap file while stopped. In the checksums-on leg, write a recomputed checksum for that malformed image | The intact diagnostic returned 0 rows. After the fault, pageinspect returned `lp_flags = 1`, `lp_len = 0`, and NULL `t_infomask2`. The legacy query returned 0 rows; the corrected query returned exactly one finding: index block 1, item 1, heap TID `(0,1)`, `uninspectable normal item`. Both checksum legs agreed | The normal flag alone does not establish valid storage. Failed storage checks produce NULL tuple-header fields, and a NULL qualification is not true. The checksum-on fixture deliberately has a valid checksum, isolating this query defect rather than a checksum rejection. [itemid.h#normal-storage](../../../../raw/postgres-12/src/include/storage/itemid.h#L18-L42) [heapfuncs.c#storage-test](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L202-L227) [heapfuncs.c#NULL-fields](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L270-L279) [execExprInterp.c#NULL-qualification](../../../../raw/postgres-12/src/backend/executor/execExprInterp.c#L820-L831) [rawpage.c#page_checksum](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L299-L324) |
| Another session holds the table lock on `empty_probe`; attempt ordinary `REINDEX INDEX empty_probe_pkey` | Under the old `SHARE` mode, REINDEX succeeded. Under `SHARE ROW EXCLUSIVE`, it failed with `55P03`, `canceling statement due to lock timeout`, at the configured 250 ms limit. After cancelling and joining the owned holder session, REINDEX succeeded. Same in both checksum legs | Ordinary REINDEX takes `ShareLock` on the heap. That is compatible with `ShareLock`, but conflicts with `ShareRowExclusiveLock`. This measures lock exclusion, not a mixed-page corruption result during a rebuild. [index.c#reindex-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L3450-L3470) [lock.c#conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L91) |
| Map block numbers 0, 2,147,483,647, 2,147,483,648 and 4,294,967,294 to signed SQL arguments; call `get_raw_page`, `bt_page_stats` and `bt_page_items` with the last three against small relations | Mapping returned `0,2147483647,-2147483648,-2`. All nine calls per checksum leg reached pageinspect's block-range rejection rather than failing a SQL cast. `get_raw_page` reported the original positive block numbers with `22023`; both B-tree functions reported `XX000`, `block number out of range`. Same in both clusters | These entry points read the SQL argument as `uint32`. The tests establish transport and dispatch, not successful reads from a real relation above 16 TiB. [rawpage.c#get_raw_page](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L46-L63) [rawpage.c#range-error](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L148-L152) [btreefuncs.c#range-check](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L52-L55) [btreefuncs.c#stats-argument](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L162-L166) [btreefuncs.c#items-argument](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L310-L314) |
| Run the guarded statement with a table as the index argument, a partitioned index, an index of the wrong locked table, or a hash index; also check a valid empty physical B-tree | All four invalid selections raised the guard's `42809`; the valid empty B-tree completed with 0 result rows. Same in both clusters | The guard runs before page enumeration, so an empty candidate set cannot bypass target validation. Index state and parent fields come from `pg_index`; kinds and AM identity come from `pg_class` and `pg_am`. [pg_index.h#state-and-parent](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L49) [pg_class.h#kind-and-AM](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L54-L85) [pg_am.h#name](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L29-L40) |
| `middle_probe`: 16,000 rows, `fillfactor = 80`, 1 MB relation segments; stop the server and empty `.1`, retaining nonempty `.2` and later files | The intact control returned 0 findings. After the fault, `pg_relation_size() / 8192` was 465 but the fresh heap boundary was 128 blocks. A read of block 128 admitted by the old summed-size predicate failed `22023`, `block number 128 is out of range`. Parent amcheck with both options passed. The revised statement completed and reported 12,544 `block past end of heap` entries. Same in both clusters | Summed file size and the first-short-segment boundary differ. The recursive extent prevents out-of-range reads and reports references to the inaccessible suffix, including rows still physically present in later segments. [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [rawpage.c#block-range](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L148-L152) |
| Delete one of 20 rows, then force an ordinary index scan of the deleted key | `bt_page_stats()` found 1 dead index item; `bt_index_parent_check(..., true, true)` passed with checksums off and on | `LP_DEAD` with storage is valid; it is excluded from the tuple fingerprint. [nbtutils.c#killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1781-L1811) [verify_nbtree.c#line-pointer](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2531-L2548) [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1072-L1083) |
| Apply the example's inventory filter to a partitioned parent and this session's temporary table | Parent: 0 matching physical indexes, although its leaf had 1. Temporary table: 0 matches, although a direct amcheck call passed. Same results in both clusters | The query's scope restrictions are real exclusions, not evidence that the excluded objects were checked. [pg_index.h#indrelid](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L42) [pg_class.h#kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) [verify_nbtree.c#temp-target](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L325-L330) |
| Pass a table as the target | `42809`, `"toast_probe" is not an index`, in both clusters | `index_open()` fails before amcheck's B-tree-specific test. [indexam.c#index_open](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L126-L140) |
| One row has a 20,000-byte external `payload`; both its plain `id` index and its partial `id` index initially work. Stop the server and truncate only its TOAST heap file to zero bytes | With `heapallindexed = true`, the plain index still passed. The partial index with `WHERE payload LIKE 'a%'` failed with `XX001`, `could not read block 0 ... read only 0 of 8192 bytes`, in both clusters | The predicate fetches TOAST data before forming the entry. This particular fixture fails in `mdread()` before the TOAST reader can report a missing chunk. It is not a measured `XX000` missing-chunk case. [heapam_handler.c#predicate](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1604-L1626) [like.c#textlike](../../../../raw/postgres-12/src/backend/utils/adt/like.c#L284-L300) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) |
| Stop the server and change byte 512, verified to lie in free space, on a heap page containing 10 rows | An index-only amcheck call passed in both clusters. The heap-checking call passed with checksums off and failed with `XX001`, `invalid page`, with checksums on | The non-indexed free byte changes no membership entry. A cold heap read with checksums on rejects its checksum mismatch. [verify_nbtree.c#heap-gate](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L553-L614) [bufpage.c#verification](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L161) |
| Read that bad checksum page with `track_counts = off`, then with its default on state | Both reads failed. Counter: initially 0, still 0 after the off leg, then 1 after the on leg | Reporting is gated independently of page rejection. The harness waits for the collector after the enabled report. [pgstat.c#report-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556) |
| Base backup with exactly that one checksum-mismatching page | Backup failed; the client reported `checksum error occurred`. The database counter remained 1, its value from the earlier backend read | This pin sends a file's backup mismatch count to the collector only when it exceeds 1. The backend still errors for one mismatch; the client translates that checksum failure. [basebackup.c#per-file-report](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1611-L1622) [basebackup.c#final-error](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L621-L635) [pg_basebackup.c#checksum-error](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L2025-L2036) |
| Restore that page, then damage the free byte of another heap's first page and append one byte while stopped | The 8,193-byte file was copied by a successful base backup, with `could not verify checksum ... read buffer size 8193 and page size 8192 differ` | The unaligned read disables verification before testing even the complete first page in that read. Success with this warning does not establish checksum coverage or a sound backup. The fixture is not a restore test. [basebackup.c#unaligned-read](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1454-L1466) |
| Include a populated `UNLOGGED` table in that successful backup's source cluster | Its live main fork was absent from the backup; its init fork was present | File selection excludes non-init unlogged forks before checksum verification. This establishes the file exclusion, not detection of damage in the excluded heap. [basebackup.c#unlogged-exclusion](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1152) |
| `vanish_probe`: 600 rows with `fillfactor = 80`; HOT-update ids 1-2 and delete ids 3-4 on heap block 0, then `VACUUM (INDEX_CLEANUP FALSE)` | Block 0 had 2 `LP_REDIRECT` and 2 `LP_DEAD` line pointers; 600 leaf entries remained; `bt_index_parent_check(..., true, true)` passed; the dangling-entry statement returned 0 rows. Same in both clusters | Index entries that name redirect and dead stubs are legitimate and are not reported. [README.HOT#redirect](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L74-L79) [vacuumlazy.c#forget-dead](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1243-L1265) |
| Stop the server, zero heap block 1 of `vanish_probe` (26 rows, ids 27-52) and cut the file to 3 blocks (removing 522 rows on blocks 3-23) | File 24,576 bytes. `bt_index_check(..., true)` and `bt_index_parent_check(..., true, true)` both passed. Sequential count 50 of 598. A forced index scan of ids 27-52 returned 0 rows without error. A forced index scan of all ids failed with `XX001`, `could not read block 3 ... read only 0 of 8192 bytes`. The dangling-entry statement returned 548 rows: 522 `block past end of heap` and 26 `no such line pointer`. Same in both clusters | amcheck's one-way probe never sees the 548 dangling entries. The executor skips TIDs that name no line pointer and fails only on a block past the end. Checksums change nothing here: an all-zero page skips verification and a short read fails first. [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1892-L1926) [heapam.c#heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1555-L1590) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) |
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
| Prerequisites | Git, Bash, GNU make, C compiler with GNU C99 mode, the Git-source build prerequisites including Bison/Flex and Perl, and standard POSIX command-line utilities (`ps`, `grep`, `sed`, `wc`, `tr`, `head`, `dd`). No network fetch or separately installed PostgreSQL is used. [installation.sgml#build-tools](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L80-L97) [installation.sgml#git-build](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L300-L318) The malformed-item fixture requires the 8192-byte, little-endian header/bitfield layout that its SQL checks before generating a fault image; other layouts are refused, not silently patched. This is a fixture restriction, not a byte-order restriction on the diagnostic SQL. |
| Build | `--without-readline --without-zlib`, private prefix, `CFLAGS='-O2 -std=gnu99'`. Optional editing/compression libraries are omitted; C99 mode avoids modern compilers treating this pin's `bool` typedef as a C23 keyword conflict. The script sets `RELSEG_SIZE = 128` only in the out-of-tree generated `pg_config.h`, making 1 MB relation segments; the configure option accepts whole gigabytes, not this small fixture size. [installation.sgml#optional-libraries](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L118-L151) [configure.in#default-cflags](../../../../raw/postgres-12/configure.in#L408-L415) [c.h#bool](../../../../raw/postgres-12/src/include/c.h#L302-L309) [configure.in#segment-size](../../../../raw/postgres-12/configure.in#L277-L305) |
| Output | Read `.wiki-runtime/tmp/amr12/results.tsv` first; `startup-interruptions.tsv` records the separate shutdown tests. Per-check `.out`/`.err` files, `platform-off.txt`, `platform-on.txt`, `host.txt`, build logs and `server.log` give detail. `verify` replaces the current result summary and fixtures. `interrupt` saves the measurement summary while its child invocations replace the disposable cluster; it also removes the prior backup copy. `cleanup` deletes all artifacts |
| Runtime | About a minute for a fresh build and this small suite on the measured host; a few seconds for `verify` with the build retained. A run after `cleanup` rebuilds |
| Cleanup | Every successful `verify` stops its servers. A pending-startup flag is set before `pg_ctl` runs, so the EXIT trap reconciles both completed and interrupted launches. It waits briefly for a PID file if an owned startup process still exists, then stops the server and checks teardown. `cleanup` removes `.wiki-runtime/tmp/amr12/` only after ownership and teardown checks. If teardown fails, it preserves the sandbox and reports the error. It never stops a pre-existing server |

The pending flag matters because `pg_ctl` starts a separate postmaster and, where supported, detaches it from the launching process group. A startup timeout also exits the launcher without proving that the postmaster exited. The harness therefore marks the launch pending before calling `pg_ctl`, and clears that state only after ownership is confirmed or shutdown is verified. [pg_ctl.c#detached-launch](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L449-L486) [pg_ctl.c#startup-timeout](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L887-L924)

### Settings and fixtures

| Change from the normal/default state | Scope and reason | Results affected |
|---|---|---|
| Default 1 GB relation segments → `RELSEG_SIZE = 128` blocks, 1 MB at this build's 8192-byte `BLCKSZ` | Compile-time override in the disposable generated header, not a GUC or raw-source edit. It lets a small heap span several files. `segment_size` reports the compiled value and cannot be changed at runtime. With the default, this small fixture would not have a middle segment to empty; that default-size fault leg was not run. [configure.in#segment-default](../../../../raw/postgres-12/configure.in#L277-L305) [guc.c#segment_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2890-L2900) | The middle-segment fixture, its 128-block boundary and 12,544 reported entries; every other fixture also runs on this disclosed build |
| Checksums off → on with `initdb -k` in the second fresh cluster | Cluster creation; `data_checksums` is internal, not settable with `SET`. The first leg measures the default. [initdb.c#default](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1827-L1835) | Free-space-damage rejection, counters and checksum-verifying backups |
| No installed extension → amcheck and pageinspect in each disposable database | The fixture explicitly installs both; amcheck provides checks, pageinspect observes dead items and validates the fault's free-space offset. [amcheck.control](../../../../raw/postgres-12/contrib/amcheck/amcheck.control#L1-L5) [pageinspect--1.5.sql#page_header](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L20-L33) [pageinspect--1.5.sql#bt_page_stats](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L160-L175) | All amcheck/inspection results |
| `autovacuum` on → off | `sighup` setting supplied in the fresh server configuration; keeps the deleted entry available for inspection, and keeps an automatic VACUUM from cleaning `vanish_probe`'s stubs and their index entries. Manual deletion and the index scan create the measured `LP_DEAD` state. [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1433) [nbtutils.c#killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1781-L1811) | Dead-item count and valid-dead-item check; `vanish_probe` control stubs |
| `enable_seqscan`, `enable_bitmapscan`, `enable_indexonlyscan` on → off in the killed-entry probe session and the two `vanish_probe` index-scan sessions | All `user` settings; session `SET` directs each probe through an ordinary index scan, so heap liveness can request a killed-entry hint and every scanned TID is fetched from the heap. The other sessions use defaults. [guc.c#scan-switches](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L884-L921) [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L588) | Dead-item fixture; zeroed-block and past-end index-scan results |
| Heap `fillfactor` 100 → 80 on `vanish_probe` and `middle_probe` | Storage parameter (`ShareUpdateExclusiveLock`, later inserts only). It leaves room on `vanish_probe` block 0 so the two updates of the non-indexed `note` column stay on the page as HOT updates. On `middle_probe` it makes the chosen fixture span more pages; that table has no HOT updates. [rel.h#HEAP_DEFAULT_FILLFACTOR](../../../../raw/postgres-12/src/include/utils/rel.h#L278-L279) [reloptions.c#fillfactor](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L166-L176) | The `LP_REDIRECT` control stubs, `vanish_probe`'s 26-row page layout, and `middle_probe`'s segment-boundary row count |
| `VACUUM` index cleanup on → `VACUUM (INDEX_CLEANUP FALSE) vanish_probe` | Command option; `vacuum_index_cleanup` defaults to true. With it false, VACUUM prunes block 0 but forgets the dead items instead of removing their index entries, so the stubs remain referenced. [reloptions.c#vacuum_index_cleanup](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L143-L150) [vacuumlazy.c#forget-dead](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1243-L1265) | Control: 2 redirect and 2 dead stubs, 600 leaf entries, 0 dangling rows |
| No table lock → `LOCK TABLE ONLY` on the named fixture in `SHARE ROW EXCLUSIVE` mode inside each dangling-entry transaction | Transaction scope; blocks writers, VACUUM and ordinary/concurrent REINDEX while the statement reads pages one at a time; `ONLY` excludes inheritance descendants. The ordinary-REINDEX control separately holds the old `SHARE` mode, which permits that rebuild. [lock.c#conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L74-L91) [index.c#reindex-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L3450-L3470) [lockcmds.c#recursion](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L63-L79) | Stability of the dangling-entry counts; the separate lock test uses two concurrent sessions on `empty_probe` |
| Session `lock_timeout = '5s'` → `'250ms'` for the expected blocked REINDEX | `user` setting, applied only to the probing session. The holder keeps the table lock while sleeping; the test waits until `pg_locks` confirms the grant before probing. `pg_cancel_backend()` signals the owned holder after each probe; `ON_ERROR_STOP` then makes its psql session exit, and the harness waits for it. The normal 5-second limit is unchanged elsewhere. [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2389-L2397) [signalfuncs.c#pg_cancel_backend](../../../../raw/postgres-12/src/backend/storage/ipc/signalfuncs.c#L104-L126) | Bounds the lock test's wait; it does not measure production REINDEX performance |
| `track_counts` on → off for one failing read | `superuser` session setting; disables reporting without suppressing the checksum error. [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1399) [pgstat.c#report-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556) | Counter remains unchanged in the off leg |
| Text's normal extended storage → `STORAGE EXTERNAL` for `toast_probe.payload` | Column storage option; forces this large, compressible fixture out of line instead of allowing inline compression. The predicate must then fetch its chunks. [tuptoaster.c#compression-and-external-storage](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L718-L800) | TOAST-predicate result |
| Permanent table → `CREATE UNLOGGED TABLE unlogged_probe` | Default persistence is permanent. The explicit option creates an init fork; backup file selection excludes the other forks when that init file exists. [gram.y#persistence-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283) [heapam_handler.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624) [basebackup.c#unlogged-exclusion](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1152) | Successful backup contains the init fork but no live main fork for this table |
| Intact files → stopped-server truncation, free-byte overwrite, appended partial block, or a zeroed heap block plus a cut to 3 blocks | Deliberate corruption fixtures; ordinary SQL maintains the index/heap and page-format invariants. Restart makes subsequent reads cold. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L796) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) [basebackup.c#unaligned-read](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1454-L1466) | TOAST, checksum and backup fault cases; these are not normal table states |
| Valid normal heap item → `LP_NORMAL` with zero length in `malformed_probe` | Deliberate line-pointer corruption, installed only while the server is stopped. Normally that flag requires positive length. The checksums-on leg recomputes a valid checksum so the buffer read accepts the page and the test reaches pageinspect's NULL-header path; without that recomputation a checksum mismatch could reject it earlier. [itemid.h#normal-storage](../../../../raw/postgres-12/src/include/storage/itemid.h#L18-L42) [bufpage.c#generic-verification](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L124) [heapfuncs.c#NULL-fields](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L202-L227) [heapfuncs.c#invalid-storage](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L270-L279) | Legacy false-negative and corrected `uninspectable normal item` results; not a random-damage or checksum-detection-rate test |
| `statement_timeout = 0`, `lock_timeout = 0` → connection defaults `2min`, `5s`; dangling-entry transactions override the statement limit to `30min` | Both are `user` settings. The SQL wrapper applies session settings; the published diagnostic transaction uses `SET LOCAL`, so its guard, preflight and query have the longer limit until transaction end. No measured check hit either limit. [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396) [set.sgml#SET-LOCAL](../../../../raw/postgres-12/doc/src/sgml/ref/set.sgml#L52-L59) | SQL failure bounds |
| `listen_addresses = 'localhost'` → `''`, `unix_socket_directories = '/tmp'` → private `$work/s`, `unix_socket_permissions = 0777` → `0700`, `port = 5432` → `55448`, `TimeZone` (boot `'GMT'`; `initdb` writes the host zone into `postgresql.conf`) → `'UTC'`, plus `initdb`'s `128MB` `shared_buffers` override (boot default `1024` blocks = 8 MB) | Listener, socket, port and `shared_buffers` settings are `postmaster` settings chosen before start; `TimeZone` is a `user` setting appended to `postgresql.conf` for new sessions. They isolate the fixtures and format time consistently. [guc.c#shared_buffers-and-port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2199) [guc.c#timezone-and-listeners](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3905-L3960) [pg_config_manual.h#DEFAULT_PGSOCKET_DIR](../../../../raw/postgres-12/src/include/pg_config_manual.h#L182) [configure.in#default_port](../../../../raw/postgres-12/configure.in#L149-L164) [initdb.c#shared_buffers-and-timezone](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1018-L1102) [initdb.c#timezone-config](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1165-L1173) | Connection isolation and platform record; not detection semantics |
| Base-backup command uses `-X none -c fast -N` | No WAL transfer, fast [checkpoint](../../../glossary.md#checkpoint), no destination sync wait. These speed a disposable verification probe; the result is not a restore-ready backup. None requests `NOVERIFY_CHECKSUMS`. [pg_basebackup.c#options](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L330-L353) | Both backup tests |

The malformed-item fixture uses Bash and SQL only. It reads the intact page with `get_raw_page()`, creates a modified `bytea` without modifying a live buffer, encodes that image as hex, and decodes it with Bash's `printf` into a sandbox file. Only after shutdown does it copy the image over the fixture's heap file. The byte functions read or copy one byte at a zero-based offset; `encode(..., 'hex')` emits two hex characters per byte. [rawpage.c#get_raw_page](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L154-L171) [varlena.c#byteaGetByte](../../../../raw/postgres-12/src/backend/utils/adt/varlena.c#L3398-L3425) [varlena.c#byteaSetByte](../../../../raw/postgres-12/src/backend/utils/adt/varlena.c#L3464-L3493) [encode.c#binary_encode](../../../../raw/postgres-12/src/backend/utils/adt/encode.c#L35-L64) [encode.c#hex_encode](../../../../raw/postgres-12/src/backend/utils/adt/encode.c#L126-L137)

The fixture's byte calculation is conditional on its checked layout, not a portable definition of C bitfield order:

1. The first line pointer starts at byte 24. Its little-endian word must equal `lp_off + (lp_flags << 15) + (lp_len << 17)` as independently decoded by `heap_page_items()`; the fixture also checks that there is exactly one normal item and that the header's little-endian `pd_upper` matches `page_header()`. The field widths are 15, 2 and 15 bits. [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164) [itemid.h#ItemIdData](../../../../raw/postgres-12/src/include/storage/itemid.h#L25-L30) [heapfuncs.c#line-pointer-fields](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L189-L200)
2. Byte 26 retains only bit 0, and byte 27 becomes zero. Together those edits clear bits 17–31, the length, while preserving the offset and flags in bits 0–16. This creates the deliberately invalid `LP_NORMAL`/zero-length combination. [itemid.h#layout-and-normal-state](../../../../raw/postgres-12/src/include/storage/itemid.h#L25-L41)
3. With checksums on, `page_checksum(image, 0)::integer & 65535` recovers the unsigned 16-bit value from pageinspect's signed `smallint` result. The low byte goes at offset 8 and the high byte at 9, the checked little-endian `pd_checksum` field. With checksums off, that field is left unchanged. The checksum hashes the malformed image rather than proving its line pointer valid. [rawpage.c#page_checksum](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L299-L324) [bufpage.h#pd_checksum](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164) [checksum_impl.h#pg_checksum_page](../../../../raw/postgres-12/src/include/storage/checksum_impl.h#L187-L215)

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
holder_pid=0
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
    if test "$holder_pid" -ne 0; then
        wait "$holder_pid" 2>/dev/null || :
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
    sed 's/^#define RELSEG_SIZE .*/#define RELSEG_SIZE 128/' \
        "$work/build/src/include/pg_config.h" > "$work/pg_config.h.new" &&
    mv "$work/pg_config.h.new" "$work/build/src/include/pg_config.h" ||
        die "Cannot set disposable build segment size"
    grep '^#define RELSEG_SIZE 128$' "$work/build/src/include/pg_config.h" >/dev/null ||
        die "Segment-size override failed"
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
CREATE /* wiki_amr_fixture_vanish */ TABLE vanish_probe(id integer PRIMARY KEY, note integer, pad text)
WITH (fillfactor = 80);
INSERT /* wiki_amr_fixture_vanish_rows */ INTO vanish_probe
SELECT g, 0, repeat('v',200) FROM generate_series(1,600) g;
UPDATE /* wiki_amr_fixture_vanish_hot */ vanish_probe SET note = 1 WHERE id IN (1,2);
DELETE /* wiki_amr_fixture_vanish_dead */ FROM vanish_probe WHERE id IN (3,4);
VACUUM /* wiki_amr_fixture_vanish_prune */ (INDEX_CLEANUP FALSE) vanish_probe;
CREATE /* wiki_amr_fixture_hash */ INDEX vanish_probe_hash ON vanish_probe USING hash(id);
CREATE /* wiki_amr_fixture_empty */ TABLE empty_probe(id integer PRIMARY KEY);
CREATE /* wiki_amr_fixture_malformed */ TABLE malformed_probe(id integer PRIMARY KEY);
INSERT /* wiki_amr_fixture_malformed_row */ INTO malformed_probe VALUES (1);
CREATE /* wiki_amr_fixture_middle */ TABLE middle_probe(id integer PRIMARY KEY, pad text)
WITH (fillfactor = 80);
INSERT /* wiki_amr_fixture_middle_rows */ INTO middle_probe
SELECT g, repeat('m',200) FROM generate_series(1,16000) g;
SQL
    test "$?" -eq 0 || die "Fixture creation failed"
    sql -c "SELECT /* wiki_amr_platform */ version(), current_setting('block_size'),
        current_setting('data_checksums'), current_setting('shared_buffers'),
        current_setting('maintenance_work_mem'), current_setting('track_counts'),
        current_setting('server_encoding'), current_setting('lc_collate'),
        current_setting('TimeZone'), current_setting('segment_size')" > "$work/platform-$mode.txt" ||
        die "Platform query failed"
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
# The same dangling-entry statement as the Answer, with the fixture's names.
dangling_sql() {
    cat <<'SQL'
BEGIN;
SET LOCAL /* wiki_dangling_statement_timeout */ statement_timeout = '30min';
SET LOCAL /* wiki_dangling_lock_timeout */ lock_timeout = '5s';
LOCK /* wiki_dangling_block_writers_vacuum_and_reindex */ TABLE ONLY vanish_probe IN SHARE ROW EXCLUSIVE MODE;
DO /* wiki_dangling_validate_target */ $guard$
BEGIN
    IF NOT EXISTS (
        SELECT /* wiki_dangling_target_kind */ 1
        FROM pg_index i
        JOIN pg_class c ON c.oid = i.indexrelid
        JOIN pg_am a ON a.oid = c.relam
        JOIN pg_class h ON h.oid = i.indrelid
        JOIN pg_am ha ON ha.oid = h.relam
        WHERE i.indexrelid = 'vanish_probe_pkey'::regclass
          AND i.indrelid = 'vanish_probe'::regclass
          AND c.relkind = 'i' AND a.amname = 'btree'
          AND h.relkind = 'r' AND ha.amname = 'heap'
          AND i.indisvalid AND i.indisready
    ) THEN
        RAISE EXCEPTION 'target must be a valid physical B-tree index of the locked heap table'
            USING ERRCODE = '42809';
    END IF;
    PERFORM /* wiki_dangling_index_preflight */
        bt_index_parent_check('vanish_probe_pkey'::regclass, false, false);
END
$guard$;
WITH RECURSIVE /* wiki_dangling_index_entries */ target AS MATERIALIZED (
    SELECT i.indexrelid::regclass AS ix, i.indrelid::regclass AS tbl,
           pg_relation_size(i.indexrelid) / current_setting('block_size')::bigint AS index_blocks,
           pg_relation_filepath(i.indrelid) AS heap_path,
           current_setting('block_size')::bigint AS block_bytes,
           pg_size_bytes(current_setting('segment_size')) AS segment_bytes
    FROM pg_index i
    WHERE i.indexrelid = 'vanish_probe_pkey'::regclass
), heap_segments(segno, bytes) AS (
    SELECT 0::bigint, (pg_stat_file(t.heap_path, true)).size
    FROM target t
    UNION ALL
    SELECT s.segno + 1,
           (pg_stat_file(t.heap_path || '.' || (s.segno + 1), true)).size
    FROM heap_segments s CROSS JOIN target t
    WHERE s.bytes = t.segment_bytes
), heap_extent AS MATERIALIZED (
    SELECT coalesce(sum(s.bytes / t.block_bytes), 0)::bigint AS heap_blocks
    FROM heap_segments s CROSS JOIN target t
), leaf AS MATERIALIZED (
    SELECT b.blk, s.btpo_next
    FROM target t
    CROSS JOIN LATERAL generate_series(1, t.index_blocks - 1) AS b(blk)
    CROSS JOIN LATERAL bt_page_stats(t.ix::text,
        (CASE WHEN b.blk > 2147483647 THEN b.blk - 4294967296 ELSE b.blk END)::integer) AS s
    WHERE s.type = 'l'
), entries AS MATERIALIZED (
    SELECT l.blk AS index_block, e.itemoffset, e.ctid AS heap_tid,
           split_part(btrim(e.ctid::text, '()'), ',', 1)::bigint AS heap_block,
           split_part(btrim(e.ctid::text, '()'), ',', 2)::integer AS heap_offset
    FROM target t
    CROSS JOIN leaf l
    CROSS JOIN LATERAL bt_page_items(t.ix::text,
        (CASE WHEN l.blk > 2147483647 THEN l.blk - 4294967296 ELSE l.blk END)::integer) AS e
    WHERE NOT (l.btpo_next <> 0 AND e.itemoffset = 1)
), heap_blocks AS MATERIALIZED (
    SELECT DISTINCT e.heap_block
    FROM entries e
    CROSS JOIN heap_extent t
    WHERE e.heap_block < t.heap_blocks
), heap_items AS MATERIALIZED (
    SELECT hb.heap_block, h.lp, h.lp_flags, h.t_infomask2
    FROM target t
    CROSS JOIN heap_blocks hb
    CROSS JOIN LATERAL heap_page_items(get_raw_page(t.tbl::text,
        (CASE WHEN hb.heap_block > 2147483647
              THEN hb.heap_block - 4294967296 ELSE hb.heap_block END)::integer)) AS h
)
SELECT e.index_block, e.itemoffset, e.heap_tid,
       CASE WHEN e.heap_block >= t.heap_blocks THEN 'block past end of heap'
            WHEN h.lp IS NULL THEN 'no such line pointer'
            WHEN h.lp_flags = 0 THEN 'unused line pointer'
            WHEN h.lp_flags = 1 AND h.t_infomask2 IS NULL THEN 'uninspectable normal item'
            ELSE 'heap-only tuple'
       END AS problem
FROM entries e
CROSS JOIN heap_extent t
LEFT JOIN heap_items h ON h.heap_block = e.heap_block AND h.lp = e.heap_offset
WHERE e.heap_block >= t.heap_blocks
   OR h.lp IS NULL
   OR h.lp_flags = 0
   OR (h.lp_flags = 1 AND h.t_infomask2 IS NULL)
   OR (h.lp_flags = 1 AND (h.t_infomask2 & 32768) <> 0)
ORDER BY e.index_block, e.itemoffset;
COMMIT;
SQL
}
dangling() {
    dangling_for "$1" vanish_probe vanish_probe_pkey ||
        die "$1 failed; see its .err file"
}
dangling_for() {
    local legacy=${4:-false}
    dangling_sql |
        sed -e "s/'vanish_probe_pkey'/'$3'/g" \
            -e "s/'vanish_probe'/'$2'/g" \
            -e "s/TABLE ONLY vanish_probe IN/TABLE ONLY $2 IN/" |
        {
            if test "$legacy" = true; then
                sed '/h.lp_flags = 1 AND h.t_infomask2 IS NULL/d'
            else
                cat
            fi
        } |
        sql > "$work/$1.out" 2> "$work/$1.err"
}
malformed_before() {
    local mode=$1 value hex i
    dangling_for "$mode-malformed-control" malformed_probe malformed_probe_pkey ||
        die "Malformed-item intact control failed"
    value=$(wc -l < "$work/$mode-malformed-control.out" | tr -d ' ')
    expect "$mode-malformed-control-rows" 0 "$value"
    value=$(sql -c "WITH /* wiki_amr_malformed_layout */ p AS MATERIALIZED
        (SELECT get_raw_page('malformed_probe',0) AS raw)
        SELECT count(*)=1 AND bool_and(h.lp=1 AND h.lp_flags=1 AND h.lp_len>0
            AND ph.lower=28
            AND get_byte(raw,14)+256*get_byte(raw,15)=ph.upper
            AND get_byte(raw,24)::bigint+256*get_byte(raw,25)::bigint
                +65536*get_byte(raw,26)::bigint+16777216*get_byte(raw,27)::bigint
                =h.lp_off::bigint+(h.lp_flags::bigint<<15)+(h.lp_len::bigint<<17))
        FROM p, LATERAL heap_page_items(raw) h, LATERAL page_header(raw) ph") ||
        die "Cannot check malformed-item fixture layout"
    test "$value" = t || die "Malformed-item fixture requires the checked little-endian layout"
    hex=$(sql -c "WITH /* wiki_amr_malformed_image */ p AS MATERIALIZED
        (SELECT get_raw_page('malformed_probe',0) AS raw),
        damaged AS MATERIALIZED
        (SELECT set_byte(set_byte(raw,26,get_byte(raw,26)&1),27,0) AS raw FROM p),
        checksum AS MATERIALIZED
        (SELECT raw, page_checksum(raw,0)::integer & 65535 AS csum FROM damaged)
        SELECT encode(CASE WHEN '$mode'='on'
            THEN set_byte(set_byte(raw,8,csum&255),9,csum>>8)
            ELSE raw END,'hex') FROM checksum") ||
        die "Cannot prepare malformed-item image"
    test "${#hex}" -eq 16384 || die "Malformed-item image is not one 8192-byte page"
    {
        for ((i=0; i<${#hex}; i+=2)); do
            printf '%b' "\\x${hex:i:2}" || die "Cannot decode malformed-item image"
        done
    } > "$work/$mode-malformed.bad" || die "Cannot write malformed-item image"
    test "$(wc -c < "$work/$mode-malformed.bad" | tr -d ' ')" = 8192 ||
        die "Malformed-item image length is wrong"
}
malformed_after() {
    local mode=$1 value
    value=$(sql -c "SELECT /* wiki_amr_malformed_header */ lp_flags || ',' || lp_len
        || ',' || (t_infomask2 IS NULL)::text
        FROM heap_page_items(get_raw_page('malformed_probe',0)) WHERE lp=1") ||
        die "Malformed-item inspection failed"
    expect "$mode-malformed-header" '1,0,true' "$value"
    dangling_for "$mode-malformed-legacy" malformed_probe malformed_probe_pkey true ||
        die "Legacy malformed-item check failed"
    value=$(wc -l < "$work/$mode-malformed-legacy.out" | tr -d ' ')
    expect "$mode-malformed-legacy-rows" 0 "$value"
    dangling_for "$mode-malformed-fixed" malformed_probe malformed_probe_pkey ||
        die "Corrected malformed-item check failed"
    value=$(wc -l < "$work/$mode-malformed-fixed.out" | tr -d ' ')
    expect "$mode-malformed-fixed-rows" 1 "$value"
    value=$(cat "$work/$mode-malformed-fixed.out")
    expect "$mode-malformed-fixed-item" '1|1|(0,1)|uninspectable normal item' "$value"
}
target_checks() {
    local mode=$1 value name tbl ix
    for name in table partitioned wrong-parent hash; do
        case "$name" in
            table) tbl=vanish_probe; ix=vanish_probe ;;
            partitioned) tbl=parent_probe; ix=parent_probe_idx ;;
            wrong-parent) tbl=checksum_probe; ix=vanish_probe_pkey ;;
            hash) tbl=vanish_probe; ix=vanish_probe_hash ;;
        esac
        if dangling_for "$mode-target-$name" "$tbl" "$ix"; then
            die "$name target unexpectedly passed"
        fi
        grep '42809:.*target must be a valid physical B-tree' \
            "$work/$mode-target-$name.err" >/dev/null || die "Wrong target error"
        record "$mode-target-$name-rejected"
    done
    dangling_for "$mode-empty-control" empty_probe empty_probe_pkey ||
        die "Empty physical B-tree check failed"
    value=$(wc -l < "$work/$mode-empty-control.out" | tr -d ' ')
    expect "$mode-empty-control-rows" 0 "$value"
    value=$(sql -c "SELECT /* wiki_amr_segment_size */ current_setting('segment_size')") ||
        die "Segment size query failed"
    expect "$mode-segment-size" 1MB "$value"
}
boundary_checks() {
    local mode=$1 value block arg
    value=$(sql -c "SELECT /* wiki_amr_block_argument_mapping */
        string_agg((CASE WHEN b > 2147483647 THEN b - 4294967296 ELSE b END)::integer::text,
                   ',' ORDER BY b)
        FROM (VALUES (0::bigint), (2147483647::bigint),
                     (2147483648::bigint), (4294967294::bigint)) v(b)") ||
        die "Block argument mapping failed"
    expect "$mode-block-argument-mapping" '0,2147483647,-2147483648,-2' "$value"
    for block in 2147483647 2147483648 4294967294; do
        arg="(CASE WHEN $block::bigint > 2147483647
                   THEN $block::bigint - 4294967296 ELSE $block::bigint END)::integer"
        bad "$mode-raw-boundary-$block" "22023:.*block number $block is out of range" \
            "SELECT /* wiki_amr_raw_block_boundary */ get_raw_page('empty_probe', $arg)"
        bad "$mode-stats-boundary-$block" 'XX000:.*block number out of range' \
            "SELECT /* wiki_amr_stats_block_boundary */ * FROM bt_page_stats('empty_probe_pkey', $arg)"
        bad "$mode-items-boundary-$block" 'XX000:.*block number out of range' \
            "SELECT /* wiki_amr_items_block_boundary */ * FROM bt_page_items('empty_probe_pkey', $arg)"
    done
}
reindex_lock_checks() {
    local mode=$1 lockmode catalogmode label pid value i rc
    for label in share share-row-exclusive; do
        if test "$label" = share; then
            lockmode=SHARE
            catalogmode=ShareLock
        else
            lockmode='SHARE ROW EXCLUSIVE'
            catalogmode=ShareRowExclusiveLock
        fi
        sql -c "BEGIN;
            LOCK /* wiki_amr_reindex_holder */ TABLE ONLY empty_probe IN $lockmode MODE;
            SELECT /* wiki_amr_hold_lock */ pg_sleep(30);
            ROLLBACK" > "$work/$mode-$label-holder.out" \
            2> "$work/$mode-$label-holder.err" &
        holder_pid=$!
        pid=
        for i in {1..10}; do
            pid=$(sql -c "SELECT /* wiki_amr_wait_for_holder */ l.pid
                FROM pg_locks l JOIN pg_stat_activity a USING (pid)
                WHERE l.relation='empty_probe'::regclass
                  AND l.mode='$catalogmode' AND l.granted
                  AND l.pid<>pg_backend_pid()
                  AND a.query LIKE '%wiki_amr_reindex_holder%'") ||
                die "Cannot observe lock holder"
            test -n "$pid" && break
            kill -0 "$holder_pid" 2>/dev/null || die "Lock holder exited early"
            sleep 1
        done
        case "$pid" in ''|*[!0-9]*) die "Cannot identify owned lock holder";; esac
        if test "$label" = share; then
            ok "$mode-share-allows-reindex" \
                "REINDEX /* wiki_amr_share_control */ INDEX empty_probe_pkey"
        else
            bad "$mode-share-row-exclusive-blocks-reindex" '55P03:.*lock timeout' \
                "SET /* wiki_amr_probe_lock_timeout */ lock_timeout='250ms';
                 REINDEX /* wiki_amr_reindex_blocked */ INDEX empty_probe_pkey"
        fi
        value=$(sql -c "SELECT /* wiki_amr_release_holder */ pg_cancel_backend($pid)") ||
            die "Cannot cancel owned lock holder"
        test "$value" = t || die "Owned holder was not signalled"
        wait "$holder_pid"
        rc=$?
        holder_pid=0
        test "$rc" -ne 0 &&
        grep '57014:.*canceling statement due to user request' \
            "$work/$mode-$label-holder.err" >/dev/null ||
            die "Lock holder did not exit on cancellation"
    done
    ok "$mode-reindex-after-unlock" \
        "REINDEX /* wiki_amr_reindex_after_unlock */ INDEX empty_probe_pkey"
}
middle_before() {
    local mode=$1 value
    value=$(sql -c "SELECT /* wiki_amr_middle_lost_rows */ count(*)
        FROM middle_probe
        WHERE split_part(btrim(ctid::text,'()'),',',1)::bigint >= 128") ||
        die "Middle-segment layout query failed"
    test "$value" -gt 0 || die "Middle fixture does not cross a segment"
    printf '%s\n' "$value" > "$work/$mode-middle-lost.txt" || die "Cannot save layout"
    dangling_for "$mode-middle-control" middle_probe middle_probe_pkey ||
        die "Middle control check failed"
    value=$(wc -l < "$work/$mode-middle-control.out" | tr -d ' ')
    expect "$mode-middle-control-rows" 0 "$value"
}
middle_after() {
    local mode=$1 file=$2 expected value
    expected=$(cat "$work/$mode-middle-lost.txt") || die "Cannot read middle layout"
    test -s "$file.2" || die "Middle fixture lacks a surviving later segment"
    value=$(sql -c "SELECT /* wiki_amr_middle_physical_pages */
        pg_relation_size('middle_probe') / 8192") || die "Physical size query failed"
    test "$value" -gt 128 || die "Physical size does not exceed logical boundary"
    record "$mode-middle-physical-pages=$value"
    bad "$mode-old-bound-out-of-range" '22023:.*block number 128 is out of range' \
        "SELECT /* wiki_amr_old_bound */ get_raw_page('middle_probe',128)
         WHERE 128 < pg_relation_size('middle_probe') / 8192"
    ok "$mode-middle-amcheck-passes" \
        "SELECT /* wiki_amr_middle_amcheck */ bt_index_parent_check('middle_probe_pkey',true,true)"
    dangling_for "$mode-middle-fault" middle_probe middle_probe_pkey ||
        die "Middle-segment check failed"
    value=$(grep -c 'block past end of heap' "$work/$mode-middle-fault.out")
    expect "$mode-middle-past-end-rows" "$expected" "$value"
    value=$(wc -l < "$work/$mode-middle-fault.out" | tr -d ' ')
    expect "$mode-middle-total-rows" "$expected" "$value"
}
vanish_before() {
    local mode=$1 value
    value=$(sql -c "SELECT /* wiki_amr_vanish_stubs */ count(*) FILTER (WHERE lp_flags=2)
        || ',' || count(*) FILTER (WHERE lp_flags=3)
        FROM heap_page_items(get_raw_page('vanish_probe',0))") || die "Stub query failed"
    expect "$mode-vanish-block0-redirect-dead-stubs" 2,2 "$value"
    value=$(sql -c "SELECT /* wiki_amr_vanish_leaf_entries */ count(*)
        FROM generate_series(1, pg_relation_size('vanish_probe_pkey')/8192 - 1) b(blk),
        LATERAL bt_page_stats('vanish_probe_pkey', blk::integer) s,
        LATERAL bt_page_items('vanish_probe_pkey', blk::integer) e
        WHERE s.type='l' AND NOT (s.btpo_next<>0 AND e.itemoffset=1)") ||
        die "Leaf entry query failed"
    expect "$mode-vanish-leaf-entries" 600 "$value"
    ok "$mode-vanish-control-amcheck" "SELECT /* wiki_amr_vanish_control */ bt_index_parent_check('vanish_probe_pkey',true,true)"
    dangling "$mode-dangling-control"
    value=$(wc -l < "$work/$mode-dangling-control.out" | tr -d ' ')
    expect "$mode-dangling-control-rows" 0 "$value"
    sql -c "SELECT /* wiki_amr_vanish_layout */ count(*) FILTER (WHERE b=1),
        count(*) FILTER (WHERE b>=3), min(id) FILTER (WHERE b=1),
        max(id) FILTER (WHERE b=1), count(*)
        FROM (SELECT id, split_part(btrim(ctid::text,'()'),',',1)::integer AS b
              FROM vanish_probe) s" > "$work/$mode-vanish-layout.txt" ||
        die "Layout query failed"
}
vanish_fault() {
    dd if=/dev/zero of="$1" bs=8192 seek=1 count=1 conv=notrunc 2>/dev/null &&
    dd if=/dev/null of="$1" bs=8192 seek=3 2>/dev/null ||
        die "Vanish fault injection failed"
}
vanish_after() {
    local mode=$1 file=$2 layout n1 n3 lo hi total value
    layout=$(cat "$work/$mode-vanish-layout.txt") || die "Missing layout"
    n1=$(printf '%s\n' "$layout" | cut -d'|' -f1)
    n3=$(printf '%s\n' "$layout" | cut -d'|' -f2)
    lo=$(printf '%s\n' "$layout" | cut -d'|' -f3)
    hi=$(printf '%s\n' "$layout" | cut -d'|' -f4)
    total=$(printf '%s\n' "$layout" | cut -d'|' -f5)
    record "$mode-vanish-layout=block1:$n1,blocks3plus:$n3,ids:$lo-$hi,rows:$total"
    value=$(wc -c < "$file" | tr -d ' ') || die "Cannot read heap size"
    expect "$mode-vanish-heap-bytes" 24576 "$value"
    ok "$mode-vanish-parent-check-passes" "SELECT /* wiki_amr_vanish_parent */ bt_index_parent_check('vanish_probe_pkey',true,true)"
    ok "$mode-vanish-index-check-passes" "SELECT /* wiki_amr_vanish_check */ bt_index_check('vanish_probe_pkey',true)"
    value=$(sql -c "SELECT /* wiki_amr_vanish_seq */ count(*) FROM vanish_probe") ||
        die "Sequential count failed"
    expect "$mode-vanish-seq-count" "$((total - n1 - n3))" "$value"
    value=$(sql -c "SET /* wiki_amr_vanish_noseq */ enable_seqscan=off" \
        -c "SET /* wiki_amr_vanish_nobitmap */ enable_bitmapscan=off" \
        -c "SET /* wiki_amr_vanish_noios */ enable_indexonlyscan=off" \
        -c "SELECT /* wiki_amr_vanish_zeroed_ids */ count(*) FROM vanish_probe WHERE id BETWEEN $lo AND $hi") ||
        die "Zeroed-block index scan failed"
    expect "$mode-vanish-index-scan-zeroed-block-rows" 0 "$value"
    bad "$mode-vanish-index-scan-past-end-error" 'XX001:.*could not read block' \
        "SET /* wiki_amr_vanish_noseq */ enable_seqscan=off;
         SET /* wiki_amr_vanish_nobitmap */ enable_bitmapscan=off;
         SET /* wiki_amr_vanish_noios */ enable_indexonlyscan=off;
         SELECT /* wiki_amr_vanish_all_ids */ count(*) FROM vanish_probe WHERE id > 0"
    dangling "$mode-dangling-fault"
    value=$(grep -c 'block past end of heap' "$work/$mode-dangling-fault.out")
    expect "$mode-dangling-block-past-end" "$n3" "$value"
    value=$(grep -c 'no such line pointer' "$work/$mode-dangling-fault.out")
    expect "$mode-dangling-no-line-pointer" "$n1" "$value"
    value=$(wc -l < "$work/$mode-dangling-fault.out" | tr -d ' ')
    expect "$mode-dangling-total" "$((n1 + n3))" "$value"
}
verify_mode() {
    local mode=$1 checksumfile partialfile unloggedfile toastrel toastfile vanishfile middlefile malformedfile value
    init_cluster "$mode"
    common_checks "$mode"
    target_checks "$mode"
    boundary_checks "$mode"
    reindex_lock_checks "$mode"
    vanish_before "$mode"
    middle_before "$mode"
    malformed_before "$mode"
    malformedfile=$(relation_path malformed_probe) || die "Missing malformed-item path"
    vanishfile=$(relation_path vanish_probe) || die "Missing vanish path"
    middlefile=$(relation_path middle_probe) || die "Missing middle path"
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
    vanish_fault "$vanishfile"
    test -s "$middlefile.1" && test -s "$middlefile.2" ||
        die "Missing middle fixture segments"
    : > "$middlefile.1" || die "Cannot empty middle segment"
    cp "$work/$mode-malformed.bad" "$malformedfile" || die "Malformed-item injection failed"
    start
    malformed_after "$mode"
    vanish_after "$mode" "$vanishfile"
    middle_after "$mode" "$middlefile"
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

- Static review and requested rerun (2026-10-26): found no new actionable defect in the page, diagnostic SQL or published harness. Checked all 830 source citations for matching-version paths and valid line ranges, wiki/internal anchors, the diagnostic's two copies and Bash syntax. A fresh build and one `verify interrupt` run passed all 127 checks and both signal tests, reproducing the filed fixture outcomes. Updated only the latest run record and review context; confirmed teardown and removed the sandbox and extracted runner.
- Remeasurement after the history-note review (2026-10-26): extracted the unchanged published harness, built from the same pin, and ran both checksum configurations twice plus both startup-interruption tests per run. Both 127-check records matched. Updated the last-run timings; fixture outcomes and executable code remain unchanged. Confirmed shutdown and removed the owned sandbox, build, extracted runner and session logs.
- Static consistency review (2026-10-26): checked the short-read zero-fill branch and checksum-report gates against the diagram, settings table and existing measurements; distinguished copied links during concurrent deletion from a later walk of an already-unlinked index; checked the silent cross-page race return separately from `DEBUG1` messages. The published measurement script and run record were not changed or rerun. [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L614-L629) [pgstat.c#reporting-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1545) [verify_nbtree.c#copied-links-and-deletion](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1327-L1359) [verify_nbtree.c#silent-race-return](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1235-L1246)
- Cross-page follow-up (2026-10-26): corrected the [emptied-segment page](amcheck-emptied-segments-with-checksums.md#why-the-index-scenarios-fail) to distinguish its measured linked-page fixtures from an unlinked deleted-page tail, and the [health checklist](../server-administration/database-health-checklist.md#observability-prerequisites) to distinguish suppressed reports from existing counter history and checksum-disabled NULLs. These two previously reported wiki contradictions are resolved; the deleted-page-tail case remains a source reading, not a new measurement. [verify_nbtree.c#deleted-pages-unlinked](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L685-L697) [pgstat.c#reporting-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1545) [pgstatfuncs.c#checksum-getter](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1529-L1544)
- amcheck module: `verify_nbtree.c` in full (entry points, `BtreeCheckState`, locks, target checks, the level walk, per-page checks, downlink and missing-downlink checks, the heap callback, normalization, root descent, the invariant helpers, page copying, line-pointer checks), the three SQL scripts, the control file, the Makefile, the regression SQL and `amcheck.sgml`.
- Core boundary: `index_open()`, `table_beginscan_strat()`, `table_index_build_scan()`, `heapam_index_build_range_scan()`, `heapgetpage()`, `index_form_tuple()`, `heap_tuple_fetch_attr()`, `textlike()` and detoast macros, `_bt_checkpage()`, `_bt_heapkeyspace()`, `_bt_killitems()`, the lock conflict table and recovery lock limit, `bloomfilter.c`.
- Checksums: `ReadBuffer_common()` read and hit paths, `FlushBuffer()`, `PageIsVerified()`, `PageSetChecksumCopy()`, `pg_checksum_page()`, `mdread()` and process-local `InRecovery`, `DataChecksumsEnabled()`, `XLogHintBitIsNeeded()`, `MarkBufferDirtyHint()` and its caller conditions, `pgstat_report_checksum_failures_in_db()`, the relevant GUC definitions, `initdb -k`, `pg_checksums`, base backup file selection and verification including unlogged/temporary exclusions, unaligned reads and per-file counters, client error translation, and `checksum_failures` docs.
- Settings and SQL: `maintenance_work_mem`, `statement_timeout`, `lock_timeout`, `default_transaction_isolation`, `STRICT` semantics, extension `superuser` default, catalog columns and the `oid`-to-`regclass` cast.
- Build/generated headers: `BLCKSZ` and `MAXIMUM_ALIGNOF` from `configure.in`; no generated header is specific to amcheck.
- History: this page makes no source-history claim; its behavioral explanations use the pinned source tree.
- Follow-up on dangling index entries: the amcheck header comment and docs principle, the fingerprint and callback, `BTreeTupleGetHeapTIDCareful()` and `_bt_check_natts()`'s TID-format checks, the Bloom filter API header, `heap_hot_search_buffer()`, `index_getnext_slot()`, `mdread()`'s short-read path, `heap_prune_chain()`, `README.HOT` on redirects, dead roots and heap-only tuples, `lazy_scan_heap()`'s index-cleanup-disabled branch, `lazy_vacuum_heap()`'s ordering comment, `vacuum_rel()`'s lock mode, the lock conflict table, `vacuum_index_cleanup` and `fillfactor` reloptions, `pageinspect`'s `bt_page_stats()`, `bt_page_items()`, `heap_page_items()`, `get_raw_page()` privilege and range checks and SQL declarations, `MATERIALIZED` grammar and docs, `btrim()`/`split_part()` docs, `LOCK TABLE` docs, and the [visible-count checker's unresolved-TID section](sql-heap-index-visible-count-checker-plan.md#heap-side-loss-and-the-unresolved-tid-signal).
- Measurement: the complete script above, two matching 127-check result records, both startup-interruption results from each run and both platform records; expected API errors in the upstream regression output; pageinspect's header and B-tree statistics fields; `pg_ctl` launch, detachment and startup-timeout paths.
- Uninspectable-item and permission review: `ItemIdData`'s storage convention, pageinspect's storage checks and NULL header output, executor NULL qualification, bytea byte access and hex encoding, `page_checksum()`'s signed return, and the page-header/checksum layouts. Compared both diagnostic copies and reproduced the legacy false negative with a checksum-valid malformed image. Checked the connected-database boundary, per-database extension registration and target exclusions independently of relation ACL checks.
- REINDEX concurrency review: ordinary `reindex_index()` takes a heap `ShareLock` and an index `AccessExclusiveLock`, then changes the relfilenode; the concurrent path holds `ShareUpdateExclusiveLock` on the heap. Checked these against the lock conflict table, the amcheck wrapper's early release, both pageinspect lock lifetimes, and `relation_openrv()`'s invalidation processing. The harness now tests ordinary REINDEX with both table-lock modes and after release. The linked Pivot tuple glossary entry now has PostgreSQL 12 definitions from the nbtree README, tuple-format header and negative-infinity comparison branches.
- Block-number and qualification review: `BlockNumber`'s unsigned range, `int84()`'s signed overflow check, all three pageinspect argument readers and range checks, checksum XOR followed by modulo reduction, and the linked Hint bits glossary entry's visibility-map consequence against `SetHintBits()`, `lazy_scan_heap()` and the corrected leaf-density page.
- Guarded-query revision: `pg_index` state/parent fields, `pg_class` and `pg_am`, `LockTableCommand()` and its allowed relation kinds, PL/pgSQL `DO` and `PERFORM`, `pg_relation_size()`'s segment sum versus `mdnblocks()`'s first-short-segment boundary, `get_raw_page()`'s block-range check, `pg_stat_file()`'s missing-file behavior, `pg_size_bytes()`, and the generated `RELSEG_SIZE`/internal `segment_size` boundary. The in-page statement and fixture copy were compared after name substitution.
- Comparison set for the contradiction check: the whole page including the script and results; the [zeroed-segment amcheck page](amcheck-zeroed-relation-segments.md), the [corruption-log page](../observability/corruption-log-messages-with-checksums-disabled.md), the [post-drop health page](../server-administration/health-after-zeroed-schema-drop.md), the [database-health checklist](../server-administration/database-health-checklist.md), related heap/visible-count and zeroed-file claims found by symbol searches, and the B-tree bloat page's contrib-boundary statement; for the follow-up, the [visible-count checker](sql-heap-index-visible-count-checker-plan.md), the [coverage page](../storage-and-vacuum/checksums-and-amcheck-detection-coverage.md) and the [emptied-segment page](amcheck-emptied-segments-with-checksums.md), found by searching for one-way, dangling, vanished and unresolved-TID claims; the glossary entries linked here; this page's index entries and the v12 manifest row. The subsequently authorized cross-page fixes corrected the [leaf-density page's hint-bit explanation](leaf-density-vs-fragmentation-index-scan-io.md#reproduction) and the health checklist's checksum-file coverage claim. PostgreSQL 12 has no common-concept pages.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| An uninspectable normal item must not silently pass the heap-only test | [itemid.h#normal-storage](../../../../raw/postgres-12/src/include/storage/itemid.h#L18-L42), [heapfuncs.c#storage-test](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L202-L227), [heapfuncs.c#NULL-fields](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L270-L279), [execExprInterp.c#NULL-qualification](../../../../raw/postgres-12/src/backend/executor/execExprInterp.c#L820-L831); measured legacy 0 findings versus corrected 1 in both checksum legs |
| Function execution grants do not bypass target or database boundaries | [verify_nbtree.c#target-exclusions](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L306-L338), [namespace.c#database-boundary](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L241-L251), [contrib/README#installation](../../../../raw/postgres-12/contrib/README#L19-L25) |
| Diagnostic transaction must also exclude ordinary REINDEX | [index.c#reindex-locks-and-rebuild](../../../../raw/postgres-12/src/backend/catalog/index.c#L3450-L3530), [lock.c#ShareRowExclusiveLock](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L91), [btreefuncs.c#stats-release](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L181-L213), [btreefuncs.c#items-release](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L369-L373); measured old-lock allowance, corrected-lock timeout and success after release |
| Preserve unsigned block numbers through signed pageinspect arguments | [block.h#BlockNumber](../../../../raw/postgres-12/src/include/storage/block.h#L17-L35), [int8.c#int84](../../../../raw/postgres-12/src/backend/utils/adt/int8.c#L1160-L1169), [rawpage.c#get_raw_page](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L46-L63), [btreefuncs.c#stats-argument](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L162-L166), [btreefuncs.c#items-argument](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L310-L314); measured mapping and range-error dispatch at the signed boundary and maximum valid block |
| Guarded target selection and first-short-segment heap extent | [pg_index.h](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L49), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308), [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755), [genfile.c#pg_stat_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L362-L427); measured four target refusals, a valid empty target and 12,544 middle-segment boundary findings in each checksum leg |
| B-tree only; no table checker | [Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L3-L4), [target test](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L314-L338) |
| Overloads, `STRICT`, `PUBLIC` revoked, superuser to install | [SQL scripts](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L6-L24), [extension.c:616](../../../../raw/postgres-12/src/backend/commands/extension.c#L616) |
| Lock modes and early release | [verify_nbtree.c:232-303](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L232-L303) |
| Walk follows links from the true root; copied links can still reach concurrently deleted pages | [verify_nbtree.c:500-551](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L500-L551), [level walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L649-L835), [copied links and deletion](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1327-L1359) |
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
| No amcheck report for dangling index entries; why | [header](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L11-L14), [Bloom API](../../../../raw/postgres-12/src/include/lib/bloomfilter.h#L18-L25), [README.HOT](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L64-L120), [vacuumlazy.c:764-785](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L764-L785), [vacuumlazy.c:1243-1265](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1243-L1265); measured 548 unreported entries in both legs |
| Index scans skip TIDs with no line pointer and fail on blocks past the end | [heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1555-L1590), [index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L624-L632), [mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630); measured in both legs |
| `pageinspect` dangling-entry check | [btreefuncs.c](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L108-L122), [btreefuncs.c:269-280](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L269-L280), [heapfuncs.c:166-206](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L166-L206), [rawpage.c:148-152](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L148-L152), [lock.c:83-86](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86); measured 0 rows on the control and 548 on the fault in both legs |
| `zero_damaged_pages`, `ignore_checksum_failure`, hot standby behavior | [guc.c:1135-1161](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161), [bufmgr.c:908-924](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L908-L924), [lock.c:758-766](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L758-L766) |

## Open Questions

- **Uninspectable-item coverage is bounded.** The new fixture tests zero length with `LP_NORMAL`, not each misalignment or out-of-page storage variant that also makes pageinspect return NULL header fields. It does not validate redirect targets or every decoded tuple-header field. The checksums-on fixture deliberately recomputes a checksum over the malformed image; it is not a test of random damage bypassing checksums. The byte-level fault builder requires its checked little-endian layout, while the diagnostic itself uses pageinspect's decoded fields. [heapfuncs.c#storage-test](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L202-L227) [heapfuncs.c#NULL-fields](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L270-L279)
- **Upper-half block reads remain unmeasured.** The added boundary checks exercise argument conversion and pageinspect's range rejection on small relations, not a successful page read from a heap or index beyond 16 TiB. Full-range behavior is supported by the unsigned argument and buffer interfaces, not by a multi-terabyte fixture. The checksum-relocation collision qualification and the glossary's hint-bit condition were corrected from source; neither was newly fault-injected.
- **Docs versus source on who may run the checks.** `amcheck.sgml` says the functions "may only be used by superusers". The source revokes `EXECUTE` from `PUBLIC` and skips relation permission checks on purpose, and the regression test runs both checks as an ordinary role after `GRANT EXECUTE`. Source wins: a superuser can grant the right. [amcheck.sgml#superuser](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L37-L39) [verify_nbtree.c#no-permission-check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L306-L313) [check_btree.sql#permissions](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L29-L39)
- **Docs wording on which heap tuples are checked.** `amcheck.sgml` says `bt_index_check` verifies "the presence of all heap tuples as index tuples". The source checks only rows visible to its MVCC snapshot under `bt_index_check`, only rows satisfying a partial index's predicate, and under the parent check excludes HOT-updated recently dead rows. Source wins; the docs sentence is broader than the code. [amcheck.sgml#bt_index_check](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L100-L104) [heapam_handler.c#vacuum-visibility](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1402-L1428) [heapam_handler.c#predicate-values-hot](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1611-L1615)
- **Exact miss probability below 2 bytes per row.** The docs and the source comment bound the rate at about 2% for about 2 bytes per row, but the pinned tree does not compute it for less memory, as in calculation B. The rate there is left unstated.
- **Docs versus source on checksum counters.** The monitoring description does not name the `track_counts` gate or this pin's single-mismatch base-backup exception. The source wins, and the measurement reproduces both exclusions. [monitoring.sgml#checksum_failures](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2600-L2612) [pgstat.c#report-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1532-L1556) [basebackup.c#per-file-report](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1611-L1622)
- **Limits of this measurement.** The script does not inject downlink corruption, estimate Bloom miss rates, exercise malformed `pd_upper == 0` pages, measure hint-update FPIs, test `zero_damaged_pages`, test standby short reads, measure temporary-relation backup exclusion, or restore the malformed backup. The dangling-entry fixtures inject an all-zero page, a shortened final file and an emptied middle segment on a 1 MB-segment build; the default 1 GB-segment fault case was not run. They do not inject an `LP_UNUSED` line pointer or an index entry that names a heap-only tuple. The separate concurrency test covers ordinary REINDEX under `SHARE` and `SHARE ROW EXCLUSIVE`, but does not interleave an actual page-by-page diagnostic with rebuilding, or test concurrent REINDEX, a writer or VACUUM against the corrected lock. Invalid/not-ready index rejection, materialized-view/TOAST exclusions and preflight handling of a missing index main fork remain source deductions rather than new fixtures. The high-key length-check and missing-downlink leaf-level exclusions are also source readings, not injected corruptions. The shutdown tests inject signals at the pending-ownership boundary after launch; they do not reproduce a real startup timeout or every signal timing. No source/measurement disagreement remains: truncating the TOAST heap fails at the earlier short-read check, the backup client translates the server's checksum error as documented by their separate source paths, and the former summed-size heap bound was reproduced failing before replacement.
- **Documentation wording versus implementation.** `pg_checksums.sgml` says every file in the cluster is scanned or rewritten. The implementation has a narrower scope: the `skip[]` list, `pgsql_tmp` temporary-entry exclusions and optional `-f` filenode filtering limit files, and `PageIsNew()` pages bypass comparison. This page's tool table and the corrected [database-health checklist](../server-administration/database-health-checklist.md#observability-prerequisites) follow that bounded scope; the same-checkout documentation discrepancy remains. [pg_checksums.sgml#operation](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L48-L52) [pg_checksums.c#skip-list](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L95-L110) [pg_checksums.c#temporary-exclusions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L301-L311) [pg_checksums.c#filenode-filter](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L353-L355) [pg_checksums.c#new-page-skip](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L210-L213)
- **No common-concept pages.** PostgreSQL 12 has no common-concept page for B-tree integrity checking or for data checksums and page verification. Both would be useful separate pages; neither was created as part of this answer.

## Source References

- Uninspectable-item fixture and qualification: [heapfuncs.c#NULL-fields](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L202-L279), [execExprInterp.c#NULL-qualification](../../../../raw/postgres-12/src/backend/executor/execExprInterp.c#L820-L831), [varlena.c#byteaSetByte](../../../../raw/postgres-12/src/backend/utils/adt/varlena.c#L3464-L3493), [encode.c#hex_encode](../../../../raw/postgres-12/src/backend/utils/adt/encode.c#L126-L137), [rawpage.c#page_checksum](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L299-L324).
- Permission and counter boundaries: [namespace.c#database-boundary](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L241-L251), [contrib/README#installation](../../../../raw/postgres-12/contrib/README#L19-L25), [pgstatfuncs.c#checksum-getter](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1529-L1544).
- Locking and vocabulary review: [index.c#reindex-locks-and-rebuild](../../../../raw/postgres-12/src/backend/catalog/index.c#L3450-L3530), [indexcmds.c#concurrent-reindex-heap-lock](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3038-L3045), [signalfuncs.c#pg_cancel_backend](../../../../raw/postgres-12/src/backend/storage/ipc/signalfuncs.c#L104-L126), [nbtree/README#pivot-tuples](../../../../raw/postgres-12/src/backend/access/nbtree/README#L17-L40).
- Block-number transport: [block.h#BlockNumber](../../../../raw/postgres-12/src/include/storage/block.h#L17-L35), [int8.c#int84](../../../../raw/postgres-12/src/backend/utils/adt/int8.c#L1160-L1169); pageinspect argument readers are cited beside the query and results.
- Guarded-query revision: [lockcmds.c](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L35-L108), [dbsize.c](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308), [genfile.c](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L362-L427), [do.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/do.sgml#L34-L44), [plpgsql.sgml](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L945-L973), [set.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/set.sgml#L52-L59).
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
- Follow-up: [bloomfilter.h](../../../../raw/postgres-12/src/include/lib/bloomfilter.h#L18-L25), [heapam.c#heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1525-L1590), [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635), [pruneheap.c#heap_prune_chain](../../../../raw/postgres-12/src/backend/access/heap/pruneheap.c#L326-L347), [README.HOT](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L60-L122), [vacuumlazy.c](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L285-L286), [vacuumlazy.c:764-785](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L764-L785), [vacuumlazy.c:1243-1265](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1243-L1265), [vacuumlazy.c:1515-1524](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1515-L1524), [vacuum.c#vacuum_rel](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1676-L1681), [vacuum.sgml#INDEX_CLEANUP](../../../../raw/postgres-12/doc/src/sgml/ref/vacuum.sgml#L187-L204), [reloptions.c](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L143-L176), [rel.h](../../../../raw/postgres-12/src/include/utils/rel.h#L278-L279), [lock.c:78-86](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L86), [lock.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/lock.sgml#L61-L67), [htup_details.h:281](../../../../raw/postgres-12/src/include/access/htup_details.h#L281), [itemid.h:38-41](../../../../raw/postgres-12/src/include/storage/itemid.h#L38-L41), [bufpage.h:357-360](../../../../raw/postgres-12/src/include/storage/bufpage.h#L357-L360), [nbtree.h:181-219](../../../../raw/postgres-12/src/include/access/nbtree.h#L181-L219), [btreefuncs.c](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L48-L350), [heapfuncs.c](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L135-L206), [rawpage.c](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L152), [pageinspect.control](../../../../raw/postgres-12/contrib/pageinspect/pageinspect.control#L1-L5), [pageinspect--1.5.sql](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L38-L189), [gram.y#opt_materialized](../../../../raw/postgres-12/src/backend/parser/gram.y#L11425-L11428), [select.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L320-L325), [func.sgml#btrim](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L1726-L1737), [func.sgml#split_part](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L2336-L2346).
- [pg_basebackup.c#options](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L330-L353), [pg_basebackup.c#checksum-error](../../../../raw/postgres-12/src/bin/pg_basebackup/pg_basebackup.c#L2025-L2036), [installation.sgml#build-tools](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L80-L151), [installation.sgml#git-build](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L300-L318).
- [gram.y#persistence-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283), [heapam_handler.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624), [basebackup.c#relation-exclusions](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1162), [xlog.c#InRecovery](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L189-L200).
- [pg_ctl.c#detached-launch](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L449-L486), [pg_ctl.c#startup-timeout](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L887-L924).

## Navigation

- [Can amcheck Detect Zeroed Relation Segments in PostgreSQL 12? (unverified)](amcheck-zeroed-relation-segments.md)
- [Corruption indicators with checksums disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md)
- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
