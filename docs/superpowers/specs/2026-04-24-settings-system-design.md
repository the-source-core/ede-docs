# Settings System Design
**Date:** 2026-04-24
**Status:** Approved for implementation

---

## Context

EDE currently has no database-backed settings layer. All configuration is file/environment-based via Pydantic (`FoundationSettings`). As the platform grows, modules need runtime-configurable settings that admins can change through the UI without touching config files — e.g., toggling features per company, selecting a default email connector, enabling signup policies system-wide.

The goal is a **pluggable, XML-driven settings system** where any module can introduce settings by dropping an XML file in `views/`, and those settings automatically appear in the **General Settings** screen. No new Python code is required to add settings to an existing module.

---

## Architecture Overview

```
Module views/*_settings.xml
        │
        ▼
  SettingsRegistry (boot)
  (parses <settings> root files,
   skipped by ViewRegistry)
        │
        ▼
  SettingsService
  (resolves: org → system → XML default)
        │
        ├── GET /api/settings          → module nav list
        ├── GET /api/settings/{module} → schema + current values
        └── PATCH /api/settings/{module} → validate + save + audit
                │
                ▼
          ir.config  (key-value store)
          ir.config.log  (audit trail)
                │
                ▼
        Frontend SettingsPage
        (schema-driven renderer,
         no hardcoded field names)
```

---

## 1. Data Models

### `ir.config`

New `DomainModel` in `src/ede/foundation/base/models/ir_config.py`.

| Column | Type | Notes |
|---|---|---|
| `key` | `Char(255)` | Dotted key e.g. `email.feature.notifications` |
| `value` | `Char(8000)` (nullable) | Serialized plaintext value; `null` when `encrypted_value` is set |
| `encrypted_value` | `Char(8000)` (nullable) | Fernet-encrypted, for `secret="true"` fields |
| `scope` | `Selection(["system", "organization"])` | Determines fallback behavior |
| `organization_id` | `Reference("res.organization")` (nullable) | Null for system-scope rows |

**Unique constraint**: `(key, scope, organization_id)` — one row per key per scope per org.

**Value serialization**: booleans as `"true"`/`"false"`, integers as strings, `many2one` as `record_uuid` string.

### `ir.config.log`

New `DomainModel` in `src/ede/foundation/base/models/ir_config_log.py`.

| Column | Type | Notes |
|---|---|---|
| `key` | `Char(255)` | Setting key that changed |
| `old_value` | `Char(8000)` (nullable) | Previous value (`"••••••"` for secrets) |
| `new_value` | `Char(8000)` | New value (`"••••••"` for secrets) |
| `scope` | `Selection(["system", "organization"])` | |
| `organization_id` | `Reference("res.organization")` (nullable) | |
| `changed_by` | `Char(255)` | `principal["sub"]` from `env.principal` |

Auto fields (`created_at_utc`) provide the timestamp.

**Critical files to modify:**
- `src/ede/foundation/base/models/__init__.py` — register both new models
- `src/ede/foundation/base/__manifest__.py` — add settings XML to `data`
- Run `ede migrate generate` to create migrations for both models

---

## 2. Settings XML DSL

Settings files live in `views/` alongside view XMLs, named `views/{module}_settings.xml`. The root element is `<settings>`, not `<ede>` — this distinguishes them from view files.

**`ViewRegistry`** is updated to skip files whose root XML element is `<settings>` (currently it would fail on them anyway, but the skip should be explicit).

### Schema

```xml
<settings module="email" label="Email" icon="mail" category="Services" order="20">

  <section string="Outgoing Mail">
    <field name="email.default_connector"
           type="many2one"
           model="ir.connector"
           scope="organization"
           label="Default Outgoing Email"
           domain="[('category','=','email')]"
           requires_migration="true"/>
  </section>

  <section string="Features">
    <field name="email.feature.notifications"
           type="boolean"
           scope="organization"
           label="Enable Email Notifications"
           default="true"/>

    <field name="email.notification_reply_to"
           type="char"
           scope="organization"
           label="Notification Reply-To Email"
           depends_on="email.feature.notifications"/>

    <field name="email.max_attachment_mb"
           type="integer"
           scope="system"
           label="Max Attachment Size (MB)"
           default="25"/>

    <field name="email.smtp_password"
           type="char"
           scope="organization"
           label="SMTP Password"
           secret="true"/>
  </section>

  <section string="Advanced">
    <field name="email.send_retry_max"
           type="integer"
           scope="system"
           label="Max Send Retries"
           default="3"
           readonly="true"/>
  </section>

</settings>
```

### `<settings>` root attributes

