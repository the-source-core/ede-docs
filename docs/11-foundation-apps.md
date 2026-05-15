# EDE Framework — Foundation Apps

## Overview

Foundation apps are built-in apps that ship with EDE. They provide core business resources
(users, organizations, countries), authentication, and the presentation system.

```python
# src/ede/foundation/settings.py
ACTIVE_MODULES = ["base", "auth", "presentation"]
```

All foundation apps live in `src/ede/foundation/`.

---

## foundation.base

**Package**: `ede.foundation.base`

The base app provides essential business resources and framework metadata models.

### Models

#### `res.country`

Countries list. Used as a reference in organizations, addresses, etc.

| Field | Type | Notes |
|---|---|---|
| `name` | Char(100) | Country name. Required, unique. |
| `code` | Char(3) | ISO 3166-1 alpha-2/3 code. Required, unique. |

#### `res.organization`

The legal entity / company for a tenant.

| Field | Type | Notes |
|---|---|---|
| `name` | Char(200) | Organization name. Required. |
| `country` | Reference(res.country) | Registered country. |
| `is_active` | Boolean | Default: True. |

Commands:
- `res.organization.change_country` — change the organization's country
- `res.organization.deactivate` — deactivate the organization

#### `res.user`

System users. Authentication is built around this model.

| Field | Type | Notes |
|---|---|---|
| `email` | Char(255) | Login email. Required, unique. |
| `name` | Char(200) | Display name. Required. |
| `password_hash` | Char(255) | bcrypt hash. Readonly via ORM. |
| `is_active` | Boolean | Default: True. |

Commands:
- `res.user.register` — create a user (hashes password automatically)
- `res.user.set_password` — change password (re-hashes)
- `res.user.deactivate` — deactivate user
- `res.user.activate` — re-activate user

Helper function (importable):
```python
from ede.foundation.base.models.user import verify_password

ok = verify_password(plain_password, stored_hash)  # → bool
```

#### `ir.action`

Defines navigation actions (which view to show for a menu item).

| Field | Type | Notes |
|---|---|---|
| `name` | Char(200) | Action display name |
| `model_key` | Char(100) | Target model key |
| `view_type` | Char(50) | Default view type: "list", "form", "kanban" |
| `view_id` | Char(200) | Specific view DSL ID (optional) |
| `domain` | JSON | Default filter domain |

#### `ir.menu`

Navigation menu tree.

| Field | Type | Notes |
|---|---|---|
| `name` | Char(200) | Menu label |
| `parent` | Reference(ir.menu) | Parent menu (nullable = top-level) |
| `action` | Reference(ir.action) | Action to execute on click |
| `sequence` | Integer | Sort order |
| `icon` | Char(100) | Icon identifier |

### Routes (base)

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Health ping (always public) |
| `/api/base/ping` | GET | Framework ping |
| `/api/base/bootstrap` | GET | Full bootstrap payload |
| `/api/res/country` | GET | List countries |
| `/api/res/organization` | GET/POST | List / create organization |
| `/api/res/user` | GET/POST | List / create users |

---

## foundation.auth

**Package**: `ede.foundation.auth`

Provides authentication: login, session management, JWT issuance.

### Models

#### `ir.session`

Tracks active user sessions per tenant. Used for JWT revocation.

| Field | Type | Notes |
|---|---|---|
| `user_id` | UUID | FK to `res.user.record_uuid` |
| `tenant_id` | Char(100) | Tenant this session belongs to |
| `refresh_token_hash` | Char(64) | SHA-256 of refresh token |
| `expires_at` | DateTime | Refresh token expiry (UTC) |
| `revoked` | Boolean | Explicitly revoked flag. Default: False. |
| `os` | Char(100) | Client OS (from User-Agent, optional) |
| `browser` | Char(100) | Client browser (from User-Agent, optional) |

### Services

`SessionService` — see [Authentication docs](08-authentication.md) for full details.

### Routes (auth)

| Endpoint | Method | Auth | Purpose |
|---|---|---|---|
| `/api/auth/login` | POST | public | Login → access + refresh tokens |
| `/api/auth/logout` | POST | user | Revoke current session |
| `/api/auth/refresh` | POST | public | Refresh access token |
| `/api/auth/me` | GET | user | Current user details |

---

## foundation.presentation

**Package**: `ede.foundation.presentation`

Provides the View DSL system and web client bootstrap endpoint.

### Models

#### `PresentationKernel` (`ede.presentation`)

Framework model for view operations.

Commands:
- `presentation.list_views` — returns all registered view IDs and metadata
- `presentation.get_view_plan` — returns the full `RenderPlan` dict for a given `view_id`

### Routes (presentation)

