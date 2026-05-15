<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Background Jobs Engine — Implementation Docs

**Module:** `foundation.jobs` (`src/ede/foundation/jobs/`)
**Roadmap:** [roadmap/foundation/jobs/](../roadmap/foundation/jobs/README.md)
**Status:** 🔴 Not Started (roadmap drafted 2026-05-11)
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **domain-agnostic background work engine**. Two entry points: (1) declarative scheduled jobs via `@api.scheduled_job("0 */6 * * *", name="...")` decorating any Python callable, and (2) programmatic ad-hoc submission via `env.enqueue_job(target, payload, run_at=None, priority=5)`. Both paths flow through the same `ir.job.run` lifecycle with retry/backoff/dead-letter, progress reporting, concurrency control, and a Settings → Technical → Jobs admin UI.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Today EDE has `EventQueue` for short-lived async fan-out (`@api.on_event` handlers) and the `ede worker` CLI to drain it. That works for "send a notification when an approval is decided" but does NOT cover: scheduled cron-style work, long-running tasks needing progress reporting, retry/backoff/dead-letter orchestration for non-event tasks, or operational visibility into "what background jobs ran today and which failed". As a result, multiple modules have rolled their own inline workers — `GatewaySaasWorker` provisions tenants, `SlaWorker` escalates approval tasks, notifications dispatches webhooks. OneMaster Phase 1 would have been the fourth (provider sync orchestrator + snapshot builder + webhook dispatcher). `foundation.jobs` ends that pattern: every module declares jobs through one decorator API and consumes one admin UI.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX** — Settings → Technical → Jobs reveals four screens: Dashboard (queue depth, due-soon, failure rate, dead-letter count), Definitions (`ir.job` list + form with a "Run Now" button), Run History (`ir.job.run` list with filter chips), Dead Letter Queue (per-run retry button + bulk "retry all failed for job X since date Y").
- **Programmatic entry points for other modules**:
    - `@api.scheduled_job(name=..., cron=..., retry_policy=..., retry_max_attempts=..., priority=..., timeout_seconds=...)` — declare a cron-driven job
    - `@api.background_job(name=...)` — declare a callable expected to be invoked ad-hoc via enqueue
    - `env.enqueue_job(target="dotted.path", payload={...}, run_at=None, priority=5, retry_policy=None)` — submit work
    - `env.job_progress(percent=37.5, message="upserted 12000/32000")` — progress reporting inside a job target (no-op outside)
    - Lifecycle events: `@api.on_event("ir.job.run.completed")`, `"ir.job.run.failed"`, `"ir.job.run.dead_letter"`, `"ir.job.run.started"` — for cross-module reactivity
- **CLI** — `ede jobs list`, `ede jobs runs --since=1h`, `ede jobs run <name>`, `ede jobs disable/enable <name>`, `ede jobs retry <run-id>`.
- **Integration boundary** — the engine PRODUCES the `ir.job.run.*` events + writes status into its own tables. It CONSUMES `notification.send` (loose coupling, via command bus) for stuck-job + dead-letter alerts.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Module]                          [foundation.jobs]
─────────                         ─────────────────
@api.scheduled_job("…")    ──►   ir.job  (definition row, boot-reconciled)
                                       │
                                       ▼ JobsScheduler thread inside `ede worker`
                                  detects next_run_at_utc ≤ now
                                       │
                                       ▼
env.enqueue_job(target, …)  ──►  ir.job.queue  (FIFO + priority)
                                       │
                                       ▼ JobRunner thread-pool picks
                                       ▼
                                  ir.job.run  (PENDING → RUNNING → SUCCESS|FAILED|DEAD_LETTER|INTERRUPTED|TIMED_OUT)
                                       │
                                       ├─►  env.job_progress(pct, msg)  writes back into ir.job.run.progress_pct
                                       │
                                       ├─►  retry policy (none/fixed/exponential/linear) reschedules on failure
                                       │    until retry_max_attempts; then DEAD_LETTER + notification.send
                                       │
                                       └─►  emit ir.job.run.{started, completed, failed, dead_letter}

