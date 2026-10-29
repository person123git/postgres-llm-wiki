---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Checking Every Table, Catalogs Included, for a Zeroed Last Segment in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Answer up front](#answer-up-front)
  - [What zeroing a file leaves behind](#what-zeroing-a-file-leaves-behind)
  - [Mental model: where the evidence lives](#mental-model-where-the-evidence-lives)
  - [Decision map](#decision-map)
  - [Why reading the last blocks cannot see a truncated segment](#why-reading-the-last-blocks-cannot-see-a-truncated-segment)
  - [The witnesses, step by step](#the-witnesses-step-by-step)
  - [Worked example: when the FSM and VM witnesses can fire](#worked-example-when-the-fsm-and-vm-witnesses-can-fire)
  - [Catalog tables](#catalog-tables)
  - [The tail and witness check](#the-tail-and-witness-check)
  - [Orphan files](#orphan-files)
  - [Index TID witness query](#index-tid-witness-query)
  - [Privileges, locks and cost](#privileges-locks-and-cost)
  - [Measured results](#measured-results)
  - [Branches, limits and false positives](#branches-limits-and-false-positives)
  - [State changes](#state-changes)
  - [Final causal summary](#final-causal-summary)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Last run](#last-run)
  - [Stages](#stages)
  - [Non-default settings in the measurement](#non-default-settings-in-the-measurement)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, how can I check every table in a database, including the catalog tables, for whether its last segment was zeroed out? Provide SQL that checks an arbitrary number of the last blocks of every table in the database.

## Answer

### Answer up front

**A fresh check of the last N blocks cannot detect a last segment that was truncated to zero bytes, because those blocks no longer belong to the measured relation.** When the last segment file is emptied, a new backend measures a shorter relation. The "last N blocks" are then the last N blocks of the previous, intact segment, and every one of them reads cleanly. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) The measured run confirms it: for all six truncated fixtures, the 32 tail pages the check read were valid. A backend with open segment state can observe a different failure path, as the [zeroed-file measurement](zeroed-relation-file.md#when-errors-do-appear) explains.

What *can* be checked is whether something in the cluster still remembers the relation being longer. PostgreSQL 12 keeps four such records outside the truncated file. The SQL below compares each of them with the current file length. It covers every relation with storage in the current database except temporary tables, with [catalogs](../../../glossary.md#catalog), [TOAST](../../../glossary.md#toast) tables and indexes included:

| Witness | What it remembers | Survives a later VACUUM? |
|---|---|---|
| [Free space map](../../../glossary.md#free-space-map) (`_fsm` fork) size | Free space was once recorded for a page past the current end | Yes, unless [VACUUM](../../../glossary.md#vacuum) truncates the relation again |
| [Visibility map](../../../glossary.md#visibility-map) (`_vm` fork) size | A page past the current end was once pinned in the map | Yes, with the same exception |
| [`pg_class.relpages`](../../../glossary.md#reltuples-and-relpages) | The page count VACUUM, ANALYZE or index build stored | Usually not: ANALYZE rewrites it, but VACUUM can preserve it when it counts no pages |
| Index entries ([TIDs](../../../glossary.md#tid)) | Heap pages the index still points to | Yes; plain VACUUM does not remove entries for pages it never scans |

The same query also reads the last `tail_blocks` pages of each relation straight from the files. That part is what catches the *other* meaning of "zeroed out": a file overwritten with zero bytes at its original length. It also checks the segment layout for damage that the storage manager's own rules forbid.

Two companion queries complete the check. The first lists data files that no visible `pg_class` row owns. It is the only way to notice that `pg_class` itself lost rows, because a query over `pg_class` cannot list relations whose rows are gone. The second, optional, uses [`pageinspect`](../../../glossary.md#pageinspect) to confirm a suspect table from its B-tree indexes.

No witness is a proof on its own, and each has a size limit below which it stays silent. The measured run flagged every injected truncation, with zero false positives on the healthy catalogs. Still, one case was left with only the index witness: a lost tail of 428 pages, too small for the FSM witness, followed by a VACUUM that rewrote `relpages`. Besides the index witness, it showed only an ambiguous layout finding that a legitimate truncation also produces. [Branches, limits and false positives](#branches-limits-and-false-positives) lists the cases.

This page builds on the [zeroed-file question](zeroed-relation-file.md), which measured what `SELECT`, `VACUUM` and `pg_dump` do after such damage, and on the [amcheck follow-up](../indexing/amcheck-zeroed-relation-segments.md), which shows that `amcheck` can pass on the same damage.

### What zeroing a file leaves behind

A relation's main [fork](../../../glossary.md#fork) is stored as numbered [segment files](../../../glossary.md#relation-segment) of `RELSEG_SIZE` blocks each: `<filenode>`, `<filenode>.1`, `<filenode>.2`, and so on. The default is 131,072 blocks, 1 GB at the default 8 kB block size. [md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L43-L66) [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L295) The [storage manager](../../../glossary.md#storage-manager) requires this layout:

1. Zero or more full segments of exactly `RELSEG_SIZE` blocks.
2. Exactly one partial segment, possibly empty.
3. Optionally, any number of inactive zero-length segments. They exist because `mdtruncate()` empties segments it no longer needs instead of deleting them. [md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66) [md.c#mdtruncate](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L807-L849)

"Zeroed out" covers two different operations, and they leave different evidence:

| Operation | Example | File afterwards | What the server sees |
|---|---|---|---|
| Truncate to zero length | `cat /dev/null > <file>` | Same name, 0 bytes | A fresh backend measures a shorter relation: `mdnblocks()` stops at the first short segment it measures, but trusts segments before the last one already open. An already-open later segment can therefore preserve a larger apparent extent. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) |
| Overwrite with zero bytes | `dd if=/dev/zero ... conv=notrunc` | Same length, all-zero pages | The same length, with [all-zero pages](../../../glossary.md#all-zero-page). `PageIsVerified()` accepts an all-zero page, so a [heap](../../../glossary.md#heap) scan can read it as empty. A [B-tree](../../../glossary.md#b-tree) reader that reaches one rejects it. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162) [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719) |

Truncating the *last* segment produces a layout the storage manager allows: full segments followed by one zero-length file. VACUUM produces exactly the same layout when it truncates a relation to an exact multiple of `RELSEG_SIZE`: `mdtruncate()` then empties segment K+1 and keeps it. [md.c#mdtruncate](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L819-L838) So the file layout alone cannot tell damage from a legitimate truncation; the measured `t_bound` control shows the identical layout from a real VACUUM.

### Mental model: where the evidence lives

| Value | Meaning | Source | Live or stored | Later use in the check |
|---|---|---|---|---|
| `cur_blocks` | Blocks a backend with fresh segment-open state would see | `stat()` of each segment file, combined the way `mdnblocks()` combines them from the first segment when none is already open | Live | The baseline every witness is compared with, and the range the tail read covers; a warm backend can use a larger bound [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) |
| `fsm_pages` | Pages in the `_fsm` fork | `stat()` of its consecutive segment files | Live file size; its contents describe the past | FSM witness |
| `vm_pages` | Pages in the `_vm` fork | `stat()` of its consecutive segment files | Live file size; its contents describe the past | VM witness |
| `relpages` | Page count stored by VACUUM, ANALYZE or index build | `pg_class` row | Stored; a VACUUM that counts no pages can preserve it | `relpages` witness |
| Index heap TIDs | Heap block numbers the index entries point to | B-tree leaf pages, through `pageinspect` | Stored in the index | Index witness (third query) |
| Tail pages | The last `tail_blocks` pages of the current extent | `pg_read_binary_file()` on the segment files | Live, from the files rather than from [shared buffers](../../../glossary.md#buffer-manager) | All-zero and invalid-header counts |

The witnesses work because truncating a file outside PostgreSQL changes only that one file. PostgreSQL's own truncation, `RelationTruncate()`, shrinks the FSM, then the VM, then the main fork, together. [storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L230-L295) An outside truncation initially leaves the side forks, the `pg_class` row and the indexes describing the old, longer relation. The SQL observes these files at different times, so concurrent extension can also create a mismatch; recheck a finding while writes are paused before interpreting it as loss. [freespace.c#fsm_extend](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L615-L661)

### Decision map

```text
For each relation with storage (pg_class seq scan)
  |
  |-- stat <path>, <path>.1, ... until a file is missing            [A]
  |     |-- first file missing ............................ "first segment missing"
  |     |-- a short segment followed by non-empty data .... "data after a short segment"
  |     |-- segment larger than RELSEG_SIZE ............... "segment longer than RELSEG_SIZE"
  |     |-- full segments then one zero-length file ....... "zero-length segment after full segments" (ambiguous)
  |     `-- cur_blocks = blocks up to the first short segment
  |
  |-- stat <path>_fsm[.N]: a leaf page for blocks >= cur_blocks exists?       [B]
  |     `-- yes ........................................... "FSM covers blocks past end"
  |-- stat <path>_vm[.N]: more map pages than cur_blocks needs?               [C]
  |     `-- yes ........................................... "VM covers blocks past end"
  |-- pg_class.relpages > cur_blocks?                                           [D]
  |     `-- yes ........................................... "relpages > current blocks" (stored-stat)
  |
  `-- read pages max(0, cur_blocks - tail_blocks) .. cur_blocks - 1 from disk   [E]
        |-- all bytes zero ................................ "all-zero pages in tail" (ambiguous)
        |-- header fails PageIsVerified's sanity test ..... "invalid page header in tail"
        `-- fewer than BLCKSZ bytes ....................... "short read in tail"

Separately: data files no visible pg_class row owns ........ orphan-file query   [F]
Separately: B-tree leaf TIDs >= heap blocks .................. index TID witness  [G]
```

- **[A]** [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755), [md.c#_mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1229-L1241), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L307)
- **[B]** [fsm_internals.h#SlotsPerFSMPage](../../../../raw/postgres-12/src/include/storage/fsm_internals.h#L51-L61), [freespace.c#fsm_logical_to_physical](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L446-L476), [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L545-L607)
- **[C]** [visibilitymap.c#HEAPBLOCKS_PER_PAGE](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L101-L115), [visibilitymap.c#visibilitymap_pin](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L189-L202)
- **[D]** [pg_class.h#relpages](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L60), [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1182-L1184)
- **[E]** [genfile.c#read_binary_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L111-L177), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162)
- **[F]** [dbsize.c#pg_relation_filenode](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L865-L904), [genfile.c#pg_ls_dir](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L447-L510)
- **[G]** [btreefuncs.c#bt_page_items](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L311-L400)

### Why reading the last blocks cannot see a truncated segment

1. **Every size-bounded read starts from `mdnblocks()`.** It starts at the last already-open segment, trusting earlier ones as full, and returns `segno * RELSEG_SIZE + nblocks` at the first short segment it measures. With fresh segment-open state, an emptied last segment therefore moves the end of the relation back to the end of the previous segment; a warm backend's bound may differ. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755)
2. **A probe by `ctid` is filtered against that end.** A TID scan keeps a TID only if its block is below the scan's `rs_nblocks`, so `WHERE ctid = '(4000,1)'` on a lost page returns no row and raises no error. The measured `t_last` probe returned 1 row before the damage and 0 after it, with a `Tid Scan` plan. [heapam_handler.c#heapam_tuple_tid_valid](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L209-L215) [nodeTidscan.c:178-182](../../../../raw/postgres-12/src/backend/executor/nodeTidscan.c#L178-L182)
3. **A bitmap heap scan also drops pages past the end.** `heapam_scan_bitmap_next_block()` ignores any page at or past `rs_nblocks`, so a bitmap scan returns fewer rows instead of failing. [heapam_handler.c:2192-2199](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2192-L2199)
4. **Reading the files directly does not help either.** `pg_read_binary_file()` at an offset past the end of a file returns an empty value, because `fread()` simply reads nothing. [genfile.c#read_binary_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L157-L172) The last pages that do exist belong to the previous, intact segment.

So the check reads the tail pages for the overwrite case and for page-header damage, but it finds truncation by comparing the file length with what the side forks, `relpages` and the indexes remember.

### The witnesses, step by step

#### Free space map witness

The FSM stores one byte per heap page, in a tree of FSM pages. With the default 8 kB pages, one leaf FSM page holds `SlotsPerFSMPage` = 4,069 slots, so leaf page `L` describes heap pages `L × 4069` to `L × 4069 + 4068`. The tree has three levels, and `fsm_logical_to_physical()` places leaf `L` at FSM block `L + ⌊L/4069⌋ + ⌊L/4069²⌋ + 2`. A non-default 1 or 2 kB block size gives fewer than 1,626 slots, so the source uses four levels; the query reads block size from `pg_control_init()` and adds the fourth-level term when required. [configure.in#blocksize](../../../../raw/postgres-12/configure.in#L247-L265) [fsm_internals.h#SlotsPerFSMPage](../../../../raw/postgres-12/src/include/storage/fsm_internals.h#L51-L61) [freespace.c#FSM_TREE_DEPTH](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L67-L77) [freespace.c#fsm_get_location](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L481-L492) [freespace.c#fsm_logical_to_physical](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L446-L476)

Recording a heap page's free space writes its slot. If that slot's FSM page does not exist yet, `fsm_readbuf(..., extend = true)` extends the fork up to it first. [freespace.c#RecordPageWithFreeSpace](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L181-L191) [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L545-L607) [freespace.c#fsm_extend](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L615-L661) Three ordinary paths write slots:

- VACUUM records every page it scans, either directly or after it removes the page's dead tuples. [vacuumlazy.c:1380-1388](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1380-L1388) [vacuumlazy.c:1558-1563](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1558-L1563)
- An insert that finds its target page full records that page before moving on. [hio.c:526-537](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L526-L537)
- Bulk extension records each new page. [hio.c#RelationAddExtraBlocks](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L186-L261)

So the fork's length says the relation once had a page in the last leaf that exists. PostgreSQL's own truncation keeps the two consistent: `FreeSpaceMapTruncateRel()` cuts the fork back to the leaf holding the new end. [freespace.c#FreeSpaceMapTruncateRel](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L259-L340)

The check therefore asks one question: **does the fork contain a leaf whose first heap page is at or past `cur_blocks`?** At 8 kB, that leaf is `l0 = ⌈cur_blocks / 4069⌉`. It exists exactly when `fsm_pages > l0 + ⌊l0/4069⌋ + ⌊l0/4069²⌋ + 2`. The query generalizes this position for four-level trees and sums every consecutive FSM segment. A positive result means the fork has space to record a page at or beyond the measured main-fork end. In a quiet cluster it supports a shrinkage inference; concurrent extension or crash-related fork state can also produce a mismatch. [freespace.c#fsm_logical_to_physical](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L446-L476) [freespace.c#fsm_extend](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L615-L661) The query also finds the last existing leaf `lmax` and reports `lmax × 4069 + 1` as a lower bound on the size the fork can describe.

#### Visibility map witness

The VM uses 2 bits per heap page, so with 8 kB pages one map page covers `HEAPBLOCKS_PER_PAGE` = (8192 − 24) × 4 = 32,672 heap pages. `visibilitymap_pin()` extends the map up to the page covering the heap page it is given. VACUUM pins pages it scans. [visibilitymap.c#HEAPBLOCKS_PER_PAGE](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L101-L115) [visibilitymap.c#visibilitymap_pin](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L189-L202) [vacuumlazy.c:816](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L816) `visibilitymap_truncate()` cuts the map back to `⌈new_size / 32672⌉` pages. [visibilitymap.c#visibilitymap_truncate](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L442-L545) So `vm_pages > ⌈cur_blocks / 32672⌉` means the fork can cover pages beyond the measured main-fork end. In a quiet cluster, `(vm_pages − 1) × 32672 + 1` is a lower bound on the size it reached; concurrent changes need rechecking.

#### `relpages` witness

VACUUM and ANALYZE can store the relation's page count in `pg_class.relpages`; index builds store it too. [vacuumlazy.c#stats-gate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L320-L356) [analyze.c:591-605](../../../../raw/postgres-12/src/backend/commands/analyze.c#L591-L605) [index.c:2978-2982](../../../../raw/postgres-12/src/backend/catalog/index.c#L2978-L2982) A stored value larger than `cur_blocks` means the statistic and current file length disagree. External shrinkage is one cause. A `TRUNCATE` of a table that has no indexes and was created or rewritten in the current [subtransaction](../../../glossary.md#subtransaction), described below, can also leave a stale value without corruption. [tablecmds.c:1776-1798](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1776-L1798) [heap.c#heap_truncate_one_rel](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3260-L3288) With indexes, the same truncation rebuilds them, and `index_update_stats()` rewrites the heap's `relpages` from its current length. [heap.c:3202](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3202) [index.c:2763](../../../../raw/postgres-12/src/backend/catalog/index.c#L2763) The value is stored as a signed `int32` copy of an unsigned block count, so the query adds 2³² back to a negative value. [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1182-L1184) [pg_class.h#relpages](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L60)

This witness is the most fragile one. ANALYZE of the damaged relation measures the shorter file and overwrites `relpages`. VACUUM usually does too, but if it counts no pages of a nonempty relation, it preserves the old value. [analyze.c:591-605](../../../../raw/postgres-12/src/backend/commands/analyze.c#L591-L605) [vacuumlazy.c#stats-gate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L320-L356) An [autovacuum](../../../glossary.md#autovacuum) run follows the same VACUUM rule. The measured `revacuum` stage counted pages and shows `t_lastvac` going from 4,090 to 3,072.

#### Index TID witness

An index is a separate file. Truncating the heap leaves every index entry that points at a lost page in place. A plain VACUUM removes only entries for dead tuples it found in heap pages it scanned, and it never scans pages past the current end. [vacuumlazy.c#lazy_scan_heap-nblocks](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L545-L556) The third query reads every B-tree leaf page through `pageinspect`, takes the largest heap block number any entry points to, and compares it with the heap's current logical extent. Its recursive segment walk stops at the first short heap segment, as `mdnblocks()` does; [`pg_relation_size()`](../../../glossary.md#relation-size-functions) instead adds file sizes even after that segment. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L307) In v12 the first item of a non-rightmost leaf is its [high key](../../../glossary.md#high-key), which is not a heap pointer, so the query skips it. [nbtree.h#P_FIRSTDATAKEY](../../../../raw/postgres-12/src/include/access/nbtree.h#L217-L219) This witness needs no FSM, no VM and no stored statistic. It does read the whole index, and it needs superuser. [btreefuncs.c#bt_page_items](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L311-L400) [btreefuncs.c#superuser-check](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L320-L323)

### Worked example: when the FSM and VM witnesses can fire

The FSM witness can fire once the old size reaches the first page of a leaf that starts beyond the new end. The VM witness can fire once the old size reaches the first page covered by an additional map page. Neither requires losing the entire range. Their boundaries differ, so one fork can witness a loss that the other misses. [freespace.c#fsm_logical_to_physical](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L446-L476) [visibilitymap.c#HEAPBLOCKS_PER_PAGE](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L101-L115) Both thresholds follow from the formulas above:

| Case | `cur_blocks` | Old size | FSM: `l0`, leaf position, `fsm_pages` | FSM fires? | VM: needed, `vm_pages` | VM fires? |
|---|---|---|---|---|---|---|
| `t_last` (measured) | 3,072 | 4,090 | `l0 = ⌈3072/4069⌉ = 1`, position 3, 4 pages | Yes (4 > 3); bound `1 × 4069 + 1 = 4,070` | 1, 1 | No |
| `t_small_last` (measured) | 3,072 | 3,500 | `l0 = 1`, position 3, 3 pages (old size < 4,069) | No | 1, 1 | No |
| `t_vm` (measured) | 31,744 | 32,760 | `l0 = ⌈31744/4069⌉ = 8`, position 10, 11 pages | Yes; bound `8 × 4069 + 1 = 32,553` | 1, 2 | Yes; bound `32,673` |
| Default 1 GB segments, one full segment survives | 131,072 | > 134,277 | `l0 = 33` covers pages 134,277 and up | Only if more than 3,205 pages (≈ 25 MB) were lost | 5 | Only if more than 32,288 pages (≈ 252 MB) were lost |
| Single-file table emptied | 0 | any | `l0 = 0`, position 2 | Yes if the FSM has its first leaf, that is, ≥ 3 pages | 0 | Yes if any VM page exists |

The `t_*` inputs come from the measurement's `fixtures.txt` and `check.txt`. The default-segment rows are derived from the same formulas, not measured. In general the FSM witness misses at most 4,068 lost pages (just under 32 MB). The VM witness misses at most 32,671 (just under 255 MB). A smaller loss can still show up in `relpages` or in the index witness.

### Catalog tables

The main query covers every catalog of the current database: `pg_class` lists them like any other relation. Three catalog details matter:

- **Mapped catalogs.** `pg_class` itself and other nailed or shared catalogs have [`relfilenode`](../../../glossary.md#relfilenumber) `= 0`; the [relation mapper](../../../glossary.md#relation-mapper) holds their file numbers. The query therefore takes paths from `pg_relation_filepath()`, which asks the mapper, never from `relfilenode`. [relmapper.c:1-28](../../../../raw/postgres-12/src/backend/utils/cache/relmapper.c#L1-L28) [dbsize.c#pg_relation_filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1022)
- **Shared catalogs.** `pg_database`, `pg_authid` and the other shared catalogs live under `global/` and appear in every database's `pg_class`. Running the query in any one database covers them. Per-database catalogs need one run per database, because `pg_class` lists only the current database's relations. [relpath.c:140-205](../../../../raw/postgres-12/src/common/relpath.c#L140-L205)
- **Damage to `pg_class` itself.** The main query reads `pg_class` from its heap; it filters on `relkind`, which no `pg_class` index covers. If `pg_class` lost its last segment, the relations whose rows were there simply do not appear, so the query cannot check them. It can still flag `pg_class` itself when its own row survived. In the measured catalog leg that row was outside the emptied segment, so it was still visible with the old `relpages` value of 1,100, and the query flagged `pg_class`. If the row itself is lost, `pg_class` drops out of the main query too. The orphan-file query covers the lost relations that have storage from the other side: it lists data files whose number no visible `pg_class` row accounts for. [dbsize.c#pg_relation_filenode](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L865-L904) Only tables, indexes, sequences, TOAST tables and materialized views get a data file, so a lost row of a view, composite type, foreign table, or partitioned table or index leaves no file, and none of the three queries detects it. [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L187-L192) [heap.c:374-378](../../../../raw/postgres-12/src/backend/catalog/heap.c#L374-L378)

The catalog leg showed one more trap. After `pg_class` lost 3,376 rows, `SELECT count(*) FROM pg_class WHERE relname ~ '^c_[0-9]+$'` still returned all 49,006. Its plan was an [index-only scan](../../../glossary.md#index-only-scan) with `Heap Fetches: 0`. Because `pg_class` had been vacuumed, the VM marked its pages all-visible, so the executor answered from the index without visiting the heap. [nodeIndexonlyscan.c:161-170](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L170) A forced [sequential scan](../../../glossary.md#sequential-scan) count returned 45,630, which is 49,006 − 3,376. A catalog query whose result "looks complete" is therefore not evidence that the catalog is intact.

### The tail and witness check

Run it in each database you want to check. Pass `tail_blocks` to read more or fewer trailing pages, and `show_all=true` to list healthy relations too:

```sh
psql -X -d mydb -v tail_blocks=64 -f relation_tail_check.sql
```

The `segs` part is a [recursive CTE](../../../glossary.md#recursive-cte) that stats main segment files until one is missing, the same loop `calculate_relation_size()` runs. `fork_segs` follows the same naming rule for `_fsm` and `_vm`, whose forks can also span segments; it stops at their first short segment so the counted pages match their logical extent. [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L307) [storage.sgml#segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L233-L243) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) `cur_blocks` repeats the `mdnblocks()` rule for the main fork.

The page decoding follows `PageHeaderData`: `pd_flags` at byte 10, `pd_lower` at 12, `pd_upper` at 14, `pd_special` at 16, `pd_pagesize_version` at 18. It tries both byte orders, applies `PageIsVerified()`'s header sanity conditions without a checksum comparison, and separately reports an unexpected page layout version. [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164) [bufpage.h#PD_VALID_FLAG_BITS](../../../../raw/postgres-12/src/include/storage/bufpage.h#L180-L199) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162) Block size, segment size and maximum alignment come from the control file through `pg_control_init()`. [pg_controldata.c#pg_control_init](../../../../raw/postgres-12/src/backend/utils/misc/pg_controldata.c#L265-L350)

```sql
-- PostgreSQL 12. Run as a superuser, or as a role granted EXECUTE on
-- pg_stat_file(text, boolean) and pg_read_binary_file(text, bigint, bigint, boolean).
-- psql variables: tail_blocks (default 16) and show_all (default false).
\if :{?tail_blocks}
\else
\set tail_blocks 16
\endif
\if :{?show_all}
\else
\set show_all false
\endif
SET /* wiki_relation_tail_check */ statement_timeout = '30min';
SET /* wiki_relation_tail_check */ lock_timeout = '5s';
WITH /* wiki_relation_tail_check */ RECURSIVE
geom AS (
  -- Cluster geometry from the control file.  fsm_slots is SlotsPerFSMPage and
  -- vm_blocks is HEAPBLOCKS_PER_PAGE; both subtract the 24-byte page header.
  SELECT (:tail_blocks)::bigint AS tail_blocks,
         ci.database_block_size::bigint AS blcksz,
         ci.blocks_per_segment::bigint AS relseg,
         ci.max_data_alignment::bigint AS maxalign,
         ((ci.database_block_size - 24 - 4) - (ci.database_block_size / 2 - 1))::bigint AS fsm_slots,
         CASE WHEN ((ci.database_block_size - 24 - 4) - (ci.database_block_size / 2 - 1)) >= 1626
              THEN 3 ELSE 4 END AS fsm_levels,
         ((ci.database_block_size - 24) * 4)::bigint AS vm_blocks,
         decode(repeat('00', ci.database_block_size), 'hex') AS zero_page
  FROM pg_control_init() AS ci
),
rels AS (
  -- Every relation with storage, shared and mapped catalogs included.
  -- Temporary tables live in their owner's local buffers, so they are skipped.
  SELECT c.oid, n.nspname, c.relname, c.relkind,
         CASE WHEN c.relpages < 0 THEN c.relpages + 4294967296
              ELSE c.relpages END::bigint AS relpages,
         pg_relation_filepath(c.oid) AS path
  FROM pg_class AS c
  JOIN pg_namespace AS n ON n.oid = c.relnamespace
  WHERE c.relkind IN ('r', 't', 'm', 'i', 'S')
    AND c.relpersistence <> 't'
),
segs (oid, path, segno, bytes) AS (
  -- Stat main-fork segment files until the first missing one, as
  -- calculate_relation_size() does.  The last row per relation has NULL bytes.
  SELECT r.oid, r.path, 0::bigint, (pg_stat_file(r.path, true)).size
  FROM rels AS r
  WHERE r.path IS NOT NULL
  UNION ALL
  SELECT s.oid, s.path, s.segno + 1,
         (pg_stat_file(s.path || '.' || (s.segno + 1), true)).size
  FROM segs AS s
  WHERE s.bytes IS NOT NULL
),
first_short AS (
  SELECT s.oid,
         count(s.bytes) AS seg_files,
         min(s.segno) FILTER (WHERE s.bytes < g.relseg * g.blcksz) AS short_seg,
         coalesce(bool_or(s.bytes > g.relseg * g.blcksz), false) AS oversized
  FROM segs AS s
  CROSS JOIN geom AS g
  GROUP BY s.oid
),
layout AS (
  SELECT f.oid, f.seg_files, f.short_seg, f.oversized,
         max(s.bytes) FILTER (WHERE s.segno = f.short_seg) AS short_bytes,
         coalesce(sum(s.bytes) FILTER (WHERE s.segno > f.short_seg), 0) AS bytes_after_short
  FROM first_short AS f
  JOIN segs AS s ON s.oid = f.oid
  GROUP BY f.oid, f.seg_files, f.short_seg, f.oversized
),
fork_segs (oid, path, fork, segno, bytes) AS (
  -- FSM and VM forks follow the same segment naming rule as the main fork.
  SELECT r.oid, r.path, f.fork, 0::bigint,
         (pg_stat_file(r.path || '_' || f.fork, true)).size
  FROM rels AS r
  CROSS JOIN (VALUES ('fsm'), ('vm')) AS f(fork)
  WHERE r.path IS NOT NULL
  UNION ALL
  SELECT s.oid, s.path, s.fork, s.segno + 1,
         (pg_stat_file(s.path || '_' || s.fork || '.' || (s.segno + 1), true)).size
  FROM fork_segs AS s
  CROSS JOIN geom AS g
  WHERE s.bytes = g.relseg * g.blcksz
),
fork_sizes AS (
  SELECT s.oid,
         (sum(s.bytes) FILTER (WHERE s.fork = 'fsm'))::bigint / g.blcksz AS fsm_pages,
         (sum(s.bytes) FILTER (WHERE s.fork = 'vm'))::bigint / g.blcksz AS vm_pages
  FROM fork_segs AS s
  CROSS JOIN geom AS g
  GROUP BY s.oid, g.blcksz
),
sized AS (
  -- cur_blocks is what mdnblocks() returns to a backend that opens the
  -- relation fresh: it stops at the first segment shorter than RELSEG_SIZE.
  SELECT r.oid, r.nspname, r.relname, r.relkind, r.relpages, r.path,
         l.seg_files, l.short_seg, l.short_bytes, l.bytes_after_short, l.oversized,
         CASE WHEN l.seg_files = 0 THEN 0
              WHEN l.short_seg IS NULL THEN l.seg_files * g.relseg
              ELSE l.short_seg * g.relseg + l.short_bytes / g.blcksz
         END AS cur_blocks,
         fs.fsm_pages, fs.vm_pages
  FROM rels AS r
  JOIN layout AS l ON l.oid = r.oid
  JOIN fork_sizes AS fs ON fs.oid = r.oid
  CROSS JOIN geom AS g
),
witness AS (
  -- FSM: the leaf page l0 covers only blocks at or past cur_blocks; its
  -- physical position follows fsm_logical_to_physical() for the configured tree depth.
  -- VM: one map page covers vm_blocks heap blocks.
  SELECT z.*,
         coalesce(z.fsm_pages > lz.l0 + lz.l0 / g.fsm_slots
                                + lz.l0 / (g.fsm_slots * g.fsm_slots)
                                + CASE WHEN g.fsm_levels = 4
                                       THEN lz.l0 / (g.fsm_slots * g.fsm_slots * g.fsm_slots)
                                       ELSE 0 END + g.fsm_levels - 1, false) AS fsm_past_end,
         coalesce(z.vm_pages > (z.cur_blocks + g.vm_blocks - 1) / g.vm_blocks, false) AS vm_past_end,
         fl.lmax
  FROM sized AS z
  CROSS JOIN geom AS g
  CROSS JOIN LATERAL (SELECT (z.cur_blocks + g.fsm_slots - 1) / g.fsm_slots AS l0) AS lz
  LEFT JOIN LATERAL (
    SELECT max(x.l) AS lmax
    -- The upper-level overhead is below P/(slots-1); using P/slots alone
    -- can start above the final leaf when a fork is very large.
    FROM generate_series(greatest(0, z.fsm_pages - g.fsm_levels - 2
                                     - z.fsm_pages / (g.fsm_slots - 1)),
                         greatest(0, z.fsm_pages - g.fsm_levels)) AS x(l)
    WHERE x.l + x.l / g.fsm_slots + x.l / (g.fsm_slots * g.fsm_slots)
          + CASE WHEN g.fsm_levels = 4
                 THEN x.l / (g.fsm_slots * g.fsm_slots * g.fsm_slots)
                 ELSE 0 END + g.fsm_levels - 1 <= z.fsm_pages - 1
  ) AS fl ON true
),
tail AS (
  -- Read the last tail_blocks blocks from the files, not from shared buffers.
  SELECT w.oid, b.blk,
         pg_read_binary_file(
           w.path || CASE WHEN b.blk >= g.relseg THEN '.' || (b.blk / g.relseg) ELSE '' END,
           (b.blk % g.relseg) * g.blcksz, g.blcksz, true) AS page
  FROM witness AS w
  CROSS JOIN geom AS g
  CROSS JOIN LATERAL generate_series(greatest(0, w.cur_blocks - g.tail_blocks),
                                     w.cur_blocks - 1) AS b(blk)
),
pages AS (
  -- Header offsets from PageHeaderData. Try both server byte orders; the
  -- validity test repeats PageIsVerified() without its checksum comparison.
  SELECT t.oid, t.blk,
         CASE
           WHEN coalesce(length(t.page), 0) < g.blcksz THEN 'short'
           WHEN t.page = g.zero_page THEN 'zero'
           WHEN (hc.le_sane AND h.le_version = g.blcksz + 4)
             OR (hc.be_sane AND h.be_version = g.blcksz + 4) THEN 'valid'
           WHEN hc.le_sane OR hc.be_sane THEN 'odd-version'
           ELSE 'invalid'
         END AS state
  FROM tail AS t
  CROSS JOIN geom AS g
  LEFT JOIN LATERAL (
    SELECT get_byte(t.page, 10) + 256 * get_byte(t.page, 11) AS le_flags,
           get_byte(t.page, 12) + 256 * get_byte(t.page, 13) AS le_lower,
           get_byte(t.page, 14) + 256 * get_byte(t.page, 15) AS le_upper,
           get_byte(t.page, 16) + 256 * get_byte(t.page, 17) AS le_special,
           get_byte(t.page, 18) + 256 * get_byte(t.page, 19) AS le_version,
           256 * get_byte(t.page, 10) + get_byte(t.page, 11) AS be_flags,
           256 * get_byte(t.page, 12) + get_byte(t.page, 13) AS be_lower,
           256 * get_byte(t.page, 14) + get_byte(t.page, 15) AS be_upper,
           256 * get_byte(t.page, 16) + get_byte(t.page, 17) AS be_special,
           256 * get_byte(t.page, 18) + get_byte(t.page, 19) AS be_version
    WHERE length(t.page) >= 24
  ) AS h ON true
  LEFT JOIN LATERAL (
    SELECT h.le_upper <> 0 AND (h.le_flags & ~7) = 0
           AND h.le_lower <= h.le_upper AND h.le_upper <= h.le_special
           AND h.le_special <= g.blcksz AND h.le_special % g.maxalign = 0 AS le_sane,
           h.be_upper <> 0 AND (h.be_flags & ~7) = 0
           AND h.be_lower <= h.be_upper AND h.be_upper <= h.be_special
           AND h.be_special <= g.blcksz AND h.be_special % g.maxalign = 0 AS be_sane
  ) AS hc ON true
),
tail_summary AS (
  SELECT p.oid,
         count(*) AS tail_read,
         count(*) FILTER (WHERE p.state = 'zero') AS tail_zero,
         count(*) FILTER (WHERE p.state = 'invalid') AS tail_invalid,
         count(*) FILTER (WHERE p.state = 'short') AS tail_short,
         count(*) FILTER (WHERE p.state = 'odd-version') AS tail_odd,
         max(p.blk) FILTER (WHERE p.state <> 'zero') AS last_nonzero_blk
  FROM pages AS p
  GROUP BY p.oid
),
flagged AS (
  SELECT w.*, ts.tail_read, ts.tail_zero, ts.last_nonzero_blk,
         coalesce(w.seg_files = 0, false) AS f_missing,
         coalesce(w.bytes_after_short > 0, false) AS f_after_short,
         w.oversized AS f_oversized,
         w.fsm_past_end AS f_fsm,
         w.vm_past_end AS f_vm,
         coalesce(ts.tail_invalid > 0, false) AS f_invalid,
         coalesce(ts.tail_short > 0, false) AS f_short,
         coalesce(w.relpages > w.cur_blocks, false) AS f_relpages,
         coalesce(w.short_seg > 0 AND w.short_bytes = 0, false) AS f_empty_seg,
         coalesce(w.short_bytes % g.blcksz <> 0, false) AS f_partial,
         coalesce(ts.tail_zero > 0, false) AS f_zero,
         coalesce(ts.tail_odd > 0, false) AS f_odd,
         CASE WHEN w.fsm_past_end THEN w.lmax * g.fsm_slots + 1 END AS fsm_min_blocks,
         CASE WHEN w.vm_past_end THEN (w.vm_pages - 1) * g.vm_blocks + 1 END AS vm_min_blocks
  FROM witness AS w
  CROSS JOIN geom AS g
  LEFT JOIN tail_summary AS ts ON ts.oid = w.oid
)
SELECT format('%I.%I', f.nspname, f.relname) AS relation,
       f.relkind,
       f.cur_blocks,
       f.relpages,
       greatest(f.cur_blocks, f.relpages, f.fsm_min_blocks, f.vm_min_blocks) AS min_expected_blocks,
       f.seg_files,
       f.fsm_pages,
       f.vm_pages,
       coalesce(f.tail_read, 0) AS tail_read,
       coalesce(f.tail_zero, 0) AS tail_zero,
       CASE WHEN f.tail_read IS NULL THEN 0
            ELSE f.cur_blocks - 1 - coalesce(f.last_nonzero_blk, f.cur_blocks - 1 - f.tail_read)
       END AS trailing_zero_run,
       CASE WHEN f.f_missing OR f.f_after_short OR f.f_oversized OR f.f_fsm OR f.f_vm
                 OR f.f_invalid OR f.f_short THEN 'strong'
            WHEN f.f_relpages THEN 'stored-stat'
            WHEN f.f_empty_seg OR f.f_partial OR f.f_zero OR f.f_odd THEN 'ambiguous'
       END AS evidence,
       array_remove(ARRAY[
         CASE WHEN f.f_missing THEN 'first segment missing' END,
         CASE WHEN f.f_after_short THEN 'data after a short segment' END,
         CASE WHEN f.f_oversized THEN 'segment longer than RELSEG_SIZE' END,
         CASE WHEN f.f_fsm THEN 'FSM covers blocks past end' END,
         CASE WHEN f.f_vm THEN 'VM covers blocks past end' END,
         CASE WHEN f.f_invalid THEN 'invalid page header in tail' END,
         CASE WHEN f.f_short THEN 'short read in tail' END,
         CASE WHEN f.f_relpages THEN 'relpages > current blocks' END,
         CASE WHEN f.f_empty_seg THEN 'zero-length segment after full segments' END,
         CASE WHEN f.f_partial THEN 'partial block at segment end' END,
         CASE WHEN f.f_zero THEN 'all-zero pages in tail' END,
         CASE WHEN f.f_odd THEN 'unexpected page layout version in tail' END
       ], NULL) AS findings
FROM flagged AS f
WHERE f.f_missing OR f.f_after_short OR f.f_oversized OR f.f_fsm OR f.f_vm
   OR f.f_invalid OR f.f_short OR f.f_relpages OR f.f_empty_seg OR f.f_partial
   OR f.f_zero OR f.f_odd OR :show_all
ORDER BY CASE WHEN f.f_missing OR f.f_after_short OR f.f_oversized OR f.f_fsm OR f.f_vm
                   OR f.f_invalid OR f.f_short THEN 1
              WHEN f.f_relpages THEN 2
              ELSE 3 END,
         relation;
```

How to read a result row:

| Column | Meaning |
|---|---|
| `cur_blocks` | Blocks a freshly opened relation has now. |
| `relpages`, `fsm_pages`, `vm_pages` | The stored statistic and the two side-fork lengths, in pages. `vm_pages` is empty for relations without a VM, such as indexes. |
| `min_expected_blocks` | The largest of `cur_blocks`, stored `relpages` and the two fork coverage bounds. It is an evidence-based size bound when those values came from the same stable relation state; concurrent changes and stale statistics require checking. |
| `tail_read`, `tail_zero`, `trailing_zero_run` | Tail pages read, how many were all-zero, and how many all-zero pages end the relation. |
| `evidence` | `strong`: a forbidden layout, a fork coverage mismatch, or a bad page. Recheck fork mismatches without concurrent writes. `stored-stat`: only `relpages` disagrees. `ambiguous`: only states that also arise normally. |
| `findings` | The individual findings, strongest first. |

### Orphan files

This query lists every first-segment main-fork file in the current database's directories, plus `global/`, that no visible `pg_class` row accounts for. Run it in each database. [relpath.c:140-205](../../../../raw/postgres-12/src/common/relpath.c#L140-L205) [relpath.h#TABLESPACE_VERSION_DIRECTORY](../../../../raw/postgres-12/src/include/common/relpath.h#L26-L27) [pg_tablespace.dat:15-20](../../../../raw/postgres-12/src/include/catalog/pg_tablespace.dat#L15-L20) It compares file numbers from `pg_relation_filenode()`, which consults the relation mapper for mapped catalogs. It deliberately avoids `pg_filenode_relation()`: that function looks the file number up through a `pg_class` index, and on a damaged `pg_class` the lookup could follow an index entry into the lost pages. [dbsize.c#pg_filenode_relation](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L906-L934) [relfilenodemap.c:200-210](../../../../raw/postgres-12/src/backend/utils/cache/relfilenodemap.c#L200-L210)

```sql
-- PostgreSQL 12. Run as a superuser, or as a role granted EXECUTE on
-- pg_ls_dir(text, boolean, boolean) and pg_stat_file(text, boolean).
SET /* wiki_orphan_relation_files */ statement_timeout = '10min';
SET /* wiki_orphan_relation_files */ lock_timeout = '5s';
WITH /* wiki_orphan_relation_files */ db AS (
  SELECT d.oid AS dboid, d.dattablespace
  FROM pg_database AS d
  WHERE d.datname = current_database()
),
dirs AS (
  -- 1663 is pg_default and 1664 is pg_global; other tablespaces use the
  -- TABLESPACE_VERSION_DIRECTORY "PG_<major>_<catalog version>".
  SELECT t.oid AS spc,
         CASE
           WHEN t.oid = 1664 THEN 'global'
           WHEN t.oid = 1663 THEN 'base/' || db.dboid
           ELSE 'pg_tblspc/' || t.oid || '/PG_'
                || current_setting('server_version_num')::int / 10000 || '_'
                || (SELECT cs.catalog_version_no FROM pg_control_system() AS cs)
                || '/' || db.dboid
         END AS dir
  FROM pg_tablespace AS t
  CROSS JOIN db
),
known AS (
  -- File numbers the visible pg_class rows account for, mapped catalogs included.
  SELECT CASE WHEN c.reltablespace = 0 THEN db.dattablespace
              ELSE c.reltablespace END AS spc,
         pg_relation_filenode(c.oid) AS filenode
  FROM pg_class AS c
  CROSS JOIN db
  WHERE c.relkind IN ('r', 't', 'm', 'i', 'S')
    AND c.relpersistence <> 't'
),
files AS (
  -- First main-fork segment files only: an all-digit name.
  SELECT d.spc, d.dir || '/' || f.name AS path, f.name::oid AS filenode
  FROM dirs AS d
  CROSS JOIN LATERAL pg_ls_dir(d.dir, true, false) AS f(name)
  WHERE f.name ~ '^[0-9]+$'
)
SELECT f.path,
       (pg_stat_file(f.path, true)).size AS first_segment_bytes,
       (pg_stat_file(f.path, true)).modification AS modified
FROM files AS f
WHERE NOT EXISTS (SELECT 1 FROM known AS k
                  WHERE k.spc = f.spc AND k.filenode = f.filenode)
ORDER BY first_segment_bytes DESC NULLS LAST, f.path;
```

An orphan is not always damage; see [Branches, limits and false positives](#branches-limits-and-false-positives). The file size alone does not tell the causes apart, because three different states leave a zero-length orphan:

| Cause of a zero-length orphan | Why the file is empty | How it changes |
|---|---|---|
| A dropped relation | `mdunlink()` truncates the first segment to zero and asks for it to be unlinked after the next [checkpoint](../../../glossary.md#checkpoint). [md.c#mdunlink](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L215-L262) | It disappears after the next checkpoint. |
| A `CREATE` still in progress, or one that crashed | `RelationCreateStorage()` creates the empty file before the `pg_class` row commits; the delete-at-abort request lives only in backend memory. [storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L79-L122) | It stops being an orphan when the transaction commits, because its row becomes visible, and is deleted if the transaction aborts; a crash leftover stays. |
| An empty relation whose `pg_class` row was lost | A relation that never held a page has a zero-length file. [storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L79-L122) | It stays. This is catalog damage. |

So run `CHECKPOINT` and repeat the orphan query with no DDL running. A zero-length orphan that is still there is either a crash leftover or a lost empty relation, and only the file's history can tell which. A non-empty orphan that persists across that re-run is not a dropped relation, because `mdunlink()` empties a dropped relation's file when the drop commits. [xact.c:2232](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2232) [storage.c#pending-deletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L56)

### Index TID witness query

Use it on tables the first query flags, or on tables whose small size or missing forks make the fork witnesses blind. It reads each B-tree index of the table in full through shared buffers, so its cost is the size of those indexes. The heap size comes from the first short segment, matching a storage manager with fresh segment-open state rather than `pg_relation_size()`; `t_mid` in the measurement checks this distinction. A warm backend with later segments already open may use a larger bound. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L307)

The index side uses the other length rule, and that matters only when an index is damaged too. The query's page range comes from `pg_relation_size()` on the index, which adds every segment file. `bt_page_stats()` refuses any block at or past the storage manager's own length with `block number out of range`. So if an index lost a middle segment, the query stops with that error instead of returning a row. [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L307) [btreefuncs.c:52-54](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L52-L54) [btreefuncs.c:201](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L201) The error is itself a damage signal, and the main query reports such an index with `data after a short segment`. An index that lost only its last segment has equal lengths under both rules, so the query runs.

```sql
-- PostgreSQL 12 with contrib pageinspect installed; superuser only.
-- Set the table first, for example: \set table_name public.orders
\if :{?table_name}
\else
\echo 'set table_name first, for example: \\set table_name public.orders'
\quit
\endif
SET /* wiki_index_tid_witness */ statement_timeout = '30min';
SET /* wiki_index_tid_witness */ lock_timeout = '5s';
WITH /* wiki_index_tid_witness */ RECURSIVE
geom AS (
  SELECT database_block_size::bigint AS blcksz,
         blocks_per_segment::bigint AS relseg
  FROM pg_control_init()
),
heap_segs(segno, path, bytes) AS (
  SELECT 0::bigint, pg_relation_filepath(:'table_name'::regclass) AS path,
         (pg_stat_file(pg_relation_filepath(:'table_name'::regclass), true)).size
  UNION ALL
  SELECT s.segno + 1, s.path,
         (pg_stat_file(s.path || '.' || (s.segno + 1), true)).size
  FROM heap_segs AS s
  CROSS JOIN geom AS g
  WHERE s.bytes = g.relseg * g.blcksz
),
heap AS (
  SELECT coalesce(max(s.segno * g.relseg + s.bytes / g.blcksz), 0) AS blocks
  FROM heap_segs AS s
  CROSS JOIN geom AS g
  WHERE s.bytes IS NOT NULL
)
SELECT /* wiki_index_tid_witness */
       i.indexrelid::regclass AS index_name,
       heap.blocks AS heap_blocks,
       m.max_block AS max_referenced_block,
       m.max_block >= heap.blocks AS references_past_end
FROM pg_index AS i
JOIN pg_class AS ic ON ic.oid = i.indexrelid
JOIN pg_am AS am ON am.oid = ic.relam AND am.amname = 'btree'
CROSS JOIN heap
CROSS JOIN LATERAL (
  SELECT max((it.ctid::text::point)[0])::bigint AS max_block
  FROM (
    -- Leaf pages only; OFFSET 0 keeps bt_page_items() off other page types.
    SELECT b.blk::int AS blk, st.btpo_next
    FROM generate_series(1, pg_relation_size(i.indexrelid)
                            / current_setting('block_size')::bigint - 1) AS b(blk)
    CROSS JOIN LATERAL bt_page_stats(i.indexrelid::regclass::text, b.blk::int) AS st
    WHERE st.type = 'l'
    OFFSET 0
  ) AS leaf
  CROSS JOIN LATERAL bt_page_items(i.indexrelid::regclass::text, leaf.blk) AS it
  -- Item 1 of a non-rightmost leaf is its high key, not a heap pointer.
  WHERE leaf.btpo_next = 0 OR it.itemoffset > 1
) AS m
WHERE i.indrelid = :'table_name'::regclass
ORDER BY 1;
```

### Privileges, locks and cost

- **Privileges.** `pg_stat_file`, `pg_read_binary_file` and `pg_ls_dir` are revoked from `PUBLIC`. Run the first two queries as a superuser, or grant only the overloads they call. [system_views.sql:1330-1338](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1330-L1338)

  ```text
  GRANT /* wiki_relation_tail_check */ EXECUTE ON FUNCTION pg_stat_file(text, boolean) TO checker;
  GRANT /* wiki_relation_tail_check */ EXECUTE ON FUNCTION pg_read_binary_file(text, bigint, bigint, boolean) TO checker;
  GRANT /* wiki_orphan_relation_files */ EXECUTE ON FUNCTION pg_ls_dir(text, boolean, boolean) TO checker;
  ```

  Such a grant lets the role read any file under the data directory, because these functions accept any relative path below it. [genfile.c#convert_and_check_filename](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L59-L100) The `pageinspect` query needs superuser. [btreefuncs.c#superuser-check](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L320-L323)
- **Locks.** The queries take two kinds of lock, and they last for different times.
  - *Catalog scans, until the transaction ends.* The first query scans `pg_class` and `pg_namespace`, the orphan query `pg_class`, `pg_database` and `pg_tablespace`, and the `pageinspect` query `pg_index`, `pg_class` and `pg_am`. Each scanned table gets `AccessShareLock`, the lock an ordinary query takes on the tables it reads, and the executor closes them at the end without releasing it. [parse_relation.c:1230](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1230) [execMain.c:1573-1582](../../../../raw/postgres-12/src/backend/executor/execMain.c#L1573-L1582) These catalogs are checked relations too, so a concurrent `VACUUM FULL` or `CLUSTER` of one of them waits for the query.
  - *No lock on any other checked relation in the first two queries.* `pg_relation_filepath()` and `pg_relation_filenode()` read the `pg_class` row from the system cache, and the file functions use the operating system directly. [dbsize.c#pg_relation_filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1022) [genfile.c#pg_stat_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L362-L428) So these two queries block no concurrent `DROP`, rewrite or truncation of a user table or index.
  - *Index locks, one function call at a time.* The `pageinspect` query opens each B-tree index with `AccessShareLock` in `pg_relation_size()`, `bt_page_stats()` and `bt_page_items()`. [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336) [btreefuncs.c:182](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L182) [btreefuncs.c:336](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L336) Each function closes the index with `relation_close(rel, AccessShareLock)` before it returns, and `relation_close()` then releases the lock. [btreefuncs.c:213](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L213) [btreefuncs.c:373](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L373) [dbsize.c:333](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L333) [relation.c#relation_close](../../../../raw/postgres-12/src/backend/access/common/relation.c#L205-L217) The query never opens the heap: the heap's path comes from the system cache and its size from `pg_stat_file()`.

  What this means for concurrent commands on the checked table:

  | Concurrent command | Lock it needs | Effect while the `pageinspect` query runs |
  |---|---|---|
  | `DROP TABLE` | `AccessExclusiveLock` on each index, because dropping the table drops its indexes [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1548) [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2047-L2050) | Waits only for the function call in progress. The query's next call on that index then queues behind the drop and fails, either at the 5-second `lock_timeout` or because the index is gone. |
  | `REINDEX`, or a rewrite that rebuilds the indexes | `AccessExclusiveLock` on each index in `reindex_index()` [index.c:3466-3470](../../../../raw/postgres-12/src/backend/catalog/index.c#L3466-L3470) | Waits only for the call in progress. Later calls read the rebuilt index, so one result can mix old and new index pages. The query took each index's page range from `pg_relation_size()` before the scan, so a smaller rebuilt index makes a later `bt_page_stats()` call fail with `block number out of range`, and a larger one leaves its extra pages unread. [btreefuncs.c:52-54](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L52-L54) |
  | Ordinary `TRUNCATE` | A new heap file, then the same index rebuild through `reindex_relation()` [tablecmds.c:1836-1840](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1836-L1840) | As for `REINDEX`. |
  | VACUUM truncation of the heap | `AccessExclusiveLock` on the heap only [vacuumlazy.c:1902](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1902) | Does not wait. The query reads the heap size once, from the files. |

  None of the three queries gives a consistent view across relations or over time. See the concurrency row under [Branches, limits and false positives](#branches-limits-and-false-positives).
- **Timeouts.** Each block sets `statement_timeout` and `lock_timeout` with `SET`. Both are `PGC_USERSET` in 12 (the `user` [GUC context](../../../glossary.md#guc-context)), so the values apply only to the session that runs the check. [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2401)
- **Cost.** The first query stats each segment and fork file and reads `tail_blocks × 8 kB` per relation. In the measured catalog leg, one run covered 45,901 relations: the 45,630 `c_` tables still visible after the damage, plus 271 catalog relations.

### Measured results

The [measurement script](#measurement-script) built 12.2 from the pin with 8 MB segments, a non-default build knob explained [below](#non-default-settings-in-the-measurement). It created the fixtures and stopped the server. It then emptied files with `cat /dev/null > <file>` or overwrote pages with `dd`, restarted, and ran the three published queries as extracted from this page. On the `t_*` fixtures the main query ran with `tail_blocks=32`. The catalog leg ran it with the default 16, and the synthetic FSM check with 0, because that check tests only fork enumeration.

| Fixture | Damage | `cur_blocks` | `relpages` | FSM, VM pages | `min_expected_blocks` | Evidence and findings |
|---|---|---|---|---|---|---|
| `t_last` | last segment (1,018 pages) emptied | 3,072 | 4,090 | 4, 1 | 4,090 | strong: FSM past end, relpages, zero-length segment |
| `t_lastvac` | same; checked before its VACUUM | 3,072 | 4,090 | 4, 1 | 4,090 | strong: same three findings |
| `t_small_last` | last segment (428 pages) emptied | 3,072 | 3,500 | 3, 1 | 3,500 | stored-stat: relpages, zero-length segment |
| `t_ins` | last segment emptied; insert-only, never vacuumed or analyzed (depends on `autovacuum = off`, see [Non-default settings](#non-default-settings-in-the-measurement)) | 3,072 | 0 | 4, none | 4,070 | strong: FSM past end, zero-length segment |
| `t_vm` | last segment (1,016 pages) emptied | 31,744 | 32,760 | 11, 2 | 32,760 | strong: FSM and VM past end, relpages, zero-length segment |
| `t_mid` | middle segment `.1` emptied | 1,024 | 2,500 | 3, 1 | 2,500 | strong: data after a short segment, relpages, zero-length segment |
| `t_zero` | last 20 pages overwritten with zeros, length kept | 2,500 | 2,500 | 3, 1 | 2,500 | ambiguous: 20 all-zero tail pages, trailing run 20 |
| `t_bound` | none: VACUUM legitimately truncated it to 2,048 | 2,048 | 2,048 | 3, 1 | 2,048 | ambiguous: zero-length segment |
| `t_ok` and all catalogs of `zt` | none | — | — | — | — | not listed |

Further measured results:

- **Before the damage** the main query listed only `t_bound`. No catalog, TOAST table or index in the fixture database produced a finding, and the orphan query returned no rows.
- **Tail pages.** For all six truncated fixtures, the 32 tail pages read were valid. Only `t_zero`, the overwrite case, showed all-zero pages.
- **After one VACUUM** of `t_lastvac` and `t_small_last`, both had `relpages` 3,072. `t_lastvac` stayed `strong`, through its FSM witness (`min_expected_blocks` 4,070). `t_small_last` fell to `ambiguous`: only its zero-length segment was left, the same finding the legitimate `t_bound` shows.
- **Index witness.** The `pageinspect` query found the largest referenced heap block at 4,089 for `t_last_pkey` and 3,499 for `t_small_last_pkey`, both past the 3,072-block heaps. `t_small_last` still showed 3,499 after the VACUUM. For `t_mid`, the heap size was 1,024 blocks at the first short segment while its index still referenced block 2,499; summing all heap segment file sizes would instead report 1,476 blocks. The healthy `t_ok_pkey` stopped at 2,499 in a 2,500-block heap. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L307)
- **Catalog leg.** Database `zc` was grown to 49,006 small tables, which made `pg_class` 1,100 blocks: one full 1,024-block segment plus 76 blocks in segment 1. After segment 1 was emptied, the main query flagged `pg_catalog.pg_class`: `cur_blocks` 1,024, `relpages` 1,100, `stored-stat`. The orphan query returned exactly 3,376 files of 8,192 bytes each: one per table whose `pg_class` row had been in segment 1. They are 8,192 bytes only because the fixture gave each of those tables one row; empty lost tables would have shown 0-byte orphans (see [Non-default settings](#non-default-settings-in-the-measurement)). The default-plan count was 49,006 and the forced sequential count 45,630, as described under [Catalog tables](#catalog-tables). Run with `show_all=true`, the main query listed 45,901 relations: the 45,630 visible `c_` tables plus 70 catalog tables, 159 indexes and 42 TOAST tables. The 3,376 lost tables were not among them; only the orphan query saw them.
- **Segmented fork check.** The separate artificial `t_ok` FSM had one full 1,024-page segment plus a one-page `_fsm.1` segment. The main query reported `fsm_pages = 1,025` and flagged its fork as covering blocks past the 2,500-block heap. This tests file enumeration, not the meaning of the zero-filled FSM pages. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755)

These are measurements of what the built server and the published SQL did on this fixture. Why each witness behaves this way is established by the source citations above.

### Branches, limits and false positives

| Condition or state | Behavior of the check | Consequence |
|---|---|---|
| Last segment emptied, old size reached the first page of an FSM leaf beyond the new end, FSM populated | FSM witness fires | `strong` until a later VACUUM truncates the forks |
| Last segment emptied, old size did not reach that FSM boundary | No FSM witness; the VM may still witness the loss if its own boundary was crossed. Otherwise `relpages` may retain evidence until ANALYZE or a VACUUM that updates it | Check each witness separately; use the index witness if the forks have none |
| Table never had FSM slots written in the lost range, for example `COPY` into a table created or truncated in the same transaction | The FSM witness cannot fire: that `COPY` sets `TABLE_INSERT_SKIP_FSM`, and the insert path does not record full pages without the FSM. [copy.c:2766-2772](../../../../raw/postgres-12/src/backend/commands/copy.c#L2766-L2772) [hio.c:526-527](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L526-L527) | Rely on `relpages` or the index witness |
| After the damage, a VACUUM finds empty pages at the new end and truncates | `RelationTruncate()` also cuts the FSM and VM back. [vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1852-L1867) [storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L230-L295) | The fork witnesses are erased; the index witness remains |
| Table without indexes | No index witness exists | Only the forks and `relpages` can show the loss |
| An index of the table lost a middle segment | The index query's page range, from `pg_relation_size()`, runs past the storage manager's length, and `bt_page_stats()` raises `block number out of range`. [btreefuncs.c:52-54](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L52-L54) [btreefuncs.c:201](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L201) | No index witness row; the error and the main query's `data after a short segment` finding on that index show the damage |
| Legitimate VACUUM truncation to exactly K segments | Same zero-length-segment layout as damage (measured: `t_bound`) | `ambiguous`; the forks were truncated together, so no fork witness fires |
| All-zero pages at the end, from bulk extension or a crash | In 12, bulk extension adds up to 512 uninitialized pages per call and records them in the FSM. Multiple calls can extend the relation further. [hio.c#RelationAddExtraBlocks](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L186-L261) | `ambiguous`; the run length alone cannot establish overwriting |
| Pages extended or modified but not yet written | `pg_read_binary_file()` reads the file, not shared buffers. A newly extended page is written to the file as zeros and holds data only in its buffer until the buffer is written. [genfile.c#read_binary_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L111-L177) [bufmgr.c:867-872](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L867-L872) | Possible all-zero tail findings on busy tables; re-run after the next checkpoint |
| [Unlogged](../../../glossary.md#unlogged-relation) table after a crash | Recovery replaces its main fork with the empty init fork and removes the other forks. [reinit.c#ResetUnloggedRelations](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L36-L44) | `relpages` can be larger than the empty table: a `stored-stat` false positive |
| `TRUNCATE` of a table without indexes that was created or rewritten in the current [subtransaction](../../../glossary.md#subtransaction); inside a savepoint opened after that creation or rewrite, the truncation is transactional instead | The truncation is done in place by `RelationTruncate()`, and no `pg_class` update resets `relpages`. With indexes, the rebuilt indexes update it. [tablecmds.c:1793-1798](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1793-L1798) [heap.c#heap_truncate_one_rel](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3260-L3288) [heapam_handler.c:630-633](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L630-L633) [index.c:2978-2982](../../../../raw/postgres-12/src/backend/catalog/index.c#L2978-L2982) | `stored-stat` false positive |
| Concurrent DROP, rewrite or VACUUM truncation while the check runs | The main query and the orphan query read files against a `pg_class` snapshot, without locking any checked relation except the catalogs they scan. The `pageinspect` query locks each index only for the duration of one function call. A `DROP TABLE`, `TRUNCATE` or index-rebuilding rewrite therefore waits only for the call in progress; the query's later calls then fail or read the rebuilt index. A VACUUM heap truncation does not wait at all (see **Locks** under [Privileges, locks and cost](#privileges-locks-and-cost)). [btreefuncs.c:336](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L336) [btreefuncs.c:373](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L373) `mdunlink()` empties a dropped relation's first segment and removes it at the next checkpoint. [md.c#mdunlink](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L215-L262) | Transient findings; re-run and compare |
| Orphan file from a transaction still in progress, or from a crash during `CREATE` | The new file exists before its `pg_class` row commits, and the delete-at-abort request is kept only in the backend's memory. [storage.c#pending-deletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L56) | Orphan rows that are not catalog damage; a crash leftover persists |
| `pg_class` lost the row of a view, composite type, foreign table, or partitioned table or index | These relkinds have no data file, so no orphan file appears, and the main query never lists them. [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L187-L192) [heap.c:374-378](../../../../raw/postgres-12/src/backend/catalog/heap.c#L374-L378) | Not detected by any of the three queries |
| Temporary tables | Skipped: their pages live in the owning backend's local buffers, so the files need not hold current data. [localbuf.c:1-5](../../../../raw/postgres-12/src/backend/storage/buffer/localbuf.c#L1-L5) | Not checked |
| Either server byte order | The header test decodes both orders and accepts the one with a sane header and the expected layout version | This avoids classifying an otherwise valid big-endian page solely because its bytes are reversed; only the little-endian measurement is verified |

### State changes

| Event | State before | State after | Effect on a later check |
|---|---|---|---|
| Last segment emptied outside PostgreSQL | File length, forks, `relpages`, indexes agree | File shorter; forks, `relpages` and indexes still describe the old size | All applicable witnesses fire |
| ANALYZE, or VACUUM that counts pages, of the damaged relation (manual or autovacuum) | `relpages` old | `relpages` = `cur_blocks` (measured VACUUM 4,090 → 3,072) | The `relpages` witness disappears; forks and indexes stay. A VACUUM that counts no pages of a nonempty relation can preserve `relpages`. [vacuumlazy.c#stats-gate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L320-L356) |
| VACUUM truncation after the damage | Forks cover the old size | Forks cut back with the heap | The fork witnesses disappear; the index witness stays |
| [`REINDEX`](../../../glossary.md#reindex) of the damaged table's indexes | Entries point past the end | Rebuilt by a scan of the surviving, shorter heap. [index.c:3530](../../../../raw/postgres-12/src/backend/catalog/index.c#L3530) [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231) | The index witness disappears |
| [`VACUUM FULL`](../../../glossary.md#vacuum-full), [`CLUSTER`](../../../glossary.md#cluster), or ordinary [`TRUNCATE`](../../../glossary.md#truncate) | Old file and forks | A new relfilenode with consistent forks. [relcache.c#RelationSetNewRelfilenode](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3419-L3440) [cluster.c#rebuild_relation](../../../../raw/postgres-12/src/backend/commands/cluster.c#L588-L626) [tablecmds.c#ExecuteTruncateGuts](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1819) | File-based witnesses disappear; lost rows are not recovered |
| `TRUNCATE` of a table created or rewritten in the current subtransaction | The relation or its current file was created in that subtransaction | PostgreSQL truncates the existing relfilenode in place with `RelationTruncate()`, then rebuilds any indexes. [tablecmds.c#ExecuteTruncateGuts](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1819) [heap.c#heap_truncate_one_rel](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3260-L3288) | Forks are shortened too. Without indexes, the stored `relpages` value remains until later statistics maintenance; with indexes, the rebuild resets it. [index.c:2763](../../../../raw/postgres-12/src/backend/catalog/index.c#L2763) |
| `CHECKPOINT` after damage done under a running server | Dirty buffers of the lost pages may exist | They are written back at their offsets, which can partly regrow the file, as measured on the [zeroed-file page](zeroed-relation-file.md#truncating-a-running-servers-file-does-not-reliably-erase-it) | Mixed findings: regrown pages, all-zero gaps |

Run the check before any maintenance on a suspect database. A VACUUM, a `REINDEX` or a rewrite each erase part of the evidence.

### Final causal summary

**With fresh segment-open state: a last segment is emptied → `mdnblocks()` reports a shorter relation → the "last N blocks" now lie in the intact previous segment and read cleanly → populated FSM or VM forks, `pg_class.relpages`, or indexes may still describe the old size → the first query compares the file length with the forks and `relpages`, and the `pageinspect` query compares it with the indexes → a mismatch provides a witness subject to concurrent activity and legitimate state changes.** A backend with previously opened segments can have a different bound. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) Later maintenance can erase witnesses: ANALYZE rewrites `relpages`, VACUUM may rewrite it or truncate forks, `REINDEX` removes old index entries, and a rewrite replaces files. Reading tail pages remains useful for the other kind of zeroing, overwriting at unchanged length. For catalogs, the same query covers every catalog in the database. Lost rows of `pg_class` itself show up as orphan files when the lost relations have storage; lost rows of views and other relations without storage leave no trace these queries can see.

## Measurement Script

### Usage

| Item | Value |
|---|---|
| Purpose | Runs the three SQL blocks published on this page, extracted from the page itself, against PostgreSQL 12.2 fixtures whose segment files were emptied or overwritten while the server was stopped. Backs every number under [Measured results](#measured-results), the measured rows of the worked example, and the `ctid` probe. |
| Invocation | From the repository root, extract the Bash fence below to `.wiki-runtime/tmp/zt12-run.sh`, then run `bash .wiki-runtime/tmp/zt12-run.sh [stage ...]`. The script checks the pinned checkout and keeps every build and data file under `.wiki-runtime/tmp/zt12/`. |
| Stages | Default order: `build cluster extract fixtures damage check revacuum catalog fork_segments report`. `stop` and `clean` tear down and are not in the default order. |
| Environment | `PORT12` (default `55442`) is the isolated port; `JOBS` (default `8`) sets parallel build jobs. The checkout, page, sandbox paths and `RELSEG_SIZE` are fixed in the script. |
| Prerequisites | A C toolchain, GNU make, bison and flex; readline and zlib development headers, because `build` configures `--with-readline --with-zlib`; the pinned checkout at `raw/postgres-12/`; about 1 GB of free disk. Locale `C`, encoding `UTF8`. `pageinspect` is built and installed from the same tree. |
| Output | Files under `.wiki-runtime/tmp/zt12/out/`. Read `platform.txt` first, then `fixtures.txt`, `baseline.txt` and `orphans_baseline.txt` (before the damage), `targets.txt` and `damage.txt`, `check.txt`, `orphans_zt.txt`, `index_*.txt` (including `index_t_small_last_after.txt`), `tid_probe.txt`, `revacuum.txt`, `after_vacuum.txt`, `catalog_fixture.txt`, `catalog_check.txt`, `catalog_check_all.txt`, `catalog_orphans.txt` and `fork_segments.txt`. Each query result has a matching `.err` file, and the server log is `server.log`. Query results use `psql -A -F '|'`. |
| Runtime | On the 2026-10-29 host (Linux x86_64 under WSL2, 22 CPUs, `JOBS=20`): about 70 to 90 seconds from an empty sandbox, most of it the build; about 18 seconds to re-run every later stage while the build is present. The earlier Apple arm64 host (10 cores, `JOBS=8`) took about 2 minutes and 30 seconds. |
| Cleanup | Run `bash .wiki-runtime/tmp/zt12-run.sh stop clean` from the repository root. It stops the owned cluster, verifies that no PID file, process, socket or listener remains, and deletes `.wiki-runtime/tmp/zt12/`, including its build. Remove the extracted `.wiki-runtime/tmp/zt12-run.sh` afterwards. |

To extract the published script without editing it, run this from the repository root:

```sh
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/storage-and-vacuum/zeroed-last-segment-check-all-tables.md | sed '1d;$d' > .wiki-runtime/tmp/zt12-run.sh
bash -n .wiki-runtime/tmp/zt12-run.sh
bash .wiki-runtime/tmp/zt12-run.sh
```

### Last run

On 2026-10-29 the final published script text was extracted with the command above and run eight times from an empty sandbox on Linux 6.18.54.1-microsoft-standard-WSL2 x86_64 against PostgreSQL 12.2 at pin `45b88269a353ad93744772791feb6d01bc7e1e42`, the last after the seventh review of that day (72 s, then 19 s for the re-run), each run followed by a re-run of every later stage while the build was present. All stages and all 25 assertions passed on every run. Every number the 2026-10-21 run on Darwin 27.0.0 arm64 also measured matched it. The catalog leg's count of 45,901 covered relations was first recorded on 2026-10-29, after the sixth review added it to the script, and was the same on every run since. The server reported `RELSEG_SIZE 1024`, `segment_size` 8 MB, `block_size` 8192, maximum data alignment 8, `data_checksums` off and `shared_buffers` 128 MB. The main check returned the eight fixture findings in [Measured results](#measured-results), the B-tree query reported `t_mid` at 1,024 logical heap blocks with a reference to block 2,499, the catalog orphan query returned 3,376 rows, and the synthetic two-segment FSM check reported 1,025 pages. The latter is an artificial file-enumeration test, not a PostgreSQL-written FSM. The cluster was stopped and the sandbox deleted with `stop clean`.

### Stages

| Stage | What it does | Needs |
|---|---|---|
| `build` | Configures 12.2 out of tree under `$SANDBOX/build`, patches the generated `pg_config.h` to `RELSEG_SIZE 1024`, then builds and installs the server and `pageinspect`. Skipped when a matching build exists. | nothing |
| `cluster` | `initdb -A trust --locale=C --encoding=UTF8` (checksums off, the default), writes socket-only settings plus `shared_buffers`, `autovacuum = off` and `fsync = off`, starts on `PORT12`, and records `platform.txt` | `build` |
| `extract` | Copies the page's three `sql` fences, in order, to `$SANDBOX/sql/q1.sql` … `q3.sql` and checks each one's tag | `cluster` |
| `fixtures` | Recreates database `zt`, vacuums its catalogs, builds the nine `t_*` tables at exact block counts, truncates `t_bound` legitimately with VACUUM, checkpoints, records sizes, then runs the main and orphan queries as a baseline | `extract` |
| `damage` | Records the target files, stops the server, empties six segment files with `cat /dev/null >`, overwrites the last 20 pages of `t_zero`'s last segment with `dd ... conv=notrunc`, and restarts | `fixtures` |
| `check` | Runs the main query with `tail_blocks=32`, the orphan query, and the index query on four tables including `t_mid`, plus the `ctid` probe, and asserts the expected findings | `damage` |
| `revacuum` | Vacuums `t_lastvac` and `t_small_last`, re-runs the main and index queries, and asserts which witnesses survived | `damage` |
| `catalog` | Recreates database `zc`, grows `pg_class` past one segment, gives each table whose `pg_class` row lands in segment 1 one row, vacuums `pg_class`, empties `pg_class`'s segment 1 while stopped, then runs both counts, the main query, the main query again with `show_all=true` to count the relations it covers, and the orphan query | `extract` |
| `fork_segments` | While stopped, replaces disposable `t_ok`'s FSM with an artificial full 8 MB segment and one 8 kB second segment; checks that the published SQL counts all 1,025 pages. This tests the segment walk only, not a PostgreSQL-created FSM. | `fixtures`, `extract` |
| `report` | Lists the output files | any |
| `stop` | Stops the owned cluster and verifies that no `postmaster.pid`, process naming `$DATA`, socket or listener on `PORT12` remains | none |
| `clean` | Runs `stop`, then deletes `$SANDBOX`, refusing a sandbox it does not own | none |

Every fixture object is disposable. The script creates and drops databases `zt` and `zc` in its own cluster. `damage` refuses to run twice on the same fixtures; re-run `fixtures` first.

### Non-default settings in the measurement

| Setting or state | Default at the pin | Value in the script, and why |
|---|---|---|
| `RELSEG_SIZE` | 131,072 blocks (1 GB at 8 kB) [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L295) [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864) | 1,024 blocks (8 MB), patched only in the out-of-tree generated header, because `--with-segsize` takes whole gigabytes. With 1 GB segments, the fixtures would need tens of gigabytes to reach the witness thresholds. The queries read the segment size from `pg_control_init()`, `mdnblocks()` branches on "shorter than `RELSEG_SIZE`" rather than on its value, and the FSM and VM geometry does not depend on it. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) Which losses each witness can see does depend on segment size; the [worked example](#worked-example-when-the-fsm-and-vm-witnesses-can-fire) derives the default-size thresholds instead of measuring them. |
| Build options `--enable-debug` and `CFLAGS="-O2 -g"` | `--enable-debug` is off by default. Without an explicit `CFLAGS`, configure uses `-O2` for GCC; for other compilers it uses `-O`, or no optimization flag in debug mode [configure.in:197-201](../../../../raw/postgres-12/configure.in#L197-L201) [configure.in:405-421](../../../../raw/postgres-12/configure.in#L405-L421) | Debug symbols, so a failed run can be examined with a debugger, and a fixed `-O2` so the optimization level does not depend on the compiler. Neither changes what the server computes, so no measured result depends on them. `--with-readline` and `--with-zlib` are already the defaults [configure.in:867](../../../../raw/postgres-12/configure.in#L867) [configure.in:966](../../../../raw/postgres-12/configure.in#L966) |
| `initdb --locale=C --encoding=UTF8` | Without these options, `initdb` takes the locale from the environment and derives the encoding from `LC_CTYPE` [initdb.c#setlocales](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2300-L2330) [initdb.c:2555-2600](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2555-L2600) | A fixed locale and encoding, so the run does not depend on the host's environment. No reported number depends on them: fixture names and data are ASCII, and the regular-expression counts run on `pg_class.relname`, a `name` column. Catalog columns of a collatable type are given the C collation whatever the database locale is [genbki.pl:793-794](../../../../raw/postgres-12/src/backend/catalog/genbki.pl#L793-L794). `-A trust` matches the method `initdb` uses when none is given; naming it only suppresses the warning `initdb` prints in that case [initdb.c#check_authmethod_unspecified](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L2393-L2400) [initdb.c:3333-3338](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3333-L3338) |
| `autovacuum` | `on` [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1424-L1434) | `off` (`PGC_SIGHUP`, in `postgresql.conf` before first start). An autovacuum or auto-analyze run between `damage` and `check` would rewrite `relpages`, which `revacuum` instead does on purpose. With the default `on`, the `relpages` witness can disappear at any time after the damage. One result depends on it before the damage too: `t_ins` reports `relpages` 0 and no `relpages` finding only because nothing analyzed it. In 12, autovacuum analyzes a table once its changes since the last analyze exceed `autovacuum_analyze_threshold` (50) plus `autovacuum_analyze_scale_factor` (0.1) times `reltuples`. [autovacuum.c:3060-3080](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060-L3080) [guc.c:2957-2962](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2957-L2962) [guc.c:3404-3409](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3404-L3409) `t_ins` starts with `reltuples` 0 and receives 73,620 rows, so an auto-analyze can run before `damage` and store 4,090 in `relpages`. [analyze.c:591-605](../../../../raw/postgres-12/src/backend/commands/analyze.c#L591-L605) `t_ins` would then also show `relpages > current blocks`, and the script's `expect_no_finding` assertion for it would fail. "Never vacuumed" does not depend on the setting, because apart from wraparound prevention 12 starts an autovacuum only for dead tuples (`n_dead_tuples`), which an insert-only table does not have. |
| `fsync` | `on` [guc.c#fsync](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1132) | `off` (`PGC_SIGHUP`): a disposable cluster. Every damage step happens with the server cleanly stopped, so no measured read path depends on it. |
| `shared_buffers` | Two defaults. The compiled boot value is 1,024 buffers (8 MB) [guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2163). `initdb` instead writes the first value from its trial list that a test start accepts, beginning with 16,384 buffers, which is 128 MB at 8 kB pages [initdb.c:959-962](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L959-L962) [initdb.c:1018-1045](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1018-L1045) [initdb.c:1096-1102](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1096-L1102) | 128 MB (`PGC_POSTMASTER`), appended to `postgresql.conf` before the first start; `platform.txt` records the value in effect. It is non-default against the boot value and equal to `initdb`'s first trial. The script sets it so the run does not depend on how far `initdb`'s probe falls back on a small host. No result depends on it: the server is restarted after every damage step, so no measured result came from a cached page. |
| Planner switches for one count | `enable_indexscan`, `enable_indexonlyscan`, `enable_bitmapscan` all `on` [guc.c#scan-enables](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L894-L922) | Turned `off` with `SET` (user context, that statement's session only) for the catalog leg's second count, to force the sequential scan that shows 45,630. The default-plan count is reported beside it. |
| `statement_timeout`, `lock_timeout` | `0`, disabled [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2401) | 10 minutes and 5 seconds through `PGOPTIONS` for fixture statements (user context), and the published blocks' own `SET` values. They only bound runaway statements. |
| Fixture shape | — | Tables are filled to exact block counts, 18 rows of 440 bytes per page, and their indexes exist from creation, so no index build rewrites `relpages`. All but `t_ins` get one VACUUM so their FSM and VM cover every page. |
| Catalog-leg table contents | A table created with `CREATE TABLE` and never written has a zero-length first segment [storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L79-L122) | The `catalog` stage inserts one row into each `c_` table whose `pg_class` row lands in segment 1, so each lost table's file holds one 8,192-byte page. This makes the orphan files distinguishable from dropped relations, which `mdunlink()` truncates to zero bytes. [md.c#mdunlink](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L215-L262) The reported size of 8,192 bytes per orphan depends on it; the orphan count of 3,376 does not, because the query lists files of any size. Without the insert, every orphan would have been 0 bytes. |
| Synthetic FSM segment | By default, PostgreSQL extends an FSM as pages need slots and each fork segment obeys `RELSEG_SIZE`. [freespace.c#fsm_extend](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L615-L661) [storage.sgml#segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L233-L243) | The `fork_segments` stage writes two zero-filled FSM files while stopped to force the query's segment-enumeration branch. The resulting 1,025-page fork count is a measurement of SQL file enumeration only; it is not a server-generated FSM or evidence of a real old table size. |

`listen_addresses = ''` and `unix_socket_directories` isolate the cluster, and the port is 55442 instead of 5432. All three are `PGC_POSTMASTER` settings written before the first start. [guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936-L3963) [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177-L2188) Damage is done with `cat /dev/null > <file>`, as the requester specified: the shell's `>` truncates the file to zero length and keeps its name.

### The script

```bash
#!/usr/bin/env bash
#
# zt12-run.sh - the measurement script for the wiki page
# wiki/v12/questions/storage-and-vacuum/zeroed-last-segment-check-all-tables.md
#
# It builds the pinned PostgreSQL 12 checkout out of tree with a shortened
# RELSEG_SIZE, starts an isolated cluster, damages disposable fixture tables
# and one disposable database's pg_class while the server is stopped, and
# runs the three SQL checks published on the page.  The SQL is extracted from
# the page itself, so the measured SQL is the filed SQL.
#
# The build has one NON-DEFAULT compile-time knob that results depend on:
# RELSEG_SIZE is patched from the pin's default 131072 blocks (1 GB) to 1024
# blocks (8 MB) in the generated pg_config.h of the out-of-tree build, because
# configure's --with-segsize accepts only whole gigabytes in PostgreSQL 12.
# The checks read the segment size from the control file (pg_control_init),
# and the free-space-map and visibility-map geometry they test does not
# depend on it.  --enable-debug and CFLAGS="-O2 -g" are also non-default
# build options; they add debug symbols and fix the optimization level, and
# no measured result depends on them.
#
# Every fixture object is disposable.  The script creates and drops its own
# databases zt and zc; do not point it at a database anyone cares about.
#
# Segment files are zeroed with "cat /dev/null > <file>", as the requester
# specified: the shell's ">" redirection opens the file with O_TRUNC, so the
# file keeps its name and inode at zero length.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/zt12-run.sh                    # default stages
#   bash .wiki-runtime/tmp/zt12-run.sh fixtures damage check revacuum
#   bash .wiki-runtime/tmp/zt12-run.sh catalog            # catalog leg only
#   bash .wiki-runtime/tmp/zt12-run.sh stop clean         # tear down
#
# Stages: build cluster extract fixtures damage check revacuum catalog fork_segments
#         report stop clean
#
# Environment: PORT12 JOBS
set -uo pipefail
set -e

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
PAGE="$WIKI_ROOT/wiki/v12/questions/storage-and-vacuum/zeroed-last-segment-check-all-tables.md"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/zt12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55442}"
JOBS="${JOBS:-8}"

# RELSEG_SIZE for the patched build, in blocks (8 MB at BLCKSZ 8192).  The
# fixture sizes below are chosen against it and against the 8 kB geometry:
# 18 fixture rows per heap page, 4069 heap blocks per FSM leaf page, and
# 32672 heap blocks per VM page.
RELSEG_BLOCKS=1024

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; DATA="$SANDBOX/data"
OUT="$SANDBOX/out"; SQLD="$SANDBOX/sql"; SOCK="$SANDBOX/sock"; BIN="$INST/bin"
export PGPORT="$PORT12" PGHOST="$SOCK"

# statement_timeout and lock_timeout are PGC_USERSET in PostgreSQL 12; they
# travel through PGOPTIONS so they also cover VACUUM and the fixture DDL.
# The published SQL blocks set their own values with SET.
export PGOPTIONS="-c statement_timeout=10min -c lock_timeout=5s"

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# A failing stage must not leave the cluster running. Preserve output for
# diagnosis; an explicit clean stage removes the sandbox after inspection.
on_exit() {
  local rc=$?
  trap - EXIT
  if [ "$rc" -ne 0 ] && [ -f "$SANDBOX/.started" ] &&
     [ -f "$SANDBOX/.zt12-owned" ] &&
     [ "$(cat "$SANDBOX/.zt12-owned")" = "$PIN12 $WIKI_ROOT" ]; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop >&2 || true
    rm -f "$SANDBOX/.started"
  fi
  exit "$rc"
}
trap on_exit EXIT

# This script owns exactly one short sandbox name.  Never adopt a pre-existing
# cluster or delete a directory that this script did not create.
own_sandbox() {
  case "$WIKI_ROOT" in
    */postgres-llm-wiki) ;;
    *) die "run from the postgres-llm-wiki repository root" ;;
  esac
  [ -f "$WIKI_ROOT/AGENTS.md" ] || die "repository root not found"
  [ -f "$SANDBOX/.zt12-owned" ] || die "sandbox is not owned by this script: $SANDBOX"
  [ "$(cat "$SANDBOX/.zt12-owned")" = "$PIN12 $WIKI_ROOT" ] || die "sandbox owner/pin mismatch"
  export TMPDIR="$SANDBOX/tmp"
  mkdir -p "$TMPDIR"
}
source_pin() {
  [ "$(git --no-optional-locks -C "$SRC12" rev-parse HEAD)" = "$PIN12" ] \
    || die "PostgreSQL 12 checkout is not at $PIN12"
  [ -z "$(git --no-optional-locks -C "$SRC12" status --porcelain --untracked-files=no)" ] \
    || die "PostgreSQL 12 checkout has tracked changes"
}
tag_sql() {
  printf '%s\n' "$1" | sed -E 's/(^|;[[:space:]]*)(SELECT|INSERT|DELETE|CREATE|DROP|VACUUM|CHECKPOINT|EXPLAIN|SET)([[:space:]])/\1\2 \/\* wiki_zt12_fixture \*\/\3/g'
}

# -X ignores ~/.psqlrc; every psql call sets ON_ERROR_STOP.
q() { # q <db> <sql>
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$1" -c "$(tag_sql "$2")" > /dev/null \
    || die "failed on $1: $2"
}
s() { # s <db> <sql>: one unaligned result; the caller checks the status
  "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$1" -c "$(tag_sql "$2")"
}
run_q() { # run_q <db> <q1|q2|q3> <out-name> [psql -v options]: a published query
  local db=$1 query=$2 name=$3
  shift 3
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -A -F '|' -d "$db" "$@" \
      -f "$SQLD/$query.sql" > "$OUT/$name.txt" 2> "$OUT/$name.err" \
    || { cat "$OUT/$name.err" >&2; die "$query failed on $db"; }
  note "$name: $(($(wc -l < "$OUT/$name.txt") - 2)) result rows"
}
row_of() { # row_of <file> <relation>: the result line whose first field is <relation>
  grep -E "^$(printf '%s' "$2" | sed 's/\./\\./g')\|" "$OUT/$1.txt"
}
expect_finding() { # expect_finding <file> <relation> <finding text>
  row_of "$1" "$2" | grep -F -q -- "$3" || die "$1: $2 lacks \"$3\""
  note "ok: $1: $2 has \"$3\""
}
expect_no_finding() { # expect_no_finding <file> <relation> <finding text>
  if row_of "$1" "$2" | grep -F -q -- "$3"; then die "$1: $2 unexpectedly has \"$3\""; fi
  note "ok: $1: $2 lacks \"$3\""
}
expect_rows() { # expect_rows <file> <count>: data rows in a psql -A result
  local n
  n=$(($(wc -l < "$OUT/$1.txt") - 2))
  [ "$n" -eq "$2" ] || die "$1: $n result rows, expected $2"
  note "ok: $1: $n result rows"
}

# relfile <db> <table> <segno-or-fork>: absolute path of one on-disk file.
relfile() {
  local rel
  rel=$(s "$1" "SELECT pg_relation_filepath('$2')") || die "no file path for $2"
  case "$3" in
    0) printf '%s/%s\n' "$DATA" "$rel" ;;
    _*) printf '%s/%s%s\n' "$DATA" "$rel" "$3" ;;
    *) printf '%s/%s.%s\n' "$DATA" "$rel" "$3" ;;
  esac
}
bytes_of() { wc -c < "$1" | tr -d '[:space:]'; }

server_start() {
  # Mark the intent before pg_ctl: a partial start is still ours to stop.
  printf '%s\n' "$DATA" > "$SANDBOX/.started"
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w -t 60 start > /dev/null 2>&1 \
    || die "pg_ctl start failed, see $OUT/server.log"
}
server_stop() {
  "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  rm -f "$SANDBOX/.started"
}

# --------------------------------------------------------------- build -------
keep_build_logs() {
  local l
  for l in configure make install; do
    [ -f "$BUILD/$l.log" ] && cp "$BUILD/$l.log" "$OUT/$l.log"
  done
  return 0
}
stage_build() {
  say "build 12.2 out of tree from $SRC12, RELSEG_SIZE=$RELSEG_BLOCKS"
  source_pin
  if [ -e "$SANDBOX" ]; then
    own_sandbox
  else
    mkdir -p "$SANDBOX" || die "could not create sandbox"
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.zt12-owned"
    own_sandbox
  fi
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  if [ -x "$BIN/postgres" ]; then
    [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 $RELSEG_BLOCKS pageinspect" ] \
      || die "existing build has a different pin or segment size"
    note "matching build already present"
    return 0
  fi
  [ -x "$SRC12/configure" ] || die "no pinned checkout at $SRC12"
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" --enable-debug \
      --with-readline --with-zlib CFLAGS="-O2 -g" > configure.log 2>&1 ) \
    || { keep_build_logs; die "configure failed, see $OUT/configure.log"; }
  # Non-default compile-time knob for this measurement: configure accepts
  # --with-segsize only in whole gigabytes, so the generated pg_config.h is
  # patched to an 8 MB segment instead of building multi-gigabyte fixtures.
  sed "s/^#define RELSEG_SIZE .*/#define RELSEG_SIZE $RELSEG_BLOCKS/" \
      "$BUILD/src/include/pg_config.h" > "$BUILD/src/include/pg_config.h.new" \
    || die "could not patch RELSEG_SIZE"
  mv "$BUILD/src/include/pg_config.h.new" "$BUILD/src/include/pg_config.h"
  sed -n '/^#define RELSEG_SIZE /{p;q;}' "$BUILD/src/include/pg_config.h" | tee "$OUT/relseg_size.txt"
  [ "$(sed -n 's/^#define RELSEG_SIZE[[:space:]]*//p' "$OUT/relseg_size.txt")" = "$RELSEG_BLOCKS" ] \
    || die "RELSEG_SIZE patch did not take"
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 \
      && make -C contrib/pageinspect install >> install.log 2>&1 ) \
    || { keep_build_logs; grep 'error:' "$BUILD/make.log" | head -n 3 >&2
         die "make failed, see $OUT/make.log"; }
  keep_build_logs
  printf '%s %s pageinspect\n' "$PIN12" "$RELSEG_BLOCKS" > "$SANDBOX/.build-spec"
  note "$("$BIN/postgres" --version)"
}

# ------------------------------------------------------------- cluster -------
# Cluster settings, all written before the first start:
#   listen_addresses, unix_socket_directories, port, shared_buffers  PGC_POSTMASTER
#   autovacuum, fsync                                                PGC_SIGHUP
# autovacuum is off so no background worker vacuums or analyzes a fixture
# between damage and check (either would rewrite relpages); fsync is off
# because the cluster is disposable.  Data checksums are off by default.
stage_cluster() {
  say "isolated 12.2 cluster, socket-only, port label $PORT12"
  own_sandbox
  source_pin
  [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 $RELSEG_BLOCKS pageinspect" ] \
    || die "build specification mismatch"
  mkdir -p "$OUT" "$SQLD" "$SOCK"
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    [ -f "$SANDBOX/.started" ] || die "refusing to adopt an unowned running server"
    note "already running"
  else
    [ -z "$(ls -A "$SOCK")" ] || die "socket directory is not free"
    "$BIN/pg_isready" -q -h 127.0.0.1 -p "$PORT12" && die "port $PORT12 is occupied"
    if [ ! -d "$DATA" ]; then
      "$BIN/initdb" -D "$DATA" -A trust --locale=C --encoding=UTF8 \
        > "$OUT/initdb.log" 2>&1 || die "initdb failed, see $OUT/initdb.log"
      {
        printf "listen_addresses = ''\n"
        printf "unix_socket_directories = '%s'\n" "$SOCK"
        printf "port = %s\n" "$PORT12"
        printf "shared_buffers = 128MB\n"
        printf "autovacuum = off\n"
        printf "fsync = off\n"
      } >> "$DATA/postgresql.conf"
    fi
    server_start
  fi
  {
    uname -a
    "$BIN/postgres" --version
    printf 'RELSEG_SIZE from build: %s\n' "$(cat "$OUT/relseg_size.txt" 2>/dev/null)"
    s postgres "SELECT 'segment_size=' || current_setting('segment_size')"
    s postgres "SELECT 'block_size=' || current_setting('block_size')"
    s postgres "SELECT 'control: max_data_alignment=' || max_data_alignment || ' database_block_size=' || database_block_size || ' blocks_per_segment=' || blocks_per_segment FROM pg_control_init()"
    s postgres "SELECT 'data_checksums=' || current_setting('data_checksums')"
    s postgres "SELECT 'shared_buffers=' || current_setting('shared_buffers')"
  } > "$OUT/platform.txt"
  cat "$OUT/platform.txt" >&2
  grep -q '^block_size=8192$' "$OUT/platform.txt" || die "fixtures assume 8 kB blocks"
  grep -q "blocks_per_segment=$RELSEG_BLOCKS\$" "$OUT/platform.txt" || die "segment size mismatch"
}

# ------------------------------------------------------------- extract -------
# Pull the three sql fences out of the page, in order.  The fence string is
# assembled at run time so this script contains no literal Markdown fence.
stage_extract() {
  say "extract the published SQL from the page"
  own_sandbox
  mkdir -p "$SQLD"
  local fence n=0 inblock=0 line
  fence=$(printf '\140\140\140')
  while IFS= read -r line; do
    if [ "$inblock" -eq 0 ] && [ "$line" = "${fence}sql" ]; then
      n=$((n + 1)); inblock=1; : > "$SQLD/q$n.sql"; continue
    fi
    if [ "$inblock" -eq 1 ] && [ "$line" = "$fence" ]; then inblock=0; continue; fi
    [ "$inblock" -eq 1 ] && printf '%s\n' "$line" >> "$SQLD/q$n.sql"
  done < "$PAGE"
  [ "$n" -eq 3 ] || die "expected 3 sql blocks in the page, found $n"
  grep -q 'wiki_relation_tail_check' "$SQLD/q1.sql" || die "q1 is not the tail check"
  grep -q 'wiki_orphan_relation_files' "$SQLD/q2.sql" || die "q2 is not the orphan check"
  grep -q 'wiki_index_tid_witness' "$SQLD/q3.sql" || die "q3 is not the index witness"
  note "q1 $(wc -l < "$SQLD/q1.sql") lines, q2 $(wc -l < "$SQLD/q2.sql") lines, q3 $(wc -l < "$SQLD/q3.sql") lines"
}

# ------------------------------------------------------------ fixtures -------
# mk_tab <table> <heap-blocks> <vacuum|novacuum>: a logged heap of exactly
# <heap-blocks> blocks (18 rows of 440 bytes per page) with a PRIMARY KEY
# that exists from the start, so no index build rewrites relpages.  "vacuum"
# deletes 1 % of the rows and runs VACUUM, which records every page in the
# FSM and sets the VM; "novacuum" leaves the table insert-only.
mk_tab() {
  local t=$1 blocks=$2 got
  q zt "DROP TABLE IF EXISTS $t"
  q zt "CREATE TABLE $t (id bigserial PRIMARY KEY, pad text NOT NULL)"
  q zt "INSERT INTO $t (pad) SELECT repeat('x', 400) FROM generate_series(1, $((blocks * 18)))"
  got=$(s zt "SELECT pg_relation_size('$t') / 8192") || die "size of $t"
  [ "$got" = "$blocks" ] || die "$t has $got blocks, expected $blocks"
  if [ "$3" = vacuum ]; then
    q zt "DELETE FROM $t WHERE id % 101 = 0"
    q zt "VACUUM $t"
  fi
}
stage_fixtures() {
  say "disposable fixture tables in database zt"
  own_sandbox
  rm -f "$SANDBOX/.damaged"
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d postgres \
    -c "DROP /* wiki_zt12_fixture */ DATABASE IF EXISTS zt" \
    -c "CREATE /* wiki_zt12_fixture */ DATABASE zt" || die "could not create database zt"
  q zt "CREATE EXTENSION pageinspect"
  # Catalog control: vacuum the fresh database first, so the catalogs carry
  # FSM and VM forks and current relpages when the check reads them.
  q zt "VACUUM"
  mk_tab t_ok 2500 vacuum
  mk_tab t_last 4090 vacuum
  mk_tab t_lastvac 4090 vacuum
  mk_tab t_small_last 3500 vacuum
  mk_tab t_ins 4090 novacuum
  mk_tab t_vm 32760 vacuum
  mk_tab t_mid 2500 vacuum
  mk_tab t_zero 2500 vacuum
  mk_tab t_bound 2500 vacuum
  # Legitimate control: VACUUM truncates t_bound to exactly two segments,
  # which leaves the third segment file in place at zero length.
  q zt "DELETE FROM t_bound WHERE (ctid::text::point)[0] >= 2048"
  q zt "VACUUM t_bound"
  [ "$(s zt "SELECT relpages FROM pg_class WHERE relname = 't_bound'")" = 2048 ] \
    || die "t_bound was not truncated to 2048 blocks"
  [ "$(bytes_of "$(relfile zt t_bound 2)")" = 0 ] || die "t_bound segment 2 is not empty"
  q zt "CHECKPOINT"
  {
    printf 'table|relpages|main_bytes|fsm_bytes|vm_bytes\n'
    s zt "SELECT relname || '|' || relpages || '|' || pg_relation_size(oid) || '|' || pg_relation_size(oid, 'fsm') || '|' || pg_relation_size(oid, 'vm') FROM pg_class WHERE relname LIKE 't\_%' AND relkind = 'r' ORDER BY relname"
    printf 'ctid (4000,1) rows in t_last before damage: %s\n' \
      "$(s zt "SELECT count(*) FROM t_last WHERE ctid = '(4000,1)'")"
  } > "$OUT/fixtures.txt"
  cat "$OUT/fixtures.txt" >&2
  run_q zt q1 baseline -v tail_blocks=32
  expect_rows baseline 1
  expect_finding baseline public.t_bound "zero-length segment after full segments"
  run_q zt q2 orphans_baseline
  expect_rows orphans_baseline 0
}

# -------------------------------------------------------------- damage -------
# With the server stopped (so no shared buffer can write a page back), truncate
# six segment files to zero bytes and overwrite the last 20 blocks of one
# segment with zeros at unchanged length.
stage_damage() {
  say "damage fixture files while the server is stopped"
  own_sandbox
  [ ! -f "$SANDBOX/.damaged" ] || die "fixtures already damaged; re-run fixtures first"
  local spec t seg f nblk
  : > "$OUT/targets.txt"
  for spec in "t_last 3" "t_lastvac 3" "t_small_last 3" "t_ins 3" "t_vm 31" "t_mid 1"; do
    set -- $spec
    f=$(relfile zt "$1" "$2")
    [ -f "$f" ] || die "missing $f"
    [ ! -f "$(relfile zt "$1" $(($2 + 1)))" ] || [ "$1" = t_mid ] || die "$1 segment $2 is not the last"
    printf 'truncate|%s|%s|%s|%s\n' "$1" "$2" "$(bytes_of "$f")" "$f" >> "$OUT/targets.txt"
  done
  f=$(relfile zt t_zero 2)
  printf 'zero-last-20|t_zero|2|%s|%s\n' "$(bytes_of "$f")" "$f" >> "$OUT/targets.txt"
  server_stop
  while IFS='|' read -r action t seg bytes f; do
    case "$action" in
      truncate) cat /dev/null > "$f" || die "could not zero $f" ;;
      zero-last-20)
        nblk=$((bytes / 8192))
        dd if=/dev/zero of="$f" bs=8192 seek=$((nblk - 20)) count=20 conv=notrunc \
          2>> "$OUT/dd.log" || die "dd failed on $f" ;;
    esac
    printf '%s %s segment %s: %s -> %s bytes\n' "$action" "$t" "$seg" "$bytes" "$(bytes_of "$f")"
  done < "$OUT/targets.txt" | tee "$OUT/damage.txt" >&2
  server_start
  touch "$SANDBOX/.damaged"
}

# --------------------------------------------------------------- check -------
stage_check() {
  say "run the published checks after the damage, fresh backends"
  own_sandbox
  [ -f "$SANDBOX/.damaged" ] || die "run damage first"
  run_q zt q1 check -v tail_blocks=32
  expect_rows check 8
  expect_finding check public.t_last "FSM covers blocks past end"
  expect_finding check public.t_last "relpages > current blocks"
  expect_finding check public.t_lastvac "FSM covers blocks past end"
  expect_finding check public.t_small_last "relpages > current blocks"
  expect_no_finding check public.t_small_last "FSM covers blocks past end"
  expect_finding check public.t_ins "FSM covers blocks past end"
  expect_no_finding check public.t_ins "relpages > current blocks"
  expect_finding check public.t_vm "VM covers blocks past end"
  expect_finding check public.t_vm "FSM covers blocks past end"
  expect_finding check public.t_mid "data after a short segment"
  expect_finding check public.t_zero "all-zero pages in tail"
  expect_finding check public.t_bound "zero-length segment after full segments"
  run_q zt q2 orphans_zt
  expect_rows orphans_zt 0
  run_q zt q3 index_t_last -v table_name=public.t_last
  run_q zt q3 index_t_small_last -v table_name=public.t_small_last
  run_q zt q3 index_t_ok -v table_name=public.t_ok
  run_q zt q3 index_t_mid -v table_name=public.t_mid
  grep -q '|t$' "$OUT/index_t_last.txt" || die "t_last index does not reference past the end"
  grep -q '|t$' "$OUT/index_t_small_last.txt" || die "t_small_last index does not reference past the end"
  grep -q '|f$' "$OUT/index_t_ok.txt" || die "t_ok index unexpectedly references past the end"
  grep -q '|1024|2499|t$' "$OUT/index_t_mid.txt" \
    || die "t_mid index query did not use the heap's first short segment"
  {
    s zt "EXPLAIN (COSTS OFF) SELECT count(*) FROM t_last WHERE ctid = '(4000,1)'"
    printf 'ctid (4000,1) rows in t_last after damage: %s\n' \
      "$(s zt "SELECT count(*) FROM t_last WHERE ctid = '(4000,1)'")"
  } > "$OUT/tid_probe.txt"
  cat "$OUT/tid_probe.txt" >&2
}

# ------------------------------------------------------------ revacuum -------
# A VACUUM after the damage rewrites relpages from the shortened relation; the
# FSM/VM forks are not truncated because no trailing page of the surviving
# relation is empty.
stage_revacuum() {
  say "VACUUM two damaged tables, then check again"
  own_sandbox
  [ -f "$SANDBOX/.damaged" ] || die "run damage first"
  q zt "VACUUM t_lastvac"
  q zt "VACUUM t_small_last"
  s zt "SELECT relname || ' relpages=' || relpages || ' fsm_bytes=' || pg_relation_size(oid, 'fsm') || ' vm_bytes=' || pg_relation_size(oid, 'vm') FROM pg_class WHERE relname IN ('t_lastvac', 't_small_last') ORDER BY relname" \
    > "$OUT/revacuum.txt" || die "relpages after VACUUM"
  cat "$OUT/revacuum.txt" >&2
  run_q zt q1 after_vacuum -v tail_blocks=32
  expect_finding after_vacuum public.t_lastvac "FSM covers blocks past end"
  expect_no_finding after_vacuum public.t_lastvac "relpages > current blocks"
  expect_no_finding after_vacuum public.t_small_last "relpages > current blocks"
  expect_finding after_vacuum public.t_small_last "zero-length segment after full segments"
  run_q zt q3 index_t_small_last_after -v table_name=public.t_small_last
  grep -q '|t$' "$OUT/index_t_small_last_after.txt" \
    || die "t_small_last index lost its witness after VACUUM"
}

# ------------------------------------------------------------- catalog -------
# Grow database zc's pg_class past one segment, give the relations whose
# pg_class rows land in the second segment one row each (so their files are
# not empty), VACUUM pg_class, then truncate pg_class's last segment while
# the server is stopped.
stage_catalog() {
  say "catalog leg: truncate the last segment of pg_class in database zc"
  own_sandbox
  local f lost before after
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d postgres \
    -c "DROP /* wiki_zt12_fixture */ DATABASE IF EXISTS zc" \
    -c "CREATE /* wiki_zt12_fixture */ DATABASE zc" || die "could not create database zc"
  cat > "$SQLD/mk_catalog.sql" <<'EOF'
DO /* wiki_zt12_fixture */ $$
DECLARE
  n int := 0;
BEGIN
  WHILE pg_relation_size('pg_class') < 1100 * 8192 LOOP
    n := n + 1;
    EXECUTE format('CREATE /* wiki_zt12_fixture */ TABLE c_%s (i int)', n);
    IF n % 1000 = 0 THEN
      COMMIT;
    END IF;
  END LOOP;
END
$$;
DO /* wiki_zt12_fixture */ $$
DECLARE
  r record;
BEGIN
  FOR r IN SELECT c.oid::regclass AS t FROM pg_class AS c
           WHERE c.relname ~ '^c_[0-9]+$' AND (c.ctid::text::point)[0] >= 1024
  LOOP
    EXECUTE format('INSERT /* wiki_zt12_fixture */ INTO %s VALUES (1)', r.t);
  END LOOP;
END
$$;
VACUUM /* wiki_zt12_fixture */ pg_class;
CHECKPOINT /* wiki_zt12_fixture */;
EOF
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d zc -f "$SQLD/mk_catalog.sql" > /dev/null \
    || die "catalog fixture failed"
  before=$(s zc "SELECT count(*) FROM pg_class WHERE relname ~ '^c_[0-9]+\$'") || die "count before"
  lost=$(s zc "SELECT count(*) FROM pg_class WHERE relname ~ '^c_[0-9]+\$' AND (ctid::text::point)[0] >= 1024") \
    || die "lost count"
  f=$(relfile zc pg_class 1)
  [ -f "$f" ] && [ ! -f "$(relfile zc pg_class 2)" ] || die "pg_class does not end in segment 1"
  {
    printf 'pg_class relpages before damage: %s\n' "$(s zc "SELECT relpages FROM pg_class WHERE relname = 'pg_class'")"
    printf 'pg_class main bytes before damage: %s\n' "$(s zc "SELECT pg_relation_size('pg_class')")"
    printf 'c_ tables before damage: %s\n' "$before"
    printf 'c_ tables whose pg_class row is in segment 1: %s\n' "$lost"
    printf 'pg_class segment 1 bytes before damage: %s\n' "$(bytes_of "$f")"
  } > "$OUT/catalog_fixture.txt"
  server_stop
  cat /dev/null > "$f" || die "could not zero $f"
  server_start
  # The default plan for this count is an index-only scan of
  # pg_class_relname_nsp_index; with pg_class all-visible it never reads the
  # heap, so it still counts the lost rows.  The forced sequential count reads
  # the shortened heap.
  after=$(s zc "SELECT count(*) FROM pg_class WHERE relname ~ '^c_[0-9]+\$'") || die "count after"
  {
    printf 'c_ tables counted after damage, default plan: %s\n' "$after"
    s zc "EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF, SUMMARY OFF) SELECT count(*) FROM pg_class WHERE relname ~ '^c_[0-9]+\$'" \
      | sed 's/^/  plan: /'
  } >> "$OUT/catalog_fixture.txt"
  after=$(s zc "SET enable_indexscan = off; SET enable_indexonlyscan = off; SET enable_bitmapscan = off; SELECT count(*) FROM pg_class WHERE relname ~ '^c_[0-9]+\$'") \
    || die "sequential count after"
  printf 'c_ tables counted after damage, sequential scan: %s\n' "$after" >> "$OUT/catalog_fixture.txt"
  [ "$after" -eq $((before - lost)) ] || die "sequential count $after is not $before - $lost"
  cat "$OUT/catalog_fixture.txt" >&2
  run_q zc q1 catalog_check
  expect_finding catalog_check pg_catalog.pg_class "relpages > current blocks"
  # The same query with show_all=true lists every relation it covers; its row
  # count is the number of relations the check saw after the damage.
  run_q zc q1 catalog_check_all -v show_all=true
  printf 'relations the main query covered after damage: %s\n' \
    "$(($(wc -l < "$OUT/catalog_check_all.txt") - 2))" >> "$OUT/catalog_fixture.txt"
  tail -n 1 "$OUT/catalog_fixture.txt" >&2
  run_q zc q2 catalog_orphans
  expect_rows catalog_orphans "$lost"
}

# ------------------------------------------------------- fork_segments -------
# This artificial fork only checks file enumeration. It is not an FSM written
# by the engine; the pages are zero bytes and must not be used by a table scan.
stage_fork_segments() {
  say "count both segments of a synthetic FSM fork"
  own_sandbox
  local f
  f=$(relfile zt t_ok _fsm)
  [ -f "$f" ] || die "t_ok FSM missing"
  server_stop
  dd if=/dev/zero of="$f" bs=8192 count=1024 2>> "$OUT/dd.log" \
    || die "could not make full first FSM segment"
  dd if=/dev/zero of="$f.1" bs=8192 count=1 2>> "$OUT/dd.log" \
    || die "could not make second FSM segment"
  server_start
  run_q zt q1 fork_segments -v show_all=true -v tail_blocks=0
  row_of fork_segments public.t_ok | grep -Eq '\|1025(\.|\|)' \
    || die "FSM query did not count both segments"
  expect_finding fork_segments public.t_ok "FSM covers blocks past end"
}

# ------------------------------------------------------------- teardown ------
stage_report() {
  own_sandbox
  say "result files"
  ls -l "$OUT" | sed 's/^/   /' >&2
  note "read platform.txt first, then fixtures.txt, check.txt and catalog_fixture.txt"
}

stage_stop() {
  say "stop cluster"
  [ -e "$SANDBOX" ] || { note "sandbox absent"; return 0; }
  own_sandbox
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    [ -f "$SANDBOX/.started" ] || die "refusing to stop an unowned server"
    server_stop
  else
    note "not running"
  fi
  [ ! -f "$DATA/postmaster.pid" ] || die "postmaster.pid survived"
  if pgrep -f -- "$DATA" > /dev/null 2>&1; then
    pgrep -f -- "$DATA" >&2 || true
    die "a process still names $DATA"
  fi
  [ ! -d "$SOCK" ] || [ -z "$(ls -A "$SOCK")" ] || die "socket directory is not free"
  if [ -x "$BIN/pg_isready" ]; then
    "$BIN/pg_isready" -q -h "$SOCK" -p "$PORT12" && die "socket is still active"
    "$BIN/pg_isready" -q -h 127.0.0.1 -p "$PORT12" && die "port $PORT12 is occupied"
  fi
  rm -f "$SANDBOX/.started"
  note "stopped: no pid, process, socket, or listener on port $PORT12"
}

stage_clean() {
  say "delete sandbox $SANDBOX"
  stage_stop
  [ -e "$SANDBOX" ] || return 0
  own_sandbox
  rm -rf "$SANDBOX"
  note "deleted $SANDBOX"
}

# ------------------------------------------------------------- driver --------
stages="$*"
[ $# -eq 0 ] && stages="build cluster extract fixtures damage check revacuum catalog fork_segments report"
for st in $stages; do
  case "$(type -t "stage_$st" 2>/dev/null)" in
    function) "stage_$st" ;;
    *) die "unknown stage: $st" ;;
  esac
done
```

## Context Reviewed

- Storage manager: the segment-layout invariant, `mdnblocks()`, `_mdnblocks()`, `mdtruncate()` keeping inactive zero-length segments, `mdunlink()` keeping a dropped relation's first segment until the next checkpoint, and segment paths (`md.c`, `relpath.c`, `relpath.h`).
- SQL-callable file and size functions: `pg_stat_file`, `pg_read_binary_file`, `pg_ls_dir`, their path checks and `PUBLIC` revokes, `calculate_relation_size`, `pg_relation_size`, `pg_relation_filepath`, `pg_relation_filenode`, `pg_filenode_relation` and `RelidByRelfilenode` (`genfile.c`, `system_views.sql`, `pg_proc.dat`, `dbsize.c`, `relfilenodemap.c`), plus `pg_control_init` and `pg_control_system` (`pg_controldata.c`).
- Free space map: `SlotsPerFSMPage`, tree depth, `fsm_get_location`, `fsm_logical_to_physical`, `fsm_readbuf`, `fsm_extend`, `RecordPageWithFreeSpace`, `FreeSpaceMapTruncateRel`, and the heap paths that record free space: VACUUM, full-page inserts, bulk extension, and `COPY`'s `TABLE_INSERT_SKIP_FSM` (`fsm_internals.h`, `freespace.c`, `vacuumlazy.c`, `hio.c`, `copy.c`).
- Visibility map: `HEAPBLOCKS_PER_PAGE`, `visibilitymap_pin`, `visibilitymap_truncate`, `visibilitymap_count`, and the index-only scan's VM test (`visibilitymap.c`, `nodeIndexonlyscan.c`).
- `RelationTruncate()`, `pg_class.relpages` writers (`vac_update_relstats`, ANALYZE, `index_update_stats`), VACUUM's `relpages` corner case and truncation gate, in-place `TRUNCATE` of a table created or rewritten in the current subtransaction, `COPY`'s separate whole-transaction FSM skip, unlogged-relation reset, `RelationCreateStorage()` creating an empty first segment, and pending deletes (`storage.c`, `vacuum.c`, `vacuumlazy.c`, `analyze.c`, `index.c`, `heap.c`, `tablecmds.c`, `heapam_handler.c`, `reinit.c`).
- Page header layout, `PageIsNew`, `PD_VALID_FLAG_BITS`, `PG_PAGE_LAYOUT_VERSION` and `PageIsVerified()` (`bufpage.h`, `bufpage.c`); extension writing zero pages (`bufmgr.c`).
- Why `ctid` and bitmap probes drop pages past the end (`heapam_handler.c`, `nodeTidscan.c`).
- Mapped and shared catalogs (`relmapper.c`, `pg_tablespace.dat`, `relpath.c`); which relkinds get a data file (`RELKIND_HAS_STORAGE` in `pg_class.h`, `heap_create()` in `heap.c`); `pageinspect` B-tree functions and their superuser checks (`btreefuncs.c`, `pageinspect--1.5.sql`); psql `:{?name}` (`psql-ref.sgml`); build-time `RELSEG_SIZE` (`configure.in`, `pg_config.h.in`); GUC definitions for every setting the script changes (`guc.c`) and autovacuum's analyze trigger (`autovacuum.c`) and the `shared_buffers` value `initdb` probes and writes (`initdb.c`); `pageinspect`'s `CHECK_RELATION_BLOCK_RANGE` and its per-call index locks, released by `relation_close()` (`btreefuncs.c`, `relation.c`); the `AccessShareLock` a scanned catalog keeps (`parse_relation.c`); the index rebuild inside in-place `TRUNCATE` (`heap.c`, `index.c`); the locks `DROP TABLE`, `REINDEX`, `TRUNCATE` and VACUUM truncation take (`dependency.c`, `index.c`, `tablecmds.c`, `vacuumlazy.c`); configure and `initdb` defaults for the build and cluster options the script passes (`configure.in`, `initdb.c`), and catalog-column collation (`genbki.pl`).
- Tests: the pinned recovery TAP test deliberately removes unlogged relation forks, including the main fork, then checks crash reinitialization ([014_unlogged_reinit.pl#fork-removal](../../../../raw/postgres-12/src/test/recovery/t/014_unlogged_reinit.pl#L46-L79)). The FSM truncation recovery test checks a legitimate fork size change ([008_fsm_truncation.pl#test](../../../../raw/postgres-12/src/test/recovery/t/008_fsm_truncation.pl#L1-L96)). Neither tests these three published SQL queries. The `pageinspect` and `amcheck` regression suites exercise their functions on intact relations ([pageinspect/sql](../../../../raw/postgres-12/contrib/pageinspect/sql/btree.sql#L1-L21), [check_btree.sql#tests](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L1-L114)).
- Comparison set for the contradiction check: the [zeroed-file page](zeroed-relation-file.md), the [amcheck follow-up](../indexing/amcheck-zeroed-relation-segments.md), the [corruption-log page](../observability/corruption-log-messages-with-checksums-disabled.md), v12 search hits for `mdnblocks`, `RELSEG_SIZE`, `relpages`, `_fsm`, `_vm` and `pg_read_binary_file`, the linked glossary entries, the page's own index entries and the v12 row of `versions.md`. PostgreSQL 12 has no common-concept pages.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| An emptied last segment shortens the relation for a backend with fresh segment-open state; tail pages then read cleanly | [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755); measured `check.txt` |
| Full segments plus a zero-length segment is also a legitimate layout | [md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66), [md.c#mdtruncate](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L819-L838); measured `t_bound` |
| `ctid` and bitmap probes silently drop pages past the end | [heapam_handler.c#heapam_tuple_tid_valid](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L209-L215), [heapam_handler.c:2192-2199](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2192-L2199); measured `tid_probe.txt` |
| FSM geometry and the witness test | [fsm_internals.h#SlotsPerFSMPage](../../../../raw/postgres-12/src/include/storage/fsm_internals.h#L51-L61), [freespace.c#fsm_logical_to_physical](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L446-L476), [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L545-L607) |
| VM geometry and the witness test | [visibilitymap.c#HEAPBLOCKS_PER_PAGE](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L101-L115), [visibilitymap.c#visibilitymap_pin](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L189-L202) |
| PostgreSQL's own truncation shrinks forks with the heap | [storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L230-L295), [freespace.c#FreeSpaceMapTruncateRel](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L259-L340), [visibilitymap.c#visibilitymap_truncate](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L442-L545) |
| `relpages` is stored; ANALYZE and most VACUUMs rewrite it, but a VACUUM can retain it | [vacuumlazy.c#stats-gate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L320-L356), [analyze.c:591-605](../../../../raw/postgres-12/src/backend/commands/analyze.c#L591-L605), [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1182-L1184); measured `revacuum.txt` |
| Index entries keep pointing past the end | [vacuumlazy.c#lazy_scan_heap-nblocks](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L545-L556), [btreefuncs.c#bt_page_items](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L311-L400); measured `index_*.txt` |
| Mapped and shared catalogs need the mapper and `global/` | [relmapper.c:1-28](../../../../raw/postgres-12/src/backend/utils/cache/relmapper.c#L1-L28), [dbsize.c#pg_relation_filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1022), [relpath.c:140-205](../../../../raw/postgres-12/src/common/relpath.c#L140-L205) |
| Lost `pg_class` rows of relations with storage show as orphan files; an index-only count hides them; relations without storage leave no file | [dbsize.c#pg_relation_filenode](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L865-L904), [nodeIndexonlyscan.c:161-170](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L170); measured `catalog_*.txt`, [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L187-L192), [heap.c:374-378](../../../../raw/postgres-12/src/backend/catalog/heap.c#L374-L378) |
| Header validity test | [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L164), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162) |
| Normal all-zero tail pages | [hio.c#RelationAddExtraBlocks](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L186-L261), [bufmgr.c:867-872](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L867-L872) |
| Lock scope: catalog scans hold `AccessShareLock` to transaction end; `pageinspect` releases each index lock when its call returns | [parse_relation.c:1230](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1230), [execMain.c:1573-1582](../../../../raw/postgres-12/src/backend/executor/execMain.c#L1573-L1582), [btreefuncs.c:213](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L213), [btreefuncs.c:373](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L373), [dbsize.c:333](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L333), [relation.c#relation_close](../../../../raw/postgres-12/src/backend/access/common/relation.c#L205-L217) |
| In-place `TRUNCATE` leaves `relpages` stale only without indexes | [heap.c#heap_truncate_one_rel](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3260-L3288), [heap.c:3202](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3202), [index.c:2763](../../../../raw/postgres-12/src/backend/catalog/index.c#L2763) |
| Zero-length orphans have three causes; the catalog leg's orphans are non-empty because of its fixture | [md.c#mdunlink](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L215-L262), [storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L79-L122), [xact.c:2232](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2232); measured `catalog_orphans.txt` |
| Privileges | [system_views.sql:1330-1338](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1330-L1338), [genfile.c#convert_and_check_filename](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L59-L100), [btreefuncs.c#superuser-check](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L320-L323) |

## Open Questions

- The witness thresholds for the default 1 GB segments are derived from the FSM and VM formulas, not measured. The measurement used 8 MB segments.
- The four-level FSM branch used by non-default 1 and 2 kB block sizes follows the pinned `FSM_TREE_DEPTH` and `fsm_logical_to_physical()` definitions, but this measurement used only the default 8 kB block size. [freespace.c#FSM_TREE_DEPTH](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L67-L77) [freespace.c#fsm_logical_to_physical](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L446-L476)
- Heap extension writes pages without [WAL](../../../glossary.md#wal), and FSM updates are not WAL-logged either. [bufmgr.c:867-872](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L867-L872) [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L577-L582) Whether an operating-system crash, a file-level base backup or a standby can therefore legitimately end up with an FSM or VM longer than its heap has not been established from the source or measured. Treat a fork witness on such a cluster that implies only a few hundred missing pages with care.
- The index witness covers B-tree indexes only. Whether GIN, GiST, SP-GiST, hash or BRIN indexes can serve as a similar witness through `pageinspect` was not examined.
- The page-header decoding tests both byte orders, but this measurement ran only on little-endian hosts (Darwin arm64 and Linux x86_64). A big-endian run has not verified the result.
- No timing of the main query on a database with large relations was taken. The catalog leg's run covered 45,901 small relations, and the fixtures were at most 256 MB.
- PostgreSQL 12 has no common-concept page for relation segments and the FSM and VM geometry. Such a page would serve this page and the two zeroed-segment pages. None was created as a side effect of this answer.

## Source References

- [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L43-L849) — segment layout, `mdunlink`, `mdnblocks`, `mdtruncate`; [md.c#_mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1229-L1241).
- [dbsize.c](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L1022), [genfile.c](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L59-L510), [system_views.sql](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1330-L1338), [pg_controldata.c](../../../../raw/postgres-12/src/backend/utils/misc/pg_controldata.c#L265-L350), [relfilenodemap.c](../../../../raw/postgres-12/src/backend/utils/cache/relfilenodemap.c#L200-L210).
- [fsm_internals.h](../../../../raw/postgres-12/src/include/storage/fsm_internals.h#L20-L61), [freespace.c](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L67-L661), [visibilitymap.c](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L101-L545), [storage.c](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L295), [xact.c:2232](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2232).
- [vacuumlazy.c](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L320-L1902), [vacuum.c](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1182-L1184), [analyze.c](../../../../raw/postgres-12/src/backend/commands/analyze.c#L591-L605), [autovacuum.c](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060-L3080), [index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L2047-L3530), [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1548), [hio.c](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L186-L537), [copy.c](../../../../raw/postgres-12/src/backend/commands/copy.c#L2766-L2772).
- [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L209-L215), [heapam_handler.c:630-633](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L630-L633), [heapam_handler.c:2192-2199](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2192-L2199), [nodeTidscan.c](../../../../raw/postgres-12/src/backend/executor/nodeTidscan.c#L178-L182), [nodeIndexonlyscan.c](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L170), [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231).
- [bufpage.h](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L229), [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162), [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L867-L872).
- [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719), [storage.sgml#segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L233-L243) — index-specific zero-page check and segmented side forks.
- [relmapper.c](../../../../raw/postgres-12/src/backend/utils/cache/relmapper.c#L1-L28), [relpath.c](../../../../raw/postgres-12/src/common/relpath.c#L140-L205), [relpath.h](../../../../raw/postgres-12/src/include/common/relpath.h#L26-L27), [pg_tablespace.dat](../../../../raw/postgres-12/src/include/catalog/pg_tablespace.dat#L15-L20), [pg_class.h](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L60), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L187-L192).
- [heap.c:374-378](../../../../raw/postgres-12/src/backend/catalog/heap.c#L374-L378), [heap.c](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3183-L3288), [tablecmds.c](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1776-L1840), [reinit.c](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L36-L44), [relcache.c](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3419-L3440), [cluster.c#rebuild_relation](../../../../raw/postgres-12/src/backend/commands/cluster.c#L588-L626).
- [parse_relation.c:1230](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1230), [execMain.c:1573-1582](../../../../raw/postgres-12/src/backend/executor/execMain.c#L1573-L1582), [relation.c#relation_close](../../../../raw/postgres-12/src/backend/access/common/relation.c#L205-L217) — the lock a scanned table gets, and the early release when a function closes a relation with a lock mode.
- [btreefuncs.c](../../../../raw/postgres-12/contrib/pageinspect/btreefuncs.c#L52-L400), [pageinspect--1.5.sql](../../../../raw/postgres-12/contrib/pageinspect/pageinspect--1.5.sql#L162-L189), [psql-ref.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L816-L818).
- [configure.in](../../../../raw/postgres-12/configure.in#L197-L966) — `--enable-debug`, default `CFLAGS`, block and segment size, readline and zlib defaults, [pg_config.h.in](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864), [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L894-L3963) — scan enables, `fsync`, `autovacuum`, `shared_buffers`, `port`, timeouts and socket settings; [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L959-L3338) — the `shared_buffers` value `initdb` writes, locale and encoding defaults, and the default authentication method; [genbki.pl](../../../../raw/postgres-12/src/backend/catalog/genbki.pl#L793-L794) — C collation for catalog columns.
- [014_unlogged_reinit.pl](../../../../raw/postgres-12/src/test/recovery/t/014_unlogged_reinit.pl#L46-L79), [008_fsm_truncation.pl](../../../../raw/postgres-12/src/test/recovery/t/008_fsm_truncation.pl#L1-L96) — recovery fixtures that alter relation or fork files, separate from the published SQL checks.

## Navigation

- [Zeroed relation file: SELECT, VACUUM and pg_dump (unverified)](zeroed-relation-file.md)
- [Can amcheck detect zeroed relation segments? (unverified)](../indexing/amcheck-zeroed-relation-segments.md)
- [Corruption indicators with checksums disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md)
- [v12/index](../../index.md) · [wiki index](../../../index.md) · [Wiki Glossary (unverified)](../../../glossary.md)
