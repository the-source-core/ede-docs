# EDE Framework — Tenancy

## Overview

EDE uses **per-tenant database isolation**: each tenant gets its own SQLite file or
PostgreSQL database. Tenant context flows through the request via `Env.with_tenant_id()`.

There is **no shared database** between tenants. All queries are automatically scoped
to the current tenant's database.

---

## Tenant ID

A tenant ID is a short, lowercase alphanumeric string that identifies a tenant:
- Pattern: `[a-z0-9][a-z0-9_\-]{0,62}`
- Examples: `"acme"`, `"my-company"`, `"tenant_42"`

---

## Tenant Resolution Order

On each HTTP request, the tenant is resolved via `resolve_tenant_id()`:

```python
from ede.core.tenancy.resolver import resolve_tenant_id, TenantResolutionInput

tenant_id = resolve_tenant_id(TenantResolutionInput(
    host="acme.myapp.com",
    tenant_id_header_value=None,
    allowed_host_suffixes=[".myapp.com"],
    fallback_tenant_id="default",
))
# → "acme"  (derived from host: strip ".myapp.com" suffix)
```

Resolution order:
1. **Explicit header**: if `X-Tenant-Id` (or `TENANT_ID_HEADER_NAME`) header is present → use it (lowercased)
2. **Host suffix**: strip an allowed suffix from `Host` → use what remains
   - `acme.myapp.com` with suffix `.myapp.com` → `"acme"`
3. **Fallback**: use `DEFAULT_TENANT_ID` from settings (default: `"default"`)

The resolved value is validated against the tenant ID pattern. Invalid values raise an error.

---

## Configuration

`src/ede/foundation/settings.py`

```python
TENANT_ID_HEADER_NAME = "X-Tenant-Id"     # Header name for explicit tenant
TENANT_HOST_SUFFIXES = [".myapp.com"]      # List of allowed host suffixes
DEFAULT_TENANT_ID = "default"              # Fallback if nothing else resolves
```

---

## Per-Request Env Cloning

`Env.with_tenant_id()` returns a **shallow clone** of `Env` with `tenant_id` set.
The clone shares `registry`, `commands`, `_event_queue`, and `persistence` with the
original — only `tenant_id` (and `principal`) differ.

```python
# At server startup: one base env
base_env = Env(registry=registry, persistence=provider, ...)

# On each request:
tenant_id = resolve_tenant_id(...)
principal = request.state.principal  # from AuthMiddleware
request_env = base_env.with_tenant_id(tenant_id).with_principal(principal)

# All ORM ops on request_env hit the correct tenant DB
records = request_env.models["res.user"].search([])
```

This is **immutable context propagation** — no global mutable state. Different concurrent
requests safely use different `Env` instances with different tenant IDs.

---

## Thread-Local Tenant Context

`src/ede/core/tenancy/context.py`

Some parts of the system (notably `EventWorker`) need to set tenant context outside of
the normal `Env.with_tenant_id()` path. A thread-local context variable is provided:

```python
from ede.core.tenancy.context import set_current_tenant_id, get_current_tenant_id

set_current_tenant_id("acme")
tenant = get_current_tenant_id()    # → "acme"
set_current_tenant_id(None)         # clear
```

`EventWorker` calls `set_current_tenant_id(event.tenant_id)` before dispatching each event,
so event handlers can access the current tenant context.

---

## Per-Tenant Databases

Each tenant has its own database. The database name is derived from `DATABASE_NAME_PREFIX`
+ `tenant_id`:

| Config | Database Name |
|---|---|
| `DATABASE_NAME_PREFIX="db_ede"`, tenant=`"acme"` | `db_ede_acme` |
| `DATABASE_NAME_PREFIX="db_ede"`, tenant=`"beta"` | `db_ede_beta` |

**SQLite**:
```
{SQLITE_TENANT_DATABASE_DIR}/{DATABASE_NAME_PREFIX}_{tenant_id}.db
# e.g. .ede/databases/db_ede_acme.db
```

