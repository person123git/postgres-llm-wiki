---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# How to Disable Data Checksums in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Values that decide the checksum state](#values-that-decide-the-checksum-state)
  - [What the disable mode does](#what-the-disable-mode-does)
  - [Step-by-step procedure](#step-by-step-procedure)
  - [Errors and refusals](#errors-and-refusals)
  - [What changes after the restart](#what-changes-after-the-restart)
  - [Replication, rewind, backups and upgrades](#replication-rewind-backups-and-upgrades)
  - [Interruption and concurrent-start safety](#interruption-and-concurrent-start-safety)
  - [Other ways to end up without checksums](#other-ways-to-end-up-without-checksums)
  - [Source history](#source-history)
  - [Tests](#tests)
  - [Causal summary](#causal-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

How do you disable data checksums in PostgreSQL 12?

## Answer

### Short answer

**Stop the cluster cleanly, run `pg_checksums --disable -D <datadir>` with the PostgreSQL 12 binaries, then start the cluster again. There is no SQL command, no configuration setting and no online method in PostgreSQL 12.** The tool changes one field in the [control file](../../../glossary.md#control-file), `data_checksum_version`, from 1 to 0. It does not read or rewrite any relation page. [pg_checksums.c#modes](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L72-L92) [pg_checksums.c#scan-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L561) [pg_checksums.c#publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L607) [ref/pg_checksums.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L38-L52)

This answer uses the repository's **PostgreSQL 12.2** pin. [Data checksums](../../../glossary.md#data-checksums) are a whole-cluster property. You cannot disable them for one database or one table. The `data_checksums` setting only reports the state. Its context is `internal` and it is flagged `GUC_DISALLOW_IN_FILE`, so `SET`, `ALTER SYSTEM`, a reload or a `postgresql.conf` edit cannot change it. [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835) [config.sgml#guc-data-checksums](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L9088-L9100) [GUC context](../../../glossary.md#guc-context)

The pinned tree has no online switch. No function such as `pg_disable_data_checksums()` exists in `src/` or `doc/`. A 2018 attempt at online enabling and disabling, `1fde38beaa0`, had its backend side reverted by `a228cc13aea` before PostgreSQL 11 shipped. See [Source history](#source-history).

Disabling is cheap in page I/O, but it has consequences. The server stops verifying pages, `pg_basebackup` stops verifying them, `pg_rewind` loses one of its two eligibility routes, and the old checksum bytes stay in the pages, out of date. The sections below explain each one.

### Values that decide the checksum state

Four stored values matter. Only the first one is the switch.

| Value | Meaning | Where it lives | Live or stored | Who uses it later |
|---|---|---|---|---|
| `ControlFileData.data_checksum_version` | 0 means off. `PG_DATA_CHECKSUM_VERSION` (1) means on. | `global/pg_control` on disk | Stored. The server reads it once, when the postmaster starts. | `DataChecksumsEnabled()`, which every page read and write consults. `pg_checksums`, `pg_rewind`, `pg_upgrade` and `pg_controldata` read the file directly. [pg_control.h#data_checksum_version](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L221-L222) [bufpage.h#PG_DATA_CHECKSUM_VERSION](../../../../raw/postgres-12/src/include/storage/bufpage.h#L200) [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826) [postmaster.c:964](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L964) |
| `data_checksums` setting | Read-only report of the value above. | Set during control-file processing at startup | A copy, taken at startup | `SHOW` and `current_setting()`. [xlog.c#data_checksums-report](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4783-L4785) |
| `ControlFileData.state` | The cluster state, for example shut down, shut down in recovery, or in production. | `global/pg_control` | Stored. A shutdown [checkpoint](../../../glossary.md#checkpoint) or shutdown restartpoint writes the shut-down states. | `pg_checksums` refuses to run unless the state is one of the two shut-down states. [xlog.c#shutdown-state](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8853-L8855) [xlog.c#restartpoint-state](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9126-L9130) [xlog.c#restartpoint-state-normal](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9211-L9212) [pg_checksums.c#state-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L521) |
| `pd_checksum` | A 16-bit field in every [page](../../../glossary.md#page) header. | Each relation page on disk | Stored, per page | A page read compares it only while checksums are on. Disabling does not clear it. [bufpage.h#PageHeaderData](../../../../raw/postgres-12/src/include/storage/bufpage.h#L151-L169) [bufpage.c#checksum-compare](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L103) |

Two values look alike but come from different places. `data_checksum_version` in `pg_control` is the truth on disk. The `data_checksums` setting is the copy a running server took when it started. Because the server reads `pg_control` only at startup, the two can differ only while a server is running against a file that something else changed, and `pg_checksums` refuses to run on a running cluster. Therefore, after a successful disable and a restart, both say off. [xlog.c#LocalProcessControlFile](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4921-L4927) [xlog.c#data_checksums-report](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4783-L4785)

### What the disable mode does

`pg_checksums` runs these tests in this order. Each failure prints an error and exits with status 1.

```text
pg_checksums --disable [-N] [-D DATADIR]
 1. parse options
    ├─ -f/--filenode also given → error, exit 1                               [A]
    └─ no -D, no positional DATADIR, no PGDATA → error, exit 1                [A]
 2. read DATADIR/global/pg_control
    ├─ cannot open or read fully → fatal, exit 1                              [B]
    ├─ CRC does not match → "pg_control CRC value is incorrect", exit 1       [B]
    ├─ pg_control_version != 1201 → "not compatible with this version"        [B]
    └─ block size != this binary's BLCKSZ → "database cluster is not compatible" [B]
 3. state is "shut down" or "shut down in recovery"?
    └─ no → "cluster must be shut down", exit 1                               [C]
 4. data_checksum_version == 0?
    └─ yes → "data checksums are already disabled in cluster", exit 1         [D]
 5. skip the relation-file scan (only --check and --enable scan)              [E]
 6. set data_checksum_version = 0 in the in-memory copy                       [F]
 7. sync the whole data directory, including pg_wal and tablespaces
    (skipped with -N/--no-sync; sync errors are logged, not fatal)            [G]
 8. rewrite global/pg_control with a new CRC; fsync it unless -N              [H]
 9. print "Checksums disabled in cluster", exit 0                             [F]
```

- **[A]** [pg_checksums.c#options](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L415-L452) [pg_checksums.c#datadir](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L454-L478) [pg_checksums.c#filenode-restriction](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L480-L487)
- **[B]** [controldata_utils.c#get_controlfile](../../../../raw/postgres-12/src/common/controldata_utils.c#L50-L144) [pg_checksums.c#control-checks](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L489-L509) [pg_control.h#PG_CONTROL_VERSION](../../../../raw/postgres-12/src/include/catalog/pg_control.h#L25)
- **[C]** [pg_checksums.c#state-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L521)
- **[D]** [pg_checksums.c#already-disabled](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L530-L535)
- **[E]** [pg_checksums.c#scan-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L580)
- **[F]** [pg_checksums.c#publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L607)
- **[G]** [pg_checksums.c#do_sync-default](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L40) [file_utils.c#fsync_pgdata](../../../../raw/postgres-12/src/common/file_utils.c#L58-L114) [file_utils.c#walkdir-ignores-result](../../../../raw/postgres-12/src/common/file_utils.c#L187-L204) [file_utils.c#fsync_fname](../../../../raw/postgres-12/src/common/file_utils.c#L257-L304)
- **[H]** [controldata_utils.c#update_controlfile](../../../../raw/postgres-12/src/common/controldata_utils.c#L155-L262)

Step 5 is why disabling is fast. The scan of `global`, `base` and `pg_tblspc` runs only for `--check` and `--enable`. The disable branch goes straight to the control-file update. [pg_checksums.c#scan-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L561)

Step 7 is the only part that can take noticeable time. By default, `do_sync` is true, so the tool calls `fsync_pgdata()`. That function walks the data directory, a symlinked `pg_wal` and every tablespace, and calls `fsync()` on each file and directory. `-N` / `--no-sync` sets `do_sync` to false, which skips this walk and also the final `fsync()` of `pg_control`. The documentation warns that `--no-sync` can leave the data directory corrupt after an operating-system crash, and recommends it only for testing. [pg_checksums.c#no-sync-option](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L436-L438) [pg_checksums.c#sync-and-update](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L592-L599) [file_utils.c#fsync_pgdata](../../../../raw/postgres-12/src/common/file_utils.c#L58-L114) [ref/pg_checksums.sgml#no-sync](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L114-L127)

A sync failure during step 7 does not stop the tool. `fsync_fname()` logs the error and returns -1, but `walkdir()` ignores that return value. The tool therefore still updates `pg_control` and exits 0. Read the tool's error output, not only its exit status. [file_utils.c#fsync_fname-error](../../../../raw/postgres-12/src/common/file_utils.c#L295-L300) [file_utils.c#walkdir-ignores-result](../../../../raw/postgres-12/src/common/file_utils.c#L187-L204)

The binary must match the cluster's build. `pg_checksums` includes `checksum_impl.h` and `bufpage.h`, so the compile-time `BLCKSZ` and `PG_CONTROL_VERSION` are built into it. A cluster initialized with a different block size, or a `pg_control` from another major version, is refused at step 2. [pg_checksums.c#includes](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L22-L31) [pg_checksums.c#control-checks](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L497-L509)

### Step-by-step procedure

These steps assume a single cluster. For a cluster with standbys, read [Replication, rewind, backups and upgrades](#replication-rewind-backups-and-upgrades) first.

1. **Confirm the current state.** `pg_controldata <datadir>` prints `Data page checksum version:`, with 1 when checksums are on. [pg_controldata.c#checksum-line](../../../../raw/postgres-12/src/bin/pg_controldata/pg_controldata.c#L343-L344)
2. **If you use `pg_rewind`, decide on `wal_log_hints` now.** After disabling, `pg_rewind` accepts this cluster as a target only if `wal_log_hints = on` is recorded in its control file. `wal_log_hints` defaults to `off` and has context `postmaster`, so changing it needs a restart. The restart in step 5 can pick it up. See [Replication, rewind, backups and upgrades](#replication-rewind-backups-and-upgrades). [pg_rewind.c#target-gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429) [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185)
3. **Stop the server cleanly.** `pg_ctl -D <datadir> stop` uses `fast` mode by default. Fast and smart shutdowns finish with a shutdown checkpoint, or a shutdown restartpoint on a standby, which writes the shut-down state that step 3 of the tool requires. Do not use `-m immediate`: it aborts the server without a clean shutdown, so the next start runs crash recovery and `pg_checksums` refuses the cluster. If the cluster crashed, start it, let recovery finish, and stop it cleanly. [checkpointer.c#shutdown](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L372-L382) [xlog.c#ShutdownXLOG](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8336-L8348) [pg_ctl.c#default-mode](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L82) [ref/pg_ctl-ref.sgml#immediate](../../../../raw/postgres-12/doc/src/sgml/ref/pg_ctl-ref.sgml#L195-L197) [ref/pg_ctl-ref.sgml#default-fast](../../../../raw/postgres-12/doc/src/sgml/ref/pg_ctl-ref.sgml#L315-L317) [xlog.c#shutdown-state](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8853-L8855) [pg_checksums.c#state-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L521)
4. **Run the tool as a user that can write `global/pg_control`.** The frontend opens the file with `O_WRONLY`. `-v` additionally prints the new version number.

   ```bash
   pg_checksums --disable -D /path/to/datadir
   # expected last line: Checksums disabled in cluster
   ```

   [controldata_utils.c#open-for-write](../../../../raw/postgres-12/src/common/controldata_utils.c#L199-L206) [pg_checksums.c#publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L598-L606)
5. **Start the server.** The postmaster reads `pg_control` and sets `data_checksums` from it. [postmaster.c:964](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L964) [xlog.c#data_checksums-report](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4783-L4785)
6. **Verify from SQL.** `pg_control_init()` reads `pg_control` from disk on each call, and `current_setting()` returns the startup copy. Both should report off.

   ```sql
   SET /* wiki_verify_checksums_disabled */ statement_timeout = '5s';
   SET /* wiki_verify_checksums_disabled */ lock_timeout = '1s';
   SELECT /* wiki_verify_checksums_disabled */
          current_setting('data_checksums')              AS data_checksums,      -- expect 'off'
          (SELECT data_page_checksum_version
             FROM pg_control_init())                     AS control_file_version; -- expect 0
   ```

   Both timeouts are suggested session limits. Each defaults to 0, meaning no timeout, and both have context `user`, so `SET` applies them to this session only. [pg_proc.dat#current_setting](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5758-L5760) [pg_proc.dat#pg_control_init](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L10604-L10611) [pg_controldata.c#pg_control_init](../../../../raw/postgres-12/src/backend/utils/misc/pg_controldata.c#L265-L349) [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2397)

### Errors and refusals

| Condition / state | Behavior | Consequence |
|---|---|---|
| Server running, or crashed and not yet recovered | `cluster must be shut down`, exit 1 | Nothing changes. Stop cleanly first. [pg_checksums.c#state-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L521) |
| Standby stopped cleanly | Accepted, because a shutdown restartpoint writes `DB_SHUTDOWNED_IN_RECOVERY` | You can disable on a stopped standby as well. [xlog.c#restartpoint-state](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9126-L9130) [xlog.c#restartpoint-state-normal](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9211-L9212) [pg_checksums.c#state-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L516-L517) |
| Checksums already off | `data checksums are already disabled in cluster`, exit 1 | A second run fails instead of silently succeeding, so a script that runs it twice sees exit 1. [pg_checksums.c#already-disabled](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L530-L535) |
| `-f` / `--filenode` combined with `--disable` | `option -f/--filenode can only be used with --check`, exit 1 | There is no per-relation disable. [pg_checksums.c#filenode-restriction](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L480-L487) |
| `pg_control` damaged, unreadable, from another major version, or built for another block size | One of the step-2 errors, exit 1 | Use the PostgreSQL 12 binary built for this cluster. [pg_checksums.c#control-checks](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L489-L509) [controldata_utils.c#frontend-read-errors](../../../../raw/postgres-12/src/common/controldata_utils.c#L71-L105) |
| `pg_control` cannot be opened or written for update | `could not open file` / `could not write file`, exit 1 | Writing happens last, so a failure to open leaves the old state. [controldata_utils.c#update-errors](../../../../raw/postgres-12/src/common/controldata_utils.c#L199-L227) |
| A file `fsync()` fails during the default sync walk | Logged error; the tool continues and exits 0 | A success message does not prove every file was flushed. [file_utils.c#walkdir-ignores-result](../../../../raw/postgres-12/src/common/file_utils.c#L187-L204) |

Unlike `--check` and `--enable`, the disable mode never reads relation files. Therefore it cannot fail with `invalid segment number` or on a checksum mismatch. Damaged pages do not block disabling. [pg_checksums.c#scan-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L561)

### What changes after the restart

Every server-side effect follows from one predicate. `DataChecksumsEnabled()` returns `data_checksum_version > 0`, and after the restart it returns false. [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826)

| Area | With checksums on | After disabling and restarting |
|---|---|---|
| Page reads from storage | `PageIsVerified()` computes the checksum of an initialized page and compares it with `pd_checksum`. | The comparison is skipped. The header sanity test and the all-zero test still run, so a page with an insane header is still rejected. [bufpage.c#checksum-compare](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L103) [bufpage.c#header-test](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L105-L120) |
| Page writes | `PageSetChecksumCopy()` and `PageSetChecksumInplace()` compute a new `pd_checksum`. | Both return without touching `pd_checksum`. The old value stays in the page while other bytes change, so it soon stops matching. [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) [bufpage.c#PageSetChecksumInplace](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1197-L1205) |
| [Hint-bit](../../../glossary.md#hint-bits) WAL | `XLogHintBitIsNeeded()` is true, so the first hint change to a page after a checkpoint can log a [full-page image](../../../glossary.md#full-page-image). | False, unless `wal_log_hints = on`. Hint changes stop producing those images, which reduces [WAL](../../../glossary.md#wal) volume. [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192) |
| `data_checksums` | `on` | `off` [xlog.c#data_checksums-report](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4783-L4785) |
| `pg_stat_database.checksum_failures` and `checksum_last_failure` | A count, and the time of the last failure | NULL. Both functions return NULL before they read the statistics entry, so previously counted failures are hidden, not reset. [pgstatfuncs.c#checksum_failures](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1529-L1545) [pgstatfuncs.c#checksum_last_failure](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1547-L1566) [system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L878-L879) |
| `pg_checksums --check` | Works | Refused with `data checksums are not enabled in cluster`. [pg_checksums.c#check-refusal](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L523-L528) |

The stale `pd_checksum` values matter if you ever turn checksums back on. Changing only the flag back to 1 would make the server compare stale values on every read. `pg_checksums --enable` avoids that by rewriting every initialized page with a fresh checksum before it publishes the new state, which takes a full read and write of the cluster. [pg_checksums.c#scan-and-publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L607) [ref/pg_checksums.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L48-L52)

Lifecycle of the relevant state:

| Event | State before | State after | Effect on later behavior |
|---|---|---|---|
| `pg_checksums --disable` completes | `pg_control`: version 1; server stopped | `pg_control`: version 0. Every `pd_checksum` is unchanged. | Nothing is visible until a server starts. [pg_checksums.c#publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L607) |
| Postmaster starts | No server | `ControlFile` loaded; `data_checksums = off` | All processes see `DataChecksumsEnabled() == false`. [postmaster.c:964](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L964) [xlog.c#LocalProcessControlFile](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4921-L4927) |
| A page is modified and written | `pd_checksum` matches the page bytes | `pd_checksum` keeps the old value; the bytes differ | The page would fail a checksum comparison if one ran. [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1189) |
| Later `pg_checksums --enable` | Version 0; stale checksums | Every initialized page rewritten; version 1 | Verification resumes with valid values. [pg_checksums.c#scan-and-publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L607) |
| A base backup is taken | Source version 0 | The copy's `pg_control` also says 0, because `pg_control` is sent last with the backup | New standbys cloned from this node start without checksums. [basebackup.c#control-copy](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L351-L357) |

### Replication, rewind, backups and upgrades

**Streaming replication.** Each server verifies and computes checksums according to its own `pg_control`. Disabling on one node does not stop WAL replay, but it leaves the nodes inconsistent. The PostgreSQL 12 documentation warns that tools which copy relation blocks directly, such as `pg_rewind`, can then leave pages with incorrect checksums. It recommends two safe orders. Stop all clusters and switch them all consistently. Or destroy the standbys, switch the primary, and recreate the standbys from scratch. A recreated standby inherits the primary's disabled state through `pg_control` in its base backup. The [mismatched-checksums replication page](../replication-and-wal/physical-replication-mismatched-data-checksums.md) traces both mismatch directions. [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826) [xlogutils.c#XLogReadBufferForRedoExtended](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L345-L407) [ref/pg_checksums.sgml#replication](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L210-L220) [basebackup.c#control-copy](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L351-L357)

**`pg_rewind`.** `pg_rewind` resynchronizes a former primary with the new primary after their timelines diverge, copying only the changed blocks of relation files. [ref/pg_rewind.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/pg_rewind.sgml#L43-L58) It refuses a target whose control file shows neither `data_checksum_version == 1` nor `wal_log_hints = on`. [pg_rewind.c#target-gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429)

- **Default:** `wal_log_hints` is `off`, so a cluster with checksums disabled fails this gate. [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185)
- **Cause:** setting `wal_log_hints = on` in the configuration file. The context is `postmaster`, so the change needs a **restart**.
- **Mechanism:** at startup, `XLogReportParameters()` compares the running `wal_log_hints` with the control file and, when they differ, stores the new value in `pg_control`. `pg_rewind` reads that stored value. Because the hint predicate is `DataChecksumsEnabled() || wal_log_hints`, the setting also restores the hint full-page images, and the WAL they cost, that disabling removed. [xlog.c#XLogReportParameters](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9435-L9484) [xlog.c#XLogReportParameters-call](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7750-L7755) [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192)

**`pg_basebackup`.** The server verifies checksums while sending files only when `DataChecksumsEnabled()` is true. After disabling, base backups never report checksum failures, because no check runs. [basebackup.c#verification-gate](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1413-L1426)

**`pg_upgrade` into PostgreSQL 12.** The pinned `pg_upgrade` requires the old and new clusters to have the same checksum version. It fails with `old cluster uses data checksums but the new one does not` in that direction. You therefore cannot disable checksums by upgrading into a 12 cluster created without `-k`. Instead, upgrade into a 12 cluster created with checksums, stop it cleanly, and run `pg_checksums --disable` on it. The checks that a later major version's `pg_upgrade` applies are not settled by this pin; see [Open Questions](#open-questions). [controldata.c#checksum-match](../../../../raw/postgres-12/src/bin/pg_upgrade/controldata.c#L660-L671)

### Interruption and concurrent-start safety

The control-file write is the last step. If the tool is killed before it, `pg_control` still says checksums are on, and you can simply run the tool again. The documentation states the same. [pg_checksums.c#sync-then-update](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L599) [ref/pg_checksums.sgml#abort](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L221-L226)

The state check does **not** lock out a concurrent server start; the source comment says so explicitly. The tool reads the whole `ControlFileData` at step 2 and writes the whole structure back at step 8. If a server started in between and updated `pg_control`, the tool would overwrite the server's newer fields, such as the state and checkpoint location, with the values it read earlier. Make sure nothing can start the cluster while the tool runs, for example a service manager or a cluster manager. This hazard is derived from the source, not measured. [pg_checksums.c#no-concurrent-guard](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L515) [pg_checksums.c#read-control](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L489-L490) [pg_checksums.c#write-control](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L589-L599) [controldata_utils.c#whole-struct-write](../../../../raw/postgres-12/src/common/controldata_utils.c#L171-L212)

### Other ways to end up without checksums

| Route | How | Trade-off |
|---|---|---|
| `pg_checksums --disable` | As above | Downtime is one clean stop, the sync walk and one start. No page rewrite. |
| New cluster plus logical copy | `initdb` without `-k`; checksums default to off. Then move the data with `pg_dump`/`pg_restore` or logical replication. | A full data copy, which this page does not cover further. [initdb.c#default](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L144) [initdb.c#k-option](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L3146-L3148) |
| `pg_upgrade` into a checksum-less 12 cluster | Refused | See above. [controldata.c#checksum-match](../../../../raw/postgres-12/src/bin/pg_upgrade/controldata.c#L660-L671) |
| `SET`, `ALTER SYSTEM`, configuration file | Impossible | `internal` context, disallowed in files. [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835) |

### Source history

The pinned checkout is a full clone, so these commits come from its own history.

| Commit | Date | Change | First tag containing it |
|---|---|---|---|
| `1fde38beaa0` | 2018-04-05 | Allowed online enabling and disabling of data checksums, and added `pg_verify_checksums`. | — |
| `a228cc13aea` | 2018-04-09 | Reverted the backend side of `1fde38beaa0`, keeping only the read-only offline verifier. | — |
| `6dd263cfaa8` | 2019-03-13 | Renamed `pg_verify_checksums` to `pg_checksums`. | `REL_12_0` |
| `ed308d78379` | 2019-03-23 | Added `--enable` and `--disable`. Its message says `--disable` updates only the control file after the data directory is synced, and that `--check` stays the default for compatibility with PostgreSQL 11's `pg_verify_checksums`. | `REL_12_0` |
| `e0090c86900` | 2019-03-23 | Added `-N` / `--no-sync`. | `REL_12_0` |

The PostgreSQL 12 release notes list the enable/disable ability and state that the cluster must be shut down for it. Therefore, PostgreSQL 12.0 is the first release whose bundled tool can disable checksums in place. [release-12.sgml#pg_checksums-enable-disable](../../../../raw/postgres-12/doc/src/sgml/release-12.sgml#L4324-L4338) [release-12.sgml#rename](../../../../raw/postgres-12/doc/src/sgml/release-12.sgml#L2976-L2987)

### Tests

`src/bin/pg_checksums/t/002_actions.pl` covers the disable path on an offline cluster. It enables checksums, disables them with the default sync, checks that a second `--disable` fails, confirms with `pg_controldata` that the version is 0, and enables them again. It also checks that `--disable --filenode` fails. The only test against a running cluster uses `--check`. No test runs `--disable` against a running cluster, interrupts a disable, or starts a server afterwards to read pages that carry stale checksums. These tests were read, not run. [002_actions.pl#disable](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L116-L154) [002_actions.pl#filenode](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L165-L172) [002_actions.pl#online-check](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L174-L177)

### Causal summary

1. **Initial state:** `pg_control` holds `data_checksum_version = 1`, and every running process derives `DataChecksumsEnabled() == true` from the copy loaded at postmaster start.
2. **Event:** after a clean shutdown, so the state is a shut-down state, `pg_checksums --disable` sets the field to 0. By default it syncs the data directory first and then rewrites and fsyncs `pg_control`. It reads no relation pages.
3. **Transformation:** the next postmaster start loads version 0, so `DataChecksumsEnabled()` is false and `data_checksums` shows `off`.
4. **Interaction:** page reads skip the checksum comparison but keep the header tests. Writes leave `pd_checksum` stale. Hint full-page images stop unless `wal_log_hints = on`. `pg_basebackup` stops verifying, `pg_checksums --check` refuses, and the checksum statistics read NULL.
5. **Observed result:** corruption that only a checksum would catch now goes undetected. `pg_rewind` needs `wal_log_hints = on`, which takes a restart. `pg_upgrade` into 12 still requires matching states.
6. **Later state change:** turning checksums back on requires `pg_checksums --enable`, a full offline rewrite, because the stored checksums went stale.

## Context Reviewed

- Read first: `wiki/versions.md`, `wiki/index.md`, `wiki/v12/index.md`, the last log entries, and the glossary entries Data checksums, Control file, Hint bits, Full-page image, WAL, Checkpoint, Page, GUC context, Base backup, pg_upgrade, Torn page and fsync. PostgreSQL 12 has no common concept pages.
- Primary source: `src/bin/pg_checksums/pg_checksums.c` (option parsing, control-file checks, mode gates, sync and publish), `src/common/controldata_utils.c` (`get_controlfile`, `update_controlfile`), `src/common/file_utils.c` (`fsync_pgdata`, `walkdir`, `fsync_fname`).
- Backend consumers: `xlog.c` (`DataChecksumsEnabled`, `LocalProcessControlFile`, the `data_checksums` report, the shutdown-state writes, `XLogReportParameters`), `postmaster.c`, `bufpage.c` (`PageIsVerified`, `PageSetChecksumCopy`, `PageSetChecksumInplace`), `xlog.h` (`XLogHintBitIsNeeded`), `guc.c` (`data_checksums`, `wal_log_hints`, timeouts), `pgstatfuncs.c`, `system_views.sql`, `pg_controldata.c` (backend and frontend), `pg_proc.dat`, `basebackup.c`.
- Other tools: `pg_rewind.c` target gate, `pg_upgrade/controldata.c` checksum match, `pg_ctl.c` default shutdown mode, `initdb.c` default and `-k`.
- Build boundary: `pg_checksums` compiles `checksum_impl.h` and `bufpage.h` into the frontend, and `PG_CONTROL_VERSION` (1201) is checked at run time. No generated header is involved.
- Docs: `ref/pg_checksums.sgml`, `ref/pg_ctl-ref.sgml`, `ref/initdb.sgml`, `config.sgml`, `release-12.sgml`.
- Tests: `src/bin/pg_checksums/t/002_actions.pl`.
- History: `git log` of `src/bin/pg_checksums/pg_checksums.c` and `src/bin/pg_verify_checksums`, plus `ed308d78379`, `6dd263cfaa8`, `e0090c86900`, `1fde38beaa0` and `a228cc13aea`, with `git tag --contains`. The checkout is not shallow.
- Same-version pages compared: [Consequences of Enabling Data Checksums](enabling-data-checksums-consequences.md), [Physical Replication With Different Data Checksum Settings](../replication-and-wal/physical-replication-mismatched-data-checksums.md), and the health checklist's checksum-counter statement.

## Evidence Map

| Claim | Evidence |
|---|---|
| Disable mode changes only `data_checksum_version` and scans no relation files | [pg_checksums.c#scan-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L544-L561), [pg_checksums.c#publish](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L607), [ref/pg_checksums.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L48-L52) |
| No SQL or configuration route | [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835), revert `a228cc13aea` |
| Clean-shutdown requirement, including a standby shut down in recovery | [pg_checksums.c#state-gate](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L521), [xlog.c#shutdown-state](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8853-L8855), [xlog.c#restartpoint-state](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9126-L9130) [xlog.c#restartpoint-state-normal](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9211-L9212) |
| Already-disabled and filenode refusals | [pg_checksums.c#already-disabled](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L530-L535), [pg_checksums.c#filenode-restriction](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L480-L487) |
| Sync by default; `-N` skips it and the `pg_control` fsync; sync errors are not fatal | [pg_checksums.c:40](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L40), [pg_checksums.c#sync-and-update](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L592-L599), [controldata_utils.c#fsync](../../../../raw/postgres-12/src/common/controldata_utils.c#L232-L249), [file_utils.c#walkdir-ignores-result](../../../../raw/postgres-12/src/common/file_utils.c#L187-L204) |
| Server reads the state only at startup | [postmaster.c:964](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L964), [xlog.c#LocalProcessControlFile](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4921-L4927), [xlog.c#data_checksums-report](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4783-L4785) |
| Reads skip comparison, header tests remain; writes leave `pd_checksum` stale | [bufpage.c#checksum-compare](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L95-L120), [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1169-L1205) |
| Hint full-page images stop unless `wal_log_hints` | [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192), [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185) |
| Statistics columns read NULL | [pgstatfuncs.c](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1529-L1566) |
| `pg_rewind` gate and `wal_log_hints` recording | [pg_rewind.c#target-gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429), [xlog.c#XLogReportParameters](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9435-L9484) |
| Base backup verification gate and `pg_control` copy | [basebackup.c#verification-gate](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1413-L1426), [basebackup.c#control-copy](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L351-L357) |
| `pg_upgrade` into 12 requires matching state | [controldata.c#checksum-match](../../../../raw/postgres-12/src/bin/pg_upgrade/controldata.c#L660-L671) |
| Interruption safety; no concurrent-start guard | [pg_checksums.c#sync-then-update](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L582-L599), [pg_checksums.c#no-concurrent-guard](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L511-L515), [ref/pg_checksums.sgml#abort](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L221-L226) |
| Replication guidance | [ref/pg_checksums.sgml#replication](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L210-L220) |
| History and first release | `ed308d78379`, `6dd263cfaa8`, `e0090c86900`, `1fde38beaa0`, `a228cc13aea`; [release-12.sgml](../../../../raw/postgres-12/doc/src/sgml/release-12.sgml#L4324-L4338) |
| Test coverage and gaps | [002_actions.pl](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L116-L177) |

## Open Questions

- **Later-release `pg_upgrade`.** This pin's `pg_upgrade` governs upgrades into PostgreSQL 12 only. Whether a later release's `pg_upgrade` accepts an old 12 cluster with checksums disabled into a new cluster with them enabled, or the reverse, cannot be established from this checkout.
- **Documentation omits the sync walk.** `ref/pg_checksums.sgml` says disabling "only updates the file `pg_control`". The source also, by default, fsyncs the whole data directory, `pg_wal` and the tablespaces first. That walk modifies nothing, so this is an omission rather than a contradiction, but it is the step whose duration grows with cluster size. [ref/pg_checksums.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L48-L52) [pg_checksums.c#sync-and-update](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L592-L599)
- **Not measured.** No cluster was started for this page. The duration of the sync walk, the WAL reduction from fewer hint images, and the concurrent-start overwrite are derived from source only.

## Source References

- [pg_checksums.c](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L1-L610)
- [controldata_utils.c#get_controlfile](../../../../raw/postgres-12/src/common/controldata_utils.c#L50-L144)
- [controldata_utils.c#update_controlfile](../../../../raw/postgres-12/src/common/controldata_utils.c#L155-L262)
- [file_utils.c#fsync_pgdata](../../../../raw/postgres-12/src/common/file_utils.c#L58-L114)
- [file_utils.c#walkdir](../../../../raw/postgres-12/src/common/file_utils.c#L150-L205)
- [file_utils.c#fsync_fname](../../../../raw/postgres-12/src/common/file_utils.c#L257-L304)
- [xlog.c#DataChecksumsEnabled](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4818-L4826)
- [xlog.c#LocalProcessControlFile](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L4921-L4927)
- [xlog.c#XLogReportParameters](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9435-L9484)
- [xlog.h#XLogHintBitIsNeeded](../../../../raw/postgres-12/src/include/access/xlog.h#L183-L192)
- [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L162)
- [bufpage.c#PageSetChecksumCopy](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L1155-L1205)
- [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835)
- [guc.c#wal_log_hints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1177-L1185)
- [pgstatfuncs.c#checksum-functions](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1529-L1566)
- [pg_rewind.c#target-gate](../../../../raw/postgres-12/src/bin/pg_rewind/pg_rewind.c#L422-L429)
- [controldata.c#checksum-match](../../../../raw/postgres-12/src/bin/pg_upgrade/controldata.c#L660-L671)
- [basebackup.c#verification-gate](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1413-L1426)
- [pg_controldata.c#checksum-line](../../../../raw/postgres-12/src/bin/pg_controldata/pg_controldata.c#L343-L344)
- [pg_ctl.c#default-mode](../../../../raw/postgres-12/src/bin/pg_ctl/pg_ctl.c#L82)
- [ref/pg_checksums.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L36-L227)
- [release-12.sgml#pg_checksums](../../../../raw/postgres-12/doc/src/sgml/release-12.sgml#L4324-L4338)
- [002_actions.pl](../../../../raw/postgres-12/src/bin/pg_checksums/t/002_actions.pl#L89-L177)

## Navigation

- [v12/index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Consequences of Enabling Data Checksums in PostgreSQL 12 (unverified)](enabling-data-checksums-consequences.md)
- [Physical Replication With Different Data Checksum Settings in PostgreSQL 12 (unverified)](../replication-and-wal/physical-replication-mismatched-data-checksums.md)
