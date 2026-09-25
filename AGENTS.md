# PostgreSQL Engine Wiki Agent Instructions

This repo is an LLM-maintained wiki for PostgreSQL internals. The pinned PostgreSQL checkout under `raw/postgres-NN/` is the evidence base.

## MANDATORY Rule Precedence

When two rules in this file conflict, or a rule conflicts with the request, follow the first that applies:

1. The user's explicit instruction for the current task.
2. `MANDATORY Environment Isolation` and `MANDATORY Version Control`.
3. `MANDATORY Evidence` and `MANDATORY Citations`.
4. Document shape, writing style, and bookkeeping rules.

Name the conflict and the rule you followed in your response.

## MANDATORY Read First

- Read `wiki/versions.md` before modifying or answering from the wiki.
- Read `wiki/index.md`.
- Read `wiki/glossary.md` and review the entries relevant to the interaction; see `MANDATORY Shared Glossary`.
- Read the last ~20 entries of `wiki/log.md`.
- For version-local work, read `wiki/vNN/index.md`.
- Read the `wiki/vNN/common-concepts/` pages that cover concepts your work touches, and link them instead of re-explaining them.
- Use the matching `raw/postgres-NN/` checkout as the PostgreSQL evidence base.

## MANDATORY Environment Isolation

- Stay inside this repo.
- Read/write only `raw/`, `wiki/`, `.wiki-runtime/`, `scripts/`, `templates/`, `tests/`, `requirements.txt`, and top-level docs.
- Treat `raw/postgres-NN/` checkouts as read-only evidence.
- Run Python scripts from `.wiki-runtime/venv/`: activate it or call `.wiki-runtime/venv/bin/python scripts/<name>`.
- If the venv is missing, create it with `python3 -m venv .wiki-runtime/venv`. That command is the only permitted use of host `python3`.
- Pin new Python deps in `requirements.txt`.
- Do not install packages globally, with `--user`, via `pipx`, or with `sudo`.
- Do not use `sudo`, host-path `chown`, or host-path `chmod`.
- Use network only for venv setup from `requirements.txt` or user-requested source fetches.
- Do not use `WIKI_ALLOW_SYSTEM_PYTHON=1` in normal work.
- Keep generated artifacts, caches, and the venv under `.wiki-runtime/`.
- Run every subagent on the orchestrator's model. Do not assume a subagent inherits it: an agent type's own definition or a configured default subagent model can override inheritance. Set the model explicitly when you launch the subagent, or use a launch mode that always inherits the launcher's model, such as a fork. This covers foreground and background subagents, read-only exploration runs, implementation runs, and any agent those subagents launch in turn.
- If the orchestrator's exact model cannot be selected, use the most capable available model of the same family, and disclose the substitution before the subagent starts: name the orchestrator's model, the substitute, and why the exact model could not be used.
- Disclose and proceed when the substitute is equal or more capable. Stop and get the user's approval when it is weaker than the orchestrator's model.
- If no same-family model is available, do not substitute across families. Report it and ask, or keep the work in the orchestrator.
- Stop every service you started for wiki processing or document generation before your final response, whether the work succeeded, failed, or was abandoned mid-run. This covers PostgreSQL postmasters, standbys and replicas, connection poolers, background `psql` sessions, watchers, and any other daemon.
- Shut a cluster down cleanly with `pg_ctl -D <datadir> -m fast stop`, then confirm the teardown: no `postmaster.pid` in the data directory, no process whose command line names the data directory in `pgrep -f -- <datadir>`, and the socket directory and port free. Do not use `pgrep -a`: on macOS `-a` adds the caller's ancestors to the match list instead of printing command lines.
- Delete the sandbox you created under `.wiki-runtime/tmp/<name>/` once it is stopped, unless the user asked to keep it. Deleting it also deletes any build tree inside it, so a later run rebuilds. If the user asked to keep it, leave it stopped, name the retained path on the page or in the log entry, and say how to restart it.
- Never stop, kill, or delete a cluster, service, or data directory you did not start. Report the process and its data directory to the user and ask first.

## MANDATORY Evidence

- Use only the target version's pinned `raw/postgres-NN/` checkout as factual evidence.
- Treat implementation source as primary evidence.
- Same-checkout PostgreSQL docs, tests, and source history may support claims when directly relevant.
- Treat `wiki/versions.md`, `wiki/index.md`, `wiki/log.md`, and version landing pages as navigation only.
- Do not use model memory, external websites, external package docs, or uncited prior wiki prose as factual support.
- If implementation source conflicts with docs or tests, source wins. Put the discrepancy under `## Open Questions`.
- Never answer one PostgreSQL version with evidence from another version.
- The shared glossary may cite multiple pinned checkouts, but each definition or version-specific qualification must identify the version its evidence supports; see `MANDATORY Shared Glossary`. A glossary link never replaces matching-version source evidence on a consumer page.

## MANDATORY Prompt Hygiene

- If a user question or any prompt that drives document generation contains typos or grammatical errors, correct them without asking. Wherever a rule calls for a verbatim restatement, restate the corrected prompt.
- Record the original wording and each correction in the task's `wiki/log.md` entry. If the interaction has no log entry, list the corrections in your response.
- Stop and ask before drafting only when a correction could change the prompt's meaning, or when the user asked to keep the prompt as written.
- Apply this to filed `## Question` text, ingest prompts, and any prompt that will be restated verbatim in a wiki page.

## MANDATORY Review Requests

- When the user asks you to review a page or file and states no scope, finish the review and report the findings before changing anything. Then ask one question that settles the scope: report only, fix in place, or fix and re-run the measurements.
- "Just review", "report only", or "do not re-run" means report only: static checks, no edits, and no measurement runs.
- "Fix issues" after a report means fix every reported finding, including the measurement script, and re-run the measurement whenever a fix can change its numbers.
- A pasted review with its own fix order, or a request with its own numbered scope, is the scope. Do not ask about scope.
- A bare "continue" after an interrupted run means finish the whole remaining task.

## MANDATORY Deep Inquiry

Deep inquiry is the default unless the user explicitly asks for a quick answer.

