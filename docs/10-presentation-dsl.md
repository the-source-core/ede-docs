# EDE Framework — Presentation DSL (Views)

## Overview

EDE has a declarative View DSL: XML files that describe UI structure (lists, forms, kanbans,
search panels). The backend parses them into `RenderPlan` dicts served to the frontend web
client.

This follows the same philosophy as the domain model: the backend is the source of truth for
structure; the frontend is a rendering engine.

---

## View Files

Views are XML files declared in `__manifest__.py` under the `"data"` key:

```python
# __manifest__.py
{
    "name": "Logistics",
    "data": [
        "views/shipment.xml",
        "views/order.xml",
    ],
}
```

Paths are **relative to the app root directory** (where `__manifest__.py` lives).

---

## View XML Structure

```xml
<view id="logistics.shipment.list" version="1" title="Shipments">
    <list model="logistics.shipment" default_order="created_at_utc desc">
        <field name="tracking_number" label="Tracking No." />
        <field name="status" />
        <field name="weight_kg" label="Weight (kg)" />
        <field name="origin_country" />
        <field name="created_at_utc" label="Created" />
    </list>
</view>
```

```xml
<view id="logistics.shipment.form" version="1" title="Shipment">
    <form model="logistics.shipment">
        <header>
            <statusbar field="status" values="pending,confirmed,shipped,delivered,cancelled" />
        </header>
        <sheet>
            <group string="General Info">
                <field name="tracking_number" />
                <field name="weight_kg" />
                <field name="origin_country" />
            </group>
            <notebook>
                <page string="Details">
                    <group>
                        <field name="dispatched_at" />
                        <field name="notes" />
                    </group>
                </page>
                <page string="Order Lines">
                    <field name="order_lines">
                        <list>
                            <field name="product" />
                            <field name="qty" />
                        </list>
                    </field>
                </page>
            </notebook>
        </sheet>
    </form>
</view>
```

---

## View ID Naming

View IDs follow a dotted namespace convention:
- Pattern: `{domain}.{model_name}.{view_type}`
- Example: `logistics.shipment.list`, `logistics.shipment.form`

**Filename Rule**: `view_id` with `.` replaced by `_` must match the filename (minus
directory path):
- `logistics.shipment` → `logistics_shipment.xml`
- `hello.world` → `hello_world.xml`

---

## Supported View Types

### List View (`<list>`)

Renders a tabular view of records.

```xml
<view id="model.name.list" version="1" title="...">
    <list model="model.key" default_order="field_name [asc|desc]" limit="80">
        <field name="field_name" label="Label" />
        <field name="field_name" widget="badge" />
        ...
    </list>
</view>
```

Attributes on `<field>`:
- `name` — required: field name on the model
- `label` — optional: override display label
- `widget` — optional: hint for frontend widget (e.g. `badge`, `date`, `money`)
- `optional` — optional: `"show"` | `"hide"` (column visibility toggle)
- `nolabel="1"` — optional: suppress the label cell, let the widget span full row. `image` / `binary` fields auto-skip the label by default, so this is mainly for explicit cases on other field types.
- `option-<key>="<value>"` — optional: passes widget-specific options. The parser collects every `option-foo="bar"` into `def.options.foo` on the frontend. Hyphens in the key become underscores (`option-display-mode` → `options.display_mode`).

### Form View (`<form>`)

Renders an editable record detail view.

Supported child elements:

| Element | Purpose |
|---|---|
| `<header>` | Top action bar (statusbar, buttons) |
| `<sheet>` | Main content area |
| `<group>` | Two-column layout group. `string` attr = group title |
| `<notebook>` | Tabbed container |
| `<page>` | Tab within notebook. `string` attr = tab label |
| `<field>` | Field widget (read or edit) |
| `<statusbar>` | Status pipeline widget. `field` + `values` attrs |
| `<button>` | Action trigger. `string` = label, `name` = command name, `type="object"`. Supports `special` handlers (see below). |
| `<chatter>` | Activity/log stream (reserved) |

