---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: not yet
---

# Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Verdict](#verdict)
  - [Calibrated thresholds per access method](#calibrated-thresholds-per-access-method)
  - [Why the metric behaves the way it does in v17](#why-the-metric-behaves-the-way-it-does-in-v17)
  - [Recording the baseline with COMMENT ON INDEX](#recording-the-baseline-with-comment-on-index)
  - [The maintenance-time check](#the-maintenance-time-check)
  - [Index-specific factors](#index-specific-factors)
  - [Space reclaimed is not query performance](#space-reclaimed-is-not-query-performance)
  - [Failure modes that no threshold can fix](#failure-modes-that-no-threshold-can-fix)
  - [Operational recipe](#operational-recipe)
  - [Settings and their apply scope](#settings-and-their-apply-scope)
  - [What changed since PostgreSQL 12](#what-changed-since-postgresql-12)
- [Test Summary](#test-summary)
  - [Environment and protocol](#environment-and-protocol)
  - [Fixture and index shapes](#fixture-and-index-shapes)
  - [Workloads](#workloads)
  - [Result matrix](#result-matrix)
  - [Threshold scoring](#threshold-scoring)
  - [Scale sweep](#scale-sweep)
  - [Access-method probes](#access-method-probes)
  - [v17-specific probes](#v17-specific-probes)
  - [COMMENT mechanics checked on the pin](#comment-mechanics-checked-on-the-pin)
  - [Reproducibility and cleanup](#reproducibility-and-cleanup)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17, copy the question from PostgreSQL 12, "Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 12", and review it for PostgreSQL 17.

The copied question is: record a stable post-build baseline for every index except btree in its PostgreSQL index comment using `COMMENT ON INDEX`. Calculate the baseline as **index size in bytes divided by the table's pg_class.reltuples**, producing bytes per tuple. During maintenance, recalculate the ratio and compare it with the stored baseline; an increase indicates that the index is consuming more space per live tuple. For each index type and workload, determine the percentage increase that serves as the best proxy for triggering a reindex while minimizing unnecessary rebuilds and missed harmful bloat. Calibrate this threshold using realistic insert, update, and delete workloads, then rebuild each index and measure the actual space reclaimed and query-performance improvement. Include index-specific factors, such as the GIN pending list, when evaluating the accuracy of the proxy. Document a summary of the tests and how they were tested.

## Answer

### Verdict

The v12 calibration transfers to v17 almost unchanged: four of the five non-B-tree access methods score identically, and **GiST is the only one whose numbers moved**. Measured on an isolated PostgreSQL 17.10 server built from this pin, over the same 132 cells (22 workloads x 6 index shapes) with `REINDEX` as ground truth.

How to read the v12 comparisons on this page. Two different kinds appear, and they carry different weight:

- Every *behavioral* v12-versus-v17 difference is attributed to a commit found in this checkout's own history; see [What changed since PostgreSQL 12](#what-changed-since-postgresql-12).
- Every *numeric* "v12 measured X" is a pointer to the numbers published on [the v12 page](../../../v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md), which was measured on a different pin and a different build. Those are cross-references to prior wiki work, not evidence from this checkout. Only the v17 figures on this page are backed by measurements taken on this pin.

The findings:

- **GiST still calibrates perfectly, but its usable window shrank and shifted up.** Any threshold in **25-33%** separates every "worth rebuilding" cell from every "not worth it" cell (14 true positives, 0 false positives, 0 false negatives). The window's lower edge is a GiST `churn25` cell reading +24.46% at 19.65% reclaimable — 0.35 points below the 20% bar. On this pin **30% is the safer choice**; the v12 page's recommended 25% still scores perfectly here, but with 0.54 points of margin instead of the ~11 its own window allowed.
- **hash, SP-GiST, GIN and BRIN reproduce the v12 page's confusion matrices cell for cell.** hash 9-11% (16/0/0/6), SP-GiST 29-32% (15/0/0/7), GIN `fastupdate = on` 56-100% (18/0/0/4), GIN `fastupdate = off` 40-62% (16/2/0/4, 90.9%), BRIN unusable at every value (0/4/0/18).
- **BRIN must still never be rebuilt on this signal.** `REINDEX` reclaimed exactly 0.00% in all 22 BRIN cells while the reading moved between -87.50% and +9900.00%.
- **v17 adds one hazard v12 could not have: `pg_class.reltuples` is now `-1` for "unknown".** The comment on the column reads "-1 means \"unknown\"" and the catalog default is `-1` [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66). The v12 formula divides by it, so on a never-analyzed table it returns a **negative** bytes-per-tuple. Measured: a 200,000-row never-analyzed table with a 7,438,336-byte hash index produced a naive ratio of **-7,438,336.00**, which any percentage comparison reads as a large improvement. Every guard in this page therefore tests `reltuples <= 0`, not `= 0`.

The reason the proxy works at all is unchanged: no compiled index-vacuum path shrinks an index main fork. `RelationTruncate` [storage.c#RelationTruncate](../../../../raw/postgres-17/src/backend/catalog/storage.c#L280-L439) has exactly four call sites in this tree — two heap-side ones in `vacuumlazy.c` and `heapam_handler.c`, and two index-side ones: `TRUNCATE`'s immediate-rebuild path [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3058-L3066) and SP-GiST's, which is still compiled out under `#ifdef NOT_USED` with the comment "Note that btree doesn't do this either" [spgvacuum.c#NOT_USED](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L877-L900). Every access method instead recycles freed pages through the FSM fork [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), which `pg_relation_size`'s default `'main'` fork does not count [system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289).

### Calibrated thresholds per access method

"Worth rebuilding" below means a `REINDEX` that reclaims at least 20% of the index's current bytes. `Window` is the measured gap between the highest reading among not-worth-it cells and the lowest reading among worth-it cells; any threshold inside the window scores perfectly on this data set. `Growth noise` is the spread of a *freshly built* index's ratio across the sweep ranges reported under [Test Summary](#test-summary), which is the drift a stored baseline accumulates with no bloat at all.

| Index type | Recommended threshold | Perfect window | TP / FP / FN / TN | Growth noise (fresh build) | Verdict |
|---|---|---|---|---|---|
| GiST | **30%** | 24.46% -> 33.33% | 14 / 0 / 0 / 8 | -3.18% to +2.17% | calibrates; noise far below the threshold, but the window is 9 points wide |
| SP-GiST | **30%** | 28.24% -> 32.17% | 15 / 0 / 0 / 7 | -4.80% to +33.08% | only with a stable row count; re-baseline on growth |
| hash | **10%** | 8.78% -> 11.11% | 16 / 0 / 0 / 6 | -20.31% to +49.42% | only with a stable row count; staircase swamps the window |
| GIN, `fastupdate = off` | **50%** | none at 20% (best 40-62%) | 16 / 2 / 0 / 4 | +13.78% to +80.48% | screening only |
| GIN, `fastupdate = on` | **75%** | 55.84% -> 100.00% | 18 / 0 / 0 / 4 | +13.78% to +80.48% | screening only; flush the pending list first |
| BRIN | **never fire** | none at any value | 0 / 4 / 0 / 18 | +700.00% to -87.50% | do not use; `REINDEX` reclaimed 0.00% in 22/22 cells |

The two right-hand columns come from different sweeps and use different reference points: the confusion counts are the 22 workload cells at a fixed 200,000 rows, the hash growth-noise range is the 60,000-row-referenced splitpoint sweep, and the other growth-noise ranges are the 200,000-row-referenced scale sweep.

A single threshold for all five is worse than per-method values but not useless: 30% scored 90.2% accuracy over all 132 cells (78 TP, 12 FP, 1 FN, 41 TN), and 93.6% over the 110 non-BRIN cells (78 TP, 6 FP, 1 FN, 25 TN) — the same counts the v12 page reports. Ten of the twelve false positives at 30% are BRIN. The single false negative at 30% is hash `del10`, reading +11.11% against 22.66% reclaimable.

Rank correlation between the reading and the reclaimable fraction is high for everything except BRIN: GiST 0.989, hash 0.987, GIN `fastupdate = on` 0.974, SP-GiST 0.958, GIN `fastupdate = off` 0.936, BRIN undefined (its reclaimable fraction has zero variance).

### Why the metric behaves the way it does in v17

The ratio mixes two independent quantities, and one of them is far noisier than the thing being measured.

**Numerator — monotone, but quantized per access method.** `pg_relation_size` sums the segment files of one fork [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L343), and the one-argument form is a SQL-body wrapper that hardcodes `'main'` [pg_proc.dat#pg_relation_size](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7491). Freed pages leave the main fork's high-water mark untouched:

- hash frees an overflow page by clearing a bitmap bit, never by truncating [hashovfl.c#_hash_freeovflpage](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c#L632-L642), and its README states it outright: "There is currently no provision to shrink a hash index, other than by rebuilding it with REINDEX" [hash/README:31-34](../../../../raw/postgres-17/src/backend/access/hash/README#L31-L34). `hashvacuumcleanup` only reports `RelationGetNumberOfBlocks` [hash.c#hashvacuumcleanup](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L642-L663).
- GiST deletes empty leaf pages and records recyclable ones in the FSM [gistvacuum.c#gistvacuumpage](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L298-L304), gated on the page's deletion XID being invisible to every snapshot [gistutil.c#gistPageRecyclable](../../../../raw/postgres-17/src/backend/access/gist/gistutil.c#L885-L908).
- GIN puts recyclable pages in the FSM and refreshes the metapage statistics [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L747-L764), then vacuums the FSM [ginvacuum.c:780-785](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L780-L785).
- SP-GiST converts expired redirect tuples to placeholders and only removes placeholders at the end of a page [spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L569-L572).
- BRIN's `ambulkdelete` is a no-op by design [brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301), and `brin_vacuum_scan` only refreshes the FSM [brin.c#brin_vacuum_scan](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2186-L2192).

Measured: in all 36 delete cells the index was byte-identical before and after `VACUUM`, for all six index shapes.

**Denominator — an estimate with four distinct states in v17.** Fresh, stale (nothing ran `ANALYZE`), extrapolated (a partial `VACUUM` reused the previous density), or the `-1` unknown sentinel. `vac_estimate_reltuples` now has two short-circuits that return the previous value verbatim, and the code says so: "(Note: we might be returning -1 here.)" [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1329-L1354). It is refreshed by block-sampled `ANALYZE` [analyze.c#acquire_sample_rows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1285-L1289), written by `vac_update_relstats` [vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1444-L1461), and — critically for a "post-build baseline" — by every index build, which rewrites the *heap's* `relpages` and `reltuples` as a side effect [index.c#index_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L3097-L3106).

Because the numerator cannot fall and the denominator can, a pure delete workload produces a reading that is a closed-form function of the delete fraction and carries **no information about the index at all**. Measured: every access method read exactly +11.11%, +33.33%, +100.00%, +300.00%, +900.00% and +9900.00% after deleting 10%, 25%, 50%, 75%, 90% and 99% of rows — that is `1/(1-f) - 1` to the cent — while the reclaimable fraction at +900.00% ranged from 84.58% (hash) to 90.12% (GiST) to 0.00% (BRIN).

### Recording the baseline with COMMENT ON INDEX

`COMMENT ON INDEX` accepts a string literal or `NULL` and nothing else — the grammar production is still `Sconst | NULL_P` [gram.y#comment_text](../../../../raw/postgres-17/src/backend/parser/gram.y#L7219-L7222). Confirmed on the pin: `COMMENT ON INDEX i IS (SELECT '1.0')` and `COMMENT ON INDEX i IS 12.5` both fail with `syntax error`. A computed baseline therefore has to go through dynamic SQL. Other mechanics verified on the pin:

| Behavior | Evidence |
|---|---|
| Takes `ShareUpdateExclusiveLock` on the index, held to commit (measured: one `relation` lock on the index, none on the table). v17 documents the lock level explicitly. | [comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L73), [ref/comment.sgml#lock](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L98) |
| Requires ownership of the index (measured: `ERROR: must be owner of index t_cm_gin`) | [objectaddress.c#check_object_ownership](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L2396-L2401) |
| One row per index in `pg_description`, `classoid = pg_class`, `objsubid = 0` (measured) | [comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L142-L226) |
| `IS NULL` and `IS ''` both delete the row (measured: 0 rows after each) | [comment.c:155-157](../../../../raw/postgres-17/src/backend/commands/comment.c#L155-L157) |
| Canonical read is `obj_description(idx, 'pg_class')`, whose second argument is `name` | [system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301) |
| Plain `REINDEX INDEX` keeps it: the OID does not change (measured: 18188 before and after) | [index.c#reindex_index](../../../../raw/postgres-17/src/backend/catalog/index.c#L3553-L3856) |
| `REINDEX INDEX CONCURRENTLY` keeps it by moving the row: OID changed 18188 -> 18190, still exactly one `pg_description` row (measured on all six index shapes) | [index.c#index_concurrently_swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1723-L1767) |
| `DROP INDEX` deletes it | [dependency.c#deleteOneObject](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1331-L1333) |
| Any user can read it, so put no secrets in it | [ref/comment.sgml#Notes](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L292-L300) |
| `CREATE TABLE ... (LIKE t INCLUDING ALL)` copies the numeric baseline onto the new, empty index (measured: clone carried `ratio=42.270720 size=2113536` while the clone index was 81,920 bytes with `reltuples = -1`) | [parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1391-L1401) |
| A comment on a *partitioned* parent index is **not** propagated to any child index, because recursive creation hard-codes `idxcomment = NULL` (measured: 0 of 3 children, including one partition added afterwards) | [parse_utilcmd.c#generateClonedIndexStmt](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1576-L1580) |

Upstream tests cover comment survival across `REINDEX TABLE` and `REINDEX TABLE CONCURRENTLY` [create_index.sql:939-948](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L939-L948), but nothing in the tree ties a comment to an index's size or bloat.

A tagged prefix keeps human text intact and is idempotent. Verified on the pin, including a re-stamp that neither duplicated the tag nor lost the sentence after it:

```sql
CREATE OR REPLACE FUNCTION wiki_bpt_stamp(idx regclass) RETURNS numeric
LANGUAGE plpgsql AS $$
DECLARE sz bigint; rt real; relid oid; r numeric; human text; tag text;
BEGIN
    SELECT /* wiki_bpt_stamp */ pg_relation_size(i.indexrelid), c.reltuples, c.oid
      INTO sz, rt, relid
      FROM pg_index i JOIN pg_class c ON c.oid = i.indrelid
     WHERE i.indexrelid = idx;
    IF rt <= 0 THEN
        RAISE EXCEPTION 'reltuples is % for the table behind %; ANALYZE it first', rt, idx;
    END IF;
    r := sz::numeric / rt::numeric;
    human := btrim(regexp_replace(coalesce(obj_description(idx, 'pg_class'), ''),
                                  '\[wiki_bpt v1[^]]*\]', '', 'g'));
    tag := format('[wiki_bpt v1 ratio=%s size=%s reltuples=%s relid=%s at=%s]',
                  round(r, 6), sz, rt, relid, date_trunc('second', now()));
    EXECUTE format('COMMENT ON INDEX %s IS %L', idx::text, btrim(tag || ' ' || human));
    RETURN r;
END $$;
```

The `rt <= 0` guard is mandatory in v17, not defensive. Measured: an empty table read `relpages = 0, reltuples = -1` while its hash index read `relpages = 10, reltuples = 0`; after inserting 200,000 rows without `ANALYZE`, the table was still `-1`; after `TRUNCATE`, the table **and both of its indexes** reset to `relpages = 0, reltuples = -1`; and `REINDEX` of an empty table left `-1` in place. The guard fired with `ERROR: reltuples is -1 for the table behind t_m1_hash; ANALYZE it first`.

Recording `relid` in the tag is what lets a later check notice that the comment was copied to a different table by `INCLUDING ALL`.

### The maintenance-time check

Run this *after* `VACUUM (ANALYZE)`, never before. It refuses to score an index whose baseline was captured for a different relation, and it applies the per-method threshold from the table above. Every `reltuples` test is `<= 0` so the v17 `-1` sentinel cannot produce a negative reading.

```sql
SET statement_timeout = '60s';   -- session scope; use SET LOCAL inside a transaction block
SET lock_timeout = '5s';

SELECT /* wiki_bpt_check */
       c.relname                                        AS index_name,
       am.amname,
       pg_size_pretty(pg_relation_size(c.oid))          AS index_size,
       t.reltuples::numeric                             AS table_reltuples,
       round(b.baseline, 4)                             AS baseline_bytes_per_tuple,
       round(pg_relation_size(c.oid)::numeric
             / nullif(greatest(t.reltuples::numeric, 0), 0), 4) AS now_bytes_per_tuple,
       round(100 * (pg_relation_size(c.oid)::numeric
                    / nullif(greatest(t.reltuples::numeric, 0), 0)
                    / b.baseline - 1), 2)               AS pct_increase,
       CASE am.amname WHEN 'gist' THEN 30 WHEN 'spgist' THEN 30 WHEN 'hash' THEN 10
                      WHEN 'gin' THEN 50 ELSE NULL END  AS threshold_pct,
       CASE
         WHEN am.amname = 'brin' THEN 'ignore: BRIN size does not track bloat'
         WHEN b.relid <> t.oid   THEN 'ignore: baseline was captured on another table'
         WHEN t.reltuples <= 0   THEN 'ignore: ANALYZE the table first'
         WHEN i.indpred IS NOT NULL THEN 'ignore: partial index, wrong denominator'
         WHEN 100 * (pg_relation_size(c.oid)::numeric
                     / nullif(greatest(t.reltuples::numeric, 0), 0) / b.baseline - 1)
              >= CASE am.amname WHEN 'gist' THEN 30 WHEN 'spgist' THEN 30
                                WHEN 'hash' THEN 10 WHEN 'gin' THEN 50 END
           THEN 'candidate'
         ELSE 'ok'
       END                                              AS verdict
FROM pg_index i
JOIN pg_class c  ON c.oid = i.indexrelid
JOIN pg_class t  ON t.oid = i.indrelid
JOIN pg_am   am  ON am.oid = c.relam
CROSS JOIN LATERAL (
    SELECT (regexp_match(obj_description(c.oid, 'pg_class'),
                         '\[wiki_bpt v1 ratio=([0-9.]+)'))[1]::numeric AS baseline,
           (regexp_match(obj_description(c.oid, 'pg_class'),
                         'relid=([0-9]+)'))[1]::oid                    AS relid
) b
WHERE am.amname <> 'btree'
  AND b.baseline IS NOT NULL
  AND c.relkind = 'i'
  AND c.relpersistence = 'p'
ORDER BY 7 DESC NULLS LAST;
```

`indpred` is the partial-index guard; the measured reason for it is below. `relkind = 'i'` excludes partitioned parent indexes, whose `pg_relation_size` is 0 (measured). `pg_relation_size` needs no privileges in this version (there is no ACL check for it in `dbsize.c`, unlike `pg_database_size`), and comments are world readable, so the check runs as any login role; only stamping needs index ownership.

### Index-specific factors

**GIN pending list — the factor that breaks the proxy most convincingly.** With `fastupdate = on` (still the v17 default, [gin_private.h:33](../../../../raw/postgres-17/src/include/access/gin_private.h#L33)), new entries land in a pending list until `nPendingPages * GIN_PAGE_FREESIZE` exceeds the effective limit [ginfast.c#ginHeapTupleFastInsert](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L460). Measured on a 200,000-row baseline followed by 200,000 more rows with `gin_pending_list_limit = 256MB`:

| Stage | Index bytes | Reading vs baseline | Pending pages | Query blocks | Query ms |
|---|---|---|---|---|---|
| Pending list full | 23,232,512 | **+261.73%** | 2,444 | 3,777 | 13.032 |
| After `gin_clean_pending_list()` (returned 2,444 pages) | **31,391,744** | **+388.78%** | 0 | 1,334 | 0.970 |
| After `VACUUM (ANALYZE)` | 31,391,744 | +388.78% | 0 | — | — |
| After `REINDEX` | 8,224,768 | +28.06% | 0 | 1,334 | 1.534 |

Three separate problems in one table. The proxy fires at +261.73%; the correct, cheap fix is a pending-list flush [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091), not a rebuild; and after that flush the reading gets **worse**, because the flush moves entries into the entry and data trees while the 2,444 emptied pending pages only go back to the FSM inside the same file [ginfast.c#shiftList](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L665-L668). The flush alone delivered the entire block-count improvement (3,777 -> 1,334, 2.8x) and 13.4x of the latency; the subsequent rebuild reclaimed 73.8% of the bytes and changed the block count not at all. A `fastupdate = off` control over the same inserts finished at 11,370,496 bytes (+77.04%) and rebuilt to 8,224,768 — the same rebuilt size, from a file 2.8x smaller. So on GIN, `fastupdate` alone moves bytes-per-tuple by a factor of nearly three with no difference in indexed content.

GIN size also is not per row: one heap row yields one entry per array element [ginarrayproc.c#ginarrayextract](../../../../raw/postgres-17/src/backend/access/gin/ginarrayproc.c#L49-L55), and the entry tree is keyed by value, so the numerator scales with distinct keys and posting-list compressibility rather than with the denominator.

**hash — three independent distortions, all reproduced byte for byte.**

1. *The post-build baseline is not a property of the data.* `hashbuild` sizes the initial bucket count from `estimate_rel_size` of the heap, not from the rows it actually scans [hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L133-L137), and that estimator invents "10 pages of average-width rows" when `reltuples < 0` [tableam.c#table_block_relation_estimate_size](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L690-L699). Measured on three tables with identical 200,000 rows: never analyzed before the build gave 930 pages / 896 buckets / **38.0928** bytes per row; analyzed first gave 843 / 768 / **34.5293**; built at one row then filled gave 908 / 652 / **37.1917**. That is a 10.3% spread in the "stable" baseline, which is comparable to the entire recommended threshold of 10%.
2. *Bucket allocation is a power-of-two staircase.* Buckets come in splitpoint groups [hashpage.c#_hash_init_metabuffer](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L509-L526), and a split extends the logical EOF across the whole group [hashpage.c#_hash_expandtable](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L779-L803). Measured fresh builds: 35.226 bytes/row at 60,000 rows, 30.193 at 70,000, **52.634** at 80,000, then falling monotonically to 28.071 at 150,000 before rising again to 34.529 at 200,000 — a 1.87x range with zero bloat anywhere.
3. *`pg_relation_size` counts pages the server never wrote.* `_hash_alloc_buckets` still extends the fork by writing only the last block of the range and relying on the filesystem hole [hashpage.c#_hash_alloc_buckets](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L1030-L1035). Measured after growing 60,000 -> 130,000 rows: 652 pages of which **87 read back as all zeroes**; after a `CHECKPOINT`, `pg_relation_size` said 5,341,184 bytes while `stat` reported 4,628,480 bytes allocated, i.e. exactly 87 x 8192 bytes of hole.

A fourth nuance: for hash, part of what `REINDEX` "reclaims" is re-bucketing, not dead space. After deleting 10% of rows and vacuuming, the index stayed at 6,905,856 bytes and 768 buckets while the metapage tuple count fell to 180,000; the rebuild produced 5,341,184 bytes and **640** buckets because `hashbuild` re-read the now-smaller `reltuples`. That is 22.66% reclaimed with no dead space involved.

**GiST — the one access method whose v17 numbers differ, because the default build path changed.** When every key column has a `GIST_SORTSUPPORT_PROC`, v17 forces `GIST_SORTED_BUILD` [gistbuild.c#GIST_SORTED_BUILD](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L231-L248), and that path **ignores fillfactor**, saying so in the code: `sizeNeeded = IndexTupleSize(itup) + sizeof(ItemIdData); /* fillfactor ignored */` [gistbuild.c#fillfactor-ignored](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L465-L470). The fillfactor computation itself is untouched [gistbuild.c#freespace](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L250-L254) but reaches only the insert and buffering paths. Measured on the 200,000-row `point` fixture:

| `buffering` | fillfactor 100 | 90 | 50 | 10 |
|---|---|---|---|---|
| `auto` (default) | 1,660 pages | 1,660 | 1,660 | 1,660 |
| `off` | 1,660 | 1,660 | 1,660 | 1,660 |
| `on` | 1,633 | 1,775 | 3,158 | 18,824 |

The same sweep on an `int4range` GiST index, whose `range_ops` has no sortsupport function, honours fillfactor on every setting: 1,379 / 1,538 / 2,816 / 15,383 pages. `point_ops` is the only GiST opclass in this tree with a sortsupport entry (measured: 1 for `point_ops`, 0 for `box_ops`, `range_ops`, `tsvector_ops`).

Two consequences for this metric. First, `ALTER INDEX ... SET (fillfactor = ...)` is a no-op for a sorted-built GiST index even after `REINDEX`. Second, a denser fresh build means more of a bloated index is reclaimable, which is exactly what moved every GiST cell: `churn25` went from a reading that was comfortably inside the v12 window to +24.46% at 19.65% reclaimable, 0.35 points under the "worth it" bar, and the `grow*` cells stopped producing double-digit *negative* reclaim. Runtime inserts still pass `freespace = 0` [gist.c#gistinsert](../../../../raw/postgres-17/src/backend/access/gist/gist.c#L183-L186), so post-build inserts fill pages to 100% regardless.

**SP-GiST — placeholders and a visibility gate.** A deleted or moved leaf tuple leaves a redirect or placeholder so later offsets do not shift, and a redirect becomes a placeholder only once its XID is invisible to every snapshot, tested with `GlobalVisTestIsRemovableXid` [spgvacuum.c#redirect-gate](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L528-L538); only trailing placeholders are removed [spgvacuum.c#trailing-placeholders](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L569-L572). Measured: after rewriting the indexed column on all 200,000 rows and before any `VACUUM`, the index read +160.86% with 67.97% of page bytes in use. Because the gate depends on the oldest snapshot in the cluster, the same maintenance window can score differently depending on unrelated long-running transactions.

**BRIN — the numerator has almost nothing to do with rows.** Total size is driven by heap block count divided by `pages_per_range` [brin.sgml#brin-intro](../../../../raw/postgres-17/doc/src/sgml/brin.sgml#L57-L65), with the revmap growing one page per `REVMAP_PAGE_MAXITEMS` ranges [brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-17/src/include/access/brin_page.h#L92-L94). Measured on one 200,000-row table: `pages_per_range = 128` gave 3 pages and 0.12288 bytes/row, `8` gave 4 pages and 0.16384, `1` gave 19 pages and 0.77824 — a 6.3x baseline spread from a reloption. Under churn the index stayed at exactly 24,576 bytes (reading +0.00%) while the same query went from 130 to 487 blocks, and the rebuild brought it only to 386 blocks with 0.00% bytes reclaimed. BRIN degrades in summary *precision*, which this metric cannot see, and the fix for that is `brin_desummarize_range` plus a summarization pass, not `REINDEX`.

v17 can build BRIN indexes in parallel [brin.c#amcanbuildparallel](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L265-L266), which raised the question of whether a fresh BRIN baseline is still deterministic. It is. Measured on a 1,600,000-row / 43,244-block table: 139 pages at `pages_per_range = 1` and 3 pages at the default, byte-identical at `max_parallel_maintenance_workers` = 0, 2 and 4, while a watcher session observed 4 live parallel workers during the `building index` phase and the build ran 70.2 ms with 4 workers against 180.6 ms with 0.

### Space reclaimed is not query performance

The two payoffs the question asks about are only loosely related, and for two access methods they are unrelated.

Restricting to cells where a 30% threshold would fire, and comparing shared-buffer blocks touched by the representative single-index query before and after the rebuild:

| Index type | Cells fired | Cells with any block reduction | Mean block ratio | Max block ratio |
|---|---|---|---|---|
| GIN (either setting) | 19 | 9 | 12.58x | 221.00x |
| SP-GiST | 14 | 13 | 2.52x | 9.10x |
| GiST | 13 | 12 | 2.08x | 10.51x |
| hash | 14 | **0** | 1.00x | 1.00x |
| BRIN | 6 | **0** | 1.00x | 1.00x |

Hash equality lookups touched 1 to 3 blocks regardless of how bloated the index was, so a hash rebuild bought space and nothing else in every cell measured. GiST's minimum block ratio was **0.93x**, i.e. one fired GiST cell needed *more* blocks after the rebuild. And the biggest query wins were not produced by the rebuild at all but by the `VACUUM` that should precede it:

| Index type | Workload | Reclaimed by rebuild | Query blocks before | after |
|---|---|---|---|---|
| GIN | 90% deleted, vacuumed | 82.40% | 3 | 3 |
| GIN | 90% deleted, **not** vacuumed | 89.80% | 663 | 3 |
| GiST | 90% deleted, vacuumed | 90.12% | 13 | 4 |
| GiST | 90% deleted, **not** vacuumed | 90.30% | 515 | 49 |
| SP-GiST | 90% deleted, vacuumed | 89.30% | 37 | 11 |
| SP-GiST | 90% deleted, **not** vacuumed | 89.23% | 755 | 83 |

Rebuild cost on the 200,000-row fixture, for weighing "unnecessary rebuilds": SP-GiST 310.4 ms, GiST 244.4 ms, GIN 193.5 ms, hash 143.1 ms, BRIN 21.1 ms. GiST is no longer the most expensive rebuild in the set; the sorted build moved it below SP-GiST.

`REINDEX INDEX CONCURRENTLY` produced byte-identical results to plain `REINDEX` on all six index shapes (24,576 / 3,211,264 / 3,211,264 / 13,598,720 / 6,905,856 / 12,705,792 bytes), so availability costs nothing in reclaimed space. It refuses system catalogs (`cannot reindex system catalogs concurrently`) and exclusion-constraint indexes (`concurrent index creation for exclusion constraints is not supported`).

### Failure modes that no threshold can fix

- **`reltuples = -1` inverts the sign.** The naive v12 formula returns a negative bytes-per-tuple on a never-analyzed or freshly truncated table, which reads as an improvement. Measured: -7,438,336.00 on a 200,000-row table with a 7,438,336-byte hash index. `TRUNCATE` resets the heap **and every index** to `-1`, so the state is reachable outside first-load.
- **Stale `reltuples` inverts the answer.** The same GiST index at the same instant read **+95.42%** with `reltuples` left at 200,000 after the table doubled, and **-2.29%** one `ANALYZE` later with `reltuples = 400,000`. The index bytes were identical (26,574,848) in both readings.
- **Partial indexes get the wrong denominator, and it hides real bloat.** After `ANALYZE`, a partial index's own `reltuples` is the predicate subset, `ceil(tupleFract * totalrows)` [analyze.c#partial-index-reltuples](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663), while the question's denominator is the whole table. Measured on a GiST index with `WHERE n < 10000` (20,000 of 200,000 rows): the baseline read 6.8813 against the table and 68.8128 against the index. After adding 200,000 rows whose qualifying members were then deleted and vacuumed, the index grew 1,376,256 -> 2,703,360 bytes (+96.4%) with the qualifying row count unchanged at 20,000 — and the metric read **+3.38%**, far below any threshold. The rebuild reclaimed **49.09%**. An independent re-run of the same probe read +3.07% against 48.94% reclaimable. This probe sits outside the 132-cell matrix, so it is not counted in the scoring above; the partial-index miss is systematic rather than marginal, which is why the check excludes `indpred IS NOT NULL` outright.
- **A rebuild does not re-stamp the baseline.** Nothing in `reindex_index` or `index_concurrently_swap` rewrites a comment's contents. Measured immediately after `REINDEX INDEX CONCURRENTLY`, the surviving baselines read -48.86% (GiST), -53.01% (hash), -60.36% (SP-GiST), -71.76% (GIN `fastupdate = off`) and -79.36% (GIN `fastupdate = on`) against the freshly rebuilt index. Re-stamping is a separate step the operator must perform.
- **A fresh rebuild does not return to the baseline.** The GIN case above rebuilt to +28.06% versus its own stored baseline, because a fresh build at 400,000 rows simply has a different bytes-per-tuple than a fresh build at 200,000.
- **`INCLUDING ALL` clones a wrong baseline** onto an empty index, as measured above. The `relid=` guard in the check catches it.
- **The planner never reads a non-partial index's own `reltuples`.** It locks the index's tuple estimate to the parent table's [plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L478), so a stale index `reltuples` moves this metric without moving any plan.
- **Partitioned parents are out of scope, and v17 makes them look plausible.** Measured: a partitioned parent read `relpages = -1, reltuples = 200000` after `ANALYZE`, because v17 writes a partitioned table's `reltuples` from the inherited pass [analyze.c#partitioned-reltuples](../../../../raw/postgres-17/src/backend/commands/analyze.c#L665-L677). Its parent index read `relpages = 0, reltuples = 0` and `pg_relation_size = 0`, so the ratio is 0 and the comment cannot be compared to anything. Apply the scheme per leaf index.
- **VACUUM can silently defer index cleanup.** When fewer than 2% of heap pages hold dead item pointers, v17 skips `ambulkdelete` entirely [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1929-L1934). Measured on a 10,811-page heap: deleting 1,500 rows produced `index scan bypassed: 41 pages from table (0.38% of total) have 1500 dead item identifiers`, and a later `VACUUM (INDEX_CLEANUP ON)` after deleting 1,500 more removed **3,000** dead item identifiers, i.e. the first pass's entries were still there. Per pass the deferral is bounded by 2% of pages, but it accumulates and is invisible to this metric.

### Operational recipe

1. Stamp the baseline immediately after each build or rebuild, on a table that has just been analyzed. For hash, analyze *before* the build too, or the bucket count is sized from a guess.
2. At maintenance time, run `VACUUM (ANALYZE)` first, then read the ratio. Skipping `ANALYZE` produces readings that are wrong by tens of points; skipping `VACUUM` inflates both the reading and the apparent rebuild benefit.
3. Refuse to score any index whose table reads `reltuples <= 0`. In v17 that value is `-1` for "unknown", and dividing by it silently flips the sign.
4. For GIN, run `SELECT gin_clean_pending_list('idx')` (or rely on the `VACUUM`, which flushes the list) and only then compare. A firing that survives a flush is the only GIN firing worth acting on.
5. Apply the per-method threshold from the table above; treat BRIN as out of scope for this signal.
6. Before rebuilding, confirm the row count is comparable to the baseline's `reltuples=` field. If the table has grown by more than roughly 25%, re-stamp instead of rebuilding: for hash, SP-GiST and GIN the growth term alone can exceed the threshold.
7. Rebuild with `REINDEX INDEX CONCURRENTLY` if availability matters; it preserves the comment and reaches the same size, as measured.
8. Re-stamp after the rebuild, and record the achieved reclaim so the threshold can be revisited with local data.

### Settings and their apply scope

Determined from the same-version GUC and reloption definitions:

| Setting | Kind | Default | Scope needed to change it |
|---|---|---|---|
| `gin_pending_list_limit` | GUC, `PGC_USERSET`, kB [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3575-L3584) | 4096 kB | session/transaction (`SET`); also settable per index |
| `fastupdate` | GIN reloption [reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131) | `true` | `ALTER INDEX`, takes `AccessExclusiveLock`; does not flush the existing list |
| `gin_pending_list_limit` | GIN reloption [reloptions.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L339-L347) | -1 (use the GUC) | `ALTER INDEX`, `AccessExclusiveLock` |
| `pages_per_range` | BRIN reloption [reloptions.c#pages_per_range](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L331-L338) | 128 [brin.h:39](../../../../raw/postgres-17/src/include/access/brin.h#L39) | `ALTER INDEX`, `AccessExclusiveLock`; only a rebuild re-lays the index out |
| `autosummarize` | BRIN reloption [reloptions.c#autosummarize](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L96-L104) | `false` | `ALTER INDEX`, `AccessExclusiveLock` |
| `buffering` | GiST reloption [reloptions.c#buffering](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L521-L531) | `auto` | `ALTER INDEX`, `AccessExclusiveLock`; setting it `on` is what makes GiST honour fillfactor at the next rebuild |
| `fillfactor` | hash 75, GiST 90, SP-GiST 80 [hash.h:295-296](../../../../raw/postgres-17/src/include/access/hash.h#L295-L296), [gist_private.h:479-480](../../../../raw/postgres-17/src/include/access/gist_private.h#L479-L480), [spgist_private.h:494-495](../../../../raw/postgres-17/src/include/access/spgist_private.h#L494-L495) | — | `ALTER INDEX`, `ShareUpdateExclusiveLock`; ignored by a sorted GiST build, and hash writes it into the metapage at build time only |
| `max_parallel_maintenance_workers` | GUC, `PGC_USERSET` | 2 | session/transaction; affects BRIN build time but not BRIN size (measured) |
| `statement_timeout`, `lock_timeout` | GUC, `PGC_USERSET` | 0 | session/transaction |

GIN and BRIN accept no `fillfactor` at all; an unknown index parameter raises `unrecognized parameter "..."` [reloptions.c#unrecognized-parameter](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L1466-L1478). No `postmaster`-context (restart) or `sighup`-context (reload) setting is required by anything on this page.

### What changed since PostgreSQL 12

Each row names the commit that changed the behavior and the first release tag in this checkout that contains it. Every hash listed is an ancestor of this page's pin.

| Change | Effect on this metric | Commit | First release |
|---|---|---|---|
| `pg_class.reltuples` gained the `-1` "unknown" sentinel | The v12 formula can now return a **negative** bytes-per-tuple. Every guard must test `<= 0` | `3d351d916b2` | 14.0 |
| `table_block_relation_estimate_size`'s "10 pages" heuristic keys on `reltuples < 0` instead of `relpages == 0` | An empty-but-analyzed table no longer gets inflated to 10 pages, so hash's build-time bucket guess differs in that one case | `3d351d916b2` | 14.0 |
| VACUUM bypasses index vacuuming below 2% of pages with dead items | Dead index entries and their pages accumulate silently across passes | `5100010ee4d` | 14.0 |
| `vac_estimate_reltuples` returns the old value when the scan covered <2% of pages, or <= 1 page | The denominator can stay frozen, including at `-1` | `74388a1ac36`, `3097bde7dd1` | 15.0, 16.0 |
| GiST sorted build (`GIST_SORTED_BUILD`) plus `gist_point_sortsupport` | The default build path for sortsupport opclasses **ignores fillfactor** and packs denser, which is why every GiST cell moved | `16fa9b2b30a` | 14.0 |
| Sorted-build page batching with the opclass picksplit; adds the `/* fillfactor ignored */` comment | Same | `f1ea98a7975` | 15.0 |
| GiST sorted build writes through the new bulk-write API | Rebuild is faster (measured 244.4 ms vs SP-GiST's 310.4 ms) and byte-deterministic | `8af25652489` | 17.0 |
| GiST recycles empty pages in every VACUUM pass, not only at cleanup | Freed GiST pages reach the FSM sooner | `4e514c6180f` | 13.0 |
| `RecentGlobalXmin` replaced by `GlobalVisState` in the GiST, GIN and SP-GiST recyclability gates | Same conclusion, different mechanism; horizons are now computed per relation | `dc7420c2c92` | 14.0 |
| SP-GiST passes the real heap relation to the visibility test | Redirects become placeholders sooner, so SP-GiST dead space is reclaimable earlier | `05a304a8551` | 16.0 |
| Parallel BRIN build and the `amcanbuildparallel` AM flag | Build time changes; measured index size does not | `b4375717147` | 17.0 |
| `CREATE INDEX` adds empty BRIN ranges | Slightly larger fresh BRIN index | `dae761a87ed` | 17.0 |
| `pgstattuple`/`pgstatindex`/`gin_clean_pending_list` reject or no-op on `indisvalid = false` | An interrupted concurrent build cannot be measured with these tools | `13503eb5905` | 17.0 |
| `pgstattuple` counts a new/empty page as free space for hash and GiST instead of erroring | `pgstattuple()` now works on a hash index containing a splitpoint hole. Measured: 0 of 21 hash cells errored | `036decbba2a` | 17.7 |
| `ANALYZE` writes a partitioned parent's `reltuples` (and `relpages = -1`) | A partitioned parent now looks like it has statistics while its parent index still has no storage | `0e69f705cc1`, `375aed36ad8` | 14.0 |
| `COMMENT`'s `SHARE UPDATE EXCLUSIVE` lock is documented | Documentation only; the lock itself is unchanged | `b2a76bb7d05` | 15.0 |
| `PGAlignedBlock` -> `PGIOAlignedBlock` in `_hash_alloc_buckets` | None; the sparse-file trick is intact | `faeedbcefd4` | 16.0 |

What did **not** change: `indexfsm.c` has zero executable-code changes since `REL_12_0` (the only diff is a comment correction, `2afdb9dd96f`, 17.5); `ginfast.c`'s pending-list flush threshold and `shiftList` FSM handling are untouched; `_hash_alloc_buckets` is functionally identical; `brinvacuumcleanup` differs by one `palloc0_object` call; hash and BRIN still never return pages to the index FSM; and GIN builds are still serial (`amcanbuildparallel = false`), so a fresh GIN index is deterministic. Parallel GIN build is `8492feb98f6`, first released in 18.0, and is **not** an ancestor of this pin.

## Test Summary

### Environment and protocol

- Server built from this checkout only, under `.wiki-runtime/pg17/install`; `postgres (PostgreSQL) 17.10`, gcc 13.3.0, x86-64 Linux under WSL2, 8192-byte blocks. contrib `pageinspect`, `pgstattuple` and `pg_freespacemap` installed from the same tree with `USE_PGXS=1` out of tree, leaving `raw/postgres-17/` unmodified.
- Isolated throwaway cluster on port 55417 with a private socket directory under `.wiki-runtime/tmp/bpt17/`. `shared_buffers = 512MB`, `maintenance_work_mem = 256MB`, `work_mem = 64MB`, `max_wal_size = 4GB`, `fsync = off`, `synchronous_commit = off`, `full_page_writes = off`, **`autovacuum = off`**, `track_io_timing = on`, **`max_parallel_maintenance_workers = 0`**.
- `max_parallel_maintenance_workers = 0` is new relative to the v12 protocol, because v17 can build BRIN indexes in parallel and the matrix must be reproducible. A dedicated probe re-enables it.
- Autovacuum was disabled so that each cell's `VACUUM`/`ANALYZE` boundary is exactly where the script puts it. In production, autovacuum is what keeps the denominator fresh; the "stale `reltuples`" probe measures what happens when it does not.
- Per-cell protocol: build the table, load rows, `VACUUM (ANALYZE)`, create the six indexes, stamp each baseline via `COMMENT ON INDEX`, run the workload, reach the maintenance point (`VACUUM (ANALYZE)` unless the cell name says otherwise), read the ratio and the access-method internals, measure the query, `REINDEX INDEX`, measure size and query again, record one row per index.
- Ground truth for space is `pg_relation_size` before and after `REINDEX INDEX` on the same index. Ground truth for query work is the root-node shared blocks (hit + read) from `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` with `enable_seqscan = off`, median execution time over 7 runs, warm cache.
- Page density for every access method came from `page_header(get_raw_page(...))` over all pages, treating `pd_lower = 0` as an unwritten page, because `pgstattuple()` opens only btree, hash and GiST indexes in this version [pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L269-L296).

### Fixture and index shapes

One table per cell, every column a pure function of `id` so a rerun reproduces the same bytes: `n = (id*7919) % 100000` (hash key), `b = id` (BRIN key, perfectly correlated), `p = point((id*7919)%100000/1e5, (id*104729)%100000/1e5)` (GiST key), `tags`/`tags2 = ARRAY[(id*3)%1000, (id*7)%1000, (id*11)%1000, (id*13)%1000, (id*17)%1000]` (GIN keys, 5 entries per row over a 1,000-value vocabulary), `s = md5(id::text)` (SP-GiST radix key), `pad = repeat('x',40)` (never indexed). Baseline scale 200,000 rows, heap 5,406 pages.

Six indexes per table, covering all five core non-btree access methods plus the GIN pending-list variable: `hash (n)`, `gist (p)`, `gin (tags) WITH (fastupdate = off)`, `gin (tags2) WITH (fastupdate = on)`, `spgist (s)`, `brin (b)`. The two GIN indexes are on separate columns so each query can only choose one of them. Fresh sizes at 200,000 rows: hash 843 pages, GiST **1,660**, GIN 392, SP-GiST 1,551, BRIN 3. Page occupancy at the `steady` control: hash 58.54%, GiST 65.74%, GIN 68.99%, SP-GiST 86.48%, BRIN 3.96%.

Representative queries, one per index: `n = 7919` (hash, index scan), `p <@ box '((0.40,0.40),(0.45,0.45))'` (GiST), `tags @> ARRAY[42]` and `tags2 @> ARRAY[42]` (GIN), `s >= 'ab' AND s < 'ac'` (SP-GiST), `b BETWEEN 100000 AND 100500` (BRIN).

### Workloads

22 labels, 132 cells. All maintenance points ran `VACUUM (ANALYZE)` except `del90_novac` (`ANALYZE` only) and the mid-workload vacuums noted below.

| Label | What it does after the baseline |
|---|---|
| `steady` | nothing; the control |
| `del10` ... `del99` | delete 10 / 25 / 50 / 75 / 90 / 99% of rows, spread evenly over all heap pages |
| `del90_novac` | delete 90%, then `ANALYZE` without `VACUUM` |
| `churn25` / `churn50` / `churn100` | `UPDATE ... SET n = n + 1` on 25 / 50 / 100% of rows (non-HOT: an indexed column changes, so all six indexes get new entries) |
| `churn200` / `churn400` | 2 / 4 full passes of the same update |
| `hot` | `fillfactor = 70` table, two full passes updating only `pad`, so most updates are HOT |
| `grow2x` / `grow4x` / `grow8x` | insert to 2x / 4x / 8x the baseline row count, no updates or deletes |
| `turnover` | delete half the rows, `VACUUM`, re-insert the same number |
| `mixed` | three rounds of +40,000 inserts, 20% indexed-column updates, 10% deletes, with a `VACUUM` between rounds |
| `grow2x+churn50` | insert to 2x, then update the indexed column on half the rows |

`churn` and `grow` are independent re-runs of `churn200` and `grow4x` respectively, run separately, and are counted in the 132 cells but not shown as separate rows below.

### Result matrix

Each cell is `pct_increase / reclaimed_pct`. `pct_increase` is the reading the proxy produces at the maintenance point; `reclaimed_pct` is what `REINDEX INDEX` actually recovered.

| Workload | hash | GiST | GIN fu=off | GIN fu=on | SP-GiST | BRIN |
|---|---|---|---|---|---|---|
| `steady` | 0.00 / 0.00 | 0.00 / 0.00 | 0.00 / 0.00 | 0.00 / 0.00 | 0.00 / 0.00 | 0.00 / 0.00 |
| `del10` | 11.11 / 22.66 | 11.11 / 12.95 | 11.11 / 8.42 | 11.11 / 8.42 | 11.11 / 10.12 | 11.11 / 0.00 |
| `del25` | 33.33 / 39.03 | 33.33 / 26.81 | 33.33 / 17.35 | 33.33 / 17.35 | 33.33 / 26.11 | 33.33 / 0.00 |
| `del50` | 100.00 / 39.03 | 100.00 / 50.36 | 100.00 / 39.03 | 100.00 / 39.03 | 100.00 / 51.84 | 100.00 / 0.00 |
| `del75` | 300.00 / 69.40 | 300.00 / 74.82 | 300.00 / 62.24 | 300.00 / 62.24 | 300.00 / 75.56 | 300.00 / 0.00 |
| `del90` | 900.00 / 84.58 | 900.00 / 90.12 | 900.00 / 82.40 | 900.00 / 82.40 | 900.00 / 89.30 | 900.00 / 0.00 |
| `del90_novac` | 900.00 / 84.58 | 900.00 / 90.30 | 900.00 / 89.80 | 900.00 / 89.80 | 900.00 / 89.23 | 900.00 / 0.00 |
| `del99` | 9900.00 / 98.81 | 9900.00 / 99.04 | 9900.00 / 97.70 | 9900.00 / 97.70 | 9900.00 / 98.52 | 9900.00 / 0.00 |
| `churn25` | 35.47 / 25.22 | **24.46 / 19.65** | 62.24 / 41.04 | 192.60 / 67.31 | 32.17 / 23.95 | 0.00 / 0.00 |
| `churn50` | 50.89 / 32.47 | 42.17 / 29.66 | 126.53 / 58.67 | 255.87 / 73.69 | 53.58 / 34.63 | 0.00 / 0.00 |
| `churn100` | 112.81 / 53.01 | 95.00 / 48.72 | 254.08 / 71.76 | 384.44 / 79.36 | 152.29 / 60.36 | 0.00 / 0.00 |
| `churn200` | 197.15 / 66.35 | 185.36 / 64.96 | 254.59 / 71.80 | 385.97 / 79.42 | 240.94 / 70.67 | 0.00 / 0.00 |
| `churn400` | 421.35 / 80.82 | 376.69 / 79.02 | 761.22 / 88.45 | 888.52 / 89.94 | 381.56 / 79.23 | 0.00 / 0.00 |
| `hot` | 81.38 / 44.87 | 72.59 / 42.06 | 246.11 / 55.39 | 379.02 / 67.77 | 105.35 / 50.96 | 0.00 / 0.00 |
| `grow2x` | 6.41 / 4.24 | -2.35 / -1.26 | 77.04 / 27.67 | 142.22 / 47.13 | 28.24 / -0.38 | -50.00 / 0.00 |
| `grow4x` | 8.30 / 5.34 | -2.74 / 0.45 | 111.10 / 14.50 | 133.23 / 22.61 | -3.35 / 0.28 | -75.00 / 0.00 |
| `grow8x` | 8.78 / 5.36 | -3.01 / -1.63 | 39.41 / 9.38 | 55.84 / 18.93 | -4.11 / 0.72 | -87.50 / 0.00 |
| `turnover` | 0.00 / 0.12 | 9.94 / 10.14 | 125.51 / 29.64 | 245.66 / 54.10 | 3.61 / 2.92 | 0.00 / 0.00 |
| `mixed` | 53.51 / 25.93 | 42.89 / 31.30 | 198.12 / 52.16 | 308.89 / 65.12 | 56.71 / 29.99 | -15.25 / 0.00 |
| `grow2x+churn50` | 48.70 / 30.08 | 42.74 / 30.72 | 169.13 / 52.56 | 198.60 / 57.24 | 72.86 / 25.66 | -50.00 / 0.00 |

Notes on reading it: the hash, GIN, SP-GiST and BRIN columns are identical to the v12 page's matrix cell for cell; every GiST cell differs. Negative `reclaimed_pct` means the rebuild produced a *larger* index (GiST at `grow2x` and `grow8x`, SP-GiST at `grow2x`); v17 has two such GiST cells where the v12 page lists four, and no GiST `turnover` regression. `steady` proves the metric is quiet when nothing happens, and the GiST `steady` rebuild reclaims exactly 0.00% here where the v12 page reports 0.17%, because the sorted build is byte-deterministic. `del*` readings are identical across access methods by construction, since only the denominator moved. `VACUUM` never shrank any index: `base_size` and `now_size` were byte-identical in all 36 delete cells, for all six index shapes. The bold GiST `churn25` cell is the one that sets the lower edge of the GiST window.

### Threshold scoring

Positive class: `REINDEX` reclaims at least 20% of current bytes. Grid searched at 1-point resolution from 0 to 300.

| Index type | Best threshold range | TP | FP | FN | TN | Accuracy |
|---|---|---|---|---|---|---|
| hash | 9 - 11 | 16 | 0 | 0 | 6 | 100.0% |
| GiST | 25 - 33 | 14 | 0 | 0 | 8 | 100.0% |
| SP-GiST | 29 - 32 | 15 | 0 | 0 | 7 | 100.0% |
| GIN `fastupdate = on` | 56 - 100 | 18 | 0 | 0 | 4 | 100.0% |
| GIN `fastupdate = off` | 40 - 62 | 16 | 2 | 0 | 4 | 90.9% |
| BRIN | 101 - 300 | 0 | 4 | 0 | 18 | 81.8% |

Only the GiST row differs from the v12 page's table, whose best GiST range was 14-33. Separability at other "worth it" bars, as (highest negative reading, lowest positive reading): at 20% GiST is separable (24.46, 33.33), hash (8.78, 11.11), SP-GiST (28.24, 32.17), GIN `fastupdate = on` (55.84, 100.00); at 30% GiST is still separable (42.17, 42.74) with 0.57 points of margin while hash and SP-GiST overlap; at 50% every access method except BRIN is separable again. The two GIN `fastupdate = off` false positives are the two independent runs of the same 4x-growth workload (`grow4x` and `grow`), both reading +111.10% against 14.50% reclaimable; the four BRIN false positives at its "best" threshold are `del75`, `del90`, `del90_novac` and `del99`, whose readings exceed any finite bound.

A single all-method threshold, over all 132 cells: 20% -> 88.6% accuracy (14 FP, 1 FN); **30% -> 90.2%** (12 FP, 1 FN); 50% -> 86.4% (8 FP, 10 FN); 100% -> 79.5% (7 FP, 20 FN). Excluding BRIN: 20% -> 91.8%, **30% -> 93.6%** (6 FP, 1 FN), 50% -> 88.2%, 100% -> 80.0%.

### Scale sweep

Fresh builds at seven scales, identical fixture, `ANALYZE` before each build, no bloat present anywhere. Bytes per table tuple:

| Rows | Heap pages | hash | GiST | GIN | SP-GiST | BRIN |
|---|---|---|---|---|---|---|
| 25,000 | 676 | 42.598 | 69.468 | 23.265 | 84.541 | 0.983 |
| 50,000 | 1,352 | 42.271 | 66.683 | 20.808 | 62.259 | 0.492 |
| 100,000 | 2,703 | 42.107 | 66.765 | 18.268 | 60.703 | 0.246 |
| 200,000 | 5,406 | 34.529 | 67.994 | 16.056 | 63.529 | 0.123 |
| 400,000 | 10,811 | 35.185 | 67.236 | 20.562 | 81.777 | 0.061 |
| 800,000 | 21,622 | 35.400 | 65.833 | 28.979 | 61.225 | 0.031 |
| 1,600,000 | 43,244 | 35.548 | 67.021 | 20.285 | 60.477 | 0.015 |

Drift of each fresh build against the 200,000-row fresh build, in percent — this is the false-positive floor a stored baseline inherits from growth alone:

| Rows | hash | GiST | GIN | SP-GiST | BRIN |
|---|---|---|---|---|---|
| 25,000 | +23.37 | +2.17 | +44.90 | +33.08 | +700.00 |
| 50,000 | +22.42 | -1.93 | +29.59 | -2.00 | +300.00 |
| 100,000 | +21.95 | -1.81 | +13.78 | -4.45 | +100.00 |
| 400,000 | +1.90 | -1.11 | +28.06 | +28.72 | -50.00 |
| 800,000 | +2.52 | -3.18 | +80.48 | -3.63 | -75.00 |
| 1,600,000 | +2.95 | -1.43 | +26.34 | -4.80 | -87.50 |

Every hash, GIN, SP-GiST and BRIN figure matches the v12 page's sweep exactly; GiST's band is -3.18% to +2.17% where the v12 page reports -0.29% to +3.84%.

Index page counts over the same scales show why: GiST is close to linear (212 / 407 / 815 / 1,660 / 3,283 / 6,429 / 13,090), hash is a staircase (130 / 258 / 514 / 843 / 1,718 / 3,457 / 6,943), SP-GiST is non-monotone in density (258 / 380 / 741 / 1,551 / 3,993 / 5,979 / 11,812), GIN is sub- then super-linear (71 / 127 / 223 / 392 / 1,004 / 2,830 / 3,962), and BRIN is constant at 3 pages. The GIN 400,000 and 800,000 points were re-measured on independent fixtures and reproduced exactly (1,004 pages / 20.5619 and 2,830 pages / 28.9792, with `n_entry_pages` 7 and `n_data_pages` 996 and 2,822 respectively, `n_entries` 1,000 in both). Note that this fixture holds distinct-key count fixed at 1,000 while rows grow, which is the regime where GIN size is dominated by posting-tree data pages.

### Access-method probes

Beyond the matrix, each measurement below was run as its own fixture:

- **GIN pending list**: four-stage table in [Index-specific factors](#index-specific-factors), plus the `fastupdate = off` control.
- **hash baseline instability**: three build paths, same 200,000 rows, 34.5293 / 38.0928 / 37.1917 bytes per row; heap `relpages`/`reltuples` after the build 5,406/200,000, 5,406/200,000 and 1/1.
- **hash splitpoint staircase**: 11 fresh builds from 60,000 to 250,000 rows, bytes per row 28.071 to 52.634, buckets 256 -> 896, zero unwritten pages in any fresh build.
- **hash sparse hole**: 652 logical pages, 87 unwritten, `pg_relation_size` 5,341,184 vs 4,628,480 bytes allocated per `stat` after `CHECKPOINT`; `pgstattuple()` accepted the index and reported 38.94% free, and `pgstathashindex` counted 138 unused pages.
- **hash re-bucketing**: 768 buckets before, 640 after the rebuild, 22.66% "reclaimed" from a 10% delete, metapage `ntuples` 200,000 -> 180,000.
- **stale and `-1` `reltuples`**: +95.42% vs -2.29% for the same GiST index; naive ratio -7,438,336.00 on a never-analyzed 200,000-row table; `TRUNCATE` resetting heap and both indexes to `-1`.
- **partial index**: +96.4% real growth reported as +3.38%, 49.09% actually reclaimable; independent re-run +3.07% against 48.94%.
- **SP-GiST pre-vacuum churn**: +160.86% with 67.97% page-byte occupancy.
- **BRIN reloption and precision**: 3 / 4 / 19 pages at `pages_per_range` 128 / 8 / 1; 130 -> 487 -> 386 query blocks across churn and rebuild at 0.00% bytes changed.
- **rebuild cost**: SP-GiST 310.4 ms, GiST 244.4 ms, GIN 193.5 / 191.4 ms, hash 143.1 ms, BRIN 21.1 ms at 200,000 rows.

### v17-specific probes

- **GiST fillfactor versus the sorted build**: the 3 x 4 table in [Index-specific factors](#index-specific-factors), plus a `range_ops` control and a `pg_amproc` count showing `point_ops` is the only sortsupport-capable GiST opclass among `point_ops`, `box_ops`, `range_ops` and `tsvector_ops`.
- **Parallel BRIN build**: 139 pages (`pages_per_range = 1`) and 3 pages (default) identical at 0, 2 and 4 workers on a 1,600,000-row table; a watcher session sampling `pg_stat_activity` and `pg_stat_progress_create_index` with `pg_stat_clear_snapshot()` before each read saw 4 live parallel workers in the `building index` phase over 43,244 blocks; build time 70.2 ms with 4 workers against 180.6 ms with 0.
- **2% index-vacuum bypass**: `index scan bypassed: 41 pages from table (0.38% of total) have 1500 dead item identifiers` on a 10,811-page heap, followed by a second pass that removed 3,000 dead item identifiers after only 1,500 more rows were deleted.
- **`REINDEX INDEX CONCURRENTLY`**: byte-identical to plain `REINDEX` on all six index shapes; OID changed on all six; exactly one `pg_description` row survived on each; the surviving baseline read -48.86% to -79.36% against the rebuilt index. Refusals reproduced for system catalogs and exclusion-constraint indexes.
- **Partitioned relations**: parent table `relpages = -1`, `reltuples = 200000`; parent index `relpages = 0`, `reltuples = 0`, `pg_relation_size = 0`, naive ratio 0; the parent index comment propagated to 0 of 3 child indexes, including a partition created after the comment was set.
- **`pgstattuple` support boundary**: hash and GiST accepted; `index "t_st_gin" (gin index) is not supported`, and the same for `spgist` and `brin`; `pgstatindex` on a hash index gives `relation "t_st_hash" is not a btree index`; `pgstatginindex` and `pgstathashindex` work.

### COMMENT mechanics checked on the pin

Every row of the table in [Recording the baseline with COMMENT ON INDEX](#recording-the-baseline-with-comment-on-index) was executed: the two syntax errors for a non-literal comment, tag-plus-human-text round-tripping and idempotent re-stamping, the `pg_description` row shape (`classoid = pg_class`, `objsubid = 0`), OID stability across plain `REINDEX` (18188 -> 18188) and OID change with comment survival across `REINDEX INDEX CONCURRENTLY` (18188 -> 18190, one row), deletion by `IS NULL` and by `IS ''`, the single `ShareUpdateExclusiveLock` on the index, a non-owner login role reading the baseline but failing to write it (`must be owner of index`), `LIKE ... INCLUDING ALL` copying the numeric baseline onto an 81,920-byte empty clone whose table read `reltuples = -1`, and the partitioned-parent non-propagation.

### Reproducibility and cleanup

Two cells (`churn100`, `del50`) were re-run end to end after the first pass: 11 of 12 index-level results were identical to the digit, including byte-identical index sizes. The exception is GiST `churn100` (95.00% vs 95.12% reading, 48.72% vs 48.75% reclaimed), where the *bloated* size differed by a few pages while the *rebuilt* size was byte-identical — all 12 `reindex_size` values matched exactly, which is the sorted build being deterministic where the runtime insert path is not. The duplicated pairs in the matrix (`churn`/`churn200` and `grow`/`grow4x`, run separately) agreed exactly for five of six index shapes, GiST again being the exception (185.54 vs 185.36, and -2.55 vs -2.74).

The test server was stopped and its data directory removed after the run; the SQL harness and captured output remain under `.wiki-runtime/tmp/bpt17/`. `raw/postgres-17/` was never modified, confirmed with `git status --porcelain`.

## Context Reviewed

- Comment storage and lifecycle: `comment.c`, `gram.y` (`CommentStmt`, `comment_text`), `objectaddress.c` (address resolution, ownership), `aclchk.c` (error text), `dependency.c` (deletion), `index.c` (`index_concurrently_swap`, `reindex_index`, `index_concurrently_create_copy`), `indexcmds.c` (`DefineIndex`, `idxcomment`), `parse_utilcmd.c` (`transformTableLikeClause`, `generateClonedIndexStmt`), `tablecmds.c` (`ATExecAttachPartitionIdx`, `ATPostAlterTypeParse`), `pg_description.h`, `system_functions.sql` and `pg_proc.dat` (`obj_description`), `pg_dump.c` / `pg_backup_archiver.c` (`--no-comments`), `ref/comment.sgml`, `create_index.sql` / `create_index.out`, `create_table_like.sql`, `alter_table.sql`.
- Size and statistics: `dbsize.c`, `system_functions.sql` and `pg_proc.dat` (`pg_relation_size`), `pg_class.h`, `heap.c` (`AddNewRelationTuple`, `RelationTruncateIndexes`), `relcache.c` (`RelationSetNewRelfilenumber`, `formrdesc`), `vacuum.c` (`vac_update_relstats`, `vac_estimate_reltuples`), `vacuumlazy.c` (`heap_vacuum_rel`, `lazy_vacuum`, `lazy_cleanup_all_indexes`, `update_relstats_all_indexes`, `should_attempt_truncation`, `lazy_truncate_heap`, `BYPASS_THRESHOLD_PAGES`), `analyze.c` (`analyze_rel`, `do_analyze_rel`, `compute_index_stats`, `acquire_sample_rows`), `index.c` (`index_update_stats`, `index_build`), `heapam_handler.c` and `tableam.c` (`heapam_estimate_rel_size`, `table_block_relation_estimate_size`), `plancat.c` (`get_relation_info`, `estimate_rel_size`), `autovacuum.c` (`relation_needs_vacanalyze`), `storage.c` (`RelationTruncate` call sites), `catalogs.sgml`, `perform.sgml`, `planstats.sgml`, `func.sgml`.
- Access methods: `hash.c`, `hashpage.c`, `hashovfl.c`, `hash.h`, `hash/README`; `gist.c`, `gistbuild.c`, `gistvacuum.c`, `gistutil.c`, `gistproc.c`, `gist_private.h`, `gistxlog.c`; `ginfast.c`, `ginvacuum.c`, `ginutil.c`, `gininsert.c`, `ginarrayproc.c`, `ginblock.h`, `gin_private.h`; `spgvacuum.c`, `spgutils.c`, `spgist_private.h`; `brin.c`, `brin_page.h`, `brin.h`, `brin.sgml`; `amapi.h` (`amoptions`, `amcanbuildparallel`); `procarray.c` (`GlobalVisTestFor`, `GlobalVisCheckRemovableFullXid`).
- Shared infrastructure: `indexfsm.c`, `freespace.c`, `reloptions.c`, `guc_tables.c`, `planner.c` (`plan_create_index_workers`), `maintenance.sgml`, `ref/reindex.sgml`, `ref/create_index.sgml`.
- Tooling boundaries: `contrib/pgstattuple` (`pgstattuple.c`, `pgstatindex.c`, `pgstatapprox.c`, its SQL scripts and expected output), `contrib/pageinspect`, `contrib/pg_freespacemap`.
- Test surfaces: `src/test/regress/sql/{create_index,create_table_like,alter_table,reloptions,vacuum}.sql` and their expected files, `contrib/pgstattuple/expected/pgstattuple.out`.
- Source history: `git log`/`git tag --contains`/`git merge-base` over `REL_12_0..54eeefaedbee` for `pg_class.h`, `vacuum.c`, `vacuumlazy.c`, `analyze.c`, `gistbuild.c`, `gistvacuum.c`, `gistutil.c`, `ginvacuum.c`, `ginfast.c`, `spgvacuum.c`, `brin.c`, `hashpage.c`, `indexfsm.c`, `comment.sgml` and `contrib/pgstattuple`.

## Evidence Map

| Claim | Evidence |
|---|---|
| `COMMENT ON INDEX` accepts only a literal or `NULL` | [gram.y#comment_text](../../../../raw/postgres-17/src/backend/parser/gram.y#L7219-L7222) + measured syntax errors |
| It locks the index with `ShareUpdateExclusiveLock`, and v17 documents it | [comment.c#CommentObject](../../../../raw/postgres-17/src/backend/commands/comment.c#L66-L73), [ref/comment.sgml#lock](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L95-L98) + measured `pg_locks` row |
| Empty string is folded to `NULL`, which deletes the row | [comment.c:155-157](../../../../raw/postgres-17/src/backend/commands/comment.c#L155-L157), [comment.c#CreateComments](../../../../raw/postgres-17/src/backend/commands/comment.c#L142-L226) |
| Index ownership is required to write a comment | [objectaddress.c#check_object_ownership](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c#L2396-L2401) + measured error |
| `REINDEX ... CONCURRENTLY` moves the comment to the new OID | [index.c#index_concurrently_swap](../../../../raw/postgres-17/src/backend/catalog/index.c#L1723-L1767), [create_index.sql:939-948](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql#L939-L948) + measured OID change on six shapes |
| `DROP INDEX` removes the comment | [dependency.c#deleteOneObject](../../../../raw/postgres-17/src/backend/catalog/dependency.c#L1331-L1333) |
| `LIKE ... INCLUDING ALL` copies index comments | [parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1391-L1401) + measured clone |
| A partitioned parent index's comment is not propagated to children | [parse_utilcmd.c#generateClonedIndexStmt](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c#L1576-L1580) + measured 0 of 3 children |
| Comments are readable by any user | [ref/comment.sgml#Notes](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml#L292-L300) + measured non-owner read |
| `obj_description`'s second argument is `name`, and it selects `objsubid = 0` | [system_functions.sql#obj_description](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L291-L301) |
| `pg_relation_size(idx)` measures the main fork only | [pg_proc.dat#pg_relation_size](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7491), [system_functions.sql#pg_relation_size](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql#L285-L289), [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L343) |
| `reltuples` is a `float4` estimate whose `-1` means "unknown" in v17 | [pg_class.h#reltuples](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L62-L66) + measured -1 states |
| `VACUUM` may keep the old `reltuples`, possibly `-1` | [vacuum.c#vac_estimate_reltuples](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1329-L1354), [vacuum.c#vac_update_relstats](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L1444-L1461) |
| `ANALYZE` extrapolates from a block sample | [analyze.c#acquire_sample_rows](../../../../raw/postgres-17/src/backend/commands/analyze.c#L1285-L1289) |
| An index build rewrites the heap's `relpages`/`reltuples`, and keeps `-1` on an empty table | [index.c#index_build](../../../../raw/postgres-17/src/backend/catalog/index.c#L3097-L3106), [index.c#index_update_stats-hack](../../../../raw/postgres-17/src/backend/catalog/index.c#L2796-L2807) |
| An index's `reltuples` refresh depends on the AM reporting an exact count | [vacuumlazy.c#update_relstats_all_indexes](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3084-L3099) |
| VACUUM skips index vacuuming below 2% of pages with dead items | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#bypass](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1929-L1934) + measured `index scan bypassed` |
| A partial index's own `reltuples` is the predicate subset | [analyze.c#partial-index-reltuples](../../../../raw/postgres-17/src/backend/commands/analyze.c#L647-L663) |
| ANALYZE writes a partitioned parent's `reltuples` with `relpages = -1` | [analyze.c#partitioned-reltuples](../../../../raw/postgres-17/src/backend/commands/analyze.c#L665-L677) + measured `-1 / 200000` |
| The planner uses the table's tuple count for non-partial indexes | [plancat.c#get_relation_info](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L471-L478) |
| No index AM truncates its main fork during VACUUM | [spgvacuum.c#NOT_USED](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L877-L900), [heap.c#RelationTruncateIndexes](../../../../raw/postgres-17/src/backend/catalog/heap.c#L3058-L3066), [hash/README:31-34](../../../../raw/postgres-17/src/backend/access/hash/README#L31-L34), [hash.c#hashvacuumcleanup](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L642-L663) + measured byte-identical sizes in 36 cells |
| Freed index pages go to the FSM fork, which is not measured | [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), [gistvacuum.c#gistvacuumpage](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c#L298-L304), [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L747-L764) |
| GIN pending list flush threshold, the flush function, and its FSM handling | [ginfast.c#ginHeapTupleFastInsert](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L448-L460), [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1030-L1091), [ginfast.c#shiftList](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L665-L668) + measured four-stage table |
| `fastupdate` defaults to on | [gin_private.h:33](../../../../raw/postgres-17/src/include/access/gin_private.h#L33), [reloptions.c#fastupdate](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L123-L131) |
| One GIN entry per array element | [ginarrayproc.c#ginarrayextract](../../../../raw/postgres-17/src/backend/access/gin/ginarrayproc.c#L49-L55) |
| Hash bucket count comes from a heap estimate, not the scan | [hash.c#hashbuild](../../../../raw/postgres-17/src/backend/access/hash/hash.c#L133-L137), [tableam.c#table_block_relation_estimate_size](../../../../raw/postgres-17/src/backend/access/table/tableam.c#L690-L699) + three measured builds |
| Hash grows in power-of-two splitpoint groups, extending the EOF over a hole | [hashpage.c#_hash_init_metabuffer](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L509-L526), [hashpage.c#_hash_expandtable](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L779-L803), [hashpage.c#_hash_alloc_buckets](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c#L1030-L1035) + measured 87 zero pages and a 712,704-byte hole |
| The v17 GiST sorted build is the default for sortsupport opclasses and ignores fillfactor | [gistbuild.c#GIST_SORTED_BUILD](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L231-L248), [gistbuild.c#fillfactor-ignored](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L465-L470), [gistbuild.c#freespace](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c#L250-L254) + measured 3 x 4 sweep |
| GiST runtime inserts pass `freespace = 0` | [gist.c#gistinsert](../../../../raw/postgres-17/src/backend/access/gist/gist.c#L183-L186) |
| GiST deletes empty leaf pages under a visibility gate | [gistutil.c#gistPageRecyclable](../../../../raw/postgres-17/src/backend/access/gist/gistutil.c#L885-L908) |
| SP-GiST redirects become placeholders only when removable, and only trailing placeholders go | [spgvacuum.c#redirect-gate](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L528-L538), [spgvacuum.c#trailing-placeholders](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c#L569-L572) |
| BRIN `ambulkdelete` is a no-op; cleanup only touches the FSM | [brin.c#brinbulkdelete](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L1283-L1301), [brin.c#brin_vacuum_scan](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L2186-L2192) |
| BRIN entry count is heap pages divided by `pages_per_range` | [brin.sgml#brin-intro](../../../../raw/postgres-17/doc/src/sgml/brin.sgml#L57-L65), [brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-17/src/include/access/brin_page.h#L92-L94) |
| v17 can build BRIN in parallel, and it does not change the size | [brin.c#amcanbuildparallel](../../../../raw/postgres-17/src/backend/access/brin/brin.c#L265-L266) + measured 139 pages at 0/2/4 workers with 4 workers observed |
| `pgstattuple` opens only btree, hash and GiST indexes | [pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L269-L296), [pgstatindex.c#btree-only](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228) + measured errors |
| `pgstattuple` now treats a new/empty hash page as free space | [pgstattuple.c#pgstat_hash_page](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L456-L492) + measured 0 of 21 hash cells erroring |
| Upstream advises monitoring physical size for non-B-tree indexes | [maintenance.sgml#routine-reindex](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L1036-L1040) |
| `REINDEX` is the documented remedy for a bloated index | [ref/reindex.sgml#bloated](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml#L54-L64) |
| `gin_pending_list_limit` is `PGC_USERSET`, default 4096 kB | [guc_tables.c#gin_pending_list_limit](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3575-L3584) |
| GIN and BRIN reject `fillfactor`; unknown parameters error | [reloptions.c#unrecognized-parameter](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L1466-L1478) |

## Open Questions

- All thresholds come from one fixture family on one platform (x86-64 Linux under WSL2, 8192-byte blocks, gcc 13.3). Key widths, key cardinality, opclass choice and `BLCKSZ` all move the constants; the windows for hash and SP-GiST are narrower than their own no-bloat noise.
- The GiST window's lower edge is set by a single cell at 19.65% reclaimable against a 20% bar. Moving the "worth it" definition to 19% would close the window entirely, so GiST's perfect score is less robust on this pin than the v12 page's wider window suggests.
- The GiST fillfactor result was measured only for `point_ops` (sortsupport) and `range_ops` (none). Whether any out-of-core GiST opclass ships a sortsupport function, and how that interacts with an existing `fillfactor` reloption, was not tested.
- The GIN measurements hold distinct-key count fixed at 1,000 while row count grows. A fixture whose distinct keys grow with rows would put GIN in a different size regime, and the +80.48% fresh-build drift at 800,000 rows may not generalize.
- The "worth rebuilding" boundary of 20% reclaimable is a judgement, not a measurement. Thresholds at 30% and 50% were scored, but no cost model ties reclaimed bytes and query blocks to an operational value.
- Query work was measured warm, single-client, with `enable_seqscan = off`, on one representative query shape per access method. Cold-cache behavior, concurrency, and plan-shape changes caused by bloat were not measured, and hash's flat result may be an artifact of a single-row equality probe. One fired GiST cell needed 7% *more* blocks after the rebuild, and that regression was not investigated.
- Timings were taken on WSL2 with `fsync = off` and `synchronous_commit = off`. Only the block counts and byte sizes should be treated as portable; the millisecond figures are indicative.
- Autovacuum was off, and `max_parallel_maintenance_workers` was 0 for the matrix. In production, autovacuum both refreshes the denominator and flushes the GIN pending list during its analyze pass, so the observed GIN readings are an upper bound on what a production maintenance window would see.
- SP-GiST's redirect-to-placeholder gate depends on the oldest snapshot in the cluster; no cell was run with a competing long-lived snapshot, so the worst-case SP-GiST reading under a held snapshot is unmeasured.
- The 2% index-vacuum bypass was demonstrated to defer dead index entries, but its cumulative effect on index size across many maintenance cycles was not measured, and no bound tighter than "2% of pages per pass" was established.
- The BRIN conclusion is derived from one correlated `int4` minmax column at three `pages_per_range` values. Inclusion opclasses, uncorrelated columns and much larger tables were not measured, though the size model in `brin.sgml` predicts the same insensitivity.
- The parallel-BRIN size result was measured at 0, 2 and 4 workers on one 1,600,000-row table at two `pages_per_range` values. Whether a larger worker count or a much larger table can perturb the tuplesort merge enough to change the page count is unmeasured.
- `pg_relation_size` on a partitioned index returned 0 in the measurement, but there is no ACL or relkind check in `dbsize.c` that guarantees it; the value follows from the relcache leaving `rd_locator` unset for a storage-less relkind. No test in the tree asserts it.
- No upstream test asserts index size, page counts or bloat for any of these five access methods, and none ties a comment to size, so there is no in-tree regression coverage that would catch a future change invalidating this calibration.
- These constants are pinned to 17.10. Parallel GIN build arrives in 18.0 (`8492feb98f6`, not an ancestor of this pin) and would make a fresh GIN baseline worker-count dependent, so they must not be reused for PostgreSQL 18 or later.

## Source References

- [comment.c](../../../../raw/postgres-17/src/backend/commands/comment.c)
- [gram.y](../../../../raw/postgres-17/src/backend/parser/gram.y)
- [objectaddress.c](../../../../raw/postgres-17/src/backend/catalog/objectaddress.c)
- [dependency.c](../../../../raw/postgres-17/src/backend/catalog/dependency.c)
- [index.c](../../../../raw/postgres-17/src/backend/catalog/index.c)
- [heap.c](../../../../raw/postgres-17/src/backend/catalog/heap.c)
- [parse_utilcmd.c](../../../../raw/postgres-17/src/backend/parser/parse_utilcmd.c)
- [system_functions.sql](../../../../raw/postgres-17/src/backend/catalog/system_functions.sql)
- [pg_proc.dat](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat)
- [pg_class.h](../../../../raw/postgres-17/src/include/catalog/pg_class.h)
- [pg_description.h](../../../../raw/postgres-17/src/include/catalog/pg_description.h)
- [dbsize.c](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c)
- [vacuum.c](../../../../raw/postgres-17/src/backend/commands/vacuum.c)
- [vacuumlazy.c](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c)
- [analyze.c](../../../../raw/postgres-17/src/backend/commands/analyze.c)
- [tableam.c](../../../../raw/postgres-17/src/backend/access/table/tableam.c)
- [heapam_handler.c](../../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c)
- [plancat.c](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c)
- [storage.c](../../../../raw/postgres-17/src/backend/catalog/storage.c)
- [indexfsm.c](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c)
- [reloptions.c](../../../../raw/postgres-17/src/backend/access/common/reloptions.c)
- [guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c)
- [hash.c](../../../../raw/postgres-17/src/backend/access/hash/hash.c)
- [hashpage.c](../../../../raw/postgres-17/src/backend/access/hash/hashpage.c)
- [hashovfl.c](../../../../raw/postgres-17/src/backend/access/hash/hashovfl.c)
- [hash.h](../../../../raw/postgres-17/src/include/access/hash.h)
- [hash/README](../../../../raw/postgres-17/src/backend/access/hash/README)
- [gist.c](../../../../raw/postgres-17/src/backend/access/gist/gist.c)
- [gistbuild.c](../../../../raw/postgres-17/src/backend/access/gist/gistbuild.c)
- [gistvacuum.c](../../../../raw/postgres-17/src/backend/access/gist/gistvacuum.c)
- [gistutil.c](../../../../raw/postgres-17/src/backend/access/gist/gistutil.c)
- [gist_private.h](../../../../raw/postgres-17/src/include/access/gist_private.h)
- [ginfast.c](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c)
- [ginvacuum.c](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c)
- [ginutil.c](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c)
- [ginarrayproc.c](../../../../raw/postgres-17/src/backend/access/gin/ginarrayproc.c)
- [gin_private.h](../../../../raw/postgres-17/src/include/access/gin_private.h)
- [spgvacuum.c](../../../../raw/postgres-17/src/backend/access/spgist/spgvacuum.c)
- [spgist_private.h](../../../../raw/postgres-17/src/include/access/spgist_private.h)
- [brin.c](../../../../raw/postgres-17/src/backend/access/brin/brin.c)
- [brin_page.h](../../../../raw/postgres-17/src/include/access/brin_page.h)
- [brin.h](../../../../raw/postgres-17/src/include/access/brin.h)
- [amapi.h](../../../../raw/postgres-17/src/include/access/amapi.h)
- [pgstattuple.c](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c)
- [pgstatindex.c](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c)
- [create_index.sql](../../../../raw/postgres-17/src/test/regress/sql/create_index.sql)
- [maintenance.sgml](../../../../raw/postgres-17/doc/src/sgml/maintenance.sgml)
- [ref/comment.sgml](../../../../raw/postgres-17/doc/src/sgml/ref/comment.sgml)
- [ref/reindex.sgml](../../../../raw/postgres-17/doc/src/sgml/ref/reindex.sgml)
- [brin.sgml](../../../../raw/postgres-17/doc/src/sgml/brin.sgml)
- [catalogs.sgml](../../../../raw/postgres-17/doc/src/sgml/catalogs.sgml)

## Navigation

- [v17 index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 17 (unverified)](comment-stored-index-heap-ratio-bloat.md)
- [Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17 (unverified)](btree-index-bloat-core-sql-only.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 17 (unverified)](reindex-index-concurrently.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 17 (unverified)](../query-planning/bloated-indexes-query-planner.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Calibrating a COMMENT-Stored Bytes-per-Table-Tuple REINDEX Threshold for Every Non-B-Tree Index in PostgreSQL 12 (unverified)](../../../v12/questions/indexing/comment-stored-bytes-per-table-tuple-non-btree.md)