- Confirm the target PostgreSQL version.
- Locate primary source files and symbols.
- Inspect adjacent callers, callees, structs, macros, includes, generated headers visible in raw source, reverse include users, tests, docs, catalogs, grammar, error paths, GUCs, and extension/contrib boundaries.
- If evidence lookup fails or is untrustworthy, stop before drafting. Fix it, rerun it, or report the target version and error.
- Inspect history when the user asks why, when intent matters, or when making a regression/change claim.
- For cross-version claims on a version-local page, support each other version's value or behavior through the target checkout's own git history. Name the commit by abbreviated hash, say what it changed, and cite the current code at the target pin. You may read another version's checkout to cross-check, but never cite it on that page. If the target history cannot establish a claim, for example because the checkout is a shallow clone, put the claim under `## Open Questions`.
- Draft from a claim-to-source map. Put unresolved claims under `## Open Questions`.
- Before drafting the explanation, map the values, information flow, branches, and lifecycle events required by `MANDATORY Technical Explanations`.
- Minimum engine answer: normal path, edge/error path, key data structures, caller/callee boundary, build/generated-header implications visible from raw source, and tests or explicit test absence.
- For planner, WAL, crash recovery, MVCC, storage, or corruption topics, missing caller/callee or data-structure context is a verification gap.

## MANDATORY Citations

- Cite every behavioral claim.
- Use page-relative Markdown links for source citations, not Obsidian wikilinks or bare raw-file links. Canonical shapes:

  ```md
  [file.c#Symbol](../../../../raw/postgres-NN/path/file.c#L42-L58)
  [file.c:42](../../../../raw/postgres-NN/path/file.c#L42)
  ```

  A source citation is a complete Markdown inline link, not a link fragment. The opening `[`, closing `]`, opening `(`, and closing `)` are mandatory.

  ```md
  [<label>](<relative-path-to-raw-file>#L<start>-L<end>)
  [<label>](<relative-path-to-raw-file>#L<line>)
  ```

  Invalid citation fragments include:

  ```md
  file.c#Symbol](../../../../raw/postgres-NN/path/file.c#L42-L58
  [file.c#Symbol]../../../../raw/postgres-NN/path/file.c#L42-L58
  raw/postgres-NN/path/file.c#L42-L58
  [[raw/postgres-NN/path/file.c]]
  ```

  When asked to describe the citation format, show complete Markdown inline-link examples. Do not omit the delimiter characters.

  - Link text: short human label, typically `file.ext#Symbol` (function, struct, macro, GUC, or doc-section name). Use `file.ext:line` for a single-line citation.
  - URL: page-relative path to the file in the matching `raw/postgres-NN/` checkout, with a `#Lstart-Lend` line-range fragment. Single-line citations use `#L42`.
  - Use enough `../` segments to make the link open from the current wiki page in VS Code. For root-level version pages such as `wiki/vNN/codebase-navigation-guide.md`, that prefix is `../../raw/postgres-NN/...`. For question pages under `wiki/vNN/questions/<category>/`, that prefix is `../../../../raw/postgres-NN/...`; see `MANDATORY Question Categories`. For common concept pages under `wiki/vNN/common-concepts/`, that prefix is `../../../raw/postgres-NN/...`; see `MANDATORY Common Concept Documents`.
  - New or edited source citations must use this page-relative format. `scripts/wiki_lint` may normalize repo-relative `raw/postgres-NN/...` URLs for validation, but that is compatibility behavior, not the citation style for new work.
  - Line numbers are stable because each version-local page pins an exact commit via `pinned_commit:`; the shared glossary records its per-version commits under `## Source Pins`. They jump correctly in VS Code and editors that understand Markdown line fragments.
- Include full extensions for non-Markdown files (`.c`, `.h`, `.sgml`, `.sql`, `.out`).
- Cite from the `raw/postgres-NN/` checkout matching the page `version:`. Never cite across versions on a version-local page. The shared glossary follows its explicit per-entry version scope and `## Source Pins` instead.
- Do not use the retired `[[raw/...]]` wikilink citation form.
- Page-to-page wiki navigation uses the same page-relative Markdown link syntax so it opens in plain VS Code Markdown preview. Do not use Obsidian wikilinks for wiki page navigation.
- Do not state a claim as fact unless it is backed by a source file, symbol, test file, documentation page, commit, or saved design discussion.
- Put uncertainty under `## Open Questions`.

Examples, as written from a question page such as `wiki/v18/questions/observability/explain-analyze-buffers-output.md`:

```md
[explain.c#ExplainOnePlan](../../../../raw/postgres-18/src/backend/commands/explain.c#L494-L598)
[instrument.h#BufferUsage](../../../../raw/postgres-18/src/include/executor/instrument.h#L24-L42)
[ref/explain.sgml#BUFFERS](../../../../raw/postgres-18/doc/src/sgml/ref/explain.sgml#L181-L208)
[bufmgr.c:4397](../../../../raw/postgres-18/src/backend/storage/buffer/bufmgr.c#L4397)
```

## MANDATORY Writing Style

- Lead with the answer.
- Write for a technically competent reader who does not yet understand this specific mechanism. Write as a tutorial, not as a compressed technical proof.
- Use plain language and short sentences.
- Use short paragraphs with one main causal idea each. Avoid sentences with several independent causal claims.
- Define terminology before relying on it. Introduce implementation details only when they become relevant.
- Link PostgreSQL jargon, acronyms, and advanced concepts on first substantive use to their entry in `wiki/glossary.md`; add or improve the entry when needed. Also link an existing matching-version common concept page for deeper explanation; see `MANDATORY Shared Glossary` and `MANDATORY Common Concept Documents`.
- Use active voice and name concrete subjects.
- Use lists, tables, diagrams, and small code blocks for dense material. For dependent steps, branches, interactions, or state changes, follow `MANDATORY Technical Explanations`.
- Name conditions precisely. Avoid vague hedges.
- Make causal relationships explicit: use language such as "because", "therefore", "so", "which means", "as a result", "this causes", and "this value is then used by" where it explains the connection.
- Preserve all important technical details, meaningful edge cases, formulas, and source citations. Do not remove an important qualification to make an explanation shorter.
- Repeat a fact only when the repetition helps connect two stages of the explanation. Prefer explicit relationships over elegant but compressed prose.
- Skip filler and setup prose.
- Cite every example and attach each citation to the exact claim it supports, including claims in diagrams and tables.
- Never trade citation precision for readability.

## MANDATORY Technical Explanations

Apply this rule when writing or rewriting technical material. Organize the explanation around the mechanism's causal structure, not the paragraph structure of the source material.

If an explanation contains a multi-step causal chain, branching logic, multiple interacting functions, state transitions, or several conditions that determine different outcomes, you MUST externalize that logic before explaining it in detailed prose. The short main idea and the concepts needed to read the map come first; the logic map comes before the walkthrough. Do not make the reader reconstruct control flow or a dependency graph from paragraphs.

Choose the representation that exposes the mechanism:

