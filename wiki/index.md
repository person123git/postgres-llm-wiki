# Wiki Index

This is the global catalog for the PostgreSQL engine wiki.

## Entry Points

- [[versions]] - PostgreSQL version index and source pin manifest.
- [[overview]] - Cross-version architecture overview.
- [[log]] - Chronological activity log.


## Version-Specific Pages

### PostgreSQL 18

- [[v18/index]] - Primary version landing page. Source checkout pinned to `REL_18_STABLE` commit `6cb307251c5c6261286c1566496920976640108e`.
- [[v18/questions/avg-leaf-density-during-vacuum|Computing and Storing avg_leaf_density During (Auto)VACUUM of a B-Tree Index (unverified)]] - Reviewed design and evidence-backed cons for computing `pgstatindex`-style `avg_leaf_density` during B-tree VACUUM, including metapage/statistics storage, scan-skip caveats, and empty-page deletion accuracy gaps.
- [[v18/questions/custom-cumulative-statistics|How Custom Cumulative Statistics Work in PostgreSQL 18 (unverified)]] - Explains custom cumulative statistics registration, variable and fixed custom stat storage, flushing, snapshots, reset/drop behavior, and clean-shutdown persistence in PostgreSQL 18.
- [[v18/questions/extension-hooks-vacuum-autovacuum|Extension Hooks for VACUUM and Autovacuum in PostgreSQL 18 (unverified)]] - Catalogs every extension point on the VACUUM/ANALYZE and autovacuum paths: in-process hook variables, table and index AM and FDW callbacks, and adjacent surfaces (background workers, custom cumulative statistics), with manual-vs-autovacuum coverage.



### PostgreSQL 17.10

- [[v17/index]] - Active version landing page. Source checkout pinned to `REL_17_STABLE` commit `54eeefaedbee0385529f3edf321bb99e49232aaa`.



### PostgreSQL 12.2

- [[v12/index]] - Legacy version landing page. Source checkout pinned to `REL_12_STABLE` commit `45b88269a353ad93744772791feb6d01bc7e1e42`.
- [[v12/questions/fk-join-optimization-two-tables|Foreign-Key Join Optimization for Two-Table Joins (unverified)]] - How the v12 planner uses foreign-key constraints when joining two tables.



## Maintenance Tooling

- `scripts/recent_log` - recent wiki activity.
- `scripts/wiki_lint` - wiki health checks.

## Maintenance Notes

- Update this page whenever a wiki page is created or substantially changed.
- Keep version-specific entries tagged with their PostgreSQL major version.
- Prefer links to version landing pages, such as `vNN/index`, once versions exist.
