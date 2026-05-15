# EDE Framework — Developer Guide

This guide walks through common development tasks: adding a new domain, app, model,
command, HTTP route, event handler, and migration.

## How the Pieces Fit Together

Before writing any code, understand the flow that connects every piece of the framework:

```
Settings (ACTIVE_MODULES)
  └─► ModuleLoader imports app packages
        └─► @api.model registers DomainModel in Registry
              └─► @api.on_command attaches handlers to models
              └─► @api.on_event registers event listeners
              └─► @api.on_hook registers lifecycle guards
              └─► @api.route_config + @api.route register HTTP routes

At runtime:
  HTTP Request → Controller → env.dispatch(Command)
    └─► CommandBus → pre-hooks → handler → post-hooks
          └─► handler emits events
                └─► EventWorker → on_event handlers
                      └─► handlers may emit web.client.* → SSE → browser
```

**Every new feature follows this exact pattern.** You declare a model, attach
handlers to it, declare the HTTP surface that triggers those handlers, and optionally
react to the resulting events. The framework wires everything together from the Registry
at boot.

**The single rule**: dependencies only flow downward. Your domain models never import
from controllers. Your controllers never contain business logic. Your event handlers
react to facts — they never call synchronous commands.

Following these rules means your system stays testable, replaceable, and growable as
requirements evolve.

---

## Setup

```bash
# Install dependencies
pip install -e ".[dev]"

# Run the server (dev mode with in-memory event worker)
ede serve --with-worker

# Run tests
pytest

# Check loaded apps
ede info
```

---

## 1. Adding a New Domain

A **domain** is a business subdomain (e.g. `logistics`, `crm`, `inventory`).

### Step 1: Create the domain package

```
src/domains/
└── logistics/
    ├── __init__.py          ← empty
    └── settings.py          ← ACTIVE_MODULES list
```

```python
# src/domains/logistics/settings.py
ACTIVE_MODULES = ["shipment"]
```

### Step 2: Register the domain

```python
# src/domains/settings.py
ACTIVE_DOMAINS = ["logistics"]
```

That's it — the framework picks up `logistics` on next boot.

---

## 2. Adding a New App

An **app** is a bounded context within a domain (e.g. `logistics.shipment`).

### Step 1: Create the app package

```
src/domains/logistics/
└── shipment/
    ├── __manifest__.py
    ├── __init__.py           ← imports models subpackage
    ├── models/
    │   ├── __init__.py       ← imports all model modules
    │   └── shipment.py
    ├── api/
    │   └── controllers.py
    └── migrations/
        └── versions/
```

### Step 2: Write the manifest

```python
# src/domains/logistics/shipment/__manifest__.py
{
    "name": "Shipment",
    "summary": "Manage logistics shipments",
    "description": "Track and manage shipment lifecycle from pending to delivery.",
    "author": "THE_BLACK_BOX",
    "category": "logistics",
    "version": "1.0.0",
    "data": [
        "views/logistics_shipment.xml",
    ],
}
```

### Step 3: Activate the app

```python
# src/domains/logistics/settings.py
ACTIVE_MODULES = ["shipment"]    # add "shipment" here
```

### Step 4: Set up `__init__.py` files

```python
# src/domains/logistics/shipment/__init__.py
from . import models   # trigger model registration
```

```python
# src/domains/logistics/shipment/models/__init__.py
from . import shipment   # import each model module
```

---

## 3. Adding a New Model