| Logic to explain | Required representation |
|---|---|
| Conditional or execution logic | Flowchart or decision tree, as a readable text diagram, or as Mermaid with its citations in a keyed list beside it, because links inside a Mermaid diagram are not clickable and a plain Markdown preview may show it as source |
| Linear causal chain | Sequence of numbered steps |
| Lifecycle events or state changes | State-transition table |
| Cases that differ along the same dimensions | Comparison table |
| Formulas and numerical values central to the result | Worked calculation |

Use more than one representation when needed. Show inputs, conditions, transformations, outputs, and their dependencies; a diagram that merely repeats function names is insufficient. Never compress complicated branching or causal logic into a dense paragraph when it can be represented structurally.

### Required process before drafting

Alongside the claim-to-source map, identify:

1. The main surprising behavior or result.
2. The variables, stored values, estimates, or state that matter.
3. The functions or components involved.
4. The order in which information flows between them.
5. Every important conditional branch.
6. Which values come from current/live state and which come from stored or potentially stale state.
7. The lifecycle events that alter those values.
8. The final causal chain that produces the observed behavior.

Then choose the structural representations and organize the explanation around that map. Preserve the evidence and qualifications while reorganizing material by causal role.

### Required explanation structure

Use the following sequence for a mechanism explanation. Include the conditional parts whenever the mechanism has the corresponding branches, lifecycle events, interactions, formulas, or numerical evidence; do not invent them to fill a section.

Fit the sequence into the existing document shape:

| Document | Placement |
|---|---|
| Question page or codebase navigation guide | Inside `## Answer`, with subsections as needed |
| Common concept page | Main idea in `## Definition`; mental model, logic map, and walkthrough in `## How It Works`; relevant interactions in `## Interactions with Other Concepts` |
| Shared glossary | Keep definitions concise and link deeper explanations; use a compact structural representation if an entry itself explains dependent steps or branches |

Keep the mandated headings and their order. Keep `## Measurement Script`, evidence sections, and navigation in their prescribed positions, and update `## Contents` for any new or changed subsections. This rule does not authorize changes to common concept pages during another document's work.

#### 1. Main idea

Start with a short explanation in plain language: what happens, why it is surprising, and the core reason it happens. Do not begin with low-level implementation details.

#### 2. Mental model

Introduce only the concepts needed to follow the mechanism. For each important value, explain what it represents, where it comes from, whether it is live/current or stored, and which later calculation uses it. A value table can make these relationships explicit:

| Value | Meaning | Source | Live/current or stored | Later use |
|---|---|---|---|---|

If similarly named values come from different sources, explicitly contrast their origins and explain why that distinction matters. Do not assume a stored value is stale merely because it is stored; state the condition that makes it stale.

#### 3. Logic map

Show the mechanism structurally before the detailed prose whenever any of the triggers above applies. Expose the decisions and what each branch supplies to the next stage. For a linear chain, numbered steps suffice; for branching logic, show the branches and their outcomes. Preserve important conditions and source citations in the representation or in a clearly keyed explanation immediately beside it.

#### 4. Step-by-step explanation

Walk through the mechanism in causal order. Each step should answer **Input → Decision → Calculation → Output → Consequence**, as applicable. Explain how that output becomes a later input. Prefer one main causal idea per paragraph and use explicit causal language so the reader knows why one step follows another.

#### 5. Branches and exceptional cases

Show the cases explicitly whenever conditions change behavior. Prefer this table shape:

| Condition / state | Behavior | Consequence |
|---|---|---|

Include meaningful edge and error cases. Do not bury several special cases inside a long sentence.

#### 6. Lifecycle or state changes

If creation, rebuild, reset, restart, migration, analysis, vacuuming, invalidation, caching, refresh, or another event changes relevant state, give those events their own table:

| Event | State before | State after | Effect on later calculation |
|---|---|---|---|

State which values change, which remain unchanged, and when later readers see the change.

#### 7. Key interaction

When the surprising behavior comes from subsystems using different values, explain the mismatch directly: "Component A uses X, while component B uses Y. Because X and Y are derived differently, ... Therefore ..." Name the actual components, values, and consequence.

State non-obvious cancellation, scaling, clamping, rounding, or other interactions explicitly. Do not expect the reader to notice them from a formula alone.

#### 8. Formula explanation

Define every term before showing a formula. Then show the formula and immediately explain its practical meaning in plain language. Preserve its conditions and qualifications. If two terms grow together and cancel or partially cancel, demonstrate that relationship explicitly, including any limits on the cancellation.

#### 9. Worked example

If the source material contains numerical evidence, reproduce the relevant calculation step by step. State where each number came from, separate the inputs from the calculation and the result, and explain what the result demonstrates. Keep measured values distinct from estimates and derived values, and cite their sources.

For numbers produced by running PostgreSQL, `MANDATORY Measurement Script` still applies. A worked example does not replace the published script, and a calculation does not establish the engine behavior behind it.

#### 10. Final causal summary

Finish the explanation with a concise causal chain: **initial state → input or event → transformation → interaction → observed result → relevant refresh or state change**. Use the actual mechanism and omit stages that do not apply. The reader should be able to understand the main mechanism from this summary alone, including the conditions that determine the result.

### Critical quality test

Before finalizing, ask:

- Would a reader need to draw a diagram, state table, or dependency graph themselves to understand this explanation? If yes, the explanation is unfinished: create that representation in the document or answer.
- Are multiple pieces of logic still compressed into a paragraph merely because they appeared together in the source? If yes, reorganize them by causal role.

Check these requirements by hand. A clean `scripts/wiki_lint` result does not establish that an explanation exposes its causal structure.

## MANDATORY GUC Changes

- When suggesting any GUC change, state whether it needs restart, reload, or only session/transaction scope.
- Determine the requirement from the same-version raw GUC definition or a validated `pg_settings` definition.
- Map contexts explicitly:

| `pg_settings.context` | Requirement |
|---|---|
| `postmaster` | restart |
| `sighup` | reload |
| `superuser`, `user` | session/transaction scope with `SET`; a `superuser` setting only for a role allowed to change it |
| `backend`, `superuser-backend` | new sessions only: set it at connection start, for example through `PGOPTIONS`, or in the configuration file followed by a reload, which only sessions started afterwards pick up. `SET` fails once a session has started. |
| `internal` | cannot be changed |

## MANDATORY Production SQL

- Verify production-bound SQL against the pinned checkout before filing.
- If syntax, catalogs, columns, functions, or GUCs cannot be verified, move the snippet under `## Open Questions`.
- Recommend reasonable session-scoped `statement_timeout` and `lock_timeout` values.
- Add an inline block-comment tag after the leading verb:

```sql
SELECT /* wiki_capture_plan_inputs */ ...;
UPDATE /* wiki_backfill_user_email */ users SET ...;
```

