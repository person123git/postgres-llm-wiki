---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# DROP SCHEMA After Relation Files Were Emptied in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The outcome depends on CASCADE and intact catalogs](#the-outcome-depends-on-cascade-and-intact-catalogs)
  - [What was emptied, and what DROP still needs](#what-was-emptied-and-what-drop-still-needs)
  - [Decision map](#decision-map)
  - [From the command to the deletion list](#from-the-command-to-the-deletion-list)
  - [Deleting objects does not scan their data](#deleting-objects-does-not-scan-their-data)
  - [Cross-schema dependencies and locks](#cross-schema-dependencies-and-locks)
  - [Commit removes storage by filename](#commit-removes-storage-by-filename)
  - [State changes and rollback](#state-changes-and-rollback)
  - [Cases that change the result](#cases-that-change-the-result)
  - [Serialization branches](#serialization-branches)
  - [Catalog damage branches](#catalog-damage-branches)
  - [Measured results](#measured-results)
  - [Build boundaries and existing tests](#build-boundaries-and-existing-tests)
  - [Causal summary](#causal-summary)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Last run](#last-run)
  - [Stages](#stages)
  - [Non-default settings in the measurement](#non-default-settings-in-the-measurement)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Follow AGENTS.md. In PostgreSQL 12, how does DROP SCHEMA work, and what happens if DROP SCHEMA is executed on a schema that has had all its objects zeroed using `cat /dev/null > file.txt`?

## Answer

### The outcome depends on CASCADE and intact catalogs

**Emptying the files does not make the schema empty.** Plain `DROP SCHEMA s` defaults to `RESTRICT`. If ordinary tables still have their [catalog](../../../glossary.md#catalog) entries and [dependencies](../../../glossary.md#dependency) on the [schema](../../../glossary.md#schema), PostgreSQL refuses the drop even when their files contain no bytes. The decision follows dependency records, not row counts or file lengths. [gram.y#opt_drop_behavior](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568) [heap.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408) [dependency.c#reportDependentObjects](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1093-L1203)

**The refusal is not immediate.** The dependency walk locks every dependent object it finds before the `RESTRICT` check runs, and the lock it takes on a [relation](../../../glossary.md#relation) is [AccessExclusiveLock](../../../glossary.md#accessexclusivelock). It requests these locks one at a time, as its `pg_depend` scan reaches each object, so a plain `DROP SCHEMA` can first wait behind concurrent users of the emptied tables. While it waits for one table, new lock requests on that table from sessions that hold no lock on it join the [lock queue](../../../glossary.md#lock-queue) behind it. Tables the walk has not reached yet carry no lock from the drop, so new sessions can still lock them without waiting for the drop, and the drop may later have to wait for those sessions too. Steps 3 and 4 of [From the command to the deletion list](#from-the-command-to-the-deletion-list) trace this, and the pinned build reproduced both the queued request and the immediate grant; see [Measured results](#measured-results). [dependency.c#scan-locks-each-dependent](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L861-L901) [dependency.c#recurse-after-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L960-L988) [dependency.c#report-after-walk](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L401-L429) [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) [lock.c#join-wait-queue](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L976-L985)

**With explicit `CASCADE`, empty files of user relations alone do not prevent the ordinary drop path from succeeding.** Several conditions must still hold: usable catalogs, the required locks and permissions, enough free shared memory for a shared [lock table](../../../glossary.md#heavyweight-lock) entry for every lock the drop holds, ordinary built-in objects, no failing added trigger or [hook](../../../glossary.md#hook), and no serialization failure in a [`SERIALIZABLE`](../../../glossary.md#isolation-level) dropping transaction. Under them, PostgreSQL deletes the metadata and schedules the files for removal without scanning the affected tables or validating their index [pages](../../../glossary.md#page). A build of the pinned PostgreSQL **12.2** checkout reproduced this with every file of a fixture schema emptied, [TOAST](../../../glossary.md#toast) tables, indexes, sequences and numbered segments included; see [Measured results](#measured-results). It is not a guarantee for a database whose catalogs were also damaged. [lock.c#out-of-shared-memory](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L956-L968) [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1998) [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173)

**`CASCADE` also changes surviving tables — and can drop dependent relations — outside the schema.** A foreign key belongs to its referencing table but has a normal dependency on the referenced columns. So when the referenced table is in the dropped schema, `CASCADE` drops a constraint of a table that can live in another schema. The deletion takes `AccessExclusiveLock` on that table, which survives without its foreign key. `CHECK` and `EXCLUDE` constraints, indexes, triggers, non-`SELECT` rewrite rules, column defaults and policies of outside tables that depend on schema objects, and outside columns whose type, [collation](../../../glossary.md#collation) or generation expression depends on schema objects, are removed from their surviving tables too, while outside relations that themselves depend on schema objects (such as partitions, inheritance children, typed tables, views and materialized views) are dropped outright. [Cross-schema dependencies and locks](#cross-schema-dependencies-and-locks) lists the cases. [pg_constraint.c#normal-dependency-on-referenced-columns](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L274-L298) [pg_constraint.c#RemoveConstraintById-lock](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L572-L580) [rewriteRemove.c#RemoveRewriteRuleById-lock](../../../../raw/postgres-12/src/backend/rewrite/rewriteRemove.c#L73) [dependency.c#doDeletion-column](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1356-L1358) [heap.c#RemoveAttributeById](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1613-L1717) [index.c#index-drop-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L2047-L2050)

### What was emptied, and what DROP still needs

This answer reads the shell command in the question as truncating the named file to zero length, not as overwriting every original byte with a zero while keeping the length. That is shell behavior, which the pinned PostgreSQL checkout cannot establish, so it is an assumption of this page, which the measurement observed on its own platform; see [Open Questions](#open-questions). Here, `file.txt` means an actual relation-file pathname. A schema is a namespace recorded in `pg_namespace`, not a directory that contains all its objects. Relation paths use database, [tablespace](../../../glossary.md#tablespace) and physical relation identifiers (the [relfilenode](../../../glossary.md#relfilenumber)), not the schema name. [pg_namespace.h#FormData_pg_namespace](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L24-L44) [relpath.c#GetRelationPath](../../../../raw/postgres-12/src/common/relpath.c#L129-L205)

“All its objects” therefore needs a boundary. Tables, indexes, sequences, materialized views and TOAST tables can have physical relation storage. Views, foreign tables, [partitioned tables](../../../glossary.md#declarative-partitioning), partitioned indexes and functions do not each have a corresponding data file to empty. A table can also have multiple [forks](../../../glossary.md#fork) and numbered [segments](../../../glossary.md#storage-manager). Emptying one pathname changes that file, not all those related files or the catalog rows describing the object. The answer assumes the user-object files were emptied and the catalogs were left usable. [heap.c#storage-bearing-kinds](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1943-L1952) [index.c#drop-storage](../../../../raw/postgres-12/src/backend/catalog/index.c#L2199-L2203) [dependency.c#object-dispatch](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1341-L1378) [functioncmds.c#RemoveFunctionById](../../../../raw/postgres-12/src/backend/commands/functioncmds.c#L1180-L1206) [storage.sgml#forks-and-segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L199-L243)

| Information or state | Origin and freshness | How deletion uses it |
|---|---|---|
| Schema identity and owner | Stored `pg_namespace` row, accessed through catalog lookup | Resolves the name, locks the schema object, then checks ownership: a superuser passes, and so does any role with the owner's privileges, which includes inheriting members of the owning role ([role membership](../../../glossary.md#role-membership)). It is not inferred from disk files. [namespace.c#get_namespace_oid](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3030-L3047) [objectaddress.c#lock-after-lookup](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1076-L1088) [aclchk.c#pg_namespace_ownercheck](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L4927-L4951) [acl.c#has_privs_of_role](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4907-L4922) |
| Object addresses and dependency edges | `ObjectAddress` identifies a catalog, object and possible subobject; stored `pg_depend` rows connect them | Builds and orders the objects to delete; determines whether `RESTRICT` blocks deletion. [objectaddress.h#ObjectAddress](../../../../raw/postgres-12/src/include/catalog/objectaddress.h#L21-L29) [pg_depend.h#FormData_pg_depend](../../../../raw/postgres-12/src/include/catalog/pg_depend.h#L36-L66) [dependency.c#dependency-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L829-L866) |
| Relation descriptor and physical identity | Catalog-derived [relcache](../../../glossary.md#relcache) descriptor. `RelationInitPhysicalAddr()` fills `rd_node` from the [`pg_class`](../../../glossary.md#pg_class) row's tablespace and `relfilenode`, or from the [relation mapper](../../../glossary.md#relation-mapper) for a mapped catalog | `RelationDropStorage` copies the identifier into a pending-delete entry. An external file truncation does not edit this catalog mapping. [relcache.c#RelationBuildDesc](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1055-L1084) [relcache.c:1227](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1227) [relcache.c#RelationInitPhysicalAddr](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1271-L1331) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L148-L172) |
| File existence and length | Current filesystem state; independently changed by the shell command | Commit-time cleanup opens/truncates or unlinks paths. It does not ask how many table rows survived. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) |
| Pending deletion and transaction outcome | In-memory `PendingRelDelete`, including `atCommit` and nesting level | Distinguishes commit-time deletion of existing storage from abort-time cleanup of newly created storage. [storage.c#PendingRelDelete](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462) |

### Decision map

```text
DROP SCHEMA s [CASCADE | RESTRICT]
  |
  +-- Look up the schema name and lock the schema object [A]
  |     +-- missing -> notice with IF EXISTS, otherwise error
  |     +-- schema lock held elsewhere -> wait
  |
  +-- Check ownership; superuser or a role with the owner's privileges [A]
  |     +-- any other role -> error 42501
  |
  +-- Walk pg_depend; lock each dependent as the scan reaches it [B]
  |     +-- dependent locked elsewhere -> wait, under RESTRICT too
  |     +-- dependents not reached yet -> not locked while it waits
  |     +-- shared lock table full -> error 53200, under RESTRICT too [B]
  |     +-- uncached catalog-file read -> can fail; see catalog branches [B]
  |     +-- dependency rows absent without an error -> list incomplete
  |
  +-- Default RESTRICT + blocking dependencies -> error 2BP01, no deletion [B]
  |     (every dependent found is already locked; the abort releases them)
  |
  +-- Explicit CASCADE, or no blocking dependencies [B]
        |
        +-- Delete dependent objects, then schema metadata [C]
        |     +-- existing user-file contents are not scanned for deletion
        |     +-- schedule storage removal using physical relation identifiers
        |     +-- deletion routines lock more tables, some outside the schema
        |     +-- outside tables can lose dependent objects or be dropped
        |
        +-- Transaction aborts -> do not execute these commit-only deletes [D]
        |                         external file truncation is not undone
        |
        +-- Transaction commits -> discard buffers and clean up paths [D]
              +-- first non-temp main segment: truncate, defer unlink [E]
              +-- first file of every other fork: unlink [E]
              +-- each fork whose first step succeeded: unlink .1, .2, ... [E]
              +-- zero length does not stop numbered-segment traversal
              +-- missing path / filesystem failure can limit cleanup
```

- **A:** [dropcmds.c#RemoveObjects](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L49-L123), [objectaddress.c#lock-after-lookup](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1076-L1088), [dropcmds.c#ownership-check](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L104-L109), [aclchk.c#pg_namespace_ownercheck](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L4927-L4951), [acl.c#has_privs_of_role](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4907-L4922), [aclchk.c#not-owner-error](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L3576-L3577), [aclchk.c#insufficient-privilege](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L3623-L3625), [errcodes.txt:332](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L332). The schema lock is taken inside `get_object_address()`, before the ownership check. Step 2 explains which roles have the owner's privileges.
- **B:** [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437), [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901), [dependency.c#recurse-after-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L960-L988), [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554), [lock.c#join-wait-queue](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L976-L985), [dependency.c#RESTRICT-error](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1113-L1203). The `RESTRICT` error carries [SQLSTATE](../../../glossary.md#sqlstate) `2BP01` (dependent objects still exist) [errcodes.txt:271](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L271). A lock request that finds no free entry in the shared lock table fails with `out of shared memory`, SQLSTATE `53200` [lock.c#out-of-shared-memory](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L956-L968) [errcodes.txt:400](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L400); [Cases that change the result](#cases-that-change-the-result) explains the table's size. The two catalog-damage branches are explained, with their evidence and cache/setting conditions, in [Catalog damage branches](#catalog-damage-branches). The incomplete-list branch is a deduction from this catalog-driven enumeration, not a measured damaged-catalog outcome.
- **C:** [dependency.c#deletion-order](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L974-L1010), [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1998), [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266), [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L214-L235), [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1525-L1530), [pg_constraint.c#RemoveConstraintById-lock](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L572-L580). The extra deletion-time locks, and what surviving tables lose, are listed under [Cross-schema dependencies and locks](#cross-schema-dependencies-and-locks).
- **D:** [storage.c#pending-delete-lifecycle](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65), [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462), [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495).
- **E:** [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359), [sync.c#checkpoint-cycle](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214).

### From the command to the deletion list

1. **Parse the requested behavior.** The [grammar](../../../glossary.md#grammar) makes a `DropStmt` with the names, object kind, `missing_ok` and `behavior`. The omitted behavior is `DROP_RESTRICT`; writing `CASCADE` selects `DROP_CASCADE`. [`standard_ProcessUtility()`](../../../glossary.md#utility-command) passes `DROP SCHEMA` to `ProcessUtilitySlow()` because [event triggers](../../../glossary.md#event-trigger) support schema objects. From there the statement takes `ExecDropStmt` → `RemoveObjects`. [gram.y#DropStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6184-L6203) [gram.y#SCHEMA](../../../../raw/postgres-12/src/backend/parser/gram.y#L6304-L6313) [gram.y#opt_drop_behavior](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568) [utility.c#standard_ProcessUtility-drop](../../../../raw/postgres-12/src/backend/tcop/utility.c#L834-L844) [event_trigger.c#EventTriggerSupportsObjectType](../../../../raw/postgres-12/src/backend/commands/event_trigger.c#L1097-L1166) [utility.c#drop-dispatch](../../../../raw/postgres-12/src/backend/tcop/utility.c#L1590-L1594) [utility.c#ExecDropStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L1731-L1751)
2. **Resolve, lock and authorize each target schema.** For each schema named in `DROP SCHEMA name [, ...]`, `RemoveObjects()` asks `get_object_address()` for the schema with `AccessExclusiveLock`. That function maps the name to `NamespaceRelationId` and the schema's [OID](../../../glossary.md#oid), then locks the schema object. Only after the lock is granted does `RemoveObjects()` check ownership. A schema belongs to no namespace, so `get_object_namespace()` returns none and `check_object_ownership()` calls `pg_namespace_ownercheck()`. That function passes a superuser at once. For any other role it returns `has_privs_of_role(role, owner)`: the owner passes, and so does a member of the owning role that inherits its privileges. Every other role gets error `42501`, "must be owner of schema". The `DROP SCHEMA` reference page names only the owner and superusers; [Open Questions](#open-questions) records that difference. That check covers only the named schemas. The dependency walk and its deletion loop make no ownership or privilege check, and neither do the core removal routines this page traces, such as `RemoveConstraintById()` and `RemoveTriggerById()`. So the schema owner can cascade to contained objects owned by others, as the reference page states, and also to dependent objects of other roles outside the schema, such as another owner's foreign key that references a table in the schema; only added code, such as a hook or an event trigger, can reject that. `IF EXISTS` changes missing-schema handling to a notice; it does not ignore dependencies or general failures. [dropcmds.c#get_object_address-call](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L71-L75) [objectaddress.c#schema-address](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1186-L1190) [objectaddress.c#lock-after-lookup](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1076-L1088) [dropcmds.c#ownership-check](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L104-L109) [objectaddress.c#namespace-has-no-namespace](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L341-L352) [objectaddress.c#get_object_namespace](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2493-L2504) [objectaddress.c#schema-ownership](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2337-L2341) [aclchk.c#pg_namespace_ownercheck](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L4927-L4951) [acl.c#has_privs_of_role](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4907-L4922) [acl.c#roles_has_privs_of](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4732-L4817) [aclchk.c#not-owner-error](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L3576-L3577) [dropcmds.c#RemoveObjects](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L49-L127) [drop_schema.sgml#ownership](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L35-L39) [dependency.c#deleteOneObject](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1233-L1331) [pg_constraint.c#RemoveConstraintById](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L552-L631) [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1496-L1568) [dependency.c#object-drop-hook](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1240-L1242)
3. **Walk dependencies, locking as the walk goes.** After looking up all named schemas, `RemoveObjects()` passes the combined `objects` list to a single `performMultipleDeletions()` call, which opens `pg_depend`, takes each target schema's deletion lock again and calls `findDependentObjects()` with `objects` as `pendingObjects`. That function recursively follows references and puts dependents before their referenced object. For each object it scans `pg_depend` for that object's direct dependents, in index order. As the scan reaches each dependent, `AcquireDeletionLock()` takes `AccessExclusiveLock` on the whole relation when `classId == RelationRelationId` (including a column entry with `objectSubId != 0`), or an `AccessExclusiveLock` database-object lock on `(classId, objectId, 0)` for other catalog classes such as constraints, triggers, rules, column defaults and policies, and then rechecks that the `pg_depend` row is still live. Only after the scan does the function sort the dependents and recurse into them. A lock request that has to wait therefore stops the walk where it is: the drop keeps the locks it has taken so far, waits for one, and has not yet requested the rest. It gives a lock back on purpose only in a few branches: when an object it waited for was deleted meanwhile, when it moves from an internal or extension member to that member's owning object, and when a named target's owning object is itself one of the named targets. The walk is not bounded by the schema, so it also locks dependents that live in other schemas. [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L368-L437) [dependency.c#scan-locks-each-dependent](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L861-L901) [dependency.c#recurse-after-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L960-L988) [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901) [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) [dependency.c#deletion-order](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L974-L1010) [dependency.c#pending-owner-release](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L640-L648) [dependency.c#redirect-and-owner-recheck](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L682-L702)
4. **Apply RESTRICT or CASCADE only after the walk.** `reportDependentObjects()` runs once the list is complete. The ordinary table-to-schema edge is `DEPENDENCY_NORMAL`, so a table with no data still blocks default `RESTRICT` (reporting `"cannot drop schema s because other objects depend on it"` when exactly one named schema was found, or `"cannot drop desired object(s) because other objects depend on them"` when more were, `objects->numrefs > 1`; a name that `IF EXISTS` skipped is not counted [dropcmds.c#missing-skipped](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L82-L87)). Automatic/internal/partition/[extension](../../../glossary.md#extension) dependency flags can allow dependent cleanup even under `RESTRICT`; this does not make an ordinary table's schema edge automatic. [dependency.c#report-after-walk](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L401-L429) [dependency.c#dependency-flags](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1113-L1203) [heap.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408)

Because the check comes last, a refused `RESTRICT` drop has already taken the deletion lock on every object the walk found. If another session holds a conflicting lock on one of them, the drop waits for that lock before it can report the dependencies. The `RESTRICT` error then aborts the transaction, and the abort releases the locks. [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901) [dependency.c#RESTRICT-error](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1187-L1203) [xact.c#abort-lock-release](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2714-L2716)

While an `AccessExclusiveLock` request from the drop waits, later requests on the same object follow these branches. Which branch applies depends on the [lock modes](../../../glossary.md#lock-mode) that the *requesting* session already holds on the object, not on the drop's. `AccessExclusiveLock` conflicts with every mode, including the weakest. [lock.c#join-wait-queue](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L976-L985) [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L99-L103) [lmgr/README#lock-acquisition-rules](../../../../raw/postgres-12/src/backend/storage/lmgr/README#L362-L381)

| Requesting session's state | Lock decision | Consequence |
|---|---|---|
| Holds no lock on the object | Its request conflicts with the waiting drop and joins the end of the queue | A later `SELECT` can wait even though it would be compatible with the current reader. Measured: before the drop arrived, a baseline session's `ACCESS SHARE` request was granted at once; while the drop waited, a second new session's request timed out with `55P03` under its 1-second [`lock_timeout`](../../../glossary.md#statement_timeout-and-lock_timeout); see [Measured results](#measured-results). [lock.c#join-wait-queue](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L976-L985) [lmgr/README#lock-acquisition-rules](../../../../raw/postgres-12/src/backend/storage/lmgr/README#L362-L381) |
| Already holds the requested mode | `LockAcquireExtended()` increments the local acquisition count and returns immediately | Repeating the same lock acquisition does not enter the queue. Measured: the repeated request was granted within session A's 500 ms `lock_timeout`, below the 1 s `deadlock_timeout`, so it did not wait behind the drop; see [Measured results](#measured-results) and the paragraph after this table. [lock.c#already-held-mode](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L828-L840) |
| Holds a different mode | `ProcSleep()` scans the queue from the front and stops at the first waiter whose request conflicts with a mode this session holds. Because the drop's request conflicts with every mode, that waiter is the drop or one ahead of it. There `ProcSleep()` reports an early [deadlock](../../../glossary.md#deadlock) if this session must also wait for that waiter's held locks. Otherwise it grants the new mode at once if neither other sessions' held locks nor the requests ahead conflict, and failing that it queues the session there. | The session proceeds at once, waits ahead of the drop, or receives a deadlock error; it never queues behind the drop. Measured: a session holding `ACCESS SHARE` got `ROW EXCLUSIVE` within its 500 ms `lock_timeout`, below the 1 s `deadlock_timeout`, while the drop kept waiting; see [Measured results](#measured-results) and the paragraph after this table. [proc.c#existing-holder-queue-position](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1100-L1204) |

Session A's 500 ms bound is what lets the last two measurements tell an immediate grant from a queued request. Had either request of A's been queued behind the drop, A would have waited for the drop while the drop waited for A's `ACCESS SHARE` lock. A deadlock check can resolve that cycle by reordering the queue and granting A's request. Each waiting backend runs that check once per wait, `deadlock_timeout` after the wait begins; the default is 1 s. The drop was already waiting for `s.t` before the 1-second blocked probe began, and A asks only once the drop's statement has run for 2 s, measured from its `pg_stat_activity.query_start` [monitoring.sgml#query_start](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L675-L679), so the drop's own check has already run and only A's check could do it. A bound at or above 1 s therefore could not distinguish the two cases. The script leaves `deadlock_timeout` at its default and checks it before the probe. [proc.c#deadlock-timer](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1248-L1264) [proc.c#deadlock-check-after-timeout](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1289-L1297) [deadlock.c#DeadLockCheck-reorder](../../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L217-L285) [guc.c#deadlock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2062-L2072) [lmgr/README#deadlock-detector-reorders](../../../../raw/postgres-12/src/backend/storage/lmgr/README#L370-L377)

The table covers requests for the object the drop is waiting for. The drop's `AccessExclusiveLock` on an object it already holds conflicts with every other session's request, so those requests wait while the drop holds it. An object the walk has not reached yet carries no lock or request from the drop, so a new session's request for it is granted unless something else conflicts, and the drop may later have to wait for that session in turn. Measured: during the wait for `s.t`, a new session got `ACCESS SHARE` at once on `s.big`, a table of `s` that the walk had not reached; see [Measured results](#measured-results). [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L99-L103) [lock.c#join-wait-queue](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L976-L985) [dependency.c#scan-locks-each-dependent](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L861-L901)

The deletion set is not bounded by the schema name. `CASCADE` follows dependent objects into other schemas. Internal and extension ownership can also redirect traversal toward an owning object; `CASCADE` is not a filesystem wildcard limited to the files that were emptied. [dependency.c#dependency-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L829-L866) [dependency.c#extension-and-internal-dependencies](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L594-L734) [drop_schema.sgml#cross-schema-cascade](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L89-L95)

A regular table's TOAST table, when present, illustrates why physical cleanup is not limited to objects named in the user schema. It lives in `pg_toast` and has an internal dependency on its owning table, so dependency deletion includes it too. Temporary tables use their temporary TOAST namespace instead. [toasting.c#toast-namespace](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L228-L235) [toasting.c#toast-dependency](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L362-L376)

### Deleting objects does not scan their data

The generic deletion dispatcher selects `index_drop()` for an index, `RemoveAttributeById()` for a single column (`objectSubId != 0`), and `heap_drop_with_catalog()` for a whole non-index relation. For a sequence it also deletes the `pg_sequence` row. These routines remove metadata and, for whole relations and indexes, schedule storage deletion; they do not perform a row-by-row `DELETE`, scan the target [heap](../../../glossary.md#heap), or run an index integrity check. Related objects such as indexes and constraints are handled by dependency traversal before their table. [dependency.c#doDeletion](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1341-L1369) [heap.c#RemoveAttributeById](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1613-L1717) [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1844-L1998) [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266)

“Open the relation” here is not “read its first data page.” `relation_open()` obtains the lock and a relcache descriptor. Building that descriptor reads catalog metadata and initializes [access-method](../../../glossary.md#access-method) information. Thus a zero-byte user index can be dropped through the ordinary path without reading its [metapage](../../../glossary.md#metapage), while an unusable catalog index needed to build the descriptor can still make the operation fail. [relation.c#relation_open](../../../../raw/postgres-12/src/backend/access/common/relation.c#L47-L78) [relcache.c#RelationBuildDesc](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1055-L1084) [relcache.c#access-method-initialization](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1188-L1217) [index.c#index-drop-open](../../../../raw/postgres-12/src/backend/catalog/index.c#L2047-L2056)

### Cross-schema dependencies and locks

Locks come from two places. The dependency walk has already taken `AccessExclusiveLock` on every relation it found, under `RESTRICT` as well as `CASCADE`; see steps 3 and 4 above. [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) On the `CASCADE` path, the deletion routines then lock more tables, including tables that survive the drop. The cases below are examples, not a complete list:

- Dropping an index locks its owning table. [index.c#index-drop-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L2029-L2056)
- Dropping a partition locks its parent. If the parent has a default partition and the dropped partition is not that default partition, it also locks the default partition. [heap.c#partition-locks](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1861-L1886)
- **A dropped table references another table.** Dropping a table cascades to its foreign-key constraints, which carry an automatic dependency on it. Each such constraint owns "action" [foreign key triggers](../../../glossary.md#foreign-key-trigger) that live on the referenced table. `RemoveTriggerById()` takes `AccessExclusiveLock` on the table a trigger belongs to. Thus removing the constraint locks the referenced table, which can be in another schema; that removal alone does not drop the referenced table. [pg_constraint.c#auto-dependency-on-table](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L233-L258) [tablecmds.c#createForeignKeyActionTriggers](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L9940-L10002) [trigger.c#internal-constraint-dependency](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1019-L1030) [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1525-L1530) The referenced table is also deleted if another dependency includes it in the deletion set, for example its own normal dependency on the schema being dropped. [heap.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408) [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437)
- **Another table references a dropped table.** The foreign-key constraint belongs to the referencing table, but it has a normal dependency on the referenced columns, so dropping the referenced table cascades to it. `RemoveConstraintById()` takes `AccessExclusiveLock` on the constraint's own table, and `RemoveTriggerById()` locks that table again for the constraint's "check" triggers. The referencing table can be in another schema. It survives, but without its foreign key. [pg_constraint.c#normal-dependency-on-referenced-columns](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L274-L298) [pg_constraint.c#RemoveConstraintById-lock](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L572-L580) [trigger.c#internal-constraint-dependency](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1019-L1030) [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1525-L1530)

An outside table can also lose an object or column that uses something in the dropped schema. The first table below follows each listed dependency in isolation: its owning table survives unless another dependency puts the whole table in the deletion set. That table's owner need not be the role that drops the schema, because no core step on this path checks ownership of, or privileges on, the surviving table; see step 2 of [From the command to the deletion list](#from-the-command-to-the-deletion-list). [dependency.c#combined-deletion-set](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) [dropcmds.c#ownership-check](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L104-L109)

| Outside object | Dependency that reaches it | Removal and lock on the surviving table |
|---|---|---|
| Trigger using a schema function | `DEPENDENCY_NORMAL` from the trigger to its function. [trigger.c#function-dependency](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1006-L1017) | `RemoveTriggerById()` removes the trigger and takes `AccessExclusiveLock` on its table. [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1496-L1568) |
| `CHECK` or foreign-key constraint | A `CHECK` records normal dependencies on objects in its expression. A foreign key records normal dependencies on the referenced relation/columns, supporting unique index and equality operators. [pg_constraint.c#foreign-key-dependencies](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L274-L346) [pg_constraint.c#check-expression-dependencies](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L356-L365) | `RemoveConstraintById()` removes the constraint and takes `AccessExclusiveLock` on its owning table. Foreign-key trigger removal also accounts for the other-side locks described above. [pg_constraint.c#RemoveConstraintById](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L552-L631) [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1496-L1568) |
| Ordinary index, including an [expression index](../../../glossary.md#expression-index) or [partial index](../../../glossary.md#partial-index) | Normal dependencies on external objects in its expressions or predicate, its [operator classes](../../../glossary.md#operator-class), and its non-default [collations](../../../glossary.md#collation). [index.c#index-dependencies](../../../../raw/postgres-12/src/backend/catalog/index.c#L1120-L1163) | `index_drop()` removes the index and takes `AccessExclusiveLock` on its owning table. [index.c#index-drop-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L2029-L2056) |
| `EXCLUDE` constraint | No direct constraint dependency is recorded on its exclusion operators. The dependency runs through the supporting index's operator class; the index is itself `DEPENDENCY_INTERNAL` on the constraint. [pg_constraint.c#indirect-exclusion-dependency](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L348-L354) [index.c#operator-class-dependency](../../../../raw/postgres-12/src/backend/catalog/index.c#L1134-L1143) [index.c#index-constraint-dependency](../../../../raw/postgres-12/src/backend/catalog/index.c#L1888-L1896) | Reaching the index redirects deletion through its owning constraint, so both are removed. `index_drop()` and `RemoveConstraintById()` take `AccessExclusiveLock` on the table. [dependency.c#redirect-to-owner](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L682-L737) [index.c#index-drop-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L2047-L2050) [pg_constraint.c#constraint-table-lock](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L572-L580) |
| Non-`SELECT` [rewrite](../../../glossary.md#rewriter) rule | Normal dependencies on referenced objects, plus `DEPENDENCY_AUTO` on its owning table (`evtype != CMD_SELECT`). [rewriteDefine.c#InsertRule-dependencies](../../../../raw/postgres-12/src/backend/rewrite/rewriteDefine.c#L160-L184) | `RemoveRewriteRuleById()` removes the rule and takes `AccessExclusiveLock` on the table. [rewriteRemove.c#RemoveRewriteRuleById](../../../../raw/postgres-12/src/backend/rewrite/rewriteRemove.c#L37-L92) |
| Column default or [row-level security](../../../glossary.md#row-level-security) policy | Normal dependencies on external objects in the expression. [heap.c#default-expression-dependencies](../../../../raw/postgres-12/src/backend/catalog/heap.c#L2293-L2321) [policy.c#policy-expression-dependencies](../../../../raw/postgres-12/src/backend/commands/policy.c#L847-L853) | `RemoveAttrDefaultById()` or `RemovePolicyById()` removes that object and takes `AccessExclusiveLock` on the table. [heap.c#RemoveAttrDefaultById](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1773-L1842) [policy.c#RemovePolicyById](../../../../raw/postgres-12/src/backend/commands/policy.c#L365-L420) |
| Column whose type or non-default collation is in the schema | `DEPENDENCY_NORMAL` on the type or collation. The dependent address is a column subobject: `classId = RelationRelationId`, `objectSubId = attnum`. [heap.c#AddNewAttributeTuples](../../../../raw/postgres-12/src/backend/catalog/heap.c#L820-L838) [tablecmds.c#column-dependencies](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L6068-L6102) | `AcquireDeletionLock()` locks the whole outside table during the walk; `RemoveAttributeById()` marks the column dropped. [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) [heap.c#RemoveAttributeById](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1613-L1717) |
| Generated column whose expression uses a schema function | `DEPENDENCY_AUTO` from the column subobject to the function; the function has `DEPENDENCY_NORMAL` on the schema. [heap.c#generated-column-dependencies](../../../../raw/postgres-12/src/backend/catalog/heap.c#L2293-L2321) [pg_proc.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/pg_proc.c#L616-L620) | The walk locks the outside table and `RemoveAttributeById()` marks the generated column dropped, as for the preceding row. [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) [heap.c#RemoveAttributeById](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1613-L1717) |

The last two rows differ in dependency reporting. A type/collation-dependent column appears in `RESTRICT` detail and in the `CASCADE` notice unless an automatic path also reaches it. The generated column takes the automatic branch, reported as `drop auto-cascades to ...` at `DEBUG2`; its function's normal dependency on the schema still requires `CASCADE`. [dependency.c#reportDependentObjects](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1113-L1203) [pg_proc.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/pg_proc.c#L616-L620)

Some dependencies instead reach the outside relation itself. `AcquireDeletionLock()` locks that relation during the walk, and `CASCADE` removes it as part of the deletion set. [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) [dependency.c#relation-deletion](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1341-L1369)

| Outside relation | Dependency path | Result under `CASCADE` |
|---|---|---|
| Partition or [inheritance](../../../glossary.md#inheritance) child of a table in the schema | A partition has `DEPENDENCY_AUTO` on its parent; an inheritance child has `DEPENDENCY_NORMAL`. [tablecmds.c#child_dependency_type](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L296-L302) [tablecmds.c#StoreCatalogInheritance1](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L2776-L2798) | The child relation is dropped. [dependency.c#dependent-relations](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L902-L988) |
| Typed table `OF s.type` | `DEPENDENCY_NORMAL` from the table to the type. [heap.c#reloftypeid-dependency](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1410-L1416) | The typed table is dropped. [dependency.c#dependent-relations](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L902-L988) |
| View or materialized view referring to schema objects | Its `ON SELECT` (`_RETURN`) rule has normal dependencies on referenced objects and an internal dependency on the view or materialized view. [rewriteDefine.c#InsertRule-dependencies](../../../../raw/postgres-12/src/backend/rewrite/rewriteDefine.c#L160-L184) | Reaching the rule redirects deletion to its owning relation, so the whole view or materialized view is dropped. [dependency.c#redirect-to-owner](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L682-L737) |

The pinned build showed both foreign-key cases and the trigger case. Inside the dropping transaction, the [backend](../../../glossary.md#backend) held `AccessExclusiveLock` on the surviving tables `other.parent`, `other.child_ref` and `other.audit`. After commit, `other.child_ref` had lost its foreign key, `other.audit` its trigger, and `other.parent` its action triggers. See [Measured results](#measured-results).

Empty files do not remove any of these lock requirements. The current session's own open uses are checked by `CheckTableNotInUse()`. [heap.c#drop-locks-and-in-use-check](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1861-L1908)

Finally, `RemoveSchemaById()` deletes the `pg_namespace` row. Generic deletion also removes outgoing dependency records, shared dependencies, comments, security labels and initial privileges. This is why usable catalogs remain essential even if every target data file is empty. [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L214-L235) [dependency.c#deleteOneObject](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1269-L1326)

### Commit removes storage by filename

`RelationDropStorage()` stores the relation's physical identifier with `atCommit = true`; it does not immediately remove the file. At commit, `smgrDoPendingDeletes(true)` passes the accumulated relations to `smgrdounlinkall()`. The latter closes storage handles, discards their remaining [buffers](../../../glossary.md#buffer-manager) without writing them, sends storage [invalidations](../../../glossary.md#invalidation-message) and calls the file deletion routine for each fork. [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462) [xact.c#commit-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232) [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495)

For ordinary non-temporary relations outside [recovery](../../../glossary.md#crash-recovery), `mdunlinkfork()` truncates the first main-fork segment to zero and queues its later unlink. For every other fork it unlinks the first file at once. Then, for each fork whose first-file step succeeded, the main fork included, it unlinks the numbered segments `.1`, `.2`, … in order. The delayed main file reserves the physical identifier until a suitable [checkpoint](../../../glossary.md#checkpoint) completes; this avoids premature reuse. A request arriving during a checkpoint waits for a later checkpoint cycle. Temporary relations and recovery use the immediate-unlink branch instead. [md.c#deferred-unlink-reason](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L217-L261) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) [sync.c#checkpoint-cycle](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214)

**An existing zero-byte segment does not stop this cleanup.** Consider existing paths `12345`, `12345.1` and `12345.2`, all emptied by the shell command. The main-file truncation succeeds, unlinking `.1` succeeds regardless of its length, and the loop then attempts `.2`. The example follows filename operations; there is no `mdnblocks()` call that would stop at `.1` because it is short. The `cascade` leg of the measurement reproduced it with the three emptied segments of `s.big`. By contrast, the size-probing path used by scans returns at a short segment it measures. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L300-L359) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L715-L755)

**Missing filenames differ from empty files.** If `.1` was removed rather than emptied, its unlink returns `ENOENT` and this traversal does not reach `.2`; the `missing` leg of the measurement reproduced this. If the first segment operation fails, the additional-segment loop is not entered at all. Other filesystem failures produce warnings and can also leave files behind. Therefore SQL-level success does not prove that every damaged or orphaned pathname was removed. A zero-byte first main-fork file remaining before the eligible checkpoint, however, is expected cleanup state. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L284-L359) [smgr.c#unlink-warning-contract](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L481-L495)

### State changes and rollback

The lifecycle below concerns objects that existed before the dropping transaction. The source separately handles objects created and dropped within the same transaction by keeping both commit-time and abort-time cleanup entries. [storage.c#pending-delete-lifecycle](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [storage.c#create-and-drop-in-one-transaction](../../../../raw/postgres-12/src/backend/catalog/storage.c#L162-L169)

| Event | State before → after | Consequence |
|---|---|---|
| External truncation | User file had contents → pathname exists with zero length; no SQL catalog deletion was requested | The dependency list still names the objects. No pending-delete entry saves the original bytes. [heap.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408) [storage.c#PendingRelDelete](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) |
| Successful drop statement in an open transaction | Metadata present → transactional catalog deletions plus commit-only file cleanup entries | Finishing the statement is not yet physical storage cleanup. [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L218-L234) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) |
| Rollback or error aborts that transaction | Uncommitted catalog deletion → aborted; commit-only deletion entries are discarded; the walk's locks are released | The objects remain, but PostgreSQL does not undo the external file truncation or restore the lost contents. [xact.c#abort-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2708-L2720) [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462) |
| Commit | Catalog deletions committed → pending storage cleanup executes | Buffers are discarded, forks/segments are removed as described above, and ordinary first main files await later unlink. [xact.c#commit-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232) [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) |
| Eligible checkpoint completes | Queued first main file → unlink attempted | Physical filename cleanup catches up with the committed deletion. [sync.c#SyncPostCheckpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214) |

The transaction machinery also protects the ordering between [WAL](../../../glossary.md#wal) and non-temporary file deletion: pending relation deletions force a WAL flush before the files can disappear. Commit replay drops the recorded relation files. These mechanisms describe PostgreSQL's own deletion; they are not an undo mechanism for the preceding shell operation. [xact.c#flush-before-delete](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1359-L1377) [xact.c#xact_redo_commit](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L5817-L5838)

### Cases that change the result

| Condition / state | Default or ordinary path, and what changes it | Consequence |
|---|---|---|
| Schema still has ordinary objects; no `CASCADE` | Default `RESTRICT` checks normal dependencies, not contents, and checks them only after the walk has locked every dependent | Refusal even with empty data files. It can wait for those locks before it refuses. Measured: with all 29 files of the fixture schema at zero bytes, the refusal listed 11 dependents. [gram.y#default](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568) [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901) [dependency.c#RESTRICT-error](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1133-L1203) |
| Explicit `CASCADE`; only user files emptied | The command option allows normal dependent objects to be deleted; the storage-removal path does not scan their contents | Empty heap, index, sequence, auxiliary fork or segment files alone do not invalidate this path. No repair setting is required by it. Measured: the drop committed with every heap, index, sequence, materialized-view, TOAST, fork and segment file of the fixture schema emptied. [dependency.c#doDeletion](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1341-L1369) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) |
| Catalogs or their indexes were emptied too | Name resolution, dependency enumeration and metadata deletion need usable catalog information; default catalog scans use indexes. [genam.c#systable_beginscan](../../../../raw/postgres-12/src/backend/access/index/genam.c#L361-L366) | Uncached truncated-file reads can fail. Cached pages, an explicitly enabled zero-fill setting, or a forced heap scan can produce different outcomes. The exact branches are in [Catalog damage branches](#catalog-damage-branches). [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L589-L629) [genam.c#heap-scan-fallback](../../../../raw/postgres-12/src/backend/access/index/genam.c#L413-L425) [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) |
| Added event trigger or C hook | Core dispatch normally runs without an installed utility hook; an empty event-trigger run list returns. Installed code is invoked when present | Extra SQL or an exception can change the outcome before commit. The pinned tests explicitly exercise a trigger rejecting `DROP SCHEMA ... CASCADE`, both through a nested `DROP TABLE` and directly for an object the schema drop removes. [utility.c:76](../../../../raw/postgres-12/src/backend/tcop/utility.c#L76) [utility.c#ProcessUtility-hook](../../../../raw/postgres-12/src/backend/tcop/utility.c#L355-L362) [event_trigger.c#EventTriggerDDLCommandStart](../../../../raw/postgres-12/src/backend/commands/event_trigger.c#L809-L817) [dependency.c#object-drop-hook](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1240-L1242) [event_trigger.out#rejected-schema-drop](../../../../raw/postgres-12/src/test/regress/expected/event_trigger.out#L277-L291) [event_trigger.out#direct-rejection](../../../../raw/postgres-12/src/test/regress/expected/event_trigger.out#L293-L308) |
| Concurrent sessions or current-session object use | The dependency walk locks every dependent, under `RESTRICT` as well as `CASCADE`, including outside relations when one of their columns, or the whole relation, depends on an object in the schema. `CASCADE` deletion then also locks other tables, among them surviving tables outside the schema: partition parents and default partitions, the tables on both sides of a cross-schema foreign key, and tables whose constraints, indexes, triggers, non-`SELECT` rewrite rules, defaults or policies depend on schema objects. In-use checks apply regardless of file length | It can wait or fail before file cleanup. While it waits, new requests from sessions that hold no lock on the object queue behind it; objects the walk has not reached yet are not locked. [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901) [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) [heap.c#partition-locks](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1861-L1886) [index.c#index-drop-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L2047-L2050) [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1525-L1530) [pg_constraint.c#RemoveConstraintById-lock](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L572-L580) [rewriteRemove.c#RemoveRewriteRuleById-lock](../../../../raw/postgres-12/src/backend/rewrite/rewriteRemove.c#L73) [heap.c#RemoveAttrDefaultById-lock](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1808-L1812) [policy.c#RemovePolicyById-lock](../../../../raw/postgres-12/src/backend/commands/policy.c#L385-L390) [lock.c#join-wait-queue](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L976-L985) [lmgr/README#lock-acquisition-rules](../../../../raw/postgres-12/src/backend/storage/lmgr/README#L362-L381) [heap.c#CheckTableNotInUse](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1890-L1908) |
| The drop's locks, together with every other session's, need more shared lock-table entries than shared memory can supply | The walk holds the lock on every object it found until the transaction ends, under `RESTRICT` as well as `CASCADE`, and `CASCADE` deletion adds locks on outside tables. These are `AccessExclusiveLock` relation locks and object locks. Neither kind can use the per-backend fast-path slots, which only relation locks weaker than `ShareUpdateExclusiveLock` use, so they go in the shared [lock table](../../../glossary.md#heavyweight-lock). That table keeps one `LOCK` entry per locked object, shared by every session that holds or awaits a lock on it, and one `PROCLOCK` entry per object and session, shared by all the modes that session holds on it. A lock on an object the dropping session has not locked yet therefore needs a new `PROCLOCK` entry, plus a new `LOCK` entry when no session holds or awaits a lock on that object. PostgreSQL estimates the number of lock entries as `max_locks_per_transaction` × (`MaxBackends` + `max_prepared_transactions`), where `MaxBackends` counts `max_connections`, the autovacuum workers and launcher, `max_worker_processes` and `max_wal_senders`. That estimate is not a hard cap. The `LOCK` hash table preallocates half of it, and `ShmemInitHash()` treats the maximum as a sizing hint, not a limit. `LockShmemSize()` adds a 10% safety margin because the estimate is approximate, so the table can grow into shared memory that is not yet allocated. `max_locks_per_transaction` defaults to 64 and has `postmaster` context, so changing it needs a restart. All sessions' locks share the table | When `SetupLockInTable()` cannot allocate a `LOCK` or `PROCLOCK` entry, the request fails with `out of shared memory` (SQLSTATE `53200`) and the hint to increase `max_locks_per_transaction`. The transaction aborts, so no file is removed. Empty files do not reduce the number of locks. Source-derived, not measured; the exact number of locks at which this happens depends on how much free shared memory remains. [lock.c#SetupLockInTable](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L1119-L1197) [lock.h#PROCLOCKTAG](../../../../raw/postgres-12/src/include/storage/lock.h#L341-L346) [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901) [lock.c#fast-path-eligibility](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L199-L204) [lock.c#NLOCKENTS](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L56-L57) [lock.c#lock-hash-size](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L384-L389) [shmem.c#max_size-not-hard-limit](../../../../raw/postgres-12/src/backend/storage/ipc/shmem.c#L303-L310) [lock.c#LockShmemSize-margin](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L3441-L3460) [postinit.c#MaxBackends](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L532-L534) [guc.c#max_locks_per_transaction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2464-L2473) [lock.c#out-of-shared-memory](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L956-L968) [errcodes.txt:400](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L400) |
| Serializable transaction tracking is active, or the dropping transaction uses `SERIALIZABLE` instead of the default `READ COMMITTED` | Index deletion can transfer [predicate locks](../../../glossary.md#predicate-lock) even for a non-serializable drop. The table conflict check additionally requires a serializable dropping transaction. See [Serialization branches](#serialization-branches) for the vocabulary, eligibility tests and order. [predicate.c#transfer-gates](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L2918-L2939) [predicate.c#write-gate](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L560-L572) | The table conflict check or pre-commit check can raise `40001` before storage cleanup. Empty files do not bypass those checks. These branches are source-derived, not measured. [heap.c#serializable-conflict-check](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1902-L1908) [predicate.c#writer-failure](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4776-L4808) [predicate.c#pre-commit-check](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4828-L4908) |
| Files truncated while the server remains active | External changes do not execute PostgreSQL's buffer invalidation and discard path. A previously cached page can remain available; a [dirty buffer](../../../glossary.md#dirty-buffer) can later be written | Do not interpret the external command as a coordinated clearing of all server state. The commit-time drop explicitly discards remaining buffers instead. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [bufmgr.c#FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2740-L2778) [smgr.c#discard-buffers](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L459-L479) |
| Missing segment or filesystem failure | Normally consecutive filenames can be removed; a missing name or failed unlink interrupts enumeration | Warning or silent `ENOENT` handling can leave later files behind; `CASCADE` does not search the directory for orphans. Measured: with segment `.1` removed, segments `.2` and `.3` survived the commit and the next checkpoint, and the server logged no warning. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L284-L359) |

### Serialization branches

The default isolation level is `READ COMMITTED`. `default_transaction_isolation` and `transaction_isolation` both have `PGC_USERSET` (`user` [GUC context](../../../glossary.md#guc-context)), but their scopes differ. When `default_transaction_isolation` is set for the session, in the configuration file or at connection start, every later transaction copies it into its own level. `transaction_isolation` affects only the current transaction: `StartTransaction()` overwrites it when the next transaction starts, and `check_XactIsoLevel()` refuses a change after the transaction's first [snapshot](../../../glossary.md#snapshot) or inside a [subtransaction](../../../glossary.md#subtransaction). So a drop runs as `SERIALIZABLE` when `default_transaction_isolation` is `serializable`, or when its own transaction selects that level before its first query, for example with `BEGIN ISOLATION LEVEL SERIALIZABLE`. An explicit `SERIALIZABLE` transaction changes the table-drop conflict path below. Index deletion has a separate responsibility: it preserves other transactions' predicate locks by transferring them from the index to its heap. [guc.c#isolation-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4254-L4273) [xact.c:1953](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1953) [variable.c#check_XactIsoLevel](../../../../raw/postgres-12/src/backend/commands/variable.c#L525-L554) [gram.y#BEGIN-transaction-modes](../../../../raw/postgres-12/src/backend/parser/gram.y#L9835-L9841) [gram.y#transaction_mode_item](../../../../raw/postgres-12/src/backend/parser/gram.y#L9936-L9939) [utility.c#BEGIN-options](../../../../raw/postgres-12/src/backend/tcop/utility.c#L416-L429) [predicate.c#SerializationNeededForWrite](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L560-L572) [predicate.c#TransferPredicateLocksToHeapRelation](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L3091-L3100)

[Serializable snapshot isolation (SSI)](../../../glossary.md#serializable-snapshot-isolation) records reads with predicate locks. These records do not block other transactions and can survive the reader's commit until overlapping transactions finish. Acquiring the drop's `AccessExclusiveLock` therefore does not prove that this read tracking is gone. [predicate.c#SIREAD-lifetime](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L27-L59) [heap.c#lock-before-conflict-check](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1890-L1908)

A **[read/write conflict](../../../glossary.md#serializable-snapshot-isolation)**, written `reader → writer`, means the reader did not see the concurrent writer's change, so the reader must come first in a possible serial execution. A **[dangerous structure](../../../glossary.md#serializable-snapshot-isolation)** has two adjacent conflicts; the middle transaction is the **[pivot](../../../glossary.md#serializable-snapshot-isolation)**. [README-SSI#rw-conflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/README-SSI#L133-L174)

```text
Tin --rw--> Tpivot --rw--> Tout
```

This pattern can exist without a complete dependency cycle, so rejecting it can produce a false positive. An actual anomaly requires `Tout` to commit first; if `Tin` is read-only, `Tout` must also have committed before `Tin` took its snapshot. The implementation checks ordering conservatively, including transactions already prepared to commit. The map below therefore asks whether those failure tests fire, not just whether the two-edge pattern exists. [README-SSI#dangerous-structure](../../../../raw/postgres-12/src/backend/storage/lmgr/README-SSI#L160-L183) [predicate.c#ordering-checks](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4634-L4723) [predicate_internals.h#commit-order](../../../../raw/postgres-12/src/include/storage/predicate_internals.h#L62-L77)

The map uses these tests and states:

| Test | Exact state checked | Meaning for the drop |
|---|---|---|
| Serializable tracking exists | `TransactionIdIsValid(PredXact->SxactGlobalXmin)` | Both the index-transfer and table-conflict routines return immediately when it is false. [predicate.c#transfer-gate](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L2918-L2928) [predicate.c#table-conflict-gate](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4534-L4548) |
| Relation is eligible | `PredicateLockingNeededForRelation()`: `rd_id >= FirstBootstrapObjectId`, no local buffers, and not a materialized view | System relations below that boundary, temporary relations using local buffers and materialized views are excluded. [predicate.c#PredicateLockingNeededForRelation](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L493-L503) |
| This transaction has serializable state | `MySerializableXact != InvalidSerializableXact` | Required by the table-write check and pre-commit check; not required for index predicate-lock transfer. [predicate.c#SerializationNeededForWrite](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L560-L572) [predicate.c#transfer-gates](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L2918-L2939) [predicate.c#pre-commit-gate](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4828-L4837) |
| This transaction is doomed | Its `SERIALIZABLEXACT.flags` contains `SXACT_FLAG_DOOMED`, marking it for rollback | The pre-commit check rejects it with `40001`. [predicate_internals.h#transaction-flags](../../../../raw/postgres-12/src/include/storage/predicate_internals.h#L110-L117) [predicate.c#doomed-check](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4839-L4848) |
| A pivot is prepared | Its `SXACT_FLAG_PREPARED` is set after it passes `PreCommit_CheckForSerializationFailure()` | This state occurs during ordinary `COMMIT` too; it does not require SQL `PREPARE TRANSACTION` ([two-phase commit](../../../glossary.md#two-phase-commit)). If the pre-commit conflict walk finds a qualifying, still-uncommitted prepared pivot, it aborts the current transaction because it can no longer choose that pivot for rollback. [predicate.c#prepared-state](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L267-L278) [predicate.c#prepared-pivot-and-state-transition](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4851-L4907) [xact.c#ordinary-commit-check](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2121-L2129) |

```text
index_drop(), before scheduling storage deletion [S1]
  tracking exists AND index is eligible?
    no  -> skip predicate-lock transfer
    yes -> transfer its predicate locks to the heap
           (the dropping transaction need not be SERIALIZABLE)

heap_drop_with_catalog(), before scheduling storage deletion [S2]
  tracking exists AND relation is eligible AND this transaction is serializable?
    no  -> skip the table conflict check
    yes -> examine other transactions' predicate locks on this relation
             for each read/write conflict not already recorded:
               SSI structure and ordering/read-only checks flag failure?
                 yes -> ERROR 40001 in the dropping transaction
                 no  -> record the conflict and continue

CommitTransaction(), before commit/storage cleanup [S3]
  this transaction has serializable state?
    no  -> skip the pre-commit serialization check
    yes -> already doomed, or a failure involving a prepared pivot?
             yes -> ERROR 40001
             no  -> pass this serialization check; mark SSI state prepared
```

- **[S1]** [index.c#transfer-before-storage](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2205), [predicate.c#DropAllPredicateLocksFromTable](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L2893-L3100).
- **[S2]** [heap.c#serializable-conflict-check](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1902-L1908), [predicate.c#CheckTableForSerializableConflictIn](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4540-L4610), [predicate.c#FlagRWConflict](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4617-L4632), [predicate.c#ordering-and-failure-tests](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4652-L4808), [predicate.c#writer-failure](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4776-L4808).
- **[S3]** [xact.c#pre-commit-check](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2128-L2130), [predicate.c#PreCommit_CheckForSerializationFailure](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4828-L4908), [xact.c#later-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2230-L2237). SQLSTATE `40001` is [errcodes.txt#serialization_failure](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L323).

1. **Index deletion preserves the read tracking.** `TransferPredicateLocksToHeapRelation()` calls `DropAllPredicateLocksFromTable(relation, true)`. Once the global and relation gates pass, it transfers existing predicate locks to the heap, including when the dropping transaction is not serializable. [predicate.c#DropAllPredicateLocksFromTable](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L2893-L3100)
2. **Table deletion can detect a conflict immediately.** `CheckTableForSerializableConflictIn()` passes the other transaction as `reader` and the dropping transaction as `writer` to `FlagRWConflict()`. `OnConflict_CheckForSerializationFailure()` tests the conflict pattern together with its commit-order and read-only conditions. If those tests set `failure`, it takes the `MySerializableXact == writer` error branch. Its later `SXACT_FLAG_DOOMED` assignment is for a different writer, not this call. Otherwise `FlagRWConflict()` records the edge and continues. [predicate.c#table-conflict-loop](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4550-L4600) [predicate.c#FlagRWConflict](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4617-L4632) [predicate.c#ordering-and-failure-tests](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4652-L4808) [predicate.c#writer-failure](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4776-L4808)
3. **Passing that check does not finish serialization checking.** Another backend can subsequently mark the dropping transaction doomed. `PreCommit_CheckForSerializationFailure()` errors if it is doomed, or if its conflict walk finds a qualifying dangerous structure with a prepared pivot that cannot be aborted. This happens before commit and before `smgrDoPendingDeletes(true)`. [predicate.c#PreCommit_CheckForSerializationFailure](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4828-L4908) [xact.c#pre-commit-check](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2128-L2130) [xact.c#later-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2230-L2237)

These paths use transaction and predicate-lock state, so empty relation files do not remove their gates or failure branches. The measurement below uses the default `READ COMMITTED` and does not reproduce a serialization failure. [predicate.c#CheckTableForSerializableConflictIn](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4534-L4610)

### Catalog damage branches

**Catalog damage has no single guaranteed outcome.** Outside recovery, the default `pg_depend` path uses catalog indexes, with `ignore_system_indexes = off` and `zero_damaged_pages = off`. A requested [block](../../../glossary.md#block) absent from shared buffers must be read from storage. The branch table follows that read from buffer lookup through segment opening, file reading and [tuple](../../../glossary.md#tuple) or index-page interpretation. [dependency.c#catalog-index-scans](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L559-L560) [dependency.c#dependent-index-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L861-L862) [genam.c#systable_beginscan](../../../../raw/postgres-12/src/backend/access/index/genam.c#L361-L366) [guc.c#ignore_system_indexes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1784-L1793) [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1162) [bufmgr.c#buffer-hit-or-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [bufmgr.c#read-and-verify](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L894-L925)

| Condition / state | Decision and output | Consequence for the drop |
|---|---|---|
| The requested catalog page is still valid in shared buffers | Buffer lookup returns that page without reading the externally truncated file | Old catalog contents can still supply the lookup or dependency rows. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) |
| A buffer miss targets a segment this backend has not opened, and a segment before it is short | Outside recovery, `_mdfd_getseg()` measures the segments from the last one this backend already has open, or from the first if none is open, up to the target, and rejects a short one before `mdread()` can perform its file read. It does not measure an already-open target segment, or an already-open segment before the last open one; a short segment of that kind is trusted as full | `previous segment is only N blocks` can stop the command. Enabling `zero_damaged_pages` does not suppress this earlier error. With earlier segments already open, the read can instead go ahead to the target segment's own file read, which the next rows describe. [md.c#segment-already-open](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1112-L1141) [md.c#segment-size-check](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1143-L1204) |
| The segment opens, but the file read returns less than one full block; settings remain at their defaults and the server is outside recovery | `mdread()` raises `could not read block ... read only ... bytes` | The read fails rather than silently omitting a catalog row. A negative file-read result also raises an error independently of the short-read setting. [md.c#read-errors-and-short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L589-L629) |
| `zero_damaged_pages` is `on` instead of its default `off`, and an indexed catalog-heap fetch reaches the short-read branch with a usable index. The setting has `PGC_SUSET` (`superuser`) GUC context: a superuser can change it at session or transaction scope with `SET` or at connection start, or set it in the server configuration file and reload [guc.h#PGC_SUSET](../../../../raw/postgres-12/src/include/utils/guc.h#L63-L64) | `mdread()` supplies zeros. Page verification accepts an [all-zero page](../../../glossary.md#all-zero-page); the heap tuple search finds no item on the zero-filled page | This fetch can silently omit a dependency row even though the catalog scan still uses its index. It does not establish that the rest of the drop succeeds. [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1162) [md.c#short-read-zero-fill](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L614-L629) [bufpage.c#all-zero-acceptance](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L130-L142) [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L129-L156) [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L348-L360) [heapam.c#heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1550-L1557) |
| The same zero-fill setting supplies a zero page to a catalog [B-tree](../../../glossary.md#b-tree) reader instead of a heap reader | `_bt_checkpage()` rejects the zero page even though `mdread()` returned without a short-read error | `index ... contains unexpected zero page` can still stop the command. Zero-fill does not make a damaged B-tree usable. [md.c#short-read-zero-fill](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L614-L629) [nbtpage.c#read-and-check](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L763-L769) [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719) |
| Connection startup explicitly sets `ignore_system_indexes = on` instead of its default `off`. The setting has `PGC_BACKEND` (`backend`) context: it applies to new sessions only (set at connection start, e.g. through `PGOPTIONS`, or in `postgresql.conf` followed by a reload that only later sessions pick up; `SET` fails once a session has started) [guc.h#PGC_BACKEND](../../../../raw/postgres-12/src/include/utils/guc.h#L54-L61) | `systable_beginscan()` chooses a heap scan. The heap scan takes its block bound from relation size; `mdnblocks()` starts at the last open segment, trusts earlier open segments as full, and returns at the first short segment it measures. `CatalogTupleDelete()` deletes only from the catalog heap, whereas `CatalogTupleUpdate()` (used when `CASCADE` updates catalog rows of surviving objects, such as `RemoveAttributeById()`, `RemoveAttrDefaultById()` or `RemoveConstraintById()`) still opens and inserts into catalog indexes via `CatalogIndexInsert()` | With fresh segment-open state, truncation can shorten the scan and omit dependency rows without requesting the erased blocks. Previously opened segments can give a larger bound instead, and any non-[HOT](../../../glossary.md#hot) catalog tuple update still touches the catalog's B-tree indexes. [guc.c#ignore_system_indexes](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1784-L1793) [genam.c#heap-scan-fallback](../../../../raw/postgres-12/src/backend/access/index/genam.c#L413-L425) [heapam.c#scan-size](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L216-L231) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L706-L755) [indexing.c#CatalogTupleUpdate](../../../../raw/postgres-12/src/backend/catalog/indexing.c#L224-L234) [indexing.c#CatalogIndexInsert](../../../../raw/postgres-12/src/backend/catalog/indexing.c#L74-L168) [indexing.c#CatalogTupleDelete](../../../../raw/postgres-12/src/backend/catalog/indexing.c#L254-L272) |
| A required metadata lookup finds no row | Name resolution or a deletion routine reports the missing metadata | The command can fail with a missing-schema or catalog-lookup error even if no storage read raised an error. [namespace.c#get_namespace_oid](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3030-L3047) [heap.c#catalog-lookup-failure](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1871-L1873) |

The silent indexed-fetch branch is a source-derived chain: **usable index supplies a heap location → uncached heap read reaches a short file → explicit zero-fill setting substitutes an all-zero page → heap lookup finds no tuple → dependency enumeration can miss that row**. It is not a measured damaged-catalog result. `systable_getnext()` uses `index_getnext_slot()`, whose heap-fetch loop advances to another index entry when a heap fetch returns no tuple. [genam.c#systable_getnext](../../../../raw/postgres-12/src/backend/access/index/genam.c#L444-L456) [indexam.c#index_getnext_slot](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L625-L636) Normally the short read errors. On this path the changed setting suppresses only that short-read error; the segment-opening and B-tree checks remain separate. Elsewhere the same setting also turns an invalid page header into a warning and a zeroed page. A truncated file never reaches that header check with damaged contents, because `mdread()` has already supplied an all-zero page, which verification accepts. [bufmgr.c#invalid-header-zeroing](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L918) [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L129-L156) [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L589-L629) [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L348-L360) [heapam.c#empty-page-search](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1550-L1557) [heapam.c#no-tuple-result](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1652-L1655) [dependency.c#dependency-enumeration](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L861-L873)

### Measured results

The [Measurement Script](#measurement-script) built the pin as 12.2 with one non-default compile-time knob, 1 MB segments, and ran an isolated cluster; see [Last run](#last-run). Its fixture schema `s` held:

- a table `s.t` with a serial key, a primary-key index and TOAST data;
- a 3.5 MB table `s.big` with numbered segments `.1` to `.3`, an index, a [free space map](../../../glossary.md#free-space-map) and a [visibility map](../../../glossary.md#visibility-map);
- a sequence, a materialized view, a view and two functions: `s.f()` and `s.trg_fn()`, the trigger function of the trigger `audit_trg` on `other.audit`;
- a table that references the table `other.parent` and a table that `other.child_ref` references, so foreign keys cross the boundary to the schema `other` in both directions.

With the server stopped, `cat /dev/null >` emptied all 29 files of 14 relations with storage: the 10 in `s`, plus the TOAST tables and TOAST indexes of its tables. A second schema `s2` held another 3.5 MB table. All 8 of its files were emptied the same way, and its segment `.1` was then removed, which left 7 files at 0 bytes and a missing `.1`. The numbers below are from the run recorded under [Last run](#last-run).

| Leg | What ran | Measured result | Claim it backs |
|---|---|---|---|
| `restrict` | `DROP SCHEMA s` | `2BP01`, "cannot drop schema s because other objects depend on it", listing 11 dependents, among them `constraint child_ref_pid_fkey on table other.child_ref` and `trigger audit_trg on table other.audit`. All 29 files unchanged. | `RESTRICT` follows dependency records, not file lengths |
| `locks` | Session A holds `ACCESS SHARE` on `s.t`, then a plain `DROP SCHEMA s` starts | The drop held `AccessExclusiveLock` on the schema object and on `s.t_id_seq`, and waited for `AccessExclusiveLock` on `s.t`. Before the drop arrived, a baseline session's `ACCESS SHARE` request was granted at once; during the wait, a second new session's request timed out with `55P03` under a 1-second `lock_timeout`. During the same wait, a third new session got `ACCESS SHARE` at once on `s.big`, on which the drop held no lock yet. Once the drop's statement had run for 2 s, session A re-acquired `ACCESS SHARE` and got `ROW EXCLUSIVE` within its 500 ms `lock_timeout`, below the server's 1 s `deadlock_timeout`. After A committed, the drop refused with `2BP01`. | The walk locks dependents one at a time, before the `RESTRICT` check, and leaves unreached ones unlocked; each row of the lock-queue table, for the third row its immediate-grant outcome |
| `owner` | `DROP SCHEMA s_own` under `SET ROLE`, rolled back | A role that does not own `s_own` but is an inheriting member of its owner role dropped it. A non-member got `42501`, "must be owner of schema s_own". | Ownership through `has_privs_of_role()` |
| `rollback` | `BEGIN; DROP SCHEMA s CASCADE;` a `pg_locks` query, `ROLLBACK` | `NOTICE` "drop cascades to 11 other objects". Before the rollback, the backend held granted `AccessExclusiveLock` on the surviving tables `other.parent`, `other.child_ref` and `other.audit`. After it, `s` still had 11 relations and all 29 files were present at 0 bytes. | Deletion-time locks outside the schema; rollback discards the commit-only deletes and does not restore truncated contents |
| `cascade` | `DROP SCHEMA s CASCADE`, then `CHECKPOINT` | After commit, only the 14 first main-fork files remained, each at 0 bytes. Every numbered segment, `_fsm` and `_vm` file was gone, including the three emptied segments of `s.big`. `other.child_ref` kept its 10 rows but no foreign key, `other.audit` had no user trigger, and `other.parent` had no action triggers. After one `CHECKPOINT`, no file of `s` remained. | No data scan; zero-length segments do not stop the unlink loop; deferred first-file unlink; surviving tables lose dependent objects |
| `missing` | `DROP SCHEMA s2 CASCADE` with `s2.big`'s segment `.1` removed, then `CHECKPOINT` | Segments `.2` and `.3` survived the commit and the checkpoint as zero-length orphans, and the server logged no warning | A missing filename ends the segment loop silently |

The server restarted after the files were emptied, so no page of these relations was cached. An uncached read of an existing block in an emptied first main-fork segment would have raised a short-read error; a read targeting a later segment could instead have failed in `_mdfd_getseg()` before reaching `mdread()`, because the preceding segment was short. All three `CASCADE` drops, the rolled-back one and the two committed ones, completed without either error. [md.c#read-errors-and-short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L589-L629) [md.c#segment-gap](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1204)

Three non-default conditions shape these numbers; [Non-default settings in the measurement](#non-default-settings-in-the-measurement) gives the default, cause and mechanism of each:

- The 1 MB segment size gives small tables numbered segments.
- [`autovacuum`](../../../glossary.md#autovacuum)` = off` and `checkpoint_timeout = 1h` keep background work from locking fixture tables or removing deferred files before the script looks.
- The probes' short `lock_timeout` values turn a lock wait into the `55P03` error the script can observe, and session A's 500 ms value stays below `deadlock_timeout` so that its two grants exclude a queued request. With the default `lock_timeout = 0`, the blocked session would instead wait until the drop's transaction ended. [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397) [proc.c#lock-timeout-timer](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1242-L1264) [postgres.c#lock-timeout-error](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3082-L3087) [errcodes.txt:418](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L418)

The measurement did not cover damaged catalogs, the `SERIALIZABLE` branch, already-open segment state, dependents owned by other roles, lock-table exhaustion, or default 1 GB segments; those rows remain source-derived. No measured result disagreed with the source reading.

### Build boundaries and existing tests

This is core backend behavior. The catalog build lists `pg_namespace.h` and `pg_depend.h`, generates their `_d.h` headers with the other catalogs, and symlinks the generated headers into the build tree's include directory; installation is handled elsewhere. Their declarations supply the catalog IDs and row layouts used above. No [contrib](../../../glossary.md#contrib) module or special build option is required for ordinary schema deletion. [catalog/Makefile#catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L51) [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L103) [pg_namespace.h#catalog-declaration](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L21-L44) [pg_depend.h#catalog-declaration](../../../../raw/postgres-12/src/include/catalog/pg_depend.h#L33-L66)

Relation opening does initialize access-method information, so the ordinary built-in-object conclusion does not certify arbitrary third-party access methods or their external storage. Hooks and extension dependencies are additional boundaries, described above. [relcache.c#access-method-initialization](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1188-L1217) [relcache.c#index-AM-initialization](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1403-L1428) [dependency.c#extension-dependency](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L594-L621)

The checked [regression tests](../../../glossary.md#regression-test) cover schema creation and rename followed by cascade deletion, missing-schema handling, and event-trigger rejection and recursive DDL. In the namespace test, the expected `NOTICE` that lists the two cascaded objects is what checks the deletion set. Its follow-up `pg_class` count cannot fail after the drop: the subquery that looks up the dropped schema's OID returns NULL, so `relnamespace = NULL` matches no row. These cited test cases do not inject zero-length relation files. This page's own [Measurement Script](#measurement-script) does, on a server built from the pin. [namespace.sql#schema-test](../../../../raw/postgres-12/src/test/regress/sql/namespace.sql#L5-L44) [namespace.out#cascade-notice-and-count](../../../../raw/postgres-12/src/test/regress/expected/namespace.out#L60-L70) [drop_if_exists.sql#schema-test](../../../../raw/postgres-12/src/test/regress/sql/drop_if_exists.sql#L51-L59) [event_trigger.sql#drop-rejection-test](../../../../raw/postgres-12/src/test/regress/sql/event_trigger.sql#L207-L261)

### Causal summary

**User files emptied while catalogs remain usable → schema dependencies remain → the dependency walk locks every dependent, waiting for any held elsewhere → default `RESTRICT` then refuses when ordinary objects still depend on the schema; explicit `CASCADE` permits dependency-ordered metadata deletion, which also removes outside objects that depend on objects in the schema, such as foreign keys of surviving tables → commit discards buffers and removes files without scanning their contents → eligible checkpoint finishes deferred first-main-file unlink.** Catalog damage, missing filenames, lock and permission failures, serialization failures and added code have the separate outcomes above. Rolling back the SQL drop does not reverse the external truncation. The pinned build reproduced each step of this chain for emptied user files; see [Measured results](#measured-results). [pg_constraint.c#normal-dependency-on-referenced-columns](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L274-L298) [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901) [storage.c#pending-deletion](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) [sync.c#SyncPostCheckpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214)

## Measurement Script

### Usage

| Item | Value |
|---|---|
| Purpose | Builds the pin as 12.2 with 1 MB segments and empties every file of a disposable schema while the server is stopped. It then runs `DROP SCHEMA` six ways: with `RESTRICT`; with `RESTRICT` under lock contention; as a non-owner member of the owning role; as a `CASCADE` that rolls back; as a committed `CASCADE` followed by `CHECKPOINT`; and with one segment removed instead of emptied. Backs every number under [Measured results](#measured-results) and the measured notes in the lead, in the lock-queue table and the paragraph after it, in [Cross-schema dependencies and locks](#cross-schema-dependencies-and-locks), in [Commit removes storage by filename](#commit-removes-storage-by-filename) and in [Cases that change the result](#cases-that-change-the-result). |
| Invocation | From the repository root, extract the Bash fence below to `.wiki-runtime/tmp/ds12-run.sh`, then run `bash .wiki-runtime/tmp/ds12-run.sh [stage ...]`. The script checks the pinned checkout, clears inherited `PG*` settings (including `PGSERVICE` and `PGHOSTADDR`), sets its own socket and port, and verifies the connected data directory before running SQL against the fixture. It keeps every build and data file under `.wiki-runtime/tmp/ds12/`. |
| Stages | Default order: `build cluster fixture empty restrict locks owner rollback cascade missing report`. `stop` and `clean` tear down and are not in the default order. |
| Environment | `PORT12` (default `55444`) is the isolated port; `JOBS` (default `8`) sets parallel build jobs. The checkout, sandbox paths and `RELSEG_SIZE` are fixed in the script. |
| Prerequisites | A C toolchain, GNU make 3.80 or newer available as `make`, Perl 5.8.3 or newer, bison and flex; readline and zlib development headers, because `build` uses the default configure options; the pinned checkout at `raw/postgres-12/`; about 150 MB of free disk for the build, installation and cluster. Locale `C`, encoding `UTF8`. PostgreSQL's build invokes Perl for catalog generation; the measurement driver uses Bash/SQL, `git`, the build tools, built PostgreSQL programs and POSIX utilities. [installation.sgml#GNU-make](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L80-L83) [installation.sgml#Perl-for-Git-builds](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L309-L318) [catalog/Makefile#genbki](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L90) |
| Output | Files under `.wiki-runtime/tmp/ds12/out/`. Read `platform.txt` first, then `results.txt`, `restrict.txt`, `locks_d_held.txt`, `locks_c1.txt`, `locks_c2.txt`, `rollback.txt`, `cascade_survivors.txt`, and the `files_s_*.txt` and `files_s2_*.txt` listings. |
| Runtime | On the review host (`Linux 6.18.54.1-microsoft-standard-WSL2 x86_64`, `JOBS=8`): about 1 minute from an empty sandbox (54 seconds on the last run; recent runs took 52 to 70 seconds), most of it the build; about 3 seconds to re-run every stage while the build is present, 2 of them the lock stage's waits. |
| Cleanup | `bash .wiki-runtime/tmp/ds12-run.sh stop clean` stops the owned cluster, verifies that no PID file, process started from the sandbox's binaries, naming the data directory, or carrying the cluster's `postgres: ds12:` process title, or socket remains and that `127.0.0.1:PORT12` can be bound, then deletes `.wiki-runtime/tmp/ds12/`, including its build. If shutdown fails, the `.started` marker remains for a later stop attempt. Remove the extracted `ds12-run.sh` afterwards. |

To extract the published script without editing it, run this from the repository root:

```sh
sed -n '/^```bash$/,/^```$/p' wiki/v12/questions/storage-and-vacuum/drop-schema-zeroed-relation-files.md | sed '1d;$d' > .wiki-runtime/tmp/ds12-run.sh
bash -n .wiki-runtime/tmp/ds12-run.sh
bash .wiki-runtime/tmp/ds12-run.sh
```

### Last run

On 2026-10-28, during the twentieth review, the published script was extracted with the command above and run from an empty sandbox on `Linux 6.18.54.1-microsoft-standard-WSL2 x86_64` against PostgreSQL 12.2 at pin `45b88269a353ad93744772791feb6d01bc7e1e42` (53.56 seconds), then run a second time with the default stages and the build present (3.10 seconds). All 16 assertions passed on both runs. The unreached-table probe chose `s.big` both times, and the restrict leg listed 11 dependents. The `empty` stage emptied the 29 files of `s` and the 8 files of `s2`, then removed `s2.big`'s segment `.1`, which left 7 files of `s2` at 0 bytes. Session A has a 500 ms `lock_timeout`, the stage checks that `deadlock_timeout` is 1 s, and A asks only once the drop statement has run for 2 s; the drop was already waiting before the 1-second blocked probe, so its own deadlock check has run and session A's assertion excludes a request queued behind the drop. All 16 results match the nineteenth review's runs. The exclusion-constraint, serialization, cross-role and lock-table-exhaustion explanations remain source-derived, not measured. The server reported:

- [`block_size`](../../../glossary.md#blcksz) 8192 and `segment_size` 128 blocks (1 MB, from the patched `RELSEG_SIZE 128`), with [maximum data alignment](../../../glossary.md#alignment) 8;
- [`data_checksums`](../../../glossary.md#data-checksums) off, [`shared_buffers`](../../../glossary.md#shared_buffers) 16384 blocks (128 MB, the value `initdb` wrote) and `cluster_name` `ds12`;
- `autovacuum` off, `checkpoint_timeout` 3600 s, `default_transaction_isolation` read committed and `deadlock_timeout` 1000 ms, its default.

`stop clean` then confirmed that no server process, PID file or socket remained and that the port was free, and deleted the sandbox.

### Stages

| Stage | What it does | Needs |
|---|---|---|
| `build` | Configures 12.2 out of tree under `$SANDBOX/build` with default options, patches the generated `pg_config.h` to `RELSEG_SIZE 128`, builds and installs PostgreSQL, and compiles the C `tcp_port_free` bind helper with the configured compiler. Skipped when a matching build exists. | nothing |
| `cluster` | Checks that `127.0.0.1:PORT12` can be bound; runs `initdb -A trust --locale=C --encoding=UTF8` (checksums off, the default), writes socket-only settings plus `cluster_name = 'ds12'`, `autovacuum = off` and `checkpoint_timeout = '1h'`, starts on `PORT12`, and records `platform.txt` after checking the connected data directory | `build` |
| `fixture` | Checks that the connection reaches the owned cluster, then recreates database `ds` and the roles `ds12_owner`, `ds12_member` (a member of `ds12_owner`) and `ds12_other`. Builds schemas `s`, `s2`, `s_own` and `other`, vacuums, saves the inventory of `s` and `s2` and every file with its length, and asserts that the segments, forks and TOAST data exist. | `cluster` |
| `empty` | Stops the server, empties every file of `s` and `s2` with `cat /dev/null >`, removes `s2.big`'s segment `.1`, restarts, and asserts that every remaining file is 0 bytes. Refuses to run twice; re-run `fixture` first. | `fixture` |
| `restrict` | Runs `DROP SCHEMA s`; expects `2BP01` and unchanged files | `empty` |
| `locks` | Checks that `deadlock_timeout` is at its default 1 s. Session A, fed through a FIFO, sets a 500 ms `lock_timeout` and holds `ACCESS SHARE` on `s.t`, and a baseline session gets the same lock. Session D then runs `DROP SCHEMA s`, and the stage records D's locks (asserting the schema lock, `s.t_id_seq` and the waiting `s.t` request). A new session's request under a 1 s `lock_timeout` must fail with `55P03`. Another new session must get `ACCESS SHARE` within a 1 s `lock_timeout` on a table of `s` that the waiting drop holds no lock on. Once the drop's statement has run for 2 s, twice `deadlock_timeout`, A asks again for `ACCESS SHARE` and for `ROW EXCLUSIVE` under its 500 ms bound, and commits. D must fail with `2BP01`. | `empty` |
| `owner` | As `ds12_member`, drops `s_own` in a rolled-back transaction; as `ds12_other`, expects `42501` | `fixture` |
| `rollback` | Runs `DROP SCHEMA s CASCADE` and a `pg_locks` query inside a transaction, then `ROLLBACK`. Expects the cross-schema notices and locks, an unchanged relation count and unchanged files. | `empty` |
| `cascade` | Runs `DROP SCHEMA s CASCADE`, lists the files after commit and after one `CHECKPOINT`, and checks the surviving tables in `other`. Consumes `s`. | `empty` |
| `missing` | Runs `DROP SCHEMA s2 CASCADE`, lists the files after commit and after one `CHECKPOINT`, and checks the server log. Consumes `s2`. | `empty` |
| `report` | Prints `results.txt` | any |
| `stop` | Stops the owned cluster and verifies that no `postmaster.pid`, process started from `$BIN`, naming `-D $DATA` or carrying `postgres: ds12:`, or socket remains and that `127.0.0.1:PORT12` can be bound. It matches command lines that begin with `$BIN/`, pass `-D $DATA` or start with `postgres: ds12:`, so a calling shell whose command line merely mentions the script path does not count while [postmaster](../../../glossary.md#postmaster) child processes whose `argv[0]` was rewritten by `init_ps_display()` are still caught. | none |
| `clean` | Runs `stop`, then deletes `$SANDBOX`, refusing a sandbox it does not own | none |

Every fixture object is disposable. `cascade` and `missing` consume the schemas they drop; re-run `fixture` and `empty` before running them again. `restrict`, `locks`, `owner` and `rollback` leave the fixture as they found it.

### Non-default settings in the measurement

| Setting or state | Default at the pin | Value in the script, and why |
|---|---|---|
| `RELSEG_SIZE` | 131,072 blocks (1 GB at 8 kB) [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L295) [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864) | 128 blocks (1 MB), patched only in the out-of-tree generated header, because `--with-segsize` takes whole gigabytes. It gives the 3.5 MB fixture tables segments `.1` to `.3`. The segment assertions of `fixture`, `cascade` and `missing` depend on it. `mdunlinkfork()` names segments by number and never compares them with `RELSEG_SIZE`, so the default runs the same loop; a table just needs more than 1 GB before it has a `.1`. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L326-L359) |
| `autovacuum` | `on` [guc.c#autovacuum](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1425-L1433) | `off` (`PGC_SIGHUP`, in `postgresql.conf` before the first start), so that no worker locks a fixture table during `locks` or touches fixture files between stages. With the default, a worker could add a lock holder or waiter on `s.t` and change the lock choreography; the drop path itself does not change. |
| `checkpoint_timeout` | 300 s [guc.c#checkpoint_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2566-L2575) | 3,600 s (`PGC_SIGHUP`), so that no timed checkpoint runs between a committed drop and the file listing after it. Such a checkpoint would remove the deferred first main-fork files before the script sees them. The script then issues `CHECKPOINT` itself. With the default, the same files go at the next timed or requested checkpoint. |
| `lock_timeout` | 0, disabled [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2397) | 60 s for every session through `PGOPTIONS`; 1 s with `SET LOCAL` for the baseline, blocked and unreached-table probes; 500 ms with `SET LOCAL` for session A. All are `PGC_USERSET`, so they apply to that session or transaction only. The 1 s value is why the blocked probe ends with `55P03`; with 0 it would wait until the drop's transaction ended. Session A's bound is below `deadlock_timeout`, which the script leaves at its default of 1 s (`PGC_SUSET`) and `locks` checks [guc.c#deadlock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2062-L2072). The drop was already waiting before the 1 s blocked probe began, and the stage lets A ask only once the drop's statement has run for 2 s, so the drop's own single deadlock check has run. A request of A's queued behind the drop could then be granted before the drop ended only by A's own deadlock check, which runs `deadlock_timeout` after A begins waiting and can reorder the queue [proc.c#deadlock-check-after-timeout](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1289-L1297) [deadlock.c#DeadLockCheck-reorder](../../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L217-L285). So A's grants within 500 ms show that neither request queued behind the drop; a bound of 1 s or more could not show that. The 1 s bound on the unreached-table probe shows that its request did not wait for the drop. |
| `statement_timeout` | 0, disabled [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386) | 5 min through `PGOPTIONS` (`PGC_USERSET`), to bound runaway statements. No result depends on it. |
| Roles | — | `ds12_member` is created `IN ROLE ds12_owner` with the default `INHERIT`; the `owner` leg depends on that membership. |
| `role`, set by `SET LOCAL ROLE` | `none`, meaning that no `SET ROLE` is in effect [guc.c#role](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3824-L3834) | `ds12_member`, then `ds12_other`, each for one transaction of the `owner` leg (`PGC_USERSET`; `LOCAL` limits it to that transaction, which rolls back or fails). `check_role()` allows the switch only when the session user is a member of the target role, and `is_member_of_role()` counts a superuser as a member of every role [variable.c#check_role-membership](../../../../raw/postgres-12/src/backend/commands/variable.c#L854-L866) [acl.c#is_member_of_role](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4931-L4945). The leg's successful drop and its `42501` depend on the switched role, because `RemoveObjects()` checks ownership for the current role [dropcmds.c#ownership-check](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L104-L109). |
| Fixture data | — | `s.t`'s values are 3,200 bytes of md5 hex digits; `fixture` asserts that its TOAST table is non-empty before anything is emptied. |

`listen_addresses = ''`, `unix_socket_directories`, port 55444 and `cluster_name = 'ds12'` isolate and tag the cluster; all four are `PGC_POSTMASTER` settings written before the first start (`cluster_name` defaults to `''` and makes `init_ps_display()` prefix postmaster child process titles with `postgres: ds12: `, so `stop` can check for surviving child processes too). [guc.c#socket-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3935-L3959) [guc.c#port](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2176-L2184) [guc.c#cluster_name](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4170-L4178) [ps_status.c#init_ps_display](../../../../raw/postgres-12/src/backend/utils/misc/ps_status.c#L265-L321) Files are emptied with `cat /dev/null > <file>`, as the question specifies. Data checksums stay off, the `initdb` default; no leg reads a page of an emptied file.

### The script

```bash
#!/usr/bin/env bash
#
# ds12-run.sh - the measurement script for the wiki page
# wiki/v12/questions/storage-and-vacuum/drop-schema-zeroed-relation-files.md
#
# It builds the pinned PostgreSQL 12 checkout out of tree with a shortened
# RELSEG_SIZE, starts an isolated cluster, creates disposable schemas, empties
# their relation files with "cat /dev/null > <file>" while the server is
# stopped, and then runs DROP SCHEMA with RESTRICT and with CASCADE.  It
# records the errors, the locks a waiting drop holds and waits for, whether a
# table the waiting drop has not reached is still free, the cascade notices,
# and the files left on disk after commit and checkpoint.
#
# The build is NON-DEFAULT in one compile-time knob: RELSEG_SIZE is patched
# from the pin's default 131072 blocks (1 GB) to 128 blocks (1 MB) in the
# generated pg_config.h of the out-of-tree build, because configure's
# --with-segsize accepts only whole gigabytes in PostgreSQL 12.  It lets a
# 3.5 MB table own the numbered segments .1, .2 and .3.  The unlink loop
# under test, mdunlinkfork(), names segments by number and never compares
# them with RELSEG_SIZE.
#
# Every fixture object is disposable.  The script creates and drops its own
# database ds and the roles ds12_owner, ds12_member and ds12_other in its own
# cluster; do not point it at a cluster anyone cares about.
#
# Files are emptied with "cat /dev/null > <file>", as the question specifies.
# That this truncates a file to zero length is shell behavior, which the
# pinned checkout cannot establish, so the empty stage asserts afterwards that
# every remaining file of s and s2 is 0 bytes.
#
# Usage, from the repository root:
#   bash .wiki-runtime/tmp/ds12-run.sh                    # default stages
#   bash .wiki-runtime/tmp/ds12-run.sh fixture empty restrict locks
#   bash .wiki-runtime/tmp/ds12-run.sh stop clean         # tear down
#
# Stages: build cluster fixture empty restrict locks owner rollback cascade
#         missing report stop clean
#
# Environment: PORT12 JOBS.  Discard inherited libpq PG* settings so a
# PGSERVICE or PGHOSTADDR cannot redirect a disposable fixture's DROP commands.
set -uo pipefail
set -e
# A write to a session that has already exited must fail as an error that
# runs the exit trap, not kill the shell with SIGPIPE.
trap '' PIPE

for var in ${!PG@}; do unset "$var"; done

WIKI_ROOT="$PWD"
SRC12="$WIKI_ROOT/raw/postgres-12"
SANDBOX="$WIKI_ROOT/.wiki-runtime/tmp/ds12"
PIN12=45b88269a353ad93744772791feb6d01bc7e1e42
PORT12="${PORT12:-55444}"
JOBS="${JOBS:-8}"
DB=ds

# RELSEG_SIZE for the patched build, in blocks (1 MB at BLCKSZ 8192).
RELSEG_BLOCKS=128

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; DATA="$SANDBOX/data"
OUT="$SANDBOX/out"; SOCK="$SANDBOX/sock"; BIN="$INST/bin"
export PGPORT="$PORT12" PGHOST="$SOCK"

# statement_timeout and lock_timeout are PGC_USERSET in PostgreSQL 12.  They
# travel through PGOPTIONS, so every session is bounded; the lock probes set
# a shorter lock_timeout themselves with SET LOCAL.
export PGOPTIONS="-c statement_timeout=5min -c lock_timeout=60s"

# -X ignores ~/.psqlrc.  ON_ERROR_STOP makes psql stop with exit status 3 at
# the first error in SQL read from standard input; VERBOSITY=verbose prints
# the SQLSTATE of every error and notice.
PSQL=("$BIN/psql" -X -q -At -v ON_ERROR_STOP=1 -v VERBOSITY=verbose)

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }
pass() { printf 'PASS %s\n' "$*" | tee -a "$OUT/results.txt" >&2; }

# Background psql sessions started by the locks stage.
BG=""

owned() {
  [ -f "$SANDBOX/.ds12-owned" ] &&
    [ "$(cat "$SANDBOX/.ds12-owned")" = "$PIN12 $WIKI_ROOT" ]
}

# A failing stage signals its background sessions and attempts to stop the
# cluster.  Output is kept for diagnosis.  If shutdown fails, leave the marker
# so stop can retry.
on_exit() {
  local rc=$? p
  trap - EXIT
  { exec 3>&-; } 2> /dev/null || true
  for p in $BG; do kill "$p" 2> /dev/null || true; done
  if [ "$rc" -ne 0 ] && [ -f "$SANDBOX/.started" ] && owned; then
    if "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop >&2; then
      rm -f "$SANDBOX/.started"
    else
      note "shutdown not confirmed; keeping .started for stop clean"
    fi
  fi
  exit "$rc"
}
trap on_exit EXIT

# This script owns exactly one short sandbox name.  Never adopt a pre-existing
# cluster or delete a directory that this script did not create.
own_sandbox() {
  case "$WIKI_ROOT" in
    */postgres-llm-wiki) ;;
    *) die "run from the postgres-llm-wiki repository root" ;;
  esac
  [ -f "$WIKI_ROOT/AGENTS.md" ] || die "repository root not found"
  owned || die "sandbox is not owned by this script: $SANDBOX"
  export TMPDIR="$SANDBOX/tmp"
  mkdir -p "$TMPDIR" "$OUT"
}
source_pin() {
  [ "$(git --no-optional-locks -C "$SRC12" rev-parse HEAD)" = "$PIN12" ] \
    || die "PostgreSQL 12 checkout is not at $PIN12"
  [ -z "$(git --no-optional-locks -C "$SRC12" status --porcelain --untracked-files=no)" ] \
    || die "PostgreSQL 12 checkout has tracked changes"
}

sql() { # sql <db>: run the SQL on standard input; any error is fatal
  "${PSQL[@]}" -d "$1" || die "SQL failed on $1"
}
assert_sql_target() {
  local target
  [ -f "$SANDBOX/.started" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1 \
    || die "owned cluster is not running"
  target=$("${PSQL[@]}" -d postgres -c 'SHOW /* wiki_ds12_target */ data_directory') \
    || die "cannot verify the fixture connection"
  [ "$target" = "$DATA" ] || die "SQL target is $target, not $DATA"
}
tcp_port_free() {
  [ -x "$BIN/tcp_port_free" ] || die "build the tcp_port_free helper first"
  "$BIN/tcp_port_free" "$PORT12" || die "TCP port $PORT12 is not free on 127.0.0.1"
}
must_fail() { # must_fail <name> <sqlstate> <db>: the SQL on stdin must stop with <sqlstate>
  local name=$1 state=$2 db=$3 rc=0
  "${PSQL[@]}" -d "$db" > "$OUT/$name.txt" 2>&1 || rc=$?
  [ "$rc" -eq 3 ] || { cat "$OUT/$name.txt" >&2; die "$name: psql exit $rc, expected 3"; }
  grep -q "ERROR:  $state:" "$OUT/$name.txt" \
    || { cat "$OUT/$name.txt" >&2; die "$name: no $state error"; }
}
expect_line() { # expect_line <file> <exact line>
  grep -F -x -q -- "$2" "$OUT/$1" || { cat "$OUT/$1" >&2; die "$1 lacks the line: $2"; }
}
expect_text() { # expect_text <file> <fixed text>
  grep -F -q -- "$2" "$OUT/$1" || { cat "$OUT/$1" >&2; die "$1 lacks: $2"; }
}

# wait_until <what> <SQL boolean>: poll every 0.1 s, for at most 30 s.
# <what> goes into a SQL string literal, so it must not contain a quote.
wait_until() {
  sql "$DB" > /dev/null <<SQL
DO /* wiki_ds12_wait */ \$\$
BEGIN
  FOR i IN 1..300 LOOP
    IF ($2) THEN RETURN; END IF;
    PERFORM pg_sleep(0.1);
  END LOOP;
  RAISE EXCEPTION 'timed out waiting until: $1';
END
\$\$;
SQL
  note "reached: $1"
}

server_start() {
  # Mark the intent before pg_ctl: a partial start is still ours to stop.
  printf '%s\n' "$DATA" > "$SANDBOX/.started"
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w -t 60 start > /dev/null 2>&1 \
    || die "pg_ctl start failed, see $OUT/server.log"
}
server_stop() {
  "$BIN/pg_ctl" -D "$DATA" -m fast -w -t 60 stop > /dev/null 2>&1 || die "pg_ctl stop failed"
  rm -f "$SANDBOX/.started"
}

# inventory <schema>: name|relkind|path of every relation with storage in the
# schema, plus the TOAST tables of its tables and their TOAST indexes.  Saved
# while the catalog rows still exist; later snapshots reuse the paths.
inventory() {
  sql "$DB" > "$OUT/inv_$1.txt" <<SQL
SELECT /* wiki_ds12_inventory */ c.relname || '|' || c.relkind || '|' || pg_relation_filepath(c.oid)
FROM pg_class AS c
WHERE c.relkind IN ('r', 'i', 'S', 'm', 't')
  AND (c.relnamespace = '$1'::regnamespace
       OR c.oid IN (SELECT reltoastrelid FROM pg_class WHERE relnamespace = '$1'::regnamespace)
       OR c.oid IN (SELECT i.indexrelid FROM pg_index AS i
                    WHERE i.indrelid IN (SELECT reltoastrelid FROM pg_class
                                         WHERE relnamespace = '$1'::regnamespace)))
ORDER BY c.relname;
SQL
  note "inv_$1.txt: $(wc -l < "$OUT/inv_$1.txt" | tr -d '[:space:]') relations with storage"
}
# files_of <path>: every existing file of one relation with its length in
# bytes: the first segment, the numbered segments, and the other forks.
files_of() {
  local f
  for f in "$DATA/$1" "$DATA/$1".[0-9]* "$DATA/$1"_*; do
    [ -e "$f" ] || continue
    printf '%s %s\n' "${f#"$DATA"/}" "$(wc -c < "$f" | tr -d '[:space:]')"
  done
}
# snapshot <schema> <label>: files_of for every inventoried relation.
snapshot() {
  local rel kind p
  : > "$OUT/files_$1_$2.txt"
  while IFS='|' read -r rel kind p; do
    files_of "$p" >> "$OUT/files_$1_$2.txt"
  done < "$OUT/inv_$1.txt"
  note "files_$1_$2.txt: $(wc -l < "$OUT/files_$1_$2.txt" | tr -d '[:space:]') files"
}
path_of() { # path_of <schema> <relname>: the inventoried path
  grep -E "^$2[|]" "$OUT/inv_$1.txt" | cut -d'|' -f3
}

# --------------------------------------------------------------- build -------
keep_build_logs() {
  local l
  for l in configure make install; do
    [ -f "$BUILD/$l.log" ] && cp "$BUILD/$l.log" "$OUT/$l.log"
  done
  return 0
}
stage_build() {
  say "build 12.2 out of tree from $SRC12, RELSEG_SIZE=$RELSEG_BLOCKS"
  source_pin
  if [ -e "$SANDBOX" ]; then
    own_sandbox
  else
    mkdir -p "$SANDBOX" || die "could not create sandbox"
    printf '%s %s\n' "$PIN12" "$WIKI_ROOT" > "$SANDBOX/.ds12-owned"
    own_sandbox
  fi
  mkdir -p "$BUILD"
  if [ -x "$BIN/postgres" ] && [ -x "$BIN/tcp_port_free" ]; then
    [ "$(cat "$SANDBOX/.build-spec" 2> /dev/null)" = "$PIN12 $RELSEG_BLOCKS" ] \
      || die "existing build has a different pin or segment size"
    note "matching build already present"
    return 0
  fi
  [ -x "$SRC12/configure" ] || die "no pinned checkout at $SRC12"
  # Default configure options apart from the install prefix.
  ( cd "$BUILD" && "$SRC12/configure" --prefix="$INST" > configure.log 2>&1 ) \
    || { keep_build_logs; die "configure failed, see $OUT/configure.log"; }
  # Non-default compile-time knob for this measurement: configure accepts
  # --with-segsize only in whole gigabytes, so the generated pg_config.h is
  # patched to a 1 MB segment instead of building multi-gigabyte fixtures.
  sed "s/^#define RELSEG_SIZE .*/#define RELSEG_SIZE $RELSEG_BLOCKS/" \
      "$BUILD/src/include/pg_config.h" > "$BUILD/src/include/pg_config.h.new" \
    || die "could not patch RELSEG_SIZE"
  mv "$BUILD/src/include/pg_config.h.new" "$BUILD/src/include/pg_config.h"
  sed -n '/^#define RELSEG_SIZE /{p;q;}' "$BUILD/src/include/pg_config.h" | tee "$OUT/relseg_size.txt"
  [ "$(sed -n 's/^#define RELSEG_SIZE[[:space:]]*//p' "$OUT/relseg_size.txt")" = "$RELSEG_BLOCKS" ] \
    || die "RELSEG_SIZE patch did not take"
  ( cd "$BUILD" && make -j"$JOBS" > make.log 2>&1 && make install > install.log 2>&1 ) \
    || { keep_build_logs; grep 'error:' "$BUILD/make.log" | head -n 3 >&2
         die "make failed, see $OUT/make.log"; }
  cat > "$BUILD/tcp_port_free.c" <<'EOF'
#include <arpa/inet.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <unistd.h>
int main(int argc, char **argv) {
  int fd, port, rc;
  struct sockaddr_in sa;
  if (argc != 2) return 2;
  port = atoi(argv[1]);
  if (port <= 0 || port > 65535) return 2;
  fd = socket(AF_INET, SOCK_STREAM, 0);
  if (fd < 0) return 1;
  sa.sin_family = AF_INET;
  sa.sin_port = htons((unsigned short) port);
  sa.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
  rc = bind(fd, (struct sockaddr *) &sa, sizeof(sa));
  close(fd);
  return rc == 0 ? 0 : 1;
}
EOF
  local cc_bin
  cc_bin=$(sed -n 's/^CC = //p' "$BUILD/src/Makefile.global")
  [ -n "$cc_bin" ] || cc_bin=cc
  $cc_bin "$BUILD/tcp_port_free.c" -o "$BIN/tcp_port_free" \
    || die "could not compile tcp_port_free helper"
  keep_build_logs
  printf '%s %s\n' "$PIN12" "$RELSEG_BLOCKS" > "$SANDBOX/.build-spec"
  note "$("$BIN/postgres" --version)"
}

# ------------------------------------------------------------- cluster -------
# Cluster settings, all written before the first start:
#   listen_addresses, unix_socket_directories, port, cluster_name  PGC_POSTMASTER
#   autovacuum, checkpoint_timeout                                 PGC_SIGHUP
# autovacuum is off so no worker takes locks on, reads or truncates a fixture
# table between stages.  checkpoint_timeout is 1 h so that no timed checkpoint
# runs between a committed drop and the file listing that follows it; the
# script then issues CHECKPOINT itself.  Data checksums are off by default.
stage_cluster() {
  say "isolated 12.2 cluster, socket-only, port $PORT12"
  own_sandbox
  source_pin
  [ "$(cat "$SANDBOX/.build-spec" 2> /dev/null)" = "$PIN12 $RELSEG_BLOCKS" ] \
    || die "build specification mismatch"
  mkdir -p "$SOCK"
  if "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    [ -f "$SANDBOX/.started" ] || die "refusing to adopt an unowned running server"
    note "already running"
  else
    [ -z "$(ls -A "$SOCK")" ] || die "socket directory is not free"
    tcp_port_free
    if [ ! -d "$DATA" ]; then
      "$BIN/initdb" -D "$DATA" -A trust --locale=C --encoding=UTF8 \
        > "$OUT/initdb.log" 2>&1 || die "initdb failed, see $OUT/initdb.log"
      {
        printf "listen_addresses = ''\n"
        printf "unix_socket_directories = '%s'\n" "$SOCK"
        printf "port = %s\n" "$PORT12"
        printf "cluster_name = 'ds12'\n"
        printf "autovacuum = off\n"
        printf "checkpoint_timeout = '1h'\n"
      } >> "$DATA/postgresql.conf"
    fi
    server_start
  fi
  assert_sql_target
  {
    uname -srm
    "$BIN/postgres" --version
    printf 'RELSEG_SIZE from build: %s\n' "$(cat "$OUT/relseg_size.txt" 2> /dev/null)"
    sql postgres <<'SQL'
SELECT /* wiki_ds12_platform */ name || '=' || setting || coalesce(' ' || unit, '') || ' (' || context || ')'
FROM pg_settings
WHERE name IN ('block_size', 'segment_size', 'data_checksums', 'shared_buffers',
               'autovacuum', 'checkpoint_timeout', 'cluster_name', 'deadlock_timeout',
               'default_transaction_isolation', 'lock_timeout', 'statement_timeout')
ORDER BY name;
SELECT /* wiki_ds12_platform */ 'max_data_alignment=' || max_data_alignment
       || ' blocks_per_segment=' || blocks_per_segment FROM pg_control_init();
SQL
  } > "$OUT/platform.txt"
  cat "$OUT/platform.txt" >&2
  expect_line platform.txt "block_size=8192 (internal)"
  grep -q "blocks_per_segment=$RELSEG_BLOCKS\$" "$OUT/platform.txt" || die "segment size mismatch"
}

# ------------------------------------------------------------- fixture -------
# Disposable fixture.  Schema s is the schema whose files are emptied; s2 is
# the missing-segment case; s_own is an empty schema for the ownership check;
# other holds the tables that survive.  Rerunning this stage starts over.
stage_fixture() {
  say "fixture: database ds, schemas s, s2, s_own, other"
  own_sandbox
  assert_sql_target
  [ -z "$BG" ] || die "background sessions still running"
  rm -f "$SANDBOX/.emptied" "$OUT/results.txt"
  sql postgres > /dev/null <<'SQL'
DROP /* wiki_ds12_fixture */ DATABASE IF EXISTS ds;
DROP /* wiki_ds12_fixture */ ROLE IF EXISTS ds12_member, ds12_other, ds12_owner;
CREATE /* wiki_ds12_fixture */ ROLE ds12_owner;
CREATE /* wiki_ds12_fixture */ ROLE ds12_member IN ROLE ds12_owner;
CREATE /* wiki_ds12_fixture */ ROLE ds12_other;
CREATE /* wiki_ds12_fixture */ DATABASE ds;
SQL
  sql "$DB" > /dev/null <<'SQL'
CREATE /* wiki_ds12_fixture */ SCHEMA s;
CREATE /* wiki_ds12_fixture */ SCHEMA s2;
CREATE /* wiki_ds12_fixture */ SCHEMA other;
CREATE /* wiki_ds12_fixture */ SCHEMA s_own AUTHORIZATION ds12_owner;

-- s.t: a serial key, its B-tree index, and TOAST data.  Each body is 100
-- md5 strings, 3,200 bytes of hex digits; the stage checks below that the
-- TOAST table ends up non-empty.
CREATE /* wiki_ds12_fixture */ TABLE s.t (id serial PRIMARY KEY, body text);
INSERT /* wiki_ds12_fixture */ INTO s.t (body)
  SELECT (SELECT string_agg(md5(i || ':' || g), '') FROM generate_series(1, 100) AS g)
  FROM generate_series(1, 200) AS i;

-- s.big and s2.big: 15,000 rows of about 236 bytes, about 3.5 MB, so with
-- 1 MB segments each owns the numbered segments .1, .2 and .3.
CREATE /* wiki_ds12_fixture */ TABLE s.big (id int, pad text);
INSERT /* wiki_ds12_fixture */ INTO s.big SELECT i, repeat('x', 200) FROM generate_series(1, 15000) AS i;
CREATE /* wiki_ds12_fixture */ INDEX big_id_idx ON s.big (id);
CREATE /* wiki_ds12_fixture */ TABLE s2.big (id int, pad text);
INSERT /* wiki_ds12_fixture */ INTO s2.big SELECT i, repeat('x', 200) FROM generate_series(1, 15000) AS i;

CREATE /* wiki_ds12_fixture */ SEQUENCE s.seq;
SELECT /* wiki_ds12_fixture */ nextval('s.seq');
CREATE /* wiki_ds12_fixture */ MATERIALIZED VIEW s.mv AS SELECT id FROM s.big WHERE id % 10 = 0;
CREATE /* wiki_ds12_fixture */ VIEW s.v AS SELECT id FROM s.t;
CREATE /* wiki_ds12_fixture */ FUNCTION s.f() RETURNS int LANGUAGE sql AS 'SELECT 1';

-- Foreign keys across the schema boundary in both directions.
CREATE /* wiki_ds12_fixture */ TABLE other.parent (id int PRIMARY KEY);
INSERT /* wiki_ds12_fixture */ INTO other.parent SELECT generate_series(1, 10);
CREATE /* wiki_ds12_fixture */ TABLE s.child (id int, pid int REFERENCES other.parent (id));
INSERT /* wiki_ds12_fixture */ INTO s.child SELECT i, i FROM generate_series(1, 10) AS i;
CREATE /* wiki_ds12_fixture */ TABLE s.parent (id int PRIMARY KEY);
INSERT /* wiki_ds12_fixture */ INTO s.parent SELECT generate_series(1, 10);
CREATE /* wiki_ds12_fixture */ TABLE other.child_ref (id int, pid int REFERENCES s.parent (id));
INSERT /* wiki_ds12_fixture */ INTO other.child_ref SELECT i, i FROM generate_series(1, 10) AS i;

-- A trigger on a table outside s whose trigger function lives in s.
CREATE /* wiki_ds12_fixture */ FUNCTION s.trg_fn() RETURNS trigger LANGUAGE plpgsql
  AS 'BEGIN RETURN NEW; END';
CREATE /* wiki_ds12_fixture */ TABLE other.audit (id int);
CREATE /* wiki_ds12_fixture */ TRIGGER audit_trg BEFORE INSERT ON other.audit
  FOR EACH ROW EXECUTE FUNCTION s.trg_fn();

-- VACUUM writes the free space map and visibility map forks.
VACUUM /* wiki_ds12_fixture */ (ANALYZE) s.t, s.big, s.child, s.parent, s.mv, s2.big;
SQL
  sql "$DB" > "$OUT/fixture_state.txt" <<'SQL'
SELECT /* wiki_ds12_fixture */ 'relations_in_s=' || count(*) FROM pg_class WHERE relnamespace = 's'::regnamespace;
SELECT /* wiki_ds12_fixture */ 't_toast_bytes_positive=' || (pg_relation_size(reltoastrelid) > 0) FROM pg_class WHERE oid = 's.t'::regclass;
SELECT /* wiki_ds12_fixture */ 'child_ref_fk=' || count(*) FROM pg_constraint WHERE conrelid = 'other.child_ref'::regclass AND contype = 'f';
SELECT /* wiki_ds12_fixture */ 'audit_user_triggers=' || count(*) FROM pg_trigger WHERE tgrelid = 'other.audit'::regclass AND NOT tgisinternal;
SELECT /* wiki_ds12_fixture */ 'parent_ri_triggers=' || count(*) FROM pg_trigger WHERE tgrelid = 'other.parent'::regclass AND tgisinternal;
SQL
  cat "$OUT/fixture_state.txt" >&2
  expect_line fixture_state.txt "t_toast_bytes_positive=true"
  expect_line fixture_state.txt "child_ref_fk=1"
  expect_line fixture_state.txt "audit_user_triggers=1"
  expect_line fixture_state.txt "parent_ri_triggers=2"
  inventory s
  inventory s2
  snapshot s fixture
  snapshot s2 fixture
  local p q
  p=$(path_of s big); q=$(path_of s2 big)
  for f in "$p.1" "$p.2" "$p.3" "${p}_fsm" "${p}_vm" "$q.1" "$q.2" "$q.3" "${q}_fsm" "${q}_vm"; do
    grep -q "^$f [1-9]" "$OUT/files_s_fixture.txt" "$OUT/files_s2_fixture.txt" \
      || die "fixture file missing or empty: $f"
  done
  touch "$SANDBOX/.fixture"
  pass "fixture: s.big and s2.big own segments .1 .2 .3 plus _fsm and _vm; s.t has TOAST data"
}

# --------------------------------------------------------------- empty -------
# While the server is stopped, every file of every relation in s becomes zero
# length.  The same happens in s2, and s2.big's emptied segment .1 is then
# removed.
stage_empty() {
  say "empty the relation files while the server is stopped"
  own_sandbox
  [ -f "$SANDBOX/.fixture" ] || die "run the fixture stage first"
  [ ! -f "$SANDBOX/.emptied" ] || die "files already emptied; re-run fixture first"
  server_stop
  local rel kind p f q
  for sch in s s2; do
    while IFS='|' read -r rel kind p; do
      for f in "$DATA/$p" "$DATA/$p".[0-9]* "$DATA/$p"_*; do
        [ -e "$f" ] || continue
        cat /dev/null > "$f"
      done
    done < "$OUT/inv_$sch.txt"
  done
  q=$(path_of s2 big)
  rm "$DATA/$q.1"
  touch "$SANDBOX/.emptied"
  server_start
  snapshot s emptied
  snapshot s2 emptied
  [ -z "$(grep -v ' 0$' "$OUT/files_s_emptied.txt")" ] || die "a file of s is not empty"
  [ -z "$(grep -v ' 0$' "$OUT/files_s2_emptied.txt")" ] || die "a file of s2 is not empty"
  grep -q "^$q.1 " "$OUT/files_s2_emptied.txt" && die "s2.big segment .1 still exists"
  pass "empty: $(wc -l < "$OUT/files_s_emptied.txt" | tr -d '[:space:]') files of s and $(wc -l < "$OUT/files_s2_emptied.txt" | tr -d '[:space:]') files of s2 at 0 bytes; s2.big .1 removed"
}

# ------------------------------------------------------------ restrict -------
stage_restrict() {
  say "plain DROP SCHEMA, which means RESTRICT"
  own_sandbox
  assert_sql_target
  [ -f "$SANDBOX/.emptied" ] || die "run the empty stage first"
  must_fail restrict 2BP01 "$DB" <<'SQL'
DROP /* wiki_ds12_restrict */ SCHEMA s;
SQL
  expect_text restrict.txt "cannot drop schema s because other objects depend on it"
  expect_text restrict.txt "table s.t depends on schema s"
  expect_text restrict.txt "table s.big depends on schema s"
  snapshot s after_restrict
  cmp -s "$OUT/files_s_emptied.txt" "$OUT/files_s_after_restrict.txt" || die "files changed"
  pass "restrict: DROP SCHEMA s refused with 2BP01 ($(grep -c 'depends on' "$OUT/restrict.txt") dependents listed); files unchanged"
}

# --------------------------------------------------------------- locks -------
# Session A holds AccessShareLock on s.t.  A plain DROP SCHEMA (session D)
# then queues for AccessExclusiveLock on s.t before RESTRICT is checked.
# Tables the walk has not reached stay unlocked while it waits.
stage_locks() {
  say "RESTRICT locks dependents during the walk, before it refuses"
  own_sandbox
  assert_sql_target
  [ -f "$SANDBOX/.emptied" ] || die "run the empty stage first"
  local fifo="$SANDBOX/a.fifo" a_pid d_pid rc unreached dl
  # Session A's 500 ms lock_timeout must stay below deadlock_timeout.  A
  # request queued behind the waiting drop would form a soft deadlock with it,
  # which only A's own deadlock check, after deadlock_timeout, could resolve by
  # reordering the queue; under the shorter bound it times out first instead.
  dl=$("${PSQL[@]}" -d "$DB" -c 'SHOW /* wiki_ds12_locks */ deadlock_timeout') \
    || die "cannot read deadlock_timeout"
  [ "$dl" = "1s" ] || die "deadlock_timeout is $dl, not the default 1s"
  rm -f "$fifo"
  mkfifo "$fifo"
  "${PSQL[@]}" -d "$DB" < "$fifo" > "$OUT/locks_a.txt" 2>&1 &
  a_pid=$!; BG="$BG $a_pid"
  exec 3> "$fifo"
  cat >&3 <<'SQL'
BEGIN /* wiki_ds12_locks_a */;
SET /* wiki_ds12_locks_a */ LOCAL lock_timeout = '500ms';
LOCK /* wiki_ds12_locks_a */ TABLE s.t IN ACCESS SHARE MODE;
SELECT /* wiki_ds12_locks_a */ 'a: AccessShareLock held';
SQL
  wait_until "session A holds AccessShareLock on s.t" \
    "EXISTS (SELECT 1 FROM pg_locks WHERE relation = 's.t'::regclass AND mode = 'AccessShareLock' AND granted)"

  # Baseline: with no drop waiting, a new session gets the same lock at once.
  sql "$DB" > "$OUT/locks_c0.txt" 2>&1 <<'SQL'
BEGIN /* wiki_ds12_locks_c0 */;
SET /* wiki_ds12_locks_c0 */ LOCAL lock_timeout = '1s';
LOCK /* wiki_ds12_locks_c0 */ TABLE s.t IN ACCESS SHARE MODE;
SELECT /* wiki_ds12_locks_c0 */ 'c0: AccessShareLock granted';
COMMIT /* wiki_ds12_locks_c0 */;
SQL
  expect_line locks_c0.txt "c0: AccessShareLock granted"
  pass "locks: with no drop waiting, a new session got AccessShareLock on s.t at once"

  "${PSQL[@]}" -d "$DB" > "$OUT/locks_d.txt" 2>&1 3>&- <<'SQL' &
DROP /* wiki_ds12_locks_d */ SCHEMA s;
SQL
  d_pid=$!; BG="$BG $d_pid"
  wait_until "the drop waits for AccessExclusiveLock on s.t" \
    "EXISTS (SELECT 1 FROM pg_locks WHERE relation = 's.t'::regclass AND mode = 'AccessExclusiveLock' AND NOT granted)"
  sql "$DB" > "$OUT/locks_d_held.txt" <<'SQL'
SELECT /* wiki_ds12_locks */ l.locktype || '|'
       || coalesce(l.relation::regclass::text, l.classid::regclass::text || ':' || coalesce(n.nspname, l.objid::text))
       || '|' || l.mode || '|' || l.granted
FROM pg_locks AS l
LEFT JOIN pg_namespace AS n ON l.classid = 'pg_namespace'::regclass AND n.oid = l.objid
WHERE l.pid = (SELECT pid FROM pg_locks
               WHERE relation = 's.t'::regclass AND mode = 'AccessExclusiveLock' AND NOT granted)
  AND l.locktype IN ('relation', 'object')
ORDER BY 1;
SQL
  cat "$OUT/locks_d_held.txt" >&2
  expect_line locks_d_held.txt "object|pg_namespace:s|AccessExclusiveLock|true"
  expect_line locks_d_held.txt "relation|s.t_id_seq|AccessExclusiveLock|true"
  expect_line locks_d_held.txt "relation|s.t|AccessExclusiveLock|false"
  pass "locks: the RESTRICT drop holds the schema and s.t_id_seq locks and waits for AccessExclusiveLock on s.t"

  # A session that holds no lock on s.t now queues behind the waiting drop.
  must_fail locks_c1 55P03 "$DB" <<'SQL'
BEGIN /* wiki_ds12_locks_c1 */;
SET /* wiki_ds12_locks_c1 */ LOCAL lock_timeout = '1s';
LOCK /* wiki_ds12_locks_c1 */ TABLE s.t IN ACCESS SHARE MODE;
SQL
  expect_text locks_c1.txt "canceling statement due to lock timeout"
  pass "locks: while the drop waits, a session holding no lock timed out (55P03) asking for AccessShareLock"

  # A table of s that the waiting drop holds no lock on has not been reached
  # by the walk yet, so a new session gets AccessShareLock on it at once.
  sql "$DB" > "$OUT/locks_unreached.txt" <<'SQL'
SELECT /* wiki_ds12_locks */ c.oid::regclass::text
FROM pg_class AS c
WHERE c.relnamespace = 's'::regnamespace AND c.relkind = 'r'
  AND NOT EXISTS (SELECT 1 FROM pg_locks AS l
                  WHERE l.relation = c.oid
                    AND l.pid = (SELECT pid FROM pg_locks
                                 WHERE relation = 's.t'::regclass
                                   AND mode = 'AccessExclusiveLock' AND NOT granted))
ORDER BY 1
LIMIT 1;
SQL
  unreached=$(cat "$OUT/locks_unreached.txt")
  [ -n "$unreached" ] || die "the waiting drop already holds every table of s"
  sql "$DB" > "$OUT/locks_c2.txt" 2>&1 <<SQL
BEGIN /* wiki_ds12_locks_c2 */;
SET /* wiki_ds12_locks_c2 */ LOCAL lock_timeout = '1s';
LOCK /* wiki_ds12_locks_c2 */ TABLE $unreached IN ACCESS SHARE MODE;
SELECT /* wiki_ds12_locks_c2 */ 'c2: AccessShareLock granted on $unreached';
COMMIT /* wiki_ds12_locks_c2 */;
SQL
  expect_line locks_c2.txt "c2: AccessShareLock granted on $unreached"
  pass "locks: while the drop waits for s.t, a new session got AccessShareLock at once on $unreached, which the walk has not reached"

  # The drop's own deadlock check runs once, deadlock_timeout after it began
  # waiting.  The drop was already waiting before the 1 s blocked probe above,
  # and A asks only once the drop's statement has run for 2 s, so that check
  # has run; only A's own check, after another deadlock_timeout, could reorder
  # the queue for a request of A's.  The condition uses the statement's start
  # time, which precedes the wait.
  wait_until "the drop statement has run for 2 s, twice deadlock_timeout" \
    "EXISTS (SELECT 1 FROM pg_stat_activity AS a WHERE a.pid = (SELECT pid FROM pg_locks WHERE relation = 's.t'::regclass AND mode = 'AccessExclusiveLock' AND NOT granted) AND clock_timestamp() - a.query_start > interval '2 s')"

  # Session A asks again for the mode it holds, then for a new mode, both
  # under its 500 ms lock_timeout, which is below deadlock_timeout.
  cat >&3 <<'SQL'
LOCK /* wiki_ds12_locks_a */ TABLE s.t IN ACCESS SHARE MODE;
SELECT /* wiki_ds12_locks_a */ 'a: same mode again';
LOCK /* wiki_ds12_locks_a */ TABLE s.t IN ROW EXCLUSIVE MODE;
SELECT /* wiki_ds12_locks_a */ 'a: RowExclusiveLock granted';
SQL
  wait_until "session A holds RowExclusiveLock on s.t" \
    "EXISTS (SELECT 1 FROM pg_locks WHERE relation = 's.t'::regclass AND mode = 'RowExclusiveLock' AND granted)"
  wait_until "the drop still waits" \
    "EXISTS (SELECT 1 FROM pg_locks WHERE relation = 's.t'::regclass AND mode = 'AccessExclusiveLock' AND NOT granted)"

  # Release A; the drop then finishes its walk and refuses.
  printf '%s\n' "COMMIT /* wiki_ds12_locks_a */;" >&3
  exec 3>&-
  rc=0; wait "$a_pid" || rc=$?
  [ "$rc" -eq 0 ] || { cat "$OUT/locks_a.txt" >&2; die "session A exited $rc"; }
  rc=0; wait "$d_pid" || rc=$?
  BG=""
  rm -f "$fifo"
  [ "$rc" -eq 3 ] || { cat "$OUT/locks_d.txt" >&2; die "the drop exited $rc, expected 3"; }
  expect_text locks_d.txt "ERROR:  2BP01:"
  expect_line locks_a.txt "a: same mode again"
  expect_line locks_a.txt "a: RowExclusiveLock granted"
  pass "locks: A re-acquired AccessShareLock and got RowExclusiveLock within its 500 ms lock_timeout, below the 1 s deadlock_timeout, while the drop waited"
  pass "locks: after A committed, the waiting drop refused with 2BP01"
}

# --------------------------------------------------------------- owner -------
# s_own belongs to ds12_owner.  ds12_member is a member of ds12_owner that
# inherits its privileges; ds12_other is not.  Both drops roll back.
stage_owner() {
  say "the schema ownership check and role membership"
  own_sandbox
  assert_sql_target
  [ -f "$SANDBOX/.fixture" ] || die "run the fixture stage first"
  sql "$DB" > "$OUT/owner_roles.txt" <<'SQL'
SELECT /* wiki_ds12_owner */ 'owner=' || nspowner::regrole FROM pg_namespace WHERE nspname = 's_own';
SELECT /* wiki_ds12_owner */ 'member_inherits=' || rolinherit FROM pg_roles WHERE rolname = 'ds12_member';
SELECT /* wiki_ds12_owner */ 'member_of_owner=' || pg_has_role('ds12_member', 'ds12_owner', 'MEMBER');
SQL
  expect_line owner_roles.txt "owner=ds12_owner"
  expect_line owner_roles.txt "member_inherits=true"
  expect_line owner_roles.txt "member_of_owner=true"
  sql "$DB" > "$OUT/owner_member.txt" 2>&1 <<'SQL'
BEGIN /* wiki_ds12_owner_member */;
SET /* wiki_ds12_owner_member */ LOCAL ROLE ds12_member;
DROP /* wiki_ds12_owner_member */ SCHEMA s_own;
SELECT /* wiki_ds12_owner_member */ 'member: s_own rows left=' || count(*) FROM pg_namespace WHERE nspname = 's_own';
ROLLBACK /* wiki_ds12_owner_member */;
SQL
  expect_line owner_member.txt "member: s_own rows left=0"
  must_fail owner_other 42501 "$DB" <<'SQL'
BEGIN /* wiki_ds12_owner_other */;
SET /* wiki_ds12_owner_other */ LOCAL ROLE ds12_other;
DROP /* wiki_ds12_owner_other */ SCHEMA s_own;
SQL
  expect_text owner_other.txt "must be owner of schema s_own"
  sql "$DB" > "$OUT/owner_after.txt" <<'SQL'
SELECT /* wiki_ds12_owner */ 's_own rows=' || count(*) FROM pg_namespace WHERE nspname = 's_own';
SQL
  expect_line owner_after.txt "s_own rows=1"
  pass "owner: a non-owner member of the owning role dropped s_own (rolled back); a non-member got 42501"
}

# ------------------------------------------------------------ rollback -------
stage_rollback() {
  say "DROP SCHEMA s CASCADE in a transaction that rolls back"
  own_sandbox
  assert_sql_target
  [ -f "$SANDBOX/.emptied" ] || die "run the empty stage first"
  sql "$DB" > "$OUT/rollback.txt" 2>&1 <<'SQL'
SELECT /* wiki_ds12_rollback */ 'before: relations_in_s=' || count(*) FROM pg_class WHERE relnamespace = 's'::regnamespace;
BEGIN /* wiki_ds12_rollback */;
DROP /* wiki_ds12_rollback */ SCHEMA s CASCADE;
SELECT /* wiki_ds12_rollback */ 'lock|' || c.relname || '|' || l.mode || '|' || l.granted
FROM pg_locks AS l JOIN pg_class AS c ON c.oid = l.relation
WHERE l.pid = pg_backend_pid() AND l.locktype = 'relation'
  AND c.relnamespace = 'other'::regnamespace AND c.relkind = 'r'
ORDER BY 1;
ROLLBACK /* wiki_ds12_rollback */;
SELECT /* wiki_ds12_rollback */ 'after: relations_in_s=' || count(*) FROM pg_class WHERE relnamespace = 's'::regnamespace;
SQL
  expect_text rollback.txt "drop cascades to constraint child_ref_pid_fkey on table other.child_ref"
  expect_text rollback.txt "drop cascades to trigger audit_trg on table other.audit"
  expect_line rollback.txt "lock|parent|AccessExclusiveLock|true"
  expect_line rollback.txt "lock|child_ref|AccessExclusiveLock|true"
  expect_line rollback.txt "lock|audit|AccessExclusiveLock|true"
  [ "$(sed -n 's/^before: //p' "$OUT/rollback.txt")" = "$(sed -n 's/^after: //p' "$OUT/rollback.txt")" ] \
    || die "the rollback did not restore the relation count"
  snapshot s after_rollback
  cmp -s "$OUT/files_s_emptied.txt" "$OUT/files_s_after_rollback.txt" || die "files changed"
  pass "rollback: CASCADE held AccessExclusiveLock on surviving other.parent, other.child_ref and other.audit"
  pass "rollback: after ROLLBACK, $(sed -n 's/^after: //p' "$OUT/rollback.txt"); files still present at 0 bytes"
}

# ------------------------------------------------------------- cascade -------
stage_cascade() {
  say "DROP SCHEMA s CASCADE commits"
  own_sandbox
  assert_sql_target
  [ -f "$SANDBOX/.emptied" ] || die "run the empty stage first"
  sql "$DB" > "$OUT/cascade_check.txt" <<'SQL'
SELECT /* wiki_ds12_cascade */ 'schema_s=' || count(*) FROM pg_namespace WHERE nspname = 's';
SQL
  expect_line cascade_check.txt "schema_s=1"
  sql "$DB" > "$OUT/cascade.txt" 2>&1 <<'SQL'
DROP /* wiki_ds12_cascade */ SCHEMA s CASCADE;
SQL
  snapshot s after_commit
  # After commit each relation keeps only its first main-fork file, emptied
  # and queued for unlink at the next checkpoint.
  cut -d'|' -f3 "$OUT/inv_s.txt" | sed 's/$/ 0/' | sort > "$OUT/expected_s_after_commit.txt"
  sort "$OUT/files_s_after_commit.txt" > "$OUT/sorted_s_after_commit.txt"
  diff "$OUT/expected_s_after_commit.txt" "$OUT/sorted_s_after_commit.txt" > "$OUT/diff_s_after_commit.txt" \
    || { cat "$OUT/diff_s_after_commit.txt" >&2; die "unexpected files after commit"; }
  sql "$DB" > "$OUT/cascade_survivors.txt" <<'SQL'
SELECT /* wiki_ds12_cascade */ 'schema_s=' || count(*) FROM pg_namespace WHERE nspname = 's';
SELECT /* wiki_ds12_cascade */ 'other_tables=' || count(*) FROM pg_class WHERE relnamespace = 'other'::regnamespace AND relkind = 'r';
SELECT /* wiki_ds12_cascade */ 'child_ref_fk=' || count(*) FROM pg_constraint WHERE conrelid = 'other.child_ref'::regclass AND contype = 'f';
SELECT /* wiki_ds12_cascade */ 'audit_user_triggers=' || count(*) FROM pg_trigger WHERE tgrelid = 'other.audit'::regclass AND NOT tgisinternal;
SELECT /* wiki_ds12_cascade */ 'parent_ri_triggers=' || count(*) FROM pg_trigger WHERE tgrelid = 'other.parent'::regclass AND tgisinternal;
SELECT /* wiki_ds12_cascade */ 'child_ref_rows=' || count(*) FROM other.child_ref;
SQL
  cat "$OUT/cascade_survivors.txt" >&2
  for l in schema_s=0 other_tables=3 child_ref_fk=0 audit_user_triggers=0 parent_ri_triggers=0 child_ref_rows=10; do
    expect_line cascade_survivors.txt "$l"
  done
  pass "cascade: committed; $(wc -l < "$OUT/inv_s.txt" | tr -d '[:space:]') first main-fork files remain at 0 bytes, every numbered segment and other fork is gone"
  pass "cascade: other.child_ref survives without its foreign key; other.audit without its trigger; other.parent without its RI triggers"
  sql "$DB" > /dev/null <<'SQL'
CHECKPOINT /* wiki_ds12_cascade */;
SQL
  snapshot s after_checkpoint
  [ ! -s "$OUT/files_s_after_checkpoint.txt" ] \
    || { cat "$OUT/files_s_after_checkpoint.txt" >&2; die "files remain after CHECKPOINT"; }
  pass "cascade: after one CHECKPOINT no file of s remains"
}

# ------------------------------------------------------------- missing -------
stage_missing() {
  say "a missing segment stops the unlink loop; empty segments do not"
  own_sandbox
  assert_sql_target
  [ -f "$SANDBOX/.emptied" ] || die "run the empty stage first"
  local q
  q=$(path_of s2 big)
  sql "$DB" > "$OUT/missing.txt" 2>&1 <<'SQL'
DROP /* wiki_ds12_missing */ SCHEMA s2 CASCADE;
SQL
  snapshot s2 after_commit
  for l in "$q 0" "$q.2 0" "$q.3 0"; do expect_line files_s2_after_commit.txt "$l"; done
  for f in "${q}_fsm" "${q}_vm"; do
    grep -q "^$f " "$OUT/files_s2_after_commit.txt" && die "$f survived the commit"
  done
  sql "$DB" > /dev/null <<'SQL'
CHECKPOINT /* wiki_ds12_missing */;
SQL
  snapshot s2 after_checkpoint
  printf '%s\n' "$q.2 0" "$q.3 0" > "$OUT/expected_s2_after_checkpoint.txt"
  sort "$OUT/files_s2_after_checkpoint.txt" | diff "$OUT/expected_s2_after_checkpoint.txt" - \
    > "$OUT/diff_s2_after_checkpoint.txt" \
    || { cat "$OUT/diff_s2_after_checkpoint.txt" >&2; die "unexpected s2 files after CHECKPOINT"; }
  if grep -F -q 'could not remove file' "$OUT/server.log"; then
    die "the server log has a file-removal warning"
  fi
  pass "missing: with s2.big .1 removed, .2 and .3 survived the commit and the CHECKPOINT as orphans, with no warning"
}

# -------------------------------------------------------------- report -------
stage_report() {
  say "results"
  own_sandbox
  cat "$OUT/results.txt" >&2
  note "read platform.txt first, then results.txt, restrict.txt, locks_d_held.txt and rollback.txt"
}

# ---------------------------------------------------------- stop, clean ------
# Processes still running from this sandbox's binaries: the postmaster's
# command line starts with "$BIN/postgres -D $DATA", a leftover psql's with
# "$BIN/psql", and postmaster child processes rewritten by init_ps_display()
# with "postgres: ds12:".  Anchoring at the start keeps the match off this
# grep and off any calling shell whose own command line mentions the paths.
procs_from_sandbox() {
  ps -A -o args= | grep -E -c -- "^$BIN/| -D $DATA([[:space:]]|\$)|^postgres: ds12:" || true
}
stage_stop() {
  say "stop cluster"
  [ -e "$SANDBOX" ] || { note "sandbox absent"; return 0; }
  own_sandbox
  if [ -x "$BIN/pg_ctl" ] && "$BIN/pg_ctl" -D "$DATA" status > /dev/null 2>&1; then
    [ -f "$SANDBOX/.started" ] || die "refusing to stop an unowned server"
    server_stop
  else
    note "not running"
  fi
  [ ! -f "$DATA/postmaster.pid" ] || die "postmaster.pid survived"
  [ "$(procs_from_sandbox)" -eq 0 ] || die "a process from $BIN or naming $DATA is still running"
  [ ! -d "$SOCK" ] || [ -z "$(ls -A "$SOCK")" ] || die "socket directory is not free"
  if [ -x "$BIN/tcp_port_free" ]; then
    tcp_port_free
  fi
  rm -f "$SANDBOX/.started"
  note "stopped: no pid file, process, or socket; 127.0.0.1:$PORT12 is free"
}
stage_clean() {
  say "delete sandbox $SANDBOX"
  stage_stop
  [ -e "$SANDBOX" ] || return 0
  own_sandbox
  rm -rf "$SANDBOX"
  note "deleted $SANDBOX"
}

# ------------------------------------------------------------- driver --------
stages="$*"
[ $# -eq 0 ] && stages="build cluster fixture empty restrict locks owner rollback cascade missing report"
for st in $stages; do
  case "$(type -t "stage_$st" 2> /dev/null)" in
    function) "stage_$st" ;;
    *) die "unknown stage: $st" ;;
  esac
done
```

## Context Reviewed

- PostgreSQL 12 pin `45b88269a353ad93744772791feb6d01bc7e1e42`; manifests, indexes and recent log entries used for navigation only.
- Grammar and utility dispatch; schema address/ownership; dependency graph, ordering, locks and extension/internal dependencies; relation/index/sequence/schema dispatch and catalog cleanup.
- Relation opening and catalog-derived descriptors; pending storage deletions; buffer discard; all-fork unlink, segment traversal, checkpoint queues; commit/abort callers and commit replay.
- Catalog and object-address structures, generated headers, reverse header inclusion and catalog build; ordinary regression and event-trigger tests.
- Review fix on 2026-10-21: lock acquisition during name lookup and the dependency walk, and the lock manager's wait-queue rule; foreign-key constraint dependencies, action-trigger creation and trigger removal; partition-drop locks; physical-address setup in the relcache; utility dispatch to `ProcessUtilitySlow`; index use in catalog scans, buffer-miss reads, the short-read and missing-segment errors; and the `zero_damaged_pages` and `ignore_system_indexes` definitions.
- Review fix on 2026-10-22: indexed heap fetch and empty-page interpretation; B-tree zero-page rejection; already-open segment bypass; immediate lock reacquisition and the conditional early grant in `ProcSleep()`; and referenced-table membership in the deletion set.
- Second review fix on 2026-10-22: the normal dependency of a foreign key on the referenced columns and the lock `RemoveConstraintById()` takes; trigger-function, column-default and policy dependencies and the locks their removal takes; `CheckTableForSerializableConflictIn()` and the serialization-failure branch; `pg_namespace_ownercheck()`, `has_privs_of_role()` and `roles_has_privs_of()`; the start point of `_mdfd_getseg()`'s size check; the `PGC_SUSET` context of `zero_damaged_pages` and its invalid-header branch; the namespace regression test's post-drop count; and, for the measurement, the GUC definitions it changes, the lock-timeout timer and error site, the SQLSTATE table and the default segment size.
- Third review fix on 2026-10-22: column-subobject locks in `AcquireDeletionLock()` and column removal in `RemoveAttributeById()`; index predicate-lock transfer (`TransferPredicateLocksToHeapRelation()`), `PredicateLockingNeededForRelation()` (`FirstBootstrapObjectId`, local buffers, materialized views) and `PreCommit_CheckForSerializationFailure()`; the C `tcp_port_free` helper replacing the Python port check in the measurement script, per-stage `assert_sql_target` guards, and a fresh end-to-end measurement run.
- Fourth review fix on 2026-10-22: `DropAllPredicateLocksFromTable()`'s non-serializable caller allowance and `OnConflict_CheckForSerializationFailure()`'s `MySerializableXact == writer` immediate-error branch versus later `SXACT_FLAG_DOOMED` / prepared-pivot failure in `PreCommit_CheckForSerializationFailure()`; `AddNewAttributeTuples()` and `add_column_datatype_dependency()` / `add_column_collation_dependency()` `DEPENDENCY_NORMAL` column-type edges versus `StoreAttrDefault()`'s `DEPENDENCY_AUTO` generated-column edge and `pg_proc`'s `DEPENDENCY_NORMAL` schema edge; cross-schema `CASCADE` removal of outside `CHECK`/`EXCLUDE` constraints, indexes, non-`SELECT` rewrite rules (`InsertRule()`, `RemoveRewriteRuleById()`), partitions, inheritance children, typed tables, views and materialized views (`_RETURN` `DEPENDENCY_INTERNAL` rules); multi-schema `DROP SCHEMA` batching in `RemoveObjects()` and `performMultipleDeletions()`; `CatalogTupleUpdate()` / `CatalogIndexInsert()` versus `CatalogTupleDelete()` under `ignore_system_indexes = on`; `PGC_BACKEND` and `PGC_USERSET` GUC contexts; `cluster_name` process-title tagging in `init_ps_display()` for `procs_from_sandbox()` and explicit assertion of the `s.t_id_seq` lock in `stage_locks`.
- Sixth review fix on 2026-10-22: the indirect exclusion-operator dependency in `CreateConstraintEntry()`, the supporting index's operator-class dependency and its internal dependency on the constraint in `index_constraint_create()`, and the dependency walk's redirection to the owning constraint. Split cross-schema cases into dependency/lock/outcome tables and serialization checks into a state table, decision map and numbered walkthrough. Rechecked GNU make and Perl build prerequisites, corrected the `CatalogTupleUpdate()` citation, linked the four existing glossary entries, and reran the unchanged published measurement script.
- Seventh review on 2026-10-22: reread the assembled answer, dependency and serialization maps, catalog-read branches, script, evidence and unresolved cases against the same pin. Corrected the decision-map note's cross-schema section link and repeated the unchanged published measurement from an empty sandbox and with the build present. No new in-scope engine contradiction was found.
- Eighth review on 2026-10-22: rechecked the answer and comparison set, then defined predicate locks, read/write conflicts, dangerous structures, pivots and the doomed/prepared transaction states. Traced `SERIALIZABLEXACT`, `SxactIsPrepared()` and `PreCommit_CheckForSerializationFailure()` through ordinary commit processing, alongside `README-SSI`. Added matching PostgreSQL 12 glossary entries and repeated the unchanged measurement from an empty sandbox and with the build present.
- Ninth review on 2026-10-22: rechecked the assembled explanation and published script. Qualified the serialization map, walkthrough and Evidence Map so the two-edge conflict pattern alone is not presented as an immediate error: `OnConflict_CheckForSerializationFailure()` also applies ordering and read-only checks. Reviewed the linked glossary qualifications and repeated the unchanged script from an empty sandbox and with the build present.
- Tenth review on 2026-10-22: rechecked the current explanation, prior SSI qualification, script, glossary links and comparison set at the same pin. No new in-scope issue was found. Repeated the unchanged script from an empty sandbox and with the build present; both runs passed all 15 assertions.
- Eleventh review on 2026-10-22: how `findDependentObjects()` locks each dependent during its `pg_depend` scan and recurses only afterwards; `StartTransaction()`'s reset of the isolation level, `check_XactIsoLevel()`, and the `BEGIN` isolation-level grammar and option handling; the `role` setting, `check_role()` and `is_member_of_role()`; the absence of ownership or privilege checks in the dependency walk and the removal routines; the script's statement tags and `RELSEG_SIZE` check. Added a probe for a table the waiting drop has not reached and repeated the measurement from an empty sandbox and with the build present.
- Twelfth review on 2026-10-22: rechecked the eleventh review's new text against the same pin. Qualified the walk's lock retention with the branches in which `findDependentObjects()` gives a lock back, limited `default_transaction_isolation`'s effect to session-level, configuration-file and connection-start settings, and tightened two sentences about lock waits. Repeated the unchanged script from an empty sandbox and with the build present; both runs passed all 16 assertions.
- Thirteenth review on 2026-10-22: the deadlock check that a waiting backend runs once after `deadlock_timeout` and its queue reordering in `DeadLockCheck()`, which a 2 s probe bound could not exclude; shared lock-table sizing in `lock.c` and `MaxBackends`, the fast-path eligibility rule and the `out of shared memory` failure; the `DROP SCHEMA`, privileges and role-membership documentation on who may drop a schema; `RemoveObjects()` skipping names that `IF EXISTS` did not find; and `RemoveRewriteRuleById()`'s table lock. Changed the script's session A bound to 500 ms with a `deadlock_timeout` check, and repeated the measurement from an empty sandbox and with the build present; both runs passed all 16 assertions.
- Fourteenth review on 2026-10-22: rechecked the thirteenth review's new text against the same pin. The probe's inference also needed the drop's own single deadlock check to have run before session A asked, so the `locks` stage now waits until the drop's statement has run for 2 s; reworded the settings, Evidence Map and lock-table statements to match `findDependentObjects()`'s give-back branches and the probe's scope, and named the documentation sections correctly. Repeated the measurement from an empty sandbox and with the build present; both runs passed all 16 assertions.
- Fifteenth review on 2026-10-22: the `locks` stage's 2-second condition measures time since the drop statement's `query_start`, not since its lock wait began, so the page now says that the drop statement has run for 2 s and that the drop was already waiting before the 1-second blocked probe. The first re-run found that the new wait description's apostrophe broke the SQL string literal in `wait_until`; the description lost the apostrophe and `wait_until` documents the restriction. Repeated the measurement from an empty sandbox and with the build present; both runs passed all 16 assertions.
- Sixteenth review on 2026-10-22: rechecked the fifteenth review's wording against the script's `query_start` condition, the `wait_until` quoting restriction, the extracted script and the cited documentation; no new in-scope issue was found. Repeated the unchanged script from an empty sandbox and with the build present; both runs passed all 16 assertions.
- Seventeenth review on 2026-10-22: checked that every output file the Usage table and the Evidence Map name is produced by a run, and rechecked the run record, runtime estimate and citations; no in-scope issue was found. Repeated the unchanged script from an empty sandbox and with the build present; both runs passed all 16 assertions.
- Eighteenth review on 2026-10-28: the lock table's sizing in `InitLocks()` and `LockShmemSize()`, `ShmemInitHash()`'s statement that the maximum size is not a hard limit, the `LOCK` and `PROCLOCK` allocations in `SetupLockInTable()`, and the `max_locks_per_transaction` documentation and help text; the `empty` stage's order of emptying and removal for `s2`. The lock-table statements now describe an estimate rather than a fixed capacity, the documentation discrepancy is filed under Open Questions, and the `s2` wording follows the script. The script changed only in a comment. Repeated the measurement from an empty sandbox and twice with the build present; all three runs passed all 16 assertions.
- Nineteenth review on 2026-10-28: rechecked the eighteenth review's text against the same pin. `SetupLockInTable()` reuses an existing `LOCK` entry and always adds a `PROCLOCK`, and its `PROCLOCK` failure returns at line 1197, so the lock-table row and its citations now say so. The Usage runtime now matches the last run. Also rechecked the lock-queue branches, commit ordering and file cleanup, the regression outputs, `check_role()`, `check_XactIsoLevel()` and the TOAST and namespace dependencies; no other issue was found. Repeated the unchanged script from an empty sandbox and with the build present; both runs passed all 16 assertions.
- Twentieth review on 2026-10-28: `SetupLockInTable()` finds or creates one `LOCK` per object, which exists while any session holds or awaits it, and one `PROCLOCK` per object and backend (`PROCLOCKTAG`). The lock-table row now says that only a lock on an object the dropping session has not locked yet needs new entries. Also rechecked object-address locking, catalog-scan and catalog-update paths, predicate-locking gates, rule, foreign-key, trigger and child dependency types, utility-hook dispatch and relcache physical-address setup; no other issue was found. Repeated the unchanged script from an empty sandbox and with the build present; both runs passed all 16 assertions.
- Shared glossary and matching-version pages about zeroed relation files, amcheck, last-segment checks, partitioned indexes, old-file disposal after VACUUM FULL, the post-drop health checklist and the empty-file failure scenarios. PostgreSQL 12 has no common-concept pages to link for these mechanisms.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| File emptiness does not satisfy `RESTRICT` | [heap.c#schema-edge](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408), [dependency.c#RESTRICT](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1113-L1203); measured `restrict.txt` |
| Cascade collects catalog dependencies before deletion | [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) |
| An exclusion constraint depends on its exclusion operators indirectly through its supporting index and operator class; the index is internally dependent on the constraint | [pg_constraint.c#indirect-exclusion-dependency](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L348-L354), [index.c#operator-class-dependency](../../../../raw/postgres-12/src/backend/catalog/index.c#L1134-L1143), [index.c#index-constraint-dependency](../../../../raw/postgres-12/src/backend/catalog/index.c#L1888-L1896), [dependency.c#redirect-to-owner](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L682-L737); not measured |
| The schema is locked during lookup, before the ownership check | [objectaddress.c#lock-after-lookup](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1076-L1088), [dropcmds.c#ownership-check](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L104-L109) |
| The ownership check passes a superuser and any role with the owner's privileges, including inheriting members of the owning role | [aclchk.c#pg_namespace_ownercheck](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L4927-L4951), [acl.c#has_privs_of_role](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4907-L4922), [acl.c#roles_has_privs_of](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4732-L4817); measured `owner_member.txt`, `owner_other.txt` |
| Ownership is checked only for the named schema; dependents, including other roles' objects outside the schema, are removed without a core ownership or privilege check | [dropcmds.c#ownership-check](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L104-L109), [dependency.c#deleteOneObject](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1233-L1331), [dependency.c#doDeletion](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1341-L1369), [pg_constraint.c#RemoveConstraintById](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L552-L631), [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1496-L1568), [drop_schema.sgml#ownership](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L35-L39); not measured |
| Every dependent is locked during the walk, one at a time as the scan reaches it, before the `RESTRICT` check; while the drop waits, sessions holding no lock on the awaited object queue behind it, and dependents the walk has not reached are not locked yet | [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901), [dependency.c#recurse-after-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L960-L988), [dependency.c#report-after-walk](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L401-L429), [lock.c#join-wait-queue](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L976-L985), [lmgr/README#lock-acquisition-rules](../../../../raw/postgres-12/src/backend/storage/lmgr/README#L362-L381); measured `locks_d_held.txt`, `locks_c0.txt`, `locks_c1.txt`, `locks_c2.txt` |
| A session that already holds a lock on the object is granted at once, waits ahead of the drop, or gets a deadlock error | [lock.c#already-held-mode](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L828-L840), [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L99-L103), [proc.c#existing-holder-queue-position](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1100-L1204); measured `locks_a.txt`, under a 500 ms bound below the 1 s `deadlock_timeout` and after the drop's statement had run for 2 s; a queued request could have been granted sooner only by a deadlock check's queue reordering, and A's own check would run only after `deadlock_timeout` ([proc.c#deadlock-check-after-timeout](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1289-L1297), [deadlock.c#DeadLockCheck-reorder](../../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L217-L285), [guc.c#deadlock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2062-L2072), [lmgr/README#deadlock-detector-reorders](../../../../raw/postgres-12/src/backend/storage/lmgr/README#L370-L377)) |
| The walk holds the lock on every object it found until the transaction ends; these `AccessExclusiveLock` relation locks and object locks cannot use fast-path slots, so each needs shared lock-table entries; the table's size formula is an estimate, not a hard cap, and a request whose entries cannot be allocated from shared memory fails with `out of shared memory` (`53200`) under `RESTRICT` and `CASCADE` alike | [dependency.c#lock-before-recursing](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L883-L901), [lock.c#fast-path-eligibility](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L199-L204), [lock.c#NLOCKENTS](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L56-L57), [shmem.c#max_size-not-hard-limit](../../../../raw/postgres-12/src/backend/storage/ipc/shmem.c#L303-L310), [lock.c#LockShmemSize-margin](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L3441-L3460), [lock.c#SetupLockInTable](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L1119-L1197), [postinit.c#MaxBackends](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L532-L534), [guc.c#max_locks_per_transaction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2464-L2473), [lock.c#out-of-shared-memory](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L956-L968), [errcodes.txt:400](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L400); not measured |
| `CASCADE` deletion locks surviving tables outside the schema, removes their dependent constraints, indexes, triggers, rules, defaults, policies and columns, and drops dependent outside relations | [pg_constraint.c#auto-dependency-on-table](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L233-L258), [pg_constraint.c#normal-dependency-on-referenced-columns](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L274-L365), [pg_constraint.c#RemoveConstraintById-lock](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L572-L580), [index.c#index-dependencies](../../../../raw/postgres-12/src/backend/catalog/index.c#L1120-L1163), [index.c#index-drop-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L2047-L2050), [rewriteDefine.c#InsertRule-dependencies](../../../../raw/postgres-12/src/backend/rewrite/rewriteDefine.c#L160-L184), [rewriteRemove.c#RemoveRewriteRuleById](../../../../raw/postgres-12/src/backend/rewrite/rewriteRemove.c#L37-L92), [tablecmds.c#createForeignKeyActionTriggers](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L9940-L10002), [trigger.c#function-dependency](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1006-L1017), [trigger.c#RemoveTriggerById](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1525-L1530), [heap.c#AddNewAttributeTuples](../../../../raw/postgres-12/src/backend/catalog/heap.c#L820-L838), [heap.c#default-expression-dependencies](../../../../raw/postgres-12/src/backend/catalog/heap.c#L2293-L2321), [heap.c#RemoveAttributeById](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1613-L1717), [tablecmds.c#StoreCatalogInheritance1](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L2776-L2798); measured `rollback.txt`, `cascade_survivors.txt` |
| `index_drop()` transfers predicate locks when serializable tracking exists and the index is eligible (its OID is not below `FirstBootstrapObjectId` and it is not temporary); a `SERIALIZABLE` drop flags `rw` conflicts in `CheckTableForSerializableConflictIn()`, raising `40001` immediately when SSI's structure, commit-order and read-only checks set `failure` for `MySerializableXact == writer`, or at pre-commit if later doomed or a qualifying prepared-pivot conflict is found | [index.c#TransferPredicateLocksToHeapRelation](../../../../raw/postgres-12/src/backend/catalog/index.c#L2195-L2197), [heap.c#serializable-conflict-check](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1902-L1908), [predicate.c#DropAllPredicateLocksFromTable](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L2893-L3100), [predicate.c#transfer-gate](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L2918-L2928), [predicate.c#PredicateLockingNeededForRelation](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L493-L503), [predicate.c#CheckTableForSerializableConflictIn](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4540-L4600), [predicate.c#ordering-and-failure-tests](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4652-L4808), [predicate.c#writer-failure](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4776-L4808), [predicate.c#PreCommit_CheckForSerializationFailure](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4828-L4908), [xact.c#PreCommit_CheckForSerializationFailure-call](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2128-L2130); not measured |
| Predicate locks do not block and can survive commit; the SSI prepared state also occurs during ordinary commit, without SQL `PREPARE TRANSACTION` | [predicate.c#SIREAD-lifetime](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L27-L59), [predicate.c#prepared-state](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L267-L278), [predicate_internals.h#commit-order](../../../../raw/postgres-12/src/include/storage/predicate_internals.h#L62-L77), [predicate.c#mark-prepared](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4904-L4907), [xact.c#ordinary-commit-check](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2121-L2129); not measured |
| `default_transaction_isolation` sets the level of each new transaction; `transaction_isolation` affects only the current transaction and cannot change after its first snapshot or inside a subtransaction | [guc.c#isolation-settings](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4254-L4273), [xact.c:1953](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1953), [variable.c#check_XactIsoLevel](../../../../raw/postgres-12/src/backend/commands/variable.c#L525-L554), [utility.c#BEGIN-options](../../../../raw/postgres-12/src/backend/tcop/utility.c#L416-L429); not measured |
| Ordinary deletion needs metadata, not target page contents | [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1998), [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266), [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L589-L629), [md.c#segment-gap](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1179-L1204); measured: no main-fork read error in `rollback.txt`, `cascade.txt` or `missing.txt` after a restart with the files emptied |
| Existing zero-length segments do not stop unlink traversal; a missing filename stops it silently | [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359); measured `files_s_after_commit.txt`, `files_s2_after_checkpoint.txt` |
| Abort and commit choose different cleanup entries | [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462); measured `files_s_after_rollback.txt`, `files_s_after_checkpoint.txt` |
| Catalog loss and installed code limit the conclusion | [dependency.c#dependency-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L829-L866), [event_trigger.out#rejected-schema-drop](../../../../raw/postgres-12/src/test/regress/expected/event_trigger.out#L277-L291) |
| Catalog outcomes depend on the buffer hit, segment-open state, settings, page reader and whether catalog rows are deleted or updated | [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L589-L629), [md.c#_mdfd_getseg](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L1112-L1204), [heapam_handler.c#heap-fetch](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L129-L156), [heapam.c#empty-page-search](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1550-L1557), [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L719), [indexing.c#CatalogTupleUpdate](../../../../raw/postgres-12/src/backend/catalog/indexing.c#L74-L272) |

## Open Questions

- The exact set of files emptied, remaining catalog integrity, installed triggers/hooks/access methods, and concurrent server activity are not supplied. This page establishes conditional engine behavior; it cannot certify the outcome on the affected database.
- The measurement covers emptied user-relation files with intact catalogs, at 1 MB segments, on one platform; see [Last run](#last-run). The damaged-catalog branches, the `SERIALIZABLE` failure branch, the `_mdfd_getseg()` open-segment branch, removal of dependents owned by other roles, lock-table exhaustion and default 1 GB segments are read from the source, not measured. No measured result disagreed with the source reading.
- That `cat /dev/null > file` truncates the file to zero length is shell behavior, which the pinned PostgreSQL checkout cannot establish, so the page assumes it. The measurement's `empty` stage observed it on its platform, where every file it emptied this way then had 0 bytes. For user files the assumption does not change the deletion outcome, because the deletion path never reads their contents and removes storage by filename. The short-file branches in [Catalog damage branches](#catalog-damage-branches) do assume truncation, because their read errors come from short files. [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1998) [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359)
- The `DROP SCHEMA` reference page says that a schema can only be dropped by its owner or a superuser, and the privileges section of the data-definition chapter says that the right to destroy an object is always the privilege of the owner only. The implementation also lets any role with the owner's privileges through, which includes inheriting members of the owning role, and the `owner` leg measured such a member dropping `s_own`. Source wins, so the page follows it. The docs may mean role-based ownership, because the role-membership section says that inheriting members can use the privileges of the roles they belong to, but neither passage says so. [drop_schema.sgml#ownership](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L35-L39) [ddl.sgml#owner-only](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L1580-L1582) [aclchk.c#pg_namespace_ownercheck](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L4927-L4951) [acl.c#has_privs_of_role](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4907-L4922) [user-manag.sgml#inherited-privileges](../../../../raw/postgres-12/doc/src/sgml/user-manag.sgml#L320-L329)
- The `max_locks_per_transaction` documentation and the GUC's own help text disagree with the lock-manager source in two ways. They size the shared lock table from `max_connections` (the documentation adds `max_prepared_transactions`; the help text does not), while `NLOCKENTS()` uses `MaxBackends + max_prepared_xacts`. The documentation also says that no more than that many distinct objects can be locked at any one time, while `ShmemInitHash()` says that the maximum is not a hard limit and `LockShmemSize()` calls the count only an estimate. Source wins, so the page follows it. [config.sgml#max_locks_per_transaction](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L8622-L8630) [guc.c#max_locks_per_transaction](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2464-L2468) [lock.c#NLOCKENTS](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L56-L57) [shmem.c#max_size-not-hard-limit](../../../../raw/postgres-12/src/backend/storage/ipc/shmem.c#L303-L306) [lock.c#LockShmemSize-margin](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L3454-L3457)
- PostgreSQL 12 lacks common-concept pages for schema dependency deletion, the relation-file deletion lifecycle and the SSI read tracking used here. Those would be useful separate pages; none was created as a side effect of this answer.

## Source References

- [gram.y#opt_drop_behavior](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568), [gram.y#DropStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6184-L6203), [gram.y#SCHEMA](../../../../raw/postgres-12/src/backend/parser/gram.y#L6304-L6313), [gram.y#TransactionStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L9826-L9939) — `DROP SCHEMA` grammar, default `DROP_RESTRICT` behavior and `BEGIN` transaction modes.
- [utility.c#ProcessUtility](../../../../raw/postgres-12/src/backend/tcop/utility.c#L76-L1751), [event_trigger.c#EventTriggerDDLCommandStart](../../../../raw/postgres-12/src/backend/commands/event_trigger.c#L809-L1166) — utility hook, dispatch to `ProcessUtilitySlow` and event-trigger support.
- [dropcmds.c#RemoveObjects](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L49-L127), [objectaddress.c#get_object_address](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L341-L2504), [namespace.c#get_namespace_oid](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3030-L3047) — command-level resolution, locking and authorization.
- [aclchk.c#aclcheck_error](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L3576-L4951), [acl.c#roles_has_privs_of](../../../../raw/postgres-12/src/backend/utils/adt/acl.c#L4732-L4945) — schema ownership through inherited role membership, the `SET ROLE` membership test and error reporting.
- [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L368-L1554) — discovery, locking, checks and deletion.
- [lock.c#already-held-mode](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L828-L840), [lock.c#join-wait-queue](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L976-L985), [proc.c#existing-holder-queue-position](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1100-L1204), [lmgr/README#lock-acquisition-rules](../../../../raw/postgres-12/src/backend/storage/lmgr/README#L362-L381), [lock.c#lock-table-size-and-overflow](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L56-L968), [proc.c#deadlock-check-after-timeout](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1248-L1297), [deadlock.c#DeadLockCheck](../../../../raw/postgres-12/src/backend/storage/lmgr/deadlock.c#L217-L285), [postinit.c#MaxBackends](../../../../raw/postgres-12/src/backend/utils/init/postinit.c#L532-L534) — ordinary queuing, immediate grant and conditional queue insertion, shared lock-table sizing and overflow, and the deadlock check that can reorder a queue.
- [pg_constraint.c#CreateConstraintEntry-dependencies](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L233-L365), [pg_constraint.c#RemoveConstraintById-lock](../../../../raw/postgres-12/src/backend/catalog/pg_constraint.c#L572-L580), [tablecmds.c#child_dependency_type](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L296-L10002), [trigger.c#CreateTrigger-dependencies](../../../../raw/postgres-12/src/backend/commands/trigger.c#L1006-L1539), [rewriteDefine.c#InsertRule-dependencies](../../../../raw/postgres-12/src/backend/rewrite/rewriteDefine.c#L160-L184), [rewriteRemove.c#RemoveRewriteRuleById](../../../../raw/postgres-12/src/backend/rewrite/rewriteRemove.c#L37-L92), [pg_proc.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/pg_proc.c#L616-L620), [heap.c#AddNewAttributeTuples](../../../../raw/postgres-12/src/backend/catalog/heap.c#L820-L2321), [policy.c#RemovePolicyById-lock](../../../../raw/postgres-12/src/backend/commands/policy.c#L385-L853) — cross-schema dependencies, foreign-key trigger ownership and removal locks on tables outside the schema.
- [predicate.c#SIREAD-lifetime](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L27-L59), [predicate.c#prepared-state](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L267-L278), [predicate.c#PredicateLockingNeededForRelation](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L497-L572), [predicate.c#DropAllPredicateLocksFromTable](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L2893-L3100), [predicate.c#CheckTableForSerializableConflictIn](../../../../raw/postgres-12/src/backend/storage/lmgr/predicate.c#L4498-L4908), [predicate_internals.h#SERIALIZABLEXACT](../../../../raw/postgres-12/src/include/storage/predicate_internals.h#L57-L129), [README-SSI#rw-conflicts-and-dangerous-structures](../../../../raw/postgres-12/src/backend/storage/lmgr/README-SSI#L133-L183) — read tracking, conflict patterns, transaction states and the `SERIALIZABLE` drop branch.
- [relation.c#relation_open](../../../../raw/postgres-12/src/backend/access/common/relation.c#L47-L78), [relcache.c#RelationInitPhysicalAddr](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1055-L1428), [relpath.c#GetRelationPath](../../../../raw/postgres-12/src/common/relpath.c#L129-L205), [toasting.c#toast-dependency](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L228-L376) — descriptor construction, physical paths and TOAST dependencies.
- [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1844-L1998), [index.c#index_create-and-index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L1120-L2266), [indexing.c#CatalogTupleUpdate](../../../../raw/postgres-12/src/backend/catalog/indexing.c#L74-L272), [functioncmds.c#RemoveFunctionById](../../../../raw/postgres-12/src/backend/commands/functioncmds.c#L1180-L1206), [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L214-L235) — per-object cleanup and catalog-index updates.
- [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L462), [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495), [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L217-L359), [sync.c#checkpoint-cycle](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214), [xact.c#commit-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1359-L5838) — pending deletion, transaction commit/abort/redo callers, filename operations and delayed unlink lifecycle.
- [genam.c#systable_beginscan](../../../../raw/postgres-12/src/backend/access/index/genam.c#L352-L456), [indexam.c#index_fetch_heap](../../../../raw/postgres-12/src/backend/access/index/indexam.c#L566-L636), [heapam_handler.c#heapam_index_fetch_tuple](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L117-L156), [heapam.c#heap_hot_search_buffer](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L216-L1655), [bufmgr.c#ReadBuffer_common](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L2778), [bufpage.h#PageGetMaxOffsetNumber](../../../../raw/postgres-12/src/include/storage/bufpage.h#L348-L360), [bufpage.c#all-zero-acceptance](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L130-L142), [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L689-L769), [md.c#mdread](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L589-L1204) — catalog reads, buffer hits, short reads, zero-fill and B-tree page checks.
- [guc.h#GucContext](../../../../raw/postgres-12/src/include/utils/guc.h#L54-L64), [guc.c#guc-tables](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L4273), [variable.c#check_XactIsoLevel-and-check_role](../../../../raw/postgres-12/src/backend/commands/variable.c#L525-L866), [ps_status.c#init_ps_display](../../../../raw/postgres-12/src/backend/utils/misc/ps_status.c#L265-L321), [proc.c#lock-timeout-timer](../../../../raw/postgres-12/src/backend/storage/lmgr/proc.c#L1242-L1264), [postgres.c#lock-timeout-error](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L3082-L3087), [errcodes.txt#sqlstates](../../../../raw/postgres-12/src/backend/utils/errcodes.txt#L271-L418), [configure.in#segsize](../../../../raw/postgres-12/configure.in#L279-L295), [pg_config.h.in#RELSEG_SIZE](../../../../raw/postgres-12/src/include/pg_config.h.in#L853-L864) — GUC definitions, isolation-level and role checks, process-title setup, lock-timeout error path, SQLSTATEs and segment-size build configuration.
- [catalog/Makefile#catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L103), [pg_namespace.h#FormData_pg_namespace](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L21-L44), [pg_depend.h#FormData_pg_depend](../../../../raw/postgres-12/src/include/catalog/pg_depend.h#L33-L66), [objectaddress.h#ObjectAddress](../../../../raw/postgres-12/src/include/catalog/objectaddress.h#L21-L29) — catalog declarations and generated headers.
- [installation.sgml#GNU-make](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L80-L83), [installation.sgml#Perl-for-Git-builds](../../../../raw/postgres-12/doc/src/sgml/installation.sgml#L309-L318), [catalog/Makefile#genbki](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L90) — build prerequisites and Perl-driven catalog generation.
- [drop_schema.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L22-L95), [ddl.sgml#owner-only](../../../../raw/postgres-12/doc/src/sgml/ddl.sgml#L1580-L1582), [user-manag.sgml#inherited-privileges](../../../../raw/postgres-12/doc/src/sgml/user-manag.sgml#L320-L329), [monitoring.sgml#query_start](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L675-L679), [storage.sgml#forks-and-segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L199-L243), [namespace.sql#schema-test](../../../../raw/postgres-12/src/test/regress/sql/namespace.sql#L5-L44), [namespace.out#cascade-notice-and-count](../../../../raw/postgres-12/src/test/regress/expected/namespace.out#L60-L70), [drop_if_exists.sql#schema-test](../../../../raw/postgres-12/src/test/regress/sql/drop_if_exists.sql#L51-L59), [event_trigger.sql#drop-rejection-test](../../../../raw/postgres-12/src/test/regress/sql/event_trigger.sql#L207-L261), [event_trigger.out#rejected-schema-drop](../../../../raw/postgres-12/src/test/regress/expected/event_trigger.out#L277-L308) — matching-checkout documentation and regression tests.

## Navigation

- [v12/index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [A Manually Zeroed Relation File in PostgreSQL 12 (unverified)](zeroed-relation-file.md)
- [Can amcheck Detect Zeroed Relation Segments in PostgreSQL 12? (unverified)](../indexing/amcheck-zeroed-relation-segments.md)
- [Checking Every Table, Catalogs Included, for a Zeroed Last Segment (unverified)](zeroed-last-segment-check-all-tables.md)
- [Failure Scenarios That Can Leave Schema Relation Files Empty in PostgreSQL 12 (unverified)](schema-empty-relation-files-failure-scenarios.md)
- [Database Health After Schema Files Were Zeroed and the Schema Dropped in PostgreSQL 12 (unverified)](../server-administration/health-after-zeroed-schema-drop.md)
