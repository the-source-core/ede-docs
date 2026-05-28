<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Background Jobs Engine — Implementation Docs

**Module:** `foundation.jobs` (`src/ede/foundation/jobs/`)
**Roadmap:** [roadmap/foundation/jobs/](../roadmap/foundation/jobs/README.md)
**Status:** ✅ Phases 1+2 Delivered 2026-05-19. **Phase 1** (Slices 1+2+3+4) — schema + Celery executor + jobs-worker CLI + cron scheduler + decorators + XML data path + boot reconciler + retry policy + dead-letter + `env.job_progress` + Settings → Technical → Jobs admin UI + RBAC seed + heartbeat first-adopter. **Phase 2** (Slices 1+2+3) — lock contention proof + Clock injection + event constants + auto-pool + `ir.job.requires` dependency graph + per-tenant concurrency cap + Prometheus metrics endpoint + stuck-job reaper + dead-letter recovery UI. **Phase 3 ✅ Delivered 2026-05-28** (Adoption Refactor — Approval + Notifications) — `approval.SlaWorker` retired onto the engine as the `approval.sla_tick` `@api.scheduled_job` (the old `@api.on_event("ede.worker.tick")` handler was dead code: the tick event was never emitted, so this both relocated the runtime AND activated SLA checking for the first time); WS-J21 (notifications webhook dispatch) deferred — notifications has no webhook surface yet. Phase 4 (Gateway SaaS Worker Retirement — retire `gateway.GatewaySaasWorker`, split from Phase 3 because provisioning is destructive and gated on the Postgres multi-worker contention proof) remains 🔴 Not Started.
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **domain-agnostic background work engine**. Two submission entry points: (1) declarative scheduled jobs via `@api.scheduled_job("0 */6 * * *", name="...")` decorating any Python callable, and (2) programmatic ad-hoc submission via `env.enqueue_job(target, payload, run_at=None, priority=5)`. Both paths flow through the same `ir.job.run` lifecycle with retry/backoff/dead-letter, progress reporting, concurrency control, and a Settings → Technical → Jobs admin UI.

`ir.job` definitions can be created three ways — by the `@api.scheduled_job` decorator (`source=decorator`), by `<record model="ir.job">` in a module's `data/*.xml` (`source=xml`), or by `env.models["ir.job"].create({...})` from the admin UI / runtime (`source=runtime`). The boot reconciler only manages `source=decorator` rows; XML and runtime rows are inviolate from its perspective.

Task execution is delegated to **Celery (Redis broker)** under a stable `Executor` interface. The user-facing surface (`ir.job` schema, decorator API, admin UI, RBAC, CLI) is ours; the worker pool, ack/nack semantics, and broker reliability are Celery's. The seam is one interface (`services/executor.py`) — future phases may swap in a thread-pool or alternate engine without touching the surface.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Today EDE has `EventQueue` for short-lived async fan-out (`@api.on_event` handlers) and the `ede worker` CLI to drain it. That works for "send a notification when an approval is decided" but does NOT cover: scheduled cron-style work, long-running tasks needing progress reporting, retry/backoff/dead-letter orchestration for non-event tasks, or operational visibility into "what background jobs ran today and which failed". As a result, multiple modules have rolled their own inline workers — `GatewaySaasWorker` provisions tenants, `SlaWorker` escalates approval tasks, notifications dispatches webhooks. OneMaster Phase 1 would have been the fourth (provider sync orchestrator + snapshot builder + webhook dispatcher). `foundation.jobs` ends that pattern: every module declares jobs through one decorator API and consumes one admin UI.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX** — Settings → Technical → Jobs reveals four screens: Dashboard (queue depth, due-soon, failure rate, dead-letter count), Definitions (`ir.job` list + form with a "Run Now" button and a `source` badge), Run History (`ir.job.run` list with filter chips, links to `celery_task_id`), Dead Letter Queue (per-run retry button + bulk "retry all failed for job X since date Y").
- **Programmatic entry points for other modules**:
    - `@api.scheduled_job(name=..., cron=..., retry_policy=..., retry_max_attempts=..., priority=..., timeout_seconds=...)` — declare a cron-driven job (code-defined, fixed schedule)
    - `@api.background_job(name=...)` — declare a callable expected to be invoked ad-hoc via enqueue
    - `<record id="..." model="ir.job"><field name="name">.../><field name="target">.../><field name="cron">.../>...</record>` in a module's `data/*.xml` — config-defined, multi-instance, or ops-managed jobs
    - `env.enqueue_job(target="dotted.path", payload={...}, run_at=None, priority=5, retry_policy=None)` — submit ad-hoc work
    - `env.job_progress(percent=37.5, message="upserted 12000/32000")` — progress reporting inside a job target (no-op outside)
    - Lifecycle events: `@api.on_event("ir.job.run.completed")`, `"ir.job.run.failed"`, `"ir.job.run.dead_letter"`, `"ir.job.run.started"` — for cross-module reactivity
