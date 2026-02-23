---
name: daxlib-svg-maintainer
description: Agent profile for authoring and maintaining DaxLib.SVG DAX UDFs for SVG visuals in Power BI.
---

# AGENTS.md

## Repository Context

This repository contains **DaxLib.SVG**, a DAX Lib package of **DAX User-Defined Functions (UDFs)** used to generate **SVG data visualizations in Power BI**.

Primary goals for changes:
- Keep functions model-independent and reusable.
- Preserve semantic correctness across model modes (Import, DirectQuery, Direct Lake, Composite).
- Optimize for maintainability and predictable performance.

## Tooling Context

- Library source is maintained as TMDL in `src/lib/functions.tmdl`.
- Example usage is available in `samples/pbip/svg/`.
- Preferred optimization workflow uses DAX Performance Tuner MCP concepts: baseline first, then iterative optimization, then validation.
- Function layering includes primitives (`Element.*`, `Def.*`), composition (`Compound.*`), and a higher-level presentation layer (`Visual.*`).

## Global DAX Rules (Always Apply)

1. **Use explicit parameter intent in UDFs**
   - Be explicit with parameter mode and type where meaningful (`VAL` vs `EXPR`, scalar/table/reference constraints).
   - Prefer explicitness over relying on defaults in shared library code.

2. **Choose `VAL` vs `EXPR` intentionally**
   - Use `VAL` when the function needs a pre-evaluated value.
   - Use `EXPR` when the expression must be re-evaluated in the function's internal context.
   - For `EXPR` evaluated inside row contexts, force context transition with `CALCULATE` where needed.

3. **Do not use `IF()` to return tables**
   - `IF()` cannot return table expressions in DAX UDF design for this library.
   - For table branching, use patterns based on `FILTER`, `CALCULATETABLE`, set operations, or branch outside table-returning code paths.

4. **Avoid `SUMMARIZECOLUMNS()` in library functions**
   - It does not support all semantic model modes consistently.
   - Prefer mode-safe alternatives (`SUMMARIZE`, `ADDCOLUMNS` over base table expressions, `GROUPBY` when appropriate).

5. **Keep generated SVG deterministic and safe**
   - Emit stable string output for equal inputs.
   - Avoid hidden dependence on volatile context unless clearly documented.

6. **Performance-first library mindset**
   - UDFs are reused by many measures; optimize repeated expression evaluation.
   - Cache repeated intermediate values in variables.
   - Validate optimized behavior with representative query contexts.

7. **Follow SQLBI UDF naming and comment conventions**
   - Function names use PascalCase and category namespaces with dot notation.
   - Keep model-independent library functions prefixed by library/category segments.
   - Parameter names use camelCase.
   - For `EXPR` parameters, use semantic suffixes where applicable: `Column`, `Table`, `Measure`, `Expr`, `Calendar`.
   - Use `///` JSDoc-style function comments with `@param` and `@returns`.

8. **Prevent naming collisions in semantic models**
   - Avoid names that can conflict with existing semantic-model objects (tables, columns, measures, functions).
   - Keep model-independent functions clearly namespaced to reduce collision risk with current/future DAX built-ins.
   - Treat reserved prefixes as unavailable for this library (`Dax`, `DaxLib`).

9. **Apply DAX Performance Tuner authoring rules**
   - Reuse repeated expressions with variable caching.
   - Remove duplicate filter predicates and consolidate filters.
   - Avoid context transition inside iterators when possible; pre-materialize base sets/values.
   - Prefer direct predicate filters in `CALCULATE`/`CALCULATETABLE` instead of wrapping simple predicates in `FILTER`.
   - Convert boolean `IF( condition, 1, 0 )` patterns to `INT( condition )` where semantics are equivalent.
   - Move context-independent expressions outside iterators.
   - Minimize materializations and duplicate scans.
   - Apply `SUMMARIZECOLUMNS` optimization guidance only when compatible with repository portability constraints.

