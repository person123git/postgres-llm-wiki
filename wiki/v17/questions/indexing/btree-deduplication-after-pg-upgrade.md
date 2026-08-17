---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Checking Whether an Index Needs a Rebuild to Enable Deduplication After pg_upgrade From PostgreSQL 12 to 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Why every carried-over index is affected](#why-every-carried-over-index-is-affected)
  - [The one byte that decides it](#the-one-byte-that-decides-it)
  - [Check 1: read the flag with core SQL](#check-1-read-the-flag-with-core-sql)
  - [Privileges the probe needs](#privileges-the-probe-needs)
  - [Check 2: prove the byte offsets on your platform](#check-2-prove-the-byte-offsets-on-your-platform)
  - [Check 3: would a rebuild even help?](#check-3-would-a-rebuild-even-help)
  - [What the gate rejects, and why](#what-the-gate-rejects-and-why)
  - [The deduplicate_items trap](#the-deduplicate_items-trap)
  - [Unique indexes get the flag but no size win](#unique-indexes-get-the-flag-but-no-size-win)
  - [What a rebuild actually recovered](#what-a-rebuild-actually-recovered)
  - [How much duplication is worth a rebuild](#how-much-duplication-is-worth-a-rebuild)
  - [The cost of leaving it alone](#the-cost-of-leaving-it-alone)
  - [Separating deduplication from ordinary bloat](#separating-deduplication-from-ordinary-bloat)
  - [What sets the flag, and what does not](#what-sets-the-flag-and-what-does-not)
  - [Rebuild order and the statistics pg_upgrade leaves behind](#rebuild-order-and-the-statistics-pg_upgrade-leaves-behind)
  - [Second opinion: make the engine say it](#second-opinion-make-the-engine-say-it)
  - [Fallbacks when you cannot read server files](#fallbacks-when-you-cannot-read-server-files)
  - [Edge cases the check has to survive](#edge-cases-the-check-has-to-survive)
  - [Where this comes from in the source history](#where-this-comes-from-in-the-source-history)
  - [Settings this page touches](#settings-this-page-touches)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, question: after a database is upgraded from v12 to v17, how do I check if an index needs to be rebuilt to enable deduplication?

> Prompt note: filed as an approved grammar-corrected restatement of "in postgreql
> 17 , question: after an database is upgraded from v12 to v17, how to check if an
> index needs to be rebuild to enable de-duplication.", per the repository's
> prompt-hygiene rule. The asker chose the corrected wording, asked for the answer
> to be measured on a real `pg_upgrade` run rather than derived from source alone,
> and restricted the check itself to core SQL, with no extensions installed.

## Answer

### Short answer

After a binary upgrade with `pg_upgrade`, **every** B-tree index that came from the
PostgreSQL 12 cluster is unable to deduplicate, and no amount of `VACUUM` will
change that. Deduplication safety is recorded once, in the index's metapage field
`btm_allequalimage`, at build time; PostgreSQL 12 never wrote that field, and
`pg_upgrade` copies the index file unchanged, so the field reads false forever until
the index is rebuilt
([nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L124-L147),
[nbtpage.c#_bt_metaversion](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L719-L791)).

So the per-index check has two parts, and core SQL can do both:

1. **Is this index deduplicating right now?** Read byte 64 of block 0 of the index's
   main fork. That is `btm_allequalimage`. Core SQL reads it with
   `pg_read_binary_file()` plus `get_byte()`; no `pageinspect` needed. See
   [Check 1](#check-1-read-the-flag-with-core-sql).
2. **Would a rebuild help?** Mirror `_bt_allequalimage()` in the catalogs: no
   `INCLUDE` columns, every key opclass has a `pg_amproc` equal-image support
   function at `amprocnum = 4`, and no key collation is nondeterministic. See
   [Check 3](#check-3-would-a-rebuild-even-help).

An index needs a rebuild exactly when part 1 says false and part 2 says true. In the
clean 12.2 → 17.10 run the check flagged 9 of 15 carried-over indexes. Rebuilding
all 15 anyway reclaimed 67.4% to 69.1% on the seven duplicate-heavy indexes it
flagged, nothing on the unique and the empty index it flagged (which its own verdict
text predicts for unique indexes), and nothing at all on five of the six it refused;
the sixth, an `INCLUDE` index, lost six pages for reasons unrelated to
deduplication.

Do not check `btm_version` for this. PostgreSQL 12 already writes version 4, so
version is 4 both before and after the rebuild
([nbtree.h#BTREE_VERSION](../../../../raw/postgres-17/src/include/access/nbtree.h#L148-L152));
all 21 carried-over indexes reported version 4 while none of them could
deduplicate.

### Why every carried-over index is affected

The v17 source says so in two places, and a third in contrib:

- `BTMetaPageData`'s header comment: "Even version 4 indexes created on PostgreSQL
  v12 will need a REINDEX to make use of deduplication, though, since there is no
  other way to set btm_allequalimage to true (pg_upgrade hasn't been taught to set
  the metapage field)."
  ([nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L124-L147))
- `_bt_metaversion()`: "We rely on the assumption that btm_allequalimage will be
  zero'ed on heapkeyspace indexes that were pg_upgrade'd from Postgres 12."
  ([nbtpage.c#_bt_metaversion](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L719-L791))
- `bt_metap()` repeats the same assumption
  ([btreefuncs.c#bt_metap](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L905-L922)).

`pg_upgrade` moves the index file without touching its contents. It selects every
valid index of every user table (plus TOAST heaps and `pg_largeobject`)
([info.c#get_rel_infos](../../../../raw/postgres-17/src/bin/pg_upgrade/info.c#L471-L524)),
has `pg_dump --binary-upgrade` preserve each index's OID and relfilenode
([pg_dump.c#binary_upgrade_set_next_index_relfilenode](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L5563-L5572)),
and then copies or links the file
([relfilenumber.c#transfer_relfile](../../../../raw/postgres-17/src/bin/pg_upgrade/relfilenumber.c#L168-L266)).
Measured on the pin: after `pg_upgrade --copy` from 12.2 to 17.10, all 21 user
B-tree index files had identical MD5s in the old and new data directories, and
`pg_class.oid` and `pg_class.relfilenode` were unchanged for every one of them.

Two consequences that surprise people:

- **`pg_upgrade` never mentions it.** The run produced exactly one script,
  `delete_old_cluster.sh`, and its completion banner talks only about statistics
  ([check.c#output_completion_banner](../../../../raw/postgres-17/src/bin/pg_upgrade/check.c#L760-L790)).
  Grepping the full output for "reindex", "rebuild" or "deduplicat" found nothing.
  The only index-rebuild help it still offers is for pre-10 hash indexes, gated on
  `GET_MAJOR_VERSION(old_cluster.major_version) <= 906`
  ([check.c#issue_warnings_and_set_wal_level](../../../../raw/postgres-17/src/bin/pg_upgrade/check.c#L744-L757)),
  which a 12 → 17 upgrade never reaches.
- **This is a `pg_upgrade`-only problem.** A dump/restore upgrade issues ordinary
  `CREATE INDEX` statements, which build v17 metapages. It is the relfilenode
  preservation above that carries the old physical index across.

### The one byte that decides it

The metapage is block 0. Its contents start at `MAXALIGN(SizeOfPageHeaderData)`
([bufpage.h#PageGetContents](../../../../raw/postgres-17/src/include/storage/bufpage.h#L246-L258)),
and `BTMetaPageData` lays the fields out in this order
([nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119)).
Compiling `offsetof()` against the pinned 17.10 build's own installed headers gives:

| field | offset in struct | absolute byte in block 0 |
|---|---|---|
| `btm_magic` | 0 | 24 |
| `btm_version` | 4 | 28 |
| `btm_root` | 8 | 32 |
| `btm_level` | 12 | 36 |
| `btm_fastroot` | 16 | 40 |
| `btm_fastlevel` | 20 | 44 |
| `btm_last_cleanup_num_delpages` | 24 | 48 |
| `btm_last_cleanup_num_heap_tuples` | 32 | 56 |
| `btm_allequalimage` | 40 | **64** |

`MAXIMUM_ALIGNOF` was 8, `SizeOfPageHeaderData` 24, and `sizeof(BTMetaPageData)`
48, so a v17-built metapage sets `pd_lower` to 72
([nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L63-L96)).
The same program compiled against the 12.2 build's headers fails: that struct has
neither `btm_allequalimage` nor `btm_last_cleanup_num_delpages`. That is why the
byte reads zero — `_bt_pageinit()` zeroes the page and PostgreSQL 12 wrote nothing
there. Measured side effect: every carried-over metapage reported `pd_lower = 64`,
every 17-built one reported 72.

Who writes the byte:

- `_bt_initmetapage()` stores whatever `_bt_allequalimage()` returned at build time,
  for `CREATE INDEX`/`REINDEX`
  ([nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L531-L565),
  [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1118-L1128))
  and for an empty index
  ([nbtree.c#btbuildempty](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L158-L171)).
- `_bt_upgrademetapage()`, the only in-place metapage version bump, explicitly
  refuses to set it: "Only a REINDEX can set this field"
  ([nbtpage.c#_bt_upgrademetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L98-L131)).
- WAL replay copies the value out of the record rather than recomputing it
  ([nbtxlog.c#_bt_restore_meta](../../../../raw/postgres-17/src/backend/access/nbtree/nbtxlog.c#L101-L125)).
  Note that replay *does* rewrite `pd_lower` to 72, so `pd_lower` is a hint, not
  the answer; read byte 64.

Who reads it: `_bt_metaversion()` loads it into the insertion scan key
([nbtutils.c#_bt_mkscankey](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L118-L160)),
and both deduplication paths gate on it — the index build
([nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1144-L1152))
and the insert-time pass that tries to avoid a page split
([nbtinsert.c#_bt_delete_or_dedup_one_page](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2774-L2782)).
So a carried-over index does not deduplicate on insert either: it just splits.

### Check 1: read the flag with core SQL

`pg_relation_filepath()` returns a path relative to the data directory
([dbsize.c#pg_relation_filepath](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L948-L1028)),
`pg_read_binary_file()` reads it, and `get_byte()` picks the byte out of the
`bytea`
([varlena.c#byteaGetByte](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L3246-L3268)).
Reading 72 bytes is enough and is block-size independent.

```sql
SET statement_timeout = '60s';
SET lock_timeout = '5s';

SELECT /* wiki_dedup_rebuild_check */
       n.nspname,
       c.relname,
       pg_size_pretty(pg_relation_size(c.oid))        AS index_size,
       m.allequalimage                                AS deduplicating_now,
       CASE
         WHEN m.allequalimage IS NULL
           THEN 'unknown: could not read metapage'
         WHEN m.allequalimage
           THEN 'no rebuild needed'
         WHEN i.indnatts <> i.indnkeyatts
           THEN 'no gain: INCLUDE index can never deduplicate'
         WHEN NOT e.equalimage_ok
           THEN 'no gain: key type or collation is not deduplication-safe'
         WHEN NOT coalesce(o.deduplicate_items, true)
           THEN 'no gain while deduplicate_items = off'
         WHEN i.indisunique
           THEN 'rebuild: unique index, no immediate size win'
         ELSE 'rebuild: REINDEX INDEX CONCURRENTLY enables deduplication'
       END                                            AS verdict
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
JOIN pg_am a ON a.oid = c.relam
JOIN pg_index i ON i.indexrelid = c.oid
CROSS JOIN LATERAL (
       SELECT CASE WHEN octet_length(z.meta) = 72
                   THEN get_byte(z.meta, 64) <> 0 END AS allequalimage
       FROM (SELECT pg_read_binary_file(pg_relation_filepath(c.oid), 0, 72, true) AS meta) z
     ) m
CROSS JOIN LATERAL (
       SELECT NOT EXISTS (
                SELECT 1
                FROM generate_series(0, i.indnkeyatts - 1) AS k
                LEFT JOIN pg_opclass oc ON oc.oid = i.indclass[k]
                LEFT JOIN pg_amproc ap ON ap.amprocfamily = oc.opcfamily
                                      AND ap.amproclefttype = oc.opcintype
                                      AND ap.amprocrighttype = oc.opcintype
                                      AND ap.amprocnum = 4
                LEFT JOIN pg_collation col ON col.oid = i.indcollation[k]
                WHERE ap.amproc IS NULL
                   OR (ap.amproc = 'btvarstrequalimage'::regproc
                       AND NOT col.collisdeterministic)
              ) AS equalimage_ok
     ) e
LEFT JOIN LATERAL (
       SELECT lower(split_part(opt, '=', 2))::bool AS deduplicate_items
       FROM unnest(c.reloptions) AS opt
       WHERE split_part(opt, '=', 1) = 'deduplicate_items'
     ) o ON true
WHERE a.amname = 'btree'
  AND c.relkind = 'i'
  AND c.relpersistence <> 't'
ORDER BY (m.allequalimage IS NOT TRUE) DESC, pg_relation_size(c.oid) DESC;
```

On the upgraded 17.10 cluster this produced, for the 21 carried-over indexes:

| verdict | indexes |
|---|---|
| `rebuild: REINDEX INDEX CONCURRENTLY enables deduplication` | `i_dup10`, `i_dup1000`, `i_txt`, `i_txt_pattern`, `i_expr`, `i_multi`, `i_partial`, `i_uuid`, `i_churn`, `i_ts`, `i_u_k`, `i_empty`, `p_main_1_k_idx`, `p_main_2_k_idx` |
| `rebuild: unique index, no immediate size win` | `i_uniq`, plus carried-over `pg_toast_16385_index` and `pg_largeobject_loid_pn_index` |
| `no gain: key type or collation is not deduplication-safe` | `i_num` (numeric), `i_flt` (float8), `i_js` (jsonb), `i_arr` (int[]), `i_multimixed` (int + numeric) |
| `no gain: INCLUDE index can never deduplicate` | `i_inc` |

Every index built after the upgrade reported `no rebuild needed`, as did 122 of the
124 `pg_catalog` B-tree indexes, which the new cluster's `initdb` created rather
than `pg_upgrade` transferring them. Cross-checked against `pageinspect`'s
`bt_metap()` — installed only as ground truth, not as part of the check — the
probe's `version` and `allequalimage` agreed in all 57 pre-rebuild rows across the
three upgraded clusters and all 18 post-rebuild rows.

### Privileges the probe needs

Measured on the pin, and consistent with
([genfile.c#convert_and_check_filename](../../../../raw/postgres-17/src/backend/utils/adt/genfile.c#L41-L92)):

- `pg_read_binary_file` is revoked from `PUBLIC`
  ([system_functions.sql:712-718](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L712-L718)),
  so a plain role gets `ERROR:  permission denied for function pg_read_binary_file`.
- **`GRANT pg_read_server_files` is not sufficient and not needed.** Granting only
  that role left the same permission error, because the predefined role widens
  *which paths* are legal, not *who may call the function*.
- What worked was `GRANT EXECUTE ON FUNCTION
  pg_read_binary_file(text,bigint,bigint,boolean) TO <role>`. The grant is
  per-signature: with only the four-argument form granted, the three-argument form
  still failed.
- With that grant and no `pg_read_server_files`, both the relative path
  `base/16384/16391` and an absolute path inside the data directory read fine,
  while `/etc/hostname` failed with `ERROR:  absolute path not allowed`. So the
  probe stays inside the data directory by construction.
- `pg_relation_filepath()` needed no grant at all; a plain role could call it.

### Check 2: prove the byte offsets on your platform

The offsets above are `MAXIMUM_ALIGNOF = 8` offsets. Before trusting the probe on a
platform this page did not measure, make the server answer a question you already
know the answer to: a freshly built index must report true.

```sql
CREATE TABLE wiki_dedup_selftest (k int);
INSERT INTO wiki_dedup_selftest VALUES (1), (1);
CREATE INDEX wiki_dedup_selftest_k ON wiki_dedup_selftest (k);

SELECT /* wiki_dedup_probe_selftest */
       octet_length(meta)                              AS meta_bytes,
       (get_byte(meta, 13) * 256) + get_byte(meta, 12) AS pd_lower,
       (get_byte(meta, 31)::bigint << 24) + (get_byte(meta, 30) << 16)
         + (get_byte(meta, 29) << 8) + get_byte(meta, 28) AS btm_version,
       get_byte(meta, 64) <> 0                         AS allequalimage
FROM (SELECT pg_read_binary_file(
               pg_relation_filepath('wiki_dedup_selftest_k'), 0, 72, true) AS meta) z;

DROP TABLE wiki_dedup_selftest;
```

On the pin this returned `72 | 72 | 4 | t`. If `btm_version` is not 4 or
`allequalimage` is not true here, the offsets do not apply to that build and the
probe's answers are meaningless. No checkpoint is needed first: an index build writes
its pages, metapage included, through the bulk-write path rather than shared buffers
([nbtsort.c#_bt_blwritepage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L631-L639)),
so the fresh metapage was already on disk when the probe read it.

### Check 3: would a rebuild even help?

`_bt_allequalimage()` is the whole rule
([nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)):

1. If the index has `INCLUDE` columns, return false immediately.
2. Otherwise, for each key column, look up support function 4
   (`BTEQUALIMAGE_PROC`,
   [nbtree.h#BTEQUALIMAGE_PROC](../../../../raw/postgres-17/src/include/access/nbtree.h#L686-L712))
   in the column's opfamily for its `opcintype`, and call it with the column's
   collation. Missing function, or a false answer, means false.

The `equalimage_ok` subquery in [Check 1](#check-1-read-the-flag-with-core-sql) is
that rule in SQL. Two support functions exist:
`btequalimage` returns true unconditionally
([datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L415-L438)),
and `btvarstrequalimage` returns false for a nondeterministic collation
([varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)),
which is why the SQL applies the `collisdeterministic` test only to columns whose
support function is `btvarstrequalimage`.

The gate was scored against the engine's own verdict, the `DEBUG1` message
`_bt_allequalimage()` emits during a build, over 19 indexes on two servers:

| fixture | key(s) | engine said | gate said |
|---|---|---|---|
| `i_dup10`, `i_dup1000`, `i_multi`, `i_partial`, `i_churn`, `i_empty` | `int4` | can safely use deduplication | true |
| `i_txt`, `i_expr` | `text` (`text_ops`, C collation) | can safely use deduplication | true |
| `i_txt_pattern` | `text` (`text_pattern_ops`) | can safely use deduplication | true |
| `i_uuid` | `uuid` | can safely use deduplication | true |
| `i_uniq` | `int4`, unique | can safely use deduplication | true |
| `i_num`, `i_multimixed` | `numeric` key | cannot use deduplication | false |
| `i_flt` | `float8` | cannot use deduplication | false |
| `i_js` | `jsonb` | cannot use deduplication | false |
| `i_arr` | `int[]` | cannot use deduplication | false |
| `i_inc` | `int4` `INCLUDE (text)` | *(silent)* | false |
| `i_nd`, `i_nd_off` | `text` with nondeterministic ICU collation | cannot use deduplication | false |
| `i_det_icu`, `i_c` | `text` with deterministic ICU / C collation | can safely use deduplication | true |

The `INCLUDE` index is silent because `_bt_allequalimage()` returns before it
reaches the `elog(DEBUG1, ...)` block. Treat "no message" as "cannot deduplicate".

### What the gate rejects, and why

The documented list of unsafe cases
([btree.sgml#btree-deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L834-L909))
matches what the catalogs encode:

- `numeric` (display scale), `jsonb` (numeric internally), `float4`/`float8`
  (`-0` versus `0`) and container types such as arrays, composites and ranges have
  no `amprocnum = 4` row at all.
- `text`, `varchar`, `char` and `name` register `btvarstrequalimage`
  ([pg_amproc.dat:201-212](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L201-L212)),
  so a nondeterministic collation disqualifies them; ordinary scalar types register
  `btequalimage`.
- `INCLUDE` indexes are refused outright, regardless of key types.

One edge worth knowing: the pattern opclasses register `btequalimage`, which would
ignore a nondeterministic collation — but such an index cannot be created in the
first place
([index.c#index_create](../../../../raw/postgres-17/src/backend/catalog/index.c#L827-L848)).
Measured: `CREATE INDEX ... (a text_pattern_ops)` on a nondeterministic-collation
column failed with `ERROR:  nondeterministic collations are not supported for
operator class "text_pattern_ops"`, so the simpler and the proc-aware forms of the
collation test cannot disagree in practice.

### The deduplicate_items trap

The metapage flag and the `deduplicate_items` storage parameter are independent.
The build path needs both plus non-uniqueness
([nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1144-L1152)),
and so does the insert path
([nbtinsert.c#_bt_delete_or_dedup_one_page](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2774-L2782)),
but only `_bt_allequalimage()` feeds the metapage. Measured consequence: after
`ALTER INDEX i_dup1000 SET (deduplicate_items = off)` and a `REINDEX INDEX
CONCURRENTLY`, the engine still logged "can safely use deduplication", byte 64 read
true — and the index came out at its full 22,519,808 bytes instead of 7,340,032.
That is why [Check 1](#check-1-read-the-flag-with-core-sql) reads `reloptions` as
well as the metapage. The parameter defaults to on
([create_index.sgml#deduplicate_items](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L448-L476)),
takes `ShareUpdateExclusiveLock`, and changing it does not convert existing tuples
([reloptions.c:159-168](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)).

### Unique indexes get the flag but no size win

`_bt_load()` skips deduplication for unique indexes, so rebuilding a carried-over
unique index changes its size by nothing: `i_uniq` measured 22,487,040 bytes before
and after, with byte 64 flipping from false to true. The value of the rebuild is
later: a unique index with the flag set can run insert-time deduplication to absorb
version churn and delay page splits
([btree.sgml#btree-deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L810-L833),
[nbtinsert.c#_bt_findinsertloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L899-L907)).
Rank those rebuilds below the duplicate-heavy non-unique ones.

### What a rebuild actually recovered

`REINDEX INDEX CONCURRENTLY`, one index at a time, on the freshly upgraded cluster
(fixtures built once on 12.2 and never updated, so there was no bloat to confuse
the numbers):

| index | keys | before | after | change |
|---|---|---|---|---|
| `i_dup10` | `k10` (10 distinct) | 22,519,808 | 6,963,200 | −69.1% |
| `i_txt` | `txt` (100 distinct) | 22,519,808 | 6,979,584 | −69.0% |
| `i_expr` | `lower(txt)` | 22,519,808 | 6,979,584 | −69.0% |
| `i_partial` | `k10 WHERE st = 'open'` | 2,277,376 | 712,704 | −68.7% |
| `i_churn` | `k` (20 distinct), after 200,000 extra inserts | 8,904,704 | 2,801,664 | −68.5% |
| `i_dup1000` | `k1000` (1000 distinct) | 22,519,808 | 7,340,032 | −67.4% |
| `i_multi` | `(k10, k1000)` | 22,519,808 | 7,340,032 | −67.4% |
| `i_uniq` | `id` unique | 22,487,040 | 22,487,040 | 0.0% |
| `i_empty` | empty table | 8,192 | 8,192 | 0.0% |
| `i_num`, `i_flt`, `i_js`, `i_arr`, `i_multimixed` | not equal-image | unchanged | unchanged | 0.0% |
| `i_inc` | `INCLUDE` | 31,612,928 | 31,563,776 | −0.16% |

The five not-equal-image indexes came out identical to the byte, which is the
strongest argument for running the gate: rebuilding them costs a full index build
and buys nothing. `i_inc` lost six pages, which is a v12-versus-v17 build
difference, not deduplication — its byte 64 stayed false.

### How much duplication is worth a rebuild

Fresh 1,000,000-row single-column builds on 17.10, each built twice, once with
`deduplicate_items = off` and once with the default, so the difference is
deduplication alone:

| rows per key | no-deduplication bytes | deduplicated bytes | saved |
|---|---|---|---|
| 1 | 22,487,040 | 22,487,040 | 0.0% |
| 2 | 22,519,808 | 20,275,200 | 10.0% |
| 5 | 22,511,616 | 11,681,792 | 48.1% |
| 20 | 22,519,808 | 7,782,400 | 65.4% |
| 100 | 22,519,808 | 6,873,088 | 69.5% |
| 1000 | 22,519,808 | 7,340,032 | 67.4% |

The curve is flat above roughly 20 rows per key, and the small reversal at 1000 is
expected: a posting list is capped, and the single-value split strategy uses a 96%
fill factor
([nbtree.h#BTREE_SINGLEVAL_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)).
A practical reading: two rows per key is barely worth it, five or more is worth it,
and unique or near-unique keys are not.

### The cost of leaving it alone

The carried-over index does not stop growing at the old rate — it keeps splitting
where a deduplicating index would merge. Measured on the 20-distinct-value
`t_churn` fixture after the upgrade, inserting 200,000 more duplicate rows with both
indexes present:

| index | before insert | after insert |
|---|---|---|
| `i_churn` (carried over from 12.2) | 4,521,984 | 8,904,704 |
| `i_churn_new` (built on 17.10) | 1,400,832 | 2,850,816 |

The carried-over index ended 3.1x larger than the index built after the upgrade over
identical data.

### Separating deduplication from ordinary bloat

On a table that has taken updates since the upgrade, a rebuild reclaims both bloat
and the deduplication win, and a size drop alone cannot tell you which. Build two
twins and the arithmetic separates:

```sql
CREATE INDEX CONCURRENTLY /* wiki_dedup_twin_off */ twin_off ON t_main (k10)
  WITH (deduplicate_items = off);
CREATE INDEX CONCURRENTLY /* wiki_dedup_twin_on */ twin_on ON t_main (k10);
```

Measured, same index definition, on a cluster whose table had been rewritten once
after the upgrade:

| relation | bytes | reading |
|---|---|---|
| `i_dup10`, carried over | 43,737,088 | current state |
| `twin_off`, fresh, deduplication disabled | 22,519,808 | bloat = 48.5% of the current index |
| `twin_on`, fresh, deduplication enabled | 6,963,200 | deduplication = 69.1% of what is left |

On the never-updated cluster the same comparison put `twin_off` at 22,519,808 bytes
— exactly the carried-over index's size, to the byte — which is the direct
measurement that a v12-built index is precisely a v17 index with deduplication
turned off. Drop the twins afterwards.

### What sets the flag, and what does not

| operation | sets `btm_allequalimage` | measured |
|---|---|---|
| `REINDEX INDEX CONCURRENTLY` | yes | flag true, `pd_lower` 72 |
| `REINDEX INDEX` | yes | flag true, relfilenode 16395 → 16492 |
| `VACUUM FULL` | yes, it rebuilds every index of the table | `i_churn` and `i_ts` both flipped, with a `DEBUG1` line each |
| `CLUSTER` | yes, same path | 17 `DEBUG1` lines for one table's indexes |
| `VACUUM`, including `INDEX_CLEANUP ON` | no | `pd_lower` stayed 64, flag stayed false |
| `ALTER INDEX ... SET (deduplicate_items = ...)` | no | flag unchanged |

`VACUUM FULL` and `CLUSTER` set it because they finish by rebuilding the table's
indexes
([cluster.c#finish_heap_swap](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1490-L1508)).
For a large production index, prefer `REINDEX INDEX CONCURRENTLY`, which holds only
`ShareUpdateExclusiveLock`; its phases, waits and failure modes are covered in
[How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md).

### Rebuild order and the statistics pg_upgrade leaves behind

`pg_upgrade` restores the schema by creating empty indexes and then swapping files
in, so the planner statistics for those indexes describe the empty build. Measured:
all 21 carried-over indexes had `relpages = 0` and `reltuples = 0` — not the v14
`-1` unknown sentinel — until the first `ANALYZE`. Any triage that divides by
`reltuples`, including the sweep on
[Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)](btree-index-bloat-core-sql-only.md),
is blind until the `vacuumdb --all --analyze-in-stages` that `pg_upgrade` itself
recommends has run
([check.c#output_completion_banner](../../../../raw/postgres-17/src/bin/pg_upgrade/check.c#L760-L790)).
The metapage probe does not depend on statistics at all, which is its main
advantage right after an upgrade.

After analyzing, rank the flagged indexes by size and duplication:

```sql
SELECT /* wiki_dedup_rebuild_priority */
       c.relname,
       pg_size_pretty(pg_relation_size(c.oid)) AS index_size,
       c.reltuples::bigint                     AS index_reltuples,
       round(s.keys::numeric, 0)               AS distinct_keys,
       CASE WHEN s.keys > 0 AND c.reltuples > 0
            THEN round((c.reltuples / s.keys)::numeric, 1) END AS rows_per_key
FROM pg_class c
JOIN pg_index i ON i.indexrelid = c.oid
JOIN pg_class t ON t.oid = i.indrelid
JOIN pg_namespace n ON n.oid = c.relnamespace
LEFT JOIN LATERAL (
       SELECT CASE WHEN count(*) = i.indnkeyatts
                   THEN exp(sum(ln(CASE WHEN st.n_distinct > 0 THEN st.n_distinct
                                        ELSE -st.n_distinct * greatest(t.reltuples, 1) END)))
              END AS keys
       FROM generate_series(0, i.indnkeyatts - 1) AS k
       JOIN pg_attribute a2 ON a2.attrelid = i.indrelid AND a2.attnum = i.indkey[k]
       JOIN pg_stats st ON st.schemaname = n.nspname AND st.tablename = t.relname
                       AND st.attname = a2.attname
       WHERE st.n_distinct <> 0
     ) s ON true
WHERE c.relkind = 'i'
  AND n.nspname = 'public'
  AND pg_relation_size(c.oid) > 8192
ORDER BY pg_relation_size(c.oid) DESC;
```

Measured behaviour after `ANALYZE`: `rows_per_key` came out 100,048.7 for the
10-distinct-value index, 1000.5 for the 1000-value one, 100.0 for the two-column
index and 1.0 for the unique index, and `NULL` for the expression index `i_expr`,
whose key has no `pg_stats` row under the table's column names. Expression and
partial indexes need the fuller model on the bloat page cited above; this query
only orders work.

### Second opinion: make the engine say it

`_bt_allequalimage()` logs its own decision at `DEBUG1` on every build, so a rebuild
can be made to state whether it enabled deduplication:

```sql
SET client_min_messages = debug1;
REINDEX INDEX CONCURRENTLY /* wiki_dedup_rebuild */ public.i_dup10;
-- DEBUG:  index "i_dup10_ccnew" can safely use deduplication
```

Two limits, both measured. The message names the transient `_ccnew` index, and it
reports `_bt_allequalimage()` only: it also said "can safely use deduplication" for
the index rebuilt under `deduplicate_items = off` that came out at full size. And it
is silent for `INCLUDE` indexes. It is a confirmation, not a pre-check.

### Fallbacks when you cannot read server files

If nobody will grant EXECUTE on `pg_read_binary_file`, two weaker options remain:

1. **Snapshot `relfilenode` right after the upgrade**, then treat any index whose
   relfilenode still matches the snapshot as un-rebuilt. Measured: a table captured
   immediately after the upgrade plus one `REINDEX INDEX` correctly identified the
   single rebuilt index (`i_partial`, 16395 → 16492).

   ```sql
   CREATE TABLE wiki_dedup_baseline AS
   SELECT /* wiki_dedup_baseline */ c.oid AS indexrelid, c.relname,
          c.relfilenode, now() AS captured
   FROM pg_class c
   JOIN pg_am a ON a.oid = c.relam
   WHERE c.relkind = 'i' AND a.amname = 'btree';
   ```

2. **Build the two twins** from
   [Separating deduplication from ordinary bloat](#separating-deduplication-from-ordinary-bloat)
   and compare sizes. Definitive for one index, but it costs two index builds.

What does *not* work as a durable after-the-fact signal is `pg_class.xmin` or
relfilenode magnitude. `pg_upgrade` sets the new cluster's XID and OID counters from
the old cluster's control data
([pg_upgrade.c#main](../../../../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L155-L197)),
so restored rows do not sit at obviously low values: every carried-over index's
`pg_class` row had `xmin = 534`, one restore transaction just past the old cluster's
`NextXID` of 487, and relfilenodes continued from the old cluster's `NextOID` of
24576. The band is only recognizable if you recorded it at upgrade time.

### Edge cases the check has to survive

All measured on the upgraded cluster:

- **Partitioned indexes.** The parent (`relkind = 'I'`) has no storage and
  `pg_relation_filepath()` returns NULL, so the `relkind = 'i'` filter drops it;
  both leaf indexes were flagged and rebuilt individually.
- **Unlogged tables.** `i_u_k` on an unlogged table probed normally and was flagged.
- **Non-default tablespaces.** `i_ts` lives in a second tablespace; the path
  `pg_relation_filepath()` returns resolves through the data directory's
  `pg_tblspc` symlink, so the probe worked unchanged.
- **Temporary indexes.** Excluded by `relpersistence <> 't'`. A temp index can be
  probed from its owning session only (`base/16384/t33_16527` read byte 64 = 1), and
  it is never carried over by an upgrade anyway.
- **TOAST indexes.** Carried over with their tables. `pg_toast_16385_index` read
  false; rebuilding one needs its schema-qualified name.
- **`pg_largeobject`.** This is the one system catalog whose files `pg_upgrade`
  transfers ([info.c#get_rel_infos](../../../../raw/postgres-17/src/bin/pg_upgrade/info.c#L471-L496)),
  and `pg_largeobject_loid_pn_index` was the single `pg_catalog` B-tree index with
  `pd_lower = 64` and byte 64 false in the upgraded cluster. Its keys are
  equal-image, so a rebuild sets the flag, but it is unique, so expect no size
  change.
- **`pg_enum_typid_sortorder_index`.** The only freshly built catalog index that
  reads false, because `enumsortorder` is `float4`. Nothing to do.
- **Zero-length or missing files.** The `octet_length(...) = 72` guard and
  `missing_ok => true` make the probe return NULL instead of raising
  `ERROR:  index 64 out of valid range`, which is what `get_byte()` would do on a
  short read ([varlena.c#byteaGetByte](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L3246-L3268)).

### Where this comes from in the source history

From this checkout's own history, with first release tags:

| commit | subject | first release |
|---|---|---|
| `612a1ab7672` | Add equalimage B-Tree support functions. | `REL_13_0` |
| `0d861bbb702` | Add deduplication to nbtree. | `REL_13_0` |
| `9f3665fbfc3` | Don't consider newly inserted tuples in nbtree VACUUM. | `REL_14_0` |

`0d861bbb702` is also the commit that introduced the "zero'ed on ... pg_upgrade'd
from Postgres 12" comment in `_bt_metaversion()`, and no commit in
`src/bin/pg_upgrade` has ever touched `allequalimage` (`git log -S allequalimage --
src/bin/pg_upgrade` is empty), so the gap the comment describes is still open in
17.10. `9f3665fbfc3` matters to carried-over indexes for a different reason: it
repurposed the metapage's old `btm_oldest_btpo_xact` field as
`btm_last_cleanup_num_delpages`, so a v12-built metapage can present a stale XID as
a deleted-page count, which the code tolerates deliberately
([nbtpage.c#_bt_set_cleanup_info](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L225-L263)).
Measured here as harmless: that field read 0 in all fixtures.

### Settings this page touches

| setting | value used | context | apply scope |
|---|---|---|---|
| `statement_timeout` | `60s` for the read-only checks | `PGC_USERSET` ([guc_tables.c:2611-2620](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)) | session or transaction |
| `lock_timeout` | `5s` for the read-only checks | `PGC_USERSET` ([guc_tables.c:2622-2632](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2632)) | session or transaction |
| `client_min_messages` | `debug1` to see the engine's verdict | `PGC_USERSET` ([guc_tables.c:4776-4785](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4776-L4785)) | session or transaction |
| `deduplicate_items` | leave at `on` | B-tree reloption, `ShareUpdateExclusiveLock` ([reloptions.c:159-168](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)) | per index, `ALTER INDEX` |

No restart or reload is needed for anything on this page. Do not raise
`statement_timeout` for the rebuilds themselves without thought: a long
`REINDEX INDEX CONCURRENTLY` that is cancelled leaves an invalid `_ccnew` index
behind, which the sibling page on `REINDEX INDEX CONCURRENTLY` describes.

## Context Reviewed

- Metapage and deduplication implementation: `src/include/access/nbtree.h`,
  `src/backend/access/nbtree/nbtpage.c`, `nbtutils.c`, `nbtsort.c`, `nbtinsert.c`,
  `nbtdedup.c`, `nbtree.c`, `nbtxlog.c`.
- Equal-image support functions and their catalog rows:
  `src/backend/utils/adt/datum.c`, `src/backend/utils/adt/varlena.c`,
  `src/include/catalog/pg_amproc.dat`, `src/include/catalog/pg_proc.dat`.
- Index creation restrictions: `src/backend/catalog/index.c`.
- Reloption plumbing: `src/backend/access/common/reloptions.c`.
- Rebuild paths: `src/backend/commands/cluster.c`.
- Core file/byte functions and their ACLs: `src/backend/utils/adt/genfile.c`,
  `src/backend/utils/adt/dbsize.c`, `src/backend/catalog/system_functions.sql`.
- `pg_upgrade`: `src/bin/pg_upgrade/pg_upgrade.c`, `check.c`, `info.c`,
  `relfilenumber.c`, and `src/bin/pg_dump/pg_dump.c` binary-upgrade support.
- Documentation: `doc/src/sgml/btree.sgml`, `doc/src/sgml/ref/create_index.sgml`,
  `doc/src/sgml/ref/pgupgrade.sgml`.
- Tests: `src/test/regress/sql/btree_index.sql` and its expected output,
  `src/test/regress/expected/collate.icu.utf8.out`,
  `src/bin/pg_amcheck/t/005_opclass_damage.pl`,
  `contrib/pageinspect/expected/btree.out`.
- Ground truth only, not part of the answer: `contrib/pageinspect/btreefuncs.c`.
- Source history: `git log`/`git tag --contains` in this checkout for
  `612a1ab7672`, `0d861bbb702`, `9f3665fbfc3`, and `git log -S allequalimage`
  over `src/bin/pg_upgrade`.
- Live measurement: an isolated 12.2 server and three isolated 17.10 clusters
  created by `pg_upgrade --copy` from it, plus one isolated ICU-enabled 17.10
  cluster for the nondeterministic-collation cases. Fixtures, sizes, `DEBUG1`
  verdicts, MD5s, privilege errors and rebuild results are reported inline.
- Pinned checkout `raw/postgres-17/` at commit
  `786db8dcf168bd9df8f55047337525ac19118b1c` (PostgreSQL 17.11,
  `REL_17_11-7-g786db8dcf16`); repinned from
  `54eeefaedbee0385529f3edf321bb99e49232aaa` (17.10) on 2026-08-17. Every number
  above is a 17.10 measurement taken against that previous pin and was not
  re-measured. In the `54eeefaed..786db8dcf16` range, `nbtree.c`, `nbtdedup.c`,
  `nbtsort.c`, `nbtutils.c`, `nbtree.h` and contrib `pageinspect`'s `bt_metap`
  are untouched; the only `pg_upgrade` change is `01992176e08`, which adds a
  check that every output plugin used by a migrated logical replication slot is
  listed in the new cluster's `output_plugin_libraries`
  ([check.c#check_new_cluster_logical_replication_slots](../../../../raw/postgres-17/src/bin/pg_upgrade/check.c#L1865-L1943)).
  That check runs only when the old cluster reports at least one logical
  replication slot (`if (nslots_on_old > 0)`), which none of these fixtures
  creates, so no upgrade result above changes.

## Evidence Map

| Claim | Source |
|---|---|
| A v12-built index needs a `REINDEX` to deduplicate; `pg_upgrade` does not set the field | [nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L124-L147) |
| The field is assumed zero on indexes upgraded from 12 | [nbtpage.c#_bt_metaversion](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L719-L791) |
| `pageinspect` relies on the same assumption | [btreefuncs.c#bt_metap](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L905-L922) |
| Version 4 predates 13, so version does not identify a v12 build | [nbtree.h#BTREE_VERSION](../../../../raw/postgres-17/src/include/access/nbtree.h#L148-L152) |
| Metapage field order and `pd_lower` | [nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L119), [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L63-L96) |
| Page contents start at `MAXALIGN(SizeOfPageHeaderData)` | [bufpage.h#PageGetContents](../../../../raw/postgres-17/src/include/storage/bufpage.h#L246-L258) |
| Only a rebuild can set the field; the in-place upgrade refuses | [nbtpage.c#_bt_upgrademetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L98-L131) |
| Replay copies the flag but rewrites `pd_lower` | [nbtxlog.c#_bt_restore_meta](../../../../raw/postgres-17/src/backend/access/nbtree/nbtxlog.c#L101-L125) |
| Build stores the flag; empty build too | [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L531-L565), [nbtsort.c#_bt_uppershutdown](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1118-L1128), [nbtree.c#btbuildempty](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L158-L171) |
| Insert-time deduplication is gated on the flag and the reloption | [nbtinsert.c#_bt_delete_or_dedup_one_page](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2774-L2782) |
| Build-time deduplication also requires non-uniqueness | [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1144-L1152) |
| Flag reaches scans/inserts through the insertion scan key | [nbtutils.c#_bt_mkscankey](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L118-L160) |
| Eligibility rule: `INCLUDE`, support function 4, collation | [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183), [nbtree.h#BTEQUALIMAGE_PROC](../../../../raw/postgres-17/src/include/access/nbtree.h#L686-L712) |
| `btequalimage` always true; `btvarstrequalimage` false for nondeterministic collations | [datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L415-L438), [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613) |
| Which opclasses register which support function | [pg_amproc.dat:201-212](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L201-L212) |
| Documented unsafe cases and the unique-index note | [btree.sgml#btree-deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L810-L909) |
| Pattern opclasses reject nondeterministic collations at DDL time | [index.c#index_create](../../../../raw/postgres-17/src/backend/catalog/index.c#L827-L848) |
| `deduplicate_items` default, lock level, no retroactive effect | [create_index.sgml#deduplicate_items](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L448-L476), [reloptions.c:159-168](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168), [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1129-L1151) |
| `pg_upgrade` transfers user indexes plus TOAST and `pg_largeobject` | [info.c#get_rel_infos](../../../../raw/postgres-17/src/bin/pg_upgrade/info.c#L471-L524) |
| Index OID and relfilenode are preserved by the dump | [pg_dump.c#binary_upgrade_set_next_index_relfilenode](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L5563-L5572) |
| Files are copied or linked, not rewritten | [relfilenumber.c#transfer_relfile](../../../../raw/postgres-17/src/bin/pg_upgrade/relfilenumber.c#L168-L266) |
| Upgrades from 9.2 and later are allowed, so 12 → 17 is in scope | [check.c#check_cluster_versions](../../../../raw/postgres-17/src/bin/pg_upgrade/check.c#L795-L824) |
| The completion banner recommends analyzing, nothing about indexes | [check.c#output_completion_banner](../../../../raw/postgres-17/src/bin/pg_upgrade/check.c#L760-L790) |
| XID/OID counters come from the old cluster, so `xmin` is not a durable marker | [pg_upgrade.c#main](../../../../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L155-L197) |
| `VACUUM FULL`/`CLUSTER` rebuild all of a table's indexes | [cluster.c#finish_heap_swap](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1490-L1508) |
| Path rules and the `pg_read_server_files` role | [genfile.c#convert_and_check_filename](../../../../raw/postgres-17/src/backend/utils/adt/genfile.c#L41-L92) |
| The file-reading functions are revoked from `PUBLIC` | [system_functions.sql:710-719](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L710-L719) |
| `pg_relation_filepath()` returns a `$PGDATA`-relative path | [dbsize.c#pg_relation_filepath](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L948-L1028) |
| `get_byte()` errors outside the `bytea` bounds | [varlena.c#byteaGetByte](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L3246-L3268) |
| Single-value fill factor explains the flat top of the savings curve | [nbtree.h#BTREE_SINGLEVAL_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202) |
| Repurposed metapage field can hold a stale XID after an upgrade | [nbtpage.c#_bt_set_cleanup_info](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L225-L263) |
| Deduplication in unique indexes is exercised in the regression tests | [btree_index.sql:208-231](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L208-L231) |
| Pattern-opclass rejection is exercised in the ICU collation test | [collate.icu.utf8.out:1796-1801](../../../../raw/postgres-17/src/test/regress/expected/collate.icu.utf8.out#L1796-L1801) |
| Timeout and message GUC contexts | [guc_tables.c:2611-2632](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2632), [guc_tables.c:4776-4785](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4776-L4785) |

## Open Questions

- **The byte offsets were verified on one platform.** `MAXIMUM_ALIGNOF` was 8,
  `BLCKSZ` 8192, on x86-64 Linux. The offsets follow from
  `MAXALIGN(SizeOfPageHeaderData)` plus the struct layout, so a 32-bit or
  unusually aligned build could shift `btm_allequalimage` away from byte 64. The
  self-test in [Check 2](#check-2-prove-the-byte-offsets-on-your-platform) detects
  that, but no such platform was tested.
- **No standby or crash-recovery run was made.** `_bt_restore_meta()` rewrites
  `pd_lower` on replay, so a carried-over metapage that has been through replay
  should report `pd_lower = 72` with byte 64 still false. That combination was
  reasoned from source, not observed; only primaries were measured.
- **The probe reads the file, not shared buffers.** A fresh build was visible
  immediately with no checkpoint because index builds bulk-write their pages, but
  no test forced a metapage update to sit dirty in shared buffers while the probe
  ran. `btm_allequalimage` never changes after build, so the risk is bounded, but
  it is unmeasured.
- **12.2 is the only old version measured.** An 11 or earlier cluster would also
  carry `btm_version = 3` metapages, where `_bt_upgrademetapage()` and the
  `heapkeyspace` differences matter as well; nothing here was run against one.
- **The gate was scored on 19 index shapes.** It agreed with the engine's `DEBUG1`
  verdict on all of them, but the space of opclasses is much larger: user-defined
  opclasses that register support function 4 and return false for reasons other
  than collation would defeat the SQL mirror, which cannot call the function.
- **`i_inc` lost six pages on rebuild and the cause was not isolated.** The
  `INCLUDE` index went from 3859 to 3853 pages with byte 64 still false, so it is
  not deduplication; it was attributed to a v12-versus-v17 build difference without
  proof.
- **Partial-index triage is approximate.** `rows_per_key` divides the index's own
  `reltuples` by the table's `n_distinct`, which assumes the predicate does not
  correlate with the key.
- **No timing was recorded for the rebuilds.** Sizes were measured, wall-clock cost
  and WAL volume of the `REINDEX INDEX CONCURRENTLY` runs were not.
- **`pg_largeobject_loid_pn_index` was flagged but not rebuilt.** It is unique, so
  no size change is expected, and whether `REINDEX` on that catalog index is
  acceptable in production was not tested.

## Source References

- [nbtree.h#BTMetaPageData](../../../../raw/postgres-17/src/include/access/nbtree.h#L103-L152)
- [nbtree.h#BTEQUALIMAGE_PROC](../../../../raw/postgres-17/src/include/access/nbtree.h#L686-L712)
- [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1129-L1151)
- [nbtpage.c#_bt_initmetapage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L63-L131)
- [nbtpage.c#_bt_set_cleanup_info](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L225-L263)
- [nbtpage.c#_bt_metaversion](../../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L719-L791)
- [nbtutils.c#_bt_mkscankey](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L118-L160)
- [nbtutils.c#_bt_allequalimage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5129-L5183)
- [nbtsort.c#_bt_leafbuild](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L531-L565)
- [nbtsort.c#_bt_load](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1118-L1152)
- [nbtinsert.c#_bt_findinsertloc](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L899-L907)
- [nbtinsert.c#_bt_delete_or_dedup_one_page](../../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2654-L2782)
- [nbtree.c#btbuildempty](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L158-L171)
- [nbtxlog.c#_bt_restore_meta](../../../../raw/postgres-17/src/backend/access/nbtree/nbtxlog.c#L101-L125)
- [bufpage.h#PageGetContents](../../../../raw/postgres-17/src/include/storage/bufpage.h#L246-L258)
- [datum.c#btequalimage](../../../../raw/postgres-17/src/backend/utils/adt/datum.c#L415-L438)
- [varlena.c#btvarstrequalimage](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)
- [varlena.c#byteaGetByte](../../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L3246-L3268)
- [pg_amproc.dat:201-212](../../../../raw/postgres-17/src/include/catalog/pg_amproc.dat#L201-L212)
- [pg_proc.dat#btvarstrequalimage](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L1055-L1057)
- [index.c#index_create](../../../../raw/postgres-17/src/backend/catalog/index.c#L827-L848)
- [reloptions.c:159-168](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)
- [cluster.c#finish_heap_swap](../../../../raw/postgres-17/src/backend/commands/cluster.c#L1433-L1508)
- [genfile.c#convert_and_check_filename](../../../../raw/postgres-17/src/backend/utils/adt/genfile.c#L41-L92)
- [dbsize.c#pg_relation_filepath](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L948-L1028)
- [system_functions.sql:710-719](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L710-L719)
- [guc_tables.c:2611-2632](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2632)
- [guc_tables.c:4776-4785](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4776-L4785)
- [pg_upgrade.c#main](../../../../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L155-L197)
- [pg_upgrade.c#copy_xact_xlog_xid](../../../../raw/postgres-17/src/bin/pg_upgrade/pg_upgrade.c#L701-L737)
- [check.c#output_completion_banner](../../../../raw/postgres-17/src/bin/pg_upgrade/check.c#L760-L790)
- [check.c#check_cluster_versions](../../../../raw/postgres-17/src/bin/pg_upgrade/check.c#L795-L824)
- [info.c#get_rel_infos](../../../../raw/postgres-17/src/bin/pg_upgrade/info.c#L471-L524)
- [relfilenumber.c#transfer_relfile](../../../../raw/postgres-17/src/bin/pg_upgrade/relfilenumber.c#L168-L266)
- [pg_dump.c#binary_upgrade_set_next_index_relfilenode](../../../../raw/postgres-17/src/bin/pg_dump/pg_dump.c#L5563-L5572)
- [btree.sgml#btree-deduplication](../../../../raw/postgres-17/doc/src/sgml/btree.sgml#L736-L911)
- [create_index.sgml#deduplicate_items](../../../../raw/postgres-17/doc/src/sgml/ref/create_index.sgml#L448-L476)
- [pgupgrade.sgml#post-upgrade-scripts](../../../../raw/postgres-17/doc/src/sgml/ref/pgupgrade.sgml#L1078-L1086)
- [btree_index.sql:208-231](../../../../raw/postgres-17/src/test/regress/sql/btree_index.sql#L208-L231)
- [collate.icu.utf8.out:1796-1801](../../../../raw/postgres-17/src/test/regress/expected/collate.icu.utf8.out#L1796-L1801)
- [005_opclass_damage.pl:45-52](../../../../raw/postgres-17/src/bin/pg_amcheck/t/005_opclass_damage.pl#L45-L52)
- [btreefuncs.c#bt_metap](../../../../raw/postgres-17/contrib/pageinspect/btreefuncs.c#L828-L922)
- [btree.out:1-16](../../../../raw/postgres-17/contrib/pageinspect/expected/btree.out#L1-L16)

## Navigation

- [v17/index](../../index.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md)
- [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)](btree-index-bloat-core-sql-only.md)
- [How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](create-index-concurrently.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [versions](../../../versions.md)
- [Wiki Index](../../../index.md)
