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
  - [What each source of evidence represents](#what-each-source-of-evidence-represents)
  - [Who can see what](#who-can-see-what)
  - [When counters reset](#when-counters-reset)
  - [How to use this checklist](#how-to-use-this-checklist)
  - [Production query guardrails](#production-query-guardrails)
  - [Observability prerequisites](#observability-prerequisites)
  - [Defaults that decide what you see](#defaults-that-decide-what-you-see)
  - [SQL health checklist](#sql-health-checklist)
  - [Database log checklist](#database-log-checklist)
  - [Issue interpretation and response](#issue-interpretation-and-response)
  - [Final causal summary](#final-causal-summary)
- [Measurement Script](#measurement-script)
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

Run the health check in layers, and read every result in light of how
PostgreSQL 12 produced it. First confirm that the server collects the data and
that your role can see it. Then read live state (sessions, locks, replication),
cumulative counters, and catalog freeze horizons. Finally, read the server log
for the same interval.

The order matters because an empty view, a zero counter, or a missing log line
has several possible meanings. It can mean "healthy", "never collected", "hidden
from this role", or "reset by a restart". The first three sections below tell
you which. PostgreSQL 12 provides dynamic views for current activity and
collected views for cumulative counters, and the two behave differently.
[monitoring.sgml#DynamicStatisticsViews](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L284-L360)
[monitoring.sgml#CollectedStatisticsViews](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L375-L425)

Treat cumulative values as rates over a known interval, not as isolated
totals. In `pg_stat_database`, only `numbackends` is current state; the other
columns accumulate until `stats_reset`.
[monitoring.sgml#pg_stat_database](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2510-L2528)
[monitoring.sgml#pg_stat_database_stats_reset](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2625-L2638)
In PostgreSQL 12 a separate statistics collector process holds these counters
(see [Cumulative statistics](../../../glossary.md#cumulative-statistics)).
Backends send it counts just before going idle, and it publishes a new report
at most every 500 ms. A session then keeps the report it fetched until its
transaction ends. Collect comparison samples in separate autocommit
transactions, or call `pg_stat_clear_snapshot()` before a repeat read in the
same transaction.
[monitoring.sgml#statistics-lag-and-snapshots](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L260)

### What each source of evidence represents

Each source below answers a different question, so read it with its own
freshness rule. The "Live or stored" column says where the value comes from.
The "Freshness and lifetime" column says when a reading can be stale or lost.

| Evidence | What it represents | Live or stored | Freshness and lifetime | Used in |
|---|---|---|---|---|
| [`pg_stat_activity`](../../../glossary.md#pg_stat_activity), `pg_locks`, `pg_blocking_pids()`, `pg_stat_progress_vacuum`, `pg_prepared_xacts` | What server processes, locks, and [prepared transactions](../../../glossary.md#two-phase-commit) are doing now | Live shared memory | The activity data is captured on first use in a transaction and reused until the transaction ends. [monitoring.sgml#activity-snapshot](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L242-L251) | Sections 1, 2, 4 |
| `pg_stat_replication`, `pg_stat_wal_receiver`, `pg_stat_subscription`, `pg_replication_slots` | Current replication processes and slot horizons | Live | Lag columns show recent measurements and revert to NULL after an idle catch-up. [monitoring.sgml#replication-lag](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1985-L2003) | Section 7 |
| `pg_stat_database`, `pg_stat_*_tables`, `pg_stat_bgwriter`, `pg_stat_archiver`, `pg_stat_database_conflicts` | Cumulative counters | Stored by the statistics collector | Lag behind activity, are snapshotted per transaction, and are reset by the events in [When counters reset](#when-counters-reset). [monitoring.sgml#statistics-lag-and-snapshots](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L260) | Sections 3, 4, 6, 7 |
| [`pg_stat_statements`](../../../glossary.md#pg_stat_statements) | Cumulative totals per user, database, and query identifier | Extension shared memory | Read entry by entry, each under its own spinlock, so one read is not an atomic snapshot. [pg_stat_statements.c#view-iteration](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1500-L1615) | Section 8 |
| `pg_class.relfrozenxid`, `relminmxid`, `pg_database.datfrozenxid`, `datminmxid` | [Freeze](../../../glossary.md#freezing) horizons | Stored catalog state | Move only when VACUUM advances them. [maintenance.sgml#xid-age-queries](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L557-L585) | Section 5 |
| `pg_settings` | Effective settings in the current session, plus `pending_restart` | Live | A row can be hidden from the reader; see [Who can see what](#who-can-see-what). [guc.c#pg_settings-access](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L8998-L9008) | [Observability prerequisites](#observability-prerequisites) |
| Server log | Events that happened and passed the severity and threshold filters | Stored log stream | Contains only what the logging settings allowed while the event happened. [config.sgml#server-log-levels](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5877-L5927) | [Database log checklist](#database-log-checklist) |

### Who can see what

The monitoring role decides which rows and fields you get, and PostgreSQL 12
fails in three different ways: it hides a row, nulls a field, or raises an
error. A health query run as the wrong role can therefore look healthy. The
[default roles](../../../glossary.md#default-roles) `pg_read_all_settings`,
`pg_read_all_stats`, and `pg_stat_scan_tables` are all granted to `pg_monitor`.
[user-manag.sgml#default-roles](../../../../raw/postgres-12/doc/src/sgml/user-manag.sgml#L519-L538)
[system_views.sql#pg_monitor-role-grants](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1349-L1351)

| Surface | Superuser | `pg_monitor` member | Ordinary role | Source |
|---|---|---|---|---|
| `state`, `query`, waits, timestamps, and `backend_type` of a session whose role the reader has no privileges of, in `pg_stat_activity` | Shown | Shown, through `pg_read_all_stats` | NULL, with query text `<insufficient privilege>` | [pgstatfuncs.c#activity-access](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L653-L656) [pgstatfuncs.c#activity-redaction](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L885-L911) |
| Any session's `pid`, `datid`, `usesysid`, `application_name`, `backend_xid`, `backend_xmin` | Shown | Shown | Shown | [pgstatfuncs.c#activity-public-fields](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L625-L651) |
| `relid`, `phase`, and counters in `pg_stat_progress_vacuum` | Shown | NULL unless the reader is a member of the reporting role | NULL unless a member | [pgstatfuncs.c#pg_stat_get_progress_info-access](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L516-L532) |
| `pg_settings` rows marked `GUC_SUPERUSER_ONLY`, such as `shared_preload_libraries` and `data_directory` | Shown | Shown, through `pg_read_all_settings` | Row hidden | [guc.c#pg_settings-access](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L8998-L9008) |
| Other users' `queryid` and `query` in `pg_stat_statements` | Shown | Shown | `queryid` NULL, text `<insufficient privilege>` | [pg_stat_statements.c#query-redaction](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1551-L1601) |
| Details in `pg_stat_replication` and `pg_stat_wal_receiver` | Shown | Shown | `pid` only | [walsender.c#detail-gate](../../../../raw/postgres-12/src/backend/replication/walsender.c#L3313-L3321) [walreceiver.c#detail-gate](../../../../raw/postgres-12/src/backend/replication/walreceiver.c#L1397-L1405) |
| `pg_ls_waldir()` and `pg_ls_archive_statusdir()` | Allowed | Allowed | `permission denied for function` | [system_views.sql#monitor-file-functions](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1320-L1347) |
| [`pgstattuple`](../../../glossary.md#pgstattuple) and [`pgstatindex`](../../../glossary.md#pgstatindex) | Allowed | Allowed, through `pg_stat_scan_tables` | Not granted | [pgstattuple--1.4--1.5.sql#scan-role-grants](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L6-L119) |
| `pg_subscription.oid` | Allowed | `permission denied for table pg_subscription` | `permission denied for table pg_subscription` | [system_views.sql#pg_subscription-column-grants](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1059-L1062) [pg_subscription.h#oid](../../../../raw/postgres-12/src/include/catalog/pg_subscription.h#L41) [execMain.c#column-privileges](../../../../raw/postgres-12/src/backend/executor/execMain.c#L676-L692) |

Three consequences follow, and the first two catch ordinary roles quietly.

1. The progress gate tests `has_privs_of_role(GetUserId(), <reporting role>)`
   and never consults `pg_read_all_stats`. A `pg_monitor` member therefore sees
   another role's `VACUUM` as a row with only `pid` and `datname`.
   [pgstatfuncs.c#pg_stat_get_progress_info-access](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L516-L532)
2. `backend_type` is one of the redacted fields. Without `pg_read_all_stats`,
   the connection-capacity query in section 1 counts only the reader's own
   client backends, and the long-transaction query drops other users' rows
   because their `xact_start` is NULL.
   [pgstatfuncs.c#activity-redaction](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L885-L911)
3. PostgreSQL 12 grants PUBLIC only six columns of `pg_subscription`, and `oid`
   is not one of them. The executor checks `SELECT` privilege on every column a
   query references, so any non-superuser query that names
   `pg_subscription.oid` fails. Section 7 therefore avoids that column.
   [system_views.sql#pg_subscription-column-grants](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1059-L1062)
   [execMain.c#ExecCheckRTPerms](../../../../raw/postgres-12/src/backend/executor/execMain.c#L580-L586)

Measured on this pin by the [measurement script](#measurement-script): during
a superuser `VACUUM`, the superuser saw `relid`, phase `scanning heap`, and
`heap_blks_total = 173`, while the `pg_monitor` member and the ordinary role
both saw only `pid` and `datname`. Of the three probed settings, the ordinary
role saw only `archive_command`. In `pg_stat_activity` the ordinary role saw
all 8 rows but 7 with a NULL `backend_type`, so it counted 1 client backend
where the superuser counted 2. In `pg_stat_statements` every other-user row
had a NULL `queryid` and redacted text for the ordinary role, and none did for
`pg_monitor`. The subscription query filed earlier on this page, which joined
on `pg_subscription.oid`, failed for `pg_monitor` with `permission denied for
table pg_subscription` and worked for the superuser.

### When counters reset

A cumulative comparison is valid only inside one counter epoch. The events
below end an epoch. The first three change `stats_reset`. The recovery events
wipe the collector's counters entirely, which a health check can mistake for an
idle, clean database.

| Event | State before | State after | Effect on later readings | Source |
|---|---|---|---|---|
| `pg_stat_reset()` in a database | Accumulated database, table, and function counters | That database's counters and table entries are empty; `stats_reset` is now | Deltas across the reset are invalid; compare `stats_reset` first | [pgstat.c#pgstat_recv_resetcounter](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6097-L6122) [pgstat.c#reset_dbentry_counters](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L4714-L4741) |
| `pg_stat_reset_shared('bgwriter')` or `('archiver')` | Cluster-wide counters | That view is zero; its `stats_reset` is now | Same as above, for `pg_stat_bgwriter` or `pg_stat_archiver` | [pgstat.c#pgstat_recv_resetsharedcounter](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6134-L6144) |
| `pg_stat_reset_single_table_counters()` | One table's counters | That table is zero; the database's `stats_reset` is now | The database timestamp moves although other tables kept their history | [pgstat.c#single-counter-reset](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6166-L6172) |
| Clean shutdown, then a start that needs no recovery | Counters in the collector | The collector wrote them at exit and reads them at start | Counters continue | [pgstat.c#collector-exit-write](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L4674-L4680) [pgstat.c#collector-start-read](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L4455-L4461) |
| Backend crash with the default `restart_after_crash = on`, or `pg_ctl -m immediate` | Counters in the collector | Startup performs recovery and deletes every statistics file; new `stats_reset` values appear when entries are recreated | Every table looks freshly reset. Autovacuum skips a table with no statistics entry except for wraparound, and later counts only post-restart dead rows | [xlog.c#recovery-decision](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6733-L6746) [xlog.c#pgstat_reset_all-call](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6843-L6846) [pgstat.c#pgstat_reset_all](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L680-L690) [pgstat.c#new-reset-timestamps](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L5163-L5167) [autovacuum.c#no-stats-entry](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3082-L3090) |
| Any standby start, including after a clean standby shutdown | The standby's own counters, such as `pg_stat_database_conflicts` | A standby always performs archive recovery at start, so the same wipe happens | Standby conflict history restarts at every start | [xlog.c#recovery-decision](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6733-L6746) |
| `pg_stat_statements_reset()` or entry eviction | Per-statement totals | Reset entries are gone; eviction discards the least-executed entries when more than `pg_stat_statements.max` statements are seen | A first-only or negative delta means the interval is broken | [pgstatstatements.sgml#reset](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L333-L357) [pgstatstatements.sgml#entry-eviction](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L399-L410) |
| Clean shutdown or `pg_ctl -m immediate`, with the default `pg_stat_statements.save = on` | Per-statement totals | The postmaster exits with code 0, the module dumps its entries, and the next start reloads them | Statement totals survive although the collector's counters were wiped | [postmaster.c#immediate-shutdown-not-fatal](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3618-L3619) [postmaster.c#normal-exit](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3874-L3896) [pg_stat_statements.c#pgss_shmem_shutdown](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L692-L702) [pg_stat_statements.c#load-dump](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L546-L560) |
| Backend crash with reinitialization | Per-statement totals | The postmaster runs `shmem_exit(1)`, the module skips its dump, and the dump read at the last start was already deleted | Statement totals are lost | [postmaster.c#crash-reinitialize](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3915-L3928) [pg_stat_statements.c#pgss_shmem_shutdown](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L692-L702) [pg_stat_statements.c#remove-dump](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L626-L639) |

Two of these rows depart from what a reader might expect, so their cause is
worth stating. An immediate shutdown is not a crash to the postmaster:
`HandleChildCrash()` sets `FatalError` only when the shutdown mode is not
immediate, so the postmaster takes its normal exit path. That is why
`pg_stat_statements` survives it. The collector's counters do not survive
either path, because the next start must run crash recovery, and recovery
always calls `pgstat_reset_all()`.
[postmaster.c#immediate-shutdown-not-fatal](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3618-L3619)
[pg_ctl-ref.sgml#immediate](../../../../raw/postgres-12/doc/src/sgml/ref/pg_ctl-ref.sgml#L192-L197)
[xlog.c#pgstat_reset_all-call](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6843-L6846)

Measured on this pin: after a `SIGKILL` of one backend, the postmaster logged
`all server processes terminated; reinitializing` and crash recovery. A table
with 500 counted dead rows then showed `n_live_tup = 0` and `n_dead_tup = 0`,
both `stats_reset` values were new, and a marker statement disappeared from
`pg_stat_statements`. After `pg_ctl -m immediate`, 250 counted dead rows also
read 0 and both timestamps moved again, but a second marker statement was still
in `pg_stat_statements`. On the standby, a clean restart reset
`confl_snapshot` from 2 to 0 and moved `stats_reset`. The test table had
`autovacuum_enabled = false` (default `true`) so that no autovacuum could change
its counters between readings, and `pg_stat_statements` was preloaded; the
[measurement script](#measurement-script) lists both.

### How to use this checklist

The steps run in this order because each later step depends on the earlier
ones being true.

1. Run the observability query and resolve disabled, missing, or
   `pending_restart` settings before interpreting empty data. Missing
   `pg_stat_statements.*` rows mean the module was not preloaded: it defines
   those settings only during `shared_preload_libraries` loading, and an
   unloaded custom setting is a hidden placeholder.
   [pg_stat_statements.c#_PG_init](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L351-L360)
   [guc.c#placeholder-flags](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4976)
2. Choose the monitoring role with [Who can see what](#who-can-see-what). Use a
   superuser for a complete capture on this pin. A `pg_monitor` member can run
   every fenced query on this page, but its progress rows and its view of
   `pg_subscription` are limited.
3. Run the core checks in every application database. Run cluster-wide checks,
   such as `pg_database`, `pg_stat_bgwriter`, `pg_stat_archiver`, and the
   replication views, once.
4. Compare each cumulative result with a previous capture from the same epoch.
   `pg_stat_database`, `pg_stat_bgwriter`, and `pg_stat_archiver` expose reset
   timestamps; the per-table views do not. Check the log for crash recovery or
   a standby restart in the interval, because either one wipes the counters;
   see [When counters reset](#when-counters-reset).
   [monitoring.sgml#DynamicStatisticsViews](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L294-L360)
   [monitoring.sgml#CollectedStatisticsViews](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L386-L425)
   [monitoring.sgml#pg_stat_all_tables-columns](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2704-L2845)
5. Search the server log over the same interval and correlate sessions by
   process ID. The PostgreSQL 12 default `log_line_prefix` is `'%m [%p] '`,
   which already prints a timestamp and the process ID.
   [guc.c#log_line_prefix](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3592-L3600)
   [config.sgml#log_min_duration_statement](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5956-L5968)
6. Interpret findings with the log table and
   [Issue interpretation and response](#issue-interpretation-and-response).

If a diagnostic transaction must refresh collected statistics before another
read, clear its cached snapshot explicitly:

```sql
SELECT /* wiki_db_health_clear_stats_snapshot */ pg_stat_clear_snapshot();
```

The next statistics read fetches a new snapshot, so it no longer belongs to the
transaction's original stable capture. The upstream statistics test uses the
same call between polls.
[monitoring.sgml#pg_stat_clear_snapshot](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L254-L259)
[stats.sql#clear-snapshot-between-polls](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L66-L70)

### Production query guardrails

Start a diagnostic session with finite timeouts:

```sql
SET /* wiki_db_health_guardrails */ statement_timeout = '30s';
SET /* wiki_db_health_guardrails */ lock_timeout = '2s';
```

Both settings default to 0, which disables the timeout. Both are
`PGC_USERSET`, so `SET` changes them for the session only.
[guc.c#statement_timeout-lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2397)
[guc.c#GucContext_Names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L594-L603)

The catalog and statistics queries below do not intentionally take
table-changing locks. The optional `pgstattuple` checks read relation pages
under a read lock, and their results can change during concurrent updates.
Run those only for selected relations, preferably outside peak traffic, and
raise `statement_timeout` only for that diagnostic session if required.
[pgstattuple.sgml#pgstattuple-locking](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L123-L142)

### Observability prerequisites

The table names what must exist for each kind of evidence, and the apply scope
of each setting ([GUC context](../../../glossary.md#guc-context)). Defaults are
listed in [Defaults that decide what you see](#defaults-that-decide-what-you-see).

| Data or signal | What must be enabled, installed, or configured | Apply scope |
|---|---|---|
| Complete cross-role monitoring | Use a superuser for a complete capture on this pin; see [Who can see what](#who-can-see-what) for each limit of `pg_monitor` and of ordinary roles. [user-manag.sgml#default-roles](../../../../raw/postgres-12/doc/src/sgml/user-manag.sgml#L519-L538) [pgstatstatements.sgml#access](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L228-L234) | Role membership is not a setting. Re-run the capture after changing the monitoring role. |
| PostgreSQL-managed text or CSV log files | Use `stderr` or `csvlog` in `log_destination`. CSV requires `logging_collector`. `current_logfiles` records the active file only while the collector manages `stderr` or `csvlog`. With the collector off, `stderr` goes wherever the postmaster's standard error points. If the deployment uses `syslog` or Windows `eventlog`, inspect that destination instead. [config.sgml#log_destination](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5439-L5486) | `logging_collector` is `PGC_POSTMASTER`: restart. `log_destination` is `PGC_SIGHUP`: reload. [guc.c#logging_collector](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1616-L1624) [guc.c#log_destination](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3848-L3859) |
| Correlatable log lines | Keep a process ID or session ID in `log_line_prefix`. The default already has the PID, so change it only if it was overridden or you need user, database, or client fields. [config.sgml#log_min_duration_statement](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5956-L5968) [guc.c#log_line_prefix](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3592-L3600) | `PGC_SIGHUP`: reload. |
| Server-log severity and diagnostic detail | Ensure `log_min_messages` admits the severities you search. In this server-log ranking `LOG` sits above `ERROR`, so the default `warning` already keeps LOG lines. `log_min_error_statement` controls whether an error's SQL text is logged, and `log_error_verbosity` controls detail. [config.sgml#server-log-levels](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5877-L5927) | All three are `PGC_SUSET`: a superuser can `SET` them per session; a cluster default is a configuration-file or `ALTER SYSTEM` change followed by a reload. [guc.c#log-level-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4286-L4316) |
| Current sessions, transaction age, query text, and waits | Keep `track_activities` enabled. `pg_stat_activity` exposes state, transaction and query start times, waits, query text, and `backend_xmin`. Raise `track_activity_query_size` if 1024 bytes cuts off useful query text. [system_views.sql#pg_stat_activity](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L756) [config.sgml#track_activity_query_size](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6820-L6834) | `track_activities` is `PGC_SUSET`. [`track_activity_query_size`](../../../glossary.md#track_activity_query_size) is `PGC_POSTMASTER`: restart. [guc.c#track_activities](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1381-L1391) [guc.c#track_activity_query_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3164-L3173) |
| Database and table counters | Keep `track_counts` enabled. It feeds the collected statistics and is required for normal [autovacuum](../../../glossary.md#autovacuum). [config.sgml#track_counts](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6838-L6850) [config.sgml#autovacuum](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6985-L7005) | `PGC_SUSET`. Set it in the configuration file so every session and autovacuum collect. [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1400) |
| Database, `EXPLAIN (BUFFERS)`, and `pg_stat_statements` I/O time | Enable `track_io_timing` when the platform's timing overhead is acceptable. It supplies `blk_read_time` and `blk_write_time`. [config.sgml#track_io_timing](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6854-L6871) | `PGC_SUSET`. [guc.c#track_io_timing](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1401-L1409) |
| Slow statements, lock waits, temp files, checkpoints, autovacuum, connection lifecycle, and replication commands in the log | Configure the matching logging setting. `deadlock_timeout` decides when `log_lock_waits` writes its line and when the deadlock check runs. [config.sgml#log_min_duration_statement](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5931-L5953) [config.sgml#logging-operational-events](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6165-L6222) [config.sgml#lock-and-temp-logging](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6477-L6574) [config.sgml#log_autovacuum_min_duration](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7010-L7033) | `log_checkpoints` and `log_autovacuum_min_duration` are `PGC_SIGHUP`: reload. `log_connections` and `log_disconnections` are `PGC_SU_BACKEND`: a reload updates the postmaster, which passes the value only to sessions started afterwards. The other settings and `deadlock_timeout` are `PGC_SUSET`. [guc.c#logging-booleans](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1217-L1252) [guc.c#log_lock_waits](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1489-L1497) [guc.c#deadlock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2062-L2072) [guc.c#logging-thresholds](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2703-L2725) [guc.c#log_temp_files](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3153-L3162) [guc.c#PGC_SU_BACKEND-reload](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6811-L6858) |
| Per-statement workload totals | Install the PostgreSQL 12 `pg_stat_statements` [contrib](../../../glossary.md#contrib) module, add it to [`shared_preload_libraries`](../../../glossary.md#shared_preload_libraries), restart, and run `CREATE /* wiki_enable_pgss */ EXTENSION pg_stat_statements;` in each database that needs the view. Confirm that `pg_stat_statements.track` is not `none`. [pgstatstatements.sgml#loading](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L10-L30) | `shared_preload_libraries` and `pg_stat_statements.max` need a restart. `pg_stat_statements.track` and `track_utility` are `PGC_SUSET`; `pg_stat_statements.save` is `PGC_SIGHUP`. [guc.c#shared_preload_libraries](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3767-L3776) [pg_stat_statements.c#module-GUCs](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L365-L410) |
| Exact or approximate heap [bloat](../../../glossary.md#bloat) and B-tree structure | Install the PostgreSQL 12 `pgstattuple` module and run `CREATE /* wiki_enable_pgstattuple */ EXTENSION pgstattuple;` in the target database. Its default version 1.5 is built by the 1.4 script plus the 1.4-to-1.5 update, which revokes public execution and grants the functions to `pg_stat_scan_tables`. [pgstattuple.sgml#module-and-privileges](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L23) [pgstattuple.control](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5) [pgstattuple--1.4--1.5.sql#scan-role-grants](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L6-L119) | Extension creation is per database. [pgstattuple--1.4.sql#functions](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L3-L20) |
| Checksum failures | [Data checksums](../../../glossary.md#data-checksums) must be enabled; `data_checksums` reports it. PostgreSQL verifies a checksum when it reads a page, so a zero counter says nothing about pages nobody read. Keep `ignore_checksum_failure` and `zero_damaged_pages` off; [Defaults that decide what you see](#defaults-that-decide-what-you-see) explains what each one changes. [bufmgr.c#checksum-read-path](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925) [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L145-L161) [monitoring.sgml#checksum_failures](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2599-L2611) | `data_checksums` is `PGC_INTERNAL`. Enabling checksums needs `initdb --data-checksums` or, after a clean shutdown, offline `pg_checksums --enable`. Offline `pg_checksums` also verifies every file. [guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835) [initdb.sgml#data-checksums](../../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L225) [pg_checksums.sgml#operation](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L38-L51) [pg_checksums.sgml#notes](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L204-L225) |
| [WAL archiving](../../../glossary.md#wal-archiving), physical and logical replication, and [slots](../../../glossary.md#replication-slot) | Define the expected topology first. Physical streaming needs a [`wal_level`](../../../glossary.md#wal-level) of `replica` or higher (the default), sender capacity, an authorized replication connection, and standby settings. Logical replication needs `wal_level = logical` on the publisher plus slot, sender, logical-worker, and background-worker capacity. An empty view is expected only when that component is absent by design. `pg_stat_replication` lists directly connected consumers only. [config.sgml#wal_level](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2453-L2462) [high-availability.sgml#physical-replication-setup](../../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L666-L715) [logical-replication.sgml#logical-replication-config](../../../../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L545-L573) [monitoring.sgml#direct-replication-connections](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1978-L1983) | `wal_level`, `archive_mode`, `max_wal_senders`, `max_replication_slots`, `primary_conninfo`, `primary_slot_name`, `hot_standby`, `max_worker_processes`, and `max_logical_replication_workers` need a restart. `archive_command`, `archive_timeout`, `wal_keep_segments`, standby feedback and delays, receiver status interval and timeout, `synchronous_standby_names`, and `max_sync_workers_per_subscription` reload. `wal_sender_timeout` and `synchronous_commit` are `PGC_USERSET`. [guc.c#archive_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1964-L1974) [guc.c#standby-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1752-L1770) [guc.c#standby-and-receiver-timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2074-L2127) [guc.c#wal_keep_segments](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2532-L2540) [guc.c#sender-capacity-and-timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2635-L2665) [guc.c#logical-worker-capacity](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2788-L2822) [guc.c#archive_command](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3454-L3462) [guc.c#standby-connection-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3560-L3579) [guc.c#synchronous_standby_names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4086-L4095) [guc.c#synchronous_commit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4353-L4361) [guc.c#archive_mode](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4363-L4371) [guc.c#wal_level](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4409-L4417) |

A cluster-wide value for a `PGC_SUSET` or `PGC_USERSET` setting is a
configuration-file or `ALTER SYSTEM` change followed by a reload. The reload
reaches running sessions, because the setting's context accepts a `SIGHUP`
change and each backend re-reads the file when the reload is pending. A
session that already ran `SET` keeps its own value, because a value from a
higher-priority source is not overwritten by the file.
[guc.c#PGC_SUSET-accepts-sighup](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6859-L6870)
[postgres.c#ConfigReloadPending](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4216-L4220)
[guc.c#source-priority](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6925-L6941)

Check the effective settings before collecting evidence:

```sql
SELECT /* wiki_db_health_observability */
       name,
       CASE
         WHEN name = 'primary_conninfo' AND setting = '' THEN '(empty)'
         WHEN name = 'primary_conninfo' THEN '(set; redacted)'
         ELSE setting
       END AS setting,
       unit,
       boot_val,
       context,
       source,
       pending_restart
FROM pg_settings
WHERE name IN (
    'archive_command',
    'archive_mode',
    'archive_timeout',
    'autovacuum',
    'autovacuum_freeze_max_age',
    'autovacuum_multixact_freeze_max_age',
    'checkpoint_warning',
    'data_checksums',
    'deadlock_timeout',
    'hot_standby',
    'hot_standby_feedback',
    'idle_in_transaction_session_timeout',
    'ignore_checksum_failure',
    'log_autovacuum_min_duration',
    'log_checkpoints',
    'log_connections',
    'log_destination',
    'log_disconnections',
    'log_error_verbosity',
    'log_line_prefix',
    'log_lock_waits',
    'log_min_duration_statement',
    'log_min_error_statement',
    'log_min_messages',
    'log_replication_commands',
    'log_temp_files',
    'logging_collector',
    'max_connections',
    'max_locks_per_transaction',
    'max_logical_replication_workers',
    'max_prepared_transactions',
    'max_replication_slots',
    'max_standby_archive_delay',
    'max_standby_streaming_delay',
    'max_sync_workers_per_subscription',
    'max_wal_senders',
    'max_wal_size',
    'max_worker_processes',
    'pg_stat_statements.max',
    'pg_stat_statements.save',
    'pg_stat_statements.track',
    'pg_stat_statements.track_utility',
    'primary_conninfo',
    'primary_slot_name',
    'restart_after_crash',
    'shared_preload_libraries',
    'superuser_reserved_connections',
    'synchronous_commit',
    'synchronous_standby_names',
    'temp_file_limit',
    'track_activities',
    'track_activity_query_size',
    'track_counts',
    'track_io_timing',
    'wal_level',
    'wal_keep_segments',
    'wal_receiver_status_interval',
    'wal_receiver_timeout',
    'wal_sender_timeout',
    'zero_damaged_pages'
)
ORDER BY name;
```

`pg_settings` is backed by `pg_show_all_settings()`, and `pending_restart` marks
a configuration-file change that needs a restart to take effect.
[system_views.sql#pg_settings](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L512-L524)
[guc.c:9211](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L9211)
[catalogs.sgml#pg_settings_pending_restart](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10494-L10498)

Absence is not proof of health. A `GUC_SUPERUSER_ONLY` row is filtered out for
an unprivileged reader, and a module setting is absent until its module loads.
Activity and extension fields can be NULL or replaced by a placeholder; see
[Who can see what](#who-can-see-what). An event-specific log line can also be
absent simply because no qualifying event happened. Resolve configuration and
access first.
[guc.c#pg_settings-access](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L8998-L9008)
[pg_stat_statements.c#query-redaction](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1588-L1601)

### Defaults that decide what you see

Most event logging is off in a default PostgreSQL 12 server, and several safety
settings change behavior only when they leave their default. The table gives
each default, what the default does, and what a non-default value changes. The
script's `default` stage read every value in the "PostgreSQL 12 default"
column from a fresh cluster's `pg_settings`.

| Setting | PostgreSQL 12 default | What the default does | What a non-default value changes | Apply scope | Source |
|---|---|---|---|---|---|
| `log_min_duration_statement` | `-1` | Logs no statement durations | `0` logs every completed statement; `N` logs statements that ran at least `N` ms | `PGC_SUSET` | [guc.c#log_min_duration_statement](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2703-L2713) [postgres.c#check_log_duration](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L2210-L2247) |
| `log_autovacuum_min_duration` | `-1` | Logs no autovacuum completions or skips; autovacuum ERRORs are still logged | `0` logs every autovacuum action and every skip caused by a lock or a dropped table; a table's storage parameter can override it | `PGC_SIGHUP` | [guc.c#log_autovacuum_min_duration](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2715-L2725) [vacuumlazy.c#autovacuum-log-gate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L372-L380) [config.sgml#log_autovacuum_min_duration](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7010-L7033) |
| `log_temp_files` | `-1` | Logs no temporary files; `temp_files` and `temp_bytes` still count them | `0` logs every [temporary file](../../../glossary.md#temporary-file) when it is deleted; `N` logs files of at least `N` kB | `PGC_SUSET` | [guc.c#log_temp_files](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3153-L3162) [fd.c#ReportTemporaryFileUsage](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1272-L1287) |
| `log_checkpoints` | `off` | Logs no checkpoint or restartpoint start and end lines | Logs both, with the cause flags and write and sync times | `PGC_SIGHUP` | [guc.c#log_checkpoints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1217-L1225) [xlog.c:8710](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8710) [xlog.c:9166](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9166) [xlog.c#LogCheckpointEnd](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8408-L8413) |
| `log_lock_waits` | `off` | Logs no lock-wait lines; a `deadlock detected` ERROR is still logged | Logs a waiter after `deadlock_timeout`, with holder and queue PIDs | `PGC_SUSET` | [guc.c#log_lock_waits](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1489-L1497) [proc.c#lock-wait-log](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1377-L1381) |
| `log_connections`, `log_disconnections` | `off` | Logs no connection lifecycle | Logs attempts and authorization, or session end with duration | `PGC_SU_BACKEND` | [guc.c#logging-booleans](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1226-L1243) |
| `log_line_prefix` | `'%m [%p] '` | Prints a millisecond timestamp and the process ID | Can add user, database, client, and session fields | `PGC_SIGHUP` | [guc.c#log_line_prefix](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3592-L3600) |
| `log_min_messages` | `warning` | Keeps WARNING and higher, which in this ranking includes LOG | A lower level adds NOTICE, INFO, and DEBUG noise; a higher level drops warnings | `PGC_SUSET` | [guc.c#log_min_messages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4296-L4305) [config.sgml#log_min_messages](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5885-L5896) |
| `track_io_timing` | `off` | I/O time columns stay 0 | Times every data-file read and write | `PGC_SUSET` | [guc.c#track_io_timing](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1401-L1409) [bufmgr.c#io-timing](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L894-L905) |
| `max_prepared_transactions` | `0` | `PREPARE TRANSACTION` fails with `prepared transactions are disabled`, so `pg_prepared_xacts` stays empty | Allows prepared transactions, which keep their locks until committed or rolled back and, while they last, hold back VACUUM | `PGC_POSTMASTER` | [guc.c#max_prepared_transactions](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2344-L2352) [twophase.c#feature-disabled](../../../../raw/postgres-12/src/backend/access/transam/twophase.c#L385-L390) [prepare_transaction.sgml#caution](../../../../raw/postgres-12/doc/src/sgml/ref/prepare_transaction.sgml#L125-L136) |
| `ignore_checksum_failure` | `off` | A checksum mismatch gives a WARNING and a counter increment, then an `invalid page in block` ERROR; the page is not loaded | A page whose header looks sane is loaded anyway after the WARNING, so the query reads possibly corrupt data; the documentation warns this can cause crashes or hide corruption | `PGC_SUSET` | [guc.c#ignore_checksum_failure](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1134-L1148) [bufpage.c#checksum-failure-handling](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L144-L161) [config.sgml#ignore_checksum_failure](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L9727-L9740) |
| `zero_damaged_pages` | `off` | An invalid page gives an ERROR | The page is zeroed in memory with a WARNING, so its rows vanish from the query. The zeroed page is not forced to disk, so the documentation recommends recreating the table or index before turning the setting off again | `PGC_SUSET` | [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1149-L1162) [bufmgr.c#zero-damaged-pages](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925) [config.sgml#zero_damaged_pages](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L9752-L9767) |
| `checkpoint_warning` | `30s` | Logs the frequency message when WAL-caused checkpoints start less than 30 s apart | `0` disables the message | `PGC_SIGHUP` | [guc.c#checkpoint_warning](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2577-L2589) [checkpointer.c#checkpoint-warning](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462) |
| `max_standby_streaming_delay` | `30s` | Replay waits up to 30 s past WAL receipt before cancelling conflicting standby queries | `-1` waits forever; smaller values cancel sooner | `PGC_SIGHUP` | [guc.c#standby-conflict-delays](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2085-L2094) [standby.c#GetStandbyLimitTime](../../../../raw/postgres-12/src/backend/storage/ipc/standby.c#L154-L169) |
| `hot_standby_feedback` | `off` | The primary can remove rows a standby query still needs, which causes snapshot conflicts | The standby reports its xmin, which avoids those cancellations but can bloat the primary | `PGC_SIGHUP` | [guc.c#hot_standby_feedback](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1762-L1770) [config.sgml#hot_standby_feedback](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4139-L4161) |
| `restart_after_crash` | `on` | After a backend crash the postmaster terminates every child, reinitializes, and runs crash recovery | The postmaster exits instead, leaving the server down | `PGC_SIGHUP` | [guc.c#restart_after_crash](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1277-L1285) [postmaster.c#no-restart-exit](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3907-L3909) |

The data checksum switch is not a setting at all: `initdb` leaves checksums off
unless `--data-checksums` is given, and `data_checksums` only reports the
result. [initdb.sgml#data-checksums](../../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L225)
[guc.c#data_checksums](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1826-L1835)

Measured on this pin: on a fresh cluster with these defaults, `PREPARE
TRANSACTION` failed with `prepared transactions are disabled` and the hint
`Set max_prepared_transactions to a nonzero value.` On a checksum-enabled
cluster, one changed byte in a page's free space produced, by default, the
WARNING `page verification failed, calculated checksum ... but expected ...`
and the ERROR `invalid page in block 0 of relation base/...`. With
`ignore_checksum_failure = on` the same table returned all 100 rows after the
WARNING. With `zero_damaged_pages = on` a second damaged table returned 0 rows
after the WARNING `invalid page in block 0 of relation base/...; zeroing out
page`. `checksum_failures` then read 3, one per failed verification.

### SQL health checklist

**1. Current sessions, old transactions, and waits**

```sql
SELECT /* wiki_db_health_activity_summary */
       backend_type,
       state,
       wait_event_type,
       wait_event,
       count(*) AS processes,
       min(xact_start) AS oldest_xact_start,
       min(query_start) FILTER (WHERE state = 'active') AS oldest_active_query_start
FROM pg_stat_activity
GROUP BY backend_type, state, wait_event_type, wait_event
ORDER BY processes DESC, backend_type, state, wait_event_type, wait_event;

SELECT /* wiki_db_health_connection_capacity */
       count(*) FILTER (WHERE backend_type = 'client backend') AS client_backends,
       count(*) FILTER (WHERE backend_type IS NULL) AS hidden_backend_type,
       current_setting('max_connections')::integer AS max_connections,
       current_setting('superuser_reserved_connections')::integer AS reserved_connections,
       current_setting('max_connections')::integer
         - current_setting('superuser_reserved_connections')::integer
         AS ordinary_client_capacity
FROM pg_stat_activity;

SELECT /* wiki_db_health_long_transactions */
       pid,
       usename,
       application_name,
       backend_type,
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
the [wait event](../../../glossary.md#wait-event) fields. Its rows include
client [backends](../../../glossary.md#backend), autovacuum processes, WAL
processes, [background workers](../../../glossary.md#background-worker), and
other server processes. A `Lock` wait is a
[heavyweight lock](../../../glossary.md#heavyweight-lock) wait. A `BufferPin`
wait can be prolonged by another process holding an open cursor on that
[buffer](../../../glossary.md#buffer-pin).
[monitoring.sgml#pg_stat_activity-timestamps-and-waits](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L659-L718)
[monitoring.sgml#backend_type](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L839-L860)
[pgstat.c#pgstat_get_backend_desc](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L4273-L4312)

Checklist:

- [ ] Compare only `client backend` rows with `max_connections`, and keep the
  `superuser_reserved_connections` margin. Autovacuum workers, background
  workers, and WAL senders draw from separate PGPROC pools, which PostgreSQL
  sizes separately in `MaxBackends`, so do not count them as client
  connections. A nonzero `hidden_backend_type` means the reader lacks
  `pg_read_all_stats` and the count is too low.
  [guc.c#max_connections-reserved](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2129-L2148)
  [postinit.c#InitializeMaxBackends](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L527-L539)
  [proc.c#InitProcess-free-lists](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L319-L327)
  [pgstatfuncs.c#activity-redaction](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L885-L911)
- [ ] Investigate old active or `idle in transaction` sessions, especially rows
  with an old `backend_xmin`, the session's [xmin horizon](../../../glossary.md#xmin-horizon)
  contribution. VACUUM warns when the oldest xmin is far in the past and points
  to open transactions, old prepared transactions, and stale replication slots.
  [vacuum.c#oldest-xmin-warning](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L933-L949)
- [ ] Investigate sustained `Lock`, `BufferPin`, or `IO` waits, and correlate
  them with the lock and log checks below.
  [monitoring.sgml#wait_event_type](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L687-L760)

**2. Blocking locks and deadlocks**

```sql
SELECT /* wiki_db_health_blockers */
       waiter.pid AS waiting_pid,
       waiter.usename AS waiting_user,
       waiter.application_name AS waiting_application,
       waiter.wait_event AS waiting_event,
       now() - waiter.query_start AS waiting_query_age,
       blocker_pid.pid AS blocking_pid,
       CASE WHEN blocker_pid.pid = 0 THEN 'prepared transaction'
            ELSE blocker.backend_type
       END AS blocking_type,
       blocker.usename AS blocking_user,
       blocker.application_name AS blocking_application,
       blocker.state AS blocking_state,
       waiter.query AS waiting_query,
       blocker.query AS blocking_query
FROM pg_stat_activity AS waiter
CROSS JOIN LATERAL
     unnest(pg_blocking_pids(waiter.pid)) AS blocker_pid(pid)
LEFT JOIN pg_stat_activity AS blocker
  ON blocker.pid = NULLIF(blocker_pid.pid, 0)
WHERE waiter.wait_event_type = 'Lock'
ORDER BY waiting_query_age DESC, waiting_pid, blocking_pid;

SELECT /* wiki_db_health_prepared_transactions */
       transaction,
       gid,
       prepared,
       now() - prepared AS prepared_age,
       owner,
       database
FROM pg_prepared_xacts
ORDER BY prepared;

SELECT /* wiki_db_health_prepared_transaction_locks */
       p.transaction,
       p.gid,
       p.prepared,
       now() - p.prepared AS prepared_age,
       p.owner,
       p.database,
       l.locktype,
       l.mode,
       l.granted,
       l.database AS lock_database_oid,
       l.relation AS relation_oid
FROM pg_prepared_xacts AS p
JOIN pg_locks AS l
  ON l.virtualtransaction = '-1/' || p.transaction::text
WHERE l.granted
ORDER BY p.prepared, p.gid, l.locktype, l.mode;
```

`pg_blocking_pids()` reports hard blockers and sessions ahead of the waiter in
the [lock queue](../../../glossary.md#lock-queue). It returns PID zero when a
prepared transaction is the blocker. A prepared transaction never waits, but
it keeps the locks it acquired. Its `pg_locks.pid` is NULL, and its
[virtual transaction ID](../../../glossary.md#virtual-transaction-id) is shown
as `-1/<transaction>`. Keep cluster-wide database and relation identifiers as
OIDs rather than casting them in the current database.
[func.sgml#pg_blocking_pids](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L17584-L17604)
[catalogs.sgml#pg_locks-pid](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9219-L9227)
[lockfuncs.c#pid-null-for-prepared](../../../../raw/postgres-12/src/backend/utils/adt/lockfuncs.c#L311-L312)
[catalogs.sgml#prepared-transaction-locks](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9300-L9331)
[catalogs.sgml#pg_prepared_xacts](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9627-L9710)

Prepared transactions exist only on a server whose `max_prepared_transactions`
is above its default of 0; at the default, `PREPARE TRANSACTION` fails and the
two prepared-transaction queries return nothing. See
[Defaults that decide what you see](#defaults-that-decide-what-you-see).
[twophase.c#feature-disabled](../../../../raw/postgres-12/src/backend/access/transam/twophase.c#L385-L390)

Measured on this pin, with `max_prepared_transactions = 10`: a prepared
transaction holding `AccessExclusiveLock` on a table made a waiting `SELECT`
report `blocking_pid = 0` and `blocking_type = prepared transaction`. The
prepared-lock query returned the retained `relation` lock and the
transaction's own `transactionid` `ExclusiveLock`, both with a NULL `pid` and
`virtualtransaction = '-1/<transaction>'`. With `log_lock_waits = on` (default
`off`, set in the script's `dh12.conf`), the wait line named the holder as
`Process holding the lock: 0.`

The prepared-lock query lists possible prepared blockers. When several prepared
transactions exist, PID zero alone does not map a waiter to one GID; an exact
match would need lock tags, conflict modes, and queue order. Poll
`pg_blocking_pids()` at a modest interval, because each call briefly takes
exclusive access to the lock manager's shared state.
[func.sgml#pg_blocking_pids-locking](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L17584-L17604)

Checklist:

- [ ] Investigate each reported blocker and the waiter's requested operation.
  Do not infer blockers by self-joining `pg_locks`: queue order, parallel
  query groups, and prepared transactions make that incomplete.
  [catalogs.sgml#pg_locks-blocker-identification](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9334-L9345)
- [ ] Enable `log_lock_waits` to get holder and waiter detail after
  `deadlock_timeout`. `deadlock_timeout` also decides when PostgreSQL runs the
  [deadlock](../../../glossary.md#deadlock) check. Both are `PGC_SUSET`.
  [config.sgml#log_lock_waits](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6477-L6490)
  [guc.c#deadlock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2062-L2072)
  [proc.c#lock-wait-log](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1377-L1497)
- [ ] Treat any increase in `pg_stat_database.deadlocks` as an application or
  operational concurrency defect. PostgreSQL detects the deadlock, counts it,
  and aborts one transaction with `deadlock detected`. Measured on this pin:
  a two-row, two-session deadlock aborted exactly one session and moved
  `deadlocks` from 0 to 1.
  [deadlock.c#DeadLockReport](../../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L1139-L1146)
  [monitoring.sgml#deadlocks](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2594-L2598)

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
  checksum failures, and I/O time. Temporary-file counters include every temp
  file regardless of `log_temp_files`, because one call site both counts and
  logs each file. I/O time needs `track_io_timing`. Checksum counters are NULL
  when checksums are disabled. A zero checksum delta covers only the pages
  read during the interval; see `Open Questions` for a base-backup accounting
  defect in this pin. Measured on this pin: one query run with
  `work_mem = 64kB` (default `4MB`, set by session `SET` so that it would
  spill) created 2 temporary files. `temp_files` rose by 2 and `temp_bytes` by
  5,634,432, the exact sum of the two sizes logged under `log_temp_files = 0`
  (default `-1`).
  [monitoring.sgml#pg_stat_database-counters](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2524-L2628)
  [fd.c#ReportTemporaryFileUsage](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1272-L1287)
  [pgstatfuncs.c#checksum-null](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1536-L1537)
  [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L145-L161)
- [ ] On a [hot standby](../../../glossary.md#hot-standby), capture
  recovery-conflict deltas by reason. The view counts cancellations that WAL
  replay needed after tablespace drops, conflicting locks, obsolete snapshots,
  buffer pins, or recovery deadlocks. Those are exactly the five
  `PROCSIG_RECOVERY_CONFLICT_*` reasons that the collector counts; a sixth, a
  dropped database, is deliberately not counted. These are not application
  `lock_timeout` events or the primary's `deadlocks`. The view holds data only
  on standbys, and every standby start resets it.
  [monitoring.sgml#pg_stat_database_conflicts](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2640-L2702)
  [procsignal.h#ProcSignalReason](../../../../raw/postgres-12/src/include/storage/procsignal.h#L37-L43)
  [pgstat.c#pgstat_recv_recoveryconflict](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6330-L6362)
  [standby.c#ResolveRecoveryConflictWithLock](../../../../raw/postgres-12/src/backend/storage/ipc/standby.c#L388-L423)

**4. Vacuum, analyze, dead rows, and progress**

```sql
SELECT /* wiki_db_health_vacuum_candidates */
       schemaname,
       relname,
       n_live_tup,
       n_dead_tup,
       last_vacuum,
       last_autovacuum,
       vacuum_count,
       autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC, schemaname, relname
LIMIT 50;

SELECT /* wiki_db_health_analyze_candidates */
       schemaname,
       relname,
       n_live_tup,
       n_mod_since_analyze,
       last_analyze,
       last_autoanalyze,
       analyze_count,
       autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_mod_since_analyze DESC, schemaname, relname
LIMIT 50;

SELECT /* wiki_db_health_catalog_maintenance */
       relname,
       n_live_tup,
       n_dead_tup,
       last_vacuum,
       last_autovacuum,
       autovacuum_count
FROM pg_stat_sys_tables
WHERE schemaname = 'pg_catalog'
ORDER BY n_dead_tup DESC, relname
LIMIT 20;

SELECT /* wiki_db_health_partitioned_parents */
       n.nspname AS schemaname,
       c.relname,
       c.oid::regclass AS partitioned_parent
FROM pg_class AS c
JOIN pg_namespace AS n ON n.oid = c.relnamespace
WHERE c.relkind = 'p'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
  AND n.nspname !~ '^pg_toast'
ORDER BY n.nspname, c.relname;

SELECT /* wiki_db_health_toast_maintenance */
       u.relid::regclass AS owning_relation,
       ts.relid::regclass AS toast_relation,
       ts.n_live_tup,
       ts.n_dead_tup,
       ts.last_vacuum,
       ts.last_autovacuum,
       ts.vacuum_count,
       ts.autovacuum_count
FROM pg_stat_user_tables AS u
JOIN pg_class AS c ON c.oid = u.relid
JOIN pg_stat_sys_tables AS ts ON ts.relid = c.reltoastrelid
ORDER BY ts.n_dead_tup DESC, owning_relation
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
WHERE datname = current_database()
ORDER BY pid;
```

Rank vacuum and analyze pressure independently. PostgreSQL computes the two
thresholds independently, each as a base threshold plus a scale factor times
`pg_class.reltuples`, and a table's [storage parameters](../../../glossary.md#storage-parameter)
can override the setting defaults. In PostgreSQL 12 the vacuum decision counts
only dead rows, so an insert-only table is vacuumed only for wraparound. The
row counts are discovery signals, not proof that maintenance is overdue.
[autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2920-L2956)
[autovacuum.c#effective-thresholds](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2995-L3067)
[autovacuum.c#threshold-decision](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060-L3080)

`pg_stat_user_tables` exposes estimated live and [dead rows](../../../glossary.md#dead-tuple),
changes since analyze, and maintenance timestamps and counts. It omits
partitioned parents, and it filters out `pg_catalog` and the `pg_toast`
schemas, which is why the catalog and [TOAST](../../../glossary.md#toast)
queries read `pg_stat_sys_tables`. PostgreSQL 12 autovacuum examines ordinary
tables, materialized views, and their TOAST tables, but it does not select
partitioned parents. `ANALYZE` on a partitioned parent recurses through the
partitions and updates parent statistics, so deployments that plan queries on
[partitioned](../../../glossary.md#declarative-partitioning) parents need an
explicit parent `ANALYZE` policy.
[system_views.sql#table-statistics-views](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L616)
[autovacuum.c#first-pass-relkind-filter](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2035-L2067)
[autovacuum.c#second-pass-TOAST-scan](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2140-L2191)
[analyze.c#analyze_rel](../../../../raw/postgres-12/src/backend/commands/analyze.c#L231-L268)
[ref/analyze.sgml#partitioned-tables](../../../../raw/postgres-12/doc/src/sgml/ref/analyze.sgml#L112-L121)

`pg_stat_progress_vacuum` has one row per running `VACUUM` or autovacuum
worker. `VACUUM FULL` reports through `pg_stat_progress_cluster` instead. The
view includes workers in every database, so filtering to `current_database()`
makes the `regclass` cast resolve in the right database.
[monitoring.sgml#pg-stat-progress-vacuum-view](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L3738-L3849)
[pgstatfuncs.c#progress-backend-loop](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L490-L535)
[regproc.c#regclassout](../../../../raw/postgres-12/src/backend/utils/adt/regproc.c#L969-L1023)

Checklist:

- [ ] Investigate tables whose dead-row or modification counts grow while
  maintenance timestamps do not advance. Autovacuum compares dead rows and
  changes since analyze with base-plus-scale-factor thresholds. It skips a
  table whose `autovacuum_enabled` is false unless wraparound forces a vacuum,
  and it skips ordinary threshold work for a table with no statistics entry.
  [autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2920-L2956)
  [autovacuum.c#threshold-decision](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3026-L3091)
- [ ] Confirm `autovacuum` and `track_counts` are on before treating missing
  automatic maintenance as a table problem. `autovacuum` is a reload setting,
  and even when it is off the server still launches workers to prevent
  wraparound.
  [config.sgml#autovacuum](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6985-L7005)
  [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1425-L1433)
- [ ] Read the autovacuum completion summary, not just its timestamp. Its
  `tuples:` line reports rows removed and rows `dead but not yet removable`
  with the `oldest xmin` used; its `pages:` line reports pages skipped because
  of [buffer pins](../../../glossary.md#buffer-pin). A run that removes nothing
  while unremovable rows grow means an old horizon, not a slow vacuum. This
  needs a nonnegative `log_autovacuum_min_duration`. Measured on this pin,
  with `log_autovacuum_min_duration = 0` and a test table whose
  `autovacuum_vacuum_threshold` and `autovacuum_vacuum_scale_factor` were 0
  (defaults `-1`, meaning the settings' 50 rows plus 20%): while a
  `REPEATABLE READ` session held a snapshot, autovacuum reported
  `0 removed, 2000 remain, 1000 are dead but not yet removable`; after that
  session ended, the next run reported `1000 removed`.
  [vacuumlazy.c#autovacuum-summary](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L401-L441)
- [ ] Look for repeated `canceling autovacuum task`. A session that waits on
  a lock held by a non-wraparound autovacuum worker cancels that worker after
  `deadlock_timeout`, so frequent conflicting DDL or `LOCK TABLE` can keep a
  table from ever finishing a vacuum. Anti-wraparound workers are exempt.
  [proc.c#autovacuum-cancel](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375)
  [postgres.c#canceling-autovacuum-task](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3096-L3101)
- [ ] Review partitioned parents separately and schedule recursive `ANALYZE`
  where queries depend on parent statistics. Review TOAST and catalog
  maintenance with the two `pg_stat_sys_tables` queries.

**5. Transaction-ID and MultiXact age**

```sql
SELECT /* wiki_db_health_database_xid_mxid_age */
       datname,
       age(datfrozenxid) AS xid_age,
       mxid_age(datminmxid) AS mxid_age
FROM pg_database
ORDER BY datname;

SELECT /* wiki_db_health_relation_xid_age */
       c.oid::regclass AS relation,
       c.relpersistence,
       greatest(age(c.relfrozenxid), age(t.relfrozenxid)) AS xid_age,
       c.reloptions AS relation_options,
       t.reloptions AS toast_options
FROM pg_class AS c
LEFT JOIN pg_class AS t ON t.oid = c.reltoastrelid
WHERE c.relkind IN ('r', 'm')
ORDER BY xid_age DESC NULLS LAST
LIMIT 50;

SELECT /* wiki_db_health_relation_mxid_age */
       c.oid::regclass AS relation,
       c.relpersistence,
       greatest(mxid_age(c.relminmxid), mxid_age(t.relminmxid)) AS mxid_age,
       c.reloptions AS relation_options,
       t.reloptions AS toast_options
FROM pg_class AS c
LEFT JOIN pg_class AS t ON t.oid = c.reltoastrelid
WHERE c.relkind IN ('r', 'm')
ORDER BY mxid_age DESC NULLS LAST
LIMIT 50;
```

PostgreSQL stores table and database freeze cutoffs in `relfrozenxid` and
`datfrozenxid`, and `age()` counts [transactions](../../../glossary.md#transaction-id)
from that cutoff. `datminmxid` is the database-level
[MultiXact](../../../glossary.md#multixact) horizon and `relminmxid` the
relation horizon. Run the database query once and the two relation rankings in
each database. Separate rankings keep one top-50 limit from hiding the other
risk.
[catalogs.sgml#pg_database-horizons](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L2662-L2687)
[maintenance.sgml#xid-age-queries](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L557-L585)
[maintenance.sgml#multixact-age](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L655-L686)

Checklist:

- [ ] Treat `autovacuum_freeze_max_age` and
  `autovacuum_multixact_freeze_max_age` as early alert ceilings, not exact
  remaining capacity. Per-relation options can lower both, and MultiXact
  member-space pressure can lower the effective MultiXact threshold further,
  down to zero. The two cluster settings need a restart.
  [autovacuum.c#effective-freeze-ages](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3018-L3041)
  [multixact.c#MultiXactMemberFreezeThreshold](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L2785-L2845)
  [config.sgml#autovacuum-freeze-max-age](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7156-L7179)
  [config.sgml#autovacuum-multixact-freeze-max-age](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7184-L7208)
- [ ] Escalate XID and MultiXact warnings independently. PostgreSQL has
  separate warning and stop-limit paths for transaction IDs, MultiXact IDs,
  and MultiXact member space; see [Wraparound](../../../glossary.md#wraparound).
  [maintenance.sgml#wraparound-warning-shutdown](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L608-L635)
  [multixact.c#MultiXact-emergency-limits](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L920-L1140)
- [ ] Treat an old temporary table (`relpersistence = 't'`) at the top of the
  ranking as a session problem. Autovacuum cannot process another session's
  temporary tables, so only that session can vacuum it. Autovacuum drops a
  temporary table whose session is gone and logs
  `autovacuum: dropping orphan temp table`.
  [autovacuum.c#temp-table-skip](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2071-L2093)
  [autovacuum.c#orphan-temp-drop](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2257-L2262)

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

- [ ] Compare two samples that share `stats_reset`. `checkpoints_req` counts
  every requested [checkpoint](../../../glossary.md#checkpoint), whatever
  requested it, so it does not say why. With `log_checkpoints` on, each
  `checkpoint starting:` line lists its cause flags, and ` wal` marks a
  checkpoint requested because WAL volume reached `max_wal_size`.
  [monitoring.sgml#pg_stat_bgwriter](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2399-L2484)
  [xlog.c#LogCheckpointStart](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8356-L8372)
  [xlog.c#WAL-caused-request](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L2553-L2557)
- [ ] Investigate repeated `checkpoints are occurring too frequently` lines.
  The [checkpointer](../../../glossary.md#checkpointer) writes this at LOG
  severity, not WARNING, with the hint to increase `max_wal_size`. It does so
  only for a WAL-caused checkpoint that starts within `checkpoint_warning` of
  the previous one, and never for a standby restartpoint. Without
  `log_checkpoints`, it is the only log evidence of the cause.
  `checkpoint_warning` and `max_wal_size` are reload settings. Measured on
  this pin, with `max_wal_size = 32MB` (the `initdb` file value is `1GB`) and
  `log_checkpoints = on` (default `off`), both in the script's `dh12.conf`:
  within the stage's capture window, a
  150,000-row insert produced three `checkpoint starting: wal` lines, each
  preceded by a LOG frequency line (`0 seconds apart` twice, `1 second apart`
  once) and its hint. The standby logged two `restartpoint starting: wal`
  lines and no frequency line.
  [checkpointer.c#checkpoint-warning](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462)
  [xlog.c#restartpoint-request](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L11564-L11572)
  [guc.c#checkpoint-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2554-L2589)
- [ ] A `maxwritten_clean` delta means the
  [background writer](../../../glossary.md#background-writer) stopped a pass
  after reaching `bgwriter_lru_maxpages`. `buffers_backend` counts buffers
  that backends wrote themselves; a nonzero value alone is not a failure, so
  compare its delta with workload and the other write counters.
  [bufmgr.c#BgBufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2260-L2300)
- [ ] Treat `buffers_backend_fsync` as the stronger signal. PostgreSQL
  increments it when a backend must perform its own [fsync](../../../glossary.md#fsync)
  because no checkpointer is running or the fsync-request queue stays full
  after compaction.
  [checkpointer.c#ForwardSyncRequest](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L1086-L1159)
- [ ] Enable `log_checkpoints` when per-checkpoint buffer counts and write and
  sync durations are needed, rather than only cumulative counters.
  [config.sgml#log_checkpoints](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6165-L6178)
  [xlog.c#LogCheckpointEnd](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8374-L8454)
- [ ] If measured deltas justify tuning `bgwriter_lru_maxpages`,
  `bgwriter_lru_multiplier`, or `bgwriter_delay`, all three reload.
  [guc.c#bgwriter-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2727-L2755)
  [guc.c#bgwriter_lru_multiplier](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3351-L3359)

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

SELECT /* wiki_db_health_archive_backlog */
       count(*) FILTER (WHERE name LIKE '%.ready') AS ready_files,
       min(modification) FILTER (WHERE name LIKE '%.ready') AS oldest_ready_time
FROM pg_ls_archive_statusdir();

SELECT /* wiki_db_health_wal_directory */
       count(*) AS wal_directory_entries,
       coalesce(sum(size), 0) AS wal_directory_bytes,
       min(modification) AS oldest_entry_time,
       max(modification) AS newest_entry_time
FROM pg_ls_waldir();

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
       CASE WHEN NOT pg_is_in_recovery()
            THEN pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)
       END AS source_to_sent_bytes,
       pg_wal_lsn_diff(sent_lsn, write_lsn) AS sent_to_write_bytes,
       pg_wal_lsn_diff(write_lsn, flush_lsn) AS write_to_flush_bytes,
       pg_wal_lsn_diff(flush_lsn, replay_lsn) AS flush_to_replay_bytes,
       write_lag,
       flush_lag,
       replay_lag,
       sync_state,
       reply_time
FROM pg_stat_replication
ORDER BY application_name, pid;

SELECT /* wiki_db_health_replication_slots */
       slot_name,
       plugin,
       slot_type,
       database,
       temporary,
       active,
       active_pid,
       xmin,
       catalog_xmin,
       restart_lsn,
       confirmed_flush_lsn,
       CASE WHEN NOT pg_is_in_recovery()
            THEN pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       END AS current_to_restart_bytes,
       age(xmin) AS xmin_age,
       age(catalog_xmin) AS catalog_xmin_age
FROM pg_replication_slots
ORDER BY current_to_restart_bytes DESC NULLS LAST, slot_name;

SELECT /* wiki_db_health_wal_receiver */
       pid,
       status,
       receive_start_lsn,
       received_lsn,
       pg_last_wal_replay_lsn() AS replay_lsn,
       pg_wal_lsn_diff(received_lsn, pg_last_wal_replay_lsn())
         AS receive_replay_gap_bytes,
       last_msg_send_time,
       last_msg_receipt_time,
       now() - last_msg_receipt_time AS time_since_last_message,
       latest_end_lsn,
       latest_end_time,
       slot_name,
       sender_host,
       sender_port
FROM pg_stat_wal_receiver;

SELECT /* wiki_db_health_standby_replay */
       pg_is_in_recovery() AS in_recovery,
       pg_last_wal_receive_lsn() AS received_lsn,
       pg_last_wal_replay_lsn() AS replayed_lsn,
       pg_wal_lsn_diff(pg_last_wal_receive_lsn(),
                       pg_last_wal_replay_lsn()) AS receive_replay_bytes,
       pg_last_xact_replay_timestamp() AS last_replayed_transaction_time;

SELECT /* wiki_db_health_subscription_workers */
       st.subid,
       st.subname,
       st.pid,
       st.relid,
       CASE
         WHEN st.pid IS NULL THEN 'worker not running'
         WHEN st.relid IS NULL THEN 'main apply worker'
         ELSE 'table synchronization worker'
       END AS worker_state,
       st.received_lsn,
       st.last_msg_send_time,
       st.last_msg_receipt_time,
       now() - st.last_msg_receipt_time AS time_since_last_message,
       st.latest_end_lsn,
       st.latest_end_time
FROM pg_stat_subscription AS st
ORDER BY st.subname, st.relid NULLS FIRST;

SELECT /* wiki_db_health_subscription_flags */
       d.datname,
       sub.subname,
       sub.subenabled,
       sub.subslotname
FROM pg_subscription AS sub
LEFT JOIN pg_database AS d ON d.oid = sub.subdbid
ORDER BY d.datname, sub.subname;
```

Checklist:

- [ ] Treat `pg_stat_archiver` as command-exit-status telemetry, not proof that
  an archive object exists, is durable, or is restorable. PostgreSQL counts an
  exit status of zero as success, so a command that returns zero without
  copying the file creates a false success. A command killed by a signal, or
  one that exits above 125, makes the archiver exit with FATAL before it
  reports the failed attempt.
  [config.sgml#archive-command-success](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L3110-L3133)
  [pgarch.c#archive-statistics](../../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L466-L545)
  [pgarch.c#archive-command-exits](../../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L623-L680)
- [ ] Interpret `last_archived_time` against WAL activity, the recovery-point
  objective, and `archive_timeout`. An old timestamp is expected on an idle
  system when no segment has completed and `archive_timeout` is at its
  default of 0. Use the archive-status and WAL-directory aggregates, which need
  `pg_monitor` or a superuser, plus host free-space monitoring to measure a
  persistent `.ready` backlog and growth.
  [config.sgml#archive_timeout](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L3138-L3168)
  [guc.c#archive_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1964-L1974)
  [func.sgml#pg_ls_waldir](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L21770-L21876)
  [system_views.sql#monitor-file-functions](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1320-L1347)
  [xlogarchive.c#XLogArchiveNotify](../../../../raw/postgres-12/src/backend/access/transam/xlogarchive.c#L500-L539)
- [ ] On a primary or publisher, investigate byte gaps from the local current
  WAL position ([LSN](../../../glossary.md#lsn)) through the sent, written,
  flushed, and replayed positions. On a cascading standby,
  `source_to_sent_bytes` is deliberately NULL: the sender's safe ceiling
  combines replay and receiver positions only when their
  [timelines](../../../glossary.md#timeline) match, which these SQL functions
  cannot reproduce, and `pg_current_wal_lsn()` raises an error during recovery.
  The other stage gaps still apply. The lag intervals describe recent
  [synchronous-commit](../../../glossary.md#synchronous-replication) stages,
  not predicted catch-up time; they persist briefly after catch-up, then become
  NULL, and a logical output plugin can omit them.
  [monitoring.sgml#pg_stat_replication](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1877-L2020)
  [walsender.c#GetStandbyFlushRecPtr](../../../../raw/postgres-12/src/backend/replication/walsender.c#L2924-L2957)
  [xlogfuncs.c#pg_current_wal_lsn](../../../../raw/postgres-12/src/backend/access/transam/xlogfuncs.c#L340-L361)
- [ ] Inspect every slot, not only inactive ones. PostgreSQL applies WAL and
  XID retention to every in-use slot without consulting `active_pid`, so
  `active` says whether a process owns the slot, not whether the consumer is
  healthy. An unexpected NULL `restart_lsn` means the slot never reserved WAL.
  PostgreSQL 12 has no way to bound the WAL a slot retains. The slot query's
  byte distance is a logical LSN distance on a non-recovery node, not the bytes
  retained in `pg_wal`, whose files are kept in whole segments; use the
  WAL-directory aggregate for those.
  [slot.c#ComputeRequiredXmin-and-LSN](../../../../raw/postgres-12/src/backend/replication/slot.c#L695-L776)
  [catalogs.sgml#pg_replication_slots-columns](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9914-L9968)
  [high-availability.sgml#replication-slot-retention](../../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L914-L930)
- [ ] Without a slot, nothing keeps WAL for a lagging standby. Segment removal
  keeps only what `wal_keep_segments` (default 0) and slots require, so a
  standby that falls one segment behind a busy primary can be cut off with
  `requested WAL segment ... has already been removed`.
  [xlog.c#KeepLogSeg](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9307-L9349)
  [guc.c#wal_keep_segments](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2532-L2540)
- [ ] On a physical standby, an empty `pg_stat_wal_receiver` means no
  [WAL receiver](../../../glossary.md#wal-receiver) is running. Inspect its
  status, stale message receipt times, and the receive-to-replay byte gap.
  Compare each standby separately, because a primary reports only directly
  connected consumers.
  [system_views.sql#pg_stat_wal_receiver](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L784-L801)
  [monitoring.sgml#pg_stat_wal_receiver](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2024-L2132)
  [monitoring.sgml#direct-replication-connections](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1978-L1983)
- [ ] On a [logical replication](../../../glossary.md#logical-replication)
  subscriber, expect one main row per [subscription](../../../glossary.md#subscription)
  in the workers query. A NULL `pid` means its main worker is not running.
  Match it to the flags query by `subname`, which is unique within a database:
  `subenabled = false` explains an intentionally stopped worker, while
  `subenabled = true` with a NULL main `pid` needs investigation. Rows with a
  non-NULL `relid` are [table synchronization](../../../glossary.md#table-synchronization)
  workers; investigate stale receipt times or copies that do not progress. Both
  queries use only columns PUBLIC can read, so `pg_monitor` can run them. If two
  databases reuse a subscription name, only a superuser can attribute workers,
  because `pg_stat_subscription.subid` equals the unreadable
  `pg_subscription.oid`.
  [system_views.sql#pg_stat_subscription](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L803-L816)
  [system_views.sql#pg_subscription-column-grants](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1059-L1062)
  [indexing.h#pg_subscription_subname_index](../../../../raw/postgres-12/src/include/catalog/indexing.h#L363)
  [monitoring.sgml#pg_stat_subscription](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2134-L2205)
  [catalogs.sgml#pg_subscription](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L6699-L6797)
- [ ] Investigate an old `backend_xmin` on a sender. It is the horizon that a
  standby reports through `hot_standby_feedback`, and it delays vacuum on the
  primary.
  [monitoring.sgml#pg_stat_replication-backend_xmin](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1835-L1840)
  [config.sgml#hot_standby_feedback](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4139-L4161)
- [ ] With synchronous replication, correlate commit latency and lock
  contention with `sync_state` and lag. Transactions keep their locks until the
  required transfer is confirmed.
  [high-availability.sgml#synchronous-replication-impact](../../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L1238-L1244)

**8. Optional query and relation diagnostics**

```sql
SELECT /* wiki_db_health_pgss_capture */
       dbid,
       userid,
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
ORDER BY dbid, userid, queryid;

SELECT /* wiki_db_health_pgstattuple */
       *
FROM pgstattuple_approx('schema.table_name'::regclass);

SELECT /* wiki_db_health_pgstatindex */
       *
FROM pgstatindex('schema.index_name'::regclass);
```

Checklist:

- [ ] Take two complete, un-`LIMIT`ed captures of `pg_stat_statements` within
  the same epoch. They are not atomic workload snapshots: the view walks the
  shared hash and copies each entry under that entry's spinlock, so active
  rows are sampled at slightly different instants. Compare captures with
  full-outer-join semantics on `dbid`, `userid`, and `queryid`, and use
  nonnegative deltas for rows present in both. Treat a second-only row's
  counters as a lower bound. A first-only row or a negative delta means
  eviction, reset, or entry recreation broke that interval. Then rank the
  deltas separately by `total_time`, `shared_blks_read`, and
  `temp_blks_written`. Ranking cumulative rows before subtraction can hide a
  statement that dominated only the interval. I/O timing needs
  `track_io_timing`.
  [pgstatstatements.sgml#view](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L22-L41)
  [pgstatstatements.sgml#query-identity](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L274-L291)
  [pg_stat_statements.c#view-iteration](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1500-L1615)
  [pgstatstatements.sgml#entry-eviction](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L399-L410)
  [pgstatstatements.sgml#reset](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L333-L357)
  [config.sgml#track_io_timing](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6867-L6871)
- [ ] Do not expect comment tags to separate entries. Statements that differ
  only in constants share one entry, and the stored text is that of the first
  statement seen; comments are not part of the identity.
  [pgstatstatements.sgml#query-merging](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L236-L255)
- [ ] Verify `pg_stat_statements.track`, `pg_stat_statements.track_utility`,
  `pg_stat_statements.max`, and `pg_stat_statements.save` before interpreting
  sparse results. Eviction, reset, or a backend crash can break a comparison
  interval; see [When counters reset](#when-counters-reset).
  [pgstatstatements.sgml#access](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L228-L234)
  [pgstatstatements.sgml#settings](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L399-L474)
- [ ] Use `pgstattuple_approx` to shortlist heap relations with large
  dead-row or free-space fractions. It skips pages marked all-visible in the
  [visibility map](../../../glossary.md#visibility-map) but can still scan
  every other page; use exact `pgstattuple` only when that extra scan is
  justified.
  [pgstattuple.sgml#pgstattuple](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L39-L58)
  [pgstattuple.sgml#pgstattuple_approx](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L484-L538)
- [ ] Use `pgstatindex` for a selected B-tree index when index size, deleted
  or empty pages, average leaf density, and leaf fragmentation are needed. It
  reads every block under a share lock while the index can change, and it
  rejects non-B-tree indexes.
  [pgstattuple.sgml#pgstatindex](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L161-L186)
  [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L315)

### Database log checklist

The table assumes the log destination is already accessible. The "possible
root causes" column holds operational inferences constrained by the cited
mechanism; confirm them with current activity, counters, and operating-system
evidence. Messages come from [`ereport`](../../../glossary.md#ereport) calls,
and the severity shown is the one the source uses.

| Log pattern or event | Repercussion | Possible root causes and next checks | Required data/configuration |
|---|---|---|---|
| `sorry, too many clients already`, `remaining connection slots are reserved...`, or `number of requested standby connections exceeds max_wal_senders` | New clients are refused. Ordinary users are refused before the reserved superuser slots; WAL senders have their own pool. [proc.c#InitProcess](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L347-L363) [postinit.c#reserved-connections](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L818-L828) | Compare only `client backend` rows with `max_connections`; inspect pool limits, abandoned sessions, and bursts. The capacity settings need a restart. [guc.c#max_connections-reserved](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2129-L2148) | The FATAL messages are built in. Enable `log_connections` and `log_disconnections` before new sessions start for attempt and duration context. [config.sgml#log_connections-disconnections](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6182-L6222) |
| `no pg_hba.conf entry`, an authentication failure, or `could not accept SSL connection` | The client cannot establish the session. Repetition can mean a deployment error, a stale credential, an incompatible TLS setup, or unwanted access attempts. [auth.c#auth_failed](../../../../raw/postgres-12/src/backend/libpq/auth.c#L255-L323) [auth.c#no-pg_hba.conf-entry](../../../../raw/postgres-12/src/backend/libpq/auth.c#L494-L528) [be-secure-openssl.c#SSL-errors](../../../../raw/postgres-12/src/backend/libpq/be-secure-openssl.c#L440-L459) | Correlate client address, database, user, and TLS state with the expected inventory; review HBA order and credential or certificate rotation. | Authentication errors are built in. `log_connections` adds attempt context. [config.sgml#log_connections](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6182-L6203) |
| `duration: ... ms  statement: ...` from the simple protocol, or `duration: ... ms  parse ...`, `bind ...`, or `execute ...` from the extended protocol | Latency, resource use, and lock holding rise while slow statements run. Clients that use the extended protocol have Parse, Bind, and Execute timed separately, so a search for `statement:` alone misses their statements. [config.sgml#log_min_duration_statement](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5931-L5953) [postgres.c#exec_simple_query-duration](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L1281-L1298) [postgres.c#parse-duration](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L1547-L1560) [postgres.c#bind-duration](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L1914-L1930) [postgres.c#execute-duration](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L2135-L2152) | Correlate with `pg_stat_statements`, lock waits, I/O timing, and temp-file lines. Measured on this pin, with `log_min_duration_statement = 250` ms in the script's `dh12.conf`: a 0.5 s sleep logged `statement:` from `psql`, `execute <unnamed>:` from `pgbench -M extended`, and `execute P0_2:` from `pgbench -M prepared`, whose statement names come from `P<script>_<command>`. [pgbench.c#prepared-statement-name](../../../../raw/postgres-12/src/bin/pgbench/pgbench.c#L2599) | Set `log_min_duration_statement` (default `-1`): `0` logs every completed statement. [guc.c#log_min_duration_statement](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2703-L2713) |
| `temporary file: path ..., size ...` or `temporary file size exceeds temp_file_limit` | Temp I/O consumes storage, and a process that exceeds `temp_file_limit` fails its statement. [fd.c#ReportTemporaryFileUsage](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1272-L1287) [fd.c#temp_file_limit](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1942-L1956) | Sorts, hashes, and materialized intermediate results create temporary files. Correlate the PID with the statement and compare `temp_bytes` and `temp_blks_written`. [config.sgml#log_temp_files](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6555-L6574) | Set `log_temp_files` (default `-1`) to `0` to log every file. Both settings are `PGC_SUSET`. [guc.c#log_temp_files](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3153-L3162) [guc.c#temp_file_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2270-L2279) |
| `process ... still waiting for ...`, `acquired ... after ...`, `detected deadlock while waiting`, `avoided deadlock ... by rearranging queue order`, or `deadlock detected` | A statement waited for a lock; a hard deadlock aborts one transaction. The optional wait LOG lines and the built-in deadlock ERROR are separate signals. A holder PID of 0 in `Process holding the lock` is a prepared transaction. [proc.c#lock-wait-log](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1377-L1497) [deadlock.c#DeadLockReport](../../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L1139-L1146) | Use `pg_blocking_pids()` and the holder and waiter PIDs. Check inconsistent lock order and prepared transactions that keep locks. [mvcc.sgml#Deadlocks](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1358-L1423) | Enable `log_lock_waits` (default `off`); it logs waits longer than `deadlock_timeout`. Both are `PGC_SUSET`. [config.sgml#log_lock_waits](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6477-L6490) [guc.c#deadlock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2062-L2072) |
| `canceling statement due to lock timeout`, `canceling statement due to statement timeout`, or `terminating connection due to idle-in-transaction timeout` | PostgreSQL cancels the statement or ends the idle transaction's session. Intentional guardrails produce these, so escalate repeated, unexpected, or service-affecting events. [postgres.c#ProcessInterrupts-timeouts](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3058-L3136) | Compare the configured limit with expected work and inspect blockers or transaction handling. | [`statement_timeout`, `lock_timeout`](../../../glossary.md#statement_timeout-and-lock_timeout), and [`idle_in_transaction_session_timeout`](../../../glossary.md#transaction_timeout-and-idle_in_transaction_session_timeout) are `PGC_USERSET` and default to 0, which disables them. [guc.c#client-timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2408) |
| ERROR `canceling statement due to conflict with recovery` or FATAL `terminating connection due to conflict with recovery` | Replay on a standby cancels a running query, or ends a session that sits idle in a transaction, so replay can continue. The idle case is FATAL because no statement is running to cancel. [postgres.c#RecoveryConflictInterrupt](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L2866-L2915) [postgres.c#idle-session-conflict](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3030-L3041) [postgres.c#running-query-conflict](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3103-L3112) | Use `pg_stat_database_conflicts` to separate tablespace, lock, snapshot, buffer-pin, and recovery-deadlock causes; long standby queries and cleanup on the primary conflict. Measured on this pin: one running and one idle `REPEATABLE READ` session were cancelled 30 s after a `VACUUM` on the primary, with one ERROR, one FATAL, and `confl_snapshot` moving from 0 to 2. [monitoring.sgml#pg_stat_database_conflicts](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2640-L2702) | Built in on a standby. Replay waits `max_standby_streaming_delay` (default 30 s) after WAL receipt before cancelling; both delays reload. [standby.c#GetStandbyLimitTime](../../../../raw/postgres-12/src/backend/storage/ipc/standby.c#L154-L169) [guc.c#standby-conflict-delays](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2074-L2094) |
| Successful `automatic vacuum of table ...`, `automatic aggressive vacuum of table ...`, or `automatic aggressive vacuum to prevent wraparound of table ...` | The vacuum finished, but it may have removed little. The summary reports duration and system usage, index scans, pages (including pages skipped because of pins), rows removed, rows `dead but not yet removable` with the `oldest xmin`, buffer hits, misses, and dirties, and read and write rates. It has no WAL-usage field in PostgreSQL 12. [vacuumlazy.c#autovacuum-message-prefixes](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L401-L413) [vacuumlazy.c#autovacuum-summary](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L380-L441) | Growing unremovable rows point to an old horizon: long transactions, prepared transactions, replication slots, or standby feedback. Compare with prior runs and current table size. | Set `log_autovacuum_min_duration` (default `-1`) to a nonnegative value and reload; `0` logs every action. [config.sgml#log_autovacuum_min_duration](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7010-L7033) [guc.c#log_autovacuum_min_duration](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2715-L2725) |
| Successful `automatic analyze of table ...` | Analyze finished. In this pin its message carries system usage only, without vacuum's page, row, or buffer fields. [analyze.c#autoanalyze-summary](../../../../raw/postgres-12/src/backend/commands/analyze.c#L667-L679) | Compare frequency with the table's change rate and the freshness of planner statistics. | The same `log_autovacuum_min_duration` threshold. |
| ERROR `canceling autovacuum task` followed by CONTEXT `automatic vacuum of table ...` | A conflicting lock request cancelled a non-wraparound autovacuum worker after `deadlock_timeout`. Repeated cancellations can stop a table from ever finishing a vacuum. [proc.c#autovacuum-cancel](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375) [postgres.c#canceling-autovacuum-task](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3096-L3101) [autovacuum.c#autovacuum-error-context](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2480-L2493) | Find the DDL, `LOCK TABLE`, or other strong lock that conflicts with `SHARE UPDATE EXCLUSIVE`. The waiter's own note is DEBUG1, so only the worker's ERROR appears by default. Measured on this pin: a `LOCK TABLE ... IN SHARE MODE` against a running autovacuum logged this ERROR and CONTEXT, and the waiter acquired its lock after about 1000 ms, the default `deadlock_timeout`. The worker was kept busy by the test table's storage parameters `autovacuum_vacuum_cost_delay = 20` and `autovacuum_vacuum_cost_limit = 1` (defaults `-1`, meaning 2 ms and 200); see the [measurement script](#measurement-script). [proc.c#autovacuum-cancel-debug](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1344-L1347) | Built in; no setting is needed. `log_lock_waits` adds the waiter's line. |
| `skipping vacuum ... lock not available`, `skipping analyze ... lock not available`, `oldest xmin is far in the past`, or `oldest multixact is far in the past` | Maintenance stays incomplete, and an old horizon delays cleanup or freezing. [vacuum.c#automatic-vacuum-and-analyze-skips](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L558-L649) [vacuum.c#old-horizon-warnings](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L933-L988) | Check relation options, blockers, old sessions, prepared transactions, slots, and row-lock-heavy activity. | Autovacuum's skip lines need a nonnegative `log_autovacuum_min_duration`; the WARNING lines are built in. [vacuum.c#skip-log-level](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L609-L614) |
| `database ... must be vacuumed within ... transactions` or `database is not accepting commands to avoid wraparound data loss` | An emergency. Ignored warnings lead to refusal of new commands, while uncontrolled wraparound would make old rows look like they are in the future. `GetNewTransactionId()` raises the WARNING at the warn limit and the ERROR at the stop limit, both naming the oldest database. [varsup.c#GetNewTransactionId-wraparound-limits](../../../../raw/postgres-12/src/backend/access/transam/varsup.c#L118-L158) [maintenance.sgml#wraparound-impact](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L391-L405) [maintenance.sgml#wraparound-warning-shutdown](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L608-L635) | Autovacuum has not advanced the oldest XIDs. Check blocked or failed anti-wraparound vacuums, old transactions, prepared transactions, stale slots, and old temporary tables. A manual fix must run as a superuser, because a non-superuser VACUUM skips catalogs. | Built in. Make sure the log keeps WARNING and ERROR. |
| `must be vacuumed before ... more MultiXactId(s) ...`, `must be vacuumed before ... more multixact member(s) ...`, `database is not accepting commands that generate new MultiXactIds ...`, or `multixact "members" limit exceeded` | PostgreSQL can force maintenance, refuse commands, or fail row-locking work before the configured MultiXact age is reached. [multixact.c#MultiXact-emergency-limits](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L920-L1140) [multixact.c#member-space-warning](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L1129-L1140) | Inspect `datminmxid`, relation MultiXact ages, long-lived row lockers, prepared transactions, and the effective member-space threshold. [multixact.c#MultiXactMemberFreezeThreshold](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L2785-L2845) | Built in. `autovacuum_multixact_freeze_max_age` needs a restart, but member-space pressure lowers the effective threshold on its own. [guc.c#autovacuum_multixact_freeze_max_age](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2976-L2985) |
| `checkpoint starting: ...`, `checkpoint complete: ...`, `restartpoint starting: ...`, or LOG `checkpoints are occurring too frequently` | Frequent requested checkpoints raise write pressure, and long write or sync phases can line up with latency spikes. The start line names the cause; the frequency line is LOG severity, appears only for WAL-caused checkpoints within `checkpoint_warning`, and never for restartpoints. [xlog.c#LogCheckpointStart](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8356-L8372) [xlog.c#LogCheckpointEnd](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8374-L8454) [checkpointer.c#checkpoint-warning](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462) | Compare `checkpoints_req`, the cause flags, WAL generation, and `max_wal_size`. | Enable `log_checkpoints` (default `off`) and reload for start and end lines; the frequency line does not need it. `checkpoint_warning` and `max_wal_size` reload. [config.sgml#log_checkpoints](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6165-L6178) [guc.c#checkpoint-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2554-L2589) |
| `archive_mode enabled, yet archive_command is not set`, `archive command failed with exit code ...`, `archive command was terminated by signal ...`, or `archiving write-ahead log file ... failed too many times` | Required WAL is not archived; WAL accumulates in `pg_wal`, and the archive chain or recovery-point objective can break. [pgarch.c#archive-failures](../../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L466-L545) [pgarch.c#archive-command-exits](../../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L623-L680) | Check the command's exit status, destination capacity and permissions, and WAL generation against archive throughput. Verify the destination independently, because a zero exit status is not proof of an archived object. | `archive_mode` needs a restart; `archive_command` and `archive_timeout` reload. The command must return zero only after success. [config.sgml#archive-mode-command](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L3072-L3133) |
| `terminating walsender process due to replication timeout`, `terminating walreceiver due to timeout`, a lost WAL stream, or `requested WAL segment ... has already been removed` | Replication disconnects or cannot continue from the requested point. If the WAL is in neither `pg_wal` nor the archive, the replica or subscriber needs a new starting point or a rebuild. [walsender.c#WalSndCheckTimeOut](../../../../raw/postgres-12/src/backend/replication/walsender.c#L2125-L2149) [walreceiver.c#receiver-errors](../../../../raw/postgres-12/src/backend/replication/walreceiver.c#L420-L529) [xlog.c#requested-WAL-removed](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L3855-L3865) | Check network continuity, sender and receiver timeouts, slot retention, `wal_keep_segments`, archive availability, and whether the expected worker is running. | `wal_receiver_timeout` reloads; `wal_sender_timeout` is `PGC_USERSET`. The messages are built in. [guc.c#wal_receiver_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2118-L2127) [guc.c#wal_sender_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2656-L2665) |
| `server process (PID ...) was terminated by signal ...`, `terminating any other active server processes`, `all server processes terminated; reinitializing`, `database system was not properly shut down; automatic recovery in progress`, or a WAL write or fsync PANIC | A WAL I/O PANIC stops the server. A backend crash makes the postmaster terminate every other server process, reinitialize shared memory, and run [crash recovery](../../../glossary.md#crash-recovery), which also deletes all collected statistics. [xlog.c#WAL-write-failures](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L2494-L2510) [xlog.c#WAL-fsync-failures](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L10093-L10126) [postmaster.c#LogChildExit](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3678-L3686) [postmaster.c#backend-crash-reinitialization](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3378-L3407) [postmaster.c#reinitialize-after-crash](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3911-L3928) [xlog.c#not-properly-shut-down](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6764-L6766) | Treat this as an availability and durability incident. Keep the whole message chain; check storage, the filesystem, the kernel and out-of-memory killer, extension crashes, and recovery completion. After recovery, schedule `VACUUM` and `ANALYZE` on busy tables, because autovacuum no longer sees their pre-crash dead rows. Measured on this pin: a `SIGKILL` of one backend produced this whole chain, ending with `database system is ready to accept connections`. | Built in, with the default `restart_after_crash = on`. Make sure the log keeps LOG and PANIC lines; collect host and kernel evidence separately. [guc.c#restart_after_crash](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1277-L1285) |
| `could not extend file ... Check free disk space` or a short-write variant | The statement that needed the relation to grow fails. Continued lack of space can also block WAL, temp, or other writes. [md.c#mdextend](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L398-L418) | Check free space, inodes, quotas, storage errors, and whether retained WAL, temp files, or relation growth used the volume. | Built in. PostgreSQL does not expose filesystem free space in the views used here, so add operating-system monitoring. |
| `out of shared memory` with `You might need to increase max_locks_per_transaction` | The lock request fails, so the statement or transaction cannot continue. [lock.c#LockAcquireExtended](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L918-L930) | Check transactions that touch many relations or partitions, concurrent lock demand, and prepared transactions. The lock table is sized as `max_locks_per_transaction * (MaxBackends + max_prepared_transactions)` plus a 10% margin, which is not a hard per-transaction cap. [lock.c#NLOCKENTS](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L53-L57) [lock.c#lock-hash-margin](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L3437-L3460) [config.sgml#max_locks_per_transaction](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L8615-L8642) | `max_locks_per_transaction` and `max_prepared_transactions` need a restart; standbys need at least the primary's values. [guc.c#max_prepared_transactions](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2344-L2352) [guc.c#max_locks_per_transaction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2463-L2473) [config.sgml#standby-lock-capacity](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L8639-L8642) |
| WARNING `page verification failed, calculated checksum ... but expected ...`, ERROR `invalid page in block ... of relation ...`, WARNING `...; zeroing out page`, a base-backup checksum failure, or a nonzero checksum counter | PostgreSQL found a page it read to be damaged. The checksum WARNING needs data checksums, but the `invalid page` ERROR also fires without them when the page header is impossible. A `zeroing out page` WARNING means rows were dropped from that read. A zero counter does not verify unread pages. [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L161) [bufmgr.c#invalid-page-handling](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925) [basebackup.c#backup-checksum-failures](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1543-L1622) | Preserve evidence, identify the relation and block, investigate storage and I/O, and validate independent backups or replicas before any repair. Measured on this pin: an impossible header on a checksum-free cluster raised `invalid page in block 0 of relation base/...`, and `checksum_failures` stayed NULL. | Checksums must be enabled for the counters. Keep `ignore_checksum_failure` and `zero_damaged_pages` at `off`; see [Defaults that decide what you see](#defaults-that-decide-what-you-see). [guc.c#damaged-page-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1134-L1162) |
| `using stale statistics instead of current ones because stats collector is not responding` | Collected counters can be stale or missing, so conclusions based on them are unreliable. [pgstat.c#collector-not-responding](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L5741-L5765) | Check the collector process, the log chain, the local socket and filesystem, and whether a crash or restart is in progress. Re-capture after the collector resumes. | A built-in LOG line, kept by the default `log_min_messages`. |
| Connection or disconnection churn, or unexpected replication commands | Short sessions add connection overhead; replication commands should match the expected replication clients. `log_disconnections` includes session duration. [config.sgml#log-connections-disconnections](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6182-L6222) [config.sgml#log_replication_commands](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6539-L6551) | Check pool behavior, authentication retries, restart loops, and the replication inventory. Duplicate connection lines can be normal during password prompting. | Reload file changes to `log_connections` and `log_disconnections`; only sessions started afterwards pick them up. `log_replication_commands` is `PGC_SUSET`. [guc.c#PGC_SU_BACKEND-reload](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6811-L6858) |

### Issue interpretation and response

The root-cause directions are operational inferences from the mechanisms and
fields cited in each row.

| Priority | Finding | Repercussion | Root-cause direction |
|---|---|---|---|
| Immediate | XID or MultiXact stop warning, WAL write or fsync PANIC, checksum or `invalid page` error, a `zeroing out page` WARNING, disk-extension failure, uncontrolled `pg_wal` growth, or required WAL already removed | Commands, writes, or replication fail; WAL I/O can stop the server; damaged pages mean possible data loss. [varsup.c#GetNewTransactionId-wraparound-limits](../../../../raw/postgres-12/src/backend/access/transam/varsup.c#L118-L158) [multixact.c#MultiXact-emergency-limits](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L920-L1140) [xlog.c#WAL-fsync-failures](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L10093-L10126) [bufmgr.c#invalid-page-handling](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925) | Preserve evidence and administrative access, reduce nonessential load, find old transactions, prepared transactions, and slots, verify storage, and confirm that the needed WAL, archive, and backups still exist. |
| High | Connection exhaustion, sustained lock waits, deadlocks, repeated unexpected timeouts, stopped replication workers, synchronous-replication delay, or a crash-recovery chain in the log | Clients are refused, work waits or aborts, replication falls behind, synchronous commits hold locks longer, and after a crash every collected counter restarts. [proc.c#InitProcess](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L347-L363) [deadlock.c#DeadLockReport](../../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L1139-L1146) [high-availability.sgml#synchronous-replication-impact](../../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L1238-L1244) [xlog.c#pgstat_reset_all-call](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6843-L6846) | Identify the client backends, blockers, configured timeout, expected topology, and replication stage. A single intentional timeout is a guardrail event, not automatically an incident. After a crash, re-baseline counters and vacuum busy tables. |
| Sustained degradation | Growing dead rows, growing `dead but not yet removable` counts, repeated `canceling autovacuum task`, stale analyze times, repeated temp files, slow statements, frequent requested checkpoints, or backend writes and fsyncs | Table and index space and query cost grow, plans use stale statistics, temp and checkpoint I/O compete with the workload, and latency rises. [monitoring.sgml#pg_stat_all_tables](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2773-L2831) [monitoring.sgml#pg_stat_bgwriter](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2412-L2475) [vacuumlazy.c#autovacuum-summary](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L380-L441) | Check autovacuum thresholds, horizons, and lock conflicts, the top `pg_stat_statements` deltas, spilling sorts and hashes, and WAL and checkpoint sizing. [autovacuum.c#threshold-decision](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3060-L3080) [config.sgml#log_temp_files](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6562-L6573) [checkpointer.c#checkpoint-warning](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462) |
| Observability gap | Redacted activity or query fields, a hidden `backend_type`, truncated query text, disabled I/O timing, a missing extension or topology view, a collector warning, wiped counters, or `pending_restart = true` | The database may be unhealthy while the evidence is missing. Empty event logs and zero I/O time are not gaps by themselves when nothing qualifying happened. [config.sgml#track_activity_query_size](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6820-L6834) [config.sgml#track_io_timing](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L6854-L6871) [catalogs.sgml#pg_settings_pending_restart](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L10494-L10498) | Verify access, effective settings, topology, extension loading, and the counter epoch first. Apply the needed restart, reload, or session setting, then start a new measurement interval before drawing conclusions. |

### Final causal summary

Settings and modules decide what PostgreSQL 12 records; the monitoring role
decides what you can read of it; the counter epoch decides whether two readings
can be compared. Within those limits, live views show current pressure
(sessions, locks, replication), cumulative counters show what happened over
the interval, catalog horizons show wraparound distance, and the log shows
the events. A finding is a health problem only after all three gates pass:
the data was collected, it was visible to the reader, and no reset, crash, or
standby restart fell inside the interval.

## Measurement Script

This is the page's single measurement script. It reproduces every run-backed
statement on this page: the visibility rows in [Who can see what](#who-can-see-what),
the default and damaged-page legs in
[Defaults that decide what you see](#defaults-that-decide-what-you-see), the
prepared-transaction blocker, lock-wait and deadlock logging, temp-file
accounting, the three duration-line forms, checkpoint and restartpoint lines,
the autovacuum summaries and cancellation, the two recovery-conflict outcomes,
the counter wipes in [When counters reset](#when-counters-reset), and one run
of every fenced SQL block on this page. A measurement shows what this build
did; the cited source explains why.

| Item | Details |
|---|---|
| Purpose | Backs the measured statements above and checks that every SQL block on the page runs on PostgreSQL 12.2 as a superuser and as a `pg_monitor` member on the primary, and as a superuser on a standby. |
| Invocation | Save the fenced script as `.wiki-runtime/tmp/dh12-work/measure.sh`, then run `bash .wiki-runtime/tmp/dh12-work/measure.sh` from the repository root. Pass stage names to run a subset, for example `bash .wiki-runtime/tmp/dh12-work/measure.sh logs report`. |
| Stages | Default order: `build default cluster standby access prepared deadlock logs autovac conflict crash checksum sqlcheck report clean`. Each stage is described in the next table. |
| Environment | `DH12_JOBS`, a positive integer, default `8`, sets build parallelism. Everything else is fixed: the pin, the sandbox `.wiki-runtime/tmp/dh12/`, its socket directory `s/`, ports 55471 (primary), 55472 (standby), and 55473 (checksum cluster), and the bootstrap superuser `wiki_dh12`. The script exports `LC_ALL=C`, `TMPDIR` inside the sandbox, and `PGCONNECT_TIMEOUT=10`, and unsets the libpq service, host, port, user, database, password, and option variables. |
| Prerequisites | Bash, Git, the C toolchain a Git checkout of PostgreSQL 12 needs (a C compiler, make, bison, flex, Perl), and POSIX utilities including `dd` and `cksum`. `pgrep` and `lsof` serve the teardown checks that MANDATORY Environment Isolation requires. The build uses `--without-readline --without-zlib`, which only change `psql` line editing and compression; no measured result depends on them. No network and no system PostgreSQL are used. Every object, role, prepared transaction, snapshot, crash, and corrupted page is a disposable fixture inside the sandbox. |
| Output | `.wiki-runtime/tmp/dh12/out/`. Read `results.txt` first; `report` prints it with the platform facts. Per-leg files sit beside it, for example `access-progress.txt`, `prepared-queries.txt`, `conflict-log.txt`, and the server logs `primary.log`, `standby.log`, and `ck.log`. |
| Runtime | On the recorded machine a cold build took about 60 s with `DH12_JOBS=16`, and the measurement stages about 60 s, 30 s of it the standby conflict delay. A re-run while the build exists skips compilation. |
| Cleanup | `clean` stops all three clusters, checks that no `postmaster.pid`, process, socket, or listening port remains, and deletes the sandbox. `stop` does the same without deleting. |

| Stage | What it does |
|---|---|
| `build` | Checks the pin and a clean source tree, configures and builds out of tree, installs the server, `pg_stat_statements`, and `pgstattuple`, then checks that the source tree is still clean. Reuses an existing build. |
| `default` | Creates a fresh primary with default settings, reads the defaults from `pg_settings`, shows that `PREPARE TRANSACTION` fails, writes an impossible header into one page of a disposable table, and reads it back. Ends with the primary stopped. |
| `cluster` | Writes the non-default settings into `dh12.conf`, includes it from `postgresql.conf`, and starts the primary. |
| `standby` | Builds a streaming standby with `pg_basebackup -R -C -S dh12_standby` and waits for streaming. |
| `access` | Creates the roles `dh_monitor` (a `pg_monitor` member) and `dh_plain`, runs a slowed superuser `VACUUM`, and probes progress, settings, activity, and `pg_stat_statements` visibility as each role; runs the earlier subscription query and `pg_ls_waldir()` as each role. |
| `prepared` | Prepares a transaction holding `ACCESS EXCLUSIVE`, blocks a reader behind it, and runs the page's blocker and prepared-lock queries. |
| `deadlock` | Runs a two-session, two-row deadlock and checks the counter and log. |
| `logs` | Runs a spilling query, three slow statements over three protocols, and a WAL burst; checks temp-file, duration, checkpoint, and restartpoint lines. |
| `autovac` | Holds a snapshot during an autovacuum, releases it, and then cancels a slowed autovacuum with a conflicting lock. |
| `conflict` | Creates one running and one idle `REPEATABLE READ` session on the standby, vacuums their rows away on the primary, waits for both cancellations, then restarts the standby cleanly. |
| `crash` | Kills one primary backend with `SIGKILL`, then stops the primary with `pg_ctl -m immediate`, reading counters and markers before and after each. |
| `checksum` | Creates a separate cluster with `initdb -k`, changes one free-space byte in two tables, and reads them with default, `ignore_checksum_failure`, and `zero_damaged_pages` settings. Ends with that cluster stopped. |
| `sqlcheck` | Extracts every fenced SQL block from this page, substitutes the two `schema.*_name` placeholders with a fixture, and runs each block as listed in Purpose. |
| `report` | Prints platform facts and `results.txt`. |
| `stop`, `clean` | Stop the clusters with the teardown checks; `clean` also deletes the sandbox. |

Every setting and fixture state that departs from the default is listed below,
with its scope and the results that depend on it.

| Setting or state | Default | Value here | Where set | Context and apply scope | Why the script needs it | Results that depend on it |
|---|---|---|---|---|---|---|
| `shared_preload_libraries` | `''` | `pg_stat_statements` | `dh12.conf` | `PGC_POSTMASTER`, restart [guc.c#shared_preload_libraries](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3767-L3776) | Loads the module so its view exists | `pg_stat_statements` redaction and crash survival |
| `max_prepared_transactions` | `0` | `10` | `dh12.conf` | `PGC_POSTMASTER`, restart [guc.c#max_prepared_transactions](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2344-L2352) | Prepared transactions fail at the default, which the `default` stage measures | The PID-zero blocker and the prepared-lock rows |
| `log_lock_waits` | `off` | `on` | `dh12.conf` | `PGC_SUSET`, reload or `SET` [guc.c#log_lock_waits](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1489-L1497) | Produces the wait lines; the default logs none | Wait lines in the `prepared`, `deadlock`, and `autovac` stages; the deadlock ERROR does not depend on it |
| `log_temp_files` | `-1` | `0` | `dh12.conf` | `PGC_SUSET`, reload or `SET` [guc.c#log_temp_files](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3153-L3162) | Logs each file so its size can be summed | Temp-file lines; the counters count either way |
| `log_min_duration_statement` | `-1` | `250` ms | `dh12.conf` | `PGC_SUSET`, reload or `SET` [guc.c#log_min_duration_statement](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2703-L2713) | Logs the 0.5 s statements | The three duration lines |
| `log_checkpoints` | `off` | `on` | `dh12.conf` | `PGC_SIGHUP`, reload [guc.c#log_checkpoints](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1217-L1225) | Logs checkpoint and restartpoint start lines with their cause | Start lines; the frequency line does not depend on it |
| `log_autovacuum_min_duration` | `-1` | `0` | `dh12.conf` | `PGC_SIGHUP`, reload [guc.c#log_autovacuum_min_duration](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2715-L2725) | Logs every autovacuum summary | The held and released summaries; the cancellation ERROR does not depend on it |
| `autovacuum_naptime` | `60` s | `1` s | `dh12.conf` | `PGC_SIGHUP`, reload [guc.c#autovacuum_naptime](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2937-L2946) | Makes autovacuum visit within seconds | Timing of the `autovac` stage only |
| `max_wal_size`, `min_wal_size` | `1GB` (written by `initdb`), `80MB` | `32MB`, `32MB` | `dh12.conf` | `PGC_SIGHUP`, reload [guc.c#min_wal_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2542-L2552) [guc.c#max_wal_size](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2554-L2564) | Makes a 150,000-row insert trigger WAL-caused checkpoints | Checkpoint, frequency, and restartpoint lines |
| Physical slot `dh12_standby` | No slot | Created by `pg_basebackup -C -S` | `standby` stage | Fixture state [pg_basebackup.sgml#write-recovery-conf](../../../../raw/postgres-12/doc/src/sgml/ref/pg_basebackup.sgml#L212-L225) [xlog.c#KeepLogSeg](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9307-L9349) | With 32 MB of WAL and no slot, the primary can remove a segment the standby has not received | Every standby result; the slot holds no xmin because the standby sends no feedback |
| `work_mem` | `4MB` | `64kB` | Session `SET` | `PGC_USERSET`, session [guc.c#work_mem](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2230-L2241) | Forces the function scan and the sort to spill | The temp-file counts and bytes |
| `vacuum_cost_delay`, `vacuum_cost_limit` | `0`, `200` | `20` ms, `1` | Session `SET` | `PGC_USERSET`, session [guc.c#vacuum_cost_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2311-L2319) [guc.c#vacuum_cost_delay](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3372-L3381) | Keeps the manual `VACUUM` running for seconds so its progress row can be probed | The progress visibility rows |
| `statement_timeout`, `lock_timeout` | `0`, `0` | `120s` and `10s` in every session; `lock_timeout` `30s` or `60s` where a session must wait | Session `SET` | `PGC_USERSET`, session [guc.c#statement_timeout-lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2397) | Guardrails required for production-style SQL | None |
| `ignore_checksum_failure`, `zero_damaged_pages` | `off`, `off` | `on` in one session each | Session `SET` | `PGC_SUSET`, session [guc.c#damaged-page-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1134-L1162) | Shows what each one changes | The 100-row and 0-row reads |
| Data checksums | Off | On for the `ck` cluster only | `initdb -k` | Cluster creation [initdb.sgml#data-checksums](../../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L225) | Checksum verification needs it | Every `checksum` stage result |
| Table storage parameters | `autovacuum_vacuum_threshold` and `scale_factor` `-1` (use 50 rows and 0.2), cost delay `-1` (use 2 ms), cost limit `-1` (use 200), `autovacuum_enabled = true` | Threshold `0` and scale `0` on `dh_av` and `dh_avc`; cost delay `20` and limit `1` on `dh_avc`; `autovacuum_enabled = false` on `dh_rc` and `dh_crash` | `CREATE TABLE ... WITH` | Per table [reloptions.c#autovacuum-options](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L107-L243) [reloptions.c#real-options](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L358-L377) [guc.c#autovacuum_vacuum_threshold](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2947-L2955) [guc.c#autovacuum_vacuum_scale_factor](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3394-L3402) [guc.c#autovacuum_vacuum_cost_delay](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3383-L3392) | Makes autovacuum visit the two test tables at once, keeps one worker busy long enough to cancel, and keeps autovacuum away from the conflict and crash tables | The `autovac`, `conflict`, and `crash` results |
| `port`, `unix_socket_directories`, `listen_addresses` | `5432`, `/tmp`, `localhost` | Sandbox ports, the sandbox socket directory, `''` | `pg_ctl -o` | `PGC_POSTMASTER`, restart [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177-L2185) [guc.c#socket-and-listen](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936-L3960) | Isolation | None |
| Fixture states | None | A prepared transaction, held snapshots on the primary and the standby, one impossible page header, two changed free-space bytes, one `SIGKILL`, and one immediate shutdown | The stages | Disposable | Each creates the condition its claim describes | The matching stage results |

**Last run:** 2026-10-08, report written at 19:52:57Z, PostgreSQL 12.2 at
`45b88269a353ad93744772791feb6d01bc7e1e42`, built by this script with
`--without-readline --without-zlib`; Linux x86_64 (a WSL2 kernel), gcc 13.3.0,
64-bit, `block_size=8192`, `max_data_alignment=8`, locale `C`, UTF-8; data
checksums off on the primary and standby and on for the `ck` cluster. The run
started from an empty sandbox and ran every stage from `build` to `report`;
each stage passed its built-in checks, and `clean` ran afterwards. The
published script is byte-identical to the text that ran. These observations
supplement the cited source; they do not replace it.

| Stage | Measured result |
|---|---|
| `default` | Defaults as listed in the defaults table; `PREPARE TRANSACTION` failed with `prepared transactions are disabled`; the impossible header raised `invalid page in block 0 of relation base/...`; `checksum_failures` was NULL |
| `access` | Superuser: `relid` visible, phase `scanning heap`, `heap_blks_total = 173`; `pg_monitor` and ordinary role: `pid` and `datname` only. Settings: ordinary role saw `archive_command` only. Activity: the ordinary role saw 8 rows, 7 with NULL `backend_type`, and counted 1 client backend against 2. `pg_stat_statements`: all other-user rows redacted for the ordinary role, none for `pg_monitor`. Earlier subscription query: superuser ok, `pg_monitor` `permission denied for table pg_subscription`. `pg_ls_waldir()`: allowed for `pg_monitor`, denied for the ordinary role |
| `prepared` | `pg_blocking_pids()` returned `{0}`; the prepared transaction held `relation` `AccessExclusiveLock` and `transactionid` `ExclusiveLock`, both with NULL `pid` and `-1/<transaction>`; the wait line said `Process holding the lock: 0.` |
| `deadlock` | One victim; `deadlocks` 0 to 1; `detected deadlock while waiting for ShareLock on transaction` and `deadlock detected` logged |
| `logs` | 2 temp files, 5,634,432 bytes in both the counters and the log; `statement:`, `execute <unnamed>:`, and `execute P0_2:` duration lines; one `checkpoint starting: immediate force wait`, three `checkpoint starting: wal`, and three LOG `checkpoints are occurring too frequently` lines (two `0 seconds apart`, one `1 second apart`) with hints; standby: two `restartpoint starting: wal`, one `restartpoint starting: time`, and no frequency line |
| `autovac` | Held: `0 removed, 2000 remain, 1000 are dead but not yet removable`; released: `1000 removed, 1000 remain, 0 are dead but not yet removable`; cancellation: `canceling autovacuum task` with CONTEXT `automatic vacuum of table "postgres.public.dh_avc"`, waiter acquired after about 1000 ms, no `sending cancel` line |
| `conflict` | One FATAL `terminating connection due to conflict with recovery` and one ERROR `canceling statement due to conflict with recovery`, both with DETAIL `User query might have needed to see row versions that must be removed.`, 30 s after the `VACUUM`; `confl_snapshot` 0 to 2; after a clean standby restart, 0 with a new `stats_reset` |
| `crash` | `SIGKILL`: the full reinitialization chain; `n_dead_tup` 500 to 0, both `stats_reset` values new, the marker statement gone from `pg_stat_statements`. Immediate shutdown: `n_dead_tup` 250 to 0, both `stats_reset` values new, the second marker still present |
| `checksum` | Default: checksum WARNING plus `invalid page` ERROR; `ignore_checksum_failure`: WARNING and 100 rows; `zero_damaged_pages`: WARNING, `zeroing out page` WARNING, and 0 rows; `checksum_failures = 3` |
| `sqlcheck` | All 11 fenced SQL blocks ran without error as listed in Purpose |

```bash
#!/usr/bin/env bash
set -uo pipefail
# PostgreSQL 12 database health checklist measurements.
# Disposable fixtures only: every table, role, prepared transaction, snapshot
# and corrupted page below lives in the three sandbox clusters this script
# creates. Never point it at a cluster that matters.
# Run from the repository root:
#   bash .wiki-runtime/tmp/dh12-work/measure.sh [stage ...]
REPO=$(pwd -P)
PIN=45b88269a353ad93744772791feb6d01bc7e1e42
SRC=$REPO/raw/postgres-12
PAGE=$REPO/wiki/v12/questions/server-administration/database-health-checklist.md
RUN=$REPO/.wiki-runtime/tmp/dh12
BUILD=$RUN/build
INSTALL=$RUN/install
OUT=$RUN/out
SOCK=$RUN/s
PDATA=$RUN/primary
SDATA=$RUN/standby
CDATA=$RUN/ck
PPORT=55471
SPORT=55472
CPORT=55473
SU=wiki_dh12
JOBS=${DH12_JOBS:-8}
case "$JOBS" in ''|*[!0-9]*|0) printf 'DH12_JOBS must be a positive integer\n' >&2; exit 2 ;; esac
OWNER="dh12 $REPO $PIN $PPORT $SPORT $CPORT"
export LC_ALL=C TMPDIR=$RUN/t PGCONNECT_TIMEOUT=10
unset PGSERVICE PGSERVICEFILE PGHOST PGHOSTADDR PGPORT PGOPTIONS PGDATABASE
unset PGUSER PGPASSWORD PGPASSFILE PGTARGETSESSIONATTRS PGSSLMODE PGAPPNAME
GUARD="SET /* wiki_dh12_guard */ statement_timeout = '120s';
SET /* wiki_dh12_guard */ lock_timeout = '10s';"

die() { printf 'FAIL: %s\n' "$*" >&2; exit 1; }
owned() { test -f "$RUN/owner" && test "$(cat "$RUN/owner")" = "$OWNER"; }
claim() {
  if test -e "$RUN"; then owned || die "Existing unowned sandbox: $RUN"; fi
  mkdir -p "$RUN" "$OUT" "$SOCK" "$TMPDIR" || die "mkdir failed"
  printf '%s\n' "$OWNER" > "$RUN/owner"
}
need() { owned || die "Run the build stage first"; test -x "$INSTALL/bin/postgres" || die "No build"; }
psql_as() {
  local port=$1 user=$2; shift 2
  "$INSTALL/bin/psql" -X -v ON_ERROR_STOP=1 -h "$SOCK" -p "$port" -U "$user" -d postgres "$@"
}
q() {
  local port=$1 user=$2; shift 2
  { printf '%s\n' "$GUARD"; cat; } | psql_as "$port" "$user" -q "$@"
}
val() { printf '%s\n' "$3" | q "$1" "$2" -A -t; }
mark() { if test -f "$1"; then wc -l < "$1" | tr -d ' '; else printf '0\n'; fi; }
since() { tail -n +"$(( $2 + 1 ))" "$1"; }
has_since() { since "$1" "$2" | grep -E -q -- "$3"; }
wait_for() {
  local left=$1 what=$2; shift 2
  while test "$left" -gt 0; do
    if "$@" > /dev/null 2>&1; then return 0; fi
    sleep 1; left=$((left - 1))
  done
  die "Timed out waiting for $what"
}
reset_results() {
  if test -f "$OUT/results.txt"; then
    grep -v "^$1\." "$OUT/results.txt" > "$OUT/results.tmp"
    mv "$OUT/results.tmp" "$OUT/results.txt"
  fi
}
record() { printf '%s\n' "$*" >> "$OUT/results.txt"; }
flat() { tr '\n' ';' < "$1"; }
running() { test -f "$1/postmaster.pid" && "$INSTALL/bin/pg_ctl" -D "$1" status > /dev/null 2>&1; }
port_busy() { lsof -nP -iTCP:"$1" -sTCP:LISTEN > /dev/null 2>&1 || test -e "$SOCK/.s.PGSQL.$1"; }
start_cluster() {
  if port_busy "$2"; then die "Port or socket $2 is already in use"; fi
  "$INSTALL/bin/pg_ctl" -D "$1" -l "$3" -w -t 120 start \
    -o "-p $2 -k $SOCK -c listen_addresses=''" > /dev/null || die "Start failed: $1"
}
stop_one() {
  owned || return 1
  if test -f "$1/postmaster.pid"; then
    "$INSTALL/bin/pg_ctl" -D "$1" -m fast -w -t 120 stop > /dev/null || return 1
  fi
  test ! -e "$1/postmaster.pid" || return 1
  # Teardown checks required by MANDATORY Environment Isolation.
  if pgrep -f -- "$1" > /dev/null 2>&1; then return 1; fi
  test ! -e "$SOCK/.s.PGSQL.$2" || return 1
  test ! -e "$SOCK/.s.PGSQL.$2.lock" || return 1
  if lsof -nP -iTCP:"$2" -sTCP:LISTEN > /dev/null 2>&1; then return 1; fi
  return 0
}
stop_all() {
  local rc=0
  stop_one "$SDATA" "$SPORT" || rc=1
  stop_one "$PDATA" "$PPORT" || rc=1
  stop_one "$CDATA" "$CPORT" || rc=1
  return "$rc"
}
on_exit() {
  status=$?
  bg=$(jobs -p)
  if test -n "$bg"; then kill $bg 2> /dev/null; fi
  if test "$status" -ne 0 && owned; then
    stop_all || printf 'Teardown check failed; inspect %s\n' "$RUN" >&2
  fi
}
trap on_exit EXIT

build() {
  test -f "$REPO/wiki/versions.md" || die "Run from the repository root"
  test "$(git --no-optional-locks -C "$SRC" rev-parse HEAD)" = "$PIN" || die "Wrong pin"
  git --no-optional-locks -C "$SRC" diff --quiet || die "Tracked source changes"
  git --no-optional-locks -C "$SRC" diff --cached --quiet || die "Staged source changes"
  claim
  mkdir -p "$BUILD" || die "mkdir failed"
  if test ! -f "$BUILD/GNUmakefile"; then
    (cd "$BUILD" && "$SRC/configure" --prefix="$INSTALL" --without-readline --without-zlib) \
      > "$OUT/configure.log" 2>&1 || die "See $OUT/configure.log"
  fi
  make -C "$BUILD" -j "$JOBS" > "$OUT/build.log" 2>&1 || die "See $OUT/build.log"
  make -C "$BUILD" install > "$OUT/install.log" 2>&1 || die "See $OUT/install.log"
  for m in pg_stat_statements pgstattuple; do
    make -C "$BUILD/contrib/$m" -j "$JOBS" all install > "$OUT/build-$m.log" 2>&1 \
      || die "See $OUT/build-$m.log"
  done
  git --no-optional-locks -C "$SRC" status --porcelain > "$OUT/source-status.txt" \
    || die "git status failed"
  test ! -s "$OUT/source-status.txt" || die "The build wrote into the source tree"
}

default_leg() {
  need; reset_results default
  stop_one "$SDATA" "$SPORT" || die "Standby teardown failed"
  stop_one "$PDATA" "$PPORT" || die "Primary teardown failed"
  rm -rf "$PDATA" "$SDATA" "$OUT/primary.log" "$OUT/standby.log"
  "$INSTALL/bin/initdb" -D "$PDATA" -A trust -U "$SU" --locale=C --encoding=UTF8 \
    > "$OUT/initdb-primary.log" 2>&1 || die "initdb failed"
  start_cluster "$PDATA" "$PPORT" "$OUT/primary.log"
  q "$PPORT" "$SU" -A -F '|' > "$OUT/default-settings.txt" <<'SQL' || die "Settings capture failed"
SELECT /* wiki_dh12_defaults */ name, setting, coalesce(unit, ''), boot_val, source, context
FROM pg_settings
WHERE name IN ('checkpoint_warning', 'data_checksums', 'deadlock_timeout',
  'hot_standby', 'hot_standby_feedback', 'ignore_checksum_failure', 'lock_timeout',
  'log_autovacuum_min_duration', 'log_checkpoints', 'log_connections',
  'log_destination', 'log_disconnections', 'log_line_prefix', 'log_lock_waits',
  'log_min_duration_statement', 'log_min_messages', 'log_temp_files',
  'logging_collector', 'max_prepared_transactions', 'max_standby_streaming_delay',
  'max_wal_size', 'restart_after_crash', 'shared_preload_libraries',
  'statement_timeout', 'track_activity_query_size', 'track_io_timing',
  'zero_damaged_pages')
ORDER BY name;
SQL
  record "default.settings: $(flat "$OUT/default-settings.txt")"
  if psql_as "$PPORT" "$SU" -c "SET /* wiki_dh12_guard */ statement_timeout = '120s'" \
      -c "SET /* wiki_dh12_guard */ lock_timeout = '10s'" \
      -c "BEGIN /* wiki_dh12_default_2pc */" \
      -c "PREPARE /* wiki_dh12_default_2pc */ TRANSACTION 'dh12_default'" \
      > "$OUT/default-2pc.txt" 2>&1; then
    die "PREPARE TRANSACTION succeeded with the default max_prepared_transactions"
  fi
  grep -q 'prepared transactions are disabled' "$OUT/default-2pc.txt" || die "Unexpected 2PC result"
  record "default.prepare_transaction: $(grep -E 'ERROR|HINT' "$OUT/default-2pc.txt" | tr '\n' ' ')"
  hdr=$(q "$PPORT" "$SU" -A -t <<'SQL'
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_hdr;
CREATE /* wiki_dh12_disposable */ TABLE dh_hdr (id integer, pad text);
INSERT /* wiki_dh12_disposable */ INTO dh_hdr SELECT g, repeat('h', 20) FROM generate_series(1, 100) AS g;
CHECKPOINT /* wiki_dh12_flush */;
SELECT /* wiki_dh12_path */ pg_relation_filepath('dh_hdr');
SQL
) || die "Header fixture failed"
  "$INSTALL/bin/pg_ctl" -D "$PDATA" -m fast -w -t 120 stop > /dev/null || die "Stop failed"
  test -f "$PDATA/$hdr" || die "Missing relation file $hdr"
  # Disposable corruption: pd_lower = 65535 > pd_upper = 256 in block 0 (little-endian).
  printf '\377\377\000\001' | dd of="$PDATA/$hdr" bs=1 seek=12 count=4 conv=notrunc \
    2> "$OUT/dd-header.txt" || die "dd failed"
  start_cluster "$PDATA" "$PPORT" "$OUT/primary.log"
  if printf '%s\n' "SELECT /* wiki_dh12_header_probe */ count(*) FROM dh_hdr;" \
      | q "$PPORT" "$SU" > "$OUT/default-header.txt" 2>&1; then
    die "The corrupted page header was read without an error"
  fi
  grep -q 'ERROR:  invalid page in block 0 of relation base/' "$OUT/default-header.txt" \
    || die "Unexpected header-corruption result"
  record "default.header_corruption: $(grep 'ERROR' "$OUT/default-header.txt")"
  ck=$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_checksum_counter */ coalesce(checksum_failures::text, 'NULL') FROM pg_stat_database WHERE datname = current_database();") \
    || die "Counter query failed"
  record "default.checksum_failures_without_checksums: $ck"
  printf '%s\n' "DROP /* wiki_dh12_disposable */ TABLE dh_hdr;" | q "$PPORT" "$SU" \
    || die "Drop failed"
  "$INSTALL/bin/pg_ctl" -D "$PDATA" -m fast -w -t 120 stop > /dev/null || die "Stop failed"
}

cluster() {
  need; reset_results cluster
  test -f "$PDATA/PG_VERSION" || die "Run the default stage first"
  stop_one "$SDATA" "$SPORT" || die "Standby teardown failed"
  stop_one "$PDATA" "$PPORT" || die "Primary teardown failed"
  grep -q "^include_if_exists = 'dh12.conf'" "$PDATA/postgresql.conf" \
    || printf '%s\n' "include_if_exists = 'dh12.conf'" >> "$PDATA/postgresql.conf"
  cat > "$PDATA/dh12.conf" <<'CONF'
# Non-default settings for the disposable measurement primary; see the page.
shared_preload_libraries = 'pg_stat_statements'
max_prepared_transactions = 10
log_lock_waits = on
log_temp_files = 0
log_min_duration_statement = 250
log_checkpoints = on
log_autovacuum_min_duration = 0
autovacuum_naptime = 1
max_wal_size = 32MB
min_wal_size = 32MB
CONF
  start_cluster "$PDATA" "$PPORT" "$OUT/primary.log"
  q "$PPORT" "$SU" -A -t > "$OUT/cluster-settings.txt" <<'SQL' || die "Settings capture failed"
SELECT /* wiki_dh12_nondefault */ name || '=' || setting || coalesce(unit, '')
FROM pg_settings WHERE sourcefile LIKE '%/dh12.conf' ORDER BY name;
SQL
  record "cluster.nondefault: $(flat "$OUT/cluster-settings.txt")"
}

streaming() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_replication */ count(*) FROM pg_stat_replication WHERE state = 'streaming';")" = 1
}
standby() {
  need; reset_results standby
  running "$PDATA" || die "Run the cluster stage first"
  stop_one "$SDATA" "$SPORT" || die "Standby teardown failed"
  rm -rf "$SDATA" "$OUT/standby.log"
  # A physical slot keeps the primary from recycling WAL the standby still needs
  # (max_wal_size is only 32MB here); the standby sends no feedback, so the slot
  # holds no xmin.
  val "$PPORT" "$SU" "SELECT /* wiki_dh12_slot_reset */ count(pg_drop_replication_slot(slot_name)) FROM pg_replication_slots WHERE slot_name = 'dh12_standby';" \
    > /dev/null || die "Slot reset failed"
  "$INSTALL/bin/pg_basebackup" -h "$SOCK" -p "$PPORT" -U "$SU" -D "$SDATA" -R -X stream \
    -C -S dh12_standby -c fast > "$OUT/basebackup.log" 2>&1 || die "pg_basebackup failed"
  start_cluster "$SDATA" "$SPORT" "$OUT/standby.log"
  wait_for 60 "streaming replication" streaming
  record "standby.replication: $(val "$PPORT" "$SU" "SELECT /* wiki_dh12_replication */ state || ' ' || sync_state FROM pg_stat_replication;")"
  record "standby.slot: $(val "$PPORT" "$SU" "SELECT /* wiki_dh12_slot */ slot_name || ' ' || slot_type || ' active=' || active || ' xmin=' || coalesce(xmin::text, 'NULL') FROM pg_replication_slots;")"
  record "standby.in_recovery: $(val "$SPORT" "$SU" "SELECT /* wiki_dh12_recovery */ pg_is_in_recovery();")"
}

roles() {
  q "$PPORT" "$SU" > "$OUT/roles.txt" 2>&1 <<'SQL' || die "Role setup failed"
DO /* wiki_dh12_roles */ $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'dh_monitor') THEN
    CREATE /* wiki_dh12_roles */ ROLE dh_monitor LOGIN;
  END IF;
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'dh_plain') THEN
    CREATE /* wiki_dh12_roles */ ROLE dh_plain LOGIN;
  END IF;
END $$;
GRANT /* wiki_dh12_roles */ pg_monitor TO dh_monitor;
CREATE /* wiki_dh12_extension */ EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE /* wiki_dh12_extension */ EXTENSION IF NOT EXISTS pgstattuple;
SQL
}
progress_row() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_progress_wait */ count(*) FROM pg_stat_progress_vacuum WHERE relid = 'dh_vac'::regclass;")" = 1
}
access() {
  need; reset_results access
  running "$PDATA" || die "Primary is not running"
  roles
  q "$PPORT" "$SU" > "$OUT/access-fixtures.txt" <<'SQL' || die "Access fixtures failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_vac;
CREATE /* wiki_dh12_disposable */ TABLE dh_vac (id integer, pad text);
INSERT /* wiki_dh12_disposable */ INTO dh_vac SELECT g, repeat('v', 100) FROM generate_series(1, 10000) AS g;
DELETE /* wiki_dh12_disposable */ FROM dh_vac WHERE id % 2 = 0;
SELECT /* wiki_dh12_pgss_marker */ 42 AS marker;
SQL
  rm -f "$OUT"/access-progress.txt "$OUT"/access-settings.txt "$OUT"/access-activity.txt \
    "$OUT"/access-pgss.txt
  { printf '%s\n' "$GUARD" \
      "SET /* wiki_dh12_slow_vacuum */ vacuum_cost_delay = 20;" \
      "SET /* wiki_dh12_slow_vacuum */ vacuum_cost_limit = 1;" \
      "VACUUM /* wiki_dh12_slow_vacuum */ dh_vac;"
  } | psql_as "$PPORT" "$SU" -q > "$OUT/access-slow-vacuum.txt" 2>&1 &
  vacpid=$!
  wait_for 30 "the VACUUM progress row" progress_row
  for who in "$SU" dh_monitor dh_plain; do
    q "$PPORT" "$who" -A -F '|' -t >> "$OUT/access-progress.txt" <<'SQL' || die "Progress probe failed"
SELECT /* wiki_dh12_progress_probe */ current_user, pid IS NOT NULL, datname,
       relid IS NOT NULL, coalesce(phase, 'NULL'), coalesce(heap_blks_total::text, 'NULL')
FROM pg_stat_progress_vacuum WHERE datname = current_database();
SQL
    q "$PPORT" "$who" -A -F '|' -t >> "$OUT/access-settings.txt" <<'SQL' || die "Settings probe failed"
SELECT /* wiki_dh12_settings_probe */ current_user, string_agg(name, ',' ORDER BY name)
FROM pg_settings WHERE name IN ('archive_command', 'data_directory', 'shared_preload_libraries');
SQL
    q "$PPORT" "$who" -A -F '|' -t >> "$OUT/access-activity.txt" <<'SQL' || die "Activity probe failed"
SELECT /* wiki_dh12_activity_probe */ current_user, count(*),
       count(*) FILTER (WHERE backend_type IS NULL),
       count(*) FILTER (WHERE query = '<insufficient privilege>'),
       count(*) FILTER (WHERE backend_type = 'client backend')
FROM pg_stat_activity;
SQL
    q "$PPORT" "$who" -A -F '|' -t >> "$OUT/access-pgss.txt" <<'SQL' || die "pg_stat_statements probe failed"
SELECT /* wiki_dh12_pgss_probe */ current_user,
       count(*) FILTER (WHERE s.userid <> me.oid),
       count(*) FILTER (WHERE s.userid <> me.oid AND s.queryid IS NULL),
       count(*) FILTER (WHERE s.userid <> me.oid AND s.query = '<insufficient privilege>')
FROM pg_stat_statements AS s
CROSS JOIN (SELECT oid FROM pg_roles WHERE rolname = current_user) AS me;
SQL
  done
  wait "$vacpid" || die "The slow VACUUM failed"
  grep -q "^$SU|t|postgres|t|" "$OUT/access-progress.txt" || die "Superuser progress row"
  grep -q '^dh_monitor|t|postgres|f|NULL|NULL$' "$OUT/access-progress.txt" || die "pg_monitor progress row"
  grep -q '^dh_plain|t|postgres|f|NULL|NULL$' "$OUT/access-progress.txt" || die "Plain progress row"
  grep -q '^dh_plain|archive_command$' "$OUT/access-settings.txt" || die "Plain settings"
  grep -q '^dh_monitor|archive_command,data_directory,shared_preload_libraries$' \
    "$OUT/access-settings.txt" || die "pg_monitor settings"
  record "access.progress(user|pid|datname|relid|phase|heap_blks_total): $(flat "$OUT/access-progress.txt")"
  record "access.settings(user|visible): $(flat "$OUT/access-settings.txt")"
  record "access.activity(user|rows|null_backend_type|redacted_query|client_backend_rows): $(flat "$OUT/access-activity.txt")"
  record "access.pgss(user|other_rows|null_queryid|redacted_text): $(flat "$OUT/access-pgss.txt")"
  # The subscription query filed on 2026-07-29, unchanged.
  cat > "$OUT/old-subscription.sql" <<'SQL'
SELECT /* wiki_db_health_subscription */
       d.datname,
       st.subid,
       st.subname,
       sub.subenabled,
       sub.subslotname,
       st.pid,
       st.relid,
       CASE
         WHEN st.pid IS NULL THEN 'worker not running'
         WHEN st.relid IS NULL THEN 'main apply worker'
         ELSE 'table synchronization worker'
       END AS worker_state,
       st.received_lsn,
       st.last_msg_send_time,
       st.last_msg_receipt_time,
       now() - st.last_msg_receipt_time AS time_since_last_message,
       st.latest_end_lsn,
       st.latest_end_time
FROM pg_stat_subscription AS st
JOIN pg_subscription AS sub ON sub.oid = st.subid
LEFT JOIN pg_database AS d ON d.oid = sub.subdbid
ORDER BY d.datname, st.subname, st.relid NULLS FIRST;
SQL
  q "$PPORT" "$SU" < "$OUT/old-subscription.sql" > "$OUT/access-subscription-su.txt" 2>&1 \
    || die "The old subscription query failed for the superuser"
  if q "$PPORT" dh_monitor < "$OUT/old-subscription.sql" > "$OUT/access-subscription-monitor.txt" 2>&1; then
    die "The old subscription query worked for pg_monitor"
  fi
  grep -q 'permission denied for table pg_subscription' "$OUT/access-subscription-monitor.txt" \
    || die "Unexpected subscription error"
  record "access.old_subscription_query.superuser: ok"
  record "access.old_subscription_query.dh_monitor: $(grep 'ERROR' "$OUT/access-subscription-monitor.txt")"
  for who in dh_monitor dh_plain; do
    if printf '%s\n' "SELECT /* wiki_dh12_ls_probe */ count(*) FROM pg_ls_waldir();" \
        | q "$PPORT" "$who" > "$OUT/access-ls-$who.txt" 2>&1; then
      record "access.pg_ls_waldir.$who: allowed"
    else
      record "access.pg_ls_waldir.$who: $(sed -n 's/.*ERROR: *//p' "$OUT/access-ls-$who.txt")"
    fi
  done
}

lock_waiter() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_px_wait */ count(*) FROM pg_stat_activity WHERE wait_event_type = 'Lock' AND query LIKE '%wiki_dh12_px_waiter%' AND pid <> pg_backend_pid();")" = 1
}
prepared() {
  need; reset_results prepared
  running "$PDATA" || die "Primary is not running"
  if test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_px_leftover */ count(*) FROM pg_prepared_xacts WHERE gid = 'dh12_px';")" != 0; then
    printf '%s\n' "ROLLBACK /* wiki_dh12_px_cleanup */ PREPARED 'dh12_px';" \
      | q "$PPORT" "$SU" || die "Leftover cleanup failed"
  fi
  q "$PPORT" "$SU" <<'SQL' || die "Prepared fixture failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_px;
CREATE /* wiki_dh12_disposable */ TABLE dh_px (id integer);
INSERT /* wiki_dh12_disposable */ INTO dh_px VALUES (1);
BEGIN /* wiki_dh12_px */;
LOCK /* wiki_dh12_px */ TABLE dh_px IN ACCESS EXCLUSIVE MODE;
PREPARE /* wiki_dh12_px */ TRANSACTION 'dh12_px';
SQL
  m=$(mark "$OUT/primary.log")
  { printf '%s\n' "$GUARD" "SET /* wiki_dh12_px_waiter */ lock_timeout = '60s';" \
      "SELECT /* wiki_dh12_px_waiter */ count(*) FROM dh_px;"
  } | psql_as "$PPORT" "$SU" -q -A -t > "$OUT/prepared-waiter.txt" 2>&1 &
  waiter=$!
  wait_for 30 "the lock waiter" lock_waiter
  wait_for 30 "the lock-wait log line" has_since "$OUT/primary.log" "$m" \
    'still waiting for AccessShareLock on relation'
  q "$PPORT" "$SU" > "$OUT/prepared-queries.txt" <<'SQL' || die "Blocker queries failed"
SELECT /* wiki_db_health_blockers */
       waiter.pid AS waiting_pid,
       waiter.usename AS waiting_user,
       waiter.application_name AS waiting_application,
       waiter.wait_event AS waiting_event,
       now() - waiter.query_start AS waiting_query_age,
       blocker_pid.pid AS blocking_pid,
       CASE WHEN blocker_pid.pid = 0 THEN 'prepared transaction'
            ELSE blocker.backend_type
       END AS blocking_type,
       blocker.usename AS blocking_user,
       blocker.application_name AS blocking_application,
       blocker.state AS blocking_state,
       waiter.query AS waiting_query,
       blocker.query AS blocking_query
FROM pg_stat_activity AS waiter
CROSS JOIN LATERAL
     unnest(pg_blocking_pids(waiter.pid)) AS blocker_pid(pid)
LEFT JOIN pg_stat_activity AS blocker
  ON blocker.pid = NULLIF(blocker_pid.pid, 0)
WHERE waiter.wait_event_type = 'Lock'
ORDER BY waiting_query_age DESC, waiting_pid, blocking_pid;

SELECT /* wiki_db_health_prepared_transaction_locks */
       p.transaction,
       p.gid,
       p.prepared,
       now() - p.prepared AS prepared_age,
       p.owner,
       p.database,
       l.locktype,
       l.mode,
       l.granted,
       l.database AS lock_database_oid,
       l.relation AS relation_oid
FROM pg_prepared_xacts AS p
JOIN pg_locks AS l
  ON l.virtualtransaction = '-1/' || p.transaction::text
WHERE l.granted
ORDER BY p.prepared, p.gid, l.locktype, l.mode;
SQL
  q "$PPORT" "$SU" -A -F '|' -t > "$OUT/prepared-check.txt" <<'SQL' || die "Blocker check failed"
SELECT /* wiki_dh12_px_check */ 'blockers', string_agg(b.pid::text, ',' ORDER BY b.pid)
FROM pg_stat_activity AS w
CROSS JOIN LATERAL unnest(pg_blocking_pids(w.pid)) AS b(pid)
WHERE w.wait_event_type = 'Lock' AND w.query LIKE '%wiki_dh12_px_waiter%' AND w.pid <> pg_backend_pid();
SELECT /* wiki_dh12_px_check */ 'prepared_locks',
       string_agg(l.locktype || ':' || l.mode || ':pid=' || coalesce(l.pid::text, 'NULL')
                  || ':vxact_matches=' || (l.virtualtransaction = '-1/' || p.transaction::text),
                  ',' ORDER BY l.locktype, l.mode)
FROM pg_prepared_xacts AS p
JOIN pg_locks AS l ON l.virtualtransaction = '-1/' || p.transaction::text
WHERE l.granted AND p.gid = 'dh12_px';
SQL
  printf '%s\n' "ROLLBACK /* wiki_dh12_px */ PREPARED 'dh12_px';" | q "$PPORT" "$SU" \
    || die "ROLLBACK PREPARED failed"
  wait "$waiter" || die "The lock waiter failed"
  wait_for 15 "the acquired log line" has_since "$OUT/primary.log" "$m" \
    'acquired AccessShareLock on relation'
  since "$OUT/primary.log" "$m" \
    | grep -E 'still waiting for AccessShareLock|acquired AccessShareLock|Process holding the lock' \
    > "$OUT/prepared-log.txt"
  grep -q '^blockers|0$' "$OUT/prepared-check.txt" || die "The blocker was not reported as PID 0"
  grep -q '| prepared transaction |' "$OUT/prepared-queries.txt" || die "No prepared-transaction row"
  grep -q '^prepared_locks|relation:AccessExclusiveLock:pid=NULL:vxact_matches=true,transactionid:ExclusiveLock:pid=NULL:vxact_matches=true$' \
    "$OUT/prepared-check.txt" || die "Unexpected prepared-transaction locks"
  record "prepared.check: $(flat "$OUT/prepared-check.txt")"
  record "prepared.log: $(sed 's/^[^]]*] //' "$OUT/prepared-log.txt" | tr '\n' ';')"
}

dl_count() {
  val "$PPORT" "$SU" "SELECT /* wiki_dh12_deadlocks */ deadlocks FROM pg_stat_database WHERE datname = current_database();"
}
dl_reached() { test "$(dl_count)" -ge "$1"; }
deadlock() {
  need; reset_results deadlock
  running "$PDATA" || die "Primary is not running"
  q "$PPORT" "$SU" <<'SQL' || die "Deadlock fixture failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_dl;
CREATE /* wiki_dh12_disposable */ TABLE dh_dl (id integer PRIMARY KEY, v integer);
INSERT /* wiki_dh12_disposable */ INTO dh_dl VALUES (1, 0), (2, 0);
SQL
  before=$(dl_count) || die "Counter read failed"
  m=$(mark "$OUT/primary.log")
  { printf '%s\n' "$GUARD" "SET /* wiki_dh12_deadlock */ lock_timeout = '30s';" \
      "BEGIN /* wiki_dh12_deadlock */;" \
      "UPDATE /* wiki_dh12_deadlock_a */ dh_dl SET v = v + 1 WHERE id = 1;"
    sleep 3
    printf '%s\n' "UPDATE /* wiki_dh12_deadlock_a */ dh_dl SET v = v + 1 WHERE id = 2;" \
      "COMMIT /* wiki_dh12_deadlock */;"
  } | psql_as "$PPORT" "$SU" -q > "$OUT/deadlock-a.txt" 2>&1 &
  a=$!
  sleep 1
  { printf '%s\n' "$GUARD" "SET /* wiki_dh12_deadlock */ lock_timeout = '30s';" \
      "BEGIN /* wiki_dh12_deadlock */;" \
      "UPDATE /* wiki_dh12_deadlock_b */ dh_dl SET v = v + 1 WHERE id = 2;"
    sleep 2
    printf '%s\n' "UPDATE /* wiki_dh12_deadlock_b */ dh_dl SET v = v + 1 WHERE id = 1;" \
      "COMMIT /* wiki_dh12_deadlock */;"
  } | psql_as "$PPORT" "$SU" -q > "$OUT/deadlock-b.txt" 2>&1 &
  b=$!
  wait "$a"; sa=$?
  wait "$b"; sb=$?
  fails=0
  if test "$sa" -ne 0; then fails=$((fails + 1)); fi
  if test "$sb" -ne 0; then fails=$((fails + 1)); fi
  test "$fails" -eq 1 || die "Expected exactly one deadlock victim (a=$sa b=$sb)"
  wait_for 15 "the deadlock counter" dl_reached $((before + 1))
  after=$(dl_count) || die "Counter read failed"
  since "$OUT/primary.log" "$m" \
    | grep -E 'ERROR:  deadlock detected|detected deadlock while waiting|DETAIL:  Process [0-9]+ waits for' \
    > "$OUT/deadlock-log.txt"
  grep -q 'ERROR:  deadlock detected' "$OUT/deadlock-log.txt" || die "No deadlock ERROR in the log"
  record "deadlock.counter: before=$before after=$after victims=$fails"
  record "deadlock.log: $(sed 's/^[^]]*] //' "$OUT/deadlock-log.txt" | tr '\n' ';')"
}

temp_counted() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_temp_wait */ temp_files FROM pg_stat_database WHERE datname = current_database();")" -gt "$1"
}
logs() {
  need; reset_results logs
  running "$PDATA" || die "Primary is not running"
  running "$SDATA" || die "Standby is not running"
  t0=$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_temp_before */ temp_files || ' ' || temp_bytes FROM pg_stat_database WHERE datname = current_database();") \
    || die "Counter read failed"
  tf0=${t0% *}; tb0=${t0#* }
  m=$(mark "$OUT/primary.log")
  q "$PPORT" "$SU" -A -t > "$OUT/logs-temp.txt" <<'SQL' || die "Temp query failed"
SET /* wiki_dh12_temp */ work_mem = '64kB';
SELECT /* wiki_dh12_temp */ count(*) FROM (SELECT g FROM generate_series(1, 200000) AS g ORDER BY g DESC) AS s;
SQL
  wait_for 15 "the temp-file counters" temp_counted "$tf0"
  t1=$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_temp_after */ temp_files || ' ' || temp_bytes FROM pg_stat_database WHERE datname = current_database();") \
    || die "Counter read failed"
  tf1=${t1% *}; tb1=${t1#* }
  since "$OUT/primary.log" "$m" | grep 'LOG:  temporary file: path' > "$OUT/logs-temp-lines.txt"
  nlines=$(wc -l < "$OUT/logs-temp-lines.txt" | tr -d ' ')
  sum=0
  for s in $(sed -n 's/.*", size \([0-9][0-9]*\)$/\1/p' "$OUT/logs-temp-lines.txt"); do
    sum=$((sum + s))
  done
  test "$((tf1 - tf0))" -eq "$nlines" || die "temp_files delta differs from the log"
  test "$((tb1 - tb0))" -eq "$sum" || die "temp_bytes delta differs from the log"
  record "logs.temp: files_delta=$((tf1 - tf0)) bytes_delta=$((tb1 - tb0)) log_lines=$nlines log_bytes=$sum"
  m=$(mark "$OUT/primary.log")
  printf '%s\n' "SELECT /* wiki_dh12_simple_slow */ pg_sleep(0.5);" | q "$PPORT" "$SU" > /dev/null \
    || die "Simple-protocol query failed"
  printf '%s\n' "$GUARD" "SELECT /* wiki_dh12_extended_slow */ pg_sleep(0.5);" \
    > "$OUT/pgbench-extended.sql"
  printf '%s\n' "$GUARD" "SELECT /* wiki_dh12_prepared_slow */ pg_sleep(0.5);" \
    > "$OUT/pgbench-prepared.sql"
  "$INSTALL/bin/pgbench" -h "$SOCK" -p "$PPORT" -U "$SU" -n -t 1 -M extended \
    -f "$OUT/pgbench-extended.sql" postgres > "$OUT/pgbench-extended.txt" 2>&1 \
    || die "pgbench -M extended failed"
  "$INSTALL/bin/pgbench" -h "$SOCK" -p "$PPORT" -U "$SU" -n -t 1 -M prepared \
    -f "$OUT/pgbench-prepared.sql" postgres > "$OUT/pgbench-prepared.txt" 2>&1 \
    || die "pgbench -M prepared failed"
  wait_for 10 "the prepared duration line" has_since "$OUT/primary.log" "$m" \
    'duration: [0-9.]+ ms  execute P0_2: SELECT /\* wiki_dh12_prepared_slow'
  since "$OUT/primary.log" "$m" | grep -E 'duration: .*wiki_dh12_(simple|extended|prepared)_slow' \
    > "$OUT/logs-duration.txt"
  grep -E -q 'duration: [0-9.]+ ms  statement: SELECT /\* wiki_dh12_simple_slow' \
    "$OUT/logs-duration.txt" || die "No simple-protocol duration line"
  grep -E -q 'duration: [0-9.]+ ms  execute <unnamed>: SELECT /\* wiki_dh12_extended_slow' \
    "$OUT/logs-duration.txt" || die "No extended-protocol duration line"
  record "logs.duration: $(sed 's/^[^]]*] //' "$OUT/logs-duration.txt" | tr '\n' ';')"
  m=$(mark "$OUT/primary.log"); ms=$(mark "$OUT/standby.log")
  q "$PPORT" "$SU" <<'SQL' || die "WAL burst failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_wal;
CREATE /* wiki_dh12_disposable */ TABLE dh_wal (id integer, pad text);
CHECKPOINT /* wiki_dh12_manual */;
INSERT /* wiki_dh12_wal_burst */ INTO dh_wal SELECT g, repeat('w', 400) FROM generate_series(1, 150000) AS g;
SQL
  wait_for 60 "a WAL-caused checkpoint" has_since "$OUT/primary.log" "$m" \
    'LOG:  checkpoint starting: wal'
  wait_for 60 "the checkpoint frequency message" has_since "$OUT/primary.log" "$m" \
    'LOG:  checkpoints are occurring too frequently'
  wait_for 120 "a standby restartpoint" has_since "$OUT/standby.log" "$ms" \
    'LOG:  restartpoint starting:'
  since "$OUT/primary.log" "$m" \
    | grep -E 'checkpoint starting:|checkpoints are occurring too frequently|HINT:  Consider increasing the configuration parameter "max_wal_size"' \
    > "$OUT/logs-checkpoint.txt"
  since "$OUT/standby.log" "$ms" \
    | grep -E 'restartpoint starting:|recovery restart point at|too frequently' \
    > "$OUT/logs-restartpoint.txt"
  if grep -q 'too frequently' "$OUT/logs-restartpoint.txt"; then
    die "The standby logged the checkpoint frequency message"
  fi
  printf '%s\n' "DROP /* wiki_dh12_disposable */ TABLE dh_wal;" | q "$PPORT" "$SU" \
    || die "Drop failed"
  record "logs.checkpoint_primary: $(sed 's/^[^]]*] //' "$OUT/logs-checkpoint.txt" | sort | uniq -c | tr -s ' ' | tr '\n' ';')"
  record "logs.checkpoint_standby: $(sed 's/^[^]]*] //' "$OUT/logs-restartpoint.txt" | sed 's/at [0-9A-F]*\/[0-9A-F]*/at X\/X/' | sort | uniq -c | tr -s ' ' | tr '\n' ';')"
}

holder_ready() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_holder_wait */ count(*) FROM pg_stat_activity WHERE query LIKE '%wiki_dh12_snapshot_holder%pg_sleep%' AND state = 'active' AND backend_xmin IS NOT NULL AND pid <> pg_backend_pid();")" = 1
}
av_summary() {
  # $1 log mark, $2 table, $3 pattern that the tuples line must match
  local ln
  since "$OUT/primary.log" "$1" > "$OUT/av-window.txt"
  for ln in $(grep -n "automatic vacuum of table \"postgres.public.$2\"" "$OUT/av-window.txt" | cut -d: -f1); do
    if sed -n "$((ln + 2))p" "$OUT/av-window.txt" | grep -E -q -- "$3"; then
      sed -n "$ln,$((ln + 2))p" "$OUT/av-window.txt" > "$OUT/av-$2.txt"
      return 0
    fi
  done
  return 1
}
avc_running() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_avc_wait */ count(*) FROM pg_stat_activity WHERE backend_type = 'autovacuum worker' AND query LIKE 'autovacuum: VACUUM% public.dh_avc%';")" = 1
}
avc_cancelled() {
  local ln
  since "$OUT/primary.log" "$1" > "$OUT/avc-window.txt"
  for ln in $(grep -n 'ERROR:  canceling autovacuum task' "$OUT/avc-window.txt" | cut -d: -f1); do
    if sed -n "$((ln + 1))p" "$OUT/avc-window.txt" \
        | grep -q 'CONTEXT:  automatic vacuum of table "postgres.public.dh_avc"'; then
      sed -n "$ln,$((ln + 1))p" "$OUT/avc-window.txt" > "$OUT/avc-cancel.txt"
      return 0
    fi
  done
  return 1
}
autovac() {
  need; reset_results autovac
  running "$PDATA" || die "Primary is not running"
  q "$PPORT" "$SU" <<'SQL' || die "Autovacuum fixture failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_av;
CREATE /* wiki_dh12_disposable */ TABLE dh_av (id integer, pad text)
  WITH (autovacuum_vacuum_threshold = 0, autovacuum_vacuum_scale_factor = 0);
INSERT /* wiki_dh12_disposable */ INTO dh_av SELECT g, repeat('a', 40) FROM generate_series(1, 2000) AS g;
SQL
  { printf '%s\n' "$GUARD" "BEGIN /* wiki_dh12_snapshot_holder */ ISOLATION LEVEL REPEATABLE READ;" \
      "SELECT /* wiki_dh12_snapshot_holder */ count(*) FROM dh_av;" \
      "SELECT /* wiki_dh12_snapshot_holder */ pg_sleep(90);" "COMMIT /* wiki_dh12_snapshot_holder */;"
  } | psql_as "$PPORT" "$SU" -q > "$OUT/autovac-holder.txt" 2>&1 &
  holder=$!
  wait_for 30 "the snapshot holder" holder_ready
  m=$(mark "$OUT/primary.log")
  printf '%s\n' "DELETE /* wiki_dh12_disposable */ FROM dh_av WHERE id % 2 = 0;" \
    | q "$PPORT" "$SU" || die "Delete failed"
  wait_for 60 "an autovacuum summary with unremovable tuples" av_summary "$m" dh_av \
    ' [1-9][0-9]* are dead but not yet removable'
  cp "$OUT/av-dh_av.txt" "$OUT/autovac-held.txt"
  val "$PPORT" "$SU" "SELECT /* wiki_dh12_holder_cancel */ pg_cancel_backend(pid) FROM pg_stat_activity WHERE query LIKE '%wiki_dh12_snapshot_holder%pg_sleep%' AND pid <> pg_backend_pid();" \
    > /dev/null || die "Cancel failed"
  if wait "$holder"; then die "The snapshot holder was not cancelled"; fi
  m2=$(mark "$OUT/primary.log")
  wait_for 60 "an autovacuum summary that removes the tuples" av_summary "$m2" dh_av \
    'tuples: [1-9][0-9]* removed'
  cp "$OUT/av-dh_av.txt" "$OUT/autovac-released.txt"
  record "autovac.held: $(sed 's/^[^]]*] //' "$OUT/autovac-held.txt" | tr -s '\t ' ' ' | tr '\n' ';')"
  record "autovac.released: $(sed 's/^[^]]*] //' "$OUT/autovac-released.txt" | tr -s '\t ' ' ' | tr '\n' ';')"
  printf '%s\n' "DROP /* wiki_dh12_disposable */ TABLE dh_av;" | q "$PPORT" "$SU" \
    || die "Drop failed"
  q "$PPORT" "$SU" <<'SQL' || die "Autovacuum cancel fixture failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_avc;
CREATE /* wiki_dh12_disposable */ TABLE dh_avc (id integer, pad text)
  WITH (autovacuum_vacuum_threshold = 0, autovacuum_vacuum_scale_factor = 0,
        autovacuum_vacuum_cost_delay = 20, autovacuum_vacuum_cost_limit = 1);
INSERT /* wiki_dh12_disposable */ INTO dh_avc SELECT g, repeat('c', 100) FROM generate_series(1, 20000) AS g;
DELETE /* wiki_dh12_disposable */ FROM dh_avc WHERE id % 2 = 0;
SQL
  wait_for 60 "an autovacuum worker on dh_avc" avc_running
  m=$(mark "$OUT/primary.log")
  q "$PPORT" "$SU" <<'SQL' || die "Conflicting LOCK failed"
SET /* wiki_dh12_av_conflict */ lock_timeout = '30s';
BEGIN /* wiki_dh12_av_conflict */;
LOCK /* wiki_dh12_av_conflict */ TABLE dh_avc IN SHARE MODE;
COMMIT /* wiki_dh12_av_conflict */;
SQL
  wait_for 15 "the autovacuum cancellation" avc_cancelled "$m"
  since "$OUT/primary.log" "$m" | grep -E 'still waiting for ShareLock on relation|acquired ShareLock on relation|sending cancel' \
    > "$OUT/avc-waiter.txt"
  record "autovac.cancel: $(sed 's/^[^]]*] //' "$OUT/avc-cancel.txt" | tr '\n' ';')"
  record "autovac.waiter: $(sed 's/^[^]]*] //' "$OUT/avc-waiter.txt" | tr '\n' ';')"
  printf '%s\n' "DROP /* wiki_dh12_disposable */ TABLE dh_avc;" | q "$PPORT" "$SU" \
    || die "Drop failed"
}

rc_replayed() { test "$(val "$SPORT" "$SU" "SELECT /* wiki_dh12_rc_replay */ count(*) FROM dh_rc;")" = 1000; }
rc_sessions() {
  test "$(val "$SPORT" "$SU" "SELECT /* wiki_dh12_rc_wait */ count(*) FROM pg_stat_activity WHERE backend_xmin IS NOT NULL AND ((state = 'active' AND query LIKE '%wiki_dh12_rc_active%pg_sleep%') OR (state = 'idle in transaction' AND query LIKE '%wiki_dh12_rc_idle%')) AND pid <> pg_backend_pid();")" = 2
}
rc_logged() {
  has_since "$OUT/standby.log" "$1" 'ERROR:  canceling statement due to conflict with recovery' \
    && has_since "$OUT/standby.log" "$1" 'FATAL:  terminating connection due to conflict with recovery'
}
rc_count() { val "$SPORT" "$SU" "SELECT /* wiki_dh12_rc_count */ confl_snapshot FROM pg_stat_database_conflicts WHERE datname = current_database();"; }
rc_reached() { test "$(rc_count)" -ge "$1"; }
standby_ready() { test "$(val "$SPORT" "$SU" "SELECT /* wiki_dh12_standby_ready */ pg_is_in_recovery();")" = t; }
standby_reset_known() {
  test "$(val "$SPORT" "$SU" "SELECT /* wiki_dh12_standby_reset */ stats_reset IS NOT NULL FROM pg_stat_database WHERE datname = current_database();")" = t
}
conflict() {
  need; reset_results conflict
  running "$PDATA" || die "Primary is not running"
  running "$SDATA" || die "Standby is not running"
  q "$PPORT" "$SU" <<'SQL' || die "Conflict fixture failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_rc;
CREATE /* wiki_dh12_disposable */ TABLE dh_rc (id integer, pad text) WITH (autovacuum_enabled = false);
INSERT /* wiki_dh12_disposable */ INTO dh_rc SELECT g, repeat('r', 40) FROM generate_series(1, 1000) AS g;
SQL
  wait_for 60 "replay of dh_rc on the standby" rc_replayed
  c0=$(rc_count) || die "Counter read failed"
  ms=$(mark "$OUT/standby.log")
  { printf '%s\n' "$GUARD" "BEGIN /* wiki_dh12_rc_active */ ISOLATION LEVEL REPEATABLE READ;" \
      "SELECT /* wiki_dh12_rc_active */ count(*) FROM dh_rc;" \
      "SELECT /* wiki_dh12_rc_active */ pg_sleep(80);" "COMMIT /* wiki_dh12_rc_active */;"
  } | psql_as "$SPORT" "$SU" -q > "$OUT/conflict-active.txt" 2>&1 &
  s1=$!
  rm -f "$OUT/rc-release"
  { printf '%s\n' "$GUARD" "BEGIN /* wiki_dh12_rc_idle */ ISOLATION LEVEL REPEATABLE READ;" \
      "SELECT /* wiki_dh12_rc_idle */ count(*) FROM dh_rc;"
    while test ! -f "$OUT/rc-release"; do sleep 1; done
    printf '%s\n' "COMMIT /* wiki_dh12_rc_idle */;"
  } | psql_as "$SPORT" "$SU" -q > "$OUT/conflict-idle.txt" 2>&1 &
  s2=$!
  wait_for 30 "both standby snapshot sessions" rc_sessions
  vac_t0=$SECONDS
  q "$PPORT" "$SU" <<'SQL' || die "Primary cleanup failed"
DELETE /* wiki_dh12_disposable */ FROM dh_rc WHERE id % 2 = 0;
VACUUM /* wiki_dh12_rc_cleanup */ dh_rc;
SQL
  wait_for 90 "both recovery-conflict messages" rc_logged "$ms"
  elapsed=$((SECONDS - vac_t0))
  : > "$OUT/rc-release"
  if wait "$s1"; then die "The active standby session was not cancelled"; fi
  if wait "$s2"; then die "The idle standby session was not terminated"; fi
  wait_for 15 "the confl_snapshot counter" rc_reached $((c0 + 2))
  c1=$(rc_count) || die "Counter read failed"
  since "$OUT/standby.log" "$ms" \
    | grep -E 'conflict with recovery|User query might have needed|In a moment you should be able to reconnect' \
    > "$OUT/conflict-log.txt"
  record "conflict.confl_snapshot: before=$c0 after=$c1 seconds_from_vacuum_to_both_messages=$elapsed"
  record "conflict.log: $(sed 's/^[^]]*] //' "$OUT/conflict-log.txt" | tr '\n' ';')"
  r0=$(val "$SPORT" "$SU" "SELECT /* wiki_dh12_reset_before */ stats_reset FROM pg_stat_database WHERE datname = current_database();") \
    || die "Reset read failed"
  stop_one "$SDATA" "$SPORT" || die "Standby teardown failed"
  start_cluster "$SDATA" "$SPORT" "$OUT/standby.log"
  wait_for 60 "the restarted standby" standby_ready
  c2=$(rc_count) || die "Counter read failed"
  wait_for 30 "a new standby stats_reset" standby_reset_known
  r1=$(val "$SPORT" "$SU" "SELECT /* wiki_dh12_reset_after */ stats_reset FROM pg_stat_database WHERE datname = current_database();") \
    || die "Reset read failed"
  record "conflict.after_clean_standby_restart: confl_snapshot=$c2 stats_reset_before=$r0 stats_reset_after=$r1"
  wait_for 60 "streaming replication" streaming
}

dead_counted() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_crash_wait */ n_dead_tup FROM pg_stat_user_tables WHERE relname = 'dh_crash';")" = "$1"
}
db_reset_known() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_crash_reset */ stats_reset IS NOT NULL FROM pg_stat_database WHERE datname = current_database();")" = t
}
primary_up() { test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_up */ 1;")" = 1; }
victim_ready() {
  test "$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_victim_wait */ count(*) FROM pg_stat_activity WHERE query LIKE '%wiki_dh12_crash_victim%pg_sleep%' AND state = 'active' AND pid <> pg_backend_pid();")" = 1
}
crash_state() {
  val "$PPORT" "$SU" "SELECT /* wiki_dh12_crash_state */ 'n_live_tup=' || n_live_tup || ' n_dead_tup=' || n_dead_tup || ' db_stats_reset=' || coalesce((SELECT stats_reset::text FROM pg_stat_database WHERE datname = current_database()), 'NULL') || ' bgwriter_stats_reset=' || (SELECT stats_reset FROM pg_stat_bgwriter) || ' pgss_backend_marker_rows=' || (SELECT count(*) FROM pg_stat_statements WHERE query LIKE '%wiki_dh12_pgss_backend%') || ' pgss_immediate_marker_rows=' || (SELECT count(*) FROM pg_stat_statements WHERE query LIKE '%wiki_dh12_pgss_immediate%') FROM pg_stat_user_tables WHERE relname = 'dh_crash';"
}
crash() {
  need; reset_results crash
  running "$PDATA" || die "Primary is not running"
  q "$PPORT" "$SU" > /dev/null <<'SQL' || die "Crash fixture failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_crash;
CREATE /* wiki_dh12_disposable */ TABLE dh_crash (id integer) WITH (autovacuum_enabled = false);
INSERT /* wiki_dh12_disposable */ INTO dh_crash SELECT generate_series(1, 1000);
DELETE /* wiki_dh12_disposable */ FROM dh_crash WHERE id <= 500;
SELECT /* wiki_dh12_pgss_backend */ count(*) FROM dh_crash WHERE id > 900;
SQL
  wait_for 30 "the dead-tuple counter" dead_counted 500
  record "crash.backend_before: $(crash_state)"
  # Leg 1, disposable crash: SIGKILL one backend of the sandbox primary.
  m=$(mark "$OUT/primary.log")
  { printf '%s\n' "$GUARD" "SELECT /* wiki_dh12_crash_victim */ pg_sleep(60);"
  } | psql_as "$PPORT" "$SU" -q > "$OUT/crash-victim.txt" 2>&1 &
  victim=$!
  wait_for 30 "the crash victim session" victim_ready
  vpid=$(val "$PPORT" "$SU" "SELECT /* wiki_dh12_victim_pid */ pid FROM pg_stat_activity WHERE query LIKE '%wiki_dh12_crash_victim%pg_sleep%' AND pid <> pg_backend_pid();") \
    || die "Victim lookup failed"
  kill -s KILL "$vpid" || die "kill failed"
  if wait "$victim"; then die "The killed session exited cleanly"; fi
  wait_for 60 "reinitialization after the backend crash" has_since "$OUT/primary.log" "$m" \
    'all server processes terminated; reinitializing'
  wait_for 60 "the primary after crash recovery" primary_up
  wait_for 30 "a new database stats_reset" db_reset_known
  record "crash.backend_after: $(crash_state)"
  since "$OUT/primary.log" "$m" \
    | grep -E 'was terminated by signal|terminating any other active server processes|all server processes terminated|database system was interrupted|not properly shut down|redo starts at|redo done at|ready to accept connections' \
    > "$OUT/crash-backend-log.txt"
  grep -q 'not properly shut down; automatic recovery in progress' "$OUT/crash-backend-log.txt" \
    || die "No crash-recovery message after the backend crash"
  record "crash.backend_log: $(sed 's/^[^]]*] //' "$OUT/crash-backend-log.txt" | sed 's/[0-9A-F]*\/[0-9A-F]*/X\/X/g; s/up at .*/up at T/; s/(PID [0-9]*)/(PID N)/' | tr '\n' ';')"
  # Leg 2: immediate shutdown of the sandbox primary.
  q "$PPORT" "$SU" > /dev/null <<'SQL' || die "Immediate-leg fixture failed"
DELETE /* wiki_dh12_disposable */ FROM dh_crash WHERE id <= 750;
SELECT /* wiki_dh12_pgss_immediate */ count(*) FROM dh_crash WHERE id < 100;
SQL
  wait_for 30 "the dead-tuple counter" dead_counted 250
  record "crash.immediate_before: $(crash_state)"
  m=$(mark "$OUT/primary.log")
  "$INSTALL/bin/pg_ctl" -D "$PDATA" -m immediate -w -t 120 stop > /dev/null || die "Immediate stop failed"
  start_cluster "$PDATA" "$PPORT" "$OUT/primary.log"
  wait_for 30 "a new database stats_reset" db_reset_known
  record "crash.immediate_after: $(crash_state)"
  since "$OUT/primary.log" "$m" \
    | grep -E 'received immediate shutdown request|abnormal database system shutdown|database system was interrupted|not properly shut down|redo starts at|redo done at|ready to accept connections' \
    > "$OUT/crash-immediate-log.txt"
  grep -q 'not properly shut down; automatic recovery in progress' "$OUT/crash-immediate-log.txt" \
    || die "No crash-recovery message after the immediate shutdown"
  record "crash.immediate_log: $(sed 's/^[^]]*] //' "$OUT/crash-immediate-log.txt" | sed 's/[0-9A-F]*\/[0-9A-F]*/X\/X/g; s/up at .*/up at T/' | tr '\n' ';')"
  printf '%s\n' "DROP /* wiki_dh12_disposable */ TABLE dh_crash;" | q "$PPORT" "$SU" \
    || die "Drop failed"
  wait_for 60 "streaming replication after the crash legs" streaming
}

ck_counted() {
  test "$(val "$CPORT" "$SU" "SELECT /* wiki_dh12_ck_wait */ checksum_failures FROM pg_stat_database WHERE datname = current_database();")" -ge 3
}
checksum() {
  need; reset_results checksum
  stop_one "$CDATA" "$CPORT" || die "Checksum cluster teardown failed"
  rm -rf "$CDATA" "$OUT/ck.log"
  "$INSTALL/bin/initdb" -D "$CDATA" -A trust -U "$SU" --locale=C --encoding=UTF8 -k \
    > "$OUT/initdb-ck.log" 2>&1 || die "initdb -k failed"
  start_cluster "$CDATA" "$CPORT" "$OUT/ck.log"
  paths=$(q "$CPORT" "$SU" -A -t <<'SQL'
CREATE /* wiki_dh12_disposable */ TABLE dh_ck1 (id integer, pad text);
CREATE /* wiki_dh12_disposable */ TABLE dh_ck2 (id integer, pad text);
INSERT /* wiki_dh12_disposable */ INTO dh_ck1 SELECT g, repeat('k', 20) FROM generate_series(1, 100) AS g;
INSERT /* wiki_dh12_disposable */ INTO dh_ck2 SELECT g, repeat('k', 20) FROM generate_series(1, 100) AS g;
CHECKPOINT /* wiki_dh12_flush */;
SELECT /* wiki_dh12_path */ pg_relation_filepath('dh_ck1') || ' ' || pg_relation_filepath('dh_ck2');
SQL
) || die "Checksum fixture failed"
  p1=${paths% *}; p2=${paths#* }
  "$INSTALL/bin/pg_ctl" -D "$CDATA" -m fast -w -t 120 stop > /dev/null || die "Stop failed"
  # Disposable corruption: one byte in the free space between pd_lower and pd_upper.
  for f in "$p1" "$p2"; do
    test -f "$CDATA/$f" || die "Missing relation file $f"
    printf '\125' | dd of="$CDATA/$f" bs=1 seek=1000 count=1 conv=notrunc \
      2>> "$OUT/dd-checksum.txt" || die "dd failed"
  done
  start_cluster "$CDATA" "$CPORT" "$OUT/ck.log"
  if printf '%s\n' "SELECT /* wiki_dh12_ck_default */ count(*) FROM dh_ck1;" \
      | q "$CPORT" "$SU" > "$OUT/checksum-default.txt" 2>&1; then
    die "A page with a bad checksum was read under the default settings"
  fi
  grep -q 'WARNING:  page verification failed, calculated checksum' "$OUT/checksum-default.txt" \
    || die "No checksum WARNING"
  grep -q 'ERROR:  invalid page in block 0 of relation base/' "$OUT/checksum-default.txt" \
    || die "No invalid-page ERROR"
  q "$CPORT" "$SU" -A -t > "$OUT/checksum-ignore.txt" 2>&1 <<'SQL' || die "ignore_checksum_failure leg failed"
SET /* wiki_dh12_ck_ignore */ ignore_checksum_failure = on;
SELECT /* wiki_dh12_ck_ignore */ count(*) FROM dh_ck1;
SQL
  q "$CPORT" "$SU" -A -t > "$OUT/checksum-zero.txt" 2>&1 <<'SQL' || die "zero_damaged_pages leg failed"
SET /* wiki_dh12_ck_zero */ zero_damaged_pages = on;
SELECT /* wiki_dh12_ck_zero */ count(*) FROM dh_ck2;
SQL
  grep -q '; zeroing out page' "$OUT/checksum-zero.txt" || die "No zeroing WARNING"
  wait_for 15 "the checksum counter" ck_counted
  record "checksum.default: $(sed 's/^psql:[^ ]* //' "$OUT/checksum-default.txt" | sed 's/checksum [0-9]* but expected [0-9]*/checksum N but expected M/' | tr '\n' ';')"
  record "checksum.ignore_checksum_failure_on: $(sed 's/^psql:[^ ]* //' "$OUT/checksum-ignore.txt" | sed 's/checksum [0-9]* but expected [0-9]*/checksum N but expected M/' | tr '\n' ';')"
  record "checksum.zero_damaged_pages_on: $(sed 's/^psql:[^ ]* //' "$OUT/checksum-zero.txt" | sed 's/checksum [0-9]* but expected [0-9]*/checksum N but expected M/' | tr '\n' ';')"
  record "checksum.counter: $(val "$CPORT" "$SU" "SELECT /* wiki_dh12_ck_counter */ 'checksum_failures=' || checksum_failures || ' last_failure_set=' || (checksum_last_failure IS NOT NULL) FROM pg_stat_database WHERE datname = current_database();")"
  stop_one "$CDATA" "$CPORT" || die "Checksum cluster teardown failed"
}

sql_replayed() { test "$(val "$SPORT" "$SU" "SELECT /* wiki_dh12_sql_replay */ count(*) FROM dh_sql_t;")" = 1000; }
sqlcheck() {
  need; reset_results sqlcheck
  running "$PDATA" || die "Primary is not running"
  running "$SDATA" || die "Standby is not running"
  roles
  q "$PPORT" "$SU" <<'SQL' || die "SQL-check fixture failed"
DROP /* wiki_dh12_disposable */ TABLE IF EXISTS dh_sql_t;
CREATE /* wiki_dh12_disposable */ TABLE dh_sql_t (id integer PRIMARY KEY, pad text);
INSERT /* wiki_dh12_disposable */ INTO dh_sql_t SELECT g, repeat('s', 30) FROM generate_series(1, 1000) AS g;
SQL
  wait_for 60 "streaming replication" streaming
  wait_for 60 "replay of the SQL-check fixture" sql_replayed
  fence=$(printf '\140\140\140')
  rm -f "$OUT"/block-*.sql "$OUT"/sqlcheck-*.txt
  n=0; inblock=0
  while IFS= read -r line; do
    if test "$inblock" -eq 0 && test "$line" = "${fence}sql"; then
      n=$((n + 1)); inblock=1; : > "$OUT/block-$n.sql"; continue
    fi
    if test "$inblock" -eq 1 && test "$line" = "$fence"; then inblock=0; continue; fi
    if test "$inblock" -eq 1; then
      printf '%s\n' "$line" | sed -e 's/schema\.table_name/public.dh_sql_t/' \
        -e 's/schema\.index_name/public.dh_sql_t_pkey/' >> "$OUT/block-$n.sql"
    fi
  done < "$PAGE"
  test "$n" -gt 0 || die "No SQL blocks found in the page"
  i=1
  while test "$i" -le "$n"; do
    q "$PPORT" "$SU" < "$OUT/block-$i.sql" > "$OUT/sqlcheck-primary-su-$i.txt" 2>&1 \
      || die "Block $i failed on the primary as superuser"
    q "$PPORT" dh_monitor < "$OUT/block-$i.sql" > "$OUT/sqlcheck-primary-monitor-$i.txt" 2>&1 \
      || die "Block $i failed on the primary as pg_monitor"
    q "$SPORT" "$SU" < "$OUT/block-$i.sql" > "$OUT/sqlcheck-standby-su-$i.txt" 2>&1 \
      || die "Block $i failed on the standby as superuser"
    i=$((i + 1))
  done
  sum=$(i=1; while test "$i" -le "$n"; do cat "$OUT/block-$i.sql"; i=$((i + 1)); done | cksum)
  record "sqlcheck.blocks: $n blocks ran without error on the primary as superuser and as pg_monitor, and on the standby as superuser; cksum=$sum"
  printf '%s\n' "DROP /* wiki_dh12_disposable */ TABLE dh_sql_t;" | q "$PPORT" "$SU" \
    || die "Drop failed"
}

report() {
  need
  { printf 'utc=%s\n' "$(date -u '+%Y-%m-%dT%H:%M:%SZ')"
    printf 'os=%s arch=%s\n' "$(uname -s)" "$(uname -m)"
    printf 'pin=%s\n' "$PIN"
    printf 'cc=%s\n' "$(cc --version 2>&1 | head -n 1)"
    printf 'configure=%s\n' "$("$INSTALL/bin/pg_config" --configure)"
    if running "$PDATA"; then
      val "$PPORT" "$SU" "SELECT /* wiki_dh12_platform */ version() || ' | block_size=' || current_setting('block_size') || ' | max_data_alignment=' || (SELECT max_data_alignment FROM pg_control_init()) || ' | data_checksums=' || current_setting('data_checksums') || ' | lc_collate=' || current_setting('lc_collate') || ' | server_encoding=' || current_setting('server_encoding');"
    fi
  } > "$OUT/platform.txt" || die "Platform capture failed"
  cat "$OUT/platform.txt"
  if test -f "$OUT/results.txt"; then cat "$OUT/results.txt"; fi
}

clean() {
  if test ! -e "$RUN"; then return 0; fi
  stop_all || die "Teardown checks failed; leaving $RUN for inspection"
  rm -rf "$RUN" || die "Sandbox removal failed"
  test ! -e "$RUN" || die "Sandbox remains"
}

if test "$#" -eq 0; then
  set -- build default cluster standby access prepared deadlock logs autovac conflict crash \
    checksum sqlcheck report clean
fi
for stage in "$@"; do
  stage_t0=$SECONDS
  case "$stage" in
    build) build ;; default) default_leg ;; cluster) cluster ;; standby) standby ;;
    access) access ;; prepared) prepared ;; deadlock) deadlock ;; logs) logs ;;
    autovac) autovac ;; conflict) conflict ;; crash) crash ;; checksum) checksum ;;
    sqlcheck) sqlcheck ;; report) report ;;
    stop) need; stop_all || die "Stop failed" ;; clean) clean ;;
    *) die "Unknown stage: $stage" ;;
  esac
  printf 'stage=%s seconds=%s\n' "$stage" "$((SECONDS - stage_t0))"
done
```

## Context Reviewed

- Monitoring access: activity, progress, settings, WAL sender, WAL receiver,
  `pg_stat_statements`, file-listing functions, `pgstattuple`, and
  `pg_subscription` column grants, plus the executor's column-privilege check.
- PostgreSQL 12 logging destinations, severity filters, the statistics
  collector and its snapshot, reset, startup, exit, and recovery paths, and the
  logging, statistics, autovacuum, and replication settings with their
  `GucContext` definitions, boot values, and reload and source-priority rules.
- The built-in `pg_stat_activity`, `pg_locks`, `pg_stat_database`,
  `pg_stat_database_conflicts`, `pg_stat_user_tables`, `pg_stat_sys_tables`,
  `pg_stat_progress_vacuum`, `pg_stat_bgwriter`, `pg_stat_archiver`,
  `pg_stat_replication`, `pg_stat_wal_receiver`, `pg_stat_subscription`, and
  `pg_replication_slots` definitions, their implementation callers, and their
  redaction boundaries.
- Message paths for connection exhaustion, authentication, prepared locks,
  lock waits, deadlocks, timeouts, recovery conflicts (ERROR and FATAL),
  temp files, simple and extended-protocol durations, checkpoints and
  restartpoints, archiving, replication disconnects and WAL removal, backend
  crashes and reinitialization, immediate shutdown, WAL I/O failures, disk
  extension, lock-table exhaustion, the statistics collector, and page
  verification with both damaged-page settings.
- Autovacuum table, TOAST, temporary-table, and orphan selection, threshold
  decisions, lock-conflict cancellation, the completion summary, XID and
  MultiXact anti-wraparound behavior, effective member-space limits, and the
  warning and shutdown paths.
- PostgreSQL 12 `pg_stat_statements` and `pgstattuple` documentation and
  implementation: query identity, dump and reload across shutdowns, access,
  scan behavior, and installation scripts.
- Generated-catalog inputs, installed system-view definitions, rules output,
  statistics and prepared-transaction regression tests, and recovery and
  subscription TAP tests relevant to the checklist.

The view shapes are handwritten in `system_views.sql` and installed by
`initdb`. Built-in signatures such as `age()` and `mxid_age()` come from
`pg_proc.dat`, which the catalog build turns into bootstrap and generated
catalog artifacts. These diagnostic queries do not change a generated header.
Upstream regression tests cover view definitions, statistics refresh, and
prepared-lock survival. Recovery, archiving, logical decoding, synchronous
replication, and subscription tests cover their facilities; there is no
upstream test for this compound checklist as a whole.
[pg_proc.dat#age-and-mxid_age](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2265-L2272)
[catalog/Makefile#catalog-generation](../../../../raw/postgres-12/src/backend/catalog/Makefile#L51-L109)
[initdb.c#setup_sysviews](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1651-L1671)
[rules.out#statistics-views](../../../../raw/postgres-12/src/test/regress/expected/rules.out#L1760-L1805)
[stats.sql#wait_for_stats](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L27-L78)
[prepared_xacts.sql#view-and-lock-tests](../../../../raw/postgres-12/src/test/regress/sql/prepared_xacts.sql#L110-L153)
[001_stream_rep.pl#streaming-test](../../../../raw/postgres-12/src/test/recovery/t/001_stream_rep.pl#L1-L20)
[002_archiving.pl#archiving-test](../../../../raw/postgres-12/src/test/recovery/t/002_archiving.pl#L1-L20)
[006_logical_decoding.pl#logical-decoding-test](../../../../raw/postgres-12/src/test/recovery/t/006_logical_decoding.pl#L1-L27)
[007_sync_rep.pl#synchronous-replication-test](../../../../raw/postgres-12/src/test/recovery/t/007_sync_rep.pl#L1-L20)
[004_sync.pl#subscription-sync-test](../../../../raw/postgres-12/src/test/subscription/t/004_sync.pl#L1-L20)

## Evidence Map

| Claim area | Primary evidence |
|---|---|
| Monitoring access and setting visibility | [user-manag.sgml#default-roles](../../../../raw/postgres-12/doc/src/sgml/user-manag.sgml#L519-L538), [pgstatfuncs.c#activity-and-progress-access](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L490-L655), [pgstatfuncs.c#activity-redaction](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L885-L911), [guc.c#pg_settings-access](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L8998-L9008), [system_views.sql#pg_subscription-column-grants](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1059-L1062), [execMain.c#column-privileges](../../../../raw/postgres-12/src/backend/executor/execMain.c#L676-L692) |
| Counter epochs, resets, and recovery wipes | [pgstat.c#pgstat_recv_resetcounter](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L6097-L6122), [pgstat.c#collector-exit-write](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L4674-L4680), [xlog.c#recovery-decision](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6733-L6746), [xlog.c#pgstat_reset_all-call](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6843-L6846), [postmaster.c#immediate-shutdown-not-fatal](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3618-L3619), [pg_stat_statements.c#pgss_shmem_shutdown](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L692-L702) |
| Logging destinations, filters, defaults, and apply scopes | [config.sgml#log_destination](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5439-L5486), [config.sgml#server-log-levels](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5877-L5927), [guc.c#GucContext_Names](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L594-L603), [guc.c#source-priority](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6925-L6941) |
| Connections, prepared locks, deadlocks, timeouts, and recovery conflicts | [postinit.c#InitializeMaxBackends](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L527-L539), [func.sgml#pg_blocking_pids](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L17584-L17604), [twophase.c#feature-disabled](../../../../raw/postgres-12/src/backend/access/transam/twophase.c#L385-L390), [proc.c#lock-wait-log](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1377-L1497), [postgres.c#RecoveryConflictInterrupt](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L2866-L2915), [postgres.c#ProcessInterrupts](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L2990-L3136) |
| Vacuum, analyze, TOAST, catalogs, partitions, and wraparound | [autovacuum.c#do_autovacuum-table-and-TOAST-passes](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2035-L2191), [autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2920-L3096), [proc.c#autovacuum-cancel](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1375), [vacuumlazy.c#autovacuum-summary](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L380-L441), [analyze.c#analyze_rel](../../../../raw/postgres-12/src/backend/commands/analyze.c#L231-L268), [varsup.c#GetNewTransactionId-wraparound-limits](../../../../raw/postgres-12/src/backend/access/transam/varsup.c#L118-L158), [multixact.c#MultiXactMemberFreezeThreshold](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L2785-L2845) |
| Checkpoints, restartpoints, and background writes | [bufmgr.c#BgBufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2260-L2300), [checkpointer.c#ForwardSyncRequest](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L1086-L1159), [checkpointer.c#checkpoint-warning](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462), [xlog.c#LogCheckpointStart](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8356-L8372), [xlog.c#restartpoint-request](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L11564-L11572) |
| WAL archiving, replication, subscriptions, and slots | [pgarch.c#archive-command-result](../../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L466-L680), [system_views.sql#replication-views](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L758-L816), [slot.c#ComputeRequiredXmin-and-LSN](../../../../raw/postgres-12/src/backend/replication/slot.c#L695-L776), [xlog.c#KeepLogSeg](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L9307-L9349), [monitoring.sgml#replication-lag](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L1978-L2020) |
| Query and relation extensions | [pg_stat_statements.c#module-GUCs](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L365-L410), [pgstatstatements.sgml#query-merging](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L236-L255), [pgstattuple.sgml#pgstattuple_approx](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L484-L538), [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L315) |
| Crashes, checksums, and storage errors | [postmaster.c#reinitialize-after-crash](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3911-L3928), [xlog.c#WAL-fsync-failures](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L10093-L10126), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L161), [bufmgr.c#invalid-page-handling](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925), [md.c#mdextend](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L398-L418) |
| Tests and build/generated boundary | [stats.sql#wait_for_stats](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L27-L78), [prepared_xacts.sql#view-and-lock-tests](../../../../raw/postgres-12/src/test/regress/sql/prepared_xacts.sql#L110-L153), [catalog/Makefile#catalog-generation](../../../../raw/postgres-12/src/backend/catalog/Makefile#L51-L109), [initdb.c#setup_sysviews](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1651-L1671) |

## Open Questions

- Numeric alert thresholds for sessions, transaction age, dead rows, temp
  bytes, checkpoint rates, and replication lag depend on workload capacity and
  service objectives. The pinned source defines the counters and safety
  limits, not a universal healthy threshold.
- Which standbys, cascading downstreams, subscriptions, slots, archive
  destinations, synchronous standbys, and recovery objectives are expected? An
  empty component view is healthy when that component is absent and an outage
  when it is required.
- What measurement window and reset policy applies to cumulative counters and
  `pg_stat_statements`? PostgreSQL 12 attaches no reset timestamp to a table or
  statement row, so the capture system must keep its own epoch and watch the
  log for crash recovery and standby restarts.
- PostgreSQL's internal views do not replace host monitoring for free space,
  inodes, filesystem or device errors, CPU, memory pressure, and network
  health. The source sends relation-extension failures to the log with a
  free-space hint.
  [md.c#mdextend](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L404-L418)
- What external evidence proves archive-object existence and durability,
  backup freshness, and successful restore? `pg_stat_archiver` reports command
  exit status and cannot prove those outcomes.
  [config.sgml#archive-command-success](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L3110-L3133)
- The default-roles documentation describes `pg_read_all_stats` as reading all
  `pg_stat_*` views, but this pin's progress function tests only
  `has_privs_of_role(GetUserId(), <reporting role>)` and never checks
  `pg_read_all_stats`, unlike `pg_stat_get_activity()`, which accepts either.
  The page follows the implementation, and the script measured it.
  [user-manag.sgml#pg_read_all_stats](../../../../raw/postgres-12/doc/src/sgml/user-manag.sgml#L524-L527)
  [pgstatfuncs.c#progress-access](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L516-L532)
  [pgstatfuncs.c#activity-access](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L653-L656)
- The PostgreSQL 12 documentation sizes the lock table with
  `max_connections + max_prepared_transactions`, while `NLOCKENTS()` uses
  `MaxBackends + max_prepared_xacts`. The page follows the implementation.
  [config.sgml#max_locks_per_transaction](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L8615-L8636)
  [lock.c#NLOCKENTS](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L53-L57)
- The documentation describes `pg_stat_database.checksum_failures` without a
  base-backup exception, but this pin reports a file's base-backup failures to
  the collector only when that file has more than one mismatch. A file with
  exactly one mismatch still makes the base backup fail with
  `checksum verification failure during base backup`, but does not increment
  the counter. Treat the log and the backup result as authoritative for that
  case. This path was not measured.
  [monitoring.sgml#checksum_failures](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2599-L2611)
  [basebackup.c#per-file-checksum-report](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1611-L1622)
  [basebackup.c#backup-checksum-failure](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L621-L635)
- The documentation for `pg_stat_database.numbackends` says it is NULL for the
  shared-objects row, but the view returns 0 for it. No check on this page
  reads that row.
  [monitoring.sgml#pg_stat_database](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2510-L2517)
  [system_views.sql#pg_stat_database](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L863)
- The standby logged `restartpoint starting: time` 15 s after each start,
  well before `checkpoint_timeout`. That spacing matches the checkpointer's
  rule of retrying a restartpoint it could not perform 15 s later with the time
  flag, but the request behind the first attempt was not traced. The page
  relies only on the absence of the frequency line on the standby, which the
  source guarantees.
  [checkpointer.c#restartpoint-retry](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L502-L520)
  [checkpointer.c#time-driven-flag](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L396-L410)

## Source References

- [analyze.c](../../../../raw/postgres-12/src/backend/commands/analyze.c#L231-L268)
- [auth.c](../../../../raw/postgres-12/src/backend/libpq/auth.c#L255-L323)
- [autovacuum.c](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L2920-L3096)
- [basebackup.c](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1543-L1622)
- [be-secure-openssl.c](../../../../raw/postgres-12/src/backend/libpq/be-secure-openssl.c#L440-L459)
- [bufmgr.c](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L925)
- [bufpage.c](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L161)
- [catalog/Makefile](../../../../raw/postgres-12/src/backend/catalog/Makefile#L51-L109)
- [catalogs.sgml](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L9221-L9345)
- [checkpointer.c](../../../../raw/postgres-12/src/backend/postmaster/checkpointer.c#L447-L462)
- [config.sgml](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L5439-L7033)
- [deadlock.c](../../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L1083-L1147)
- [execMain.c](../../../../raw/postgres-12/src/backend/executor/execMain.c#L571-L692)
- [fd.c](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L1272-L1287)
- [func.sgml](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L17584-L17604)
- [guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L594-L603)
- [high-availability.sgml](../../../../raw/postgres-12/doc/src/sgml/high-availability.sgml#L914-L930)
- [indexing.h](../../../../raw/postgres-12/src/include/catalog/indexing.h#L360-L363)
- [initdb.c](../../../../raw/postgres-12/src/bin/initdb/initdb.c#L1651-L1671)
- [initdb.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/initdb.sgml#L212-L225)
- [lock.c](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L53-L57)
- [lockfuncs.c](../../../../raw/postgres-12/src/backend/utils/adt/lockfuncs.c#L311-L312)
- [logical-replication.sgml](../../../../raw/postgres-12/doc/src/sgml/logical-replication.sgml#L545-L573)
- [maintenance.sgml](../../../../raw/postgres-12/doc/src/sgml/maintenance.sgml#L391-L705)
- [md.c](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L398-L418)
- [monitoring.sgml](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L284-L425)
- [multixact.c](../../../../raw/postgres-12/src/backend/access/transam/multixact.c#L920-L1140)
- [mvcc.sgml](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1358-L1423)
- [pg_basebackup.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/pg_basebackup.sgml#L212-L225)
- [pg_checksums.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/pg_checksums.sgml#L38-L51)
- [pg_ctl-ref.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/pg_ctl-ref.sgml#L192-L197)
- [pg_proc.dat](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2265-L2272)
- [pg_stat_statements.c](../../../../raw/postgres-12/contrib/pg_stat_statements/pg_stat_statements.c#L1500-L1615)
- [pg_subscription.h](../../../../raw/postgres-12/src/include/catalog/pg_subscription.h#L39-L41)
- [pgarch.c](../../../../raw/postgres-12/src/backend/postmaster/pgarch.c#L430-L680)
- [pgstat.c](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L680-L690)
- [pgstatfuncs.c](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L490-L655)
- [pgstatindex.c](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L216-L315)
- [pgstatstatements.sgml](../../../../raw/postgres-12/doc/src/sgml/pgstatstatements.sgml#L10-L41)
- [pgstattuple--1.4--1.5.sql](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L6-L119)
- [pgstattuple--1.4.sql](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple--1.4.sql#L3-L20)
- [pgstattuple.control](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [pgstattuple.sgml](../../../../raw/postgres-12/doc/src/sgml/pgstattuple.sgml#L10-L23)
- [postgres.c](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L2990-L3136)
- [postinit.c](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L527-L539)
- [postmaster.c](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L3378-L3407)
- [prepared_xacts.sql](../../../../raw/postgres-12/src/test/regress/sql/prepared_xacts.sql#L110-L153)
- [proc.c](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1308-L1497)
- [procsignal.h](../../../../raw/postgres-12/src/include/storage/procsignal.h#L37-L43)
- [ref/analyze.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/analyze.sgml#L112-L121)
- [regproc.c](../../../../raw/postgres-12/src/backend/utils/adt/regproc.c#L969-L1023)
- [reloptions.c](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L107-L243)
- [rules.out](../../../../raw/postgres-12/src/test/regress/expected/rules.out#L1760-L1805)
- [slot.c](../../../../raw/postgres-12/src/backend/replication/slot.c#L695-L776)
- [standby.c](../../../../raw/postgres-12/src/backend/storage/ipc/standby.c#L154-L169)
- [stats.sql](../../../../raw/postgres-12/src/test/regress/sql/stats.sql#L27-L78)
- [system_views.sql](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L887)
- [twophase.c](../../../../raw/postgres-12/src/backend/access/transam/twophase.c#L385-L390)
- [user-manag.sgml](../../../../raw/postgres-12/doc/src/sgml/user-manag.sgml#L519-L538)
- [vacuum.c](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L880-L996)
- [vacuumlazy.c](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L372-L441)
- [varsup.c](../../../../raw/postgres-12/src/backend/access/transam/varsup.c#L118-L158)
- [walreceiver.c](../../../../raw/postgres-12/src/backend/replication/walreceiver.c#L1397-L1405)
- [walsender.c](../../../../raw/postgres-12/src/backend/replication/walsender.c#L2924-L2957)
- [xlog.c](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6733-L6846)
- [xlogarchive.c](../../../../raw/postgres-12/src/backend/access/transam/xlogarchive.c#L500-L539)
- [xlogfuncs.c](../../../../raw/postgres-12/src/backend/access/transam/xlogfuncs.c#L340-L361)
- [001_stream_rep.pl](../../../../raw/postgres-12/src/test/recovery/t/001_stream_rep.pl#L1-L20)
- [002_archiving.pl](../../../../raw/postgres-12/src/test/recovery/t/002_archiving.pl#L1-L20)
- [004_sync.pl](../../../../raw/postgres-12/src/test/subscription/t/004_sync.pl#L1-L20)
- [006_logical_decoding.pl](../../../../raw/postgres-12/src/test/recovery/t/006_logical_decoding.pl#L1-L27)
- [007_sync_rep.pl](../../../../raw/postgres-12/src/test/recovery/t/007_sync_rep.pl#L1-L20)

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Wiki index](../../../index.md)
- [Version manifest](../../../versions.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
