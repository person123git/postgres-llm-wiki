# PostgreSQL Engine Wiki Agent Instructions

This repo is an LLM-maintained wiki for PostgreSQL internals. The pinned PostgreSQL checkout under `raw/postgres-NN/` is the evidence base.

## MANDATORY Read First

- Read `wiki/versions.md` before modifying or answering from the wiki.
- Read `wiki/index.md`.
- Read the last ~20 entries of `wiki/log.md`.
- For version-local work, read `wiki/vNN/index.md`.
- Use the matching `raw/postgres-NN/` checkout as the PostgreSQL evidence base.

## MANDATORY Environment Isolation

- Stay inside this repo.
- Read/write only `raw/`, `wiki/`, `.wiki-runtime/`, `scripts/`, `tests/`, `requirements.txt`, and top-level docs.
- Treat `raw/postgres-NN/` checkouts as read-only evidence.
- Run Python scripts from `.wiki-runtime/venv/`: activate it or call `.wiki-runtime/venv/bin/python scripts/<name>`.
- If the venv is missing, run `scripts/bootstrap_venv`. Only that script may use host `python3`.
- Pin new Python deps in `requirements.txt`.
- Do not install packages globally, with `--user`, via `pipx`, or with `sudo`.
- Do not use `sudo`, host-path `chown`, or host-path `chmod`.
- Use network only for `scripts/bootstrap_venv` or user-requested source fetches.
- Do not use `WIKI_ALLOW_SYSTEM_PYTHON=1` in normal work.
- Keep generated artifacts, caches, and the venv under `.wiki-runtime/`.

## MANDATORY Evidence

- Use only the target version's pinned `raw/postgres-NN/` checkout as factual evidence.
- Treat implementation source as primary evidence.
- Same-checkout PostgreSQL docs, tests, and source history may support claims when directly relevant.
- Treat `wiki/versions.md`, `wiki/index.md`, `wiki/log.md`, and version landing pages as navigation only.
- Do not use model memory, external websites, external package docs, or uncited prior wiki prose as factual support.
- If implementation source conflicts with docs or tests, source wins. Put the discrepancy under `## Open Questions`.
- Never answer one PostgreSQL version with evidence from another version.

## MANDATORY Prompt Hygiene

- If a user question or any prompt that drives document generation contains typos or grammatical errors, stop before generating and ask the user whether to correct the prompt issues or keep them as written.
- Apply this to filed `## Question` text, ingest prompts, and any prompt that will be restated verbatim in a wiki page.
- Wait for the user's answer before drafting. Do not silently rewrite the prompt.

## MANDATORY Deep Inquiry

Deep inquiry is the default unless the user explicitly asks for a quick answer.

- Confirm the target PostgreSQL version.
- Locate primary source files and symbols.
- Inspect adjacent callers, callees, structs, macros, includes, generated headers visible in raw source, reverse include users, tests, docs, catalogs, grammar, error paths, GUCs, and extension/contrib boundaries.
- If evidence lookup fails or is untrustworthy, stop before drafting. Fix it, rerun it, or report the target version and error.
- Inspect history when the user asks why, when intent matters, or when making a regression/change claim.
- For cross-version claims, collect evidence for each relevant version.
- Draft from a claim-to-source map. Put unresolved claims under `## Open Questions`.
- Minimum engine answer: normal path, edge/error path, key data structures, caller/callee boundary, build/generated-header implications visible from raw source, and tests or explicit test absence.
- For planner, WAL, crash recovery, MVCC, storage, or corruption topics, missing caller/callee or data-structure context is a verification gap.

## MANDATORY Citations

- Cite every behavioral claim.
- Use page-relative Markdown links for source citations, not Obsidian wikilinks or bare raw-file links. Canonical shapes:

  ```md
  [file.c#Symbol](../../../raw/postgres-NN/path/file.c#L42-L58)
  [file.c:42](../../../raw/postgres-NN/path/file.c#L42)
  ```

  A source citation is a complete Markdown inline link, not a link fragment. The opening `[`, closing `]`, opening `(`, and closing `)` are mandatory.

  ```md
  [<label>](<relative-path-to-raw-file>#L<start>-L<end>)
  [<label>](<relative-path-to-raw-file>#L<line>)
  ```

  Invalid citation fragments include:

  ```md
  file.c#Symbol](../../../raw/postgres-NN/path/file.c#L42-L58
  [file.c#Symbol]../../../raw/postgres-NN/path/file.c#L42-L58
  raw/postgres-NN/path/file.c#L42-L58
  [[raw/postgres-NN/path/file.c]]
  ```

  When asked to describe the citation format, show complete Markdown inline-link examples. Do not omit the delimiter characters.

  - Link text: short human label, typically `file.ext#Symbol` (function, struct, macro, GUC, or doc-section name). Use `file.ext:line` for a single-line citation.
  - URL: page-relative path to the file in the matching `raw/postgres-NN/` checkout, with a `#Lstart-Lend` line-range fragment. Single-line citations use `#L42`.
  - Use enough `../` segments to make the link open from the current wiki page in VS Code. For root-level version pages such as `wiki/vNN/codebase-navigation-guide.md`, that prefix is `../../raw/postgres-NN/...`. For pages under `wiki/vNN/questions/`, that prefix is `../../../raw/postgres-NN/...`.
  - New or edited source citations must use this page-relative format. `scripts/wiki_lint` may normalize repo-relative `raw/postgres-NN/...` URLs for validation, but that is compatibility behavior, not the citation style for new work.
  - Line numbers are stable because every page pins an exact commit via `pinned_commit:`; they jump correctly in VS Code and editors that understand Markdown line fragments.
