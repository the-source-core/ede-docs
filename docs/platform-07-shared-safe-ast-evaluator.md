<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Shared Safe AST Evaluator — Implementation Docs

**Module:** `ede.core.engines.formula` (kernel cross-cutting utility)
**Roadmap:** [roadmap/platform/07-shared-safe-ast-evaluator.md](../roadmap/platform/07-shared-safe-ast-evaluator.md)
**Status:** 🔴 Not Started (drafted 2026-05-12)
**Layer:** Platform-wide

> Source of truth is the roadmap. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A single AST-restricted Python expression evaluator at `src/ede/core/engines/formula/safe_eval.py` plus a curated function library at `formula/functions.py`. Public API: `safe_eval_number(expr: str, scope: dict | None = None, function_set="numeric"|"full", expected_type=None)`. Parses an expression to `ast.parse(..., mode='eval')`, walks the tree against a closed node-type whitelist, resolves names against the read-only `scope` dict, and either returns a numeric/string result or raises a typed error.

Two consumers share the single implementation:
1. **Metric formula engine** (`engines/metric/formula_engine.py`) — evaluates `Metric.expr` like `"{{a}} - {{b}}"` with `function_set="numeric"`.
2. **DML variables engine** (`engines/document/dml/variables.py`) — evaluates `<var formula="$subtotal * $taxRate"/>` with `function_set="full"` (numeric + string + date + format).
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Both consumers face the same security problem: the expression is authored by tenant admins (metrics) or per-customer template authors (DML). Passing the string to Python `eval()` would let a malicious author run `__import__('os').system(...)` or any other arbitrary code.

The fix is an AST-restricted evaluator. Two independent implementations would mean two attack surfaces, two test suites, and twice the chance of a security regression. A single shared evaluator means one inventory of allowed AST node types, one function whitelist, one set of pytest cases, and one place to harden when a Python AST CVE drops.

The reference reporting stack we evaluated has a similar pattern but lives only at the metric layer. We extend the idea: same evaluator, two function sets (numeric for metrics, full for DML), one security boundary.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Metric authors:** declare `engine="formula"` + `depends_on=[...]` + `expr="..."` on a `Metric`. The formula engine substitutes `{{dep_key}}` with resolved values, then calls `safe_eval_number(expr, scope=resolved_scope, function_set="numeric")`. No direct interaction with the evaluator API.
- **DML template authors:** declare `<var id="dueDate" formula="add_days($invoiceDate, $paymentDays)" type="date"/>`. The DML variables engine substitutes `$varName` with resolved scope values, then calls `safe_eval_number(..., function_set="full")`. No direct interaction.
- **Engine authors adding a new expression surface:** consult [roadmap/platform/07-shared-safe-ast-evaluator.md](../roadmap/platform/07-shared-safe-ast-evaluator.md) before forking. New surfaces with different requirements (e.g. workflow guard expressions) should be reviewed against this design.
- **Security reviewers:** the AST node whitelist + function whitelist + `mode='eval'` parse mode form the inventory of safely-evaluatable inputs. Reviewed when a Python AST CVE is announced.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
src/ede/core/engines/formula/
├── __init__.py
├── safe_eval.py       ← AST walker + node-type whitelist + name resolver
├── functions.py       ← whitelisted callable registry (numeric + full sets)
├── errors.py          ← FormulaSyntaxError / FormulaNameError / FormulaFunctionError / ...
└── dag.py             ← (Phase 3) shared cycle-detection for metric formula DAG + DML <var> DAG

Caller side:
  Metric formula engine: substitutes {{dep_key}} → values, then safe_eval_number(..., function_set="numeric")
  DML variables engine:  substitutes $varName    → values, then safe_eval_number(..., function_set="full")

AST node whitelist (closed):
  Module, Expression, Constant (num+str), Name, BinOp (+ - * / % **),
  UnaryOp (+ -), Compare (== != < <= > >=), BoolOp (and or),
  Call (only to whitelisted functions), IfExp (ternary)
  Everything else → FormulaSyntaxError.

Function whitelist (closed):
  numeric set: abs round floor ceil min max pow sum avg count if coalesce is_empty
  full set:    numeric ∪ today add_days add_months add_years days_between format_date
                          ∪ concat upper lower length substr replace
                          ∪ currency percent format_number
```

Hardening reviewed when a Python `ast` CVE is announced — the AST whitelist + function whitelist + `mode='eval'` are the three security gates.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none_ | Pure library — no ORM models. | n/a |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet — all 🔴 Not Started_ | Phase 1: `safe_eval_number` + numeric function set + error types. Phase 2: extends `functions.py` with date + string + format for DML. Phase 3: extracts `dag.py` shared by metric formula DAG and DML `<var>` DAG. | (planned) `src/ede/core/engines/formula/...` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none_ | The evaluator is a pure library, not a command. | n/a |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none_ | Pure library, no runtime events. | n/a |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none_ | Consumed indirectly via `ede.metric.run` (formula metrics) + `ede.document.render` (DML variables). | n/a |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
*Not applicable — stateless library.*
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- No `ACTIVE_MODULES` entry — pure library.
- No manifest changes.
<!-- /SYNC-BLOCK -->

### Foundation-level settings
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Evaluator + Numeric Function Set | 🔴 | [roadmap/platform/07-shared-safe-ast-evaluator.md](../roadmap/platform/07-shared-safe-ast-evaluator.md) — lands with `foundation.dataset` Phase 2 (W2) |
| Phase 2 | Full Function Set for DML | 🔴 | Same file — lands with `foundation.document` Phase 2 (W6) |
| Phase 3 | Shared Cycle Detection Helper | 🔴 | Same file — lands when DML `<var>` cross-references are needed (W6) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Evaluator not yet implemented | 🔴 Not Started | [roadmap/platform/07-shared-safe-ast-evaluator.md](../roadmap/platform/07-shared-safe-ast-evaluator.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
*Will be populated as integrations land — most likely entry: "reaching for `eval()` for any user-authored expression in a new surface — instead use this evaluator with the right function_set."*

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
*Pre-build. No schema changes.*
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ | Pure library. |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`platform-05-engine-substrate`](./platform-05-engine-substrate.md) — establishes `engines/formula/` where the evaluator lives.
- [`foundation.dataset`](./foundation-dataset.md) — Phase 2 introduces the metric formula engine that consumes this evaluator with `function_set="numeric"`.
- [`foundation.document`](./foundation-document.md) — Phase 2 introduces DML variables-with-formula that consumes this evaluator with `function_set="full"`.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-12. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
