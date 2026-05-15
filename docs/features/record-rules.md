# Record Rules

Per-record visibility filters expressed as data. Layer them on top of the permission engine to say "this user can read sale orders, but only the ones in their region."

```xml
<record id="rule_my_region_only" model="ir.rbac.record.rule">
    <field name="name">Sales reps see only their own region</field>
    <field name="model_key">crm.sale.order</field>
    <field name="role_id" ref="role_sales_rep"/>
    <field name="op">read</field>
    <field name="domain">[("region_id", "=", "$principal.region_id")]</field>
</record>
```

Once seeded, every `search`, `read_one`, `count`, and `read_group` for that model automatically appends the rule's domain — server-side. Per-record writes (update / delete) are likewise gated.

---

## What you get

-   **`ir.rbac.record.rule`** — the rule record.
-   **`RecordRuleEngine`** — composes the active rules into a single filter at each read callsite.
-   **`apply_record_rules_filter`** — called at all 8 ORM read callsites (search, read_one, count, read_group, plus the 4 relational read paths).
-   **`AuthorizationService._enforce_record_rules`** — per-record gate on writes; raises with `reason="record_rule_violation"`.
-   **Operation scoping** — rules apply to one or more of `read`, `write`, `delete`, `create`.
-   **Role-OR composition** — when a user holds multiple roles, the rules combine as `GLOBAL AND ((ROLE_A_OR_BLOCK) OR (ROLE_B_OR_BLOCK))`.
-   **Admin UI** — Settings → Security → Record Rules.

## How to use it

### Author a read rule

```xml
<record id="rule_own_org_only" model="ir.rbac.record.rule">
    <field name="name">Users only see records in their org</field>
    <field name="model_key">crm.sale.order</field>
    <field name="role_id" ref="role_sales_rep"/>
    <field name="op">read</field>
    <field name="domain">[("organization_id", "=", "$principal.active_organization_id")]</field>
</record>
```

The `domain` string is parsed as the standard domain-filter DSL. `$principal.*` variables are substituted from `env.principal` at evaluation time.

### Available `$principal.*` variables

| Variable | Type | Meaning |
|---|---|---|
| `$principal.user_id` | uuid | Current user. |
| `$principal.active_organization_id` | uuid | Active org from the JWT. |
| `$principal.allowed_organization_ids` | list[uuid] | Orgs the user can switch into. |
| `$principal.org_ids` | list[uuid] | Org-unit IDs of the active org. |
| `$principal.branch_ids` | list[uuid] | Branch IDs. |
| `$principal.department_ids` | list[uuid] | Department IDs. |

### Author a write rule

```xml
<record id="rule_only_owner_can_update" model="ir.rbac.record.rule">
    <field name="model_key">blog.post</field>
    <field name="role_id" ref="role_author"/>
    <field name="op">write</field>
    <field name="domain">[("author_id", "=", "$principal.user_id")]</field>
</record>
```

If the rule does not match a target record, the write is denied with `PermissionDeniedError(reason="record_rule_violation")`.

### Bypass rules in trusted code

```python
with env.with_sudo():
    # No record rules applied; for trusted server-side jobs.
    orders = env.models["crm.sale.order"].search([])
```

Use sparingly — `with_sudo()` is the kernel's escape hatch and bypasses all RBAC.

### Inspect what rules apply to a user

```python
from ede.foundation.security.services.authorization_service import AuthorizationService

rules = AuthorizationService.from_env(env).rules_for_model("crm.sale.order")
```

The admin UI exposes the same view under each user's profile.

## How it composes with other features

-   **[Security & Authorization](security.md)** — record rules are the row-level layer; the permission engine is the operation-level layer.
-   **[Active-Organization](auth.md)** — `$principal.active_organization_id` comes from the JWT switch-org endpoint.

## Reference

| Concept | Where it lives |
|---|---|
| `ir.rbac.record.rule` | `src/ede/foundation/security/models/record_rule.py` |
| `RecordRuleEngine` | `src/ede/foundation/security/services/record_rule_engine.py` |
| Filter application | `src/ede/core/orm/` (8 callsites — search, read_one, count, read_group + relational) |
