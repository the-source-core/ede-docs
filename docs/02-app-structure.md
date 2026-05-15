# EDE Framework — App Structure & Loading

## Source Tree

```
src/
├── ede/
│   ├── core/                          ← Framework kernel (never modify for app code)
│   │   ├── api.py                     ← Public decorator surface
│   │   ├── registry.py                ← Single source of truth (models/commands/events/providers)
│   │   ├── env.py                     ← Runtime context (Env)
│   │   ├── loader.py                  ← ModuleLoader (settings-driven activation)
│   │   ├── bootstrap.py               ← bootstrap_environment()
│   │   ├── boot.py                    ← boot_runtime_and_environment() (canonical entry)
│   │   ├── types.py                   ← Command, Event, AppMeta, RouteMeta, type aliases
│   │   ├── errors.py                  ← EdeError exception hierarchy
│   │   ├── utils.py                   ← iter_modules_with_prefix(), import_string()
│   │   ├── runtime.py                 ← Global process-level state
│   │   │
│   │   ├── kernel/
│   │   │   ├── model.py               ← DomainModel, AbstractDomainModel base classes
│   │   │   ├── fields.py              ← Field descriptors (UUID, Char, Integer, etc.)
│   │   │   ├── decorators.py          ← @api.model decorator implementation
│   │   │   └── schema.py              ← ORM-agnostic schema metadata (ColumnSpec, etc.)
│   │   │
│   │   ├── bus/
│   │   │   ├── types.py               ← Event envelope, EventDelivery, RetryPolicy
│   │   │   ├── queue.py               ← EventQueue Protocol
│   │   │   ├── inmemory_queue.py      ← In-memory EventQueue (dev/test)
│   │   │   ├── command_bus.py         ← CommandBus (sync dispatch)
│   │   │   ├── dispatcher.py          ← EventDispatcher (calls event handlers)
│   │   │   └── worker.py              ← EventWorker (poll → dispatch → ack/retry)
│   │   │
│   │   ├── adapters/
│   │   │   ├── __init__.py            ← adapter factory accessors
│   │   │   ├── http/fastapi/
│   │   │   │   ├── fastapi.py         ← FastApiHttpAdapter (mounts routes)
│   │   │   │   ├── auth_middleware.py ← JWT AuthMiddleware + route auth registry
│   │   │   │   └── middlewares.py     ← add_all_middlewares() factory
│   │   │   └── persistence/sqlalchemy/
│   │   │       ├── sqlalchemy_provider.py  ← SqlAlchemyPersistenceProvider
│   │   │       ├── metadata_builder.py     ← SQLAlchemy MetaData from EDE schemas
│   │   │       ├── database_url.py         ← per-tenant URL builder
│   │   │       └── admin.py                ← ensure DB exists (PostgreSQL)
│   │   │
│   │   ├── services/
│   │   │   ├── http/
│   │   │   │   ├── controller.py      ← RouteController base class
│   │   │   │   ├── decorators.py      ← @route_config, @route decorators
│   │   │   │   ├── types.py           ← RouteConfig, RouteMeta data classes
│   │   │   │   ├── registry.py        ← HttpRouteRegistry (collision detection)
│   │   │   │   └── scanner.py         ← HttpRouteScanner (scans sys.modules)
│   │   │   │
│   │   │   ├── auth/
│   │   │   │   ├── errors.py          ← TokenError, ExpiredTokenError, InvalidTokenError
│   │   │   │   └── jwt_service.py     ← JwtService (encode/decode/validate JWT)
│   │   │   │
│   │   │   ├── persistence/
│   │   │   │   ├── contracts.py       ← PersistenceProvider Protocol, UnitOfWork Protocol
│   │   │   │   ├── null_provider.py   ← NullPersistenceProvider
│   │   │   │   ├── registry.py        ← PersistenceProviderRegistry
│   │   │   │   ├── generic_repository.py ← SqlAlchemy Core CRUD + search repository
│   │   │   │   ├── domain_filter.py   ← domain filter list → WHERE clause
│   │   │   │   └── tenant_identity.py ← TenantDatabaseIdentityResolver
│   │   │   │
│   │   │   └── presentation/
│   │   │       ├── view_registry.py   ← ViewRegistry (XML view discovery)
│   │   │       └── dsl/
│   │   │           ├── loader.py      ← DslFileLoader (read XML from disk)
│   │   │           └── parser.py      ← DslParser (XML → RenderPlan dict)
│   │   │
│   │   ├── orm/
│   │   │   ├── model_proxy.py         ← ModelProxy + ModelAccessor (env.models[key])
│   │   │   ├── recordset.py           ← RecordSet (live record handle, lazy-loaded)
│   │   │   ├── commands.py            ← RelationalCommand tuples (M2M / O2M ops)
│   │   │   └── relational.py          ← process_relational_commands()
│   │   │
│   │   └── tenancy/
│   │       ├── context.py             ← thread-local tenant_id (set/get)
│   │       └── resolver.py            ← resolve_tenant_id() from Host / header / fallback
│   │
│   ├── foundation/                    ← Built-in foundation apps
│   │   ├── settings.py                ← FoundationSettings + ACTIVE_MODULES
│   │   ├── base/                      ← res.country, res.organization, res.user, ir.action, ir.menu
│   │   ├── auth/                      ← ir.session, login/logout/refresh/me endpoints
│   │   └── presentation/              ← PresentationKernel, DSL views, web client bootstrap
│   │
│   └── cli/
│       ├── main.py                    ← `ede` Click group
│       └── commands/
│           ├── server.py              ← `ede serve`
│           ├── worker.py              ← `ede worker`
│           ├── migrate.py             ← `ede migrate`
│           └── info.py                ← `ede info`
│
├── domains/
│   └── settings.py                    ← ACTIVE_DOMAINS = ["logistics", ...]
│
├── migrations/
│   └── env.py                         ← Alembic multi-tenant env.py
│
└── tests/
```

