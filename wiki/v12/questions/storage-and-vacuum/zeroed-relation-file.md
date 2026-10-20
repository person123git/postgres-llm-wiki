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
  - [Measured outcomes, by file and by backend state](#measured-outcomes-by-file-and-by-backend-state)
  - [Why scans stay silent: the size check stops early](#why-scans-stay-silent-the-size-check-stops-early)
  - [When errors do appear](#when-errors-do-appear)
  - [The side forks are even quieter](#the-side-forks-are-even-quieter)
  - [Truncating a running server's file does not reliably erase it](#truncating-a-running-servers-file-does-not-reliably-erase-it)
  - [Regrowth and the free space map](#regrowth-and-the-free-space-map)
  - [What the size and statistics functions then report](#what-the-size-and-statistics-functions-then-report)
  - [Deleting the file is a different failure](#deleting-the-file-is-a-different-failure)
  - [Non-default build knob used for the measurement](#non-default-build-knob-used-for-the-measurement)
  - [Data checksums do not change any of this](#data-checksums-do-not-change-any-of-this)
  - [Tests](#tests)
  - [Final causal summary](#final-causal-summary)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
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

**Usually there is no error at all — the table silently looks shorter.** A 12.2 server built from this pin was measured: truncating any one main-fork segment file to zero bytes makes `mdnblocks()` report a smaller relation, so `SELECT` of all rows, `VACUUM`, and `pg_dump` (which runs `COPY ... TO stdout` over the same sequential read) all succeed and simply see fewer rows. Errors appear only on paths that request a block *by number* without trusting the computed end of the relation — an index-driven heap fetch, an `INSERT` steered by a surviving free space map, or a scan inside a backend that had already opened later segment files — and when the *first* file is deleted outright rather than zeroed. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L295) [vacuumlazy.c#lazy_scan_heap-nblocks](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L545-L556) [copy.c#CopyTo-table_beginscan](../../../../raw/postgres-12/src/backend/commands/copy.c#L2114-L2137) [pg_dump.c#dumpTableData_copy](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1826-L1929)

### One premise needs correcting first

Thousands of rows do **not** give a table multiple data files at the default build. A table or index splits into `RELSEG_SIZE`-block [segments](../../../glossary.md#storage-manager) only when it exceeds the segment size, which is 1 GB by default and set at compile time by `--with-segsize` in whole gigabytes; subsequent files are named `filenode.1`, `filenode.2`, and so on. The several files a small table actually has are its [forks](../../../glossary.md#fork): the main fork plus a [free space map](../../../glossary.md#free-space-map) (`_fsm`) and a [visibility map](../../../glossary.md#visibility-map) (`_vm`), and unlogged relations also have an init fork (`_init`). `pg_relation_filepath()` returns only the first segment of the main fork. [storage.sgml#forks](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L199-L243) [configure.in#with-segsize](../../../../raw/postgres-12/configure.in#L279-L292) [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864) [storage.sgml#pg_relation_filepath](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L281-L289)

The measurement below therefore uses a build whose `RELSEG_SIZE` is 128 blocks (1 MB segments), disclosed under [Non-default build knob used for the measurement](#non-default-build-knob-used-for-the-measurement), so that a ~4 MB table spans four segment files.

### Measured outcomes, by file and by backend state

A fresh backend means one that opens the relation's files after the truncation — any new connection, or every backend after a restart. A warm backend already had the relation's segments open when the file was truncated. The fixture was a 8,614-row table spread over four segment files (three full 1 MB segments plus a partial fourth) with a `PRIMARY KEY` index, an `_fsm`, and a `_vm`; `data_checksums` was off, `autovacuum` off.

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

So a zeroed *middle* segment makes `SELECT`, `VACUUM`, and `pg_dump` all succeed while silently dropping the truncated segment **and every segment after it** — the dump is smaller than the data on disk with no warning anywhere.

### When errors do appear

Three mechanisms turn the same zeroed file into an error:

1. **Block-addressed reads inside the hole.** An index scan still carries TIDs pointing at blocks in the truncated or later segments, because the index files are untouched. `heap_hot_search_buffer()` → `ReadBuffer` on a block inside the zeroed segment hits `mdread()`'s short read: `FileRead` returns fewer than `BLCKSZ` bytes and, with `zero_damaged_pages` off and no recovery in progress, raises `ERROR XX001 could not read block %u in file "%s": read only %d of %d bytes` (SQLSTATE `ERRCODE_DATA_CORRUPTED`). A block in a segment *after* the hole instead reaches `_mdfd_getseg()`, which refuses to open it because the previous segment is not `RELSEG_SIZE` long: `XX000 could not open file "%s" (target block %u): previous segment is only %u blocks`. That message's SQLSTATE is `ERRCODE_INTERNAL_ERROR`, since `errcode_for_file_access()` maps a stale `errno` — no syscall actually failed — through its default branch. [heapam_handler.c#heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L135-L160) [md.c#mdread-short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L575-L631) [md.c#_mdfd_getseg-segment-gap](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1137-L1223) [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L583-L652)

2. **A warm backend's stale segment table.** `mdnblocks()` skips re-checking already-open segments: it starts from the last open segment and trusts earlier ones as exactly `RELSEG_SIZE`, because normally only an engine truncation shrinks a file and that path sends a smgr invalidation to every backend. An external `truncate` sends no such invalidation. A backend that had opened all segments (any prior `mdnblocks()` call does this, including the planner's `heapam_estimate_rel_size` during `EXPLAIN`) therefore computes the *pre-truncation* `nblocks`, scans into the hole, and hits the `mdread` short read — the `XX001` error measured above. A `VACUUM` in that session can still succeed, because it skips all-visible pages without reading them, and its `vac_update_relstats()` catalog write invalidates the relcache, so the *next* statement reopens the files and sees the short relation. [md.c#mdnblocks-open-segments](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L715-L729) [smgr.c#smgrtruncate-inval](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L645-L670) [heapam_handler.c#heapam_estimate_rel_size](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L2120-L2130) [vacuumlazy.c#vm-page-skip](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L609-L636) [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1117-L1262) [relcache.c#RelationClearRelation](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L2399-L2429)

3. **A surviving free space map.** The `_fsm` fork is a separate file and keeps its stale entries. An `INSERT` consults it via `GetPageWithFreeSpace()`; if it nominates a page inside the zeroed segment, `ReadBufferBI` on that block hits the same `XX001` short read — measured as `could not read block 129 in file ".1"`. [hio.c#GetPageWithFreeSpace](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L381-L400) [hio.c#target-block-loop](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L403-L433)

Conversely, an **index-only** scan never touches the heap when the visibility map (also untouched) says the pages are all-visible, so `count(id)` returned all 8,614 rows even while the sequential count saw 2,282 — two different answers to "how many rows" from the same truncated table. [nodeIndexonlyscan.c#vm-check](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L161-L189)

### The side forks are even quieter

The `_fsm` and `_vm` readers do not trust the fork file blindly: `fsm_readbuf()` and `vm_readbuf()` first bound every read by `smgrnblocks()` of the fork itself (`smgr_fsm_nblocks`/`smgr_vm_nblocks`), so a zero-length fork looks exactly like a missing one and `ReadBufferExtended` is never issued past EOF. A missing FSM page means "no information", and a missing VM page means "not all-visible" — both degrade to slower but correct behavior: a fresh backend after a zeroed `_vm` answered `count(id)` by falling back to heap fetches, and the next `INSERT` or `VACUUM` recreated the zeroed `_fsm`/`_vm` file via `fsm_extend`/`vm_extend`. The `RBM_ZERO_ON_ERROR` mode they use does *not* cover a short read — it only zeroes a page whose header fails `PageIsVerified` — but the size gate means that path is not reached for a zeroed fork in a fresh backend. [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L538-L607) [visibilitymap.c#vm_readbuf](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L547-L618) [freespace.c#fsm_extend](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L609-L661) [bufmgr.c#read-modes](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L605-L639) [bufmgr.c#zero-on-error-scope](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L881-L925)

### Truncating a running server's file does not reliably erase it

`truncate -s0` shortens the file, but any dirty page still in shared buffers is written back at its block offset by the next buffer flush — measured: after dirtying a heap page in segment 1 and some `_fsm` pages, zeroing both files and running `CHECKPOINT` regrew `filenode.1` to 335,872 bytes (a sparse file: the unwritten prefix reads back as all-zero pages, which `PageIsVerified` accepts) and `_fsm` to 24,576 bytes with its old contents. `mdwrite` writes at `blocknum % RELSEG_SIZE` offsets; there is no check that the file was shortened. So on a live server the "corruption" can partially undo itself, nondeterministically, depending on which buffers happen to be dirty. [md.c#mdwrite](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L633-L695) [bufpage.c#PageIsVerified-all-zero](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L161)

### Regrowth and the free space map

After a middle segment is zeroed, the table can regrow *over* the hole. `mdextend()` places the new block at `blocknum % RELSEG_SIZE` inside whichever segment the block number maps to, so extending past the shortened size rewrites the zeroed `.1` block by block; once it is full again, `mdnblocks()` sees the surviving `.2`/`.3` as continuation and appends go to a new `.4`. Measured with page-sized tuples (bigger than any FSM-recorded free space, so no candidate is nominated): 300 inserted rows refilled `.1`, spread over `.3`, `.4`, and `.5`, and the sequential count jumped back to 6,633 — including 2,281 "lost" rows from `.2` that simply reappeared, and an index fetch of a stale row returned its pre-truncation contents. [md.c#mdextend](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L365-L427) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L731-L755) [hio.c#extend-path](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L545-L617)

### What the size and statistics functions then report

- `pg_relation_size(rel, 'main')` does **not** use `mdnblocks`: `calculate_relation_size()` `stat()`s every existing segment file and sums `st_size`, stopping only at the first missing name — so it still counted the later segments (2,916,352 bytes) while a scan could only reach 1 MB. For a *deleted* middle file it stops at the gap (1,048,576). [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L331)
- `VACUUM` then rewrites `pg_class.relpages`/`reltuples` from the shrunken scan (`vac_update_relstats`), so after one `VACUUM` the catalog itself says the table is 128 pages — matching what scans see, and hiding that the disk still holds more. [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1157-L1262) [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1059-L1116)

### Deleting the file is a different failure

A **missing** file fails differently from a **zero-length** one, because `mdopen()`/`_mdfd_openseg()` cannot open it at all:

- First segment deleted: every access errors at open — `SELECT` → `58P01 could not open file "base/...": No such file or directory`, `VACUUM` the same, `pg_dump` aborts with `Dumping the contents of table ... failed: PQgetResult() failed.` [md.c#mdopen](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L437-L475)
- Middle segment deleted: `mdnblocks()` still stops at the gap (sequential operations stay silent), but an index fetch past it gets `58P01 could not open file "%s" (target block %u): No such file or directory` from `_mdfd_getseg`. [md.c#_mdfd_getseg-open-fail](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1207-L1219)

### Non-default build knob used for the measurement

PostgreSQL 12's `configure` accepts `--with-segsize` only in whole gigabytes (`RELSEG_SIZE = 1024/blocksize * segsize * 1024` blocks), so exercising multi-segment behavior at the default 1 GB needs a >1 GB table. The script instead configures out of tree, then edits the generated `pg_config.h` to `RELSEG_SIZE 128` — a disclosed compile-time override, valid because the code treats `RELSEG_SIZE` as a constant with no behavioral special-casing of its value ("a power-of-2 value is recommended but not absolutely required; changing it requires an initdb", and the cluster here was initdb'd by the same build). Every measured code path (`mdnblocks`, `_mdfd_getseg`, `mdread`, `mdextend`, the buffer modes) branches on *whether a segment is shorter than `RELSEG_SIZE`*, never on the constant's magnitude. The build's `SHOW segment_size` printed `1MB`, recorded in `out/platform.txt`. [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L292) [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864)

### Data checksums do not change any of this

Nothing on these paths verifies page *contents*. A zero-length file produces a short read, which is governed by `zero_damaged_pages`/`InRecovery` rather than checksums; a hole or all-zero region produces a page that `PageIsVerified` accepts even *with* checksums on (the all-zero case returns valid before any comparison). The only checksum-gated diagnostic — the `page verification failed` warning — requires a readable non-zero page with a bad `pd_checksum`, which this scenario never produces. The related page [Corruption Indicators in PostgreSQL 12 Logs With Data Checksums Disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md) catalogs that separate surface. [md.c#mdread-zero-damaged](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L614-L630) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142)

### Tests

The pin's regression and recovery suites carry no test that truncates a relation file externally; `src/test/recovery` covers engine-driven truncation and `zero_damaged_pages`, not external file damage. The measured script is therefore the evidence for this page.

### Final causal summary

A manually zeroed relation file in PostgreSQL 12 is a **silent data-loss event for reads and a recoverable-looking table for size-bounded operations**: `SELECT *`, `VACUUM`, and `pg_dump` all succeed because the storage manager defines the relation's end as the first short segment, while `pg_relation_size`, index-only scans, index-driven heap fetches, warm backends, and FSM-steered inserts each expose a different and partly contradictory view of the same files — some erroring with `XX001`/`XX000`/`58P01`, some returning stale or partial data, and buffered writes able to regrow the truncated file underneath the running server.

## Measurement Script

### Usage

| Field | Value |
|---|---|
| Purpose | measures what `SELECT`, `count(id)`, `VACUUM`, `pg_dump`, index-driven fetches, `INSERT`, and `CHECKPOINT` do in PostgreSQL 12.2 with data checksums disabled after one relation file (main-fork segment, `_fsm`, or `_vm`) is truncated to zero bytes, for both fresh and already-warmed backends |
| Invocation | `bash zerofile_v12.sh [stage ...]`, run from the repository root (it reads `raw/postgres-12/` relative to `$PWD` and builds out of tree under `$SANDBOX`) |
| Stages | `build cluster fixtures legs` in default order; the individual `leg_*` names re-run single legs; `report` lists outputs; `stop` and `clean` tear the sandbox down and are not in the default order |
| Environment | `WIKI_ROOT SRC12 SANDBOX PORT12 JOBS RELSEG_BLOCKS ROWS_PER_SEG` |
| Prerequisites | a C toolchain, GNU make, bison and flex (the pin is a git checkout, so generated parsers are built); readline and zlib development headers, because `build` configures `--with-readline --with-zlib`; the pinned checkout at `SRC12` |
| Output | one file per leg under `$SANDBOX/out/`; read `platform.txt` first — it records the patched `RELSEG_SIZE` (`#define RELSEG_SIZE 128`, `SHOW segment_size` → `1MB`) and `data_checksums = off` |
| Runtime | on the review host: about 3 minutes for `build` plus `cluster` from an empty sandbox, under a minute for `fixtures` and all legs |
| Cleanup | `bash zerofile_v12.sh stop clean` stops the cluster with `pg_ctl -m fast -w stop`, verifies no `postmaster.pid` and no process naming `$DATA`, then deletes `$SANDBOX` |

### Stages

| Stage | What it does | Needs |
| --- | --- | --- |
| `build` | configures 12.2 out of tree under `$SANDBOX/build`, patches the generated `pg_config.h` to `RELSEG_SIZE 128` (a non-default compile-time knob, because `--with-segsize` accepts only whole gigabytes), then `make`/`make install` into `$SANDBOX/install`; skips everything when the binary exists | nothing |
| `cluster` | `initdb -A trust --locale=C --encoding=UTF8` (checksums off by default), writes socket-only settings plus `autovacuum=off` and `fsync=off`, starts on `PORT12`, creates database `zerofile`, records platform facts into `out/platform.txt` | `build` |
| `fixtures` | creates one disposable fixture per leg: `t_small` (one segment) and ten ~4 MB tables `t_mid…t_rsu` (four segments), each with a `PRIMARY KEY` index, deleted rows, `VACUUM` so `_fsm`/`_vm` exist, and `CHECKPOINT` | `cluster` |
| `legs` | runs every `leg_*` stage below | `fixtures` |
| `leg_baseline` | inventories files, row counts, per-segment row distribution, and sizes of `t_small`/`t_mid` | `fixtures` |
| `leg_small` | zeroes the only file of `t_small` while running; probes a fresh backend and a warmed session | `fixtures` |
| `leg_mid` | zeroes segment 1 of `t_mid`, fresh backend | `fixtures` |
| `leg_midwarm` | zeroes segment 1 of `t_midw` inside a session that had opened all segments via `EXPLAIN` | `fixtures` |
| `leg_last` | zeroes the last segment of `t_last` | `fixtures` |
| `leg_first` | zeroes segment 0 of `t_first` | `fixtures` |
| `leg_delfirst` | deletes segment 0 of `t_delfirst` outright | `fixtures` |
| `leg_delmid` | deletes segment 1 of `t_delmid` outright | `fixtures` |
| `leg_fsm` | zeroes the `_fsm` fork of `t_fsm`; fresh and warmed backends | `fixtures` |
| `leg_vm` | zeroes the `_vm` fork of `t_vm`; fresh and warmed backends | `fixtures` |
| `leg_regrow` | zeroes segment 1 of `t_grow`, shows an FSM-steered `INSERT` error, then a page-sized `INSERT` that regrows over the hole | `fixtures` |
| `leg_resurrect` | dirties buffers, zeroes a segment and `_fsm`, then `CHECKPOINT` shows the files regrowing from flushed dirty buffers | `fixtures` |
| `leg_restart` | zeroes a middle segment, restarts the cluster, and probes the first backend afterwards | `fixtures` |
| `report` | lists the result files | any |
| `stop` | stops the cluster and verifies the teardown (no `postmaster.pid`, no process naming `$DATA`) | none |
| `clean` | deletes `$SANDBOX`, refusing any path outside `.wiki-runtime/tmp/` | `stop` |

Every fixture object is disposable: the script creates and drops its own database and tables, and must not be pointed at a database anyone cares about. The leg stages mutate their fixtures and truncate cluster files; re-run `fixtures` before re-running a single leg on its own.

Cluster settings and their apply scope, all written into `postgresql.conf` before the first start: `listen_addresses`, `unix_socket_directories`, `port`, and `shared_buffers` are `PGC_POSTMASTER` (restart); `autovacuum` and `fsync` are `PGC_SIGHUP` (reload). `autovacuum` is off so that no background worker touches a truncated relation, and `fsync` is off because the cluster is disposable. The script's own statements carry `statement_timeout = 10min` and `lock_timeout = 5s`, both `PGC_USERSET` in this version, passed through `PGOPTIONS` because a multi-statement `psql -c` would wrap `VACUUM` in a transaction block.

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
#   bash zerofile_v12.sh                 # all stages, default order
#   bash zerofile_v12.sh legs            # re-run just the legs
#   bash zerofile_v12.sh leg_mid         # one leg
#   bash zerofile_v12.sh stop clean      # tear the sandbox down
#
# Stages: build cluster fixtures legs
#         leg_baseline leg_small leg_mid leg_midwarm leg_last leg_first
#         leg_delfirst leg_delmid leg_fsm leg_vm leg_regrow leg_resurrect
#         leg_restart report stop clean
#
# Environment: WIKI_ROOT SRC12 SANDBOX PORT12 JOBS RELSEG_BLOCKS ROWS_PER_SEG
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
SRC12="${SRC12:-$WIKI_ROOT/raw/postgres-12}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/zerofile-measure}"
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

# statement_timeout and lock_timeout are PGC_USERSET in PostgreSQL 12; they
# travel through PGOPTIONS so they also cover VACUUM, which a multi-statement
# psql -c would wrap in one implicit transaction.
export PGOPTIONS="-c statement_timeout=10min -c lock_timeout=5s"

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

# -X ignores ~/.psqlrc; ON_ERROR_STOP is on the load helpers so a failed DDL
# statement cannot slip through silently.  The probe helpers deliberately
# leave it off: their whole job is to capture the server's error text.
q()  { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -c "$1" > /dev/null || die "failed: $1"; }
s()  { "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$DB" -c "$1"; }

# probe_* helpers run one statement in a fresh backend, print its output or
# the server's error, and finish with rc=<psql exit status>.  VERBOSITY
# verbose exposes the SQLSTATE of each error.
probe_sql() { # probe_sql <label> <sql>
  printf -- '--- %s\n' "$1"
  "$BIN/psql" -X -v VERBOSITY=verbose -d "$DB" -c "$2" 2>&1
  printf 'rc=%s\n' "$?"
}
probe_count() { # probe_count <table>: a literal "read every row" scan
  probe_sql "seq count(*) on $1" \
    "SET enable_indexscan=off; SET enable_bitmapscan=off; SET enable_indexonlyscan=off; SELECT count(*) FROM $1"
}
probe_idx() { # probe_idx <table> <id>: index-driven fetch of one heap block
  probe_sql "index fetch id=$2 on $1" "SELECT id, left(pad,4) FROM $1 WHERE id = $2"
}
probe_vac() { probe_sql "VACUUM $1" "VACUUM $1"; }
probe_ioscount() { # probe_ioscount <table>: index-only count, never reads heap
  probe_sql "plan of count(id) on $1" "EXPLAIN (COSTS OFF) SELECT count(id) FROM $1"
  probe_sql "index-only count(id) on $1" "SELECT count(id) FROM $1"
}
probe_size() {
  probe_sql "size functions on $1" \
    "SELECT pg_relation_size('$1') AS main_bytes, pg_relation_size('$1','fsm') AS fsm_bytes, pg_relation_size('$1','vm') AS vm_bytes, (SELECT relpages FROM pg_class WHERE oid = '$1'::regclass) AS relpages"
}
probe_dump() { # probe_dump <table>
  printf -- '--- pg_dump -t %s\n' "$1"
  "$BIN/pg_dump" -h "$SOCK" -p "$PORT12" -d "$DB" -t "$1" -f "$OUT/dump_$1.sql" \
      2> "$OUT/dump_$1.err"
  printf 'rc=%s\n' "$?"
  [ -s "$OUT/dump_$1.err" ] && sed 's/^/stderr: /' "$OUT/dump_$1.err"
  if [ -f "$OUT/dump_$1.sql" ]; then
    printf 'dumped rows: %s\n' \
      "$(awk '/^COPY /{f=1;next} f&&/^\\\.$/{f=0;next} f{c++} END{print c+0}' "$OUT/dump_$1.sql")"
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
  mkdir -p "$BUILD" "$OUT" "$SQLD"
  [ -x "$BIN/postgres" ] && { note "already built, skipping"; return 0; }
  [ -x "$SRC12/configure" ] || die "no pinned checkout at $SRC12; set SRC12 or run from the repository root"
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" --enable-debug \
      --with-readline --with-zlib CFLAGS="-O2 -g" > configure.log 2>&1 ) \
    || { keep_build_logs; die "configure failed, see $OUT/configure.log"; }
  # Non-default compile-time knob for this measurement: configure accepts
  # --with-segsize only in whole gigabytes, so the generated pg_config.h is
  # patched to a 1 MB segment instead of building a >1 GB fixture.
  sed -i "s/^#define RELSEG_SIZE .*/#define RELSEG_SIZE $RELSEG_BLOCKS/" \
      "$BUILD/src/include/pg_config.h" \
    || die "could not patch RELSEG_SIZE"
  grep -m1 '^#define RELSEG_SIZE' "$BUILD/src/include/pg_config.h" | tee "$OUT/relseg_size.txt"
  [ "$(awk '{print $3}' "$OUT/relseg_size.txt")" = "$RELSEG_BLOCKS" ] \
    || die "RELSEG_SIZE patch did not take"
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { keep_build_logs; grep -m3 'error:' "$BUILD/make.log" >&2
         die "make failed, see $OUT/make.log"; }
  keep_build_logs
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
  mkdir -p "$OUT" "$SQLD" "$SOCK"
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    note "already running"
  else
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
  fi
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d postgres \
    -c "DROP DATABASE IF EXISTS $DB" -c "CREATE DATABASE $DB" \
    || die "could not create database $DB"
  {
    uname -a
    "$BIN/postgres" --version
    printf 'RELSEG_SIZE from build: %s\n' "$(cat "$OUT/relseg_size.txt" 2>/dev/null)"
    s "SHOW segment_size"
    s "SHOW block_size"
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
fresh_battery() { # fresh_battery <table>
  probe_count "$1"
  probe_ioscount "$1"
  probe_vac "$1"
  probe_size "$1"
  probe_dump "$1"
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
    probe_idx t_small 5
    {
      printf '\\echo --- warm session: EXPLAIN opens the segment\n'
      printf 'EXPLAIN (COSTS OFF) SELECT count(*) FROM t_small;\n'
      printf "\\! : > %s\n" "$f"
      printf '\\echo --- warm session: scan after zeroing\n'
      printf 'SET enable_indexscan=off; SET enable_bitmapscan=off; SET enable_indexonlyscan=off;\n'
      printf 'SELECT count(*) FROM t_small;\n'
      printf 'SELECT id, left(pad,4) FROM t_small WHERE id = 5;\n'
    } > "$SQLD/small_warm.sql"
    printf -- '--- warm session output\n'
    "$BIN/psql" -X -v VERBOSITY=verbose -d "$DB" -f "$SQLD/small_warm.sql" 2>&1
    printf 'rc=%s\n' "$?"
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
    probe_idx t_mid "$id1"
    probe_idx t_mid "$id2"
    probe_sql "pg_class relpages after vacuum" \
      "SELECT relpages, reltuples FROM pg_class WHERE oid = 't_mid'::regclass"
  } > "$OUT/leg_mid.txt" 2>&1
  cat "$OUT/leg_mid.txt" >&2
}

# Middle segment -> 0 bytes, backend already holds every segment open.
# Expected: mdnblocks trusts the open tail segment as full, computes the
# pre-truncation nblocks, and the scan errors on a short read.  The same
# session's VACUUM errors the same way; a later fresh session does not.
stage_leg_midwarm() {
  say "leg midwarm: zero segment 1 of t_midw, warmed backend"
  restart_cluster
  local f; f=$(relfile t_midw 1)
  {
    printf '\\echo --- warm session: planner opens all segments\n'
    printf 'SET enable_indexscan=off; SET enable_bitmapscan=off; SET enable_indexonlyscan=off;\n'
    printf 'EXPLAIN (COSTS OFF) SELECT count(*) FROM t_midw;\n'
    printf "\\! : > %s\n" "$f"
    printf '\\echo --- warm session: scan after zeroing\n'
    printf 'SELECT count(*) FROM t_midw;\n'
    printf 'VACUUM t_midw;\n'
    printf '\\echo --- warm session: second attempt\n'
    printf 'SELECT count(*) FROM t_midw;\n'
  } > "$SQLD/midwarm.sql"
  {
    printf 'zeroed file: %s\n' "$f"; lsrel t_midw
    "$BIN/psql" -X -v VERBOSITY=verbose -d "$DB" -f "$SQLD/midwarm.sql" 2>&1
    printf 'rc=%s\n' "$?"
    printf -- '--- fresh backend afterwards\n'
    probe_count t_midw
  } > "$OUT/leg_midwarm.txt" 2>&1
  cat "$OUT/leg_midwarm.txt" >&2
}

# Last segment -> 0 bytes.  Expected: silent shrink for every backend state,
# because the truncated segment is always the last open one and its size is
# always rechecked; index fetches into it hit the short-read error.
stage_leg_last() {
  say "leg last: zero the final segment of t_last"
  local nseg f idl
  nseg=$(ls "$DATA/$(s "SELECT pg_relation_filepath('t_last')")".* 2>/dev/null | wc -l)
  [ "$nseg" -ge 1 ] || die "t_last has no numbered segments"
  f=$(relfile t_last "$nseg")
  idl=$(pick_id t_last "$nseg")
  restart_cluster
  : > "$f" || die "truncate failed"
  {
    printf 'zeroed file: %s (segment %s)\n' "$f" "$nseg"; lsrel t_last
    fresh_battery t_last
    probe_idx t_last "$idl"
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
    probe_idx t_first "$id2"
  } > "$OUT/leg_first.txt" 2>&1
  cat "$OUT/leg_first.txt" >&2
}

# Contrast leg: first segment deleted outright (not zeroed).  Expected:
# every access errors at open time with "could not open file".
stage_leg_delfirst() {
  say "leg delfirst: delete segment 0 of t_delfirst"
  restart_cluster
  local f; f=$(relfile t_delfirst 0)
  rm -f "$f" || die "rm failed"
  {
    printf 'deleted file: %s\n' "$f"; lsrel t_delfirst
    fresh_battery t_delfirst
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
    probe_idx t_delmid "$id2"
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
      printf "INSERT INTO t_fsm (pad) VALUES ('warm');\n"
      printf "\\! : > %s\n" "$f"
      printf '\\echo --- warm session: bulk INSERT forces an FSM consult\n'
      printf "INSERT INTO t_fsm (pad) SELECT repeat('y',400) FROM generate_series(1,50);\n"
      printf '\\echo --- warm session: retry and VACUUM in same session\n'
      printf "INSERT INTO t_fsm (pad) SELECT repeat('y',400) FROM generate_series(1,50);\n"
      printf 'VACUUM t_fsm;\n'
    } > "$SQLD/fsm_warm.sql"
    printf -- '--- warm session output\n'
    "$BIN/psql" -X -v VERBOSITY=verbose -d "$DB" -f "$SQLD/fsm_warm.sql" 2>&1
    printf 'rc=%s\n' "$?"
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
      printf 'VACUUM t_vm;\n'
      printf '\\echo --- warm session: index-only scan caches the vm size\n'
      printf 'EXPLAIN (COSTS OFF) SELECT count(id) FROM t_vm;\n'
      printf 'SELECT count(id) FROM t_vm;\n'
      printf "\\! : > %s\n" "$f"
      printf '\\echo --- warm session: index-only scan after re-zeroing\n'
      printf 'SELECT count(id) FROM t_vm;\n'
      printf '\\echo --- warm session: VACUUM\n'
      printf 'VACUUM t_vm;\n'
    } > "$SQLD/vm_warm.sql"
    printf -- '--- warm session output\n'
    "$BIN/psql" -X -v VERBOSITY=verbose -d "$DB" -f "$SQLD/vm_warm.sql" 2>&1
    printf 'rc=%s\n' "$?"
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
      "INSERT INTO t_grow (pad) SELECT repeat('y',400) FROM generate_series(1,$((ROWS_PER_SEG * 2)))"
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
    probe_idx t_rest "$id2"
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
  say "result files"
  ls -l "$OUT" | sed 's/^/   /' >&2
  note "read platform.txt first, then leg_*.txt"
}

stage_stop() {
  say "stop cluster"
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop || die "pg_ctl stop failed"
  else
    note "not running"
  fi
  [ ! -f "$DATA/postmaster.pid" ] || die "postmaster.pid survived"
  if pgrep -f -- "$DATA" > /dev/null 2>&1; then
    pgrep -af -- "$DATA" >&2 || true
    die "a process still names $DATA"
  fi
  note "stopped: no postmaster.pid, no process names $DATA"
}

stage_clean() {
  say "delete sandbox $SANDBOX"
  stage_stop
  case "$SANDBOX" in
    "$WIKI_ROOT"/.wiki-runtime/tmp/*) ;;
    *) die "refusing to delete SANDBOX outside .wiki-runtime/tmp/: $SANDBOX" ;;
  esac
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
- `src/backend/commands/vacuum.c` — `vac_update_relstats`/`vac_estimate_reltuples` rewriting `relpages`/`reltuples` from the scan.
- `src/backend/commands/copy.c` — `CopyTo` uses `table_beginscan`, the same sequential read a `SELECT` uses.
- `src/bin/pg_dump/pg_dump.c` — `dumpTableData_copy` issues `COPY ... TO stdout` and aborts on a failed COPY.
- `src/backend/utils/adt/dbsize.c` — `calculate_relation_size` sums `stat().st_size` over existing segments, stopping at the first missing name.
- `src/backend/executor/nodeIndexonlyscan.c` — the visibility-map check that lets an index-only scan skip the heap entirely.
- `src/backend/access/heap/heapam_handler.c` — `heapam_index_fetch_tuple`'s `heap_hot_search_buffer` heap fetch, and `heapam_estimate_rel_size`'s `RelationGetNumberOfBlocks` call that opens all segments.
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
| Sequential scans, vacuum, and `COPY TO` are bounded by `nblocks` | [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L295), [vacuumlazy.c#nblocks](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L549-L556), [copy.c#CopyTo](../../../../raw/postgres-12/src/backend/commands/copy.c#L2114-L2137); measured rc=0 with fewer rows on every leg |
| `pg_dump` inherits the COPY outcome, including its failure | [pg_dump.c#dumpTableData_copy](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1826-L1929); measured exit 0 with 2,282 rows, exit 1 on the deleted-first-file leg |
| Short reads error `XX001` unless `zero_damaged_pages` or recovery zeroes them | [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630); measured `read only 0 of 8192 bytes` |
| Reading past a short segment errors `previous segment is only %u blocks` (XX000 via stale errno) | [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1219), [elog.c#errcode_for_file_access](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L583-L652); measured `XX000` |
| Open segments are trusted as `RELSEG_SIZE`, so a warm backend computes stale `nblocks` | [md.c#mdnblocks-trust](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L715-L729); measured `XX001` in the warm session and silent 2,282 after the session's own `VACUUM` |
| A missing first segment fails every open with `58P01` | [md.c#mdopen](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L450-L463), [elog.c#enoent](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L611-L614); measured |
| `fsm_readbuf`/`vm_readbuf` gate reads on the fork's own `smgrnblocks` | [freespace.c#fsm_readbuf](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L552-L575), [visibilitymap.c#vm_readbuf](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L567-L587); measured no error on both fork legs |
| `RBM_ZERO_ON_ERROR` only substitutes for invalid page headers, not short reads | [bufmgr.c#modes](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L615-L639), [bufmgr.c#verify-path](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) |
| An all-zero page is a valid page | [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142); observed as sparse holes in the resurrected segment |
| Dirty buffered pages are written back at block offsets regardless of file truncation | [md.c#mdwrite](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L640-L695); measured `.1` regrew to 335,872 bytes and `_fsm` to 24,576 bytes at `CHECKPOINT` |
| `INSERT` consults the FSM for a target page and extends only when it finds none | [hio.c#fsm-target](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L381-L400), [hio.c#extend](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L555-L594); measured `XX001` for normal tuples and `INSERT 0 300` for page-sized tuples |
| `pg_relation_size` sums `st_size` of every existing segment | [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L272-L308); measured 2,916,352 while scans reached 1 MB |
| `VACUUM` rewrites `relpages`/`reltuples` from the shortened scan | [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1157-L1262); measured `relpages` 484 → 128 |
| Index-only scans can answer without the heap | [nodeIndexonlyscan.c#vm-check](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L155-L189); measured `count(id)` 8,614 vs `count(*)` 2,282 |
| Segments are 1 GB by default and GB-only configurable in 12 | [storage.sgml#segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L233-L243), [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L292) |
| Engine-side truncation invalidates other backends' smgr state; an external truncate does not | [smgr.c#smgrtruncate](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L645-L670), [md.c#mdnblocks-note](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L720-L727); measured warm-backend `XX001` |

## Open Questions

- **Warm-backend `_fsm`/`_vm` short read.** `fsm_readbuf`/`vm_readbuf` can issue `ReadBufferExtended` for a fork block below a *stale cached* `smgr_fsm_nblocks`/`smgr_vm_nblocks`, where `RBM_ZERO_ON_ERROR` does not mask the `mdread` short read. In the measured runs the fork pages stayed in shared buffers, so the warmed sessions never hit the disk and showed no error; whether the `XX001` path is reachable in practice needs a run that evicts the fork page while keeping the stale size cache. [freespace.c#fsm_readbuf-recheck](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L552-L575) [bufmgr.c#verify-path](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925)
- **A partially truncated or sparse segment.** `PageIsVerified` accepts all-zero pages, so a segment that still has length but whose middle blocks were zeroed should read as silently empty pages rather than error; the resurrect leg's sparse file shows the same shape but the page was not re-read in that state. Not separately measured.
- **TOAST and other relations' files.** Zeroing a TOAST heap's file should surface `missing chunk number` style errors on access — that family is catalogued on the [corruption-indicators page](../observability/corruption-log-messages-with-checksums-disabled.md) — but this page measured only heap, `_fsm`, and `_vm` files of ordinary tables.
- **`zero_damaged_pages` / recovery interplay.** With `zero_damaged_pages = on`, the short reads measured here should return zeroed buffers instead of `XX001`, converting the index-fetch and warm-scan errors into silent all-zero pages. The setting is a documented data-loss debugging aid and was not enabled in these runs. [md.c#zero-damaged](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L614-L623)

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
- [bufmgr.h](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L1-L279) — `RelationGetNumberOfBlocks`
- [storage.sgml](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L1-L1073) — fork/segment naming and defaults
- [configure.in](../../../../raw/postgres-12/configure.in#L1-L2488), [pg_config.h.in](../../../../raw/postgres-12/src/include/pg_config.h.in#L1-L1043) — `--with-segsize`, `RELSEG_SIZE`

## Navigation

- Up: [PostgreSQL 12 wiki](../../index.md) · [Wiki index](../../../index.md)
- Same category: [VACUUM FULL I/O on a near-empty heap (unverified)](vacuum-full-io-on-near-empty-heap.md) · [pgstattuple_approx heap and TOAST bloat (unverified)](pgstattuple-approx-heap-and-toast-bloat.md) · [catalog statistics bloat query (unverified)](catalog-statistics-bloat-query-reliability.md) · [very large shared_buffers (unverified)](very-large-shared-buffers.md)
- Related: [Corruption indicators in logs with checksums disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md) · [Glossary](../../../glossary.md)
