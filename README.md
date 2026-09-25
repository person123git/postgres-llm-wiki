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

A clone is ready to read. Every page under `wiki/` is committed Markdown, so nothing is generated, compiled, or rebuilt to use the wiki.

1. Open the clone in VS Code, or any Markdown viewer that follows relative links. Wiki navigation and source citations are plain relative Markdown links — no Obsidian wikilinks — so the built-in preview follows them with no extension installed.
2. Enter through [wiki/index.md](wiki/index.md) for the global catalog, [wiki/versions.md](wiki/versions.md) for the supported versions and their exact source pins, or a version landing page such as [wiki/v18/index.md](wiki/v18/index.md).
3. Read [AGENTS.md](AGENTS.md) before asking an agent to add or change a page. It is the authority for evidence, citation, verification, and bookkeeping rules.

### Query the wiki with an LLM CLI harness

Reading the pages is half of it. New answers come from asking an LLM CLI harness — an agent running in your terminal with read and write access to the clone — to do the work under the repo's rules. Use this prompt format:

```text
Follow AGENTS.md. In PostgreSQL vNN, question: <your question>
```

For example:

```text
Follow AGENTS.md. In PostgreSQL v18, question: how does VACUUM decide to truncate a relation?
```

- `Follow AGENTS.md` binds the agent to the evidence, citation, verification, and bookkeeping rules. Some harnesses load `AGENTS.md` on their own; naming it makes the binding explicit either way.
- `In PostgreSQL vNN` picks the pinned checkout the answer must cite, and the wiki tree the page lands in. Omit it and the agent works on the primary version and says so.
- The answer is filed as a page, not chat output: a `type: question` page under `wiki/vNN/questions/<category>/` that restates your question verbatim, answers it inline with citations into `raw/postgres-NN/`, and lists what stayed unresolved under `## Open Questions`. The agent then updates the indexes and appends to `wiki/log.md`.
- Expect one round trip before drafting if the question has typos: `AGENTS.md` makes the agent ask whether to correct them, because the page restates your wording verbatim.
- In a fresh clone the first thing to ask for is the evidence base itself, since the agent cannot cite what is not on disk.

### What a clone does not carry

`.gitignore` excludes the evidence base and the runtime directory, so a fresh clone is far smaller than a working copy.

| Absent after cloning | What it costs you | How it comes back |
|---|---|---|
| `raw/postgres-NN/` pinned PostgreSQL checkouts | Citation links point at files that are not on disk, so they do not open, and `scripts/wiki_lint` reports `missing source checkout for vNN` plus one broken-citation error per cited range. | The LLM harness fetches them. Ask the agent for the version you need; it must park the checkout on the exact commit [wiki/versions.md](wiki/versions.md) pins, because lint compares `HEAD` against that pin and cited line numbers are only valid there. Each checkout is a full PostgreSQL clone, roughly 0.6-0.9 GB. |
| `.wiki-runtime/` (venv, caches, logs, sandboxes) | `scripts/wiki_lint` refuses to run, because the scripts require the project venv. | Create it once per clone: `python3 -m venv .wiki-runtime/venv`. The tooling makes its own cache, log, and tmp subdirectories. |

Wiki prose does not depend on either one: reading, searching, and following page links all work in a bare clone. Only citation links and lint need `raw/postgres-NN/`.

### If you will edit pages

```bash
python3 -m venv .wiki-runtime/venv                # once per clone; requirements.txt pins no packages
.wiki-runtime/venv/bin/python scripts/wiki_lint   # after every wiki-facing change
```

`scripts/wiki_lint` checks links, front matter, pins, citation form, verification fields, and landing-page coverage. With every pinned checkout in place it reports `0 error(s), 0 warning(s)`.

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
- [AGENTS.md](AGENTS.md): Instructions for contributing to the wiki

For detailed coverage, start with [versions](wiki/versions.md) or the version-specific landing pages: [v19/index](wiki/v19/index.md), [v18/index](wiki/v18/index.md), [v17/index](wiki/v17/index.md), [v12/index](wiki/v12/index.md).
