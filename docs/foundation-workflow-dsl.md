# Foundation Workflow — `<workflow>` XML DSL

**Status:** 🟡 In Progress (Phase 1 — DSL parser + named-guard registry shipped)
**Module:** [foundation.workflow](./foundation-workflow.md)
**Schema:** [src/ede/foundation/workflow/data_loader/schemas/workflow.rng](../src/ede/foundation/workflow/data_loader/schemas/workflow.rng)

> Source of truth is the roadmap at [roadmap/foundation/workflow/](../roadmap/foundation/workflow/README.md). This page is the authoring reference for the `<workflow>` DSL — the supported way to seed workflow rows from a domain's `data/` folder.

---

## What This DSL Is For

`foundation.workflow` stores stage / transition / guard configuration in `ir.workflow.*` rows. The DSL is the install-time front-end that compiles a single, readable XML document down to those rows.

Three problems the DSL solves vs raw `<record model="ir.workflow.*">` seeding:

1. **Diff-readable guards.** Guard expressions live as DSL text on a `<guard>` element instead of as embedded JSON inside a CSV cell.
2. **Local code resolution.** `from`/`to` and `guard` reference each other by `code` within the same `<workflow>` block — no external IDs required.
3. **One-place, named guards.** Recurring predicates live as named `<guard>` rows; transitions reference them by `code`. Threshold changes require one edit, not N.

The Phase 3 Visual Editor will round-trip back to this same XML format — what you author here is what `Export → XML` produces in the canvas.

---

## Quick Example — End-to-End

```xml
<workflow code="pricing.rate.lifecycle"
          model="pricing.rate"
          field="status"
          name="Pricing Rate Lifecycle"
          description="Maker-checker rate approval flow with auto-approve for safe margins."
          noupdate="true">

  <guards>
    <guard code="rate_risky_margin"
           label="Margin requires approval"
           description="Margin sits below the commercial floor — needs maker-checker."
           expression="subject.margin_risk_level == 'risk' OR subject.margin_risk_level == 'negative_risk'"/>
    <guard code="rate_safe_margin"
           label="Margin within auto-approve band"
           expression="subject.margin_risk_level == 'safe' OR subject.margin_risk_level == 'watch' OR subject.margin_risk_level == null"/>
  </guards>

  <stage code="draft"             label="Draft"            initial="true"  color="grey"/>
  <stage code="pending_approval"  label="Pending Approval"                 color="amber"/>
  <stage code="approved"          label="Approved"                         color="green"/>
  <stage code="rejected"          label="Rejected"         terminal="true" color="red"/>

  <transition code="submit_for_approval"
              from="draft" to="pending_approval"
              label="Submit for Approval"
              on-command="pricing.rate.submit"
              guard="rate_risky_margin">
    <action type="request_approval"
            policy-set="pricing.rate.standard"
            success-event="approval.case.approved"
            failure-event="approval.case.rejected"/>
  </transition>

  <transition code="submit_auto_approve"
              from="draft" to="approved"
              on-command="pricing.rate.submit"
              guard="rate_safe_margin"/>

  <transition code="approval_completed"
              from="pending_approval" to="approved"
              on-event="approval.case.approved"/>
</workflow>
```

Save this as `data/<your-name>_workflow.xml` under your app, list it in the manifest's `data` array, and `ede migrate upgrade` will compile it into `ir.workflow.*` rows.

---

## Element Reference

### `<workflow>` — root element

| Attribute | Required | Notes |
|---|---|---|
| `code` | ✅ | Globally unique, e.g. `pricing.rate.lifecycle`. Becomes `ir.workflow.definition.code`. |
| `model` | ✅ | The model whose field this workflow controls (e.g. `pricing.rate`). |
| `field` | ✅ | The field name on `model` that has `workflow=True`. |
| `name` | ✅ | Human-readable label shown in the admin UI. |
| `description` | – | Optional one-line summary. |
| `noupdate` | – | `true`/`false` (default `false`). When `true`, re-running `ede migrate upgrade` skips the definition row so admin edits in the Visual Editor survive. Recommended for production-ready workflows. |

### `<guards>` — named guard registry

A single optional block containing one or more `<guard>` elements. Guards declared here are referenced by `code` from `<transition guard="...">`.

