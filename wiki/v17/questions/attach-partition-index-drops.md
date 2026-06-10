---
type: question
version: 17
pinned_commit: 54eeefaedbee0385529f3edf321bb99e49232aaa
verified: false
verified_by_agent: not yet
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
behavior is the same in PostgreSQL 12 and 17; only the *matching* rules were
refined (see [What changed from PostgreSQL 12](#what-changed-from-postgresql-12)).

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
| An incompatible index exists (different columns/AM, partial, or expression) | Ignores it; creates a new matching index | No — incompatible one is kept, now redundant |
| A matching index exists but is **invalid** | Skips it (v17); creates a new valid index | No — invalid one is kept |
| Parent index backs a PRIMARY KEY but partition has only a plain UNIQUE | Doesn't match (v17); creates a new index/constraint | No |
| Attaching a **foreign table** while the parent has any unique/PK index | Raises an error; nothing is attached ([tablecmds.c:18848-18868](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18848-L18868)) | No — operation refused |

The matching test is `CompareIndexInfo`
([tablecmds.c:18918-18923](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18918-L18923)),
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
re-parents or errors. If the partition already has an index attached for that
slot it refuses with *"Another index is already attached for partition"* rather
than replacing one
([tablecmds.c:19919-19932](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19919-L19932),
[indexing.out:247-251](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L247-L251)),
and mismatched definitions error with *"The index definitions do not match"*
([tablecmds.c:19961-19972](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19961-L19972)).

### Recommended workflow to avoid a costly index *build* during ATTACH

Because the "no match" branch builds a full index on the partition under the
attach's lock, the docs recommend pre-building it so ATTACH only re-parents:
`CREATE INDEX ON ONLY parent` (creates an invalid parent index), then
`CREATE INDEX CONCURRENTLY` on each partition, then
`ALTER INDEX parent_idx ATTACH PARTITION child_idx`; the parent index is marked
valid once all partitions are attached
([ddl.sgml:4318-4354](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4318-L4354)).
This avoids the rebuild — it does not avoid any drop, because there is none.

### What changed from PostgreSQL 12

The **no-drop invariant is unchanged**: PostgreSQL 12's
`AttachPartitionEnsureIndexes` is also strictly match-or-create, ending in either
`IndexSetParentIndex` (attach) or `DefineIndex` (create), with no drop path. Two
*matching* refinements landed after the pinned v12.2 checkout, plus mechanical
refactors. None of them introduces an index drop; they only change whether an
existing partition index is reused or a fresh one is created.

**1. Invalid partition indexes are skipped when matching (commit `fc55c7ff`,
2023-06-28; back-patched to v11+).** v17 adds `if (!...->rd_index->indisvalid)
continue;` to the candidate loop
([tablecmds.c:18914-18916](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18914-L18916)).
The original v12 logic had no such guard, so an *invalid* partitioned child
index (which exists, for example, with multi-level partitioning and
`CREATE INDEX ON ONLY`) could be chosen as a match and produce an inconsistent
index tree. The commit message — verifiable in this checkout's history — calls
it a bug fix and states "backpatch all the way down to v11," so a *current* 12.x
minor also has it; the wiki's pinned 12.2 checkout (2020-02-10) predates the fix.
Effect: where original v12 would attach an invalid child index, v17 leaves it
alone and creates a new valid one — still no drop.

**2. A PRIMARY KEY no longer matches a plain UNIQUE (commit `cee8db3f`,
2024-04-15; first shipped in v17).** When the parent index backs a constraint,
v17 additionally requires the child's constraint to be the **same type**
([tablecmds.c:18940-18944](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18940-L18944)):

```c
/* Ensure they're both the same type of constraint */
if (get_constraint_type(constraintOid) !=
    get_constraint_type(cldConstrOid))
    continue;
```

Original v12 only checked that *a* child constraint existed, so a UNIQUE key
could incorrectly satisfy a PRIMARY KEY requirement. This commit carries no
back-patch note and `git tag --contains` shows it first in `REL_17_0`, so it is
effectively v17-new. Effect: a mismatched-type child constraint is no longer
reused; ATTACH creates a new index/constraint instead — still no drop.

**3. Mechanical refactors (no behavioral effect).** The attribute map switched
from `convert_tuples_by_name_map()` (returning `AttrNumber *`, with an explicit
`natts`) to `build_attrmap_by_name(..., false)` (returning `AttrMap *`), so
`CompareIndexInfo` and `generateClonedIndexStmt` no longer take a `natts`
argument
([tablecmds.c:18895-18897](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18895-L18897));
`DefineIndex` gained a `total_parts` progress-reporting parameter, passed `-1` by
the attach path
([indexcmds.c#DefineIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L550),
[tablecmds.c:18967-18974](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18967-L18974));
and the candidate-array build loop now uses the `foreach_oid` /
`foreach_current_index` macros
([tablecmds.c:18834-18840](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18834-L18840)).
These change neither the match/create decision nor the absence of a drop.

Foreign-key and trigger cloning during ATTACH (`CloneRowTriggersToPartition`,
the FK-cloning calls right after `AttachPartitionEnsureIndexes`) did change
substantially between v12 and v17, but that path clones constraints and triggers
onto the partition; it does not drop indexes and is out of scope for this
question.

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
  ([indexing.out:218-251](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L218-L251)).

## Context Reviewed

- `wiki/versions.md`, `wiki/index.md`, `wiki/v17/index.md`, and recent
  `wiki/log.md` entries (navigation only).
- Pinned checkout `raw/postgres-17/` at commit
  `54eeefaedbee0385529f3edf321bb99e49232aaa`.
- `ATExecAttachPartition`, `AttachPartitionEnsureIndexes`, and
  `ATExecAttachPartitionIdx` in `src/backend/commands/tablecmds.c`;
  `MergeConstraintsIntoExisting` (CHECK-only) confirmed not to touch indexes.
- A `tablecmds.c`-wide check that every `index_drop` / `performDeletion` call
  sits in the DETACH, DROP CONSTRAINT, or trigger paths, never the attach path.
- `DefineIndex` signature in `src/backend/commands/indexcmds.c`.
- `doc/src/sgml/ddl.sgml` partitioning maintenance section.
- Tests `src/test/regress/sql/indexing.sql` and `expected/indexing.out`.
- v12 vs v17 deltas established from the `raw/postgres-17/` checkout's own commit
  history (`fc55c7ff`, `cee8db3f`), each verified as an ancestor of the pinned
  HEAD via `git merge-base --is-ancestor`, with release reach checked via
  `git tag --contains`.

## Evidence Map

| Claim | Source |
|---|---|
| Attach index handling is match-or-create; no drop path | [tablecmds.c:18812-18986](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18812-L18986) |
| Called from `ATExecAttachPartition` | [tablecmds.c:18700](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18700) |
| Only partitioned indexes on the parent drive matching | [tablecmds.c:18887-18891](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18887-L18891) |
| Match → re-parent via `IndexSetParentIndex` (kept) | [tablecmds.c:18946-18954](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18946-L18954) |
| No match → create cloned index via `DefineIndex` | [tablecmds.c:18962-18975](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18962-L18975) |
| Skip already-attached (`relispartition`) candidates | [tablecmds.c:18911-18912](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18911-L18912) |
| Skip invalid (`indisvalid`) candidates — v17 | [tablecmds.c:18914-18916](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18914-L18916) |
| Constraint must be same type — v17 | [tablecmds.c:18940-18944](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18940-L18944) |
| Foreign-table attach refused if parent has unique/PK index | [tablecmds.c:18848-18868](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18848-L18868) |
| Incompatible indexes kept; matching one added | [indexing.out:260-282](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L260-L282) |
| Attached child index cannot be dropped on its own | [indexing.out:175-177](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L175-L177) |
| Drop redundant CHECK constraint is a manual, non-index step | [ddl.sgml:4290-4297](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4290-L4297) |
| Pre-build + `ALTER INDEX ATTACH` recommended workflow | [ddl.sgml:4318-4354](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4318-L4354) |
| `ALTER INDEX ATTACH` refuses dupes / mismatches | [tablecmds.c:19919-19972](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19919-L19972) |
| Invalid-index skip added post-v12, back-patched to v11+ | commit `fc55c7ff` (2023-06-28) |
| PK-vs-UNIQUE constraint-type check first in v17 | commit `cee8db3f` (2024-04-15) |
| `DefineIndex` `total_parts` parameter | [indexcmds.c:540-550](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L550) |

## Open Questions

- The v12 behavioral statements in
  [What changed from PostgreSQL 12](#what-changed-from-postgresql-12) are anchored
  to the v17 checkout's own commit history (`git show`, `git tag --contains`,
  `git merge-base --is-ancestor`), not re-cited against the PostgreSQL 12 source
  checkout, per the one-version-per-page citation rule. The companion v12 page
  set ([v12 questions](../../v12/index.md)) does not yet include an
  ATTACH-PARTITION index page.
- `fc55c7ff` was back-patched as separate cherry-picks to the stable branches;
  the exact minor release in which it reached `REL_12_STABLE` was not pinned
  here, only that it post-dates the pinned 12.2 checkout.
- The `convert_tuples_by_name_map` → `build_attrmap_by_name` rename is described
  as a mechanical refactor; the precise introducing commit was not traced, since
  it has no effect on the drop/create behavior in scope.

## Source References

- [tablecmds.c#ATExecAttachPartition](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18497-L18800)
- [tablecmds.c#AttachPartitionEnsureIndexes](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L18812-L18986)
- [tablecmds.c#ATExecAttachPartitionIdx](../../../raw/postgres-17/src/backend/commands/tablecmds.c#L19863-L19972)
- [indexcmds.c#DefineIndex](../../../raw/postgres-17/src/backend/commands/indexcmds.c#L540-L550)
- [ddl.sgml#partition-maintenance](../../../raw/postgres-17/doc/src/sgml/ddl.sgml#L4289-L4355)
- [indexing.sql#attach-partition](../../../raw/postgres-17/src/test/regress/sql/indexing.sql#L55-L212)
- [indexing.out#attach-partition](../../../raw/postgres-17/src/test/regress/expected/indexing.out#L160-L282)

## Navigation

- [v17 index](../index.md)
- [versions](../../versions.md)
- [wiki index](../../index.md)
- [v17: CREATE INDEX CONCURRENTLY](create-index-concurrently.md)
