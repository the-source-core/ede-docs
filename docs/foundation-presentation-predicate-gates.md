# Predicate-Gated View Rendering

> Developer guide for the `visible-when` / `hidden-when` DSL attributes and the `@api.view_predicate` decorator that powers them.

**Status:** ✅ Delivered 2026-05-25 (foundation.presentation Enhancement 04 — Slices A + B + C). Cache layer + Prometheus metric deferred to a follow-up perf enhancement when scale demands it; strip-pass runs per-request and is correct without cache.

## What it is

Two DSL attributes that gate the rendering of any element in a form / list / search / kanban / settings view, evaluated server-side at RenderPlan compile time against arbitrary predicates registered by the module that owns the concept:

```xml
<field name="allowed_unit_ids" visible-when="config('base.allow_multiple_units')" />

<field name="margin_override"
       visible-when="has_roles('pricing.approver') and config('base.advanced_mode')" />

<field name="legacy_field" hidden-when="config('base.modern_mode')" />

<field name="premium_only" visible-when="config('base.subscription_tier') == 'PREMIUM'" />

<field name="any_admin" visible-when="has_roles('crm.admin,sales.admin')" />
<field name="approval_team" visible-when="has_team_roles('SALES.SALES_MANAGER,PRICING.APPROVER')" />
```

Mutual exclusion: a single element MUST NOT carry both `visible-when` AND `hidden-when` — the parser rejects it at boot time.

## Where it applies

Every renderable DSL element:

| View type | Gateable elements |
|---|---|
| Form | `<field>` · `<section>` · `<page>` (notebook) · `<button>` · `<statusbar>` · `<DynamicProperties>` |
| List | column `<field>` |
| Search | `<field>` · `<filter>` · `<groupby>` |
| Kanban | `<KanbanRibbon>` · `<KanbanField>` · `<button>` |
| Settings | `<section>` |

Implementation pattern: a shared RelaxNG `<define name="gate-attrs">` is `<ref/>`'d from every gateable element's define in [`view.rng`](../src/ede/core/services/presentation/dsl/schemas/view.rng). Adding the attributes to a new element type later = one `<ref name="gate-attrs"/>` line.

**Explicitly NOT gateable** (kept narrow on purpose):
- `<view>` root — view existence is RBAC + menu, one source of truth
- `<ir.menu>` / `<ir.action>` — already gated via RBAC permission rows; double-gating creates ambiguity
- `<ListView>` / `<FormView>` / `<KanbanView>` / `<SearchView>` view-type roots — gate at menu, not at view shape

## How expressions work

Expressions are evaluated by a dedicated safe AST evaluator at [`src/ede/core/services/presentation/predicate_expression.py`](../src/ede/core/services/presentation/predicate_expression.py). The grammar:

- **Predicate calls** with positional + keyword arguments: `config('K')`, `config('K', defValue='fallback')`, `has_roles('a,b')`
- **Boolean operators**: `and` / `or` / `not`
- **Comparison operators**: `==` / `!=` / `<` / `<=` / `>` / `>=`
- **Literals**: single-quoted strings, integers, decimals, `True`, `False`, `None`
- **Parens for grouping**

What is explicitly **rejected**:
- Bare identifiers (must call a predicate, not reference a variable)
- Attribute access (`os.environ`)
- Lambdas, comprehensions, subscript, `**kwargs` splat

A grammar violation surfaces as `PredicateExpressionError` at strip time → fail-closed strip.

## The three first-party predicates

All three live in [`src/ede/foundation/base/services/predicates/`](../src/ede/foundation/base/services/predicates/) — owned by foundation.base, NOT foundation.presentation (which owns only the registry mechanism).

| View DSL form | Python env method | Returns | Module |
|---|---|---|---|
| `config('K')` · `config('K', defValue=X)` | `env.get_config('K', defValue=X)` | value (bool / str / int / None) | foundation.base |
| `has_roles('A,B')` | `env.has_roles('A,B')` | bool — OR semantics across the CSV | foundation.base |
| `has_team_roles('TEAM.ROLE,T2.R2')` | `env.has_team_roles('T1.R1,T2.R2')` | bool — OR semantics; pair format `<team_code>.<role_code>` | foundation.base |