---

## App Layout (Per App)

Every app — whether foundation or user-domain — follows this layout:

```
<domain>/
└── <app_name>/
    ├── __manifest__.py       ← Required: app metadata dict
    ├── __init__.py           ← App package root; triggers import chain
    ├── models/
    │   ├── __init__.py       ← Imports all model modules (triggers registration)
    │   └── <model>.py        ← @api.model DomainModel subclass
    ├── api/
    │   └── controllers.py    ← @api.route_config RouteController subclasses
    ├── services/             ← Optional: pure Python service objects
    ├── migrations/
    │   └── versions/         ← Alembic migration scripts for this app
    └── views/                ← Optional: XML DSL view files
        └── *.xml
```

---

## The Manifest (`__manifest__.py`)

Every app declares a pure dict literal. The file is **AST-parsed** (not executed), so only
literal values are allowed — no function calls, no imports.

```python
# src/ede/foundation/base/__manifest__.py
{
    "name": "Base",
    "summary": "Core resources: country, organization, user",
    "description": "Foundation base app providing essential business resources.",
    "author": "THE_BLACK_BOX",
    "category": "foundation",
    "version": "1.0.0",
    "data": [
        "views/base.xml",
    ],
}
```

**Required keys**: `name`, `summary`, `description`, `author`, `category`, `version`

**Optional keys**:
- `"data"`: list of relative paths to XML view files (resolved from app root)

**Rule**: `"author"` must always be `"THE_BLACK_BOX"`.

---

## App Key Convention

App keys are **auto-generated** — never declared in the manifest. The loader derives them:

```
app_key = "{domain_type}.{app_folder_name}"
```

Examples:
| Domain Type | App Folder | App Key |
|---|---|---|
| `foundation` | `base` | `foundation.base` |
| `foundation` | `auth` | `foundation.auth` |
| `logistics` | `shipment` | `logistics.shipment` |

---

## App Activation (Settings-Driven)

No auto-discovery. Every app must be **explicitly listed** in settings.

### Foundation apps (`src/ede/foundation/settings.py`)

```python
ACTIVE_MODULES = ["base", "auth", "presentation"]
```

The loader translates each entry to `foundation.<name>` and imports
`ede.foundation.<name>`.

### User domain apps (`src/domains/settings.py`)

```python
ACTIVE_DOMAINS = ["logistics"]
```

Each domain may have its own `settings.py`:

```python
# src/domains/logistics/settings.py
ACTIVE_MODULES = ["shipment", "inventory"]
```

This activates `logistics.shipment` → imports `domains.logistics.shipment`.

---

## Boot Sequence (Deterministic)

```
boot_runtime_and_environment()
  │
  ├── 1. Import foundation settings  → apply ede.conf overrides
  ├── 2. Set runtime.settings
  │
  └── bootstrap_environment()
        │
        ├── 3. Register CRUD handlers on DomainModel
        ├── 4. Register adapters (persistence factories, message broker factories)
        │
        └── ModuleLoader.load_all()
              │
              ├── 5. load_foundation()
              │     For each app in ACTIVE_MODULES:
              │       a. Read + validate __manifest__.py (AST parse)
              │       b. import_module(app_package)        ← triggers __init__ → models/__init__
              │       c. Scan sys.modules for prefix → register DomainModels, event handlers
              │       d. Scan sys.modules for prefix → register RouteController routes
              │       e. Register AppSpec in registry
              │
              └── 6. load_domains()
                    For each domain in ACTIVE_DOMAINS:
                      For each app in domain.ACTIVE_MODULES:
                        (same a-e steps as foundation)
```

