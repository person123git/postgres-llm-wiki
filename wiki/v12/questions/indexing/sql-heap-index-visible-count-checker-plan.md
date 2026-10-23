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
  - [Stronger SQL diagnostics and pageinspect](#stronger-sql-diagnostics-and-pageinspect)
  - [Executable SQL installation](#executable-sql-installation)
  - [Operator workflow and implementation policy](#operator-workflow-and-implementation-policy)
  - [Test strategy](#test-strategy)
  - [Measured coverage and remaining test gaps](#measured-coverage-and-remaining-test-gaps)
  - [Optional relaxations](#optional-relaxations)
  - [Review of the implementation](#review-of-the-implementation)
  - [Follow-up: a Bash and SQL runner](#follow-up-a-bash-and-sql-runner)
  - [How the runner talks to psql](#how-the-runner-talks-to-psql)
  - [The runner's control flow](#the-runners-control-flow)
  - [Runner branches and failure handling](#runner-branches-and-failure-handling)
  - [Running the runner](#running-the-runner)
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

> Scope: before drafting, the asker chose two readings. "The script" is a runner that performs the operator's part of the checker unattended, not only the measurement harness. Bash makes every decision, and psql receives only SQL statements, plus `\echo` sync lines. The checker SQL stays the single source of the checks, the harness now tests the runner, and the harness's embedded C port probe is replaced by Bash.

## Answer

### Feasibility and execution contract

**A verified positive result is feasible with SQL and ordinary [executor](../../../glossary.md#executor) scans. An unattended, universally complete checker is not guaranteed under literal SQL alone.** The deliverable is a SQL generator plus a staged SQL execution and validation protocol. It uses checker-owned temporary tables to retain exact counts, and the operator retains and imports the corresponding executed plans. The follow-up adds a Bash runner that plays the operator's part unattended through one [psql](../../../glossary.md#psql) session; it automates the orchestration, not the coverage, so universal completeness is still not guaranteed. See [the runner](#follow-up-a-bash-and-sql-runner). Choosing a particular index remains best effort; accepting a count requires evidence from that execution. The reasons are PostgreSQL's [utility-command](../../../glossary.md#utility-command) interface for [EXPLAIN](../../../glossary.md#explain), its [prepared-statement](../../../glossary.md#prepared-statement) meaning of SQL `EXECUTE`, and its cost-based choice of index paths. [explain.sgml#execution-and-results](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L78-L105) [execute.sgml#prepared-statement](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50) [indxpath.c#build_index_paths](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050)

The user approved implementation after the planning review. This page now includes the complete operator-assisted SQL implementation, the follow-up's Bash and SQL runner, and a separate reproduction harness. The original planning-only Question remains above as the request that motivated the design. The evidence target is **PostgreSQL 12.2**, commit `45b88269a353ad93744772791feb6d01bc7e1e42`. The checkout identifies itself as 12.2. “Official documentation” below means the official PostgreSQL SGML documentation shipped in that same checkout, checked alongside implementation source. Later 12.x minor releases require a compatibility check before claiming validation on them. The SQL version guard checks `server_version_num`, not the provenance of an arbitrary server binary; the measurement harness builds the exact pin. [configure.in#version](../../../../raw/postgres-12/configure.in#L19-L34)

The implementation runs on a primary and requires no extension. Its only DML writes are to its own temporary result tables. Create those tables before entering the checking transaction. A `READ ONLY` transaction permits DML on temporary tables, because the executor's read-only check skips relations in a temporary namespace. It rejects `CREATE`, including temporary-table creation, because `check_xact_readonly()` refuses a `CreateStmt` whenever the transaction is read-only. Read/write is the default; the explicit transaction option activates that guard. A [standby](../../../glossary.md#hot-standby) cannot create the temporary tables this design needs. [set_transaction.sgml#read-only](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L126-L139) [execMain.c#ExecCheckXactReadOnly](../../../../raw/postgres-12/src/backend/executor/execMain.c#L759-L795) [utility.c#check_xact_readonly](../../../../raw/postgres-12/src/backend/tcop/utility.c#L128-L140) [utility.c#create-refusal](../../../../raw/postgres-12/src/backend/tcop/utility.c#L176-L224) [namespace.c#temporary-recovery-refusal](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3915-L3927)

| Capability | Assessment and handling |
|---|---|
| Catalog inventory, SQL text generation, exact counting, JSON validation and report aggregation | Feasible with core SQL. Keep dynamic object names in generated statements, using catalog-derived identifiers and quoting. The catalog and deparser interfaces are detailed below. |
| Bind an exact count to its executed access path | Feasible with `EXPLAIN ANALYZE INSERT ... SELECT count(*) ...` into a pre-existing temporary table. The same execution inserts the count and produces the plan; see [capture protocol](#capturing-an-exact-count-and-its-executed-plan). |
| Execute arbitrary SQL text returned by an inventory query | A generated string is not executed by ordinary SQL `EXECUTE`; that command executes a named prepared statement. Have the operator, or the follow-up's Bash runner, execute the generated SQL in the same session. [execute.sgml#prepared-statement](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50) |
| Put an `EXPLAIN` utility result directly in a SQL CTE or `INSERT ... SELECT` | No ordinary relational `EXPLAIN` interface is provided by this command. Its result goes to the client. Import that result explicitly before running the SQL validator. [explain.sgml#command-and-output](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L32-L46) [explain.sgml#outputs](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L268-L275) |
| Use `query_to_xml()` as a dynamic-EXPLAIN workaround | Rejected for this pin. It executes through read-only [SPI](../../../glossary.md#spi) and requires `SPI_OK_SELECT`; SPI's read-only test rejects utility statements. It can execute suitable SELECT text, but does not solve this executed-plan capture problem. [xml.c#query_to_xml_internal](../../../../raw/postgres-12/src/backend/utils/adt/xml.c#L2671-L2689) [spi.c#read-only-check](../../../../raw/postgres-12/src/backend/executor/spi.c#L2258-L2263) [utility.c#CommandIsReadOnly](../../../../raw/postgres-12/src/backend/tcop/utility.c#L99-L118) |
| Choose every eligible [B-tree](../../../glossary.md#b-tree) by name and stop automatically after a match | No such guarantee follows from scan settings or ordering. Literal SQL also has no procedural loop over dynamically generated statements. Use a staged “generate next attempt” protocol; the follow-up's Bash runner loops over it, and a procedural wrapper remains a separate option. [indices.sgml#index-versus-sort](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L480-L502) [plpgsql.sgml#dynamic-commands](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1157-L1185) |

“Reliable” means a completed, correctly identified executor observation under the stated snapshot and scope, with a validated plan and faithful capture. It does not mean a proof that a damaged database has retained all historical rows. “Best effort” describes obtaining a suitable index path and achieving candidate coverage; an unsuccessful attempt must never manufacture a reliable result.

### Values and decision flow

The checker delegates [MVCC](../../../glossary.md#mvcc) visibility to PostgreSQL's executor. A [sequential scan](../../../glossary.md#sequential-scan) passes the execution [snapshot](../../../glossary.md#snapshot) into the table access method. An ordinary [index scan](../../../glossary.md#index-scan) passes the same snapshot into [heap](../../../glossary.md#heap) fetching after obtaining index tuple identifiers. [nodeSeqscan.c#SeqNext](../../../../raw/postgres-12/src/backend/executor/nodeSeqscan.c#L50-L82) [nodeIndexscan.c#IndexNext](../../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L105-L149) [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L573)

| Value | Meaning and origin | Live or stored | Use |
|---|---|---|---|
| `T` | Database [OID](../../../glossary.md#oid) plus physical [relation](../../../glossary.md#relation) OID, with recorded schema/name and catalog classification | Stored in `hic_object` when the run starts | Stable identity for one target during the coordinated run |
| `S` | Snapshot retained by the checking transaction; record `txid_current_snapshot()::text` | Live while the repeatable-read transaction lasts; its text is stored in `hic_run` and on every count and plan row ([snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370)) | Associates both accepted scan results with the same visibility boundary |
| `H(T,S)` | `bigint` from an accepted sequential-scan count | Stored | Baseline |
| `J` | Actual index OID resolved from the executed plan and inventory | Stored | Coverage identity; the intended candidate is recorded separately |
| `I(T,J,S)` | `bigint` from an accepted full ordinary Index Scan count | Stored | Comparison with `H` |
| `delta` | `I::numeric - H::numeric` | Derived in `hic_attempt_report` | Zero means equal cardinality; negative means fewer returned rows through that index |
| `indcheckxmin` and `pg_index.xmin` | The index's usability flag and the xmin of its `pg_index` row | Stored in the catalog. The inventory copies the row its snapshot sees; the planner reads the relcache copy, which a relcache rebuild refreshes ([relcache.c#index-xmin-refresh](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L2210-L2224)) | With `TransactionXmin`, decides whether the planner may use the index ([plancat.c#indcheckxmin](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234)) |
| `TransactionXmin` | The backend's oldest snapshot xmin in the current transaction | Live and backend-local; the transaction's first snapshot sets it ([procarray.c#horizons](../../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1489-L1499), [procarray.c#first-snapshot](../../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1709-L1710)) | The planner skips an index with `indcheckxmin` set until its `pg_index.xmin` precedes this value |
| Candidate state | Inventory facts, snapshot-usability evidence, attempted and actually scanned OIDs | Stored inventory plus derived report state | Determines whether “none matched” means `MISMATCH`, `INCONCLUSIVE`, or `ERROR` |

`txid_current_snapshot()` is the version-12 SQL snapshot interface. `count(*)` returns `bigint` and counts all input rows, including rows whose keys are NULL. Neither `reltuples` nor EXPLAIN's estimated rows is one of these count values. [func.sgml#snapshot-interface](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L19384-L19418) [func.sgml#count](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L14917-L14936) [pg_class.h#estimates](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L63)

The objective is:

```text
PASS(T,S) iff there exists an eligible, actually checked J
              for which H(T,S) = I(T,J,S).
```

Equality establishes this count objective only. As an illustrative multiset example, heap results `{a,b,c}` and index results `{a,a,c}` both contain three rows: one missing result and one duplicate cancel. Passing index `J1` supplies no evidence that `J2` passes. The stronger diagnostic below compares tuple-identity multisets to expose that particular cancellation.

The decision flow is:

```text
Catalog inventory and explicit scope policy
  |
  +-- outside physical heap scope --> NOT_APPLICABLE + reason
  |
  +-- required discovery/privilege/identity failure --> ERROR
  |
  +-- in scope
        |
        +-- no supported eligible or unresolved candidate --> NO_ELIGIBLE_INDEX
        |
        +-- obtain H and its executed plan
              |
              +-- execution/capture failure --> ERROR
              +-- unacceptable scan path --> INCONCLUSIVE
              |
              +-- validated H
                    |
                    +-- generate candidate attempt; execute; capture; validate
                          |
                          +-- different eligible index used --> credit actual J
                          +-- valid I = H --> PASS; stop unless exhaustive
                          +-- valid I != H --> record mismatch; continue
                          +-- rejected path/unresolved usability --> untested
                          +-- execution/permission/capture failure --> attempt ERROR
                    |
                    +-- no match after attempts:
                          required execution/inspection failure --> ERROR
                          any eligible/unresolved candidate untested --> INCONCLUSIVE
                          every eligible candidate checked --> MISMATCH
```

These are checker rules. The source-grounded conditions feeding them are the inventory and eligibility gates, the shared-snapshot mechanism, and the same-execution capture and plan rules in the next sections.

### Scope and catalog inventory

Resolve the requested schema to a `pg_namespace` OID, then inventory its [`pg_class`](../../../glossary.md#pg_class) rows without first filtering by privileges or index existence. Join the table's `relam` to `pg_am`; separately join [`pg_index`](../../../glossary.md#pg_index) rows on `pg_index.indrelid = pg_class.oid`, the index's own `pg_class` row, and its index `pg_am` row. Require table `amtype = 't'` and index `amtype = 'i'` in their respective classifications. Keep a relation row even when the left join finds no index. These catalogs distinguish namespace, relation kind, table versus index access method, and index-to-table ownership. [pg_class.h#identity-and-AM](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L30-L54) [pg_class.h#kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) [pg_index.h#catalog](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L57) [pg_am.h#method-types](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L53-L56) [pg_am.dat#built-in-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L35)

The inventory specification is:

| Catalog condition | Classification and treatment |
|---|---|
| `relkind = 'r'`, built-in heap table AM | In scope: ordinary tables and physical table partitions, including ordinary [inheritance](../../../glossary.md#inheritance) parents that store their own rows |
| `relkind = 'm'`, heap AM, `relispopulated` | In scope: count the stored rows of the [materialized view](../../../glossary.md#materialized-view), not the defining query |
| `relkind = 'p'` | `NOT_APPLICABLE: partitioned_parent_summary` as a scan target; `hic_parent_result` summarizes its children |
| `relkind = 'm'`, not populated | `NOT_APPLICABLE: unpopulated_materialized_view`; never invent a zero count |
| `relkind = 'r'` or `'m'`, another table AM | `NOT_APPLICABLE: unsupported_table_access_method`, with the `unsupported_table_am` flag set; report a physical-object coverage gap |
| View `'v'`, foreign table `'f'`, sequence `'S'` | `NOT_APPLICABLE` with the reason `view`, `foreign_table` or `sequence`; no local heap-count claim |
| Index `'i'` or [partitioned index](../../../glossary.md#partitioned-index) `'I'` | `NOT_APPLICABLE: index_metadata`; inventoried as index metadata, not as a second table to count |
| [TOAST](../../../glossary.md#toast) `'t'`, composite type `'c'`, any other kind | `NOT_APPLICABLE: outside_physical_heap_scope`; explicitly outside this initial table/materialized-view scope |

The classifications use `relkind`, `relam`, `relispopulated`, and `relispartition`, not file size or the existence of a parent. PostgreSQL excludes partitioned parents from its storage-bearing kinds. Ordinary materialized-view reads skip rule expansion, and execution of an unpopulated materialized view raises an error. [pg_class.h#state](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L104-L119) [pg_class.h#storage-kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L183-L192) [rewriteHandler.c#materialized-view-reads](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L1900-L1911) [execUtils.c#unpopulated-refusal](../../../../raw/postgres-12/src/backend/executor/execUtils.c#L694-L714) [rules.sgml#materialized-views](../../../../raw/postgres-12/doc/src/sgml/rules.sgml#L898-L929)

**Default schema boundary: namespace only.** Include physical relations whose own `relnamespace` is the requested schema. Include such a partition even when its parent is elsewhere. Exclude an outside-schema partition from scanning even when its parent is inside the schema, but keep it in that parent's summary: its `hic_partition.outside_schema` flag is true, and `hic_parent_result.outside_schema_descendants` counts it. A parent summary must say whether it covers all descendants or only the selected subset.

Offer an explicit `include_external_partition_descendants` option. When enabled, recursively traverse `pg_inherits` from partitioned parents in the requested schema and include their physical descendants across schemas, deduplicated by relation OID. Record the added schemas and apply their privileges too. Restrict traversal to table relationships: `pg_inherits` also carries partitioned-index links in this implementation. Do not automatically expand ordinary inheritance into this optional partition closure. [pg_inherits.h#parent-links](../../../../raw/postgres-12/src/include/catalog/pg_inherits.h#L30-L37) [index.c#index-inheritance-link](../../../../raw/postgres-12/src/backend/catalog/index.c#L1002-L1010) [ddl.sgml#declarative-storage](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3611-L3626)

Every physical-target scan uses `FROM ONLY schema.relation`. Without `ONLY`, a parent query also scans inherited descendants; that would mix multiple physical objects into its baseline. A partitioned-parent summary aggregates child statuses and coverage, never accepts equality of aggregate parent totals as a replacement for individual child passes. The optional sum of child counts is meaningful as a common-snapshot total only in schema-wide snapshot mode. [select.sgml#ONLY](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L355-L365)

Retain `relpersistence` in the inventory. Permanent and [unlogged](../../../glossary.md#unlogged-relation) heap relations are eligible for this executor comparison; their durability is not being assessed. Own-session temporary application tables can be included when they are the explicitly requested scope. Classify another session's temporary relation as `ERROR: unsupported_remote_temp` before attempting a count: the reader cannot see its owner's local buffers. Keep the object in discovery rather than silently dropping it. Always exclude checker-owned scratch OIDs from the target set. [pg_class.h#persistence](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L165-L167) [bufmgr.c#temporary-relation-access](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L658)

### Index eligibility and ordering generation

A supported structural candidate must belong to an in-scope target and satisfy all of:

```text
index pg_class.relkind = 'i'
index AM = the built-in btree AM
pg_index.indpred IS NULL
pg_index.indisvalid AND pg_index.indisready AND pg_index.indislive
```

Keep rejected indexes with the reason codes the installation emits: `partial_index`, `invalid_index`, `index_not_ready`, `index_not_live`, `nonphysical_index` (a partitioned index) and `unsupported_index_am`. A [partial index](../../../glossary.md#partial-index) with a logically trivial predicate still fails this requested `indpred IS NULL` contract. The three state flags have distinct meanings; they are not estimates of physical correctness. The [planner](../../../glossary.md#planner) rejects invalid and partitioned indexes, while the [relcache](../../../glossary.md#relcache)'s usable index list omits indexes marked not live. [catalogs.sgml#index-state](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3822-L3865) [pg_index.h#predicate](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L53-L57) [plancat.c#validity-and-partitioned-indexes](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L199-L234) [relcache.c#non-live-indexes](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4383-L4400)

**Snapshot usability is an additional gate.** [`indcheckxmin`](../../../glossary.md#indcheckxmin) does not mean “always reject this index.” When true, the planner requires the `xmin` of the index's relcache `pg_index` tuple to precede the backend's [TransactionXmin](../../../glossary.md#transactionxmin). Otherwise it skips the index and marks the plan [transient](../../../glossary.md#transient-plan). This protects snapshots that could see incompatible [HOT](../../../glossary.md#hot) chain versions. [plancat.c#indcheckxmin](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234) [catalogs.sgml#indcheckxmin](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3835-L3844)

Record `indcheckxmin`, the observed `pg_index.xmin`, and the snapshot descriptor. Do not reproduce that gate with `xmin::text::bigint < txid_snapshot_xmin(...)`: the planner uses wrap-aware [transaction-ID](../../../glossary.md#transaction-id) comparison and a backend horizon, and its relcache tuple is not necessarily the catalog row an earlier SQL statement observed. The initial design treats a true flag as **usability unresolved until an actual accepted plan uses that index**, rather than guessing the horizon. A refused candidate remains unresolved/untested unless there is independent conclusive exclusion evidence. It therefore blocks `MISMATCH` and `NO_ELIGIBLE_INDEX`, but does not block a valid pass through another index. [transam.c#TransactionIdPrecedes](../../../../raw/postgres-12/src/backend/access/transam/transam.c#L296-L312) [procarray.c#horizons](../../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1489-L1499) [relcache.c#index-xmin-refresh](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L2210-L2224)

Non-partial status is insufficient across [access methods](../../../glossary.md#access-method). B-tree advertises ordinary tuple scans, ordering, optional search keys, and NULL support. Hash omits NULL entries and requires a search key; GIN and BRIN have no ordinary `amgettuple` callback. Other methods and their operator classes need a separate completeness argument. Defer hash, GiST, SP-GiST, GIN, BRIN and extension AMs in the first implementation; do not use [bitmap-scan](../../../glossary.md#bitmap-scan) deduplication or a row-restricting predicate to simulate their full scan. [nbtree.c#bthandler](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L109-L142) [hashutil.c#NULL-omission](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L312-L337) [hash.c#capabilities](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L60-L74) [ginutil.c#scan-callbacks](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L63-L73) [brin.c#scan-callbacks](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L111-L121)

For each structural B-tree candidate, generate an ordering specification as follows:

1. Walk key positions `1..indnkeyatts`, not all `indnatts`. Included columns do not define index order. The deparser resolves `indkey`; the generator reads `indclass`, `indcollation`, and `indoption` at their corresponding zero-based vector positions. [pg_index.h#key-metadata](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L32-L57) [pathkeys.c#INCLUDE-exclusion](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L487-L492) [ruleutils.c#index-keys](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1304-L1350)
2. Obtain each column or [expression](../../../glossary.md#expression-index) with `pg_get_indexdef(index_oid, position, false)`. On this pin the positive-position form requests only the variable/expression and suppresses index decorations. It also wraps any key that is not a bare function call in parentheses. Do not split `pg_get_expr(indexprs, ...)` at commas or paste a full `CREATE INDEX` key specification into `ORDER BY`. Expressions themselves can contain commas, and an opclass name is not an `ORDER BY` suffix. [ruleutils.c#column-deparse](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1085-L1131) [ruleutils.c#expression-and-decoration](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1331-L1399)

   Wrap each deparsed key in another pair of parentheses, except a key that deparses to a bare numeric literal, such as an index on `(2)`. Parentheses do not protect such a key. With the default `operator_precedence_warning = off`, the grammar discards redundant parentheses, and a SELECT's `ORDER BY` follows SQL92 rules: a bare integer constant names an output column by position, and any other bare constant is an error. `pg_get_indexdef()` prints a non-negative `integer` constant, and a `numeric` constant with a decimal point, without a type label. The generator therefore emits such a key as `CAST(<key> AS <its type>)`. The parser returns a same-type cast of a constant as the constant itself, so the sort key still equals the index expression. [ruleutils.c#get_const_expr-bare-numbers](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9554-L9585) [gram.y#parenthesized-expression](../../../../raw/postgres-12/src/backend/parser/gram.y#L13464-L13497) [guc.c#operator_precedence_warning](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1806-L1814) [analyze.c#SQL92-order-by](../../../../raw/postgres-12/src/backend/parser/analyze.c#L1252-L1256) [parse_clause.c#integer-constant](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1931-L1966) [parse_coerce.c#same-type-no-op](../../../../raw/postgres-12/src/backend/parser/parse_coerce.c#L156-L169)
3. Append the exact indexed [collation](../../../glossary.md#collation) when `indcollation` is nonzero. Qualify its namespace and quote identifiers. Generate and execute under the same controlled `search_path` so deparsed expression names resolve consistently. [ruleutils.c#index-collation](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1372-L1381) [pathkeys.c#sort-identity](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L512-L522)
4. Resolve the [operator class and family](../../../glossary.md#operator-class). For ordinary default ordering, `ASC`/`DESC` can suffice. For a nondefault family such as a pattern opclass, derive the same-type B-tree less-than or greater-than operator from `pg_opclass` and `pg_amop`, then use `USING OPERATOR(qualified_operator)`. Use strategy 1 for forward ascending and 5 for forward descending, with `amoppurpose = 's'`; these are B-tree search-family members, not `amoppurpose = 'o'` distance-ordering operators. Validate type compatibility instead of adding arbitrary casts. If overloaded/polymorphic/custom operators cannot be reconstructed confidently, retain the candidate as untested. [pg_amop.h#operator-metadata](../../../../raw/postgres-12/src/include/catalog/pg_amop.h#L54-L80) [parse_clause.c#SORTBY_USING](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L3263-L3295) [lsyscache.c#ordering-operator](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L216-L242)
5. Decode `INDOPTION_DESC` and `INDOPTION_NULLS_FIRST`; specify NULL positioning explicitly for every key. A backward scan reverses both ordering and NULL positioning for every key. The generator emits complete forward ordering; a complete reverse-order variant remains optional and is not implemented here. The validator accepts either actual scan direction. [pg_index.h#ordering-bits](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L70-L76) [pathkeys.c#scan-direction](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L497-L519) [indices.sgml#ordering](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L505-L542)

The generator must treat quoted schema, relation, column, collation and function names correctly. Use `format('%I.%I', ...)` for identifiers and literal quoting for values. Operator tokens require their SQL operator syntax, not identifier quoting; use only catalog-derived, validated operator names. Expressions, collation implementations and custom operator functions must also be trusted to execute in the checking role. Refuse or leave untested an expression whose execution is outside the allowed function policy; an `IMMUTABLE` declaration alone is not an audit of arbitrary user code. [func.sgml#format-examples](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L3402-L3414) [parse_clause.c#ordering-resolution](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L3277-L3295)

The implementation checks dependencies and index-column types **before** calling the deparser. Deparsing a non-NULL constant invokes its type-output function, so “we will not execute the generated query” is not enough to avoid all user code during inspection. The generator conservatively defers non-stock functions/operators and unreviewed types, including domains; it permits enums and reviewed stock opclasses/operators. Its OID cutoff follows `FirstNormalObjectId`; it is a coverage policy for the pinned stock catalogs, not an authentication boundary against modified catalogs, malicious code or EXPLAIN hooks. [ruleutils.c#get_const_expr](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9522-L9550) [transam.h#FirstNormalObjectId](../../../../raw/postgres-12/src/include/access/transam.h#L136-L141) [explain.c#index-name-hook](../../../../raw/postgres-12/src/backend/commands/explain.c#L2844-L2862)

Compatible ordering encourages a path; it does not name an index. With index-only scans disabled, the planner builds an ordinary index path only when restrictions, useful ordering, or another qualifying reason makes it useful. An index containing only a constant expression gets no such path: a constant sort key falls into an equivalence class that contains a constant, and the planner drops a pathkey for such a class as redundant, so no useful ordering remains. Identically ordered indexes may repeatedly select the same cheaper index. These are coverage limits, not proof of corruption. [indxpath.c#path-generation-gate](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050) [equivclass.c#constant-member](../../../../raw/postgres-12/src/backend/optimizer/path/equivclass.c#L560-L570) [pathnodes.h#EC_MUST_BE_REDUNDANT](../../../../raw/postgres-12/src/include/nodes/pathnodes.h#L945-L950) [pathkeys.c#pathkey_is_redundant](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L132-L140) [pathkeys.c#redundant-ordering](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L524-L546)

### Snapshot permissions and DDL coordination

Use `BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY` after scratch-table setup. The default [isolation level](../../../glossary.md#isolation-level) is read committed, which takes a new snapshot for later statements. Repeatable read retains the transaction's first snapshot; the checker must not perform application DML between the two scans. Recording a snapshot descriptor helps audit provenance but does not recreate a completed transaction's snapshot later. [guc.c#isolation-default](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4255-L4272) [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [mvcc.sgml#repeatable-read](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L454-L479)

| Transaction mode | Guarantee | Operational trade-off |
|---|---|---|
| One schema-wide transaction, implemented here | Every accepted relation result shares one snapshot | Holds the snapshot and acquired relation/index locks until the run ends; one aborted transaction can lose all temporary result changes from that run |
| One transaction per physical relation, optional architecture not implemented here | Each `H(T,S_T)` and its index attempts share one `S_T` | Shorter snapshots and easier retries; the report must say there is no single schema-wide snapshot, and must not describe summed counts as one point-in-time schema total |

The run's inventory manifest must also remain complete across per-relation transactions. Require the membership/identity DDL freeze for the whole run, or re-inventory and invalidate a schema report whose target set changed. Shorter per-relation snapshots do not remove that scope-completeness requirement.

A long snapshot can retain old row versions that VACUUM would otherwise remove because its cleanup cutoff depends on the [xmin horizon](../../../glossary.md#xmin-horizon) of active transactions. Ordinary scan locks are compatible with DML, but conflicting DDL can wait behind the checker. [vacuum.c#cleanup-horizon](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L858-L865) [vacuum.c#GetOldestXmin](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L902-L912) [lock.c#lock-conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)

Require schema `USAGE`, relation `SELECT`, and database `TEMP` privileges for scratch tables, plus permission to execute any functions used by the generated expressions. Table-wide SELECT is the proposed minimum privilege policy, even where a narrower column grant could execute one particular query. [namespace.c#schema-usage](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L2903-L2910) [namespace.c#temporary-privilege](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3889-L3905) [execMain.c#relation-permissions](../../../../raw/postgres-12/src/backend/executor/execMain.c#L637-L665) [execExpr.c#function-permission](../../../../raw/postgres-12/src/backend/executor/execExpr.c#L2159-L2163)

Do not remove inaccessible tables from the inventory with a privilege predicate. Report them as `ERROR`. Use a role that sees all rows, preferably a suitably authorized role with `BYPASSRLS` and the required object privileges; an authorized superuser also qualifies. Table ownership bypasses [RLS](../../../glossary.md#row-level-security) only when `FORCE ROW LEVEL SECURITY` does not apply. `SET LOCAL row_security = off` raises an error instead of applying a policy; it grants no access. [rls.c#bypass-and-error](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132) [config.sgml#row-security](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L7395-L7405)

Coordinate a DDL-stable window **before the first snapshot-acquiring statement and through the end of the applicable checking transaction**. This includes inventory/preflight queries and scratch-table INSERTs, not just the first target count. The window covers relation creation/drop/rename/schema moves, partition attachment/detachment, table rewrites, `TRUNCATE`, materialized-view refresh, and index creation/drop/rebuild. Ordinary reads acquire `ACCESS SHARE`; planner inspection also opens indexes and retains its locks. Those locks do not constitute a universal schema or concurrent-DDL freeze. In particular, the documented non-MVCC-safe rewrite/TRUNCATE case can affect a relation not yet accessed when an older snapshot was taken. [parse_relation.c#read-lock](../../../../raw/postgres-12/src/backend/parser/parse_relation.c#L1223-L1235) [plancat.c#index-locks](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L170-L196) [mvcc.sgml#DDL-caveat](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1689-L1702)

An optional pre-lock phase can use `LOCK TABLE ONLY ... IN ACCESS SHARE MODE` for known ordinary targets in deterministic OID order. This uses `SELECT` privilege. **Do not emit that command for a materialized view on PostgreSQL 12**: its `LOCK TABLE` implementation accepts tables, partitioned tables and views, but rejects materialized views. An ordinary read, such as a preflight `SELECT ... LIMIT 0`, can acquire the materialized-view read lock under the already established DDL coordination policy. That preflight is not a count and supplies no scan evidence. The implementation below does not add this pre-lock phase; ordinary count planning/execution acquires read locks as targets are reached. Ordinary refresh requests `ACCESS EXCLUSIVE`; concurrent refresh requests `EXCLUSIVE`, which permits `ACCESS SHARE`, so the latter must be covered by the explicit coordination policy rather than presumed blocked. [lockcmds.c#allowed-kinds](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L90-L103) [lockcmds.c#privilege](../../../../raw/postgres-12/src/backend/commands/lockcmds.c#L302-L310) [matview.c#refresh-lock](../../../../raw/postgres-12/src/backend/commands/matview.c#L159-L169) [lock.c#exclusive-conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L93-L103)

If identity, membership, eligibility or a DDL assumption changes, invalidate the affected attempt or run. Do not mix an old heap count with a count taken after restarting the transaction. Re-discover and recount in a new transaction with a new run/snapshot identifier.

### Settings and query shapes

All [GUC](../../../glossary.md#guc) settings in this table are `user` / `PGC_USERSET`; none requires reload or restart. The checking profiles use `SET LOCAL`. Installation also sets the dedicated session's search path, 60-second statement timeout and 2-second lock timeout before the checking transaction begins. Record effective values as well as the requested profile. The isolation level is selected at transaction start, before a query takes a snapshot. [guc.c#scan-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L881-L941) [guc.c#parallel-setting](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3007-L3015) [guc.c#row-security](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1571-L1578) [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396) [guc.c#search-path](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3789-L3797) [guc.c#quote-all-identifiers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1816-L1824) [set_transaction.sgml#isolation-timing](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L116-L123)

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

The nondefault scan profiles change costing, not the acceptance contract. `enable_seqscan = off` and `enable_indexscan = off` add [`disable_cost`](../../../glossary.md#disable_cost); disabling index-only scans is checked when constructing an index-only path. The official sequential-scan and sort documentation expressly says these methods cannot be suppressed entirely. [costsize.c#seqscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L225-L232) [costsize.c#indexscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L525-L535) [costsize.c#bitmap-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L963-L970) [indxpath.c#index-only-gate](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L1898-L1907) [config.sgml#seqscan-and-sort](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4616-L4648)

The heap counting shape is:

```sql
SELECT /* wiki_heap_count_shape */ count(*)
FROM ONLY target_schema.target_relation;
```

The initial index counting shape is:

```sql
SELECT /* wiki_index_count_shape */ count(*)
FROM (
    SELECT 1
    FROM ONLY target_schema.target_relation
    ORDER BY indexed_key ASC NULLS LAST
) AS ordered_rows;
```

These are proposed templates, not installed checker code. Replace the names and the entire ordering list using the inventory/deparser process above. A key with a different direction, NULL positioning, collation or operator family needs its corresponding specification. The inner SELECT intentionally has no restricting `WHERE`, `DISTINCT`, grouping, sampling, join, limit, offset or row-locking clause. The outer aggregate counts rows, not non-NULL keys. A sorting subquery is not subject to the simple-subquery pull-up rule on this pin; useful ordering can therefore motivate an index path, but the chosen plan is still subject to validation. [prepjointree.c#sorting-subquery](../../../../raw/postgres-12/src/backend/optimizer/prep/prepjointree.c#L1442-L1463) [indxpath.c#ordered-path](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050) [func.sgml#count](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L14917-L14936)

Disabling [index-only scans](../../../glossary.md#index-only-scan) is deliberate. Their [visibility-map](../../../glossary.md#visibility-map) shortcut can avoid the heap fetch entirely. An ordinary Index Scan instead calls `index_fetch_heap()`, whose table-AM implementation follows HOT chains and checks visibility using the scan snapshot. The checker wants the latter observation. [nodeIndexonlyscan.c#VM-shortcut](../../../../raw/postgres-12/src/backend/executor/nodeIndexonlyscan.c#L127-L170) [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635) [heapam_handler.c#heap-fetch](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L146-L167)

### Capturing an exact count and its executed plan

Do not execute `EXPLAIN` and then a separate plain count and claim the first plan proves the second execution. Plain EXPLAIN does not run the plan; a later statement can be replanned. Nor does `EXPLAIN ANALYZE SELECT count(*) ...` return the aggregate value: it discards SELECT output. Its Aggregate node returns one row regardless of that row's count value. [Instrumentation](../../../glossary.md#plan-node-instrumentation) stores floating-point row counters and EXPLAIN prints rows per loop, rounded to zero decimal places; those counters should not replace the exact `bigint` aggregate. [explain.c#execution-and-destination](../../../../raw/postgres-12/src/backend/commands/explain.c#L494-L541) [instrument.h#row-counters](../../../../raw/postgres-12/src/include/executor/instrument.h#L44-L65) [explain.c#rows-per-loop](../../../../raw/postgres-12/src/backend/commands/explain.c#L1505-L1534) [explain.sgml#discarded-output](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L87-L105)

Use this protocol for each attempt:

1. Before the checking transaction, create checker-owned temporary capture, inventory and audit tables. Do not attach application-supplied triggers, rules, defaults or generated expressions. This implementation permits only its fixed metadata defaults and its own key/check constraints; the settings default calls the published SQL helper. Use explicit run/attempt keys so a failed attempt cannot accidentally reuse an old result.
2. Start repeatable read, fix scope and snapshot, save the proposed SQL text/settings, and establish an attempt [savepoint](../../../glossary.md#subtransaction).
3. Execute **one explained INSERT** that reads the target and stores its `count(*)` in the temporary capture table.
4. Retain the JSON plan returned to the client with that statement's run/attempt identity. Read the inserted count using the same keys.
5. Import that exact JSON result into the temporary audit table with a literal SQL INSERT, using proper literal escaping, or retain it in the operator's query transcript and mark the audit pending until imported. This is an explicit manual bridge in literal SQL mode; the follow-up's runner performs the same import with a dollar-quoted literal.
6. Run the SQL validator over the imported JSON and saved statement contract. Only a validated attempt may supply `H` or `I`; any missing or incorrectly associated evidence leaves the result unaccepted.

For example, a heap attempt has this shape after the generator has saved its attempt record, established the savepoint and applied the heap profile:

```sql
EXPLAIN /* wiki_heap_count_capture */
    (ANALYZE, VERBOSE, BUFFERS, SETTINGS, TIMING OFF, SUMMARY ON, FORMAT JSON)
INSERT /* wiki_heap_capture_value */ INTO pg_temp.hic_count(attempt_id, row_count)
SELECT /* wiki_heap_capture_input */ 1::bigint, count(*)
FROM ONLY target_schema.target_relation;
```

The index attempt uses the same explained INSERT with a new attempt ID and the ordered subquery from the index shape. The example IDs and object names are placeholders, not observed results. The resulting plan has a temporary-table insertion wrapper; validation focuses on its source branch while verifying that the only write target is the expected temporary capture relation.

`EXPLAIN ANALYZE` executes INSERT side effects even though it discards ordinary output, and `ExecutorRun(..., 0L, ...)` runs the plan to completion. Therefore the stored count and the plan describe one execution. `TIMING OFF` removes per-node timing overhead while preserving row instrumentation and total execution timing; it is an EXPLAIN option, not a server GUC. `FORMAT JSON` changes the default text output to structured output. [explain.c#run-to-completion](../../../../raw/postgres-12/src/backend/commands/explain.c#L501-L541) [explain.sgml#side-effects](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L87-L105) [explain.sgml#timing-and-summary](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L195-L223) [explain.sgml#format](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L227-L235)

Bare EXPLAIN defaults `ANALYZE`, `VERBOSE`, `BUFFERS` and `SETTINGS` to false. This protocol turns them on to execute the statement, expose schema-qualified names/output expressions, and retain buffer and planner-setting evidence. `TIMING` and `SUMMARY` normally follow `ANALYZE`; the protocol explicitly disables the former and keeps the latter. [`SETTINGS`](../../../glossary.md#explain-settings) reports only nondefault parameters affecting planning, so it cannot replace the run's separate effective-setting record. [explain.c#option-defaults](../../../../raw/postgres-12/src/backend/commands/explain.c#L154-L221) [explain.c#NewExplainState](../../../../raw/postgres-12/src/backend/commands/explain.c#L285-L295) [explain.sgml#verbose](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L130-L139) [explain.sgml#settings](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L156-L162)

The literal-SQL bridge assumes faithful capture of unedited server output. A SQL validator can reject mismatched identifiers and bad shapes; it does not cryptographically authenticate a JSON literal supplied by an operator. Record who associated each plan with its attempt, and leave the attempt pending if that association cannot be established. Under the runner, the association is the runner's: it imports only the text that psql printed for that attempt's statement, and it keeps the plan file and a transcript of every request.

The generated count also projects its attempt ID as a `bigint` constant beside `count(*)`. The validator checks that marker in the verbose Aggregate output and checks the saved transaction, snapshot, backend, role and scan settings. Those checks catch accidental mismatches; they do not authenticate an old document manually imported into a new run.

The generator deliberately ends `sql_text` with EXPLAIN. `PQexecFinish()` returns the last result of a multi-command submission, so appending settings-reset commands can hide the JSON result from a client using that interface. `post_count_sql` is returned separately for this reason. psql reading a script does not hit this case, because it sends each semicolon-terminated statement as its own query and prints each result. [mainloop.c#send-on-semicolon](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L410-L440) [fe-exec.c#PQexecFinish](../../../../raw/postgres-12/src/interfaces/libpq/fe-exec.c#L2096-L2141)

On this pin, the JSON `"Index Name"` value already contains SQL identifier quoting: `explain_get_index_name()` calls `quote_identifier()` before `ExplainIndexScanDetails()` emits the property. Resolve it using the same quoting convention, not direct equality to raw `pg_class.relname`. The checker fixes `quote_all_identifiers = off` for captures and in its identity helper, so a caller's restored setting after COMMIT does not change historical matching. The setting's default is off and its context is `user`; it is not a scan-forcing control. [explain.c#index-name](../../../../raw/postgres-12/src/backend/commands/explain.c#L2844-L2862) [explain.c#index-JSON](../../../../raw/postgres-12/src/backend/commands/explain.c#L2991-L3025) [guc.c#quote-all-identifiers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1816-L1824)

The temporary write is allowed in `READ ONLY`; creating the capture table inside that transaction is not. `EXPLAIN ANALYZE CREATE TEMP TABLE AS ...` would also retain a query result, but conflicts with the recommended read-only transaction mode, so it is not the baseline design. [execMain.c#temporary-DML](../../../../raw/postgres-12/src/backend/executor/execMain.c#L769-L791) [set_transaction.sgml#read-only](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L126-L139)

Literal SQL cannot automatically catch an execution error, retrieve its diagnostics, branch to `ROLLBACK TO SAVEPOINT`, and resume the dynamic loop. The operator must do that, or stop and preserve the transcript; the follow-up's runner does it in Bash. Keep the core process fail-closed: a transaction abort, disconnect, missing plan or missing result never becomes an empty table or a pass. A generated script must mark these interaction points, not bury them in a `DO` block. [plpgsql.sgml#dynamic-execution](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1157-L1185) [rollback_to.sgml#savepoint](../../../../raw/postgres-12/doc/src/sgml/ref/rollback_to.sgml#L29-L55)

Before retrying an attempt, reapply and record its scan profile. Rolling back to a savepoint also cancels SET/SET LOCAL commands issued after that savepoint. [set.sgml#savepoint-settings](../../../../raw/postgres-12/doc/src/sgml/ref/set.sgml#L61-L65)

After a validated match, the next-step generator returns no more index attempts for that relation unless exhaustive mode is on. It generates the next untested candidate only after the prior plan has been imported and validated. This supplies early stopping without pretending that one static all-candidates batch can react to its own unimported EXPLAIN output.

### Mandatory plan validation

Implement the validator as core SQL over saved `jsonb`, recursively visiting every `Plans` child with array expansion such as the built-in `jsonb_array_elements()`. Use a narrow allowlist and default to rejection of unknown shapes. Merely searching the JSON text for `"Seq Scan"` or `"Index Scan"` is insufficient: such a node may be in an unused branch or below a filter or limit. EXPLAIN exposes node type, actual loops, relation identity, index name and scan direction, but the normal JSON fields do not provide a relation/index OID pair. Resolve names through the retained inventory and verify `pg_index.indrelid`; retain that resolution with the plan. [pg_proc.dat#jsonb-array-elements](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L9100-L9104) [explain.c#instrumentation](../../../../raw/postgres-12/src/backend/commands/explain.c#L1505-L1549) [explain.c#index-details](../../../../raw/postgres-12/src/backend/commands/explain.c#L2991-L3023) [explain.c#relation-details](../../../../raw/postgres-12/src/backend/commands/explain.c#L3156-L3162)

The acceptance rules are:

| Gate | Heap result | Index result |
|---|---|---|
| Provenance | Successful completion; exactly one newly captured count row; JSON from that exact explained statement; correct run, attempt, target and snapshot | Same |
| Target scan | Exactly one source `Seq Scan` on the intended physical relation | Exactly one source ordinary `Index Scan` on that relation |
| Index identity | No index source | Resolve the **actual** `Index Name` to one physical index OID belonging to `T`; require the supported eligibility gates |
| Unrestricted source | No scan or upper filter, one-time filter, row-security qual, sampling or row-restricting operation | Same, plus no `Index Cond`, parameterized index keys, recheck restriction or other scan restriction |
| Completion and multiplicity | Target scan `Actual Loops = 1`, executed even when it returns zero rows; no parallel-aware execution or workers | Same; ordinary forward or backward scan permitted |
| Source tree | One plain `count(*)` aggregate without grouping, aggregate filter or DISTINCT; row-preserving wrappers only | Same |
| Writes | The sole ModifyTable operation is insertion into the expected checker-owned temporary capture table | Same |
| Coverage | Only accepted heap evidence supplies `H` | Only actual accepted index OIDs count as checked |

Allow a `Sort`, `Subquery Scan`, projection-only `Result`, or `Materialize` only when its full definition is verified to preserve rows, its children satisfy the one-scan contract, and it has no filtering or extra subplans. Reject joins, Append/MergeAppend, Gather, bitmap, index-only, TID, foreign/custom and sample scans, Limit, Unique, SetOp, ProjectSet, additional scans, and unexpected InitPlans/SubPlans in these count statements. The checker chooses this restrictive contract; a rejected plan is not a PostgreSQL correctness finding.

Source validation must also verify the generated statement's `count(*)`, `ONLY`, lack of restrictive clauses and safe expression policy. The plan alone does not reliably reveal every property of arbitrary SQL or of user code called by an expression. Start with statements produced by this generator, not an API that accepts arbitrary SQL and attempts to certify it.

An accepted B-tree scan with no boundary keys begins at a tree endpoint and follows the executor's index traversal. “Full scan” here means that unrestricted executor traversal completed. It is not a separate proof that every physical page is reachable from a possibly damaged tree. [nbtsearch.c#unbounded-start](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L997) [indexam.c#scan-completion](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635)

If an attempt aimed at `J2` actually uses `J1`, record `intended_index_oid = J2` and `actual_index_oid = J1`. A matching `J1` can establish PASS. If `J1` already mismatched, a second scan of it does not check `J2`. After a bounded set of ordering/profile retries, mark `J2` untested and move on; do not claim exhaustive coverage because every generated query was executed.

### Result records and aggregation

The installation implements these logical records with explicit keys and checks. `hic_run` includes the snapshot fields; `hic_attempt_report` joins the attempt, exact count, imported plan, error and accepted heap baseline.

| Record | Stored or reported fields |
|---|---|
| Run | `run_id bigint`, database OID/name, requested schema OID/name, server version/version number, backend PID, session/current role, transaction mode, schema policy, exhaustive flag, start/end timestamps, effective settings `jsonb`, DDL-coordination declaration, inventory-complete flag; this page supplies the binary/source assumption |
| Snapshot | `snapshot_text text`, transaction start, transaction ID and observation time; count/plan rows also record transaction, snapshot, backend and role, and the integrity view checks consistency |
| Relation | Relation OID and captured qualified name, `relkind`, table AM OID/name, persistence, population/RLS flags, parent OIDs, inclusion/exclusion reason, accepted heap attempt ID/count, matching actual index OID, status, scope-gap flags |
| Candidate | Relation/index OIDs and qualified names, index AM, definition when safe to deparse, key expressions/order metadata, predicate/state flags, observed catalog xmin, structural eligibility, snapshot-usability state, exclusion reason, attempted/actually-checked flags, untested reason |
| Attempt | Attempt ID, intended and actual index OIDs, phase, query text, settings, snapshot reference, exact count `bigint`, heap count reference, `delta numeric`, executed plan `jsonb`, validation decision/reasons, start/capture/import timestamps and plan execution time, accepted/rejected/error fields |
| Error | `hic_error`: attempt ID, which leads to the relation and the intended candidate, plus stage, SQLSTATE, message, detail, hint and whether the snapshot survived. `hic_run_error`: run-level stage, SQLSTATE and message. Whether accepted evidence still covers a failed candidate is derived as `uncovered_error_attempts` in the relation records, not stored |
| Parent summary | `hic_parent_result`: descendant and physical-child counts, outside-schema descendants, `all_descendants_in_scope`, `all_descendants_supported` and `subtree_success`; per-child statuses come from the relation records |
| Schema summary | `hic_schema_result`: counts by relation status, unsupported table AMs, eligible/checked/matching/untested index totals over in-scope relations, observed mismatches and attempt errors, `inventory_complete`, `evidence_valid`, `heap_scope_all_pass`, `schema_success`, `exhaustive_coverage_complete` and `all_tested_indexes_match` |

A timestamp is not a snapshot identifier, and a transaction ID is not the snapshot's in-progress transaction list. Use the version-12 snapshot function rather than inventing a timestamp-based comparison. The implementation deliberately assigns a transaction ID with `txid_current()` at run initialization and records it with each count/import; its active-transaction guard then uses `txid_current_if_assigned()` to avoid assigning a new ID just to inspect a completed run. [func.sgml#transaction-and-snapshot-functions](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L19384-L19418)

Apply relation statuses in this order:

1. **NOT_APPLICABLE:** explicit object/scope classification excludes the object from physical heap checking.
2. **ERROR for an unresolved prerequisite failure:** discovery, identity, permission or snapshot integrity failed, or execution/capture failed and no valid baseline remains. An attempt's failure is not an irreversible relation verdict if a permitted retry establishes all required evidence under the same valid snapshot.
3. **NO_ELIGIBLE_INDEX:** inventory is complete and there is no supported eligible candidate and no unresolved eligibility candidate. Report excluded/unsupported indexes separately.
4. **PASS:** a valid heap count and at least one accepted eligible index count match. Stop by default. Errors or mismatches from other attempts remain in the attempt records; they do not logically refute this existential result.
5. **ERROR:** no match, and a required scan/inspection failed with an execution, permission or evidence-capture error. Preserve any other observed mismatches.
6. **MISMATCH:** a valid heap count exists, the eligible set is nonempty and known, every eligible candidate was actually checked successfully, and none matched.
7. **INCONCLUSIVE:** no match and no decisive error under the preceding rules, but a baseline path was rejected, or at least one eligible/provisionally eligible candidate remained untested or unresolved.

These rules deliberately distinguish an **observed index mismatch** from the relation-level **MISMATCH** verdict. One mismatch plus one untested candidate is `INCONCLUSIVE`, not a proof that no eligible index matches. A failed optional diagnostic after a count PASS has its own error record; it does not erase the established count equality.

For aggregation, a required error means a failure that leaves discovery, scope, snapshot, baseline or the asserted matching evidence unestablished. A failed other-index attempt becomes a retained warning once independent accepted evidence establishes PASS; it is not silently deleted. An error prevents complete candidate coverage only while that candidate has no independent accepted actual scan. A failed attempt followed by successful same-snapshot coverage can coexist with complete coverage, while the error remains visible.

Define schema aggregation with explicit booleans:

```text
heap_scope_all_pass =
    inventory complete
    AND at least one in-scope physical heap relation
    AND every such relation has status PASS
    AND no loss of the run's required snapshot/identity evidence

schema_success =
    heap_scope_all_pass
    AND no unresolved required error
    AND no unsupported local physical table-AM coverage gap
```

Any in-scope `MISMATCH`, `INCONCLUSIVE`, `NO_ELIGIBLE_INDEX`, or `ERROR` makes `schema_success` false. Zero applicable relations yields a “no applicable physical heap relations” report, not a vacuous green pass. Views, sequences, foreign tables and unpopulated materialized views are listed as expected exclusions and do not by themselves fail this explicitly local, populated-heap contract. An unsupported local table AM blocks unqualified schema success even though its object record is `NOT_APPLICABLE`; `heap_scope_all_pass` may still describe the supported subset.

Partition summaries must carry `all_descendants_in_scope` and `all_descendants_supported`. An outside-schema child under namespace-only mode does not fail the namespace check, but prevents presenting the parent's summary as a check of the entire partition tree. Likewise, a foreign partition is an explicit hole in any claimed all-children heap comparison. Never add a parent's summary to the physical relation denominator.

For exhaustive mode, retain the requested existential PASS semantics and add `all_tested_indexes_match`, `exhaustive_coverage_complete`, and anomaly counts, at both the relation and the schema level. A PASS with another index mismatch or attempt error is visibly qualified in the report. An optional stricter operational flag can require zero attempt errors and zero observed mismatches; do not silently redefine the requested objective.

### Concurrency performance and correctness limits

| Condition | Mechanism and implication |
|---|---|
| Concurrent INSERT/UPDATE/DELETE or abort | Both accepted scans use `S`, so later commits do not change the logical visible population. Old index entries can remain, but heap fetching applies `S` before returning a row. [mvcc.sgml#repeatable-read](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L454-L479) [indexam.c#visible-fetch](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L625-L632) |
| HOT update or pruning | One index root can lead to a later visible heap tuple or through a redirect. The heap AM resolves this; a raw root-TID census cannot substitute for it. [README.HOT#chain-and-redirect](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L48-L103) [heapam.c#HOT-search](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1559-L1622) |
| DDL, refresh, or membership change during inventory/checking | May invalidate identity, eligibility, coverage, or old-snapshot assumptions; require the coordinated window and reject violations. Read locks alone cover only conflicting operations on already locked objects. [lock.c#conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103) [mvcc.sgml#DDL-caveat](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L1689-L1702) |
| Many candidates or poorly correlated index order | One heap baseline plus potentially several full index traversals and heap fetches can be expensive. Ordered index access can perform substantially more scattered heap I/O than a sequential scan; sort alternatives may be cheaper. Early stopping and coverage-aware retry limits bound avoidable work. [indices.sgml#index-versus-sort](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L489-L502) [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635) |
| Long snapshot | Can hold back cleanup; observe duration and abort within an agreed operating window. With nondefault `old_snapshot_threshold`, cleanup can instead invalidate sufficiently old reads and cause an error. Its default is `-1`, disabled, and changing it requires server start; the checker records it and does not change it. [vacuum.c#cleanup-horizon](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L902-L912) [config.sgml#old-snapshot-threshold](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2378-L2396) |
| Shared corruption or missing historical data | Both scans may agree on an already incomplete heap. Counts do not validate key values, ordering, uniqueness, payload/TOAST contents, unreachable pages, other indexes, or historical completeness. A full executor scan is the observation defined by this contract, not an independent physical census. [indexam.c#heap-based-results](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L625-L632) [nbtsearch.c#endpoint-traversal](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsearch.c#L980-L997) |
| Heap visibility metadata is damaged | Even the sequential heap path can trust the page's `PD_ALL_VISIBLE` flag outside recovery. Disabling index-only scans does not make the baseline an independent reimplementation of visibility or a guarantee against damaged heap metadata. [heapam.c#page-visible-fast-path](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L428-L453) |
| Timeouts, cancellation, numeric overflow or storage errors | Record `ERROR` and preserve failure details. `count(*)` uses an `int8` transition and raises on overflow, rather than producing an arbitrary-precision count. [pg_aggregate.dat#count](../../../../raw/postgres-12/src/include/catalog/pg_aggregate.dat#L213-L219) [int8.c#int8inc-overflow](../../../../raw/postgres-12/src/backend/utils/adt/int8.c#L682-L705) |

The run's lifecycle is also part of the result:

| Event | What changes | Required action |
|---|---|---|
| First checking query | Acquires the retained repeatable-read snapshot | Record `S`; apply it to all later attempts in that transaction |
| An attempt fails and is rolled back to its savepoint | That attempt's writes disappear; the surrounding transaction can continue | Record the client error and confirm the run/snapshot still exists before retrying |
| Commit, full rollback, disconnect or transaction restart | Ends the old checking snapshot | Never compare a new index count with an old baseline; preserve exported report/transcript separately |
| Index rebuild, table rewrite, partition change or role change | Can change a premise of the saved inventory/plan | End or invalidate the affected run and rediscover |

The first and third rows follow the snapshot lifecycle; savepoint rollback is the recovery boundary for the second. The final row is a conservative checker policy. [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [rollback_to.sgml#savepoint](../../../../raw/postgres-12/doc/src/sgml/ref/rollback_to.sgml#L29-L55)

“No application-data writes” does not mean no disk writes: temporary result storage, executor work files, and ordinary heap maintenance such as pruning are separate from application DML. The read-only transaction contract itself explicitly does not prohibit every disk write. [set_transaction.sgml#read-only-boundary](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L126-L139) [heapam_handler.c#read-side-pruning](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L130-L148)

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

These are executor-visible [TIDs](../../../glossary.md#tid). After following HOT, the heap fetch updates the returned tuple's `t_self`, which supplies SQL `ctid`. Thus both bags identify the visible row version, rather than equating a raw index's chain-root TID with the visible successor. Materialization can be much larger than a count and can require substantial sorting/storage for `EXCEPT ALL`; it still does not validate stored index keys or search behavior for all predicates. [heapam.c#visible-HOT-member](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1582-L1619) [heapam_handler.c#returned-tid](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L148-L167) [heaptuple.c#ctid-system-attribute](../../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L636-L640)

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

Run this complete block once in a dedicated checking session, before starting its read-only checking transaction. It creates only checker-owned temporary objects and `LANGUAGE sql` helpers. It does not execute generated count statements, import client output, or catch exceptions automatically. The operator workflow below supplies those steps, and the follow-up's runner performs them unattended.

```sql
SET /* wiki_hic12_install_path */ search_path = pg_catalog, pg_temp;
SET /* wiki_hic12_install_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_install_lock_timeout */ lock_timeout = '2s';

CREATE /* wiki_hic12_config */ TEMP TABLE hic_config (
    singleton boolean PRIMARY KEY DEFAULT true CHECK (singleton),
    target_schema text NOT NULL CHECK (target_schema <> ''),
    ddl_stable boolean NOT NULL CHECK (ddl_stable),
    include_external_partitions boolean NOT NULL DEFAULT false,
    exhaustive boolean NOT NULL DEFAULT false
) USING heap;

CREATE /* wiki_hic12_run */ TEMP TABLE hic_run (
    run_id bigint PRIMARY KEY CHECK (run_id = 1),
    database_oid oid NOT NULL,
    database_name text NOT NULL,
    schema_oid oid NOT NULL CHECK (schema_oid <> 0),
    schema_name text NOT NULL,
    version_num integer NOT NULL CHECK (version_num = 120002),
    server_version text NOT NULL,
    backend_pid integer NOT NULL,
    session_role text NOT NULL,
    checking_role text NOT NULL,
    transaction_id bigint NOT NULL,
    snapshot_text text NOT NULL,
    transaction_started timestamptz NOT NULL,
    started timestamptz NOT NULL,
    finished timestamptz,
    isolation_level text NOT NULL CHECK (isolation_level = 'repeatable read'),
    read_only boolean NOT NULL CHECK (read_only),
    primary_server boolean NOT NULL CHECK (primary_server),
    ddl_stable boolean NOT NULL CHECK (ddl_stable),
    include_external_partitions boolean NOT NULL,
    exhaustive boolean NOT NULL,
    inventory_complete boolean NOT NULL DEFAULT false,
    invalid_reason text,
    settings jsonb NOT NULL
) USING heap;

CREATE /* wiki_hic12_objects */ TEMP TABLE hic_object (
    relation_oid oid PRIMARY KEY,
    namespace_oid oid NOT NULL,
    schema_name text NOT NULL,
    relation_name text NOT NULL,
    qualified_name text NOT NULL,
    relkind text NOT NULL,
    table_am_oid oid NOT NULL,
    table_am text,
    persistence text NOT NULL,
    populated boolean NOT NULL,
    is_partition boolean NOT NULL,
    row_security boolean NOT NULL,
    force_row_security boolean NOT NULL,
    parent_oids oid[] NOT NULL,
    relfilenode oid NOT NULL,
    in_heap_scope boolean NOT NULL,
    exclusion_reason text,
    unsupported_table_am boolean NOT NULL,
    permission_error text
) USING heap;

CREATE /* wiki_hic12_indexes */ TEMP TABLE hic_index (
    index_oid oid PRIMARY KEY,
    relation_oid oid NOT NULL REFERENCES hic_object(relation_oid),
    schema_name text NOT NULL,
    index_name text NOT NULL,
    qualified_name text NOT NULL,
    index_am_oid oid NOT NULL,
    index_am text NOT NULL,
    definition text,
    predicate text,
    indisvalid boolean NOT NULL,
    indisready boolean NOT NULL,
    indislive boolean NOT NULL,
    indcheckxmin boolean NOT NULL,
    catalog_xmin text NOT NULL,
    eligible boolean NOT NULL,
    exclusion_reason text,
    key_count integer NOT NULL,
    key_metadata jsonb NOT NULL,
    order_sql text,
    generation_reason text
) USING heap;

CREATE /* wiki_hic12_attempts */ TEMP TABLE hic_attempt (
    attempt_id bigint PRIMARY KEY CHECK (attempt_id > 0),
    relation_oid oid NOT NULL REFERENCES hic_object(relation_oid),
    phase text NOT NULL CHECK (phase IN ('heap', 'index')),
    intended_index_oid oid REFERENCES hic_index(index_oid),
    variant integer NOT NULL CHECK (variant IN (1, 2)),
    statement_text text NOT NULL,
    started timestamptz NOT NULL DEFAULT clock_timestamp(),
    CHECK ((phase = 'heap') = (intended_index_oid IS NULL))
) USING heap;

CREATE /* wiki_hic12_settings */ FUNCTION pg_temp.hic_settings()
RETURNS jsonb
LANGUAGE sql STABLE SECURITY INVOKER
AS $function$
    SELECT /* wiki_hic12_read_settings */ pg_catalog.jsonb_object_agg(name, pg_catalog.current_setting(name))
    FROM pg_catalog.unnest(ARRAY[
        'enable_seqscan', 'enable_indexscan', 'enable_indexonlyscan',
        'enable_bitmapscan', 'enable_sort', 'max_parallel_workers_per_gather',
        'row_security', 'search_path', 'quote_all_identifiers', 'statement_timeout', 'lock_timeout',
        'transaction_isolation', 'transaction_read_only', 'old_snapshot_threshold'
    ]) AS setting(name);
$function$;

CREATE /* wiki_hic12_counts */ TEMP TABLE hic_count (
    attempt_id bigint PRIMARY KEY REFERENCES hic_attempt(attempt_id),
    row_count bigint NOT NULL CHECK (row_count >= 0),
    transaction_id bigint NOT NULL DEFAULT txid_current(),
    snapshot_text text NOT NULL DEFAULT txid_current_snapshot()::text,
    backend_pid integer NOT NULL DEFAULT pg_backend_pid(),
    checking_role text NOT NULL DEFAULT current_user,
    captured timestamptz NOT NULL DEFAULT clock_timestamp(),
    settings jsonb NOT NULL DEFAULT pg_temp.hic_settings()
) USING heap;

CREATE /* wiki_hic12_plans */ TEMP TABLE hic_plan (
    attempt_id bigint PRIMARY KEY REFERENCES hic_attempt(attempt_id),
    document jsonb NOT NULL,
    transaction_id bigint NOT NULL DEFAULT txid_current(),
    snapshot_text text NOT NULL DEFAULT txid_current_snapshot()::text,
    backend_pid integer NOT NULL DEFAULT pg_backend_pid(),
    checking_role text NOT NULL DEFAULT current_user,
    imported timestamptz NOT NULL DEFAULT clock_timestamp()
) USING heap;

CREATE /* wiki_hic12_errors */ TEMP TABLE hic_error (
    attempt_id bigint PRIMARY KEY REFERENCES hic_attempt(attempt_id),
    stage text NOT NULL CHECK (stage IN ('execution', 'capture', 'inspection')),
    sqlstate text CHECK (sqlstate ~ '^[0-9A-Z]{5}$'),
    message text NOT NULL CHECK (message <> ''),
    detail text,
    hint text,
    snapshot_survived boolean NOT NULL,
    recorded timestamptz NOT NULL DEFAULT clock_timestamp()
) USING heap;

CREATE /* wiki_hic12_run_errors */ TEMP TABLE hic_run_error (
    event_id bigint PRIMARY KEY,
    requested_schema text NOT NULL,
    stage text NOT NULL CHECK (stage IN ('initialization', 'inventory', 'snapshot', 'connection')),
    sqlstate text CHECK (sqlstate ~ '^[0-9A-Z]{5}$'),
    message text NOT NULL CHECK (message <> ''),
    recorded timestamptz NOT NULL DEFAULT clock_timestamp()
) USING heap;

CREATE /* wiki_hic12_owned_objects */ TEMP TABLE hic_owned (
    relation_oid oid PRIMARY KEY
) USING heap;

CREATE /* wiki_hic12_partition_inventory */ TEMP TABLE hic_partition (
    parent_oid oid NOT NULL REFERENCES hic_object(relation_oid),
    descendant_oid oid NOT NULL,
    schema_name text NOT NULL,
    relation_name text NOT NULL,
    relkind text NOT NULL,
    included boolean NOT NULL,
    supported boolean NOT NULL,
    outside_schema boolean NOT NULL,
    PRIMARY KEY (parent_oid, descendant_oid)
) USING heap;

CREATE /* wiki_hic12_inventory */ TEMP VIEW hic_inventory AS
WITH RECURSIVE scope(relation_oid) AS (
    SELECT c.oid
    FROM pg_catalog.pg_class AS c
    CROSS JOIN pg_temp.hic_run AS r
    WHERE c.relnamespace = r.schema_oid
    UNION
    SELECT i.inhrelid
    FROM scope AS s
    JOIN pg_catalog.pg_class AS p ON p.oid = s.relation_oid AND p.relkind = 'p'
    JOIN pg_catalog.pg_inherits AS i ON i.inhparent = p.oid
    JOIN pg_catalog.pg_class AS c ON c.oid = i.inhrelid AND c.relispartition
    CROSS JOIN pg_temp.hic_run AS r
    WHERE r.include_external_partitions
)
SELECT c.oid AS relation_oid, n.oid AS namespace_oid,
       n.nspname::text AS schema_name, c.relname::text AS relation_name,
       format('%I.%I', n.nspname, c.relname) AS qualified_name,
       c.relkind::text AS relkind, c.relam AS table_am_oid, am.amname::text AS table_am,
       c.relpersistence::text AS persistence, c.relispopulated AS populated,
       c.relispartition AS is_partition, c.relrowsecurity AS row_security,
       c.relforcerowsecurity AS force_row_security,
       ARRAY(SELECT i.inhparent FROM pg_catalog.pg_inherits AS i
             WHERE i.inhrelid = c.oid ORDER BY i.inhparent) AS parent_oids,
       c.relfilenode,
       COALESCE(c.relkind IN ('r', 'm') AND am.amname = 'heap' AND am.amtype = 't'
                AND (c.relkind <> 'm' OR c.relispopulated), false) AS in_heap_scope,
       CASE
           WHEN c.relkind = 'p' THEN 'partitioned_parent_summary'
           WHEN c.relkind = 'm' AND NOT c.relispopulated THEN 'unpopulated_materialized_view'
           WHEN c.relkind IN ('r', 'm') AND (am.amname IS DISTINCT FROM 'heap' OR am.amtype IS DISTINCT FROM 't')
               THEN 'unsupported_table_access_method'
           WHEN c.relkind = 'v' THEN 'view'
           WHEN c.relkind = 'f' THEN 'foreign_table'
           WHEN c.relkind = 'S' THEN 'sequence'
           WHEN c.relkind IN ('i', 'I') THEN 'index_metadata'
           WHEN c.relkind NOT IN ('r', 'm') THEN 'outside_physical_heap_scope'
       END AS exclusion_reason,
       c.relkind IN ('r', 'm') AND (am.amname IS DISTINCT FROM 'heap' OR am.amtype IS DISTINCT FROM 't')
           AS unsupported_table_am,
       CASE
           WHEN pg_catalog.pg_is_other_temp_schema(n.oid) THEN 'unsupported_remote_temp'
           WHEN NOT pg_catalog.has_schema_privilege(n.oid, 'USAGE') THEN 'schema_usage_denied'
           WHEN c.relkind IN ('r', 'm') AND NOT pg_catalog.has_table_privilege(c.oid, 'SELECT')
               THEN 'table_select_denied'
       END AS permission_error
FROM scope AS s
JOIN pg_catalog.pg_class AS c ON c.oid = s.relation_oid
JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
LEFT JOIN pg_catalog.pg_am AS am ON am.oid = c.relam
WHERE NOT EXISTS (
    SELECT 1 FROM pg_temp.hic_owned AS own WHERE own.relation_oid = c.oid
);

CREATE /* wiki_hic12_untrusted_indexes */ TEMP VIEW hic_untrusted_indexes AS
SELECT DISTINCT d.objid AS index_oid
FROM pg_catalog.pg_depend AS d
JOIN pg_catalog.pg_index AS idx ON idx.indexrelid = d.objid
JOIN pg_temp.hic_object AS obj ON obj.relation_oid = idx.indrelid
LEFT JOIN pg_catalog.pg_proc AS fn
    ON d.refclassid = 'pg_catalog.pg_proc'::regclass AND fn.oid = d.refobjid
LEFT JOIN pg_catalog.pg_operator AS op
    ON d.refclassid = 'pg_catalog.pg_operator'::regclass AND op.oid = d.refobjid
LEFT JOIN pg_catalog.pg_type AS typ
    ON d.refclassid = 'pg_catalog.pg_type'::regclass AND typ.oid = d.refobjid
LEFT JOIN pg_catalog.pg_attribute AS attr
    ON d.refclassid = 'pg_catalog.pg_class'::regclass
   AND d.refobjid = idx.indrelid AND d.refobjsubid >= 0
   AND attr.attrelid = d.refobjid AND attr.attnum > 0 AND NOT attr.attisdropped
   AND (d.refobjsubid = 0 OR attr.attnum = d.refobjsubid)
LEFT JOIN pg_catalog.pg_type AS attrtype ON attrtype.oid = attr.atttypid
WHERE d.classid = 'pg_catalog.pg_class'::regclass
  AND (
      (fn.oid IS NOT NULL AND (fn.oid >= 16384 OR fn.pronamespace <> 'pg_catalog'::regnamespace))
      OR (op.oid IS NOT NULL AND (op.oid >= 16384 OR op.oprnamespace <> 'pg_catalog'::regnamespace))
      OR (typ.oid >= 16384 AND typ.typtype <> 'e')
      OR (attrtype.oid >= 16384 AND attrtype.typtype <> 'e')
  )
UNION
SELECT i.indexrelid
FROM pg_temp.hic_object AS obj
JOIN pg_catalog.pg_index AS i ON i.indrelid = obj.relation_oid
JOIN pg_catalog.pg_attribute AS a ON a.attrelid = i.indexrelid AND a.attnum > 0
JOIN pg_catalog.pg_type AS typ ON typ.oid = a.atttypid
WHERE a.atttypid >= 16384 AND typ.typtype <> 'e';

CREATE /* wiki_hic12_key_inventory */ TEMP VIEW hic_index_keys AS
SELECT i.indexrelid AS index_oid, k.n AS key_number,
       key_text.expression_sql,
       a.atttypid AS key_type_oid, opc.oid AS opclass_oid,
       opn.nspname::text AS opclass_schema, opc.opcname::text AS opclass_name,
       opc.opcfamily AS opfamily_oid, i.indcollation[k.n - 1] AS collation_oid,
       cn.nspname::text AS collation_schema, coll.collname::text AS collation_name,
       (i.indoption[k.n - 1] & 1) <> 0 AS descending,
       (i.indoption[k.n - 1] & 2) <> 0 AS nulls_first,
       ordering.oprname::text AS operator_name,
       ordering.operator_schema,
       CASE WHEN ordering.oid IS NOT NULL
                  AND key_text.expression_sql IS NOT NULL
                  AND opn.nspname = 'pg_catalog'
                  AND opc.oid < 16384
                  AND (a.atttypid < 16384 OR typ.typtype = 'e')
                  AND (i.indcollation[k.n - 1] = 0 OR coll.oid IS NOT NULL)
            THEN format('%s%s USING OPERATOR(%I.%s) NULLS %s',
                  CASE WHEN i.indkey[k.n - 1] = 0
                            AND key_text.expression_sql ~ '^[(]*[0-9]+([.][0-9]+)?[)]*$'
                       THEN format('CAST(%s AS %s)', key_text.expression_sql,
                                   pg_catalog.format_type(a.atttypid, NULL))
                       ELSE format('(%s)', key_text.expression_sql) END,
                  CASE WHEN coll.oid IS NULL THEN ''
                       ELSE format(' COLLATE %I.%I', cn.nspname, coll.collname) END,
                  ordering.operator_schema, ordering.oprname,
                  CASE WHEN (i.indoption[k.n - 1] & 2) <> 0 THEN 'FIRST' ELSE 'LAST' END)
       END AS order_part
FROM pg_temp.hic_object AS obj
JOIN pg_catalog.pg_index AS i ON i.indrelid = obj.relation_oid
JOIN pg_catalog.pg_class AS ic ON ic.oid = i.indexrelid
LEFT JOIN pg_temp.hic_untrusted_indexes AS u ON u.index_oid = i.indexrelid
CROSS JOIN LATERAL pg_catalog.generate_series(1, i.indnkeyatts::integer) AS k(n)
CROSS JOIN LATERAL (
    SELECT /* wiki_hic12_key_deparse */ CASE WHEN obj.in_heap_scope AND ic.relam = 403 AND u.index_oid IS NULL
                THEN pg_catalog.pg_get_indexdef(i.indexrelid, k.n, false) END AS expression_sql
) AS key_text
JOIN pg_catalog.pg_attribute AS a ON a.attrelid = i.indexrelid AND a.attnum = k.n
JOIN pg_catalog.pg_type AS typ ON typ.oid = a.atttypid
JOIN pg_catalog.pg_opclass AS opc ON opc.oid = i.indclass[k.n - 1]
JOIN pg_catalog.pg_namespace AS opn ON opn.oid = opc.opcnamespace
LEFT JOIN pg_catalog.pg_collation AS coll ON coll.oid = i.indcollation[k.n - 1]
LEFT JOIN pg_catalog.pg_namespace AS cn ON cn.oid = coll.collnamespace
LEFT JOIN LATERAL (
    SELECT op.oid, op.oprname, ns.nspname::text AS operator_schema
    FROM pg_catalog.pg_amop AS ao
    JOIN pg_catalog.pg_operator AS op ON op.oid = ao.amopopr
    JOIN pg_catalog.pg_namespace AS ns ON ns.oid = op.oprnamespace
    JOIN pg_catalog.pg_proc AS proc ON proc.oid = op.oprcode
    WHERE ao.amopfamily = opc.opcfamily
      AND ao.amoplefttype = opc.opcintype AND ao.amoprighttype = opc.opcintype
      AND ao.amopstrategy = CASE WHEN (i.indoption[k.n - 1] & 1) <> 0 THEN 5 ELSE 1 END
      AND ao.amoppurpose = 's' AND ao.amopmethod = 403
      AND ns.nspname = 'pg_catalog' AND op.oid < 16384
      AND proc.pronamespace = 'pg_catalog'::regnamespace AND proc.oid < 16384
    ORDER BY op.oid LIMIT 1
) AS ordering ON true;

CREATE /* wiki_hic12_eligibility */ FUNCTION pg_temp.hic_index_eligible(
    p_kind text, p_am_oid oid, p_am_name text, p_am_type text,
    p_partial boolean, p_valid boolean, p_ready boolean, p_live boolean
)
RETURNS boolean
LANGUAGE sql IMMUTABLE SECURITY INVOKER
SET search_path = pg_catalog, pg_temp
AS $function$
    SELECT /* wiki_hic12_eligibility_test */ COALESCE(
        p_kind = 'i' AND p_am_oid = 403 AND p_am_name = 'btree' AND p_am_type = 'i'
        AND NOT p_partial AND p_valid AND p_ready AND p_live, false
    );
$function$;

CREATE /* wiki_hic12_candidate_inventory */ TEMP VIEW hic_candidates AS
WITH metadata AS (
    SELECT k.index_oid,
           jsonb_agg(to_jsonb(k) - 'index_oid' ORDER BY key_number) AS keys,
           bool_and(k.order_part IS NOT NULL) AS ordering_supported,
           string_agg(k.order_part, ', ' ORDER BY key_number) AS order_sql,
           count(*) AS key_count
    FROM pg_temp.hic_index_keys AS k GROUP BY k.index_oid
)
SELECT c.oid AS index_oid, i.indrelid AS relation_oid,
       n.nspname::text AS schema_name, c.relname::text AS index_name,
       format('%I.%I', n.nspname, c.relname) AS qualified_name,
       c.relam AS index_am_oid, am.amname::text AS index_am,
       CASE WHEN obj.in_heap_scope AND am.oid = 403 AND u.index_oid IS NULL
            THEN pg_catalog.pg_get_indexdef(c.oid) END AS definition,
       CASE WHEN obj.in_heap_scope AND am.oid = 403 AND u.index_oid IS NULL
            THEN pg_catalog.pg_get_expr(i.indpred, i.indrelid, false)
            ELSE i.indpred::text END AS predicate,
       i.indisvalid, i.indisready, i.indislive, i.indcheckxmin, i.xmin::text AS catalog_xmin,
       pg_temp.hic_index_eligible(c.relkind::text, am.oid, am.amname::text, am.amtype::text,
                                i.indpred IS NOT NULL, i.indisvalid, i.indisready, i.indislive) AS eligible,
       CASE WHEN c.relkind <> 'i' THEN 'nonphysical_index'
            WHEN am.oid <> 403 OR am.amname <> 'btree' OR am.amtype <> 'i' THEN 'unsupported_index_am'
            WHEN i.indpred IS NOT NULL THEN 'partial_index'
            WHEN NOT i.indisvalid THEN 'invalid_index'
            WHEN NOT i.indisready THEN 'index_not_ready'
            WHEN NOT i.indislive THEN 'index_not_live'
       END AS exclusion_reason,
       i.indnkeyatts::integer AS key_count, COALESCE(m.keys, '[]'::jsonb) AS key_metadata,
       CASE WHEN u.index_oid IS NULL AND m.ordering_supported AND m.key_count = i.indnkeyatts
            THEN m.order_sql END AS order_sql,
       CASE WHEN NOT obj.in_heap_scope THEN 'relation_outside_physical_heap_scope'
            WHEN u.index_oid IS NOT NULL THEN 'unreviewed_user_code_or_type'
            WHEN m.ordering_supported IS DISTINCT FROM true OR m.key_count IS DISTINCT FROM i.indnkeyatts::bigint
                THEN 'ordering_not_represented_by_generator'
       END AS generation_reason
FROM pg_temp.hic_object AS obj
JOIN pg_catalog.pg_index AS i ON i.indrelid = obj.relation_oid
JOIN pg_catalog.pg_class AS c ON c.oid = i.indexrelid
JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
JOIN pg_catalog.pg_am AS am ON am.oid = c.relam
LEFT JOIN metadata AS m ON m.index_oid = c.oid
LEFT JOIN pg_temp.hic_untrusted_indexes AS u ON u.index_oid = c.oid;

CREATE /* wiki_hic12_plan_nodes */ FUNCTION pg_temp.hic_plan_nodes(p_document jsonb)
RETURNS TABLE (path text[], node jsonb)
LANGUAGE sql IMMUTABLE SECURITY INVOKER
SET search_path = pg_catalog, pg_temp
AS $function$
    WITH /* wiki_hic12_walk */ RECURSIVE walk(path, node) AS (
        SELECT ARRAY['0', 'Plan']::text[], p_document #> '{0,Plan}'
        WHERE jsonb_typeof(p_document #> '{0,Plan}') = 'object'
        UNION ALL
        SELECT w.path || ARRAY['Plans', (c.ordinality - 1)::text], c.value
        FROM walk AS w
        CROSS JOIN LATERAL jsonb_array_elements(
            CASE WHEN jsonb_typeof(w.node -> 'Plans') = 'array'
                 THEN w.node -> 'Plans' ELSE '[]'::jsonb END
        ) WITH ORDINALITY AS c(value, ordinality)
    )
    SELECT path, node FROM walk;
$function$;

CREATE /* wiki_hic12_check_plan */ FUNCTION pg_temp.hic_check_plan(
    p_document jsonb,
    p_phase text,
    p_schema text,
    p_relation text,
    p_capture_schema text
)
RETURNS TABLE (valid boolean, reasons text[], actual_index_name text)
LANGUAGE sql IMMUTABLE SECURITY INVOKER
ROWS 1
SET search_path = pg_catalog, pg_temp
AS $function$
    WITH /* wiki_hic12_plan_checks */ nodes AS MATERIALIZED (
        SELECT path, node,
               node ->> 'Node Type' AS kind,
               CASE WHEN jsonb_typeof(node -> 'Plans') = 'array'
                    THEN jsonb_array_length(node -> 'Plans')
                    WHEN NOT node ? 'Plans' THEN 0 ELSE -1 END AS children
        FROM pg_temp.hic_plan_nodes(p_document)
    ),
    root AS (
        SELECT p_document #> '{0,Plan}' AS node
    ),
    checks(issue, ok) AS (
        SELECT 'document_shape',
            CASE WHEN jsonb_typeof(p_document) = 'array'
                 THEN jsonb_array_length(p_document) = 1
                      AND jsonb_typeof(p_document -> 0) = 'object'
                 ELSE false END
        UNION ALL
        SELECT 'phase', p_phase IN ('heap', 'index')
        UNION ALL
        SELECT 'execution_summary',
            jsonb_typeof(p_document #> '{0,Execution Time}') = 'number'
            AND p_document #> '{0,Execution Time}' >= '0'::jsonb
        UNION ALL
        SELECT 'capture_target',
            node ->> 'Node Type' = 'ModifyTable'
            AND node ->> 'Operation' = 'Insert'
            AND node ->> 'Schema' = p_capture_schema
            AND node ->> 'Relation Name' = 'hic_count'
            AND node -> 'Actual Rows' = '0'::jsonb
        FROM root
        UNION ALL
        SELECT 'one_insert', count(*) FILTER (WHERE kind = 'ModifyTable') = 1
        FROM nodes
        UNION ALL
        SELECT 'one_aggregate', count(*) FILTER (WHERE kind = 'Aggregate') = 1
        FROM nodes
        UNION ALL
        SELECT 'one_source', count(*) FILTER (WHERE kind IN ('Seq Scan', 'Index Scan')) = 1
        FROM nodes
        UNION ALL
        SELECT 'node_shape', jsonb_typeof(node) = 'object'
            AND kind IN ('ModifyTable', 'Aggregate', 'Seq Scan', 'Index Scan',
                         'Sort', 'Result', 'Subquery Scan', 'Materialize')
            AND children = CASE WHEN kind IN ('Seq Scan', 'Index Scan') THEN 0 ELSE 1 END
        FROM nodes
        UNION ALL
        SELECT 'executed_once', node -> 'Actual Loops' = '1'::jsonb
            AND jsonb_typeof(node -> 'Actual Rows') = 'number'
            AND node -> 'Actual Rows' >= '0'::jsonb
        FROM nodes
        UNION ALL
        SELECT 'serial_plan', COALESCE(node -> 'Parallel Aware', 'false'::jsonb) = 'false'::jsonb
            AND NOT node ?| ARRAY['Workers', 'Workers Planned', 'Workers Launched']
        FROM nodes
        UNION ALL
        SELECT 'unrestricted_plan',
            NOT node ?| ARRAY['Filter', 'Index Cond', 'Recheck Cond', 'Join Filter',
                              'One-Time Filter', 'Rows Removed by Filter',
                              'Rows Removed by Index Recheck', 'Group Key',
                              'Grouping Sets', 'Hash Cond', 'Merge Cond']
            AND COALESCE(node ->> 'Parent Relationship', '') NOT IN ('InitPlan', 'SubPlan')
            AND NOT node ?| ARRAY['InitPlan', 'SubPlan']
        FROM nodes
        UNION ALL
        SELECT 'plain_count', node ->> 'Strategy' = 'Plain'
            AND node ->> 'Partial Mode' = 'Simple'
            AND node -> 'Actual Rows' = '1'::jsonb
            AND node -> 'Output' @> '["count(*)"]'::jsonb
        FROM nodes WHERE kind = 'Aggregate'
        UNION ALL
        SELECT 'source_identity', node ->> 'Schema' = p_schema
            AND node ->> 'Relation Name' = p_relation
            AND kind = CASE p_phase WHEN 'heap' THEN 'Seq Scan' ELSE 'Index Scan' END
        FROM nodes WHERE kind IN ('Seq Scan', 'Index Scan')
        UNION ALL
        SELECT 'index_identity', COALESCE(node ->> 'Index Name', '') <> ''
            AND node ->> 'Scan Direction' IN ('Forward', 'Backward')
        FROM nodes WHERE kind = 'Index Scan'
    ),
    failures AS (
        SELECT ARRAY(
            SELECT DISTINCT issue FROM checks
            WHERE NOT COALESCE(ok, false) ORDER BY issue
        ) AS reasons
    )
    SELECT cardinality(f.reasons) = 0, f.reasons,
           (SELECT node ->> 'Index Name' FROM nodes WHERE kind = 'Index Scan' LIMIT 1)
    FROM failures AS f;
$function$;

CREATE /* wiki_hic12_start_script */ TEMP VIEW hic_start_script AS
SELECT $script$
BEGIN /* wiki_hic12_begin */ ISOLATION LEVEL REPEATABLE READ READ ONLY;
SET /* wiki_hic12_scope_path */ LOCAL search_path = pg_catalog, pg_temp;
SET /* wiki_hic12_scope_quote */ LOCAL quote_all_identifiers = off;
SET /* wiki_hic12_scope_rls */ LOCAL row_security = off;
SET /* wiki_hic12_scope_parallel */ LOCAL max_parallel_workers_per_gather = 0;
SET /* wiki_hic12_scope_timeout */ LOCAL statement_timeout = '15min';
SET /* wiki_hic12_scope_lock_timeout */ LOCAL lock_timeout = '2s';
SET /* wiki_hic12_scope_seq */ LOCAL enable_seqscan = on;
SET /* wiki_hic12_scope_index */ LOCAL enable_indexscan = on;
SET /* wiki_hic12_scope_only */ LOCAL enable_indexonlyscan = on;
SET /* wiki_hic12_scope_bitmap */ LOCAL enable_bitmapscan = on;
SET /* wiki_hic12_scope_sort */ LOCAL enable_sort = on;
INSERT /* wiki_hic12_start_run */ INTO pg_temp.hic_run (
    run_id, database_oid, database_name, schema_oid, schema_name,
    version_num, server_version, backend_pid, session_role, checking_role,
    transaction_id, snapshot_text, transaction_started, started,
    isolation_level, read_only, primary_server, ddl_stable,
    include_external_partitions, exhaustive, settings
)
SELECT 1, db.oid, current_database(), COALESCE(n.oid, 0::oid), cfg.target_schema,
       current_setting('server_version_num')::integer, version(), pg_backend_pid(),
       session_user, current_user, txid_current(), txid_current_snapshot()::text,
       transaction_timestamp(), clock_timestamp(), current_setting('transaction_isolation'),
       current_setting('transaction_read_only')::boolean, NOT pg_is_in_recovery(),
       cfg.ddl_stable, cfg.include_external_partitions, cfg.exhaustive,
       pg_temp.hic_settings()
FROM pg_temp.hic_config AS cfg
CROSS JOIN pg_catalog.pg_database AS db
LEFT JOIN pg_catalog.pg_namespace AS n ON n.nspname = cfg.target_schema
WHERE db.datname = current_database();
INSERT /* wiki_hic12_capture_objects */ INTO pg_temp.hic_object
SELECT * FROM pg_temp.hic_inventory ORDER BY relation_oid;
INSERT /* wiki_hic12_capture_indexes */ INTO pg_temp.hic_index
SELECT * FROM pg_temp.hic_candidates ORDER BY index_oid;
INSERT /* wiki_hic12_capture_partitions */ INTO pg_temp.hic_partition
WITH RECURSIVE tree(parent_oid, descendant_oid) AS (
    SELECT obj.relation_oid, inh.inhrelid
    FROM pg_temp.hic_object AS obj
    JOIN pg_catalog.pg_inherits AS inh ON inh.inhparent = obj.relation_oid
    WHERE obj.relkind = 'p'
    UNION
    SELECT tree.parent_oid, inh.inhrelid
    FROM tree
    JOIN pg_catalog.pg_class AS p ON p.oid = tree.descendant_oid AND p.relkind = 'p'
    JOIN pg_catalog.pg_inherits AS inh ON inh.inhparent = p.oid
)
SELECT tree.parent_oid, c.oid, n.nspname::text, c.relname::text, c.relkind::text,
       obj.relation_oid IS NOT NULL,
       COALESCE(c.relkind = 'p' OR (c.relkind = 'r' AND am.oid = 2 AND am.amname = 'heap'), false),
       c.relnamespace <> r.schema_oid
FROM tree
JOIN pg_catalog.pg_class AS c ON c.oid = tree.descendant_oid
JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
LEFT JOIN pg_catalog.pg_am AS am ON am.oid = c.relam AND am.amtype = 't'
LEFT JOIN pg_temp.hic_object AS obj ON obj.relation_oid = c.oid
CROSS JOIN pg_temp.hic_run AS r;
UPDATE /* wiki_hic12_inventory_complete */ pg_temp.hic_run SET inventory_complete = true;
$script$::text AS sql_text;

CREATE /* wiki_hic12_run_integrity */ TEMP VIEW hic_run_integrity AS
SELECT r.run_id,
       r.inventory_complete AND r.invalid_reason IS NULL
       AND NOT EXISTS (SELECT 1 FROM pg_temp.hic_run_error)
       AND NOT EXISTS (
           SELECT 1 FROM pg_temp.hic_error AS e WHERE NOT e.snapshot_survived
       )
       AND NOT EXISTS (
           SELECT 1 FROM (
               SELECT transaction_id, snapshot_text, backend_pid, checking_role FROM pg_temp.hic_count
               UNION ALL
               SELECT transaction_id, snapshot_text, backend_pid, checking_role FROM pg_temp.hic_plan
           ) AS evidence
           WHERE evidence.transaction_id <> r.transaction_id
              OR evidence.snapshot_text <> r.snapshot_text
              OR evidence.backend_pid <> r.backend_pid
              OR evidence.checking_role <> r.checking_role
       ) AS evidence_valid,
       r.transaction_id = txid_current_if_assigned()
           AND r.snapshot_text = txid_current_snapshot()::text
           AND r.backend_pid = pg_backend_pid()
           AND r.checking_role = current_user
           AND current_setting('transaction_isolation') = 'repeatable read'
           AND current_setting('transaction_read_only') = 'on'
           AND r.finished IS NULL AS active_snapshot
FROM pg_temp.hic_run AS r;

CREATE /* wiki_hic12_explain_identifier */ FUNCTION pg_temp.hic_explain_identifier(p_name text)
RETURNS text
LANGUAGE sql IMMUTABLE SECURITY INVOKER
SET search_path = pg_catalog, pg_temp
SET quote_all_identifiers = off
AS $function$
    SELECT /* wiki_hic12_canonical_identifier */ quote_ident(p_name);
$function$;

CREATE /* wiki_hic12_attempt_results */ TEMP VIEW hic_attempt_result AS
WITH checked AS (
    SELECT a.*, obj.qualified_name AS relation_name,
           c.row_count, c.captured, c.settings, p.document, p.imported,
           e.sqlstate, e.message, e.stage AS error_stage,
           v.valid AS plan_valid, v.reasons AS plan_reasons,
           v.actual_index_name,
           CASE WHEN resolved.matches = 1 THEN resolved.index_oid END AS actual_index_oid,
           c.attempt_id IS NOT NULL AS count_present, p.attempt_id IS NOT NULL AS plan_present,
           e.attempt_id IS NOT NULL AS error_present,
           integrity.evidence_valid,
           EXISTS (
               SELECT 1 FROM pg_temp.hic_plan_nodes(p.document) AS binding
               WHERE binding.node ->> 'Node Type' = 'Aggregate'
                 AND binding.node #>> '{Output,0}' = format('%L::bigint', a.attempt_id::text)
                 AND binding.node #>> '{Output,1}' = 'count(*)'
           ) AS binding_valid,
           COALESCE(
               c.settings ->> 'row_security' = 'off'
               AND c.settings ->> 'transaction_isolation' = 'repeatable read'
               AND c.settings ->> 'transaction_read_only' = 'on'
               AND c.settings ->> 'max_parallel_workers_per_gather' = '0'
               AND c.settings ->> 'enable_indexonlyscan' = 'off'
               AND c.settings ->> 'enable_bitmapscan' = 'off'
               AND c.settings ->> 'enable_seqscan' = CASE a.phase WHEN 'heap' THEN 'on' ELSE 'off' END
               AND c.settings ->> 'enable_indexscan' = CASE a.phase WHEN 'heap' THEN 'off' ELSE 'on' END
               AND c.settings ->> 'quote_all_identifiers' = 'off'
               AND c.settings ->> 'search_path' = 'pg_catalog, pg_temp',
               false
           ) AS settings_valid
    FROM pg_temp.hic_attempt AS a
    JOIN pg_temp.hic_object AS obj ON obj.relation_oid = a.relation_oid
    CROSS JOIN pg_temp.hic_run_integrity AS integrity
    LEFT JOIN pg_temp.hic_count AS c ON c.attempt_id = a.attempt_id
    LEFT JOIN pg_temp.hic_plan AS p ON p.attempt_id = a.attempt_id
    LEFT JOIN pg_temp.hic_error AS e ON e.attempt_id = a.attempt_id
    CROSS JOIN LATERAL pg_temp.hic_check_plan(
        p.document, a.phase, obj.schema_name, obj.relation_name,
        (SELECT nspname::text FROM pg_catalog.pg_namespace WHERE oid = pg_my_temp_schema())
    ) AS v
    LEFT JOIN LATERAL (
        SELECT count(*) AS matches, (array_agg(i.index_oid ORDER BY i.index_oid))[1] AS index_oid
        FROM pg_temp.hic_index AS i
        WHERE i.relation_oid = a.relation_oid
          AND pg_temp.hic_explain_identifier(i.index_name) = v.actual_index_name AND i.eligible
    ) AS resolved ON true
)
SELECT checked.*,
       evidence_valid AND count_present AND plan_present AND NOT error_present
           AND plan_valid AND settings_valid AND binding_valid
           AND (phase = 'heap' OR actual_index_oid IS NOT NULL) AS accepted,
       COALESCE(plan_reasons, ARRAY[]::text[]) ||
       array_remove(ARRAY[
           CASE WHEN NOT evidence_valid THEN 'run_evidence_invalid' END,
           CASE WHEN NOT count_present THEN 'count_missing' END,
           CASE WHEN NOT plan_present THEN 'plan_missing' END,
           CASE WHEN error_present THEN 'attempt_error' END,
           CASE WHEN plan_present AND NOT binding_valid THEN 'attempt_marker_not_confirmed' END,
           CASE WHEN count_present AND NOT settings_valid THEN 'scan_profile_not_confirmed' END,
           CASE WHEN phase = 'index' AND plan_present AND actual_index_oid IS NULL
                THEN 'actual_index_not_uniquely_eligible' END
       ], NULL) AS rejection_reasons,
       EXTRACT(epoch FROM captured - started) AS elapsed_to_capture_seconds,
       CASE WHEN jsonb_typeof(document #> '{0,Execution Time}') = 'number'
            THEN (document #>> '{0,Execution Time}')::numeric END AS execution_ms
FROM checked;

CREATE /* wiki_hic12_relation_results */ TEMP VIEW hic_relation_result AS
WITH summaries AS (
    SELECT obj.*,
           integrity.evidence_valid,
           COALESCE(candidates.eligible_count, 0) AS eligible_indexes,
           h.attempt_id AS heap_attempt_id, h.row_count AS heap_count,
           h.distinct_counts AS distinct_heap_counts,
           COALESCE(observed.checked_count, 0) AS checked_indexes,
           COALESCE(observed.matching_count, 0) AS matching_indexes,
           COALESCE(observed.mismatching_count, 0) AS mismatching_attempts,
           observed.matching_index_oid,
           COALESCE(observed.error_count, 0) AS error_attempts,
           COALESCE(observed.uncovered_error_count, 0) AS uncovered_error_attempts
    FROM pg_temp.hic_object AS obj
    CROSS JOIN pg_temp.hic_run_integrity AS integrity
    LEFT JOIN LATERAL (
        SELECT count(*) AS eligible_count FROM pg_temp.hic_index AS i
        WHERE i.relation_oid = obj.relation_oid AND i.eligible
    ) AS candidates ON true
    LEFT JOIN LATERAL (
        SELECT min(a.attempt_id) AS attempt_id, min(a.row_count) AS row_count,
               count(DISTINCT a.row_count) AS distinct_counts
        FROM pg_temp.hic_attempt_result AS a
        WHERE a.relation_oid = obj.relation_oid AND a.phase = 'heap' AND a.accepted
    ) AS h ON true
    LEFT JOIN LATERAL (
        SELECT count(DISTINCT a.actual_index_oid) FILTER (WHERE a.accepted AND a.phase = 'index') AS checked_count,
               count(DISTINCT a.actual_index_oid) FILTER (
                   WHERE a.accepted AND a.phase = 'index' AND a.row_count = h.row_count
               ) AS matching_count,
               count(*) FILTER (
                   WHERE a.accepted AND a.phase = 'index' AND a.row_count <> h.row_count
               ) AS mismatching_count,
               (array_agg(a.actual_index_oid ORDER BY a.actual_index_oid) FILTER (
                   WHERE a.accepted AND a.phase = 'index' AND a.row_count = h.row_count
               ))[1] AS matching_index_oid,
               count(*) FILTER (WHERE a.error_present) AS error_count,
               count(*) FILTER (
                   WHERE a.error_present AND (
                       (a.phase = 'heap' AND h.row_count IS NULL)
                       OR (a.phase = 'index' AND NOT EXISTS (
                           SELECT 1 FROM pg_temp.hic_attempt_result AS recovered
                           WHERE recovered.relation_oid = a.relation_oid
                             AND recovered.phase = 'index' AND recovered.accepted
                             AND recovered.actual_index_oid = a.intended_index_oid
                       ))
                   )
               ) AS uncovered_error_count
        FROM pg_temp.hic_attempt_result AS a WHERE a.relation_oid = obj.relation_oid
    ) AS observed ON true
)
SELECT summaries.*,
       eligible_indexes - checked_indexes AS untested_indexes,
       CASE
           WHEN NOT in_heap_scope THEN 'NOT_APPLICABLE'
           WHEN NOT evidence_valid OR permission_error IS NOT NULL OR distinct_heap_counts > 1 THEN 'ERROR'
           WHEN eligible_indexes = 0 THEN 'NO_ELIGIBLE_INDEX'
           WHEN heap_count IS NOT NULL AND matching_indexes > 0 THEN 'PASS'
           WHEN uncovered_error_attempts > 0 THEN 'ERROR'
           WHEN heap_count IS NOT NULL AND checked_indexes = eligible_indexes THEN 'MISMATCH'
           ELSE 'INCONCLUSIVE'
       END AS status,
       checked_indexes = eligible_indexes AS exhaustive_coverage_complete,
       checked_indexes > 0 AND mismatching_attempts = 0 AND heap_count IS NOT NULL AS all_tested_indexes_match
FROM summaries;

CREATE /* wiki_hic12_candidate_results */ TEMP VIEW hic_candidate_result AS
SELECT i.*, COALESCE(a.attempts, 0) AS intended_attempts,
       COALESCE(used.checked, false) AS actually_checked,
       CASE WHEN NOT i.eligible THEN i.exclusion_reason
            WHEN used.checked THEN NULL
            WHEN i.generation_reason IS NOT NULL THEN i.generation_reason
            WHEN r.status = 'PASS' AND NOT run.exhaustive THEN 'stopped_after_first_match'
            WHEN a.attempts >= 2 THEN 'attempt_budget_exhausted_without_actual_coverage'
            ELSE 'not_yet_checked'
       END AS untested_reason,
       CASE WHEN used.checked THEN 'accepted_by_executed_plan'
            WHEN i.indcheckxmin AND i.eligible THEN 'indcheckxmin_unresolved'
            ELSE 'not_established'
       END AS snapshot_usability
FROM pg_temp.hic_index AS i
CROSS JOIN pg_temp.hic_run AS run
JOIN pg_temp.hic_relation_result AS r ON r.relation_oid = i.relation_oid
LEFT JOIN LATERAL (
    SELECT count(*) AS attempts FROM pg_temp.hic_attempt AS a WHERE a.intended_index_oid = i.index_oid
) AS a ON true
LEFT JOIN LATERAL (
    SELECT bool_or(a.accepted) AS checked FROM pg_temp.hic_attempt_result AS a
    WHERE a.actual_index_oid = i.index_oid AND a.phase = 'index'
) AS used ON true;

CREATE /* wiki_hic12_schema_results */ TEMP VIEW hic_schema_result AS
SELECT run.run_id, run.database_name, run.schema_name, run.snapshot_text,
       run.started, run.finished, run.inventory_complete, integrity.evidence_valid,
       count(*) FILTER (WHERE r.in_heap_scope) AS physical_relations,
       count(*) FILTER (WHERE r.status = 'PASS') AS passed_relations,
       count(*) FILTER (WHERE r.status = 'MISMATCH') AS mismatched_relations,
       count(*) FILTER (WHERE r.status = 'INCONCLUSIVE') AS inconclusive_relations,
       count(*) FILTER (WHERE r.status = 'NO_ELIGIBLE_INDEX') AS no_index_relations,
       count(*) FILTER (WHERE r.status = 'ERROR') AS error_relations,
       count(*) FILTER (WHERE r.status = 'NOT_APPLICABLE') AS excluded_objects,
       count(*) FILTER (WHERE r.unsupported_table_am) AS unsupported_table_ams,
       COALESCE(sum(r.eligible_indexes) FILTER (WHERE r.in_heap_scope), 0) AS eligible_indexes,
       COALESCE(sum(r.checked_indexes) FILTER (WHERE r.in_heap_scope), 0) AS checked_indexes,
       COALESCE(sum(r.matching_indexes) FILTER (WHERE r.in_heap_scope), 0) AS matching_indexes,
       COALESCE(sum(r.untested_indexes) FILTER (WHERE r.in_heap_scope), 0) AS untested_indexes,
       integrity.evidence_valid
           AND count(*) FILTER (WHERE r.in_heap_scope) > 0
           AND bool_and(r.status = 'PASS') FILTER (WHERE r.in_heap_scope) AS heap_scope_all_pass,
       integrity.evidence_valid
           AND count(*) FILTER (WHERE r.in_heap_scope) > 0
           AND bool_and(r.status = 'PASS') FILTER (WHERE r.in_heap_scope)
           AND count(*) FILTER (WHERE r.unsupported_table_am) = 0 AS schema_success,
       COALESCE(bool_and(r.exhaustive_coverage_complete) FILTER (WHERE r.in_heap_scope), false)
           AS exhaustive_coverage_complete,
       COALESCE(sum(r.checked_indexes) FILTER (WHERE r.in_heap_scope), 0) > 0
           AND COALESCE(sum(r.mismatching_attempts) FILTER (WHERE r.in_heap_scope), 0) = 0
           AS all_tested_indexes_match,
       COALESCE(sum(r.mismatching_attempts), 0) AS observed_index_mismatches,
       COALESCE(sum(r.error_attempts), 0) AS attempt_errors
FROM pg_temp.hic_run AS run
JOIN pg_temp.hic_run_integrity AS integrity USING (run_id)
LEFT JOIN pg_temp.hic_relation_result AS r ON true
GROUP BY run.run_id, run.database_name, run.schema_name, run.snapshot_text,
         run.started, run.finished, run.inventory_complete, integrity.evidence_valid;

CREATE /* wiki_hic12_parent_results */ TEMP VIEW hic_parent_result AS
SELECT obj.relation_oid, obj.qualified_name,
       count(p.descendant_oid) AS descendants,
       count(p.descendant_oid) FILTER (WHERE p.outside_schema) AS outside_schema_descendants,
       COALESCE(bool_and(p.included), false) AS all_descendants_in_scope,
       COALESCE(bool_and(p.supported), false) AS all_descendants_supported,
       count(*) FILTER (WHERE p.relkind <> 'p') AS physical_children,
       COALESCE(bool_and(p.included AND p.supported), false)
           AND count(*) FILTER (WHERE p.relkind <> 'p') > 0
           AND COALESCE(bool_and(r.status = 'PASS') FILTER (WHERE p.relkind <> 'p'), false)
           AS subtree_success
FROM pg_temp.hic_object AS obj
LEFT JOIN pg_temp.hic_partition AS p ON p.parent_oid = obj.relation_oid
LEFT JOIN pg_temp.hic_relation_result AS r ON r.relation_oid = p.descendant_oid
WHERE obj.relkind = 'p'
GROUP BY obj.relation_oid, obj.qualified_name;

CREATE /* wiki_hic12_attempt_report */ TEMP VIEW hic_attempt_report AS
SELECT a.*, r.heap_count,
       CASE WHEN a.phase = 'index' AND a.accepted
            THEN a.row_count::numeric - r.heap_count::numeric END AS delta
FROM pg_temp.hic_attempt_result AS a
JOIN pg_temp.hic_relation_result AS r USING (relation_oid);

CREATE /* wiki_hic12_pending */ TEMP VIEW hic_pending AS
SELECT a.attempt_id, a.relation_oid, a.phase, a.statement_text,
       c.attempt_id IS NOT NULL AS count_present,
       p.attempt_id IS NOT NULL AS plan_present,
       CASE WHEN c.attempt_id IS NOT NULL AND p.attempt_id IS NULL
            THEN 'Import the JSON from this exact execution, or record a capture error.'
            WHEN c.attempt_id IS NULL AND p.attempt_id IS NOT NULL
            THEN 'The explained INSERT did not leave a count; recover and record an error.'
            ELSE 'Execute the saved statement or recover and record the execution error.'
       END AS required_action
FROM pg_temp.hic_attempt AS a
LEFT JOIN pg_temp.hic_count AS c USING (attempt_id)
LEFT JOIN pg_temp.hic_plan AS p USING (attempt_id)
LEFT JOIN pg_temp.hic_error AS e USING (attempt_id)
WHERE e.attempt_id IS NULL AND (c.attempt_id IS NULL OR p.attempt_id IS NULL);

CREATE /* wiki_hic12_control_profile */ FUNCTION pg_temp.hic_control_profile()
RETURNS text
LANGUAGE sql IMMUTABLE SECURITY INVOKER
SET search_path = pg_catalog, pg_temp
AS $function$
    SELECT /* wiki_hic12_control_text */ $profile$
SET /* wiki_hic12_control_seq */ LOCAL enable_seqscan = on;
SET /* wiki_hic12_control_index */ LOCAL enable_indexscan = on;
SET /* wiki_hic12_control_only */ LOCAL enable_indexonlyscan = on;
SET /* wiki_hic12_control_bitmap */ LOCAL enable_bitmapscan = on;
SET /* wiki_hic12_control_sort */ LOCAL enable_sort = on;
SET /* wiki_hic12_control_quote */ LOCAL quote_all_identifiers = off;
$profile$::text;
$function$;

CREATE /* wiki_hic12_next_attempt */ TEMP VIEW hic_next AS
WITH choices AS (
    SELECT r.relation_oid, r.qualified_name, 'heap'::text AS phase,
           NULL::oid AS intended_index_oid, NULL::text AS order_sql,
           previous.attempts + 1 AS variant
    FROM pg_temp.hic_relation_result AS r
    CROSS JOIN LATERAL (
        SELECT count(*) AS attempts FROM pg_temp.hic_attempt AS a
        WHERE a.relation_oid = r.relation_oid AND a.phase = 'heap'
    ) AS previous
    WHERE r.in_heap_scope AND r.permission_error IS NULL AND r.evidence_valid
      AND r.eligible_indexes > 0 AND r.heap_count IS NULL AND previous.attempts < 2
    UNION ALL
    SELECT r.relation_oid, r.qualified_name, 'index', i.index_oid, i.order_sql,
           i.intended_attempts + 1
    FROM pg_temp.hic_relation_result AS r
    JOIN pg_temp.hic_candidate_result AS i USING (relation_oid)
    CROSS JOIN pg_temp.hic_run AS run
    WHERE r.in_heap_scope AND r.permission_error IS NULL AND r.evidence_valid
      AND r.heap_count IS NOT NULL AND r.distinct_heap_counts = 1
      AND (r.status <> 'PASS' OR run.exhaustive)
      AND i.eligible AND NOT i.actually_checked AND i.order_sql IS NOT NULL
      AND i.intended_attempts < 2
),
chosen AS (
    SELECT choices.*, COALESCE((SELECT max(attempt_id) FROM pg_temp.hic_attempt), 0) + 1 AS attempt_id
    FROM choices
    CROSS JOIN pg_temp.hic_run_integrity AS guard
    WHERE guard.evidence_valid AND guard.active_snapshot
      AND NOT EXISTS (SELECT 1 FROM pg_temp.hic_pending)
    ORDER BY relation_oid, phase, intended_index_oid NULLS FIRST
    LIMIT 1
),
statement AS (
    SELECT chosen.*,
           format(
               E'EXPLAIN /* wiki_hic12_executed_count */ (ANALYZE, VERBOSE, BUFFERS, SETTINGS, TIMING OFF, SUMMARY ON, FORMAT JSON)\nINSERT /* wiki_hic12_save_count */ INTO pg_temp.hic_count(attempt_id, row_count)\nSELECT /* wiki_hic12_exact_count */ %s::bigint, count(*) FROM %s;',
               attempt_id,
               CASE WHEN phase = 'heap' THEN format('ONLY %s', qualified_name)
                    ELSE format('(SELECT /* wiki_hic12_full_order */ 1 FROM ONLY %s ORDER BY %s) AS ordered_rows',
                                qualified_name, order_sql) END
           ) AS count_sql
    FROM chosen
)
SELECT statement.*,
       format(
           E'INSERT /* wiki_hic12_record_attempt */ INTO pg_temp.hic_attempt(attempt_id, relation_oid, phase, intended_index_oid, variant, statement_text) VALUES (%s, %s, %L, %s, %s, %L);\nSAVEPOINT /* wiki_hic12_attempt_boundary */ hic_attempt;\nSET /* wiki_hic12_count_path */ LOCAL search_path = pg_catalog, pg_temp;\nSET /* wiki_hic12_count_rls */ LOCAL row_security = off;\nSET /* wiki_hic12_count_parallel */ LOCAL max_parallel_workers_per_gather = 0;\nSET /* wiki_hic12_count_timeout */ LOCAL statement_timeout = ''15min'';\nSET /* wiki_hic12_count_lock_timeout */ LOCAL lock_timeout = ''2s'';\nSET /* wiki_hic12_count_seq */ LOCAL enable_seqscan = %s;\nSET /* wiki_hic12_count_index */ LOCAL enable_indexscan = %s;\nSET /* wiki_hic12_count_only */ LOCAL enable_indexonlyscan = off;\nSET /* wiki_hic12_count_bitmap */ LOCAL enable_bitmapscan = off;\nSET /* wiki_hic12_count_sort */ LOCAL enable_sort = %s;\n%s',
           attempt_id, relation_oid, phase, COALESCE(intended_index_oid::text, 'NULL'),
           variant, count_sql,
           CASE phase WHEN 'heap' THEN 'on' ELSE 'off' END,
           CASE phase WHEN 'heap' THEN 'off' ELSE 'on' END,
           CASE WHEN phase = 'index' AND variant = 2 THEN 'off' ELSE 'on' END,
           count_sql
       ) AS sql_text,
       pg_temp.hic_control_profile() AS post_count_sql,
       format(
           E'INSERT /* wiki_hic12_import_plan */ INTO pg_temp.hic_plan(attempt_id, document) VALUES (%s, <SQL-quoted JSON from this execution>::jsonb);\nRELEASE /* wiki_hic12_accept_attempt */ SAVEPOINT hic_attempt;',
           attempt_id
       ) AS import_template,
       format(
           E'ROLLBACK /* wiki_hic12_recover_attempt */ TO SAVEPOINT hic_attempt;\nINSERT /* wiki_hic12_record_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, detail, hint, snapshot_survived) VALUES (%s, <SQL-quoted stage>, <SQL-quoted SQLSTATE or NULL>, <SQL-quoted message>, <SQL-quoted detail or NULL>, <SQL-quoted hint or NULL>, true);\nRELEASE /* wiki_hic12_release_error */ SAVEPOINT hic_attempt;',
           attempt_id
       ) AS error_template
FROM statement;

CREATE /* wiki_hic12_progress */ TEMP VIEW hic_progress AS
SELECT CASE
    WHEN EXISTS (SELECT 1 FROM pg_temp.hic_run_error) THEN 'ERROR'
    WHEN NOT EXISTS (SELECT 1 FROM pg_temp.hic_run) THEN 'CONFIGURE_AND_START'
    WHEN EXISTS (SELECT 1 FROM pg_temp.hic_run_integrity WHERE NOT evidence_valid) THEN 'INVALID_RUN'
    WHEN NOT EXISTS (SELECT 1 FROM pg_temp.hic_run_integrity WHERE active_snapshot) THEN 'NOT_ACTIVE'
    WHEN EXISTS (SELECT 1 FROM pg_temp.hic_pending) THEN 'AWAITING_EVIDENCE_OR_ERROR'
    WHEN EXISTS (SELECT 1 FROM pg_temp.hic_next) THEN 'ATTEMPT_AVAILABLE'
    ELSE 'ATTEMPTS_COMPLETE_REVIEW_REPORT'
END AS state;

INSERT /* wiki_hic12_register_owned */ INTO pg_temp.hic_owned
WITH named AS (
    SELECT c.oid, c.reltoastrelid FROM pg_catalog.pg_class AS c
    WHERE c.relnamespace = pg_my_temp_schema() AND c.relname IN (
        'hic_config', 'hic_run', 'hic_object', 'hic_index', 'hic_attempt',
        'hic_count', 'hic_plan', 'hic_error', 'hic_run_error', 'hic_owned', 'hic_partition',
        'hic_inventory', 'hic_untrusted_indexes', 'hic_index_keys', 'hic_candidates',
        'hic_start_script', 'hic_run_integrity', 'hic_attempt_result', 'hic_relation_result',
        'hic_candidate_result', 'hic_schema_result', 'hic_parent_result',
        'hic_attempt_report', 'hic_pending', 'hic_next', 'hic_progress'
    )
),
relations AS (
    SELECT oid FROM named UNION SELECT reltoastrelid FROM named WHERE reltoastrelid <> 0
)
SELECT oid FROM relations
UNION
SELECT i.indexrelid FROM pg_catalog.pg_index AS i JOIN relations AS r ON r.oid = i.indrelid;
```

### Operator workflow and implementation policy

The installation block implements one **schema-wide**, serial, primary-server run per dedicated session. It defaults to namespace-only scope and stopping after the first match. It allows two heap attempts and two attempts directed at each eligible candidate: an index attempt first leaves sorting enabled, then discourages sorting. An accepted scan of a different eligible index credits that actual OID; it does not consume that index's intended-attempt budget or prove the requested index was used. Exhaustive mode continues after a match but cannot guarantee exhaustive coverage.

This is a generated-statement protocol, not a stored procedure. Execute the steps separately, or let the follow-up's [runner](#follow-up-a-bash-and-sql-runner) execute them:

1. Arrange the DDL coordination and privileges described above. Connect as the checking role, without an open transaction, and run the complete installation block. The default installation timeouts are 60 seconds and 2 seconds; generated checking statements use 15 minutes and 2 seconds. Review these budgets for the workload before running.
2. Configure the schema. `ddl_stable = true` is an **operator attestation**, not a lock or an automatically checked fact. Substitute the intended schema for `public`. Change the two optional booleans only when that scope is intended.

   ```sql
   INSERT /* wiki_hic12_operator_config */ INTO pg_temp.hic_config(
       target_schema, ddl_stable, include_external_partitions, exhaustive
   ) VALUES ('public', true, false, false);
   SELECT /* wiki_hic12_operator_initialize */ sql_text FROM pg_temp.hic_start_script;
   ```

3. Copy and execute the returned initialization script unchanged. It begins the repeatable-read, read-only transaction, fixes the snapshot/context and captures the inventory. If initialization fails, save the diagnostic in the client, roll back the failed transaction, and record an `initialization` or `inventory` row in the pre-created `hic_run_error` table. No schema-result row means **no initialized result**, not success. End that failed run; do not carry observations into a new checking snapshot.
4. Read progress and the next generated attempt. Save the returned attempt ID, count statement and both recovery templates before executing anything:

   ```sql
   SELECT /* wiki_hic12_operator_progress */ * FROM pg_temp.hic_progress;
   SELECT /* wiki_hic12_operator_next */ attempt_id, relation_oid, phase,
          intended_index_oid, sql_text, post_count_sql, import_template, error_template
   FROM pg_temp.hic_next;
   ```

5. Execute that row's `sql_text` unchanged and save its complete JSON result. **EXPLAIN must remain the last command in this submitted batch.** Execute `post_count_sql` as a separate submission before asking for another attempt. It restores planner controls for the metadata queries; it cannot change the already captured count or plan.
6. On success, fill `import_template` with a properly SQL-quoted JSON value from **that exact execution**. Execute its `INSERT` line, inspect `hic_attempt_report`, and only then execute its `RELEASE` line. Inspect before releasing: once `RELEASE SAVEPOINT` has run, the savepoint no longer exists, so `ROLLBACK TO SAVEPOINT hic_attempt` fails with `savepoint "hic_attempt" does not exist` and an inspection failure would leave the whole checking transaction aborted. A syntactically valid but unacceptable plan remains recorded with its rejection reasons; it earns no count credit. [xact.c#ReleaseSavepoint](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4140-L4221) [xact.c#RollbackToSavepoint](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4249-L4308)
7. On execution, JSON-import or inspection failure, preserve SQLSTATE/message/detail/hint in the client. Execute `error_template`, filling the stage (`execution`, `capture` or `inspection`), SQLSTATE, message, detail and hint; the template already carries the attempt ID. Its rollback-to-savepoint removes any count or partially imported evidence but preserves the attempt record created before the savepoint. Execute `post_count_sql` separately if the failure path rolled back its settings. Retry only through the next generated attempt. If the connection or the whole transaction is lost, stop: a savepoint in a replacement transaction is not recovery of this run.
8. Repeat from progress. `AWAITING_EVIDENCE_OR_ERROR` means finish the pending import/recovery first. An empty `hic_next` alone does not establish success: `INVALID_RUN`, `NOT_ACTIVE` and `ERROR` are stop states. `ATTEMPTS_COMPLETE_REVIEW_REPORT` means examine the reports, including all untested reasons and errors.
9. Export the result tables and views through the client while the session still exists. Finalize only after the report has been reviewed:

   ```sql
   SELECT /* wiki_hic12_operator_schema */ * FROM pg_temp.hic_schema_result;
   SELECT /* wiki_hic12_operator_relations */ * FROM pg_temp.hic_relation_result;
   SELECT /* wiki_hic12_operator_candidates */ * FROM pg_temp.hic_candidate_result;
   SELECT /* wiki_hic12_operator_attempts */ * FROM pg_temp.hic_attempt_report;
   SELECT /* wiki_hic12_operator_parents */ * FROM pg_temp.hic_parent_result;
   SELECT /* wiki_hic12_operator_global_errors */ * FROM pg_temp.hic_run_error;
   UPDATE /* wiki_hic12_operator_finish */ pg_temp.hic_run SET finished = clock_timestamp();
   COMMIT /* wiki_hic12_operator_commit */;
   ```

The generated SQL and result API above are checker policy. The engine mechanisms that make the protocol work are the previously cited [same-statement capture](#capturing-an-exact-count-and-its-executed-plan) and [snapshot/savepoint rules](#snapshot-permissions-and-ddl-coordination). The temporary report survives COMMIT in this session, but new attempts are disabled. Export the final run row, including `finished`, and close the dedicated connection. A later run starts in a new session with a new inventory, heap baseline and snapshot.

If installation fails before the error table exists, the client must retain the error; there is no SQL report to read from that failed setup. If the DDL agreement is broken after initialization, set `hic_run.invalid_reason` to the observed reason, export the invalidated evidence and end the run. The checker does not automatically detect every DDL race or enforce the agreement.

The implementation keeps generation policy separate from eligibility. It generates orderings only for reviewed stock operators/opclasses and supported types; user-defined functions/operators, domains and other unreviewed types can leave an otherwise eligible B-tree untested. It avoids deparsing those expressions, rather than merely declining to execute their generated query. A key that deparses to a bare numeric literal is emitted as a same-type `CAST`, so `ORDER BY` can never read it as an output-column position; see step 2 of the [ordering generation](#index-eligibility-and-ordering-generation). This is a conservative coverage policy, **not a security sandbox** for hostile database code or server hooks. Plain table-level SELECT is required by preflight even if a narrower privilege combination could execute a particular query. Relations with no eligible candidate need no heap attempt and retain a NULL baseline, never a fabricated zero.

`hic_check_plan()` declares `ROWS 1` because its aggregate-based validator returns one verdict row. This changes the planner's row estimate, not the function's execution semantics; the default estimate for a set-returning function would be 1000 and is inappropriate for these repeated report joins. [functioncmds.c#default-rows](../../../../raw/postgres-12/src/backend/commands/functioncmds.c#L1134-L1139) [create_function.sgml#ROWS](../../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L468-L477)

Checker-owned temporary relations are excluded by recorded OID, including their indexes and TOAST objects, not by an `hic_` naming prefix. A real application temporary table with that prefix remains in scope. Other backends' temporary tables are rejected. The implementation does not implement the optional identity-bag scan generator, per-relation transaction mode, exported-snapshot coordination, arbitrary custom expression execution, or physical page diagnostics; those remain separate extensions.

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
| One mismatching accepted index plus an untested candidate; all candidates mismatch; one match plus another failure | Respect INCONCLUSIVE/MISMATCH/PASS distinctions and retain per-attempt diagnostics |

Tests of deliberately corrupted on-disk indexes are a separate boundary. Ordinary SQL DML maintains indexes, and pageinspect only exposes page contents; this plan does not establish a supported SQL-only method to create arbitrary missing/duplicate physical index entries. Use synthetic count/plan/bag records to test the verdict evaluator, clearly labelled as unit tests. A real corruption campaign would require a separately approved fault-injection capability or supplied damaged disposable database; synthetic evaluator tests must not be reported as physical corruption detection coverage.

Likewise, this pin supplies heap as its only built-in table AM. Test unsupported table-AM classification with synthetic inventory records; a real custom-AM integration test would need a separately authorized capability, not an unannounced extension dependency. [pg_am.dat#built-in-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L35)

The acceptance gate is that all positive fixtures produce a count only from validated executed paths, and every exercised negative/coverage fixture refuses an unjustified PASS or MISMATCH. Actual plans remain in the attempt exports even when they differ from the intended path.

### Measured coverage and remaining test gaps

The published [Measurement Script](#measurement-script) passed a fresh out-of-tree build and full suite, then a `verify` rerun with the build retained; both runs passed all 124 assertions. Every case drives the checker through the follow-up's runner, either as a program or through the runner functions the harness sources. These are results of this checker on disposable healthy fixtures, not evidence that every index or damaged database can be diagnosed.

| Executed case | Observed result and assertion |
|---|---|
| Namespace-only positive schema | **13 physical relations PASS**, zero observed count mismatches; all **13 eligible indexes** are checked and matching, none untested. Includes empty heap, mixed NULL/duplicate keys, `lower(k)` plus a comma-containing `coalesce` expression, `text_pattern_ops`, explicit `"C"` collation, mixed key directions/NULL placement and INCLUDE, unique/nonunique indexes, inheritance parent/child, physical partition, populated and empty-populated materialized views, unlogged heap and quoted relation/column/index names. The aborted inserted row is absent. |
| Explicit descendant closure | **14 physical relations PASS**, with 14 of 14 eligible indexes checked and matching; the outside-schema physical leaf is added, its unrelated neighbor is not. A foreign leaf keeps `all_descendants_supported` and `subtree_success` false. Namespace success is not an all-children heap claim. |
| Own temporary schema | The application temporary table with an `hic_` prefix passes with **2 rows**. The checker's OID-registered objects are excluded. |
| Concurrent DML between counts | Both accepted counts remain **100** in the checking snapshot. The writer's later snapshot sees **95**: updates preserve cardinality, six committed deletes remove rows, one committed insert adds a row, and the aborted insert adds none. The fixture uses fillfactor 50 to leave room for HOT updates and asserts both HOT and non-HOT transaction counters. |
| Native `indcheckxmin` gate | With a separate old-XID holder active, a HOT update followed by a nonconcurrent index build sets the flag; the checker sees **120 heap rows**, cannot obtain an accepted index scan and reports **INCONCLUSIVE**. After ending the holder, a **new checking transaction** reports **PASS**, still with the flag true. Both checking snapshots are taken after the build; the distinction is their backend horizon, not inventory invisibility of the new index. |
| Equivalent-order candidate competition | Exhaustive mode reports PASS but incomplete coverage. Both attempts intended for `wide_idx` actually use `compact_idx`; the wider INCLUDE index remains untested. The schema totals read 2 eligible, 1 checked and matching and 1 untested, with `all_tested_indexes_match` true and `exhaustive_coverage_complete` false. The fixture analyzes the table and deliberately makes the INCLUDE payload wide. |
| Negative inventory/path cases | Eight physical relations. Four report **NO_ELIGIBLE_INDEX**: absent index, partial-only, hash-only, and a genuinely invalid concurrent unique build. Four report **INCONCLUSIVE**: the constant keys `(1)`, `(2)` and `(1.5)` lose their useful index path, with two rejected attempts per index and no attempt error, or generation is deferred for a user function/domain. Each constant key's generated ordering starts with `CAST(`. A parser probe in the fixtures shows why: the uncast forms fail, `((2))` with SQLSTATE `42P10` ("ORDER BY position 2 is not in select list") and `((1.5))` with `42601` ("non-integer constant in ORDER BY"), while the `CAST` forms run. No case is converted to a zero-count pass. |
| Role/permission cases | An ordinary effective role selected with SET ROLE, without superuser or BYPASSRLS privileges, checks the unrestricted positive schema. A forced-RLS owner gets ERROR with SQLSTATE `42501`, not a filtered count; an authorized superuser counts both RLS-table rows. Missing table SELECT or schema USAGE remains an inventoried ERROR. |
| Lost-plan retry | Discarding the first executed plan and rolling back its savepoint leaves one retained capture error. The same-snapshot retry establishes PASS; there are **three attempt records**, two accepted and one failed. |
| Validator and evaluator units | **33 plan-shape assertions** pass. Additional synthetic records test state-flag exclusions, complete-coverage MISMATCH with a recovered failure, one mismatch plus an untested candidate yielding INCONCLUSIVE, missing evidence, incorrect settings/attempt marker, mixed snapshots, PASS with an optional error, unsupported-AM aggregation and equal-size but different identity bags. These records do not represent physical index corruption. |
| Initialization and history | Missing DDL attestation and missing schema cannot establish a run; the initialization-error channel is exercised. Empty physical scope is not success. Reports retain their verdict after COMMIT, including when the caller's `quote_all_identifiers = on` setting is restored. |
| Runner as a program | Five invocations. `hic_positive` exits **0**, with 26 saved plans and all seven CSV reports; `hic_negative` exits **1** with `schema_success=false`; a missing schema exits **2** and exports the `initialization` row with SQLSTATE `23514`; no `HIC_DDL_STABLE=yes` and an invalid yes/no value each exit **64**, the first without creating an output directory. |
| Error reports parsed from psql | The forced-RLS attempts are recorded with SQLSTATE `42501`, the server message and its HINT, “To disable the policy for the table's owner, use ALTER TABLE NO FORCE ROW LEVEL SECURITY.” |
| Lock timeout | A second session takes `ACCESS EXCLUSIVE` on `hic_lock.t` after inventory. Both heap attempts fail after the 2-second `lock_timeout` with SQLSTATE `55P03`; the relation is `ERROR` with no heap count, not a zero count. |
| Truncated plan | The harness cuts a saved plan to its first three lines before import. jsonb rejects it with SQLSTATE `22P02` and a DETAIL; the runner records a `capture` error, and the retry passes with `uncovered_error_attempts = 0`. |
| Lost session | After one accepted attempt, `pg_terminate_backend()` ends the checking backend. The next request gets no sync line, the runner reports the session lost, and psql exits with status **2**; its error log shows `FATAL:  57P01:`. No report is produced. |
| psql interpolation guard | Through psql, `(ARRAY[1, 2, 3])[1:"ERROR"]` fails with SQLSTATE `42601`, while `[1 : "ERROR"]` returns `{1,2}`. The guard index deparses as `(a[1:"ERROR"])`; the runner refuses both of its index attempts, records two execution errors with no SQLSTATE and sends no count, so the relation is `ERROR` with heap count 2. |

The concurrency arithmetic is `100 - 6 + 1 = 95` for the writer's current view, while the checker's retained snapshot produces `H = I = 100`. The held-snapshot behavior comes from repeatable read and heap visibility; the deliberate HOT fixture and retained-old-horizon index refusal have separate causes, detailed below. [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [heapam.c#HOT-search](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1559-L1622) [plancat.c#indcheckxmin-gate](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234)

**Not exercised by this suite:** physical file corruption; a real custom table AM; arbitrary expression/opclass/type combinations; non-C locale coverage or ICU-specific behavior; multiple inheritance and multilevel/attach/detach races; another backend's temporary relation; denied TEMP/function EXECUTE; a separate non-superuser BYPASSRLS role; open writers at initial snapshot acquisition; concurrent pruning/redirect coverage; forced statement timeouts or query cancellation; a connection lost while a statement is running, since the measured loss happens between requests; standby operation; large-schema scaling and production I/O costs. The runner was run only with GNU Bash 5.2.21 and the psql built from the pin, against a server whose `lc_messages` is `C`. Backward direction is covered by a plan unit, not a measured backward executor leg. The optional visible-`ctid` bag scan/capture pipeline is not implemented or measured; only the duplicate-sensitive comparison algebra is unit-tested.

### Optional relaxations

Keep these choices separate from the literal SQL baseline:

- **A Bash and SQL runner, implemented by the follow-up.** The [runner](#follow-up-a-bash-and-sql-runner) drives one psql session from Bash and keeps every check in the installed SQL. It is the relaxation the asker authorized; the PL/pgSQL wrapper below remains unimplemented.
- **A PL/pgSQL wrapper, only if authorized.** A [PL/pgSQL](../../../glossary.md#plpgsql) function/procedure can loop over catalog rows, use dynamic `EXECUTE`, receive the EXPLAIN JSON result, manage exceptions and save error diagnostics. It must retain the explained temporary INSERT technique for an exact count and preserve the same validator and status rules. Use invoker privileges; do not mark a function wrapper STABLE or IMMUTABLE, because SPI's nonvolatile-function read-only check rejects these commands. It improves orchestration; it cannot force a named index or eliminate coverage gaps. [plpgsql.sgml#dynamic-commands](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1157-L1185) [plpgsql.sgml#data-versus-identifiers](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1225-L1249) [spi.c#nonvolatile-command-refusal](../../../../raw/postgres-12/src/backend/executor/spi.c#L2258-L2263)
- **psql conveniences.** Built-in `psql` has `\gexec` for executing generated SQL and `\if` for client-side branches. These are client meta-commands, not literal SQL. `\gexec` does not itself import each generated EXPLAIN result into an audit table, and generated rows cannot contain psql meta-commands or variable references. Treat this as an optional client guide, not a promised fully automated capture layer. The follow-up's runner does not use these meta-commands: the asker required psql to receive only SQL statements and sync lines. [psql-ref.sgml#gexec](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L2059-L2094) [psql-ref.sgml#if](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L2233-L2260)
- **Exported-snapshot coordination.** Separate sessions can import an exported snapshot before their first query if the exporting transaction remains available. This adds orchestration and still retains a long snapshot; it is unnecessary for the first serial design. [set_transaction.sgml#snapshot-import](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L155-L177)
- **Stronger diagnostics.** Identity bags remain SQL-only but add storage and time. pageinspect is a separate extension/privilege choice for selected mismatch investigation. Other AM support, other PostgreSQL versions and physical fault injection each need their own evidence and tests.

### Review of the implementation

The implementation and its tests preserve the original review decisions:

| Review issue | Resolution |
|---|---|
| An EXPLAIN Aggregate node's row count is not the aggregate value | Save the exact count with the same explained INSERT |
| A separate EXPLAIN/SELECT pair does not bind evidence to the counted execution | Capture the executed plan of the statement that stores the count |
| A planner switch or compatible ordering does not select an index by name | Validate actual scan type and identity; keep untested candidates |
| `indcheckxmin = true` is not unconditional ineligibility | Defer to the planner horizon gate and actual accepted use |
| Temporary CTAS inside READ ONLY would violate the transaction mode | Create scratch tables before the checking transaction and use temporary INSERTs |
| PostgreSQL 12 cannot `LOCK TABLE` a materialized view | Use an ordinary read under explicit DDL coordination |
| One observed mismatch is weaker than the requested relation MISMATCH | Require complete actual candidate coverage before that verdict |
| Raw page items or raw index TIDs are not the visible population | Use the executor; keep physical inspection separate |
| Literal SQL does not supply an automatic EXPLAIN-result ingestion/error-handling loop | Make operator interaction explicit; the follow-up's Bash runner automates it, and PL/pgSQL remains an unimplemented relaxation |
| An inspection after `RELEASE SAVEPOINT` cannot be rolled back to the attempt savepoint | Operator step 6 now inspects before releasing; the runner inspects inside the savepoint |
| psql rewrites `:name`, `:'name'` and `:"name"` outside quotes | The runner dollar-quotes every value it sends and refuses a count statement with a colon followed by a quote or brace |
| A key that deparses to a bare numeric literal is read by `ORDER BY` as an output position, or rejected | Emit it as a same-type `CAST`; a parser probe and a literal-key fixture keep that case measured |

The review compared the complete plan, its query shapes, statuses, diagrams and test expectations with pinned implementation and official documentation, the relevant glossary definitions, matching-version comparison pages and the new navigation summaries. One existing glossary contradiction was identified and corrected: its `indcheckxmin` entry said the planner leaves an index out “while the flag holds,” whereas the source also requires the horizon test to fail. One pinned documentation wording discrepancy is retained under Open Questions: the index-scan switch is described as enabling/disabling plans, while implementation uses a cost penalty. No unresolved contradiction is intended outside Open Questions.

Implementation testing also caught two concrete capture/identity issues: a trailing command could hide the batch's EXPLAIN result, and the JSON index name uses SQL identifier quoting. The final generator separates the post-count controls; the identity helper applies the same quoting independently of the caller's later settings. The exact published installation and Bash blocks were compared byte for byte with their tested artifacts, and the extracted script was run from an empty sandbox and with the build retained. Human verification remains separate from these agent tests. Universal coverage, automatic DDL protection, unattended literal-SQL orchestration and corruption-detection rates are not claimed.

A later 2026-10-23 review found that the generator wrapped every deparsed key only in parentheses, so a key on `(2)` or `(1.5)` produced a failing statement and a relation ERROR, and that the `(1)` fixture had passed only because position 1 is the inner query's constant. The generator now casts such keys; the same review aligned the documented reason codes and record fields with the installed SQL, added schema-level index totals, replaced `pgrep` in the harness with POSIX `ps`, and re-ran the full suite from an empty sandbox and with the build retained.

Final causal chain: coordinated scope and one retained snapshot → captured inventory → generated explained count INSERT → exact count and faithfully imported plan → context/shape validation → actual-index coverage → relation/schema verdicts → export and COMMIT. A new run starts the chain again; it does not reuse a completed snapshot's baseline.

### Follow-up: a Bash and SQL runner

**The follow-up replaces the operator with a Bash program; the checker's SQL, its plan validator and its verdict rules do not change.** The [runner](#runner-script) opens one [psql](../../../glossary.md#psql) session and installs the same SQL block. It then executes each generated count statement as the database returns it, saves the JSON plan that statement printed, and imports that plan. It records every failure through the attempt's savepoint, exports the reports, commits, and exits with a status that says whether the schema passed. The literal-SQL protocol above remains valid without Bash; the runner automates that protocol and adds no acceptance rule.

The asker scoped the follow-up: Bash makes every decision, and psql receives only SQL statements plus one `\echo` line per request. That line is needed because no SQL statement can report back once a transaction has aborted. The server then rejects every statement except `COMMIT`, `PREPARE TRANSACTION`, `ROLLBACK` and `ROLLBACK TO SAVEPOINT`, while a psql meta-command never reaches the server. [postgres.c#aborted-transaction-filter](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L1091-L1105) [postgres.c#IsTransactionExitStmt](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L2600-L2615) [command.c#exec_command_echo](../../../../raw/postgres-12/src/bin/psql/command.c#L1116-L1152)

The runner leaves these properties as the protocol defines them:

- One serial, schema-wide `REPEATABLE READ READ ONLY` transaction holds the snapshot, because the runner sends the generated start script, which opens it.
- Each count and its plan come from one explained `INSERT`, because the runner sends each generated `sql_text` without changing its text.
- The planner still chooses the index. The generator's two-attempt budgets and the coverage limits above are unchanged, so the runner adds speed, not coverage.
- `HIC_DDL_STABLE=yes` is the operator's attestation of the DDL-stable window, just as `ddl_stable = true` was; the runner checks nothing about DDL.

### How the runner talks to psql

**Bash and psql exchange one request and one reply at a time through two named pipes.** psql reads SQL from one pipe as its standard input and writes results to the other. Because neither is a terminal, psql runs non-interactively: it prints no prompt, and it does not ask the server for a locale-derived client encoding, so the database encoding applies unless `PGCLIENTENCODING` is set. [startup.c#notty](../../../../raw/postgres-12/src/bin/psql/startup.c#L182) [mainloop.c#interactive](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L66) [startup.c#client-encoding](../../../../raw/postgres-12/src/bin/psql/startup.c#L264-L265) [psql-ref.sgml#client-encoding](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L710-L719)

The exchange works because psql flushes its output at the points where the runner waits. It flushes standard output before reading each input line, after printing each query result, and after each backslash command. Therefore, when psql has printed the sync line and is waiting for more input, the whole reply is already in the pipe. [mainloop.c#flush-before-read](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L147) [common.c#flush-after-result](../../../../raw/postgres-12/src/bin/psql/common.c#L1299) [command.c#flush-after-backslash](../../../../raw/postgres-12/src/bin/psql/command.c#L259-L260)

| Value | Meaning | Source | Live or stored | Later use |
|---|---|---|---|---|
| Request | SQL text the runner sends | Built by Bash, or fetched from `hic_start_script`, `hic_next` or `hic_control_profile()` | Stored in the session's `transcript.sql` | psql runs it statement by statement |
| Sync line | `HIC12_SYNC_<pid>_<session>_<n>` followed by psql's `ERROR` and `SQLSTATE` | psql's `\echo`, after the request's last statement | Live | Ends the reply and reports the last statement's outcome |
| Reply | Standard-output lines before the sync line: rows, EXPLAIN JSON or COPY CSV | psql | Copied to the session's `response` file | Values, the plan file and the report CSVs |
| Request errors | psql's error reports printed during this request | psql standard error, appended to `stderr.log` | This request's slice is kept in `stderr.last` | Failure test, then SQLSTATE, message, detail and hint |
| Failed flag | Whether any statement of the request failed | Sync line `ERROR`, or an error report in the slice | Live | Every branch below |
| Plan file | The JSON one `EXPLAIN` printed | Reply to the attempt request | Stored as `plans/attempt-<id>.json` | Imported into `hic_plan` and kept for audit |
| Attempt bound | Most attempts the generator can offer | `2 × heap-scope relations + 2 × eligible indexes`, read after inventory | Held in Bash | Stops a loop that would exceed it |

One request goes through these steps:

1. Bash refuses a request whose last non-blank line does not end with a semicolon. psql runs a backslash line at once even while a statement is incomplete, so a sync line after an unterminated statement would arrive before that statement ran. [mainloop.c#backslash-line](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L457-L491)
2. Bash records the size of psql's error log, writes the request, then writes `\echo HIC12_SYNC_... :ERROR :SQLSTATE`.
3. psql sends each semicolon-terminated statement as its own query and prints its rows; with `-q` it prints no command tags. [mainloop.c#send-on-semicolon](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L410-L440) [common.c#PrintQueryStatus](../../../../raw/postgres-12/src/bin/psql/common.c#L1211-L1225)
4. psql prints a server error to standard error as [libpq](../../../glossary.md#libpq) formats it. In this stdin mode the line carries no file or line prefix. With `VERBOSITY` set to `verbose`, the first line reads `SEVERITY:  SQLSTATE: message`, and `DETAIL:` and `HINT:` lines follow. [common.c#AcceptResult](../../../../raw/postgres-12/src/bin/psql/common.c#L530-L538) [startup.c#stdin-terse-logging](../../../../raw/postgres-12/src/bin/psql/startup.c#L438-L446) [startup.c#log_locus_callback](../../../../raw/postgres-12/src/bin/psql/startup.c#L101-L114) [logging.c#locus-prefix](../../../../raw/postgres-12/src/common/logging.c#L164-L179) [fe-protocol3.c#verbose-message](../../../../raw/postgres-12/src/interfaces/libpq/fe-protocol3.c#L1017-L1046) [fe-protocol3.c#detail-and-hint](../../../../raw/postgres-12/src/interfaces/libpq/fe-protocol3.c#L1090-L1095) [startup.c#verbosity_hook](../../../../raw/postgres-12/src/bin/psql/startup.c#L1113-L1123)
5. psql reaches the `\echo` line, replaces `:ERROR` and `:SQLSTATE` with the values it set for the last query, prints the line and flushes. [psqlscanslash.l#argument-variables](../../../../raw/postgres-12/src/bin/psql/psqlscanslash.l#L374-L389) [common.c#SetResultVariables](../../../../raw/postgres-12/src/bin/psql/common.c#L544-L584) [psql-ref.sgml#ERROR](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3647-L3655) [psql-ref.sgml#SQLSTATE](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3962-L3971)
6. Bash copies standard-output lines into the reply until the sync line, then cuts this request's slice of the error log.
7. Bash marks the request failed when `ERROR` is `true` or the slice holds an error report: a report whose SQLSTATE class is not `00`, `01` or `02`, the success, warning and no-data classes. On failure it parses the first report. [errcodes.txt#classes-00-02](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L77-L99)

Step 7 needs both signals. Inside a transaction block, a failed statement makes every later one fail with SQLSTATE `25P02`, so the last statement's `ERROR` covers the whole request. Outside a block, each statement commits on its own, and a failure in the middle leaves the last `ERROR` false; only the error-log slice shows it. [postgres.c#aborted-transaction-filter](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L1091-L1105)

**Bash never builds a single-quoted literal.** Each value it sends, such as the schema name, the role name, an error message or a plan, goes between dollar-quote tags that do not occur in the value. Inside a [dollar-quoted](../../../glossary.md#dollar-quoting) string the server applies no escape processing. psql treats the text as a literal, because its lexer recognizes backslash commands and variable references only outside quotes. [syntax.sgml#dollar-quoting](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L563-L588) [syntax.sgml#tag-rules](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L610-L621) [scan.l#dolqdelim](../../../../raw/postgres-12/src/backend/parser/scan.l#L292-L294) [psqlscan.l#dollar-quote](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L551-L588) [psqlscan.l#backslash-and-variables](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L702-L766) [psql-ref.sgml#no-interpolation-in-quotes](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L4055-L4061)

**psql rewrites unquoted variable references, so the runner guards the one text it does not write.** Outside quotes, psql replaces `:name`, `:'name'` and `:"name"` whenever a psql variable of that name exists, and psql always defines variables such as `ERROR`. Its lexer reads `::` as a typecast token first, so a cast is never taken for a reference. [psqlscan.l#variable-substitution](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L708-L762) [psqlscan.l#typecast](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L301) [psql-ref.sgml#interpolation](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L4020-L4053)

The runner's own statements are fixed text with dollar-quoted values. The generated statements are fixed text too, except for the deparsed key expressions inside each count statement. The deparser prints an array slice with no space around its colon, so a slice whose bound is a quoted column becomes `a[1:"ERROR"]`, which psql would rewrite. [ruleutils.c#printSubscripts](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10614-L10637) The fixtures measure the hazard: through psql, `(ARRAY[1, 2, 3])[1:"ERROR"]` fails with SQLSTATE `42601`, while `[1 : "ERROR"]` returns `{1,2}`.

Therefore the runner checks each count statement for a colon that does not follow another colon and is followed by `'`, `"` or `{`. When it finds one, it sends only the attempt's prelude, which inserts the attempt row, sets the savepoint and applies the scan profile. It then rolls back to the savepoint and records an `execution` error with no SQLSTATE and the message “runner did not send the count statement”. Under the status rules above, an error that no accepted scan covers makes the relation `ERROR`, never `PASS`. The check is conservative: the same characters inside a string constant are refused too.

### The runner's control flow

The runner's decisions, in order. Every arrow is a Bash branch on the request's failed flag, the reply or psql ending.

```text
validate HIC_* environment ─ invalid ──────────────────────────────> exit 64, no connection
open psql on two named pipes; SELECT 1 ─ psql ends ───────────────> exit 2
[HIC_ROLE] SET ROLE <dollar-quoted name> ─ fails ─────────────────> exit 2
BEGIN; installation block; COMMIT; progress = CONFIGURE_AND_START ─ no ─> exit 2
INSERT hic_config (schema, ddl_stable = true, options)
start script, part 1: BEGIN REPEATABLE READ READ ONLY ... hic_run
    └ fails ─> ROLLBACK; hic_run_error 'initialization'; export run errors ─> exit 2
start script, part 2: inventory INSERTs; inventory_complete
    └ fails ─> ROLLBACK; hic_run_error 'inventory'; export run errors ──────> exit 2
bound = 2 × heap-scope relations + 2 × eligible indexes
┌─> progress state
│     ATTEMPTS_COMPLETE_REVIEW_REPORT ─────────────────────> finish
│     not ATTEMPT_AVAILABLE ───────────────────────────────> stop
│   fetch attempt id, sql_text, prelude, count statement; id > bound ─> stop
│   count statement has colon + quote or brace?
│     yes ─> send prelude; recover(execution, no SQLSTATE); controls ─┐
│   send sql_text unchanged                                          │
│     failed ─> recover(execution, parsed error); controls ──────────┤
│     no JSON or plan not saved ─> recover(capture); controls ───────┤
│   send controls (post_count_sql); save plans/attempt-<id>.json     │
│   INSERT hic_plan ($tag$<json>$tag$::jsonb)                        │
│     failed ─> recover(capture, parsed error); controls ────────────┤
│   SELECT verdict FROM hic_attempt_result                           │
│     failed ─> recover(inspection, parsed error); controls ─────────┤
│   RELEASE SAVEPOINT hic_attempt                                    │
└────────────────────────────────────────────────────────────────────┘
recover = ROLLBACK TO SAVEPOINT; INSERT hic_error; RELEASE
    └ rollback fails ─> stop (the transaction is lost)
stop: SELECT 1 works ─> finish, then exit 2
      SELECT 1 fails ─> ROLLBACK; hic_run_error 'snapshot'; export run errors ─> exit 2
finish: UPDATE finished; COPY seven reports; COMMIT; same schema_success after COMMIT?
    └ no ─> exit 2
exit 0 when schema_success is true, else 1; psql ending anywhere ─> exit 2
```

1. **Before connecting**, Bash checks the environment: a schema, `HIC_DDL_STABLE=yes`, `yes` or `no` for the two options, a readable SQL file and a new output directory. Any failure exits 64 before psql starts.
2. **The session** starts psql with `-X -w -q -A -t`, `ON_ERROR_STOP=0` and `VERBOSITY=verbose`: no startup file, no password prompt, no command tags, unaligned rows without headers, continue after errors, and verbose error reports. A first `SELECT 1` must return `1`. An optional `SET ROLE` takes the name as a dollar-quoted string, which the grammar accepts for a role. [psql-ref.sgml#no-psqlrc](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L553-L563) [psql-ref.sgml#no-password](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L496-L514) [psql-ref.sgml#quiet](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L374-L387) [psql-ref.sgml#no-align](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L65-L75) [psql-ref.sgml#tuples-only](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L433-L443) [psql-ref.sgml#set-variable](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L469-L484) [psql-ref.sgml#ON_ERROR_STOP](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3844-L3861) [psql-ref.sgml#VERBOSITY](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3984-L3996) [gram.y#SET-ROLE](../../../../raw/postgres-12/src/backend/parser/gram.y#L1524-L1531)
3. **Installation** runs the page's SQL block between `BEGIN` and `COMMIT`. A failed statement aborts the block, so `COMMIT` ends it with a rollback and the confirming query on `hic_progress` fails because the views do not exist. The runner then stops before configuring anything.
4. **Initialization** fetches the generated start script and sends it in two parts, split at the first inventory `INSERT`, so a failure is recorded under the right stage. A missing schema fails part 1 on the run row's `schema_oid <> 0` check with SQLSTATE `23514`.
5. **Each attempt** sends `sql_text` exactly as fetched. On success the runner first sends `post_count_sql`, which restores planner controls before the metadata queries, as operator step 5 does. It then saves the plan file, imports it, reads the validator's verdict and releases the savepoint. The verdict is read before the release, because a savepoint that no longer exists cannot be rolled back to; see [operator step 6](#operator-workflow-and-implementation-policy).
6. **Recovery** rolls back to `hic_attempt`, records `hic_error` with the parsed SQLSTATE, message, detail and hint, and releases the savepoint. The rollback succeeds only while the savepoint exists: with no savepoint, the server raises `savepoint "hic_attempt" does not exist` and the transaction stays aborted, so the runner stops. [xact.c#RollbackToSavepoint](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4249-L4308) [xact.c#ReleaseSavepoint](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4140-L4221)
7. **Finishing** sets `finished` and exports seven reports with `COPY (query) TO STDOUT`. A read-only transaction allows this, because the read-only test in `COPY` applies only to `COPY FROM` into a non-temporary table. psql writes the COPY data to its query output, which is the reply pipe. The runner commits, reads `schema_success` again, and requires the committed report to keep the verdict. [copy.c#read-only-COPY-FROM](../../../../raw/postgres-12/src/backend/commands/copy.c#L1060-L1066) [common.c#copy-out-stream](../../../../raw/postgres-12/src/bin/psql/common.c#L1097-L1130) [copy.c#handleCopyOut](../../../../raw/postgres-12/src/bin/psql/copy.c#L437-L467)
8. **Closing** ends psql's input. psql reaches end of file and exits normally, and the backend shuts down the connection; its exit callback aborts any transaction that is still open. [mainloop.c#end-of-input](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L172-L190) [postgres.c#client-EOF](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4433-L4455) [postinit.c#ShutdownPostgres](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L1189-L1193)

### Runner branches and failure handling

| Condition / state | Behavior | Consequence |
|---|---|---|
| Missing schema name, no `HIC_DDL_STABLE=yes`, a bad `yes`/`no` value, unreadable SQL file, existing output directory | Refuse before starting psql | Exit 64; measured without creating an output directory |
| psql cannot connect | No sync line arrives for the first `SELECT 1` | Exit 2; the session's `stderr.log` keeps psql's message |
| A statement in the installation fails | The whole block rolls back | Exit 2 with no checker objects |
| The schema does not exist | Start script part 1 fails with `23514`; `ROLLBACK`; `hic_run_error` row with stage `initialization` | Exit 2; measured, and `run-errors.csv` is exported |
| An inventory statement fails | Same, with stage `inventory` | Exit 2 |
| The count statement fails, for example forced RLS (`42501`) or a lock wait past `lock_timeout` (`55P03`) | Recover with stage `execution` and the parsed report | That attempt earns nothing; measured for both codes, including the RLS hint |
| psql printed no plan, or the runner cannot save it | Recover with stage `capture` | No count credit |
| The server rejects the imported plan, such as truncated JSON | Recover with stage `capture`; truncated JSON gives `22P02` with a DETAIL | Measured; the retry passes. jsonb input runs the JSON parser, whose end-of-input report carries that code and DETAIL. [jsonb.c#jsonb_from_cstring](../../../../raw/postgres-12/src/backend/utils/adt/jsonb.c#L238-L257) [json.c#report_parse_error](../../../../raw/postgres-12/src/backend/utils/adt/json.c#L1126-L1137) |
| Reading the verdict fails | Recover with stage `inspection`, still inside the savepoint | No count credit |
| The count statement contains a colon followed by a quote or brace | Send only the prelude; recover with stage `execution` and no SQLSTATE | Measured: relation `ERROR`, no count sent |
| `ROLLBACK TO SAVEPOINT` fails | The transaction is lost; `ROLLBACK`; `hic_run_error` with stage `snapshot` | Exit 2 |
| Progress state `INVALID_RUN`, `NOT_ACTIVE` or `ERROR`, or an attempt beyond the bound | Stop; a `SELECT 1` probe still works, so the runner exports the reports and commits | Exit 2 |
| The backend ends, for example through `pg_terminate_backend()` | psql sees the lost connection and, being non-interactive, exits with status 2 before printing the sync line | Exit 2 and no report; measured with `FATAL:  57P01:`. [common.c#CheckConnection](../../../../raw/postgres-12/src/bin/psql/common.c#L386-L397) [psql-ref.sgml#exit-status](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L635-L643) [signalfuncs.c#pg_terminate_backend](../../../../raw/postgres-12/src/backend/storage/ipc/signalfuncs.c#L128-L137) [postgres.c#administrator-termination](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3008-L3011) |
| The committed report changes `schema_success` | Stop | Exit 2 |
| Interrupt or termination of the runner | Its exit trap stops psql and closes the pipe | The backend aborts the transaction when it reads end of input, so no partial run commits; a statement already running is bounded by its 15-minute `statement_timeout` |

The run's state changes at these events:

| Event | State before | State after | Effect on later steps |
|---|---|---|---|
| Session opens | No psql process | psql connected, no transaction | Requests can be sent |
| Installation commits | No checker objects | Temporary tables, views and SQL functions for this session | Survive `COMMIT`, vanish with the session |
| Start script part 1 | Autocommit | Read-only repeatable-read transaction with its snapshot and run row | Every later count uses this snapshot |
| Attempt prelude | No savepoint | Attempt row, then savepoint `hic_attempt` with the scan profile | A failure after this point is recoverable |
| Release or recovery | Savepoint open | Savepoint gone; evidence kept, or rolled back with an error row | The generator sees the attempt as complete |
| `COMMIT` | Checking transaction | Autocommit; reports still readable in the session | No new attempt can start |
| Session closes | Temporary objects | Gone; the exported CSV files and plan files remain | A new run starts from installation |

Key interaction: the checker's SQL decides *what* counts as evidence, while Bash decides *what happens next*. The SQL cannot see a client-side failure, and Bash cannot see a verdict without asking. Therefore every Bash branch either reads a SQL value or records a SQL row, and the generator's view of the run stays complete.

Final causal chain: environment check → one psql session on two pipes → installation → configuration → start script with one snapshot → for each generated attempt: unchanged count statement, plan file, import and verdict inside its savepoint, or a recorded error → reports exported → `COMMIT` → exit status from `schema_success`. A lost transaction or session ends the run with status 2 instead of a report.

### Running the runner

Extract both blocks from this page, then run the runner with a libpq connection environment. Run it from any directory; it writes only below its output directory.

```text
fence=$(printf '\140\140\140')
page=wiki/v12/questions/indexing/sql-heap-index-visible-count-checker-plan.md
sed -n '/^### Executable SQL installation$/,/^### /p' "$page" | sed -n "/^${fence}sql$/,/^${fence}$/p" | sed '1d;$d' > hic12-checker.sql
sed -n '/^### Runner script$/,/^##/p' "$page" | sed -n "/^${fence}bash$/,/^${fence}$/p" | sed '1d;$d' > hic12-check.sh
HIC_SCHEMA=app HIC_DDL_STABLE=yes PGHOST=/var/run/postgresql PGDATABASE=appdb bash hic12-check.sh
```

| Item | Contract |
|---|---|
| Environment | `HIC_SCHEMA`, required; `HIC_DDL_STABLE=yes`, required; `HIC_INCLUDE_EXTERNAL_PARTITIONS` and `HIC_EXHAUSTIVE`, `yes` or `no`, default `no`; `HIC_SQL`, default `hic12-checker.sql` beside the runner; `HIC_OUT`, default `hic12-run-<UTC time>-<pid>` in the current directory; `HIC_PSQL`, default `psql`; `HIC_ROLE`, optional role for `SET ROLE`. The connection comes from libpq's environment, such as `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGSERVICE` and `PGPASSFILE`. [libpq.sgml#environment](../../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L7296-L7319) |
| Prerequisites | Bash with `mkfifo`, `sed`, `grep`, `tail`, `wc`, `cp`, `tr`, `cat` and `date`; no Bash 4 features are used. The psql measured here is the 12.2 build from the pin. The server must be the 12.2 primary that the SQL block's version guard accepts. |
| Output | A new `0700` directory: `summary.txt` first, then `schema.csv`, `relations.csv`, `candidates.csv`, `attempts.csv`, `parents.csv`, `run-errors.csv` and `run.csv`; `plans/attempt-<id>.json`; and `session-main/` with every request in `transcript.sql` and psql's errors in `stderr.log`. |
| Exit status | 0, run completed with `schema_success`; 1, run completed without it; 2, run did not complete; 64, refused before connecting. |
| Settings | `application_name` defaults to `hic12-check` through `PGAPPNAME` unless the caller sets it; it is a connection-time label, not a planner input. The client-side psql variables are `ON_ERROR_STOP=0`, psql's default, and `VERBOSITY=verbose` instead of `default`. psql runs with `LC_ALL=C`, which affects only psql's own message translations; the server's messages follow its `lc_messages`. The checking settings are the ones the generated SQL sets, listed under [settings and query shapes](#settings-and-query-shapes). [libpq.sgml#PGOPTIONS](../../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L7410-L7419) |
| Safety | No password is placed on a command line; use a password file or service file. The output directory can contain data values that error messages include, such as key values in a DETAIL. The runner writes only to its own temporary tables, and it never stops or retries after a lost session. |

The parser reads the SQLSTATE from the report itself, so a server whose `lc_messages` translates the severity word still parses. The `DETAIL:` and `HINT:` labels come from libpq, which `LC_ALL=C` keeps in English.

### Runner script

```bash
# hic12-check.sh: unattended Bash and SQL runner for the PostgreSQL 12.2
# heap and index visible-count checker. Bash makes every decision. psql
# receives SQL statements, plus one \echo sync line per request that prints
# psql's ERROR and SQLSTATE variables.
#
#   HIC_SCHEMA=app HIC_DDL_STABLE=yes bash hic12-check.sh
#
# The connection comes from the standard libpq environment (PGHOST, PGPORT,
# PGDATABASE, PGUSER, PGSERVICE, PGPASSFILE). Exit status: 0 run completed
# with schema_success; 1 run completed without it; 2 run did not complete;
# 64 usage error. Sourcing this file defines the functions without running.

: "${HIC_PSQL:=psql}"
: "${HIC_S:=main}"
HIC_OPEN=${HIC_OPEN:-}
HIC_ERROR_LINE='^[^ ]+:  ([1-9A-Z][0-9A-Z]|0[3-9A-Z])[0-9A-Z]{3}: '
HIC_GUARD='(^|[^:]):['"'"'"{]'

hic_say() { printf 'hic12-check: %s\n' "$*"; }
hic_warn() { printf 'hic12-check: %s\n' "$*" >&2; }

# ---- session layer: one psql process per named session --------------------

hic_open() {
    # hic_open NAME FD_IN FD_OUT: start psql reading SQL from a FIFO.
    local name=$1 fdin=$2 fdout=$3 dir other closers=
    dir="$HIC_OUT/session-$name"
    trap '' PIPE
    mkdir -m 700 "$dir" && mkfifo -m 600 "$dir/in" "$dir/out" || return 2
    : > "$dir/stderr.log"
    : > "$dir/transcript.sql"
    for other in $HIC_OPEN; do
        eval "closers=\"\$closers \${HIC_FDIN_$other}>&- \${HIC_FDOUT_$other}<&-\""
    done
    eval "LC_ALL=C \"\$HIC_PSQL\" -X -w -q -A -t -v ON_ERROR_STOP=0 -v VERBOSITY=verbose \
        < \"\$dir/in\" > \"\$dir/out\" 2>> \"\$dir/stderr.log\" $closers &"
    eval "HIC_PID_$name=\$! HIC_FDIN_$name=$fdin HIC_FDOUT_$name=$fdout"
    eval "HIC_DIR_$name=\$dir HIC_SEQ_$name=0"
    eval "exec $fdin>\"\$dir/in\" $fdout<\"\$dir/out\""
    HIC_OPEN="$HIC_OPEN $name"
    if hic_sql "$name" 'SELECT /* wiki_hic12_runner_hello */ 1;' && [ "$HIC_VALUE" = 1 ]; then
        return 0
    fi
    hic_warn "session $name did not start; see $dir/stderr.log"
    return 2
}

hic_close() {
    # hic_close NAME [kill]: end of input lets psql exit; the server then
    # rolls back any transaction that is still open.
    local name=$1 pid fdin fdout rest= other rc=0
    eval "pid=\${HIC_PID_$name:-} fdin=\${HIC_FDIN_$name:-} fdout=\${HIC_FDOUT_$name:-}"
    [ -n "$pid" ] || return 0
    if [ "${2:-}" = kill ]; then kill "$pid" 2> /dev/null; fi
    eval "exec $fdin>&-"
    wait "$pid" || rc=$?
    eval "exec $fdout<&-"
    eval "HIC_PID_$name="
    for other in $HIC_OPEN; do
        [ "$other" = "$name" ] || rest="$rest $other"
    done
    HIC_OPEN=$rest
    return "$rc"
}

hic_cleanup() {
    local name
    for name in $HIC_OPEN; do
        hic_close "$name" kill || :
    done
}

hic_request() {
    # hic_request NAME FILE: send FILE and one \echo sync line, then copy the
    # stdout lines that precede the sync line into the session's response
    # file and this request's stderr into stderr.last. Sets HIC_ERROR and
    # HIC_SQLSTATE (last statement) and HIC_FAILED (any statement). Returns 2
    # when psql ends first or the request does not end with a semicolon.
    local name=$1 file=$2 dir fdin fdout seq marker line before status=
    eval "dir=\$HIC_DIR_$name fdin=\$HIC_FDIN_$name fdout=\$HIC_FDOUT_$name seq=\$HIC_SEQ_$name"
    HIC_ERROR= HIC_SQLSTATE= HIC_FAILED=true
    : > "$dir/response"
    : > "$dir/stderr.last"
    if ! grep -v -E '^[[:space:]]*$' "$file" | tail -n 1 | grep -q -E ';[[:space:]]*$'; then
        hic_warn "refusing request without a final semicolon: $file"
        return 2
    fi
    seq=$((seq + 1))
    eval "HIC_SEQ_$name=$seq"
    marker="HIC12_SYNC_$$_${name}_$seq"
    before=$(wc -c < "$dir/stderr.log")
    { printf '%s\n' "-- request $seq"; cat "$file"; } >> "$dir/transcript.sql"
    if ! { cat "$file"; printf '\n\\echo %s :ERROR :SQLSTATE\n' "$marker"; } >&"$fdin" 2> /dev/null; then
        return 2
    fi
    while IFS= read -r line; do
        case $line in
            "$marker "*) status=${line#"$marker "}; break ;;
        esac
        printf '%s\n' "$line"
    done <&"$fdout" > "$dir/response"
    tail -c +"$((before + 1))" "$dir/stderr.log" > "$dir/stderr.last"
    [ -n "$status" ] || return 2
    HIC_ERROR=${status%% *}
    HIC_SQLSTATE=${status#* }
    printf '%s\n' "-- sync $status" >> "$dir/transcript.sql"
    if [ "$HIC_ERROR" = false ] && ! grep -E -q "$HIC_ERROR_LINE" "$dir/stderr.last"; then
        HIC_FAILED=false
    fi
    return 0
}

hic_sql() {
    # hic_sql NAME TEXT: returns 0 when every statement succeeded, 1 when one
    # failed, 2 when the session is gone. HIC_VALUE is the first result line.
    local name=$1 dir
    eval "dir=\$HIC_DIR_$name"
    printf '%s\n' "$2" > "$dir/request"
    HIC_VALUE=
    hic_request "$name" "$dir/request" || return 2
    if [ -s "$dir/response" ]; then IFS= read -r HIC_VALUE < "$dir/response" || :; fi
    [ "$HIC_FAILED" = false ] || return 1
}

hic_parse_error() {
    # Sets HIC_ERR_CODE, HIC_ERR_MESSAGE, HIC_ERR_DETAIL and HIC_ERR_HINT from
    # the first error report that psql printed for the last request.
    local f start end first rest
    eval "f=\$HIC_DIR_$1/stderr.last"
    HIC_ERR_CODE= HIC_ERR_MESSAGE= HIC_ERR_DETAIL= HIC_ERR_HINT=
    start=$(grep -n -E "$HIC_ERROR_LINE" "$f" | sed -n '1s/:.*//p')
    [ -n "$start" ] || return 1
    first=$(sed -n "${start}p" "$f")
    rest=${first#*:  }
    HIC_ERR_CODE=${rest%%:*}
    HIC_ERR_MESSAGE=${rest#*: }
    end=$(sed -n "$((start + 1)),\$p" "$f" | grep -n -E '^[^ ]+:  [0-9A-Z]{5}: ' | sed -n '1s/:.*//p')
    if [ -n "$end" ]; then end=$((start + end - 1)); else end='$'; fi
    HIC_ERR_DETAIL=$(sed -n "${start},${end}p" "$f" | sed -n 's/^DETAIL:  //p' | sed -n 1p)
    HIC_ERR_HINT=$(sed -n "${start},${end}p" "$f" | sed -n 's/^HINT:  //p' | sed -n 1p)
    return 0
}

hic_literal() {
    # Dollar-quote $1 with a tag that does not occur in it.
    local n=0 tag
    while :; do
        tag="\$hic12_q$n\$"
        case $1 in *"$tag"*) n=$((n + 1)) ;; *) break ;; esac
    done
    HIC_LITERAL="$tag$1$tag"
}

hic_fetch() {
    # hic_fetch FILE SQL: save a multi-line text value returned by SQL.
    local dir
    eval "dir=\$HIC_DIR_$HIC_S"
    hic_sql "$HIC_S" "$2" || return
    [ -s "$dir/response" ] || return 1
    cp "$dir/response" "$1"
}

# ---- checker layer ----------------------------------------------------------

hic_install() {
    # Install the page's SQL block in one transaction, then confirm it.
    local dir
    eval "dir=\$HIC_DIR_$HIC_S"
    {
        printf '%s\n' 'BEGIN /* wiki_hic12_runner_install */;'
        cat "$1"
        printf '%s\n' 'COMMIT /* wiki_hic12_runner_installed */;'
    } > "$dir/install.sql"
    hic_request "$HIC_S" "$dir/install.sql" || return 2
    [ "$HIC_FAILED" = false ] || hic_parse_error "$HIC_S" || :
    hic_sql "$HIC_S" 'SELECT /* wiki_hic12_runner_install_check */ state FROM pg_temp.hic_progress;' || return
    [ "$HIC_VALUE" = CONFIGURE_AND_START ]
}

hic_configure() {
    # hic_configure SCHEMA EXTERNAL EXHAUSTIVE (booleans as true or false).
    hic_literal "$1"
    hic_sql "$HIC_S" "INSERT /* wiki_hic12_runner_config */ INTO pg_temp.hic_config(target_schema, ddl_stable, include_external_partitions, exhaustive) VALUES ($HIC_LITERAL, true, $2, $3);"
}

hic_run_error() {
    # Record a run-level failure outside the checking transaction.
    local code=NULL
    case $2 in [0-9A-Z][0-9A-Z][0-9A-Z][0-9A-Z][0-9A-Z]) code="'$2'" ;; esac
    hic_literal "${3:-run failed without a server message}"
    hic_sql "$HIC_S" "INSERT /* wiki_hic12_runner_run_error */ INTO pg_temp.hic_run_error(event_id, requested_schema, stage, sqlstate, message) SELECT COALESCE((SELECT max(event_id) FROM pg_temp.hic_run_error), 0) + 1, target_schema, '$1', $code, $HIC_LITERAL FROM pg_temp.hic_config;"
}

hic_initialize() {
    # Run the generated start script in its two halves so that a failure is
    # attributed to initialization or to inventory. Returns 0, 1 recorded
    # failure, or 2 session lost.
    local dir stage
    eval "dir=\$HIC_DIR_$HIC_S"
    hic_fetch "$dir/start-run.sql" "SELECT /* wiki_hic12_runner_start_run */ substr(sql_text, 1, strpos(sql_text, 'INSERT /* wiki_hic12_capture_objects */') - 1) FROM pg_temp.hic_start_script WHERE strpos(sql_text, 'INSERT /* wiki_hic12_capture_objects */') > 0;" || return
    hic_fetch "$dir/start-inventory.sql" "SELECT /* wiki_hic12_runner_start_inventory */ substr(sql_text, strpos(sql_text, 'INSERT /* wiki_hic12_capture_objects */')) FROM pg_temp.hic_start_script WHERE strpos(sql_text, 'INSERT /* wiki_hic12_capture_objects */') > 0;" || return
    for stage in initialization inventory; do
        if [ "$stage" = initialization ]; then
            hic_request "$HIC_S" "$dir/start-run.sql" || return 2
        else
            hic_request "$HIC_S" "$dir/start-inventory.sql" || return 2
        fi
        if [ "$HIC_FAILED" = true ]; then
            hic_parse_error "$HIC_S" || HIC_ERR_CODE=$HIC_SQLSTATE
            HIC_STOP="$stage failed: ${HIC_ERR_MESSAGE:-no server message}"
            hic_sql "$HIC_S" 'ROLLBACK /* wiki_hic12_runner_abandon_start */;'
            [ $? -ne 2 ] || return 2
            hic_run_error "$stage" "$HIC_ERR_CODE" "$HIC_ERR_MESSAGE"
            [ $? -ne 2 ] || return 2
            return 1
        fi
    done
    hic_sql "$HIC_S" 'SELECT /* wiki_hic12_runner_started */ r.inventory_complete AND i.evidence_valid AND i.active_snapshot FROM pg_temp.hic_run AS r JOIN pg_temp.hic_run_integrity AS i USING (run_id);' || return
    [ "$HIC_VALUE" = t ] || { HIC_STOP='run did not start cleanly'; return 1; }
    hic_fetch "$dir/post-count.sql" 'SELECT /* wiki_hic12_runner_controls */ pg_temp.hic_control_profile();' || return
    hic_sql "$HIC_S" 'SELECT /* wiki_hic12_runner_bound */ 2 * count(*) FILTER (WHERE in_heap_scope) + 2 * (SELECT count(*) FROM pg_temp.hic_index WHERE eligible) FROM pg_temp.hic_object;' || return
    HIC_MAX_ATTEMPTS=$HIC_VALUE
}

hic_attempt_fetch() {
    # Read the next generated attempt. Returns 3 when none is offered.
    local dir meta
    eval "dir=\$HIC_DIR_$HIC_S"
    hic_sql "$HIC_S" 'SELECT /* wiki_hic12_runner_state */ state FROM pg_temp.hic_progress;' || return
    case $HIC_VALUE in
        ATTEMPT_AVAILABLE) ;;
        ATTEMPTS_COMPLETE_REVIEW_REPORT) return 3 ;;
        *) HIC_STOP="progress state $HIC_VALUE"; return 1 ;;
    esac
    hic_sql "$HIC_S" "SELECT /* wiki_hic12_runner_next */ attempt_id || ' ' || phase || ' ' || relation_oid || ' ' || COALESCE(intended_index_oid::text, '-') FROM pg_temp.hic_next;" || return
    meta=$HIC_VALUE
    HIC_ATTEMPT_ID=${meta%% *}; meta=${meta#* }
    HIC_ATTEMPT_PHASE=${meta%% *}; meta=${meta#* }
    HIC_ATTEMPT_RELATION=${meta%% *}
    HIC_ATTEMPT_INDEX=${meta#* }
    case $HIC_ATTEMPT_ID in ''|*[!0-9]*) HIC_STOP='no attempt identifier'; return 1 ;; esac
    if [ "$HIC_ATTEMPT_ID" -gt "${HIC_MAX_ATTEMPTS:-0}" ]; then
        HIC_STOP="attempt $HIC_ATTEMPT_ID exceeds the bound of $HIC_MAX_ATTEMPTS"
        return 1
    fi
    hic_fetch "$dir/attempt.sql" 'SELECT /* wiki_hic12_runner_statement */ sql_text FROM pg_temp.hic_next;' || return
    hic_fetch "$dir/prelude.sql" "SELECT /* wiki_hic12_runner_prelude */ substr(sql_text, 1, strpos(sql_text, chr(10) || 'EXPLAIN /* wiki_hic12_executed_count */')) FROM pg_temp.hic_next WHERE strpos(sql_text, chr(10) || 'EXPLAIN /* wiki_hic12_executed_count */') > 0;" || return
    hic_fetch "$dir/count.sql" "SELECT /* wiki_hic12_runner_count */ substr(sql_text, strpos(sql_text, chr(10) || 'EXPLAIN /* wiki_hic12_executed_count */') + 1) FROM pg_temp.hic_next WHERE strpos(sql_text, chr(10) || 'EXPLAIN /* wiki_hic12_executed_count */') > 0;" || return
}

hic_attempt_execute() {
    # Send the generated statement unchanged and save its JSON plan. Returns
    # 4 when a statement failed and 5 when no plan could be kept; both leave
    # recovery to the caller.
    local dir plans
    eval "dir=\$HIC_DIR_$HIC_S"
    plans="$HIC_OUT/plans"
    hic_request "$HIC_S" "$dir/attempt.sql" || return 2
    if [ "$HIC_FAILED" = true ]; then
        hic_parse_error "$HIC_S" || HIC_ERR_CODE=$HIC_SQLSTATE
        return 4
    fi
    HIC_ERR_CODE= HIC_ERR_DETAIL= HIC_ERR_HINT=
    if [ ! -s "$dir/response" ]; then
        HIC_ERR_MESSAGE='psql printed no EXPLAIN output'
        return 5
    fi
    if ! { mkdir -p "$plans" && cp "$dir/response" "$plans/attempt-$HIC_ATTEMPT_ID.json"; }; then
        HIC_ERR_MESSAGE="runner could not save the EXPLAIN output under $plans"
        return 5
    fi
}

hic_attempt_import() {
    # Import the saved plan as a dollar-quoted literal. Returns 4 when the
    # server rejects it and 5 when the saved plan is missing.
    local dir json n=0 tag
    eval "dir=\$HIC_DIR_$HIC_S"
    json="$HIC_OUT/plans/attempt-$HIC_ATTEMPT_ID.json"
    if [ ! -s "$json" ]; then
        HIC_ERR_CODE= HIC_ERR_MESSAGE="saved plan is missing: $json" HIC_ERR_DETAIL= HIC_ERR_HINT=
        return 5
    fi
    while grep -F -q -e "\$hic12_plan$n\$" "$json"; do n=$((n + 1)); done
    tag="\$hic12_plan$n\$"
    {
        printf 'INSERT /* wiki_hic12_runner_import_plan */ INTO pg_temp.hic_plan(attempt_id, document) VALUES (%s, %s' "$HIC_ATTEMPT_ID" "$tag"
        cat "$json"
        printf '%s::jsonb);\n' "$tag"
    } > "$dir/import.sql"
    hic_request "$HIC_S" "$dir/import.sql" || return 2
    if [ "$HIC_FAILED" = true ]; then
        hic_parse_error "$HIC_S" || HIC_ERR_CODE=$HIC_SQLSTATE
        return 4
    fi
}

hic_attempt_inspect() {
    # Read the validator's verdict while the savepoint still exists, so a
    # failure here can still be rolled back. Returns 4 on failure.
    hic_sql "$HIC_S" "SELECT /* wiki_hic12_runner_inspect */ 'accepted=' || accepted || ' count=' || COALESCE(row_count::text, '-') || ' actual_index=' || COALESCE(actual_index_oid::text, '-') || ' reasons=' || COALESCE(NULLIF(array_to_string(rejection_reasons, ','), ''), '-') FROM pg_temp.hic_attempt_result WHERE attempt_id = $HIC_ATTEMPT_ID;"
    case $? in
        0) HIC_VERDICT=$HIC_VALUE ;;
        2) return 2 ;;
        *) hic_parse_error "$HIC_S" || HIC_ERR_CODE=$HIC_SQLSTATE; return 4 ;;
    esac
}

hic_record_error() {
    # hic_record_error STAGE CODE MESSAGE DETAIL HINT for the current attempt.
    local code=NULL message detail=NULL hint=NULL
    case $2 in [0-9A-Z][0-9A-Z][0-9A-Z][0-9A-Z][0-9A-Z]) code="'$2'" ;; esac
    hic_literal "${3:-error without a server message}"; message=$HIC_LITERAL
    if [ -n "$4" ]; then hic_literal "$4"; detail=$HIC_LITERAL; fi
    if [ -n "$5" ]; then hic_literal "$5"; hint=$HIC_LITERAL; fi
    hic_sql "$HIC_S" "INSERT /* wiki_hic12_runner_record_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, detail, hint, snapshot_survived) VALUES ($HIC_ATTEMPT_ID, '$1', $code, $message, $detail, $hint, true);"
}

hic_recover() {
    # hic_recover STAGE CODE MESSAGE DETAIL HINT: roll back to the attempt's
    # savepoint, record the error and release the savepoint. Returns 1 when
    # the savepoint is gone, which means the checking transaction is lost.
    local rc
    hic_sql "$HIC_S" 'ROLLBACK /* wiki_hic12_runner_recover */ TO SAVEPOINT hic_attempt;'
    rc=$?
    if [ "$rc" -ne 0 ]; then
        HIC_STOP="savepoint recovery failed after: ${3:-an error}"
        return "$rc"
    fi
    hic_record_error "$@" || return
    hic_sql "$HIC_S" 'RELEASE /* wiki_hic12_runner_release_error */ SAVEPOINT hic_attempt;'
}

hic_post_count() {
    local dir
    eval "dir=\$HIC_DIR_$HIC_S"
    hic_request "$HIC_S" "$dir/post-count.sql" || return 2
    [ "$HIC_FAILED" = false ] || { HIC_STOP='post-count controls failed'; return 1; }
}

hic_attempt_once() {
    # One attempt: fetch, execute, restore planner controls, import, inspect,
    # release. Returns 0 processed, 1 run cannot continue, 2 session lost,
    # 3 no attempt offered.
    local dir rc stage
    eval "dir=\$HIC_DIR_$HIC_S"
    hic_attempt_fetch || return
    HIC_VERDICT=
    if grep -E -q "$HIC_GUARD" "$dir/count.sql"; then
        # psql would interpolate :'name', :"name" or :{?name} in this text.
        hic_request "$HIC_S" "$dir/prelude.sql" || return 2
        [ "$HIC_FAILED" = false ] || { HIC_STOP='attempt prelude failed'; return 1; }
        hic_recover execution '' 'runner did not send the count statement: psql would interpolate a colon followed by a quote or brace' '' 'Run this attempt through the operator workflow with a client that sends the text unchanged.' || return
        hic_post_count || return
        HIC_VERDICT='refused by the interpolation guard'
    else
        stage=execution
        hic_attempt_execute; rc=$?
        if [ "$rc" -eq 0 ]; then
            hic_post_count || return
            stage=capture
            hic_attempt_import; rc=$?
        fi
        if [ "$rc" -eq 0 ]; then
            stage=inspection
            hic_attempt_inspect; rc=$?
        fi
        case $rc in
            0)
                hic_sql "$HIC_S" 'RELEASE /* wiki_hic12_runner_accept */ SAVEPOINT hic_attempt;'
                rc=$?
                [ "$rc" -eq 0 ] || { HIC_STOP='release failed'; return "$rc"; } ;;
            4|5)
                hic_recover "$stage" "$HIC_ERR_CODE" "$HIC_ERR_MESSAGE" "$HIC_ERR_DETAIL" "$HIC_ERR_HINT" || return
                hic_post_count || return
                HIC_VERDICT="error stage=$stage sqlstate=${HIC_ERR_CODE:--}" ;;
            *) return "$rc" ;;
        esac
    fi
    hic_say "attempt $HIC_ATTEMPT_ID $HIC_ATTEMPT_PHASE relation=$HIC_ATTEMPT_RELATION intended_index=$HIC_ATTEMPT_INDEX $HIC_VERDICT"
    return 0
}

hic_attempt_loop() {
    local rc
    while :; do
        hic_attempt_once; rc=$?
        case $rc in
            0) ;;
            3) return 0 ;;
            *) return "$rc" ;;
        esac
    done
}

hic_export() {
    # hic_export NAME QUERY: write the query result as CSV with a header.
    local dir
    eval "dir=\$HIC_DIR_$HIC_S"
    hic_sql "$HIC_S" "COPY /* wiki_hic12_runner_export */ ($2) TO STDOUT WITH (FORMAT csv, HEADER true);" || return
    cp "$dir/response" "$HIC_OUT/$1.csv"
}

hic_finish() {
    # Mark the run finished, export every report, commit, and confirm that
    # the committed report keeps its verdict.
    local before
    hic_sql "$HIC_S" 'UPDATE /* wiki_hic12_runner_finish */ pg_temp.hic_run SET finished = clock_timestamp();' || return
    hic_export schema 'SELECT * FROM pg_temp.hic_schema_result' || return
    hic_export relations 'SELECT * FROM pg_temp.hic_relation_result ORDER BY relation_oid' || return
    hic_export candidates 'SELECT * FROM pg_temp.hic_candidate_result ORDER BY index_oid' || return
    hic_export attempts 'SELECT * FROM pg_temp.hic_attempt_report ORDER BY attempt_id' || return
    hic_export parents 'SELECT * FROM pg_temp.hic_parent_result ORDER BY relation_oid' || return
    hic_export run-errors 'SELECT * FROM pg_temp.hic_run_error ORDER BY event_id' || return
    hic_export run 'SELECT * FROM pg_temp.hic_run' || return
    hic_sql "$HIC_S" "SELECT /* wiki_hic12_runner_summary */ concat_ws(' ', 'schema_success=' || schema_success, 'heap_scope_all_pass=' || heap_scope_all_pass, 'physical_relations=' || physical_relations, 'passed=' || passed_relations, 'mismatched=' || mismatched_relations, 'inconclusive=' || inconclusive_relations, 'no_eligible_index=' || no_index_relations, 'error=' || error_relations, 'unsupported_table_ams=' || unsupported_table_ams, 'eligible_indexes=' || eligible_indexes, 'checked_indexes=' || checked_indexes, 'untested_indexes=' || untested_indexes, 'observed_index_mismatches=' || observed_index_mismatches, 'attempt_errors=' || attempt_errors) FROM pg_temp.hic_schema_result;" || return
    HIC_SUMMARY=$HIC_VALUE
    hic_sql "$HIC_S" 'SELECT /* wiki_hic12_runner_verdict */ schema_success FROM pg_temp.hic_schema_result;' || return
    before=$HIC_VALUE
    hic_sql "$HIC_S" 'COMMIT /* wiki_hic12_runner_commit */;' || return
    hic_sql "$HIC_S" 'SELECT /* wiki_hic12_runner_history */ schema_success FROM pg_temp.hic_schema_result;' || return
    [ "$HIC_VALUE" = "$before" ] || { HIC_STOP='committed report changed its verdict'; return 1; }
    HIC_SCHEMA_SUCCESS=$HIC_VALUE
}

hic_abandon() {
    # The checking transaction can no longer run statements: roll it back
    # and record the loss outside it.
    hic_parse_error "$HIC_S" || HIC_ERR_CODE=$HIC_SQLSTATE
    hic_sql "$HIC_S" 'ROLLBACK /* wiki_hic12_runner_abandon */;'
    [ $? -ne 2 ] || return 2
    hic_run_error snapshot "$HIC_ERR_CODE" "checking transaction lost: ${HIC_STOP:-unknown cause}"
    [ $? -ne 2 ] || return 2
    hic_export run-errors 'SELECT * FROM pg_temp.hic_run_error ORDER BY event_id' || :
}

hic_check() {
    # hic_check SCHEMA EXTERNAL EXHAUSTIVE: the whole run in the open session.
    # Returns 0 schema_success, 1 completed without it, 2 incomplete.
    local rc
    HIC_STOP= HIC_SUMMARY= HIC_SCHEMA_SUCCESS= HIC_MAX_ATTEMPTS=0
    hic_install "$HIC_SQL"; rc=$?
    if [ "$rc" -ne 0 ]; then
        HIC_STOP="installation failed: ${HIC_ERR_MESSAGE:-see the session stderr log}"
        return 2
    fi
    hic_configure "$1" "$2" "$3" || { HIC_STOP='configuration was refused'; return 2; }
    hic_initialize; rc=$?
    if [ "$rc" -ne 0 ]; then
        [ "$rc" -eq 2 ] || hic_export run-errors 'SELECT * FROM pg_temp.hic_run_error ORDER BY event_id' || :
        return 2
    fi
    hic_attempt_loop; rc=$?
    case $rc in
        0) ;;
        2) HIC_STOP=${HIC_STOP:-session lost}; return 2 ;;
        *)
            if hic_sql "$HIC_S" 'SELECT /* wiki_hic12_runner_probe */ 1;'; then
                hic_finish || :
            else
                hic_abandon || :
            fi
            return 2 ;;
    esac
    hic_finish || return 2
    [ "$HIC_SCHEMA_SUCCESS" = t ]
}

hic_main() {
    local external exhaustive rc
    set -u -o pipefail
    umask 077
    case "${1:-}" in
        -h|--help) sed -n '1,13p' "${BASH_SOURCE[0]}"; return 0 ;;
        '') ;;
        *) hic_warn "unexpected argument: $1"; return 64 ;;
    esac
    [ -n "${HIC_SCHEMA:-}" ] || { hic_warn 'set HIC_SCHEMA to the schema to check'; return 64; }
    [ "${HIC_DDL_STABLE:-}" = yes ] || { hic_warn 'set HIC_DDL_STABLE=yes to attest the DDL-stable window'; return 64; }
    case "${HIC_INCLUDE_EXTERNAL_PARTITIONS:-no}" in yes) external=true ;; no) external=false ;; *) hic_warn 'HIC_INCLUDE_EXTERNAL_PARTITIONS must be yes or no'; return 64 ;; esac
    case "${HIC_EXHAUSTIVE:-no}" in yes) exhaustive=true ;; no) exhaustive=false ;; *) hic_warn 'HIC_EXHAUSTIVE must be yes or no'; return 64 ;; esac
    case "${BASH_SOURCE[0]}" in */*) : "${HIC_SQL:=${BASH_SOURCE[0]%/*}/hic12-checker.sql}" ;; *) : "${HIC_SQL:=hic12-checker.sql}" ;; esac
    [ -r "$HIC_SQL" ] || { hic_warn "cannot read the installation SQL: $HIC_SQL"; return 64; }
    : "${HIC_OUT:=hic12-run-$(date -u +%Y%m%dT%H%M%SZ)-$$}"
    mkdir -m 700 "$HIC_OUT" || { hic_warn "cannot create a new output directory: $HIC_OUT"; return 64; }
    export PGAPPNAME="${PGAPPNAME:-hic12-check}"
    trap hic_cleanup EXIT
    trap 'exit 130' INT
    trap 'exit 143' TERM
    if hic_open main 7 8; then
        if [ -n "${HIC_ROLE:-}" ]; then
            hic_literal "$HIC_ROLE"
            hic_sql main "SET /* wiki_hic12_runner_role */ ROLE $HIC_LITERAL;" || HIC_STOP='SET ROLE failed'
        fi
        if [ -z "${HIC_STOP:-}" ]; then
            hic_check "$HIC_SCHEMA" "$external" "$exhaustive"; rc=$?
        else
            rc=2
        fi
        hic_close main || rc=2
    else
        HIC_STOP='psql session did not start'
        rc=2
    fi
    trap - EXIT
    {
        printf 'exit_status=%s\n' "$rc"
        printf 'stop_reason=%s\n' "${HIC_STOP:-}"
        printf '%s\n' "${HIC_SUMMARY:-}" | tr ' ' '\n'
    } > "$HIC_OUT/summary.txt"
    hic_say "exit $rc ${HIC_SUMMARY:-${HIC_STOP:-}} (reports in $HIC_OUT)"
    return "$rc"
}

if [ "${BASH_SOURCE[0]}" = "$0" ]; then
    hic_main "$@"
    exit
fi
```

## Measurement Script

This is the **disposable verification harness**, not an extra requirement for the operator-assisted checker. Its fixtures deliberately create objects and change rows in an isolated cluster. Do not point them at an application database. The script is Bash and SQL only: its loopback-port check is a Bash `/dev/tcp` connection attempt, which replaced the earlier embedded C probe, and it inspects no PostgreSQL data.

Save the complete Bash block as `.wiki-runtime/tmp/hic12-measure.sh`. Run it from the repository root:

```text
bash .wiki-runtime/tmp/hic12-measure.sh prepare build init verify
bash .wiki-runtime/tmp/hic12-measure.sh verify
bash .wiki-runtime/tmp/hic12-measure.sh cleanup
```

With no arguments, the default order is `prepare build init verify cleanup`. Every invocation stops its owned server on exit. Omitting `cleanup` retains the stopped build and results for a rerun; it does not leave a server running.

| Item | Contract |
|---|---|
| Purpose | Test the installed SQL and the follow-up's runner: the count-and-plan bridge, verdict rules, scope rules, concurrent DML, permissions, the native `indcheckxmin` gate, the literal-key `ORDER BY` generation, the runner's exit statuses, error parsing, recovery paths and interpolation guard. Its results back the [Measured coverage](#measured-coverage-and-remaining-test-gaps) table and the concurrency arithmetic below it. |
| `prepare` | Verify the pin and clean raw checkout, create or recognize the owned sandbox, extract this page's SQL installation block and runner block, check the runner's Bash syntax, and emit the embedded fixtures and test driver. |
| `build` | Run `prepare`, stop an already-running owned cluster, and build/install PostgreSQL out of tree. Reuses an existing build. |
| `init` | Run `prepare`; initialize UTF-8/C-locale disposable data if absent; start only the owned Unix-socket server on port 55447. |
| `verify` | Run `init`; capture platform/settings; create a new uniquely named fixture database and run every assertion. The test driver sources the extracted runner for its in-session cases and runs it as a program for its exit-status cases. Repeated invocations do not reuse fixture tables. |
| `cleanup` | Cleanly stop the owned cluster, check PID/process/socket/loopback-port teardown, then delete `.wiki-runtime/tmp/hic12/`, including its build and results. Idempotent when absent. |
| Environment | The script reads only `JOBS`, default 2. Ordinary tool lookup uses `PATH`, and `configure` honors its usual compiler variables, which the script neither sets nor reads. The script fixes `LC_ALL=C`, `PGCLIENTENCODING=UTF8`, `TMPDIR` and `PGPASSFILE` inside the sandbox, and unsets connection-service/host-address and `PGOPTIONS` overrides. The test driver exports `PGHOST`, `PGPORT`, `PGUSER`, `PGDATABASE` and `PGAPPNAME=hic12-suite` for the runner, and sets `PGOPTIONS` only while it opens each case session. |
| Prerequisites | Bash with `/dev/tcp` support, Git and POSIX utilities, including `ps` and `mkfifo`. Building the pinned Git checkout needs GNU make 3.80 or later, a C99 compiler, Bison 1.875 or later, Flex 2.5.31 or later and Perl 5.8.3 or later; the last three are needed because a Git checkout has no generated parser and scanner files. [installation.sgml#required-make-and-compiler](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L80-L97) [installation.sgml#git-checkout-tools](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L300-L318) No network, Python, external harness, extension download or planner-hint extension is used by the published script. The repository path must be short and contain no whitespace or shell metacharacters. |
| Build | `--without-readline --without-zlib`, with a private install prefix. These omit client command-line editing and zlib support; they are not planner or visibility settings. |
| Output | Read `suite-current.log` and `last-test-output` first. Each case writes the runner's output directory under `results/<unique-database>/<case>/`: the seven report CSVs, saved plans and the session transcript and error log. Program runs also write `<case>.log` and `summary.txt`. Platform facts are in `platform.txt`, `platform-sql.txt` and `build-options.txt`. `processes.txt` holds the process listing from the last teardown check. Keep a private copy before cleanup if needed. |
| Runtime | The recorded fresh run spent 104 seconds building and 61 seconds verifying, under three minutes overall; the build-retained `verify` re-run took 57 seconds. About 4 seconds of each verification are the two deliberate 2-second lock timeouts. The script prints each stage's actual `elapsed_seconds`. These are local correctness-test timings, not production-scan performance estimates. |

**Last run:** 2026-10-23, PostgreSQL 12.2 built from `45b88269a353ad93744772791feb6d01bc7e1e42`; Linux `6.18.54.1-microsoft-standard-WSL2`, `x86_64`, GCC 13.3.0, 64-bit, GNU Bash 5.2.21; block size 8192 bytes, maximum data alignment 8 bytes; UTF-8 encoding, C collation/ctype, `lc_messages` C, `standard_conforming_strings` on, data checksums disabled. Both runs passed all 124 assertions. The script records these platform facts rather than assuming them for other hosts.

Non-default setup is intentional:

| Setting or fixture | Default → this run, apply scope | Why the result depends on it |
|---|---|---|
| Scan profiles, timeouts, search path and identifier quoting | The preceding settings table gives defaults and session/transaction scope. Fixture sessions use `statement_timeout = '60s'`, validator units use `'30s'`, both with `lock_timeout = '2s'`. Each case session starts with `quote_all_identifiers = on` through `PGOPTIONS`, a connection-time session default, while counts use local off. | Count profiles seek the required path; post-count controls avoid applying scan-discouragement penalties to report queries. The quoting override deliberately exercises restoration after COMMIT, which the runner checks on every finish. [libpq.sgml#PGOPTIONS](../../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L7410-L7419) |
| Runner psql options | psql's defaults are `ON_ERROR_STOP` off and `VERBOSITY` `default`, with command tags printed → `ON_ERROR_STOP=0`, `VERBOSITY=verbose` and `-q`; client-side, per session | The runner continues after an error so that it can recover, and it reads the SQLSTATE from each verbose report. These are psql variables, not server settings. [psql-ref.sgml#ON_ERROR_STOP](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3844-L3861) [psql-ref.sgml#VERBOSITY](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3984-L3996) |
| Lock holder, terminated backend and truncated plan | No second session, no termination and an unchanged plan by default → a second session holds `ACCESS EXCLUSIVE` on `hic_lock.t`; another calls `pg_terminate_backend()` on the checking backend; the harness cuts one saved plan | These fixtures force the `55P03`, lost-session and `22P02` branches. Without them those branches stay unexercised, as they were before the follow-up. |
| psql-guard fixture | No such index by default → an index on `(a[1 : "ERROR"])`, created with spaces so that psql passes it unchanged | Its deparsed key has no spaces, which triggers the runner's guard; the measured result is a relation `ERROR`, not a pass. |
| `port` | Default 5432 → 55447; `postmaster`, startup/restart | Uses a separate test endpoint, including the Unix-socket name. [configure.in#default-port](../../../../raw/postgres-12/configure.in#L149-L162) [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2176-L2184) |
| `listen_addresses`, `unix_socket_directories`, `unix_socket_permissions` | `localhost`, `/tmp`, `0777` → empty TCP-listen list, private sandbox socket directory, `0700`; all `postmaster`, startup/restart | Postmaster opens sockets from these lists and applies the Unix-socket mode. The disposable trust-authentication cluster is not exposed through a TCP listener or a generally accessible socket. [guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3935-L3960) [guc.c#socket-mode](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2186-L2199) [pg_config_manual.h#socket-default](../../../../raw/postgres-12/src/include/pg_config_manual.h#L178-L182) [postmaster.c#TCP-list](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L1011-L1043) [postmaster.c#Unix-list](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L1109-L1138) [pqcomm.c#socket-permissions](../../../../raw/postgres-12/src/backend/libpq/pqcomm.c#L692-L699) |
| Effective role | No SET ROLE override → `hic_reader` in permission cases or `hic_dba` in administrative cases; `user`, session scope, subject to authorization | These branches must exercise actual current-role privileges, not merely label a superuser result as ordinary. [guc.c#role](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3824-L3834) [rls.c#role-check](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132) |
| `client_encoding` | Boot value SQL_ASCII, dynamically defaults to database encoding; explicitly UTF8 here through `PGCLIENTENCODING`; `user`, connection/session scope | Keeps the non-ASCII fixture text and JSON imports consistent with the UTF-8 database. [guc.c#client-encoding](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3581-L3589) [postinit.c#encoding-default](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L394-L401) |
| Heap fillfactor | Default 100 → 50 on the concurrent-write and horizon fixtures; table storage parameter, set at CREATE TABLE, not a GUC | Insertion reserves free page space; a same-page update that changes no HOT-blocking indexed attributes can use HOT. Updating the indexed `id` instead exercises non-HOT. The gate fixture changes `k` before its index exists. [rel.h#heap-fillfactor](../../../../raw/postgres-12/src/include/utils/rel.h#L278-L279) [hio.c#reserved-space](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L348-L350) [heapam.c#HOT-decision](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3591-L3601) |
| Retained old horizon and late index build | A newly created index normally starts with `indcheckxmin = false`; this fixture retains old versions while a nonconcurrent build encounters a potentially broken HOT chain | `index_build()` sets the flag for that branch. The planner rejects the index while its catalog xmin does not precede `TransactionXmin`. Ending the holder and beginning a new checker transaction changes the horizon; the flag alone need not clear. `old_snapshot_threshold` remains at its default `-1`, so early pruning is not the cause here. [index.c#initial-flag](../../../../raw/postgres-12/src/backend/catalog/index.c#L609-L615) [index.c#broken-chain-gate](../../../../raw/postgres-12/src/backend/catalog/index.c#L2921-L2971) [plancat.c#horizon](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234) |
| Forced RLS owner | New relation row-security flags start false; fixture explicitly ENABLEs and FORCEs RLS and installs the subset policy | This forces the owner through the RLS decision. `row_security = off` then errors for that role, while the authorized superuser bypass branch remains available. [relcache.c#new-relation-state](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3287-L3295) [heap.c#security-flags](../../../../raw/postgres-12/src/backend/catalog/heap.c#L916-L919) [rls.c#forced-owner](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132) |

The build normally includes readline and zlib support; the two configure options disable interactive line editing and compressed dump/archive support, respectively. Those features are unnecessary for this noninteractive SQL test. [installation.sgml#optional-libraries](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L118-L151)

The one script below contains all fixtures and test orchestration. It reads the installation block and the runner block from this same page, so there is only one maintained copy of the checker SQL and of the runner. Its fixtures send psql only SQL statements: the expected-error probes run through the runner's session functions, and Bash checks their SQLSTATEs.

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
    if [ -f "$data/postmaster.pid" ]; then
        same_server || return 1
        "$bin/pg_ctl" -D "$data" -m fast -w stop
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
    if owned && [ -f "$data/postmaster.pid" ]; then
        stop_owned || rc=1
    fi
    exit "$rc"
}
trap on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
prepare() {
    [ "$(git -C "$src" rev-parse HEAD)" = "$pin" ]
    [ -z "$(git --no-optional-locks -C "$src" status --porcelain)" ]
    ls -ld "$root/.wiki-runtime/tmp"
    if [ -e "$work" ]; then
        owned
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
        (cd "$work/build" && "$src/configure" --prefix="$work/install" --without-readline --without-zlib) \
            > "$work/configure.log" 2>&1
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
    sed -n '/^### Executable SQL installation$/,/^### /p' "$page" |
        sed -n "/^${fence}sql$/,/^${fence}$/p" |
        sed '1d;$d' > "$work/checker.sql"
    [ -s "$work/checker.sql" ]
    IFS= read -r first_line < "$work/checker.sql"
    [ "$first_line" = 'SET /* wiki_hic12_install_path */ search_path = pg_catalog, pg_temp;' ]
    sed -n '/^### Runner script$/,/^##/p' "$page" |
        sed -n "/^${fence}bash$/,/^${fence}$/p" |
        sed '1d;$d' > "$work/hic12-check.sh"
    [ -s "$work/hic12-check.sh" ]
    IFS= read -r first_line < "$work/hic12-check.sh"
    [ "$first_line" = '# hic12-check.sh: unattended Bash and SQL runner for the PostgreSQL 12.2' ]
    bash -n "$work/hic12-check.sh"
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
HIC_FIXTURES_SQL
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
          "Actual Loops":1,"Actual Rows":1,"Output":["count(*)"],
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
INSERT /* wiki_hic12_unit_validate */ INTO hic_unit_assertions
SELECT t.label, v.valid IS NOT DISTINCT FROM t.expected
FROM hic_unit_cases AS t
CROSS JOIN LATERAL pg_temp.hic_check_plan(
    t.document, t.phase, 'fixture', 'target', 'pg_temp_test') AS v;
SELECT /* wiki_hic12_unit_result */ count(*) AS plan_assertions
FROM hic_unit_assertions;
HIC_VALIDATOR_SQL
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
    cat > "$work/test.sh" <<'HIC_TEST_SH'
set -u -o pipefail
umask 077
root=$(pwd -P)
work="$root/.wiki-runtime/tmp/hic12"
port=55447
bin="$work/install/bin"
. "$work/hic12-check.sh"
HIC_PSQL="$bin/psql"
HIC_SQL="$work/checker.sql"
export PGHOST="$work/socket" PGPORT="$port" PGUSER=hic_dba PGDATABASE=postgres PGAPPNAME=hic12-suite
db="hic_suite_$(date -u +%Y%m%d%H%M%S)_$$"
out="$work/results/$db"
mkdir -p "$out" || exit 1
ls -ld "$work" "$out"
passed=0
case_name=setup
trap hic_cleanup EXIT

fail() { printf 'FAIL [%s] %s\n' "$case_name" "$*"; exit 1; }
ok() { passed=$((passed + 1)); printf 'ok %s\n' "$1"; }
sfile() { eval "printf '%s' \"\$HIC_DIR_$HIC_S/$1\""; }
expect() {
    # expect LABEL SQL: the SQL must succeed and return exactly one line, t.
    hic_sql "$HIC_S" "$2" || fail "$1: status $?: $(cat "$(sfile stderr.last)")"
    [ "$HIC_VALUE" = t ] && [ "$(grep -c '' "$(sfile response)")" -eq 1 ] \
        || fail "$1: returned $(cat "$(sfile response)")"
    ok "$1"
}
expect_error() {
    # expect_error LABEL SQLSTATE SQL: the SQL must fail with that SQLSTATE.
    local rc
    hic_sql "$HIC_S" "$3"; rc=$?
    [ "$rc" -eq 1 ] || fail "$1: expected SQLSTATE $2, request status $rc"
    hic_parse_error "$HIC_S" && [ "$HIC_ERR_CODE" = "$2" ] || fail "$1: expected $2, got ${HIC_ERR_CODE:-none}"
    ok "$1"
}
open_case() {
    # open_case NAME [ROLE]: a session whose caller sets quote_all_identifiers.
    case_name=$1
    HIC_OUT="$out/$1"
    mkdir -m 700 "$HIC_OUT" || fail 'output directory'
    PGOPTIONS='-c quote_all_identifiers=on'
    export PGOPTIONS
    hic_open main 7 8 || fail 'session did not start'
    unset PGOPTIONS
    if [ -n "${2:-}" ]; then
        hic_sql main "SET /* wiki_hic12_test_role */ ROLE $2;" || fail 'SET ROLE failed'
    fi
}
start_run() {
    hic_install "$HIC_SQL" || fail "installation: ${HIC_ERR_MESSAGE:-}"
    hic_configure "$1" "$2" "$3" || fail 'configuration'
    hic_initialize || fail "initialization: $HIC_STOP"
}
run_loop() { hic_attempt_loop || fail "attempt loop stopped: $HIC_STOP"; }
finish_case() {
    hic_finish || fail "finish: $HIC_STOP"
    expect "${case_name}_only_sql_helpers" "SELECT /* wiki_hic12_test_language */ bool_and(l.lanname = 'sql' AND NOT p.prosecdef) FROM pg_catalog.pg_proc AS p JOIN pg_catalog.pg_language AS l ON l.oid = p.prolang WHERE p.pronamespace = pg_my_temp_schema();"
}
check_case() {
    # check_case NAME SCHEMA EXTERNAL EXHAUSTIVE [ROLE]: one complete run.
    open_case "$1" "${5:-}"
    start_run "$2" "$3" "$4"
    run_loop
    finish_case
}
close_case() {
    hic_close main || fail "psql exit status $?"
    printf 'case %s closed\n' "$case_name"
}
synthetic() {
    # synthetic LABEL SETUP CHECK: change evidence inside a savepoint, read the
    # verdict, roll the change back, then assert the verdict in Bash.
    local verdict
    hic_sql main 'SAVEPOINT /* wiki_hic12_test_fixture */ verdict_fixture;' || fail "$1: savepoint"
    hic_sql main "$2" || fail "$1: setup: $(cat "$(sfile stderr.last)")"
    hic_sql main "$3" || fail "$1: check: $(cat "$(sfile stderr.last)")"
    verdict=$HIC_VALUE
    hic_sql main 'ROLLBACK /* wiki_hic12_test_restore */ TO SAVEPOINT verdict_fixture;' || fail "$1: rollback"
    hic_sql main 'RELEASE /* wiki_hic12_test_release */ SAVEPOINT verdict_fixture;' || fail "$1: release"
    [ "$verdict" = t ] || fail "$1: verdict $verdict"
    ok "$1"
}
cli() {
    # cli NAME EXPECTED_STATUS [VARIABLE=VALUE ...]: run the runner as a program.
    local name=$1 want=$2 rc
    shift 2
    env "$@" HIC_PSQL="$HIC_PSQL" HIC_SQL="$HIC_SQL" HIC_OUT="$out/$name" PGDATABASE="$db" \
        bash "$work/hic12-check.sh" > "$out/$name.log" 2>&1
    rc=$?
    case_name=$name
    [ "$rc" -eq "$want" ] || fail "exit $rc, expected $want: $(cat "$out/$name.log")"
    ok "${name}_exit_$want"
}

"$bin/psql" -X -q -v ON_ERROR_STOP=1 -c "SET /* wiki_hic12_suite_timeout */ statement_timeout = '60s'" \
    -c "SET /* wiki_hic12_suite_lock_timeout */ lock_timeout = '2s'" \
    -c "CREATE /* wiki_hic12_suite_database */ DATABASE $db" || fail 'database creation'
export PGDATABASE="$db"

case_name=fixtures
HIC_OUT="$out/fixtures"
mkdir -m 700 "$HIC_OUT" || fail 'output directory'
hic_open main 7 8 || fail 'session did not start'
hic_sql main "SELECT /* wiki_hic12_fixture_role */ count(*) FROM pg_catalog.pg_roles WHERE rolname = 'hic_reader';" || fail 'role lookup'
if [ "$HIC_VALUE" = 0 ]; then
    hic_sql main 'CREATE /* wiki_hic12_reader */ ROLE hic_reader;' || fail 'role creation'
fi
hic_request main "$work/fixtures.sql" || fail 'fixture session lost'
[ "$HIC_FAILED" = false ] || fail "fixtures: $(cat "$(sfile stderr.last)")"
ok fixtures_created
expect_error positional_order_probe 42P10 'SELECT /* wiki_hic12_positional_probe */ count(*) FROM (SELECT /* wiki_hic12_positional_order */ 1 FROM ONLY hic_negative.constant_literals ORDER BY ((2)) USING OPERATOR(pg_catalog.<) NULLS LAST) AS probe;'
expect_error decimal_order_probe 42601 'SELECT /* wiki_hic12_decimal_probe */ count(*) FROM (SELECT /* wiki_hic12_decimal_order */ 1 FROM ONLY hic_negative.constant_literals ORDER BY ((1.5)) USING OPERATOR(pg_catalog.<) NULLS LAST) AS probe;'
expect cast_order_probe 'SELECT /* wiki_hic12_cast_probe */ count(*) = 2 FROM (SELECT /* wiki_hic12_cast_order */ 1 FROM ONLY hic_negative.constant_literals ORDER BY CAST((2) AS integer) USING OPERATOR(pg_catalog.<) NULLS LAST, CAST((1.5) AS numeric) USING OPERATOR(pg_catalog.<) NULLS LAST) AS probe;'
expect_error invalid_concurrent_build 23505 'CREATE /* wiki_hic12_invalid_index */ UNIQUE INDEX CONCURRENTLY invalid_idx ON hic_negative.invalid_index(id);'
expect invalid_flag_left_false "SELECT /* wiki_hic12_invalid_flag */ NOT indisvalid FROM pg_catalog.pg_index WHERE indexrelid = 'hic_negative.invalid_idx'::regclass;"
expect_error psql_rewrites_colon_quote 42601 'SELECT /* wiki_hic12_psql_rewrite_probe */ (ARRAY[1, 2, 3])[1:"ERROR"] FROM (SELECT 2 AS "ERROR") AS s;'
expect psql_keeps_spaced_colon 'SELECT /* wiki_hic12_psql_spaced_probe */ (ARRAY[1, 2, 3])[1 : "ERROR"] = ARRAY[1, 2] FROM (SELECT 2 AS "ERROR") AS s;'
expect guard_key_deparses_unspaced "SELECT /* wiki_hic12_guard_deparse */ pg_catalog.pg_get_indexdef('hic_psql_guard.psql_guard_idx'::regclass, 1, false) = '(a[1:\"ERROR\"])';"
close_case

case_name=validator
HIC_OUT="$out/validator"
mkdir -m 700 "$HIC_OUT" || fail 'output directory'
hic_open main 7 8 || fail 'session did not start'
hic_install "$HIC_SQL" || fail 'installation'
hic_request main "$work/validator-tests.sql" || fail 'validator session lost'
[ "$HIC_FAILED" = false ] || fail "validator units: $(cat "$(sfile stderr.last)")"
[ "$(sed -n '$p' "$(sfile response)")" = 33 ] || fail "validator units: $(cat "$(sfile response)")"
ok validator_33_plan_shape_units
close_case

cli cli_positive 0 HIC_SCHEMA=hic_positive HIC_DDL_STABLE=yes PGOPTIONS='-c quote_all_identifiers=on'
grep -F -x -q 'schema_success=true' "$out/cli_positive/summary.txt" || fail 'summary lacks schema_success=true'
grep -F -x -q 'physical_relations=13' "$out/cli_positive/summary.txt" || fail 'summary lacks 13 relations'
[ "$(ls "$out/cli_positive/plans" | grep -c -E '^attempt-[0-9]+[.]json$')" -eq 26 ] || fail 'expected 26 saved plans'
for f in schema relations candidates attempts parents run-errors run; do
    [ -s "$out/cli_positive/$f.csv" ] || fail "missing $f.csv"
done
ok cli_positive_reports
cli cli_negative 1 HIC_SCHEMA=hic_negative HIC_DDL_STABLE=yes
grep -F -x -q 'schema_success=false' "$out/cli_negative/summary.txt" || fail 'summary lacks schema_success=false'
ok cli_negative_summary
cli cli_missing_schema 2 HIC_SCHEMA=hic_missing_schema HIC_DDL_STABLE=yes
grep -F -q ',initialization,23514,' "$out/cli_missing_schema/run-errors.csv" || fail 'run error not exported'
grep -F -q 'stop_reason=initialization failed' "$out/cli_missing_schema/summary.txt" || fail 'stop reason not recorded'
ok cli_missing_schema_recorded
cli cli_no_attestation 64 HIC_SCHEMA=hic_positive
[ ! -e "$out/cli_no_attestation" ] || fail 'created an output directory before refusing'
ok cli_no_attestation_refused_before_connecting
cli cli_bad_option 64 HIC_SCHEMA=hic_positive HIC_DDL_STABLE=yes HIC_EXHAUSTIVE=maybe

check_case positive hic_positive false false
expect schema_pass 'SELECT /* wiki_hic12_positive_success */ schema_success FROM pg_temp.hic_schema_result;'
expect all_thirteen_physical_relations 'SELECT /* wiki_hic12_positive_scope */ physical_relations = 13 FROM pg_temp.hic_schema_result;'
expect empty_table "SELECT /* wiki_hic12_positive_empty */ status = 'PASS' AND heap_count = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 'empty_table';"
expect nulls_duplicates_aborts "SELECT /* wiki_hic12_positive_nulls */ status = 'PASS' AND heap_count = 5 FROM pg_temp.hic_relation_result WHERE relation_name = 'nulls_duplicates';"
expect inheritance_parent_only "SELECT /* wiki_hic12_positive_parent */ status = 'PASS' AND heap_count = 1 FROM pg_temp.hic_relation_result WHERE relation_name = 'inh_parent';"
expect inheritance_child_only "SELECT /* wiki_hic12_positive_child */ status = 'PASS' AND heap_count = 2 FROM pg_temp.hic_relation_result WHERE relation_name = 'inh_child';"
expect empty_populated_matview "SELECT /* wiki_hic12_positive_matview */ status = 'PASS' AND heap_count = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 'empty_mv';"
expect unpopulated_not_empty "SELECT /* wiki_hic12_positive_unpopulated */ status = 'NOT_APPLICABLE' FROM pg_temp.hic_relation_result WHERE relation_name = 'unpopulated_mv';"
expect quoted_names_pass "SELECT /* wiki_hic12_positive_quoted */ status = 'PASS' AND heap_count = 2 FROM pg_temp.hic_relation_result WHERE relation_name = 'Odd table \"x\"';"
expect parent_scope_and_foreign_gaps 'SELECT /* wiki_hic12_positive_parent_gap */ NOT all_descendants_in_scope AND NOT all_descendants_supported AND NOT subtree_success FROM pg_temp.hic_parent_result;'
expect outside_partition_excluded "SELECT /* wiki_hic12_positive_outside */ NOT EXISTS (SELECT 1 FROM pg_temp.hic_object WHERE schema_name = 'hic_outside');"
expect checker_objects_excluded 'SELECT /* wiki_hic12_positive_owned */ NOT EXISTS (SELECT 1 FROM pg_temp.hic_object AS obj JOIN pg_temp.hic_owned AS own USING (relation_oid));'
expect counts_have_validated_plans 'SELECT /* wiki_hic12_positive_accepted */ bool_and(accepted) AND count(*) = 26 FROM pg_temp.hic_attempt_result;'
expect thirteen_indexes_checked_and_matching 'SELECT /* wiki_hic12_positive_indexes */ eligible_indexes = 13 AND checked_indexes = 13 AND matching_indexes = 13 AND untested_indexes = 0 AND observed_index_mismatches = 0 AND all_tested_indexes_match FROM pg_temp.hic_schema_result;'
expect aborted_row_absent "SELECT /* wiki_hic12_positive_abort */ heap_count = 5 FROM pg_temp.hic_relation_result WHERE relation_name = 'nulls_duplicates';"
close_case

check_case external hic_positive true false
expect schema_pass 'SELECT /* wiki_hic12_external_success */ schema_success FROM pg_temp.hic_schema_result;'
expect all_fourteen_physical_relations 'SELECT /* wiki_hic12_external_scope */ physical_relations = 14 FROM pg_temp.hic_schema_result;'
expect outside_leaf_checked "SELECT /* wiki_hic12_external_leaf */ status = 'PASS' AND heap_count = 1 FROM pg_temp.hic_relation_result WHERE relation_name = 'partition_outside';"
expect outside_unrelated_excluded "SELECT /* wiki_hic12_external_unrelated */ NOT EXISTS (SELECT 1 FROM pg_temp.hic_object WHERE relation_name = 'unrelated');"
expect foreign_child_remains_explicit 'SELECT /* wiki_hic12_external_parent */ all_descendants_in_scope AND NOT all_descendants_supported AND NOT subtree_success FROM pg_temp.hic_parent_result;'
expect fourteen_indexes_checked 'SELECT /* wiki_hic12_external_indexes */ eligible_indexes = 14 AND checked_indexes = 14 AND matching_indexes = 14 FROM pg_temp.hic_schema_result;'
close_case

check_case negative hic_negative false false
expect schema_not_pass 'SELECT /* wiki_hic12_negative_success */ NOT schema_success FROM pg_temp.hic_schema_result;'
expect all_eight_physical_relations 'SELECT /* wiki_hic12_negative_scope */ physical_relations = 8 FROM pg_temp.hic_schema_result;'
expect four_no_eligible_index "SELECT /* wiki_hic12_negative_no_index */ count(*) = 4 AND bool_and(status = 'NO_ELIGIBLE_INDEX') FROM pg_temp.hic_relation_result WHERE relation_name IN ('no_index', 'partial_only', 'hash_only', 'invalid_index');"
expect four_inconclusive "SELECT /* wiki_hic12_negative_inconclusive */ count(*) = 4 AND bool_and(status = 'INCONCLUSIVE') FROM pg_temp.hic_relation_result WHERE relation_name IN ('constant_index', 'constant_literals', 'user_expression', 'domain_table');"
expect user_code_not_deparsed "SELECT /* wiki_hic12_negative_deparse */ definition IS NULL AND order_sql IS NULL AND generation_reason = 'unreviewed_user_code_or_type' FROM pg_temp.hic_index WHERE index_name = 'user_expression_idx';"
expect constant_order_did_not_force_index "SELECT /* wiki_hic12_negative_constant */ count(*) = 2 AND NOT bool_or(accepted) FROM pg_temp.hic_attempt_result WHERE intended_index_oid = 'hic_negative.constant_idx'::regclass;"
expect literal_keys_untested_without_errors "SELECT /* wiki_hic12_negative_literals */ count(*) = 4 AND count(DISTINCT intended_index_oid) = 2 AND NOT bool_or(accepted) AND NOT bool_or(error_present) FROM pg_temp.hic_attempt_result WHERE relation_oid = 'hic_negative.constant_literals'::regclass AND phase = 'index';"
expect literal_keys_cast "SELECT /* wiki_hic12_negative_cast */ count(*) = 3 AND bool_and(order_sql LIKE 'CAST(%') FROM pg_temp.hic_index WHERE relation_oid IN ('hic_negative.constant_index'::regclass, 'hic_negative.constant_literals'::regclass);"
close_case

check_case choice hic_choice false true
expect existential_pass_without_exhaustive_coverage 'SELECT /* wiki_hic12_choice_success */ schema_success AND NOT exhaustive_coverage_complete FROM pg_temp.hic_schema_result;'
expect actual_not_intended "SELECT /* wiki_hic12_choice_actual */ count(*) = 2 AND bool_and(accepted AND actual_index_oid = 'hic_choice.compact_idx'::regclass) FROM pg_temp.hic_attempt_result WHERE intended_index_oid = 'hic_choice.wide_idx'::regclass;"
expect wide_remains_untested "SELECT /* wiki_hic12_choice_untested */ NOT actually_checked AND intended_attempts = 2 FROM pg_temp.hic_candidate_result WHERE index_name = 'wide_idx';"
expect schema_index_totals 'SELECT /* wiki_hic12_choice_totals */ eligible_indexes = 2 AND checked_indexes = 1 AND matching_indexes = 1 AND untested_indexes = 1 AND all_tested_indexes_match AND NOT exhaustive_coverage_complete FROM pg_temp.hic_schema_result;'
close_case

check_case psql_guard hic_psql_guard false false
expect guard_order_contains_colon_quote "SELECT /* wiki_hic12_guard_order */ order_sql LIKE '%[1:\"ERROR\"]%' FROM pg_temp.hic_index WHERE index_name = 'psql_guard_idx';"
expect guard_relation_is_error "SELECT /* wiki_hic12_guard_status */ status = 'ERROR' AND heap_count = 2 AND checked_indexes = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 'psql_trigger';"
expect guard_refusals_recorded "SELECT /* wiki_hic12_guard_errors */ count(*) = 2 AND bool_and(e.stage = 'execution' AND e.sqlstate IS NULL AND e.message LIKE 'runner did not send%') FROM pg_temp.hic_error AS e JOIN pg_temp.hic_attempt AS a USING (attempt_id) WHERE a.phase = 'index';"
expect guard_sent_no_count "SELECT /* wiki_hic12_guard_counts */ NOT EXISTS (SELECT 1 FROM pg_temp.hic_count AS c JOIN pg_temp.hic_attempt AS a USING (attempt_id) WHERE a.phase = 'index');"
close_case

check_case rls hic_rls false false hic_reader
expect rls_is_error_not_subset "SELECT /* wiki_hic12_rls_status */ status = 'ERROR' AND heap_count IS NULL FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
expect rls_errors_parsed "SELECT /* wiki_hic12_rls_errors */ count(*) = 2 AND bool_and(stage = 'execution' AND sqlstate = '42501' AND message LIKE '%row-level security%' AND hint LIKE 'To disable the policy%') FROM pg_temp.hic_error;"
close_case

for item in denied namespace_denied; do
    check_case "$item" "hic_$item" false false hic_reader
    expect permission_not_silently_skipped "SELECT /* wiki_hic12_permission_error */ status = 'ERROR' AND permission_error IS NOT NULL FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
    expect no_unpermitted_attempt 'SELECT /* wiki_hic12_permission_attempts */ count(*) = 0 FROM pg_temp.hic_attempt;'
    expect denied_object_is_present 'SELECT /* wiki_hic12_permission_scope */ physical_relations = 1 FROM pg_temp.hic_schema_result;'
    close_case
done

check_case empty_scope hic_empty_scope false false
expect no_vacuous_pass 'SELECT /* wiki_hic12_empty_scope */ physical_relations = 0 AND NOT schema_success AND NOT heap_scope_all_pass FROM pg_temp.hic_schema_result;'
close_case

check_case ordinary_role hic_positive false false hic_reader
expect non_superuser_can_check 'SELECT /* wiki_hic12_ordinary_success */ schema_success AND physical_relations = 13 FROM pg_temp.hic_schema_result;'
expect not_superuser_or_bypassrls 'SELECT /* wiki_hic12_ordinary_identity */ NOT rolsuper AND NOT rolbypassrls FROM pg_catalog.pg_roles WHERE rolname = current_user;'
close_case

check_case rls_bypass hic_rls false false
expect privileged_role_sees_all_rows "SELECT /* wiki_hic12_rls_bypass */ status = 'PASS' AND heap_count = 2 FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
close_case

open_case own_temp_scope
hic_install "$HIC_SQL" || fail 'installation'
hic_sql main 'CREATE /* wiki_hic12_temp_fixture */ TEMP TABLE hic_user_data(k integer) USING heap; INSERT /* wiki_hic12_temp_rows */ INTO pg_temp.hic_user_data VALUES (1), (NULL); CREATE /* wiki_hic12_temp_index */ INDEX hic_user_data_idx ON pg_temp.hic_user_data(k);' || fail 'temporary fixture'
hic_sql main 'SELECT /* wiki_hic12_temp_schema */ nspname FROM pg_catalog.pg_namespace WHERE oid = pg_my_temp_schema();' || fail 'temporary schema'
hic_configure "$HIC_VALUE" false false || fail 'configuration'
hic_initialize || fail "initialization: $HIC_STOP"
run_loop
finish_case
expect only_application_temp_relation 'SELECT /* wiki_hic12_temp_scope */ schema_success AND physical_relations = 1 FROM pg_temp.hic_schema_result;'
expect prefix_does_not_hide_user_table "SELECT /* wiki_hic12_temp_name */ status = 'PASS' AND heap_count = 2 FROM pg_temp.hic_relation_result WHERE relation_name = 'hic_user_data';"
close_case

open_case bootstrap
hic_install "$HIC_SQL" || fail 'installation'
expect_error missing_ddl_attestation 23514 "INSERT /* wiki_hic12_missing_ddl_agreement */ INTO pg_temp.hic_config(target_schema, ddl_stable) VALUES ('hic_positive', false);"
expect no_false_configuration 'SELECT /* wiki_hic12_no_configuration */ count(*) = 0 FROM pg_temp.hic_config;'
hic_configure hic_missing_schema false false || fail 'configuration'
hic_initialize
rc=$?
[ "$rc" -eq 1 ] || fail "initialization status $rc, expected 1"
case $HIC_STOP in initialization*) ok missing_schema_stops_at_initialization ;; *) fail "stop reason $HIC_STOP" ;; esac
expect initialization_error_recorded "SELECT /* wiki_hic12_initialization_error */ count(*) = 1 AND bool_and(stage = 'initialization' AND sqlstate = '23514' AND requested_schema = 'hic_missing_schema') FROM pg_temp.hic_run_error;"
expect global_error_state "SELECT /* wiki_hic12_global_error */ state = 'ERROR' FROM pg_temp.hic_progress;"
expect no_partial_success 'SELECT /* wiki_hic12_no_partial_success */ NOT EXISTS (SELECT 1 FROM pg_temp.hic_schema_result) AND NOT EXISTS (SELECT 1 FROM pg_temp.hic_next);'
close_case

open_case concurrent
start_run hic_concurrent false false
hic_attempt_once || fail "first attempt: $HIC_STOP"
expect only_heap_before_writer "SELECT /* wiki_hic12_before_writer */ count(*) = 1 AND bool_and(phase = 'heap' AND accepted AND row_count = 100) FROM pg_temp.hic_attempt_result;"
"$bin/psql" -X -q -v ON_ERROR_STOP=1 -f "$work/writer.sql" > "$out/writer.log" 2>&1 || fail "writer: $(cat "$out/writer.log")"
ok writer_committed_changes
run_loop
finish_case
expect snapshot_survived_concurrent_changes 'SELECT /* wiki_hic12_snapshot_stable */ schema_success FROM pg_temp.hic_schema_result;'
expect same_100_rows 'SELECT /* wiki_hic12_old_rows_visible */ count(*) = 2 AND bool_and(accepted AND row_count = 100) FROM pg_temp.hic_attempt_result;'
close_case
cat "$out/writer.log"

open_case verdict_units
start_run hic_concurrent false false
run_loop
expect eligibility_units "SELECT /* wiki_hic12_eligibility_units */ count(*) = 9 AND bool_and(pg_temp.hic_index_eligible(kind, am_oid, am_name, am_type, partial, valid, ready, live) = expected) FROM (VALUES ('i', 403::oid, 'btree', 'i', false, true, true, true, true), ('I', 403::oid, 'btree', 'i', false, true, true, true, false), ('i', 403::oid, 'btree', 'i', true, true, true, true, false), ('i', 403::oid, 'btree', 'i', false, false, true, true, false), ('i', 403::oid, 'btree', 'i', false, true, false, true, false), ('i', 403::oid, 'btree', 'i', false, true, true, false, false), ('i', 403::oid, 'btree', 'i', false, true, NULL::boolean, true, false), ('i', 0::oid, 'hash', 'i', false, true, true, true, false), ('i', 403::oid, 'btree', 't', false, true, true, true, false)) AS fixture(kind, am_oid, am_name, am_type, partial, valid, ready, live, expected);"
synthetic synthetic_complete_mismatch "UPDATE /* wiki_hic12_synthetic_count */ pg_temp.hic_count AS c SET row_count = row_count + 1 FROM pg_temp.hic_attempt AS a WHERE a.attempt_id = c.attempt_id AND a.phase = 'index'; INSERT /* wiki_hic12_synthetic_covered_attempt */ INTO pg_temp.hic_attempt(attempt_id, relation_oid, phase, intended_index_oid, variant, statement_text) SELECT 98, relation_oid, 'index', index_oid, 2, 'synthetic covered-candidate failure' FROM pg_temp.hic_index WHERE eligible; INSERT /* wiki_hic12_synthetic_covered_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, snapshot_survived) VALUES (98, 'capture', NULL, 'synthetic failure with independent accepted coverage', true);" \
    "SELECT /* wiki_hic12_mismatch_verdict */ status = 'MISMATCH' AND checked_indexes = eligible_indexes AND error_attempts = 1 AND uncovered_error_attempts = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
synthetic one_mismatch_with_untested_candidate_is_inconclusive "UPDATE /* wiki_hic12_synthetic_unmatched_count */ pg_temp.hic_count AS c SET row_count = row_count + 1 FROM pg_temp.hic_attempt AS a WHERE a.attempt_id = c.attempt_id AND a.phase = 'index'; INSERT /* wiki_hic12_synthetic_untested_candidate */ INTO pg_temp.hic_index SELECT (jsonb_populate_record(NULL::pg_temp.hic_index, to_jsonb(idx) || jsonb_build_object('index_oid', 4000000002::bigint, 'index_name', 'synthetic_untested_idx', 'qualified_name', 'hic_concurrent.synthetic_untested_idx'))).* FROM pg_temp.hic_index AS idx WHERE eligible;" \
    "SELECT /* wiki_hic12_partial_coverage_verdict */ status = 'INCONCLUSIVE' AND checked_indexes = 1 AND untested_indexes = 1 AND mismatching_attempts = 1 FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
synthetic missing_plan_is_not_pass "DELETE /* wiki_hic12_synthetic_remove_plan */ FROM pg_temp.hic_plan AS p USING pg_temp.hic_attempt AS a WHERE a.attempt_id = p.attempt_id AND a.phase = 'index';" \
    "SELECT /* wiki_hic12_missing_plan_verdict */ status = 'INCONCLUSIVE' AND EXISTS (SELECT 1 FROM pg_temp.hic_pending) FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
synthetic profile_mismatch_is_not_pass "UPDATE /* wiki_hic12_synthetic_wrong_profile */ pg_temp.hic_count AS c SET settings = jsonb_set(settings, '{row_security}', '\"on\"'::jsonb) FROM pg_temp.hic_attempt AS a WHERE a.attempt_id = c.attempt_id AND a.phase = 'index';" \
    "SELECT /* wiki_hic12_profile_verdict */ status = 'INCONCLUSIVE' FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
synthetic mixed_snapshots_invalidate_run "UPDATE /* wiki_hic12_synthetic_wrong_snapshot */ pg_temp.hic_count SET snapshot_text = 'wrong snapshot' WHERE attempt_id = 1;" \
    "SELECT /* wiki_hic12_snapshot_verdict */ status = 'ERROR' FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
synthetic wrong_attempt_marker_is_not_pass "UPDATE /* wiki_hic12_synthetic_wrong_marker */ pg_temp.hic_plan AS p SET document = jsonb_set(p.document, marker.path || ARRAY['Output', '0'], to_jsonb('999::bigint'::text)) FROM (SELECT plan.attempt_id, node.path FROM pg_temp.hic_plan AS plan JOIN pg_temp.hic_attempt AS a USING (attempt_id) CROSS JOIN LATERAL pg_temp.hic_plan_nodes(plan.document) AS node WHERE a.phase = 'index' AND node.node ->> 'Node Type' = 'Aggregate') AS marker WHERE marker.attempt_id = p.attempt_id;" \
    "SELECT /* wiki_hic12_marker_verdict */ status = 'INCONCLUSIVE' FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
synthetic existential_pass_retains_optional_error "INSERT /* wiki_hic12_synthetic_attempt */ INTO pg_temp.hic_attempt(attempt_id, relation_oid, phase, intended_index_oid, variant, statement_text) SELECT 99, relation_oid, 'index', index_oid, 2, 'synthetic verdict fixture only' FROM pg_temp.hic_index WHERE eligible; INSERT /* wiki_hic12_synthetic_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, snapshot_survived) VALUES (99, 'execution', 'XX000', 'synthetic optional failure, not physical index corruption', true);" \
    "SELECT /* wiki_hic12_optional_error_verdict */ status = 'PASS' AND error_attempts = 1 FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
synthetic unsupported_am_prevents_blanket_success "INSERT /* wiki_hic12_synthetic_unsupported_object */ INTO pg_temp.hic_object SELECT (jsonb_populate_record(NULL::pg_temp.hic_object, to_jsonb(obj) || jsonb_build_object('relation_oid', 4000000000::bigint, 'relation_name', 'synthetic_unsupported', 'qualified_name', 'hic_concurrent.synthetic_unsupported', 'table_am_oid', 4000000001::bigint, 'table_am', 'synthetic', 'in_heap_scope', false, 'exclusion_reason', 'unsupported_table_access_method', 'unsupported_table_am', true))).* FROM pg_temp.hic_object AS obj WHERE relation_name = 't';" \
    'SELECT /* wiki_hic12_unsupported_verdict */ heap_scope_all_pass AND NOT schema_success AND unsupported_table_ams = 1 FROM pg_temp.hic_schema_result;'
expect equal_counts_can_hide_different_bags "WITH /* wiki_hic12_bag_cancellation */ heap_bag(tid) AS (VALUES ('(0,1)'::tid), ('(0,2)'::tid)), index_bag(tid) AS (VALUES ('(0,1)'::tid), ('(0,1)'::tid)), missing AS (SELECT tid FROM heap_bag EXCEPT ALL SELECT tid FROM index_bag), extra AS (SELECT tid FROM index_bag EXCEPT ALL SELECT tid FROM heap_bag) SELECT (SELECT count(*) FROM heap_bag) = (SELECT count(*) FROM index_bag) AND (SELECT count(*) FROM missing) = 1 AND (SELECT count(*) FROM extra) = 1;"
finish_case
close_case

open_case capture_retry
start_run hic_concurrent false false
hic_attempt_fetch || fail "fetch: $HIC_STOP"
hic_attempt_execute || fail 'first execution failed'
hic_post_count || fail 'controls'
hic_recover capture '' 'test deliberately discards the first executed plan' '' '' || fail "recovery: $HIC_STOP"
hic_post_count || fail 'controls'
run_loop
finish_case
expect capture_retry_keeps_same_snapshot "SELECT /* wiki_hic12_retry_status */ status = 'PASS' AND error_attempts = 1 AND uncovered_error_attempts = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
expect one_failed_and_two_verified_attempts 'SELECT /* wiki_hic12_retry_attempts */ count(*) = 3 AND count(*) FILTER (WHERE accepted) = 2 AND count(*) FILTER (WHERE error_present) = 1 FROM pg_temp.hic_attempt_result;'
close_case

open_case malformed_json
start_run hic_concurrent false false
hic_attempt_fetch || fail "fetch: $HIC_STOP"
hic_attempt_execute || fail 'first execution failed'
hic_post_count || fail 'controls'
plan="$HIC_OUT/plans/attempt-$HIC_ATTEMPT_ID.json"
sed -n '1,3p' "$plan" > "$plan.cut" && mv "$plan.cut" "$plan" || fail 'could not truncate the plan'
hic_attempt_import
rc=$?
[ "$rc" -eq 4 ] && [ "$HIC_ERR_CODE" = 22P02 ] && [ -n "$HIC_ERR_DETAIL" ] || fail "import status $rc, SQLSTATE ${HIC_ERR_CODE:-none}"
ok truncated_plan_rejected_by_jsonb
hic_recover capture "$HIC_ERR_CODE" "$HIC_ERR_MESSAGE" "$HIC_ERR_DETAIL" "$HIC_ERR_HINT" || fail "recovery: $HIC_STOP"
hic_post_count || fail 'controls'
run_loop
finish_case
expect malformed_plan_recorded "SELECT /* wiki_hic12_malformed_error */ count(*) = 1 AND bool_and(stage = 'capture' AND sqlstate = '22P02' AND detail IS NOT NULL) FROM pg_temp.hic_error;"
expect malformed_plan_retry_passes "SELECT /* wiki_hic12_malformed_status */ status = 'PASS' AND error_attempts = 1 AND uncovered_error_attempts = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
close_case

open_case lock_timeout
start_run hic_lock false false
hic_open aux 5 6 || fail 'lock holder did not start'
hic_sql aux 'BEGIN /* wiki_hic12_lock_holder */; LOCK /* wiki_hic12_lock_table */ TABLE hic_lock.t IN ACCESS EXCLUSIVE MODE;' || fail 'lock holder'
run_loop
hic_sql aux 'ROLLBACK /* wiki_hic12_lock_release */;' || fail 'lock release'
hic_close aux || fail 'lock holder exit status'
finish_case
expect lock_timeout_is_error "SELECT /* wiki_hic12_lock_status */ status = 'ERROR' AND heap_count IS NULL FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
expect lock_timeout_recorded "SELECT /* wiki_hic12_lock_errors */ count(*) = 2 AND bool_and(stage = 'execution' AND sqlstate = '55P03') FROM pg_temp.hic_error;"
close_case

open_case session_lost
start_run hic_positive false false
hic_attempt_once || fail "first attempt: $HIC_STOP"
hic_sql main 'SELECT /* wiki_hic12_session_pid */ pg_backend_pid();' || fail 'backend pid'
victim=$HIC_VALUE
hic_open aux 5 6 || fail 'terminator did not start'
hic_sql aux "SELECT /* wiki_hic12_terminate */ pg_terminate_backend($victim);" && [ "$HIC_VALUE" = t ] || fail 'termination request'
tries=0
while hic_sql aux "SELECT /* wiki_hic12_wait_exit */ count(*) FROM pg_catalog.pg_stat_activity WHERE pid = $victim;" && [ "$HIC_VALUE" != 0 ]; do
    [ "$tries" -lt 30 ] || fail 'terminated backend did not exit'
    sleep 1
    tries=$((tries + 1))
done
hic_close aux || fail 'terminator exit status'
hic_attempt_once
rc=$?
[ "$rc" -eq 2 ] || fail "attempt status $rc after termination, expected 2"
ok lost_session_stops_the_run
hic_close main
rc=$?
[ "$rc" -eq 2 ] || fail "psql exit status $rc, expected 2"
ok psql_reports_lost_connection
grep -F -q 'FATAL:  57P01:' "$HIC_OUT/session-main/stderr.log" || fail 'termination message missing'
ok termination_reported_with_sqlstate

case_name=keeper
HIC_OUT="$out/keeper"
mkdir -m 700 "$HIC_OUT" || fail 'output directory'
hic_open aux 5 6 || fail 'horizon holder did not start'
hic_sql aux 'BEGIN /* wiki_hic12_keeper_begin */ ISOLATION LEVEL REPEATABLE READ READ ONLY; SELECT /* wiki_hic12_keeper_xid */ txid_current();' || fail 'horizon holder'
"$bin/psql" -X -q -v ON_ERROR_STOP=1 -f "$work/gate-setup.sql" > "$out/gate-setup.log" 2>&1 || fail "gate setup: $(cat "$out/gate-setup.log")"
ok late_index_has_indcheckxmin
check_case old_horizon hic_gate false false
expect old_horizon_remains_inconclusive "SELECT /* wiki_hic12_old_horizon_status */ status = 'INCONCLUSIVE' AND heap_count = 120 AND checked_indexes = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
expect old_horizon_not_falsely_excluded "SELECT /* wiki_hic12_old_horizon_candidate */ eligible AND indcheckxmin AND NOT actually_checked AND snapshot_usability = 'indcheckxmin_unresolved' FROM pg_temp.hic_candidate_result WHERE index_name = 'gate_idx';"
close_case
hic_sql aux 'COMMIT /* wiki_hic12_keeper_finish */;' || fail 'horizon release'
hic_close aux || fail 'horizon holder exit status'
check_case fresh_horizon hic_gate false false
expect fresh_horizon_passes "SELECT /* wiki_hic12_new_horizon_status */ status = 'PASS' AND heap_count = 120 FROM pg_temp.hic_relation_result WHERE relation_name = 't';"
expect true_flag_can_be_usable "SELECT /* wiki_hic12_new_horizon_candidate */ indcheckxmin AND actually_checked AND snapshot_usability = 'accepted_by_executed_plan' FROM pg_temp.hic_candidate_result WHERE index_name = 'gate_idx';"
close_case
cat "$out/gate-setup.log"

case_name=summary
printf 'assertions_passed=%s\n' "$passed"
printf '%s\n' "$db" > "$work/last-test-database"
printf '%s\n' "$out" > "$work/last-test-output"
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

- Evidence target: PostgreSQL 12.2 at `45b88269a353ad93744772791feb6d01bc7e1e42`; official documentation from the same source checkout.
- Catalog and object context: `pg_namespace`, `pg_class`, `pg_index`, `pg_am`, `pg_inherits`, per-key metadata, materialized-view rewriting and population checks.
- Planner context: `get_relation_info()`, relcache index lists and xmin refresh, `build_index_paths()`, index-only eligibility, `build_index_pathkeys()`, deparsing, ordering-operator lookup, subquery pull-up and scan costs.
- Executor context: EXPLAIN utility execution/destinations/instrumentation, ordinary and index-only scans, table-AM heap fetching, HOT root/redirect traversal and SQL `ctid`.
- Snapshot and operational context: repeatable-read retention, read-only temporary DML, RLS checks, ordinary and materialized-view locks, VACUUM's cleanup horizon, timeouts and snapshot-age failures.
- Extension/build boundary: all main operations use existing SQL catalogs and executor facilities. Catalog `.h`/`.dat` definitions feed generated bootstrap data and catalog headers; the checker introduces no C module or generated backend header. The runner and the reproduction harness are Bash and SQL only; the harness's earlier C port probe is replaced by a Bash `/dev/tcp` check. pageinspect is a separately built contrib module with page/B-tree/BRIN/GIN/hash regression targets and was not used for the counts. [catalog/Makefile#catalog-inputs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L55-L69) [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L101) [pageinspect/Makefile#module-and-tests](../../../../raw/postgres-12/contrib/pageinspect/Makefile#L1-L15)
- Tests inspected: core expression/pattern-index, RLS, materialized-view and inheritance cases cited in the test section; adjacent index-only isolation coverage is not an end-to-end test of this design.
- Glossary reviewed: heap/access method, B-tree, MVCC/snapshot/isolation, sequential/index/index-only scans, RLS, expression/partial indexes, collation/opclass, HOT, TID, pageinspect, PL/pgSQL, indcheckxmin and TransactionXmin. Added Materialized view and PostgreSQL 12 qualifications for the two horizon terms.
- Comparison pages: NULL values in indexes; CREATE INDEX CONCURRENTLY; invalid-index outcomes; physical index statistics; partitioned indexes; EXPLAIN ANALYZE BUFFERS; relation-tail checking; post-schema-drop health. Navigation and dated records were used for context, not as PostgreSQL evidence.
- PostgreSQL 12 has no common-concept pages. This answer supplies the mechanism context needed here and does not create a concept page.
- Implementation-specific context: client result handling in `PQexecFinish()`, SQL-quoted EXPLAIN index names, custom type-output calls during deparsing, row estimates for SQL set-returning helpers, initialization/attempt error channels and historical reports after COMMIT.
- Review re-check: the ORDER BY parse path for deparsed keys (`pg_get_indexdef()` parenthesization and constant printing, the grammar's parenthesis handling, SQL92 `ORDER BY` targets, same-type coercion), constant pathkey redundancy, `check_xact_readonly()`, partial-path creation, `cost_sort()`, function-name qualification and the timeout error sites.
- Runtime verification: the complete published SQL and extracted test harness were exercised on an isolated exact-pin build, including fresh and build-retained workflows. Only the owned sandbox was used; no application database, network fetch, subagent, commit or push was used.
- Follow-up runner context: psql's non-interactive stdin mode and client-encoding choice, its flush points, statement splitting and quiet output, its error printing and the libpq verbose message format, the `ERROR` and `SQLSTATE` variables, `\echo` and variable interpolation in SQL text and meta-command arguments, dollar quoting in both lexers, the typecast token, slice deparsing, the aborted-transaction statement filter, savepoint rollback and release states, `COPY TO` in read-only transactions and psql's COPY output, end-of-input and lost-connection exits, backend shutdown on client disconnect, `pg_terminate_backend()`, `SET ROLE` with a string constant, libpq environment variables and the JSON parser's end-of-input error. A first development run of the driver composed the runner's steps without creating the plan directory; every attempt then failed closed as a `capture` error with no PASS, and the runner now creates that directory where it saves each plan.
- Follow-up glossary review: added psql, libpq and Dollar quoting, checked on PostgreSQL 12; re-read SQLSTATE, Subtransaction, EXPLAIN, Utility command, Prepared statement, SPI and PL/pgSQL, whose PostgreSQL 12 statements agree with the runner sections.

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
| Preserve the utility result and resolve its actual index name | [fe-exec.c#last-result](../../../../raw/postgres-12/src/interfaces/libpq/fe-exec.c#L2096-L2141), [explain.c#quoted-index](../../../../raw/postgres-12/src/backend/commands/explain.c#L2844-L2862) | The published suite checks quoted identifiers and post-COMMIT reports |
| A bare numeric-literal key needs a same-type `CAST` in `ORDER BY` | [gram.y](../../../../raw/postgres-12/src/backend/parser/gram.y#L13464-L13497), [analyze.c](../../../../raw/postgres-12/src/backend/parser/analyze.c#L1252-L1256), [parse_clause.c](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L1931-L1966), [ruleutils.c](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9554-L9585), [parse_coerce.c](../../../../raw/postgres-12/src/backend/parser/parse_coerce.c#L156-L169) | Parser probe and literal-key fixture in the published measurement script |
| Inspection itself can invoke a type-output function | [ruleutils.c#get_const_expr](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9522-L9550) | User-function/domain fixtures retain untested candidates rather than silently excluding them |
| Nonconcurrent build's broken-HOT-chain usability gate | [index.c#index_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L2921-L2971), [plancat.c#horizon](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234) | Old-holder/new-transaction fixture in the published measurement script |
| SQL orchestration limitation | [xml.c](../../../../raw/postgres-12/src/backend/utils/adt/xml.c#L2671-L2689), [utility.c](../../../../raw/postgres-12/src/backend/tcop/utility.c#L99-L118) | [execute.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50), [psql-ref.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L2059-L2094) |
| Heap visibility and HOT resolution | [indexam.c](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635), [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1559-L1637) | [README.HOT](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L48-L122) |
| Physical inspection is not visible-row counting | [rawpage.c](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L159-L169), [heapfuncs.c](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L180-L224) | [pageinspect.sgml](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L169-L175) |
| Runner request/reply protocol over psql's standard input | [mainloop.c#flush-and-send](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L147-L440), [command.c#echo-and-flush](../../../../raw/postgres-12/src/bin/psql/command.c#L1116-L1152), [common.c#result-variables](../../../../raw/postgres-12/src/bin/psql/common.c#L544-L584) | [psql-ref.sgml#echo](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L1860-L1873); 124 measured assertions |
| Runner error classification and parsing | [fe-protocol3.c#verbose-message](../../../../raw/postgres-12/src/interfaces/libpq/fe-protocol3.c#L1017-L1095), [logging.c#locus-prefix](../../../../raw/postgres-12/src/common/logging.c#L164-L179), [postgres.c#aborted-transaction-filter](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L1091-L1105) | [errcodes.txt#classes](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L77-L99); measured RLS, lock-timeout, truncated-plan and missing-schema cases |
| Dollar-quoted values and the interpolation guard | [psqlscan.l#quotes-and-variables](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L551-L766), [scan.l#dolqdelim](../../../../raw/postgres-12/src/backend/parser/scan.l#L292-L294), [ruleutils.c#printSubscripts](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L10614-L10637) | [syntax.sgml#dollar-quoting](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L563-L621), [psql-ref.sgml#interpolation](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L4020-L4061); measured slice probe and guard fixture |
| Recovery only while the attempt savepoint exists | [xact.c#savepoint-states](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4140-L4308) | [rollback_to.sgml#savepoint](../../../../raw/postgres-12/doc/src/sgml/ref/rollback_to.sgml#L29-L55); measured capture retry and truncated-plan retry |
| Lost session and closed input end the run without a commit | [common.c#CheckConnection](../../../../raw/postgres-12/src/bin/psql/common.c#L386-L397), [postgres.c#client-EOF](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4433-L4455), [postinit.c#ShutdownPostgres](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L1189-L1193) | [psql-ref.sgml#exit-status](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L635-L643); measured `pg_terminate_backend()` case |
| Duplicate-sensitive visible-identity comparison | [heaptuple.c](../../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L636-L640), [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L148-L167) | [select.sgml#EXCEPT-ALL](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L1281-L1294), [ddl.sgml#ctid](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L1198-L1205) |

## Open Questions

- **Compatibility beyond the exact pin is unverified.** The installed version guard is intentionally 12.2-only. Other minor releases, patched server binaries, EXPLAIN/planner hooks, custom types/opclasses and different locale combinations need their own review and tests.
- **Pinned documentation wording:** `config.sgml` says `enable_indexscan` “enables or disables” index-scan plan types, while `cost_index()` adds a penalty when it is off. The implementation settles the checker rule: a setting cannot certify the chosen path. The page uses that source behavior, and this wording discrepancy remains here rather than being treated as a hard planner prohibition. [config.sgml#indexscan-wording](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4449-L4462) [costsize.c#indexscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L525-L535)
- **Complete arbitrary-candidate coverage is not established.** Constant/redundant orderings, competing equivalent indexes, snapshot usability, custom expressions and operator resolution can leave eligible indexes untested. No SQL-only per-index forcing mechanism is proposed. This is a documented feasibility limit, not a promised future outcome.
- **No physical fault-injection coverage yet.** Evaluator tests using synthetic records cannot establish detection rates for damaged index files. A real corruption suite needs separate authorization and a capability beyond the checker defined here.
- **Operational/test gaps remain explicit.** The unexercised cases are listed under measured coverage. The SQL does not prove that the DDL agreement was honored or that damaged catalogs retained a complete inventory; `inventory_complete` means the inventory statements completed. The optional visible-identity capture pipeline and large-schema/performance validation remain future work.
- **Runner portability is measured on one client stack.** The runner avoids Bash 4 features and uses POSIX utilities, but it ran only under GNU Bash 5.2.21 with the psql built from the pin. Another psql release could print errors or flush output differently; this page has no evidence for one, so use the pinned psql or re-run the harness with the intended client.
- **The interpolation guard is conservative, and its refusals are not coverage.** A colon followed by a quote or brace inside a string constant of a key expression is refused as well, and a refused candidate leaves its relation `ERROR` under the existing status rules. Such a candidate can still be checked through the operator workflow with a client that sends the text unchanged.
- **Concept-layer gap:** PostgreSQL 12 has no common-concept page for MVCC snapshots with heap/HOT index fetching or for executed-plan validation. Such pages could be separate follow-up tasks; none is created by this task.

## Source References

- [configure.in#version](../../../../raw/postgres-12/configure.in#L19-L34)
- [pg_class.h#relation-fields](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L30-L63)
- [pg_index.h#index-fields](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L76)
- [pg_am.dat#access-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L35)
- [pg_am.h#method-types](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L53-L56)
- [pg_proc.dat#jsonb-array-elements](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L9100-L9104)
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
- [spi.c#read-only-command](../../../../raw/postgres-12/src/backend/executor/spi.c#L2258-L2263)
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
- [functioncmds.c#function-row-estimate](../../../../raw/postgres-12/src/backend/commands/functioncmds.c#L1134-L1139)
- [create_function.sgml#ROWS](../../../../raw/postgres-12/doc/src/sgml/ref/create_function.sgml#L468-L477)
- [rel.h#heap-fillfactor](../../../../raw/postgres-12/src/include/utils/rel.h#L278-L279)
- [hio.c#fillfactor-reserve](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L348-L350)
- [heap.c#row-security-state](../../../../raw/postgres-12/src/backend/catalog/heap.c#L916-L919)
- [pg_config_manual.h#socket-directory](../../../../raw/postgres-12/src/include/pg_config_manual.h#L178-L182)
- [postmaster.c#listening-endpoints](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L1011-L1043)
- [pqcomm.c#socket-permissions](../../../../raw/postgres-12/src/backend/libpq/pqcomm.c#L692-L699)
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
- [psqlscanslash.l#argument-variables](../../../../raw/postgres-12/src/bin/psql/psqlscanslash.l#L374-L389)
- [copy.c#handleCopyOut](../../../../raw/postgres-12/src/bin/psql/copy.c#L437-L467)
- [psqlscan.l#variable-substitution](../../../../raw/postgres-12/src/fe_utils/psqlscan.l#L708-L762)
- [logging.c#locus-prefix](../../../../raw/postgres-12/src/common/logging.c#L164-L179)
- [fe-protocol3.c#verbose-message](../../../../raw/postgres-12/src/interfaces/libpq/fe-protocol3.c#L1017-L1046)
- [errcodes.txt#classes-00-02](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L77-L99)
- [syntax.sgml#dollar-quoting](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L563-L588)
- [scan.l#dolqdelim](../../../../raw/postgres-12/src/backend/parser/scan.l#L292-L294)
- [xact.c#RollbackToSavepoint](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L4249-L4308)
- [copy.c#read-only-COPY-FROM](../../../../raw/postgres-12/src/backend/commands/copy.c#L1060-L1066)
- [signalfuncs.c#pg_terminate_backend](../../../../raw/postgres-12/src/backend/storage/ipc/signalfuncs.c#L128-L137)
- [libpq.sgml#environment](../../../../raw/postgres-12/doc/src/sgml/libpq.sgml#L7296-L7319)
- [json.c#report_parse_error](../../../../raw/postgres-12/src/backend/utils/adt/json.c#L1126-L1137)
- [jsonb.c#jsonb_from_cstring](../../../../raw/postgres-12/src/backend/utils/adt/jsonb.c#L238-L257)

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Versions and source pins](../../../versions.md)
- [How NULL Values Are Handled in PostgreSQL 12 Indexes (unverified)](null-values-in-indexes.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](create-index-concurrently.md)
- [Physical Index Statistics, Tuple Counts, and Bytes per Tuple in PostgreSQL 12 (unverified)](physical-index-statistics-tuple-counts-and-bytes.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](../observability/explain-analyze-buffers-output.md)