| Endpoint | Method | Auth | Purpose |
|---|---|---|---|
| `/api/presentation/views` | GET | user | List all registered views |
| `/api/presentation/view/{view_id}` | GET | user | Get RenderPlan for a view |
| `/api/presentation/bootstrap` | GET | user | Full frontend bootstrap payload |

### Bootstrap Payload

`GET /api/presentation/bootstrap` returns:
```json
{
    "user": { "id": "...", "name": "...", "email": "..." },
    "tenant_id": "acme",
    "apps": [
        {
            "key": "logistics",
            "name": "Logistics",
            "menus": [...]
        }
    ],
    "actions": { "action-id": { "model_key": "...", "views": [...] } }
}
```

---

## Foundation Model Naming Convention

See also: [docs/foundation-model-naming.md](foundation-model-naming.md)

| Prefix | Namespace | Examples |
|---|---|---|
| `res.*` | Shared business resources | `res.country`, `res.organization`, `res.user` |
| `ir.*` | Internal framework metadata | `ir.session`, `ir.menu`, `ir.action` |
| `ede.*` | Kernel commands only (reserved) | `ede.create`, `ede.update`, `ede.delete` |

**User domain models** do NOT use these prefixes — use `{domain}.{name}`:
`logistics.shipment`, `crm.lead`, `inventory.product`.

---

## Default Data

The base app seeds default data at initialization:

- **Default Organization**: `My Organization`
- **Default Admin User**: Configured via env vars / ede.conf
- **Countries**: Seeded from a standard list (ISO 3166)

---

## Field Change Tracking on `res.organization`

`res.organization.code` is the URL slug used by the frontend workspace router.
When an admin renames it, the frontend URL breaks — field tracking detects the
change and pushes a reload to all connected browser tabs automatically.

```python
# src/ede/foundation/base/models/organization.py

@api.on_event("res.organization.field_changed", track_fields=["code"])
def handle_code_changed(self, event, env) -> None:
    _try_emit(self, "web.client.reload", {
        "reason": "org_slug_changed",
        "org_id": event.payload.get("record_id"),
        "new_slug": event.payload.get("new_value"),
        "tenant_id": event.tenant_id,
    })
```

**No explicit `__ede_track_fields__` declaration is needed.** The `registry.register_model()`
call sees `track_fields=["code"]` on the method and auto-registers the
`pre.res.organization.update` / `post.res.organization.update` hooks.

Data flow:

```
write({"code": "acme-new"})
  → post.res.organization.update hook
  → emits res.organization.field_changed {field="code", old="acme", new="acme-new"}
  → EventWorker: handle_code_changed fires (filtered: field == "code")
  → emits web.client.reload {new_slug="acme-new"}
  → EventWorker: handle_web_client_reload
  → WebPushRegistry.broadcast(tenant_id, msg)
  → SSE stream → browser navigates to new slug URL
```

---

## Loading Order

Foundation apps load **before** user domain apps. Within foundation, order follows
`ACTIVE_MODULES` list:

```python
ACTIVE_MODULES = ["base", "auth", "presentation"]
# loads: foundation.base → foundation.auth → foundation.presentation
```

**Important dependency**: `ir.action` is imported before `ir.menu` (FK dependency).
The `base/models/__init__.py` enforces this order explicitly.

---

## Extending Foundation Models

User-domain apps can reference foundation models but should not modify them.

```python
# In logistics.shipment model
@api.model("logistics.shipment")
class Shipment(DomainModel):
    created_by = fields.Reference("res.user")        # reference to res.user
    destination_country = fields.Reference("res.country")
```

**Never** add commands to `res.*` or `ir.*` models in user-domain apps. If you need
custom behavior on, say, `res.user`, add a new model that references it and put
your logic there.

---

## Why Shared Foundation Apps?

### Shared resources prevent model fragmentation

Without foundation apps, every domain would define its own user, organization, and country
models. `logistics.user` and `crm.user` would be different tables, different FK targets,
and impossible to join. Authentication would need to pick one or duplicate logic across
domains.

`res.user`, `res.organization`, and `res.country` exist exactly once. Every domain
references them. User authentication, session management, and tenant identity all operate
on the same records that domain models reference via `fields.Reference("res.user")`.

### `ir.*` models are the framework's own persistence

`ir.session` stores authentication state. `ir.menu` and `ir.action` store navigation
structure. These are internal framework concerns — they use the same `DomainModel`
infrastructure as application models, but their keys (`ir.*`) signal that they are
owned by the framework and should not be modified by user-domain apps.

### Foundation as a stable platform contract

The foundation apps define the contract between the framework and the application:
"Every system built on EDE has users, organizations, and countries. Auth works this way.
Sessions work this way." User-domain apps can rely on these resources being present and
use them freely as reference targets.

This is the same design principle as a standard library: you don't rewrite `list` or
`dict` for each project. You build on them. Foundation apps are EDE's standard library
for business domain primitives.
