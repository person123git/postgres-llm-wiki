---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# PostgreSQL 12 Database Health Checklist (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [How to use this checklist](#how-to-use-this-checklist)
  - [Production query guardrails](#production-query-guardrails)
  - [Observability prerequisites](#observability-prerequisites)
  - [SQL health checklist](#sql-health-checklist)
  - [Database log checklist](#database-log-checklist)
  - [Issue interpretation and response](#issue-interpretation-and-response)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Prompt note: The user approved correcting typos and grammar before filing.

Produce a document with a checklist to check the health of the database. Include
what to check in the database log. For each possible issue found in the
checklist and/or log, explain the repercussions of the finding and possible root
causes. For each log item or internal stats data point, explain what needs to be
enabled, installed, or configured to produce the data.

## Answer

Use this as a layered health check. First confirm that PostgreSQL is collecting
the required data. Then inspect current activity, cumulative counters,
maintenance state, WAL and replication state, and optional query or bloat
extensions. Finally, correlate the findings with the server log. PostgreSQL 12
provides built-in dynamic views for current sessions and replication, and
collected views for database, table, archiver, and background-writer counters.
[monitoring.sgml#DynamicStatisticsViews](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L284-L360)
[monitoring.sgml#CollectedStatisticsViews](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L375-L425)

Treat cumulative values as rates over a known interval, not as isolated raw
totals. In `pg_stat_database`, only `numbackends` is current state; the other
columns accumulate until `stats_reset`.
[monitoring.sgml#pg_stat_database](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2510-L2528)
[monitoring.sgml#pg_stat_database_stats_reset](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2625-L2638)

### How to use this checklist

1. Run the observability query and resolve disabled, missing, or
   `pending_restart` settings before interpreting empty data.
2. Run the core SQL checks in every application database. Run cluster-wide
   checks such as `pg_database`, `pg_stat_bgwriter`, `pg_stat_archiver`, and
   replication checks once from a database where the monitoring role can read
   them.
3. Compare the result with a previous capture that has the same
   `stats_reset` timestamp. The database, table, archiver, and background-writer
   views expose accumulated counters, while `pg_stat_activity` and replication
   views expose current state.
   [monitoring.sgml#DynamicStatisticsViews](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L294-L360)
   [monitoring.sgml#CollectedStatisticsViews](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L386-L425)
4. Search the server log over the same interval and correlate sessions by PID
   or session identifier. PostgreSQL recommends including PID or session ID in
   `log_line_prefix` when statement and duration messages need to be joined.
   [config.sgml#log_min_duration_statement](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5931-L5968)

### Production query guardrails

Start a diagnostic session with finite timeouts:

```sql
SET /* wiki_db_health_guardrails */ statement_timeout = '30s';
SET /* wiki_db_health_guardrails */ lock_timeout = '2s';
```

Both settings are `PGC_USERSET`, so they are session/transaction-scope changes;
zero disables each timeout.
[guc.c#statement_timeout-lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396)
[guc.c#GucContext_Names](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L594-L603)

The catalog and statistics queries below do not intentionally take
table-changing locks. The optional `pgstattuple` checks read relation pages and
acquire a read lock; their results can also change during concurrent updates.
Run those checks only for selected relations, preferably outside peak traffic,
and raise `statement_timeout` only for that diagnostic session if required.
[pgstattuple.sgml#pgstattuple-locking](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L123-L142)

### Observability prerequisites

| Data or signal | What must be enabled, installed, or configured | Apply scope |
|---|---|---|
| PostgreSQL-managed text or CSV log files | Use `stderr` or `csvlog` in `log_destination`. CSV requires `logging_collector`. `current_logfiles` records the active file path when the collector manages `stderr` or `csvlog`. If the deployment uses `syslog` or Windows `eventlog`, inspect that destination instead. [config.sgml#log_destination](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5439-L5486) | `logging_collector` is `PGC_POSTMASTER`: restart. `log_destination` is `PGC_SIGHUP`: reload. [guc.c#logging_collector](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1617-L1623) [guc.c#log_destination](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3849-L3858) |
| Correlatable log lines | Configure `log_line_prefix` with at least PID or a session identifier so statement, duration, lock, and disconnect messages can be joined. [config.sgml#log_min_duration_statement](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5956-L5968) | `log_line_prefix` is `PGC_SIGHUP`: reload. [guc.c#log_line_prefix](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3593-L3599) |
| Current sessions, transaction age, query text, and waits | Keep `track_activities` enabled. `pg_stat_activity` exposes session state, transaction/query start times, waits, query text, and `backend_xmin`. Increase `track_activity_query_size` if 1024 bytes is too short for useful query text. [system_views.sql#pg_stat_activity](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L756) [config.sgml#track_activity_query_size](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6820-L6834) | `track_activities` is `PGC_SUSET`: superuser session/transaction scope. `track_activity_query_size` is `PGC_POSTMASTER`: restart. [guc.c#track_activities](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1382-L1390) [guc.c#track_activity_query_size](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3164-L3172) |
| Database and table counters used by autovacuum and health checks | Keep `track_counts` enabled. It enables database activity statistics and is required for normal autovacuum operation. [config.sgml#track_counts](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6838-L6850) [config.sgml#autovacuum](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6985-L7005) | `track_counts` is `PGC_SUSET`: superuser session/transaction scope. Configure it as the cluster default when all sessions and autovacuum must collect data. [guc.c#track_counts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1399) |
| Database, `EXPLAIN (BUFFERS)`, and `pg_stat_statements` I/O time | Enable `track_io_timing` when the platform overhead is acceptable. It is off by default and supplies `blk_read_time` and `blk_write_time`. [config.sgml#track_io_timing](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6854-L6871) | `track_io_timing` is `PGC_SUSET`: superuser session/transaction scope. Set it as a cluster default if complete workload timing is required. [guc.c#track_io_timing](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1401-L1408) |
| Slow statements, lock waits, temp files, checkpoints, autovacuum, connection lifecycle, and replication commands in the log | Configure the corresponding logging GUC. `log_min_duration_statement`, `log_lock_waits`, `log_temp_files`, and `log_replication_commands` are disabled by default. `log_checkpoints`, `log_connections`, `log_disconnections`, and `log_autovacuum_min_duration` are also disabled by default. [config.sgml#log_min_duration_statement](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5931-L5949) [config.sgml#log_checkpoints-connections-disconnections](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6165-L6222) [config.sgml#log_lock_waits-temp_files](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6477-L6574) [config.sgml#log_autovacuum_min_duration](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7010-L7033) | `log_checkpoints` and `log_autovacuum_min_duration` are `PGC_SIGHUP`: reload. `log_connections` and `log_disconnections` are `PGC_SU_BACKEND`: superuser session-start settings that cannot change inside a session. The remaining listed logging GUCs are `PGC_SUSET`: superuser session/transaction scope; set cluster defaults for cluster-wide coverage. [guc.c#log_checkpoints-connections-replication](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1217-L1251) [guc.c#log_lock_waits](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1489-L1496) [guc.c#log_duration-settings](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2703-L2724) [guc.c#log_temp_files](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3153-L3161) |
| Per-statement workload totals | Install the PostgreSQL 12 `pg_stat_statements` contrib module, add it to `shared_preload_libraries`, restart, and run `CREATE /* wiki_enable_pgss */ EXTENSION pg_stat_statements;` in each database where its SQL view is needed. [pgstatstatements.sgml#loading](../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L10-L30) | `shared_preload_libraries` and `pg_stat_statements.max` require restart. `pg_stat_statements.track` and `track_utility` are superuser session/transaction settings; `pg_stat_statements.save` is a reload setting. [guc.c#shared_preload_libraries](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3768-L3775) [pgstatstatements.sgml#settings](../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L399-L465) |
| Exact or approximate heap bloat and B-tree structure | Install the PostgreSQL 12 `pgstattuple` contrib module and run `CREATE /* wiki_enable_pgstattuple */ EXTENSION pgstattuple;` in the target database. Use a superuser or grant the functions, normally through `pg_stat_scan_tables`. [pgstattuple.sgml#module-and-privileges](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L23) [pgstattuple--1.4.sql#extension-install](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L1-L20) | The extension creation step is per database. The checked SQL surface loads its C functions when the extension is created. [pgstattuple--1.4.sql#functions](../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L3-L20) |
| Checksum failures | `data_checksums` reports whether checksums are enabled. Enable them during `initdb`, or only after a clean shutdown with `pg_checksums --enable`; the offline tool scans or rewrites cluster files. `pg_stat_database.checksum_failures` is `NULL` when checksums are disabled. [config.sgml#data_checksums](../../../raw/postgres-12/doc/src/sgml/config.sgml#L9088-L9100) [initdb.sgml#data-checksums](../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L222) [pg_checksums.sgml#operation](../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L38-L51) [monitoring.sgml#checksum_failures](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2599-L2611) | This is not a live GUC change. Stop the cluster cleanly before using `pg_checksums`; keep all replication nodes consistent. [pg_checksums.sgml#notes](../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L204-L225) |
| Archiving, physical replication, logical replication, and slots | Archiver data becomes operationally meaningful when `archive_mode` is enabled and `archive_command` succeeds. Replication views require the corresponding WAL sender, WAL receiver, subscription worker, or slot to exist. [config.sgml#archive_mode-command](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3072-L3128) [monitoring.sgml#replication-views](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L307-L327) [catalogs.sgml#pg_replication_slots](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9835-L9850) | `archive_mode`, `max_wal_senders`, and `max_replication_slots` require restart. `archive_command` reloads. [guc.c#archive_mode](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4364-L4370) [guc.c#archive_command](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3455-L3461) [guc.c#max_wal_senders-slots](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2636-L2653) |

Check the effective settings before collecting evidence:

```sql
SELECT /* wiki_db_health_observability */
       name,
       setting,
       unit,
       context,
       source,
       pending_restart
FROM pg_settings
WHERE name IN (
    'archive_command',
    'archive_mode',
    'autovacuum',
    'data_checksums',
    'log_autovacuum_min_duration',
    'log_checkpoints',
    'log_connections',
    'log_destination',
    'log_disconnections',
    'log_line_prefix',
    'log_lock_waits',
    'log_min_duration_statement',
    'log_replication_commands',
    'log_temp_files',
    'logging_collector',
    'max_connections',
    'max_replication_slots',
    'max_wal_senders',
    'shared_preload_libraries',
    'superuser_reserved_connections',
    'track_activities',
    'track_activity_query_size',
    'track_counts',
    'track_io_timing'
)
ORDER BY name;
```

`pg_settings` is backed by `pg_show_all_settings()`, and `pending_restart`
identifies a file change that has not taken effect because a restart is still
required.
[system_views.sql#pg_settings](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L512-L524)
[catalogs.sgml#pg_settings_pending_restart](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10494-L10498)

### SQL health checklist

**1. Current sessions, old transactions, and waits**

```sql
SELECT /* wiki_db_health_activity_summary */
       state,
       wait_event_type,
       wait_event,
       count(*) AS sessions,
       min(xact_start) AS oldest_xact_start,
       min(query_start) FILTER (WHERE state = 'active') AS oldest_active_query_start
FROM pg_stat_activity
GROUP BY state, wait_event_type, wait_event
ORDER BY sessions DESC, state, wait_event_type, wait_event;

SELECT /* wiki_db_health_long_transactions */
       pid,
       usename,
       application_name,
       state,
       now() - xact_start AS transaction_age,
       backend_xmin,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

`pg_stat_activity` defines `xact_start`, `query_start`, state-change time, and
wait-event fields. A `Lock` wait is a heavyweight lock wait; a `BufferPin` wait
can be prolonged by another process holding an open cursor.
[monitoring.sgml#pg_stat_activity-timestamps-and-waits](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L659-L718)

Checklist:

- [ ] Compare total sessions with `max_connections` and preserve the
  `superuser_reserved_connections` margin. PostgreSQL emits `sorry, too many
  clients already` when backend process slots are exhausted and separately
  reserves the last configured slots for superusers.
  [guc.c#max_connections-reserved](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2130-L2147)
  [proc.c#InitProcess](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L330-L363)
  [postinit.c#reserved-connections](../../../raw/postgres-12/src/backend/utils/init/postinit.c#L818-L828)
- [ ] Investigate old active or `idle in transaction` sessions, especially rows
  with old `backend_xmin`. VACUUM warns when the oldest xmin is far in the past
  and points to open transactions, old prepared transactions, and stale
  replication slots as possible causes.
  [vacuum.c#vacuum_set_xid_limits](../../../raw/postgres-12/src/backend/commands/vacuum.c#L933-L949)
- [ ] Investigate sustained `Lock`, `BufferPin`, or `IO` waits. These identify
  different wait classes and should be correlated with the lock and log checks
  below.
  [monitoring.sgml#wait_event_type](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L687-L760)

**2. Blocking locks and deadlocks**

```sql
SELECT /* wiki_db_health_locks */
       a.pid,
       a.usename,
       a.application_name,
       a.state,
       a.wait_event_type,
       a.wait_event,
       now() - a.query_start AS query_age,
       l.locktype,
       l.mode,
       l.granted,
       a.query
FROM pg_stat_activity AS a
JOIN pg_locks AS l ON l.pid = a.pid
WHERE a.wait_event_type = 'Lock'
   OR NOT l.granted
ORDER BY l.granted, query_age DESC;
```

Checklist:

- [ ] Treat ungranted locks and long `Lock` waits as blocked work. Enable
  `log_lock_waits` to obtain holder and waiter PIDs after `deadlock_timeout`.
  [config.sgml#log_lock_waits](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6477-L6490)
  [proc.c#lock-wait-log](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1377-L1497)
- [ ] Treat any increase in `pg_stat_database.deadlocks` as an application or
  operational concurrency defect. PostgreSQL detects the deadlock, reports it
  to statistics, and aborts one transaction with `deadlock detected`.
  [deadlock.c#DeadLockReport](../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L1139-L1146)
  [monitoring.sgml#deadlocks](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2594-L2598)

**3. Database-wide counters**

```sql
SELECT /* wiki_db_health_database_counters */
       datname,
       numbackends,
       xact_commit,
       xact_rollback,
       conflicts,
       temp_files,
       temp_bytes,
       deadlocks,
       checksum_failures,
       checksum_last_failure,
       blk_read_time,
       blk_write_time,
       stats_reset
FROM pg_stat_database
ORDER BY datname NULLS LAST;

SELECT /* wiki_db_health_standby_conflicts */
       datname,
       confl_tablespace,
       confl_lock,
       confl_snapshot,
       confl_bufferpin,
       confl_deadlock
FROM pg_stat_database_conflicts
ORDER BY datname;
```

Checklist:

- [ ] Capture deltas for rollbacks, temporary-file counts and bytes, deadlocks,
  checksum failures, and I/O time. Temporary-file counters include all temp
  files regardless of `log_temp_files`; I/O time requires `track_io_timing`;
  checksum counters are `NULL` when checksums are disabled.
  [monitoring.sgml#pg_stat_database-counters](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2524-L2628)
- [ ] On a standby, capture recovery-conflict deltas by reason. The view records
  cancellations caused by tablespace drops, lock timeouts, old snapshots,
  buffer pins, and deadlocks; it contains useful data only on standby servers.
  [monitoring.sgml#pg_stat_database_conflicts](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2640-L2702)

**4. Vacuum, analyze, dead rows, and progress**

```sql
SELECT /* wiki_db_health_table_maintenance */
       schemaname,
       relname,
       n_live_tup,
       n_dead_tup,
       n_mod_since_analyze,
       last_autovacuum,
       last_autoanalyze,
       autovacuum_count,
       autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC, n_mod_since_analyze DESC
LIMIT 50;

SELECT /* wiki_db_health_vacuum_progress */
       pid,
       datname,
       relid::regclass AS relation,
       phase,
       heap_blks_total,
       heap_blks_scanned,
       heap_blks_vacuumed,
       index_vacuum_count,
       max_dead_tuples,
       num_dead_tuples
FROM pg_stat_progress_vacuum
ORDER BY pid;
```

`pg_stat_user_tables` exposes estimated live/dead rows, modifications since
analyze, and manual/automatic vacuum and analyze timestamps and counts.
`pg_stat_progress_vacuum` exposes one row for each running regular or autovacuum
worker, but not `VACUUM FULL`.
[monitoring.sgml#pg_stat_all_tables](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2704-L2845)
[monitoring.sgml#pg_stat_progress_vacuum](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3742-L3788)

Checklist:

- [ ] Investigate tables whose dead-row or modification counts grow while
  autovacuum/analyze timestamps do not advance. Autovacuum compares dead tuples
  and changes-since-analyze with base-plus-scale-factor thresholds, skips
  per-table-disabled relations unless wraparound forces a vacuum, and skips
  ordinary threshold work when statistics are unavailable.
  [autovacuum.c#relation_needs_vacanalyze](../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2925-L2955)
  [autovacuum.c#threshold-decision](../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2995-L3080)
- [ ] Confirm `autovacuum` and `track_counts` are enabled before treating missing
  automatic maintenance as a table-level problem. `autovacuum` is a reload
  setting; per-table storage parameters can still disable normal autovacuum.
  [config.sgml#autovacuum](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6985-L7005)
  [guc.c#autovacuum](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426-L1432)

**5. Transaction-ID and MultiXact age**

```sql
SELECT /* wiki_db_health_database_xid_age */
       datname,
       age(datfrozenxid) AS xid_age
FROM pg_database
ORDER BY xid_age DESC;

SELECT /* wiki_db_health_relation_xid_mxid_age */
       c.oid::regclass AS relation,
       greatest(age(c.relfrozenxid), age(t.relfrozenxid)) AS xid_age,
       greatest(mxid_age(c.relminmxid), mxid_age(t.relminmxid)) AS mxid_age
FROM pg_class AS c
LEFT JOIN pg_class AS t ON c.reltoastrelid = t.oid
WHERE c.relkind IN ('r', 'm')
ORDER BY xid_age DESC NULLS LAST, mxid_age DESC NULLS LAST
LIMIT 50;
```

PostgreSQL stores table and database freeze cutoffs in `relfrozenxid` and
`datfrozenxid`; `age()` measures transactions from that cutoff. MultiXact IDs
represent multiple concurrent row lockers, and `mxid_age(relminmxid)` measures
their age.
[maintenance.sgml#xid-age-queries](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L557-L585)
[maintenance.sgml#multixact-age](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L655-L686)

Checklist:

- [ ] Alert before ages approach the configured
  `autovacuum_freeze_max_age` or `autovacuum_multixact_freeze_max_age`.
  PostgreSQL forces anti-wraparound vacuum at those ages even if normal
  autovacuum is disabled; both max-age settings require restart.
  [config.sgml#autovacuum-freeze-max-age](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7156-L7179)
  [config.sgml#autovacuum-multixact-freeze-max-age](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7184-L7208)
- [ ] Escalate immediately if the log says a database must be vacuumed within a
  stated transaction count. Ignoring the warning eventually causes PostgreSQL
  to refuse new commands to avoid wraparound data loss.
  [maintenance.sgml#wraparound-warning-shutdown](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L608-L635)

**6. Checkpoints and background writes**

```sql
SELECT /* wiki_db_health_bgwriter */
       checkpoints_timed,
       checkpoints_req,
       checkpoint_write_time,
       checkpoint_sync_time,
       buffers_checkpoint,
       buffers_clean,
       maxwritten_clean,
       buffers_backend,
       buffers_backend_fsync,
       buffers_alloc,
       stats_reset
FROM pg_stat_bgwriter;
```

Checklist:

- [ ] Compare requested checkpoints with timed checkpoints over the same
  interval. `checkpoints_req` counts requested checkpoints, while the view also
  separates checkpoint write/sync time and backend versus background-writer
  writes.
  [monitoring.sgml#pg_stat_bgwriter](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2399-L2484)
- [ ] Investigate repeated `checkpoints are occurring too frequently` messages.
  PostgreSQL emits that warning for WAL-caused checkpoints inside
  `checkpoint_warning` and explicitly suggests increasing `max_wal_size`.
  Both settings are reload settings.
  [checkpointer.c#CheckpointerMain](../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462)
  [guc.c#checkpoint-settings](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2555-L2587)
- [ ] Enable `log_checkpoints` when buffer counts and checkpoint write/sync
  duration are needed per checkpoint, rather than only as cumulative counters.
  [config.sgml#log_checkpoints](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6165-L6178)
  [xlog.c#LogCheckpointEnd](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8408-L8435)

**7. Archiving, replication, and retained WAL**

```sql
SELECT /* wiki_db_health_archiver */
       archived_count,
       last_archived_wal,
       last_archived_time,
       failed_count,
       last_failed_wal,
       last_failed_time,
       stats_reset
FROM pg_stat_archiver;

SELECT /* wiki_db_health_replication */
       pid,
       application_name,
       client_addr,
       backend_xmin,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag,
       sync_state,
       reply_time
FROM pg_stat_replication
ORDER BY application_name, pid;

SELECT /* wiki_db_health_replication_slots */
       slot_name,
       slot_type,
       database,
       active,
       active_pid,
       xmin,
       catalog_xmin,
       restart_lsn,
       confirmed_flush_lsn
FROM pg_replication_slots
ORDER BY slot_name;

SELECT /* wiki_db_health_wal_receiver */ *
FROM pg_stat_wal_receiver;

SELECT /* wiki_db_health_subscription */ *
FROM pg_stat_subscription
ORDER BY subname, relid;
```

Checklist:

- [ ] Investigate increasing `failed_count`, an old `last_archived_time`, or an
  archive warning in the log. The archiver reports success/failure to the
  statistics collector, and failed or missing archiving causes old WAL to
  accumulate in `pg_wal`.
  [pgarch.c#pgarch_ArchiverCopyLoop](../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L466-L539)
  [wal.sgml#WAL-retention](../../../raw/postgres-12/doc/src/sgml/wal.sgml#L579-L588)
- [ ] Investigate send/write/flush/replay gaps and lag over time. The LSN columns
  identify the last positions sent, written, flushed, and replayed; the lag
  columns measure recent stage delay and are not predictions of catch-up time.
  [monitoring.sgml#pg_stat_replication](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1877-L2020)
- [ ] Investigate inactive slots with old `restart_lsn`, `xmin`, or
  `catalog_xmin`. A slot can retain WAL without a built-in size bound and can
  prevent VACUUM from removing rows still required by the consumer.
  [catalogs.sgml#pg_replication_slots-columns](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9914-L9968)
  [high-availability.sgml#replication-slot-retention](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L914-L930)
- [ ] On synchronous replication, correlate commit latency and lock contention
  with `sync_state` and lag. Transactions retain their locks until the required
  transfer is confirmed.
  [high-availability.sgml#synchronous-replication-impact](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L1238-L1244)

**8. Optional query and relation diagnostics**

```sql
SELECT /* wiki_db_health_pgss */
       queryid,
       calls,
       total_time,
       rows,
       shared_blks_hit,
       shared_blks_read,
       temp_blks_written,
       blk_read_time,
       blk_write_time,
       left(query, 160) AS query_sample
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;

SELECT /* wiki_db_health_pgstattuple */
       *
FROM pgstattuple_approx('schema.table_name'::regclass);

SELECT /* wiki_db_health_pgstatindex */
       *
FROM pgstatindex('schema.index_name'::regclass);
```

Checklist:

- [ ] In `pg_stat_statements`, investigate statements dominating total time,
  reads, or temporary writes. The module groups statistics by database, user,
  and query ID; its I/O timing fields require `track_io_timing`.
  [pgstatstatements.sgml#view](../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L22-L41)
  [config.sgml#track_io_timing](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6867-L6871)
- [ ] Use `pgstattuple_approx` to shortlist heap relations with large dead-tuple
  or free-space fractions. Use exact `pgstattuple` only when the extra scan is
  justified.
  [pgstattuple.sgml#pgstattuple](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L39-L58)
  [pgstattuple.sgml#pgstattuple_approx](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L484-L506)
- [ ] Use `pgstatindex` for a selected B-tree index when index size, deleted or
  empty pages, average leaf density, and leaf fragmentation are needed.
  [pgstattuple.sgml#pgstatindex](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L186)

### Database log checklist

The table below assumes the server log destination is already accessible. The
"possible root causes" column contains operational inferences constrained by
the cited PostgreSQL mechanism; confirm them with current activity, counters,
and operating-system evidence.

| Log pattern or event | Repercussion | Possible root causes and next checks | Required data/configuration |
|---|---|---|---|
| `sorry, too many clients already` or `remaining connection slots are reserved...` | New clients are refused; non-superusers can be locked out while reserved slots remain. [proc.c#InitProcess](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L347-L363) [postinit.c#reserved-connections](../../../raw/postgres-12/src/backend/utils/init/postinit.c#L818-L828) | Compare `pg_stat_activity` with `max_connections`; check connection-pool behavior, abandoned sessions, and traffic bursts. Raising `max_connections` or `superuser_reserved_connections` requires restart. [guc.c#max_connections-reserved](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2130-L2147) | The FATAL messages are built in. Enable `log_connections` and `log_disconnections` at session start to add connection-attempt and session-duration context. [config.sgml#log_connections-disconnections](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6182-L6222) |
| `duration: ... statement: ...` or repeated slow-duration records | User latency, resource occupancy, and longer lock holding can rise while slow statements run. PostgreSQL logs a completed statement when its duration reaches `log_min_duration_statement`. [config.sgml#log_min_duration_statement](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5931-L5949) [postgres.c#check_log_duration](../../../raw/postgres-12/src/backend/tcop/postgres.c#L2194-L2247) | Correlate with `pg_stat_statements`, lock waits, I/O timing, and temp-file messages. Possible mechanisms are expensive execution, lock waits, I/O, or sort/hash spills. | Set `log_min_duration_statement` as a superuser session/transaction setting; use a cluster default for full coverage. `0` logs all completed statements and `-1` disables it. [guc.c#log_min_duration_statement](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2703-L2712) |
| `temporary file: path ..., size ...` or `temporary file size exceeds temp_file_limit` | Temp I/O consumes storage and can fail a statement when the per-process limit is exceeded. [fd.c#ReportTemporaryFileUsage](../../../raw/postgres-12/src/backend/storage/file/fd.c#L1272-L1286) [fd.c#temp_file_limit](../../../raw/postgres-12/src/backend/storage/file/fd.c#L1942-L1956) | PostgreSQL creates temporary files for sorts, hashes, and temporary query results. Correlate the PID with the statement and compare `pg_stat_database.temp_bytes` and `pg_stat_statements.temp_blks_written`. [config.sgml#log_temp_files](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6555-L6574) | `log_temp_files` and `temp_file_limit` are superuser session/transaction settings. `log_temp_files = 0` logs every temp file; `-1` disables logging. [guc.c#log_temp_files](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3153-L3161) [guc.c#temp_file_limit](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2271-L2278) |
| `process ... still waiting for ...`, `acquired ... after ...`, or `detected deadlock while waiting` | Statements are delayed by lock contention; a hard deadlock aborts one transaction. [proc.c#lock-wait-log](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1463-L1497) [deadlock.c#DeadLockReport](../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L1139-L1146) | Use the holder/waiter PIDs, `pg_locks`, and `pg_stat_activity`. Source-visible mechanisms include conflicting table or row locks and inconsistent lock acquisition order; PostgreSQL recommends consistent ordering and retrying transactions aborted by deadlocks. [mvcc.sgml#Deadlocks](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1358-L1423) | Enable `log_lock_waits` as a superuser session/transaction setting. It logs waits that exceed `deadlock_timeout`. [config.sgml#log_lock_waits](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6477-L6490) |
| `canceling statement due to lock timeout`, `canceling statement due to statement timeout`, `terminating connection due to idle-in-transaction timeout`, or `canceling statement due to conflict with recovery` | PostgreSQL cancels the statement, terminates the idle transaction session, or cancels a standby query. The application must handle the error and may need to retry the transaction. [postgres.c#ProcessInterrupts](../../../raw/postgres-12/src/backend/tcop/postgres.c#L3070-L3136) | Check whether the configured limit is intentionally lower than normal workload duration; for recovery conflicts, use `pg_stat_database_conflicts` to identify lock, snapshot, buffer-pin, tablespace, or deadlock causes. [monitoring.sgml#pg_stat_database_conflicts](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2640-L2702) | `statement_timeout` and `lock_timeout` are user session/transaction settings. A zero value disables them. Recovery-conflict reporting is built in on a standby. [guc.c#statement_timeout-lock_timeout](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396) |
| `automatic vacuum/analyze...`, autovacuum skipped due to a conflicting lock, `oldest xmin is far in the past`, or `oldest multixact is far in the past` | Dead rows and stale statistics can persist; old horizons can prevent freezing and move the cluster toward XID or MultiXact wraparound. [config.sgml#log_autovacuum_min_duration](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7010-L7033) [vacuum.c#old-horizon-warnings](../../../raw/postgres-12/src/backend/commands/vacuum.c#L933-L988) | Check per-table autovacuum reloptions and thresholds, long transactions, prepared transactions, stale slots, and row-lock-heavy workloads. [autovacuum.c#relation_needs_vacanalyze](../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2932-L2955) [vacuum.c#vacuum_set_xid_limits](../../../raw/postgres-12/src/backend/commands/vacuum.c#L942-L988) | Set `log_autovacuum_min_duration` to a nonnegative value and reload; `0` logs all actions, while `-1` disables the log. Keep `autovacuum` and `track_counts` enabled. [guc.c#log_autovacuum_min_duration](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2715-L2724) [config.sgml#autovacuum](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6985-L7005) |
| `database ... must be vacuumed within ... transactions` or `database is not accepting commands to avoid wraparound data loss` | This is an emergency. Ignored warnings progress to refusal of new commands, while uncontrolled wraparound would make old data appear to be in the future. [maintenance.sgml#wraparound-impact](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L391-L405) [maintenance.sgml#wraparound-warning-shutdown](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L608-L635) | Autovacuum has not advanced old XIDs. Check blocked/failed anti-wraparound vacuum, old transactions, prepared transactions, stale slots, and affected database/table ages. | The warnings are built in; no optional logging GUC produces them. Ensure the configured log destination captures WARNING and ERROR messages. |
| `checkpoint starting`, `checkpoint complete`, or `checkpoints are occurring too frequently` | Frequent requested checkpoints increase write pressure; long write/sync phases can align with latency spikes. The frequency warning is specifically tied to WAL-caused checkpoints. [xlog.c#checkpoint-logging](../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8706-L8712) [checkpointer.c#checkpoint-warning](../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462) | Compare `checkpoints_req`, checkpoint times, WAL generation, and `max_wal_size`. The built-in warning recommends increasing `max_wal_size`. | Enable `log_checkpoints` and reload for start/complete detail. `checkpoint_warning` and `max_wal_size` are reload settings. [config.sgml#log_checkpoints](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6165-L6178) [guc.c#checkpoint-settings](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2555-L2587) |
| `archive_mode enabled, yet archive_command is not set`, repeated archive-command failures, or `archiving write-ahead log file ... failed too many times` | Required WAL is not archived; WAL can accumulate in `pg_wal`, consume disk, and break the intended archive chain or recovery-point objective. [pgarch.c#archive-failures](../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L466-L539) [config.sgml#archive_command](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3102-L3132) | Check an empty or failing `archive_command`, archive destination capacity/permissions, command exit status, and whether WAL generation exceeds archive throughput. | `archive_mode` requires restart. `archive_command` reloads and must return zero only on success. Inspect `pg_stat_archiver` without installing an extension. [config.sgml#archive_mode-command](../../../raw/postgres-12/doc/src/sgml/config.sgml#L3072-L3128) |
| `could not extend file ... Check free disk space` or a short-write variant | The statement that needs relation growth fails. Continued lack of space can also prevent WAL, temp, or other database writes. [md.c#mdextend](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L398-L418) | Check filesystem free space, inode availability, quotas, storage errors, and whether WAL retention, temp files, or relation growth consumed the volume. PostgreSQL directly identifies free disk space as the first check. | The ERROR is built in; no optional logging GUC produces it. PostgreSQL does not expose general filesystem free-space health in the views used by this checklist, so add operating-system monitoring. |
| `out of shared memory` with `You might need to increase max_locks_per_transaction` | The lock request fails, so the current statement or transaction cannot continue normally. [lock.c#LockAcquireExtended](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L918-L930) | Check transactions touching very many relations/partitions, concurrent lock demand, and prepared transactions retaining locks. The PostgreSQL hint identifies lock-table sizing as the immediate mechanism. | `max_locks_per_transaction` is `PGC_POSTMASTER`: restart. The shared lock table is sized from it and `max_connections`. [guc.c#max_locks_per_transaction](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2460-L2472) |
| `page verification failed, calculated checksum ... but expected ...` or a nonzero checksum counter | PostgreSQL detected a page checksum mismatch, which is evidence of possible data-page corruption. [bufpage.c#PageIsVerified](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L155) | Investigate the storage and I/O path, preserve evidence, identify the relation/block from surrounding messages, and validate backups or replicas before repair. Checksums are designed to detect otherwise-silent I/O corruption. [initdb.sgml#data-checksums](../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L222) | Data checksums must be enabled. Confirm with the `data_checksums` row in the observability query; otherwise the database checksum counters are `NULL`. [config.sgml#data_checksums](../../../raw/postgres-12/doc/src/sgml/config.sgml#L9088-L9100) [monitoring.sgml#checksum_failures](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2599-L2611) |
| Connection/disconnection churn or unexpected replication commands | Short sessions can add connection overhead and make saturation harder to diagnose; unexpected replication commands are an audit signal that should be matched to the expected replication clients. `log_disconnections` includes session duration. [config.sgml#log_connections-disconnections](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6182-L6222) [config.sgml#log_replication_commands](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6539-L6551) | Check application pooling, authentication retries, client restart loops, and expected replication tooling. Duplicate connection messages can be normal because clients such as `psql` may connect twice during password handling. [config.sgml#log_connections](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6188-L6203) | Enable `log_connections` and `log_disconnections` for new sessions; enable `log_replication_commands` as a superuser session/transaction setting or cluster default. |

### Issue interpretation and response

The root-cause directions below are operational inferences from the mechanisms
and fields cited in each row.

| Priority | Finding | Repercussion | Root-cause direction |
|---|---|---|---|
| Immediate | Wraparound shutdown warning, checksum mismatch, disk-extension failure, or uncontrolled `pg_wal` growth | New commands or writes can fail; checksum findings indicate possible page corruption; ignored XID wraparound warnings progress to safety shutdown. [maintenance.sgml#wraparound-warning-shutdown](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L608-L635) [bufpage.c#PageIsVerified](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L145-L155) [md.c#mdextend](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L404-L418) | Stop nonessential load, preserve access for administration, identify old transactions/slots or failed archiving, and verify operating-system storage. WAL accumulates when archiving cannot keep up or a slot-backed standby is slow or failed. [vacuum.c#vacuum_set_xid_limits](../../../raw/postgres-12/src/backend/commands/vacuum.c#L942-L988) [wal.sgml#WAL-retention](../../../raw/postgres-12/doc/src/sgml/wal.sgml#L579-L588) |
| High | Connection exhaustion, sustained lock waits, deadlocks, timeout cancellations, or synchronous-replication delay | Clients are refused, statements wait or abort, and synchronous commits can retain locks longer. [proc.c#InitProcess](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L347-L363) [deadlock.c#DeadLockReport](../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L1139-L1146) [high-availability.sgml#synchronous-replication-impact](../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L1238-L1244) | Identify the specific sessions and lock holders, then inspect pool behavior, transaction duration, lock ordering, timeout policy, and replication stage lag. |
| Sustained degradation | Growing dead rows, stale analyze timestamps, repeated temp files, slow statements, frequent requested checkpoints, or backend writes/fsyncs | Table/index space and query cost can grow, plans can use stale statistics, temp and checkpoint I/O can compete with workload I/O, and latency can rise. [monitoring.sgml#pg_stat_all_tables](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2773-L2831) [monitoring.sgml#pg_stat_bgwriter](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2412-L2475) | Check autovacuum thresholds and blockers, the top `pg_stat_statements` entries, temp-file-producing sorts/hashes, and WAL/checkpoint sizing. [autovacuum.c#threshold-decision](../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060-L3080) [config.sgml#log_temp_files](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6562-L6573) [checkpointer.c#checkpoint-warning](../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462) |
| Observability gap | Empty query text, zero I/O times, absent slow/temp/lock/autovacuum logs, missing extension views, or `pending_restart = true` | The database may be unhealthy while the expected evidence is unavailable or incomplete. [config.sgml#track_activity_query_size](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6820-L6834) [config.sgml#track_io_timing](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6854-L6871) [catalogs.sgml#pg_settings_pending_restart](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10494-L10498) | Correct collection first: apply the required restart/reload/session setting, install and create the extension where needed, and begin a new measurement interval before drawing conclusions. |

## Context Reviewed

- PostgreSQL 12 logging destinations, collector, logging GUCs, statistics GUCs,
  autovacuum GUCs, WAL archiving settings, and their `GucContext` definitions.
- Built-in `pg_stat_activity`, `pg_locks`, `pg_stat_database`,
  `pg_stat_database_conflicts`, `pg_stat_user_tables`,
  `pg_stat_progress_vacuum`, `pg_stat_bgwriter`, `pg_stat_archiver`,
  `pg_stat_replication`, `pg_stat_wal_receiver`, `pg_stat_subscription`, and
  `pg_replication_slots` surfaces.
- Connection exhaustion, lock-wait, deadlock, timeout, temp-file, checkpoint,
  archiver, disk-extension, lock-table exhaustion, and page-checksum message
  paths.
- Autovacuum threshold decisions, XID and MultiXact anti-wraparound behavior,
  warning/shutdown paths, and old-horizon warning hints.
- PostgreSQL 12 `pg_stat_statements` and `pgstattuple` contrib documentation and
  SQL installation boundaries.

## Evidence Map

| Claim area | Primary evidence |
|---|---|
| Logging destinations and apply scopes | [config.sgml#log_destination](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5439-L5486), [config.sgml#logging_collector](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5518-L5534), [guc.c#GucContext_Names](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L594-L603) |
| Current and cumulative statistics surfaces | [system_views.sql#activity-through-database](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L887), [system_views.sql#archiver-bgwriter-vacuum](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L924-L965) |
| Connections, locks, deadlocks, and timeouts | [proc.c#InitProcess](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L330-L363), [proc.c#lock-wait-log](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1377-L1497), [deadlock.c#DeadLockReport](../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L1139-L1146), [postgres.c#ProcessInterrupts](../../../raw/postgres-12/src/backend/tcop/postgres.c#L3070-L3136) |
| Vacuum, analyze, and wraparound | [autovacuum.c#relation_needs_vacanalyze](../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2925-L3080), [vacuum.c#vacuum_set_xid_limits](../../../raw/postgres-12/src/backend/commands/vacuum.c#L896-L996), [maintenance.sgml#wraparound-warning-shutdown](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L557-L635) |
| Checkpoints, WAL archiving, replication, and slots | [checkpointer.c#checkpoint-warning](../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462), [pgarch.c#pgarch_ArchiverCopyLoop](../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L430-L539), [monitoring.sgml#pg_stat_replication](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1771-L2020), [catalogs.sgml#pg_replication_slots](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9835-L9968) |
| Query and bloat extensions | [pgstatstatements.sgml#loading](../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L10-L30), [pgstatstatements.sgml#settings](../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L399-L465), [pgstattuple.sgml#module-and-privileges](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L23), [pgstattuple.sgml#pgstatindex](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L186) |
| Checksums and storage errors | [bufpage.c#PageIsVerified](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L155), [initdb.sgml#data-checksums](../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L222), [md.c#mdextend](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L398-L418) |

## Open Questions

- Numeric alert thresholds for sessions, transaction age, dead rows, temp
  bytes, checkpoint rates, and replication lag depend on workload capacity and
  service objectives. The pinned source defines the counters and safety limits,
  but not a universal healthy production threshold.
- PostgreSQL's internal views do not replace host monitoring for free space,
  inodes, filesystem or device errors, CPU, memory pressure, and network health.
  The source explicitly sends relation-extension failures to the log with a
  free-space hint.
  [md.c#mdextend](../../../raw/postgres-12/src/backend/storage/smgr/md.c#L404-L418)
- `pg_stat_archiver` proves archive attempts and outcomes, not that a backup is
  restorable. Restore testing and backup-tool evidence must be added from the
  deployment's backup system; the built-in view contains archive success,
  failure, and reset fields only.
  [monitoring.sgml#pg_stat_archiver](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2342-L2397)

## Source References

- [guc.c#GucContext_Names](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L594-L603)
- [config.sgml#Logging](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5439-L5534)
- [config.sgml#WhatToLog](../../../raw/postgres-12/doc/src/sgml/config.sgml#L5931-L6574)
- [config.sgml#StatisticsAndAutovacuum](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6818-L7033)
- [monitoring.sgml#StatisticsViews](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L284-L425)
- [system_views.sql#StatisticsViews](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L887)
- [maintenance.sgml#TransactionIdWraparound](../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L391-L705)
- [autovacuum.c#relation_needs_vacanalyze](../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2925-L3080)
- [vacuum.c#vacuum_set_xid_limits](../../../raw/postgres-12/src/backend/commands/vacuum.c#L896-L996)
- [pgarch.c#pgarch_ArchiverCopyLoop](../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L430-L539)
- [pgstatstatements.sgml#pg_stat_statements](../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L10-L41)
- [pgstattuple.sgml#pgstattuple](../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L23)
- [bufpage.c#PageIsVerified](../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L155)

## Navigation

- [PostgreSQL 12 index](../index.md)
- [Wiki index](../../index.md)
- [Version manifest](../../versions.md)