```xml
<guards>
  <guard code="<unique-code>"
         label="<short label>"
         description="<why this guard exists — commercial intent>"
         expression="<DSL expression>"/>
</guards>
```

`expression` is parsed by the [expression compiler](#expression-grammar) at install time. Compilation errors halt the migration with a precise message naming the guard code.

### `<stage>` — workflow stages

```xml
<stage code="<unique-within-workflow>"
       label="<display label>"
       [initial="true|false"]
       [terminal="true|false"]
       [color="<token>"]
       [sequence="<int>"]
       [target-kind="enum|reference"]
       [target-value="<override>"]/>
```

- Exactly one stage should set `initial="true"`. If none is marked, the parser auto-flags the first stage as initial.
- `target-value` defaults to `code` — only set it when the literal stored in the parent field differs from the stage code (rare).
- `target-kind="reference"` is reserved for stages backed by an `ir.workflow.stage` master record (not used in Phase 1 demos).

### `<transition>` — directed edges

```xml
<transition code="<unique-within-workflow>"
            from="<stage-code>"
            to="<stage-code>"
            [label="<button label>"]
            (on-command="..." | on-event="..." | on-cron="...")
            [guard="<guard-code>" | guard-expr="<inline-expression>"]>
  [<action ... />]
</transition>
```

Rules enforced by the parser (the schema covers most; the parser fills in cross-element checks):

- `from` and `to` must reference `<stage code>` declared in the same `<workflow>`.
- Exactly one of `on-command`, `on-event`, `on-cron` must be present.
- `guard` and `guard-expr` are mutually exclusive. Use `guard` for any predicate worth naming; `guard-expr` only for self-explanatory one-offs (`subject.amount > 0`).
- Omitting both means the transition always permits.

### `<action>` — what happens when the transition fires

```xml
<action type="request_approval|dispatch_command|emit_event"
        [policy-set="<approval-policy-set-code>"]
        [command="<command-name>"]
        [event="<event-name>"]
        [success-event="<event-name>"]
        [failure-event="<event-name>"]
        [config-json="<JSON literal>"]/>
```

| `type` | Required attributes |
|---|---|
| `request_approval` | `policy-set` (which approval policy set selects the flow). `success-event` + `failure-event` map to the engine events that complete or abort the transition. |
| `dispatch_command` | `command` (the EDE command to dispatch on success). |
| `emit_event` | `event` (the bus event to emit on success). |

`config-json` is a free-form JSON dict merged into `ir.workflow.transition.action_config`. Use it for engine-specific extras (`{"domain": "pricing"}`).

If the `<action>` element is omitted, the transition is treated as `action_type="none"` — the engine just writes the new stage value and emits `workflow.stage.exited` / `workflow.stage.entered`.

---

## Expression Grammar

The same grammar applies to `<guard expression="...">` and inline `guard-expr="..."`. The compiler translates DSL keywords to Python equivalents and verifies the result against the safe-AST evaluator's whitelist.

### Operators

| Category | Allowed |
|---|---|
| Comparison | `==`, `!=`, `<`, `<=`, `>`, `>=` |
| Boolean | `AND`, `OR`, `NOT` (uppercase), parentheses for grouping |
| Literals | numbers, single-quoted strings (`'admin'`), `true`, `false`, `null` |

### Identifiers

Only these root names are accepted at compile time — anything else fails the migration:

| Root | Meaning |
|---|---|
| `subject` | The record being transitioned. Read fields via `subject.<field>[.<nested>]`. |
| `principal` / `requester` | Same dict — the actor's principal claims (user_id, role codes, …). |
| `now` | UTC datetime at transition evaluation. Use as `now()`. |
| `today` | Local date. Use as `today()`. |

### Built-in calls

Only three call shapes are allowed:

- `now()` — current UTC datetime
- `today()` — current local date
- `<root>.has_role('<code>')` — string-literal arg only

Any other function call (`len(...)`, `str(...)`, `re.match(...)`) raises `ValueError` at compile time.

### Examples

```xml
<!-- field comparison -->
<guard expression="subject.margin_pct < 0.15"/>

<!-- composite -->
<guard expression="subject.amount > 1000 AND NOT subject.is_internal"/>

<!-- role check -->
<guard expression="requester.has_role('pricing.admin')"/>

<!-- enum membership via OR chain -->
<guard expression="subject.risk_level == 'risk' OR subject.risk_level == 'negative_risk'"/>
```

---

## When To Use Named Guards vs Inline `guard-expr`

Reach for `<guard code="...">` whenever:
- The same predicate appears on more than one transition.
- The predicate encodes a commercial / compliance threshold (margin floor, credit limit). Named rows make threshold changes a one-line edit.
- A non-developer is likely to want to inspect or change the rule via the Visual Editor.

Reach for inline `guard-expr="..."` only when:
- The predicate is trivially obvious from reading it (`subject.amount > 0`).
- It's used exactly once.
- There's no commercial / compliance threshold inside it.

The mutex hook on `ir.workflow.transition` (see [`services/validators.py`](../src/ede/foundation/workflow/services/validators.py)) rejects any row that sets both — pick exactly one.

---

## Recompile-on-Save

Admins editing a guard policy in the Visual Editor (Phase 3) type the new expression in the `expression_text` field. The `pre.ir.workflow.guard.policy.update` hook runs at write time:

1. Reads the new `expression_text`.
2. Calls `compile_expression(text)` — the same compiler the loader uses.
3. Overwrites `expression_ast` with the result.

If the expression is invalid, the write fails with a clear `ValueError` naming the offence — the admin sees the error in the editor before the row is saved. Tests for this path live in [test_guard_policy.py](../src/tests/foundation/workflow/test_guard_policy.py).

---

## Migration From Raw `<record>` Seeds

If you have an existing workflow expressed as `<ede><data><record model="ir.workflow.*">...</record></data></ede>`:

1. Replace the document root with a single `<workflow>` element carrying `code`, `model`, `field`, `name`.
2. Move each stage `<record>` into a `<stage>` element. The record's `code` becomes the `code` attribute; everything else maps directly.
3. Move each transition `<record>` into a `<transition>` element. Replace `from_stage_id`/`to_stage_id` UUIDs with `from`/`to` stage codes.
4. Lift recurring `guard_ast` strings into `<guards><guard code="..." expression="..."/></guards>` rows; reference them from transitions via `guard="..."`.
5. Move trigger fields (`trigger_type`/`trigger_key`) into the matching `on-command="..." | on-event="..." | on-cron="..."` attribute.
6. Move `action_type`/`action_config` into a child `<action>` element.

Once it parses, delete the old `<record>` files. The data loader routes by root element — `<workflow>` files go to the workflow loader; everything else continues to use the record DSL.

For a worked example, compare [pricing_rate_workflow.xml](../src/domains/logistics/pricing/data/pricing_rate_workflow.xml) (current) against the prior `<ede><data><record>` form in git history before commit `c8a4f2d9e7b1`.

---

## Validation Pipeline

Authoring errors surface in this order:

1. **RelaxNG schema** — structural shape (required attributes, element nesting, attribute value enums). See [workflow.rng](../src/ede/foundation/workflow/data_loader/schemas/workflow.rng). Failures mention line numbers.
2. **Parser cross-checks** — code resolution, mutex enforcement, multi-trigger detection. Failures name the transition / stage / guard code.
3. **Expression compiler** — DSL grammar, identifier whitelist, allowed call shapes. Failures quote the offending expression.
4. **Loader idempotency** — the loader upserts rows by `(definition_id, code)`; re-running is safe.

All three stages run before any `ede.create` or `ede.update` command is dispatched. A bad XML never partially loads.

---

## See also

- [foundation-workflow.md](./foundation-workflow.md) — module overview, models, engine flow.
- [roadmap/foundation/workflow/phase-1-implementation.md](../roadmap/foundation/workflow/phase-1-implementation.md) — phase 1 contract this DSL satisfies.
- [src/tests/foundation/workflow/test_dsl_parser.py](../src/tests/foundation/workflow/test_dsl_parser.py) — parser test cases (handy as additional examples).
- [src/tests/foundation/workflow/test_expression_compiler.py](../src/tests/foundation/workflow/test_expression_compiler.py) — expression grammar coverage.
