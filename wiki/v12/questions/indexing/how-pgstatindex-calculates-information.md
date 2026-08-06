---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-07-10T16:00:43Z
---

# How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [What `pgstatindex` does](#what-pgstatindex-does)
  - [SQL and C call path](#sql-and-c-call-path)
  - [Accepted relations, permissions, and index state](#accepted-relations-permissions-and-index-state)
  - [Data structures and generated-header boundary](#data-structures-and-generated-header-boundary)
  - [Metapage read](#metapage-read)
  - [Physical page scan](#physical-page-scan)
  - [Page classification](#page-classification)
  - [How `index_size` is calculated](#how-indexsize-is-calculated)
  - [How `avg_leaf_density` is calculated](#how-avgleafdensity-is-calculated)
  - [How `leaf_fragmentation` is calculated](#how-leaffragmentation-is-calculated)
  - [Result construction and edge paths](#result-construction-and-edge-paths)
  - [Concurrency and integrity limits](#concurrency-and-integrity-limits)
  - [Regression coverage](#regression-coverage)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, provide a comprehensive explanation of how `pgstatindex` calculates its information.

## Answer

### What `pgstatindex` does

`pgstatindex` is a physical B-tree main-fork scan in the `pgstattuple` contrib
extension. It does not sample pages, read `pg_class.relpages`, traverse the tree
from its root, or call the B-tree access-method API. The worker reads block 0,
captures the main-fork length once, then reads each block from `1` through that
captured length minus one and interprets PostgreSQL's private B-tree page
layout directly
([pgstatindex.c#includes-and-AM-tests](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L28-L73),
[pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365)).

For a completed call on a normal B-tree main fork, the core calculations are:

```text
index_size = nblocks_captured_once * BLCKSZ
           = (1 + internal_pages + leaf_pages
                + empty_pages + deleted_pages) * BLCKSZ

leaf_capacity(page) = pd_special - SizeOfPageHeaderData
leaf_free(page)     = PageGetFreeSpace(page)

avg_leaf_density = format_to_2_decimals(
    100 - sum(leaf_free(page)) / sum(leaf_capacity(page)) * 100
)

fragments = count(live leaf pages where
                  btpo_next != P_NONE and btpo_next < current_block_number)

leaf_fragmentation = format_to_2_decimals(
    fragments / leaf_pages * 100
)
```

The first `1` is the metapage. Only live leaf pages contribute to either
percentage. The worker emits `NaN` when the relevant denominator is zero, and
formats each non-`NaN` percentage with `%.2f` before converting the text to the
declared `float8` output
([pgstatindex.c#leaf-accumulation](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L288-L307),
[pgstatindex.c#result-formulas](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L359)).

The surprising field is `empty_pages`. It does not test whether a page has zero
items. It counts the post-`P_ISDELETED` `P_IGNORE` branch, which is the
half-dead B-tree state in this implementation
([pgstatindex.c#classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310),
[nbtree.h#page-state-macros](../../../../raw/postgres-12/src/include/access/nbtree.h#L183-L196)).

### SQL and C call path

The extension control file selects version 1.5. There is no standalone
`pgstattuple--1.5.sql` install script in the Makefile. A fresh default install
therefore starts from the installable 1.4 script and follows the 1.4-to-1.5
update path, which is the extension manager's defined fallback when the target
version has no direct install script
([pgstattuple.control#default-version](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5),
[Makefile#extension-scripts](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L3-L13),
[extension.c#install-path-selection](../../../../raw/postgres-12/src/backend/commands/extension.c#L1297-L1400),
[extension.c#install-then-update](../../../../raw/postgres-12/src/backend/commands/extension.c#L1536-L1550)).

The 1.4 base definitions point the two SQL overloads at the legacy C symbols.
The 1.5 update replaces them as follows
([pgstattuple--1.4.sql#legacy-pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31),
[pgstattuple--1.4.sql#legacy-pgstatindexbyid](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L62-L74),
[pgstattuple--1.4--1.5.sql#v1.5-pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L37),
[pgstattuple--1.4--1.5.sql#v1.5-pgstatindexbyid](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)):

```text
pgstatindex(text)
  -> pgstatindex_v1_5
  -> makeRangeVarFromNameList(textToQualifiedNameList(...))
  -> relation_openrv(..., AccessShareLock)
  -> pgstatindex_impl

pgstatindex(regclass)
  -> pgstatindexbyid_v1_5
  -> relation_open(oid, AccessShareLock)
  -> pgstatindex_impl
```

The text overload resolves a possibly schema-qualified name in the C wrapper.
The `regclass` overload receives an already-resolved OID. Both wrappers acquire
`AccessShareLock` on the index and transfer the open `Relation` to the same
worker
([pgstatindex.c#text-wrappers](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L180),
[pgstatindex.c#regclass-wrappers](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L182-L213),
[relation.c#relation_openrv](../../../../raw/postgres-12/src/backend/access/common/relation.c#L131-L161)).

Both 1.5 overloads are `STRICT` and `PARALLEL SAFE`. The update script revokes
`PUBLIC` execution and grants execution to `pg_stat_scan_tables`; superusers
bypass that SQL privilege check. `STRICT` means PostgreSQL returns null for a
null input without entering the C function
([pgstattuple--1.4--1.5.sql#v1.5-pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L37),
[pgstattuple--1.4--1.5.sql#v1.5-pgstatindexbyid](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pgstattuple.sgml#access](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L24),
[xfunc.sgml#STRICT](../../../../raw/postgres-12/doc/src/sgml/xfunc.sgml#L2397-L2404)).

The shared library retains the legacy C entry points. They still call
`superuser()` because a library can be upgraded while the installed extension
SQL remains older than 1.5 and still grants the old SQL functions to `PUBLIC`.
The 1.5 C wrappers omit that internal check because SQL function privileges now
control access
([pgstatindex.c#legacy-text-wrapper](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L160),
[pgstatindex.c#legacy-regclass-wrapper](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L182-L201),
[pgstatindex.c#v1.5-wrappers](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L162-L213)).

### Accepted relations, permissions, and index state

`pgstatindex_impl` accepts only `RELKIND_INDEX` relations whose `relam` equals
the built-in `BTREE_AM_OID`. It therefore rejects tables, views, foreign tables,
partitioned indexes, and every non-B-tree index. A physical B-tree index on a
table partition is accepted
([pgstatindex.c#object-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238),
[pg_class.h#relation-kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L163),
[pg_am.dat#BTREE_AM_OID](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L23),
[pgstattuple.sql#relation-kind-tests](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L55-L113)).

The worker rejects another session's temporary relation because the backend
cannot see that session's local buffers. It can inspect a temporary B-tree
owned by the current session
([pgstatindex.c#other-session-temp-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L230-L238),
[rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-12/src/include/utils/rel.h#L541-L549)).

Once SQL function execution is allowed, neither wrapper nor the worker checks
relation ownership or a relation-level `SELECT` privilege. `relation_open` gets
the requested relation lock and relcache entry; the caller is responsible for
checking whether it can handle that relation kind
([pgstatindex.c#wrappers-and-worker](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L238),
[relation.c#relation_open](../../../../raw/postgres-12/src/backend/access/common/relation.c#L35-L79)).

The worker also does not read `pg_index.indisvalid`, `indisready`, or
`indislive`. An invalid or not-ready physical B-tree can therefore be scanned
if its storage is readable; the output describes pages, not whether the planner
or executor may use the index
([pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365),
[pg_index.h#index-state-flags](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L32-L44)).

### Data structures and generated-header boundary

`BTIndexStat` is the local whole-index accumulator. It stores three metapage
values, four page-class counters, the summed leaf capacity and free space, and
the backward-link count
([pgstatindex.c#BTIndexStat](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L75-L95)).

The worker reads two core B-tree structures directly:

- `BTMetaPageData` supplies `btm_version`, `btm_root`, and `btm_level`
  ([nbtree.h#BTMetaPageData](../../../../raw/postgres-12/src/include/access/nbtree.h#L90-L110)).
- `BTPageOpaqueData`, stored in each B-tree page's special space, contains the
  sibling links, level-or-deletion-XID union, flags, and vacuum cycle ID. The
  worker uses its flags and `btpo_next` right-link
  ([nbtree.h#BTPageOpaqueData](../../../../raw/postgres-12/src/include/access/nbtree.h#L29-L78),
  [pgstatindex.c#opaque-fields-used](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L283-L307)).

This is a core-layout dependency, not an index-AM callback boundary. The source
includes private `access/nbtree.h` and compares `relam` with `BTREE_AM_OID`
directly
([pgstatindex.c#includes-and-AM-tests](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L28-L73)).

`BTREE_AM_OID` also exposes a build-time generated-header dependency.
`pgstatindex.c` includes `catalog/pg_am.h`; that header includes generated
`catalog/pg_am_d.h`. The catalog Makefile generates every `*_d.h` from catalog
headers and `.dat` seed files through `genbki.pl`, and `pg_am.dat` supplies the
`BTREE_AM_OID` symbol and value
([pg_am.h#generated-header-include](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L18-L29),
[pg_am.dat#BTREE_AM_OID](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L23),
[catalog/Makefile#generated-catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L100),
[genbki.pl#OID-symbol-emission](../../../../raw/postgres-12/src/backend/catalog/genbki.pl#L595-L603)).

The contrib Makefile links `pgstatindex.o` with `pgstattuple.o` and
`pgstatapprox.o` into one `pgstattuple` module. `PG_FUNCTION_INFO_V1` emits each
`pg_finfo_<symbol>` function that the function manager requires when loading an
SQL-callable C symbol
([Makefile#module](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13),
[pgstatindex.c#function-info](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L67),
[fmgr.h#PG_FUNCTION_INFO_V1](../../../../raw/postgres-12/src/include/fmgr.h#L383-L413),
[fmgr.c#fetch_finfo_record](../../../../raw/postgres-12/src/backend/utils/fmgr/fmgr.c#L458-L511)).

### Metapage read

Block 0 is `BTREE_METAPAGE`. The worker reads it with `ReadBufferExtended(...,
MAIN_FORKNUM, 0, RBM_NORMAL, bstrategy)`, obtains `BTMetaPageData` with
`BTPageGetMeta`, copies three fields, and releases the buffer
([pgstatindex.c#metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
[nbtree.h#metapage-layout](../../../../raw/postgres-12/src/include/access/nbtree.h#L90-L135)).

| Output | Metapage source | Exact meaning |
|---|---|---|
| `version` | `btm_version` | On-disk B-tree format version. Version 4 is current at this pin; versions 2 and 3 remain readable. |
| `tree_level` | `btm_level` | Level of the true root, counting upward from leaf level 0. |
| `root_block_no` | `btm_root` | True-root block number; `P_NONE`, represented by zero, means no root exists yet. |

Those meanings come from the structure and version definitions
([nbtree.h#metapage-layout](../../../../raw/postgres-12/src/include/access/nbtree.h#L90-L135),
[nbtree.h#page-levels](../../../../raw/postgres-12/src/include/access/nbtree.h#L29-L39)).
The worker does not report `btm_fastroot` or `btm_fastlevel`. Normal B-tree
searches may start from that effective root, while `pgstatindex` reports the
true-root fields
([nbtpage.c#fast-root-selection](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L426-L465),
[nbtpage.c#root-height](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L574-L625)).

A metapage-only empty index therefore reports format version 4, level 0, root
block 0, zero classified non-metapages, one block of `index_size`, and `NaN` for
both ratios in the checked-in regression result
([pgstattuple.out#empty-index](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

Two limits matter:

1. The worker pins the metapage but does not acquire a buffer content lock
   before reading it. A root split updates `btm_root` and `btm_level` while
   holding the metapage's B-tree write lock, so the three copied fields are not
   protected as one metapage observation
   ([pgstatindex.c#unlocked-metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
   [nbtinsert.c#new-root-metapage-update](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2060-L2110)).
2. The worker does not call `_bt_getmeta` and does not verify `BTP_META`,
   `BTREE_MAGIC`, or the supported version range. The normal B-tree metadata
   accessor performs all three checks
   ([pgstatindex.c#unvalidated-metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253),
   [nbtpage.c#_bt_getmeta](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148)).

### Physical page scan

After the metapage read, the worker zeros its counters and calls
`RelationGetNumberOfBlocks(rel)` once. That macro asks for the main fork;
`RelationGetNumberOfBlocksInFork` opens storage-manager state if necessary and
calls `smgrnblocks`
([pgstatindex.c#counter-init-and-length](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L255-L271),
[bufmgr.h#RelationGetNumberOfBlocks](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199),
[bufmgr.c#RelationGetNumberOfBlocksInFork](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2790-L2810)).

The loop visits every captured block number from 1 through `nblocks - 1`. For
each block it:

1. checks for interrupts;
2. reads the main-fork block with the `BAS_BULKREAD` strategy;
3. acquires `BUFFER_LOCK_SHARE` on the page contents;
4. reads its B-tree opaque flags and, for a live leaf, page-header free space;
5. releases the content lock and buffer pin.

The implementation is at
[pgstatindex.c#scan-loop](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315).
`BAS_BULKREAD` is the buffer manager's strategy for a large read-only scan, and
`BUFFER_LOCK_SHARE` maps to a shared buffer content lock
([bufmgr.h#BufferAccessStrategyType](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L26-L47),
[bufmgr.h#buffer-lock-modes](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L83-L89),
[bufmgr.c#LockBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3585-L3607)).

The worker never follows a downlink or sibling link to choose the next page.
Physical block order drives the scan; `btpo_next` is read only to calculate
fragmentation
([pgstatindex.c#scan-and-fragment-test](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).

### Page classification

Each scanned block enters exactly one branch:

| Test order | Counter | What the code counts |
|---|---:|---|
| `P_ISDELETED(opaque)` | `deleted_pages` | Every page carrying `BTP_DELETED`, without checking whether its deletion XID is old enough for page reuse ([pgstatindex.c#classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtpage.c#_bt_page_recyclable](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L931-L963)). |
| Remaining `P_IGNORE(opaque)` | `empty_pages` | A page carrying `BTP_HALF_DEAD`; deleted pages have already taken the first branch ([nbtree.h#page-state-macros](../../../../raw/postgres-12/src/include/access/nbtree.h#L183-L196)). |
| `P_ISLEAF(opaque)` | `leaf_pages` | A live, non-half-dead leaf; this branch also accumulates capacity, free space, and backward right-links ([pgstatindex.c#leaf-branch](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307)). |
| Fallback | `internal_pages` | For a well-formed B-tree, any remaining non-leaf page, including an internal root; `P_ISROOT` has no separate counter ([pgstatindex.c#classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtree.h#page-flags](../../../../raw/postgres-12/src/include/access/nbtree.h#L70-L78)). |

`empty_pages` is therefore a state count, not an item-count test. PostgreSQL
marks an empty deletable leaf half-dead during the first page-deletion stage,
removes its old high key, and installs one dummy high-key tuple. A live leaf
with no data items remains in `leaf_pages`; the root cannot be deleted merely
because it is empty
([nbtpage.c#page-deletion-preconditions](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1356-L1389),
[nbtpage.c#mark-half-dead](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1642-L1665)).

The classification ignores orthogonal flags such as `BTP_HAS_GARBAGE`,
`BTP_SPLIT_END`, and `BTP_INCOMPLETE_SPLIT`. Such a live page is still counted
only as leaf or internal
([nbtree.h#page-flags](../../../../raw/postgres-12/src/include/access/nbtree.h#L70-L78),
[pgstatindex.c#classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)).

### How `index_size` is calculated

The worker does not call a byte-size function at result time. It adds the four
page-class counters, adds one metapage, and multiplies by the compile-time
`BLCKSZ`
([pgstatindex.c#index-size](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L346)).
Because every scanned block takes exactly one classification branch, a
completed call has this exact arithmetic invariant:

```text
1 + internal_pages + leaf_pages + empty_pages + deleted_pages
    = nblocks captured before the loop
```

Consequently, `index_size` equals the captured main-fork block count times
`BLCKSZ`. It includes deleted and half-dead pages, but excludes the free-space
map and any init fork because the worker uses `MAIN_FORKNUM` only
([pgstatindex.c#length-and-scan](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315),
[relpath.h#fork-numbers](../../../../raw/postgres-12/src/include/common/relpath.h#L32-L53)).
The documentation describes the same metapage relationship
([pgstattuple.sgml#index-size-note](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L273)).

Pages appended after the one-time `nblocks` read are outside this call's scan
and size. B-tree VACUUM, by contrast, takes the relation-extension lock,
rechecks the relation length, and explains why extension synchronization is
needed around an in-progress all-zero page
([pgstatindex.c#one-time-length](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L271),
[nbtree.c#btvacuumscan-extension-loop](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L991-L1041)).

### How `avg_leaf_density` is calculated

For each live leaf page, the worker computes:

```text
max_avail = BLCKSZ - (BLCKSZ - pd_special + SizeOfPageHeaderData)
          = pd_special - SizeOfPageHeaderData
```

This is the page area between the fixed page header and B-tree special space.
It includes the area used by line pointers and index tuples. `pd_special` is
the start of special space, while `SizeOfPageHeaderData` deliberately excludes
the line-pointer array
([pgstatindex.c#leaf-capacity](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L299),
[bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164),
[bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L207-L217)).

The free-space term is `PageGetFreeSpace(page)`, not the raw gap. It calculates
`pd_upper - pd_lower`, returns zero if less than one line pointer remains, and
otherwise subtracts one `ItemIdData` slot for a future tuple
([bufpage.c#PageGetFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597)).
`PageGetExactFreeSpace` would return the raw nonnegative gap without that
reservation, but `pgstatindex` does not call it
([bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L626-L647),
[pgstatindex.c#leaf-free-space](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L296-L299)).

The final percentage is a ratio of sums across live leaf pages, not a scan of
live index tuples:

```text
100 - sum(PageGetFreeSpace(page))
      / sum(pd_special - SizeOfPageHeaderData) * 100
```

This measures physical occupancy. High keys, line pointers, and the stored
bytes of `LP_DEAD` index tuples all reduce free space. Marking an item
`LP_DEAD` preserves its storage, and `pgstatindex` does not inspect item states
or `BTP_HAS_GARBAGE`
([nbtree.h#high-key-layout](../../../../raw/postgres-12/src/include/access/nbtree.h#L198-L219),
[itemid.h#ItemIdMarkDead](../../../../raw/postgres-12/src/include/storage/itemid.h#L160-L182),
[nbtutils.c#LP_DEAD-and-BTP_HAS_GARBAGE](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1687-L1813),
[pgstatindex.c#leaf-branch](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307)).

The result is formatted to two decimal places. If no live leaf contributes
capacity, the worker emits `NaN`
([pgstatindex.c#density-result](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351)).

### How `leaf_fragmentation` is calculated

For each live leaf page at physical block `blkno`, the worker increments
`fragments` when its right-link is not `P_NONE` and is strictly lower than
`blkno`:

```text
btpo_next != P_NONE && btpo_next < blkno
```

`btpo_next` is the B-tree link to the page's right sibling, and `P_NONE` is
zero. Deleted and half-dead pages are excluded because their branches run
before the live-leaf branch
([pgstatindex.c#fragment-count](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L307),
[nbtree.h#BTPageOpaqueData](../../../../raw/postgres-12/src/include/access/nbtree.h#L29-L68),
[nbtree.h#P_NONE](../../../../raw/postgres-12/src/include/access/nbtree.h#L173-L194)).

The output is:

```text
fragments / leaf_pages * 100
```

The denominator includes every live leaf, including the rightmost leaf whose
right-link is `P_NONE`. The result is formatted to two decimal places, or is
`NaN` when `leaf_pages` is zero
([pgstatindex.c#fragmentation-result](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307),
[pgstatindex.c#fragmentation-result-format](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)).

This is a backward-right-link percentage. It is not a count of fragmented
runs, bytes, filesystem extents, or reciprocal-link errors. The worker tests
one link independently on each physically scanned live leaf and does not
validate the sibling chain
([pgstatindex.c#scan-and-fragment-test](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).

### Result construction and edge paths

After the scan, the worker closes the relation and releases `AccessShareLock`.
It then requires the caller's declared result type to be composite, formats ten
C strings, feeds them through the tuple descriptor's input functions, and
returns the resulting record
([pgstatindex.c#close-and-build-result](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L317-L365),
[relation.c#relation_close](../../../../raw/postgres-12/src/backend/access/common/relation.c#L196-L217)).

| Output column | Source or calculation |
|---|---|
| `version` | `BTMetaPageData.btm_version` ([pgstatindex.c#metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253)). |
| `tree_level` | `BTMetaPageData.btm_level`, the true root's level ([nbtree.h#BTMetaPageData](../../../../raw/postgres-12/src/include/access/nbtree.h#L97-L110)). |
| `index_size` | Captured classified blocks, including the metapage, times `BLCKSZ` ([pgstatindex.c#index-size](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L346)). |
| `root_block_no` | `BTMetaPageData.btm_root` ([pgstatindex.c#metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253)). |
| `internal_pages` | Classification fallback after deleted, half-dead, and leaf tests ([pgstatindex.c#classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)). |
| `leaf_pages` | Live leaf branch count ([pgstatindex.c#leaf-branch](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307)). |
| `empty_pages` | Half-dead branch count ([pgstatindex.c#classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)). |
| `deleted_pages` | `BTP_DELETED` branch count ([pgstatindex.c#classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310)). |
| `avg_leaf_density` | Summed physical leaf occupancy, formatted to two decimals, or `NaN` ([pgstatindex.c#density-result](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L351)). |
| `leaf_fragmentation` | Backward-right-link count divided by live leaves, formatted to two decimals, or `NaN` ([pgstatindex.c#fragmentation-result-format](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356)). |

The explicit errors and exits are:

- SQL `STRICT` handles a null argument without entering C
  ([pgstattuple--1.4--1.5.sql#pgstatindex-definitions](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L92),
  [xfunc.sgml#STRICT](../../../../raw/postgres-12/doc/src/sgml/xfunc.sgml#L2397-L2404)).
- Name/OID resolution can fail while opening the relation; the legacy wrappers
  can first reject a non-superuser
  ([pgstatindex.c#wrappers](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L213),
  [relation.c#relation_open](../../../../raw/postgres-12/src/backend/access/common/relation.c#L35-L79)).
- The worker raises `wrong_object_type` for a non-physical-B-tree object and
  `feature_not_supported` for another session's temporary relation
  ([pgstatindex.c#object-errors](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238)).
- `CHECK_FOR_INTERRUPTS()` makes a long page scan cancellable
  ([pgstatindex.c#scan-loop](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315)).
- A non-composite call context fails with `return type must be a row type` after
  the relation has been closed
  ([pgstatindex.c#result-type-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L317-L332)).

### Concurrency and integrity limits

`pgstatindex` derives its counters from the page images it reads, but those
images do not form one whole-index snapshot. The documentation explicitly says
that results accumulate page by page and should not be expected to represent
an instantaneous index snapshot
([pgstattuple.sgml#page-by-page-caveat](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279)).

| Boundary | PostgreSQL 12 behavior |
|---|---|
| Relation lock | Both wrappers hold `AccessShareLock` during the scan. That mode conflicts only with `AccessExclusiveLock`; ordinary index maintenance opens indexes with `RowExclusiveLock`, as does lazy VACUUM, so inserts, deletes, page splits, and VACUUM can overlap the scan ([lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105), [execIndexing.c#ExecOpenIndices](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L141-L213), [vacuumlazy.c#index-lock](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L279-L288)). |
| Metapage | It is pinned but not content-locked, and its three reported fields are loaded before the page scan ([pgstatindex.c#unlocked-metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253)). |
| Non-metapage blocks | Each page gets a shared content lock while inspected, but that lock is released before the next page. The call can therefore combine states observed at different times ([pgstatindex.c#scan-loop](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L315), [bufmgr.c#LockBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3585-L3607)). |
| Relation growth | `nblocks` is read once without the relation-extension lock. Later blocks are omitted; an extension already in progress also lacks the synchronization used by B-tree VACUUM around newly extended all-zero pages ([pgstatindex.c#one-time-length](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L266-L271), [nbtree.c#btvacuumscan-extension-loop](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L991-L1041)). |
| Generic buffer validation | `RBM_NORMAL` validates a page header and, when enabled, its checksum when reading from storage, but accepts an all-zero page as generically valid ([bufmgr.c#ReadBufferExtended](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L600-L638), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L142)). |
| B-tree validation | The worker does not call `_bt_getmeta` or `_bt_checkpage`. It does not verify metapage magic/version, reject an all-zero ordinary page, or verify `BTPageOpaqueData` special-space size; normal B-tree accessors do those checks ([pgstatindex.c#raw-page-access](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315), [nbtpage.c#_bt_getmeta](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148), [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)). |

Therefore, use `pgstatindex` as a physical diagnostic, not as proof that a
B-tree is structurally valid and not as a transactionally consistent size or
occupancy snapshot. It does not validate parent/downlink structure, sibling
reciprocity, key order, tuple-to-heap correspondence, or index-state flags
([pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365),
[pg_index.h#index-state-flags](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L32-L44)).

### Regression coverage

The contrib Makefile defines one regression target, `pgstattuple`. Its test
creates an empty primary-key B-tree and calls `pgstatindex` through an untyped
literal, explicit `text`, `name`, and `regclass`. The expected output checks a
one-metapage index, root block 0, zero page counters, and `NaN` ratios
([Makefile#regression-target](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13),
[pgstattuple.sql#empty-B-tree](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L37),
[pgstattuple.out#empty-B-tree](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)).

The same test verifies wrong-AM errors, unsupported relation kinds, and success
for a physical B-tree index on a table partition
([pgstattuple.sql#object-tests](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L47-L113),
[pgstattuple.out#object-test-results](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L126-L236)).

The checked-in test does not create a populated B-tree. It therefore does not
assert a non-`NaN` density, internal-page counting, backward-link
fragmentation, half-dead pages, deleted pages, percentage rounding, concurrent
extension, malformed-page handling, another-session temporary-index errors,
index-state flags, or non-superuser privileges. The module has no declared
isolation or TAP test target
([Makefile#regression-target](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13),
[pgstattuple.sql#complete-test](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119)).

## Context Reviewed

- Pinned checkout: `raw/postgres-12/` at
  `45b88269a353ad93744772791feb6d01bc7e1e42` (`REL_12_2`).
- SQL/module boundary: control file, Makefile, 1.4 base SQL, 1.4-to-1.5 update
  SQL, extension install-path code, fmgr V1 registration and loading.
- Core worker: every `pgstatindex` wrapper, `BTIndexStat`, and every statement
  in `pgstatindex_impl`.
- B-tree boundary: metapage and opaque structures, true/fast roots, root split,
  page deletion, half-dead/deleted/recyclable states, high keys, and ordinary
  B-tree page checks.
- Storage and concurrency: relation opening/closing, lock conflict table, DML
  and VACUUM index locks, `ReadBufferExtended`, generic page validation,
  per-buffer content locks, block-count path, forks, and extension-lock handling
  in B-tree VACUUM.
- Tests and docs: complete `pgstattuple.sql`/expected output, contrib Makefile,
  and the PostgreSQL 12 `pgstattuple` documentation.
- Direct history inspected: `fd321a1dfd6` (v1.5 privilege split),
  `48e6c943e5f` (page classification, metapage inclusion in `index_size`, and
  root counting), and `af7d181298f` (explicit empty-index `NaN` handling); each
  is an ancestor of the pinned commit.
- Supplementary exact-pin check: built PostgreSQL 12.2 and `pgstattuple` under
  `.wiki-runtime/`, ran the module's `installcheck` (the one declared test
  passed), and spot-checked a populated index before and after `VACUUM`. The
  pinned source remains the evidence base.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Fresh v1.5 installation follows the 1.4 base plus update path | [Makefile#extension-scripts](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L3-L13), [extension.c#install-path-selection](../../../../raw/postgres-12/src/backend/commands/extension.c#L1297-L1400) |
| Both SQL overloads reach `pgstatindex_impl` under `AccessShareLock` | [pgstatindex.c#wrappers](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L216) |
| Only physical built-in B-tree indexes are accepted | [pgstatindex.c#object-check](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L224-L238), [pg_am.dat#BTREE_AM_OID](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L23) |
| Reported metadata comes from true-root fields on block 0 | [pgstatindex.c#metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253), [nbtree.h#BTMetaPageData](../../../../raw/postgres-12/src/include/access/nbtree.h#L90-L110) |
| The metapage read is neither content-locked nor B-tree-validated | [pgstatindex.c#metapage-read](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L253), [nbtpage.c#_bt_getmeta](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L111-L148) |
| The worker captures main-fork length once and scans every captured non-metapage block | [pgstatindex.c#scan-loop](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L255-L315), [bufmgr.h#RelationGetNumberOfBlocks](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199) |
| `empty_pages` means half-dead pages, while deleted-page recyclability is not distinguished | [pgstatindex.c#classification](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L286-L310), [nbtpage.c#_bt_page_recyclable](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L931-L963) |
| `index_size` is captured main-fork blocks times `BLCKSZ` | [pgstatindex.c#index-size](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L333-L346), [relpath.h#fork-numbers](../../../../raw/postgres-12/src/include/common/relpath.h#L32-L53) |
| Density is summed physical leaf occupancy using `PageGetFreeSpace` | [pgstatindex.c#leaf-and-result-formulas](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L356), [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L597) |
| `LP_DEAD` tuple storage still contributes to density | [itemid.h#ItemIdMarkDead](../../../../raw/postgres-12/src/include/storage/itemid.h#L160-L182), [pgstatindex.c#leaf-branch](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L292-L307) |
| Fragmentation is the percentage of live leaves with a backward physical right-link | [pgstatindex.c#fragmentation](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L300-L307), [pgstatindex.c#fragmentation-result](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L352-L356) |
| Both percentages are limited to two decimal places | [pgstatindex.c#result-formulas](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L347-L359) |
| Results are page-by-page, not a whole-index snapshot | [pgstattuple.sgml#page-by-page-caveat](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L268-L279), [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105) |
| The worker is not a B-tree integrity checker | [pgstatindex.c#raw-page-access](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L240-L315), [nbtpage.c#B-tree-page-checks](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720) |
| In-tree coverage is limited to empty output and object/error paths | [pgstattuple.sql#complete-test](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119), [pgstattuple.out#complete-output](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L1-L248) |

## Open Questions

- No unresolved source claim remains for the PostgreSQL 12.2 calculation path
  documented above.
- The in-tree verification gap remains substantial: there is no populated,
  concurrent, malformed-page, permissions, temporary-relation, or index-state
  regression/isolation/TAP case
  ([Makefile#regression-target](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L13),
  [pgstattuple.sql#complete-test](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119)).
- Malformed-page outcomes can depend on whether the page is already cached,
  checksums are enabled, or assertions are compiled in. This page therefore
  records the source-verifiable absence of B-tree-specific checks but does not
  promise one particular error or misclassification for every corruption shape
  ([bufmgr.c#ReadBufferExtended](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L600-L638),
  [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L142),
  [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L720)).

## Source References

- [pgstatindex.c#entry-points-and-structures](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L47-L132) - fmgr registrations, AM tests, accumulators, and worker declaration.
- [pgstatindex.c#wrappers](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L134-L216) - legacy and v1.5 text/regclass wrappers.
- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L215-L365) - checks, scan, formulas, and result construction.
- [pgstattuple.control](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5) - default extension version and module path.
- [Makefile#pgstattuple](../../../../raw/postgres-12/contrib/pgstattuple/Makefile#L1-L24) - shared module, SQL scripts, and regression target.
- [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L79) - legacy SQL-to-C bindings.
- [pgstattuple--1.4--1.5.sql#pgstatindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L22-L100) - v1.5 bindings, attributes, revoke, and grants.
- [extension.c#CreateExtensionInternal](../../../../raw/postgres-12/src/backend/commands/extension.c#L1261-L1400) - default version and install/update-path selection.
- [extension.c#install-then-update](../../../../raw/postgres-12/src/backend/commands/extension.c#L1536-L1550) - base script followed by update scripts.
- [pgstattuple.sgml#pgstatindex](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L295) - documented columns and page-by-page caveat.
- [pgstattuple.sql#pgstatindex-tests](../../../../raw/postgres-12/contrib/pgstattuple/sql/pgstattuple.sql#L1-L119) - complete module regression input.
- [pgstattuple.out#pgstatindex-results](../../../../raw/postgres-12/contrib/pgstattuple/expected/pgstattuple.out#L44-L236) - empty output and object/error results.
- [nbtree.h#page-structures](../../../../raw/postgres-12/src/include/access/nbtree.h#L29-L135) - opaque and metapage layouts, flags, and format versions.
- [nbtree.h#page-navigation-macros](../../../../raw/postgres-12/src/include/access/nbtree.h#L173-L219) - `P_NONE`, state macros, and high-key layout.
- [nbtpage.c#metapage-access](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L46-L148) - initialization, upgrade, and standard validation.
- [nbtpage.c#root-access](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L228-L625) - true/fast root selection and root height.
- [nbtpage.c#page-checks-and-reuse](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L688-L963) - ordinary-page checks, extension, zero pages, and recyclability.
- [nbtpage.c#page-deletion](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1320-L1517) - deletion preconditions and two-stage flow.
- [nbtpage.c#mark-half-dead](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L1520-L1707) - half-dead transition and dummy high key.
- [nbtinsert.c#new-root](../../../../raw/postgres-12/src/backend/access/nbtree/nbtinsert.c#L2050-L2110) - locked metapage update during a root split.
- [nbtree.c#btvacuumscan](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L944-L1041) - repeated length checks and extension synchronization.
- [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L229) - page header, header size, empty/new-page macros.
- [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L142) - generic page-header/checksum validation and all-zero acceptance.
- [bufpage.c#free-space-functions](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L572-L647) - allocatable versus exact free space.
- [bufmgr.c#ReadBufferExtended](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L600-L669) - read mode, pin, generic validation, and other-session temp check.
- [bufmgr.c#RelationGetNumberOfBlocksInFork](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2790-L2810) - storage-manager block count.
- [bufmgr.c#LockBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L3585-L3607) - buffer content-lock modes.
- [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L61-L105) - table/index relation-lock conflict matrix.
- [relation.c#relation-open-close](../../../../raw/postgres-12/src/backend/access/common/relation.c#L35-L79) and [relation.c#relation_close](../../../../raw/postgres-12/src/backend/access/common/relation.c#L196-L217) - relation lock and relcache boundary.
- [execIndexing.c#ExecOpenIndices](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L141-L213) - DML's `RowExclusiveLock` on indexes.
- [vacuumlazy.c#index-lock](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L279-L288) - lazy VACUUM's index lock.
- [pg_am.h#generated-header-include](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L18-L29), [pg_am.dat#BTREE_AM_OID](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L23), and [catalog/Makefile#generated-catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L28-L100) - generated OID symbol path.
- [fmgr.h#PG_FUNCTION_INFO_V1](../../../../raw/postgres-12/src/include/fmgr.h#L383-L413) and [fmgr.c#fetch_finfo_record](../../../../raw/postgres-12/src/backend/utils/fmgr/fmgr.c#L458-L511) - SQL-callable C function metadata.
- [itemid.h#ItemIdMarkDead](../../../../raw/postgres-12/src/include/storage/itemid.h#L160-L182) and [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1687-L1813) - retained `LP_DEAD` storage and garbage hinting.
- [pg_index.h#index-state-flags](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L32-L44) - validity/readiness/liveness fields that the worker does not inspect.

## Navigation

- [v12/index](../../index.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](../query-planning/bloated-indexes-query-planner.md)
- [Impact of B-Tree Leaf Density (60% vs 90%) on Index Scan Queries in PostgreSQL 12 (unverified)](leaf-density-60-vs-90-query-impact.md)
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](leaf-density-vs-fragmentation-index-scan-io.md)
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 12 (unverified)](pgstatindex-sample-variant-proposal.md)
- [versions](../../../versions.md)