```python
# src/domains/logistics/shipment/models/shipment.py

from ede.core import api
from ede.core.kernel.model import DomainModel
from ede.core.kernel import fields
from ede.core.types import Command


@api.model(
    "logistics.shipment",
    description="A logistics shipment",
    default_order="created_at_utc desc",
)
class Shipment(DomainModel):
    tracking_number = fields.Char(max_length=50, required=True, unique=True)
    status          = fields.Enum(
        ["pending", "confirmed", "shipped", "delivered", "cancelled"],
        default="pending",
        required=True,
    )
    weight_kg       = fields.Decimal(precision=10, scale=3)
    notes           = fields.Char(max_length=1000)

    # Reference to foundation model
    origin_country  = fields.Reference("res.country")
    assigned_to     = fields.Reference("res.user")

    @api.on_command("shipment.create")
    def create(self, cmd: Command) -> dict:
        record = self.env.models["logistics.shipment"].create(cmd.payload)
        return record.read()[0]

    @api.on_command("shipment.confirm")
    def confirm(self, cmd: Command) -> dict:
        record = cmd.record
        record.write({"status": "confirmed"})
        self.emit("shipment.confirmed", {
            "shipment_id": record.id,
            "tenant_id": self.env.tenant_id,
        })
        return {"ok": True, "id": record.id}
```

**Checklist**:
- [ ] `@api.model("domain.name")` with correct key
- [ ] Inherit `DomainModel`
- [ ] Fields declared as class attributes using `fields.*`
- [ ] Model imported in `models/__init__.py`

---

## 4. Adding a Command Handler

Inside a `DomainModel`:

```python
@api.on_command("shipment.cancel")
def cancel(self, cmd: Command) -> dict:
    record = cmd.record    # target record (set when dispatched with model_id)
    if record.status == "delivered":
        raise ValueError("Cannot cancel a delivered shipment")
    record.write({"status": "cancelled"})
    self.emit("shipment.cancelled", {"shipment_id": record.id})
    return {"ok": True}
```

**Naming**: `"{domain}.{app}.{verb}"` or `"{app_key}.{verb}"`. Both work, but be consistent.

---

## 5. Adding an Event Handler

Event handlers are **module-level functions** (not class methods):

```python
# src/domains/logistics/shipment/models/shipment.py  (or a separate events.py)

from ede.core import api
from ede.core.bus.types import Event
from ede.core.env import Env

@api.on_event("shipment.confirmed")
def on_shipment_confirmed(event: Event, env: Env) -> None:
    payload = event.payload
    shipment_id = payload.get("shipment_id")
    tenant_id = event.tenant_id

    # Example: notify the assigned user
    shipment = env.models["logistics.shipment"].browse(shipment_id)
    if shipment.assigned_to.exists():
        user = shipment.assigned_to
        # send notification, update stats, etc.
```

Make sure the module containing the handler is imported (i.e. listed in `models/__init__.py`).

---

## 6. Adding an HTTP Controller

```python
# src/domains/logistics/shipment/api/controllers.py

from ede.core import api
from ede.core.services.http.controller import RouteController
from ede.core.types import Command


@api.route_config(prefix="/api/logistics/shipments", tags=["Shipments"])
class ShipmentController(RouteController):

    @api.route("/", methods=["GET"], auth="user")
    def list(self) -> list:
        return self.env.dispatch(Command(
            name="ede.search",
            payload={"domain": [], "order": "created_at_utc desc", "limit": 80},
            model_key="logistics.shipment",
        ))

    @api.route("/{id}", methods=["GET"], auth="user")
    def get_one(self, id: str) -> dict:
        return self.env.dispatch(Command(
            name="ede.read_one",
            payload={},
            model_key="logistics.shipment",
            model_id=id,
        ))

    @api.route("/", methods=["POST"], auth="user")
    def create(self, body: dict) -> dict:
        return self.env.dispatch(Command(
            name="shipment.create",
            payload=body,
        ))

    @api.route("/{id}/confirm", methods=["POST"], auth="user")
    def confirm(self, id: str) -> dict:
        return self.env.dispatch(Command(
            name="shipment.confirm",
            payload={},
            model_key="logistics.shipment",
            model_id=id,
        ))

    @api.route("/{id}", methods=["PUT"], auth="user")
    def update(self, id: str, body: dict) -> dict:
        return self.env.dispatch(Command(
            name="ede.update",
            payload=body,
            model_key="logistics.shipment",
            model_id=id,
        ))

    @api.route("/{id}", methods=["DELETE"], auth="user")
    def delete(self, id: str) -> dict:
        self.env.dispatch(Command(
            name="ede.delete",
            payload={},
            model_key="logistics.shipment",
            model_id=id,
        ))
        return {"ok": True}
```