| Attribute | Required | Description |
|---|---|---|
| `module` | Yes | Unique module key (e.g. `email`, `storage`, `base`) |
| `label` | Yes | Display name in sidebar nav |
| `icon` | No | Lucide icon name |
| `category` | No | Groups modules in sidebar (e.g. `Services`, `System`, `Company`) |
| `order` | No | Integer, controls sidebar ordering within category |

### `<field>` attributes

| Attribute | Required | Description |
|---|---|---|
| `name` | Yes | Dotted key stored in `ir.config.key` |
| `type` | Yes | `boolean`, `integer`, `char`, `selection`, `many2one` |
| `scope` | Yes | `system` or `organization` |
| `label` | Yes | Display label |
| `default` | No | Default value (string-serialized) if no `ir.config` row exists |
| `model` | many2one only | Target model key |
| `domain` | many2one only | Domain expression string to filter options |
| `secret` | No | `"true"` — value stored in `encrypted_value`, returned masked |
| `requires_migration` | No | `"true"` — field is read-only in settings; shown with "Change Provider →" button |
| `depends_on` | No | Key of a boolean field in the same module; this field is hidden when that field is `false`. Parser raises `SettingsParseError` if the referenced field does not exist or is not `type="boolean"`. |
| `readonly` | No | `"true"` — displayed but not editable |

### `<section>` attributes

| Attribute | Required | Description |
|---|---|---|
| `string` | Yes | Section heading |

### `selection` field

```xml
<field name="base.signup_method" type="selection" scope="system" label="Signup Method">
  <option value="invite_only" label="Invite Only"/>
  <option value="open" label="Open Registration"/>
  <option value="disabled" label="Disabled"/>
</field>
```

---

## 3. SettingsRegistry

**File:** `src/ede/core/services/settings/registry.py`

### Dataclasses

```python
@dataclass
class SettingsFieldDescriptor:
    name: str
    field_type: str       # boolean | integer | char | selection | many2one
    scope: str            # system | organization
    label: str
    default: str | None
    model: str | None
    domain: str | None
    secret: bool
    requires_migration: bool
    depends_on: str | None
    readonly: bool
    options: list[dict]   # [{value, label}] for selection fields

@dataclass
class SettingsSectionDescriptor:
    string: str
    fields: list[SettingsFieldDescriptor]

@dataclass
class SettingsModuleDescriptor:
    module: str
    label: str
    icon: str | None
    category: str
    order: int
    sections: list[SettingsSectionDescriptor]
```

### `SettingsRegistry`

```python
class SettingsRegistry:
    def __init__(self): ...
    def load_from_app(self, app_spec: AppSpec, base_path: str) -> None: ...
    def get_module(self, module_key: str) -> SettingsModuleDescriptor | None: ...
    def list_modules(self) -> list[SettingsModuleDescriptor]: ...  # sorted by order
    def all_field_keys(self) -> set[str]: ...
```

**Boot integration:** `bootstrap_environment()` in `src/ede/core/bootstrap.py` constructs a `SettingsRegistry`, calls `load_from_app()` for each loaded app, and stores it on the `Environment` (the global bootstrap result). `Env` gains two new attributes populated at boot:
- `env.settings_registry: SettingsRegistry` — the parsed module descriptors
- `env.get_setting(key, default=None)` — convenience method that internally calls `SettingsService(env.settings_registry).get_value(key, env)`

`SettingsService` is stateless (takes the registry in `__init__`); no singleton needed.

**File detection:** For each entry in `app_spec.data_files` matching `*/views/*.xml`, peek at the root XML element. If it is `<settings>`, parse it as a settings descriptor. If it is `<ede>` or `<view>`, delegate to `ViewRegistry`. This avoids any naming convention requirement.

---

## 4. SettingsService

**File:** `src/ede/core/services/settings/service.py`

```python
class SettingsService:
    def get_value(self, key: str, env: Env) -> Any:
        """Resolves: org-scoped row → system-scoped row → XML default → None."""

    def set_value(self, key: str, value: Any, scope: str, env: Env) -> None:
        """Validates type against registry, upserts ir.config row, writes audit log."""

    def get_module_schema_with_values(self, module_key: str, env: Env) -> dict:
        """Returns SettingsModuleDescriptor serialized + values resolved for current org."""

    def list_modules_nav(self, env: Env) -> list[dict]:
        """Returns [{module, label, icon, category, order}] for sidebar nav."""
```

**Encryption:** For `secret=True` fields, `set_value` uses `cryptography.fernet.Fernet` with a key derived from `settings.JWT_SECRET_KEY` (SHA-256 → base64-url-safe-32-bytes). `get_value` decrypts transparently. The API layer masks the decrypted value with `"••••••"` when serializing for the response.