## MANDATORY Measurement Script

Any page that reports a number produced by running PostgreSQL must publish the script that produced it. This covers timings, block and page counts, byte sizes, densities, row and tuple counts, buffer counts, WAL volumes, and any other measured value, wherever on the page the number appears.

- File the script under one top-level `## Measurement Script` section, placed after `## Answer` and before `## Context Reviewed`, and list it in `## Contents`.
- Publish the script in full, in the page, inside a fenced block. Do not link to an uncommitted file, do not summarize it, and do not leave a reader to reassemble it from prose.
- Use Bash and SQL. Besides `bash`, the script may call `git`, the build toolchain (`configure`, `make`, the C compiler), the PostgreSQL programs it builds, and standard POSIX utilities such as `sed`, `grep`, `sort`, `cut`, `tr`, `wc`, `head`, and `tail`. No Python, no `awk`, no `perl`, no `jq`, no external harness. The `.wiki-runtime/venv/` Python is wiki tooling, not measurement tooling.
- Use only options and regular-expression syntax that POSIX specifies, so the script behaves the same with BSD (macOS) and GNU tools. For example, use `grep -E 'a|b'`, not `grep 'a\|b'`, and do not use `sed -i`.
- File one script per page. The only exception is a measurement with more than one version leg: file one script per leg, each in its own `###` subsection under the same `## Measurement Script` section, named for the version it runs.

Reuse and maintenance:

- Re-run the page's existing script instead of writing a new one. A second, parallel script for the same page is a defect.
- Edit the filed script in place when the statement, fixtures, thresholds, or stages change, then re-run it before filing the new numbers.
- Keep stages selectable and idempotent so a reviewer can re-run one stage without rebuilding everything: `bash <script>.sh <stage> [<stage> ...]`.
- Record the last run on the page: date, server version and pin, and the platform facts the numbers depend on, such as `block_size`, `max_data_alignment`, OS, and architecture.
- If the filed numbers predate the current script text, say so under `## Open Questions` and leave `verified_by_agent: not yet`.

Usage information is mandatory. The section must state:

| Item | What to give |
|---|---|
| Purpose | what the script measures, and which page claims its numbers back |
| Invocation | the exact command, and the directory it runs from |
| Stages | every stage name, the default order, and what each stage does |
| Environment | every variable the script reads, with its default |
| Prerequisites | build toolchain, configure flags, extensions, locale and encoding requirements |
| Output | where results land, and which file or table to read first |
| Runtime | roughly how long a full run takes, and how long a re-run takes while the build is still present (before the cleanup stage deletes it) |
| Cleanup | the stage or command that stops the server and deletes the sandbox |

Isolation and safety, on top of `MANDATORY Environment Isolation`:

- Treat `raw/postgres-NN/` as read only. Build out of tree and write every artifact under `.wiki-runtime/tmp/<name>/`.
- Run an isolated cluster with its own data directory, its own socket directory, and a non-default port. Never measure against a cluster the user did not name.
- Keep the sandbox name short. PostgreSQL refuses a Unix-socket path longer than the platform's `sun_path` buffer, which is about 103 usable bytes on macOS, and the repository path plus `.wiki-runtime/tmp/` already uses much of that.
- Start with `set -uo pipefail`, and call `psql` with `-X -v ON_ERROR_STOP=1` so a stray `~/.psqlrc` cannot change the result and no error passes silently.
- Follow `MANDATORY Production SQL` for the statements the script sends: the inline tag comment after the leading verb, and session-scoped `statement_timeout` and `lock_timeout`.
- Name the context and apply scope of every GUC the script sets, per `MANDATORY GUC Changes`.
- Mark fixture statements as disposable. They create and drop objects and are not meant for a database anyone cares about.
- Give the script a cleanup stage that stops the server and deletes the sandbox, and run that stage before your final response. Publishing the stage is not enough; see the teardown bullets in `MANDATORY Environment Isolation`.
- A script published inside a fenced block must not contain a literal Markdown fence. Assemble one at run time when the script has to read fenced blocks out of the page, e.g. `fence=$(printf '\140\140\140')`.

Evidence boundary:

- A measurement is evidence for what the built server did, not for why the engine does it. Every behavioral claim still needs a matching-version raw citation; see `MANDATORY Evidence` and `MANDATORY Citations`.
- When a measurement disagrees with the source reading, file the disagreement under `## Open Questions` instead of dropping either side.
- `scripts/wiki_lint` does not check for this section, its script, or its usage information. Check them by hand before filing.

Migration note: existing pages that report measured numbers without a `## Measurement Script` section remain valid and need not be changed until they are next substantially revised or re-measured, at which point add the section and the script. A prose-only reproduction recipe, such as a bare `### Reproduction` list of statements, does not satisfy this rule; turn it into a runnable script at that revision.

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

- Common concept pages use this exact front matter order and are read-only from other documents' work; see `MANDATORY Common Concept Documents`:

```yaml
type: common-concept
version: NN
pinned_commit: abc123...
verified: false
verified_by_agent: not yet
```

- The shared glossary uses `type: glossary`, `verified:`, and `verified_by_agent:` in that order. It has no single `version:` or `pinned_commit:`; its exact per-version pins belong in `## Source Pins`. See `MANDATORY Shared Glossary` for its full shape. The human-only verification rule and unverified title/link hints still apply.

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

The timestamp form is `<model-name> <ISO-8601-UTC>`: a single space separator, then a UTC timestamp ending in `Z`, e.g. `claude-opus-4-8 2026-06-06T14:30:00Z`. The model name must match `[a-zA-Z0-9_-]+` (no spaces or dots). `scripts/wiki_lint` enforces exactly this shape on version-local pages; it does not check the glossary's verification fields or title, so check those by hand. Use the exact current model name and the real verification time when filing an agent-verified page.

## MANDATORY Version Awareness

- `wiki/versions.md` is the source pin manifest.
- Each supported version has `wiki/vNN/index.md`.
- Each supported version has `wiki/vNN/codebase-navigation-guide.md`.
- All supported versions share exactly one `wiki/glossary.md`; never create a per-version glossary.
- Default new ingests and answers to the primary version unless the user specifies another.
- If the user omits a version, assume the primary version and state that assumption.
- Every source citation must use the matching `raw/postgres-NN/` checkout.
- Never use citations from another PostgreSQL version to support a claim about the target version. The shared glossary labels evidence by version rather than declaring one target version for the whole document.

## MANDATORY Shared Glossary

