# EDE Framework — Overview

## What Is EDE?

**EDE (Enterprise Digital Engine)** is a Domain-Driven Design (DDD) platform kernel for Python.
Its core promise: protect business logic from infrastructure churn. Databases, message brokers,
and HTTP frameworks are replaceable; domain truth is not.

EDE enforces a strict separation of concerns through a **five-layer architecture** where
dependencies only flow downward — the domain never imports from HTTP or persistence.

---

## Five-Layer Model

```
┌─────────────────────────────────────────────────────┐
│  1. Experience & Access  (HTTP Controllers)          │
│     Entry point. Translates HTTP → Command.          │
├─────────────────────────────────────────────────────┤
│  2. Application          (Command Dispatch)          │
│     Orchestrates. Does not own business rules.       │
├─────────────────────────────────────────────────────┤
│  3. Domain               (DomainModel + Fields)      │
│     Business truth. No HTTP, no SQL, no Kafka.       │
├─────────────────────────────────────────────────────┤
│  4. Platform Capability  (Persistence, Events,       │
│                           Tenancy, Observability)    │
│     Cross-cutting infrastructure contracts.          │
├─────────────────────────────────────────────────────┤
│  5. Infrastructure Adapters (FastAPI, SQLAlchemy,    │
│                              Kafka, InMemory)         │
│     Replaceable. Vendor lock-in ends here.           │
└─────────────────────────────────────────────────────┘
```

**Rule**: Layer N may only import from layer N+1 or lower. Domain (3) never imports from HTTP
(1) or Persistence adapters (5).

---

## Request Flow (End to End)

```
HTTP Request
  └─► FastAPI ASGI app
        └─► AuthMiddleware      (validates JWT → sets request.state.principal)
              └─► LogMiddleware  (timing, correlation IDs)
                    └─► FastApiHttpAdapter
                          └─► Env built per-request (tenant_id resolved, principal attached)
                                └─► RouteController.method()   (@api.route handler)
                                      └─► env.dispatch(Command(name="...", payload={...}))
                                            └─► CommandBus.dispatch()
                                                  └─► Registry.resolve_command(name)
                                                        └─► ModelClass()  (new instance)
                                                              └─► model.handler(cmd)
                                                                    └─► return result (dict/list)
                                              ◄─── JSONResponse
```

---

## Event Flow (Outbox-safe)

```
model.emit("event.name", payload)
  └─► Env.emit()
        └─► EventQueue.enqueue(Event)      (in-memory or Kafka)

EventWorker (separate thread / process)
  └─► EventQueue.dequeue(batch)
        └─► EventDispatcher.dispatch(event, env)
              └─► Registry.get_event_handlers(event.name)
                    └─► handler(event, env)     (@api.on_event function)
```

No dual-write. Business transaction completes first; broker receives after commit.

---

## Public API Surface

All framework entry points for application developers live in `ede.core.api`:

```python
from ede.core import api

@api.model("logistics.shipment")          # registers a DomainModel
class Shipment(DomainModel): ...

@api.on_command("shipment.create")        # registers a command handler
def create(self, cmd): ...

@api.on_event("shipment.created")         # registers an async event handler
def on_shipment_created(event, env): ...

@api.route_config(prefix="/shipments")    # HTTP controller prefix
class ShipmentController(RouteController): ...

@api.route("/", methods=["GET"])          # HTTP route
def list_shipments(self): ...
```

---

## Technology Stack

| Component        | Technology                          |
|------------------|-------------------------------------|
| HTTP server      | FastAPI + Uvicorn                   |
| ORM / Migrations | SQLAlchemy 2.x Core + Alembic       |
| Databases        | SQLite (dev) / PostgreSQL (prod)    |
| Message broker   | In-memory (dev) / Kafka (prod)      |
| Auth             | PyJWT (HS256)                       |
| CLI              | Click                               |
| Settings         | pydantic-settings                   |
| Python           | 3.10+                               |
| Tests            | pytest + pytest-cov                 |

---

## CLI Commands

```bash
ede serve                         # start FastAPI HTTP server
ede serve --with-worker           # dev mode: server + in-memory worker thread
ede worker                        # standalone event worker process
ede migrate generate -m "msg"     # generate Alembic migration for all apps
ede migrate upgrade -t acme       # upgrade tenant DB to latest heads
ede info                          # list loaded apps and registered models
```

---

## Core Concepts At a Glance

| Concept | What It Is |
|---|---|
| `DomainModel` | A Python class decorated with `@api.model(key)`. The unit of domain logic. |
| `Command` | An intent to change state. Dispatched via `env.dispatch(cmd)`. |
| `Event` | A fact that something happened. Emitted via `env.emit(name, payload)`. |
| `Env` | Per-request runtime context — holds registry, persistence, tenant, principal. |
| `Registry` | In-memory index of models, commands, events, and providers. Built at boot. |
| `RecordSet` | Live handle to 0-N records of a model. Lazy-loaded, relational-aware. |
| `CommandBus` | Sync dispatch: resolves handler from Registry, calls model method. |
| `EventQueue` | Async FIFO queue for events (in-memory or Kafka). |
| `RouteController` | A class that groups HTTP route handlers. Adapts HTTP → Command. |

---

## Why This Architecture?

### The core problem it solves

Enterprise systems accumulate logic in the wrong places: business rules end up in HTTP
handlers, persistence queries leak into domain objects, and every new integration requires
touching code that was already working. EDE prevents this by making each layer's
responsibility explicit and enforcing one-way dependencies between them.

### What you get in return

**Longevity without rewrites.** When FastAPI is eventually replaced, you rewrite one
adapter. When you move from SQLite to PostgreSQL, you change one config line. When you
move from in-memory events to Kafka, you change one config line. Domain models don't change.

**Safe growth.** New features are added by writing new command handlers or event listeners
— not by modifying existing working code. A new notification system doesn't touch the
shipment domain. A new analytics pipeline doesn't touch the command that creates orders.

**Confidence in tests.** Domain models are plain Python classes. They can be unit-tested
without a database, without HTTP, and without Kafka. Infrastructure contracts accept null
and in-memory implementations specifically for this purpose.

**Multi-tenant without effort.** Per-request `Env` cloning means no global state leaks
between tenants. Every model, every query, every event automatically operates on the
correct tenant's database because `env.tenant_id` flows through every operation.

**Observable by default.** The Registry is the single source of truth for what the system
does. `ede info` shows every loaded app. Every command dispatch fires lifecycle hooks that
can emit audit events. Every field change can be tracked and reacted to.

---

## Further Reading

- [02 — App Structure & Loading](02-app-structure.md)
- [03 — Domain Model & Fields](03-domain-model.md)
- [04 — Command & Event Bus](04-command-event-bus.md)
- [05 — ORM Layer (RecordSet / ModelProxy)](05-orm-layer.md)
- [06 — Persistence](06-persistence.md)
- [07 — HTTP Layer](07-http-layer.md)
- [08 — Authentication (JWT + Sessions)](08-authentication.md)
- [09 — Tenancy](09-tenancy.md)
- [10 — Presentation DSL (Views)](10-presentation-dsl.md)
- [11 — Foundation Apps](11-foundation-apps.md)
- [12 — Migrations](12-migrations.md)
- [13 — Developer Guide (How to Add an App)](13-dev-guide.md)