**`env` shortcut:** `Env` gets a `get_setting(key, default=None)` method that delegates to `SettingsService.get_value()`. This is the canonical way for module code to read a setting at runtime:

```python
notifications_on = env.get_setting("email.feature.notifications", default=True)
```

**`requires_migration` fields:** `set_value` raises `SettingsMigrationRequired(key)` if the field is tagged `requires_migration="true"`. The API translates this to a `409` response.

---

## 5. API Endpoints

**Controller:** `src/ede/foundation/base/controllers/settings_controller.py`

All endpoints require authenticated principal. System-scope writes require admin role (checked via RBAC).

### `GET /api/settings`

Returns module nav list.

```json
[
  {"module": "base", "label": "General", "icon": "settings", "category": "Company", "order": 10},
  {"module": "email", "label": "Email", "icon": "mail", "category": "Services", "order": 20},
  {"module": "storage", "label": "Storage", "icon": "hard-drive", "category": "Services", "order": 30}
]
```

### `GET /api/settings/{module}`

Returns schema + resolved values for current org.

```json
{
  "module": "email",
  "label": "Email",
  "icon": "mail",
  "sections": [
    {
      "string": "Outgoing Mail",
      "fields": [
        {
          "name": "email.default_connector",
          "type": "many2one",
          "scope": "organization",
          "label": "Default Outgoing Email",
          "requires_migration": true,
          "secret": false,
          "depends_on": null,
          "readonly": false,
          "value": "uuid-of-current-connector",
          "display_value": "Gmail — company@example.com"
        }
      ]
    }
  ]
}
```