The wiki must have exactly one glossary at `wiki/glossary.md`, shared by every PostgreSQL version. It explains the vocabulary a reader who is not a PostgreSQL source-code developer needs to follow the wiki and the pinned source trees.

Scope and entries:

- Include PostgreSQL-specific jargon, acronyms, source-code terminology, and advanced database or systems concepts encountered in wiki work. Cover the terms needed to understand the current interaction rather than adding an unrelated vocabulary dump.
- Keep one canonical entry per term, with aliases and expanded acronyms in the same entry. Use alphabetical `###` term headings under `## Terms` and keep their anchors stable. If a term is renamed or merged, preserve its old anchor or repair every incoming link.
- Lead each entry with a short, plain-language definition. Explain why it matters when reading PostgreSQL source, name the relevant symbols or structures, and cite the evidence. Distinguish easily confused meanings and link related glossary entries when useful.
- State the PostgreSQL versions checked for each entry. Keep a shared definition where the evidence supports it, and place version-specific meanings or implementation differences inside that same entry with matching-version citations. Do not infer that a definition or behavior applies to every version merely because the glossary is shared.
- Keep entries concise and source-backed. Detailed engine walkthroughs, user questions, operational SQL, and measurements belong on their existing document types. Link matching-version common concept pages for deeper treatment when they exist; the glossary does not replace them.

Maintenance on every wiki interaction:

1. Review the glossary before answering, ingesting, creating, editing, reviewing, verifying, measuring, reorganizing, adding a version, or repinning. This also applies to wiki questions answered in chat without filing a page. If the glossary does not exist, create it as part of the next authorized wiki-content task.
2. Identify the jargon and advanced concepts the interaction uses. Check the relevant entries for missing definitions, unexplained acronyms, misleading wording, stale source links, and missing version qualifications.
3. Add or revise the affected entries in the same task. Glossary maintenance is mandatory and does not require a separate user request. Reuse existing entries; do not create duplicate glossaries or a new common concept page as a side effect.
4. Verify additions and corrections against the relevant pinned `raw/postgres-NN/` checkout. Put unresolved meanings or unsupported version applicability under `## Open Questions`; do not publish guesses as definitions. When a version is repinned, re-check every glossary citation and qualification for that version before updating its recorded pin.
5. Add or repair glossary links on the documents being worked on. Check term anchors, the glossary Contents, source references, and its pin table before finishing.
6. Record the glossary changes in the task's `wiki/log.md` entry. If the review finds nothing to change, record that outcome instead of making a cosmetic edit. For an interaction that otherwise needs no log entry, state the glossary review outcome briefly in the response; any actual glossary edit still requires normal bookkeeping and lint.

Links from wiki documents:

- Every content page other than the glossary itself must link the glossary in `## Navigation` and link relevant term entries on first substantive use in explanatory prose. Preserve verbatim `## Question` text and executable code blocks; put links in the surrounding explanation. A glossary navigation link alone does not replace useful term links.
- Link the glossary from `wiki/index.md`, `wiki/overview.md`, `wiki/versions.md`, and every `wiki/vNN/index.md`. Use `(unverified)` in navigation link text while the glossary's `verified:` is not `true`.
- Use complete page-relative Markdown links. From a categorized question page: `[MVCC](../../../glossary.md#mvcc)` and `[Wiki Glossary (unverified)](../../../glossary.md)`. From a common concept page use `../../glossary.md`; from a version-root guide or landing page use `../glossary.md`; from a wiki-root page use `glossary.md`.
- Check that each linked entry exists and that any version qualification fits the consumer page. A link supplies vocabulary, not proof of behavior: the consumer still needs its own matching-version raw citations.
- Existing pages gain the navigation link and relevant term links when next worked on. Do not mass-edit unrelated pages solely to backfill glossary links. Common concept pages gain links only during their own authorized work or a repin, preserving their read-only rule.

Evidence and document shape:

- Use `type: glossary`. This is the only shared glossary, not a question page or a version-local common concept page. Do not put it under `wiki/vNN/`, and do not copy its entries into per-version glossaries.
- Use implementation source as primary evidence and only the pinned checkouts listed in `wiki/versions.md`. Same-checkout docs, tests, and history may support definitions. Do not use model memory, external websites, or prior wiki prose as factual evidence.
- Cite every definition and behavioral qualification with a complete Markdown link. From this page the source prefix is `../raw/postgres-NN/`, for example `[file.c#Symbol](../raw/postgres-NN/path/file.c#L42-L58)`. Each citation must match the version stated for the claim it supports.
- Record every cited version and its full exact commit in `## Source Pins`, matching `wiki/versions.md`. A pin table entry does not imply that every term was checked on that version; each term carries its own checked-version scope.
- Use this front matter, in this exact order, when creating the document:

```yaml
type: glossary
verified: false
verified_by_agent: not yet
```

- Use these required headings in order, with a Contents entry for every `##` and `###` heading except Contents itself:

```md
# Wiki Glossary (unverified)

## Contents
## Scope
## Source Pins
## Terms
## Open Questions
## Source References
## Navigation
```

- Keep `## Terms` and `## Source References` non-empty. `## Scope` explains the audience, shared-document scope, and per-entry version qualifications. `## Navigation` links the global index and version manifest.
- Preserve the human-only `verified:` field. After changing definitions or their evidence, set `verified_by_agent: not yet` unless every glossary claim has been re-checked against its recorded pins. Updating a few entries does not justify an agent-verification timestamp for the entire glossary.
- Run `scripts/wiki_lint` after glossary edits. Its current checks do not enforce the glossary's existence, type, shape, pin table, per-entry evidence scope, required inbound links, or term anchors; verify those by hand rather than treating a clean lint result as complete verification.

## MANDATORY Codebase Navigation Guide

Every supported version must have one codebase navigation guide:

```text
wiki/vNN/codebase-navigation-guide.md
```

- Use `type: codebase-navigation-guide`.
- File it at the version root, not under `questions/`, `common-concepts/`, or `answers/`.
- Treat it as version-local content and as a question-style document. It must have front matter, a `## Contents` table of contents, `## Question`, inline `## Answer`, matching-version raw source citations, `## Context Reviewed`, `## Evidence Map`, `## Open Questions`, `## Source References`, and `## Navigation`.
- Apply every `MANDATORY Question Documents` rule unless it conflicts with the fixed `type:` or fixed root-level path for this guide.
- If a user-requested guide prompt exists, restate that prompt verbatim under `## Question` after applying `MANDATORY Prompt Hygiene`. If the guide is generated as a mandatory per-version scaffold without a user prompt, use this canonical question text: `Create a codebase navigation guide document for PostgreSQL NN.`
- The guide must orient readers to the pinned checkout's source layout, normal SQL statement path, utility-command path, generated files and catalog/parser/header implications, key data structures, extension/contrib boundaries, tests, and docs.
- Update the guide when adding a supported version, repinning a supported version, or making a meaningful source-tree coverage change that affects codebase navigation.
- Link the guide from `wiki/index.md` and the matching `wiki/vNN/index.md`.