Concurrency control: ir.job.lock row per active claim, Postgres SELECT FOR UPDATE SKIP LOCKED.
Graceful shutdown: JOBS_GRACEFUL_TIMEOUT_SECONDS=30 to drain in-flight runs.
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| _none yet — all 🔴 Not Started_ | Phase 1 will introduce: `ir.job` (definitions), `ir.job.run` (per-execution log), `ir.job.queue` (ad-hoc FIFO+priority), `ir.job.lock` (concurrency control). Phase 2 adds `ir.job.requires` (dependency graph). | (planned) `src/ede/foundation/jobs/models/...` |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| _none yet_ | Phase 1 planned: `JobsScheduler` (cron-driven dispatcher, `croniter`-backed), `JobRunner` (thread-pool executor with graceful shutdown), `JobRegistry` (in-memory map of decorated callables, boot-reconciled with `ir.job` rows), `RetryPolicy` (four policies: none/fixed/exponential/linear with ±20% jitter), `ProgressReporter` (thread-local-backed `env.job_progress` plumbing). | (planned) |
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
- External Python dep added to `pyproject.toml`: `croniter>=2.0`
- The `ede worker` process must be running — the scheduler thread + runner pool live inside it. Production deployments typically run 2+ worker processes for redundancy (Phase 2 adds multi-worker safety).
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none yet — Phase 1 will introduce_ | | | | Planned: `JOBS_ENABLED` (bool, true), `JOBS_SCHEDULER_TICK_SECONDS` (int, 10), `JOBS_RUNNER_POOL_SIZE` (int, 4), `JOBS_GRACEFUL_TIMEOUT_SECONDS` (int, 30), `JOBS_DEFAULT_RETRY_POLICY` (enum, "exponential"), `JOBS_DEFAULT_TIMEOUT_SECONDS` (int, 600). |
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
| _none yet_ | Phase 1 will seed: `data/jobs_rbac.csv` (three roles — admin / operator / viewer), `data/jobs_menus.xml` (Settings → Technical → Jobs menu tree), `data/notification_templates.xml` (stuck-job + dead-letter templates). No seed `ir.job` rows — each consuming module registers its own jobs via `@api.scheduled_job`. |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Core Engine + First Adopter | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/jobs/phase-1-implementation.md) |
| Phase 2 | Advanced (Multi-worker + Observability) | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/jobs/phase-2-implementation.md) |
| Phase 3 | Adoption Refactor (Retire Ad-Hoc Workers) | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/jobs/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Entire engine is not yet built — all 3 phases are 🔴 Not Started | 🔴 Not Started | [roadmap/foundation/jobs/README.md](../roadmap/foundation/jobs/README.md) |
| **HARD BLOCKS:** `onemaster` Phase 1 cannot start until `foundation.jobs` Phase 1 ships | 🔴 Not Started | [roadmap/onemaster/phase-1-implementation.md §Hard Prerequisites](../roadmap/onemaster/phase-1-implementation.md#hard-prerequisites) |
| Inline workers in `foundation.gateway` (`GatewaySaasWorker`), `foundation.approval` (`SlaWorker`), and `foundation.notifications` (webhook dispatch) all remain ad-hoc until Phase 3 of this module | 🟠 High gap | [phase-3-implementation.md](../roadmap/foundation/jobs/phase-3-implementation.md) |
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
- Phase 1: introduces 4 new tables (`ir_job`, `ir_job_run`, `ir_job_queue`, `ir_job_lock`) + `croniter` dependency.
- Phase 2: extends `ir_job` with `requires_ids` (Many2Many self-link for dependency graph) + `tenant_concurrency_limit` integer. Adds `/metrics` endpoint.
- Phase 3: NO schema changes — pure adoption pass. `GatewaySaasWorker`, `SlaWorker`, and notifications-webhook-dispatch are deleted from their source files; their behaviour moves into `@api.scheduled_job` / `@api.background_job` decorators on the existing target functions.
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
- [Foundation Gateway](../roadmap/foundation/gateway/) — Phase 3 of jobs retires `GatewaySaasWorker`
- [Foundation Approval](./foundation-approval.md) — Phase 3 of jobs retires `SlaWorker`
- [Foundation Notifications](./foundation-notifications.md) — Phase 3 of jobs retires webhook dispatch (when notifications Phase 2 lands)
- [Cross-module roadmap tracker](../roadmap/roadmap-tracker.md)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-11. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