- Include full extensions for non-Markdown files (`.c`, `.h`, `.sgml`, `.sql`, `.out`).
- Cite from the `raw/postgres-NN/` checkout matching the page `version:`. Never cite across versions.
- Use one citation style per page. Don't mix Markdown citations with the old `[[raw/...]]` wikilink form on the same page.
- Page-to-page wiki navigation uses the same page-relative Markdown link syntax so it opens in plain VS Code Markdown preview. Do not use Obsidian wikilinks for wiki page navigation.
- Do not state a claim as fact unless it is backed by a source file, symbol, test file, documentation page, commit, or saved design discussion.
- Put uncertainty under `## Open Questions`.

Examples:

```md
[explain.c#ExplainOnePlan](../../../raw/postgres-18/src/backend/commands/explain.c#L494-L598)
[instrument.h#BufferUsage](../../../raw/postgres-18/src/include/executor/instrument.h#L24-L42)
[ref/explain.sgml#BUFFERS](../../../raw/postgres-18/doc/src/sgml/ref/explain.sgml#L181-L208)
[bufmgr.c:4397](../../../raw/postgres-18/src/backend/storage/buffer/bufmgr.c#L4397)
```

Migration note: existing pages that still use `[[raw/postgres-NN/...]]` wikilink citations remain valid and need not be rewritten until they are next edited. New and substantially-revised pages must use the Markdown form.

## MANDATORY Writing Style

- Lead with the answer.
- Use plain language and short sentences.
- Define PostgreSQL terms of art on first use or link to an existing page.
- Use active voice and name concrete subjects.
- Use lists, tables, and small code blocks for dense material.
- Name conditions precisely. Avoid vague hedges.
- Skip filler and setup prose.
- Cite every example.
- Never trade citation precision for readability.

## MANDATORY GUC Changes

- When suggesting any GUC change, state whether it needs restart, reload, or only session/transaction scope.
- Determine the requirement from the same-version raw GUC definition or a validated `pg_settings` definition.
- Map contexts explicitly: `postmaster` -> restart; `sighup` -> reload; `superuser`, `user`, `backend` -> session/transaction scope.

## MANDATORY Production SQL

- Verify production-bound SQL against the pinned checkout before filing.
- If syntax, catalogs, columns, functions, or GUCs cannot be verified, move the snippet under `## Open Questions`.
- Recommend reasonable session-scoped `statement_timeout` and `lock_timeout` values.
- Add an inline block-comment tag after the leading verb:

```sql
SELECT /* wiki_capture_plan_inputs */ ...;
UPDATE /* wiki_backfill_user_email */ users SET ...;
```

## MANDATORY Verification Fields

- `verified:` is human-only. Agents must not set, change, or remove it.
- `verified_by_agent:` records agent verification. Use `not yet` for drafts. Use the timestamp form only after re-checking every claim against pinned raw source.
- New filed question pages use this exact front matter order, and must include a `## Question` section that restates the user prompt verbatim plus an inline `## Answer` section:

```yaml
type: question
version: NN
pinned_commit: abc123...
verified: false
verified_by_agent: not yet
```

- Codebase navigation guide pages use this exact front matter order and otherwise follow all question-document rules:

```yaml
type: codebase-navigation-guide
version: NN
pinned_commit: abc123...
verified: false
verified_by_agent: not yet
```

- Legacy `type: answer` pages use the same field order with `type: answer`. Do not file new answer pages; see `MANDATORY Question Documents`.

- Do not set the timestamp form if any claim is unverified. Fix it, move it under `## Open Questions`, or leave `verified_by_agent: not yet`.
- Unverified managed pages must show `(unverified)` in the visible title and in index/landing-page link text until `verified: true`.

Title rule before creating, editing, or filing any wiki page:

- If `verified:` is not `true`, the top-level title must end with ` (unverified)`.
- If `verified:` is `true`, the title must not contain `(unverified)`.

`verified_by_agent` must be one of:

```yaml
verified_by_agent: not yet
verified_by_agent: <model-name> YYYY-MM-DDTHH:MM:SSZ
```

The timestamp form is `<model-name> <ISO-8601-UTC>`: a single space separator, then a UTC timestamp ending in `Z`, e.g. `claude-opus-4-8 2026-06-06T14:30:00Z`. The model name must match `[a-zA-Z0-9_-]+` (no spaces or dots). `scripts/wiki_lint` enforces exactly this shape. Use the exact current model name and the real verification time when filing an agent-verified page.

## MANDATORY Version Awareness

- `wiki/versions.md` is the source pin manifest.
- Each supported version has `wiki/vNN/index.md`.
- Each supported version has `wiki/vNN/codebase-navigation-guide.md`.
- Default new ingests and answers to the primary version unless the user specifies another.
- If the user omits a version, assume the primary version and state that assumption.
- Every source citation must use the matching `raw/postgres-NN/` checkout.
- Never use citations from another PostgreSQL version.

## MANDATORY Codebase Navigation Guide

Every supported version must have one codebase navigation guide:

```text
wiki/vNN/codebase-navigation-guide.md
```

- Use `type: codebase-navigation-guide`.
- File it at the version root, not under `questions/`, `concepts/`, or `answers/`.
- Treat it as version-local content and as a question-style document. It must have front matter, a `## Contents` table of contents, `## Question`, inline `## Answer`, matching-version raw source citations, `## Context Reviewed`, `## Evidence Map`, `## Open Questions`, `## Source References`, and `## Navigation`.
- Apply every `MANDATORY Question Documents` rule unless it conflicts with the fixed `type:` or fixed root-level path for this guide.
- If a user-requested guide prompt exists, restate that prompt verbatim under `## Question` after applying `MANDATORY Prompt Hygiene`. If the guide is generated as a mandatory per-version scaffold without a user prompt, use this canonical question text: `Create a codebase navigation guide document for PostgreSQL NN.`
- The guide must orient readers to the pinned checkout's source layout, normal SQL statement path, utility-command path, generated files and catalog/parser/header implications, key data structures, extension/contrib boundaries, tests, and docs.
- Update the guide when adding a supported version, repinning a supported version, or making a meaningful source-tree coverage change that affects codebase navigation.
- Link the guide from `wiki/index.md` and the matching `wiki/vNN/index.md`.

## MANDATORY Question Documents

When a user asks a question, the deliverable is a single `type: question` page that holds both the question and its answer. Do not spin off a separate answer document.

- File the page under `wiki/vNN/questions/`.
- Restate the user prompt verbatim under `## Question`.
- Put the full answer, with matching-version raw citations, inline under `## Answer`.
- Keep `## Context Reviewed`, `## Evidence Map`, and `## Open Questions` on the same page when gaps exist.

Why one document per question, not a question page plus an answer page:

- A user's question is the canonical entry point. The answer belongs with the question that motivated it, so a reader sees both without a second hop.
- One page per Q&A removes duplicate, drifting pages. A separate answer document forces two titles, two `(unverified)` hints, two `pinned_commit:` values, and two index entries to keep in sync.
- It keeps verification honest. One page carries one `verified:` / `verified_by_agent:` state over one claim-to-source map, instead of a question page that silently goes stale against its answer page.

Separate `type: answer` pages under `wiki/vNN/answers/` are legacy. Do not create new ones. Leave existing answer pages in place until they are next substantially revised, then fold them back into their question page.

## MANDATORY Table of Contents

- Every content page must open with a `## Contents` table of contents: `type: question`, `type: codebase-navigation-guide`, `type: concept`, and legacy `type: answer` pages, regardless of page length.
- Navigation pages are exempt: `wiki/index.md`, `wiki/versions.md`, `wiki/log.md`, `wiki/overview.md`, and the `wiki/vNN/index.md` version landing pages.
- Place the `## Contents` block between the page title (`# ...`) and the first content section. On a question-style page, including `type: codebase-navigation-guide`, that means immediately before `## Question`.
- List every `##` and `###` section in document order as a nested Markdown bullet list: each `##` is a top-level bullet and its `###` subsections are indented two spaces beneath it. Do not list `####` or deeper headings.
- The `## Contents` section never lists itself.
- Keep it in sync with the page: update the table of contents whenever a `##`/`###` section is added, removed, renamed, or reordered.
- Link with page-internal Markdown anchors `[Section title](#slug)`. Build each `#slug` with the VS Code Markdown-preview rule: trim, lowercase, replace each whitespace run with `-`, then strip punctuation — including backticks, underscores, and em dashes (`—`). So `### The three pg_index state flags` -> `#the-three-pgindex-state-flags` and `## Open Questions` -> `#open-questions`. Suffix later duplicate slugs with `-1`, `-2`, and so on.
- `scripts/wiki_lint` does not validate `#`-anchor links, so every Contents link must be checked to resolve to its heading in VS Code Markdown preview.

