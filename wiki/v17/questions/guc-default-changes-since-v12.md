---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: GPT-5-Codex 2026-05-30T11:59:13Z
---

# GUC Default-Value Changes Since PostgreSQL 12 (unverified)

## Question

In PostgreSQL 17, summarize all GUC default-value changes since version 12.

## Answer Up Front

Seven server settings that already existed in PostgreSQL 12 ship a different
built-in default value in PostgreSQL 17. Each row below gives the old default
(the value compiled into PostgreSQL 12), the PostgreSQL 17 default, the major
version that introduced the change, and how a change to the setting takes effect
(`postmaster` -> restart, `sighup` -> reload, `user`/`superuser` ->
session/transaction, per the context flag in `guc_tables.c`).

| Setting | Old default (v12) | New default (v17) | First in | Apply scope |
|---|---|---|---|---|
| `ssl_min_protocol_version` | `TLSv1` | `TLSv1.2` | 13 | Reload (`PGC_SIGHUP`) |
| `password_encryption` | `md5` | `scram-sha-256` | 14 | Session (`PGC_USERSET`) |
| `vacuum_cost_page_miss` | `10` | `2` | 14 | Session (`PGC_USERSET`) |
| `checkpoint_completion_target` | `0.5` | `0.9` | 14 | Reload (`PGC_SIGHUP`) |
| `shared_buffers` (built-in default) | `8MB` (1024 blocks) | `128MB` (16384 blocks) | 15 | Restart (`PGC_POSTMASTER`) |
| `log_checkpoints` | `off` | `on` | 15 | Reload (`PGC_SIGHUP`) |
| `log_autovacuum_min_duration` | `-1` (off) | `10min` | 15 | Reload (`PGC_SIGHUP`) |