The verb prefix (`get_*` vs `has_*`) encodes the return-shape contract — caller knows the shape from the method name alone.

## Calling from Python

The same predicates registered via `@api.view_predicate` are dynamically bound onto every `Env` instance:

```python
def my_handler(env):
    if env.get_config('base.advanced_mode'):
        ...
    if env.has_roles('pricing.approver'):
        ...
    if env.has_team_roles('SALES.MANAGER'):
        ...
```

The binding happens via `Env.__getattr__` fall-through — no per-predicate `Env` class edit. Adding `@api.view_predicate(view_name="my_thing", env_method="has_my_thing")` in any module immediately makes `env.has_my_thing(...)` callable.

## Registering a new predicate

```python
# src/ede/foundation/<your-module>/services/predicates/my_predicate.py
from ede.core import api


@api.view_predicate(view_name="my_predicate", env_method="has_my_predicate")
def predicate_my_predicate(arg: str, env, **kwargs) -> bool:
    """True when ..."""
    ...
```

Then import the module from your `__init__.py` chain so the decorator fires at boot.

**Naming rules:**
- `view_name` is what appears inside `visible-when="..."` — keep it short (`config` / `roles` / `team_roles`).
- `env_method` is what appears on `env.X(...)` — verb-prefixed (`get_*` for value, `has_*` for bool).
- Both names must be unique across the registry. Boot fails with `DuplicateViewPredicate` if two modules try to claim the same name.

**Signature contract:** `(arg, env, **kwargs) -> Any`. The first positional is always the string argument from the DSL; `env` is the second positional; any keyword args come from the call site.

**Where to put it:** the module that owns the concept. Foundation.base owns `config` / `roles` / `team_roles` because `SettingsService` + RBAC + `TeamRoleService` all live there. Future predicates (`active_unit`, `country_pack`, custom-tenant feature flags) live in their owning modules.

## Decision tree — visible-when vs the alternatives

| Constraint | Mechanism |
|---|---|
| Per-record dynamic state (`state == 'draft'`, `amount > 1000`) | `invisible="..."` (client-side, per-record) — unchanged |
| Setting / role / team-role driven | `visible-when=` / `hidden-when=` (this enhancement) |
| Write / command authorization | `ir.rbac.permission` rows — server-enforced at command bus |
| Menu / action visibility | `ir.rbac.permission` rows + `is_visible` on `ir.menu` |
| Field always read-only when condition met | `readonly="..."` |
| Domain restriction on records returned | record rules (`domain` column on permissions / record-rule engine) |
| ORM-level data isolation by org / unit | `@api.model(unit_scope=...)` (foundation.security Enh 01) |

**`visible-when` and `invisible` coexist** on the same element — both must be truthy (no strip + no client-hide) for the field to render.

## Strip pipeline

```
parse XML
  → apply <extend ref=...> inheritance composition
  → strip-by-predicate pass            ←──── this enhancement
  → serialize RenderPlan
  → ship to React client
```

Strip pass: depth-first walk; every node with `visible-when` / `hidden-when` is evaluated. visible-when False → strip; hidden-when True → strip. Section / page subtree-strip is efficient — descendants are never re-walked.

## Fail-closed semantics

If anything goes wrong during expression evaluation — syntax error, unknown predicate, predicate exception — the element is **stripped**. Better to lose a field from the UI than to leak it.

When `PRESENTATION_PREDICATE_DEBUG_LOG=True`, the strip emits a structured WARNING log record:

```
[VIEW-STRIP] element=field name='allowed_unit_ids' attr=visible-when
  expr="config('base.allow_multiple_units')" outcome=stripped detail=False
```

Production silent by default.

## Boot validation

`validate_render_plan_predicates(plan, registered_names=...)` walks every gate expression in a parsed plan, collects every referenced predicate name, and reports two failure modes:

- Expression syntax error / disallowed node
- Reference to an unregistered predicate name

Wired into the boot pipeline via the `PRESENTATION_PREDICATE_VALIDATION` setting:
- `strict` (production default) → boot fails on unknown predicate
- `warn` → log + continue (DEV ergonomics)
- `off` → skip entirely (runtime fail-closed still applies)

## Admin debug tool

`presentation.debug_view_strip` command + `view.debug.read` RBAC permission (system_admin only). Returns:

```json
{
  "status": "ok",
  "view_id": "res_organization_form_view",
  "principal": {"user_id": "...", "role_codes": [...], "is_system": false},
  "unstripped": <RenderPlan with gates intact>,
  "stripped":   <RenderPlan the principal would actually receive>,
  "registered_predicates": {
    "view_names":  ["config", "has_roles", "has_team_roles"],
    "env_methods": ["get_config", "has_roles", "has_team_roles"]
  }
}
```

Helps operators answer "where did my field go?" — every stripped element appears in `unstripped` but is missing from `stripped`.

## Settings (FoundationSettings)

| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `PRESENTATION_PREDICATE_VALIDATION` | `str` (`strict` / `warn` / `off`) | `strict` | `EDE_PRESENTATION_PREDICATE_VALIDATION` | Boot-validation behavior |
| `PRESENTATION_PREDICATE_DEBUG_LOG` | `bool` | `False` | `EDE_PRESENTATION_PREDICATE_DEBUG_LOG` | Emit structured `[VIEW-STRIP]` records on every strip |

## RBAC

| Permission Code | Resource | Action | Default Role |
|---|---|---|---|
| `view.debug.read` | `foundation.presentation` | `read` | `rbac.role_system_admin` |

## What this does NOT change

- `invisible=` attribute — stays as-is. Per-record dynamic, client-side evaluated.
- RBAC permission rows — gating writes / commands / actions stays in `ir.rbac.permission`.
- Menu visibility — stays gated by RBAC + `is_visible` on `ir.menu`.
- `<if expr="...">` tag form — out of scope; attribute form is the only DSL surface.
- Convenience aliases (`<if-config>`, `<if-role>`) — out of scope.
- `<else>` companion — out of scope; authors use two elements with negated expressions.
- Per-tenant custom predicates via admin UI — Phase 2 candidate.

## Things developers commonly get wrong

- **Putting a predicate in `foundation.presentation`**. Predicates live in the module that owns the concept they gate on. foundation.presentation owns only the registry mechanism, not any concepts.
- **Mixing `visible-when` and `hidden-when` on one element**. Pick one; negate the expression if you need to invert.
- **Using `field=` on `<groupby>` for gating**. There is no `field=` on groupby; the gate attributes are `visible-when` / `hidden-when` like every other element.
- **Expecting the strip to happen client-side**. Strip is **server-side** in the RenderPlan compile pass — stripped elements never reach the browser. UI cannot be DevTools-inspected to reveal hidden fields.
- **Bare identifiers in expressions**. `admin_flag` is not allowed; must be a predicate call: `has_roles('admin')`. The parser rejects bare identifiers with `PredicateExpressionError`.
- **Forgetting that the env method name must be unique too**. Both `view_name` AND `env_method` must be unique across the registry. Boot fails with `DuplicateViewPredicate` if two modules try to claim either name.

## Related

- [foundation-presentation.md](./foundation-presentation.md) — Presentation module overview
- [foundation-base.md](./foundation-base.md) — Where the three first-party predicates live
- [foundation-base-extensions.md](./foundation-base-extensions.md) — `@api.extend_model` decorator (mirrors the `@api.view_predicate` boot pattern)
- Roadmap: [Enhancement 04](../roadmap/foundation/presentation/enhancements/04-predicate-gated-view-rendering.md)
- Reusable skill: [`.claude/skills/using-predicate-view-gates/SKILL.md`](../.claude/skills/using-predicate-view-gates/SKILL.md)
