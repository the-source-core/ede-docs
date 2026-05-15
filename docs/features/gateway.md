# Multi-Tenant Gateway

A SaaS control plane that provisions per-tenant databases on demand, publishes Traefik routes to direct traffic, and gives administrators a dedicated React SPA at port 8001.

```bash
# Start the gateway: app server on :8000 + admin API + SPA on :8001
ede serve gateway --config ede.gateway.conf

# Provision a tenant (typically from the admin UI; CLI shown for clarity).
curl -X POST http://localhost:8001/api/gateway/tenants \
    -H "Content-Type: application/json" \
    -d '{"key": "acme", "company_name": "Acme Logistics", "pool_id": "<pool_uuid>"}'
```

The worker thread picks up the pending tenant, runs migrations against a fresh database, and publishes the routing keys to Redis. Traefik picks up the new route immediately — `acme.example.com` now resolves to the app server.

---

## What you get

-   **`gateway.tenant`** — per-tenant lifecycle row (`pending → provisioning → ready → suspended`).
-   **`gateway.shared_pool`** — database connection-pool metadata (PostgreSQL host, max tenants).
-   **`GatewaySaasWorker`** — background thread that polls `pending` tenants and provisions them.
-   **`MigrationRunner`** — programmatic Alembic upgrade against any tenant DB.
-   **`TraefikRoutePublisher`** — pushes route definitions to Redis keys watched by Traefik.
-   **Dual-port `ede serve gateway`** — `:8000` for tenant traffic, `:8001` for the admin SPA + `/api/gateway/*`.
-   **`build:gateway` frontend variant** — separate Vite build that includes the SaaS-manager routes (`/wc/saas-manager/**`).

## How to use it

### Provision a tenant from code

```python
env.models["gateway.tenant"].create({
    "key": "acme",
    "company_name": "Acme Logistics",
    "pool_id": pool.id,
    "status": "pending",
})
```

The worker picks it up on its next poll. To watch progress, tail `ir.job` events or refresh the tenant row.

### Run migrations against an existing tenant

```bash
ede migrate upgrade -t acme --config ede.conf
```

Or programmatically:

```python
from ede.core.services.migration.runner import MigrationRunner

MigrationRunner.from_settings(settings).upgrade("acme")
```

### Suspend a tenant

```python
env.models["gateway.tenant"].browse(tenant.id).write({"status": "suspended"})
```

The next route publish drops Traefik's rule for that tenant; traffic returns 503 until you flip back to `ready`.

### Front-end build variant

```bash
cd src/frontend
bun run build:gateway
```

Outputs to the same `web-client/` asset directory but includes the SaaS-manager routes. Use this build when running `ede serve gateway`.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `GATEWAY_ADMIN_PORT` | `8001` | Port for the admin API + SPA. |
| `GATEWAY_REDIS_URL` | _(required for production)_ | Redis where Traefik watches for route changes. |
| `GATEWAY_DEFAULT_POOL_HOST` | _(per-deployment)_ | PostgreSQL host new tenants are placed on. |
| `GATEWAY_WORKER_POLL_SECONDS` | `5` | How often the provisioner scans for `pending` tenants. |

See `ede.conf.gateway.example` for a complete annotated config.

## Reference

-   Source: `src/ede/foundation/gateway/`
-   `MigrationRunner`: `src/ede/core/services/migration/runner.py`
-   Admin SPA routes: `src/frontend/src/gateway/`
-   Docker: `docker-compose.gateway.yml` includes Traefik v3.6 + Redis + Postgres + the gateway app.
-   Architecture: [Tenancy](../09-tenancy.md) for per-tenant DB isolation primitives.
