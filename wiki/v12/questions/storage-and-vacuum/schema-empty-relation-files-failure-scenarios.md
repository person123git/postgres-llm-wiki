---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Failure Scenarios That Can Leave Schema Relation Files Empty in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The short answer](#the-short-answer)
  - [First establish what is empty](#first-establish-what-is-empty)
  - [Decision map](#decision-map)
  - [Unlogged tables after crash recovery](#unlogged-tables-after-crash-recovery)
  - [A committed clear followed by a failed reload](#a-committed-clear-followed-by-a-failed-reload)
  - [Discarded storage waiting for cleanup](#discarded-storage-waiting-for-cleanup)
  - [Other operations and failure boundaries](#other-operations-and-failure-boundaries)
  - [Read-only inventory and incident evidence](#read-only-inventory-and-incident-evidence)
  - [Lifecycle and causal summary](#lifecycle-and-causal-summary)
  - [Build boundaries and tests](#build-boundaries-and-tests)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Follow AGENTS.md. For PostgreSQL 12, what database operation failure scenario could leave all object files empty for a schema?

## Answer

### The short answer

**The clearest built-in failure scenario is crash recovery of [unlogged relations](../../../glossary.md#unlogged-relation).** If every table in the affected [schema](../../../glossary.md#schema) was unlogged, recovery can reset all their heap contents, even previously committed contents. It selects relations by their initialization forks, not by schema. This does **not** make every associated file zero bytes: an unlogged [B-tree](../../../glossary.md#b-tree) is restored with an initialized metapage. [xlog.c#unlogged-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6878-L6884) [reinit.c#selection-and-reset](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L180-L308) [nbtree.c#btbuildempty](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L153-L183)

**A failed multi-transaction reload is another explanation for empty tables.** An earlier committed [`TRUNCATE`](../../../glossary.md#truncate), or committed creation of replacement tables, can leave an empty state that a later failed load does not undo. That differs from a `TRUNCATE` transaction itself aborting: for ordinary pre-existing tables, PostgreSQL preserves the old storage until commit and discards the new storage on abort. [tablecmds.c#transaction-safe-truncation](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1840) [storage.c#transactional-storage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462)

**Zero-byte filenames can also be normal cleanup artifacts rather than empty live objects.** A committed drop or storage replacement, and an orderly abort that discards newly created storage, can leave the first main-fork filenames at zero length until a suitable [checkpoint](../../../glossary.md#checkpoint). If instead the observation means *every current file of previously populated, logged tables and initialized indexes remains zero bytes*, the ordinary paths examined here do not establish a cause. External file modification, a storage/durability failure, added code, or an unidentified defect require incident evidence; the symptom alone cannot choose between them. [storage.c#delete-at-commit-or-abort](../../../../raw/postgres-12/src/backend/catalog/storage.c#L111-L119) [storage.c#RelationDropStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [md.c#deferred-unlink](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L217-L359) [sync.c#checkpoint-cycle](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214)

This is a source-only analysis of the repository's **PostgreSQL 12.2** pin, not a diagnosis of an inspected database or a claim that every later PostgreSQL 12 minor release is identical. No server, measurement, or destructive reproduction was run. [configure.in#version](../../../../raw/postgres-12/configure.in#L20)

### First establish what is empty

A schema is a namespace in the [catalogs](../../../glossary.md#catalog), **not a filesystem directory**. `pg_class.relnamespace` identifies membership, while physical paths use database, [tablespace](../../../glossary.md#tablespace), [relfilenode](../../../glossary.md#relfilenumber), backend and fork identifiers. A storage operation cannot infer a schema merely from a database directory. [pg_namespace.h#schema](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L24-L44) [pg_class.h#membership-and-storage](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L34-L78) [relpath.c#GetRelationPath](../../../../raw/postgres-12/src/common/relpath.c#L129-L205)

Not every schema object has a data file. The storage-bearing [relation](../../../glossary.md#relation) kinds include tables, indexes, sequences, materialized views and TOAST tables; views, foreign tables and partitioned parents have no storage of their own in this mapping. Ordinary [TOAST](../../../glossary.md#toast) tables live in `pg_toast`, outside their owning table's namespace. [dbsize.c#storage-bearing-kinds](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L877-L903) [toasting.c#namespace](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L228-L235)

| Observation or value | Meaning and origin | What it decides |
|---|---|---|
| A file's current length is zero | There are no bytes in that particular file. A relation can have several [forks](../../../glossary.md#fork), each split into [segments](../../../glossary.md#storage-manager). | Identify the fork and segment before calling the whole relation empty. Zero-length inactive segments are legitimate. [relfilenode.h#physical-address](../../../../raw/postgres-12/src/include/storage/relfilenode.h#L20-L38) [md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66) |
| A nonempty file contains [all-zero pages](../../../glossary.md#all-zero-page) | The length exists but the contents differ. Page validation explicitly recognizes the all-zero case. | This is not zero-length truncation; it needs a different investigation. [bufpage.c#all-zero-case](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142) |
| A query finds no rows | This is a query result, not a file-length measurement. For example, an aborted `COPY FROM` can leave invisible inserted rows occupying storage. | Do not infer that every heap or index file is zero bytes. [copy.c#tuple-insertion](../../../../raw/postgres-12/src/backend/commands/copy.c#L3292-L3304) [copy.sgml#error-and-disk-space](../../../../raw/postgres-12/doc/src/sgml/ref/copy.sgml#L532-L540) |
| Current object-to-file mapping | `pg_relation_filenode()` and `pg_relation_filepath()` resolve current catalog/mapper state, not a historical filename list. | Determine whether the observed file still belongs to the object. Rewrites and `TRUNCATE` can change the mapping. [dbsize.c#mapping-contract](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L850-L903) [dbsize.c#filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1021) [relcache.c#new-filenode](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3405-L3416) |
| `pg_class.relpersistence` | Stored relation property: `p` is permanent, `u` unlogged, `t` temporary. | Separates recovery's unlogged reset from the ordinary logged-table case. [pg_class.h#persistence](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L154-L167) |
| Commit boundary and `PendingRelDelete.atCommit` | Transaction state determines which old or newly created storage to remove. | A failed transaction, a failed later transaction, and a committed drop are different states. [storage.c#PendingRelDelete](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [storage.c#delete-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L410-L455) |

File state and server memory can disagree after an external change. A valid page in [shared buffers](../../../glossary.md#buffer-manager) can satisfy a read without consulting the file. For size discovery, `mdnblocks()` starts at the last already-open segment and trusts earlier open segments as full; with a fresh segment-open state, the first short segment ends discovery. Therefore neither an old filename list nor a successful read is a complete physical inventory. [bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795) [md.c#open-state-and-size](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L755)

### Decision map

This map assumes usable catalogs and distinguishes a completed state from an operation still in progress.

```text
What does "empty" mean?
|
+-- No rows, or zero contents but a nonzero file length?
|   `-- Do not equate that with zero-byte relation files. [A]
|
`-- File lengths really are zero
    |
    +-- Do the paths still map to the affected live objects?
    |   `-- No -> investigate dropped/old/aborted replacement storage;
    |            zero-byte first-main-file stubs can be normal. [B]
    |
    `-- Yes
        |
        +-- Unlogged heaps, after a startup that performed recovery?
        |   `-- INIT reset is an explanation for empty heaps.
        |       Initialized B-tree files are not all zero length. [C]
        |
        +-- Clearing or table creation committed before a failed load?
        |   `-- The earlier empty state survives the later failure. [D]
        |
        +-- Only inactive tail segments are zero?
        |   `-- Legitimate truncation can leave that layout. [E]
        |
        `-- All current, previously populated logged heaps and initialized
            indexes are zero, with none of those qualifications?
            `-- The inspected normal paths do not diagnose this.
                Preserve incident evidence; investigate beyond them. [F]
```

- **A:** [copy.sgml#failed-COPY](../../../../raw/postgres-12/doc/src/sgml/ref/copy.sgml#L532-L540), [bufpage.c#all-zero-pages](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142).
- **B:** [relcache.c#old-and-new-storage](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3445-L3479), [storage.c#delete-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462), [md.c#cleanup](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359).
- **C:** [xlog.c#recovery-condition](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6728-L6749), [reinit.c#INIT-copy](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L267-L308), [nbtree.c#init-metapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L153-L183).
- **D:** [tablecmds.c#truncate](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1840), [storage.c#transaction-lifetime](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L49), [pg_backup_archiver.c#restore-transaction](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L474-L480).
- **E:** [md.c#inactive-segments](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66), [md.c#mdtruncate](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L798-L837).
- **F:** The limits follow from the qualified paths above, not from proof that no PostgreSQL defect can exist. See [Open Questions](#open-questions).

### Unlogged tables after crash recovery

Ordinary `CREATE TABLE` defaults to permanent storage. Explicit `UNLOGGED` chooses a different persistence property, and `RelationNeedsWAL()` is false for it. This is why ordinary unlogged data changes do not have the same [WAL](../../../glossary.md#wal) recovery path as permanent table data. Initialization is a separate exception: the heap's init-fork creation and the B-tree's init metapage are WAL-logged. [gram.y#default-and-UNLOGGED](../../../../raw/postgres-12/src/backend/parser/gram.y#L3282-L3283) [rel.h#RelationNeedsWAL](../../../../raw/postgres-12/src/include/utils/rel.h#L515-L520) [heapam_handler.c#init-fork](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624) [nbtree.c#init-WAL](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L165-L183)

The reset follows this sequence:

1. **Startup decides whether recovery is needed.** An unclean control-file state, a checkpoint needing redo, or requested archive recovery sets `InRecovery`. An ordinary clean restart with no recovery request does not take that branch. [xlog.c#InRecovery](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6728-L6749)
2. **Cleanup identifies physical relations with `_init` files.** It visits database directories in the default and other tablespaces, collects those physical identifiers, and removes their non-init files. There is no schema-name filter. [xlog.c#cleanup-call](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6878-L6884) [reinit.c#directory-scope](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L68-L86) [reinit.c#database-directories](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L127-L139) [reinit.c#cleanup-passes](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L180-L258)
3. **After replay, initialization copies `_init` to the main fork.** The heap init fork is created without heap rows. A B-tree init fork contains its empty-tree [metapage](../../../glossary.md#metapage), so its reset result is an initialized empty index, not a zero-byte file. [xlog.c#init-call](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7323-L7331) [reinit.c#copy](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L267-L308) [heapam_handler.c#empty-heap-init](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624) [nbtree.c#btbuildempty](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L153-L183)

Indexes inherit their table's persistence; TOAST tables inherit it too. Thus a schema used entirely for unlogged staging tables can appear wholly emptied after recovery, while permanent relations elsewhere remain outside this reset. The selector is the init fork, not membership in the staging schema. A schema containing ordinary sequence objects is an important counterexample to “all objects were reset this way”: PostgreSQL 12 rejects unlogged sequences. [index.c#persistence-inheritance](../../../../raw/postgres-12/src/backend/catalog/index.c#L739-L749) [toasting.c#persistence](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L254-L268) [reinit.c#selection](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L188-L203) [sequence.c#unlogged-rejection](../../../../raw/postgres-12/src/backend/commands/sequence.c#L136-L140)

The reset also removes non-init auxiliary forks and extra segments instead of leaving every original pathname present at zero length. An incident in which all old `_fsm`, `_vm`, numbered segments and B-tree main files still exist at zero bytes therefore does not match the completed reset's normal layout. [reinit.c#remove-non-init-files](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L222-L258) [reinit.c#recreate-main-files](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L267-L308)

### A committed clear followed by a failed reload

`TRUNCATE` accepts a relation list, not a schema-wide erase option. Its ordinary dependency behavior is `RESTRICT`; explicit `CASCADE` follows referencing foreign keys and can add relations outside the named schema. A schema-wide effect can therefore come from a caller enumerating its tables, or from the dependency set, rather than a storage-level schema operation. [gram.y#TRUNCATE](../../../../raw/postgres-12/src/backend/parser/gram.y#L6348-L6362) [gram.y#default-RESTRICT](../../../../raw/postgres-12/src/backend/parser/gram.y#L2564-L2568) [tablecmds.c#explicit-targets](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1524-L1556) [tablecmds.c#FK-cascade](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1648-L1698)

For an ordinary pre-existing table, the utility dispatcher calls `ExecuteTruncate()`, which locks the targets and calls `ExecuteTruncateGuts()`. The latter assigns new empty storage to the heap and its TOAST table, then rebuilds their indexes. `RelationSetNewRelfilenode()` schedules old storage for commit-time deletion and updates the catalog mapping; creation registers the new file for deletion on abort. This keeps a statement error from turning the old committed table into an empty one. [utility.c#dispatch](../../../../raw/postgres-12/src/backend/tcop/utility.c#L543-L545) [tablecmds.c#locking](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1524-L1540) [tablecmds.c#new-storage-and-indexes](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1812-L1840) [relcache.c#storage-and-mapping](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3445-L3479) [relcache.c#catalog-update](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3525-L3559) [storage.c#abort-registration](../../../../raw/postgres-12/src/backend/catalog/storage.c#L105-L119)

There is an intentional in-place branch. If the relation was created, or received a new relfilenode, in the **current [subtransaction](../../../glossary.md#subtransaction)**, PostgreSQL can truncate it directly because rollback will discard that current storage anyway. It is not an exception that destroys the old committed contents of an unrelated existing table. The in-place path truncates the heap and rebuilds indexes, including TOAST indexes. [tablecmds.c#same-subtransaction](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1797) [heap.c#heap_truncate_one_rel](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3251-L3287) [heap.c#index-rebuild](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3170-L3205)

| Workflow state | What survives a later failure? |
|---|---|
| Existing populated tables; clear and reload in one transaction; that transaction aborts | The old storage mapping survives, and the transaction's replacement storage is discarded. [storage.c#transactional-lifetime](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L49) [xact.c#abort-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2707-L2720) |
| Clear commits; reload is a separate transaction and aborts | The committed empty state remains. Aborting the later transaction does not restore the already-deleted old storage. [xact.c#commit-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232) [storage.c#delete-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L410-L455) |
| Empty replacement tables are created and committed; loading never begins | Their empty main files can remain. Creation allocates files before any data insertion. [storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L79-L119) [md.c#mdcreate](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L176-L210) |
| Load inserts rows and then fails | No committed loaded rows need survive, but physical space can remain occupied by aborted inserts. “Empty in SQL” is not “zero bytes.” [copy.c#insertion](../../../../raw/postgres-12/src/backend/commands/copy.c#L3292-L3304) [copy.sgml#failure](../../../../raw/postgres-12/doc/src/sgml/ref/copy.sgml#L532-L540) |

`pg_restore` provides a concrete way to reach partial reconstruction. **By default it neither wraps the whole restore in one transaction nor exits on each SQL error.** Table definitions and table data are separate work items. A table creation can therefore remain committed when subsequent data restoration fails. The explicit `--single-transaction` option changes this by enabling a surrounding transaction and error exit; `--exit-on-error` alone does not enable that transaction. These are command options, not server GUC changes. [pg_backup_archiver.c#defaults](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L969-L980) [pg_backup_archiver.c#schema-entry](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L779-L825) [pg_backup_archiver.c#data-entry](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L921-L944) [pg_backup_archiver.c#error-policy](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L1789-L1792) [pg_restore.c#options](../../../../raw/postgres-12/src/bin/pg_dump/pg_restore.c#L172-L174) [pg_restore.c#single-transaction](../../../../raw/postgres-12/src/bin/pg_dump/pg_restore.c#L267-L269) [pg_backup_archiver.c#BEGIN](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L474-L480) [pg_backup_archiver.c#COMMIT](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L720-L726)

Do not confuse that workflow with the parallel restore optimization: when a worker truncates a table that this restore created, it wraps **that truncation and its data load together** in a transaction. A failed load can leave the earlier-created empty table, but the worker does not separately commit its preparatory `TRUNCATE` before `COPY`. [pg_backup_archiver.c#parallel-created-table](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L894-L944)

Even a committed clear does not explain literally zero-length files for every initialized object. The B-tree rebuild writes a metapage when there are no entries. Sequence values remain unchanged by default (`CONTINUE IDENTITY`); explicit `RESTART IDENTITY` invokes sequence reset rather than generically emptying sequence files. [nbtsort.c#empty-tree-metapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1128-L1136) [gram.y#identity-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L6359-L6362) [tablecmds.c#sequence-selection](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1702-L1727) [tablecmds.c#ResetSequence](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1846-L1853)

### Discarded storage waiting for cleanup

The important interaction is between **transactional catalog identity** and **delayed filesystem cleanup**. A table can now point at new storage while its previous physical filename still exists. Conversely, a failed creation can leave discarded filenames even though the created objects do not survive. The pending-delete list records whether each file belongs to the commit or abort cleanup set. [relcache.c#mapping-change](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3445-L3479) [relcache.c#new-catalog-filenode](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3525-L3541) [storage.c#PendingRelDelete](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65)

For ordinary non-temporary storage cleaned up outside redo:

1. **Commit or abort chooses the files.** `smgrDoPendingDeletes()` checks `atCommit` and sends only the selected storage to `smgrdounlinkall()`. A drop of a pre-existing relation schedules deletion at commit, not at statement entry. [storage.c#drop-scheduling](../../../../raw/postgres-12/src/backend/catalog/storage.c#L144-L173) [storage.c#cleanup-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L455)
2. **Storage cleanup discards buffers and invalidates handles.** It does not flush the discarded old data back to its files. [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495)
3. **`mdunlinkfork()` empties the first main file and queues its unlink.** It normally unlinks other forks and numbered segments immediately. The retained main filename prevents premature reuse of the physical identifier before recovery can safely distinguish the old and new storage. [md.c#reason](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L225-L258) [md.c#file-operations](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359)
4. **An eligible checkpoint attempts the deferred unlink.** A request arriving during a checkpoint waits for a later checkpoint cycle. An unlink failure is a warning, so a stub can outlive the normal cleanup interval. Temporary files and redo take the immediate-unlink branch instead. [sync.c#cycle-and-unlink](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214) [md.c#temporary-and-redo-branch](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L284-L298)

Thus **a committed `DROP SCHEMA ... CASCADE` can leave zero-byte first-main-file remnants of the removed relations**; it has not left an intact schema whose live objects all became empty. An orderly failure during creation or a rewrite can likewise leave remnants of *discarded new storage*. A failed drop transaction does not normally delete the pre-existing relations' files. See the separate [DROP SCHEMA walkthrough](drop-schema-zeroed-relation-files.md) for the dependency and catalog paths. [heap.c#drop-storage](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1943-L1959) [storage.c#commit-versus-abort](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L49) [storage.c#delete-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L426-L455) [md.c#deferred-main-file](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L300-L323)

An abrupt process or operating-system failure is not the same as an orderly transaction abort completing this cleanup. The pending list is in backend memory, and filesystem failures can interrupt cleanup. Do not assume every crash leftover has already been converted to a zero-byte stub, or that every unmatched filename is harmless. [storage.c#in-memory-pending-list](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) [md.c#cleanup-failures](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L284-L359)

### Other operations and failure boundaries

| Event or condition | Source-backed behavior | Does it explain every live file of a populated schema becoming zero bytes? |
|---|---|---|
| Ordinary SQL error or transaction abort | The top-level error path aborts the current transaction; storage cleanup selects that transaction's abort-time entries. | Not as a general erase action against pre-existing logged relations. It can explain discarded candidate files, or interact with an earlier committed clear. [postgres.c#error-abort](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4019-L4022) [storage.c#abort-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L455) |
| Disk full while extending or writing a relation | The affected write reports an error; these branches do not enumerate a schema and truncate its files. | Not by itself. It can make a reload fail after earlier stages committed; a separate storage failure still needs evidence. [md.c#extend-error](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L404-L418) [md.c#write-error](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L676-L690) |
| Ordinary [`VACUUM`](../../../glossary.md#vacuum) tail [truncation](../../../glossary.md#truncation) | When eligible, it rechecks trailing pages and cuts only the tail it finds deletable. It can reach an empty heap if no nondeletable pages remain; `mdtruncate()` leaves obsolete segments at zero length. | Explains already-empty heaps or inactive tails, not a schema-wide error handler that erases live rows. [vacuumlazy.c#eligibility](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1852-L1866) [vacuumlazy.c#tail-recheck](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1933-L1965) [vacuumlazy.c#keep-used-pages](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2084-L2133) [md.c#inactive-segments](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L798-L837) |
| `VACUUM FULL` / [`CLUSTER`](../../../glossary.md#cluster) rewrite | Builds another heap, copies data, swaps storage and drops the transient relation that now owns the old storage. | Explains obsolete filenames and, after orderly abort, discarded replacements. Do not judge the surviving table using a pre-rewrite path list. [cluster.c#rebuild](../../../../raw/postgres-12/src/backend/commands/cluster.c#L610-L626) [cluster.c#swap](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1362-L1369) [cluster.c#drop-old-storage](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1449-L1458) |
| `zero_damaged_pages = on` | Default is `off`. Explicitly enabling it substitutes zero page contents on damaged reads instead of the normal error; the short-read path also has a recovery exception. These are page reads, not schema-wide file truncations. | Not a direct explanation for every file length becoming zero. This page does not recommend enabling it. [guc.c#default](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161) [bufmgr.c#damaged-page](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) [md.c#short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L614-L629) |
| Crash with `fsync = off` | Default is `on`. Turning it off bypasses durable flush calls; the same-checkout documentation warns of unrecoverable corruption after a power/system failure. | A durability hypothesis, not proof of this exact zero-length pattern or its schema boundary. No setting change is proposed. [guc.c#fsync-default](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1132) [fd.c#flush-bypass](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L345-L355) [config.sgml#corruption-risk](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2498-L2520) |
| External file changes, a faulty physical restore, storage failure, or added code | The pinned core paths do not identify which actor changed a real incident's files. Catalog membership alone says nothing about the external operation. | Unresolved causes to investigate, not findings established by this source reading. See [Open Questions](#open-questions). |

Temporary relations are a separate persistence class. Omitting `ON COMMIT` does not select row deletion; explicit `ON COMMIT DELETE ROWS` selects the temporary-table in-place truncation path at commit. Its existence is not evidence of the same behavior for a permanent application schema. This page's permanent/unlogged conclusions must not be applied to temporary storage without checking its options. [pg_class.h#persistence-classes](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L165-L167) [gram.y#ON-COMMIT-default](../../../../raw/postgres-12/src/backend/parser/gram.y#L3929-L3932) [tablecmds.c#ON-COMMIT-dispatch](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L14676-L14707) [heap.c#temporary-ON-COMMIT-path](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3210-L3247)

### Read-only inventory and incident evidence

**First collect evidence; do not use a repair, forced checkpoint, rewrite or drop as a diagnostic experiment on the affected database.** A checkpoint can remove queued old filenames, while a rewrite changes the object-to-file mapping. If the schema was already dropped, current catalog enumeration cannot reconstruct the deleted objects' original persistence or paths; a pre-incident inventory or backup is needed. [sync.c#deferred-removal](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214) [relcache.c#mapping-replacement](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3525-L3541) [dbsize.c#current-mapping](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L850-L875)

For a schema that still exists, replace the literal `target_schema` below and run in the affected database. The query inventories its storage-bearing relations plus their TOAST tables and TOAST indexes. It does **not** read file contents, measure lengths, or certify data completeness. `indisvalid` and `indisready` help distinguish an initialized usable index from unfinished index work. These columns and functions were checked against the pin; this SQL was not executed in this task. [pg_class.h#columns](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L29-L81) [pg_index.h#index-identity-and-state](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L43) [index.c#index-namespace](../../../../raw/postgres-12/src/backend/catalog/index.c#L739-L749) [toasting.c#TOAST-namespace](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L228-L235) [pg_proc.dat#mapping-functions](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6916-L6925)

```sql
BEGIN /* wiki_empty_schema_inventory */ READ ONLY;
SET /* wiki_empty_schema_statement_timeout */ LOCAL statement_timeout = '15s';
SET /* wiki_empty_schema_lock_timeout */ LOCAL lock_timeout = '2s';

WITH /* wiki_empty_schema_inventory */ roots AS (
    SELECT c.oid, c.reltoastrelid
    FROM pg_catalog.pg_class AS c
    JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
    WHERE n.nspname = 'target_schema'
), related AS (
    SELECT oid FROM roots
    UNION
    SELECT reltoastrelid FROM roots WHERE reltoastrelid <> 0
    UNION
    SELECT i.indexrelid
    FROM pg_catalog.pg_index AS i
    JOIN roots AS r ON r.reltoastrelid = i.indrelid
)
SELECT c.oid AS relation_oid,
       n.nspname AS schema_name,
       c.relname AS relation_name,
       c.relkind,
       c.relpersistence,
       pg_catalog.pg_relation_filenode(c.oid) AS current_filenode,
       pg_catalog.pg_relation_filepath(c.oid) AS current_main_path,
       c.reltoastrelid AS toast_oid,
       i.indisvalid,
       i.indisready
FROM related AS r
JOIN pg_catalog.pg_class AS c ON c.oid = r.oid
JOIN pg_catalog.pg_namespace AS n ON n.oid = c.relnamespace
LEFT JOIN pg_catalog.pg_index AS i ON i.indexrelid = c.oid
WHERE c.relkind IN ('r', 'i', 'S', 't', 'm')
ORDER BY n.nspname, c.relname, c.oid;

COMMIT /* wiki_empty_schema_inventory */;
```

The two timeout values are diagnostic choices, not measured performance claims. Both settings default to zero, meaning disabled, and have `PGC_USERSET` context. `SET LOCAL` applies them only to this transaction, with no reload or restart. Roll back the diagnostic transaction if a statement fails. The read-only transaction and local-setting syntax are present in the pinned grammar. [guc.c#timeout-defaults-and-context](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396) [guc.c#local-setting-action](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L8100-L8123) [guc.c#transaction-restoration](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L5757-L5772) [gram.y#SET-LOCAL](../../../../raw/postgres-12/src/backend/parser/gram.y#L1409-L1414) [gram.y#BEGIN](../../../../raw/postgres-12/src/backend/parser/gram.y#L9835-L9840) [gram.y#READ-ONLY](../../../../raw/postgres-12/src/backend/parser/gram.y#L9937-L9945)

Interpret the inventory with these boundaries:

- `current_main_path` names the first main-fork segment relative to the data directory, not every fork or numbered segment. A current mapping is not a claim that the named file exists or is healthy. [dbsize.c#filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L1021) [relfilenode.h#multiple-files](../../../../raw/postgres-12/src/include/storage/relfilenode.h#L20-L38)
- It is catalog-derived and does not lock every listed relation. Concurrent drops or rewrites can invalidate the comparison; the mapping functions explicitly allow null results for objects that disappeared. Other-schema partitions and foreign-key cascade targets are not added by this schema filter. [dbsize.c#concurrency-contract](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L850-L862) [pg_class.h#namespace](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L34-L38) [tablecmds.c#cascade-targets](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1658-L1685)
- Zero rows from this inventory mean no matching current storage-bearing catalog rows were found, not that the lost schema was healthy. Missing or damaged metadata is outside the query's ability to reconstruct history. [dbsize.c#catalog-based-mapping](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L850-L875)

For a real incident, collect the exact server minor/build, the actual path list with fork/segment identities and lengths, a pre-incident mapping if available, persistence settings, recovery logs, and the exact maintenance/import/restore commands with their transaction boundaries and outcomes. Determine whether indexes and sequences are also affected, and whether similarly configured objects outside the schema are affected. These are requested evidence, not assumptions about what happened. The existing [file-tail investigation](zeroed-last-segment-check-all-tables.md), [corruption-log reference](../observability/corruption-log-messages-with-checksums-disabled.md), and [post-schema-drop checklist](../server-administration/health-after-zeroed-schema-drop.md) cover follow-up checks and their limits.

### Lifecycle and causal summary

| Event | Before → after | Meaning of empty files |
|---|---|---|
| Recovery of unlogged relations | Existing main contents → main recreated from init; other non-init files removed | Expected loss of unlogged contents, not a schema selector or an all-files-zero layout. [reinit.c#reset](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L222-L308) |
| Clean restart without recovery request or required redo | `InRecovery` remains false → reset calls are not taken | A clean restart alone is not this unlogged-reset scenario. [xlog.c#recovery-gate](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6728-L6749) [xlog.c#init-gate](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7323-L7331) |
| Existing-table `TRUNCATE` aborts | Old storage plus new candidate → old storage retained; candidate discarded | Any empty cleanup stub belongs to discarded storage, not a newly emptied old table. [tablecmds.c#new-storage](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1840) [storage.c#abort-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L455) |
| `TRUNCATE` commits, then a separate reload fails | New empty state committed → later load aborts | A real empty-table state can remain, without every index or auxiliary file being zero length. [xact.c#commit-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232) [storage.c#transaction-lifetime](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L49) [nbtsort.c#metapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1128-L1136) |
| Drop/replacement cleanup, then eligible checkpoint | Obsolete first main file → zero-byte stub → unlink attempted | An old filename observation changes without any live table losing rows at that checkpoint. [md.c#stub](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L300-L323) [sync.c#unlink](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L168-L214) |

**The deciding chain is: identify the current object and file → establish persistence and transaction/recovery state → distinguish an init reset, a committed empty state, and discarded storage → investigate unexplained current-file damage separately.** The implementation supports those mechanisms, not a generic “one SQL operation failed, so PostgreSQL erased that schema” rule. [tablecmds.c#transaction-branches](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1840) [storage.c#cleanup-selection](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L455) [reinit.c#reset-selection](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L180-L308)

### Build boundaries and tests

The table default is the built-in heap [access method](../../../glossary.md#access-method). The table-AM interface dispatches storage creation and nontransactional truncation through callbacks; the heap callbacks create storage/init forks and call `RelationTruncate()`. Index building similarly dispatches through `ambuild` and `ambuildempty`. The B-tree file-layout statements above concern that core implementation, not arbitrary custom callbacks. No contrib extension is required for these core paths or the inventory query. [tableam.h#default](../../../../raw/postgres-12/src/include/access/tableam.h#L27) [tableam.h#storage-dispatch](../../../../raw/postgres-12/src/include/access/tableam.h#L1358-L1377) [heapam_handler.c#callbacks](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L579-L633) [index.c#build-dispatch](../../../../raw/postgres-12/src/backend/catalog/index.c#L2899-L2918) [pg_proc.dat#core-functions](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6916-L6925)

`pg_class`, `pg_index`, `pg_namespace` and function declarations feed catalog generation; the build generates catalog `_d.h` headers. `RELSEG_SIZE` comes from the generated `pg_config.h` and controls segmentation, not schema membership. This investigation did not change build options, generate headers, or modify the raw checkout. [catalog/Makefile#catalogs-and-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L51) [catalog/Makefile#generation](../../../../raw/postgres-12/src/backend/catalog/Makefile#L87-L103) [md.c#segment-constant](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L44-L66)

The inspected tests supply adjacent coverage, **not measurements from this task**:

- `truncate.sql` exercises rollback, commit and foreign-key restrictions/cascade. Its expected output preserves rows after rollback and empties the table after commit. [truncate.sql#transaction-and-FK-tests](../../../../raw/postgres-12/src/test/regress/sql/truncate.sql#L1-L34) [truncate.out#rollback-and-commit](../../../../raw/postgres-12/src/test/regress/expected/truncate.out#L1-L30)
- `014_unlogged_reinit.pl` performs an immediate shutdown, removes main forks, creates fake auxiliary forks and checks recreation/removal after startup, both in the default and another tablespace. It explicitly says restoration from backup is not tested there; it does not test an entire populated schema with every object file set to zero length. [014_unlogged_reinit.pl#scope-and-fixtures](../../../../raw/postgres-12/src/test/recovery/t/014_unlogged_reinit.pl#L1-L81)
- `008_fsm_truncation.pl` tests replay of an ordinary VACUUM truncation's free-space-map effects, followed by promotion/restart and another insert. It is not a schema-wide file-erasure test. [008_fsm_truncation.pl#test](../../../../raw/postgres-12/src/test/recovery/t/008_fsm_truncation.pl#L23-L96)

## Context Reviewed

- Navigation: version manifest, global/version indexes, recent log entries and the question template. The target checkout matches the full pin and identifies itself as 12.2; navigation prose was not used as engine evidence.
- Identity and state: `pg_namespace`, `pg_class`, `pg_index`, `RelFileNode`, fork/segment addressing, current mapping functions, TOAST namespace/persistence, and index persistence inheritance.
- Normal/error paths: utility dispatch; `ExecuteTruncateGuts`; current-subtransaction optimization; table-AM creation/truncation; relcache mapping updates; `PendingRelDelete`; commit/abort callers; `smgrdounlinkall`; `mdunlinkfork`; checkpoint-cycle unlink. The target checkout's local history was also consulted for the deferred-unlink intent; no cross-version regression claim is made.
- Recovery: the `InRecovery` gate, cleanup/init callers, init-file enumeration, heap init creation, index empty-build dispatch, B-tree init and ordinary-build metapages, and the unlogged-sequence rejection.
- Adjacent operations: `pg_restore` defaults, schema/data items and parallel-load transaction; COPY insertion/error documentation; VACUUM tail checks; rewrite/copy/swap/drop; file-write errors; `fsync` and `zero_damaged_pages` defaults and branches.
- Build and tests: core catalog generation, table/index AM boundaries, the regression/TAP inputs listed above, and matching-checkout documentation. No test or SQL block was executed.
- Contradiction comparison: the assembled claim inventory, linked glossary entries, own navigation, v12 pin row, and matching-version hits in the DROP SCHEMA, zeroed-file, last-segment, amcheck, CREATE INDEX CONCURRENTLY and VACUUM FULL pages. PostgreSQL 12 has no common-concept pages for these mechanisms.

## Evidence Map

| Claim | Primary evidence |
|---|---|
| Schema membership is not a physical directory boundary | [pg_class.h#namespace](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L34-L57), [relpath.c#paths](../../../../raw/postgres-12/src/common/relpath.c#L129-L205) |
| Unlogged recovery resets by init files, not schema | [xlog.c#cleanup](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6878-L6884), [reinit.c#selection-and-copy](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L180-L308) |
| Empty unlogged heap does not mean zero-byte initialized B-tree | [heapam_handler.c#init-creation](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L605-L624), [nbtree.c#init-metapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L153-L183) |
| Existing-table truncation protects old storage until commit; current-subtransaction exception is deliberate | [tablecmds.c#branches](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1840), [storage.c#pending-lifetime](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L65) |
| A failed later load cannot roll back an earlier committed clear | [xact.c#commit-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232), [storage.c#transaction-local-cleanup](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462) |
| Restore defaults permit partial reconstruction; parallel preparatory truncation and load share a transaction | [pg_backup_archiver.c#defaults](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L969-L980), [pg_backup_archiver.c#parallel-load](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L894-L944) |
| Dropped/aborted storage can leave zero-byte first-main-file stubs | [storage.c#cleanup](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L455), [md.c#stub-and-unlink](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L359), [sync.c#checkpoint](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214) |
| Inactive zero segments differ from lost live data | [md.c#layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L52-L66), [vacuumlazy.c#tail-check](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1933-L1965) |
| Disk-full and damaged-page read branches are not schema-wide file erasure | [md.c#extend-error](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L404-L418), [bufmgr.c#page-zeroing](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924) |
| Current mapping and open file state limit incident inference | [dbsize.c#mapping-contract](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L850-L862), [md.c#open-segment-state](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L715-L755) |

## Open Questions

- **The actual incident is not established.** No affected database, logs, paths, file lengths, backup, previous catalog inventory, transaction sequence or exact server build was supplied. The question alone cannot distinguish empty query results, zero-length current files, overwritten pages and obsolete filenames, or attribute a specific external/storage fault.
- **Scope of the source conclusion:** this is the pinned 12.2 implementation with the described core paths. No full-schema failure fixture, restore-failure reproduction or storage-fault test was run. Custom access methods, triggers/hooks, a vendor build and later-minor defects/fixes need their own matching evidence; the page does not claim to rule them out.
- **Existing comparison-page wording remains too broad.** The [original zeroed-file page's size explanation](zeroed-relation-file.md#why-scans-stay-silent-the-size-check-stops-early) and the [last-segment page's size explanation](zeroed-last-segment-check-all-tables.md#why-reading-the-last-blocks-cannot-see-a-truncated-segment) say `mdnblocks()` stops at the first short segment without naming already-open state. Their qualified discussions elsewhere, and this page, allow an earlier externally shortened segment to be trusted as full when later segments are already open. The pin supports the qualified claim: discovery starts at the last open segment. Their fresh-state measurements are not contradicted. Those other pages were not edited. [md.c#start-at-last-open-segment](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L715-L737)
- **Another comparison-page condition needs narrowing.** The [last-segment page's false-positive table](zeroed-last-segment-check-all-tables.md#branches-limits-and-false-positives) describes in-place TRUNCATE for a table created or rewritten in the same transaction. The actual test requires the same **current subtransaction**. A creation in a parent subtransaction does not satisfy that equality when truncation runs in a child. This page and the glossary use the source's narrower condition; the other page was not edited. [tablecmds.c#subtransaction-equality](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1776-L1819)
- **Missing concept layer:** PostgreSQL 12 has no common-concept page for transactional relation-file lifecycle or unlogged crash reinitialization. A separately requested relation-file lifecycle page would be useful; none was created as a side effect.

## Source References

- [configure.in#version](../../../../raw/postgres-12/configure.in#L20)
- [pg_namespace.h#schema](../../../../raw/postgres-12/src/include/catalog/pg_namespace.h#L24-L44)
- [pg_class.h#relation-properties](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L29-L167)
- [pg_index.h#index-properties](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L43)
- [pg_proc.dat#mapping-functions](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6916-L6925)
- [relfilenode.h#physical-identity](../../../../raw/postgres-12/src/include/storage/relfilenode.h#L20-L79)
- [rel.h#RelationNeedsWAL](../../../../raw/postgres-12/src/include/utils/rel.h#L515-L520)
- [tableam.h#storage-callbacks](../../../../raw/postgres-12/src/include/access/tableam.h#L1358-L1377)
- [relpath.c#GetRelationPath](../../../../raw/postgres-12/src/common/relpath.c#L129-L205)
- [gram.y#TRUNCATE](../../../../raw/postgres-12/src/backend/parser/gram.y#L6348-L6362)
- [utility.c#TRUNCATE-dispatch](../../../../raw/postgres-12/src/backend/tcop/utility.c#L543-L545)
- [postgres.c#error-recovery](../../../../raw/postgres-12/src/backend/tcop/postgres.c#L4019-L4022)
- [tablecmds.c#ExecuteTruncate](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1504-L1853)
- [storage.c#transactional-storage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L36-L173), [storage.c#pending-delete-execution](../../../../raw/postgres-12/src/backend/catalog/storage.c#L398-L462)
- [heap.c#storage-drop](../../../../raw/postgres-12/src/backend/catalog/heap.c#L1943-L1959), [heap.c#in-place-truncation](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3170-L3287)
- [index.c#index-persistence](../../../../raw/postgres-12/src/backend/catalog/index.c#L739-L749), [index.c#build-dispatch](../../../../raw/postgres-12/src/backend/catalog/index.c#L2899-L2918)
- [toasting.c#namespace-and-persistence](../../../../raw/postgres-12/src/backend/catalog/toasting.c#L228-L268)
- [relcache.c#RelationSetNewRelfilenode](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L3405-L3559)
- [heapam_handler.c#storage-callbacks](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L579-L633)
- [nbtree.c#btbuildempty](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L153-L183)
- [nbtsort.c#empty-tree-metapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1128-L1136)
- [sequence.c#unlogged-rejection](../../../../raw/postgres-12/src/backend/commands/sequence.c#L136-L140)
- [xact.c#commit-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2223-L2232), [xact.c#abort-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L2707-L2720)
- [xlog.c#recovery-gate](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6728-L6749), [xlog.c#unlogged-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6878-L6884), [xlog.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7323-L7331)
- [reinit.c#unlogged-reset](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L36-L358)
- [md.c#segment-layout](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L44-L75), [md.c#create-and-unlink](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L176-L359), [md.c#size-and-truncate](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L705-L849)
- [smgr.c#smgrdounlinkall](../../../../raw/postgres-12/src/backend/storage/smgr/smgr.c#L433-L495)
- [sync.c#checkpoint-unlink](../../../../raw/postgres-12/src/backend/storage/sync/sync.c#L145-L214)
- [bufmgr.c#cached-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L762-L795), [bufmgr.c#damaged-page-read](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924)
- [bufpage.c#all-zero-pages](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L122-L142)
- [vacuumlazy.c#tail-truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1852-L1965), [vacuumlazy.c#tail-recheck](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2084-L2133)
- [cluster.c#rewrite](../../../../raw/postgres-12/src/backend/commands/cluster.c#L610-L626), [cluster.c#swap-and-drop](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1341-L1458)
- [copy.c#insertion](../../../../raw/postgres-12/src/backend/commands/copy.c#L3292-L3304)
- [guc.c#durability-and-page-defaults](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1123-L1161), [guc.c#timeouts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2396)
- [fd.c#fsync-bypass](../../../../raw/postgres-12/src/backend/storage/file/fd.c#L345-L355)
- [dbsize.c#current-mappings](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L850-L1021)
- [pg_restore.c#transaction-option](../../../../raw/postgres-12/src/bin/pg_dump/pg_restore.c#L267-L269)
- [pg_backup_archiver.c#restore-items-and-transactions](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L765-L980), [pg_backup_archiver.c#error-policy](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L1789-L1792)
- [catalog/Makefile#generated-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L103)
- [copy.sgml#failure-space](../../../../raw/postgres-12/doc/src/sgml/ref/copy.sgml#L532-L540)
- [config.sgml#fsync](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L2498-L2520)
- [truncate.sql#transaction-tests](../../../../raw/postgres-12/src/test/regress/sql/truncate.sql#L1-L34), [truncate.out#expected-results](../../../../raw/postgres-12/src/test/regress/expected/truncate.out#L1-L30)
- [014_unlogged_reinit.pl#test](../../../../raw/postgres-12/src/test/recovery/t/014_unlogged_reinit.pl#L1-L81)
- [008_fsm_truncation.pl#test](../../../../raw/postgres-12/src/test/recovery/t/008_fsm_truncation.pl#L23-L96)

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Version pins](../../../versions.md)
- [DROP SCHEMA After Relation Files Were Emptied (unverified)](drop-schema-zeroed-relation-files.md)
- [A Manually Zeroed Relation File (unverified)](zeroed-relation-file.md)
- [Checking Every Table for a Zeroed Last Segment (unverified)](zeroed-last-segment-check-all-tables.md)
- [Can amcheck Detect Zeroed Relation Segments? (unverified)](../indexing/amcheck-zeroed-relation-segments.md)
- [Database Health After Schema Files Were Zeroed and the Schema Dropped (unverified)](../server-administration/health-after-zeroed-schema-drop.md)
