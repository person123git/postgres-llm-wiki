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
  - [Commit removes storage by filename](#commit-removes-storage-by-filename)
  - [State changes and rollback](#state-changes-and-rollback)
  - [Cases that change the result](#cases-that-change-the-result)
  - [Build boundaries and existing tests](#build-boundaries-and-existing-tests)
  - [Causal summary](#causal-summary)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Follow AGENTS.md. In PostgreSQL 12, how does DROP SCHEMA work, and what happens if DROP SCHEMA is executed on a schema that has had all its objects zeroed using `cat /dev/null > file.txt`?

## Answer

### The outcome depends on CASCADE and intact catalogs

**Emptying the files does not make the schema empty.** Plain `DROP SCHEMA s` defaults to `RESTRICT`. If ordinary tables still have their [catalog](../../../glossary.md#catalog) entries and [dependencies](../../../glossary.md#dependency) on the schema, PostgreSQL refuses the drop even when their files contain no bytes. The decision follows dependency records, not row counts or file lengths. [gram.y#opt_drop_behavior](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568) [heap.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408) [dependency.c#reportDependentObjects](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1093-L1203)

**With explicit `CASCADE`, empty user-relation files alone do not prevent the ordinary drop path from succeeding.** With usable catalogs, required locks and permissions, ordinary built-in objects, and no failing added trigger or hook, PostgreSQL deletes the metadata and schedules the files for removal without scanning the affected tables or validating their index pages. This is a source-derived conclusion for the pinned PostgreSQL **12.2** checkout, not a runtime reproduction. It is not a guarantee for a database whose catalogs were also damaged. [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1998) [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173)

### What was emptied, and what DROP still needs

The shell command in the question truncates the named file to zero length; it does not overwrite every original byte with a zero while retaining the length. Here, `file.txt` means an actual relation-file pathname. A [schema](../../../glossary.md#schema) is a namespace recorded in `pg_namespace`, not a directory that contains all its objects. Relation paths use database, tablespace and physical relation identifiers, not the schema name. [pg_namespace.h#FormData_pg_namespace](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L24-L44) [relpath.c#GetRelationPath](../../../../raw/postgres-12/src/common/relpath.c#L129-L205)

“All its objects” therefore needs a boundary. Tables, indexes and sequences can have physical relation storage; views and functions do not each have a corresponding data file to empty. A table can also have multiple [forks](../../../glossary.md#fork) and numbered [segments](../../../glossary.md#storage-manager). Emptying one pathname changes that file, not all those related files or the catalog rows describing the object. The answer assumes the user-object files were emptied and the catalogs were left usable. [heap.c#storage-bearing-kinds](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1943-L1952) [dependency.c#object-dispatch](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1341-L1378) [functioncmds.c#RemoveFunctionById](../../../../raw/postgres-12/src/backend/commands/functioncmds.c#L1180-L1206) [storage.sgml#forks-and-segments](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L199-L243)

| Information or state | Origin and freshness | How deletion uses it |
|---|---|---|
| Schema identity and owner | Stored `pg_namespace` row, accessed through catalog lookup | Resolves the name and checks ownership. It is not inferred from disk files. [namespace.c#get_namespace_oid](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3030-L3047) [aclchk.c#pg_namespace_ownercheck](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L4927-L4951) |
| Object addresses and dependency edges | `ObjectAddress` identifies a catalog, object and possible subobject; stored `pg_depend` rows connect them | Builds and orders the objects to delete; determines whether `RESTRICT` blocks deletion. [objectaddress.h#ObjectAddress](../../../../raw/postgres-12/src/include/catalog/objectaddress.h#L21-L29) [pg_depend.h#FormData_pg_depend](../../../../raw/postgres-12/src/include/catalog/pg_depend.h#L36-L66) [dependency.c#dependency-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L829-L866) |
| Relation descriptor and physical identity | Catalog-derived [relcache](../../../glossary.md#relcache) descriptor; `rd_node` supplies the physical identifier | `RelationDropStorage` copies the identifier into a pending-delete entry. An external file truncation does not edit this catalog mapping. [relcache.c#RelationBuildDesc](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1055-L1084) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L148-L172) |
| File existence and length | Current filesystem state; independently changed by the shell command | Commit-time cleanup opens/truncates or unlinks paths. It does not ask how many table rows survived. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) |
| Pending deletion and transaction outcome | In-memory `PendingRelDelete`, including `atCommit` and nesting level | Distinguishes commit-time deletion of existing storage from abort-time cleanup of newly created storage. [storage.c#PendingRelDelete](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462) |

### Decision map

```text
DROP SCHEMA s [CASCADE | RESTRICT]
  |
  +-- Resolve schema, check owner, acquire deletion locks [A]
  |     +-- missing / denied / cannot acquire locks -> notice, error or wait
  |
  +-- Read pg_depend and collect dependent objects [B]
  |     +-- required catalog lookup/read fails -> error
  |     +-- dependency records were lost -> list cannot be trusted as complete
  |
  +-- Default RESTRICT + blocking dependencies -> error before deletion [B]
  |
  +-- Explicit CASCADE, or no blocking dependencies [B]
        |
        +-- Delete dependent objects, then schema metadata [C]
        |     +-- existing user-file contents are not scanned for deletion
        |     +-- schedule storage removal using physical relation identifiers
        |
        +-- Transaction aborts -> do not execute these commit-only deletes [D]
        |                         external file truncation is not undone
        |
        +-- Transaction commits -> discard buffers and clean up paths [D]
              +-- first non-temp main segment: truncate, defer unlink [E]
              +-- other forks and numbered segments: unlink [E]
              +-- zero length does not stop numbered-segment traversal
              +-- missing path / filesystem failure can limit cleanup
```

- **A:** [dropcmds.c#RemoveObjects](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L49-L123), [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554).
- **B:** [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437), [dependency.c#dependency-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L829-L866), [dependency.c#RESTRICT-error](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1113-L1203). Lost-edge incompleteness is a deduction from this catalog-driven enumeration, not a measured damaged-catalog outcome.
- **C:** [dependency.c#deletion-order](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L974-L1010), [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1998), [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266), [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L214-L235).
- **D:** [storage.c#pending-delete-lifecycle](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65), [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462), [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495).
- **E:** [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359), [sync.c#SyncPostCheckpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214).

### From the command to the deletion list

1. **Parse the requested behavior.** The grammar makes a `DropStmt` with the names, object kind, `missing_ok` and `behavior`. The omitted behavior is `DROP_RESTRICT`; writing `CASCADE` selects `DROP_CASCADE`. Schema deletion takes `ProcessUtilitySlow` → `ExecDropStmt` → `RemoveObjects`. [gram.y#DropStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6184-L6203) [gram.y#SCHEMA](../../../../raw/postgres-12/src/backend/parser/gram.y#L6304-L6313) [gram.y#opt_drop_behavior](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568) [utility.c#drop-dispatch](../../../../raw/postgres-12/src/backend/tcop/utility.c#L1590-L1594) [utility.c#ExecDropStmt](../../../../raw/postgres-12/src/backend/tcop/utility.c#L1731-L1751)
2. **Resolve and authorize the schema.** `get_object_address()` maps it to `NamespaceRelationId` and its schema [OID](../../../glossary.md#oid). Ownership is checked for that schema; a superuser bypasses the ownership check. The schema owner can cascade to contained objects owned by others. `IF EXISTS` changes missing-schema handling to a notice; it does not ignore dependencies or general failures. [objectaddress.c#schema-address](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1186-L1190) [objectaddress.c#schema-ownership](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2337-L2341) [aclchk.c#pg_namespace_ownercheck](../../../../raw/postgres-12/src/backend/catalog/aclchk.c#L4927-L4951) [dropcmds.c#RemoveObjects](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L49-L123) [drop_schema.sgml#ownership](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L35-L39)
3. **Walk dependencies before deleting.** `performMultipleDeletions()` opens `pg_depend`; `findDependentObjects()` recursively follows references and puts dependents before their referenced object. The ordinary table-to-schema edge is `DEPENDENCY_NORMAL`, so a table with no data still blocks default `RESTRICT`. Automatic/internal/partition/extension dependency flags can allow dependent cleanup even under `RESTRICT`; this does not make an ordinary table's schema edge automatic. [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) [dependency.c#deletion-order](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L974-L1010) [dependency.c#dependency-flags](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1113-L1155) [heap.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408)

The deletion set is not bounded by the schema name. `CASCADE` follows dependent objects into other schemas. Internal and extension ownership can also redirect traversal toward an owning object; `CASCADE` is not a filesystem wildcard limited to the files that were emptied. [dependency.c#dependency-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L829-L866) [dependency.c#extension-and-internal-dependencies](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L594-L734) [drop_schema.sgml#cross-schema-cascade](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L89-L95)

A regular table's [TOAST](../../../glossary.md#toast) table, when present, illustrates why physical cleanup is not limited to objects named in the user schema. It lives in `pg_toast` and has an internal dependency on its owning table, so dependency deletion includes it too. Temporary tables use their temporary TOAST namespace instead. [toasting.c#toast-namespace](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L228-L235) [toasting.c#toast-dependency](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L362-L376)

### Deleting objects does not scan their data

The generic deletion dispatcher selects `index_drop()` for an index and `heap_drop_with_catalog()` for a whole non-index relation. For a sequence it also deletes the `pg_sequence` row. These routines remove metadata and schedule storage deletion; they do not perform a row-by-row `DELETE`, scan the target heap, or run an index integrity check. Related objects such as indexes and constraints are handled by dependency traversal before their table. [dependency.c#doDeletion](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1341-L1369) [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1844-L1998) [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266)

“Open the relation” here is not “read its first data page.” `relation_open()` obtains the lock and a relcache descriptor. Building that descriptor reads catalog metadata and initializes access-method information. Thus a zero-byte user index can be dropped through the ordinary path without reading its [metapage](../../../glossary.md#metapage), while an unusable catalog index needed to build the descriptor can still make the operation fail. [relation.c#relation_open](../../../../raw/postgres-12/src/backend/access/common/relation.c#L47-L78) [relcache.c#RelationBuildDesc](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1055-L1084) [relcache.c#access-method-initialization](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1188-L1217) [index.c#index-drop-open](../../../../raw/postgres-12/src/backend/catalog/index.c#L2047-L2056)

Deletion takes [AccessExclusiveLock](../../../glossary.md#accessexclusivelock) on affected relations. Dropping an index also locks its owning table; dropping a partition can additionally lock its parent and the default partition. Empty files do not remove these lock requirements. The current session's own open uses are checked by `CheckTableNotInUse()`. [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) [index.c#index-drop-locks](../../../../raw/postgres-12/src/backend/catalog/index.c#L2029-L2056) [heap.c#drop-locks-and-in-use-check](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1861-L1908)

Finally, `RemoveSchemaById()` deletes the `pg_namespace` row. Generic deletion also removes outgoing dependency records, shared dependencies, comments, security labels and initial privileges. This is why usable catalogs remain essential even if every target data file is empty. [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L214-L235) [dependency.c#deleteOneObject](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1269-L1326)

### Commit removes storage by filename

`RelationDropStorage()` stores the relation's physical identifier with `atCommit = true`; it does not immediately remove the file. At commit, `smgrDoPendingDeletes(true)` passes the accumulated relations to `smgrdounlinkall()`. The latter closes storage handles, discards their remaining [buffers](../../../glossary.md#buffer-manager) without writing them, sends storage invalidations and calls the file deletion routine for each fork. [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462) [xact.c#commit-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232) [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495)

For ordinary non-temporary relations outside recovery, `mdunlinkfork()` truncates the first main-fork segment to zero and queues its later unlink. It immediately unlinks the first file of other forks and then tries numbered segments in order. The delayed main file reserves the physical identifier until a suitable [checkpoint](../../../glossary.md#checkpoint) completes; this avoids premature reuse. A request arriving during a checkpoint waits for a later checkpoint cycle. Temporary relations and recovery use the immediate-unlink branch instead. [md.c#deferred-unlink-reason](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L217-L261) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) [sync.c#checkpoint-cycle](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214)

**An existing zero-byte segment does not stop this cleanup.** Consider existing paths `12345`, `12345.1` and `12345.2`, all emptied by the shell command. The main-file truncation succeeds, unlinking `.1` succeeds regardless of its length, and the loop then attempts `.2`. This source-derived example follows filename operations; there is no `mdnblocks()` call that would stop at `.1` because it is short. By contrast, the size-probing path used by scans returns at a short segment it measures. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L300-L359) [md.c#mdnblocks](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L715-L755)

**Missing filenames differ from empty files.** If `.1` was removed rather than emptied, its unlink returns `ENOENT` and this traversal does not reach `.2`. If the first segment operation fails, the additional-segment loop is not entered at all. Other filesystem failures produce warnings and can also leave files behind. Therefore SQL-level success does not prove that every damaged or orphaned pathname was removed. A zero-byte first main-fork file remaining before the eligible checkpoint, however, is expected cleanup state. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L284-L359) [smgr.c#unlink-warning-contract](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L481-L495)

### State changes and rollback

The lifecycle below concerns objects that existed before the dropping transaction. The source separately handles objects created and dropped within the same transaction by keeping both commit-time and abort-time cleanup entries. [storage.c#pending-delete-lifecycle](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [storage.c#create-and-drop-in-one-transaction](../../../../raw/postgres-12/src/backend/catalog/storage.c#L162-L169)

| Event | State before → after | Consequence |
|---|---|---|
| External truncation | User file had contents → pathname exists with zero length; no SQL catalog deletion was requested | The dependency list still names the objects. No pending-delete entry saves the original bytes. [heap.c#namespace-dependency](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408) [storage.c#PendingRelDelete](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) |
| Successful drop statement in an open transaction | Metadata present → transactional catalog deletions plus commit-only file cleanup entries | Finishing the statement is not yet physical storage cleanup. [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L218-L234) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) |
| Rollback or error aborts that transaction | Uncommitted catalog deletion → aborted; commit-only deletion entries are discarded | The objects remain, but PostgreSQL does not undo the external file truncation or restore the lost contents. [xact.c#abort-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2708-L2720) [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462) |
| Commit | Catalog deletions committed → pending storage cleanup executes | Buffers are discarded, forks/segments are removed as described above, and ordinary first main files await later unlink. [xact.c#commit-storage-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232) [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) |
| Eligible checkpoint completes | Queued first main file → unlink attempted | Physical filename cleanup catches up with the committed deletion. [sync.c#SyncPostCheckpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214) |

The transaction machinery also protects the ordering between [WAL](../../../glossary.md#wal) and non-temporary file deletion: pending relation deletions force a WAL flush before the files can disappear. Commit replay drops the recorded relation files. These mechanisms describe PostgreSQL's own deletion; they are not an undo mechanism for the preceding shell operation. [xact.c#flush-before-delete](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1359-L1377) [xact.c#xact_redo_commit](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L5817-L5838)

### Cases that change the result

| Condition / state | Default or ordinary path, and what changes it | Consequence |
|---|---|---|
| Schema still has ordinary objects; no `CASCADE` | Default `RESTRICT` checks normal dependencies, not contents | Refusal even with empty data files. [gram.y#default](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568) [dependency.c#RESTRICT-error](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1133-L1203) |
| Explicit `CASCADE`; only user files emptied | The command option allows normal dependent objects to be deleted; the storage-removal path does not scan their contents | Empty heap, index, sequence, auxiliary fork or segment files alone do not invalidate this path. No repair setting is required by it. [dependency.c#doDeletion](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1341-L1369) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) |
| Catalogs or their indexes were emptied too | Normally name resolution, dependency enumeration and metadata deletion read intact catalogs. Damage changes that premise | A missing schema lookup or relation-cache error is possible; missing dependency rows can yield an incomplete deletion set. No single guaranteed result follows from “catalog files zeroed.” [namespace.c#get_namespace_oid](../../../../raw/postgres-12/src/backend/catalog/namespace.c#L3030-L3047) [dependency.c#dependency-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L829-L866) [heap.c#catalog-lookup-failure](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1871-L1873) |
| Added [event trigger](../../../glossary.md#event-trigger) or C [hook](../../../glossary.md#hook) | Core dispatch normally runs without an installed utility hook; an empty event-trigger run list returns. Installed code is invoked when present | Extra SQL or an exception can change the outcome before commit. The pinned tests explicitly exercise a trigger rejecting `DROP SCHEMA ... CASCADE`. [utility.c:76](../../../../raw/postgres-12/src/backend/tcop/utility.c#L76) [utility.c#ProcessUtility-hook](../../../../raw/postgres-12/src/backend/tcop/utility.c#L355-L362) [event_trigger.c#EventTriggerDDLCommandStart](../../../../raw/postgres-12/src/backend/commands/event_trigger.c#L809-L817) [dependency.c#object-drop-hook](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1240-L1242) [event_trigger.out#rejected-schema-drop](../../../../raw/postgres-12/src/test/regress/expected/event_trigger.out#L277-L291) |
| Concurrent sessions or current-session object use | Deletion still needs exclusive locks and in-use checks, regardless of file length | It can wait or fail before file cleanup. [dependency.c#AcquireDeletionLock](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1533-L1554) [heap.c#CheckTableNotInUse](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1890-L1908) |
| Files truncated while the server remains active | External changes do not execute PostgreSQL's buffer invalidation and discard path. A previously cached page can remain available; a dirty buffer can later be written | Do not interpret the external command as a coordinated clearing of all server state. The commit-time drop explicitly discards remaining buffers instead. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [bufmgr.c#FlushBuffer](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L2740-L2778) [smgr.c#discard-buffers](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L459-L479) |
| Missing segment or filesystem failure | Normally consecutive filenames can be removed; a missing name or failed unlink interrupts enumeration | Warning or silent `ENOENT` handling can leave later files behind; `CASCADE` does not search the directory for orphans. [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L284-L359) |

### Build boundaries and existing tests

This is core backend behavior. The catalog build lists `pg_namespace.h` and `pg_depend.h`, generates their `_d.h` headers with the other catalogs, and installs links into the build include directory. Their declarations supply the catalog IDs and row layouts used above. No contrib module or special build option is required for ordinary schema deletion. [catalog/Makefile#catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L51) [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L101) [pg_namespace.h#catalog-declaration](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L21-L44) [pg_depend.h#catalog-declaration](../../../../raw/postgres-12/src/include/catalog/pg_depend.h#L33-L66)

Relation opening does initialize [access-method](../../../glossary.md#access-method) information, so the ordinary built-in-object conclusion does not certify arbitrary third-party access methods or their external storage. Hooks and extension dependencies are additional boundaries, described above. [relcache.c#access-method-initialization](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1188-L1217) [relcache.c#index-AM-initialization](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L1403-L1428) [dependency.c#extension-dependency](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L594-L621)

The checked regression tests cover schema creation/rename followed by cascade deletion and catalog verification, missing-schema handling, and event-trigger rejection/recursive DDL. These cited test cases do not inject zero-length relation files. No server was built or run for this page, and no measurement results are reported. [namespace.sql#schema-test](../../../../raw/postgres-12/src/test/regress/sql/namespace.sql#L5-L44) [drop_if_exists.sql#schema-test](../../../../raw/postgres-12/src/test/regress/sql/drop_if_exists.sql#L51-L59) [event_trigger.sql#drop-rejection-test](../../../../raw/postgres-12/src/test/regress/sql/event_trigger.sql#L207-L261)

### Causal summary

**User files emptied while catalogs remain usable → schema dependencies remain → default `RESTRICT` refuses when ordinary objects still depend on the schema; explicit `CASCADE` permits dependency-ordered metadata deletion → commit discards buffers and removes files without scanning their contents → eligible checkpoint finishes deferred first-main-file unlink.** Catalog damage, missing filenames, lock/permission failures and added code have the separate outcomes above. Rolling back the SQL drop does not reverse the external truncation. [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) [storage.c#pending-deletion](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495) [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) [sync.c#SyncPostCheckpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214)

## Context Reviewed

- PostgreSQL 12 pin `45b88269a353ad93744772791feb6d01bc7e1e42`; manifests, indexes and recent log entries used for navigation only.
- Grammar and utility dispatch; schema address/ownership; dependency graph, ordering, locks and extension/internal dependencies; relation/index/sequence/schema dispatch and catalog cleanup.
- Relation opening and catalog-derived descriptors; pending storage deletions; buffer discard; all-fork unlink, segment traversal, checkpoint queues; commit/abort callers and commit replay.
- Catalog and object-address structures, generated headers, reverse header inclusion and catalog build; ordinary regression and event-trigger tests.
- Shared glossary and matching-version pages about zeroed relation files, amcheck, last-segment checks, partitioned indexes, and old-file disposal after VACUUM FULL. PostgreSQL 12 has no common-concept pages to link for these mechanisms.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| File emptiness does not satisfy `RESTRICT` | [heap.c#schema-edge](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1388-L1408), [dependency.c#RESTRICT](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L1113-L1203) |
| Cascade collects catalog dependencies before deletion | [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) |
| Ordinary deletion needs metadata, not target page contents | [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1998), [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L2193-L2266) |
| Existing zero-length segments do not stop unlink traversal | [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) |
| Abort and commit choose different cleanup entries | [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462) |
| Catalog loss and installed code limit the conclusion | [dependency.c#dependency-scan](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L829-L866), [event_trigger.out#rejected-schema-drop](../../../../raw/postgres-12/src/test/regress/expected/event_trigger.out#L277-L291) |

## Open Questions

- The exact set of files emptied, remaining catalog integrity, installed triggers/hooks/access methods, and concurrent server activity are not supplied. This page establishes conditional engine behavior; it cannot certify the outcome on the affected database.
- No runtime reproduction of the exact all-files-emptied fixture was performed here. The zero-file conclusion follows from the pinned deletion path, and is distinct from the earlier pages' measurements of scans and integrity checks.
- PostgreSQL 12 lacks common-concept pages for schema dependency deletion and the relation-file deletion lifecycle. Those would be useful separate pages; none was created as a side effect of this answer.

## Source References

- [dropcmds.c#RemoveObjects](../../../../raw/postgres-12/src/backend/commands/dropcmds.c#L49-L123) — command-level resolution and authorization.
- [dependency.c#performMultipleDeletions](../../../../raw/postgres-12/src/backend/catalog/dependency.c#L374-L437) — discovery, checks and deletion.
- [heap.c#heap_drop_with_catalog](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1854-L1998), [index.c#index_drop](../../../../raw/postgres-12/src/backend/catalog/index.c#L1996-L2266), [schemacmds.c#RemoveSchemaById](../../../../raw/postgres-12/src/backend/commands/schemacmds.c#L214-L235) — per-object cleanup.
- [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173), [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495), [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359) — pending deletion to filename operations.
- [sync.c#SyncPostCheckpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214) — delayed unlink lifecycle.
- [drop_schema.sgml](../../../../raw/postgres-12/doc/src/sgml/ref/drop_schema.sgml#L22-L95) — matching-checkout command documentation.

## Navigation

- [v12/index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [A Manually Zeroed Relation File in PostgreSQL 12 (unverified)](zeroed-relation-file.md)
- [Can amcheck Detect Zeroed Relation Segments in PostgreSQL 12? (unverified)](../indexing/amcheck-zeroed-relation-segments.md)
- [Checking Every Table, Catalogs Included, for a Zeroed Last Segment (unverified)](zeroed-last-segment-check-all-tables.md)
