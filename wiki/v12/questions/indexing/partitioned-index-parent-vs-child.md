---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Indexes Only on the Parent Versus Only on the Child Tables of a Declaratively Partitioned Table in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [What an index on the parent actually is](#what-an-index-on-the-parent-actually-is)
  - [What is identical in both designs](#what-is-identical-in-both-designs)
  - [What the parent index buys you](#what-the-parent-index-buys-you)
  - [What the parent index costs you](#what-the-parent-index-costs-you)
  - [What per-partition indexes buy you](#what-per-partition-indexes-buy-you)
  - [What per-partition indexes cost you](#what-per-partition-indexes-cost-you)
  - [The lock footprint of each design](#the-lock-footprint-of-each-design)
  - [Capabilities only a parent index can provide](#capabilities-only-a-parent-index-can-provide)
  - [Maintenance and lifecycle differences](#maintenance-and-lifecycle-differences)
  - [Observability and tooling differences](#observability-and-tooling-differences)
  - [The hybrid the documentation recommends](#the-hybrid-the-documentation-recommends)
  - [Settings that matter](#settings-that-matter)
  - [Test coverage](#test-coverage)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, what are the pros and cons of creating indexes only on the parent table versus only on the child tables of a declaratively partitioned table?

Prompt-hygiene note: the prompt as submitted read `follow agents.md, in postgresql 12, question: what are the pros and cons of having on a declarative partitioning indexes only on the parent table versus only on the child tables.` The asker chose "correct and restate", so the misplaced modifier, the lowercase product name, and the missing capitalization were fixed before drafting. The asker also scoped the page to the `CREATE INDEX ON parent` cascade versus per-partition `CREATE INDEX`, and asked for a source-only analysis with no server measurements.

## Answer

### Verdict

**The choice is not symmetric, because "an index only on the parent" does not exist as a physical object in PostgreSQL 12.** `CREATE INDEX` on a partitioned table builds nothing itself; it creates a storage-less catalog row and then recurses, so that every partition ends up with a real index of its own ([indexcmds.c#DefineIndex-partitioned](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L595-L604), [indexcmds.c#recursion](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1041-L1076)). Both designs therefore end up with exactly one physical index per partition, and **query planning and execution are identical between them** for the same set of per-partition index definitions.

Everything that actually differs is DDL semantics, locking, and operations:

| | `CREATE INDEX ON parent` (cascade) | `CREATE INDEX` per partition, no parent index |
|---|---|---|
| Future partitions indexed automatically | Yes | No |
| `CONCURRENTLY` available | No | Yes, per partition |
| Writes blocked during the build | Every partition, until the one transaction commits | One partition at a time, or none with `CONCURRENTLY` |
| Definition may vary per partition | No | Yes |
| Drop an index on one partition only | No | Yes |
| Parent-level `UNIQUE` / `PRIMARY KEY` | Yes, if it contains all partition key columns | No |
| Table can be an FK target | Yes, via that unique index | No |
| `INSERT ... ON CONFLICT` on the parent | Yes | No |
| `REINDEX` the whole family in one command | No | No |
| Single `DROP INDEX` removes all of them | Yes | No |

Use the parent index when you need a partition-wide constraint, or when partitions are created and attached often enough that a missed index is a real risk. Use per-partition indexes when the index differs by partition (for example only recent partitions need it), or when you cannot afford a write outage across the whole table during the build. The documented middle path — `CREATE INDEX ON ONLY parent`, then `CREATE INDEX CONCURRENTLY` on each partition, then `ALTER INDEX ... ATTACH PARTITION` — gives you the parent's guarantees without the whole-hierarchy write lock ([ddl.sgml#index-partitioning](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3973-L4009)).

### What an index on the parent actually is

`DefineIndex` accepts `RELKIND_PARTITIONED_TABLE` as a target and sets a `partitioned` flag ([indexcmds.c#relkind-switch](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L568-L604)). From there:

1. It adds `INDEX_CREATE_SKIP_BUILD` and `INDEX_CREATE_PARTITIONED` to the flags, with the comment "or doing a partitioned index (because those don't have storage)" ([indexcmds.c#create-flags](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L962-L987)).
2. `index_create` gives the new relation `RELKIND_PARTITIONED_INDEX` ([index.c:735](../../../../raw/postgres-12/src/backend/catalog/index.c#L735)) and, because the build is skipped, only marks the table `relhasindex = true` ([index.c#skip-build](../../../../raw/postgres-12/src/backend/catalog/index.c#L1206-L1218)).
3. Unless `ONLY` was given, it walks `RelationGetPartitionDesc`, opens every partition, and for each one either **adopts** an existing index whose definition matches or **creates** a new one by calling itself recursively ([indexcmds.c#per-partition-loop](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1076-L1236)).
4. It then returns without building anything: "Indexes on partitioned tables are not themselves built, so we're done here" ([indexcmds.c#no-build](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1266-L1276)).

The parent row has no file. `index_drop` skips `RelationDropStorage` for it ([index.c#drop-storage](../../../../raw/postgres-12/src/backend/catalog/index.c#L2199-L2203)) and `pg_relation_filenode()` returns `NULL` for any relkind outside the storage list ([dbsize.c#pg_relation_filenode](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L877-L896)). The link between parent and child is a `pg_inherits` row plus two `pg_depend` rows of type `DEPENDENCY_PARTITION_PRI` and `DEPENDENCY_PARTITION_SEC` ([indexcmds.c#IndexSetParentIndex](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3398-L3406), [indexcmds.c#partition-dependencies](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3508-L3538), [index.c#index_create-dependencies](../../../../raw/postgres-12/src/backend/catalog/index.c#L1098-L1118)).

Adoption uses `CompareIndexInfo`, which requires the same access method, the same number of key and included columns, the same column mapping, the same collations and operator families, identical expressions, and an identical partial-index predicate; exclusion indexes are never matched ([index.c#CompareIndexInfo](../../../../raw/postgres-12/src/backend/catalog/index.c#L2408-L2523)). It does **not** compare index names, reloptions, or tablespace, so a partition index that differs only in `fillfactor` is still adopted.

There is a third spelling, `CREATE INDEX ON ONLY parent`, which really does index nothing: it skips the recursion, and if the table already has partitions the parent index is marked invalid ([indexcmds.c#ON-ONLY-invalid](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L988-L998), [create_index.sgml#ONLY](../../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L211-L219)). On a partitioned table with **no** partitions yet, that same statement leaves a valid parent index, because the `nparts != 0` test fails. This page treats `ON ONLY` as a build technique rather than a design, and covers it under [The hybrid the documentation recommends](#the-hybrid-the-documentation-recommends).

### What is identical in both designs

These are not trade-offs. They are the same either way, which is why the decision is an operational one.

- **Plan shape.** The planner explicitly skips partitioned indexes: "Ignore partitioned indexes, since they are not usable for queries" ([plancat.c#skip-partitioned-index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L205-L220)). Only the per-partition indexes reach `RelOptInfo.indexlist`, and those are the same relations in both designs.
- **Partition pruning.** Pruning is driven by the partition bounds, not by indexes: "partition pruning is driven only by the constraints defined implicitly by the partition keys, not by the presence of indexes" ([ddl.sgml#pruning-and-indexes](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L4541-L4549)).
- **Write-path maintenance cost.** Routed tuples are indexed against the leaf partition's own indexes, opened from `RelationGetIndexList(partition)` ([execPartition.c#ExecOpenIndices](../../../../raw/postgres-12/src/backend/executor/execPartition.c#L535-L545)). One physical index per partition is maintained either way.
- **Physical size.** The parent contributes no pages, so total index bytes are the sum over partitions in both designs.

One small asymmetry sits on the write path. Because the cascade sets `relhasindex` on the partitioned parent, `ExecInitModifyTable` opens the root's indexes for every non-`DELETE` statement ([nodeModifyTable.c#open-result-rel-indices](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L2336-L2349)), and `ExecOpenIndices` has no relkind filter — it `index_open`s each one under `RowExclusiveLock` ([execIndexing.c#ExecOpenIndices](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L160-L217)). So a parent index adds one relcache open plus one lock per statement that the per-partition design does not pay. No tuple is ever inserted into it.

### What the parent index buys you

- **Every future partition is indexed, with no extra step.** `CREATE TABLE ... PARTITION OF` clones each parent index into the new partition inside `DefineRelation` ([tablecmds.c#clone-parent-indexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1057-L1110)), and `ALTER TABLE ... ATTACH PARTITION` enforces the same rule — "every partition must have an index attached to each index on the partitioned table" ([tablecmds.c#AttachPartitionEnsureIndexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L15863-L15872)). The docs state the guarantee outright ([ddl.sgml#automatic-index](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3854-L3867)).
- **One definition, enforced.** Adoption goes through `CompareIndexInfo`, so a partition can only be covered by an index that really matches; drift is not silently accepted ([index.c#CompareIndexInfo](../../../../raw/postgres-12/src/backend/catalog/index.c#L2408-L2523)).
- **One `DROP INDEX` removes the whole family.** Dropping the parent index cascades to every attached child ([indexing.out#drop-parent](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L160-L177)).
- **A single handle for the family.** `pg_partition_tree()` accepts `RELKIND_PARTITIONED_INDEX` ([partitionfuncs.c#check_rel_can_be_partition](../../../../raw/postgres-12/src/backend/utils/adt/partitionfuncs.c#L43-L54)), which is what `\dPi+` uses to report a "Total size" summed over the whole index tree ([describe.c#dP-total-size](../../../../raw/postgres-12/src/bin/psql/describe.c#L3874-L3894), [describe.c#pg_partition_tree-size](../../../../raw/postgres-12/src/bin/psql/describe.c#L3929-L3940)).
- **Parent-level constraints.** See [Capabilities only a parent index can provide](#capabilities-only-a-parent-index-can-provide).

### What the parent index costs you

- **No `CONCURRENTLY`.** `DefineIndex` rejects it for a partitioned table before anything else, and does so even for temporary tables so the behavior is consistent ([indexcmds.c#no-concurrently](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L604-L616), [create_index.sgml#no-concurrent-partitioned](../../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L624-L631)).
- **A write outage across the whole hierarchy.** Detailed in [The lock footprint of each design](#the-lock-footprint-of-each-design).
- **All-or-nothing progress.** The non-concurrent path never splits transactions — the only `CommitTransactionCommand` calls in `DefineIndex` are on the concurrent path, which the partitioned case can never reach ([indexcmds.c#non-concurrent-return](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1266-L1288), [indexcmds.c#concurrent-commits](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1319-L1320)). A failure on the last partition discards every index built so far.
- **`REINDEX` does not work on the family.** `REINDEX INDEX parent_idx` raises `REINDEX is not yet implemented for partitioned indexes` ([indexcmds.c#ReindexPartitionedIndex](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3384-L3396)), and `REINDEX TABLE parent` emits `REINDEX of partitioned tables is not yet implemented, skipping` and returns — in both the plain and the concurrent path ([index.c#reindex_relation-skip](../../../../raw/postgres-12/src/backend/catalog/index.c#L3686-L3700), [indexcmds.c#reindex-concurrently-skip](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2917-L2923)). You still reindex partition by partition, exactly as in the per-partition design.
- **You cannot drop or replace one partition's index.** The `DEPENDENCY_PARTITION_PRI` row makes the child index undroppable while attached: `cannot drop index idxpart1_a_idx because index idxpart_a_idx requires it` / `HINT: You can drop index idxpart_a_idx instead.` ([dependency.c#partition-dependency-error](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1040-L1065), [indexing.out#cannot-drop-child](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L157-L159)). There is no `ALTER INDEX ... DETACH PARTITION` in v12 — the index partition grammar has only `ATTACH` ([gram.y#index_partition_cmd](../../../../raw/postgres-12/src/backend/parser/gram.y#L2048-L2062)). The only ways out are to drop the parent index or to detach the whole table partition, which un-parents its indexes ([tablecmds.c#detach-indexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16327-L16351)).
- **No per-partition variation.** Every partition gets the same access method, key list, opclasses, collations, expressions, and predicate, because that is what `CompareIndexInfo` demands and what the cloned `IndexStmt` reproduces ([indexcmds.c#clone-child-stmt](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1178-L1236)). A parent index *can* be partial or expression-based, but it is the same partial or expression definition everywhere ([indexing.out#partial-partitioned-index](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L739-L749)).
- **`ATTACH PARTITION` may turn into an index build.** If the incoming table has no matching index, ATTACH builds one right there, while holding `AccessExclusiveLock` on that table ([tablecmds.c#attach-build-index](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16012-L16029), [tablecmds.c#attach-lock](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L15598-L15607)).
- **Two restrictions with no per-partition counterpart.** Exclusion constraints are rejected on a partitioned table ([indexcmds.c#no-exclusion](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L617-L621)), and an explicit `TABLESPACE` equal to the database default is rejected with `cannot specify default tablespace for partitioned relations` ([indexcmds.c#tablespace](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L662-L669)).
- **A partition that is a foreign table is skipped, or fatal.** The cascade skips foreign-table partitions for a plain index and errors for a unique one ([indexcmds.c#foreign-partition](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1088-L1105)).

### What per-partition indexes buy you

- **`CREATE INDEX CONCURRENTLY` works.** A leaf partition is an ordinary `RELKIND_RELATION`, so `DefineIndex` takes the concurrent path and locks that one table with `ShareUpdateExclusiveLock` instead of `ShareLock` ([indexcmds.c#lockmode](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L559-L564)). Detail on the concurrent path is on the dedicated page, [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](create-index-concurrently.md).
- **Incremental, resumable rollout.** Each partition is its own statement and its own transaction, so a failure costs you one partition, not the batch.
- **Per-partition definitions.** Different access methods, different key sets, partial predicates tailored to a partition's data, different `fillfactor`, or simply no index at all on cold partitions. Nothing in the engine ties partitions together in this design.
- **Independent drops, including `DROP INDEX CONCURRENTLY`.** An unattached index carries no partition dependency, so the `cannot drop ... because ... requires it` check never fires and the concurrent drop path is available ([dependency.c#partition-dependency-error](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1040-L1065), [tablecmds.c#drop-concurrently](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1235-L1253)).
- **`ATTACH PARTITION` never turns into an index build.** With no partitioned index on the parent, `AttachPartitionEnsureIndexes` finds nothing to enforce — it only considers parent indexes whose relkind is `RELKIND_PARTITIONED_INDEX` ([tablecmds.c#ignore-plain-parent-indexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L15945-L15953)).
- **`REINDEX` per partition works normally,** including `CONCURRENTLY`, because only `RELKIND_PARTITIONED_INDEX` is rejected ([indexcmds.c#ReindexIndex](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2366-L2382), [index.c#reindex_index-relkind](../../../../raw/postgres-12/src/backend/catalog/index.c#L3476-L3482)). This is equally true of a child index that is attached to a parent.

### What per-partition indexes cost you

- **New partitions are not indexed.** Nothing clones an index into a new partition, because the clone loops are driven by the parent's index list ([tablecmds.c#clone-parent-indexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1057-L1110), [tablecmds.c#AttachPartitionEnsureIndexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L15932-L15940)). With monthly partitions and no automation this is the classic way to end up with a sequential scan on the newest, hottest partition.
- **No parent-level `UNIQUE`, `PRIMARY KEY`, FK target, or `ON CONFLICT`.** See the next section. Per-partition unique indexes enforce uniqueness *within* each partition only, which is the limitation the documentation states directly ([ddl.sgml#unique-limitation](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L4026-L4033)).
- **Drift is unpoliced.** Nothing compares definitions across partitions, so a typo or an omitted column on one partition is invisible until a query plan changes.
- **N-way DDL.** Every add, drop, rename, or reindex is a loop over partitions with no single catalog handle.
- **`relhasindex` on the parent stays false**, so `pg_tables.hasindexes` reports `false` for the partitioned table ([system_views.sql#pg_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L127-L139)) and `\dPi` lists nothing for it ([describe.c#dPi](../../../../raw/postgres-12/src/bin/psql/describe.c#L3824-L3862)).

### The lock footprint of each design

This is the dominant practical difference.

`CREATE INDEX ON parent` runs non-concurrently, so `lockmode` is `ShareLock` ([indexcmds.c#lockmode](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L559-L564)). The recursion opens every partition with that same `lockmode` and deliberately keeps it — the code closes each child with `NoLock` under the comment "keep lock till commit" ([indexcmds.c#child-open](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1076-L1086), [indexcmds.c#keep-lock](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1167-L1176)). `ShareLock` conflicts with `RowExclusiveLock` ([lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L83-L87)), which is what `INSERT`, `UPDATE`, and `DELETE` take. So:

- Writes are blocked on **every partition** — including partitions already finished and partitions not yet started — until the single transaction commits.
- Reads are unaffected: `AccessShareLock` conflicts only with `AccessExclusiveLock` ([lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L101)).
- The lock count scales with the hierarchy: one `ShareLock` per partition plus one relation lock per new index, all held in one transaction, which is `max_locks_per_transaction` pressure on wide hierarchies.

Per-partition `CREATE INDEX CONCURRENTLY` takes `ShareUpdateExclusiveLock` on one partition at a time, which does not conflict with `RowExclusiveLock` ([lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L78-L82)); writes continue everywhere. A plain per-partition `CREATE INDEX` still takes `ShareLock`, but on one partition, briefly.

The attach-side locks are worth knowing in both designs:

| Operation | Locks taken |
|---|---|
| `ALTER TABLE parent ATTACH PARTITION t` | `ShareUpdateExclusiveLock` on the parent ([tablecmds.c#attach-lock-level](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L3806-L3808)), `AccessExclusiveLock` on `t` ([tablecmds.c#attach-lock](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L15598-L15607)), plus `AccessExclusiveLock` on the default partition if one exists |
| `ALTER TABLE parent DETACH PARTITION t` | `AccessExclusiveLock` on the parent ([tablecmds.c#detach-lock-level](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L3810-L3812)) and on each of `t`'s attached indexes ([tablecmds.c#detach-indexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16327-L16351)) |
| `ALTER INDEX parent_idx ATTACH PARTITION child_idx` | `ShareUpdateExclusiveLock` on `parent_idx`, `AccessExclusiveLock` on `child_idx`, `AccessShareLock` on both tables ([tablecmds.c#ATExecAttachPartitionIdx](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16514-L16541)); in a multi-level hierarchy the validation step also takes `AccessExclusiveLock` on the grandparent index *and* the grandparent table ([tablecmds.c#validate-recursion](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16744-L16768)) |

### Capabilities only a parent index can provide

Three features read the *parent's* index list, so per-partition indexes cannot supply them.

**Parent-level `UNIQUE` and `PRIMARY KEY`.** They exist only as partitioned indexes, and every partition key column must appear in the key: otherwise `insufficient columns in %s constraint definition` with a detail naming the missing column. An expression partition key is refused outright with `unsupported %s constraint with partition key definition` ([indexcmds.c#unique-partition-key](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L823-L898)). The code comment explains the rule: "A partition-local index can enforce global uniqueness iff the PK value completely determines the partition that a row is in." There is no escape hatch through an existing index, because `ALTER TABLE ... ADD CONSTRAINT ... USING INDEX is not supported on partitioned tables` ([tablecmds.c#using-index-unsupported](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L7344-L7351)).

**Being the referenced side of a foreign key.** v12 lets `pkrel` be a partitioned table ([tablecmds.c#pkrel-relkind](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L7722-L7727)), but the matching index is found by scanning `RelationGetIndexList(pkrel)` for one that is unique, valid, immediate, non-partial, and non-expression; otherwise `there is no unique constraint matching given keys for referenced table` ([tablecmds.c#transformFkeyCheckAttrs](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L9550-L9647)). Only a parent-level unique index appears in that list.

**`INSERT ... ON CONFLICT` against the parent.** `infer_arbiter_indexes` scans the target relation's own index list, does not skip partitioned indexes, and raises `there is no unique or exclusion constraint matching the ON CONFLICT specification` when nothing matches ([plancat.c#infer_arbiter_indexes](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L588-L598), [plancat.c#arbiter-no-match](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L674-L676), [plancat.c#arbiter-error](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L819-L822)). The executor then maps each chosen parent index down to the partition's own index through `get_partition_ancestors` ([execPartition.c#arbiter-mapping](../../../../raw/postgres-12/src/backend/executor/execPartition.c#L690-L728)).

### Maintenance and lifecycle differences

- **Adopting existing per-partition indexes.** You are not locked into your first choice. If matching indexes already exist on the partitions, `CREATE INDEX ON parent` adopts them via `IndexSetParentIndex` instead of building duplicates ([indexcmds.c#adopt](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1114-L1173), [indexing.out#no-duplicate](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L125-L150)). Migrating from per-partition to parent-managed is therefore a metadata-only operation when the definitions already agree.
- **Invalid children make the parent invalid.** If the cascade adopts an index that is itself invalid — for example a partition left behind by a failed `CREATE INDEX CONCURRENTLY` — the parent's `pg_index` row is updated to `indisvalid = false` ([indexcmds.c#invalidate-parent](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1243-L1265)). See [All Outcomes That Leave an Invalid Index in PostgreSQL 12](invalid-index-outcomes.md).
- **Detaching restores independence.** `ALTER TABLE ... DETACH PARTITION` calls `IndexSetParentIndex(idx, InvalidOid)`, which deletes the `pg_inherits` row and both partition dependencies ([tablecmds.c#detach-indexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16327-L16351), [indexcmds.c#partition-dependencies](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3508-L3538)). After a detach the indexes can be dropped individually, and dropping the parent index no longer removes them ([indexing.out#detach-then-drop](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L442-L523)).
- **`ATTACH PARTITION` never drops an index** in either design; it is strictly match-or-create. That question has its own page, [Can ALTER TABLE ... ATTACH PARTITION Drop Indexes in PostgreSQL 12?](attach-partition-index-drops.md).
- **Dump and restore cost the same.** `pg_dump` never emits a recursive `CREATE INDEX ON parent`. It emits the parent index as `CREATE INDEX ... ON ONLY ...` — `pg_get_indexdef` inserts `ONLY` for `RELKIND_PARTITIONED_INDEX` ([ruleutils.c#ON-ONLY](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1287-L1298)) — then one `ALTER INDEX ... ATTACH PARTITION` per child in the post-data section ([pg_dump.c#dumpIndexAttach](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16470-L16498), [pg_dump.c#getIndexes-parentidx](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6877-L6883)). Restore therefore builds exactly the same per-partition indexes in both designs, plus N cheap attach statements.

### Observability and tooling differences

| Surface | With a parent index | Per-partition only |
|---|---|---|
| `pg_indexes` | Lists the parent as `CREATE INDEX ... ON ONLY ...` alongside every child; the view admits both `i` and `I` ([system_views.sql#pg_indexes](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L154-L165), [indexing.out#pg_indexes](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L13-L18)) | Children only |
| `pg_stat_all_indexes` | Parent absent — the view is restricted to `relkind IN ('r','t','m')`, which excludes partitioned tables ([system_views.sql#pg_stat_all_indexes](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672)) | Same; usage must be summed over partitions in both designs |
| `pg_indexes_size(parent)` | Sums only the parent's own storage-less indexes, because `RelationGetIndexList(parent)` never returns child index OIDs ([dbsize.c#calculate_indexes_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L410-L448)) | Not applicable |
| `pg_partition_tree()` | Enumerates the whole index family from one OID ([partitionfuncs.c#check_rel_can_be_partition](../../../../raw/postgres-12/src/backend/utils/adt/partitionfuncs.c#L43-L54)) | No index-side root to start from |
| `psql` | `\d parent_idx` prints `Partitioned index`; `\d+ child_idx` prints `Partition of: parent_idx`; `\dPi` and `\dPi+` list the family and its total size ([describe.c#partitioned-index-title](../../../../raw/postgres-12/src/bin/psql/describe.c#L1960-L1967), [describe.c#Partition-of](../../../../raw/postgres-12/src/bin/psql/describe.c#L2127-L2134), [indexing.out#psql-partition-of](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L105-L122)) | No parent to describe |
| `pg_class.relhassubclass` on the parent index | `false` until the first child index is attached; `IndexSetParentIndex` sets it ([indexcmds.c#SetRelationHasSubclass](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3501-L3506), [indexing.out#relhassubclass](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L1-L50)) | Not applicable |

### The hybrid the documentation recommends

The documentation names the write-outage problem and gives the workaround: `CREATE INDEX ON ONLY` the parent (an invalid, storage-less parent index that does not touch the partitions), build each partition's index with `CONCURRENTLY`, then attach them; the parent becomes valid automatically once all partitions are covered ([ddl.sgml#index-partitioning](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3973-L4009), [create_index.sgml#partitioned-notes](../../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L670-L690)). The `CREATE INDEX` reference page adds that finishing the parent this way "is a metadata only operation" ([create_index.sgml#no-concurrent-partitioned](../../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L624-L631)).

Validation is `validatePartitionedIndex`: it counts valid attached children against `RelationGetPartitionDesc(parent)->nparts` and flips `indisvalid` only when they match, recursing upward for sub-partitioned hierarchies ([tablecmds.c#validatePartitionedIndex](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16675-L16769)). Attaching an *invalid* child does not validate the parent ([indexing.out#validate-chain](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L400-L439)).

A skeleton, with the session-scoped guards this repository requires. All syntax is from the pinned checkout's own documentation examples; adjust the timeouts to your build times.

```sql
SET /* wiki_partitioned_index_rollout */ lock_timeout = '5s';
SET /* wiki_partitioned_index_rollout */ statement_timeout = '2h';

-- 1. Metadata-only: invalid parent, partitions untouched.
CREATE /* wiki_partitioned_index_rollout */ INDEX measurement_usls_idx
    ON ONLY measurement (unitsales);

-- 2. Per partition, writes stay available. Repeat for each partition.
CREATE /* wiki_partitioned_index_rollout */ INDEX CONCURRENTLY measurement_usls_200602_idx
    ON measurement_y2006m02 (unitsales);

-- 3. Metadata-only; the parent turns valid after the last attach.
ALTER /* wiki_partitioned_index_rollout */ INDEX measurement_usls_idx
    ATTACH PARTITION measurement_usls_200602_idx;
```

`CREATE INDEX CONCURRENTLY` cannot run inside a transaction block — `ProcessUtilitySlow` calls `PreventInTransactionBlock(isTopLevel, "CREATE INDEX CONCURRENTLY")` for any `IndexStmt` with `concurrent` set ([utility.c#T_IndexStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L1301-L1310)) — so step 2 must be its own statement per partition. The step-by-step lock and wait behavior of that command is on [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](create-index-concurrently.md).

Two caveats specific to v12:

- Once the parent index exists, even invalid, any partition created by `CREATE TABLE ... PARTITION OF` gets a matching index immediately and non-concurrently, because `RelationGetIndexList` returns invalid indexes too ([relcache.c#RelationGetIndexList](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4384-L4398), [ddl.sgml#index-partitioning](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3973-L4009)). The `ONLY` escape covers existing partitions, not future ones.
- The same applies to `ATTACH PARTITION`: `AttachPartitionEnsureIndexes` does not check the parent index's validity before enforcing coverage ([tablecmds.c#AttachPartitionEnsureIndexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L15932-L15953)).

### Settings that matter

Only one GUC is materially different between the two designs.

| GUC | Why it matters here | Context and apply scope |
|---|---|---|
| `max_locks_per_transaction` | The parent cascade holds one relation lock per partition plus one per created index in a single transaction; the per-partition design spreads them over separate transactions. The shared lock table is sized as `max_locks_per_transaction * (MaxBackends + max_prepared_xacts)` ([lock.c#NLOCKENTS](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L53-L57)) | Default 64, `PGC_POSTMASTER` — **requires a restart** ([guc.c#max_locks_per_transaction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2463-L2473)) |

`statement_timeout` and `lock_timeout` are both `PGC_USERSET` and default to 0, so they take session or transaction scope with no reload or restart ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397)). Their exact behavior around index builds and lock waits is tabulated on [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 12](create-index-concurrently.md). `enable_partition_pruning` is not an index setting — pruning does not consult indexes at all ([ddl.sgml#pruning-and-indexes](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L4541-L4549)).

### Test coverage

`src/test/regress/sql/indexing.sql` is the dedicated suite, and it covers the design difference directly:

| Behavior | Test evidence |
|---|---|
| Cascade indexes existing and future partitions; multi-level `inhparent` chain | [indexing.out:1-50](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L1-L50) |
| `CREATE INDEX CONCURRENTLY` refused on a partitioned table | [indexing.out:53-58](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L53-L58) |
| ATTACH adopts an existing matching partition index; `\d+` shows `Partition of:` | [indexing.out:92-122](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L92-L122) |
| No duplicate index created when the partition already has a match | [indexing.out:125-150](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L125-L150) |
| Attached child index cannot be dropped; parent drop removes both | [indexing.out:153-177](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L153-L177) |
| `ON ONLY` parents stay invalid until every child is attached, recursively | [indexing.out:400-439](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L400-L439) |
| After DETACH, indexes drop independently and the parent drop spares them | [indexing.out:442-523](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L442-L523) |
| Partial partitioned index propagates its predicate to every partition | [indexing.out:729-749](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L729-L749) |
| PK on partitions cannot be dropped independently of the parent PK | [indexing.out:1024-1052](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L1024-L1052) |

What the suite does **not** contain: any comparison of query plans, timings, or lock waits between the two designs, and no isolation spec covering concurrent writes during a cascading `CREATE INDEX`. The lock and performance statements on this page are read from the source, not from a test.

## Context Reviewed

- `DefineIndex` end to end, including the partitioned branch, the per-partition recursion, adoption, and the invalid-parent update ([indexcmds.c](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L429-L1288)).
- `index_create` flags, relkind selection, `pg_inherits` and `pg_depend` wiring, `index_update_stats`, `index_drop` storage handling, `CompareIndexInfo`, `reindex_index`, `reindex_relation` ([index.c](../../../../raw/postgres-12/src/backend/catalog/index.c#L643-L3700)).
- `DefineRelation`'s partition-of index cloning, ALTER TABLE lock levels, `ATExecAttachPartition`, `AttachPartitionEnsureIndexes`, `ATExecDetachPartition` index handling, `ATExecAttachPartitionIdx`, `refuseDupeIndexAttach`, `validatePartitionedIndex`, `ATExecAddIndexConstraint`, `ATAddForeignKeyConstraint`, `transformFkeyCheckAttrs` ([tablecmds.c](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1057-L16769)).
- Planner index collection and arbiter inference ([plancat.c](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L205-L825)).
- Executor index opening on the root result relation and on leaf partitions, and arbiter mapping ([nodeModifyTable.c](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L2336-L2349), [execIndexing.c](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L140-L248), [execPartition.c](../../../../raw/postgres-12/src/backend/executor/execPartition.c#L529-L728)).
- Partition dependency handling and the drop error ([dependency.c](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L755-L1065)).
- Lock conflict table and lock-table sizing ([lock.c](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L53-L101)); `max_locks_per_transaction` definition ([guc.c](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2463-L2473)).
- Grammar for `ALTER TABLE`/`ALTER INDEX` partition commands ([gram.y](../../../../raw/postgres-12/src/backend/parser/gram.y#L2019-L2062)).
- Catalog views, size functions, index deparse, `pg_partition_tree`, `RelationGetIndexList` ([system_views.sql](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L128-L699), [dbsize.c](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L509), [ruleutils.c](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1287-L1298), [partitionfuncs.c](../../../../raw/postgres-12/src/backend/utils/adt/partitionfuncs.c#L34-L65), [relcache.c](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4317-L4409)).
- Client tools: `pg_dump` index and index-attach emission, `psql` describe paths ([pg_dump.c](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L6812-L16498), [describe.c](../../../../raw/postgres-12/src/bin/psql/describe.c#L1956-L3958)).
- Documentation: declarative partitioning setup, maintenance, limitations, pruning; `CREATE INDEX` reference ([ddl.sgml](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3854-L4549), [create_index.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L101-L690)).
- Regression suite ([indexing.sql](../../../../raw/postgres-12/src/test/regress/sql/indexing.sql#L1-L350), [indexing.out](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L1-L1052)).

## Evidence Map

| Claim | Source |
|---|---|
| A parent index is storage-less; the build is skipped | [indexcmds.c:962-987](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L962-L987), [index.c:735](../../../../raw/postgres-12/src/backend/catalog/index.c#L735), [index.c:2199-2203](../../../../raw/postgres-12/src/backend/catalog/index.c#L2199-L2203) |
| `CREATE INDEX ON parent` recurses to every partition | [indexcmds.c:1041-1076](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1041-L1076), [ddl.sgml:3854-3867](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3854-L3867) |
| Existing matching partition index is adopted, not duplicated | [indexcmds.c:1114-1173](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1114-L1173), [indexing.out:125-150](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L125-L150) |
| Definition equality rules for adoption | [index.c#CompareIndexInfo](../../../../raw/postgres-12/src/backend/catalog/index.c#L2408-L2523) |
| Planner ignores partitioned indexes | [plancat.c:205-220](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L205-L220) |
| Pruning does not depend on indexes | [ddl.sgml:4541-4549](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L4541-L4549) |
| Leaf partition indexes are what the write path maintains | [execPartition.c:535-545](../../../../raw/postgres-12/src/backend/executor/execPartition.c#L535-L545) |
| Root's partitioned indexes are opened and `RowExclusiveLock`ed per statement | [nodeModifyTable.c:2336-2349](../../../../raw/postgres-12/src/backend/executor/nodeModifyTable.c#L2336-L2349), [execIndexing.c:160-217](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L160-L217) |
| `CONCURRENTLY` rejected on a partitioned table | [indexcmds.c:604-616](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L604-L616), [indexing.out:53-58](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L53-L58) |
| Cascade holds `ShareLock` on every partition until commit | [indexcmds.c:559-564](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L559-L564), [indexcmds.c:1076-1086](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1076-L1086), [indexcmds.c:1167-1176](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1167-L1176) |
| `ShareLock` blocks DML but not reads | [lock.c:65-101](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L65-L101) |
| Cascade is one transaction | [indexcmds.c:1266-1288](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1266-L1288), [indexcmds.c:1319-1320](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1319-L1320) |
| `REINDEX` unimplemented for partitioned index and partitioned table | [indexcmds.c:3384-3396](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3384-L3396), [index.c:3686-3700](../../../../raw/postgres-12/src/backend/catalog/index.c#L3686-L3700), [indexcmds.c:2917-2923](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2917-L2923) |
| `REINDEX` on a child index still works | [indexcmds.c:2366-2382](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L2366-L2382), [index.c:3476-3482](../../../../raw/postgres-12/src/backend/catalog/index.c#L3476-L3482) |
| Attached child index cannot be dropped | [dependency.c:1040-1065](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1040-L1065), [indexing.out:157-159](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L157-L159) |
| No `ALTER INDEX ... DETACH PARTITION` in v12 | [gram.y:2048-2062](../../../../raw/postgres-12/src/backend/parser/gram.y#L2048-L2062) |
| DETACH un-parents the partition's indexes | [tablecmds.c:16327-16351](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16327-L16351), [indexing.out:442-523](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L442-L523) |
| ATTACH builds a missing index under `AccessExclusiveLock` | [tablecmds.c:15598-15607](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L15598-L15607), [tablecmds.c:16012-16029](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16012-L16029) |
| ALTER TABLE attach/detach lock levels | [tablecmds.c:3806-3812](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L3806-L3812) |
| `ALTER INDEX ... ATTACH PARTITION` lock levels and validation recursion | [tablecmds.c:16514-16541](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16514-L16541), [tablecmds.c:16744-16768](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16744-L16768) |
| Unique/PK must include all partition key columns | [indexcmds.c:823-898](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L823-L898), [ddl.sgml:4026-4033](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L4026-L4033) |
| `ADD CONSTRAINT ... USING INDEX` unsupported on partitioned tables | [tablecmds.c:7344-7351](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L7344-L7351) |
| FK may reference a partitioned table, needs a unique valid index on it | [tablecmds.c:7722-7727](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L7722-L7727), [tablecmds.c:9550-9647](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L9550-L9647) |
| `ON CONFLICT` needs a parent-level unique index, then maps to children | [plancat.c:588-598](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L588-L598), [plancat.c:819-822](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L819-L822), [execPartition.c:690-728](../../../../raw/postgres-12/src/backend/executor/execPartition.c#L690-L728) |
| `ON ONLY` marks the parent invalid only when partitions exist | [indexcmds.c:988-998](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L988-L998) |
| Validation flips `indisvalid` when all partitions are covered | [tablecmds.c:16675-16769](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16675-L16769), [indexing.out:400-439](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L400-L439) |
| Invalid indexes stay in `RelationGetIndexList`, so future partitions still get one | [relcache.c:4384-4398](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4384-L4398), [ddl.sgml:3973-4009](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3973-L4009) |
| `CREATE TABLE ... PARTITION OF` clones parent indexes | [tablecmds.c:1057-1110](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1057-L1110) |
| `relhasindex` set on the partitioned parent | [index.c:1206-1218](../../../../raw/postgres-12/src/backend/catalog/index.c#L1206-L1218), [system_views.sql:127-139](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L127-L139) |
| `pg_indexes` includes partitioned indexes, deparsed with `ON ONLY` | [system_views.sql:154-165](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L154-L165), [ruleutils.c:1287-1298](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1287-L1298) |
| `pg_stat_all_indexes` excludes partitioned tables | [system_views.sql:658-672](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L658-L672) |
| `pg_indexes_size` does not aggregate partition indexes | [dbsize.c:410-448](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L410-L448) |
| `pg_partition_tree` accepts a partitioned index; `\dPi+` uses it for total size | [partitionfuncs.c:43-54](../../../../raw/postgres-12/src/backend/utils/adt/partitionfuncs.c#L43-L54), [describe.c:3929-3940](../../../../raw/postgres-12/src/bin/psql/describe.c#L3929-L3940) |
| `pg_dump` emits `ON ONLY` plus one `ALTER INDEX ... ATTACH PARTITION` per child | [pg_dump.c:16470-16498](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16470-L16498) |
| `max_locks_per_transaction` is `PGC_POSTMASTER` and sizes the lock table | [guc.c:2463-2473](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2463-L2473), [lock.c:53-57](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L53-L57) |
| Exclusion constraints and default tablespace rejected on partitioned tables | [indexcmds.c:617-621](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L617-L621), [indexcmds.c:662-669](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L662-L669) |

## Open Questions

- The page states that a cascading `CREATE INDEX` blocks writes on every partition for the duration of one transaction. That is read from the lock mode, the per-child `table_open`, the "keep lock till commit" comment, and the conflict table; it is **not** measured, and there is no isolation spec in the pinned checkout that exercises a concurrent writer against a cascading build. A measured version would need a two-session isolation test.
- `pg_relation_size()` on a partitioned index is not claimed here. `pg_relation_filenode()` provably returns `NULL` for it, and `index_drop` skips storage, but the exact value `calculate_relation_size` returns for a zero `relfilenode` was inferred from `stat()` failing with `ENOENT`, not observed.
- The per-statement cost of opening the root's partitioned indexes on `INSERT`/`UPDATE` is established from the code path but not quantified. Whether it is measurable against a wide hierarchy is unknown without a benchmark.
- This page does not compare v12 against later majors. `REINDEX` on partitioned tables, `ALTER INDEX ... DETACH PARTITION`, and `ALTER TABLE ... DETACH PARTITION CONCURRENTLY` are absent in v12; when they arrived is a cross-version question that would need per-version evidence, per the one-version-per-page citation rule.

## Source References

- [indexcmds.c#DefineIndex](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L429-L1288)
- [indexcmds.c#ReindexPartitionedIndex](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3384-L3396)
- [indexcmds.c#IndexSetParentIndex](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3398-L3543)
- [index.c#index_create](../../../../raw/postgres-12/src/backend/catalog/index.c#L643-L1231)
- [index.c#CompareIndexInfo](../../../../raw/postgres-12/src/backend/catalog/index.c#L2408-L2523)
- [index.c#reindex_relation](../../../../raw/postgres-12/src/backend/catalog/index.c#L3670-L3700)
- [tablecmds.c#DefineRelation-clone-indexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1057-L1110)
- [tablecmds.c#AttachPartitionEnsureIndexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L15863-L16040)
- [tablecmds.c#ATExecAttachPartitionIdx](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16503-L16652)
- [tablecmds.c#validatePartitionedIndex](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L16675-L16769)
- [tablecmds.c#transformFkeyCheckAttrs](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L9512-L9652)
- [plancat.c#get_relation_info](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L205-L220)
- [plancat.c#infer_arbiter_indexes](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L568-L825)
- [execPartition.c#ExecInitPartitionInfo](../../../../raw/postgres-12/src/backend/executor/execPartition.c#L529-L728)
- [execIndexing.c#ExecOpenIndices](../../../../raw/postgres-12/src/backend/executor/execIndexing.c#L140-L217)
- [dependency.c#reportDependentObjects](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1013-L1065)
- [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L53-L101)
- [guc.c#max_locks_per_transaction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2463-L2473)
- [gram.y#partition_cmd](../../../../raw/postgres-12/src/backend/parser/gram.y#L2019-L2062)
- [system_views.sql#index-views](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L128-L699)
- [ruleutils.c#pg_get_indexdef_worker](../../../../raw/postgres-12/src/backend/utils/adt/ruleutils.c#L1167-L1302)
- [partitionfuncs.c#pg_partition_tree](../../../../raw/postgres-12/src/backend/utils/adt/partitionfuncs.c#L34-L65)
- [pg_dump.c#dumpIndexAttach](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16470-L16498)
- [describe.c#partitioned-index](../../../../raw/postgres-12/src/bin/psql/describe.c#L1956-L1967)
- [ddl.sgml#declarative-partitioning](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L3854-L4549)
- [create_index.sgml#partitioned-tables](../../../../raw/postgres-12/doc/src/sgml/ref/create_index.sgml#L101-L690)
- [indexing.out#partitioned-index-suite](../../../../raw/postgres-12/src/test/regress/expected/indexing.out#L1-L1052)

## Navigation

- [v12 index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [v12: Can ALTER TABLE ... ATTACH PARTITION Drop Indexes?](attach-partition-index-drops.md)
- [v12: How CREATE INDEX CONCURRENTLY Is Implemented](create-index-concurrently.md)
- [v12: All Outcomes That Leave an Invalid Index](invalid-index-outcomes.md)
- [v12: Pros and Cons of Partial Indexes](partial-indexes-pros-cons.md)
- [v12: Table Partitioning Optimizations During Query Planning and Execution](../query-planning/partitioning-planning-execution-optimizations.md)
