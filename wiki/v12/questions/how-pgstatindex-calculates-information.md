---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: grok-4-3 2026-05-27T12:15:35Z
---

# How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)

## Question

In PostgreSQL 12, provide a comprehensive explanation of how `pgstatindex` calculates its information.

## Answer Up Front

`pgstatindex` is a physical B-tree page scan implemented by the `pgstattuple`
contrib extension. In PostgreSQL 12, the extension SQL exposes
`pgstatindex(text)` and `pgstatindex(regclass)` with ten output columns, and
the v1.5 SQL definitions route those entry points to `pgstatindex_v1_5` and
`pgstatindexbyid_v1_5`
([pgstattuple--1.4--1.5.sql#pgstatindex-text](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L37),
[pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).
Both v1.5 C wrappers open the target relation with `AccessShareLock` and call
the shared `pgstatindex_impl` worker
([pgstatindex.c#v1_5-entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212)).

`pgstatindex_impl` rejects anything that is not a physical B-tree index, reads
block 0 as the B-tree metapage, scans every block from `1` through
`nblocks - 1`, classifies each scanned page from `BTPageOpaqueData` flags, and
builds the result from counters accumulated in `BTIndexStat`
([pgstatindex.c#BTIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L75-L95),
[pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)).
It does not sample, read catalog estimates, or use the free space map for the
B-tree statistics; the count and ratio columns come from pages it reads in the
index main fork
([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315),
[bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199),
[relpath.h#MAIN_FORKNUM](../../../raw/postgres-12/src/include/common/relpath.h#L35-L53)).

The formulas are:

```text
index_size = (1 + internal_pages + leaf_pages + empty_pages + deleted_pages) * BLCKSZ

avg_leaf_density = 100 - (sum(PageGetFreeSpace(leaf_page)) /
                          sum(pd_special - SizeOfPageHeaderData for leaf_page)) * 100

leaf_fragmentation = fragments / leaf_pages * 100

fragments = count(live leaf pages where btpo_next != P_NONE and btpo_next < current_block_number)
```

The code returns `NaN` for `avg_leaf_density` when there is no leaf-page
capacity denominator, and `NaN` for `leaf_fragmentation` when there are no leaf
pages
([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356)).

One v12 quirk is worth flagging up front: the output column `empty_pages` does
not count pages with zero items. It counts half-dead pages only, because
`pgstatindex_impl` tests `P_ISDELETED` first and then `P_IGNORE`, so the
`empty_pages` branch is reachable only by pages whose `P_IGNORE`-true state is
not also `P_ISDELETED` — i.e. half-dead pages. The source labels the branch
with the literal comment `/* this is the "half dead" state */`
([pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310),
[nbtree.h#page-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L187-L196)).

## SQL Surface and Permissions

The PostgreSQL 12 `pgstattuple` control file sets `default_version = '1.5'`,
and the extension's Makefile builds `pgstattuple.o`, `pgstatindex.o`, and
`pgstatapprox.o` into the `pgstattuple` module
([pgstattuple.control#default-version](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5),
[Makefile#module](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13)).
The v1.5 SQL script declares both `pgstatindex(text)` and
`pgstatindex(regclass)` as C functions with `STRICT` and `PARALLEL SAFE`, then
revokes `PUBLIC` execute privilege and grants execute to
`pg_stat_scan_tables`
([pgstattuple--1.4--1.5.sql#pgstatindex-text](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L37),
[pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).
The docs describe the same default access rule: detailed page-level functions
are restricted by default to `pg_stat_scan_tables`, with superusers able to
bypass that privilege check
([pgstattuple.sgml#access](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L24)).

The older non-v1.5 C entry points still check `superuser()` before opening the
relation. The source keeps those checks because an upgraded library might be
used with a pre-1.5 extension installation
([pgstatindex.c#legacy-text-entry](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L160),
[pgstatindex.c#legacy-regclass-entry](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L182-L201)).
The v1.5 wrappers omit the superuser check because the extension script manages
execution privilege in SQL
([pgstatindex.c#v1_5-entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212),
[pgstattuple--1.4--1.5.sql#grants](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L36-L37)).

The published output columns are `version`, `tree_level`, `index_size`,
`root_block_no`, `internal_pages`, `leaf_pages`, `empty_pages`,
`deleted_pages`, `avg_leaf_density`, and `leaf_fragmentation`
([pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pgstattuple.sgml#pgstatindex-columns](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L189-L265)).

## Object Checks

`pgstatindex_impl` accepts only relations whose `relkind` is `RELKIND_INDEX`
and whose access method is `BTREE_AM_OID`
([pgstatindex.c#am-macros](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L70-L73),
[pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238)).
`RELKIND_INDEX` is the physical secondary-index relkind, and `BTREE_AM_OID`
is the catalog OID symbol for the `btree` index access method
([pg_class.h#relkind-index](../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L163),
[pg_am.dat#btree](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L18-L20)).
If either test fails, the function raises `wrong_object_type` with
`relation "<name>" is not a btree index`
([pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238),
[pgstattuple.out#wrong-am-errors](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L140-L152)).

The function also rejects another session's temporary relation. The source says
that the backend would likely read wrong data because it cannot see the owning
session's local buffers
([pgstatindex.c#other-temp-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L230-L238),
[rel.h#RELATION_IS_OTHER_TEMP](../../../raw/postgres-12/src/include/utils/rel.h#L541-L549)).

## Data Structures Used

`BTIndexStat` is the whole-index accumulator used only by `pgstatindex`. It
stores metapage fields (`version`, `level`, `root_blkno`), page-class counts,
leaf-space sums, and a `fragments` count
([pgstatindex.c#BTIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L75-L95)).

The metapage data comes from `BTMetaPageData`. The fields used by
`pgstatindex` are `btm_version`, `btm_level`, and `btm_root`; the B-tree
metapage is block 0
([nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L90-L113),
[nbtree.h#BTREE_METAPAGE](../../../raw/postgres-12/src/include/access/nbtree.h#L131-L135)).
PostgreSQL 12's current B-tree format version constant is `BTREE_VERSION = 4`,
while versions 2 and 3 remain readable for older upgraded indexes
([nbtree.h#btree-version](../../../raw/postgres-12/src/include/access/nbtree.h#L115-L135)).

Every non-metapage B-tree page has `BTPageOpaqueData` in special space. The
fields relevant to `pgstatindex` are the right-sibling link `btpo_next`, the
tree-level/transaction union, and the `btpo_flags` bits
([nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)).
The flag macros used by `pgstatindex` are `P_ISDELETED`, `P_IGNORE`, and
`P_ISLEAF`; `P_IGNORE` is true for deleted or half-dead pages, but the function
tests `P_ISDELETED` first, so the later `P_IGNORE` branch counts half-dead
pages as `empty_pages`
([nbtree.h#page-flags](../../../raw/postgres-12/src/include/access/nbtree.h#L70-L79),
[nbtree.h#page-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L181-L196),
[pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)).

## Metapage Read

The worker reads block 0 with `ReadBufferExtended(rel, MAIN_FORKNUM, 0,
RBM_NORMAL, bstrategy)`, obtains the page with `BufferGetPage`, and reads the
metapage through `BTPageGetMeta`
([pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
[nbtree.h#BTPageGetMeta](../../../raw/postgres-12/src/include/access/nbtree.h#L97-L113)).
It copies:

| Output field | Source field |
|---|---|
| `version` | `BTMetaPageData.btm_version` |
| `tree_level` | `BTMetaPageData.btm_level`, the root page's level (`0` means the root is also a leaf, since B-tree levels count upward from zero at the leaves) |
| `root_block_no` | `BTMetaPageData.btm_root` |

The docs describe `root_block_no` as the root page location, with zero meaning
there is no root page
([pgstattuple.sgml#pgstatindex-columns](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L203-L225)).
The regression output for an empty primary-key B-tree shows exactly that edge
case: version `4`, tree level `0`, one block of index size, root block `0`,
zero counted pages, and `NaN` for both ratio fields
([pgstattuple.out#empty-btree-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

## Full Page Scan

After the metapage read, the worker zeros all counters
([pgstatindex.c#counter-init](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L255-L264))
and gets the relation's current block count with `RelationGetNumberOfBlocks(rel)`
([pgstatindex.c#nblocks](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L269)).
That macro asks for the main fork, and `MAIN_FORKNUM` is fork number 0
([bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199),
[relpath.h#MAIN_FORKNUM](../../../raw/postgres-12/src/include/common/relpath.h#L35-L53)).
For index relkinds, `RelationGetNumberOfBlocksInFork` opens storage-manager
state if needed and calls `smgrnblocks`
([bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2790-L2810)).

The scan loop visits every block number from `1` to `nblocks - 1`, so it skips
only the metapage. Before each page read it calls `CHECK_FOR_INTERRUPTS`, then
reads the main-fork block with `ReadBufferExtended` and locks the buffer in
`BUFFER_LOCK_SHARE` mode
([pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315),
[bufmgr.h#buffer-lock-modes](../../../raw/postgres-12/src/include/storage/bufmgr.h#L83-L89),
[bufmgr.c#LockBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3585-L3607)).
The access strategy is `BAS_BULKREAD`, which the buffer manager describes as a
large read-only scan strategy
([pgstatindex.c#bulkread](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L218-L223),
[bufmgr.h#BufferAccessStrategyType](../../../raw/postgres-12/src/include/storage/bufmgr.h#L26-L34)).

The scan is page-by-page, not an instantaneous whole-index snapshot. The docs
warn that, like `pgstattuple`, `pgstatindex` accumulates results page-by-page
and concurrent changes can affect the result
([pgstattuple.sgml#pgstatindex-not-snapshot](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L278)).

## Page Classification

For each scanned page, the function reads the page's special-space pointer as
`BTPageOpaque` and then chooses exactly one page class
([pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L283-L310),
[bufpage.h#special-space](../../../raw/postgres-12/src/include/storage/bufpage.h#L69-L76),
[nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)).

| Branch | Counter changed | Meaning in source |
|---|---:|---|
| `P_ISDELETED(opaque)` | `deleted_pages++` | The B-tree flag `BTP_DELETED` is set. |
| `P_IGNORE(opaque)` after the deleted check | `empty_pages++` | The remaining ignored state is half-dead; the source comment says "this is the \"half dead\" state". |
| `P_ISLEAF(opaque)` | `leaf_pages++` plus leaf-space and fragmentation accumulation | The B-tree flag `BTP_LEAF` is set. |
| none of the above | `internal_pages++` | Any other non-metapage, non-deleted, non-half-dead, non-leaf B-tree page is counted as internal. |

The corresponding B-tree flag definitions are `BTP_LEAF`, `BTP_DELETED`, and
`BTP_HALF_DEAD`, and `P_IGNORE` is defined as deleted or half-dead
([nbtree.h#page-flags](../../../raw/postgres-12/src/include/access/nbtree.h#L70-L79),
[nbtree.h#page-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L187-L196)).

## Leaf Density

For every live leaf page, `pgstatindex` adds one page-capacity value to
`indexStat.max_avail` and one free-space value to `indexStat.free_space`
([pgstatindex.c#leaf-accumulation](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307)).
The capacity expression in the source is:

```c
max_avail = BLCKSZ - (BLCKSZ - ((PageHeader) page)->pd_special + SizeOfPageHeaderData);
```

That simplifies to `pd_special - SizeOfPageHeaderData`. `pd_special` is the
offset to special space, and `SizeOfPageHeaderData` is the page-header size
excluding line pointers
([pgstatindex.c#leaf-capacity](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L299),
[bufpage.h#PageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164),
[bufpage.h#SizeOfPageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L213-L217)).

The free-space expression is `PageGetFreeSpace(page)`. In PostgreSQL 12, that
routine returns `pd_upper - pd_lower` reduced by one `ItemIdData` line-pointer
slot when enough space exists, and returns zero if the gap is smaller than one
line pointer
([pgstatindex.c#leaf-free-space](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L296-L299),
[bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597),
[bufpage.h#PageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164)).
That is not the same as `PageGetExactFreeSpace`, which returns the raw
`pd_upper - pd_lower` gap without reserving line-pointer space
([bufpage.c#PageGetExactFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L627-L647)).

At result time, `avg_leaf_density` is:

```text
100.0 - free_space / max_avail * 100.0
```

where both numerator and denominator are sums over all counted leaf pages. If
`max_avail` is zero, the output string is `NaN`
([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L356)).

## Leaf Fragmentation

For every live leaf page, `pgstatindex` checks the B-tree right-link
`opaque->btpo_next`. If the link is not `P_NONE` and points to a lower physical
block number than the current page, it increments `indexStat.fragments`
([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307),
[nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68),
[nbtree.h#P_NONE](../../../raw/postgres-12/src/include/access/nbtree.h#L173-L181)).
The result column `leaf_fragmentation` is `fragments / leaf_pages * 100.0`; if
there are no leaf pages, the output string is `NaN`
([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)).

This is a physical-order metric. It counts the percentage of live leaf pages
whose logical right sibling is stored earlier in the file, not the length of
fragmented runs or the number of bytes out of order
([pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307),
[nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68)).

## Result Tuple

After scanning, `pgstatindex_impl` closes the relation and builds a composite
result using the caller's declared row type
([pgstatindex.c#close-and-result](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L317-L365)).
The ten values are emitted in the same order as the SQL OUT columns
([pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L356),
[pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).

| Output | Calculation |
|---|---|
| `version` | Copied from `btm_version` on the metapage. |
| `tree_level` | Copied from `btm_level` on the metapage; this is the root page's level, with `0` meaning the root is also a leaf. |
| `index_size` | `(1 + leaf_pages + internal_pages + deleted_pages + empty_pages) * BLCKSZ`; the `1` is the metapage. |
| `root_block_no` | Copied from `btm_root` on the metapage. |
| `internal_pages` | Count of non-metapage pages that are not deleted, not ignored/half-dead, and not leaf pages. |
| `leaf_pages` | Count of non-metapage pages where `P_ISLEAF` is true. |
| `empty_pages` | Count of non-metapage pages in the post-deleted `P_IGNORE` branch, which is the half-dead state in this code path. |
| `deleted_pages` | Count of non-metapage pages where `P_ISDELETED` is true. |
| `avg_leaf_density` | `100 - free_space / max_avail * 100`, or `NaN` when `max_avail == 0`. |
| `leaf_fragmentation` | `fragments / leaf_pages * 100`, or `NaN` when `leaf_pages == 0`. |

The docs note that reported `index_size` normally corresponds to one more page
than the sum of `internal_pages + leaf_pages + empty_pages + deleted_pages`,
because the metapage is included in `index_size`
([pgstattuple.sgml#metapage-size-note](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L273)).

## Regression Coverage

The v12 `pgstattuple` regression test creates an empty table with a primary-key
B-tree index and calls `pgstatindex` through text-like and regclass paths. The
expected output checks the empty-index case: one page after dividing
`index_size` by `current_setting('block_size')`, root block `0`, zero counted
pages, and `NaN` for both ratio columns
([pgstattuple.sql#empty-btree-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L37),
[pgstattuple.out#empty-btree-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

The same regression file checks wrong-access-method errors by calling
`pgstatindex` on GIN and hash indexes, and it checks unsupported relation
objects by calling `pgstatindex` on a partitioned table, view, foreign table,
and ordinary table partition
([pgstattuple.sql#wrong-am-and-relkind-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L47-L113),
[pgstattuple.out#wrong-am-and-relkind-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L126-L236)).
It also verifies that a real physical B-tree index on a table partition can be
inspected and returns the same empty-index shape
([pgstattuple.sql#partition-index-success](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L108-L113),
[pgstattuple.out#partition-index-success](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L228-L236)).

The checked-in `pgstattuple` regression target is only `pgstattuple`, and that
test file does not create a populated B-tree case that asserts non-`NaN`
`avg_leaf_density`, nonzero `leaf_fragmentation`, internal pages, deleted pages,
or half-dead pages
([Makefile#regress](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13),
[pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113)).

## Context Reviewed

- [pgstatindex.c#entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L66)
- [pgstatindex.c#BTIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L75-L95)
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)
- [pgstattuple--1.4--1.5.sql#pgstatindex](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L92)
- [pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [pgstattuple.sgml#pgstatindex](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L294)
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113)
- [pgstattuple.out#pgstatindex-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L236)
- [nbtree.h#page-and-meta-structures](../../../raw/postgres-12/src/include/access/nbtree.h#L29-L196)
- [bufpage.h#page-layout](../../../raw/postgres-12/src/include/storage/bufpage.h#L22-L76)
- [bufpage.h#PageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L217)
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)
- [bufpage.c#PageGetExactFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L627-L647)
- [bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199)
- [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2790-L2810)
- [bufmgr.c#LockBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3585-L3607)
- [pg_class.h#relkind-index](../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L163)
- [pg_am.dat#btree](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L18-L20)

## Evidence Map

| Claim | Evidence |
|---|---|
| v12 exposes `pgstatindex(text)` and `pgstatindex(regclass)` with the ten documented output columns | [pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92), [pgstattuple.sgml#pgstatindex-columns](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L189-L265) |
| The shared worker is `pgstatindex_impl`, reached by both v1.5 wrappers | [pgstatindex.c#v1_5-entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212), [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365) |
| The function accepts only physical B-tree indexes and rejects other-session temporary relations | [pgstatindex.c#object-check](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238), [pg_class.h#relkind-index](../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L163), [pg_am.dat#btree](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L18-L20) |
| Metapage fields supply `version`, `tree_level`, and `root_block_no` | [pgstatindex.c#metapage-read](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253), [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L97-L113) |
| The scan visits every non-metapage block in the main fork | [pgstatindex.c#scan-loop](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315), [bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199), [relpath.h#MAIN_FORKNUM](../../../raw/postgres-12/src/include/common/relpath.h#L35-L53) |
| Page classes come from B-tree opaque flags | [pgstatindex.c#classification](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtree.h#page-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L187-L196) |
| v12 `avg_leaf_density` uses `PageGetFreeSpace`, not `PageGetExactFreeSpace` | [pgstatindex.c#leaf-free-space](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L296-L299), [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597), [bufpage.c#PageGetExactFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L627-L647) |
| `leaf_fragmentation` is the percentage of live leaf pages whose right link points backward in physical block order | [pgstatindex.c#fragmentation-count](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [pgstatindex.c#result-formulas](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356), [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L55-L68) |
| Empty B-tree edge cases and wrong-AM errors are covered by regression tests | [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113), [pgstattuple.out#pgstatindex-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L236) |

## Source References

- [pgstatindex.c#entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L66) - C function registrations for the `pgstattuple` index helpers.
- [pgstatindex.c#BTIndexStat](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L75-L95) - accumulator fields used by `pgstatindex`.
- [pgstatindex.c#legacy-entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L201) - pre-v1.5 wrappers and retained superuser checks.
- [pgstatindex.c#v1_5-entry-points](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L212) - v1.5 wrappers that rely on SQL privileges.
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365) - object checks, metapage read, full scan, page classification, formulas, and tuple build.
- [pgstattuple--1.4--1.5.sql](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L136) - extension SQL definitions, output columns, `PARALLEL SAFE`, and grants.
- [pgstattuple.control](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5) - extension default version.
- [pgstattuple.sgml#pgstatindex](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L294) - documented `pgstatindex` output columns, metapage-size note, and non-snapshot caveat.
- [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113) - regression inputs for `pgstatindex`.
- [pgstattuple.out#pgstatindex-output](../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L236) - expected empty-index output and error outputs.
- [nbtree.h#BTPageOpaqueData](../../../raw/postgres-12/src/include/access/nbtree.h#L29-L79) - B-tree page opaque fields and flag bits.
- [nbtree.h#BTMetaPageData](../../../raw/postgres-12/src/include/access/nbtree.h#L90-L135) - B-tree metapage fields, metapage block number, and version constants.
- [nbtree.h#page-macros](../../../raw/postgres-12/src/include/access/nbtree.h#L181-L196) - `P_NONE`, `P_ISLEAF`, `P_ISDELETED`, and `P_IGNORE`.
- [bufpage.h#page-layout](../../../raw/postgres-12/src/include/storage/bufpage.h#L22-L76) - general page layout and special-space location.
- [bufpage.h#PageHeaderData](../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L217) - `pd_lower`, `pd_upper`, `pd_special`, and `SizeOfPageHeaderData`.
- [bufpage.c#PageGetFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597) - free-space routine used by v12 `pgstatindex`.
- [bufpage.c#PageGetExactFreeSpace](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L627-L647) - contrast with exact page gap, not used by v12 `pgstatindex`.
- [bufmgr.h#buffer-access](../../../raw/postgres-12/src/include/storage/bufmgr.h#L26-L47) - `BAS_BULKREAD` and `RBM_NORMAL`.
- [bufmgr.h#buffer-locks-and-blocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L83-L89) and [bufmgr.h#RelationGetNumberOfBlocks](../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199) - buffer lock modes and main-fork block-count macro.
- [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2790-L2810) - block-count implementation for index relkinds.
- [bufmgr.c#LockBuffer](../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3585-L3607) - share/exclusive content-lock behavior.
- [relpath.h#MAIN_FORKNUM](../../../raw/postgres-12/src/include/common/relpath.h#L35-L53) - main fork number used by the scan.
- [rel.h#RELATION_IS_OTHER_TEMP](../../../raw/postgres-12/src/include/utils/rel.h#L541-L549) - other-session temporary relation test.
- [pg_class.h#relam-relkind](../../../raw/postgres-12/src/include/catalog/pg_class.h#L49-L81) - `relam` and `relkind` catalog fields.
- [pg_class.h#relkind-index](../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L163) - physical and partitioned index relkind constants.
- [pg_am.dat#btree](../../../raw/postgres-12/src/include/catalog/pg_am.dat#L18-L20) - `BTREE_AM_OID` catalog seed.

## Open Questions

- There are no unresolved source questions for the v12 calculation path above.
  The main verification gap is test coverage: the checked-in regression test
  exercises empty B-tree output and error paths, but not populated B-tree
  density, fragmentation, internal-page, deleted-page, or half-dead-page cases
  ([Makefile#regress](../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13),
  [pgstattuple.sql#pgstatindex-tests](../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L18-L113)).

## Related Pages

- [v12/index](../index.md)
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](pgstatindex-sample-variant-proposal.md)
- [versions](../../versions.md)
