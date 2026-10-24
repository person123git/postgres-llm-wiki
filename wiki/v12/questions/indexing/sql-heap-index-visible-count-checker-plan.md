---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# SQL Heap and Index Visible-Count Checker in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Feasibility and execution contract](#feasibility-and-execution-contract)
  - [Values and decision flow](#values-and-decision-flow)
  - [Scope and catalog inventory](#scope-and-catalog-inventory)
  - [Index eligibility and ordering generation](#index-eligibility-and-ordering-generation)
  - [Snapshot permissions and DDL coordination](#snapshot-permissions-and-ddl-coordination)
  - [Settings and query shapes](#settings-and-query-shapes)
  - [Capturing an exact count and its executed plan](#capturing-an-exact-count-and-its-executed-plan)
  - [Mandatory plan validation](#mandatory-plan-validation)
  - [Result records and aggregation](#result-records-and-aggregation)
  - [Concurrency performance and correctness limits](#concurrency-performance-and-correctness-limits)
  - [Heap-side loss and the unresolved-TID signal](#heap-side-loss-and-the-unresolved-tid-signal)
  - [Stronger SQL diagnostics and pageinspect](#stronger-sql-diagnostics-and-pageinspect)
  - [Executable SQL installation](#executable-sql-installation)
  - [How the runner talks to psql](#how-the-runner-talks-to-psql)
  - [Runner branches and failure handling](#runner-branches-and-failure-handling)
  - [Running the runner](#running-the-runner)
  - [Test strategy](#test-strategy)
  - [Measured coverage and remaining test gaps](#measured-coverage-and-remaining-test-gaps)
  - [Optional relaxations](#optional-relaxations)
  - [Runner script](#runner-script)
- [Measurement Script](#measurement-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question
Follow AGENTS.md. In PostgreSQL 12, propose a detailed implementation plan for a PostgreSQL checker that uses only SQL, with pageinspect if necessary, to determine whether every eligible physical relation in a specified schema has a visible heap tuple count matching the count returned through at least one of its non-partial indexes.

Do not implement the checker yet. Identify feasibility limits and distinguish reliable checks from best-effort approaches.

Objective
For each physical relation T under a consistent MVCC snapshot S:
- H(T,S): visible row count obtained through a verified sequential heap scan.
- I(T,J,S): visible row count obtained through a verified full scan of eligible index J.
- PASS if at least one eligible index J satisfies H(T,S) = I(T,J,S).

Explain that matching counts do not prove complete index correctness: missing rows and duplicate results could cancel out, and passing one index does not validate the others.

Constraints
- Use SQL and built-in PostgreSQL capabilities.
- Use pageinspect only if necessary, preferably for mismatch diagnosis.
- Do not assume access to external scripts, planner-hint extensions, or other extensions.
- Do not silently introduce PL/pgSQL. If literal SQL cannot automate part of the process, explain the limitation and propose a generated SQL script. Describe a PL/pgSQL wrapper only as an optional relaxation.
- State PostgreSQL version assumptions and identify version-dependent behavior.
- Verify technical claims against official PostgreSQL documentation.

1. Scope and inventory
Design an inventory using pg_namespace, pg_class, pg_index, and pg_am.

Include:
- Ordinary heap tables.
- Physical table partitions.
- Populated heap-backed materialized views.

Requirements:
- Check each physical relation independently.
- Avoid counting inherited descendants in a parent’s baseline.
- Treat partitioned parents as summaries of their children.
- Explicitly define whether partitions outside the requested schema are included.
- Classify views, foreign tables, sequences, unpopulated materialized views, and unsupported table access methods rather than silently skipping them.

2. Index eligibility
Start with non-partial B-tree indexes:
- indpred IS NULL.
- indisvalid, indisready, and indislive are true.
- Account for planner usability under the checking snapshot, including indcheckxmin.

Explain why “non-partial” alone is insufficient for supporting all index access methods. Defer unsupported access methods explicitly.

Address expression indexes, collations, operator classes, NULL ordering, and other properties relevant to generating a compatible full index scan.

3. Snapshot, permissions, and concurrency
Plan for:
- REPEATABLE READ execution.
- No application-data writes by the checker.
- Either one schema-wide transaction or one transaction per relation, explaining the trade-offs.
- A role that can see every row.
- row_security = off as a guard against silently checking an RLS-filtered subset, not as a permission bypass.
- Required privileges, DDL coordination, lock behavior, and operational costs of long-lived snapshots.

4. Heap baseline
Describe how to:
- Enable sequential scans.
- Disable index, index-only, and bitmap scans.
- Disable parallelism initially to simplify plan validation.
- Count all visible rows in the intended physical relation.
- Capture and validate the executed access path.

Accept the baseline only when it uses an unfiltered sequential scan of the intended relation.

5. Index-driven count
Describe how to:
- Enable ordinary index scans.
- Disable index-only and bitmap scans.
- Discourage sequential scans.
- Generate a full-table counting query that encourages a candidate B-tree index, for example through a compatible inner ordering.
- Include every row, including rows with NULL keys.
- Avoid restricting predicates and limits.
- Capture and validate the executed plan.

Do not claim that planner settings or ORDER BY force a particular index.

Accept an index result only if the executed plan demonstrates a full ordinary Index Scan on an eligible index. Record the actual index used, even if it differs from the intended candidate.

Explain concretely how SQL can obtain the count and evidence of its executed access path, including any limitations of EXPLAIN ANALYZE, dynamic SQL, and separate planning/execution.

Stop after the first match unless exhaustive checking is requested. If the planner repeatedly selects the same index, mark untested candidates as untested rather than claiming coverage.

Explain the rationale for disabling index-only scans: this check should consult heap visibility instead of relying on visibility-map shortcuts.

6. Result model
Define:
- PASS: at least one verified eligible index scan matched.
- MISMATCH: all eligible candidates were successfully checked and none matched.
- INCONCLUSIVE: no match was found and at least one eligible candidate could not be checked.
- NO_ELIGIBLE_INDEX: no supported eligible index exists.
- ERROR: execution, permission, or inspection failure.
- NOT_APPLICABLE: the object is outside the physical-heap scope.

Record:
- Relation and index OIDs and names.
- Snapshot information.
- Heap and index counts.
- Count difference.
- Executed plans.
- Duration.
- Candidate coverage and errors.

Schema-level success must require every in-scope physical relation to pass. Define how errors and unsupported objects affect the overall report.

7. pageinspect and stronger diagnostics
Explain why directly counting heap_page_items or B-tree page items is not equivalent to counting MVCC-visible rows.

Discuss:
- Page-level physical consistency versus a relation-wide MVCC snapshot.
- HOT chains and redirects.
- Dead entries, B-tree posting lists, and concurrent physical changes.
- Why direct equality between raw index TIDs and visible tuple ctids is insufficient.

Keep physical inspection separate from the main executor-based comparison.

As an optional stronger SQL-only diagnostic, propose materializing visible ctids from both verified scan paths and comparing them with EXCEPT ALL in both directions, under the same snapshot. Use relation identity alongside ctid if results span multiple physical relations.

8. Tests and deliverables
Provide a test matrix covering:
- Empty tables.
- NULL keys and duplicate keys.
- Expression indexes.
- HOT and non-HOT updates.
- Deletes and aborted transactions.
- Concurrent writes.
- Partitions and inheritance.
- RLS and insufficient permissions.
- Tables with only partial or unsupported indexes.
- Cases where the planner refuses the desired access path.

Deliver:
1. A feasibility assessment for literal SQL-only execution.
2. A step-by-step implementation architecture.
3. Proposed query and plan-validation strategies.
4. A result schema and aggregation rules.
5. Concurrency, performance, and correctness limitations.
6. A test strategy.
7. Clearly separated optional extensions.

Prioritize executor-based counts with mandatory access-path validation. Do not attempt to reimplement PostgreSQL MVCC visibility using pageinspect unless you establish why that is necessary and feasible. Review the plan first, then implement only after user confirmation.

Follow-up (2026-10-23):

Use Bash and SQL for the script.

> Historical scope (superseded by the refactor below): before drafting, the asker chose two readings. "The script" is a runner that performs the operator's part of the checker unattended, not only the measurement harness. Bash makes every decision, and psql receives only SQL statements, plus `\echo` sync lines. The checker SQL stays the single source of the checks, the harness now tests the runner, and the harness's embedded C port probe is replaced by Bash.

Follow-up (2026-10-23):

Follow AGENTS.md. In PostgreSQL 12, for the question "SQL Heap and Index Visible-Count Checker in PostgreSQL 12 (unverified)", propose a plan to simplify "Executable SQL installation". Put all logic in a Bash script and try to reduce the complexity of the SQL.

Implementation approval: Implement the plan.

> Current scope: one self-contained Bash checker owns all checking decisions. SQL remains for catalog reads, native quoting/deparsing, same-execution exact-count capture and JSON decoding. This supersedes the historical requirement that the installed SQL remain the source of all checks.

Follow-up (2026-10-23):

Review the "Runner script" section and make its code more readable and simpler.

> Scope of that readability revision: the checking contract stayed the same; its SQL differed only by one redundant join removed from the plan-decoding query.

Follow-up (2026-10-24):

Fix all issues.

> Scope: fix every finding from the PostgreSQL 12 review, including the runner, measurement harness and affected descriptions, and rerun the measurements. This adds complete multiline-value handling, consistent text encoding and per-invocation server ownership to the same checking contract.

Follow-up (2026-10-24):

Follow AGENTS.md. In PostgreSQL 12, review the question page "SQL Heap and Index Visible-Count Checker in PostgreSQL 12 (unverified)". Find possible false negatives and edge cases. Propose a plan to fix the issues, but do not fix them.

Implementation approval: Implement the plan.

> Scope: the review found five false negatives and six edge cases. The asker chose a new relation status for anomalies, SQL-only unresolved-TID counters with documentation for heap-side loss, and one physical fault-injection leg in the harness. This adds `PASS_WITH_ANOMALY` to the six result statuses above, refuses server settings that hide damage, detects identity changes after the snapshot, and stops on any unreadable local evidence.

Follow-up (2026-10-24):

Follow AGENTS.md. In PostgreSQL 12, review the question page "SQL Heap and Index Visible-Count Checker in PostgreSQL 12 (unverified)".

Scope choice: Fix and re-run.

> Scope: the review found a shared-buffer blind spot, false identity drift for mapped catalogs, damage-hiding settings checked only at the start, an ambiguous unresolved-TID total, an over-broad interpolation guard and several documentation contradictions. The fixes keep the checking contract: they re-check those settings with every count, pin `client_min_messages`, read a mapped catalog's file number from the relation mapper, guard only ordering text, count unavailable TID signals, and add harness cases, including a cached-damage leg and a `pg_catalog` run.

Follow-up (2026-10-24):

Follow AGENTS.md. In PostgreSQL 12, review the question page "SQL Heap and Index Visible-Count Checker in PostgreSQL 12 (unverified)".

Scope choice: Fix and re-run.

> Scope: fix the unchecked evidence-list reads, qualify killed-index-tuple hinting, and correct the statistics-socket startup explanation. Extend the existing harness and rerun its measurements.

## Answer

### Feasibility and execution contract

**The checker is now one self-contained Bash script. Bash owns eligibility, candidate selection, plan acceptance, retries, coverage and verdicts. SQL reads catalogs, deparses expressions, captures exact counts and decodes JSON.** The [runner](#runner-script) creates two temporary tables and installs no functions or views. Its SQL templates are embedded in that script, so there is no separate installation file or second implementation of the checking rules.

The target is **PostgreSQL 12.2**, commit `45b88269a353ad93744772791feb6d01bc7e1e42`; later 12.x minors require a compatibility review. The source declares version 12.2. The runner checks the server's version number, which does not authenticate an arbitrary binary's provenance; the reproduction harness builds the pin. [configure.in#version](../../../../raw/postgres-12/configure.in#L19-L34)

This is the implemented replacement for the earlier operator-assisted SQL installation. The original Question records that earlier request. The user's subsequent approval of the Bash refactor supersedes the earlier follow-up's requirement that the checker SQL remain the source of all checks. Literal SQL still cannot execute arbitrary returned query text or receive [EXPLAIN](../../../glossary.md#explain) output as a relational result; the current implementation delegates those steps to [psql](../../../glossary.md#psql) and Bash. SQL `EXECUTE` means execution of a named [prepared statement](../../../glossary.md#prepared-statement), and EXPLAIN sends its result to the client. [execute.sgml#prepared-statement](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50) [explain.sgml#outputs](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L268-L275)

The run uses one dedicated connection to a primary. It writes only its own temporary tables and local report files. Create the temporary tables before starting `REPEATABLE READ READ ONLY`: PostgreSQL permits writes to existing temporary tables in read-only mode, but rejects `CREATE TABLE`, including temporary creation, there. Read/write is the default; the explicit read-only option activates this guard. A [standby](../../../glossary.md#hot-standby) cannot create the scratch tables. [execMain.c#ExecCheckXactReadOnly](../../../../raw/postgres-12/src/backend/executor/execMain.c#L759-L795) [utility.c#create-refusal](../../../../raw/postgres-12/src/backend/tcop/utility.c#L176-L224) [namespace.c#temporary-recovery-refusal](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3915-L3927) [guc.c#read-only-default](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1533-L1542)

The checker guarantees neither a particular index choice nor complete candidate coverage. A usable positive result requires an accepted execution; matching counts do not prove complete index correctness. The [planner](../../../glossary.md#planner) decides which ordered index path is cheapest. [indxpath.c#build_index_paths](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050)

**Count equality cannot detect heap-side data loss.** Both scans read the same heap, and an index scan silently skips an index entry whose heap row is gone, so lost rows drop out of both counts equally. A table whose heap blocks were all zeroed would pass with 0 = 0; the measured zeroed-block fixture loses 58 of 300 rows and still passes. The runner reports each accepted index scan's TIDs that reached no visible row, `unresolved_index_tids`, as a best-effort signal that never changes a status. The [heap-side loss](#heap-side-loss-and-the-unresolved-tid-signal) walkthrough gives the mechanism. [heapam.c#unresolved-tid](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1538-L1590) [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635)

Three more rules keep positive evidence of damage from reading as success. A relation with a matching index gets the added status `PASS_WITH_ANOMALY` when another index's accepted count differs or an attempt failed with a storage-class [SQLSTATE](../../../glossary.md#sqlstate); without a match, such an error makes it `ERROR`. The run stops when a server setting that zeroes or accepts damaged pages, or that turns off the statistics counters, is in effect at its start or in the settings recorded with any count. And a TRUNCATE, rewrite, refresh, re-creation or index rename committed after the snapshot, before the checker first locks the relation, makes the relation `ERROR` instead of a stale PASS.

**The scans see pages as the [buffer manager](../../../glossary.md#buffer-manager) supplies them, not the bytes on disk.** A block already in shared buffers is used without verification; verification runs only when the block is read from the file. Damage on disk under a clean cached copy therefore reads as intact until that copy is evicted or the server restarts. The measured cached-damage leg passes with all 300 rows before a restart and loses the zeroed block's 58 rows after it. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L763-L793) [bufmgr.c#verify-after-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925)

### Values and decision flow

The checker delegates [MVCC](../../../glossary.md#mvcc) visibility to PostgreSQL's [executor](../../../glossary.md#executor). A [sequential scan](../../../glossary.md#sequential-scan) passes the execution [snapshot](../../../glossary.md#snapshot) to the table access method. An ordinary [index scan](../../../glossary.md#index-scan) passes the same snapshot into [heap](../../../glossary.md#heap) fetching after obtaining index tuple identifiers. [nodeSeqscan.c#SeqNext](../../../../raw/postgres-12/src/backend/executor/nodeSeqscan.c#L50-L82) [nodeIndexscan.c#IndexNext](../../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L105-L149) [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L573)

| Value | Meaning and source | Live or stored | Later use |
|---|---|---|---|
| `T` | Database and physical relation [OIDs](../../../glossary.md#oid), names and catalog classification | Captured in `run-facts` and `relations/<oid>/facts` | Identifies one target during the coordinated run |
| `S` | First snapshot of the checking transaction | Live until transaction end; its descriptor is saved in run and count records | Binds heap and index observations to the same visibility boundary ([snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370)) |
| `H(T,S)` | Exact `bigint` from an accepted heap count | First in `hic_count`, then local attempt records | Baseline |
| `J` | Actual index OID resolved against retained catalog facts | Stored separately from the intended candidate | Determines actual coverage |
| `I(T,J,S)` | Exact count from an accepted ordinary Index Scan | Temporary capture plus local records | Comparison with `H` |
| `delta` | `I - H` | Derived by Bash for accepted index attempts | Difference of two nonnegative signed-64-bit counts; its range remains within signed 64 bits |
| `unresolved_index_tids` | TIDs an accepted index scan returned minus the visible heap rows they reached, from this backend's per-transaction counters | Live counters read before and after the count; stored per attempt | Best-effort sign of index entries whose heap rows are gone; never changes a status ([pgstatfuncs.c#xact-tuple-counters](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1683-L1711)) |
| Anomaly | An accepted index count that differs from `H`, or an attempt error of SQLSTATE class XX, 58 or 01 | Derived by Bash from attempt records | Turns a match into `PASS_WITH_ANOMALY` |
| File number and names | Each table's and index's name and storage file number at `S`: `pg_class.relfilenode`, or for a mapped catalog, whose `relfilenode` is 0, the [relation mapper](../../../glossary.md#relation-mapper)'s number read during inventory ([pg_class.dat#mapped-catalogs](../../../../raw/postgres-12/src/include/catalog/pg_class.dat#L24-L55), [dbsize.c#mapper-lookup](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L885-L889)) | Stored by the inventory; compared once with the current catalog | A difference is identity drift, which makes the relation `ERROR` |
| Candidate state | Eligibility facts, generated ordering, intended attempts and actual accepted OIDs | Local files keyed by numeric OID and attempt ID | Drives retries, stopping and verdicts |
| `indcheckxmin`, catalog `xmin`, `TransactionXmin` | Catalog flag, stored index-tuple xmin and backend horizon | The inventory stores the first two; the planner evaluates its current relcache tuple against the backend horizon | Determines snapshot usability; see [eligibility](#index-eligibility-and-ordering-generation) ([plancat.c#horizon](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234)) |

`count(*)` returns `bigint` and counts input rows whose keys are NULL too. `txid_current_snapshot()` supplies the version-12 snapshot descriptor. Catalog `reltuples` and EXPLAIN estimates are not substitutes for either exact count. [func.sgml#count](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L14917-L14936) [func.sgml#snapshot-interface](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L19384-L19418) [pg_class.h#estimates](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63)

A [savepoint](../../../glossary.md#subtransaction) is the rollback boundary for one attempt. Rolling back to it can discard that attempt while preserving the surrounding transaction. [xact.c#RollbackToSavepoint](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4249-L4345)

The objective is `PASS(T,S)` when at least one eligible, actually checked index has `H(T,S) = I(T,J,S)`. As a derived multiset example, `{a,b,c}` and `{a,a,c}` have equal cardinality despite different members. Passing one index gives no evidence about another. The [stronger optional diagnostic](#stronger-sql-diagnostics-and-pageinspect) compares identity bags instead.

```text
validate configuration → one psql session → two temporary tables
  → begin retained read-only snapshot → refuse damage-hiding server settings
  → inventory catalog facts
  → Bash classifies each object
      outside scope ───────────────────────────────→ NOT_APPLICABLE
      permission/identity failure ─────────────────→ ERROR
      no eligible index ──────────────────────────→ NO_ELIGIBLE_INDEX
      supported target
        → up to two heap attempts; require an accepted baseline
        → names and relfilenodes still match the snapshot?
            no ───────────────────────────────────→ ERROR (identity drift)
        → for each eligible, not-yet-covered index with supported ordering
            up to two attempts: normal sort profile, then discouraged sort
            TID counters → savepoint → exact count INSERT + executed plan → native JSON decoding
              SQL/client/capture failure or warning → rollback → local error record
              refused setting recorded with the count → stop the run (exit 2)
              rejected plan             → release → local rejection record
              accepted plan             → release → credit actual index OID, unresolved TIDs
            matching count → stop this relation unless exhaustive
        → Bash evaluates retained evidence, anomalies and uncovered errors
  → verify transaction identity → COMMIT → Bash writes reports
  → summary marks completion only when reporting and session close succeed
```

A rejected plan earns no coverage. A failed attempt can be recovered only under the same surviving checking transaction. The detailed status precedence is under [results](#result-records-and-aggregation). These branches are checker policy, rather than additional PostgreSQL guarantees.

### Scope and catalog inventory

Resolve the requested schema to a `pg_namespace` OID, then inventory its [`pg_class`](../../../glossary.md#pg_class) rows without first filtering by privileges or index existence. Join the table's `relam` to `pg_am`; separately join [`pg_index`](../../../glossary.md#pg_index) rows on `pg_index.indrelid = pg_class.oid`, the index's own `pg_class` row, and its index `pg_am` row. Require table `amtype = 't'` and index `amtype = 'i'` in their respective classifications. Keep a relation row even when its separate index query returns no rows. These catalogs distinguish namespace, relation kind, table versus index access method, and index-to-table ownership. [pg_class.h#identity-and-AM](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L30-L54) [pg_class.h#kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) [pg_index.h#catalog](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L57) [pg_am.h#method-types](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L53-L56) [pg_am.dat#built-in-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L35)

The inventory specification is:

| Catalog condition | Classification and treatment |
|---|---|
| `relkind = 'r'`, built-in heap table AM | In scope: ordinary tables and physical table partitions, including ordinary [inheritance](../../../glossary.md#inheritance) parents that store their own rows |
| `relkind = 'm'`, heap AM, `relispopulated` | In scope: count the stored rows of the [materialized view](../../../glossary.md#materialized-view), not the defining query |
| `relkind = 'p'` | `NOT_APPLICABLE: partitioned_parent_summary` as a scan target; `parents.csv` summarizes its children |
| `relkind = 'm'`, not populated | `NOT_APPLICABLE: unpopulated_materialized_view`; never invent a zero count |
| `relkind = 'r'` or `'m'`, another table AM | `NOT_APPLICABLE: unsupported_table_access_method`, with the unsupported table AM counted in the schema report; report a physical-object coverage gap |
| View `'v'`, foreign table `'f'`, sequence `'S'` | `NOT_APPLICABLE` with the reason `view`, `foreign_table` or `sequence`; no local heap-count claim |
| Index `'i'` or [partitioned index](../../../glossary.md#partitioned-index) `'I'` | `NOT_APPLICABLE: index_metadata`; inventoried as index metadata, not as a second table to count |
| [TOAST](../../../glossary.md#toast) `'t'`, composite type `'c'`, any other kind | `NOT_APPLICABLE: outside_physical_heap_scope`; explicitly outside this initial table/materialized-view scope |

The classifications use `relkind`, `relam`, `relispopulated`, and `relispartition`, not file size or the existence of a parent. PostgreSQL excludes partitioned parents from its storage-bearing kinds. Ordinary materialized-view reads skip rule expansion, and execution of an unpopulated materialized view raises an error. [pg_class.h#state](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L104-L119) [pg_class.h#storage-kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L183-L192) [rewriteHandler.c#materialized-view-reads](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L1900-L1911) [execUtils.c#unpopulated-refusal](../../../../raw/postgres-12/src/backend/executor/execUtils.c#L694-L714) [rules.sgml#materialized-views](../../../../raw/postgres-12/doc/src/sgml/rules.sgml#L898-L929)

**Default schema boundary: namespace only.** Include physical relations whose own `relnamespace` is the requested schema. Include such a partition even when its parent is elsewhere. Exclude an outside-schema partition from scanning even when its parent is inside the schema, but keep it in that parent's summary: the descendant namespace differs from the requested schema, and the `parents.csv` field `outside_schema_descendants` counts it. A parent summary must say whether it covers all descendants or only the selected subset.

Use the explicit `HIC_INCLUDE_EXTERNAL_PARTITIONS=yes` option. When enabled, recursively traverse `pg_inherits` from partitioned parents in the requested schema and include their physical descendants across schemas, deduplicated by relation OID. Record the added schemas and apply their privileges too. Restrict traversal to table relationships: `pg_inherits` also carries partitioned-index links in this implementation. Do not automatically expand ordinary inheritance into this optional partition closure. [pg_inherits.h#parent-links](../../../../raw/postgres-12/src/include/catalog/pg_inherits.h#L30-L37) [index.c#index-inheritance-link](../../../../raw/postgres-12/src/backend/catalog/index.c#L1002-L1010) [ddl.sgml#declarative-storage](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3611-L3626)

Every physical-target scan uses `FROM ONLY schema.relation`. Without `ONLY`, a parent query also scans inherited descendants; that would mix multiple physical objects into its baseline. A partitioned-parent summary aggregates child statuses and coverage, never accepts equality of aggregate parent totals as a replacement for individual child passes. The optional sum of child counts is meaningful as a common-snapshot total only in schema-wide snapshot mode. [select.sgml#ONLY](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L355-L365)

Retain `relpersistence` in the inventory. Permanent and [unlogged](../../../glossary.md#unlogged-relation) heap relations are eligible for this executor comparison; their durability is not being assessed. Own-session temporary application tables can be included when they are the explicitly requested scope. Classify another session's temporary relation as `ERROR: unsupported_remote_temp` before attempting a count: the reader cannot see its owner's local buffers. Keep the object in discovery rather than silently dropping it. Always exclude checker-owned scratch OIDs from the target set. [pg_class.h#persistence](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L165-L167) [bufmgr.c#temporary-relation-access](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L658)

### Index eligibility and ordering generation

A supported structural [B-tree](../../../glossary.md#b-tree) candidate must belong to an in-scope target and satisfy all of:

```text
index pg_class.relkind = 'i'
index AM = the built-in btree AM
pg_index.indpred IS NULL
pg_index.indisvalid AND pg_index.indisready AND pg_index.indislive
```

Keep rejected indexes with the reason codes Bash emits: `partial_index`, `invalid_index`, `index_not_ready`, `index_not_live`, `nonphysical_index` (a partitioned index) and `unsupported_index_am`. A [partial index](../../../glossary.md#partial-index) with a logically trivial predicate still fails this requested `indpred IS NULL` contract. The three state flags have distinct meanings; they are not estimates of physical correctness. The [planner](../../../glossary.md#planner) rejects invalid and partitioned indexes, while the [relcache](../../../glossary.md#relcache)'s usable index list omits indexes marked not live. [catalogs.sgml#index-state](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3822-L3865) [pg_index.h#predicate](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L53-L57) [plancat.c#validity-and-partitioned-indexes](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L199-L234) [relcache.c#non-live-indexes](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4383-L4400)

**Snapshot usability is an additional gate.** [`indcheckxmin`](../../../glossary.md#indcheckxmin) does not mean “always reject this index.” When true, the planner requires the `xmin` of the index's relcache `pg_index` tuple to precede the backend's [TransactionXmin](../../../glossary.md#transactionxmin). Otherwise it skips the index and marks the plan [transient](../../../glossary.md#transient-plan). This protects snapshots that could see incompatible [HOT](../../../glossary.md#hot) chain versions. [plancat.c#indcheckxmin](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234) [catalogs.sgml#indcheckxmin](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3835-L3844)

Record `indcheckxmin`, the observed `pg_index.xmin`, and the snapshot descriptor. Do not reproduce that gate with `xmin::text::bigint < txid_snapshot_xmin(...)`: the planner uses wrap-aware [transaction-ID](../../../glossary.md#transaction-id) comparison and a backend horizon, and its relcache tuple is not necessarily the catalog row an earlier SQL statement observed. Bash treats a true flag as **usability unresolved until an actual accepted plan uses that index**, rather than guessing the horizon. A refused candidate remains unresolved/untested unless there is independent conclusive exclusion evidence. It therefore blocks `MISMATCH` and `NO_ELIGIBLE_INDEX`, but does not block a valid pass through another index. [transam.c#TransactionIdPrecedes](../../../../raw/postgres-12/src/backend/access/transam/transam.c#L296-L312) [procarray.c#horizons](../../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1489-L1499) [relcache.c#index-xmin-refresh](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L2210-L2224)

Non-partial status is insufficient across [access methods](../../../glossary.md#access-method). B-tree advertises ordinary tuple scans, ordering, optional search keys, and NULL support. Hash omits NULL entries and requires a search key; GIN and BRIN have no ordinary `amgettuple` callback. Other methods and their operator classes need a separate completeness argument. Defer hash, GiST, SP-GiST, GIN, BRIN and extension AMs in the first implementation; do not use [bitmap-scan](../../../glossary.md#bitmap-scan) deduplication or a row-restricting predicate to simulate their full scan. [nbtree.c#bthandler](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L109-L142) [hashutil.c#NULL-omission](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L312-L337) [hash.c#capabilities](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L60-L74) [ginutil.c#scan-callbacks](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L63-L73) [brin.c#scan-callbacks](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L111-L121)

For each structural B-tree candidate, generate an ordering specification as follows:

1. Walk key positions `1..indnkeyatts`, not all `indnatts`. Included columns do not define index order. The deparser resolves `indkey`; the Bash generator reads `indclass`, `indcollation`, and `indoption` at their corresponding zero-based vector positions. [pg_index.h#key-metadata](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L32-L57) [pathkeys.c#INCLUDE-exclusion](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L487-L492) [ruleutils.c#index-keys](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1304-L1350)
2. Obtain each column or [expression](../../../glossary.md#expression-index) with `pg_get_indexdef(index_oid, position, false)`. On this pin the positive-position form requests only the variable/expression and suppresses index decorations. A simple column is printed as an identifier, quoted when needed; an expression key receives parentheses unless it is a bare function call. Do not split `pg_get_expr(indexprs, ...)` at commas or paste a full `CREATE INDEX` key specification into `ORDER BY`. Expressions themselves can contain commas, and an opclass name is not an `ORDER BY` suffix. [ruleutils.c#column-deparse](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1085-L1131) [ruleutils.c#expression-and-decoration](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1331-L1399)

   Wrap each deparsed key in another pair of parentheses, except a key that deparses to a bare numeric literal, such as an index on `(2)`. Parentheses do not protect such a key. With the default `operator_precedence_warning = off`, the grammar discards redundant parentheses, and a SELECT's `ORDER BY` follows SQL92 rules: a bare integer constant names an output column by position, and any other bare constant is an error. `pg_get_indexdef()` prints a non-negative `integer` constant, and a `numeric` constant with a decimal point, without a type label. The Bash generator therefore emits such a key as `CAST(<key> AS <its type>)`. The parser returns a same-type cast of a constant as the constant itself, so the sort key still equals the index expression. [ruleutils.c#get_const_expr-bare-numbers](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9554-L9585) [gram.y#parenthesized-expression](../../../../raw/postgres-12/src/backend/parser/gram.y#L13464-L13497) [guc.c#operator_precedence_warning](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1806-L1814) [analyze.c#SQL92-order-by](../../../../raw/postgres-12/src/backend/parser/analyze.c#L1252-L1256) [parse_clause.c#integer-constant](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1931-L1966) [parse_coerce.c#same-type-no-op](../../../../raw/postgres-12/src/backend/parser/parse_coerce.c#L156-L169)
3. Append the exact indexed [collation](../../../glossary.md#collation) when `indcollation` is nonzero. Qualify its namespace and quote identifiers. Generate and execute under the same controlled `search_path` so deparsed expression names resolve consistently. [ruleutils.c#index-collation](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1372-L1381) [pathkeys.c#sort-identity](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L512-L522)
4. Resolve the [operator class and family](../../../glossary.md#operator-class). For ordinary default ordering, `ASC`/`DESC` can suffice. For a nondefault family such as a pattern opclass, derive the same-type B-tree less-than or greater-than operator from `pg_opclass` and `pg_amop`, then use `USING OPERATOR(qualified_operator)`. Use strategy 1 for forward ascending and 5 for forward descending, with `amoppurpose = 's'`; these are B-tree search-family members, not `amoppurpose = 'o'` distance-ordering operators. Validate type compatibility instead of adding arbitrary casts. If overloaded/polymorphic/custom operators cannot be reconstructed confidently, retain the candidate as untested. [pg_amop.h#operator-metadata](../../../../raw/postgres-12/src/include/catalog/pg_amop.h#L54-L80) [parse_clause.c#SORTBY_USING](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L3263-L3295) [lsyscache.c#ordering-operator](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L216-L242)
5. Decode `INDOPTION_DESC` and `INDOPTION_NULLS_FIRST`; specify NULL positioning explicitly for every key. A backward scan reverses both ordering and NULL positioning for every key. The Bash generator emits complete forward ordering; a complete reverse-order variant remains optional and is not implemented here. The validator accepts either actual scan direction. [pg_index.h#ordering-bits](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L70-L76) [pathkeys.c#scan-direction](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L497-L519) [indices.sgml#ordering](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L505-L542)

The Bash generator must treat quoted schema, relation, column, collation and function names correctly. Use `format('%I.%I', ...)` for identifiers and literal quoting for values. Operator tokens require their SQL operator syntax, not identifier quoting; use only catalog-derived, validated operator names. Expressions, collation implementations and custom operator functions must also be trusted to execute in the checking role. Refuse or leave untested an expression whose execution is outside the allowed function policy; an `IMMUTABLE` declaration alone is not an audit of arbitrary user code. [func.sgml#format-examples](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L3402-L3414) [parse_clause.c#ordering-resolution](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L3277-L3295)

The implementation checks dependencies and index-column types **before** calling the deparser. Deparsing a non-NULL constant invokes its type-output function, so “we will not execute the generated query” is not enough to avoid all user code during inspection. The Bash generator conservatively defers non-stock functions/operators and unreviewed types, including domains; it permits enums and reviewed stock opclasses/operators. Its OID cutoff follows `FirstNormalObjectId`; it is a coverage policy for the pinned stock catalogs, not an authentication boundary against modified catalogs, malicious code or EXPLAIN hooks. [ruleutils.c#get_const_expr](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9522-L9550) [transam.h#FirstNormalObjectId](../../../../raw/postgres-12/src/include/access/transam.h#L136-L141) [explain.c#index-name-hook](../../../../raw/postgres-12/src/backend/commands/explain.c#L2844-L2862)

Compatible ordering encourages a path; it does not name an index. With index-only scans disabled, the planner builds an ordinary index path only when restrictions, useful ordering, or another qualifying reason makes it useful. An index containing only a constant expression gets no such path: a constant sort key falls into an [equivalence class](../../../glossary.md#equivalence-class) that contains a constant, and the planner drops a [pathkey](../../../glossary.md#pathkey) for such a class as redundant, so no useful ordering remains. Identically ordered indexes may repeatedly select the same cheaper index. These are coverage limits, not proof of corruption. [indxpath.c#path-generation-gate](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050) [equivclass.c#constant-member](../../../../raw/postgres-12/src/backend/optimizer/path/equivclass.c#L560-L570) [pathnodes.h#EC_MUST_BE_REDUNDANT](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L945-L950) [pathkeys.c#pathkey_is_redundant](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L132-L140) [pathkeys.c#redundant-ordering](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L524-L546)

### Snapshot permissions and DDL coordination

Use `BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY` after scratch-table setup. The default [isolation level](../../../glossary.md#isolation-level) is read committed, which takes a new snapshot for later statements. Repeatable read retains the transaction's first snapshot; the checker must not perform application DML between the two scans. Recording a snapshot descriptor helps audit provenance but does not recreate a completed transaction's snapshot later. [guc.c#isolation-default](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4255-L4272) [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [mvcc.sgml#repeatable-read](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L454-L479)

| Transaction mode | Guarantee | Operational trade-off |
|---|---|---|
| One schema-wide transaction, implemented here | Every accepted relation result shares one snapshot | Holds the snapshot and acquired relation/index locks until the run ends; one aborted transaction can lose all temporary result changes from that run |
| One transaction per physical relation, optional architecture not implemented here | Each `H(T,S_T)` and its index attempts share one `S_T` | Shorter snapshots and easier retries; the report must say there is no single schema-wide snapshot, and must not describe summed counts as one point-in-time schema total |

The run's inventory manifest must also remain complete across per-relation transactions. Require the membership/identity DDL freeze for the whole run, or re-inventory and invalidate a schema report whose target set changed. Shorter per-relation snapshots do not remove that scope-completeness requirement.

A long snapshot can retain old row versions that [VACUUM](../../../glossary.md#vacuum) would otherwise remove because its cleanup cutoff depends on the [xmin horizon](../../../glossary.md#xmin-horizon) of active transactions. Ordinary scan locks are compatible with DML, but conflicting DDL can wait behind the checker. [vacuum.c#cleanup-horizon](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L858-L865) [vacuum.c#GetOldestXmin](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L902-L912) [lock.c#lock-conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)

Require schema `USAGE`, relation `SELECT`, and database `TEMP` privileges for scratch tables, plus permission to execute any functions used by the generated expressions. Table-wide SELECT is the proposed minimum privilege policy, even where a narrower column grant could execute one particular query. [namespace.c#schema-usage](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L2903-L2910) [namespace.c#temporary-privilege](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3889-L3905) [execMain.c#relation-permissions](../../../../raw/postgres-12/src/backend/executor/execMain.c#L637-L665) [execExpr.c#function-permission](../../../../raw/postgres-12/src/backend/executor/execExpr.c#L2159-L2163)

Do not remove inaccessible tables from the inventory with a privilege predicate. Report them as `ERROR`. Use a role that sees all rows, preferably a suitably authorized role with `BYPASSRLS` and the required object privileges; an authorized superuser also qualifies. Table ownership bypasses [RLS](../../../glossary.md#row-level-security) only when `FORCE ROW LEVEL SECURITY` does not apply. `SET LOCAL row_security = off` raises an error instead of applying a policy; it grants no access. [rls.c#bypass-and-error](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132) [config.sgml#row-security](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7395-L7405)

Coordinate a DDL-stable window **before the first snapshot-acquiring statement and through the end of the applicable checking transaction**. This includes inventory/preflight queries and scratch-table INSERTs, not just the first target count. The window covers relation creation/drop/rename/schema moves, partition attachment/detachment, table rewrites, `TRUNCATE`, materialized-view refresh, and index creation/drop/rebuild. Ordinary reads acquire `ACCESS SHARE`; planner inspection also opens indexes and retains its locks. Those locks do not constitute a universal schema or concurrent-DDL freeze. In particular, the documented non-MVCC-safe rewrite/TRUNCATE case can affect a relation not yet accessed when an older snapshot was taken. [parse_relation.c#read-lock](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1223-L1235) [plancat.c#index-locks](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L170-L196) [mvcc.sgml#DDL-caveat](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1689-L1702)

**The runner detects identity drift itself.** After a relation's accepted heap attempt, the planner's `ACCESS SHARE` locks on the table and all of its indexes last until COMMIT, because a released savepoint passes its locks to the transaction. [plancat.c#index-locks](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L170-L196) [resowner.c#subtransaction-locks](../../../../raw/postgres-12/src/backend/utils/resowner/resowner.c#L594-L603)

The runner then compares, for the table and each index, the snapshot-time name and [file number](../../../glossary.md#relfilenumber) with the current catalog. `to_regclass()` and `pg_relation_filenode()` look names and rows up in the [system caches](../../../glossary.md#syscache) rather than with the snapshot. Once a backend is granted a lock it did not already hold, it processes pending invalidations. The heap attempt takes the checking transaction's first lock on the target, so the caches then reflect every commit up to that lock. For a mapped catalog the inventory's number already came from `pg_relation_filenode()`, so both sides read the relation mapper. A difference saves the relation's `identity_drift` and makes it `ERROR`; no index attempt follows. [regproc.c#to_regclass](../../../../raw/postgres-12/src/backend/utils/adt/regproc.c#L947-L968) [dbsize.c#pg_relation_filenode](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L864-L904) [lmgr.c#invalidations-after-new-lock](../../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L116-L138)

| Change committed after `S`, before the lock | Effect without the check | What the check sees |
|---|---|---|
| [`TRUNCATE`](../../../glossary.md#truncate) or a [table-rewriting](../../../glossary.md#table-rewrite) `ALTER TABLE` | The table looks empty to `S`, so both counts are 0 [mvcc.sgml#DDL-caveat](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1689-L1702) | A new `relfilenode` |
| Non-concurrent `REFRESH MATERIALIZED VIEW` | The refresh inserts frozen tuples, which `S` can see, so the counts describe post-snapshot data [matview.c#frozen-refresh](../../../../raw/postgres-12/src/backend/commands/matview.c#L460-L467) | A new `relfilenode` |
| Drop and re-create under the same name | The count statement names, and counts, the new table | The name resolves to another OID |
| `ALTER INDEX ... RENAME`, or the `REINDEX CONCURRENTLY` swap | Coverage is credited to the wrong index, because EXPLAIN prints the current name | The name resolves to another OID |

An index rename takes `ShareUpdateExclusiveLock`, as does `REINDEX CONCURRENTLY` while it swaps names, and neither conflicts with `ACCESS SHARE`. Such a swap can therefore commit even after the identity check, so the runner also resolves each executed `Index Name` through `to_regclass()` and rejects the attempt, recording drift, when it maps to another OID. A rename to a new name that the backend has absorbed leaves the executed name without a match in the inventory, so that attempt is rejected as `actual_index_not_uniquely_eligible` and earns no coverage instead of recording drift. A rolled-back attempt releases its locks, which is why the check waits for the accepted heap attempt. New relations, new indexes and changed index flags are not detected; the DDL window remains an operator premise for them. [tablecmds.c#index-rename-lock](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L3305-L3312) [indexcmds.c#session-lock](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3070-L3074) [indexcmds.c#concurrent-swap](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3202-L3262) [lock.c#lock-conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)

An optional pre-lock phase can use `LOCK TABLE ONLY ... IN ACCESS SHARE MODE` for known ordinary targets in deterministic OID order. This uses `SELECT` privilege. **Do not emit that command for a materialized view on PostgreSQL 12**: its `LOCK TABLE` implementation accepts tables, partitioned tables and views, but rejects materialized views. An ordinary read, such as a preflight `SELECT ... LIMIT 0`, can acquire the materialized-view read lock under the already established DDL coordination policy. That preflight is not a count and supplies no scan evidence. The implementation below does not add this pre-lock phase; ordinary count planning/execution acquires read locks as targets are reached. Ordinary refresh requests `ACCESS EXCLUSIVE`; concurrent refresh requests `EXCLUSIVE`, which permits `ACCESS SHARE`, so the latter must be covered by the explicit coordination policy rather than presumed blocked. [lockcmds.c#allowed-kinds](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L90-L103) [lockcmds.c#privilege](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L302-L310) [matview.c#refresh-lock](../../../../raw/postgres-12/src/backend/commands/matview.c#L159-L169) [lock.c#exclusive-conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L93-L103)

If identity, membership, eligibility or another DDL assumption changes, the affected attempt or run is not valid evidence; the runner detects only the identity drift described below. Do not mix an old heap count with a count taken after restarting the transaction. Re-discover and recount in a new transaction with a new run/snapshot identifier.

### Settings and query shapes

All [GUC](../../../glossary.md#guc) settings in this table are `user` / `PGC_USERSET`; none requires reload or restart. The checking profiles use `SET LOCAL`. Installation also sets the dedicated session's search path, 60-second statement timeout and 2-second lock timeout before the checking transaction begins, and turns off `idle_in_transaction_session_timeout`: a role or database value would otherwise end the session while Bash works between statements. Record effective values as well as the requested profile. The isolation level is selected at transaction start, before a query takes a snapshot. [guc.c#scan-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L881-L941) [guc.c#parallel-setting](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3007-L3015) [guc.c#row-security](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1571-L1578) [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396) [guc.c#search-path](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3789-L3797) [guc.c#quote-all-identifiers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1816-L1824) [set_transaction.sgml#isolation-timing](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L116-L123)

| Setting | Pin's default | Heap profile | Index profile | Purpose/mechanism |
|---|---|---|---|---|
| `enable_seqscan` | `on` | `on` | `off` | Index profile adds the sequential-scan cost penalty; it does not eliminate the path |
| `enable_indexscan` | `on` | `off` | `on` | Heap profile adds the ordinary index-scan penalty |
| `enable_indexonlyscan` | `on` | `off` | `off` | `check_index_only()` rejects index-only paths when off |
| `enable_bitmapscan` | `on` | `off` | `off` | Discourage bitmap paths; reject them in either accepted plan |
| `max_parallel_workers_per_gather` | `2` | `0` | `0` | Simplify execution to a serial scan: with the limit at 0 the planner builds no partial scan paths ([allpaths.c#create_plain_partial_paths](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L796-L805)); also reject any parallel plan actually produced |
| `enable_sort` | `on` | Leave on | Start on; optional retry off | Off adds `disable_cost` to an explicit sort ([costsize.c#cost_sort](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L1705-L1706)), which may help an ordered index compete; it is not a per-index hint |
| `row_security` | `on` | `off` | `off` | Error if RLS would filter the target |
| `search_path` | `"$user", public` | `pg_catalog, pg_temp` | Same | Qualify all target objects and deparse names in this controlled context: the deparser schema-qualifies a function name only when an unqualified lookup on this path would not find it ([ruleutils.c#generate_function_name](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10935-L10956)) |
| `quote_all_identifiers` | `off` | `off` | `off` | Fix deparser/EXPLAIN identifier spelling independently of the caller's settings |
| `lock_timeout` | `0`, disabled | Starting budget `'2s'` | Same | Bound each lock wait; expiry raises SQLSTATE `55P03` ([postgres.c#timeout-errors](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3082-L3095)); adapt to the operating window |
| `statement_timeout` | `0`, disabled | Starting budget `'15min'` | Same | Bound each statement; expiry raises SQLSTATE `57014` ([postgres.c#timeout-errors](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3082-L3095)), which is an error, never a zero count |
| `idle_in_transaction_session_timeout` | `0`, disabled ([guc.c#idle-timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2400-L2408)) | `0` for the session | Same | Overrides a role or database value; the server arms this timer whenever the session waits inside a transaction ([postgres.c#idle-timer](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4141-L4152)) |

The nondefault scan profiles change costing, not the acceptance contract. `enable_seqscan = off` and `enable_indexscan = off` add [`disable_cost`](../../../glossary.md#disable_cost); disabling index-only scans is checked when constructing an index-only path. The official sequential-scan and sort documentation expressly says these methods cannot be suppressed entirely. [costsize.c#seqscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L225-L232) [costsize.c#indexscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L525-L535) [costsize.c#bitmap-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L963-L970) [indxpath.c#index-only-gate](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L1898-L1907) [config.sgml#seqscan-and-sort](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4616-L4648)

The runner reads five server settings it never sets. It refuses the run, with exit status 2 and the reason in `summary.txt`, unless four of them have their defaults; `data_checksums` is only recorded. A configuration reload reaches the checking session between statements, even inside its transaction, so each count row records the four settings again, and a refused value there stops the run the same way. [postgres.c#reload-between-statements](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4216-L4220)

| Setting | Default and context | Why a nondefault value is refused |
|---|---|---|
| `zero_damaged_pages` | `off`; `superuser` [guc.c#damage-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1133-L1162) | A page that fails verification is zeroed into a valid buffer, with a WARNING. A retry reads that cached buffer without verification and succeeds, and a read past the end of a file returns zeros with no report at all. [bufmgr.c#zero-damaged-pages](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925) [bufmgr.c#valid-buffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L956) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L763-L793) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L631) |
| `ignore_checksum_failure`, for [data checksums](../../../glossary.md#data-checksums) | `off`; `superuser` [guc.c#damage-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1133-L1162) | A page that fails its checksum but has a sane header is used. The only report is a WARNING with SQLSTATE 01000, because `PageIsVerified()` passes `ERRCODE_DATA_CORRUPTED` without `errcode()` and a WARNING defaults to 01000. [bufpage.c#checksum-warning](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L143-L161) [elog.c#default-sqlstate](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L381-L388) |
| `track_counts` | `on`; `superuser` [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1399) | Off leaves relations without statistics counters, so the runner refuses the run. Failure to initialize the statistics socket also forces this setting off with `PGC_S_OVERRIDE`; that startup failure therefore reaches the same refusal, rather than a completed report with an unavailable signal. Independently, an accepted index count whose fetched-counter difference does not equal its exact row count leaves its signal empty; `summary.txt` counts relations with no usable accepted signal as `unresolved_tids_unavailable`. [pgstat.c#initstats-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1751-L1772) [pgstat.c#socket-startup-failure](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L614-L631) |
| `old_snapshot_threshold` | `-1`, disabled; `postmaster`, restart [guc.c#old_snapshot_threshold](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3041-L3049) | Enabled, [pruning](../../../glossary.md#pruning) can remove rows `S` still sees. Sequential-scan page reads test for that and raise "snapshot too old", but the index-path heap fetch never tests, so an index count can come out silently low. [heapam.c#seqscan-old-snapshot](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L402-L405) [bufmgr.h#TestForOldSnapshot](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L264-L275) [heapam_handler.c#index-fetch](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L115-L167) |

A superuser can change the first three for one session, per role or database, or in the configuration file with a reload; the last needs a restart. Separately, a count statement that prints a class-01 report fails its attempt, so such a warning never accompanies an accepted count. The server sends a warning to psql only when `client_min_messages` is at `warning` or below, so a role or database value of `error` would hide it; the runner therefore sets the default, `notice`, for its own session. [elog.c#client-output](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L301-L305) [guc.c#client_min_messages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4231-L4240)

The heap counting shape is:

```sql
SELECT /* wiki_heap_count_shape */ count(*)
FROM ONLY target_schema.target_relation;
```

The initial index counting shape is:

```sql
SELECT /* wiki_index_count_shape */ count(*)
FROM (
    SELECT
    FROM ONLY target_schema.target_relation
    ORDER BY indexed_key ASC NULLS LAST
) AS ordered_rows;
```

These illustrative shapes are embedded as explained count INSERTs in the runner. Replace the names and the entire ordering list using the inventory/deparser process above. A key with a different direction, NULL positioning, collation or operator family needs its corresponding specification. The inner SELECT intentionally has no restricting `WHERE`, `DISTINCT`, grouping, sampling, join, limit, offset or row-locking clause. The outer aggregate counts rows, not non-NULL keys. The inner SELECT has no output column on purpose: `ORDER BY` first matches a bare column name against output names, and an unnamed item such as `SELECT 1` is named `?column?`. With an output column, a key column named `"?column?"` would sort by that constant instead; PostgreSQL 12 accepts an empty select list. [parse_clause.c#SQL92-name-rule](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1865-L1930) [parse_target.c#default-name](../../../../raw/postgres-12/src/backend/parser/parse_target.c#L1661) [gram.y#empty-target-list](../../../../raw/postgres-12/src/backend/parser/gram.y#L14633-L14634)  A sorting subquery is not subject to the simple-subquery pull-up rule on this pin; useful ordering can therefore motivate an index path, but the chosen plan is still subject to validation. [prepjointree.c#sorting-subquery](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L1442-L1463) [indxpath.c#ordered-path](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050) [func.sgml#count](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L14917-L14936)

Disabling [index-only scans](../../../glossary.md#index-only-scan) is deliberate. Their [visibility-map](../../../glossary.md#visibility-map) shortcut can avoid the heap fetch entirely. An ordinary Index Scan instead calls `index_fetch_heap()`, whose table-AM implementation follows HOT chains and checks visibility using the scan snapshot. The checker wants the latter observation. [nodeIndexonlyscan.c#VM-shortcut](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L127-L170) [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635) [heapam_handler.c#heap-fetch](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L146-L167)

### Capturing an exact count and its executed plan

Keep the count and its plan tied to **one execution**. Ordinary EXPLAIN does not execute its query, and `EXPLAIN ANALYZE SELECT count(*)` discards the result value. Instrumentation reports floating-point row counts per loop, rounded for output; an Aggregate's one output row does not reveal its count value. [explain.c#execution-and-destination](../../../../raw/postgres-12/src/backend/commands/explain.c#L494-L541) [instrument.h#row-counters](../../../../raw/postgres-12/src/include/executor/instrument.h#L44-L65) [explain.c#rows-per-loop](../../../../raw/postgres-12/src/backend/commands/explain.c#L1505-L1534)

Each Bash attempt follows this sequence:

1. Allocate a monotonic attempt ID and save the intended relation, phase, candidate, variant and SQL text locally. For an index attempt, also save this backend's TID counters for every index of the relation.
2. Establish a [savepoint](../../../glossary.md#subtransaction), then apply the corresponding scan profile.
3. Execute one explained INSERT that saves `attempt_id, count(*)` in `pg_temp.hic_count`. Built-in defaults record transaction, snapshot, backend, role, settings and capture time with that row. A class-01 report printed by that statement fails the attempt.
4. Save that request's EXPLAIN JSON as `plans/attempt-<id>.json`, restore the control profile, and import the saved JSON into `pg_temp.hic_plan` using a collision-free [dollar-quoted](../../../glossary.md#dollar-quoting) literal.
5. Decode the JSON into typed node/property records. Bash validates every node, the attempt marker, the exact captured count and its execution context, and resolves the actual index. It then requires the executed index name to resolve to that same OID in the current catalog, and reads the counters again for the unresolved-TID count.
6. Release the savepoint before publishing accepted local evidence. On execution, capture or inspection error, or a warning while counting, roll back to the savepoint, release it and record the error locally. Failed recovery ends the run. A rejected but successfully inspected plan is retained as a rejection and earns no coverage.

The heap statement has this shape; the complete maintained template is in the runner:

```sql
EXPLAIN /* wiki_hic12_capture_example */
    (ANALYZE, VERBOSE, BUFFERS, SETTINGS, TIMING OFF, SUMMARY ON, FORMAT JSON)
INSERT /* wiki_hic12_save_example */ INTO pg_temp.hic_count(attempt_id, row_count)
SELECT /* wiki_hic12_input_example */ 1::bigint, count(*)
FROM ONLY target_schema.target_relation;
```

`EXPLAIN ANALYZE` executes the INSERT to completion, so the stored count and returned plan describe that execution. Bare EXPLAIN defaults ANALYZE, VERBOSE, BUFFERS and SETTINGS off; this template enables them to execute and expose identity/output evidence. TIMING and SUMMARY normally follow ANALYZE: this template disables per-node timing while retaining overall execution time, and chooses JSON instead of default text output. These are EXPLAIN options, not GUC changes. SETTINGS reports only nondefault planner-affecting parameters, so the separate captured settings remain necessary. [explain.c#run-to-completion](../../../../raw/postgres-12/src/backend/commands/explain.c#L501-L541) [explain.c#option-defaults](../../../../raw/postgres-12/src/backend/commands/explain.c#L154-L221) [explain.c#NewExplainState](../../../../raw/postgres-12/src/backend/commands/explain.c#L285-L295) [explain.sgml#settings](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L156-L162) [explain.sgml#timing-and-format](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L195-L235)

The verbose Aggregate output must begin with this attempt's `'<id>'::bigint` and `count(*)`. These markers detect accidental association with another attempt; they do not authenticate a document supplied by an adversary. The runner imports only the reply it just saved and keeps its transcript. It never executes a separate count and treats an earlier plan as proof of that execution.

On this pin, EXPLAIN's JSON `Index Name` already contains SQL identifier quoting. Inventory therefore records `quote_ident(relname)` with `quote_all_identifiers = off`, and Bash compares that encoded value with the executed name before crediting an OID. It never equates a quoted JSON identifier directly with an unquoted catalog name. [explain.c#index-name](../../../../raw/postgres-12/src/backend/commands/explain.c#L2844-L2862) [explain.c#index-JSON](../../../../raw/postgres-12/src/backend/commands/explain.c#L2991-L3025) [guc.c#quote-all-identifiers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1816-L1824)

Rollback cancels settings issued after the savepoint too, so every new attempt reapplies its profile. Control queries run with scan methods and sorting enabled to avoid applying diagnostic penalties to catalog/decoding queries. [set.sgml#savepoint-settings](../../../../raw/postgres-12/doc/src/sgml/ref/set.sgml#L61-L65)

### Mandatory plan validation

**Bash owns the validator.** The recursive SQL query supplies document shape, node paths, child-array shape/count, property names/types/values and Output elements. It does not return an accepted/rejected verdict. Native `jsonb_array_elements()` and `jsonb_each()` perform JSON expansion; malformed child values remain observable so the Bash validator can reject them. [jsonfuncs.c#jsonb_array_elements](../../../../raw/postgres-12/src/backend/utils/adt/jsonfuncs.c#L1976-L2014) [jsonfuncs.c#jsonb_each](../../../../raw/postgres-12/src/backend/utils/adt/jsonfuncs.c#L1655-L1695)

| Gate | Required evidence |
|---|---|
| Document | One JSON-array element, object root, nonnegative numeric execution time |
| Capture | One root ModifyTable Insert into the expected temporary schema's `hic_count`; zero returned rows |
| Aggregate | One plain, simple count aggregate; one output row; this attempt's marker and `count(*)` |
| Source | Exactly one Seq Scan for heap phase, or ordinary Index Scan for index phase, on the intended schema/relation |
| Completion | Every node executed once, with nonnegative numeric rows; no parallel-aware node or worker fields |
| Shape | One child per wrapper, none per source; only ModifyTable, Aggregate, Seq Scan, Index Scan, Sort, Result, Subquery Scan and Materialize |
| Restrictions | No Filter, Index Cond, Recheck Cond, join conditions, one-time filters, grouped keys, removed-row fields or subplans |
| Identity | Actual index name resolves uniquely to an eligible index belonging to this target, and through `to_regclass()` to the same OID in the current catalog; ordinary forward/backward direction |
| Reports | The count statement printed no error and no class-01 warning |
| Count provenance | Exactly one captured count row for this ID; expected transaction, snapshot, backend, role, read-only mode and scan profile |

Missing or wrongly typed required fields fail validation. Joins, Append, Gather, bitmap/index-only/TID/foreign/custom/sample scans, Limit, Unique, SetOp, ProjectSet, extra scans and unknown node kinds earn no acceptance. The narrow allowlist is checker policy. PostgreSQL supplies the node identities, loops, relation names and index details used by the gates. [explain.c#instrumentation](../../../../raw/postgres-12/src/backend/commands/explain.c#L1505-L1549) [explain.c#index-details](../../../../raw/postgres-12/src/backend/commands/explain.c#L2991-L3023) [explain.c#relation-details](../../../../raw/postgres-12/src/backend/commands/explain.c#L3156-L3162)

The validator accepts only statements assembled by this runner. It is not an API for certifying arbitrary supplied SQL or user code. A full unbounded B-tree executor scan starts at a tree endpoint and traverses the access method; acceptance does not independently prove that every physical page of a damaged tree is reachable. [nbtsearch.c#unbounded-start](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L997) [indexam.c#scan-completion](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635)

An attempt intended for `J2` can actually use `J1`. Bash credits `J1` only. If it matches, the existential objective passes; if repeated attempts keep returning a mismatching `J1`, `J2` stays untested and prevents complete-coverage MISMATCH.

### Result records and aggregation

Reports and audit state are local files. Bash keeps OIDs and attempt IDs as numeric keys; arbitrary names, expressions and messages are data, never shell code. A completed report requires `summary.txt` to say `completed=true` with exit status 0 or 1. Partial output from an interrupted run is not a completed verdict.

| Record | Location and contents |
|---|---|
| Run and snapshot | `run.csv`, `run-facts`, `settings.json`: schema/database identity, server version, backend, session/checking roles, transaction ID, snapshot descriptor, transaction times, effective settings and scope/exhaustive policy |
| Relations | `relations.csv` and `relations/<oid>/facts`: OIDs/names, kind, AM, persistence, population/RLS flags, parent OIDs, privileges, exclusions, accepted heap count, verdict/coverage totals, anomalies, storage-error attempts, the largest unresolved-TID count and any identity drift |
| Candidates | `candidates.csv`, `indexes/<oid>/`: intended/actual coverage, state flags, catalog xmin, exclusion/generation reasons, safely deparsed definition, dependency/key facts and generated ordering |
| Attempts | `attempts.csv`, `attempts/<id>/`: phase, intended/actual OIDs, variant, exact count, accepted/rejected/error state, delta, reasons, captured execution context, query, decoded plan facts, start/capture times, plan execution milliseconds, and for index attempts the TIDs returned and left unresolved |
| Plans | `plans/attempt-<id>.json`: original executed JSON; malformed/truncated capture is retained for diagnosis |
| Errors | Attempt columns/files with stage, SQLSTATE, message and, when parsed, detail and hint; `run-errors.csv` with stage, SQLSTATE and message; the complete original report stays in the session error log |
| Parent and schema summaries | `parents.csv` and `schema.csv`: descendant coverage, status counts including `PASS_WITH_ANOMALY`, actual-index totals, mismatches/errors, anomaly and unresolved-TID totals, relations whose unresolved-TID signal was unavailable, incomplete partition subtrees and success booleans |
| Client audit | `session-main/transcript.sql`, `stderr.log`, request/response files: SQL sent and client/server output |

A timestamp is not a snapshot descriptor. The run deliberately assigns `txid_current()` and records `txid_current_snapshot()`, rather than inferring visibility from a clock or treating a transaction ID as the list of in-progress transactions. [func.sgml#transaction-and-snapshot-functions](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L19384-L19418)

Apply relation statuses in this order:

1. **NOT_APPLICABLE:** explicit object/scope classification excludes the object from physical heap checking.
2. **ERROR:** a prerequisite failed, permission is missing, run evidence is invalid, accepted heap baselines conflict, or [identity drift](#snapshot-permissions-and-ddl-coordination) was found.
3. **NO_ELIGIBLE_INDEX:** complete inventory contains no supported eligible index. Unsupported/partial/invalid indexes retain their reasons.
4. **PASS_WITH_ANOMALY:** an accepted heap count and at least one accepted actual-index count match, and the relation also has an anomaly: an accepted index count that differs from the heap count, or an attempt error whose SQLSTATE class is XX, 58 or 01.
5. **PASS:** a match and no anomaly. Other attempt errors and rejections remain visible.
6. **ERROR:** no match and an execution/capture/inspection error remains uncovered by accepted evidence for that baseline or intended candidate.
7. **MISMATCH:** an accepted heap baseline exists, every eligible index was actually checked, and none matched.
8. **INCONCLUSIVE:** no match, no preceding decisive error, and a baseline path or candidate remained unaccepted/untested.

These are implemented in `hic_evaluate()`. An error becomes covered only when an accepted baseline exists, or an accepted scan actually covers its intended candidate. An error of class XX (internal error, data or index corrupted), 58 (system or file error) or 01 (a warning while counting) is never covered, because a retry can succeed on a page that the failed read zeroed. [errcodes.txt#storage-classes](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L431-L437) [errcodes.txt#internal-class](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L489-L494) Successful retries preserve their earlier error records. One observed mismatch with another untested candidate is INCONCLUSIVE. A matching actual index still establishes the requested existential match when another attempt failed for an ordinary reason such as a lock timeout.

`PASS_WITH_ANOMALY` extends the Question's six statuses at the asker's request in the 2026-10-24 follow-up. The existential match still holds, but positive evidence of damage keeps the relation from reading PASS. The unresolved-TID count never changes a status, because dead or snapshot-invisible versions also count.

```text
heap_scope_all_pass = completed inventory and valid evidence
                     AND at least one supported physical heap relation
                     AND every such relation has status PASS
schema_success     = heap_scope_all_pass
                     AND no unsupported local table-AM coverage gap
```

Any in-scope PASS_WITH_ANOMALY, ERROR, MISMATCH, INCONCLUSIVE or NO_ELIGIBLE_INDEX makes schema success false, so a completed run with an anomaly exits 1. Empty physical scope is not success. Expected exclusions such as views, sequences, foreign tables and unpopulated materialized views do not by themselves fail this defined heap scope. An unsupported local table AM does block unqualified schema success.

Parent summaries report whether all descendants are included and supported, and whether every physical descendant passed; `summary.txt` counts the partitioned parents whose subtree did not. Namespace-only scope can pass while an outside-schema or foreign partition prevents a complete subtree claim. Parent rows never enter the physical-relation denominator.

Exhaustive mode preserves the existential match and additionally reports actual candidate coverage, all-tested-match and anomaly totals. A mismatching index found after a match makes the relation `PASS_WITH_ANOMALY`, so `all_tested_indexes_match=false` no longer coexists with schema success. Checking every generated candidate query is not necessarily checking every index: repeated use of the same actual index earns one coverage unit. The relation's scalar totals plus its checked/matching OID files support the same assessment; the schema report exposes `exhaustive_coverage_complete` and `all_tested_indexes_match` explicitly.

### Concurrency performance and correctness limits

| Condition | Mechanism and implication |
|---|---|
| Concurrent INSERT/UPDATE/DELETE or abort | Both accepted scans use `S`, so later commits do not change the logical visible population. Old index entries can remain, but heap fetching applies `S` before returning a row. [mvcc.sgml#repeatable-read](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L454-L479) [indexam.c#visible-fetch](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L625-L632) |
| HOT update or pruning | One index root can lead to a later visible heap tuple or through a redirect. The heap AM resolves this; a raw root-TID census cannot substitute for it. [README.HOT#chain-and-redirect](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L48-L103) [heapam.c#HOT-search](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1559-L1622) |
| DDL, refresh, or membership change during inventory/checking | May invalidate identity, eligibility, coverage, or old-snapshot assumptions; require the coordinated window. Read locks alone cover only conflicting operations on already locked objects. The runner itself detects [identity drift](#snapshot-permissions-and-ddl-coordination) of each checked relation after its heap attempt, and nothing else. [lock.c#conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103) [mvcc.sgml#DDL-caveat](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1689-L1702) |
| Many candidates or poorly correlated index order | One heap baseline plus potentially several full index traversals and heap fetches can be expensive. Ordered index access can perform substantially more scattered heap I/O than a sequential scan; sort alternatives may be cheaper. Early stopping and coverage-aware retry limits bound avoidable work. [indices.sgml#index-versus-sort](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L489-L502) [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635) |
| Long snapshot | Can hold back cleanup; observe duration and abort within an agreed operating window. With nondefault `old_snapshot_threshold`, pruning can remove rows the snapshot still sees. Sequential-scan reads then raise "snapshot too old", but the index-path heap fetch does not test for it and can miss rows silently, so the runner [refuses](#settings-and-query-shapes) the setting. Its default is `-1`, disabled, and changing it requires a restart; the checker never changes it. [vacuum.c#cleanup-horizon](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L902-L912) [config.sgml#old-snapshot-threshold](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2378-L2396) |
| Shared corruption or missing historical data | Both scans agree on an incomplete heap: an index scan skips entries whose heap rows are gone, as the [heap-side loss](#heap-side-loss-and-the-unresolved-tid-signal) walkthrough below shows. Counts do not validate key values, ordering, uniqueness, payload/TOAST contents, unreachable pages, other indexes, or historical completeness. A full executor scan is the observation defined by this contract, not an independent physical census. [indexam.c#heap-based-results](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L625-L632) [nbtsearch.c#endpoint-traversal](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L997) |
| Heap visibility metadata is damaged | Even the sequential heap path can trust the page's `PD_ALL_VISIBLE` flag outside recovery. Disabling index-only scans does not make the baseline an independent reimplementation of visibility or a guarantee against damaged heap metadata. [heapam.c#page-visible-fast-path](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L428-L453) |
| Damage on disk under a cached copy | Both scans read through shared buffers. A block already in a buffer is used without verification, which runs only when the block is read from the file, so a damaged block whose clean copy is still cached reads as intact until the copy is evicted or the server restarts. The measured cached-damage leg shows it: 300 rows and `PASS` before a restart, the zeroed block's rows gone after it. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L763-L793) [bufmgr.c#verify-after-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925) |
| Timeouts, cancellation, numeric overflow or storage errors | Record the attempt error and preserve failure details. Without a match, an error that no accepted attempt covers makes the relation `ERROR`, while a covered ordinary error leaves `MISMATCH` or `INCONCLUSIVE` possible. A storage-class error (XX, 58 or 01) is never covered: with a match it makes the relation `PASS_WITH_ANOMALY`, without one `ERROR`. `count(*)` uses an `int8` transition and raises on overflow, rather than producing an arbitrary-precision count. [pg_aggregate.dat#count](../../../../raw/postgres-12/src/include/catalog/pg_aggregate.dat#L213-L219) [int8.c#int8inc-overflow](../../../../raw/postgres-12/src/backend/utils/adt/int8.c#L682-L705) |

The run's lifecycle is also part of the result:

| Event | What changes | Required action |
|---|---|---|
| First checking query | Acquires the retained repeatable-read snapshot | Record `S`; apply it to all later attempts in that transaction |
| An attempt fails and is rolled back to its savepoint | That attempt's writes disappear; the surrounding transaction can continue | Record the client error and confirm the run/snapshot still exists before retrying |
| Commit, full rollback, disconnect or transaction restart | Ends the old checking snapshot | Never compare a new index count with an old baseline; preserve exported report/transcript separately |
| Index rebuild, table rewrite, partition change or role change | Can change a premise of the saved inventory/plan | A changed name or file number found by the identity check, or an executed index name that now maps to another index, makes that relation `ERROR`; an index renamed to a new name after the lock only loses coverage; other changes need a new run |

The first and third rows follow the snapshot lifecycle; savepoint rollback is the recovery boundary for the second. The final row combines the runner's identity check with the operator's DDL window. [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [rollback_to.sgml#savepoint](../../../../raw/postgres-12/doc/src/sgml/ref/rollback_to.sgml#L29-L55)

“No application-data writes” does not mean no disk writes: temporary result storage, executor work files, and ordinary heap maintenance such as pruning are separate from application DML. The read-only transaction contract itself explicitly does not prohibit every disk write. [set_transaction.sgml#read-only-boundary](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L126-L139) [heapam_handler.c#read-side-pruning](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L130-L148)

### Heap-side loss and the unresolved-TID signal

The count comparison detects an index that misses visible rows or returns one twice. It cannot detect heap rows that are gone:

1. A zeroed heap page, an [all-zero page](../../../glossary.md#all-zero-page), has no [line pointers](../../../glossary.md#line-pointer), so the sequential scan returns nothing from it. [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L357-L360) [heapam.c:405](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L405)
2. The index still holds entries for the lost rows. For each one, `heap_hot_search_buffer()` returns no row when the offset is past the page's line pointers, the line pointer is not normal, or the TID names a heap-only tuple at chain start. [heapam.c#unresolved-tid](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1538-L1590)
3. `index_getnext_slot()` then takes the next TID without any report. [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635)
4. Both counts drop by the same rows, so the relation passes. The [identity-bag diagnostic](#stronger-sql-diagnostics-and-pageinspect) has the same blind spot, because both bags come from heap-filtered scans.

The same holds for line pointers that became unused, an older page image, or commit-status damage that hides committed rows. Both paths also apply the same per-tuple visibility test, so damage that makes a deleted or aborted version look live passes too whenever that version has its own index entry, as after a non-HOT update. [heapam.c:447](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L447) [heapam.c:1612](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1612) The measured zeroed-block fixture lost 58 of 300 rows and passed with 242 = 242.

To make such entries visible, the runner reads the per-transaction [statistics counters](../../../glossary.md#cumulative-statistics) `pg_stat_get_xact_tuples_returned()` and `pg_stat_get_xact_tuples_fetched()` for the relation's indexes before and after each index count. For an index, the first counts TIDs that the access method returned and the second counts visible heap rows those TIDs reached. The backend reports both only when idle outside a transaction, so inside the checking transaction they are exact for this backend. [pgstatfuncs.c#xact-tuple-counters](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1683-L1711) [indexam.c#returned-counter](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L541) [indexam.c#kill-prior-tuple](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L589) [pgstat.h#tuple-counters](../../../../raw/postgres-12/src/include/pgstat.h#L1369-L1383) [postgres.c#stats-flush](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4141-L4171) The fetched difference must equal the exact count, or the runner leaves the signal empty. The remainder, `unresolved_index_tids`, is reported per attempt, relation and schema; the zeroed-block fixture reports 58. A relation whose accepted index counts all left the signal empty is not added to the schema total as zero: `schema.csv` and `summary.txt` count it under `unresolved_tids_unavailable` instead.

The signal is evidence to diagnose, not proof, and it never changes a status:

| Entry | Counted as unresolved |
|---|---|
| Its heap row was deleted or updated away before `S` and is not yet removed: a [dead tuple](../../../glossary.md#dead-tuple) | Yes |
| It points to a version `S` cannot see, such as a later insert | Yes |
| Its heap row is gone: zeroed page, unused line pointer | Yes |
| An earlier index scan already marked it dead | No: the scan skips it |

An index scan asks the access method to mark an entry dead, a [killed index tuple](../../../glossary.md#killed-index-tuple), when the heap search finds no live chain member, which includes a TID that reaches no tuple at all. B-tree remembers the entry and attempts to mark it when leaving the page. Later scans outside recovery skip entries carrying that hint. [heapam.c#all-dead](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1538-L1540) [indexam.c#kill-prior-tuple](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L589) [nbtree.c#remember-killed](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L253-L269) [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1718-L1812) [genam.c#ignore-killed](../../../../raw/postgres-12/src/backend/access/index/genam.c#L108-L119) [nbtsearch.c#skip-killed](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L1476-L1486)

The marking is best effort. An ordinary MVCC scan of an index logged in [WAL](../../../glossary.md#wal) releases the page pin while processing its saved TIDs. `_bt_killitems()` reacquires the page and compares its [LSN](../../../glossary.md#lsn) with the saved value. [nbtsearch.c#release-page-pin](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L43-L67)

| Page state when hinting resumes | Result |
|---|---|
| The LSN still matches | Find matching entries and mark them dead; later scans outside recovery can skip those entries. [nbtutils.c#matching-hints](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1760-L1812) |
| Another operation changed the LSN while the page was unpinned | Release the page and return without marking; a later scan can encounter the same unresolved entries again. [nbtutils.c#hint-abandoned-after-change](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1760-L1768) |

The measured deleted-row fixture reports 10 on its first run and 0 on the next; it does not establish guaranteed suppression after every first scan. A physical census of index pages, which this runner does not perform, can still see entries carrying the hint.

### Stronger SQL diagnostics and pageinspect

The first stronger diagnostic should remain executor-based. Pre-create two temporary **bags**, without a uniqueness constraint, for `(relation_oid oid, tuple_tid tid)`. Under the same checking snapshot, explain and validate an INSERT selecting `tableoid, ctid` from the heap path and another INSERT selecting them from the ordered ordinary-index path. These statements differ from the count query, so their own executed plans must be checked; reuse of the earlier plans is insufficient.

Then compare the two saved bags:

```sql
SELECT /* wiki_heap_minus_index_bag */ relation_oid, tuple_tid
FROM pg_temp.heap_visible_tids
EXCEPT ALL
SELECT relation_oid, tuple_tid
FROM pg_temp.index_visible_tids;

SELECT /* wiki_index_minus_heap_bag */ relation_oid, tuple_tid
FROM pg_temp.index_visible_tids
EXCEPT ALL
SELECT relation_oid, tuple_tid
FROM pg_temp.heap_visible_tids;
```

Both differences must be empty to establish equality of the returned identity multisets. The default `EXCEPT` removes duplicates, whereas `EXCEPT ALL` preserves the excess multiplicity, which is necessary to detect the cancellation example. Keep separate run/attempt/snapshot identities, and filter to the intended bag if the scratch table holds several attempts. Relation identity is mandatory when bags contain more than one physical relation because a `ctid` is only a row-version location within its own table. [select.sgml#EXCEPT-ALL](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L1281-L1294) [ddl.sgml#ctid](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L1198-L1205)

These are executor-visible [TIDs](../../../glossary.md#tid). After following HOT, the heap fetch updates the returned tuple's `t_self`, which supplies SQL `ctid`. Thus both bags identify the visible row version, rather than equating a raw index's chain-root TID with the visible successor. Like the counts, both bags come from heap-filtered scans, so neither shows an index entry whose heap row is gone ([heap-side loss](#heap-side-loss-and-the-unresolved-tid-signal)). Materialization can be much larger than a count and can require substantial sorting/storage for `EXCEPT ALL`; it still does not validate stored index keys or search behavior for all predicates. [heapam.c#visible-HOT-member](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1582-L1619) [heapam_handler.c#returned-tid](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L148-L167) [heaptuple.c#ctid-system-attribute](../../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L636-L640)

[pageinspect](../../../glossary.md#pageinspect) is optional follow-up evidence, with separate privilege and consistency requirements:

| Physical inspection issue | Why raw item counts or TID equality are insufficient |
|---|---|
| Snapshot scope | `get_raw_page()` copies one buffer under a shared buffer-content lock. This is one time-consistent block image, not a relation-wide image corresponding to `S`; the next page can reflect a different physical instant. It reads through shared buffers. [rawpage.c#page-copy](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L159-L169) [pageinspect.sgml#page-copy](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L28-L35) |
| Heap line pointers | `heap_page_items()` shows line pointers and tuple data irrespective of MVCC visibility. Old, aborted or deleted versions and non-tuple pointer states do not equal visible rows. [pageinspect.sgml#heap-page-items](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L169-L175) [heapfuncs.c#physical-items](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L180-L224) |
| HOT and redirects | Index entries can point to a root whose storage is gone but whose line pointer redirects to a visible successor. Comparing raw index TIDs directly with visible `ctid` values incorrectly flags this legitimate state. [README.HOT#redirects](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L64-L103) |
| Dead index entries and B-tree structure | Physical items include entries awaiting cleanup and structural items; internal downlinks and high keys are not a census of visible heap rows. Index traversal and heap visibility are separate operations. [README.HOT#dead-root-cleanup](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L116-L122) [nbtree.h#pivot-layout](../../../../raw/postgres-12/src/include/access/nbtree.h#L249-L291) [indexam.c#heap-filtering](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L625-L632) |
| Posting lists and version differences | **This PostgreSQL 12 pin has no B-tree posting-list deduplication format.** Its non-pivot leaf tuples carry individual heap TIDs; the alternative representation is reserved for pivots. Do not import a posting-list expansion algorithm from another release. GIN's posting structures are a different AM, excluded here. [nbtree.h#tuple-representations](../../../../raw/postgres-12/src/include/access/nbtree.h#L249-L291) [nbtree.h#BTreeTupleGetHeapTID](../../../../raw/postgres-12/src/include/access/nbtree.h#L342-L364) |
| Concurrent physical changes | MVCC keeps a logical visibility boundary, not immutable page bytes. HOT pruning changes line pointers while preserving required versions. Reading one index page and then one heap page is not an atomic cross-page observation. [README.HOT#pruning](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L74-L103) [rawpage.c#lock-lifetime](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L159-L169) |

If mismatch localization warrants it, an authorized superuser can inspect selected implicated heap/index pages and record page numbers, acquisition times, raw images and the visible executor observations separately. `get_raw_page()` requires superuser on this pin. Repeated observations or a separately coordinated maintenance window may strengthen a diagnosis; they do not retroactively turn raw images into the original MVCC snapshot. [rawpage.c#privilege](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L95-L109)

Do not implement SQL MVCC visibility from header `xmin`/`xmax` fields. The executor already resolves snapshots, transaction state and HOT traversal; raw-page decoders do not perform that task. It is unnecessary for this objective, and no complete SQL reconstruction is established by this plan. [heapam.c#visibility-and-chain](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1559-L1637) [heapam_visibility.c#HeapTupleSatisfiesMVCC](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L944-L981)

### Executable SQL installation

**Extract only the Bash runner. There is no separate SQL installation step.** `hic_install()` contains the complete setup and creates just these temporary objects:

| Object | Purpose |
|---|---|
| `hic_count` | Exact `bigint` count keyed by attempt ID; built-in defaults capture transaction, snapshot, backend, role, settings and time |
| `hic_plan` | The current run's saved JSON documents keyed by attempt ID, for native decoding |

The setup creates no views, SQL functions, PL/pgSQL blocks or application objects. Configuration, inventory classification, error records, scheduling, acceptance and reports belong to Bash. Database-side work remains necessary for catalog access, SQL deparsing, counting and JSON decoding; it is contained in named templates in the same script. The [capture section](#capturing-an-exact-count-and-its-executed-plan) explains why existing temporary count storage remains necessary in a read-only transaction.

### How the runner talks to psql

Bash reuses one persistent psql process through two named pipes. It sends SQL requests serially, followed by `\echo HIC12_SYNC_<pid>_<session>_<sequence> :ERROR :SQLSTATE`. This sync line is a client command, so it can still report completion after a server error has aborted the transaction. PostgreSQL rejects ordinary statements in an aborted transaction, while psql executes `\echo` itself. [postgres.c#aborted-transaction-filter](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L1091-L1105) [command.c#exec_command_echo](../../../../raw/postgres-12/src/bin/psql/command.c#L1116-L1152)

The transport reads until that unique marker, preserving stdout and the request's stderr slice. It checks both the last statement's `ERROR` and every error report in that slice; a successful last statement outside a transaction must not hide an earlier failure. psql flushes output before reading input, after printing results and after backslash commands. Its last-result variables are set after each query. [mainloop.c#flush-before-read](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L147) [common.c#flush-after-result](../../../../raw/postgres-12/src/bin/psql/common.c#L1299) [command.c#flush-after-command](../../../../raw/postgres-12/src/bin/psql/command.c#L259-L260) [common.c#SetResultVariables](../../../../raw/postgres-12/src/bin/psql/common.c#L544-L584)

| Transport value | Representation | Consumer |
|---|---|---|
| Catalog identity, counts, flags | Numeric fields or fixed tokens separated by `\|` | Bash validates/compares them |
| Arbitrary names, key expressions and property text | `x` followed by UTF-8 hex; empty text is `x` | `hic_decode()` uses Bash byte decoding without `eval` |
| Node facts | `D` document, `N` node, `P` typed property, `O` output element, with numeric node paths | Bash's plan validator |
| SQL string values and imported JSON | Collision-free dollar-quoted literals | PostgreSQL/psql lexers preserve the content |
| Generated query text | Dedicated file for that attempt, sent unchanged | psql, after the interpolation guard |
| CSV reports | Every field quoted, embedded quotes doubled | Human/tools; never parsed back as internal state |

SQL's `convert_to(...,'UTF8')` produces encoded bytes and `encode(...,'hex')` serializes them. Empty fields stay explicit, so tabs, pipes, quotes, backslashes, newlines and non-ASCII names cannot become field delimiters. JSON decoding emits types and child shapes as well as values; it does not disguise malformed required fields as absent ones. [mbutils.c#pg_convert_to](../../../../raw/postgres-12/src/backend/utils/mb/mbutils.c#L414-L432) [encode.c#binary_encode](../../../../raw/postgres-12/src/backend/utils/adt/encode.c#L36-L69)

The [runner](#runner-script) carries arbitrary text through four steps:

1. SQL returns a hex-encoded catalog field. Access-method names and server-quoted collation names use this same transport.
2. Bash decodes the field into a complete value.
3. Local evidence files retain the complete value, including embedded and trailing newlines.
4. Bash reloads the complete value and writes the generated query file.

This matters even for ordinary expression indexes: `pg_get_indexdef(..., false)` retains indentation, and deparsing `CASE` inserts newlines. Quoted identifiers also preserve embedded newlines. The runner therefore never reads generated ordering text as a single line. [ruleutils.c#index-pretty-flags](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1117-L1131) [ruleutils.c#CASE-newlines](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L7741-L7755) [ruleutils.c#CASE-expression](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L8309-L8315) [ruleutils.c#quoted-identifier-bytes](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10703-L10716)

Bash dollar-quotes every arbitrary SQL value it sends, choosing a tag that neither occurs in the value nor matches its end: a value ending in the tag without its final `$` would otherwise close the literal early. Both lexers preserve dollar-quoted content without interpolation. The runner never passes catalog text to `eval`; its only `eval` calls open and close the numbered file descriptors of its psql sessions. [syntax.sgml#dollar-quoting](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L563-L621) [psqlscan.l#dollar-quotes](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L551-L588)

Deparsed expressions require an additional guard. Outside quotes psql substitutes `:name`, `:'name'` and `:"name"`; it defines variables such as `ERROR` itself. The deparser can print an array slice as `a[1:"ERROR"]`, which psql would rewrite. The runner conservatively refuses an ordering whose text has a colon not preceded by another colon and followed by a quote or brace. It records an execution error without sending that count. A similar sequence inside a literal or a quoted identifier of the ordering is refused too; that conservatism remains a coverage limitation. Typecasts `::` are lexed separately. The rest of every count statement is fixed text plus names from `format('%I.%I')`, which quotes any name containing a colon, and psql's variable rules do not apply inside a quoted identifier. A schema or table name containing such a sequence therefore still gets its heap count; the measured `odd:'name:` table passes. [psqlscan.l#quoted-identifier-state](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L132) [psqlscan.l#quoted-identifier-text](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L623) [ruleutils.c#quote_identifier](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10644-L10697) [psqlscan.l#substitution](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L708-L762) [psqlscan.l#typecast](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L301) [ruleutils.c#printSubscripts](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10614-L10637)

### Runner branches and failure handling

| Condition | Bash action | Consequence |
|---|---|---|
| Missing schema/DDL attestation, invalid yes/no option or unusable output directory | Refuse before connecting | Exit 64 |
| Installation, initialization or inventory fails | Record a run error where possible; roll back and end the session | Exit 2; no completed report |
| A [refused server setting](#settings-and-query-shapes) is in effect at the start, or a count row records one after a reload | Record the stop reason; roll back | Exit 2; no count taken under it is accepted |
| Count execution, plan import or inspection errors, or a warning while counting | Recover inside the attempt savepoint; record stage/SQLSTATE/message/detail/hint | No coverage; same-snapshot bounded retry remains possible; an XX, 58 or 01 error is an anomaly and never covered |
| Name or `relfilenode` changed after the snapshot | Record identity drift after the accepted heap attempt, or reject an attempt whose executed index name moved | Relation `ERROR`; no further index attempts |
| Executed plan fails acceptance gates | Retain rejection and reasons after releasing savepoint | No coverage; candidate may remain untested |
| Valid attempt | Release savepoint, then publish accepted count and actual-index identity | May establish baseline, coverage or PASS |
| Savepoint recovery or connection fails | Invalidate the run; do not reconnect | Exit 2 |
| No candidate can be accepted | Apply the defined status precedence | Completed non-success exits 1 |
| Final context differs from retained transaction/snapshot/backend/role | End as incomplete | Exit 2 |
| COMMIT, report writing or session close fails, or a local evidence file cannot be read | Keep partial audit output, omit a successful completion marker | Exit 2 |
| Complete inventory, checking, commit and reporting succeed | Write seven reports and `summary.txt` | Exit 0 for schema success, otherwise 1; `PASS_WITH_ANOMALY` is not success |

| Event | Database state | Local evidence state |
|---|---|---|
| Setup | Two temporary tables, outside the read-only transaction | Empty audit directories |
| First checking query | One retained repeatable-read snapshot | Run manifest and catalog facts saved |
| Attempt rollback | Attempt-local count/plan writes disappear | Error retained, provisional evidence receives no credit |
| Attempt release | Count/plan rows retained | Accepted/rejected record published |
| COMMIT | Checking snapshot ends | Verdicts derived from retained evidence, never from a new count |
| Session close | Temporary objects disappear; unfinished transaction aborts | Reports, raw plans and transcript remain |

The database snapshot and savepoint lifecycles justify those boundaries. Backend shutdown aborts an open transaction. [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [xact.c#RollbackToSavepoint](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4249-L4345) [postinit.c#ShutdownPostgres](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L1189-L1193)

Final causal chain: coordinated scope → retained snapshot → Bash-selected attempt → one explained exact-count INSERT → native decoding and Bash acceptance → actual-index coverage → Bash verdicts → context check, COMMIT and completed report. Lost execution context or incomplete output never earns schema success.

### Running the runner

Extract the single maintained block and run it with a [libpq](../../../glossary.md#libpq) connection environment. In this repository, keep the extracted script and output under `.wiki-runtime/`:

```text
fence=$(printf '\140\140\140')
page=wiki/v12/questions/indexing/sql-heap-index-visible-count-checker-plan.md
mkdir -p .wiki-runtime
sed -n '/^### Runner script$/,/^##/p' "$page" | sed -n "/^${fence}bash$/,/^${fence}$/p" | sed '1d;$d' > .wiki-runtime/hic12-check.sh
HIC_SCHEMA=app HIC_DDL_STABLE=yes HIC_OUT=.wiki-runtime/hic12-results PGHOST=/var/run/postgresql PGDATABASE=appdb bash .wiki-runtime/hic12-check.sh
```

| Item | Contract |
|---|---|
| Required environment | `HIC_SCHEMA`; `HIC_DDL_STABLE=yes`, the operator's DDL-window attestation rather than an automatic freeze |
| Options | `HIC_INCLUDE_EXTERNAL_PARTITIONS` and `HIC_EXHAUSTIVE`: `yes` or `no`, default `no`; optional `HIC_ROLE` |
| Programs/output | `HIC_PSQL`, default `psql`; `HIC_OUT`, default `hic12-run-<UTC time>-<pid>` in the current directory, must be new; no `HIC_SQL` input |
| Connection | Standard libpq environment, including `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGSERVICE` and `PGPASSFILE`; `PGAPPNAME` defaults to `hic12-check` ([libpq.sgml#environment](../../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L7296-L7319)) |
| Prerequisites | Bash without Bash-4-only features, pinned psql, `mkfifo`, `sed`, `grep`, `tail`, `wc`, `cp`, `tr`, `cat`, `date`, `paste` and ordinary directory utilities; no JSON parser, extension or PL/pgSQL installation |
| Reports | Read `summary.txt` first; exit 0/1 plus `completed=true` establishes completion. `summary.txt` also counts `pass_with_anomaly`, `anomalies`, `unresolved_index_tids`, `unresolved_tids_unavailable` and `incomplete_partition_subtrees`. Then `schema.csv`, `relations.csv`, `candidates.csv`, `attempts.csv`, `parents.csv`, `run-errors.csv`, `run.csv`, and the supporting facts/plans/transcript |
| Exit codes | 0 completed schema success; 1 completed without schema success, including any `PASS_WITH_ANOMALY`; 2 incomplete, including a refused server setting; 64 configuration refused |
| Server settings | Refuses `zero_damaged_pages` or `ignore_checksum_failure` on, `track_counts` off and an enabled `old_snapshot_threshold`, at the start and in the settings recorded with every count; sets `idle_in_transaction_session_timeout = 0` and `client_min_messages = notice` for its own session |
| Permissions | Output directory mode 0700 and umask 077. Reports can contain error values; use password/service files rather than command-line passwords |

psql runs with `-X -w -q -A -t`, `ON_ERROR_STOP=0` and `VERBOSITY=verbose`. The default ON_ERROR_STOP is already off; keeping it off permits Bash to recover after server errors. Verbosity changes from default to verbose so diagnostics include SQLSTATE. `-X` suppresses startup files, `-w` suppresses password prompts, `-q` suppresses command tags, and unaligned tuples-only output supports the transport. These are client options/variables, not GUCs. [psql-ref.sgml#no-psqlrc](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L553-L563) [psql-ref.sgml#no-password](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L496-L514) [psql-ref.sgml#quiet](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L374-L387) [psql-ref.sgml#no-align](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L65-L75) [psql-ref.sgml#tuples-only](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L433-L443) [psql-ref.sgml#ON_ERROR_STOP](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3844-L3861) [psql-ref.sgml#VERBOSITY](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3984-L3996)

The transport fixes `PGCLIENTENCODING=UTF8` on each psql process because decoded catalog fragments are UTF-8. The default client encoding follows the database; this connection-time session override prevents reinterpreting decoded fragments in another encoding. [guc.c#client-encoding](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3581-L3589) [postinit.c#encoding-default](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L394-L401)

The transport sets `LC_ALL=C` for psql's own labels; the server's messages still follow its `lc_messages`. It parses SQLSTATE from the verbose report and retains the raw error log. psql in noninteractive stdin mode flushes its reports and does not add an input-file line prefix. [startup.c#stdin-logging](../../../../raw/postgres-12/src/bin/psql/startup.c#L438-L446) [fe-protocol3.c#verbose-message](../../../../raw/postgres-12/src/interfaces/libpq/fe-protocol3.c#L1017-L1095)

### Test strategy

The matrix below is the **design-level test target**, including cases broader than the implemented fixture suite. The following measured-coverage section distinguishes actual runs from remaining gaps. Existing source tests provide adjacent coverage for expression/partial indexes and concurrent builds, pattern opclasses, RLS denial, materialized views and inheritance; they are not a test of this new checker. [create_index.sql#expression-and-partial](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L463-L498) [create_index.sql#pattern-opclass](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L700-L713) [rowsecurity.sql#FORCE-and-off](../../../../raw/postgres-12/src/test/regress/sql/rowsecurity.sql#L1511-L1542) [matview.sql#population-and-indexes](../../../../raw/postgres-12/src/test/regress/sql/matview.sql#L14-L40) [inherit.sql#ONLY](../../../../raw/postgres-12/src/test/regress/sql/inherit.sql#L40-L59)

| Fixture or action | Required assertion |
|---|---|
| Empty heap with a valid B-tree | Accept a zero count only when both source scans actually execute and satisfy the plan contract; absence of an index is not an empty-table pass |
| NULL-only, mixed NULL, duplicate keys; unique and nonunique indexes | Count every visible row with `count(*)`; no NULL-excluding predicate; duplicate keys are not duplicate result identities |
| Multiple keys with mixed ASC/DESC and NULL positioning; INCLUDE columns | Generate compatible key order without ordering by INCLUDE columns; validate both accepted scan directions |
| Expression indexes, including NULL-returning expressions and expressions containing commas | Per-position deparse round-trips without splitting expression text; either accepted full scan or an explicit untested reason |
| Nondefault collations, pattern opclasses, quoted/mixed-case names and schemas | Correct identifier quoting and family-specific ordering; record actual index; do not accept a sorted sequential fallback as an index result |
| Constant-expression keys, including integer and decimal literals, and two indexes with equivalent ordering | Literal keys reach `ORDER BY` as constant expressions, never as output positions; exercise absent full-index path and repeated selection; untested OIDs remain untested, so no false exhaustive MISMATCH |
| HOT updates, redirects, pruning; non-HOT indexed-column updates | Compare one snapshot's visible row versions; the stronger bags use visible `ctid`, not raw root TID |
| Deletes; aborted insert/update/delete; an open writer at snapshot creation | Both paths agree with the visibility boundary; physical old/aborted entries do not become visible counts |
| Concurrent writers before, between and after the two scans | Both accepted observations retain `S`; a control run using separate read-committed snapshots demonstrates why that mode is rejected |
| Ordinary inheritance parent/child, multiple inheritance, multilevel partitions | Every physical OID checked once with ONLY; no inherited double count or compensation between child mismatches |
| Cross-schema parent/leaf, foreign leaf, detached or newly attached partition | Namespace-only and explicit descendant-closure policies report their exact coverage; DDL races invalidate rather than silently alter the target set |
| Populated, empty-populated and unpopulated materialized views | Stored heap rows are counted when populated; unpopulated is NOT_APPLICABLE; no unsupported `LOCK TABLE` command is generated |
| RLS owner, forced RLS owner, BYPASSRLS role and ordinary restricted role | Off is a guard: filtered roles error; bypass requires the role and ordinary object privileges |
| Missing schema USAGE, table SELECT, database TEMP or function EXECUTE | An explicit ERROR; denied objects remain in the inventory |
| Only partial indexes; only unsupported AMs; invalid/not-ready/not-live indexes | NO_ELIGIBLE_INDEX only when eligibility is completely settled; record all exclusion reasons |
| `indcheckxmin` true with older/newer checking horizons, and a new transaction retry | Do not reject by flag alone or integer-xmin comparison; credit only actual accepted use; never carry `H` across snapshot changes |
| Invalid, mismatched, stale or unsupported plan evidence | Reject index-only, bitmap, parallel, filtered, limited, joined, wrong-relation, wrong-index, never-executed or stale-attempt evidence; do not claim authentication of forged operator input |
| Timeout, cancel, permission error, malformed JSON, missing capture row and disconnect | ERROR or invalidated run; no stale value reuse, zero substitution or accidental continuation with a new snapshot |
| All tables PASS; one NO_ELIGIBLE_INDEX; one INCONCLUSIVE; one ERROR; zero physical targets; unsupported table-AM inventory record; another session's temporary table | Schema aggregation follows the explicit booleans and cannot hide a required relation; another backend's temporary relation is rejected before any count |
| Count equality with differing identity bags, supplied as synthetic evaluator data | Equal cardinalities can still produce nonempty EXCEPT ALL results; duplicate-sensitive comparison works |
| One mismatching accepted index plus an untested candidate; all candidates mismatch; one match plus another ordinary failure | Respect INCONCLUSIVE/MISMATCH/PASS distinctions and retain per-attempt diagnostics |
| One match plus a mismatching accepted index, or plus an XX, 58 or 01 attempt error | `PASS_WITH_ANOMALY` and no schema success; a storage-class error is never covered by a retry |
| `zero_damaged_pages`, `ignore_checksum_failure`, `track_counts` or `old_snapshot_threshold` changed from its default, at the start or by a reload during the run | Refused before any count; after a reload, the run stops at the next count |
| A damaged block whose clean copy is still in shared buffers | `PASS` with every row while the copy is cached; after a restart the block's rows drop out of both counts |
| Mapped catalogs, with `pg_catalog` as the target | No identity drift; the mapped catalogs pass |
| A table name containing `:'` and ending in `:` | The heap count is sent and the relation is checked |
| A role-level `client_min_messages = error` | Warnings still reach the runner |
| Unavailable TID counters | Counted under `unresolved_tids_unavailable`, never added to the total as zero |
| TRUNCATE, REFRESH, drop and re-create, or an index-name swap after the snapshot | Relation `ERROR` with identity drift, never a stale PASS |
| Deleted rows not yet vacuumed; a zeroed heap block | The unresolved-TID count equals the entries without a visible row, on the first scan only |
| A key column named `?column?`; a value ending in the dollar-quote tag; a role-level idle timeout; missing or unreadable scalar and list evidence | Correct ordering, literal round trip, surviving session, and exit 2 respectively |

Tests of deliberately corrupted on-disk indexes are a separate boundary. Ordinary SQL DML maintains indexes, and pageinspect only exposes page contents; this plan does not establish a supported SQL-only method to create arbitrary missing/duplicate physical index entries. Use synthetic count/plan/bag records to test the verdict evaluator, clearly labelled as unit tests. A real corruption campaign would require a separately approved fault-injection capability or supplied damaged disposable database; synthetic evaluator tests must not be reported as physical corruption detection coverage. At the asker's request in the 2026-10-24 follow-up, the harness now has one such physical leg: it stops the owned server, zeroes one heap block of a disposable table and fills one heap block of another with 0xFF bytes, then restarts. Damaged index pages and other corruption shapes remain outside the suite.

Likewise, this pin supplies heap as its only built-in table AM. Test unsupported table-AM classification with synthetic inventory records; a real custom-AM integration test would need a separately authorized capability, not an unannounced extension dependency. [pg_am.dat#built-in-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L35)

The acceptance gate is that all positive fixtures produce a count only from validated executed paths, and every exercised negative/coverage fixture refuses an unjustified PASS or MISMATCH. Actual plans remain in the attempt exports even when they differ from the intended path.

### Measured coverage and remaining test gaps

The current Bash-owned implementation passed **263 assertions** in the published reproduction suite. The last-run record under [Measurement Script](#measurement-script) identifies the platform and runs. These measurements establish the exercised outcomes, not physical corruption-detection rates.

| Exercised case | Result |
|---|---|
| Positive schema, ordinary restricted reader | 13 physical targets and 13 actual indexes pass; empty heap and empty populated materialized view have zero baselines; NULL/duplicate/aborted fixture has 5 rows; inheritance parent/child have 1 and 2 rows independently |
| Optional external partition closure | 14 physical targets pass; outside leaf has 1 row; foreign child remains an explicit unsupported subtree gap |
| Negative schema | 8 physical targets: 4 NO_ELIGIBLE_INDEX and 4 INCONCLUSIVE; literal-key queries use casts and earn no false index coverage; custom expression/domain generation is deferred |
| Competing equivalent indexes, exhaustive mode | PASS with 2 eligible indexes, 1 actually checked and 1 untested |
| Permission/RLS cases | Forced-RLS owner, missing table SELECT and missing schema USAGE each yield ERROR; superuser bypass case passes |
| Snapshot held across committed HOT/non-HOT updates, deletes, insertion and aborted insertion | Checker heap baseline remains 100, while the writer's current count is 95 |
| `indcheckxmin` horizon | Old holder yields INCONCLUSIVE; a new transaction after the holder ends passes without requiring the flag to clear |
| Plan-shape units | 41 assertions: valid heap/index/backward plans and malformed, restricted, parallel, wrongly typed, missing, grouped, extra-branch, wrong-marker and unknown-node shapes |
| Synthetic evaluator | 14 cases cover partial versus complete mismatch coverage, recovered/optional errors, conflicting baselines, invalid evidence and exact comparison at signed-bigint maximum. Two mismatches then a match, an XX001 error plus a match, and a warning-class heap error followed by an accepted retry each give `PASS_WITH_ANOMALY`; a rejected attempt whose executed index name moved, and saved drift, each give ERROR |
| Truncated plan and rollback | `22P02` with DETAIL; count row is absent after savepoint rollback; same-snapshot retry passes and retains 1 failed attempt |
| Held ACCESS EXCLUSIVE lock | Two heap attempts time out with `55P03`; relation ERROR, never an empty-table pass |
| Disconnect and output failure | Terminated backend ends the next request as incomplete; an unusable report filename fails reporting |
| Client/program boundary | CLI exits 0, 1, 2 and 64; no attestation refuses before connecting; seven reports emitted for a completed positive run |
| Text transport | UTF-8, quotes, pipes, tabs, newlines and backslashes round-trip; CSV quote escaping matches expected bytes |
| Multiline orderings, ordinary restricted reader | Four targets pass through four actual indexes: a `CASE` expression, a newline-containing column name, a newline-containing collation name and a newline-containing expression literal; each baseline is 4 |
| Unusual index-AM names | Two custom AM identities using the stock hash handler have pipe/newline names; both remain explicit `unsupported_index_am` candidates and the table is NO_ELIGIBLE_INDEX |
| Local evidence values | Empty values, embedded/trailing newlines and a literal sentinel character survive both single-value and multi-value save/load paths |
| Harness startup ownership | Child `prepare`, `build`, `init`, `cleanup` and unknown-stage invocations refuse or fail without stopping the parent invocation's running server; it still accepts queries |
| Installation contract | Completed sourced cases confirm exactly two temporary tables and no temporary functions/views |
| Refused server settings | Guard units for all four settings; CLI runs with `zero_damaged_pages=on`, `ignore_checksum_failure=on` or `track_counts=off` exit 2 with the reason in `summary.txt`; a reload that turns `zero_damaged_pages` on after the snapshot reaches the open checking session, and the next count stops the run with exit status 2 and that reason |
| Identity drift after the snapshot | A second session's TRUNCATE, REFRESH, drop and re-create and index-name swap each make that relation ERROR with recorded drift (storage drift for TRUNCATE, name drift for the swap); the unchanged table passes |
| Edge schema | A `"?column?"` key column passes with 3 rows; ten deleted rows' entries give `unresolved_index_tids=10`, and 0 on the next run because the first scan marked them dead |
| Idle timeout | A 1-second session `idle_in_transaction_session_timeout` and a 2-second pause inside the checking transaction do not end the run |
| Missing or unreadable evidence | A directory replacing a scalar `scope` file makes reporting exit 2. Missing-file and directory substitutions for `relation-ids`, `partitions.raw`, `index-ids` and `attempt-ids` also return 2; restoring the lists restores successful reporting. `hic_finish()` propagates a missing `partitions.raw` failure instead of completing the run |
| Dollar quoting and warnings | A value ending in `$hic12_q0` gets the next tag and round-trips; a class-01 report is found by the warning pattern, never as an error, and counts as a storage-class code |
| Physical leg, zeroed block | 58 of 300 rows lost; PASS with heap 242, and `unresolved_index_tids=58` |
| Physical leg, invalid header | Both heap attempts fail with `XX001`; relation ERROR with 2 storage errors; a CLI run with `zero_damaged_pages=on` is refused |
| Physical leg, cached copy | Block 1 of a third table zeroed under the running server after `VACUUM (FREEZE)` and `CHECKPOINT`: `PASS` with 300 rows and `unresolved_index_tids=0`; after the restart the same table gives `PASS` with 242 rows and 58 unresolved TIDs |
| Mapped catalogs | `pg_catalog`, checked as superuser: 63 physical catalogs pass, including `pg_class`, `pg_attribute`, `pg_proc`, `pg_type` and `pg_database`, none with drift; 117 eligible indexes, 63 checked, 54 untested. Unresolved index TIDs: 1454 in the fresh-build run and 1836 in the retained-build run; this signal never changes a status |
| Guarded names | The table `odd:'name:` passes with 3 rows |
| Message level | A plain psql session with `client_min_messages=error` prints no warning for `ROLLBACK` outside a transaction; the runner's session reads `notice` and receives the `25P01` warning |
| Unavailable signal, synthetic report input | Blanking the accepted index attempt's signal gives `unresolved_index_tids=0` and `unresolved_tids_unavailable=1` |

The concurrent fixture's derived arithmetic is `100 - 6 + 1 = 95`, while both accepted checker scans retain the earlier 100-row population. Repeatable read retains its snapshot, and ordinary index fetching checks heap visibility against it. [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [indexam.c#visible-fetch](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L625-L632)

The HOT/horizon fixtures deliberately use heap fillfactor 50 instead of default 100, reserving room for same-page updates. The writer verifies HOT updates and also changes an indexed column to exercise non-HOT updates. A retained old transaction followed by a nonconcurrent index build creates the broken-chain horizon gate; the planner tests catalog xmin against `TransactionXmin`, not merely the flag. These special fixture causes are documented with their source evidence under [Measurement Script](#measurement-script). [heapam.c#HOT-decision](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3591-L3601) [plancat.c#horizon](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234)

Not exercised: physical corruption beyond the three damaged heap blocks, including damaged index pages; a reload that changes `ignore_checksum_failure` or `track_counts` during a run; a checksum failure, because the test cluster has checksums off (the warning path is a parser unit); real custom table AM; arbitrary custom opclass/type/expression combinations; non-C/ICU locale behavior; multilevel/multiple-inheritance and attach/detach races; another backend's temporary table; denied TEMP/function EXECUTE; a separate non-superuser BYPASSRLS role; an open writer at initial snapshot acquisition; pruning/redirect races; a default-versus-read-committed control leg; forced statement timeout/cancel; loss while a statement is running; standby support; large-schema/production-I/O scaling; actual backward executor traversal; not-ready/not-live catalog states; the optional identity-bag pipeline. The maximum-bigint case is synthetic evaluator input, not an enormous measured table. Units do not establish corruption-detection rates.

### Optional relaxations

The implemented Bash runner is the user's authorized orchestration and checking-policy choice. Per-relation transactions, exported-snapshot coordination, a PL/pgSQL wrapper, broader access-method support and the visible-identity capture pipeline remain separate designs. They must retain the same-execution count/plan and actual-coverage requirements. SQL `EXECUTE` in PL/pgSQL supports dynamic commands, while ordinary SQL EXECUTE names a prepared statement; no PL/pgSQL wrapper is installed here. [plpgsql.sgml#dynamic-commands](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1157-L1185) [execute.sgml#prepared-statement](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50)

### Runner script

The script reads top to bottom in sections: small helpers, the psql session layer, setup, inventory, plan validation, attempts, verdicts, reports and the driver. Each catalog fact line is read into named variables (`r_*` for relations, `i_*` for indexes), and each attempt, relation or index keeps its evidence as one-value files (`hic_put` and `hic_get` handle one value, `hic_save` and `hic_load` several). The [measurement harness](#measurement-script) sources this file and calls its functions directly, so those function names and evidence files are part of its test interface. `hic_server_guard` refuses damage-hiding settings; `hic_read_count` repeats that guard on every count row's recorded settings. `hic_identity` checks names and relfilenodes, using the relation mapper for mapped catalogs. `hic_tid_counters` and `hic_unresolved_tids` derive the unresolved-TID count, and reports count relations whose signal is unavailable. The third 2026-10-24 review fixes evidence-list handling: verdicts and reports read each list completely with `hic_get` before iterating over its captured contents, `hic_contains` distinguishes a read failure from an absent set member, and line counts check command status. A failed read stops the run instead of becoming an empty list or a changed verdict.

```bash
# hic12-check.sh: self-contained Bash checker for PostgreSQL 12.2
# heap and index visible-count checker. Bash makes every decision. psql
# receives SQL statements, plus one \echo sync line per request that prints
# psql's ERROR and SQLSTATE variables.
#
#   HIC_SCHEMA=app HIC_DDL_STABLE=yes bash hic12-check.sh
#
# The connection comes from the standard libpq environment (PGHOST, PGPORT,
# PGDATABASE, PGUSER, PGSERVICE, PGPASSFILE). Options: HIC_EXHAUSTIVE and
# HIC_INCLUDE_EXTERNAL_PARTITIONS (yes or no, default no), HIC_ROLE, HIC_OUT
# (a new output directory) and HIC_PSQL (default psql). Exit status: 0 run
# completed with schema_success; 1 run completed without it; 2 run did not
# complete; 64 usage error. Sourcing this file only defines the functions.
# The run stops when zero_damaged_pages or ignore_checksum_failure is on,
# track_counts is off, or old_snapshot_threshold is enabled, whether at its
# start or in the settings recorded with any count.

: "${HIC_PSQL:=psql}"
: "${HIC_S:=main}"                  # name of the checking session
HIC_OPEN=${HIC_OPEN:-}              # names of the open psql sessions

# A psql verbose report line looks like "ERROR:  42P01: message".
HIC_REPORT_LINE='^[^ ]+:  [0-9A-Z]{5}: '
# The same, for error classes only: 00, 01 and 02 are success, warning, no data.
HIC_ERROR_LINE='^[^ ]+:  ([1-9A-Z][0-9A-Z]|0[3-9A-Z])[0-9A-Z]{3}: '
# Warnings (class 01). A count attempt that prints one is treated as failed.
HIC_WARNING_LINE='^[^ ]+:  01[0-9A-Z]{3}: '
# SQLSTATE classes that point at damaged storage: XX internal or corrupted
# data, 58 file/system errors, and 01 warnings raised while counting.
HIC_STORAGE_CODE='^(XX|58|01)'
# Ordering text containing psql's :'var', :"var" or :{?var} forms is not sent.
# The rest of a count statement is fixed text plus names that format('%I.%I')
# quotes whenever they contain a colon, and psql never substitutes inside a
# quoted identifier.
HIC_GUARD="(^|[^:]):['\"{]"
HIC_FIRST_USER_OID=16384            # FirstNormalObjectId
HIC_CATALOG_NS=11                   # pg_catalog
HIC_HEAP_HEX=x68656170              # "heap" in the hex transport
HIC_BTREE_HEX=x6274726565           # "btree" in the hex transport
HIC_COUNT_HEX=x6869635f636f756e74   # "hic_count" in the hex transport

# ---- small helpers -----------------------------------------------------------

hic_say() { printf 'hic12-check: %s\n' "$*"; }
hic_warn() { printf 'hic12-check: %s\n' "$*" >&2; }

hic_uint() { case $1 in ''|*[!0-9]*) return 1 ;; esac; }
hic_decimal() { [[ $1 =~ ^[0-9]+([.][0-9]+)?$ ]]; }
hic_one() { [[ $1 =~ ^1([.]0+)?$ ]]; }
hic_zero() { [[ $1 =~ ^0([.]0+)?$ ]]; }
hic_user_oid() { [ "$1" -ge "$HIC_FIRST_USER_OID" ]; }
hic_heap_am() { [ "$1" = 2 ] && [ "$2" = "$HIC_HEAP_HEX" ] && [ "$3" = t ]; }  # relam, hex amname, amtype
hic_btree_am() { [ "$1" = 403 ] && [ "$2" = "$HIC_BTREE_HEX" ] && [ "$3" = i ]; } # relam, hex amname, amtype

hic_literal() {
    # hic_literal TEXT: set HIC_LITERAL to TEXT dollar-quoted with an unused tag.
    local n=0 tag
    while :; do
        tag="\$hic12_q$n\$"
        # A value ending in the tag minus its final $ would close it early too.
        case $1 in *"$tag"*|*"${tag%\$}") n=$((n + 1)) ;; *) break ;; esac
    done
    HIC_LITERAL="$tag$1$tag"
}

# Arbitrary SQL text travels as "x" plus its UTF-8 bytes in hex; OIDs and
# counts travel as decimal digits. Catalog text never reaches eval.
hic_decode() {
    # hic_decode HEX: set HIC_TEXT; refuses malformed hex and NUL bytes.
    local hex=${1#x} escapes= pair
    case $1 in x*) ;; *) return 1 ;; esac
    case $hex in *[!0-9a-f]*) return 1 ;; esac
    [ $((${#hex} % 2)) -eq 0 ] || return 1
    while [ -n "$hex" ]; do
        pair=${hex:0:2} hex=${hex:2}
        [ "$pair" != 00 ] || return 1
        escapes="$escapes\\x$pair"
    done
    printf -v HIC_TEXT '%b' "$escapes"
}

hic_csv() {
    # hic_csv FILE FIELD...: append one CSV row, every field quoted.
    local file=$1 row= sep= value
    shift
    for value in "$@"; do
        value=${value//\"/\"\"}
        row="$row$sep\"$value\""
        sep=,
    done
    printf '%s\n' "$row" >> "$file"
}

# Local evidence is one complete value per file, followed by one newline.
hic_put() { printf '%s\n' "$3" > "$1/$2"; }                    # hic_put DIR NAME VALUE
hic_get() {
    # A sentinel protects trailing newlines from command substitution. Remove
    # that sentinel and only the one newline hic_put appended to the value.
    HIC_VALUE=$(cat "$1/$2" && printf '.') || return 2
    HIC_VALUE=${HIC_VALUE%.}
    HIC_VALUE=${HIC_VALUE%$'\n'}
}
hic_save() {
    # hic_save DIR VAR...: write each variable VAR to DIR/VAR.
    local _dir=$1 _var
    shift
    for _var in "$@"; do
        hic_put "$_dir" "$_var" "${!_var-}" || return 2
    done
}
hic_load() {
    # hic_load DIR VAR...: read each DIR/VAR into the caller's variable VAR.
    local _dir=$1 _var
    shift
    for _var in "$@"; do
        hic_get "$_dir" "$_var" || return 2
        printf -v "$_var" '%s' "$HIC_VALUE"
    done
}
hic_contains() {
    # hic_contains DIR NAME LINE: read the complete set, then set HIC_FOUND.
    # Read failure is distinct from a successfully read set without LINE.
    hic_get "$1" "$2" || return 2
    HIC_FOUND=false
    case $'\n'"$HIC_VALUE"$'\n' in *$'\n'"$3"$'\n'*) HIC_FOUND=true ;; esac
}
hic_add_line() {
    # hic_add_line FILE LINE: add LINE only after the existing set was read.
    hic_contains "${1%/*}" "${1##*/}" "$2" || return 2
    [ "$HIC_FOUND" = true ] || printf '%s\n' "$2" >> "$1"
}

# ---- session layer: one psql process per named session ------------------------
# Bash 3.2 has no associative arrays, so session NAME keeps its state in
# HIC_DIR_NAME, HIC_FDIN_NAME, HIC_FDOUT_NAME, HIC_PID_NAME and HIC_SEQ_NAME.
# eval is used only to open and close those numbered file descriptors.

hic_session() {
    # hic_session NAME: copy NAME's state into the caller's s_* variables.
    local v
    case $1 in ''|*[!a-z]*) return 2 ;; esac
    v=HIC_DIR_$1;   s_dir=${!v-}
    v=HIC_FDIN_$1;  s_in=${!v-}
    v=HIC_FDOUT_$1; s_out=${!v-}
    v=HIC_PID_$1;   s_pid=${!v-}
    v=HIC_SEQ_$1;   s_seq=${!v-0}
}
hic_session_set() { printf -v "HIC_$2_$1" '%s' "$3"; }        # hic_session_set NAME FIELD VALUE

hic_open() {
    # hic_open NAME FD_IN FD_OUT: start psql reading SQL from a FIFO that the
    # runner writes through FD_IN; results come back through FD_OUT.
    local name=$1 fdin=$2 fdout=$3 dir="$HIC_OUT/session-$1" other s_dir s_in s_out s_pid s_seq
    case $name in ''|*[!a-z]*) return 2 ;; esac
    hic_uint "$fdin" && hic_uint "$fdout" || return 2
    trap '' PIPE    # writing to an exited psql must fail, not end the runner
    mkdir -m 700 "$dir" && mkfifo -m 600 "$dir/in" "$dir/out" || return 2
    : > "$dir/stderr.log" && : > "$dir/transcript.sql" || return 2
    (
        # Close the other sessions' descriptors in this child. A copy left
        # open here would keep their psql from ever reading end of input.
        for other in $HIC_OPEN; do
            hic_session "$other"
            eval "exec $s_in>&- $s_out<&-"
        done
        export PGCLIENTENCODING=UTF8 LC_ALL=C
        exec "$HIC_PSQL" -X -w -q -A -t -v ON_ERROR_STOP=0 -v VERBOSITY=verbose \
            < "$dir/in" > "$dir/out" 2>> "$dir/stderr.log"
    ) &
    hic_session_set "$name" PID "$!"
    hic_session_set "$name" DIR "$dir"
    hic_session_set "$name" FDIN "$fdin"
    hic_session_set "$name" FDOUT "$fdout"
    hic_session_set "$name" SEQ 0
    eval "exec $fdin>\"\$dir/in\" $fdout<\"\$dir/out\""
    HIC_OPEN="$HIC_OPEN $name"
    hic_sql "$name" 'SELECT /* wiki_hic12_runner_hello */ 1;' && [ "$HIC_VALUE" = 1 ] && return 0
    hic_warn "session $name did not start; see $dir/stderr.log"
    return 2
}

hic_close() {
    # hic_close NAME [kill]: closing psql's input ends it; the server then
    # rolls back any transaction that is still open.
    local name=$1 other rest= rc=0 s_dir s_in s_out s_pid s_seq
    hic_session "$name" || return 2
    [ -n "$s_pid" ] || return 0
    if [ "${2:-}" = kill ]; then kill "$s_pid" 2> /dev/null; fi
    eval "exec $s_in>&-"
    wait "$s_pid" || rc=$?
    eval "exec $s_out<&-"
    hic_session_set "$name" PID ''
    for other in $HIC_OPEN; do [ "$other" = "$name" ] || rest="$rest $other"; done
    HIC_OPEN=$rest
    return "$rc"
}

hic_cleanup() {
    local name
    for name in $HIC_OPEN; do hic_close "$name" kill || :; done
}

hic_request() {
    # hic_request NAME FILE: send FILE and one \echo sync line, then copy the
    # stdout lines before the sync line to the session's response file, and
    # this request's stderr to stderr.last. Sets HIC_ERROR and HIC_SQLSTATE
    # (last statement) and HIC_FAILED (any statement). Returns 2 when psql
    # ends first or FILE does not end with a semicolon.
    local name=$1 file=$2 marker line before status= s_dir s_in s_out s_pid s_seq
    hic_session "$name" || return 2
    HIC_ERROR= HIC_SQLSTATE= HIC_FAILED=true
    : > "$s_dir/response" && : > "$s_dir/stderr.last" || return 2
    if ! grep -v -E '^[[:space:]]*$' "$file" | tail -n 1 | grep -q -E ';[[:space:]]*$'; then
        hic_warn "refusing request without a final semicolon: $file"
        return 2
    fi
    s_seq=$((s_seq + 1))
    hic_session_set "$name" SEQ "$s_seq"
    marker="HIC12_SYNC_$$_${name}_$s_seq"
    before=$(wc -c < "$s_dir/stderr.log")
    { printf '%s\n' "-- request $s_seq"; cat "$file"; } >> "$s_dir/transcript.sql" || return 2
    { cat "$file"; printf '\n\\echo %s :ERROR :SQLSTATE\n' "$marker"; } >&"$s_in" 2> /dev/null || return 2
    while IFS= read -r line; do
        case $line in "$marker "*) status=${line#"$marker "}; break ;; esac
        printf '%s\n' "$line"
    done <&"$s_out" > "$s_dir/response"
    tail -c +"$((before + 1))" "$s_dir/stderr.log" > "$s_dir/stderr.last" || return 2
    [ -n "$status" ] || return 2    # psql ended before the sync line
    HIC_ERROR=${status%% *} HIC_SQLSTATE=${status#* }
    printf '%s\n' "-- sync $status" >> "$s_dir/transcript.sql" || return 2
    # ERROR describes only the last statement, so also scan the stderr slice.
    if [ "$HIC_ERROR" = false ] && ! grep -E -q "$HIC_ERROR_LINE" "$s_dir/stderr.last"; then
        HIC_FAILED=false
    fi
    return 0
}

hic_sql() {
    # hic_sql NAME TEXT: returns 0 when every statement succeeded, 1 when one
    # failed, 2 when the session is gone. HIC_VALUE is the first result line.
    local s_dir s_in s_out s_pid s_seq
    hic_session "$1" || return 2
    HIC_VALUE=
    printf '%s\n' "$2" > "$s_dir/request" || return 2
    hic_request "$1" "$s_dir/request" || return 2
    if [ -s "$s_dir/response" ]; then IFS= read -r HIC_VALUE < "$s_dir/response" || :; fi
    [ "$HIC_FAILED" = false ] || return 1
}

hic_data() {
    # hic_data FILE SQL: run SQL in the checking session; save its response.
    local s_dir s_in s_out s_pid s_seq
    hic_session "$HIC_S" || return 2
    hic_sql "$HIC_S" "$2" || return
    cp "$s_dir/response" "$1"
}

hic_parse_error() {
    # hic_parse_error NAME [PATTERN]: set HIC_ERR_CODE, HIC_ERR_MESSAGE,
    # HIC_ERR_DETAIL and HIC_ERR_HINT from the first report of NAME's last
    # request that matches PATTERN (default: an error report). Returns 1 when
    # that request printed no such report.
    local line rest pattern=${2:-$HIC_ERROR_LINE} s_dir s_in s_out s_pid s_seq
    hic_session "$1" || return 1
    HIC_ERR_CODE= HIC_ERR_MESSAGE= HIC_ERR_DETAIL= HIC_ERR_HINT=
    while IFS= read -r line || [ -n "$line" ]; do
        if [ -z "$HIC_ERR_CODE" ]; then
            [[ $line =~ $pattern ]] || continue
            rest=${line#*:  }                   # "SQLSTATE: message"
            HIC_ERR_CODE=${rest%%:*}
            HIC_ERR_MESSAGE=${rest#*: }
        elif [[ $line =~ $HIC_REPORT_LINE ]]; then
            break                               # the next report starts
        else
            case $line in
                'DETAIL:  '*) [ -n "$HIC_ERR_DETAIL" ] || HIC_ERR_DETAIL=${line#'DETAIL:  '} ;;
                'HINT:  '*) [ -n "$HIC_ERR_HINT" ] || HIC_ERR_HINT=${line#'HINT:  '} ;;
            esac
        fi
    done < "$s_dir/stderr.last"
    [ -n "$HIC_ERR_CODE" ]
}

# ---- setup: temporary tables, checking transaction, planner profiles ---------

hic_profiles() {
    # hic_profiles heap|index|control: discourage competing scan methods;
    # the index profile's second variant also penalizes sorting. Index-only
    # paths are disabled; executed-plan validation enforces the scan contract.
    # control turns every scan method and sorting back on.
    local seq=on idx=on sort=on only=on bitmap=on
    case $1 in
        heap) idx=off only=off bitmap=off ;;
        index) seq=off only=off bitmap=off
               [ "${HIC_VARIANT:-1}" = 1 ] || sort=off ;;
        control) ;;
        *) return 1 ;;
    esac
    hic_sql "$HIC_S" "SET /* wiki_hic12_profile_seq */ LOCAL enable_seqscan = $seq;
SET /* wiki_hic12_profile_index */ LOCAL enable_indexscan = $idx;
SET /* wiki_hic12_profile_only */ LOCAL enable_indexonlyscan = $only;
SET /* wiki_hic12_profile_bitmap */ LOCAL enable_bitmapscan = $bitmap;
SET /* wiki_hic12_profile_sort */ LOCAL enable_sort = $sort;
SET /* wiki_hic12_profile_quote */ LOCAL quote_all_identifiers = off;
SET /* wiki_hic12_profile_path */ LOCAL search_path = pg_catalog, pg_temp;"
}

hic_install() {
    # Create the two temporary tables, the only installed objects. The session
    # pins client_min_messages to its default, notice, so a role or database
    # value cannot hide the warnings that fail a count attempt. Each count row
    # also records the damage-hiding settings, which a reload can change.
    hic_sql "$HIC_S" "SET /* wiki_hic12_install_path */ search_path = pg_catalog, pg_temp;
SET /* wiki_hic12_install_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_install_lock */ lock_timeout = '2s';
SET /* wiki_hic12_install_idle */ idle_in_transaction_session_timeout = 0;
SET /* wiki_hic12_install_messages */ client_min_messages = notice;
BEGIN /* wiki_hic12_install_begin */;
CREATE /* wiki_hic12_count_table */ TEMP TABLE hic_count (
    attempt_id bigint PRIMARY KEY, row_count bigint NOT NULL CHECK (row_count >= 0),
    transaction_id bigint DEFAULT txid_current(),
    snapshot_text text DEFAULT txid_current_snapshot()::text,
    backend_pid integer DEFAULT pg_backend_pid(), checking_role text DEFAULT current_user,
    captured timestamptz DEFAULT clock_timestamp(),
    settings jsonb DEFAULT jsonb_build_object(
        'seq', current_setting('enable_seqscan'), 'index', current_setting('enable_indexscan'),
        'only', current_setting('enable_indexonlyscan'), 'bitmap', current_setting('enable_bitmapscan'),
        'sort', current_setting('enable_sort'), 'statement_timeout', current_setting('statement_timeout'),
        'lock_timeout', current_setting('lock_timeout'), 'old_snapshot_threshold', current_setting('old_snapshot_threshold'),
        'zero_damaged_pages', current_setting('zero_damaged_pages'),
        'ignore_checksum_failure', current_setting('ignore_checksum_failure'),
        'track_counts', current_setting('track_counts'),
        'parallel', current_setting('max_parallel_workers_per_gather'),
        'rls', current_setting('row_security'), 'path', current_setting('search_path'),
        'quote', current_setting('quote_all_identifiers'),
        'isolation', current_setting('transaction_isolation'), 'readonly', current_setting('transaction_read_only'))
) USING heap;
CREATE /* wiki_hic12_plan_table */ TEMP TABLE hic_plan (
    attempt_id bigint PRIMARY KEY, document jsonb NOT NULL
) USING heap;
COMMIT /* wiki_hic12_install_commit */;" || return
    hic_data "$HIC_OUT/owned" "WITH /* wiki_hic12_owned */ tables AS (
 SELECT oid, reltoastrelid FROM pg_catalog.pg_class
 WHERE relnamespace = pg_my_temp_schema() AND relname IN ('hic_count','hic_plan')
), rels AS (SELECT oid FROM tables UNION SELECT reltoastrelid FROM tables WHERE reltoastrelid <> 0)
SELECT /* wiki_hic12_owned_ids */ oid FROM rels
UNION SELECT indexrelid FROM pg_catalog.pg_index JOIN rels ON indrelid = oid;" || return
    hic_sql "$HIC_S" "SELECT /* wiki_hic12_temp_namespace */ 'x' || encode(convert_to(nspname::text,'UTF8'),'hex')
FROM pg_catalog.pg_namespace WHERE oid = pg_my_temp_schema();" || return
    HIC_CAPTURE_SCHEMA=$HIC_VALUE
}

hic_start() {
    # Open the repeatable-read, read-only checking transaction; record run facts.
    hic_literal "$HIC_SCHEMA"
    hic_sql "$HIC_S" "BEGIN /* wiki_hic12_begin */ ISOLATION LEVEL REPEATABLE READ READ ONLY;
SET /* wiki_hic12_scope_path */ LOCAL search_path = pg_catalog, pg_temp;
SET /* wiki_hic12_scope_quote */ LOCAL quote_all_identifiers = off;
SET /* wiki_hic12_scope_rls */ LOCAL row_security = off;
SET /* wiki_hic12_scope_parallel */ LOCAL max_parallel_workers_per_gather = 0;
SET /* wiki_hic12_scope_timeout */ LOCAL statement_timeout = '15min';
SET /* wiki_hic12_scope_lock */ LOCAL lock_timeout = '2s';" || return
    hic_data "$HIC_OUT/run-facts" "SELECT /* wiki_hic12_run_facts */ current_setting('server_version_num'),
 pg_is_in_recovery(), pg_backend_pid(), txid_current(), txid_current_snapshot()::text,
 'x' || encode(convert_to(current_user::text,'UTF8'),'hex'),
 COALESCE((SELECT oid FROM pg_catalog.pg_namespace WHERE nspname = $HIC_LITERAL), 0),
 'x' || encode(convert_to(current_database(),'UTF8'),'hex'),
 'x' || encode(convert_to(version(),'UTF8'),'hex'), transaction_timestamp(),
 current_setting('transaction_isolation'), current_setting('transaction_read_only'),
 'x'||encode(convert_to(session_user::text,'UTF8'),'hex'),
 (SELECT oid FROM pg_catalog.pg_database WHERE datname=current_database()),
 current_setting('zero_damaged_pages'), current_setting('ignore_checksum_failure'),
 current_setting('track_counts'), current_setting('old_snapshot_threshold'),
 current_setting('data_checksums');" || return
    IFS='|' read -r HIC_SERVER HIC_RECOVERY HIC_BACKEND HIC_XID HIC_SNAPSHOT HIC_ROLE_HEX \
        HIC_SCHEMA_OID HIC_DATABASE HIC_VERSION HIC_STARTED HIC_ISOLATION HIC_READONLY \
        HIC_SESSION_ROLE HIC_DATABASE_OID HIC_ZERO_DAMAGED HIC_IGNORE_CHECKSUM \
        HIC_TRACK_COUNTS HIC_OLD_SNAPSHOT HIC_DATA_CHECKSUMS < "$HIC_OUT/run-facts" || return 1
    if [ "$HIC_SERVER" != 120002 ] || [ "$HIC_RECOVERY" != f ] || [ "$HIC_SCHEMA_OID" = 0 ] \
        || [ "$HIC_ISOLATION" != 'repeatable read' ] || [ "$HIC_READONLY" != on ]; then
        HIC_STOP='requires PostgreSQL 12.2 primary, an existing schema, repeatable read and read only'
        return 1
    fi
    hic_server_guard "$HIC_ZERO_DAMAGED" "$HIC_IGNORE_CHECKSUM" "$HIC_TRACK_COUNTS" "$HIC_OLD_SNAPSHOT" \
        || return 1
    hic_profiles control || return
    hic_data "$HIC_OUT/settings.json" "SELECT /* wiki_hic12_run_settings */ jsonb_object_agg(name,current_setting(name))
FROM unnest(ARRAY['enable_seqscan','enable_indexscan','enable_indexonlyscan','enable_bitmapscan','enable_sort',
'max_parallel_workers_per_gather','row_security','search_path','quote_all_identifiers','statement_timeout','lock_timeout',
'transaction_isolation','transaction_read_only','old_snapshot_threshold','client_encoding',
'zero_damaged_pages','ignore_checksum_failure','track_counts','data_checksums',
'idle_in_transaction_session_timeout','client_min_messages']) setting(name);"
}

hic_server_guard() {
    # hic_server_guard ZERO_DAMAGED IGNORE_CHECKSUM TRACK_COUNTS OLD_SNAPSHOT:
    # refuse settings that can hide damage or bias a count. zero_damaged_pages
    # zeroes a bad page in a buffer a retry then reuses; ignore_checksum_failure
    # uses a failed page and reports only a class-01 warning; track_counts off
    # disables the unresolved-TID counters; old_snapshot_threshold lets pruning
    # remove rows the index-path heap fetch then misses without an error.
    # hic_start checks the run's first values and hic_read_count each count's.
    local why=
    [ "$1" = off ] || why="${why:+$why, }zero_damaged_pages=$1"
    [ "$2" = off ] || why="${why:+$why, }ignore_checksum_failure=$2"
    [ "$3" = on ] || why="${why:+$why, }track_counts=$3"
    [ "$4" = -1 ] || why="${why:+$why, }old_snapshot_threshold=$4"
    [ -z "$why" ] && return 0
    HIC_STOP="refusing server settings: $why"
    return 1
}

hic_run_error() {
    # hic_run_error STAGE [SQLSTATE [MESSAGE]]: one run-errors.csv row.
    hic_csv "$HIC_OUT/run-errors.csv" "$1" "${2:-}" "${3:-run failed}" || return 2
}

# ---- inventory: SQL returns facts, Bash classifies them ------------------------
# Fact lines are '|'-separated in their query's column order. These readers
# name the columns, setting the r_* and i_* variables; a non-empty *_extra
# means the line had too many fields.

hic_read_relation() {   # columns of wiki_hic12_object_facts
    IFS='|' read -r r_oid r_nspoid r_schema r_name r_qualified r_kind r_am r_amname r_amtype \
        r_persistence r_populated r_partition r_rls r_force_rls r_filenode \
        r_other_temp r_usage r_select r_parents r_extra
}
hic_read_index() {      # columns of wiki_hic12_index_facts
    IFS='|' read -r i_oid i_relid i_schema i_name i_ident i_kind i_am i_amname i_amtype \
        i_partial i_valid i_ready i_live i_checkxmin i_xmin i_nkeys i_filenode i_pred i_extra
}

hic_inventory() {
    # Classify every relation in the schema, plus the partitions that its
    # partitioned tables reach in other schemas when HIC_EXTERNAL is true.
    # The file number is pg_class.relfilenode as of the snapshot or, for a
    # mapped catalog, whose relfilenode is 0, the relation mapper's number.
    local line oid dir scope reason permission drift ids
    hic_data "$HIC_OUT/objects.raw" "WITH /* wiki_hic12_scope */ RECURSIVE scope(oid) AS (
 SELECT oid FROM pg_catalog.pg_class WHERE relnamespace = $HIC_SCHEMA_OID
 UNION SELECT inh.inhrelid FROM scope s
 JOIN pg_catalog.pg_class p ON p.oid = s.oid AND p.relkind = 'p'
 JOIN pg_catalog.pg_inherits inh ON inh.inhparent = p.oid
 JOIN pg_catalog.pg_class c ON c.oid = inh.inhrelid AND c.relispartition
 WHERE $HIC_EXTERNAL
)
SELECT /* wiki_hic12_object_facts */ c.oid, c.relnamespace,
 'x'||encode(convert_to(n.nspname::text,'UTF8'),'hex'),
 'x'||encode(convert_to(c.relname::text,'UTF8'),'hex'),
 'x'||encode(convert_to(format('%I.%I',n.nspname,c.relname),'UTF8'),'hex'),
 c.relkind, c.relam, 'x'||encode(convert_to(COALESCE(am.amname::text,''),'UTF8'),'hex'),
 COALESCE(am.amtype::text,'-'), c.relpersistence, c.relispopulated, c.relispartition,
 c.relrowsecurity, c.relforcerowsecurity, COALESCE(NULLIF(c.relfilenode,0),pg_relation_filenode(c.oid),0),
 pg_is_other_temp_schema(n.oid), has_schema_privilege(n.oid,'USAGE'),
 CASE WHEN c.relkind IN ('r','m') THEN has_table_privilege(c.oid,'SELECT') ELSE true END,
 'x'||encode(convert_to(ARRAY(SELECT inhparent FROM pg_catalog.pg_inherits WHERE inhrelid=c.oid ORDER BY inhparent)::text,'UTF8'),'hex')
FROM scope s JOIN pg_catalog.pg_class c ON c.oid=s.oid
JOIN pg_catalog.pg_namespace n ON n.oid=c.relnamespace
LEFT JOIN pg_catalog.pg_am am ON am.oid=c.relam ORDER BY c.oid;" || return
    : > "$HIC_OUT/relation-ids" || return 2
    while IFS= read -r line; do
        oid=${line%%|*}
        hic_uint "$oid" || return 1
        if grep -x -F -q "$oid" "$HIC_OUT/owned"; then continue; fi    # the runner's own tables
        dir="$HIC_OUT/relations/$oid"
        mkdir -p "$dir" && : > "$dir/index-ids" && : > "$dir/attempt-ids" || return 2
        printf '%s\n' "$oid" >> "$HIC_OUT/relation-ids" || return 2
        hic_put "$dir" facts "$line" || return 2
        hic_read_relation < "$dir/facts" && [ -n "$r_parents" ] && [ -z "$r_extra" ] || return 1
        scope=false reason=
        case $r_kind in
            r|m)
                if [ "$r_kind" = m ] && [ "$r_populated" != t ]; then reason=unpopulated_materialized_view
                elif ! hic_heap_am "$r_am" "$r_amname" "$r_amtype"; then reason=unsupported_table_access_method
                else scope=true
                fi ;;
            p) reason=partitioned_parent_summary ;;
            v) reason=view ;;
            f) reason=foreign_table ;;
            S) reason=sequence ;;
            i|I) reason=index_metadata ;;
            *) reason=outside_physical_heap_scope ;;
        esac
        permission=
        if [ "$r_other_temp" = t ]; then permission=unsupported_remote_temp
        elif [ "$r_usage" != t ]; then permission=schema_usage_denied
        elif [ "$r_select" != t ]; then permission=table_select_denied
        fi
        drift=                          # set later if identity changes after S
        hic_save "$dir" scope reason permission drift || return 2
    done < "$HIC_OUT/objects.raw"
    ids=$(paste -s -d, "$HIC_OUT/relation-ids")
    hic_data "$HIC_OUT/partitions.raw" "WITH /* wiki_hic12_partition_tree */ RECURSIVE tree(parent_oid,oid) AS (
 SELECT p.oid, inh.inhrelid FROM pg_catalog.pg_class p
 JOIN pg_catalog.pg_inherits inh ON inh.inhparent=p.oid
 WHERE p.relkind='p' AND p.oid IN (${ids:-0})
 UNION SELECT tree.parent_oid, inh.inhrelid FROM tree
 JOIN pg_catalog.pg_class p ON p.oid=tree.oid AND p.relkind='p'
 JOIN pg_catalog.pg_inherits inh ON inh.inhparent=p.oid
)
SELECT /* wiki_hic12_partition_facts */ tree.parent_oid,c.oid,c.relkind,c.relnamespace,c.relam,
 'x'||encode(convert_to(n.nspname::text,'UTF8'),'hex'),
 'x'||encode(convert_to(c.relname::text,'UTF8'),'hex')
FROM tree JOIN pg_catalog.pg_class c ON c.oid=tree.oid
JOIN pg_catalog.pg_namespace n ON n.oid=c.relnamespace ORDER BY 1,2;" || return
    hic_indexes
}

hic_indexes() {
    # Classify every index; generate orderings for eligible in-scope B-trees.
    local oid dir scope line iid idir eligible reason
    while IFS= read -r oid; do
        dir="$HIC_OUT/relations/$oid"
        hic_get "$dir" scope || return 2; scope=$HIC_VALUE
        hic_data "$dir/indexes.raw" "SELECT /* wiki_hic12_index_facts */ c.oid, i.indrelid,
 'x'||encode(convert_to(n.nspname::text,'UTF8'),'hex'),
 'x'||encode(convert_to(c.relname::text,'UTF8'),'hex'),
 'x'||encode(convert_to(quote_ident(c.relname),'UTF8'),'hex'),
 c.relkind, c.relam, 'x'||encode(convert_to(am.amname::text,'UTF8'),'hex'), am.amtype, i.indpred IS NOT NULL,
 i.indisvalid,i.indisready,i.indislive,i.indcheckxmin,i.xmin::text,i.indnkeyatts,
 COALESCE(NULLIF(c.relfilenode,0),pg_relation_filenode(c.oid),0),
 'x'||encode(convert_to(COALESCE(i.indpred::text,''),'UTF8'),'hex')
FROM pg_catalog.pg_index i JOIN pg_catalog.pg_class c ON c.oid=i.indexrelid
JOIN pg_catalog.pg_namespace n ON n.oid=c.relnamespace JOIN pg_catalog.pg_am am ON am.oid=c.relam
WHERE i.indrelid=$oid ORDER BY c.oid;" || return
        while IFS= read -r line; do
            iid=${line%%|*}
            hic_uint "$iid" || return 1
            idir="$HIC_OUT/indexes/$iid"
            mkdir -p "$idir" || return 2
            printf '%s\n' "$iid" >> "$dir/index-ids" || return 2
            hic_put "$idir" facts "$line" || return 2
            hic_read_index < "$idir/facts" && [ -n "$i_pred" ] && [ -z "$i_extra" ] || return 1
            reason=
            if [ "$i_kind" != i ]; then reason=nonphysical_index
            elif ! hic_btree_am "$i_am" "$i_amname" "$i_amtype"; then reason=unsupported_index_am
            elif [ "$i_partial" != f ]; then reason=partial_index
            elif [ "$i_valid" != t ]; then reason=invalid_index
            elif [ "$i_ready" != t ]; then reason=index_not_ready
            elif [ "$i_live" != t ]; then reason=index_not_live
            fi
            eligible=true
            [ -z "$reason" ] || eligible=false
            hic_save "$idir" eligible reason || return 2
            if [ "$scope" = true ] && [ "$eligible" = true ]; then
                hic_order "$iid" "$i_nkeys" || return
            else
                hic_put "$idir" generation relation_or_index_outside_supported_scope || return 2
            fi
        done < "$dir/indexes.raw"
    done < "$HIC_OUT/relation-ids"
}

hic_order() {
    # hic_order INDEX_OID KEY_COUNT: save the ORDER BY list that follows the
    # index's key order, or the reason the generator cannot represent it.
    local iid=$1 expected=$2 dir="$HIC_OUT/indexes/$1" why= n=0 order= part nulls
    local kind ref ns typtype expr type opname opschema
    local keyno expr_hex type_hex opc opc_ns coll coll_ns_hex coll_name_hex desc nulls_first
    local op op_ns op_name_hex op_schema_hex proc proc_ns attnum atttype extra
    hic_data "$dir/dependencies.raw" "WITH /* wiki_hic12_index_dependencies */ deps AS (
 SELECT d.* FROM pg_catalog.pg_depend d
 WHERE d.classid='pg_catalog.pg_class'::regclass AND d.objid=$iid
)
SELECT /* wiki_hic12_dependency_facts */ 'p',p.oid,p.pronamespace,'-' FROM deps d
JOIN pg_catalog.pg_proc p ON d.refclassid='pg_catalog.pg_proc'::regclass AND d.refobjid=p.oid
UNION SELECT 'o',p.oid,p.oprnamespace,'-' FROM deps d
JOIN pg_catalog.pg_operator p ON d.refclassid='pg_catalog.pg_operator'::regclass AND d.refobjid=p.oid
UNION SELECT 't',t.oid,t.typnamespace,t.typtype FROM deps d
JOIN pg_catalog.pg_type t ON d.refclassid='pg_catalog.pg_type'::regclass AND d.refobjid=t.oid
UNION SELECT 't',t.oid,t.typnamespace,t.typtype FROM deps d
JOIN pg_catalog.pg_index i ON i.indexrelid=$iid
JOIN pg_catalog.pg_attribute a ON a.attrelid=i.indrelid AND a.attnum>0 AND NOT a.attisdropped
 AND (d.refobjsubid=0 OR a.attnum=d.refobjsubid)
JOIN pg_catalog.pg_type t ON t.oid=a.atttypid
WHERE d.refclassid='pg_catalog.pg_class'::regclass AND d.refobjid=i.indrelid
UNION SELECT 't',t.oid,t.typnamespace,t.typtype FROM pg_catalog.pg_attribute a
JOIN pg_catalog.pg_type t ON t.oid=a.atttypid WHERE a.attrelid=$iid AND a.attnum>0;" || return
    # User-defined functions, operators and types (other than enums) are not reviewed.
    while IFS='|' read -r kind ref ns typtype; do
        hic_uint "$ref" && hic_uint "$ns" || return 1
        case $kind in
            p|o) if hic_user_oid "$ref" || [ "$ns" != "$HIC_CATALOG_NS" ]; then why=unreviewed_user_code_or_type; fi ;;
            t) if hic_user_oid "$ref" && [ "$typtype" != e ]; then why=unreviewed_user_code_or_type; fi ;;
            *) return 1 ;;
        esac
    done < "$dir/dependencies.raw"
    if [ -n "$why" ]; then hic_put "$dir" generation "$why" || return 2; return 0; fi
    hic_data "$dir/definition.sql" "SELECT /* wiki_hic12_index_definition */ pg_get_indexdef($iid);" || return
    hic_data "$dir/keys.raw" "SELECT /* wiki_hic12_key_facts */ k.n,
 'x'||encode(convert_to(pg_get_indexdef(i.indexrelid,k.n,false),'UTF8'),'hex'),
 'x'||encode(convert_to(format_type(a.atttypid,NULL),'UTF8'),'hex'),
 opc.oid, opc.opcnamespace, i.indcollation[k.n-1],
 'x'||encode(convert_to(COALESCE(cn.nspname::text,''),'UTF8'),'hex'),
 'x'||encode(convert_to(COALESCE(coll.collname::text,''),'UTF8'),'hex'),
 (i.indoption[k.n-1]&1)<>0,(i.indoption[k.n-1]&2)<>0,
 COALESCE(op.oid,0),COALESCE(op.oprnamespace,0),
 'x'||encode(convert_to(COALESCE(op.oprname::text,''),'UTF8'),'hex'),
 'x'||encode(convert_to(COALESCE(onsp.nspname::text,''),'UTF8'),'hex'),
 COALESCE(proc.oid,0),COALESCE(proc.pronamespace,0),i.indkey[k.n-1],a.atttypid,t.typtype
FROM pg_catalog.pg_index i CROSS JOIN LATERAL generate_series(1,i.indnkeyatts::integer) k(n)
JOIN pg_catalog.pg_attribute a ON a.attrelid=i.indexrelid AND a.attnum=k.n
JOIN pg_catalog.pg_type t ON t.oid=a.atttypid
JOIN pg_catalog.pg_opclass opc ON opc.oid=i.indclass[k.n-1]
LEFT JOIN pg_catalog.pg_collation coll ON coll.oid=i.indcollation[k.n-1]
LEFT JOIN pg_catalog.pg_namespace cn ON cn.oid=coll.collnamespace
LEFT JOIN LATERAL (
 SELECT o.* FROM pg_catalog.pg_amop ao JOIN pg_catalog.pg_operator o ON o.oid=ao.amopopr
 WHERE ao.amopfamily=opc.opcfamily AND ao.amoplefttype=opc.opcintype AND ao.amoprighttype=opc.opcintype
 AND ao.amopstrategy=CASE WHEN (i.indoption[k.n-1]&1)<>0 THEN 5 ELSE 1 END
 AND ao.amoppurpose='s' AND ao.amopmethod=403 ORDER BY o.oid LIMIT 1
) op ON true
LEFT JOIN pg_catalog.pg_namespace onsp ON onsp.oid=op.oprnamespace
LEFT JOIN pg_catalog.pg_proc proc ON proc.oid=op.oprcode
WHERE i.indexrelid=$iid ORDER BY k.n;" || return
    while IFS='|' read -r keyno expr_hex type_hex opc opc_ns coll coll_ns_hex coll_name_hex desc nulls_first \
            op op_ns op_name_hex op_schema_hex proc proc_ns attnum atttype typtype extra; do
        n=$((n + 1))
        [ "$keyno" = "$n" ] && [ -n "$typtype" ] && [ -z "$extra" ] || return 1
        # The ordering operator, its function and the opclass must be built in.
        if hic_user_oid "$opc" || [ "$opc_ns" != "$HIC_CATALOG_NS" ] \
            || [ "$op" = 0 ] || hic_user_oid "$op" || [ "$op_ns" != "$HIC_CATALOG_NS" ] \
            || hic_user_oid "$proc" || [ "$proc_ns" != "$HIC_CATALOG_NS" ]; then
            why=ordering_not_represented_by_generator
        fi
        if hic_user_oid "$atttype" && [ "$typtype" != e ]; then why=unreviewed_user_code_or_type; fi
        hic_decode "$expr_hex" || return 1; expr=$HIC_TEXT
        hic_decode "$type_hex" || return 1; type=$HIC_TEXT
        # ORDER BY reads a bare integer as a column position and rejects other
        # bare constants, so a literal key is cast to its own type instead.
        part="($expr)"
        if [ "$attnum" = 0 ] && [[ $expr =~ ^[(]*[0-9]+([.][0-9]+)?[)]*$ ]]; then part="CAST($expr AS $type)"; fi
        if [ "$coll" != 0 ]; then
            [ "$coll_ns_hex" != x ] && [ "$coll_name_hex" != x ] || why=ordering_not_represented_by_generator
            # Quote identifiers with the server, not with shell substitutions.
            hic_sql "$HIC_S" "SELECT /* wiki_hic12_quote_collation */ 'x'||encode(convert_to(quote_ident(n.nspname)||'.'||quote_ident(c.collname),'UTF8'),'hex')
FROM pg_catalog.pg_collation c JOIN pg_catalog.pg_namespace n ON n.oid=c.collnamespace WHERE c.oid=$coll;" || return
            hic_decode "$HIC_VALUE" || return 1
            part="$part COLLATE $HIC_TEXT"
        fi
        hic_decode "$op_name_hex" || return 1; opname=$HIC_TEXT
        hic_decode "$op_schema_hex" || return 1; opschema=$HIC_TEXT
        [ "$opschema" = pg_catalog ] || why=ordering_not_represented_by_generator
        nulls=LAST; [ "$nulls_first" != t ] || nulls=FIRST
        order="${order:+$order, }$part USING OPERATOR(pg_catalog.$opname) NULLS $nulls"
    done < "$dir/keys.raw"
    [ "$n" = "$expected" ] && [ "$n" -gt 0 ] || why=ordering_not_represented_by_generator
    hic_put "$dir" generation "$why" || return 2
    if [ -z "$why" ]; then hic_put "$dir" order "$order" || return 2; fi
    return 0
}

# ---- plan decoding: the query exposes fields; every gate below is Bash --------

hic_plan_facts() {
    # hic_plan_facts ATTEMPT_ID FILE: decode the imported plan into D (document),
    # N (node), P (property) and O (Output element) records.
    hic_data "$2" "WITH /* wiki_hic12_plan_walk */ RECURSIVE doc AS (
 SELECT document d FROM pg_temp.hic_plan WHERE attempt_id=$1
), walk(path,node) AS (
 SELECT '0'::text,d#>'{0,Plan}' FROM doc WHERE jsonb_typeof(d#>'{0,Plan}')='object'
 UNION ALL SELECT w.path||'.'||wc.ordinality,wc.value
 FROM walk w CROSS JOIN LATERAL jsonb_array_elements(
 CASE WHEN jsonb_typeof(w.node->'Plans')='array' THEN w.node->'Plans' ELSE '[]'::jsonb END
 ) WITH ORDINALITY wc(value,ordinality)
), records(path,rank,ord,line) AS (
 SELECT '',0,0::bigint,concat_ws('|','D',jsonb_typeof(d),
 CASE WHEN jsonb_typeof(d)='array' THEN jsonb_array_length(d)::text ELSE '-' END,
 COALESCE(jsonb_typeof(d->0),'-'),COALESCE(jsonb_typeof(d#>'{0,Execution Time}'),'-'),
 COALESCE(d#>>'{0,Execution Time}','-')) FROM doc
 UNION ALL SELECT path,1,0,concat_ws('|','N',path,jsonb_typeof(node),
 COALESCE(jsonb_typeof(node->'Plans'),'-'),
 CASE WHEN jsonb_typeof(node->'Plans')='array' THEN jsonb_array_length(node->'Plans')::text ELSE '0' END)
 FROM walk
 UNION ALL SELECT path,2,0,concat_ws('|','P',path,
 'x'||encode(convert_to(k.key,'UTF8'),'hex'),jsonb_typeof(k.value),
 'x'||encode(convert_to(CASE WHEN jsonb_typeof(k.value)='string' THEN k.value#>>'{}' ELSE k.value::text END,'UTF8'),'hex'))
 FROM walk CROSS JOIN LATERAL jsonb_each(CASE WHEN jsonb_typeof(node)='object' THEN node ELSE '{}'::jsonb END) k
 WHERE k.key<>'Plans'
 UNION ALL SELECT path,3,o.ordinality,concat_ws('|','O',path,o.ordinality,jsonb_typeof(o.value),
 'x'||encode(convert_to(COALESCE(o.value#>>'{}',''),'UTF8'),'hex'))
 FROM walk CROSS JOIN LATERAL jsonb_array_elements(
 CASE WHEN jsonb_typeof(node->'Output')='array' THEN node->'Output' ELSE '[]'::jsonb END
 ) WITH ORDINALITY o(value,ordinality)
)
SELECT /* wiki_hic12_plan_fields */ line FROM records ORDER BY path,rank,ord,line;"
}

hic_reject() { HIC_REASONS="${HIC_REASONS:+$HIC_REASONS,}$1"; }

hic_node_finish() {
    # Check the node just read; uses hic_validate's locals (dynamic scope).
    local want=1                        # expected children: 1, or 0 for a scan
    [ -n "${nodepath:-}" ] || return 0
    [ "$nodetype" = object ] || hic_reject node_shape
    case $plans_type in -|array) ;; *) hic_reject children_type ;; esac
    case $kind in
        'Seq Scan'|'Index Scan')
            want=0
            sources=$((sources + 1))
            [ "$schema" = "$HIC_REL_SCHEMA" ] && [ "$relation" = "$HIC_REL_NAME" ] || hic_reject source_identity
            case $HIC_PHASE:$kind in 'heap:Seq Scan'|'index:Index Scan') ;; *) hic_reject source_kind ;; esac
            if [ "$kind" = 'Index Scan' ]; then
                [ -n "$indexname" ] && [ "$indexname" != x ] || hic_reject index_identity
                case $direction in Forward|Backward) ;; *) hic_reject scan_direction ;; esac
                HIC_ACTUAL_NAME=$indexname
            fi ;;
        ModifyTable)
            inserts=$((inserts + 1))
            [ "$nodepath" = 0 ] && [ "$operation" = Insert ] && [ "$schema" = "$HIC_CAPTURE_SCHEMA" ] \
                && [ "$relation" = "$HIC_COUNT_HEX" ] && hic_zero "$rows" || hic_reject capture_target ;;
        Aggregate)
            aggregates=$((aggregates + 1))
            [ "$strategy" = Plain ] && [ "$partial" = Simple ] && hic_one "$rows" || hic_reject plain_count
            [ "$output_type" = array ] && [ "$marker" = "$expected_marker" ] \
                && [ "$count_output" = 'count(*)' ] || hic_reject attempt_binding ;;
        Sort|Result|'Subquery Scan'|Materialize) ;;
        *) hic_reject node_kind ;;
    esac
    [ "$children" = "$want" ] || hic_reject child_count
    hic_one "$loops" && hic_decimal "$rows" || hic_reject execution_fields
}

hic_validate() {
    # hic_validate FACTS_FILE: set HIC_ACCEPTED, HIC_REASONS, HIC_ACTUAL_NAME
    # and HIC_EXECUTION_MS. Never changes published evidence.
    local record f2 f3 f4 f5 f6 key value expected_marker="'$HIC_ATTEMPT_ID'::bigint"
    local document=0 nodes=0 inserts=0 aggregates=0 sources=0
    # The current node's fields, checked by hic_node_finish.
    local nodepath= nodetype plans_type children kind schema relation operation rows loops
    local strategy partial direction indexname output_type marker count_output
    HIC_REASONS= HIC_ACTUAL_NAME= HIC_ACCEPTED=false HIC_EXECUTION_MS=
    while IFS='|' read -r record f2 f3 f4 f5 f6; do
        case $record in
            D)  # D|document type|array length|element type|time type|Execution Time
                document=$((document + 1)) HIC_EXECUTION_MS=$f6
                [ "$f2" = array ] && [ "$f3" = 1 ] && [ "$f4" = object ] && [ "$f5" = number ] \
                    && hic_decimal "$f6" || hic_reject document_shape ;;
            N)  # N|path|node type|Plans type|child count: a new node starts
                hic_node_finish
                nodepath=$f2 nodetype=$f3 plans_type=$f4 children=$f5
                kind= schema= relation= operation= rows= loops= strategy= partial= direction=
                indexname= output_type= marker= count_output=
                nodes=$((nodes + 1)) ;;
            P)  # P|path|key hex|value type|value hex
                [ "$f2" = "$nodepath" ] || hic_reject field_path
                hic_decode "$f3" || { hic_reject field_encoding; continue; }
                key=$HIC_TEXT
                hic_decode "$f5" || { hic_reject field_encoding; continue; }
                value=$HIC_TEXT
                case $key in
                    'Node Type'|Schema|'Relation Name'|Operation|Strategy|'Partial Mode'|'Scan Direction'|'Index Name'|'Parent Relationship')
                        [ "$f4" = string ] || hic_reject field_type ;;
                    'Actual Rows'|'Actual Loops') [ "$f4" = number ] || hic_reject field_type ;;
                esac
                case $key in
                    'Node Type') kind=$value ;;
                    Schema) schema=$f5 ;;
                    'Relation Name') relation=$f5 ;;
                    Operation) operation=$value ;;
                    Strategy) strategy=$value ;;
                    'Partial Mode') partial=$value ;;
                    'Actual Rows') rows=$value ;;
                    'Actual Loops') loops=$value ;;
                    'Scan Direction') direction=$value ;;
                    'Index Name') indexname=$f5 ;;
                    Output) output_type=$f4 ;;
                    'Parallel Aware') [ "$f4" = boolean ] && [ "$value" = false ] || hic_reject parallel ;;
                    'Parent Relationship') case $value in InitPlan|SubPlan) hic_reject subplan ;; esac ;;
                    Filter|'Index Cond'|'Recheck Cond'|'Join Filter'|'One-Time Filter'|'Rows Removed by Filter'|\
                    'Rows Removed by Index Recheck'|'Group Key'|'Grouping Sets'|'Hash Cond'|'Merge Cond'|\
                    InitPlan|SubPlan|Workers|'Workers Planned'|'Workers Launched') hic_reject restricted_or_parallel ;;
                esac ;;
            O)  # O|path|position|element type|element hex
                [ "$f2" = "$nodepath" ] && [ "$f4" = string ] || hic_reject output_shape
                hic_decode "$f5" || { hic_reject output_encoding; continue; }
                case $f3 in 1) marker=$HIC_TEXT ;; 2) count_output=$HIC_TEXT ;; esac ;;
            *) hic_reject unknown_record ;;
        esac
    done < "$1"
    hic_node_finish
    [ "$document" = 1 ] && [ "$nodes" -gt 0 ] && [ "$inserts" = 1 ] && [ "$aggregates" = 1 ] \
        && [ "$sources" = 1 ] || hic_reject plan_counts
    [ -n "$HIC_REASONS" ] || HIC_ACCEPTED=true
    return 0
}

# ---- attempts and recovery ----------------------------------------------------

hic_prepare_attempt() {
    # Allocate attempt HIC_ATTEMPT_ID (directory HIC_A), write its count
    # statement, save the index TID counters, open its savepoint and apply
    # its planner profile.
    local source
    HIC_ATTEMPT_ID=$((HIC_ATTEMPT_ID + 1))
    HIC_A="$HIC_OUT/attempts/$HIC_ATTEMPT_ID"
    mkdir -p "$HIC_A" "$HIC_OUT/plans" || return 2
    hic_put "$HIC_A" relation "$HIC_RELATION" && hic_put "$HIC_A" phase "$HIC_PHASE" \
        && hic_put "$HIC_A" intended "$HIC_INTENDED" && hic_put "$HIC_A" variant "$HIC_VARIANT" \
        && hic_put "$HIC_A" started "$(date -u +%Y-%m-%dT%H:%M:%SZ)" || return 2
    printf '%s\n' "$HIC_ATTEMPT_ID" >> "$HIC_OUT/relations/$HIC_RELATION/attempt-ids" || return 2
    hic_decode "$HIC_REL_QUALIFIED" || return 1
    source="ONLY $HIC_TEXT"
    if [ "$HIC_PHASE" = index ]; then
        # An inner ORDER BY in the index's key order invites a full index scan.
        # The inner SELECT has no output column, so ORDER BY's output-name rule
        # cannot capture a key that is a bare column name such as "?column?".
        hic_get "$HIC_OUT/indexes/$HIC_INTENDED" order || return 2
        source="(SELECT /* wiki_hic12_full_order */ FROM $source ORDER BY $HIC_VALUE) AS ordered_rows"
        hic_tid_counters "$HIC_A/tids-before" || return
    fi
    printf '%s\n' "EXPLAIN /* wiki_hic12_executed_count */ (ANALYZE, VERBOSE, BUFFERS, SETTINGS, TIMING OFF, SUMMARY ON, FORMAT JSON)
INSERT /* wiki_hic12_capture */ INTO pg_temp.hic_count(attempt_id,row_count)
SELECT /* wiki_hic12_exact_count */ $HIC_ATTEMPT_ID::bigint,count(*) FROM $source;" > "$HIC_A/count.sql" || return 2
    hic_sql "$HIC_S" 'SAVEPOINT /* wiki_hic12_attempt_boundary */ hic_attempt;' || return
    hic_profiles "$HIC_PHASE"
}

hic_tid_counters() {
    # hic_tid_counters FILE: save "index|returned|fetched" for every index of
    # the relation. These are this backend's per-transaction counts of TIDs an
    # index scan returned and of visible heap rows those TIDs reached; the
    # backend reports them only after the transaction, so they are exact here.
    local ids
    ids=$(paste -s -d, "$HIC_OUT/relations/$HIC_RELATION/index-ids") || return 2
    hic_data "$1" "SELECT /* wiki_hic12_index_tid_counters */ c.i, pg_stat_get_xact_tuples_returned(c.i),
 pg_stat_get_xact_tuples_fetched(c.i) FROM unnest(ARRAY[${ids:-0}]::oid[]) AS c(i) ORDER BY c.i;"
}

hic_unresolved_tids() {
    # Set HIC_TIDS_RETURNED and HIC_UNRESOLVED for the actual index: TIDs it
    # returned during this attempt, and how many of them reached no visible
    # heap row (dead or invisible versions, or entries whose heap row is gone).
    # The rows fetched must equal the exact count, else both stay empty.
    local iid r f before_r= before_f= after_r= after_f=
    HIC_TIDS_RETURNED= HIC_UNRESOLVED=
    [ "$HIC_PHASE" = index ] && [ -n "$HIC_ACTUAL_INDEX" ] && hic_uint "$HIC_COUNT" || return 0
    hic_tid_counters "$HIC_A/tids-after" || return
    while IFS='|' read -r iid r f; do
        [ "$iid" = "$HIC_ACTUAL_INDEX" ] && before_r=$r before_f=$f
    done < "$HIC_A/tids-before"
    while IFS='|' read -r iid r f; do
        [ "$iid" = "$HIC_ACTUAL_INDEX" ] && after_r=$r after_f=$f
    done < "$HIC_A/tids-after"
    hic_uint "$before_r" && hic_uint "$before_f" && hic_uint "$after_r" && hic_uint "$after_f" || return 0
    [ $((after_f - before_f)) = "$HIC_COUNT" ] || return 0
    HIC_TIDS_RETURNED=$((after_r - before_r))
    HIC_UNRESOLVED=$((HIC_TIDS_RETURNED - HIC_COUNT))
}

hic_read_count() {
    # Check the one captured count row: same transaction, snapshot, backend,
    # role, and this attempt's planner profile. A reload can change the
    # damage-hiding settings between statements, so the row records them too;
    # a refused value stops the run (status 2), as it does at the start.
    local xid snapshot backend role seq idx only bitmap parallel rls path quote isolation readonly settings_hex
    local zero ignore track old
    hic_data "$HIC_A/count.raw" "SELECT /* wiki_hic12_capture_facts */ row_count,transaction_id,snapshot_text,backend_pid,
 'x'||encode(convert_to(checking_role,'UTF8'),'hex'),settings->>'seq',settings->>'index',settings->>'only',settings->>'bitmap',
 settings->>'parallel',settings->>'rls',settings->>'path',settings->>'quote',settings->>'isolation',settings->>'readonly',
 settings->>'zero_damaged_pages',settings->>'ignore_checksum_failure',settings->>'track_counts',
 settings->>'old_snapshot_threshold',captured,'x'||encode(convert_to(settings::text,'UTF8'),'hex')
FROM pg_temp.hic_count WHERE attempt_id=$HIC_ATTEMPT_ID;" || return
    [ "$(wc -l < "$HIC_A/count.raw")" -eq 1 ] || { hic_reject count_missing; return 0; }
    IFS='|' read -r HIC_COUNT xid snapshot backend role seq idx only bitmap parallel rls path quote \
        isolation readonly zero ignore track old HIC_CAPTURED settings_hex < "$HIC_A/count.raw" || return 1
    if ! hic_server_guard "$zero" "$ignore" "$track" "$old"; then
        HIC_STOP="$HIC_STOP (count of attempt $HIC_ATTEMPT_ID)"
        return 2
    fi
    hic_uint "$HIC_COUNT" || hic_reject count_type
    [ "$xid" = "$HIC_XID" ] && [ "$snapshot" = "$HIC_SNAPSHOT" ] && [ "$backend" = "$HIC_BACKEND" ] \
        && [ "$role" = "$HIC_ROLE_HEX" ] || hic_reject capture_context
    [ "$only" = off ] && [ "$bitmap" = off ] && [ "$parallel" = 0 ] && [ "$rls" = off ] \
        && [ "$path" = 'pg_catalog, pg_temp' ] && [ "$quote" = off ] \
        && [ "$isolation" = 'repeatable read' ] && [ "$readonly" = on ] || hic_reject capture_settings
    case $HIC_PHASE:$seq:$idx in heap:on:off|index:off:on) ;; *) hic_reject capture_profile ;; esac
}

hic_attempt_import() {
    # Import the saved plan into pg_temp.hic_plan as a dollar-quoted literal.
    local json="$HIC_OUT/plans/attempt-$HIC_ATTEMPT_ID.json" tag n=0
    [ -s "$json" ] || { HIC_ERR_MESSAGE='missing executed plan'; return 1; }
    while grep -F -q -e "\$hic12_plan$n\$" "$json"; do n=$((n + 1)); done
    tag="\$hic12_plan$n\$"
    {
        printf 'INSERT /* wiki_hic12_import_plan */ INTO pg_temp.hic_plan(attempt_id,document) VALUES (%s,%s' "$HIC_ATTEMPT_ID" "$tag"
        cat "$json"
        printf '%s::jsonb);\n' "$tag"
    } > "$HIC_A/import.sql" || return 2
    hic_request "$HIC_S" "$HIC_A/import.sql" || return 2
    [ "$HIC_FAILED" = false ]
}

hic_resolve_index() {
    # Map the plan's (quoted) Index Name to exactly one eligible index, then
    # require that name to resolve to the same index in the current catalog.
    # EXPLAIN prints an index's current name, while the inventory is the
    # catalog at the snapshot: a rename or swap committed after the snapshot
    # would otherwise credit coverage to the wrong index.
    local iid matches=0 nsp
    HIC_ACTUAL_INDEX=
    [ "$HIC_PHASE" = index ] || return 0
    while IFS= read -r iid; do
        hic_get "$HIC_OUT/indexes/$iid" eligible || return 2
        [ "$HIC_VALUE" = true ] || continue
        hic_read_index < "$HIC_OUT/indexes/$iid/facts" || return 1
        if [ "$i_ident" = "$HIC_ACTUAL_NAME" ]; then HIC_ACTUAL_INDEX=$iid matches=$((matches + 1)); fi
    done < "$HIC_OUT/relations/$HIC_RELATION/index-ids"
    [ "$matches" = 1 ] || { HIC_ACTUAL_INDEX=; hic_reject actual_index_not_uniquely_eligible; return 0; }
    hic_read_index < "$HIC_OUT/indexes/$HIC_ACTUAL_INDEX/facts" || return 1
    hic_decode "$i_schema" || return 1
    hic_literal "$HIC_TEXT"; nsp=$HIC_LITERAL
    hic_decode "$HIC_ACTUAL_NAME" || return 1
    hic_literal "$HIC_TEXT"
    hic_sql "$HIC_S" "SELECT /* wiki_hic12_executed_index */ COALESCE(to_regclass(quote_ident($nsp)||'.'||$HIC_LITERAL)::oid,0);" || return
    [ "$HIC_VALUE" = "$HIC_ACTUAL_INDEX" ] || hic_reject actual_index_identity_changed
}

hic_attempt_inspect() {
    hic_plan_facts "$HIC_ATTEMPT_ID" "$HIC_A/plan-facts" || return
    hic_validate "$HIC_A/plan-facts" || return
    hic_read_count || return
    hic_resolve_index || return
    hic_unresolved_tids || return
    HIC_ACCEPTED=false
    [ -n "$HIC_REASONS" ] || HIC_ACCEPTED=true
}

hic_recover() {
    hic_sql "$HIC_S" 'ROLLBACK /* wiki_hic12_recover */ TO SAVEPOINT hic_attempt;' || return
    hic_sql "$HIC_S" 'RELEASE /* wiki_hic12_release_error */ SAVEPOINT hic_attempt;' || return
    hic_profiles control
}

hic_attempt_error() {
    # Keep the runner's own message, else parse psql's report, else use $1.
    [ -n "$HIC_ERR_MESSAGE" ] || hic_parse_error "$HIC_S" || HIC_ERR_MESSAGE=$1
}

hic_attempt_run() {
    # Execute, save, import and inspect, setting hic_attempt_once's stage.
    # Returns 0 when inspected, 1 for an error the savepoint can recover
    # (HIC_ERR_* describe it), 2 when the run cannot continue.
    local rc s_dir s_in s_out s_pid s_seq
    HIC_ERR_CODE= HIC_ERR_MESSAGE= HIC_ERR_DETAIL= HIC_ERR_HINT=
    stage=execution
    if [ "$HIC_PHASE" = index ] && grep -E -q "$HIC_GUARD" "$HIC_OUT/indexes/$HIC_INTENDED/order"; then
        HIC_ERR_MESSAGE='runner did not send the count statement: psql interpolation guard'
        return 1
    fi
    hic_request "$HIC_S" "$HIC_A/count.sql" || return 2
    [ "$HIC_FAILED" = false ] || { hic_attempt_error 'count statement failed'; return 1; }
    hic_session "$HIC_S" || return 2
    # A warning while counting, such as a checksum failure the server was set
    # to ignore, means the count may rest on damaged pages: fail the attempt.
    if grep -E -q "$HIC_WARNING_LINE" "$s_dir/stderr.last"; then
        hic_parse_error "$HIC_S" "$HIC_WARNING_LINE" || HIC_ERR_MESSAGE='warning while counting'
        return 1
    fi

    stage=capture
    cp "$s_dir/response" "$HIC_OUT/plans/attempt-$HIC_ATTEMPT_ID.json" \
        || { HIC_ERR_MESSAGE='could not save executed plan'; return 1; }
    hic_profiles control || return 2
    hic_attempt_import; rc=$?
    [ "$rc" != 2 ] || return 2
    [ "$rc" = 0 ] || { hic_attempt_error 'capture or inspection failed'; return 1; }

    stage=inspection
    hic_attempt_inspect; rc=$?
    [ "$rc" != 2 ] || return 2
    [ "$rc" = 0 ] || { hic_attempt_error 'capture or inspection failed'; return 1; }
}

hic_attempt_once() {
    # Run one attempt inside savepoint hic_attempt and publish its record.
    # Failed and rejected attempts return 0; nonzero stops the run.
    local stage status=error count actual reasons execution_ms captured tids_returned unresolved
    local sqlstate= message= detail= hint=
    HIC_COUNT= HIC_ACTUAL_INDEX= HIC_REASONS= HIC_ACCEPTED=false HIC_EXECUTION_MS= HIC_CAPTURED=
    HIC_TIDS_RETURNED= HIC_UNRESOLVED=
    hic_prepare_attempt || return
    hic_attempt_run
    case $? in
        0)  # Inspected: releasing the savepoint keeps the count and plan rows.
            hic_sql "$HIC_S" 'RELEASE /* wiki_hic12_accept */ SAVEPOINT hic_attempt;' || return 2
            hic_profiles control || return 2
            status=rejected
            [ "$HIC_ACCEPTED" != true ] || status=accepted ;;
        1)  # Recoverable: keep the diagnostics, then roll the attempt back.
            sqlstate=$HIC_ERR_CODE message=$HIC_ERR_MESSAGE detail=$HIC_ERR_DETAIL hint=$HIC_ERR_HINT
            hic_recover || { HIC_STOP='attempt recovery failed'; return 2; }
            HIC_COUNT= HIC_ACTUAL_INDEX= HIC_ACCEPTED=false HIC_TIDS_RETURNED= HIC_UNRESOLVED= ;;
        *)  return 2 ;;
    esac
    # Evidence is published only after the savepoint was released or rolled back.
    count=$HIC_COUNT actual=$HIC_ACTUAL_INDEX reasons=$HIC_REASONS
    execution_ms=$HIC_EXECUTION_MS captured=$HIC_CAPTURED
    tids_returned=$HIC_TIDS_RETURNED unresolved=$HIC_UNRESOLVED
    hic_save "$HIC_A" status count actual reasons execution_ms captured stage sqlstate message detail hint \
        tids_returned unresolved || return 2
    hic_say "attempt=$HIC_ATTEMPT_ID phase=$HIC_PHASE relation=$HIC_RELATION intended=$HIC_INTENDED actual=${HIC_ACTUAL_INDEX:--} status=$status"
}

hic_identity() {
    # hic_identity OID: after the accepted heap attempt, the planner's locks on
    # the table and every index last until COMMIT. Compare each one's
    # snapshot-time name and relfilenode with the current catalog. TRUNCATE, a
    # rewrite, a refresh, a drop and re-create or an index rename committed
    # after the snapshot changes one of them; save that as drift.
    local oid=$1 dir="$HIC_OUT/relations/$1" rows iid nsp id now_oid now_node want drift=
    hic_read_relation < "$dir/facts" || return 2
    hic_decode "$r_qualified" || return 1
    hic_literal "$HIC_TEXT"
    rows="($oid::oid,$HIC_LITERAL::text,$r_filenode::oid)"
    while IFS= read -r iid; do
        hic_read_index < "$HIC_OUT/indexes/$iid/facts" || return 2
        hic_decode "$i_schema" || return 1
        hic_literal "$HIC_TEXT"; nsp=$HIC_LITERAL
        hic_decode "$i_name" || return 1
        hic_literal "$HIC_TEXT"
        rows="$rows,($iid::oid,format('%I.%I',$nsp,$HIC_LITERAL),$i_filenode::oid)"
    done < "$dir/index-ids"
    hic_data "$dir/identity.raw" "SELECT /* wiki_hic12_identity */ x.id,COALESCE(to_regclass(x.qualified)::oid,0),
 COALESCE(pg_relation_filenode(x.id),0),x.filenode FROM (VALUES $rows) AS x(id,qualified,filenode) ORDER BY x.id;" || return
    while IFS='|' read -r id now_oid now_node want; do
        [ "$now_oid" = "$id" ] || drift="${drift:+$drift,}name_$id"
        [ "$now_node" = "$want" ] || drift="${drift:+$drift,}storage_$id"
    done < "$dir/identity.raw"
    [ -s "$dir/identity.raw" ] || drift=identity_unreadable
    hic_put "$dir" drift "$drift"
}

# ---- verdicts: derived from published attempts only ---------------------------

hic_matched() { case $1 in PASS|PASS_WITH_ANOMALY) return 0 ;; esac; return 1; }

hic_evaluate() {
    # hic_evaluate OID: derive the relation's status from published attempts,
    # save it with its counts, and set HIC_STATUS, HIC_HEAP and HIC_DRIFT.
    # Every evidence read is checked, so a lost file stops the run instead of
    # changing a verdict.
    local oid=$1 dir="$HIC_OUT/relations/$1" iid aid verdict
    local scope permission drift phase status count actual intended sqlstate reasons unresolved
    local heap= conflicting=false eligible=0 checked_count matching_count anomalies
    local mismatch_count=0 error_count=0 storage_errors=0 uncovered=0 match= unresolved_index_tids=
    : > "$dir/checked" && : > "$dir/matching" || return 2
    hic_load "$dir" scope permission drift || return 2
    hic_get "$dir" index-ids || return 2
    while IFS= read -r iid; do
        [ -n "$iid" ] || continue
        hic_get "$HIC_OUT/indexes/$iid" eligible || return 2
        [ "$HIC_VALUE" != true ] || eligible=$((eligible + 1))
    done <<< "$HIC_VALUE"
    # Pass 1: the accepted heap baseline. Two different accepted counts conflict.
    hic_get "$dir" attempt-ids || return 2
    while IFS= read -r aid; do
        [ -n "$aid" ] || continue
        hic_load "$HIC_OUT/attempts/$aid" phase status count || return 2
        [ "$phase:$status" = heap:accepted ] || continue
        hic_uint "$count" || return 2
        if [ -n "$heap" ] && [ "$heap" != "$count" ]; then conflicting=true; fi
        heap=$count
    done <<< "$HIC_VALUE"
    # Pass 2: accepted index counts, credited to the index that actually ran.
    hic_get "$dir" attempt-ids || return 2
    while IFS= read -r aid; do
        [ -n "$aid" ] || continue
        hic_load "$HIC_OUT/attempts/$aid" phase status count actual reasons unresolved || return 2
        if [ "$status" = error ]; then error_count=$((error_count + 1)); fi
        case ,$reasons, in *,actual_index_identity_changed,*) drift=${drift:-index_identity_changed} ;; esac
        [ "$phase:$status" = index:accepted ] || continue
        hic_uint "$actual" && hic_uint "$count" || return 2
        hic_add_line "$dir/checked" "$actual" || return 2
        if hic_uint "$unresolved" && { [ -z "$unresolved_index_tids" ] || [ "$unresolved" -gt "$unresolved_index_tids" ]; }; then
            unresolved_index_tids=$unresolved
        fi
        [ -n "$heap" ] || continue
        if [ "$count" = "$heap" ]; then
            hic_add_line "$dir/matching" "$actual" || return 2
            [ -n "$match" ] || match=$actual
        else
            mismatch_count=$((mismatch_count + 1))
        fi
    done <<< "$HIC_VALUE"
    # Pass 3: errors not replaced by an accepted heap count or intended index.
    # A storage-class error (XX, 58, or a warning while counting) is never
    # replaced: a retry can succeed on a page the failed read zeroed.
    hic_get "$dir" attempt-ids || return 2
    while IFS= read -r aid; do
        [ -n "$aid" ] || continue
        hic_load "$HIC_OUT/attempts/$aid" status phase intended sqlstate || return 2
        [ "$status" = error ] || continue
        if [[ $sqlstate =~ $HIC_STORAGE_CODE ]]; then
            storage_errors=$((storage_errors + 1)) uncovered=$((uncovered + 1))
        elif [ "$phase" = heap ]; then
            [ -n "$heap" ] || uncovered=$((uncovered + 1))
        else
            hic_contains "$dir" checked "$intended" || return 2
            [ "$HIC_FOUND" = true ] || uncovered=$((uncovered + 1))
        fi
    done <<< "$HIC_VALUE"
    anomalies=$((mismatch_count + storage_errors))
    checked_count=$(wc -l < "$dir/checked") || return 2
    matching_count=$(wc -l < "$dir/matching") || return 2
    checked_count=$((checked_count))                  # drop BSD wc's padding
    matching_count=$((matching_count))
    if [ "$scope" != true ]; then verdict=NOT_APPLICABLE
    elif [ "${HIC_EVIDENCE_VALID:-true}" != true ] || [ -n "$permission" ] || [ "$conflicting" = true ] \
        || [ -n "$drift" ]; then verdict=ERROR
    elif [ "$eligible" -eq 0 ]; then verdict=NO_ELIGIBLE_INDEX
    elif [ -n "$heap" ] && [ "$matching_count" -gt 0 ] && [ "$anomalies" -gt 0 ]; then verdict=PASS_WITH_ANOMALY
    elif [ -n "$heap" ] && [ "$matching_count" -gt 0 ]; then verdict=PASS
    elif [ "$uncovered" -gt 0 ]; then verdict=ERROR
    elif [ -n "$heap" ] && [ "$checked_count" -eq "$eligible" ]; then verdict=MISMATCH
    else verdict=INCONCLUSIVE
    fi
    hic_put "$dir" status "$verdict" || return 2
    hic_save "$dir" heap eligible checked_count matching_count mismatch_count error_count storage_errors \
        anomalies unresolved_index_tids match drift || return 2
    HIC_STATUS=$verdict HIC_HEAP=$heap HIC_DRIFT=$drift
}

hic_attempt_loop() {
    # Per relation: up to two heap attempts, an identity check, then up to two
    # attempts per index until one matches (or every index is tried, in
    # exhaustive mode).
    local oid dir iid scope permission eligible
    hic_get "$HIC_OUT" relation-ids || return 2
    while IFS= read -r oid; do
        [ -n "$oid" ] || continue
        dir="$HIC_OUT/relations/$oid"
        hic_evaluate "$oid" || return
        hic_load "$dir" scope permission eligible || return 2
        [ "$scope" = true ] && [ -z "$permission" ] && [ "$eligible" -gt 0 ] || continue
        hic_read_relation < "$dir/facts" || return 2
        HIC_RELATION=$oid HIC_REL_SCHEMA=$r_schema HIC_REL_NAME=$r_name HIC_REL_QUALIFIED=$r_qualified
        HIC_PHASE=heap HIC_INTENDED=
        for HIC_VARIANT in 1 2; do
            hic_attempt_once || return
            hic_evaluate "$oid" || return
            [ -z "$HIC_HEAP" ] || break
        done
        [ -n "$HIC_HEAP" ] || continue
        hic_identity "$oid" || return
        hic_evaluate "$oid" || return
        [ -z "$HIC_DRIFT" ] || continue
        HIC_PHASE=index
        hic_get "$dir" index-ids || return 2
        while IFS= read -r iid; do
            [ -n "$iid" ] || continue
            hic_get "$HIC_OUT/indexes/$iid" eligible || return 2
            [ "$HIC_VALUE" = true ] || continue
            hic_get "$HIC_OUT/indexes/$iid" generation || return 2
            [ -z "$HIC_VALUE" ] || continue
            for HIC_VARIANT in 1 2; do
                hic_evaluate "$oid" || return
                if hic_matched "$HIC_STATUS" && [ "$HIC_EXHAUSTIVE_BOOL" = false ]; then break; fi
                [ -z "$HIC_DRIFT" ] || break
                # An attempt intended for another index may already have used this one.
                hic_contains "$dir" checked "$iid" || return 2
                if [ "$HIC_FOUND" = true ]; then break; fi
                HIC_INTENDED=$iid
                hic_attempt_once || return
            done
        done <<< "$HIC_VALUE"
        hic_evaluate "$oid" || return
    done <<< "$HIC_VALUE"
}

# ---- reports: CSV escaping and every aggregation rule live in Bash ------------

hic_report_relation() {
    # One relations.csv row; adds to hic_reports' totals (dynamic scope).
    local oid=$1 dir="$HIC_OUT/relations/$1" schema_name relation_name untested
    local scope reason permission status heap eligible checked_count matching_count mismatch_count error_count match
    local storage_errors anomalies unresolved_index_tids drift
    hic_evaluate "$oid" || return 2
    hic_load "$dir" scope reason permission status heap eligible checked_count matching_count \
        mismatch_count error_count match storage_errors anomalies unresolved_index_tids drift || return 2
    hic_read_relation < "$dir/facts" || return 2
    hic_decode "$r_schema" || return 2; schema_name=$HIC_TEXT
    hic_decode "$r_name" || return 2; relation_name=$HIC_TEXT
    untested=$((eligible - checked_count))
    hic_csv "$HIC_OUT/relations.csv" "$oid" "$schema_name" "$relation_name" "$r_kind" "$scope" "$reason" \
        "$permission" "$status" "$heap" "$eligible" "$checked_count" "$matching_count" "$untested" \
        "$mismatch_count" "$error_count" "$match" "$anomalies" "$storage_errors" "$unresolved_index_tids" \
        "$drift" || return 2
    case $status in
        PASS) passed=$((passed + 1)) ;;
        PASS_WITH_ANOMALY) anomalous=$((anomalous + 1)) ;;
        MISMATCH) mismatched=$((mismatched + 1)) ;;
        INCONCLUSIVE) inconclusive=$((inconclusive + 1)) ;;
        NO_ELIGIBLE_INDEX) no_index=$((no_index + 1)) ;;
        ERROR) errored=$((errored + 1)) ;;
        NOT_APPLICABLE) excluded=$((excluded + 1)) ;;
    esac
    case $r_kind in
        r|m) hic_heap_am "$r_am" "$r_amname" "$r_amtype" || unsupported=$((unsupported + 1)) ;;
    esac
    if [ "$scope" = true ]; then
        physical=$((physical + 1))
        total_eligible=$((total_eligible + eligible))
        total_checked=$((total_checked + checked_count))
        total_matching=$((total_matching + matching_count))
        total_untested=$((total_untested + untested))
        [ "$checked_count" -eq "$eligible" ] || exhaustive=false
        # An accepted index count without a usable TID signal is counted apart,
        # never added to the total as zero.
        if [ -n "$unresolved_index_tids" ]; then
            total_unresolved=$((total_unresolved + unresolved_index_tids))
        elif [ "$checked_count" -gt 0 ]; then
            unresolved_unavailable=$((unresolved_unavailable + 1))
        fi
    fi
    total_mismatches=$((total_mismatches + mismatch_count))
    total_errors=$((total_errors + error_count))
    total_anomalies=$((total_anomalies + anomalies))
}

hic_report_candidates() {
    # One candidates.csv row per index: was it actually checked, and if not, why.
    local oid=$1 dir="$HIC_OUT/relations/$1" iid idir aid status drift attempted actual untested usability
    local eligible reason generation schema_name index_name
    hic_load "$dir" status drift || return 2
    hic_get "$dir" index-ids || return 2
    while IFS= read -r iid; do
        [ -n "$iid" ] || continue
        idir="$HIC_OUT/indexes/$iid"
        hic_read_index < "$idir/facts" || return 2
        hic_load "$idir" eligible reason generation || return 2
        attempted=0
        hic_get "$dir" attempt-ids || return 2
        while IFS= read -r aid; do
            [ -n "$aid" ] || continue
            hic_get "$HIC_OUT/attempts/$aid" intended || return 2
            [ "$HIC_VALUE" != "$iid" ] || attempted=$((attempted + 1))
        done <<< "$HIC_VALUE"
        actual=false usability=not_established
        hic_contains "$dir" checked "$iid" || return 2
        if [ "$HIC_FOUND" = true ]; then actual=true usability=accepted_by_executed_plan
        elif [ "$eligible" = true ] && [ "$i_checkxmin" = t ]; then usability=indcheckxmin_unresolved
        fi
        if [ "$eligible" != true ]; then untested=$reason
        elif [ "$actual" = true ]; then untested=
        elif [ -n "$generation" ]; then untested=$generation
        elif [ -n "$drift" ]; then untested=relation_identity_changed
        elif hic_matched "$status" && [ "$HIC_EXHAUSTIVE_BOOL" = false ]; then untested=stopped_after_first_match
        elif [ "$attempted" -ge 2 ]; then untested=attempt_budget_exhausted_without_actual_coverage
        else untested=not_yet_checked
        fi
        hic_decode "$i_schema" || return 2; schema_name=$HIC_TEXT
        hic_decode "$i_name" || return 2; index_name=$HIC_TEXT
        hic_csv "$HIC_OUT/candidates.csv" "$iid" "$oid" "$schema_name" "$index_name" "$eligible" "$reason" \
            "$generation" "$i_checkxmin" "$i_xmin" "$actual" "$attempted" "$untested" "$usability" || return 2
    done <<< "$HIC_VALUE"
}

hic_report_attempts() {
    # One attempts.csv row per attempt; delta = index count - heap count.
    local oid=$1 dir="$HIC_OUT/relations/$1" aid heap delta
    local phase intended actual variant status count reasons stage sqlstate message detail hint
    local execution_ms started captured tids_returned unresolved
    hic_get "$dir" heap || return 2; heap=$HIC_VALUE
    hic_get "$dir" attempt-ids || return 2
    while IFS= read -r aid; do
        [ -n "$aid" ] || continue
        hic_load "$HIC_OUT/attempts/$aid" phase intended actual variant status count reasons stage \
            sqlstate message detail hint execution_ms started captured tids_returned unresolved || return 2
        delta=
        if [ "$phase" = index ] && [ "$status" = accepted ] && [ -n "$heap" ]; then delta=$((count - heap)); fi
        hic_csv "$HIC_OUT/attempts.csv" "$aid" "$oid" "$phase" "$intended" "$actual" "$variant" "$status" \
            "$count" "$delta" "$reasons" "$stage" "$sqlstate" "$message" "$detail" "$hint" \
            "plans/attempt-$aid.json" "$execution_ms" "$started" "$captured" "$tids_returned" "$unresolved" \
            || return 2
    done <<< "$HIC_VALUE"
}

hic_report_parent() {
    # For a partitioned table, one parents.csv row: are all descendants in
    # scope and supported, and did every leaf pass? Adds to hic_reports'
    # count of incomplete subtrees (dynamic scope).
    local oid=$1 parent child kind ns am schema_hex name_hex
    local descendants=0 outside=0 leaves=0 all_in_scope=true all_supported=true subtree=true
    hic_read_relation < "$HIC_OUT/relations/$oid/facts" || return 2
    [ "$r_kind" = p ] || return 0
    hic_get "$HIC_OUT" partitions.raw || return 2
    while IFS='|' read -r parent child kind ns am schema_hex name_hex; do
        [ "$parent" = "$oid" ] || continue
        descendants=$((descendants + 1))
        [ "$ns" = "$HIC_SCHEMA_OID" ] || outside=$((outside + 1))
        if [ ! -d "$HIC_OUT/relations/$child" ]; then all_in_scope=false subtree=false; fi
        if [ "$kind" = p ]; then continue; fi
        leaves=$((leaves + 1))
        if [ "$kind" != r ] || [ "$am" != 2 ]; then all_supported=false subtree=false; fi
        if [ -d "$HIC_OUT/relations/$child" ]; then
            hic_evaluate "$child" || return 2
            [ "$HIC_STATUS" = PASS ] || subtree=false
        fi
    done <<< "$HIC_VALUE"
    if [ "$descendants" = 0 ]; then all_in_scope=false all_supported=false; fi
    [ "$leaves" -gt 0 ] || subtree=false
    [ "$subtree" = true ] || incomplete_subtrees=$((incomplete_subtrees + 1))
    hic_csv "$HIC_OUT/parents.csv" "$oid" "$descendants" "$outside" "$all_in_scope" "$all_supported" \
        "$leaves" "$subtree"
}

hic_reports() {
    # Write the five CSV reports from published evidence; set HIC_SUMMARY.
    local oid report
    local physical=0 passed=0 anomalous=0 mismatched=0 inconclusive=0 no_index=0 errored=0 excluded=0 unsupported=0
    local total_eligible=0 total_checked=0 total_matching=0 total_untested=0 total_mismatches=0 total_errors=0
    local total_anomalies=0 total_unresolved=0 unresolved_unavailable=0 incomplete_subtrees=0
    local exhaustive=true all_match=false
    for report in relations candidates attempts parents schema; do
        : > "$HIC_OUT/$report.csv" || return 2
    done
    hic_csv "$HIC_OUT/relations.csv" relation_oid schema_name relation_name relkind in_heap_scope exclusion_reason permission_error status heap_count eligible_indexes checked_indexes matching_indexes untested_indexes mismatching_attempts error_attempts matching_index_oid anomalies storage_error_attempts unresolved_index_tids identity_drift || return 2
    hic_csv "$HIC_OUT/candidates.csv" index_oid relation_oid schema_name index_name eligible exclusion_reason generation_reason indcheckxmin catalog_xmin actually_checked intended_attempts untested_reason snapshot_usability || return 2
    hic_csv "$HIC_OUT/attempts.csv" attempt_id relation_oid phase intended_index_oid actual_index_oid variant status row_count delta rejection_reasons error_stage sqlstate message detail hint plan_file execution_ms started captured index_tids_returned unresolved_index_tids || return 2
    hic_csv "$HIC_OUT/parents.csv" relation_oid descendants outside_schema_descendants all_descendants_in_scope all_descendants_supported physical_children subtree_success || return 2
    hic_get "$HIC_OUT" relation-ids || return 2
    while IFS= read -r oid; do
        [ -n "$oid" ] || continue
        hic_report_relation "$oid" || return 2
        hic_report_candidates "$oid" || return 2
        hic_report_attempts "$oid" || return 2
        hic_report_parent "$oid" || return 2
    done <<< "$HIC_VALUE"
    HIC_SCHEMA_SUCCESS=false HIC_HEAP_ALL_PASS=false
    if [ "$physical" -gt 0 ] && [ "$physical" -eq "$passed" ] && [ "$HIC_EVIDENCE_VALID" = true ]; then
        HIC_HEAP_ALL_PASS=true
        [ "$unsupported" -ne 0 ] || HIC_SCHEMA_SUCCESS=true
    fi
    [ "$physical" -gt 0 ] || exhaustive=false
    if [ "$total_checked" -gt 0 ] && [ "$total_mismatches" -eq 0 ]; then all_match=true; fi
    hic_csv "$HIC_OUT/schema.csv" schema_success heap_scope_all_pass physical_relations passed_relations pass_with_anomaly_relations mismatched_relations inconclusive_relations no_index_relations error_relations excluded_objects unsupported_table_ams eligible_indexes checked_indexes matching_indexes untested_indexes exhaustive_coverage_complete all_tested_indexes_match observed_index_mismatches attempt_errors anomalies unresolved_index_tids incomplete_partition_subtrees unresolved_tids_unavailable_relations || return 2
    hic_csv "$HIC_OUT/schema.csv" "$HIC_SCHEMA_SUCCESS" "$HIC_HEAP_ALL_PASS" "$physical" "$passed" "$anomalous" \
        "$mismatched" "$inconclusive" "$no_index" "$errored" "$excluded" "$unsupported" "$total_eligible" \
        "$total_checked" "$total_matching" "$total_untested" "$exhaustive" "$all_match" "$total_mismatches" \
        "$total_errors" "$total_anomalies" "$total_unresolved" "$incomplete_subtrees" "$unresolved_unavailable" || return 2
    HIC_SUMMARY="schema_success=$HIC_SCHEMA_SUCCESS physical_relations=$physical passed=$passed pass_with_anomaly=$anomalous mismatched=$mismatched inconclusive=$inconclusive no_eligible_index=$no_index error=$errored eligible_indexes=$total_eligible checked_indexes=$total_checked untested_indexes=$total_untested observed_index_mismatches=$total_mismatches attempt_errors=$total_errors anomalies=$total_anomalies unresolved_index_tids=$total_unresolved unresolved_tids_unavailable=$unresolved_unavailable incomplete_partition_subtrees=$incomplete_subtrees"
}

hic_finish() {
    # Check the context is unchanged, COMMIT, then write the reports.
    local expected="$HIC_BACKEND|$HIC_XID|$HIC_SNAPSHOT|$HIC_ROLE_HEX|repeatable read|on"
    hic_sql "$HIC_S" "SELECT /* wiki_hic12_final_context */ pg_backend_pid(),txid_current(),txid_current_snapshot()::text,
 'x'||encode(convert_to(current_user::text,'UTF8'),'hex'),current_setting('transaction_isolation'),current_setting('transaction_read_only');" || return 2
    [ "$HIC_VALUE" = "$expected" ] || { HIC_EVIDENCE_VALID=false HIC_STOP='checking context changed'; return 2; }
    hic_sql "$HIC_S" 'COMMIT /* wiki_hic12_commit */;' || return 2
    hic_reports || return 2
    hic_decode "$HIC_ROLE_HEX" || return 2
    : > "$HIC_OUT/run.csv" || return 2
    hic_csv "$HIC_OUT/run.csv" schema_name schema_oid database_oid version_num backend_pid transaction_id snapshot checking_role session_role_hex started finished evidence_valid ddl_stable include_external_partitions exhaustive settings_file || return 2
    hic_csv "$HIC_OUT/run.csv" "$HIC_SCHEMA" "$HIC_SCHEMA_OID" "$HIC_DATABASE_OID" "$HIC_SERVER" "$HIC_BACKEND" \
        "$HIC_XID" "$HIC_SNAPSHOT" "$HIC_TEXT" "$HIC_SESSION_ROLE" "$HIC_STARTED" "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
        "$HIC_EVIDENCE_VALID" true "$HIC_EXTERNAL" "$HIC_EXHAUSTIVE_BOOL" settings.json || return 2
}

# ---- driver ------------------------------------------------------------------

hic_initialize() {
    # Set up, open the checking transaction and take the inventory.
    HIC_STAGE=initialization
    HIC_ATTEMPT_ID=0 HIC_EVIDENCE_VALID=true HIC_STOP= HIC_SUMMARY= HIC_SCHEMA_SUCCESS=false
    mkdir -p "$HIC_OUT/relations" "$HIC_OUT/indexes" "$HIC_OUT/attempts" "$HIC_OUT/plans" || return 2
    : > "$HIC_OUT/run-errors.csv" || return 2
    hic_csv "$HIC_OUT/run-errors.csv" stage sqlstate message || return 2
    hic_install || return
    hic_start || return
    HIC_STAGE=inventory
    hic_inventory
}

hic_check() {
    # hic_check SCHEMA INCLUDE_EXTERNAL EXHAUSTIVE (true or false): returns 0
    # for schema success, 1 for a completed run without it, 2 when incomplete.
    HIC_SCHEMA=$1 HIC_EXTERNAL=$2 HIC_EXHAUSTIVE_BOOL=$3
    if ! hic_initialize; then
        hic_parse_error "$HIC_S" || :
        hic_run_error "$HIC_STAGE" "${HIC_ERR_CODE:-}" \
            "${HIC_STOP:-${HIC_ERR_MESSAGE:-initialization or inventory failed}}" || :
        hic_sql "$HIC_S" 'ROLLBACK /* wiki_hic12_abandon_start */;' || :
        return 2
    fi
    if ! hic_attempt_loop; then
        HIC_EVIDENCE_VALID=false HIC_STOP=${HIC_STOP:-attempt loop interrupted}
        hic_run_error snapshot "${HIC_SQLSTATE:-}" "$HIC_STOP" || :
        hic_sql "$HIC_S" 'ROLLBACK /* wiki_hic12_abandon */;' || :
        return 2
    fi
    hic_finish || { HIC_STOP=${HIC_STOP:-report or commit failed}; return 2; }
    [ "$HIC_SCHEMA_SUCCESS" = true ]
}

hic_main() {
    local external exhaustive rc completed=false
    set -u -o pipefail
    umask 077
    case ${1:-} in
        -h|--help) sed -n '/^#/!q;p' "${BASH_SOURCE[0]}"; return 0 ;;   # the header comment
        '') ;;
        *) hic_warn "unexpected argument: $1"; return 64 ;;
    esac
    [ -n "${HIC_SCHEMA:-}" ] || { hic_warn 'set HIC_SCHEMA'; return 64; }
    [ "${HIC_DDL_STABLE:-}" = yes ] || { hic_warn 'set HIC_DDL_STABLE=yes'; return 64; }
    case ${HIC_INCLUDE_EXTERNAL_PARTITIONS:-no} in
        yes) external=true ;; no) external=false ;;
        *) hic_warn 'HIC_INCLUDE_EXTERNAL_PARTITIONS must be yes or no'; return 64 ;;
    esac
    case ${HIC_EXHAUSTIVE:-no} in
        yes) exhaustive=true ;; no) exhaustive=false ;;
        *) hic_warn 'HIC_EXHAUSTIVE must be yes or no'; return 64 ;;
    esac
    : "${HIC_OUT:=hic12-run-$(date -u +%Y%m%dT%H%M%SZ)-$$}"
    mkdir -m 700 "$HIC_OUT" || return 64
    export PGAPPNAME="${PGAPPNAME:-hic12-check}"
    trap hic_cleanup EXIT
    trap 'exit 130' INT
    trap 'exit 143' TERM
    HIC_STOP= HIC_SUMMARY=
    if ! hic_open main 7 8; then
        HIC_STOP='psql session did not start' rc=2
    else
        if [ -n "${HIC_ROLE:-}" ]; then
            hic_literal "$HIC_ROLE"
            hic_sql main "SET /* wiki_hic12_role */ ROLE $HIC_LITERAL;" || HIC_STOP='SET ROLE failed'
        fi
        rc=2
        if [ -z "$HIC_STOP" ]; then hic_check "$HIC_SCHEMA" "$external" "$exhaustive"; rc=$?; fi
        hic_close main || rc=2
    fi
    trap - EXIT
    case $rc in 0|1) completed=true ;; *) HIC_SUMMARY= ;; esac
    {
        printf 'exit_status=%s\ncompleted=%s\nstop_reason=%s\n' "$rc" "$completed" "${HIC_STOP:-}"
        printf '%s\n' "${HIC_SUMMARY:-}" | tr ' ' '\n'
    } > "$HIC_OUT/summary.txt" || rc=2
    hic_say "exit $rc ${HIC_SUMMARY:-${HIC_STOP:-}} (reports in $HIC_OUT)"
    return "$rc"
}

if [ "${BASH_SOURCE[0]}" = "$0" ]; then hic_main "$@"; exit; fi
```

## Measurement Script

This is the page's single reproduction harness. It extracts the self-contained runner from this page, builds the pinned server out of tree and runs disposable Bash/SQL fixtures and assertions. It does not install a parallel checker implementation.

From the repository root, extract and run it:

```text
fence=$(printf '\140\140\140')
page=wiki/v12/questions/indexing/sql-heap-index-visible-count-checker-plan.md
mkdir -p .wiki-runtime/tmp
sed -n '/^## Measurement Script$/,/^## Context Reviewed$/p' "$page" | sed -n "/^${fence}bash$/,/^${fence}$/p" | sed '1d;$d' > .wiki-runtime/hic12-measure.sh
bash .wiki-runtime/hic12-measure.sh
```

| Item | Contract |
|---|---|
| Purpose | Reproduce the current [measured coverage](#measured-coverage-and-remaining-test-gaps), including Bash validation/evaluation, database fixtures and client failures |
| Default stages | `prepare build init verify cleanup`; stages are selectable, e.g. `bash .wiki-runtime/hic12-measure.sh verify` for a retained build |
| `prepare` | Confirm exact pin and unchanged tracked raw source; refuse a running server this invocation did not start; materialize runner/fixtures/tests under the owned sandbox |
| `build` | Prepare and stop a server started by this invocation if necessary; configure/build/install out of tree, reusing the build if present |
| `init` | Initialize UTF-8/C-locale disposable data if absent; start the owned socket-only server and confirm its data directory |
| `verify` | Record platform/configuration, create a unique disposable database, and run the whole suite |
| `cleanup` | Fast-stop the owned server, confirm PID/process/socket/port teardown, then delete the sandbox including build/results |
| Environment | `JOBS`, default 2; normal compiler/PATH lookup. The harness fixes LC_ALL=C, PGCLIENTENCODING=UTF8, TMPDIR/PGPASSFILE under the sandbox; unsets host-address/service/PGOPTIONS overrides; test sessions use the private socket and roles. Test names use UTC time and PID |
| Prerequisites | Bash with `/dev/tcp`, Git, POSIX utilities including `ps`, `paste`, `cmp`, `mkfifo`, `dd`, `tr` and `sleep`; GNU make, a C compiler that accepts `-std=gnu99`, Bison/Flex and Perl required by a Git-source build. Run it from the repository root; the extraction command and `prepare` create `.wiki-runtime/tmp/` when it is missing. No network or extra extension. ([installation.sgml#build-tools](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L80-L97), [installation.sgml#git-checkout-tools](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L300-L318)) |
| Build | `--without-readline --without-zlib`, `CFLAGS='-O2 -std=gnu99'` and a private install prefix. The two `--without` options omit optional interactive editing/compression facilities, not visibility/planning behavior ([installation.sgml#optional-libraries](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L118-L151)). Without `CFLAGS`, configure uses `-O2` and the compiler's default language mode ([configure.in#default-cflags](../../../../raw/postgres-12/configure.in#L408-L415)). With GCC 15.2 that build stopped at `c.h:308`, `typedef unsigned char bool`, with "'bool' cannot be defined via 'typedef'" and GCC's note that `bool` is a keyword from `-std=c23` on ([c.h#bool-typedef](../../../../raw/postgres-12/src/include/c.h#L302-L309)). PostgreSQL 12 asks for a C99 compiler ([installation.sgml#C99](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L93-L94)), so the flag selects that mode and keeps `-O2` |
| Output | `.wiki-runtime/tmp/hic12/suite-current.log`, `platform.txt`, `platform-sql.txt`, `build-options.txt`, then `results/<unique-db>/<case>/`; `last-test-output` names the latest result directory |
| Runtime | Fresh build with `JOBS=16`: 34s; verification: 229s right after that build and 228s with the build retained, including the `pg_catalog` run and the physical leg's stop and restart. This fresh build plus verification took about 4½ minutes; retained-build verification took about 4 minutes. These correctness-fixture timings do not predict production scan cost |
| Teardown | Each invocation stops only a server it started. A persistent sandbox marker identifies retained artifacts, not startup ownership. The `verify` stage stops and restarts its own server once, for the physical leg. A pre-existing running server is refused and left running, including on an early error. Omitting cleanup retains stopped build/results; the cleanup stage deletes them. Save any desired results before cleanup. The mid-run leg resets `zero_damaged_pages` with `ALTER SYSTEM RESET` and a reload right after its loop |

**Last run:** 2026-10-24, after the third review's fixes. Unchanged PostgreSQL 12.2 pin `45b88269a353ad93744772791feb6d01bc7e1e42`, built with GCC 15.2.0 (`CFLAGS='-O2 -std=gnu99'`) on Linux 6.18.54.1-microsoft-standard-WSL2 x86_64, GNU Bash 5.3.9 and pinned psql. Block size 8192, maximum alignment 8, UTF-8, C collation/ctype/lc_messages, server TimeZone America/Toronto, checksums disabled. From an absent sandbox, `prepare build` with `JOBS=16` (build 34s) and then `init verify` (19:38:47 to 19:42:36 UTC, 229s) passed all 263 assertions. A second `verify` with the build retained (19:42:43 to 19:46:31 UTC, 228s) passed all 263 again, and `cleanup` (under 1s) confirmed PID/process/socket/port teardown and deleted the sandbox including its build and results. In both runs, the zeroed block and the cached block each held 58 of 300 rows of a 6-block heap. The runner and harness blocks on this page were byte-identical to the files that ran. Earlier dated measurements, including the Darwin runs, remain historical.

Non-default behavior is intentional. The [settings table](#settings-and-query-shapes) gives the checker profiles, their defaults, mechanisms and transaction/session scope. These additional fixture changes supply specific test conditions:

| Setting/fixture | Default → fixture, scope | Mechanism/result affected |
|---|---|---|
| Client psql | ON_ERROR_STOP off retained; default verbosity → verbose; runner uses quiet/unaligned/tuples-only stdin | Bash must recover after errors and inspect SQLSTATE. Harness `psql -f/-c` calls use ON_ERROR_STOP=1 so fixture failures abort ([psql-ref.sgml#ON_ERROR_STOP](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3844-L3861)) |
| Fixture timeouts | Both 0 → statement 60s, lock 2s; user/session scope | Bounds fixture work/waits; runner counts retain their 15-minute statement budget ([guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396)) |
| Caller identifier quoting | Default off → on through PGOPTIONS for sourced cases; user/session scope | Exercises capture profiles resetting quote_all_identifiers off before setting search_path; restores caller setting after COMMIT ([guc.c#quote-all-identifiers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1816-L1824), [libpq.sgml#PGOPTIONS](../../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L7410-L7419)) |
| Server listeners | Default localhost TCP/port 5432 → no TCP and private socket/port 55447, mode 0700; postmaster settings, new test server only | Isolates fixtures; does not change counts ([guc.c#listener-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3935-L3960), [guc.c#port-and-permissions](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2176-L2199), [configure.in#default-port](../../../../raw/postgres-12/configure.in#L149-L162)) |
| Client encoding | Database encoding by default → explicit UTF8 through PGCLIENTENCODING; user/connection scope | Matches UTF-8 fixture/database text ([guc.c#client-encoding](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3581-L3589), [postinit.c#encoding-default](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L394-L401)) |
| Heap fillfactor | Default 100 → 50 on concurrent/horizon tables; CREATE TABLE storage parameter | Reserves same-page space; HOT/non-HOT writer assertions and late-index horizon fixture depend on this ([rel.h#heap-fillfactor](../../../../raw/postgres-12/src/include/utils/rel.h#L278-L279), [hio.c#reserved-space](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L348-L350), [heapam.c#HOT-decision](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3591-L3601)) |
| Old horizon and late index build | New indexes normally start indcheckxmin false → retained old transaction and broken-HOT-chain nonconcurrent build | Sets the flag; planner skips until catalog xmin precedes TransactionXmin. New checking transaction passes after holder ends; flag need not clear. old_snapshot_threshold stays default -1 ([index.c#initial-flag](../../../../raw/postgres-12/src/backend/catalog/index.c#L609-L615), [index.c#broken-chain-gate](../../../../raw/postgres-12/src/backend/catalog/index.c#L2921-L2971), [plancat.c#horizon](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234)) |
| Forced RLS | New relation flags false → ENABLE/FORCE policy and owner role | row_security off errors for owner; superuser bypass passes ([heap.c#security-flags](../../../../raw/postgres-12/src/backend/catalog/heap.c#L916-L919), [rls.c#forced-owner](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132)) |
| Multiline key text | Ordinary column names/literals → explicitly quoted newline names and a literal containing newlines; a stock `CASE` expression also produces newlines without a special setting | Quoting preserves identifier bytes, and `CASE` deparsing adds indentation even with `pretty=false`; these fixtures exercise complete local-value reads and generated-query execution ([ruleutils.c#identifier-bytes](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10703-L10716), [ruleutils.c#pretty-flags](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1117-L1131), [ruleutils.c#CASE-formatting](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L8309-L8315)) |
| Collation name | Stock `C` collation → a copy with a newline in its name | `CREATE COLLATION ... FROM` copies the source locale/provider properties; this changes the name, not the comparison semantics, and exercises encoded server-quoted collation transport ([collationcmds.c#from-properties](../../../../raw/postgres-12/src/backend/commands/collationcmds.c#L119-L145)) |
| Custom index-AM identities | Built-in hash OID 405/name `hash` → two new AM OIDs with pipe/newline names, stock `hashhandler` and integer hash support | `CREATE ACCESS METHOD` stores the supplied name and handler under a new OID. The checker's explicit built-in identity policy classifies both as unsupported before ordering generation; the fixture tests inventory framing, not broader AM completeness ([pg_am.dat#hash-identity](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L21-L23), [amcmds.c#new-name-and-handler](../../../../raw/postgres-12/src/backend/commands/amcmds.c#L74-L95)) |
| Deleted-row fixture | `autovacuum_enabled` default true → false on `hic_edge.deleted`; storage parameter | Keeps the ten deleted rows' index entries until the checker reads them, so `unresolved_index_tids` is 10 on the first run; with the default, autovacuum could remove them first ([reloptions.c#autovacuum_enabled](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L108-L115), [autovacuum.c#av_enabled](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3026)) |
| Damage-hiding settings | Defaults `off`, `off`, `on` → `zero_damaged_pages=on`, `ignore_checksum_failure=on` or `track_counts=off` through PGOPTIONS for one superuser CLI run each; session scope | Exercises the refusal, exit 2; no count runs under them ([guc.c#damage-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1133-L1162), [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1393-L1399)) |
| Idle timeout | Default 0 → 1 second through PGOPTIONS; user context, session scope | Shows that the runner's own session setting keeps the checking session alive across a 2-second pause ([guc.c#idle-timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2400-L2408)) |
| Drift DDL | No concurrent DDL by default → a second session runs TRUNCATE, REFRESH, DROP and CREATE, and three ALTER INDEX RENAME after the checker's snapshot | Each changed relation is ERROR; without the DDL all five would pass ([mvcc.sgml#DDL-caveat](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1689-L1702)) |
| Physical damage | Intact files by default → with the owned server stopped, `dd ... conv=notrunc` writes block 1 of `hic_damage_zero.t` as zeros and block 1 of `hic_damage_header.t` as 0xFF bytes | Verification accepts an all-zero page, so both scans lose the zeroed block's 58 rows; the 0xFF block fails verification, so every read of it raises XX001 ([bufpage.c#all-zero-accepted](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L123-L142), [bufmgr.c#zero-damaged-pages](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925)). Undamaged, both tables would pass with 300 rows. The server is stopped first because the shutdown checkpoint writes every dirty buffer, so no later write covers the injected bytes, and restarted because a block still in shared buffers is used without verification ([xlog.c#shutdown-checkpoint](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8348), [bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1775-L1785), [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L763-L793)) |
| Cached damage | No file write under a running server by default → after `VACUUM (FREEZE)` and `CHECKPOINT`, `dd` zeroes block 1 of `hic_damage_cached.t` while the owned server runs | The block's clean copy stays in shared buffers and a buffer hit is not verified, so the checker reads all 300 rows; after the physical leg's restart the block comes from the file and loses its 58 rows ([bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L763-L793), [bufmgr.c#verify-after-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925)). The checkpoint leaves the cached copy clean, so the shutdown checkpoint does not write it back over the zeros ([bufmgr.c#BufferSync](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L1775-L1785)) |
| Mid-run reload | `zero_damaged_pages` `off` from the configuration → `ALTER SYSTEM SET zero_damaged_pages = on` and `pg_reload_conf()` after the checker's snapshot, reset the same way after its loop; `superuser` context, reload | The reload reaches the open checking session between statements, and the next count's recorded settings stop the run with exit status 2 ([postgres.c#reload-between-statements](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4216-L4220)) |
| Message level | `client_min_messages` default `notice` → `error` through PGOPTIONS for one plain psql probe and one runner session; user context, session scope | The plain session receives no warning for `ROLLBACK` outside a transaction; the runner's own `SET` restores `notice`, and its session receives the `25P01` warning ([elog.c#client-output](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L301-L305)) |
| Unavailable TID signal | Recorded signal → the accepted index attempt's `unresolved` file blanked after the loop; synthetic report input | Shows that the schema total stays 0 and `unresolved_tids_unavailable` counts the relation; nothing is damaged |
| Lock/error/output fixtures | No blocker/truncation/termination/output obstacle by default → explicit second-session lock, cut JSON, terminated backend, report filename directory | Exercises 55P03, JSON parse failure, connection loss and local I/O failure; no default-leg measurement is claimed ([postgres.c#timeout-errors](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3082-L3095), [json.c#parse-error](../../../../raw/postgres-12/src/backend/utils/adt/json.c#L1126-L1137), [signalfuncs.c#termination](../../../../raw/postgres-12/src/backend/storage/ipc/signalfuncs.c#L128-L137)) |

All CREATE/ALTER/INSERT/UPDATE/DELETE/DROP statements in this harness are **disposable fixtures**, confined to its owned cluster and test databases. Plan and evaluator units are synthetic evidence, not damaged-index experiments. The driver and its expected-error probes use only Bash and SQL, with the same `\echo` synchronization as the checker.

```bash
set -uo pipefail
set -e
umask 077
root=$(pwd -P)
work="$root/.wiki-runtime/tmp/hic12"
src="$root/raw/postgres-12"
bin="$work/install/bin"
data="$work/data"
socket="$work/socket"
page="$root/wiki/v12/questions/indexing/sql-heap-index-visible-count-checker-plan.md"
pin=45b88269a353ad93744772791feb6d01bc7e1e42
jobs=${JOBS:-2}
started_here=false
case "$root" in *[!a-zA-Z0-9_./-]*) printf '%s\n' 'Use a short repository path without whitespace or shell metacharacters.' >&2; exit 1;; esac
case "$jobs" in ''|*[!0-9]*|0) printf '%s\n' 'JOBS must be a positive integer.' >&2; exit 1;; esac
[ "${#socket}" -le 80 ]
[ -f "$root/AGENTS.md" ] && [ -f "$src/configure" ]
export LC_ALL=C
export TMPDIR="$work/tmp"
export PGPASSFILE="$work/pgpass"
export PGCLIENTENCODING=UTF8
unset PGHOSTADDR PGSERVICE PGSERVICEFILE PGOPTIONS
owned() {
    [ -f "$work/.hic12-owned" ] || return 1
    IFS= read -r owner < "$work/.hic12-owned"
    [ "$owner" = postgres-llm-wiki-hic12-v1 ]
}
refuse_unowned_server() {
    if [ -f "$data/postmaster.pid" ] && [ "$started_here" != true ]; then
        printf 'Refusing running server not started by this invocation: %s\n' "$data" >&2
        return 1
    fi
}
same_server() {
    local actual
    actual=$("$bin/psql" -X -qAt -v ON_ERROR_STOP=1 -h "$socket" -p 55447 -U hic_dba -d postgres \
        -c "SET /* wiki_hic12_own_timeout */ statement_timeout = '10s'" \
        -c "SET /* wiki_hic12_own_lock_timeout */ lock_timeout = '2s'" \
        -c "SHOW /* wiki_hic12_own_directory */ data_directory;")
    [ "$actual" = "$data" ]
}
port_free() {
    # Nothing may accept a TCP connection on loopback port 55447. A refused
    # IPv4 connection also shows that this Bash supports /dev/tcp.
    if (exec 9<> /dev/tcp/127.0.0.1/55447) 2> "$work/port-ipv4.txt"; then
        printf '%s\n' 'A listener accepted a connection on 127.0.0.1:55447.' >&2
        return 1
    fi
    if ! grep -F -q 'Connection refused' "$work/port-ipv4.txt"; then
        printf 'Bash could not probe the loopback port: %s\n' "$(cat "$work/port-ipv4.txt")" >&2
        return 1
    fi
    if (exec 9<> /dev/tcp/::1/55447) 2> "$work/port-ipv6.txt"; then
        printf '%s\n' 'A listener accepted a connection on [::1]:55447.' >&2
        return 1
    fi
}
stop_owned() {
    owned || return 1
    refuse_unowned_server || return 1
    if [ -f "$data/postmaster.pid" ]; then
        same_server || return 1
        "$bin/pg_ctl" -D "$data" -m fast -w stop
        started_here=false
    fi
    [ ! -e "$data/postmaster.pid" ]
    ps -A -o pid= -o args= > "$work/processes.txt"
    if grep -F -- "$data" "$work/processes.txt"; then return 1; fi
    [ ! -e "$socket/.s.PGSQL.55447" ]
    [ ! -e "$socket/.s.PGSQL.55447.lock" ]
    port_free
}
on_exit() {
    local rc=$?
    trap - EXIT
    if [ "$started_here" = true ]; then
        stop_owned || rc=1
    fi
    exit "$rc"
}
trap on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
prepare() {
    [ "$(git -C "$src" rev-parse HEAD)" = "$pin" ]
    [ -z "$(git --no-optional-locks -C "$src" status --porcelain --untracked-files=no)" ]
    mkdir -p "$root/.wiki-runtime/tmp"
    ls -ld "$root/.wiki-runtime/tmp"
    if [ -e "$work" ]; then
        owned
        refuse_unowned_server
    else
        mkdir -m 700 "$work"
        printf '%s\n' postgres-llm-wiki-hic12-v1 > "$work/.hic12-owned"
    fi
    ls -ld "$work"
    mkdir -p "$work/build" "$work/socket" "$work/tmp"
    : > "$work/pgpass"
    materialize
}
build() {
    prepare
    if [ -f "$data/postmaster.pid" ]; then stop_owned; fi
    if [ ! -f "$work/build/Makefile" ]; then
        # GCC 15 defaults to C23, where bool is a keyword that c.h cannot
        # typedef; PostgreSQL 12 is C99 code, so the language mode is pinned.
        (cd "$work/build" && "$src/configure" --prefix="$work/install" --without-readline --without-zlib \
            CFLAGS='-O2 -std=gnu99') > "$work/configure.log" 2>&1
    fi
    make -C "$work/build" -j "$jobs" > "$work/build.log" 2>&1
    make -C "$work/build" install > "$work/install.log" 2>&1
    "$bin/postgres" --version
}
init() {
    prepare
    [ -x "$bin/pg_ctl" ]
    if [ ! -f "$data/PG_VERSION" ]; then
        [ ! -e "$data" ]
        "$bin/initdb" -D "$data" -U hic_dba -A trust --locale=C --encoding=UTF8 > "$work/initdb.log" 2>&1
    fi
    if [ ! -f "$data/postmaster.pid" ]; then
        port_free
        # Preflight excluded any pre-existing server. Own even a partially
        # successful startup so the EXIT trap can clean it up on failure.
        started_here=true
        "$bin/pg_ctl" -D "$data" -l "$work/postgres.log" \
            -o "-p 55447 -c listen_addresses='' -c unix_socket_directories='$socket' -c unix_socket_permissions=0700" \
            -w start
    fi
    same_server
}
verify() {
    init
    "$bin/pg_config" --configure > "$work/build-options.txt"
    uname -s -r -m > "$work/platform.txt"
    bash --version | sed -n 1p >> "$work/platform.txt"
    "$bin/psql" -X -qAt -v ON_ERROR_STOP=1 -h "$socket" -p 55447 -U hic_dba -d postgres \
        > "$work/platform-sql.txt" <<'HIC_PLATFORM_SQL'
SET /* wiki_hic12_platform_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_platform_lock_timeout */ lock_timeout = '2s';
SELECT /* wiki_hic12_platform_version */ version();
SELECT /* wiki_hic12_platform_settings */ name, setting, context, boot_val
FROM pg_catalog.pg_settings
WHERE name IN (
    'block_size', 'server_encoding', 'lc_collate', 'lc_ctype', 'lc_messages', 'TimeZone',
    'enable_seqscan', 'enable_indexscan', 'enable_indexonlyscan', 'enable_bitmapscan',
    'enable_sort', 'max_parallel_workers_per_gather', 'row_security', 'quote_all_identifiers',
    'statement_timeout', 'lock_timeout', 'old_snapshot_threshold', 'listen_addresses',
    'unix_socket_directories', 'unix_socket_permissions', 'port', 'search_path', 'role',
    'application_name', 'standard_conforming_strings', 'track_counts', 'default_table_access_method'
) ORDER BY name;
SELECT /* wiki_hic12_platform_control */ row_to_json(control) FROM pg_catalog.pg_control_init() AS control;
HIC_PLATFORM_SQL
    bash "$work/test.sh" > "$work/suite-current.log" 2>&1
    cat "$work/platform.txt" "$work/platform-sql.txt" "$work/suite-current.log"
    printf 'Latest artifacts: %s\n' "$(cat "$work/last-test-output")"
}
cleanup() {
    if [ ! -e "$work" ]; then return; fi
    owned
    stop_owned
    [ "$work" = "$root/.wiki-runtime/tmp/hic12" ] && [ ! -L "$work" ]
    rm -r "$work" < /dev/null
    printf '%s\n' 'Owned cluster stopped; PID, processes, sockets and loopback port checked; sandbox deleted.'
}

materialize() {
    local fence first_line
    fence=$(printf '\140\140\140')
    sed -n '/^### Runner script$/,/^##/p' "$page" |
        sed -n "/^${fence}bash$/,/^${fence}$/p" |
        sed '1d;$d' > "$work/hic12-check.sh"
    [ -s "$work/hic12-check.sh" ]
    IFS= read -r first_line < "$work/hic12-check.sh"
    [ "$first_line" = '# hic12-check.sh: self-contained Bash checker for PostgreSQL 12.2' ]
    bash -n "$work/hic12-check.sh"
    sed -n '/^## Measurement Script$/,/^## Context Reviewed$/p' "$page" |
        sed -n "/^${fence}bash$/,/^${fence}$/p" |
        sed '1d;$d' > "$work/hic12-measure.sh"
    bash -n "$work/hic12-measure.sh"
    cat > "$work/fixtures.sql" <<'HIC_FIXTURES_SQL'
SET /* wiki_hic12_fixture_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_fixture_lock_timeout */ lock_timeout = '2s';
CREATE /* wiki_hic12_positive_schema */ SCHEMA hic_positive;
CREATE /* wiki_hic12_outside_schema */ SCHEMA hic_outside;
CREATE /* wiki_hic12_negative_schema */ SCHEMA hic_negative;
CREATE /* wiki_hic12_choice_schema */ SCHEMA hic_choice;
CREATE /* wiki_hic12_rls_schema */ SCHEMA hic_rls;
CREATE /* wiki_hic12_denied_schema */ SCHEMA hic_denied;
CREATE /* wiki_hic12_namespace_schema */ SCHEMA hic_namespace_denied;
CREATE /* wiki_hic12_concurrent_schema */ SCHEMA hic_concurrent;
CREATE /* wiki_hic12_gate_schema */ SCHEMA hic_gate;
CREATE /* wiki_hic12_empty_schema */ SCHEMA hic_empty_scope;
CREATE /* wiki_hic12_multiline_schema */ SCHEMA hic_multiline;
CREATE /* wiki_hic12_am_names_schema */ SCHEMA hic_am_names;

-- Disposable regressions: deparser formatting and quoted text must survive
-- catalog transport, local storage, query assembly and actual index execution.
CREATE /* wiki_hic12_case_table */ TABLE hic_multiline.case_key(k integer);
INSERT /* wiki_hic12_case_rows */ INTO hic_multiline.case_key VALUES (1), (2), (NULL), (-1);
CREATE /* wiki_hic12_case_index */ INDEX case_idx ON hic_multiline.case_key
((CASE WHEN k < 2 THEN k ELSE -k END));
CREATE /* wiki_hic12_newline_column */ TABLE hic_multiline.column_key("key
name" integer);
INSERT /* wiki_hic12_newline_column_rows */ INTO hic_multiline.column_key VALUES (1), (2), (NULL), (-1);
CREATE /* wiki_hic12_newline_column_index */ INDEX column_idx ON hic_multiline.column_key("key
name");
CREATE /* wiki_hic12_newline_collation */ COLLATION hic_multiline."C
copy" FROM pg_catalog."C";
CREATE /* wiki_hic12_newline_collation_table */ TABLE hic_multiline.collation_key(k text);
INSERT /* wiki_hic12_newline_collation_rows */ INTO hic_multiline.collation_key VALUES ('a'), ('z'), (NULL), ('A');
CREATE /* wiki_hic12_newline_collation_index */ INDEX collation_idx ON hic_multiline.collation_key(k COLLATE hic_multiline."C
copy");
CREATE /* wiki_hic12_newline_literal_table */ TABLE hic_multiline.literal_key(k text);
INSERT /* wiki_hic12_newline_literal_rows */ INTO hic_multiline.literal_key VALUES ('a'), ('z'), (NULL), ('A');
CREATE /* wiki_hic12_newline_literal_index */ INDEX literal_idx ON hic_multiline.literal_key
((coalesce(k, 'line
end
')));
GRANT /* wiki_hic12_multiline_usage */ USAGE ON SCHEMA hic_multiline TO hic_reader;
GRANT /* wiki_hic12_multiline_select */ SELECT ON ALL TABLES IN SCHEMA hic_multiline TO hic_reader;

-- These custom index AM identities reuse only the stock hash handler and
-- integer equality/hash function. The checker must classify them as unsupported.
CREATE /* wiki_hic12_pipe_am */ ACCESS METHOD "pipe|am" TYPE INDEX HANDLER pg_catalog.hashhandler;
CREATE /* wiki_hic12_pipe_opclass */ OPERATOR CLASS hic_am_names.pipe_ops
DEFAULT FOR TYPE integer USING "pipe|am" AS
OPERATOR 1 = (integer,integer), FUNCTION 1 pg_catalog.hashint4(integer);
CREATE /* wiki_hic12_newline_am */ ACCESS METHOD "line
am" TYPE INDEX HANDLER pg_catalog.hashhandler;
CREATE /* wiki_hic12_newline_opclass */ OPERATOR CLASS hic_am_names.newline_ops
DEFAULT FOR TYPE integer USING "line
am" AS
OPERATOR 1 = (integer,integer), FUNCTION 1 pg_catalog.hashint4(integer);
CREATE /* wiki_hic12_am_name_table */ TABLE hic_am_names.t(k integer);
INSERT /* wiki_hic12_am_name_rows */ INTO hic_am_names.t VALUES (1), (NULL);
CREATE /* wiki_hic12_pipe_am_index */ INDEX pipe_idx ON hic_am_names.t USING "pipe|am"(k);
CREATE /* wiki_hic12_newline_am_index */ INDEX newline_idx ON hic_am_names.t USING "line
am"(k);

CREATE /* wiki_hic12_empty_table */ TABLE hic_positive.empty_table(id integer);
CREATE /* wiki_hic12_empty_index */ INDEX empty_idx ON hic_positive.empty_table(id);
CREATE /* wiki_hic12_null_table */ TABLE hic_positive.nulls_duplicates(id integer, k text);
INSERT /* wiki_hic12_null_rows */ INTO hic_positive.nulls_duplicates
VALUES (1, 'A'), (1, 'a'), (NULL, NULL), (2, NULL), (NULL, 'Z');
CREATE /* wiki_hic12_null_index */ INDEX nulls_idx ON hic_positive.nulls_duplicates(id);
BEGIN /* wiki_hic12_abort_fixture */;
INSERT /* wiki_hic12_abort_row */ INTO hic_positive.nulls_duplicates VALUES (99, 'aborted');
ROLLBACK /* wiki_hic12_abort_fixture_done */;
CREATE /* wiki_hic12_expression_table */ TABLE hic_positive.expression_table(k text);
INSERT /* wiki_hic12_expression_rows */ INTO hic_positive.expression_table VALUES ('A'), ('a'), (NULL), ('Z');
CREATE /* wiki_hic12_expression_index */ INDEX expression_idx ON hic_positive.expression_table(lower(k), (coalesce(k, 'x,y')));
CREATE /* wiki_hic12_pattern_table */ TABLE hic_positive.pattern_table(k text);
INSERT /* wiki_hic12_pattern_rows */ INTO hic_positive.pattern_table VALUES ('aa'), ('ab'), ('A'), (NULL);
CREATE /* wiki_hic12_pattern_index */ INDEX pattern_idx ON hic_positive.pattern_table(k text_pattern_ops);
CREATE /* wiki_hic12_collation_table */ TABLE hic_positive.collation_table(k text);
INSERT /* wiki_hic12_collation_rows */ INTO hic_positive.collation_table VALUES ('z'), ('A'), (NULL), ('é');
CREATE /* wiki_hic12_collation_index */ INDEX collation_idx ON hic_positive.collation_table(k COLLATE "C" DESC NULLS LAST);
CREATE /* wiki_hic12_composite_table */ TABLE hic_positive.composite_table(a integer, b text, payload text);
INSERT /* wiki_hic12_composite_rows */ INTO hic_positive.composite_table
VALUES (1, 'a', 'x'), (1, NULL, 'y'), (NULL, 'b', 'z'), (NULL, NULL, NULL);
CREATE /* wiki_hic12_composite_index */ INDEX composite_idx
ON hic_positive.composite_table(a DESC NULLS LAST, b ASC NULLS FIRST) INCLUDE(payload);

CREATE /* wiki_hic12_inheritance_parent */ TABLE hic_positive.inh_parent(id integer);
CREATE /* wiki_hic12_inheritance_child */ TABLE hic_positive.inh_child() INHERITS(hic_positive.inh_parent);
INSERT /* wiki_hic12_parent_row */ INTO hic_positive.inh_parent VALUES (1);
INSERT /* wiki_hic12_child_rows */ INTO hic_positive.inh_child VALUES (2), (3);
CREATE /* wiki_hic12_parent_index */ UNIQUE INDEX inh_parent_idx ON hic_positive.inh_parent(id);
CREATE /* wiki_hic12_child_index */ INDEX inh_child_idx ON hic_positive.inh_child(id);

CREATE /* wiki_hic12_stub_fdw */ FOREIGN DATA WRAPPER hic_stub;
CREATE /* wiki_hic12_stub_server */ SERVER hic_stub FOREIGN DATA WRAPPER hic_stub;
CREATE /* wiki_hic12_partition_parent */ TABLE hic_positive.partitioned(id integer) PARTITION BY RANGE(id);
CREATE /* wiki_hic12_partition_local */ TABLE hic_positive.partition_local
PARTITION OF hic_positive.partitioned FOR VALUES FROM (0) TO (10);
CREATE /* wiki_hic12_partition_outside */ TABLE hic_outside.partition_outside
PARTITION OF hic_positive.partitioned FOR VALUES FROM (10) TO (20);
CREATE /* wiki_hic12_partition_foreign */ FOREIGN TABLE hic_positive.partition_foreign
PARTITION OF hic_positive.partitioned FOR VALUES FROM (20) TO (30) SERVER hic_stub;
INSERT /* wiki_hic12_partition_rows */ INTO hic_positive.partitioned VALUES (1), (2), (11);
CREATE /* wiki_hic12_partition_local_index */ INDEX partition_local_idx ON hic_positive.partition_local(id);
CREATE /* wiki_hic12_partition_outside_index */ INDEX partition_outside_idx ON hic_outside.partition_outside(id);
CREATE /* wiki_hic12_outside_unrelated */ TABLE hic_outside.unrelated(id integer);

CREATE /* wiki_hic12_materialized */ MATERIALIZED VIEW hic_positive.populated_mv
AS SELECT id FROM hic_positive.nulls_duplicates;
CREATE /* wiki_hic12_materialized_index */ INDEX populated_mv_idx ON hic_positive.populated_mv(id);
CREATE /* wiki_hic12_empty_materialized */ MATERIALIZED VIEW hic_positive.empty_mv
AS SELECT id FROM hic_positive.nulls_duplicates WHERE false;
CREATE /* wiki_hic12_empty_materialized_index */ INDEX empty_mv_idx ON hic_positive.empty_mv(id);
CREATE /* wiki_hic12_unpopulated_materialized */ MATERIALIZED VIEW hic_positive.unpopulated_mv
AS SELECT id FROM hic_positive.nulls_duplicates WITH NO DATA;
CREATE /* wiki_hic12_plain_view */ VIEW hic_positive.plain_view AS SELECT * FROM hic_positive.nulls_duplicates;
CREATE /* wiki_hic12_sequence */ SEQUENCE hic_positive.unchecked_sequence;
CREATE /* wiki_hic12_unlogged */ UNLOGGED TABLE hic_positive.unlogged_table(id integer);
INSERT /* wiki_hic12_unlogged_rows */ INTO hic_positive.unlogged_table VALUES (1), (NULL);
CREATE /* wiki_hic12_unlogged_index */ INDEX unlogged_idx ON hic_positive.unlogged_table(id);
CREATE /* wiki_hic12_quoted_table */ TABLE hic_positive."Odd table ""x"""("k'"";" integer);
INSERT /* wiki_hic12_quoted_rows */ INTO hic_positive."Odd table ""x""" VALUES (1), (NULL);
CREATE /* wiki_hic12_quoted_index */ INDEX "odd idx ""name"""
ON hic_positive."Odd table ""x"""("k'"";");
GRANT /* wiki_hic12_reader_schema_usage */ USAGE ON SCHEMA hic_positive, hic_outside TO hic_reader;
GRANT /* wiki_hic12_reader_select */ SELECT ON ALL TABLES IN SCHEMA hic_positive, hic_outside TO hic_reader;

CREATE /* wiki_hic12_no_index */ TABLE hic_negative.no_index(id integer);
CREATE /* wiki_hic12_partial_table */ TABLE hic_negative.partial_only(id integer);
INSERT /* wiki_hic12_partial_rows */ INTO hic_negative.partial_only VALUES (-1), (0), (1);
CREATE /* wiki_hic12_partial_index */ INDEX partial_idx ON hic_negative.partial_only(id) WHERE id > 0;
CREATE /* wiki_hic12_hash_table */ TABLE hic_negative.hash_only(id integer);
INSERT /* wiki_hic12_hash_rows */ INTO hic_negative.hash_only VALUES (1), (NULL);
CREATE /* wiki_hic12_hash_index */ INDEX hash_idx ON hic_negative.hash_only USING hash(id);
CREATE /* wiki_hic12_constant_table */ TABLE hic_negative.constant_index(id integer);
INSERT /* wiki_hic12_constant_rows */ INTO hic_negative.constant_index VALUES (1), (2);
CREATE /* wiki_hic12_constant_index */ INDEX constant_idx ON hic_negative.constant_index((1));
CREATE /* wiki_hic12_literal_table */ TABLE hic_negative.constant_literals(id integer);
INSERT /* wiki_hic12_literal_rows */ INTO hic_negative.constant_literals VALUES (1), (2);
CREATE /* wiki_hic12_integer_literal_index */ INDEX constant_two_idx ON hic_negative.constant_literals((2));
CREATE /* wiki_hic12_decimal_literal_index */ INDEX constant_decimal_idx ON hic_negative.constant_literals((1.5));
CREATE /* wiki_hic12_user_function */ FUNCTION hic_negative.increment(integer) RETURNS integer
LANGUAGE sql IMMUTABLE AS 'SELECT /* wiki_hic12_fixture_increment */ $1 + 1';
CREATE /* wiki_hic12_user_expression */ TABLE hic_negative.user_expression(id integer);
INSERT /* wiki_hic12_user_expression_rows */ INTO hic_negative.user_expression VALUES (1), (2);
CREATE /* wiki_hic12_user_expression_index */ INDEX user_expression_idx
ON hic_negative.user_expression(hic_negative.increment(id));
CREATE /* wiki_hic12_user_domain */ DOMAIN hic_negative.integer_domain AS integer;
CREATE /* wiki_hic12_domain_table */ TABLE hic_negative.domain_table(id hic_negative.integer_domain);
CREATE /* wiki_hic12_domain_index */ INDEX domain_idx ON hic_negative.domain_table(id);
CREATE /* wiki_hic12_invalid_table */ TABLE hic_negative.invalid_index(id integer);
INSERT /* wiki_hic12_invalid_rows */ INTO hic_negative.invalid_index VALUES (1), (1);

CREATE /* wiki_hic12_choice_table */ TABLE hic_choice.t(k integer, payload text);
INSERT /* wiki_hic12_choice_rows */ INTO hic_choice.t SELECT n, repeat(md5(n::text), 24) FROM generate_series(1, 500) AS n;
CREATE /* wiki_hic12_wide_index */ INDEX wide_idx ON hic_choice.t(k) INCLUDE(payload);
CREATE /* wiki_hic12_small_index */ INDEX compact_idx ON hic_choice.t(k);
ANALYZE /* wiki_hic12_choice_statistics */ hic_choice.t;

CREATE /* wiki_hic12_rls_table */ TABLE hic_rls.t(id integer);
INSERT /* wiki_hic12_rls_rows */ INTO hic_rls.t VALUES (1), (2);
CREATE /* wiki_hic12_rls_index */ INDEX rls_idx ON hic_rls.t(id);
CREATE /* wiki_hic12_rls_policy */ POLICY subset ON hic_rls.t USING (id = 1);
ALTER /* wiki_hic12_rls_enabled */ TABLE hic_rls.t ENABLE ROW LEVEL SECURITY;
ALTER /* wiki_hic12_rls_forced */ TABLE hic_rls.t FORCE ROW LEVEL SECURITY;
ALTER /* wiki_hic12_rls_owner */ TABLE hic_rls.t OWNER TO hic_reader;
GRANT /* wiki_hic12_rls_schema_usage */ USAGE ON SCHEMA hic_rls TO hic_reader;
CREATE /* wiki_hic12_denied_table */ TABLE hic_denied.t(id integer);
CREATE /* wiki_hic12_denied_index */ INDEX denied_idx ON hic_denied.t(id);
GRANT /* wiki_hic12_denied_schema_usage */ USAGE ON SCHEMA hic_denied TO hic_reader;
CREATE /* wiki_hic12_namespace_table */ TABLE hic_namespace_denied.t(id integer);
CREATE /* wiki_hic12_namespace_index */ INDEX namespace_idx ON hic_namespace_denied.t(id);
GRANT /* wiki_hic12_namespace_select */ SELECT ON hic_namespace_denied.t TO hic_reader;

CREATE /* wiki_hic12_concurrent_table */ TABLE hic_concurrent.t(id integer, payload text) WITH (fillfactor = 50);
INSERT /* wiki_hic12_concurrent_rows */ INTO hic_concurrent.t SELECT n, repeat('a', 32) FROM generate_series(1, 100) AS n;
CREATE /* wiki_hic12_concurrent_index */ INDEX concurrent_idx ON hic_concurrent.t(id);
CREATE /* wiki_hic12_gate_table */ TABLE hic_gate.t(k integer, payload text) WITH (fillfactor = 50);
INSERT /* wiki_hic12_gate_rows */ INTO hic_gate.t SELECT n, repeat('a', 32) FROM generate_series(1, 120) AS n;
CREATE /* wiki_hic12_lock_schema */ SCHEMA hic_lock;
CREATE /* wiki_hic12_lock_table */ TABLE hic_lock.t(id integer);
INSERT /* wiki_hic12_lock_rows */ INTO hic_lock.t SELECT n FROM generate_series(1, 10) AS n;
CREATE /* wiki_hic12_lock_index */ INDEX lock_idx ON hic_lock.t(id);
CREATE /* wiki_hic12_guard_schema */ SCHEMA hic_psql_guard;
CREATE /* wiki_hic12_guard_table */ TABLE hic_psql_guard.psql_trigger(a integer[], "ERROR" integer);
INSERT /* wiki_hic12_guard_rows */ INTO hic_psql_guard.psql_trigger VALUES (ARRAY[1, 2, 3], 2), (ARRAY[4, 5], 1);
CREATE /* wiki_hic12_guard_index */ INDEX psql_guard_idx ON hic_psql_guard.psql_trigger ((a[1 : "ERROR"]));

-- Edge cases: a key column whose name equals the default output name of an
-- unnamed SELECT item, and index entries left behind by committed deletes.
-- autovacuum_enabled = off keeps those entries until the checker reads them.
CREATE /* wiki_hic12_edge_schema */ SCHEMA hic_edge;
CREATE /* wiki_hic12_output_name_table */ TABLE hic_edge.output_name("?column?" integer);
INSERT /* wiki_hic12_output_name_rows */ INTO hic_edge.output_name VALUES (3), (1), (NULL);
CREATE /* wiki_hic12_output_name_index */ INDEX output_name_idx ON hic_edge.output_name("?column?");
CREATE /* wiki_hic12_deleted_table */ TABLE hic_edge.deleted(id integer, pad text) WITH (autovacuum_enabled = off);
CREATE /* wiki_hic12_deleted_index */ INDEX deleted_idx ON hic_edge.deleted(id);
INSERT /* wiki_hic12_deleted_rows */ INTO hic_edge.deleted SELECT n, 'p' FROM generate_series(1, 100) AS n;
DELETE /* wiki_hic12_deleted_remove */ FROM hic_edge.deleted WHERE id <= 10;

-- Identity drift: each relation is changed by another session after the
-- checker's snapshot and before the checker first locks it.
CREATE /* wiki_hic12_drift_schema */ SCHEMA hic_drift;
CREATE /* wiki_hic12_drift_base_schema */ SCHEMA hic_drift_base;
CREATE /* wiki_hic12_drift_truncated */ TABLE hic_drift.truncated(id integer);
INSERT /* wiki_hic12_drift_truncated_rows */ INTO hic_drift.truncated SELECT n FROM generate_series(1, 5) AS n;
CREATE /* wiki_hic12_drift_truncated_index */ INDEX truncated_idx ON hic_drift.truncated(id);
CREATE /* wiki_hic12_drift_mv_base */ TABLE hic_drift_base.mv_base(id integer);
INSERT /* wiki_hic12_drift_mv_base_rows */ INTO hic_drift_base.mv_base VALUES (1), (2), (3);
CREATE /* wiki_hic12_drift_refreshed */ MATERIALIZED VIEW hic_drift.refreshed AS SELECT id FROM hic_drift_base.mv_base;
CREATE /* wiki_hic12_drift_refreshed_index */ INDEX refreshed_idx ON hic_drift.refreshed(id);
CREATE /* wiki_hic12_drift_recreated */ TABLE hic_drift.recreated(id integer);
INSERT /* wiki_hic12_drift_recreated_rows */ INTO hic_drift.recreated VALUES (1), (2), (3), (4);
CREATE /* wiki_hic12_drift_recreated_index */ INDEX recreated_idx ON hic_drift.recreated(id);
CREATE /* wiki_hic12_drift_swapped */ TABLE hic_drift.swapped(a integer, b integer);
INSERT /* wiki_hic12_drift_swapped_rows */ INTO hic_drift.swapped VALUES (1, 3), (2, 2), (3, 1);
CREATE /* wiki_hic12_drift_swap_a */ INDEX swap_a ON hic_drift.swapped(a);
CREATE /* wiki_hic12_drift_swap_b */ INDEX swap_b ON hic_drift.swapped(b);
CREATE /* wiki_hic12_drift_stable */ TABLE hic_drift.stable(id integer);
INSERT /* wiki_hic12_drift_stable_rows */ INTO hic_drift.stable VALUES (1), (2);
CREATE /* wiki_hic12_drift_stable_index */ INDEX stable_idx ON hic_drift.stable(id);

-- Physical fault leg: these tables get one damaged heap block each while the
-- owned server is stopped. About 50 rows fit in one 8 kB block.
CREATE /* wiki_hic12_damage_zero_schema */ SCHEMA hic_damage_zero;
CREATE /* wiki_hic12_damage_zero_table */ TABLE hic_damage_zero.t(id integer, pad text);
INSERT /* wiki_hic12_damage_zero_rows */ INTO hic_damage_zero.t SELECT n, repeat('z', 100) FROM generate_series(1, 300) AS n;
CREATE /* wiki_hic12_damage_zero_index */ INDEX damage_zero_idx ON hic_damage_zero.t(id);
CREATE /* wiki_hic12_damage_header_schema */ SCHEMA hic_damage_header;
CREATE /* wiki_hic12_damage_header_table */ TABLE hic_damage_header.t(id integer, pad text);
INSERT /* wiki_hic12_damage_header_rows */ INTO hic_damage_header.t SELECT n, repeat('h', 100) FROM generate_series(1, 300) AS n;
CREATE /* wiki_hic12_damage_header_index */ INDEX damage_header_idx ON hic_damage_header.t(id);

-- Cached-damage leg: block 1 of this table is zeroed while the server runs
-- and the block's clean copy is still in shared buffers.
CREATE /* wiki_hic12_damage_cached_schema */ SCHEMA hic_damage_cached;
CREATE /* wiki_hic12_damage_cached_table */ TABLE hic_damage_cached.t(id integer, pad text);
INSERT /* wiki_hic12_damage_cached_rows */ INTO hic_damage_cached.t SELECT n, repeat('c', 100) FROM generate_series(1, 300) AS n;
CREATE /* wiki_hic12_damage_cached_index */ INDEX damage_cached_idx ON hic_damage_cached.t(id);

-- A table name with psql's :' form and a trailing colon. psql never
-- substitutes inside a quoted identifier, so the heap count is safe to send.
CREATE /* wiki_hic12_guard_names_schema */ SCHEMA hic_guard_names;
CREATE /* wiki_hic12_guard_names_table */ TABLE hic_guard_names."odd:'name:"(k integer);
INSERT /* wiki_hic12_guard_names_rows */ INTO hic_guard_names."odd:'name:" VALUES (1), (2), (NULL);
CREATE /* wiki_hic12_guard_names_index */ INDEX guard_names_idx ON hic_guard_names."odd:'name:"(k);
HIC_FIXTURES_SQL
    cat > "$work/drift.sql" <<'HIC_DRIFT_SQL'
SET /* wiki_hic12_drift_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_drift_lock_timeout */ lock_timeout = '2s';
TRUNCATE /* wiki_hic12_drift_truncate */ hic_drift.truncated;
INSERT /* wiki_hic12_drift_mv_base_more */ INTO hic_drift_base.mv_base VALUES (4), (5);
REFRESH /* wiki_hic12_drift_refresh */ MATERIALIZED VIEW hic_drift.refreshed;
DROP /* wiki_hic12_drift_drop */ TABLE hic_drift.recreated;
CREATE /* wiki_hic12_drift_recreate */ TABLE hic_drift.recreated(id integer);
INSERT /* wiki_hic12_drift_recreate_rows */ INTO hic_drift.recreated VALUES (1), (2), (3), (4);
CREATE /* wiki_hic12_drift_recreate_index */ INDEX recreated_idx ON hic_drift.recreated(id);
ALTER /* wiki_hic12_drift_swap_1 */ INDEX hic_drift.swap_a RENAME TO swap_tmp;
ALTER /* wiki_hic12_drift_swap_2 */ INDEX hic_drift.swap_b RENAME TO swap_a;
ALTER /* wiki_hic12_drift_swap_3 */ INDEX hic_drift.swap_tmp RENAME TO swap_b;
HIC_DRIFT_SQL
    cat > "$work/writer.sql" <<'HIC_WRITER_SQL'
SET /* wiki_hic12_writer_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_writer_lock_timeout */ lock_timeout = '2s';
BEGIN /* wiki_hic12_writer_begin */;
UPDATE /* wiki_hic12_writer_hot */ hic_concurrent.t SET payload = repeat('b', 32);
SELECT /* wiki_hic12_writer_hot_count */ 'hot_updates', pg_stat_get_xact_tuples_hot_updated('hic_concurrent.t'::regclass);
SELECT /* wiki_hic12_writer_hot_assert */ 1 / (pg_stat_get_xact_tuples_hot_updated('hic_concurrent.t'::regclass) > 0)::integer;
UPDATE /* wiki_hic12_writer_nonhot */ hic_concurrent.t SET id = id + 1000 WHERE id <= 5;
SELECT /* wiki_hic12_writer_nonhot_assert */ 1 / (
    pg_stat_get_xact_tuples_updated('hic_concurrent.t'::regclass)
    > pg_stat_get_xact_tuples_hot_updated('hic_concurrent.t'::regclass)
)::integer;
DELETE /* wiki_hic12_writer_delete */ FROM hic_concurrent.t WHERE id BETWEEN 10 AND 15;
INSERT /* wiki_hic12_writer_insert */ INTO hic_concurrent.t VALUES (2000, 'committed');
COMMIT /* wiki_hic12_writer_commit */;
BEGIN /* wiki_hic12_writer_abort_begin */;
INSERT /* wiki_hic12_writer_abort_insert */ INTO hic_concurrent.t VALUES (3000, 'aborted');
ROLLBACK /* wiki_hic12_writer_abort */;
SELECT /* wiki_hic12_writer_current */ 'current_rows', count(*) FROM hic_concurrent.t;
SELECT /* wiki_hic12_writer_current_assert */ 1 / (count(*) = 95)::integer FROM hic_concurrent.t;
HIC_WRITER_SQL
    cat > "$work/gate-setup.sql" <<'HIC_GATE_SQL'
SET /* wiki_hic12_gate_setup_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_gate_setup_lock_timeout */ lock_timeout = '2s';
BEGIN /* wiki_hic12_gate_update_begin */;
UPDATE /* wiki_hic12_gate_hot_update */ hic_gate.t SET k = k + 1000;
SELECT /* wiki_hic12_gate_hot_count */ 'gate_hot_updates', pg_stat_get_xact_tuples_hot_updated('hic_gate.t'::regclass);
SELECT /* wiki_hic12_gate_hot_assert */ 1 / (pg_stat_get_xact_tuples_hot_updated('hic_gate.t'::regclass) > 0)::integer;
COMMIT /* wiki_hic12_gate_update_commit */;
CREATE /* wiki_hic12_gate_late_index */ INDEX gate_idx ON hic_gate.t(k);
SELECT /* wiki_hic12_gate_flag_assert */ 1 / indcheckxmin::integer
FROM pg_catalog.pg_index WHERE indexrelid = 'hic_gate.gate_idx'::regclass;
HIC_GATE_SQL
    cat > "$work/validator-tests.sql" <<'HIC_VALIDATOR_SQL'
SET /* wiki_hic12_unit_timeout */ statement_timeout = '30s';
SET /* wiki_hic12_unit_lock_timeout */ lock_timeout = '2s';
CREATE /* wiki_hic12_unit_cases */ TEMP TABLE hic_unit_cases (
    label text PRIMARY KEY,
    document jsonb NOT NULL,
    phase text NOT NULL,
    expected boolean NOT NULL
);
CREATE /* wiki_hic12_unit_assertions */ TEMP TABLE hic_unit_assertions (
    label text PRIMARY KEY,
    ok boolean NOT NULL CHECK (ok)
);
INSERT /* wiki_hic12_unit_heap */ INTO hic_unit_cases VALUES (
    'valid heap',
    '[{"Plan":{"Node Type":"ModifyTable","Operation":"Insert",
       "Schema":"pg_temp_test","Relation Name":"hic_count",
       "Actual Loops":1,"Actual Rows":0,"Parallel Aware":false,
       "Plans":[{"Node Type":"Aggregate","Strategy":"Plain","Partial Mode":"Simple",
          "Actual Loops":1,"Actual Rows":1,"Output":["''1''::bigint","count(*)"],
          "Plans":[{"Node Type":"Seq Scan","Schema":"fixture","Relation Name":"target",
             "Actual Loops":1,"Actual Rows":2,"Parallel Aware":false}]}]},
       "Execution Time":0.1}]'::jsonb,
    'heap', true
);
INSERT /* wiki_hic12_unit_index */ INTO hic_unit_cases
SELECT 'valid index',
       jsonb_set(document, '{0,Plan,Plans,0,Plans,0}',
         document #> '{0,Plan,Plans,0,Plans,0}' ||
         '{"Node Type":"Index Scan","Index Name":"idx","Scan Direction":"Forward"}'::jsonb),
       'index', true
FROM hic_unit_cases WHERE label = 'valid heap';
INSERT /* wiki_hic12_unit_leaf_rejections */ INTO hic_unit_cases
SELECT change.label,
       jsonb_set(document, '{0,Plan,Plans,0,Plans,0}',
         document #> '{0,Plan,Plans,0,Plans,0}' || change.patch),
       phase, false
FROM hic_unit_cases
CROSS JOIN (VALUES
    ('heap filter', '{"Filter":"true"}'::jsonb),
    ('heap index condition', '{"Index Cond":"true"}'::jsonb),
    ('heap recheck', '{"Recheck Cond":"true"}'::jsonb),
    ('heap never executed', '{"Actual Loops":0}'::jsonb),
    ('heap repeated', '{"Actual Loops":2}'::jsonb),
    ('heap parallel aware', '{"Parallel Aware":true}'::jsonb),
    ('wrong relation', '{"Relation Name":"another"}'::jsonb),
    ('wrong schema', '{"Schema":"another"}'::jsonb),
    ('index-only baseline', '{"Node Type":"Index Only Scan"}'::jsonb),
    ('custom scan', '{"Node Type":"Custom Scan"}'::jsonb),
    ('initplan', '{"Parent Relationship":"InitPlan"}'::jsonb),
    ('malformed children', '{"Plans":{}}'::jsonb)
) AS change(label, patch)
WHERE hic_unit_cases.label = 'valid heap';
INSERT /* wiki_hic12_unit_index_rejections */ INTO hic_unit_cases
SELECT change.label,
       jsonb_set(document, '{0,Plan,Plans,0,Plans,0}',
         document #> '{0,Plan,Plans,0,Plans,0}' || change.patch),
       phase, false
FROM hic_unit_cases
CROSS JOIN (VALUES
    ('index-only candidate', '{"Node Type":"Index Only Scan"}'::jsonb),
    ('index condition', '{"Index Cond":"(k IS NOT NULL)"}'::jsonb),
    ('index filter', '{"Filter":"true"}'::jsonb),
    ('index without identity', '{"Index Name":null}'::jsonb),
    ('index without direction', '{"Scan Direction":null}'::jsonb),
    ('index never executed', '{"Actual Loops":0}'::jsonb)
) AS change(label, patch)
WHERE hic_unit_cases.label = 'valid index';
INSERT /* wiki_hic12_unit_backward */ INTO hic_unit_cases
SELECT 'backward index',
       jsonb_set(document, '{0,Plan,Plans,0,Plans,0,Scan Direction}', '"Backward"'),
       phase, true
FROM hic_unit_cases WHERE label = 'valid index';
INSERT /* wiki_hic12_unit_top_rejections */ INTO hic_unit_cases
SELECT change.label, change.document, 'heap', false
FROM hic_unit_cases
CROSS JOIN LATERAL (VALUES
    ('multiple commands', document || document),
    ('missing document', '[]'::jsonb),
    ('not an array', '{}'::jsonb),
    ('no executed timing', document #- '{0,Execution Time}'),
    ('wrong capture target', jsonb_set(document, '{0,Plan,Relation Name}', '"another"')),
    ('not an insert', jsonb_set(document, '{0,Plan,Operation}', '"Update"')),
    ('extra branch', jsonb_set(document, '{0,Plan,Plans}',
        (document #> '{0,Plan,Plans}') || (document #> '{0,Plan,Plans}'))),
    ('grouped aggregate', jsonb_set(document, '{0,Plan,Plans,0,Strategy}', '"Sorted"')),
    ('partial aggregate', jsonb_set(document, '{0,Plan,Plans,0,Partial Mode}', '"Partial"')),
    ('wrong aggregate', jsonb_set(document, '{0,Plan,Plans,0,Output}', '["sum(k)"]')),
    ('one-time filter', jsonb_set(document, '{0,Plan,Plans,0,One-Time Filter}', '"true"'))
) AS change(label, document)
WHERE hic_unit_cases.label = 'valid heap';
INSERT /* wiki_hic12_unit_phase_rejection */ INTO hic_unit_cases
SELECT 'index in heap phase', document, 'heap', false
FROM hic_unit_cases WHERE label = 'valid index';
INSERT /* wiki_hic12_extra_shape_units */ INTO hic_unit_cases
SELECT change.label, jsonb_set(document,change.path,change.patch), 'heap', false
FROM hic_unit_cases CROSS JOIN (VALUES
 ('missing source loops', '{0,Plan,Plans,0,Plans,0,Actual Loops}'::text[], 'null'::jsonb),
 ('string source rows','{0,Plan,Plans,0,Plans,0,Actual Rows}'::text[], '"2"'::jsonb),
 ('negative source rows','{0,Plan,Plans,0,Plans,0,Actual Rows}'::text[], '-1'::jsonb),
 ('null children','{0,Plan,Plans,0,Plans}'::text[], 'null'::jsonb),
 ('scalar child','{0,Plan,Plans,0,Plans}'::text[], '[null]'::jsonb),
 ('wrong attempt marker','{0,Plan,Plans,0,Output}'::text[], '["99::bigint","count(*)"]'::jsonb),
 ('negative execution time','{0,Execution Time}'::text[], '-1'::jsonb),
 ('unknown wrapper','{0,Plan,Plans,0,Node Type}'::text[], '"Mystery"'::jsonb)
) change(label,path,patch) WHERE hic_unit_cases.label='valid heap';
HIC_VALIDATOR_SQL
    cat > "$work/test.sh" <<'HIC_TEST_SH'
set -u -o pipefail
umask 077
root=$(pwd -P)
work="$root/.wiki-runtime/tmp/hic12"
bin="$work/install/bin"
# The harness materializes the current published runner here.
. "$work/hic12-check.sh"
HIC_PSQL="$bin/psql"
export PGHOST="$work/socket" PGPORT=55447 PGUSER=hic_dba PGDATABASE=postgres PGAPPNAME=hic12-suite
db="hic_suite_$(date -u +%Y%m%d%H%M%S)_$$"
out="$work/results/$db"
mkdir -p "$out" || exit 1
passed=0
case_name=setup
trap hic_cleanup EXIT
fail() { printf 'FAIL [%s] %s\n' "$case_name" "$*"; exit 1; }
ok() { passed=$((passed+1)); printf 'ok %s\n' "$1"; }
sfile() { eval "printf '%s' \"\$HIC_DIR_$HIC_S/$1\""; }
expect() {
    hic_sql "$HIC_S" "$2" || fail "$1: $(cat "$(sfile stderr.last)")"
    [ "$HIC_VALUE" = t ] || fail "$1: returned $HIC_VALUE"
    ok "$1"
}
expect_error() {
    local rc
    hic_sql "$HIC_S" "$3"; rc=$?
    [ "$rc" = 1 ] || fail "$1: expected SQLSTATE $2, status $rc"
    hic_parse_error "$HIC_S" && [ "$HIC_ERR_CODE" = "$2" ] || fail "$1: SQLSTATE ${HIC_ERR_CODE:-none}"
    ok "$1"
}
open_case() {
    case_name=$1; HIC_OUT="$out/$1"; mkdir -m 700 "$HIC_OUT" || fail directory
    export PGOPTIONS="-c quote_all_identifiers=on ${CASE_OPTIONS:-}"
    hic_open main 7 8 || fail connection
    unset PGOPTIONS
    if [ -n "${2:-}" ]; then hic_sql main "SET /* wiki_hic12_test_role */ ROLE $2;" || fail role; fi
}
start_run() {
    HIC_SCHEMA=$1; HIC_EXTERNAL=$2; HIC_EXHAUSTIVE_BOOL=$3
    hic_initialize || fail "initialization: ${HIC_STOP:-} $(cat "$(sfile stderr.last)")"
}
run_loop() { hic_attempt_loop || fail "loop: ${HIC_STOP:-} $(cat "$(sfile stderr.last)")"; }
finish_case() {
    hic_finish || fail "finish: $(cat "$(sfile stderr.last)")"
    expect "${case_name}_no_helpers_or_views" "SELECT /* wiki_hic12_minimal_objects */
 (SELECT count(*)=0 FROM pg_catalog.pg_proc WHERE pronamespace=pg_my_temp_schema())
 AND (SELECT count(*)=0 FROM pg_catalog.pg_class WHERE relnamespace=pg_my_temp_schema() AND relkind='v')
 AND (SELECT count(*)=2 FROM pg_catalog.pg_class WHERE relnamespace=pg_my_temp_schema() AND relkind='r');"
}
check_case() { open_case "$1" "${5:-}"; start_run "$2" "$3" "$4"; run_loop; finish_case; }
close_case() { hic_close main || fail close; }
sum_is() { case " $HIC_SUMMARY " in *" $1=$2 "*) ok "$case_name $1=$2" ;; *) fail "$1=$2: $HIC_SUMMARY" ;; esac; }
relation() {
    local oid
    for oid in $(cat "$HIC_OUT/relation-ids"); do
        IFS='|' read -r -a HIC_R < "$HIC_OUT/relations/$oid/facts" || fail facts
        hic_decode "${HIC_R[3]}" || fail decode
        if [ "$HIC_TEXT" = "$1" ]; then REL_DIR="$HIC_OUT/relations/$oid"; REL_OID=$oid; return; fi
    done
    fail "missing relation $1"
}
field_is() { hic_get "$REL_DIR" "$1"; [ "$HIC_VALUE" = "$2" ] || fail "$1=$HIC_VALUE expected $2"; ok "$case_name $1=$2"; }
cli() {
    local name=$1 want=$2 rc
    shift 2
    env "$@" HIC_PSQL="$HIC_PSQL" HIC_OUT="$out/$name" PGDATABASE="$db" \
        bash "$work/hic12-check.sh" > "$out/$name.log" 2>&1
    rc=$?; case_name=$name
    [ "$rc" = "$want" ] || fail "exit $rc expected $want: $(cat "$out/$name.log")"
    ok "${name}_exit_$want"
}
"$bin/psql" -X -q -v ON_ERROR_STOP=1 -c "SET /* wiki_hic12_suite_timeout */ statement_timeout='60s'" \
    -c "SET /* wiki_hic12_suite_lock */ lock_timeout='2s'" \
    -c "CREATE /* wiki_hic12_suite_database */ DATABASE $db" || fail database
export PGDATABASE="$db"
open_case fixtures
hic_sql main "SELECT /* wiki_hic12_reader_exists */ count(*) FROM pg_catalog.pg_roles WHERE rolname='hic_reader';" || fail lookup
[ "$HIC_VALUE" != 0 ] || hic_sql main 'CREATE /* wiki_hic12_reader */ ROLE hic_reader;' || fail role
hic_request main "$work/fixtures.sql" || fail fixtures
[ "$HIC_FAILED" = false ] || fail "fixtures: $(cat "$(sfile stderr.last)")"
ok fixtures_created
# A child invocation must not adopt or stop this invocation's running server,
# even when it fails before prepare or exits through the EXIT trap.
server_pid=$(head -n 1 "$work/data/postmaster.pid")
for stage in prepare build init cleanup unknown_stage; do
    bash "$work/hic12-measure.sh" "$stage" > "$out/ownership-$stage.log" 2>&1
    rc=$?
    [ "$rc" = 1 ] || fail "pre-existing server: $stage returned $rc"
    [ "$(head -n 1 "$work/data/postmaster.pid")" = "$server_pid" ] || fail "$stage stopped pre-existing server"
    ok "preexisting_server_${stage}_preserved"
done
expect server_still_accepts_queries 'SELECT /* wiki_hic12_server_survived */ true;'
expect_error positional_order_probe 42P10 'SELECT /* wiki_hic12_position_probe */ count(*) FROM (SELECT /* wiki_hic12_position_inner */ 1 FROM ONLY hic_negative.constant_literals ORDER BY ((2)) USING OPERATOR(pg_catalog.<) NULLS LAST) probe;'
expect_error decimal_order_probe 42601 'SELECT /* wiki_hic12_decimal_probe */ count(*) FROM (SELECT /* wiki_hic12_decimal_inner */ 1 FROM ONLY hic_negative.constant_literals ORDER BY ((1.5)) USING OPERATOR(pg_catalog.<) NULLS LAST) probe;'
expect cast_order_probe 'SELECT /* wiki_hic12_cast_probe */ count(*)=2 FROM (SELECT /* wiki_hic12_cast_inner */ 1 FROM ONLY hic_negative.constant_literals ORDER BY CAST((2) AS integer) USING OPERATOR(pg_catalog.<) NULLS LAST) probe;'
expect_error invalid_build 23505 'CREATE /* wiki_hic12_invalid_index */ UNIQUE INDEX CONCURRENTLY invalid_idx ON hic_negative.invalid_index(id);'
expect_error interpolation 42601 'SELECT /* wiki_hic12_interpolation_probe */ (ARRAY[1,2,3])[1:"ERROR"] FROM (SELECT 2 AS "ERROR") s;'
expect spaced_colon 'SELECT /* wiki_hic12_colon_probe */ (ARRAY[1,2,3])[1 : "ERROR"]=ARRAY[1,2] FROM (SELECT 2 AS "ERROR") s;'
close_case
cli cli_positive 0 HIC_SCHEMA=hic_positive HIC_DDL_STABLE=yes
for report in schema relations candidates attempts parents run-errors run; do [ -s "$out/cli_positive/$report.csv" ] || fail "$report"; done
ok cli_reports
cli cli_negative 1 HIC_SCHEMA=hic_negative HIC_DDL_STABLE=yes
cli cli_missing 2 HIC_SCHEMA=hic_missing_schema HIC_DDL_STABLE=yes
grep -F -q 'initialization' "$out/cli_missing/run-errors.csv" || fail missing_error
ok missing_error_recorded
cli cli_unattested 64 HIC_SCHEMA=hic_positive
[ ! -e "$out/cli_unattested" ] || fail unexpected_output
ok no_connect_before_attestation
cli cli_bad_option 64 HIC_SCHEMA=hic_positive HIC_DDL_STABLE=yes HIC_EXHAUSTIVE=maybe
check_case multiline hic_multiline false false hic_reader
sum_is physical_relations 4
sum_is passed 4
sum_is checked_indexes 4
sum_is attempt_errors 0
for name in case_key column_key collation_key literal_key; do
    relation "$name"; field_is status PASS; field_is heap 4
done
close_case
check_case am_names hic_am_names false false
sum_is physical_relations 1
sum_is no_eligible_index 1
sum_is error 0
relation t
for iid in $(cat "$REL_DIR/index-ids"); do
    hic_read_index < "$HIC_OUT/indexes/$iid/facts" || fail am_facts
    [ -z "$i_extra" ] && hic_decode "$i_amname" || fail am_name_decode
    case $HIC_TEXT in 'pipe|am'|$'line\nam') ;; *) fail "unexpected AM name $HIC_TEXT";; esac
    hic_get "$HIC_OUT/indexes/$iid" reason
    [ "$HIC_VALUE" = unsupported_index_am ] || fail "AM reason $HIC_VALUE"
    ok am_name_preserved_and_classified
done
close_case
check_case positive hic_positive false false hic_reader
sum_is physical_relations 13
sum_is passed 13
sum_is checked_indexes 13
sum_is attempt_errors 0
for name in empty_table empty_mv; do relation "$name"; field_is status PASS; field_is heap 0; done
relation nulls_duplicates; field_is heap 5
relation inh_parent; field_is heap 1
relation inh_child; field_is heap 2
relation unpopulated_mv; field_is status NOT_APPLICABLE
relation 'Odd table "x"'; field_is status PASS
close_case
check_case external hic_positive true false
sum_is physical_relations 14
sum_is passed 14
relation partition_outside; field_is heap 1
if grep -q ',"true","true",' "$HIC_OUT/parents.csv"; then fail foreign_child_unsupported; fi
ok partition_summary_retains_gap
close_case
check_case negative hic_negative false false
sum_is physical_relations 8
sum_is no_eligible_index 4
sum_is inconclusive 4
sum_is error 0
relation constant_literals; field_is status INCONCLUSIVE
for iid in $(cat "$REL_DIR/index-ids"); do grep -q 'CAST(' "$HIC_OUT/indexes/$iid/order" || fail literal_cast; done
ok literal_casts_preserved
relation user_expression; field_is status INCONCLUSIVE
relation domain_table; field_is status INCONCLUSIVE
close_case
check_case choice hic_choice false true
sum_is passed 1
sum_is eligible_indexes 2
sum_is checked_indexes 1
sum_is untested_indexes 1
close_case
for schema in hic_rls hic_denied hic_namespace_denied; do
    check_case "$schema" "$schema" false false hic_reader
    sum_is error 1
    close_case
done
check_case bypass hic_rls false false
sum_is passed 1
close_case
check_case empty hic_empty_scope false false
sum_is schema_success false
sum_is physical_relations 0
close_case
check_case guard hic_psql_guard false false
sum_is error 1
sum_is attempt_errors 2
close_case
# Same-snapshot concurrent HOT/non-HOT writes and aborted writes.
open_case concurrent
start_run hic_concurrent false false
HIC_OUT_MAIN=$HIC_OUT
mkdir -m 700 "$HIC_OUT/session-writer"
# psql -f is a disposable fixture writer, independent of the checking session.
"$bin/psql" -X -q -v ON_ERROR_STOP=1 -f "$work/writer.sql" > "$HIC_OUT/writer.log" 2>&1 || fail writer
run_loop; finish_case
relation t; field_is heap 100
sum_is passed 1
close_case
# Old horizon versus a new checking transaction; flag may remain true.
open_case gate
hic_open old 5 6 || fail old_connection
hic_sql old "BEGIN /* wiki_hic12_old_begin */; SET /* wiki_hic12_old_timeout */ LOCAL statement_timeout='60s'; SET /* wiki_hic12_old_lock */ LOCAL lock_timeout='2s'; SELECT /* wiki_hic12_old_xid */ txid_current();" || fail old_xid
"$bin/psql" -X -q -v ON_ERROR_STOP=1 -f "$work/gate-setup.sql" > "$HIC_OUT/gate.log" 2>&1 || fail gate_setup
start_run hic_gate false false
run_loop; finish_case
sum_is inconclusive 1
hic_sql old 'ROLLBACK /* wiki_hic12_old_end */;' || fail old_end
hic_close old || fail old_close
close_case
check_case new_horizon hic_gate false false
sum_is passed 1
close_case
open_case validator
hic_install || fail install
hic_request main "$work/validator-tests.sql" || fail units
[ "$HIC_FAILED" = false ] || fail "unit setup: $(cat "$(sfile stderr.last)")"
hic_data "$HIC_OUT/unit-list" "SELECT /* wiki_hic12_unit_list */ 'x'||encode(convert_to(label,'UTF8'),'hex'),phase,expected FROM hic_unit_cases ORDER BY label;" || fail list
HIC_ATTEMPT_ID=1
# Hex-encode ASCII text the way the runner's SQL transport does.
ascii_hex() {
    local s=$1 c n
    HEX=x
    while [ -n "$s" ]; do
        c=${s:0:1} s=${s:1}
        printf -v n '%02x' "'$c"
        HEX=$HEX$n
    done
}
ascii_hex pg_temp_test; HIC_CAPTURE_SCHEMA=$HEX
ascii_hex fixture; HIC_REL_SCHEMA=$HEX
ascii_hex target; HIC_REL_NAME=$HEX
while IFS='|' read -r label HIC_PHASE expected; do
    hic_decode "$label" || fail unit_label
    label=$HIC_TEXT; hic_literal "$label"
    hic_sql main "DELETE /* wiki_hic12_unit_clear */ FROM pg_temp.hic_plan;
INSERT /* wiki_hic12_unit_stage */ INTO pg_temp.hic_plan SELECT 1,document FROM hic_unit_cases WHERE label=$HIC_LITERAL;" || fail stage
    hic_plan_facts 1 "$HIC_OUT/unit-facts" || fail decode
    hic_validate "$HIC_OUT/unit-facts" || fail validate
    case $expected:$HIC_ACCEPTED in t:true|f:false) ok "$label" ;; *) fail "$label expected $expected accepted $HIC_ACCEPTED reasons=$HIC_REASONS" ;; esac
done < "$HIC_OUT/unit-list"
close_case
# Pure evaluator cases: these synthetic files do not represent corruption.
open_case evaluator
start_run hic_choice false true
relation t
ids=(); while IFS= read -r iid; do ids+=("$iid"); done < "$REL_DIR/index-ids"
synth() {
    local id=$1 phase=$2 state=$3 value=$4 intended=$5 actual=$6 base="$HIC_OUT/attempts/$1"
    mkdir -p "$base" || fail synth
    printf '%s\n' "$id" >> "$REL_DIR/attempt-ids"
    hic_put "$base" phase "$phase"; hic_put "$base" status "$state"; hic_put "$base" count "$value"
    hic_put "$base" intended "$intended"; hic_put "$base" actual "$actual"
    hic_put "$base" sqlstate "${7:-}"; hic_put "$base" reasons "${8:-}"; hic_put "$base" unresolved ''
}
eval_is() { hic_evaluate "$REL_OID" || fail evaluator; [ "$HIC_STATUS" = "$1" ] || fail "evaluator $HIC_STATUS expected $1"; ok "evaluator $1 $2"; }
: > "$REL_DIR/attempt-ids"
synth 1 heap accepted 10 '' ''
synth 2 index accepted 9 "${ids[0]}" "${ids[0]}"
eval_is INCONCLUSIVE one_unchecked
synth 3 index accepted 8 "${ids[1]}" "${ids[1]}"
eval_is MISMATCH complete_coverage
synth 4 index error '' "${ids[1]}" ''
eval_is MISMATCH recovered_error
synth 5 index accepted 10 "${ids[0]}" "${ids[0]}"
eval_is PASS_WITH_ANOMALY matching_actual_index_after_mismatches
HIC_EVIDENCE_VALID=false; eval_is ERROR invalid_evidence; HIC_EVIDENCE_VALID=true
synth 6 heap accepted 11 '' ''
eval_is ERROR conflicting_baselines
: > "$REL_DIR/attempt-ids"
synth 7 heap accepted 9223372036854775807 '' ''
synth 8 index accepted 9223372036854775807 "${ids[0]}" "${ids[0]}"
eval_is PASS exact_bigint
: > "$REL_DIR/attempt-ids"
synth 9 heap accepted 10 '' ''
synth 10 index error '' "${ids[1]}" ''
eval_is ERROR uncovered_execution_error
synth 11 index accepted 10 "${ids[0]}" "${ids[0]}"
eval_is PASS optional_error_after_match
synth 12 index error '' "${ids[1]}" '' XX001
eval_is PASS_WITH_ANOMALY storage_error_after_match
: > "$REL_DIR/attempt-ids"
synth 13 heap error '' '' '' 01000
synth 14 heap accepted 10 '' ''
synth 15 index accepted 10 "${ids[0]}" "${ids[0]}"
eval_is PASS_WITH_ANOMALY retry_after_warning
: > "$REL_DIR/attempt-ids"
synth 16 heap accepted 10 '' ''
synth 17 index rejected 10 "${ids[0]}" "${ids[0]}" '' actual_index_identity_changed
eval_is ERROR index_identity_changed
: > "$REL_DIR/attempt-ids"
synth 18 heap accepted 10 '' ''
synth 19 index accepted 10 "${ids[0]}" "${ids[0]}"
hic_put "$REL_DIR" drift storage_1; eval_is ERROR relation_drift; hic_put "$REL_DIR" drift ''
eval_is PASS drift_cleared
hic_sql main 'ROLLBACK /* wiki_hic12_synthetic_end */;' || fail rollback
close_case
# Truncated JSON and count rollback: retry in the same snapshot.
open_case retry
start_run hic_concurrent false false
relation t
HIC_RELATION=$REL_OID; HIC_REL_SCHEMA=${HIC_R[2]}; HIC_REL_NAME=${HIC_R[3]}; HIC_REL_QUALIFIED=${HIC_R[4]}
HIC_PHASE=heap; HIC_INTENDED=; HIC_VARIANT=1
hic_prepare_attempt || fail prelude
hic_request main "$HIC_A/count.sql" || fail execute
[ "$HIC_FAILED" = false ] || fail execute_error
head -n 3 "$(sfile response)" > "$HIC_OUT/plans/attempt-$HIC_ATTEMPT_ID.json"
hic_profiles control || fail profile
hic_attempt_import && fail accepted_truncated_json
hic_parse_error main || fail parse
[ "$HIC_ERR_CODE" = 22P02 ] && [ -n "$HIC_ERR_DETAIL" ] || fail truncated_diagnostic
ok truncated_json_rejected
hic_recover || fail recover
expect rolled_back_count 'SELECT /* wiki_hic12_rollback_assert */ count(*)=0 FROM pg_temp.hic_count;'
# Keep failed evidence locally after rollback, as the normal path does.
for spec in 'status:error' 'count:' 'actual:' 'reasons:' 'stage:capture' 'sqlstate:22P02' 'message:truncated' 'detail:expected more JSON' 'hint:' 'execution_ms:' 'captured:' 'tids_returned:' 'unresolved:'; do
    hic_put "$HIC_A" "${spec%%:*}" "${spec#*:}" || fail error_record
done
run_loop; finish_case
sum_is passed 1
sum_is attempt_errors 1
close_case
# Held lock: two recoverable statement errors cannot become an empty pass.
open_case locked
start_run hic_lock false false
hic_open holder 5 6 || fail holder
hic_sql holder "BEGIN /* wiki_hic12_holder_begin */; SET /* wiki_hic12_holder_timeout */ LOCAL statement_timeout='60s'; SET /* wiki_hic12_holder_lock */ LOCAL lock_timeout='2s'; LOCK /* wiki_hic12_holder_lock */ TABLE hic_lock.t IN ACCESS EXCLUSIVE MODE;" || fail hold
run_loop; finish_case
sum_is error 1
sum_is attempt_errors 2
grep -F -q '55P03' "$HIC_OUT/attempts.csv" || fail lock_code
ok lock_diagnostic
hic_sql holder 'ROLLBACK /* wiki_hic12_holder_end */;' || fail unlock
hic_close holder || fail holder_close
close_case
# Termination between requests: no reconnection to a different snapshot.
open_case disconnect
start_run hic_lock false false
pid=$HIC_BACKEND
hic_open killer 5 6 || fail killer
hic_sql killer "SELECT /* wiki_hic12_terminate_fixture */ pg_terminate_backend($pid);" || fail terminate
hic_sql main 'SELECT /* wiki_hic12_lost_probe */ 1;'; rc=$?
[ "$rc" = 2 ] || fail "disconnect returned $rc"
ok lost_session_is_incomplete
hic_close main || :
hic_close killer || fail killer_close
# Report write failure remains incomplete after COMMIT.
open_case report_failure
start_run hic_lock false false
run_loop
mkdir "$HIC_OUT/schema.csv" || fail fault_fixture
hic_finish; rc=$?
[ "$rc" = 2 ] || fail "report fault returned $rc"
ok report_write_error
close_case
# Transport and CSV preserve separators, quotes, newlines, slashes and UTF-8.
open_case encoding
hic_sql main "SELECT /* wiki_hic12_hex_probe */ 'x'||encode(convert_to(E'line\\npipe|tab\\tquote\"slash\\\\é','UTF8'),'hex');" || fail hex
hic_decode "$HIC_VALUE" || fail decode
[ "$HIC_TEXT" = $'line\npipe|tab\tquote"slash\\é' ] || fail hex_roundtrip
ok hex_roundtrip
hic_csv "$HIC_OUT/escaped.csv" "$HIC_TEXT" 'last' || fail csv
printf '%s\n' $'"line\npipe|tab\tquote""slash\\é","last"' > "$HIC_OUT/expected.csv"
cmp "$HIC_OUT/escaped.csv" "$HIC_OUT/expected.csv" || fail csv_roundtrip
ok csv_roundtrip
# Local evidence must also preserve empty values and trailing newlines.
for local_value in '' $'first\nsecond\n\n' '.'; do
    hic_put "$HIC_OUT" local_value "$local_value" || fail local_put
    hic_get "$HIC_OUT" local_value || fail local_get
    [ "$HIC_VALUE" = "$local_value" ] || fail local_roundtrip
    loaded=$local_value
    hic_save "$HIC_OUT" loaded || fail local_save
    loaded=discarded
    hic_load "$HIC_OUT" loaded || fail local_load
    [ "$loaded" = "$local_value" ] || fail saved_roundtrip
    ok local_value_roundtrip
done
# A value ending in the dollar-quote tag minus its final $ gets another tag.
hic_literal 'tail$hic12_q0'
case $HIC_LITERAL in '$hic12_q1$'*) ok literal_suffix_tag ;; *) fail "literal tag $HIC_LITERAL" ;; esac
expect literal_suffix_roundtrip "SELECT /* wiki_hic12_literal_probe */ $HIC_LITERAL = 'tail\$hic12_q0';"
# A warning-only report is found by the warning pattern and never as an error.
printf '%s\n' 'WARNING:  01000: page verification failed, calculated checksum 1 but expected 2' > "$(sfile stderr.last)"
if hic_parse_error main; then fail warning_read_as_error; fi
hic_parse_error main "$HIC_WARNING_LINE" && [ "$HIC_ERR_CODE" = 01000 ] || fail warning_parse
[[ $HIC_ERR_CODE =~ $HIC_STORAGE_CODE ]] || fail warning_storage_class
ok warning_report_classified
close_case
# Settings that hide damage or bias counts are refused.
hic_server_guard off off on -1 || fail guard_default
ok server_guard_default
for args in 'on off on -1' 'off on on -1' 'off off off -1' 'off off on 60'; do
    set -- $args
    if hic_server_guard "$@"; then fail "guard accepted $args"; fi
    ok "server_guard_refuses_${args// /_}"
done
for setting in zero_damaged_pages=on ignore_checksum_failure=on track_counts=off; do
    cli "cli_refuse_${setting%%=*}" 2 HIC_SCHEMA=hic_positive HIC_DDL_STABLE=yes PGOPTIONS="-c $setting"
    grep -F -q "refusing server settings: $setting" "$out/cli_refuse_${setting%%=*}/summary.txt" || fail "reason $setting"
    ok "refusal_recorded_${setting%%=*}"
done
# A "?column?" key column, and index entries whose rows were deleted.
check_case edge hic_edge false false
sum_is passed 2
sum_is unresolved_index_tids 10
relation output_name; field_is status PASS; field_is heap 3
relation deleted; field_is status PASS; field_is heap 90; field_is unresolved_index_tids 10
close_case
# The first scan marked those ten entries LP_DEAD, so a later scan skips them.
check_case edge_again hic_edge false false
relation deleted; field_is status PASS; field_is unresolved_index_tids 0
close_case
# A role-level idle-in-transaction timeout must not end the checking session.
CASE_OPTIONS='-c idle_in_transaction_session_timeout=1000'
open_case idle
CASE_OPTIONS=
start_run hic_lock false false
sleep 2
run_loop; finish_case
sum_is passed 1
close_case
# DDL committed after the snapshot, before the checker locks each relation.
open_case drift
start_run hic_drift false false
"$bin/psql" -X -q -v ON_ERROR_STOP=1 -f "$work/drift.sql" > "$HIC_OUT/drift.log" 2>&1 || fail drift_ddl
run_loop; finish_case
sum_is physical_relations 5
sum_is error 4
sum_is passed 1
for name in truncated refreshed recreated swapped; do
    relation "$name"; field_is status ERROR
    hic_get "$REL_DIR" drift || fail drift_read
    [ -n "$HIC_VALUE" ] || fail "no drift for $name"
    ok "drift_recorded_$name"
done
relation truncated; hic_get "$REL_DIR" drift
case $HIC_VALUE in *storage_*) ok truncate_storage_drift ;; *) fail "truncate drift $HIC_VALUE" ;; esac
relation swapped; hic_get "$REL_DIR" drift
case $HIC_VALUE in *name_*) ok swap_name_drift ;; *) fail "swap drift $HIC_VALUE" ;; esac
relation stable; field_is status PASS
close_case
# A lost evidence file makes reporting fail instead of dropping a relation.
open_case evidence_read
start_run hic_lock false false
run_loop
relation t
rm "$REL_DIR/scope" && mkdir "$REL_DIR/scope" || fail evidence_fixture
hic_finish; rc=$?
[ "$rc" = 2 ] || fail "unreadable evidence returned $rc"
ok unreadable_evidence_incomplete
close_case
# List-file failures must not become an empty inventory or a harmless
# partition-subtree gap. Read errors include a missing file and a directory
# in place of a file; the scalar evidence test above covers neither case.
open_case evidence_lists
start_run hic_positive false false
run_loop
relation nulls_duplicates
for list in "$HIC_OUT/relation-ids" "$HIC_OUT/partitions.raw" "$REL_DIR/index-ids" "$REL_DIR/attempt-ids"; do
    mv "$list" "$list.saved" || fail list_fixture
    hic_reports; rc=$?
    [ "$rc" = 2 ] || fail "missing list $list returned $rc"
    ok "missing_${list##*/}_incomplete"
    mkdir "$list" || fail list_directory
    hic_reports; rc=$?
    [ "$rc" = 2 ] || fail "unreadable list $list returned $rc"
    ok "unreadable_${list##*/}_incomplete"
    rmdir "$list" && mv "$list.saved" "$list" || fail restore_list
done
hic_reports || fail restored_reports
sum_is schema_success true
rm "$HIC_OUT/partitions.raw" || fail missing_partitions
hic_finish; rc=$?
[ "$rc" = 2 ] || fail "missing partitions finish returned $rc"
ok missing_partitions_finish_incomplete
close_case
# Quoted names with psql's :' form and a trailing colon are counted: psql
# never substitutes inside a quoted identifier, so only ordering text is guarded.
check_case guard_names hic_guard_names false false
sum_is passed 1
relation "odd:'name:"; field_is status PASS; field_is heap 3
close_case
# A role- or database-level client_min_messages = error keeps warnings from
# psql; the runner pins the default, notice, so a warning still arrives.
PGOPTIONS='-c client_min_messages=error' "$bin/psql" -X -q -v ON_ERROR_STOP=1 -v VERBOSITY=verbose \
    -c "SET /* wiki_hic12_hidden_timeout */ statement_timeout = '60s'" \
    -c 'ROLLBACK /* wiki_hic12_hidden_warning */' > "$out/hidden-warning.log" 2>&1 || fail hidden_warning_probe
[ ! -s "$out/hidden-warning.log" ] || fail "warning not hidden: $(cat "$out/hidden-warning.log")"
ok role_setting_hides_warning
CASE_OPTIONS='-c client_min_messages=error'
open_case messages
CASE_OPTIONS=
hic_install || fail install
expect messages_pinned "SELECT /* wiki_hic12_messages_probe */ current_setting('client_min_messages') = 'notice';"
hic_sql main 'ROLLBACK /* wiki_hic12_warning_probe */;'
grep -E -q '^[^ ]+:  25P01: ' "$(sfile stderr.last)" || fail warning_hidden
ok warning_reaches_runner
close_case
# A reload during the run that turns zero_damaged_pages on reaches the
# checking session between statements; the next count records it, and the
# runner stops instead of accepting a count taken under it.
open_case midrun
start_run hic_lock false false
"$bin/psql" -X -q -v ON_ERROR_STOP=1 -c "SET /* wiki_hic12_midrun_timeout */ statement_timeout = '60s'" \
    -c "ALTER /* wiki_hic12_midrun_on */ SYSTEM SET zero_damaged_pages = on" \
    -c "SELECT /* wiki_hic12_midrun_reload */ pg_reload_conf()" > "$HIC_OUT/reload-on.log" 2>&1 || fail reload_on
seen=no
for try in 1 2 3 4 5 6 7 8 9 10; do
    hic_sql main "SELECT /* wiki_hic12_midrun_seen */ current_setting('zero_damaged_pages');" || fail midrun_probe
    if [ "$HIC_VALUE" = on ]; then seen=yes; break; fi
    sleep 1
done
hic_attempt_loop; rc=$?
"$bin/psql" -X -q -v ON_ERROR_STOP=1 -c "SET /* wiki_hic12_midrun_off_timeout */ statement_timeout = '60s'" \
    -c "ALTER /* wiki_hic12_midrun_off */ SYSTEM RESET zero_damaged_pages" \
    -c "SELECT /* wiki_hic12_midrun_reload_off */ pg_reload_conf()" > "$HIC_OUT/reload-off.log" 2>&1 || fail reload_off
[ "$seen" = yes ] || fail midrun_not_seen
ok midrun_setting_reached_session
[ "$rc" = 2 ] || fail "midrun loop returned $rc"
case $HIC_STOP in 'refusing server settings: zero_damaged_pages=on'*) ok midrun_refused ;; *) fail "midrun stop $HIC_STOP" ;; esac
hic_sql main 'ROLLBACK /* wiki_hic12_midrun_end */;' || :
close_case
now=
for try in 1 2 3 4 5 6 7 8 9 10; do
    now=$("$bin/psql" -X -qAt -v ON_ERROR_STOP=1 -c "SET /* wiki_hic12_midrun_reset_timeout */ statement_timeout = '60s'" \
        -c "SHOW /* wiki_hic12_midrun_reset_seen */ zero_damaged_pages") || fail midrun_reset_probe
    [ "$now" = off ] && break
    sleep 1
done
[ "$now" = off ] || fail "midrun reset left $now"
ok midrun_setting_reset
# Synthetic report input: an accepted index count whose TID counters were
# unavailable is counted apart, never added to the total as zero.
open_case unavailable
start_run hic_lock false false
run_loop
relation t
for aid in $(cat "$REL_DIR/attempt-ids"); do
    hic_get "$HIC_OUT/attempts/$aid" phase || fail unavailable_read
    if [ "$HIC_VALUE" = index ]; then hic_put "$HIC_OUT/attempts/$aid" unresolved '' || fail unavailable_write; fi
done
finish_case
sum_is unresolved_index_tids 0
sum_is unresolved_tids_unavailable 1
close_case
# Mapped catalogs keep relfilenode 0 in pg_class; the relation mapper holds
# their file numbers, so the identity check must not report drift for them.
check_case catalog pg_catalog false false
printf '%s\n' "$HIC_SUMMARY" > "$out/catalog-summary.txt"
for name in pg_class pg_attribute pg_proc pg_type pg_database; do
    relation "$name"; field_is status PASS; field_is drift ''
done
close_case
# Physical fault leg: a fast stop writes every dirty buffer, so no later write
# can cover the injected bytes, and the restart empties shared buffers, so the
# first read of each damaged block comes from the file and is verified. Block
# 1 of the first table becomes all zeros; block 1 of the second gets an
# invalid header (all 0xFF).
case_name=damage
damage_sql() {
    "$bin/psql" -X -qAt -v ON_ERROR_STOP=1 -c "SET /* wiki_hic12_damage_timeout */ statement_timeout = '60s'" \
        -c "SET /* wiki_hic12_damage_lock */ lock_timeout = '2s'" -c "$1"
}
# Cached clean copy first: after VACUUM and a checkpoint, block 1 of the third
# table is clean in shared buffers. Zeroing its file block under the running
# server changes nothing the scans read, because a buffer hit is not verified.
damage_sql "VACUUM /* wiki_hic12_cached_vacuum */ (FREEZE) hic_damage_cached.t" > /dev/null || fail cached_vacuum
damage_sql "CHECKPOINT /* wiki_hic12_cached_checkpoint */" > /dev/null || fail cached_checkpoint
cached_lost=$(damage_sql "SELECT /* wiki_hic12_cached_lost */ count(*) FROM hic_damage_cached.t WHERE (ctid::text::point)[0] = 1") || fail cached_lost
cached_path=$(damage_sql "SELECT /* wiki_hic12_cached_path */ pg_relation_filepath('hic_damage_cached.t')") || fail cached_path
hic_uint "$cached_lost" && [ "$cached_lost" -gt 0 ] || fail "cached layout lost=$cached_lost"
dd if=/dev/zero of="$work/data/$cached_path" bs=8192 seek=1 count=1 conv=notrunc 2> "$out/dd-cached.log" || fail dd_cached
check_case damage_cached hic_damage_cached false false
sum_is passed 1
relation t; field_is status PASS; field_is heap 300; field_is unresolved_index_tids 0
close_case
case_name=damage
lost=$(damage_sql "SELECT /* wiki_hic12_damage_lost */ count(*) FROM hic_damage_zero.t WHERE (ctid::text::point)[0] = 1") || fail damage_lost
blocks=$(damage_sql "SELECT /* wiki_hic12_damage_blocks */ pg_relation_size('hic_damage_zero.t') / 8192") || fail damage_blocks
zero_path=$(damage_sql "SELECT /* wiki_hic12_damage_zero_path */ pg_relation_filepath('hic_damage_zero.t')") || fail zero_path
header_path=$(damage_sql "SELECT /* wiki_hic12_damage_header_path */ pg_relation_filepath('hic_damage_header.t')") || fail header_path
hic_uint "$lost" && [ "$lost" -gt 0 ] && [ "$blocks" -ge 3 ] || fail "damage layout lost=$lost blocks=$blocks"
printf 'lost_rows=%s heap_blocks=%s cached_lost_rows=%s\n' "$lost" "$blocks" "$cached_lost" > "$out/damage-layout.txt"
"$bin/pg_ctl" -D "$work/data" -m fast -w stop > "$out/damage-stop.log" 2>&1 || fail damage_stop
dd if=/dev/zero of="$work/data/$zero_path" bs=8192 seek=1 count=1 conv=notrunc 2> "$out/dd-zero.log" || fail dd_zero
dd if=/dev/zero bs=8192 count=1 2> "$out/dd-ff.log" | tr '\000' '\377' > "$out/ff.blk" || fail ff_block
[ "$(($(wc -c < "$out/ff.blk")))" = 8192 ] || fail ff_size
dd if="$out/ff.blk" of="$work/data/$header_path" bs=8192 seek=1 count=1 conv=notrunc 2> "$out/dd-header.log" || fail dd_header
"$bin/pg_ctl" -D "$work/data" -l "$work/postgres.log" \
    -o "-p 55447 -c listen_addresses='' -c unix_socket_directories='$work/socket' -c unix_socket_permissions=0700" \
    -w start > "$out/damage-start.log" 2>&1 || fail damage_start
ok damage_injected
# Zeroed block: both scans lose its rows, so the counts agree (PASS); only the
# unresolved-TID counter shows the index entries whose rows are gone.
check_case damage_zero hic_damage_zero false false
sum_is passed 1
relation t; field_is status PASS; field_is heap "$((300 - lost))"; field_is unresolved_index_tids "$lost"
close_case
# Invalid header: every heap read fails with XX001, so there is no baseline.
check_case damage_header hic_damage_header false false
sum_is error 1
relation t; field_is status ERROR; field_is storage_errors 2
grep -F -q '"XX001"' "$HIC_OUT/attempts.csv" || fail damage_sqlstate
ok damage_header_xx001
close_case
# After the restart the cached copy is gone: the same block now comes from the
# file, and its rows drop out of both counts.
check_case damage_cached_restarted hic_damage_cached false false
relation t; field_is status PASS; field_is heap "$((300 - cached_lost))"; field_is unresolved_index_tids "$cached_lost"
close_case
cli cli_damage_zero_damaged_pages 2 HIC_SCHEMA=hic_damage_header HIC_DDL_STABLE=yes PGOPTIONS='-c zero_damaged_pages=on'

printf '%s\n' "$out" > "$work/last-test-output"
printf 'passed=%s\n' "$passed"
HIC_TEST_SH
}

if [ "$#" -eq 0 ]; then set -- prepare build init verify cleanup; fi
for stage in "$@"; do
    started=$SECONDS
    case "$stage" in
        prepare|build|init|verify|cleanup) "$stage";;
        *) printf 'Unknown stage: %s\n' "$stage" >&2; exit 1;;
    esac
    printf 'stage=%s elapsed_seconds=%s\n' "$stage" "$((SECONDS - started))"
done
```

## Context Reviewed

- PostgreSQL 12.2 exact pin; only matching implementation, same-checkout SGML, catalogs and regression/isolation tests support engine claims.
- Catalogs and caller/callee boundaries: namespace/class/index/AM/inheritance, index key/operator-family metadata and dependencies, deparsing and type output, materialized-view rewriting/population, permission checks and RLS.
- Planner/executor: `get_relation_info()`, relcache xmin, path ordering and costs, constant pathkeys, subquery handling, `ExplainOnePlan()`, count transition, ordinary versus index-only scans, table-AM fetching and HOT chains.
- State lifecycle: first repeatable-read snapshot, temporary DML/read-only DDL boundary, savepoint rollback/release, final context and connection shutdown. Explicit DDL coordination remains an operator premise.
- Client protocol: psql noninteractive stdin, flush/error/result-variable behavior, dollar-quote and interpolation lexers, verbose libpq reports, JSON parsing and typed expansion, UTF-8/hex conversion.
- Build/generated-header boundary: SQL uses existing catalogs/functions; no C module or generated header is added. Catalog `.h`/`.dat` files feed generated catalog/bootstrap headers. pageinspect remains a separate optional contrib module, unused by the checker. [catalog/Makefile#catalog-inputs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L55-L69) [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L101) [pageinspect/Makefile#module-and-tests](../../../../raw/postgres-12/contrib/pageinspect/Makefile#L1-L15)
- Glossary: every linked term entry carries a PostgreSQL 12 qualification. The second 2026-10-24 review added first-use links to SQLSTATE, libpq, Dollar quoting, VACUUM, Equivalence class, Pathkey, Syscache, Buffer manager, Cumulative statistics and Relation mapper, and gave Equivalence class and Pathkey their PostgreSQL 12 notes.
- PostgreSQL 12 has no matching common-concept pages; no concept page was created or edited.
- Contradiction comparison: assembled explanation, embedded runner and harness, manifests/reports/tests, matching glossary and own navigation entries; same-version concurrent-index, invalid-index, partitioned-index, EXPLAIN, physical-index-statistics and health/tail-check pages searched for shared claims. The earlier SQL-owned architecture is replaced consistently rather than retained as a second implementation.
- 2026-10-24 false-negative review and fixes: index heap fetch of TIDs that reach no row, killed-tuple hints, per-transaction statistics counters and their flush point, `zero_damaged_pages`, `ignore_checksum_failure` and `old_snapshot_threshold` paths, default SQLSTATEs of warnings, syscache-based name and relfilenode lookup, lock levels of index rename and `REINDEX CONCURRENTLY`, subtransaction lock release, `REFRESH` frozen inserts and `ORDER BY` output-name matching.
- Verification: the single published harness exercises the runner and its fixtures. The run record is under Measurement Script above and teardown is in the task log; no application cluster, raw source edit, network fetch, subagent, commit or push is part of this work.
- 2026-10-24 second review and fixes: buffer hits versus verified reads, the relation mapper behind a 0 `relfilenode`, configuration reload between statements, warning delivery under `client_min_messages`, psql's quoted-identifier state and `quote_identifier()`, the shutdown checkpoint, and the C99 language mode that a GCC 15 build of this pin needs.
- 2026-10-24 third review and fixes: complete evidence-list reads and checked membership/count operations in verdicts and reports; missing-file and directory-substitution regressions; `_bt_killitems()` abandoning hints after a page LSN change; and statistics-socket startup failure forcing `track_counts` off. The hint race and socket-startup explanation are source-derived, not new PostgreSQL measurement cases.

## Evidence Map

| Claim or design dependency | Primary implementation | Official documentation / supporting tests |
|---|---|---|
| Physical relation, population and index inventory | [pg_class.h](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L104-L192), [pg_index.h](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L57), [pg_am.dat](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L35) | [catalogs.sgml#index-flags](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3822-L3865), [matview.sql](../../../../raw/postgres-12/src/test/regress/sql/matview.sql#L14-L40) |
| ONLY and stored materialized-view data | [rewriteHandler.c](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L1900-L1911) | [select.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L355-L365), [rules.sgml](../../../../raw/postgres-12/doc/src/sgml/rules.sgml#L898-L929) |
| Snapshot index usability | [plancat.c](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L199-L234), [transam.c](../../../../raw/postgres-12/src/backend/access/transam/transam.c#L296-L312) | [catalogs.sgml#indcheckxmin](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3835-L3844) |
| Full B-tree path is possible but not guaranteed | [indxpath.c](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050), [nbtsearch.c](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L997) | [indices.sgml](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L480-L542) |
| Expressions, key order, opfamily and collation | [ruleutils.c](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1331-L1399), [pathkeys.c](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L487-L522), [parse_clause.c](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L3263-L3295) | [create_index.sql](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L700-L713) |
| One retained snapshot; temporary writes permitted | [snapmgr.c](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370), [execMain.c](../../../../raw/postgres-12/src/backend/executor/execMain.c#L769-L795) | [set_transaction.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L116-L139) |
| RLS off errors rather than granting bypass | [rls.c](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132) | [config.sgml](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7395-L7405), [rowsecurity.sql](../../../../raw/postgres-12/src/test/regress/sql/rowsecurity.sql#L1511-L1542) |
| Same-execution count/plan capture | [explain.c](../../../../raw/postgres-12/src/backend/commands/explain.c#L494-L541) | [explain.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L87-L105) |
| Preserve the utility result and resolve its actual index name | [fe-exec.c#last-result](../../../../raw/postgres-12/src/interfaces/libpq/fe-exec.c#L2096-L2141), [explain.c#quoted-index](../../../../raw/postgres-12/src/backend/commands/explain.c#L2844-L2862) | The published suite checks quoted identifiers and Bash reports after COMMIT |
| A bare numeric-literal key needs a same-type `CAST` in `ORDER BY` | [gram.y](../../../../raw/postgres-12/src/backend/parser/gram.y#L13464-L13497), [analyze.c](../../../../raw/postgres-12/src/backend/parser/analyze.c#L1252-L1256), [parse_clause.c](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1931-L1966), [ruleutils.c](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9554-L9585), [parse_coerce.c](../../../../raw/postgres-12/src/backend/parser/parse_coerce.c#L156-L169) | Parser probe and literal-key fixture in the published measurement script |
| Inspection itself can invoke a type-output function | [ruleutils.c#get_const_expr](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9522-L9550) | User-function/domain fixtures retain untested candidates rather than silently excluding them |
| Nonconcurrent build's broken-HOT-chain usability gate | [index.c#index_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L2921-L2971), [plancat.c#horizon](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234) | Old-holder/new-transaction fixture in the published measurement script |
| SQL orchestration limitation | [xml.c](../../../../raw/postgres-12/src/backend/utils/adt/xml.c#L2671-L2689), [utility.c](../../../../raw/postgres-12/src/backend/tcop/utility.c#L99-L118) | [execute.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50), [psql-ref.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L2059-L2094) |
| Heap visibility and HOT resolution | [indexam.c](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635), [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1559-L1637) | [README.HOT](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L48-L122) |
| Physical inspection is not visible-row counting | [rawpage.c](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L159-L169), [heapfuncs.c](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L180-L224) | [pageinspect.sgml](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L169-L175) |
| Runner request/reply protocol over psql's standard input | [mainloop.c#flush-and-send](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L147-L440), [command.c#echo-and-flush](../../../../raw/postgres-12/src/bin/psql/command.c#L1116-L1152), [common.c#result-variables](../../../../raw/postgres-12/src/bin/psql/common.c#L544-L584) | [psql-ref.sgml#echo](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L1860-L1873); the current published suite |
| Runner error classification and parsing | [fe-protocol3.c#verbose-message](../../../../raw/postgres-12/src/interfaces/libpq/fe-protocol3.c#L1017-L1095), [logging.c#locus-prefix](../../../../raw/postgres-12/src/common/logging.c#L164-L179), [postgres.c#aborted-transaction-filter](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L1091-L1105) | [errcodes.txt#classes](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L77-L99); measured RLS, lock-timeout, truncated-plan and missing-schema cases |
| Dollar-quoted values and the interpolation guard | [psqlscan.l#quotes-and-variables](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L551-L766), [scan.l#dolqdelim](../../../../raw/postgres-12/src/backend/parser/scan.l#L292-L294), [ruleutils.c#printSubscripts](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10614-L10637) | [syntax.sgml#dollar-quoting](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L563-L621), [psql-ref.sgml#interpolation](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L4020-L4061); measured slice probe and guard fixture |
| Recovery only while the attempt savepoint exists | [xact.c#savepoint-states](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4140-L4308) | [rollback_to.sgml#savepoint](../../../../raw/postgres-12/doc/src/sgml/ref/rollback_to.sgml#L29-L55); measured capture retry and truncated-plan retry |
| Lost session and closed input end the run without a commit | [common.c#CheckConnection](../../../../raw/postgres-12/src/bin/psql/common.c#L386-L397), [postgres.c#client-EOF](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4433-L4455), [postinit.c#ShutdownPostgres](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L1189-L1193) | [psql-ref.sgml#exit-status](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L635-L643); measured `pg_terminate_backend()` case |
| Duplicate-sensitive visible-identity comparison | [heaptuple.c](../../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L636-L640), [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L148-L167) | [select.sgml#EXCEPT-ALL](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L1281-L1294), [ddl.sgml#ctid](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L1198-L1205) |
| Heap-side loss is invisible to both counts | [heapam.c#unresolved-tid](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1538-L1590), [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635) | Physical zeroed-block leg in the published measurement script |
| Unresolved-TID counters and killed-tuple hints | [pgstatfuncs.c#xact-tuple-counters](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1683-L1711), [indexam.c#kill-prior-tuple](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L589), [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1718-L1812) | Deleted-row fixture: 10, then 0 |
| Refused damage-hiding settings | [bufmgr.c#zero-damaged-pages](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925), [bufpage.c#checksum-warning](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L143-L161), [elog.c#default-sqlstate](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L381-L388) | Guard units, CLI refusals and the invalid-header leg |
| Identity drift after the snapshot | [dbsize.c#pg_relation_filenode](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L864-L904), [regproc.c#to_regclass](../../../../raw/postgres-12/src/backend/utils/adt/regproc.c#L947-L968), [tablecmds.c#index-rename-lock](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L3305-L3312) | [mvcc.sgml#DDL-caveat](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1689-L1702); drift fixtures |
| An empty inner select list avoids ORDER BY output-name capture | [parse_clause.c#SQL92-name-rule](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1865-L1930), [gram.y#empty-target-list](../../../../raw/postgres-12/src/backend/parser/gram.y#L14633-L14634) | `?column?` fixture |
| Missing evidence cannot become a completed report | Bash reads complete evidence lists with `hic_get()` before iteration; membership tests distinguish read failure from absence, and line counts check command status | Missing-file and directory substitutions for the relation list, partition facts, index list and attempt list; `hic_finish()` propagates the missing partition-facts failure |
| Native decoding and UTF-8/hex transport, with Bash acceptance | [jsonfuncs.c#array-expansion](../../../../raw/postgres-12/src/backend/utils/adt/jsonfuncs.c#L1976-L2014), [jsonfuncs.c#object-expansion](../../../../raw/postgres-12/src/backend/utils/adt/jsonfuncs.c#L1655-L1695), [encode.c#binary_encode](../../../../raw/postgres-12/src/backend/utils/adt/encode.c#L36-L69), [mbutils.c#pg_convert_to](../../../../raw/postgres-12/src/backend/utils/mb/mbutils.c#L414-L432) | Typed malformed-plan units, text/CSV and complete local-value round trips, four executed multiline-ordering fixtures and two unusual AM-name classifications in the published harness |
| Damage under a cached copy reads as intact | [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L763-L793), [bufmgr.c#verify-after-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L925) | Cached-damage leg: 300 rows before the restart, 242 after |
| A mapped catalog's file number comes from the relation mapper | [pg_class.dat#mapped-catalogs](../../../../raw/postgres-12/src/include/catalog/pg_class.dat#L24-L55), [dbsize.c#mapper-lookup](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L885-L889) | `pg_catalog` run: 63 physical catalogs pass, none with drift |
| A reload reaches the open session; each count re-checks the settings | [postgres.c#reload-between-statements](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4216-L4220) | Mid-run reload leg |
| Warnings reach psql only at `client_min_messages` warning or below | [elog.c#client-output](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L301-L305), [guc.c#client_min_messages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4231-L4240) | Message-level probe and pinned runner session |
| psql does not substitute inside quoted names | [psqlscan.l#quoted-identifier-state](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L132), [psqlscan.l#quoted-identifier-text](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L623), [ruleutils.c#quote_identifier](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10644-L10697) | `odd:'name:` table passes |
| A GCC 15 build of this pin needs C99 mode | [configure.in#default-cflags](../../../../raw/postgres-12/configure.in#L408-L415), [c.h#bool-typedef](../../../../raw/postgres-12/src/include/c.h#L302-L309), [installation.sgml#C99](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L93-L94) | Observed build stop at `c.h:308`; the published build passes `CFLAGS='-O2 -std=gnu99'` |

## Open Questions

- Compatibility beyond exact 12.2, alternate psql releases, planner/EXPLAIN hooks and arbitrary custom types/opclasses/locale combinations remains unverified. A version number check does not prove binary provenance.
- Complete candidate coverage remains best effort: equivalent/constant orderings, planner choice, unsafe expressions and snapshot usability may leave an eligible index untested.
- Pinned documentation says enable_indexscan enables/disables paths, but implementation adds a cost penalty. Source wins; acceptance requires the executed path, not the setting. [config.sgml#indexscan-wording](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4449-L4462) [costsize.c#indexscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L525-L535)
- The interpolation guard deliberately refuses some harmless ordering text too, including literals and quoted identifiers inside an ordering. It records errors and provides no coverage for refused candidates. Schema and table names are not guarded, because psql never substitutes inside a quoted identifier.
- DDL stability is detected only as identity drift of each checked relation and its indexes after the accepted heap attempt; new relations, new indexes, changed index flags and an intact catalog inventory remain premises. Successful inventory means its queries completed, not that lost historical objects were recovered.
- Corruption detection rates and production scalability are unmeasured; the harness damages only three heap blocks, one of them under a cached copy. Only `zero_damaged_pages` is exercised by the mid-run reload leg; the per-count check covers the other reloadable settings by code reading. The detailed test gaps remain under measured coverage. Identity-bag capture, or a cheaper order-independent identity checksum that would catch a misdirected TID cancelling itself, pageinspect diagnosis and alternative transaction architectures are optional future work.
- The unresolved-TID signal cannot tell dead, snapshot-invisible and orphaned index entries apart, and misses entries an earlier scan already marked dead. A census of index pages would see them, but this runner uses no extension.
- Platform portability is limited to the recorded client/server stack. Avoiding Bash-4-only features is not proof of execution on every Bash release, locale or platform.
- Concept-layer gap: no PostgreSQL 12 common-concept page covers retained MVCC/HOT fetching or executed-plan validation; these would require separate tasks.

## Source References

- [configure.in#version](../../../../raw/postgres-12/configure.in#L19-L34)
- [pg_class.h#relation-fields](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L30-L63)
- [pg_index.h#index-fields](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L76)
- [pg_am.dat#access-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L35)
- [pg_am.h#method-types](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L53-L56)
- [pg_inherits.h#parent-links](../../../../raw/postgres-12/src/include/catalog/pg_inherits.h#L30-L37)
- [pg_amop.h#operators](../../../../raw/postgres-12/src/include/catalog/pg_amop.h#L54-L80)
- [pg_aggregate.dat#count](../../../../raw/postgres-12/src/include/catalog/pg_aggregate.dat#L213-L219)
- [instrument.h#row-counters](../../../../raw/postgres-12/src/include/executor/instrument.h#L44-L65)
- [nbtree.h#tuple-layout](../../../../raw/postgres-12/src/include/access/nbtree.h#L249-L291)
- [catalog/Makefile#generated-catalogs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L55-L101)
- [index.c#partitioned-index-link](../../../../raw/postgres-12/src/backend/catalog/index.c#L1002-L1010)
- [nbtree.c#bthandler](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L109-L142)
- [nbtsearch.c#unbounded-scan](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L997)
- [hash.c#capabilities](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L60-L74)
- [hashutil.c#NULL-omission](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L312-L337)
- [ginutil.c#callbacks](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L63-L73)
- [brin.c#callbacks](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L111-L121)
- [indexam.c#heap-visibility](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635)
- [heapam.c#visibility-fast-path](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L428-L453)
- [heapam_handler.c#HOT-fetch](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L130-L167)
- [heapam_visibility.c#MVCC](../../../../raw/postgres-12/src/backend/access/heap/heapam_visibility.c#L944-L981)
- [README.HOT#chains](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L48-L122)
- [heaptuple.c#ctid](../../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L636-L640)
- [transam.c#xid-ordering](../../../../raw/postgres-12/src/backend/access/transam/transam.c#L296-L312)
- [explain.c#execution](../../../../raw/postgres-12/src/backend/commands/explain.c#L494-L541)
- [lockcmds.c#relation-kinds](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L90-L103)
- [matview.c#refresh-locks](../../../../raw/postgres-12/src/backend/commands/matview.c#L159-L169)
- [vacuum.c#horizon](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L902-L912)
- [execMain.c#read-only](../../../../raw/postgres-12/src/backend/executor/execMain.c#L769-L795)
- [execExpr.c#function-permission](../../../../raw/postgres-12/src/backend/executor/execExpr.c#L2159-L2163)
- [execUtils.c#population](../../../../raw/postgres-12/src/backend/executor/execUtils.c#L694-L714)
- [nodeSeqscan.c#SeqNext](../../../../raw/postgres-12/src/backend/executor/nodeSeqscan.c#L50-L82)
- [nodeIndexscan.c#IndexNext](../../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L105-L149)
- [nodeIndexonlyscan.c#VM](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L127-L170)
- [plancat.c#index-gates](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L170-L234)
- [indxpath.c#index-paths](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050)
- [pathkeys.c#index-order](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L487-L546)
- [costsize.c#scan-penalties](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L525-L535)
- [prepjointree.c#subquery-sorting](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L1442-L1463)
- [parse_clause.c#ordering-operator](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L3263-L3295)
- [parse_relation.c#read-lock](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1223-L1235)
- [rewriteHandler.c#materialized-view](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L1900-L1911)
- [procarray.c#TransactionXmin](../../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1489-L1499)
- [bufmgr.c#temporary-relation-access](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L658)
- [lock.c#conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)
- [utility.c#CommandIsReadOnly](../../../../raw/postgres-12/src/backend/tcop/utility.c#L99-L118)
- [namespace.c#temporary-tables](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3889-L3927)
- [ruleutils.c#index-key-deparse](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1085-L1131)
- [xml.c#query-to-XML](../../../../raw/postgres-12/src/backend/utils/adt/xml.c#L2671-L2689)
- [int8.c#count-overflow](../../../../raw/postgres-12/src/backend/utils/adt/int8.c#L682-L705)
- [lsyscache.c#sort-operator-family](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L216-L242)
- [relcache.c#index-xmin](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L2210-L2224)
- [guc.c#scan-profiles](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L881-L941)
- [rls.c#bypass-and-guard](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132)
- [snapmgr.c#repeatable-read](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370)
- [pageinspect/Makefile#module](../../../../raw/postgres-12/contrib/pageinspect/Makefile#L1-L15)
- [rawpage.c#buffer-copy](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L95-L109)
- [heapfuncs.c#physical-decoding](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L180-L224)
- [catalogs.sgml#index-catalog](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3822-L3865)
- [config.sgml#planner-controls](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4616-L4648)
- [ddl.sgml#ctid](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L1198-L1205)
- [func.sgml#count](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L14917-L14936)
- [indices.sgml#ordering](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L480-L542)
- [mvcc.sgml#repeatable-read](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L454-L479)
- [pageinspect.sgml#visibility-boundary](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L169-L175)
- [plpgsql.sgml#dynamic-SQL](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1157-L1185)
- [rules.sgml#materialized-views](../../../../raw/postgres-12/doc/src/sgml/rules.sgml#L898-L929)
- [execute.sgml#SQL-EXECUTE](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50)
- [explain.sgml#execution-effects](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L78-L105)
- [psql-ref.sgml#gexec](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L2059-L2094)
- [rollback_to.sgml#savepoint](../../../../raw/postgres-12/doc/src/sgml/ref/rollback_to.sgml#L29-L55)
- [select.sgml#ONLY](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L355-L365)
- [set.sgml#savepoint-settings](../../../../raw/postgres-12/doc/src/sgml/ref/set.sgml#L61-L65)
- [set_transaction.sgml#read-only](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L116-L139)
- [create_index.sql#expression-index-tests](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L463-L498)
- [inherit.sql#ONLY-tests](../../../../raw/postgres-12/src/test/regress/sql/inherit.sql#L40-L59)
- [matview.sql#population-tests](../../../../raw/postgres-12/src/test/regress/sql/matview.sql#L14-L40)
- [rowsecurity.sql#RLS-tests](../../../../raw/postgres-12/src/test/regress/sql/rowsecurity.sql#L1511-L1542)
- [fe-exec.c#last-command-result](../../../../raw/postgres-12/src/interfaces/libpq/fe-exec.c#L2096-L2141)
- [transam.h#normal-object-identifiers](../../../../raw/postgres-12/src/include/access/transam.h#L136-L141)
- [rel.h#heap-fillfactor](../../../../raw/postgres-12/src/include/utils/rel.h#L278-L279)
- [hio.c#fillfactor-reserve](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L348-L350)
- [heap.c#row-security-state](../../../../raw/postgres-12/src/backend/catalog/heap.c#L916-L919)
- [postinit.c#encoding-default](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L394-L401)
- [installation.sgml#optional-libraries](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L118-L151)
- [gram.y#parenthesized-expression](../../../../raw/postgres-12/src/backend/parser/gram.y#L13464-L13497)
- [analyze.c#SQL92-order-by](../../../../raw/postgres-12/src/backend/parser/analyze.c#L1252-L1256)
- [parse_coerce.c#same-type-no-op](../../../../raw/postgres-12/src/backend/parser/parse_coerce.c#L156-L169)
- [equivclass.c#constant-member](../../../../raw/postgres-12/src/backend/optimizer/path/equivclass.c#L560-L570)
- [pathnodes.h#EC_MUST_BE_REDUNDANT](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L945-L950)
- [allpaths.c#create_plain_partial_paths](../../../../raw/postgres-12/src/backend/optimizer/path/allpaths.c#L796-L805)
- [postgres.c#timeout-errors](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3082-L3095)
- [startup.c#stdin-main-loop](../../../../raw/postgres-12/src/bin/psql/startup.c#L438-L446)
- [mainloop.c#send-on-semicolon](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L410-L440)
- [common.c#SetResultVariables](../../../../raw/postgres-12/src/bin/psql/common.c#L544-L584)
- [command.c#exec_command_echo](../../../../raw/postgres-12/src/bin/psql/command.c#L1116-L1152)
- [psqlscan.l#variable-substitution](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L708-L762)
- [logging.c#locus-prefix](../../../../raw/postgres-12/src/common/logging.c#L164-L179)
- [fe-protocol3.c#verbose-message](../../../../raw/postgres-12/src/interfaces/libpq/fe-protocol3.c#L1017-L1046)
- [errcodes.txt#classes-00-02](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L77-L99)
- [syntax.sgml#dollar-quoting](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L563-L588)
- [scan.l#dolqdelim](../../../../raw/postgres-12/src/backend/parser/scan.l#L292-L294)
- [xact.c#RollbackToSavepoint](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4249-L4308)
- [signalfuncs.c#pg_terminate_backend](../../../../raw/postgres-12/src/backend/storage/ipc/signalfuncs.c#L128-L137)
- [libpq.sgml#environment](../../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L7296-L7319)
- [json.c#report_parse_error](../../../../raw/postgres-12/src/backend/utils/adt/json.c#L1126-L1137)
- [jsonfuncs.c#jsonb-expansion](../../../../raw/postgres-12/src/backend/utils/adt/jsonfuncs.c#L1655-L1695)
- [encode.c#binary_encode](../../../../raw/postgres-12/src/backend/utils/adt/encode.c#L36-L69)
- [mbutils.c#pg_convert_to](../../../../raw/postgres-12/src/backend/utils/mb/mbutils.c#L414-L432)
- [ruleutils.c#index-pretty-flags](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1117-L1131)
- [ruleutils.c#CASE-newlines](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L7741-L7755)
- [ruleutils.c#CASE-expression](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L8309-L8315)
- [ruleutils.c#quoted-identifier-bytes](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10703-L10716)
- [collationcmds.c#from-properties](../../../../raw/postgres-12/src/backend/commands/collationcmds.c#L119-L145)
- [amcmds.c#new-name-and-handler](../../../../raw/postgres-12/src/backend/commands/amcmds.c#L74-L95)
- [pgstatfuncs.c#xact-tuple-counters](../../../../raw/postgres-12/src/backend/utils/adt/pgstatfuncs.c#L1683-L1711)
- [resowner.c#subtransaction-locks](../../../../raw/postgres-12/src/backend/utils/resowner/resowner.c#L594-L603)
- [regproc.c#to_regclass](../../../../raw/postgres-12/src/backend/utils/adt/regproc.c#L947-L968)
- [dbsize.c#pg_relation_filenode](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L864-L904)
- [lmgr.c#invalidations-after-new-lock](../../../../raw/postgres-12/src/backend/storage/lmgr/lmgr.c#L116-L138)
- [tablecmds.c#index-rename-lock](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L3305-L3312)
- [indexcmds.c#session-lock](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3070-L3074)
- [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L631)
- [bufpage.c#checksum-warning](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L143-L161)
- [elog.c#default-sqlstate](../../../../raw/postgres-12/src/backend/utils/error/elog.c#L381-L388)
- [pgstat.c#initstats-gate](../../../../raw/postgres-12/src/backend/postmaster/pgstat.c#L1751-L1772)
- [bufmgr.h#TestForOldSnapshot](../../../../raw/postgres-12/src/include/storage/bufmgr.h#L264-L275)
- [parse_target.c#default-name](../../../../raw/postgres-12/src/backend/parser/parse_target.c#L1661)
- [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L357-L360)
- [pgstat.h#tuple-counters](../../../../raw/postgres-12/src/include/pgstat.h#L1369-L1383)
- [nbtutils.c#_bt_killitems](../../../../raw/postgres-12/src/backend/access/nbtree/nbtutils.c#L1718-L1812)
- [genam.c#ignore-killed](../../../../raw/postgres-12/src/backend/access/index/genam.c#L108-L119)
- [reloptions.c#autovacuum_enabled](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L108-L115)
- [autovacuum.c#av_enabled](../../../../raw/postgres-12/src/backend/postmaster/autovacuum.c#L3026)
- [pg_class.dat#mapped-catalogs](../../../../raw/postgres-12/src/include/catalog/pg_class.dat#L24-L55)
- [c.h#bool-typedef](../../../../raw/postgres-12/src/include/c.h#L302-L309)
- [xlog.c#shutdown-checkpoint](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L8348)

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Versions and source pins](../../../versions.md)
- [How NULL Values Are Handled in PostgreSQL 12 Indexes (unverified)](null-values-in-indexes.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](create-index-concurrently.md)
- [Physical Index Statistics, Tuple Counts, and Bytes per Tuple in PostgreSQL 12 (unverified)](physical-index-statistics-tuple-counts-and-bytes.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](../observability/explain-analyze-buffers-output.md)
