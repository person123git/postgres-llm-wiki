---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: claude-opus-5-max 2026-08-26T14:01:59Z
---

# Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib Extensions on PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Deviations from the brief, and why](#deviations-from-the-brief-and-why)
  - [Why no single contrib function suffices](#why-no-single-contrib-function-suffices)
  - [The three waste classes, from GIN's own definitions](#the-three-waste-classes-from-gins-own-definitions)
  - [The procedure](#the-procedure)
  - [The census statement](#the-census-statement)
  - [Three cross-checks](#three-cross-checks)
  - [The fixtures](#the-fixtures)
  - [What the census meant against REINDEX](#what-the-census-meant-against-reindex)
  - [Whole-page waste is not a lower bound](#whole-page-waste-is-not-a-lower-bound)
  - [Entry-page slack is growth room, not waste](#entry-page-slack-is-growth-room-not-waste)
  - [The pending-list lifecycle, measured end to end](#the-pending-list-lifecycle-measured-end-to-end)
  - [Deleted pages need the horizon to move before they count](#deleted-pages-need-the-horizon-to-move-before-they-count)
  - [Concurrency: a census of a busy index is a mixed-instant reading](#concurrency-a-census-of-a-busy-index-is-a-mixed-instant-reading)
  - [Cost of the census](#cost-of-the-census)
  - [Privileges](#privileges)
  - [Refusals, silent answers, and other traps](#refusals-silent-answers-and-other-traps)
  - [Timeouts and GUC scope](#timeouts-and-guc-scope)
  - [Reading rules](#reading-rules)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

Follow AGENTS.md, in PostgreSQL 17.

Question: Design an accurate procedure for measuring wasted/reclaimable bytes in a
GIN index in PostgreSQL 17 using contrib extensions.

Deliverable: one type: question page under wiki/v17/questions/indexing/, with the
question restated verbatim, the answer inline, full front matter, Contents,
Context Reviewed, Evidence Map, Open Questions, Source References, and Navigation.

Implement and validate this plan:

1. Establish from source why no single contrib function suffices: pgstattuple()
   rejects GIN (pgstattuple.c pgstat_relation), and pgstatginindex() reads only the
   metapage's version / pending_pages / pending_tuples (pgstatindex.c).

2. Define the waste classes from GIN's own source definitions:
   - whole-page waste: GIN_DELETED pages plus all-zero (PageIsNew) pages, both
     recyclable per GinPageIsRecyclable and FSM-recorded as BLCKSZ-1 by
     RecordFreeIndexPage; reusable only within the index because ginvacuumcleanup
     never truncates;
   - intra-page slack on live pages: pd_upper - pd_lower, matching
     GinDataLeafPageGetFreeSpace = PageGetExactFreeSpace, gated on
     ginVersion = 2 because pre-9.4 posting-tree pd_lower is untrustworthy;
   - pending-list pages: deferred work, reported separately, never counted as
     waste (a flush merges entries and can grow the index).

3. The procedure: install pageinspect + pgstattuple + pg_freespacemap; VACUUM the
   table (or core gin_clean_pending_list()) to settle state and refresh metapage
   stats; record pg_relation_size(main) and gin_metapage_info(get_raw_page(idx, 0));
   scan blocks 1..relpages-1 reading each raw page once and feeding the same bytea
   to page_header() and gin_page_opaque_info() via LATERAL, classifying pages by
   the flags array (NULL row = new page); report whole_page_waste,
   live_page_slack_bytes, and pending_bytes against the size denominator; cross-
   check whole-page waste with pg_freespace() = 8191 counts and VACUUM VERBOSE
   pages_free, and page-type counts against gin_metapage_info's
   n_entry_pages / n_data_pages.

4. Validate on an isolated exact-pin 17.11 server (reuse
   .wiki-runtime/tmp/pstate/install/ if still present): build GIN fixtures with
   churn, deleted pages, a populated pending list, and an empty control; run the
   full procedure; use REINDEX INDEX before/after pg_relation_size as ground truth
   for filesystem-reclaimable bytes and report how the contrib estimate bounds it.

5. Document caveats with citations: superuser-only raw-page functions versus the
   pg_stat_scan_tables grant for pgstatginindex added in pgstattuple 1.5, invalid-index and other-
   session-temp refusals, non-atomic scan races, entry-page ItemIdData overhead
   making slack an upper bound, gin_leafpage_items' exact flag requirement, and
   the absence of a GIN verifier in v17 contrib amcheck.

6. Recommend session-scoped statement_timeout and lock_timeout on every
   production-bound statement and tag each with a /* wiki_... */ comment. Anything
   not verified on the live server goes under ## Open Questions. Leave
   verified: false; set verified_by_agent only after re-checking every claim.
   Update wiki/index.md, wiki/v17/index.md, wiki/versions.md, append to
   wiki/log.md, and run scripts/wiki_lint.

Prompt corrections, agreed before drafting: the original first line read
`Follow AGENTS.md. in PostgreSQL 17.`, whose stray period left `in PostgreSQL 17`
as a lowercase fragment; and item 5 read `the pg_stat_scan_tables grant for
pgstatginindex 1.5`, where 1.5 is the *pgstattuple extension* version that added
the `pgstatginindex(regclass)` grant, not a version of the function. Everything
else is verbatim, including the `pg_freespace() = 8191` premise in item 3, which
the answer corrects on evidence.

Review prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md, in PostgreSQL 17. Review the question: Measuring Wasted and
> Reclaimable Bytes in a GIN Index With Contrib Extensions on PostgreSQL 17
> (unverified).

The original read `follow agents.md , in postgresql 17 , review question : ...`:
`agents.md` for AGENTS.md, lowercase `postgresql`, and a space before each comma
and before the colon. The asker chose a full review — source re-verification plus
a rebuilt server and re-measurement — with corrections made in place, and asked
that the stale sandbox pointer be corrected and the sandbox rebuilt.

What the review did: re-read every source citation on this page against the pinned
checkout (119 of them at review time; 121 now, the review having added two),
rebuilt PostgreSQL 17.11 from `raw/postgres-17/` because the original sandbox had
been deleted, republished the fixtures as SQL so the numbers are reproducible,
re-ran every measurement, and added a physical standby to test the one refusal the
first run could not reach. See [The fixtures](#the-fixtures) for what reproduced
and what did not.

## Answer

### Short answer

Build a **page census** from `pageinspect`, and treat "wasted" as three separate
numbers, never one:

| Class | How it is measured | What it means |
|---|---|---|
| Whole-page waste | count of pages whose `gin_page_opaque_info` flags contain `deleted`, plus pages where that function returns a NULL row (all-zero pages), times `block_size` | dead pages inside the file; reusable by this index only, never returned to the filesystem without a rebuild |
| Live-page slack | `page_header.upper - page_header.lower` summed over live pages, split into entry-tree and posting-tree (data) pages | free bytes inside live pages; on entry pages this is mostly *growth room*, not waste |
| Pending-list bytes | count of pages flagged `list`, times `block_size` | deferred insert work, not waste; flushing it frees pages inside the file and can grow it |

Measured on an isolated 17.11 server with `REINDEX INDEX` as ground truth over
seven fixtures, `whole_page_waste + live_page_slack` was an **upper bound** on the
bytes a rebuild returned in 7 of 7 cases, and `whole_page_waste` alone was **not**
a lower bound: one fixture had 64.64% of its file in dead pages while `REINDEX`
returned only 58.05%, because a fresh GIN build is less dense than an aged one.
`pgstatginindex` reported `0 / 0` for that same index.

### Deviations from the brief, and why

Three items in the plan did not survive contact with the source or the server.

1. **`pg_freespace()` cannot return 8191.** `RecordFreeIndexPage` does record
   `BLCKSZ - 1` ([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)),
   but the FSM stores one byte per page: `fsm_space_avail_to_cat` maps anything at
   or above `MaxFSMRequestSize` to category 255
   ([freespace.c#fsm_space_avail_to_cat](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L398-L421)),
   and `GetRecordedFreeSpace` converts 255 back to exactly `MaxFSMRequestSize`
   ([freespace.c#GetRecordedFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L249-L271),
   [freespace.c#fsm_space_cat_to_avail](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L423-L435)).
   `MaxFSMRequestSize` is `MaxHeapTupleSize`
   ([freespace.c:66](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L66)), which is
   `BLCKSZ - MAXALIGN(SizeOfPageHeaderData + sizeof(ItemIdData))`
   ([htup_details.h#MaxHeapTupleSize](../../../../raw/postgres-17/src/include/access/htup_details.h#L563)) —
   **8160** on the pinned build with `block_size` 8192 and 8-byte MAXALIGN. The
   server agrees: the FSM census of the deleted-pages fixture reads
   `avail = 8160 -> 768 blocks, avail = 0 -> 130 blocks`, and the maximum `avail`
   observed over every GIN index was 8160. The cross-check therefore tests
   `avail = 8160`, not 8191. Do not take the number from the category table's own
   comment, which is illustrative and stale: it works its example "assuming
   default 8k `BLCKSZ`, and that `MaxFSMRequestSize` is 8164 bytes"
   ([freespace.c:36-62](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L36-L62)),
   a value the macro cannot produce on this build. Derive it from
   `MaxHeapTupleSize`, or read it off `pg_freespace` as above.

2. **`VACUUM VERBOSE` has no field literally called `pages_free`.** v17 prints
   `index "%s": pages: %u in total, %u newly deleted, %u currently deleted, %u reusable`,
   and the fourth number is `IndexBulkDeleteResult.pages_free`
   ([vacuumlazy.c:718-731](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731)).
   For GIN, "reusable" is the whole-index census set by `ginvacuumcleanup`
   ([ginvacuum.c:786-794](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L786-L794)), but
   "currently deleted" is **not** a census: GIN only ever increments
   `pages_deleted` for pages it deletes in this run, in `ginDeletePage`
   ([ginvacuum.c:234-235](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L234-L235)) and in
   `shiftList` ([ginfast.c:590-591](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L590-L591)).
   The server shows the difference inside one VACUUM: on the run that finally
   recycled the pages, the B-tree on the same table reported
   `551 in total, 0 newly deleted, 520 currently deleted, 520 reusable` while its
   GIN sibling reported `898 in total, 0 newly deleted, 0 currently deleted,
   768 reusable`.

3. **The `ginVersion = 2` gate is right but for a narrower reason than "posting
   tree pd_lower".** In a version-2 index every data page in the file was written
   by 9.4-or-later code, so `pd_lower` is maintained on compressed leaves
   ([gindatapage.c#dataPlaceToPageLeafRecompress](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L1018-L1023))
   and on internal data pages
   ([gindatapage.c#GinDataPageAddPostingItem](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L402-L411)) —
   both through `GinDataPageSetDataSize`, which writes `pd_lower` directly
   ([ginblock.h#GinDataPageSetDataSize](../../../../raw/postgres-17/src/include/access/ginblock.h#L310-L314)).
   The untrustworthy case is a binary-upgraded index that still holds pre-9.4
   pages, which is exactly what `ginVersion` distinguishes
   ([ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L85-L103),
   [ginblock.h:302-309](../../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309)). The
   census also classifies uncompressed data leaves separately, because GIN itself
   refuses to count their free space
   ([gindatapage.c#dataPlaceToPageLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L520-L528)).

### Why no single contrib function suffices

`pgstattuple` refuses GIN outright. `pgstat_relation` dispatches on `relam` and
the GIN arm falls straight into the "not supported" error
([pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L260-L296)):

```text
ERROR:  index "f5_deleted_gin" (gin index) is not supported
```

`pgstatginindex` accepts GIN but reads three fields from block 0 and nothing else
— `ginVersion`, `nPendingPages`, `nPendingHeapTuples`
([pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L507-L577),
[pgstatindex.c#GinIndexStat](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L97-L108)). It never
reads a second page, so it cannot see a dead page, a slack byte, or even the index
size; the documentation lists exactly those three output columns
([pgstattuple.sgml#pgstatginindex](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L298-L350)).
Measured consequence: on the flushed pending-list fixture, whose file was 64.64%
dead pages, `pgstatginindex` returned `version 2 | pending_pages 0 | pending_tuples 0`.

There is no GIN equivalent of `pgstatindex`/`pgstathashindex`, and `contrib/amcheck`
in this checkout ships only B-tree and heap verifiers — `bt_index_check`,
`bt_index_parent_check`
([amcheck--1.0.sql:9-20](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.0.sql#L9-L20),
[amcheck--1.3--1.4.sql:11-24](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.3--1.4.sql#L11-L24)) and
`verify_heapam`
([amcheck--1.2--1.3.sql:9-21](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.2--1.3.sql#L9-L21)) — so
there is no structural verifier to borrow page counts from either.

That leaves `pageinspect`, whose three GIN functions plus `page_header` are enough,
and `pg_freespacemap` as an independent check.

### The three waste classes, from GIN's own definitions

**Whole-page waste.** A GIN page is reusable when it is all-zero, or flagged
deleted with either no deletion xid or one the whole cluster has moved past
([ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)).
`PageIsNew` is `pd_upper == 0`
([bufpage.h#PageIsNew](../../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234)). Deleted pages
come from two places, and both keep their identity visible in the flags word:
posting-tree page deletion ORs in `GIN_DELETED` and stamps the delete xid into
`pd_prune_xid`
([ginvacuum.c#ginDeletePage](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192),
[ginblock.h:132-138](../../../../raw/postgres-17/src/include/access/ginblock.h#L132-L138)), while a
pending-list shift *replaces* the flags word with `GIN_DELETED`
([ginfast.c#shiftList](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L630-L635)) — which is why
no stale `list` pages can survive a flush and confuse the census.

`ginvacuumcleanup` then walks every block from `GIN_ROOT_BLKNO` up, records each
recyclable page in the FSM, counts the rest as entry or data pages, and reports the
recyclable count as `pages_free`
([ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L752-L802)). Those
pages are reusable **only inside the index**: `GinNewBuffer` takes them back from
the FSM before extending the file
([ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335)), and nothing
under `src/backend/access/gin/` truncates a relation — a grep for `RelationTruncate`
and `smgrtruncate` across that directory returns zero hits, and `ginvacuumcleanup`
ends by re-reading, not shortening, the file length
([ginvacuum.c:796-802](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L796-L802)).

**Live-page slack.** GIN's own free-space test on a data leaf is
`GinDataLeafPageGetFreeSpace`, defined as `PageGetExactFreeSpace`
([ginblock.h:287](../../../../raw/postgres-17/src/include/access/ginblock.h#L287)), which is exactly
`pd_upper - pd_lower` with no allowance for a line pointer
([bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L952-L973)).
`page_header` exposes both fields
([pageinspect--1.9--1.10.sql#page_header](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L10-L21)),
so the same subtraction is available in SQL. On an **entry** page the comparable
engine test is `PageGetFreeSpace`, which subtracts `sizeof(ItemIdData)` because a
new tuple also needs a line pointer
([ginentrypage.c#entryIsEnoughSpace](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482),
[bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923)); the
census therefore over-states entry-page slack by 4 bytes per tuple that would be
added, which is why it is an upper bound.

**Pending-list pages.** With `fastupdate` on, inserts land in a linked list of
`GIN_LIST` pages whose head, tail and counts live in the metapage
([ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L55-L83),
[ginfast.c#makeSublist](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L144-L209)), and the
documentation describes them as postponed work that VACUUM, autoanalyze,
`gin_clean_pending_list()` or a `gin_pending_list_limit` overflow will merge into
the main structure
([gin.sgml#GIN-Fast-Update](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537)). They are live data,
not waste — and the flush is not free, because merging appends to entry tuples and
allocates pages through `GinNewBuffer` when it has to.

### The procedure

1. **Install the three extensions.** `pageinspect` (1.12), `pgstattuple` (1.5) and
   `pg_freespacemap` (1.2) in this checkout
   ([pageinspect.control](../../../../raw/postgres-17/contrib/pageinspect/pageinspect.control#L1-L5),
   [pgstattuple.control](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.control#L1-L5),
   [pg_freespacemap.control](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.control#L1-L5)).
2. **Settle the state.** Run `VACUUM` on the table, or
   `gin_clean_pending_list(index)` for the pending list alone
   ([func.sgml#gin_clean_pending_list](../../../../raw/postgres-17/doc/src/sgml/func.sgml#L30104-L30123)). VACUUM
   is what refreshes `n_entry_pages` / `n_data_pages` / `n_total_pages` in the
   metapage ([ginvacuum.c:786-789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L786-L789)),
   and those fields are stale otherwise: the pending fixture's metapage said
   `n_total_pages = 268` while the file held 758 blocks. Expect to VACUUM
   **twice or more** for deleted pages; see
   [Deleted pages need the horizon to move before they count](#deleted-pages-need-the-horizon-to-move-before-they-count).
   On a standby you cannot do this step at all — a physical replica of this
   sandbox answered `ERROR: cannot execute VACUUM during recovery` and
   `ERROR: recovery is in progress` for `gin_clean_pending_list`, while every
   census function kept working — so a standby census reads whatever state
   replay has produced.
3. **Record the denominator and the metapage.** `pg_relation_size(idx, 'main')`
   is the main fork only
   ([func.sgml#pg_relation_size](../../../../raw/postgres-17/doc/src/sgml/func.sgml#L29627-L29643)), which is
   the right denominator because the census counts main-fork blocks;
   `gin_metapage_info(get_raw_page(idx, 0))` supplies `version` and the pending
   counters ([ginfuncs.c#gin_metapage_info](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95)).
4. **Census blocks 1 .. relpages-1**, reading each page once and passing the same
   `bytea` to `gin_page_opaque_info` and `page_header`.
5. **Cross-check three ways** before believing any number.
6. **Only then** compare with a rebuild, if you are deciding whether to rebuild.

### The census statement

Run on the pinned server exactly as printed; the timeouts are session-scoped and
the tags are the `wiki_` markers this repo requires.

```sql
SET /* wiki_gin_waste_guards */ statement_timeout = '10min';
SET /* wiki_gin_waste_guards */ lock_timeout = '2s';

WITH /* wiki_gin_waste_census */ gin_idx AS (
    SELECT c.oid                                  AS idx,
           c.oid::regclass::text                  AS idx_name,
           pg_relation_size(c.oid, 'main')         AS main_bytes,
           current_setting('block_size')::bigint   AS bs
    FROM pg_class c
         JOIN pg_am a    ON a.oid = c.relam
         JOIN pg_index x ON x.indexrelid = c.oid
    WHERE a.amname = 'gin'
      AND c.relkind = 'i'            -- 'I' (partitioned) has no storage
      AND x.indisvalid               -- an invalid index is still readable, see notes
      AND c.relpersistence <> 't'    -- another session's temp index is refused
),
meta AS (
    SELECT g.*, m.version, m.n_pending_pages, m.n_pending_tuples,
           m.n_total_pages, m.n_entry_pages, m.n_data_pages
    FROM gin_idx g
         CROSS JOIN LATERAL gin_metapage_info(get_raw_page(g.idx_name, 0)) m
),
pages AS (
    SELECT g.idx,
           CASE
               WHEN o.flags IS NULL                     THEN 'new'
               WHEN o.flags @> '{meta}'                 THEN 'meta'
               WHEN o.flags @> '{deleted}'              THEN 'deleted'
               WHEN o.flags @> '{list}'                 THEN 'pending'
               WHEN o.flags @> '{data,leaf,compressed}' THEN 'data_leaf'
               WHEN o.flags @> '{data,leaf}'            THEN 'data_leaf_uncompressed'
               WHEN o.flags @> '{data}'                 THEN 'data_internal'
               ELSE                                          'entry'
           END                                          AS page_class,
           COALESCE(h.upper - h.lower, 0)               AS slack
    FROM gin_idx g
         CROSS JOIN LATERAL generate_series(1, g.main_bytes / g.bs - 1) AS b(blkno)
         CROSS JOIN LATERAL (SELECT get_raw_page(g.idx_name, b.blkno) AS pg
                             OFFSET 0) AS r          -- read each page exactly once
         LEFT JOIN LATERAL gin_page_opaque_info(r.pg) AS o ON true
         LEFT JOIN LATERAL page_header(r.pg)          AS h ON true
),
census AS (
    SELECT idx,
           count(*) FILTER (WHERE page_class = 'entry')                  AS entry_pages,
           count(*) FILTER (WHERE page_class = 'data_leaf')              AS data_leaf_pages,
           count(*) FILTER (WHERE page_class = 'data_internal')          AS data_internal_pages,
           count(*) FILTER (WHERE page_class = 'data_leaf_uncompressed') AS uncompressed_pages,
           count(*) FILTER (WHERE page_class = 'pending')                AS pending_pages,
           count(*) FILTER (WHERE page_class = 'deleted')                AS deleted_pages,
           count(*) FILTER (WHERE page_class = 'new')                    AS new_pages,
           count(*) FILTER (WHERE page_class = 'meta')                   AS stray_meta_pages,
           COALESCE(sum(slack) FILTER (WHERE page_class = 'entry'), 0)   AS entry_slack,
           COALESCE(sum(slack) FILTER (WHERE page_class IN ('data_leaf',
                                            'data_internal')), 0)        AS data_slack
    FROM pages
    GROUP BY idx
)
SELECT m.idx_name                                       AS index_name,
       m.version                                        AS gin_version,
       m.main_bytes                                     AS main_fork_bytes,
       m.main_bytes / m.bs                              AS blocks,
       c.entry_pages, c.data_leaf_pages, c.data_internal_pages,
       c.uncompressed_pages, c.pending_pages, c.deleted_pages, c.new_pages,
       (c.deleted_pages + c.new_pages) * m.bs           AS whole_page_waste_bytes,
       round(100.0 * (c.deleted_pages + c.new_pages) * m.bs
             / nullif(m.main_bytes, 0), 2)              AS whole_page_waste_pct,
       CASE WHEN m.version = 2 THEN c.entry_slack + c.data_slack END
                                                        AS live_page_slack_bytes,
       CASE WHEN m.version = 2
            THEN round(100.0 * (c.entry_slack + c.data_slack)
                       / nullif(m.main_bytes, 0), 2) END AS live_page_slack_pct,
       c.entry_slack, c.data_slack,
       c.pending_pages * m.bs                           AS pending_bytes,
       round(100.0 * c.pending_pages * m.bs
             / nullif(m.main_bytes, 0), 2)              AS pending_pct,
       m.main_bytes - (c.deleted_pages + c.new_pages + c.pending_pages) * m.bs
                    - c.entry_slack - c.data_slack      AS payload_bytes,
       m.n_pending_pages                                AS meta_pending_pages,
       m.n_entry_pages                                  AS meta_entry_pages,
       m.n_data_pages                                   AS meta_data_pages,
       m.n_total_pages                                  AS meta_total_pages,
       1 + c.entry_pages + c.data_leaf_pages + c.data_internal_pages
         + c.uncompressed_pages + c.pending_pages + c.deleted_pages
         + c.new_pages + c.stray_meta_pages             AS census_total_pages
FROM meta m JOIN census c ON c.idx = m.idx
ORDER BY 1;
```

Add `AND c.oid = 'myschema.myindex'::regclass` to the `gin_idx` filter for a single
index.

Design points that are not cosmetic:

- **`OFFSET 0` keeps the page read single.** `get_raw_page` is volatile and the
  subquery is not flattened, so the page is fetched once and the same value feeds
  both LATERAL calls. Measured with `EXPLAIN (ANALYZE, BUFFERS)` on a 1235-block
  index: this form reports **1234** shared hits, the naive form that calls
  `get_raw_page` inside each function reports **2468** — exactly twice, one read
  per function call.
- **A NULL row means an all-zero page.** Both GIN functions return NULL early on
  `PageIsNew`
  ([ginfuncs.c:49-50](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L49-L50),
  [ginfuncs.c:119-120](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L119-L120)); the shipped
  test asserts exactly that
  ([pageinspect/sql/gin.sql:35-39](../../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L35-L39),
  [pageinspect/expected/gin.out:57-70](../../../../raw/postgres-17/contrib/pageinspect/expected/gin.out#L57-L70)).
  On the server, an all-zero `bytea` gave a row of NULLs from
  `gin_page_opaque_info`, NULL from `gin_metapage_info`, and
  `lower 0 | upper 0 | pagesize 0` from `page_header`, so the `COALESCE(..., 0)`
  contributes no slack. Real all-zero pages do occur: killing the server with
  `pg_ctl stop -m immediate` during a 600,000-row insert left **4** of them in
  `f16_zero_gin` on the first attempt and **7** on the second, out of 623 and 1043
  blocks. GIN extends the file through `GinNewBuffer` and initializes the page
  afterwards
  ([ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335)), so a crash
  in between leaves a zeroed block that `PageIsNew` — and therefore
  `GinPageIsRecyclable` — accepts.
- **`deleted` is tested before `data`/`list`.** A deleted posting-tree page keeps
  its `GIN_DATA` and `GIN_LEAF` bits, because `ginDeletePage` only ORs the deleted
  bit in ([ginvacuum.c:187-192](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192)); the
  server shows the resulting flag set as `{data,leaf,deleted,compressed}`.
- **`census_total_pages` must equal `blocks`.** It did on all 8 GIN indexes the
  statement selected in the sandbox, which is its own self-check that no page
  class was missed. Entry-tree *internal* pages are the class that makes the
  `ELSE` arm necessary: their flags word is 0, so `gin_page_opaque_info` returns
  an empty array rather than a NULL row, and each of the three rebuilt `tsvector`
  fixtures held exactly 6 of them beside 1193 entry leaves.

### Three cross-checks

**FSM.** `pg_freespace` reports the recorded value per block
([pg_freespacemap.c#pg_freespace](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L50)),
and for indexes the documentation says the value is meaningful only as
in-use-versus-empty
([pgfreespacemap.sgml:61-71](../../../../raw/postgres-17/doc/src/sgml/pgfreespacemap.sgml#L61-L71)). Free index
pages therefore read `8160`, and the count must match `deleted + new`:

```sql
SELECT /* wiki_gin_waste_fsm_check */ count(*) FILTER (WHERE avail = 8160) AS fsm_free_pages,
       count(*) AS blocks
FROM pg_freespace('myschema.myindex');
```

Measured: `768` FSM-free pages against `768` census-deleted pages on the
deleted-pages fixture, and `490` against `490` on the flushed pending fixture;
every other GIN index reported no free pages and no deleted pages. Note the
direction of the check — a page can be deleted but not yet recorded free, as it
was immediately after the first VACUUM (768 deleted, 0 in the FSM).

**VACUUM VERBOSE.** The fourth number of the index line is `pages_free`
([vacuumlazy.c:718-731](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731)):

```text
index "f5_deleted_gin": pages: 898 in total, 0 newly deleted, 0 currently deleted, 768 reusable
```

**Metapage page-type counts.** After a VACUUM, `n_entry_pages` and `n_data_pages`
are `ginvacuumcleanup`'s own census
([ginvacuum.c:766-789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L766-L789)), so they must
match the SQL census of live pages. They did, exactly, on every fixture — for
example `meta_entry_pages 1 / meta_data_pages 128` against a census of 1 entry
page, 96 compressed data leaves and 32 internal data pages. Two caveats fall
straight out of the code, and both showed up in the sandbox. A deleted page that
is not yet recyclable is counted by `ginvacuumcleanup` as a *data* page, because
`GinPageIsData` is tested after `GinPageIsRecyclable`: the churned fixture read
`meta_data_pages 77` against a census of 49 compressed data leaves and 14
internal data pages, the missing 14 being exactly its 14 deleted pages that were
not yet recyclable. And live `list` pages are counted in no bucket at all, which
is why `n_total_pages` can exceed `1 + n_entry_pages + n_data_pages`.

### The fixtures

Seven fixtures on an isolated 17.11 cluster built from the pinned source, with
`autovacuum = off` and `fsync = off`. The generators are deterministic — no
`random()`, no ordering dependence — so the block counts below are reproducible:

```sql
-- f1: entry-tree churn that replaces the key population
CREATE TABLE t1_churn (id int primary key, doc text);
INSERT INTO t1_churn SELECT i,
       'w' || (i % 50021) || ' w' || ((i * 7) % 50021) || ' w' || ((i * 13) % 50021)
              || ' w' || ((i * 17) % 50021) || ' w' || ((i * 23) % 50021)
              || ' w' || ((i * 29) % 50021) || ' hot' || (i % 7)
FROM generate_series(1, 200000) i;
CREATE INDEX f1_churn_gin ON t1_churn USING gin (to_tsvector('simple', doc))
       WITH (fastupdate = off);
UPDATE t1_churn SET doc =                     -- every row gets new terms ('v...')
       'v' || (id % 50021) || ' v' || ((id * 7) % 50021) || ' v' || ((id * 13) % 50021)
            || ' v' || ((id * 17) % 50021) || ' v' || ((id * 23) % 50021)
            || ' v' || ((id * 29) % 50021) || ' warm' || (id % 7);
VACUUM t1_churn; VACUUM t1_churn;

-- f3: untouched twin holding f1's post-churn content
CREATE TABLE t3_fresh (id int primary key, doc text);
INSERT INTO t3_fresh SELECT id, doc FROM t1_churn;
CREATE INDEX f3_fresh_gin ON t3_fresh USING gin (to_tsvector('simple', doc))
       WITH (fastupdate = off);
VACUUM t3_fresh;

-- f2: fastupdate, also the lifecycle fixture.  The pending list must be allowed
-- past the 4MB default or GIN flushes it in the foreground mid-insert.
SET gin_pending_list_limit = '1GB';
CREATE TABLE t2_pending (id int primary key, tags int[]);
INSERT INTO t2_pending SELECT i, ARRAY[i % 1000, (i * 7) % 1000, (i * 13) % 1000, i % 97]
FROM generate_series(1, 150000) i;
CREATE INDEX f2_pending_gin ON t2_pending USING gin (tags) WITH (fastupdate = on);
VACUUM t2_pending;
INSERT INTO t2_pending SELECT i, ARRAY[i % 1000, (i * 7) % 1000, (i * 13) % 1000, i % 97]
FROM generate_series(150001, 200000) i;      -- 50k rows into the pending list
SELECT gin_clean_pending_list('f2_pending_gin');

-- f4: empty table
CREATE TABLE t4_empty (id int primary key, tags int[]);
CREATE INDEX f4_empty_gin ON t4_empty USING gin (tags) WITH (fastupdate = off);
VACUUM t4_empty;

-- f5: every row carries all 32 keys, so each key gets a deep posting tree;
--     deleting the first 95% of the heap empties whole posting-tree leaves
CREATE TABLE t5_deleted (id int primary key, tags int[]);
INSERT INTO t5_deleted SELECT i, ARRAY(SELECT generate_series(0, 31))
FROM generate_series(1, 200000) i;
CREATE INDEX f5_deleted_gin ON t5_deleted USING gin (tags) WITH (fastupdate = off);
CREATE INDEX f5_deleted_btree ON t5_deleted (id);
DELETE FROM t5_deleted WHERE id <= 190000;   -- then the 3-VACUUM sequence below

-- f6: same shape, every other row deleted: leaves half-empty, nothing deletable
CREATE TABLE t6_slack (id int primary key, tags int[]);
INSERT INTO t6_slack SELECT i, ARRAY(SELECT generate_series(0, 31))
FROM generate_series(1, 200000) i;
CREATE INDEX f6_slack_gin ON t6_slack USING gin (tags) WITH (fastupdate = off);
DELETE FROM t6_slack WHERE id % 2 = 0;
VACUUM t6_slack; VACUUM t6_slack;

-- f7: churn that keeps the key population stable (three rewrite passes)
CREATE TABLE t7_reupdate (id int primary key, doc text);
INSERT INTO t7_reupdate SELECT i,
       'w' || (i % 50021) || ' w' || ((i * 7) % 50021) || ' w' || ((i * 13) % 50021)
              || ' w' || ((i * 17) % 50021) || ' w' || ((i * 23) % 50021)
              || ' w' || ((i * 29) % 50021) || ' hot' || (i % 7) || ' tog0'
FROM generate_series(1, 200000) i;
CREATE INDEX f7_reupdate_gin ON t7_reupdate USING gin (to_tsvector('simple', doc))
       WITH (fastupdate = off);
UPDATE t7_reupdate SET doc = replace(doc, 'tog0', 'tog1'); VACUUM t7_reupdate;
UPDATE t7_reupdate SET doc = replace(doc, 'tog1', 'tog0'); VACUUM t7_reupdate;
UPDATE t7_reupdate SET doc = replace(doc, 'tog0', 'tog1'); VACUUM t7_reupdate;
VACUUM t7_reupdate;
```

`f1` and `f7` are the two churn shapes that matter, and the difference is the key
population: `f1` replaces every term, so the old terms keep entry tuples with
empty posting lists, while `f7` rewrites the same terms three times and only
churns their TIDs. They score very differently below.

Eight more fixtures answer questions the first seven left open, and their recipes
are the ones above with one thing changed each:

| Fixture | Recipe |
|---|---|
| `f8_horizon_gin` | the `f5` recipe again, VACUUMed with a `REPEATABLE READ` snapshot held open |
| `f9_ric_gin` | the `f6` recipe again, rebuilt with `REINDEX INDEX CONCURRENTLY` |
| `f10_race_gin` | 1,000,000 rows, `int[]` of 6 keys drawn mod 5000 and mod 97, `fastupdate = on` |
| `f11_json_gin` | 200k `jsonb` rows, `USING gin (doc jsonb_path_ops)` |
| `f12_trgm_gin` | 200k text rows, `USING gin (txt gin_trgm_ops)` |
| `f13_btgin_gin` | 200k `int` rows mod 20011, `USING gin (n)` from `btree_gin` |
| `f14_multi_gin` | multicolumn `USING gin (tags, to_tsvector('simple', doc))` |
| `f15_partial_gin` | `USING gin (tags) WHERE live`, with `live` true for 10% of rows |
| `f16_zero_gin` | a 600k-row insert killed mid-flight by `pg_ctl stop -m immediate` |

`f11` through `f15` were then churned — every key replaced, and for `f15` the
predicate column rewritten too — and VACUUMed twice, so each one is scored the
same way as the first seven.

Provenance: these fixtures were rebuilt for the review because the first run's
sandbox had been deleted and its fixture SQL was never published. Three of the
seven — `f4`, `f5` and `f6` — came out byte-identical to the first run's filed
figures (16,384 / 7,356,416 / 7,356,416 bytes, the same page-class censuses, the
same `REINDEX` results), which is what identifies the reconstruction as faithful;
the other numbers on this page are this run's. The recipe behind `f5`/`f6` has now
built the same 7,356,416-byte, 898-block index **four** times (`f5`, `f6`, `f8`,
`f9`), so that shape is the page's reproducibility anchor.

### What the census meant against REINDEX

`REINDEX INDEX` before/after `pg_relation_size` is the ground truth for
filesystem-reclaimable bytes.

| Fixture | What it is | bytes | blocks | entry / data-leaf / data-int / pending / deleted |
|---|---|---|---|---|
| `f1_churn_gin` | `tsvector`, `fastupdate=off`, every row's term set replaced, then 2 VACUUMs | 17,571,840 | 2145 | 2067 / 49 / 14 / 0 / 14 |
| `f2_pending_gin` | `int[]`, `fastupdate=on`, 50k rows left in the pending list, then flushed | 6,209,536 | 758 | 170 / 97 / 0 / 0 / 490 |
| `f3_fresh_gin` | untouched twin of `f1` | 10,117,120 | 1235 | 1199 / 28 / 7 / 0 / 0 |
| `f4_empty_gin` | empty table | 16,384 | 2 | 1 / 0 / 0 / 0 / 0 |
| `f5_deleted_gin` | 32 keys per row, first 95% of the heap deleted, 3 VACUUMs | 7,356,416 | 898 | 1 / 96 / 32 / 0 / 768 |
| `f6_slack_gin` | same shape, every other row deleted | 7,356,416 | 898 | 1 / 864 / 32 / 0 / 0 |
| `f7_reupdate_gin` | `tsvector`, same terms rewritten 3 times, then VACUUMs | 11,927,552 | 1456 | 1200 / 102 / 9 / 0 / 144 |

| Fixture | waste % | slack % (entry / data bytes) | waste+slack % | REINDEX reclaimed % | waste ≤ truth | waste+slack ≥ truth |
|---|---|---|---|---|---|---|
| `f1_churn_gin` | 0.65 | 63.47 (10,861,112 / 292,082) | 64.12 | 42.42 | yes | yes |
| `f2_pending_gin` | 64.64 | 10.65 (318,084 / 343,360) | 75.30 | 58.05 | **no** | yes |
| `f3_fresh_gin` | 0.00 | 48.05 (4,796,424 / 65,192) | 48.05 | 0.00 | yes | yes |
| `f4_empty_gin` | 0.00 | 49.80 (8,160 / 0) | 49.80 | 0.00 | yes | yes |
| `f5_deleted_gin` | 85.52 | 9.70 (7,520 / 706,048) | 95.22 | 89.09 | yes | yes |
| `f6_slack_gin` | 0.00 | 51.34 (7,520 / 3,769,344) | 51.34 | 46.33 | yes | yes |
| `f7_reupdate_gin` | 9.89 | 44.26 (4,804,868 / 474,756) | 54.15 | 13.26 | yes | yes |

Absolute rebuild results: `f1` 17,571,840 -> 10,117,120; `f2` 6,209,536 ->
2,605,056; `f3` and `f4` unchanged; `f5` 7,356,416 -> 802,816; `f6` 7,356,416 ->
3,948,544; `f7` 11,927,552 -> 10,346,496. `f1` rebuilt to exactly its untouched
twin's size and page-class census, which is a second self-check on the pair.

The same scoring over five other opclasses and index shapes, each freshly built,
then churned so every key is replaced, then VACUUMed twice:

| Fixture | churned bytes | waste % | slack % | waste+slack % | REINDEX reclaimed % | fresh-build fill % |
|---|---|---|---|---|---|---|
| `f11_json_gin` (`jsonb_path_ops`) | 11,378,688 | 32.69 | 39.74 | 72.42 | 50.90 | 55.78 |
| `f12_trgm_gin` (`gin_trgm_ops`) | 20,652,032 | 29.63 | 47.87 | 77.50 | 68.98 | 72.11 |
| `f13_btgin_gin` (`btree_gin` int4) | 6,414,336 | 62.71 | 18.22 | 80.93 | 62.84 | 51.30 |
| `f14_multi_gin` (multicolumn) | 12,795,904 | 26.89 | 47.27 | 74.16 | 58.83 | 53.48 |
| `f15_partial_gin` (partial) | 2,834,432 | 83.53 | 10.30 | 93.82 | 89.60 | 58.54 |

Both bounds hold on all five, so across all twelve scored fixtures
`waste + slack` bounded the truth from above **12 of 12** and `waste` alone
bounded it from below **11 of 12**. `f13_btgin_gin` is the near miss that shows
the lower bound is not safe by much: 62.71% dead against 62.84% reclaimed, a
margin of 0.13 points. Fresh-build fill is clearly opclass-dependent — 51.30% for
`btree_gin` against 72.11% for `pg_trgm`, with `f10_race_gin`'s six-key `int[]`
at 1,000,000 rows reading 68.98% — so "how full is a fresh GIN build" has no
single answer to compare an aged index against.

`REINDEX INDEX CONCURRENTLY` is the same ground truth as the plain form, at least
on this shape: `f9_ric_gin`, built by the identical recipe to `f6_slack_gin`, went
from 7,356,416 bytes to **3,948,544** — the same byte count plain `REINDEX` gave
`f6` — in 379 ms against 151 ms, leaving `indisvalid`/`indisready`/`indislive` all
true and no `_ccnew`/`_ccold` leftovers, and a plain `REINDEX` immediately
afterwards returned the same 3,948,544.

The census's fourth number, `payload_bytes` (size minus dead pages minus pending
pages minus slack), is the quantity that is *supposed* to survive a rebuild.
Dividing it by the fill fraction of the rebuilt index predicts the rebuilt size
within 3.1% on six of the seven fixtures (`f2` −0.09%, `f3` −0.00%, `f4` +0.00%,
`f5` +1.12%, `f6` +3.07%, `f7` +0.07%) and misses `f1` by **+19.94%**. The miss
has a mechanism, and it is the reason `f1` and `f7` are both here: GIN's entry
tree never deletes a tuple
([README:389-396](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396)), so after
`f1`'s churn the index still carries an entry tuple for each of the 50k terms
that no longer occur. The census counts those tuples as payload; the rebuild
drops them. `f7`, whose churn leaves the term set alone, is predicted to +0.07%.

The five opclass fixtures reproduce that split independently. Four of them land at
+0.65%, +1.49%, +0.00% and +0.63%, and the fifth — `f14_multi_gin`, whose churn
replaced *both* key columns — misses by **+18.12%**, the same failure mode as
`f1`. So the model's error is a function of how much of the key population died,
not of the opclass. Two further caveats keep this out of predictor territory: the
fill fraction comes from the rebuild it is predicting, and both terms are measured
after the fact; see [Open Questions](#open-questions).

### Whole-page waste is not a lower bound

`f2_pending_gin` is the counterexample. After flushing its pending list it held
490 dead pages out of 758 — 64.64% of the file, confirmed independently by the
FSM — yet `REINDEX` returned only 58.05%. The rebuild is *bigger* than the
in-use part of the aged index: 268 in-use blocks became 318 blocks after
`REINDEX`, 18.7% larger.

The mechanism is in the entry-tree split rule. `entrySplitPage` divides a full
page by equalizing data size, halving it
([ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691)),
and there is no fast-append special case, so a build that inserts keys in order
leaves every page around half to two-thirds full. Payload as a fraction of file
size measured 51.95%, 58.94%, 51.95%, 50.20%, 43.28%, 87.96% and 52.81% on the
seven rebuilt fixtures — only `f6`, whose posting-tree leaves are packed by a
bulk build, is dense. An aged index whose slack has been eaten by later inserts
is denser than its own rebuild.

Practical consequence: **do not report dead pages as "reclaimable by REINDEX".**
Report them as what the source says they are — pages this index will reuse before
extending the file. That reuse is measurable, not just a code path: with
`f2_pending_gin` sitting at 809 blocks of which 491 were dead, pushing another
50,000 rows through its pending list grew the file by **26 blocks**, not by the
~490 the new pending pages needed, because `GinNewBuffer` took the rest back from
the FSM
([ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335)).

### Entry-page slack is growth room, not waste

GIN entry tuples grow in place: the posting list for a key lives inside the entry
tuple until it outgrows `GinMaxItemSize` and becomes a posting tree
([ginblock.h#GinMaxItemSize](../../../../raw/postgres-17/src/include/access/ginblock.h#L242-L253),
[ginentrypage.c#GinFormTuple](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L96-L114)), and
the entry tree never deletes tuples or pages at all
([README:389-396](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396),
[README:26-30](../../../../raw/postgres-17/src/backend/access/gin/README#L26-L30)). Free space on an entry page
is therefore the room its existing tuples need in order to grow.

Three measurements make the point:

- The never-churned control `f3_fresh_gin` reports **48.05% slack** and `REINDEX`
  reclaims **0 bytes**. A fresh GIN index looks half wasteful by this metric.
- Merging 50,000 rows' worth of pending entries into the pending fixture consumed
  slack rather than pages: entry slack fell from 575,940 to 318,084 bytes while
  the entry tree stayed at 170 pages and the data pages at 97.
- Repeating that on the freshly rebuilt index (41.06% slack, 318 blocks) absorbed
  another 50,000 rows with **zero** growth in either the entry tree (220 pages
  before and after) or the posting trees (97 pages).

Data-leaf slack behaves much more like waste: `f6_slack_gin`'s 51.34% (almost all
of it on posting-tree leaves) against 46.33% actually reclaimed. Posting-tree
leaves do not merge — `ginScanToDelete` deletes only pages that are entirely empty,
and never the leftmost or rightmost branch
([ginvacuum.c#ginScanToDelete](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L303-L318)) — so a
half-emptied leaf stays half-empty until new TIDs in its key range arrive. The
census sees the difference in the split: `f6` holds 3,769,344 bytes of data slack
against 7,520 bytes of entry slack, and it is the only fixture whose slack
percentage lands near its reclaimed percentage.

That is why the statement reports `entry_slack` and `data_slack` separately, and
why summing them into a single "bloat" number is misleading.

### The pending-list lifecycle, measured end to end

One fixture, four states, same statement:

| State | bytes | blocks | entry | pending | deleted | whole-page waste | entry slack | `pgstatginindex` |
|---|---|---|---|---|---|---|---|---|
| built, 150k rows | 2,195,456 | 268 | 170 | 0 | 0 | 0 | 575,940 | `0 / 0` |
| +50k rows into the pending list | 6,209,536 | 758 | 170 | 490 | 0 | 0 (64.64% pending) | 575,940 | `490 / 50000` |
| after `gin_clean_pending_list()` | 6,209,536 | 758 | 170 | 0 | 490 | 4,014,080 (64.64%) | 318,084 | `0 / 0` |
| after `REINDEX INDEX` | 2,605,056 | 318 | 220 | 0 | 0 | 0 | 725,084 | `0 / 0` |

Four things to take from it. The flush **did not shrink the file**: it converted
64.64% of it into dead pages rather than returning them (`shiftList` records each
shifted page free and `ginInsertCleanup` vacuums the FSM at the end
([ginfast.c:662-670](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L662-L670),
[ginfast.c:1014-1020](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1014-L1020))), and it can
just as easily *grow* it, because the merge allocates through `GinNewBuffer` and
extends when the FSM has nothing to give — here the file happened to stay at 758
blocks, while the first run of this fixture family grew by one block. The census's
`pending_pages` matched the metapage's `n_pending_pages` exactly (490), which is a
free correctness check on the classification. `pgstatginindex` is blind in three
of the four states. And a `VACUUM` instead of the SQL flush reports the freed pages
as reusable **in the same run** —

```text
index "f2_pending_gin": pages: 835 in total, 0 newly deleted, 490 currently deleted, 490 reusable
```

— because `shiftList` leaves `pd_prune_xid` at 0, so `GinPageIsRecyclable` returns
true at its "delete xid is invalid" branch
([ginvacuum.c:816-822](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L816-L822)).

### Deleted pages need the horizon to move before they count

Posting-tree deletions behave in the opposite way, and this is the single most
surprising operational detail in the whole procedure. `ginDeletePage` stamps
`ReadNextTransactionId()` into the page
([ginvacuum.c:187-192](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192)) and
`GinPageIsRecyclable` clears the page only once
`GlobalVisCheckRemovableXid(NULL, delete_xid)` agrees
([ginvacuum.c:805-829](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829)). Passing
`NULL` for the relation selects `VISHORIZON_SHARED`, documented as the most
conservative horizon
([procarray.c#GlobalVisHorizonKindForRel](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1966-L1991),
[procarray.c#GlobalVisCheckRemovableXid](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L4294-L4306)).

On the idle sandbox that produced this sequence for one index:

| Step | VACUUM VERBOSE index line | FSM free pages |
|---|---|---|
| VACUUM 1 (does the deleting) | `898 in total, 768 newly deleted, 768 currently deleted, 0 reusable` | 0 |
| VACUUM 2, nothing else ran | `898 in total, 0 newly deleted, 0 currently deleted, 0 reusable` | 0 |
| three `pg_current_xact_id()` calls, then VACUUM 3 | `898 in total, 0 newly deleted, 0 currently deleted, 768 reusable` | 768 |

The census saw the pages the whole time — 768 blocks flagged
`{data,leaf,deleted,compressed}`, every one carrying the same `prune_xid` (791 in
this run; the value is whatever `ReadNextTransactionId()` returned during the
deleting VACUUM, so it is the one number here that is not reproducible across
clusters) — which is the practical argument for the page census over the FSM
cross-check: `pg_freespace` under-reports until the horizon has moved *and*
another VACUUM has run, while the flags are true immediately. The file never
shrank at any step, and stayed at 7,356,416 bytes through all three VACUUMs.

**One idle transaction is enough to freeze that indefinitely.** `f8_horizon_gin`
repeats the sequence with a `REPEATABLE READ` snapshot held open in a second
session:

| Step | VACUUM VERBOSE index line | FSM free pages |
|---|---|---|
| VACUUM 1, snapshot held | `898 in total, 768 newly deleted, 768 currently deleted, 0 reusable` | 0 |
| 3 xids, VACUUM 2, snapshot held | `898 in total, 0 newly deleted, 0 currently deleted, 0 reusable` | 0 |
| 3 more xids, VACUUM 3, snapshot held | `898 in total, 0 newly deleted, 0 currently deleted, 0 reusable` | 0 |
| snapshot released, VACUUM 4, **no new xids** | `898 in total, 0 newly deleted, 0 currently deleted, 768 reusable` | 768 |

The numbers line up exactly: all 768 pages carry `prune_xid = 870`, which is the
value the holder's `pg_stat_activity.backend_xmin` reported, and after release the
cluster's snapshot xmax was already 882. So consuming transaction ids is not the
requirement — moving the *shared* horizon past the stamped xid is, and the very
next VACUUM after the snapshot went away recycled all 768 pages without a single
new transaction. An idle `REPEATABLE READ` session, a forgotten `BEGIN`, or any
long-running query therefore keeps every dead GIN page unreusable, while the
census keeps reporting them as dead all along.

### Concurrency: a census of a busy index is a mixed-instant reading

The non-atomic scan is not a theoretical caveat. Censusing `f10_race_gin` (a
1,000,000-row `fastupdate` index) 25 times while another session pushed 5,000 rows
through its pending list and flushed them in a loop:

| What was compared | Result over 25 runs |
|---|---|
| `census_total_pages = blocks` (the statement's self-check) | held **25 of 25** — it never noticed |
| census `pending_pages` against `meta_pending_pages` in the same statement | disagreed in **19 of 25** (74 against 2, 12 against 74, 8 against 74 with 66 already `deleted`, and so on) |
| `blocks` against the file size read immediately afterwards | differed in **25 of 25**, by up to **664 blocks** (5,438 against 6,102) |

The self-check is structurally blind here: `census_total_pages` and `blocks` are
both derived from the one `pg_relation_size` reading taken in the first CTE, so
they agree even when that reading is already stale. What actually catches the race
is the metapage cross-check — a census whose `pending_pages` does not equal
`meta_pending_pages` has read pages from two different instants.

**Autovacuum is the version of this that hits an idle-looking table.** With
`autovacuum = on` — a reload, not a restart, since the GUC is `PGC_SIGHUP`
([guc_tables.c:1449-1457](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457)) — and
only *analyze* made eager, a 491-page pending list vanished within 10 seconds with
no manual VACUUM: `pending_pages` went 491 -> 0 with `last_autoanalyze` set and
`last_autovacuum` still null, which is `ginvacuumcleanup`'s `analyze_only` branch
calling `ginInsertCleanup` in an autovacuum worker
([ginvacuum.c:705-717](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717)). The
file went from 836 to 958 blocks in the process and the census's waste reading
jumped to 51.25%. Two readings a few seconds apart can therefore differ by 15% of
the file with nobody touching the table.

### Cost of the census

The scan is one buffer read per block, through the ordinary buffer manager with no
ring buffer: `get_raw_page_internal` calls `ReadBufferExtended(..., RBM_NORMAL, NULL)`
([rawpage.c#get_raw_page_internal](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L181-L196)),
where `pgstattuple` and `pgstatindex` both allocate a `BAS_BULKREAD` strategy
([pgstattuple.c:544](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L544),
[pgstatindex.c:222](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L222)). A census of a large
index therefore pulls the whole index into `shared_buffers` and can evict live
data — the one real operational cost of this approach.

Measured on a 1263-block index after a server restart: cold, `shared read=1262`
(plus 20 catalog hits) and 16.311 ms; warm, `shared hit=1282` and 11.330 /
11.669 / 11.656 ms over three runs (`fsync = off`, local SSD, everything in page
cache). One read per block, exactly as the source says. Two consecutive runs of
the published statement over all 8 GIN indexes produced byte-identical output
(1,244 bytes of CSV), and the whole-database census cost 34-37 ms.

The cache effect is real and was measured with `pg_buffercache` at
`shared_buffers = 64MB` (8,192 buffers):

- A census of the 6,665-block `f10_race_gin` left **all 6,665 pages resident**,
  81% of the cache, every one at `usagecount = 1`.
- The contrast with a ring-buffered reader is stark. A plain `SELECT count(*)` over
  the 4,092-block `t1_churn` left only **98** of its pages behind, because a seq
  scan over a table bigger than `NBuffers / 4` takes a `BAS_BULKREAD` strategy
  ([heapam.c:434-458](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L434-L458)). The census has
  no such limiter.
- Eviction follows immediately once the cache is full: censusing four more indexes
  (4,238 blocks) after that first census dropped `f10_race_gin` from 6,665
  resident pages to **3,723**, with 0 free buffers left. The census evicts, and
  what it evicts first is its own earlier pages.

So on a production box, size the census against `shared_buffers`, or accept that
an index-sized slice of the cache is going to turn over.

### Privileges

| Function | Who can run it |
|---|---|
| `get_raw_page`, `page_header`, `gin_metapage_info`, `gin_page_opaque_info`, `gin_leafpage_items` | superuser only, hard-coded |
| `pgstatginindex` | superuser, or a role with `pg_stat_scan_tables`, or an explicit grant |
| `pg_freespace` | same |
| `gin_clean_pending_list` | the index owner |

Every `pageinspect` entry point is gated on `superuser()` in C, some directly and
some through a shared internal — the raw-page reader, which is where all four
`get_raw_page` variants land
([rawpage.c:150-153](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L153)), `page_header`
([rawpage.c:261-264](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L261-L264)) and each GIN function
([ginfuncs.c:42-45](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L42-L45),
[ginfuncs.c:112-115](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L112-L115),
[ginfuncs.c:188-191](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L188-L191)) — which the module
documentation states as a blanket rule
([pageinspect.sgml:10-14](../../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L10-L14)). A `GRANT` cannot
help: the check is not an ACL. By contrast `pgstatginindex` dropped its
`superuser()` call in the 1.5 wrapper and is granted to `pg_stat_scan_tables`
([pgstattuple--1.4--1.5.sql:49-57](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57),
[pgstatindex.c:497-504](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L497-L504)), and
`pg_freespacemap` 1.2 grants both `pg_freespace` forms to the same role
([pg_freespacemap--1.1--1.2.sql:6-7](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1--1.2.sql#L6-L7)).
`gin_clean_pending_list` requires ownership, "comparable to privileges needed for
VACUUM" ([ginfast.c:1061-1064](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1061-L1064)).

Verified with a role holding only `pg_stat_scan_tables` and `USAGE` on the schema:
`pgstatginindex` returned `2 | 0 | 0`, `pg_freespace` returned all 98 rows, and
`pgstattuple` on the table returned its length; `get_raw_page` returned
`ERROR: must be superuser to use raw page functions`; and
`gin_clean_pending_list` returned `ERROR: must be owner of index f6_slack_gin`.

### Refusals, silent answers, and other traps

Everything in this table was reproduced on the pinned server.

| Case | Behavior |
|---|---|
| `pgstattuple` on a valid GIN index | `ERROR: index "..." (gin index) is not supported` ([pgstattuple.c:280-296](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L280-L296)) |
| `pgstattuple` on an *invalid* GIN index | `ERROR: index "..." is not valid` — the validity check runs before the AM switch ([pgstattuple.c:263-267](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L263-L267)) |
| `pgstatginindex` on a non-GIN index | `ERROR: relation "..." is not a GIN index` ([pgstatindex.c:522-526](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L522-L526)) |
| `pgstatginindex` on an invalid index | `ERROR: index "..." is not valid` ([pgstatindex.c:538-543](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L538-L543)) |
| `pgstatginindex` on a *partitioned* GIN index | `ERROR: relation "..." is not a GIN index`, because `IS_INDEX` tests `relkind = 'i'` ([pgstatindex.c:70-73](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L70-L73)) |
| `get_raw_page` on a partitioned index | `ERROR: cannot get raw page from relation "..."`, `DETAIL: This operation is not supported for partitioned indexes.` ([rawpage.c:158-163](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L158-L163)) |
| `pg_freespace` on a partitioned index | **0 rows, no error** — the set-returning wrapper's `generate_series` is empty ([pg_freespacemap--1.1.sql:13-21](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L13-L21)) |
| any of them on another session's temp GIN index | `pgstatginindex`: `cannot access temporary indexes of other sessions` ([pgstatindex.c:528-536](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L528-L536)); `get_raw_page`: `cannot access temporary tables of other sessions` ([rawpage.c:165-173](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L165-L173)); **`pg_freespace` answers anyway**, with no such guard in its C function ([pg_freespacemap.c:24-50](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L50)) |
| `get_raw_page` past the end of the index | `ERROR: block number 100000 is out of range for relation "..."` ([rawpage.c:175-179](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L175-L179)) |
| `gin_leafpage_items` on any page that is not exactly `{data,leaf,compressed}` | `ERROR: input page is not a compressed GIN data leaf page`, `DETAIL: Flags 0002, expected 0083` — the reported flags are the page's own, so an entry leaf shows `0002` and an entry-tree internal page `0000` ([ginfuncs.c:219-226](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L219-L226)) |
| `gin_clean_pending_list` on an invalid index | returns 0, logs at `DEBUG1` ([ginfast.c:1068-1086](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1068-L1086)) |
| `gin_clean_pending_list` during recovery | `ERROR: recovery is in progress`, `HINT: GIN pending list cannot be cleaned up during recovery.`, reproduced on a physical standby of this sandbox, where `VACUUM` is also refused (`cannot execute VACUUM during recovery`) while `pgstatginindex`, `get_raw_page` and `gin_page_opaque_info` all answer ([ginfast.c:1037-1041](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1037-L1041)) |
| a `bytea` that is not one block long | `ERROR: invalid page size` ([rawpage.c#get_page_from_raw](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L214-L234)) |

Two more, from the code rather than from an error message:

- **The scan is not atomic.** `get_raw_page_internal` locks and copies **one**
  page per call ([rawpage.c:186-196](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L186-L196)), so a
  census of *N* blocks is *N* consistent page images taken at *N* different
  instants. Concurrent inserts, a VACUUM, or a pending-list flush can move pages
  between classes mid-scan, and the `census_total_pages = blocks` check does not
  notice — see
  [Concurrency: a census of a busy index is a mixed-instant reading](#concurrency-a-census-of-a-busy-index-is-a-mixed-instant-reading)
  for what that costs in practice.
- **`gin_leafpage_items` is not needed by this procedure**, and its exact-flag
  requirement is the reason: it accepts a page only when `opaq->flags` equals
  `GIN_DATA | GIN_LEAF | GIN_COMPRESSED` exactly, so a deleted or
  incompletely-split leaf is rejected. Per-TID detail is not required for a byte
  census.

Also worth knowing: an index whose `indisvalid` is false is still fully readable by
`get_raw_page`, which has no validity check. The census statement filters those
indexes out to keep its results interpretable, but dropping `AND x.indisvalid`
makes it the only working way to size the waste in a failed
`CREATE INDEX CONCURRENTLY` leftover — verified on an invalid GIN index where
`pgstatginindex` and `pgstattuple` both refused while
`gin_metapage_info(get_raw_page(...))` reported `version 2` over its 2 blocks.

### Timeouts and GUC scope

| Setting | Context | Scope | Use here |
|---|---|---|---|
| `statement_timeout` | `PGC_USERSET` ([guc_tables.c:2611-2620](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)) | session/transaction | cap the census; `10min` for a multi-GB index |
| `lock_timeout` | `PGC_USERSET` ([guc_tables.c:2622-2631](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)) | session/transaction | `2s`, so a concurrent `DROP INDEX`/`REINDEX` does not park the census |
| `gin_pending_list_limit` | `PGC_USERSET` ([guc_tables.c:3576-3585](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585)) | session/transaction | only relevant if you deliberately grow a pending list; it can also be set per index as a storage parameter ([gin.sgml#GIN-Tips](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L595-L616)) |

Both timeouts were exercised against an uncommitted `DROP INDEX` holding
`AccessExclusiveLock`: `lock_timeout = '2s'` cancelled the census at 2001.272 ms
with `canceling statement due to lock timeout`, and `statement_timeout = '1500ms'`
with `lock_timeout = 0` cancelled at 1500.477 ms with `canceling statement due to
statement timeout`. Neither left anything behind, and the index was still present
afterwards.

### Reading rules

- Report the three classes separately. Never publish their sum as "bloat".
- `whole_page_waste_bytes` is space this index will reuse. It is not a promise
  about `REINDEX`, in either direction.
- `entry_slack` on a healthy index is a large fraction of an entry-tree-dominated
  file: the freshly built and freshly rebuilt indexes here read 41.06%, 47.19%,
  48.05% and 48.05% slack while reclaiming nothing, and across opclasses the fresh
  payload fraction ran from 51.30% (`btree_gin`) to 72.11% (`pg_trgm`). Treat it as
  a level, not a defect, and compare an index to its own history or to a rebuilt
  twin — never to another opclass.
- Read a busy index twice before believing either reading, and check
  `pending_pages` against `meta_pending_pages`: on a fixture under concurrent
  inserts those two disagreed in 19 of 25 censuses while the statement's own
  self-check passed every time.
- Dead pages that will not go away may be waiting on one idle transaction, not on
  VACUUM. A held `REPEATABLE READ` snapshot kept 768 recyclable-looking pages out
  of the FSM across three VACUUMs.
- `data_slack` is the honest bloat signal: a posting-tree-dominated index with
  slack far above a fresh rebuild's has genuinely reclaimable space.
- `pending_bytes` is a `fastupdate` tuning signal, not waste. Fix it with VACUUM,
  `gin_clean_pending_list()`, or `gin_pending_list_limit`.
- If `gin_version <> 2`, publish the page counts and suppress the slack columns,
  which the statement already does.
- If `uncompressed_pages > 0`, the index carries pre-9.4 posting-tree leaves and
  its `data_slack` is understated.

## Context Reviewed

- Contrib limits: `pgstat_relation`'s AM switch, `pgstatginindex_internal`, the
  `GinIndexStat` struct, the 1.5 grant script, and the `BAS_BULKREAD` strategy
  that `pgstattuple`/`pgstatindex` use and `pageinspect` does not.
- `pageinspect`: `get_raw_page_internal`, `get_page_from_raw`, `page_header`, all
  three GIN functions, the `page_header` signature added in 1.10, and the shipped
  GIN test including its all-zero-page expectations.
- `pg_freespacemap`: the C function, both SQL forms, the 1.2 grants, and the
  documentation's index-specific note.
- GIN on-disk structures: `GinPageOpaqueData` and every flag, `GinMetaPageData`,
  the page-type and delete-xid macros, `GinMaxItemSize`, the posting-tree layout
  comment, `GinDataLeafPageGetFreeSpace`, and the `pd_lower` trust note.
- GIN vacuum: `ginbulkdelete`, `ginvacuumcleanup`, `GinPageIsRecyclable`,
  `ginDeletePage`, `ginScanToDelete`, `ginVacuumPostingTreeLeaves`, and the
  README's page-deletion and entry-tree sections.
- GIN fast update: `writeListPage`, `makeSublist`, `ginHeapTupleFastInsert`,
  `shiftList`, `ginInsertCleanup`, `gin_clean_pending_list`.
- Page and FSM primitives: `PageIsNew`, `PageGetFreeSpace`,
  `PageGetExactFreeSpace`, `indexfsm.c` in full, the FSM category arithmetic,
  `GetRecordedFreeSpace`, and `MaxHeapTupleSize`.
- Visibility: `GlobalVisHorizonKindForRel`, `GlobalVisCheckRemovableXid`.
- Adjacent code: `GinNewBuffer`, `GinInitMetabuffer`, `ginUpdateStats`,
  `entryIsEnoughSpace`, `entrySplitPage`, `GinDataPageAddPostingItem`,
  `dataPlaceToPageLeaf`, and `vacuumlazy.c`'s per-index verbose line.
- Docs and GUCs: `pgstattuple.sgml`, `pageinspect.sgml`, `pgfreespacemap.sgml`,
  `gin.sgml` fast-update and tips sections, `func.sgml` for `pg_relation_size` and
  `gin_clean_pending_list`, and `gin_pending_list_limit`, `statement_timeout` and
  `lock_timeout` in `guc_tables.c`.
- Test surfaces: the `pageinspect` GIN test, the `pg_freespacemap` test, and the
  `amcheck` SQL scripts that show no GIN verifier exists.
- Live measurements on an isolated 17.11 cluster built from the pinned checkout by
  a VPATH build that leaves `raw/` untouched (`.wiki-runtime/tmp/ginw2/`, port
  55432, `autovacuum = off`, `fsync = off`, `shared_buffers = 256MB`): the seven
  fixtures published above, a partitioned GIN index, an invalid GIN index, another
  session's temp GIN index, a `pg_stat_scan_tables`-only role, a physical standby
  built with `pg_basebackup` for the recovery refusals, `REINDEX INDEX` ground
  truth, `EXPLAIN (ANALYZE, BUFFERS)` read counts, cold/warm timings across a
  restart, and both timeout cancellations. The first run's sandbox
  (`.wiki-runtime/tmp/ginw`) and the 17.11 install it borrowed no longer exist,
  which is why the server was rebuilt and the fixtures republished.
- Follow-up experiments on the same cluster, with `pg_trgm` 1.6, `btree_gin` 1.3
  and `pg_buffercache` 1.5 additionally installed from the pinned tree: a held
  `REPEATABLE READ` snapshot against page recyclability, `REINDEX INDEX
  CONCURRENTLY` as an alternative ground truth, 25 censuses of a 1,000,000-row
  index under concurrent inserts and flushes, an autoanalyze-only pending-list
  flush with `autovacuum = on` reloaded, five other opclasses and index shapes
  built then churned, `pg_buffercache` accounting at `shared_buffers = 64MB`, and
  two `pg_ctl stop -m immediate` crashes during a bulk insert.
- Review re-verification: all 119 source citations the page carried at review time
  re-read against the pinned checkout, plus three repo-wide checks — `RelationTruncate` and
  `smgrtruncate` have zero hits under `src/backend/access/gin/`, `pages_deleted`
  is incremented in exactly two places
  ([ginvacuum.c:234-235](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L234-L235),
  [ginfast.c:590-591](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L590-L591)), and
  `contrib/amcheck` declares only `bt_index_check`, `bt_index_parent_check` and
  `verify_heapam` across all its SQL scripts.

## Evidence Map

| Claim | Evidence |
|---|---|
| `pgstattuple` rejects GIN | [pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L260-L296); server: `index "f5_deleted_gin" (gin index) is not supported` |
| `pgstattuple` checks validity before the AM | [pgstattuple.c:263-267](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L263-L267); server: invalid GIN index reported `is not valid` |
| `pgstatginindex` reads only three metapage fields | [pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L507-L577), [pgstattuple.sgml:298-350](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L298-L350); server: `0 / 0` on a 64.64%-dead index |
| No GIN verifier in v17 contrib amcheck | [amcheck--1.0.sql:9-20](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.0.sql#L9-L20), [amcheck--1.2--1.3.sql:9-21](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.2--1.3.sql#L9-L21), [amcheck--1.3--1.4.sql:11-24](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.3--1.4.sql#L11-L24) |
| Recyclable = new or deleted-and-past-the-horizon | [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829), [bufpage.h#PageIsNew](../../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234) |
| Recyclability uses the most conservative horizon | [procarray.c#GlobalVisHorizonKindForRel](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1966-L1991), [procarray.c#GlobalVisCheckRemovableXid](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L4294-L4306); server: 0 reusable on two VACUUMs, 768 after three xids were consumed |
| Deleted pages keep their data/leaf flags | [ginvacuum.c:187-192](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L187-L192); server flag set `{data,leaf,deleted,compressed}` |
| Flushed pending pages become `deleted`, not stale `list` | [ginfast.c#shiftList](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L630-L635); server: 490 `pending` became 490 `deleted`, flags exactly `{deleted}` |
| Flushed pending pages are immediately reusable | [ginvacuum.c:816-822](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L816-L822); server: `490 currently deleted, 490 reusable` in one VACUUM |
| Free pages are recorded as `BLCKSZ - 1` but read back as 8160 | [indexfsm.c:48-55](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55), [freespace.c:398-435](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L398-L435), [htup_details.h:563](../../../../raw/postgres-17/src/include/access/htup_details.h#L563); server: `avail = 8160` on 768 and 490 pages, max `avail` 8160 over every GIN index |
| Free pages are reused in-index, never truncated | [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335), [ginvacuum.c:796-802](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L796-L802); server: file size never shrank across any VACUUM or flush, and 491 dead pages absorbed 50k rows' pending list for +26 blocks |
| GIN's own slack measure is `pd_upper - pd_lower` | [ginblock.h:287](../../../../raw/postgres-17/src/include/access/ginblock.h#L287), [bufpage.c#PageGetExactFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L952-L973) |
| Entry-page slack over-states usable space by a line pointer | [ginentrypage.c#entryIsEnoughSpace](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L482), [bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923) |
| Version 2 is the right gate for trusting `pd_lower` | [ginblock.h:85-103](../../../../raw/postgres-17/src/include/access/ginblock.h#L85-L103), [ginblock.h:302-309](../../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309), [gindatapage.c:520-528](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L520-L528) |
| Entry tree never loses tuples or pages | [README:389-396](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396), [README:26-30](../../../../raw/postgres-17/src/backend/access/gin/README#L26-L30) |
| Entry pages split in half, so fresh builds are far from full | [ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691); server payload fraction after rebuild: 51.95 / 58.94 / 51.95 / 50.20 / 43.28 / 87.96 / 52.81% |
| Only entirely empty posting-tree pages are deleted | [ginvacuum.c#ginScanToDelete](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L303-L318); server: uniform deletes gave 0 deleted pages and 51.34% slack |
| `n_entry_pages`/`n_data_pages` are a VACUUM-time census | [ginvacuum.c:766-789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L766-L789); server: matched the SQL census on every fixture; `n_total_pages` 268 against 758 real blocks before a VACUUM; `meta_data_pages 77` counted 14 not-yet-recyclable deleted pages as data |
| GIN's `pages_deleted` is per-run, `pages_free` is a census | [ginvacuum.c:234-235](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L234-L235), [ginfast.c:590-591](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L590-L591), [ginvacuum.c:786-794](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L786-L794), [vacuumlazy.c:718-731](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731); server: GIN `0 currently deleted, 768 reusable` beside B-tree `520 currently deleted, 520 reusable` in the same VACUUM |
| All-zero pages return NULL from the GIN functions | [ginfuncs.c:49-50](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L49-L50), [ginfuncs.c:119-120](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L119-L120), [pageinspect/expected/gin.out:57-70](../../../../raw/postgres-17/contrib/pageinspect/expected/gin.out#L57-L70); server: row of NULLs, `pagesize 0` |
| Raw-page functions are superuser-only | [rawpage.c:150-153](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L153), [ginfuncs.c:42-45](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L42-L45), [pageinspect.sgml:10-14](../../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L10-L14); server: `must be superuser to use raw page functions` |
| `pgstatginindex` and `pg_freespace` reach `pg_stat_scan_tables` | [pgstattuple--1.4--1.5.sql:49-57](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57), [pg_freespacemap--1.1--1.2.sql:6-7](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1--1.2.sql#L6-L7); server: both worked for such a role |
| `gin_clean_pending_list` needs ownership | [ginfast.c:1061-1064](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1061-L1064); server: `must be owner of index f6_slack_gin` |
| `pg_freespace` has no other-session-temp guard | [pg_freespacemap.c:24-50](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L50); server: returned rows where the other two refused |
| The scan is page-at-a-time, not a snapshot | [rawpage.c:186-196](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L186-L196) |
| `gin_leafpage_items` needs an exact flag match | [ginfuncs.c:219-226](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L219-L226); server: `Flags 0002, expected 0083` |
| The census reads no ring buffer, unlike pgstattuple | [rawpage.c:181-196](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L181-L196), [pgstattuple.c:544](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L544), [pgstatindex.c:222](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L222); server: cold run read 1262 blocks for a 1263-block index |
| `gin_clean_pending_list` and VACUUM are refused during recovery | [ginfast.c:1037-1041](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1037-L1041); standby: `recovery is in progress` and `cannot execute VACUUM during recovery`, while the census functions answered |
| `gin_pending_list_limit` is session-scoped | [guc_tables.c:3576-3585](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585) |
| `pg_relation_size` with one argument is the main fork | [func.sgml:29627-29643](../../../../raw/postgres-17/doc/src/sgml/func.sgml#L29627-L29643) |
| A flush merges entries and never shrinks the file | [ginfast.c#ginInsertCleanup](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1014-L1020), [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335); server: 6,209,536 -> 6,209,536 bytes with 490 pages turned dead, entry slack 575,940 -> 318,084 |
| The payload/fill model breaks when churn replaces the key set | [README:389-396](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396); server: +19.94% on `f1` and +18.12% on the multicolumn `f14` (keys replaced) against +0.07% on `f7` and +0.00% to +1.49% on the other opclasses |
| One idle snapshot blocks recyclability indefinitely | [ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829), [procarray.c#GlobalVisHorizonKindForRel](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1966-L1991); server: 0 reusable across three VACUUMs and 6 xids with a `REPEATABLE READ` snapshot held at `backend_xmin` 870 = the pages' `prune_xid`, then 768 reusable on the next VACUUM after release with no new xids |
| The self-check does not detect a concurrent writer | [rawpage.c:186-196](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L186-L196); server: `census_total_pages = blocks` held 25 of 25 while `pending_pages` disagreed with `meta_pending_pages` in 19 of 25 and the block count went stale by up to 664 blocks |
| Autoanalyze alone flushes the pending list | [ginvacuum.c:705-717](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717), [guc_tables.c:1449-1457](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457); server: 491 pending pages -> 0 in under 10 s with `last_autovacuum` still null, the file growing 836 -> 958 blocks |
| The census has no ring buffer, a seq scan does | [rawpage.c:181-196](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L181-L196), [heapam.c:434-458](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L434-L458); server at 8,192 buffers: a 6,665-block census left all 6,665 pages resident and later lost 2,942 of them to the next censuses, while a 4,092-block seq scan left 98 |
| Real all-zero pages exist after a crash | [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335), [bufpage.h#PageIsNew](../../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234); server: 4 and 7 all-zero pages after two `-m immediate` crashes during a bulk insert |
| `REINDEX CONCURRENTLY` gives the same size as `REINDEX` | server: 7,356,416 -> 3,948,544 bytes either way on the same fixture recipe, 379 ms against 151 ms |

## Open Questions

1. **The `ginVersion <> 2` path is untested.** v17 always writes
   `GIN_CURRENT_VERSION = 2` ([ginutil.c:355-382](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L355-L382)),
   so no fixture could exercise the suppressed-slack branch or an uncompressed
   posting-tree leaf. `uncompressed_pages` was 0 on every index measured, in both
   runs.
2. **The payload-and-fill model is now known to have a failure mode, and its
   boundary is not mapped.** It predicts the rebuilt size within 3.1% on six of
   seven fixtures and misses `f1` by +19.94% because dead entry tuples count as
   payload. What is not known is how to detect that case from the census alone: a
   count of entry tuples with empty posting lists would do it, but no contrib
   function exposes one, and `gin_leafpage_items` only reads posting-tree leaves.
   The fill fraction is still taken from the rebuild it predicts.
3. **One page size, one alignment, one cluster.** Everything was measured at
   `block_size` 8192, `MAXALIGN` 8, `fsync = off`, on a single 17.11 cluster, at
   150k to 1,000,000 rows. `MaxFSMRequestSize` is 8160 only under those alignment
   and block size assumptions; the FSM cross-check should read
   `current_setting('block_size')`-derived values on unusual builds, and none was
   tested.
4. **Whether the flush grows the file is still not predictable from the census.**
   The same fixture stayed at 758 blocks in one flush, grew by one block in the
   first run's, and grew 836 -> 958 blocks in the autoanalyze flush. The deciding
   factor is whether the FSM holds free pages when the merge needs them — the
   pending list itself consumes them first — which the census cannot see before
   the fact.
5. **The lower bound has a 0.13-point margin, and no theory says it must hold at
   all.** `f13_btgin_gin` read 62.71% dead against 62.84% reclaimed, and
   `f2_pending_gin` already breaks the bound outright. What is missing is a rule
   that says when dead pages *will* be given back by a rebuild; the twelve
   fixtures only show that they usually are.
6. **The race measurement is one workload at one index size.** 25 censuses of a
   2,575-to-6,665-block index against a single-writer insert-and-flush loop showed
   mixed-instant readings in 19 of 25, but nothing was measured for a concurrent
   `VACUUM`, a concurrent posting-tree page deletion, or several writers, and no
   bound is known on how wrong a single census can be.
7. **`REINDEX CONCURRENTLY` was checked on one shape.** It returned the identical
   byte count on the `f6`/`f9` recipe; a fixture whose rebuild is sensitive to the
   extra transactions and waits (for example one still receiving inserts during the
   rebuild) was not tried.
8. **The opclass sweep is one fixture each, all at 200k rows.** `jsonb_path_ops`,
   `gin_trgm_ops`, `btree_gin`, multicolumn and partial were each built once,
   churned once, and rebuilt once; `jsonb_ops`, `array_ops` on other element types,
   `tsvector` with weights, and `INCLUDE`-less multicolumn variants beyond the one
   tried were not measured.
9. **The eviction measurement used a 64 MB cache.** Eviction was demonstrated
   (6,665 resident pages down to 3,723), but nothing shows what a census costs a
   production workload with a large cache and a working set that matters — the
   victim pages here were the census's own.
10. **The crash test produced 4 and 7 all-zero pages, not a rule.** Both attempts
    used the same 600k-row insert and the same 4-second kill point, so the count is
    incidental; what governs how many zeroed blocks survive a crash was not
    investigated, and no case was found where a zero page appeared without a crash.
11. **Cross-run reproducibility is established for one fixture family, not all.**
    `f4`, `f5` and `f6` reproduced the first run's figures byte for byte, and the
    `f5`/`f6` recipe has now built the same 898-block index four times. `f1`, `f2`,
    `f3`, `f7` and everything from `f8` on are this run only, because the first
    run's corpora were never published.

## Source References

- [contrib/pgstattuple/pgstattuple.c](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L236-L308) — `pgstat_relation`'s relkind/AM dispatch and the GIN refusal.
- [contrib/pgstattuple/pgstattuple.c:544](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L544) — `BAS_BULKREAD` for index scans.
- [contrib/pgstattuple/pgstatindex.c](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L484-L577) — `pgstatginindex` and its metapage-only read.
- [contrib/pgstattuple/pgstattuple--1.4--1.5.sql](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57) — the `pg_stat_scan_tables` grant for `pgstatginindex(regclass)`.
- [contrib/pageinspect/rawpage.c](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L234) — `get_raw_page_internal` and `get_page_from_raw`.
- [contrib/pageinspect/ginfuncs.c](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L285) — all three GIN inspection functions.
- [contrib/pageinspect/pageinspect--1.9--1.10.sql](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L10-L21) — the `page_header` signature in use.
- [contrib/pageinspect/sql/gin.sql](../../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L35-L39) and [contrib/pageinspect/expected/gin.out](../../../../raw/postgres-17/contrib/pageinspect/expected/gin.out#L57-L70) — the all-zero-page expectations.
- [contrib/pg_freespacemap/pg_freespacemap.c](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L18-L50) — `pg_freespace`.
- [contrib/pg_freespacemap/pg_freespacemap--1.1.sql](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap--1.1.sql#L6-L25) — both SQL forms and the initial revokes.
- [contrib/amcheck/amcheck--1.0.sql](../../../../raw/postgres-17/contrib/amcheck/amcheck--1.0.sql#L9-L24) — the only verifiers amcheck ships for indexes.
- [src/include/access/ginblock.h](../../../../raw/postgres-17/src/include/access/ginblock.h#L29-L138) — opaque data, flags, metapage, delete xid.
- [src/include/access/ginblock.h](../../../../raw/postgres-17/src/include/access/ginblock.h#L242-L314) — `GinMaxItemSize`, posting-tree layout, `GinDataLeafPageGetFreeSpace`, the `pd_lower` trust note, `GinDataPageSetDataSize`.
- [src/backend/access/gin/ginvacuum.c](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L126-L338) — `ginDeletePage` and `ginScanToDelete`.
- [src/backend/access/gin/ginvacuum.c](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L694-L829) — `ginvacuumcleanup` and `GinPageIsRecyclable`.
- [src/backend/access/gin/ginfast.c](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L547-L671) — `shiftList`.
- [src/backend/access/gin/ginfast.c](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1014-L1091) — the end of `ginInsertCleanup` and `gin_clean_pending_list`.
- [src/backend/access/gin/ginutil.c](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L382) — `GinNewBuffer`, `GinInitPage`, `GinInitMetabuffer`.
- [src/backend/access/gin/ginentrypage.c](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L458-L692) — `entryIsEnoughSpace` and `entrySplitPage`.
- [src/backend/access/gin/gindatapage.c](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L396-L528) — `pd_lower` maintenance and the compressed-only free-space rule.
- [src/backend/access/gin/README](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L412) — the page-deletion design.
- [src/backend/storage/freespace/indexfsm.c](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L74) — the index FSM's used/unused convention.
- [src/backend/storage/freespace/freespace.c](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L36-L66) — the FSM category table and `MaxFSMRequestSize`.
- [src/backend/storage/page/bufpage.c](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L973) — `PageGetFreeSpace` and `PageGetExactFreeSpace`.
- [src/backend/access/heap/vacuumlazy.c](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731) — the per-index VACUUM VERBOSE line.
- [src/backend/access/heap/heapam.c](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L434-L458) — the `NBuffers / 4` seq-scan ring-buffer rule the census does not have.
- [src/backend/storage/ipc/procarray.c](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1966-L1991) — horizon selection when the relation is NULL.
- [src/backend/utils/misc/guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3576-L3585) — `gin_pending_list_limit`.
- [src/backend/utils/misc/guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1449-L1457) — `autovacuum` is `PGC_SIGHUP`, so a reload turns the pending-list flusher on.
- [doc/src/sgml/gin.sgml](../../../../raw/postgres-17/doc/src/sgml/gin.sgml#L500-L537) — GIN fast update.
- [doc/src/sgml/pgstattuple.sgml](../../../../raw/postgres-17/doc/src/sgml/pgstattuple.sgml#L298-L350) — `pgstatginindex` output columns.
- [doc/src/sgml/pageinspect.sgml](../../../../raw/postgres-17/doc/src/sgml/pageinspect.sgml#L634-L712) — the documented GIN functions.
- [doc/src/sgml/pgfreespacemap.sgml](../../../../raw/postgres-17/doc/src/sgml/pgfreespacemap.sgml#L61-L71) — what the FSM means for indexes.
- [doc/src/sgml/func.sgml](../../../../raw/postgres-17/doc/src/sgml/func.sgml#L30104-L30123) — `gin_clean_pending_list`.

## Navigation

- [v17/index](../../index.md)
- [wiki index](../../../index.md)
- [versions](../../../versions.md)
- [PostgreSQL 17 Codebase Navigation Guide (unverified)](../../codebase-navigation-guide.md)
- [Detecting Inflated Non-B-Tree Indexes From Catalogs and a COMMENT-Stored Baseline in PostgreSQL 17 (unverified)](non-btree-index-inflation-comment-baseline.md)
- [B-Tree Bloat and Wasted Space From pgstatindex Alone, on PostgreSQL 12 and 17 (unverified)](btree-bloat-with-pgstatindex.md)
- [PostgreSQL 17 Contrib Extensions (unverified)](../server-administration/contrib-extensions.md)
