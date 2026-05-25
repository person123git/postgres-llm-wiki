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