**Key**: The import in step (b) triggers the entire module tree for the app. Python's import
system ensures each module is only loaded once per process.

---

## Registry (Single Source of Truth)

`Registry` is the central in-memory index. It holds everything the runtime needs to dispatch
commands, route events, and resolve models.

| Store | Key Example | Value |
|---|---|---|
| `_models_by_key` | `"foundation.base"` | `Ping` class |
| `_commands` | `"presentation.list_views"` | `CommandTarget(model_cls, method_name)` |
| `_events` | `"base.ponged"` | `[handler_fn, ...]` |
| `_message_broker_factories` | `"inmemory"` | `InMemoryEventQueue.from_settings` |
| `_persistence_factories` | `"sqlalchemy"` | `SqlAlchemyPersistenceProvider.from_settings` |
| `_apps_by_key` | `"logistics.shipment"` | `AppSpec(...)` |

Key methods:

```python
registry.register_model(model_cls)
registry.get_model("foundation.base")
registry.resolve_command("shipment.create")      # → CommandTarget
registry.get_event_handlers("base.ponged")       # → [fn, ...]
registry.sorted_app_specs()                      # → deterministic order (foundation first)
registry.list_migration_version_locations()      # → List[Path]  (all Alembic versions dirs)
```

---

## Env (Runtime Context per Request)

`Env` is created once at server startup (base env), then **shallow-cloned** per request with
tenant and principal applied. It is passed to every controller and model.

```python
class Env:
    registry: Registry
    commands: CommandBus
    persistence: PersistenceProvider | None
    _event_queue: EventQueue | None
    tenant_id: str | None
    principal: Mapping | None       # authenticated user claims
    _active_uow: UnitOfWork | None  # set by env.transaction()

# Per-request cloning:
request_env = base_env.with_tenant_id("acme").with_principal(principal_claims)

# ORM access:
shipments = request_env.models["logistics.shipment"]
record = shipments.search([("status", "=", "pending")])

# Dispatch:
result = request_env.dispatch(Command(name="shipment.create", payload={...}))

# Transaction:
with request_env.transaction():
    record = shipments.create({"name": "S001"})
    record.write({"status": "confirmed"})
```

---

## Why This Structure Matters

### Explicit activation over auto-discovery

EDE requires every app to be listed in `settings.ACTIVE_MODULES`. Nothing loads
automatically. This may feel like more ceremony than Django's `INSTALLED_APPS` or
FastAPI's router scanning, but it provides critical guarantees:

- **Predictable boot order.** Foundation apps always load before domain apps.
  Within a domain, apps load in the order declared. This prevents import-time
  dependency failures that are invisible in auto-discovery systems.
- **No surprise registrations.** You can never accidentally activate an app by
  installing a package. Only explicitly listed apps affect the running system.
- **Clear surface area.** `ede info` shows exactly what is loaded. There is no
  "maybe this route exists" — if the app isn't in `ACTIVE_MODULES`, no routes,
  models, or handlers from it exist.

### One-way dependency rule

The app structure enforces the five-layer rule physically: `models/` never imports
from `api/`, and neither imports from `adapters/`. This isn't a convention — it's
a structural constraint. If you try to import FastAPI inside a model, you've made
a mistake that future infrastructure changes will punish.

### Per-app migrations

Each app owns its own `migrations/versions/` directory. When two teams develop two
apps simultaneously, their migrations never conflict. When you deactivate an app, its
migration history stays intact and can be re-applied to a new tenant at any time.

---

## Naming Conventions Summary

| Thing | Convention | Example |
|---|---|---|
| App key | `{domain_type}.{app_folder_name}` | `logistics.shipment` |
| Model key | `{domain_type}.{model_name}` (manual, in `@api.model`) | `res.country` |
| Table name | model key with `.` → `_` (or `__ede_table_name__` override) | `res_country` |
| Command name | `{app_key}.{verb}` | `shipment.create` |
| Event name | `{app_key}.{past_tense}` | `shipment.created` |
| View id | dotted namespace, `.` → `_` for filename | `hello.world` → `hello_world.xml` |
| Manifest author | Always `"THE_BLACK_BOX"` | — |
