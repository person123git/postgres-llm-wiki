---
type: question
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib Extensions on PostgreSQL 17 (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Plan review](#plan-review)
  - [Revised plan](#revised-plan)
  - [Deviations from the brief, and why](#deviations-from-the-brief-and-why)
  - [Why no single contrib function suffices](#why-no-single-contrib-function-suffices)
  - [The three waste classes, from GIN's own definitions](#the-three-waste-classes-from-gins-own-definitions)
  - [The procedure](#the-procedure)
  - [The census statement](#the-census-statement)
  - [The bloat percentage column](#the-bloat-percentage-column)
  - [Four cross-checks](#four-cross-checks)
  - [The fixtures](#the-fixtures)
  - [What the census meant against REINDEX](#what-the-census-meant-against-reindex)
  - [The failure boundary is a straight line](#the-failure-boundary-is-a-straight-line)
  - [Counting entry tuples, and what that fixes](#counting-entry-tuples-and-what-that-fixes)
  - [Whole-page waste is not a lower bound](#whole-page-waste-is-not-a-lower-bound)
  - [Entry-page slack is growth room, not waste](#entry-page-slack-is-growth-room-not-waste)
  - [The pending-list lifecycle, measured end to end](#the-pending-list-lifecycle-measured-end-to-end)
  - [Why a flush sometimes grows the file](#why-a-flush-sometimes-grows-the-file)
  - [Deleted pages need the horizon to move before they count](#deleted-pages-need-the-horizon-to-move-before-they-count)
  - [Concurrency: a census of a busy index is a mixed-instant reading](#concurrency-a-census-of-a-busy-index-is-a-mixed-instant-reading)
  - [Cost of the census](#cost-of-the-census)
  - [Privileges](#privileges)
  - [Refusals, silent answers, and other traps](#refusals-silent-answers-and-other-traps)
  - [Timeouts and GUC scope](#timeouts-and-guc-scope)
  - [Running all four statements on PostgreSQL 12](#running-all-four-statements-on-postgresql-12)
  - [What 12.2 does with an all-zero page](#what-122-does-with-an-all-zero-page)
  - [The corpus on both majors: 26 of 27 fixtures byte-identical](#the-corpus-on-both-majors-26-of-27-fixtures-byte-identical)
  - [Two more bound failures, and what they mean](#two-more-bound-failures-and-what-they-mean)
  - [A rebuild is not one number: maintenance_work_mem moves it](#a-rebuild-is-not-one-number-maintenance_work_mem-moves-it)
  - [The VACUUM VERBOSE cross-check reads differently on 12.2](#the-vacuum-verbose-cross-check-reads-differently-on-122)
  - [Refusals that differ between the majors](#refusals-that-differ-between-the-majors)
  - [What is identical on both majors](#what-is-identical-on-both-majors)
  - [Concurrency on both majors, and a correction to the detector ranking](#concurrency-on-both-majors-and-a-correction-to-the-detector-ranking)
  - [Why round five splits the entry tree](#why-round-five-splits-the-entry-tree)
  - [Eviction, re-measured on both majors](#eviction-re-measured-on-both-majors)
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
checkout (119 distinct file-and-range citations at review time, 121 when it
finished; the open-questions pass below took that to 141 over 43 files),
rebuilt PostgreSQL 17.11 from `raw/postgres-17/` because the original sandbox had
been deleted, republished the fixtures as SQL so the numbers are reproducible,
re-ran every measurement, and added a physical standby to test the one refusal the
first run could not reach. See [The fixtures](#the-fixtures) for what reproduced
and what did not.

Second follow-up prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md, in PostgreSQL 17, for the question: Measuring Wasted and
> Reclaimable Bytes in a GIN Index With Contrib Extensions on PostgreSQL 17
> (unverified). Review the open questions.

The original read `follow agents.md , in postgresql 17 , for question : ... ,
review open questions`: `agents.md` for AGENTS.md, lowercase `postgresql`, a space
before each comma and before the colon, and no sentence capitalisation. The asker
chose to attack the open questions with measurements rather than audit them on
paper, and ruled out the two expensive probes — no second build at a different
`BLCKSZ`, and no repeated crash runs.

What this pass did, on the same pin and the then-retained `.wiki-runtime/tmp/ginw2/`
sandbox: **three of the eleven open questions closed, six narrowed, two untouched
by request**. The page's whole fixture corpus was rebuilt from the published SQL in
two virgin databases and reproduced byte for byte; a five-point churn sweep mapped
the payload model's failure boundary to a straight line; an entry-tuple probe built
out of `pd_lower` was found to measure dead keys exactly; a second index was found
that breaks the waste-is-a-lower-bound claim; the flush-growth question got a
mechanism out of the FSM's upper levels; a concurrent VACUUM was shown to defeat
every cross-check the page recommends; and the eviction claim was corrected — a
census does not evict a hot working set, it strips one round of its usage count.
Fourteen more fixtures were scored, twenty new citations were added, and all 141
of the page's file-and-range citations were re-checked for in-bounds line ranges.

Third follow-up prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md, in PostgreSQL 17, for the question: Measuring Wasted and
> Reclaimable Bytes in a GIN Index With Contrib Extensions on PostgreSQL 17
> (unverified). Add a follow-up: make sure that statement runs on v12, and run all
> tests on v12 and v17.

The original read `follow agents.md , in postgresql 17 , for question : ... , add
follow up: make sure that statement run on v12 , run all test on v12 and v17`:
`agents.md` for AGENTS.md, lowercase `postgresql`, a space before each comma and
before the colon, `for question :` for `for the question:`, `add follow up:` for
`add a follow-up:`, `that statement run` for `that statement runs`, and `all test`
for `all tests`. Three scoping answers were taken before drafting: **all four**
published statements had to run on 12.2 (not just the census), the **whole** corpus
had to be re-run on both majors rather than a portable subset, and both sandboxes
were to be deleted once the page was filed.

What this pass did: built 12.2 and 17.11 from this repo's two pins by out-of-tree
builds, ported all four statements (**three edits**, one of which is not cosmetic),
and re-ran the entire measurement programme on both servers — 27 scored fixtures,
the pending-list lifecycle, the three-VACUUM horizon sequence, a held
`REPEATABLE READ` snapshot, four concurrency cases, plain and concurrent rebuilds
with and without load, privileges, refusals, timeouts, a physical standby, crash
and hand-zeroed pages, `pg_buffercache` eviction, and the cost measurements.
**26 of the 27 fixtures came out byte-identical on the two majors**, and so did every
lifecycle, horizon, flush, privilege, timeout and standby reading. The differences
are concentrated in three places: what `pageinspect` does with an all-zero page,
which refusals exist, and what a rebuild costs when the build spills. Two of the
page's headline claims were corrected on evidence — `waste + slack` is not an upper
bound when the dead-key population is large, and a census *does* evict a hot working
set once the target exceeds the cache — and open question 7 now has a mechanism.

Fourth follow-up prompt, corrected and restated with the asker's agreement:

> Follow AGENTS.md, in PostgreSQL 17, for the question: Measuring Wasted and
> Reclaimable Bytes in a GIN Index With Contrib Extensions on PostgreSQL 17
> (unverified). Add to the census statement a bloat percentage column based on
> information the statement already calculates.

The original read `follow agents.md, in postgresql 17, for question: ... , add to
the census statement a bloat percentage column based on the already calculated
information on the statement`: `agents.md` for AGENTS.md, lowercase `postgresql`,
`for question:` for `for the question:`, a space before the comma after the page
title, no sentence capitalisation, one comma splice where a second sentence belongs,
and `based on the already calculated information on the statement` for `based on
information the statement already calculates`; the restatement also adds the
`(unverified)` suffix the page title carries. Three scoping answers were taken
before drafting: the column reports **`waste + slack`** — the quantity the scoring
tables below already print as `waste+slack %` — rather than the honest-signal subset
or the everything-but-payload form; it had to be **verified on 12.2 as well as
17.11**; and both sandboxes were to be deleted once the page was filed.

What this pass did: added exactly one column, `bloat_pct`, as a four-line hunk in
the census statement's `SELECT` list and nothing else. The column was scored against
`REINDEX INDEX` on the seven published fixtures, rebuilt from this page's own
fixture SQL on a 17.11 server and on a 12.2 server; it reproduces the `waste+slack %`
column of the scoring table exactly, is **identical on both majors** for all seven,
and costs nothing (`EXPLAIN (ANALYZE, BUFFERS)` reads `shared hit=7568` with and
without it). It also required two of the page's own rules to be rewritten rather
than quietly contradicted — "never publish their sum as bloat" was one of them —
because a healthy index reads 48% to 50% on this column while a rebuild returns
nothing.

Fifth follow-up prompt, corrected with the asker's agreement:

> Follow AGENTS.md. In PostgreSQL 17, review the plan for the question:
> Measuring Wasted and Reclaimable Bytes in a GIN Index With Contrib Extensions
> on PostgreSQL 17 (unverified).

The correction fixes capitalization, spacing around punctuation, and the missing
articles in the request; the page title is unchanged. The original six-step plan
above remains the record of the brief. The 2026-09-05 review and revised plan are
filed under Answer. This pass reviewed source and the published SQL; it did not
repeat the historical database experiments or re-verify every earlier claim, so
the page's agent verification is `not yet`.

Version scope, stated once: this page may cite only `raw/postgres-17/`, so every
12.2 statement below rests on exact-pin execution against a 12.2 server built from
this repo's v12 pin, plus this checkout's own commit history. No v12 source file is
cited, and none of the 12.2 findings should be read as v12 source analysis.

## Answer

### Short answer

Build a **page census** from `pageinspect`, and report three separate quantities.
They describe page allocation and free gaps; they do not determine how many bytes
a rebuild will return. GIN can retain entry tuples with empty posting lists, while
a rebuild constructs a new index from the heap
([ginvacuum.c:507-558](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L507-L558),
[gininsert.c#ginbuild](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L317-L406)).

| Class | How it is measured | What it means |
|---|---|---|
| Whole-page waste | count of pages whose `gin_page_opaque_info` flags contain `deleted`, plus uninitialized pages, times `block_size` | marked-deleted or uninitialized bytes inside the file; the deleted flag alone does not establish immediate recyclability |
| Live-page slack | `page_header.upper - page_header.lower` summed over live pages, split into entry-tree and posting-tree (data) pages | free bytes inside live pages; on entry pages this is mostly *growth room*, not waste |
| Pending-list bytes | count of pages flagged `list`, times `block_size` | deferred insert work, not waste; flushing it frees pages inside the file and can grow it |

The flags and uninitialized-page result come from `gin_page_opaque_info`;
recycling additionally tests the deletion transaction ID. Live-page gap and
pending-list definitions come from GIN's page layout and cleanup path
([ginfuncs.c#gin_page_opaque_info](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L171),
[ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829),
[ginblock.h:267-287](../../../../raw/postgres-17/src/include/access/ginblock.h#L267-L287),
[ginfast.c:951-987](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L951-L987),
[ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335)).

The statement also prints the first two of those as one derived percentage,
`bloat_pct`, because that sum is the quantity every scoring table below uses. Read
it as a page-accounting percentage, with no guaranteed bound on a rebuild: the
never-churned control reads **48.05** while `REINDEX` returns **nothing**, and an
index on an empty table reads **49.80**. See
[The bloat percentage column](#the-bloat-percentage-column).

Measured on an isolated 17.11 server with `REINDEX INDEX` as ground truth over
**26 fixtures**, `whole_page_waste + live_page_slack` was an **upper bound** on the
bytes a rebuild returned in 26 of 26 cases, and `whole_page_waste` alone was **not**
a lower bound: it failed on 2 of 26, one of them holding 64.64% of its file in dead
pages while `REINDEX` returned only 58.05%, because a fresh GIN build can be less
dense than an aged one. `pgstatginindex` reported `0 / 0` for that same index. The
bound fails exactly when the aged index's in-use core is smaller than its own
rebuild, and both failures were indexes grown entirely through the pending list.

Two later results qualify that. **The upper bound is not unconditional**: on a
second corpus of 27 fixtures run on both 12.2 and 17.11 it failed twice — once on an
index censused with a live pending list, and once on an 800,000-row
`jsonb_path_ops` index carrying 819,770 dead entry tuples, where `waste + slack`
read 43.43% against 50.00% reclaimed. And **the statement needs three edits to run
on PostgreSQL 12**, one of which is not cosmetic: on 12.2 the census silently
classifies an all-zero page as an entry page and reports its waste as zero. See
[Running all four statements on PostgreSQL 12](#running-all-four-statements-on-postgresql-12).

### Plan review

**Keep the census approach, but revise the promises in steps 2 through 5.** The
review below uses PostgreSQL 17 pin `786db8dcf168bd9df8f55047337525ac19118b1c`.
It checks the plan against implementation source and audits the filed statement;
the existing experimental tables are historical results, not new measurements.

1. **Steps 4 and 5: remove the promised bounds on REINDEX savings.** A page gap
   measures available space on that page. The line-pointer deduction in
   `PageGetFreeSpace` says how much room an insertion needs; it establishes no
   bound on the size of a different index built later. VACUUM recreates an entry
   tuple even when its posting list becomes empty. Those retained bytes are
   outside `waste + slack`, and adding pending bytes does not account for them.
   Rebuilds scan the heap and insert accumulated entries in batches controlled by
   `maintenance_work_mem`. Report the census and measured rebuild reduction as
   distinct outputs, and report failed bound hypotheses as failures
   ([bufpage.c#PageGetFreeSpace](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L923),
   [ginvacuum.c:507-558](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L507-L558),
   [gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314),
   [gininsert.c#ginbuild](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L378-L406)).

2. **Step 2: distinguish marked-deleted bytes from recyclable bytes.**
   `GinPageIsRecyclable` accepts an uninitialized page immediately. For a deleted
   page, it accepts an invalid deletion transaction ID or requires
   `GlobalVisCheckRemovableXid(NULL, delete_xid)` to succeed. This uses the most
   conservative visibility horizon, the boundary needed to protect older scans.
   The census does not evaluate that predicate. Its `whole_page_waste_bytes`
   therefore measures a page class, not the number of bytes immediately available
   for reuse. `GinNewBuffer` rechecks eligibility even after the free space map
   supplies a candidate
   ([ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829),
   [procarray.c#GlobalVisHorizonKindForRel](../../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1966-L1991),
   [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335)).

3. **Step 3: require completed index cleanup before using VACUUM's statistics.**
   `gin_clean_pending_list` calls `ginInsertCleanup`; it does not traverse the
   entire index or refresh `n_entry_pages` and `n_data_pages`. That refresh is in
   `ginvacuumcleanup`. A table VACUUM can skip this callback when index cleanup is
   disabled, including through the table's `vacuum_index_cleanup` option, or when
   the wraparound failsafe disables it. Thus neither a pending-list flush nor a
   successful VACUUM command alone establishes refreshed page counts. Record the
   maintenance outcome and any failsafe warning before interpreting cross-checks
   ([ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1031-L1091),
   [ginvacuum.c#ginvacuumcleanup](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L694-L802),
   [vacuum.c:2155-2178](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2178),
   [vacuumlazy.c:392-401](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L401),
   [vacuumlazy.c:1064-1066](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066),
   [vacuumlazy.c:2323-2335](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2335)).

4. **Step 3: treat consistency checks as diagnostics.** `get_raw_page` opens the
   named relation, locks and copies one buffer, then releases both locks before
   returning. Sharing its result between decoders gives one page image; it does
   not give an index-wide snapshot. The filed `census_total_pages = blocks` is
   principally an accounting identity: `generate_series` uses the captured size
   and the `CASE` assigns every generated row a class. It cannot prove stable
   contents or correct classification. The revised plan needs a controlled
   measurement interval for exact comparisons, and an explicit mixed-time status
   for an online scan. Size and metapage agreement alone cannot certify it
   ([rawpage.c#get_raw_page_internal](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L198),
   [ginvacuum.c:754-789](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L754-L789)).

5. **Step 3: correct the free-space-map cross-check and its equality condition.**
   The FSM records `BLCKSZ - 1` but decodes its highest category as
   `MaxHeapTupleSize`, or `BLCKSZ - MAXALIGN(SizeOfPageHeaderData + sizeof(ItemIdData))`.
   This gives 8160 for an 8192-byte block and 8-byte alignment, not 8191. Derive
   the value using the build's block size and alignment. Compare counts only after
   accounting for deletion horizons, maintenance completion, and intervening
   reuse. A nonrecyclable deleted data page still enters VACUUM's data-page count;
   the SQL puts it in the deleted bucket. In the v17 verbose line, `pages_free`
   appears as **reusable**
   ([indexfsm.c#RecordFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55),
   [freespace.c:398-435](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L398-L435),
   [htup_details.h#MaxHeapTupleSize](../../../../raw/postgres-17/src/include/access/htup_details.h#L563),
   [pg_controldata.c#pg_control_init](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L203-L228),
   [ginvacuum.c:766-794](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L766-L794),
   [vacuumlazy.c:718-731](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731)).

6. **Steps 2 and 5: make unsupported-format and malformed-page handling explicit.**
   The SQL gates the combined slack fields and `bloat_pct` on version 2, but still
   prints `entry_slack`, `data_slack`, and the derived `payload_bytes` without that
   gate. Suppress or qualify every affected value. Also, `PageIsNew` tests only
   `pd_upper == 0`; a NULL GIN decoder result is not a byte-for-byte zero-page
   check. `gin_page_opaque_info` emits unknown flag bits as hexadecimal text, and
   the SQL's final `ELSE 'entry'` has no unknown-class rejection. Define explicit
   diagnostic outcomes for unknown flags, invalid headers, and a missing or
   invalid metapage before trusting numeric results. These are inspection limits,
   not a substitute for a GIN structural verifier
   ([ginblock.h:302-309](../../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309),
   [bufpage.h#PageIsNew](../../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234),
   [ginfuncs.c#gin_metapage_info](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95),
   [ginfuncs.c#gin_page_opaque_info](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L98-L171),
   [amcheck/Makefile](../../../../raw/postgres-17/contrib/amcheck/Makefile#L3-L13)).

7. **Step 4: name the measured size precisely.** A before/after comparison of
   `pg_relation_size(index, 'main')` measures the reduction in the main fork's
   file lengths. The implementation sums `stat.st_size` for that fork's segments;
   it does not measure filesystem allocation, other forks, or the rebuild's peak
   disk demand. Keep this denominator throughout the experiment, record signed
   size changes, and record the rebuild settings and data state
   ([dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L343),
   [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L370),
   [index.c:3781-3789](../../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)).

**Retain step 1 and the privilege split in step 5.** The GIN rejection in
`pgstattuple`, the three-field metapage-only result from `pgstatginindex`, and the
superuser requirement in raw-page readers are correctly identified. The
`pg_stat_scan_tables` grant in pgstattuple 1.5 does not satisfy the raw readers'
own superuser checks. Keep the `OFFSET 0` page-read boundary: the planner rejects
pull-up of a subquery with an offset or a volatile target list
([pgstattuple.c#pgstat_relation](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L260-L296),
[pgstatindex.c#pgstatginindex_internal](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L507-L576),
[pgstattuple--1.4--1.5.sql:49-57](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57),
[rawpage.c:150-173](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L173),
[prepjointree.c:1689-1701](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1689-L1701),
[prepjointree.c:1772-1781](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1772-L1781)).

### Revised plan

This is the plan for the next implementation and validation pass. The source
findings above are reviewed; the additional query guards and experiments below
are not implemented by this review.

1. **Specify the outputs and scope.** Select one physical GIN index and record
   its identity, definition, main-fork size, block size, GIN format version, and
   installed extension versions. Define marked-deleted/uninitialized bytes,
   entry-page gaps, posting-tree gaps, pending bytes, and measured main-fork
   reduction separately. Keep `bloat_pct` as the existing accounting formula
   without promising a reclaimable percentage or either bound. See findings 1,
   2, and 7.
2. **Choose the measurement conditions before maintenance.** Use a controlled
   interval for the calibration experiment, covering writers, VACUUM, pending
   cleanup, and index replacement. For an online report, record that page images
   can come from different instants. Capture the pre-maintenance state; then
   record whether only pending cleanup or full index vacuum/cleanup completed.
   Allow deletion-horizon mismatches to remain unresolved instead of treating a
   fixed number of VACUUM runs as sufficient. See findings 2 through 5.
3. **Implement explicit result validity.** Preserve the single raw-page read and
   decoder sharing. Derive all block-size arithmetic, including the size bracket
   that currently divides by 8192. Add explicit handling for unsupported versions,
   untrusted slack-derived fields, unknown classes, metapage failures, and target
   changes. Keep page accounting, FSM state, and metapage comparisons as separate
   diagnostics. See findings 4 through 6.
4. **Validate ordinary and adversarial cases on the exact pin.** Run the final
   statement extracted from this page on an isolated build. Include fresh and
   empty controls, posting-tree deletions with a held old snapshot, key-replacement
   churn, a live pending list, cleanup-only versus full VACUUM, and disabled index
   cleanup. Test reader privileges, invalid and temporary targets, malformed or
   uninitialized page inputs, and unsupported-format handling. Exercise concurrent
   writers and maintenance separately. Require diagnostic or withheld output for
   unsupported cases. The shipped `pageinspect` tests cover basic GIN decoding,
   error inputs, and all-zero pages; the `pgstattuple` tests cover metapage reads
   and wrong access methods. They do not validate this combined report or a
   REINDEX-savings bound
   ([pageinspect/sql/gin.sql](../../../../raw/postgres-17/contrib/pageinspect/sql/gin.sql#L1-L41),
   [pgstattuple/sql/pgstattuple.sql](../../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L47-L63)).
5. **Score rebuild comparisons without assuming their result.** Rebuild each
   calibration fixture with its definition and input data held constant. Record
   `maintenance_work_mem`, the before/after main-fork byte counts, and prediction
   error for each candidate metric. Include several build-memory budgets and the
   dead-key and pending-list cases that challenge the proposed bounds. GIN's
   accumulator flush and entry-page split rules make those relevant variables
   ([gininsert.c#ginBuildCallback](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L314),
   [ginentrypage.c:667-691](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L667-L691)).
6. **Apply the operational and filing checks to the tested text.** Retain
   `statement_timeout = '10min'` and `lock_timeout = '2s'` as starting session
   limits for the full scan, and apply them in each session running an operational
   statement. Both settings, and any experimental `maintenance_work_mem` change,
   use session/transaction scope and need no reload or restart
   ([guc_tables.c:2611-2631](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2631),
   [guc_tables.c:2466-2474](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2474)).
   Preserve the `/* wiki_... */` tags, capture the exact tested SQL, and file
   unexecuted paths under Open Questions. Update this page's claim map, Contents,
   navigation summaries, and log; run wiki lint. Change agent verification only
   after a full claim review, and leave human verification to the user.

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
   a value the macro cannot produce on this build. **Do not hardcode 8160
   either** — derive it, as [Four cross-checks](#four-cross-checks) now does,
   from `pg_control_init()`, which publishes both terms of the macro
   ([pg_controldata.c#pg_control_init](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L203-L228),
   [func.sgml#pg_control_init](../../../../raw/postgres-17/doc/src/sgml/func.sgml#L27721-L27742)).

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
   ([gindatapage.c#dataBeginPlaceToPageLeaf](../../../../raw/postgres-17/src/backend/access/gin/gindatapage.c#L520-L528)).

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

**Whole-page waste.** A GIN page is reusable when `PageIsNew` is true, or flagged
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
census includes gap bytes that a future insertion would spend on line pointers.
This distinguishes physical gaps from insertion capacity; it establishes no
upper bound on what a rebuild returns.

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
   For this step, require completed index cleanup: `INDEX_CLEANUP OFF` disables
   the callback, and the failsafe can disable it even when cleanup was requested
   ([vacuumlazy.c:392-401](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L401),
   [vacuumlazy.c:1064-1066](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066),
   [vacuumlazy.c:2323-2335](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2335)).
3. **Record the denominator and the metapage.** `pg_relation_size(idx, 'main')`
   is the main fork only
   ([func.sgml#pg_relation_size](../../../../raw/postgres-17/doc/src/sgml/func.sgml#L29627-L29643)), which is
   the right denominator because the census counts main-fork blocks;
   `gin_metapage_info(get_raw_page(idx, 0))` supplies `version` and the pending
   counters ([ginfuncs.c#gin_metapage_info](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L95)).
4. **Census blocks 1 .. relpages-1**, reading each page once and passing the same
   `bytea` to `gin_page_opaque_info` and `page_header`.
5. **Cross-check four ways** before believing any number, and run the entry-tuple
   probe too if you intend to predict a rebuild.
6. **Only then** compare with a rebuild, if you are deciding whether to rebuild.

### The census statement

This is the previously tested statement. The [plan review](#plan-review) records
its interpretation limits and proposed guards; those query changes remain
unimplemented. The timeout settings are session-scoped, and the tags are the
`wiki_` markers this repo requires.

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
               WHEN h.pagesize = 0                      THEN 'new'   -- all-zero page
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
         CROSS JOIN LATERAL (SELECT get_raw_page(g.idx_name, b.blkno::int) AS pg
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
       CASE WHEN m.version = 2
            THEN round(100.0 * ((c.deleted_pages + c.new_pages) * m.bs
                                + c.entry_slack + c.data_slack)
                       / nullif(m.main_bytes, 0), 2) END AS bloat_pct,
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

Two things in that text are there so the same statement runs on PostgreSQL 12 as
well as 17: the `h.pagesize = 0` arm and the `b.blkno::int` cast. Both are no-ops on
17.11 — the edited and unedited texts returned identical output on every index where
both were run there, including one carrying two all-zero pages, because the
`pagesize = 0` arm and the `flags IS NULL` arm fire on exactly the same pages — and
both are load-bearing on 12.2. See
[Running all four statements on PostgreSQL 12](#running-all-four-statements-on-postgresql-12).

Design points that are not cosmetic:

- **`OFFSET 0` keeps the page read single.** `get_raw_page` is volatile and the
  subquery is not flattened, so the page is fetched once and the same value feeds
  both LATERAL calls. Measured with `EXPLAIN (ANALYZE, BUFFERS)` on a 1235-block
  index: this form reports **1234** shared hits, the naive form that calls
  `get_raw_page` inside each function reports **2468** — exactly twice, one read
  per function call.
- **A NULL result means `PageIsNew`, which tests `pd_upper == 0`.** This is not
  a check that every byte is zero
  ([bufpage.h#PageIsNew](../../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234)).
  Both GIN functions return NULL early on that predicate
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
- **`census_total_pages = blocks` checks accounting only.** The generated block
  range and exhaustive `CASE` make every scanned row count toward the total,
  including rows assigned by the fallback `ELSE`. Equality does not prove correct
  classification or an unchanged index; the raw reader locks one page at a time
  ([rawpage.c#get_raw_page_internal](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L198)).
  It held on all 8 selected GIN indexes in the historical sandbox. Entry-tree
  *internal* pages are the class that makes the
  `ELSE` arm necessary: their flags word is 0, so `gin_page_opaque_info` returns
  an empty array rather than a NULL row, and each of the three rebuilt `tsvector`
  fixtures held exactly 6 of them beside 1193 entry leaves.

### The bloat percentage column

`bloat_pct` is the added derived column — dead pages plus live-page
slack, over the main fork:

```text
bloat_pct = 100 * ((deleted_pages + new_pages) * block_size
                   + entry_slack + data_slack) / main_fork_bytes
```

Every term is already in the census, so the column reads no extra page:
`EXPLAIN (ANALYZE, BUFFERS)` over the seven-fixture database reported
`shared hit=7568` for the text without it and `shared hit=7568` for the text with
it. It is gated on `gin_version = 2` exactly like the slack columns, because
`pd_lower` — which the slack term is built from — is only trustworthy in a
version-2 index
([ginblock.h#GinMetaPageData](../../../../raw/postgres-17/src/include/access/ginblock.h#L85-L103),
[ginblock.h:302-309](../../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309)). On a
version-1 index it returns NULL. The SQL still exposes `entry_slack`,
`data_slack`, and `payload_bytes` without the version guard; do not interpret the
untrusted components as measured payload or space. The revised plan calls for
explicit handling of all affected fields. Page counts and pending bytes remain
separate outputs.

It reports the quantity the scoring tables below print as `waste+slack %`, and it
scores the way they do. The seven published fixtures, censused and then rebuilt with
`REINDEX INDEX`:

| Fixture | `bloat_pct` | `whole_page_waste_pct` | REINDEX reclaimed % | over-read |
|---|---|---|---|---|
| `f1_churn_gin` | 64.12 | 0.65 | 42.42 | +21.70 |
| `f2_pending_gin` | 75.30 | 64.64 | 58.05 | +17.25 |
| `f3_fresh_gin` | 48.05 | 0.00 | 0.00 | **+48.05** |
| `f4_empty_gin` | 49.80 | 0.00 | 0.00 | **+49.80** |
| `f5_deleted_gin` | 95.22 | 85.52 | 89.09 | +6.13 |
| `f6_slack_gin` | 51.34 | 0.00 | 46.33 | +5.01 |
| `f7_reupdate_gin` | 54.15 | 9.89 | 13.26 | +40.89 |

**Both majors print the same column.** The amended text runs on 12.2 with no further
edit — the `::int` cast and the `pagesize = 0` arm it already carries are what it
needs there, see
[Running all four statements on PostgreSQL 12](#running-all-four-statements-on-postgresql-12)
— and over these seven fixtures **178 of 182 cells came out identical** on the two
servers. The four that differ are `f7`'s slack bytes, the 128-and-10-byte difference
already reported in
[The corpus on both majors](#the-corpus-on-both-majors-26-of-27-fixtures-byte-identical);
they round away, so `bloat_pct` is identical on all seven. The 12.2 rebuilds
returned the same seven byte counts as the 17.11 rebuilds, so the reclaimed column
above is the same on both too.

Four things the column does not mean:

1. **It is not reclaimable space, and on a healthy index it is not close.**
   `f3_fresh_gin` was never churned: it reads **48.05** and `REINDEX` returns
   **0 bytes**. `f4_empty_gin` indexes an empty table — one metapage and one entry
   page — and reads **49.80**, because that entry page is 8,160 bytes empty. The
   floor is structural rather than accidental: `entrySplitPage` splits a full entry
   page by equalizing data size, so a fresh build leaves its pages about half full
   ([ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691)).
   Over-read across these seven fixtures ran from **+5.01 to +49.80 points**.
2. **It is not an unconditional upper bound.** It bounded the truth on all seven
   here and on 26 of 26 in the first corpus, but the second corpus broke it twice —
   43.43% against 50.00% reclaimed on an index carrying 819,770 dead entry tuples,
   which sit inside `payload_bytes` where neither term can see them
   ([README:389-396](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396)). See
   [Two more bound failures, and what they mean](#two-more-bound-failures-and-what-they-mean).
3. **It excludes the pending list**, which is what step 2 of
   [the procedure](#the-procedure) is for. A `fastupdate` index censused with 246
   pending pages read `bloat_pct` **8.66** beside `pending_pct` **81.73**, and
   `REINDEX` then took it from 2,465,792 to 794,624 bytes — **67.77% reclaimed**,
   eight times what the column reported. Byte-identical on 12.2. Adding
   `pending_pct` gives 90.39 here, above this fixture's measured reduction, but
   establishes no general bound: retained empty-key entry tuples remain outside
   that sum
   ([ginvacuum.c:507-558](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L507-L558)). Censused again
   straight after the rebuild, the same index reads **44.96** with nothing left to
   take, which is point 1 restated as a before-and-after pair.
4. **It is not the sum of the two rounded columns.** It rounds the summed bytes
   once, so it can land 0.01 above `whole_page_waste_pct + live_page_slack_pct`:
   `f2_pending_gin` prints 64.64 and 10.65, which add to 75.29, against a
   `bloat_pct` of **75.30**.

The pending-list fixture, so that third reading is reproducible on either major
(`gin_pending_list_limit` is `PGC_USERSET`, so this `SET` is session-scoped and needs
no reload):

```sql
SET gin_pending_list_limit = '1GB';          -- or GIN flushes mid-insert at 4MB
CREATE TABLE p1 (id int primary key, tags int[]);
INSERT INTO p1 SELECT i, ARRAY[i % 1000, (i * 7) % 1000] FROM generate_series(1, 50000) i;
CREATE INDEX p1_gin ON p1 USING gin (tags) WITH (fastupdate = on);
VACUUM p1;
INSERT INTO p1 SELECT i, ARRAY[i % 1000, (i * 7) % 1000]
FROM generate_series(50001, 100000) i;       -- 50k rows left in the pending list
```

So the column earns its place for one job — comparing an index against its own
earlier readings, or against a rebuilt twin — and for no other. A single reading of
a single index says almost nothing, because about half of it is the fill fraction of
any GIN build.

### Four cross-checks

**FSM.** `pg_freespace` reports the recorded value per block
([pg_freespacemap.c#pg_freespace](../../../../raw/postgres-17/contrib/pg_freespacemap/pg_freespacemap.c#L24-L50)),
and for indexes the documentation says the value is meaningful only as
in-use-versus-empty
([pgfreespacemap.sgml:61-71](../../../../raw/postgres-17/doc/src/sgml/pgfreespacemap.sgml#L61-L71)). A free index
page reads `MaxFSMRequestSize`. Its count can match `deleted + new` only when
those deleted pages are recyclable, the free pages have been recorded, and no
intervening allocation or maintenance changes the compared states
([ginvacuum.c:766-794](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L766-L794),
[ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829),
[indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L32-L45)). Derive
the constant rather than typing 8160 — `pg_control_init()` publishes both terms of
`BLCKSZ - MAXALIGN(SizeOfPageHeaderData + sizeof(ItemIdData))`, is executable by
`PUBLIC` (measured: `has_function_privilege('public','pg_control_init()','execute')`
is true), and needs no extension:

```sql
SELECT /* wiki_gin_waste_fsm_check */
       (SELECT database_block_size
               - ((28 + max_data_alignment - 1) / max_data_alignment)
                 * max_data_alignment          -- 28 = SizeOfPageHeaderData + sizeof(ItemIdData)
        FROM pg_control_init())                          AS free_page_avail,
       count(*) FILTER (WHERE avail = (SELECT database_block_size
               - ((28 + max_data_alignment - 1) / max_data_alignment)
                 * max_data_alignment FROM pg_control_init())) AS fsm_free_pages,
       count(*)                                          AS blocks
FROM pg_freespace('myschema.myindex');
```

On the pinned build that expression returns **8160**, equal to the largest `avail`
seen over 3,478 free GIN pages, so the derived form and the measured form agree.
Measured counts: `768` FSM-free pages against `768` census-deleted pages on the
deleted-pages fixture, and `490` against `490` on the flushed pending fixture;
every other GIN index reported no free pages and no deleted pages. Note the
direction of the check — a page can be deleted but not yet recorded free, as it
was immediately after the first VACUUM (768 deleted, 0 in the FSM).

**Size bracket.** Read `pg_relation_size` again after the census and compare it
with the statement's own `blocks` column. In one historical four-writer test,
this check caught the race in **13
of 14** censuses where the metapage cross-check below caught **0 of 14**. See
[Concurrency](#concurrency-a-census-of-a-busy-index-is-a-mixed-instant-reading).

```sql
SELECT /* wiki_gin_waste_size_bracket */ pg_relation_size('myschema.myindex','main') / 8192
       AS blocks_after_census;
```

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

Fourteen more fixtures were added by the open-questions pass. The five `g`
fixtures are the churn sweep, and they are the only ones whose recipe is not a
variation on the seven above, because they have to hold the *volume* of churn
constant while varying the share of the key population that dies. Keys are owned
by groups of four consecutive ids, so rewriting a contiguous id prefix kills
exactly that prefix's keys, and every row is updated in every variant:

```sql
-- g0 / g25 / g50 / g75 / g100: p% of the key population replaced, 100% of rows updated
CREATE TABLE s100 (id int primary key, doc text);
INSERT INTO s100 SELECT i, 'w' || (i/4) || ' x' || (i/4) || ' hot' || (i % 7) || ' tog0'
FROM generate_series(1, 200000) i;
CREATE INDEX g100_gin ON s100 USING gin (to_tsvector('simple', doc))
       WITH (fastupdate = off);
-- threshold t = 200000 * p / 100; below it new keys, above it the same keys again
UPDATE s100 SET doc = 'v'||(id/4)||' y'||(id/4)||' warm'||(id%7)||' tog1' WHERE id <= 200000;
UPDATE s100 SET doc = 'w'||(id/4)||' x'||(id/4)||' hot' ||(id%7)||' tog1' WHERE id >  200000;
VACUUM s100; VACUUM s100;
```

| Fixture | Recipe |
|---|---|
| `fh1_gin` | `int[]` over 1,000 keys, built at 50k rows then grown by 5 x 50k rows **entirely through the pending list**, flushed each round |
| `fh2_gin` | the same growth pattern over a 50k-key `tsvector` space — the second lower-bound violation |
| `fh3_gin` | the same growth pattern over 97 keys only, so every entry becomes a posting tree |
| `fg_flush_gin` | posting-tree-dominated `int[]` (6 keys mod 5000/97, 300k rows), `fastupdate = on`, five identical 50k-row insert-and-flush rounds |
| `fa_race_gin` | the `f5` shape at 600k rows, censused while its first VACUUM ran |
| `fb_race_gin`, `fz_race_gin` | 600k-row `fastupdate` indexes censused under four concurrent writers |
| `fw1`-`fw4_gin` | four identical `int[]` fixtures rebuilt by `REINDEX` and `REINDEX CONCURRENTLY`, with and without an insert stream |
| `k1_gin` | 200k `jsonb` rows, `USING gin (doc jsonb_ops)` — the default opclass, which indexes keys *and* values |
| `k2_gin` | 200k `text[]` rows, `USING gin (ws)` (`array_ops`) |
| `k3_gin` | 200k rows, `USING gin ((setweight(to_tsvector('simple', doc), 'A')))` |
| `k4_gin`, `k5_gin` | `jsonb_path_ops` at **50,000** and **800,000** rows, the scale check |
| `tn_null` / `fn_null_gin` | 20k rows mixing NULL arrays, empty arrays and NULL elements, for the entry-tuple identity |

`k1` through `k5` were churned so that every key changes, then VACUUMed twice, and
scored exactly like `f11`-`f15`.

Provenance, and the reproducibility answer. These fixtures were first rebuilt for
the 2026-08-26 review because the original sandbox had been deleted and its fixture
SQL was never published. The open-questions pass then re-ran the **published**
fixture SQL above, unmodified, in two freshly created databases on the same
cluster. All seven came out byte-identical to the filed figures — the same sizes
(17,571,840 / 6,209,536 / 10,117,120 / 16,384 / 7,356,416 / 7,356,416 /
11,927,552 bytes), the same page-class counts, the same `entry_slack` and
`data_slack` to the byte, the same `f2` lifecycle states (2,195,456 -> 6,209,536 ->
6,209,536 with `pgstatginindex` reading `2 | 490 | 50000`), the same three-VACUUM
`f5` sequence including its B-tree sibling line, and the same seven `REINDEX`
results (42.42 / 58.05 / 0.00 / 0.00 / 89.09 / 46.33 / 13.26 percent). The churn
sweep also re-ran identically, all five sizes and all five model errors. The one
number that moved is the one the page already said would: `f5`'s `prune_xid` read
**953** in the new database against 791 and 870 in the two earlier runs, because it
is whatever `ReadNextTransactionId()` returned during the deleting VACUUM. The
`f5`/`f6` recipe has now built the same 7,356,416-byte, 898-block index **six**
times.

### What the census meant against REINDEX

`REINDEX INDEX` before/after `pg_relation_size` is the ground truth for
filesystem-reclaimable bytes. The `waste+slack %` column in every table below is
what the statement prints as `bloat_pct`; see
[The bloat percentage column](#the-bloat-percentage-column).

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

Both bounds hold on all five. Four more opclasses and shapes were scored the same
way by the open-questions pass, including the same opclass at two scales:

| Fixture | churned bytes | waste % | slack % | waste+slack % | REINDEX reclaimed % | fresh-build fill % | dead entry tuples |
|---|---|---|---|---|---|---|---|
| `k1_gin` (`jsonb_ops`) | 12,009,472 | 36.83 | 36.66 | 73.49 | 57.57 | 61.83 | 2 |
| `k2_gin` (`text[]` `array_ops`) | 14,204,928 | 26.53 | 47.54 | 74.07 | 60.73 | 51.09 | 40,119 |
| `k3_gin` (weighted `tsvector`) | 21,307,392 | 17.03 | 50.80 | 67.84 | 55.09 | 50.16 | 100,139 |
| `k4_gin` (`jsonb_path_ops`, 50k rows) | 3,874,816 | 52.01 | 26.83 | 78.84 | 68.92 | 51.12 | 10,006 |
| `k5_gin` (`jsonb_path_ops`, 800k rows) | 32,407,552 | 8.19 | 51.71 | 59.90 | 41.05 | 50.92 | 160,034 |

Counting the churn sweep, the three pending-list-grown fixtures and the
flush-rounds fixture, **26 fixtures** have now been scored against `REINDEX`:
`waste + slack` bounded the truth from above **26 of 26**, and `waste` alone
bounded it from below **24 of 26**. The two failures are `f2_pending_gin` and
`fh2_gin`, and they share a mechanism; see
[Whole-page waste is not a lower bound](#whole-page-waste-is-not-a-lower-bound).
`f13_btgin_gin` remains the closest call on the right side of the line: 62.71%
dead against 62.84% reclaimed, a margin of 0.13 points, which is **one block**.
`fh1_gin` is the second-closest at 0.67 points, or six blocks.

Fresh-build fill is clearly opclass-dependent — 50.16% for a weighted `tsvector`
and 51.30% for `btree_gin` against 72.11% for `pg_trgm`, with `f10_race_gin`'s
six-key `int[]` at 1,000,000 rows reading 68.98% and `jsonb_ops` 61.83% — so "how
full is a fresh GIN build" has no single answer to compare an aged index against.
It is, however, **scale-insensitive**: `k4` and `k5` are the same opclass and the
same churn pattern 16x apart in row count, and they read 51.12% and 50.92% fill
with model errors of +33.18% and +33.60%.

`REINDEX INDEX CONCURRENTLY` is the same ground truth as the plain form, on three
shapes including one under load. `f9_ric_gin`, built by the identical recipe to
`f6_slack_gin`, went from 7,356,416 bytes to **3,948,544** — the same byte count
plain `REINDEX` gave `f6` — in 379 ms against 151 ms, leaving
`indisvalid`/`indisready`/`indislive` all true and no `_ccnew`/`_ccold` leftovers,
and a plain `REINDEX` immediately afterwards returned the same 3,948,544. The
open-questions pass then ran the case that matters operationally: **an index taking
inserts while it is rebuilt**. Two identical `int[]` fixtures at 3,276,800 bytes
each took the same 100,000-row insert stream, one rebuilt with `REINDEX` and one
with `REINDEX CONCURRENTLY`:

| Rebuild | elapsed | before | after | rows after |
|---|---|---|---|---|
| `REINDEX INDEX` under load | 1901 ms | 3,276,800 | **2,662,400** | 200,000 |
| `REINDEX INDEX CONCURRENTLY` under load | 2013 ms | 3,276,800 | **2,662,400** | 200,000 |
| `REINDEX INDEX`, no load | — | 3,276,800 | **1,548,288** | 100,000 |
| `REINDEX INDEX CONCURRENTLY`, no load | — | 3,276,800 | **1,548,288** | 100,000 |

Identical byte counts in both conditions, and both indexes ended
`indisvalid`/`indisready`/`indislive` true. The loaded rebuild is legitimately
larger than the quiet one because it indexes 100,000 more live rows. So either form
is usable as ground truth; the concurrent form only costs elapsed time.

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
not of the opclass. One caveat still keeps this out of predictor territory: the
fill fraction comes from the rebuild it is predicting.

### The failure boundary is a straight line

The five-point churn sweep holds the churn volume constant — all 200,000 rows are
updated in every variant — and varies only the share `p` of the key population
that is replaced. There is no threshold and no safe region except `p` near zero:

| Fixture | keys replaced | churned bytes | payload/fill prediction | error | dead entry tuples |
|---|---|---|---|---|---|
| `g0_gin` | 0% | 8,323,072 | 7,756,432 | **+0.09%** | 1 |
| `g25_gin` | 25% | 10,067,968 | 8,727,825 | **+12.62%** | 25,001 |
| `g50_gin` | 50% | 11,894,784 | 9,771,313 | **+25.16%** | 50,001 |
| `g75_gin` | 75% | 13,615,104 | 10,670,309 | **+37.69%** | 75,001 |
| `g100_gin` | 100% | 15,261,696 | 11,639,344 | **+50.19%** | 100,010 |

The increments are +12.53, +12.54, +12.53 and +12.50 points per 25 points of `p`,
so on this shape the model over-predicts by very close to **half a point per
percent of the key population replaced**. The whole sweep re-ran with identical
sizes and identical errors.

### Counting entry tuples, and what that fixes

The census can detect the failure case after all, because `pd_lower` counts line
pointers. GIN entry pages are ordinary item-pointer pages — `entryPreparePage` and
`entrySplitPage` place tuples with `PageAddItem`
([ginentrypage.c:561-568](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L561-L568),
[ginentrypage.c:683-691](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L683-L691)) — so
on an entry page `pd_lower` is `SizeOfPageHeaderData + nitems * sizeof(ItemIdData)`
([bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L214)),
and `page_header` exposes it. Entry **leaf** pages carry flags exactly `{leaf}` and
entry-tree internal pages exactly `{}`, which separates the two:

```sql
SET /* wiki_gin_entry_probe_guards */ statement_timeout = '10min';
SET /* wiki_gin_entry_probe_guards */ lock_timeout = '2s';

WITH /* wiki_gin_entry_probe */ pages AS (
    SELECT o.flags, h.lower, h.upper, h.special, h.pagesize
    FROM generate_series(1, pg_relation_size('myschema.myindex','main') / 8192 - 1) AS b(blkno)
         CROSS JOIN LATERAL (SELECT get_raw_page('myschema.myindex', b.blkno::int) AS pg
                             OFFSET 0) AS r
         LEFT JOIN LATERAL gin_page_opaque_info(r.pg) AS o ON true
         LEFT JOIN LATERAL page_header(r.pg)          AS h ON true
)
SELECT count(*)                 FILTER (WHERE flags = '{leaf}') AS entry_leaf_pages,
       sum((lower - 24) / 4)    FILTER (WHERE flags = '{leaf}') AS entry_leaf_tuples,
       sum(special - upper)     FILTER (WHERE flags = '{leaf}') AS entry_leaf_tuple_bytes,
       sum((lower - 24) / 4)    FILTER (WHERE flags = '{}'
                                        AND pagesize > 0)       AS internal_downlinks
FROM pages;
```

The `pagesize > 0` guard on the last column is the same portability fix as in the
census, and it matters more here: on 12.2 an all-zero page reports `flags = '{}'`
and contributes `(0 - 24) / 4 = -6` downlinks each. With two zeroed blocks present,
the unguarded form read **41** downlinks on 12.2 against **53** on 17.11 for the same
index; the guarded form reads 53 on both.

`entry_leaf_tuples` is the number of keys the index holds, live or dead, and the
identity is exact on every shape tried:

| Index | `entry_leaf_tuples` | distinct keys in the table |
|---|---|---|
| `f3_fresh_gin` (`tsvector`) | 50,028 | 50,028 lexemes |
| `f6_slack_gin` (`int[]`) | 32 | 32 tags |
| `fn_null_gin` (NULLs and empty arrays) | 53 | 50 tags, plus 3 |

The `+3` on the last row is GIN's null bookkeeping: a null key, a null item and an
empty item each get their own entry
([ginblock.h#GinNullCategory](../../../../raw/postgres-17/src/include/access/ginblock.h#L204-L213)), and all three
categories occur in that fixture. The internal-downlink count is a free structural
check — `f3_fresh_gin` reads 1,198 downlinks over 6 internal pages against 1,193
leaves, which is a root of 5 plus 1,193.

Subtract the live key count and you have the dead-key population that the payload
model trips over. It separates the two churn shapes exactly:

| Index | entry tuples before | after `REINDEX` | dead | model error |
|---|---|---|---|---|
| `f1_churn_gin` (term set replaced) | 100,056 | 50,028 | **50,028** | +19.94% |
| `f7_reupdate_gin` (same terms rewritten) | 50,030 | 50,029 | **1** | +0.07% |
| `k1_gin` (`jsonb_ops`, values recur) | 20,016 | 20,014 | **2** | +0.79% |
| `k3_gin` (weighted `tsvector`) | 200,278 | 100,139 | **100,139** | +42.79% |

What the probe does **not** buy is a corrected prediction. Subtracting the dead
tuples at the average entry-tuple size over-corrects, because a dead key's tuple is
just the key with an empty posting list while a live key's carries its TIDs: on the
sweep the average tuple falls from 32.0 bytes at `p = 0` to 24.0 bytes at
`p = 100`, which puts the dead tuples at 16 bytes against the live 32. The
corrected estimate therefore lands **under** the truth by −7.48%, −12.42%, −15.89%
and −18.44% across the sweep, and by −36.80% on `f1`. The pair is still useful,
because the two estimates **bracket** the rebuilt size on all nine fixtures with a
material dead population (10,006 dead tuples and up), while on the eleven with
none or a handful — `f2`-`f6` and `fh1`-`fh3` at zero, `f7` and `g0` at one, `k1`
at two — the correction is a rounding of the plain estimate and the plain model's
±3.07% band is what applies. Cost: the probe ran a whole 26,195-block database in
146-241 ms and twice produced byte-identical output, and the live-key count it has
to be compared against is a full table scan — 758 ms for one 200,000-row table.

### Whole-page waste is not a lower bound

`f2_pending_gin` is the counterexample. After flushing its pending list it held
490 dead pages out of 758 — 64.64% of the file, confirmed independently by the
FSM — yet `REINDEX` returned only 58.05%. The rebuild is *bigger* than the
in-use part of the aged index: 268 in-use blocks became 318 blocks after
`REINDEX`, 18.7% larger.

**The rule is an identity, not a heuristic.** Reclaimed bytes are
`size - rebuilt`, so `waste <= reclaimed` holds exactly when
`rebuilt <= size - waste`: dead pages are a lower bound on what a rebuild returns
if and only if **the aged index's in-use core is at least as big as its own
rebuild**. With no dead keys in play that reduces to a density comparison — the
aged core's fill against a fresh build's fill — and it gets *easier* to satisfy the
more of the key population is dead, because then the rebuild has less to store.
That predicted the direction on the three pending-list-grown fixtures, including
the sign of a six-block near miss:

| Fixture | waste % | reclaimed % | in-use bytes | rebuilt bytes | aged core fill % | fresh fill % | lower bound |
|---|---|---|---|---|---|---|---|
| `fh1_gin` | 54.69 | 55.36 | 3,325,952 | 3,276,800 | 68.71 | 69.62 | holds by 6 blocks |
| `fh2_gin` | **21.66** | **19.25** | 10,903,552 | 11,239,424 | 53.18 | 51.61 | **fails** |
| `fh3_gin` | 43.65 | 47.94 | 2,580,480 | 2,383,872 | 34.30 | 36.85 | holds |

So `fh2_gin` is the second measured violation, and it was built to be one: all
three fixtures were grown entirely through the pending list, which is the mechanism
`f2` exposed. A merge appends into whichever entry page already holds the key,
packing it, while a fresh build splits pages in half. Only the one whose aged core
ended up denser than its rebuild broke the bound.

`entrySplitPage` divides a full page by equalizing data size, halving it
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

That is why the statement reports `entry_slack` and `data_slack` separately, and why
`bloat_pct`, which sums them with the dead pages, has to be read beside that split
rather than instead of it. On an entry-tree-dominated index most of that single
number is growth room: `f3_fresh_gin`'s 48.05 is 4,796,424 bytes of entry slack
against 65,192 of data slack, and a rebuild returns none of it.

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

### Why a flush sometimes grows the file

Two rules govern it, and the first one is exact. Five identical rounds — the same
50,000-row insert, then `gin_clean_pending_list()` — on a posting-tree-dominated
`fastupdate` index:

| Round | free pages when the pending list was built | file growth building it | free pages at flush time | file growth flushing | what the merge added |
|---|---|---|---|---|---|
| 1 | 0 | **+736** (= the 736 pending pages) | 0 | +0 | nothing |
| 2 | 736 | **+0** | 0 | **+194** | 194 data-leaf pages |
| 3 | 736 | **+0** | 0 | +0 | nothing |
| 4 | 736 | **+0** | 0 | +15 | 15 entry pages |
| 5 | 736 | **+0** | 0 | **+649** | 649 entry pages |

An earlier run of the same recipe that used `VACUUM` instead of
`gin_clean_pending_list()` for round 4's flush produced the identical +15 blocks and
the identical 834 entry pages, so the flush path does not change the arithmetic.

**Rule one: the pending list is built out of the FSM's free stock, and the merge
never gets any of it.** Round 1 had nothing to take and extended the file by exactly
the number of pending pages it needed. Rounds 2 to 5 each started with 736 free
pages and built the same pending list for **zero** growth — and every one of them
then reached the flush with `pg_freespace` reporting **0** free pages again.

**Rule two: a flush cannot reuse the pages it is itself freeing.**
`RecordFreeIndexPage` is `RecordPageWithFreeSpace`
([indexfsm.c:48-55](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L48-L55)), which
updates only the bottom-level FSM page and says so: "the space might not become
visible to searchers until the next `FreeSpaceMapVacuum` call, which updates the
upper level pages"
([freespace.c#RecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L186-L204)).
The searcher is the other half of `GinNewBuffer`
([ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L328)):
`GetFreeIndexPage` asks for half a block
([indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L37-L46)),
`GetPageWithFreeSpace` hands that to `fsm_search`
([freespace.c#GetPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L136-L142)),
and `fsm_search` starts at `FSM_ROOT_ADDRESS`
([freespace.c#fsm_search](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L684-L691)) — the
level that only an FSM vacuum maintains. `ginInsertCleanup` calls
`IndexFreeSpaceMapVacuum` after the whole merge is done
([ginfast.c:1014-1020](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1014-L1020)). So the pages
`shiftList` frees mid-flush are invisible to the merge that is running.

What is left unpredictable is how many pages the merge itself wants, and total
slack does not answer it: round 2 grew by 194 blocks with 2,326,056 bytes of slack
on hand while round 3 grew by nothing with 3,197,144, and round 5 grew by 649
blocks with 1,885,966. Slack is bound to individual pages — an entry page with too
little room for *its own* key's new TIDs splits no matter how much free space other
pages have.

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
they agree even when that reading is already stale.

**Three more concurrency cases, and the detector ranking they produce.** Each
census below was bracketed by a `pg_relation_size` reading taken in a separate
statement before and after.

| Case | What the census saw | `census_total_pages = blocks` | metapage check | size bracket |
|---|---|---|---|---|
| A: one VACUUM deleting posting-tree pages | dead pages read 0, 0, 0, 74, 222, 444, 666, 888, 1110, 1406, 1628, 1850, 2072, 2294 over 14 censuses of an unchanging 2,594-block file (final truth: 2,368) | held **14 of 14** | held 14 of 14 | held **14 of 14** |
| B: four concurrent writers | block count stale by up to 134 blocks | held **14 of 14** | caught **0 of 14** | caught **13 of 14** |
| D: concurrent `REINDEX INDEX CONCURRENTLY` | ten censuses at 2,594 blocks, then the eleventh silently read the swapped-in 162-block index | held 11 of 11 | held 11 of 11 | held 11 of 11 |

Case A is the worst case on this page and it defeats everything. A VACUUM deletes
posting-tree pages in place, so the file never changes size, `pending_pages` stays
0 on both sides of the cross-check, and the census is free to report **any**
intermediate dead-page count — here anything from 0% to 91% of the file — with
every check passing. Two consecutive censuses disagreeing is the only signal.

Case B reverses the earlier ranking. The metapage cross-check caught 19 of 25 in
the insert-and-flush test above, because that writer kept moving the metapage; a
writer that only inserts leaves `pending_pages` and `meta_pending_pages` agreeing
while the file grows underneath. **Read the size again after the census** — that is
the check to run, and it is the cheapest of the three.

Case D is a trap rather than a wrong number: `get_raw_page` resolves the index name
on every call and holds no lock between calls, so a concurrent rebuild's swap can
land mid-census. It landed between censuses in all eleven attempts here, but a swap
landing after the size read and before a page read would put the block number past
the end of the new, smaller file — the documented
`block number ... is out of range` error.

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

**But it does not evict a genuinely hot working set, and that is worth knowing
before you refuse to run it.** Re-measured at `shared_buffers = 128MB` (16,384
buffers) against two 3,704-page tables — one read eight times, one read once — a
single census of a **26,195-block** database, twice the size of the cache:

| | before the census | after one census | after two |
|---|---|---|---|
| hot table (8 passes) | 3,704 pages at `usagecount = 5` | **3,704 pages**, `usagecount = 1` | **0 pages** |
| table read once | 3,704 pages at `usagecount = 1` | **0 pages** | 0 pages |
| the census's own pages | — | 12,481 resident (7,481 at 0, 5,000 at 1) | — |

The mechanism is the clock sweep. `PinBuffer` raises a buffer's usage count on
every touch, capped at `BM_MAX_USAGE_COUNT`
([bufmgr.c:2700-2705](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2700-L2705),
[buf_internals.h#BM_MAX_USAGE_COUNT](../../../../raw/postgres-17/src/include/storage/buf_internals.h#L72-L79)),
and `StrategyGetBuffer` decrements every non-zero buffer it passes and takes the
first zero it finds
([freelist.c#StrategyGetBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L314-L341)). A
census reads each page once, so its own pages sit at 1 and are the cheapest victims
in the cache. The real cost is therefore not eviction but **the usage count it
spends**: one census cost the hot set four of its five levels of protection, and the
second census, finding it at 1, took all of it. The control behaved as before — a
16,303-block seq scan left 96 pages behind and disturbed the hot set not at all.

So on a production box the census is safe for a working set that is genuinely hot
and repeatedly read, dangerous for one that is read once, and should not be run
back to back over a cache-sized index.

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

### Running all four statements on PostgreSQL 12

All four published statements run on a 12.2 server after **three edits**, two in the
census and one in the entry-tuple probe. The FSM cross-check and the size bracket
need none.

| Statement | Runs unchanged on 12.2? | Edit needed |
|---|---|---|
| census (`wiki_gin_waste_census`) | no | `b.blkno::int`, and a `pagesize = 0` arm ahead of the flags tests |
| FSM check (`wiki_gin_waste_fsm_check`) | **yes** | none — `pg_control_init()` has `database_block_size` and `max_data_alignment` on both, and the derived value is **8160** on both |
| size bracket (`wiki_gin_waste_size_bracket`) | **yes** | none |
| entry-tuple probe (`wiki_gin_entry_probe`) | no | `b.blkno::int`, and `AND pagesize > 0` on the `'{}'` filter |

**Edit 1: the block number is an `int4` on 12.** Without a cast, 12.2 answers

```text
ERROR:  function get_raw_page(text, bigint) does not exist
```

because `generate_series` hands it a `bigint` and `pageinspect` on that server
declares `get_raw_page(text, int4)`. v17 declares the `int8` form — the widening is
`pageinspect--1.8--1.9.sql`, which drops the `int4` signatures and creates `int8`
ones
([pageinspect--1.8--1.9.sql#get_raw_page](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L46-L58)),
committed as `f18aa1b2039` "pageinspect: Change block number arguments to bigint"
(2021-01-19, earliest containing tag `REL_14_0`, no backpatch line), whose message
gives the reason: block numbers are 32-bit *unsigned*, so `bigint` is the smallest
SQL type that holds them. `::int` is the portable spelling: on 17.11 the `int`
argument reaches the `int8` function through the implicit cast, and the cast form
returned the same rows as the filed form on every index where both were run. The
residual limitation is v12's own: a fork past 2^31 blocks cannot be addressed there
at all.

**Edit 2: `flags IS NULL` does not mean "all-zero page" on 12.2.** This is the one
that changes numbers rather than raising an error, and it gets its own section:
[What 12.2 does with an all-zero page](#what-122-does-with-an-all-zero-page).

**Edit 3** is the same cause inside the probe, and is quantified beside the probe
above.

One difference needs no edit at `block_size` 8192. `page_header` returns
`lower`, `upper`, `special` and `pagesize` as `int` on 17.11
([pageinspect--1.9--1.10.sql#page_header](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L10-L21))
and as `smallint` on 12.2 — commit `127404fbe28` "pageinspect: Improve
page_header() for pages of 32kB" (2021-07-12, earliest tag `REL_15_0`). Every value
fits in a signed 16-bit integer at 8 kB pages, `smallint - smallint` stays
`smallint`, and `sum()` promotes to `bigint` either way, so the census arithmetic is
unaffected. At `BLCKSZ` 32768 the 12.2 columns would overflow and the statement
would need casts there; that is untested, and stays in
[Open Questions](#open-questions).

Two constructs the *harness* needed are also v12-relevant, though no published
statement uses them: `pg_current_xact_id()` does not exist on 12.2 (the horizon
sequence used `txid_current()` on both servers), and the reloption
`autovacuum_vacuum_insert_threshold` does not exist there either — see
[What is identical on both majors](#what-is-identical-on-both-majors).

### What 12.2 does with an all-zero page

On 17.11 both GIN inspection functions return early on `PageIsNew`
([ginfuncs.c:49-50](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L49-L50),
[ginfuncs.c:119-120](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L119-L120)). On 12.2
they do not, and the whole difference follows from that. Fed one all-zero `bytea`:

| Call | 17.11 | 12.2 |
|---|---|---|
| `gin_page_opaque_info` | a row of NULLs | a row: `flags = {}`, `rightlink 0`, `maxoff 0` |
| `gin_metapage_info` | a row of NULLs | `ERROR: input page is not a GIN metapage`, `DETAIL: Flags 0000, expected 0008` |
| `page_header` | `lower 0 / upper 0 / special 0 / pagesize 0` | identical |

The NULL behaviour is `cd4868a5700` "pageinspect: Fix handling of all-zero pages"
(2022-04-14), whose message explains that every pageinspect function used to crash
or misread a new page, chooses NULL over an error so a full-relation scan still
returns a batch, and carries `Backpatch-through: 10`. So this is a **minor**-version
boundary, not a major one: in this checkout's history the 12-branch backport is
`5378d55cb2f`, whose earliest `REL_12_*` tag is **`REL_12_11`**, and this repo's v12
pin (12.2) is an ancestor of it. A 12.11-or-later server should behave like 17.11
here; 12.2 does not, and that was not tested.

Measured end to end, by stopping the server and appending two all-zero blocks to a
55-block GIN index file, then restarting:

| Statement | 17.11 | 12.2 |
|---|---|---|
| filed text with only the `::int` cast | 54 entry, **2 new**, `whole_page_waste_bytes` **16,384** | 56 entry, **0 new**, `whole_page_waste_bytes` **0** |
| the portable text (`pagesize = 0` first) | 54 entry, 2 new, 16,384 | identical to 17.11 |

So on 12.2 the filed statement folds zeroed blocks into the entry-page count and
reports their waste as zero — a silent under-count of exactly `new_pages * block_size`,
with no error and with `census_total_pages = blocks` still passing. Everything
*else* about those pages is identical on the two majors: `VACUUM` reported
`2 reusable`, the FSM then showed 2 pages at `avail = 8160` and 55 at 0, and a
following 10,000-row insert reused none of them (the file stayed at 57 blocks),
because the new entries fitted in existing pages' slack.

**A zeroed metapage is worse on 12.2: it aborts the whole report.** Zeroing block 0
of one small GIN index with `dd` gives, on 17.11, a row of NULLs from
`gin_metapage_info` and a census that still lists every index; on 12.2 the same
census dies with `ERROR: input page is not a GIN metapage / DETAIL: Flags 0000,
expected 0008`, so one corrupt index destroys the reading for all of them. On 12.2,
census one index at a time if any index might be damaged.

The page's crash result did **not** reproduce. `pg_ctl stop -m immediate` four
seconds into a 600,000-row insert produced **zero** all-zero pages on both servers
(647 blocks, `new_pages = 0`, twice), where the earlier run got 4 and 7. The
deterministic route to a zero page is the file append above, not a crash.

### The corpus on both majors: 26 of 27 fixtures byte-identical

Every fixture was rebuilt from SQL on both servers and scored against
`REINDEX INDEX` by the same harness: census and probe into tables, then a rebuild of
every GIN index, then census and probe again. The published `f1`-`f7` SQL was run
verbatim; the rest are the recipes printed above, reconstructed as SQL because the
earlier passes published them only as one-line descriptions.

The reconstructions are printed here so the two-major numbers are reproducible. The
churn sweep used the template already published above with the five thresholds
0 / 50000 / 100000 / 150000 / 200000; the rest are:

```sql
-- f11 / k4 / k5: jsonb_path_ops at 200,000 / 50,000 / 800,000 rows
CREATE TABLE t11_json (id int primary key, doc jsonb);
INSERT INTO t11_json SELECT i, jsonb_build_object('k'||(i%997), i%50021, 'tag', 'w'||(i%20011))
FROM generate_series(1,200000) i;
CREATE INDEX f11_json_gin ON t11_json USING gin (doc jsonb_path_ops) WITH (fastupdate=off);
UPDATE t11_json SET doc = jsonb_build_object('m'||(id%997), (id%50021)+100000, 'tag2', 'v'||(id%20011));
VACUUM t11_json; VACUUM t11_json;

-- f12: pg_trgm
CREATE TABLE t12_trgm (id int primary key, txt text);
INSERT INTO t12_trgm SELECT i, 'alpha'||(i%20011)||' beta'||(i%997) FROM generate_series(1,200000) i;
CREATE INDEX f12_trgm_gin ON t12_trgm USING gin (txt gin_trgm_ops) WITH (fastupdate=off);
UPDATE t12_trgm SET txt = 'gamma'||(id%20011)||' delta'||(id%997);

-- f13: btree_gin on int4
CREATE TABLE t13_btgin (id int primary key, n int);
INSERT INTO t13_btgin SELECT i, i%20011 FROM generate_series(1,200000) i;
CREATE INDEX f13_btgin_gin ON t13_btgin USING gin (n) WITH (fastupdate=off);
UPDATE t13_btgin SET n = (id%20011)+1000000;

-- f14: multicolumn, both key columns replaced
CREATE TABLE t14_multi (id int primary key, tags int[], doc text);
INSERT INTO t14_multi SELECT i, ARRAY[i%1000,(i*7)%1000], 'w'||(i%50021)||' hot'||(i%7)
FROM generate_series(1,200000) i;
CREATE INDEX f14_multi_gin ON t14_multi USING gin (tags, to_tsvector('simple', doc))
       WITH (fastupdate=off);
UPDATE t14_multi SET tags = ARRAY[(id%1000)+5000, ((id*7)%1000)+5000],
                     doc  = 'v'||(id%50021)||' warm'||(id%7);

-- f15: partial, predicate column rewritten too
CREATE TABLE t15_partial (id int primary key, tags int[], live boolean);
INSERT INTO t15_partial SELECT i, ARRAY[i%1000,(i*7)%1000,i%97], (i%10=0)
FROM generate_series(1,200000) i;
CREATE INDEX f15_partial_gin ON t15_partial USING gin (tags) WHERE live;
UPDATE t15_partial SET tags = ARRAY[(id%1000)+5000, ((id*7)%1000)+5000, (id%97)+5000],
                       live = (id%10=1);

-- k1: jsonb_ops.  k2: text[] array_ops.  k3: weighted tsvector expression index
CREATE INDEX k1_gin ON tk1 USING gin (doc) WITH (fastupdate=off);            -- jsonb_build_object('k'||(i%97),'v'||(i%997),'n',i%20011)
CREATE INDEX k2_gin ON tk2 USING gin (ws)  WITH (fastupdate=off);            -- ARRAY['w'||(i%20011), 'x'||(i%997)]
CREATE INDEX k3_gin ON tk3 USING gin ((setweight(to_tsvector('simple', doc), 'A')))
       WITH (fastupdate=off);                                               -- 'w'||(i%50021)||' hot'||(i%7)

-- fn_null: 20,000 rows mixing NULL arrays, empty arrays and NULL elements
CREATE TABLE tn_null (id int primary key, tags int[]);
INSERT INTO tn_null SELECT i,
       CASE WHEN i%5 = 0 THEN NULL
            WHEN i%5 = 1 THEN '{}'::int[]
            WHEN i%5 = 2 THEN ARRAY[i%50, NULL]
            ELSE ARRAY[i%50] END
FROM generate_series(1,20000) i;
CREATE INDEX fn_null_gin ON tn_null USING gin (tags) WITH (fastupdate=off);
VACUUM tn_null;

-- fh1 / fh2 / fh3: built at 50k rows, then five 50k rounds through the pending list
SET gin_pending_list_limit = '1GB';
CREATE TABLE h1 (id int primary key, tags int[]);
INSERT INTO h1 SELECT i, ARRAY[i%1000, (i*7)%1000] FROM generate_series(1,50000) i;
CREATE INDEX fh1_gin ON h1 USING gin (tags) WITH (fastupdate=on);
VACUUM h1;
-- then, five times, with the ranges 50001-100000 ... 250001-300000:
--   INSERT INTO h1 SELECT i, ARRAY[i%1000,(i*7)%1000] FROM generate_series(...) i;
--   SELECT gin_clean_pending_list('fh1_gin');
-- fh2 replaces the key with to_tsvector('simple','w'||(i%50021)||' x'||((i*7)%50021)||' hot'||(i%7));
-- fh3 uses ARRAY[i%97,(i*7)%97].  fh1b is fh1 plus a sixth round, flushed, then VACUUMed twice.

-- fg_flush: posting-tree-dominated fastupdate index, five identical insert-and-flush rounds
CREATE TABLE fg (id int primary key, tags int[]);
INSERT INTO fg SELECT i, ARRAY[i%5000,(i*7)%5000,(i*13)%5000,i%97,(i*3)%97,(i*11)%97]
FROM generate_series(1,300000) i;
CREATE INDEX fg_flush_gin ON fg USING gin (tags) WITH (fastupdate=on);
VACUUM fg;
-- then five rounds of 50,000 rows each, flushed with gin_clean_pending_list()

-- fw1-fw4: four identical fixtures for the rebuild comparison
CREATE TABLE w1 (id int primary key, tags int[]);
INSERT INTO w1 SELECT i, ARRAY[i%1000,(i*7)%1000,(i*13)%1000,i%97] FROM generate_series(1,100000) i;
CREATE INDEX fw1_gin ON w1 USING gin (tags) WITH (fastupdate=off);
```

**The seven published fixtures reproduced the filed figures byte for byte on
17.11 — and 12.2 returned the same numbers.** All seven sizes (17,571,840 /
6,209,536 / 10,117,120 / 16,384 / 7,356,416 / 7,356,416 / 11,927,552), every
page-class count, `entry_slack` and `data_slack`, and all seven `REINDEX` results
(42.42 / 58.05 / 0.00 / 0.00 / 89.09 / 46.33 / 13.26%). The `f1`-into-`f3` identity
held again: `f1` rebuilt to exactly its untouched twin's 10,117,120 bytes. The only
difference across the seven is `f7`, where 12.2 reads `entry_slack` 4,804,740 and
`data_slack` 474,746 against 17.11's 4,804,868 and 474,756 — 128 and 10 bytes, on an
identical 11,927,552-byte, 1456-block file with identical page classes, and a model
prediction that moves from 10,354,059 to 10,354,074.

Over all 27 scored fixtures — the seven published ones, five opclass/shape fixtures,
five `k` opclass fixtures, the five-point churn sweep, `fn_null`, and the four
pending-list fixtures — **26 produced byte-identical rows on the two servers** across
every column the harness prints: size, rebuilt size, waste, slack, pending,
reclaimed, both bound verdicts, fresh fill, model error and dead entry tuples. The
bound tallies are therefore identical too:

| Rule | 17.11 | 12.2 |
|---|---|---|
| `waste <= reclaimed` (lower bound) | 24 of 27 | 24 of 27 |
| `waste + slack >= reclaimed` (upper bound) | 25 of 27 | 25 of 27 |
| `waste + slack + pending >= reclaimed` | 26 of 27 | 26 of 27 |

The lower bound failed on the same three fixtures on both: `f2_pending_gin` (64.64%
dead against 58.05% reclaimed), `fg_flush_gin` (35.96 against 33.85) and `fh2_gin`
(33.36 against **−1.54** — the rebuild came out *bigger*, 9,035,776 to 9,175,040
bytes). A fourth, `fh1b_gin`, scored separately, fails it as well at 51.04 against
42.74. Every one satisfies the page's identity: the aged in-use core is smaller than
its own rebuild.

The one fixture that differs between the majors is `k5_gin`, 800,000 `jsonb` rows
under `jsonb_path_ops`:

| | 17.11 | 12.2 |
|---|---|---|
| churned size | 74,661,888 | 69,214,208 |
| `REINDEX` result | 37,330,944 | 34,611,200 |
| waste / slack % | 0.00 / 43.43 | 0.00 / 39.03 |
| reclaimed % | 50.00 | 49.99 |
| fresh fill % | 68.59 | 73.93 |
| dead entry tuples | 819,770 | 819,770 |

Same heap (128,753,664 bytes, 800,000 rows, 35,555,416 bytes of `jsonb` on both),
same dead-key population, same verdicts — a 7.9% larger index on 17.11 at both ends.
That difference is not the opclass or the churn; it is the build, and it is measured
in
[A rebuild is not one number](#a-rebuild-is-not-one-number-maintenance_work_mem-moves-it).

### Two more bound failures, and what they mean

The upper bound failed twice, identically on both majors, and the two failures have
different causes.

**`fh1_gin`: a live pending list.** Censused with 246 pending pages still unflushed,
it reported `waste 0.00 / slack 13.73` against **40.00%** reclaimed, because
`REINDEX` rebuilds from the heap and therefore returns the pending pages too, while
the census deliberately excludes them from waste. The pending share was 53.48% of
the file, so `waste + slack + pending = 67.21%` bounds the truth comfortably. This is
an effect of skipping step 2 of the procedure: **pending bytes must be reported
beside the other quantities.** Adding them repaired the inequality for this
fixture, not for every index. The same recipe with the last round flushed and
the table VACUUMed — `fh1b_gin`, 3,948,544 bytes, 246 deleted pages, waste 51.04,
slack 12.62 — rebuilds to 2,260,992 (42.74% reclaimed) and satisfies the upper bound
with 63.66%.

**`k5_gin`: dead entry tuples counted as payload.** There is no pending list here, so
the pending term does not rescue it: `waste + slack` read 43.43% against 50.00%
reclaimed on 17.11 and 39.03 against 49.99 on 12.2. The cause is the mechanism this
page already documents for the payload model — GIN's entry tree never deletes a
tuple
([README:389-396](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396)) — reaching the
*bound*: the probe counts 1,639,711 entry tuples before the rebuild and 819,941
after, so 819,770 dead tuples at the measured ~16 bytes each are roughly 13.1 MB, or
17.6% of the file, sitting inside `payload_bytes` where neither `waste` nor `slack`
can see them. Add that term and the bound holds again (43.43 + 17.6 >= 50.00).

These results establish **no universal upper bound**, including for an index with
a settled pending list. The entry-tuple probe can help diagnose retained keys,
but a tuple count is not their byte size. The source retains empty-key entries
and rebuilds a new index from the heap; it supplies no identity equating a sum of
old-page gaps with the new file's size
([ginvacuum.c:507-558](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L507-L558),
[gininsert.c#ginbuild](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L317-L406)).
Treat the repaired inequality above as an observation about that fixture.

### A rebuild is not one number: maintenance_work_mem moves it

`REINDEX` is the page's ground truth, and it is not a constant. Rebuilding one index
— `k5_gin`, serially, everything else fixed — across thirteen settings on 17.11 and
sixteen on 12.2, from 4MB to 1GB:

| `maintenance_work_mem` | 12.2 | 17.11 |
|---|---|---|
| 4MB | 34,734,080 | 35,930,112 |
| 16MB | 35,127,296 | 37,765,120 |
| **64MB (default)** | **34,611,200** | **37,330,944** |
| 72MB | 39,256,064 | 42,156,032 |
| 80MB | 43,941,888 | 47,112,192 |
| 88MB | 48,390,144 | 50,814,976 |
| 96MB and up | 50,814,976 | 50,814,976 |

96, 128, 160, 192, 224MB, 256MB and 1GB all return exactly 50,814,976 on both
servers. So the
same index, same data, same server, rebuilds to anything from 34.6 MB to 50.8 MB
depending only on the rebuild's memory budget — **47% apart** — and the largest
budget produces the *largest* index.

The census explains it. `k5_gin` is a pure entry tree (`data_leaf_pages = 0`), and
across builds the payload barely moves while the slack does everything:

| Build | blocks | payload | `entry_slack` | fill % |
|---|---|---|---|---|
| 12.2 at 64MB | 4225 | 25,587,976 | 9,023,224 | 73.93 |
| 17.11 at 64MB | 4557 | 25,605,240 | 11,725,704 | 68.59 |
| both at 96MB | 6203 | 25,690,832 | 25,124,144 | 50.56 |

A build that flushes its accumulator once leaves every entry page split in half —
which is `entrySplitPage` equalizing data size
([ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691)) —
and a build that flushes several times spends later flushes refilling the slack the
earlier ones left. It is the same "an aged index can be denser than its rebuild"
effect the page describes, happening *inside* one build.

The flush point is explicit in the build path: `ginBuildCallback` dumps the whole
accumulator to the index when it has "maxed out our available memory"
([gininsert.c:290-291](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L291)), and the
quantity compared against `maintenance_work_mem` is `BuildAccumulator.allocatedMemory`
([gin_private.h:434](../../../../raw/postgres-17/src/include/access/gin_private.h#L434)), which is summed
with `GetMemoryChunkSpace`
([ginbulk.c:139](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L139),
[ginbulk.c:183](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L183)) — a function documented as
returning the space a chunk occupies *including all memory-allocation overhead*
([mcxt.c#GetMemoryChunkSpace](../../../../raw/postgres-17/src/backend/utils/mmgr/mcxt.c#L718-L730)). Allocator
overhead is therefore inside the budget, so the same data reaches the threshold at
different points on the two majors. The direction and the size were measured: v12 at
68MB (36,986,880) and 70MB (38,019,072) bracket v17 at 64MB (37,330,944), so **17.11
fits about 7% more accumulator entries into the same setting** and behaves like a
slightly larger budget. Parallelism is not involved — `max_parallel_maintenance_workers`
of 0, 2 and 4 all return the same bytes on 17.11 — and above the single-flush
threshold the two majors produce byte-identical builds. This checkout's history
offers two candidate causes in the allocator, `c6e0fe1f2a0` "Improve performance of
and reduce overheads of memory management" (earliest tag `REL_16_0`) and
`2c2eb0d6b27` "Shrink memory contexts struct sizes" (`REL_17_0`); which one moves
this number was not established, and is filed as an open question.

Operational consequence, and it applies to every table on this page: a
"reclaimable bytes" figure has to name the `maintenance_work_mem` it was measured
at, because both the `REINDEX` result and the `fresh_fill_pct` that the payload
model divides by are functions of it — 50.56% to 73.93% fill for this one index.
`maintenance_work_mem` is `PGC_USERSET`
([guc_tables.c:2466-2474](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2474)), so it
takes effect in session or transaction scope with no reload or restart; every number
on this page was taken at the 64MB default.

### The VACUUM VERBOSE cross-check reads differently on 12.2

The numbers agree; the message does not. For the same `f5` three-VACUUM sequence:

| | 17.11 | 12.2 |
|---|---|---|
| VACUUM 1 | `index "f5_deleted_gin": pages: 898 in total, 768 newly deleted, 768 currently deleted, 0 reusable` | `INFO: index "f5_deleted_gin" now contains 10000 row versions in 898 pages` / `DETAIL: 6080000 index row versions were removed.` / `768 index pages have been deleted, 0 are currently reusable.` |
| VACUUM 2 | `898 in total, 0 newly deleted, 0 currently deleted, 0 reusable` | `0 index pages have been deleted, 0 are currently reusable.` |
| VACUUM 3 (after three xids) | `898 in total, 0 newly deleted, 0 currently deleted, 768 reusable` | `0 index pages have been deleted, 768 are currently reusable.` |
| FSM free pages after each | 0 / 0 / 768 | 0 / 0 / 768 |

Three consequences for the cross-check. On 12.2 it lives in a `DETAIL` block rather
than a one-line index summary, so it has to be parsed differently. 12.2 has no
`newly deleted` versus `currently deleted` split at all — it prints one
"have been deleted" number, which is `pages_deleted`, the per-run counter this page
warns is not a census
([ginvacuum.c:234-235](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L234-L235),
[ginfast.c:590-591](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L590-L591)) — so the
"reusable" number is the only census-like figure available there
([ginvacuum.c:786-794](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L786-L794)). And 12.2
still prints an index row-version count that 17.11 dropped, which is the heap tuple
count, not an entry count. The B-tree sibling line differs in the same experiment by
one: on the deleting VACUUM 12.2 reported `519 index pages have been deleted` where
17.11 reported `520 newly deleted, 520 currently deleted`.

### Refusals that differ between the majors

Everything here was reproduced on both servers.

| Case | 17.11 | 12.2 |
|---|---|---|
| `pgstatginindex` on an **invalid** GIN index | `ERROR: index "..." is not valid` | **returns a row**: `2 \| 0 \| 0` |
| `pgstattuple` on an invalid GIN index | `ERROR: index "..." is not valid` | `ERROR: "..." (gin index) is not supported` — no validity check, so it falls through to the AM refusal |
| `pgstattuple` on a valid GIN index | `ERROR: index "f5_deleted_gin" (gin index) is not supported` | same without the leading word `index` |
| `get_raw_page` on a partitioned index | `ERROR: cannot get raw page from relation "..."` + `DETAIL: This operation is not supported for partitioned indexes.` | `ERROR: cannot get raw page from partitioned index "..."`, no DETAIL |
| a `bytea` that is not one block long | `ERROR: invalid page size` + `DETAIL: Expected 8192 bytes, got 2.` | `ERROR: input page too small (2 bytes)` |
| `gin_metapage_info` on an all-zero page | a row of NULLs | `ERROR: input page is not a GIN metapage` |

The invalid-index refusal is the one with operational weight, and it is v17-only:
commit `13503eb5905` "Diagnose !indisvalid in more SQL functions." (2023-10-30,
earliest containing tag `REL_17_0`) added the check that this page cites
([pgstatindex.c:538-543](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L538-L543),
[pgstattuple.c:263-267](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L263-L267)). On 12.2 a
failed `CREATE INDEX CONCURRENTLY` leftover therefore *answers* `pgstatginindex`
with a metapage reading rather than refusing — a two-block index reporting
`2 | 0 | 0` — which is a number that looks healthy and means nothing.

Identical on both majors: `pgstatginindex` on a non-GIN index
(`relation "..." is not a GIN index`) and on a partitioned index (the same message,
because `IS_INDEX` tests `relkind = 'i'`), `pg_freespace` on a partitioned index
(0 rows, no error), `get_raw_page` past the end
(`block number 100000 is out of range for relation "..."`), `gin_leafpage_items` on
an entry leaf (`Flags 0000, expected 0083`), both other-session-temp refusals
(`cannot access temporary indexes of other sessions`,
`cannot access temporary tables of other sessions`) with `pg_freespace` answering
anyway (2 rows), `gin_metapage_info` on a non-meta page, and
`gin_clean_pending_list` returning 0 on an invalid index. The census's own filters
behaved identically too: it excluded the invalid index and the other session's temp
index on both servers, and `gin_metapage_info(get_raw_page(...))` read `version 2`
on the invalid index on both.

### What is identical on both majors

Numbers below are 17.11 then 12.2 where they differ at all.

- **Privileges.** A role holding only `pg_stat_scan_tables` and schema `USAGE` got
  `pgstatginindex` `2 | 0 | 0`, `pg_freespace` all 12 rows, `pgstattuple` on the
  table (`table_len` 1,368,064), `pg_control_init()` `8192 | 8`, and the whole FSM
  cross-check (`8160 | 0 | 12`); `get_raw_page` refused with `must be superuser to
  use raw page functions` and `gin_clean_pending_list` with `must be owner of index
  r1_gin`. Identical text on both servers.
- **Timeouts.** Against an uncommitted `DROP INDEX`, `lock_timeout = '2s'` cancelled
  the census at 2007 ms on both, and `statement_timeout = '1500ms'` at 1505 / 1504 ms;
  the index survived both.
- **Standby.** A `pg_basebackup` replica of each server censused its GIN index
  identically (1,662,976 bytes, 203 blocks, 54 entry, 148 pending, 213,500 entry
  slack), answered `pgstatginindex` `2 | 148 | 30000` and `pg_freespace` (0 free of
  203), derived 8160, and refused `VACUUM` (`cannot execute VACUUM during
  recovery`), `gin_clean_pending_list` (`recovery is in progress` plus the
  pending-list HINT) and `REINDEX` (`cannot execute REINDEX during recovery`).
- **The horizon sequence.** A held `REPEATABLE READ` snapshot kept all 768 deleted
  pages out of the FSM across three VACUUMs on both, with every page's `prune_xid`
  equal to the holder's `backend_xmin` (20594 on 17.11, 21330 on 12.2), and the
  first VACUUM after the holder was terminated recycled all 768 **with no new
  transaction ids**. The file stayed 7,356,416 bytes at every step on both.
- **The `f2` lifecycle**, all four states, byte-identical to the filed table on both:
  2,195,456 / 268 blocks / `entry_slack` 575,940 and `pgstatginindex` `2 | 0 | 0`;
  then 6,209,536 / 758 / 490 pending / `2 | 490 | 50000`; then 6,209,536 / 758 / 490
  deleted / 4,014,080 waste bytes / `entry_slack` 318,084 / `2 | 0 | 0`; then
  `REINDEX` to 2,605,056.
- **The five flush rounds**, including every FSM reading: 0 free pages then +736
  blocks to build the pending list, then 736 free and zero growth on rounds 2-5, with
  `pg_freespace` reading 0 free at every flush.
- **Rebuild equivalence.** `f9_ric_gin` went 7,356,416 -> 3,948,544 under
  `REINDEX INDEX CONCURRENTLY` (522 / 564 ms) and a plain `REINDEX` immediately
  afterwards returned the same 3,948,544 (156 / 146 ms). Of four identical
  1,261,568-byte fixtures, the two rebuilt while a 100,000-row insert stream ran both
  ended at **2,605,056** (plain 1525 / 1486 ms, concurrent 2084 / 2004 ms) with
  200,000 rows indexed, and the two rebuilt with no load both stayed at 1,261,568;
  all four ended `indisvalid`/`indisready`/`indislive` true with no
  `_ccnew`/`_ccold` leftovers.
- **The autoanalyze flush.** `autovacuum` is `sighup` on both, so a reload sufficed:
  a 246-page pending list vanished within 2 s, the file grew 343 -> 344 blocks, the
  pages became 246 deleted, waste read 71.51% and `entry_slack` 167,376 — the same
  numbers on both servers. One wrinkle is version-specific: on 17.11 the first run
  also recorded `last_autovacuum`, because v13 and later pick up an insert-only table
  through `autovacuum_vacuum_insert_threshold`, a `PGC_SIGHUP` setting defaulting to
  1000 ([guc_tables.c:3359-3366](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3359-L3366)).
  With that reloption set to `-1` the flush still happened in under 2 s with
  `last_autovacuum` null, which is `ginvacuumcleanup`'s `analyze_only` branch
  ([ginvacuum.c:705-717](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717)) on both.
  12.2 has no such setting at all (`pg_settings` returns 0 rows for it), so the
  analyze-only path is the only one available there.
- **The single-read proof.** On a 1106-block index the `OFFSET 0` form read 1125 /
  1122 buffers and the naive double-call form 2230 / 2227 — exactly twice, on both.
- **Cost.** Cold after a restart: 1109 / 1116 blocks read. Warm: 1153 / 1198 hits.
  Two consecutive whole-database censuses produced byte-identical output (110 bytes)
  on both.
- **The entry-tuple identity.** `f3_fresh_gin` read 50,028 entry tuples against
  50,028 distinct lexemes, and the NULL fixture read 33 against 30 distinct integer
  keys plus GIN's three null categories
  ([ginblock.h#GinNullCategory](../../../../raw/postgres-17/src/include/access/ginblock.h#L204-L213)) — on
  both servers, exactly.
- **The derived FSM constant.** `pg_control_init()` gives 8160 on both, and free GIN
  pages read `avail = 8160` on both.

### Concurrency on both majors, and a correction to the detector ranking

Case A is the worst case and it behaves the same way on both, only more so on 12.2.
Fourteen censuses of an unchanging 2,594-block file while one VACUUM deleted
posting-tree pages returned, on 17.11, 0, 0, 253, 654, 993, 1332, 1585, 1881, 2177
and then 2,368 dead pages; on 12.2, whose VACUUM took longer, they returned 0, 0, 0,
0, 74, 148, 253, 327, 401, 549, 592, 666, 740 and 814 against the same final truth of
**2,368** — the last reading understating the truth by a factor of three. The
self-check, the metapage check and the size bracket passed on all 28 censuses.

With sustained load the detectors invert, and this corrects the ranking this page
gave:

| Case | Detector | 17.11 | 12.2 |
|---|---|---|---|
| B: four writers inserting for 90 s | `census_total_pages = blocks` | 0 of 25 caught | 0 of 25 |
| | metapage vs census `pending_pages` | **23 of 25** | **22 of 25** |
| | size bracket | 4 of 25 | 3 of 25 |
| C: insert-and-flush loop, 90 s | self-check | 0 of 25 | 0 of 25 |
| | metapage | 12 of 25 | 11 of 25 |
| | size bracket | 11 of 25 | 12 of 25 |
| D: concurrent `REINDEX CONCURRENTLY` | size bracket | caught the swap (2594 -> 162 blocks) | caught it |

The file grew from 1,721 to 2,496 blocks under the four writers on 17.11 and 1,746 to
2,497 on 12.2, so both detectors had something to find. **Which one wins depends on
the writer, not on the version:** a writer that keeps moving the metapage — any
`fastupdate` insert stream — is caught far more often by the metapage cross-check
than by the size bracket, which is the opposite of what the earlier four-writer run
on this page found. Run both. The statement's own self-check remains blind
throughout: it did not fail once in any census of any of the four cases, on either
server.

### Why round five splits the entry tree

The page's open question 7 — a flush that suddenly grew the entry tree by hundreds of
pages after four rounds that grew it by nothing — has a mechanism, and the five
rounds reproduced it identically on both majors:

| Round | blocks after flush | entry pages | data leaves | `entry_slack` after flush |
|---|---|---|---|---|
| 1 | 1598 | 572 | 193 | 1,735,572 |
| 2 | 1696 | 572 | 290 | 1,340,484 |
| 3 | 1696 | 572 | 290 | 945,372 |
| 4 | 1696 | 572 | 290 | 538,900 |
| 5 | **2047** | **827** | 386 | **2,223,216** |

Entry slack falls by almost exactly 395,000 bytes a round — 2,126,732 to 1,735,572 to
1,340,484 to 945,372 to 538,900 — while the entry tree does not grow at all. The
fifth merge needs the same room, has 538,900 bytes spread across 572 pages that each
hold their own keys, and pays for it with 255 new entry pages at once; `entry_slack`
jumps back up because `entrySplitPage` halves every page it splits
([ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691)).
So the cascade is the exhaustion of accumulated per-page slack, which is why total
slack does not predict the next flush's growth: what matters is the slack on the page
that holds each key. The earlier run's +649 pages and this run's +255 are the same
event at different fill levels.

### Eviction, re-measured on both majors

The earlier finding — that a census strips a hot working set's usage count without
evicting it — holds only while the census target fits the cache. With
`shared_buffers = 64MB` (8,192 buffers), a 9,616-block census target, and two
1,862-page tables (below the `NBuffers / 4` ring-buffer threshold of 2,048), one
census destroys the hot set:

| | 17.11 | 12.2 |
|---|---|---|
| before the census | hot 1,862 pages at `usagecount` 3, cold 1,535 | hot 1,853 at 3, cold 1,549 |
| after one census | hot **80** pages at `usagecount` 0, cold **0**, census pages 7,961 | hot **0**, cold 0, census pages 8,046 |
| after two censuses | hot 0 | hot 0 |
| control: seq scan of a 16k-block table, hot re-warmed to `usagecount` 5 | hot 1,862 **intact** | hot 1,853 **intact** |

The mechanism is the one the page already cites — `PinBuffer` raises the count,
`StrategyGetBuffer` decrements what it passes and takes the first zero
([bufmgr.c:2700-2705](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2700-L2705),
[freelist.c#StrategyGetBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L314-L341)) —
but the outcome depends on the count the set is carrying: from 5 it survives a
census, from 3 it does not, and 12.2 lost it one census sooner than 17.11 in this
run. The control confirms the asymmetry the page draws: a sequential scan of a table
larger than `NBuffers / 4` takes a `BAS_BULKREAD` ring
([heapam.c:434-458](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L434-L458)) and left the hot
set untouched on both servers, where the census has no such limiter.

### Reading rules

- Report the three classes separately, and keep them beside `bloat_pct` rather than
  behind it. The single number is `waste + slack`, so it carries an entry tree's
  growth room: a never-churned index reads 48.05 and an empty one 49.80, both with
  nothing to reclaim. Publish it as a page-accounting percentage with no guaranteed
  bound on rebuild savings. Adding `pending_pct` still misses retained empty-key
  entries, so it does not establish a bound either
  ([ginvacuum.c:507-558](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L507-L558)).
- `whole_page_waste_bytes` counts marked-deleted and uninitialized pages; immediate
  reuse also depends on the deletion horizon and allocation path
  ([ginvacuum.c#GinPageIsRecyclable](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L805-L829),
  [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L294-L335)).
  With `W = whole_page_waste_bytes`, `B = old main-fork bytes`, and
  `R = rebuilt main-fork bytes`, the comparison is exact algebra:
  `W > B - R` if and only if `B - W < R`. Whole-page waste therefore
  **overstates** the size reduction when the aged in-use core is smaller than the
  rebuild. A rebuild creates a new physical index
  ([index.c:3781-3789](../../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)).
- `entry_slack` on a healthy index is a large fraction of an entry-tree-dominated
  file: the freshly built and freshly rebuilt indexes here read 41.06%, 47.19%,
  48.05% and 48.05% slack while reclaiming nothing, and across opclasses the fresh
  payload fraction ran from 51.30% (`btree_gin`) to 72.11% (`pg_trgm`). Treat it as
  a level, not a defect, and compare an index to its own history or to a rebuilt
  twin — never to another opclass.
- Read a busy index twice before believing either reading, and **re-read
  `pg_relation_size` after the census**: that bracket caught a four-writer race in
  13 of 14 censuses where the metapage cross-check caught 0 of 14, and the
  statement's own self-check passed every time in both tests. A concurrent VACUUM
  defeats all three, so on a table being vacuumed, only two disagreeing censuses
  tell you anything.
- If the payload/fill model matters to your decision, run the entry-tuple probe
  beside the census. Entry tuples far above the table's live distinct-key count
  mean the model is over-predicting the rebuild, by about half a point per percent
  of the key population that has died.
- Dead pages that will not go away may be waiting on one idle transaction, not on
  VACUUM. A held `REPEATABLE READ` snapshot kept 768 recyclable-looking pages out
  of the FSM across three VACUUMs.
- `data_slack` is the honest bloat signal: a posting-tree-dominated index with
  slack far above a fresh rebuild's has genuinely reclaimable space.
- `pending_bytes` is a `fastupdate` tuning signal, not waste. Fix it with VACUUM,
  `gin_clean_pending_list()`, or `gin_pending_list_limit`.
- If `gin_version <> 2`, publish the page counts and suppress the slack columns —
  and every payload estimate derived from untrusted slack
  ([ginblock.h:302-309](../../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309)).
  The statement currently gates only the combined slack fields and `bloat_pct`;
  its exposed component and payload fields are a known gap in the revised plan.
- If `uncompressed_pages > 0`, the index carries pre-9.4 posting-tree leaves and
  its `data_slack` is understated.
- Report the `maintenance_work_mem` a rebuild comparison was measured at. One
  800,000-row index rebuilt to anything from 34.6 MB to 50.8 MB across budgets, and
  the biggest budget gave the biggest index, so "what a rebuild would reclaim" is
  not a property of the index alone.
- On a server older than 12.11, keep the `pagesize = 0` arm and the probe's
  `pagesize > 0` guard. Without them the census counts all-zero pages as entry
  pages and reports their waste as zero, and the probe subtracts 6 downlinks per
  zeroed page. The `flags IS NULL` arm is a no-op there.
- On 12.x, census a suspect index on its own. A zeroed metapage aborts the whole
  multi-index statement rather than returning NULLs for that one index.
- On 12.x, do not trust `pgstatginindex` to tell you an index is invalid: it
  answers with a metapage reading instead of refusing.
- On 12.x, read the VACUUM cross-check out of the `DETAIL` block, and expect one
  deleted-pages number rather than the "newly deleted" versus "currently deleted"
  pair.

## Context Reviewed

- **2026-09-05 plan review:** rechecked the seven findings in
  [Plan review](#plan-review) against the pinned v17 implementation, and audited
  the original six steps plus the published census and cross-check statements.
  No database server was built or started; the historical observations below
  were not rerun. SQL code blocks were preserved during this review.
- **Caller and data-structure context for this review:** core `ginhandler`
  installs `ginbulkdelete` and `ginvacuumcleanup`; the index AM dispatcher calls
  those callbacks, with `IndexVacuumInfo` supplied by the heap vacuum path.
  Contrib readers and GIN use the shared `GinPageOpaqueData`, `GinMetaPageData`,
  and page-header definitions, rather than an extension-specific page layout
  ([ginutil.c#ginhandler](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L36-L69),
  [indexam.c:748-777](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L748-L777),
  [vacuumlazy.c#lazy_cleanup_one_index](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2469-L2499),
  [ginfuncs.c:10-22](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L10-L22),
  [gin_private.h:13-21](../../../../raw/postgres-17/src/include/access/gin_private.h#L13-L21),
  [ginblock.h:30-138](../../../../raw/postgres-17/src/include/access/ginblock.h#L30-L138),
  [bufpage.h#PageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L155-L168)).
- **Build and SQL interfaces for this review:** the `pageinspect` build links
  `ginfuncs` and `rawpage` and installs the extension SQL upgrades. The core
  catalog-header build generates `*_d.h`, including the catalog header included
  by `gin_private.h`; those headers are build products, not evidence files to
  manufacture in `raw/`. Block size comes from configuration, and the extension's
  SQL version controls argument/result widths: `get_raw_page` takes `int8` in
  1.9, while `page_header` exposes integer offsets from 1.10. The source-only
  revision requires no core catalog, parser, or header edits
  ([pageinspect/Makefile](../../../../raw/postgres-17/contrib/pageinspect/Makefile#L3-L35),
  [catalog/Makefile:86-88](../../../../raw/postgres-17/src/include/catalog/Makefile#L86-L88),
  [catalog/Makefile:126-143](../../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143),
  [gin_private.h:13-21](../../../../raw/postgres-17/src/include/access/gin_private.h#L13-L21),
  [configure.ac:267-288](../../../../raw/postgres-17/configure.ac#L267-L288),
  [pageinspect--1.8--1.9.sql:46-58](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L46-L58),
  [pageinspect--1.9--1.10.sql:10-21](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L10-L21)).
- **Additional boundaries reviewed:** maintenance-option parsing and callback
  suppression, core function signatures, source GUC contexts, per-call raw-page
  locks, the decoder's unknown-bit handling, the planner's subquery pull-up
  restrictions, and the shipped contrib tests. The `SET`, `VACUUM (...)`, and
  `REINDEX` grammar and the core function catalog entries were checked; no new
  executable SQL is introduced by this review
  ([gram.y:1663-1681](../../../../raw/postgres-17/src/backend/parser/gram.y#L1663-L1681),
  [gram.y#ReindexStmt](../../../../raw/postgres-17/src/backend/parser/gram.y#L9189-L9202),
  [gram.y:11821-11829](../../../../raw/postgres-17/src/backend/parser/gram.y#L11821-L11829),
  [pg_proc.dat:7487-7495](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L7487-L7495),
  [pg_proc.dat:9513-9516](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L9513-L9516),
  [pg_proc.dat:11989-11996](../../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L11989-L11996)).
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
  which is why the server was rebuilt and the fixtures republished. **`ginw2` no
  longer exists either**: it was deleted on 2026-08-26 at the user's instruction,
  so reproducing anything below means rebuilding 17.11 from `raw/postgres-17/`
  again. Every fixture needed to do that is published above as SQL, and the
  open-questions pass already proved that path returns the filed numbers byte for
  byte.
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
- Open-questions pass, source side: the FSM's write and search paths and the
  asymmetry between them (`RecordPageWithFreeSpace`, `fsm_set_and_search`,
  `fsm_search` from `FSM_ROOT_ADDRESS`, `GetPageWithFreeSpace`,
  `FreeSpaceMapVacuum`, the FSM README's account of when upper nodes are updated),
  the index FSM wrappers (`GetFreeIndexPage`, `RecordFreeIndexPage`,
  `IndexFreeSpaceMapVacuum`) and `GinNewBuffer`'s use of them; `pg_control_init`
  and its documented output columns; `SizeOfPageHeaderData` and the entry-page
  `PageAddItem` call sites in `ginentrypage.c`; `GinNullCategory` and its five
  category constants; and the buffer-replacement path — `PinBuffer`'s usage-count
  increment, `BM_MAX_USAGE_COUNT`, and `StrategyGetBuffer`'s clock sweep.
- Open-questions pass, server side, all on the same sandbox
  (`.wiki-runtime/tmp/ginw2/`, exact-pin 17.11, port 55432): the published fixture
  SQL re-run unmodified in two freshly created databases and scored again; a
  five-point churn sweep; an entry-tuple probe validated against
  `count(DISTINCT ...)` on three shapes including one with NULL and empty arrays;
  three fixtures grown entirely through the pending list; a five-round
  insert-and-flush experiment with `pg_freespace` and per-class slack captured at
  every step; four concurrency cases (a VACUUM deleting posting-tree pages, four
  writers, an insert-and-flush loop, a `REINDEX INDEX CONCURRENTLY`) with each
  census bracketed by separate size readings; plain and concurrent rebuilds of
  identical fixtures with and without a 100,000-row insert stream; four more
  opclasses plus the same opclass at 50,000 and 800,000 rows; and a
  `pg_buffercache` eviction experiment at `shared_buffers = 128MB` with a hot and a
  cold working set, restored to 256MB afterwards.
- Two-major pass, source side: the `pageinspect` extension scripts that changed the
  three signatures this statement depends on (`get_raw_page`'s block-number type in
  1.9, `page_header`'s column types in 1.10) and the `PageIsNew` early returns in
  `ginfuncs.c`; the GIN build path's memory rule (`ginBuildCallback`'s
  `maintenance_work_mem` comparison, `BuildAccumulator.allocatedMemory`, the
  `GetMemoryChunkSpace` accounting in `ginbulk.c`, and `GetMemoryChunkSpace`'s own
  definition and comment in `mcxt.c`); `maintenance_work_mem` and
  `autovacuum_vacuum_insert_threshold` in `guc_tables.c`; the `indisvalid` checks in
  `pgstatindex.c` and `pgstattuple.c`; and this checkout's own commit history for
  `f18aa1b2039`, `127404fbe28`, `cd4868a5700` (with its `REL_12_11` backport
  `5378d55cb2f`), `13503eb5905`, `c6e0fe1f2a0` and `2c2eb0d6b27`, plus the GIN
  commits between `REL_12_0` and the pin that touch `ginbulk.c`, `gininsert.c`,
  `ginentrypage.c` and `gindatapage.c`.
- Two-major pass, server side: two isolated clusters built out of tree from this
  repo's own pins — **12.2** (`server_version_num` 120002, port 55412) and **17.11**
  (170011, port 55417), both `--without-readline --without-zlib`, `block_size` 8192,
  `autovacuum = off`, `fsync = off`, `shared_buffers = 256MB`, `pageinspect`,
  `pgstattuple`, `pg_freespacemap`, `pg_trgm`, `btree_gin` and `pg_buffercache`
  installed from their own trees (1.7/1.5/1.2/1.4/1.3/1.3 on 12.2 and
  1.12/1.5/1.2/1.6/1.3/1.5 on 17.11). Run on both: all four published statements
  verbatim and portable; 27 scored fixtures with `REINDEX INDEX` ground truth; the
  `f2` lifecycle; the `f5` three-VACUUM sequence with `VACUUM VERBOSE` and FSM
  captures; a held `REPEATABLE READ` snapshot; five insert-and-flush rounds; plain
  and concurrent rebuilds with and without a 100,000-row insert stream; four
  concurrency cases with per-census size brackets (100 censuses in total); an
  autoanalyze-only flush with and without `autovacuum_vacuum_insert_threshold`; a
  `pg_stat_scan_tables`-only role; the refusal matrix including a failed
  `CREATE INDEX CONCURRENTLY` leftover and another session's temp index; both
  timeout cancellations; a `pg_basebackup` standby each; a `-m immediate` crash
  during a bulk insert; two hand-appended all-zero blocks and a `dd`-zeroed
  metapage; a `maintenance_work_mem` sweep from 4MB to 1GB with parallel-worker
  controls;
  `EXPLAIN (ANALYZE, BUFFERS)` single-read proofs; cold and warm cost after a
  restart; and a `pg_buffercache` eviction experiment at `shared_buffers = 64MB`,
  restored to 256MB. Both sandboxes were deleted when this page was filed, at the
  asker's instruction, so reproducing any of it means rebuilding both servers from
  this repo's two pinned checkouts and re-running the SQL printed here.
- `bloat_pct` pass, source side: the version gate the column inherits
  (`GinMetaPageData` and the `pd_lower` trust note in `ginblock.h`) and
  `entrySplitPage`, which is why a fresh build's fill sets the column's floor. No new
  source surface was needed, because the column is arithmetic over terms the census
  already computes.
- `bloat_pct` pass, server side: two fresh clusters, both deleted when this page was
  filed. **17.11** at `.wiki-runtime/tmp/ginpct17/` (port 55437, VPATH build from
  `raw/postgres-17/`, `autovacuum = off`, `fsync = off`, `shared_buffers = 256MB`,
  `pageinspect` 1.12 / `pgstattuple` 1.5 / `pg_freespacemap` 1.2) carrying the seven
  published fixtures built from this page's own fixture SQL plus its documented
  three-VACUUM sequence for `f5`, in an aged database and a rebuilt one. **12.2** at
  `.wiki-runtime/tmp/ginpct12/` (port 55412, binaries from the retained
  `.wiki-runtime/tmp/pta12/pg` install of this repo's v12 pin, with `pageinspect` 1.7
  and `pg_freespacemap` 1.2 built from the same tree and removed again afterwards,
  same server settings), carrying the same fixtures with `txid_current()` in place of
  `pg_current_xact_id()`. Run on both: the filed census text and the amended text
  compared cell by cell, the amended text scored against `REINDEX INDEX` over all
  seven fixtures, a `fastupdate` fixture censused with a live 246-page pending list
  and then rebuilt, and `EXPLAIN (ANALYZE, BUFFERS)` on both texts. The amended text
  published above was extracted from this page's own Markdown and re-run on both
  servers; on 17.11 it returned the aged corpus's table byte for byte. The 17.11
  aged corpus was then rebuilt index by index and landed on the same seven byte
  counts as the separately rebuilt database and as the 12.2 rebuilds, which is what
  the `REINDEX reclaimed %` column above rests on.
- Version scope of the 12.2 column: measurement plus this checkout's history only.
  No v12 source file is cited on this page, and the v12-side source analysis is not
  filed anywhere yet.

## Evidence Map

| Claim | Evidence |
|---|---|
| Plan review: page gaps exclude retained empty-key entry tuples; a rebuild builds from the heap | [ginvacuum.c:507-558](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L507-L558), [gininsert.c#ginbuild](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L317-L406) |
| Plan review: a pending flush does not perform the full metapage-count refresh | [ginfast.c#gin_clean_pending_list](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1031-L1091), [ginvacuum.c:754-802](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L754-L802) |
| Plan review: disabled cleanup and the failsafe can prevent the GIN cleanup callback | [vacuum.c:2155-2178](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2178), [vacuumlazy.c:392-401](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L401), [vacuumlazy.c:1064-1066](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066), [vacuumlazy.c:2323-2335](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2335) |
| Plan review: a raw-page copy releases its locks before the next call | [rawpage.c#get_raw_page_internal](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L198) |
| Plan review: `PageIsNew` tests one header field; the GIN opaque decoder reports unknown bits | [bufpage.h#PageIsNew](../../../../raw/postgres-17/src/include/storage/bufpage.h#L226-L234), [ginfuncs.c:119-159](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L119-L159) |
| Plan review: the version gate must cover every value derived from untrusted data-page offsets | [ginblock.h:302-309](../../../../raw/postgres-17/src/include/access/ginblock.h#L302-L309); static audit of the filed SELECT list: the component slack and payload expressions have no version guard |
| Plan review: the offset and volatile-target checks preserve the raw-read subquery boundary | [prepjointree.c:1689-1701](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1689-L1701), [prepjointree.c:1772-1781](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1772-L1781) |
| Plan review: the rebuild comparison measures main-fork file lengths | [dbsize.c#calculate_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L343), [dbsize.c#pg_relation_size](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L345-L370) |
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
| `REINDEX CONCURRENTLY` gives the same size as `REINDEX` | server: 7,356,416 -> 3,948,544 bytes either way on the same fixture recipe, 379 ms against 151 ms, and 3,276,800 -> 2,662,400 either way under a 100,000-row insert stream (1901 ms against 2013 ms) |
| The published fixtures reproduce across runs | server: the published SQL re-run in two virgin databases gave all seven filed sizes, page-class counts, slack bytes, lifecycle states and `REINDEX` results byte for byte; only `f5`'s `prune_xid` moved (953 against 791 and 870) |
| Entry-leaf `pd_lower` counts the index's keys | [ginentrypage.c:561-568](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L561-L568), [ginentrypage.c:683-691](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L683-L691), [bufpage.h#SizeOfPageHeaderData](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L214); server: 50,028 = 50,028 lexemes, 32 = 32 tags, and 53 = 50 tags + 3 null categories ([ginblock.h#GinNullCategory](../../../../raw/postgres-17/src/include/access/ginblock.h#L204-L213)) |
| The payload model's error is linear in the dead-key share | [README:389-396](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396); server: +0.09 / +12.62 / +25.16 / +37.69 / +50.19% at 0 / 25 / 50 / 75 / 100% of the key population replaced, increments of +12.53, +12.54, +12.53, +12.50 |
| Dead entry tuples are smaller than live ones, so the average over-corrects | server: average entry tuple 32.0 bytes at `p = 0` falling to 24.0 at `p = 100`, putting dead tuples at 16 bytes; corrected estimate −7.48 to −18.44% on the sweep and −36.80% on `f1` |
| A pending list is built from the FSM's free stock, the merge is not | [ginutil.c#GinNewBuffer](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L299-L328), [indexfsm.c#GetFreeIndexPage](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L37-L46); server: 0 free pages -> the file grew by exactly the 736 pending pages, 736 free -> zero growth on four consecutive rounds, and `pg_freespace` read 0 at every flush |
| A flush cannot reuse the pages it frees | [freespace.c#RecordPageWithFreeSpace](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L186-L204), [freespace.c#fsm_search](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L684-L691), [ginfast.c:1014-1020](../../../../raw/postgres-17/src/backend/access/gin/ginfast.c#L1014-L1020) |
| Flush growth is not a function of total slack | server: +194 blocks at 2,326,056 bytes of slack against +0 at 3,197,144 and +649 at 1,885,966 |
| Waste is a lower bound iff the in-use core is at least the rebuild | server: `fh2_gin` 21.66% dead against 19.25% reclaimed with in-use 10,903,552 under a 11,239,424-byte rebuild, against `fh1_gin` holding by six blocks and `fh3_gin` holding; aged-core fill predicted the direction 3 of 3 |
| The size bracket beats the metapage cross-check on writers | server: 13 of 14 against 0 of 14 under four concurrent writers, with the statement's self-check passing 14 of 14 |
| A concurrent VACUUM defeats every check | server: 14 censuses of an unchanging 2,594-block file reported 0 to 2,294 dead pages against a truth of 2,368, all three checks passing every time |
| A census strips usage count rather than evicting a hot set | [bufmgr.c:2700-2705](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2700-L2705), [buf_internals.h#BM_MAX_USAGE_COUNT](../../../../raw/postgres-17/src/include/storage/buf_internals.h#L72-L79), [freelist.c#StrategyGetBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L314-L341); server at 16,384 buffers: a 26,195-block census left a hot 3,704-page set fully resident at usagecount 1 (from 5) and destroyed a once-read set of the same size; a second census took the hot set too |
| The FSM free-page value is derivable, not a constant to type | [pg_controldata.c#pg_control_init](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L203-L228), [func.sgml#pg_control_init](../../../../raw/postgres-17/doc/src/sgml/func.sgml#L27721-L27742), [htup_details.h#MaxHeapTupleSize](../../../../raw/postgres-17/src/include/access/htup_details.h#L563); server: the derived expression returned 8160, equal to the largest `avail` over 3,478 free GIN pages, and `pg_control_init()` is executable by `PUBLIC` |
| Fresh-build fill is opclass-dependent but scale-insensitive | server: 50.16% to 72.11% across nine opclasses and shapes, while the same opclass at 50,000 and 800,000 rows read 51.12% and 50.92% with model errors of +33.18% and +33.60% |
| The census statement needs a block-number cast to run on 12 | [pageinspect--1.8--1.9.sql#get_raw_page](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L46-L58) (`f18aa1b2039`, earliest tag `REL_14_0`); 12.2: `ERROR: function get_raw_page(text, bigint) does not exist`, and `::int` makes both servers produce byte-identical output |
| `page_header`'s width columns are narrower on 12 | [pageinspect--1.9--1.10.sql#page_header](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L10-L21) (`127404fbe28`, earliest tag `REL_15_0`); 12.2 returns `smallint` for `lower`/`upper`/`special`/`pagesize`, which is harmless at `block_size` 8192 |
| 12.2 does not return NULL for an all-zero page | [ginfuncs.c:49-50](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L49-L50), [ginfuncs.c:119-120](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L119-L120); `cd4868a5700` carries `Backpatch-through: 10` and its 12-branch backport `5378d55cb2f` is first tagged `REL_12_11`, after this repo's 12.2 pin; 12.2 returned `flags = {}` where 17.11 returned a NULL row |
| Without the `pagesize` guard the 12.2 census hides zeroed pages | server: two appended all-zero blocks read 54 entry / 2 new / 16,384 waste bytes on 17.11 and 56 entry / 0 new / 0 bytes on 12.2 from the same text, with the self-check passing; the probe's downlinks read 41 against 53 |
| A zeroed metapage aborts the whole census on 12.2 | server: `gin_metapage_info` returned a NULL row on 17.11 and `ERROR: input page is not a GIN metapage` on 12.2, killing the multi-index statement |
| `pgstatginindex` answers for an invalid index on 12.2 | [pgstatindex.c:538-543](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L538-L543), [pgstattuple.c:263-267](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L263-L267) (`13503eb5905`, earliest tag `REL_17_0`); server: 17.11 `index "..." is not valid`, 12.2 returned `2 \| 0 \| 0` on a failed-CIC leftover |
| 26 of 27 fixtures score identically on the two majors | server: identical size, rebuilt size, waste, slack, pending, reclaimed, both bound verdicts, fill, model error and dead-tuple counts; identical tallies of 24 / 25 / 26 of 27 for the lower bound, upper bound and upper-plus-pending |
| `waste + slack` is not an upper bound with a large dead-key population | [README:389-396](../../../../raw/postgres-17/src/backend/access/gin/README#L389-L396); server: `k5_gin` read 43.43% against 50.00% reclaimed on 17.11 and 39.03% against 49.99% on 12.2, with 819,770 dead entry tuples of 1,639,711 |
| A live pending list breaks the upper bound too | server: `fh1_gin` read 0.00 + 13.73% against 40.00% reclaimed with 246 pending pages (53.48% of the file); the settled twin `fh1b_gin` read 51.04 + 12.62% against 42.74% |
| A fresh GIN build's size is a function of `maintenance_work_mem` | [gininsert.c:290-291](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L290-L291), [gin_private.h:434](../../../../raw/postgres-17/src/include/access/gin_private.h#L434), [ginbulk.c:139](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L139), [mcxt.c#GetMemoryChunkSpace](../../../../raw/postgres-17/src/backend/utils/mmgr/mcxt.c#L718-L730), [guc_tables.c:2466-2474](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2474); server: one index rebuilt to 34,611,200 at 64MB and 50,814,976 at 96MB and above, on both majors, with payload constant at about 25.6 MB and all the movement in `entry_slack` |
| The build divergence is a spill effect, not parallelism | server: `max_parallel_maintenance_workers` 0, 2 and 4 gave identical bytes; both majors returned exactly 50,814,976 once the build fitted one flush; v12 at 68MB and 70MB brackets v17 at 64MB |
| 12.2 prints the VACUUM cross-check differently | [vacuumlazy.c:718-731](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L731), [ginvacuum.c:786-794](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L786-L794); server: 12.2 prints `768 index pages have been deleted, 0 are currently reusable.` in a DETAIL block with no "newly deleted" split, and the same 768 / 0 / 768 sequence |
| Which race detector wins depends on the writer | server: under four writers the metapage check caught 23 of 25 (17.11) and 22 of 25 (12.2) where the size bracket caught 4 and 3; under an insert-and-flush loop 12 and 11 against 11 and 12; the self-check never fired in any census of any case on either server |
| A concurrent VACUUM is worse on the slower server | server: 14 censuses of an unchanging 2,594-block file read up to 2,368 dead pages on 17.11 and only 814 on 12.2 against the same 2,368 truth, with all three checks passing throughout |
| The round-five split cascade is slack exhaustion | [ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691); server: `entry_slack` fell 2,126,732 -> 1,735,572 -> 1,340,484 -> 945,372 -> 538,900 with the entry tree fixed at 572 pages, then round five added 255 entry pages and slack jumped to 2,223,216 — identical on both majors |
| A census evicts a hot set once the target exceeds the cache | [bufmgr.c:2700-2705](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2700-L2705), [freelist.c#StrategyGetBuffer](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L314-L341), [heapam.c:434-458](../../../../raw/postgres-17/src/backend/access/heap/heapam.c#L434-L458); server at 8,192 buffers: a 9,616-block census took a 1,862-page hot set from `usagecount` 3 to 80 resident pages (17.11) and to 0 (12.2), while a 16k-block seq scan left it intact on both |
| v13+ also vacuums an insert-only table, which 12.2 does not | [guc_tables.c:3359-3366](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3359-L3366), [ginvacuum.c:705-717](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L705-L717); server: 17.11 recorded `last_autovacuum` until `autovacuum_vacuum_insert_threshold` was set to -1, 12.2 has no such setting, and both then flushed 246 pending pages in under 2 s with `last_autovacuum` null |
| Rebuild equivalence holds on both majors | server: `f9` 7,356,416 -> 3,948,544 by either form; of four identical fixtures the two rebuilt under a 100,000-row insert stream both ended at 2,605,056 and the two rebuilt idle both stayed at 1,261,568, all ending valid/ready/live |
| `bloat_pct` changes nothing else in the statement | server: the filed text and the amended text returned identical values in all 25 pre-existing columns over 7 indexes (175 of 175 cells), the amended text adding only `bloat_pct` |
| The column reads no extra page | server: `EXPLAIN (ANALYZE, BUFFERS)` reported `shared hit=7568` for both texts over the same seven-index census |
| `bloat_pct` reproduces the page's `waste+slack %` scoring column | server: 64.12 / 75.30 / 48.05 / 49.80 / 95.22 / 51.34 / 54.15 on `f1`-`f7`, the same seven values the scoring table prints |
| A healthy GIN index reads about half its file as `bloat_pct` | [ginentrypage.c#entrySplitPage](../../../../raw/postgres-17/src/backend/access/gin/ginentrypage.c#L666-L691); server: `f3_fresh_gin` 48.05 with `REINDEX` returning 0 bytes, `f4_empty_gin` 49.80 over one entry page holding 8,160 free bytes |
| `bloat_pct` bounded the rebuild on all seven fixtures, loosely | server: over-read +5.01 to +49.80 points against `REINDEX` truths of 42.42 / 58.05 / 0.00 / 0.00 / 89.09 / 46.33 / 13.26% |
| A live pending list makes the column under-read badly | server: 246 pending pages gave `bloat_pct` 8.66 and `pending_pct` 81.73 while `REINDEX` took the index 2,465,792 -> 794,624 bytes (67.77%), identical on 12.2; the sum, 90.39, bounds it |
| The amended text needs no further edit on 12.2 | server: it ran there unchanged and returned 178 of 182 identical cells over the seven fixtures, the four differences being `f7`'s known 128-and-10-byte slack gap, with `bloat_pct` identical on all seven and the 12.2 rebuilds landing on the same seven byte counts |
| Rounding the sum is not summing the rounded columns | server: `f2_pending_gin` prints 64.64 and 10.65 against a `bloat_pct` of 75.30 |

## Open Questions

Entries 1 through 16 retain the earlier investigation's results and gaps; the
closed entry 7 remains for continuity. The 2026-09-05 plan review adds entries
17 through 19 for work needed to implement and validate the revised procedure.

1. **The `ginVersion <> 2` path is still untested.** v17 always writes
   `GIN_CURRENT_VERSION = 2` ([ginutil.c:355-382](../../../../raw/postgres-17/src/backend/access/gin/ginutil.c#L355-L382)),
   so no fixture can exercise the suppressed-slack branch or an uncompressed
   posting-tree leaf. `uncompressed_pages` was 0 and `gin_version` was 2 on every
   index measured, across all runs, all 26-then-27 scored fixtures, and now both
   majors — including a 12.2 server, which is the closest thing to a pre-9.4-format
   producer this repo has and still writes version 2.
2. **The payload-and-fill model can be diagnosed but not corrected.** The
   entry-tuple probe now measures the dead-key population exactly, and the error is
   linear in it, but turning that into a corrected prediction needs the *size* of a
   dead entry tuple, which no contrib function exposes: `gin_leafpage_items` reads
   only posting-tree leaves, and using the average tuple size over-corrects by
   −7.5% to −36.8%. The two estimates bracket the truth, which is weaker than a
   prediction. The fill fraction is also still taken from the rebuild it predicts —
   and it is now known to depend on that rebuild's `maintenance_work_mem`, which
   ranged 50.56% to 73.93% for one index, so the divisor is a property of the
   rebuild rather than of the index.
3. **One page size and one alignment, though no longer hardcoded.** The FSM
   cross-check now derives `MaxFSMRequestSize` from `pg_control_init()` instead of
   assuming 8160, and the entry-tuple identity confirms
   `SizeOfPageHeaderData = 24` and `sizeof(ItemIdData) = 4` empirically on this
   build. But everything was still measured at `block_size` 8192 with `MAXALIGN` 8,
   now on two clusters instead of one (both returned 8160 from the derived
   expression), and no build at another `BLCKSZ` was made, so the expression is
   unverified off the default. There is one concrete known risk there: 12.2's
   `page_header` returns `smallint` widths, which would overflow at `BLCKSZ` 32768,
   so on that build the census would need explicit casts that 17.11 does not.
4. **The merge's own page demand is still unpredictable.** Two of the three moving
   parts are now pinned: the pending list is built from the FSM's free stock (0 free
   pages -> the file grew by exactly the 736 pending pages; 736 free -> zero growth,
   four times running), and a flush can never reuse the pages it is itself freeing.
   What is left is how many pages the merge wants, and total slack does not predict
   it — 194 blocks of growth at 2.33 MB of slack against none at 3.20 MB — because
   slack is bound to the page that holds each key. The two-major run adds the
   *shape* of the answer (see number 7) but not a prediction: total slack fell by an
   almost constant 395 kB a round until it ran out, so the round in which the
   cascade lands is a function of the per-page slack distribution, which was still
   not measured.
5. **The lower bound now has a rule, but the rule needs a rebuild to evaluate.**
   `waste <= reclaimed` holds exactly when the in-use core is at least as big as the
   rebuild, and the density form of that test predicted the direction on 3 of 3 new
   fixtures including a six-block near miss. But the fresh-build fill it compares
   against can only come from an actual rebuild or a twin, and fresh fill ranges
   from 50.16% to 72.11% across opclasses, so there is still no way to evaluate the
   rule in advance on an index you have not rebuilt. It got harder rather than
   easier: the rebuild the rule compares against is itself a function of
   `maintenance_work_mem`, so "the in-use core against its own rebuild" needs the
   rebuild's memory budget named before the comparison means anything.
6. **A concurrent VACUUM defeats every cross-check, and no bound is known.** 14
   censuses during one VACUUM of a 2,594-block index reported dead-page counts from
   0 to 2,294 against a truth of 2,368 — 91% of the file — with the self-check, the
   metapage check and the size bracket all passing every time. The size bracket
   catches writers (13 of 14) but is blind here because the file does not change
   size. Nothing was measured for several simultaneous vacuums, for a census
   straddling a `REINDEX CONCURRENTLY` swap (in 22 attempts across two servers it
   always landed between censuses, not inside one), or for how wrong a single census
   can get in the worst case. The 12.2 run makes it worse rather than better: its
   slower VACUUM left the last of 14 censuses reading 814 dead pages against a truth
   of 2,368, so the error is bounded by nothing except how long the VACUUM takes.
7. **Closed: the round-five entry-tree explosion is slack exhaustion.** Five
   identical rounds on both majors show `entry_slack` falling by an almost constant
   395 kB per merge — 2,126,732 -> 1,735,572 -> 1,340,484 -> 945,372 -> 538,900 —
   with the entry tree pinned at 572 pages, and then the fifth merge, needing the
   same room and no longer having it on the pages that hold its keys, adding 255
   entry pages at once and taking slack back up to 2,223,216 because
   `entrySplitPage` halves what it splits. The earlier +649 pages and this +255 are
   the same event at different fill levels. What remains unpredictable is *which*
   round it lands in, which is number 4.
8. **The opclass sweep is broader but still one fixture per opclass.** Nine
   opclasses and shapes have now been scored — `array_ops` on `int[]` and `text[]`,
   `jsonb_ops`, `jsonb_path_ops`, `gin_trgm_ops`, `btree_gin`, plain and weighted
   `tsvector`, multicolumn, partial — and the 16x scale check moved the model error
   by 0.42 points, so scale is no longer a suspect. Untested: collation-dependent
   text keys, additional multicolumn or expression-index shapes, and any
   opclass under a *mixture* of surviving and dying keys other than the five-point
   `tsvector` sweep.
9. **Eviction now has two cache sizes and a corrected conclusion, but still a
   synthetic workload.** At 16,384 buffers a hot set at `usagecount` 5 survived one
   census; at 8,192 buffers with a census target larger than the whole cache, a hot
   set at `usagecount` 3 did not survive one on 12.2 or two on 17.11. So the answer
   depends on the usage count the set carries and on whether the target fits the
   cache, and the flat claim that a census "does not evict a hot working set" was
   wrong. What a census costs a real mixed workload — many relations at different
   usage counts, a bgwriter and checkpointer running — is still unmeasured, and so is
   the boundary in usage count between surviving and not.
10. **All-zero pages are producible on demand but not predictable from a crash.**
    The earlier run got 4 and 7 zeroed pages from `-m immediate` kills; the same
    recipe on both majors this time got **none**, twice. Appending zeroed blocks to
    the file with the server down is the deterministic route and is what the
    classification test used, so the census behaviour is now measured on both majors
    — but what governs whether a crash leaves zeroed blocks at all is still
    uninvestigated.
11. **The exact commit behind the build-memory divergence was not identified.** The
    effect is measured (17.11 fits about 7% more accumulator entries into the same
    `maintenance_work_mem` than 12.2, and both converge above the single-flush
    threshold) and the mechanism is in the source (`GetMemoryChunkSpace` includes
    allocator overhead in the budget), but attributing it to
    `c6e0fe1f2a0` (`REL_16_0`), `2c2eb0d6b27` (`REL_17_0`) or something else would
    need per-major builds of the intermediate releases, which were not made.
12. **Only 12.2 and 17.11 were run, so every difference is a range, not a boundary.**
    v13 through v16 were not built. Each difference above is attributed to a commit
    plus its earliest containing tag from this checkout's history, not measured at
    the release where it lands. The most consequential untested inference is that a
    12.11-or-later server behaves like 17.11 on all-zero pages, which follows from
    `cd4868a5700`'s `Backpatch-through: 10` and the `REL_12_11` tag on its 12-branch
    backport but was not run.
13. **The dead-entry-tuple term that repairs the upper bound is not itself a bound.**
    Adding roughly 16 bytes per dead entry tuple restores `waste + slack >= reclaimed`
    on `k5_gin`, but 16 bytes is an average inferred from one churn sweep, no contrib
    function reports a dead entry tuple's size, and the repaired inequality was
    checked on one fixture rather than derived. The `::int` and `pagesize` edits were
    also only cross-checked against the filed text on a handful of indexes on 17.11,
    not on all 27 fixtures.
14. **Where the spill threshold falls was mapped for one index only.** The
    `maintenance_work_mem` sweep covers one 800,000-row `jsonb_path_ops` index. Which
    opclasses and scales cross into the multi-flush regime at the 64MB default — and
    therefore which indexes report different rebuild sizes on different majors — was
    not surveyed; the other 26 fixtures happen to fit one flush, which is why they
    are byte-identical.
15. **The 12.2 findings are measurement without source.** Every 12.2 statement here
    is an exact-pin observation plus this checkout's commit history. The v12-side
    source analysis — what its `ginfuncs.c` does with `PageGetSpecialPointer` on a
    zeroed page, and what its `GinPageIsRecyclable` macro compares against — belongs
    on a v12 page and is not filed anywhere yet.
16. **`bloat_pct` has no actionable threshold, and was scored on eight fixtures.**
    The column was run against `REINDEX` on the seven published fixtures plus one
    live-pending-list fixture, on both majors; the two indexes that are known to
    break the bound, `k5_gin` and `fh1_gin`, were scored by the earlier pass's
    harness computing the same `waste + slack` arithmetic, not by this column, and
    their corpus was not rebuilt here. More importantly, no reading of the column is
    known to mean "rebuild this index": its floor is a fresh build's fill, which runs
    from 50.16% to 72.11% across opclasses and moves with the rebuild's
    `maintenance_work_mem` (numbers 5 and 14), so a threshold would have to be
    per-opclass and per-budget. What a baseline reading costs to establish — a
    rebuilt twin, or a stored history of the same index — was not measured.

17. **The proposed query guards are not implemented or executed.** The current
    statement still exposes component slack and `payload_bytes` for unsupported
    formats, assigns unrecognized classes through `ELSE 'entry'`, and has no
    explicit result status for invalid metapage or header state. The size-bracket
    snippet still divides by 8192. These are identified limitations, not successful
    test outcomes; the revised plan requires a separately validated SQL change.
18. **An exact comparison needs a tested consistency protocol.** The review
    verified per-call raw-reader locking, but did not implement or test a protocol
    that keeps the intended relation and its contents stable throughout the
    measurement. The required controls must cover maintenance and replacement as
    well as application writes. Agreeing size, metapage, or repeated census
    readings do not prove that protocol exists
    ([rawpage.c#get_raw_page_internal](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L198)).
19. **The revised acceptance cases have not been run.** In particular, this pass
    adds no measurements for disabled index cleanup, unsupported-format output
    guards, unknown flags, or target replacement during a scan. The existing
    measurements also do not prove any general upper bound after adding pending
    bytes or an estimated dead-entry term. Keep each future implementation's
    exact SQL and results together before updating the verification state.

## Source References

- [src/backend/catalog/index.c](../../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789) — replacement storage and rebuild in `reindex_index`.
- [src/backend/access/gin/ginvacuum.c](../../../../raw/postgres-17/src/backend/access/gin/ginvacuum.c#L507-L558) — reconstructing an entry tuple after deleting all posting-list items.
- [src/backend/access/gin/gininsert.c](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L276-L406) — memory-triggered accumulator flush and heap-scan build.
- [src/backend/access/index/indexam.c](../../../../raw/postgres-17/src/backend/access/index/indexam.c#L748-L777) — index vacuum callback dispatch.
- [src/backend/access/heap/vacuumlazy.c](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L392-L401) and [vacuumlazy.c](../../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2323-L2335) — disabled cleanup and failsafe paths.
- [src/backend/commands/vacuum.c](../../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2178) — the table cleanup option.
- [src/backend/utils/adt/dbsize.c](../../../../raw/postgres-17/src/backend/utils/adt/dbsize.c#L301-L370) — fork-specific file-length calculation.
- [src/backend/optimizer/prep/prepjointree.c](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1689-L1701) and [prepjointree.c](../../../../raw/postgres-17/src/backend/optimizer/prep/prepjointree.c#L1772-L1781) — subquery pull-up restrictions.
- [contrib/pageinspect/Makefile](../../../../raw/postgres-17/contrib/pageinspect/Makefile#L3-L35) — decoder objects, SQL upgrades, and regression targets.
- [contrib/amcheck/Makefile](../../../../raw/postgres-17/contrib/amcheck/Makefile#L3-L13) — heap and B-tree verification objects and tests.
- [src/include/catalog/Makefile](../../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143) — generated catalog-header dependency rules.
- [configure.ac](../../../../raw/postgres-17/configure.ac#L267-L288) — configured block sizes.
- [contrib/pgstattuple/pgstattuple.c](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L236-L308) — `pgstat_relation`'s relkind/AM dispatch and the GIN refusal.
- [contrib/pgstattuple/pgstattuple.c:544](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple.c#L544) — `BAS_BULKREAD` for index scans.
- [contrib/pgstattuple/pgstatindex.c](../../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L484-L577) — `pgstatginindex` and its metapage-only read.
- [contrib/pgstattuple/pgstattuple--1.4--1.5.sql](../../../../raw/postgres-17/contrib/pgstattuple/pgstattuple--1.4--1.5.sql#L49-L57) — the `pg_stat_scan_tables` grant for `pgstatginindex(regclass)`.
- [contrib/pageinspect/rawpage.c](../../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L141-L234) — `get_raw_page_internal` and `get_page_from_raw`.
- [contrib/pageinspect/ginfuncs.c](../../../../raw/postgres-17/contrib/pageinspect/ginfuncs.c#L30-L285) — all three GIN inspection functions.
- [contrib/pageinspect/pageinspect--1.9--1.10.sql](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.9--1.10.sql#L10-L21) — the `page_header` signature in use, `int` where 12.2 has `smallint`.
- [contrib/pageinspect/pageinspect--1.8--1.9.sql](../../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.8--1.9.sql#L46-L58) — `get_raw_page`'s block number widened to `int8`, which is why 12.2 needs the `::int` cast.
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
- [src/backend/storage/freespace/indexfsm.c](../../../../raw/postgres-17/src/backend/storage/freespace/indexfsm.c#L14-L74) — the index FSM's used/unused convention, `GetFreeIndexPage` and `RecordFreeIndexPage`.
- [src/backend/storage/freespace/freespace.c](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L36-L66) — the FSM category table and `MaxFSMRequestSize`.
- [src/backend/storage/freespace/freespace.c](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L118-L204) — `GetPageWithFreeSpace` and `RecordPageWithFreeSpace`'s bottom-level-only warning.
- [src/backend/storage/freespace/freespace.c](../../../../raw/postgres-17/src/backend/storage/freespace/freespace.c#L648-L691) — `fsm_set_and_search` and `fsm_search`, which starts at the root.
- [src/backend/storage/freespace/README](../../../../raw/postgres-17/src/backend/storage/freespace/README#L183-L188) — when upper-level FSM nodes are brought up to date.
- [src/backend/storage/page/bufpage.c](../../../../raw/postgres-17/src/backend/storage/page/bufpage.c#L898-L973) — `PageGetFreeSpace` and `PageGetExactFreeSpace`.
- [src/include/storage/bufpage.h](../../../../raw/postgres-17/src/include/storage/bufpage.h#L211-L234) — `SizeOfPageHeaderData` and `PageIsNew`.
- [src/backend/storage/buffer/freelist.c](../../../../raw/postgres-17/src/backend/storage/buffer/freelist.c#L314-L341) — the clock sweep that decrements usage counts and takes the first zero.
- [src/include/storage/buf_internals.h](../../../../raw/postgres-17/src/include/storage/buf_internals.h#L72-L79) — `BM_MAX_USAGE_COUNT` and why it is small.
- [src/backend/storage/buffer/bufmgr.c](../../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L2700-L2705) — `PinBuffer` raising the usage count on an unstrategised read.
- [src/backend/utils/misc/pg_controldata.c](../../../../raw/postgres-17/src/backend/utils/misc/pg_controldata.c#L203-L228) — `pg_control_init`, source of `max_data_alignment` and `database_block_size`.
- [src/backend/access/gin/gininsert.c](../../../../raw/postgres-17/src/backend/access/gin/gininsert.c#L283-L303) — `ginBuildCallback` dumping the accumulator when `maintenance_work_mem` is reached.
- [src/include/access/gin_private.h](../../../../raw/postgres-17/src/include/access/gin_private.h#L425-L440) — `BuildAccumulator`, including the `allocatedMemory` the flush rule tests.
- [src/backend/access/gin/ginbulk.c](../../../../raw/postgres-17/src/backend/access/gin/ginbulk.c#L120-L190) — the accumulator's `GetMemoryChunkSpace` accounting.
- [src/backend/utils/mmgr/mcxt.c](../../../../raw/postgres-17/src/backend/utils/mmgr/mcxt.c#L718-L730) — `GetMemoryChunkSpace`, which counts allocation overhead inside the build budget.
- [src/backend/utils/misc/guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2466-L2474) — `maintenance_work_mem`, `PGC_USERSET`, default 65536 kB.
- [src/backend/utils/misc/guc_tables.c](../../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3359-L3366) — `autovacuum_vacuum_insert_threshold`, `PGC_SIGHUP`, absent on 12.2.
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
- [v12/index](../../../v12/index.md) — the pin the 12.2 column was measured against
- [v12: Physical Index Statistics, Tuple Counts, and Bytes per Tuple (unverified)](../../../v12/questions/indexing/physical-index-statistics-tuple-counts-and-bytes.md) — the v12-side GIN page and metapage fields, cited against the v12 checkout