10. **Be aware of new DAX functions (including `TABLEOF`)**
   - Treat `TABLEOF` as an available modern DAX option and evaluate it for readability/performance where appropriate.
   - Before using newly introduced functions, verify compatibility across target engines and semantic model modes used by this library.
   - Respect `TABLEOF` support constraints from Microsoft Learn (not supported in DirectQuery mode when used in calculated columns or RLS rules).
   - If compatibility is uncertain, prefer established cross-mode-safe patterns already adopted in this repository.

---

## Skill: `dax-udf-authoring`

### Use this skill when
- Creating or modifying functions in `src/lib/functions.tmdl`.
- Refactoring repeated measure logic into reusable DAX UDFs.
- Defining function signatures, parameter typing, and passing modes.

### Workflow
1. Define function contract first (name, return type intent, parameter types/modes).
2. Decide each parameter as `VAL` or `EXPR` based on evaluation semantics.
3. Apply SQLBI naming conventions to function and parameter names.
4. Implement with variables for readability and performance.
5. For `EXPR` inside iterators/row context, use explicit `CALCULATE` when context transition is required.
6. Add or update `///` JSDoc-style header docs including `@param` and `@returns`.

### Guardrails
- Use PascalCase for UDF names and dot notation to delineate categories/namespaces.
- Keep model-independent function names library-prefixed; do not introduce unprefixed model-independent names.
- Check for collisions with existing semantic-model object names before introducing new UDF names.
- Avoid implicit casting surprises; choose parameter types deliberately.
- Avoid over-broad `ANYVAL` if a stricter scalar or table type improves safety.
- Use camelCase for parameters, including explicit `EXPR` semantic suffixes (`Column`, `Table`, `Measure`, `Expr`, `Calendar`) when relevant.
- Do not use reserved leading prefixes for general community libraries (`Dax`, `DaxLib`).

### Done criteria
- Signature reflects intended evaluation behavior.
- No ambiguous `VAL`/`EXPR` behavior remains.
- Function is reusable, documented, and consistent with existing naming patterns.
- Function comments follow `///` JSDoc-style tags (`@param`, `@returns`) and naming conventions are compliant.

---

## Skill: `svg-visual-composition`

### Use this skill when
- Adding/changing SVG wrapper, elements, defs, attributes, transforms, scales, or color helpers.
- Designing reusable primitives for charts and custom visual marks.
- Designing higher-level `Visual.*` functions that package reusable chart patterns.

### Workflow
1. Keep primitive functions single-purpose (`Element.*`, `Def.*`, `Attr.*`, `Transforms`, `Scale.*`).
2. Compose larger visuals from primitives rather than hardcoding monolithic strings.
3. Build `Compound.*` functions as reusable mid-level components.
4. Build `Visual.*` functions above `Compound.*` with predefined formatting preferences aligned to sample PBIP patterns.
5. In `Visual.*`, prefer parameter surfaces focused on data inputs and color inputs; avoid exposing many low-level formatting knobs by default.
6. Prefer `path` markup instead of more verbose primitives (for example `line`) when this reduces output string size without harming readability.
7. For repeated marks with shared styling/shape, prefer `<defs>` + `<use>` patterns to minimize duplicated attribute strings.
8. Keep optional attributes conditional (`ISBLANK` checks) to avoid noisy markup.
9. Ensure output is valid SVG fragments and properly wrapped by `DaxLib.SVG.SVG`.

### Guardrails
- Do not introduce assumptions tied to one report layout or one data model.
- Preserve composability (functions should accept content/attrs as inputs).
- Keep outputs compact but readable for debugging.
- Favor string-size-efficient SVG patterns for high-density visuals to reduce risk of Power BI string/memory limit issues.
- Keep `Visual.*` defaults opinionated and consistent with sample PBIP examples; only expose parameters that are broadly needed for data and color variation.

### Done criteria
- New/changed function fits existing category taxonomy.
- SVG output remains valid and composable.
- Optional behavior is parameter-driven, not hardcoded.
- For repetitive marks, implementation uses size-aware patterns (`path`, `<defs>`, `<use>`) where beneficial.
- `Visual.*` functions exist above `Compound.*` and encapsulate formatting preferences while primarily accepting data-related and color parameters.

