---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Can amcheck Detect Zeroed Relation Segments in PostgreSQL 12? (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Main answer](#main-answer)
  - [What each check actually reads](#what-each-check-actually-reads)
  - [Detection map](#detection-map)
  - [Why the last heap segment can disappear silently](#why-the-last-heap-segment-can-disappear-silently)
  - [Overwriting bytes and damaging an index are different cases](#overwriting-bytes-and-damaging-an-index-are-different-cases)
  - [State changes and additional limits](#state-changes-and-additional-limits)
  - [Build boundary and tests](#build-boundary-and-tests)
  - [Causal summary](#causal-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Follow AGENTS.md. In PostgreSQL 12, for the question “A Manually Zeroed Relation File in PostgreSQL 12: Do SELECT, VACUUM, and pg_dump Error? (unverified)”, can amcheck on the table or on the index catch all the scenarios, especially if the last segment was zeroed out?

## Answer

### Main answer

**No. A previously nonempty last heap segment truncated to zero length can escape even `bt_index_parent_check(index, true, true)`.** PostgreSQL 12's [`amcheck`](../../../glossary.md#amcheck) accepts [B-tree](../../../glossary.md#b-tree) indexes, not tables, and has no `verify_heapam()` function. Its optional heap check looks for index entries missing from the index for tuples that the heap scan finds. It does not establish that every index entry still has a corresponding heap tuple. Consequently, an intact index can pass while rows have disappeared from the [heap](../../../glossary.md#heap). [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L315-L338) [Makefile#amcheck](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10) [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624) [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920)

This is a source-only follow-up to the [zeroed-file question](../storage-and-vacuum/zeroed-relation-file.md). There, “zeroed” means **shortened to zero bytes**. Below, a separate case covers overwriting existing bytes with zeros while preserving file length. No new measurement was run and no result from that page is presented as an `amcheck` test.

### What each check actually reads

The check's argument identifies the index. The checker opens and locks its associated heap itself. Passing a heap table name is rejected; that error says the argument is unsupported, not that corruption was detected. The following are function-call shapes, not executable SQL for a particular installation. [verify_nbtree.c#open-relations](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L224-L277) [check_btree.sql#table-rejection](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L41-L43)

| Call / option | Default and explicitly enabled behavior | What it cannot establish |
|---|---|---|
| `bt_index_check(index)` | Checks the B-tree with [`AccessShareLock`](../../../glossary.md#lock-mode) on heap and index. `heapallindexed` is false. | Does not perform the optional heap scan. [verify_nbtree.c#bt_index_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L192) |
| `bt_index_check(index, true)` | Passing true enables `heapallindexed`: fingerprint index entries, then scan the heap and test the tuples that should have entries. | Does not look up every index entry in the heap. [verify_nbtree.c#heapallindexed](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L413-L435) [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624) |
| `bt_index_parent_check(index, true)` | Choosing the parent checker takes [`ShareLock`](../../../glossary.md#lock-mode) and adds parent/child checks; true also enables the heap check. The one-argument form defaults that option to false. | Stronger index invariants do not reverse the heap-membership test. [verify_nbtree.c#bt_index_parent_check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L195-L236) [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920) |
| Third argument `true` on the parent checker | `rootdescend` defaults false. Explicit true searches from the index root for each checked ordinary leaf tuple; it requires B-tree on-disk version 4. | The search stays inside the index. The heap [TID](../../../glossary.md#tid) is part of the search key, not an instruction to fetch the heap row. [verify_nbtree.c#version-gate](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L486-L492) [verify_nbtree.c#rootdescend](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2059-L2108) |

The important state has three different origins:

| Value | Source and freshness | Later use |
|---|---|---|
| Heap scan limit, `rs_nblocks` | Computed when the scan starts from the storage manager's segment lengths and already-open segment list. An externally shortened earlier segment can invalidate assumptions in that list. | Limits which heap pages can supply tuples to the checker. [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) |
| Index membership filter | Built during this check from the index entries it visits. It is a [Bloom filter](../../../glossary.md#amcheck): a compact, probabilistic set-membership test, not an exact inventory. | Tests each heap-derived index tuple for absence. [verify_nbtree.c#fingerprint](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1073-L1080) [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L1-L15) |
| Page contents seen by the checker | `ReadBufferExtended` can return a valid page already in [shared buffers](../../../glossary.md#buffer-manager). That page can predate an external file change. | Both traversal and heap scanning may see buffered contents instead of the changed file. [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) |

### Detection map

```text
Valid B-tree index supplied
  -> Read metapage; follow root, child and sibling links [A]
       -> Reached missing/zero index page, no cached old copy? Error [B]
       -> Index checks succeed
            -> heapallindexed false? Finish without heap comparison [C]
            -> heapallindexed true
                 -> Start size-bounded heap scan [D]
                 -> For each qualifying tuple actually found:
                      form expected index tuple -> test membership [E]
                      -> Definitely absent? Error
                      -> Possibly present? Continue
                 -> No more heap tuples? Success

Truncated-away or all-zero heap pages supply no lost tuples to [E].
rootdescend adds index searches to [A], not heap fetches to [E]. [F]
```

- **[A]** [verify_nbtree.c#root-and-levels](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551), [verify_nbtree.c#sibling-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L650-L833).
- **[B]** [verify_nbtree.c#page-read](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362), [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630), [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719). Short-read errors here assume normal operation outside recovery with `zero_damaged_pages` at its default false. [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1149-L1161)
- **[C–E]** [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624), [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231), [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920).
- **[F]** [verify_nbtree.c#bt_rootdescend](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2059-L2108).

### Why the last heap segment can disappear silently

1. **External truncation changes the scan's input.** Ordinarily `mdnblocks()` counts the existing main-fork blocks. If the final segment is externally shortened to zero, the same calculation reports an earlier end. Unlike an earlier segment already assumed full, the last open segment is measured again. A scan started before the truncation can still retain its old `rs_nblocks`; a newly started scan that measures the shortened last segment has the smaller limit. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [heapam.c#scan-limit](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231)
2. **The heap check uses that bounded scan.** `bt_check_every_level()` calls `table_beginscan_strat()`, then `table_index_build_scan()`. The table access-method interface dispatches to the heap's index-build scan, which consumes `heap_getnext()`. It does not enumerate index TIDs and fetch their heap pages. [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L571-L614) [tableam.h#scan-dispatch](../../../../raw/postgres-12/src/include/access/tableam.h#L761-L773) [tableam.h#index-build-dispatch](../../../../raw/postgres-12/src/include/access/tableam.h#L1512-L1533) [heapam_handler.c#callback-registration](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2640-L2645) [heapam_handler.c#heap-loop](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1306-L1321)
3. **Only surviving tuples create membership tests.** The callback forms an index tuple from a heap tuple's indexed values and TID, then probes the filter. With an intact index and otherwise consistent surviving heap, those probes can all pass. Extra index entries for erased heap rows create no failed probe, because no surviving heap tuple asks about them. This is a deduction from the callback and scan, not a measured outcome. [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920)

The index traversal uses stored index links, while the heap comparison uses the current scan's block limit. Because those inputs are independent, the checker can see the old index entries and only the shortened heap, yet find no violated condition. Enabling both optional checks does not change that direction of comparison. [verify_nbtree.c#root-and-levels](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551) [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624) [verify_nbtree.c#bt_rootdescend](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2059-L2108)

### Overwriting bytes and damaging an index are different cases

These cases assume otherwise intact structures and describe detection limits, not a promise of a particular result on every damaged cluster.

| Changed file or state | What the check can do | Why |
|---|---|---|
| Last heap segment truncated to zero length | Can pass, including with both options true. | Lost rows fall outside the newly measured scan limit; the callback tests only found tuples. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920) |
| First or middle heap segment truncated | A fresh file-open state can likewise hide that segment and later ones. A stale larger scan limit can instead cause a short-read error when the scan needs an uncached damaged page. | The first short segment ends fresh size discovery; already-open later segments or an already-started scan change the bound. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) |
| Heap segment overwritten entirely with zeros, length preserved | Can also pass. | A normal populated page has line pointers. Overwriting the entire page removes them: `PageIsVerified()` accepts an all-zero page, `PageGetMaxOffsetNumber()` returns zero, and the scan produces no tuples from it. [bufpage.c#all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L142) [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L348-L360) [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L352-L449) |
| Last B-tree index segment truncated, containing a page reached through the intact tree | An uncached read of a missing block raises an error in normal operation. | Traversal follows block references; it does not silently stop at the shortened file's physical end. [verify_nbtree.c#sibling-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L650-L833) [verify_nbtree.c#page-read](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) |
| Reached B-tree page overwritten with zeros, length preserved | Raises `index ... contains unexpected zero page at block ...` when the checker sees the zero contents. | The buffer-level all-zero allowance is followed by `_bt_checkpage()`, which rejects a new/zero page in this traversal. [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719) |
| Index tail contains only pages that no checked tree link reaches | There is no guarantee of detecting its removal or zeroing. | The checker follows the metapage and tree links; it does not inspect every physical block simply because that block exists in the index file. Normal page deletion can leave fully deleted pages with no parent or sibling links. [verify_nbtree.c#tree-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551) [verify_nbtree.c#deleted-pages](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L685-L703) |
| Heap `_fsm` or `_vm` fork truncated | No comprehensive fork validation. | Neither optional check performs an integrity walk of these auxiliary forks; `heapallindexed` scans the heap main fork. Incidental accesses do not establish completeness. [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L571-L614) [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L352-L382) |

Here `_fsm` is the [free space map](../../../glossary.md#free-space-map) and `_vm` is the [visibility map](../../../glossary.md#visibility-map); they are separate [forks](../../../glossary.md#fork), not later segments of the heap main fork. [storage.sgml#forks](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L199-L243)

### State changes and additional limits

| Event | State before → after | Effect on a later check |
|---|---|---|
| External last-segment truncation before a heap scan starts | Original last-segment length → zero length | New size discovery supplies a smaller `rs_nblocks`; the erased suffix supplies no callback input. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231) |
| External truncation after the scan starts | Stored scan bound stays unchanged while disk becomes shorter | A later uncached read within the old bound can fail. [heapam.c#scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L713-L733) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) |
| Old page remains buffered | Disk changed; valid buffer still contains old bytes | Reading the buffer can hide the disk change. A later cache miss takes the storage read path. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [bufmgr.c#disk-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) |

A missing index entry for a surviving heap tuple is a different defect from an erased heap tuple. The former can fail the membership probe. The filter can also return “possibly present” for a genuinely absent entry, so a successful probe is not an exact proof. Repeating a probabilistic check cannot fix the erased-row blind spot: the absent heap tuple never generates a probe at all. [bloomfilter.c#membership](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L1-L15) [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920)

An ordinary index without a predicate considers qualifying tuples across the table. A [partial index](../../../glossary.md#partial-index) adds its predicate: the index-build scan skips tuples that do not satisfy it. Indexed attributes and expressions are then used to form the comparison tuple. Thus checking an index is not an audit of every non-indexed value or every external [TOAST](../../../glossary.md#toast) value. [heapam_handler.c#predicate-and-values](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1603-L1626)

The [data checksum](../../../glossary.md#data-checksums) setting does not remove the all-zero-page blind spot. Checksums are off by default at this pin; even with them enabled, the page verifier accepts an entirely zero page without requiring a matching checksum. A requested missing block instead takes the short-read path before page verification. [initdb.c#data_checksums](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)

### Build boundary and tests

The pinned PostgreSQL 12 checkout is `45b88269a353ad93744772791feb6d01bc7e1e42`. Its contrib Makefile builds `verify_nbtree.o`; the extension SQL scripts register the two function names and their overloads, with execution revoked from `PUBLIC`. This is an explicitly invoked extension check, not an automatic table scan on ordinary queries. [Makefile#amcheck](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10) [amcheck--1.0.sql#functions](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L6-L24) [amcheck--1.0--1.1.sql#heapallindexed](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0--1.1.sql#L9-L29) [amcheck--1.1--1.2.sql#rootdescend](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.1--1.2.sql#L9-L19)

Segment boundaries depend on `RELSEG_SIZE`, generated into `pg_config.h`. The default build uses 1 GB segments; `configure.in` derives the block count from the segment and block sizes. No special build option is needed for the blind spot: `mdnblocks()` branches on a segment being shorter than the configured limit. This page uses the source calculation and does not reuse the original page's modified measurement build. [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L293) [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L730-L755)

The module's regression SQL tests table/non-B-tree rejection, ordinary and enhanced checks, root descent, page deletion, and normalized TOAST representations. These test inputs do not reproduce manual heap/index segment truncation or a byte-for-byte all-zero overwrite. No test was executed for this follow-up. [check_btree.sql#tests](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L1-L114) [Makefile#REGRESS](../../../../raw/postgres-12/contrib/amcheck/Makefile#L10)

### Causal summary

**Heap suffix erased before a new scan → smaller scan limit, or zero pages with no tuples → no expected index tuple generated for the lost rows → intact index can pass all requested checks.** A damaged index segment instead errors when an actual tree reference makes the checker read a missing or zero page. Neither success certifies that no rows were lost. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L348-L360) [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920) [verify_nbtree.c#page-read](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2362) [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719)

## Context Reviewed

- Extension entry points, SQL overloads, target eligibility, locking, `BtreeCheckState`, index-level traversal, fingerprint insertion, heap callback and root searches.
- Adjacent core boundary: table AM scan dispatch, `HeapScanDesc.rs_nblocks`, index-build predicate/value handling, buffer hits, `mdnblocks`, `mdread`, page macros and B-tree page checks.
- Build declarations, segment-size generation, checksums and short-read defaults, module regression SQL and expected outputs, and same-checkout `amcheck.sgml` detection limits.
- Comparison set: the original zeroed-file page, the v12 corruption-log page, same-version search hits for `amcheck`, `heapallindexed`, `rootdescend`, `mdnblocks` and zero pages, glossary entries, both indexes, and the v12 pin row. No v12 common-concept page exists for these mechanisms.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| No table-target checker in this module | [Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10), [target check](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L315-L338) |
| Heap comparison is one-way | [heap scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624), [membership callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920) |
| Last-segment truncation can evade the comparison | [size discovery](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755), [scan limit](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231) |
| Zero heap pages and zero B-tree pages differ | [generic verifier](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L142), [tuple count macro](../../../../raw/postgres-12/src/include/storage/bufpage.h#L348-L360), [B-tree verifier](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719) |
| Root descent does not fetch heap tuples | [bt_rootdescend](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2059-L2108) |
| Index traversal is not every-block validation | [root traversal](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551), [sibling traversal](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L650-L833) |
| Cache and probabilistic limits | [buffer hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795), [Bloom semantics](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L1-L15) |

## Open Questions

- The scenarios above are source deductions. An isolated reproduction matrix for the two functions, both options, last/middle segments, byte overwrites, and warm/cold buffers has not been run. The existing zeroed-file page's measurements do not test `amcheck`.
- PostgreSQL 12 has no common-concept page for relation segments and page-read validation or for B-tree integrity checking. These would be useful separate concept pages; none was created as a side effect of this answer.

## Source References

- [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L338) — extension entry points; detailed ranges accompany claims above.
- [Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L21), [amcheck--1.0.sql](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0.sql#L1-L24), [amcheck--1.0--1.1.sql](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0--1.1.sql#L1-L29), [amcheck--1.1--1.2.sql](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.1--1.2.sql#L1-L19).
- [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L295), [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1154-L1670), [tableam.h](../../../../raw/postgres-12/src/include/access/tableam.h#L1500-L1533).
- [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L575-L755), [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L925), [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L142), [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L348-L360), [nbtpage.c](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719).
- [bloomfilter.c](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L1-L85), [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1149-L1161), [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144).
- [configure.in](../../../../raw/postgres-12/configure.in#L279-L293), [pg_config.h.in](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864), [storage.sgml](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L199-L243).
- [check_btree.sql](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L1-L114), [amcheck.sgml#detection-limits](../../../../raw/postgres-12/doc/src/sgml/amcheck.sgml#L314-L336).

## Navigation

- [Original zeroed-file question (unverified)](../storage-and-vacuum/zeroed-relation-file.md)
- [Corruption indicators with checksums disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md)
- [PostgreSQL 12 wiki](../../index.md) · [Wiki index](../../../index.md) · [Wiki Glossary (unverified)](../../../glossary.md)
