---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [The statement](#the-statement)
  - [How to read the output](#how-to-read-the-output)
  - [What one pgstatindex call actually measures](#what-one-pgstatindex-call-actually-measures)
  - [Why every candidate filter is there](#why-every-candidate-filter-is-there)
  - [The one behavioural difference between 12 and 17](#the-one-behavioural-difference-between-12-and-17)
  - [The model, from avg_leaf_density to a rebuilt size](#the-model-from-avg_leaf_density-to-a-rebuilt-size)
  - [Why the two page-layout constants are safe](#why-the-two-page-layout-constants-are-safe)
  - [wasted_space is not est_reclaimable](#wasted_space-is-not-est_reclaimable)
  - [NaN is the trap](#nan-is-the-trap)
  - [Accuracy against REINDEX INDEX](#accuracy-against-reindex-index)
  - [The three shapes it gets wrong](#the-three-shapes-it-gets-wrong)
  - [Deduplication changes the input, not the arithmetic](#deduplication-changes-the-input-not-the-arithmetic)
  - [What it costs to run](#what-it-costs-to-run)
  - [Privileges](#privileges)
  - [Locking, timeouts, and the concurrent-drop race](#locking-timeouts-and-the-concurrent-drop-race)
  - [Everything the two servers agreed on](#everything-the-two-servers-agreed-on)
  - [How this was measured](#how-this-was-measured)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In PostgreSQL 17: provide SQL that uses `pgstatindex` only, for B-tree indexes,
to report bloat and wasted space. Make sure that it works on v12 and v17.

Prompt note: the request was filed as `in postgresql 17 , question:  provide sql
that using pgstatindex only for btree indexes to report bloat, wasted space,
make sure that it works on v12 and v17`. It had lowercase `postgresql` and `sql`,
a space before a comma, a double space, `that using` for `that uses`, `btree` for
`B-tree`, and two comma splices; the asker approved the corrected restatement
above. Four scoping answers are recorded with it: `pgstatindex` is the only
measurement function, while `pg_class`, `pg_index`, `pg_am`, `pg_namespace` and
`pg_relation_size` may be read to choose, size and label indexes; the deliverable
is one statement that runs unchanged on both majors; the statement was executed on
isolated 12.2 and 17.11 servers and scored against `REINDEX INDEX`; and it is filed
as a new page rather than a follow-up on
[Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17](btree-index-bloat-core-sql-only.md),
which is deliberately the no-contrib method.

## Answer

### The statement

One statement, eight stages, no measurement function other than `pgstatindex`. It
returned 27 rows on the 12.2 server and 28 on the 17.11 server, from the same text.

```sql
-- B-tree bloat and wasted space from pgstatindex alone.
-- Runs unchanged on PostgreSQL 12 and 17.
--
--   params    tunables and the two page-layout constants
--   cand      every index pgstatindex can be called on without raising
--   measured  one pgstatindex() call per candidate
--   modelled  per-index constants: leaf capacity, fillfactor, target free space
--   sized     bytes actually holding index entries, and bytes that do not
--   est       leaf pages a rebuild at this index's fillfactor would need
--   final     the modelled rebuilt size
--
-- wasted_space measures the file against perfect packing, so a healthy index
-- reports roughly its fillfactor's worth of waste.  est_reclaimable measures it
-- against a rebuild at its own fillfactor, which is what REINDEX gives back.
-- Alert on est_reclaimable_pct; read wasted_pct for composition only.

SET statement_timeout = '15min';
SET lock_timeout = '5s';

WITH params AS (
    SELECT current_setting('block_size')::bigint AS bs,
           24::bigint      AS page_header,     -- SizeOfPageHeaderData
           16::bigint      AS btree_special,   -- MAXALIGN(sizeof(BTPageOpaqueData))
           1048576::bigint AS min_index_bytes, -- skip anything smaller
           20::numeric     AS alert_pct        -- rebuild-candidate threshold
),
cand AS MATERIALIZED (
    SELECT c.oid       AS idx_oid,
           n.nspname   AS schema_name,
           c.relname   AS index_name,
           t.relname   AS table_name,
           (SELECT o.option_value::int
              FROM pg_options_to_table(c.reloptions) o
             WHERE o.option_name = 'fillfactor') AS fillfactor_opt
      FROM pg_class c
      JOIN pg_namespace n ON n.oid = c.relnamespace
      JOIN pg_index x     ON x.indexrelid = c.oid
      JOIN pg_class t     ON t.oid = x.indrelid
      JOIN pg_am a        ON a.oid = c.relam
     CROSS JOIN params p
     WHERE a.amname = 'btree'                        -- pgstatindex takes no other AM
       AND c.relkind = 'i'                           -- not 'I', a partitioned index has no storage
       AND x.indisvalid AND x.indisready AND x.indislive
       AND NOT pg_is_other_temp_schema(c.relnamespace)
       AND (c.relpersistence <> 'u' OR NOT pg_is_in_recovery())
       AND pg_relation_size(c.oid) >= p.min_index_bytes
),
measured AS (
    SELECT c.*, s.*
      FROM cand c, LATERAL pgstatindex(c.idx_oid::regclass) s
),
modelled AS (
    SELECT m.*,
           p.bs,
           p.alert_pct,
           p.bs - p.page_header - p.btree_special AS leaf_capacity,
           COALESCE(m.fillfactor_opt, 90)         AS fillfactor,
           -- what a build leaves free on a leaf page: BLCKSZ * (100 - fillfactor) / 100
           (p.bs * (100 - COALESCE(m.fillfactor_opt, 90))) / 100 AS target_free,
           m.empty_pages + m.deleted_pages        AS dead_pages,
           -- an index with no leaf pages reports NaN; NaN sorts above every
           -- number, so it must never reach a comparison
           CASE WHEN m.leaf_pages > 0 AND m.avg_leaf_density <> 'NaN'::float8
                THEN (m.avg_leaf_density / 100)::numeric
                ELSE 0::numeric END               AS density
      FROM measured m CROSS JOIN params p
),
sized AS (
    SELECT d.*,
           d.leaf_pages * d.leaf_capacity                       AS leaf_bytes,
           round(d.leaf_pages * d.leaf_capacity * d.density)    AS live_leaf_bytes,
           d.dead_pages * d.bs                                  AS dead_bytes,
           (d.leaf_capacity - d.target_free)::numeric / d.leaf_capacity AS target_density
      FROM modelled d
),
est AS (
    SELECT s.*,
           s.leaf_bytes - s.live_leaf_bytes + s.dead_bytes AS wasted_space,
           CASE WHEN s.leaf_pages = 0 THEN 0
                ELSE ceil(s.leaf_pages * s.density / s.target_density) END
               AS est_leaf_pages
      FROM sized s
),
final AS (
    SELECT e.*,
           (1 + e.est_leaf_pages
              + CASE WHEN e.leaf_pages = 0 THEN 0
                     ELSE round(e.internal_pages * e.est_leaf_pages / e.leaf_pages) END
           ) * e.bs AS est_rebuilt_bytes
      FROM est e
)
SELECT /* wiki_btree_bloat_pgstatindex_12_17 */
       f.schema_name,
       f.index_name,
       f.table_name,
       pg_size_pretty(f.index_size) AS index_size,
       f.leaf_pages,
       f.dead_pages,
       CASE WHEN f.leaf_pages > 0 THEN round(f.avg_leaf_density::numeric, 2) END
           AS avg_leaf_density,
       CASE WHEN f.leaf_pages > 0 THEN round(f.leaf_fragmentation::numeric, 2) END
           AS leaf_fragmentation,
       pg_size_pretty(f.wasted_space::bigint) AS wasted_space,
       round(100 * f.wasted_space / f.index_size, 1) AS wasted_pct,
       pg_size_pretty(f.est_rebuilt_bytes::bigint) AS est_rebuilt_size,
       pg_size_pretty((f.index_size - f.est_rebuilt_bytes)::bigint) AS est_reclaimable,
       round(100 * (f.index_size - f.est_rebuilt_bytes) / f.index_size, 1)
           AS est_reclaimable_pct,
       CASE WHEN 100 * (f.index_size - f.est_rebuilt_bytes) / f.index_size >= f.alert_pct
            THEN 'rebuild candidate' ELSE 'ok' END AS status,
       array_to_string(array_remove(ARRAY[
           CASE WHEN f.leaf_pages = 0 THEN 'no leaf pages' END,
           CASE WHEN f.version < 4 THEN 'metapage version ' || f.version END,
           CASE WHEN f.fillfactor <> 90 THEN 'fillfactor ' || f.fillfactor END,
           CASE WHEN f.leaf_pages > 0 AND f.leaf_fragmentation >= 30
                THEN 'fragmented, not wasted space' END,
           CASE WHEN 100 * (f.index_size - f.est_rebuilt_bytes) / f.index_size <= -1
                THEN 'denser than a rebuild would leave it' END,
           CASE WHEN f.dead_pages > 0
                 AND f.dead_bytes >= (f.index_size - f.est_rebuilt_bytes) / 2
                 AND f.index_size > f.est_rebuilt_bytes
                THEN 'reclaim is mostly empty/deleted pages' END
       ], NULL), '; ') AS notes
  FROM final f
 ORDER BY f.index_size - f.est_rebuilt_bytes DESC;
```

It needs `CREATE EXTENSION pgstattuple` in the database being examined. Both
checkouts ship the same `pgstattuple` 1.5 control file and the same install
script, so the function signature is the same on both
([pgstattuple.control:1-5](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5),
[pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)).

`statement_timeout` and `lock_timeout` are both `PGC_USERSET`
([guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620),
[guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)),
so they apply at session/transaction scope and need no reload or restart. The
statement timeout is minutes, not seconds, because this report reads every page
of every index it reports on; see [What it costs to run](#what-it-costs-to-run).

### How to read the output

Alert on `est_reclaimable_pct`. It is the modelled answer to "how much smaller
would `REINDEX INDEX` make this file", and on the fixture suite it landed within
one point of the truth for 91 of 94 indexes on 12.2 and 89 of 93 on 17.11.

Read the other columns as supporting detail:

| Column | Means | Watch for |
|---|---|---|
| `wasted_space`, `wasted_pct` | Free bytes inside live leaf pages, plus every empty and deleted page, measured against perfect packing | A healthy index reports roughly `100 - fillfactor`. A fresh `fillfactor = 10` index reads 89.6% here and −0.5% reclaimable |
| `avg_leaf_density` | Share of leaf-page space holding entries | Low density is the usual bloat signal, but it is blind to whole pages that hold nothing |
| `dead_pages` | `empty_pages + deleted_pages` | These are 100% waste and invisible to `avg_leaf_density`. A measured index read 89.94% density and was still 69.9% reclaimable |
| `leaf_fragmentation` | Share of leaves whose right sibling sits at a lower block number | Not wasted space at all. Physical disorder that costs sequential-scan I/O; the note says so |
| `status` | `est_reclaimable_pct >= 20` | Tune `alert_pct` in `params` |
| `notes` | Why a row looks odd | `no leaf pages`, `fillfactor N`, `fragmented, not wasted space`, `denser than a rebuild would leave it`, `reclaim is mostly empty/deleted pages` |

Two rows from the 17.11 run show why both percentages exist:

```text
 index_name  | index_size | leaf_pages | dead_pages | avg_leaf_density | wasted_pct | est_reclaimable_pct | notes
 i_delhead   | 21 MB      |        821 |       1918 |            89.94 |       72.9 |                69.9 | reclaim is mostly empty/deleted pages
 i_ff10      | 206 MB     |      26316 |          0 |             9.62 |       89.6 |                -0.5 | fillfactor 10
```

`i_delhead` has textbook-perfect leaves and is two thirds reclaimable. `i_ff10`
looks catastrophic by density and is exactly the size its owner asked for.

### What one pgstatindex call actually measures

`pgstatindex` opens the index with `AccessShareLock`, reads the metapage, then
walks every remaining block under a shared buffer lock with a `BAS_BULKREAD`
strategy, and buckets each page into deleted, half-dead (`empty_pages`), leaf, or
internal
([pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L215-L331),
[nbtree.h#P_ISLEAF](../../../../raw/postgres-17/src/include/access/nbtree.h#L212-L227)).
Three details drive the model:

- **`index_size` is the whole file**, computed as `(1 + leaf + internal + deleted
  + empty) * BLCKSZ`
  ([pgstatindex.c#index_size](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L349-L357)).
  Measured: it equalled `pg_relation_size()` for 218 of 218 candidate indexes on
  17.11 and 212 of 212 on 12.2.
- **`avg_leaf_density` covers live leaves only.** It is
  `100 - free_space / max_avail * 100`, where both sums are accumulated only in
  the leaf branch of the loop; deleted and half-dead pages contribute to neither
  ([pgstatindex.c#leaf-accounting](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L304-L324),
  [pgstatindex.c#avg_leaf_density](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L367)).
  That is why the statement adds `dead_pages * bs` separately.
- **`leaf_fragmentation` counts pages, not bytes**: leaves whose `btpo_next` is a
  lower block number, over `leaf_pages`
  ([pgstatindex.c#fragments](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L318-L323),
  [pgstatindex.c#leaf_fragmentation](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L368-L372)).

Nothing here reads `pg_class.reltuples`, `pg_statistic` or the cumulative
statistics views, so the report does not care whether the table was ever analyzed.
That is the main thing it buys over a catalog-only estimator, whose accuracy rests
on statistics freshness and on the `reltuples = -1` sentinel; both hazards are
measured in
[Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17](btree-index-bloat-core-sql-only.md).

The reason `REINDEX` is the remedy and `VACUUM` is not: the nbtree code contains
no call to `RelationTruncate` or `smgrtruncate` (0 matches under
`src/backend/access/nbtree/`). A vacuum puts deleted pages into the free space map
for reuse instead
([nbtree.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1170),
[nbtree.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059)).
Measured on both servers: after deleting 70% of a table and vacuuming, its index
still occupied 22,487,040 bytes with 1,918 dead pages, and `REINDEX` took it to
6,758,400.

### Why every candidate filter is there

Every filter in `cand` exists because `pgstatindex` raises on that shape, and one
raised call aborts the whole statement. Each was reproduced on both servers:

| Filter | What happens without it |
|---|---|
| `a.amname = 'btree'` | `ERROR: relation "s_hash" is not a btree index` — reproduced for hash, GIN, GiST, SP-GiST and BRIN ([pgstatindex.c#IS_BTREE](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228)) |
| `c.relkind = 'i'` | A partitioned index is `'I'` and fails the same `IS_INDEX` test: `ERROR: relation "i_part" is not a btree index` ([pg_class.h#RELKIND_PARTITIONED_INDEX](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L165-L173), [pgstattuple.out#partitioned](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L155-L171)) |
| `indisvalid AND indisready AND indislive` | On 17.11, `ERROR: index "i_invalid" is not valid`. On 12.2 the same index returns a row. See the next section ([pgstatindex.c#indisvalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L240-L250), [pg_index.h#indisvalid](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)) |
| `NOT pg_is_other_temp_schema(...)` | `ERROR: cannot access temporary tables of other sessions`. Measured: one other session holding a 4.4 MB temp index moved the candidate count from 27 to 28 on 17.11 and 26 to 27 on 12.2, and the statement still returned every row with that session open ([pgstatindex.c#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L230-L238), [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L652-L669)) |
| `relpersistence <> 'u' OR NOT pg_is_in_recovery()` | Untested belt and braces. The planner refuses unlogged relations during recovery ([plancat.c#recovery](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L149-L153)) but a function call never goes through that path, and `pgstatindex_impl` has no such guard, so on a standby it would read whatever is in the file. On a primary, unlogged indexes are read normally and one is in the fixture output |
| `pg_relation_size(c.oid) >= min_index_bytes` | Not a correctness filter, a cost filter. It is the only place a non-`pgstatindex` measurement appears, and it is a `stat()`-level answer, not a page read. Replace it with `c.relpages` for a strictly-catalog prefilter, at the price of trusting a stale number |

The index is passed by OID, not by name (`pgstatindex(c.idx_oid::regclass)`). That
is deliberate: see [Privileges](#privileges).

`cand` is `AS MATERIALIZED` — v12 syntax, and the earliest major this statement
claims — so the candidate list is fixed before the first page is read.

### The one behavioural difference between 12 and 17

**An invalid index is the only shape where the two majors disagree**, and it is
the reason the `indisvalid` filter is not optional.

On the 17.11 server, `pgstatindex` on an index with `indisvalid = false` fails:

```text
ERROR:  index "i_invalid" is not valid
```

On the 12.2 server, the identical call on the identical fixture returns a row:

```text
 version | tree_level | index_size | root_block_no | internal_pages | leaf_pages | empty_pages | deleted_pages | avg_leaf_density | leaf_fragmentation
       4 |          2 |    6758400 |           290 |              4 |        820 |           0 |             0 |            90.05 |                  0
```

The check is `13503eb5905`, "Diagnose !indisvalid in more SQL functions"
(2023-10-30), whose earliest containing release tag in this checkout is
`REL_17_0`; it also states its own reasoning, that a `!indisready` index could
lead to `ERRCODE_DATA_CORRUPTED` and that an `indisready && !indisvalid` index
gives confusing results because its size can be too low for a valid index of the
table
([pgstatindex.c#indisvalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L240-L250)).
Filtering the index out is right on both majors: on 17 it prevents an abort, and
on 12 it prevents a half-built `CREATE INDEX CONCURRENTLY` leftover from being
reported as a healthy index. Failed concurrent builds are exactly how such indexes
appear; see
[How CREATE INDEX CONCURRENTLY Is Implemented in PostgreSQL 17](create-index-concurrently.md).

Everything else matched. Both servers produced the same message text for
non-B-tree access methods, partitioned indexes, tables, views, sequences,
another session's temp index, and a stale OID
(`ERROR: could not open relation with OID 2147483647`).

### The model, from avg_leaf_density to a rebuilt size

The estimate is a ratio, so most constants cancel:

```text
payload_leaf_pages  = leaf_pages * avg_leaf_density / 100
est_leaf_pages      = ceil(payload_leaf_pages / target_density)
est_internal_pages  = round(internal_pages * est_leaf_pages / leaf_pages)
est_rebuilt_bytes   = (1 + est_leaf_pages + est_internal_pages) * block_size
est_reclaimable     = index_size - est_rebuilt_bytes
```

`target_density` is not the fillfactor. It is what the build code actually leaves
behind. A sorted build closes a leaf page when the remaining free space drops
below `BLCKSZ * (100 - fillfactor) / 100`
([nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671),
[nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145),
[nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L845-L860)),
so the density a rebuild reaches is `(leaf_capacity - target_free) / leaf_capacity`
and not `fillfactor / 100`. At `block_size` 8192 and the default fillfactor of 90
([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202),
[reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194))
that is `(8152 - 819) / 8152 = 89.95%`, marginally below the fillfactor, which
makes the estimate conservative by construction.

Measured fresh-build densities, byte-identical on both servers over a 200,000-row
`int` index, against what the formula predicts:

| fillfactor | modelled target density | measured `avg_leaf_density` | reported `est_reclaimable_pct` |
|---|---|---|---|
| 100 | 100.00 | 99.82 | 0.0 |
| 90 | 89.95 | 90.00 | −0.2 |
| 50 | 49.75 | 49.81 | −0.2 |
| 10 | 9.57 | 9.62 | −0.5 |

`est_internal_pages` uses `round`, not `ceil`, because it is a proportional
estimate rather than a capacity bound; internal pages were under 0.5% of every
fixture. Non-leaf pages are built to a fixed 70%
([nbtree.h#BTREE_NONLEAF_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)),
which the model does not use.

### Why the two page-layout constants are safe

`leaf_capacity` is `block_size - 24 - 16`, which reproduces `pgstatindex`'s own
`max_avail`, computed as `pd_special - SizeOfPageHeaderData`
([pgstatindex.c#max_avail](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L314)).
The 24 is `SizeOfPageHeaderData`
([bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L214));
the 16 is `MAXALIGN(sizeof(BTPageOpaqueData))`, whose four fields are two
`BlockNumber`, one `uint32` and two `uint16`
([nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71)).
The same subtraction appears in core as
`BLCKSZ - SizeOfPageHeaderData - sizeof(BTPageOpaqueData)`
([nbtree.h#MaxTIDsPerBTreePage](../../../../raw/postgres-17/src/include/access/nbtree.h#L185-L187)).

Rather than trust the arithmetic, both servers were asked to imply the constant
from pages with known contents. One `int4` key on a root leaf page leaves
`8160 - 28 - 4 = 8128` free, and an empty leaf page leaves `8176 - 24 - 4 = 8148`;
the 4 is the line pointer that `PageGetFreeSpace` deducts
([bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)).
Both servers reported `0.29` and `0.05`, implying `max_avail` of 8151.6 and
8152.1 against the statement's 8152.

The statement reads `block_size` from the server, so a cluster built at another
`BLCKSZ` scales; the 24 and the 16 do not depend on `BLCKSZ`. Nothing was run at a
block size other than 8192.

### wasted_space is not est_reclaimable

```text
wasted_space = leaf_pages * leaf_capacity * (1 - avg_leaf_density/100)
             + (empty_pages + deleted_pages) * block_size
```

That is the file measured against 100% packing, which no B-tree ever reaches on
purpose. It answers "how many bytes hold nothing", and it is the literal reading
of "wasted space". It is not an action threshold: a perfectly healthy default
index reports about 10% wasted, and the `fillfactor = 10` fixture reports 89.6%.
`est_reclaimable` is the action threshold. Both are in the output because they
answer different questions, and the header comment in the statement says which is
which.

### NaN is the trap

An index with no leaf pages returns `NaN` for both `avg_leaf_density` and
`leaf_fragmentation`, because the C code guards on `max_avail > 0` and
`leaf_pages > 0`
([pgstatindex.c#NaN](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372)).
Upstream's own expected output records this for an empty index
([pgstattuple.out#empty-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)).

In PostgreSQL, `NaN` compares greater than every number. Measured on both servers:

```text
 is_nan | nan_over_threshold | numeric_nan_over_threshold
 t      | t                  | t
```

So a naive `WHERE bloat_pct > 20` reports every empty index as maximally bloated.
The statement converts density to `0` behind a `leaf_pages > 0` test before any
arithmetic, prints `NULL` instead of `NaN`, and tags the row `no leaf pages`. In
the scoring run 62 of 94 indexes on 12.2 and 59 of 93 on 17.11 had `NaN` density —
mostly empty catalog TOAST indexes — and every one of them reported exactly `0.0`
estimated reclaim against `0.0` actual.

### Accuracy against REINDEX INDEX

Ground truth is `pg_relation_size` before and after `REINDEX INDEX`, run over
every scored index on each server. The estimator was scored from a view generated
mechanically from this page's statement text, with only two edits, both printed by
the generator: the two `SET` lines dropped, and `min_index_bytes` set to 0 so
sub-megabyte fixtures are scored too.

| Index | 12.2 est / actual | 17.11 est / actual | density | dead pages |
|---|---|---|---|---|
| `i_del90` 90% deleted, vacuumed | 89.7 / 89.9 | 89.7 / 89.9 | 9.27 | 0 |
| `i_partial` partial, 90% of matches deleted | 89.6 / 89.8 | 89.6 / 89.8 | 9.27 | 0 |
| `i_uniq` unique, 7 of 8 deleted | 87.2 / 87.4 | 87.2 / 87.4 | 11.52 | 0 |
| `t_part_1_id_idx` partition leaf index | 85.3 / 85.6 | 85.3 / 85.6 | 13.11 | 0 |
| `t_part_2_id_idx` partition leaf index | 85.3 / 85.6 | 85.3 / 85.6 | 13.11 | 0 |
| `i_unlogged` unlogged table | 83.0 / 83.3 | 83.0 / 83.3 | 15.25 | 0 |
| `i_expr` expression key | 79.4 / 79.9 | 79.4 / 79.9 | 18.55 | 0 |
| `i_text_del` `text` key | 79.4 / 79.9 | 79.4 / 79.9 | 18.55 | 0 |
| `i_incl` `INCLUDE` column | 74.8 / 75.0 | 74.8 / 75.0 | 22.62 | 0 |
| `i_wide` 400-byte keys | 69.9 / 75.0 | 69.9 / 75.0 | 27.23 | 44 |
| `i_delhead` contiguous head deleted | 69.9 / 69.9 | 69.9 / 69.9 | 89.94 | 1918 |
| `i_multi` three-column key | 66.4 / 66.6 | 66.4 / 66.6 | 30.22 | 0 |
| `t_toast_pkey` TOAST table primary key | 64.9 / 63.2 | 64.9 / 63.2 | 30.02 | 0 |
| `i_del50` half deleted | 49.7 / 49.9 | 49.7 / 49.9 | 45.18 | 0 |
| `i_churn` update churn on the key | 46.2 / 46.4 | 46.2 / 46.9 | 48.36 | 0 |
| `i_frag` reverse-order inserts | 44.0 / 44.3 | 44.0 / 44.3 | 50.34 | 0 |
| `i_ff100` fresh, fillfactor 100 | 0.1 / 0.0 | 0.1 / 0.0 | 99.86 | 0 |
| `i_text` fresh `text` | 0.0 / 0.0 | 0.0 / 0.0 | 89.98 | 0 |
| `i_fresh` fresh, default fillfactor | −0.1 / 0.0 | −0.1 / 0.0 | 90.06 | 0 |
| `i_ff50` fresh, fillfactor 50 | −0.2 / 0.0 | −0.2 / 0.0 | 49.85 | 0 |
| `i_ff10` fresh, fillfactor 10 | −0.5 / 0.0 | −0.5 / 0.0 | 9.62 | 0 |
| `t_empty_pkey` empty table | 0.0 / 0.0 | 0.0 / 0.0 | NaN | 0 |
| `i_dup` 10 distinct values, built by `CREATE INDEX` | −0.3 / 0.0 | 0.0 / 0.0 | 89.91 | 0 |
| `i_dup_ins` same data, built by inserts | −6.6 / −6.4 | −6.7 / −6.8 | 95.94 | 0 |
| `i_novac` 90% deleted, **not** vacuumed | −0.1 / 89.9 | −0.1 / 89.9 | 90.06 | 0 |
| `i_dedup_off` duplicates, `deduplicate_items = off` | not constructible | −0.3 / 69.1 | 90.16 | 0 |

Totals over every scored index, including the 60-odd empty catalog TOAST indexes
not listed above: **94 indexes on 12.2, 91 within 1.0 point and 92 within 2.0**;
**93 indexes on 17.11, 89 within 1.0 point and 90 within 2.0**. The largest
over-estimate on both servers is the same `+1.7` points, on a 456 kB TOAST
primary key; every other row over-estimates by at most `+0.1`. Over-estimates are
the dangerous direction, because they promise space a rebuild will not return,
and on this suite they are bounded by two points on indexes small enough that the
per-page rounding dominates.

### The three shapes it gets wrong

All three are under-estimates: the index is more reclaimable than the report says.

1. **Entries deleted but not yet vacuumed** (`i_novac`, −90.0 points on both
   servers). Deleting 90% of the rows changes nothing on the index pages until a
   vacuum removes the entries, so the density is a healthy 90.06 while a rebuild
   takes the file from 22,487,040 to 2,260,992 bytes. `pgstatindex` counts bytes,
   not liveness, and cannot see this. A vacuum first, then this report, is the
   only fix — which is also the right operational order.
2. **Deduplication that the current index is not using** (`i_dedup_off`, −69.4
   points, 17.11 only). See the next section.
3. **Wide keys** (`i_wide`, −5.1 points on both). With 400-byte tuples a page is
   closed when the free space falls below 819 bytes, so on average it ends up
   several hundred bytes fuller than the model's bound; the rebuilt index measured
   92.77% density against the modelled 89.95%. The error is bounded by the tuple
   size over the leaf capacity, and it is always in the safe direction.

### Deduplication changes the input, not the arithmetic

B-tree deduplication arrived after 12, and it is the largest source of
same-statement, different-numbers between the two servers. Two fixtures with a
key of ten distinct values over a million rows:

| Fixture | 12.2 size | 17.11 size | 12.2 est / actual | 17.11 est / actual |
|---|---|---|---|---|
| `i_dup`, built by `CREATE INDEX` | 21 MB | 6800 kB | −0.3 / 0.0 | 0.0 / 0.0 |
| `i_dup_ins`, built empty then filled | 20 MB | 6368 kB | −6.6 / −6.4 | −6.7 / −6.8 |

The estimator is right on both servers. What changed is the index: on 17.11 the
duplicates are already posting lists, so the payload `pgstatindex` measures is
already the compressed payload, and a rebuild reproduces it.

The failure case is an index whose pages are **not** deduplicated but whose
rebuild would be. `i_dedup_off` builds one deliberately with
`deduplicate_items = off`
([nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1151)),
then turns the option back on: the report reads 90.16% density and `−0.3%`, and
`REINDEX` takes the file from 22,519,808 to 6,963,200 bytes, 69.1% reclaimed. The
reloption does not exist on 12.2, so the fixture is unconstructible there.

The realistic version of this is a cluster upgraded from 12: those indexes cannot
deduplicate until they are rebuilt, and no `pgstatindex` column distinguishes them
(they report `version` 4 like everything else). That case, and the core-SQL test
for it, is
[Checking Whether an Index Needs a Rebuild to Enable Deduplication After pg_upgrade From PostgreSQL 12 to 17](btree-deduplication-after-pg-upgrade.md).

### What it costs to run

This report reads every page of every index it reports on. That is the price of
not depending on statistics. `EXPLAIN (ANALYZE, BUFFERS)` attributes it precisely,
because the reads go through the buffer manager:

| Server | Indexes | Buffers at the `pgstatindex` function scan | Execution |
|---|---|---|---|
| 12.2 | 27 | 6,693 hit + 99,658 read (~831 MB) | 124.2 ms |
| 17.11 | 28 | 5,616 hit + 99,797 read (~824 MB) | 132.6 ms |

End to end through psql, warm: 153.1 ms then 135.2 ms on 12.2, 147.5 ms then
142.2 ms on 17.11. These are millisecond numbers only because the fixture database
is under a gigabyte and the pages were in the OS cache; the cost scales with the
bytes of index, not the number of indexes.

Two mitigations are built in. `min_index_bytes` skips small indexes using a
`stat()`-level size, before any page is read. And the scan uses a `BAS_BULKREAD`
strategy, a 256 kB ring
([freelist.c#GetAccessStrategy](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L540-L574)),
so a large report does not evict the buffer pool; the `written=832`/`written=874`
counters in the same plans are the ring writing back its own dirty buffers.

### Privileges

`pgstattuple` 1.5 revokes `EXECUTE` from `PUBLIC` and grants it to
`pg_stat_scan_tables`
([pgstattuple--1.4--1.5.sql#grants](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92),
[pgstattuple.sgml#access](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L15-L24)).
The `_v1_5` entry points carry no `superuser()` check; only the pre-1.5 symbols do
([pgstatindex.c#pgstatindex_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L162-L180),
[pgstatindex.c#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L144-L160)).

Measured identically on both servers with a login role `mon` whose only privilege
is membership in `pg_stat_scan_tables`:

- `mon` runs the whole statement and gets every row, including indexes on tables
  in a schema it has no `USAGE` on.
- `pgstatindex('bl.i_del90')` by **name** fails for `mon` with
  `ERROR: permission denied for schema bl`, because resolving the name needs
  schema `USAGE`. This is why the statement passes `c.idx_oid::regclass`: the OID
  overload never resolves a name.
- A role without the membership gets
  `ERROR: permission denied for function pgstatindex`.

So the grant needed is exactly `GRANT pg_stat_scan_tables TO <role>`, and it
carries page-level visibility into every table in the database. No `SELECT`
privilege on the indexed tables is involved at any point.

### Locking, timeouts, and the concurrent-drop race

`pgstatindex` takes `AccessShareLock` on the index
([pgstatindex.c#pgstatindexbyid_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L204-L213)),
so it waits behind anything holding `AccessExclusiveLock`. With another session
sitting on an uncommitted `DROP INDEX`, `lock_timeout = '2s'` cancelled the call
at 2000.9 ms on 17.11 and 2001.0 ms on 12.2:

```text
ERROR:  canceling statement due to lock timeout
```

The unfixable hole is a drop that commits while the statement is running. The
candidate list is a snapshot of the catalog; `relation_open` is not. An index
dropped three seconds into a running report killed the whole statement on both
servers:

```text
ERROR:  could not open relation with OID 16897
```

Nothing in a single statement can survive that: one failed index loses the whole
report. On a database with heavy DDL, run the report against a candidate list that
excludes tables under migration, or drive `pgstatindex` from a loop that catches
the error per index and keeps going.

### Everything the two servers agreed on

Of the 27 rows the 12.2 report produced and the 28 from 17.11, **24 are identical
character for character across the two servers**, including `leaf_pages`,
`avg_leaf_density`, `wasted_space`, `est_reclaimable` and `notes`. The exceptions
are the two duplicate-key indexes (deduplication) and `i_churn`, where the churn
fixture is not deterministic (2,551 leaves against 2,550, and 48.38% against
48.36%). Both servers also produced the same error text for every rejected shape,
the same `+1.7` worst over-estimate, the same `−90.0` and `−5.1` under-estimates,
the same implied leaf capacity, and the same fresh-build densities at four
fillfactors.

### How this was measured

Two isolated servers on one host, each initialised for this run:

- **12.2**, `server_version_num` 120002, built from this repo's v12 pin
  `45b88269a353ad93744772791feb6d01bc7e1e42`, with `contrib/pgstattuple` compiled
  from the same checkout through PGXS into a copy of the source tree, never into
  `raw/`.
- **17.11**, `server_version_num` 170011, from the `--with-icu --enable-debug`
  install of this page's pin `786db8dcf168bd9df8f55047337525ac19118b1c`.

Both at `block_size` 8192, `autovacuum = off`, `fsync = off`, one shared fixture
script of 24 index shapes plus one 17-only deduplication fixture, `REINDEX INDEX`
as ground truth, and `pgstatindex` as the only measurement tool — no
`pageinspect`, no `pgstattuple()`, no `amcheck`. The sandbox is retained at
`.wiki-runtime/tmp/pgsi/`.

## Context Reviewed

- `contrib/pgstattuple/pgstatindex.c` in both pinned checkouts, function by
  function: the four `pgstatindex` entry points, `pgstatindex_impl`, and the
  `pgstatginindex`/`pgstathashindex` neighbours that share its guards.
- `contrib/pgstattuple/pgstattuple--1.4.sql`, `pgstattuple--1.4--1.5.sql` and
  `pgstattuple.control` in both checkouts; the three files are byte-identical
  between them, which is why one statement can target both.
- `contrib/pgstattuple/sql/pgstattuple.sql` and `expected/pgstattuple.out`, for
  the upstream expectations on empty indexes, wrong access methods, partitioned
  indexes, views, foreign tables and sequences.
- `doc/src/sgml/pgstattuple.sgml`, for the documented column meanings and the
  `pg_stat_scan_tables` access rule.
- `src/backend/access/nbtree/nbtsort.c` and `src/include/access/nbtree.h`, for
  what a rebuild targets: `_bt_pagestate`, `_bt_buildadd`, `BTGetFillFactor`,
  `BTGetTargetPageFreeSpace`, the fillfactor constants, and the page-opaque
  layout.
- `src/backend/access/nbtree/nbtree.c`, for what a vacuum does with deleted pages,
  and the absence of any truncation call under `src/backend/access/nbtree/`.
- `src/include/storage/bufpage.h` and `src/backend/storage/page/bufpage.c`, for
  `SizeOfPageHeaderData` and the line-pointer deduction in `PageGetFreeSpace`.
- `src/include/catalog/pg_index.h`, `pg_class.h` and `src/include/utils/rel.h`,
  for the catalog columns and the temp-relation test the filters mirror.
- `src/backend/utils/misc/guc_tables.c` for the two timeout GUCs' contexts, and
  `src/backend/optimizer/util/plancat.c` for the unlogged-during-recovery rule
  that does not apply to function calls.
- The v17 checkout's own history for `13503eb5905` and its containing release
  tags.

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstatindex` accepts only a B-tree index relation | [pgstatindex.c#IS_BTREE](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L224-L228); measured errors for hash, GIN, GiST, SP-GiST, BRIN and a partitioned index on both servers |
| It refuses another session's temp index | [pgstatindex.c#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L230-L238), [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L652-L669); reproduced on both servers with a second session holding a 4.4 MB temp index |
| 17 refuses an invalid index, 12.2 returns a row | [pgstatindex.c#indisvalid](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L240-L250); commit `13503eb5905`, earliest containing tag `REL_17_0` in this checkout; measured both ways |
| `index_size` is the whole file | [pgstatindex.c#index_size](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L349-L357); equal to `pg_relation_size` for 218/218 and 212/212 candidates |
| `avg_leaf_density` ignores empty and deleted pages | [pgstatindex.c#leaf-accounting](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L304-L324); `i_delhead` at 89.94% density and 69.9% reclaimable, confirmed by `REINDEX` |
| No leaf pages gives `NaN`, and `NaN` outranks every threshold | [pgstatindex.c#NaN](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L363-L372), [pgstattuple.out#empty-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52); measured `NaN > 20` true for `float8` and `numeric` on both servers |
| A rebuild's leaf density is `(leaf_capacity - BLCKSZ*(100-ff)/100) / leaf_capacity` | [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671), [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L845-L860), [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145); four fillfactors measured within 0.18 points on both servers |
| Leaf capacity is `block_size - 24 - 16` | [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L214), [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71), [pgstatindex.c#max_avail](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L308-L314); implied 8151.6 and 8152.1 from two known-content pages on both servers |
| `PageGetFreeSpace` deducts one line pointer | [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923); an empty leaf page reads 0.05% density, not 0.00% |
| VACUUM never returns index pages to the filesystem | no `RelationTruncate`/`smgrtruncate` under `src/backend/access/nbtree/`, [nbtree.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1170); 1,918 dead pages survived VACUUM and disappeared on REINDEX |
| Access is `pg_stat_scan_tables`, and the OID form needs no schema `USAGE` | [pgstattuple--1.4--1.5.sql#grants](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92), [pgstattuple.sgml#access](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L15-L24); measured with two non-superuser roles on both servers |
| The call waits on `AccessExclusiveLock`; `lock_timeout` bounds the wait | [pgstatindex.c#pgstatindexbyid_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L204-L213), [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631); cancelled at 2000.9 ms and 2001.0 ms |
| A concurrent drop aborts the whole report | measured `could not open relation with OID 16897` (17.11) and `17173` (12.2) after 6.0 s |
| The scan uses a 256 kB bulk-read ring | [pgstatindex.c#BAS_BULKREAD](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L215-L222), [freelist.c#GetAccessStrategy](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L540-L574) |
| Deduplication is the only thing that made the two servers' numbers differ | [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1151); 24 of the report's rows identical, `i_dup`/`i_dup_ins` 21 MB against 6800 kB and 20 MB against 6368 kB |

## Open Questions

- **Nothing was run on a standby.** The `relpersistence <> 'u' OR NOT
  pg_is_in_recovery()` filter is reasoning from
  [plancat.c#recovery](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L149-L153)
  and from the absence of any recovery guard in `pgstatindex_impl`, not from a
  measurement. What `pgstatindex` actually returns for an unlogged index on a hot
  standby is untested, and so is the whole report's behaviour there.
- **Only two minor versions were tested**, 12.2 and 17.11, both from this repo's
  pins. The invalid-index check has no release tag before `REL_17_0` in the v17
  checkout, but whether some later 12.x minor changed anything relevant was not
  checked and cannot be checked from this page's evidence base.
- **One block size.** Every measurement is at `block_size` 8192. The statement
  reads `block_size` from the server, but the 24 and 16 constants and the whole
  target-density model are unverified at 4 kB, 16 kB or 32 kB.
- **The internal-page term is a proportional guess.** `round(internal_pages *
  est_leaf / leaf_pages)` was never tested against a case where the rebuilt tree
  loses a level; internal pages were under 0.5% of every fixture, so the suite
  cannot distinguish a good model from a lucky one.
- **The `+1.7` over-estimate is unexplained in detail.** It reproduced exactly on
  both servers on the same 456 kB TOAST primary key, which suggests per-page
  rounding rather than noise, but no per-page accounting was done to confirm it,
  and no fixture was built to find the worst case for small indexes.
- **`i_novac` and `i_dedup_off` have no in-statement warning.** Both return
  `status = ok` with an empty `notes` string on an index a rebuild would shrink by
  90% and 69%. Neither condition is visible in any `pgstatindex` column, so
  closing them would require a second tool and would break the "pgstatindex only"
  constraint; the page documents them instead.
- **The churn fixture is not deterministic**, so `i_churn` is the one row that
  cannot be used as a cross-version identity check.
- **No test of a partitioned table with hundreds of partitions**, where the report
  returns one row per leaf index and the aggregate reading a DBA wants is per
  parent. The statement has no roll-up.

## Source References

- [pgstatindex.c#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L144-L160)
- [pgstatindex.c#pgstatindex_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L162-L180)
- [pgstatindex.c#pgstatindexbyid_v1_5](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L204-L213)
- [pgstatindex.c#pgstatindex_impl](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L215-L381)
- [pgstattuple--1.4.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4.sql#L19-L30)
- [pgstattuple--1.4--1.5.sql#pgstatindex-regclass](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L77-L92)
- [pgstattuple.control:1-5](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5)
- [pgstattuple.sgml#access](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L15-L24)
- [pgstattuple.sgml#pgstatindex](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L161-L281)
- [pgstattuple.sql#pgstatindex](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37)
- [pgstattuple.out#empty-index](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L44-L52)
- [pgstattuple.out#wrong-relkinds](../../../../raw/postgres-17/contrib/pgstattuple/expected/pgstattuple.out#L140-L171)
- [nbtree.h#BTPageOpaqueData](../../../../raw/postgres-17/src/include/access/nbtree.h#L62-L71)
- [nbtree.h#MaxTIDsPerBTreePage](../../../../raw/postgres-17/src/include/access/nbtree.h#L185-L187)
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../../raw/postgres-17/src/include/access/nbtree.h#L189-L202)
- [nbtree.h#P_ISLEAF](../../../../raw/postgres-17/src/include/access/nbtree.h#L212-L227)
- [nbtree.h#BTGetTargetPageFreeSpace](../../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)
- [nbtree.h#BTGetDeduplicateItems](../../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1151)
- [nbtsort.c#_bt_pagestate](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L645-L671)
- [nbtsort.c#_bt_buildadd](../../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L845-L860)
- [nbtree.c#_bt_pendingfsm_finalize](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1048-L1059)
- [nbtree.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1170)
- [reloptions.c#btree-fillfactor](../../../../raw/postgres-17/src/backend/access/common/reloptions.c#L185-L194)
- [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L214)
- [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)
- [freelist.c#GetAccessStrategy](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L540-L574)
- [rel.h#RELATION_IS_OTHER_TEMP](../../../../raw/postgres-17/src/include/utils/rel.h#L652-L669)
- [pg_index.h#indisvalid](../../../../raw/postgres-17/src/include/catalog/pg_index.h#L42-L45)
- [pg_class.h#RELKIND_PARTITIONED_INDEX](../../../../raw/postgres-17/src/include/catalog/pg_class.h#L165-L173)
- [pg_proc.dat#pg_is_other_temp_schema](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L6448-L6450)
- [plancat.c#recovery](../../../../raw/postgres-17/src/backend/optimizer/util/plancat.c#L149-L153)
- [guc_tables.c#statement_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)
- [guc_tables.c#lock_timeout](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)

## Navigation

- [v17 index](../../index.md)
- [versions](../../../versions.md)
- [wiki index](../../../index.md)
- [v17: Testing the PostgreSQL 12 Core-SQL B-Tree Bloat Method on PostgreSQL 17](btree-index-bloat-core-sql-only.md)
- [v17: Checking Whether an Index Needs a Rebuild to Enable Deduplication After pg_upgrade](btree-deduplication-after-pg-upgrade.md)
- [v17: How CREATE INDEX CONCURRENTLY Is Implemented](create-index-concurrently.md)
- [v17: How REINDEX INDEX CONCURRENTLY Is Implemented](reindex-index-concurrently.md)
- [v17: Planner Penalties for Bloated Indexes](../query-planning/bloated-indexes-query-planner.md)
- [v12: How pgstatindex Calculates B-Tree Index Statistics](../../../v12/questions/indexing/how-pgstatindex-calculates-information.md)
- [v12: Leaf Density Versus Fragmentation for Index-Scan I/O](../../../v12/questions/indexing/leaf-density-vs-fragmentation-index-scan-io.md)
