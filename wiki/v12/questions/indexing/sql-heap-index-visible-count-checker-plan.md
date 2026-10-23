---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: gpt-6-astra-max-thinking 2026-10-23T17:40:53Z
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

## Answer

### Feasibility and execution contract

**A verified positive result is feasible with SQL and ordinary [executor](../../../glossary.md#executor) scans. An unattended, universally complete checker is not guaranteed under literal SQL alone.** The deliverable is a SQL generator plus a staged SQL execution and validation protocol. It uses checker-owned temporary tables to retain exact counts, and the operator retains and imports the corresponding executed plans. Choosing a particular index remains best effort; accepting a count requires evidence from that execution. The reasons are PostgreSQL's [utility-command](../../../glossary.md#utility-command) interface for [EXPLAIN](../../../glossary.md#explain), its prepared-statement meaning of SQL `EXECUTE`, and its cost-based choice of index paths. [explain.sgml#execution-and-results](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L78-L105) [execute.sgml#prepared-statement](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50) [indxpath.c#build_index_paths](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050)

The user approved implementation after the planning review. This page now includes the complete operator-assisted SQL implementation and a separate reproduction harness. The original planning-only Question remains above as the request that motivated the design. The evidence target is **PostgreSQL 12.2**, commit `45b88269a353ad93744772791feb6d01bc7e1e42`. The checkout identifies itself as 12.2. “Official documentation” below means the official PostgreSQL SGML documentation shipped in that same checkout, checked alongside implementation source. Later 12.x minor releases require a compatibility check before claiming validation on them. The SQL version guard checks `server_version_num`, not the provenance of an arbitrary server binary; the measurement harness builds the exact pin. [configure.in#version](../../../../raw/postgres-12/configure.in#L19-L34)

The implementation runs on a primary and requires no extension. Its only DML writes are to its own temporary result tables. Create those tables before entering the checking transaction: a `READ ONLY` transaction permits DML on temporary tables but rejects `CREATE`, including temporary-table creation. Read/write is the default; the explicit transaction option activates that guard. A standby cannot create the temporary tables this design needs. [set_transaction.sgml#read-only](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L126-L139) [execMain.c#ExecCheckXactReadOnly](../../../../raw/postgres-12/src/backend/executor/execMain.c#L759-L795) [namespace.c#temporary-recovery-refusal](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3915-L3927)

| Capability | Assessment and handling |
|---|---|
| Catalog inventory, SQL text generation, exact counting, JSON validation and report aggregation | Feasible with core SQL. Keep dynamic object names in generated statements, using catalog-derived identifiers and quoting. The catalog and deparser interfaces are detailed below. |
| Bind an exact count to its executed access path | Feasible with `EXPLAIN ANALYZE INSERT ... SELECT count(*) ...` into a pre-existing temporary table. The same execution inserts the count and produces the plan; see [capture protocol](#capturing-an-exact-count-and-its-executed-plan). |
| Execute arbitrary SQL text returned by an inventory query | A generated string is not executed by ordinary SQL `EXECUTE`; that command executes a named prepared statement. Have the operator execute the generated SQL in the same session. [execute.sgml#prepared-statement](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50) |
| Put an `EXPLAIN` utility result directly in a SQL CTE or `INSERT ... SELECT` | No ordinary relational `EXPLAIN` interface is provided by this command. Its result goes to the client. Import that result explicitly before running the SQL validator. [explain.sgml#command-and-output](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L32-L46) [explain.sgml#outputs](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L268-L275) |
| Use `query_to_xml()` as a dynamic-EXPLAIN workaround | Rejected for this pin. It executes through read-only SPI and requires `SPI_OK_SELECT`; SPI's read-only test rejects utility statements. It can execute suitable SELECT text, but does not solve this executed-plan capture problem. [xml.c#query_to_xml_internal](../../../../raw/postgres-12/src/backend/utils/adt/xml.c#L2671-L2689) [spi.c#read-only-check](../../../../raw/postgres-12/src/backend/executor/spi.c#L2258-L2263) [utility.c#CommandIsReadOnly](../../../../raw/postgres-12/src/backend/tcop/utility.c#L99-L118) |
| Choose every eligible [B-tree](../../../glossary.md#b-tree) by name and stop automatically after a match | No such guarantee follows from scan settings or ordering. Literal SQL also has no procedural loop over dynamically generated statements. Use a staged “generate next attempt” protocol; offer a procedural wrapper separately. [indices.sgml#index-versus-sort](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L480-L502) [plpgsql.sgml#dynamic-commands](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1157-L1185) |

“Reliable” means a completed, correctly identified executor observation under the stated snapshot and scope, with a validated plan and faithful capture. It does not mean a proof that a damaged database has retained all historical rows. “Best effort” describes obtaining a suitable index path and achieving candidate coverage; an unsuccessful attempt must never manufacture a reliable result.

### Values and decision flow

The checker delegates [MVCC](../../../glossary.md#mvcc) visibility to PostgreSQL's executor. A [sequential scan](../../../glossary.md#sequential-scan) passes the execution [snapshot](../../../glossary.md#snapshot) into the table access method. An ordinary [index scan](../../../glossary.md#index-scan) passes the same snapshot into [heap](../../../glossary.md#heap) fetching after obtaining index tuple identifiers. [nodeSeqscan.c#SeqNext](../../../../raw/postgres-12/src/backend/executor/nodeSeqscan.c#L50-L82) [nodeIndexscan.c#IndexNext](../../../../raw/postgres-12/src/backend/executor/nodeIndexscan.c#L105-L149) [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L573)

| Value | Meaning and origin | Use |
|---|---|---|
| `T` | Database [OID](../../../glossary.md#oid) plus physical [relation](../../../glossary.md#relation) OID, with recorded schema/name and catalog classification | Stable identity for one target during the coordinated run |
| `S` | Snapshot retained by the checking transaction; record `txid_current_snapshot()::text` | Associates both accepted scan results with the same visibility boundary |
| `H(T,S)` | Stored `bigint` from an accepted sequential-scan count | Baseline |
| `J` | Actual index OID resolved from the executed plan and inventory | Coverage identity; the intended candidate is recorded separately |
| `I(T,J,S)` | Stored `bigint` from an accepted full ordinary Index Scan count | Comparison with `H` |
| `delta` | `I::numeric - H::numeric` | Zero means equal cardinality; negative means fewer returned rows through that index |
| Candidate state | Inventory facts, snapshot-usability evidence, attempted and actually scanned OIDs | Determines whether “none matched” means `MISMATCH`, `INCONCLUSIVE`, or `ERROR` |

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

Resolve the requested schema to a `pg_namespace` OID, then inventory its `pg_class` rows without first filtering by privileges or index existence. Join the table's `relam` to `pg_am`; separately join `pg_index.indrelid = pg_class.oid`, the index's own `pg_class` row, and its index `pg_am` row. Require table `amtype = 't'` and index `amtype = 'i'` in their respective classifications. Keep a relation row even when the left join finds no index. These catalogs distinguish namespace, relation kind, table versus index access method, and index-to-table ownership. [pg_class.h#identity-and-AM](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L30-L54) [pg_class.h#kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) [pg_index.h#catalog](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L57) [pg_am.h#method-types](../../../../raw/postgres-12/src/include/catalog/pg_am.h#L53-L56) [pg_am.dat#built-in-methods](../../../../raw/postgres-12/src/include/catalog/pg_am.dat#L15-L35)

The inventory specification is:

| Catalog condition | Classification and treatment |
|---|---|
| `relkind = 'r'`, built-in heap table AM | In scope: ordinary tables and physical table partitions, including ordinary inheritance parents that store their own rows |
| `relkind = 'm'`, heap AM, `relispopulated` | In scope: count the stored rows of the [materialized view](../../../glossary.md#materialized-view), not the defining query |
| `relkind = 'p'` | `NOT_APPLICABLE` as a scan target; produce a child-summary record |
| `relkind = 'm'`, not populated | `NOT_APPLICABLE: unpopulated_materialized_view`; never invent a zero count |
| `relkind = 'r'` or `'m'`, another table AM | `NOT_APPLICABLE: unsupported_table_am`; report a physical-object coverage gap |
| View `'v'`, foreign table `'f'`, sequence `'S'` | `NOT_APPLICABLE`, each with its own reason; no local heap-count claim |
| Index `'i'` or partitioned index `'I'` | Inventory as index metadata, not as a second table to count |
| TOAST `'t'`, composite type `'c'`, any other kind | Explicitly outside this initial table/materialized-view scope |

The classifications use `relkind`, `relam`, `relispopulated`, and `relispartition`, not file size or the existence of a parent. PostgreSQL excludes partitioned parents from its storage-bearing kinds. Ordinary materialized-view reads skip rule expansion, and execution of an unpopulated materialized view raises an error. [pg_class.h#state](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L104-L119) [pg_class.h#storage-kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L183-L192) [rewriteHandler.c#materialized-view-reads](../../../../raw/postgres-12/src/backend/rewrite/rewriteHandler.c#L1900-L1911) [execUtils.c#unpopulated-refusal](../../../../raw/postgres-12/src/backend/executor/execUtils.c#L694-L714) [rules.sgml#materialized-views](../../../../raw/postgres-12/doc/src/sgml/rules.sgml#L898-L929)

**Default schema boundary: namespace only.** Include physical relations whose own `relnamespace` is the requested schema. Include such a partition even when its parent is elsewhere. Exclude an outside-schema partition from scanning even when its parent is inside the schema, but list it in that parent's summary as `outside_requested_schema`. A parent summary must say whether it covers all descendants or only the selected subset.

Offer an explicit `include_external_partition_descendants` option. When enabled, recursively traverse `pg_inherits` from partitioned parents in the requested schema and include their physical descendants across schemas, deduplicated by relation OID. Record the added schemas and apply their privileges too. Restrict traversal to table relationships: `pg_inherits` also carries partitioned-index links in this implementation. Do not automatically expand ordinary inheritance into this optional partition closure. [pg_inherits.h#parent-links](../../../../raw/postgres-12/src/include/catalog/pg_inherits.h#L30-L37) [index.c#index-inheritance-link](../../../../raw/postgres-12/src/backend/catalog/index.c#L1002-L1010) [ddl.sgml#declarative-storage](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3611-L3626)

Every physical-target scan uses `FROM ONLY schema.relation`. Without `ONLY`, a parent query also scans inherited descendants; that would mix multiple physical objects into its baseline. A partitioned-parent summary aggregates child statuses and coverage, never accepts equality of aggregate parent totals as a replacement for individual child passes. The optional sum of child counts is meaningful as a common-snapshot total only in schema-wide snapshot mode. [select.sgml#ONLY](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L355-L365)

Retain `relpersistence` in the inventory. Permanent and unlogged heap relations are eligible for this executor comparison; their durability is not being assessed. Own-session temporary application tables can be included when they are the explicitly requested scope. Classify another session's temporary relation as `ERROR: unsupported_remote_temp` before attempting a count: the reader cannot see its owner's local buffers. Keep the object in discovery rather than silently dropping it. Always exclude checker-owned scratch OIDs from the target set. [pg_class.h#persistence](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L165-L167) [bufmgr.c#temporary-relation-access](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L640-L658)

### Index eligibility and ordering generation

A supported structural candidate must belong to an in-scope target and satisfy all of:

```text
index pg_class.relkind = 'i'
index AM = the built-in btree AM
pg_index.indpred IS NULL
pg_index.indisvalid AND pg_index.indisready AND pg_index.indislive
```

Keep rejected indexes with reason codes such as `partial`, `invalid`, `not_ready`, `not_live`, `partitioned_index`, and `unsupported_index_am`. A [partial index](../../../glossary.md#partial-index) with a logically trivial predicate still fails this requested `indpred IS NULL` contract. The three state flags have distinct meanings; they are not estimates of physical correctness. The planner rejects invalid and partitioned indexes, while the relcache's usable index list omits indexes marked not live. [catalogs.sgml#index-state](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3822-L3865) [pg_index.h#predicate](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L53-L57) [plancat.c#validity-and-partitioned-indexes](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L199-L234) [relcache.c#non-live-indexes](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4383-L4400)

**Snapshot usability is an additional gate.** [`indcheckxmin`](../../../glossary.md#indcheckxmin) does not mean “always reject this index.” When true, the planner requires the `xmin` of the index's relcache `pg_index` tuple to precede the backend's [TransactionXmin](../../../glossary.md#transactionxmin). Otherwise it skips the index and marks the plan transient. This protects snapshots that could see incompatible [HOT](../../../glossary.md#hot) chain versions. [plancat.c#indcheckxmin](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234) [catalogs.sgml#indcheckxmin](../../../../raw/postgres-12/doc/src/sgml/catalogs.sgml#L3835-L3844)

Record `indcheckxmin`, the observed `pg_index.xmin`, and the snapshot descriptor. Do not reproduce that gate with `xmin::text::bigint < txid_snapshot_xmin(...)`: the planner uses wrap-aware transaction-ID comparison and a backend horizon, and its relcache tuple is not necessarily the catalog row an earlier SQL statement observed. The initial design treats a true flag as **usability unresolved until an actual accepted plan uses that index**, rather than guessing the horizon. A refused candidate remains unresolved/untested unless there is independent conclusive exclusion evidence. It therefore blocks `MISMATCH` and `NO_ELIGIBLE_INDEX`, but does not block a valid pass through another index. [transam.c#TransactionIdPrecedes](../../../../raw/postgres-12/src/backend/access/transam/transam.c#L296-L312) [procarray.c#horizons](../../../../raw/postgres-12/src/backend/storage/ipc/procarray.c#L1489-L1499) [relcache.c#index-xmin-refresh](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L2210-L2224)

Non-partial status is insufficient across [access methods](../../../glossary.md#access-method). B-tree advertises ordinary tuple scans, ordering, optional search keys, and NULL support. Hash omits NULL entries and requires a search key; GIN and BRIN have no ordinary `amgettuple` callback. Other methods and their operator classes need a separate completeness argument. Defer hash, GiST, SP-GiST, GIN, BRIN and extension AMs in the first implementation; do not use bitmap deduplication or a row-restricting predicate to simulate their full scan. [nbtree.c#bthandler](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L109-L142) [hashutil.c#NULL-omission](../../../../raw/postgres-12/src/backend/access/hash/hashutil.c#L312-L337) [hash.c#capabilities](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L60-L74) [ginutil.c#scan-callbacks](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L63-L73) [brin.c#scan-callbacks](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L111-L121)

For each structural B-tree candidate, generate an ordering specification as follows:

1. Walk key positions `1..indnkeyatts`, not all `indnatts`. Included columns do not define index order. The deparser resolves `indkey`; the generator reads `indclass`, `indcollation`, and `indoption` at their corresponding zero-based vector positions. [pg_index.h#key-metadata](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L32-L57) [pathkeys.c#INCLUDE-exclusion](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L487-L492) [ruleutils.c#index-keys](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1304-L1350)
2. Obtain each column or [expression](../../../glossary.md#expression-index) with `pg_get_indexdef(index_oid, position, false)`. On this pin the positive-position form requests only the variable/expression and suppresses index decorations. Do not split `pg_get_expr(indexprs, ...)` at commas or paste a full `CREATE INDEX` key specification into `ORDER BY`. Expressions themselves can contain commas, and an opclass name is not an `ORDER BY` suffix. [ruleutils.c#column-deparse](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1085-L1131) [ruleutils.c#expression-and-decoration](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1331-L1399)
3. Append the exact indexed [collation](../../../glossary.md#collation) when `indcollation` is nonzero. Qualify its namespace and quote identifiers. Generate and execute under the same controlled `search_path` so deparsed expression names resolve consistently. [ruleutils.c#index-collation](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1372-L1381) [pathkeys.c#sort-identity](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L512-L522)
4. Resolve the [operator class and family](../../../glossary.md#operator-class). For ordinary default ordering, `ASC`/`DESC` can suffice. For a nondefault family such as a pattern opclass, derive the same-type B-tree less-than or greater-than operator from `pg_opclass` and `pg_amop`, then use `USING OPERATOR(qualified_operator)`. Use strategy 1 for forward ascending and 5 for forward descending, with `amoppurpose = 's'`; these are B-tree search-family members, not `amoppurpose = 'o'` distance-ordering operators. Validate type compatibility instead of adding arbitrary casts. If overloaded/polymorphic/custom operators cannot be reconstructed confidently, retain the candidate as untested. [pg_amop.h#operator-metadata](../../../../raw/postgres-12/src/include/catalog/pg_amop.h#L54-L80) [parse_clause.c#SORTBY_USING](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L3263-L3295) [lsyscache.c#ordering-operator](../../../../raw/postgres-12/src/backend/utils/cache/lsyscache.c#L216-L242)
5. Decode `INDOPTION_DESC` and `INDOPTION_NULLS_FIRST`; specify NULL positioning explicitly for every key. A backward scan reverses both ordering and NULL positioning for every key. The generator emits complete forward ordering; a complete reverse-order variant remains optional and is not implemented here. The validator accepts either actual scan direction. [pg_index.h#ordering-bits](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L70-L76) [pathkeys.c#scan-direction](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L497-L519) [indices.sgml#ordering](../../../../raw/postgres-12/doc/src/sgml/indices.sgml#L505-L542)

The generator must treat quoted schema, relation, column, collation and function names correctly. Use `format('%I.%I', ...)` for identifiers and literal quoting for values. Operator tokens require their SQL operator syntax, not identifier quoting; use only catalog-derived, validated operator names. Expressions, collation implementations and custom operator functions must also be trusted to execute in the checking role. Refuse or leave untested an expression whose execution is outside the allowed function policy; an `IMMUTABLE` declaration alone is not an audit of arbitrary user code. [func.sgml#format-examples](../../../../raw/postgres-12/doc/src/sgml/func.sgml#L3402-L3414) [parse_clause.c#ordering-resolution](../../../../raw/postgres-12/src/backend/parser/parse_clause.c#L3277-L3295)

The implementation checks dependencies and index-column types **before** calling the deparser. Deparsing a non-NULL constant invokes its type-output function, so “we will not execute the generated query” is not enough to avoid all user code during inspection. The generator conservatively defers non-stock functions/operators and unreviewed types, including domains; it permits enums and reviewed stock opclasses/operators. Its OID cutoff follows `FirstNormalObjectId`; it is a coverage policy for the pinned stock catalogs, not an authentication boundary against modified catalogs, malicious code or EXPLAIN hooks. [ruleutils.c#get_const_expr](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9522-L9550) [transam.h#FirstNormalObjectId](../../../../raw/postgres-12/src/include/access/transam.h#L136-L141) [explain.c#index-name-hook](../../../../raw/postgres-12/src/backend/commands/explain.c#L2844-L2862)

Compatible ordering encourages a path; it does not name an index. With index-only scans disabled, the planner builds an ordinary index path only when restrictions, useful ordering, or another qualifying reason makes it useful. An index containing only a constant expression may lose its useful ordering and get no such path. Identically ordered indexes may repeatedly select the same cheaper index. These are coverage limits, not proof of corruption. [indxpath.c#path-generation-gate](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L991-L1050) [pathkeys.c#redundant-ordering](../../../../raw/postgres-12/src/backend/optimizer/path/pathkeys.c#L524-L546)

### Snapshot permissions and DDL coordination

Use `BEGIN ISOLATION LEVEL REPEATABLE READ READ ONLY` after scratch-table setup. The default isolation level is read committed, which takes a new snapshot for later statements. Repeatable read retains the transaction's first snapshot; the checker must not perform application DML between the two scans. Recording a snapshot descriptor helps audit provenance but does not recreate a completed transaction's snapshot later. [guc.c#isolation-default](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4255-L4272) [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [mvcc.sgml#repeatable-read](../../../../raw/postgres-12/doc/src/sgml/mvcc.sgml#L454-L479)

| Transaction mode | Guarantee | Operational trade-off |
|---|---|---|
| One schema-wide transaction, implemented here | Every accepted relation result shares one snapshot | Holds the snapshot and acquired relation/index locks until the run ends; one aborted transaction can lose all temporary result changes from that run |
| One transaction per physical relation, optional architecture not implemented here | Each `H(T,S_T)` and its index attempts share one `S_T` | Shorter snapshots and easier retries; the report must say there is no single schema-wide snapshot, and must not describe summed counts as one point-in-time schema total |

The run's inventory manifest must also remain complete across per-relation transactions. Require the membership/identity DDL freeze for the whole run, or re-inventory and invalidate a schema report whose target set changed. Shorter per-relation snapshots do not remove that scope-completeness requirement.

A long snapshot can retain old row versions that VACUUM would otherwise remove because its cleanup cutoff depends on active transaction horizons. Ordinary scan locks are compatible with DML, but conflicting DDL can wait behind the checker. [vacuum.c#cleanup-horizon](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L858-L865) [vacuum.c#GetOldestXmin](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L902-L912) [lock.c#lock-conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L103)

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
| `max_parallel_workers_per_gather` | `2` | `0` | `0` | Simplify execution to a serial scan; also reject any parallel plan actually produced |
| `enable_sort` | `on` | Leave on | Start on; optional retry off | A sort penalty may help an ordered index compete; it is not a per-index hint |
| `row_security` | `on` | `off` | `off` | Error if RLS would filter the target |
| `search_path` | `"$user", public` | `pg_catalog, pg_temp` | Same | Qualify all target objects and deparse names in this controlled context |
| `quote_all_identifiers` | `off` | `off` | `off` | Fix deparser/EXPLAIN identifier spelling independently of the caller's settings |
| `lock_timeout` | `0`, disabled | Starting budget `'2s'` | Same | Bound each lock wait; adapt to the operating window |
| `statement_timeout` | `0`, disabled | Starting budget `'15min'` | Same | Bound each statement; timeout is an error, never a zero count |

The nondefault scan profiles change costing, not the acceptance contract. `enable_seqscan = off` and `enable_indexscan = off` add `disable_cost`; disabling index-only scans is checked when constructing an index-only path. The official sequential-scan and sort documentation expressly says these methods cannot be suppressed entirely. [costsize.c#seqscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L225-L232) [costsize.c#indexscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L525-L535) [costsize.c#bitmap-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L963-L970) [indxpath.c#index-only-gate](../../../../raw/postgres-12/src/backend/optimizer/path/indxpath.c#L1898-L1907) [config.sgml#seqscan-and-sort](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4616-L4648)

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

Do not execute `EXPLAIN` and then a separate plain count and claim the first plan proves the second execution. Plain EXPLAIN does not run the plan; a later statement can be replanned. Nor does `EXPLAIN ANALYZE SELECT count(*) ...` return the aggregate value: it discards SELECT output. Its Aggregate node returns one row regardless of that row's count value. Instrumentation stores floating-point row counters and EXPLAIN prints rows per loop, rounded to zero decimal places; those counters should not replace the exact `bigint` aggregate. [explain.c#execution-and-destination](../../../../raw/postgres-12/src/backend/commands/explain.c#L494-L541) [instrument.h#row-counters](../../../../raw/postgres-12/src/include/executor/instrument.h#L44-L65) [explain.c#rows-per-loop](../../../../raw/postgres-12/src/backend/commands/explain.c#L1505-L1534) [explain.sgml#discarded-output](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L87-L105)

Use this protocol for each attempt:

1. Before the checking transaction, create checker-owned temporary capture, inventory and audit tables. Do not attach application-supplied triggers, rules, defaults or generated expressions. This implementation permits only its fixed metadata defaults and its own key/check constraints; the settings default calls the published SQL helper. Use explicit run/attempt keys so a failed attempt cannot accidentally reuse an old result.
2. Start repeatable read, fix scope and snapshot, save the proposed SQL text/settings, and establish an attempt [savepoint](../../../glossary.md#subtransaction).
3. Execute **one explained INSERT** that reads the target and stores its `count(*)` in the temporary capture table.
4. Retain the JSON plan returned to the client with that statement's run/attempt identity. Read the inserted count using the same keys.
5. Import that exact JSON result into the temporary audit table with a literal SQL INSERT, using proper literal escaping, or retain it in the operator's query transcript and mark the audit pending until imported. This is an explicit manual bridge in literal SQL mode.
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

Bare EXPLAIN defaults `ANALYZE`, `VERBOSE`, `BUFFERS` and `SETTINGS` to false. This protocol turns them on to execute the statement, expose schema-qualified names/output expressions, and retain buffer and planner-setting evidence. `TIMING` and `SUMMARY` normally follow `ANALYZE`; the protocol explicitly disables the former and keeps the latter. `SETTINGS` reports only nondefault parameters affecting planning, so it cannot replace the run's separate effective-setting record. [explain.c#option-defaults](../../../../raw/postgres-12/src/backend/commands/explain.c#L154-L221) [explain.c#NewExplainState](../../../../raw/postgres-12/src/backend/commands/explain.c#L285-L295) [explain.sgml#verbose](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L130-L139) [explain.sgml#settings](../../../../raw/postgres-12/doc/src/sgml/ref/explain.sgml#L156-L162)

The literal-SQL bridge assumes faithful capture of unedited server output. A SQL validator can reject mismatched identifiers and bad shapes; it does not cryptographically authenticate a JSON literal supplied by an operator. Record who associated each plan with its attempt, and leave the attempt pending if that association cannot be established.

The generated count also projects its attempt ID as a `bigint` constant beside `count(*)`. The validator checks that marker in the verbose Aggregate output and checks the saved transaction, snapshot, backend, role and scan settings. Those checks catch accidental mismatches; they do not authenticate an old document manually imported into a new run.

The generator deliberately ends `sql_text` with EXPLAIN. `PQexecFinish()` returns the last result of a multi-command submission, so appending settings-reset commands can hide the JSON result from a client using that interface. `post_count_sql` is returned separately for this reason. [fe-exec.c#PQexecFinish](../../../../raw/postgres-12/src/interfaces/libpq/fe-exec.c#L2096-L2141)

On this pin, the JSON `"Index Name"` value already contains SQL identifier quoting: `explain_get_index_name()` calls `quote_identifier()` before `ExplainIndexScanDetails()` emits the property. Resolve it using the same quoting convention, not direct equality to raw `pg_class.relname`. The checker fixes `quote_all_identifiers = off` for captures and in its identity helper, so a caller's restored setting after COMMIT does not change historical matching. The setting's default is off and its context is `user`; it is not a scan-forcing control. [explain.c#index-name](../../../../raw/postgres-12/src/backend/commands/explain.c#L2844-L2862) [explain.c#index-JSON](../../../../raw/postgres-12/src/backend/commands/explain.c#L2991-L3025) [guc.c#quote-all-identifiers](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1816-L1824)

The temporary write is allowed in `READ ONLY`; creating the capture table inside that transaction is not. `EXPLAIN ANALYZE CREATE TEMP TABLE AS ...` would also retain a query result, but conflicts with the recommended read-only transaction mode, so it is not the baseline design. [execMain.c#temporary-DML](../../../../raw/postgres-12/src/backend/executor/execMain.c#L769-L791) [set_transaction.sgml#read-only](../../../../raw/postgres-12/doc/src/sgml/ref/set_transaction.sgml#L126-L139)

Literal SQL cannot automatically catch an execution error, retrieve its diagnostics, branch to `ROLLBACK TO SAVEPOINT`, and resume the dynamic loop. The operator must do that, or stop and preserve the transcript. Keep the core process fail-closed: a transaction abort, disconnect, missing plan or missing result never becomes an empty table or a pass. A generated script must mark these interaction points, not bury them in a `DO` block. [plpgsql.sgml#dynamic-execution](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1157-L1185) [rollback_to.sgml#savepoint](../../../../raw/postgres-12/doc/src/sgml/ref/rollback_to.sgml#L29-L55)

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
| Error | Run/relation/candidate/attempt references as available, stage, SQLSTATE, message/detail/hint, whether the snapshot survived, whether other accepted evidence remains valid |
| Parent/schema summary | Counts by relation status, distinct eligible/checked/matching/untested indexes, outside-schema descendants, unsupported AMs, unresolved errors, scope-completeness and objective-success booleans |

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

For exhaustive mode, retain the requested existential PASS semantics and add `all_tested_indexes_match`, `exhaustive_coverage_complete`, and anomaly counts. A PASS with another index mismatch or attempt error is visibly qualified in the report. An optional stricter operational flag can require zero attempt errors and zero observed mismatches; do not silently redefine the requested objective.

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

Run this complete block once in a dedicated checking session, before starting its read-only checking transaction. It creates only checker-owned temporary objects and `LANGUAGE sql` helpers. It does not execute generated count statements, import client output, or catch exceptions automatically. The operator workflow below supplies those steps.

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
       CASE WHEN obj.in_heap_scope AND ic.relam = 403 AND u.index_oid IS NULL
            THEN pg_catalog.pg_get_indexdef(i.indexrelid, k.n, false) END AS expression_sql,
       a.atttypid AS key_type_oid, opc.oid AS opclass_oid,
       opn.nspname::text AS opclass_schema, opc.opcname::text AS opclass_name,
       opc.opcfamily AS opfamily_oid, i.indcollation[k.n - 1] AS collation_oid,
       cn.nspname::text AS collation_schema, coll.collname::text AS collation_name,
       (i.indoption[k.n - 1] & 1) <> 0 AS descending,
       (i.indoption[k.n - 1] & 2) <> 0 AS nulls_first,
       ordering.oprname::text AS operator_name,
       ordering.operator_schema,
       CASE WHEN ordering.oid IS NOT NULL
                  AND obj.in_heap_scope
                  AND ic.relam = 403 AND u.index_oid IS NULL
                  AND opn.nspname = 'pg_catalog'
                  AND opc.oid < 16384
                  AND (a.atttypid < 16384 OR typ.typtype = 'e')
                  AND (i.indcollation[k.n - 1] = 0 OR coll.oid IS NOT NULL)
            THEN format('(%s)%s USING OPERATOR(%I.%s) NULLS %s',
                  pg_catalog.pg_get_indexdef(i.indexrelid, k.n, false),
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
       integrity.evidence_valid
           AND count(*) FILTER (WHERE r.in_heap_scope) > 0
           AND bool_and(r.status = 'PASS') FILTER (WHERE r.in_heap_scope) AS heap_scope_all_pass,
       integrity.evidence_valid
           AND count(*) FILTER (WHERE r.in_heap_scope) > 0
           AND bool_and(r.status = 'PASS') FILTER (WHERE r.in_heap_scope)
           AND count(*) FILTER (WHERE r.unsupported_table_am) = 0 AS schema_success,
       COALESCE(bool_and(r.exhaustive_coverage_complete) FILTER (WHERE r.in_heap_scope), false)
           AS exhaustive_coverage_complete,
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
           E'ROLLBACK /* wiki_hic12_recover_attempt */ TO SAVEPOINT hic_attempt;\nINSERT /* wiki_hic12_record_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, snapshot_survived) VALUES (%s, <SQL-quoted stage>, <SQL-quoted SQLSTATE or NULL>, <SQL-quoted message>, true);\nRELEASE /* wiki_hic12_release_error */ SAVEPOINT hic_attempt;',
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

This is a generated-statement protocol, not a stored procedure. Execute the steps separately:

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
6. On success, fill `import_template` with a properly SQL-quoted JSON value from **that exact execution**, execute it, and inspect `hic_attempt_report`. The template imports the document and releases the savepoint. A syntactically valid but unacceptable plan remains recorded with its rejection reasons; it earns no count credit.
7. On execution, JSON-import or inspection failure, preserve SQLSTATE/message/detail/hint in the client. Execute `error_template`, filling the stage (`execution`, `capture` or `inspection`), diagnostic fields and attempt ID. Its rollback-to-savepoint removes any count or partially imported evidence but preserves the attempt record created before the savepoint. Execute `post_count_sql` separately if the failure path rolled back its settings. Retry only through the next generated attempt. If the connection or the whole transaction is lost, stop: a savepoint in a replacement transaction is not recovery of this run.
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

The implementation keeps generation policy separate from eligibility. It generates orderings only for reviewed stock operators/opclasses and supported types; user-defined functions/operators, domains and other unreviewed types can leave an otherwise eligible B-tree untested. It avoids deparsing those expressions, rather than merely declining to execute their generated query. This is a conservative coverage policy, **not a security sandbox** for hostile database code or server hooks. Plain table-level SELECT is required by preflight even if a narrower privilege combination could execute a particular query. Relations with no eligible candidate need no heap attempt and retain a NULL baseline, never a fabricated zero.

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
| Constant-expression index and two indexes with equivalent ordering | Exercise absent full-index path and repeated selection; untested OIDs remain untested, so no false exhaustive MISMATCH |
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

The published [Measurement Script](#measurement-script) passed a fresh out-of-tree build and full suite, then a `verify` rerun with the build retained. These are results of this checker on disposable healthy fixtures, not evidence that every index or damaged database can be diagnosed.

| Executed case | Observed result and assertion |
|---|---|
| Namespace-only positive schema | **13 physical relations PASS**, zero observed count mismatches. Includes empty heap, mixed NULL/duplicate keys, `lower(k)` plus a comma-containing `coalesce` expression, `text_pattern_ops`, explicit `"C"` collation, mixed key directions/NULL placement and INCLUDE, unique/nonunique indexes, inheritance parent/child, physical partition, populated and empty-populated materialized views, unlogged heap and quoted relation/column/index names. The aborted inserted row is absent. |
| Explicit descendant closure | **14 physical relations PASS**; the outside-schema physical leaf is added, its unrelated neighbor is not. A foreign leaf keeps `all_descendants_supported` and `subtree_success` false. Namespace success is not an all-children heap claim. |
| Own temporary schema | The application temporary table with an `hic_` prefix passes with **2 rows**. The checker's OID-registered objects are excluded. |
| Concurrent DML between counts | Both accepted counts remain **100** in the checking snapshot. The writer's later snapshot sees **95**: updates preserve cardinality, six committed deletes remove rows, one committed insert adds a row, and the aborted insert adds none. The fixture uses fillfactor 50 to leave room for HOT updates and asserts both HOT and non-HOT transaction counters. |
| Native `indcheckxmin` gate | With a separate old-XID holder active, a HOT update followed by a nonconcurrent index build sets the flag; the checker sees **120 heap rows**, cannot obtain an accepted index scan and reports **INCONCLUSIVE**. After ending the holder, a **new checking transaction** reports **PASS**, still with the flag true. Both checking snapshots are taken after the build; the distinction is their backend horizon, not inventory invisibility of the new index. |
| Equivalent-order candidate competition | Exhaustive mode reports PASS but incomplete coverage. Both attempts intended for `wide_idx` actually use `compact_idx`; the wider INCLUDE index remains untested. The fixture analyzes the table and deliberately makes the INCLUDE payload wide. |
| Negative inventory/path cases | Four relations report **NO_ELIGIBLE_INDEX**: absent index, partial-only, hash-only, and a genuinely invalid concurrent unique build. Three report **INCONCLUSIVE**: constant ordering loses its useful index path, or generation is deferred for a user function/domain. No case is converted to a zero-count pass. |
| Role/permission cases | An ordinary effective role selected with SET ROLE, without superuser or BYPASSRLS privileges, checks the unrestricted positive schema. A forced-RLS owner gets ERROR with SQLSTATE `42501`, not a filtered count; an authorized superuser counts both RLS-table rows. Missing table SELECT or schema USAGE remains an inventoried ERROR. |
| Lost-plan retry | Discarding the first executed plan and rolling back its savepoint leaves one retained capture error. The same-snapshot retry establishes PASS; there are **three attempt records**, two accepted and one failed. |
| Validator and evaluator units | **33 plan-shape assertions** pass. Additional synthetic records test state-flag exclusions, complete-coverage MISMATCH with a recovered failure, one mismatch plus an untested candidate yielding INCONCLUSIVE, missing evidence, incorrect settings/attempt marker, mixed snapshots, PASS with an optional error, unsupported-AM aggregation and equal-size but different identity bags. These records do not represent physical index corruption. |
| Initialization and history | Missing DDL attestation and missing schema cannot establish a run; the initialization-error channel is exercised. Empty physical scope is not success. Reports retain their verdict after COMMIT, including when the caller's `quote_all_identifiers = on` setting is restored. |

The concurrency arithmetic is `100 - 6 + 1 = 95` for the writer's current view, while the checker's retained snapshot produces `H = I = 100`. The held-snapshot behavior comes from repeatable read and heap visibility; the deliberate HOT fixture and retained-old-horizon index refusal have separate causes, detailed below. [snapmgr.c#retained-snapshot](../../../../raw/postgres-12/src/backend/utils/time/snapmgr.c#L339-L370) [heapam.c#HOT-search](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1559-L1622) [plancat.c#indcheckxmin-gate](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234)

**Not exercised by this suite:** physical file corruption; a real custom table AM; arbitrary expression/opclass/type combinations; non-C locale coverage or ICU-specific behavior; multiple inheritance and multilevel/attach/detach races; another backend's temporary relation; denied TEMP/function EXECUTE; a separate non-superuser BYPASSRLS role; open writers at initial snapshot acquisition; concurrent pruning/redirect coverage; forced lock/statement timeouts, cancellation, malformed JSON syntax or disconnect; standby operation; large-schema scaling and production I/O costs. Backward direction is covered by a plan unit, not a measured backward executor leg. The optional visible-`ctid` bag scan/capture pipeline is not implemented or measured; only the duplicate-sensitive comparison algebra is unit-tested.

### Optional relaxations

Keep these choices separate from the literal SQL baseline:

- **A PL/pgSQL wrapper, only if authorized.** A [PL/pgSQL](../../../glossary.md#plpgsql) function/procedure can loop over catalog rows, use dynamic `EXECUTE`, receive the EXPLAIN JSON result, manage exceptions and save error diagnostics. It must retain the explained temporary INSERT technique for an exact count and preserve the same validator and status rules. Use invoker privileges; do not mark a function wrapper STABLE or IMMUTABLE, because SPI's nonvolatile-function read-only check rejects these commands. It improves orchestration; it cannot force a named index or eliminate coverage gaps. [plpgsql.sgml#dynamic-commands](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1157-L1185) [plpgsql.sgml#data-versus-identifiers](../../../../raw/postgres-12/doc/src/sgml/plpgsql.sgml#L1225-L1249) [spi.c#nonvolatile-command-refusal](../../../../raw/postgres-12/src/backend/executor/spi.c#L2258-L2263)
- **psql conveniences.** Built-in `psql` has `\gexec` for executing generated SQL and `\if` for client-side branches. These are client meta-commands, not literal SQL. `\gexec` does not itself import each generated EXPLAIN result into an audit table, and generated rows cannot contain psql meta-commands or variable references. Treat this as an optional client guide, not a promised fully automated capture layer. [psql-ref.sgml#gexec](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L2059-L2094) [psql-ref.sgml#if](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L2233-L2260)
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
| Literal SQL does not supply an automatic EXPLAIN-result ingestion/error-handling loop | Make operator interaction explicit; offer PL/pgSQL only as a relaxation |

The review compared the complete plan, its query shapes, statuses, diagrams and test expectations with pinned implementation and official documentation, the relevant glossary definitions, matching-version comparison pages and the new navigation summaries. One existing glossary contradiction was identified and corrected: its `indcheckxmin` entry said the planner leaves an index out “while the flag holds,” whereas the source also requires the horizon test to fail. One pinned documentation wording discrepancy is retained under Open Questions: the index-scan switch is described as enabling/disabling plans, while implementation uses a cost penalty. No unresolved contradiction is intended outside Open Questions.

Implementation testing also caught two concrete capture/identity issues: a trailing command could hide the batch's EXPLAIN result, and the JSON index name uses SQL identifier quoting. The final generator separates the post-count controls; the identity helper applies the same quoting independently of the caller's later settings. The exact published installation and Bash blocks were compared byte for byte with their tested artifacts, and the extracted script was run from an empty sandbox and with the build retained. Human verification remains separate from these agent tests. Universal coverage, automatic DDL protection, unattended literal-SQL orchestration and corruption-detection rates are not claimed.

Final causal chain: coordinated scope and one retained snapshot → captured inventory → generated explained count INSERT → exact count and faithfully imported plan → context/shape validation → actual-index coverage → relation/schema verdicts → export and COMMIT. A new run starts the chain again; it does not reuse a completed snapshot's baseline.

## Measurement Script

This is the **disposable verification harness**, not an extra requirement for the operator-assisted checker. Its fixtures deliberately create objects and change rows in an isolated cluster. Do not point them at an application database. The C helper only checks that the loopback TCP port is available; it does not inspect PostgreSQL data.

Save the complete Bash block as `.wiki-runtime/tmp/hic12-measure.sh`. Run it from the repository root:

```text
bash .wiki-runtime/tmp/hic12-measure.sh prepare build init verify
bash .wiki-runtime/tmp/hic12-measure.sh verify
bash .wiki-runtime/tmp/hic12-measure.sh cleanup
```

With no arguments, the default order is `prepare build init verify cleanup`. Every invocation stops its owned server on exit. Omitting `cleanup` retains the stopped build and results for a rerun; it does not leave a server running.

| Item | Contract |
|---|---|
| Purpose | Test the installed SQL, the count-and-plan bridge, verdict rules, scope rules, concurrent DML, permissions and the native `indcheckxmin` gate. |
| `prepare` | Verify the pin and clean raw checkout, create or recognize the owned sandbox, extract this page's SQL installation block, emit the embedded fixtures/test driver, and compile the port probe. |
| `build` | Run `prepare`, stop an already-running owned cluster, and build/install PostgreSQL out of tree. Reuses an existing build. |
| `init` | Run `prepare`; initialize UTF-8/C-locale disposable data if absent; start only the owned Unix-socket server on port 55447. |
| `verify` | Run `init`; capture platform/settings; create a new uniquely named fixture database and run every assertion. Repeated invocations do not reuse fixture tables. |
| `cleanup` | Cleanly stop the owned cluster, check PID/process/socket/loopback-port teardown, then delete `.wiki-runtime/tmp/hic12/`, including its build and results. Idempotent when absent. |
| Environment | Explicit inputs: `JOBS=2`, `CC=cc`. Ordinary tool lookup uses `PATH`; the compiler accepts its normal build environment. The script fixes `LC_ALL=C`, `PGCLIENTENCODING=UTF8`, `TMPDIR` and `PGPASSFILE` inside the sandbox, and unsets connection-service/host-address and `PGOPTIONS` overrides. Client connection arguments are explicit. |
| Prerequisites | Bash, Git, POSIX utilities, `pgrep`, a PostgreSQL source-build toolchain and C compiler. No network, Python, external harness, extension download or planner-hint extension is used by the published script. The repository path must be short and contain no whitespace or shell metacharacters. |
| Build | `--without-readline --without-zlib`, with a private install prefix. These omit client command-line editing and zlib support; they are not planner or visibility settings. |
| Output | Read `suite-current.log` and `last-test-output` first. Each scenario exports attempts with plans, candidates, objects and run metadata as CSV under `results/<unique-database>/`; platform facts are in `platform.txt`, `platform-sql.txt` and `build-options.txt`. Keep a private copy before cleanup if needed. |
| Runtime | The recorded fresh run spent 101 seconds building and 15 seconds verifying, roughly two minutes overall; the earlier build-retained verification legs also took 15 seconds. The script prints each stage's actual `elapsed_seconds`. These are local correctness-test timings, not production-scan performance estimates. |

**Last run:** 2026-10-23, PostgreSQL 12.2 built from `45b88269a353ad93744772791feb6d01bc7e1e42`; Linux `6.18.54.1-microsoft-standard-WSL2`, `x86_64`, GCC 13.3.0, 64-bit; block size 8192 bytes, maximum data alignment 8 bytes; UTF-8 encoding, C collation/ctype, data checksums disabled. The script records these platform facts rather than assuming them for other hosts.

Non-default setup is intentional:

| Setting or fixture | Default → this run, apply scope | Why the result depends on it |
|---|---|---|
| Scan profiles, timeouts, search path and identifier quoting | The preceding settings table gives defaults and session/transaction scope. Fixture sessions use `statement_timeout = '60s'`, validator units use `'30s'`, both with `lock_timeout = '2s'`. Tests set session `quote_all_identifiers = on` before initialization, while counts use local off. | Count profiles seek the required path; post-count controls avoid applying scan-discouragement penalties to report queries. The quoting override deliberately exercises restoration after COMMIT. |
| `port` | Default 5432 → 55447; `postmaster`, startup/restart | Uses a separate test endpoint, including the Unix-socket name. [configure.in#default-port](../../../../raw/postgres-12/configure.in#L149-L162) [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2176-L2184) |
| `listen_addresses`, `unix_socket_directories`, `unix_socket_permissions` | `localhost`, `/tmp`, `0777` → empty TCP-listen list, private sandbox socket directory, `0700`; all `postmaster`, startup/restart | Postmaster opens sockets from these lists and applies the Unix-socket mode. The disposable trust-authentication cluster is not exposed through a TCP listener or a generally accessible socket. [guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3935-L3960) [guc.c#socket-mode](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2186-L2199) [pg_config_manual.h#socket-default](../../../../raw/postgres-12/src/include/pg_config_manual.h#L178-L182) [postmaster.c#TCP-list](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L1011-L1043) [postmaster.c#Unix-list](../../../../raw/postgres-12/src/backend/postmaster/postmaster.c#L1109-L1138) [pqcomm.c#socket-permissions](../../../../raw/postgres-12/src/backend/libpq/pqcomm.c#L692-L699) |
| Effective role | No SET ROLE override → `hic_reader` in permission cases or `hic_dba` in administrative cases; `user`, session scope, subject to authorization | These branches must exercise actual current-role privileges, not merely label a superuser result as ordinary. [guc.c#role](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3824-L3834) [rls.c#role-check](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132) |
| `client_encoding` | Boot value SQL_ASCII, dynamically defaults to database encoding; explicitly UTF8 here through `PGCLIENTENCODING`; `user`, connection/session scope | Keeps the non-ASCII fixture text and JSON imports consistent with the UTF-8 database. [guc.c#client-encoding](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3581-L3589) [postinit.c#encoding-default](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L394-L401) |
| Heap fillfactor | Default 100 → 50 on the concurrent-write and horizon fixtures; table storage parameter, set at CREATE TABLE, not a GUC | Insertion reserves free page space; a same-page update that changes no HOT-blocking indexed attributes can use HOT. Updating the indexed `id` instead exercises non-HOT. The gate fixture changes `k` before its index exists. [rel.h#heap-fillfactor](../../../../raw/postgres-12/src/include/utils/rel.h#L278-L279) [hio.c#reserved-space](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L348-L350) [heapam.c#HOT-decision](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3591-L3601) |
| Retained old horizon and late index build | A newly created index normally starts with `indcheckxmin = false`; this fixture retains old versions while a nonconcurrent build encounters a potentially broken HOT chain | `index_build()` sets the flag for that branch. The planner rejects the index while its catalog xmin does not precede `TransactionXmin`. Ending the holder and beginning a new checker transaction changes the horizon; the flag alone need not clear. `old_snapshot_threshold` remains at its default `-1`, so early pruning is not the cause here. [index.c#initial-flag](../../../../raw/postgres-12/src/backend/catalog/index.c#L609-L615) [index.c#broken-chain-gate](../../../../raw/postgres-12/src/backend/catalog/index.c#L2921-L2971) [plancat.c#horizon](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234) |
| Forced RLS owner | New relation row-security flags start false; fixture explicitly ENABLEs and FORCEs RLS and installs the subset policy | This forces the owner through the RLS decision. `row_security = off` then errors for that role, while the authorized superuser bypass branch remains available. [relcache.c#new-relation-state](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3287-L3295) [heap.c#security-flags](../../../../raw/postgres-12/src/backend/catalog/heap.c#L916-L919) [rls.c#forced-owner](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132) |

The build normally includes readline and zlib support; the two configure options disable interactive line editing and compressed dump/archive support, respectively. Those features are unnecessary for this noninteractive SQL test. [installation.sgml#optional-libraries](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L118-L151)

The one script below contains all fixtures and test orchestration. It reads the installation block from this same page, so there is only one maintained copy of the checker SQL.

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
compiler=${CC:-cc}
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
        -c "SHOW /* wiki_hic12_own_directory */ data_directory;")
    [ "$actual" = "$data" ]
}
stop_owned() {
    owned || return 1
    if [ -f "$data/postmaster.pid" ]; then
        same_server || return 1
        "$bin/pg_ctl" -D "$data" -m fast -w stop
    fi
    [ ! -e "$data/postmaster.pid" ]
    if pgrep -f -- "$data"; then return 1; fi
    [ ! -e "$socket/.s.PGSQL.55447" ]
    [ ! -e "$socket/.s.PGSQL.55447.lock" ]
    "$work/port-free"
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
    cat > "$work/port-free.c" <<'HIC_PORT_C'
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    int fd, on = 1, rc;
    struct sockaddr_in address;
    struct sockaddr_in6 address6;
    memset(&address, 0, sizeof(address));
    address.sin_family = AF_INET;
    address.sin_port = htons(55447);
    address.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) { perror("IPv4 socket"); return 1; }
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0) {
        perror("IPv4 reuse"); close(fd); return 1;
    }
    rc = bind(fd, (struct sockaddr *)&address, sizeof(address));
    if (rc < 0) perror("IPv4 loopback port");
    close(fd);
    if (rc < 0) return 1;
    fd = socket(AF_INET6, SOCK_STREAM, 0);
    if (fd < 0) {
        if (errno == EAFNOSUPPORT || errno == EPROTONOSUPPORT) return 0;
        perror("IPv6 socket"); return 1;
    }
    memset(&address6, 0, sizeof(address6));
    address6.sin6_family = AF_INET6;
    address6.sin6_port = htons(55447);
    address6.sin6_addr = in6addr_loopback;
    if (setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, &on, sizeof(on)) < 0 ||
        setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0) {
        perror("IPv6 options"); close(fd); return 1;
    }
    rc = bind(fd, (struct sockaddr *)&address6, sizeof(address6));
    if (rc < 0 && errno == EADDRNOTAVAIL) { close(fd); return 0; }
    if (rc < 0) perror("IPv6 loopback port");
    close(fd);
    return rc < 0;
}
HIC_PORT_C
    "$compiler" -O2 -o "$work/port-free" "$work/port-free.c"
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
    "$work/port-free"
    if [ ! -f "$data/PG_VERSION" ]; then
        [ ! -e "$data" ]
        "$bin/initdb" -D "$data" -U hic_dba -A trust --locale=C --encoding=UTF8 > "$work/initdb.log" 2>&1
    fi
    if [ ! -f "$data/postmaster.pid" ]; then
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
    "$bin/psql" -X -qAt -v ON_ERROR_STOP=1 -h "$socket" -p 55447 -U hic_dba -d postgres \
        > "$work/platform-sql.txt" <<'HIC_PLATFORM_SQL'
SET /* wiki_hic12_platform_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_platform_lock_timeout */ lock_timeout = '2s';
SELECT /* wiki_hic12_platform_version */ version();
SELECT /* wiki_hic12_platform_settings */ name, setting, context, boot_val
FROM pg_catalog.pg_settings
WHERE name IN (
    'block_size', 'server_encoding', 'lc_collate', 'lc_ctype', 'TimeZone',
    'enable_seqscan', 'enable_indexscan', 'enable_indexonlyscan', 'enable_bitmapscan',
    'enable_sort', 'max_parallel_workers_per_gather', 'row_security', 'quote_all_identifiers',
    'statement_timeout', 'lock_timeout', 'old_snapshot_threshold', 'listen_addresses',
    'unix_socket_directories', 'unix_socket_permissions', 'port', 'search_path', 'role',
    'application_name', 'track_counts', 'default_table_access_method'
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
    local fence
    fence=$(printf '\140\140\140')
    sed -n '/^### Executable SQL installation$/,/^### /p' "$page" |
        sed -n "/^${fence}sql$/,/^${fence}$/p" |
        sed '1d;$d' > "$work/checker.sql"
    [ -s "$work/checker.sql" ]
    IFS= read -r first_line < "$work/checker.sql"
    [ "$first_line" = 'SET /* wiki_hic12_install_path */ search_path = pg_catalog, pg_temp;' ]
    cat > "$work/fixtures.sql" <<'HIC_FIXTURES_SQL'
SET /* wiki_hic12_fixture_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_fixture_lock_timeout */ lock_timeout = '2s';
SELECT /* wiki_hic12_fixture_role */ 'CREATE /* wiki_hic12_reader */ ROLE hic_reader'
WHERE NOT EXISTS (SELECT 1 FROM pg_catalog.pg_roles WHERE rolname = 'hic_reader')
\gexec
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
\set ON_ERROR_STOP off
CREATE /* wiki_hic12_invalid_index */ UNIQUE INDEX CONCURRENTLY invalid_idx ON hic_negative.invalid_index(id);
\set invalid_sqlstate :SQLSTATE
\set ON_ERROR_STOP on
SELECT /* wiki_hic12_invalid_error_assert */ 1 / (:'invalid_sqlstate' = '23505')::integer;
SELECT /* wiki_hic12_invalid_flag_assert */ 1 / (NOT indisvalid)::integer
FROM pg_catalog.pg_index WHERE indexrelid = 'hic_negative.invalid_idx'::regclass;

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
    cat > "$work/test.sh" <<'HIC_TEST_SH'
set -uo pipefail
set -e
root=$(pwd)
work="$root/.wiki-runtime/tmp/hic12"
bin="$work/install/bin"
db="hic_suite_$(date -u +%Y%m%d%H%M%S)_$$"
out="$work/results/$db"
ls -ld "$work"
mkdir -p "$out"
ls -ld "$out"
keeper_pid=
trap 'if [ -n "$keeper_pid" ]; then touch "$out/keeper.release"; wait "$keeper_pid" || :; fi' EXIT
psql=("$bin/psql" -X -qAt -v ON_ERROR_STOP=1 -h "$work/socket" -p 55447 -U hic_dba -d "$db")
"$bin/createdb" -h "$work/socket" -p 55447 -U hic_dba "$db"
"${psql[@]}" -f "$work/fixtures.sql" > "$out/fixtures.log" 2>&1
"${psql[@]}" -f "$work/checker.sql" -f "$work/validator-tests.sql" > "$out/validator.log" 2>&1

cat > "$out/loop.sql" <<SQL
SELECT /* wiki_hic12_test_next */ EXISTS(SELECT 1 FROM pg_temp.hic_next) AS have_attempt
\\gset
\\if :have_attempt
\\o $out/plan.json
\\set ON_ERROR_STOP off
SELECT /* wiki_hic12_test_execute */ sql_text FROM pg_temp.hic_next
\\gexec
\\set attempt_failed :ERROR
\\set ON_ERROR_STOP on
\\o
\\if :attempt_failed
\\set failed_code :LAST_ERROR_SQLSTATE
\\set failed_message :LAST_ERROR_MESSAGE
ROLLBACK /* wiki_hic12_test_recover */ TO SAVEPOINT hic_attempt;
INSERT /* wiki_hic12_test_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, snapshot_survived)
SELECT max(attempt_id), 'execution', :'failed_code', :'failed_message', true FROM pg_temp.hic_attempt;
\\else
\\set captured_json \`cat "$out/plan.json"\`
INSERT /* wiki_hic12_test_plan */ INTO pg_temp.hic_plan(attempt_id, document)
SELECT max(attempt_id), :'captured_json'::jsonb FROM pg_temp.hic_attempt;
\\endif
SELECT /* wiki_hic12_test_control */ pg_temp.hic_control_profile()
\\gexec
RELEASE /* wiki_hic12_test_accept */ SAVEPOINT hic_attempt;
INSERT /* wiki_hic12_test_bound */ INTO pg_temp.hic_assert
SELECT 'attempt_bound_' || max(attempt_id),
       count(*) <= 2 * (SELECT count(*) FROM pg_temp.hic_object WHERE in_heap_scope)
                   + 2 * (SELECT count(*) FROM pg_temp.hic_index WHERE eligible)
FROM pg_temp.hic_attempt;
\\i $out/loop.sql
\\endif
SQL
sed '/^\\i /d' "$out/loop.sql" > "$out/once.sql"
printf '' > "$out/manual.sql"

begin_case() {
    name=$1
    schema=$2
    external=$3
    exhaustive=$4
    role=$5
    driver="$out/$name.sql"
    cat > "$driver" <<SQL
SET /* wiki_hic12_test_role */ ROLE $role;
SET /* wiki_hic12_test_quote_restore */ quote_all_identifiers = on;
\\i $work/checker.sql
CREATE /* wiki_hic12_test_assertions */ TEMP TABLE hic_assert(label text PRIMARY KEY, ok boolean NOT NULL CHECK(ok));
INSERT /* wiki_hic12_test_config */ INTO pg_temp.hic_config(target_schema, ddl_stable, include_external_partitions, exhaustive)
VALUES ('$schema', true, $external, $exhaustive);
SELECT /* wiki_hic12_test_start */ sql_text FROM pg_temp.hic_start_script
\\gexec
\\i $out/${6:-loop}.sql
SQL
}

finish_case() {
    cat >> "$driver" <<SQL
INSERT /* wiki_hic12_test_finished */ INTO pg_temp.hic_assert
SELECT 'no_pending_evidence', NOT EXISTS (SELECT 1 FROM pg_temp.hic_pending);
INSERT /* wiki_hic12_test_terminal */ INTO pg_temp.hic_assert
SELECT 'attempts_complete', state = 'ATTEMPTS_COMPLETE_REVIEW_REPORT' FROM pg_temp.hic_progress;
INSERT /* wiki_hic12_test_language */ INTO pg_temp.hic_assert
SELECT 'only_sql_helpers', bool_and(l.lanname = 'sql' AND NOT p.prosecdef)
FROM pg_catalog.pg_proc AS p JOIN pg_catalog.pg_language AS l ON l.oid = p.prolang
WHERE p.pronamespace = pg_my_temp_schema();
UPDATE /* wiki_hic12_test_finish */ pg_temp.hic_run SET finished = clock_timestamp();
\\o $out/$name-attempts.csv
COPY /* wiki_hic12_test_export_attempts */ (SELECT * FROM pg_temp.hic_attempt_report ORDER BY attempt_id) TO STDOUT WITH (FORMAT CSV, HEADER true);
\\o $out/$name-candidates.csv
COPY /* wiki_hic12_test_export_candidates */ (SELECT * FROM pg_temp.hic_candidate_result ORDER BY index_oid) TO STDOUT WITH (FORMAT CSV, HEADER true);
\\o $out/$name-objects.csv
COPY /* wiki_hic12_test_export_objects */ (SELECT * FROM pg_temp.hic_relation_result ORDER BY relation_oid) TO STDOUT WITH (FORMAT CSV, HEADER true);
\\o $out/$name-run.csv
COPY /* wiki_hic12_test_export_run */ pg_temp.hic_run TO STDOUT WITH (FORMAT CSV, HEADER true);
\\o
SELECT /* wiki_hic12_test_relation_report */ qualified_name, status, heap_count, eligible_indexes, checked_indexes, untested_indexes
FROM pg_temp.hic_relation_result WHERE in_heap_scope ORDER BY relation_oid;
SELECT /* wiki_hic12_test_schema_report */ row_to_json(r) FROM pg_temp.hic_schema_result AS r;
SELECT /* wiki_hic12_test_assertion_total */ '$name', count(*) FROM pg_temp.hic_assert;
SELECT /* wiki_hic12_test_expected_history */ schema_success AS expected_schema_success FROM pg_temp.hic_schema_result
\\gset
COMMIT /* wiki_hic12_test_commit */;
SELECT /* wiki_hic12_test_history */ schema_success FROM pg_temp.hic_schema_result;
SELECT /* wiki_hic12_test_stable_history */ 1 / (schema_success = :'expected_schema_success'::boolean)::integer
FROM pg_temp.hic_schema_result;
SQL
    "${psql[@]}" -f "$driver" > "$out/$name.log" 2>&1
    cat "$out/$name.log"
}

begin_case positive hic_positive false false hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_positive_success */ INTO pg_temp.hic_assert SELECT 'schema_pass', schema_success FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_positive_scope_size */ INTO pg_temp.hic_assert
SELECT 'all_thirteen_physical_relations', physical_relations = 13 FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_positive_empty */ INTO pg_temp.hic_assert
SELECT 'empty_table', status = 'PASS' AND heap_count = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 'empty_table';
INSERT /* wiki_hic12_positive_nulls */ INTO pg_temp.hic_assert
SELECT 'nulls_duplicates_aborts', status = 'PASS' AND heap_count = 5 FROM pg_temp.hic_relation_result WHERE relation_name = 'nulls_duplicates';
INSERT /* wiki_hic12_positive_parent_only */ INTO pg_temp.hic_assert
SELECT 'inheritance_parent_only', status = 'PASS' AND heap_count = 1 FROM pg_temp.hic_relation_result WHERE relation_name = 'inh_parent';
INSERT /* wiki_hic12_positive_child_only */ INTO pg_temp.hic_assert
SELECT 'inheritance_child_only', status = 'PASS' AND heap_count = 2 FROM pg_temp.hic_relation_result WHERE relation_name = 'inh_child';
INSERT /* wiki_hic12_positive_matviews */ INTO pg_temp.hic_assert
SELECT 'empty_populated_matview', status = 'PASS' AND heap_count = 0 FROM pg_temp.hic_relation_result WHERE relation_name = 'empty_mv';
INSERT /* wiki_hic12_positive_unpopulated */ INTO pg_temp.hic_assert
SELECT 'unpopulated_not_empty', status = 'NOT_APPLICABLE' FROM pg_temp.hic_relation_result WHERE relation_name = 'unpopulated_mv';
INSERT /* wiki_hic12_positive_parent_gap */ INTO pg_temp.hic_assert
SELECT 'parent_scope_and_foreign_gaps', NOT all_descendants_in_scope AND NOT all_descendants_supported AND NOT subtree_success
FROM pg_temp.hic_parent_result;
INSERT /* wiki_hic12_positive_external_excluded */ INTO pg_temp.hic_assert
SELECT 'outside_partition_excluded', NOT EXISTS (SELECT 1 FROM pg_temp.hic_object WHERE schema_name = 'hic_outside');
INSERT /* wiki_hic12_positive_owned_excluded */ INTO pg_temp.hic_assert
SELECT 'checker_objects_excluded', NOT EXISTS (
    SELECT 1 FROM pg_temp.hic_object AS obj JOIN pg_temp.hic_owned AS own USING (relation_oid)
);
INSERT /* wiki_hic12_positive_all_counts */ INTO pg_temp.hic_assert
SELECT 'counts_have_validated_plans', bool_and(accepted) FROM pg_temp.hic_attempt_result;
SQL
finish_case

cat > "$out/writer.sql" <<'SQL'
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
SQL
begin_case concurrent hic_concurrent false false hic_dba once
cat >> "$driver" <<SQL
INSERT /* wiki_hic12_before_writer */ INTO pg_temp.hic_assert
SELECT 'only_heap_before_writer', count(*) = 1 AND bool_and(phase = 'heap' AND accepted AND row_count = 100)
FROM pg_temp.hic_attempt_result;
\\! "$bin/psql" -X -qAt -v ON_ERROR_STOP=1 -h "$work/socket" -p 55447 -U hic_dba -d "$db" -f "$out/writer.sql" > "$out/writer.log" 2>&1 && touch "$out/writer.ok"
\\set writer_ok \`if test -f "$out/writer.ok"; then printf true; else printf false; fi\`
INSERT /* wiki_hic12_writer_succeeded */ INTO pg_temp.hic_assert VALUES ('writer_succeeded', :'writer_ok'::boolean);
\\i $out/loop.sql
INSERT /* wiki_hic12_snapshot_stable */ INTO pg_temp.hic_assert
SELECT 'snapshot_survived_concurrent_changes', schema_success FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_old_rows_visible */ INTO pg_temp.hic_assert
SELECT 'same_100_rows', count(*) = 2 AND bool_and(accepted AND row_count = 100) FROM pg_temp.hic_attempt_result;
SQL
finish_case
cat "$out/writer.log"

begin_case external hic_positive true false hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_external_success */ INTO pg_temp.hic_assert SELECT 'schema_pass', schema_success FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_external_scope_size */ INTO pg_temp.hic_assert
SELECT 'all_fourteen_physical_relations', physical_relations = 14 FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_external_leaf */ INTO pg_temp.hic_assert
SELECT 'outside_leaf_checked', status = 'PASS' AND heap_count = 1 FROM pg_temp.hic_relation_result WHERE relation_name = 'partition_outside';
INSERT /* wiki_hic12_external_unrelated */ INTO pg_temp.hic_assert
SELECT 'outside_unrelated_excluded', NOT EXISTS (SELECT 1 FROM pg_temp.hic_object WHERE relation_name = 'unrelated');
INSERT /* wiki_hic12_external_parent */ INTO pg_temp.hic_assert
SELECT 'foreign_child_remains_explicit', all_descendants_in_scope AND NOT all_descendants_supported AND NOT subtree_success
FROM pg_temp.hic_parent_result;
SQL
finish_case

begin_case negative hic_negative false false hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_negative_schema */ INTO pg_temp.hic_assert SELECT 'schema_not_pass', NOT schema_success FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_negative_scope_size */ INTO pg_temp.hic_assert
SELECT 'all_seven_physical_relations', physical_relations = 7 FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_negative_no_indexes */ INTO pg_temp.hic_assert
SELECT 'no_eligible_' || relation_name, status = 'NO_ELIGIBLE_INDEX'
FROM pg_temp.hic_relation_result WHERE relation_name IN ('no_index', 'partial_only', 'hash_only', 'invalid_index');
INSERT /* wiki_hic12_negative_inconclusive */ INTO pg_temp.hic_assert
SELECT 'inconclusive_' || relation_name, status = 'INCONCLUSIVE'
FROM pg_temp.hic_relation_result WHERE relation_name IN ('constant_index', 'user_expression', 'domain_table');
INSERT /* wiki_hic12_negative_deparse_guard */ INTO pg_temp.hic_assert
SELECT 'user_code_not_deparsed', definition IS NULL AND order_sql IS NULL AND generation_reason = 'unreviewed_user_code_or_type'
FROM pg_temp.hic_index WHERE index_name = 'user_expression_idx';
INSERT /* wiki_hic12_negative_rejected_baseline_reuse */ INTO pg_temp.hic_assert
SELECT 'constant_order_did_not_force_index', count(*) = 2 AND NOT bool_or(accepted)
FROM pg_temp.hic_attempt_result WHERE intended_index_oid = 'hic_negative.constant_idx'::regclass;
SQL
finish_case

begin_case choice hic_choice false true hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_choice_schema */ INTO pg_temp.hic_assert
SELECT 'existential_pass_without_exhaustive_coverage', schema_success AND NOT exhaustive_coverage_complete FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_choice_actual */ INTO pg_temp.hic_assert
SELECT 'actual_not_intended', count(*) = 2 AND bool_and(accepted AND actual_index_oid = 'hic_choice.compact_idx'::regclass)
FROM pg_temp.hic_attempt_result WHERE intended_index_oid = 'hic_choice.wide_idx'::regclass;
INSERT /* wiki_hic12_choice_untested */ INTO pg_temp.hic_assert
SELECT 'wide_remains_untested', NOT actually_checked AND intended_attempts = 2
FROM pg_temp.hic_candidate_result WHERE index_name = 'wide_idx';
SQL
finish_case

begin_case rls hic_rls false false hic_reader
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_rls_error */ INTO pg_temp.hic_assert
SELECT 'rls_is_error_not_subset', status = 'ERROR' AND heap_count IS NULL FROM pg_temp.hic_relation_result WHERE relation_name = 't';
INSERT /* wiki_hic12_rls_sqlstate */ INTO pg_temp.hic_assert
SELECT 'rls_errors_preserved', count(*) = 2 AND bool_and(sqlstate = '42501') FROM pg_temp.hic_error;
SQL
finish_case

for item in denied namespace_denied; do
    begin_case "$item" "hic_$item" false false hic_reader
    cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_permission_error */ INTO pg_temp.hic_assert
SELECT 'permission_not_silently_skipped', status = 'ERROR' AND permission_error IS NOT NULL
FROM pg_temp.hic_relation_result WHERE relation_name = 't';
INSERT /* wiki_hic12_permission_no_attempt */ INTO pg_temp.hic_assert SELECT 'no_unpermitted_attempt', count(*) = 0 FROM pg_temp.hic_attempt;
INSERT /* wiki_hic12_permission_scope */ INTO pg_temp.hic_assert
SELECT 'denied_object_is_present', physical_relations = 1 FROM pg_temp.hic_schema_result;
SQL
    finish_case
done

begin_case empty_scope hic_empty_scope false false hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_empty_scope_assert */ INTO pg_temp.hic_assert
SELECT 'no_vacuous_pass', physical_relations = 0 AND NOT schema_success AND NOT heap_scope_all_pass FROM pg_temp.hic_schema_result;
SQL
finish_case

begin_case ordinary_role hic_positive false false hic_reader
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_ordinary_role_success */ INTO pg_temp.hic_assert
SELECT 'non_superuser_can_check', schema_success AND physical_relations = 13 FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_ordinary_role_identity */ INTO pg_temp.hic_assert
SELECT 'not_superuser_or_bypassrls', NOT rolsuper AND NOT rolbypassrls FROM pg_catalog.pg_roles WHERE rolname = current_user;
SQL
finish_case

begin_case rls_bypass hic_rls false false hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_rls_bypass_success */ INTO pg_temp.hic_assert
SELECT 'privileged_role_sees_all_rows', status = 'PASS' AND heap_count = 2 FROM pg_temp.hic_relation_result WHERE relation_name = 't';
SQL
finish_case

begin_case verdict_units hic_concurrent false false hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_eligibility_units */ INTO pg_temp.hic_assert
SELECT label, pg_temp.hic_index_eligible(kind, am_oid, am_name, am_type, partial, valid, ready, live) = expected
FROM (VALUES
    ('eligible_btree', 'i', 403::oid, 'btree', 'i', false, true, true, true, true),
    ('nonphysical_index', 'I', 403::oid, 'btree', 'i', false, true, true, true, false),
    ('partial', 'i', 403::oid, 'btree', 'i', true, true, true, true, false),
    ('invalid', 'i', 403::oid, 'btree', 'i', false, false, true, true, false),
    ('not_ready', 'i', 403::oid, 'btree', 'i', false, true, false, true, false),
    ('not_live', 'i', 403::oid, 'btree', 'i', false, true, true, false, false),
    ('unknown_ready', 'i', 403::oid, 'btree', 'i', false, true, NULL::boolean, true, false),
    ('unsupported_am', 'i', 0::oid, 'hash', 'i', false, true, true, true, false),
    ('wrong_am_type', 'i', 403::oid, 'btree', 't', false, true, true, true, false)
) AS fixture(label, kind, am_oid, am_name, am_type, partial, valid, ready, live, expected);

SAVEPOINT /* wiki_hic12_synthetic_mismatch */ verdict_fixture;
UPDATE /* wiki_hic12_synthetic_count */ pg_temp.hic_count AS c SET row_count = row_count + 1
FROM pg_temp.hic_attempt AS a WHERE a.attempt_id = c.attempt_id AND a.phase = 'index';
INSERT /* wiki_hic12_synthetic_covered_attempt */ INTO pg_temp.hic_attempt(
    attempt_id, relation_oid, phase, intended_index_oid, variant, statement_text
)
SELECT 98, relation_oid, 'index', index_oid, 2, 'synthetic covered-candidate failure'
FROM pg_temp.hic_index WHERE eligible;
INSERT /* wiki_hic12_synthetic_covered_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, snapshot_survived)
VALUES (98, 'capture', NULL, 'synthetic failure with independent accepted coverage', true);
SELECT /* wiki_hic12_mismatch_verdict */ status = 'MISMATCH' AND checked_indexes = eligible_indexes
AND error_attempts = 1 AND uncovered_error_attempts = 0 AS unit_ok
FROM pg_temp.hic_relation_result WHERE relation_name = 't'
\gset
ROLLBACK /* wiki_hic12_mismatch_restore */ TO SAVEPOINT verdict_fixture;
RELEASE /* wiki_hic12_mismatch_release */ SAVEPOINT verdict_fixture;
INSERT /* wiki_hic12_mismatch_assert */ INTO pg_temp.hic_assert VALUES ('synthetic_complete_mismatch', :'unit_ok'::boolean);

SAVEPOINT /* wiki_hic12_synthetic_partial_coverage */ verdict_fixture;
UPDATE /* wiki_hic12_synthetic_unmatched_count */ pg_temp.hic_count AS c SET row_count = row_count + 1
FROM pg_temp.hic_attempt AS a WHERE a.attempt_id = c.attempt_id AND a.phase = 'index';
INSERT /* wiki_hic12_synthetic_untested_candidate */ INTO pg_temp.hic_index
SELECT (jsonb_populate_record(NULL::pg_temp.hic_index, to_jsonb(idx) || jsonb_build_object(
    'index_oid', 4000000002::bigint, 'index_name', 'synthetic_untested_idx',
    'qualified_name', 'hic_concurrent.synthetic_untested_idx'
))).* FROM pg_temp.hic_index AS idx WHERE eligible;
SELECT /* wiki_hic12_partial_coverage_verdict */ status = 'INCONCLUSIVE'
AND checked_indexes = 1 AND untested_indexes = 1 AND mismatching_attempts = 1 AS unit_ok
FROM pg_temp.hic_relation_result WHERE relation_name = 't'
\gset
ROLLBACK /* wiki_hic12_partial_coverage_restore */ TO SAVEPOINT verdict_fixture;
RELEASE /* wiki_hic12_partial_coverage_release */ SAVEPOINT verdict_fixture;
INSERT /* wiki_hic12_partial_coverage_assert */ INTO pg_temp.hic_assert
VALUES ('one_mismatch_with_untested_candidate_is_inconclusive', :'unit_ok'::boolean);

SAVEPOINT /* wiki_hic12_synthetic_missing_plan */ verdict_fixture;
DELETE /* wiki_hic12_synthetic_remove_plan */ FROM pg_temp.hic_plan AS p
USING pg_temp.hic_attempt AS a WHERE a.attempt_id = p.attempt_id AND a.phase = 'index';
SELECT /* wiki_hic12_missing_plan_verdict */ status = 'INCONCLUSIVE'
AND EXISTS (SELECT 1 FROM pg_temp.hic_pending) AS unit_ok
FROM pg_temp.hic_relation_result WHERE relation_name = 't'
\gset
ROLLBACK /* wiki_hic12_missing_plan_restore */ TO SAVEPOINT verdict_fixture;
RELEASE /* wiki_hic12_missing_plan_release */ SAVEPOINT verdict_fixture;
INSERT /* wiki_hic12_missing_plan_assert */ INTO pg_temp.hic_assert VALUES ('missing_plan_is_not_pass', :'unit_ok'::boolean);

SAVEPOINT /* wiki_hic12_synthetic_profile */ verdict_fixture;
UPDATE /* wiki_hic12_synthetic_wrong_profile */ pg_temp.hic_count AS c
SET settings = jsonb_set(settings, '{row_security}', '"on"'::jsonb)
FROM pg_temp.hic_attempt AS a WHERE a.attempt_id = c.attempt_id AND a.phase = 'index';
SELECT /* wiki_hic12_profile_verdict */ status = 'INCONCLUSIVE' AS unit_ok
FROM pg_temp.hic_relation_result WHERE relation_name = 't'
\gset
ROLLBACK /* wiki_hic12_profile_restore */ TO SAVEPOINT verdict_fixture;
RELEASE /* wiki_hic12_profile_release */ SAVEPOINT verdict_fixture;
INSERT /* wiki_hic12_profile_assert */ INTO pg_temp.hic_assert VALUES ('profile_mismatch_is_not_pass', :'unit_ok'::boolean);

SAVEPOINT /* wiki_hic12_synthetic_snapshot */ verdict_fixture;
UPDATE /* wiki_hic12_synthetic_wrong_snapshot */ pg_temp.hic_count SET snapshot_text = 'wrong snapshot' WHERE attempt_id = 1;
SELECT /* wiki_hic12_snapshot_verdict */ status = 'ERROR' AS unit_ok
FROM pg_temp.hic_relation_result WHERE relation_name = 't'
\gset
ROLLBACK /* wiki_hic12_snapshot_restore */ TO SAVEPOINT verdict_fixture;
RELEASE /* wiki_hic12_snapshot_release */ SAVEPOINT verdict_fixture;
INSERT /* wiki_hic12_snapshot_assert */ INTO pg_temp.hic_assert VALUES ('mixed_snapshots_invalidate_run', :'unit_ok'::boolean);

SAVEPOINT /* wiki_hic12_synthetic_marker */ verdict_fixture;
UPDATE /* wiki_hic12_synthetic_wrong_marker */ pg_temp.hic_plan AS p
SET document = jsonb_set(p.document, marker.path || ARRAY['Output', '0'], to_jsonb('999::bigint'::text))
FROM (
    SELECT plan.attempt_id, node.path
    FROM pg_temp.hic_plan AS plan
    JOIN pg_temp.hic_attempt AS a USING (attempt_id)
    CROSS JOIN LATERAL pg_temp.hic_plan_nodes(plan.document) AS node
    WHERE a.phase = 'index' AND node.node ->> 'Node Type' = 'Aggregate'
) AS marker WHERE marker.attempt_id = p.attempt_id;
SELECT /* wiki_hic12_marker_verdict */ status = 'INCONCLUSIVE' AS unit_ok
FROM pg_temp.hic_relation_result WHERE relation_name = 't'
\gset
ROLLBACK /* wiki_hic12_marker_restore */ TO SAVEPOINT verdict_fixture;
RELEASE /* wiki_hic12_marker_release */ SAVEPOINT verdict_fixture;
INSERT /* wiki_hic12_marker_assert */ INTO pg_temp.hic_assert VALUES ('wrong_attempt_marker_is_not_pass', :'unit_ok'::boolean);

SAVEPOINT /* wiki_hic12_synthetic_optional_error */ verdict_fixture;
INSERT /* wiki_hic12_synthetic_attempt */ INTO pg_temp.hic_attempt(
    attempt_id, relation_oid, phase, intended_index_oid, variant, statement_text
)
SELECT 99, relation_oid, 'index', index_oid, 2, 'synthetic verdict fixture only'
FROM pg_temp.hic_index WHERE eligible;
INSERT /* wiki_hic12_synthetic_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, snapshot_survived)
VALUES (99, 'execution', 'XX000', 'synthetic optional failure, not physical index corruption', true);
SELECT /* wiki_hic12_optional_error_verdict */ status = 'PASS' AND error_attempts = 1 AS unit_ok
FROM pg_temp.hic_relation_result WHERE relation_name = 't'
\gset
ROLLBACK /* wiki_hic12_optional_error_restore */ TO SAVEPOINT verdict_fixture;
RELEASE /* wiki_hic12_optional_error_release */ SAVEPOINT verdict_fixture;
INSERT /* wiki_hic12_optional_error_assert */ INTO pg_temp.hic_assert VALUES ('existential_pass_retains_optional_error', :'unit_ok'::boolean);

SAVEPOINT /* wiki_hic12_synthetic_unsupported_am */ verdict_fixture;
INSERT /* wiki_hic12_synthetic_unsupported_object */ INTO pg_temp.hic_object
SELECT (jsonb_populate_record(NULL::pg_temp.hic_object, to_jsonb(obj) || jsonb_build_object(
    'relation_oid', 4000000000::bigint, 'relation_name', 'synthetic_unsupported',
    'qualified_name', 'hic_concurrent.synthetic_unsupported',
    'table_am_oid', 4000000001::bigint, 'table_am', 'synthetic',
    'in_heap_scope', false, 'exclusion_reason', 'unsupported_table_access_method',
    'unsupported_table_am', true
))).* FROM pg_temp.hic_object AS obj WHERE relation_name = 't';
SELECT /* wiki_hic12_unsupported_verdict */ heap_scope_all_pass AND NOT schema_success AND unsupported_table_ams = 1 AS unit_ok
FROM pg_temp.hic_schema_result
\gset
ROLLBACK /* wiki_hic12_unsupported_restore */ TO SAVEPOINT verdict_fixture;
RELEASE /* wiki_hic12_unsupported_release */ SAVEPOINT verdict_fixture;
INSERT /* wiki_hic12_unsupported_assert */ INTO pg_temp.hic_assert VALUES ('unsupported_am_prevents_blanket_success', :'unit_ok'::boolean);

INSERT /* wiki_hic12_bag_cancellation */ INTO pg_temp.hic_assert
WITH heap_bag(tid) AS (VALUES ('(0,1)'::tid), ('(0,2)'::tid)),
index_bag(tid) AS (VALUES ('(0,1)'::tid), ('(0,1)'::tid)),
missing AS (SELECT tid FROM heap_bag EXCEPT ALL SELECT tid FROM index_bag),
extra AS (SELECT tid FROM index_bag EXCEPT ALL SELECT tid FROM heap_bag)
SELECT 'equal_counts_can_hide_different_bags',
       (SELECT count(*) FROM heap_bag) = (SELECT count(*) FROM index_bag)
       AND (SELECT count(*) FROM missing) = 1 AND (SELECT count(*) FROM extra) = 1;
SQL
finish_case
begin_case capture_retry hic_concurrent false false hic_dba manual
cat >> "$driver" <<SQL
\\o $out/discarded-plan.json
SELECT /* wiki_hic12_discard_capture */ sql_text FROM pg_temp.hic_next
\\gexec
\\o
ROLLBACK /* wiki_hic12_retry_capture_rollback */ TO SAVEPOINT hic_attempt;
INSERT /* wiki_hic12_retry_capture_error */ INTO pg_temp.hic_error(attempt_id, stage, sqlstate, message, snapshot_survived)
SELECT max(attempt_id), 'capture', NULL, 'test deliberately discards the first executed plan', true FROM pg_temp.hic_attempt;
RELEASE /* wiki_hic12_retry_capture_release */ SAVEPOINT hic_attempt;
SELECT /* wiki_hic12_retry_capture_control */ pg_temp.hic_control_profile()
\\gexec
\\i $out/loop.sql
INSERT /* wiki_hic12_retry_capture_assert */ INTO pg_temp.hic_assert
SELECT 'capture_retry_keeps_same_snapshot', status = 'PASS' AND error_attempts = 1 AND uncovered_error_attempts = 0
FROM pg_temp.hic_relation_result WHERE relation_name = 't';
INSERT /* wiki_hic12_retry_capture_attempts */ INTO pg_temp.hic_assert
SELECT 'one_failed_and_two_verified_attempts', count(*) = 3
AND count(*) FILTER (WHERE accepted) = 2 AND count(*) FILTER (WHERE error_present) = 1
FROM pg_temp.hic_attempt_result;
SQL
finish_case
cat > "$out/keeper.sql" <<SQL
SET /* wiki_hic12_keeper_timeout */ statement_timeout = '60s';
SET /* wiki_hic12_keeper_lock_timeout */ lock_timeout = '2s';
BEGIN /* wiki_hic12_keeper_begin */ ISOLATION LEVEL REPEATABLE READ READ ONLY;
SELECT /* wiki_hic12_keeper_xid */ txid_current();
\\! touch "$out/keeper.ready"
\\! while ! test -f "$out/keeper.release"; do sleep 1; done
COMMIT /* wiki_hic12_keeper_finish */;
SQL
"${psql[@]}" -f "$out/keeper.sql" > "$out/keeper.log" 2>&1 &
keeper_pid=$!
tries=0
while [ ! -f "$out/keeper.ready" ]; do
    kill -0 "$keeper_pid"
    [ "$tries" -lt 30 ]
    sleep 1
    tries=$((tries + 1))
done
"${psql[@]}" > "$out/gate-setup.log" 2>&1 <<'SQL'
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
SQL
begin_case old_horizon hic_gate false false hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_old_horizon_status */ INTO pg_temp.hic_assert
SELECT 'old_horizon_remains_inconclusive', status = 'INCONCLUSIVE' AND heap_count = 120 AND checked_indexes = 0
FROM pg_temp.hic_relation_result WHERE relation_name = 't';
INSERT /* wiki_hic12_old_horizon_candidate */ INTO pg_temp.hic_assert
SELECT 'old_horizon_not_falsely_excluded', eligible AND indcheckxmin AND NOT actually_checked
AND snapshot_usability = 'indcheckxmin_unresolved'
FROM pg_temp.hic_candidate_result WHERE index_name = 'gate_idx';
SQL
finish_case
touch "$out/keeper.release"
wait "$keeper_pid"
keeper_pid=
begin_case fresh_horizon hic_gate false false hic_dba
cat >> "$driver" <<'SQL'
INSERT /* wiki_hic12_new_horizon_status */ INTO pg_temp.hic_assert
SELECT 'fresh_horizon_passes', status = 'PASS' AND heap_count = 120 FROM pg_temp.hic_relation_result WHERE relation_name = 't';
INSERT /* wiki_hic12_new_horizon_candidate */ INTO pg_temp.hic_assert
SELECT 'true_flag_can_be_usable', indcheckxmin AND actually_checked AND snapshot_usability = 'accepted_by_executed_plan'
FROM pg_temp.hic_candidate_result WHERE index_name = 'gate_idx';
SQL
finish_case
cat "$out/gate-setup.log"

name=own_temp_scope
driver="$out/$name.sql"
cat > "$driver" <<SQL
\\i $work/checker.sql
CREATE /* wiki_hic12_temp_assertions */ TEMP TABLE hic_assert(label text PRIMARY KEY, ok boolean NOT NULL CHECK(ok));
INSERT /* wiki_hic12_register_test_helper */ INTO pg_temp.hic_owned VALUES ('pg_temp.hic_assert'::regclass);
CREATE /* wiki_hic12_temp_fixture */ TEMP TABLE hic_user_data(k integer) USING heap;
INSERT /* wiki_hic12_temp_rows */ INTO pg_temp.hic_user_data VALUES (1), (NULL);
CREATE /* wiki_hic12_temp_index */ INDEX hic_user_data_idx ON pg_temp.hic_user_data(k);
INSERT /* wiki_hic12_temp_configuration */ INTO pg_temp.hic_config(target_schema, ddl_stable)
SELECT nspname, true FROM pg_catalog.pg_namespace WHERE oid = pg_my_temp_schema();
SELECT /* wiki_hic12_temp_start */ sql_text FROM pg_temp.hic_start_script
\\gexec
\\i $out/loop.sql
INSERT /* wiki_hic12_temp_scope_assert */ INTO pg_temp.hic_assert
SELECT 'only_application_temp_relation', schema_success AND physical_relations = 1 FROM pg_temp.hic_schema_result;
INSERT /* wiki_hic12_temp_name_assert */ INTO pg_temp.hic_assert
SELECT 'prefix_does_not_hide_user_table', status = 'PASS' AND heap_count = 2
FROM pg_temp.hic_relation_result WHERE relation_name = 'hic_user_data';
SQL
finish_case

cat > "$out/bootstrap.sql" <<SQL
\\i $work/checker.sql
\\set ON_ERROR_STOP off
INSERT /* wiki_hic12_missing_ddl_agreement */ INTO pg_temp.hic_config(target_schema, ddl_stable) VALUES ('hic_positive', false);
\\set ddl_code :LAST_ERROR_SQLSTATE
\\set ON_ERROR_STOP on
SELECT /* wiki_hic12_ddl_guard_assert */ 1 / (:'ddl_code' = '23514')::integer;
SELECT /* wiki_hic12_no_false_configuration */ 1 / (count(*) = 0)::integer FROM pg_temp.hic_config;
INSERT /* wiki_hic12_missing_schema_configuration */ INTO pg_temp.hic_config(target_schema, ddl_stable) VALUES ('hic_missing_schema', true);
\\set ON_ERROR_STOP off
SELECT /* wiki_hic12_missing_schema_start */ sql_text FROM pg_temp.hic_start_script
\\gexec
\\set schema_code :LAST_ERROR_SQLSTATE
\\set schema_message :LAST_ERROR_MESSAGE
\\set ON_ERROR_STOP on
ROLLBACK /* wiki_hic12_failed_initialization */;
SELECT /* wiki_hic12_missing_schema_assert */ 1 / (:'schema_code' = '23514')::integer;
INSERT /* wiki_hic12_initialization_error */ INTO pg_temp.hic_run_error(
    event_id, requested_schema, stage, sqlstate, message
) VALUES (1, 'hic_missing_schema', 'initialization', :'schema_code', :'schema_message');
SELECT /* wiki_hic12_global_error_state */ 1 / (state = 'ERROR')::integer FROM pg_temp.hic_progress;
SELECT /* wiki_hic12_no_partial_success */ 1 / (
    NOT EXISTS (SELECT 1 FROM pg_temp.hic_schema_result)
    AND NOT EXISTS (SELECT 1 FROM pg_temp.hic_next)
)::integer;
SELECT /* wiki_hic12_bootstrap_result */ 'bootstrap_assertions', 5;
SQL
"${psql[@]}" -f "$out/bootstrap.sql" > "$out/bootstrap.log" 2>&1
cat "$out/bootstrap.log"
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
- Extension/build boundary: all main operations use existing SQL catalogs and executor facilities. Catalog `.h`/`.dat` definitions feed generated bootstrap data and catalog headers; the checker introduces no C module or generated backend header. The reproduction harness's C port probe is teardown tooling only. pageinspect is a separately built contrib module with page/B-tree/BRIN/GIN/hash regression targets and was not used for the counts. [catalog/Makefile#catalog-inputs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L55-L69) [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L101) [pageinspect/Makefile#module-and-tests](../../../../raw/postgres-12/contrib/pageinspect/Makefile#L1-L15)
- Tests inspected: core expression/pattern-index, RLS, materialized-view and inheritance cases cited in the test section; adjacent index-only isolation coverage is not an end-to-end test of this design.
- Glossary reviewed: heap/access method, B-tree, MVCC/snapshot/isolation, sequential/index/index-only scans, RLS, expression/partial indexes, collation/opclass, HOT, TID, pageinspect, PL/pgSQL, indcheckxmin and TransactionXmin. Added Materialized view and PostgreSQL 12 qualifications for the two horizon terms.
- Comparison pages: NULL values in indexes; CREATE INDEX CONCURRENTLY; invalid-index outcomes; physical index statistics; partitioned indexes; EXPLAIN ANALYZE BUFFERS; relation-tail checking; post-schema-drop health. Navigation and dated records were used for context, not as PostgreSQL evidence.
- PostgreSQL 12 has no common-concept pages. This answer supplies the mechanism context needed here and does not create a concept page.
- Implementation-specific context: client result handling in `PQexecFinish()`, SQL-quoted EXPLAIN index names, custom type-output calls during deparsing, row estimates for SQL set-returning helpers, initialization/attempt error channels and historical reports after COMMIT.
- Runtime verification: the complete published SQL and extracted test harness were exercised on an isolated exact-pin build, including fresh and build-retained workflows. Only the owned sandbox was used; no application database, network fetch, subagent, commit or push was used.

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
| Inspection itself can invoke a type-output function | [ruleutils.c#get_const_expr](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L9522-L9550) | User-function/domain fixtures retain untested candidates rather than silently excluding them |
| Nonconcurrent build's broken-HOT-chain usability gate | [index.c#index_build](../../../../raw/postgres-12/src/backend/catalog/index.c#L2921-L2971), [plancat.c#horizon](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L222-L234) | Old-holder/new-transaction fixture in the published measurement script |
| SQL orchestration limitation | [xml.c](../../../../raw/postgres-12/src/backend/utils/adt/xml.c#L2671-L2689), [utility.c](../../../../raw/postgres-12/src/backend/tcop/utility.c#L99-L118) | [execute.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/execute.sgml#L28-L50), [psql-ref.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L2059-L2094) |
| Heap visibility and HOT resolution | [indexam.c](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L607-L635), [heapam.c](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1559-L1637) | [README.HOT](../../../../raw/postgres-12/src/backend/access/heap/README.HOT#L48-L122) |
| Physical inspection is not visible-row counting | [rawpage.c](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L159-L169), [heapfuncs.c](../../../../raw/postgres-12/contrib/pageinspect/heapfuncs.c#L180-L224) | [pageinspect.sgml](../../../../raw/postgres-12/doc/src/sgml/pageinspect.sgml#L169-L175) |
| Duplicate-sensitive visible-identity comparison | [heaptuple.c](../../../../raw/postgres-12/src/backend/access/common/heaptuple.c#L636-L640), [heapam_handler.c](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L148-L167) | [select.sgml#EXCEPT-ALL](../../../../raw/postgres-12/doc/src/sgml/ref/select.sgml#L1281-L1294), [ddl.sgml#ctid](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L1198-L1205) |

## Open Questions

- **Compatibility beyond the exact pin is unverified.** The installed version guard is intentionally 12.2-only. Other minor releases, patched server binaries, EXPLAIN/planner hooks, custom types/opclasses and different locale combinations need their own review and tests.
- **Pinned documentation wording:** `config.sgml` says `enable_indexscan` “enables or disables” index-scan plan types, while `cost_index()` adds a penalty when it is off. The implementation settles the checker rule: a setting cannot certify the chosen path. The page uses that source behavior, and this wording discrepancy remains here rather than being treated as a hard planner prohibition. [config.sgml#indexscan-wording](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L4449-L4462) [costsize.c#indexscan-penalty](../../../../raw/postgres-12/src/backend/optimizer/path/costsize.c#L525-L535)
- **Complete arbitrary-candidate coverage is not established.** Constant/redundant orderings, competing equivalent indexes, snapshot usability, custom expressions and operator resolution can leave eligible indexes untested. No SQL-only per-index forcing mechanism is proposed. This is a documented feasibility limit, not a promised future outcome.
- **No physical fault-injection coverage yet.** Evaluator tests using synthetic records cannot establish detection rates for damaged index files. A real corruption suite needs separate authorization and a capability beyond the checker defined here.
- **Operational/test gaps remain explicit.** The unexercised cases are listed under measured coverage. The SQL does not prove that the DDL agreement was honored or that damaged catalogs retained a complete inventory; `inventory_complete` means the inventory statements completed. The optional visible-identity capture pipeline and large-schema/performance validation remain future work.
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

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Versions and source pins](../../../versions.md)
- [How NULL Values Are Handled in PostgreSQL 12 Indexes (unverified)](null-values-in-indexes.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](create-index-concurrently.md)
- [Physical Index Statistics, Tuple Counts, and Bytes per Tuple in PostgreSQL 12 (unverified)](physical-index-statistics-tuple-counts-and-bytes.md)
- [EXPLAIN ANALYZE BUFFERS Output in PostgreSQL 12 (unverified)](../observability/explain-analyze-buffers-output.md)
