---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Database Health After Schema Files Were Zeroed and the Schema Dropped in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The conclusion to start with](#the-conclusion-to-start-with)
  - [Incident boundary and mental model](#incident-boundary-and-mental-model)
  - [Decision map](#decision-map)
  - [Health checklist](#health-checklist)
  - [Run the diagnostic script](#run-the-diagnostic-script)
  - [What each stage covers](#what-each-stage-covers)
  - [Interpret findings without declaring false health](#interpret-findings-without-declaring-false-health)
  - [Settings, defaults and apply scopes](#settings-defaults-and-apply-scopes)
  - [State changes that affect the evidence](#state-changes-that-affect-the-evidence)
  - [Recovery and acceptance criteria](#recovery-and-acceptance-criteria)
  - [Build boundaries and validation](#build-boundaries-and-validation)
  - [Causal summary](#causal-summary)
  - [Diagnostic script](#diagnostic-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Follow AGENTS.md. For PostgreSQL 12, create a script and a health checklist for a database that had all objects in one schema zeroed out using `cat /dev/null > file.txt`, after which the schema was dropped using `DROP SCHEMA ... CASCADE`.

Follow-up requirement: “The script should be inside the question document.”

## Answer

### The conclusion to start with

**Check the surviving database and reconcile it against independent pre-incident evidence. Do not treat the successful drop as a health check.** With usable [catalogs](../../../glossary.md#catalog), the ordinary built-in deletion path schedules relation-file removal without scanning the table rows or verifying the index contents first. Consequently, success proves neither that the removed data was intact nor that the rest of the database is complete. [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1908) [heap.c#schedule-storage-removal](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1943-L1959) [index.c#index-drop-storage](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2209)

The full runnable deliverable is embedded under [Diagnostic script](#diagnostic-script). It collects evidence, reports failures and coverage gaps, and performs no repair. Its default stage is inventory; expensive file, value-scan, B-tree and dump stages require selection. A successful script exit is **not** a certificate that no rows or objects were lost. A newly measured short heap extent can hide an erased suffix from both ordinary scans and `amcheck`'s heap comparison. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [heapam.c#scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L219-L231) [verify_nbtree.c#heap-comparison](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624)

The engine evidence is PostgreSQL **12.2**, at the exact pin above. The script accepts PostgreSQL 12 servers and records their actual version; it does not establish that a different minor release or a patched server has identical implementation.

### Incident boundary and mental model

This runbook interprets the incident as **external truncation of selected relation files to zero length**. It does not assume same-length byte overwriting. The shell command and its actual pathname expansion were not supplied for inspection; that interpretation is an incident assumption, not a PostgreSQL source finding.

A [schema](../../../glossary.md#schema) is a namespace, not a directory. Physical paths use database, [tablespace](../../../glossary.md#tablespace), [relfilenode](../../../glossary.md#relfilenumber) and [fork](../../../glossary.md#fork) identities. “All objects” also cannot literally mean a separate file for every object: views and partitioned-table parents lack ordinary relation storage, while functions have namespace-bearing catalog definitions rather than their own table data files. First recover the actual list of affected paths rather than infer it from the schema name. [pg_namespace.h#namespace](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L24-L44) [relpath.c#GetRelationPath](../../../../raw/postgres-12/src/common/relpath.c#L129-L205) [dbsize.c#storage-bearing-kinds](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L949-L985) [pg_proc.h#function-identity](../../../../raw/postgres-12/src/include/catalog/pg_proc.h#L31-L45)

| Evidence | What it represents | Current or historical? | What it cannot establish |
|---|---|---|---|
| `pg_namespace`, `pg_class` and `pg_depend` | Objects and [dependencies](../../../glossary.md#dependency) currently visible to a catalog query | Current transactional state | The full list that existed before a committed drop; deletion removes both namespace and dependency records. [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L214-L235) [dependency.c#delete-dependency-records](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1269-L1309) |
| Current file lengths | Bytes at the mapped filenames when the file functions run | Live filesystem state, not a database-wide snapshot | The original lengths or contents. The scan-size reader stops at the first short segment it measures. [genfile.c#pg_stat_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L362-L427) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) |
| `pg_class.relpages` and `reltuples` | Stored page and tuple estimates | Stored, not necessarily up to date | An exact pre-incident extent or row count. [pg_class.h#estimates](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66) |
| Query and checker results | What the selected read paths could inspect | Per-query state; pages may come from [shared buffers](../../../glossary.md#buffer-manager) | An uncached disk audit or completeness of data beyond the current scan bound. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [heapam.c#scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L219-L231) |
| Backup, application ledger, saved definitions and incident file inventory | An independent expected state | Historical, with an explicit date and scope | These are required inputs to the acceptance policy below, not facts reconstructed by the script. |

### Decision map

```text
Preserve incident evidence and identify the database, times and exact paths
|
+-- Scope unknown, or catalogs/shared/non-relation files possibly affected?
|   `-- Escalate scope; do not certify just the surviving user tables [A]
|
`-- Assess the named database on a controlled target
    |
    +-- Drop not known committed / schema name still present?
    |   `-- Resolve transaction, database, name and recreation history [B]
    |
    `-- Inventory all survivors, not only objects in the removed namespace [C]
        |
        +-- Guard fails / permission denied / timeout / query fails / skip?
        |   `-- Incomplete evidence; preserve the error and resolve it [D]
        |
        `-- Catalog, extent, value, B-tree and export checks complete
            |
            +-- Missing expected objects/data or a reproducible inconsistency?
            |   `-- Recovery/reconciliation decision; no automatic repair [E]
            |
            `-- No detected inconsistency
                `-- Independent baseline + restore/application checks still
                    required before an operator accepts the database [E]
```

- **[A]** Physical addressing distinguishes shared `global` relations from per-database files; no schema directory limits the damage. [relpath.c#GetRelationPath](../../../../raw/postgres-12/src/common/relpath.c#L143-L205)
- **[B]** File cleanup is conditional on transaction outcome, not merely statement completion. [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L461)
- **[C]** Explicit `CASCADE` permits deletion of dependents beyond the named schema. The default is `RESTRICT`; it is the explicit option, not the empty files, that changes the dependency decision. [gram.y#opt_drop_behavior](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568) [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) [dependency.c#cascade-branch](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1113-L1175) [drop_schema.sgml#other-schemas](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L89-L95)
- **[D]** Ordinary value reads enforce permissions and [row-level security](../../../glossary.md#row-level-security); lock and statement timeouts bound work rather than validate data. [copy.c#permissions](../../../../raw/postgres-12/src/backend/commands/copy.c#L894-L951) [rls.c#row_security-off](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L120-L132) [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396)
- **[E]** The policy follows from the detection limits: shortened heaps supply fewer tuples, and the B-tree membership callback tests only the heap tuples supplied to it. [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) [verify_nbtree.c#membership-callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920)

### Health checklist

For each item, record **passed**, **failed/escalated**, or **skipped/incomplete**, together with the evidence location and reviewer. A passed item means its stated check completed satisfactorily, not that the whole database is complete. Never silently convert a skipped item into a pass.

- [ ] **Preserve first.** Record the incident window, original commands, expanded paths, database/server identity, tablespaces, client output, server logs and available backups. Arrange an incident-preservation copy through the established backup procedure before heavy reads or maintenance. The script uses read-only SQL, but this is not a promise of byte-for-byte nonmutation: an ordinary heap read can opportunistically prune a page. [heapam.c#heapgetpage](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L380-L405)
- [ ] **Establish the actual damage boundary.** Match each known path to its pre-incident object, database, fork and segment using saved metadata or a separately restored baseline. Include indexes and [TOAST](../../../glossary.md#toast), which stores oversized values separately. After deletion, the current catalogs cannot supply the removed rows' mapping. If the file list is unavailable, leave the scope unresolved rather than claim “one schema only.” If catalogs, shared relations or other database paths may have been hit, widen the investigation to the cluster and every affected database. [pg_class.h#toast-link](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L68-L69) [dbsize.c#pg_relation_filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1022) [dependency.c#metadata-removal](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1269-L1309)
- [ ] **Confirm the drop outcome.** Use a new connection to the intended database, check the exact schema name, and correlate with transaction completion and any subsequent recreation. Inspect the script's prepared-transaction and activity reports; do not automatically commit, roll back or terminate anything found there. A statement finishing inside an open transaction is not the same as commit-time file removal. [storage.c#atCommit](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [xact.c#commit-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232) [system_views.sql#pg_prepared_xacts](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L325-L335)
- [ ] **Audit cross-schema losses.** Compare saved definitions with the survivors: tables/partitions, views, functions, types, extensions, constraints and triggers. In particular, a surviving table can lose its foreign-key constraint when the referenced object is dropped; checking only surviving `convalidated` rows cannot detect a constraint that no longer exists. A foreign key has an automatic dependency on its owning relation and a normal dependency on the referenced relation. [pg_constraint.c#constraint-dependencies](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L233-L299) [dependency.c#cascade-walk](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437)
- [ ] **Run and review inventory.** Read every `REVIEW`, `ERROR`, `SKIP` and `.stderr` file, not just the process exit code. Reconcile the catalog-link candidates, invalid-index states, unvalidated constraints and trigger configuration with a baseline. [pg_index.h#state-flags](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L44) [pg_constraint.h#convalidated](../../../../raw/postgres-12/src/include/catalog/pg_constraint.h#L49-L80) [pg_trigger.h#trigger-state](../../../../raw/postgres-12/src/include/catalog/pg_trigger.h#L34-L49)
- [ ] **Inspect surviving physical extents.** Run `files` with suitable privileges. For auxiliary-fork witnesses, last-page reads and orphan-file candidates, also use the existing [last-segment investigation](../storage-and-vacuum/zeroed-last-segment-check-all-tables.md#the-tail-and-witness-check) and its [orphan-file section](../storage-and-vacuum/zeroed-last-segment-check-all-tables.md#orphan-files). Do not delete anything they list automatically. A zero-byte first main file after a committed drop can be normal deferred-unlink state; an unlink traversal stops at a missing filename and can leave later files. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) [sync.c#SyncPostCheckpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214)
- [ ] **Read surviving values and check supported indexes.** Run `scan` and, if already installed and authorized, `amcheck`. Investigate every excluded object and every failed read. A [B-tree](../../../glossary.md#b-tree) check does not validate GIN, GiST, SP-GiST, BRIN or hash indexes, and PostgreSQL 12's [`amcheck`](../../../glossary.md#amcheck) has no table-target heap checker. [verify_nbtree.c#btree_index_checkable](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L315-L338) [amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10)
- [ ] **Review logs across the incident and diagnostics.** Look for `could not read block`, `invalid page`, TOAST chunk errors, catalog lookup failures, and `could not truncate/remove file` warnings. Preserve full context; a timeout is not itself corruption, and a TOAST fetch failure can originate in its index rather than prove an absent heap chunk. Use the [version-12 log catalog](../observability/corruption-log-messages-with-checksums-disabled.md) for the broader surface. [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L629) [bufmgr.c#invalid-page](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) [tuptoaster.c#indexed-fetch](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1910-L1935) [tuptoaster.c#chunk-checks](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1965-L2016) [heap.c#lookup-error](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1871-L1873) [md.c#unlink-warnings](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L294-L355)
- [ ] **If checksums were already enabled, consider an offline check on an approved, cleanly stopped target.** `pg_checksums --check` requires a compatible, cleanly shut-down cluster with checksums enabled. It is supplemental, not proof against truncation: EOF ends a file scan, and new pages are skipped. Do not enable checksums now as a way to validate historical bytes; enable mode calculates and writes checksums for current contents. The script neither stops a cluster nor runs this utility. [pg_checksums.c#preconditions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L490-L527) [pg_checksums.c#scan-and-enable](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L169-L252)
- [ ] **Export, restore separately, and reconcile meaning.** Run the optional `dump` stage only with disk space and load approval. Restore a successful archive into a new isolated target, then compare expected keys, values, relationships, definitions and application invariants. A readable export is not a historical row-count guarantee; `pg_dump` uses COPY and the current catalog inventory. [pg_dump.c#dumpTableData_copy](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1786-L1837) [copy.c#table-scan](../../../../raw/postgres-12/src/backend/commands/copy.c#L2114-L2142) [heapam.c#scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L219-L231)
- [ ] **Complete the operational health review.** Apply the existing [database health checklist](database-health-checklist.md) to connections, lock waits, maintenance horizons, replication, archiving and workload behavior. Keep that operational assessment separate from the integrity and completeness decision. Current activity and cumulative database counters are different evidence surfaces; neither is a list of missing objects. [system_views.sql#activity](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L768) [system_views.sql#database-statistics](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L887)

### Run the diagnostic script

Run from the repository root. Replace the example target with the actual **direct** PostgreSQL connection, not a transaction-pooling endpoint. Preserve evidence and obtain load approval first. Nothing below reconnects to an automatically discovered database.

```bash
sed -n '/^#!\/usr\/bin\/env bash$/,/^exit 0$/p' \
  wiki/v12/questions/server-administration/health-after-zeroed-schema-drop.md \
  > .wiki-runtime/pg12-post-drop-health.sh

export PGHOST='/explicit/postgresql/socket'
export PGPORT='5432'
export PGDATABASE='affected_database'
export PGUSER='authorized_diagnostic_role'
export DROPPED_SCHEMA='actual_dropped_schema'
export PG12_BINDIR='/explicit/postgresql-12/bin'

bash .wiki-runtime/pg12-post-drop-health.sh inventory
bash .wiki-runtime/pg12-post-drop-health.sh files scan amcheck
bash .wiki-runtime/pg12-post-drop-health.sh dump
```

Alternatively, select `all`. Inventory runs first in every invocation; requested extra stages run in `files`, `scan`, `amcheck`, `dump` order. Each invocation creates a **new** private report directory under `.wiki-runtime/pg12-health-TIMESTAMP-PID/`. It does not overwrite previous reports or maintain a resumable server.

Use libpq authentication, such as an already configured `PGPASSFILE`; do not put passwords in the command. `-w` disables an interactive password prompt. The runner uses `psql -X -v ON_ERROR_STOP=1`, qualifies generated identifiers, quotes the schema variable as a SQL literal, and keeps per-object failures rather than calling them successes. `PGOPTIONS`, `PGSERVICE`, `PGSERVICEFILE` and `PGHOSTADDR` are replaced or disabled so they cannot silently change the script's selected options/host. Other libpq authentication and TLS configuration is inherited. [fe-connect.c#connection-options](../../../../raw/postgres-12/src/interfaces/libpq/fe-connect.c#L198-L251) [startup.c#password-and-startup-file-options](../../../../raw/postgres-12/src/bin/psql/startup.c#L647-L658) [mainloop.c#error-stop](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L558-L565) [psql-ref.sgml#ON_ERROR_STOP](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3845-L3858)

| Input | Default or requirement |
|---|---|
| `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `DROPPED_SCHEMA` | All required. One host/socket; a literal database name, not a URI/connection string. Database names containing `=` are deliberately refused. Use the schema's actual case-sensitive name. |
| `PG12_BINDIR` | Unset: find clients through `PATH`. The runner requires `psql` 12 and, for `dump`, `pg_dump` 12. |
| `PGCONNECT_TIMEOUT` | `10` seconds, a runner policy. |
| `HEALTH_STATEMENT_TIMEOUT_MS` | `30000`, per inventory/file statement. |
| `HEALTH_SCAN_TIMEOUT_MS` | `300000`, per value-scan/amcheck statement. |
| `HEALTH_LOCK_TIMEOUT_MS` | `2000`, per lock wait for psql checks and the dump's initial-lock option. |
| `HEALTH_DUMP_TIMEOUT_SECONDS` | `900`, deadline at which GNU `timeout` sends TERM to the dump job, with KILL after a further `10` seconds if needed. |

The time limits are **operator-selected limits**, not PostgreSQL health thresholds or predicted running times. Increase them only after reviewing the failing operation and its load. A timeout remains incomplete coverage until resolved.

Start with `summary.tsv`. `relations.csv` maps object OIDs in filenames back to names and current paths. `constraints.csv` saves surviving constraint definitions. `files_OID.csv` contains candidates, while `scan_OID.stderr` and `amcheck_OID.stderr` retain errors. Scan row data goes to `/dev/null`; a selected dump saves actual database data. Treat archives, metadata and error text as sensitive.

| Exit/status | Meaning |
|---|---|
| `0` | Requested automated work completed without an automatic finding; manual acceptance remains outstanding. |
| `1` / `ERROR`, `REVIEW` | A SQL/tool operation failed or a candidate needs review. A failed query's CSV can be partial. Neither label alone proves physical corruption. |
| `2` / `SKIP` | A prerequisite/guard prevented the run, or requested coverage was excluded/unavailable. If errors/findings also occurred, exit `1` takes precedence. |
| `130`, `143` / `INCOMPLETE` | Interrupted; do not treat the existing partial report as complete. Other abnormal exits also record `INCOMPLETE` when the exit trap runs. |
| `NOT_REQUESTED` | An optional stage was not selected, not passed. |

No database service is started by this script, so it has no cluster-cleanup stage. Reports remain for incident retention. It never installs `amcheck`, modifies privileges, vacuums, reindexes, forces a checkpoint, drops objects, deletes relation files, or restores over the affected database.

### What each stage covers

| Stage | Work and source-backed meaning | Important boundary |
|---|---|---|
| `inventory` | Captures identity, effective settings, namespaces, all `pg_class` rows, extensions, constraints, trigger states, activity, prepared transactions and database statistics. Screens namespace/TOAST/index/attribute/constraint/inheritance/trigger links and dependency endpoints for `pg_class`, `pg_namespace`, `pg_type`, `pg_proc` and `pg_constraint`. These identifiers come from the catalog layouts, not guessed filenames. [pg_class.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L29-L107) [pg_attribute.h#attribute-numbering](../../../../raw/postgres-12/src/include/catalog/pg_attribute.h#L37-L75) [pg_depend.h#endpoints](../../../../raw/postgres-12/src/include/catalog/pg_depend.h#L41-L64) [pg_constraint.h#references](../../../../raw/postgres-12/src/include/catalog/pg_constraint.h#L49-L95) [pg_inherits.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_inherits.h#L32-L37) [pg_trigger.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_trigger.h#L34-L49) | Not a full catalog validator. It does not validate every object class, dependency subobject, type invariant or missing dependency edge. Rows that have entirely disappeared cannot generate an orphan-link finding. A missing schema is not independent evidence that it used to exist. |
| `files` | For every current non-temporary storage-bearing relation, stats consecutive **main-fork** segment names through the first missing filename. Computes the logical extent through the first short segment and reports a missing main file, non-block-aligned/oversized segment, nonzero segment after a short one, or stored `relpages` exceeding that extent. Mapped catalogs use `pg_relation_filepath()`, not `relfilenode` alone. [dbsize.c#filepath-and-mapper](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L949-L976) [genfile.c#pg_stat_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L362-L427) [md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L44-L66) [md.c#logical-extent](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) | Does not read page contents, inspect auxiliary forks, enumerate files beyond a missing segment, or find objects absent from `pg_class`. Empty storage and zero-length inactive segments can be legitimate. `relpages` is only a weak witness; stale estimates can also flag a healthy table. [pg_class.h#relpages](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66) [md.c#inactive-segments](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66) |
| `scan` | Separately serializes `SELECT * FROM ONLY qualified_relation` through COPY for current non-temporary heap tables, populated materialized views, TOAST heaps and sequences, including system catalogs. Planner index paths are discouraged for the query scan. COPY invokes value output functions; external values can therefore exercise TOAST retrieval rather than merely count rows. [copy.c#query-and-output-paths](../../../../raw/postgres-12/src/backend/commands/copy.c#L2114-L2142) [copy.c#output-functions](../../../../raw/postgres-12/src/backend/commands/copy.c#L2202-L2213) [tuptoaster.c#toast-fetch](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1910-L1935) | Foreign tables, temporary relations, unpopulated materialized views and non-heap table access methods are listed as exclusions. Partitioned parents have no ordinary storage; the inventory supplies their individual storage-bearing leaves. Only visible values are serialized, not every dead/dropped attribute or unreachable TOAST value. Custom type output code remains a boundary. [pg_class.h#kinds](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) [copy.c#output-function-selection](../../../../raw/postgres-12/src/backend/commands/copy.c#L2035-L2052) |
| `amcheck` | Finds the extension-owned `bt_index_check(regclass, boolean)` overload, even after extension relocation, and calls it with `true` for eligible valid/live/ready non-temporary B-tree indexes on built-in heap relations, system and TOAST indexes included. The normal one-argument API has `heapallindexed = false`; this runner explicitly enables the additional heap-to-index membership pass. [amcheck--1.0--1.1.sql#overload](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0--1.1.sql#L9-L29) [verify_nbtree.c#defaults-and-lock](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L191) [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624) | No automatic installation; missing privileges/overload and unsupported indexes are explicit gaps. `AccessShareLock` does not make the operation free of load or DDL waits. This is not `bt_index_parent_check`, and the comparison does not run index-to-heap for every index entry. [verify_nbtree.c#locks-and-targets](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L224-L338) [verify_nbtree.c#membership-direction](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920) |
| `dump` | Runs a full-database custom-format, uncompressed logical dump with an initial-lock limit and an external whole-job deadline. The runner gives it a `.partial` filename and renames it only after success. PostgreSQL's dump transaction is read-only and repeatable-read. [pg_dump.c#dump-snapshot](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1166-L1187) | No restore is performed. `pg_dump` disables SQL statement/lock timeouts; its `--lock-wait-timeout` applies to the initial LOCK loop, not total export time. Materialized views are represented by refresh operations, and foreign-table data is skipped. [pg_dump.c#timeout-resets](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1147) [pg_dump.c#initial-lock-timeout](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6536-L6548) [pg_dump.c#reset-after-locks](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6679-L6682) [pg_dump.c#data-object-selection](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L2342-L2368) |

For complete local catalog scans, use an appropriately authorized DBA on the controlled target. A monitoring role alone is not blanket permission to read every relation. `pg_stat_file(text, boolean)` is revoked from `PUBLIC`, and the `amcheck` overload is also not public by default. The runner does not hide unreadable objects behind privilege filters in its scan inventory: their reads fail and are reported. `row_security = off` causes an error when a policy would filter the current role; it does **not** grant that role a bypass. [system_views.sql#file-function-revokes](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1334-L1338) [amcheck--1.0--1.1.sql#privileges](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0--1.1.sql#L27-L29) [rls.c#bypass-and-error](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132)

Run against a quiescent, controlled copy where possible. Each psql check has its own connection/snapshot; an invocation is not an atomic inventory of a changing database. Concurrent rename, drop, rewrite, extension relocation or settings changes can invalidate enumeration, and objects created after enumeration may not be checked. Reconcile such failures and repeat the affected stages after stabilizing the target. New connections also do not empty shared buffers. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795)

### Interpret findings without declaring false health

| Observation | Correct interpretation |
|---|---|
| Schema absent, catalog-link query empty | No matching namespace or selected broken references were found now. The committed drop removes dependency records, so the vanished cascade set cannot be reconstructed from that absence alone. [dependency.c#cleanup](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1269-L1309) |
| Invalid index or `NOT VALID` constraint | Review its lifecycle and baseline. These are catalog state flags, not data checksums. Conversely, a `true` flag is not an on-demand integrity test. [pg_index.h#flags](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L35-L44) [pg_constraint.h#validation-flag](../../../../raw/postgres-12/src/include/catalog/pg_constraint.h#L49-L54) |
| Zero-byte surviving heap and `relpages = 0` | Could be legitimately empty or lost data with no remaining size witness. The file stage intentionally does not label every empty relation corrupt. A bounded scan cannot manufacture evidence for its old end. [pg_class.h#stored-estimates](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66) [md.c#size-discovery](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755) |
| COPY/SELECT/dump succeeds | The chosen read path succeeded on currently visible data. A truncated last segment may simply lower the scan bound; cached pages and [all-zero pages](../../../glossary.md#all-zero-page) add other blind spots. [heapam.c#scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L219-L231) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [bufpage.c#all-zero-acceptance](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) |
| `bt_index_check(index, true)` succeeds | Qualifying heap tuples found by its scan have no detected membership failure. The Bloom membership test is probabilistic: “possibly present” is not exact set equality. Extra index entries for disappeared heap tuples are not a reverse-check failure. Even the parent checker does not reverse the callback direction. [bloomfilter.c#membership-contract](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L3-L16) [verify_nbtree.c#parent-options](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L195-L215) [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920) |
| No checksum errors | With [data checksums](../../../glossary.md#data-checksums) off, that comparison is not performed. With them on, readable initialized pages can be compared, but missing historical extents are not reconstructed and all-zero pages are accepted. Offline checking also stops at EOF. [bufpage.c#verification](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L92-L142) [pg_checksums.c#EOF](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L192-L224) |
| Permission error, timeout, unsupported relation or failed connection | Incomplete diagnostics, not proof of corruption and not a pass. Preserve the error, identify the blocker/prerequisite and rerun deliberately. [rls.c#policy-error](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L120-L129) [verify_nbtree.c#unsupported-targets](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L315-L338) [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396) |

### Settings, defaults and apply scopes

These choices belong to the runner, not a recommendation to alter the production cluster globally. Their contexts are [GUC contexts](../../../glossary.md#guc-context). Except for the startup-only setting checked below, the runner's explicit settings require **no restart or reload**. The psql checks verify version, database, primary status, read-only state and bypass settings on every connection; the dump gets a separate preflight and a final psql guard, not an atomic settings lock around its own connection.

| Setting | Pinned default → runner behavior | Cause, mechanism and scope |
|---|---|---|
| `default_transaction_read_only` | `off` → `on` in diagnostic `PGOPTIONS` | New diagnostic transactions default to read-only. `PGC_USERSET`, session scope; the scan also explicitly begins `READ ONLY`. This does not mean no physical page changes from reads. [guc.c#read-only](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1534-L1549) [heapam.c#read-pruning](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L391-L394) |
| `row_security` | `on` → `off` | Refuse policy-filtered scans instead of silently presenting a subset as coverage. `PGC_USERSET`, session scope; privileged/owner bypass rules remain separate. [guc.c#row_security](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1572-L1578) [rls.c#policy-handling](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132) |
| `search_path` | `"$user", public` → `pg_catalog` | Keep unqualified diagnostic functions/catalog names on the intended path; target relations and extension functions are separately identifier-quoted and schema-qualified. `PGC_USERSET`, session scope. [guc.c#search_path](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3790-L3797) |
| `statement_timeout`, `lock_timeout` | Both `0` → the positive limits in the usage table | Bound each psql statement and lock wait. Both `PGC_USERSET`, session scope through `PGOPTIONS`; expiration means incomplete coverage. `pg_dump` resets them, so the runner uses its initial-lock option plus an external deadline. [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396) [pg_dump.c#reset](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1147) |
| `enable_seqscan`, `enable_indexscan`, `enable_indexonlyscan`, `enable_bitmapscan` | All `on` → seqscan `on`, other three `off` inside each scan transaction | Discourage planner index paths so a no-filter value scan exercises the heap rather than relying on an index-only path. `PGC_USERSET`, `SET LOCAL`, transaction scope. These are planner choices, not a switch that disables internal catalog or TOAST index lookups. [guc.c#scan-methods](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L884-L921) [tuptoaster.c#internal-index-fetch](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1910-L1935) |
| `zero_damaged_pages`, `ignore_checksum_failure` | Both `off`; **not changed** | The guard refuses `on`, because those options can mask read/page/checksum failures. Both `PGC_SUSET`: changing them through session `SET` requires superuser privileges in 12; this runner does not change them. [guc.c#developer-options](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161) [guc.c#superuser-context-gate](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6859-L6868) [md.c#short-read-override](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L614-L629) |
| `ignore_system_indexes` | `off`; **not changed** | The guard refuses an altered catalog-access regime. `PGC_BACKEND`: only connection startup/new sessions can set it, not a later `SET`. [guc.c#ignore_system_indexes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1784-L1793) |

The runner also makes `heapallindexed = true` explicit rather than relying on the false API default. It does not use the parent checker automatically: that checker takes `ShareLock`, whereas `bt_index_check` takes `AccessShareLock`. A stronger check can be a separate, scheduled investigation, but still cannot prove that an erased heap suffix used to exist. [verify_nbtree.c#API-defaults-and-locks](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L245) [verify_nbtree.c#membership-callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920)

### State changes that affect the evidence

| Event | Before → after | Consequence for the runbook |
|---|---|---|
| External truncation, under the incident assumption | File length changes without the backend's normal drop/buffer-discard protocol | Cached pages may still be returned. Do not call a new connection a cold-disk test. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [smgr.c#explicit-drop-invalidation](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L459-L479) |
| Drop finishes, transaction remains open | Metadata deletion and `atCommit` pending deletes are uncommitted | Verify transaction outcome, not just the command tag. [storage.c#pending-drop](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) |
| Abort | Commit-only delete entries do not remove storage | Rollback does not restore bytes lost before the transaction. The pending-delete mechanism records physical identities and outcome flags, not old file contents. [storage.c#pending-delete-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L461) |
| Commit | Catalog deletion becomes visible; remaining buffers for dropped relations are discarded and file cleanup runs | The drop cannot subsequently scan the removed objects for integrity. [xact.c#commit-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232) [smgr.c#discard-and-unlink](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495) |
| Eligible checkpoint completes | Deferred first-main-file unlink is attempted | A leftover zero-byte main file before this point can be normal. Missing names or filesystem failures require separate interpretation; do not force a checkpoint merely to make an incident directory look clean. [md.c#deferred-main-file](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L217-L261) [sync.c#unlink-cycle](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214) |
| Maintenance/rebuild after the incident | Current relation state is used to derive new metadata or index contents | Preserve independent evidence first. For example, index-building reads through the table access method's current scan; it is not retrieval of the pre-truncation tuples. [tableam.h#table_index_build_scan](../../../../raw/postgres-12/src/include/access/tableam.h#L1512-L1533) [heapam.c#current-scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L219-L231) |

### Recovery and acceptance criteria

Use the following as an **operator acceptance policy**, not a built-in PostgreSQL verdict:

1. The affected files and transaction history have a justified scope, or the unresolved uncertainty is explicitly accepted by the incident owner.
2. Every missing object and cross-schema cascade effect matches the approved loss/recovery plan. Compare definitions, constraints and data against a trusted baseline, not just against the post-drop catalogs.
3. Required checks have completed with their intended privileges. Every candidate, error, warning and exclusion has an explanation and disposition.
4. A logical archive has been restored into a separate empty target and its schema/data/application invariants reviewed. Equal aggregate counts alone are not the acceptance criterion.
5. There is a tested recovery path and an explicit decision to keep the survivors, recover selected objects, or restore/rebuild more broadly. Do not replace evidence collection with `VACUUM`, `REINDEX`, direct catalog edits, filesystem deletion, or `zero_damaged_pages`.

If data or objects must be recovered, select a trusted backup predating the first suspected damage and validate recovery on a separate target. Do not assume that a replica is a pre-drop backup: commit replay processes recorded relation deletions. The backend's WAL-before-delete rule makes its own drop durable; it is not an undo journal for the assumed external shell operation. [xact.c#flush-before-delete](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1359-L1377) [xact.c#replay-deletion](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L5817-L5838) [storage.c#pending-delete-contents](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173)

### Build boundaries and validation

The inventory and COPY portions use core SQL. The catalog layouts are generated from the pinned headers through `genbki.pl`; their `_d.h` files are build artifacts. The script reads `block_size` and `segment_size` rather than hard-code a segment geometry. The B-tree portion is the separately built and explicitly installed `contrib/amcheck` module. Custom table/index access methods, user-defined type output routines and hooks are outside a blanket built-in-code integrity guarantee. [catalog/Makefile#generation](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L51) [catalog/Makefile#genbki](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L103) [tableam.h#AM-dispatch](../../../../raw/postgres-12/src/include/access/tableam.h#L761-L772) [copy.c#type-output-dispatch](../../../../raw/postgres-12/src/backend/commands/copy.c#L2035-L2052) [amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10)

The pinned regression inputs cover schema deletion, COPY with row security, and B-tree checker permissions, invalid targets, optional checks and lock release. They are adjacent tests, not a complete reproduction of this incident. [namespace.sql#schema-tests](../../../../raw/postgres-12/src/test/regress/sql/namespace.sql#L5-L44) [rowsecurity.sql#COPY-tests](../../../../raw/postgres-12/src/test/regress/sql/rowsecurity.sql#L1210-L1311) [check_btree.sql#API-tests](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L1-L83)

The runner was integration-tested on an isolated Linux server built out of tree from the exact 12.2 pin, with readline and zlib disabled to avoid optional build dependencies. Verification exercised the generated SQL, relocated `amcheck`, catalog and value scans, quoted identifiers, inheritance/partition leaves, TOAST-capable values, sequences, materialized views, permission/RLS refusal, missing-extension handling, unsafe-setting refusal, schema-presence findings, lock timeouts, dump-deadline failure, explicit exclusions, and logical export/restore. The embedded copy was extracted using the published command and rerun successfully. This page publishes no measured row counts, sizes or timings; the script is operational diagnostic tooling, not a benchmark or a fault-injection harness. Actual incident files were not supplied or inspected.

### Causal summary

**External file damage does not remove catalog dependencies → explicit cascade can remove objects inside and outside the schema → commit removes metadata and storage without certifying their former contents → current catalogs and scan bounds describe only what survives → layered diagnostics can find inconsistencies, but independent historical evidence and restore/application checks decide whether the survivors are acceptable.** [dependency.c#cascade](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) [storage.c#drop-storage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [smgr.c#unlink](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495) [md.c#scan-size](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755)

### Diagnostic script

This is the only maintained copy. Extract it with the command above and run it from the repository root. The extracted file is a disposable runtime artifact.

```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077

usage() {
    cat <<'USAGE'
Usage: bash .wiki-runtime/pg12-post-drop-health.sh [inventory] [files] [scan] [amcheck] [dump]
       bash .wiki-runtime/pg12-post-drop-health.sh all

Required environment: PGHOST PGPORT PGDATABASE PGUSER DROPPED_SCHEMA.
Use a direct connection to a PostgreSQL 12 primary or a detached restored copy.
PGDATABASE must be a database name, not a connection string or URI.
Authentication uses libpq (for example, PGPASSFILE); no password argument.
Run from the repository root.

Optional environment:
  PG12_BINDIR                    directory containing PostgreSQL 12 clients
  PGCONNECT_TIMEOUT              connection timeout in seconds (default 10)
  HEALTH_STATEMENT_TIMEOUT_MS     inventory/files statement limit (default 30000)
  HEALTH_SCAN_TIMEOUT_MS          per-relation scan/amcheck limit (default 300000)
  HEALTH_LOCK_TIMEOUT_MS          per-lock wait limit (default 2000)
  HEALTH_DUMP_TIMEOUT_SECONDS     whole dump deadline (default 900)

PGOPTIONS, PGSERVICE, PGSERVICEFILE and PGHOSTADDR are replaced/disabled.
Other libpq authentication and TLS settings are inherited.
Output: a new private .wiki-runtime/pg12-health-TIMESTAMP-PID directory.
Read summary.tsv, then its named CSV/error files. Scan row data is discarded.
The dump stage requires pg_dump 12 and GNU timeout; it saves database data.

Exit 0: requested checks completed without an automatic finding.
Exit 1: a query/tool failed, or a finding needs review.
Exit 2: a prerequisite/guard failed, or requested coverage was skipped.
Exit 130/143: interrupted. An incomplete run is never a health certificate.
Even exit 0 cannot prove completeness of rows, objects, or files.
USAGE
}

case "${1:-}" in
    -h|--help) usage; exit 0 ;;
esac

want_files=false
want_scan=false
want_amcheck=false
want_dump=false
for stage in "$@"; do
    case "$stage" in
        inventory) ;;
        files) want_files=true ;;
        scan) want_scan=true ;;
        amcheck) want_amcheck=true ;;
        dump) want_dump=true ;;
        all) want_files=true; want_scan=true; want_amcheck=true; want_dump=true ;;
        *) usage >&2; exit 2 ;;
    esac
done

for required in PGHOST PGPORT PGDATABASE PGUSER DROPPED_SCHEMA; do
    if [[ -z "${!required:-}" ]]; then
        printf 'Required environment variable is empty: %s\n' "$required" >&2
        exit 2
    fi
done
case "$PGDATABASE" in
    *=*|postgresql://*|postgres://*)
        printf 'PGDATABASE must be a literal database name, not a connection string.\n' >&2
        exit 2 ;;
esac
case "$PGHOST" in
    *,*) printf 'Use one explicit host/socket, not a host failover list.\n' >&2; exit 2 ;;
esac

meta_ms=${HEALTH_STATEMENT_TIMEOUT_MS:-30000}
scan_ms=${HEALTH_SCAN_TIMEOUT_MS:-300000}
lock_ms=${HEALTH_LOCK_TIMEOUT_MS:-2000}
dump_seconds=${HEALTH_DUMP_TIMEOUT_SECONDS:-900}
connect_seconds=${PGCONNECT_TIMEOUT:-10}
for value in "$PGPORT" "$meta_ms" "$scan_ms" "$lock_ms" "$dump_seconds" "$connect_seconds"; do
    if [[ ! "$value" =~ ^[1-9][0-9]{0,8}$ ]]; then
        printf 'Ports and timeouts must be positive decimal integers (at most nine digits).\n' >&2
        exit 2
    fi
done
if (( PGPORT > 65535 )); then
    printf 'PGPORT must be at most 65535.\n' >&2
    exit 2
fi

repo_root=$(pwd -P)
if [[ ! -f "$repo_root/AGENTS.md" || ! -d "$repo_root/wiki/v12" ||
      ! -d "$repo_root/.wiki-runtime" ]]; then
    printf 'Run from the repository root, with .wiki-runtime present.\n' >&2
    exit 2
fi
psql_bin=${PG12_BINDIR:+$PG12_BINDIR/}psql
dump_bin=${PG12_BINDIR:+$PG12_BINDIR/}pg_dump
if ! client_version=$("$psql_bin" --version 2>&1) ||
   [[ ! "$client_version" =~ PostgreSQL\)\ 12\. ]]; then
    printf 'psql 12 is required; select it with PG12_BINDIR.\n' >&2
    exit 2
fi

out="$repo_root/.wiki-runtime/pg12-health-$(date -u +%Y%m%dT%H%M%SZ)-$$"
mkdir -- "$out"
printf 'status\tcheck\tdetail\n' > "$out/summary.tsv"
printf 'Report: %s\n' "$out"
errors=0
gaps=0
completed=false
export PGCONNECT_TIMEOUT="$connect_seconds"
export PGAPPNAME=wiki_pg12_post_drop_health
unset PGSERVICE PGSERVICEFILE PGHOSTADDR

record() {
    printf '%s\t%s\t%s\n' "$1" "$2" "$3" >> "$out/summary.tsv"
    case "$1" in
        ERROR|REVIEW) errors=$((errors + 1)) ;;
        SKIP) gaps=$((gaps + 1)) ;;
    esac
}

finish() {
    local rc=$?
    if [[ "$completed" != true ]]; then
        printf 'INCOMPLETE\trun\tStopped before the final summary; exit=%s\n' "$rc" >> "$out/summary.tsv"
    fi
    printf 'No automatic health verdict. Review %s/summary.tsv and the checklist.\n' "$out" >&2
}
trap finish EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

psql_args=(-X -w -q -v ON_ERROR_STOP=1 -v VERBOSITY=verbose -P pager=off
    --host="$PGHOST" --port="$PGPORT" --username="$PGUSER" --dbname="$PGDATABASE"
    -v "dropped_schema=$DROPPED_SCHEMA" -v "expected_database=$PGDATABASE")

sql() {
    local limit=$1
    shift
    {
        cat <<'SQL'
SELECT /* wiki_pg12_health_guard */
       (CASE WHEN current_setting('server_version_num')::integer BETWEEN 120000 AND 129999
                   AND current_database() = :'expected_database'
                   AND NOT pg_is_in_recovery()
                   AND current_setting('zero_damaged_pages') = 'off'
                   AND current_setting('ignore_checksum_failure') = 'off'
                   AND current_setting('ignore_system_indexes') = 'off'
                   AND current_setting('transaction_read_only') = 'on'
             THEN 'true'
             ELSE 'STOP: require the named PostgreSQL 12 primary, read-only SQL, and all three corruption-bypass settings off'
        END)::boolean AS health_guard
\gset
SQL
        cat
    } | PGOPTIONS="-c default_transaction_read_only=on -c row_security=off -c search_path=pg_catalog -c statement_timeout=$limit -c lock_timeout=$lock_ms" \
        "$psql_bin" "${psql_args[@]}" "$@" -f -
}

report() {
    local label=$1 mode=$2 rc
    if sql "$meta_ms" --csv > "$out/$label.csv" 2> "$out/$label.stderr"; then
        if [[ "$mode" == findings ]] && (( $(wc -l < "$out/$label.csv") > 1 )); then
            record REVIEW "$label" "$label.csv; candidates, not a corruption verdict"
        else
            record OK "$label" "$label.csv; review $label.stderr as well"
        fi
    else
        rc=$?
        record ERROR "$label" "exit=$rc; $label.stderr; CSV may be partial"
        if (( rc == 2 )); then
            exit 1
        fi
    fi
}

ids() {
    local label=$1 rc
    if sql "$meta_ms" -At > "$out/$label.ids" 2> "$out/$label.stderr"; then
        if grep -Ev '^[0-9]+$' "$out/$label.ids" > "$out/$label.invalid"; then
            record ERROR "$label" "Unexpected non-OID output; see $label.invalid"
            return 1
        fi
        record OK "$label" "$label.ids; enumeration completed"
    else
        rc=$?
        record ERROR "$label" "exit=$rc; $label.stderr; enumeration incomplete"
        if (( rc == 2 )); then
            exit 1
        fi
        return 1
    fi
}

if ! sql "$meta_ms" -At > "$out/preflight.txt" 2> "$out/preflight.stderr" <<'SQL'
SELECT /* wiki_pg12_health_preflight */ version(), current_database(), current_user;
SQL
then
    record SKIP preflight "Connection/version/safety guard failed; preflight.stderr"
    exit 2
fi
record OK preflight "preflight.txt; source review is pinned to 12.2, actual server recorded here"

report environment inventory <<'SQL'
SELECT /* wiki_pg12_health_environment */
       clock_timestamp() AS captured_at, version(), current_database(), current_user,
       inet_server_addr(), inet_server_port(), pg_postmaster_start_time(),
       pg_is_in_recovery(), current_setting('data_checksums') AS data_checksums,
       current_setting('block_size') AS block_size,
       current_setting('segment_size') AS segment_size;
SQL
report settings inventory <<'SQL'
SELECT /* wiki_pg12_health_settings */ name, setting, unit, context, source
FROM pg_settings
WHERE name IN ('data_checksums', 'zero_damaged_pages', 'ignore_checksum_failure',
               'ignore_system_indexes', 'default_transaction_read_only',
               'transaction_read_only', 'row_security', 'search_path',
               'statement_timeout', 'lock_timeout', 'track_counts',
               'log_min_messages', 'log_min_error_statement', 'logging_collector')
ORDER BY name;
SQL
report schemas inventory <<'SQL'
SELECT /* wiki_pg12_health_schemas */ oid, nspname, nspowner,
       nspname = :'dropped_schema' AS matches_requested_schema
FROM pg_namespace ORDER BY oid;
SQL
report schema_still_present findings <<'SQL'
SELECT /* wiki_pg12_health_schema_presence */ oid, nspname,
       'Verify commit, database, spelling, and any later schema recreation' AS action
FROM pg_namespace WHERE nspname = :'dropped_schema';
SQL
report relations inventory <<'SQL'
SELECT /* wiki_pg12_health_relations */
       c.oid, n.nspname, c.relname, c.relkind, c.relpersistence, c.relisshared,
       c.relam, a.amname, c.relfilenode, pg_relation_filepath(c.oid) AS current_path,
       c.relpages, c.reltuples, c.reltoastrelid, c.relispopulated,
       c.relrowsecurity, c.relforcerowsecurity
FROM pg_class c
LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN pg_am a ON a.oid = c.relam
ORDER BY c.oid;
SQL
report extensions inventory <<'SQL'
SELECT /* wiki_pg12_health_extensions */ e.oid, e.extname, e.extversion, n.nspname
FROM pg_extension e LEFT JOIN pg_namespace n ON n.oid = e.extnamespace
ORDER BY e.oid;
SQL
report constraints inventory <<'SQL'
SELECT /* wiki_pg12_health_constraints */
       oid, conname, connamespace, contype, conrelid, confrelid,
       conindid, convalidated, pg_get_constraintdef(oid) AS definition
FROM pg_constraint ORDER BY oid;
SQL
report index_state findings <<'SQL'
SELECT /* wiki_pg12_health_index_state */
       indexrelid, indrelid, indisvalid, indisready, indislive,
       'Can be pre-existing or concurrent DDL; compare with baseline' AS qualification
FROM pg_index WHERE NOT (indisvalid AND indisready AND indislive)
ORDER BY indexrelid;
SQL
report constraint_state findings <<'SQL'
SELECT /* wiki_pg12_health_constraint_state */ oid, conname, conrelid, contype,
       'NOT VALID is legitimate; this flag does not recheck existing rows' AS qualification
FROM pg_constraint WHERE NOT convalidated ORDER BY oid;
SQL
report triggers inventory <<'SQL'
SELECT /* wiki_pg12_health_triggers */ oid, tgrelid, tgname, tgenabled,
       tgisinternal, tgconstraint
FROM pg_trigger ORDER BY oid;
SQL
report catalog_links findings <<'SQL'
WITH /* wiki_pg12_health_catalog_links */
attrs AS (
    SELECT attrelid, count(*) AS n, min(attnum) AS lo, max(attnum) AS hi
    FROM pg_attribute WHERE attnum > 0 GROUP BY attrelid
), endpoints AS (
    SELECT classid, objid FROM pg_depend WHERE deptype <> 'p'
    UNION
    SELECT refclassid, refobjid FROM pg_depend
), known_objects AS (
    SELECT 'pg_class'::regclass::oid AS classid, oid FROM pg_class
    UNION ALL SELECT 'pg_namespace'::regclass::oid, oid FROM pg_namespace
    UNION ALL SELECT 'pg_type'::regclass::oid, oid FROM pg_type
    UNION ALL SELECT 'pg_proc'::regclass::oid, oid FROM pg_proc
    UNION ALL SELECT 'pg_constraint'::regclass::oid, oid FROM pg_constraint
), findings AS (
    SELECT 'relation_namespace_missing' AS issue, c.oid AS object_oid
    FROM pg_class c LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE n.oid IS NULL
    UNION ALL
    SELECT 'toast_target_missing_or_wrong_kind', c.oid
    FROM pg_class c LEFT JOIN pg_class t ON t.oid = c.reltoastrelid
    WHERE c.reltoastrelid <> 0 AND (t.oid IS NULL OR t.relkind <> 't')
    UNION ALL
    SELECT 'attribute_numbering_mismatch', c.oid
    FROM pg_class c LEFT JOIN attrs a ON a.attrelid = c.oid
    WHERE coalesce(a.n, 0) <> c.relnatts
       OR (c.relnatts > 0 AND (a.lo <> 1 OR a.hi <> c.relnatts))
    UNION ALL
    SELECT 'attribute_relation_missing', a.attrelid
    FROM pg_attribute a LEFT JOIN pg_class c ON c.oid = a.attrelid
    WHERE c.oid IS NULL
    UNION ALL
    SELECT 'index_relation_or_table_missing', i.indexrelid
    FROM pg_index i
    LEFT JOIN pg_class c ON c.oid = i.indexrelid
    LEFT JOIN pg_class t ON t.oid = i.indrelid
    WHERE c.oid IS NULL OR t.oid IS NULL OR c.relkind NOT IN ('i', 'I')
    UNION ALL
    SELECT 'index_metadata_missing', c.oid
    FROM pg_class c LEFT JOIN pg_index i ON i.indexrelid = c.oid
    WHERE c.relkind IN ('i', 'I') AND i.indexrelid IS NULL
    UNION ALL
    SELECT 'constraint_target_missing', k.oid
    FROM pg_constraint k
    LEFT JOIN pg_class r ON r.oid = k.conrelid
    LEFT JOIN pg_class f ON f.oid = k.confrelid
    LEFT JOIN pg_class i ON i.oid = k.conindid
    LEFT JOIN pg_type t ON t.oid = k.contypid
    LEFT JOIN pg_namespace n ON n.oid = k.connamespace
    LEFT JOIN pg_constraint p ON p.oid = k.conparentid
    WHERE (k.conrelid <> 0 AND r.oid IS NULL)
       OR (k.confrelid <> 0 AND f.oid IS NULL)
       OR (k.conindid <> 0 AND i.oid IS NULL)
       OR (k.contypid <> 0 AND t.oid IS NULL)
       OR (k.conparentid <> 0 AND p.oid IS NULL) OR n.oid IS NULL
    UNION ALL
    SELECT 'inheritance_target_missing', h.inhrelid
    FROM pg_inherits h
    LEFT JOIN pg_class c ON c.oid = h.inhrelid
    LEFT JOIN pg_class p ON p.oid = h.inhparent
    WHERE c.oid IS NULL OR p.oid IS NULL
    UNION ALL
    SELECT 'trigger_relation_missing', t.oid
    FROM pg_trigger t LEFT JOIN pg_class c ON c.oid = t.tgrelid
    WHERE c.oid IS NULL
    UNION ALL
    SELECT 'dependency_endpoint_missing_in_catalog_' || e.classid, e.objid
    FROM endpoints e
    LEFT JOIN known_objects o ON o.classid = e.classid AND o.oid = e.objid
    WHERE e.classid IN ('pg_class'::regclass, 'pg_namespace'::regclass,
                       'pg_type'::regclass, 'pg_proc'::regclass, 'pg_constraint'::regclass)
      AND o.oid IS NULL
)
SELECT DISTINCT issue, object_oid FROM findings ORDER BY issue, object_oid;
SQL
report activity inventory <<'SQL'
SELECT /* wiki_pg12_health_activity */ pid, usename, application_name, backend_type,
       state, xact_start, wait_event_type, wait_event, pg_blocking_pids(pid) AS blockers
FROM pg_stat_activity WHERE datname = current_database() ORDER BY pid;
SQL
report prepared_transactions inventory <<'SQL'
SELECT /* wiki_pg12_health_prepared */ transaction, gid, prepared, owner, database
FROM pg_prepared_xacts WHERE database = current_database() ORDER BY prepared;
SQL
report database_statistics inventory <<'SQL'
SELECT /* wiki_pg12_health_database_statistics */ *
FROM pg_stat_database WHERE datname = current_database();
SQL

if [[ "$want_files" == true ]]; then
    if ids file_targets <<'SQL'
SELECT /* wiki_pg12_health_file_targets */ oid FROM pg_class
WHERE relkind IN ('r', 't', 'm', 'i', 'S') AND relpersistence <> 't' ORDER BY oid;
SQL
    then
        while IFS= read -r oid; do
            report "files_$oid" findings <<SQL
SELECT /* wiki_pg12_health_file_identity */ oid AS target_oid
FROM pg_class WHERE oid = '$oid'::oid
\\gset
WITH /* wiki_pg12_health_files */ RECURSIVE
g AS (
    SELECT current_setting('block_size')::bigint AS bs,
           pg_size_bytes(current_setting('segment_size')) AS segbytes
), r AS (
    SELECT oid, pg_relation_filepath(oid) AS path, relpages::bigint AS relpages
    FROM pg_class WHERE oid = :'target_oid'::oid
), s(segno, bytes) AS (
    SELECT 0::bigint, (pg_stat_file(r.path, true)).size FROM r
    UNION ALL
    SELECT s.segno + 1, (pg_stat_file(r.path || '.' || (s.segno + 1), true)).size
    FROM s CROSS JOIN r WHERE s.bytes IS NOT NULL
), first_short AS (
    SELECT min(segno) FILTER (WHERE bytes < g.segbytes) AS segno
    FROM s CROSS JOIN g
), totals AS (
    SELECT coalesce(sum(s.bytes) FILTER
               (WHERE f.segno IS NULL OR s.segno <= f.segno), 0) AS logical_bytes,
           bool_or(s.segno = 0 AND s.bytes IS NULL) AS main_missing,
           bool_or(s.bytes % g.bs <> 0) AS partial_page,
           bool_or(s.bytes > g.segbytes) AS oversized,
           bool_or(s.segno > f.segno AND s.bytes > 0) AS nonzero_after_short
    FROM s CROSS JOIN first_short f CROSS JOIN g
)
SELECT r.oid, r.path, t.logical_bytes, r.relpages, f.issue
FROM r CROSS JOIN totals t CROSS JOIN g
CROSS JOIN LATERAL (VALUES
    ('missing_main_file', t.main_missing),
    ('partial_page_at_file_end', t.partial_page),
    ('segment_exceeds_build_limit', t.oversized),
    ('nonzero_segment_after_short_segment', t.nonzero_after_short),
    ('stored_relpages_exceeds_current_extent', r.relpages > t.logical_bytes / g.bs)
) f(issue, flagged)
WHERE f.flagged ORDER BY f.issue;
SQL
        done < "$out/file_targets.ids"
    fi
else
    record NOT_REQUESTED files "No filesystem extent screen"
fi

if [[ "$want_scan" == true ]]; then
    report scan_exclusions inventory <<'SQL'
SELECT /* wiki_pg12_health_scan_exclusions */ c.oid, c.relkind, c.relpersistence,
       a.amname,
       CASE WHEN c.relpersistence = 't' THEN 'temporary relation of another session'
            WHEN c.relkind = 'f' THEN 'foreign data is outside local scan coverage'
            WHEN c.relkind = 'm' AND NOT c.relispopulated THEN 'unpopulated materialized view'
            ELSE 'table access method is outside the built-in heap scope' END AS reason
FROM pg_class c LEFT JOIN pg_am a ON a.oid = c.relam
WHERE c.relkind IN ('r', 't', 'm', 'S', 'f')
  AND (c.relpersistence = 't' OR c.relkind = 'f'
       OR (c.relkind = 'm' AND NOT c.relispopulated)
       OR (c.relkind <> 'S' AND a.amname IS DISTINCT FROM 'heap'))
ORDER BY c.oid;
SQL
    if [[ -f "$out/scan_exclusions.csv" ]] && (( $(wc -l < "$out/scan_exclusions.csv") > 1 )); then
        record SKIP scan_exclusions "scan_exclusions.csv; reconcile every exclusion"
    fi
    if ids scan_targets <<'SQL'
SELECT /* wiki_pg12_health_scan_targets */ c.oid
FROM pg_class c LEFT JOIN pg_am a ON a.oid = c.relam
WHERE c.relpersistence <> 't'
  AND ((c.relkind IN ('r', 't', 'm') AND a.amname = 'heap' AND c.relispopulated)
       OR c.relkind = 'S')
ORDER BY c.oid;
SQL
    then
        while IFS= read -r oid; do
            if sql "$scan_ms" -v "target_oid=$oid" > /dev/null 2> "$out/scan_$oid.stderr" <<'SQL'
BEGIN /* wiki_pg12_health_scan */ READ ONLY;
SET /* wiki_pg12_health_scan */ LOCAL enable_seqscan = on;
SET /* wiki_pg12_health_scan */ LOCAL enable_indexscan = off;
SET /* wiki_pg12_health_scan */ LOCAL enable_indexonlyscan = off;
SET /* wiki_pg12_health_scan */ LOCAL enable_bitmapscan = off;
SELECT /* wiki_pg12_health_scan */
       format('COPY /* wiki_pg12_health_scan */ (SELECT * FROM ONLY %I.%I) TO STDOUT',
              n.nspname, c.relname) AS health_copy_sql
FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.oid = :'target_oid'::oid
\gset
SELECT /* wiki_pg12_health_scan_dispatch */ :'health_copy_sql'
\gexec
COMMIT /* wiki_pg12_health_scan */;
SQL
            then
                record OK "scan_$oid" "Visible values serialized; row data discarded; scan_$oid.stderr"
            else
                rc=$?
                record ERROR "scan_$oid" "exit=$rc; scan_$oid.stderr; scan incomplete"
                if (( rc == 2 )); then exit 1; fi
            fi
        done < "$out/scan_targets.ids"
    fi
else
    record NOT_REQUESTED scan "No per-relation value scan"
fi

amcheck_lookup=$(cat <<'SQL'
FROM pg_extension e
JOIN pg_depend d ON d.refclassid = 'pg_extension'::regclass
                 AND d.refobjid = e.oid AND d.deptype = 'e'
                 AND d.classid = 'pg_proc'::regclass
JOIN pg_proc p ON p.oid = d.objid
JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE e.extname = 'amcheck' AND p.proname = 'bt_index_check' AND p.pronargs = 2
  AND p.proargtypes[0] = 'regclass'::regtype AND p.proargtypes[1] = 'bool'::regtype
  AND has_function_privilege(p.oid, 'EXECUTE')
SQL
)
if [[ "$want_amcheck" == true ]]; then
    if available=$(sql "$meta_ms" -At 2> "$out/amcheck_available.stderr" <<SQL
SELECT /* wiki_pg12_health_amcheck_available */ EXISTS (SELECT 1 $amcheck_lookup);
SQL
    ) && [[ "$available" == t ]]; then
        report amcheck_exclusions inventory <<'SQL'
SELECT /* wiki_pg12_health_amcheck_exclusions */
       c.oid, a.amname, ta.amname AS table_am, c.relpersistence,
       i.indisvalid, i.indisready, i.indislive
FROM pg_class c LEFT JOIN pg_am a ON a.oid = c.relam
LEFT JOIN pg_index i ON i.indexrelid = c.oid
LEFT JOIN pg_class t ON t.oid = i.indrelid
LEFT JOIN pg_am ta ON ta.oid = t.relam
WHERE c.relkind = 'i'
  AND (c.relpersistence = 't' OR a.amname IS DISTINCT FROM 'btree'
       OR ta.amname IS DISTINCT FROM 'heap'
       OR NOT coalesce(i.indisvalid AND i.indisready AND i.indislive, false))
ORDER BY c.oid;
SQL
        if (( $(wc -l < "$out/amcheck_exclusions.csv") > 1 )); then
            record SKIP amcheck_exclusions "amcheck_exclusions.csv; these indexes were not checked"
        fi
        if ids amcheck_targets <<'SQL'
SELECT /* wiki_pg12_health_amcheck_targets */ c.oid
FROM pg_class c JOIN pg_index i ON i.indexrelid = c.oid
JOIN pg_am a ON a.oid = c.relam
JOIN pg_class t ON t.oid = i.indrelid
JOIN pg_am ta ON ta.oid = t.relam
WHERE c.relkind = 'i' AND c.relpersistence <> 't' AND a.amname = 'btree'
  AND ta.amname = 'heap'
  AND i.indisvalid AND i.indisready AND i.indislive ORDER BY c.oid;
SQL
        then
            while IFS= read -r oid; do
                if sql "$scan_ms" -v "target_oid=$oid" > "$out/amcheck_$oid.txt" 2> "$out/amcheck_$oid.stderr" <<SQL
SELECT /* wiki_pg12_health_amcheck_function */ format('%I.%I', n.nspname, p.proname) AS checker
$amcheck_lookup
\\gset
SELECT /* wiki_pg12_health_amcheck */ :checker(:'target_oid'::oid::regclass, true);
SQL
                then
                    record OK "amcheck_$oid" "B-tree and one-way heap membership check completed; amcheck_$oid.stderr"
                else
                    rc=$?
                    record ERROR "amcheck_$oid" "exit=$rc; amcheck_$oid.stderr; check incomplete"
                    if (( rc == 2 )); then exit 1; fi
                fi
            done < "$out/amcheck_targets.ids"
        fi
    else
        record SKIP amcheck "Extension-owned bt_index_check(regclass,boolean) unavailable or not executable; amcheck_available.stderr"
    fi
else
    record NOT_REQUESTED amcheck "No B-tree integrity check"
fi

if [[ "$want_dump" == true ]]; then
    if dump_version=$("$dump_bin" --version 2>&1) &&
       [[ "$dump_version" =~ PostgreSQL\)\ 12\. ]] &&
       timeout_version=$(timeout --version 2>&1) &&
       [[ "$timeout_version" == *"GNU coreutils"* ]]; then
        if ! sql "$meta_ms" -At > "$out/dump_guard.txt" 2> "$out/dump_guard.stderr" <<'SQL'
SELECT /* wiki_pg12_health_dump_guard */ clock_timestamp();
SQL
        then
            record SKIP dump "Safety guard failed immediately before dump; dump_guard.stderr"
        elif PGOPTIONS="-c default_transaction_read_only=on -c row_security=off -c search_path=pg_catalog" \
            timeout --signal=TERM --kill-after=10s "${dump_seconds}s" \
            "$dump_bin" --no-password --host="$PGHOST" --port="$PGPORT" \
            --username="$PGUSER" --dbname="$PGDATABASE" --format=custom --compress=0 \
            --lock-wait-timeout="${lock_ms}ms" --file="$out/database.dump.partial" \
            > "$out/dump.stdout" 2> "$out/dump.stderr"; then
            mv -- "$out/database.dump.partial" "$out/database.dump"
            record OK dump "database.dump; restore-test separately; inspect dump.stderr"
        else
            rc=$?
            record ERROR dump "exit=$rc; dump.stderr; .partial is not a successful archive"
        fi
    else
        record SKIP dump "Requires pg_dump 12 and GNU timeout"
    fi
else
    record NOT_REQUESTED dump "No logical archive or restore test"
fi

if sql "$meta_ms" -At > "$out/final_guard.txt" 2> "$out/final_guard.stderr" <<'SQL'
SELECT /* wiki_pg12_health_final_guard */ clock_timestamp();
SQL
then
    record OK final_guard "final_guard.txt; endpoint/settings guard still holds"
else
    record ERROR final_guard "final_guard.stderr; final connection or guard failed"
fi
record NOTE manual "Compare pre-incident inventory/data; review logs, backups, file mapping, non-B-tree indexes, and restore results"
record NOTE coverage "Catalog-derived, per-connection observations; no proof of lost rows/objects or absence of unlisted files"
completed=true
if (( errors > 0 )); then exit 1; fi
if (( gaps > 0 )); then exit 2; fi
exit 0
```

## Context Reviewed

- Target pin `45b88269a353ad93744772791feb6d01bc7e1e42`; manifests, indexes and recent log entries used only for navigation.
- Pre-draft claim/causal inventory: namespace versus physical identity; dependency/commit boundaries; current versus stored/historical extent; buffer versus disk reads; checker membership direction; visible-value versus completeness checks; timeouts, privileges and exclusions.
- Callers/callees and structures: `performMultipleDeletions`, `RelationDropStorage`, `PendingRelDelete`, `smgrDoPendingDeletes`, `smgrdounlinkall`, `mdunlinkfork`, `SyncPostCheckpoint`, catalog headers, `mdnblocks`, the heap scan bound, COPY output dispatch, TOAST indexed chunk fetch, B-tree checker setup/callback and pg_dump setup/data selection.
- SQL surface: catalog columns, file functions and grants, extension-owned overload discovery, settings and scopes, COPY query syntax, psql variable quoting/execution and error-stop behavior; build-generated catalog headers and contrib separation.
- Comparison set: this page and runner, linked glossary entries, matching-version zero-file/drop/amcheck/last-segment/log/health pages, the new index entries and v12 manifest row. No matching v12 common-concept pages exist.

## Evidence Map

| Claim group | Primary evidence |
|---|---|
| Successful cascade is deletion, not integrity validation | [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437), [heap.c#storage-scheduling](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1943-L1959), [index.c#index-storage](../../../../raw/postgres-12/src/backend/catalog/index.c#L2199-L2203) |
| Current catalogs lose the old dependency inventory | [dependency.c#record-removal](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1269-L1309), [schemacmds.c#namespace-removal](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L218-L234) |
| Catalog checks are identifier/state screens, not historical validation | [pg_class.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L29-L107), [pg_depend.h#endpoints](../../../../raw/postgres-12/src/include/catalog/pg_depend.h#L41-L64), [pg_index.h#flags](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L31-L44), [pg_constraint.h#references](../../../../raw/postgres-12/src/include/catalog/pg_constraint.h#L49-L95) |
| File extent and stored estimate have different origins | [genfile.c#stat](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L362-L427), [md.c#size](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755), [pg_class.h#estimates](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L59-L66) |
| Reads can succeed after suffix loss; buffers are another boundary | [heapam.c#bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L219-L231), [bufmgr.c#hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) |
| Value serialization exercises more than row counting, but not all physical state | [copy.c#output](../../../../raw/postgres-12/src/backend/commands/copy.c#L2035-L2052), [copy.c#dispatch](../../../../raw/postgres-12/src/backend/commands/copy.c#L2202-L2213), [tuptoaster.c#fetch](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1910-L2016) |
| B-tree-only, optional one-way heap membership | [verify_nbtree.c#API](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L245), [verify_nbtree.c#targets](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L315-L338), [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920) |
| Dump SQL timeouts do not bound the export | [pg_dump.c#timeout-reset](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1147), [pg_dump.c#lock-window](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6536-L6548), [pg_dump.c#reset-after-locks](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6679-L6682) |
| Offline checksums are supplemental and can miss absent data | [pg_checksums.c#EOF-and-new-pages](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L192-L224), [pg_checksums.c#preconditions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L490-L527) |

## Open Questions

- The exact affected paths, incident timing, commit evidence, live-server activity, actual server minor/build, backup quality and required application invariants are unknown. No health verdict for the user's database is claimed.
- The shell truncation interpretation is an explicit input assumption. A different operation, catalog damage, shared-state damage, or an incomplete file inventory requires a revised incident scope.
- Tests validate the diagnostic implementation and selected failure handling, not exhaustive corruption detection. No destructive truncation/drop incident was injected for this page. Non-B-tree integrity, custom access methods/output code, all dependency classes/subobjects, auxiliary forks, unknown orphan files, unreachable/dead data and concurrent DDL require separate investigation.
- **Outside-scope contradiction:** the existing [general health checklist, checksum row](database-health-checklist.md#observability-prerequisites) says offline `pg_checksums` “verifies every file.” This page instead states its bounded scope. The pin supports the qualification: there is an explicit excluded-file list, temporary entries are skipped, EOF ends each scan and `PageIsNew` pages bypass comparison. The other page was not changed. [pg_checksums.c#excluded-files](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L94-L109) [pg_checksums.c#temporary-exclusions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L297-L327) [pg_checksums.c#page-scan](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L192-L224)
- PostgreSQL 12 has no common-concept pages for schema dependency deletion or relation-file deletion/integrity boundaries. These would be useful separate tasks; this answer did not create or edit concept pages.

## Source References

- [dependency.c#cascade](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437), [dependency.c#cleanup](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1269-L1309), [gram.y#default-drop-behavior](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568), [drop_schema.sgml#CASCADE](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L65-L95).
- [heap.c#drop](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1971), [index.c#drop-storage](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2209), [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L214-L235), [pg_constraint.c#dependencies](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L233-L299).
- [storage.c#pending-drop](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173), [storage.c#pending-deletion](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L461), [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495), [md.c#unlink](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L217-L359), [sync.c#SyncPostCheckpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214).
- [xact.c#flush-before-delete](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1359-L1377), [xact.c#commit-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232), [xact.c#replay](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L5817-L5838).
- [pg_namespace.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L24-L44), [pg_class.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L29-L107), [pg_attribute.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_attribute.h#L37-L75), [pg_depend.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_depend.h#L41-L64), [pg_index.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L58), [pg_constraint.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_constraint.h#L49-L95), [pg_inherits.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_inherits.h#L32-L37), [pg_trigger.h#layout](../../../../raw/postgres-12/src/include/catalog/pg_trigger.h#L34-L49).
- [pg_proc.h#function-identity](../../../../raw/postgres-12/src/include/catalog/pg_proc.h#L31-L45), [bloomfilter.c#membership-contract](../../../../raw/postgres-12/src/backend/lib/bloomfilter.c#L3-L16).
- [relpath.c#GetRelationPath](../../../../raw/postgres-12/src/common/relpath.c#L129-L205), [dbsize.c#pg_relation_filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1022), [genfile.c#pg_stat_file](../../../../raw/postgres-12/src/backend/utils/adt/genfile.c#L362-L427), [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755).
- [heapam.c#scan-bound](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L219-L231), [heapam.c#page-read-and-prune](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L380-L405), [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795), [bufmgr.c#validation](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L897-L924), [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L64-L142).
- [copy.c#permission-path](../../../../raw/postgres-12/src/backend/commands/copy.c#L894-L951), [copy.c#output-functions](../../../../raw/postgres-12/src/backend/commands/copy.c#L2035-L2052), [copy.c#scan-dispatch](../../../../raw/postgres-12/src/backend/commands/copy.c#L2114-L2142), [copy.c#value-output](../../../../raw/postgres-12/src/backend/commands/copy.c#L2202-L2213), [tuptoaster.c#toast_fetch_datum](../../../../raw/postgres-12/src/backend/access/heap/tuptoaster.c#L1910-L2016), [rls.c#policy-gate](../../../../raw/postgres-12/src/backend/utils/misc/rls.c#L80-L132).
- [verify_nbtree.c#API-and-targets](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L171-L338), [verify_nbtree.c#heap-scan](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L554-L624), [verify_nbtree.c#callback](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L1893-L1920), [amcheck--1.0--1.1.sql#overloads](../../../../raw/postgres-12/contrib/amcheck/amcheck--1.0--1.1.sql#L9-L29), [amcheck/Makefile](../../../../raw/postgres-12/contrib/amcheck/Makefile#L1-L10).
- [pg_dump.c#session-setup](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1140-L1187), [pg_dump.c#COPY](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L1786-L1837), [pg_dump.c#data-selection](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L2342-L2368), [pg_dump.c#lock-timeout](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6536-L6548).
- [pg_checksums.c#exclusions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L94-L109), [pg_checksums.c#file-scan](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L169-L252), [pg_checksums.c#preconditions](../../../../raw/postgres-12/src/bin/pg_checksums/pg_checksums.c#L490-L527).
- [guc.c#scan-options](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L884-L921), [guc.c#developer-options](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1135-L1161), [guc.c#read-only](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1534-L1549), [guc.c#row-security](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1572-L1578), [guc.c#system-indexes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1784-L1793), [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378-L2396), [guc.c#search-path](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3790-L3797).
- [guc.c#superuser-context-gate](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L6859-L6868).
- [system_views.sql#prepared-transactions](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L325-L335), [system_views.sql#activity](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L732-L768), [system_views.sql#database-statistics](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L856-L887), [system_views.sql#file-privileges](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L1334-L1338).
- [tableam.h#scan-dispatch](../../../../raw/postgres-12/src/include/access/tableam.h#L761-L772), [tableam.h#index-build-dispatch](../../../../raw/postgres-12/src/include/access/tableam.h#L1512-L1533), [catalog/Makefile#catalogs](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L51), [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L103).
- [namespace.sql#schema-tests](../../../../raw/postgres-12/src/test/regress/sql/namespace.sql#L5-L44), [rowsecurity.sql#COPY-tests](../../../../raw/postgres-12/src/test/regress/sql/rowsecurity.sql#L1210-L1311), [check_btree.sql#checker-tests](../../../../raw/postgres-12/contrib/amcheck/sql/check_btree.sql#L1-L83).
- [fe-connect.c#connection-options](../../../../raw/postgres-12/src/interfaces/libpq/fe-connect.c#L198-L251), [startup.c#password-and-startup-file-options](../../../../raw/postgres-12/src/bin/psql/startup.c#L647-L658), [mainloop.c#error-stop](../../../../raw/postgres-12/src/bin/psql/mainloop.c#L558-L565), [psql-ref.sgml#ON_ERROR_STOP](../../../../raw/postgres-12/doc/src/sgml/ref/psql-ref.sgml#L3845-L3858).

## Navigation

- [v12/index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [DROP SCHEMA After Relation Files Were Emptied (unverified)](../storage-and-vacuum/drop-schema-zeroed-relation-files.md)
- [A Manually Zeroed Relation File (unverified)](../storage-and-vacuum/zeroed-relation-file.md)
- [Can amcheck Detect Zeroed Relation Segments? (unverified)](../indexing/amcheck-zeroed-relation-segments.md)
- [Checking Every Table for a Zeroed Last Segment (unverified)](../storage-and-vacuum/zeroed-last-segment-check-all-tables.md)
- [Corruption Indicators With Checksums Disabled (unverified)](../observability/corruption-log-messages-with-checksums-disabled.md)
- [General Database Health Checklist (unverified)](database-health-checklist.md)