**PostgreSQL**:
```
{DATABASE_NAME_PREFIX}_{tenant_id}
# e.g. database name: db_ede_acme
```

The database must exist before migrations can run. `ede migrate upgrade -t acme` creates
the PostgreSQL database if it doesn't exist (via `admin.ensure_database_exists()`).

---

## Migration Scope

Migrations run **per tenant**:

```bash
ede migrate upgrade -t acme      # upgrade "acme" tenant DB to heads
ede migrate upgrade -t beta      # upgrade "beta" tenant DB to heads
```

The Alembic `env.py` uses the `--tenant` flag to build the correct connection URL.

---

## Tenant ID in Events

When an event is emitted, the current `env.tenant_id` is attached:

```python
# env.emit() internally:
event = Event.build(
    name=name,
    payload=payload,
    tenant_id=self.tenant_id,   # ← carried from Env
    correlation_id=...,
)
self._event_queue.enqueue(event)
```

Event handlers receive the event with `event.tenant_id` populated. The `EventWorker`
sets the thread-local context before calling handlers:

```python
set_current_tenant_id(event.tenant_id)
dispatcher.dispatch(event, env)
```

This ensures event handlers can access the correct tenant's DB.

---

## Tenant Isolation Summary

| Layer | How Tenancy Is Applied |
|---|---|
| HTTP | `resolve_tenant_id()` → `Env.with_tenant_id()` per request |
| ORM | `env.tenant_id` → repo builds per-tenant DB URL → separate DB connection |
| Events | `event.tenant_id` carried; `EventWorker` sets thread-local context |
| Migrations | `ede migrate upgrade -t <tenant>` per tenant |
| Sessions (`ir.session`) | Stored in tenant's own DB (not shared) |

---

## Adding a New Tenant

1. Provision the database (automatic for PostgreSQL if using `ede migrate upgrade`):
   ```bash
   ede migrate upgrade -t newtenant
   ```
   For SQLite: the file is created automatically when migrations run.

2. Create initial data (e.g. default organization, admin user):
   ```bash
   # Via a one-time seed command or admin API
   curl -X POST http://localhost:8000/api/auth/login \
     -H "X-Tenant-Id: newtenant" \
     -d '{"email": "admin@example.com", "password": "initial"}'
   ```

3. The tenant is now active — all requests with `X-Tenant-Id: newtenant` (or via Host
   suffix resolution) will use its own database.

---

## Why Database-per-Tenant?

### The alternatives and their problems

**Row-level security (shared DB):** Every query must include a `tenant_id = ?` filter.
A missing filter causes data leakage. As query complexity grows, enforcing this
consistently becomes the hardest part of the codebase. Auditing it is difficult.

**Schema-per-tenant (shared DB, separate schemas):** Better isolation than RLS, but
migrations must touch every schema, and a single DB server is still a bottleneck and
single point of failure for all tenants.

**Database-per-tenant:** Each tenant's data is physically isolated. There is no row-level
filter to forget. A leaked connection string exposes one tenant's data, not all tenants'.
Tenant databases can be independently backed up, restored, scaled, or migrated.

### How EDE makes database-per-tenant painless

**`Env.with_tenant_id()`** is a shallow clone — the base `Env` is created once at startup
and every request gets a cheap clone with `tenant_id` set. No connection pool per request,
no thread-local tricks. The persistence layer resolves the DB URL from `tenant_id` lazily.

**Migrations are per-tenant by design.** `ede migrate upgrade -t acme` runs all app
migrations for `acme`'s database. New tenants are onboarded by running their migration.
There is no global schema version to coordinate.

**`EventWorker` restores tenant context.** When an event is delivered, `EventWorker`
sets the thread-local `tenant_id` from `event.tenant_id`. Event handlers automatically
operate on the correct tenant's database through `env.models[...]` without any explicit
tenant filtering.