`display_value` is populated by the service for `many2one` fields (fetches the record's display name). Secret fields return `"••••••"` as `value`. Fields with `requires_migration=true` are included in the response but excluded from `PATCH`.

### `PATCH /api/settings/{module}`

Saves values for one module pane.

**Request:**
```json
{
  "values": {
    "email.feature.notifications": true,
    "email.max_attachment_mb": 50
  }
}
```

**Response:** `200 OK` with updated schema+values (same shape as GET).

**Error responses:**
- `409 Conflict` if any key has `requires_migration=true`
- `403 Forbidden` if principal lacks permission to write system-scope settings
- `422 Unprocessable Entity` for type validation failures

---

## 6. Frontend

### Routing

New TanStack Router routes added to `src/frontend/src/router.tsx`:
- `/wc/settings` → `SettingsPage` (redirects to first module)
- `/wc/settings/$module` → `SettingsPage` with module selected

New `ir.action` + `ir.menu` entries in `src/ede/foundation/base/data/base_settings_menu.xml` register "Settings" as a navigation app.

### Component Tree

```
SettingsPage  (/wc/settings/$module)
├── SettingsNav
│   ├── category header (e.g. "Company", "Services", "System")
│   └── SettingsNavItem (icon + label, active state)
│       └── unsaved-changes guard on click → confirm dialog
└── SettingsPane (lazy-loaded per module)
    ├── module title + breadcrumb
    ├── SettingsSection (per <section>)
    │   └── SettingsField (per <field>)
    │       ├── label + scope badge ("System" | "Company")
    │       ├── depends_on → hidden when referenced boolean is false
    │       └── widget by type:
    │           ├── boolean → toggle
    │           ├── char → text input (password input if secret)
    │           ├── integer → number input
    │           ├── selection → dropdown (options from schema)
    │           └── many2one → async searchable dropdown
    └── SaveBar (visible when form is dirty)
        ├── [Discard] — resets to loaded values
        └── [Save Changes] → PATCH /api/settings/{module}
```

**State management:** Each `SettingsPane` holds local React state for form values. On mount it fetches `GET /api/settings/{module}`. The `SaveBar` appears when local state diverges from loaded values. `Discard` resets to loaded values. `Save` calls `PATCH` and updates loaded values on success.

**Unsaved changes guard:** `SettingsNav` tracks whether the current pane is dirty. If user clicks a different nav item while dirty, a confirm dialog asks "Discard unsaved changes?" before navigating.

**Scope badge:** A small pill badge (`bg-surface-muted text-xs`) next to each field label shows `System` or `Company` to indicate which scope the setting affects.

**Migration-requiring fields:** Rendered as read-only display showing the current value + a `"Change Provider →"` button. The button navigates to `/wc/settings/migrate/{module}/{field}` which in v1 is a stub page: shows current value, allows selecting a new value, and on confirm directly updates the setting (no actual data movement). Real migration logic is deferred to a later sprint.

### Key Files

```
src/frontend/src/workspace/settings/
├── SettingsPage.tsx          # route component, fetches module list
├── SettingsNav.tsx           # sidebar with category grouping + dirty guard
├── SettingsPane.tsx          # module pane: fetch schema+values, form state, save bar
├── SettingsSection.tsx       # section heading + field list
├── SettingsField.tsx         # field renderer: label, badge, depends_on, widget
├── fields/
│   ├── SettingsToggle.tsx
│   ├── SettingsTextInput.tsx
│   ├── SettingsNumberInput.tsx
│   ├── SettingsSelect.tsx
│   └── SettingsMany2One.tsx
└── MigrationStub.tsx         # v1 placeholder migration wizard
```

---

## 7. Module Integration Pattern

To add settings for a new module (e.g., `crm`):

**Step 1 — Create the settings XML:**
```xml
<!-- src/domains/crm/views/crm_settings.xml -->
<settings module="crm" label="CRM" icon="users" category="Sales" order="50">
  <section string="Features">
    <field name="crm.feature.auto_assign"
           type="boolean"
           scope="organization"
           label="Auto-assign Leads"
           default="false"/>
  </section>
</settings>
```

**Step 2 — Add to manifest:**
```python
# src/domains/crm/__manifest__.py
{
    "data": [
        "views/crm_settings.xml",   # ← add this line
        "views/crm_lead_views.xml",
    ]
}
```

**Step 3 — Read in module code:**
```python
auto_assign = env.get_setting("crm.feature.auto_assign", default=False)
```

That is the complete integration. No Python backend registration code needed.

---

## 8. Encryption

**Library:** `cryptography` (already a transitive dep; add explicit to `pyproject.toml` if missing).

**Key derivation:**
```python
import hashlib, base64
raw = settings.JWT_SECRET_KEY.encode()
key = base64.urlsafe_b64encode(hashlib.sha256(raw).digest())
fernet = Fernet(key)
```

**Storage:** Encrypted value stored in `ir.config.encrypted_value`. The plaintext `value` column is `null` for secret fields. On read, `SettingsService` decrypts and returns the plaintext to internal callers; the API layer masks it before sending to the browser.

**Key rotation:** Out of scope for v1. Document that changing `JWT_SECRET_KEY` invalidates all stored encrypted settings (they must be re-entered). A `SETTINGS_ENCRYPTION_KEY` env var can be added later to decouple.

---

## 9. Audit Log

Every `SettingsService.set_value()` call writes an `ir.config.log` row:
- `key`, `old_value` (fetched before write), `new_value`, `scope`, `organization_id`
- `changed_by` = `env.principal.get("sub", "system")`
- Secret fields log `"••••••"` for both `old_value` and `new_value`

No UI for the audit log in v1. The rows are queryable via the existing CRUD API (`ede.search` on `ir.config.log`) for future admin tooling.

---

## 10. Scoping Behavior Reference

| Setting type | Scope in XML | Who can write | Fallback |
|---|---|---|---|
| Feature flags | `organization` | Org admin | System default → XML default |
| System policies | `system` | Super-admin only | XML default |
| Service connectors | `organization` | Org admin | None (must be set explicitly) |
| Service limits (e.g. attachment size) | `system` | Super-admin | XML default |
| Secret credentials | `organization` or `system` | Corresponding admin | None |

---

## 11. Verification

### Backend unit tests (`src/tests/settings/`)
- `test_ir_config.py` — CRUD, unique constraint, `(key, scope, org_id)` resolution
- `test_settings_service.py` — fallback chain (org → system → default), encryption round-trip, audit log written on change, `SettingsMigrationRequired` raised for migration fields
- `test_settings_registry.py` — XML parsing, `depends_on` wired correctly, unknown field types raise parse error

### Integration test
- Boot a test environment with a module that has a `<settings>` XML
- `GET /api/settings/{module}` returns expected schema
- `PATCH /api/settings/{module}` persists to `ir.config` and returns updated values
- Re-`GET` shows updated value

### Manual frontend test
1. `ede serve --config ede.conf` + `bun run dev`
2. Navigate to Settings in sidebar nav
3. Verify modules appear grouped by category
4. Toggle a boolean field → SaveBar appears
5. Click a different nav item → unsaved changes confirm dialog appears
6. Save → verify scope badge matches setting scope
7. Refresh page → verify saved value persists
8. Edit a `secret` field → verify `"••••••"` mask on re-load

---

## Phasing

**v1 (this sprint):** Full backend (ir.config, ir.config.log, SettingsRegistry, SettingsService, API), XML DSL, frontend SettingsPage with all field types, stub migration wizard, base module settings example XML.

**v2 (future):** Real migration wizard for storage/email provider switches (background job, progress polling), settings search, RBAC settings permissions UI, `SETTINGS_ENCRYPTION_KEY` decoupled from JWT key.
