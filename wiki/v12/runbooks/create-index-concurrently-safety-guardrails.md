---
type: runbook
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Safe CREATE INDEX CONCURRENTLY Runbook for PostgreSQL 12 (unverified)

## Question

In PostgreSQL 12, in a database where `pg_dump` could be running, sessions may
have `statement_timeout` settings, and `VACUUM` could be running, what checks
and guardrails should be used to determine whether `CREATE INDEX CONCURRENTLY`
can run safely without being blocked or never finishing? Provide a step-by-step
list of all checks and safeguards with a comprehensive explanation.

## Answer

Do not treat `CREATE INDEX CONCURRENTLY` (CIC) as "safe" just because it lets
normal `INSERT`/`UPDATE`/`DELETE` continue. In PostgreSQL 12, CIC still takes a
`ShareUpdateExclusiveLock`, performs two table scans, and waits three times:
before the first scan, before validation, and before marking the index valid
([indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1307-L1472),
[ref/create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L572)).

The safe operating pattern is:

1. Prove the target is eligible and clean.
2. Check the four CIC blocking classes before starting.
3. Use a dedicated session with explicit timeout settings.
4. Monitor `pg_stat_progress_create_index`.
5. If the command is canceled or times out after it starts, check for and remove
   an invalid leftover index.

The SQL examples below assume the target table is `public.my_table` and the new
index will be `public.my_table_new_idx`. Replace those names before running.

### Step 0: Run the checks from a session that can see activity