Example (question page):

```md
# Page Title (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [First answer subsection](#first-answer-subsection)
  - [Second answer subsection](#second-answer-subsection)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question
```

Migration note: existing content pages without a `## Contents` block remain valid and need not be changed until they are next substantially revised, at which point add the table of contents.

## MANDATORY Wiki Structure

- Keep version-specific pages under `wiki/vNN/`.
- Each `wiki/vNN/` root must contain `index.md` and the mandatory `codebase-navigation-guide.md`.
- Within each `wiki/vNN/`, file pages by `type:` into a per-type subdirectory:
  - `wiki/vNN/questions/` for `type: question` pages. A question page carries its own answer inline; see `MANDATORY Question Documents`.
  - `wiki/vNN/concepts/` for `type: concept` pages.
  - `wiki/vNN/answers/` holds legacy `type: answer` pages only. Do not file new answer pages there.
- The version landing page `wiki/vNN/index.md` and `wiki/vNN/codebase-navigation-guide.md` are the only Markdown pages allowed at the version root.
- Use page-relative Markdown links for wiki page navigation, e.g. `[v18/index](../index.md)` and `[versions](../../versions.md)` from a `wiki/v18/questions/` page. `scripts/wiki_lint` checks that local Markdown wiki links resolve and rejects Obsidian wikilinks for wiki page navigation.
- Include the version segment and the type subdirectory in links into per-version typed directories. The mandatory codebase navigation guide is the root-level exception, e.g. `wiki/v18/codebase-navigation-guide.md`.
- Create a page only when the work justifies it.
- Do not create standalone call-chain or source-trace document families.
- Treat generated pages as drafts until source references are checked.
- Use a unicode/ASCII tree for visual directory representations.

## MANDATORY Bookkeeping

After each meaningful wiki change:

- Update `wiki/index.md` for created or substantially changed pages.
- Update `wiki/versions.md` for supported-version lifecycle, repin, or meaningful coverage changes.
- Update `wiki/vNN/index.md` for created or substantially changed version-local pages.
- Append to `wiki/log.md` after scaffold changes, ingests, lint passes, filed answers, or version lifecycle events.
- Run `scripts/wiki_lint` after every wiki-facing edit, including small edits to existing pages, indexes, version pages, log entries, citations, titles, or front matter.

Log heading format:

```md
## [YYYY-MM-DD] <kind> v<NN> | <subject>
## [YYYY-MM-DD] <kind> | <subject>
```

## Core Workflows

### MANDATORY Add A Supported Version

1. Add the source checkout under `raw/postgres-NN/`.
2. Pin it to an exact commit.
3. Add it to `wiki/versions.md`.
4. Create `wiki/vNN/index.md`.
5. Create `wiki/vNN/codebase-navigation-guide.md`.
6. Update `wiki/index.md`.
7. Append to `wiki/log.md`.

### MANDATORY Answer And File

1. Assume the primary version unless the user specifies another.
2. Use `wiki/versions.md`, `wiki/index.md`, and the version landing page as navigation only.
3. Build the deep-inquiry context envelope from the pinned checkout.
4. Draft a claim-to-source map.
5. Move unverified claims to `## Open Questions`.
6. Answer with matching-version raw citations.
7. File the answer inline in the question page under `wiki/vNN/questions/` (`type: question`). Do not create a separate answer page; see `MANDATORY Question Documents`.
8. Include `## Context Reviewed`, `## Evidence Map`, and `## Open Questions` in filed pages when gaps exist.
9. Add the `## Contents` table of contents; see `MANDATORY Table of Contents`.
10. Update indexes and log.

## MANDATORY Lint

Lint is required after every wiki-facing change. Do not treat it as optional or only for new pages; run it before the final response whenever any wiki document changed.

Check broken links, orphan pages, missing source references, stale pins, wrong-version citations, invalid verification fields, unverified title hints, and version landing-page links.

Use the project venv:

```bash
scripts/recent_log --limit 20
scripts/wiki_lint
```

## MANDATORY Version Control

- Never commit or push without permission.

## MANDATORY Script Changes

- Keep durable project tooling under `scripts/`.
- Keep runtime state under `.wiki-runtime/`.
- When changing script contents, update adjacent workflow examples, lint examples, or tests that depend on the change.
- Keep top-level `run_*` files ignored. Do not edit, cite, or use them for wiki work unless the user explicitly names one.
