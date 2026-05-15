# 8. Deploying

You have a working app on your laptop running against SQLite. This chapter walks the route to production: a containerised single-tenant deployment with PostgreSQL, then how the multi-tenant **gateway** mode works when you need a SaaS-shape distribution. By the end you can ship the same code that runs under `ede serve --with-worker` to a real server, with the standalone worker, real secrets, and the production checklist sorted.

```bash
# What changes from dev to prod, in one screen.
DATABASE_ENGINE=postgresql
JWT_SECRET_KEY=$(openssl rand -hex 32)
DEBUG=false
ENABLE_API_DOCS=false

docker compose up -d                  # boots the server + worker + Postgres
docker compose exec ede ede migrate upgrade -t system
```

By the end you know: the production checklist, how to run the standalone event worker, the Docker Compose layout, how to point at PostgreSQL, and when to flip into multi-tenant gateway mode.

## What you're going to deploy

A standard single-tenant production setup with three containers:

```
┌─────────────────────────────────────────────────────────────┐
│  ede-server      FastAPI / uvicorn                          │
│                  ede serve --no-worker                      │
│                                                             │
│  ede-worker      Background event processor                 │
│                  ede worker                                 │
│                                                             │
│  postgres        Per-tenant DB (system tenant here)         │
└─────────────────────────────────────────────────────────────┘
```

The worker runs in its own container so it can scale independently and crash without taking the HTTP server down.

## 1. The production checklist

These are the defaults you must change before any production traffic. Most live in `ede.conf` or as environment variables on the container:

| Setting | Dev default | Production target |
|---|---|---|
| `JWT_SECRET_KEY` | `change-me-in-production` | **A long random secret** — `openssl rand -hex 32` is fine. Stored in a real secret manager. |
| `DEBUG` | `False` (or `True` if you flipped it) | `False`. Production tracebacks leak internals. |
| `ENABLE_API_DOCS` | `False` (or `True` if you flipped it) | `False`. The `/docs` and `/redoc` OpenAPI pages expose every endpoint. |
| `DATABASE_ENGINE` | `sqlite` | `postgresql`. |
| `DATABASE_URL_<tenant>` | unset (SQLite path) | Real Postgres URL per tenant. |
| `DEFAULT_MESSAGE_BROKER_PROVIDER` | `inmemory` | `kafka` if you have a Kafka cluster; otherwise leave `inmemory` and run a single worker. |
| `JWT_ACCESS_TOKEN_EXPIRE_MINUTES` | `30` | Match your security policy. 15–30 minutes is typical. |
| `JWT_REFRESH_TOKEN_EXPIRE_DAYS` | `7` | Match your security policy. 7–30 days is typical. |
| CORS allowlist | (open) | Restrict to your real frontend origin(s). |
| TLS | (none) | Terminated at the reverse proxy or load balancer in front of the server. |

Run through every row before exposing the server to the internet. The `JWT_SECRET_KEY` default is **publicly known** — leaving it in place is equivalent to disabling authentication.

## 2. Switch to PostgreSQL

PostgreSQL is the only supported production engine. The schema specs that drive autogen handle SQLite and Postgres identically — you should not see schema differences between dev and prod.

Two settings:

```bash
DATABASE_ENGINE=postgresql
DATABASE_URL_system=postgresql+psycopg://ede:strong-password@db:5432/ede_system
```

The URL pattern is per tenant: `DATABASE_URL_<tenant_id>`. The framework looks up the URL for the current `Env.tenant_id` and routes every query there. Two tenants → two URLs → two databases.

After updating the config, run the full migration chain against the new DB:

```bash
ede migrate upgrade -t system
```

