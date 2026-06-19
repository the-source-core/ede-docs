# Record Rules

Per-record visibility filters expressed as data. Layer them on top of the permission engine to say "this user can read posts, but only the ones they authored."

```xml
<record id="rule_own_posts_only" model="ir.rbac.record.rule">
    <field name="name">Authors see only their own posts</field>
    <field name="model_id" ref="base.model_blog_post"/>
    <field name="role_ids">
        <ref id="role_author" op="link"/>
    </field>
    <field name="perm_read">true</field>
    <field name="domain">[("author_id", "=", "$principal.user_id")]</field>
</record>
```

Once seeded, every `search`, `read_one`, `count`, and `read_group` for that model automatically appends the rule's domain — server-side. Per-record writes (create / update / delete) are gated by the matching `perm_*` flag.

---

## What you get

-   **`ir.rbac.record.rule`** — the rule record. A rule targets one model (`model_id`), carries one `domain` filter, opts into one or more operations via the `perm_read` / `perm_create` / `perm_update` / `perm_delete` booleans, and applies to the roles in `role_ids`.
-   **`RecordRuleEngine`** — composes the active rules into a single filter at each read callsite. `evaluate_filter(user_id=, model_key=, action=)` returns the combined `DomainFilter`; `evaluate_record(...)` gates a single write.
-   **Eight read callsites** — `search`, `read_one`, `count`, `read_group`, plus the four relational read paths, all append the composed filter automatically.
-   **Operation scoping** — a rule applies to any combination of read / create / update / delete via its four boolean flags.
-   **Global vs role-scoped** — a rule with empty `role_ids` is GLOBAL (applies to every user). Role-scoped rules combine as `GLOBAL AND ((ROLE_A rules) OR (ROLE_B rules))` for multi-role users.
-   **Admin UI** — Settings → Security → Record Rules.

## How to use it

### Author a read rule

A rule targets a model through `model_id`, a `fields.Reference` to the `ir.model` registry row. In XML, resolve it with the `base.model_<table>` external id:

```xml
<record id="rule_own_org_only" model="ir.rbac.record.rule">
    <field name="name">Users only see records in their org</field>
    <field name="model_id" ref="base.model_blog_post"/>
    <field name="role_ids">
        <ref id="role_author" op="link"/>
    </field>
    <field name="perm_read">true</field>
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
| `$principal.org_ids` | list[uuid] | Org IDs in scope. |
| `$principal.branch_ids` | list[uuid] | Branch (org-unit) IDs. |
| `$principal.department_ids` | list[uuid] | Department IDs. |

### Author a write rule

Set the operation flags you want the rule to gate. A rule with only `perm_update` true filters update but leaves reads untouched:

```xml
<record id="rule_only_owner_can_update" model="ir.rbac.record.rule">
    <field name="name">Only the author can edit a post</field>
    <field name="model_id" ref="base.model_blog_post"/>
    <field name="role_ids">
        <ref id="role_author" op="link"/>
    </field>
    <field name="perm_update">true</field>
    <field name="perm_delete">true</field>
    <field name="domain">[("author_id", "=", "$principal.user_id")]</field>
</record>
```

If a write targets a record the rule's domain excludes, the operation is denied.

### Bypass rules in trusted code

`env.sudo()` returns an env with the system principal, which skips all record rules and RBAC. Use it only for trusted server-side jobs and migrations:

```python
# No record rules applied — system principal.
posts = env.sudo().models["blog.post"].search([])
```

Use sparingly — `sudo()` is the kernel's escape hatch and bypasses the entire authorization layer.

### Inspect the filter a user would get

```python
from ede.foundation.base.services.record_rule_engine import RecordRuleEngine

flt = RecordRuleEngine(env).evaluate_filter(
    user_id=user.id,
    model_key="blog.post",
    action="read",
)
```

`evaluate_filter` returns the composed domain that the read callsites apply for that user, model, and action — useful for debugging why a user does or doesn't see a row.

## How it composes with other features

-   **[Security & Authorization](security.md)** — record rules are the row-level layer; the permission engine is the operation-level layer.
-   **[Auth — Login & Sessions](auth.md)** — `$principal.active_organization_id` comes from the JWT switch-org endpoint.

## Reference

| Concept | Where it lives |
|---|---|
| `ir.rbac.record.rule` | `src/ede/foundation/base/models/record_rule.py` |
| `RecordRuleEngine` | `src/ede/foundation/base/services/record_rule_engine.py` |
| `AuthorizationService` | `src/ede/foundation/base/services/authorization_service.py` |
| Filter application | `src/ede/core/orm/` (8 read callsites) |
