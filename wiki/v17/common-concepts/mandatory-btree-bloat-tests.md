---
type: common-concept
version: 17
pinned_commit: 786db8dcf168bd9df8f55047337525ac19118b1c
verified: false
verified_by_agent: not yet
---

# Mandatory B-Tree Bloat Tests (unverified)

## Contents

- [Definition](#definition)
- [Why It Exists](#why-it-exists)
- [How It Works](#how-it-works)
  - [The five phases of a run](#the-five-phases-of-a-run)
  - [The maintenance assumption](#the-maintenance-assumption)
  - [The maintenance must not be defeated](#the-maintenance-must-not-be-defeated)
  - [Recommended: proving VACUUM did all its work](#recommended-proving-vacuum-did-all-its-work)
  - [Rule 1 splitting every recipe at the index build](#rule-1-splitting-every-recipe-at-the-index-build)
  - [Rule 2 the uniform heap-block drain](#rule-2-the-uniform-heap-block-drain)
  - [Rule 3 the simulated auto-analyze](#rule-3-the-simulated-auto-analyze)
  - [The oracle and the verdict bands](#the-oracle-and-the-verdict-bands)
  - [Family 1 the deduplication gate](#family-1-the-deduplication-gate)
  - [Family 2 partial indexes](#family-2-partial-indexes)
  - [Family 3 false-positive constructions](#family-3-false-positive-constructions)
  - [Family 4 false-negative constructions](#family-4-false-negative-constructions)
  - [Family 5 the change A to D controls](#family-5-the-change-a-to-d-controls)
  - [Family 6 the drained queue and zero row counts](#family-6-the-drained-queue-and-zero-row-counts)
  - [Catalog forgeries run last](#catalog-forgeries-run-last)
  - [Feature gates that skip a fixture](#feature-gates-that-skip-a-fixture)
  - [What the suite does not cover](#what-the-suite-does-not-cover)
- [Where It Appears in Source](#where-it-appears-in-source)
  - [Callers and callees the fixtures depend on](#callers-and-callees-the-fixtures-depend-on)
  - [Generated files the fixtures depend on](#generated-files-the-fixtures-depend-on)
  - [Shipped tests that cover the same behavior](#shipped-tests-that-cover-the-same-behavior)
- [Related Structures and Functions](#related-structures-and-functions)
- [Interactions with Other Concepts](#interactions-with-other-concepts)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Definition

The mandatory B-tree bloat tests are this wiki's shared acceptance suite for any claim about how much space a PostgreSQL [B-tree](../../glossary.md#b-tree) index is wasting, or whether it should be rebuilt. The suite is 113 numbered tests, several of them carrying lettered variants, in six families, run on disposable fixtures against one isolated server, where the only oracle is a measured [`REINDEX INDEX`](../../glossary.md#reindex): build a fixture, record its as-built state, churn it, let the method under test decide, rebuild, and compare. Each family targets one engine behavior that makes a catalog-only or page-level reading of [bloat](../../glossary.md#bloat) go wrong - the [deduplication](../../glossary.md#deduplication) equal-image gate, a [partial index](../../glossary.md#partial-index) whose population is not its table's, [statistics](../../glossary.md#statistics) that mislead in one direction, statistics that mislead in the other, the controls that pin a threshold, and the row counts the engine leaves at zero or stale. A method is not "tested" against a claim in this wiki until these families have been run against the exact text that claim describes, and the suite's contract is that a failing test is corrected in the method, not merely reported. Every fixture assumes that normal [autovacuum](../../glossary.md#autovacuum) and auto-analyze maintain every table, and stands in for them by hand: after its churn it runs [`VACUUM ANALYZE`](../../glossary.md#vacuum) on the tables the churn touched, so no fixture scores a method against a table whose churn was never vacuumed or never analyzed, and no fixture may defeat that maintenance, because a `VACUUM ANALYZE` can be skipped, cancelled or stripped of its [removal horizon](../../glossary.md#xmin-horizon) and still return success. Beside that binding rule the page recommends, without requiring it, a procedure that proves from the statement's own report that the maintenance did all its work and escalates when it did not; see [Recommended: proving VACUUM did all its work](#recommended-proving-vacuum-did-all-its-work).

## Why It Exists

Three properties of the v17 engine make an untested bloat method plausible and wrong at the same time, and each is the reason one family exists.

A B-tree's on-disk size does not follow from its row count. Deduplication merges duplicate keys into [posting lists](../../glossary.md#posting-list), so two indexes over identical data can hold their entries in very different numbers of pages depending on one [metapage](../../glossary.md#metapage) flag and one [reloption](../../glossary.md#storage-parameter). The flag is `btm_allequalimage`, recorded once when the index is built and read back by `_bt_metaversion` ([nbtree.h#btm_allequalimage](../../../raw/postgres-17/src/include/access/nbtree.h#L113-L119), [nbtpage.c#_bt_metaversion](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L739-L750)). What it records is the verdict of `_bt_allequalimage`, which refuses `INCLUDE` indexes outright and otherwise calls each key opclass's support function 4, returning false if any opclass lacks the procedure or answers false ([nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)). The build path applies a second, independent switch: it deduplicates only when the index is equal-image, is not unique, and has `deduplicate_items` on ([nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)).

A partial index's population is not its table's row count. The executor skips an index whose predicate the new row fails ([execIndexing.c#partial-predicate-skip](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L378-L387)), and `ANALYZE` estimates the index's own population by counting the sampled rows that pass the predicate and scaling: `tupleFract = numindexrows / numrows`, then `ceil(tupleFract * totalrows)` ([analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)). A method reading the table's `reltuples` for a partial index is reading a different population, and the size of that error is set by the fixture, not by the method.

Row counts in the catalog are written by whichever command ran last, and sometimes not at all. [`reltuples`](../../glossary.md#reltuples-and-relpages) is documented as "not always up-to-date; -1 means unknown" ([pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L64-L66)); a build writes an exact count through `index_update_stats` ([index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2830)); `ANALYZE` writes the scaled estimate above through `vac_update_relstats` ([analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660), [vacuum.c#vac_update_relstats](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1410-L1425)); `VACUUM` writes an exact count only when the index AM did not mark it estimated ([vacuumlazy.c#update_relstats_all_indexes](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3096)), and `btvacuumcleanup` marks it estimated for a cleanup-only scan ([nbtree.c#btvacuumcleanup-estimated_count](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L884-L894)); and a new relfilenode resets it to `-1` ([relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3945-L3953)). Three of those writers have a fixture in family 6: the build, `ANALYZE`, and the new relfilenode. `VACUUM` has none, because the fixtures that ran one and then withheld the `ANALYZE` are no longer in the suite; which command last wrote a remaining fixture's count follows from rule 3's census rather than from its recipe.

The families of deliberately misleading fixtures exist for the fourth reason: a bloat method has two ways to be wrong, and only one of them is visible without an oracle. Family 3 builds indexes that are fresh and healthy but whose statistics invite a rebuild; family 4 builds indexes that are genuinely reclaimable but whose statistics invite doing nothing. Without both, a method can be tuned until it looks right on whichever error it happens to make.

The maintenance assumption exists for a fifth reason: a table whose churn was never followed by a `VACUUM` or an `ANALYZE` is a state the engine's default configuration does not leave behind. Autovacuum is on by default, and each worker checks every table in its database and runs `VACUUM` and/or `ANALYZE` as needed ([guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457), [maintenance.sgml#autovacuum-daemon](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L853-L885)). A method scored against a table that never got its maintenance is scored against a fault in operations rather than in the method, and the verdict says nothing about how the method behaves on a database anyone runs.

The prohibition beside it exists for a sixth reason: a maintenance command can run to completion and still do nothing to the index. `VACUUM` takes its removal horizon from `GetOldestNonRemovableTransactionId` ([vacuum.c#vacuum_get_cutoffs-OldestXmin](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1109-L1122), [procarray.c#GetOldestNonRemovableTransactionId](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2005-L2026)), and that computation folds in every backend's `xmin` and every [replication slot](../../glossary.md#replication-slot)'s, exempting only a backend that is itself vacuuming or decoding ([procarray.c#ComputeXidHorizons-backend-xmin](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1792-L1815), [procarray.c#skip-vacuum-and-decoding](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1817-L1832), [procarray.c#slot-horizons](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1896-L1902)). Anything that horizon still covers stays [`HEAPTUPLE_RECENTLY_DEAD`](../../glossary.md#dead-tuple) and is only counted ([vacuumlazy.c#recently_dead_tuples](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L208-L212), [vacuumlazy.c#lazy_scan_noprune-recently-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1770-L1776)), and `lazy_scan_heap` enters [index vacuuming](../../glossary.md#index-vacuuming) only when dead [TIDs](../../glossary.md#tid) were collected ([vacuumlazy.c#lazy_vacuum-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1052)). A drain under a held [snapshot](../../glossary.md#snapshot) therefore leaves the dense-leaf, dead-entry shape the retired fixtures 65, 67, 113a and 113c used to build, with no missing statement in the recipe to show for it.

The recommended completeness procedure answers a seventh property: even when no fixture state defeats it, a `VACUUM` can leave part of its work undone for transient reasons and still return success. A non-aggressive `VACUUM` that cannot take a page's [cleanup lock](../../glossary.md#buffer-pin) processes the page without pruning and counts its dead rows as [missed](../../glossary.md#missed-dead-tuple) ([vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L923-L961), [vacuumlazy.c#missed-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1762-L1769)), and under the default [`INDEX_CLEANUP`](../../glossary.md#index_cleanup) setting it may bypass index vacuuming altogether ([vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934)). Neither raises an error, and the first leaves no trace in the `dead but not yet removable` count that the binding proofs read ([vacuumlazy.c#verbose-tuples-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L657-L663), [vacuumlazy.c#verbose-missed-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668)).

## How It Works

### The five phases of a run

Every fixture goes through the same five phases in one isolated cluster with `autovacuum` off, so no background worker moves a fixture between the phases. `autovacuum` is `PGC_SIGHUP`, so turning it off is a reload, not a restart ([guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457)). The launcher is off for isolation only, not because the fixtures model an unmaintained database: the maintenance it would have run is run by the fixture itself, at a fixed point in the churn phase, under [The maintenance assumption](#the-maintenance-assumption).

| Phase | What happens | Why it is a separate phase |
|---|---|---|
| build | create the table, load it, `ANALYZE`, create the index that is scored | the as-built state is the only state in which the index is known to be dense |
| baseline | record size, the table's `reltuples` and the index's own `reltuples` | a before-and-after method has nothing to compare against otherwise |
| churn | run the recipe's own writes, or the uniform drain, then the maintenance `VACUUM ANALYZE` on every table the churn touched | this is the state the method is asked about, and it is a maintained state |
| decide | run the method under test, unmodified | the text that is scored must be the text that is published |
| oracle | `REINDEX INDEX`, then measure the file again | the only ground truth for "how much would a rebuild give back" |

The assumption and the three rules below are what turn a one-shot estimator fixture into a five-phase fixture, and they are what a consumer page has to state it followed.

### The maintenance assumption

Every fixture assumes that normal autovacuum and auto-analyze maintain every table, and simulates that maintenance by hand. The assumption has three parts:

1. **After its churn, a fixture runs `VACUUM ANALYZE` on every table the churn touched, before the decide phase.** The churn is the recipe's own writes or rule 2's drain, and the tables it touched are the ones whose rows it inserted, deleted or updated. A fixture with no churn - the drain-exempt fixtures listed under rule 2 - has nothing to maintain, and the `ANALYZE` in its build phase is the maintenance it carries into the decide phase.
2. **Disabling autovacuum does not exempt a table.** The suite's cluster runs with `autovacuum` off, and a fixture may also set `autovacuum_enabled = false` on a table through its reloption ([reloptions.c#autovacuum_enabled](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L105-L113)). Either setting stops the launcher's verdicts for the table, wraparound forcing aside ([autovacuum.c#AutoVacuumingActive](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3235-L3241), [autovacuum.c#av_enabled-return](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3048-L3054)); neither stops the fixture from running the maintenance the launcher would have run, because the assumption replaces the launcher rather than depending on it.
3. **No fixture may leave churn unmaintained.** A recipe that deletes, updates or inserts rows and then asks the method about the index with no `VACUUM ANALYZE` in between describes a fixture this page does not define. The retired numbers under [What the suite does not cover](#what-the-suite-does-not-cover) are the fixtures that did exactly that.

A foreground `VACUUM ANALYZE` is a faithful stand-in because an autovacuum worker runs no private kind of maintenance. `autovacuum_do_vac_analyze` builds one `VacuumRelation` for the table and calls `vacuum()` ([autovacuum.c#autovacuum_do_vac_analyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3120-L3147)), the same routine the SQL command reaches from `ExecVacuum` ([vacuum.c#ExecVacuum-vacuum](../../../raw/postgres-17/src/backend/commands/vacuum.c#L450-L451), [vacuum.c#vacuum](../../../raw/postgres-17/src/backend/commands/vacuum.c#L479-L481)). Inside it each relation is vacuumed first and analyzed second when both options are set ([vacuum.c#vacuum-then-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650)), so `VACUUM ANALYZE` leaves `ANALYZE` as the last writer of every `reltuples` on the table: the sampled estimate of [Why It Exists](#why-it-exists), not `VACUUM`'s exact count. The `VACUUM` half reports the live and dead counts it found and zeroes the insert counter ([pgstat_relation.c#report_vacuum-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L233-L246)); the `ANALYZE` half zeroes `mod_since_analyze` ([pgstat_relation.c#report_analyze-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L331-L337)).

What the assumption stands in for is the launcher's pair of verdicts. A launcher at default settings wakes every `autovacuum_naptime` of 60 s ([guc_tables.c#autovacuum_naptime](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3339-L3348)), and its worker walks every table in `pg_class`, asking `relation_needs_vacanalyze` for a vacuum verdict and an analyze verdict at once ([autovacuum.c#do_autovacuum-census](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2021-L2033), [autovacuum.c#dovacuum](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095)). The assumption does not reproduce those verdicts on the vacuum side, and a consumer page may not claim that it does: it runs the `VACUUM ANALYZE` after every churn, whether or not the counters had crossed the launcher's thresholds at that moment. The thresholds are recorded here so that the difference is visible, not so that a run applies them:

| Launcher verdict | Engine test, on effective per-table values | At the defaults | Evidence |
|---|---|---|---|
| vacuum | `dead_tuples > vac_base_thresh + vac_scale_factor * reltuples` | `50 + 0.2 * reltuples` | [autovacuum.c#vacthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3075), [autovacuum.c#dovacuum](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3093-L3094), [guc_tables.c#autovacuum_vacuum_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3349-L3357), [guc_tables.c#autovacuum_vacuum_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3886-L3894) |
| insert vacuum | `ins_since_vacuum > vac_ins_base_thresh + vac_ins_scale_factor * reltuples`, skipped when the base threshold is `-1` | `1000 + 0.2 * reltuples` | [autovacuum.c#vacthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3075), [autovacuum.c#dovacuum](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3093-L3094), [guc_tables.c#autovacuum_vacuum_insert_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3358-L3366), [guc_tables.c#autovacuum_vacuum_insert_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3896-L3904) |
| analyze | `mod_since_analyze > anl_base_thresh + anl_scale_factor * reltuples` | `50 + 0.1 * reltuples` | [autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3076), [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095) |

The per-table values follow the same reloption-over-GUC precedence as rule 3's analyze parameters, and the four vacuum reloptions take `ShareUpdateExclusiveLock` like the analyze pair ([autovacuum.c#vac-effective-values](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2993-L3009), [reloptions.c#autovacuum_vacuum_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L225-L233), [reloptions.c#autovacuum_vacuum_insert_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L234-L242), [reloptions.c#autovacuum_vacuum_scale_factor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L399-L407), [reloptions.c#autovacuum_vacuum_insert_scale_factor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L408-L416)). The counters the launcher reads are the `n_dead_tup`, `n_ins_since_vacuum` and `n_mod_since_analyze` columns of `pg_stat_all_tables` ([system_views.sql#n_dead_tup](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L688), [system_views.sql#n_ins_since_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L690)). All five GUCs are `PGC_SIGHUP`, so a cluster-wide change is a reload. The documentation states the same three formulas ([maintenance.sgml#thresholds](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L899-L952)).

The assumption is deliberately stronger than the launcher, on both sides. On the vacuum side: at the defaults, deleting 90,000 rows of a 500,000-row table leaves 90,000 dead tuples against a threshold of 100,050, so the launcher would have left the index dead but dense, which is the shape the retired fixtures 65 and 67 built; the suite runs the `VACUUM ANALYZE` anyway and no longer scores a method against that shape. On the analyze side: 92 to 95, 98 and 118 are churned, so every one of them is analyzed after its changes whatever its threshold says, where before only the ones past their threshold were, and rule 3's census then finds their counters at zero. What that costs each fixture is recorded in its family section and under [Open Questions](#open-questions).

The `VACUUM ANALYZE` inherits rule 3's first publication point. The churn's counts must be in the shared statistics entry before the `ANALYZE` half runs, because `pgstat_report_analyze` zeroes the counter and forgets changes committed while it was running, so churn published afterwards reads as new churn to the census ([pgstat_relation.c#report_analyze-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L331-L337)). Run `SELECT pg_stat_force_next_flush();` in each churn session, or let it exit, before the maintenance statement; see [Publication before maintenance and before the census](#publication-before-maintenance-and-before-the-census).

### The maintenance must not be defeated

**No fixture may carry a state into its maintenance `VACUUM ANALYZE` that would have stopped a real server's autovacuum, `VACUUM`, `ANALYZE` or auto-analyze from doing the work the assumption credits it with.** Part 3 above forbids skipping the maintenance; this rule forbids sabotaging it. The two failures look nothing alike from the outside: a skipped command leaves a missing statement in the recipe, while a defeated one leaves a statement that ran, returned success and changed nothing, and both produce the shape the retired fixtures built.

Five states are forbidden, on every table a fixture touches, from its first churn statement until rule 3's census has finished.

| Forbidden state | Why it defeats the maintenance | Evidence |
|---|---|---|
| an open transaction or held snapshot anywhere in the cluster, other than a lazy `VACUUM` | `VACUUM`'s `OldestXmin` comes from `GetOldestNonRemovableTransactionId`, which folds in every backend's `xmin`, so nothing that horizon still covers is removable: the drained rows stay `HEAPTUPLE_RECENTLY_DEAD`, no dead TIDs are collected, index vacuuming is never entered, and `btbulkdelete` never runs. A lazy `VACUUM` is the one exemption the engine itself grants, through the `PROC_IN_VACUUM` flag the proc-array scan skips | [vacuum.c#vacuum_get_cutoffs-OldestXmin](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1109-L1122), [procarray.c#ComputeXidHorizons-backend-xmin](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1792-L1815), [vacuumlazy.c#lazy_vacuum-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1052), [vacuum.c#PROC_IN_VACUUM](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2006-L2029), [procarray.c#skip-vacuum-and-decoding](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1817-L1832) |
| a prepared transaction | its dummy `PGPROC` is what keeps the xid considered running, so the horizon stays pinned after the session that prepared it is gone, with no live session to show for it | [twophase.c#dummy-pgproc](../../../raw/postgres-17/src/backend/access/transam/twophase.c#L24-L26) |
| a replication slot holding an `xmin` | the horizon computation takes the older of its own answer and the slot's `xmin`, for shared and non-shared relations alike | [procarray.c#slot-horizons](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1896-L1902) |
| a lock on the table conflicting with `ShareUpdateExclusiveLock`, held while the maintenance statement runs | that is the lock `VACUUM` and `ANALYZE` take, so a foreground maintenance statement blocks on it. A real non-wraparound worker does not block: it is given `VACOPT_SKIP_LOCKED`, takes the lock conditionally and gives up with `lock not available`, and a backend that queues behind one signals it to cancel | [vacuum.c#vacuum-lockmode](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2056), [analyze.c#analyze-lockmode](../../../raw/postgres-17/src/backend/commands/analyze.c#L134-L145), [autovacuum.c#skip-locked](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2825-L2830), [vacuum.c#vacuum_open_relation-conditional](../../../raw/postgres-17/src/backend/commands/vacuum.c#L785-L800), [vacuum.c#skip-lock-not-available](../../../raw/postgres-17/src/backend/commands/vacuum.c#L828-L855), [proc.c#cancel-blocking-autovacuum](../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1472) |
| a `statement_timeout`, `transaction_timeout` or `lock_timeout` short enough to fire inside the maintenance statement | it dies part-done, leaving a fixture that is neither churned nor maintained. All three are `PGC_USERSET`, so session or transaction scope, and an autovacuum worker forces each of them, plus `idle_in_transaction_session_timeout`, to `0` for this exact reason: "to avoid letting these settings prevent regular maintenance from being executed" | [guc_tables.c#statement_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631), [guc_tables.c#idle_in_transaction_session_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2633-L2642), [guc_tables.c#transaction_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2644-L2653), [autovacuum.c#launcher-timeouts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L518-L526), [autovacuum.c#worker-timeouts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L1462-L1470) |

**A disabled launcher is not a defeat.** `autovacuum = off` and a per-table `autovacuum_enabled = false` stop the launcher's verdicts, and part 2 of the assumption already accounts for them: the fixture runs the maintenance the launcher would have run ([autovacuum.c#AutoVacuumingActive](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3235-L3241), [autovacuum.c#av_enabled-return](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3048-L3054)). The five states above are different in kind: the maintenance statement runs and cannot do its work.

**What a pinned horizon costs a B-tree fixture, specifically.** It costs the drain twice. No entry is removed, so `_bt_pagedel` empties nothing and the leaf pages stay as dense as the build left them while a rebuild would write a much smaller file. And deleted pages stay unusable. A page an earlier `VACUUM` deleted goes into the free space map only when `btvacuumpage` finds that `BTPageIsRecyclable` passes, and a page this `VACUUM` deletes only when `_bt_pendingfsm_finalize`, which stops at the first page `GlobalVisCheckRemovableFullXid` refuses, finds its `safexid` behind the horizon; both are the same horizon test ([nbtree.c#btvacuumpage-recycle](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1179), [nbtpage.c#pendingfsm-horizon](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3023-L3052), [nbtree.h#BTPageIsRecyclable](../../../raw/postgres-17/src/include/access/nbtree.h#L279-L318)). Under a pinned horizon they therefore stay out of the map, where a later page split could otherwise reuse them ([nbtpage.c#_bt_allocbuf-reuse](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L958)). `pgstatindex` reports them as `deleted_pages` either way, because it counts every deleted page, recyclable or not ([pgstatindex.c#deleted-pages-count](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L307)).

What a run records, per fixture:

| Proof | What it establishes |
|---|---|
| the maintenance statement completed, and no `skipping vacuum of ... --- lock not available` or `skipping analyze of ...` line names the fixture's table | the command was neither skipped for want of the lock nor cut short |
| `VACUUM (VERBOSE)`'s `tuples: ... are dead but not yet removable` count for every churned table ([vacuumlazy.c#verbose-tuples-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L657-L663)) | a nonzero count on a drained or churned fixture is a pinned horizon; the fixture is repaired and re-run, not scored |
| the horizon holders at that moment: `pg_stat_activity`'s `xact_start` and `backend_xmin`, and `pg_replication_slots`' `xmin` ([system_views.sql#pg_stat_activity-xmin](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L878-L885), [system_views.sql#pg_replication_slots-xmin](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1006-L1017)) | which session or slot pinned it, when one did. These are reads, not interlocks; see [Open Questions](#open-questions) |
| the index's page classes after the maintenance, `deleted_pages` and `empty_pages` ([pgstatindex.c#page-classes](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L295-L320)) | the page shape the maintenance left. `deleted_pages` counts recyclable and stuck pages alike ([pgstatindex.c#deleted-pages-count](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L307)), so it cannot show a pinned horizon on its own; the `VERBOSE` index line's `reusable` against `currently deleted` can, after a later `VACUUM`. See [The index half on a B-tree](#the-index-half-on-a-b-tree) |

These four proofs stay the binding minimum. They do not catch a cleanup-lock miss or an index-vacuum bypass, because both leave the `dead but not yet removable` count at 0 ([vacuumlazy.c#verbose-tuples-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L657-L663), [vacuumlazy.c#verbose-missed-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668), [vacuumlazy.c#verbose-index-scan-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L717)). [Recommended: proving VACUUM did all its work](#recommended-proving-vacuum-did-all-its-work) adds the checks that do.

**No fixture is exempt today.** The suite defines no test whose point is a pinned horizon, a skipped maintenance command or a cancelled one, so there is no exemption list to apply. A fixture added later whose point is one of those states is filed as its own numbered test, with the state declared in its recipe and its reading published as a defeated-maintenance reading rather than a maintained one. The index-vacuum bypass is not such a state and stays where it is, under [What the suite does not cover](#what-the-suite-does-not-cover): it is the engine declining to remove entries rather than the fixture preventing it, and it keeps index cleanup ([vacuumlazy.c#bypass-applies](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1957)). The recommended procedure detects it and names the declared escalation that overrides it; see [The INDEX_CLEANUP ON escalation](#the-index_cleanup-on-escalation).

### Recommended: proving VACUUM did all its work

**This procedure is recommended, not required.** [The maintenance must not be defeated](#the-maintenance-must-not-be-defeated) stays the binding rule, and its proofs stay the minimum a conforming run records. This section adds a way for a tester to show, for one maintenance statement, that every part of [`VACUUM`](../../glossary.md#vacuum)'s work actually happened, and what to do when a part did not.

The behavior it guards against is easy to miss: a `VACUUM` that returns success can still leave work undone, and most of the ways it does so raise no error. It can skip the table before it starts. It can keep [dead rows](../../glossary.md#dead-tuple) that an old [snapshot](../../glossary.md#snapshot) can still see, or leave dead rows on a page it could not lock for cleanup. It can decline [index vacuuming](../../glossary.md#index-vacuuming), give up [truncation](../../glossary.md#truncation), or never reach the [TOAST](../../glossary.md#toast) table. Each of those leaves a trace, either in the [report `VERBOSE` prints](../../glossary.md#vacuum-verbose) or in the statistics the statement writes. So the procedure removes every cause it can before the statement runs, and then proves completion from those traces rather than from the exit status.

#### The values the proof reads

Every check below reads one of these values. The first six exist only while one statement runs and are published once, in its `VERBOSE` report. The last three are stored, and later commands overwrite them, which is why the procedure reads them at a fixed point.

| Value | What it means | Where it comes from | Live or stored | How the proof reads it |
|---|---|---|---|---|
| removable cutoff (`OldestXmin`) | the oldest [transaction ID](../../glossary.md#transaction-id) that a session, a [prepared transaction](../../glossary.md#two-phase-commit) or a [replication slot](../../glossary.md#replication-slot) may still need. A row stays when its deleting transaction is not older than the cutoff ([heapam_visibility.c#HeapTupleSatisfiesVacuum](../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1162-L1181)) | `vacuum_get_cutoffs` calls `GetOldestNonRemovableTransactionId` once, when the table's heap pass starts ([vacuum.c#vacuum_get_cutoffs-OldestXmin](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1109-L1122), [vacuumlazy.c#heap_vacuum_rel-cutoffs](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L433-L451)) | live, sampled once per table | the `removable cutoff:` line ([vacuumlazy.c#verbose-cutoff-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L669-L673)) |
| dead but not yet removable | dead row versions the cutoff still protects | counted while pruning, and by the no-prune path ([vacuumlazy.c#recently_dead_tuples](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L208-L212), [vacuumlazy.c#lazy_scan_noprune-recently-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1770-L1776)) | one statement | the third number on the `tuples:` line ([vacuumlazy.c#verbose-tuples-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L657-L663)) |
| [missed dead tuples](../../glossary.md#missed-dead-tuple) | dead rows the cutoff no longer protects, left on a page `VACUUM` could not lock for cleanup | the no-prune path counts each `HEAPTUPLE_DEAD` row as missed, and each page that had one ([vacuumlazy.c#missed-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1762-L1769), [vacuumlazy.c#missed-dead-pages](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1834-L1838)) | one statement | the `tuples missed:` line, printed only when the count is nonzero ([vacuumlazy.c#verbose-missed-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668)) |
| dead item identifiers | `LP_DEAD` [line pointers](../../glossary.md#line-pointer) whose index entries must go before the pointers can be reused | collected by pruning and by the no-prune path ([vacuumlazy.c#lazy_scan_prune-lpdead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1488-L1505), [vacuumlazy.c#noprune-lpdead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1799-L1829)) | one statement | the `index scan ...:` line ([vacuumlazy.c#verbose-index-scan-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L717)) |
| the three phase switches | whether index vacuuming, index cleanup and truncation run | set from the `INDEX_CLEANUP` and `TRUNCATE` options or, when the command gives none, from the table's [storage parameters](../../glossary.md#storage-parameter); then cleared by the bypass or the [failsafe](../../glossary.md#vacuum-failsafe) ([vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407), [vacuum.c#index_cleanup-reloption](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2179), [vacuum.c#truncate-reloption](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2190-L2201), [vacuumlazy.c#bypass-applies](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1957), [vacuumlazy.c#failsafe-disables-cleanup](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2320-L2326)) | one statement | the label of the `index scan` line, and the truncation messages |
| per-index result | each index access method's own counts: pages in total, newly deleted, currently deleted, reusable | returned by `ambulkdelete` and `amvacuumcleanup`, and NULL when the access method did nothing ([genam.h#IndexBulkDeleteResult](../../../raw/postgres-17/src/include/access/genam.h#L56-L84), [vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732)) | one statement | one `index "...": pages:` line per non-NULL result |
| `vacuum_count`, `last_vacuum` | that the heap pass reached its end | written only by `pgstat_report_vacuum`, the last statistics step of `heap_vacuum_rel` ([vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591), [pgstat_relation.c#pgstat_report_vacuum](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L227-L257), [system_views.sql#vacuum-counters](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L691-L698)) | stored in [cumulative statistics](../../glossary.md#cumulative-statistics) | one more than before, for every relation the statement vacuumed |
| `n_dead_tup` | dead rows left behind | the `VACUUM` half writes dead-but-not-removable plus missed; the `ANALYZE` half then overwrites it with its own sample's count ([vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591), [pgstat_relation.c#report_analyze-dead](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L329), [analyze.c#report-analyze-dead](../../../raw/postgres-17/src/backend/commands/analyze.c#L688-L692)) | stored, rewritten by each command | zero after the statement |
| `relpages`, `relallvisible` | the table's page count, and how many of its pages the [visibility map](../../glossary.md#visibility-map) marks all-visible | written by the `VACUUM` half, then rewritten by the `ANALYZE` half from the live block count and the live visibility map ([vacuumlazy.c#relallvisible](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L556-L575), [analyze.c#relpages](../../../raw/postgres-17/src/backend/commands/analyze.c#L190-L197), [analyze.c#analyze-relstats](../../../raw/postgres-17/src/backend/commands/analyze.c#L623-L645)) | stored | equal after the statement, all-zero pages aside |

#### Where VACUUM can leave work undone

The map follows one tier-0 statement, `VACUUM (VERBOSE, ANALYZE, PARALLEL 0)`, on one table, in execution order. Each branch on the left is a way part of the work is not done; the right column is the trace that branch leaves. The numbered nodes are keyed to their evidence in the table below the map.

```text
VACUUM (VERBOSE, ANALYZE, PARALLEL 0) t                 trace when the work is not done
|
+-[1] open t and take ShareUpdateExclusiveLock
|     +- SKIP_LOCKED given and the lock is held ....... WARNING: skipping vacuum of "t" --- lock not available
|     +- role lacks MAINTAIN on t ..................... WARNING: permission denied to vacuum "t", skipping it
|     +- t is another session's temporary table ....... nothing; vacuum_count does not move
|     +- otherwise wait for the lock .................. wait event type Lock while it waits
|        (a relation skipped at [1] also skips its ANALYZE)
|
+-[2] take the removal cutoff ......................... removable cutoff: X, which was N XIDs old ...
|
+-[3] heap pass, page by page
|     +- all-visible, in a run of >= 32 such pages,
|     |  and the VACUUM is not aggressive ............. skipped by design; see "pages: ... scanned (P% of total)"
|     +- cleanup lock granted: prune and freeze
|     |     +- row deleted by a transaction not
|     |        older than the cutoff .................. tuples: ..., D are dead but not yet removable
|     +- cleanup lock refused (another pin on the page)
|           +- not aggressive, or no tuple to freeze .. tuples missed: M dead from P pages ...
|           +- aggressive, and a tuple older than
|              FreezeLimit: wait, then prune .......... wait event type BufferPin while it waits
|
+-[4] index vacuuming, only if dead item identifiers were collected
|     +- INDEX_CLEANUP off (option or storage parameter) index scan bypassed: ...
|     +- AUTO, under 2 % of pages with dead items,
|     |  under 32 MB of TIDs .......................... index scan bypassed: ...   (index cleanup still runs)
|     +- wraparound failsafe .......................... WARNING: bypassing nonessential maintenance ...
|     |                                                 index scan bypassed by failsafe: ...
|     +- otherwise: ambulkdelete per index, then the
|        second heap pass ............................. index scan needed: ... dead item identifiers removed
|
+-[5] index cleanup (amvacuumcleanup), unless INDEX_CLEANUP off or failsafe
|                                                       index "i": pages: T in total, N newly deleted,
|                                                       C currently deleted, R reusable
+-[6] truncation of empty pages at the end of the heap
|     +- TRUNCATE off, vacuum_truncate = false, or failsafe ..... nothing; no attempt
|     +- under 1000 and under 1/16 of the pages freeable ........ nothing; no attempt, by design
|     +- AccessExclusiveLock not granted within about 5 s ....... "t": stopping truncate due to conflicting lock request
|     +- the table grew while VACUUM ran ........................ nothing; gives up
|     +- another session queues for a lock on t ................. table "t": suspending truncate due to conflicting lock request
|
+-[7] write pg_class and the statistics ............... vacuum_count + 1, n_dead_tup = D + M
|
+-[8] TOAST table: [1] to [7] again for pg_toast.pg_toast_<oid>
|                                                       a second "finished vacuuming" report
+-[9] ANALYZE half, unless [1] skipped t .............. "t": scanned S of T pages, containing L live rows and K dead rows; ...
|
+-[10] pg_database.datfrozenxid, unless SKIP_DATABASE_STATS
```

| Node | What decides the branch | Evidence |
|---|---|---|
| [1] | `vacuum_rel` opens the table through `vacuum_open_relation` with [`ShareUpdateExclusiveLock`](../../glossary.md#shareupdateexclusivelock), which waits for the lock unless `SKIP_LOCKED` is given. It then checks the role's privilege, the relation kind and whether the table is another session's temporary table, and returns false on any of them; a false return also skips that relation's `ANALYZE` | [vacuum.c#vacuum-lockmode](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2056), [vacuum.c#vacuum_open_relation-conditional](../../../raw/postgres-17/src/backend/commands/vacuum.c#L785-L800), [vacuum.c#skip-lock-not-available](../../../raw/postgres-17/src/backend/commands/vacuum.c#L828-L855), [vacuum.c#vacuum_is_permitted_for_relation](../../../raw/postgres-17/src/backend/commands/vacuum.c#L732-L758), [vacuum.c#relkind-skip](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2096-L2111), [vacuum.c#other-temp-silent](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2113-L2126), [vacuum.c#skipped-vacuum-skips-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L622-L634) |
| [2] | the cutoff for an ordinary table folds in the xid and xmin of every backend connected to the same database, except a lazy `VACUUM` or a logical-decoding backend. Every database's horizon also takes every slot's `xmin` and a physical [WAL sender](../../glossary.md#wal-sender)'s [hot standby](../../glossary.md#hot-standby) feedback, which lands in the sender's own xmin or, when it uses a slot, in the slot's. A slot's `catalog_xmin` holds back catalogs only | [procarray.c#skip-vacuum-and-decoding](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1817-L1832), [procarray.c#data-horizon-same-database](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1838-L1864), [procarray.c#slot-horizons](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1896-L1902), [procarray.c#slot-catalog-xmin](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1904-L1918), [walsender.c#affects-all-horizons](../../../raw/postgres-17/src/backend/replication/walsender.c#L318-L331), [walsender.c#feedback-xmin](../../../raw/postgres-17/src/backend/replication/walsender.c#L2740-L2749) |
| [3] | a page is skipped when the visibility map marks it all-visible, except the last page, any page under `DISABLE_PAGE_SKIPPING`, and an all-visible but not all-frozen page in an [aggressive VACUUM](../../glossary.md#aggressive-vacuum); a run shorter than 32 pages is read anyway. A scanned page needs a [cleanup lock](../../glossary.md#buffer-pin), which `ConditionalLockBufferForCleanup` refuses while anyone else holds a pin; the no-prune fallback then waits only when an aggressive VACUUM meets a tuple older than its freeze cutoffs | [vacuumlazy.c#SKIP_PAGES_THRESHOLD](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L107-L111), [vacuumlazy.c#skip-range](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1138-L1143), [vacuumlazy.c#find_next_unskippable_block](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1203-L1245), [vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L923-L961), [vacuumlazy.c#noprune-aggressive-wait](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1713-L1733), [bufmgr.c#ConditionalLockBufferForCleanup](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5456-L5508), [bufmgr.c#LockBufferForCleanup](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5301-L5330), [bufmgr.c:5411](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5411) |
| [4] | index vacuuming starts only when dead item identifiers were collected. `INDEX_CLEANUP` off clears it from the start, `AUTO` lets the bypass test decide, and the failsafe clears it mid-run | [vacuumlazy.c#lazy_vacuum-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1052), [vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407), [vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934), [vacuumlazy.c#bypass-applies](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1957), [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2299-L2347) |
| [5] | index cleanup runs while its switch is on, and each access method decides what its cleanup does and whether it returns a result | [vacuumlazy.c#lazy_cleanup_all_indexes-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066), [vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732) |
| [6] | truncation is attempted only past the size threshold and with its switch on; the lock is taken conditionally and retried every 50 ms for up to 5 s, and the scan of end pages stops early for a waiting locker | [vacuumlazy.c#truncate-constants](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L64-L83), [vacuumlazy.c#should_attempt_truncation](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2529-L2544), [vacuumlazy.c#truncate-lock-retry](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2577-L2608), [vacuumlazy.c#truncate-rel-grew](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2610-L2627), [vacuumlazy.c#truncated-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2664-L2672), [vacuumlazy.c#truncate-suspend](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2716-L2735) |
| [7] | `pg_class` gets the new page count, all-visible count and freeze horizons; the statistics get the live count, dead-but-not-removable plus missed, and one more `vacuum_count` | [vacuumlazy.c#relallvisible](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L556-L575), [vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591) |
| [8] | `PROCESS_TOAST` is on by default; `vacuum_rel` remembers the TOAST table and recurses into it after committing the main table | [vacuum.c#ExecVacuum-defaults](../../../raw/postgres-17/src/backend/commands/vacuum.c#L153-L169), [vacuum.c#toast-relid](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2212-L2223), [vacuum.c#toast-recursion](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2282-L2300) |
| [9] | `vacuum()` analyzes each relation right after vacuuming it, and `ANALYZE` reports its sample in one line | [vacuum.c#vacuum-then-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650), [analyze.c#verbose-sample-line](../../../raw/postgres-17/src/backend/commands/analyze.c#L1296-L1306) |
| [10] | after the relation loop, `vacuum()` updates `datfrozenxid` unless told to skip it | [vacuum.c#datfrozenxid](../../../raw/postgres-17/src/backend/commands/vacuum.c#L700-L707) |

#### Stage A: remove the causes before the statement

Run these checks immediately before the maintenance statement, in the session that will run it, so that "no other session" in check 2 means exactly that. Each check removes a branch of the map, so the statement can no longer take it.

1. **No horizon holder.** The first query below must return no row. A backend with an xid or xmin, a prepared transaction or a slot with an `xmin` moves the cutoff of node [2] back, and rows deleted after it then stay as dead but not yet removable. The engine counts only same-database backends for an ordinary table ([procarray.c#data-horizon-same-database](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1838-L1864)); the query looks at the whole cluster anyway, because the stricter check costs nothing and also covers the shared catalogs. A prepared transaction can exist only when `max_prepared_transactions` is above its default of 0, because `PREPARE TRANSACTION` fails outright at 0; the setting has `postmaster` context, so changing it needs a restart ([guc_tables.c#max_prepared_transactions](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2579-L2585), [twophase.c#prepared-disabled](../../../raw/postgres-17/src/backend/access/transam/twophase.c#L371-L376)).
2. **No other session on the database, and no lock on the table.** The second query must list no other client backend, and the third no lock held or awaited by anyone else on the table, its TOAST table or any of their indexes. No core view shows buffer pins, and `pg_buffercache`'s `pinning_backends` counts them per buffer without naming the backend ([pg_buffercache--1.2.sql#pinning_backends](../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache--1.2.sql#L17), [pgbuffercache.sgml#pinning_backends](../../../raw/postgres-17/doc/src/sgml/pgbuffercache.sgml#L166-L172)). A session that is not connected holds no pin, though, so ruling out the session rules out its pins at node [3]. Any lock at all blocks truncation's [`AccessExclusiveLock`](../../glossary.md#accessexclusivelock) at node [6], and a lock that conflicts with `ShareUpdateExclusiveLock` makes the statement itself wait at node [1]. A prepared transaction's locks show in `pg_locks` with no `pid` ([pg_proc.dat#pg_lock_status](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L6308-L6313), [system_views.sql#pg_locks](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L401-L402)), which is why the query compares with `IS DISTINCT FROM`.
3. **No dirty buffers left by the churn.** Run `CHECKPOINT`. The [checkpointer](../../glossary.md#checkpointer) and the [background writer](../../glossary.md#background-writer) pin and share-lock a dirty buffer while they write it ([bufmgr.c#SyncOneBuffer-pin](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3546-L3567)), and a pin held at that moment is enough to refuse the cleanup lock of node [3] ([bufmgr.c#ConditionalLockBufferForCleanup](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5456-L5508)). After the checkpoint the churned pages are clean, so neither process has a reason to write them while `VACUUM` reads them. `CHECKPOINT` requires the privileges of `pg_checkpoint` ([utility.c#checkpoint-privilege](../../../raw/postgres-17/src/backend/tcop/utility.c#L942-L950)). It reduces the risk rather than removing it, which is why Stage C still checks for missed tuples.
4. **No phase switched off by a storage parameter.** The fourth query must show no `vacuum_index_cleanup` of `off` and no `vacuum_truncate` of `false` on the table or its TOAST table. Their defaults are `auto` and `true` ([reloptions.c#vacuum_index_cleanup](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L510-L520), [reloptions.c#vacuum_truncate](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L151-L158)), and a command that gives neither option takes them from the parameters ([vacuum.c#index_cleanup-reloption](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2179), [vacuum.c#truncate-reloption](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2190-L2201)).
5. **Far from the failsafe.** The same query must show `age(relfrozenxid)` below the larger of `vacuum_failsafe_age` and 1.05 times `autovacuum_freeze_max_age`, and `mxid_age(relminmxid)` below the MultiXact equivalent, which is the test `vacuum_xid_failsafe_check` applies ([vacuum.c#vacuum_xid_failsafe_check](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1252-L1298)). The defaults are 1.6 billion and 200 million; the failsafe ages have `user` context and the `autovacuum_*` ages `postmaster` context ([guc_tables.c#vacuum_failsafe_age](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2707-L2723), [guc_tables.c#autovacuum_freeze_max_age](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3378-L3398)). A fresh fixture is nowhere near either.
6. **The churn's statistics are published.** Every churn session has exited or run `SELECT pg_stat_force_next_flush();`, as [Publication before maintenance and before the census](#publication-before-maintenance-and-before-the-census) requires. Otherwise a flush that lands after the statement adds to `n_dead_tup` and Stage D reads a count the statement never left.
7. **The maintenance session cannot be cut short.** Set its four timeouts to 0. That is each one's boot value ([guc_tables.c#statement_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631), [guc_tables.c#idle_in_transaction_session_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2633-L2642), [guc_tables.c#transaction_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2644-L2653)), so the `SET` only undoes a role, database or configuration-file value. All four have `user` context, so the `SET` applies to the session.
8. **Record the counters.** Keep `vacuum_count` and `analyze_count` for the table and its TOAST table, so that Stage D can check that each moved by exactly one.

The probe queries carry their own `statement_timeout` and `lock_timeout`, so a probe that blocks fails instead of stalling the run. `public.t` stands for the table being maintained.

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';

-- 1. Horizon holders anywhere in the cluster: expect no row.
SELECT /* wiki_vacuum_gate_horizon */
       'backend' AS holder, pid::text AS id, datname::text AS db,
       backend_type AS detail, backend_xid::text AS xid, backend_xmin::text AS xmin
  FROM pg_stat_activity
 WHERE pid <> pg_backend_pid()
   AND (backend_xid IS NOT NULL OR backend_xmin IS NOT NULL)
UNION ALL
SELECT 'prepared', gid, database::text, owner::text, transaction::text, NULL
  FROM pg_prepared_xacts
UNION ALL
SELECT 'slot', slot_name::text, database::text, slot_type, NULL,
       coalesce(xmin::text, '-') || ' / ' || coalesce(catalog_xmin::text, '-')
  FROM pg_replication_slots
 WHERE xmin IS NOT NULL OR catalog_xmin IS NOT NULL;

-- 2. Other sessions on this database: expect no row.
SELECT /* wiki_vacuum_gate_sessions */ pid, usename, application_name, state, backend_type
  FROM pg_stat_activity
 WHERE pid <> pg_backend_pid()
   AND datname = current_database()
   AND backend_type IN ('client backend', 'autovacuum worker');

-- 3. Locks held or awaited by anyone else on the table, its TOAST table and their indexes: expect no row.
WITH /* wiki_vacuum_gate_locks */ heap AS (
       SELECT c.oid FROM pg_class c WHERE c.oid = 'public.t'::regclass
       UNION ALL
       SELECT c.reltoastrelid FROM pg_class c
        WHERE c.oid = 'public.t'::regclass AND c.reltoastrelid <> 0
     ), rels AS (
       SELECT oid FROM heap
       UNION ALL
       SELECT i.indexrelid FROM pg_index i JOIN heap h ON i.indrelid = h.oid
     )
SELECT l.pid, l.relation::regclass AS rel, l.mode, l.granted
  FROM pg_locks l
 WHERE l.locktype = 'relation'
   AND l.database = (SELECT oid FROM pg_database WHERE datname = current_database())
   AND l.relation IN (SELECT oid FROM rels)
   AND l.pid IS DISTINCT FROM pg_backend_pid();

-- 4 and 5. Storage parameters and distance to the failsafe, for the table and its TOAST table.
SELECT /* wiki_vacuum_gate_options */ c.oid::regclass AS rel, c.reloptions,
       age(c.relfrozenxid) AS xid_age,
       greatest(current_setting('vacuum_failsafe_age')::numeric,
                current_setting('autovacuum_freeze_max_age')::numeric * 1.05) AS xid_failsafe_age,
       mxid_age(c.relminmxid) AS mxid_age,
       greatest(current_setting('vacuum_multixact_failsafe_age')::numeric,
                current_setting('autovacuum_multixact_freeze_max_age')::numeric * 1.05) AS mxid_failsafe_age
  FROM pg_class c
 WHERE c.oid IN ('public.t'::regclass::oid,
                 (SELECT reltoastrelid FROM pg_class WHERE oid = 'public.t'::regclass));

-- 8. Counters before the statement.
SELECT /* wiki_vacuum_gate_counters */ relid::regclass AS rel, vacuum_count, analyze_count
  FROM pg_stat_all_tables
 WHERE relid IN ('public.t'::regclass::oid,
                 (SELECT reltoastrelid FROM pg_class WHERE oid = 'public.t'::regclass));

CHECKPOINT;
```

Every column these queries read is defined at the pin: `pg_stat_activity` ([system_views.sql#pg_stat_activity](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L865-L891)), `pg_prepared_xacts` ([system_views.sql#pg_prepared_xacts](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L421-L426)), `pg_replication_slots` ([system_views.sql#pg_replication_slots](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1006-L1029)), `pg_locks` ([pg_proc.dat#pg_lock_status](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L6308-L6313)) and `pg_stat_all_tables` ([system_views.sql#pg_stat_all_tables](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703)); the two `backend_type` names are the engine's own ([miscinit.c#backend-type-names](../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L277-L283)).

#### Stage B: run the statement and keep its report

Run one statement per table, so each report belongs to one table and its TOAST table:

```sql
SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET transaction_timeout = 0;
VACUUM /* wiki_vacuum_maintenance */ (VERBOSE, ANALYZE, PARALLEL 0) public.t;
```

The options stay at their defaults except `VERBOSE`, `ANALYZE` and `PARALLEL 0`, which together are tier 0 of the escalation in Stage E. `PARALLEL 0` is there because an autovacuum worker never vacuums indexes in parallel, while the SQL command does by default whenever at least two of the table's indexes support [parallel vacuum](../../glossary.md#parallel-vacuum) and reach `min_parallel_index_scan_size`; the option sets the worker count to -1, so `dead_items_alloc` never starts a parallel context ([autovacuum.c#no-parallel](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2839-L2840), [vacuum.c#parallel-default](../../../raw/postgres-17/src/backend/commands/vacuum.c#L171-L172), [vacuum.c#PARALLEL-zero](../../../raw/postgres-17/src/backend/commands/vacuum.c#L277-L282), [vacuumlazy.c#parallel-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2830-L2857), [vacuumparallel.c#parallel_vacuum_compute_workers](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L547-L601)). A parallel run would skip no index, because the leader processes every index no worker took ([vacuumparallel.c#leader-participates](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L719-L726)), so the option changes which process does the work, not how much of it is done. Setting `max_parallel_maintenance_workers` to 0 has the same effect for every statement in the session: the setting has `user` context and a default of 2, and the worker computation returns 0 when it is 0 ([guc_tables.c#max_parallel_maintenance_workers](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417), [vacuumparallel.c#parallelism-disabled](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L555-L560)). Six options are never used, because each one creates a branch of the map on purpose: `SKIP_LOCKED` turns the wait of node [1] into a skip, `INDEX_CLEANUP OFF` and `TRUNCATE false` switch off nodes [4] to [6], `PROCESS_TOAST false` and `PROCESS_MAIN false` drop node [8] or the main table, and `SKIP_DATABASE_STATS` drops node [10] ([vacuum.c#ExecVacuum-defaults](../../../raw/postgres-17/src/backend/commands/vacuum.c#L153-L169), [ref/vacuum.sgml#options](../../../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L112-L323)).

The report is made of server notices. `INFO` always reaches the client, whatever `client_min_messages` says ([elog.c#should_output_to_client](../../../raw/postgres-17/src/backend/utils/error/elog.c#L248-L264)), and `psql` writes every notice to standard error, prefixed with the script name and line number when it reads a file ([common.c#NoticeProcessor](../../../raw/postgres-17/src/bin/psql/common.c#L269-L273), [startup.c#log_locus_callback](../../../raw/postgres-17/src/bin/psql/startup.c#L99-L111), [logging.c#locus-prefix](../../../raw/postgres-17/src/common/logging.c#L244-L261)). So capture standard error to one file per table, for example `psql -X -v ON_ERROR_STOP=1 -f vacuum_t.sql 2> vacuum_t.log`. Each report's first line carries the prefix; its other lines start at column 0, because the server sends the report as one multi-line message ([vacuumlazy.c#report-ereport](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L762-L763)).

A second session may watch the statement through [progress reporting](../../glossary.md#progress-reporting) in `pg_stat_progress_vacuum` and the [wait event](../../glossary.md#wait-event) columns of `pg_stat_activity` ([system_views.sql#pg_stat_progress_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1209-L1227)). A wait of type `Lock` is node [1], type `BufferPin` is the aggressive wait of node [3], and type `Timeout` with event `VacuumTruncate` is the retry loop of node [6] ([wait_event.c#wait-event-types](../../../raw/postgres-17/src/backend/utils/activity/wait_event.c#L391-L409), [wait_event_names.txt#VACUUM_TRUNCATE](../../../raw/postgres-17/src/backend/utils/activity/wait_event_names.txt#L181), [wait_event_names.txt#BUFFER_PIN](../../../raw/postgres-17/src/backend/utils/activity/wait_event_names.txt#L278-L280), [generate-wait_event_types.pl#camel-case](../../../raw/postgres-17/src/backend/utils/activity/generate-wait_event_types.pl#L112-L128)). The watcher must send single statements and never hold a transaction open: a snapshot taken after the churn committed cannot protect the churn's rows, but one taken before it is itself a horizon holder.

#### Stage C: read the report

Each line below is checked for the table and again for its TOAST table, which prints its own block.

| Report line | Pass | What a failure means | Evidence |
|---|---|---|---|
| `vacuuming "db.schema.t"`, or `aggressively vacuuming "db.schema.t"` | present | absent: node [1] skipped the relation | [vacuumlazy.c#verbose-vacuuming-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L469-L481) |
| `finished vacuuming "db.schema.t": index scans: N` | present | absent: the statement stopped before the end of the heap pass | [vacuumlazy.c#finished-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L619-L652) |
| `pages: R removed, N remain, S scanned (P% of total)` | read only | `S` below the total is visibility-map skipping, not undone work | [vacuumlazy.c#pages-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L653-L658) |
| `tuples: X removed, Y remain, D are dead but not yet removable` | `D` is 0 on a churned relation | a horizon holder at node [2] | [vacuumlazy.c#verbose-tuples-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L657-L663) |
| `tuples missed: M dead from P pages not removed due to cleanup lock contention` | absent | a pin at node [3]; the rows and their index entries stay until a later `VACUUM` | [vacuumlazy.c#verbose-missed-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668) |
| `removable cutoff: X, which was N XIDs old when operation ended` | read only | when `D` is not 0, `X` names the oldest transaction that was still considered running | [vacuumlazy.c#verbose-cutoff-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L669-L673) |
| `new relfrozenxid: ...` and `new relminmxid: ...` | read only | absent means the freeze horizon did not advance, which a non-aggressive run that skipped an all-visible page always does, and which a remaining old XID also causes; see [Behavior that looks like a skip but is by design](#behavior-that-looks-like-a-skip-but-is-by-design) | [vacuumlazy.c#keep-relfrozenxid](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L544-L554), [vacuumlazy.c#freeze-lines](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L674-L694), [vacuum.c#relfrozenxid-advance-only](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1489-L1519) |
| `index scan not needed: ...` or `index scan needed: ... removed` | either | - | [vacuumlazy.c#verbose-index-scan-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L717) |
| `index scan bypassed: ...` | absent | `INDEX_CLEANUP` off, or the bypass at node [4]; the dead item identifiers and their index entries stay | [vacuumlazy.c#verbose-index-scan-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L717) |
| `index scan bypassed by failsafe: ...`, `WARNING: bypassing nonessential maintenance of table ...` | absent | the failsafe fired; index vacuuming, index cleanup and truncation stopped | [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2299-L2347) |
| `WARNING: cutoff for removing and freezing tuples is far in the past` | absent | the cutoff is more than `autovacuum_freeze_max_age` behind the next XID | [vacuum.c#cutoff-far-in-past](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1139-L1158) |
| `index "i": pages: T in total, N newly deleted, C currently deleted, R reusable` | per access method; see the index half below | - | [vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732) |
| `WARNING: skipping vacuum of "t" ...`, `WARNING: permission denied to vacuum "t", skipping it`, `WARNING: skipping "t" --- cannot vacuum non-tables or special system tables` | absent | node [1] skipped the relation, and its `ANALYZE` with it | [vacuum.c#skip-lock-not-available](../../../raw/postgres-17/src/backend/commands/vacuum.c#L828-L855), [vacuum.c#vacuum_is_permitted_for_relation](../../../raw/postgres-17/src/backend/commands/vacuum.c#L732-L758), [vacuum.c#relkind-skip](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2096-L2111) |
| `"t": stopping truncate due to conflicting lock request` | absent | truncation gave up at node [6] | [vacuumlazy.c#truncate-lock-retry](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2577-L2608) |
| `table "t": suspending truncate due to conflicting lock request` | absent | truncation stopped part-way for a waiting locker | [vacuumlazy.c#truncate-suspend](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2716-L2735) |
| `table "t": truncated A to B pages` | present when the heap's end was empty past the threshold | absent then: no attempt was made, or a silent give-up | [vacuumlazy.c#truncated-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2664-L2672) |
| `launched N parallel vacuum workers for index vacuuming (planned: M)` | absent under `PARALLEL 0` | present means the statement ran in parallel; `N` below `M` is still not a skip, because the leader processes every index no worker took | [vacuumparallel.c#launched-line](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L705-L716), [vacuumparallel.c#leader-participates](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L719-L726) |
| `"t": scanned S of T pages, containing L live rows and K dead rows; ...` | `K` is 0 | dead rows of any kind remain; the count is exact when `S` equals `T` | [analyze.c#verbose-sample-line](../../../raw/postgres-17/src/backend/commands/analyze.c#L1296-L1306) |

The last line is the strongest single check, for two reasons. `ANALYZE` counts every `LP_DEAD` stub and every dead or recently dead row it reads ([heapam_handler.c#analyze-dead-rows](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1054-L1085)), so it catches a horizon holder, a missed page and a bypass alike. And it reads every block when the table has no more blocks than its sample target, `300` times the statistics target ([analyze.c#two-stage-sample](../../../raw/postgres-17/src/backend/commands/analyze.c#L1136-L1137), [analyze.c#std_typanalyze-minrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L1888-L1895)); above that size `K` is an estimate.

A pin does not always show up as missed tuples. A backend that reads a nearly full page prunes it on the spot when it can take the cleanup lock, before it keeps its pin ([pruneheap.c#heap_page_prune_opt-minfree](../../../raw/postgres-17/src/backend/access/heap/pruneheap.c#L226-L264)). `VACUUM`'s no-prune path then finds dead item identifiers instead of dead rows and collects them ([vacuumlazy.c#noprune-lpdead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1799-L1829)), and the second heap pass can free them despite the pin, because it needs only an exclusive lock ([vacuumlazy.c#heap-pass-exclusive-lock](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2148-L2153)). The same pin can therefore leave no trace at all, or a bypass line when those few pages fall under the bypass test.

These checks need nothing but a POSIX shell. They print the failing lines and nothing else when the report is clean:

```sh
log=vacuum_t.log
grep -c 'INFO:  finished vacuuming "' "$log"                  # one per relation: t, and its TOAST table
grep '^tuples: ' "$log" | grep -v ', 0 are dead but not yet removable$'
grep -E '^tuples missed: |^index scan bypassed|WARNING:  (skipping|permission denied|bypassing nonessential|cutoff for)|(stopping|suspending) truncate' "$log"
grep -E '": scanned [0-9]+ of [0-9]+ pages, containing [0-9]+ live rows and [0-9]+ dead rows' "$log" | grep -v ' and 0 dead rows;'
```

#### Stage D: cross-check the statistics and the catalog

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';
SELECT /* wiki_vacuum_post_check */ s.relid::regclass AS rel,
       s.vacuum_count, s.analyze_count, s.n_dead_tup,
       c.relpages, c.relallvisible,
       pg_relation_size(c.oid) / current_setting('block_size')::int AS blocks_now
  FROM pg_stat_all_tables s
  JOIN pg_class c ON c.oid = s.relid
 WHERE s.relid IN ('public.t'::regclass::oid,
                   (SELECT reltoastrelid FROM pg_class WHERE oid = 'public.t'::regclass));
```

| Check | Pass | What a failure means | Evidence |
|---|---|---|---|
| `vacuum_count` | one more than in Stage A, for the table and for its TOAST table | the relation was skipped at node [1], or the statement died before node [7] | [vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591), [pgstat_relation.c#pgstat_report_vacuum](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L227-L257) |
| `analyze_count` | one more for the table; unchanged for the TOAST table, which `VACUUM` never analyzes | the `ANALYZE` half was skipped | [pgstat_relation.c#analyze-count](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L339-L348), [vacuum.c#toast-recursion](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2282-L2300) |
| `n_dead_tup` | 0 for both | the table: `ANALYZE` saw dead rows. The TOAST table: its `VACUUM` left dead-but-not-removable or missed rows | [pgstat_relation.c#report_analyze-dead](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L329), [vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591) |
| `relpages` against `blocks_now` | equal | the table grew or shrank after the statement | [analyze.c#relpages](../../../raw/postgres-17/src/backend/commands/analyze.c#L190-L197) |
| `relallvisible` against `relpages` | equal, apart from all-zero pages | a page `VACUUM` could not set all-visible: a dead or not-yet-visible row, a dead item identifier a bypass left, or a page it could not lock for cleanup | [vacuumlazy.c#prune-sets-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1523-L1556), [vacuumlazy.c#heap-pass-sets-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2260-L2281), [vacuumlazy.c#new-page-not-in-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1290-L1321) |

The visibility-map check works because each path that finishes a page also sets its bit. Pruning sets it when every remaining row is visible to all and no `LP_DEAD` item is left, the second heap pass sets it after it frees the dead items, and an empty page is set all-visible and all-frozen ([vacuumlazy.c#prune-sets-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1523-L1556), [vacuumlazy.c#heap-pass-sets-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2260-L2281), [vacuumlazy.c#empty-page-sets-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1323-L1381)). The no-prune path sets nothing, and an all-zero page is deliberately left out of the map ([vacuumlazy.c#new-page-not-in-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1290-L1321)). A shortfall therefore points at a page that kept work, unless it is an all-zero page. A superuser can count those with [`pageinspect`](../../glossary.md#pageinspect), where an all-zero page reads `upper = 0` ([pageinspect--1.5.sql#page_header](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.5.sql#L22-L33), [rawpage.c#superuser](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L153)):

```sql
SELECT /* wiki_vacuum_zero_pages */ count(*) AS all_zero_pages
  FROM generate_series(0, pg_relation_size('public.t') / current_setting('block_size')::int - 1) AS b
 WHERE (page_header(get_raw_page('public.t', b::int))).upper = 0;
```

#### The index half on a B-tree

The heap lines above decide whether the dead entries were handed to the index. This table says what a [B-tree](../../glossary.md#b-tree) then does with them, which traces are normal, and which ones are not.

| What to read | Pass | What is by design | Evidence |
|---|---|---|---|
| that `btbulkdelete` ran | an `index "i"` line after every statement whose `index scan` label is `needed`, because `btbulkdelete` always returns a result | a statement with no dead item identifiers, or a bypassed one, reaches only `btvacuumcleanup`. That skips the index scan, and prints no line, unless the metapage's `btm_last_cleanup_num_delpages` exceeds 5 % of the index | [nbtree.c#btbulkdelete](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L820-L843), [nbtree.c#btvacuumcleanup-skip](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L855-L874), [nbtpage.c#_bt_vacuum_needs_cleanup](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L178-L223) |
| pins on leaf pages | nothing to check | `btvacuumpage` waits for a cleanup lock on every leaf page, so a pin delays index vacuuming in a `BufferPin` wait and never leaves an entry behind | [nbtree.c#cleanup-lock-every-leaf](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1202-L1208), [nbtpage.c#_bt_upgradelockbufcleanup](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1105-L1120) |
| `newly deleted` | above 0 when the fixture's precondition is emptied leaf pages; see [Rule 2 the uniform heap-block drain](#rule-2-the-uniform-heap-block-drain) | 0 when no leaf page was emptied, which the drain does not guarantee | [nbtpage.c#pages_newly_deleted](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2755-L2764) |
| `reusable` against `currently deleted` | equal after a later `VACUUM` that scans the index, once some XID has been consumed in between | on a quiet cluster the pages this statement deleted are not reusable yet. `_bt_pendingfsm_finalize` records such a page in the [free space map](../../glossary.md#free-space-map) only when its `safexid` is already behind the horizon, and its own comment says this never succeeds unless another backend consumes an XID meanwhile. Pages an earlier `VACUUM` deleted are recorded by `btvacuumpage` once `BTPageIsRecyclable` passes | [nbtpage.c#safexid](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2646), [nbtpage.c#pendingfsm-horizon](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3012-L3052), [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1179) |
| [`pgstatindex`](../../glossary.md#pgstatindex)'s `deleted_pages` | read as "deleted", never as "stuck" | it counts every deleted page, recyclable or not, and a deleted page stays deleted until a later page split takes it from the free space map | [pgstatindex.c#deleted-pages-count](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L307), [nbtpage.c#_bt_allocbuf-reuse](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L958) |

A pinned horizon therefore shows on a B-tree as a heap-side failure first - a nonzero `dead but not yet removable` count, and no `newly deleted` pages where the drain should have emptied leaves - and only later, if at all, as `reusable` staying below `currently deleted`.

#### Stage E: verdict, re-run and escalation

The escalation is tiered. Tier 0 is the autovacuum-equivalent statement of Stage B. A failure is first classified: a structural cause is repaired and tier 0 is run again, and a transient cause gets one more tier-0 run once its cause is removed. Only a transient failure that survives that re-run moves to tier 1, which changes one option and is declared.

| Failure seen | Cause class | What to do at tier 0 | Tier 1, only if the tier-0 re-run fails the same way |
|---|---|---|---|
| `D` above 0 on the `tuples:` line, or `K` above 0 with neither a `tuples missed:` nor an `index scan bypassed` line | structural: a horizon holder at node [2] | find it with the first gate query, end it, and run Stages A to D again | none. No option moves the cutoff; the holder has to go |
| `tuples missed:` | transient: a pin at node [3] | run `CHECKPOINT`, confirm the second gate query is still empty, and run the statement again. The missed pages are not all-visible, so the next `VACUUM` scans them again | `VACUUM (VERBOSE, ANALYZE, PARALLEL 0, FREEZE) public.t`; see [The FREEZE escalation](#the-freeze-escalation) |
| `index scan bypassed:` | transient by design: the bypass test at node [4] | record it. A re-run on the same pages makes the same choice, so a fixture that needs the entries gone goes straight to tier 1 | `VACUUM (VERBOSE, ANALYZE, PARALLEL 0, INDEX_CLEANUP ON) public.t`; see [The INDEX_CLEANUP ON escalation](#the-index_cleanup-on-escalation) |
| `bypassed by failsafe`, or the `bypassing nonessential maintenance` warning | structural: the table's freeze horizon is too old | the fixture is not usable; rebuild it. `INDEX_CLEANUP ON` does not help, because the failsafe can still bypass it ([vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407)) | none |
| a skip warning, or no `vacuuming` line | structural: privilege, relation kind or `SKIP_LOCKED` | fix the cause, then run Stages A to D again | none |
| `stopping truncate`, `suspending truncate`, or no `truncated` line when truncation was due | transient: some lock on the table at node [6], or growth | remove the lock holder, then run the statement again | none. The `TRUNCATE` option can prevent truncation but cannot force it past the size threshold or the lock ([vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407), [vacuumlazy.c#should_attempt_truncation](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2529-L2544)) |
| no `finished vacuuming` line, or an `ERROR` | structural: the statement was cancelled or failed | find what cancelled it, make sure the four timeouts are 0, and run again | none |
| no TOAST block although the table has a TOAST table | structural: `PROCESS_TOAST false` was given | run without it | none |
| `relallvisible` below `relpages` with every report check passing | none, when the gap equals the all-zero pages | count the all-zero pages; any page beyond them is a failure to explain | none |

`DISABLE_PAGE_SKIPPING` is not a remedy for a missed page. It makes the run aggressive and turns off visibility-map skipping ([vacuumlazy.c#aggressive-and-skipwithvm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L449-L467)), but the freeze cutoff stays `vacuum_freeze_min_age`, 50 million XIDs by default, behind the next XID ([vacuum.c#FreezeLimit](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1160-L1177), [guc_tables.c#vacuum_freeze_min_age](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2667-L2674)). On a young fixture no tuple is that old, so a pinned page still takes the no-prune path ([vacuumlazy.c#noprune-aggressive-wait](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1713-L1733)). Its only use here is the case the documentation names, a visibility map that is itself suspect.

##### The FREEZE escalation

- **Default.** A tier-0 statement is not aggressive on a young table, and a non-aggressive `VACUUM` never waits for a cleanup lock: the no-prune path always returns true for it ([vacuumlazy.c#noprune-contract](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1633-L1651), [vacuum.c#aggressive-decision](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1199-L1241)).
- **Cause.** The `FREEZE` option sets all four freeze ages to 0 for this statement ([vacuum.c#FREEZE-ages](../../../raw/postgres-17/src/backend/commands/vacuum.c#L385-L402)).
- **Mechanism.** A freeze table age of 0 makes the run aggressive, and a freeze minimum age of 0 puts `FreezeLimit` at the next XID, clamped to the removal cutoff ([vacuum.c#FreezeLimit](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1160-L1177), [vacuum.c#aggressive-decision](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1199-L1241)). A row is only dead when its deleting XID precedes the cutoff ([heapam_visibility.c#HeapTupleSatisfiesVacuum](../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1162-L1181)), and `heap_tuple_should_freeze` answers true for any tuple whose plain xmin or xmax precedes `FreezeLimit` ([heapam.c#heap_tuple_should_freeze](../../../raw/postgres-17/src/backend/access/heap/heapam.c#L8247-L8284)). So on a page that holds a dead row with a plain deleting XID, or any unfrozen row older than the cutoff, the no-prune path returns false and the scan waits for the cleanup lock before pruning ([vacuumlazy.c#noprune-aggressive-wait](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1713-L1733), [vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L950-L961)). A dead row whose deleter is a MultiXact is decided by the MultiXact cutoff and its members instead ([heapam.c#heap_tuple_should_freeze](../../../raw/postgres-17/src/backend/access/heap/heapam.c#L8247-L8284)).
- **What it costs.** The wait has no timeout of its own, so a pin that is never released stalls the statement in a `BufferPin` wait instead of skipping the page; Stage A's session check is what makes that wait short. `FREEZE` also freezes every eligible row, which an autovacuum worker at the defaults would not do, so the run declares that the maintained state is a frozen one.

##### The INDEX_CLEANUP ON escalation

- **Default.** `INDEX_CLEANUP` is `AUTO`, and under `AUTO` index vacuuming is bypassed when fewer than 2 % of the heap's pages hold dead items and their TIDs take under 32 MB ([vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407), [vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934)).
- **Cause.** `INDEX_CLEANUP ON` given in the command ([vacuum.c#INDEX_CLEANUP-option](../../../raw/postgres-17/src/backend/commands/vacuum.c#L233-L248)).
- **Mechanism.** The option sets `consider_bypass_optimization` to false before the scan, so `lazy_vacuum` never applies the bypass, and index vacuuming runs whenever dead item identifiers were collected; only the failsafe can still turn it off ([vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407), [vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934)).
- **What it costs.** The documentation calls `ON` the conservative choice that removes all dead tuples from indexes ([ref/vacuum.sgml#INDEX_CLEANUP](../../../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L186-L231)). An autovacuum worker takes `vacuum_index_cleanup` instead, `auto` by default ([autovacuum.c#unspecified-options](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2832-L2840), [vacuum.c#index_cleanup-reloption](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2179)), so the run declares the departure.

#### Behavior that looks like a skip but is by design

These leave a trace that resembles undone work. None of them is a failure of the procedure, and a run reports them rather than repairing them.

| Behavior | Why it is by design | Evidence |
|---|---|---|
| `pages: ... scanned` below 100 % | an all-visible page has nothing to remove, so a non-aggressive run skips runs of 32 or more of them | [vacuumlazy.c#skip-range](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1138-L1143), [vacuumlazy.c#find_next_unskippable_block](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1203-L1245) |
| no `new relfrozenxid` line | a non-aggressive run that skipped an all-visible page keeps the old horizon, because it did not see that page's XIDs; and the horizon never moves backward | [vacuumlazy.c#keep-relfrozenxid](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L544-L554), [vacuum.c#relfrozenxid-advance-only](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1489-L1519) |
| no `truncated` line on a heap whose end is empty | fewer than 1000 pages, and fewer than one sixteenth of the table, could be freed | [vacuumlazy.c#should_attempt_truncation](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2529-L2544) |
| `index scans: 2` or more | the dead-item store filled, so `VACUUM` emptied it with an extra round; nothing was skipped | [vacuumlazy.c#dead-items-full](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L872-L910) |
| fewer parallel workers than planned, in a run that drops `PARALLEL 0` | the leader processes the remaining indexes | [vacuumparallel.c#leader-participates](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L719-L726) |
| `VacuumDelay` waits | cost-based delay slows the run and skips nothing; its default is 0 for a command | [guc_tables.c#vacuum_cost_delay](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3865-L3873) |
| an index without a `VERBOSE` line | the access method returned no result for this run; see the index half above | [vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732) |
| an index left out entirely | `VACUUM` opens only indexes marked `indisready`, so one left unready by a failed concurrent build is not vacuumed | [vacuum.c#vac_open_indexes-indisready](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2345-L2357) |
| `datfrozenxid` unchanged | it moves only when every relation's horizon allows, and is skipped outright when a relation's horizon looks newer than the next XID | [vacuum.c#datfrozenxid-bogus](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1663-L1715) |

#### State changes that move the evidence

The proof reads stored values, so it has to read them after the right event and before the wrong one.

| Event | State before | State after | Effect on the proof |
|---|---|---|---|
| a churn session commits but has not flushed its counts | `n_dead_tup` lower than the truth | unchanged until the flush | a flush after the statement adds dead tuples that the statement already removed; Stage A item 6 prevents it ([pgstat.c#pgstat_report_stat](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L583-L600)) |
| `CHECKPOINT` | churned pages dirty | clean | background writes no longer pin the pages `VACUUM` is about to read ([bufmgr.c#SyncOneBuffer-pin](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3546-L3567)) |
| the `VACUUM` half ends | old counters | `vacuum_count` + 1, `n_dead_tup` = dead-but-not-removable plus missed, `n_ins_since_vacuum` = 0; `relpages`, `relallvisible` and the freeze horizons in `pg_class` | the completion check reads `vacuum_count` from here on ([pgstat_relation.c#pgstat_report_vacuum](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L227-L257), [vacuumlazy.c#relallvisible](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L556-L575)) |
| the `ANALYZE` half ends | `n_dead_tup` from `VACUUM` | `n_dead_tup` from the sample, `analyze_count` + 1, `n_mod_since_analyze` = 0; `relpages` and `relallvisible` rewritten | `n_dead_tup` now says what `ANALYZE` saw, including stubs a bypass left ([pgstat_relation.c#report_analyze-dead](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L329), [analyze.c#analyze-relstats](../../../raw/postgres-17/src/backend/commands/analyze.c#L623-L645)) |
| any statement that consumes an XID, then another `VACUUM` | deleted index pages not yet reusable | reusable, where the access method's cleanup scans the index | closes the recycling lag the index half describes |
| new writes to the table | pages all-visible | the touched pages lose their visibility-map bit | the proof applies to one statement; new churn needs the whole procedure again |

#### The two dead-row counters disagree on purpose

`VACUUM` and `ANALYZE` both publish a dead-row count, and they count different things. `VACUUM`'s `tuples:` line and its `n_dead_tup` write leave out the dead item identifiers a bypass or the failsafe kept: the code says so where it reports to the statistics system, and the bypass decision says so again ([vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591), [vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934)). `ANALYZE` counts every `LP_DEAD` stub as a dead row ([heapam_handler.c#analyze-dead-rows](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1054-L1085)). So after a bypass the `VACUUM` half reads clean and the `ANALYZE` half does not. A proof that read only the `VACUUM` half would pass a bypassed run; a proof that read only `ANALYZE` could not tell a bypass from a horizon holder. That is why Stage C reads both, together with the `index scan` label that names the bypass.

#### Summary of the causal chain

A quiet cluster - no horizon holder, no other session, clean buffers, no phase switched off, far from the failsafe, timeouts at 0 - runs one `VACUUM (VERBOSE, ANALYZE, PARALLEL 0)` per table. Its removal cutoff is then newer than every churn transaction, so every deleted row is dead rather than recently dead. With no other pins every scanned page takes the cleanup lock and is pruned, and the dead item identifiers go through index vacuuming and the second heap pass unless the bypass declines them. Index cleanup and, when due, truncation follow, and the statistics are written. The report then shows 0 dead-but-not-removable rows, no missed or bypassed line, no warning, a finished block for the table and its TOAST table, and an `ANALYZE` line with 0 dead rows. The counters move by one and `relallvisible` meets `relpages`. Any failed check names its node of the map and its tier: a holder is removed, a pin is retried and then waited out with `FREEZE`, and a bypass is overridden with `INDEX_CLEANUP ON`, both declared.

### Rule 1 splitting every recipe at the index build

Each numbered recipe is cut in two. Everything up to and including `CREATE INDEX` is the build phase; everything after it is the churn phase. A recipe written for a one-shot estimator ends in a single final state and asks one question about it, which leaves a before-and-after method with no baseline to compare against. Splitting at the build also fixes what the baseline means: a fresh serial build packs leaves to the index's fillfactor, `BTGetTargetPageFreeSpace` for a leaf level and `BTREE_NONLEAF_FILLFACTOR` above it ([nbtsort.c#_bt_pagestate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L660-L666), [nbtree.h#BTGetTargetPageFreeSpace](../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)), so the baseline is a known density rather than an arbitrary one.

### Rule 2 the uniform heap-block drain

Fixtures whose recipe has no churn of its own get one: delete every heap tuple outside one heap block in ten, then the maintenance `VACUUM ANALYZE`. The drain selects by block number out of the tuple's `ctid`, the self item pointer whose two fields are a block id and an offset ([itemptr.h#ItemPointerData](../../../raw/postgres-17/src/include/storage/itemptr.h#L36-L40), [sysattr.h#SelfItemPointerAttributeNumber](../../../raw/postgres-17/src/include/access/sysattr.h#L21)), so the surviving heap tuples are spread over the whole heap rather than clustered at one end. The `VACUUM` is what turns those dead entries into reclaimable index pages: `btbulkdelete` runs `btvacuumscan` ([nbtree.c#btbulkdelete](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L820-L843)), which visits every index block from `BTREE_METAPAGE + 1` to the current relation length and re-reads that length until it has caught up ([nbtree.c#btvacuumscan-block-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1016-L1041)), removing the entries whose TIDs the callback matches, and `_bt_pagedel` marks pages that end up empty as deleted ([nbtpage.c#_bt_pagedel](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1802-L1815)), which is the state `pgstatindex` reports as `deleted_pages` and half-dead pages as `empty_pages` ([pgstatindex.c#page-classes](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L295-L320)).

The drain's guarantee is about volume, not about distribution across the index, and the rule may not be stated the other way round. A serial build loads leaf pages from sorted input, so leaf order is key order ([nbtsort.c#sorted-build](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L4-L15), [nbtsort.c#_bt_load](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1130-L1136)), while the drain's predicate is on the heap block number. Whether the survivors leave entries in every leaf or in a contiguous run of leaves therefore follows from how the fixture's key correlates with physical row order, which each recipe sets and this rule does not.

| The drain does guarantee | The drain does not guarantee |
|---|---|
| about nine tenths of the table's live heap tuples are deleted, in a pattern spread over the heap | that any particular leaf page loses an entry |
| `btvacuumscan` visits every index block and removes every entry whose TID was drained | that entries are removed evenly across key space |
| pages emptied by that removal become `deleted_pages`, half-dead ones `empty_pages` | that any page is emptied at all, since a leaf keeps whatever survivors it holds |

A fixture whose point depends on the distribution - sparse leaves throughout rather than a run of empty pages at one end - must assert the post-drain shape it needs from the page classes and density its own measurement reports, and record a failed precondition when that shape did not arrive. Test 77 is the one fixture that asks for the opposite shape on purpose, by deleting a contiguous 95 % of its subset.

The drain is not applied to the fixtures whose whole point is that a fresh index must not be touched: tests 70 and 71, the whole of family 3, and controls 96, 97, 101 to 105, 108 to 112, 116 and 120. Draining those would destroy the test. They have no churn, so the maintenance assumption runs nothing on them either: the `ANALYZE` in their build phase is the maintenance they carry into the decide phase, and rule 3's census still runs over them.

### Rule 3 the simulated auto-analyze

Rule 3 is the launcher's analyze verdict, recomputed by the run for every table after the maintenance `VACUUM ANALYZE` has run. With [The maintenance assumption](#the-maintenance-assumption) in force the census finds every churned table freshly analyzed, so the tables it decides are the ones no churn touched: a table whose build left it past the threshold, such as the never-analysed table of 108, is analyzed here, and a table whose build-phase `ANALYZE` still describes it is left alone. The census also serves as the recheck that the maintenance step's counters were published in the right order.

Any table that has changed by more than a tenth of its heap tuples since its last `ANALYZE` must be analyzed before the decide phase, because a server with autovacuum on would have done so. The rule is not hand-annotated per fixture. It applies the engine's own test: `relation_needs_vacanalyze` sets `doanalyze` when the table's `mod_since_analyze` is **strictly greater** than `anl_base_thresh + anl_scale_factor * reltuples` ([autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3076), [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095)).

#### The effective per-table values, not the cluster settings

`anl_base_thresh` and `anl_scale_factor` are not the GUCs. Each is the table's own reloption whenever that reloption holds a non-negative value, and the cluster GUC only otherwise ([autovacuum.c#anl-effective-values](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3017)). The reloptions arrive as the `AutoVacOpts` member of the table's `StdRdOptions`, decoded from `pg_class.reloptions` by `extract_autovac_opts` ([autovacuum.c#extract_autovac_opts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2704-L2723), [rel.h#AutoVacOpts](../../../raw/postgres-17/src/include/utils/rel.h#L308-L326)), and an unset member reads back below zero, which is exactly what makes the `>= 0` test the precedence rule. A census that reads `current_setting('autovacuum_analyze_threshold')` for every table is therefore wrong on the fixtures that deliberately override it - 94 at 100 and 0, and 95 at 200000 and 1. Under the maintenance assumption both tables reach the census already analyzed, so the override no longer changes what the census does to them; the census must still compute the effective value, because the defect is in the arithmetic, not in the fixtures that exposed it.

| Input | Effective value the census must use | Evidence |
|---|---|---|
| base threshold | `relopts->analyze_threshold` when `>= 0`, else `autovacuum_anl_thresh` | [autovacuum.c#anl_base_thresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3015-L3017) |
| scale factor | `relopts->analyze_scale_factor` when `>= 0`, else `autovacuum_anl_scale` | [autovacuum.c#anl_scale_factor](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3013) |
| row count | `pg_class.reltuples`, taken as 0 when it is negative | [autovacuum.c#reltuples-clamp](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3065-L3072) |
| counter | `mod_since_analyze` from the table's shared statistics entry | [autovacuum.c#anltuples](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3063-L3068) |
| comparison | strictly greater than, so a fixture landing exactly on the threshold is left alone | [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3095) |
| short circuit | `autovacuum_enabled = false` makes `doanalyze` false outright, wraparound forcing aside | [autovacuum.c#av_enabled](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3027), [autovacuum.c#av_enabled-return](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3048-L3054) |

At the shipped defaults of 50 and 0.1 the effective threshold is 50 rows plus a tenth of the estimated row count ([guc_tables.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3367-L3375), [guc_tables.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3906-L3914)). Both GUCs are `PGC_SIGHUP`, so a cluster-wide change is a reload; setting either reloption is an `ALTER TABLE` that takes `ShareUpdateExclusiveLock` ([reloptions.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L243-L251), [reloptions.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L416-L425)).

#### Publication before maintenance and before the census

Running the phases in order does not put the churn's writes into `mod_since_analyze`. A backend accumulates its table counts locally, and they reach the shared entry only when `pgstat_report_stat` flushes them - which it does between transactions and, unforced, at most once per `PGSTAT_MIN_INTERVAL` of 1000 ms ([pgstat.c#PGSTAT_MIN_INTERVAL](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122), [pgstat.c#pgstat_report_stat](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L583-L600), [pgstat.c#flush-intervals](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665), [postgres.c#pgstat_report_stat-idle](../../../raw/postgres-17/src/backend/tcop/postgres.c#L4662-L4681)). Only such a flush increases the counter ([pgstat_relation.c#mod_since_analyze](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860)). Meanwhile `pgstat_report_analyze` zeroes the shared counter, and the code says in as many words that this forgets changes committed while the `ANALYZE` was running ([pgstat_relation.c#report_analyze-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L331-L337)).

Two orderings therefore go wrong on their own, and sequential phases prevent neither: unpublished churn reads as no change, so the census skips a table it should have analyzed; and churn published after the maintenance `ANALYZE` lands in a counter that was just reset, so the census's own recheck reads a freshly analyzed table as still dirty. The rule has two publication points, not one.

| When | What must hold | How to get it |
|---|---|---|
| after the churn, before the maintenance `VACUUM ANALYZE` and before the census reads the counter | every churn session's table counts are in the shared entry | `SELECT pg_stat_force_next_flush();`, which forces the flush at that transaction's end ([pg_proc.dat#pg_stat_force_next_flush](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5916-L5920), [pgstat.c#pgStatForceNextFlush](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L595-L600)); or let each churn session exit; or wait out `PGSTAT_MIN_INTERVAL` |
| after the maintenance `ANALYZE`, before any recheck | nothing published after the reset is read as new churn | publish first, analyze second, then re-read; a recheck that still disagrees is recorded as a disagreement, not analyzed again |

The read side is a third hazard. The census reads `n_mod_since_analyze` from `pg_stat_all_tables` ([system_views.sql#n_mod_since_analyze](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L689)), and `stats_fetch_consistency` - `PGC_USERSET`, so session or transaction scope, default `cache` ([guc_tables.c#stats_fetch_consistency](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4966-L4974), [pgstat.h#PgStat_FetchConsistency](../../../raw/postgres-17/src/include/pgstat.h#L68-L73)) - decides whether a second read of the same entry returns the first answer again. A census that reads, analyzes and re-reads inside one transaction must discard the cached statistics between reads with `pg_stat_clear_snapshot()` ([pg_proc.dat#pg_stat_clear_snapshot](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5911-L5915)). The engine's own statistics test uses exactly this force-flush-then-read sequence, under an explicit `stats_fetch_consistency = snapshot` ([stats.sql#force-flush](../../../raw/postgres-17/src/test/regress/sql/stats.sql#L99-L110)).

#### What the rule costs

The rule and the maintenance assumption together cost coverage, and the cost has to be named, not hidden: every churned table is analyzed by the maintenance `VACUUM ANALYZE`, and every unchurned table past the threshold is analyzed by this census, so a fixture whose point was a stale row count tests the method against fresh statistics instead. The stale-count shapes are preserved by the catalog forgeries below, which run after both.

### The oracle and the verdict bands

The oracle is a measured `REINDEX INDEX` on the churned fixture, and `actual_pct` is the fraction of the churned file the rebuild gave back. `REINDEX INDEX` reaches the blocking path through `ReindexIndex`'s final branch ([indexcmds.c#ReindexIndex-dispatch](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2838-L2849)), and `reindex_index` then suppresses the target index, assigns it a new relfilenode and calls `index_build` ([index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)). `index_build` refreshes both catalog rows through `index_update_stats`, once for the heap and once for the index ([index.c#index_build-update-stats](../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3138)), which is why the count written after a rebuild is the build's own tuple count and not a sample ([index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2842)).

That is not an exact count in every case, and the exception belongs to family 6. A rebuild of an **empty** index leaves the count unknown, not zero. `RelationSetNewRelfilenumber` writes `relpages = 0` and `reltuples = -1` and makes the change visible before the build runs ([relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3943-L3960), [relcache.c#relfilenumber-visible](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3967-L3973)); `index_update_stats` then turns a build count of zero back into `-1` whenever the existing `reltuples` is already negative, and that same test is what suppresses the `relpages` and `relallvisible` update ([index.c#index_update_stats-empty-hack](../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842)). A fresh `CREATE INDEX` on an empty table lands in the same state, because a newly created relation's row starts at `relpages = 0`, `reltuples = -1` ([heap.c#AddNewRelationTuple-reltuples](../../../raw/postgres-17/src/backend/catalog/heap.c#L1004-L1016)). So a fixture that must show an empty population has to read that population from somewhere other than `pg_class.reltuples`, and a run may not report `-1` as a measured zero.

Four bands score each fixture, and the two false-positive bands are separate because their costs differ by an order of magnitude:

| Verdict | Rule |
|---|---|
| `CRITICAL FALSE POSITIVE` | rebuilt, and the rebuild gave back less than 10 % |
| `FALSE POSITIVE` | rebuilt, and the rebuild gave back less than 35 % |
| `FALSE NEGATIVE` | not rebuilt, and a rebuild would have given back 50 % or more |
| `PASS` | everything else |

Two further columns are mandatory for a run to be readable: `expected_stage`, the gate arithmetic recomputed independently from the recorded baseline, so a disagreement between the method and the arithmetic is visible; and `want_stage`, a per-fixture prediction filed before the run, which may not be rewritten afterwards.

### Family 1 the deduplication gate

Tests 1 to 17, less the retired 11 and its 11b transition control, built on two 500,000-row tables whose deduplication key columns each carry 5,000 distinct values, 100 rows per key, beside one strictly increasing column for the unique control. The family asks one question across the remaining fixtures: can this index deduplicate, and does the method under test know it.

`_bt_allequalimage` logs its verdict at `DEBUG1` ([nbtutils.c#_bt_allequalimage-debug](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180)), which makes the engine its own oracle for **most** of this family. It is not a complete oracle, and two conditions bound it:

- The `INCLUDE` refusal returns before the logging block is reached ([nbtutils.c#INCLUDE-refusal](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5147)), so test 10 produces no line at all. Reading its silence as a verdict is reading nothing.
- The message is emitted only when the caller asks for it. On the build path that caller is `_bt_leafbuild`'s sorted build, which passes `true` ([nbtsort.c#_bt_leafbuild-allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L559-L563)); `btbuildempty`, which initializes the init fork of an unlogged index, passes `false` ([nbtree.c#btbuildempty-allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L158-L167)). Those are the only two callers in the tree.

A fixture whose verdict must be known but whose index produces no `DEBUG1` line has to read `btm_allequalimage` back out of the metapage instead ([nbtpage.c#_bt_metaversion](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L739-L750)).

| Test | Fixture index | What it builds | What it must prove |
|---|---|---|---|
| 1, 2 | `i_int4`, `i_int8` | integer keys, 100 rows per key | the integer opclasses register `btequalimage`, which returns true unconditionally ([datum.c#btequalimage](../../../raw/postgres-17/src/backend/utils/adt/datum.c#L432-L438)) |
| 3 | `i_text_det`, `i_text_icu_det` | `text` key under the default collation, and under a deterministic ICU collation | `btvarstrequalimage` returns true for C, the default collation, or any deterministic collation ([varlena.c#btvarstrequalimage](../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)) |
| 4 | `i_text_nondet` | `text` key under a nondeterministic ICU collation | the same function returns false, so the index is not equal-image; `deterministic = false` is stored in `pg_collation` ([pg_collation.h#collisdeterministic](../../../raw/postgres-17/src/include/catalog/pg_collation.h#L40), [collationcmds.c#deterministic](../../../raw/postgres-17/src/backend/commands/collationcmds.c#L200-L207)) |
| 5, 6 | `i_numeric`, `i_float4`, `i_float8` | `numeric` and float keys | an opclass with no `BTEQUALIMAGE_PROC` is assumed unsafe ([nbtutils.c#allequalimage-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5149-L5170), [nbtree.h#BTEQUALIMAGE_PROC](../../../raw/postgres-17/src/include/access/nbtree.h#L706-L712)) |
| 7 | `i_multi_ok`, `i2_ok` | two equal-image key columns | every key column is tested, not only the first |
| 8 | `i_multi_bad` | one equal-image and one non-equal-image column | one failing column breaks the loop and condemns the index |
| 9 | `i_expr_num`, `i_expr_lower_ci` | expression keys, numeric and a `lower()` under a nondeterministic collation | the expression's result type and collation decide, not the base column's |
| 10 | `i_inc` | an `INCLUDE` column | `INCLUDE` indexes are refused before any opclass lookup ([nbtutils.c#INCLUDE-refusal](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5143-L5147), [pg_index.h#indnkeyatts](../../../raw/postgres-17/src/include/catalog/pg_index.h#L33-L36)) |
| 12 | `i_ei_none` | a custom operator class declaring no `FUNCTION 4` | a missing procedure is unsafe, distinct from one that answers false |
| 13 | `i_ei_false` | a custom `FUNCTION 4` returning false | the callback is really called, not merely looked up |
| 14 | `i_ei_true`, `i_ei_alias` | a custom `FUNCTION 4` returning true, and a `LANGUAGE internal` alias of `btequalimage` | an internal function is resolved by `prosrc`, so an alias under another name still resolves to the builtin ([fmgr.c#internal-alias](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240)) |
| 15 | `i_mixed_tf`, `i_mixed_ft`, `i2_tf`, `i2_ft` | true/false and false/true callbacks in both column orders | the verdict does not depend on which column fails |
| 16 | `i_squat`, `i_text_det2` | a SQL impostor named `btequalimage`, and a renamed internal alias of `btvarstrequalimage` | the name is not the identity; `prolang` and `prosrc` are |
| 17 | `i_uniq` | a unique index over equal-image keys | a serial build never deduplicates a unique index ([nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)) |

What the family does *not* pin is the page-level merge itself: `_bt_dedup_pass` runs on insert to avoid a page split, so a deduplicating index's density is a function of its write history as well as its keys ([nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L56-L70)).

### Family 2 partial indexes

Tests 18 to 77, less the retired 38, 65, 67 and 69, the largest family. Every fixture is a partial index whose subset differs from its table in one named way, which is exactly the axis on which a catalog-only method has to guess. The predicate lives in `pg_index.indpred` ([pg_index.h#indpred](../../../raw/postgres-17/src/include/catalog/pg_index.h#L58-L62)), and `ANALYZE` refines the index's own row estimate only for an index that has statistics columns or a predicate ([analyze.c#partial-index-skip](../../../raw/postgres-17/src/backend/commands/analyze.c#L859-L863), [analyze.c#tupleFract-default](../../../raw/postgres-17/src/backend/commands/analyze.c#L445-L450)).

| Tests | Fixtures | What they build | What they must prove |
|---|---|---|---|
| 18-21 | `p18`-`p21` | one 1,000,000-row table, four predicates selecting roughly 20 %, 1 %, 10 % and 80 % | selectivity alone moves the subset's population from about a hundredth of the table to four fifths of it while the table's `reltuples` never changes |
| 22-26 | `p22`-`p26` | duplication, uniqueness, `n_distinct` and most-common-value distributions that differ inside the subset | per-column statistics describe the table, not the subset, so key-modelling assumptions break here |
| 27-29 | `p27`-`p29` | NULL-heavy subset, NULL-free subset of a NULL-heavy table, and an all-NULL partial index under `WHERE s IS NULL` | NULLs are indexed entries in a B-tree, and the table's null fraction does not describe the subset's |
| 30-33 | `p30`-`p33` | subset values wider than the rest, narrower than the rest, extreme mismatch, and variable widths with the same range inside and out | the average width `ANALYZE` records is the table's, so entry size is a guess |
| 34-37 | `p34`-`p37` | a deduplication-heavy subset, a unique subset of a duplicate-heavy table, one key group of 100,000 TIDs, and an all-NULL key subset | posting lists change the file size for an unchanged entry count ([nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L56-L70)) |
| 39 | `p39` | a partial `UNIQUE` index | uniqueness suppresses build-time deduplication even where the keys are equal-image |
| 40-45 | `p40`-`p45` | two-column keys correlated only inside the subset, independent only inside the subset, duplicated, unique, with and without a `CREATE STATISTICS (ndistinct)` object, and one whose extended statistics are wrong for the subset | extended statistics are collected for the table, and an `ndistinct` object does not know about the predicate ([mvdistinct.c#statext_ndistinct_build](../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L86-L100), [pg_statistic_ext.h#STATS_EXT_NDISTINCT](../../../raw/postgres-17/src/include/catalog/pg_statistic_ext.h#L84)) |
| 46-47 | `p46`, `p47` | partial indexes with `INCLUDE` columns, one with wide payloads inside the subset | non-key columns occupy leaf space and are counted by no key statistic |
| 48-50 | `p48`-`p50b` | partial expression indexes, each in two orders: `ANALYZE` before the build, and `ANALYZE` after it | an expression index has no statistics row until an `ANALYZE` runs with the index in place |
| 51-52 | `p51`, `p52` | the same subset under a deterministic and a nondeterministic ICU collation | the collation decides deduplication safety for a partial index exactly as it does in family 1 |
| 53-55 | `p53`-`p55` | the same subset at fillfactor 90, 100 and 70 | target leaf free space is `BLCKSZ * (100 - fillfactor) / 100`, so the as-built density is a parameter ([nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-17/src/include/access/nbtree.h#L197-L202)) |
| 56-63 | `p56`-`p63` | boolean, equality, range, `IS NULL`, `IS NOT NULL`, multi-column, and correlated and anti-correlated predicates | predicate shape changes which rows enter the index, and correlation changes where they land in key order |
| 64, 66, 68 | `p64`, `p66`, `p68` | inserts into the subset, rows entering the predicate, and heavy predicate churn followed by `VACUUM` and `ANALYZE` | an insert, a predicate transition and a vacuumed churn move the subset's population three different ways, and only the insert moves the table's row count with it |
| 70-71 | `p70`, `p71` | a freshly created partial index, and one freshly rebuilt behind the method's back | neither may be touched; 71 also checks that a rebuild nobody recorded does not become a false positive |
| 72-75 | `p72`-`p75` | a quarter, a half, three quarters and nine tenths of the subset deleted, each with `VACUUM` and `ANALYZE` | this is the calibration ladder: four known reclaimable fractions with a measured rebuild behind each |
| 76 | `p76` | rows updated so the indexed key changes while the predicate still holds | the file grows while the entry count barely moves, which no row-count test can see |
| 77 | `p77` | a contiguous 95 % of the subset deleted, then `VACUUM` | produces empty and deleted pages rather than sparse leaves ([nbtpage.c#_bt_pagedel](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1802-L1815)) |

### Family 3 false-positive constructions

Tests 78 to 85: eight freshly built indexes whose statistics invite a rebuild and whose rebuild would return nothing. None of them is churned, and none of them may be rebuilt. Five differ from their table in a way conditioned on the predicate, one has no statistics row for its expressions, one forges a catalog count, and one carries table statistics that predate a widening `UPDATE`.

| Test | Fixture | The trap |
|---|---|---|
| 78 | `f78` | subset values far wider than the table average |
| 79 | `f79` | the subset is the only non-NULL part of the column |
| 80 | `f80` | the subset is unique while the table is duplicate-heavy |
| 81 | `f81` | the table's most common value does not occur in the subset |
| 82 | `f82` | two key columns correlated in the table and not in the subset, with an `ndistinct` object present |
| 83 | `f83` | a two-expression index with no statistics row for either expression |
| 84 | `f84` | the index's own `reltuples` is forged low, so a method reading it sees a shrunken population |
| 85 | `f85` | the table's statistics predate an `UPDATE` that widened every subset row |

### Family 4 false-negative constructions

Tests 86 to 91: six indexes that really are reclaimable, each `VACUUM`ed and `ANALYZE`d after the churn so that nothing about them is stale. All six must be rebuilt. They exist because the cheapest way to score zero false positives is to rebuild nothing, and this family makes that strategy fail.

| Test | Fixture | Why a model under-reads it |
|---|---|---|
| 86 | `f86` | duplicates concentrated inside the subset, so the real entry count per key is far above the table's |
| 87 | `f87` | NULLs concentrated inside the subset |
| 88 | `f88` | the subset is much narrower than the table's average width |
| 89 | `f89` | two key columns perfectly correlated only inside the subset |
| 90 | `f90` | real deduplication compresses more than a per-key model predicts |
| 91 | `f91` | many deleted pages *and* an over-predicting width model in the same fixture |

### Family 5 the change A to D controls

Tests 92 to 112, less the retired 106, the controls that pin thresholds and isolate one input at a time.

| Tests | Fixtures | What they hold fixed |
|---|---|---|
| 92-95 | `b92`-`b95` | a reclaimable partial index plus a known number of row changes, twice at cluster defaults and twice with per-table `autovacuum_analyze_threshold` and `autovacuum_analyze_scale_factor` reloptions set to 100/0 and 200000/1 ([reloptions.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L243-L251)); under the maintenance assumption all four are analyzed after their changes, so the reloption pair no longer changes the census's decision, see [Open Questions](#open-questions) |
| 96-99 | `np96`-`np99` | non-partial controls: a plain index with fresh statistics, an expression index with no statistics row, a plain index after 300,000 inserts, maintained by the assumption's `VACUUM ANALYZE` once they have landed, and a duplicate-heavy plain index that is genuinely reclaimable |
| 100-105 | `i100`-`i105` | width controls: partial and non-partial `INCLUDE` payloads, payloads wider inside the subset, narrower inside the subset, mixed non-key widths, and one partial index whose *key* column is wide and unique inside the subset |
| 107-112 | `x107`-`x112` | expression-statistics controls: an expression index with one `ANALYZE` after the build, a never-analysed table, a key column at `SET STATISTICS 0` ([analyze.c#attstattarget-zero](../../../raw/postgres-17/src/backend/commands/analyze.c#L1024-L1032), [pg_attribute.h#attstattarget](../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L166-L177)), a mixed column-and-expression key, a narrow expression, and a partial expression index |

### Family 6 the drained queue and zero row counts

Tests 113 to 120, less the retired 117: the shapes where a row count is zero, unknown, or stale. Each is one table of a million rows, 115 reaching that size only during the churn, and 113 keeps one leg, `p113b`.

This family must keep an empty population and an unknown catalog count apart, because for an index that ended up with no entries they are not the same state and `pg_class.reltuples` shows only the second. Both the build and the oracle rebuild leave such an index at `reltuples = -1` with `relpages` untouched, not at `0`; see [The oracle and the verdict bands](#the-oracle-and-the-verdict-bands). The fixtures that reach it are 113b, 115 before its load, 116, and 118 before its arrivals. A fixture in this family asserts its population from the index itself, and a run that prints `-1` as a zero row count has recorded a different fixture from the one the suite defines.

| Test | Fixture | State it reaches |
|---|---|---|
| 113b | `p113b` | a queue drained to empty, then `VACUUM` and `ANALYZE` |
| 114 | `p114` | drained to 1 %, then `VACUUM` and `ANALYZE` |
| 115 | `p115` | the index built on an analysed empty table, then a million rows loaded |
| 116 | `p116` | the subset empty from the start and measured empty |
| 118 | `p118` | the subset measured empty, then 50,000 rows arrive; the maintenance `VACUUM ANALYZE` follows them, so the index's own count is re-estimated before the decide phase |
| 119 | `p119` | fixture 118 after one more `ANALYZE`; under the maintenance assumption the two differ only by a second sample, see [Open Questions](#open-questions) |
| 120 | `p120` | a 2,000-row subset the sample is *meant* to miss at `default_statistics_target = 1`, which sets the sample to `300 * attstattarget` rows ([analyze.c#std_typanalyze-minrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L1888-L1895), [guc_tables.c#default_statistics_target](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2069-L2078)); the miss is asserted after the census, never assumed |

Test 120 is the suite's one probabilistic fixture, so it carries an assertion the others do not need. `ANALYZE` does not sample deterministically: `acquire_sample_rows` seeds its block sampler from the global PRNG and then keeps rows by reservoir sampling ([analyze.c#acquire_sample_rows-seed](../../../raw/postgres-17/src/backend/commands/analyze.c#L1185-L1194), [analyze.c#reservoir-replacement](../../../raw/postgres-17/src/backend/commands/analyze.c#L1207-L1229)). A 300-row sample of a million-row table usually misses a 2,000-row subset, and sometimes finds one of its rows. The fixture's precondition is the resulting state, not the recipe that aims at it: after the census the partial index's own `reltuples` must read 0, which is what `ceil(tupleFract * totalrows)` writes when no sampled row passed the predicate ([analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953), [analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660)). A run that finds a non-zero estimate records an unmet fixture precondition for 120 and scores nothing from it; it may not credit the fixture as covered.

### Catalog forgeries run last

Fixture 84's forged index `reltuples`, and any forged `-1` counts or forged `indisvalid`, are applied *after* the maintenance `VACUUM ANALYZE` and the rule 3 census, not during the churn. An `ANALYZE` of the table rewrites `reltuples` for the table and for every index on it ([analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660)), so a forgery written before either is silently repaired and the fixture stops testing anything. Ordering the forgeries last is what keeps the stale-count shapes alive once rule 3 is in force.

### Feature gates that skip a fixture

A fixture that needs a feature the server under test does not have is recorded as skipped, never silently dropped and never rewritten. Two gates matter for a 12-through-17 claim: B-tree support function 4 and the `btequalimage`/`btvarstrequalimage` builtins, which tests 13 to 16 need; and ICU collations, which tests 3, 4, 9, 51 and 52 need and which a build configured without ICU cannot create. A run reports how many fixtures were skipped and why, beside the fixtures it scored.

### What the suite does not cover

Named limits, so a consumer page does not claim more than the suite gives:

- **Explicit deduplication settings are outside the suite.** Tests 11, 11b and 38 are retired, and their numbers are not reused. No mandatory fixture explicitly enables or disables `deduplicate_items`, at build time or afterward. The remaining eligibility fixtures keep their types, collations, operator classes, uniqueness and `INCLUDE` definitions.

- **No fixture builds a dead-but-not-vacuumed index.** Entries that leave an index with no following `VACUUM` stay physically present, so leaf pages stay dense while a rebuild would empty them. The maintenance assumption rules that shape out: every churn is followed by `VACUUM ANALYZE`, the fixtures that withheld the `VACUUM` are retired, and the suite neither creates the state nor scores a method against it. The state can still arrive under a `VACUUM` the suite did run, because `VACUUM` itself can decline to remove the entries, and "fewer than 2 % of heap pages carry dead items" is one of four conditions rather than the test: the bypass applies only when it is still under consideration, when the relation has at least one page, when `lpdead_item_pages` is *strictly* below `rel_pages * BYPASS_THRESHOLD_PAGES`, and when the dead-item store is under 32 MB ([vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934)). `consider_bypass_optimization` starts true, is forced false by `INDEX_CLEANUP = ON`, and is cleared as soon as this VACUUM has already run one round of index vacuuming ([vacuumlazy.c#consider_bypass_optimization-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L407), [vacuumlazy.c#consider_bypass_optimization-cleared](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L895-L897)). When it does apply it turns off index *vacuuming* only; index cleanup still runs ([vacuumlazy.c#bypass-applies](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1957)), which is how `btvacuumcleanup` gets to mark the count estimated ([nbtree.c#btvacuumcleanup-estimated_count](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L884-L894)). A fixture may not claim a bypass from the page fraction alone. The bypass is one of the ways the state survives a `VACUUM` the suite did run. The others are a pinned removal horizon, which [The maintenance must not be defeated](#the-maintenance-must-not-be-defeated) forbids a fixture from creating; a page `VACUUM` could not lock for cleanup, whose dead rows it counts as missed and leaves in place with their index entries ([vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L923-L961), [vacuumlazy.c#missed-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1762-L1769)); an `INDEX_CLEANUP` switched off by option or storage parameter ([vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407)); and the wraparound failsafe ([vacuumlazy.c#lazy_check_wraparound_failsafe](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2299-L2347)). [Recommended: proving VACUUM did all its work](#recommended-proving-vacuum-did-all-its-work) shows how a run tells them apart.
- **Withheld maintenance is not a fixture shape.** Numbers 65, 67, 69, 106, 117 and 121, and legs 113a and 113c, are retired: they were the fixtures that deliberately skipped a `VACUUM`, or skipped the `ANALYZE` after one, and the maintenance assumption now forbids the shape for any fixture added later. The numbers are not reused, so a run that reports one is reporting a fixture this page does not define. The boundary is the skipped maintenance command, not the misleading statistic: fixtures whose point is a missing or stale statistics row - 64, 83, 85, 97, 98, 108, 110 to 112 and 118 - stay in the suite. Those among them that are churned (64, 98 and 118) reach the decide phase analyzed by the maintenance step, 85 and 108 by rule 3's census, and 83, 97 and 110 to 112 keep the row they lack, since nothing churns them and the census has no reason to analyze them.
- **One block size and one platform.** Every geometry constant derives from `BLCKSZ`; the suite fixes no second block size.
- **No concurrency.** Every phase runs alone. Two sessions racing on one index, or a rebuild against a concurrent `DROP INDEX`, is not a fixture.
- **No partitioned table end to end.** Leaf indexes can be scored, but no fixture drains a partition and then asks about the parent.
- **The engine's own regression suites validate the build, not the method.** `make check` and the contrib checks exercise PostgreSQL; nothing in them reads a bloat estimate ([regress.sgml#make-check](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59), [regress.sgml#contrib-suites](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195)).

## Where It Appears in Source

The suite has no code in the PostgreSQL tree. What it has in the tree is the behavior each family targets, and these are the files a reviewer reads to judge a fixture:

| Area | Files | What the suite takes from it |
|---|---|---|
| Deduplication safety | `src/backend/access/nbtree/nbtutils.c`, `src/include/access/nbtree.h` | `_bt_allequalimage`, the `INCLUDE` refusal, `BTEQUALIMAGE_PROC`, the `DEBUG1` oracle ([nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)) |
| Equal-image callbacks | `src/backend/utils/adt/datum.c`, `src/backend/utils/adt/varlena.c` | `btequalimage`, `btvarstrequalimage` and its collation test ([varlena.c#btvarstrequalimage](../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)) |
| Build geometry | `src/backend/access/nbtree/nbtsort.c` | the fillfactor targets and the build-time deduplication gate ([nbtsort.c#_bt_pagestate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L660-L666)) |
| Page reclamation | `src/backend/access/nbtree/nbtpage.c`, `src/backend/access/nbtree/nbtree.c` | `_bt_pagedel`, `btbulkdelete`, `btvacuumscan`'s every-block loop and the cleanup-only estimate ([nbtree.c#btvacuumscan-block-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1016-L1041)) |
| Row counts | `src/backend/catalog/index.c`, `src/backend/commands/analyze.c`, `src/backend/commands/vacuum.c`, `src/backend/access/heap/vacuumlazy.c`, `src/backend/utils/cache/relcache.c` | who writes `reltuples`, when it is exact, when it is sampled, and when it becomes `-1` |
| Autovacuum | `src/backend/postmaster/autovacuum.c`, `src/backend/utils/misc/guc_tables.c`, `src/backend/access/common/reloptions.c` | the two verdicts the maintenance assumption stands in for, the analyze threshold rule 3 applies, and their GUC and reloption forms |
| Maintenance commands | `src/backend/commands/vacuum.c` | the `vacuum()` routine both an autovacuum worker and the SQL command call, and its vacuum-then-analyze order |
| The removal horizon and the lock wait | `src/backend/storage/ipc/procarray.c`, `src/backend/access/transam/twophase.c`, `src/backend/storage/lmgr/proc.c`, `src/backend/commands/vacuum.c` | the horizon a `VACUUM` is given, the backend `xmin`s, slot `xmin`s and prepared transactions that pin it, the skip a worker takes when it cannot lock a table, and the cancel a blocked backend sends one ([procarray.c#ComputeXidHorizons-backend-xmin](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1792-L1815)) |
| Completeness signals | `src/backend/access/heap/vacuumlazy.c`, `src/backend/commands/vacuum.c`, `src/backend/commands/analyze.c`, `src/backend/access/heap/heapam_handler.c`, `src/backend/storage/buffer/bufmgr.c` | the `VERBOSE` report lines, the cleanup-lock fallback and its missed tuples, the bypass, truncation and failsafe switches, the TOAST recursion and `ANALYZE`'s dead-row count that [Recommended: proving VACUUM did all its work](#recommended-proving-vacuum-did-all-its-work) reads ([vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L923-L961), [vacuumlazy.c#verbose-missed-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668)) |
| Statistics reporting | `src/backend/catalog/system_views.sql`, `src/backend/utils/activity/pgstat_relation.c` | `n_mod_since_analyze` and its reset, and the `vacuum_count` and `n_dead_tup` a `VACUUM` writes last |
| Measurement | `contrib/pgstattuple/pgstatindex.c` | the page classes and density a physical reading reports ([pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L252)) |
| Write path | `src/backend/executor/execIndexing.c` | the partial-index predicate skip that makes a subset a subset |

### Callers and callees the fixtures depend on

A citation that lands on a function's signature and locals resolves the name; it does not support a claim about what the function does. These are the caller-to-callee boundaries a fixture's behavior actually crosses:

| Fixture behavior | Boundary |
|---|---|
| the `DEBUG1` deduplication verdict | `btbuild` -> `_bt_leafbuild` -> `_bt_allequalimage(index, true)`, whose answer `_bt_load` then reads out of `inskey->allequalimage` ([nbtsort.c#_bt_leafbuild-allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L559-L563), [nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)) |
| the silent verdict | `btbuildempty` -> `_bt_allequalimage(index, false)` -> `_bt_initmetapage`, no message at any level ([nbtree.c#btbuildempty-allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L158-L167)) |
| insert-time deduplication | `_bt_findinsertloc` -> `_bt_dedup_pass`, behind `BTGetDeduplicateItems` and the scan key's `allequalimage` ([nbtinsert.c#dedup-on-insert](../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2774-L2782)) |
| the oracle rebuild | `ReindexIndex` -> `reindex_index` -> `RelationSetNewRelfilenumber` then `index_build` -> `index_update_stats` twice ([indexcmds.c#ReindexIndex-dispatch](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2838-L2849), [index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789), [index.c#index_build-update-stats](../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3138)) |
| the drain's index work | `lazy_vacuum` -> `lazy_vacuum_all_indexes` -> `btbulkdelete` -> `btvacuumscan` -> `btvacuumpage` -> `_bt_pagedel` ([vacuumlazy.c#bypass-applies](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1957), [nbtree.c#btbulkdelete](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L820-L843), [nbtree.c#btvacuumscan-block-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1016-L1041)) |
| the maintenance step | `ExecVacuum` -> `vacuum()` -> `vacuum_rel` then `analyze_rel` for each relation, the same `vacuum()` an autovacuum worker enters from `autovacuum_do_vac_analyze` ([vacuum.c#ExecVacuum-vacuum](../../../raw/postgres-17/src/backend/commands/vacuum.c#L450-L451), [vacuum.c#vacuum-then-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650), [autovacuum.c#autovacuum_do_vac_analyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3120-L3147)) |
| the maintenance step's effectiveness | `vacuum_rel` -> `vacuum_open_relation`, which for a worker takes the lock conditionally, then `lazy_vacuum_rel` -> `vacuum_get_cutoffs` -> `GetOldestNonRemovableTransactionId` -> `ComputeXidHorizons`, and `lazy_scan_heap`'s dead-TID gate into `lazy_vacuum`, which is where a pinned horizon stops the drain's index work ([vacuum.c#vacuum_open_relation-conditional](../../../raw/postgres-17/src/backend/commands/vacuum.c#L785-L800), [vacuum.c#vacuum_get_cutoffs-OldestXmin](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1109-L1122), [procarray.c#GetOldestNonRemovableTransactionId](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2005-L2026), [vacuumlazy.c#lazy_vacuum-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1052)) |
| the completeness report | `heap_vacuum_rel` -> `lazy_scan_heap`, which takes each page's cleanup lock with `ConditionalLockBufferForCleanup` or falls back to `lazy_scan_noprune` -> `lazy_vacuum`, which applies the bypass or runs `lazy_vacuum_all_indexes` and `lazy_vacuum_heap_rel` -> `lazy_cleanup_all_indexes` -> `should_attempt_truncation` and `lazy_truncate_heap` -> `vac_update_relstats` -> `pgstat_report_vacuum` -> the `VERBOSE` report; `vacuum_rel` then recurses into the TOAST table, and `vacuum()` calls `analyze_rel` ([vacuumlazy.c#heap_vacuum_rel-flow](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L497-L591), [vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L923-L961), [vacuumlazy.c#bypass-applies](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1957), [vacuumlazy.c#report-ereport](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L762-L763), [vacuum.c#toast-recursion](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2282-L2300), [vacuum.c#vacuum-then-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650)) |
| rule 3's threshold | `do_autovacuum` -> `extract_autovac_opts` plus `pgstat_fetch_stat_tabentry_ext` -> `relation_needs_vacanalyze` ([autovacuum.c#do_autovacuum-census](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2021-L2033)) |
| rule 3's counter | backend DML -> `pgstat_report_stat` between transactions -> the relation flush callback -> shared `mod_since_analyze` ([postgres.c#pgstat_report_stat-idle](../../../raw/postgres-17/src/backend/tcop/postgres.c#L4662-L4681), [pgstat_relation.c#mod_since_analyze](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860)) |
| the partial-index estimate | `do_analyze_rel` -> `acquire_sample_rows` -> `compute_index_stats`'s `tupleFract` -> `vac_update_relstats` ([analyze.c#acquire_sample_rows-seed](../../../raw/postgres-17/src/backend/commands/analyze.c#L1185-L1194), [analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953), [analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660)) |

### Generated files the fixtures depend on

Two build-time artifacts decide fixtures that look like pure SQL, so a checkout whose generated headers are stale builds a server the suite's verdicts do not describe.

- **The builtin function table, for tests 14 and 16.** A `LANGUAGE internal` function created under any name resolves through `fmgr_lookupByName`, a name scan of `fmgr_builtins` ([fmgr.c#fmgr_lookupByName](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L95-L111), [fmgr.c#internal-alias](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L243)). That table is not in the source tree: `Gen_fmgrtab.pl` generates `fmgrtab.c`, `fmgroids.h` and `fmgrprotos.h` from `pg_proc.dat`, and the headers are symlinked into `src/include/utils` ([Makefile#fmgr-stamp](../../../raw/postgres-17/src/backend/utils/Makefile#L46-L53), [Makefile#header-stamp](../../../raw/postgres-17/src/backend/utils/Makefile#L76-L83)). This is why an SQL impostor named `btequalimage` cannot become the builtin and a renamed internal alias of it still is: both answers come out of a table built from `pg_proc.dat`, not out of the function's name.
- **The catalog headers, for every column the suite reads.** `pg_class.reltuples`, `pg_index.indpred`, `pg_index.indnkeyatts`, `pg_collation.collisdeterministic`, `pg_attribute.attstattarget` and the `pg_amproc` rows a custom `FUNCTION 4` adds are all declared in catalog headers whose `_d.h` companions, `schemapg.h` and BKI data `genbki.pl` produces under a `generated-headers` target ([Makefile#catalog-generated-headers](../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143), [Makefile#backend-generated-headers](../../../raw/postgres-17/src/backend/Makefile#L116-L140)).

### Shipped tests that cover the same behavior

The suite is not in the tree, but part of the behavior it leans on is covered by tests that are. Naming the absences matters as much as naming the coverage, because an absent test is why a fixture exists.

| Behavior | Shipped coverage |
|---|---|
| a deduplicating index, and an `INCLUDE` unique index, checked structurally | [check_btree.sql#deduplicate_items](../../../raw/postgres-17/contrib/amcheck/sql/check_btree.sql#L16-L19) |
| reading the metapage fields a fixture falls back on | [btree.sql#bt_metap](../../../raw/postgres-17/contrib/pageinspect/sql/btree.sql#L5-L8) |
| `INCLUDE` indexes, key versus non-key columns | [index_including.sql#CREATE-INDEX-INCLUDE](../../../raw/postgres-17/src/test/regress/sql/index_including.sql#L8-L27) |
| `pgstatindex` output on an empty index, and its wrong-AM and unsupported-relation refusals | [pgstattuple.sql#pgstatindex](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37), [pgstattuple.sql#refusals](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L55-L120) |
| forcing a statistics flush before reading a counter, under an explicit fetch consistency | [stats.sql#force-flush](../../../raw/postgres-17/src/test/regress/sql/stats.sql#L99-L110) |
| `reltuples` after `ANALYZE` on a shape with no live rows of its own | [vacuum.sql#reltuples](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L70-L97) |
| a non-aggressive `VACUUM` that cannot cleanup-lock a page a cursor has pinned, run with freeze cutoffs at their most aggressive, and what that does to `reltuples` and to MultiXact freezing | [vacuum-no-cleanup-lock.spec#header](../../../raw/postgres-17/src/test/isolation/specs/vacuum-no-cleanup-lock.spec#L1-L5), [vacuum-no-cleanup-lock.spec#pinholder](../../../raw/postgres-17/src/test/isolation/specs/vacuum-no-cleanup-lock.spec#L24-L35), [vacuum-no-cleanup-lock.spec#vacuumer](../../../raw/postgres-17/src/test/isolation/specs/vacuum-no-cleanup-lock.spec#L58-L69) |
| `VACUUM` keeps deleted rows another session's snapshot can still see in a permanent table, and removes them in a temporary one | [horizons.spec#vacuum-permanent](../../../raw/postgres-17/src/test/isolation/specs/horizons.spec#L143-L155), [horizons.spec#vacuum-temporary](../../../raw/postgres-17/src/test/isolation/specs/horizons.spec#L157-L169) |
| `SKIP_LOCKED` on a locked relation | [vacuum-skip-locked.spec#purpose](../../../raw/postgres-17/src/test/isolation/specs/vacuum-skip-locked.spec#L1-L4) |
| `INDEX_CLEANUP` and `TRUNCATE` as options and as storage parameters, including a truncation that empties the file | [vacuum.sql#INDEX_CLEANUP](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L154-L189), [vacuum.sql#TRUNCATE](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L191-L200) |
| the auto-analyze threshold decision, the index-vacuum bypass decision, and the value of `n_mod_since_analyze` | **no coverage.** `pg_regress` adds only `log_autovacuum_min_duration = 0` to the temporary instance's configuration, so nothing asserts a launcher decision ([pg_regress.c#extra-config](../../../raw/postgres-17/src/test/regress/pg_regress.c#L2392-L2404)), and no shipped test references the bypass at all |

## Related Structures and Functions

| Symbol | Role in the suite |
|---|---|
| [nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183) | the verdict family 1 is built around, and the `DEBUG1` line that makes the engine its own oracle |
| [nbtree.h#BTEQUALIMAGE_PROC](../../../raw/postgres-17/src/include/access/nbtree.h#L706-L712) | support function 4, the slot tests 12 to 16 fill with custom procedures |
| [nbtree.h#BTGetDeduplicateItems](../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150) | the reloption read separately by the insert path and by the build; retained fixtures leave it at its default |
| [nbtree.h#BTGetTargetPageFreeSpace](../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145) | the leaf fill target behind the fillfactor controls 53 to 55 |
| [nbtree.h#P_ISDELETED](../../../raw/postgres-17/src/include/access/nbtree.h#L218-L225) | the page flags the drain and test 77 produce |
| [pg_index.h#indpred](../../../raw/postgres-17/src/include/catalog/pg_index.h#L58-L62) | the stored predicate that defines every family 2 fixture |
| [pg_index.h#indnkeyatts](../../../raw/postgres-17/src/include/catalog/pg_index.h#L33-L36) | the key-versus-`INCLUDE` split behind tests 10, 46, 47 and 100 to 105 |
| [pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L64-L66) | the count families 5 and 6 make stale, zero and unknown |
| [analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953) | how a partial index's own population is estimated, and test 120's post-census assertion |
| [analyze.c#acquire_sample_rows-seed](../../../raw/postgres-17/src/backend/commands/analyze.c#L1185-L1194) | the random block seed and reservoir state that make test 120 probabilistic |
| [autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3076) | the exact arithmetic rule 3 applies |
| [autovacuum.c#anl-effective-values](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3017) | the reloption-over-GUC precedence that decides fixtures 94 and 95 |
| [rel.h#AutoVacOpts](../../../raw/postgres-17/src/include/utils/rel.h#L308-L326) | the per-table option struct rule 3's effective values come from |
| [autovacuum.c#dovacuum](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3093-L3094) | the launcher's vacuum verdict, which the maintenance assumption stands in for without recomputing |
| [autovacuum.c#autovacuum_do_vac_analyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3120-L3147) | the worker's call into `vacuum()`, which is why a foreground `VACUUM ANALYZE` is a faithful stand-in |
| [reloptions.c#autovacuum_enabled](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L105-L113) | the per-table opt-out a fixture may set without being exempted from the maintenance step |
| [pgstat_relation.c#report_vacuum-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L233-L246) | what the `VACUUM` half of the maintenance step leaves in the counters the launcher reads |
| [pgstat.c#pgstat_report_stat](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L583-L600) | the only path by which churn reaches `mod_since_analyze`, and the forced-flush flag |
| [index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2842) | the count a build and the oracle rebuild leave behind, including the `-1` it preserves for an empty index |
| [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L252) | the physical reading, with its B-tree, other-session temp and invalid-index refusals |
| [vacuumlazy.c#lazy_scan_noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1653-L1849) | the fallback for a page whose cleanup lock is refused, which counts missed dead tuples and is the only place an aggressive run decides to wait for the lock |
| [bufmgr.c#ConditionalLockBufferForCleanup](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5456-L5508) | the cleanup-lock test that any other backend's pin fails, checkpointer and background writer included |
| [vacuumlazy.c#lazy_truncate_heap](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2546-L2673) | the truncation retry loop, its two give-up messages and its silent give-up when the table grew |
| [pgstat_relation.c#pgstat_report_vacuum](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L227-L257) | the one writer of `vacuum_count`, called only at the end of the heap pass |
| [heapam_handler.c#analyze-dead-rows](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1054-L1085) | why `ANALYZE`'s dead-row count includes the `LP_DEAD` stubs a bypass left |
| [nbtpage.c#_bt_vacuum_needs_cleanup](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L178-L223) | the 5 % rule that decides whether a `VACUUM` with nothing to delete scans the index at all |

## Interactions with Other Concepts

- **Deduplication.** Family 1 tests deduplication eligibility with the default reloption. Explicitly enabling or disabling the reloption is outside the suite. How much a page actually saves is a property of the insert-time pass and the write history ([nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L56-L70)), which the suite treats as an outcome to measure, not an input to model.
- **VACUUM.** The suite depends on `VACUUM` to convert dead entries into reclaimable pages, and the maintenance assumption runs it after every churn, so it withholds it nowhere: the four fixtures that skipped it after entries left an index (65, 67, 113a and 113c) and the four that ran it and skipped the following `ANALYZE` (69, 106, 117 and 121) are retired. Four boundaries: the suite never reproduces the launcher's vacuum verdict, since it runs the `VACUUM` whether or not the dead-tuple or insert threshold was crossed; it does not require a run to assert that a `VACUUM` it did run removed the entries, because the bypass and a refused cleanup lock both leave them in place, though it recommends a procedure that asserts exactly that ([Recommended: proving VACUUM did all its work](#recommended-proving-vacuum-did-all-its-work)); it forbids a fixture from defeating the `VACUUM` it issued, by a held snapshot, a prepared transaction, a slot `xmin`, a conflicting lock or a timeout ([The maintenance must not be defeated](#the-maintenance-must-not-be-defeated)); and it scores no method against an index whose dead entries were never vacuumed.
- **ANALYZE and the planner's statistics.** The maintenance assumption, rule 3 and families 2, 5 and 6 lean on what `ANALYZE` writes. The suite does not check plan shape or selectivity; a claim about how the planner prices a bloated index is a different question and a different page.
- **Cumulative statistics.** Rule 3 is the only required part of the suite that reads a statistics view rather than the catalog, and the recommended completeness procedure also reads `vacuum_count`, `analyze_count` and `n_dead_tup`; both inherit that subsystem's timing: pending counts are published by a flush, not by a commit, and a view read may be served from a cached snapshot. The suite's boundary is that it fixes publication points and a fetch consistency and then reads; it makes no claim about the subsystem's own behavior beyond the counter it uses.
- **REINDEX.** The oracle is the blocking `REINDEX INDEX` form. Comment survival, lock levels and the concurrent form belong to the method under test, not to this suite.
- **The measurement interface.** Whether a method reads `pgstatindex`, `bt_metap`, catalog columns only, or raw pages is out of scope here; the suite scores the decision, not the instrument. A method that cannot run on the fixture set says so and records the fixtures it skipped.

## Context Reviewed

- The nbtree deduplication path: `_bt_allequalimage` and its `INCLUDE` refusal, the per-key-column `BTEQUALIMAGE_PROC` lookup and call, the `DEBUG1` verdict and its `debugmessage` parameter, both callers of the function (`_bt_leafbuild` with `true`, `btbuildempty` with `false`), the build-time gate in `_bt_load`, and the insert-time gate in `_bt_findinsertloc` around `_bt_dedup_pass`.
- The `deduplicate_items` reloption: its `RELOPT_KIND_BTREE` entry, its `ShareUpdateExclusiveLock` level and the "applies only to later inserts" reason recorded beside it, and how `ALTER TABLE`/`ALTER INDEX ... SET` derives that lock level through `AlterTableGetRelOptionsLockLevel`.
- The two shipped equal-image callbacks and the collation determinism they read, plus `CREATE COLLATION`'s `deterministic` option and its catalog column.
- Internal-function resolution by `prosrc`, which decides tests 14 and 16, down to `fmgr_lookupByName`'s name scan of the generated `fmgr_builtins` table.
- The B-tree build geometry: `_bt_pagestate`'s leaf and non-leaf fill targets, `BTGetFillFactor`, `BTGetTargetPageFreeSpace` and the fillfactor constants; and `nbtsort.c`'s sorted-input contract, which is why leaf order is key order.
- Page reclamation: `btbulkdelete`, `btvacuumscan`'s loop over every block from `BTREE_METAPAGE + 1` to the relation length, `btvacuumpage`, `_bt_pagedel`, the deleted and half-dead page flags, and how `pgstatindex` classifies them.
- The index-vacuum bypass in full: `BYPASS_THRESHOLD_PAGES`, the `consider_bypass_optimization` initialization and the two places that clear it, the `rel_pages > 0` guard, the strict `lpdead_item_pages` comparison, the 32 MB `TidStoreMemoryUsage` cap, and the fact that cleanup still runs when the bypass applies.
- The four writers of `reltuples`: a build through `index_build`'s two `index_update_stats` calls, `ANALYZE` through `tupleFract`, `VACUUM` through `update_relstats_all_indexes` unless the AM marked the count estimated, and `RelationSetNewRelfilenumber`'s `-1`; plus `AddNewRelationTuple`'s initial `-1` and `index_update_stats`'s hack that preserves `-1` for an empty relation.
- The `REINDEX INDEX` path: `ReindexIndex`'s lock acquisition and four-way dispatch, and `reindex_index`'s suppress / new-relfilenode / `index_build` sequence.
- The autovacuum analyze threshold end to end: `relation_needs_vacanalyze`'s reloption-or-GUC selection for both analyze parameters, `extract_autovac_opts` and the `AutoVacOpts` member of `StdRdOptions`, the negative-`reltuples` clamp, the strict comparison, the `autovacuum_enabled` short circuit, the two GUCs, and the two per-table reloptions with their lock level.
- The vacuum side of the same function: the dead-tuple and insert-vacuum thresholds, their reloption-or-GUC selection, the `-1` that disables insert vacuums, the `AutoVacuumingActive` gate on `autovacuum` and `track_counts`, the two verdicts returned together, and `do_autovacuum`'s walk over every table; the four vacuum-side GUCs and `autovacuum_naptime`, and the five matching reloptions - `autovacuum_enabled` included - with their lock level.
- The worker's path into `vacuum()` through `autovacuum_do_vac_analyze`, the SQL command's path into the same routine through `ExecVacuum`, and the per-relation loop that vacuums before it analyzes; `pgstat_report_vacuum`'s live, dead and insert-counter writes; `n_dead_tup` and `n_ins_since_vacuum` in `pg_stat_all_tables`; and the documentation's statement of the three thresholds and of the default-on daemon.
- The maintenance statement's silent failure modes: `vacuum_get_cutoffs`' acquisition of `OldestXmin` from `GetOldestNonRemovableTransactionId`, that function's per-relation horizon switch, `ComputeXidHorizons`' per-backend `xid`/`xmin` loop with its `PROC_IN_VACUUM`/`PROC_IN_LOGICAL_DECODING` skip and its replication-slot arms, `twophase.c`'s note that a prepared transaction's dummy `PGPROC` is what keeps its xid considered running, `vacuum_rel`'s `PROC_IN_VACUUM` comment and its wraparound flag, `lazy_scan_heap`'s dead-TID gate on `lazy_vacuum`, the `recently_dead_tuples` counter with `lazy_scan_noprune`'s `HEAPTUPLE_RECENTLY_DEAD` arm and the `VERBOSE` tuples line that publishes it, `_bt_pendingfsm_finalize`'s horizon recomputation and its give-up-at-the-first-non-recyclable-page loop beside `BTPageIsRecyclable`, `vacuum_open_relation`'s conditional lock with its four skip messages, the `VACOPT_SKIP_LOCKED` bit a non-wraparound worker is given, `ProcSleep`'s cancel of a blocking worker, the launcher's and worker's forcing of four settable timeouts to zero with the comment naming the reason, those GUCs' contexts, and the `pg_stat_activity` and `pg_replication_slots` columns that name a horizon holder.
- Statistics publication: `pgstat_report_stat`'s force flag and interval constants, its call site in the backend's idle path, `pgstat_force_next_flush` and its SQL wrapper, the relation flush callback that adds to `mod_since_analyze`, `pgstat_report_analyze`'s reset and the comment on what it forgets, `n_mod_since_analyze` in `pg_stat_all_tables`, `stats_fetch_consistency` and `pg_stat_clear_snapshot`.
- Sampling: `acquire_sample_rows`'s random block seed and reservoir replacement, `std_typanalyze`'s `300 * attstattarget`, `default_statistics_target`, and the `attstattarget = 0` skip.
- Extended `ndistinct` statistics and their catalog kind.
- `pgstatindex`'s entry checks, page classes, size arithmetic and density output.
- The executor's partial-index predicate skip, and `pg_index`'s `indpred`, `indnkeyatts` and `indisvalid`.
- The build's generated artifacts: the backend `generated-headers` target, `genbki.pl`'s catalog outputs, and `Gen_fmgrtab.pl`'s `fmgrtab.c`/`fmgroids.h`/`fmgrprotos.h`.
- The shipped tests that touch the same behavior - `check_btree.sql`, `pageinspect`'s `btree.sql`, `index_including.sql`, `pgstattuple.sql`, `stats.sql`, `vacuum.sql` - and `pg_regress`'s extra configuration, which is where the absence of launcher and bypass coverage is visible.
- The documentation of `make check` and the contrib test suites.
- For the recommended completeness procedure: `heap_vacuum_rel` from its phase switches to its `VERBOSE` report, including every line of that report; `lazy_scan_heap`'s conditional cleanup lock, `lazy_scan_noprune` with its missed-tuple count and its aggressive wait, and `heap_tuple_should_freeze`; `find_next_unskippable_block` and `SKIP_PAGES_THRESHOLD`; the bypass, the failsafe, the second heap pass and its visibility-map update; `should_attempt_truncation`, `lazy_truncate_heap` and `count_nondeletable_pages`; `vacuum_rel`'s skip paths, storage-parameter defaults and TOAST recursion, `vacuum_get_cutoffs` with the `FREEZE` ages, `vac_update_relstats` and `vac_update_datfrozenxid`; `ConditionalLockBufferForCleanup`, `LockBufferForCleanup` and the checkpointer and background writer's `SyncOneBuffer`; `ComputeXidHorizons`' same-database rule, standby feedback and slot `catalog_xmin`; `pgstat_report_vacuum`, `pgstat_report_analyze` and `ANALYZE`'s dead-row count and block sample; `btvacuumcleanup`, `_bt_vacuum_needs_cleanup`, `btvacuumpage`'s recycling and `_bt_allocbuf`; opportunistic pruning in `heap_page_prune_opt`; the `pg_stat_activity`, `pg_prepared_xacts`, `pg_replication_slots`, `pg_locks`, `pg_stat_all_tables` and `pg_stat_progress_vacuum` definitions; wait-event names; `psql`'s notice printing; the `VACUUM` reference page; and the `vacuum-no-cleanup-lock`, `horizons` and `vacuum-skip-locked` isolation specs.
- The procedure's SQL and shell checks were run, byte for byte, against a disposable cluster built from this pin, with a clean fixture and with a held snapshot, a held pin, a bypass and a lock holder injected, to check syntax and message shapes. No number from those runs is published here, because this page is source-only; every behavioral claim rests on the citations.

## Evidence Map

| Claim | Evidence |
|---|---|
| Deduplication safety is per index, refuses `INCLUDE`, and calls support function 4 per key column | [nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183) |
| A build deduplicates only when equal-image, not unique, and `deduplicate_items` is on | [nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152) |
| A nondeterministic collation makes a character key non-equal-image | [varlena.c#btvarstrequalimage](../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613) |
| An internal alias resolves by `prosrc`, so a rename still reaches the builtin | [fmgr.c#internal-alias](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240) |
| A partial index's population is estimated as `ceil(tupleFract * totalrows)` | [analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953) |
| The executor skips an index whose predicate the row fails | [execIndexing.c#partial-predicate-skip](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L378-L387) |
| The `deduplicate_items` reloption applies to later inserts and takes `ShareUpdateExclusiveLock` | [reloptions.c#deduplicate_items](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168), [reloptions.c#AlterTableGetRelOptionsLockLevel](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L2110-L2141) |
| The `DEBUG1` verdict is silent for an `INCLUDE` index and for `btbuildempty` | [nbtutils.c#INCLUDE-refusal](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5147), [nbtsort.c#_bt_leafbuild-allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L559-L563), [nbtree.c#btbuildempty-allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L158-L167) |
| Rule 3's threshold is the engine's own analyze test, applied strictly | [autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3076), [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095) |
| A table's analyze reloptions take precedence over the cluster GUCs | [autovacuum.c#anl-effective-values](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3017), [autovacuum.c#extract_autovac_opts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2704-L2723), [rel.h#AutoVacOpts](../../../raw/postgres-17/src/include/utils/rel.h#L308-L326) |
| A negative `reltuples` counts as zero in that arithmetic | [autovacuum.c#reltuples-clamp](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3065-L3072) |
| The defaults behind rule 3 are 50 and 0.1, both reloadable | [guc_tables.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3367-L3375), [guc_tables.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3906-L3914) |
| Autovacuum is on by default, and a worker checks every table for both verdicts at once | [guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457), [maintenance.sgml#autovacuum-daemon](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L853-L885), [autovacuum.c#do_autovacuum-census](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2021-L2033), [autovacuum.c#dovacuum](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3093-L3094) |
| A worker's maintenance is the `vacuum()` the SQL command calls, and it vacuums before it analyzes | [autovacuum.c#autovacuum_do_vac_analyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3120-L3147), [vacuum.c#ExecVacuum-vacuum](../../../raw/postgres-17/src/backend/commands/vacuum.c#L450-L451), [vacuum.c#vacuum-then-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650) |
| The launcher's vacuum verdicts are `50 + 0.2 * reltuples` dead tuples and `1000 + 0.2 * reltuples` inserts, reloptions first, all reloadable | [autovacuum.c#vacthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3075), [autovacuum.c#vac-effective-values](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2993-L3009), [guc_tables.c#autovacuum_vacuum_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3349-L3357), [guc_tables.c#autovacuum_vacuum_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3886-L3894), [guc_tables.c#autovacuum_vacuum_insert_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3358-L3366), [guc_tables.c#autovacuum_vacuum_insert_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3896-L3904), [reloptions.c#autovacuum_vacuum_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L225-L233) |
| `autovacuum = off` and `autovacuum_enabled = false` each stop the launcher's verdicts for a table, wraparound forcing aside | [autovacuum.c#AutoVacuumingActive](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3235-L3241), [autovacuum.c#av_enabled-return](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3048-L3054), [reloptions.c#autovacuum_enabled](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L105-L113) |
| A `VACUUM` reports live and dead counts and zeroes the insert counter the launcher reads | [pgstat_relation.c#report_vacuum-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L233-L246), [system_views.sql#n_dead_tup](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L688), [system_views.sql#n_ins_since_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L690) |
| Churn reaches `mod_since_analyze` only through a flush, unforced at most once a second | [pgstat.c#PGSTAT_MIN_INTERVAL](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122), [pgstat.c#flush-intervals](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665), [pgstat_relation.c#mod_since_analyze](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860) |
| `ANALYZE` resets the counter and forgets changes committed while it ran | [pgstat_relation.c#report_analyze-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L331-L337) |
| A flush can be forced from SQL, and a cached read discarded | [pg_proc.dat#pg_stat_force_next_flush](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5916-L5920), [pgstat.c#pgStatForceNextFlush](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L595-L600), [pg_proc.dat#pg_stat_clear_snapshot](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5911-L5915), [guc_tables.c#stats_fetch_consistency](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4966-L4974) |
| An `ANALYZE` rewrites index `reltuples`, which is why forgeries run last | [analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660) |
| `VACUUM` writes an exact index count unless the AM marked it estimated | [vacuumlazy.c#update_relstats_all_indexes](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3096), [nbtree.c#btvacuumcleanup-estimated_count](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L884-L894) |
| A new relfilenode leaves `reltuples = -1`, visible before the build runs | [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3943-L3960), [relcache.c#relfilenumber-visible](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3967-L3973) |
| A rebuilt or freshly built **empty** index keeps `reltuples = -1`, not `0` | [index.c#index_update_stats-empty-hack](../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842), [heap.c#AddNewRelationTuple-reltuples](../../../raw/postgres-17/src/backend/catalog/heap.c#L1004-L1016) |
| The oracle's rebuild is a new relfilenode plus `index_build`, which updates both rows | [index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789), [index.c#index_build-update-stats](../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3138), [indexcmds.c#ReindexIndex-dispatch](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2838-L2849) |
| Index vacuuming is bypassed only under four conditions, of which the 2 % page fraction is one | [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89), [vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934), [vacuumlazy.c#consider_bypass_optimization-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L407) |
| A held snapshot, a prepared transaction or a replication slot pins `VACUUM`'s removal horizon, and a vacuuming or decoding backend is the only exemption | [vacuum.c#vacuum_get_cutoffs-OldestXmin](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1109-L1122), [procarray.c#GetOldestNonRemovableTransactionId](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2005-L2026), [procarray.c#ComputeXidHorizons-backend-xmin](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1792-L1815), [procarray.c#skip-vacuum-and-decoding](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1817-L1832), [procarray.c#slot-horizons](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1896-L1902), [twophase.c#dummy-pgproc](../../../raw/postgres-17/src/backend/access/transam/twophase.c#L24-L26), [vacuum.c#PROC_IN_VACUUM](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2006-L2029) |
| Under a pinned horizon the drained rows stay recently dead and are only counted, and index vacuuming is entered only when dead TIDs were collected | [vacuumlazy.c#recently_dead_tuples](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L208-L212), [vacuumlazy.c#lazy_scan_noprune-recently-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1770-L1776), [vacuumlazy.c#lazy_vacuum-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1052), [vacuumlazy.c#verbose-tuples-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L657-L663) |
| A deleted B-tree page is recycled only once the horizon passes its `safexid`, and the same-VACUUM pass stops at the first page that fails the test | [nbtpage.c#pendingfsm-horizon](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3023-L3052), [nbtree.h#BTPageIsRecyclable](../../../raw/postgres-17/src/include/access/nbtree.h#L279-L318) |
| A non-wraparound autovacuum is given `VACOPT_SKIP_LOCKED` and skips a table it cannot lock, and a backend blocked behind one signals it to cancel | [autovacuum.c#skip-locked](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2825-L2830), [vacuum.c#vacuum_open_relation-conditional](../../../raw/postgres-17/src/backend/commands/vacuum.c#L785-L800), [vacuum.c#skip-lock-not-available](../../../raw/postgres-17/src/backend/commands/vacuum.c#L828-L855), [proc.c#cancel-blocking-autovacuum](../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1472) |
| The launcher and the worker force four settable `PGC_USERSET` timeouts to zero so they cannot prevent regular maintenance | [autovacuum.c#launcher-timeouts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L518-L526), [autovacuum.c#worker-timeouts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L1462-L1470), [guc_tables.c#statement_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620), [guc_tables.c#lock_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631), [guc_tables.c#idle_in_transaction_session_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2633-L2642), [guc_tables.c#transaction_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2644-L2653) |
| A horizon holder is visible as a session's `xact_start` and `backend_xmin` or a slot's `xmin` | [system_views.sql#pg_stat_activity-xmin](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L878-L885), [system_views.sql#pg_replication_slots-xmin](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1006-L1017) |
| The drain's `ctid` block number is the item pointer's block id | [itemptr.h#ItemPointerData](../../../raw/postgres-17/src/include/storage/itemptr.h#L36-L40) |
| Leaf order is key order, so the drain's spread across leaves is fixture-dependent | [nbtsort.c#sorted-build](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L4-L15), [nbtsort.c#_bt_load](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1130-L1136) |
| The drain's `VACUUM` does visit every index block | [nbtree.c#btvacuumscan-block-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1016-L1041) |
| Test 120's sample size is `300 * attstattarget`, and its selection is random | [analyze.c#std_typanalyze-minrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L1888-L1895), [analyze.c#acquire_sample_rows-seed](../../../raw/postgres-17/src/backend/commands/analyze.c#L1185-L1194) |
| An internal alias resolves against a table generated from `pg_proc.dat` | [fmgr.c#fmgr_lookupByName](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L95-L111), [Makefile#fmgr-stamp](../../../raw/postgres-17/src/backend/utils/Makefile#L46-L53) |
| No shipped test asserts a launcher analyze decision or a bypass decision | [pg_regress.c#extra-config](../../../raw/postgres-17/src/test/regress/pg_regress.c#L2392-L2404) |
| The engine's regression suites do not exercise a bloat method | [regress.sgml#make-check](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59) |
| A non-aggressive `VACUUM` never waits for a cleanup lock; a refused one leaves dead rows as missed, reported only when nonzero and added to `n_dead_tup` | [vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L923-L961), [vacuumlazy.c#noprune-contract](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1633-L1651), [vacuumlazy.c#missed-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1762-L1769), [vacuumlazy.c#verbose-missed-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668), [vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591) |
| The checkpointer and the background writer pin a dirty buffer while they write it, and any other pin refuses the cleanup lock | [bufmgr.c#SyncOneBuffer-pin](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3546-L3567), [bufmgr.c#ConditionalLockBufferForCleanup](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5456-L5508) |
| `FREEZE` makes the run aggressive with `FreezeLimit` at the removal cutoff, so the no-prune path waits for the lock on a page holding an old unfrozen or dead row | [vacuum.c#FREEZE-ages](../../../raw/postgres-17/src/backend/commands/vacuum.c#L385-L402), [vacuum.c#FreezeLimit](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1160-L1177), [vacuum.c#aggressive-decision](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1199-L1241), [heapam.c#heap_tuple_should_freeze](../../../raw/postgres-17/src/backend/access/heap/heapam.c#L8247-L8284), [vacuumlazy.c#noprune-aggressive-wait](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1713-L1733) |
| `INDEX_CLEANUP ON` turns the bypass off, and only the failsafe can still skip index vacuuming | [vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407), [vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934) |
| `VACUUM` keeps bypassed dead item identifiers out of its statistics, while `ANALYZE` counts every `LP_DEAD` stub as a dead row | [vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591), [heapam_handler.c#analyze-dead-rows](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1054-L1085) |
| `vacuum_count` moves only when the heap pass reaches its end, and `ANALYZE` reads every block of a table no larger than its sample target | [pgstat_relation.c#pgstat_report_vacuum](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L227-L257), [analyze.c#two-stage-sample](../../../raw/postgres-17/src/backend/commands/analyze.c#L1136-L1137) |
| Truncation retries a conditional `AccessExclusiveLock` for about 5 s, reports its give-up, and gives up silently when the table grew | [vacuumlazy.c#truncate-lock-retry](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2577-L2608), [vacuumlazy.c#truncate-rel-grew](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2610-L2627) |
| `VACUUM` skips another session's temporary table without a message, and opens only indexes marked `indisready` | [vacuum.c#other-temp-silent](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2113-L2126), [vacuum.c#vac_open_indexes-indisready](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2345-L2357) |
| `INFO` always reaches the client, and `psql` prints notices on standard error | [elog.c#should_output_to_client](../../../raw/postgres-17/src/backend/utils/error/elog.c#L248-L264), [common.c#NoticeProcessor](../../../raw/postgres-17/src/bin/psql/common.c#L269-L273), [logging.c#locus-prefix](../../../raw/postgres-17/src/common/logging.c#L244-L261) |
| A B-tree page deleted by this `VACUUM` is recyclable within it only if another backend consumes an XID meanwhile, and `pgstatindex` counts recyclable and stuck deleted pages alike | [nbtpage.c#pendingfsm-horizon](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3012-L3052), [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1179), [pgstatindex.c#deleted-pages-count](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L307) |
| `btvacuumcleanup` skips the index scan unless prior deleted pages exceed 5 % of the index | [nbtree.c#btvacuumcleanup-skip](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L855-L874), [nbtpage.c#_bt_vacuum_needs_cleanup](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L178-L223) |

## Open Questions

- **Consumer suites still include retired deduplication controls.** The core-SQL estimator, COMMENT-baseline heuristic and `pgstatindex` consumer pages retain fixtures or coverage text for tests 11, 11b or 38. Their scripts and historical results have not been revised or re-run as part of this concept-only change.

- **The unvacuumed blind spot is no longer built at all.** Tests 65, 67, 113a and 113c left their index entries dead but physically present, so a density reading saw a full file while a rebuild would write an almost empty one. With those fixtures retired and the maintenance assumption in force the suite neither builds that state nor scores what a method does with it, and it still pins no second input - an index entry count against leaf capacity, or `n_dead_tup` - that could catch the shape. A method that misreads a not-yet-vacuumed index now passes this suite.
- **The effectiveness proof of [The maintenance must not be defeated](#the-maintenance-must-not-be-defeated) is one-sided.** A nonzero `dead but not yet removable` count proves the horizon was pinned ([vacuumlazy.c#verbose-tuples-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L657-L663)), but a zero count proves only that nothing was pinned for that table at that moment, and the drain-exempt fixtures - 70, 71, the whole of family 3 and the listed controls - have no churn and therefore no signal at all. Nothing in the engine reports "no session or slot held the horizon for the duration of this command": `pg_stat_activity` and `pg_replication_slots` are reads taken beside the maintenance statement rather than interlocks around it ([system_views.sql#pg_stat_activity-xmin](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L878-L885), [system_views.sql#pg_replication_slots-xmin](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1006-L1017)). On a no-churn fixture the rule reduces to the prohibition itself, unenforced by a measurement. [Recommended: proving VACUUM did all its work](#recommended-proving-vacuum-did-all-its-work) narrows the gap without closing it: its `tuples missed` line, `index scan` label and `ANALYZE` dead-row count are readings taken inside the statement rather than beside it ([vacuumlazy.c#verbose-missed-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668), [vacuumlazy.c#verbose-index-scan-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L717), [analyze.c#verbose-sample-line](../../../raw/postgres-17/src/backend/commands/analyze.c#L1296-L1306)), but its horizon-holder queries are still reads, and on a no-churn fixture none of those counters has anything to count.
- **`DISABLE_PAGE_SKIPPING` does less than its documentation says.** The reference page says the option disables all page skipping, including pages skipped "to avoid waiting for other sessions to finish using them" ([ref/vacuum.sgml#DISABLE_PAGE_SKIPPING](../../../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L147-L164)). The source makes the run aggressive and turns off visibility-map skipping, but a page whose cleanup lock is refused still takes the no-prune path unless one of its tuples is older than the freeze cutoff, which stays at `vacuum_freeze_min_age` ([vacuumlazy.c#aggressive-and-skipwithvm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L449-L467), [vacuumlazy.c#noprune-aggressive-wait](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1713-L1733), [vacuum.c#FreezeLimit](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1160-L1177)). The source wins, which is why the procedure escalates with `FREEZE` instead.
- **The recommended procedure has no consumer run yet.** No consumer page has filed a run under it, and no shipped test exercises the bypass ([pg_regress.c#extra-config](../../../raw/postgres-17/src/test/regress/pg_regress.c#L2392-L2404)). Its SQL and shell checks were exercised against a disposable build of this pin, but the page publishes no number from that run, so how often each branch of the map occurs on the suite's fixtures is unmeasured.
- **`CHECKPOINT` lowers the risk of a background-writer pin; it does not remove it.** A checkpoint that starts while the statement runs may write a page the statement has not reached yet, and the background writer keeps running ([bufmgr.c#SyncOneBuffer-pin](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3546-L3567)). The procedure therefore keeps the missed-tuple check and its re-run rather than claiming prevention. A run that recorded which process held the pin would close this.
- **An escalation changes the state it proves.** `FREEZE` freezes every eligible row, and `INDEX_CLEANUP ON` removes entries an autovacuum worker at the defaults might have left ([vacuum.c#FREEZE-ages](../../../raw/postgres-17/src/backend/commands/vacuum.c#L385-L402), [autovacuum.c#unspecified-options](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2832-L2840)). Whether a fixture whose maintenance needed tier 1 still models an autovacuum-maintained table is the asker's call; the procedure only requires the departure to be declared.
- **Nothing pins the row count `VACUUM` writes.** 69, 106, 117 and 121 were the fixtures whose recipe ended on a `VACUUM`, the state that leaves `update_relstats_all_indexes` as the last writer of a count ([vacuumlazy.c#update_relstats_all_indexes](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3096), [nbtree.c#btvacuumcleanup-estimated_count](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L884-L894)). The maintenance assumption closes the last way to reach it: `VACUUM ANALYZE` analyzes after it vacuums ([vacuum.c#vacuum-then-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650)), so on every churned table the sampled estimate overwrites the exact count before the decide phase, and no fixture vacuums an unchurned table.
- **The vacuum side is assumed, not simulated.** The launcher's dead-tuple and insert thresholds are recorded under the maintenance assumption but never applied: the suite runs `VACUUM ANALYZE` after every churn. A method whose behavior differs between a table the launcher would have vacuumed and one it would have left is therefore tested on the first case only. A vacuum-side census in the shape of rule 3 would recover the second case, at the price of re-creating sub-threshold unvacuumed indexes unless the churn `VACUUM` stayed mandatory beside it.
- **Fixtures 92 to 95 no longer isolate the analyze threshold.** They were built so that rule 3's census would analyze 94 and leave 95 alone, exercising the reloption-over-GUC precedence; under the maintenance assumption all four are analyzed after their row changes, so the census finds nothing to decide on them. They keep their numbers and their reclaimable index. Whether to redefine them as unchurned tables whose build-phase changes sit on either side of the threshold, or to retire them, is the asker's call; until then the precedence arithmetic remains part of the census's definition, but no remaining fixture makes a wrong reading of it visible.
- **118 and 119 now name one state.** 118's point was an index whose own `reltuples` read 0 from the empty-subset `ANALYZE` while 50,000 entries had since arrived. The maintenance `VACUUM ANALYZE` after the arrivals re-estimates that count, which is what 119's extra `ANALYZE` did, so the two fixtures differ only by a second sample. The state 118 used to build is one the launcher would have left, because 50,000 changes on a million-row table are under the 100,050 analyze threshold; the assumption is stronger than the launcher on purpose, and this fixture is the price. Retiring one number, or redefining 118, is the asker's call.
- **Only two of the six majors a 12-through-17 claim covers have fixture legs.** The 13, 14, 15 and 16 legs exist only as feature gates, not as runs, so a behavior that changed in one of them would not be caught.
- **No second block size.** Every geometry expectation follows from `BLCKSZ`, and the suite has no `--with-blocksize=16` leg to prove that a method's arithmetic is not tuned to 8192.
- **The drain is a shape, not a distribution.** One heap block in ten is kept, which produces a specific spread of surviving entries; nothing in the suite varies that ratio or its clustering, so a method sensitive to the *pattern* of deletion rather than its volume would pass unexamined.
- **The drain has no shared per-fixture distribution assertion.** Rule 2 now requires a fixture that depends on the spread across leaves to assert it, but the suite does not define one assertion for all of them, because leaf-level inspection needs an instrument and the suite deliberately leaves the instrument to the method. Two runs may therefore assert the same precondition differently, or one may assert nothing because its method cannot see leaves.
- **Rule 3's second publication point has no way to prove it worked.** A run can force a flush and then read, but nothing in the engine reports "all pending counts for this relation are published". The rule names the publication points and the mechanism; a run that saw a partial flush under lock contention would look identical to one that had nothing pending ([pgstat.c#flush-intervals](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665)).
- **The reloption precedence is applied to the analyze parameters only.** Rule 3 defines the effective per-table analyze threshold and scale factor. The same reloption-or-GUC selection exists for the vacuum and insert-vacuum parameters ([autovacuum.c#vac-effective-values](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2993-L3009)), and the maintenance assumption records it without applying it, since the `VACUUM` runs whatever the counters say; no fixture overrides those four reloptions, so nothing in the suite exercises that half of `relation_needs_vacanalyze`.
- **`want_stage` predictions are not derivable.** A prediction filed before the run is a judgement, and two reviewers can file different ones for the same fixture. The suite records disagreement between prediction and measurement but has no rule for which is at fault.
- **Locale and encoding are fixed.** The ICU fixtures vary a collation on one column; no fixture varies the cluster's locale or encoding, so a character-key method is tested under one collation environment.

## Source References

- [nbtutils.c#_bt_allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5139-L5183)
- [nbtutils.c#INCLUDE-refusal](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5143-L5147)
- [nbtutils.c#allequalimage-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5149-L5170)
- [nbtutils.c#_bt_allequalimage-debug](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5172-L5180)
- [nbtinsert.c#dedup-on-insert](../../../raw/postgres-17/src/backend/access/nbtree/nbtinsert.c#L2774-L2782)
- [nbtree.h#btm_allequalimage](../../../raw/postgres-17/src/include/access/nbtree.h#L113-L119)
- [nbtree.h#BTREE_DEFAULT_FILLFACTOR](../../../raw/postgres-17/src/include/access/nbtree.h#L197-L202)
- [nbtree.h#P_ISDELETED](../../../raw/postgres-17/src/include/access/nbtree.h#L218-L225)
- [nbtree.h#BTPageIsRecyclable](../../../raw/postgres-17/src/include/access/nbtree.h#L279-L318)
- [nbtree.h#BTEQUALIMAGE_PROC](../../../raw/postgres-17/src/include/access/nbtree.h#L706-L712)
- [nbtree.h#BTGetTargetPageFreeSpace](../../../raw/postgres-17/src/include/access/nbtree.h#L1138-L1145)
- [nbtree.h#BTGetDeduplicateItems](../../../raw/postgres-17/src/include/access/nbtree.h#L1146-L1150)
- [nbtsort.c#sorted-build](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L4-L15)
- [nbtsort.c#_bt_leafbuild-allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L559-L563)
- [nbtsort.c#_bt_pagestate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L660-L666)
- [nbtsort.c#_bt_load](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1130-L1136)
- [nbtsort.c#_bt_load-deduplicate](../../../raw/postgres-17/src/backend/access/nbtree/nbtsort.c#L1147-L1152)
- [nbtdedup.c#_bt_dedup_pass](../../../raw/postgres-17/src/backend/access/nbtree/nbtdedup.c#L56-L70)
- [nbtpage.c#_bt_metaversion](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L739-L750)
- [nbtpage.c#_bt_pagedel](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1802-L1815)
- [nbtpage.c#pendingfsm-horizon](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3023-L3052)
- [nbtree.c#btbuildempty-allequalimage](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L158-L167)
- [nbtree.c#btbulkdelete](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L820-L843)
- [nbtree.c#btvacuumcleanup-estimated_count](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L884-L894)
- [nbtree.c#btvacuumscan-block-loop](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1016-L1041)
- [datum.c#btequalimage](../../../raw/postgres-17/src/backend/utils/adt/datum.c#L432-L438)
- [varlena.c#btvarstrequalimage](../../../raw/postgres-17/src/backend/utils/adt/varlena.c#L2595-L2613)
- [fmgr.c#fmgr_lookupByName](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L95-L111)
- [fmgr.c#internal-alias](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L243)
- [pg_collation.h#collisdeterministic](../../../raw/postgres-17/src/include/catalog/pg_collation.h#L40)
- [collationcmds.c#deterministic](../../../raw/postgres-17/src/backend/commands/collationcmds.c#L200-L207)
- [pg_index.h#indnkeyatts](../../../raw/postgres-17/src/include/catalog/pg_index.h#L33-L36)
- [pg_index.h#indpred](../../../raw/postgres-17/src/include/catalog/pg_index.h#L58-L62)
- [pg_class.h#reltuples](../../../raw/postgres-17/src/include/catalog/pg_class.h#L64-L66)
- [pg_attribute.h#attstattarget](../../../raw/postgres-17/src/include/catalog/pg_attribute.h#L166-L177)
- [pg_statistic_ext.h#STATS_EXT_NDISTINCT](../../../raw/postgres-17/src/include/catalog/pg_statistic_ext.h#L84)
- [pg_proc.dat#pg_stat_clear_snapshot](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5911-L5915)
- [pg_proc.dat#pg_stat_force_next_flush](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L5916-L5920)
- [rel.h#AutoVacOpts](../../../raw/postgres-17/src/include/utils/rel.h#L308-L326)
- [pgstat.h#PgStat_FetchConsistency](../../../raw/postgres-17/src/include/pgstat.h#L68-L73)
- [mvdistinct.c#statext_ndistinct_build](../../../raw/postgres-17/src/backend/statistics/mvdistinct.c#L86-L100)
- [execIndexing.c#partial-predicate-skip](../../../raw/postgres-17/src/backend/executor/execIndexing.c#L378-L387)
- [analyze.c#analyze-lockmode](../../../raw/postgres-17/src/backend/commands/analyze.c#L134-L145)
- [analyze.c#tupleFract-default](../../../raw/postgres-17/src/backend/commands/analyze.c#L445-L450)
- [analyze.c#totalindexrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L648-L660)
- [analyze.c#partial-index-skip](../../../raw/postgres-17/src/backend/commands/analyze.c#L859-L863)
- [analyze.c#tupleFract](../../../raw/postgres-17/src/backend/commands/analyze.c#L948-L953)
- [analyze.c#attstattarget-zero](../../../raw/postgres-17/src/backend/commands/analyze.c#L1024-L1032)
- [analyze.c#acquire_sample_rows-seed](../../../raw/postgres-17/src/backend/commands/analyze.c#L1185-L1194)
- [analyze.c#reservoir-replacement](../../../raw/postgres-17/src/backend/commands/analyze.c#L1207-L1229)
- [analyze.c#std_typanalyze-minrows](../../../raw/postgres-17/src/backend/commands/analyze.c#L1888-L1895)
- [heap.c#AddNewRelationTuple-reltuples](../../../raw/postgres-17/src/backend/catalog/heap.c#L1004-L1016)
- [index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2809-L2842)
- [index.c#index_update_stats-empty-hack](../../../raw/postgres-17/src/backend/catalog/index.c#L2825-L2842)
- [index.c#index_build-update-stats](../../../raw/postgres-17/src/backend/catalog/index.c#L3126-L3138)
- [index.c#reindex_index-rebuild](../../../raw/postgres-17/src/backend/catalog/index.c#L3781-L3789)
- [indexcmds.c#ReindexIndex-dispatch](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L2838-L2849)
- [vacuum.c#vac_update_relstats](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1410-L1425)
- [vacuumlazy.c#BYPASS_THRESHOLD_PAGES](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L85-L89)
- [vacuumlazy.c#recently_dead_tuples](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L208-L212)
- [vacuumlazy.c#consider_bypass_optimization-init](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L387-L407)
- [vacuumlazy.c#verbose-tuples-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L657-L663)
- [vacuumlazy.c#consider_bypass_optimization-cleared](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L895-L897)
- [vacuumlazy.c#lazy_vacuum-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1047-L1052)
- [vacuumlazy.c#lazy_scan_noprune-recently-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1770-L1776)
- [vacuumlazy.c#bypass-conditions](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1899-L1934)
- [vacuumlazy.c#bypass-applies](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1936-L1957)
- [vacuumlazy.c#update_relstats_all_indexes](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L3073-L3096)
- [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3943-L3960)
- [relcache.c#relfilenumber-visible](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3967-L3973)
- [tablecmds.c#AT_SetRelOptions-lock](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L4693-L4704)
- [autovacuum.c#launcher-timeouts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L518-L526)
- [autovacuum.c#worker-timeouts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L1462-L1470)
- [autovacuum.c#do_autovacuum-census](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2021-L2033)
- [autovacuum.c#extract_autovac_opts](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2704-L2723)
- [autovacuum.c#skip-locked](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2825-L2830)
- [autovacuum.c#vac-effective-values](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2993-L3009)
- [autovacuum.c#anl-effective-values](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3017)
- [autovacuum.c#av_enabled](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3027)
- [autovacuum.c#av_enabled-return](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3048-L3054)
- [autovacuum.c#anltuples](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3063-L3068)
- [autovacuum.c#reltuples-clamp](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3065-L3072)
- [autovacuum.c#vacthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3075)
- [autovacuum.c#anlthresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3074-L3076)
- [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3092-L3095)
- [autovacuum.c#dovacuum](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3093-L3094)
- [autovacuum.c#autovacuum_do_vac_analyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3120-L3147)
- [autovacuum.c#AutoVacuumingActive](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3235-L3241)
- [vacuum.c#ExecVacuum-vacuum](../../../raw/postgres-17/src/backend/commands/vacuum.c#L450-L451)
- [vacuum.c#vacuum](../../../raw/postgres-17/src/backend/commands/vacuum.c#L479-L481)
- [vacuum.c#vacuum-then-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L618-L650)
- [vacuum.c#vacuum_open_relation-conditional](../../../raw/postgres-17/src/backend/commands/vacuum.c#L785-L800)
- [vacuum.c#skip-lock-not-available](../../../raw/postgres-17/src/backend/commands/vacuum.c#L828-L855)
- [vacuum.c#vacuum_get_cutoffs-OldestXmin](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1109-L1122)
- [vacuum.c#PROC_IN_VACUUM](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2006-L2029)
- [vacuum.c#vacuum-lockmode](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2049-L2056)
- [twophase.c#dummy-pgproc](../../../raw/postgres-17/src/backend/access/transam/twophase.c#L24-L26)
- [procarray.c#ComputeXidHorizons-backend-xmin](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1792-L1815)
- [procarray.c#skip-vacuum-and-decoding](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1817-L1832)
- [procarray.c#slot-horizons](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1896-L1902)
- [procarray.c#GetOldestNonRemovableTransactionId](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L2005-L2026)
- [proc.c#cancel-blocking-autovacuum](../../../raw/postgres-17/src/backend/storage/lmgr/proc.c#L1412-L1472)
- [guc_tables.c#autovacuum](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L1450-L1457)
- [guc_tables.c#default_statistics_target](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2069-L2078)
- [guc_tables.c#statement_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2611-L2620)
- [guc_tables.c#lock_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2622-L2631)
- [guc_tables.c#idle_in_transaction_session_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2633-L2642)
- [guc_tables.c#transaction_timeout](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2644-L2653)
- [guc_tables.c#autovacuum_naptime](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3339-L3348)
- [guc_tables.c#autovacuum_vacuum_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3349-L3357)
- [guc_tables.c#autovacuum_vacuum_insert_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3358-L3366)
- [guc_tables.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3367-L3375)
- [guc_tables.c#autovacuum_vacuum_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3886-L3894)
- [guc_tables.c#autovacuum_vacuum_insert_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3896-L3904)
- [guc_tables.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3906-L3914)
- [guc_tables.c#stats_fetch_consistency](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L4966-L4974)
- [reloptions.c#autovacuum_enabled](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L105-L113)
- [reloptions.c#deduplicate_items](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L159-L168)
- [reloptions.c#autovacuum_vacuum_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L225-L233)
- [reloptions.c#autovacuum_vacuum_insert_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L234-L242)
- [reloptions.c#autovacuum_analyze_threshold](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L243-L251)
- [reloptions.c#autovacuum_vacuum_scale_factor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L399-L407)
- [reloptions.c#autovacuum_vacuum_insert_scale_factor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L408-L416)
- [reloptions.c#autovacuum_analyze_scale_factor](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L416-L425)
- [reloptions.c#AlterTableGetRelOptionsLockLevel](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L2110-L2141)
- [system_views.sql#n_dead_tup](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L688)
- [system_views.sql#n_mod_since_analyze](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L689)
- [system_views.sql#n_ins_since_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L690)
- [system_views.sql#pg_stat_activity-xmin](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L878-L885)
- [system_views.sql#pg_replication_slots-xmin](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1006-L1017)
- [pgstat.c#PGSTAT_MIN_INTERVAL](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L117-L122)
- [pgstat.c#pgstat_report_stat](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L583-L600)
- [pgstat.c#pgStatForceNextFlush](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L595-L600)
- [pgstat.c#flush-intervals](../../../raw/postgres-17/src/backend/utils/activity/pgstat.c#L636-L665)
- [pgstat_relation.c#report_vacuum-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L233-L246)
- [pgstat_relation.c#report_analyze-reset](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L331-L337)
- [pgstat_relation.c#mod_since_analyze](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L857-L860)
- [postgres.c#pgstat_report_stat-idle](../../../raw/postgres-17/src/backend/tcop/postgres.c#L4662-L4681)
- [itemptr.h#ItemPointerData](../../../raw/postgres-17/src/include/storage/itemptr.h#L36-L40)
- [sysattr.h#SelfItemPointerAttributeNumber](../../../raw/postgres-17/src/include/access/sysattr.h#L21)
- [Makefile#backend-generated-headers](../../../raw/postgres-17/src/backend/Makefile#L116-L140)
- [Makefile#fmgr-stamp](../../../raw/postgres-17/src/backend/utils/Makefile#L46-L53)
- [Makefile#header-stamp](../../../raw/postgres-17/src/backend/utils/Makefile#L76-L83)
- [Makefile#catalog-generated-headers](../../../raw/postgres-17/src/include/catalog/Makefile#L126-L143)
- [pgstatindex.c#pgstatindex_impl](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L216-L252)
- [pgstatindex.c#page-classes](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L295-L320)
- [index_including.sql#CREATE-INDEX-INCLUDE](../../../raw/postgres-17/src/test/regress/sql/index_including.sql#L8-L27)
- [stats.sql#force-flush](../../../raw/postgres-17/src/test/regress/sql/stats.sql#L99-L110)
- [vacuum.sql#reltuples](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L70-L97)
- [pg_regress.c#extra-config](../../../raw/postgres-17/src/test/regress/pg_regress.c#L2392-L2404)
- [check_btree.sql#deduplicate_items](../../../raw/postgres-17/contrib/amcheck/sql/check_btree.sql#L16-L19)
- [btree.sql#bt_metap](../../../raw/postgres-17/contrib/pageinspect/sql/btree.sql#L5-L8)
- [pgstattuple.sql#pgstatindex](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L18-L37)
- [pgstattuple.sql#refusals](../../../raw/postgres-17/contrib/pgstattuple/sql/pgstattuple.sql#L55-L120)
- [regress.sgml#make-check](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L40-L59)
- [regress.sgml#contrib-suites](../../../raw/postgres-17/doc/src/sgml/regress.sgml#L171-L195)
- [maintenance.sgml#autovacuum-daemon](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L853-L885)
- [maintenance.sgml#thresholds](../../../raw/postgres-17/doc/src/sgml/maintenance.sgml#L899-L952)
- [index.c#index_update_stats](../../../raw/postgres-17/src/backend/catalog/index.c#L2810-L2830)
- [relcache.c#RelationSetNewRelfilenumber-reltuples](../../../raw/postgres-17/src/backend/utils/cache/relcache.c#L3945-L3953)
- [vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L923-L961)
- [vacuumlazy.c#missed-dead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1762-L1769)
- [vacuumlazy.c#verbose-missed-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L664-L668)
- [nbtree.c#btvacuumpage-recycle](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1165-L1179)
- [nbtpage.c#_bt_allocbuf-reuse](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L901-L958)
- [pgstatindex.c#deleted-pages-count](../../../raw/postgres-17/contrib/pgstattuple/pgstatindex.c#L298-L307)
- [vacuumlazy.c#verbose-index-scan-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L695-L717)
- [heapam_visibility.c#HeapTupleSatisfiesVacuum](../../../raw/postgres-17/src/backend/access/heap/heapam_visibility.c#L1162-L1181)
- [vacuumlazy.c#heap_vacuum_rel-cutoffs](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L433-L451)
- [vacuumlazy.c#verbose-cutoff-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L669-L673)
- [vacuumlazy.c#missed-dead-pages](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1834-L1838)
- [vacuumlazy.c#lazy_scan_prune-lpdead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1488-L1505)
- [vacuumlazy.c#noprune-lpdead](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1799-L1829)
- [vacuumlazy.c#phase-switches](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L370-L407)
- [vacuum.c#index_cleanup-reloption](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2155-L2179)
- [vacuum.c#truncate-reloption](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2190-L2201)
- [vacuumlazy.c#failsafe-disables-cleanup](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2320-L2326)
- [genam.h#IndexBulkDeleteResult](../../../raw/postgres-17/src/include/access/genam.h#L56-L84)
- [vacuumlazy.c#verbose-index-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L718-L732)
- [vacuumlazy.c#report-vacuum](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L577-L591)
- [pgstat_relation.c#pgstat_report_vacuum](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L227-L257)
- [system_views.sql#vacuum-counters](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L691-L698)
- [pgstat_relation.c#report_analyze-dead](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L328-L329)
- [analyze.c#report-analyze-dead](../../../raw/postgres-17/src/backend/commands/analyze.c#L688-L692)
- [vacuumlazy.c#relallvisible](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L556-L575)
- [analyze.c#relpages](../../../raw/postgres-17/src/backend/commands/analyze.c#L190-L197)
- [analyze.c#analyze-relstats](../../../raw/postgres-17/src/backend/commands/analyze.c#L623-L645)
- [vacuum.c#vacuum_is_permitted_for_relation](../../../raw/postgres-17/src/backend/commands/vacuum.c#L732-L758)
- [vacuum.c#relkind-skip](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2096-L2111)
- [vacuum.c#other-temp-silent](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2113-L2126)
- [vacuum.c#skipped-vacuum-skips-analyze](../../../raw/postgres-17/src/backend/commands/vacuum.c#L622-L634)
- [procarray.c#data-horizon-same-database](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1838-L1864)
- [procarray.c#slot-catalog-xmin](../../../raw/postgres-17/src/backend/storage/ipc/procarray.c#L1904-L1918)
- [walsender.c#affects-all-horizons](../../../raw/postgres-17/src/backend/replication/walsender.c#L318-L331)
- [walsender.c#feedback-xmin](../../../raw/postgres-17/src/backend/replication/walsender.c#L2740-L2749)
- [vacuumlazy.c#SKIP_PAGES_THRESHOLD](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L107-L111)
- [vacuumlazy.c#skip-range](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1138-L1143)
- [vacuumlazy.c#find_next_unskippable_block](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1203-L1245)
- [vacuumlazy.c#noprune-aggressive-wait](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1713-L1733)
- [bufmgr.c#ConditionalLockBufferForCleanup](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5456-L5508)
- [bufmgr.c#LockBufferForCleanup](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5301-L5330)
- [bufmgr.c:5411](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L5411)
- [vacuumlazy.c#lazy_check_wraparound_failsafe](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2299-L2347)
- [vacuumlazy.c#lazy_cleanup_all_indexes-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1064-L1066)
- [vacuumlazy.c#truncate-constants](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L64-L83)
- [vacuumlazy.c#should_attempt_truncation](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2529-L2544)
- [vacuumlazy.c#truncate-lock-retry](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2577-L2608)
- [vacuumlazy.c#truncate-rel-grew](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2610-L2627)
- [vacuumlazy.c#truncated-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2664-L2672)
- [vacuumlazy.c#truncate-suspend](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2716-L2735)
- [vacuum.c#ExecVacuum-defaults](../../../raw/postgres-17/src/backend/commands/vacuum.c#L153-L169)
- [vacuum.c#toast-relid](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2212-L2223)
- [vacuum.c#toast-recursion](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2282-L2300)
- [analyze.c#verbose-sample-line](../../../raw/postgres-17/src/backend/commands/analyze.c#L1296-L1306)
- [vacuum.c#datfrozenxid](../../../raw/postgres-17/src/backend/commands/vacuum.c#L700-L707)
- [guc_tables.c#max_prepared_transactions](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2579-L2585)
- [twophase.c#prepared-disabled](../../../raw/postgres-17/src/backend/access/transam/twophase.c#L371-L376)
- [pg_buffercache--1.2.sql#pinning_backends](../../../raw/postgres-17/contrib/pg_buffercache/pg_buffercache--1.2.sql#L17)
- [pgbuffercache.sgml#pinning_backends](../../../raw/postgres-17/doc/src/sgml/pgbuffercache.sgml#L166-L172)
- [pg_proc.dat#pg_lock_status](../../../raw/postgres-17/src/include/catalog/pg_proc.dat#L6308-L6313)
- [system_views.sql#pg_locks](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L401-L402)
- [bufmgr.c#SyncOneBuffer-pin](../../../raw/postgres-17/src/backend/storage/buffer/bufmgr.c#L3546-L3567)
- [utility.c#checkpoint-privilege](../../../raw/postgres-17/src/backend/tcop/utility.c#L942-L950)
- [reloptions.c#vacuum_index_cleanup](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L510-L520)
- [reloptions.c#vacuum_truncate](../../../raw/postgres-17/src/backend/access/common/reloptions.c#L151-L158)
- [vacuum.c#vacuum_xid_failsafe_check](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1252-L1298)
- [guc_tables.c#vacuum_failsafe_age](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2707-L2723)
- [guc_tables.c#autovacuum_freeze_max_age](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3378-L3398)
- [system_views.sql#pg_stat_activity](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L865-L891)
- [system_views.sql#pg_prepared_xacts](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L421-L426)
- [system_views.sql#pg_replication_slots](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1006-L1029)
- [system_views.sql#pg_stat_all_tables](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L670-L703)
- [miscinit.c#backend-type-names](../../../raw/postgres-17/src/backend/utils/init/miscinit.c#L277-L283)
- [ref/vacuum.sgml#options](../../../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L112-L323)
- [elog.c#should_output_to_client](../../../raw/postgres-17/src/backend/utils/error/elog.c#L248-L264)
- [common.c#NoticeProcessor](../../../raw/postgres-17/src/bin/psql/common.c#L269-L273)
- [startup.c#log_locus_callback](../../../raw/postgres-17/src/bin/psql/startup.c#L99-L111)
- [logging.c#locus-prefix](../../../raw/postgres-17/src/common/logging.c#L244-L261)
- [vacuumlazy.c#report-ereport](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L762-L763)
- [system_views.sql#pg_stat_progress_vacuum](../../../raw/postgres-17/src/backend/catalog/system_views.sql#L1209-L1227)
- [wait_event.c#wait-event-types](../../../raw/postgres-17/src/backend/utils/activity/wait_event.c#L391-L409)
- [wait_event_names.txt#VACUUM_TRUNCATE](../../../raw/postgres-17/src/backend/utils/activity/wait_event_names.txt#L181)
- [wait_event_names.txt#BUFFER_PIN](../../../raw/postgres-17/src/backend/utils/activity/wait_event_names.txt#L278-L280)
- [generate-wait_event_types.pl#camel-case](../../../raw/postgres-17/src/backend/utils/activity/generate-wait_event_types.pl#L112-L128)
- [vacuumlazy.c#verbose-vacuuming-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L469-L481)
- [vacuumlazy.c#finished-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L619-L652)
- [vacuumlazy.c#pages-line](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L653-L658)
- [vacuumlazy.c#keep-relfrozenxid](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L544-L554)
- [vacuumlazy.c#freeze-lines](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L674-L694)
- [vacuum.c#relfrozenxid-advance-only](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1489-L1519)
- [vacuum.c#cutoff-far-in-past](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1139-L1158)
- [vacuumparallel.c#launched-line](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L705-L716)
- [vacuumparallel.c#leader-participates](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L719-L726)
- [heapam_handler.c#analyze-dead-rows](../../../raw/postgres-17/src/backend/access/heap/heapam_handler.c#L1054-L1085)
- [analyze.c#two-stage-sample](../../../raw/postgres-17/src/backend/commands/analyze.c#L1136-L1137)
- [pruneheap.c#heap_page_prune_opt-minfree](../../../raw/postgres-17/src/backend/access/heap/pruneheap.c#L226-L264)
- [vacuumlazy.c#heap-pass-exclusive-lock](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2148-L2153)
- [pgstat_relation.c#analyze-count](../../../raw/postgres-17/src/backend/utils/activity/pgstat_relation.c#L339-L348)
- [vacuumlazy.c#prune-sets-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1523-L1556)
- [vacuumlazy.c#heap-pass-sets-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2260-L2281)
- [vacuumlazy.c#new-page-not-in-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1290-L1321)
- [vacuumlazy.c#empty-page-sets-vm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1323-L1381)
- [pageinspect--1.5.sql#page_header](../../../raw/postgres-17/contrib/pageinspect/pageinspect--1.5.sql#L22-L33)
- [rawpage.c#superuser](../../../raw/postgres-17/contrib/pageinspect/rawpage.c#L150-L153)
- [nbtree.c#btvacuumcleanup-skip](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L855-L874)
- [nbtpage.c#_bt_vacuum_needs_cleanup](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L178-L223)
- [nbtree.c#cleanup-lock-every-leaf](../../../raw/postgres-17/src/backend/access/nbtree/nbtree.c#L1202-L1208)
- [nbtpage.c#_bt_upgradelockbufcleanup](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L1105-L1120)
- [nbtpage.c#pages_newly_deleted](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2755-L2764)
- [nbtpage.c#safexid](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L2646)
- [nbtpage.c#pendingfsm-horizon](../../../raw/postgres-17/src/backend/access/nbtree/nbtpage.c#L3012-L3052)
- [vacuumlazy.c#aggressive-and-skipwithvm](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L449-L467)
- [vacuum.c#FreezeLimit](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1160-L1177)
- [guc_tables.c#vacuum_freeze_min_age](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L2667-L2674)
- [vacuumlazy.c#noprune-contract](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1633-L1651)
- [vacuum.c#aggressive-decision](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1199-L1241)
- [vacuum.c#FREEZE-ages](../../../raw/postgres-17/src/backend/commands/vacuum.c#L385-L402)
- [heapam.c#heap_tuple_should_freeze](../../../raw/postgres-17/src/backend/access/heap/heapam.c#L8247-L8284)
- [vacuumlazy.c#cleanup-lock-or-noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L950-L961)
- [vacuum.c#INDEX_CLEANUP-option](../../../raw/postgres-17/src/backend/commands/vacuum.c#L233-L248)
- [ref/vacuum.sgml#INDEX_CLEANUP](../../../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L186-L231)
- [autovacuum.c#unspecified-options](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2832-L2840)
- [vacuumlazy.c#dead-items-full](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L872-L910)
- [guc_tables.c#vacuum_cost_delay](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3865-L3873)
- [vacuum.c#vac_open_indexes-indisready](../../../raw/postgres-17/src/backend/commands/vacuum.c#L2345-L2357)
- [vacuum.c#datfrozenxid-bogus](../../../raw/postgres-17/src/backend/commands/vacuum.c#L1663-L1715)
- [autovacuum.c#anl_base_thresh](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3015-L3017)
- [autovacuum.c#anl_scale_factor](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3011-L3013)
- [autovacuum.c#doanalyze](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L3095)
- [nbtutils.c#INCLUDE-refusal](../../../raw/postgres-17/src/backend/access/nbtree/nbtutils.c#L5144-L5147)
- [fmgr.c#internal-alias](../../../raw/postgres-17/src/backend/utils/fmgr/fmgr.c#L216-L240)
- [vacuumlazy.c#heap_vacuum_rel-flow](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L497-L591)
- [vacuum-no-cleanup-lock.spec#header](../../../raw/postgres-17/src/test/isolation/specs/vacuum-no-cleanup-lock.spec#L1-L5)
- [vacuum-no-cleanup-lock.spec#pinholder](../../../raw/postgres-17/src/test/isolation/specs/vacuum-no-cleanup-lock.spec#L24-L35)
- [vacuum-no-cleanup-lock.spec#vacuumer](../../../raw/postgres-17/src/test/isolation/specs/vacuum-no-cleanup-lock.spec#L58-L69)
- [horizons.spec#vacuum-permanent](../../../raw/postgres-17/src/test/isolation/specs/horizons.spec#L143-L155)
- [horizons.spec#vacuum-temporary](../../../raw/postgres-17/src/test/isolation/specs/horizons.spec#L157-L169)
- [vacuum-skip-locked.spec#purpose](../../../raw/postgres-17/src/test/isolation/specs/vacuum-skip-locked.spec#L1-L4)
- [vacuum.sql#INDEX_CLEANUP](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L154-L189)
- [vacuum.sql#TRUNCATE](../../../raw/postgres-17/src/test/regress/sql/vacuum.sql#L191-L200)
- [vacuumlazy.c#lazy_scan_noprune](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L1653-L1849)
- [vacuumlazy.c#lazy_truncate_heap](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2546-L2673)
- [ref/vacuum.sgml#DISABLE_PAGE_SKIPPING](../../../raw/postgres-17/doc/src/sgml/ref/vacuum.sgml#L147-L164)
- [autovacuum.c#no-parallel](../../../raw/postgres-17/src/backend/postmaster/autovacuum.c#L2839-L2840)
- [vacuum.c#parallel-default](../../../raw/postgres-17/src/backend/commands/vacuum.c#L171-L172)
- [vacuum.c#PARALLEL-zero](../../../raw/postgres-17/src/backend/commands/vacuum.c#L277-L282)
- [vacuumlazy.c#parallel-gate](../../../raw/postgres-17/src/backend/access/heap/vacuumlazy.c#L2830-L2857)
- [vacuumparallel.c#parallel_vacuum_compute_workers](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L547-L601)
- [guc_tables.c#max_parallel_maintenance_workers](../../../raw/postgres-17/src/backend/utils/misc/guc_tables.c#L3409-L3417)
- [vacuumparallel.c#parallelism-disabled](../../../raw/postgres-17/src/backend/commands/vacuumparallel.c#L555-L560)

## Navigation

- [v17/index](../index.md)
- [wiki index](../../index.md)
- [versions](../../versions.md)
- [Wiki Glossary (unverified)](../../glossary.md)
