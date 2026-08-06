---
type: question
version: 19
pinned_commit: 99e47536bbf1a165f5dc8d504f928821ebc8df6a
verified: false
verified_by_agent: not yet
---

# PostgreSQL 19 Autovacuum and VACUUM: Parallel Workers, Table Scoring, and Setting Pages All-Visible During Reads (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short Answer](#short-answer)
  - [1. Parallel autovacuum workers](#1-parallel-autovacuum-workers)
  - [2. Autovacuum table scoring (prioritization)](#2-autovacuum-table-scoring-prioritization)
  - [3. Setting pages all-visible during read-only query scans](#3-setting-pages-all-visible-during-read-only-query-scans)
  - [Source Commit History](#source-commit-history)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Source References](#source-references)
- [Open Questions](#open-questions)
- [Navigation](#navigation)

## Question

> Light-copyedit note: the prompt below is restated with three small wording fixes you approved — the GUC name is shown in code font, "vacuum"/"visible" are made precise ("VACUUM" the command; "all-visible in the visibility map"), and a terminal period is added. No meaning was changed.

Follow AGENTS.md. In PostgreSQL 19, create a question document to explain all these subjects related to autovacuum: Autovacuum can now use parallel workers, which can be configured with the new `autovacuum_max_parallel_workers` setting, and a new autovacuum scoring system helps prioritize tables to vacuum. PostgreSQL 19 further enhances VACUUM with a new strategy that can automatically reduce future vacuuming work by marking pages as all-visible in the visibility map while they're being queried.

## Answer

### Short Answer

All three statements are true for the pinned PostgreSQL 19 line (the `REL_19_STABLE` branch, commit `99e47536bbf1a165f5dc8d504f928821ebc8df6a`). They are three independent features:

1. **Parallel autovacuum.** An autovacuum worker can now hand its **index** vacuuming and index cleanup phases to parallel workers, exactly like manual `VACUUM (PARALLEL)`. The new GUC `autovacuum_max_parallel_workers` caps how many parallel workers a *single* autovacuum worker may use; it defaults to `0`, which keeps autovacuum serial. A per-table reloption `autovacuum_parallel_workers` requests a specific degree [autovacuum.c#parallel-workers](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2940-L2959) [vacuumparallel.c#compute-workers](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L741-L802) [config.sgml#autovacuum_max_parallel_workers](../../../../raw/postgres-19/doc/src/sgml/config.sgml#L9718-L9739).

2. **Table scoring (prioritization).** When an autovacuum worker has collected the tables that need work in a database, it now computes a numeric **score** per table and processes them in descending score order, so the most urgent tables go first. Five `*_score_weight` GUCs tune the components, and setting all five to `0.0` disables sorting entirely [autovacuum.c#AutoVacuumScores](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L330-L343) [autovacuum.c#sort-by-score](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2310-L2321) [autovacuum.c#score-comparator](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1906-L1917).

3. **Setting pages all-visible during read-only scans.** Ordinary read-only query table scans (seq scan, index scan, index-only scan, sample scan, TID-range scan, bitmap heap scan) can now set all-visible bits in the visibility map (VM) as a side effect of on-access pruning. They do not set all-frozen bits, because the read path passes `HEAP_PAGE_PRUNE_SET_VM` without `HEAP_PAGE_PRUNE_FREEZE`; `VACUUM` is the path that passes both today. Previously only `VACUUM` and `COPY ... FREEZE` set visibility-map bits. Pages already marked all-visible/all-frozen are skipped by future `VACUUM`, which is how this "reduces future vacuuming work." When on-access pruning newly marks a page all-visible, it also records the page's actual free space in the free space map (FSM); this closes a post-beta1 gap that could leave the FSM stale after `VACUUM` began skipping that page [pruneheap.c#heap_page_prune_opt](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L243-L410) [pruneheap.c#set_all_frozen](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L513-L538) [vacuumlazy.c#vacuum-uses-flag](../../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L2031-L2037) [execUtils.c#ScanRelIsReadOnly](../../../../raw/postgres-19/src/backend/executor/execUtils.c#L739-L758) [vacuumlazy.c#vm-skip](../../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L41-L67).

The three features touch different layers (the autovacuum daemon, the autovacuum scheduling logic, and the heap access method's read path) and can be reasoned about separately. Each is detailed below, followed by the per-feature [Source Commit History](#source-commit-history).

---

### 1. Parallel autovacuum workers

#### What it does

Parallel vacuum (parallelizing the **index** bulk-delete and index cleanup phases across worker processes) has existed for manual `VACUUM` since v13. PostgreSQL 19 lets the autovacuum daemon use the same machinery. The file that implements it now says so in its header: it is "Support routines for parallel vacuum and autovacuum execution" [vacuumparallel.c#header](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L1-L34).

Only the index phases are parallelized; the heap scan is not. The new GUC documentation makes this explicit: the limit "applies specifically to the index vacuuming and index cleanup phases," and it is "the per-autovacuum worker equivalent of the `PARALLEL` option of the `VACUUM` command" [config.sgml#autovacuum_max_parallel_workers](../../../../raw/postgres-19/doc/src/sgml/config.sgml#L9718-L9739).

#### How a table gets parallel workers (normal path)

`VacuumParams` carries an `nworkers` field with these semantics: `0` = choose automatically from the number of indexes, `-1` = parallelism disabled, `> 0` = request that many [vacuum.h#nworkers](../../../../raw/postgres-19/src/include/commands/vacuum.h#L245-L250).

For autovacuum, `table_recheck_autovac()` fills `nworkers` from the per-table reloption `autovacuum_parallel_workers`:

- reloption `0` → `nworkers = -1` (parallel vacuum disabled for this table);
- reloption `> 0` → `nworkers =` that value;
- reloption `-1` (the default) → falls through with `nworkers = 0` (automatic)

[autovacuum.c#nworkers-from-reloption](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2940-L2959).

The reloption itself is defined as a heap reloption with default `-1`, minimum `-1`, maximum `1024`, taking `ShareUpdateExclusiveLock` to set (so `ALTER TABLE ... SET (autovacuum_parallel_workers = N)` does not block reads/writes and takes effect on the next autovacuum of the table) [reloptions.c#reloption-def](../../../../raw/postgres-19/src/backend/access/common/reloptions.c#L239-L247) [reloptions.c#reloption-parse](../../../../raw/postgres-19/src/backend/access/common/reloptions.c#L1981-L1982).

#### The per-autovacuum-worker cap

`parallel_vacuum_compute_workers()` is where the degree is finally chosen. The key v19 change is the source of the cap: an autovacuum worker uses `autovacuum_max_parallel_workers`, while a regular backend running manual `VACUUM` uses `max_parallel_maintenance_workers` [vacuumparallel.c#compute-workers](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L741-L802):

```c
max_workers = AmAutoVacuumWorkerProcess() ?
    autovacuum_max_parallel_workers :
    max_parallel_maintenance_workers;

/* ... */
if (!IsUnderPostmaster || max_workers == 0)
    return 0;
```

So a `max_workers` of `0` short-circuits to no parallelism. Because `autovacuum_max_parallel_workers` defaults to `0`, **autovacuum is serial out of the box**; you opt in by raising the GUC [globals.c#default-0](../../../../raw/postgres-19/src/backend/utils/init/globals.c#L148). The function also requires at least two indexes large enough (`> min_parallel_index_scan_size`) and that support parallel vacuum, then caps the result by `max_workers` [vacuumparallel.c#cap](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L765-L801). The total is further bounded system-wide by `max_parallel_workers` and the available background-worker slots, as the docs note [config.sgml#further-limited](../../../../raw/postgres-19/doc/src/sgml/config.sgml#L9731-L9732).

#### Cost-delay propagation to parallel workers (new infrastructure)

Manual `VACUUM` workers inherit static cost-delay settings, but an autovacuum leader's cost-delay parameters can change mid-table (config reload, or the autovacuum balancing algorithm). So v19 added a shared-memory struct, `PVSharedCostParams`, with a generation counter. The leader updates the shared parameters and bumps the generation; each parallel worker polls the counter and refreshes its local `VacuumCost*` state when it changes [vacuumparallel.c#cost-params-header](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L21-L26) [vacuumparallel.c#propagate](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L644-L726). The refresh happens at the vacuum delay point, before the cost check, so a newly enabled delay takes effect promptly [vacuum.c#delay-point-refresh](../../../../raw/postgres-19/src/backend/commands/vacuum.c#L2448-L2456).

#### When no parallelism happens (edge cases)

- `autovacuum_max_parallel_workers = 0` (default) → serial [vacuumparallel.c#compute-workers](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L759-L760).
- Per-table `autovacuum_parallel_workers = 0` → `nworkers = -1` → parallel disabled for that table [autovacuum.c#nworkers-from-reloption](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2944-L2950).
- Fewer than two suitable, large-enough indexes → degree resolves to `0` [vacuumparallel.c#index-eligibility](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L762-L792).
- Standalone backend (`!IsUnderPostmaster`) → no parallelism [vacuumparallel.c#standalone](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L755-L760).

#### GUC / reloption scope

| Setting | Kind | Context → apply scope | Default | Range |
|---|---|---|---|---|
| `autovacuum_max_parallel_workers` | GUC `int` | `PGC_SIGHUP` → **reload** | `0` | `0` … `MAX_PARALLEL_WORKER_LIMIT` |
| `autovacuum_parallel_workers` | table reloption `int` | `ShareUpdateExclusiveLock` → set via `ALTER TABLE`; effective next autovacuum | `-1` (auto) | `-1` … `1024` |

`autovacuum_max_parallel_workers` is `PGC_SIGHUP`, so a config reload (`SELECT pg_reload_conf()` or `SIGHUP`) is enough — no restart [guc_parameters.dat#max_parallel_workers](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L172-L178) [miscadmin.h#extern](../../../../raw/postgres-19/src/include/miscadmin.h#L183).

#### Tests

The core commit shipped a dedicated TAP test module, `src/test/modules/test_autovacuum/`, whose single test `t/001_parallel_autovacuum.pl` drives autovacuum with `autovacuum_max_parallel_workers` set and checks that parallel workers run [001_parallel_autovacuum.pl](../../../../raw/postgres-19/src/test/modules/test_autovacuum/t/001_parallel_autovacuum.pl#L1-L40).

---

### 2. Autovacuum table scoring (prioritization)

#### What it does

Before v19, an autovacuum worker processed the tables that needed work in the order it found them while scanning `pg_class`. v19 adds "rudimentary table prioritization": the worker computes a **score** for every candidate table and sorts the work list so higher-scoring (more urgent) tables are vacuumed/analyzed first. This changes only the **order** of processing within one worker's pass; it does not change *which* tables are eligible, nor the launcher's choice of which database to visit next.

#### How the score is computed

`relation_needs_vacanalyze()` is the function that decides whether a table needs vacuum/analyze. v19 extended it to also fill an `AutoVacuumScores` out-parameter — a `max` plus five component scores [autovacuum.c#AutoVacuumScores](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L330-L343) [autovacuum.c#relation_needs_vacanalyze](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3067-L3081):

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

- **Freeze components.** `xid`/`mxid` = age divided by the respective `*_freeze_max_age`, then multiplied by the weight. If a table is past its (weight-adjusted) failsafe age, the score is raised aggressively via `pow()` so near-wraparound tables dominate [autovacuum.c#freeze-score](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3198-L3243). The `mxid` divisor is the per-table-clamped `effective_multixact_freeze_max_age` (from `MultiXactMemberFreezeThreshold()`), which is driven toward `0` as multixact member space fills up; v19 therefore guards it as `Max(1, multixact_freeze_max_age)`, so when the threshold reaches `0` the `mxid` component degrades to the raw multixact age instead of dividing by zero (a post-beta1 fix, `1f2297b5487`) [autovacuum.c#mxid-divisor-guard](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3198-L3208) [autovacuum.c#mxid-max-age-clamp](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3172-L3174).
- **Dead-tuple / insert / analyze components.** `vac` = dead tuples ÷ vacuum threshold, `vac_ins` = inserts ÷ insert threshold, `anl` = modified tuples ÷ analyze threshold, each multiplied by its weight [autovacuum.c#vac-ins-anl-score](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3292-L3320).

`scores->max` is the running maximum of the components, and that single number is the table's score. Note that the eligibility decision (`*dovacuum` / `*doanalyze`) is still threshold-based and independent of the weights — weights only affect ordering, except that anti-wraparound (`force_vacuum`) still forces a vacuum regardless [autovacuum.c#force-vacuum](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3244-L3245).

#### How tables are sorted

`do_autovacuum()` builds a list of `TableToProcess { oid, score }`, storing `scores.max` for each table that needs work [autovacuum.c#TableToProcess](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L345-L352) [autovacuum.c#collect](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2079-L2094). After collecting both ordinary tables and TOAST tables, it sorts the list in **descending** score order [autovacuum.c#comparator](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1906-L1917):

```c
return (t2->score < t1->score) ? -1 : (t2->score > t1->score) ? 1 : 0;
```

#### The five score-weight GUCs (and the escape hatch)

There is one weight per component, all `real`, `PGC_SIGHUP` (reload), default `1.0`, range `0.0`–`10.0`:

| GUC | Component | Definition |
|---|---|---|
| `autovacuum_freeze_score_weight` | `xid` | [guc_parameters.dat#L164-L170](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L164-L170) |
| `autovacuum_multixact_freeze_score_weight` | `mxid` | [guc_parameters.dat#L197-L203](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L197-L203) |
| `autovacuum_vacuum_score_weight` | `vac` | [guc_parameters.dat#L275-L281](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L275-L281) |
| `autovacuum_vacuum_insert_score_weight` | `vac_ins` | [guc_parameters.dat#L241-L247](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L241-L247) |
| `autovacuum_analyze_score_weight` | `anl` | [guc_parameters.dat#L138-L144](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L138-L144) |

Setting **all five** weights to `0.0` is a deliberate escape hatch: `do_autovacuum()` skips `list_sort()` entirely in that case, restoring the old unsorted behavior [autovacuum.c#escape-hatch](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2310-L2321).

> Documentation note: the v19 release notes list all five score-weight parameters, including `autovacuum_vacuum_insert_score_weight` [release-19.sgml#score-weight-list](../../../../raw/postgres-19/doc/src/sgml/release-19.sgml#L1331-L1334) [guc_parameters.dat#vacuum-insert-score-weight](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L241-L247), matching the source definition [autovacuum.c#weight-globals](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L137-L141). At the previous pin the release-note link text dropped the `autovacuum_` prefix from this one parameter; that typo was fixed on the v19 line by `1a7fa06dbcd` (in the `cdae794a..01c544e1` repin range).

#### Monitoring: `pg_stat_autovacuum_scores`

A companion commit added a system view, `pg_stat_autovacuum_scores`, exposing each table's overall and component scores plus the do-vacuum/do-analyze/for-wraparound decisions, sourced from `pg_stat_get_autovacuum_scores()` [system_views.sql#pg_stat_autovacuum_scores](../../../../raw/postgres-19/src/backend/catalog/system_views.sql#L798-L814):

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

#### Not the same as the launcher's database scheduling

Do not confuse the new per-table score with the autovacuum *launcher's* long-standing per-database ordering. The launcher keeps an `avl_dbase` list with an `adl_score` field used only to round-robin databases when choosing which one to send the next worker to [autovacuum.c#avl_dbase](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L181-L187) [autovacuum.c#db_comparator](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1118-L1123). That `adl_score` is unrelated to v19's table `AutoVacuumScores`.

#### Tests

The scoring *algorithm* itself ships without a dedicated regression or TAP test — the core commit touched only C, GUC data, docs, and headers (see [Source Commit History](#source-commit-history)). The `pg_stat_autovacuum_scores` *view definition* is covered by the standard `rules` regression test [rules.out#view](../../../../raw/postgres-19/src/test/regress/expected/rules.out#L1863).

---

### 3. Setting pages all-visible during read-only query scans

#### What it does

PostgreSQL maintains a per-relation visibility map (VM) with two bits per heap page: all-visible and all-frozen. These bits let index-only scans skip heap fetches and let `VACUUM` skip pages. Historically only `VACUUM` and `COPY ... FREEZE` set visibility-map bits. v19 lets ordinary **read-only query scans** set the all-visible bit as a side effect of *on-access pruning*, the opportunistic page cleanup that already happens while reading. Read-only scans do **not** set the all-frozen bit, because `heap_page_prune_opt()` passes `HEAP_PAGE_PRUNE_SET_VM` but not `HEAP_PAGE_PRUNE_FREEZE`; `VACUUM` passes both options [pruneheap.c#heap_page_prune_opt](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L359-L364) [pruneheap.c#set_all_frozen](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L513-L538) [vacuumlazy.c#vacuum-uses-flag](../../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L2031-L2037). The release note phrases the feature as "Allow query table scans to mark pages as all-visible in the visibility map" [release-19.sgml#all-visible-scans](../../../../raw/postgres-19/doc/src/sgml/release-19.sgml#L685-L696).

#### Normal path

1. **Plan time / scan setup — is the scan read-only?** `ScanRelIsReadOnly()` returns true when the scanned relation is neither a result (modified) relation nor a row-mark relation [executor.h#decl](../../../../raw/postgres-19/src/include/executor/executor.h#L698) [execUtils.c#ScanRelIsReadOnly](../../../../raw/postgres-19/src/backend/executor/execUtils.c#L739-L758). All the heap-scanning executor nodes consult it: sequential [nodeSeqscan.c#read-only](../../../../raw/postgres-19/src/backend/executor/nodeSeqscan.c#L67-L85), index and index-only [nodeIndexscan.c#read-only](../../../../raw/postgres-19/src/backend/executor/nodeIndexscan.c#L117) [nodeIndexonlyscan.c#read-only](../../../../raw/postgres-19/src/backend/executor/nodeIndexonlyscan.c#L100), sample [nodeSamplescan.c#read-only](../../../../raw/postgres-19/src/backend/executor/nodeSamplescan.c#L302), TID-range [nodeTidrangescan.c#read-only](../../../../raw/postgres-19/src/backend/executor/nodeTidrangescan.c#L249), and bitmap heap [nodeBitmapHeapscan.c#read-only](../../../../raw/postgres-19/src/backend/executor/nodeBitmapHeapscan.c#L149).

2. **Flag travels on the scan descriptor.** A read-only seq scan begins with `SO_HINT_REL_READ_ONLY` set [nodeSeqscan.c#flag](../../../../raw/postgres-19/src/backend/executor/nodeSeqscan.c#L67-L85).

3. **Heap read path forwards it to pruning.** When preparing a page for a page-mode scan, `heap_prepare_pagescan()` calls on-access pruning, passing the read-only hint through [heapam.c#heap_prepare_pagescan](../../../../raw/postgres-19/src/backend/access/heap/heapam.c#L612-L641):

   ```c
   heap_page_prune_opt(scan->rs_base.rs_rd, buffer, &scan->rs_vmbuffer,
                       sscan->rs_flags & SO_HINT_REL_READ_ONLY);
   ```

   The index-fetch and other heap read entry points pass the flag the same way [heapam_handler.c#prune-opt](../../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c#L2568) [heapam_indexscan.c#prune-opt](../../../../raw/postgres-19/src/backend/access/heap/heapam_indexscan.c#L260).

4. **On-access pruning requests VM-setting.** `heap_page_prune_opt()` gained a `rel_read_only` parameter; when true it adds `HEAP_PAGE_PRUNE_SET_VM` to the prune options [pruneheap.c#heap_page_prune_opt](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L243-L364) [heapam.h#flag-def](../../../../raw/postgres-19/src/include/access/heapam.h#L45):

   ```c
   params.options = HEAP_PAGE_PRUNE_ALLOW_FAST_PATH;
   if (rel_read_only)
       params.options |= HEAP_PAGE_PRUNE_SET_VM;
   ```

#### How the VM bits get set

Inside pruning, `HEAP_PAGE_PRUNE_SET_VM` becomes `PruneState.attempt_set_vm`, which seeds `set_all_visible`; `set_all_frozen` additionally requires that freezing is being attempted (only `VACUUM` freezes today, so a read scan sets all-visible but not all-frozen) [pruneheap.c#prune-state-init](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L425-L538). After scanning the page, `heap_page_will_set_vm()` decides whether to actually set the bits and, if so, what `new_vmbits` to write [pruneheap.c#will_set_vm](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L959-L1006). The same `HEAP_PAGE_PRUNE_SET_VM` flag is what `VACUUM` passes, so both paths share one VM-setting implementation [vacuumlazy.c#vacuum-uses-flag](../../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L2036).

#### The read-query safety valve

The feature is careful not to make read queries pay VACUUM's write costs. For an on-access call that is not also pruning or freezing, `heap_page_will_set_vm()` declines to set the VM if doing so would newly dirty the heap page, or — if the page is already dirty — would force a full-page image (FPI) into WAL [pruneheap.c#safety-valve](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L981-L992):

```c
if (reason == PRUNE_ON_ACCESS && !do_prune && !do_freeze &&
    (!BufferIsDirty(prstate->buffer) || XLogCheckBufferNeedsBackup(prstate->buffer)))
{
    prstate->set_all_visible = prstate->set_all_frozen = false;
    return false;
}
```

So a read query opportunistically sets the VM mainly when it was already going to write the page (it pruned/froze) or when setting the bit is cheap.

#### Why this reduces future vacuuming work

`VACUUM` uses the VM to skip pages: a normal vacuum may skip ranges of pages "marked all-visible (and even all-frozen) in the visibility map," scanning them only to break up small skippable ranges (`SKIP_PAGES_THRESHOLD`) or to freeze [vacuumlazy.c#vm-skip](../../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L41-L67). Pages a read scan already marked all-visible are therefore candidates for being skipped by the next ordinary `VACUUM`, which would otherwise have to re-scan them, recheck visibility, and set the bit itself. Two precisions: a read scan sets all-visible but **not** all-frozen (only `VACUUM` freezes), so anti-wraparound vacuums must still visit these pages to freeze them — the saving here is on ordinary vacuums; and marking pages during reads also immediately benefits index-only scans, which can skip heap visibility checks for all-visible pages.

#### Keeping the free space map current

The original on-access VM feature could make a heap page newly all-visible without refreshing its free space map (FSM) entry. A later `VACUUM` could then skip that page because of the VM bit and never repair its leaf-level FSM value. Commit `e9eaeb04248` fixes this in the same caller: after `heap_page_prune_and_freeze()` reports `PruneFreezeResult.newly_all_visible`, `heap_page_prune_opt()` captures `PageGetHeapFreeSpace()` while it still holds the heap-buffer cleanup lock, releases that lock, and calls `RecordPageWithFreeSpace()` [heapam.h#PruneFreezeResult](../../../../raw/postgres-19/src/include/access/heapam.h#L317-L323) [pruneheap.c#record-newly-all-visible-FSM](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L323-L409).

The update is deliberately narrow. On-access pruning still does not publish free space after every prune, because reserving that space for same-page HOT updates is useful; it records free space only when this call newly made the page all-visible. `RecordPageWithFreeSpace()` writes the page's FSM category, and a later `FreeSpaceMapVacuum()` propagates an increased value to upper FSM levels; skipped VACUUM ranges still receive that FSM-vacuum pass [pruneheap.c#FSM-policy](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L384-L408) [freespace.c#RecordPageWithFreeSpace](../../../../raw/postgres-19/src/backend/storage/freespace/freespace.c#L186-L204).

#### Clearing VM bits: WAL and incremental-backup correctness

The current pin also contains adjacent fixes for the opposite transition: heap modifications clearing VM bits. They do not change the read-only `HEAP_PAGE_PRUNE_SET_VM` path above [pruneheap.c#on-access-SET_VM](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L343-L364). `visibilitymap_clear()` requires its caller to pass the already-pinned VM buffer while holding its exclusive content lock. If the buffer is invalid or belongs to the wrong VM block, the function now raises `ERROR` in every build instead of relying on an assertion-only check; the exclusive-lock check remains an assertion [visibilitymap.c#visibilitymap_clear](../../../../raw/postgres-19/src/backend/access/heap/visibilitymap.c#L145-L184). For example, `heap_insert()` locks that buffer before its critical section, clears the VM bits, registers the VM buffer in the heap WAL record when it changed, assigns the same record LSN to the VM page, and releases the VM lock only after WAL insertion [heapam.c#heap_insert-VM-WAL](../../../../raw/postgres-19/src/backend/access/heap/heapam.c#L2058-L2197). An UPDATE can clear the old and new heap pages' VM bits; named block-reference IDs distinguish the two VM pages, and the update WAL record registers each modified VM buffer [heapam_xlog.h#update-VM-block-IDs](../../../../raw/postgres-19/src/include/access/heapam_xlog.h#L213-L245) [heapam.c#update-VM-locks](../../../../raw/postgres-19/src/backend/access/heap/heapam.c#L4103-L4164) [heapam.c#update-VM-registration](../../../../raw/postgres-19/src/backend/access/heap/heapam.c#L9226-L9244).

Redo now reads the registered VM block, applies its full-page image when present or clears the requested bits itself, and sets the VM page LSN [heapam_xlog.c#heap_xlog_vm_clear](../../../../raw/postgres-19/src/backend/access/heap/heapam_xlog.c#L32-L67). This also makes VM clears visible to WAL summaries and therefore to incremental backups. The new TAP test starts with `VACUUM (FREEZE)`, exercises INSERT, DELETE, same-page and cross-page UPDATE, `SELECT FOR UPDATE`, and COPY, then combines a full and incremental backup and checks both VM counts and `pg_check_visible()` on the restored server [012_vm_consistency.pl#operations](../../../../raw/postgres-19/src/bin/pg_combinebackup/t/012_vm_consistency.pl#L1-L95) [012_vm_consistency.pl#verification](../../../../raw/postgres-19/src/bin/pg_combinebackup/t/012_vm_consistency.pl#L116-L168) [012_vm_consistency.pl#backup-restore](../../../../raw/postgres-19/src/bin/pg_combinebackup/t/012_vm_consistency.pl#L171-L253). The WAL-record change also advances `XLOG_PAGE_MAGIC` to `0xD121` [xlog_internal.h#XLOG_PAGE_MAGIC](../../../../raw/postgres-19/src/include/access/xlog_internal.h#L32-L35).

The stronger protocol can hold the VM content lock through the heap critical section and WAL insertion, and a registered VM buffer may require a VM full-page image. That is a correctness cost for writers that actually clear VM bits; it does not relax the separate no-extra-dirtying/FPI safety valve used when read-only scans try to set bits [heapam.c#heap_insert-VM-WAL](../../../../raw/postgres-19/src/backend/access/heap/heapam.c#L2058-L2197) [heapam_xlog.c#FPI-or-redo](../../../../raw/postgres-19/src/backend/access/heap/heapam_xlog.c#L49-L64) [pruneheap.c#safety-valve](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L981-L992).

#### Edge cases

- **Recovery.** On-access pruning (and thus VM-setting) is skipped during recovery, since standbys cannot write WAL for it [pruneheap.c#recovery](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L280-L286).
- **`INSERT ... SELECT` from the same table.** The scanned relation is *not* recorded as a result relation, so it is reported read-only even though the statement modifies it. The code comments note this is harmless in practice: pages it inserts into rarely meet the prune heuristic, and those that do usually have in-progress inserts that keep them from being all-visible [pruneheap.c#insert-select-note](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L260-L269) [execUtils.c#caveats](../../../../raw/postgres-19/src/backend/executor/execUtils.c#L739-L758).
- **Row-marked relations.** If any relation in the query has a modifying row mark, the others get a `ROW_MARK_REFERENCE` and are conservatively reported *not* read-only [execUtils.c#rowmark-note](../../../../raw/postgres-19/src/backend/executor/execUtils.c#L745-L758).

#### Tests

The headline commit `b46e1e54d` changed only heap access-method code and headers; it added no dedicated regression or isolation test of its own (see [Source Commit History](#source-commit-history)). The follow-up FSM fix `e9eaeb04248` likewise changed only `pruneheap.c` and added no direct test. The new `012_vm_consistency.pl` TAP test directly covers heap operations that **clear** VM bits and the incremental-backup restore path, but it seeds those bits with `VACUUM (FREEZE)` rather than an ordinary read scan [012_vm_consistency.pl#setup-and-modify](../../../../raw/postgres-19/src/bin/pg_combinebackup/t/012_vm_consistency.pl#L116-L205). The make build installs `contrib/pg_visibility` for this test, and the Meson test list registers `t/012_vm_consistency.pl`; no generated catalog or header artifact is involved [pg_combinebackup/Makefile#EXTRA_INSTALL](../../../../raw/postgres-19/src/bin/pg_combinebackup/Makefile#L12-L19) [pg_combinebackup/meson.build#tests](../../../../raw/postgres-19/src/bin/pg_combinebackup/meson.build#L32-L44). The on-access set-VM/FSM path therefore still has no dedicated test.

---

### Source Commit History

All hashes are ancestors of the pinned `REL_19_STABLE` commit `99e47536bbf1a165f5dc8d504f928821ebc8df6a`. Two feature-code fixes landed after `REL_19_BETA1` (tagged 2026-06-01): the autovacuum MXID-score division-by-zero guard `1f2297b5487` and the on-access VM/FSM correction `e9eaeb04248`, each listed under its feature below. No parallel-autovacuum feature code changed after beta1. In the `cdae794a..01c544e1` repin range (the 2026-07-09 move from the former `master` pin onto the new `REL_19_STABLE` branch), no scoring/parallel-vacuum/VM feature code changed; the one touch to this page's area was the release-note prefix typo fix `1a7fa06dbcd` in `release-19.sgml`. In the 12-commit `01c544e1..8055e337` range, only `e9eaeb04248` changed these three feature scopes. The 31-commit `8055e337..3aa54433` range changes neither parallel autovacuum nor scoring and leaves the on-access VM-setting algorithm intact; it adds the adjacent three-commit VM-clear WAL sequence `56bf5fa5d67` / `b01c31eef9c` / `9171f77db23`. The 52-commit `3aa54433..99e47536` range also leaves parallel autovacuum, scoring, and the on-access VM-setting algorithm unchanged; `3180ce3d7a8` hardens the adjacent VM-clear caller contract, while `99e47536bbf` changes logical-decoding activation outside all three feature scopes. Dates are author dates.

#### Parallel autovacuum workers

| Date | Commit | Subject |
|---|---|---|
| 2026-03-19 | `adcdbe93860` | Add parallel vacuum worker usage to VACUUM (VERBOSE) and autovacuum logs |
| 2026-04-06 | `1ff3180ca01` | **Allow autovacuum to use parallel vacuum workers** (core) |
| 2026-04-09 | `8030b839d3d` | Remove an unstable wait from parallel autovacuum regression test |
| 2026-04-10 | `2a3d2f9f68d` | doc: Improve consistency of parallel vacuum description |
| 2026-04-10 | `c22d115f1d1` | Fix unstable log verification in test_autovacuum |

The core commit `1ff3180ca01` (committed by Masahiko Sawada; the v19 release notes credit Daniil Davydov) added the `autovacuum_max_parallel_workers` GUC, the `autovacuum_parallel_workers` reloption, the autovacuum-vs-maintenance worker cap in `parallel_vacuum_compute_workers()`, the `PVSharedCostParams` cost-delay propagation, and the `test_autovacuum` TAP module [vacuumparallel.c#compute-workers](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L741-L802). `adcdbe93860` is supporting infrastructure that reports parallel-worker usage in `VACUUM (VERBOSE)` and autovacuum logs.

#### Autovacuum table scoring

| Date | Commit | Subject |
|---|---|---|
| 2026-03-27 | `d7965d65fc5` | **Add rudimentary table prioritization to autovacuum** (core) |
| 2026-04-03 | `8261ee24fe3` | Refactor relation_needs_vacanalyze() |
| 2026-04-03 | `53b8ca6881a` | Teach relation_needs_vacanalyze() to always compute scores |
| 2026-04-06 | `87f61f0c828` | Add pg_stat_autovacuum_scores system view |
| 2026-04-09 | `71ff232a5bc` | Fix double-free in pg_stat_autovacuum_scores |
| 2026-06-18 | `1f2297b5487` | Avoid division-by-zero when calculating autovacuum MXID score (post-beta1) |

All by Nathan Bossart. `d7965d65fc5` introduced `AutoVacuumScores`, `TableToProcess`, the descending sort with its all-weights-zero escape hatch, and the five `*_score_weight` GUCs. `87f61f0c828` added the `pg_stat_autovacuum_scores` view and `pg_stat_get_autovacuum_scores()`; `71ff232a5bc` fixed a double-free in it. `1f2297b5487` is a post-beta1 fix that guards the `mxid` score divisor against a zero `effective_multixact_freeze_max_age` (see [How the score is computed](#how-the-score-is-computed)).

#### Setting pages all-visible during reads

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
| 2026-07-10 | `e9eaeb04248` | **Update FSM after updating VM on-access** (post-beta1 fix) |
| 2026-07-15 | `56bf5fa5d67` | Name heap WAL block-reference IDs in preparation for registering VM pages (adjacent VM-clear infrastructure) |
| 2026-07-15 | `b01c31eef9c` | **Fix VM-clear WAL logging by registering modified VM pages** (adjacent post-19beta2-stamp fix) |
| 2026-07-15 | `9171f77db23` | Test VM clears through WAL summaries and incremental-backup restore (adjacent test) |
| 2026-07-17 | `3180ce3d7a8` | Restore an unconditional error for the wrong buffer passed to `visibilitymap_clear()` (adjacent VM-clear hardening) |

The headline commit `b46e1e54d07` added the `rel_read_only` parameter to `heap_page_prune_opt()` and the `HEAP_PAGE_PRUNE_SET_VM` option [pruneheap.c#heap_page_prune_opt](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L243-L364). `50eb5faea29` is the executor plumbing (`ScanRelIsReadOnly()` / `SO_HINT_REL_READ_ONLY`); `99bf1f8aa6c` saved the `rs_vmbuffer` so scans can pin the VM page; `01b7e4a46d0` added the all-visible/all-frozen fast path. The post-beta1 `e9eaeb04248` follow-up records free space when that path newly sets the page all-visible, preventing the VM-driven VACUUM skip from leaving a stale FSM entry [pruneheap.c#record-newly-all-visible-FSM](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L384-L408). The three July 15 entries and `3180ce3d7a8` are listed separately as adjacent current-source correctness work: they govern clearing bits after writes, preserving those clears through WAL/incremental backup, and enforcing the clear caller's buffer contract, not the read path that sets bits [heapam_xlog.c#heap_xlog_vm_clear](../../../../raw/postgres-19/src/backend/access/heap/heapam_xlog.c#L32-L67) [visibilitymap.c#wrong-buffer-error](../../../../raw/postgres-19/src/backend/access/heap/visibilitymap.c#L145-L173) [012_vm_consistency.pl#purpose](../../../../raw/postgres-19/src/bin/pg_combinebackup/t/012_vm_consistency.pl#L1-L6). This list is the cluster identified for the v19 cycle; it builds on the pre-v19 WAL-record unification (`f83d709760d`, 2024, Heikki Linnakangas) noted under [Open Questions](#open-questions).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v19/index.md`, and the last ~20 `wiki/log.md` entries for navigation and the v19 pin. The 2026-06-10 log note records an earlier orphan draft of this question that was removed at the user's request; this page is the filed replacement.
- Autovacuum daemon: `src/backend/postmaster/autovacuum.c` (score weights, `AutoVacuumScores`, `TableToProcess`, `relation_needs_vacanalyze`, `do_autovacuum`, comparator, `table_recheck_autovac` worker selection, launcher `avl_dbase`/`adl_score`).
- Parallel vacuum: `src/backend/commands/vacuumparallel.c` (`parallel_vacuum_compute_workers`, `PVSharedCostParams`), `src/backend/commands/vacuum.c` (delay-point refresh), `src/include/commands/vacuum.h` (`nworkers`).
- GUC/reloption definitions: `src/backend/utils/misc/guc_parameters.dat`, `src/backend/access/common/reloptions.c`, `src/backend/utils/init/globals.c`, `src/include/miscadmin.h`.
- Heap read/prune path: `src/backend/access/heap/pruneheap.c`, `heapam.c`, `heapam_handler.c`, `heapam_indexscan.c`, `vacuumlazy.c`, `src/include/access/heapam.h`, and `src/backend/storage/freespace/freespace.c`.
- VM-clear WAL/recovery boundary: `heapam.c`, `heapam_xlog.c`, `visibilitymap.c`, `heapam_xlog.h`, `xlog_internal.h`, the WAL-summary test, and `src/bin/pg_combinebackup/t/012_vm_consistency.pl`.
- Executor scan nodes and `ScanRelIsReadOnly`: `src/backend/executor/execUtils.c`, `nodeSeqscan.c`, `nodeIndexscan.c`, `nodeIndexonlyscan.c`, `nodeSamplescan.c`, `nodeTidrangescan.c`, `nodeBitmapHeapscan.c`, `src/include/executor/executor.h`.
- Catalog/monitoring: `src/backend/catalog/system_views.sql`, `src/test/regress/expected/rules.out`.
- Same-checkout docs (`doc/src/sgml/config.sgml`, `release-19.sgml`) used to locate features and their commits; behavioral claims are cited to source.
- `git log` on the pinned checkout for the three commit clusters above, plus changed-file and commit-body reviews of the `cdae794a..01c544e1`, `01c544e1..8055e337`, and `8055e337..3aa54433` repin ranges.
- Repin range `3aa54433..99e47536`: reviewed all 52 commits and changed paths. `3180ce3d7a8` restores the unconditional wrong-buffer error in `visibilitymap_clear()` and is included as adjacent VM-clear hardening. `1d4c81ad626` changes index-only tuple deformation, not the `ScanRelIsReadOnly()` setup used by this page; its include insertion shifts that citation by one line. `99e47536bbf` changes logical-decoding activation, outside all three feature scopes. Release-note editing shifts the two cited v19 entries without changing their feature descriptions. No commit changes parallel-autovacuum or scoring code.

## Evidence Map

| Claim | Evidence |
|---|---|
| Autovacuum can use parallel index-vacuum workers | [vacuumparallel.c#L1-L34](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L1-L34) [vacuumparallel.c#L741-L802](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L741-L802) |
| Cap source differs: autovacuum vs manual VACUUM | [vacuumparallel.c#L751-L753](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L751-L753) |
| `autovacuum_max_parallel_workers` GUC, PGC_SIGHUP, default 0 | [guc_parameters.dat#L172-L178](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L172-L178) [globals.c#L148](../../../../raw/postgres-19/src/backend/utils/init/globals.c#L148) [config.sgml#L9718-L9739](../../../../raw/postgres-19/doc/src/sgml/config.sgml#L9718-L9739) |
| Per-table `autovacuum_parallel_workers` reloption; -1/0/N mapping | [reloptions.c#L239-L247](../../../../raw/postgres-19/src/backend/access/common/reloptions.c#L239-L247) [autovacuum.c#L2940-L2959](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2940-L2959) [vacuum.h#L245-L250](../../../../raw/postgres-19/src/include/commands/vacuum.h#L245-L250) |
| Cost-delay propagation to parallel autovacuum workers | [vacuumparallel.c#L21-L26](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L21-L26) [vacuumparallel.c#L644-L726](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c#L644-L726) [vacuum.c#L2448-L2456](../../../../raw/postgres-19/src/backend/commands/vacuum.c#L2448-L2456) |
| Parallel autovacuum TAP test | [001_parallel_autovacuum.pl#L1-L40](../../../../raw/postgres-19/src/test/modules/test_autovacuum/t/001_parallel_autovacuum.pl#L1-L40) |
| Per-table score struct and components | [autovacuum.c#L330-L343](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L330-L343) [autovacuum.c#L3067-L3081](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3067-L3081) |
| Freeze/dead/insert/analyze score formulas; `mxid` divisor guarded by `Max(1, ...)` (post-beta1 `1f2297b5487`) | [autovacuum.c#L3198-L3243](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3198-L3243) [autovacuum.c#L3292-L3320](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3292-L3320) [autovacuum.c#L3172-L3174](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L3172-L3174) |
| Descending sort; all-weights-zero escape hatch | [autovacuum.c#L1906-L1917](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1906-L1917) [autovacuum.c#L2310-L2321](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L2310-L2321) |
| Five `*_score_weight` GUCs (PGC_SIGHUP, 1.0, 0.0–10.0) | [guc_parameters.dat#L138-L144](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L138-L144) [guc_parameters.dat#L164-L170](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L164-L170) [guc_parameters.dat#L197-L203](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L197-L203) [guc_parameters.dat#L241-L247](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L241-L247) [guc_parameters.dat#L275-L281](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat#L275-L281) |
| `pg_stat_autovacuum_scores` view | [system_views.sql#L798-L814](../../../../raw/postgres-19/src/backend/catalog/system_views.sql#L798-L814) [rules.out#L1863](../../../../raw/postgres-19/src/test/regress/expected/rules.out#L1863) |
| Launcher `adl_score` is separate database scheduling | [autovacuum.c#L181-L187](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L181-L187) [autovacuum.c#L1118-L1123](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c#L1118-L1123) |
| Read-only scans flagged via `ScanRelIsReadOnly` | [execUtils.c#L739-L758](../../../../raw/postgres-19/src/backend/executor/execUtils.c#L739-L758) [nodeSeqscan.c#L67-L85](../../../../raw/postgres-19/src/backend/executor/nodeSeqscan.c#L67-L85) |
| Flag forwarded into on-access pruning → SET_VM | [heapam.c#L612-L641](../../../../raw/postgres-19/src/backend/access/heap/heapam.c#L612-L641) [pruneheap.c#L243-L364](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L243-L364) [heapam.h#L45](../../../../raw/postgres-19/src/include/access/heapam.h#L45) |
| VM-bit decision + read-query safety valve | [pruneheap.c#L425-L538](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L425-L538) [pruneheap.c#L959-L1006](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L959-L1006) |
| A newly all-visible page records its free space so a later VM-based VACUUM skip does not leave stale FSM state (`e9eaeb04248`) | [pruneheap.c#L323-L409](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c#L323-L409) [heapam.h#L317-L323](../../../../raw/postgres-19/src/include/access/heapam.h#L317-L323) [freespace.c#L186-L204](../../../../raw/postgres-19/src/backend/storage/freespace/freespace.c#L186-L204) |
| Heap operations register modified VM pages in WAL; redo and incremental-backup tests preserve clears (`56bf5fa5d67` / `b01c31eef9c` / `9171f77db23`) | [heapam.c#L2058-L2197](../../../../raw/postgres-19/src/backend/access/heap/heapam.c#L2058-L2197) [heapam_xlog.c#L32-L67](../../../../raw/postgres-19/src/backend/access/heap/heapam_xlog.c#L32-L67) [012_vm_consistency.pl#L1-L6](../../../../raw/postgres-19/src/bin/pg_combinebackup/t/012_vm_consistency.pl#L1-L6) [012_vm_consistency.pl#L224-L253](../../../../raw/postgres-19/src/bin/pg_combinebackup/t/012_vm_consistency.pl#L224-L253) |
| `visibilitymap_clear()` errors on an invalid or wrong-block VM buffer (`3180ce3d7a8`) | [visibilitymap.c#L145-L173](../../../../raw/postgres-19/src/backend/access/heap/visibilitymap.c#L145-L173) |
| Future VACUUM skips all-visible/all-frozen pages | [vacuumlazy.c#L41-L67](../../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c#L41-L67) |

## Source References

- [autovacuum.c](../../../../raw/postgres-19/src/backend/postmaster/autovacuum.c) — autovacuum daemon: score-weight globals, `AutoVacuumScores`/`TableToProcess`, `relation_needs_vacanalyze`, `do_autovacuum` collection and descending sort, `table_recheck_autovac` parallel-worker selection, and the launcher's separate `avl_dbase`/`adl_score`.
- [vacuumparallel.c](../../../../raw/postgres-19/src/backend/commands/vacuumparallel.c) — parallel vacuum/autovacuum support: `parallel_vacuum_compute_workers` cap, `PVSharedCostParams` cost-delay propagation.
- [vacuum.c](../../../../raw/postgres-19/src/backend/commands/vacuum.c) / [vacuum.h](../../../../raw/postgres-19/src/include/commands/vacuum.h) — `VacuumParams.nworkers` semantics and the delay-point cost refresh.
- [reloptions.c](../../../../raw/postgres-19/src/backend/access/common/reloptions.c) — `autovacuum_parallel_workers` heap reloption definition and parse mapping.
- [guc_parameters.dat](../../../../raw/postgres-19/src/backend/utils/misc/guc_parameters.dat) / [globals.c](../../../../raw/postgres-19/src/backend/utils/init/globals.c) / [miscadmin.h](../../../../raw/postgres-19/src/include/miscadmin.h) — `autovacuum_max_parallel_workers` and the five `*_score_weight` GUC definitions and defaults.
- [pruneheap.c](../../../../raw/postgres-19/src/backend/access/heap/pruneheap.c) — `heap_page_prune_opt` read-only path, `PruneState` VM fields, `heap_page_will_set_vm` decision, on-access safety valve, and newly-all-visible FSM update.
- [freespace.c](../../../../raw/postgres-19/src/backend/storage/freespace/freespace.c) — `RecordPageWithFreeSpace()` and upper-level FSM propagation boundary.
- [heapam.c](../../../../raw/postgres-19/src/backend/access/heap/heapam.c) / [heapam_handler.c](../../../../raw/postgres-19/src/backend/access/heap/heapam_handler.c) / [heapam_indexscan.c](../../../../raw/postgres-19/src/backend/access/heap/heapam_indexscan.c) / [heapam.h](../../../../raw/postgres-19/src/include/access/heapam.h) — heap read paths forwarding `SO_HINT_REL_READ_ONLY`, the `HEAP_PAGE_PRUNE_SET_VM` flag, and writer-side VM clearing/WAL registration.
- [heapam_xlog.c](../../../../raw/postgres-19/src/backend/access/heap/heapam_xlog.c) / [visibilitymap.c](../../../../raw/postgres-19/src/backend/access/heap/visibilitymap.c) / [heapam_xlog.h](../../../../raw/postgres-19/src/include/access/heapam_xlog.h) / [xlog_internal.h](../../../../raw/postgres-19/src/include/access/xlog_internal.h) — VM-clear redo, caller-held VM locking, named WAL block references, and the WAL-format magic.
- [012_vm_consistency.pl](../../../../raw/postgres-19/src/bin/pg_combinebackup/t/012_vm_consistency.pl) / [pg_combinebackup/Makefile](../../../../raw/postgres-19/src/bin/pg_combinebackup/Makefile) / [pg_combinebackup/meson.build](../../../../raw/postgres-19/src/bin/pg_combinebackup/meson.build) — VM-clear coverage across heap operations, WAL summaries, combined incremental-backup restore, VM counts, `pg_check_visible()`, and the test's make/Meson wiring.
- [vacuumlazy.c](../../../../raw/postgres-19/src/backend/access/heap/vacuumlazy.c) — VACUUM's VM-skip behavior (the future-work reduction) and its own use of `HEAP_PAGE_PRUNE_SET_VM`.
- [execUtils.c](../../../../raw/postgres-19/src/backend/executor/execUtils.c) / [executor.h](../../../../raw/postgres-19/src/include/executor/executor.h) and the scan nodes [nodeSeqscan.c](../../../../raw/postgres-19/src/backend/executor/nodeSeqscan.c), [nodeIndexscan.c](../../../../raw/postgres-19/src/backend/executor/nodeIndexscan.c), [nodeIndexonlyscan.c](../../../../raw/postgres-19/src/backend/executor/nodeIndexonlyscan.c), [nodeSamplescan.c](../../../../raw/postgres-19/src/backend/executor/nodeSamplescan.c), [nodeTidrangescan.c](../../../../raw/postgres-19/src/backend/executor/nodeTidrangescan.c), [nodeBitmapHeapscan.c](../../../../raw/postgres-19/src/backend/executor/nodeBitmapHeapscan.c) — `ScanRelIsReadOnly` and where read-only scans set the hint.
- [system_views.sql](../../../../raw/postgres-19/src/backend/catalog/system_views.sql) / [rules.out](../../../../raw/postgres-19/src/test/regress/expected/rules.out) — `pg_stat_autovacuum_scores` view and its regression coverage.
- [001_parallel_autovacuum.pl](../../../../raw/postgres-19/src/test/modules/test_autovacuum/t/001_parallel_autovacuum.pl) — parallel autovacuum TAP test.
- [config.sgml](../../../../raw/postgres-19/doc/src/sgml/config.sgml) / [release-19.sgml](../../../../raw/postgres-19/doc/src/sgml/release-19.sgml) — same-checkout documentation for `autovacuum_max_parallel_workers`, score-weight release-note wording, and the all-visible read-scan release-note entry (used to locate features; behavior cited to source).

## Open Questions

- **On-access VM commit list is a cluster, not a verified-exhaustive count.** Unlike the self-contained `pg_plan_advice` and `REPACK` features, "setting pages all-visible during reads" is the v19 portion of a long-running pruning/freezing/VM refactor. The table above lists the v19-cycle commits I identified as building the capability; it is not claimed to be exhaustive, and it explicitly excludes the pre-v19 foundation `f83d709760d` (2024, "Merge prune, freeze and vacuum WAL record formats"). A full commit-by-commit attribution would need a deeper history pass.
- **Score tuning guidance is not derivable from source.** The source defines the score math and weight ranges but gives no recommended non-default weights; choosing weights for a given workload is an operational question outside this checkout.
- **Verification status.** This page is filed from a claim-to-source map against pinned raw source but has not had an independent claim-by-claim re-verification pass, so `verified_by_agent` remains `not yet` and the title keeps `(unverified)`.

## Navigation

- [v19/index](../../index.md) — PostgreSQL 19 version landing page.
- [versions](../../../versions.md) — version index and source pin manifest.
- [wiki index](../../../index.md) — global catalog.
