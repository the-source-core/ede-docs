# Security & Authorization

Declarative role-based access control with company-scoped records and per-role row-level filters. Every command goes through the permission engine; every read goes through the record-rule engine.

```python
# Grant a role to a user, scoped to one organization.
env.models["ir.rbac.binding"].create({
    "user_id": alice.id,
    "role_id": sales_manager.id,
    "organization_id": acme.id,
})

# Inside a command handler, the engine has already gated this call.
@api.on_command("sale.order.confirm")
def confirm(self, cmd):
    self.write({"state": "confirmed"})
    # PermissionDeniedError already raised upstream if alice
    # lacks sale.order:confirm — handler never runs.
```

---

## What you get

-   **RBAC**: `ir.rbac.role`, `ir.rbac.permission`, `ir.rbac.binding`.
-   **Record rules**: `ir.rbac.record.rule` — declarative row-level filters scoped to a role.
-   **Active-organization context**: every JWT carries `active_organization_id`; the ORM filters records to that org.
-   **Company-scope decorator**: `@api.model("...", company_scope="strict"|"optional"|"multi")` auto-injects the org FK and the filter.
-   **`$principal.*` ABAC variables**: `user_id`, `active_organization_id`, `allowed_organization_ids`, `org_ids`, `branch_ids`, `department_ids` — usable in record-rule domain filters.
-   **Allowed-org write guard**: pre-create / pre-update hooks reject writes that target an org outside the user's allowed set.
-   **Audit log**: `ir.rbac.decision.log` (per-request decisions) and `ir.rbac.binding.change.log` (binding changes).
-   **`AuthorizationService`** — `can(env, resource, action)` for explicit gating in code; `_enforce_record_rules` for per-record checks.

---

## How to use it

### Declare a company-scoped model

```python
@api.model("sale.order", company_scope="strict")
class SaleOrder(DomainModel):
    name = fields.Char(required=True)
    # organization_id field is auto-injected as required FK to res.organization.
```

`company_scope="strict"` injects an `organization_id` FK (required, indexed), filters every `search/read` to the user's active organization, and stamps `organization_id` on every create. Use `"optional"` if some records are tenant-global; `"multi"` injects `organization_ids` M2M for cross-org sharing.

### Add a record rule

```xml
<!-- src/domains/sales/data/record_rules.xml -->
<ir.rbac.record.rule id="sales_own_orders">
    <field name="model">sale.order</field>
    <field name="role_id" ref="role_sales_rep"/>
    <field name="domain">[("owner_id", "=", $principal.user_id)]</field>
</ir.rbac.record.rule>
```

Users with the `sales_rep` role only see orders they own. The engine composes rules across roles as `GLOBAL AND ((ROLE_A_OR_BLOCK) OR (ROLE_B_OR_BLOCK))`.

### Gate a command explicitly

```python
from ede.foundation.security.services.authorization_service import AuthorizationService

if not AuthorizationService.can(env, "sale.order", "confirm"):
    raise PermissionDeniedError("sale.order:confirm")
```

For most cases you don't need this — the engine auto-gates commands declared in `ir.rbac.permission` rows. Use the explicit form when the check straddles models or is conditional.

### Switch the active organization

`POST /api/auth/switch-organization` re-issues the JWT with a different `active_organization_id`, provided the target org is in the user's `allowed_organization_ids`. Every subsequent request is scoped to the new org.

### Inspect "why was I denied"

Every denial writes a row to `ir.rbac.decision.log` with a `reason` code:

-   `permission_missing` — no role grants the requested action.
-   `record_rule_violation` — record visible to nobody under this role's rules.
-   `wrong_organization` — write targets an org outside `allowed_organization_ids`.

Filter the audit view by `reason = "record_rule_violation"` to debug rule issues.

---

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `SECURITY_DECISION_LOG_RETAIN_DAYS` | `90` | Rolling retention for `ir.rbac.decision.log`. |

---

## Reference

-   Source: `src/ede/foundation/security/`
-   `AuthorizationService`: `src/ede/foundation/security/services/authorization_service.py`
-   `RecordRuleEngine`: `src/ede/foundation/security/services/record_rule_engine.py`
-   Architecture: [Permissions](../18-permissions.md), [Authentication](../08-authentication.md).
-   Guide: [Tutorial — Permissions & Security](../tutorial/06-permissions.md).