- **CLI** — `ede jobs list`, `ede jobs runs --since=1h`, `ede jobs run <name>`, `ede jobs disable/enable <name>`, `ede jobs retry <run-id>`. Plus operational entry points: `ede worker` (scheduler + supervisor + event drainer) and `ede jobs-worker` (Celery prefork pool, horizontally scalable). Dev one-liner: `ede serve --with-worker --with-jobs-worker`.
- **Integration boundary** — the engine PRODUCES the `ir.job.run.*` events + writes status into its own tables. It CONSUMES `notification.send` (loose coupling, via command bus) for stuck-job + dead-letter alerts. It also CONSUMES a Redis broker (mandatory runtime dep) as the Celery transport.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Module]                          [ede worker]                         [Redis broker]    [ede jobs-worker]
─────────                         ──────────────                        ──────────────    ─────────────────
@api.scheduled_job("…")    ──►   ir.job  (definition row)
   ↘ data/*.xml (records)            │
   ↘ env.models["ir.job"].create     ▼ JobsScheduler thread, every 10s
                                  SELECT due ir.job rows
                                  acquire ir.job.lock                                            (Celery prefork pool)
                                  INSERT ir.job.run (status=pending)  ──►  send_task(execute_run, run_id)
                                                                                          │
                                                                                          ▼ Celery prefork child picks task
                                                                                       execute_run wrapper:
                                                                                         1. bootstrap Env (tenant from ir.job.run.tenant_id)
                                                                                         2. mark run RUNNING, record celery_task_id
                                                                                         3. set thread-local for env.job_progress
                                                                                         4. resolve target callable
env.enqueue_job(target, …)  ──►  INSERT ir.job.run (status=pending)  ──►  send_task ──►  5. call target(env, payload)
                                                                                         6. capture output / exception
                                                                                         7. apply OUR retry policy (re-enqueue with eta=)
                                                                                         8. on exhaustion → DEAD_LETTER + notification.send
                                                                                         9. write outcome row, release ir.job.lock

Process topology:
  ede worker              — scheduler + supervisor + event drainer (one per node)
  ede jobs-worker         — Celery prefork pool (N replicas, horizontally scalable)
  Redis (DB 2)            — broker; per-priority queues ede.jobs.p0 .. p9
  Postgres                — schema + ir.job.lock (SELECT FOR UPDATE SKIP LOCKED scheduler dedup)

Failure isolation:
  Each task runs in its own Celery prefork child — a segfault or OOM in a target
  cannot kill siblings, the scheduler, or the event drainer.
  Worker thread death is caught by the ede worker supervisor → exit 2 → orchestrator restart.
  Celery child crash → broker redelivers after visibility_timeout; startup reconciler reaps
  stuck `running` rows whose celery_task_id is no longer known to the broker.
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none yet — all 🔴 Not Started_ | Phase 1 will introduce: `ir.job` (definitions, with `source` enum: decorator \| xml \| runtime), `ir.job.run` (per-execution log, with `celery_task_id` for broker correlation), `ir.job.lock` (scheduler-side dedup via `SELECT FOR UPDATE SKIP LOCKED`). The originally-planned `ir.job.queue` is **dropped** — Celery's Redis broker is the queue. Phase 2 adds `ir.job.requires` (dependency graph). | (planned) `src/ede/foundation/jobs/models/...` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ | Phase 1 planned: `JobsScheduler` (cron-driven dispatcher, `croniter`-backed, dispatches via `executor.submit`), `CeleryExecutor` implementing `Executor` protocol (the swap seam — Celery lives only here), `celery_app` (Celery app construction, signal wiring for per-process SQLAlchemy pool, 10 priority queues), `execute_run` task wrapper (Env bootstrap, target call, outcome write, retry, lock release), `JobRegistry` (in-memory map of decorated callables), `BootReconciler` (creates/updates/deactivates `source=decorator` rows; never touches `source=xml` or `source=runtime`), `RetryPolicy` (four policies: none/fixed/exponential/linear with ±20% jitter), `ProgressReporter` (thread-local-backed `env.job_progress` plumbing), `StartupReconciler` (reaps `running` rows with unknown `celery_task_id` after a crash). | (planned) `src/ede/foundation/jobs/services/...` |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none yet_ | Phase 1 plans: `ir.job.run_now` (admin button or `ede jobs run <name>`), `ir.job.disable` / `ir.job.enable`, `ir.job.run.retry` (force-retry a dead-letter run). Generic `ede.{create,update}` covers everything else. | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none yet_ | Phase 1 will emit: `ir.job.run.started` (runner picks up), `ir.job.run.completed` (success), `ir.job.run.failed` (will retry), `ir.job.run.dead_letter` (retries exhausted; triggers notification dispatch). Subscribers in Phase 2+: cross-module observability, alerting rules, audit pipelines. | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| _none yet_ | Phase 1 will introduce `/api/foundation/jobs/*`: `GET /jobs` (list definitions), `GET /jobs/{id}` (detail), `POST /jobs/{id}/run-now`, `POST /jobs/{id}/disable` + `/enable`, `GET /runs` (paged history), `GET /runs/{id}` (detail with payload/output/traceback/progress), `POST /runs/{id}/retry`. Phase 2 adds `/api/foundation/jobs/metrics` (Prometheus). | (planned) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none yet_ | The engine itself emits events rather than relying on hooks. Phase 1's boot reconciler runs after `loader.load_app()` for each app — it walks the `JOB_REGISTRY` module-scoped lists and reconciles with `ir.job` rows (INSERT for new decorators, deactivate for removed). |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
`ir.job.run.status`:

`PENDING → RUNNING → (SUCCESS | FAILED | TIMED_OUT | INTERRUPTED | DEAD_LETTER)`

- `PENDING` — created by scheduler or enqueue, not yet picked by a runner
- `RUNNING` — runner has acquired lock and is executing the target callable
- `SUCCESS` — callable returned normally
- `FAILED` — callable raised an exception; if retry policy allows, a new `ir.job.run` row is created with `attempt_number = parent + 1` (the parent stays `FAILED`)
- `TIMED_OUT` — exceeded `ir.job.timeout_seconds`; runner killed the worker thread
- `INTERRUPTED` — worker process exited mid-run (SIGTERM after graceful timeout); re-eligible per `retry_on_interrupt` flag
- `DEAD_LETTER` — exhausted `retry_max_attempts`; manual retry only via UI / `ede jobs retry`
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `jobs`
- Manifest `depends`: `["base", "presentation"]` (notifications is loose-coupled via command bus, not module-load)
- External Python deps added to `pyproject.toml`: `celery>=5.4,<6`, `croniter>=2.0`, `redis>=5.0,<6`
- Runtime infra requirement: a reachable **Redis server** (mandatory — Celery broker). EDE uses DB 2 for jobs (gateway uses DB 0).
- Two worker processes must be running in production: `ede worker` (scheduler + supervisor + event drainer — one per node) and `ede jobs-worker` (Celery prefork pool — N replicas, scaled horizontally). Dev convenience: `ede serve --with-worker --with-jobs-worker` spins up everything in one command.
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `JOBS_ENABLED` | bool | `True` | `JOBS_ENABLED` | Master switch — `False` disables scheduler + executor wiring |
| `JOBS_SCHEDULER_TICK_SECONDS` | int | `10` | `JOBS_SCHEDULER_TICK_SECONDS` | How often the scheduler polls `ir.job` for due rows |
| `JOBS_GRACEFUL_TIMEOUT_SECONDS` | int | `30` | `JOBS_GRACEFUL_TIMEOUT_SECONDS` | SIGTERM grace window before in-flight runs marked `interrupted` |
| `JOBS_DEFAULT_RETRY_POLICY` | enum | `"exponential"` | `JOBS_DEFAULT_RETRY_POLICY` | Default `ir.job.retry_policy` |
| `JOBS_DEFAULT_TIMEOUT_SECONDS` | int | `600` | `JOBS_DEFAULT_TIMEOUT_SECONDS` | Default `ir.job.timeout_seconds` |
| `JOBS_CELERY_BROKER_URL` | str | `"redis://localhost:6379/2"` | `JOBS_CELERY_BROKER_URL` | Redis broker URL |
| `JOBS_CELERY_DEFAULT_QUEUE` | str | `"ede.jobs.default"` | `JOBS_CELERY_DEFAULT_QUEUE` | Fallback queue name |
| `JOBS_CELERY_PREFORK_CONCURRENCY` | int | `4` | `JOBS_CELERY_PREFORK_CONCURRENCY` | Prefork children per `ede jobs-worker` process |
| `JOBS_CELERY_TASK_SERIALIZER` | str | `"json"` | `JOBS_CELERY_TASK_SERIALIZER` | Always `json` — pickle is forbidden |
| `JOBS_CELERY_VISIBILITY_TIMEOUT_SECONDS` | int | `3600` | `JOBS_CELERY_VISIBILITY_TIMEOUT_SECONDS` | Redis visibility timeout — must exceed max `ir.job.timeout_seconds` |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| _none yet_ | | | | Engine-wide tuning lives in `FoundationSettings`. Per-job tuning lives on `ir.job` rows themselves (cron, retry policy, max attempts, base seconds, priority, timeout, active flag). |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| _none yet_ | | Phase 1 ships Settings → Technical → Jobs (dashboard + definitions + run history + dead-letter), not a `<settings>` config panel — the engine has no module-level toggles beyond the foundation settings above. |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `data/jobs_rbac.csv` (planned) | Three roles — `jobs.admin` / `jobs.operator` / `jobs.viewer` |
| `data/jobs_menus.xml` (planned) | Settings → Technical → Jobs menu tree (Dashboard / Definitions / Run History / Dead Letter) |
| `data/notification_templates.xml` (planned) | `jobs.stuck_job_alert` + `jobs.dead_letter_alert` templates |
| `data/example_jobs.xml` (planned) | Heartbeat `ir.job` row (`source=xml`, `cron="*/2 * * * *"`) — proves the XML data path and acts as Phase 1's first-adopter |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Core Engine + First Adopter | ✅ Delivered 2026-05-19 | [phase-1-implementation.md](../roadmap/foundation/jobs/phase-1-implementation.md) |
| Phase 2 | Advanced (Multi-worker + Observability) | ✅ Delivered 2026-05-19 (Slices 1+2+3) | [phase-2-implementation.md](../roadmap/foundation/jobs/phase-2-implementation.md) |
| Phase 3 | Adoption Refactor (Approval + Notifications) | ✅ Delivered 2026-05-28 (WS-J20 + J22 + J23; WS-J21 deferred) | [phase-3-implementation.md](../roadmap/foundation/jobs/phase-3-implementation.md) |
| Phase 4 | Gateway SaaS Worker Retirement | 🔴 Not Started | [phase-4-implementation.md](../roadmap/foundation/jobs/phase-4-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| **Slice 1 — Schema + Celery executor + jobs-worker CLI** (✅ 2026-05-18) | `ir.job` (with `source` enum: decorator \| xml \| runtime) · `ir.job.run` (with `celery_task_id`) · `ir.job.lock` (unique on `lock_key`) | `src/ede/foundation/jobs/{models/job.py, models/job_run.py, models/job_lock.py, services/celery_app.py, services/executor.py, services/task_wrapper.py, services/lock.py, migrations/versions/3438ba0d57d1_jobs_init.py}` · `src/ede/cli/commands/jobs_worker.py` · `src/ede/core/env.py` (`enqueue_job` method) · `src/ede/foundation/settings.py` (10 `JOBS_*` settings) | [phase-1-implementation.md](../roadmap/foundation/jobs/phase-1-implementation.md) WS-J1 + parts of WS-J3 + parts of WS-J5 |
| **Slice 2 — Cron scheduler + decorators + XML data path + source-aware boot reconciler** (✅ 2026-05-19) | uses existing `ir.job` schema; no new models | `src/ede/foundation/jobs/services/{cron.py, scheduler.py, job_registry.py, reconciler.py}` · `src/ede/core/api.py` (`@api.scheduled_job` / `@api.background_job` re-exports) · `src/ede/core/boot.py` (reconciler hook) · `src/ede/cli/commands/worker.py` (`--no-jobs` flag + scheduler thread) · `src/ede/foundation/jobs/data/test_jobs.xml` (test fixture) | [phase-1-implementation.md](../roadmap/foundation/jobs/phase-1-implementation.md) WS-J2 + WS-J4 + parts of WS-J5 |
| **Slice 3 — Retry policy + dead-letter + env.job_progress** (✅ 2026-05-19) | uses existing `ir.job` / `ir.job.run` schema; retry chain via `parent_run_id`; status enum already had `dead_letter` | `src/ede/foundation/jobs/services/{retry_policy.py, progress.py}` · `src/ede/foundation/jobs/services/task_wrapper.py` (retry-vs-dead-letter branching + `_schedule_retry_attempt` + `_dispatch_dead_letter_notification` + progress set/clear in finally) · `src/ede/core/env.py` (`env.job_progress(*, percent, message)` method) | [phase-1-implementation.md](../roadmap/foundation/jobs/phase-1-implementation.md) WS-J6 + WS-J7 |
| **Slice 4 — Admin UI + RBAC + heartbeat first-adopter (Phase 1 ✅)** (✅ 2026-05-19) | uses existing schema; no new models | `src/ede/foundation/jobs/demo/heartbeat.py` (target callable) · `src/ede/foundation/jobs/data/example_jobs.xml` (XML-declared `ir.job` row, source=xml, cron="*/2 * * * *") · `src/ede/foundation/jobs/data/ir.rbac.permission.csv` (8 permission rows) · `src/ede/foundation/jobs/data/jobs_menus.xml` (3 `ir.action` + 4 `ir.menu` records, Settings → Technical → Jobs tree) · `src/ede/foundation/jobs/views/{job_views.xml, job_run_views.xml, job_lock_views.xml}` (list + form + search views) · `src/ede/foundation/jobs/api/jobs_routes.py` (`JobsController` at `/api/foundation/jobs/*` — run_now, disable, enable, retry_run) | [phase-1-implementation.md](../roadmap/foundation/jobs/phase-1-implementation.md) WS-J8 + WS-J9 + WS-J10 |
| **Phase 2 Slice 1 — Lock contention proof + Clock injection + event constants** (✅ 2026-05-19) | uses existing `ir.job.lock` unique constraint; no schema changes | `src/ede/foundation/jobs/services/{clock.py (Clock Protocol + SystemClock + FakeClock), events.py (4 event-name constants + payload contract)}` · `src/ede/foundation/jobs/services/scheduler.py` (Clock kwarg) · `src/ede/foundation/jobs/services/task_wrapper.py` (3 event-name string literals swapped for constants) | [phase-2-implementation.md](../roadmap/foundation/jobs/phase-2-implementation.md) WS-J11 + WS-J18 + WS-J19 |
| **Phase 2 Slice 2 — Auto-pool + dependency graph + tenant concurrency** (✅ 2026-05-19) | `ir.job.requires_ids` (self-M2M; kernel extended to support self-references — `column1`/`column2` derived from field-name slug) + `ir.job.tenant_concurrency_limit` (Integer, default 0=unlimited) | `src/ede/foundation/jobs/services/pool.py` (`compute_prefork_concurrency(settings)` — `multiprocessing.cpu_count()` clamped to [1,32]) · `src/ede/foundation/settings.py` (`JOBS_RUNNER_POOL_AUTOSIZE: bool = False`) · `src/ede/cli/commands/jobs_worker.py` (auto-pool wire-up in concurrency fallback) · `src/ede/foundation/jobs/models/job.py` (2 new fields) · `src/ede/foundation/jobs/migrations/versions/2c432dc92acb_phase2_slice2_dependency_graph_and_.py` · `src/ede/foundation/jobs/services/scheduler.py` (dep-graph gate + tenant-concurrency gate in `tick()` between lock-acquire and run-create, both with `release_lock` on skip) · kernel: `src/ede/core/kernel/schema.py` + `src/ede/core/orm/relational.py` (self-M2M support) | [phase-2-implementation.md](../roadmap/foundation/jobs/phase-2-implementation.md) WS-J12 + WS-J13 + WS-J14 |
| **Phase 2 Slice 3 — Prometheus metrics + stuck-job reaper + dead-letter recovery UI (Phase 2 ✅)** (✅ 2026-05-19) | uses existing `ir.job` / `ir.job.run` schema; no new models | `src/ede/foundation/jobs/services/{metrics.py (Prometheus exposition — counters + 3 gauges off ir.job.run state), reaper.py (reap_stuck_runs detects runs past 2×timeout_seconds, marks interrupted)}` · `src/ede/foundation/settings.py` (`JOBS_REAPER_TICK_SECONDS=60`, `JOBS_REAPER_TIMEOUT_MULTIPLIER=2`) · `src/ede/foundation/jobs/api/jobs_routes.py` (3 new endpoints: `GET /metrics` Prometheus text/plain via `request_type="http"` envelope + `POST /runs/{run_id}/retry-dead-letter` single + `POST /runs/retry-dead-letter-bulk` filtered by job_id+since) · `src/ede/cli/commands/worker.py` (reaper thread alongside scheduler, daemon=True, ticks every JOBS_REAPER_TICK_SECONDS) · `pyproject.toml` (`prometheus-client>=0.20,<1`) | [phase-2-implementation.md](../roadmap/foundation/jobs/phase-2-implementation.md) WS-J15 + WS-J16 + WS-J17 |
| **Phase 3 — Approval SLA escalation adopted onto the engine (WS-J20)** (✅ 2026-05-28) | uses existing `ir.approval.task` / `ir.job` schema; the `approval.sla_tick` `ir.job` row is created by the boot reconciler (source=decorator) | `src/ede/foundation/approval/services/sla_worker.py` (`@api.scheduled_job(name="approval.sla_tick", cron="* * * * *", retry_policy="fixed")` on `sla_tick(env, payload)`; `_check_sla_breaches` kept internal, returns overdue count; the dead `@api.on_event("ede.worker.tick")` handler removed) · `src/ede/foundation/approval/__manifest__.py` (`depends` += `foundation.jobs`) · `src/ede/foundation/settings.py` (`ACTIVE_MODULES` reordered — `jobs` before `approval`) · `src/tests/foundation/approval/test_sla_escalation.py` (registration + behaviour tests) | [phase-3-implementation.md](../roadmap/foundation/jobs/phase-3-implementation.md) WS-J20 + WS-J22 + WS-J23 |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| `foundation.notifications` webhook dispatch not yet on the engine — deferred (WS-J21) until notifications Phase 2 ships its dispatch redesign, then migrates to `@api.background_job` | 🟢 Low backlog | [phase-3-implementation.md](../roadmap/foundation/jobs/phase-3-implementation.md) |
| Inline worker in `foundation.gateway` (`GatewaySaasWorker`) remains ad-hoc until Phase 4 (destructive provisioning — gated on the Postgres multi-worker contention proof) | 🟠 High gap | [phase-4-implementation.md](../roadmap/foundation/jobs/phase-4-implementation.md) |
| Postgres multi-worker contention test (exact-once under 2+ `ede worker` processes) still owed from the Phase 2 deliverable checklist — HARD GATE for Phase 4 | 🟠 High gap | [phase-2-implementation.md](../roadmap/foundation/jobs/phase-2-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- Writing a new ad-hoc threading loop instead of `@api.scheduled_job`. **Don't.** The whole point of this engine is that you stop doing that. If you have a use case the decorator API doesn't cover, raise an issue rather than rolling your own — the gap is likely a Phase 2 feature.
- Calling `env.job_progress()` outside a job target. It's safe (no-op) but a misleading pattern — readers will assume the function is inside a job context. Don't use it for general logging.
- Returning non-JSON-serialisable values from a job target. The runner serialises `output` into the `ir.job.run.output` JSONB column — a SQLAlchemy model instance or a numpy array breaks the write. Convert to dict/list/primitive first.
- Setting `timeout_seconds` to a low value on a job that does heavy work. The runner kills the worker thread on timeout — partially-completed work may leave the system in an inconsistent state if the target isn't idempotent. Either make the target idempotent OR set `retry_on_interrupt=False` to avoid auto-rerun.
- Two jobs with the same `name`. Boot reconciler will UPSERT by name — last-declared wins, the prior decorator's settings are clobbered. Treat `name` as a globally unique key.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1: introduces 3 new tables (`ir_job` with `source` enum, `ir_job_run` with `celery_task_id` column, `ir_job_lock`) + new deps `celery>=5.4`, `croniter>=2.0`, `redis>=5.0` + Redis as a mandatory runtime dep + new `ede jobs-worker` process to deploy alongside `ede worker`.
- Phase 2: extends `ir_job` with `requires_ids` (Many2Many self-link for dependency graph) + `tenant_concurrency_limit` integer. Adds `/metrics` endpoint.
- Phase 3 ✅: NO schema changes — pure adoption pass. The approval SLA `@api.on_event("ede.worker.tick")` handler (dead — tick never emitted) became the `approval.sla_tick` `@api.scheduled_job` in the same `sla_worker.py` file (`_check_sla_breaches` kept as internal logic). `foundation.jobs` added to `approval` `depends`; `ACTIVE_MODULES` reordered (`jobs` before `approval`). Notifications-webhook-dispatch (WS-J21) deferred — no webhook surface yet.
- Phase 4: NO schema changes — `GatewaySaasWorker` is deleted from its source file; its behaviour moves into an `@api.scheduled_job` decorator. Gated on the Postgres multi-worker contention proof (destructive provisioning).
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| _none yet_ | Phase 1 plans three roles: `jobs.admin` (full — view/edit/run-now/disable/retry/dead-letter recovery), `jobs.operator` (view + manual trigger + retry only), `jobs.viewer` (read-only dashboard + run history). |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [OneMaster](../src/domains/onemaster/docs/onemaster.md) — HARD prereq consumer for Phase 1 (provider sync orchestrator, snapshot builder, webhook dispatcher all consume the engine)
- [Foundation Gateway](../roadmap/foundation/gateway/) — Phase 4 of jobs retires `GatewaySaasWorker`
- [Foundation Approval](./foundation-approval.md) — Phase 3 of jobs retired `SlaWorker` → `approval.sla_tick` `@api.scheduled_job` ✅
- [Foundation Notifications](./foundation-notifications.md) — Phase 3 of jobs retires webhook dispatch (when notifications Phase 2 lands)
- [Cross-module roadmap tracker](../roadmap/roadmap-tracker.md)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-28 (Phase 3 ✅ Delivered — approval SLA escalation adopted onto foundation.jobs as the `approval.sla_tick` scheduled job; WS-J21 notifications webhook deferred). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