---

### Button Special Handlers

Buttons can trigger pre-dispatch side effects before the command is sent to the server. Declare
with the `special` attribute. All `special-*` attributes are passed as options to the handler.

**Currently supported special handlers:**

#### `special="file_upload"`

Opens the OS native file picker. After the user selects a file, the frontend reads it as base64
and **merges** the following keys into the command payload before dispatching:

| Key | Value |
|-----|-------|
| `file_content` | Base64-encoded file content (no data URL prefix) |
| `file_name` | Original filename, e.g. `credentials.json` |
| `file_mime` | MIME type, e.g. `application/json` (falls back to `application/octet-stream`) |

If the user cancels the file picker the command is **not** dispatched.

Use `special-accept` to restrict the file types shown in the picker (standard HTML `accept` syntax):

```xml
<form model="ir.connector">
    <header>
        <button string="Import JSON Config"
                name="ir.connector.import_config"
                type="object"
                special="file_upload"
                special-accept=".json,application/json" />
    </header>
    ...
</form>
```

The backend command handler receives the merged payload:

```python
@api.on_command("ir.connector.import_config")
def handle_import_config(self, cmd: Command) -> dict:
    payload = cmd.payload or {}
    # payload["file_content"] is base64 string
    # payload["file_name"]    is original filename
    # payload["file_mime"]    is MIME type
    json_data = base64.b64decode(payload["file_content"])
    ...
```

Source: `src/frontend/src/workspace/views/specials/FileUploadSpecial.ts`

### Image / Binary Field Widgets

Models declaring `fields.Image(...)` or `fields.Binary(...)` (kernel field types
`"image"` / `"binary"`) render through dedicated widgets on the form view. Each
displays a square tile with a pencil (replace) and trash (clear) icon-button
overlay on hover; binary tiles additionally render a download icon and a
mimetype-derived file icon (PDF, archive, spreadsheet, code, etc.).

```xml
<field name="image_avatar" option-size="xxl" nolabel="1"/>
<field name="contract_pdf"  option-size="lg"/>
```

**Widget options:**

| Option | Values | Default | Effect |
|---|---|---|---|
| `option-size` | `sm` (64px) · `md` (96px) · `lg` (128px) · `xl` (160px) · `xxl` (192px) | `md` | Tile + placeholder-icon size. Same scale for image and binary. |

**Behavior notes:**

- `image` and `binary` fields auto-suppress the label cell (same effect as
  `nolabel="1"`). Adding `nolabel="1"` to them is redundant but harmless.
- Uploads POST the raw file body to `/api/document/attachments` with a Bearer
  JWT and the polymorphic back-reference (`model_key`, `record_uuid`,
  `field_name`, `usage_kind`) in query params. The endpoint returns the new
  `ir.attachment.record_uuid` which the widget writes into the form draft via
  `onChange`. The record must already exist (be saved at least once) before a
  field upload can happen.
- Previews are fetched via the authenticated `httpClient.getBlob` and rendered
  through in-component `URL.createObjectURL(blob)` — `<img src="...">` cannot
  carry a bearer header, so the blob-URL indirection is mandatory.

### Kanban View (`<kanban>`)

```xml
<view id="logistics.shipment.kanban" version="1" title="Shipments Kanban">
    <kanban model="logistics.shipment" groupby_field="status">
        <column value="pending"   label="Pending" />
        <column value="confirmed" label="Confirmed" />
        <column value="shipped"   label="Shipped" />
        <column value="delivered" label="Delivered" />
        <card_field name="tracking_number" />
        <card_field name="weight_kg" />
        <card_field name="origin_country" />
    </kanban>
</view>
```

### Search View (`<search>`)