## MANDATORY Question Documents

When a user asks a question, the deliverable is a single `type: question` page that holds both the question and its answer. Do not spin off a separate answer document.

- File the page under `wiki/vNN/questions/<category>/`, using a category from `MANDATORY Question Categories`. Never file a question directly under `wiki/vNN/questions/`.
- Restate the user prompt verbatim under `## Question`, with any corrections made under `MANDATORY Prompt Hygiene`.
- Put the full answer, with matching-version raw citations, inline under `## Answer`.
- Add `## Measurement Script` when the page reports a measured number; see `MANDATORY Measurement Script`.
- Always include `## Context Reviewed`, `## Evidence Map`, and `## Open Questions` on the same page. When nothing is open, say so under `## Open Questions`.
- Scaffold a new page from `templates/question.md`.

Why one document per question, not a question page plus an answer page:

- A user's question is the canonical entry point. The answer belongs with the question that motivated it, so a reader sees both without a second hop.
- One page per Q&A removes duplicate, drifting pages. A separate answer document forces two titles, two `(unverified)` hints, two `pinned_commit:` values, and two index entries to keep in sync.
- It keeps verification honest. One page carries one `verified:` / `verified_by_agent:` state over one claim-to-source map, instead of a question page that silently goes stale against its answer page.

`type: answer` is retired, and no answer page remains. Do not file one.

## MANDATORY Question Categories

Every `type: question` page lives in exactly one category directory:

```text
wiki/vNN/questions/<category>/<page>.md
```

A question page filed directly under `wiki/vNN/questions/` is misfiled. There is no uncategorized location.

The category set is closed. Use exactly these six directory names, in every version:

| Directory | Covers |
|---|---|
| `query-planning` | Planner and optimizer behavior: cost and selectivity estimation, the statistics the planner reads, join and scan strategy, partition pruning, plan caching and re-planning, plan shape, and planner-visible expression and routine semantics. |
| `indexing` | Index access methods and their on-disk structure: index builds and rebuilds, index-specific DDL, bloat and page density, index-only scans, and per-AM behavior such as B-tree, GIN, GiST, BRIN, and hash. |
| `storage-and-vacuum` | How rows and pages are stored and reclaimed: heap and page layout, TOAST, the buffer manager, the free space and visibility maps, VACUUM and autovacuum, REPACK and CLUSTER, and MVCC bookkeeping such as MultiXact and the xid horizon. |
| `replication-and-wal` | WAL generation and replay: checkpoints, crash recovery, archiving, and physical and logical replication including slots, origins, and apply workers. |
| `observability` | How the server reports on itself: cumulative and dynamic statistics views, `pg_stat_statements`, EXPLAIN instrumentation output, progress reporting, wait events, and logging. |
| `server-administration` | The server as an installation rather than as a query: GUC configuration, security and access control, extension and hook surfaces, contrib inventory, client tools, and operational runbooks. |

Pick the category in this order and stop at the first rule that decides:

1. File by the subsystem the question is *about*, not by the tool used to observe it. B-tree page density is `indexing` even when `pgstatindex` supplies the numbers; what the statistics interface itself exposes is `observability`.
2. If two categories still fit, file under the subsystem that supplies most of the page's source citations. Vacuum extension hooks are `storage-and-vacuum` because the cited symbols are vacuum and autovacuum code.
3. If it is still tied, take the first matching category in the table order above.

Category rules:

- One page, one category directory. Do not copy, symlink, or re-file the same page under a second category. When a page matters to a second category, link it from that category's group on the version landing page.
- The same question basename uses the same category in every version, so `wiki/v12/questions/indexing/create-index-concurrently.md` and `wiki/v18/questions/indexing/create-index-concurrently.md` stay parallel and cross-version comparison is a directory diff.
- Create a category directory when its first page is filed. Do not create empty category directories.
- Never invent a category name. Adding, renaming, or removing a category is a repo-wide change: update this section, move every affected page, fix the page-relative links in and to those pages, update `wiki/index.md`, every `wiki/vNN/index.md`, and `wiki/versions.md`, then append to `wiki/log.md` and run `scripts/wiki_lint`.
- `scripts/wiki_lint` does not check category placement, so verify the directory name against the table above before filing.

Index grouping:

- `wiki/vNN/index.md` groups its `## Questions` list under one `### <Category label>` heading per non-empty category, in the table order above.
- `wiki/index.md` uses the same grouping one level deeper, as `#### <Category label>` under each `### PostgreSQL NN` section.
- Category labels are the title-cased directory name: Query Planning, Indexing, Storage and Vacuum, Replication and WAL, Observability, Server Administration.

## MANDATORY Common Concept Documents

A common concept document is the wiki's detailed explanation of one PostgreSQL concept for one version. Other pages link it instead of repeating that explanation. The shared glossary supplies concise vocabulary across versions; common concept pages retain their version-local depth and read-only rules.

- Use `type: common-concept`.
- File it at `wiki/vNN/common-concepts/<concept-slug>.md`.
- Create a concept page only when the user asks for that page. It is never a side effect of another document and never a mandatory per-version scaffold, unlike `MANDATORY Codebase Navigation Guide`. A version with no concept page is a valid state.
- Common concept pages are not categorized. `MANDATORY Question Categories` does not apply to them.
- One page per concept per version. The page is pinned to that version like any other page, and every citation comes from the matching `raw/postgres-NN/` checkout.
- Use the same basename for the same concept in every version, so `wiki/v17/common-concepts/visibility-map.md` and `wiki/v18/common-concepts/visibility-map.md` stay parallel.
- Name the page for the concept, not for a question: `visibility-map.md`, `multixact.md`, `shared-buffer-mapping.md`.
- Create the `common-concepts/` directory when its first page is filed. Do not create it empty.

What belongs on a concept page, one concept per page:

- What it is, in the lead, in one paragraph.
- Why the engine has it.
- How it works on the normal path, plus the edge and error paths that change its meaning.
- Where it lives in the pinned source tree.
- The structs, macros, functions, catalogs, and GUCs that carry it.
- Where it meets neighboring concepts, and where that boundary is.

What does not belong on a concept page:

