---
type: question
version: 19
pinned_commit: 9a60f295bcb186a729d04e76377b7f122b2a1dd9
verified: false
verified_by_agent: not yet
---

# PostgreSQL 19 Autovacuum and VACUUM: Parallel Workers, Table Scoring, and Setting Pages All-Visible During Reads (unverified)

## Contents

- [Question](#question)
- [Short Answer](#short-answer)
- [1. Parallel autovacuum workers](#1-parallel-autovacuum-workers)
  - [What it does](#what-it-does)
  - [How a table gets parallel workers (normal path)](#how-a-table-gets-parallel-workers-normal-path)
  - [The per-autovacuum-worker cap](#the-per-autovacuum-worker-cap)
  - [Cost-delay propagation to parallel workers (new infrastructure)](#cost-delay-propagation-to-parallel-workers-new-infrastructure)
  - [When no parallelism happens (edge cases)](#when-no-parallelism-happens-edge-cases)
  - [GUC / reloption scope](#guc--reloption-scope)
  - [Tests](#tests)
- [2. Autovacuum table scoring (prioritization)](#2-autovacuum-table-scoring-prioritization)
  - [What it does](#what-it-does-1)
  - [How the score is computed](#how-the-score-is-computed)
  - [How tables are sorted](#how-tables-are-sorted)
  - [The five score-weight GUCs (and the escape hatch)](#the-five-score-weight-gucs-and-the-escape-hatch)
  - [Monitoring: `pg_stat_autovacuum_scores`](#monitoring-pgstatautovacuumscores)
  - [Not the same as the launcher's database scheduling](#not-the-same-as-the-launchers-database-scheduling)
  - [Tests](#tests-1)
- [3. Setting pages all-visible during read-only query scans](#3-setting-pages-all-visible-during-read-only-query-scans)
  - [What it does](#what-it-does-2)
  - [Normal path](#normal-path)
  - [How the VM bits get set](#how-the-vm-bits-get-set)
  - [The read-query safety valve](#the-read-query-safety-valve)
  - [Why this reduces future vacuuming work](#why-this-reduces-future-vacuuming-work)
  - [Edge cases](#edge-cases)
  - [Tests](#tests-2)
- [Source Commit History](#source-commit-history)
  - [Parallel autovacuum workers](#parallel-autovacuum-workers)
  - [Autovacuum table scoring](#autovacuum-table-scoring)
  - [Setting pages all-visible during reads](#setting-pages-all-visible-during-reads)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Source References](#source-references)
- [Open Questions](#open-questions)
- [Related Pages](#related-pages)

## Question

> Light-copyedit note: the prompt below is restated with three small wording fixes you approved — the GUC name is shown in code font, "vacuum"/"visible" are made precise ("VACUUM" the command; "all-visible in the visibility map"), and a terminal period is added. No meaning was changed.

Follow AGENTS.md. In PostgreSQL 19, create a question document to explain all these subjects related to autovacuum: Autovacuum can now use parallel workers, which can be configured with the new `autovacuum_max_parallel_workers` setting, and a new autovacuum scoring system helps prioritize tables to vacuum. PostgreSQL 19 further enhances VACUUM with a new strategy that can automatically reduce future vacuuming work by marking pages as all-visible in the visibility map while they're being queried.

## Short Answer

All three statements are true for the pinned PostgreSQL 19 development line (post-`REL_19_BETA1` `master` commit `9a60f295bcb186a729d04e76377b7f122b2a1dd9`). They are three independent features:

1. **Parallel autovacuum.** An autovacuum worker can now hand its **index** vacuuming and index cleanup phases to parallel workers, exactly like manual `VACUUM (PARALLEL)`. The new GUC `autovacuum_max_parallel_workers` caps how many parallel workers a *single* autovacuum worker may use; it defaults to `0`, which keeps autovacuum serial. A per-table reloption `autovacuum_parallel_workers` requests a specific degree [autovacuum.c#parallel-workers](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2940-L2959) [vacuumparallel.c#compute-workers](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L741-L802) [config.sgml#autovacuum_max_parallel_workers](../../../raw/postgres-19/doc/src/sgml/config.sgml#L9715-L9736).

2. **Table scoring (prioritization).** When an autovacuum worker has collected the tables that need work in a database, it now computes a numeric **score** per table and processes them in descending score order, so the most urgent tables go first. Five `*_score_weight` GUCs tune the components, and setting all five to `0.0` disables sorting entirely [autovacuum.c#AutoVacuumScores](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L330-L343) [autovacuum.c#sort-by-score](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2310-L2321) [autovacuum.c#score-comparator](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1906-L1917).

3. **Setting pages all-visible during read-only scans.** Ordinary read-only query table scans (seq scan, index scan, index-only scan, sample scan, TID-range scan, bitmap heap scan) can now set the all-visible (and possibly all-frozen) bits in the visibility map (VM) as a side effect of on-access pruning. Previously only `VACUUM` and `COPY ... FREEZE` set those bits. Pages already marked all-visible/all-frozen are skipped by future `VACUUM`, which is how this "reduces future vacuuming work" [pruneheap.c#heap_page_prune_opt](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L242-L360) [execUtils.c#ScanRelIsReadOnly](../../../raw/postgres-19/src/backend/executor/execUtils.c#L739-L758) [vacuumlazy.c#vm-skip](../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L41-L67).

The three features touch different layers (the autovacuum daemon, the autovacuum scheduling logic, and the heap access method's read path) and can be reasoned about separately. Each is detailed below, followed by the per-feature [Source Commit History](#source-commit-history).

---

## 1. Parallel autovacuum workers

### What it does

Parallel vacuum (parallelizing the **index** bulk-delete and index cleanup phases across worker processes) has existed for manual `VACUUM` since v13. PostgreSQL 19 lets the autovacuum daemon use the same machinery. The file that implements it now says so in its header: it is "Support routines for parallel vacuum and autovacuum execution" [vacuumparallel.c#header](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L1-L34).

Only the index phases are parallelized; the heap scan is not. The new GUC documentation makes this explicit: the limit "applies specifically to the index vacuuming and index cleanup phases," and it is "the per-autovacuum worker equivalent of the `PARALLEL` option of the `VACUUM` command" [config.sgml#autovacuum_max_parallel_workers](../../../raw/postgres-19/doc/src/sgml/config.sgml#L9715-L9736).

### How a table gets parallel workers (normal path)

`VacuumParams` carries an `nworkers` field with these semantics: `0` = choose automatically from the number of indexes, `-1` = parallelism disabled, `> 0` = request that many [vacuum.h#nworkers](../../../raw/postgres-19/src/include/commands/vacuum.h#L245-L250).

For autovacuum, `table_recheck_autovac()` fills `nworkers` from the per-table reloption `autovacuum_parallel_workers`:

- reloption `0` → `nworkers = -1` (parallel vacuum disabled for this table);
- reloption `> 0` → `nworkers =` that value;
- reloption `-1` (the default) → falls through with `nworkers = 0` (automatic)

[autovacuum.c#nworkers-from-reloption](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2940-L2959).

The reloption itself is defined as a heap reloption with default `-1`, minimum `-1`, maximum `1024`, taking `ShareUpdateExclusiveLock` to set (so `ALTER TABLE ... SET (autovacuum_parallel_workers = N)` does not block reads/writes and takes effect on the next autovacuum of the table) [reloptions.c#reloption-def](../../../raw/postgres-19/src/backend/access/common/reloptions.c#L239-L247) [reloptions.c#reloption-parse](../../../raw/postgres-19/src/backend/access/common/reloptions.c#L1981-L1982).

### The per-autovacuum-worker cap

`parallel_vacuum_compute_workers()` is where the degree is finally chosen. The key v19 change is the source of the cap: an autovacuum worker uses `autovacuum_max_parallel_workers`, while a regular backend running manual `VACUUM` uses `max_parallel_maintenance_workers` [vacuumparallel.c#compute-workers](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L741-L802):

```c
max_workers = AmAutoVacuumWorkerProcess() ?
    autovacuum_max_parallel_workers :
    max_parallel_maintenance_workers;

/* ... */
if (!IsUnderPostmaster || max_workers == 0)
    return 0;
```

So a `max_workers` of `0` short-circuits to no parallelism. Because `autovacuum_max_parallel_workers` defaults to `0`, **autovacuum is serial out of the box**; you opt in by raising the GUC [globals.c#default-0](../../../raw/postgres-19/src/backend/utils/init/globals.c#L148). The function also requires at least two indexes large enough (`> min_parallel_index_scan_size`) and that support parallel vacuum, then caps the result by `max_workers` [vacuumparallel.c#cap](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L765-L801). The total is further bounded system-wide by `max_parallel_workers` and the available background-worker slots, as the docs note [config.sgml#further-limited](../../../raw/postgres-19/doc/src/sgml/config.sgml#L9728-L9729).

### Cost-delay propagation to parallel workers (new infrastructure)

Manual `VACUUM` workers inherit static cost-delay settings, but an autovacuum leader's cost-delay parameters can change mid-table (config reload, or the autovacuum balancing algorithm). So v19 added a shared-memory struct, `PVSharedCostParams`, with a generation counter. The leader updates the shared parameters and bumps the generation; each parallel worker polls the counter and refreshes its local `VacuumCost*` state when it changes [vacuumparallel.c#cost-params-header](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L21-L26) [vacuumparallel.c#propagate](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L644-L726). The refresh happens at the vacuum delay point, before the cost check, so a newly enabled delay takes effect promptly [vacuum.c#delay-point-refresh](../../../raw/postgres-19/src/backend/commands/vacuum.c#L2448-L2456).

### When no parallelism happens (edge cases)

- `autovacuum_max_parallel_workers = 0` (default) → serial [vacuumparallel.c#compute-workers](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L759-L760).
- Per-table `autovacuum_parallel_workers = 0` → `nworkers = -1` → parallel disabled for that table [autovacuum.c#nworkers-from-reloption](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2944-L2950).
- Fewer than two suitable, large-enough indexes → degree resolves to `0` [vacuumparallel.c#index-eligibility](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L762-L792).
- Standalone backend (`!IsUnderPostmaster`) → no parallelism [vacuumparallel.c#standalone](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L755-L760).

### GUC / reloption scope

| Setting | Kind | Context → apply scope | Default | Range |
|---|---|---|---|---|
| `autovacuum_max_parallel_workers` | GUC `int` | `PGC_SIGHUP` → **reload** | `0` | `0` … `MAX_PARALLEL_WORKER_LIMIT` |
| `autovacuum_parallel_workers` | table reloption `int` | `ShareUpdateExclusiveLock` → set via `ALTER TABLE`; effective next autovacuum | `-1` (auto) | `-1` … `1024` |

`autovacuum_max_parallel_workers` is `PGC_SIGHUP`, so a config reload (`SELECT pg_reload_conf()` or `SIGHUP`) is enough — no restart [guc_parameters.dat#max_parallel_workers](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L173-L179) [miscadmin.h#extern](../../../raw/postgres-19/src/include/miscadmin.h#L183).

### Tests

The core commit shipped a dedicated TAP test module, `src/test/modules/test_autovacuum/`, whose single test `t/001_parallel_autovacuum.pl` drives autovacuum with `autovacuum_max_parallel_workers` set and checks that parallel workers run [001_parallel_autovacuum.pl](../../../raw/postgres-19/src/test/modules/test_autovacuum/t/001_parallel_autovacuum.pl#L1-L40).

---

## 2. Autovacuum table scoring (prioritization)

### What it does

Before v19, an autovacuum worker processed the tables that needed work in the order it found them while scanning `pg_class`. v19 adds "rudimentary table prioritization": the worker computes a **score** for every candidate table and sorts the work list so higher-scoring (more urgent) tables are vacuumed/analyzed first. This changes only the **order** of processing within one worker's pass; it does not change *which* tables are eligible, nor the launcher's choice of which database to visit next.

### How the score is computed

`relation_needs_vacanalyze()` is the function that decides whether a table needs vacuum/analyze. v19 extended it to also fill an `AutoVacuumScores` out-parameter — a `max` plus five component scores [autovacuum.c#AutoVacuumScores](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L330-L343) [autovacuum.c#relation_needs_vacanalyze](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3067-L3081):

```c
typedef struct
{
    double  max;       /* maximum of all values below */
    double  xid;       /* transaction ID component */
    double  mxid;      /* multixact ID component */
    double  vac;       /* vacuum component */
    double  vac_ins;   /* vacuum insert component */
    double  anl;       /* analyze component */
} AutoVacuumScores;
```

Each component is roughly "how far past its threshold is this table," scaled by a weight:

- **Freeze components.** `xid`/`mxid` = age divided by the respective `*_freeze_max_age`, then multiplied by the weight. If a table is past its (weight-adjusted) failsafe age, the score is raised aggressively via `pow()` so near-wraparound tables dominate [autovacuum.c#freeze-score](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3198-L3243). The `mxid` divisor is the per-table-clamped `effective_multixact_freeze_max_age` (from `MultiXactMemberFreezeThreshold()`), which is driven toward `0` as multixact member space fills up; v19 therefore guards it as `Max(1, multixact_freeze_max_age)`, so when the threshold reaches `0` the `mxid` component degrades to the raw multixact age instead of dividing by zero (a post-beta1 fix, `1f2297b5487`) [autovacuum.c#mxid-divisor-guard](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3198-L3208) [autovacuum.c#mxid-max-age-clamp](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3172-L3174).
- **Dead-tuple / insert / analyze components.** `vac` = dead tuples ÷ vacuum threshold, `vac_ins` = inserts ÷ insert threshold, `anl` = modified tuples ÷ analyze threshold, each multiplied by its weight [autovacuum.c#vac-ins-anl-score](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3292-L3320).

`scores->max` is the running maximum of the components, and that single number is the table's score. Note that the eligibility decision (`*dovacuum` / `*doanalyze`) is still threshold-based and independent of the weights — weights only affect ordering, except that anti-wraparound (`force_vacuum`) still forces a vacuum regardless [autovacuum.c#force-vacuum](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3244-L3245).

### How tables are sorted

`do_autovacuum()` builds a list of `TableToProcess { oid, score }`, storing `scores.max` for each table that needs work [autovacuum.c#TableToProcess](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L345-L352) [autovacuum.c#collect](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2079-L2094). After collecting both ordinary tables and TOAST tables, it sorts the list in **descending** score order [autovacuum.c#comparator](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1906-L1917):

```c
return (t2->score < t1->score) ? -1 : (t2->score > t1->score) ? 1 : 0;
```

### The five score-weight GUCs (and the escape hatch)

There is one weight per component, all `real`, `PGC_SIGHUP` (reload), default `1.0`, range `0.0`–`10.0`:

| GUC | Component | Definition |
|---|---|---|
| `autovacuum_freeze_score_weight` | `xid` | [guc_parameters.dat#L165-L171](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L165-L171) |
| `autovacuum_multixact_freeze_score_weight` | `mxid` | [guc_parameters.dat#L198-L204](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L198-L204) |
| `autovacuum_vacuum_score_weight` | `vac` | [guc_parameters.dat#L276-L282](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L276-L282) |
| `autovacuum_vacuum_insert_score_weight` | `vac_ins` | [guc_parameters.dat#L242-L248](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L242-L248) |
| `autovacuum_analyze_score_weight` | `anl` | [guc_parameters.dat#L139-L145](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L139-L145) |

Setting **all five** weights to `0.0` is a deliberate escape hatch: `do_autovacuum()` skips `list_sort()` entirely in that case, restoring the old unsorted behavior [autovacuum.c#escape-hatch](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2310-L2321).

> Documentation note: the v19 release notes' link text spells one GUC `vacuum_insert_score_weight`, but the actual parameter is `autovacuum_vacuum_insert_score_weight`. Per AGENTS.md, source wins; the variable is defined as `autovacuum_vacuum_insert_score_weight` [autovacuum.c#weight-globals](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L137-L141).

### Monitoring: `pg_stat_autovacuum_scores`

A companion commit added a system view, `pg_stat_autovacuum_scores`, exposing each table's overall and component scores plus the do-vacuum/do-analyze/for-wraparound decisions, sourced from `pg_stat_get_autovacuum_scores()` [system_views.sql#pg_stat_autovacuum_scores](../../../raw/postgres-19/src/backend/catalog/system_views.sql#L798-L814):

```sql
CREATE VIEW pg_stat_autovacuum_scores AS
    SELECT s.oid AS relid, n.nspname AS schemaname, c.relname AS relname,
           s.score, s.xid_score, s.mxid_score, s.vacuum_score,
           s.vacuum_insert_score, s.analyze_score,
           s.do_vacuum, s.do_analyze, s.for_wraparound
    FROM pg_stat_get_autovacuum_scores() s
    JOIN pg_class c on c.oid = s.oid
    LEFT JOIN pg_namespace n ON n.oid = c.relnamespace;
```

### Not the same as the launcher's database scheduling

Do not confuse the new per-table score with the autovacuum *launcher's* long-standing per-database ordering. The launcher keeps an `avl_dbase` list with an `adl_score` field used only to round-robin databases when choosing which one to send the next worker to [autovacuum.c#avl_dbase](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L181-L187) [autovacuum.c#db_comparator](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1118-L1123). That `adl_score` is unrelated to v19's table `AutoVacuumScores`.

### Tests

The scoring *algorithm* itself ships without a dedicated regression or TAP test — the core commit touched only C, GUC data, docs, and headers (see [Source Commit History](#source-commit-history)). The `pg_stat_autovacuum_scores` *view definition* is covered by the standard `rules` regression test [rules.out#view](../../../raw/postgres-19/src/test/regress/expected/rules.out#L1863).

---

## 3. Setting pages all-visible during read-only query scans

### What it does

PostgreSQL maintains a per-relation visibility map (VM) with two bits per heap page: all-visible and all-frozen. These bits let index-only scans skip heap fetches and let `VACUUM` skip pages. Historically only `VACUUM` and `COPY ... FREEZE` set them. v19 lets ordinary **read-only query scans** set the all-visible bit (and, when eligible, the all-frozen bit) as a side effect of *on-access pruning*, the opportunistic page cleanup that already happens while reading. The release note phrases it as "Allow query table scans to mark pages as all-visible in the visibility map."

### Normal path

1. **Plan time / scan setup — is the scan read-only?** `ScanRelIsReadOnly()` returns true when the scanned relation is neither a result (modified) relation nor a row-mark relation [executor.h#decl](../../../raw/postgres-19/src/include/executor/executor.h#L698) [execUtils.c#ScanRelIsReadOnly](../../../raw/postgres-19/src/backend/executor/execUtils.c#L739-L758). All the heap-scanning executor nodes consult it: sequential [nodeSeqscan.c#read-only](../../../raw/postgres-19/src/backend/executor/nodeSeqscan.c#L67-L85), index and index-only [nodeIndexscan.c#read-only](../../../raw/postgres-19/src/backend/executor/nodeIndexscan.c#L117) [nodeIndexonlyscan.c#read-only](../../../raw/postgres-19/src/backend/executor/nodeIndexonlyscan.c#L99), sample [nodeSamplescan.c#read-only](../../../raw/postgres-19/src/backend/executor/nodeSamplescan.c#L302), TID-range [nodeTidrangescan.c#read-only](../../../raw/postgres-19/src/backend/executor/nodeTidrangescan.c#L249), and bitmap heap [nodeBitmapHeapscan.c#read-only](../../../raw/postgres-19/src/backend/executor/nodeBitmapHeapscan.c#L149).

2. **Flag travels on the scan descriptor.** A read-only seq scan begins with `SO_HINT_REL_READ_ONLY` set [nodeSeqscan.c#flag](../../../raw/postgres-19/src/backend/executor/nodeSeqscan.c#L67-L85).

3. **Heap read path forwards it to pruning.** When preparing a page for a page-mode scan, `heap_prepare_pagescan()` calls on-access pruning, passing the read-only hint through [heapam.c#heap_prepare_pagescan](../../../raw/postgres-19/src/backend/access/heap/heapam.c#L611-L640):

   ```c
   heap_page_prune_opt(scan->rs_base.rs_rd, buffer, &scan->rs_vmbuffer,
                       sscan->rs_flags & SO_HINT_REL_READ_ONLY);
   ```

   The index-fetch and other heap read entry points pass the flag the same way [heapam_handler.c#prune-opt](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L2568) [heapam_indexscan.c#prune-opt](../../../raw/postgres-19/src/backend/access/heap/heapam_indexscan.c#L260).

4. **On-access pruning requests VM-setting.** `heap_page_prune_opt()` gained a `rel_read_only` parameter; when true it adds `HEAP_PAGE_PRUNE_SET_VM` to the prune options [pruneheap.c#heap_page_prune_opt](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L242-L360) [heapam.h#flag-def](../../../raw/postgres-19/src/include/access/heapam.h#L45):

   ```c
   params.options = HEAP_PAGE_PRUNE_ALLOW_FAST_PATH;
   if (rel_read_only)
       params.options |= HEAP_PAGE_PRUNE_SET_VM;
   ```

### How the VM bits get set

Inside pruning, `HEAP_PAGE_PRUNE_SET_VM` becomes `PruneState.attempt_set_vm`, which seeds `set_all_visible`; `set_all_frozen` additionally requires that freezing is being attempted (only `VACUUM` freezes today, so a read scan sets all-visible but not all-frozen) [pruneheap.c#prune-state-init](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L405-L518). After scanning the page, `heap_page_will_set_vm()` decides whether to actually set the bits and, if so, what `new_vmbits` to write [pruneheap.c#will_set_vm](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L937-L984). The same `HEAP_PAGE_PRUNE_SET_VM` flag is what `VACUUM` passes, so both paths share one VM-setting implementation [vacuumlazy.c#vacuum-uses-flag](../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L2036).

### The read-query safety valve

The feature is careful not to make read queries pay VACUUM's write costs. For an on-access call that is not also pruning or freezing, `heap_page_will_set_vm()` declines to set the VM if doing so would newly dirty the heap page, or — if the page is already dirty — would force a full-page image (FPI) into WAL [pruneheap.c#safety-valve](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L959-L970):

```c
if (reason == PRUNE_ON_ACCESS && !do_prune && !do_freeze &&
    (!BufferIsDirty(prstate->buffer) || XLogCheckBufferNeedsBackup(prstate->buffer)))
{
    prstate->set_all_visible = prstate->set_all_frozen = false;
    return false;
}
```

So a read query opportunistically sets the VM mainly when it was already going to write the page (it pruned/froze) or when setting the bit is cheap.

### Why this reduces future vacuuming work

`VACUUM` uses the VM to skip pages: a normal vacuum may skip ranges of pages "marked all-visible (and even all-frozen) in the visibility map," scanning them only to break up small skippable ranges (`SKIP_PAGES_THRESHOLD`) or to freeze [vacuumlazy.c#vm-skip](../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L41-L67). Pages a read scan already marked all-visible are therefore candidates for being skipped by the next ordinary `VACUUM`, which would otherwise have to re-scan them, recheck visibility, and set the bit itself. Two precisions: a read scan sets all-visible but **not** all-frozen (only `VACUUM` freezes), so anti-wraparound vacuums must still visit these pages to freeze them — the saving here is on ordinary vacuums; and marking pages during reads also immediately benefits index-only scans, which can skip heap visibility checks for all-visible pages.

### Edge cases

- **Recovery.** On-access pruning (and thus VM-setting) is skipped during recovery, since standbys cannot write WAL for it [pruneheap.c#recovery](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L279-L285).
- **`INSERT ... SELECT` from the same table.** The scanned relation is *not* recorded as a result relation, so it is reported read-only even though the statement modifies it. The code comments note this is harmless in practice: pages it inserts into rarely meet the prune heuristic, and those that do usually have in-progress inserts that keep them from being all-visible [pruneheap.c#insert-select-note](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L259-L268) [execUtils.c#caveats](../../../raw/postgres-19/src/backend/executor/execUtils.c#L739-L758).
- **Row-marked relations.** If any relation in the query has a modifying row mark, the others get a `ROW_MARK_REFERENCE` and are conservatively reported *not* read-only [execUtils.c#rowmark-note](../../../raw/postgres-19/src/backend/executor/execUtils.c#L745-L758).

### Tests

The headline commit `b46e1e54d` changed only heap access-method code and headers; it added no dedicated regression or isolation test of its own (see [Source Commit History](#source-commit-history)). Related VM-corruption and pruning behavior is exercised by neighboring commits in the same cluster, but the on-access set-VM path itself relies on existing visibility-map and pruning tests rather than a new one.

---

## Source Commit History

All hashes are ancestors of the pinned post-`REL_19_BETA1` `master` commit `9a60f295bcb186a729d04e76377b7f122b2a1dd9`. Between `REL_19_BETA1` (tagged 2026-06-01) and this pin, the only scoring/parallel-vacuum/visibility feature-file change is the post-beta1 autovacuum MXID-score division-by-zero fix `1f2297b5487` (listed under [Autovacuum table scoring](#autovacuum-table-scoring) below); no parallel-vacuum or visibility-map/pruning feature files changed in that range. Dates are author dates.

### Parallel autovacuum workers

| Date | Commit | Subject |
|---|---|---|
| 2026-03-19 | `adcdbe93860` | Add parallel vacuum worker usage to VACUUM (VERBOSE) and autovacuum logs |
| 2026-04-06 | `1ff3180ca01` | **Allow autovacuum to use parallel vacuum workers** (core) |
| 2026-04-09 | `8030b839d3d` | Remove an unstable wait from parallel autovacuum regression test |
| 2026-04-10 | `2a3d2f9f68d` | doc: Improve consistency of parallel vacuum description |
| 2026-04-10 | `c22d115f1d1` | Fix unstable log verification in test_autovacuum |

The core commit `1ff3180ca01` (committed by Masahiko Sawada; the v19 release notes credit Daniil Davydov) added the `autovacuum_max_parallel_workers` GUC, the `autovacuum_parallel_workers` reloption, the autovacuum-vs-maintenance worker cap in `parallel_vacuum_compute_workers()`, the `PVSharedCostParams` cost-delay propagation, and the `test_autovacuum` TAP module [vacuumparallel.c#compute-workers](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L741-L802). `adcdbe93860` is supporting infrastructure that reports parallel-worker usage in `VACUUM (VERBOSE)` and autovacuum logs.

### Autovacuum table scoring

| Date | Commit | Subject |
|---|---|---|
| 2026-03-27 | `d7965d65fc5` | **Add rudimentary table prioritization to autovacuum** (core) |
| 2026-04-03 | `8261ee24fe3` | Refactor relation_needs_vacanalyze() |
| 2026-04-03 | `53b8ca6881a` | Teach relation_needs_vacanalyze() to always compute scores |
| 2026-04-06 | `87f61f0c828` | Add pg_stat_autovacuum_scores system view |
| 2026-04-09 | `71ff232a5bc` | Fix double-free in pg_stat_autovacuum_scores |
| 2026-06-18 | `1f2297b5487` | Avoid division-by-zero when calculating autovacuum MXID score (post-beta1) |

All by Nathan Bossart. `d7965d65fc5` introduced `AutoVacuumScores`, `TableToProcess`, the descending sort with its all-weights-zero escape hatch, and the five `*_score_weight` GUCs. `87f61f0c828` added the `pg_stat_autovacuum_scores` view and `pg_stat_get_autovacuum_scores()`; `71ff232a5bc` fixed a double-free in it. `1f2297b5487` is a post-beta1 fix that guards the `mxid` score divisor against a zero `effective_multixact_freeze_max_age` (see [How the score is computed](#how-the-score-is-computed)).

### Setting pages all-visible during reads

This feature is the v19 tail of a longer, multi-release pruning/freezing/visibility-map refactor by Melanie Plageman. The v19-cycle commits that build up to and include the headline change:

| Date | Commit | Subject |
|---|---|---|
| 2025-10-13 | `add323da40a` | Eliminate XLOG_HEAP2_VISIBLE from vacuum phase III |
| 2025-11-20 | `65ec565b19f` | Update PruneState.all_[visible|frozen] earlier in pruning |
| 2026-03-05 | `34cb4254bdb` | Prefix PruneState->all_{visible,frozen} with set_ |
| 2026-03-15 | `99bf1f8aa6c` | Save vmbuffer in heap-specific scan descriptors for on-access pruning |
| 2026-03-22 | `01b7e4a46d0` | Add pruning fast path for all-visible and all-frozen pages |
| 2026-03-22 | `4f7ecca84dd` | Detect and fix visibility map corruption in more cases |
| 2026-03-24 | `dd5716f3c74` | Use GlobalVisState in vacuum to determine page level visibility |
| 2026-03-24 | `9ba3ec076a6` | Keep newest live XID up-to-date even if page not all-visible |
| 2026-03-24 | `1252a4ee286` | WAL log VM setting during vacuum phase I in XLOG_HEAP2_PRUNE_VACUUM_SCAN |
| 2026-03-24 | `a759ced2f1e` | WAL log VM setting for empty pages in XLOG_HEAP2_PRUNE_VACUUM_SCAN |
| 2026-03-30 | `50eb5faea29` | Pass down information on table modification to scan nodes |
| 2026-03-30 | `378a216187a` | Set pd_prune_xid on insert |
| 2026-03-30 | `b46e1e54d07` | **Allow on-access pruning to set pages all-visible** (core) |

The headline commit `b46e1e54d07` added the `rel_read_only` parameter to `heap_page_prune_opt()` and the `HEAP_PAGE_PRUNE_SET_VM` option [pruneheap.c#heap_page_prune_opt](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L242-L360). `50eb5faea29` is the executor plumbing (`ScanRelIsReadOnly()` / `SO_HINT_REL_READ_ONLY`); `99bf1f8aa6c` saved the `rs_vmbuffer` so scans can pin the VM page; `01b7e4a46d0` added the all-visible/all-frozen fast path. This list is the cluster identified for the v19 cycle; it builds on the pre-v19 WAL-record unification (`f83d709760d`, 2024, Heikki Linnakangas) noted under [Open Questions](#open-questions).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v19/index.md`, and the last ~20 `wiki/log.md` entries for navigation and the v19 pin. The 2026-06-10 log note records an earlier orphan draft of this question that was removed at the user's request; this page is the filed replacement.
- Autovacuum daemon: `src/backend/postmaster/autovacuum.c` (score weights, `AutoVacuumScores`, `TableToProcess`, `relation_needs_vacanalyze`, `do_autovacuum`, comparator, `table_recheck_autovac` worker selection, launcher `avl_dbase`/`adl_score`).
- Parallel vacuum: `src/backend/commands/vacuumparallel.c` (`parallel_vacuum_compute_workers`, `PVSharedCostParams`), `src/backend/commands/vacuum.c` (delay-point refresh), `src/include/commands/vacuum.h` (`nworkers`).
- GUC/reloption definitions: `src/backend/utils/misc/guc_parameters.dat`, `src/backend/access/common/reloptions.c`, `src/backend/utils/init/globals.c`, `src/include/miscadmin.h`.
- Heap read/prune path: `src/backend/access/heap/pruneheap.c`, `heapam.c`, `heapam_handler.c`, `heapam_indexscan.c`, `vacuumlazy.c`, `src/include/access/heapam.h`.
- Executor scan nodes and `ScanRelIsReadOnly`: `src/backend/executor/execUtils.c`, `nodeSeqscan.c`, `nodeIndexscan.c`, `nodeIndexonlyscan.c`, `nodeSamplescan.c`, `nodeTidrangescan.c`, `nodeBitmapHeapscan.c`, `src/include/executor/executor.h`.
- Catalog/monitoring: `src/backend/catalog/system_views.sql`, `src/test/regress/expected/rules.out`.
- Same-checkout docs (`doc/src/sgml/config.sgml`, `release-19.sgml`) used to locate features and their commits; behavioral claims are cited to source.
- `git log` on the pinned checkout for the three commit clusters above.

## Evidence Map

| Claim | Evidence |
|---|---|
| Autovacuum can use parallel index-vacuum workers | [vacuumparallel.c#L1-L34](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L1-L34) [vacuumparallel.c#L741-L802](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L741-L802) |
| Cap source differs: autovacuum vs manual VACUUM | [vacuumparallel.c#L751-L753](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L751-L753) |
| `autovacuum_max_parallel_workers` GUC, PGC_SIGHUP, default 0 | [guc_parameters.dat#L173-L179](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L173-L179) [globals.c#L148](../../../raw/postgres-19/src/backend/utils/init/globals.c#L148) [config.sgml#L9715-L9736](../../../raw/postgres-19/doc/src/sgml/config.sgml#L9715-L9736) |
| Per-table `autovacuum_parallel_workers` reloption; -1/0/N mapping | [reloptions.c#L239-L247](../../../raw/postgres-19/src/backend/access/common/reloptions.c#L239-L247) [autovacuum.c#L2940-L2959](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2940-L2959) [vacuum.h#L245-L250](../../../raw/postgres-19/src/include/commands/vacuum.h#L245-L250) |
| Cost-delay propagation to parallel autovacuum workers | [vacuumparallel.c#L21-L26](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L21-L26) [vacuumparallel.c#L644-L726](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L644-L726) [vacuum.c#L2448-L2456](../../../raw/postgres-19/src/backend/commands/vacuum.c#L2448-L2456) |
| Parallel autovacuum TAP test | [001_parallel_autovacuum.pl#L1-L40](../../../raw/postgres-19/src/test/modules/test_autovacuum/t/001_parallel_autovacuum.pl#L1-L40) |
| Per-table score struct and components | [autovacuum.c#L330-L343](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L330-L343) [autovacuum.c#L3067-L3081](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3067-L3081) |
| Freeze/dead/insert/analyze score formulas; `mxid` divisor guarded by `Max(1, ...)` (post-beta1 `1f2297b5487`) | [autovacuum.c#L3198-L3243](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3198-L3243) [autovacuum.c#L3292-L3320](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3292-L3320) [autovacuum.c#L3172-L3174](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3172-L3174) |
| Descending sort; all-weights-zero escape hatch | [autovacuum.c#L1906-L1917](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1906-L1917) [autovacuum.c#L2310-L2321](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2310-L2321) |
| Five `*_score_weight` GUCs (PGC_SIGHUP, 1.0, 0.0–10.0) | [guc_parameters.dat#L139-L145](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L139-L145) [guc_parameters.dat#L165-L171](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L165-L171) [guc_parameters.dat#L242-L248](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L242-L248) |
| `pg_stat_autovacuum_scores` view | [system_views.sql#L798-L814](../../../raw/postgres-19/src/backend/catalog/system_views.sql#L798-L814) [rules.out#L1863](../../../raw/postgres-19/src/test/regress/expected/rules.out#L1863) |
| Launcher `adl_score` is separate database scheduling | [autovacuum.c#L181-L187](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L181-L187) [autovacuum.c#L1118-L1123](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1118-L1123) |
| Read-only scans flagged via `ScanRelIsReadOnly` | [execUtils.c#L739-L758](../../../raw/postgres-19/src/backend/executor/execUtils.c#L739-L758) [nodeSeqscan.c#L67-L85](../../../raw/postgres-19/src/backend/executor/nodeSeqscan.c#L67-L85) |
| Flag forwarded into on-access pruning → SET_VM | [heapam.c#L611-L640](../../../raw/postgres-19/src/backend/access/heap/heapam.c#L611-L640) [pruneheap.c#L242-L360](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L242-L360) [heapam.h#L45](../../../raw/postgres-19/src/include/access/heapam.h#L45) |
| VM-bit decision + read-query safety valve | [pruneheap.c#L405-L518](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L405-L518) [pruneheap.c#L937-L984](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L937-L984) |
| Future VACUUM skips all-visible/all-frozen pages | [vacuumlazy.c#L41-L67](../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L41-L67) |

## Source References

- [autovacuum.c](../../../raw/postgres-19/src/backend/postmaster/autovacuum.c) — autovacuum daemon: score-weight globals, `AutoVacuumScores`/`TableToProcess`, `relation_needs_vacanalyze`, `do_autovacuum` collection and descending sort, `table_recheck_autovac` parallel-worker selection, and the launcher's separate `avl_dbase`/`adl_score`.
- [vacuumparallel.c](../../../raw/postgres-19/src/backend/commands/vacuumparallel.c) — parallel vacuum/autovacuum support: `parallel_vacuum_compute_workers` cap, `PVSharedCostParams` cost-delay propagation.
- [vacuum.c](../../../raw/postgres-19/src/backend/commands/vacuum.c) / [vacuum.h](../../../raw/postgres-19/src/include/commands/vacuum.h) — `VacuumParams.nworkers` semantics and the delay-point cost refresh.
- [reloptions.c](../../../raw/postgres-19/src/backend/access/common/reloptions.c) — `autovacuum_parallel_workers` heap reloption definition and parse mapping.
- [guc_parameters.dat](../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat) / [globals.c](../../../raw/postgres-19/src/backend/utils/init/globals.c) / [miscadmin.h](../../../raw/postgres-19/src/include/miscadmin.h) — `autovacuum_max_parallel_workers` and the five `*_score_weight` GUC definitions and defaults.
- [pruneheap.c](../../../raw/postgres-19/src/backend/access/heap/pruneheap.c) — `heap_page_prune_opt` read-only path, `PruneState` VM fields, `heap_page_will_set_vm` decision and on-access safety valve.
- [heapam.c](../../../raw/postgres-19/src/backend/access/heap/heapam.c) / [heapam_handler.c](../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c) / [heapam_indexscan.c](../../../raw/postgres-19/src/backend/access/heap/heapam_indexscan.c) / [heapam.h](../../../raw/postgres-19/src/include/access/heapam.h) — heap read paths forwarding `SO_HINT_REL_READ_ONLY` and the `HEAP_PAGE_PRUNE_SET_VM` flag.
- [vacuumlazy.c](../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c) — VACUUM's VM-skip behavior (the future-work reduction) and its own use of `HEAP_PAGE_PRUNE_SET_VM`.
- [execUtils.c](../../../raw/postgres-19/src/backend/executor/execUtils.c) / [executor.h](../../../raw/postgres-19/src/include/executor/executor.h) and the scan nodes [nodeSeqscan.c](../../../raw/postgres-19/src/backend/executor/nodeSeqscan.c), [nodeIndexscan.c](../../../raw/postgres-19/src/backend/executor/nodeIndexscan.c), [nodeIndexonlyscan.c](../../../raw/postgres-19/src/backend/executor/nodeIndexonlyscan.c), [nodeSamplescan.c](../../../raw/postgres-19/src/backend/executor/nodeSamplescan.c), [nodeTidrangescan.c](../../../raw/postgres-19/src/backend/executor/nodeTidrangescan.c), [nodeBitmapHeapscan.c](../../../raw/postgres-19/src/backend/executor/nodeBitmapHeapscan.c) — `ScanRelIsReadOnly` and where read-only scans set the hint.
- [system_views.sql](../../../raw/postgres-19/src/backend/catalog/system_views.sql) / [rules.out](../../../raw/postgres-19/src/test/regress/expected/rules.out) — `pg_stat_autovacuum_scores` view and its regression coverage.
- [001_parallel_autovacuum.pl](../../../raw/postgres-19/src/test/modules/test_autovacuum/t/001_parallel_autovacuum.pl) — parallel autovacuum TAP test.
- [config.sgml](../../../raw/postgres-19/doc/src/sgml/config.sgml) — same-checkout documentation for `autovacuum_max_parallel_workers` (used to locate the feature; behavior cited to source).

## Open Questions

- **On-access VM commit list is a cluster, not a verified-exhaustive count.** Unlike the self-contained `pg_plan_advice` and `REPACK` features, "setting pages all-visible during reads" is the v19 portion of a long-running pruning/freezing/VM refactor. The table above lists the v19-cycle commits I identified as building the capability; it is not claimed to be exhaustive, and it explicitly excludes the pre-v19 foundation `f83d709760d` (2024, "Merge prune, freeze and vacuum WAL record formats"). A full commit-by-commit attribution would need a deeper history pass.
- **Score tuning guidance is not derivable from source.** The source defines the score math and weight ranges but gives no recommended non-default weights; choosing weights for a given workload is an operational question outside this checkout.
- **Verification status.** This page is filed from a claim-to-source map against pinned raw source but has not had an independent claim-by-claim re-verification pass, so `verified_by_agent` remains `not yet` and the title keeps `(unverified)`.

## Related Pages

- [v19/index](../index.md) — PostgreSQL 19 version landing page.
- [versions](../../versions.md) — version index and source pin manifest.
- [wiki index](../../index.md) — global catalog.
