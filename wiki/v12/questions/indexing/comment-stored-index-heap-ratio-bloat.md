---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# Detecting Bloat in All Index Types by Storing an Index/Heap Size Ratio in COMMENT in PostgreSQL 12 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The proposal](#the-proposal)
  - [Why COMMENT is a practical slot in core v12](#why-comment-is-a-practical-slot-in-core-v12)
  - [What the ratio actually measures](#what-the-ratio-actually-measures)
  - [Where a fresh ratio is scale-invariant, and where it is not](#where-a-fresh-ratio-is-scale-invariant-and-where-it-is-not)
  - [The hash sawtooth](#the-hash-sawtooth)
  - [Measured detection power on seven access methods](#measured-detection-power-on-seven-access-methods)
  - [The three heap-side failure modes](#the-three-heap-side-failure-modes)
  - [A 200k-row delete-and-reload cycle test on all seven index types](#a-200k-row-delete-and-reload-cycle-test-on-all-seven-index-types)
  - [Use the main fork, not pg_table_size](#use-the-main-fork-not-pg_table_size)
  - [The detection query](#the-detection-query)
  - [The baseline audit query](#the-baseline-audit-query)
  - [Capture discipline](#capture-discipline)
  - [Baseline durability across maintenance commands](#baseline-durability-across-maintenance-commands)
  - [Locking, permissions, and visibility](#locking-permissions-and-visibility)
  - [Cost of the survey, and the relpages variant](#cost-of-the-survey-and-the-relpages-variant)
  - [Partitioned indexes](#partitioned-indexes)
  - [Fixture thresholds, and what to do with a flagged index](#fixture-thresholds-and-what-to-do-with-a-flagged-index)
  - [Why this exists: the contrib boundary](#why-this-exists-the-contrib-boundary)
  - [Catalog, generated-header, and extension boundary](#catalog-generated-header-and-extension-boundary)
  - [Tests in the pinned checkout](#tests-in-the-pinned-checkout)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 12, propose a way to detect bloat in all types of indexes by using the `COMMENT` command to store the ratio between the table heap and the index.

Prompt note: the user's original wording was "in postgreql 12 , question: propose a way to detect bloat to all types of indexes by using the comment command to store the ratio between the table heap and the index". The user approved correcting the spelling, the preposition, and the command capitalization before filing.

## Answer

Use it as a screening signal, not as a bloat measurement for every index type. In these fixtures the signal was calibratable for B-tree, SP-GiST, contrib `bloom`, and—more cautiously—GiST. It was not calibratable for hash, GIN, or BRIN. `pg_relation_size()` is the direct core-v12 measurement of an ordinary physical index's current main-fork bytes, regardless of its access method ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)). An index comment is a practical arbitrary-text field that follows the index through `REINDEX CONCURRENTLY`, but it is not a private monitoring slot: human comments share it, `CREATE TABLE ... LIKE INCLUDING ALL` can copy it, and dump/restore can omit it ([index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656), [parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-12/src/backend/parser/parse_utilcmd.c#L1184-L1217), [pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L9491-L9502)).

The one-line summary of the mathematics, which drives every result on this page:

```text
drift = current_ratio / baseline_ratio = (index growth factor) / (heap growth factor)
```

The numerator is **index size growth**, not automatically bloat. A rebuild-reclaimable fraction is a different quantity:

```text
reclaimable_fraction = (current_index_bytes - rebuilt_index_bytes)
                       / current_index_bytes
```

The tables below use that fraction where they report rebuild-reclaimable space. The delete-and-reload table instead reports excess bytes relative to the rebuilt size, because that denominator is directly comparable to growth from the initial build.

### The proposal

Three moving parts.

1. **Capture.** In a quiescent maintenance window, immediately after the index is built or rebuilt, compute `pg_relation_size(index) / pg_relation_size(table)` and store the bare number as the index's comment. Record or otherwise validate the heap's main-fork size at the same time. Ordinary `VACUUM` removes dead tuples and can trim empty pages only from the physical end; it does not compact free space in the middle, so “vacuumed” does not mean “fresh heap” ([vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1850-L1867), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1870-L1970)). `pg_relation_size(regclass)` is the one-argument form that resolves to the `'main'` fork, and it `stat()`s each segment file ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308)).
2. **Store.** `COMMENT ON INDEX <index> IS '<number>'` writes one `pg_description` row keyed by `(objoid, classoid, objsubid)` with `objsubid = 0` for a whole relation ([pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L48-L57), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225)). Re-running it replaces the row rather than appending; an empty string deletes it ([comment.c:154-156](../../../../raw/postgres-12/src/backend/commands/comment.c#L154-L156)).
3. **Detect.** Periodically recompute the ratio, divide by the stored baseline, and flag indexes whose drift exceeds a locally calibrated, per-access-method threshold. Read the baseline with `obj_description(indexrelid, 'pg_class')`, which is a SQL-language builtin selecting the `objsubid = 0` row ([pg_proc.dat#obj_description](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2311-L2314)). Treat a zero-byte heap as an explicit measurement state, not as a row to discard.

### Why COMMENT is a practical slot in core v12

The main storage choices have different lifecycle costs:

| Keyed by | Survives `REINDEX` | Survives `REINDEX CONCURRENTLY` | Survives `pg_dump`/restore |
|---|---|---|---|
| `pg_index.indexrelid` in your own table | yes | **no** — the OID changes | no — OIDs are not stable across restore |
| `relfilenode` in your own table | **no** | no | no |
| Logical name in your own table | yes, if kept in sync with rename/schema changes | yes | yes, if the baseline table is included |
| Index comment | yes | yes | by default; `--no-comments` disables dump or restore |

The `REINDEX CONCURRENTLY` case is the interesting one. `index_concurrently_swap()` explicitly rewrites the `pg_description` row's `objoid` from the old index to the new one, under the header comment `Move comment if any` ([index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)). Upstream regression coverage asserts exactly this behavior for both `REINDEX TABLE` and `REINDEX TABLE CONCURRENTLY` ([create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2116)). By default `pg_dump` emits the index comment, but `pg_dump --no-comments` and `pg_restore --no-comments` deliberately omit it ([pg_dump.c#dumpIndex-comment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16456-L16463), [pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L9491-L9502)).

There is no access-method-neutral arbitrary-text index reloption in v12 to use instead. The index API delegates options to the selected AM's `amoptions` parser, and validated parsers reject an unrecognized parameter ([amapi.h#IndexAmRoutine-amoptions](../../../../raw/postgres-12/src/include/access/amapi.h#L215-L233), [reloptions.c#unrecognized-parameter](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L1125-L1155), [reloptions.c#index_reloptions](../../../../raw/postgres-12/src/backend/access/common/reloptions.c#L1507-L1524)). A dedicated baseline table remains a valid alternative. It avoids commandeering the human comment and can store the index and heap components separately, but it must track OID changes, renames, schema moves, drops, and restores.

### What the ratio actually measures

Since `current_ratio / baseline_ratio` expands to `(idx1/idx0) / (heap1/heap0)`, the drift number conflates two independent quantities. When the heap growth factor is 1, drift reduces to index growth. It still does not prove bloat: hash bucket allocation, GIN pending-list state, key distribution, and access-method build behavior can change index size without creating rebuild-reclaimable space.

Three structural facts about v12 fix the shape of both factors:

- **Ordinary index VACUUM does not shrink a main fork.** `RelationTruncate()` is the main-fork truncation entry point ([storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L229-L295)). Its active index call is the `TRUNCATE` path, which empties and rebuilds indexes under `AccessExclusiveLock` ([heap.c#RelationTruncateIndexes](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3163-L3207)). The apparent SP-GiST VACUUM call is compiled out under `#ifdef NOT_USED` because it is unsafe against concurrent inserts ([spgvacuum.c#truncate-disabled](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886)). `REINDEX` recreates an index, while `VACUUM FULL` and `CLUSTER` swap in a rewritten heap and rebuild all of its indexes ([index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3436-L3515), [cluster.c#finish_heap_swap](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1341-L1411)).
- **Index VACUUM returns pages to the free space map instead.** B-tree ([nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1078-L1095)), GiST ([gistvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L255-L265)), GIN ([ginvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L790)), and SP-GiST ([spgvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L848-L861)) all call `RecordFreeIndexPage`/`IndexFreeSpaceMapVacuum`. Reclaimed space is reusable but invisible to file size.
- **The heap can shrink.** `lazy_truncate_heap()` drops trailing empty pages, so a heap that loses its newest rows can get smaller while ordinary index VACUUM keeps each index's main-fork block count. It can also skip truncation because too few trailing pages qualify or because it cannot obtain the conditional `AccessExclusiveLock` ([vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1850-L1867), [vacuumlazy.c#lazy_truncate_heap-lock](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1870-L1930)).

That asymmetry makes drift sensitive to both index growth and heap movement. Neither direction is a bloat verdict by itself.

### Where a fresh ratio is scale-invariant, and where it is not

A stored baseline is only meaningful if a *freshly built* index keeps the same ratio to its heap as the table grows. Measured on the pinned 12.2 build: one table per access method, identical 40-byte payload column, built at three scales, each `VACUUM FULL`-ed and index-built from scratch before measuring.

| AM | ratio at 100k rows | at 400k | at 1.6M | drift over 16x growth |
|---|---|---|---|---|
| btree | 0.267701 | 0.266489 | 0.266141 | **0.9942** |
| spgist | 0.459109 | 0.457785 | 0.456060 | **0.9934** |
| bloom (contrib) | 0.192047 | 0.190592 | 0.190300 | **0.9909** |
| gist | 0.460862 | 0.482622 | 0.475250 | **1.0312** |
| hash | 0.498545 | 0.395732 | 0.396969 | 0.7963 |
| gin | 0.204648 | 0.158980 | 0.130168 | 0.6361 |
| brin | 0.002910 | 0.000727 | 0.000182 | **0.0625** |

Read that last column as "how much a perfectly healthy index's ratio moved for reasons that have nothing to do with bloat". Three access methods are structurally disqualified:

- **BRIN.** The index stayed at exactly 24,576 bytes at all three measured scales. A BRIN index summarizes `pages_per_range` heap pages per tuple, defaulting to 128 ([brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-12/src/include/access/brin.h#L39-L43)), while each revmap page holds many range pointers ([brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-12/src/include/access/brin_page.h#L88-L94)). That does **not** make BRIN size constant at every scale: the revmap computes additional pages from the number of covered heap ranges and extends as needed ([brin_revmap.c#range-to-revmap-page](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L37-L44), [brin_revmap.c#brinRevmapExtend](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L115-L123)). The tested ratio therefore decayed 16x inside one size plateau; at larger scales it changes in steps. Either shape defeats one fixed drift threshold.
- **GIN.** In this fixture, which held 10,000 distinct array elements, 16x more rows produced only 10x more index, so the fresh ratio fell 36%. The build path groups each extracted key with its sorted heap-TID list before inserting the entries, so row count alone does not determine the representation ([ginbulk.c#ginGetBAEntry](../../../../raw/postgres-12/src/backend/access/gin/ginbulk.c#L255-L280), [gininsert.c#ginbuild-entry-insert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L380-L406)).
- **Hash.** See the next section.

The measured B-tree, SP-GiST, GiST, and contrib `bloom` ratios stayed within 0.7%, 0.7%, 3.1%, and 0.9%, respectively, over the 16x growth. That stability is the precondition the scheme needs.

### The hash sawtooth

Hash sizing is a step function by construction. `_hash_init()` picks the initial bucket count from the estimated tuple count divided by the fill factor, rounded up to a whole splitpoint ([hashpage.c#_hash_init-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L505-L525)), with a default fill factor of 75 ([hash.h:279](../../../../raw/postgres-12/src/include/access/hash.h#L279)). `_hash_expandtable()` then allocates a whole splitpoint's worth of buckets at once when the split point advances ([hashpage.c#_hash_expandtable-splitpoint](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L785-L853)).

Freshly built hash and B-tree indexes over the same data, measured on the pinned build:

| rows | hash index bytes | hash ratio | btree ratio |
|---|---|---|---|
| 100,000 | 4,210,688 | 0.498545 | 0.267701 |
| 150,000 | 4,210,688 | 0.332256 | 0.267615 |
| 200,000 | 6,676,480 | 0.395247 | 0.267216 |
| 300,000 | 8,404,992 | 0.331717 | 0.266731 |
| 450,000 | 16,523,264 | 0.434698 | 0.266595 |
| 600,000 | 16,793,600 | 0.331393 | 0.266408 |
| 900,000 | 33,103,872 | 0.435500 | 0.266300 |

The healthy hash ratio oscillates between 0.3314 and 0.4985, a 1.50x swing with no rebuild-reclaimable space introduced by the workload, while B-tree moves 0.5% over the same 9x range. Depending on where the baseline lands in the sawtooth, an alert threshold below 1.50 can produce a false positive. Raising the threshold sacrifices sensitivity; these measurements do not prove that severe hash growth can never exceed 1.50.

### Measured detection power on seven access methods

Each fixture started with 200,000 rows and a freshly built index. After the workload, `REINDEX` supplied a reference index size and `VACUUM FULL` supplied a compacted-heap reference size. In the next tables, “index reclaimable fraction” is `(current index - rebuilt index) / current index`, and “heap reclaimable fraction” uses the analogous heap formula. These are operational rebuild comparisons, not access-method-independent definitions of bloat. `idx_factor` and `heap_factor` are the two components of drift.

**Index-heavy churn** (five rounds updating 20% of rows' indexed key, `VACUUM` after each round):

| AM | baseline | current | drift | idx_factor | heap_factor | index reclaimable fraction |
|---|---|---|---|---|---|---|
| btree | 0.267216 | 0.444040 | **1.662** | 1.995 | 1.200 | 0.499 |
| gin | 0.176978 | 0.239688 | **1.354** | 1.625 | 1.200 | 0.381 |
| hash | 0.398642 | 0.465455 | 1.168 | 1.401 | 1.200 | 0.271 |
| spgist | 0.455061 | 0.460007 | 1.011 | 1.213 | 1.200 | 0.176 |
| bloom | 0.191077 | 0.191111 | 1.000 | 1.201 | 1.200 | 0.167 |
| gist | 0.471183 | 0.433651 | 0.920 | 1.105 | 1.200 | 0.101 |
| brin | 0.001455 | 0.001212 | 0.833 | 1.000 | 1.200 | 0.000 |

The B-tree result came from **five rounds**, each updating the indexed key of the same 20% of rows and vacuuming afterward; it was not one 20%-of-rows update. The five rounds grew the index by 1.995x in this fixture. A fresh B-tree build uses default fillfactor 90 ([nbtree.h:169](../../../../raw/postgres-12/src/include/access/nbtree.h#L169)). More importantly, PostgreSQL collects every indexed attribute into the HOT-safety bitmap, and `heap_update()` permits a heap-only tuple (HOT) update only when no such attribute changed ([relcache.c#RelationGetIndexAttrBitmap](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4782-L4806), [heapam.c#HOT-decision](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3588-L3603)). Because these updates changed the indexed key, each successful update required a new index entry regardless of the heap fillfactor. The exact 1.995 factor remains fixture-specific.

Drift understates index growth here—1.662 against 1.995x—because the heap grew 20% at the same time. Where index and heap size grow by the same factor, as with `bloom` (1.201 against 1.200), drift is exactly 1.000 even though a rebuild would reclaim 16.7% of the current index file.

**Ordinary 4x growth with no maintenance** — the baseline for "what does normal look like":

| AM | drift | index reclaimable fraction |
|---|---|---|
| btree | 1.058 | 0.058 |
| spgist | 1.004 | 0.007 |
| bloom | 0.997 | 0.001 |
| gist | 0.942 | -0.079 |
| hash | 1.106 | 0.104 |
| gin | 0.773 | -0.056 |
| brin | 0.250 | 0.000 |

For B-tree, SP-GiST and bloom, this particular growth fixture moved drift by less than 6%, which separates it from this fixture's 1.66 B-tree churn signal. Negative reclaimable fractions for GiST and GIN are not arithmetic errors: rebuilding those two produced a *larger* index than the incrementally built one, so “REINDEX shrinks it” is not universal.

### The three heap-side failure modes

Each measured on all seven access methods.

**1. Shared retained space is invisible.** Delete 80% of rows, then `VACUUM`. Neither file shrinks in this scattered-delete fixture, so drift is exactly 1.000 on all seven access methods, while the index reclaimable fraction is 0.667 to 0.797 (BRIN 0.000):

| AM | drift | index reclaimable fraction | heap reclaimable fraction |
|---|---|---|---|
| btree | 1.000 | 0.797 | 0.800 |
| bloom | 1.000 | 0.797 | 0.800 |
| spgist | 1.000 | 0.789 | 0.800 |
| gist | 1.000 | 0.781 | 0.800 |
| hash | 1.000 | 0.684 | 0.800 |
| gin | 1.000 | 0.667 | 0.800 |
| brin | 1.000 | 0.000 | 0.800 |

This is the central limitation: a mass delete followed by ordinary VACUUM can be completely invisible to a heap-relative ratio.

**2. Heap growth masks index retained space.** Widening every row to 400 bytes grew the heap 6.4x while the index doubled. Drift fell to 0.312 for B-tree—a “healthy” reading on an index with a 49.9% rebuild-reclaimable fraction:

| AM | drift | idx_factor | heap_factor | index reclaimable fraction |
|---|---|---|---|---|
| btree | 0.312 | 1.995 | 6.388 | 0.499 |
| bloom | 0.313 | 1.997 | 6.388 | 0.499 |
| gist | 0.325 | 1.911 | 5.888 | 0.477 |
| hash | 0.340 | 2.174 | 6.388 | 0.540 |
| spgist | 0.349 | 2.012 | 5.763 | 0.503 |
| gin | 0.393 | 2.125 | 5.411 | 0.510 |
| brin | 0.157 | 1.000 | 6.388 | 0.000 |

**3. Heap truncation fakes an index-growth alarm.** Deleting the newest 75% of rows by `id`, then `VACUUM`, let `lazy_truncate_heap()` drop trailing heap pages while ordinary index VACUUM kept all index main-fork blocks. Drift landed at 3.995-3.998 on all seven access methods. On six of them the index also had rebuild-reclaimable space; on BRIN the 4.0x drift was a false positive against a 0.000 reclaimable fraction.

### A 200k-row delete-and-reload cycle test on all seven index types

This cycle is the scheme's best case. The heap ends the cycle at exactly the size it started, so the heap growth factor is 1.000 and drift collapses to pure index growth. Ratio drift matched raw index growth to the last reported digit on all seven access methods. On six, that growth also matched the excess over the rebuilt reference size.

What the cycle exposes is that **one post-delete `VACUUM` was not enough in this fixture**: three of the seven indexes grew 82-99% while the row count, row width, and heap size returned to their starting values. The ratio reported that growth faithfully, but only the rebuild comparison established how much of the final allocation was avoidable in this workload.

#### The fixture

One table, 200,000 rows, seven indexes — the six core access methods plus contrib `bloom` — on an isolated 12.2 server built from the pinned checkout with `autovacuum = off`, so the number of `VACUUM` passes is exactly what the script says. This differs from the one-table-per-access-method fixtures used earlier on this page: here all seven indexes share a single heap, so every row of the results table has the same denominator and the drift differences are entirely index-side.

Every index key is a strictly increasing function of the generated series, so the reload's keys (`200001`-`400000`) are all greater than every key of the first load (`1`-`200000`). The `pad` column keeps the same 40-byte payload as the earlier fixtures.

#### The script

A throwaway lab script for a scratch database, not production SQL: it creates and drops a schema, bulk-loads twice, and rebuilds every index. The production path is [The detection query](#the-detection-query), which carries the session-scoped timeouts.

```sql
CREATE EXTENSION IF NOT EXISTS bloom;
DROP SCHEMA IF EXISTS cyc CASCADE;
CREATE SCHEMA cyc;

CREATE TABLE cyc.t (id int, k int, k2 int, r int4range, p point, a int[], pad text);
CREATE TABLE cyc.res (phase text, obj text, am text,
                      heap_bytes bigint, idx_bytes bigint, ratio numeric);

CREATE FUNCTION cyc.snap(p_phase text) RETURNS void LANGUAGE sql AS $$
  INSERT INTO cyc.res
  SELECT p_phase, ic.relname, am.amname,
         pg_relation_size(i.indrelid),
         pg_relation_size(i.indexrelid),
         round(pg_relation_size(i.indexrelid)::numeric
               / nullif(pg_relation_size(i.indrelid), 0), 6)
  FROM pg_index i
  JOIN pg_class ic ON ic.oid = i.indexrelid
  JOIN pg_am    am ON am.oid = ic.relam
  WHERE i.indrelid = 'cyc.t'::regclass;
$$;

-- 1. add 200k rows
INSERT INTO cyc.t
SELECT g, g, g + 1000000, int4range(g, g + 10), point(g, (g * 37) % 1000),
       ARRAY[g, g + 1000000, g + 2000000], repeat('x', 40)
FROM generate_series(1, 200000) g;

-- 2. create all indexes
CREATE INDEX i_btree  ON cyc.t USING btree  (k);
CREATE INDEX i_hash   ON cyc.t USING hash   (k);
CREATE INDEX i_brin   ON cyc.t USING brin   (k);
CREATE INDEX i_gist   ON cyc.t USING gist   (r);
CREATE INDEX i_spgist ON cyc.t USING spgist (p);
CREATE INDEX i_gin    ON cyc.t USING gin    (a);
CREATE INDEX i_bloom  ON cyc.t USING bloom  (k, k2);

-- 3. calculate the ratio and store it for each index
DO $$
DECLARE r record;
BEGIN
  FOR r IN
    SELECT i.indexrelid::regclass AS idx,
           round(pg_relation_size(i.indexrelid)::numeric
                 / pg_relation_size(i.indrelid), 6) AS ratio
    FROM pg_index i WHERE i.indrelid = 'cyc.t'::regclass
  LOOP
    EXECUTE format('COMMENT ON INDEX %s IS %L', r.idx, r.ratio::text);
  END LOOP;
END $$;
SELECT cyc.snap('initial');

-- 4. run vacuum
VACUUM cyc.t;

-- 5. delete all rows
DELETE FROM cyc.t;

-- 6. run vacuum
VACUUM cyc.t;
SELECT cyc.snap('mid_cycle');

-- 7. add 200k rows again, with fresh ascending keys
INSERT INTO cyc.t
SELECT g, g, g + 1000000, int4range(g, g + 10), point(g, (g * 37) % 1000),
       ARRAY[g, g + 1000000, g + 2000000], repeat('x', 40)
FROM generate_series(200001, 400000) g;
SELECT cyc.snap('final');

-- reference: what a rebuilt index over the final 200k rows costs
REINDEX TABLE cyc.t;
SELECT cyc.snap('rebuilt');

-- 8. report
SELECT /* wiki_index_ratio_cycle_report */
       i.am,
       i.idx_bytes / 8192                                  AS initial_blocks,
       f.idx_bytes / 8192                                  AS final_blocks,
       round(100.0 * (f.idx_bytes - i.idx_bytes) / i.idx_bytes, 2) AS index_size_pct,
       round(100.0 * (f.heap_bytes - i.heap_bytes) / i.heap_bytes, 2) AS heap_size_pct,
       i.ratio                                             AS baseline_ratio,
       f.ratio                                             AS current_ratio,
       round(100.0 * (f.ratio - i.ratio) / i.ratio, 2)     AS ratio_pct,
       r.idx_bytes / 8192                                  AS rebuilt_blocks,
       round(100.0 * (f.idx_bytes - r.idx_bytes) / r.idx_bytes, 2) AS excess_vs_rebuilt_pct
FROM      cyc.res i
JOIN      cyc.res f USING (obj)
JOIN      cyc.res r USING (obj)
WHERE i.phase = 'initial' AND f.phase = 'final' AND r.phase = 'rebuilt'
ORDER BY index_size_pct DESC, i.am;
```

#### Results

The heap measured 3,847 blocks before the cycle and 3,847 blocks after it, so `heap_size_pct` is `0.00` on every row and the ratio column carries the index change alone. Sizes are in 8 KB blocks; `rebuilt_blocks` is the `REINDEX` reference.

| AM | initial | final | index size % | baseline ratio | current ratio | ratio % | rebuilt | excess vs rebuilt % |
|---|---|---|---|---|---|---|---|---|
| btree | 551 | 1098 | **+99.27** | 0.143228 | 0.285417 | **+99.27** | 551 | +99.27 |
| gist | 1538 | 2914 | **+89.47** | 0.399792 | 0.757473 | **+89.47** | 1538 | +89.47 |
| gin | 4101 | 7482 | **+82.44** | 1.066025 | 1.944892 | **+82.44** | 4101 | +82.44 |
| spgist | 1082 | 1091 | +0.83 | 0.281258 | 0.283598 | +0.83 | 1082 | +0.83 |
| bloom | 394 | 394 | 0.00 | 0.102417 | 0.102417 | 0.00 | 394 | 0.00 |
| brin | 3 | 3 | 0.00 | 0.000780 | 0.000780 | 0.00 | 3 | 0.00 |
| hash | 822 | 822 | 0.00 | 0.213673 | 0.213673 | 0.00 | 815 | +0.86 |

Three readings matter:

- **`ratio_pct` equals `index_size_pct` on all seven rows, digit for digit.** That is the arithmetic of a heap that returns to its starting size. It makes drift an index-growth signal, not by itself a proof that the growth is reclaimable. Drift was 1.9927, 1.8947, 1.8244, 1.0083, 1.0000, 1.0000, and 1.0000, in the same order.
- **`ratio_pct` equals `excess_vs_rebuilt_pct` on six of seven rows.** Hash is the exception. `pageinspect`'s `hash_metapage_info` reported `maxbucket = 767` and `ovflpoint = 11` for both the initial and rebuilt index; they differed in overflow pages, 53 against 46, which is the 7-block gap. The bucket count is fixed by the estimated tuple count at build time ([hashpage.c#_hash_init-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L505-L525)). The measurement does not establish whether the different overflow-page counts came from collision distribution, build order, or another hash-build detail.
- **BRIN reported 0.00% on a workload that deleted and reinserted every row.** `brinbulkdelete()` is a documented no-op — "there are no per-heap-tuple index tuples in BRIN indexes, there's not a lot we can do here" ([brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)) — so BRIN's 0.00% here is a correct reading, unlike its false positives elsewhere on this page.

#### Mid-cycle, the ratio is undefined

Between step 6 and step 7 the heap was **0 bytes**: `DELETE` emptied every page and `lazy_truncate_heap()` dropped all 3,847 of them ([vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1955-L1970)), while all seven indexes kept every block. The ratio is therefore `NULL` for all seven indexes. The reviewed detection query below preserves those rows and reports `heap main fork is zero`; the earlier query silently filtered them out.

This is a distinct failure mode from the mass delete in [The three heap-side failure modes](#the-three-heap-side-failure-modes). Deleting *most* rows left both files intact and pinned drift at 1.000 in that fixture; deleting *all* rows made the ratio undefined. A monitor must surface that undefined state rather than hide the index.

#### Why the cycle splits the access methods: the recyclability gate

Rerunning the identical cycle with two and three post-`DELETE` `VACUUM` passes instead of one isolates the cause. Only the `VACUUM` count changed:

| AM | 1 VACUUM | 2 VACUUMs | 3 VACUUMs |
|---|---|---|---|
| btree | **+99.27%** | 0.00% | 0.00% |
| gist | **+89.47%** | 0.00% | 0.00% |
| gin | **+82.44%** | **+82.44%** | **+82.44%** |
| spgist | +0.83% | +0.83% | +0.83% |
| hash, brin, bloom | 0.00% | 0.00% | 0.00% |

Three groups, and each one is visible in source.

**B-tree and GiST: deleted, but not yet recyclable.** Both gate FSM reuse on `RecentGlobalXmin`. `_bt_page_recyclable()` returns true only for a page that is deleted *and* whose `btpo.xact` precedes `RecentGlobalXmin` ([nbtpage.c#_bt_page_recyclable](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L940-L963)); `gistPageRecyclable()` applies the same test to `deleteXid` ([gistutil.c#gistPageRecyclable](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L881-L906)). The `VACUUM` that deletes the pages therefore cannot recycle them in the same pass. `btvacuumpage()` takes its `else if (P_ISDELETED(opaque))` branch — commented "Already deleted, but can't recycle yet" — instead of calling `RecordFreeIndexPage()` ([nbtree.c#btvacuumpage-recycle](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1179)), and GiST's cleanup has the same two-branch shape, with `else if (GistPageIsDeleted(page))` bypassing the FSM call ([gistvacuum.c#gistvacuum-recycle](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L298-L320)).

The measurement confirms it: after the single post-delete `VACUUM` the FSM fork of both indexes was **0 bytes**, while SP-GiST, bloom and BRIN each had a 24,576-byte FSM. Hash also showed 0 bytes, for the opposite reason — it never uses the index FSM at all, as below. `pgstatindex` on the B-tree tells the whole story in three snapshots:

| phase | blocks | internal | leaf | deleted | avg_leaf_density |
|---|---|---|---|---|---|
| after build | 551 | 3 | 547 | 0 | 90.00 |
| after DELETE + 1 VACUUM | 551 | 2 | 1 | 547 | 0.05 |
| after reload | 1098 | 3 | 547 | 547 | 90.00 |

The final 1,098 blocks are exactly `1 metapage + 3 internal + 547 leaf + 547 deleted`. Every leaf page from the first load is still allocated and still dead weight, and the reload had to extend the relation for 547 brand-new leaves.

Note what this is *not*: it is not a consequence of the reload's keys being ascending. `_bt_getbuf()` takes whatever block `GetFreeIndexPage()` hands back ([nbtpage.c#_bt_getbuf-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L801-L842)), and that function just returns the first page the FSM reports with at least half a block free ([indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L33-L46)). Free-page reuse is key-order agnostic. With a second `VACUUM` before the reload the pages reached the FSM in time, the inserts reused them, and both indexes ended at *exactly* their initial 551 and 1,538 blocks.

**GIN: no recovery in the measured one-, two-, or three-pass arms.** GIN uses a `RecentGlobalXmin` recyclability gate ([ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L786)), but that was not the limiting path in this fixture. `ginVacuumEntryPage()` does not remove an entry tuple merely because its posting list becomes empty; when `nitems` reaches 0 it sets `plist = NULL, plistsize = 0` and puts the tuple back on the page ([ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556)). Those retained entry tuples kept the measured entry pages nonempty and out of the FSM.

A `pageinspect` probe on an isolated GIN fixture measured this directly. After deleting all 200,000 rows and running `VACUUM` three times, `gin_metapage_info` still reported **600,000 entries across 4,100 entry pages** in a 4,101-block index, and the FSM fork was still 0 bytes. The reload then grew the file to 7,482 blocks. `REINDEX` removed that retained allocation in the measured cycle; the experiment did not test every workload or later maintenance sequence.

**SP-GiST, hash, BRIN, and bloom: different reuse paths.** SP-GiST's final decision to put a new or empty non-root page in the FSM has no separate transaction test ([spgvacuum.c#spgvacuumpage](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L607-L670)). That does not make all SP-GiST cleanup ungated: a redirect tuple becomes a removable placeholder only when its XID precedes `RecentGlobalXmin` ([spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L482-L545)). The measured SP-GiST file retained nine extra blocks (+0.83%) in every one-, two-, and three-pass arm. Contrib `bloom` records new or deleted pages directly ([blvacuum.c#blvacuumcleanup](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L189-L214)). Hash does not use the index FSM: `_hash_freeovflpage()` clears a bit in the index's own bitmap and `_hash_getovflpage()` reuses a free overflow page from that map ([hashovfl.c:632](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L632), [hashovfl.c#_hash_getovflpage-setbit](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L313-L329)). BRIN has no per-row entries to remove ([brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)).

#### What the cycle changes about the recommendation

Nothing in the per-access-method verdict in [Fixture thresholds, and what to do with a flagged index](#fixture-thresholds-and-what-to-do-with-a-flagged-index) moves, but two operational points sharpen:

1. **For B-tree and GiST, a second `VACUUM` before the reload prevented the measured growth.** That is a fixture result, not a universal prescription: a held transaction horizon can delay recycling, and another `VACUUM` has its own I/O cost. Once the reload extended the files, a later `VACUUM` could make old pages reusable for future inserts but could not return their current main-fork blocks to the operating system. Plain or concurrent `REINDEX` can replace the index; `VACUUM FULL` and `CLUSTER` rebuild all indexes while rewriting the table ([index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3436-L3515), [cluster.c#finish_heap_swap](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1341-L1411)).
2. **Do not normalize unexplained growth by re-capturing it.** After a bulk delete-and-reload, investigate the index and heap components. Capture a new baseline only after a deliberate rebuild or after proving that the changed allocation is the intended healthy reference. Otherwise, re-capture encodes the very retained space the monitor is meant to detect.

### Use the main fork, not pg_table_size

`pg_table_size()` sums every fork plus the TOAST relation and its index ([dbsize.c#calculate_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L380-L408), [dbsize.c#calculate_toast_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L338-L378)). TOAST traffic can move independently of an index on a narrow key, so this denominator can overwhelm the signal. Moving 20% of rows to TOAST while the B-tree doubled:

| denominator | drift, btree | drift, gin | drift, spgist |
|---|---|---|---|
| `pg_relation_size` (main fork) | 1.746 | 1.408 | 1.030 |
| `pg_table_size` (all forks + TOAST) | 0.095 | 0.099 | 0.067 |

The B-tree rebuild-reclaimable fraction in that fixture was 0.499. The main-fork denominator flagged its index growth; the `pg_table_size` denominator reported a 10x improvement. Use `pg_relation_size` on both sides.

One caveat that made this fixture harder to build than expected: a first attempt used `repeat('z', 3000)`, which pglz compressed to a 44-byte value that never left the heap. Wide-but-compressible columns do not produce TOAST traffic.

### The detection query

Read-only. Verified against the pinned 12.2 build.

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';

WITH candidates AS MATERIALIZED (
    SELECT /* wiki_index_bloat_ratio_candidates */
           i.indexrelid,
           i.indrelid,
           n.nspname AS schema_name,
           tc.relname AS table_name,
           ic.relname AS index_name,
           am.amname AS index_am,
           CASE WHEN d.description ~ '^[0-9]+([.][0-9]+)?$'
                THEN d.description::numeric END AS baseline_ratio
    FROM pg_index i
    JOIN pg_class ic ON ic.oid = i.indexrelid
    JOIN pg_class tc ON tc.oid = i.indrelid
    JOIN pg_namespace n ON n.oid = ic.relnamespace
    JOIN pg_am am ON am.oid = ic.relam
    LEFT JOIN pg_description d
           ON d.objoid = i.indexrelid
          AND d.classoid = 'pg_class'::regclass
          AND d.objsubid = 0
    WHERE ic.relkind = 'i'
), measured AS MATERIALIZED (
    SELECT /* wiki_index_bloat_ratio_measure */
           c.*,
           pg_relation_size(c.indexrelid) AS index_bytes,
           pg_relation_size(c.indrelid) AS heap_bytes
    FROM candidates c
    WHERE c.baseline_ratio > 0
)
SELECT /* wiki_index_bloat_ratio_drift */
       m.schema_name,
       m.table_name,
       m.index_name,
       m.index_am,
       m.baseline_ratio,
       CASE WHEN m.index_bytes IS NULL THEN 'index unavailable'
            WHEN m.heap_bytes IS NULL THEN 'heap unavailable'
            WHEN m.heap_bytes = 0 THEN 'heap main fork is zero'
            WHEN m.index_bytes = 0 THEN 'index main fork is zero'
            ELSE 'ok'
       END AS measurement_state,
       round(m.index_bytes::numeric / nullif(m.heap_bytes, 0), 6)
           AS current_ratio,
       round((m.index_bytes::numeric / nullif(m.heap_bytes, 0))
             / m.baseline_ratio, 3) AS drift,
       pg_size_pretty(m.index_bytes) AS index_size,
       pg_size_pretty(m.heap_bytes) AS heap_main_size,
       s.n_live_tup,
       s.n_dead_tup
FROM measured m
LEFT JOIN pg_stat_all_tables s ON s.relid = m.indrelid
ORDER BY CASE WHEN m.index_bytes IS NULL OR m.heap_bytes IS NULL
                        OR m.heap_bytes = 0 OR m.index_bytes = 0
                     THEN 0 ELSE 1 END,
         drift DESC NULLS LAST;
COMMIT;
```

Details that matter:

- The `CASE` guard is not decoration. A bare `description::numeric` fails the whole query when an index carries a human comment, and PostgreSQL does not promise that a `WHERE` filter runs before a cast; the documentation names `CASE` as the supported way to force evaluation order ([syntax.sgml#expression-evaluation](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L2500-L2527)). The accepted storage format is a positive unsigned decimal such as `0.267216`.
- The materialized `measured` common table expression calls `pg_relation_size` once per side and retains zero or unavailable relations. `pg_relation_size` opens each relation separately with `AccessShareLock`, returns `NULL` if a concurrently dropped relation can no longer be opened, `stat()`s its segment files, and then closes it ([dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)). The two byte counts are not one atomic snapshot; run the survey in a quiet period if concurrent extension would make the ratio misleading.
- `relkind = 'i'` excludes partitioned indexes, which are `'I'` and have no storage ([pg_class.h#relkind](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L163), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L185-L192)).
- `n_live_tup` and `n_dead_tup` are estimated row counts, not heap-compaction or bloat measurements ([monitoring.sgml#estimated-tuples](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2773-L2785)). Collector reports can lag by 500 ms by default, and a backend caches one statistics snapshot for the rest of its transaction ([monitoring.sgml#statistics-lag](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L248)). Use these columns as context only. They require `track_counts`, a `PGC_SUSET` setting that a superuser can change for a session or transaction; it defaults to on ([guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1399)).
- `statement_timeout` and `lock_timeout` are both `PGC_USERSET`, so `SET LOCAL` here is transaction scope with no restart or reload ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)).
- `pg_index.indexrelid` and `pg_index.indrelid` are the index and table OIDs ([pg_index.h#pg_index](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L35)).

### The baseline audit query

The comment is a shared, human-visible slot, so a monitoring scheme built on it needs a second query that reports which indexes have no usable baseline. Read-only, verified against the pinned build:

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';

WITH comments AS MATERIALIZED (
    SELECT /* wiki_index_baseline_comments */
           n.nspname AS schema_name,
           ic.relname AS index_name,
           am.amname AS index_am,
           d.description,
           CASE WHEN d.description ~ '^[0-9]+([.][0-9]+)?$'
                THEN d.description::numeric END AS baseline_ratio
    FROM pg_index i
    JOIN pg_class ic ON ic.oid = i.indexrelid
    JOIN pg_namespace n ON n.oid = ic.relnamespace
    JOIN pg_am am ON am.oid = ic.relam
    LEFT JOIN pg_description d
           ON d.objoid = i.indexrelid
          AND d.classoid = 'pg_class'::regclass
          AND d.objsubid = 0
    WHERE ic.relkind = 'i'
      AND n.nspname !~ '^pg_'
      AND n.nspname <> 'information_schema'
)
SELECT /* wiki_index_baseline_audit */
       c.schema_name,
       c.index_name,
       c.index_am,
       CASE WHEN c.description IS NULL THEN 'no baseline stored'
            WHEN c.baseline_ratio IS NULL THEN 'comment is not a positive decimal'
            ELSE 'baseline is zero'
       END AS problem,
       left(coalesce(c.description, ''), 40) AS comment_prefix
FROM comments c
WHERE c.baseline_ratio IS NULL OR c.baseline_ratio <= 0
ORDER BY 1, 2;
COMMIT;
```

The earlier audit query cast `description::numeric` in an `OR` condition and could fail before its regex predicate ran. This version puts the cast inside `CASE`. On the test server it reports an index whose comment was replaced with `rebuilt by dba on 2026-08-01`, as well as partition child indexes whose baselines were overwritten by the hazard described below.

### Capture discipline

Since the payload is a bare number with no metadata, every guarantee has to live in the capture procedure. Writes are described here rather than filed as production SQL:

- Capture immediately after building or rebuilding the index, but only when the heap main-fork size is also a trusted reference. Ordinary `VACUUM` does not make the heap compact; it can leave internal free space in place and may skip tail truncation ([vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1850-L1867), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1870-L1970)). Prefer a freshly loaded table or a deliberate heap rewrite when defining a compact baseline, or store the heap bytes separately so later movement remains visible.
- Store enough decimal precision that a small valid ratio cannot round to zero. Six decimal places are used only to present the fixture tables on this page; they are not a safe universal storage format.
- Capture again after a deliberate reset of either component, such as `REINDEX`, `VACUUM FULL`, or `CLUSTER`, and for each newly created or attached leaf index. After a row-width change or bulk reload, first establish that the resulting allocation is the intended healthy reference. Never re-capture merely to silence unexplained drift.
- Treat `n_live_tup` and `n_dead_tup` as lagged estimates, not proof that a heap is compact. A zero dead-tuple estimate after `VACUUM` does not show whether the main fork still contains internal free space ([monitoring.sgml#estimated-tuples](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2773-L2785)).
- Refuse to overwrite a non-numeric comment without a human decision. A numeric human comment is indistinguishable from a baseline, so reserve the field by policy or use a dedicated table.
- Audit after `CREATE TABLE ... LIKE INCLUDING ALL`: PostgreSQL copies the source indexes and their comments, so the new indexes inherit numeric ratios measured against a different heap ([parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-12/src/backend/parser/parse_utilcmd.c#L1181-L1221), [create_table_like.out#index-comments-copied](../../../../raw/postgres-12/src/test/regress/expected/create_table_like.out#L352-L373)).

### Baseline durability across maintenance commands

Measured on the pinned 12.2 build, tracking the index OID and the stored comment:

| Operation | index OID | `relfilenode` | baseline |
|---|---|---|---|
| `REINDEX INDEX` | unchanged | changed | kept |
| `REINDEX INDEX CONCURRENTLY` | **changed** (297875 → 297881) | changed | kept |
| `VACUUM FULL` on the table | unchanged | changed | kept |
| `CLUSTER` | unchanged | changed | kept |
| `ALTER INDEX ... RENAME` | unchanged | unchanged | kept |
| `ALTER TABLE ... ALTER COLUMN TYPE`, plain table | **changed** (297881 → 297896) | changed | kept |
| `ALTER TABLE ... ALTER COLUMN TYPE`, partitioned table | changed | changed | **overwritten with the parent index's comment** |
| `CREATE TABLE ... LIKE INCLUDING ALL` | new | new | **copied from the source index** |
| `DROP INDEX` | — | — | `pg_description` row deleted |
| default `pg_dump`/restore | — | — | emits and restores `COMMENT ON INDEX cm.i_k IS '0.267216';` |

The partitioned-table row is a genuine data-loss hazard, not a test artifact. A parent index commented `parent-baseline` with children commented `0.111111` and `0.222222` ended up with **both children carrying `parent-baseline`** after one `ALTER TABLE ... ALTER COLUMN TYPE`. Upstream knows: the regression suite covers this case and labels its own expected output `-- Note: these tests currently show the wrong behavior for comments :-(` ([alter_table.sql#partitioned-comments](../../../../raw/postgres-12/src/test/regress/sql/alter_table.sql#L1421-L1440), [alter_table.out#partitioned-comments](../../../../raw/postgres-12/src/test/regress/expected/alter_table.out#L2103-L2134)). On partitioned tables, one column type change silently replaces every child baseline.

`CREATE TABLE ... LIKE INCLUDING ALL` copies index definitions and comments through `IndexStmt.idxcomment`; upstream regression output shows the copied index comments ([parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-12/src/backend/parser/parse_utilcmd.c#L1181-L1221), [create_table_like.sql#index-comments-copied](../../../../raw/postgres-12/src/test/regress/sql/create_table_like.sql#L104-L146), [create_table_like.out#index-comments-copied](../../../../raw/postgres-12/src/test/regress/expected/create_table_like.out#L352-L373)). Default dump/restore preserves comments, but either `pg_dump --no-comments` or `pg_restore --no-comments` omits them ([pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L9491-L9502), [pg_backup_archiver.c#restore-no-comments](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L2857-L2865)). `DROP INDEX` removing the comment is expected behavior, documented as comments being dropped with their object ([comment.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L88-L101)).

### Locking, permissions, and visibility

`CommentObject()` resolves the target through `get_object_address()` with `ShareUpdateExclusiveLock` and then demands ownership ([comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L66-L78)). For an index, that ownership check is `pg_class_ownercheck` ([objectaddress.c#check_object_ownership](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2266-L2289)), and the accepted relkinds are `RELKIND_INDEX` and `RELKIND_PARTITIONED_INDEX` ([objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1246-L1262)). The grammar production is `COMMENT ON comment_type_any_name any_name IS comment_text` ([gram.y#CommentStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6395-L6403)).

Measured consequences on the pinned build:

- Inside an explicit transaction, `COMMENT ON INDEX` held a `ShareUpdateExclusiveLock` on the **target index**, with no target lock on its heap. That is not the statement's only catalog lock: `CreateComments()` also opens `pg_description` with `RowExclusiveLock` and keeps that lock to transaction end ([comment.c#CreateComments-catalog-lock](../../../../raw/postgres-12/src/backend/commands/comment.c#L172-L224)).
- A non-owner got `ERROR: must be owner of index i_k`.
- The same non-owner read the baseline back successfully. Comments have no read privilege gate at all, which the documentation states explicitly and warns about ([comment.sgml#notes](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L274-L286)).

The conflict table shows that `ShareUpdateExclusiveLock` conflicts with another lock of the same mode plus `ShareLock`, `ShareRowExclusiveLock`, `ExclusiveLock`, and `AccessExclusiveLock` ([lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L60-L105)). A capture therefore serializes against another capture and the conflicting index-maintenance/DDL paths. It does not conflict with the `RowExclusiveLock` that lazy VACUUM takes on indexes ([vacuumlazy.c:283-284](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L284)) or the `AccessShareLock` ANALYZE takes ([analyze.c:419-421](../../../../raw/postgres-12/src/backend/commands/analyze.c#L419-L421)).

The detection query itself opens each measured index and heap with `AccessShareLock`, one call at a time ([dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)). Its `lock_timeout` limits waits for a conflicting `AccessExclusiveLock`; it does not turn the two size calls into one coherent snapshot. The baseline is world-readable, so it must never encode anything sensitive. In verbose mode, `psql`'s `\di+` selects `obj_description(c.oid, 'pg_class')` into its `Description` column, making the machine value look like an ordinary human comment ([describe.c#listTables-description](../../../../raw/postgres-12/src/bin/psql/describe.c#L3610-L3624), [describe.c#listTables-verbose](../../../../raw/postgres-12/src/bin/psql/describe.c#L3682-L3700)).

### Cost of the survey, and the relpages variant

With 300 indexes in one database on the pinned build, the original size-based detection query ran in 5.3-7.1 ms on a warm, idle server. Each `pg_relation_size` call probes segment files until the first missing segment, so the work scales with the number of measured relations plus their segment files; very large relations can require more `stat()` calls ([dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308)).

A catalog-only variant that reads `pg_class.relpages` instead of calling `pg_relation_size` ran in 3.7-3.8 ms and matched the live file size for all 300 freshly analyzed indexes. It is not equivalent. `VACUUM` and `ANALYZE` update relation statistics through `vac_update_relstats()` ([vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1156-L1197)); `CREATE INDEX` and `REINDEX` also update `relpages` through `index_update_stats()` ([index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2679), [index.c#index_update_stats-relpages](../../../../raw/postgres-12/src/backend/catalog/index.c#L2761-L2775)); and heap rewrites update or swap the values ([cluster.c#copy_heap_data-relpages](../../../../raw/postgres-12/src/backend/commands/cluster.c#L950-L969), [cluster.c#swap_relation_files-relpages](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1131-L1148)). Ordinary DML does not keep the field current. After inserting 100,000 rows without analyzing, the measured `relpages` value still said 8 blocks against a live 281. Use `pg_relation_size` for detection; use a `relpages` approximation only when its update history is known.

### Partitioned indexes

A partitioned index is relkind `'I'` and has no storage: `pg_relation_size()` returned 0 for the parent while the leaf index returned 8192 on the same test. Commenting the parent is allowed, but a parent-level ratio is meaningless. Combined with the `ALTER COLUMN TYPE` overwrite hazard above, the practical rule is to store baselines only on leaf indexes, and to re-capture every leaf baseline after any partitioned-table column type change.

### Fixture thresholds, and what to do with a flagged index

These values separate the workloads on this page. They are fixture-calibration points, not production defaults:

| AM | usable in these fixtures? | observed separating drift | why |
|---|---|---|---|
| btree | yes | 1.30 | ordinary 4x growth reached 1.058; five 20%-key-churn rounds reached 1.662 |
| spgist | yes | 1.30 | 1.004 under normal growth |
| bloom | yes | 1.30 | 0.997 under normal growth |
| gist | with care | 1.35 | fresh ratio itself moved 3.1% over 16x growth |
| hash | no | — | healthy sawtooth spans 1.50x |
| gin | no | — | healthy ratio fell 36% over 16x growth |
| brin | no | — | index size is nearly constant; ratio decays with table size |

A flagged index means only “the index grew faster than its heap since the baseline.” Before acting, compare the current index and heap bytes with their captured components or maintenance history, check whether the heap recently shrank or widened, and use an access-method-specific diagnostic where one exists. `n_dead_tup` provides lagged tuple-count context; it does not confirm heap bloat. If the investigation justifies a rebuild, choose plain or concurrent `REINDEX` according to the required lock and availability tradeoff; `VACUUM FULL` and `CLUSTER` are broader heap rewrites that also rebuild the table's indexes ([index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3436-L3515), [cluster.c#finish_heap_swap](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1341-L1411)). Capture a new baseline after the chosen rebuild.

The scheme's blind spot cannot be fixed by tuning the threshold: in the mass-delete fixture, ordinary `VACUUM` left drift at exactly 1.000 while a rebuild reclaimed up to 80% of the index file. Do not make this ratio the installation's only retained-space monitor.

### Why this exists: the contrib boundary

The reason a home-grown ratio is attractive at all is that v12 has no core function that reports per-index bloat, and the contrib tools do not expose one uniform result for every access method. The generic `pgstattuple(regclass)` dispatcher handles heaps plus B-tree, hash, and GiST indexes; it rejects GIN, SP-GiST, BRIN, unknown index AMs, and partitioned indexes ([pgstattuple.c#pgstat_relation](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L240-L313)). The extension does have a separate `pgstatginindex(regclass)`, but that reads only the GIN metapage version, pending-page count, and pending-tuple count—not a live/dead/free-space census ([pgstatindex.c#pgstatginindex](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L480-L506), [pgstatindex.c#pgstatginindex-fields](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L564)). Installing the extension needs superuser: its control file does not set `superuser = false`, and the control-file parser defaults that field to true ([extension.c#read_extension_control_file](../../../../raw/postgres-12/src/backend/commands/extension.c#L605-L625)), enforced at script execution ([extension.c#execute_extension_script](../../../../raw/postgres-12/src/backend/commands/extension.c#L798-L810)).

`pg_relation_size` has neither extension restriction: for an ordinary physical index, it measures the main-fork files without dispatching to the index access method ([dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)). That uniform allocation measurement—not uniform bloat semantics—is the real argument for this design. Custom access methods were not tested.

One access-method-specific size confounder worth naming: GIN keeps a pending list. `gin_pending_list_limit` is a cleanup trigger, not a hard size bound: after the threshold is exceeded, insertion requests a non-forced cleanup that can contend with another cleanup process ([ginfast.c#pending-list-cleanup-trigger](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L461)). The setting defaults to 4 MB and is `PGC_USERSET`, so it has session/transaction scope, and GIN also accepts it as a per-index reloption ([guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184), [ginutil.c#ginoptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L624)). Pending-list state can therefore move file size independently of rebuild-reclaimable space.

### Catalog, generated-header, and extension boundary

The proposal writes an existing catalog; it does not add a catalog column or change the server build. `pg_description.h` defines the catalog and includes generated `pg_description_d.h`. It also states that `genbki.pl` assembles initial descriptions and `initdb` loads them ([pg_description.h#catalog-generation](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L1-L41), [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L43-L64)). The catalog makefile lists `pg_description.h` among catalog inputs, derives `_d.h` names, and feeds those headers to the BKI-generation rule ([catalog/Makefile#catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L58), [catalog/Makefile#bki-generation](../../../../raw/postgres-12/src/backend/catalog/Makefile#L71-L100)). The generated `_d.h` file is therefore a build artifact, not a missing source dependency in the pinned checkout.

At runtime, `COMMENT` writes the existing `pg_description` row through `CreateComments()` ([comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225)). Detection needs no extension. The `bloom` measurements do cross into contrib and require that extension, while `pgstattuple` and `pageinspect` appear only as validation aids in this page's experiments.

### Tests in the pinned checkout

- Comment survival across `REINDEX TABLE` and `REINDEX TABLE CONCURRENTLY` is directly covered ([create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2116)).
- Basic `COMMENT ON INDEX` creation and deletion are covered ([create_index.sql#index-comments](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L42-L44), [create_index.out#index-comments](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L29-L33)).
- `CREATE TABLE ... LIKE INCLUDING ALL` copying source-index comments is covered ([create_table_like.sql#index-comments-copied](../../../../raw/postgres-12/src/test/regress/sql/create_table_like.sql#L104-L146), [create_table_like.out#index-comments-copied](../../../../raw/postgres-12/src/test/regress/expected/create_table_like.out#L352-L373)).
- The partitioned-index comment overwrite is covered, and the expected output records it as known-wrong behavior ([alter_table.sql#partitioned-comments](../../../../raw/postgres-12/src/test/regress/sql/alter_table.sql#L1421-L1440), [alter_table.out#partitioned-comments](../../../../raw/postgres-12/src/test/regress/expected/alter_table.out#L2103-L2134)).
- The closest upstream analogue of the delete-and-reload cycle is the B-tree multilevel-page-deletion case: insert 80,000 rows, add a primary key, delete all but 10, `VACUUM`, then insert 1,000 more, with a comment stating it "also creates coverage for nbtree FSM page recycling" ([btree_index.sql#multilevel-page-deletion](../../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L144-L162), [btree_index.out#multilevel-page-deletion](../../../../raw/postgres-12/src/test/regress/expected/btree_index.out#L315-L332)). It exists for WAL-record and page-deletion coverage and asserts nothing about index size, so it does not contradict or confirm the growth measured above.
- There is **no** upstream test that compares index size to heap size, none that exercises a comment as machine-readable data, and none that asserts an index's block count after a delete-and-reload cycle. Everything in the measurement tables above was produced on a purpose-built 12.2 server, not by the regression suite.

## Context Reviewed

- `COMMENT` path end to end: grammar production, `CommentObject`, `CreateComments`, `DeleteComments`, `GetComment`, the `pg_description` catalog definition, and the `obj_description` builtin.
- Object resolution and privileges for indexes: `get_object_address`, `get_relation_by_qualified_name`, `check_object_ownership`, and the lock conflict table.
- Comment lifecycle across DDL: `index_concurrently_swap`, `RelationTruncateIndexes`, `CREATE TABLE ... LIKE` comment cloning, default and `--no-comments` dump/restore behavior, and the `create_index`, `create_table_like`, and `alter_table` regression suites.
- Size measurement: `pg_relation_size` one- and two-argument forms, relation opening and missing-relation behavior, `calculate_relation_size`, `calculate_table_size`, `calculate_toast_table_size`, `pg_table_size`, `pg_indexes_size`, and all inspected writers of `pg_class.relpages` relevant to the catalog-only alternative.
- Heap baseline behavior: ordinary VACUUM's conditional tail truncation, and the heap/index rebuild boundaries in `REINDEX`, `VACUUM FULL`, and `CLUSTER`.
- Per-access-method storage behavior: default fill factors; hash splitpoint allocation; BRIN pages-per-range, revmap capacity, and revmap extension; and the main-FSM, GIN-entry, SP-GiST redirect, hash-overflow-map, contrib `bloom`, and BRIN bulk-delete paths.
- Update behavior: the indexed-attribute bitmap and HOT decision in `RelationGetIndexAttrBitmap()` and `heap_update()`.
- Catalog and build boundary: `pg_description.h`, generated `pg_description_d.h`, the catalog BKI make rules, and reverse include users in comment, index, and syscache code.
- Adjacent observability: `pg_stat_all_tables`, statistics lag and snapshot semantics, `psql` verbose index display, `track_counts`, both query timeouts, GIN's global and per-index pending-list cleanup settings, and the `pgstattuple` access-method support matrix with its extension privilege gate.
- Prior exact-pin measurements recorded by the original page: fresh-ratio scale checks, a hash/B-tree row-count sweep, six workload scenarios, maintenance/comment lifecycle probes, lock and visibility probes, and survey timings over 300 indexes. This review rechecked the source claims and production queries; it did not independently reproduce every historical measurement table.
- Exact-pin delete-and-reload measurements: all seven access methods at 200,000 rows; the one-, two-, and three-pass post-delete arms rerun during this review, with the one-pass arm repeated; a GIN metapage census after three passes; and a zero-byte mid-cycle heap. The rerun corrected the recorded GIN counts from 4,102/7,495 to 4,101/7,482 blocks, confirmed SP-GiST's +0.83% result, and corrected the zero-heap query behavior.

## Evidence Map

| Claim | Evidence |
|---|---|
| `COMMENT ON INDEX` takes `ShareUpdateExclusiveLock` on the index and requires ownership | [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L66-L78), [objectaddress.c#check_object_ownership](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2266-L2289), [objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1246-L1262), [gram.y#CommentStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6395-L6403) |
| One comment per index, stored as a single replaceable `pg_description` row with `objsubid = 0`; empty string deletes it | [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L48-L57), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225), [comment.c:154-156](../../../../raw/postgres-12/src/backend/commands/comment.c#L154-L156) |
| The baseline is readable by any user in the database and has no privilege gate | [comment.sgml#notes](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L274-L286), [pg_proc.dat#obj_description](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2311-L2314) |
| The comment follows the index through `REINDEX CONCURRENTLY`, which changes the index OID | [index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656), [create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2116) |
| `ALTER TABLE ... ALTER COLUMN TYPE` on a partitioned table overwrites child index comments with the parent's | [alter_table.sql#partitioned-comments](../../../../raw/postgres-12/src/test/regress/sql/alter_table.sql#L1421-L1440), [alter_table.out#partitioned-comments](../../../../raw/postgres-12/src/test/regress/expected/alter_table.out#L2103-L2134) |
| Default dump/restore preserves baselines; `--no-comments` can omit them | [pg_dump.c#dumpIndex-comment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16456-L16463), [pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L9491-L9502), [pg_backup_archiver.c#restore-no-comments](../../../../raw/postgres-12/src/bin/pg_dump/pg_backup_archiver.c#L2857-L2865) |
| `CREATE TABLE ... LIKE INCLUDING ALL` copies source-index comments into new indexes | [parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-12/src/backend/parser/parse_utilcmd.c#L1181-L1221), [create_table_like.out#index-comments-copied](../../../../raw/postgres-12/src/test/regress/expected/create_table_like.out#L352-L373) |
| `pg_relation_size(regclass)` opens the relation and measures the main fork by `stat()`ing each segment | [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336) |
| `pg_table_size` adds every fork plus TOAST and its index, which destroys the ratio | [dbsize.c#calculate_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L380-L408), [dbsize.c#calculate_toast_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L338-L378), [dbsize.c#pg_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L450-L467) |
| Ordinary index VACUUM does not truncate the main fork; heap VACUUM can conditionally trim empty tail pages | [storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L229-L295), [vacuumlazy.c#should_attempt_truncation](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1850-L1867), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1870-L1970), [spgvacuum.c#truncate-disabled](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886) |
| B-tree, GiST, GIN, and SP-GiST VACUUM record reusable pages in the index FSM rather than shrinking the file | [nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1078-L1095), [gistvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L255-L265), [ginvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L790), [spgvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L848-L861) |
| B-tree and GiST gate free-page reuse on `RecentGlobalXmin`, so the VACUUM that deletes a page cannot recycle it in the same pass | [nbtpage.c#_bt_page_recyclable](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L940-L963), [gistutil.c#gistPageRecyclable](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L881-L906), [nbtree.c#btvacuumpage-recycle](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1179), [gistvacuum.c#gistvacuum-recycle](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L298-L320) |
| B-tree free-page reuse is key-order agnostic: `_bt_getbuf` takes whatever block the FSM reports with at least half a page free | [nbtpage.c#_bt_getbuf-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L801-L842), [indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L33-L46), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L49-L55) |
| GIN retains an entry tuple when its posting list becomes empty; its cleanup separately gates recyclable pages on `RecentGlobalXmin` | [ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556), [ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L786) |
| SP-GiST records new or empty non-root pages in the FSM, but redirect-to-placeholder cleanup has a `RecentGlobalXmin` gate | [spgvacuum.c#spgvacuumpage](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L607-L670), [spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L482-L545) |
| Contrib `bloom` records new or deleted pages in the FSM | [blvacuum.c#blvacuumcleanup](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L189-L214) |
| Hash does not use the index FSM at all; it frees and reuses overflow pages through its own bitmap pages | [hashovfl.c:632](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L632), [hashovfl.c#_hash_getovflpage-setbit](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L313-L329) |
| BRIN removes nothing on delete, so its size does not respond to a delete-and-reload cycle | [brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784) |
| Updating an indexed attribute prevents a HOT update and requires index maintenance | [relcache.c#RelationGetIndexAttrBitmap](../../../../raw/postgres-12/src/backend/utils/cache/relcache.c#L4782-L4806), [heapam.c#HOT-decision](../../../../raw/postgres-12/src/backend/access/heap/heapam.c#L3588-L3603) |
| Hash index size is a step function of estimated tuples, allocated a whole splitpoint at a time at fillfactor 75 | [hashpage.c#_hash_init-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L505-L525), [hashpage.c#_hash_expandtable-splitpoint](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L785-L853), [hash.h:279](../../../../raw/postgres-12/src/include/access/hash.h#L279) |
| BRIN revmap size changes in steps as the number of summarized heap ranges crosses page capacity | [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-12/src/include/access/brin.h#L39-L43), [brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-12/src/include/access/brin_page.h#L88-L94), [brin_revmap.c#brinRevmapExtend](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L115-L123) |
| GIN's pending-list limit triggers a non-forced cleanup and is also a per-index reloption | [ginfast.c#pending-list-cleanup-trigger](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L461), [ginutil.c#ginoptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L624) |
| Default fill factors differ per access method, so fresh ratios differ per access method | [nbtree.h:169](../../../../raw/postgres-12/src/include/access/nbtree.h#L169), [hash.h:279](../../../../raw/postgres-12/src/include/access/hash.h#L279), [spgist.h:25](../../../../raw/postgres-12/src/include/access/spgist.h#L25), [gist_private.h:465](../../../../raw/postgres-12/src/include/access/gist_private.h#L465) |
| A `CASE` guard is the documented way to force evaluation order around the numeric cast | [syntax.sgml#expression-evaluation](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L2500-L2527) |
| Partitioned indexes are relkind `'I'` and have no storage | [pg_class.h#relkind](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L163), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L185-L192) |
| `relpages` is updated by VACUUM/ANALYZE and by relevant index-build and heap-rewrite paths, but ordinary DML can leave it stale | [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1156-L1197), [index.c#index_update_stats-relpages](../../../../raw/postgres-12/src/backend/catalog/index.c#L2761-L2775), [cluster.c#swap_relation_files-relpages](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1131-L1148) |
| `n_live_tup` and `n_dead_tup` are lagged estimates, not heap-compaction measurements | [monitoring.sgml#statistics-lag](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L248), [monitoring.sgml#estimated-tuples](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2773-L2785) |
| `pg_description_d.h` and the BKI input are generated from catalog headers during the build | [pg_description.h#catalog-generation](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L1-L41), [catalog/Makefile#catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L58), [catalog/Makefile#bki-generation](../../../../raw/postgres-12/src/backend/catalog/Makefile#L71-L100) |
| Capture conflicts with `REINDEX` but not with VACUUM's or ANALYZE's index locks | [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L60-L105), [vacuumlazy.c:283-284](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L284), [analyze.c:419-421](../../../../raw/postgres-12/src/backend/commands/analyze.c#L419-L421) |
| Generic `pgstattuple(regclass)` covers heaps, B-tree, hash, and GiST; separate `pgstatginindex` reports only GIN pending-list metadata; extension installation needs superuser | [pgstattuple.c#pgstat_relation](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L240-L313), [pgstatindex.c#pgstatginindex-fields](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L564), [extension.c#read_extension_control_file](../../../../raw/postgres-12/src/backend/commands/extension.c#L605-L625), [extension.c#execute_extension_script](../../../../raw/postgres-12/src/backend/commands/extension.c#L798-L810) |
| Query timeouts used above are session/transaction scope | [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396) |

## Open Questions

- Every number in the measurement tables comes from one purpose-built 12.2 server with synthetic fixtures (200k-1.6M rows, one 40-byte payload column, one index per table). The per-access-method drift thresholds are calibrated to those fixtures, not to any production workload. Different key widths, key distributions, fill factors, or multi-index tables will move them.
- This review did not independently reproduce every historical measurement table. It rechecked their surrounding source claims, arithmetic definitions, and production queries. The page therefore remains `verified_by_agent: not yet`.
- The GiST and GIN cases where `REINDEX` produced a *larger* index than incremental insertion were observed but not traced to the responsible build path in source; they are reported as measurements only.
- Whether the partitioned-child comment overwrite has a source-level fix in a later minor release of the 12 branch was not checked, and cannot be checked against another major version under this wiki's evidence rules.
- The scheme was not tested against `CREATE ACCESS METHOD` custom index AMs, only against the six core access methods plus contrib `bloom`. The core size function reads a physical relation's fork without consulting its index AM, but nothing was measured about a custom AM's storage or whether its fresh ratio is scale-invariant.
- The 300-index survey timings were taken on a warm, idle, single-user server with all relation files in page cache. Cold-cache `stat()` cost on an installation with tens of thousands of relations was not measured.
- The delete-and-reload cycle ran on a single-user server with `autovacuum = off` and no concurrent transactions, which is the friendliest possible setting for the `RecentGlobalXmin` gate. A long-running concurrent snapshot would hold that horizon back and delay recycling past any number of `VACUUM` passes; that case was not measured. On a real server autovacuum also decides the `VACUUM` count for you, so the one-, two- and three-pass arms bracket the behavior rather than predict it.
- The cycle used strictly ascending keys in both loads. Reusing the first load's values or using random keys was not measured. B-tree and GiST both ask the FSM for a reusable page before extending ([nbtpage.c#_bt_getbuf-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L801-L842), [gistutil.c#gistNewBuffer](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L802-L840)), but the resulting block counts under other key orders remain unverified.
- Immediately after one back-to-back `DELETE`; `VACUUM` pair, `pg_stat_all_tables.n_dead_tup` read 200,000 for a zero-byte heap; two seconds later, a repeat read 0. The documented collector lag and transaction-local statistics snapshot explain why the value is not a physical heap invariant, but the precise flush ordering in that experiment was not traced ([monitoring.sgml#statistics-lag](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L248)).
- Three of the cycle's supporting measurements — the B-tree `pgstatindex` page census, the hash `hash_metapage_info` bucket and overflow comparison, and the GIN `gin_metapage_info` entry census — come from `pgstattuple` and `pageinspect`, which are exactly the contrib modules [Why this exists: the contrib boundary](#why-this-exists-the-contrib-boundary) rules out for the monitoring scheme itself. They are diagnostics for this page, not part of the proposal.

## Source References

- [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L39-L130)
- [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225)
- [comment.c#DeleteComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L317-L367)
- [gram.y#CommentStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6395-L6403)
- [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L48-L57)
- [catalog/Makefile#catalog-headers](../../../../raw/postgres-12/src/backend/catalog/Makefile#L32-L58)
- [catalog/Makefile#bki-generation](../../../../raw/postgres-12/src/backend/catalog/Makefile#L71-L100)
- [pg_proc.dat#obj_description](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2311-L2314)
- [objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1246-L1262)
- [objectaddress.c#check_object_ownership](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2266-L2289)
- [index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)
- [index.c#index_update_stats](../../../../raw/postgres-12/src/backend/catalog/index.c#L2655-L2775)
- [index.c#reindex_index](../../../../raw/postgres-12/src/backend/catalog/index.c#L3436-L3515)
- [cluster.c#finish_heap_swap](../../../../raw/postgres-12/src/backend/commands/cluster.c#L1341-L1411)
- [heap.c#RelationTruncateIndexes](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3163-L3207)
- [storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L229-L295)
- [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308)
- [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)
- [dbsize.c#calculate_toast_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L338-L378)
- [dbsize.c#calculate_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L380-L408)
- [dbsize.c#pg_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L450-L467)
- [dbsize.c#pg_indexes_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L469-L486)
- [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)
- [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1156-L1197)
- [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1955-L1970)
- [vacuumlazy.c:283-284](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L284)
- [analyze.c:419-421](../../../../raw/postgres-12/src/backend/commands/analyze.c#L419-L421)
- [nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1078-L1095)
- [nbtree.c#btvacuumpage-recycle](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1179)
- [nbtpage.c#_bt_getbuf-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L801-L842)
- [nbtpage.c#_bt_page_recyclable](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L940-L963)
- [indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L33-L46)
- [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L49-L55)
- [gistvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L255-L265)
- [gistvacuum.c#gistvacuum-recycle](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L298-L320)
- [gistutil.c#gistPageRecyclable](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L881-L906)
- [gistutil.c#gistNewBuffer](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L802-L840)
- [ginvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L790)
- [ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556)
- [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L786)
- [ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138)
- [ginbulk.c#ginGetBAEntry](../../../../raw/postgres-12/src/backend/access/gin/ginbulk.c#L255-L280)
- [gininsert.c#ginbuild-entry-insert](../../../../raw/postgres-12/src/backend/access/gin/gininsert.c#L380-L406)
- [spgvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L848-L861)
- [spgvacuum.c#spgvacuum-fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L650-L670)
- [spgvacuum.c#vacuumRedirectAndPlaceholder](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L482-L545)
- [spgvacuum.c#truncate-disabled](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886)
- [blvacuum.c#blvacuumcleanup](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L189-L214)
- [brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)
- [hashpage.c#_hash_init-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L505-L525)
- [hashpage.c#_hash_expandtable-splitpoint](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L785-L853)
- [hashovfl.c#_hash_getovflpage-setbit](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L313-L329)
- [hashovfl.c:632](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L632)
- [nbtree.h:169](../../../../raw/postgres-12/src/include/access/nbtree.h#L169)
- [hash.h:279](../../../../raw/postgres-12/src/include/access/hash.h#L279)
- [spgist.h:25](../../../../raw/postgres-12/src/include/access/spgist.h#L25)
- [gist_private.h:465](../../../../raw/postgres-12/src/include/access/gist_private.h#L465)
- [rel.h:279](../../../../raw/postgres-12/src/include/utils/rel.h#L279)
- [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-12/src/include/access/brin.h#L39-L43)
- [brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-12/src/include/access/brin_page.h#L88-L94)
- [brin_revmap.c#brinRevmapExtend](../../../../raw/postgres-12/src/backend/access/brin/brin_revmap.c#L115-L123)
- [pg_index.h#pg_index](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L35)
- [pg_class.h#relkind](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L163)
- [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L185-L192)
- [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L60-L105)
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)
- [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1399)
- [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386)
- [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)
- [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184)
- [ginfast.c#pending-list-cleanup-trigger](../../../../raw/postgres-12/src/backend/access/gin/ginfast.c#L438-L461)
- [ginutil.c#ginoptions](../../../../raw/postgres-12/src/backend/access/gin/ginutil.c#L602-L624)
- [pg_dump.c#dumpIndex-comment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16456-L16463)
- [pg_dump.c#dumpComment-no-comments](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L9491-L9502)
- [parse_utilcmd.c#LIKE-index-comments](../../../../raw/postgres-12/src/backend/parser/parse_utilcmd.c#L1181-L1221)
- [describe.c#listTables-verbose](../../../../raw/postgres-12/src/bin/psql/describe.c#L3682-L3700)
- [pgstattuple.c#pgstat_relation](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L240-L313)
- [pgstatindex.c#pgstatginindex-fields](../../../../raw/postgres-12/contrib/pgstattuple/pgstatindex.c#L509-L564)
- [extension.c#read_extension_control_file](../../../../raw/postgres-12/src/backend/commands/extension.c#L605-L625)
- [extension.c#execute_extension_script](../../../../raw/postgres-12/src/backend/commands/extension.c#L798-L810)
- [comment.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L88-L101)
- [comment.sgml#notes](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L274-L286)
- [syntax.sgml#expression-evaluation](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L2500-L2527)
- [monitoring.sgml#statistics-lag](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L229-L248)
- [monitoring.sgml#estimated-tuples](../../../../raw/postgres-12/doc/src/sgml/monitoring.sgml#L2773-L2785)
- [create_table_like.sql#index-comments-copied](../../../../raw/postgres-12/src/test/regress/sql/create_table_like.sql#L104-L146)
- [create_table_like.out#index-comments-copied](../../../../raw/postgres-12/src/test/regress/expected/create_table_like.out#L352-L373)
- [create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854)
- [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2116)
- [alter_table.sql#partitioned-comments](../../../../raw/postgres-12/src/test/regress/sql/alter_table.sql#L1421-L1440)
- [alter_table.out#partitioned-comments](../../../../raw/postgres-12/src/test/regress/expected/alter_table.out#L2103-L2134)
- [btree_index.sql#multilevel-page-deletion](../../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L144-L162)
- [btree_index.out#multilevel-page-deletion](../../../../raw/postgres-12/src/test/regress/expected/btree_index.out#L315-L332)

## Navigation

- [v12/index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [Measuring B-Tree Index Bloat With Core SQL Only in PostgreSQL 12 (unverified)](btree-index-bloat-core-sql-only.md)
- [How pgstatindex Calculates B-Tree Index Statistics in PostgreSQL 12 (unverified)](how-pgstatindex-calculates-information.md)
- [B-Tree Leaf Density vs Fragmentation Impact on Index Scan I/O in PostgreSQL 12 (unverified)](leaf-density-vs-fragmentation-index-scan-io.md)
- [How REINDEX INDEX CONCURRENTLY Is Implemented in PostgreSQL 12 (unverified)](reindex-index-concurrently.md)
- [Planner Penalties for Bloated Indexes in PostgreSQL 12 (unverified)](../query-planning/bloated-indexes-query-planner.md)
