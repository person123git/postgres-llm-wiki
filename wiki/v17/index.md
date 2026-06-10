# PostgreSQL 17.10

## Source Pin

- Branch: `REL_17_STABLE`
- Commit: `54eeefaedbee0385529f3edf321bb99e49232aaa`
- Status: `active`
- Source path: `raw/postgres-17/`
- Added: 2026-05-13

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-17/`.

- [PostgreSQL 17 Contrib Extensions (unverified)](questions/contrib-extensions.md) - Complete inventory of PostgreSQL 17 contrib extensions backed by `*.control` files, with explanations for data types, indexes, diagnostics, FDWs, transforms, and SPI trigger examples.
- [Proposing a Sampling pgstatindex Variant for PostgreSQL 17 (unverified)](questions/pgstatindex-sample-variant-proposal.md) - Proposal for a `pgstatindex_approx` that random-samples physical B-tree pages instead of scanning all of them, with exact-vs-estimated fields, pros/cons, extension wiring, and a `pageinspect` SQL prototype.
- [GUC Default-Value Changes Since PostgreSQL 12 (unverified)](questions/guc-default-changes-since-v12.md) - The seven settings present in both v12 and v17 whose built-in default changed (`ssl_min_protocol_version`, `password_encryption`, `vacuum_cost_page_miss`, `checkpoint_completion_target`, `shared_buffers` boot value, `log_checkpoints`, `log_autovacuum_min_duration`), with old/new value, introducing major version, apply scope, exclusions for sample-only/type/added/removed/range-only changes, and test-coverage notes.
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](questions/create-index-concurrently.md) - Comprehensive walkthrough of the v17 CIC implementation (four internal transactions, two heap scans, three waits; the `indislive`/`indisready`/`indisvalid` progression); a dedicated steps-and-locks section (transaction- and session-level `ShareUpdateExclusiveLock`, the `WaitForLockers(ShareLock)` writer waits, `WaitForOlderSnapshots`); and a "what changed from PostgreSQL 12" section centered on the PG14 `PROC_IN_SAFE_IC` optimization (safe builds ignored in the snapshot wait, set via `set_indexsafe_procflags`), the reverted VACUUM-ignores-CIC attempt, and the `PGXACT`->`PGPROC` `statusFlags` move — all anchored to the v17 checkout's own commit history.
