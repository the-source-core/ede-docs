<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Gateway — Implementation Docs

**Module:** `foundation.gateway` (`src/ede/foundation/gateway/`)
**Roadmap:** [roadmap/foundation/gateway/](../roadmap/foundation/gateway/README.md)
**Status:** ✅ Delivered (baseline — pre-roadmap)
**Layer:** Foundation engine — platform substrate (multi-tenant control plane)

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
The gateway is the **SaaS control plane** for an EDE deployment that serves many tenants from one set of infrastructure. It owns tenant identity, drives provisioning end-to-end (PostgreSQL database + Docker app container + Traefik route), and exposes both an admin REST API and a dedicated React SPA for operating the fleet. Two core models: `gateway.tenant` (lifecycle registry) and `gateway.shared_pool` (pool admission for shared-tier tenants). A background worker (`GatewaySaasWorker`) polls pending tenants and drives them through `pending → provisioning → active`; Traefik routes are pushed to Redis under `traefik/http/routers/tenant-{key}/**` in real time. The control plane runs as a separate dual-port server invocation (`ede serve gateway`).
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Running EDE as a multi-tenant SaaS needs a control plane that owns tenant lifecycle (create / provision DB / wire routes / activate / suspend) without each operator running the steps manually. Gateway is that plane. Without it, every new tenant would require manual `createdb` + `alembic upgrade` + `docker run` + Traefik config edits — error-prone, slow, and impossible to scale.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **SaaS administrators** open the SaaS-manager SPA at `https://<GATEWAY_ADMIN_HOSTNAME>/wc/saas-manager/**` (port `8001`) to create tenants, assign tiers, manage shared pools, and inspect lifecycle status.
- **Self-onboarding consumers** call `POST /api/gateway/public/signup` to register a new tenant; the gateway inserts a `gateway.tenant` row with `status=pending` and the worker takes it from there.
- **Programmatic consumers** call `/api/gateway/tenants/**` (admin), `/api/gateway/pools/**` (shared-pool ops), or `/api/gateway/traefik/**` (Traefik diagnostics).
- **Other foundation modules** reuse `MigrationRunner` (kernel-level service, `src/ede/core/services/migration/runner.py`) for any programmatic Alembic upgrade — it is not gateway-specific.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
admin UI / API ──> gateway.tenant row (status=pending)
                        │
                        ▼
                GatewaySaasWorker (polls)
                        │
                        ▼
                MigrationRunner ── provision per-tenant DB
                        │
                        ▼
                TraefikRoutePublisher ── traefik/http/routers/tenant-{key}/**
                        │
                        ▼
                gateway.tenant.status = active
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `gateway.tenant` | Tenant registry. Lifecycle Enum `pending → provisioning → active → suspended \| error`. Holds tier (`shared`/`dedicated`), DB name, container id, shared-pool back-reference, error message. | [models/tenant.py](../src/ede/foundation/gateway/models/tenant.py) |
| `gateway.shared_pool` | Pool of EDE app containers serving shared-tier tenants. Tracks capacity, routing weight, lifecycle. | [models/shared_pool.py](../src/ede/foundation/gateway/models/shared_pool.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `GatewaySaasWorker` | Background thread polling `gateway.tenant` rows with `status=pending` and driving provisioning. | [services/saas_worker.py](../src/ede/foundation/gateway/services/saas_worker.py) |
| `DbProvisioner` | Creates per-tenant PostgreSQL database and runs Alembic upgrade via `MigrationRunner`. | [services/db_provisioner.py](../src/ede/foundation/gateway/services/db_provisioner.py) |
| `DockerProvisioner` | Launches dedicated-tier per-tenant Docker containers on the configured Docker network. | [services/docker_provisioner.py](../src/ede/foundation/gateway/services/docker_provisioner.py) |
| `TraefikRoutePublisher` | Pushes per-tenant routes to Traefik via the Redis provider under `traefik/http/routers/tenant-{key}/**`. | [services/traefik_publisher.py](../src/ede/foundation/gateway/services/traefik_publisher.py) |
| `TraefikApi` | Read-only client for Traefik REST API — health checks + dashboard probes. | [services/traefik_api.py](../src/ede/foundation/gateway/services/traefik_api.py) |
| `MigrationRunner` (kernel; **shared primitive**) | Programmatic Alembic upgrade with thread-safe lock. Reused outside gateway too. | [src/ede/core/services/migration/runner.py](../src/ede/core/services/migration/runner.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `gateway.tenant.suspend` | Admin action | Validates `status=active`; flips to `suspended`. |
| `gateway.tenant.reactivate` | Admin action | Validates `status in (suspended, error)`; flips to `pending` (worker re-drives). |
| `gateway.shared_pool.provision` | Admin action | Spins up a new shared-pool container. |
| `gateway.shared_pool.restart` | Admin action | Restarts the pool container. |
| `gateway.shared_pool.stop` | Admin action | Stops the pool container. |
| `ede.create` on `gateway.tenant` (with `status=pending`) | Self-signup or admin create | Worker picks up and provisions. |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `ede.record.created` (`gateway.tenant`) | New tenant row inserted | `GatewaySaasWorker` (via polling, not event-subscription today) |
| `ede.record.updated` (`gateway.tenant`) | Status / config change | Future billing/metering subscribers |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `/api/gateway/tenants/**` | Admin CRUD on tenants. | [api/tenant_routes.py](../src/ede/foundation/gateway/api/tenant_routes.py) |
| `/api/gateway/pools/**` | Admin CRUD + lifecycle on shared pools. | [api/pool_routes.py](../src/ede/foundation/gateway/api/pool_routes.py) |
| `/api/gateway/traefik/**` | Traefik dashboard / health diagnostics. | [api/traefik_routes.py](../src/ede/foundation/gateway/api/traefik_routes.py) |
| `/api/gateway/public/signup` | Self-onboarding signup endpoint (unauthenticated). | [api/public_routes.py](../src/ede/foundation/gateway/api/public_routes.py) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.gateway.tenant.delete` | Blocks deletion of tenants in `status=active` (must suspend first). |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
```
pending ─> provisioning ─> active ─> suspended ─> pending (reactivate)
                       └─> error ────────────────┘ (reactivate)
```
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` (in `src/ede/foundation/settings.py`): `gateway` is **NOT** in the default list. It is activated by running `ede serve gateway`, which loads a gateway-flavored settings profile and includes the gateway app in the loader.
- `ACTIVE_DOMAINS`: _not applicable — foundation app_.
- Manifest `depends`: `foundation.base`, `foundation.auth`.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `GATEWAY_TENANT_ID` | `str` | `gateway` | `EDE_GATEWAY_TENANT_ID` | Tenant id under which gateway control-plane data (tenant registry, pools) is stored. |
| `GATEWAY_ADMIN_HOSTNAME` | `str` | `admin.localhost.net` | `EDE_GATEWAY_ADMIN_HOSTNAME` | Hostname the SaaS-manager admin SPA + API serves on (port `8001`). |
| `GATEWAY_HOST_SUFFIX` | `str` | `.localhost.net` | `EDE_GATEWAY_HOST_SUFFIX` | Host suffix appended to a tenant key to form its public hostname (`acme.localhost.net`). |
| `GATEWAY_APP_BACKEND_URL` | `str` | `http://ede-gateway:8000` | `EDE_GATEWAY_APP_BACKEND_URL` | Upstream URL the Traefik routes point at for tenant traffic. |
| `DOCKER_NETWORK` | `str` | `ede-saas` | `EDE_DOCKER_NETWORK` | Docker network used by spawned per-tenant containers. |
| `DOCKER_EDE_IMAGE` | `str` | `ede-framework:latest` | `EDE_DOCKER_EDE_IMAGE` | Image launched for dedicated-tier tenant containers (and shared-pool members). |
| `DOCKER_APP_PORT` | `int` | `8000` | `EDE_DOCKER_APP_PORT` | App port inside the spawned container. |
| `DOCKER_APP_ENV_OVERRIDES` | `str` (JSON) | `""` | `EDE_DOCKER_APP_ENV_OVERRIDES` | JSON-encoded extra env vars merged into the spawned container's environment. |
| `TRAEFIK_ENTRYPOINT` | `str` | `web` | `EDE_TRAEFIK_ENTRYPOINT` | Traefik entrypoint name used in per-tenant router definitions. |
| `TRAEFIK_API_URL` | `str` | `http://traefik:8080` | `EDE_TRAEFIK_API_URL` | Traefik read-only REST API (health + dashboard probes). |
| `REDIS_HOST` | `str` | `redis` | `EDE_REDIS_HOST` | Redis host backing the Traefik Redis provider. |
| `REDIS_PORT` | `int` | `6379` | `EDE_REDIS_PORT` | Redis port. |
| `REDIS_PASSWORD` | `str` | `""` | `EDE_REDIS_PASSWORD` | Redis password (empty = no auth). |
| `REDIS_TRAEFIK_ROOT_KEY` | `str` | `traefik` | `EDE_REDIS_TRAEFIK_ROOT_KEY` | Top-level Redis key namespace under which `http/routers/tenant-{key}/**` entries are written. |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| [data/gateway_roles.xml](../src/ede/foundation/gateway/data/gateway_roles.xml) | Gateway admin / operator roles. |
| [data/ir.rbac.permission.csv](../src/ede/foundation/gateway/data/ir.rbac.permission.csv) | RBAC permissions for gateway models and admin API. |
| [data/gateway_menus.xml](../src/ede/foundation/gateway/data/gateway_menus.xml) | SaaS-manager menu tree. |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Baseline | Tenant lifecycle + provisioning worker + Traefik publisher + admin API + SaaS-manager SPA | ✅ Delivered (pre-roadmap) | [roadmap/foundation/gateway/README.md](../roadmap/foundation/gateway/README.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Tenant lifecycle (registry + state machine + suspend/reactivate commands) | `gateway.tenant` | [models/tenant.py](../src/ede/foundation/gateway/models/tenant.py) | [roadmap README](../roadmap/foundation/gateway/README.md) |
| Shared-pool admission for shared-tier tenants | `gateway.shared_pool` | [models/shared_pool.py](../src/ede/foundation/gateway/models/shared_pool.py) | [roadmap README](../roadmap/foundation/gateway/README.md) |
| Background provisioning worker (polls `pending` → drives DB + container + route) | `gateway.tenant` | [services/saas_worker.py](../src/ede/foundation/gateway/services/saas_worker.py), [services/db_provisioner.py](../src/ede/foundation/gateway/services/db_provisioner.py), [services/docker_provisioner.py](../src/ede/foundation/gateway/services/docker_provisioner.py) | [roadmap README](../roadmap/foundation/gateway/README.md) |
| Traefik route publishing via Redis (push, real-time) | — | [services/traefik_publisher.py](../src/ede/foundation/gateway/services/traefik_publisher.py), [services/traefik_api.py](../src/ede/foundation/gateway/services/traefik_api.py) | [roadmap README](../roadmap/foundation/gateway/README.md) |
| Per-tenant migration runner (programmatic Alembic, thread-safe) | — | [src/ede/core/services/migration/runner.py](../src/ede/core/services/migration/runner.py) | [roadmap README](../roadmap/foundation/gateway/README.md) |
| Dual-port server CLI (`ede serve gateway`) | — | `server.py` (CLI group) | [roadmap README](../roadmap/foundation/gateway/README.md) |
| Admin HTTP API surface (`/api/gateway/tenants`, `/pools`, `/traefik`, `/public`) | `gateway.tenant`, `gateway.shared_pool` | [api/](../src/ede/foundation/gateway/api/) | [roadmap README](../roadmap/foundation/gateway/README.md) |
| Dedicated React SaaS-manager UI build variant | — | [src/frontend/src/gateway/](../src/frontend/src/gateway/); `bun run build:gateway`; `docker-compose.gateway.yml` | [roadmap README](../roadmap/foundation/gateway/README.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| No per-tenant resource quotas (CPU / memory / request rate / DB connection caps). | 🟠 High | [roadmap README — Known Gaps](../roadmap/foundation/gateway/README.md#known-gaps) |
| No billing / metering event hooks — provisioning does not emit billable events. | 🟠 High | [roadmap README — Known Gaps](../roadmap/foundation/gateway/README.md#known-gaps) |
| No tenant DB backup / export / restore flow. | 🟠 High | [roadmap README — Known Gaps](../roadmap/foundation/gateway/README.md#known-gaps) |
| SaaS-manager UI auth is gateway-local; not federated with `foundation.auth` SSO. | 🟡 Medium | [roadmap README — Known Gaps](../roadmap/foundation/gateway/README.md#known-gaps) |
| No tenant data-export GDPR flow (right-to-erasure / right-to-portability). | 🟡 Medium | [roadmap README — Known Gaps](../roadmap/foundation/gateway/README.md#known-gaps) |
| Shared-pool autoscaling is manual — no pressure-driven container scale-out. | 🟡 Medium | [roadmap README — Known Gaps](../roadmap/foundation/gateway/README.md#known-gaps) |
| No demo data for `gateway.tenant` / `gateway.shared_pool` — gateway is operator-driven. | 🟢 Low (likely exempt) | [roadmap README — Known Gaps](../roadmap/foundation/gateway/README.md#known-gaps) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- The status field uses `fields.Enum`, **not** `fields.Selection`. `Selection` does not exist in the EDE kernel field set.
- `MigrationRunner` lives in `src/ede/core/services/migration/runner.py` (kernel), **not** under `foundation.gateway`. It is shared infrastructure; consume it, do not duplicate it.
- Gateway is not in the default `ACTIVE_MODULES`. To run it, use `ede serve gateway` (CLI subcommand) — it loads a gateway-flavored settings profile and binds dual ports (`8000` app, `8001` admin).
- The frontend is unified: `src/frontend/src/gateway/` lives inside the main webclient tree. `bun run build:gateway` toggles the `__GATEWAY_BUILD__` Vite define to include the `/wc/saas-manager/**` routes; standard `bun run build` excludes them.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Baseline tables (`gateway.tenant`, `gateway.shared_pool`) ship in the module's initial Alembic migration; no backfills required.
- Tenant provisioning per row runs Alembic upgrade against the new per-tenant DB via `MigrationRunner` — head computation respects the multi-app version-locations setup.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Gateway Admin | Full CRUD on `gateway.tenant`, `gateway.shared_pool`; lifecycle commands; Traefik diagnostics. See [data/gateway_roles.xml](../src/ede/foundation/gateway/data/gateway_roles.xml) + [data/ir.rbac.permission.csv](../src/ede/foundation/gateway/data/ir.rbac.permission.csv). |
| Public (unauthenticated) | `POST /api/gateway/public/signup` only. |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [foundation.base](foundation-base.md) — `res.*` masters; gateway depends on it.
- [foundation.auth](foundation-auth.md) — gateway depends on it for admin authentication.
- [Platform execution rules](../roadmap/platform/00-execution-rules.md)
- `docker-compose.gateway.yml` (repo root) — compose stack including gateway, app, Traefik, Redis.
- [src/frontend/src/gateway/](../src/frontend/src/gateway/) — SaaS-manager SPA source (gateway build variant).
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-14. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
