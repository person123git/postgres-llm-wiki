---
type: question
version: 14
pinned_commit: 5c00f4e2e3bcee6931ae93429d53f7c2a4f46156
verified: false
verified_by_agent: not yet
---

# Performance Implications of Functions and Procedures in a WHERE Clause in PostgreSQL 14, and How to Minimize the Overhead (unverified)

## Contents

- [Question](#question)
- [Answer](#answer)
  - [Short answer](#short-answer)
  - [Procedures cannot appear in a WHERE clause](#procedures-cannot-appear-in-a-where-clause)
  - [Where and how often a WHERE-clause function runs](#where-and-how-often-a-where-clause-function-runs)
  - [How volatility changes what the executor and planner can do](#how-volatility-changes-what-the-executor-and-planner-can-do)
  - [How the planner costs and estimates a function qual](#how-the-planner-costs-and-estimates-a-function-qual)
  - [Functions on a column defeat a plain index](#functions-on-a-column-defeat-a-plain-index)
  - [Leakproofness, security barriers, and qual ordering](#leakproofness-security-barriers-and-qual-ordering)
  - [Parallel safety](#parallel-safety)
  - [Minimizing the overhead](#minimizing-the-overhead)
- [Context Reviewed](#context-reviewed)
- [Evidence Map](#evidence-map)
- [Open Questions](#open-questions)
- [Source References](#source-references)
- [Navigation](#navigation)

## Question

In a statement execution that has functions or procedures used in the WHERE clause, what are the performance implications, and what can be done to minimize the overhead of calling functions or procedures in the statement's WHERE clause?

> Prompt note: this is a grammar-corrected restatement of the original request, applied with the asker's approval per the wiki's prompt-hygiene rule.

## Answer

### Short answer

Two facts frame everything below.

1. **Procedures cannot appear in a WHERE clause at all.** A routine created with `CREATE PROCEDURE` has `pg_proc.prokind = 'p'`, and the parser rejects it in any expression context with `ERRCODE_WRONG_OBJECT_TYPE` "`X is a procedure`" / hint "`To call a procedure, use CALL.`" [parse_func.c#ParseFuncOrColumn](../../../raw/postgres-14/src/backend/parser/parse_func.c#L293-L302). Procedures run only through the `CALL` utility statement, entirely outside the query planner and expression evaluator [utility.c#T_CallStmt](../../../raw/postgres-14/src/backend/tcop/utility.c#L849-L851). So the performance discussion is about **functions**.

2. **A function in a WHERE clause is a per-row cost.** The scan node loops over tuples and calls `ExecQual` on each one [execScan.c#ExecScan](../../../raw/postgres-14/src/backend/executor/execScan.c#L202-L234); each function in the qual is dispatched through a function pointer for every tuple that reaches it [execExprInterp.c#EEOP_FUNCEXPR](../../../raw/postgres-14/src/backend/executor/execExprInterp.c#L727-L738). The dominant performance implications are therefore: (a) the function is invoked once per candidate row, (b) its result is not estimated well by the planner, so row counts and plan choice suffer, and (c) it can block index scans, parallelism, and qual push-down.

The levers that reduce the overhead are, in rough order of impact: label the function's **volatility** correctly (`IMMUTABLE`/`STABLE`, not the default `VOLATILE`); provide an **expression index** so the function is precomputed and stored; wrap an **uncorrelated** call in a scalar sub-SELECT so it runs once per execution; let simple SQL functions be **inlined**; give the planner accurate `COST`/`ROWS` and a `SUPPORT` function; mark the function `LEAKPROOF` and `PARALLEL SAFE` when that is truthful; and arrange cheaper, more selective, indexable quals to filter rows before the function runs.

### Procedures cannot appear in a WHERE clause

`CREATE PROCEDURE` stores `pg_proc.prokind = 'p'` (`PROKIND_PROCEDURE`) [pg_proc.h#PROKIND](../../../raw/postgres-14/src/include/catalog/pg_proc.h#L150-L153). Every expression context (SELECT list, WHERE, JOIN/ON, etc.) resolves routine names through `ParseFuncOrColumn` with its `proc_call` argument set to `false`, and that function raises an error the moment the resolved routine is a procedure [parse_func.c#ParseFuncOrColumn](../../../raw/postgres-14/src/backend/parser/parse_func.c#L293-L302). The only caller that passes `proc_call = true` is `transformCallStmt`, reached solely from the grammar's `CallStmt` production [analyze.c#transformCallStmt](../../../raw/postgres-14/src/backend/parser/analyze.c#L3021-L3027) [gram.y#CallStmt](../../../raw/postgres-14/src/backend/parser/gram.y#L1038-L1043). `CALL` is a utility statement executed by `standard_ProcessUtility` -> `ExecuteCallStmt` [utility.c#T_CallStmt](../../../raw/postgres-14/src/backend/tcop/utility.c#L849-L851); it never becomes a `FuncExpr` in a plan tree. The documentation states the same rule: "a procedure is called in isolation using the `CALL` command" [xfunc.sgml#procedures](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml#L105-L111), and the regression suite proves it — `SELECT ptest1('x')` errors with "`ptest1(unknown) is a procedure`" [create_procedure.out:52](../../../raw/postgres-14/src/test/regress/expected/create_procedure.out#L52-L56).

Practical consequence: if you have logic packaged as a procedure that you want to filter rows with, you must expose it as a **function** (with an appropriate return type and volatility) to use it in a WHERE clause. The rest of this page treats that function case.

### Where and how often a WHERE-clause function runs

WHERE-clause conditions become the scan/join node's *qual*. `ExecScan` fetches tuples in a loop and, for each one, evaluates the qual with `ExecQual`; a tuple that fails is discarded and the loop fetches the next [execScan.c#ExecScan](../../../raw/postgres-14/src/backend/executor/execScan.c#L202-L234). `ExecQual` runs the compiled expression program [executor.h#ExecQual](../../../raw/postgres-14/src/include/executor/executor.h#L400-L420), and a function call in that program is the `EEOP_FUNCEXPR` step, which calls the function through `fcinfo`'s stored address on every evaluation [execExprInterp.c#EEOP_FUNCEXPR](../../../raw/postgres-14/src/backend/executor/execExprInterp.c#L727-L738) using the standard fmgr dispatch [fmgr.h#FunctionCallInvoke](../../../raw/postgres-14/src/include/fmgr.h#L172).

So the base cost model is: **calls = number of rows that reach the qual node**. Two things make this worse than it sounds:

- The number of rows is often *not* reduced first, because a function-based qual usually cannot be turned into an index condition (see [Functions on a column defeat a plain index](#functions-on-a-column-defeat-a-plain-index)), so the function runs on every row of a sequential scan.
- Each call carries fmgr overhead, and for non-C languages substantially more: the `CREATE FUNCTION` `COST` default is "1 unit ... for C-language and internal functions, and 100 units for functions in all other languages" [create_function.sgml#COST](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L462-L476), and SQL functions in particular carry "the rather high per-call overhead of SQL functions" unless the planner can inline them [clauses.c#inline_function](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4350-L4380).

### How volatility changes what the executor and planner can do

A function's `pg_proc.provolatile` category is a promise that controls which optimizations are legal [pg_proc.h#PROVOLATILE](../../../raw/postgres-14/src/include/catalog/pg_proc.h#L155-L165). `CREATE FUNCTION` defaults to `VOLATILE` [xfunc.sgml#volatility](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml#L1626-L1666), which is the worst case for a WHERE clause.

- **`VOLATILE`** — "A query using a volatile function will re-evaluate the function at every row where its value is needed" [xfunc.sgml#volatile](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml#L1635-L1639). It also blocks the index-condition and constant-folding optimizations below.
- **`STABLE`** — "guaranteed to return the same results given the same arguments for all rows within a single statement ... it is safe to use an expression containing such a function in an index scan condition. (Since an index scan will evaluate the comparison value only once, not once at each row, it is not valid to use a `VOLATILE` function in an index scan condition.)" [xfunc.sgml#stable](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml#L1644-L1652). Two caveats matter: (1) `STABLE` does **not** memoize a plain filter — `WHERE stable_fn(col) = 'x'` where `stable_fn(col)` depends on the row is still evaluated per row; the once-per-scan benefit applies to the *comparison value* side of an indexable operator (a runtime key, below). (2) `STABLE` is folded to a constant only in *estimation* mode, never for actual execution [clauses.c#evaluate_function](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4308-L4348).
- **`IMMUTABLE`** — "allows the optimizer to pre-evaluate the function when a query calls it with constant arguments" [xfunc.sgml#immutable](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml#L1657-L1663). Constant folding happens in `eval_const_expressions` at plan time [clauses.c#eval_const_expressions](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L2123-L2159): a function call is replaced by a `Const` only if all its inputs are constants *and* it is `IMMUTABLE` (or merely `STABLE` when the planner is only estimating) [clauses.c#evaluate_function](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4308-L4348). So `WHERE x = 2 + 2` becomes `WHERE x = 4`; but `WHERE immutable_fn(col)` has a non-constant input and is *not* folded — it still runs per row (though it can now feed an expression index).

The runtime "evaluate the comparison value once per scan" behavior is real and lives in the index-scan executor: on each (re)scan, `ExecReScanIndexScan` recomputes *all* runtime keys once [nodeIndexscan.c#ExecReScanIndexScan](../../../raw/postgres-14/src/backend/executor/nodeIndexscan.c#L553-L593) via `ExecIndexEvalRuntimeKeys` [nodeIndexscan.c#ExecIndexEvalRuntimeKeys](../../../raw/postgres-14/src/backend/executor/nodeIndexscan.c#L600-L652); the per-row `IndexNext` loop never re-evaluates them. The planner only allows a non-index operand to become such a key if it contains no volatile function [indxpath.c#match_opclause_to_indexcol](../../../raw/postgres-14/src/backend/optimizer/path/indxpath.c#L2504-L2535).

### How the planner costs and estimates a function qual

Poor estimates are as damaging as the per-row cost, because they make the planner pick the wrong plan (wrong join order, seq scan vs index, no parallelism).

**Cost.** For each function/operator node the planner "charge[s] the estimated execution cost given by `pg_proc.procost` (remember to multiply this by `cpu_operator_cost`)" [costsize.c#cost_qual_eval_walker](../../../raw/postgres-14/src/backend/optimizer/path/costsize.c#L4412-L4438). The charge is applied in `add_function_cost`, which uses `procost * cpu_operator_cost` unless a support function overrides it [plancat.c#add_function_cost](../../../raw/postgres-14/src/backend/optimizer/util/plancat.c#L2006-L2050). `procost` defaults to 1 [pg_proc.h#procost](../../../raw/postgres-14/src/include/catalog/pg_proc.h#L46-L56). If a genuinely expensive function is left at the default cost, the planner has no reason to avoid calling it more than necessary [create_function.sgml#COST](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L462-L476). (The per-clause cost is cached on the `RestrictInfo` so it is computed once per clause, not per row [costsize.c#eval-cost-cache](../../../raw/postgres-14/src/backend/optimizer/path/costsize.c#L4367-L4409).)

**Selectivity.** A boolean function used directly as a qual — `WHERE f(x)` — is estimated by `function_selectivity` [clausesel.c#clause_selectivity](../../../raw/postgres-14/src/backend/optimizer/path/clausesel.c#L890-L904). With no planner support function, this returns the "historical default estimate, 0.3333333" [plancat.c#function_selectivity](../../../raw/postgres-14/src/backend/optimizer/util/plancat.c#L1944-L1991). Any boolean-valued expression the planner does not otherwise recognize falls to `boolvarsel`, whose default without statistics is 0.5 [selfuncs.c#boolvarsel](../../../raw/postgres-14/src/backend/utils/adt/selfuncs.c#L1508-L1531). These are fixed guesses, unrelated to your data (contrast the statistics-driven defaults for ordinary operators [selfuncs.h#defaults](../../../raw/postgres-14/src/include/utils/selfuncs.h#L33-L56)). A function wrapped in an operator, e.g. `WHERE f(x) = 'a'`, is an `OpExpr` estimated by the operator's restriction estimator, which likewise has no statistics for the derived value `f(x)` unless a matching expression index has been analyzed.

**Set-returning functions** get a row estimate from `prorows` (default 1000 rows) unless a support function overrides it [plancat.c#get_function_rows](../../../raw/postgres-14/src/backend/optimizer/util/plancat.c#L2067-L2111) [create_function.sgml#ROWS](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L478-L489).

### Functions on a column defeat a plain index

The planner matches an index only when a clause operand *is* the indexed key. For a simple-column index the operand must be a matching `Var`; for an expression index it must be structurally `equal()` to the stored index expression [indxpath.c#match_index_to_operand](../../../raw/postgres-14/src/backend/optimizer/path/indxpath.c#L3751-L3814). Therefore `WHERE lower(email) = 'a@b.com'` cannot use a plain index on `email` — the operand is `lower(email)`, not a bare `Var`. You need an expression index on `lower(email)`. Once that exists, "the index expressions are not recomputed during an indexed search, since they are already stored in the index ... the system sees the query as just `WHERE indexedcolumn = 'constant'`" [indices.sgml#expression-indexes](../../../raw/postgres-14/doc/src/sgml/indices.sgml#L785-L795). The trade-off is that the expression is computed on every insert and non-HOT update instead.

The *other* side of an index comparison — the value the column is compared to — may itself be a function call, and that is fine as long as it is not volatile: the planner requires the non-index operand to contain no volatile functions and no `Var` of the indexed table [indxpath.c#match_opclause_to_indexcol](../../../raw/postgres-14/src/backend/optimizer/path/indxpath.c#L2504-L2535). Such an operand becomes a runtime key evaluated once per scan (previous section). A `VOLATILE` function there disqualifies the clause as an index condition.

### Leakproofness, security barriers, and qual ordering

At plan-creation time the executor's qual list is sorted by `order_qual_clauses`: quals of lower `security_level` must run before quals of higher `security_level`, and within a level cheaper (lower per-tuple cost) quals run first [createplan.c#order_qual_clauses](../../../raw/postgres-14/src/backend/optimizer/plan/createplan.c#L5191-L5310). This ordering is what lets the planner run cheap, selective conditions ahead of an expensive function — provided the function's cost is labeled truthfully (previous section). Note the sort key is `(security_level, per-tuple cost)`; selectivity is deliberately *not* part of it [createplan.c#order_qual_clauses-comment](../../../raw/postgres-14/src/backend/optimizer/plan/createplan.c#L5197-L5223).

`security_level` matters when the table has row-level security or is queried through a `security_barrier` view: those built-in conditions get the lowest level, and a user-supplied qual "cannot be evaluated before another clause with a lower `security_level` value unless the first clause is leakproof" [pathnodes.h#security_level](../../../raw/postgres-14/src/include/nodes/pathnodes.h#L2003-L2071). A non-leakproof function is therefore *pinned behind* the security quals: it cannot be reordered earlier and, for subquery/view flattening, cannot be pushed down into the protected subquery at all [allpaths.c#qual_is_pushdown_safe](../../../raw/postgres-14/src/backend/optimizer/path/allpaths.c#L3422-L3443). A leakproof, cheap qual is exempted and may move ahead [createplan.c#leakproof-exception](../../../raw/postgres-14/src/backend/optimizer/plan/createplan.c#L5261-L5274). Marking a safe function `LEAKPROOF` (superuser-only) lets it be "executed before conditions from security policies and security barrier views" [create_function.sgml#LEAKPROOF](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L357-L378), which can restore an index scan or earlier filtering that a security barrier otherwise blocks.

### Parallel safety

A function's `pg_proc.proparallel` category can disable parallelism for the whole statement [pg_proc.h#PROPARALLEL](../../../raw/postgres-14/src/include/catalog/pg_proc.h#L167-L174). User-defined functions default to `PARALLEL UNSAFE` [parallel.sgml#unsafe-default](../../../raw/postgres-14/doc/src/sgml/parallel.sgml#L182-L190), and "the presence of such a function in an SQL statement forces a serial execution plan" [create_function.sgml#PARALLEL](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L426-L459). The planner scans the whole query for the worst hazard and sets `parallelModeOK = (maxParallelHazard != PROPARALLEL_UNSAFE)` [planner.c#parallelModeOK](../../../raw/postgres-14/src/backend/optimizer/plan/planner.c#L336-L352). A `PARALLEL RESTRICTED` function is milder but still costly for a WHERE clause: "if a WHERE clause applied to a particular table is parallel restricted, the query planner will not consider performing a scan of that table in the parallel portion of a plan" [parallel.sgml#restricted-where](../../../raw/postgres-14/doc/src/sgml/parallel.sgml#L581-L591). So a large scan whose filter calls an unlabeled user function silently loses parallelism.

### Minimizing the overhead

Ordered roughly by leverage. Each maps to a mechanism above.

#### Label volatility correctly

Declare the function `IMMUTABLE` or `STABLE` when that is truthful, instead of accepting the `VOLATILE` default [xfunc.sgml#volatility](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml#L1626-L1666). `IMMUTABLE` enables constant folding with constant args and lets an expression index be built; `STABLE` allows the function to appear as an index-scan comparison value evaluated once per scan [clauses.c#evaluate_function](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4308-L4348) [nodeIndexscan.c#ExecIndexEvalRuntimeKeys](../../../raw/postgres-14/src/backend/executor/nodeIndexscan.c#L600-L652). This is `ALTER FUNCTION ... IMMUTABLE|STABLE`, a catalog change (no restart/reload); new plans referencing the function pick it up.

#### Provide an expression index (or restructure to an indexable column)

If the filter is `WHERE f(col) = k`, build `CREATE INDEX ... ON t ((f(col)))` so the derived value is stored and searched directly, turning a per-row seq-scan filter into an index lookup [indxpath.c#match_index_to_operand](../../../raw/postgres-14/src/backend/optimizer/path/indxpath.c#L3751-L3814) [indices.sgml#expression-indexes](../../../raw/postgres-14/doc/src/sgml/indices.sgml#L785-L795). `f` must be `IMMUTABLE` to index it. A stored generated column over `f(col)` plus an ordinary index is an equivalent precompute-and-store option. The cost moves to write time.

#### Wrap an uncorrelated call in a scalar sub-SELECT

If the function does not depend on the current row — e.g. `WHERE col = expensive_config()` — write it as `WHERE col = (SELECT expensive_config())`. An uncorrelated scalar sub-SELECT is planned as an InitPlan whose result is a `PARAM_EXEC` parameter [subselect.c#build_subplan](../../../raw/postgres-14/src/backend/optimizer/plan/subselect.c#L396-L409) [subselect.c#make_subplan](../../../raw/postgres-14/src/backend/optimizer/plan/subselect.c#L139-L160); the InitPlan is executed lazily at most once and its value cached, so subsequent reads just return the stored `Param` [nodeSubplan.c#initplan](../../../raw/postgres-14/src/backend/executor/nodeSubplan.c#L853-L876) [execExprInterp.c#ExecEvalParamExec](../../../raw/postgres-14/src/backend/executor/execExprInterp.c#L2426-L2441). This converts a per-row call into one call per execution. It only helps when the sub-SELECT is genuinely uncorrelated (a correlated sub-SELECT becomes a per-row SubPlan) and is a once-per-execution cache, not per-scan.

#### Let simple SQL functions be inlined

Write thin wrappers as SQL-language functions of the form `SELECT expression`, which the planner can inline to remove "the rather high per-call overhead of SQL functions" and expose the body to constant folding [clauses.c#inline_function](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4350-L4380). Inlining is skipped for non-SQL languages, `SECURITY DEFINER`, set-returning, `RECORD`-returning, or functions with a `SET` clause [clauses.c#inline-conditions](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4408-L4419), and it preserves the declared volatility/strictness [clauses.c#inline-volatility](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4603-L4616).

#### Give the planner accurate COST and ROWS

Set `COST` to reflect real per-call expense so the planner orders the qual after cheaper conditions and avoids redundant calls [create_function.sgml#COST](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L462-L476) [add_function_cost](../../../raw/postgres-14/src/backend/optimizer/util/plancat.c#L2006-L2050) [order_qual_clauses](../../../raw/postgres-14/src/backend/optimizer/plan/createplan.c#L5191-L5310); for set-returning functions set `ROWS` [create_function.sgml#ROWS](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L478-L489). These are `ALTER FUNCTION`/`CREATE FUNCTION` catalog settings.

#### Add a planner SUPPORT function

For a function whose selectivity, cost, or row count the defaults cannot capture, attach a `SUPPORT` function (superuser-only) [create_function.sgml#SUPPORT](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L491-L502). It can answer `SupportRequestSelectivity` (replace the 0.3333333 guess) [supportnodes.h#SupportRequestSelectivity](../../../raw/postgres-14/src/include/nodes/supportnodes.h#L91-L107), `SupportRequestCost` [supportnodes.h#SupportRequestCost](../../../raw/postgres-14/src/include/nodes/supportnodes.h#L131-L143), `SupportRequestRows` [supportnodes.h#SupportRequestRows](../../../raw/postgres-14/src/include/nodes/supportnodes.h#L158-L169), and even `SupportRequestIndexCondition` to synthesize a directly-indexable condition from an otherwise non-indexable function call [supportnodes.h#SupportRequestIndexCondition](../../../raw/postgres-14/src/include/nodes/supportnodes.h#L223-L240). These requests are consulted by `function_selectivity`, `add_function_cost`, `get_function_rows`, and `match_funcclause_to_indexcol` respectively.

#### Mark leakproof / parallel-safe when truthful

If the function leaks no information through errors or timing, `ALTER FUNCTION ... LEAKPROOF` (superuser) lets it be reordered ahead of security-barrier/RLS quals and pushed into protected subqueries, which can re-enable an index scan or earlier filtering [create_function.sgml#LEAKPROOF](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L357-L378) [qual_is_pushdown_safe](../../../raw/postgres-14/src/backend/optimizer/path/allpaths.c#L3422-L3443). If it is safe to run in parallel, mark it `PARALLEL SAFE` (or at least `RESTRICTED`) so a large scan can go parallel [create_function.sgml#PARALLEL](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L426-L459) [planner.c#parallelModeOK](../../../raw/postgres-14/src/backend/optimizer/plan/planner.c#L336-L352). Mislabeling can cause wrong answers, so only assert what is true.

#### Reduce how many rows the function sees

Because cost scales with rows reaching the qual, add cheaper, more selective, indexable conditions so the executor filters most rows before the expensive function runs; the planner already sorts quals cheapest-first within a security level [order_qual_clauses](../../../raw/postgres-14/src/backend/optimizer/plan/createplan.c#L5191-L5310), so an accurate `COST` plus a selective indexable predicate is often enough. For repeated reporting, precomputing into a materialized view moves the function cost out of the query path entirely.

## Context Reviewed

- Executor per-row path: `ExecScan` qual loop, `ExecQual`, `EEOP_FUNCEXPR`, fmgr dispatch.
- Parser/utility path for procedures: `ParseFuncOrColumn`/`prokind`, `CallStmt` grammar, `transformCallStmt`, `standard_ProcessUtility`; the `create_procedure` regression output.
- Volatility and constant folding: `pg_proc` `provolatile`/`PROVOLATILE_*`, `eval_const_expressions`, `evaluate_function`, `contain_volatile_functions`/`contain_mutable_functions`, plus the `xfunc.sgml` volatility categories.
- Cost/selectivity: `cost_qual_eval_walker`, `add_function_cost`, `procost`/`prorows`/`prosupport`, `function_selectivity`, `boolvarsel`, `selfuncs.h` defaults, the four `supportnodes.h` request types, `get_function_rows`; `create_function.sgml` COST/ROWS/SUPPORT.
- Index usage: `match_index_to_operand`, `match_opclause_to_indexcol`, `ExecReScanIndexScan`/`ExecIndexEvalRuntimeKeys`; `indices.sgml` expression-index section.
- Inlining and sub-SELECT caching: `inline_function`, `build_subplan`/`make_subplan`, `ExecEvalParamExec`, `nodeSubplan` InitPlan setup.
- Leakproof/parallel/ordering: `order_qual_clauses`, `RestrictInfo.security_level`, `qual_is_pushdown_safe`, `planner.c` parallel-mode gate, `PROPARALLEL_*`; `create_function.sgml` LEAKPROOF/PARALLEL and `parallel.sgml`.

## Evidence Map

| Claim | Source |
|---|---|
| Procedures (`prokind='p'`) are rejected in any expression; only `CALL` invokes them | [pg_proc.h#PROKIND](../../../raw/postgres-14/src/include/catalog/pg_proc.h#L150-L153), [parse_func.c#ParseFuncOrColumn](../../../raw/postgres-14/src/backend/parser/parse_func.c#L293-L302), [analyze.c#transformCallStmt](../../../raw/postgres-14/src/backend/parser/analyze.c#L3021-L3027), [gram.y#CallStmt](../../../raw/postgres-14/src/backend/parser/gram.y#L1038-L1043), [utility.c#T_CallStmt](../../../raw/postgres-14/src/backend/tcop/utility.c#L849-L851), [xfunc.sgml#procedures](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml#L105-L111), [create_procedure.out:52](../../../raw/postgres-14/src/test/regress/expected/create_procedure.out#L52-L56) |
| A WHERE-clause function runs once per row reaching the qual | [execScan.c#ExecScan](../../../raw/postgres-14/src/backend/executor/execScan.c#L202-L234), [executor.h#ExecQual](../../../raw/postgres-14/src/include/executor/executor.h#L400-L420), [execExprInterp.c#EEOP_FUNCEXPR](../../../raw/postgres-14/src/backend/executor/execExprInterp.c#L727-L738), [fmgr.h#FunctionCallInvoke](../../../raw/postgres-14/src/include/fmgr.h#L172) |
| SQL/non-C functions carry high per-call overhead (COST default 1 vs 100) | [create_function.sgml#COST](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L462-L476), [clauses.c#inline_function](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4350-L4380) |
| Volatility categories and default `VOLATILE`; per-row vs once-per-scan vs pre-evaluate | [pg_proc.h#PROVOLATILE](../../../raw/postgres-14/src/include/catalog/pg_proc.h#L155-L165), [xfunc.sgml#volatility](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml#L1626-L1666) |
| Constant folding needs all-const inputs and IMMUTABLE (STABLE only in estimation) | [clauses.c#eval_const_expressions](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L2123-L2159), [clauses.c#evaluate_function](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4308-L4348) |
| Runtime keys recomputed once per (re)scan, not per row | [nodeIndexscan.c#ExecReScanIndexScan](../../../raw/postgres-14/src/backend/executor/nodeIndexscan.c#L553-L593), [nodeIndexscan.c#ExecIndexEvalRuntimeKeys](../../../raw/postgres-14/src/backend/executor/nodeIndexscan.c#L600-L652) |
| Function/operator cost = `procost * cpu_operator_cost`; `procost` default 1; cached per RestrictInfo | [costsize.c#cost_qual_eval_walker](../../../raw/postgres-14/src/backend/optimizer/path/costsize.c#L4412-L4438), [plancat.c#add_function_cost](../../../raw/postgres-14/src/backend/optimizer/util/plancat.c#L2006-L2050), [pg_proc.h#procost](../../../raw/postgres-14/src/include/catalog/pg_proc.h#L46-L56), [costsize.c#eval-cost-cache](../../../raw/postgres-14/src/backend/optimizer/path/costsize.c#L4367-L4409) |
| Bare boolean function qual → 0.3333333 default selectivity; other boolean exprs → 0.5 | [clausesel.c#clause_selectivity](../../../raw/postgres-14/src/backend/optimizer/path/clausesel.c#L890-L904), [plancat.c#function_selectivity](../../../raw/postgres-14/src/backend/optimizer/util/plancat.c#L1944-L1991), [selfuncs.c#boolvarsel](../../../raw/postgres-14/src/backend/utils/adt/selfuncs.c#L1508-L1531), [selfuncs.h#defaults](../../../raw/postgres-14/src/include/utils/selfuncs.h#L33-L56) |
| SRF row estimate = `prorows` default 1000 unless support fn | [plancat.c#get_function_rows](../../../raw/postgres-14/src/backend/optimizer/util/plancat.c#L2067-L2111), [create_function.sgml#ROWS](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L478-L489) |
| A function on a column needs an expression index; stored, not recomputed at search | [indxpath.c#match_index_to_operand](../../../raw/postgres-14/src/backend/optimizer/path/indxpath.c#L3751-L3814), [indices.sgml#expression-indexes](../../../raw/postgres-14/doc/src/sgml/indices.sgml#L785-L795) |
| Non-index operand must be non-volatile / no Var of the indexed rel to be a runtime key | [indxpath.c#match_opclause_to_indexcol](../../../raw/postgres-14/src/backend/optimizer/path/indxpath.c#L2504-L2535) |
| Quals ordered by security_level then cost; leaky quals pinned behind security quals; leakproof exempt | [createplan.c#order_qual_clauses](../../../raw/postgres-14/src/backend/optimizer/plan/createplan.c#L5191-L5310), [pathnodes.h#security_level](../../../raw/postgres-14/src/include/nodes/pathnodes.h#L2003-L2071), [allpaths.c#qual_is_pushdown_safe](../../../raw/postgres-14/src/backend/optimizer/path/allpaths.c#L3422-L3443), [create_function.sgml#LEAKPROOF](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L357-L378) |
| PARALLEL UNSAFE forces serial plan; RESTRICTED blocks parallel scan of that table | [pg_proc.h#PROPARALLEL](../../../raw/postgres-14/src/include/catalog/pg_proc.h#L167-L174), [planner.c#parallelModeOK](../../../raw/postgres-14/src/backend/optimizer/plan/planner.c#L336-L352), [create_function.sgml#PARALLEL](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L426-L459), [parallel.sgml#restricted-where](../../../raw/postgres-14/doc/src/sgml/parallel.sgml#L581-L591) |
| Uncorrelated scalar sub-SELECT → InitPlan run once, cached in a PARAM_EXEC | [subselect.c#build_subplan](../../../raw/postgres-14/src/backend/optimizer/plan/subselect.c#L396-L409), [subselect.c#make_subplan](../../../raw/postgres-14/src/backend/optimizer/plan/subselect.c#L139-L160), [nodeSubplan.c#initplan](../../../raw/postgres-14/src/backend/executor/nodeSubplan.c#L853-L876), [execExprInterp.c#ExecEvalParamExec](../../../raw/postgres-14/src/backend/executor/execExprInterp.c#L2426-L2441) |
| Simple SQL functions can be inlined; conditions and volatility preservation | [clauses.c#inline_function](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4350-L4380), [clauses.c#inline-conditions](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4408-L4419), [clauses.c#inline-volatility](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c#L4603-L4616) |
| Planner support requests improve selectivity/cost/rows/index-condition | [supportnodes.h#SupportRequestSelectivity](../../../raw/postgres-14/src/include/nodes/supportnodes.h#L91-L107), [supportnodes.h#SupportRequestCost](../../../raw/postgres-14/src/include/nodes/supportnodes.h#L131-L143), [supportnodes.h#SupportRequestRows](../../../raw/postgres-14/src/include/nodes/supportnodes.h#L158-L169), [supportnodes.h#SupportRequestIndexCondition](../../../raw/postgres-14/src/include/nodes/supportnodes.h#L223-L240), [create_function.sgml#SUPPORT](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml#L491-L502) |

## Open Questions

- This page documents mechanisms and source-evident consequences at the pin; it does **not** include a quantified benchmark (calls/sec, plan-time deltas, or seq-scan-vs-index crossover) for a WHERE-clause function. No in-tree benchmark isolating this cost was located.
- The interaction between `ANALYZE` and expression-index statistics (how much a matching expression index improves `OpExpr` selectivity for `f(col) = k`) is asserted only in outline here; the exact statistics-gathering path for expression indexes was not traced in this checkout.
- Minor-release provenance within the 14.x series (the checkout is pinned at `REL_14_23-3-g5c00f4e2e3b`) for any of the planner/executor code above is not traced; this page describes behavior at the pin, not its change history.

## Source References

- [src/backend/executor/execScan.c](../../../raw/postgres-14/src/backend/executor/execScan.c) — `ExecScan` per-row qual loop.
- [src/include/executor/executor.h](../../../raw/postgres-14/src/include/executor/executor.h) — `ExecQual`.
- [src/backend/executor/execExprInterp.c](../../../raw/postgres-14/src/backend/executor/execExprInterp.c) — `EEOP_FUNCEXPR`, `ExecEvalParamExec`.
- [src/include/fmgr.h](../../../raw/postgres-14/src/include/fmgr.h) — `FunctionCallInvoke` dispatch.
- [src/backend/parser/parse_func.c](../../../raw/postgres-14/src/backend/parser/parse_func.c), [src/backend/parser/analyze.c](../../../raw/postgres-14/src/backend/parser/analyze.c), [src/backend/parser/gram.y](../../../raw/postgres-14/src/backend/parser/gram.y), [src/backend/tcop/utility.c](../../../raw/postgres-14/src/backend/tcop/utility.c) — procedure rejection and the `CALL` path.
- [src/include/catalog/pg_proc.h](../../../raw/postgres-14/src/include/catalog/pg_proc.h) — `prokind`, `provolatile`, `proparallel`, `proleakproof`, `procost`, `prorows`, `prosupport` and their symbolic macros.
- [src/backend/optimizer/util/clauses.c](../../../raw/postgres-14/src/backend/optimizer/util/clauses.c) — `eval_const_expressions`, `evaluate_function`, `inline_function`, volatility checkers.
- [src/backend/optimizer/path/costsize.c](../../../raw/postgres-14/src/backend/optimizer/path/costsize.c) — `cost_qual_eval_walker` and per-RestrictInfo cost caching.
- [src/backend/optimizer/util/plancat.c](../../../raw/postgres-14/src/backend/optimizer/util/plancat.c) — `function_selectivity`, `add_function_cost`, `get_function_rows`.
- [src/backend/optimizer/path/clausesel.c](../../../raw/postgres-14/src/backend/optimizer/path/clausesel.c), [src/backend/utils/adt/selfuncs.c](../../../raw/postgres-14/src/backend/utils/adt/selfuncs.c), [src/include/utils/selfuncs.h](../../../raw/postgres-14/src/include/utils/selfuncs.h) — clause selectivity dispatch, `boolvarsel`, default constants.
- [src/include/nodes/supportnodes.h](../../../raw/postgres-14/src/include/nodes/supportnodes.h) — planner support request structs.
- [src/backend/optimizer/path/indxpath.c](../../../raw/postgres-14/src/backend/optimizer/path/indxpath.c) — `match_index_to_operand`, `match_opclause_to_indexcol`.
- [src/backend/executor/nodeIndexscan.c](../../../raw/postgres-14/src/backend/executor/nodeIndexscan.c) — `ExecReScanIndexScan`, `ExecIndexEvalRuntimeKeys`.
- [src/backend/optimizer/plan/subselect.c](../../../raw/postgres-14/src/backend/optimizer/plan/subselect.c), [src/backend/executor/nodeSubplan.c](../../../raw/postgres-14/src/backend/executor/nodeSubplan.c) — uncorrelated sub-SELECT → InitPlan once-per-execution caching.
- [src/backend/optimizer/plan/createplan.c](../../../raw/postgres-14/src/backend/optimizer/plan/createplan.c), [src/include/nodes/pathnodes.h](../../../raw/postgres-14/src/include/nodes/pathnodes.h), [src/backend/optimizer/path/allpaths.c](../../../raw/postgres-14/src/backend/optimizer/path/allpaths.c) — `order_qual_clauses`, `security_level`, `qual_is_pushdown_safe`.
- [src/backend/optimizer/plan/planner.c](../../../raw/postgres-14/src/backend/optimizer/plan/planner.c) — whole-query parallel-mode gate.
- [doc/src/sgml/xfunc.sgml](../../../raw/postgres-14/doc/src/sgml/xfunc.sgml), [doc/src/sgml/indices.sgml](../../../raw/postgres-14/doc/src/sgml/indices.sgml), [doc/src/sgml/parallel.sgml](../../../raw/postgres-14/doc/src/sgml/parallel.sgml), [doc/src/sgml/ref/create_function.sgml](../../../raw/postgres-14/doc/src/sgml/ref/create_function.sgml) — volatility, expression indexes, parallel safety, and `CREATE FUNCTION` COST/ROWS/SUPPORT/LEAKPROOF/PARALLEL.
- [src/test/regress/expected/create_procedure.out](../../../raw/postgres-14/src/test/regress/expected/create_procedure.out) — proof that a procedure is rejected in a `SELECT`/expression context.

## Navigation

- [v14/index](../index.md)
- [PostgreSQL 14 Codebase Navigation Guide](../codebase-navigation-guide.md)
- [Row-Level Security (RLS) in PostgreSQL 14](row-level-security-rls.md)
- [Wiki Index](../../index.md)
- [Versions](../../versions.md)
