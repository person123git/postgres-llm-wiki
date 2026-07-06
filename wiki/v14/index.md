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