```xml
<view id="logistics.shipment.search" version="1" title="Search Shipments">
    <search model="logistics.shipment">
        <field name="tracking_number" string="Tracking No." />
        <field name="status" />
        <filter string="Pending" domain="[('status','=','pending')]" />
        <filter string="This Month"
                domain="[('created_at_utc','>=','%(month_start)s')]" />
        <group_by string="Status" field="status" />
        <group_by string="Origin Country" field="origin_country" />
    </search>
</view>
```

---

## DslParser → RenderPlan

`src/ede/core/services/presentation/dsl/parser.py`

The `DslParser` converts a parsed XML tree into a `RenderPlan` dict. This dict is
serialized directly to JSON and consumed by the frontend.

```python
from ede.core.services.presentation.dsl.parser import DslParser

parser = DslParser()
render_plan = parser.parse(xml_string)
# → {
#     "view_id": "logistics.shipment.list",
#     "version": 1,
#     "title": "Shipments",
#     "type": "list",
#     "model": "logistics.shipment",
#     "fields": [...],
#     ...
# }
```

---

## ViewRegistry

`src/ede/core/services/presentation/view_registry.py`

The `ViewRegistry` is populated at boot from `AppSpec.data_files` (paths from manifests).
It provides fast lookup by `view_id`.

```python
# Access in a model or controller:
view_plan = env.registry.view_registry.get("logistics.shipment.list")
```

---

## PresentationKernel Model

`src/ede/foundation/presentation/models/presentation.py`

The `PresentationKernel` model exposes the view system over the command bus:

| Command | Payload | Returns |
|---|---|---|
| `presentation.list_views` | `{}` | List of all registered view IDs + metadata |
| `presentation.get_view_plan` | `{"view_id": "..."}` | Full `RenderPlan` dict |

These commands are called by the frontend bootstrap endpoint.

---

## Web Client Bootstrap

`GET /api/presentation/bootstrap`

Returns the full bootstrap payload for the frontend web client:
- Active apps + menus
- Actions (which view to load for each menu item)
- View definitions
- Current user info

This single endpoint is the entry point for the React frontend on initial load.

---

## DSL File Loader

`src/ede/core/services/presentation/dsl/loader.py`

`DslFileLoader.load(file_path: Path) -> str` — reads the XML file from disk and returns
the raw string. Called by `ViewRegistry` at boot to populate the view cache.

---

## Why a Backend View DSL?

### The backend is the source of truth for structure

In a typical React SPA, the frontend owns both the data fetching logic and the UI structure.
Adding a new field to a form requires a frontend code change, a build, and a deployment.
In EDE, the frontend is a **rendering engine** — it receives a `RenderPlan` dict from the
backend and renders it. Adding a field to a form is a one-line XML change on the server,
loaded at boot with no frontend rebuild required.

### Structure is data, not code

A `RenderPlan` dict is serializable, versionable, and inspectable. It can be cached,
diffed, and compared across versions. A hardcoded React form is none of these things.
When a form has a `version` field, you can detect breaking changes before they reach
production.

### Decoupling view lifecycle from data lifecycle

Views describe *how to present* a model, not *what the model contains*. A model can
have multiple views (list, form, kanban, search) that evolve independently. A form
layout change doesn't require a data migration. A new kanban column doesn't require
touching the model's command handlers.

### Future-proofing: the same views for different frontends

Because views are data (XML → RenderPlan JSON), the same view definition can drive a
React web client, a mobile app, or a third-party integration that wants to know the
canonical field order for a model. The rendering layer is swappable; the view definitions
are not duplicated.

---

## Best Practices

1. **One XML file per model** — name it `{model_key_with_underscores}.xml`
   (e.g. `logistics_shipment.xml`)

2. **Include all standard view types** — list, form, kanban (if applicable), search. The
   frontend relies on these being available.

3. **Use `<group string="...">` for sections** — improves readability and UI layout.

4. **Keep view IDs globally unique** — they are indexed globally across all apps.

5. **Declare data files in manifest** — views not listed in `"data"` will not be discovered.

6. **Version field** — start at `1`. Increment when making breaking changes to the
   view structure that require frontend updates.