- A user's question or its answer. That is a `type: question` page, and it links the concept page.
- Measured numbers. A concept page is source-only, so `MANDATORY Measurement Script` never applies to it. Measurements belong on the question page that ran them.
- A list of the pages that use the concept. Links run from consumer to concept only, never back, so adding a consumer never edits the concept page.
- Cross-version comparison. Each version's page describes that version; a "changed in NN" answer is a question page.

Read-only from other work. A common concept page changes only when the user asks for a change to that concept page:

- While creating or editing any other document, do not create, edit, rename, re-scope, split, merge, or delete a common concept page. Use it as a source and link it.
- Filing, revising, or verifying a question page or a navigation guide is never a reason to touch a concept page, not to add a backlink, not to fix wording, not to widen a definition so it covers the new page.
- If a concept page is wrong, incomplete, or missing for the document you are writing, do not fix it in passing. Write what the current document needs on the current document, record the gap under that document's `## Open Questions`, and tell the user the concept page needs its own change.
- A change to a concept page is its own task, with the user's go-ahead, its own `wiki/log.md` entry, and a re-read of the pages that link it.
- Repinning a version is the only exception. A repin rewrites `pinned_commit:` and citation line ranges across that version's pages, concept pages included, as part of the repin task.

Use the concept layer when you file a document:

- Before drafting any page, list the concepts the answer leans on, then read `wiki/vNN/common-concepts/` for that version and link the pages that already cover them.
- Link the glossary entry on first substantive use of the term and also link the matching-version concept page for its deeper explanation; see `MANDATORY Writing Style`. Keep the consumer page's own coverage to what its question needs.
- Link only your own version's concept page. If the concept has no page for that version, explain what the consumer page needs inline and do not link another version's page.
- When a document needs a concept that has no page yet, say so in your response and propose the page. Do not create it as a side effect of the other document's work.

Evidence boundary:

- Every claim on a concept page cites matching-version raw source, like on any other content page.
- A consumer page may link a concept page instead of repeating the explanation, but every behavioral claim the consumer page makes still needs its own matching-version raw citation. `MANDATORY Evidence` still forbids uncited wiki prose as factual support.
- A concept page never cites a wiki page as evidence, including another concept page. It links wiki pages for navigation only.

Concept page shape. Front matter, in this exact order:

```yaml
type: common-concept
version: NN
pinned_commit: abc123...
verified: false
verified_by_agent: not yet
```

Required headings, in this order and with this exact text:

```md
# Concept Name (unverified)

## Contents
## Definition
## Why It Exists
## How It Works
## Where It Appears in Source
## Related Structures and Functions
## Interactions with Other Concepts
## Open Questions
## Source References
## Navigation
```

- Scaffold a new page from `templates/common-concept.md`, which carries this front matter and these headings.
- `## Definition` must be non-empty and must lead with the definition.
- Add `## Context Reviewed` and `## Evidence Map` before `## Open Questions` when the claim-to-source map is large.
- Do not drop or rename a required heading. `scripts/wiki_lint` matches each one by exact text.
- `MANDATORY Table of Contents`, `MANDATORY Citations`, `MANDATORY Writing Style`, and `MANDATORY Verification Fields` apply unchanged. From `wiki/vNN/common-concepts/`, the citation prefix is `../../../raw/postgres-NN/...`.

Bookkeeping and lint:

- `wiki/vNN/index.md` lists its concept pages under a `## Common Concepts` section, placed after `## Questions`.
- `wiki/index.md` lists them under a `#### Common Concepts` heading inside that version's `### PostgreSQL NN` section, after the question-category groups.
- `scripts/wiki_lint` checks front matter presence and key order, `version:` and `pinned_commit:` against `wiki/versions.md`, citations from the matching checkout only, complete Markdown citation form, a non-empty `## Source References`, the required headings above, a non-empty `## Definition`, the `(unverified)` title hint, and that `type: common-concept` and `wiki/vNN/common-concepts/` always pair.
- Lint cannot check that a consumer links an existing concept page, that a concept page stays source-only, or that another document's work left concept pages untouched. Check those by hand.

`type: concept` under `wiki/vNN/concepts/` is retired and replaced by this type. Do not file one. If an old-style concept page appears, refile it as `type: common-concept` under `wiki/vNN/common-concepts/`, fix the links into it, and log the move.

## MANDATORY Table of Contents

- Every content page must open with a `## Contents` table of contents: `type: question`, `type: codebase-navigation-guide`, `type: common-concept`, and `type: glossary` pages, regardless of page length.
- Navigation pages are exempt: `wiki/index.md`, `wiki/versions.md`, `wiki/log.md`, `wiki/overview.md`, and the `wiki/vNN/index.md` version landing pages.
- Place the `## Contents` block between the page title (`# ...`) and the first content section. On a question-style page, including `type: codebase-navigation-guide`, that means immediately before `## Question`.
- List every `##` and `###` section in document order as a nested Markdown bullet list: each `##` is a top-level bullet and its `###` subsections are indented two spaces beneath it. Do not list `####` or deeper headings.
- The `## Contents` section never lists itself.
- Keep it in sync with the page: update the table of contents whenever a `##`/`###` section is added, removed, renamed, or reordered.
- Link with page-internal Markdown anchors `[Section title](#slug)`. Build each `#slug` with the VS Code Markdown-preview rule: trim, lowercase, replace each whitespace run with `-`, then strip punctuation such as backticks and em dashes (`—`), but preserve underscores. So `### The three pg_index state flags` -> `#the-three-pg_index-state-flags`, `### How maintenance_work_mem is used` -> `#how-maintenance_work_mem-is-used`, and `## Open Questions` -> `#open-questions`. Suffix later duplicate slugs with `-1`, `-2`, and so on.
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

- Keep the mandatory shared glossary at `wiki/glossary.md`, with `type: glossary`. It is the single vocabulary reference for the entire wiki; see `MANDATORY Shared Glossary`.
- Keep version-specific pages under `wiki/vNN/`.
- Each `wiki/vNN/` root must contain `index.md` and the mandatory `codebase-navigation-guide.md`.
- Within each `wiki/vNN/`, file pages by `type:` into a per-type subdirectory:
  - `wiki/vNN/questions/<category>/` for `type: question` pages. A question page carries its own answer inline; see `MANDATORY Question Documents`. The category directory is mandatory; see `MANDATORY Question Categories`.
  - `wiki/vNN/common-concepts/` for `type: common-concept` pages. Concept pages are not categorized, and other documents only read them; see `MANDATORY Common Concept Documents`.