The reference DB used by `migrate generate` stays SQLite (it's a local convenience). The runtime DB is whatever you point each tenant at.

## 3. Run the worker as a separate process

In dev you used `ede serve --with-worker`, which spawns the worker as an in-process thread. In production the worker must be a **separate process** — that way you can scale it independently and the worker can crash without taking the HTTP server down:

```bash
# Process 1 — HTTP server only.
ede serve

# Process 2 — drain the event queue.
ede worker
```

The worker accepts these flags:

| Flag | Default | Purpose |
|---|---|---|
| `--poll-timeout-seconds` | `1.0` | How long the worker blocks waiting for the next event. |
| `--batch-size` | `25` | Max events processed per drain cycle. |
| `--max-attempts` | `5` | Retry ceiling before an event is marked failed. |

Run as many `ede worker` processes as your event throughput requires. They coordinate through the message broker (in-memory or Kafka), so concurrent workers are safe by construction.

## 4. The Docker layout

The repo ships three Docker Compose files. The default single-tenant one is the right starting point:

```yaml
# docker-compose.yml — single-tenant production.
services:
    db:
        image: postgres:16
        environment:
            POSTGRES_USER: ede
            POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
            POSTGRES_DB: ede_system
        volumes:
            - postgres-data:/var/lib/postgresql/data

    ede-server:
        build: .
        command: ede serve
        environment:
            DATABASE_ENGINE: postgresql
            DATABASE_URL_system: postgresql+psycopg://ede:${POSTGRES_PASSWORD}@db:5432/ede_system
            JWT_SECRET_KEY: ${JWT_SECRET_KEY}
            DEBUG: "false"
            ENABLE_API_DOCS: "false"
        ports:
            - "8000:8000"
        depends_on:
            - db

    ede-worker:
        build: .
        command: ede worker
        environment:
            DATABASE_ENGINE: postgresql
            DATABASE_URL_system: postgresql+psycopg://ede:${POSTGRES_PASSWORD}@db:5432/ede_system
            JWT_SECRET_KEY: ${JWT_SECRET_KEY}
        depends_on:
            - db

volumes:
    postgres-data:
```

Boot:

```bash
docker compose up -d --build
docker compose exec ede-server ede migrate upgrade -t system
```

The `migrate upgrade` step runs once against the empty database; subsequent boots find an up-to-date schema and start serving immediately.

For an external Kafka broker / Redis cache / S3-compatible storage, the repo also ships `docker-compose.services.yml` — overlay it with `docker compose -f docker-compose.yml -f docker-compose.services.yml up`. Keep `DEPLOYMENT_DOCKER.md` at the repo root open while wiring this — it's the canonical reference for every variant.

## 5. Reverse proxy + TLS

The container exposes plain HTTP on `:8000`. In production every byte should be encrypted on the wire. Run any reverse proxy that terminates TLS in front of `ede-server`:

| Proxy | Why you'd pick it |
|---|---|
| Caddy | Single binary, automatic Let's Encrypt. The simplest production-grade option. |
| Traefik | First-class Docker integration; the gateway variant (next section) uses Traefik directly. |
| nginx | Industry standard; pair with `certbot` for ACME. |

Whatever you pick, the proxy:

-   Listens on `:443`, terminates TLS, forwards to `ede-server:8000`.
-   Forwards `X-Forwarded-For` / `X-Forwarded-Proto` headers so the FastAPI app sees the real client IP and scheme.
-   Restricts CORS at the proxy edge if your CORS policy is uniform across endpoints.

## 6. Multi-tenant — the gateway mode

When one deployment serves many isolated customer organisations, switch to the **gateway** variant. The framework ships a control plane that provisions per-tenant databases on demand and publishes routing rules to Traefik so each tenant gets its own subdomain.

```bash
ede serve gateway
```

What changes:

-   The server boots in **dual-port** mode: `:8000` serves the regular app for whichever tenant the request resolves to; `:8001` serves the SaaS admin SPA.
-   A background worker polls `gateway.tenant` rows in `status=pending` and provisions a new database + Docker network + Traefik route for each one.
-   Traefik routing is pushed to a Redis key namespace under `traefik/http/routers/tenant-<key>/**` — Traefik picks up new routes within seconds.
-   Each tenant's database is migrated automatically on first boot using the same `ede migrate upgrade` plumbing as single-tenant.

The gateway variant comes with its own Compose file:

```bash
docker compose -f docker-compose.gateway.yml up -d
```

For the full topology — Traefik wiring, tenant provisioning lifecycle, admin SPA URLs — see [Features: Multi-Tenant Gateway](../features/gateway.md). It is opt-in: `foundation.gateway` is **not** in the default `ACTIVE_MODULES` list. Enable it explicitly when you need it.

## 7. Backups

EDE writes data to two surfaces. Both need backups.

| Surface | What's in it | Back up how |
|---|---|---|
| Per-tenant PostgreSQL DB | All business data, user accounts, sessions, audit logs. | `pg_dump` per tenant, on a schedule. Keep at least 7 daily + 4 weekly + 12 monthly snapshots. |
| Storage backend(s) | Uploaded files, generated documents. Default backend is local FS; configure cloud connectors for S3 / Google Drive in production. | Whatever backup tooling your chosen backend supports. For local FS, `rsync` to off-host storage. |

Restore drill quarterly. An untested backup is wishful thinking, not a backup.

## 8. Observability

Three signals to wire before things go wrong:

1.  **HTTP logs** — uvicorn already structured-logs every request. Pipe stdout to your log aggregator (Loki, CloudWatch, Datadog, whatever).
2.  **Worker logs** — `ede worker` logs every event it processes plus retry attempts. Same destination as HTTP logs.
3.  **Health endpoint** — `GET /health` returns 200 when the server is alive. Wire it as a liveness probe on your container orchestrator.

For deeper signals — query timing, slow handlers, queue depth — the [QA Automation](../features/qa-automation.md) and dataset features let you build dashboards on top of the same ORM, but you can ship without them on day one.

## 9. Shipping a new release

The standard sequence for promoting changes from dev → prod:

1.  Open a branch. Make the model / view / migration / permission changes.
2.  `ede migrate generate --app <app>` to scaffold any schema migration. Author the body if autogen wasn't enough.
3.  Run `mkdocs build --strict` if you touched docs, and the test suite (`./run_tests.sh`) for everything else.
4.  Merge. Build a container image with the new commit tagged.
5.  On the production host: pull the new image, `docker compose up -d` (rolling restart).
6.  Run `docker compose exec ede-server ede migrate upgrade -t all` once. Migrations are idempotent — re-running is safe.
7.  Watch logs for the first few minutes. The `web.client.command.done` events that connected clients receive will trigger automatic form reloads if you changed any field schema.

## 10. The deployment checklist (one place to scan)

Before flipping any DNS or routing real traffic:

-   [ ] `JWT_SECRET_KEY` is a real random secret, not the default.
-   [ ] `DEBUG=false`, `ENABLE_API_DOCS=false`.
-   [ ] `DATABASE_ENGINE=postgresql`, every tenant has a URL, every URL works.
-   [ ] `ede migrate upgrade -t <tenant>` ran cleanly on every tenant.
-   [ ] HTTP server and event worker are separate processes (or containers).
-   [ ] TLS termination is in front of the server, not on it.
-   [ ] CORS is restricted to your real frontend origin(s).
-   [ ] Logs aggregate somewhere queryable.
-   [ ] `/health` is wired to a liveness check.
-   [ ] Postgres + storage backend are backed up on a tested schedule.
-   [ ] Secrets live in your secret manager, not in `ede.conf` committed to git.

---

## What you just did

-   Walked the production checklist: secret, debug, docs, CORS, TLS, DB engine.
-   Switched from SQLite to PostgreSQL by changing two settings.
-   Ran the **event worker as a separate process** with `ede worker` rather than `--with-worker`.
-   Booted a containerised setup with `docker compose` — server, worker, Postgres.
-   Saw the upgrade path to **multi-tenant gateway** mode for SaaS distributions.
-   Picked up the deployment checklist that catches the common production misconfigurations.

That concludes the tutorial track. From here you build. The [Features](../features/index.md) section is the catalogue of every shipped platform capability you can reach for; the [Architecture](../01-overview.md) section is the deep reference when you want to know exactly how a layer is wired.