The controller is discovered automatically when the app package is imported.

---

## 7. Adding a View

Create an XML file under the app's `views/` directory:

```xml
<!-- src/domains/logistics/shipment/views/logistics_shipment.xml -->

<view id="logistics.shipment.list" version="1" title="Shipments">
    <list model="logistics.shipment" default_order="created_at_utc desc">
        <field name="tracking_number" label="Tracking No." />
        <field name="status" widget="badge" />
        <field name="weight_kg" label="Weight (kg)" />
        <field name="origin_country" />
        <field name="assigned_to" />
    </list>
</view>

<view id="logistics.shipment.form" version="1" title="Shipment">
    <form model="logistics.shipment">
        <header>
            <statusbar field="status"
                       values="pending,confirmed,shipped,delivered,cancelled" />
        </header>
        <sheet>
            <group string="Shipment Details">
                <field name="tracking_number" />
                <field name="weight_kg" />
                <field name="origin_country" />
                <field name="assigned_to" />
            </group>
            <group string="Notes">
                <field name="notes" />
            </group>
        </sheet>
    </form>
</view>

<view id="logistics.shipment.search" version="1" title="Search Shipments">
    <search model="logistics.shipment">
        <field name="tracking_number" string="Tracking No." />
        <filter string="Pending" domain="[('status','=','pending')]" />
        <filter string="Confirmed" domain="[('status','=','confirmed')]" />
        <group_by string="Status" field="status" />
        <group_by string="Origin Country" field="origin_country" />
    </search>
</view>
```

Declare in manifest:
```python
"data": ["views/logistics_shipment.xml"]
```

---

## 8. Adding a Migration

After adding or changing a model, generate a migration:

```bash
ede migrate generate -m "add logistics shipment"
```

Review the generated file in `src/domains/logistics/shipment/migrations/versions/`.

Apply to a tenant:
```bash
ede migrate upgrade -t acme
```

---

## 9. Using Transactions

Wrap multiple ORM operations in a transaction for atomicity:

```python
@api.on_command("order.create_with_lines")
def create_with_lines(self, cmd: Command) -> dict:
    payload = cmd.payload
    lines = payload.pop("order_lines", [])

    with self.env.transaction():
        order = self.env.models["logistics.order"].create(payload)
        for line in lines:
            line["order_id"] = order.id
            self.env.models["logistics.order_line"].create(line)

    return order.read()[0]
```

---

## 10. Relational Fields in Practice

```python
# Create with M2M links
record = env.models["content.article"].create({
    "title": "Hello World",
    "tags": [
        RelationalCommand.link("tag-uuid-1"),
        RelationalCommand.link("tag-uuid-2"),
    ],
})

# Update with O2M lines
order = env.models["logistics.order"].browse("order-uuid")
order.write({
    "order_lines": [
        RelationalCommand.create({"product": "Widget", "qty": 3}),
        RelationalCommand.update("existing-line-uuid", {"qty": 5}),
        RelationalCommand.delete("old-line-uuid"),
    ]
})

# Read related records
country = shipment.origin_country     # → RecordSet(res.country)
if country.exists():
    print(country.name)
```

---

## 11. Seeding Data Files

Apps can ship seed records (lookup tables, default roles, permissions, menus, etc.)
that are loaded automatically during `ede migrate upgrade`. Declare files in
`__manifest__.py` under `"data"`:

```python
# src/domains/crm/__manifest__.py
{
    "name": "CRM",
    ...
    "data": [
        "data/crm.lead.status.csv",    # CSV: model key from filename
        "data/default_views.xml",      # XML: arbitrary records
        "data/ir.rbac.permission.csv", # permission seed
    ],
}
```

Files are processed in declaration order. XML files can cross-reference CSV records
loaded earlier in the same run.

---

### CSV Format

File name = model key + `.csv` (e.g. `crm.lead.status.csv` → model `crm.lead.status`).

