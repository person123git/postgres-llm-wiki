# PostgreSQL Engine Wiki

An LLM-maintained knowledge base for PostgreSQL engine internals, built with version-pinned raw source evidence.

## Overview

This repository documents PostgreSQL internals through source-cited wiki pages. Every behavioral claim should cite the matching PostgreSQL checkout under `raw/postgres-NN/`; uncertainty is preserved under `Open Questions`.

This wiki is based on the LLM wiki concept from [Andrej Karpathy's gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), adapted for maintaining technical documentation with version-pinned source evidence.

## Supported Versions

| Version | Status | Pinned Commit |
|---------|--------|---|
| 18 | Primary | `6cb307251c5c6261286c1566496920976640108e` |
| 17 | Active | `54eeefaedbee0385529f3edf321bb99e49232aaa` |
| 12 | Legacy | `45b88269a353ad93744772791feb6d01bc7e1e42` |

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

**PostgreSQL 18** (Primary): 10 filed questions covering buffer partitioning, statistics, vacuum hooks, custom stats, pgstatindex sampling, and query analysis.

**PostgreSQL 17** (Active): 2 filed questions covering contrib extensions and pgstatindex sampling.

**PostgreSQL 12** (Legacy): 6 filed questions covering foreign-key optimization, pgstatindex, index density, query analysis, and statistics.

All pages are source-backed with citations to the pinned PostgreSQL checkouts. Recent work includes comprehensive behavioral verification and citation precision audits. See [log](wiki/log.md) for recent activity.

## More Information

- [The Idea](idea.md): The core concept behind LLM-maintained wikis
- [Implementation Plan](postgresql-engine-wiki-plan.md): Detailed technical specifications and setup
- [AGENTS.md](AGENTS.md): Instructions for contributing to the wiki

For detailed coverage, start with [versions](wiki/versions.md) or the version-specific landing pages: [v18/index](wiki/v18/index.md), [v17/index](wiki/v17/index.md), [v12/index](wiki/v12/index.md).
