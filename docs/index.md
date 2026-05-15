# EDE Framework

**Enterprise Domain Enabler** — a Domain-Driven Design **ERP platform kernel** for Python enterprise systems. Build pluggable business domains (Logistics, CRM, HR, Finance, Procurement…) on top of one shared platform: cross-cutting masters, command/event bus, multi-tenant persistence, declarative views, lifecycle hooks, RBAC, and a React web client — all wired together by an explicit module registry, never by auto-discovery.

---

## Where to start

<div class="grid cards" markdown>

-   :material-rocket-launch:{ .lg } **New here?**

    ---

    Install the framework, boot the server, and build your first module in under an hour.

    [:octicons-arrow-right-24: Start the tutorial](tutorial/00-installation.md)

-   :material-sitemap:{ .lg } **Designing a feature?**

    ---

    Read the architecture: five-layer model, command/event flow, ORM, tenancy, presentation DSL.

    [:octicons-arrow-right-24: Architecture overview](01-overview.md)

-   :material-package-variant:{ .lg } **Looking up an engine?**

    ---

    Browse the built-in foundation features: auth, approval, workflow, dataset, dashboard, storage, gateway, and more.

    [:octicons-arrow-right-24: Browse the features](features/index.md)

</div>

---

## What EDE is

EDE is the platform on which multiple business domains are enabled as pluggable modules. Two layers, one strict dependency direction:

```
┌─────────────────────────────────────────────────────────────┐
│  DOMAIN LAYER  (src/domains/)                               │
│  logistics.*, crm.*, hr.*, finance.*, procurement.*         │
│  Domain-specific models, business logic, workflows          │
│  CAN reference foundation.base — CANNOT reference each other│
└────────────────────┬────────────────────────────────────────┘
                     │ depends on
┌────────────────────▼────────────────────────────────────────┐
│  PLATFORM LAYER  (src/ede/foundation/)                      │
│  res.*, ir.*, foundation.*                                  │
│  Cross-cutting masters, shared engines, platform services   │
│  CANNOT reference any domain package                        │
└─────────────────────────────────────────────────────────────┘
```

Domains depend on the platform. The platform never depends on a domain. Domains never depend on each other — if two domains need the same concept, it belongs in the platform.

## Architecture at a glance

EDE separates a request into five layers, each replaceable without touching the others:

1. **Experience & Access** — FastAPI HTTP adapter, route controllers, middleware (JWT, tenancy resolution, web-push SSE).
2. **Application** — synchronous `CommandBus` dispatches typed `Command` objects to model handlers; emits `Event`s downstream.
3. **Domain** — `DomainModel` subclasses with declarative kernel fields, `@api.on_command` / `@api.on_event` / `@api.on_hook` handlers.
4. **Platform Capability** — persistence, eventing, tenancy, presentation, RBAC — all contract-driven (swap SQLAlchemy for another store without touching domain code).
5. **Infrastructure Adapters** — concrete implementations: FastAPI, SQLAlchemy, Kafka, Redis, Traefik.

## Why EDE

-   **Dependencies flow downward only** — Domain never imports from HTTP or Persistence.
-   **Explicit activation** — no auto-discovery; every app must be listed in `ACTIVE_MODULES` / `ACTIVE_DOMAINS`.
-   **Protocol-driven infrastructure** — persistence, events, and auth are contracts, not implementations.
-   **Immutable `Env` propagation** — per-request context is a shallow clone of the base Env; no global mutable state.
-   **Per-tenant isolation** — each tenant has its own database; no shared state between tenants.
-   **Command → Event flow** — mutations are synchronous commands; notifications are asynchronous events with retry.
-   **Registry as single source of truth** — all models, commands, events, and providers are indexed at boot; nothing discovered at runtime.

## Quick install

```bash
# 1. Obtain a copy of the framework source.
cd ede-framework
pip install -e ".[dev]"
cp ede.conf.example ede.conf
ede migrate upgrade -t system
ede serve --with-worker
```

Open `http://localhost:8000` — the React web client boots against the in-process server.

Full walk-through with explanations: [Installation tutorial](tutorial/00-installation.md).

## Pick your reading order

| You are… | Start with |
|---|---|
| New developer onboarding to the platform | [Tutorial: Getting Started](tutorial/00-installation.md) → [Your First Module](tutorial/01-your-first-module.md) |
| Adding a new domain or feature | [Developer Guide](13-dev-guide.md) → [App Structure & Loading](02-app-structure.md) |
| Investigating the request path | [Architecture: Overview](01-overview.md) → [Command & Event Bus](04-command-event-bus.md) → [HTTP Layer](07-http-layer.md) |
| Wiring permissions or record rules | [Permissions](18-permissions.md) → [Record Rules](features/record-rules.md) → [Security & Authorization](features/security.md) |
| Writing a foundation engine | [Foundation Apps](11-foundation-apps.md) → browse the [Features](features/index.md) section |
| Building the UI / presentation | [Presentation DSL](10-presentation-dsl.md) → [Features: Presentation](features/presentation.md) |