```
id,name,code,is_active
crm.status_new,New,NEW,true
crm.status_won,Won,WON,true
crm.status_lost,Lost,LOST,false
```

**Rules:**
- First column must be `id` — the external ID (`module.name`).
- All values are strings; the loader coerces `true`/`false` to `bool`, integers to `int`,
  and empty string to `None` (NULL).
- CSVs are always `noupdate=False` — each upgrade re-applies field values.

**FK/reference columns — `/id` suffix:**

When a column holds an external ID that should be resolved to a record UUID, append
`/id` to the column name. The loader will look it up in `ir.data.reference`:

```
id,name,code,resource,action,role_id/id,domain
myapp.p_lead_read,Read Leads,crm.lead.read,crm.lead,read,rbac.role_portal_user,
myapp.p_lead_create,Create Leads,crm.lead.create,crm.lead,create,rbac.role_internal_user,
```

Without `/id`, the raw string is passed to the DB. With `/id`, the loader resolves
`rbac.role_portal_user` → UUID of that role record.

Leave the value empty to store NULL (e.g. a permission with no role assigned yet).

---

### XML Format

```xml
<ede><data noupdate="1">

  <!-- Basic record -->
  <record id="crm.stage_new" model="crm.stage">
    <field name="name">New</field>
    <field name="sequence">1</field>
  </record>

  <!-- Cross-reference another record with ref= -->
  <record id="crm.default_pipeline" model="crm.pipeline">
    <field name="name">Default</field>
    <field name="default_stage_id" ref="crm.stage_new"/>
  </record>

  <!-- Custom command instead of ede.create -->
  <record id="rbac.admin_binding" model="ir.rbac.role.binding"
          command="ir.rbac.role.binding.assign">
    <field name="user_id" ref="base.admin_user"/>
    <field name="role_id" ref="rbac.role_system_admin"/>
    <field name="scope_type">GLOBAL</field>
  </record>

</data></ede>
```

**Attributes:**
- `id` — external ID (`module.name`), used to track the record in `ir.data.reference`
- `model` — model key
- `command` — (optional) dispatch this command instead of `ede.create` / `ede.update`
- `noupdate="1"` — skip this record on re-runs if it already exists (user may have edited it)

**`<field>` attributes:**
- `name` — field name
- `ref="module.name"` — resolves to the UUID of the referenced record

---

### Idempotency

The loader tracks every created record via `ir.data.reference`. On subsequent runs:
- Record already in `ir.data.reference` + `noupdate=True` → **skipped**
- Record already in `ir.data.reference` + `noupdate=False` → **updated** with fresh values
- Record not in `ir.data.reference` but already in DB (unique constraint) →
  loader finds it by natural key (`code`, `email`, or `path`) and **registers** it,
  so dependent refs resolve correctly

---

## Common Patterns Reference

### Pattern: Read + Write in one command
```python
@api.on_command("shipment.update_status")
def update_status(self, cmd: Command) -> dict:
    record = cmd.record
    record.write({"status": cmd.payload["status"]})
    return record.read()[0]
```

### Pattern: Search then act
```python
@api.on_command("shipment.cancel_all_pending")
def cancel_all_pending(self, cmd: Command) -> dict:
    pending = self.env.models["logistics.shipment"].search(
        [("status", "=", "pending")]
    )
    with self.env.transaction():
        for shipment in pending:
            shipment.write({"status": "cancelled"})
    return {"cancelled": len(pending.ids)}
```

### Pattern: Create then link
```python
with env.transaction():
    user = env.models["res.user"].create({
        "email": "new@example.com",
        "name": "New User",
    })
    # dispatch password set
    env.dispatch(Command(
        name="res.user.set_password",
        payload={"password": "initial-pass"},
        model_key="res.user",
        model_id=user.id,
    ))
```

---

## Error Handling

EDE errors:

