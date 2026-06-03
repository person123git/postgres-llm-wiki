# PostgreSQL Engine Wiki

An LLM-maintained knowledge base for PostgreSQL engine internals, built with version-pinned raw source evidence.

## Overview

This repository documents PostgreSQL internals through source-cited wiki pages. Every behavioral claim should cite the matching PostgreSQL checkout under `raw/postgres-NN/`; uncertainty is preserved under `Open Questions`.

This wiki is based on the LLM wiki concept from [Andrej Karpathy's gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), adapted for maintaining technical documentation with version-pinned source evidence.

## Supported Versions

| Version | Status | Branch | Pinned Commit |
|---------|--------|--------|---|
| 19 | Active | `master` (`19beta1`) | `298bdd379552148f6043b4595374a7a6fbdd13c3` |
| 18 | Primary | `REL_18_STABLE` | `6cb307251c5c6261286c1566496920976640108e` |
| 17 | Active | `REL_17_STABLE` | `54eeefaedbee0385529f3edf321bb99e49232aaa` |
| 12 | Legacy | `REL_12_STABLE` | `45b88269a353ad93744772791feb6d01bc7e1e42` |

See [versions](wiki/versions.md) for full version index.

## Getting Started

1. Bootstrap the environment: `./bootstrap_venv`
2. Activate the venv: `source .wiki-runtime/venv/bin/activate`
3. Read [AGENTS.md](AGENTS.md) before making wiki changes.
4. Run wiki maintenance commands through the project venv.

## Project Structure

- `wiki/`: Wiki content
- `raw/`: PostgreSQL source checkouts
- `scripts/`: Tooling scripts
- `.wiki-runtime/`: Generated artifacts and venv

## Coverage Summary

**PostgreSQL 19** (Active): 2 filed questions covering the new `pg_plan_advice` contrib module and the new in-core `REPACK` / `REPACK (CONCURRENTLY)` command, each with scoped source history.

**PostgreSQL 18** (Primary): 11 filed questions covering buffer partitioning, custom cumulative statistics, vacuum/autovacuum extension hooks, B-tree leaf density during vacuum, pgstatindex approximation and sampling, `pg_stat_statements` mechanics, `track_activity_query_size`, `EXPLAIN (ANALYZE, BUFFERS)`, and GUC default-value changes since v12.

**PostgreSQL 17** (Active): 3 filed questions covering the contrib extension inventory, GUC default-value changes since v12, and a pgstatindex sampling-variant proposal.

**PostgreSQL 12** (Legacy): 9 filed questions covering foreign-key join selectivity, pgstatindex calculation behavior, B-tree leaf density and fragmentation, the bloated-index query planner, `psql` environment/timeout behavior, `pg_stat_statements`, `EXPLAIN (ANALYZE, BUFFERS)`, and a pgstatindex sampling-variant proposal.

All pages are source-backed with citations to the pinned PostgreSQL checkouts. Recent work includes comprehensive behavioral verification and citation precision audits. See [log](wiki/log.md) for recent activity.

## More Information

- [The Idea](idea.md): The core concept behind LLM-maintained wikis
- [Implementation Plan](postgresql-engine-wiki-plan.md): Detailed technical specifications and setup
- [AGENTS.md](AGENTS.md): Instructions for contributing to the wiki

For detailed coverage, start with [versions](wiki/versions.md) or the version-specific landing pages: [v19/index](wiki/v19/index.md), [v18/index](wiki/v18/index.md), [v17/index](wiki/v17/index.md), [v12/index](wiki/v12/index.md).