Several checks need `pg_stat_activity.query`, `state`, and wait columns. In v12,
those details are visible to the same role, to roles the current user is a
member of, or to `pg_read_all_stats`
([pgstatfuncs.c#activity-visibility](../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L653-L687)).
The view exposes `application_name`, `xact_start`, `query_start`,
`wait_event_type`, `wait_event`, `state`, `backend_xid`, `backend_xmin`,
`query`, and `backend_type`
([system_views.sql#pg_stat_activity](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L756)).

```sql
SELECT /* wiki_cic_timeout_settings */
       name, setting, unit, context, source
FROM pg_settings
WHERE name IN (
  'application_name',
  'statement_timeout',
  'lock_timeout',
  'idle_in_transaction_session_timeout',
  'maintenance_work_mem',
  'max_parallel_maintenance_workers'
)
ORDER BY name;
```

`pg_settings` is backed by `pg_show_all_settings()`
([system_views.sql#pg_settings](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L512-L513)),
and its `context` text is produced from the v12 GUC context table
([guc.c#GucContext_Names](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L590-L603),
[guc.c#pg_settings_context](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L9026-L9032)).
For this runbook, `user` context means the setting can be applied at
session/transaction scope. The relevant v12 GUC definitions are `PGC_USERSET`
for `application_name`, `statement_timeout`, `lock_timeout`,
`idle_in_transaction_session_timeout`, `maintenance_work_mem`, and
`max_parallel_maintenance_workers`
([guc.c#application_name](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4159-L4167),
[guc.c#timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2408),
[guc.c#maintenance_work_mem](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2243-L2252),
[guc.c#max_parallel_maintenance_workers](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2998-L3005)).

### Step 1: Confirm the target is eligible

CIC is rejected inside an explicit transaction block, and PostgreSQL 12 rejects
concurrent index creation on partitioned tables
([utility.c#CIC-transaction-block](../../../raw/postgres-12/src/backend/tcop/utility.c#L1307-L1326),
[indexcmds.c#partitioned-error](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L603-L616)).
Temporary tables silently use a non-concurrent build because other sessions
cannot access them
([indexcmds.c#temp-fallback](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L489-L499)).

```sql
SELECT /* wiki_cic_target_shape */
       c.oid::regclass AS target_table,
       c.relkind,
       CASE c.relkind
         WHEN 'r' THEN 'ordinary table'
         WHEN 'm' THEN 'materialized view'
         WHEN 'p' THEN 'partitioned table - CIC is rejected in v12'
         ELSE 'not a CIC target for this runbook'
       END AS relkind_meaning,
       c.relpersistence,
       CASE c.relpersistence
         WHEN 'p' THEN 'permanent'
         WHEN 'u' THEN 'unlogged'
         WHEN 't' THEN 'temporary - concurrent build falls back'
       END AS persistence_meaning,
       pg_size_pretty(pg_table_size(c.oid)) AS table_size,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class AS c
WHERE c.oid = 'public.my_table'::regclass;
```

The `relkind` and `relpersistence` codes come from `pg_class`
([pg_class.h#relkind-relpersistence](../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167)).
The size functions are present in v12 as `pg_table_size`,
`pg_total_relation_size`, and `pg_size_pretty`
([pg_proc.dat#size-functions](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6915)).

### Step 2: Check for invalid leftover indexes first

If CIC fails after it has created the catalog entry, PostgreSQL can leave an
invalid index behind. The docs say such an index is ignored for queries, still
consumes update overhead, and for a failed unique build can continue enforcing
uniqueness
([ref/create_index.sgml#invalid-index](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L606)).
The relevant catalog flags are `indisvalid`, `indisready`, and `indislive`
([pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L30-L44),
[catalogs.sgml#pg_index-flags](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3822-L3866)).

```sql
SELECT /* wiki_cic_existing_invalid_indexes */
       i.indexrelid::regclass AS index_name,
       i.indisvalid,
       i.indisready,
       i.indislive
FROM pg_index AS i
WHERE i.indrelid = 'public.my_table'::regclass
  AND (NOT i.indisvalid OR NOT i.indisready OR NOT i.indislive)
ORDER BY i.indexrelid::regclass::text;
```

Do not start another attempt until the owner decides whether to drop or rebuild
those leftovers. If the leftover is not constraint-backed, the low-blocking
cleanup shape is `DROP INDEX CONCURRENTLY`, which v12 supports with one index
name and without `CASCADE`; it cannot run inside a transaction block
([ref/drop_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/drop_index.sgml#L20-L65),
[utility.c#drop-index-concurrently](../../../raw/postgres-12/src/backend/tcop/utility.c#L1733-L1739)).

```sql
DROP /* wiki_cic_drop_invalid_leftover */
INDEX CONCURRENTLY IF EXISTS public.my_table_new_idx;
```

### Step 3: Check target-table locks that block the initial lock

At command start, CIC locks the table in `ShareUpdateExclusiveLock`
([utility.c#CIC-lockmode](../../../raw/postgres-12/src/backend/tcop/utility.c#L1311-L1326),
[indexcmds.c#CIC-lockmode](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L548-L564)).
That lock conflicts with another `ShareUpdateExclusiveLock`, `ShareLock`,
`ShareRowExclusiveLock`, `ExclusiveLock`, and `AccessExclusiveLock`
([lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L105)).
In v12 command terms, that includes target-table `VACUUM`, autovacuum,
`ANALYZE`, another CIC, `REINDEX CONCURRENTLY`, plain `CREATE INDEX`, many DDL
forms, `VACUUM FULL`, `CLUSTER`, `TRUNCATE`, and `DROP TABLE`
([mvcc.sgml#table-lock-modes](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L912-L1030)).

```sql
SELECT /* wiki_cic_target_conflicting_locks */
       l.pid,
       a.usename,
       a.application_name,
       a.backend_type,
       a.state,
       a.xact_start,
       now() - a.xact_start AS xact_age,
       l.mode,
       a.wait_event_type,
       a.wait_event,
       left(a.query, 300) AS query
FROM pg_locks AS l
JOIN pg_stat_activity AS a ON a.pid = l.pid
WHERE l.locktype = 'relation'
  AND l.relation = 'public.my_table'::regclass
  AND l.granted
  AND l.mode IN (
    'ShareUpdateExclusiveLock',
    'ShareLock',
    'ShareRowExclusiveLock',
    'ExclusiveLock',
    'AccessExclusiveLock'
  )
ORDER BY a.xact_start NULLS LAST, l.mode;
```

If this returns rows, the CIC command would wait before it can even create its
catalog entry. A target-table autovacuum normally has low priority: after the
waiter reaches the deadlock-check path, the lock manager can send the worker
`SIGINT`; anti-wraparound autovacuum is not canceled
([proc.c#autovacuum-cancel](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375)).
That makes autovacuum less dangerous than a manual `VACUUM`, but it is still a
start-time blocker.

Use the VACUUM progress view to see whether the target table is being vacuumed
and where it is in the scan. The view exists for both manual `VACUUM` and
autovacuum workers
([monitoring.sgml#vacuum-progress](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3738-L3752),
[system_views.sql#pg_stat_progress_vacuum](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L949-L965)).

```sql
SELECT /* wiki_cic_target_vacuum_progress */
       v.pid,
       a.backend_type,
       a.application_name,
       v.relid::regclass AS table_name,
       v.phase,
       v.heap_blks_scanned,
       v.heap_blks_total,
       round(100.0 * v.heap_blks_scanned / NULLIF(v.heap_blks_total, 0), 1)
         AS heap_scan_pct,
       v.index_vacuum_count,
       v.num_dead_tuples,
       left(a.query, 300) AS query
FROM pg_stat_progress_vacuum AS v
JOIN pg_stat_activity AS a ON a.pid = v.pid
WHERE v.relid = 'public.my_table'::regclass;
```

A `VACUUM` on another table does not hold the target table's
`ShareUpdateExclusiveLock`, so it is not an initial lock blocker. It can still
compete for I/O and CPU. Lazy `VACUUM` and autovacuum are also excluded from
CIC's old-snapshot wait by `WaitForOlderSnapshots`
([indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402),
[proc.h#vacuumFlags](../../../raw/postgres-12/src/include/storage/proc.h#L53-L56)).

### Step 4: Check open writers on the target table

The first and second CIC waits call `WaitForLockers(..., ShareLock, true)`.
That routine snapshots the current lockers, waits on their virtual transaction
IDs, and does not wait for new lockers that arrive later
([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)).
Because `ShareLock` conflicts with `RowExclusiveLock`, these waits catch
transactions that have modified the target table and are still open
([lock.c#ShareLock-conflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86),
[mvcc.sgml#RowExclusiveLock](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L910)).

```sql
SELECT /* wiki_cic_target_open_writers */
       l.pid,
       a.usename,
       a.application_name,
       a.backend_type,
       a.state,
       a.xact_start,
       now() - a.xact_start AS xact_age,
       a.backend_xid,
       a.wait_event_type,
       a.wait_event,
       left(a.query, 300) AS query
FROM pg_locks AS l
JOIN pg_stat_activity AS a ON a.pid = l.pid
WHERE l.locktype = 'relation'
  AND l.relation = 'public.my_table'::regclass
  AND l.granted
  AND l.mode = 'RowExclusiveLock'
ORDER BY a.xact_start NULLS LAST;
```

Rows with `state = 'idle in transaction'` are the high-risk case. PostgreSQL
documents that this state means the backend is in a transaction but not
executing a query
([monitoring.sgml#activity-state](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L774-L814)).
Canceling the current query may not end an idle transaction; `pg_cancel_backend`
signals the current query, while `pg_terminate_backend` terminates the backend
process, subject to role/superuser checks
([signalfuncs.c#backend-signals](../../../raw/postgres-12/src/backend/storage/ipc/signalfuncs.c#L106-L150),
[pg_proc.dat#backend-signals](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5926-L5931)).

### Step 5: Check old snapshots, including pg_dump

The final CIC wait calls `WaitForOlderSnapshots(limitXmin, true)`. In v12 this
collects same-database backends whose advertised `xmin` is not newer than the
reference snapshot, skips backends with `xmin = 0`, and skips autovacuum/lazy
VACUUM flags
([indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402),
[procarray.c#GetCurrentVirtualXIDs](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2467-L2548)).
Before the run starts, any same-database backend with `backend_xmin` is a risk
for this final wait. It may not actually be older than the future reference
snapshot, but old long-running entries are the ones to clear before starting.

```sql
SELECT /* wiki_cic_same_database_old_snapshot_risk */
       pid,
       usename,
       application_name,
       backend_type,
       state,
       xact_start,
       now() - xact_start AS xact_age,
       query_start,
       now() - query_start AS query_age,
       backend_xmin,
       wait_event_type,
       wait_event,
       left(query, 300) AS query
FROM pg_stat_activity
WHERE datname = current_database()
  AND pid <> pg_backend_pid()
  AND backend_xmin IS NOT NULL
ORDER BY xact_start NULLS LAST, query_start NULLS LAST;
```

`backend_xmin` is the current backend's xmin horizon in `pg_stat_activity`
([monitoring.sgml#backend_xmin](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L817-L836)).
`current_database()` and `pg_backend_pid()` are present in v12
([pg_proc.dat#current_database](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L1681-L1682),
[pg_proc.dat#pg_backend_pid](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5154-L5155)).

`pg_dump` is important because it intentionally pins a snapshot. The v12 client
sets `fallback_application_name` to the program name, unless the caller provides
`application_name` or `PGAPPNAME`
([pg_backup_db.c#fallback-application-name](../../../raw/postgres-12/src/bin/pg_dump/pg_backup_db.c#L268-L288),
[libpq.sgml#fallback_application_name](../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L1179-L1189)).
It disables `statement_timeout`, `lock_timeout`, and
`idle_in_transaction_session_timeout`; then it starts one `REPEATABLE READ,
READ ONLY` transaction, or `SERIALIZABLE, READ ONLY, DEFERRABLE` when requested
([pg_dump.c#disable-timeouts](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1147),
[pg_dump.c#dump-transaction](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1166-L1194)).
`REPEATABLE READ` and `SERIALIZABLE` keep the first transaction snapshot until
transaction end
([snapmgr.c#GetTransactionSnapshot](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L305-L373)).

```sql
SELECT /* wiki_cic_possible_pg_dump_snapshot */
       pid,
       usename,
       application_name,
       state,
       xact_start,
       now() - xact_start AS xact_age,
       backend_xmin,
       left(query, 300) AS query
FROM pg_stat_activity
WHERE datname = current_database()
  AND backend_xmin IS NOT NULL
  AND (
    application_name = 'pg_dump'
    OR query LIKE '%_pg_dump_cursor%'
    OR query LIKE 'LOCK TABLE % IN ACCESS SHARE MODE%'
  )
ORDER BY xact_start NULLS LAST;
```

A same-database `pg_dump` that started before CIC's reference snapshot can block
the final old-snapshot wait until the dump finishes. Its table locks are
`ACCESS SHARE`, which do not conflict with CIC's `ShareUpdateExclusiveLock` or
with the `ShareLock` used by the two writer waits
([pg_dump.c#lock-table](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6646-L6671),
[lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L105)).

### Step 6: Check prepared transactions

Prepared transactions can hold locks after `PREPARE TRANSACTION`, and those
locks remain until `COMMIT PREPARED` or `ROLLBACK PREPARED`
([lock.c#prepared-locks](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2873-L2876)).
The v12 `pg_prepared_xacts` view lists prepared transactions with transaction
ID, GID, prepare time, owner, and database
([system_views.sql#pg_prepared_xacts](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L325-L330)).

```sql
SELECT /* wiki_cic_prepared_xacts */
       transaction,
       gid,
       prepared,
       owner,
       database
FROM pg_prepared_xacts
WHERE database = current_database()
ORDER BY prepared;
```

For an operational runbook, treat any same-database prepared transaction as a
stop-and-investigate condition. A prepared transaction holding one of the
initial conflicting table locks can block the start indefinitely, and the
writer waits intentionally do not report prepared transactions
([lmgr.c#prepared-xacts-skipped](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L890-L894)).

### Step 7: Do a short NOWAIT lock probe

This probe is not a guarantee. It only checks whether the target table's
`ShareUpdateExclusiveLock` is immediately available now. It is useful because a
failure here happens before CIC creates an invalid index. `LOCK TABLE ...
NOWAIT` errors instead of waiting, and table locks are released at transaction
end
([ref/lock.sgml#LOCK-synopsis](../../../raw/postgres-12/doc/src/sgml/ref/lock.sgml#L21-L45),
[ref/lock.sgml#NOWAIT](../../../raw/postgres-12/doc/src/sgml/ref/lock.sgml#L151-L158)).

```sql
BEGIN /* wiki_cic_probe_begin */;
SET /* wiki_cic_probe_lock_timeout */ LOCAL lock_timeout = '2s';
SET /* wiki_cic_probe_statement_timeout */ LOCAL statement_timeout = '30s';
LOCK /* wiki_cic_probe_sue_lock */
TABLE public.my_table IN SHARE UPDATE EXCLUSIVE MODE NOWAIT;
ROLLBACK /* wiki_cic_probe_rollback */;
```

If this probe fails, do not start CIC. Return to the lock and VACUUM checks.

### Step 8: Run CIC from a dedicated session with explicit guardrails

CIC cannot run inside `BEGIN`/`COMMIT`, because `PreventInTransactionBlock`
rejects it and the implementation commits internally
([utility.c#CIC-transaction-block](../../../raw/postgres-12/src/backend/tcop/utility.c#L1307-L1309),
[ref/create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L615-L621)).
Use a dedicated autocommit session and set the guardrails before the command.

Recommended starting values:

| Setting | Suggested starting value | Why |
|---|---:|---|
| `application_name` | `cic public.my_table my_table_new_idx` | Makes the session easy to find in `pg_stat_activity`; the setting is reported in stats/logs ([config.sgml#application_name](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6095-L6112)). |
| `lock_timeout` | `1min` | Bounds heavyweight and virtual-xid lock waits. If it fires after CIC has created the index, expect cleanup. It is `PGC_USERSET` and a value of `0` disables it ([guc.c#timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2408), [config.sgml#lock_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7701-L7735)). |
| `statement_timeout` | A real maintenance-window cap, for example `6h` | Avoid inheriting an application default such as `30s`. A value of `0` disables it; use `0` only if an external watchdog will cancel the session ([config.sgml#statement_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7669-L7698)). |
| `idle_in_transaction_session_timeout` | `1min` | Defensive only, because CIC itself is not run inside a user transaction block. It is `PGC_USERSET`; `0` disables it ([guc.c#timeouts](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2408)). |
| `maintenance_work_mem` | A value sized for the host, such as `1GB` on a host that can spare it | Index build speed depends on it, but oversizing can cause swapping ([config.sgml#maintenance_work_mem](../../../raw/postgres-12/doc/src/sgml/config.sgml#L1686-L1715), [ref/create_index.sgml#maintenance_work_mem](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L707-L713)). |
| `max_parallel_maintenance_workers` | `2`, or lower if the host is busy | v12 defaults to 2. Only B-tree `CREATE INDEX` uses parallel utility workers, and CIC parallelizes only the first scan ([guc.c#max_parallel_maintenance_workers](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2998-L3005), [config.sgml#parallel-maintenance](../../../raw/postgres-12/doc/src/sgml/config.sgml#L2283-L2317), [ref/create_index.sgml#parallel-cic](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L715-L771)). |

```sql
SET /* wiki_cic_run_application_name */
    application_name = 'cic public.my_table my_table_new_idx';
SET /* wiki_cic_run_lock_timeout */ lock_timeout = '1min';
SET /* wiki_cic_run_statement_timeout */ statement_timeout = '6h';
SET /* wiki_cic_run_idle_timeout */ idle_in_transaction_session_timeout = '1min';
SET /* wiki_cic_run_maintenance_work_mem */ maintenance_work_mem = '1GB';
SET /* wiki_cic_run_parallel_workers */ max_parallel_maintenance_workers = 2;

CREATE /* wiki_cic_run_create_index */
INDEX CONCURRENTLY my_table_new_idx
ON public.my_table (some_column);
```

The low `lock_timeout` is deliberate: it prevents "wait forever" behavior, but
it can also abort a CIC after work has begun. That is why Step 10 is mandatory.

### Step 9: Monitor progress while it runs

`pg_stat_progress_create_index` reports one row for each backend creating an
index, including command, phase, locker counts, current locker PID, block
counts, and tuple counts
([monitoring.sgml#create-index-progress-view](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3493-L3620),
[system_views.sql#pg_stat_progress_create_index](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L994-L1027)).
The CIC wait phases are `waiting for writers before build`, `waiting for
writers before validation`, and `waiting for old snapshots`
([monitoring.sgml#create-index-phases](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3639-L3708)).

```sql
SELECT /* wiki_cic_progress_watch */
       p.pid,
       p.datname,
       p.relid::regclass AS table_name,
       p.index_relid::regclass AS index_name,
       p.command,
       p.phase,
       p.lockers_done,
       p.lockers_total,
       p.current_locker_pid,
       locker.application_name AS current_locker_application,
       locker.state AS current_locker_state,
       locker.xact_start AS current_locker_xact_start,
       now() - locker.xact_start AS current_locker_xact_age,
       left(locker.query, 300) AS current_locker_query,
       p.blocks_done,
       p.blocks_total,
       CASE
         WHEN p.blocks_total > 0
         THEN round(100.0 * p.blocks_done / p.blocks_total, 1)
       END AS blocks_pct,
       p.tuples_done,
       p.tuples_total
FROM pg_stat_progress_create_index AS p
LEFT JOIN pg_stat_activity AS locker
  ON locker.pid = p.current_locker_pid
WHERE p.relid = 'public.my_table'::regclass;
```

If the CIC session is blocked before `DefineIndex()` starts reporting progress,
look for the session in `pg_stat_activity` and use `pg_blocking_pids()`.
`pg_blocking_pids()` reports PIDs that either hold a conflicting lock or are
ahead in the lock wait queue
([lockfuncs.c#pg_blocking_pids](../../../raw/postgres-12/src/backend/utils/adt/lockfuncs.c#L399-L515),
[pg_proc.dat#pg_blocking_pids](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5797-L5805)).

```sql
SELECT /* wiki_cic_backend_blockers */
       a.pid,
       a.application_name,
       a.state,
       a.wait_event_type,
       a.wait_event,
       pg_blocking_pids(a.pid) AS blocking_pids,
       left(a.query, 300) AS query
FROM pg_stat_activity AS a
WHERE a.application_name = 'cic public.my_table my_table_new_idx'
   OR a.query LIKE 'CREATE%INDEX%CONCURRENTLY%my_table_new_idx%';
```

Phase interpretation:

| Phase | What to do |
|---|---|
| `waiting for writers before build` | Inspect `current_locker_pid`; it is a transaction that held a conflicting writer lock when the wait list was collected ([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)). |
| `waiting for writers before validation` | Same response as the first writer wait; an idle writer on the target table can hold the wait until transaction end ([mvcc.sgml#RowExclusiveLock](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L910)). |
| `waiting for old snapshots` | Inspect `current_locker_pid` and the old-snapshot query. A same-database `pg_dump`, old `REPEATABLE READ`/`SERIALIZABLE` transaction, or cursor-holding session can be the blocker ([indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402), [pg_dump.c#dump-transaction](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1166-L1194)). |
| `building index` / validation scan phases | Usually not lock-blocked; watch `blocks_done`, `blocks_total`, `tuples_done`, and `tuples_total` because the docs report those columns for build and validation phases ([monitoring.sgml#create-index-phases](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3651-L3697)). |

### Step 10: After success, failure, cancel, or timeout, verify the result

On success, the new index should have `indisvalid = true`, `indisready = true`,
and `indislive = true`. On failure, timeout, or manual cancel, verify the flags
before retrying
([pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L30-L44),
[index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)).

```sql
SELECT /* wiki_cic_post_run_index_state */
       i.indexrelid::regclass AS index_name,
       i.indisvalid,
       i.indisready,
       i.indislive
FROM pg_index AS i
WHERE i.indrelid = 'public.my_table'::regclass
  AND i.indexrelid = 'public.my_table_new_idx'::regclass;
```

If the index exists but is invalid or not live, decide between a retry and
cleanup. For a plain non-constraint index, cleanup is:

```sql
DROP /* wiki_cic_post_run_drop_invalid */
INDEX CONCURRENTLY IF EXISTS public.my_table_new_idx;
```

Do not run `DROP INDEX CONCURRENTLY` inside `BEGIN`/`COMMIT`, and do not use it
for a constraint-backed index that requires `CASCADE`; v12 documents both
caveats
([ref/drop_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/drop_index.sgml#L43-L65)).

### Specific risks from the question

| Risk | Practical answer |
|---|---|
| Running `pg_dump` | Do not start if the dump is in the same database and already has `backend_xmin`. It can block CIC's final old-snapshot wait until it finishes; it disables its own timeouts, so a site-wide timeout policy will not release it ([pg_dump.c#disable-timeouts](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1147), [pg_dump.c#dump-transaction](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1166-L1194)). |
| Session `statement_timeout` defaults | Do not inherit application defaults. Set an explicit CIC-session `statement_timeout`; a timeout aborts the statement, and CIC failure can leave an invalid index ([config.sgml#statement_timeout](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7669-L7698), [ref/create_index.sgml#invalid-index](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L606)). |
| Target-table `VACUUM` or autovacuum | Treat it as a start-time blocker because `VACUUM` and CIC both use `ShareUpdateExclusiveLock`. Autovacuum may be canceled unless it is anti-wraparound, but manual `VACUUM` will not be canceled by that autovacuum-specific path ([mvcc.sgml#ShareUpdateExclusiveLock](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L912-L936), [proc.c#autovacuum-cancel](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375)). |
| Idle writer transaction | Do not start while one exists on the target table. The writer waits sleep until the whole transaction ends, not until the last statement ends ([lmgr.c#WaitForLockersMultiple](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)). |
| Long read-only `READ COMMITTED` transaction | It is a risk only while it advertises `backend_xmin`. A backend with no active or registered snapshots can clear its advertised xmin through `SnapshotResetXmin` ([snapmgr.c#SnapshotResetXmin](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L989-L1028)). |
| Long `REPEATABLE READ` / `SERIALIZABLE` transaction | Treat it as an old-snapshot risk because the first snapshot is kept until transaction end ([snapmgr.c#GetTransactionSnapshot](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L336-L356)). |

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v12/index.md`, and the last ~20
  `wiki/log.md` entries (navigation only).
- Pinned checkout `raw/postgres-12/` at commit
  `45b88269a353ad93744772791feb6d01bc7e1e42`.
- CIC orchestration in `src/backend/commands/indexcmds.c`, including
  `DefineIndex` and `WaitForOlderSnapshots`.
- Lock conflict behavior in `src/backend/storage/lmgr/lock.c`,
  `src/backend/storage/lmgr/lmgr.c`, `src/backend/storage/lmgr/proc.c`, and
  `doc/src/sgml/mvcc.sgml`.
- Activity, lock, progress, prepared-transaction, and settings view definitions
  in `src/backend/catalog/system_views.sql`, `pgstatfuncs.c`, `lockfuncs.c`,
  `guc.c`, and `pg_proc.dat`.
- `pg_dump` connection setup, timeout disabling, transaction setup, and table
  locks in `src/bin/pg_dump/pg_backup_db.c` and `src/bin/pg_dump/pg_dump.c`.
- User-facing docs for `CREATE INDEX`, `DROP INDEX`, `LOCK`, monitoring views,
  libpq application names, and relevant GUCs.

## Evidence Map

| Claim | Source |
|---|---|
| CIC uses multiple transactions, two scans, and three waits | [indexcmds.c:1307-1472](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1307-L1472), [ref/create_index.sgml:545-572](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L572) |
| CIC cannot run inside a user transaction block | [utility.c:1307-1309](../../../raw/postgres-12/src/backend/tcop/utility.c#L1307-L1309), [ref/create_index.sgml:615-621](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L615-L621) |
| Partitioned-table CIC is rejected; temp-table CIC falls back | [indexcmds.c:603-616](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L603-L616), [indexcmds.c:489-499](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L489-L499) |
| Failed CIC can leave an invalid index with write overhead | [ref/create_index.sgml:574-606](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L574-L606), [catalogs.sgml:3822-3866](../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3822-L3866) |
| Start-time blockers are the modes conflicting with `ShareUpdateExclusiveLock` | [utility.c:1311-1326](../../../raw/postgres-12/src/backend/tcop/utility.c#L1311-L1326), [lock.c:65-105](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L105), [mvcc.sgml:912-1030](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L912-L1030) |
| Writer waits use `WaitForLockers(..., ShareLock)` and wait on current VXID holders | [indexcmds.c:1328-1389](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1328-L1389), [lmgr.c:850-949](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949), [lock.c:83-86](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L86) |
| Old-snapshot wait filters same-database backends by xmin and excludes vacuum flags | [indexcmds.c:339-402](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402), [procarray.c:2467-2548](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2467-L2548), [proc.h:53-56](../../../raw/postgres-12/src/include/storage/proc.h#L53-L56) |
| `pg_dump` pins a transaction snapshot and disables timeouts | [pg_dump.c:1140-1194](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1194), [snapmgr.c:305-373](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L305-L373) |
| Monitoring views and lock helper functions used by the SQL snippets exist in v12 | [system_views.sql:306-330](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L306-L330), [system_views.sql:732-756](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L756), [system_views.sql:949-1027](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L949-L1027), [lockfuncs.c:399-515](../../../raw/postgres-12/src/backend/utils/adt/lockfuncs.c#L399-L515) |
| Timeout, memory, parallel-worker, and application-name GUCs are session-settable in this runbook | [guc.c:2377-2408](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2408), [guc.c:2243-2252](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2243-L2252), [guc.c:2998-3005](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2998-L3005), [guc.c:4159-4167](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4159-L4167) |

## Open Questions

- The source proves the blocking classes and the monitoring fields, but it
  cannot choose a universal `statement_timeout`, `lock_timeout`, or
  `maintenance_work_mem` for a specific production table. The values above are
  starting guardrails; table size, I/O budget, backup windows, and incident
  policy decide the final numbers.

## Source References

- [indexcmds.c#DefineIndex](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L429-L1472)
- [indexcmds.c#WaitForOlderSnapshots](../../../raw/postgres-12/src/backend/commands/indexcmds.c#L339-L402)
- [index.c#index_set_state_flags](../../../raw/postgres-12/src/backend/catalog/index.c#L3331-L3403)
- [utility.c#CIC-dispatch](../../../raw/postgres-12/src/backend/tcop/utility.c#L1305-L1326)
- [utility.c#DROP-INDEX-CONCURRENTLY](../../../raw/postgres-12/src/backend/tcop/utility.c#L1728-L1742)
- [lmgr.c#WaitForLockers](../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L850-L949)
- [lock.c#LockConflicts](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L105)
- [lock.c#prepared-locks](../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L2873-L2876)
- [proc.c#ProcSleep-autovacuum-cancel](../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375)
- [procarray.c#GetCurrentVirtualXIDs](../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L2467-L2548)
- [proc.h#vacuumFlags](../../../raw/postgres-12/src/include/storage/proc.h#L53-L56)
- [snapmgr.c#GetTransactionSnapshot](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L305-L373)
- [snapmgr.c#SnapshotResetXmin](../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L989-L1028)
- [signalfuncs.c#backend-signals](../../../raw/postgres-12/src/backend/storage/ipc/signalfuncs.c#L106-L150)
- [lockfuncs.c#pg_blocking_pids](../../../raw/postgres-12/src/backend/utils/adt/lockfuncs.c#L399-L515)
- [pgstatfuncs.c#pg_stat_get_activity](../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L547-L754)
- [guc.c#GUC-definitions](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2231-L2408)
- [guc.c#parallel-application-settings](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2998-L3005)
- [guc.c#application_name](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4159-L4167)
- [guc.c#pg_settings-output](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L590-L603)
- [system_views.sql#monitoring-views](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L306-L330)
- [system_views.sql#pg_stat_activity](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L756)
- [system_views.sql#progress-views](../../../raw/postgres-12/src/backend/catalog/system_views.sql#L949-L1027)
- [pg_proc.dat#monitoring-functions](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5758-L5805)
- [pg_proc.dat#backend-signals](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L5926-L5931)
- [pg_proc.dat#size-functions](../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6884-L6915)
- [pg_class.h#relkind-relpersistence](../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167)
- [pg_index.h#index-state-flags](../../../raw/postgres-12/src/include/catalog/pg_index.h#L30-L44)
- [pg_backup_db.c#fallback-application-name](../../../raw/postgres-12/src/bin/pg_dump/pg_backup_db.c#L268-L288)
- [pg_dump.c#dump-transaction](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1194)
- [pg_dump.c#lock-table](../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6646-L6671)
- [ref/create_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L545-L631)
- [ref/create_index.sgml#resource-notes](../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L705-L771)
- [ref/drop_index.sgml#CONCURRENTLY](../../../raw/postgres-12/doc/src/sgml/ref/drop_index.sgml#L20-L65)
- [ref/lock.sgml#LOCK-NOWAIT](../../../raw/postgres-12/doc/src/sgml/ref/lock.sgml#L21-L45)
- [ref/lock.sgml#NOWAIT](../../../raw/postgres-12/doc/src/sgml/ref/lock.sgml#L151-L158)
- [mvcc.sgml#table-locks](../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L890-L1030)
- [monitoring.sgml#pg_stat_activity](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L589-L872)
- [monitoring.sgml#create-index-progress](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3493-L3708)
- [monitoring.sgml#vacuum-progress](../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3738-L3925)
- [config.sgml#memory-parallel-timeouts-appname](../../../raw/postgres-12/doc/src/sgml/config.sgml#L1686-L1715)
- [config.sgml#parallel-maintenance](../../../raw/postgres-12/doc/src/sgml/config.sgml#L2283-L2317)
- [config.sgml#application_name](../../../raw/postgres-12/doc/src/sgml/config.sgml#L6095-L6112)
- [config.sgml#timeouts](../../../raw/postgres-12/doc/src/sgml/config.sgml#L7669-L7748)
- [libpq.sgml#fallback_application_name](../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L1179-L1189)

## Navigation

- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](../questions/create-index-concurrently.md)
- [v12 index](../index.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
