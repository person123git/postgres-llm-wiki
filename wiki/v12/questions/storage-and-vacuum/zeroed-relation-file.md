---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# A Manually Zeroed Relation File in PostgreSQL 12: Do SELECT, VACUUM, and pg_dump Error? (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Answer up front](#answer-up-front)
  - [One premise needs correcting first](#one-premise-needs-correcting-first)
  - [Mental model](#mental-model)
  - [Decision map](#decision-map)
  - [Measured outcomes, by file and by backend state](#measured-outcomes-by-file-and-by-backend-state)
  - [Why scans stay silent: the size check stops early](#why-scans-stay-silent-the-size-check-stops-early)
  - [When errors do appear](#when-errors-do-appear)
  - [The side forks are even quieter](#the-side-forks-are-even-quieter)
  - [Truncating a running server's file does not reliably erase it](#truncating-a-running-servers-file-does-not-reliably-erase-it)
  - [Regrowth and the free space map](#regrowth-and-the-free-space-map)
  - [State changes](#state-changes)
  - [What the size and statistics functions then report](#what-the-size-and-statistics-functions-then-report)
  - [Deleting the file is a different failure](#deleting-the-file-is-a-different-failure)
  - [Non-default build knob used for the measurement](#non-default-build-knob-used-for-the-measurement)
  - [Data checksums and complete page reads](#data-checksums-and-complete-page-reads)
  - [Tests](#tests)
  - [Final causal summary](#final-causal-summary)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Last run](#last-run)
  - [Stages](#stages)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

For PostgreSQL 12: with data checksums disabled, on a table with thousands of rows that has multiple files on disk, if one of these files is manually set to zero size, does a `SELECT` of all rows throw an error? Do `VACUUM` and `pg_dump` of the table error out?

## Answer

### Answer up front

**A sequential read can silently lose rows after a relation file is zeroed.** In the measured fresh-[backend](../../../glossary.md#backend) cases, `mdnblocks()` treats the first zero-length segment as the relation's end. A sequential `SELECT`, ordinary [`VACUUM`](../../../glossary.md#vacuum), and `pg_dump`'s `COPY TO` then succeed over that shortened block range. An already open segment list can instead give a scan a stale, larger block count. A request for a specific missing heap block can fail, although a page still in [shared buffers](../../../glossary.md#buffer-manager) can satisfy it without disk I/O. The result depends on file length, backend state, buffer state, the chosen plan, and VACUUM's page-skipping decision. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L295) [vacuumlazy.c#lazy_scan_heap-nblocks](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L545-L556) [copy.c#CopyTo-table_beginscan](../../../../raw/postgres-12/src/backend/commands/copy.c#L2114-L2137) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795)

### One premise needs correcting first

Row count alone does not establish that a table has multiple data files. A table or index splits into [segments](../../../glossary.md#storage-manager) when a fork exceeds `RELSEG_SIZE` blocks: 1 GB with the default 8 kB block and 1 GB segment settings. Subsequent files are named `filenode.1`, `filenode.2`, and so on. A small table can instead have separate [forks](../../../glossary.md#fork): the main fork, a [free space map](../../../glossary.md#free-space-map) (`_fsm`), and a [visibility map](../../../glossary.md#visibility-map) (`_vm`); an unlogged relation also has an init fork (`_init`). `pg_relation_filepath()` names the first main-fork segment. [storage.sgml#forks-and-segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L199-L243) [configure.in#with-segsize](../../../../raw/postgres-12/configure.in#L279-L292) [storage.sgml#pg_relation_filepath](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L281-L289)

The measurement below therefore uses a build whose `RELSEG_SIZE` is 128 blocks (1 MB segments), disclosed under [Non-default build knob used for the measurement](#non-default-build-knob-used-for-the-measurement), so that a ~4 MB table spans four segment files.

### Mental model

| State | Where it comes from | Later use |
|---|---|---|
| Segment length | `mdnblocks()` measures the last open segment and opens following segments until one is short or absent. Earlier open segments are trusted as full. | Sequential scans and VACUUM choose their block range from it. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) |
| Segment-open list | Each backend's `SMgrRelationData.md_seg_fds` retains the files it has opened; a relcache flush closes them. | A previously opened backend can calculate a larger block range after an external truncate. [smgr.h#SMgrRelationData](../../../../raw/postgres-12/src/include/storage/smgr.h#L40-L71) [relcache.c#RelationClearRelation](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L2412-L2419) |
| Shared buffer | A valid page may have been read before the file was zeroed. | `ReadBuffer_common()` returns a hit without a new `mdread()`, so a stale page can still supply tuples. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) |
| Visibility and free-space maps | The `_vm` holds all-visible bits; `_fsm` holds possible insertion targets. External truncation does not update them. | An index-only scan may avoid the heap, while an INSERT may request a block in the hole. [nodeIndexonlyscan.c#VM-check](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L170) [hio.c#FSM-target](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L381-L400) |

### Decision map

```text
Open relation after the external file change
  |-- first main file missing -> open error
  |-- first short segment found -> size-bounded relation end is before its missing blocks
  |     |-- sequential scan / ordinary VACUUM / COPY TO -> bounded by that end
  |     |-- index or FSM names a later block -> shared-buffer hit, or disk read/error
  |-- backend already opened later segments -> size may remain stale
        |-- shared-buffer hit or VACUUM page skip -> no read of the hole
        |-- disk read of the hole -> short-read error
```

The first two branches follow `mdopen()` and `mdnblocks()`; block-addressed reads follow the buffer lookup, `mdread()`, and `_mdfd_getseg()`; VACUUM can skip runs of all-visible pages under its regular-scan conditions. [md.c#mdopen](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L437-L475) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [bufmgr.c#buffer-hit-or-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L589-L630) [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1112-L1219) [vacuumlazy.c#page-skipping](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L565-L636)

### Measured outcomes, by file and by backend state

A fresh backend means one that opens the relation's files after the truncation — a new connection, or every backend after a restart. A warm backend already had the relation's segments open when the file was truncated. The fixture was an 8,614-row table spread over four segment files (three full 1 MB segments plus a partial fourth) with a `PRIMARY KEY` index, an `_fsm`, and a `_vm`. Data checksums were off, their `initdb` default; the script also disabled autovacuum so its background worker could not alter the fixture while it was probed. [initdb.c#data_checksums](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1424-L1434)

| File zeroed | `SELECT` (seq scan) | `count(id)` plan | `VACUUM` | `pg_dump -t` | Index-driven fetch |
|---|---|---|---|---|---|
| Only file of a 1-segment table | 0 rows, no error | 0 rows (Seq Scan) | OK; `relpages` → 0 | OK, 0 rows | `XX001 could not read block 0 ... read only 0 of 8192 bytes` |
| Middle segment (`.1` of `.0`-`.3`) | 2,282 rows (segment 0 only), no error | **8,614 rows (Index Only Scan)** | OK; `relpages` 484 → 128 | OK, 2,282 rows | block in `.1` → `XX001 ... read only 0 of 8192 bytes`; block in `.2`/`.3` → `XX000 could not open file ".2" (target block 256): previous segment is only 0 blocks` |
| Last segment (`.3`) | 6,844 rows, no error | 8,614 rows (Index Only Scan) | OK; `relpages` → 384 | OK, 6,844 rows | `XX001 could not read block 384 in file ".3": read only 0 of 8192 bytes` |
| First segment of a multi-segment table | 0 rows, no error | 0 rows (Seq Scan) | OK; `relpages` → 0 | OK, 0 rows | block past the hole → `XX000 ... ".1" ... previous segment is only 0 blocks` |
| `_fsm` fork | OK, all rows | Index Only Scan, all rows | OK; fork is recreated | OK | n/a |
| `_vm` fork | OK, all rows | Index Only Scan falls back to heap fetches, all rows | OK; fork is recreated | OK | n/a |

And for the same middle-segment truncation seen by a **warm** backend — one that had just run `EXPLAIN` over the table, which opens every existing segment as a side effect of `mdnblocks()`:

| Operation in the warm session | Result |
|---|---|
| `SELECT count(*)` (seq scan) | `ERROR XX001 could not read block 128 in file ".1": read only 0 of 8192 bytes` |
| `VACUUM` | **OK** — the visibility map marks every page all-visible, so lazy vacuum's page-skipping never reads the hole; its `pg_class` update then flushes the relcache and smgr, so... |
| `SELECT count(*)` again, same session | 2,282 rows, no error — the reopened smgr now measures the shrunken relation |

### Why scans stay silent: the size check stops early

Every sequential read path first asks how many blocks the relation has. `RelationGetNumberOfBlocks()` calls `smgrnblocks()` → `mdnblocks()`, which walks the segment files and returns `segno * RELSEG_SIZE + nblocks` at the **first** segment that is shorter than `RELSEG_SIZE`, or `segno * RELSEG_SIZE` at the first segment that fails to open. A zero-length segment is therefore interpreted as end-of-relation, not corruption: whatever exists in later files becomes unreachable to size-bounded scans. [bufmgr.h#RelationGetNumberOfBlocks](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L188-L199) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [md.c#_mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1226-L1241)

The consumers all bound themselves by that number:

- `SELECT`: `initscan()` sets `scan->rs_nblocks = RelationGetNumberOfBlocks(...)` and `heapgettup` never reads a page `>= rs_nblocks`. [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L295) [heapam.c#heapgettup](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L713-L733)
- `VACUUM`: `lazy_scan_heap()` takes `nblocks = RelationGetNumberOfBlocks(onerel)` and scans `blkno < nblocks`, reading each page with `ReadBufferExtended(RBM_NORMAL)`. [vacuumlazy.c#nblocks](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L549-L556) [vacuumlazy.c#scan-loop](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L638-L657) [vacuumlazy.c#read-block](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L816-L820)
- `pg_dump`: `dumpTableData_copy()` issues `COPY <table> TO stdout`; `CopyTo()` builds the same `table_beginscan` sequential scan, so the dump contains exactly the rows a sequential `SELECT` sees, and exits 0. A *failed* COPY instead makes pg_dump abort with `Dumping the contents of table "<name>" failed: PQgetResult() failed.` and exit status 1. [pg_dump.c#copy-to-stdout](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1826-L1836) [pg_dump.c#copy-failure](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1900-L1929) [copy.c#CopyTo](../../../../raw/postgres-12/src/backend/commands/copy.c#L2114-L2137)

In a fresh backend, if those operations read only their size-bounded range and encounter no other corruption, a zeroed *middle* segment can make `SELECT`, `VACUUM`, and `pg_dump` succeed while omitting the truncated segment **and every segment after it**. In this fixture, the dump held fewer rows than the surviving files contained and emitted no corruption warning.

### When errors do appear

Three mechanisms turn the same zeroed file into an error:

1. **Block-addressed reads inside the hole.** An index scan can retain tuple identifiers for blocks in the truncated or later segments because the index files are untouched. `heapam_index_fetch_tuple()` requests the heap page with `ReleaseAndReadBuffer()` before calling `heap_hot_search_buffer()`. On a shared-buffer miss, a read inside the zeroed segment reaches `mdread()`: a short read with `zero_damaged_pages` off and no recovery raises `XX001 could not read block ... read only 0 of 8192 bytes`. A requested block *after* the hole instead reaches `_mdfd_getseg()`, which rejects the segment gap. The fixture showed `XX000` for this error, but that SQLSTATE is not guaranteed: `errcode_for_file_access()` inspects `errno` even though this branch made no failing system call. A shared-buffer hit need not read the damaged file at all. [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L129-L160) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [md.c#mdread-short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L575-L631) [md.c#_mdfd_getseg-segment-gap](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1219) [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L583-L652)

2. **A warm backend's stale segment table.** `mdnblocks()` starts from the last open segment and trusts earlier open segments as full. Engine-driven truncation sends an smgr invalidation; an external file change does not. A backend that previously opened all segments, including through the planner's `heapam_estimate_rel_size()` during `EXPLAIN`, can therefore compute the old block count. If its scan needs an uncached page in the hole, it gets the `mdread()` short-read error. An ordinary `VACUUM` may still succeed when its visibility-map skip covers the hole; if it subsequently changes `pg_class` statistics, the resulting relcache invalidation can make the next command reopen the files and see the shorter length. [md.c#mdnblocks-open-segments](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L715-L729) [smgr.c#smgrtruncate-inval](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L645-L670) [heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2071-L2084) [vacuumlazy.c#vm-page-skip](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L565-L636) [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1179-L1259) [relcache.c#RelationClearRelation](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L2399-L2429)

3. **A surviving free space map.** The `_fsm` fork is a separate file and keeps its stale entries. An `INSERT` consults it via `GetPageWithFreeSpace()`; if it nominates a page inside the zeroed segment, `ReadBufferBI` on that block hits the same `XX001` short read — measured as `could not read block 129 in file ".1"`. [hio.c#GetPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L381-L400) [hio.c#target-block-loop](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L403-L433)

Conversely, an **index-only** scan never touches the heap when the visibility map (also untouched) says the pages are all-visible, so `count(id)` returned all 8,614 rows even while the sequential count saw 2,282 — two different answers to "how many rows" from the same truncated table. [nodeIndexonlyscan.c#vm-check](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L189)

### The side forks are even quieter

The `_fsm` and `_vm` readers cache each fork's block count in the relation's storage-manager state. If that count is invalid, they measure the fork; if the requested block is beyond the count, they return no information or extend the fork. Thus a newly opened backend measures a zeroed fork as empty: an absent FSM page supplies no free-space hint, and an absent VM page supplies no all-visible bit. In the fixture, `count(id)` still returned all rows after zeroing `_vm` but fetched 8,614 heap tuples; subsequent `INSERT` or `VACUUM` recreated the side-fork contents. A warm backend can retain a stale fork count, so the fresh-backend result does not guarantee that every request avoids disk I/O. [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L552-L605) [visibilitymap.c#vm_readbuf](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L567-L618) [nodeIndexonlyscan.c#VM-check](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L170)

These fork readers use `RBM_ZERO_ON_ERROR`. It substitutes a zero page when a *complete* disk read fails `PageIsVerified()`, whether from an invalid header or a bad checksum on a checksummed cluster. It does not rescue `mdread()`'s short-read error. In the measured warm-fork sessions, the needed fork buffers remained cached, so no short read was observed. [bufmgr.c#zero-on-error](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L161) [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)

### Truncating a running server's file does not reliably erase it

`truncate -s0` shortens the file, but a dirty page still in shared buffers can later be written back at its original block offset. After the script dirtied a heap page in segment 1 and FSM pages, zeroed both files, and ran `CHECKPOINT`, segment `.1` measured 335,872 bytes and `_fsm` measured 24,576 bytes. A server restart discarded shared buffers; the next sequential count read 2,466 rows from disk, including 162 in the partly restored segment. This confirms that the observed rows were not merely old buffer hits. `mdwrite()` uses the block's offset within its segment and does not check whether another process shortened the file. Which dirty pages are flushed determines the visible result. [md.c#mdwrite](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L633-L695) [bufpage.c#PageIsVerified-all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142)

### Regrowth and the free space map

After a middle segment is zeroed, the table can regrow *over* the hole. With the text column's default EXTENDED storage, ordinary 400-byte rows may use the surviving FSM's old free-space hints and fail when directed into the empty segment. The script then sets that column to PLAIN storage and inserts page-sized text values, which cannot fit on the FSM's existing pages. The insertion path therefore extends the relation instead. [pg_type.dat#text](../../../../raw/postgres-12/src/include/catalog/pg_type.dat#L81-L88) [hio.c#FSM-target](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L370-L423) [hio.c#extend-path](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L545-L609)

`mdextend()` writes at `blocknum % RELSEG_SIZE` in the chosen segment. In the fixture, 300 large rows refilled `.1`, then insertion resumed in the previously partial `.3` before creating `.4` and `.5`. Once `.1` was full, `mdnblocks()` followed the surviving `.2` and `.3` again. The sequential count rose to 6,633, including the 2,281 older rows in `.2`; an index fetch also reached a stale row. [md.c#mdextend](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L374-L425) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L731-L755)

### State changes

| Event | State before | State after | Effect on the next reader |
|---|---|---|---|
| External truncation | Full files and possibly open segment descriptors or cached pages | File is short; backend metadata and buffers are unchanged | A new opener measures the short file; an already open backend can retain the old bound. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) |
| Ordinary `VACUUM` | Short file; possibly stale catalog statistics | It may skip all-visible pages. It updates relation statistics only as its scan and dirty-field checks permit. | A changed `pg_class` row can invalidate relcache state; an unchanged row need not. [vacuumlazy.c#stats-gate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L320-L356) [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1179-L1259) |
| Extension or checkpoint | A short file with surviving later segments or dirty buffers | New blocks refill the short segment, or dirty buffers write their old offsets | Later reads may again reach old segments or partly restored pages. [md.c#mdextend](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L374-L425) [md.c#mdwrite](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L640-L695) |
| Restart | Shared buffers and per-backend segment state may hide disk state | Both caches are discarded | New backends use current file lengths and read pages from the surviving files. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [bufmgr.c#ReadBuffer_common](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) |

### What the size and statistics functions then report

- `pg_relation_size(rel, 'main')` does **not** use `mdnblocks`: `calculate_relation_size()` `stat()`s every existing segment file and sums `st_size`, stopping only at the first missing name — so it still counted the later segments (2,916,352 bytes) while a scan could only reach 1 MB. For a *deleted* middle file it stops at the gap (1,048,576). [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L331)
- In the middle-segment fixture, ordinary `VACUUM` changed `pg_class.relpages` from 484 to 128 and `reltuples` to 2,278. This is conditional, not an inevitable consequence of every vacuum: when no pages are counted because all are skipped as all-visible, `lazy_scan_heap()` retains old page and tuple statistics; `vac_update_relstats()` writes a catalog row only if a tracked field differs. [vacuumlazy.c#stats-gate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L320-L356) [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1179-L1259)

### Deleting the file is a different failure

A **missing** file fails differently from a **zero-length** one, because `mdopen()`/`_mdfd_openseg()` cannot open it at all:

- First segment deleted: operations that open the heap fail — `SELECT` → `58P01 could not open file "base/...": No such file or directory`, `VACUUM` the same, and `pg_dump` aborts with `Dumping the contents of table ... failed: PQgetResult() failed.` The size function can instead return 0. [md.c#mdopen](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L437-L475) [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L331)
- Middle segment deleted: `mdnblocks()` still stops at the gap (sequential operations stay silent), but an index fetch past it gets `58P01 could not open file "%s" (target block %u): No such file or directory` from `_mdfd_getseg`. [md.c#_mdfd_getseg-open-fail](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1207-L1219)

### Non-default build knob used for the measurement

PostgreSQL 12's `configure` accepts `--with-segsize` only in whole gigabytes (`RELSEG_SIZE = 1024/blocksize * segsize * 1024` blocks), so exercising multi-segment behavior at the default 1 GB needs a >1 GB table. The script instead configures out of tree, then edits the generated `pg_config.h` to `RELSEG_SIZE 128` — a disclosed compile-time override, valid because the code treats `RELSEG_SIZE` as a constant with no behavioral special-casing of its value ("a power-of-2 value is recommended but not absolutely required; changing it requires an initdb", and the cluster here was initdb'd by the same build). Every measured code path (`mdnblocks`, `_mdfd_getseg`, `mdread`, `mdextend`, the buffer modes) branches on *whether a segment is shorter than `RELSEG_SIZE`*, never on the constant's magnitude. The build's `SHOW segment_size` printed `1MB`, recorded in `out/platform.txt`. [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L292) [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864)

### Data checksums and complete page reads

The script uses `initdb`'s default of [data checksums](../../../glossary.md#data-checksums) off. A zero-length segment produces a *short* read, and `mdread()` handles that before the buffer manager can verify a page. With the default `zero_damaged_pages = off` and outside recovery, it errors. A complete page read instead passes through `PageIsVerified()`: with checksums enabled, an initialized page is checked against `pd_checksum`; an entirely zero page is accepted even though it has no usable page header. Thus checksums can matter for a readable damaged page, but they do not alter this script's zero-length short-read path. [md.c#mdread-short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) [bufmgr.c#verify-path](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L161) [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1149-L1161)

The related [corruption indicators page](../observability/corruption-log-messages-with-checksums-disabled.md) covers other log messages.

### Tests

The pinned recovery suite includes an engine-driven FSM truncation test, but a search of the pinned regression and recovery tests found no test that externally zeroes a relation file while the server runs. This page's script records that case; the source paths above explain why each result occurs. [008_fsm_truncation.pl#test](../../../../raw/postgres-12/src/test/recovery/t/008_fsm_truncation.pl#L1-L96)

### Final causal summary

External truncation → a fresh backend measures the first short segment as the relation end → size-bounded scans, ordinary `VACUUM`, and `pg_dump` can succeed with fewer rows. A warm backend, a block-directed fetch, or a surviving buffer can see a different result. Later extension or dirty-buffer flush can make part of the truncated file visible again. A short read raises `XX001` under the measured defaults; a request beyond the gap reaches a separate open error whose SQLSTATE depends on `errno`. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1219) [md.c#mdwrite](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L640-L695)

## Measurement Script

### Usage

| Field | Value |
|---|---|
| Purpose | measures what `SELECT`, `count(id)`, `VACUUM`, `pg_dump`, index-driven fetches, `INSERT`, and `CHECKPOINT` do in PostgreSQL 12.2 with data checksums disabled after one relation file (main-fork segment, `_fsm`, or `_vm`) is truncated to zero bytes, for both fresh and already-warmed backends |
| Invocation | From the repository root, extract the Bash fence below to `.wiki-runtime/tmp/zf12-run.sh`, then run `bash .wiki-runtime/tmp/zf12-run.sh [stage ...]`. The script checks the pinned checkout and uses `.wiki-runtime/tmp/zf12/` for all build and data files. |
| Stages | `build cluster fixtures legs` in default order; the individual `leg_*` names re-run single legs; `report` lists outputs; `stop` and `clean` tear the sandbox down and are not in the default order |
| Environment | `PORT12=55440` selects the isolated port; `JOBS=8` sets parallel build jobs; `RELSEG_BLOCKS=128` sets blocks per segment in the patched build; `ROWS_PER_SEG=2400` sets the fixture row scale. The checkout and sandbox paths are fixed inside the script. |
| Prerequisites | a C toolchain, GNU make, bison and flex (the pin is a git checkout, so generated parsers are built); readline and zlib development headers, because `build` configures `--with-readline --with-zlib`; the pinned checkout at `raw/postgres-12/` |
| Output | one file per leg under `$SANDBOX/out/`; read `platform.txt` first — it records the patched `RELSEG_SIZE` (`#define RELSEG_SIZE 128`, `SHOW segment_size` → `1MB`) and `data_checksums = off` |
| Runtime | on the review host: about 3 minutes for `build` plus `cluster` from an empty sandbox, under a minute for `fixtures legs` with the build present |
| Cleanup | `bash .wiki-runtime/tmp/zf12-run.sh stop clean` stops the owned cluster, verifies that its PID, processes, socket, and port are gone, then deletes `.wiki-runtime/tmp/zf12/`. Remove the extracted `zf12-run.sh` after use. |

To extract the published script without editing it, run this from the repository root:

```sh
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/storage-and-vacuum/zeroed-relation-file.md | sed '1d;$d' > .wiki-runtime/tmp/zf12-run.sh
bash -n .wiki-runtime/tmp/zf12-run.sh
bash .wiki-runtime/tmp/zf12-run.sh
```

### Last run

On 2026-10-20, the published script was run on Darwin arm64 against PostgreSQL 12.2 at pin `45b88269a353ad93744772791feb6d01bc7e1e42`. Its isolated build reported `RELSEG_SIZE 128`, `segment_size 1MB`, `block_size 8192`, maximum data alignment 8, `data_checksums off`, and `shared_buffers 32MB`. All thirteen legs completed with their expected exit statuses. The middle-segment fixture returned 2,282 rows by sequential count and dump, 8,614 by index-only count with `Heap Fetches: 0`, and `pg_class.relpages` changed from 484 to 128 after `VACUUM`. The warm scan raised one `XX001` before its successful `VACUUM` and returned 2,282 rows afterward. In the checkpoint leg, a restart and fresh disk read returned 2,466 rows, including 162 in the partially restored segment. The cluster and sandbox were then removed with `stop clean`.

### Stages

| Stage | What it does | Needs |
| --- | --- | --- |
| `build` | configures 12.2 out of tree under `$SANDBOX/build`, patches the generated `pg_config.h` to `RELSEG_SIZE 128` (a non-default compile-time knob, because `--with-segsize` accepts only whole gigabytes), then `make`/`make install` into `$SANDBOX/install`; skips everything when the binary exists | nothing |
| `cluster` | `initdb -A trust --locale=C --encoding=UTF8` (checksums off by default), writes socket-only settings plus `autovacuum=off` and `fsync=off`, starts on `PORT12`, creates database `zerofile`, records platform facts into `out/platform.txt` | `build` |
| `fixtures` | creates one disposable fixture per leg: `t_small` (one segment) and ten ~4 MB tables `t_mid…t_rsu` (four segments), each with a `PRIMARY KEY` index, deleted rows, `VACUUM` so `_fsm`/`_vm` exist, and `CHECKPOINT` | `cluster` |
| `legs` | runs every `leg_*` stage below | `fixtures` |
| `leg_baseline` | inventories files, row counts, per-segment row distribution, and sizes of `t_small`/`t_mid` | `fixtures` |
| `leg_small` | zeroes the only file of `t_small` after a restart; probes fresh backends | `fixtures` |
| `leg_mid` | zeroes segment 1 of `t_mid`, fresh backend | `fixtures` |
| `leg_midwarm` | zeroes segment 1 of `t_midw` inside a session that had opened all segments via `EXPLAIN` | `fixtures` |
| `leg_last` | zeroes the last segment of `t_last` | `fixtures` |
| `leg_first` | zeroes segment 0 of `t_first` | `fixtures` |
| `leg_delfirst` | deletes segment 0 of `t_delfirst` outright | `fixtures` |
| `leg_delmid` | deletes segment 1 of `t_delmid` outright | `fixtures` |
| `leg_fsm` | zeroes the `_fsm` fork of `t_fsm`; fresh and warmed backends | `fixtures` |
| `leg_vm` | zeroes the `_vm` fork of `t_vm`; fresh and warmed backends | `fixtures` |
| `leg_regrow` | zeroes segment 1 of `t_grow`, shows an FSM-steered `INSERT` error, then a page-sized `INSERT` that regrows over the hole | `fixtures` |
| `leg_resurrect` | dirties buffers, zeroes a segment and `_fsm`, then `CHECKPOINT` regrows the files; a restart forces the following probes to read disk | `fixtures` |
| `leg_restart` | zeroes a middle segment, restarts the cluster, and probes the first backend afterwards | `fixtures` |
| `report` | lists the result files | any |
| `stop` | stops the owned cluster and verifies no `postmaster.pid`, process naming `$DATA`, socket, or listener on `PORT12` | none |
| `clean` | deletes `$SANDBOX`, refusing any path outside `.wiki-runtime/tmp/` | `stop` |

Every fixture object is disposable: the script creates and drops its own database and tables. The leg stages mutate their fixtures and truncate cluster files; re-run `fixtures` before re-running a single leg on its own.

The settings that differ from PostgreSQL 12 defaults are explicit in the script:

| Setting or state | Pinned default | Script value and purpose |
|---|---|---|
| `RELSEG_SIZE` | 131,072 blocks (1 GB with 8 kB blocks) | 128 blocks (1 MB), patched only in the out-of-tree generated header, to fit a four-segment table in a small fixture. [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L295) [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864) |
| `autovacuum` and `fsync` | Both on | Both off in this disposable cluster's `postgresql.conf`. Disabling autovacuum prevents an unexpected background scan of the damaged fixtures; disabling fsync avoids durability cost but does not select the read branches measured here. [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1424-L1434) [guc.c#fsync](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1132) |
| `shared_buffers` | Compiled boot value 8 MB; `initdb` may write a different cluster configuration | 32 MB in this cluster's `postgresql.conf`, giving pages room to remain cached for the warm-buffer cases. [guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2163) |
| Planner enables | Index, index-only, and bitmap scans are enabled | `probe_count` sets all three off only in its query, forcing the measured sequential count. Other probes use the fixture's chosen plan, verified by `EXPLAIN`. [guc.c#scan-enables](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L894-L922) |
| `statement_timeout` / `lock_timeout` | Both 0 (disabled) | `PGOPTIONS` sets 10 minutes / 5 seconds for script connections so a stalled probe fails. They bound execution or lock waits and do not change the storage path on successful probes. [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2401) |
| Text-column storage | `text` uses EXTENDED storage | `leg_regrow` changes `pad` to PLAIN before inserting page-sized values. This keeps those values inline, too large for old FSM free-space hints, and sends insertion to the extension path. [pg_type.dat#text](../../../../raw/postgres-12/src/include/catalog/pg_type.dat#L81-L88) [hio.c#FSM-target](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L370-L423) |

`listen_addresses = ''` and `unix_socket_directories` isolate the cluster; their pinned defaults are `localhost` and `DEFAULT_PGSOCKET_DIR`. The port is 55440 instead of the compiled 5432. These settings are written before the first start and read at postmaster startup; `autovacuum` and `fsync` are reload settings, while the timeouts are user settings. [guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936-L3963) [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177-L2188) [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2401)

### The script

```bash
#!/usr/bin/env bash
#
# zerofile_v12.sh - the measurement script for the wiki page
# wiki/v12/questions/storage-and-vacuum/zeroed-relation-file.md
#
# It builds the pinned PostgreSQL 12 checkout out of tree with a shortened
# RELSEG_SIZE so a multi-segment heap fits in a few megabytes, starts an
# isolated cluster, and measures what SELECT, VACUUM and pg_dump do when a
# relation's segment or fork files are truncated to zero bytes underneath
# the server, with data checksums disabled.
#
# The build is NON-DEFAULT in one compile-time knob: RELSEG_SIZE is patched
# from the pin's default 1 GB (131072 blocks) to 128 blocks (1 MB) in the
# generated pg_config.h of the out-of-tree build, because configure's
# --with-segsize accepts only whole gigabytes in PostgreSQL 12.  The
# measured code paths (mdnblocks, _mdfd_getseg, mdread, the buffer read
# modes) are written against RELSEG_SIZE as a constant and do not change
# behavior with its value; the cluster's own "SHOW segment_size" output is
# recorded so the substitution is visible in the results.
#
# Every fixture object is disposable.  The script creates and drops its own
# database and tables; do not point it at a database anyone cares about.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/zf12-run.sh              # default stages
#   bash .wiki-runtime/tmp/zf12-run.sh fixtures legs # re-run fixtures
#   bash .wiki-runtime/tmp/zf12-run.sh leg_mid       # one fresh leg
#   bash .wiki-runtime/tmp/zf12-run.sh stop clean    # tear down
#
# Stages: build cluster fixtures legs
#         leg_baseline leg_small leg_mid leg_midwarm leg_last leg_first
#         leg_delfirst leg_delmid leg_fsm leg_vm leg_regrow leg_resurrect
#         leg_restart report stop clean
#
# Environment: PORT12 JOBS RELSEG_BLOCKS ROWS_PER_SEG
set -uo pipefail

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/zf12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55440}"
JOBS="${JOBS:-8}"

# RELSEG_SIZE for the patched build, in blocks.  The pin's default is 131072
# (1 GB at BLCKSZ 8192); 128 gives 1 MB segments so a ~4 MB table spans four
# files.  ROWS_PER_SEG estimates how many fixture rows fit in one segment.
RELSEG_BLOCKS="${RELSEG_BLOCKS:-128}"
ROWS_PER_SEG="${ROWS_PER_SEG:-2400}"

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; DATA="$SANDBOX/data"
OUT="$SANDBOX/out"; SQLD="$SANDBOX/sql"; SOCK="$SANDBOX/sock"; BIN="$INST/bin"
DB=zerofile
export PGPORT="$PORT12" PGHOST="$SOCK"

# This script owns exactly one short sandbox name.  Never adopt a pre-existing
# cluster or delete a directory that this script did not create.
own_sandbox() {
  case "$WIKI_ROOT" in
    */postgres-llm-wiki) ;;
    *) die "run from the postgres-llm-wiki repository root" ;;
  esac
  [ -f "$WIKI_ROOT/AGENTS.md" ] || die "repository root not found"
  [ -f "$SANDBOX/.zerofile-owned" ] || die "sandbox is not owned by this script: $SANDBOX"
  [ "$(cat "$SANDBOX/.zerofile-owned")" = "$PIN12 $WIKI_ROOT" ] || die "sandbox owner/pin mismatch"
}
source_pin() {
  [ "$(git -C "$SRC12" rev-parse HEAD)" = "$PIN12" ] || die "PostgreSQL 12 checkout is not at $PIN12"
  [ -z "$(git -C "$SRC12" status --porcelain --untracked-files=no)" ] || die "PostgreSQL 12 checkout has tracked changes"
}
tag_sql() {
  printf '%s\n' "$1" | sed -E 's/(^|;[[:space:]]*)(SELECT|INSERT|DELETE|UPDATE|CREATE|DROP|ALTER|VACUUM|CHECKPOINT|EXPLAIN|SET)([[:space:]])/\1\2 \/\* wiki_zerofile_v12 \*\/\3/g'
}

# statement_timeout and lock_timeout are PGC_USERSET in PostgreSQL 12; they
# travel through PGOPTIONS so they also cover VACUUM, which a multi-statement
# psql -c would wrap in one implicit transaction.
export PGOPTIONS="-c statement_timeout=10min -c lock_timeout=5s"

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# -X ignores ~/.psqlrc; every psql call sets ON_ERROR_STOP.  Probes compare
# the result with an explicit expected success or SQLSTATE.
q()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -c "$(tag_sql "$1")" > /dev/null || die "failed: $1"; }
s()  { "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$DB" -c "$(tag_sql "$1")"; }

# probe_* helpers run one statement in a fresh backend, print its output or
# the server's error, and finish with rc=<psql exit status>.  VERBOSITY
# verbose exposes the SQLSTATE of each error.
probe_sql() { # probe_sql <label> <sql> [success|SQLSTATE]
  local output rc expected=${3:-success}
  printf -- '--- %s\n' "$1"
  output=$("$BIN/psql" -X -v ON_ERROR_STOP=1 -v VERBOSITY=verbose -d "$DB" -c "$(tag_sql "$2")" 2>&1)
  rc=$?
  printf '%s\nrc=%s\n' "$output" "$rc"
  if [ "$expected" = success ]; then
    [ "$rc" -eq 0 ] || die "unexpected SQL failure: $1"
  else
    [ "$rc" -ne 0 ] || die "expected $expected but got success: $1"
    if [ "$expected" = error ]; then
      printf '%s\n' "$output" | grep -q 'ERROR:' || die "no server error: $1"
    else
      printf '%s\n' "$output" | grep -q "ERROR:  $expected:" \
        || die "expected $expected but got rc=$rc: $1"
    fi
  fi
}
probe_count() { # probe_count <table> [success|SQLSTATE]: literal full scan
  probe_sql "seq count(*) on $1" \
    "SET enable_indexscan=off; SET enable_bitmapscan=off; SET enable_indexonlyscan=off; SELECT count(*) FROM $1" "${2:-success}"
}
probe_idx() { # probe_idx <table> <id> [success|SQLSTATE]
  probe_sql "index fetch id=$2 on $1" "SELECT id, left(pad,4) FROM $1 WHERE id = $2" "${3:-success}"
}
probe_vac() { probe_sql "VACUUM $1" "VACUUM $1" "${2:-success}"; }
probe_ioscount() { # probe_ioscount <table> [success|SQLSTATE]
  probe_sql "plan of count(id) on $1" "EXPLAIN (COSTS OFF) SELECT count(id) FROM $1" "${2:-success}"
  probe_sql "analyze count(id) on $1" "EXPLAIN (ANALYZE, COSTS OFF, BUFFERS) SELECT count(id) FROM $1" "${2:-success}"
  probe_sql "count(id) on $1" "SELECT count(id) FROM $1" "${2:-success}"
}
probe_size() {
  probe_sql "size functions on $1" \
    "SELECT pg_relation_size('$1') AS main_bytes, pg_relation_size('$1','fsm') AS fsm_bytes, pg_relation_size('$1','vm') AS vm_bytes, (SELECT relpages FROM pg_class WHERE oid = '$1'::regclass) AS relpages"
}
probe_dump() { # probe_dump <table> [success|error]
  local rc expected=${2:-success}
  printf -- '--- pg_dump -t %s\n' "$1"
  "$BIN/pg_dump" -h "$SOCK" -p "$PORT12" -d "$DB" -t "$1" -f "$OUT/dump_$1.sql" \
      2> "$OUT/dump_$1.err"
  rc=$?
  printf 'rc=%s\n' "$rc"
  if [ "$expected" = success ]; then
    [ "$rc" -eq 0 ] || die "unexpected pg_dump failure on $1"
  else
    [ "$rc" -ne 0 ] || die "expected pg_dump failure on $1"
  fi
  [ -s "$OUT/dump_$1.err" ] && sed 's/^/stderr: /' "$OUT/dump_$1.err"
  if [ -f "$OUT/dump_$1.sql" ]; then
    printf 'dumped rows: %s\n' \
      "$(sed -n '/^COPY /,/^\\\.$/p' "$OUT/dump_$1.sql" | sed '1d;$d' | wc -l | tr -d '[:space:]')"
  fi
}

# relfile <table> <segno-or-fork>: absolute path of one on-disk file.
# segno 0 is the first segment; "_fsm"/"_vm" name the side forks.
relfile() {
  local p="$DATA/$(s "SELECT pg_relation_filepath('$1')")"
  case "$2" in
    0) ;;
    _*) p="$p$2" ;;
    *)  p="$p.$2" ;;
  esac
  printf '%s\n' "$p"
}
lsrel() { ls -l "$DATA/$(s "SELECT pg_relation_filepath('$1')")"* 2>/dev/null; }

# pick_id <table> <segno>: one tuple id whose heap block lives in that segment.
pick_id() {
  s "SELECT min(id) FROM $1 \
     WHERE (ctid::text::point)[0]::int >= $2 * $RELSEG_BLOCKS \
       AND (ctid::text::point)[0]::int < ($2 + 1) * $RELSEG_BLOCKS"
}

restart_cluster() {
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w -t 60 -m fast restart \
    >> "$OUT/restarts.log" 2>&1 || die "restart failed"
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
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.zerofile-owned"
  fi
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  if [ -x "$BIN/postgres" ]; then
    [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 $RELSEG_BLOCKS" ] \
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
  # patched to a 1 MB segment instead of building a >1 GB fixture.
  sed "s/^#define RELSEG_SIZE .*/#define RELSEG_SIZE $RELSEG_BLOCKS/" \
      "$BUILD/src/include/pg_config.h" > "$BUILD/src/include/pg_config.h.new" \
    || die "could not patch RELSEG_SIZE"
  mv "$BUILD/src/include/pg_config.h.new" "$BUILD/src/include/pg_config.h"
  sed -n '/^#define RELSEG_SIZE /{p;q;}' "$BUILD/src/include/pg_config.h" | tee "$OUT/relseg_size.txt"
  [ "$(sed -n 's/^#define RELSEG_SIZE[[:space:]]*//p' "$OUT/relseg_size.txt")" = "$RELSEG_BLOCKS" ] \
    || die "RELSEG_SIZE patch did not take"
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { keep_build_logs; grep 'error:' "$BUILD/make.log" | head -n 3 >&2
         die "make failed, see $OUT/make.log"; }
  keep_build_logs
  printf '%s %s\n' "$PIN12" "$RELSEG_BLOCKS" > "$SANDBOX/.build-spec"
  note "$("$BIN/postgres" --version)"
}

# ------------------------------------------------------------- cluster -------
# Cluster settings, all written before the first start:
#   listen_addresses, unix_socket_directories, port, shared_buffers  PGC_POSTMASTER
#   autovacuum, fsync                                                PGC_SIGHUP
# autovacuum is off so no background worker touches a truncated relation;
# fsync is off because the cluster is disposable; data checksums are off by
# default and verified with SHOW data_checksums = off.
stage_cluster() {
  say "isolated 12.2 cluster, socket-only, port label $PORT12"
  own_sandbox
  source_pin
  [ "$(cat "$SANDBOX/.build-spec" 2>/dev/null)" = "$PIN12 $RELSEG_BLOCKS" ] || die "build specification mismatch"
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
        printf "shared_buffers = 32MB\n"
        printf "autovacuum = off\n"
        printf "fsync = off\n"
      } >> "$DATA/postgresql.conf"
    fi
    "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w -t 60 start \
      > /dev/null 2>&1 || die "pg_ctl start failed, see $OUT/server.log"
    printf '%s\n' "$DATA" > "$SANDBOX/.started"
  fi
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d postgres \
    -c "DROP /* wiki_zerofile_v12 */ DATABASE IF EXISTS $DB" -c "CREATE /* wiki_zerofile_v12 */ DATABASE $DB" \
    || die "could not create database $DB"
  {
    uname -a
    "$BIN/postgres" --version
    printf 'RELSEG_SIZE from build: %s\n' "$(cat "$OUT/relseg_size.txt" 2>/dev/null)"
    s "SHOW segment_size"
    s "SHOW block_size"
    "$BIN/pg_controldata" "$DATA" | sed -n '/^Maximum data alignment:/p'
    s "SHOW data_checksums"
    s "SHOW shared_buffers"
  } > "$OUT/platform.txt"
  cat "$OUT/platform.txt" >&2
}

# ------------------------------------------------------------ fixtures -------
# mk_seg_table <name> <total-rows>: a logged, indexed heap wider than three
# RELSEG_SIZE segments, then DELETE + VACUUM so the _fsm and _vm forks exist.
mk_seg_table() {
  local t=$1 rows=$2
  q "DROP TABLE IF EXISTS $t"
  q "CREATE TABLE $t (id bigserial PRIMARY KEY, pad text NOT NULL)"
  q "INSERT INTO $t (pad) SELECT repeat('x',400) FROM generate_series(1,$rows)"
  q "DELETE FROM $t WHERE id % 101 = 0"
  q "VACUUM $t"
  q "CHECKPOINT"
  [ -f "$(relfile "$t" 1)" ] || die "$t did not reach a second segment"
  [ -f "$(relfile "$t" _fsm)" ] || die "$t has no _fsm fork"
  [ -f "$(relfile "$t" _vm)" ]  || die "$t has no _vm fork"
}
# mk_small_table <name>: one-segment heap with _fsm/_vm forks.
mk_small_table() {
  local t=$1
  q "DROP TABLE IF EXISTS $t"
  q "CREATE TABLE $t (id bigserial PRIMARY KEY, pad text NOT NULL)"
  q "INSERT INTO $t (pad) SELECT repeat('x',400) FROM generate_series(1,200)"
  q "DELETE FROM $t WHERE id % 10 = 0"
  q "VACUUM $t"
  q "CHECKPOINT"
  [ ! -f "$(relfile "$t" 1)" ] || die "$t unexpectedly has a second segment"
  [ -f "$(relfile "$t" _fsm)" ] || die "$t has no _fsm fork"
  [ -f "$(relfile "$t" _vm)" ]  || die "$t has no _vm fork"
}
stage_fixtures() {
  say "disposable fixture tables"
  mk_small_table t_small
  local t
  for t in t_mid t_midw t_last t_first t_delfirst t_delmid t_fsm t_vm t_grow t_rest t_rsu; do
    mk_seg_table "$t" $((ROWS_PER_SEG * 3 + 1500))
  done
  note "fixtures loaded"
}

# ---------------------------------------------------------------- legs -------
# Common probe battery for a table whose files were just altered, run in a
# fresh backend: literal full-table read, VACUUM, catalog size, pg_dump.
fresh_battery() { # fresh_battery <table> [success|SQLSTATE]
  probe_count "$1" "${2:-success}"
  probe_ioscount "$1" "${2:-success}"
  probe_vac "$1" "${2:-success}"
  probe_size "$1"
  if [ "${2:-success}" = success ]; then
    probe_dump "$1"
  else
    probe_dump "$1" error
  fi
}

stage_leg_baseline() {
  say "leg baseline: file inventory of the fixtures"
  {
    for t in t_small t_mid; do
      printf -- '--- %s files\n' "$t"; lsrel "$t"
      probe_sql "$t row count" "SELECT count(*) FROM $t"
      probe_sql "$t blocks per segment" \
        "SELECT (ctid::text::point)[0]::int / $RELSEG_BLOCKS AS seg, count(*), min(id), max(id) FROM $t GROUP BY 1 ORDER BY 1"
      probe_size "$t"
    done
  } > "$OUT/leg_baseline.txt" 2>&1
  cat "$OUT/leg_baseline.txt" >&2
}

# Only file of a small table -> 0 bytes.  Expected: scans read an empty
# relation with no error; only out-of-band block access (the index) errors.
stage_leg_small() {
  say "leg small: zero the only file of t_small while running"
  local f; f=$(relfile t_small 0)
  restart_cluster
  : > "$f" || die "truncate failed"
  {
    printf 'zeroed file: %s\n' "$f"; lsrel t_small
    fresh_battery t_small
    probe_idx t_small 5 XX001
  } > "$OUT/leg_small.txt" 2>&1
  cat "$OUT/leg_small.txt" >&2
}

# Middle segment -> 0 bytes, next backend is fresh.  Expected: no error on
# SELECT/VACUUM/pg_dump; the relation silently ends at the truncated segment
# while pg_relation_size still counts the later files.  Only index fetches
# into the hole report errors, with two different messages.
stage_leg_mid() {
  say "leg mid: zero segment 1 of t_mid, fresh backend"
  local f; f=$(relfile t_mid 1)
  local id1 id2
  id1=$(pick_id t_mid 1); id2=$(pick_id t_mid 2)
  restart_cluster
  : > "$f" || die "truncate failed"
  {
    printf 'zeroed file: %s\n' "$f"; lsrel t_mid
    fresh_battery t_mid
    probe_idx t_mid "$id1" XX001
    probe_idx t_mid "$id2" error
    probe_sql "pg_class relpages after vacuum" \
      "SELECT relpages, reltuples FROM pg_class WHERE oid = 't_mid'::regclass"
  } > "$OUT/leg_mid.txt" 2>&1
  cat "$OUT/leg_mid.txt" >&2
}

# Middle segment -> 0 bytes, backend already holds every segment open.
# Expected: mdnblocks trusts the open tail segment as full, computes the
# pre-truncation nblocks, and the scan errors on a short read.  Plain VACUUM
# may skip the all-visible hole and update pg_class; the next scan reopens it.
stage_leg_midwarm() {
  say "leg midwarm: zero segment 1 of t_midw, warmed backend"
  local f expected_first
  expected_first=$(s "SELECT count(*) FROM t_midw WHERE (ctid::text::point)[0]::int < $RELSEG_BLOCKS")
  [ -n "$expected_first" ] || die "could not count the first segment"
  restart_cluster
  f=$(relfile t_midw 1)
  {
    printf '\\echo --- warm session: planner opens all segments\n'
    printf 'SET /* wiki_zerofile_v12 */ enable_indexscan=off; SET /* wiki_zerofile_v12 */ enable_bitmapscan=off; SET /* wiki_zerofile_v12 */ enable_indexonlyscan=off;\n'
    printf 'EXPLAIN /* wiki_zerofile_v12 */ (COSTS OFF) SELECT count(*) FROM t_midw;\n'
    printf "\\! : > %s\n" "$f"
    printf '\\echo --- warm session: scan after zeroing\n'
    printf '\\set ON_ERROR_STOP off\n'
    printf 'SELECT /* wiki_zerofile_v12 */ count(*) FROM t_midw;\n'
    printf '\\set ON_ERROR_STOP on\n'
    printf 'VACUUM /* wiki_zerofile_v12 */ t_midw;\n'
    printf '\\echo --- warm session: second attempt\n'
    printf 'SELECT /* wiki_zerofile_v12 */ count(*) FROM t_midw;\n'
  } > "$SQLD/midwarm.sql"
  {
    printf 'zeroed file: %s\n' "$f"; lsrel t_midw
    "$BIN/psql" -X -v ON_ERROR_STOP=1 -v VERBOSITY=verbose -d "$DB" -f "$SQLD/midwarm.sql" > "$OUT/midwarm.psql" 2>&1
    [ "$?" -eq 0 ] || die "warm session failed outside the expected scan error"
    cat "$OUT/midwarm.psql"
    [ "$(grep -c 'ERROR:  XX001:' "$OUT/midwarm.psql")" -eq 1 ] || die "expected exactly one XX001 in warm session"
    grep -q '^VACUUM$' "$OUT/midwarm.psql" || die "warm VACUUM did not complete"
    grep -Eq "^[[:space:]]+$expected_first[[:space:]]*$" "$OUT/midwarm.psql" || die "second warm count did not match the first segment"
    printf -- '--- fresh backend afterwards\n'
    probe_count t_midw
  } > "$OUT/leg_midwarm.txt" 2>&1
  cat "$OUT/leg_midwarm.txt" >&2
}

# Last segment -> 0 bytes.  In the fresh-backend fixture, the last open
# segment is remeasured, so size-bounded scans see the shorter relation.
# A block-directed fetch missing shared buffers hits the short-read error.
stage_leg_last() {
  say "leg last: zero the final segment of t_last"
  local nseg f idl
  nseg=$(ls "$DATA/$(s "SELECT pg_relation_filepath('t_last')")".* 2>/dev/null | wc -l | tr -d '[:space:]')
  [ "$nseg" -ge 1 ] || die "t_last has no numbered segments"
  f=$(relfile t_last "$nseg")
  idl=$(pick_id t_last "$nseg")
  restart_cluster
  : > "$f" || die "truncate failed"
  {
    printf 'zeroed file: %s (segment %s)\n' "$f" "$nseg"; lsrel t_last
    fresh_battery t_last
    probe_idx t_last "$idl" XX001
  } > "$OUT/leg_last.txt" 2>&1
  cat "$OUT/leg_last.txt" >&2
}

# First segment of a multi-segment table -> 0 bytes.  Expected: the relation
# reports 0 blocks; everything after it is unreachable by scans, silently.
stage_leg_first() {
  say "leg first: zero segment 0 of t_first"
  local f id2
  f=$(relfile t_first 0); id2=$(pick_id t_first 2)
  restart_cluster
  : > "$f" || die "truncate failed"
  {
    printf 'zeroed file: %s\n' "$f"; lsrel t_first
    fresh_battery t_first
    probe_idx t_first "$id2" error
  } > "$OUT/leg_first.txt" 2>&1
  cat "$OUT/leg_first.txt" >&2
}

# Contrast leg: first segment deleted outright (not zeroed).  Expected:
# operations that open the heap error with "could not open file".
stage_leg_delfirst() {
  say "leg delfirst: delete segment 0 of t_delfirst"
  restart_cluster
  local f; f=$(relfile t_delfirst 0)
  rm -f "$f" || die "rm failed"
  {
    printf 'deleted file: %s\n' "$f"; lsrel t_delfirst
    fresh_battery t_delfirst 58P01
  } > "$OUT/leg_delfirst.txt" 2>&1
  cat "$OUT/leg_delfirst.txt" >&2
}

# Contrast leg: middle segment deleted outright.  Expected: mdnblocks stops
# at the first missing segment, so scans silently truncate there; index
# fetches past the gap get "could not open file" with SQLSTATE 58P01.
stage_leg_delmid() {
  say "leg delmid: delete segment 1 of t_delmid"
  local f id2
  f=$(relfile t_delmid 1); id2=$(pick_id t_delmid 2)
  restart_cluster
  rm -f "$f" || die "rm failed"
  {
    printf 'deleted file: %s\n' "$f"; lsrel t_delmid
    fresh_battery t_delmid
    probe_idx t_delmid "$id2" 58P01
  } > "$OUT/leg_delmid.txt" 2>&1
  cat "$OUT/leg_delmid.txt" >&2
}

# Zero the _fsm fork.  Expected: a fresh backend treats it as absent and
# recreates it; a backend that cached the old fork size reads past EOF only
# if the fork page is also absent from shared buffers.
stage_leg_fsm() {
  say "leg fsm: zero t_fsm's _fsm fork"
  restart_cluster
  local f; f=$(relfile t_fsm _fsm)
  : > "$f" || die "truncate failed"
  {
    printf 'zeroed file: %s\n' "$f"; lsrel t_fsm
    fresh_battery t_fsm
    probe_sql "INSERT after zeroed _fsm (fresh backend)" \
      "INSERT INTO t_fsm (pad) VALUES ('x')"
    lsrel t_fsm
    {
      printf '\\echo --- warm session: first INSERT caches the fsm size\n'
      printf "INSERT /* wiki_zerofile_v12 */ INTO t_fsm (pad) VALUES ('warm');\n"
      printf "\\! : > %s\n" "$f"
      printf '\\echo --- warm session: bulk INSERT forces an FSM consult\n'
      printf "INSERT /* wiki_zerofile_v12 */ INTO t_fsm (pad) SELECT repeat('y',400) FROM generate_series(1,50);\n"
      printf '\\echo --- warm session: retry and VACUUM in same session\n'
      printf "INSERT /* wiki_zerofile_v12 */ INTO t_fsm (pad) SELECT repeat('y',400) FROM generate_series(1,50);\n"
      printf 'VACUUM /* wiki_zerofile_v12 */ t_fsm;\n'
    } > "$SQLD/fsm_warm.sql"
    printf -- '--- warm session output\n'
    "$BIN/psql" -X -v ON_ERROR_STOP=1 -v VERBOSITY=verbose -d "$DB" -f "$SQLD/fsm_warm.sql" 2>&1
    [ "$?" -eq 0 ] || die "warm FSM session failed"
    probe_sql "INSERT after warm errors (fresh backend)" \
      "INSERT INTO t_fsm (pad) VALUES ('fresh')"
  } > "$OUT/leg_fsm.txt" 2>&1
  cat "$OUT/leg_fsm.txt" >&2
}

# Zero the _vm fork.  Expected: a fresh backend treats every page as
# not-all-visible and reads the heap anyway; a warmed backend's index-only
# scan and VACUUM read the cached vm page rather than the file.
stage_leg_vm() {
  say "leg vm: zero t_vm's _vm fork"
  restart_cluster
  local f; f=$(relfile t_vm _vm)
  : > "$f" || die "truncate failed"
  {
    printf 'zeroed file: %s\n' "$f"; lsrel t_vm
    fresh_battery t_vm
    probe_sql "index-only count(id) after zeroed _vm (fresh)" \
      "SELECT count(id) FROM t_vm"
    {
      printf '\\echo --- warm session: VACUUM rebuilds the vm fork\n'
      printf 'VACUUM /* wiki_zerofile_v12 */ t_vm;\n'
      printf '\\echo --- warm session: index-only scan caches the vm size\n'
      printf 'EXPLAIN /* wiki_zerofile_v12 */ (COSTS OFF) SELECT count(id) FROM t_vm;\n'
      printf 'SELECT /* wiki_zerofile_v12 */ count(id) FROM t_vm;\n'
      printf "\\! : > %s\n" "$f"
      printf '\\echo --- warm session: index-only scan after re-zeroing\n'
      printf 'SELECT /* wiki_zerofile_v12 */ count(id) FROM t_vm;\n'
      printf '\\echo --- warm session: VACUUM\n'
      printf 'VACUUM /* wiki_zerofile_v12 */ t_vm;\n'
    } > "$SQLD/vm_warm.sql"
    printf -- '--- warm session output\n'
    "$BIN/psql" -X -v ON_ERROR_STOP=1 -v VERBOSITY=verbose -d "$DB" -f "$SQLD/vm_warm.sql" 2>&1
    [ "$?" -eq 0 ] || die "warm VM session failed"
  } > "$OUT/leg_vm.txt" 2>&1
  cat "$OUT/leg_vm.txt" >&2
}

# After a middle segment is zeroed, a normal INSERT first errors: the
# surviving _fsm nominates a free page inside the hole.  An INSERT of tuples
# larger than any recorded free space finds no FSM candidate and extends the
# relation instead, rewriting the zeroed segment block by block and then
# appending at the end of the surviving chain.
stage_leg_regrow() {
  say "leg regrow: zero segment 1 of t_grow, then insert to extend"
  restart_cluster
  local f; f=$(relfile t_grow 1)
  : > "$f" || die "truncate failed"
  {
    printf 'zeroed file: %s\n' "$f"; lsrel t_grow
    probe_count t_grow
    probe_sql "bulk INSERT routed by the surviving fsm" \
      "INSERT INTO t_grow (pad) SELECT repeat('y',400) FROM generate_series(1,$((ROWS_PER_SEG * 2)))" XX001
    printf -- '--- phase 2: tuples bigger than any recorded free space\n'
    probe_sql "mark pad as un-toastable" \
      "ALTER TABLE t_grow ALTER pad SET STORAGE PLAIN"
    probe_sql "bulk INSERT of one-page tuples" \
      "INSERT INTO t_grow (pad) SELECT repeat('y',7500) FROM generate_series(1,300)"
    probe_count t_grow
    probe_sql "blocks per segment after regrow" \
      "SELECT (ctid::text::point)[0]::int / $RELSEG_BLOCKS AS seg, count(*), min(id), max(id) FROM t_grow GROUP BY 1 ORDER BY 1"
    printf -- '--- files after regrow\n'; lsrel t_grow
    probe_size t_grow
    probe_idx t_grow "$(s "SELECT max(id) FROM t_grow WHERE id < 8701")"
  } > "$OUT/leg_regrow.txt" 2>&1
  cat "$OUT/leg_regrow.txt" >&2
}

# Truncating a file does not purge shared buffers: a CHECKPOINT flushes any
# dirty buffered pages back into the file at their block offsets, so a
# "zeroed" file can regrow with stale content, and the hole before the first
# written block reads back as all-zero pages.
stage_leg_resurrect() {
  say "leg resurrect: dirty buffers rewrite a zeroed file at CHECKPOINT"
  local fseg1 ffsm id1
  fseg1=$(relfile t_rsu 1); ffsm=$(relfile t_rsu _fsm); id1=$(pick_id t_rsu 1)
  {
    printf -- '--- dirty a heap page in segment 1 and the fsm\n'
    probe_sql "UPDATE one row in segment 1" "UPDATE t_rsu SET pad='q' WHERE id = $id1"
    probe_sql "INSERT rows (dirties the fsm)" \
      "INSERT INTO t_rsu (pad) SELECT repeat('z',400) FROM generate_series(1,30)"
    printf 'zeroing %s and %s\n' "$fseg1" "$ffsm"
    : > "$fseg1"; : > "$ffsm"
    lsrel t_rsu
    probe_sql "CHECKPOINT" "CHECKPOINT"
    printf -- '--- files after the checkpoint\n'; lsrel t_rsu
    restart_cluster
    printf -- '--- disk reread after restart (shared buffers discarded)\n'
    probe_count t_rsu
    probe_sql "blocks per segment after resurrection" \
      "SELECT (ctid::text::point)[0]::int / $RELSEG_BLOCKS AS seg, count(*), min(id), max(id) FROM t_rsu GROUP BY 1 ORDER BY 1"
    probe_idx t_rsu "$id1"
  } > "$OUT/leg_resurrect.txt" 2>&1
  cat "$OUT/leg_resurrect.txt" >&2
}

# Restart does not repair a zeroed middle segment: the first backend after
# the restart measures the shortened relation exactly like any other fresh
# backend.
stage_leg_restart() {
  say "leg restart: truncate mid segment, restart, probe"
  local f id2
  f=$(relfile t_rest 1); id2=$(pick_id t_rest 2)
  : > "$f" || die "truncate failed"
  restart_cluster
  {
    printf 'zeroed file: %s (server restarted after truncate)\n' "$f"
    fresh_battery t_rest
    probe_idx t_rest "$id2" error
  } > "$OUT/leg_restart.txt" 2>&1
  cat "$OUT/leg_restart.txt" >&2
}

stage_legs() {
  local l
  for l in leg_baseline leg_small leg_mid leg_midwarm leg_last leg_first \
           leg_delfirst leg_delmid leg_fsm leg_vm leg_regrow leg_resurrect \
           leg_restart; do
    "stage_$l"
  done
}

# ------------------------------------------------------------- teardown ------
stage_report() {
  own_sandbox
  say "result files"
  ls -l "$OUT" | sed 's/^/   /' >&2
  note "read platform.txt first, then leg_*.txt"
}

stage_stop() {
  say "stop cluster"
  [ -e "$SANDBOX" ] || { note "sandbox absent"; return 0; }
  own_sandbox
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    [ -f "$SANDBOX/.started" ] || die "refusing to stop an unowned server"
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop || die "pg_ctl stop failed"
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
[ $# -eq 0 ] && stages="build cluster fixtures legs report"
for st in $stages; do
  case "$(type -t "stage_$st" 2>/dev/null)" in
    function) "stage_$st" ;;
    *) die "unknown stage: $st" ;;
  esac
done
```

## Context Reviewed

- `src/backend/storage/smgr/md.c` — the file/segment layer: `mdnblocks`'s early return, `_mdfd_getseg`'s missing/short-segment errors, `mdread`'s short-read error, `mdopen`'s first-segment failure, `mdextend`/`mdwrite` offsets, `mdexists`.
- `src/backend/storage/buffer/bufmgr.c` — the `RBM_*` read modes, the `P_NEW → smgrnblocks` substitution, and the `smgrread` + `PageIsVerified` sequence that `RBM_ZERO_ON_ERROR` can and cannot mask.
- `src/backend/storage/page/bufpage.c` — `PageIsVerified`, which accepts all-zero pages before any checksum comparison.
- `src/backend/storage/freespace/freespace.c` — `fsm_readbuf`'s `smgr_fsm_nblocks` gate and `fsm_extend`'s create-if-missing path.
- `src/backend/access/heap/visibilitymap.c` — `vm_readbuf`, same pattern.
- `src/backend/access/heap/heapam.c` — `initscan`'s `rs_nblocks` bound for sequential scans.
- `src/backend/access/heap/hio.c` — `RelationGetBufferForTuple`: the FSM target, the last-page fallback, and the `P_NEW` extension.
- `src/backend/access/heap/vacuumlazy.c` — `lazy_scan_heap`'s `nblocks` bound, the visibility-map page skip, `PageIsNew`/`PageIsEmpty` handling, and `count_nondeletable_pages`.
- `src/backend/commands/vacuum.c` — `vac_update_relstats`'s dirty-field checks and `vac_estimate_reltuples`.
- `src/backend/commands/copy.c` — `CopyTo` uses `table_beginscan`, the same sequential read a `SELECT` uses.
- `src/bin/pg_dump/pg_dump.c` — `dumpTableData_copy` issues `COPY ... TO stdout` and aborts on a failed COPY.
- `src/backend/utils/adt/dbsize.c` — `calculate_relation_size` sums `stat().st_size` over existing segments, stopping at the first missing name.
- `src/backend/executor/nodeIndexonlyscan.c` — the visibility-map check that lets an index-only scan skip the heap entirely.
- `src/backend/access/heap/heapam_handler.c` — `heapam_index_fetch_tuple`'s buffer read before heap tuple search, and `heapam_estimate_rel_size`'s `RelationGetNumberOfBlocks` call that opens all segments.
- `src/include/storage/smgr.h` — `SMgrRelationData`'s per-fork segment descriptors and cached fork block counts.
- `src/backend/utils/misc/guc.c`, `src/bin/initdb/initdb.c`, and `src/include/catalog/pg_type.dat` — defaults for this measurement's settings, checksums, and text storage.
- `src/backend/commands/explain.c` — index-only `Heap Fetches` reported by the analyzed plan.
- `src/backend/utils/error/elog.c` — `errcode_for_file_access`'s errno-to-SQLSTATE mapping, including the `ENOENT → 58P01` and default → `XX000` branches.
- `src/backend/storage/smgr/smgr.c` — `smgrtruncate`'s `CacheInvalidateSmgr`, the invalidation that an engine-side truncation sends and an external `truncate` does not.
- `src/backend/utils/cache/relcache.c` — `RelationClearRelation`, which drops the smgr and its open segment list on relcache invalidation.
- `doc/src/sgml/storage.sgml` — fork and segment naming, the default 1 GB segment size, and `pg_relation_filepath`'s scope.
- `configure.in`, `src/include/pg_config.h.in` — `--with-segsize` granularity and the `RELSEG_SIZE` contract.
- `src/test/regress` and `src/test/recovery` — no test truncates a relation file externally; measured evidence comes from the script.

## Evidence Map

| Claim | Evidence |
|---|---|
| `mdnblocks` returns at the first short or missing segment, re-measuring only the last *open* segment | [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755); measured `count(*)` 2,282 after zeroing `.1` |
| Sequential scans, ordinary vacuum, and `COPY TO` use the measured block bound; other conditions can still make them fail | [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L295), [vacuumlazy.c#nblocks](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L549-L556), [copy.c#CopyTo](../../../../raw/postgres-12/src/backend/commands/copy.c#L2114-L2137); measured rc=0 with fewer rows in fresh-backend zeroed-main-fork legs |
| `pg_dump` inherits the COPY outcome, including its failure | [pg_dump.c#dumpTableData_copy](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1826-L1929); measured exit 0 with 2,282 rows, exit 1 on the deleted-first-file leg |
| Short reads error `XX001` unless `zero_damaged_pages` or recovery zeroes them | [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630); measured `read only 0 of 8192 bytes` |
| Reading past a short segment errors `previous segment is only %u blocks`; the SQLSTATE is errno-dependent because this branch makes no failed syscall | [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1219), [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L583-L652); measured `XX000` in this run |
| Open segments are trusted as `RELSEG_SIZE`, so a warm backend computes stale `nblocks` | [md.c#mdnblocks-trust](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L715-L729); measured `XX001` in the warm session and silent 2,282 after the session's own `VACUUM` |
| A missing first segment fails operations that need to open it with `58P01`; `pg_relation_size` itself can return 0 | [md.c#mdopen](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L450-L463), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L331); measured |
| `fsm_readbuf`/`vm_readbuf` use cached fork block counts, measuring them when invalid and extending on demand | [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L552-L605), [visibilitymap.c#vm_readbuf](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L567-L618); measured no error on the fresh-fork legs |
| `RBM_ZERO_ON_ERROR` substitutes for any complete page that fails `PageIsVerified()`, including a bad checksum, but not a short file read | [bufmgr.c#verify-path](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L161), [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) |
| An all-zero page passes verification without a checksum comparison | [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142); the checkpoint leg reread the partly restored segment after restart |
| Dirty buffered pages are written back at block offsets regardless of file truncation | [md.c#mdwrite](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L640-L695); measured `.1` regrew to 335,872 bytes and `_fsm` to 24,576 bytes at `CHECKPOINT` |
| `INSERT` can use an FSM target or extend the table when no target has enough free space | [hio.c#fsm-target](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L370-L423), [hio.c#extend](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L555-L609); measured `XX001` for ordinary tuples and `INSERT 0 300` for page-sized PLAIN values |
| `pg_relation_size` sums `st_size` of every existing segment | [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308); measured 2,916,352 while scans reached 1 MB |
| `VACUUM` can update `relpages`/`reltuples` from the shortened scan, but preserves old values in the all-skipped case and avoids an unchanged catalog write | [vacuumlazy.c#stats-gate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L320-L356), [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1179-L1259); measured `relpages` 484 → 128 |
| An index-only scan with all-visible VM bits can answer without heap reads | [nodeIndexonlyscan.c#vm-check](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L155-L189), [explain.c#Heap-Fetches](../../../../raw/postgres-12/src/backend/commands/explain.c#L1594-L1608); measured `count(id)` 8,614 with `Heap Fetches: 0` versus sequential `count(*)` 2,282 |
| Segments are 1 GB by default and GB-only configurable in 12 | [storage.sgml#segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L233-L243), [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L292) |
| Engine-side truncation invalidates other backends' smgr state; an external truncate does not | [smgr.c#smgrtruncate](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L645-L670), [md.c#mdnblocks-note](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L720-L727); measured warm-backend `XX001` |

## Open Questions

- **Warm-backend `_fsm`/`_vm` short read.** `fsm_readbuf`/`vm_readbuf` can issue `ReadBufferExtended` for a fork block below a *stale cached* `smgr_fsm_nblocks`/`smgr_vm_nblocks`, where `RBM_ZERO_ON_ERROR` does not mask the `mdread` short read. In the measured runs the fork pages stayed in shared buffers, so the warmed sessions never hit the disk and showed no error; whether the `XX001` path is reachable in practice needs a run that evicts the fork page while keeping the stale size cache. [freespace.c#fsm_readbuf-recheck](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L552-L575) [bufmgr.c#verify-path](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925)
- **An isolated all-zero page inside an otherwise full segment.** `PageIsVerified()` accepts an all-zero page, but a standalone page-overwrite scenario was not measured here. The checkpoint leg did reread a partly restored segment after restart; it is a different fixture because dirty buffers wrote selected pages before the reread. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142)
- **TOAST and other relations' files.** This page measured only the ordinary table's heap, `_fsm`, and `_vm` files. Whether a zeroed TOAST file first produces a storage read error or a later missing-chunk error remains unmeasured.
- **`zero_damaged_pages` / recovery interplay.** The default is off. If a superuser enables it, `mdread()` zero-fills a short read instead of throwing `XX001`; recovery takes the same branch. The interaction with subsequent tuple access was not measured here, so this page does not claim a specific SQL result. [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1149-L1161) [md.c#zero-damaged](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L614-L630)

## Source References

- [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1-L1325) — segment naming, `mdnblocks`, `_mdfd_getseg`, `mdread`, `mdwrite`, `mdextend`, `mdopen`, `mdexists`
- [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1-L4402) — `ReadBufferExtended` modes and the `smgrread`/`PageIsVerified` read path
- [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1-L1205) — `PageIsVerified` and the all-zero acceptance
- [freespace.c](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L1-L906) — `fsm_readbuf`, `fsm_extend`, `GetPageWithFreeSpace`, `RecordAndGetPageWithFreeSpace`
- [visibilitymap.c](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L1-L681) — `vm_readbuf`, `vm_extend`, `visibilitymap_test`
- [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1-L9045) — `initscan`, `heapgettup` scan bound
- [hio.c](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L1-L674) — `RelationGetBufferForTuple`, `RelationAddExtraBlocks`
- [vacuumlazy.c](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1-L2356) — `lazy_scan_heap`, page skipping, `count_nondeletable_pages`
- [vacuum.c](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1-L1983) — `vac_update_relstats`, `vac_estimate_reltuples`
- [copy.c](../../../../raw/postgres-12/src/backend/commands/copy.c#L1-L5217) — `CopyTo`
- [pg_dump.c](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1-L18414) — `dumpTableData_copy`
- [dbsize.c](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L1-L1022) — `calculate_relation_size`
- [nodeIndexonlyscan.c](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L1-L737) — index-only scan's visibility-map gate
- [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L1-L2669) — `heapam_index_fetch_tuple`, `heapam_estimate_rel_size`
- [elog.c](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L1-L3481) — `errcode_for_file_access`
- [smgr.c](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L1-L731) — `smgrnblocks`, `smgrtruncate`
- [relcache.c](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1-L6126) — `RelationClearRelation`
- [smgr.h](../../../../raw/postgres-12/src/include/storage/smgr.h#L40-L71) — `SMgrRelationData`
- [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L894-L2401) — defaults and contexts for relevant settings
- [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) — default checksum selection
- [pg_type.dat](../../../../raw/postgres-12/src/include/catalog/pg_type.dat#L81-L88) — text storage default
- [explain.c](../../../../raw/postgres-12/src/backend/commands/explain.c#L1594-L1608) — index-only heap fetch count
- [008_fsm_truncation.pl](../../../../raw/postgres-12/src/test/recovery/t/008_fsm_truncation.pl#L1-L96) — engine-driven FSM truncation test
- [bufmgr.h](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L1-L279) — `RelationGetNumberOfBlocks`
- [storage.sgml](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L1-L1073) — fork/segment naming and defaults
- [configure.in](../../../../raw/postgres-12/configure.in#L1-L2488), [pg_config.h.in](../../../../raw/postgres-12/src/include/pg_config.h.in#L1-L1043) — `--with-segsize`, `RELSEG_SIZE`

## Navigation

- Up: [PostgreSQL 12 wiki](../../index.md) · [Wiki index](../../../index.md)
- Same category: [VACUUM FULL I/O on a near-empty heap (unverified)](vacuum-full-io-on-near-empty-heap.md) · [pgstattuple_approx heap and TOAST bloat (unverified)](pgstattuple-approx-heap-and-toast-bloat.md) · [catalog statistics bloat query (unverified)](catalog-statistics-bloat-query-reliability.md) · [very large shared_buffers (unverified)](very-large-shared-buffers.md)
- Related: [Corruption indicators in logs with checksums disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md) · [Glossary](../../../glossary.md)
