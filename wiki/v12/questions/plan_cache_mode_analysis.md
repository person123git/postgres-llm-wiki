---
type: question
version: 12
pinned_commit: 45b88269a353ad93744772791feb6d01bc7e1e42
verified: false
verified_by_agent: not yet
---

# plan_cache_mode_analysis (unverified)

## Question
provide a comprehensive plan_cache_mode analysis

## Answer

The `plan_cache_mode` configuration parameter determines whether PostgreSQL uses a generic plan or a custom plan for prepared statements.

### GUC Configuration
- **Context**: `PGC_USERSET` (can be set by the user for the session or a transaction) [guc.c#plan_cache_mode](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4504).
- **Default**: `auto`.

### Available Modes

| Mode | Description |
| --- | --- |
| `auto` (Default) | PostgreSQL decides between a generic and a custom plan based on estimated costs. |
| `force_generic_plan` | Forces the use of a generic plan, regardless of cost. |
| `force_custom_plan` | Forces the use of a custom plan, regardless of cost. |

### Internal Decision Logic

The decision process is handled in the `choose_custom_plan` function [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016).

#### Decision Hierarchy
PostgreSQL evaluates the following conditions in order to decide if a custom plan should be used (returns `true` for custom, `false` for generic):

1. **One-shot Plans**: Always use custom plans [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1021).
2. **No Parameters**: Always use generic plans if `boundParams` is `NULL` [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1025).
3. **Transaction Statements**: Always use generic plans for transaction control statements [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1028).
4. **Force Generic**: If `plan_cache_mode` is `force_generic_plan`, use a generic plan [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1032).
5. **Force Custom**: If `plan_cache_mode` is `force_custom_plan`, use a custom plan [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1034).
6. **Cursor Options**:
    - `CURSOR_OPT_GENERIC_PLAN` forces a generic plan [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1038).
    - `CURSOR_OPT_CUSTOM_PLAN` forces a custom plan [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1040).
7. **Auto Heuristic**:
    - **Initial Sampling**: The first 5 executions always use custom plans to gather cost data [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1043).
    - **Cost Comparison**: After 5 custom plans, it compares the estimated cost of a generic plan (`generic_cost`) with the average cost of the custom plans (`avg_custom_cost`) [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1047).
    - **Preference**: If `generic_cost < avg_custom_cost`, a generic plan is selected [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1059).

#### The "Correction" Step
In `GetCachedPlan`, if the `auto` heuristic chooses a generic plan, the engine first builds the generic plan. It then re-evaluates `choose_custom_plan` using the now-known `generic_cost` [plancache.c#GetCachedPlan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1200). If the generic plan turns out to be more expensive than the average custom plan, the engine reverts and generates a custom plan instead to avoid executing a suboptimal plan.

### Planning Cost Estimation

The estimated cost of a plan is calculated by `cached_plan_cost` [plancache.c#cached_plan_cost](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1066).

When evaluating custom plans, PostgreSQL adds a crude estimate of the planning effort to the execution cost:
`result += 1000.0 * cpu_operator_cost * (nrelations + 1);` [plancache.c#cached_plan_cost](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1112).

This ensures that a generic plan is only preferred if its execution cost is lower than the sum of the custom plan's execution cost and its planning overhead.

### Summary Table of Behavior

| Feature | `auto` | `force_generic_plan` | `force_custom_plan` |
| --- | --- | --- | --- |
| **Decision Basis** | Heuristic (Cost-based) | Always Generic | Always Custom |
| **Initial Threshold** | First 5 plans are custom | N/A | N/A |
| **Complexity Awareness** | Yes (includes planning cost estimate) | No | No |
| **Reverts to Custom?** | Yes (if generic cost > avg custom) | No | N/A |

## Context Reviewed
- `src/include/utils/plancache.h`
- `src/backend/utils/cache/plancache.c`
- `src/backend/utils/misc/guc.c`

## Source References
- [plancache.c#choose_custom_plan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1016)
- [plancache.c#GetCachedPlan](../../../raw/postgres-12/src/backend/utils/cache/plancache.c#L1137)
- [guc.c#plan_cache_mode](../../../raw/postgres-12/src/backend/utils/misc/guc.c#L4504)

## Open Questions
- Pending deep inquiry.
