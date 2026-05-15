# Permissions — RBAC + ABAC Reference

EDE uses a two-layer authorization model:

- **RBAC** (Role-Based Access Control) — *who* can perform an action on *which model*
- **ABAC** (Attribute-Based Access Control) — *on which specific records*, expressed as a domain filter on the permission itself

Both layers are enforced automatically on every command via pre-hooks registered at
boot time. You define permissions in a CSV data file; no code changes are needed to
protect a model.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Permission Record](#2-permission-record)
3. [Actions](#3-actions)
4. [Defining Permissions (CSV)](#4-defining-permissions-csv)
5. [Assigning Permissions to Roles](#5-assigning-permissions-to-roles)
6. [constraint_schema — Domain Filter Style](#6-constraint_schema--domain-filter-style)
7. [$principal Variables](#7-principal-variables)
8. [Domain Filter Grammar](#8-domain-filter-grammar)
9. [Full Examples](#9-full-examples)
10. [How the Check Works (Decision Flow)](#10-how-the-check-works-decision-flow)
11. [Bypass List](#11-bypass-list)
12. [Time-Limited Grants](#12-time-limited-grants)
13. [env.sudo() — System Principal](#13-envsudo--system-principal)

---

## 1. Core Concepts

```
res.user ──── ir.rbac.role.binding ────► ir.rbac.role
                                              │
                                    M2M join table
                                              │
                                              ▼
                                   ir.rbac.permission
                                     resource = "res.user"
                                     action   = "read"
                                     constraint_schema = null   ← or a domain filter
```

- A **permission** is one `resource × action` pair.
- A **role** is a named collection of permissions, with optional inheritance via `parent_id`.
- A **binding** links a user to a role, optionally scoped to an org unit or branch.
- At request time: user → bindings → roles (+ parent chain) → permissions → ABAC filter.

---

## 2. Permission Record

```
ir.rbac.permission
──────────────────
name              Char(150)   Human-readable label
code              Char(100)   Unique identifier  e.g. "internal_user.res.user.read"
resource          Char(100)   Model key          e.g. "res.user"
action            Enum        create | read | update | delete | execute
constraint_schema Text        Domain filter JSON (optional). Empty = unconditional.
description       Text        Free-text explanation
is_active         Boolean     Soft-disable without deleting
```

Permissions are **stable configuration** — they are created via the
`data/rbac_permissions.csv` seed file, not via application code or API calls.

---

## 3. Actions

| Action | Triggers on |
|---|---|
| `create` | `ede.create` |
| `read` | `ede.search`, `ede.read_one`, `ede.count`, `ede.read_group` |
| `update` | `ede.update` |
| `delete` | `ede.delete` |
| `execute` | Every custom model command (e.g. `ir.approval.case.decide`) |

All custom commands on a model share the single `execute` permission for that model.
If you need finer control between commands, split them into separate models.

---

## 4. Defining Permissions (CSV)

Place a file named `ir.rbac.permission.csv` inside your app's `data/` directory and declare
it in `__manifest__.py` under the `"data"` key.

```csv
id,name,code,resource,action,role_id/id,domain
myapp.p_crm_lead_read,Read Leads,crm.lead.read,crm.lead,read,rbac.role_portal_user,
myapp.p_crm_lead_create,Create Leads,crm.lead.create,crm.lead,create,rbac.role_internal_user,
myapp.p_crm_lead_delete,Delete Leads,crm.lead.delete,crm.lead,delete,rbac.role_system_admin,
myapp.p_crm_lead_read_own,Read Own Leads,crm.lead.read.own,crm.lead,read,rbac.role_portal_user,"[[""requester_id"",""="",""$principal.user_id""]]"
```

**Column reference:**

| Column | Required | Notes |
|---|---|---|
| `id` | ✓ | External ID — `<module>.<name>` (must be unique) |
| `name` | ✓ | Human-readable label |
| `code` | ✓ | Unique identifier, convention: `{resource}.{action}` |
| `resource` | ✓ | Model key being protected e.g. `crm.lead` |
| `action` | ✓ | `create` \| `read` \| `update` \| `delete` \| `execute` |
| `role_id/id` | — | External ID of the `ir.rbac.role` to attach this permission to. Empty = no role (permission exists but is unassigned). The `/id` suffix tells the loader to resolve this as a ref. |
| `domain` | — | ABAC domain filter JSON (see §6). Empty = unconditional. |

**`role_id/id` suffix**: CSV columns ending in `/id` are resolved as external ID references —
the loader looks up the value in `ir.data.reference` and substitutes the record UUID. Without
the `/id` suffix, the value is treated as a plain string.

**Code convention:** `{resource_dotted}.{action}` — the code tells you exactly what is
protected without opening the roles config.

---

## 5. Assigning Permissions to Roles

File: `src/ede/foundation/rbac/data/rbac_role_permissions.xml`

```xml
<ede><data noupdate="1">

  <record id="rbac.rp_portal_user_read" model="ir.rbac.role"
          command="ir.rbac.role.assign.permission">
    <field name="record_id" ref="rbac.role_portal_user"/>
    <field name="permission_id" ref="rbac.p_portal_user_read"/>
  </record>

  <record id="rbac.rp_internal_case_execute" model="ir.rbac.role"
          command="ir.rbac.role.assign.permission">
    <field name="record_id" ref="rbac.role_internal_user"/>
    <field name="permission_id" ref="rbac.p_internal_case_execute"/>
  </record>

</data></ede>
```

Role inheritance means you only assign a permission to the **lowest role that should
have it**. Child roles inherit automatically at evaluation time by walking the
`parent_id` chain.

```
portal_user   ← gets: res.user read, ir.approval.case read + create
    └── internal_user  ← gets: res.user create + update, ir.approval.case execute
            └── system_admin  ← gets: delete + all admin permissions
```

---

## 6. `constraint_schema` — Domain Filter Style

`constraint_schema` is an **optional** field on a permission. It uses the exact same
domain filter syntax as `ede.search`, extended with `$principal.*` variable references.

When `constraint_schema` is empty or `null`: the permission is **unconditional** — the
role-level check alone decides access.

When present: the permission also requires that the **target record** matches the filter.

```
Role check passes?
    YES → is constraint_schema empty?
            YES → ALLOW
            NO  → evaluate domain filter against the record
                    passes? → ALLOW
                    fails?  → DENY
    NO  → DENY
```

`constraint_schema` is stored as a JSON string in the database column.

---

## 7. `$principal` Variables

`$principal` represents the authenticated user's enriched context, resolved fresh from
the database at the start of each request (then cached for the duration of that request).

| Variable | Type | Description |
|---|---|---|
| `$principal.user_id` | `str` | UUID of the authenticated user |
| `$principal.tenant_id` | `str` | Current tenant identifier |
| `$principal.role_codes` | `list[str]` | All role codes held by the user, including transitively inherited roles |
| `$principal.active_organization_id` | `Optional[str]` | UUID of the organization the user has currently selected. `None` when no active org. |
| `$principal.allowed_organization_ids` | `list[str]` | UUIDs the user is permitted to operate as (`res.user.organization_ids` ∪ home org). Index 0 is always the home org. |
| `$principal.org_ids` | `list[str]` | UUIDs from role-bindings with `scope_type='ORG'` |
| `$principal.branch_ids` | `list[str]` | UUIDs from role-bindings with `scope_type='BRANCH'` |
| `$principal.department_ids` | `list[str]` | UUIDs from role-bindings with `scope_type='DEPARTMENT'` |
| `$principal.org_unit_ids` | `list[str]` | Union of `org_ids ∪ branch_ids ∪ department_ids`. Retained for back-compat. |

**How the split scopes are populated:**

A user may have multiple bindings:

```
binding 1: role=internal_user, scope_type=BRANCH, scope_id="branch-mumbai-uuid"
binding 2: role=internal_user, scope_type=BRANCH, scope_id="branch-pune-uuid"
binding 3: role=regional_lead, scope_type=ORG,    scope_id="org-acme-india-uuid"
```

→ `$principal.branch_ids = ["branch-mumbai-uuid", "branch-pune-uuid"]`
→ `$principal.org_ids    = ["org-acme-india-uuid"]`
→ `$principal.org_unit_ids = ["branch-mumbai-uuid", "branch-pune-uuid", "org-acme-india-uuid"]` (union)

If a user has only GLOBAL or TENANT bindings, all four lists are empty (`[]`).

**Active-organization context:**

`$principal.active_organization_id` mirrors `env.active_organization_id`, populated from the JWT
`active_organization_id` claim by `AuthMiddleware`. It is `None` when no active org is set (system
principals, migrations, users that haven't picked an org yet). Domain-filter leaf semantics treat
`None` as no-match for `=` / `in`, so permissions that require an active-org fail closed naturally.

**Allowed vs scope-bound:** `allowed_organization_ids` is the **white-list** (`res.user.organization_ids`).
`org_ids` is a **subset** — only orgs where the user has a role-binding. Use the right one:

- "any org the user can switch to" → `allowed_organization_ids`
- "any org the user has a role in" → `org_ids`

### Worked examples

**Example I — active-org filter:** Only the user's currently selected org's records.

```python
'[["organization_id", "=", "$principal.active_organization_id"]]'
```

**Example J — branch + active-org composite:** Records in the active org AND in a branch the user has a role in.

```python
'["&", ["organization_id", "=", "$principal.active_organization_id"], ["branch_id", "in", "$principal.branch_ids"]]'
```

---

## 8. Domain Filter Grammar

Same grammar used by `ede.search` `domain` parameter.

### Leaf condition

```
["field", "operator", value]
```

| Operator | Behaviour |
|---|---|
| `=` | exact match |
| `!=` | not equal |
| `in` | value is in list |
| `not in` | value is not in list |
| `<` `<=` `>` `>=` | numeric comparison |
| `like` | substring match (case-sensitive) |
| `ilike` | substring match (case-insensitive) |
| `not like` | substring not present (case-sensitive) |
| `not ilike` | substring not present (case-insensitive) |

### Logical operators

```python
# AND — default, just list conditions together
[cond1, cond2, cond3]

# explicit AND
["&", cond1, cond2]

# OR
["|", cond1, cond2]

# NOT
["!", cond1]

# nesting
["|", ["&", cond1, cond2], cond3]
```

### `$principal.*` in values

Any string value starting with `$principal.` is resolved from the current user's
principal before the filter is evaluated.

```python
["owner_id", "=", "$principal.user_id"]
# becomes at evaluation time:
["owner_id", "=", "abc-123-uuid"]
```

```python
["org_unit_id", "in", "$principal.org_unit_ids"]
# becomes:
["org_unit_id", "in", ["branch-mumbai-uuid", "branch-pune-uuid"]]
```

---

## 9. Full Examples

### Example A — Unconditional read (no constraint)

Any `internal_user` can read any `res.country` record.

```csv
rbac.p_internal_country_read,Read Countries,internal_user.res.country.read,res.country,read,View countries
```

`constraint_schema` = `null` → no record-level filter, role check alone decides.

---

### Example B — Only read your own user record

```csv
rbac.p_portal_own_profile,Read Own Profile,portal_user.res.user.read_own,res.user,read,View own profile only
```

```
constraint_schema = '[["record_uuid", "=", "$principal.user_id"]]'
```

**Evaluation:**

```
Priya (user_id = "priya-uuid") reads her own profile:
  record.record_uuid = "priya-uuid"
  "$principal.user_id" resolves to "priya-uuid"
  "priya-uuid" = "priya-uuid"  →  ✓ ALLOW

Priya reads Ravi's profile:
  record.record_uuid = "ravi-uuid"
  "ravi-uuid" = "priya-uuid"  →  ✗ DENY
```

---

### Example C — Only records in the user's org scope

```
constraint_schema = '[["org_unit_id", "in", "$principal.org_unit_ids"]]'
```

**Evaluation:**

```
Priya has branch bindings: ["mumbai-uuid", "pune-uuid"]

Read record where org_unit_id = "mumbai-uuid":
  "mumbai-uuid" in ["mumbai-uuid", "pune-uuid"]  →  ✓ ALLOW

Read record where org_unit_id = "london-uuid":
  "london-uuid" in ["mumbai-uuid", "pune-uuid"]  →  ✗ DENY
```

---

### Example D — Own record OR in scope (OR)

A manager can see records they created, or records in their branch.

```
constraint_schema = '["|", ["created_uid", "=", "$principal.user_id"], ["org_unit_id", "in", "$principal.org_unit_ids"]]'
```

**Evaluation:**

```
Record A: created_uid = "priya-uuid", org_unit_id = "london-uuid"
  created_uid = priya?  ✓  →  OR short-circuits  →  ALLOW

Record B: created_uid = "ravi-uuid", org_unit_id = "mumbai-uuid"
  created_uid = priya?  ✗
  org_unit_id in scope? ✓  →  ALLOW

Record C: created_uid = "ravi-uuid", org_unit_id = "london-uuid"
  created_uid = priya?  ✗
  org_unit_id in scope? ✗  →  DENY
```

---

### Example E — Status constraint (static value)

Only allow updating contracts that are still in `DRAFT` state.

```
constraint_schema = '[["state", "=", "DRAFT"]]'
```

```
Record state = "DRAFT"    →  ✓ ALLOW
Record state = "APPROVED" →  ✗ DENY
```

---

### Example F — Scoped AND active (AND)

```
constraint_schema = '[["org_unit_id", "in", "$principal.org_unit_ids"], ["is_active", "=", true]]'
```

Both conditions must pass.

---

### Example G — NOT (exclude a specific status)

```
constraint_schema = '[["!", ["state", "=", "CANCELLED"]]]'
```

Allow as long as the record is not in CANCELLED state.

---

### Example H — Nested OR inside AND

```
constraint_schema = '[["is_active", "=", true], ["|", ["owner_id", "=", "$principal.user_id"], ["org_unit_id", "in", "$principal.org_unit_ids"]]]'
```

Must be active AND (owned by me OR in my scope).

---

## 10. How the Check Works (Decision Flow)

```
incoming command (e.g. ede.update on res.user)
          │
          ▼
pre-hook fires (registered by permission_registry at boot)
          │
          ▼
AuthorizationService.require(resource="res.user", action="update")
          │
          ├─ resource in BYPASS_MODELS?  →  return (no check)
          │
          ├─ env.principal.is_system?  →  return (sudo bypass — no log written)
          │
          ├─ env.principal has user_id?
          │     NO  →  DENY  (no authenticated principal)
          │
          ├─ PrincipalEnricher.load_principal(user_id)
          │     → bindings → role chain → org_unit_ids
          │     → cached on env for this request
          │
          ├─ get_permissions_for_roles(role_ids)   via M2M join
          ├─ get_active_grants(user_id)             time-limited elevations
          │
          ├─ filter: resource="res.user" + action="update"
          │     none match  →  DENY  (no matching permission)
          │
          └─ for each matching permission:
                constraint_schema empty?
                    YES  →  ALLOW  ✓  (write decision log, return)
                    NO   →  evaluate_constraint(schema, enriched, record)
                              passes?  →  ALLOW  ✓
                              fails?   →  try next permission
                all failed  →  DENY  ✗

ALLOW  →  write ir.rbac.decision.log (decision=ALLOW)  →  command proceeds
DENY   →  write ir.rbac.decision.log (decision=DENY)   →  raise PermissionDeniedError
                                                             → HTTP 403
```

---

## 11. Bypass List

These models skip all authorization checks. They are written internally by the
framework and must never block themselves.

| Model | Reason |
|---|---|
| `ir.session` | Login endpoint has no principal yet |
| `ir.rbac.decision.log` | Written by the auth service itself |
| `ir.rbac.binding.change.log` | Written by role binding hooks |
| `ir.approval.decision` | Written by approval service internally |
| `ir.approval.event.log` | Written by approval service internally |

---

## 12. Time-Limited Grants

For temporary elevation without changing role bindings, use `ir.rbac.access.grant`:

```
ir.rbac.access.grant
────────────────────
user_id       → the user receiving the temporary permission
permission_id → specific ir.rbac.permission to grant
expires_at    → grant is ignored after this datetime
resource_id   → (optional) restrict to one specific record UUID
reason        → audit note
```

Grants are collected alongside role-derived permissions during the authorization
check. A grant scoped to a `resource_id` only passes if `resource_id` matches the
record being accessed.

**Example use cases:**

- Give a contractor `update` on a single contract record for 30 days
- Temporarily elevate a user to approve a specific case they would not normally see
- Break-glass access for an admin to a single sensitive record

Grants do not inherit — they are exact permission grants with no role chain expansion.

---

## 13. `env.sudo()` — System Principal

Some operations run without an authenticated user session: migration-time data seeding,
background workers, and internal service calls. These need to bypass RBAC without creating
a fake DB user.

`env.sudo()` returns a shallow-cloned `Env` carrying a **system sentinel principal**:

```python
{"user_id": "__system__", "is_system": True}
```

`AuthorizationService.require()` checks `is_system` before anything else and returns
immediately — no role lookup, no decision log written.

### Usage

```python
# In a service or background worker:
system_env = env.sudo()
system_env.models["res.country"].search([])   # no RBAC check

# Inline for a single operation:
env.sudo().dispatch(Command(name="ede.create", model_key="ir.rbac.role", payload={"values": {...}}))
```

The migrate command uses this automatically — `DataLoader` always receives `env.sudo()`,
so seed data loads without a real principal.

### What sudo() does NOT do

- It does **not** create a `res.user` record named `__system__`.
- It does **not** write to `ir.session` or any other table.
- It does **not** affect `created_uid` / `updated_uid` — those fields are set to `NULL`
  when the system principal is active (since `"__system__"` is not a valid FK to `res_user`).
- It does **not** bypass model lifecycle hooks — `pre.*` / `post.*` hooks still fire.

### When to use

| Context | Use sudo()? |
|---|---|
| Migration data seeding (`ede migrate upgrade`) | ✓ automatic |
| Background worker processing queued jobs | ✓ yes |
| Internal service calling another model on behalf of the system | ✓ yes |
| HTTP request handler (user is logged in) | ✗ no — use the real principal |
| Tests that don't care about RBAC | ✓ acceptable |

### Security note

`sudo()` is a **trust boundary** — call it only in code paths that run outside the normal
HTTP request cycle (workers, migrations, internal services). Never call it on an `env`
that was derived from a user's HTTP request to skip a permission check you don't want to
deal with.
