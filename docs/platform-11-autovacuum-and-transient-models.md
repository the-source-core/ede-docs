<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# `@api.autovacuum` + Transient Models — Implementation Docs

**Module:** `ede.core` (kernel) + `foundation.base` (sweep target) + `foundation.jobs` (daily schedule)
**Roadmap:** [roadmap/platform/11-autovacuum-and-transient-models.md](../roadmap/platform/11-autovacuum-and-transient-models.md)
**Status:** ✅ Delivered 2026-06-06
**Layer:** Platform-wide (kernel primitive + jobs schedule)

> Source of truth is the roadmap. This doc reflects the current built state. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A reusable daily-cleanup primitive. `@api.autovacuum` marks any `DomainModel` method as a cleanup callback; `__ede_transient__ = True` declares a model whose short-lived rows are auto-deleted once older than a TTL. One daily "sunrise" job runs every registered cleanup.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
EDE had no generic "clean this up every morning" hook. Every model accumulating short-lived rows (wizard drafts, expired sessions, consumed tokens, stale logs) would otherwise hand-roll its own cron + delete query — the N-reinventions smell `foundation.jobs` was built to end, one layer up. The trigger was the admin password-reset wizard ([foundation.base Enhancement 10](../roadmap/foundation/base/enhancements/10-admin-password-reset.md)), which needs transient rows that must not linger.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- Decorate a model method `@api.autovacuum` to register a cleanup callback (it runs whatever cleanup the model owns through `self.env`; takes only `self`).
- Or set `__ede_transient__ = True` (+ optional `__ede_transient_ttl_minutes__`, default `1440` = one day) for an automatic TTL vacuum — no method needed.
- The `ir.autovacuum.gc` job (cron `0 6 * * *`) runs all collected callbacks daily; one failing callback is logged and skipped so it cannot abort the rest.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
- **Kernel** (`src/ede/core/api.py`, `registry.py`): `@api.autovacuum` sets `__ede_autovacuum__` on the method. `register_model` collects every such method — plus a TTL vacuum for each `__ede_transient__` model — into `registry._autovacuum`, each wrapped into an `(env) -> None` callable (env injected like a hook). `registry.get_autovacuum_callables()` returns them.
- **Sweep target** (`src/ede/foundation/base/services/autovacuum_gc.py`): a **plain function** `autovacuum_gc(env, payload=None)` that invokes every collected callable inside its own try/except.
- **Schedule** (`src/ede/foundation/jobs/data/autovacuum_sweep_job.xml`): a config-defined `ir.job` (`source=xml`) whose `target` is the dotted path to the sweep function. It is intentionally **NOT** `@api.scheduled_job`: `foundation.base` is imported while `ede.core.api` is still mid-load (api re-exports `scheduled_job` from `foundation.jobs`, whose import pulls in `base.models`), so the decorator is unavailable at base import time. The XML `ir.job` sidesteps that — the scheduler resolves the target by dotted path on demand.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none_ — kernel primitive (no models of its own); the `ir.job` row uses the existing `ir.job` model | | |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `api.autovacuum` | Decorator tagging a method `__ede_autovacuum__` | `src/ede/core/api.py` |
| `Registry._autovacuum` + `get_autovacuum_callables()` | Collect + expose every cleanup callable | `src/ede/core/registry.py` |
| `_make_model_autovacuum_wrapper` / `_make_transient_vacuum_wrapper` | Wrap a method / a transient TTL delete into `(env)->None` | `src/ede/core/registry.py` |
| `autovacuum_gc(env, payload)` | The sweep target — runs all callables | `src/ede/foundation/base/services/autovacuum_gc.py` |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none_ — `@api.autovacuum` is a cleanup decorator, not a `pre`/`post` lifecycle hook | |
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- No new `ACTIVE_MODULES` / `ACTIVE_DOMAINS` entry. The kernel decorator is always available; the daily sweep runs when `foundation.jobs` is active (it owns the `ir.job` row).
- Manifest `depends`: unchanged.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
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
| `src/ede/foundation/jobs/data/autovacuum_sweep_job.xml` | the `ir.autovacuum.gc` config-defined `ir.job` (cron `0 6 * * *`, source=xml) |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| PC-11 | `@api.autovacuum` + Transient Models | ✅ Delivered 2026-06-06 | [spec](../roadmap/platform/11-autovacuum-and-transient-models.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| `@api.autovacuum` decorator + registry collection | — | `core/api.py`, `core/registry.py` | [PC-11](../roadmap/platform/11-autovacuum-and-transient-models.md) |
| `__ede_transient__` model-kind + TTL vacuum | — | `core/registry.py` | [PC-11](../roadmap/platform/11-autovacuum-and-transient-models.md) |
| `ir.autovacuum.gc` daily sweep (target + ir.job) | uses `ir.job` | `foundation/base/services/autovacuum_gc.py`, `foundation/jobs/data/autovacuum_sweep_job.xml` | [PC-11](../roadmap/platform/11-autovacuum-and-transient-models.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- **Do not use `@api.scheduled_job` for a base-level sweep.** `foundation.base` imports during `ede.core.api`'s own load, so the re-exported decorator isn't ready — use a config-defined `ir.job` (source=xml) + a plain target function instead.
- **The transient vacuum compares against `created_at_utc` as a SQLAlchemy DateTime column.** Pass a naive datetime object as the cutoff, not an isoformat string — the driver binds it in the column's stored format; a hand-built `'T'`-separated string mis-sorts against the `' '`-separated stored values.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- No schema of its own. The first consumer (Enhancement 10) ships `res_user_password_reset` (Alembic `cf1ec4d1d4aa`).
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [foundation.base — Admin Password Reset (Enhancement 10)](../roadmap/foundation/base/enhancements/10-admin-password-reset.md) — first consumer.
- [foundation.jobs](foundation-jobs.md) — owns the `ir.job` schedule.
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-06-06. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
