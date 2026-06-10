---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: claude-fable-5 2026-06-10T18:01:25Z
---

# Can ALTER TABLE ... ATTACH PARTITION Drop Indexes in PostgreSQL 17? (unverified)

## Question

In PostgreSQL 17, in what scenarios can indexes be dropped from a table during
ALTER TABLE ... ATTACH PARTITION on a declarative partitioned table? Add a
section with changes since PostgreSQL 12.

## Answer

**None. `ALTER TABLE ... ATTACH PARTITION` never drops an index from the table
being attached.** The index-handling subroutine is strictly *match-or-create*:
for each index on the partitioned parent it either **attaches** a compatible
existing index on the partition (re-parenting it, keeping its data) or
**creates** a new one. It has no code path that drops, replaces, or rebuilds an
existing index, and any extra or incompatible indexes on the partition are left
in place. This has been true since partitioned indexes were introduced, so the
no-drop invariant is the same in PostgreSQL 12 and 17; only the *matching*
rules and the companion `ALTER INDEX ... ATTACH PARTITION` validation were
refined, release by release (see
[What changed from PostgreSQL 12](#what-changed-from-postgresql-12)).

All of this happens in `AttachPartitionEnsureIndexes`, called by
`ATExecAttachPartition` after the new partition's catalog wiring is in place
([tablecmds.c#ATExecAttachPartition](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18700),
[tablecmds.c#AttachPartitionEnsureIndexes](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18812-L18986)).
The comment states the rule it enforces: "every partition must have an index
attached to each index on the partitioned table"
([tablecmds.c:18808-18810](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18808-L18810)).

### What ATTACH PARTITION does instead of dropping

For every **partitioned index** on the parent (plain, non-partitioned indexes on
the parent are ignored —
[tablecmds.c:18887-18891](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18887-L18891)),
ATTACH scans the partition's existing indexes and takes exactly one of two
actions:

| Situation on the partition being attached | What ATTACH does | Index dropped? |
|---|---|---|
| No compatible index exists | **Creates** a cloned index via `DefineIndex` ([tablecmds.c:18962-18975](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18962-L18975)) | No — one is *added* |
| A valid, unattached, same-definition index exists (and, if the parent index backs a constraint, a child constraint of the same type) | **Re-parents** it via `IndexSetParentIndex` — kept, not rebuilt ([tablecmds.c:18946-18954](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18946-L18954)) | No — reused/absorbed |
| An incompatible index exists (different columns/AM, partial, expression, or — since v15 — different `NULLS DISTINCT` treatment) | Ignores it; creates a new matching index | No — incompatible one is kept, now redundant |
| A matching index exists but is **invalid** | Skips it (v16+, back-patched; see below); creates a new valid index | No — invalid one is kept |
| Parent index backs a PRIMARY KEY but partition has only a plain UNIQUE | Doesn't match (v17+); creates a new index/constraint | No |
| Attaching a **foreign table** while the parent has any unique/PK index | Raises an error; nothing is attached ([tablecmds.c:18848-18868](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18848-L18868)) | No — operation refused |

The matching test is `CompareIndexInfo`
([tablecmds.c:18918-18923](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18918-L18923),
implemented in
[index.c#CompareIndexInfo](../../../raw/postgres-17/src/backend/catalog/index.c#L2510-L2627)),
gated by three guards: the candidate must not already be a partition of another
index (`relispartition`,
[tablecmds.c:18911-18912](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18911-L18912)),
must be valid (`indisvalid`,
[tablecmds.c:18914-18916](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18914-L18916)),
and, when the parent index backs a constraint, the partition must have a
constraint of the **same type**
([tablecmds.c:18931-18944](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18931-L18944)).
If all match, the only catalog change is re-parenting (`IndexSetParentIndex` plus
`ConstraintSetParentConstraint`) — no drop. If nothing matches, a new index is
*created*, which is the opposite of dropping.

The regression suite demonstrates each branch:

- Attaching a table with **no** index to a parent that has one **creates** the
  index on the partition
  ([indexing.sql:55-65](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L55-L65)).
- Attaching a table that **already has a matching** index does **not** create a
  duplicate; the existing index is attached
  ([indexing.sql:192-209](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L192-L209)).
- Attaching a table with four **incompatible** indexes (hash, partial,
  expression, `(a,a)`) keeps all four and merely **adds** one matching index —
  the `\d idxpart1` output after ATTACH still lists every original index plus the
  new `idxpart1_a_idx2`
  ([indexing.out:260-282](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L260-L282)).

### The closest things to a "drop" (and why they are not)

Three behaviors are sometimes mistaken for "ATTACH dropped my index":

1. **A matched index is *absorbed*, not removed.** After re-parenting, the
   partition's index has `relispartition = true` and can no longer be dropped on
   its own — `DROP INDEX` on it errors with *"cannot drop index ... because
   index ... requires it"*
   ([indexing.out:175-177](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L175-L177)).
   The index still exists and still holds its data; only its independent
   droppability changes.

2. **The redundant CHECK constraint you drop is not an index.** The documented
   fast-attach recipe is to add a `CHECK` constraint matching the partition
   bound *before* ATTACH (to skip the validation scan), then **manually** drop
   that now-redundant `CHECK` constraint afterward
   ([ddl.sgml:4290-4297](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4290-L4297)).
   That is a CHECK constraint and a manual step — ATTACH itself drops nothing.

3. **Later `DROP INDEX` on the parent cascades.** Dropping the parent
   partitioned index removes the attached child indexes too, and dropping the
   partition table removes its index
   ([indexing.out:180-197](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L180-L197)).
   That is a separate `DROP` command, not part of ATTACH.

### `ALTER INDEX ... ATTACH PARTITION` also never drops

The companion command that attaches one index to a partitioned index,
`ALTER INDEX ... ATTACH PARTITION` (`ATExecAttachPartitionIdx`), likewise only
re-parents, validates, or errors. If the partition already has a *different*
index attached for that slot it refuses with *"Another index is already
attached for partition"* rather than replacing one
([tablecmds.c:19919-19932](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19919-L19932),
[indexing.out:247-251](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L247-L251)),
and mismatched definitions error with *"The index definitions do not match"*
([tablecmds.c:19961-19972](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19961-L19972)).
Re-issuing the command for an index that is **already attached to that parent**
is accepted, and — in the pinned 17.x checkout — additionally attempts one
round of parent validation if the parent index is still invalid
([tablecmds.c:19902-19908](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19902-L19908),
[tablecmds.c:20007-20014](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20007-L20014));
see the [17.x stable updates](#postgresql-17x-stable-updates) below.

### Recommended workflow to avoid a costly index *build* during ATTACH

Because the "no match" branch builds a full index on the partition under the
attach's lock, the docs recommend pre-building it so ATTACH only re-parents:
`CREATE INDEX ON ONLY parent` (creates an invalid parent index), then
`CREATE INDEX CONCURRENTLY` on each partition, then
`ALTER INDEX parent_idx ATTACH PARTITION child_idx`; the parent index is marked
valid once all partitions are attached
([ddl.sgml:4318-4354](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4318-L4354)).
`CREATE INDEX CONCURRENTLY` directly on the partitioned table is still
rejected in v17, as it was in v12
([indexcmds.c:729](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L729)),
so this per-partition recipe remains the concurrent option. It avoids the
rebuild — it does not avoid any drop, because there is none.

### What changed from PostgreSQL 12

The **no-drop invariant is unchanged in every release from 12 through 17**:
`AttachPartitionEnsureIndexes` always ends in either `IndexSetParentIndex`
(attach) or `DefineIndex` (create), and a `tablecmds.c`-wide sweep of the
pinned checkout shows no `index_drop`/`performDeletion`/
`performMultipleDeletions` call inside `ATExecAttachPartition`,
`AttachPartitionEnsureIndexes`, or `ATExecAttachPartitionIdx` (the nearest
ones, [tablecmds.c:19537](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19537)
and [tablecmds.c:19789](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19789),
sit in the DETACH path). What did change, release by release, is *which*
existing partition index is accepted as a match and how the companion
`ALTER INDEX ... ATTACH PARTITION` validates the parent. None of these changes
introduces a drop.

Two bookkeeping notes for the per-version attribution below:

- The wiki's v12 baseline is the pinned 12.2 checkout (2020-02-10). Several
  fixes below carry a back-patch note in their commit message ("Backpatch-
  through: 11" and similar), so *current* 12.x minors also contain them even
  though the pinned 12.2 checkout predates them.
- This checkout carries no release tags, so each commit's first major release
  is bracketed by the version-stamp commits in its own history
  (`615cebc94b` "Stamp HEAD as 13devel" 2019-07-01, `d10b19e224` 14devel
  2020-06-07, `596b5af1d3` 15devel 2021-06-28, `d31d30973a` 16devel
  2022-06-30, `5bcc7e6dc8` 17devel 2023-06-29): a commit that is an ancestor
  of stamp N+1 first shipped in major N. Each commit below was confirmed an
  ancestor of the pinned HEAD.

The v12-side behavior is cited on the companion page
[Can ALTER TABLE ... ATTACH PARTITION Drop Indexes in PostgreSQL 12?](../../v12/questions/attach-partition-index-drops.md);
per the one-version-per-page citation rule, all source links here are to the
v17 checkout.

#### PostgreSQL 13

- **Mechanical attribute-map refactor (commit `e1551f96e6`, 2019-12-18).**
  `convert_tuples_by_name_map()` (returning `AttrNumber *` plus an explicit
  `natts`) was replaced by `build_attrmap_by_name()` (returning `AttrMap *`),
  so `CompareIndexInfo` and `generateClonedIndexStmt` no longer take a `natts`
  argument
  ([tablecmds.c:18895-18897](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18895-L18897)).
  No behavioral effect on matching or creation.
- **Error-code-only fix (commit `5b1c61e8b8`, 2020-05-28; back-patched to
  v11+).** The `ALTER INDEX ... ATTACH PARTITION` error *"cannot attach index
  ... not an index on any partition"* gained
  `ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE`
  ([tablecmds.c:19945-19953](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19945-L19953)).
  Message and behavior unchanged.

#### PostgreSQL 14

- **Cosmetic error wording (commit `e1ae40f381`, 2021-03-17).** The
  foreign-table refusal's detail changed from *"Table ... contains unique
  indexes"* to *"Partitioned table ... contains unique indexes"*
  ([tablecmds.c:18862](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18862)).
  The refusal itself exists in v12 too.
- **Adjacent DROP-path change (commit `afc7e0ad55`, 2020-09-01; back-patched
  to v11+).** `DROP INDEX CONCURRENTLY` on a partitioned index now raises the
  dedicated error *"cannot drop partitioned index ... concurrently"* instead
  of a confusing transaction-block error
  ([tablecmds.c:1589-1598](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1589-L1598)).
  This concerns the separate `DROP INDEX` command (look-alike 3 above), not
  ATTACH.

#### PostgreSQL 15

- **New matching dimension: `UNIQUE NULLS NOT DISTINCT` (commit `94aa7cc5f7`,
  2022-02-03).** v15 added the nulls-distinct option for unique indexes, and
  `CompareIndexInfo` now also requires both indexes to agree on it
  ([index.c:2521-2522](../../../raw/postgres-17/src/backend/catalog/index.c#L2521-L2522)).
  Effect on ATTACH: a partition unique index whose `NULLS DISTINCT` treatment
  differs from the parent's no longer matches; ATTACH creates a new index and
  keeps the old one — still no drop.
- **Adjacent DROP-path change (commit `7b6ec86532`, 2022-03-21; back-patched
  to v11+).** Dropping a partitioned index now locks all child tables before
  the child indexes, fixing a deadlock risk
  ([tablecmds.c:1600-1610](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1600-L1610)).
  Again a `DROP INDEX` change, not ATTACH.

#### PostgreSQL 16

- **Invalid partition indexes are skipped when matching (commit `fc55c7ff`,
  2023-06-28; back-patched to v11+).** Adds `if (!...->rd_index->indisvalid)
  continue;` to the candidate loop
  ([tablecmds.c:18914-18916](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18914-L18916)).
  Without it, an *invalid* partitioned child index (possible with multi-level
  partitioning and `CREATE INDEX ON ONLY`) could be chosen as a match and
  produce an inconsistent index tree. The commit is an ancestor of the 17devel
  stamp, so 16.0 is the first major release with it; the back-patch reaches
  v11+, so a *current* 12.x minor also has it, while the wiki's pinned 12.2
  checkout (2020-02-10) predates it. Effect: where original v12 would attach
  an invalid child index, v16+ leaves it alone and creates a new valid one —
  still no drop. Regression coverage:
  [indexing.sql:891-937](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L891-L937).
- **Mechanical progress-reporting parameter (commit `27f5c712b2`,
  2023-03-25).** `DefineIndex` gained a `total_parts` parameter for
  multi-level `CREATE INDEX` progress reporting; the attach path passes `-1`
  ([indexcmds.c#DefineIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L550),
  [tablecmds.c:18967-18974](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18967-L18974)).
  No effect on the match/create decision.

#### PostgreSQL 17 (17.0)

- **A PRIMARY KEY no longer matches a plain UNIQUE (commit `cee8db3f`,
  2024-04-15).** When the parent index backs a constraint, v17 additionally
  requires the child's constraint to be the **same type**
  ([tablecmds.c:18940-18944](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18940-L18944)):

  ```c
  /* Ensure they're both the same type of constraint */
  if (get_constraint_type(constraintOid) !=
      get_constraint_type(cldConstrOid))
      continue;
  ```

  Earlier versions only checked that *a* child constraint existed, so a UNIQUE
  key could incorrectly satisfy a PRIMARY KEY requirement. The commit sits in
  the 17devel cycle and carries no back-patch note, so it is v17-new. Effect:
  a mismatched-type child constraint is no longer reused; ATTACH creates a new
  index/constraint instead — still no drop.
- **`CompareIndexInfo` expression-vs-column fix (commit `9f71e10d65`,
  2023-09-28; back-patched to all supported branches).** The per-column loop
  now explicitly rejects a pair where one index has a plain column and the
  other an expression in the same position, instead of reading off the start
  of the attribute map
  ([index.c:2547-2560](../../../raw/postgres-17/src/backend/catalog/index.c#L2547-L2560)).
  A bogus match is now reliably rejected, so ATTACH creates a fresh index in
  that case — still no drop.
- **`ALTER INDEX ... ATTACH` validation uses a fresh catalog tuple (commit
  `38ea6aa90e`, 2023-07-14; back-patched to v11+).** `validatePartitionedIndex`
  now flips the parent's `indisvalid` on a `SearchSysCacheCopy1` copy of the
  `pg_index` tuple instead of a relcache copy, fixing *"attempted to update
  invisible tuple"* errors in multi-command transactions
  ([tablecmds.c:20104-20118](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20104-L20118)).
- **Locking before setting `relhassubclass` (commit `0cecc908e9`, 2024-06-27;
  back-patched to v12+).** `IndexSetParentIndex` — the re-parenting primitive
  ATTACH uses — now takes `ShareUpdateExclusiveLock` on the parent partitioned
  index before setting `relhassubclass`
  ([indexcmds.c:4384-4389](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4384-L4389)),
  so a concurrent `ALTER INDEX SET TABLESPACE` blocks instead of failing with
  *"tuple concurrently updated"*.
- **Mechanical refactors.** The catalogued-NOT-NULL work and its partial
  revert (commits `b0e96f3119`, 2023-08-25, and `6f8bb7c1e9`, 2024-05-13) left
  `AttachPartitionEnsureIndexes` with an unused `wqueue` parameter
  ([tablecmds.c:18812-18813](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18812-L18813))
  and rewrote the candidate-array loop with the v17 `foreach_oid` /
  `foreach_current_index` macros
  ([tablecmds.c:18834-18840](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18834-L18840)).
  Neither changes the match/create decision.

#### PostgreSQL 17.x stable updates

- **Re-attaching an attached index can validate an invalid parent (commit
  `becf6d2696`, 2026-04-22 on this stable branch; back-patched through v14).**
  `ALTER INDEX ... ATTACH PARTITION` for an index that is already attached to
  that parent — previously a silent no-op — now attempts one round of
  `validatePartitionedIndex` when the parent is still invalid
  ([tablecmds.c:19902-19908](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19902-L19908),
  [tablecmds.c:20007-20014](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20007-L20014)).
  This gives users a built-in way to bring an invalid parent index (for
  example, one polluted by attaching an invalid child) back to valid after
  fixing the child, e.g. with `REINDEX INDEX CONCURRENTLY`. Regression
  coverage:
  [indexing.sql:225-306](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L225-L306).
  It validates; it never drops.

Foreign-key and trigger cloning during ATTACH (`CloneRowTriggersToPartition`,
the FK-cloning calls right after `AttachPartitionEnsureIndexes`) also changed
substantially between v12 and v17 (e.g., commits `b0284bfb1d`, `e7936f8b3e`,
`344f9f5e2b`, `5914a22f6e`), but that path clones constraints and triggers
onto the partition; it does not drop indexes and is out of scope for this
question, as are the v14 `DETACH PARTITION CONCURRENTLY` and partitioned
`REINDEX` features, which act outside the attach path.

### Test coverage

`src/test/regress/sql/indexing.sql` (expected output in
`expected/indexing.out`) is the primary coverage:

- auto-create on attach
  ([indexing.sql:55-65](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L55-L65));
- reuse/attach an existing matching index, no duplicate
  ([indexing.sql:192-209](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L192-L209));
- incompatible indexes kept, matching one added
  ([indexing.out:260-282](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L260-L282));
- DROP semantics of an attached child index vs. the parent
  ([indexing.out:171-197](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L171-L197));
- `ALTER INDEX ... ATTACH PARTITION` error cases including dupe rejection
  ([indexing.out:218-251](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L218-L251));
- invalid partition indexes are not selected as matches, and an invalid child
  invalidates its ancestors
  ([indexing.sql:891-937](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L891-L937));
- re-attach of an already-attached index validates a still-invalid parent
  ([indexing.sql:225-306](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L225-L306)).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v17/index.md`, and recent
  `wiki/log.md` entries (navigation only).
- Pinned checkout `raw/postgres-17/` at commit
  `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- `ATExecAttachPartition`, `AttachPartitionEnsureIndexes`,
  `ATExecAttachPartitionIdx`, `refuseDupeIndexAttach`, and
  `validatePartitionedIndex` in `src/backend/commands/tablecmds.c`;
  `MergeConstraintsIntoExisting` (CHECK-only) confirmed not to touch indexes.
- `CompareIndexInfo` in `src/backend/catalog/index.c`; `IndexSetParentIndex`
  and the `DefineIndex` signature in `src/backend/commands/indexcmds.c`.
- A `tablecmds.c`-wide check that every `index_drop` / `performDeletion` /
  `performMultipleDeletions` call sits in the DROP, DETACH, DROP CONSTRAINT,
  trigger, or sequence paths, never inside the three attach functions.
- `doc/src/sgml/ddl.sgml` partitioning maintenance section.
- Tests `src/test/regress/sql/indexing.sql` and `expected/indexing.out`.
- v12 vs v17 deltas established from the `raw/postgres-17/` checkout's own
  commit history: a `git log -L` / file-log sweep of attach-related commits,
  each candidate inspected with `git show` and confirmed an ancestor of the
  pinned HEAD via `git merge-base --is-ancestor`. First-major attribution uses
  the "Stamp HEAD as NNdevel" commits as cycle brackets because this checkout
  carries no release tags; back-patch reach is taken from each commit message.
- A line-by-line diff of `AttachPartitionEnsureIndexes` between the pinned v12
  and v17 checkouts was used to confirm the per-version commit list is
  complete (every textual difference maps to a listed commit); citations
  remain v17-only.

## Evidence Map

| Claim | Source |
|---|---|
| Attach index handling is match-or-create; no drop path | [tablecmds.c:18812-18986](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18812-L18986) |
| Called from `ATExecAttachPartition` | [tablecmds.c:18700](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18700) |
| Only partitioned indexes on the parent drive matching | [tablecmds.c:18887-18891](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18887-L18891) |
| Match → re-parent via `IndexSetParentIndex` (kept) | [tablecmds.c:18946-18954](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18946-L18954) |
| No match → create cloned index via `DefineIndex` | [tablecmds.c:18962-18975](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18962-L18975) |
| Skip already-attached (`relispartition`) candidates | [tablecmds.c:18911-18912](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18911-L18912) |
| Skip invalid (`indisvalid`) candidates — v16+, back-patched to v11+ | [tablecmds.c:18914-18916](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18914-L18916), commit `fc55c7ff` (2023-06-28) |
| Constraint must be same type — v17 | [tablecmds.c:18940-18944](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18940-L18944), commit `cee8db3f` (2024-04-15) |
| Foreign-table attach refused if parent has unique/PK index | [tablecmds.c:18848-18868](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18848-L18868) |
| No deletion calls inside the attach functions | nearest are [tablecmds.c:19537](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19537), [tablecmds.c:19789](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19789) (DETACH path) |
| Incompatible indexes kept; matching one added | [indexing.out:260-282](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L260-L282) |
| Attached child index cannot be dropped on its own | [indexing.out:175-177](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L175-L177) |
| Drop redundant CHECK constraint is a manual, non-index step | [ddl.sgml:4290-4297](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4290-L4297) |
| Pre-build + `ALTER INDEX ATTACH` recommended workflow | [ddl.sgml:4318-4354](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4318-L4354) |
| CIC still rejected on partitioned tables in v17 | [indexcmds.c:729](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L729) |
| `ALTER INDEX ATTACH` refuses dupes / mismatches | [tablecmds.c:19919-19972](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19919-L19972) |
| Re-attach validates invalid parent — 17.x, back-patched to v14+ | [tablecmds.c:20007-20014](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20007-L20014), commit `becf6d2696` (2026-04-22) |
| v13 attmap refactor removed `natts` arguments | [tablecmds.c:18895-18897](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18895-L18897), commit `e1551f96e6` (2019-12-18) |
| v13 error-code-only fix in `ALTER INDEX ATTACH` | [tablecmds.c:19945-19953](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19945-L19953), commit `5b1c61e8b8` (2020-05-28) |
| v14 error-wording change in foreign-table refusal | [tablecmds.c:18862](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18862), commit `e1ae40f381` (2021-03-17) |
| v14 dedicated error for `DROP INDEX CONCURRENTLY` on partitioned index | [tablecmds.c:1589-1598](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1589-L1598), commit `afc7e0ad55` (2020-09-01) |
| v15 `NULLS NOT DISTINCT` must match | [index.c:2521-2522](../../../raw/postgres-17/src/backend/catalog/index.c#L2521-L2522), commit `94aa7cc5f7` (2022-02-03) |
| v15 DROP of partitioned index locks child tables first | [tablecmds.c:1600-1610](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L1600-L1610), commit `7b6ec86532` (2022-03-21) |
| v16 `DefineIndex` `total_parts` parameter | [indexcmds.c:540-550](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L550), commit `27f5c712b2` (2023-03-25) |
| v17 expression-vs-column matching fix | [index.c:2547-2560](../../../raw/postgres-17/src/backend/catalog/index.c#L2547-L2560), commit `9f71e10d65` (2023-09-28) |
| v17 `validatePartitionedIndex` syscache-copy fix | [tablecmds.c:20104-20118](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20104-L20118), commit `38ea6aa90e` (2023-07-14) |
| v17 lock before `relhassubclass` in `IndexSetParentIndex` | [indexcmds.c:4384-4389](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4384-L4389), commit `0cecc908e9` (2024-06-27) |
| v17 `wqueue`/`foreach_oid` refactors | [tablecmds.c:18812-18813](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18812-L18813), [tablecmds.c:18834-18840](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18834-L18840), commits `b0e96f3119` / `6f8bb7c1e9` |

## Open Questions

- Version attribution relies on the convention that each stable branch forks
  at the next "Stamp HEAD as NNdevel" commit. The checkout carries no release
  tags, so first-release claims (e.g., `fc55c7ff` first major-released in
  16.0; `cee8db3f` first in 17.0) are inferred from that bracketing plus the
  absence or presence of back-patch notes, not from `git tag --contains`.
- For back-patched fixes (`fc55c7ff`, `9f71e10d65`, `38ea6aa90e`,
  `0cecc908e9`, `becf6d2696`, `afc7e0ad55`, `7b6ec86532`, `5b1c61e8b8`), the
  back-patches are separate cherry-picks on the other stable branches; the
  exact minor release in which each reached `REL_12_STABLE` (or, for
  `becf6d2696` and `0cecc908e9`, the exact 17.x minor) was not pinned here,
  only that all post-date the wiki's pinned 12.2 checkout and are ancestors of
  the pinned 17.x HEAD.
- The v12 behavioral statements are anchored to the v17 checkout's own commit
  history and to a v12-vs-v17 function diff, per the one-version-per-page
  citation rule; the v12-side citations live on the companion page
  [v12: ATTACH PARTITION index drops](../../v12/questions/attach-partition-index-drops.md).

## Source References

- [tablecmds.c#ATExecAttachPartition](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18497-L18800)
- [tablecmds.c#AttachPartitionEnsureIndexes](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18812-L18986)
- [tablecmds.c#ATExecAttachPartitionIdx](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19863-L20022)
- [tablecmds.c#validatePartitionedIndex](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L20051-L20146)
- [index.c#CompareIndexInfo](../../../raw/postgres-17/src/backend/catalog/index.c#L2510-L2627)
- [indexcmds.c#DefineIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L550)
- [indexcmds.c#IndexSetParentIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L4304-L4429)
- [ddl.sgml#partition-maintenance](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4289-L4355)
- [indexing.sql#attach-partition](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L55-L306)
- [indexing.out#attach-partition](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L160-L282)

## Navigation

- [v17 index](../index.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
- [v17: CREATE INDEX CONCURRENTLY](create-index-concurrently.md)
- [v12: Can ALTER TABLE ... ATTACH PARTITION Drop Indexes?](../../v12/questions/attach-partition-index-drops.md)
