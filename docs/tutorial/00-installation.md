# Getting Started — Install & Run

This chapter takes you from an empty machine to a running EDE Framework server with the React web client live in your browser. Allow ~15 minutes.

## Prerequisites

You need:

-   **Python 3.10 or newer** — `python3 --version` should print `3.10.x` or higher.
-   **git**
-   **A working `pip`** (bundled with modern Python).
-   (Optional, for the frontend dev server) **[Bun](https://bun.sh/)** — npm/yarn are not used in this repo.
-   (Optional, for production-shape testing) **PostgreSQL 14+** or **Docker**. The default development DB is SQLite — no setup needed.

## 1. Clone the repository

```bash
git clone git@github.com:the-framework/ede-framework.git
cd ede-framework
```

## 2. Create a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate          # fish:   source .venv/bin/activate.fish
```

## 3. Install the framework + dev tooling

```bash
pip install -e ".[dev]"
```

The `-e` flag installs the package in **editable** mode — code changes you make under `src/` are picked up without reinstalling. The `[dev]` extra pulls in `pytest`, `ruff`, and other development dependencies declared in `pyproject.toml`.

## 4. Create your local config

```bash
cp .env.sample ede.conf
```

Open `ede.conf` and skim the defaults. The shipping defaults work out of the box for local dev:

| Key | Default | What it does |
|---|---|---|
| `DEFAULT_PERSISTENCE_PROVIDER` | `sqlalchemy` | Use the SQLAlchemy persistence adapter (the other option is `none`). |
| `DATABASE_ENGINE` | `sqlite` | Per-tenant SQLite files; switch to `postgresql` for production. |
| `DEFAULT_MESSAGE_BROKER_PROVIDER` | `inmemory` | In-process event queue; switch to `kafka` for production. |
| `DEFAULT_TENANT_ID` | `system` | The bootstrap tenant. |
| `JWT_SECRET_KEY` | `change-me-in-production` | **Must override in production.** |

!!! warning "JWT_SECRET_KEY"
    Override `JWT_SECRET_KEY` before deploying. The default `change-me-in-production` is fine for local dev only.

## 5. Apply migrations (initial DB seed)

```bash
ede migrate upgrade -t system
```

This runs Alembic upgrades for every active app against the `system` tenant. On first run it creates the SQLite file under your data directory and seeds:

-   Foundation `res.*` masters (country, organization, user, currency, …).
-   Foundation `ir.*` registry rows (menus, actions, view metadata).
-   Auth scaffolding (sessions table, default admin user).

Re-running this command is safe — Alembic skips revisions that are already applied.

## 6. Start the server

```bash
ede serve --with-worker
```

What this does:

-   Starts the **FastAPI HTTP server** on `http://localhost:8000`.
-   Spawns an **in-process event worker** (the `--with-worker` flag is a dev convenience; in production you run `ede worker` as a separate process).
-   Mounts the React web client assets at `/wc/` and redirects `/` to the login page.

You should see startup logs like:

```
[INFO]  ede.core.bootstrap  : registered 28 models, 14 controllers
[INFO]  ede.core.adapters.http.fastapi : mounted 47 routes
[INFO]  uvicorn.error : Uvicorn running on http://0.0.0.0:8000
```

## 7. Verify in the browser

Open **<http://localhost:8000>**. You will be redirected to the login screen. Log in with the seeded admin (check `ede.conf` for `ADMIN_EMAIL` / `ADMIN_PASSWORD`, or use the default `admin@example.com` / `admin` if those keys are unset).

After login you land on the web client home — app switcher in the top bar, menus on the left, list/form views in the main area.

## 8. Inspect what is loaded (sanity check)

In a new terminal:

```bash
ede info
```

This prints the active foundation apps, active domains, and the count of registered models/commands/events. If a model you expect is missing, this is the first place to look.

## 9. Run the tests

```bash
./run_tests.sh
```

This is the single entrypoint for the pytest suite. It boots a clean Env, runs every test under `src/tests/`, and tears down. Use `./run_tests.sh --cov` for an HTML coverage report.

!!! note
    Never cherry-pick individual pytest cases for verification. The repo convention is `./run_tests.sh` as the single entry point — it captures fixture initialization, registry resets, and cleanup that ad-hoc pytest invocations miss.

## 10. (Optional) Start the frontend dev server

You **don't need this** for normal work — `ede serve` already serves a built React bundle. Use the dev server only when you are actively editing files under `src/frontend/`:

```bash
cd src/frontend
bun install
bun run dev          # http://localhost:5173 with hot reload, proxies API to :8000
```

Edits to `.tsx` / `.ts` / `.css` files hot-reload. When done, build a fresh bundle:

```bash
bun run build
```

The bundle is written to `src/ede/foundation/presentation/assets/web-client/` and picked up by `ede serve` on next restart.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `ede: command not found` | Virtualenv isn't activated, or `pip install -e .` wasn't run. Activate `.venv`, re-run install. |
| `sqlalchemy.exc.OperationalError: no such table` | Migrations weren't applied. Run `ede migrate upgrade -t system`. |
| `ModuleNotFoundError: No module named 'ede.core...'` after edits | You installed without `-e`. Reinstall with `pip install -e ".[dev]"`. |
| Port 8000 already in use | Pass `--port 8001` to `ede serve`, or kill the other process. |
| Login fails | Check `ede.conf` for `ADMIN_EMAIL` / `ADMIN_PASSWORD`. If unset, the seeded credentials are in the auth migration — grep `src/ede/foundation/auth/migrations/` for the bootstrap email. |
| Frontend shows a blank page | The built bundle is stale. Run `cd src/frontend && bun run build` and reload. |

## What you can do next

You have a working dev environment. Time to build something.

[:octicons-arrow-right-24: **Next — Your First Module**](01-your-first-module.md)
