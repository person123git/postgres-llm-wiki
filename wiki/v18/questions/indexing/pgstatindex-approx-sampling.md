---
type: question
version: 18
pinned_commit: 6cb307251c5c6261286c1566496920976640108e
verified: false
verified_by_agent: claude-opus-4-7 2026-05-25T13:59:04Z
---

# Why pgstatindex Cannot Use pgstattuple_approx-Style Approximation in PostgreSQL 18 (unverified)

## Question

Follow AGENTS.md.
In PostgreSQL 18, why is it not possible to have an implementation of `pgstatindex` that samples the index similarly to `pgstattuple_approx`?

## Answer Up Front

It is possible to create a new approximate B-tree diagnostic function, but it
cannot use the same shortcut as `pgstattuple_approx` and still mean the same
thing as `pgstatindex`. `pgstattuple_approx` is heap-specific: it skips heap
pages only when the visibility map proves the heap page is all-visible, uses the
heap free space map (FSM) for those skipped pages, scans the remaining heap
pages, and then estimates live tuple count with `vac_estimate_reltuples`
([pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L48-L204),
[vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-18/src/backend/commands/vacuum.c#L1332-L1398)).
The function explicitly restricts itself to ordinary relations, materialized
views, TOAST relations, and the heap table access method because that is where
those visibility-map/FSM assumptions apply
([pgstatapprox.c#pgstattuple_approx_internal](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L241-L287)).

`pgstatindex` has a different contract. It reads the B-tree metapage, then reads
and share-locks every index block except the metapage, classifies each page as
deleted, ignored/half-dead, leaf, or internal, and computes
`avg_leaf_density` from exact free space on every live leaf page
([pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L328)).
The B-tree/index FSM is not a per-page free-space estimator for live leaf pages:
it records index pages as either completely free or in use, using `BLCKSZ - 1`
for free and `0` for used
([indexfsm.c](../../../../raw/postgres-18/src/backend/storage/freespace/indexfsm.c#L14-L20),
[indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-18/src/backend/storage/freespace/indexfsm.c#L48-L65)).
So there is no v18 side channel equivalent to "VM says safe to skip, FSM gives
the missing space" for in-use B-tree leaf pages.

The practical answer is: do not change `pgstatindex` to sample silently. Add a
separate `pgstatindex_approx`-style function if approximate answers are useful,
and make the output names and error model explicit.

## What pgstattuple_approx Actually Does

`pgstattuple_approx` is not a random sampler. It still iterates over the heap
block number range; the shortcut is that all-visible heap pages do not need to
be read. For an all-visible page it calls `GetRecordedFreeSpace()` and assumes
the rest of the block is live tuple space. For a page that is not all-visible,
it reads and locks the buffer, adds exact free space with
`PageGetExactFreeSpace()`, and classifies each heap tuple with
`HeapTupleSatisfiesVacuum()`
([pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L71-L170)).

The PostgreSQL docs describe the same split: `pgstattuple_approx` avoids a full
heap scan by skipping pages marked all-visible in the visibility map, takes
free-space values for skipped pages from the FSM, scans unskipped pages, and
estimates live tuple count at the end
([pgstattuple.sgml#pgstattuple_approx](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L515-L539)).
The visibility map itself is defined as two bits per heap page, where an
all-visible bit means all tuples on that heap page are visible to all
transactions; it is conservative, so a set bit is trusted but a clear bit does
not prove the opposite
([visibilitymap.c#visibility-map-notes](../../../../raw/postgres-18/src/backend/access/heap/visibilitymap.c#L23-L55),
[visibilitymapdefs.h#BITS_PER_HEAPBLOCK](../../../../raw/postgres-18/src/include/access/visibilitymapdefs.h#L16-L23)).

## Why That Does Not Transfer To pgstatindex

`pgstatindex` needs page structure, not just relation size. The SQL interface
returns exact-looking B-tree fields: version, root block, internal page count,
leaf page count, empty page count, deleted page count, average leaf density, and
leaf fragmentation
([pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L31)).
The implementation fills those fields by reading each page: deleted pages are
counted from `P_ISDELETED`, half-dead pages from `P_IGNORE`, leaf pages from
`P_ISLEAF`, and internal pages from the fallback branch
([pgstatindex.c#page-classification](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L275-L328),
[nbtree.h#btpo_flags](../../../../raw/postgres-18/src/include/access/nbtree.h#L76-L87)).

`avg_leaf_density` is especially tied to live leaf pages. The numerator is the
sum of exact free space from `PageGetExactFreeSpace()`, which returns
`pd_upper - pd_lower`, and the denominator is the usable space below the B-tree
special area on every leaf page
([pgstatindex.c#avg_leaf_density](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L305-L367),
[bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L951-L972)).
If a live B-tree leaf page is skipped, PostgreSQL 18 has no stored value that
can substitute for its exact `pd_upper - pd_lower`.

`leaf_fragmentation` also needs leaf-page right links. `pgstatindex` increments
the counter when a leaf page's `btpo_next` points to an earlier physical block,
which requires reading the page opaque data of the counted leaf page
([pgstatindex.c#leaf_fragmentation](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L315-L320),
[nbtree.h#BTPageOpaqueData](../../../../raw/postgres-18/src/include/access/nbtree.h#L32-L70)).
A sampler can estimate that rate, but skipped pages do not have a separate
summary that can recover the exact count.

The heap FSM stores an approximate free-space category for each heap block, and
`GetRecordedFreeSpace()` returns the category value for a block
([freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-18/src/backend/storage/freespace/freespace.c#L36-L67),
[freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-18/src/backend/storage/freespace/freespace.c#L239-L260)).
The index FSM deliberately uses the same machinery differently: it tracks only
whether an index page is free enough to be reused or is already in use
([indexfsm.c](../../../../raw/postgres-18/src/backend/storage/freespace/indexfsm.c#L14-L20)).
B-tree allocation checks candidate FSM pages for actual recyclability before
reuse, which reinforces that the index FSM is a reusable-page locator, not a
leaf-density store
([nbtpage.c#_bt_allocbuf](../../../../raw/postgres-18/src/backend/access/nbtree/nbtpage.c#L868-L988)).

## What Could Be Implemented

A separate approximate function could sample index blocks. It could read the
metapage exactly, sample a set of block numbers, classify sampled B-tree pages,
and report estimates such as `approx_leaf_pages`,
`approx_deleted_pages`, `approx_avg_leaf_density`,
`approx_leaf_fragmentation`, `sampled_pages`, and `scanned_percent`. That would
be a statistical estimate of the current physical index, not the same result as
`pgstatindex`, because unvisited live leaf pages contribute no measured free
space in v18
([pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L328),
[indexfsm.c](../../../../raw/postgres-18/src/backend/storage/freespace/indexfsm.c#L14-L20)).

Changing `pgstatindex` itself to do that would be misleading. The docs say
`pgstatindex` accumulates results page-by-page and should not be treated as an
instantaneous whole-index snapshot; they do not describe sampling or approximate
columns
([pgstattuple.sgml#pgstatindex](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L268-L279)).
By contrast, `pgstattuple_approx` has an explicitly approximate result shape
with `scanned_percent`, `approx_tuple_count`, `approx_tuple_len`, and
`approx_free_space`
([pgstattuple--1.4.sql#pgstattuple_approx](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4.sql#L83-L95)).

If the goal is `avg_leaf_density` with minimal extra I/O after VACUUM, the
better v18 hook is not random sampling. When B-tree VACUUM actually enters
`btvacuumscan`, it scans all index pages except the metapage and `btvacuumpage`
takes a cleanup lock on every live leaf page, so exact leaf-density
accumulation can piggyback on pages that VACUUM already has in memory
([nbtree.c#btvacuumscan](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1173-L1346),
[nbtree.c#btvacuumpage](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1360-L1642)).
That does not help when `btvacuumcleanup` decides that the whole physical
B-tree scan can be skipped
([nbtree.c#btvacuumcleanup](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1097-L1140)).

## Current pgstatindex Tests

The current `pgstatindex` coverage is in the `pgstattuple` regression test:
`contrib/pgstattuple/Makefile` sets `REGRESS = pgstattuple`, which runs
`sql/pgstattuple.sql` against `expected/pgstattuple.out`
([Makefile#REGRESS](../../../../raw/postgres-18/contrib/pgstattuple/Makefile#L1-L17),
[pgstattuple.sql](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138),
[pgstattuple.out](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L1-L305)).

| Test area | What the test does | What it proves |
|---|---|---|
| Empty primary-key B-tree, text-ish input paths | The test creates `test (a int primary key, b int[])`, then selects the visible `pgstatindex` columns from `pgstatindex('test_pkey')`, `pgstatindex('test_pkey'::text)`, `pgstatindex('test_pkey'::name)`, and `pgstatindex('test_pkey'::regclass)` ([pgstattuple.sql#empty-btree-pgstatindex](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L9-L37)). The expected output is B-tree version `4`, tree level `0`, one block of index size after dividing by `current_setting('block_size')`, root block `0`, zero internal/leaf/empty/deleted pages, and `NaN` for both `avg_leaf_density` and `leaf_fragmentation` ([pgstattuple.out#empty-btree-pgstatindex](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L44-L82)). | This checks the empty-index edge case and the text/name/regclass entry points. It matches `pgstatindex_impl`'s behavior of returning `NaN` when no leaf-page denominator exists ([pgstatindex.c#pgstatindex-result](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L336-L369)). |
| Wrong index access method | After creating a GIN index and a hash index, the test calls `pgstatindex('test_ginidx')` and `pgstatindex('test_hashidx')`; both are expected to error with "is not a btree index" ([pgstattuple.sql#wrong-index-am](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L47-L63), [pgstattuple.out#wrong-index-am](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L126-L152)). | This checks the front-door type guard: `pgstatindex_impl` rejects any relation that is not both an index and a B-tree index ([pgstatindex.c#type-check](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L225)). It does not exercise GIN or hash page inspection; those have separate functions. |
| Unsupported relation kinds | The test calls `pgstatindex` on a partitioned table, a view, a foreign table, a regular table partition, and a sequence; each expected result is "is not a btree index" ([pgstattuple.sql#unsupported-relations](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L65-L113), [pgstattuple.sql#sequence-failure](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L122-L131), [pgstattuple.out#unsupported-relations](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L153-L213), [pgstattuple.out#partition-and-sequence-failures](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L253-L299)). | This checks that `pgstatindex` rejects stored or non-stored objects that are not physical B-tree indexes. The expected error is the same guard in `pgstatindex_impl`, not a deeper page-level corruption or AM-specific path ([pgstatindex.c#type-check](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L225)). |
| Physical index on a partition | The test creates `test_partition` as a partition of `test_partitioned`, creates `test_partition_idx` on that partition, then calls `pgstatindex('test_partition_idx')`; the expected composite output is `(4,0,8192,0,0,0,0,0,NaN,NaN)` ([pgstattuple.sql#partition-index-success](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L99-L120), [pgstattuple.out#partition-index-success](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L214-L274)). | This checks that a real physical B-tree index attached to a partition is accepted and inspected like any other empty B-tree index. It does not test a partitioned index object itself with `pgstatindex`; the SQL creates `test_partitioned_index`, but the direct `pgstatindex` calls in this file target the table, the partition, and the partition's physical index ([pgstattuple.sql#partition-setup](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L65-L77), [pgstattuple.sql#partition-index-success](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L99-L120)). |

The important coverage gap for this page is that the regression test does not
populate a B-tree and check non-`NaN` `avg_leaf_density`, nonzero
`leaf_fragmentation`, internal pages, deleted pages, half-dead/empty pages, or
approximate behavior. It also does not test an approximate index function,
because the v18 extension's installed SQL (`default_version = '1.5'`) defines
`pgstatindex`, `pgstatginindex`, `pg_relpages`, `pgstattuple`, and
`pgstattuple_approx` in the base script, then adds `pgstathashindex` in the
1.4->1.5 upgrade script, but no `pgstatindex_approx`
([pgstattuple.control#default_version](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple.control#L3),
[pgstattuple--1.4.sql](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L95),
[pgstattuple--1.4--1.5.sql#pgstathashindex](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L123-L136),
[pgstattuple.sql](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L18-L120)).

## Source History

The pinned checkout's source history matches the current code shape. Commit
`5850b20f58` introduced `pgstattuple_approx` as a heap bloat estimator using
the visibility map plus free space map, and commit `7c91a0364f` later synced
its live-tuple estimate with PostgreSQL's `reltuples` estimation rules. I found
no local source-history entry in the pinned checkout that adds or discusses a
`pgstatindex_approx` function.

## Context Reviewed

- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L328)
- [pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L48-L204)
- [pgstatapprox.c#pgstattuple_approx_internal](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L241-L287)
- [pgstattuple.c#pgstat_index](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple.c#L533-L603)
- [Makefile#REGRESS](../../../../raw/postgres-18/contrib/pgstattuple/Makefile#L1-L17)
- [pgstattuple.sql](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L1-L138)
- [pgstattuple.out](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L1-L305)
- [pgstattuple--1.4.sql](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L95)
- [pgstattuple.sgml](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L161-L613)
- [visibilitymap.c#visibility-map-notes](../../../../raw/postgres-18/src/backend/access/heap/visibilitymap.c#L23-L55)
- [freespace.c](../../../../raw/postgres-18/src/backend/storage/freespace/freespace.c#L36-L260)
- [indexfsm.c](../../../../raw/postgres-18/src/backend/storage/freespace/indexfsm.c#L1-L74)
- [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-18/src/include/access/nbtree.h#L32-L120)
- [nbtree.c#btvacuumcleanup](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1097-L1140)
- [nbtree.c#btvacuumscan](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1173-L1346)
- [nbtree.c#btvacuumpage](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1360-L1642)
- Pinned checkout history: `git show 5850b20f58`, `git show 7c91a0364f`

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstattuple_approx` skips heap pages only when VM marks them all-visible and uses FSM for skipped pages | [pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L48-L204), [pgstattuple.sgml#pgstattuple_approx](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L515-L539) |
| `pgstattuple_approx` is restricted to heap-backed relation kinds | [pgstatapprox.c#pgstattuple_approx_internal](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L270-L287) |
| Visibility-map bits are per heap page, not per index page | [visibilitymap.c#visibility-map-notes](../../../../raw/postgres-18/src/backend/access/heap/visibilitymap.c#L23-L55), [visibilitymapdefs.h#BITS_PER_HEAPBLOCK](../../../../raw/postgres-18/src/include/access/visibilitymapdefs.h#L16-L23) |
| `pgstatindex` reads and classifies every B-tree block except the metapage | [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L328) |
| `avg_leaf_density` requires exact free space from every counted leaf page | [pgstatindex.c#avg_leaf_density](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L305-L367), [bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-18/src/backend/storage/page/bufpage.c#L951-L972) |
| Index FSM tracks completely free vs used pages, not live leaf free bytes | [indexfsm.c](../../../../raw/postgres-18/src/backend/storage/freespace/indexfsm.c#L14-L20), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-18/src/backend/storage/freespace/indexfsm.c#L48-L65) |
| B-tree VACUUM can compute exact density without extra page reads only when its physical scan actually runs | [nbtree.c#btvacuumscan](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1173-L1346), [nbtree.c#btvacuumcleanup](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1097-L1140) |
| Current `pgstatindex` regression coverage focuses on empty B-tree output, wrong-object-type failures, and one physical partition-index success case | [Makefile#REGRESS](../../../../raw/postgres-18/contrib/pgstattuple/Makefile#L1-L17), [pgstattuple.sql](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L18-L131), [pgstattuple.out](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L44-L299) |

## Source References

- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L212-L328) - `pgstatindex` metapage read, full B-tree page scan, page classification, and leaf-page accumulation.
- [pgstatindex.c#avg_leaf_density](../../../../raw/postgres-18/contrib/pgstattuple/pgstatindex.c#L305-L367) - exact `avg_leaf_density` and `leaf_fragmentation` calculations.
- [pgstatapprox.c#statapprox_heap](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L48-L204) - heap-only approximate scan that skips all-visible pages and uses FSM values.
- [pgstatapprox.c#pgstattuple_approx_internal](../../../../raw/postgres-18/contrib/pgstattuple/pgstatapprox.c#L241-L287) - relation-kind and heap-AM restrictions for `pgstattuple_approx`.
- [Makefile#REGRESS](../../../../raw/postgres-18/contrib/pgstattuple/Makefile#L1-L17) - regression-test entry for the `pgstattuple` test file.
- [pgstattuple.control#default_version](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple.control#L3) - the installed extension default version (`1.5`).
- [pgstattuple--1.4.sql](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L95) - base-script SQL result shapes for `pgstatindex` and `pgstattuple_approx`.
- [pgstattuple--1.4--1.5.sql#pgstathashindex](../../../../raw/postgres-18/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L123-L136) - the 1.4->1.5 upgrade that adds `pgstathashindex`; no `pgstatindex_approx` is added.
- [pgstattuple.sql#empty-btree-pgstatindex](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37) - empty B-tree and input-type `pgstatindex` calls.
- [pgstattuple.sql#wrong-index-am](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L47-L63) - wrong-index-AM `pgstatindex` calls.
- [pgstattuple.sql#unsupported-relations](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L65-L113) and [pgstattuple.sql#sequence-failure](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L122-L131) - unsupported relation-kind `pgstatindex` calls.
- [pgstattuple.sql#partition-index-success](../../../../raw/postgres-18/contrib/pgstattuple/sql/pgstattuple.sql#L99-L120) - physical partition-index success case.
- [pgstattuple.out#empty-btree-pgstatindex](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L44-L82) - expected empty B-tree output.
- [pgstattuple.out#wrong-index-am](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L126-L152) - expected wrong-index-AM errors.
- [pgstattuple.out#unsupported-relations](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L153-L213) and [pgstattuple.out#partition-and-sequence-failures](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L253-L299) - expected unsupported relation-kind errors.
- [pgstattuple.out#partition-index-success](../../../../raw/postgres-18/contrib/pgstattuple/expected/pgstattuple.out#L214-L274) - expected physical partition-index output.
- [pgstattuple.sgml](../../../../raw/postgres-18/doc/src/sgml/pgstattuple.sgml#L161-L613) - documentation for `pgstatindex` and `pgstattuple_approx` behavior.
- [visibilitymap.c#visibility-map-notes](../../../../raw/postgres-18/src/backend/access/heap/visibilitymap.c#L23-L55) - heap visibility-map semantics.
- [visibilitymapdefs.h#BITS_PER_HEAPBLOCK](../../../../raw/postgres-18/src/include/access/visibilitymapdefs.h#L16-L23) - two visibility-map bits per heap page.
- [freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-18/src/backend/storage/freespace/freespace.c#L36-L67) - FSM category representation.
- [freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-18/src/backend/storage/freespace/freespace.c#L239-L260) - recorded free-space read path.
- [indexfsm.c](../../../../raw/postgres-18/src/backend/storage/freespace/indexfsm.c#L1-L74) - index FSM free-vs-used page convention.
- [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-18/src/include/access/nbtree.h#L32-L120) - B-tree page opaque data, flags, and metapage fields.
- [nbtpage.c#_bt_allocbuf](../../../../raw/postgres-18/src/backend/access/nbtree/nbtpage.c#L868-L988) - B-tree page allocation and FSM candidate validation.
- [nbtree.c#btvacuumcleanup](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1097-L1140) - cleanup path that can skip a physical B-tree scan.
- [nbtree.c#btvacuumscan](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1173-L1346) - physical B-tree VACUUM scan.
- [nbtree.c#btvacuumpage](../../../../raw/postgres-18/src/backend/access/nbtree/nbtree.c#L1360-L1642) - per-page B-tree VACUUM handler and live-leaf cleanup lock.

## Open Questions

- Whether PostgreSQL would accept a new `pgstatindex_approx` API, and which
  estimates or confidence/error fields it should expose, is a design question
  not answered by the v18 source tree.
- The pinned source history shows why `pgstattuple_approx` is heap-oriented, but
  I found no same-checkout design discussion that explicitly rejects an
  approximate `pgstatindex` function.
