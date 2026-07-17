---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: GPT-5-6-Sol-Max-Thinking 2026-07-17T19:53:21Z
---

# Can ALTER TABLE ... ATTACH PARTITION Drop Indexes in PostgreSQL 17? (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer and scope](#short-answer-and-scope)
  - [Parser and utility path](#parser-and-utility-path)
  - [Locks prevent a concurrent DROP from interleaving](#locks-prevent-a-concurrent-drop-from-interleaving)
  - [The core match-or-create decision](#the-core-match-or-create-decision)
  - [What compatible means](#what-compatible-means)
  - [What re-parenting changes](#what-re-parenting-changes)
  - [Creation and multilevel partitions](#creation-and-multilevel-partitions)
  - [Errors roll back the whole statement](#errors-roll-back-the-whole-statement)
  - [The closest look-alikes](#the-closest-look-alikes)
  - [ALTER INDEX ATTACH PARTITION also does not drop](#alter-index-attach-partition-also-does-not-drop)
  - [Custom code can issue a separate DROP](#custom-code-can-issue-a-separate-drop)
  - [Avoiding an index build during ATTACH](#avoiding-an-index-build-during-attach)
  - [Changes since PostgreSQL 12](#changes-since-postgresql-12)
  - [Test coverage and gaps](#test-coverage-and-gaps)
  - [Build, generated-file, access-method, and contrib boundaries](#build-generated-file-access-method-and-contrib-boundaries)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, in what scenarios can indexes be dropped from a table during
ALTER TABLE ... ATTACH PARTITION on a declarative partitioned table? Add a
section with changes since PostgreSQL 12.

## Answer

### Short answer and scope

**PostgreSQL 17's built-in `ALTER TABLE ... ATTACH PARTITION` path does not
drop an existing index.** For each partitioned index on the parent, it either
reuses a compatible, valid, unattached index on the table being attached or
creates a new matching index. Extra, incompatible, already-attached, and
directly encountered invalid indexes remain in place
([tablecmds.c#AttachPartitionEnsureIndexes](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18805-L18986)).

That answer is about core PostgreSQL semantics. A user-defined event trigger or
a loadable module's `ProcessUtility_hook` runs outside the match-or-create
routine and can execute a separate `DROP INDEX`. If an index disappears while
the client is executing `ATTACH PARTITION`, inspect that custom code before
attributing the drop to ATTACH itself
([utility.c#ProcessUtility](../../../raw/postgres-17/src/backend/tcop/utility.c#L498-L525),
[event_trigger.c#EventTriggerDDLCommandStart](../../../raw/postgres-17/src/backend/commands/event_trigger.c#L721-L769),
[event_trigger.c#EventTriggerDDLCommandEnd](../../../raw/postgres-17/src/backend/commands/event_trigger.c#L772-L817)).

### Parser and utility path

The grammar converts table ATTACH syntax into an `AlterTableCmd` with subtype
`AT_AttachPartition`; its `PartitionCmd` carries the table name and transformed
partition bound
([gram.y#partition_cmd](../../../raw/postgres-17/src/backend/parser/gram.y#L2311-L2325),
[parsenodes.h#PartitionCmd](../../../raw/postgres-17/src/include/nodes/parsenodes.h#L950-L959)).
`ProcessUtilitySlow` fires `ddl_command_start`, chooses and acquires the parent
lock, and calls `AlterTable`
([utility.c#ProcessUtilitySlow](../../../raw/postgres-17/src/backend/tcop/utility.c#L1083-L1114),
[utility.c#T_AlterTableStmt](../../../raw/postgres-17/src/backend/tcop/utility.c#L1268-L1321)).

`ATExecCmd` then dispatches the same subtype to `ATExecAttachPartition` for a
partitioned table or to `ATExecAttachPartitionIdx` for a partitioned index
([tablecmds.c#ATExecCmd](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L5510-L5520)).
The table path establishes table inheritance and the partition bound, then
calls `AttachPartitionEnsureIndexes` before cloning row triggers and foreign
keys
([tablecmds.c#ATExecAttachPartition](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18686-L18710)).

### Locks prevent a concurrent DROP from interleaving

ATTACH selects `ShareUpdateExclusiveLock` for the parent table. It takes
`AccessExclusiveLock` on the table being attached, all of that table's
descendants, and any affected default partition; the child lock is retained
until transaction end
([tablecmds.c#AlterTableGetLockLevel](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L4697-L4705),
[tablecmds.c#ATExecAttachPartition-locks](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18517-L18526),
[tablecmds.c#ATExecAttachPartition-descendants](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18583-L18600),
[tablecmds.c:18798-18799](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18798-L18799)).

`DROP INDEX` first locks the index's table. A normal drop requests
`AccessExclusiveLock`; a concurrent drop requests `ShareUpdateExclusiveLock`.
Both conflict with ATTACH's lock on the relevant parent or child table, so the
commands serialize rather than deleting an index midway through the core
ATTACH routine
([tablecmds.c#RemoveRelations](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1462-L1494),
[tablecmds.c:1558-1568](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1558-L1568),
[tablecmds.c#RangeVarCallbackForDropRelation](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1745-L1759),
[lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L59-L104)).
`DROP INDEX CONCURRENTLY` cannot target a partitioned parent index at all
([tablecmds.c:1589-1598](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1589-L1598)).

### The core match-or-create decision

`AttachPartitionEnsureIndexes` snapshots the parent and child index lists,
opens each existing child index, and builds an `IndexInfo` description for
matching
([tablecmds.c:18813-18840](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18813-L18840)).
Only parent relations whose `relkind` is `RELKIND_PARTITIONED_INDEX` drive this
loop
([tablecmds.c:18870-18891](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18870-L18891)).

| State on the table being attached | Core action | Existing index dropped? |
|---|---|---|
| A compatible, valid, unattached index exists | Re-parent the first match with `IndexSetParentIndex`; re-parent its constraint too when required ([tablecmds.c:18900-18954](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18900-L18954)) | No. The same index OID and storage are retained. |
| No candidate passes all checks | Generate a clone and call `DefineIndex` ([tablecmds.c:18958-18975](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18958-L18975)) | No. A new index is attempted. |
| An extra or incompatible index exists | Ignore it and continue looking; create a matching index if none is found ([tablecmds.c:18905-18975](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18905-L18975)) | No. The extra index remains. |
| A direct candidate is invalid | Skip it because `indisvalid` is false, then keep looking ([tablecmds.c:18910-18917](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18910-L18917)) | No. The invalid index remains. |
| A candidate already belongs to another index tree | Skip it because `relispartition` is true ([tablecmds.c:18910-18912](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18910-L18912)) | No. It remains attached to its existing parent. |
| The parent index backs a constraint | Reuse requires a child constraint and, in v17, the same constraint type; otherwise creation is attempted ([tablecmds.c:18925-18950](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18925-L18950)) | No. Creation can error, but it does not replace the old index. |
| The table being attached is foreign and the parent has a unique or primary index | Error before doing index work ([tablecmds.c:18842-18868](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18842-L18868)) | No. ATTACH is refused. |
| The parent has an exclusion index | `CompareIndexInfo` deliberately returns false for exclusion indexes, so ATTACH does not reuse an existing child exclusion index and instead attempts creation ([index.c:2622-2626](../../../raw/postgres-17/src/backend/catalog/index.c#L2622-L2626), [tablecmds.c:18958-18975](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18958-L18975)) | No. The old exclusion index remains even if another is created. |

The regression output confirms the keep-and-add behavior: four mismatched
indexes—hash, partial, expression, and a two-column index—remain after ATTACH,
and a fifth matching B-tree index is added
([indexing.sql:145-155](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L145-L155),
[indexing.out:260-283](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L260-L283)).

### What compatible means

`CompareIndexInfo` does not require textual DDL identity. It compares:

- uniqueness, `NULLS NOT DISTINCT`, access method, and key/total attribute
  counts
  ([index.c:2518-2534](../../../raw/postgres-17/src/backend/catalog/index.c#L2518-L2534));
- mapped column positions, key collations, and operator families
  ([index.c:2536-2570](../../../raw/postgres-17/src/backend/catalog/index.c#L2536-L2570));
- mapped expression trees and mapped partial-index predicates
  ([index.c:2572-2620](../../../raw/postgres-17/src/backend/catalog/index.c#L2572-L2620)); and
- no exclusion operators at all—the presence of either exclusion definition
  forces a non-match
  ([index.c:2622-2626](../../../raw/postgres-17/src/backend/catalog/index.c#L2622-L2626)).

Therefore, **matching expression and partial indexes can be reused**. The
four-index regression case above proves only that an expression or predicate
that differs from the parent's plain `(a)` index does not match. A separate
regression case shows a matching partial expression index being absorbed when
the parent partitioned index is created after the child index
([indexing.sql:210-222](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L210-L222),
[indexing.out:439-499](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L439-L499)).

The matcher contains no comparison of index names, tablespaces, relation
storage parameters, or the `pg_index.indoption` vector. “Equivalent” in the
ATTACH documentation means equivalent under this routine, not byte-for-byte
identical DDL
([index.c#CompareIndexInfo](../../../raw/postgres-17/src/backend/catalog/index.c#L2500-L2627),
[pg_index.h#FormData_pg_index](../../../raw/postgres-17/src/include/catalog/pg_index.h#L29-L62),
[alter_table.sgml#ATTACH-PARTITION](../../../raw/postgres-17/doc/src/sgml/ref/alter_table.sgml#L995-L1012)).

### What re-parenting changes

Reusing an index does not rewrite its tuples. `IndexSetParentIndex`:

1. inserts a `pg_inherits` link to the parent index;
2. sets the parent's `relhassubclass` hint;
3. sets the child index's `pg_class.relispartition`; and
4. adds partition dependencies on the parent index and the indexed child table
   ([indexcmds.c#IndexSetParentIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4300-L4428)).

If the parent index backs a constraint, `ConstraintSetParentConstraint` also
marks the child constraint inherited and adds partition dependencies. Those
dependencies prevent independent deletion
([pg_constraint.c#ConstraintSetParentConstraint](../../../raw/postgres-17/src/backend/catalog/pg_constraint.c#L814-L887)).
The existing index's OID is the input `partRelid`; no new index relation or
access-method build callback appears in this re-parenting function
([indexcmds.c:4304-4315](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4304-L4315)).

### Creation and multilevel partitions

When no direct match exists, `generateClonedIndexStmt` produces a
search-path-independent `IndexStmt`, and ATTACH passes it to `DefineIndex` with
the parent index and parent constraint OIDs
([tablecmds.c:18962-18974](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18962-L18974),
[indexcmds.c:1451-1486](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1451-L1486)).
For a stored table, `index_create` normally calls `index_build`, which invokes
the selected index access method's `ambuild` callback
([index.c:1248-1277](../../../raw/postgres-17/src/backend/catalog/index.c#L1248-L1277),
[index.c#index_build](../../../raw/postgres-17/src/backend/catalog/index.c#L2945-L3024)).
This is a new index build, not a rebuild or replacement of an existing index.

If the attached table is itself partitioned, the new intermediate partitioned
index has no storage. `DefineIndex` recursively scans each child for a match and
creates another index where none exists
([indexcmds.c:1249-1261](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1249-L1261),
[indexcmds.c:1315-1323](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1315-L1323),
[indexcmds.c:1535-1550](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1535-L1550)).
Unlike the direct ATTACH matcher, this recursive `DefineIndex` matcher does not
skip invalid candidates. It may absorb one, then marks each newly created
partitioned ancestor invalid when a descendant is invalid
([indexcmds.c:1372-1423](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1372-L1423),
[indexcmds.c:1490-1531](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1490-L1531)).
This distinction matters to the multilevel edge under
[Open Questions](#open-questions).

### Errors roll back the whole statement

Any error in compatibility checks, constraint creation, index creation, the
index build, or later partition validation aborts the current transaction
command
([postgres.c:4478-4491](../../../raw/postgres-17/src/backend/tcop/postgres.c#L4478-L4491),
[xact.c#AbortCurrentTransaction](../../../raw/postgres-17/src/backend/access/transam/xact.c#L3392-L3476)).
Physical relation files created in the transaction are registered for deletion
on abort
([storage.c#RelationCreateStorage](../../../raw/postgres-17/src/backend/catalog/storage.c#L108-L170),
[xact.c:2925-2930](../../../raw/postgres-17/src/backend/access/transam/xact.c#L2925-L2930)).
Consequently, ATTACH cannot commit only the earlier re-parent/create steps after
a later built-in error. It leaves neither a partially attached table nor a
committed partial set of newly created indexes.

### The closest look-alikes

Three core behaviors can look like ATTACH removed an index:

1. **A reused index loses independent droppability, not existence.** After
   re-parenting, `DROP INDEX` on the child errors because its parent index
   requires it
   ([indexing.out:171-180](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L171-L180),
   [alter_index.sgml#ATTACH-PARTITION](../../../raw/postgres-17/doc/src/sgml/ref/alter_index.sgml#L90-L104)).
2. **The fast-attach recipe manually drops a redundant CHECK constraint.** A
   CHECK constraint can prove the partition bound and avoid a validation scan;
   the documentation then tells the user to drop that constraint manually. It
   is not an index, and ATTACH does not execute the drop
   ([ddl.sgml:4289-4297](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4289-L4297)).
3. **A later parent DROP cascades.** Dropping the parent partitioned index drops
   its attached child indexes, and dropping a partition table drops its own
   index. These are separate commands after ATTACH
   ([indexing.sql:90-103](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L90-L103),
   [indexing.out:171-199](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L171-L199)).

### ALTER INDEX ATTACH PARTITION also does not drop

The companion `ALTER INDEX ... ATTACH PARTITION` path locks the child table,
parent table, and child index, checks that the index belongs to the correct
table partition, compares definitions, verifies any constraint, and then calls
`IndexSetParentIndex`
([tablecmds.c#ATExecAttachPartitionIdx](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19859-L20021)).
It refuses a different index when that table partition already has one attached
for the parent slot
([tablecmds.c#refuseDupeIndexAttach](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20024-L20042))
and rejects mismatched definitions instead of replacing either index
([tablecmds.c:19955-19972](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19955-L19972)).

At this pin, repeating the command for an index already attached to that same
parent attempts one validation pass if the parent remains invalid
([tablecmds.c:19902-19908](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19902-L19908),
[tablecmds.c:20005-20014](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20005-L20014),
[alter_index.sgml:100-104](../../../raw/postgres-17/doc/src/sgml/ref/alter_index.sgml#L100-L104)).
`validatePartitionedIndex` marks a parent valid only when its count of valid
attached indexes equals the table's partition count, then recursively checks an
invalid ancestor
([tablecmds.c#validatePartitionedIndex](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20045-L20145)).

### Custom code can issue a separate DROP

There are two supported interception boundaries around the core path:

- `ProcessUtility_hook` receives the utility statement before
  `standard_ProcessUtility`; a module normally chains to the standard path but
  is not forced to do only that
  ([utility.c#ProcessUtility](../../../raw/postgres-17/src/backend/tcop/utility.c#L489-L525)).
- `ddl_command_start` runs before `ProcessUtilitySlow` locks and executes the
  ALTER TABLE command, while `ddl_command_end` runs after the core command but
  in the same transaction. PostgreSQL explicitly performs command-counter
  increments so trigger changes are visible across those boundaries
  ([utility.c:1106-1114](../../../raw/postgres-17/src/backend/tcop/utility.c#L1106-L1114),
  [utility.c:1927-1931](../../../raw/postgres-17/src/backend/tcop/utility.c#L1927-L1931),
  [event_trigger.c:759-769](../../../raw/postgres-17/src/backend/commands/event_trigger.c#L759-L769),
  [event_trigger.c:807-817](../../../raw/postgres-17/src/backend/commands/event_trigger.c#L807-L817),
  [event_trigger.c#EventTriggerInvoke](../../../raw/postgres-17/src/backend/commands/event_trigger.c#L1063-L1116)).

Such custom code can submit its own `DROP INDEX`, including a parent drop that
cascades to newly attached child indexes. That drop is attributable to the
hook or trigger, not to `AttachPartitionEnsureIndexes`. If the enclosing ATTACH
later errors, the custom DDL rolls back with the same transaction
([event_trigger.c#EventTriggerEndCompleteQuery](../../../raw/postgres-17/src/backend/commands/event_trigger.c#L1215-L1237),
[postgres.c:4483-4487](../../../raw/postgres-17/src/backend/tcop/postgres.c#L4483-L4487)).

The two shipped contrib modules that assign `ProcessUtility_hook` do not add
this behavior: `sepgsql` performs its checks and chains to the next or standard
handler, while `pg_stat_statements` wraps and measures the same chain
([sepgsql/hooks.c#sepgsql_utility_command](../../../raw/postgres-17/contrib/sepgsql/hooks.c#L306-L393),
[sepgsql/hooks.c:476-480](../../../raw/postgres-17/contrib/sepgsql/hooks.c#L476-L480),
[pg_stat_statements.c#pgss_ProcessUtility](../../../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L1092-L1170),
[pg_stat_statements.c:477-481](../../../raw/postgres-17/contrib/pg_stat_statements/pg_stat_statements.c#L477-L481)).

### Avoiding an index build during ATTACH

If no compatible index exists, ATTACH can build one while holding the child
table's `AccessExclusiveLock`. PostgreSQL documents a lower-disruption pattern:
create the parent index with `CREATE INDEX ON ONLY`, build each child index with
`CREATE INDEX CONCURRENTLY`, and attach each child index with `ALTER INDEX ...
ATTACH PARTITION`; the parent becomes valid once all required child indexes are
valid and attached
([ddl.sgml:4318-4354](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4318-L4354)).

PostgreSQL 17 still rejects `CREATE INDEX CONCURRENTLY` directly on a
partitioned table, so concurrency is per stored partition
([indexcmds.c:709-730](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L709-L730)).
This workflow avoids an ATTACH-time build. It does not avoid an ATTACH-time
drop, because the core command has no drop branch.

### Changes since PostgreSQL 12

The no-drop rule is unchanged from PostgreSQL 12: the v17 function still ends
in re-parent or create. The changes below affect parser validation, which index
is reusable, recursive creation, validity propagation, event-trigger metadata,
locking, or the companion command—not deletion
([tablecmds.c#AttachPartitionEnsureIndexes](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18805-L18986)).
The first-major assignments use this checkout's own “Stamp HEAD as NNdevel”
commits as cycle boundaries; all listed commits are ancestors of the pin.

#### PostgreSQL 13

- Commit `e1551f96e6` (2019-12-18) replaced the separate attribute-number array
  and length with `AttrMap`; ATTACH now passes that object to
  `CompareIndexInfo`. This was a mechanical safety refactor
  ([tablecmds.c:18893-18898](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18893-L18898),
  [index.c:2505-2514](../../../raw/postgres-17/src/backend/catalog/index.c#L2505-L2514)).
- Commit `0b48f1335d` (2020-03-03, back-patched through v11) made `ALTER TABLE ...
  ATTACH PARTITION` reject a partitioned index cleanly instead of reaching an
  assertion; index attachment uses `ALTER INDEX` syntax
  ([parse_utilcmd.c#transformPartitionCmd](../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L3928-L3960),
  [indexing.sql:67-76](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L67-L76)).
- Commit `5b1c61e8b8` (2020-05-28, back-patched through v11) added
  `ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE` to one companion-command error;
  it did not change object state
  ([tablecmds.c:19945-19953](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19945-L19953)).

#### PostgreSQL 14

- Commit `a24ae3d7b9` (2021-03-25, back-patched through v11) replaced duplicated
  `pg_inherits` insertion code in `IndexSetParentIndex` with
  `StoreSingleInheritance`; the resulting catalog link is the same
  ([indexcmds.c:4317-4347](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4317-L4347)).
- Commit `e1ae40f381` (2021-03-17) changed only the foreign-table error detail to
  name the partitioned table
  ([tablecmds.c:18855-18863](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18855-L18863)).
- Adjacent commit `afc7e0ad55` (2020-09-01, back-patched to v11) introduced the
  explicit error for `DROP INDEX CONCURRENTLY` on a partitioned index. It
  concerns a separate DROP command
  ([tablecmds.c:1589-1598](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1589-L1598)).

#### PostgreSQL 15

- Commit `94aa7cc5f7` (2022-02-03) added `NULLS NOT DISTINCT`; equality of that
  flag became another reuse requirement. A mismatch causes create-and-keep,
  not replacement
  ([index.c:2518-2526](../../../raw/postgres-17/src/backend/catalog/index.c#L2518-L2526)).
- Adjacent commit `7b6ec86532` (2022-03-21, back-patched to v11) changed DROP of
  a partitioned index to lock all child tables before child indexes. It did not
  add a drop to ATTACH
  ([tablecmds.c:1600-1610](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1600-L1610)).

#### PostgreSQL 16

- Commit `43231423da` (2022-07-31) made ATTACH return the attached table's
  `ObjectAddress` to `ddl_command_end` collection
  ([tablecmds.c:5510-5520](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L5510-L5520),
  [tablecmds.c:5540-5549](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L5540-L5549),
  [test_ddl_deparse/alter_table.out:43-56](../../../raw/postgres-17/src/test/modules/test_ddl_deparse/expected/alter_table.out#L43-L56)).
- Commit `e6dbb48487` (2022-08-18, back-patched to v11) made recursive
  partitioned-index creation build both sides' `IndexInfo` with
  `BuildIndexInfo`, fixing missed expression/predicate matches that created
  duplicates
  ([indexcmds.c:1302-1312](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1302-L1312),
  [indexing.sql:210-222](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L210-L222)).
- Commit `27f5c712b2` (2023-03-25) added `DefineIndex.total_parts`; ATTACH passes
  `-1`, allowing the recursive path to count partitions for progress. It does
  not change match versus create
  ([indexcmds.c:1272-1296](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1272-L1296),
  [tablecmds.c:18967-18974](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18967-L18974)).
- Commit `fc55c7ff8d` (2023-06-28, back-patched through v11) added the direct
  `indisvalid` guard. The current regression proves that an invalid index on
  the table being attached is left unattached while a valid replacement tree
  is created
  ([tablecmds.c:18910-18917](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18910-L18917),
  [indexing.sql:891-911](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L891-L911)).

#### PostgreSQL 17.0

- Commit `cfc43aeb38` (2023-06-30, back-patched through v11) propagated an
  invalid recursively absorbed descendant through newly created partitioned
  indexes and added the missing command-counter increment
  ([indexcmds.c:1490-1531](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1490-L1531),
  [indexing.sql:913-937](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L913-L937)).
- Commit `8c852ba9a4` (2023-07-12) allowed partitioned exclusion constraints when
  each partition key is covered by an equality strategy
  ([indexing.sql:545-565](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L545-L565)).
  Once such a parent constraint exists, ATTACH's create branch can clone it,
  while `CompareIndexInfo` deliberately refuses to reuse an existing child
  exclusion index
  ([parse_utilcmd.c:1600-1667](../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1600-L1667),
  [index.c:2622-2626](../../../raw/postgres-17/src/backend/catalog/index.c#L2622-L2626),
  [tablecmds.c:18958-18975](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18958-L18975)).
- Commit `38ea6aa90e` (2023-07-14, back-patched through v11) changed
  `validatePartitionedIndex` to update a fresh syscache copy of `pg_index`,
  avoiding an invisible-tuple error in multi-command transactions
  ([tablecmds.c:20098-20118](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20098-L20118)).
- Commit `9f71e10d65` (2023-09-28, back-patched to all supported branches) made
  expression-versus-column mismatches explicit before indexing into the
  attribute map
  ([index.c:2536-2560](../../../raw/postgres-17/src/backend/catalog/index.c#L2536-L2560)).
- Commit `cee8db3f68` (2024-04-15) made a parent primary-key constraint require
  a child primary-key constraint rather than accepting a unique constraint
  ([tablecmds.c:18925-18944](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18925-L18944)).
- Commit `0cecc908e9` (2024-06-27, back-patched through v12) added
  `ShareUpdateExclusiveLock` before setting the parent index's
  `relhassubclass`, so concurrent catalog updates block instead of producing a
  tuple-concurrently-updated error
  ([indexcmds.c:4384-4389](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4384-L4389)).
- Commit `a15b0edb5d` (2024-07-15, back-patched through v17) restored the
  restricted search path after index functions or owner-context switches in
  `DefineIndex`, hardening the recursive creation branch
  ([indexcmds.c:1235-1243](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1235-L1243),
  [indexcmds.c:1335-1343](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1335-L1343)).

#### PostgreSQL 17.x stable

- Commit `fee8cb9473` (2024-10-05, back-patched to v17) replaced ad hoc
  recursive `IndexStmt` copying with `generateClonedIndexStmt`. This preserves
  references to non-search-path expressions, collations, and operator classes
  and stops propagating the parent index comment to children
  ([indexcmds.c:1451-1486](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1451-L1486),
  [contrib/seg/partition.sql:1-35](../../../raw/postgres-17/contrib/seg/sql/partition.sql#L1-L35)).
- Commit `becf6d2696` (2026-04-22, back-patched through v14) made re-attachment
  of an already-attached index retry parent validation; commit `1f0a58a0c2`
  documented it. PostgreSQL 17.10's release notes list the change
  ([tablecmds.c:19902-20014](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19902-L20014),
  [release-17.sgml:696-700](../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L696-L700),
  [indexing.sql:249-306](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L249-L306)).

None of these commits adds an index-deletion call to the table or index ATTACH
routines.

### Test coverage and gaps

The current core regression suite directly covers:

- automatic creation when the attached table has no index
  ([indexing.sql:55-65](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L55-L65));
- reuse of existing compatible indexes without duplicates
  ([indexing.sql:192-209](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L192-L209));
- retention of mismatched indexes plus creation of a match
  ([indexing.out:260-283](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L260-L283));
- direct invalid-index rejection and recursive validity propagation
  ([indexing.sql:891-937](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L891-L937));
- attached-index drop restrictions and parent-drop cascade
  ([indexing.sql:90-103](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L90-L103));
- companion-command mismatch, duplicate-slot, and repeated-attach cases
  ([indexing.sql:117-140](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L117-L140)); and
- repeated attach validating all-valid parents but not a parent with another
  invalid child
  ([indexing.sql:249-306](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L249-L306)).

The source tree has no direct upstream test for a concurrent DROP racing table
ATTACH, custom event-trigger or utility-hook deletion, direct table-ATTACH reuse
of a matching partial/expression index (the cited case creates the parent index
after ATTACH), an existing exclusion-index non-match, or the multilevel
invalid-leaf edge in [Open Questions](#open-questions). The
v17 same-constraint-type commit originally added a primary-key-versus-unique
regression case, but the later structural not-null revert removed that test
block; the implementation check remains. These absences were confirmed across
`indexing.sql`, `constraints.sql`, isolation specs, TAP tests, and
`test_ddl_deparse`; they do not create a source ambiguity about the core
no-drop branch
([indexing.sql](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L1-L980),
[constraints.sql](../../../raw/postgres-17/src/test/regress/sql/constraints.sql#L1-L645),
[test_ddl_deparse/alter_table.out](../../../raw/postgres-17/src/test/modules/test_ddl_deparse/expected/alter_table.out#L43-L56)).

### Build, generated-file, access-method, and contrib boundaries

The ATTACH syntax is Bison input in `gram.y`; the Make and Meson builds generate
the parser outputs from it
([parser/Makefile](../../../raw/postgres-17/src/backend/parser/Makefile#L42-L66),
[parser/meson.build](../../../raw/postgres-17/src/backend/parser/meson.build#L24-L41)).
`AT_AttachPartition`, `AlterTableCmd`, and `PartitionCmd` are ordinary parser
node definitions, so the current feature requires no extension SQL or fmgr
entry point
([parsenodes.h#AlterTableType](../../../raw/postgres-17/src/include/nodes/parsenodes.h#L2380-L2440),
[parsenodes.h#PartitionCmd](../../../raw/postgres-17/src/include/nodes/parsenodes.h#L950-L959)).

The catalog state uses `pg_class.relispartition`/`relhassubclass`,
`pg_inherits`, and `pg_index.indisvalid` plus its definition fields
([pg_class.h#FormData_pg_class](../../../raw/postgres-17/src/include/catalog/pg_class.h#L74-L120),
[pg_inherits.h#FormData_pg_inherits](../../../raw/postgres-17/src/include/catalog/pg_inherits.h#L27-L48),
[pg_index.h#FormData_pg_index](../../../raw/postgres-17/src/include/catalog/pg_index.h#L24-L62)).
Those catalog headers include generated `_d.h` files; Make and Meson list all
three as `genbki.pl` inputs. No ATTACH-specific catalog or generated header is
introduced
([catalog/Makefile](../../../raw/postgres-17/src/include/catalog/Makefile#L17-L28),
[catalog/Makefile#generated-headers](../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143),
[catalog/meson.build](../../../raw/postgres-17/src/include/catalog/meson.build#L7-L15),
[catalog/meson.build#generated_catalog_headers](../../../raw/postgres-17/src/include/catalog/meson.build#L122-L145)).

Re-parenting is access-method independent because `IndexSetParentIndex` only
changes catalogs and dependencies. Creation reaches the chosen AM's `ambuild`;
a custom AM can therefore participate only when ATTACH must build a new stored
index, not when ATTACH reuses one
([indexcmds.c#IndexSetParentIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4300-L4428),
[amapi.h#IndexAmRoutine](../../../raw/postgres-17/src/include/access/amapi.h#L101-L108),
[index.c:3021-3024](../../../raw/postgres-17/src/backend/catalog/index.c#L3021-L3024)).
Creating an index also invokes the object-access post-create hook before the
build. Re-parenting an existing index does not invoke an object-access hook in
`IndexSetParentIndex`
([index.c:1218-1226](../../../raw/postgres-17/src/backend/catalog/index.c#L1218-L1226),
[indexcmds.c#IndexSetParentIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4300-L4428)).
Shipped contrib coverage is adjacent rather than ATTACH-specific or a second
implementation: `btree_gist` tests creating partitioned exclusion constraints,
and `seg` tests `CREATE INDEX` propagation for non-`pg_catalog` functions,
collations, and operator classes
([btree_gist/partitions.sql](../../../raw/postgres-17/contrib/btree_gist/sql/partitions.sql#L1-L39),
[seg/partition.sql](../../../raw/postgres-17/contrib/seg/sql/partition.sql#L1-L35)).

## Context Reviewed

- Navigation only: `wiki/versions.md`, `wiki/index.md`, `wiki/v17/index.md`, and
  the last 20 `wiki/log.md` entries.
- Pinned source: `raw/postgres-17/` at
  `54eeefaedbee0385529f3edf321bb99e49232aaa` (PostgreSQL 17.10).
- Parser and utility path: `gram.y`, `parsenodes.h`, `parse_utilcmd.c`,
  `utility.c`, and ALTER TABLE phase dispatch in `tablecmds.c`.
- Core index path: `ATExecAttachPartition`, `AttachPartitionEnsureIndexes`,
  `ATExecAttachPartitionIdx`, `refuseDupeIndexAttach`,
  `validatePartitionedIndex`, `DefineIndex`, `IndexSetParentIndex`,
  `CompareIndexInfo`, `index_create`, and `index_build`.
- Structures and catalogs: `RelationData` fields used at call sites,
  `IndexInfo`, `AttrMap`, `PartitionDesc`, `pg_class`, `pg_index`,
  `pg_inherits`, `pg_constraint`, and `pg_depend` operations.
- Error, lock, and concurrency paths: `LockConflicts`, `RemoveRelations`,
  `RangeVarCallbackForDropRelation`, transaction abort, and pending relation
  deletion.
- Custom boundaries: `ProcessUtility_hook`, event triggers, object-access
  post-create behavior, custom `IndexAmRoutine`, contrib `sepgsql`,
  `pg_stat_statements`, `btree_gist`, `seg`, and `postgres_fdw` ATTACH tests.
- Documentation and tests: `alter_table.sgml`, `alter_index.sgml`, `ddl.sgml`,
  `release-17.sgml`, `indexing.sql/.out`, `constraints.sql/.out`, isolation/TAP
  searches, and `test_ddl_deparse`.
- History: inspected each listed commit body and patch, checked ancestry to the
  pin, and bracketed first major releases with the checkout's 13devel through
  17devel stamp commits.
- Exact-pin execution: a disposable PostgreSQL 17.10 cluster confirmed direct
  reuse of matching expression and partial indexes, exclusion-index non-reuse,
  rollback after a failed auto-build, a `ddl_command_start` trigger dropping an
  extra index, a `ddl_command_end` trigger dropping the parent and cascading to
  the new child index, and the multilevel invalid-index state in Open Questions.
  The server was stopped.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Core table ATTACH is match-or-create, with no drop branch | [tablecmds.c#AttachPartitionEnsureIndexes](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18805-L18986) |
| Table ATTACH calls index reconciliation after partition catalog wiring | [tablecmds.c:18686-18710](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18686-L18710) |
| Compatible direct index is re-parented | [tablecmds.c:18900-18954](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18900-L18954) |
| No direct match creates a clone | [tablecmds.c:18958-18975](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18958-L18975) |
| Direct invalid and already-attached candidates are skipped | [tablecmds.c:18910-18917](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18910-L18917) |
| Compatible partial/expression indexes can match | [index.c:2572-2620](../../../raw/postgres-17/src/backend/catalog/index.c#L2572-L2620) |
| Exclusion indexes never match in `CompareIndexInfo` | [index.c:2622-2626](../../../raw/postgres-17/src/backend/catalog/index.c#L2622-L2626) |
| Constraint-backed reuse requires a child constraint of the same type | [tablecmds.c:18925-18950](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18925-L18950) |
| Re-parenting changes `pg_inherits`, `pg_class`, and partition dependencies | [indexcmds.c#IndexSetParentIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4300-L4428) |
| Constraint re-parenting prevents independent deletion | [pg_constraint.c#ConstraintSetParentConstraint](../../../raw/postgres-17/src/backend/catalog/pg_constraint.c#L814-L887) |
| New stored indexes invoke the AM build callback | [index.c:1248-1277](../../../raw/postgres-17/src/backend/catalog/index.c#L1248-L1277), [index.c:3021-3024](../../../raw/postgres-17/src/backend/catalog/index.c#L3021-L3024) |
| Recursive creation may absorb invalid descendants and propagates invalidity through new ancestors | [indexcmds.c:1372-1423](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1372-L1423), [indexcmds.c:1490-1531](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1490-L1531) |
| Parent/child locks serialize concurrent DROP | [tablecmds.c:18517-18600](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18517-L18600), [tablecmds.c:1558-1568](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1558-L1568), [lock.c#LockConflicts](../../../raw/postgres-17/src/backend/storage/lmgr/lock.c#L59-L104) |
| Errors abort all statement changes | [postgres.c:4478-4491](../../../raw/postgres-17/src/backend/tcop/postgres.c#L4478-L4491), [xact.c#AbortCurrentTransaction](../../../raw/postgres-17/src/backend/access/transam/xact.c#L3392-L3476) |
| Attached child cannot be dropped alone; parent drop cascades | [alter_index.sgml:90-104](../../../raw/postgres-17/doc/src/sgml/ref/alter_index.sgml#L90-L104), [indexing.out:171-199](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L171-L199) |
| Companion command re-parents, validates, or errors; it does not replace | [tablecmds.c#ATExecAttachPartitionIdx](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19859-L20021) |
| Event triggers and utility hooks can execute independent custom behavior | [utility.c#ProcessUtility](../../../raw/postgres-17/src/backend/tcop/utility.c#L489-L525), [event_trigger.c:721-817](../../../raw/postgres-17/src/backend/commands/event_trigger.c#L721-L817) |
| Tests retain four mismatched indexes and add one | [indexing.out:260-283](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L260-L283) |
| v17.10 repeated-attach validation behavior | [tablecmds.c:19902-20014](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19902-L20014), [release-17.sgml:696-700](../../../raw/postgres-17/doc/src/sgml/release-17.sgml#L696-L700) |
| Catalog and parser generated-file boundaries | [catalog/Makefile:17-28](../../../raw/postgres-17/src/include/catalog/Makefile#L17-L28), [catalog/Makefile:126-143](../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143), [parser/meson.build:38-41](../../../raw/postgres-17/src/backend/parser/meson.build#L38-L41) |

## Open Questions

- **Multilevel ATTACH can leave the existing top parent index valid above an
  invalid new child.** The direct matcher skips an invalid index on the table
  being attached, but a recursive `DefineIndex` call may absorb an invalid leaf
  index and mark only the newly created intermediate partitioned index invalid
  ([tablecmds.c:18910-18917](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18910-L18917),
  [indexcmds.c:1372-1423](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1372-L1423),
  [indexcmds.c:1490-1531](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1490-L1531)).
  In an exact-pin smoke test, a valid top parent expression index stayed
  `indisvalid = true` after ATTACH created and linked an invalid intermediate
  index above a failed-CIC leaf. `validatePartitionedIndex` describes the
  intended validation rule as one valid attached index per table partition
  ([tablecmds.c:20063-20098](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20063-L20098)),
  but ATTACH does not call it for this newly invalid child. The current upstream
  tests separately cover direct invalid-index skipping and recursive validity
  propagation during `CREATE INDEX`, not their combination during table
  ATTACH
  ([indexing.sql:891-937](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L891-L937)).
  Whether the top parent's retained validity is intentional needs upstream
  confirmation. It does not create an index-drop path.

## Source References

- [gram.y#partition_cmd](../../../raw/postgres-17/src/backend/parser/gram.y#L2311-L2325)
- [parsenodes.h#PartitionCmd](../../../raw/postgres-17/src/include/nodes/parsenodes.h#L950-L959)
- [utility.c#ProcessUtility](../../../raw/postgres-17/src/backend/tcop/utility.c#L489-L525)
- [utility.c#T_AlterTableStmt](../../../raw/postgres-17/src/backend/tcop/utility.c#L1268-L1321)
- [tablecmds.c#ATExecAttachPartition](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18491-L18801)
- [tablecmds.c#AttachPartitionEnsureIndexes](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18805-L18986)
- [tablecmds.c#ATExecAttachPartitionIdx](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19859-L20021)
- [tablecmds.c#validatePartitionedIndex](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20045-L20145)
- [indexcmds.c#DefineIndex-partitions](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L1249-L1550)
- [indexcmds.c#IndexSetParentIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4300-L4428)
- [index.c#CompareIndexInfo](../../../raw/postgres-17/src/backend/catalog/index.c#L2500-L2627)
- [index.c#index_build](../../../raw/postgres-17/src/backend/catalog/index.c#L2945-L3040)
- [pg_constraint.c#ConstraintSetParentConstraint](../../../raw/postgres-17/src/backend/catalog/pg_constraint.c#L814-L887)
- [event_trigger.c#DDL-command-triggers](../../../raw/postgres-17/src/backend/commands/event_trigger.c#L721-L817)
- [alter_table.sgml#ATTACH-PARTITION](../../../raw/postgres-17/doc/src/sgml/ref/alter_table.sgml#L995-L1012)
- [alter_index.sgml#ATTACH-PARTITION](../../../raw/postgres-17/doc/src/sgml/ref/alter_index.sgml#L88-L104)
- [ddl.sgml#partition-maintenance](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4289-L4354)
- [indexing.sql#partitioned-indexes](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L55-L306)
- [indexing.sql#invalid-indexes](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L891-L937)
- [indexing.out#partitioned-indexes](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L78-L283)

## Navigation

- [v17 index](../index.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
- [v17: CREATE INDEX CONCURRENTLY](create-index-concurrently.md)
- [v12: Can ALTER TABLE ... ATTACH PARTITION Drop Indexes?](../../v12/questions/attach-partition-index-drops.md)
