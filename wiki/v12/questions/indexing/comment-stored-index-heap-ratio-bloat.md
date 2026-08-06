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
  - [Why COMMENT is the right slot in core v12](#why-comment-is-the-right-slot-in-core-v12)
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
  - [Thresholds, and what to do with a flagged index](#thresholds-and-what-to-do-with-a-flagged-index)
  - [Why this exists: the contrib boundary](#why-this-exists-the-contrib-boundary)
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

It works, but only for four of the seven index access methods available in a v12 tree, and only as a screening signal that must be read next to a heap-side signal. The design is sound in one specific way: `pg_relation_size()` is the only bloat-relevant measurement in core v12 that is identical for every access method, and an index comment is the only per-index writable slot in core that survives `REINDEX`, `REINDEX CONCURRENTLY`, `VACUUM FULL`, `CLUSTER`, rename, and `pg_dump`. Everything else about the scheme is a tradeoff, and the measurements below quantify each one.

The one-line summary of the mathematics, which drives every result on this page:

```text
drift = current_ratio / baseline_ratio = (index growth factor) / (heap growth factor)
```

Index bloat is the numerator alone. The ratio hands you the numerator divided by an unknown denominator.

### The proposal

Three moving parts.

1. **Capture.** Immediately after the index is built or rebuilt *and* the table is vacuumed, compute `pg_relation_size(index) / pg_relation_size(table)` and store the bare number as the index's comment. `pg_relation_size(regclass)` is the one-argument form that resolves to the `'main'` fork, defined in the catalog as a SQL wrapper over the two-argument C function ([pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891)), and it measures the actual files by `stat()`ing every segment ([dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336)).
2. **Store.** `COMMENT ON INDEX <index> IS '<number>'` writes one `pg_description` row keyed by `(objoid, classoid, objsubid)` with `objsubid = 0` for a whole relation ([pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L48-L57), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225)). Re-running it replaces the row rather than appending; an empty string deletes it ([comment.c:154-156](../../../../raw/postgres-12/src/backend/commands/comment.c#L154-L156)).
3. **Detect.** Periodically recompute the ratio, divide by the stored baseline, and flag indexes whose drift exceeds a per-access-method threshold. Read the baseline with `obj_description(indexrelid, 'pg_class')`, which is a SQL-language builtin selecting the `objsubid = 0` row ([pg_proc.dat#obj_description](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2311-L2314)).

### Why COMMENT is the right slot in core v12

The competing places to keep a per-index baseline all lose it exactly when it matters:

| Keyed by | Survives `REINDEX` | Survives `REINDEX CONCURRENTLY` | Survives `pg_dump`/restore |
|---|---|---|---|
| `pg_index.indexrelid` in your own table | yes | **no** — the OID changes | no |
| `relfilenode` in your own table | **no** | no | no |
| Index name in your own table | yes | yes | your table must be dumped too |
| Index comment | yes | yes | yes |

The `REINDEX CONCURRENTLY` case is the interesting one. `index_concurrently_swap()` explicitly rewrites the `pg_description` row's `objoid` from the old index to the new one, under the header comment `Move comment if any` ([index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)). Upstream regression coverage asserts exactly this behavior for both `REINDEX TABLE` and `REINDEX TABLE CONCURRENTLY` ([create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2116)). `pg_dump` emits a `COMMENT ON INDEX` for every dumped index ([pg_dump.c#dumpIndex-comment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16456-L16463)).

There is no per-index user-defined storage parameter in v12 to abuse instead, and `pg_description` costs one small catalog row per index.

### What the ratio actually measures

Since `current_ratio / baseline_ratio` expands to `(idx1/idx0) / (heap1/heap0)`, the drift number conflates two independent quantities. The scheme is only a bloat measure when the heap growth factor is 1, or is known separately.

Three structural facts about v12 fix the shape of both factors:

- **Index files never shrink outside a rebuild.** The only `MAIN_FORKNUM` truncation entry point is `RelationTruncate()` ([storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L229-L295)), and its call sites are the heap vacuum truncation ([vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1955-L1970)), the non-transactional heap truncate ([heapam_handler.c:629-633](../../../../raw/postgres-12/src/backend/access/heap/heapam_handler.c#L629-L633)), and the `TRUNCATE` path that empties and rebuilds indexes under `AccessExclusiveLock` ([heap.c#RelationTruncateIndexes](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3163-L3207)). SP-GiST contains index truncation code, but it is compiled out under `#ifdef NOT_USED`, with a comment stating it is unsafe against concurrent inserts and that "btree doesn't do this either" ([spgvacuum.c#truncate-disabled](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886)).
- **Index VACUUM returns pages to the free space map instead.** B-tree ([nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1078-L1095)), GiST ([gistvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L255-L265)), GIN ([ginvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L790)), and SP-GiST ([spgvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L848-L861)) all call `RecordFreeIndexPage`/`IndexFreeSpaceMapVacuum`. Reclaimed space is reusable but invisible to file size.
- **The heap can shrink.** `lazy_truncate_heap()` drops trailing empty pages, so a heap that loses its newest rows really does get smaller while every index keeps its blocks.

That asymmetry means drift is a ratchet upward for real index bloat, and is also extremely sensitive to anything that moves the heap.

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

- **BRIN.** The index stayed at exactly 24,576 bytes at every scale. A BRIN index summarizes `pages_per_range` heap pages per tuple, defaulting to 128 ([brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-12/src/include/access/brin.h#L39-L43)), and the revmap packs many range pointers per page ([brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-12/src/include/access/brin_page.h#L88-L94)). The ratio decays roughly as `1/table_size`, so BRIN indexes will always look like they are *improving*.
- **GIN.** Index size tracks distinct keys and posting-list compression, not row count. With 10,000 distinct array elements, 16x more rows produced only 10x more index, so the healthy ratio fell 36%.
- **Hash.** See the next section.

B-tree, SP-GiST, GiST and contrib `bloom` hold their fresh ratio to within 0.7%, 0.7%, 3.1% and 0.9% respectively over a 16x growth, which is the precondition the whole scheme needs.

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

The healthy hash ratio oscillates between 0.3314 and 0.4985, a 1.50x swing with zero bloat, while B-tree moves 0.5% over the same 9x range. Any drift threshold below 1.5 produces false positives on hash; any threshold above it will not fire on real hash bloat either.

### Measured detection power on seven access methods

Each fixture: 200,000 rows, index built fresh, baseline captured, a workload applied, then the index rebuilt with `REINDEX` to get true index bloat and the table `VACUUM FULL`-ed to get true heap bloat. `idx_factor` and `heap_factor` are the two components of drift.

**Index-heavy churn** (five rounds updating 20% of rows' indexed key, `VACUUM` after each round):

| AM | baseline | current | drift | idx_factor | heap_factor | true index bloat |
|---|---|---|---|---|---|---|
| btree | 0.267216 | 0.444040 | **1.662** | 1.995 | 1.200 | 0.499 |
| gin | 0.176978 | 0.239688 | **1.354** | 1.625 | 1.200 | 0.381 |
| hash | 0.398642 | 0.465455 | 1.168 | 1.401 | 1.200 | 0.271 |
| spgist | 0.455061 | 0.460007 | 1.011 | 1.213 | 1.200 | 0.176 |
| bloom | 0.191077 | 0.191111 | 1.000 | 1.201 | 1.200 | 0.167 |
| gist | 0.471183 | 0.433651 | 0.920 | 1.105 | 1.200 | 0.101 |
| brin | 0.001455 | 0.001212 | 0.833 | 1.000 | 1.200 | 0.000 |

The B-tree result is the headline: a single pass of key updates touching 20% of rows grew a freshly built B-tree by 1.995x. A fresh B-tree is packed to its default fill factor of 90 ([nbtree.h:169](../../../../raw/postgres-12/src/include/access/nbtree.h#L169)), so nearly every leaf page splits on first contact. The heap default fill factor is 100 ([rel.h:279](../../../../raw/postgres-12/src/include/utils/rel.h#L279)), so those updates could not be HOT and each added an index entry.

Note also that drift *understates* real bloat here — 1.662 against a true 1.995x index growth — purely because the heap grew 20% at the same time. Where the index and the heap bloat by the same factor, as with `bloom` (1.201 against 1.200), drift is exactly 1.000 and the scheme reports perfect health on a 17%-bloated index.

**Ordinary 4x growth with no maintenance** — the baseline for "what does normal look like":

| AM | drift | true index bloat |
|---|---|---|
| btree | 1.058 | 0.058 |
| spgist | 1.004 | 0.007 |
| bloom | 0.997 | 0.001 |
| gist | 0.942 | -0.079 |
| hash | 1.106 | 0.104 |
| gin | 0.773 | -0.056 |
| brin | 0.250 | 0.000 |

For B-tree, SP-GiST and bloom, ordinary growth moves drift by less than 6%, which cleanly separates it from the 1.66 churn signal. Negative "true bloat" for GiST and GIN is not a measurement error: rebuilding those two produced a *larger* index than the incrementally built one, so "REINDEX shrinks it" is not universally true.

### The three heap-side failure modes

Each measured on all seven access methods.

**1. Shared bloat is invisible.** Delete 80% of rows, then `VACUUM`. Neither file shrinks, so drift is exactly 1.000 on all seven access methods, while true index bloat is 0.667 to 0.797 (BRIN 0.000):

| AM | drift | true index bloat | true heap bloat |
|---|---|---|---|
| btree | 1.000 | 0.797 | 0.800 |
| bloom | 1.000 | 0.797 | 0.800 |
| spgist | 1.000 | 0.789 | 0.800 |
| gist | 1.000 | 0.781 | 0.800 |
| hash | 1.000 | 0.684 | 0.800 |
| gin | 1.000 | 0.667 | 0.800 |
| brin | 1.000 | 0.000 | 0.800 |

This is the single most important limitation: the most common real-world bloat pattern, a mass delete followed by autovacuum, is completely invisible to a heap-relative ratio.

**2. Heap growth masks index bloat.** Widening every row to 400 bytes grew the heap 6.4x while the index doubled. Drift fell to 0.312 for B-tree — a "healthy" reading on an index carrying 49.9% bloat:

| AM | drift | idx_factor | heap_factor | true index bloat |
|---|---|---|---|---|
| btree | 0.312 | 1.995 | 6.388 | 0.499 |
| bloom | 0.313 | 1.997 | 6.388 | 0.499 |
| gist | 0.325 | 1.911 | 5.888 | 0.477 |
| hash | 0.340 | 2.174 | 6.388 | 0.540 |
| spgist | 0.349 | 2.012 | 5.763 | 0.503 |
| gin | 0.393 | 2.125 | 5.411 | 0.510 |
| brin | 0.157 | 1.000 | 6.388 | 0.000 |

**3. Heap truncation fakes bloat.** Deleting the newest 75% of rows by `id`, then `VACUUM`, let `lazy_truncate_heap()` drop the trailing heap pages while every index kept all its blocks. Drift landed at 3.995-3.998 on all seven access methods. On six of them the alarm is directionally right; on BRIN it is a pure false positive at 4.0x drift against 0.000 true bloat.

### A 200k-row delete-and-reload cycle test on all seven index types

This cycle is the scheme's best case, and it passes. The heap ends the cycle at exactly the size it started, so the heap growth factor is exactly 1.000 and drift collapses to pure index growth. Ratio drift matched raw index growth to the last reported digit on all seven access methods, and matched `REINDEX` ground truth on six of the seven.

What the cycle actually exposes is not a flaw in the ratio. It is that **one post-delete `VACUUM` is not enough**: three of the seven indexes grew 83-99% while the row count, the row width and the heap size all ended where they began. The ratio reported that growth faithfully.

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

-- ground truth: what a fresh index over the final 200k rows costs
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
       round(100.0 * (f.idx_bytes - r.idx_bytes) / r.idx_bytes, 2) AS true_bloat_pct
FROM      cyc.res i
JOIN      cyc.res f USING (obj)
JOIN      cyc.res r USING (obj)
WHERE i.phase = 'initial' AND f.phase = 'final' AND r.phase = 'rebuilt'
ORDER BY index_size_pct DESC, i.am;
```

#### Results

The heap measured 3,847 blocks before the cycle and 3,847 blocks after it, so `heap_size_pct` is `0.00` on every row and the ratio column carries the index change alone. Sizes are in 8 KB blocks; `rebuilt_blocks` is the `REINDEX` ground truth.

| AM | initial | final | index size % | baseline ratio | current ratio | ratio % | rebuilt | true bloat % |
|---|---|---|---|---|---|---|---|---|
| btree | 551 | 1098 | **+99.27** | 0.143228 | 0.285417 | **+99.27** | 551 | +99.27 |
| gist | 1538 | 2914 | **+89.47** | 0.399792 | 0.757473 | **+89.47** | 1538 | +89.47 |
| gin | 4102 | 7495 | **+82.72** | 1.066285 | 1.948271 | **+82.72** | 4102 | +82.72 |
| spgist | 1082 | 1091 | +0.83 | 0.281258 | 0.283598 | +0.83 | 1082 | +0.83 |
| bloom | 394 | 394 | 0.00 | 0.102417 | 0.102417 | 0.00 | 394 | 0.00 |
| brin | 3 | 3 | 0.00 | 0.000780 | 0.000780 | 0.00 | 3 | 0.00 |
| hash | 822 | 822 | 0.00 | 0.213673 | 0.213673 | 0.00 | 815 | +0.86 |

Three readings matter:

- **`ratio_pct` equals `index_size_pct` on all seven rows, digit for digit.** That is the arithmetic of a heap that returns to its starting size, and it is exactly the condition [What the ratio actually measures](#what-the-ratio-actually-measures) names as the only case in which drift *is* index bloat. Drift was 1.9927, 1.8947, 1.8272, 1.0083, 1.0000, 1.0000 and 1.0000, in the same order.
- **`ratio_pct` equals `true_bloat_pct` on six of seven rows.** Hash is the single exception, and its 0.86% gap is not stale space. `pageinspect`'s `hash_metapage_info` reports `maxbucket = 767` and `ovflpoint = 11` for both the initial and the rebuilt index; they differ only in overflow pages, 53 against 46, which is exactly the 7-block gap. The bucket count is fixed by the estimated tuple count at build time ([hashpage.c#_hash_init-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L505-L525)), so two builds over 200,000 rows land on the same bucket array and differ only in how the two key sets collide.
- **BRIN reported 0.00% on a workload that deleted and reinserted every row.** `brinbulkdelete()` is a documented no-op — "there are no per-heap-tuple index tuples in BRIN indexes, there's not a lot we can do here" ([brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784)) — so BRIN's 0.00% here is a correct reading, unlike its false positives elsewhere on this page.

#### Mid-cycle, the scheme reports nothing at all

Between step 6 and step 7 the heap is **0 bytes**: `DELETE` emptied every page and `lazy_truncate_heap()` dropped all 3,847 of them ([vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1955-L1970)), while all seven indexes kept every block. Measured at that point, the raw ratio is `NULL` for all seven indexes and [The detection query](#the-detection-query) returns **zero rows**, because its `pg_relation_size(i.indrelid) > 0` guard excludes them.

This is a distinct failure mode from the mass delete in [The three heap-side failure modes](#the-three-heap-side-failure-modes). Deleting *most* rows leaves both files intact and pins drift at exactly 1.000; deleting *all* rows makes the ratio undefined and the index invisible to the monitor.

#### Why the cycle splits the access methods: the recyclability gate

Rerunning the identical cycle with two and three post-`DELETE` `VACUUM` passes instead of one isolates the cause. Only the `VACUUM` count changed:

| AM | 1 VACUUM | 2 VACUUMs | 3 VACUUMs |
|---|---|---|---|
| btree | **+99.27%** | 0.00% | 0.00% |
| gist | **+89.47%** | 0.00% | 0.00% |
| gin | **+82.72%** | **+82.72%** | **+82.72%** |
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

**GIN: never recovers, at any `VACUUM` count.** GIN uses the same `RecentGlobalXmin` gate ([ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L786)), but that is not why it stays large. `ginVacuumEntryPage()` never removes an entry tuple whose posting list becomes empty; when `nitems` reaches 0 it sets `plist = NULL, plistsize = 0` and re-adds the tuple to the page ([ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556)). The entry pages are therefore neither new nor deleted, `GinPageIsRecyclable` is false for all of them, and they never reach the FSM.

A `pageinspect` probe on an isolated GIN fixture measures this directly. After deleting all 200,000 rows and running `VACUUM` three times, `gin_metapage_info` still reported **600,000 entries across 4,101 entry pages** and the FSM fork was still 0 bytes. The reload then doubled the file from 4,102 to 7,495 blocks. For GIN this cycle is genuine, permanent bloat that only a rebuild clears.

**SP-GiST, hash, BRIN and bloom: no xid gate to wait for.** SP-GiST records a free page whenever it is new or empty, with no transaction test ([spgvacuum.c#spgvacuum-fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L650-L670)), and contrib `bloom` does the same for new or deleted pages ([blvacuum.c#blvacuumcleanup](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L189-L214)). Hash does not use the index FSM at all: `_hash_freeovflpage()` clears the page's bit in the index's own bitmap ([hashovfl.c:632](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L632)) and `_hash_getovflpage()` finds and re-marks that bit on the next allocation ([hashovfl.c#_hash_getovflpage-setbit](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L313-L329)), so reuse is immediate and unconditional. BRIN never had per-row entries to remove.

#### What the cycle changes about the recommendation

Nothing in the per-access-method verdict in [Thresholds, and what to do with a flagged index](#thresholds-and-what-to-do-with-a-flagged-index) moves, but two operational points sharpen:

1. **For B-tree and GiST the cheap fix is a second `VACUUM`, but only before the reload.** Adding one `VACUUM` pass between the delete and the reload prevented 100% of the growth at a fraction of a rebuild's cost. Once the reload has already happened the file is committed: no index access method truncates its main fork outside a rebuild ([storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L229-L295), [spgvacuum.c#truncate-disabled](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886)), so a later `VACUUM` only makes those 547 pages reusable by the *next* load and leaves the current 1,098 blocks on disk. At that point `REINDEX INDEX CONCURRENTLY` is the only thing that shrinks the file. For GIN a rebuild is the only option at any point in the cycle.
2. **Re-capture the baseline after a bulk delete-and-reload, not just after a rebuild.** [Capture discipline](#capture-discipline) already lists `REINDEX`, `VACUUM FULL`, `CLUSTER`, partition attach and row-width changes. This cycle adds a case where the *table* looks untouched — same row count, same heap size, same `pg_class` shape — while three of seven indexes carry 83-99% more blocks than their baseline describes.

### Use the main fork, not pg_table_size

`pg_table_size()` sums every fork plus the TOAST relation and its index ([dbsize.c#calculate_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L380-L408), [dbsize.c#calculate_toast_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L338-L378)). TOAST traffic is invisible to an index on a narrow key, so it destroys the signal. Moving 20% of rows to TOAST while the B-tree doubled:

| denominator | drift, btree | drift, gin | drift, spgist |
|---|---|---|---|
| `pg_relation_size` (main fork) | 1.746 | 1.408 | 1.030 |
| `pg_table_size` (all forks + TOAST) | 0.095 | 0.099 | 0.067 |

True index bloat in that fixture was 0.499 for B-tree. The main-fork denominator flagged it; the `pg_table_size` denominator reported a 10x improvement. Use `pg_relation_size` on both sides.

One caveat that made this fixture harder to build than expected: a first attempt used `repeat('z', 3000)`, which pglz compressed to a 44-byte value that never left the heap. Wide-but-compressible columns do not produce TOAST traffic.

### The detection query

Read-only. Verified against the pinned 12.2 build.

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';

SELECT /* wiki_index_bloat_ratio_drift */
       n.nspname AS schema_name,
       tc.relname AS table_name,
       ic.relname AS index_name,
       am.amname AS index_am,
       b.baseline_ratio,
       round(pg_relation_size(i.indexrelid)::numeric
             / nullif(pg_relation_size(i.indrelid), 0), 6) AS current_ratio,
       round((pg_relation_size(i.indexrelid)::numeric
              / nullif(pg_relation_size(i.indrelid), 0))
             / b.baseline_ratio, 3) AS drift,
       pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
       pg_size_pretty(pg_relation_size(i.indrelid)) AS heap_main_size,
       s.n_live_tup,
       s.n_dead_tup
FROM pg_index i
JOIN pg_class ic ON ic.oid = i.indexrelid
JOIN pg_class tc ON tc.oid = i.indrelid
JOIN pg_namespace n ON n.oid = ic.relnamespace
JOIN pg_am am ON am.oid = ic.relam
LEFT JOIN pg_stat_all_tables s ON s.relid = i.indrelid
CROSS JOIN LATERAL (
    SELECT CASE WHEN d.description ~ '^[0-9]*\.?[0-9]+$'
                THEN d.description::numeric END AS baseline_ratio
    FROM pg_description d
    WHERE d.objoid = i.indexrelid
      AND d.classoid = 'pg_class'::regclass
      AND d.objsubid = 0
) b
WHERE ic.relkind = 'i'
  AND b.baseline_ratio IS NOT NULL
  AND b.baseline_ratio > 0
  AND pg_relation_size(i.indrelid) > 0
ORDER BY drift DESC NULLS LAST;
COMMIT;
```

Details that matter:

- The `CASE` guard is not decoration. A bare `description::numeric` fails the whole query with `invalid input syntax for type numeric` the moment one index carries a human comment, and PostgreSQL does not promise that a `WHERE` filter runs before a select-list cast; the documentation names `CASE` as the supported way to force evaluation order ([syntax.sgml#expression-evaluation](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L2500-L2527)).
- `relkind = 'i'` excludes partitioned indexes, which are `'I'` and have no storage ([pg_class.h#relkind](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L163), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L185-L192)).
- `n_live_tup` / `n_dead_tup` come from `pg_stat_all_tables` ([system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)) and are the heap-side sanity check that a bare ratio cannot provide. They require `track_counts`, which is `PGC_SUSET`, so it is session/transaction scope for a superuser and reload scope from the configuration file; it defaults to on ([guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1399)).
- `statement_timeout` and `lock_timeout` are both `PGC_USERSET`, so `SET LOCAL` here is transaction scope with no restart or reload ([guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)).
- `pg_index.indexrelid` and `pg_index.indrelid` are the index and table OIDs ([pg_index.h#pg_index](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L35)).

### The baseline audit query

The comment is a shared, human-visible slot, so a monitoring scheme built on it needs a second query that reports which indexes have no usable baseline. Read-only, verified against the pinned build:

```sql
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '5s';

SELECT /* wiki_index_baseline_audit */
       n.nspname AS schema_name,
       ic.relname AS index_name,
       am.amname AS index_am,
       CASE WHEN d.description IS NULL THEN 'no baseline stored'
            WHEN d.description !~ '^[0-9]*\.?[0-9]+$' THEN 'comment is not a number'
            ELSE 'baseline is zero'
       END AS problem,
       left(coalesce(d.description, ''), 40) AS comment_prefix
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
  AND (d.description IS NULL
       OR d.description !~ '^[0-9]*\.?[0-9]+$'
       OR d.description::numeric = 0)
ORDER BY 1, 2;
COMMIT;
```

On the test server this correctly reported an index whose comment had been replaced by a human with `rebuilt by dba on 2026-08-01`, and the two partition child indexes whose baselines had been overwritten by the hazard described below.

### Capture discipline

Since the payload is a bare number with no metadata, every guarantee has to live in the capture procedure. Writes are described here rather than filed as SQL:

- Issue `COMMENT ON INDEX <index> IS '<ratio>'` only in the same maintenance window as the rebuild, after both the index rebuild and a `VACUUM` of the table, so that both sides of the ratio are fresh. A baseline captured on an already-bloated index encodes the bloat, and drift will never fire.
- Round to six decimal places. A bare number is all the format allows, so the regex guard in the read query is the only validation the scheme has.
- Re-capture after anything that resets either side: `REINDEX`, `VACUUM FULL`, `CLUSTER`, a partition attach, or a bulk load that changes row width. The comment survives all of these, which means a stale baseline silently persists.
- Never capture when `pg_stat_all_tables.n_dead_tup` is a large fraction of `n_live_tup` for the table; that bakes heap bloat into the denominator.
- Refuse to overwrite a non-numeric comment without a human decision. The audit query above finds them.

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
| `DROP INDEX` | — | — | `pg_description` row deleted |
| `pg_dump` | — | — | emits `COMMENT ON INDEX cm.i_k IS '0.267216';` |

The partitioned-table row is a genuine data-loss hazard, not a test artifact. A parent index commented `parent-baseline` with children commented `0.111111` and `0.222222` ended up with **both children carrying `parent-baseline`** after one `ALTER TABLE ... ALTER COLUMN TYPE`. Upstream knows: the regression suite covers this case and labels its own expected output `-- Note: these tests currently show the wrong behavior for comments :-(` ([alter_table.sql#partitioned-comments](../../../../raw/postgres-12/src/test/regress/sql/alter_table.sql#L1421-L1440), [alter_table.out#partitioned-comments](../../../../raw/postgres-12/src/test/regress/expected/alter_table.out#L2103-L2134)). On partitioned tables, one column type change silently replaces every child baseline.

`DROP INDEX` removing the comment is expected behavior, documented as comments being dropped with their object ([comment.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L88-L101)).

### Locking, permissions, and visibility

`CommentObject()` resolves the target through `get_object_address()` with `ShareUpdateExclusiveLock` and then demands ownership ([comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L66-L78)). For an index, that ownership check is `pg_class_ownercheck` ([objectaddress.c#check_object_ownership](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2266-L2289)), and the accepted relkinds are `RELKIND_INDEX` and `RELKIND_PARTITIONED_INDEX` ([objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1246-L1262)). The grammar production is `COMMENT ON comment_type_any_name any_name IS comment_text` ([gram.y#CommentStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6395-L6403)).

Measured consequences on the pinned build:

- Inside an explicit transaction, `COMMENT ON INDEX` held exactly one relation lock: `ShareUpdateExclusiveLock` **on the index**, with no lock row for the table.
- A non-owner got `ERROR: must be owner of index i_k`.
- The same non-owner read the baseline back successfully. Comments have no read privilege gate at all, which the documentation states explicitly and warns about ([comment.sgml#notes](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L274-L286)).

From the conflict table ([lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L60-L105)), that index-level `ShareUpdateExclusiveLock` conflicts with itself and with `AccessExclusiveLock`, so a capture serializes against `REINDEX` and against another capture on the same index, but it does not conflict with the `RowExclusiveLock` that lazy VACUUM takes on indexes ([vacuumlazy.c:283-284](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L284)) or the `AccessShareLock` ANALYZE takes ([analyze.c:419-421](../../../../raw/postgres-12/src/backend/commands/analyze.c#L419-L421)).

Two operational consequences: the baseline is world-readable, so it must never encode anything sensitive; and `psql`'s `\di+` prints it in the Description column, where a human will eventually mistake it for a comment and replace it.

### Cost of the survey, and the relpages variant

With 300 indexes in one database on the pinned build, the size-based detection query ran in 5.3-7.1 ms. Each `pg_relation_size` call `stat()`s every segment file of the relation, so cost is linear in relations, not in data volume.

A catalog-only variant that reads `pg_class.relpages` instead of calling `pg_relation_size` ran in 3.7-3.8 ms and matched the live file size for all 300 freshly analyzed indexes. It is not equivalent, though: `relpages` only moves when `vac_update_relstats()` writes it during VACUUM or ANALYZE ([vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1156-L1197)). After inserting 100,000 rows without analyzing, `relpages` still said 8 blocks against a live 281 — a 35x understatement. Use `pg_relation_size` for detection; the `relpages` variant is only safe on tables you know were just analyzed.

### Partitioned indexes

A partitioned index is relkind `'I'` and has no storage: `pg_relation_size()` returned 0 for the parent while the leaf index returned 8192 on the same test. Commenting the parent is allowed, but a parent-level ratio is meaningless. Combined with the `ALTER COLUMN TYPE` overwrite hazard above, the practical rule is to store baselines only on leaf indexes, and to re-capture every leaf baseline after any partitioned-table column type change.

### Thresholds, and what to do with a flagged index

From the measurements, per access method:

| AM | usable? | suggested drift alert | why |
|---|---|---|---|
| btree | yes | 1.30 | normal 4x growth reached only 1.058; a 20% key-churn pass reached 1.662 |
| spgist | yes | 1.30 | 1.004 under normal growth |
| bloom | yes | 1.30 | 0.997 under normal growth |
| gist | with care | 1.35 | fresh ratio itself moved 3.1% over 16x growth |
| hash | no | — | healthy sawtooth spans 1.50x |
| gin | no | — | healthy ratio fell 36% over 16x growth |
| brin | no | — | index size is nearly constant; ratio decays with table size |

A flagged index means "the index grew faster than its heap since the baseline". Before acting, confirm with a second signal, because drift alone cannot distinguish the three failure modes: check `n_dead_tup` against `n_live_tup` for heap bloat, check whether the table recently shrank, and for B-tree only, confirm with `pgstatindex` if contrib is available. The remediation is `REINDEX INDEX CONCURRENTLY`, after which the baseline must be re-captured.

The scheme's blind spot cannot be fixed by tuning the threshold: a mass delete plus VACUUM leaves drift at exactly 1.000 with up to 80% real bloat, so this must not be the only bloat monitor in an installation.

### Why this exists: the contrib boundary

The reason a home-grown ratio is attractive at all is that v12 has no core function that reports per-index bloat, and the contrib tools do not cover every access method. `pgstattuple` handles heaps and B-tree, hash and GiST indexes, and raises `"%s" (%s) is not supported` for GIN, SP-GiST, BRIN, unknown index AMs, and partitioned indexes ([pgstattuple.c#pgstat_relation](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L240-L313)). Installing it needs superuser: neither `pgstattuple.control` nor `pageinspect.control` sets `superuser = false`, and the control-file parser defaults that field to true ([extension.c#read_extension_control_file](../../../../raw/postgres-12/src/backend/commands/extension.c#L605-L625)), enforced at script execution ([extension.c#execute_extension_script](../../../../raw/postgres-12/src/backend/commands/extension.c#L798-L810)).

`pg_relation_size` has neither restriction: it works identically for every access method, including any AM registered through `CREATE ACCESS METHOD`, and needs no extension. That, and not accuracy, is the real argument for this design.

One access-method-specific size confounder worth naming: GIN keeps a pending list whose size is bounded by `gin_pending_list_limit`, which is `PGC_USERSET` (session/transaction scope, also settable as a per-index storage parameter) and defaults to 4 MB ([guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184)). A GIN index's file size therefore moves with pending-list state independent of any bloat.

### Tests in the pinned checkout

- Comment survival across `REINDEX TABLE` and `REINDEX TABLE CONCURRENTLY` is directly covered ([create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2116)).
- The partitioned-index comment overwrite is covered, and the expected output records it as known-wrong behavior ([alter_table.sql#partitioned-comments](../../../../raw/postgres-12/src/test/regress/sql/alter_table.sql#L1421-L1440), [alter_table.out#partitioned-comments](../../../../raw/postgres-12/src/test/regress/expected/alter_table.out#L2103-L2134)).
- The closest upstream analogue of the delete-and-reload cycle is the B-tree multilevel-page-deletion case: insert 80,000 rows, add a primary key, delete all but 10, `VACUUM`, then insert 1,000 more, with a comment stating it "also creates coverage for nbtree FSM page recycling" ([btree_index.sql#multilevel-page-deletion](../../../../raw/postgres-12/src/test/regress/sql/btree_index.sql#L144-L162), [btree_index.out#multilevel-page-deletion](../../../../raw/postgres-12/src/test/regress/expected/btree_index.out#L315-L332)). It exists for WAL-record and page-deletion coverage and asserts nothing about index size, so it does not contradict or confirm the growth measured above.
- There is **no** upstream test that compares index size to heap size, none that exercises a comment as machine-readable data, and none that asserts an index's block count after a delete-and-reload cycle. Everything in the measurement tables above was produced on a purpose-built 12.2 server, not by the regression suite.

## Context Reviewed

- `COMMENT` path end to end: grammar production, `CommentObject`, `CreateComments`, `DeleteComments`, `GetComment`, the `pg_description` catalog definition, and the `obj_description` builtin.
- Object resolution and privileges for indexes: `get_object_address`, `get_relation_by_qualified_name`, `check_object_ownership`, and the lock conflict table.
- Comment lifecycle across DDL: `index_concurrently_swap`, `RelationTruncateIndexes`, `pg_dump`'s index comment emission, and the `create_index` and `alter_table` regression suites.
- Size measurement: `pg_relation_size` one- and two-argument forms, `calculate_relation_size`, `calculate_table_size`, `calculate_toast_table_size`, `pg_table_size`, `pg_indexes_size`, and `vac_update_relstats` for the `relpages` alternative.
- Per-access-method storage behavior: default fill factors for heap, B-tree, hash, GiST and SP-GiST; `_hash_init` and `_hash_expandtable` bucket allocation; BRIN pages-per-range and revmap capacity; the free-space-map reclaim paths in B-tree, GiST, GIN and SP-GiST vacuum; the disabled SP-GiST truncation block; and every `RelationTruncate`/`smgrtruncate` call site in the backend.
- Free-page recycling per access method: the shared index FSM entry points `GetFreeIndexPage`/`RecordFreeIndexPage`; the `RecentGlobalXmin` recyclability gates in `_bt_page_recyclable`, `gistPageRecyclable` and `GinPageIsRecyclable` and their vacuum-side callers; `_bt_getbuf`'s FSM allocation loop; `ginVacuumEntryPage`'s retention of zero-posting entry tuples and `ginvacuumcleanup`'s page census; the ungated SP-GiST and contrib `bloom` cleanup paths; hash's bitmap-based overflow-page reuse in `_hash_freeovflpage`/`_hash_getovflpage`; and the BRIN `brinbulkdelete` no-op.
- Adjacent observability: `pg_stat_all_tables`, `track_counts`, `statement_timeout`, `lock_timeout`, `gin_pending_list_limit`, and the `pgstattuple` access-method support matrix with its extension privilege gate.
- Exact-pin measurements on an isolated PostgreSQL 12.2 server built from the pinned checkout: fresh-ratio scale invariance at three scales for seven access methods; a hash/B-tree sawtooth sweep at seven row counts; six workload scenarios across seven access methods with `REINDEX` and `VACUUM FULL` ground truth; a COMMENT durability matrix across nine operations; lock, ownership and read-visibility probes; and survey timings over 300 indexes.
- A second exact-pin run for the delete-and-reload cycle, on an `autovacuum = off` 12.2 server built from the pinned checkout: one table carrying all seven access methods at 200,000 rows, run at one, two and three post-`DELETE` `VACUUM` passes; an independent replicate of the one-`VACUUM` arm; `pgstatindex` snapshots of the B-tree at build, mid-cycle and reload; `hash_metapage_info` bucket and overflow-page comparison between the initial and rebuilt hash indexes; a `gin_metapage_info` entry and entry-page census after three `VACUUM` passes; the mid-cycle zero-byte-heap behavior of the filed detection query; and a standalone re-run of the published script on a fresh database to confirm it reproduces every number in the results table.

## Evidence Map

| Claim | Evidence |
|---|---|
| `COMMENT ON INDEX` takes `ShareUpdateExclusiveLock` on the index and requires ownership | [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L66-L78), [objectaddress.c#check_object_ownership](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2266-L2289), [objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1246-L1262), [gram.y#CommentStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6395-L6403) |
| One comment per index, stored as a single replaceable `pg_description` row with `objsubid = 0`; empty string deletes it | [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L48-L57), [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225), [comment.c:154-156](../../../../raw/postgres-12/src/backend/commands/comment.c#L154-L156) |
| The baseline is readable by any user in the database and has no privilege gate | [comment.sgml#notes](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L274-L286), [pg_proc.dat#obj_description](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2311-L2314) |
| The comment follows the index through `REINDEX CONCURRENTLY`, which changes the index OID | [index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656), [create_index.sql#comments-preserved](../../../../raw/postgres-12/src/test/regress/sql/create_index.sql#L845-L854), [create_index.out#comments-preserved](../../../../raw/postgres-12/src/test/regress/expected/create_index.out#L2093-L2116) |
| `ALTER TABLE ... ALTER COLUMN TYPE` on a partitioned table overwrites child index comments with the parent's | [alter_table.sql#partitioned-comments](../../../../raw/postgres-12/src/test/regress/sql/alter_table.sql#L1421-L1440), [alter_table.out#partitioned-comments](../../../../raw/postgres-12/src/test/regress/expected/alter_table.out#L2103-L2134) |
| Baselines survive dump/restore | [pg_dump.c#dumpIndex-comment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16456-L16463) |
| `pg_relation_size(regclass)` measures the main fork by `stat()`ing every segment, identically for every access method | [pg_proc.dat#pg_relation_size](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L6883-L6891), [dbsize.c#calculate_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L266-L308), [dbsize.c#pg_relation_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L310-L336) |
| `pg_table_size` adds every fork plus TOAST and its index, which destroys the ratio | [dbsize.c#calculate_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L380-L408), [dbsize.c#calculate_toast_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L338-L378), [dbsize.c#pg_table_size](../../../../raw/postgres-12/src/backend/utils/adt/dbsize.c#L450-L467) |
| No index access method truncates its main fork; the heap can | [storage.c#RelationTruncate](../../../../raw/postgres-12/src/backend/catalog/storage.c#L229-L295), [vacuumlazy.c#lazy_truncate_heap](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L1955-L1970), [heap.c#RelationTruncateIndexes](../../../../raw/postgres-12/src/backend/catalog/heap.c#L3163-L3207), [spgvacuum.c#truncate-disabled](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L863-L886) |
| Index VACUUM recycles pages through the free space map instead of shrinking the file | [nbtree.c#btvacuumscan-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1078-L1095), [gistvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L255-L265), [ginvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L790), [spgvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L848-L861) |
| B-tree and GiST gate free-page reuse on `RecentGlobalXmin`, so the VACUUM that deletes a page cannot recycle it in the same pass | [nbtpage.c#_bt_page_recyclable](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L940-L963), [gistutil.c#gistPageRecyclable](../../../../raw/postgres-12/src/backend/access/gist/gistutil.c#L881-L906), [nbtree.c#btvacuumpage-recycle](../../../../raw/postgres-12/src/backend/access/nbtree/nbtree.c#L1166-L1179), [gistvacuum.c#gistvacuum-recycle](../../../../raw/postgres-12/src/backend/access/gist/gistvacuum.c#L298-L320) |
| B-tree free-page reuse is key-order agnostic: `_bt_getbuf` takes whatever block the FSM reports with at least half a page free | [nbtpage.c#_bt_getbuf-fsm](../../../../raw/postgres-12/src/backend/access/nbtree/nbtpage.c#L801-L842), [indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L33-L46), [indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-12/src/backend/storage/freespace/indexfsm.c#L49-L55) |
| GIN keeps entry tuples whose posting list becomes empty, so emptied entry pages are never recyclable and never reach the FSM | [ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556), [ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138), [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L786) |
| SP-GiST and contrib `bloom` record free pages with no transaction gate, so their space returns on the first VACUUM | [spgvacuum.c#spgvacuum-fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L650-L670), [blvacuum.c#blvacuumcleanup](../../../../raw/postgres-12/contrib/bloom/blvacuum.c#L189-L214) |
| Hash does not use the index FSM at all; it frees and reuses overflow pages through its own bitmap pages | [hashovfl.c:632](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L632), [hashovfl.c#_hash_getovflpage-setbit](../../../../raw/postgres-12/src/backend/access/hash/hashovfl.c#L313-L329) |
| BRIN removes nothing on delete, so its size does not respond to a delete-and-reload cycle | [brin.c#brinbulkdelete](../../../../raw/postgres-12/src/backend/access/brin/brin.c#L766-L784) |
| A fresh B-tree is packed to fillfactor 90 and the heap to 100, so a first key-update pass splits nearly every leaf | [nbtree.h:169](../../../../raw/postgres-12/src/include/access/nbtree.h#L169), [rel.h:279](../../../../raw/postgres-12/src/include/utils/rel.h#L279) |
| Hash index size is a step function of estimated tuples, allocated a whole splitpoint at a time at fillfactor 75 | [hashpage.c#_hash_init-buckets](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L505-L525), [hashpage.c#_hash_expandtable-splitpoint](../../../../raw/postgres-12/src/backend/access/hash/hashpage.c#L785-L853), [hash.h:279](../../../../raw/postgres-12/src/include/access/hash.h#L279) |
| BRIN size is governed by pages-per-range and revmap capacity, not row count | [brin.h#BRIN_DEFAULT_PAGES_PER_RANGE](../../../../raw/postgres-12/src/include/access/brin.h#L39-L43), [brin_page.h#REVMAP_PAGE_MAXITEMS](../../../../raw/postgres-12/src/include/access/brin_page.h#L88-L94) |
| GIN file size also moves with pending-list state | [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184) |
| Default fill factors differ per access method, so fresh ratios differ per access method | [nbtree.h:169](../../../../raw/postgres-12/src/include/access/nbtree.h#L169), [hash.h:279](../../../../raw/postgres-12/src/include/access/hash.h#L279), [spgist.h:25](../../../../raw/postgres-12/src/include/access/spgist.h#L25), [gist_private.h:465](../../../../raw/postgres-12/src/include/access/gist_private.h#L465) |
| A `CASE` guard is the documented way to force evaluation order around the numeric cast | [syntax.sgml#expression-evaluation](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L2500-L2527) |
| Partitioned indexes are relkind `'I'` and have no storage | [pg_class.h#relkind](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L163), [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L185-L192) |
| `relpages` only advances when VACUUM or ANALYZE writes it | [vacuum.c#vac_update_relstats](../../../../raw/postgres-12/src/backend/commands/vacuum.c#L1156-L1197) |
| The heap-side sanity columns exist in core and depend on `track_counts` | [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581), [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1399) |
| Capture conflicts with `REINDEX` but not with VACUUM's or ANALYZE's index locks | [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L60-L105), [vacuumlazy.c:283-284](../../../../raw/postgres-12/src/backend/access/heap/vacuumlazy.c#L283-L284), [analyze.c:419-421](../../../../raw/postgres-12/src/backend/commands/analyze.c#L419-L421) |
| `pgstattuple` covers only heap, B-tree, hash and GiST, and needs superuser to install | [pgstattuple.c#pgstat_relation](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L240-L313), [extension.c#read_extension_control_file](../../../../raw/postgres-12/src/backend/commands/extension.c#L605-L625), [extension.c#execute_extension_script](../../../../raw/postgres-12/src/backend/commands/extension.c#L798-L810) |
| Query timeouts used above are session/transaction scope | [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386), [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396) |

## Open Questions

- Every number in the measurement tables comes from one purpose-built 12.2 server with synthetic fixtures (200k-1.6M rows, one 40-byte payload column, one index per table). The per-access-method drift thresholds are calibrated to those fixtures, not to any production workload. Different key widths, key distributions, fill factors, or multi-index tables will move them.
- The GiST and GIN cases where `REINDEX` produced a *larger* index than incremental insertion were observed but not traced to the responsible build path in source; they are reported as measurements only.
- Whether the partitioned-child comment overwrite has a source-level fix in a later minor release of the 12 branch was not checked, and cannot be checked against another major version under this wiki's evidence rules.
- The scheme was not tested against `CREATE ACCESS METHOD` custom index AMs, only against the six core access methods plus contrib `bloom`. `pg_relation_size` is AM-agnostic, but nothing was measured about whether a custom AM's fresh ratio is scale-invariant.
- The 300-index survey timings were taken on a warm, idle, single-user server with all relation files in page cache. Cold-cache `stat()` cost on an installation with tens of thousands of relations was not measured.
- The delete-and-reload cycle ran on a single-user server with `autovacuum = off` and no concurrent transactions, which is the friendliest possible setting for the `RecentGlobalXmin` gate. A long-running concurrent snapshot would hold that horizon back and delay recycling past any number of `VACUUM` passes; that case was not measured. On a real server autovacuum also decides the `VACUUM` count for you, so the one-, two- and three-pass arms bracket the behavior rather than predict it.
- The cycle used strictly ascending keys in both loads, as chosen for this test. Reusing the first load's key values, or using random keys, will move the SP-GiST and hash numbers and may move GIN's, and was not measured. The B-tree and GiST results should not change, because their recovery depends on the FSM rather than on key order, but that was inferred from `_bt_getbuf`/`GetFreeIndexPage` rather than measured against a second key policy.
- Immediately after the back-to-back `DELETE`; `VACUUM` pair, `pg_stat_all_tables.n_dead_tup` read 200,000 for a heap of 0 bytes; repeating the same two statements two seconds apart gave 0. The reading is therefore a statistics-collector timing artifact rather than a property of the cycle, but the flush ordering that produces it was not traced in source. It matters only because the filed detection query offers `n_dead_tup` as its heap-side sanity column.
- Three of the cycle's supporting measurements — the B-tree `pgstatindex` page census, the hash `hash_metapage_info` bucket and overflow comparison, and the GIN `gin_metapage_info` entry census — come from `pgstattuple` and `pageinspect`, which are exactly the contrib modules [Why this exists: the contrib boundary](#why-this-exists-the-contrib-boundary) rules out for the monitoring scheme itself. They are diagnostics for this page, not part of the proposal.

## Source References

- [comment.c#CommentObject](../../../../raw/postgres-12/src/backend/commands/comment.c#L39-L130)
- [comment.c#CreateComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L141-L225)
- [comment.c#DeleteComments](../../../../raw/postgres-12/src/backend/commands/comment.c#L317-L367)
- [gram.y#CommentStmt](../../../../raw/postgres-12/src/backend/parser/gram.y#L6395-L6403)
- [pg_description.h#pg_description](../../../../raw/postgres-12/src/include/catalog/pg_description.h#L48-L57)
- [pg_proc.dat#obj_description](../../../../raw/postgres-12/src/include/catalog/pg_proc.dat#L2311-L2314)
- [objectaddress.c#get_relation_by_qualified_name](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L1246-L1262)
- [objectaddress.c#check_object_ownership](../../../../raw/postgres-12/src/backend/catalog/objectaddress.c#L2266-L2289)
- [index.c#index_concurrently_swap-comment](../../../../raw/postgres-12/src/backend/catalog/index.c#L1612-L1656)
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
- [ginvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L755-L790)
- [ginvacuum.c#ginVacuumEntryPage-empty](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L505-L556)
- [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-12/src/backend/access/gin/ginvacuum.c#L744-L786)
- [ginblock.h#GinPageIsRecyclable](../../../../raw/postgres-12/src/include/access/ginblock.h#L131-L138)
- [spgvacuum.c#fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L848-L861)
- [spgvacuum.c#spgvacuum-fsm](../../../../raw/postgres-12/src/backend/access/spgist/spgvacuum.c#L650-L670)
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
- [pg_index.h#pg_index](../../../../raw/postgres-12/src/include/catalog/pg_index.h#L29-L35)
- [pg_class.h#relkind](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L152-L163)
- [pg_class.h#RELKIND_HAS_STORAGE](../../../../raw/postgres-12/src/include/catalog/pg_class.h#L185-L192)
- [lock.c#LockConflicts](../../../../raw/postgres-12/src/backend/storage/lmgr/lock.c#L60-L105)
- [system_views.sql#pg_stat_all_tables](../../../../raw/postgres-12/src/backend/catalog/system_views.sql#L552-L581)
- [guc.c#track_counts](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L1392-L1399)
- [guc.c#statement_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2377-L2386)
- [guc.c#lock_timeout](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L2388-L2396)
- [guc.c#gin_pending_list_limit](../../../../raw/postgres-12/src/backend/utils/misc/guc.c#L3175-L3184)
- [pg_dump.c#dumpIndex-comment](../../../../raw/postgres-12/src/bin/pg_dump/pg_dump.c#L16456-L16463)
- [pgstattuple.c#pgstat_relation](../../../../raw/postgres-12/contrib/pgstattuple/pgstattuple.c#L240-L313)
- [extension.c#read_extension_control_file](../../../../raw/postgres-12/src/backend/commands/extension.c#L605-L625)
- [extension.c#execute_extension_script](../../../../raw/postgres-12/src/backend/commands/extension.c#L798-L810)
- [comment.sgml#description](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L88-L101)
- [comment.sgml#notes](../../../../raw/postgres-12/doc/src/sgml/ref/comment.sgml#L274-L286)
- [syntax.sgml#expression-evaluation](../../../../raw/postgres-12/doc/src/sgml/syntax.sgml#L2500-L2527)
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