| Exception | When Raised |
|---|---|
| `CommandHandlerNotFound` | Command name not registered |
| `ModelNotRegistered` | `env.models["unknown"]` |
| `DuplicateHandler` | Two handlers for same command name |
| `LoaderError` | App load failure |
| `ManifestNotFound` | Missing `__manifest__.py` |
| `InvalidManifest` | Manifest missing required keys |
| `TenantDatabaseNotFound` | Tenant DB doesn't exist |
| `TokenError` | JWT decode failure |
| `ExpiredTokenError` | JWT expired |
| `InvalidTokenError` | Bad JWT signature/claims |

Handle errors in controllers:

```python
from ede.core.errors import CommandHandlerNotFound

@api.route("/{id}/action", methods=["POST"], auth="user")
def run_action(self, id: str, body: dict) -> dict:
    try:
        return self.env.dispatch(Command(
            name=body["command"],
            payload=body.get("payload", {}),
            model_key="logistics.shipment",
            model_id=id,
        ))
    except CommandHandlerNotFound:
        return {"error": f"Unknown command: {body['command']}"}
```

---

## Running Tests

```bash
# All tests (unit + integration; e2e excluded by default)
./run_tests.sh

# Specific test file
pytest tests/test_command_event_flow.py

# With coverage
./run_tests.sh --cov

# End-to-end browser tests (Playwright)
./run_tests.sh --e2e
```

Test patterns:
- Use `NullPersistenceProvider` for unit tests that don't need DB
- Use `InMemoryEventQueue` (always) for event-driven tests
- Use `SqlAlchemy + SQLite in-memory` for integration tests with persistence

---

## 12. End-to-End Testing (`foundation.qa-automation`)

E2E tests drive a real browser (Chromium via Playwright) against a real FastAPI
server backed by a fresh PostgreSQL tenant. They live under:

- [src/tests/e2e/](../src/tests/e2e/) — platform-wide tests (auth, base, presentation, workflow, approval, communication, notifications)
- `src/domains/<domain>/<module>/tests/e2e/` — module-specific tests (e.g. `src/domains/logistics/sales_crm/tests/e2e/`)

### 12.1 Quick start

```bash
# Install Playwright extras + browser binaries (one-time)
pip install -e ".[dev,e2e]"
playwright install chromium --with-deps

# Run the full e2e suite
./run_tests.sh --e2e

# Run a single test, live in a visible browser, with 500ms slow-motion
./run_tests.sh --e2e --headed --slowmo=500

# Run a single file directly via pytest (faster iteration)
pytest src/tests/e2e/foundation/auth/test_login.py \
    -p ede.foundation.qa_automation.fixtures \
    --browser=chromium --headed --slowmo=500 -vv -s
```

The pytest plugin `ede.foundation.qa_automation.fixtures` ships all e2e fixtures
(`live_server`, `seed_admin`, `authenticated_page`, `page_as_user`,
`seed_deterministic`, etc.). `./run_tests.sh --e2e` registers it automatically.

### 12.2 Runtime configuration — env vars

All e2e knobs are env vars (no config file edits needed):

| Variable | Default | Purpose |
|---|---|---|
| `QA_E2E_WINDOW` | `1280x720` | Chromium window + viewport + video-recording size. Format `WIDTHxHEIGHT`. Applies to both headed and headless modes. |
| `QA_E2E_ENABLED` | `false` | Hard kill-switch; the `--e2e` branch in `run_tests.sh` flips it on for you. |
| `QA_E2E_BROWSERS` | `chromium` | Comma-separated list of browsers to launch. |
| `QA_E2E_HEADLESS` | `true` | When `false`, headed mode is forced even without `--headed`. |
| `QA_E2E_VIDEO_DIR` | `qa-report/artifacts` | Per-test `video.webm` location. |
| `QA_E2E_TRACE_DIR` | `qa-report/artifacts` | Per-test `trace.zip` location. |

Common overrides:

```bash
# Bigger window for visual review
QA_E2E_WINDOW=1920x1080 ./run_tests.sh --e2e --headed

# Retina-ish for snapshot work
QA_E2E_WINDOW=2560x1440 ./run_tests.sh --e2e --headed --slowmo=500

# Custom browser list
QA_E2E_BROWSERS=chromium,firefox ./run_tests.sh --e2e
```

