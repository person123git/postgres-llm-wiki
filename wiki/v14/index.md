# PostgreSQL 14.23

## Source Pin

- Branch: `REL_14_STABLE`
- Commit: `5c00f4e2e3bcee6931ae93429d53f7c2a4f46156`
- Status: `active`
- Source path: `raw/postgres-14/`
- Added: 2026-06-30

## Coverage

Behavioral claims cite the matching pinned checkout under `raw/postgres-14/`.

- [PostgreSQL 14 Codebase Navigation Guide (unverified)](codebase-navigation-guide.md) - Mandatory root-level question-style map for navigating the pinned v14 source tree: layout, SQL statement flow, utility dispatch, generated/catalog artifacts, key structs, contrib boundaries, tests, and docs.
- [Row-Level Security (RLS) in PostgreSQL 14: Implementation, Scalability and Performance, and Settings (unverified)](questions/row-level-security-rls.md) - Explains RLS as a rewrite-time feature (`pg_policy` storage, relcache `rd_rsdesc`, `check_enable_rls` bypass logic, `get_row_security_policies` producing security-barrier `USING` quals and `WITH CHECK` options, permissive-OR/restrictive-AND combination and default-deny, command/role matching, the planner `security_level`/leakproof qual ordering, executor `ExecWithCheckOptions`, plan-cache `dependsOnRLS` re-planning, partition/inheritance handling, and COPY/CTAS/RI/error-leak boundaries); the source-evident scalability and performance issues (per-row evaluation, leakproof ordering blocking index use, policy subqueries, qual-tree growth with many policies, plan-cache churn on role/`row_security` change, FK bulk-check downgrade, partition fan-out); a dedicated RLS-and-the-plan-cache section (RLS-aware plan caching keyed on role + `row_security`, the role-independent relcache `rd_rsdesc` policy cache, the scenarios where caching does not help, and the no-cache re-analyze/rewrite/re-plan cost); and all related settings (the `row_security` GUC, the `ALTER TABLE` table flags, the `BYPASSRLS` role attribute, the `CREATE`/`ALTER POLICY` options, and the two extension hooks).
- [How MultiXact Works in PostgreSQL 14, and How Foreign Keys and Other Operations Degrade Performance When the Local Cache Spills to Secondary Storage (unverified)](questions/multixact-foreign-keys-cache-spill.md) - Explains MultiXact as the shared-row-lock mechanism (a `MultiXactId` in `xmax` standing for a set of `(xid, MultiXactStatus)` members, the offsets/members SLRUs backed by `pg_multixact/`, the `MultiXactStateData`/per-backend oldest arrays, the create path `MultiXactIdCreate`/`Expand`/`CreateFromMembers` -> `GetNewMultiXactId`/`RecordNewMultiXact`, and the read path `GetMultiXactIdMembers`); the backend-local 256-entry per-transaction LRU cache (`mXactCacheEnt`/`MAX_CACHE_ENTRIES`, `MXactContext`, `mXactCachePut` tail eviction) and how a lookup miss falls back through the tiny 8-offset/16-member SLRU pool to the on-disk `pg_multixact/` files (`SlruPhysicalReadPage` -> `SLRURead`); how foreign keys create MultiXacts (RI `FOR KEY SHARE` -> `heap_lock_tuple` -> `compute_new_xmax_infomask`) plus other operations (explicit row locks, UPDATE/DELETE of locked rows, VACUUM freezing); the three degradation paths (cache spill to secondary storage, exclusive-locked SLRU buffer contention on `MultiXactOffsetSLRULock`/`MultiXactMemberSLRULock`, and member-space growth driving aggressive vacuum and `multixact "members" limit exceeded`); observability via `pg_stat_slru`/wait events/`mxid_age`; and the vacuum GUCs with apply scopes.