- Do not create `wiki/vNN/answers/` or `wiki/vNN/concepts/`: `type: answer` and `type: concept` are retired.
- The version landing page `wiki/vNN/index.md` and `wiki/vNN/codebase-navigation-guide.md` are the only Markdown pages allowed at the version root.
- `wiki/vNN/questions/` itself holds only category directories, never Markdown pages.
- Use page-relative Markdown links for wiki page navigation, e.g. `[v18/index](../../index.md)` and `[versions](../../../versions.md)` from a `wiki/v18/questions/<category>/` page. `scripts/wiki_lint` checks that local Markdown wiki links resolve and rejects Obsidian wikilinks for wiki page navigation.
- Include the version segment, the type subdirectory, and the question category in links into per-version typed directories, e.g. `wiki/v18/questions/indexing/create-index-concurrently.md`. The mandatory codebase navigation guide is the root-level exception, e.g. `wiki/v18/codebase-navigation-guide.md`.
- Create a page only when the work justifies it.
- Do not create standalone call-chain or source-trace document families.
- Treat generated pages as drafts until source references are checked.
- Use a unicode/ASCII tree for visual directory representations.

## MANDATORY Bookkeeping

After each meaningful wiki change:

- Maintain the shared glossary and record the review outcome; see `MANDATORY Shared Glossary`. Review is required even when the interaction produces no document change.
- Update `wiki/index.md` for created or substantially changed pages.
- Update `wiki/versions.md` for supported-version lifecycle, repin, or meaningful coverage changes.
- Update `wiki/vNN/index.md` for created or substantially changed version-local pages.
- Append to `wiki/log.md` after scaffold changes, ingests, filed answers, measurement runs, version lifecycle events, or a standalone lint run the user asked for. Record the lint result in that entry.
- Record teardown in the `wiki/log.md` entry for any step that ran a service: what was stopped, what was deleted, what was kept and where, or that nothing was running.
- Run `scripts/wiki_lint` last, after the log entry is written, so the final run covers every edit including the log; see `MANDATORY Lint`.

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
6. Review the shared glossary for the new version, adding source-backed terms or version qualifications as needed; link it from the new landing page and guide.
7. Update `wiki/index.md`.
8. Append to `wiki/log.md` and run `scripts/wiki_lint`.

### MANDATORY Answer And File

1. Assume the primary version unless the user specifies another.
2. Use `wiki/versions.md`, `wiki/index.md`, and the version landing page as navigation only.
3. Build the deep-inquiry context envelope from the pinned checkout.
4. List the jargon and concepts the answer leans on, review `wiki/glossary.md`, and read `wiki/vNN/common-concepts/` for that version. Maintain the relevant glossary entries and plan links to both vocabulary and existing deeper explanations; see `MANDATORY Shared Glossary` and `MANDATORY Common Concept Documents`.
5. Draft a claim-to-source map and the causal map required by `MANDATORY Technical Explanations`.
6. Move unverified claims to `## Open Questions`.
7. Answer with matching-version raw citations, following `MANDATORY Technical Explanations` and linking the concept pages instead of re-explaining their concepts. Do not edit a concept page as part of this work.
8. If the page reports a measured number, run the page's script and file it under `## Measurement Script`, then run the script's cleanup stage so no server or sandbox is left behind; see `MANDATORY Measurement Script`.
9. File the answer inline in the question page under `wiki/vNN/questions/<category>/` (`type: question`). Choose the category with `MANDATORY Question Categories`. Do not create a separate answer page; see `MANDATORY Question Documents`.
10. Include `## Context Reviewed`, `## Evidence Map`, and `## Open Questions` in every filed page; see `MANDATORY Question Documents`.
11. Add the `## Contents` table of contents; see `MANDATORY Table of Contents`.
12. Check glossary term links and the glossary link in `## Navigation`, then update indexes and log. Name any missing or wrong common concept page in your response instead of changing it; glossary maintenance remains part of this task.

### MANDATORY File Or Change A Common Concept Document

Run this workflow only when the user asks for the concept page itself. Never as a step inside another document's work.

1. Confirm the target version and the exact concept boundary with the user.
2. Read `wiki/vNN/common-concepts/` for that version to check the concept has no page and no overlapping page. Review the shared glossary's related entries so its concise definitions and the concept page's deeper explanation agree.
3. Build the deep-inquiry context envelope from the pinned checkout and draft a claim-to-source map. Map the mechanism and structure its explanation according to `MANDATORY Technical Explanations`, within the required common concept headings.
4. File or edit `wiki/vNN/common-concepts/<concept-slug>.md` with `type: common-concept`, the required headings, and matching-version raw citations only; see `MANDATORY Common Concept Documents`.
5. Keep it source-only. Move anything unresolved to `## Open Questions`, and leave measurements to the question page that ran them.
6. Re-read the pages that link the concept page and report, without editing them, any consumer the change now contradicts.
7. Maintain the relevant glossary entries and add the concept page's term and glossary navigation links. Link the concept page from `wiki/vNN/index.md` under `## Common Concepts` and from `wiki/index.md`, then append to `wiki/log.md` and run `scripts/wiki_lint`.

## MANDATORY Lint

Lint is required after every wiki-facing change, including small edits to existing pages, indexes, version pages, log entries, citations, titles, or front matter. Do not treat it as optional or only for new pages. Run it after the task's log entry is written and before the final response. If it reports errors, fix them, update the lint result in the log entry, and run it again.

Check broken links, orphan pages, missing source references, stale pins, wrong-version citations, invalid verification fields, unverified title hints, required common-concept sections, and version landing-page links.

Also perform the manual glossary checks in `MANDATORY Shared Glossary`; the current linter does not enforce those requirements.

Use the project venv:

```bash
.wiki-runtime/venv/bin/python scripts/wiki_lint
```

## MANDATORY Version Control

- Never commit or push without permission.
- When the user asks for a commit, commit on `master` and push to `origin/master`. Another host pushes to the same branch, so fetch and rebase before pushing. Resolve `wiki/log.md` and `wiki/versions.md` conflicts by re-applying your own edits on top of the incoming text.
- Finder `.DS_Store` files are tracked on purpose. Commit changes to them separately, as `chore: update workspace metadata`.

## MANDATORY Script Changes

- Keep durable project tooling under `scripts/`, and track every durable script in git. One-off helpers, such as a script that edits a single page, belong under `.wiki-runtime/tmp/` and are deleted with it.
- Keep runtime state under `.wiki-runtime/`.
- Measurement scripts are page content, not `scripts/` tooling. File them in the page that reports their numbers; see `MANDATORY Measurement Script`.
- When changing script contents, update adjacent workflow examples, lint examples, or tests that depend on the change.
- Keep top-level `run_*` files ignored. Do not edit, cite, or use them for wiki work unless the user explicitly names one.