The default 1280×720 is deliberate — it stays inside every snapshot baseline
captured by `seed_deterministic` visual-regression tests, and matches the
recorded `video.webm` dimensions byte-for-byte.

### 12.3 Live runtime — what the 5 stages mean

When `./run_tests.sh --e2e` starts you'll see a banner and five staged log lines.
This is normal; the suite is not stuck. Each stage runs **once per session**:

```
[qa-automation] Stage 1/5: Building frontend bundle (bun run build)…
[qa-automation] Stage 2/5: Migrating e2e tenant `e2e_qa_<hex>` (PostgreSQL)…
[qa-automation] Stage 3/5: Booting runtime (registry + apps + persistence)…
[qa-automation] Stage 4/5: Starting in-memory EventWorker thread…
[qa-automation] Stage 5/5: Launching uvicorn on http://127.0.0.1:<port>
```

Stages 1–5 typically take 20–40 seconds total. After Stage 5 each test reuses
the same server and browser context; only the test body runs per-test.

### 12.4 Per-test artifacts

After every run, look under [qa-report/](../qa-report/) (gitignored):

```
qa-report/
├── junit.xml                                  ← CI-friendly test summary
└── artifacts/<sanitised-test-id>/
    ├── video.webm                             ← full session recording
    └── trace.zip                              ← Playwright trace (DOM + network + console + screenshots)
```

Replay a trace interactively:

```bash
python -m playwright show-trace qa-report/artifacts/test_login_good_creds/trace.zip
```

### 12.5 Authoring a new test

Two ways:

**(a) By hand.** Create the file under the right directory and import the shared
markers — fixtures are auto-wired by the plugin:

```python
# src/tests/e2e/foundation/<area>/test_<scenario>.py
import pytest

pytestmark = pytest.mark.e2e


@pytest.mark.qa_module("foundation.base")
@pytest.mark.brs("FND-BASE-NN")
class TestMyScenario:
    def test_happy_path(self, authenticated_page, live_server):
        page = authenticated_page
        page.goto(f"{live_server.base_url}/wc/")
        page.get_by_role("button", name="…").click()
        # assertions…
```

**(b) Via codegen** (records clicks against the live server, emits a scaffold):

```bash
ede e2e record foundation.base/quick_smoke --brs FND-BASE-99
# → src/tests/e2e/foundation/base/test_quick_smoke.py
# → docs/demo-usecase/modules/foundation-base/usecases/quick_smoke.md (stub)
```

The CLI launches Chromium with codegen, lets you click through the flow, then
writes a scaffolded pytest file you edit to taste.

### 12.6 Common fixtures

| Fixture | Scope | What it gives you |
|---|---|---|
| `live_server` | session | `LiveServer(base_url, env, boot_output, server)` — the running FastAPI app + booted `Env` |
| `seed_admin` | session | `(user_record, password)` for the bootstrapped admin |
| `authenticated_page` | function | Playwright `Page` already logged in as the seeded admin |
| `unauthenticated_page` | function | Fresh `Page` with no session — for login/redirect tests |
| `page_as_user(email, password)` | function | Factory: log in as an arbitrary user |
| `seed_deterministic` | function | Insert rows with fixed UUIDs via `session.execute(insert(...))` — required for visual regression baselines |
| `with_demo_data(app_key)` | session | Loads the module's `demo/*.xml` files into the e2e tenant |

### 12.7 Tips

- Headed runs use `--window-size=W,H` on the Chromium command line (more
  reliable than `--start-maximized` on Linux WMs).
- `-s` / `--capture=no` is already set by `run_tests.sh --e2e` so stage banners
  print live — don't override it.
- The e2e tenant DB (`e2e_qa_<hex>`) is dropped on session teardown; if a run
  aborts hard, drop it manually: `psql -c "DROP DATABASE e2e_qa_<hex>"`.
- The default test suite (`./run_tests.sh` without `--e2e`) explicitly excludes
  `**/tests/e2e` via `--ignore-glob`, so unit tests stay fast.