The current value and apply scope of each setting come from the PostgreSQL 17
GUC table
([guc_tables.c](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c))
and the configuration docs
([config.sgml](../../../raw/postgres-17/doc/src/sgml/config.sgml)). The old
value and the introducing major version come from the same `raw/postgres-17/`
checkout's own commit history; the introducing version is read from the in-tree
`AC_INIT` development-version string at each commit (see
[Context Reviewed](#context-reviewed)).

Three things that *look* like default changes against PostgreSQL 12 are not
changes to a GUC's built-in default: `lc_messages` and `krb_server_keyfile`
only changed their `postgresql.conf.sample` comment, and `wal_compression`
changed type (boolean to enum) while keeping the `off` default. See
[Not Default-Value Changes](#not-default-value-changes). Settings that were
*added* after v12 are out of scope here, even though they have PostgreSQL 17
defaults; examples include `default_toast_compression`, `compute_query_id`, and
`maintenance_io_concurrency`
([guc_tables.c#added-after-v12-examples](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3123-L3133),
[guc_tables.c#more-added-after-v12-examples](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4775-L4803)).
Settings removed before v17 are also out of scope; for example, the v17 release
notes record the removal of `old_snapshot_threshold`
([release-17.sgml#old_snapshot_threshold](../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L11210-L11219)).
This page is about defaults of settings present in both v12 and v17.

## Changed in PostgreSQL 13

### `ssl_min_protocol_version`: `TLSv1` -> `TLSv1.2`

The minimum SSL/TLS protocol the server will negotiate was raised from TLS 1.0
to TLS 1.2. The PostgreSQL 17 boot value is `PG_TLS1_2_VERSION`
([guc_tables.c#ssl_min_protocol_version](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5112-L5118)),
the docs state "The default is `TLSv1.2`"
([config.sgml#ssl_min_protocol_version](../../../raw/postgres-17/doc/src/sgml/config.sgml#L1503)),
and the sample file shows `TLSv1.2`
([postgresql.conf.sample:116](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L116)).
The change was made by commit `b1abfec8` (2019-12-04, "Update minimum SSL
version"), which carried the in-tree version string `13devel`. It is a
`PGC_SIGHUP` setting, so a change takes effect on configuration reload.

## Changed in PostgreSQL 14

### `password_encryption`: `md5` -> `scram-sha-256`

New passwords set with `CREATE ROLE` / `ALTER ROLE ... PASSWORD` are now hashed
with SCRAM-SHA-256 instead of MD5. The boot value is
`PASSWORD_TYPE_SCRAM_SHA_256`
([guc_tables.c#password_encryption](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5089-L5094)),
and the docs state "The default is `scram-sha-256`"
([config.sgml#password_encryption](../../../raw/postgres-17/doc/src/sgml/config.sgml#L1120)),
matching the sample
([postgresql.conf.sample:97](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L97)).
Changed by commit `c7eab0e9` (2020-06-10, "Change default of
password_encryption to scram-sha-256"), in-tree version `14devel`. As a
`PGC_USERSET` setting it can be changed for a single session by any role; a
change to the cluster default in `postgresql.conf` takes effect on reload.

### `vacuum_cost_page_miss`: `10` -> `2`

The cost charged during cost-based vacuum delay for reading a page not already
in shared buffers dropped from 10 to 2, reflecting that buffer misses are
cheaper relative to dirtying a page than the old ratio assumed. The boot value
is `2`
([guc_tables.c#vacuum_cost_page_miss](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2525-L2530)),
the docs state "The default value is 2"
([config.sgml#vacuum_cost_page_miss](../../../raw/postgres-17/doc/src/sgml/config.sgml#L2475)),
and the sample shows `2`
([postgresql.conf.sample:195](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L195)).
Changed by commit `e19594c5` (2021-01-27, "Reduce the default value of
vacuum_cost_page_miss"), in-tree version `14devel`. It is `PGC_USERSET`
(session/transaction; reload for the global default).

### `checkpoint_completion_target`: `0.5` -> `0.9`

The fraction of the checkpoint interval over which dirty buffers are spread was
raised from 0.5 to 0.9, smoothing checkpoint write bursts. The boot value is
`0.9`
([guc_tables.c#checkpoint_completion_target](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3916-L3922)),
and the docs state "The default is 0.9"
([config.sgml#checkpoint_completion_target](../../../raw/postgres-17/doc/src/sgml/config.sgml#L3620)),
matching the sample
([postgresql.conf.sample:259](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L259)).
Changed by commit `bbcc4eb2` (2021-03-24, "Change checkpoint_completion_target
default to 0.9"), in-tree version `14devel`. It is `PGC_SIGHUP` (reload).

## Changed in PostgreSQL 15

### `shared_buffers` built-in default: `8MB` -> `128MB`

The value compiled into the GUC table (the `pg_settings.boot_val`, used when no
value is written by `initdb`) rose from 1024 blocks (8MB) to 16384 blocks
(128MB) at the standard 8kB `BLCKSZ`. The PostgreSQL 17 boot value is `16384`
([guc_tables.c#shared_buffers](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2267)),
the docs say "The default is typically 128 megabytes"
([config.sgml#shared_buffers](../../../raw/postgres-17/doc/src/sgml/config.sgml#L1646)),
and the sample shows `128MB`
([postgresql.conf.sample:129](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L129)).

Caveat on practical impact: this is mainly a correction so the built-in default,
`pg_settings`, and the sample agree with what `initdb` already produces. The
introducing commit `f7bda63a` (2021-08-23, "Improve defaults shown in
postgresql.conf.sample and pg_settings", in-tree version `15devel`) notes the
realistic default has effectively been 128MB since PostgreSQL 10 because
`initdb` probes several increasing values and normally writes an explicit value
into the generated `postgresql.conf`. `shared_buffers` is `PGC_POSTMASTER`, so a
change requires a server restart.

### `log_checkpoints`: `off` -> `on`

Checkpoint activity is now logged by default. The boot value is `true`
([guc_tables.c#log_checkpoints](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1200-L1205)),
the docs state "The default is on"
([config.sgml#log_checkpoints](../../../raw/postgres-17/doc/src/sgml/config.sgml#L7365)),
and the sample shows `on`
([postgresql.conf.sample:583](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L583)).

### `log_autovacuum_min_duration`: `-1` -> `10min`

Autovacuum and autoanalyze actions taking at least 10 minutes are now logged by
default, where `-1` previously disabled this logging entirely. The boot value is
`600000` milliseconds
([guc_tables.c#log_autovacuum_min_duration](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3041-L3048)),
and the docs state "The default is `10min`"
([config.sgml#log_autovacuum_min_duration](../../../raw/postgres-17/doc/src/sgml/config.sgml#L7344)),
matching the sample
([postgresql.conf.sample:578](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L578)).

Both `log_checkpoints` and `log_autovacuum_min_duration` were changed in the same
commit `64da07c4` (2021-12-13, "Default to log_checkpoints=on,
log_autovacuum_min_duration=10m"), in-tree version `15devel`. Both are
`PGC_SIGHUP` (reload).

## Not Default-Value Changes

These differ from PostgreSQL 12 in the sample file or in type, but the GUC's
built-in default value did not change:

- **`lc_messages`**: the built-in default is the empty string `""` in both v12
  and v17
  ([guc_tables.c#lc_messages](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4206-L4211)).
  Only the `postgresql.conf.sample` comment changed from `'C'` to `''`
  ([postgresql.conf.sample:757](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L757));
  the commit that did so (`3d185cfc`, 2024-01-10, in-tree version `17devel`)
  touched only the sample and `initdb`, not the GUC table.
- **`krb_server_keyfile`**: the boot value is the macro `PG_KRB_SRVTAB`
  ([guc_tables.c#krb_server_keyfile](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4185-L4191)),
  whose in-source fallback is `""`
  ([guc_tables.c#PG_KRB_SRVTAB](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L94-L96))
  and is normally supplied by the build as `FILE:${sysconfdir}/krb5.keytab` in
  both v12 and v17. Commit `860fe27e` (2020-12-30, "Fix up usage of
  krb_server_keyfile GUC parameter", back-patched to v12) only corrected
  authentication behavior, the docs, and the sample comment
  ([postgresql.conf.sample:101](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L101)).
- **`wal_compression`**: changed from a boolean to an enum (to add `pglz`/`lz4`/
  `zstd`), but the default remains `off`
  ([guc_tables.c#wal_compression](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4964-L4969)),
  ([postgresql.conf.sample:243](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L243)).
- **Allowed-range tweaks, not default changes**: `maintenance_work_mem` and
  `autovacuum_work_mem` kept their defaults while allowing smaller explicit
  values (`64MB` for `maintenance_work_mem`, `-1` for `autovacuum_work_mem`;
  explicit autovacuum values below `64kB` are clamped to `64kB`)
  ([guc_tables.c#maintenance_work_mem](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2472),
  [guc_tables.c#autovacuum_work_mem](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3441-L3448),
  [autovacuum.c#check_autovacuum_work_mem](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3367-L3385)).
  `max_files_per_process` still defaults to `1000` while its minimum is `64`
  ([guc_tables.c#max_files_per_process](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2565-L2571)),
  and `track_activity_query_size` still defaults to `1024` while its maximum is
  `1048576`
  ([guc_tables.c#track_activity_query_size](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3565-L3572)).

## Context Reviewed

- **Current PostgreSQL 17 defaults and apply scope**: read from the GUC table
  `src/backend/utils/misc/guc_tables.c` (boot value and `PGC_*` context flag),
  cross-checked against `doc/src/sgml/config.sgml` and
  `src/backend/utils/misc/postgresql.conf.sample`, all in the pinned
  `raw/postgres-17/` checkout (commit `54eeefae`).
- **Enumeration method**: extracted every `name -> boot_val` pair from the
  PostgreSQL 17 `guc_tables.c` and compared it against the PostgreSQL 12
  `guc.c` table (read for cross-checking only; not cited on this page, per the
  one-version-per-page citation rule). After filtering range-only edits, macro
  renames, type changes, and preprocessor lines, exactly seven settings present
  in both versions show a changed boot value. The `postgresql.conf.sample` diff
  agrees with that set.
- **Old value and introducing version**: taken from the `raw/postgres-17/`
  checkout's own Git history (`git blame` on the current line, then the
  introducing commit). The major version is read from the in-tree
  `AC_INIT([PostgreSQL], [NNdevel], ...)` string in `configure.ac`/`configure.in`
  as it stood at each commit, which records the development version that shipped
  the change. This keeps all evidence inside the single pinned checkout.
- **Scope rule applied**: `postmaster` -> restart; `sighup` -> reload;
  `user`/`superuser` -> session/transaction (with reload for the global value).

## Test Coverage

I did not find a dedicated PostgreSQL 17 test that enumerates this
v12-to-v17 default-change set as a group. Same-checkout tests and harness code
exercise individual settings instead: `password_encryption` accepted values and
password storage paths
([password.sql#password_encryption](../../../raw/postgres-17/src/test/regress/sql/password.sql#L5-L19)),
SSL protocol bounds using `ssl_min_protocol_version`
([001_ssltests.pl#ssl_min_protocol_version](../../../raw/postgres-17/src/test/ssl/t/001_ssltests.pl#L101-L116)),
the `64kB` `maintenance_work_mem` minimum
([vacuum.sql#maintenance_work_mem-minimum](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L137-L143)),
and the regression harness's explicit logging settings for test clusters
([pg_regress.c#test-cluster-logging-gucs](../../../raw/postgres-17/src/test/regress/pg_regress.c#L2397-L2402)).

## Evidence Map

| Claim | Evidence |
|---|---|
| `ssl_min_protocol_version` default is `TLSv1.2`, reload scope | [guc_tables.c#L5112-L5118](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5112-L5118), [config.sgml#L1503](../../../raw/postgres-17/doc/src/sgml/config.sgml#L1503) |
| `password_encryption` default is `scram-sha-256`, session scope | [guc_tables.c#L5089-L5094](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L5089-L5094), [config.sgml#L1120](../../../raw/postgres-17/doc/src/sgml/config.sgml#L1120) |
| `vacuum_cost_page_miss` default is `2`, session scope | [guc_tables.c#L2525-L2530](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2525-L2530), [config.sgml#L2475](../../../raw/postgres-17/doc/src/sgml/config.sgml#L2475) |
| `checkpoint_completion_target` default is `0.9`, reload scope | [guc_tables.c#L3916-L3922](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3916-L3922), [config.sgml#L3620](../../../raw/postgres-17/doc/src/sgml/config.sgml#L3620) |
| `shared_buffers` built-in default is `16384` blocks (128MB), restart scope | [guc_tables.c#L2261-L2267](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2261-L2267), [config.sgml#L1646](../../../raw/postgres-17/doc/src/sgml/config.sgml#L1646) |
| `log_checkpoints` default is `on`, reload scope | [guc_tables.c#L1200-L1205](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1200-L1205), [config.sgml#L7365](../../../raw/postgres-17/doc/src/sgml/config.sgml#L7365) |
| `log_autovacuum_min_duration` default is `10min`, reload scope | [guc_tables.c#L3041-L3048](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3041-L3048), [config.sgml#L7344](../../../raw/postgres-17/doc/src/sgml/config.sgml#L7344) |
| `lc_messages` built-in default unchanged (`""`) | [guc_tables.c#L4206-L4211](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4206-L4211) |
| `krb_server_keyfile` boot value is `PG_KRB_SRVTAB` (fallback `""`) | [guc_tables.c#L4185-L4191](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4185-L4191), [guc_tables.c#L94-L96](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L94-L96) |
| `wal_compression` type changed but default remains off | [guc_tables.c#L4964-L4969](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4964-L4969), [postgresql.conf.sample#L243](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample#L243) |
| Range-only edits are not default-value changes | [guc_tables.c#L2465-L2472](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2465-L2472), [guc_tables.c#L3441-L3448](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3441-L3448), [guc_tables.c#L2565-L2571](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2565-L2571), [guc_tables.c#L3565-L3572](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3565-L3572) |
| Tests cover individual settings, not the full cross-version inventory | [password.sql#L5-L19](../../../raw/postgres-17/src/test/regress/sql/password.sql#L5-L19), [001_ssltests.pl#L101-L116](../../../raw/postgres-17/src/test/ssl/t/001_ssltests.pl#L101-L116), [vacuum.sql#L137-L143](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L137-L143), [pg_regress.c#L2397-L2402](../../../raw/postgres-17/src/test/regress/pg_regress.c#L2397-L2402) |
| Old values / introducing major versions | `raw/postgres-17/` Git history: commits `b1abfec8` (13), `c7eab0e9`/`e19594c5`/`bbcc4eb2` (14), `f7bda63a`/`64da07c4` (15), with the in-tree `AC_INIT` dev version at each commit |

## Source References

- [guc_tables.c](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c) - GUC table: boot values and `PGC_*` apply-scope flags for all settings on this page.
- [postgresql.conf.sample](../../../raw/postgres-17/src/backend/utils/misc/postgresql.conf.sample) - shipped sample showing the documented defaults.
- [config.sgml](../../../raw/postgres-17/doc/src/sgml/config.sgml) - configuration documentation stating each default.
- [password.sql](../../../raw/postgres-17/src/test/regress/sql/password.sql), [001_ssltests.pl](../../../raw/postgres-17/src/test/ssl/t/001_ssltests.pl), [vacuum.sql](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql), and [pg_regress.c](../../../raw/postgres-17/src/test/regress/pg_regress.c) - individual GUC behavior tests and harness settings.

## Open Questions

- The page scopes "default changes" to settings present in both v12 and v17.
  A full v12-to-17 settings inventory would also enumerate settings added
  (with new defaults) and removed across v13-v17; those are tracked in the
  per-version release notes rather than here.
- Old values and introducing versions are derived from this checkout's own Git
  history and in-tree version strings, not from the separate PostgreSQL 12
  checkout, because a v17 page cites only its own version. A reviewer wanting
  independent confirmation of a v12 value can read the GUC table at
  `src/backend/utils/misc/guc.c` in the separate PostgreSQL 12 checkout.

## Related Pages

- [v17/index](../index.md)
- [versions](../../versions.md)
