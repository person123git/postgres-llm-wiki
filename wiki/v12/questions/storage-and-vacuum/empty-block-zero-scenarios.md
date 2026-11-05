---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Scenarios That Leave Block Zero of a Schema's Objects Empty in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Answer up front](#answer-up-front)
  - [What an empty block zero can mean](#what-an-empty-block-zero-can-mean)
  - [Which objects have a block zero, and what it holds](#which-objects-have-a-block-zero-and-what-it-holds)
  - [Decision map](#decision-map)
  - [Scenario catalog](#scenario-catalog)
  - [Family A: ordinary operations that leave no rows](#family-a-ordinary-operations-that-leave-no-rows)
  - [Family B: crashes, recovery and repair settings](#family-b-crashes-recovery-and-repair-settings)
  - [Family C: interrupted concurrent index builds](#family-c-interrupted-concurrent-index-builds)
  - [Family D and E: leftover files and outside causes](#family-d-and-e-leftover-files-and-outside-causes)
  - [The free space map and visibility map forks](#the-free-space-map-and-visibility-map-forks)
  - [Lifecycle of block zero](#lifecycle-of-block-zero)
  - [Non-default settings that change the outcome](#non-default-settings-that-change-the-outcome)
  - [How to tell the states apart](#how-to-tell-the-states-apart)
  - [Measured results](#measured-results)
  - [Tests in the pinned tree](#tests-in-the-pinned-tree)
  - [Final causal summary](#final-causal-summary)
- [Measurement Script](#measurement-script)
  - [Usage](#usage)
  - [Last run](#last-run)
  - [Non-default settings in the measurement](#non-default-settings-in-the-measurement)
  - [The script](#the-script)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

List all possible scenarios that lead to block zero of the objects in a schema being empty.

> Prompt note: the original prompt was "list all possible scenarios that leads to have the block zero of objects in a schema empty?". Corrections: capitalized the first word, changed "leads" to "lead", rephrased "to have the block zero of objects in a schema empty" as "to block zero of the objects in a schema being empty", and dropped the question mark because the sentence is an instruction. The meaning is unchanged. The user asked for PostgreSQL 12; this page uses the repository's 12.2 pin.

## Answer

### Answer up front

**Block zero is the first [page](../../../glossary.md#page) of a [relation](../../../glossary.md#relation)'s main [fork](../../../glossary.md#fork).** "Empty" can mean five different physical states, and each object kind reaches a different subset of them. So the scenarios below are grouped by the state they leave, not only by the command that caused them.

**Tables, materialized views and [TOAST](../../../glossary.md#toast) tables lose block zero entirely in ordinary, healthy operation.** Creating one, [`TRUNCATE`](../../../glossary.md#truncate), a [VACUUM](../../../glossary.md#vacuum) that [truncates](../../../glossary.md#truncation) the whole table, a rewrite of a table with no live rows, and the crash reset of an [unlogged relation](../../../glossary.md#unlogged-relation) all leave a main fork with zero blocks. This was measured on an isolated 12.2 build; see [Measured results](#measured-results).

**A heap block zero that exists but holds no tuples is also normal.** A VACUUM that cannot or may not truncate leaves block zero with only unused [line pointers](../../../glossary.md#line-pointer). [Relation extension](../../../glossary.md#relation-extension) can leave an [all-zero page](../../../glossary.md#all-zero-page): bulk extension does so on purpose, and a crash after a single-page extension can do so by accident.

**Indexes keep an initialized block zero in every ordinary path.** Every core index [access method](../../../glossary.md#access-method) writes a [metapage](../../../glossary.md#metapage) or root page there when it builds, even on an empty table. The exceptions are interrupted concurrent builds, whose catalog row already committed. A failed or cancelled `CREATE INDEX CONCURRENTLY` or `REINDEX CONCURRENTLY` can leave a B-tree with no block zero, or with an all-zero block zero; both outcomes were measured. A server crash during such a build can leave a GiST, GIN or SP-GiST block zero all-zero in the file; that case is derived from the source only. Such an index is always an [invalid index](../../../glossary.md#invalid-index). Recovery extension and outside damage, in Families B and E, can affect indexes too.

**Sequences always keep block zero with one tuple** on the paths examined here, because the code that creates or resets a sequence writes that tuple in the same step.

**A schema is never the selector.** No storage operation chooses relations by [schema](../../../glossary.md#schema). A whole schema shows empty block zeros only when the same per-relation scenario hit every object in it, for example a scripted `TRUNCATE` of each table, a crash with only unlogged tables, or an outside process that emptied the files. The companion page [Failure Scenarios That Can Leave Schema Relation Files Empty (unverified)](schema-empty-relation-files-failure-scenarios.md) covers that schema-wide view for zero-length files.

This is an analysis of the **PostgreSQL 12.2** pin, plus measurements on one isolated server built from it. It is not a diagnosis of a particular database.

### What an empty block zero can mean

A page starts with a 24-byte header. Two header fields decide the states below: `pd_lower`, the end of the line pointer array, and `pd_upper`, the start of tuple space. `PageIsEmpty()` is true when `pd_lower` has not grown past the header, so the page has no line pointers. `PageIsNew()` is true when `pd_upper` is zero, which only a never-initialized page has ([bufpage.h#PageIsEmpty-PageIsNew](../../../../raw/postgres-12/src/include/storage/bufpage.h#L213-L229)). `PageInit()` is what sets those fields on a fresh page ([bufpage.c#PageInit](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L42-L60)).

| State | What it is | Where the value comes from | How it reads through SQL |
|---|---|---|---|
| S0 | The object has no storage at all | `RELKIND_HAS_STORAGE()` is false, so `heap_create()` creates no file ([pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L183-L192), [heap.c#create-storage](../../../../raw/postgres-12/src/backend/catalog/heap.c#L368-L380)) | `pg_relation_filepath()` returns NULL ([dbsize.c#pg_relation_filepath](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L940-L991)) |
| S1 | The main fork has zero blocks, so block zero does not exist | File length, read live by the [storage manager](../../../glossary.md#storage-manager) | `pg_relation_size(rel, 'main')` is 0, and `get_raw_page(rel, 0)` fails with "block number 0 is out of range" ([rawpage.c#get_raw_page_internal](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L148-L152)) |
| S2 | Block zero exists and is all zero bytes | Bytes in the file or in a shared buffer | `page_header()` shows `lsn 0/0`, `lower 0`, `upper 0` |
| S3 | Block zero is initialized and has no line pointers | `pd_lower` equals the 24-byte header size | `page_header()` shows `lower 24` |
| S4 | A heap block zero has line pointers but no stored tuple | Every line pointer is unused or dead | `heap_page_items()` shows no `lp_flags` 1 or 2 row |
| S5 | An index block zero is a metapage or root page of an index with no entries | The access method's build or empty-build code | For a B-tree, `bt_metap()` shows `root 0` and `level 0` |

**S2 needs one extra caution.** A read goes to [shared buffers](../../../glossary.md#buffer-manager) first. So a backend can see an initialized page in a buffer while the file still holds zeros, and the reverse ([bufmgr.c#buffer-hit](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L745-L757)). The SQL inspection functions read through that same buffer path ([rawpage.c#get_raw_page_internal](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L95-L172)). Crash-related S2 cases are about the file, which is what survives the crash.

**S2 is never rejected as corrupt.** `PageIsVerified()` accepts a page that is entirely zero, with or without [data checksums](../../../glossary.md#data-checksums), because an all-zero page is not evidence of damage on its own ([bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L150)).

### Which objects have a block zero, and what it holds

Only tables, indexes, sequences, TOAST tables and materialized views have storage ([pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L183-L192)). Views, composite types, foreign tables, partitioned tables and partitioned indexes are S0 by construction ([heap.c#create-storage](../../../../raw/postgres-12/src/backend/catalog/heap.c#L416-L443)). A table's TOAST table and TOAST index live in schema `pg_toast`, not in the table's schema, so a schema inventory must follow `pg_class.reltoastrelid` to find them.

| Object | Block zero holds | Built by | Measured on an empty table |
|---|---|---|---|
| Table, TOAST table, materialized view | The first heap data page | `RelationCreateStorage()` creates the file with no blocks ([storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L70-L122)); the first insert extends it ([hio.c#single-page-extension](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L585-L609)) | S1 |
| B-tree index | The metapage ([nbtree.h#BTREE_METAPAGE](../../../../raw/postgres-12/src/include/access/nbtree.h#L131)) | The sorted build writes the metapage last, pointing at no root when there were no tuples ([nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1127-L1137)) | S5, `root 0`, `level 0` |
| GiST index | The root page ([gist_private.h#GIST_ROOT_BLKNO](../../../../raw/postgres-12/src/include/access/gist_private.h#L263)) | `gistbuild()` initializes an empty leaf root first ([gistbuild.c#root-init](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L173-L185)) | S3, `lower 24` |
| GIN index | The metapage; block 1 is the root ([ginblock.h#block-numbers](../../../../raw/postgres-12/src/include/access/ginblock.h#L51-L52)) | `ginbuild()` initializes both first ([gininsert.c#meta-and-root](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L339-L352)) | S5, 2 blocks |
| SP-GiST index | The metapage; blocks 1 and 2 are roots ([spgist_private.h#block-numbers](../../../../raw/postgres-12/src/include/access/spgist_private.h#L26-L27)) | `spgbuild()` initializes all three first ([spginsert.c#meta-and-roots](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L88-L109)) | S5, 3 blocks |
| BRIN index | The metapage ([brin_page.h#BRIN_METAPAGE_BLKNO](../../../../raw/postgres-12/src/include/access/brin_page.h#L75)) | `brinbuild()` writes and WAL-logs it first ([brin.c#metapage](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L684-L695)) | S5, 3 blocks |
| Hash index | The metapage ([hash.h#HASH_METAPAGE](../../../../raw/postgres-12/src/include/access/hash.h#L196)) | `_hash_init()` writes and WAL-logs it, then the buckets and bitmap ([hashpage.c#_hash_init](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L351-L400)) | S5, 4 blocks |
| Sequence | The single sequence tuple | `fill_seq_with_data()` extends to block zero and writes the tuple at once ([sequence.c#fill_seq_with_data](../../../../raw/postgres-12/src/backend/commands/sequence.c#L343-L360)) | 1 tuple |

Every index access method refuses to build into a file that already has blocks ("index already contains data"), so a build always starts from an S1 file ([nbtsort.c:334](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L334), [gistbuild.c:160](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L160), [gininsert.c:332](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L332), [spginsert.c:85](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L85), [brin.c:676](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L676), [hash.c:123](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L123)).

A TOAST table gains its first block only when a stored value is moved out of line. `heap_prepare_insert()` calls the toaster only for a tuple that has external values or exceeds `TOAST_TUPLE_THRESHOLD` ([heapam.c:2087](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L2087), [tuptoaster.h#TOAST_TUPLE_THRESHOLD](../../../../raw/postgres-12/src/include/access/tuptoaster.h#L55)). So a TOAST table with no block zero is normal even for a large table: the measured `payments` table had 37,038 heap blocks and a TOAST table with none.

### Decision map

Start from what you observe on one relation. The letters refer to the [Scenario catalog](#scenario-catalog).

```text
Does the object have storage? (relkind r, m, t, i or S)
|
+-- No -> S0: views, foreign tables, partitioned parents.  Nothing to explain.
|
`-- Yes: how many blocks does the main fork have?
    |
    +-- 0 blocks (S1)
    |   +-- table / matview / TOAST -> A1 never loaded, A2 TRUNCATE,
    |   |      A3 VACUUM truncation, A5 rewrite with no rows,
    |   |      B3 unlogged reset, E1/E2 outside causes
    |   +-- index -> only C1/C2: concurrent build failed before
    |   |      writing any page (index is invalid), or E1/E2
    |   `-- sequence -> no ordinary path; E1/E2
    |
    `-- 1 or more blocks: read block 0
        |
        +-- all zero (S2)
        |   +-- heap -> B2 bulk extension (no crash needed),
        |   |      B1 crash after single-page extension, B4 recovery
        |   |      extension, B5 pg_resetwal, B6 zero_damaged_pages
        |   |      (in memory only), E1 outside zero-fill
        |   `-- index -> C1/C2 B-tree build interrupted while loading
        |          (index is invalid), C3 crash during a GiST, GIN or
        |          SP-GiST concurrent build, B4, E1
        |
        +-- initialized, no line pointers (S3)
        |   +-- heap -> A7 an insert initialized the page, then failed
        |   |      or put its tuple on another page
        |   `-- index -> GiST root of an empty index (A6)
        |
        +-- heap page, line pointers but no tuples (S4)
        |      -> A4 VACUUM did not truncate
        |
        `-- index metapage of an empty index (S5)
               -> A1/A2/A6 built or rebuilt on an empty table,
                  B3 unlogged index reset, A3 after VACUUM emptied it
```

### Scenario catalog

| # | Scenario | Block zero becomes | Objects | Normal operation? | Survives restart? | Measured |
|---|---|---|---|---|---|---|
| A1 | Created and never loaded: `CREATE TABLE`, partition leaf, CTAS or matview `WITH NO DATA` or with zero rows | S1 heap, S5 index | tables, matviews, TOAST, their indexes | yes | yes | yes |
| A2 | `TRUNCATE` | S1 heap, S5 index | tables, TOAST, indexes | yes | yes | yes |
| A3 | All rows deleted, then VACUUM truncates to zero blocks | S1 heap; index keeps its pages | tables, TOAST | yes | yes | yes |
| A4 | Rows deleted, VACUUM ran but did not truncate | S4 | tables, TOAST | yes | yes | yes |
| A5 | Rewrite with no live rows: `VACUUM FULL`, `CLUSTER`, rewriting `ALTER TABLE`, `REFRESH MATERIALIZED VIEW` | S1 | tables, matviews | yes | yes | `VACUUM FULL`, `REFRESH` |
| A6 | Index built on an empty table | S5, or S3 for GiST | indexes | yes | yes | yes |
| A7 | An insert initialized a new page, then failed or placed its tuple elsewhere | S3 | tables, TOAST | yes, or an error | yes, once the dirty buffer is written | no |
| B1 | Crash after a single-page extension, before the first insert's WAL reached disk | S2 in the file | tables, TOAST | crash | yes | no |
| B2 | Bulk extension under extension-lock contention | S2, no crash needed | tables, TOAST | yes | yes | no |
| B3 | Unlogged relation after crash recovery, or on any server started from a base backup | S1 heap, S5 index | unlogged tables, their TOAST tables and indexes | yes, by design | is the restart | yes |
| B4 | Recovery replays a record for a block beyond the file's end | S2 for the gap | any WAL-logged relation | crash or standby | yes | no |
| B5 | WAL discarded with `pg_resetwal` after a crash | S2 persists | tables | repair | yes | no |
| B6 | `zero_damaged_pages = on` meets a damaged block zero | S2 in memory only; a later insert writes an initialized page holding its row | any | non-default setting | no, unless rewritten | no |
| B7 | WAL-skipped load under `wal_level = minimal`, then a crash | not established; see [Open Questions](#open-questions) | tables | non-default setting | — | no |
| C1 | `CREATE INDEX CONCURRENTLY` fails or is cancelled during its build | S1 or S2, index invalid | B-tree | failure | yes | yes |
| C2 | `REINDEX CONCURRENTLY` fails or is cancelled during its build | S1 or S2 for the `_ccnew` copy | B-tree | failure | yes | no; same code as C1 |
| C3 | Crash during a GiST, GIN or SP-GiST concurrent build | S2 in the file unless block zero was flushed | those indexes | crash | yes | no |
| D1 | Dropped or replaced storage waiting for the next checkpoint, or an orphaned file left by a crash | first segment is 0 bytes, or any state; no object | files only | yes, or crash | stub until a checkpoint; orphan indefinitely | no |
| E1 | Outside truncation, zero-fill or filesystem loss | S1 or S2 | any | no | yes | elsewhere |
| E2 | `pg_rewind` matched a file to a source file of that length | S1 | any | tool behavior | yes | no |

### Family A: ordinary operations that leave no rows

**A1. Created and never loaded.** `RelationCreateStorage()` creates only the main fork file and adds no block ([storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L70-L122)). For a table, TOAST table or materialized view, `heap_create()` reaches it through the table access method callback ([heap.c#create-storage](../../../../raw/postgres-12/src/backend/catalog/heap.c#L416-L443), [heapam_handler.c#heapam_relation_set_new_filenode](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L580-L627)). `CREATE TABLE AS ... WITH NO DATA` defines the relation without running the query at all ([createas.c#skipData](../../../../raw/postgres-12/src/backend/commands/createas.c#L298-L307)). A load that produces zero rows also writes no heap page, because a page is only added when an insert needs space ([hio.c#single-page-extension](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L585-L609)). Measured: `customers`, `customer_import`, `customer_summary` and the TOAST table all had zero blocks, while the TOAST index had a B-tree metapage.

**A2. TRUNCATE.** For a table that existed before the current transaction, `TRUNCATE` gives the heap and its TOAST table a new, empty relfilenode and rebuilds every index ([tablecmds.c#ExecuteTruncateGuts](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1840)). If the table was created or given a new relfilenode in the same subtransaction, it truncates the existing file to zero blocks instead ([heapam_handler.c#nontransactional_truncate](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L630-L633)). Either way the heap ends in S1 and each index in S5. Measured: `orders` went from 109 blocks to none, its relfilenode changed, and `orders_pkey` became a single metapage.

**A3. DELETE everything, then VACUUM truncates.** VACUUM tracks `nonempty_pages`, one past the last page that still holds a tuple ([vacuumlazy.c:1376-1378](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1376-L1378)). It attempts [truncation](../../../glossary.md#truncation) only when the truncate option is enabled, at least 1,000 pages or one sixteenth of the table are freeable, and `old_snapshot_threshold` is disabled ([vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1851-L1867), [vacuumlazy.c#REL_TRUNCATE](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L68-L72)). It then rechecks the tail under an `AccessExclusiveLock` and truncates ([vacuumlazy.c#truncate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1947-L1992)). A page whose line pointers are all unused counts as deletable ([vacuumlazy.c#count_nondeletable_pages](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L2092-L2125)). So when every row is gone, the table is cut to zero blocks. Measured: `orders` went from 109 blocks to none. Its index did not shrink: `orders_pkey` kept 57 blocks with block zero still a metapage, because only the heap is truncated.

**A4. VACUUM ran but did not truncate.** VACUUM marks the dead tuples' line pointers unused and compacts the page, but PostgreSQL 12 never shortens the line pointer array, so `pd_lower` stays where it was ([vacuumlazy.c#lazy_vacuum_page](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1604-L1619), [bufpage.c#PageRepairFragmentation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L516-L569)). Block zero then has line pointers and no tuple: state S4. Truncation is skipped in these cases:

| Condition | Default | Why truncation is skipped |
|---|---|---|
| Storage parameter `vacuum_truncate = false` | `true` ([reloptions.c#vacuum_truncate](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L152-L160)) | `vacuum_rel()` copies the parameter into the truncate option when the command gave none ([vacuum.c#truncate-option](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1782-L1790)), and a disabled option returns false at once ([vacuumlazy.c:1856-1857](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1856-L1857)) |
| `VACUUM (TRUNCATE false)` | option absent, so the parameter decides | The same disabled option, for that command only |
| `old_snapshot_threshold` set to 0 or more | `-1`, disabled ([guc.c#old_snapshot_threshold](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3041-L3049)) | Truncation would hide pages that a "snapshot too old" check needs ([vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1833-L1867)) |
| A live tuple remains on a later page | — | `nonempty_pages` stays past block zero, so nothing up to that page is freeable |
| Too few freeable pages | — | Fewer than 1,000 pages and less than one sixteenth of the table |
| The exclusive lock is not granted within about 5 seconds, or another backend waits for it | — | VACUUM gives up or stops truncating early ([vacuumlazy.c#lock-retry](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1899-L1922)) |
| VACUUM has not run yet | — | Deleted rows stay as stored tuples, so block zero is not empty at all |

Measured: `order_archive` with `vacuum_truncate = false`, `order_audit` after `VACUUM (TRUNCATE false)`, and `order_events` with one live row on its last page each kept 109 blocks. Block zero had `lower 764` and `upper 8192`, which is 185 line pointers, all with `lp_flags` 0 (unused).

**A5. A rewrite with no live rows.** `VACUUM FULL`, `CLUSTER` and rewriting `ALTER TABLE` forms copy live tuples into a new relfilenode. `end_heap_rewrite()` writes a last page only if one holds a tuple ([rewriteheap.c#end_heap_rewrite](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L313-L345)), so no live tuple means no block. `REFRESH MATERIALIZED VIEW` fills a new heap and swaps it in ([matview.c#refresh](../../../../raw/postgres-12/src/backend/commands/matview.c#L296-L340)), so a query that returns no rows leaves S1. Measured: `order_archive` after `VACUUM FULL` and `order_event_totals` after a refresh with zero rows both had zero blocks. `ALTER TABLE ... SET TABLESPACE` is different: `RelationCopyStorage()` copies every existing block, all-zero ones included, so it preserves whatever block zero was ([storage.c#RelationCopyStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L305-L365)).

**A6. An index built on an empty table.** Each access method initializes block zero before it reads any heap tuple, as the table in [Which objects have a block zero](#which-objects-have-a-block-zero-and-what-it-holds) shows. The B-tree is the exception in order only: it writes the metapage last, which matters in Family C. Measured on the empty `customers` table: the B-tree, GIN, SP-GiST, BRIN and hash indexes had initialized metapages, and the GiST root had no line pointers.

**A7. A page initialized for an insert that did not stay.** When an insert picks an all-zero page from the free space map, `RelationGetBufferForTuple()` initializes it and marks the buffer dirty before it checks whether the tuple fits ([hio.c#init-on-use](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L486-L508)). If the tuple plus the page's fillfactor reserve does not fit, the insert moves on and leaves the page initialized with no line pointer. An ERROR after the initialization has the same effect, and the VACUUM code names that case ([vacuumlazy.c#empty-page](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L930-L960)). The result is S3, an [empty page](../../../glossary.md#empty-page). VACUUM marks such a page all-visible and WAL-logs it if it never was. Block zero reaches this state only if it was an all-zero page first, as in B1 or B2.

### Family B: crashes, recovery and repair settings

**B1. A crash after a single-page extension.** Extension goes through `ReadBuffer` with `P_NEW`. The buffer manager writes an all-zero page to the file with `smgrextend()` and hands back a zeroed buffer ([bufmgr.c#extend](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L866-L879)). `RelationGetBufferForTuple()` then runs `PageInit()` only in that buffer ([hio.c#single-page-extension](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L594-L609)). The insert's WAL record carries `XLOG_HEAP_INIT_PAGE` when it is the first tuple on the page, so replay rebuilds the page ([heapam.c#init-page-flag](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1955-L1965), [heapam.c#heap_xlog_insert](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L8142-L8151)). The steps that leave S2:

1. The file grows by one all-zero block. This write happens at once, unlike the page's content.
2. The initialized, filled page stays in a dirty shared buffer.
3. The server crashes before that buffer is written and before the insert's WAL record is flushed.
4. Recovery finds no record for the block, so nothing rewrites it. Block zero stays all-zero on disk.

The VACUUM code names exactly this case as a source of all-zero pages and treats them as free space ([vacuumlazy.c#new-page-handling](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L885-L928)).

**B2. Bulk extension.** When a backend has to wait for the relation extension lock, it extends by `20` pages per waiter, up to 512, records each new page in the [free space map](../../../glossary.md#free-space-map) and deliberately does not initialize it ([hio.c#contended-extension](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L555-L582), [hio.c#RelationAddExtraBlocks](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L186-L261)). A page is initialized only when an insert later picks it ([hio.c#init-on-use](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L486-L500)). So unused bulk-extended pages stay all-zero with no crash at all. Block zero is in that set only when several sessions insert into a still-empty table at once. Commit `0d1fe9f74e3` (2019-02-03, in 12) made this the design: it moved page initialization out of `RelationAddExtraBlocks()` and made VACUUM accept uninitialized pages as normal free space.

**B3. Unlogged relations after recovery.** A server that needs recovery first removes every fork of each relation that has an init fork, except the init fork, and after replay copies the init fork over the main fork ([xlog.c#recovery-gate](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6728-L6749), [xlog.c#unlogged-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6878-L6884), [xlog.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7323-L7331), [reinit.c#init-copy](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L274-L345)). A heap's init fork is an empty file ([heapam_handler.c#init-fork](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L616-L624)), so the table becomes S1. An index's init fork holds an empty index, so the index becomes S5 ([nbtree.c#btbuildempty](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L153-L183), [gist.c#gistbuildempty](../../../../raw/postgres-12/src/backend/access/gist/gist.c#L125-L142), [gininsert.c#ginbuildempty](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L434-L460), [spginsert.c#spgbuildempty](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L156-L201), [brin.c#brinbuildempty](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L746-L764), [hash.c#hashbuildempty](../../../../raw/postgres-12/src/backend/access/hash/hash.c#L194-L197), [storage.sgml#init-fork](../../../../raw/postgres-12/doc/src/sgml/storage.sgml#L687-L693)). A [base backup](../../../glossary.md#base-backup) leaves out every unlogged fork except the init fork ([basebackup.c#unlogged-exclusion](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1150)). A server started from one runs recovery, and the init copy happens when that recovery ends, so a standby gets the same S1 and S5 state when it is promoted ([xlog.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7323-L7331)). Measured: `cart_sessions` had 20,000 rows and 167 blocks; after an immediate shutdown and restart it had 0 rows and 0 blocks, and `cart_sessions_pkey` was one metapage with `root 0`.

**B4. Recovery extends a file to reach a block.** When a WAL record names a block past the end of the file, `XLogReadBufferExtended()` creates the file if needed and extends it one zeroed page at a time until the block exists ([xlogutils.c#XLogReadBufferExtended](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L437-L520)). A block in that gap stays all-zero unless a later record initializes it. During recovery a short read also returns zeros instead of an error ([md.c#mdread-short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)). Block zero lands in the gap only when the first record replayed for a file targets a later block.

**B5. pg_resetwal after a crash.** [pg_resetwal](../../../glossary.md#pg_resetwal) throws away WAL that recovery would have replayed, and the documentation warns that the data may then be inconsistent ([pg_resetwal.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/pg_resetwal.sgml#L46-L60)). Combined with B1, a block zero whose content existed only in WAL and in a lost buffer stays all-zero.

**B6. zero_damaged_pages.** [zero_damaged_pages](../../../glossary.md#zero_damaged_pages) is off by default and is a `PGC_SUSET` setting, so a superuser can turn it on for a session with `SET` ([guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161)). With it on, a page that fails verification is replaced by zeros in the buffer with a WARNING instead of an ERROR ([bufmgr.c#verify-or-zero](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924)), and a short read returns zeros ([md.c#mdread-short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630)). The documentation says such pages are not forced to disk ([config.sgml#zero_damaged_pages](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L9745-L9768)). So the file keeps its damaged bytes until something modifies the page: an insert initializes it and writes its own row, and an evicted unmodified buffer is simply dropped.

**B7. WAL-skipped bulk loads.** With `wal_level = minimal`, `COPY` into a table created or truncated in the same transaction, CTAS, materialized view fills and table rewrites skip WAL for the new pages and sync the file before commit instead ([copy.c#wal-skip](../../../../raw/postgres-12/src/backend/commands/copy.c#L2718-L2774), [createas.c:561-562](../../../../raw/postgres-12/src/backend/commands/createas.c#L561-L562), [matview.c:465-467](../../../../raw/postgres-12/src/backend/commands/matview.c#L465-L467), [tablecmds.c:4798-4799](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L4798-L4799), [heapam_handler.c#heap_sync](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L565-L571)). The default `wal_level` is `replica`, which never skips ([guc.c#wal_level](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4410-L4416)). The `COPY` comment itself records a case where the same-transaction test is wrong. This page does not establish whether that case can leave a committed block zero all-zero after a crash; see [Open Questions](#open-questions).

### Family C: interrupted concurrent index builds

**Why only concurrent builds.** A plain `CREATE INDEX` or `REINDEX` builds in the same transaction that creates the storage. If it fails, the abort deletes the new file, because `RelationCreateStorage()` queued it for deletion at abort ([storage.c#pending-delete](../../../../raw/postgres-12/src/backend/catalog/storage.c#L111-L119), [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L399-L446)). `CREATE INDEX CONCURRENTLY` instead commits the catalog row and the empty file first, then builds in a later transaction ([indexcmds.c#first-commit](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1316-L1320), [indexcmds.c#build-transaction](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1366-L1379)). A failure in that build aborts only the build transaction, so the committed index keeps whatever the build had written. The [CIC page](../indexing/create-index-concurrently.md) and the [invalid-index outcomes page](../indexing/invalid-index-outcomes.md) describe the transaction sequence in full.

**C1. The B-tree build writes block zero last.** The sorted B-tree build reserves block zero for the metapage and starts the leaf pages at block 1 ([nbtsort.c#reserve-metapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L575-L589)). When it writes block 1, it first fills the skipped block zero with an all-zero page, which it does not WAL-log ([nbtsort.c#_bt_blwritepage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L652-L700)). It writes the real metapage only after every level is done ([nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1127-L1137)). So the failure point decides block zero:

| Build stops | Block zero | Measured |
|---|---|---|
| Before any index page is written: the heap scan or the sort, including a unique violation found while sorting ([tuplesort.c#unique-violation](../../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4031-L4056)) | S1, the file has no blocks | `payments_id_uniq`: 0 blocks after "could not create unique index" |
| After the first leaf page and before the metapage | S2, all-zero, followed by leaf pages | `payments_token_idx`: all-zero block zero, followed by a few hundred leaf blocks |
| After the metapage | initialized metapage, index still invalid until the later phases finish | not measured |

In both measured cases `indisvalid` and `indisready` were false. The planner ignores an invalid index ([plancat.c#invalid-index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L200-L210)), and `amcheck` refuses to check one ([verify_nbtree.c#invalid](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L332-L337)). B-tree code that does read such a page reports "index contains unexpected zero page at block 0" and hints to REINDEX ([nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L692-L708)). Measured: `REINDEX INDEX payments_token_idx` produced a valid index with a metapage at block zero.

**C2. REINDEX CONCURRENTLY.** It creates the `_ccnew` copy with `INDEX_CREATE_SKIP_BUILD | INDEX_CREATE_CONCURRENT`, commits, and builds it in a later transaction through the same `index_concurrently_build()` ([index.c#index_concurrently_create_copy](../../../../raw/postgres-12/src/backend/catalog/index.c#L1362-L1380), [indexcmds.c#reindex-build](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L3077-L3127)). So a failed or cancelled build leaves the `_ccnew` B-tree in the same S1 or S2 state as C1. The original index is untouched. The [REINDEX CONCURRENTLY page](../indexing/reindex-index-concurrently.md) covers its phases.

**C3. Other access methods.** GiST, GIN and SP-GiST initialize block zero first, in a shared buffer, and WAL-log all their pages only when the build ends ([gistbuild.c#root-init](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L173-L185), [gistbuild.c#log-at-end](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L221-L226), [gininsert.c#meta-and-root](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L339-L352), [gininsert.c#log-at-end](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L412-L416), [spginsert.c#meta-and-roots](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L88-L109), [spginsert.c#log-at-end](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L138-L142)). An ordinary error therefore leaves an initialized block zero in the buffer, which is eventually written. Only a server crash during the build can leave block zero all-zero in the file: the extension wrote zeros ([bufmgr.c#extend](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L866-L879)), and no WAL record exists yet to restore the page. BRIN and hash WAL-log their metapage as soon as they write it, so recovery restores it ([brin.c#metapage](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L684-L695), [hashpage.c#_hash_init](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L351-L400)). C3 is derived from the source and was not measured.

### Family D and E: leftover files and outside causes

**D1. Storage waiting to be removed.** Dropping a relation, and the old file of a `TRUNCATE` or rewrite, do not remove the first main-fork segment at once. `mdunlinkfork()` truncates it to zero bytes and registers it for removal after the next [checkpoint](../../../glossary.md#checkpoint) ([md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L323), [md.c#register_unlink_segment](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L916-L930)). Such a file has no block zero, but it also belongs to no live object. A server crash during a transaction that created storage, such as a plain `CREATE INDEX` or `CREATE TABLE AS`, leaves a different leftover: an [orphaned relation file](../../../glossary.md#orphaned-relation-file). An orderly abort writes an abort record that lists the created files, and recovery drops the files such a record lists ([xact.c#abort-record-files](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1650-L1652), [xact.c#xact_redo_abort](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L5925-L5926)). A crash writes no abort record, so the file keeps whatever the build had written, including an all-zero B-tree block zero ([nbtsort.c#_bt_blwritepage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L652-L700)), while no catalog row points to it. The [empty relation files page](schema-empty-relation-files-failure-scenarios.md) covers this case and how to tell it from a live file.

**E1. Outside causes.** Truncating or zero-filling a file from outside the server, losing data in the filesystem, or restoring a file copied while it was being extended can leave S1 or S2. The source does not create these states; it only tolerates them: an all-zero page passes verification ([bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L150)) and VACUUM treats it as free space ([vacuumlazy.c#new-page-handling](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L885-L928)). The [manually zeroed relation file page](zeroed-relation-file.md) and the [zeroed last segment checker](zeroed-last-segment-check-all-tables.md) show what reads report after such a change.

**E2. pg_rewind.** For a relation file that exists on both servers, `pg_rewind` copies the missing tail when the target is shorter and truncates the target when it is longer ([filemap.c#size-compare](../../../../raw/postgres-12/src/bin/pg_rewind/filemap.c#L262-L291)). A target file can therefore end with no block zero when the source file had none. This mirrors the source server's state; it is not a new cause.

### The free space map and visibility map forks

The free space map and [visibility map](../../../glossary.md#visibility-map) forks are not created with the relation. Each is created and extended on first need, and every page it adds is initialized at once ([storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L70-L77), [freespace.c#fsm_extend](../../../../raw/postgres-12/src/backend/storage/freespace/freespace.c#L615-L661), [visibilitymap.c#vm_extend](../../../../raw/postgres-12/src/backend/access/heap/visibilitymap.c#L625-L681)). So a missing side fork is normal for a small or never-vacuumed table. Truncation of the heap truncates both forks too ([storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L230-L293)), and the unlogged reset removes them ([reinit.c#ResetUnloggedRelations](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L36-L46)).

### Lifecycle of block zero

| Event | State before | State after | Effect on later reads |
|---|---|---|---|
| Table created | none | S1 | A scan reads no block. |
| First insert | S1 | file: S2 block written by extension; buffer: initialized page with the row | Readers see the buffer. The file catches up at the next write of that buffer. |
| Bulk extension under contention | S1 or more blocks | extra pages S2 in file and buffer, listed in the FSM | Inserts initialize a page when they pick it. |
| Crash before the first insert's WAL is flushed | file S2 | S2 stays | VACUUM records the page as free space. |
| Commit and checkpoint | buffer initialized | file matches | — |
| DELETE of all rows | tuples present | tuples present, deleted | Scans return no rows; the page is not empty yet. |
| VACUUM after every row died, truncation allowed and thresholds met | deleted tuples | S1 | Block zero is gone. |
| VACUUM, truncation skipped | deleted tuples | S4 | Later inserts reuse the unused line pointers. |
| TRUNCATE or rewrite with no rows | any | S1 in a new relfilenode | The old file becomes a D1 stub. |
| Crash, then recovery, unlogged table | any | S1 heap, S5 index | Committed rows are gone. |
| CIC build starts | S1 | B-tree: S1 during the sort, S2 once leaves are written, metapage at the end | Failure freezes whichever state was reached. |
| REINDEX of the invalid index | S1 or S2 | new relfilenode with a metapage | The index becomes valid. |

### Non-default settings that change the outcome

| Setting | Default | Non-default effect | Mechanism | How to change it |
|---|---|---|---|---|
| `vacuum_truncate` storage parameter | `true` ([reloptions.c#vacuum_truncate](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L152-L160)) | `false` turns A3 into A4 | Copied into the VACUUM truncate option ([vacuum.c#truncate-option](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1782-L1790)) | `ALTER TABLE ... SET (vacuum_truncate = false)`, which takes `ShareUpdateExclusiveLock` |
| `VACUUM (TRUNCATE false)` | option absent | the same, for one command | the same | Per command |
| `old_snapshot_threshold` | `-1` ([guc.c#old_snapshot_threshold](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3041-L3049)) | 0 or more turns A3 into A4 | `should_attempt_truncation()` returns false ([vacuumlazy.c:1860-1863](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1860-L1863)) | `PGC_POSTMASTER`: restart |
| `zero_damaged_pages` | `off` ([guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161)) | `on` gives B6 | `ReadBuffer_common()` zeroes instead of raising an error ([bufmgr.c#verify-or-zero](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924)) | `PGC_SUSET`: superuser `SET` in a session |
| `wal_level` | `replica` ([guc.c#wal_level](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4410-L4416)) | `minimal` enables the WAL skips of B7 | `XLogIsNeeded()` is false, so loaders add `TABLE_INSERT_SKIP_WAL` ([copy.c#wal-skip](../../../../raw/postgres-12/src/backend/commands/copy.c#L2766-L2774)) | `PGC_POSTMASTER`: restart |
| `UNLOGGED` | permanent tables | B3 | Init fork plus recovery reset ([heapam_handler.c#init-fork](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L616-L624)) | `CREATE UNLOGGED TABLE` or `ALTER TABLE ... SET UNLOGGED` |

This page suggests no setting change. The table only explains how each changed setting produces its case.

### How to tell the states apart

The query below lists every storage-bearing relation of one schema, plus its TOAST tables and TOAST indexes, with the state of block zero. It is the measurement script's `s5` stage, run on the fixtures; its output is under [Measured results](#measured-results).

- **Privileges:** `get_raw_page()` requires a superuser ([rawpage.c#superuser](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L103-L106)), and the [pageinspect](../../../glossary.md#pageinspect) extension must be installed.
- **Locks:** it takes `AccessShareLock` on each relation it reads ([rawpage.c#lock](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L109)), so it waits behind `TRUNCATE`, `VACUUM FULL` or `DROP`. The session `lock_timeout` stops such a wait.
- **What it sees:** the page as the buffer manager returns it, which is the shared-buffer copy when one exists. It reads one block per relation.
- **Exclusions:** temporary relations are skipped, because `get_raw_page()` refuses other sessions' temporary tables ([rawpage.c#other-temp](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L138-L146)).

```sql
-- Run in psql as a superuser: psql -X -v target_schema=sales -f inventory.sql
SET /* wiki_blk0_inventory */ statement_timeout = '60s';
SET /* wiki_blk0_inventory */ lock_timeout = '5s';
WITH /* wiki_blk0_inventory */ targets AS (
  SELECT c.oid FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE n.nspname = :'target_schema'
    AND c.relkind IN ('r', 'm', 'i', 'S', 't') AND c.relpersistence <> 't'
  UNION
  SELECT t.reltoastrelid FROM pg_class t JOIN pg_namespace n ON n.oid = t.relnamespace
  WHERE n.nspname = :'target_schema' AND t.reltoastrelid <> 0 AND t.relpersistence <> 't'
  UNION
  SELECT i.indexrelid FROM pg_index i JOIN pg_class t ON t.reltoastrelid = i.indrelid
  JOIN pg_namespace n ON n.oid = t.relnamespace
  WHERE n.nspname = :'target_schema' AND t.relpersistence <> 't'
), sized AS (
  SELECT c.oid, c.relname, c.relkind, c.relpersistence,
         pg_relation_size(c.oid, 'main') / current_setting('block_size')::int AS main_blocks
  FROM targets JOIN pg_class c ON c.oid = targets.oid
), inspected AS (
  SELECT s.*, CASE WHEN s.main_blocks > 0
                   THEN get_raw_page(s.oid::regclass::text, 'main', 0) END AS raw0
  FROM sized s
)
SELECT x.relname, x.relkind, x.relpersistence, x.main_blocks, ix.indisvalid,
       h.lsn, h.lower, h.upper,
       CASE WHEN x.main_blocks = 0 THEN 'S1 block 0 does not exist'
            WHEN h.upper = 0 THEN 'S2 all-zero page'
            WHEN h.lower <= 24 THEN 'S3 initialized, no line pointers'
            WHEN x.relkind IN ('r', 'm', 't')
                 AND (SELECT count(*) FROM heap_page_items(x.raw0)
                      WHERE lp_flags IN (1, 2)) = 0
              THEN 'S4 line pointers but no tuples'
            WHEN x.relkind = 'i' THEN 'initialized index page (S5 if the index is empty)'
            ELSE 'holds tuples' END AS block0_state
FROM inspected x
LEFT JOIN pg_index ix ON ix.indexrelid = x.oid
LEFT JOIN LATERAL page_header(x.raw0) h ON true
ORDER BY x.relkind, x.relname;
```

How the query decides, row by row:

1. `pg_relation_size()` gives the live length of the main fork. Zero blocks is S1, and `get_raw_page()` is then not called, because it would fail on a block that does not exist ([rawpage.c#out-of-range](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L148-L152)).
2. `page_header()` returns `upper = 0` only for an uninitialized page, which is S2.
3. `lower <= 24` means no line pointer, which is S3.
4. For a heap page, a line pointer with `lp_flags` 1 (normal) or 2 (redirect) means a stored tuple. None of them is S4. A deleted but not yet vacuumed tuple still counts as stored, so a table whose rows are merely deleted reads as "holds tuples".
5. For an index, the query does not decode the metapage. Use `bt_metap()` for a B-tree, or the access method's own pageinspect function, to tell S5 from a populated index. `indisvalid = false` together with S1 or S2 points to Family C.

The shared-buffer view in step 2 matters for B1 and C3. Those cases describe the file, and a running server may still hold an initialized buffer for the same block. The query cannot see that difference.

### Measured results

Measured on 2026-11-05 on an isolated PostgreSQL 12.2 server built from the pin, with 8 kB blocks and data checksums off; see [Last run](#last-run). The table names come from the script's disposable fixtures, which generate their own data.

| Scenario | Relation | Before | After |
|---|---|---|---|
| A1 created, never loaded | `customers`, `customer_import` (CTAS `WITH NO DATA`), `customer_summary` (matview `WITH NO DATA`), the TOAST table | — | 0 blocks |
| A1 TOAST index | TOAST index of `customers` | — | 1 block, B-tree metapage |
| A6 empty indexes | `customers_pkey`, GIN, SP-GiST, BRIN, hash | — | metapage at block zero; 1, 2, 3, 3 and 4 blocks |
| A6 empty GiST | `customers_city_gist` | — | 1 block, `lower 24`, no line pointers |
| A6 sequence | `invoice_number_seq` | — | 1 block holding 1 tuple |
| A2 TRUNCATE | `orders`, 20,000 rows | 109 blocks | 0 blocks, new relfilenode; `orders_pkey` 1 metapage block |
| A3 DELETE then VACUUM | `orders`, 20,000 rows | 109 blocks | 0 blocks; `orders_pkey` keeps 57 blocks |
| A4 `vacuum_truncate = false` | `order_archive` | 109 blocks | 109 blocks; block zero `lower 764`, `upper 8192`, 185 unused line pointers |
| A4 `VACUUM (TRUNCATE false)` | `order_audit` | 109 blocks | 109 blocks, same block zero |
| A4 last row kept | `order_events`, all rows but the last deleted | 109 blocks | 109 blocks, same block zero |
| A5 `VACUUM FULL` | `order_archive` with no live rows | 109 blocks | 0 blocks |
| A5 `REFRESH MATERIALIZED VIEW` | `order_event_totals`, source emptied | 1 block | 0 blocks |
| B3 unlogged reset | `cart_sessions`, 20,000 rows | 167 blocks, 57 index blocks | 0 rows, 0 blocks; index 1 metapage block, `root 0` |
| C1 unique CIC failure | `payments_id_uniq` | — | 0 blocks, `indisvalid` and `indisready` false |
| C1 CIC cancelled during leaf load | `payments_token_idx` | — | block zero all-zero (`lsn 0/0`, `upper 0`); block 1 an initialized leaf page; both flags false |
| C1 REINDEX | `payments_token_idx` | invalid | valid, metapage at block zero |

The cancelled build's block count depends on when the cancel arrived: 288 blocks after 10,441 loaded tuples in the full run and 449 blocks after 20,146 tuples in the re-run; see [Last run](#last-run). What did not vary is the state of block zero. The unlogged relations' pages carried `lsn 0/0`, because unlogged changes are not WAL-logged.

The inventory query on the same fixtures classified every relation the same way as the per-stage checks. It reported the empty GiST root as S3, `order_audit` and `order_events` as S4, `payments_id_uniq` as invalid with S1, and every heap and TOAST table listed in the table above as S1.

### Tests in the pinned tree

- `src/test/recovery/t/014_unlogged_reinit.pl` checks that the init and main forks exist and that recovery restores the main fork from the init fork ([014_unlogged_reinit.pl#header](../../../../raw/postgres-12/src/test/recovery/t/014_unlogged_reinit.pl#L1-L27)).
- `src/test/recovery/t/015_promotion_pages.pl` checks replay of records for pages beyond the minimum consistent point after promotion, which exercises the invalid-page bookkeeping beside B4 ([015_promotion_pages.pl#header](../../../../raw/postgres-12/src/test/recovery/t/015_promotion_pages.pl#L1-L4)).
- `src/test/regress/sql/vacuum.sql` runs `VACUUM (TRUNCATE FALSE)` on a table with `vacuum_truncate` set ([vacuum.sql#truncate-option](../../../../raw/postgres-12/src/test/regress/sql/vacuum.sql#L113-L121)).
- No test in the pinned tree checks block zero after a cancelled `CREATE INDEX CONCURRENTLY`, after bulk extension, or after a crash during an index build.

### Final causal summary

- **Ordinary emptying:** a table is created, truncated, rewritten with no rows, or vacuumed after every row died → the heap's main fork has no block → block zero does not exist (S1). Its indexes are rebuilt or built empty → block zero is a metapage of an empty index (S5).
- **VACUUM without truncation:** the rows die and VACUUM runs, but a setting, a live tail row, a size threshold or a lock blocks truncation → the line pointer array keeps its length → block zero exists with only unused line pointers (S4).
- **Extension without content:** a backend extends the heap → an all-zero page reaches the file at once while the content stays in a buffer → bulk-extended pages stay all-zero until used, and a crash before WAL reaches disk leaves the page all-zero (S2) → VACUUM accepts it as free space.
- **Unlogged reset:** a crash or a base-backup start triggers recovery → the init fork replaces the main fork → the heap has no block zero and each index has an empty-index metapage.
- **Interrupted concurrent B-tree build:** the empty index file commits first → the build fails during the sort (S1) or after writing leaves (S2, because the metapage comes last) → the index stays invalid until `REINDEX` or `DROP INDEX`.
- **Schema-wide emptiness:** none of these paths selects by schema → an entire schema with empty block zeros means every object went through one of these paths, or an outside process changed the files.

## Measurement Script

### Usage

| Item | Value |
|---|---|
| Purpose | Measures the state of block zero after each scenario in Families A, B3 and C1, and runs the inventory query. It backs every number under [Measured results](#measured-results) and the measured claims in the Family sections. |
| Invocation | Save the script below as `.wiki-runtime/tmp/blk0.sh`, then run `bash .wiki-runtime/tmp/blk0.sh [stage ...]` from the root of this wiki repository. With no stage it runs the default order. |
| Stages | Default order `build cluster s1 s2 s3 s4 s5 stop`. `build` configures, builds and installs the pinned checkout out of tree with default `configure` options, plus `contrib/pageinspect`; it is skipped when the install records the pin. `cluster` runs `initdb` without data checksums, starts the server, creates database `shop` and the `blk0_state()` helper. `s1` creates relations that are never loaded. `s2` runs TRUNCATE, DELETE plus VACUUM, the three truncation-skip cases, VACUUM FULL and REFRESH. `s3` loads an unlogged table and restarts after an immediate shutdown. `s4` runs a failing unique CIC and a CIC cancelled during its leaf load. `s5` runs the inventory query. `stop` stops the server, runs the teardown checks and writes `summary.txt`. `clean`, not in the default order, stops the server, runs the checks and deletes the sandbox. Each SQL stage drops and recreates its own fixtures, so it can be re-run alone. |
| Environment | `WIKI_ROOT` (`$PWD`), `SRC` (`$WIKI_ROOT/raw/postgres-12`), `SANDBOX` (`$WIKI_ROOT/.wiki-runtime/tmp/blk0`, which must lie under `.wiki-runtime/tmp`), `PORT` (`55471`), `JOBS` (`8`). The script clears `PGSERVICE`, `PGSERVICEFILE`, `PGHOSTADDR`, `PGOPTIONS`, `PGTZ`, `PGDATESTYLE` and `PGCLIENTENCODING`, and exports `PGHOST`, `PGPORT`, `PGUSER=postgres`, `PGDATABASE=shop` and `PGAPPNAME=wiki_blk0`. |
| Prerequisites | `bash`, `git`, a C toolchain, `make`, `flex`, `bison`, `perl`, readline and zlib headers, and the POSIX `ps`, `grep -F` and `kill -0` that the teardown checks use. About 1.4 GB of disk. The pinned checkout must be at the pin with no changed tracked files. |
| Output | `$SANDBOX/out/summary.txt`, read first: platform facts, then every stage's statements and results. Per-stage files `s1.txt` to `s5.txt`, `server.log`, and the build logs sit beside it. |
| Runtime | About 1 minute from an empty sandbox on the last-run host, most of it the build. A re-run of `cluster s1 s2 s3 s4 s5 stop` with the build present takes about 8 seconds. |
| Cleanup | `bash .wiki-runtime/tmp/blk0.sh clean` stops the server and deletes `$SANDBOX`, refusing a directory without the script's `.blk0-sandbox` mark; then delete `.wiki-runtime/tmp/blk0.sh`. Every invocation also stops the server on exit and runs the POSIX teardown checks: no `postmaster.pid`, no process in a `ps -A -o pid= -o args=` listing that names the data directory, `kill -0` failing for the remembered postmaster PID, no `.s.PGSQL.55471` socket or lock file, and `pg_isready` returning 2. The server never listens on TCP, so no `127.0.0.1` check is needed. |

To extract the script from this page without retyping it, run this from the repository root. It builds the Markdown fence at run time:

```bash
fence=$(printf '\140\140\140')
sed -n '/^### The script/,$p' wiki/v12/questions/storage-and-vacuum/empty-block-zero-scenarios.md \
  | sed -n "/^${fence}bash\$/,/^${fence}\$/p" | sed '1d;$d' > .wiki-runtime/tmp/blk0.sh
```

### Last run

| Item | Value |
|---|---|
| Date | 2026-11-05. A full run from an empty sandbox ran from 14:42:10Z to 14:43:09Z (59 seconds). A re-run of `cluster s1 s2 s3 s4 s5 stop` with the build present ran from 14:43:27Z to 14:43:35Z (8 seconds). |
| Script | The text filed on this page, MD5 `f983cca01b1f837920d9abb053f382d2` as extracted with the command above. Two earlier runs the same day used a text without the inline tags on `SET`, `RESET`, `CHECKPOINT` and `ANALYZE` and without stage `s5`; every deterministic value they reported matches the filed run. |
| Server | PostgreSQL 12.2 built from pin `45b88269a353ad93744772791feb6d01bc7e1e42` with default `configure` options, compiled by gcc 13.3.0 (Ubuntu 13.3.0-6ubuntu2~24.04.1). |
| Platform | Linux 6.18.54.1-microsoft-standard-WSL2, x86_64. `pg_controldata`: block size 8192, 131072 blocks per segment, maximum data alignment 8, data page checksum version 0. |
| Disk | 1,336,476 kB for the sandbox at the end of the full run, mostly the 3,000,001-row `payments` fixture and its rebuilt index. |
| Timing-dependent values | The cancelled build was stopped after 10,441 loaded tuples with 288 index blocks in the full run, and after 20,146 tuples with 449 blocks in the re-run. The two earlier texts gave 17,314 tuples with 423 blocks and 6,178 tuples with 211 blocks. Block zero was all-zero in all four. |
| Teardown | Every invocation's `stop` stage and exit trap found no `postmaster.pid`, no process naming the data directory, no surviving postmaster PID, no socket or lock file, and `pg_isready` status 2. The `clean` stage then deleted the sandbox, and the extracted script was deleted. |

### Non-default settings in the measurement

| Setting or state | Value in the script | Default | Context and scope | Why the script needs it | Results that depend on it |
|---|---|---|---|---|---|
| `port`, `unix_socket_directories`, `listen_addresses` | `55471`, `$SANDBOX/sock`, empty, on the `pg_ctl` command line | 5432, the configure-time socket directory, `localhost` | `PGC_POSTMASTER`: set at server start ([guc.c:2177](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2177), [guc.c:3936](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3936), [guc.c:3951](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3951)) | Isolate the cluster | None |
| `autovacuum` | `off` on the command line | `on` | `PGC_SIGHUP`: reload ([guc.c:1426](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1426)) | Keeps background VACUUM from truncating or vacuuming a fixture between two steps | None of the measured end states: every VACUUM the results depend on is explicit. With autovacuum on, a fixture could reach its end state earlier, and the "after DELETE" rows could already show the VACUUM result. |
| `statement_timeout`, `lock_timeout` | `5min` and `10s` through `PGOPTIONS`; the inventory also sets `60s` and `5s` | `0`, no limit | `PGC_USERSET`: session ([guc.c:2378](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2378), [guc.c:2389](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2389)) | Stop a stuck statement | None |
| `vacuum_truncate = false` on `order_archive`, `VACUUM (TRUNCATE false)` on `order_audit` | fixture state | `true`, option absent | storage parameter; command option | They are the A4 cases being measured | The two A4 rows; the default produced the A3 row |
| `UNLOGGED` on `cart_sessions` | fixture state | permanent | table property | It is the B3 case | The B3 row |
| Immediate shutdown | `pg_ctl -m immediate stop` in `s3` | fast or smart shutdown with a shutdown checkpoint | server control | Forces recovery at the next start, the B3 trigger ([xlog.c#recovery-gate](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6728-L6749)) | The B3 row. A clean shutdown does not reset unlogged tables. |
| Duplicate `payment_id` 1, and a cancel during the leaf load | fixture state, `pg_cancel_backend()` | — | — | They are the two C1 failure points | The two C1 rows |
| Data checksums | off, the `initdb` default | off | fixed at `initdb` | — | None: `PageIsVerified()` accepts all-zero pages either way |

### The script

```bash
#!/usr/bin/env bash
# blk0.sh - which operations leave block 0 of a relation missing, all-zero,
# empty, or a metapage of an empty index, in PostgreSQL 12.2.
#
# Usage, from the root of the wiki repository:
#   bash .wiki-runtime/tmp/blk0.sh [stage ...]
# Stages, default order: build cluster s1 s2 s3 s4 s5 stop
#   build    configure, build and install the pinned checkout out of tree,
#            plus contrib/pageinspect; skipped when the install records the pin
#   cluster  initdb (no data checksums) and start an isolated server
#   s1       relations created and never loaded, and block 0 of empty indexes
#   s2       TRUNCATE, DELETE plus VACUUM, vacuum_truncate off, VACUUM FULL,
#            REFRESH MATERIALIZED VIEW
#   s3       unlogged table and index across an immediate shutdown
#   s4       CREATE INDEX CONCURRENTLY failing in the sort, and cancelled
#            while it loads leaf pages
#   s5       read-only block 0 inventory of the fixtures' schema
#   stop     stop the server and run the teardown checks
#   clean    stop, check, and delete the sandbox
# Every fixture is disposable: it lives in a throw-away cluster.
# Results: $SANDBOX/out/<stage>.txt; read summary.txt first.
# Environment: WIKI_ROOT ($PWD), SRC ($WIKI_ROOT/raw/postgres-12),
#   SANDBOX ($WIKI_ROOT/.wiki-runtime/tmp/blk0), PORT (55471), JOBS (8).
set -uo pipefail

WIKI_ROOT="${WIKI_ROOT:-$PWD}"
SRC="${SRC:-$WIKI_ROOT/raw/postgres-12}"
SANDBOX="${SANDBOX:-$WIKI_ROOT/.wiki-runtime/tmp/blk0}"
PORT="${PORT:-55471}"
JOBS="${JOBS:-8}"
PIN=45b88269a353ad93744772791feb6d01bc7e1e42
MARKER=.blk0-sandbox

say()  { printf '\n== %s\n' "$*" >&2; }
note() { printf '   %s\n' "$*" >&2; }
die()  { printf '!! %s\n' "$*" >&2; exit 1; }

[ -f "$WIKI_ROOT/AGENTS.md" ] && [ -d "$WIKI_ROOT/raw" ] \
  || die "WIKI_ROOT=$WIKI_ROOT is not the wiki repository"
case "$SANDBOX" in "$WIKI_ROOT"/.wiki-runtime/tmp/?*) ;; *) die "SANDBOX must be under .wiki-runtime/tmp" ;; esac
case "$SANDBOX" in *..*) die "SANDBOX must not contain .." ;; esac

BUILD="$SANDBOX/build"; INST="$SANDBOX/install"; BIN="$INST/bin"
DATA="$SANDBOX/data"; SOCK="$SANDBOX/sock"; OUT="$SANDBOX/out"
unset PGSERVICE PGSERVICEFILE PGHOSTADDR PGOPTIONS PGTZ PGDATESTYLE PGCLIENTENCODING
export PGHOST="$SOCK" PGPORT="$PORT" PGUSER=postgres PGDATABASE=shop PGAPPNAME=wiki_blk0
# Session scope only (both GUCs are PGC_USERSET): applied at connection start.
SESSION_OPTS="-c statement_timeout=5min -c lock_timeout=10s"

pg()  { PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -v ON_ERROR_STOP=1 "$@" || die "psql failed: $*"; }
# sql <stage>: run SQL from stdin, echo every statement, append to the stage file.
sql() { PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -q -e -v ON_ERROR_STOP=1 -f - >> "$OUT/$1.txt" 2>&1 \
          || die "psql failed in stage $1, see $OUT/$1.txt"; }
# sqlfail <stage>: the statement on stdin must fail with psql status 3.
sqlfail() { PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -q -e -v ON_ERROR_STOP=1 -f - >> "$OUT/$1.txt" 2>&1
            [ $? -eq 3 ] || die "a statement that must fail did not, see $OUT/$1.txt"; }

running() { [ -x "$BIN/pg_ctl" ] && "$BIN/pg_ctl" -D "$DATA" status >/dev/null 2>&1; }
PM_PID=""
remember_pid() {
  if [ -f "$DATA/postmaster.pid" ]; then
    PM_PID=$(head -n 1 "$DATA/postmaster.pid")
    case "$PM_PID" in ''|*[!0-9]*) PM_PID="" ;; esac
  fi
}
start_server() {
  running && return 0
  # Non-default, command line, postmaster context: an own port and socket
  # directory and no TCP listener isolate the cluster; autovacuum=off keeps
  # background VACUUM from truncating or rewriting fixtures between steps.
  "$BIN/pg_ctl" -D "$DATA" -l "$OUT/server.log" -w \
    -o "-p $PORT -k $SOCK -c listen_addresses= -c autovacuum=off" start >/dev/null \
    || die "server did not start, see $OUT/server.log"
  remember_pid
}
stop_server() {
  remember_pid
  running && "$BIN/pg_ctl" -D "$DATA" -m fast -w stop >/dev/null 2>&1
  return 0
}
# teardown_problems: POSIX checks; prints problems, returns 0 if any found.
teardown_problems() {
  local found=1 listing="$SANDBOX/ps.txt" pid args rc
  running && { echo "pg_ctl still reports a server"; found=0; }
  [ -f "$DATA/postmaster.pid" ] && { echo "postmaster.pid is still present"; found=0; }
  if ps -A -o pid= -o args= > "$listing" 2>/dev/null; then
    grep -F -- "$DATA" "$listing" > "$listing.hits"
    while read -r pid args; do
      [ "$pid" = "$$" ] && continue
      case "$args" in sh|sh\ *|bash|bash\ *|*/sh\ *|*/bash\ *) continue ;; esac
      echo "process $pid still names $DATA: $args"; found=0
    done < "$listing.hits"
    rm -f "$listing" "$listing.hits"
  else
    echo "ps failed"; found=0
  fi
  if [ -n "$PM_PID" ] && kill -0 "$PM_PID" 2>/dev/null; then echo "postmaster $PM_PID still exists"; found=0; fi
  [ -e "$SOCK/.s.PGSQL.$PORT" ] && { echo "socket still present"; found=0; }
  [ -e "$SOCK/.s.PGSQL.$PORT.lock" ] && { echo "socket lock file still present"; found=0; }
  if [ -x "$BIN/pg_isready" ]; then
    "$BIN/pg_isready" -q -h "$SOCK" -p "$PORT"; rc=$?
    [ "$rc" -eq 2 ] || { echo "pg_isready returned $rc, not 2"; found=0; }
  fi
  return $found
}
on_exit() {
  local st=$? problems
  trap - EXIT
  if [ -f "$SANDBOX/$MARKER" ]; then
    stop_server
    if problems=$(teardown_problems); then
      printf '!! teardown incomplete:\n%s\n' "$problems" >&2; [ "$st" -ne 0 ] || st=1
    fi
  fi
  exit "$st"
}
trap on_exit EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

claim() {
  if [ -e "$SANDBOX" ] && [ ! -f "$SANDBOX/$MARKER" ] && [ -n "$(ls -A "$SANDBOX")" ]; then
    die "refusing $SANDBOX: not created by this script"
  fi
  mkdir -p "$SANDBOX" "$OUT" "$SOCK" && : > "$SANDBOX/$MARKER" || die "cannot create $SANDBOX"
  export TMPDIR="$SANDBOX"
}

stage_build() {
  say "build"
  [ "$(git -C "$SRC" rev-parse HEAD)" = "$PIN" ] || die "$SRC is not at $PIN"
  [ -z "$(git -C "$SRC" --no-optional-locks status --porcelain --untracked-files=no)" ] \
    || die "$SRC has changed tracked files"
  if [ "$(cat "$INST/.pin" 2>/dev/null)" = "$PIN" ]; then note "already built"; return 0; fi
  rm -rf "$BUILD" "$INST"; mkdir -p "$BUILD"
  ( cd "$BUILD" && "$SRC/configure" --prefix="$INST" > "$OUT/configure.log" 2>&1 \
      && make -s -j "$JOBS" > "$OUT/make.log" 2>&1 \
      && make -s install > "$OUT/install.log" 2>&1 \
      && make -s -C contrib/pageinspect install >> "$OUT/install.log" 2>&1 ) \
    || die "build failed, see $OUT/*.log"
  echo "$PIN" > "$INST/.pin"
}

stage_cluster() {
  say "cluster"
  [ "$(cat "$INST/.pin" 2>/dev/null)" = "$PIN" ] || die "run the build stage first"
  stop_server; rm -rf "$DATA"; rm -f "$OUT"/s?.txt "$OUT/summary.txt"
  # Default initdb: no -k, so data checksums are off.
  "$BIN/initdb" -D "$DATA" -U postgres -A trust -E UTF8 --no-locale > "$OUT/initdb.log" 2>&1 \
    || die "initdb failed, see $OUT/initdb.log"
  start_server
  pg -d postgres -c "CREATE DATABASE shop" >/dev/null
  pg -c "CREATE EXTENSION pageinspect" >/dev/null
  # blk0_state: describe block 0 of the main fork of one relation.
  sql cluster <<'SQL'
CREATE /* wiki_blk0_helper */ FUNCTION blk0_state(rel regclass) RETURNS text
LANGUAGE plpgsql AS $$
DECLARE nblocks bigint; h record; raw bytea;
BEGIN
  nblocks := pg_relation_size(rel, 'main') / current_setting('block_size')::int;
  IF nblocks = 0 THEN RETURN 'S1 no block 0 (main fork has 0 blocks)'; END IF;
  raw := get_raw_page(rel::text, 'main', 0);
  SELECT * INTO h FROM page_header(raw);
  IF h.upper = 0 THEN
    RETURN format('S2 all-zero/new page (lsn %s, lower %s, upper %s), %s blocks',
                  h.lsn, h.lower, h.upper, nblocks);
  END IF;
  RETURN format('initialized: lsn %s, lower %s, upper %s, special %s, %s blocks',
                h.lsn, h.lower, h.upper, h.special, nblocks);
END $$;
SQL
}

stage_s1() {
  say "s1: created, never loaded; empty indexes and sequences"
  rm -f "$OUT/s1.txt"
  sql s1 <<'SQL'
SET /* wiki_blk0_s1 */ client_min_messages = warning;
DROP /* wiki_blk0_s1 */ TABLE IF EXISTS customers, customer_import CASCADE;
DROP /* wiki_blk0_s1 */ MATERIALIZED VIEW IF EXISTS customer_summary;
DROP /* wiki_blk0_s1 */ SEQUENCE IF EXISTS invoice_number_seq;
RESET /* wiki_blk0_s1 */ client_min_messages;
CREATE /* wiki_blk0_s1 */ TABLE customers (customer_id int PRIMARY KEY, full_name text,
  city_point point, tags text[], region text, signup_day int);
CREATE /* wiki_blk0_s1 */ INDEX customers_city_gist ON customers USING gist (city_point);
CREATE /* wiki_blk0_s1 */ INDEX customers_tags_gin ON customers USING gin (tags);
CREATE /* wiki_blk0_s1 */ INDEX customers_city_spgist ON customers USING spgist (city_point);
CREATE /* wiki_blk0_s1 */ INDEX customers_signup_brin ON customers USING brin (signup_day);
CREATE /* wiki_blk0_s1 */ INDEX customers_region_hash ON customers USING hash (region);
CREATE /* wiki_blk0_s1 */ TABLE customer_import AS SELECT * FROM customers WITH NO DATA;
CREATE /* wiki_blk0_s1 */ MATERIALIZED VIEW customer_summary AS
  SELECT region, count(*) AS customer_count FROM customers GROUP BY region WITH NO DATA;
CREATE /* wiki_blk0_s1 */ SEQUENCE invoice_number_seq;
SELECT /* wiki_blk0_s1 */ c.relname, c.relkind, c.relpersistence, blk0_state(c.oid)
FROM pg_class c
WHERE c.relname IN ('customers','customers_pkey','customers_city_gist','customers_tags_gin',
                    'customers_city_spgist','customers_signup_brin','customers_region_hash',
                    'customer_import','customer_summary','invoice_number_seq')
   OR c.oid IN (SELECT reltoastrelid FROM pg_class WHERE relname = 'customers')
   OR c.oid IN (SELECT i.indexrelid FROM pg_index i JOIN pg_class t ON t.reltoastrelid = i.indrelid
                WHERE t.relname = 'customers')
ORDER BY c.relkind, c.relname;
SELECT /* wiki_blk0_s1 */ magic, version, root, level FROM bt_metap('customers_pkey');
SELECT /* wiki_blk0_s1 */ count(*) AS seq_block0_tuples FROM heap_page_items(get_raw_page('invoice_number_seq', 0));
SQL
}

stage_s2() {
  say "s2: cleared after load"
  rm -f "$OUT/s2.txt"
  sql s2 <<'SQL'
SET /* wiki_blk0_s2 */ client_min_messages = warning;
DROP /* wiki_blk0_s2 */ TABLE IF EXISTS orders, order_archive, order_audit, order_events CASCADE;
DROP /* wiki_blk0_s2 */ MATERIALIZED VIEW IF EXISTS order_event_totals;
RESET /* wiki_blk0_s2 */ client_min_messages;
-- A2: TRUNCATE replaces the heap file and rebuilds the index
CREATE /* wiki_blk0_s2 */ TABLE orders (order_id int PRIMARY KEY, status text);
INSERT /* wiki_blk0_s2 */ INTO orders SELECT g, 'shipped' FROM generate_series(1, 20000) g;
SELECT /* wiki_blk0_s2 */ 'orders before TRUNCATE' AS step, pg_relation_filenode('orders') AS filenode, blk0_state('orders');
TRUNCATE /* wiki_blk0_s2 */ orders;
SELECT /* wiki_blk0_s2 */ 'orders after TRUNCATE' AS step, pg_relation_filenode('orders') AS filenode, blk0_state('orders');
SELECT /* wiki_blk0_s2 */ 'orders_pkey after TRUNCATE' AS step, blk0_state('orders_pkey');
-- A3: DELETE everything, then VACUUM truncates the heap to 0 blocks
INSERT /* wiki_blk0_s2 */ INTO orders SELECT g, 'shipped' FROM generate_series(1, 20000) g;
DELETE /* wiki_blk0_s2 */ FROM orders;
SELECT /* wiki_blk0_s2 */ 'orders after DELETE' AS step, blk0_state('orders');
VACUUM /* wiki_blk0_s2 */ orders;
SELECT /* wiki_blk0_s2 */ 'orders after VACUUM' AS step, blk0_state('orders');
SELECT /* wiki_blk0_s2 */ 'orders_pkey after VACUUM' AS step, blk0_state('orders_pkey');
-- A4: vacuum_truncate = false keeps every block; block 0 keeps unused line pointers
CREATE /* wiki_blk0_s2 */ TABLE order_archive (order_id int, status text) WITH (vacuum_truncate = false);
INSERT /* wiki_blk0_s2 */ INTO order_archive SELECT g, 'archived' FROM generate_series(1, 20000) g;
DELETE /* wiki_blk0_s2 */ FROM order_archive;
VACUUM /* wiki_blk0_s2 */ order_archive;
SELECT /* wiki_blk0_s2 */ 'order_archive after VACUUM, vacuum_truncate=false' AS step, blk0_state('order_archive');
SELECT /* wiki_blk0_s2 */ lp_flags, count(*) FROM heap_page_items(get_raw_page('order_archive', 0)) GROUP BY lp_flags;
-- A4: the VACUUM (TRUNCATE false) option has the same effect for one run
CREATE /* wiki_blk0_s2 */ TABLE order_audit (order_id int, status text);
INSERT /* wiki_blk0_s2 */ INTO order_audit SELECT g, 'audited' FROM generate_series(1, 20000) g;
DELETE /* wiki_blk0_s2 */ FROM order_audit;
VACUUM /* wiki_blk0_s2 */ (TRUNCATE false) order_audit;
SELECT /* wiki_blk0_s2 */ 'order_audit after VACUUM (TRUNCATE false)' AS step, blk0_state('order_audit');
-- A4: a live row on the last block stops truncation; block 0 stays, with no tuples
CREATE /* wiki_blk0_s2 */ TABLE order_events (event_id int, note text);
INSERT /* wiki_blk0_s2 */ INTO order_events SELECT g, 'created' FROM generate_series(1, 20000) g;
DELETE /* wiki_blk0_s2 */ FROM order_events WHERE event_id < 20000;
VACUUM /* wiki_blk0_s2 */ order_events;
SELECT /* wiki_blk0_s2 */ 'order_events after VACUUM, last row kept' AS step, blk0_state('order_events');
SELECT /* wiki_blk0_s2 */ lp_flags, count(*) FROM heap_page_items(get_raw_page('order_events', 0)) GROUP BY lp_flags;
-- A5: rewrites of a relation with no live rows write no block
VACUUM /* wiki_blk0_s2 */ FULL order_archive;
SELECT /* wiki_blk0_s2 */ 'order_archive after VACUUM FULL' AS step, blk0_state('order_archive');
CREATE /* wiki_blk0_s2 */ MATERIALIZED VIEW order_event_totals AS
  SELECT note, count(*) AS event_count FROM order_events GROUP BY note;
SELECT /* wiki_blk0_s2 */ 'order_event_totals populated' AS step, blk0_state('order_event_totals');
DELETE /* wiki_blk0_s2 */ FROM order_events;
REFRESH /* wiki_blk0_s2 */ MATERIALIZED VIEW order_event_totals;
SELECT /* wiki_blk0_s2 */ 'order_event_totals refreshed with zero rows' AS step, blk0_state('order_event_totals');
SQL
}

stage_s3() {
  say "s3: unlogged relations across an immediate shutdown"
  rm -f "$OUT/s3.txt"
  start_server
  sql s3 <<'SQL'
DROP /* wiki_blk0_s3 */ TABLE IF EXISTS cart_sessions;
CREATE /* wiki_blk0_s3 */ UNLOGGED TABLE cart_sessions (session_id int PRIMARY KEY, cart jsonb);
INSERT /* wiki_blk0_s3 */ INTO cart_sessions SELECT g, '{"items": 3}' FROM generate_series(1, 20000) g;
CHECKPOINT /* wiki_blk0_s3 */;
SELECT /* wiki_blk0_s3 */ 'before immediate shutdown' AS step, count(*) AS rows_visible FROM cart_sessions;
SELECT /* wiki_blk0_s3 */ relname, blk0_state(oid) FROM pg_class
WHERE relname IN ('cart_sessions', 'cart_sessions_pkey') ORDER BY relname;
SQL
  remember_pid
  # Immediate shutdown: no shutdown checkpoint, so the next start runs crash recovery.
  "$BIN/pg_ctl" -D "$DATA" -m immediate -w stop >/dev/null || die "immediate stop failed"
  echo "-- pg_ctl -m immediate stop; pg_ctl start (crash recovery)" >> "$OUT/s3.txt"
  start_server
  grep -E 'database system was not properly shut down|redo starts|redo done' "$OUT/server.log" \
    | tail -n 3 | sed 's/^[^]]*] //' >> "$OUT/s3.txt"
  sql s3 <<'SQL'
SELECT /* wiki_blk0_s3 */ 'after crash recovery' AS step, count(*) AS rows_visible FROM cart_sessions;
SELECT /* wiki_blk0_s3 */ relname, blk0_state(oid) FROM pg_class
WHERE relname IN ('cart_sessions', 'cart_sessions_pkey') ORDER BY relname;
SELECT /* wiki_blk0_s3 */ magic, root, level FROM bt_metap('cart_sessions_pkey');
SQL
}

stage_s4() {
  say "s4: CREATE INDEX CONCURRENTLY that fails or is cancelled"
  rm -f "$OUT/s4.txt"
  start_server
  sql s4 <<'SQL'
SET /* wiki_blk0_s4 */ client_min_messages = warning;
DROP /* wiki_blk0_s4 */ TABLE IF EXISTS payments;
RESET /* wiki_blk0_s4 */ client_min_messages;
CREATE /* wiki_blk0_s4 */ TABLE payments (payment_id int, card_token text);
INSERT /* wiki_blk0_s4 */ INTO payments
  SELECT g, md5(g::text) || md5((g * 7)::text) FROM generate_series(1, 3000000) g;
INSERT /* wiki_blk0_s4 */ INTO payments VALUES (1, 'duplicate payment id');
ANALYZE /* wiki_blk0_s4 */ payments;
SQL
  # C1a: a unique build fails inside the sort, before any index page is written.
  sqlfail s4 <<'SQL'
CREATE /* wiki_blk0_s4 */ UNIQUE INDEX CONCURRENTLY payments_id_uniq ON payments (payment_id);
SQL
  sql s4 <<'SQL'
SELECT /* wiki_blk0_s4 */ 'payments_id_uniq after failed CIC' AS step, i.indisvalid, i.indisready,
       blk0_state('payments_id_uniq')
FROM pg_index i WHERE i.indexrelid = 'payments_id_uniq'::regclass;
SQL
  # C1b: cancel the build while it writes leaf pages (best effort, timing-dependent).
  PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -q -v ON_ERROR_STOP=1 \
    -c "CREATE /* wiki_blk0_s4 */ INDEX CONCURRENTLY payments_token_idx ON payments (card_token)" \
    > "$OUT/s4-cic.txt" 2>&1 &
  local bg=$! tries=0 hit=""
  while [ $tries -lt 3000 ]; do
    hit=$(pg -At -c "SELECT /* wiki_blk0_s4 */ pid || ' ' || tuples_done FROM pg_stat_progress_create_index
                     WHERE phase LIKE '%loading tuples%' AND tuples_done > 0")
    [ -n "$hit" ] && break
    kill -0 "$bg" 2>/dev/null || break
    tries=$((tries + 1))
  done
  if [ -n "$hit" ]; then
    pg -At -c "SELECT /* wiki_blk0_s4 */ pg_cancel_backend(${hit%% *})" >/dev/null
    echo "-- cancelled the CIC backend during leaf load, after ${hit#* } tuples" >> "$OUT/s4.txt"
  else
    echo "-- leaf-load phase not observed; nothing cancelled" >> "$OUT/s4.txt"
  fi
  wait "$bg"
  echo "-- CIC session exit status $? and output:" >> "$OUT/s4.txt"
  cat "$OUT/s4-cic.txt" >> "$OUT/s4.txt"
  sql s4 <<'SQL'
SELECT /* wiki_blk0_s4 */ 'payments_token_idx after cancelled CIC' AS step, i.indisvalid, i.indisready,
       blk0_state('payments_token_idx')
FROM pg_index i WHERE i.indexrelid = 'payments_token_idx'::regclass;
SELECT /* wiki_blk0_s4 */ 'block 1' AS block, lsn, lower, upper, special
FROM page_header(get_raw_page('payments_token_idx', 1));
REINDEX /* wiki_blk0_s4 */ INDEX payments_token_idx;
SELECT /* wiki_blk0_s4 */ 'payments_token_idx after REINDEX' AS step, i.indisvalid, i.indisready,
       blk0_state('payments_token_idx')
FROM pg_index i WHERE i.indexrelid = 'payments_token_idx'::regclass;
SQL
}

stage_s5() {
  say "s5: read-only block 0 inventory of one schema"
  rm -f "$OUT/s5.txt"
  PGOPTIONS="$SESSION_OPTS" "$BIN/psql" -X -q -e -v ON_ERROR_STOP=1 -v target_schema=public \
    -f - >> "$OUT/s5.txt" 2>&1 <<'SQL' || die "psql failed in stage s5, see $OUT/s5.txt"
SET /* wiki_blk0_inventory */ statement_timeout = '60s';
SET /* wiki_blk0_inventory */ lock_timeout = '5s';
WITH /* wiki_blk0_inventory */ targets AS (
  SELECT c.oid FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE n.nspname = :'target_schema'
    AND c.relkind IN ('r', 'm', 'i', 'S', 't') AND c.relpersistence <> 't'
  UNION
  SELECT t.reltoastrelid FROM pg_class t JOIN pg_namespace n ON n.oid = t.relnamespace
  WHERE n.nspname = :'target_schema' AND t.reltoastrelid <> 0 AND t.relpersistence <> 't'
  UNION
  SELECT i.indexrelid FROM pg_index i JOIN pg_class t ON t.reltoastrelid = i.indrelid
  JOIN pg_namespace n ON n.oid = t.relnamespace
  WHERE n.nspname = :'target_schema' AND t.relpersistence <> 't'
), sized AS (
  SELECT c.oid, c.relname, c.relkind, c.relpersistence,
         pg_relation_size(c.oid, 'main') / current_setting('block_size')::int AS main_blocks
  FROM targets JOIN pg_class c ON c.oid = targets.oid
), inspected AS (
  SELECT s.*, CASE WHEN s.main_blocks > 0
                   THEN get_raw_page(s.oid::regclass::text, 'main', 0) END AS raw0
  FROM sized s
)
SELECT x.relname, x.relkind, x.relpersistence, x.main_blocks, ix.indisvalid,
       h.lsn, h.lower, h.upper,
       CASE WHEN x.main_blocks = 0 THEN 'S1 block 0 does not exist'
            WHEN h.upper = 0 THEN 'S2 all-zero page'
            WHEN h.lower <= 24 THEN 'S3 initialized, no line pointers'
            WHEN x.relkind IN ('r', 'm', 't')
                 AND (SELECT count(*) FROM heap_page_items(x.raw0)
                      WHERE lp_flags IN (1, 2)) = 0
              THEN 'S4 line pointers but no tuples'
            WHEN x.relkind = 'i' THEN 'initialized index page (S5 if the index is empty)'
            ELSE 'holds tuples' END AS block0_state
FROM inspected x
LEFT JOIN pg_index ix ON ix.indexrelid = x.oid
LEFT JOIN LATERAL page_header(x.raw0) h ON true
ORDER BY x.relkind, x.relname;
SQL
}

stage_stop() {
  say "stop"
  stop_server
  local problems
  if problems=$(teardown_problems); then die "teardown incomplete: $problems"; fi
  note "server stopped; teardown checks passed"
  {
    echo "platform: $(uname -sm)"
    "$BIN/postgres" --version
    pg_controldata_out=$("$BIN/pg_controldata" -D "$DATA")
    printf '%s\n' "$pg_controldata_out" | grep -E 'Database block size|Blocks per segment|Maximum data alignment|Data page checksum'
    for s in cluster s1 s2 s3 s4 s5; do [ -f "$OUT/$s.txt" ] && { echo "==== $s"; cat "$OUT/$s.txt"; }; done
  } > "$OUT/summary.txt"
}

stage_clean() {
  say "clean"
  [ -d "$SANDBOX" ] || { note "no sandbox"; return 0; }
  [ -f "$SANDBOX/$MARKER" ] || die "refusing $SANDBOX: not created by this script"
  stop_server
  local problems
  if problems=$(teardown_problems); then die "teardown incomplete: $problems"; fi
  rm -rf "$SANDBOX" || die "cannot delete $SANDBOX"
  note "deleted $SANDBOX"
}

[ $# -gt 0 ] || set -- build cluster s1 s2 s3 s4 s5 stop
for s in "$@"; do
  case "$s" in build|cluster|s1|s2|s3|s4|s5|stop|clean) ;; *) die "unknown stage $s" ;; esac
done
for s in "$@"; do
  if [ "$s" = clean ]; then stage_clean; continue; fi
  claim
  case "$s" in
    build|cluster|stop) "stage_$s" ;;
    *) [ "$(cat "$INST/.pin" 2>/dev/null)" = "$PIN" ] || die "run build first"
       start_server; "stage_$s" ;;
  esac
done
```

## Context Reviewed

- Navigation: `wiki/versions.md`, `wiki/index.md`, `wiki/v12/index.md`, the last 20 entries of `wiki/log.md`, and the glossary entries for page, block, fork, all-zero page, unlogged relation, truncation, TRUNCATE, free space map, visibility map, metapage, relation segment, relfilenumber, storage manager, invalid index, line pointer and WAL level. v12 has no common concept pages.
- Related v12 pages, read for consistency: the empty relation files page, the zeroed relation file page, the zeroed last segment checker, the DROP SCHEMA page, the post-incident health page, the invalid-index outcomes page, and the CIC and REINDEX CONCURRENTLY pages.
- Page layout and verification: `bufpage.h`, `bufpage.c` (`PageInit`, `PageIsVerified`, `PageRepairFragmentation`).
- Storage creation, truncation and removal: `heap.c`, `pg_class.h`, `storage.c`, `heapam_handler.c`, `md.c`, `tablecmds.c`, `rewriteheap.c`, `createas.c`, `matview.c`, `sequence.c`, `dbsize.c`.
- Extension and first insert: `hio.c`, `bufmgr.c`, `heapam.c` (insert and redo), and commit `0d1fe9f74e3`.
- VACUUM: `vacuumlazy.c` (new and empty pages, truncation decision, lock retry, tail recheck), `vacuum.c`, `reloptions.c`, `pruneheap.c`.
- Recovery and unlogged relations: `xlog.c`, `xlogutils.c`, `reinit.c`, `basebackup.c`, `storage.sgml`, and every access method's `buildempty` function.
- Index builds: `nbtsort.c`, `nbtpage.c`, `nbtree.c`, `gistbuild.c`, `gist.c`, `gininsert.c`, `spginsert.c`, `brin.c`, `hash.c`, `hashpage.c`, `tuplesort.c`, `indexcmds.c`, `index.c`, `plancat.c`, `verify_nbtree.c`, `system_views.sql` (progress view phases).
- Settings and docs: `guc.c`, `config.sgml`, `pg_resetwal.sgml`, `copy.c`, `filemap.c`, `freespace.c`, `visibilitymap.c`, `pageinspect--1.5.sql`, `rawpage.c`.
- Tests: `014_unlogged_reinit.pl`, `015_promotion_pages.pl`, `vacuum.sql`.

## Evidence Map

| Claim | Evidence |
|---|---|
| `PageIsEmpty` and `PageIsNew` definitions | [bufpage.h#PageIsEmpty-PageIsNew](../../../../raw/postgres-12/src/include/storage/bufpage.h#L213-L229) |
| All-zero pages pass verification | [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L150) |
| Which relkinds have storage | [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L183-L192), [heap.c#create-storage](../../../../raw/postgres-12/src/backend/catalog/heap.c#L368-L443) |
| New storage has no blocks and is deleted at abort | [storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L70-L122), [storage.c#smgrDoPendingDeletes](../../../../raw/postgres-12/src/backend/catalog/storage.c#L399-L446) |
| TRUNCATE gives new storage or truncates in place | [tablecmds.c#ExecuteTruncateGuts](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1840), [heapam_handler.c#nontransactional_truncate](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L630-L633) |
| VACUUM truncation conditions and execution | [vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1851-L1867), [vacuumlazy.c#lock-retry](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1899-L1922), [vacuumlazy.c#truncate](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1947-L1992), [vacuum.c#truncate-option](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1782-L1790) |
| VACUUM keeps the line pointer array length | [vacuumlazy.c#lazy_vacuum_page](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1604-L1619), [bufpage.c#PageRepairFragmentation](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L516-L569) |
| Rewrites write no page without tuples | [rewriteheap.c#end_heap_rewrite](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L313-L345), [matview.c#refresh](../../../../raw/postgres-12/src/backend/commands/matview.c#L296-L340) |
| Block zero of each index access method | [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1127-L1137), [gistbuild.c#root-init](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L173-L185), [gininsert.c#meta-and-root](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L339-L352), [spginsert.c#meta-and-roots](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L88-L109), [brin.c#metapage](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L684-L695), [hashpage.c#_hash_init](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L351-L400) |
| Sequence block zero holds its tuple | [sequence.c#fill_seq_with_data](../../../../raw/postgres-12/src/backend/commands/sequence.c#L343-L360) |
| Extension writes a zero page; init happens in the buffer | [bufmgr.c#extend](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L866-L879), [hio.c#single-page-extension](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L594-L609), [heapam.c#init-page-flag](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1955-L1965) |
| Bulk extension leaves pages uninitialized | [hio.c#RelationAddExtraBlocks](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L186-L261), commit `0d1fe9f74e3` |
| VACUUM treats all-zero pages as free space | [vacuumlazy.c#new-page-handling](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L885-L928) |
| Unlogged reset | [xlog.c#unlogged-cleanup](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L6878-L6884), [xlog.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7323-L7331), [reinit.c#init-copy](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L274-L345), [heapam_handler.c#init-fork](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L616-L624), [basebackup.c#unlogged-exclusion](../../../../raw/postgres-12/src/backend/replication/basebackup.c#L1123-L1150) |
| Recovery extends files with zero pages | [xlogutils.c#XLogReadBufferExtended](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L437-L520), [md.c#mdread-short-read](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L606-L630) |
| `zero_damaged_pages` behavior and default | [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161), [bufmgr.c#verify-or-zero](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L907-L924), [config.sgml#zero_damaged_pages](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L9745-L9768) |
| CIC commits storage before building | [indexcmds.c#first-commit](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1316-L1320), [indexcmds.c#build-transaction](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1366-L1379), [index.c#index_concurrently_create_copy](../../../../raw/postgres-12/src/backend/catalog/index.c#L1362-L1380) |
| The B-tree build pads block zero and writes the metapage last | [nbtsort.c#reserve-metapage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L575-L589), [nbtsort.c#_bt_blwritepage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L652-L700), [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L1127-L1137) |
| Invalid index handling | [plancat.c#invalid-index](../../../../raw/postgres-12/src/backend/optimizer/util/plancat.c#L200-L210), [verify_nbtree.c#invalid](../../../../raw/postgres-12/contrib/amcheck/verify_nbtree.c#L332-L337), [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L692-L708) |
| Abort records list created files; a crash writes none | [xact.c#abort-record-files](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L1650-L1652), [xact.c#xact_redo_abort](../../../../raw/postgres-12/src/backend/access/transam/xact.c#L5925-L5926) |
| Deferred removal of the first segment | [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L323), [md.c#register_unlink_segment](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L916-L930) |
| Measured states | [Measurement Script](#measurement-script), run on 2026-11-05 |

## Open Questions

- **WAL-skipped loads under `wal_level = minimal`.** The `COPY` comment says the same-transaction test behind the WAL skip "is not always set correctly" after a `TRUNCATE`, `SAVEPOINT`, `TRUNCATE`, `ROLLBACK TO` sequence ([copy.c#wal-skip](../../../../raw/postgres-12/src/backend/commands/copy.c#L2729-L2741)). This page does not establish whether that case, or another WAL-skip path at the 12.2 pin, can leave a committed block zero all-zero after a crash. No test in the pinned tree covers it, and no measurement ran with `wal_level = minimal`.
- **Crash-timing scenarios were not measured.** B1, B2, B4, B5 and C3 are derived from the source only. B2 needs concurrent inserters waiting on the extension lock, and B1, B4 and C3 need a crash at a chosen point.
- **The cancelled-build leg is timing-dependent.** The script cancels the build once the progress view shows the leaf-load phase with at least one tuple loaded. On a faster or slower host the cancel could arrive after the metapage was written, and the leg would then report an initialized block zero. All four runs on the last-run host reached the all-zero state.
- **The incident, if there is one, is not established.** No affected database, file listing or log was supplied. Outside causes in E1 cannot be identified from source.
- **The inventory reads buffers, not files.** For B1 and C3, where the file and a buffer can differ, the query reports the buffer. A file-level check needs the server stopped or a tool outside SQL.
- **Missing concept layer.** PostgreSQL 12 has no common concept page for page layout or for the relation-file lifecycle. A separately requested page on either would let this page and its siblings link instead of repeating the explanation. None was created here.

## Source References

- [bufpage.h#PageIsEmpty-PageIsNew](../../../../raw/postgres-12/src/include/storage/bufpage.h#L213-L229)
- [bufpage.c#PageIsVerified](../../../../raw/postgres-12/src/backend/storage/page/bufpage.c#L82-L150)
- [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L183-L192)
- [heap.c#create-storage](../../../../raw/postgres-12/src/backend/catalog/heap.c#L368-L443)
- [storage.c#RelationCreateStorage](../../../../raw/postgres-12/src/backend/catalog/storage.c#L70-L122)
- [heapam_handler.c#heapam_relation_set_new_filenode](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L580-L627)
- [tablecmds.c#ExecuteTruncateGuts](../../../../raw/postgres-12/src/backend/commands/tablecmds.c#L1786-L1840)
- [vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1851-L1867)
- [vacuum.c#truncate-option](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1782-L1790)
- [reloptions.c#vacuum_truncate](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L152-L160)
- [rewriteheap.c#end_heap_rewrite](../../../../raw/postgres-12/src/backend/access/heap/rewriteheap.c#L313-L345)
- [matview.c#refresh](../../../../raw/postgres-12/src/backend/commands/matview.c#L296-L340)
- [createas.c#skipData](../../../../raw/postgres-12/src/backend/commands/createas.c#L298-L307)
- [sequence.c#fill_seq_with_data](../../../../raw/postgres-12/src/backend/commands/sequence.c#L343-L360)
- [hio.c#RelationAddExtraBlocks](../../../../raw/postgres-12/src/backend/access/heap/hio.c#L186-L261)
- [bufmgr.c#extend](../../../../raw/postgres-12/src/backend/storage/buffer/bufmgr.c#L866-L879)
- [heapam.c#init-page-flag](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L1955-L1965)
- [xlog.c#unlogged-init](../../../../raw/postgres-12/src/backend/access/transam/xlog.c#L7323-L7331)
- [reinit.c#init-copy](../../../../raw/postgres-12/src/backend/storage/file/reinit.c#L274-L345)
- [xlogutils.c#XLogReadBufferExtended](../../../../raw/postgres-12/src/backend/access/transam/xlogutils.c#L437-L520)
- [md.c#mdunlinkfork](../../../../raw/postgres-12/src/backend/storage/smgr/md.c#L277-L323)
- [nbtsort.c#_bt_blwritepage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtsort.c#L652-L700)
- [nbtpage.c#_bt_checkpage](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L692-L708)
- [gistbuild.c#root-init](../../../../raw/postgres-12/src/backend/access/gist/gistbuild.c#L173-L185)
- [gininsert.c#meta-and-root](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L339-L352)
- [spginsert.c#meta-and-roots](../../../../raw/postgres-12/src/backend/access/spgist/spginsert.c#L88-L109)
- [brin.c#metapage](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L684-L695)
- [hashpage.c#_hash_init](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L351-L400)
- [indexcmds.c#build-transaction](../../../../raw/postgres-12/src/backend/commands/indexcmds.c#L1366-L1379)
- [index.c#index_concurrently_create_copy](../../../../raw/postgres-12/src/backend/catalog/index.c#L1362-L1380)
- [tuplesort.c#unique-violation](../../../../raw/postgres-12/src/backend/utils/sort/tuplesort.c#L4031-L4056)
- [guc.c#zero_damaged_pages](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1150-L1161)
- [copy.c#wal-skip](../../../../raw/postgres-12/src/backend/commands/copy.c#L2718-L2774)
- [rawpage.c#get_raw_page_internal](../../../../raw/postgres-12/contrib/pageinspect/rawpage.c#L95-L172)
- [config.sgml#zero_damaged_pages](../../../../raw/postgres-12/doc/src/sgml/config.sgml#L9745-L9768)
- [014_unlogged_reinit.pl#header](../../../../raw/postgres-12/src/test/recovery/t/014_unlogged_reinit.pl#L1-L27)

## Navigation

- [PostgreSQL 12 index](../../index.md)
- [Wiki index](../../../index.md)
- [Wiki Glossary (unverified)](../../../glossary.md)
- [Version pins](../../../versions.md)
- [Failure Scenarios That Can Leave Schema Relation Files Empty (unverified)](schema-empty-relation-files-failure-scenarios.md)
- [A Manually Zeroed Relation File (unverified)](zeroed-relation-file.md)
- [Checking Every Table for a Zeroed Last Segment (unverified)](zeroed-last-segment-check-all-tables.md)
- [DROP SCHEMA After Relation Files Were Emptied (unverified)](drop-schema-zeroed-relation-files.md)
- [All Outcomes That Leave an Invalid Index (unverified)](../indexing/invalid-index-outcomes.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented (unverified)](../indexing/create-index-concurrently.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented (unverified)](../indexing/reindex-index-concurrently.md)