---

## Skill: `semantic-model-safe-table-logic`

### Use this skill when
- Writing table-returning UDF logic.
- Implementing grouping, ranking, filtering, or table-shaping helpers.
- Ensuring compatibility across Import/DirectQuery/Direct Lake/Composite models.

### Required constraints
- Never rely on `IF()` to branch between table expressions.
- Do not use `SUMMARIZECOLUMNS()` in this library.

### Preferred patterns
- Table filters: `FILTER`, `CALCULATETABLE`.
- Grouping/projection: `SUMMARIZE`, `ADDCOLUMNS`, `SELECTCOLUMNS`, `GROUPBY` (case-by-case).
- Set logic: `UNION`, `INTERSECT`, `EXCEPT`, `NATURAL*JOIN` (when model semantics are clear).

### Validation checklist
- Works under different filter contexts.
- Does not depend on unsupported mode-specific behavior.
- Table shape and column lineage remain clear to consumers.
- New function usage (such as `TABLEOF`) is gated by compatibility checks and safe fallback patterns.

---

## Skill: `dax-performance-tuning`

### Use this skill when
- Optimizing measure/UDF performance.
- Refactoring expensive repeated expressions.
- Comparing original vs optimized DAX behavior.

### Workflow (aligned with DAX Performance Tuner MCP guidance)
1. Establish baseline query behavior and performance.
2. Apply one optimization at a time.
3. Compare results for semantic equivalence in representative contexts.
4. Keep optimized and original versions side-by-side during validation.
5. Re-test before finalizing.

### Authoring rules from `ArticlePatterns.cs`
- Cache repeated measure/expression evaluations in variables.
- Eliminate duplicate filter conditions across nested contexts.
- Prefer set-based rewrites over row-by-row iterator logic when possible.
- Replace simple `IF(...,1,0)` with `INT(...)` when equivalent.
- Avoid unnecessary `FILTER` wrappers for simple column predicates; pass predicates directly as filter arguments.
- Move constants and context-invariant calculations outside iterators.
- Monitor for callback-prone patterns and reduce FE-heavy execution shapes.
- For this repository, keep the existing semantic-mode portability rule: avoid introducing `SUMMARIZECOLUMNS()` in library functions.

### Guardrails
- Semantic equivalence checks are valuable but not exhaustive.
- Validate across realistic filters/slicers and edge-case contexts.
- Never trade correctness for speed.

### Done criteria
- Performance improvement is measurable for target scenarios.
- Results are consistent with baseline intent in tested contexts.
- Optimization rationale is clear and maintainable.

## Sources Used to Build This Guidance

- DAX Lib docs: https://docs.daxlib.org/
- DAX UDF concepts and parameter semantics: https://www.sqlbi.com/articles/introducing-user-defined-functions-in-dax/
- SQLBI naming conventions for DAX/UDFs and JSDoc-style comments: https://docs.sqlbi.com/dax-style/dax-naming-conventions#function-names
- DAX Performance Tuner MCP workflow and cautions: https://github.com/microsoft/fabric-toolbox/tree/main/tools/DAXPerformanceTunerMCPServer
- DAX Performance Tuner pattern rules (`ArticlePatterns.cs`): https://github.com/microsoft/fabric-toolbox/blob/cf345d114e77a8c90bf8f346c9dfaa56ba653dd9/tools/DAXPerformanceTunerMCPServer/src/DaxPerformanceTuner.Library/Data/ArticlePatterns.cs#L111
- Microsoft Learn reference for `TABLEOF`: https://learn.microsoft.com/en-us/dax/tableof-function-dax
- Power BI semantic model skill structure and validation discipline: https://github.com/RuiRomano/powerbi-agentic-plugins/blob/main/plugins/powerbi/skills/powerbi-semantic-model/SKILL.md
- SVG string minimization strategies (`path`, `<defs>`, `<use>`): https://evaluationcontext.github.io/posts/minimizing-svg-strings