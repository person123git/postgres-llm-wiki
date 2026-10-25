---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Does amcheck Detect an Emptied Heap or Index Segment When Data Checksums Are On in PostgreSQL 12? (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Measured results](#measured-results)
  - [Mental model](#mental-model)
  - [Logic map](#logic-map)
  - [Why the heap scenarios pass](#why-the-heap-scenarios-pass)
  - [Why the index scenarios fail](#why-the-index-scenarios-fail)
  - [Why data checksums do not help](#why-data-checksums-do-not-help)
  - [Branches and limits](#branches-and-limits)
  - [Causal summary](#causal-summary)
- [Measurement Script](#measurement-script)
  - [Last run](#last-run)
  - [Usage](#usage)
  - [Non-default settings](#non-default-settings)
  - [Script](#script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Follow AGENTS.md. In PostgreSQL 12, question: On a database with data checksums enabled and using amcheck, what happens if a disk file in the middle of a table heap is emptied and amcheck is used? Is the issue detected? Another scenario: what happens if the disk file at the end of a table heap is emptied and amcheck is used? Is the issue detected? Run these two scenarios also for an index.

## Answer

### Short answer

**For the heap, no. For the index, yes.** This holds whether the emptied file is a middle segment or the last segment. Data checksums made no difference in any of the four cases.

| Emptied file | `bt_index_check` / `bt_index_parent_check`, with or without `heapallindexed` | Detected? |
|---|---|---|
| Middle heap segment | All three calls succeed | **No** |
| Last heap segment | All three calls succeed | **No** |
| Middle index segment | All three calls fail with `XX001 could not read block … read only 0 of 8192 bytes` | **Yes** |
| Last index segment | All three calls fail with the same error | **Yes** |

Here "emptied" means a segment file truncated to zero bytes while the server was stopped. PostgreSQL 12 stores each relation fork as numbered segment files of `RELSEG_SIZE` blocks; see [storage manager](../../../glossary.md#storage-manager) ([md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66)). The measurement used a build with 1 MB segments so that a small table spans several files; see [Non-default settings](#non-default-settings).

PostgreSQL 12's [`amcheck`](../../../glossary.md#amcheck) checks only [B-tree](../../../glossary.md#b-tree) indexes. It has no heap checker; the module builds only `verify_nbtree.c` ([Makefile:4](../../../../raw/postgres-12/contrib/amcheck/Makefile#L4)). Its only view of the [heap](../../../glossary.md#heap) is the optional `heapallindexed` scan, and that scan cannot see missing heap pages. This page complements the source-only [zeroed-segment amcheck page](amcheck-zeroed-relation-segments.md), which predicted the heap result without running it.

### Measured results

The [script](#script) built 16,000-row fixtures. Each heap had 843 blocks (seven 128-block segments, `.0` to `.6`). Each index on the 180-byte `k` column had 450 blocks (four segments, `.0` to `.3`). The script stopped the cluster cleanly, emptied one file per fixture, ran `pg_checksums --check`, and restarted. Each probe then ran in a new backend.

| Fixture | File emptied | Bytes before → after | amcheck (3 calls) | Sequential `count(*)` | Index fetch of a row from the lost range |
|---|---|---|---|---|---|
| `ctl` (control) | none | — | success | 16,000 | — |
| `h_mid` | heap `.1` | 1,048,576 → 0 | **success** | **2,432** | `XX001 could not read block 128 … read only 0 of 8192 bytes` |
| `h_last` | heap `.6` | 614,400 → 0 | **success** | **14,592** | `XX001 could not read block 768 … read only 0 of 8192 bytes` |
| `i_mid` | index `.1` | 1,048,576 → 0 | **`XX001` at block 155** | 16,000 | — |
| `i_last` | index `.3` | 540,672 → 0 | **`XX001` at block 395** | 16,000 | — |

The three amcheck calls were `bt_index_check(idx)`, `bt_index_check(idx, true)` and `bt_index_parent_check(idx, true)`. They returned the same outcome on each fixture. `pg_checksums --check` scanned 1,336 files and 9,979 blocks, reported `Bad checksums: 0`, and exited 0. The server log contained no checksum or invalid-page lines.

Two numbers show how much data the heap cases lost. In `h_mid`, only segment `.0` remained readable to a sequential scan: 2,432 of 16,000 rows. Segments `.2` to `.6` were still intact on disk, but no scan read them. In `h_last`, 14,592 rows remained, which is everything in segments `.0` to `.5`.

### Mental model

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| Relation size in blocks | `mdnblocks()` walks the segments and stops at the first one shorter than `RELSEG_SIZE` | Segment file sizes, measured when asked | Live in a new backend | Sets the heap scan's end block, `rs_nblocks` ([heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231), [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755)) |
| B-tree page links | The metapage, downlinks and right-links that amcheck follows level by level | Stored in index pages | Stored | Names each index block amcheck reads ([verify_nbtree.c#root-and-levels](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551), [verify_nbtree.c#level-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L650-L833)) |
| Index fingerprint | A [Bloom filter](../../../glossary.md#bloom-filter) of the index tuples amcheck visited | Built during the check | Live, discarded afterwards | `heapallindexed` tests each heap tuple against it ([verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624), [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920)) |
| `pd_checksum` | A 16-bit page checksum stored in each page header; see [data checksums](../../../glossary.md#data-checksums) | Written when a page is flushed | Stored | Compared only when a full, non-zero page is read ([bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L142)) |

The key contrast is between the two ways of naming a block. A heap scan asks the file system how big the relation is. An index walk follows block numbers stored inside the index pages. An emptied segment shrinks the first answer, so nothing looks wrong. It does not change the second answer, so a stored link points to a block that no longer exists.

### Logic map

```text
amcheck on index I of table T  (new backend, nothing cached)
│
├─ 1. Walk I from the metapage: every level, left to right [A]
│     └─ each block: ReadBuffer -> smgrread -> mdread [B]
│          ├─ block's segment empty  -> short read -> ERROR XX001 [C]
│          ├─ block beyond an empty earlier segment
│          │                        -> ERROR "previous segment is only N blocks" [D]
│          └─ full page read -> checksum + header check [E] -> B-tree checks
│
├─ 2. heapallindexed = false -> done, success
│
└─ 3. heapallindexed = true -> scan T from block 0 to mdnblocks(T) [F]
      ├─ mdnblocks stops at the first short segment, so an emptied
      │  middle or last heap segment just ends the scan early [G]
      ├─ each tuple found -> "is it in the fingerprint?" [H]
      │    ├─ definitely absent -> ERROR (heap tuple missing from index)
      │    └─ possibly present  -> continue
      └─ scan ends -> success; index entries for lost rows are never checked [I]
```

- **[A]** [verify_nbtree.c#root-and-levels](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551), [verify_nbtree.c#level-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L650-L833).
- **[B]** [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2351), [bufmgr.c#read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L880-L898).
- **[C]** [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630), [errcodes.txt:493](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L493).
- **[D]** [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1204).
- **[E]** [bufmgr.c#PageIsVerified-call](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L923), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L142).
- **[F]** [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624), [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231).
- **[G]** [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L725-L755).
- **[H]** [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920).
- **[I]** The callback runs only for heap tuples the scan returns ([verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624)).

### Why the heap scenarios pass

1. **The index is intact, so step 1 succeeds.** amcheck reads only index blocks in the walk, and none of them was touched ([verify_nbtree.c#root-and-levels](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551)).
2. **The heap scan's length comes from `mdnblocks()`.** A new backend has opened no segments. `mdnblocks()` therefore measures segment `.0`, finds it full, opens `.1`, and so on. It returns at the first segment shorter than `RELSEG_SIZE` ([md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L725-L755)).
3. **An empty segment looks like the normal end of the relation.** In `h_mid`, segment `.1` has 0 blocks, so the relation appears to have 128 blocks. In `h_last`, segment `.6` has 0 blocks, so it appears to have 768. A fork whose last segment is zero length is also a layout that `mdtruncate()` creates legitimately ([md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66)). As a result, nothing in the size path raises an error.
4. **The scan never visits the lost blocks.** It reads blocks 0 to 127 in `h_mid` and 0 to 767 in `h_last` ([heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231)).
5. **The comparison is one-way.** `heapallindexed` asks only "is each heap tuple I found in the index?" Every tuple the scan found is still indexed, so every test answers "possibly present" ([verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920)). Index entries that point into the lost range are never looked up in the heap, so the dangling entries go unreported.

The middle case is worse than the last case. Emptying segment `.1` also hid the intact segments `.2` to `.6`, because `mdnblocks()` stops at the first short segment and never opens the later ones ([md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L725-L755)). That is why the sequential count fell to 2,432 rows.

The damage is still reachable from outside amcheck. An index scan that fetches a lost row reads the heap block named by the entry's [TID](../../../glossary.md#tid). That read hits the empty segment and fails with `XX001`, as both index-fetch probes show ([md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)). The same `mdread()` error is what catches the index cases below.

### Why the index scenarios fail

amcheck walks every level of the B-tree from the true root, then follows right-links across each level ([verify_nbtree.c#root-and-levels](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551), [verify_nbtree.c#level-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L650-L833)). Every page it reaches is read through `ReadBufferExtended()` in normal mode ([verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2351)). Because a healthy index has links to pages in every segment, the walk is bound to ask for a block in the emptied file.

1. **Input:** a stored block number, 155 in `i_mid` and 395 in `i_last`. Both lie in the emptied segment: 155 is in `.1` (blocks 128–255) and 395 is in `.3` (blocks 384–449).
2. **Segment lookup:** `_mdfd_getseg()` opens the target segment. The file exists, so the open succeeds ([md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1101-L1215)).
3. **Read:** `mdread()` reads 8,192 bytes at the block's offset and gets 0. Outside recovery and with `zero_damaged_pages` off, a short read raises `ERRCODE_DATA_CORRUPTED`, which is SQLSTATE `XX001` ([md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630), [errcodes.txt:493](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L493)).
4. **Consequence:** the error aborts the check, so `heapallindexed` never runs. All three calls fail the same way because they share the walk.

The reported block is the first one the walk needed from the emptied range, not necessarily the first block of that segment. The script did not record whether it was a leaf or internal page; see [Open Questions](#open-questions).

### Why data checksums do not help

Checksums verify the contents of a page that was read. An emptied file supplies no page to verify.

| Check | What it needs | What an emptied segment gives it | Result |
|---|---|---|---|
| Online read, `PageIsVerified()` | A full 8 kB page from `mdread()` | No page: `mdread()` errors first on the short read, or the block is never requested | Never runs on the lost blocks ([bufmgr.c#PageIsVerified-call](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L923), [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)) |
| Offline `pg_checksums --check` | Blocks returned by `read()` on each file | `read()` returns 0 at once, which the tool treats as end of file | Zero blocks checked, no error ([pg_checksums.c#scan_file](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L170-L199)) |

So in the index cases, detection comes from the short-read error in `mdread()`, not from a checksum. In the heap cases, neither path reads the lost blocks. The measured `Bad checksums: 0` and the empty server-log search agree with this.

Checksums are a non-default cluster state in PostgreSQL 12. By default `initdb` leaves them off, and `-k` turns them on ([initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [initdb.c:3147](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3147)). The script used `-k` because the question asks about that state. Since both detection paths above run before or without any checksum comparison, the four outcomes do not depend on it; the script ran no checksums-off leg to measure that.

### Branches and limits

| Condition / state | Behavior | Consequence |
|---|---|---|
| Index walk needs a block beyond an empty earlier segment before any block inside it | `_mdfd_getseg()` raises "previous segment is only N blocks" with a file-access SQLSTATE ([md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1204)) | Still detected, with a different message. Not seen in this run. |
| `zero_damaged_pages = on` (default off, superuser session scope) | `mdread()` returns a zero page instead of failing on the short read ([md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L615-L630), [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161)) | The index error changes form or may disappear. Not measured here. |
| File emptied while the server runs, with its pages in [shared buffers](../../../glossary.md#buffer-manager) or segments already open | A buffer hit returns the old page without a disk read; an open-segment list can make `mdnblocks()` trust a short segment as full ([bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795), [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755)) | Results can differ. The script avoided this by truncating while stopped. |
| Segment overwritten with zeros, length kept | An all-zero page passes `PageIsVerified()` without a checksum comparison ([bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142)) | A different scenario, covered from source on the [zeroed-segment amcheck page](amcheck-zeroed-relation-segments.md#overwriting-bytes-and-damaging-an-index-are-different-cases). |
| Index on an emptied heap is not a B-tree, or the table has no index | amcheck in 12 has no entry point at all ([verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L315-L338)) | The heap loss cannot be checked with amcheck. |

### Causal summary

Clean stop → one segment truncated to 0 bytes → restart, so nothing is cached.

- **Heap emptied:** `mdnblocks()` stops at the first short segment, so the heap seems to end there. The `heapallindexed` scan reads only the remaining blocks, and every tuple it finds is in the index, so amcheck **passes**. Checksums are never consulted because no lost page is read. Only an index fetch of a lost row reveals the damage, with `XX001`.
- **Index emptied:** the B-tree walk follows a stored link into the empty segment. `mdread()` gets a short read and raises `XX001`, so amcheck **fails**, in all three call forms. The error comes from the short read, not from a checksum.

## Measurement Script

### Last run

On 2026-10-25 the script below was run from an empty sandbox on Darwin 27.0.0 arm64 against PostgreSQL 12.2 at pin `45b88269a353ad93744772791feb6d01bc7e1e42`. The server reported `segment_size` 1MB (`RELSEG_SIZE` 128), `block_size` 8192, maximum data alignment 8, `data_checksums` on (checksum version 1), and `shared_buffers` 128MB. All probes matched their expected outcomes. The full run took about 46 seconds, including the build. `stop clean` then stopped the cluster, confirmed no pid file, process or socket remained, and deleted the sandbox.

### Usage

| Item | Value |
|---|---|
| Purpose | Measures whether `bt_index_check` and `bt_index_parent_check` detect a middle or last heap or index segment truncated to zero bytes, with data checksums on. It also measures `pg_checksums --check`, sequential counts and lost-row index fetches. It backs every number in [Measured results](#measured-results). |
| Invocation | From the repository root, extract the fenced `bash` block below with the commands that follow this table, then run `bash .wiki-runtime/tmp/aes12-run.sh [stage ...]`. |
| Stages | Default order: `build cluster fixtures damage probes report`. `build` configures and builds 12.2 out of tree with `RELSEG_SIZE` patched to 128 and installs `amcheck` (skipped when present). `cluster` runs `initdb -k` and starts the server. `fixtures` creates the five tables and their indexes. `damage` records lost-row ids, stops the server, empties four files, runs `pg_checksums --check`, and restarts. `probes` runs the amcheck calls and reads. `report` lists the outputs. `stop` and `clean` are teardown. `probes` can be re-run on its own. `damage` cannot, so rerun from `stop clean`. |
| Environment | `PORT12` (default 55447), `JOBS` (default 8), `ROWS` (default 16000). Paths and `RELSEG_SIZE` are fixed. |
| Prerequisites | C toolchain, make, bison and flex, zlib headers; the pinned checkout at `raw/postgres-12/`. The build uses `--without-readline`. The cluster uses locale `C` and encoding UTF8. |
| Output | `.wiki-runtime/tmp/aes12/out/`. Read `platform.txt` first, then `fixtures.txt`, `damage.txt`, `pg_checksums.txt` and `probe_*.txt`. |
| Runtime | About 1 minute for a full run on this host. A re-run of `probes` with the build present takes a few seconds. |
| Cleanup | `bash .wiki-runtime/tmp/aes12-run.sh stop clean` stops the cluster, checks the teardown, and deletes `.wiki-runtime/tmp/aes12/`. Delete the extracted script afterwards. |

To extract the published script without editing it:

```sh
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/indexing/amcheck-emptied-segments-with-checksums.md | sed '1d;$d' > .wiki-runtime/tmp/aes12-run.sh
bash -n .wiki-runtime/tmp/aes12-run.sh
bash .wiki-runtime/tmp/aes12-run.sh
bash .wiki-runtime/tmp/aes12-run.sh stop clean
```

### Non-default settings

| Setting or state | Default | Script value, why, and what the default does |
|---|---|---|
| `RELSEG_SIZE` | 131,072 blocks (1 GB) ([configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L293), [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864)) | 128 blocks (1 MB), patched into the generated `pg_config.h`, because `--with-segsize` accepts only whole gigabytes. A default build would need a heap over 3 GB for a middle segment. `mdnblocks()`, `_mdfd_getseg()` and `mdread()` test whether a segment is shorter than `RELSEG_SIZE`, not its value, so every measured branch is the one a default build takes ([md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L725-L755), [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1204)). All block numbers in the results depend on it. |
| Data checksums | Off ([initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144)) | On, via `initdb -k`, because the question asks for that state. The `data_checksums` setting is `internal` and cannot be changed afterwards ([guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1827-L1835)). No checksums-off leg was run. |
| `autovacuum` | On ([guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1424-L1434)) | Off in `postgresql.conf` (context `sighup`, reload) so no worker reads a damaged fixture between probes. |
| `listen_addresses`, `unix_socket_directories`, `port` | `localhost`, the compiled socket directory, 5432 | Empty, the sandbox socket directory, 55447 (context `postmaster`, restart), to isolate the cluster ([guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936-L3963), [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177-L2188)). |
| `statement_timeout`, `lock_timeout` | 0 (disabled) | 10 min and 5 s through `PGOPTIONS` (context `user`, session scope), so a stall fails ([guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2401)). |
| Scan enables | All on | The sequential-count probe sets index, bitmap and index-only scans off, and the lost-row probe sets sequential and bitmap scans off. Both are session `SET`s (context `user`), and they force the intended access path ([guc.c#scan-enables](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L894-L922)). |
| Fixtures | — | Disposable tables `ctl`, `h_mid`, `h_last`, `i_mid` and `i_last`, each with a B-tree on `k`, in a script-created database. The files are truncated while the server is stopped, after a clean shutdown checkpoint, so no WAL replay or cached page can rewrite them. |

`shared_buffers` was 128MB, the value `initdb` wrote into the new cluster's configuration. The compiled boot value is 8 MB ([guc.c#shared_buffers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2155-L2163)). The restart before probing emptied the buffer pool, so buffer size does not affect the results.

### Script

```bash
#!/usr/bin/env bash
#
# aes12.sh - the measurement script for the wiki page
# wiki/v12/questions/indexing/amcheck-emptied-segments-with-checksums.md
#
# It builds the pinned PostgreSQL 12 checkout out of tree with a shortened
# RELSEG_SIZE, creates a cluster WITH data checksums (initdb -k), empties
# (truncates to zero bytes) one segment file of four disposable fixtures
# while the server is stopped, then runs amcheck, pg_checksums and plain
# reads against them:
#   h_mid   heap, a middle segment emptied     (index intact)
#   h_last  heap, the last segment emptied     (index intact)
#   i_mid   index, a middle segment emptied    (heap intact)
#   i_last  index, the last segment emptied    (heap intact)
#   ctl     undamaged control
#
# NON-DEFAULT build knob: RELSEG_SIZE is patched from the pin's 131072
# blocks (1 GB) to 128 blocks (1 MB) in the generated pg_config.h, because
# --with-segsize accepts only whole gigabytes in PostgreSQL 12.
# NON-DEFAULT cluster state: initdb -k turns data checksums on (default off).
#
# Every fixture object is disposable; the script creates its own cluster.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/aes12-run.sh            # build cluster fixtures damage probes
#   bash .wiki-runtime/tmp/aes12-run.sh probes     # re-run probes on the damaged cluster
#   bash .wiki-runtime/tmp/aes12-run.sh stop clean # tear down
#
# Stages: build cluster fixtures damage probes report stop clean
# Environment: PORT12 (55447) JOBS (8) ROWS (16000)
set -uo pipefail

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/aes12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55447}"
JOBS="${JOBS:-8}"
ROWS="${ROWS:-16000}"
RELSEG_BLOCKS=128

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; DATA="$SANDBOX/data"
OUT="$SANDBOX/out"; SOCK="$SANDBOX/sock"; BIN="$INST/bin"
DB=aes
export PGPORT="$PORT12" PGHOST="$SOCK"
export TMPDIR="$SANDBOX/tmpdir"
# statement_timeout and lock_timeout are PGC_USERSET: session scope.
export PGOPTIONS="-c statement_timeout=10min -c lock_timeout=5s"

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

own_sandbox() {
  case "$WIKI_ROOT" in */postgres-llm-wiki) ;; *) die "run from the repository root" ;; esac
  [ -f "$SANDBOX/.aes-owned" ] || die "sandbox is not owned by this script: $SANDBOX"
  [ "$(cat "$SANDBOX/.aes-owned")" = "$PIN12 $WIKI_ROOT" ] || die "sandbox owner/pin mismatch"
}
source_pin() {
  [ "$(git --no-optional-locks -C "$SRC12" rev-parse HEAD)" = "$PIN12" ] || die "checkout is not at $PIN12"
}
tag_sql() {
  printf '%s\n' "$1" | sed -E 's/(^|;[[:space:]]*)(SELECT|INSERT|CREATE|DROP|VACUUM|CHECKPOINT|SET|ANALYZE)([[:space:]])/\1\2 \/\* wiki_amcheck_emptied_v12 \*\/\3/g'
}
q() { "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d "$DB" -c "$(tag_sql "$1")" > /dev/null || die "failed: $1"; }
s() { "$BIN/psql" -X -At -q -v ON_ERROR_STOP=1 -d "$DB" -c "$(tag_sql "$1")"; }

# probe <label> <sql> <success|error>: one fresh backend per probe.
probe() {
  local output rc
  printf -- '--- %s\n' "$1"
  output=$("$BIN/psql" -X -At -v ON_ERROR_STOP=1 -v VERBOSITY=verbose -d "$DB" -c "$(tag_sql "$2")" 2>&1)
  rc=$?
  printf '%s\nrc=%s\n' "$output" "$rc"
  case "$3" in
    success) [ "$rc" -eq 0 ] || die "unexpected failure: $1" ;;
    error)   [ "$rc" -ne 0 ] && printf '%s\n' "$output" | grep -q 'ERROR:' \
               || die "expected a server error: $1" ;;
  esac
}

# --------------------------------------------------------------- build -------
stage_build() {
  say "build 12.2 out of tree, RELSEG_SIZE=$RELSEG_BLOCKS, with amcheck"
  source_pin
  if [ -e "$SANDBOX" ]; then own_sandbox; else
    mkdir -p "$SANDBOX" || die "could not create sandbox"
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.aes-owned"
  fi
  mkdir -p "$BUILD" "$OUT" "$TMPDIR"
  if [ -x "$BIN/postgres" ] && [ -f "$INST/lib/amcheck.so" ]; then note "build present"; return 0; fi
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" --without-readline \
      --with-zlib CFLAGS="-O2 -g" > configure.log 2>&1 ) || die "configure failed: $BUILD/configure.log"
  # Non-default compile-time knob, see header comment.
  sed "s/^#define RELSEG_SIZE .*/#define RELSEG_SIZE $RELSEG_BLOCKS/" \
      "$BUILD/src/include/pg_config.h" > "$BUILD/src/include/pg_config.h.new" || die "patch failed"
  mv "$BUILD/src/include/pg_config.h.new" "$BUILD/src/include/pg_config.h"
  grep -q "^#define RELSEG_SIZE $RELSEG_BLOCKS\$" "$BUILD/src/include/pg_config.h" || die "RELSEG_SIZE patch did not take"
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 \
      && make -C contrib/amcheck install > amcheck.log 2>&1 ) || die "make failed: $BUILD/make.log"
  note "$("$BIN/postgres" --version)"
}

# ------------------------------------------------------------- cluster -------
# listen_addresses, unix_socket_directories, port: postmaster (restart).
# autovacuum: sighup (reload); off so no worker touches a damaged fixture.
start_server() {
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w -t 60 start > /dev/null 2>&1 || die "start failed: $OUT/server.log"
  printf '%s\n' "$DATA" > "$SANDBOX/.started"
}
stop_server() {
  "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "stop failed"
  rm -f "$SANDBOX/.started"
}
stage_cluster() {
  say "isolated cluster with data checksums on"
  own_sandbox
  mkdir -p "$SOCK"
  [ -d "$DATA" ] && die "data directory exists; run 'stop clean' first"
  "$BIN/pg_isready" -q -h "$SOCK" -p "$PORT12" && die "port $PORT12 is occupied"
  # -k: non-default; data checksums are off unless initdb is given -k.
  "$BIN/initdb" -k -D "$DATA" -A trust --locale=C --encoding=UTF8 > "$OUT/initdb.log" 2>&1 || die "initdb failed"
  {
    printf "listen_addresses = ''\n"
    printf "unix_socket_directories = '%s'\n" "$SOCK"
    printf "port = %s\n" "$PORT12"
    printf "autovacuum = off\n"
  } >> "$DATA/postgresql.conf"
  start_server
  "$BIN/psql" -X -q -v ON_ERROR_STOP=1 -d postgres -c "CREATE /* wiki_amcheck_emptied_v12 */ DATABASE $DB" || die "createdb failed"
  {
    uname -srm
    "$BIN/postgres" --version
    printf 'segment_size=%s block_size=%s data_checksums=%s shared_buffers=%s\n' \
      "$(s 'SHOW segment_size')" "$(s 'SHOW block_size')" "$(s 'SHOW data_checksums')" "$(s 'SHOW shared_buffers')"
    "$BIN/pg_controldata" "$DATA" | grep -E '^(Maximum data alignment|Data page checksum version):'
  } > "$OUT/platform.txt"
  cat "$OUT/platform.txt" >&2
  [ "$(s 'SHOW data_checksums')" = on ] || die "checksums are not on"
}

# ------------------------------------------------------------ fixtures -------
# Each fixture: heap row ~420 bytes, index key 180 bytes, so both the heap
# and the B-tree on k span several 128-block segments.
stage_fixtures() {
  say "disposable fixtures"
  own_sandbox
  q "CREATE EXTENSION IF NOT EXISTS amcheck"
  local t
  for t in ctl h_mid h_last i_mid i_last; do
    q "DROP TABLE IF EXISTS $t"
    q "CREATE TABLE $t (id int NOT NULL, k text NOT NULL, pad text NOT NULL)"
    q "INSERT INTO $t SELECT g, lpad(g::text, 180, '0'), repeat('x', 200) FROM generate_series(1, $ROWS) g"
    q "CREATE INDEX ${t}_k ON $t (k)"
    q "VACUUM ANALYZE $t"
  done
  q "CHECKPOINT"
  {
    for t in ctl h_mid h_last i_mid i_last; do
      printf '%s heap_blocks=%s index_blocks=%s heap_file=%s index_file=%s\n' "$t" \
        "$(s "SELECT pg_relation_size('$t')/8192")" "$(s "SELECT pg_relation_size('${t}_k')/8192")" \
        "$(s "SELECT pg_relation_filepath('$t')")" "$(s "SELECT pg_relation_filepath('${t}_k')")"
    done
  } > "$OUT/fixtures.txt"
  cat "$OUT/fixtures.txt" >&2
}

# -------------------------------------------------------------- damage -------
# path_of <relation>: data-directory-relative first-segment path.
path_of() { grep "^${1%_k} " "$OUT/fixtures.txt" | sed -E "s/.* ${2}_file=([^ ]*).*/\1/"; }
blocks_of() { grep "^$1 " "$OUT/fixtures.txt" | sed -E "s/.* ${2}_blocks=([0-9]*).*/\1/"; }
stage_damage() {
  say "stop cleanly, empty one segment per fixture, check checksums offline, restart"
  own_sandbox
  # Lost-row ids: the first id stored in the segment that will be emptied.
  local hl=$(( ($(blocks_of h_last heap) - 1) / RELSEG_BLOCKS ))
  printf 'h_mid_seg=1 h_last_seg=%s\n' "$hl" > "$OUT/lost.txt"
  printf 'h_mid_lost_id=%s\n' "$(s "SELECT min(id) FROM h_mid WHERE (ctid::text::point)[0]::int >= $RELSEG_BLOCKS")" >> "$OUT/lost.txt"
  printf 'h_last_lost_id=%s\n' "$(s "SELECT min(id) FROM h_last WHERE (ctid::text::point)[0]::int >= $hl * $RELSEG_BLOCKS")" >> "$OUT/lost.txt"
  stop_server
  local il=$(( ($(blocks_of i_last index) - 1) / RELSEG_BLOCKS ))
  [ "$hl" -ge 2 ] && [ "$il" -ge 2 ] || die "fixtures need at least three segments"
  local f
  for f in "$(path_of h_mid heap).1" "$(path_of h_last heap).$hl" \
           "$(path_of i_mid index).1" "$(path_of i_last index).$il"; do
    [ -f "$DATA/$f" ] || die "missing $f"
    printf '%s before=%s ' "$f" "$(wc -c < "$DATA/$f" | tr -d ' ')"
    : > "$DATA/$f"
    printf 'after=%s\n' "$(wc -c < "$DATA/$f" | tr -d ' ')"
  done > "$OUT/damage.txt"
  printf 'i_last_seg=%s\n' "$il" >> "$OUT/lost.txt"
  cat "$OUT/damage.txt" "$OUT/lost.txt" >&2
  # pg_checksums needs a cleanly stopped cluster; run it on the damaged files.
  "$BIN/pg_checksums" --check -D "$DATA" > "$OUT/pg_checksums.txt" 2>&1
  printf 'rc=%s\n' "$?" >> "$OUT/pg_checksums.txt"
  cat "$OUT/pg_checksums.txt" >&2
  start_server
}

# -------------------------------------------------------------- probes -------
# Fresh backends after a restart: no damaged page is in shared buffers and no
# backend has segment files open.
stage_probes() {
  say "amcheck and reads on every fixture"
  own_sandbox
  local t heap idx lost
  for t in ctl h_mid h_last i_mid i_last; do
    case "$t" in i_*) idx=error ;; *) idx=success ;; esac
    {
      probe "$t bt_index_check" "SELECT bt_index_check('${t}_k')" "$idx"
      probe "$t bt_index_check heapallindexed" "SELECT bt_index_check('${t}_k', true)" "$idx"
      probe "$t bt_index_parent_check heapallindexed" "SELECT bt_index_parent_check('${t}_k', true)" "$idx"
      probe "$t seq count" "SET enable_indexscan=off; SET enable_bitmapscan=off; SET enable_indexonlyscan=off; SELECT count(*) FROM $t" success
      lost=$(sed -n "s/^${t}_lost_id=//p" "$OUT/lost.txt")
      if [ -n "$lost" ]; then
        probe "$t index fetch of lost id $lost" \
          "SET enable_seqscan=off; SET enable_bitmapscan=off; SELECT id FROM $t WHERE k = lpad('$lost', 180, '0')" error
      fi
    } > "$OUT/probe_$t.txt"
    cat "$OUT/probe_$t.txt" >&2
  done
  grep -E 'checksum|invalid page' "$OUT/server.log" > "$OUT/checksum_log.txt"
  printf 'server-log checksum lines: %s\n' "$(wc -l < "$OUT/checksum_log.txt" | tr -d ' ')" >&2
}

stage_report() { own_sandbox; ls -l "$OUT" >&2; note "read platform.txt, damage.txt, pg_checksums.txt, probe_*.txt"; }

stage_stop() {
  say "stop cluster"
  [ -e "$SANDBOX" ] || { note "sandbox absent"; return 0; }
  own_sandbox
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    [ -f "$SANDBOX/.started" ] || die "refusing to stop an unowned server"
    stop_server
  fi
  [ ! -f "$DATA/postmaster.pid" ] || die "postmaster.pid survived"
  pgrep -f -- "^$BIN/postgres" > /dev/null 2>&1 && die "a postgres from $BIN is still running"
  [ ! -d "$SOCK" ] || [ -z "$(ls -A "$SOCK")" ] || die "socket directory is not free"
  note "stopped: no pid file, process, or socket"
}
stage_clean() {
  stage_stop
  [ -e "$SANDBOX" ] || return 0
  own_sandbox
  rm -rf "$SANDBOX"; note "deleted $SANDBOX"
}

stages="$*"
[ $# -eq 0 ] && stages="build cluster fixtures damage probes report"
for st in $stages; do
  case "$(type -t "stage_$st" 2>/dev/null)" in
    function) "stage_$st" ;;
    *) die "unknown stage: $st" ;;
  esac
done
```

## Context Reviewed

- amcheck entry points, target eligibility, the level-by-level walk, page reads through `palloc_btree_page()`, the `heapallindexed` heap scan and its membership callback.
- Storage path below both: `RelationGetNumberOfBlocks()` and `initscan()`, `mdnblocks()`, `_mdfd_getseg()`, `mdread()` and its short-read branch, the segment-layout comment, and `ERRCODE_DATA_CORRUPTED`.
- Checksum path: `ReadBuffer_common()`'s call to `PageIsVerified()`, the all-zero branch, `initdb -k`, the `data_checksums` GUC, and `pg_checksums`'s `scan_file()` end-of-file rule.
- Settings used by the script: `zero_damaged_pages`, `autovacuum`, socket and port settings, timeouts, scan enables, `shared_buffers`, and the `--with-segsize` / `RELSEG_SIZE` generation.
- Comparison set: the [zeroed-segment amcheck page](amcheck-zeroed-relation-segments.md), the [amcheck and checksums page](amcheck-indexes-tables-and-data-checksums.md), the [zeroed relation file page](../storage-and-vacuum/zeroed-relation-file.md), the glossary entries linked here, both indexes and the v12 manifest row. No v12 common concept page covers segments, page reads or amcheck.
- Tests: the amcheck regression suite (`contrib/amcheck/sql/check_btree.sql`) has no truncated-segment case; nothing under `src/test` truncates a segment file underneath amcheck.

## Evidence Map

| Claim | Evidence |
|---|---|
| amcheck in 12 has only B-tree checks | [Makefile:4](../../../../raw/postgres-12/contrib/amcheck/Makefile#L4), [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L315-L338) |
| Heap scan length comes from `mdnblocks()`, which stops at the first short segment | [heapam.c#initscan](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231), [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L725-L755) |
| A zero-length segment is a legitimate layout | [md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66) |
| `heapallindexed` tests only heap tuples it finds | [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624), [verify_nbtree.c#bt_tuple_present_callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920) |
| The index walk reads every level through normal buffer reads | [verify_nbtree.c#root-and-levels](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L502-L551), [verify_nbtree.c#level-walk](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L650-L833), [verify_nbtree.c#palloc_btree_page](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L2335-L2351) |
| A short read raises `XX001` unless `zero_damaged_pages` or recovery | [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630), [errcodes.txt:493](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L493) |
| Reaching past an empty earlier segment raises "previous segment is only N blocks" | [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1204) |
| Checksums are compared only on a full, non-zero page read | [bufmgr.c#PageIsVerified-call](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L923), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L142) |
| `pg_checksums` stops a file at a zero-byte read | [pg_checksums.c#scan_file](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L170-L199) |
| Checksums default off; `-k` turns them on | [initdb.c:144](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [initdb.c:3147](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3147) |
| Measured outcomes and row counts | [Measurement Script](#measurement-script), run on 2026-10-25 |

## Open Questions

- The script did not record whether blocks 155 (`i_mid`) and 395 (`i_last`) were leaf or internal pages, or where the root was. Recording that would need `pageinspect`. The detection result does not depend on it, because the walk reads every non-deleted page.
- No leg emptied a file while the server was running, or with `zero_damaged_pages = on`. The source branches for both are listed under [Branches and limits](#branches-and-limits), but their outcomes were not measured.
- No checksums-off leg was run. The source shows that neither detection path compares a checksum, so the outcomes should be the same, but that is a source reading, not a measurement.
- The emptied segment was always segment `.1` or the last one. Emptying segment `.0` was not tested.
- PostgreSQL 12 has no common concept page for relation segments and page reads, or for B-tree integrity checking. Those would be separate pages; none was created as part of this answer.

## Source References

- [verify_nbtree.c](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L338), [Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L21), [check_btree.sql](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L1-L114)
- [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66), [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L576-L755), [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1101-L1215)
- [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L226-L231), [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L925), [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L142)
- [pg_checksums.c](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L170-L199), [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144), [errcodes.txt](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L493)
- [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161), [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1827-L1835), [configure.in](../../../../raw/postgres-12/configure.in#L279-L293), [pg_config.h.in](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864)

## Navigation

- [Can amcheck Detect Zeroed Relation Segments in PostgreSQL 12? (unverified)](amcheck-zeroed-relation-segments.md)
- [Original zeroed-file question (unverified)](../storage-and-vacuum/zeroed-relation-file.md)
- [PostgreSQL 12 wiki](../../index.md) · [Wiki index](../../../index.md) · [Versions](../../../versions.md) · [Wiki Glossary (unverified)](../../../glossary.md)
