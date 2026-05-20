# Foundation.jobs Phase 1 — Slice 1: Schema + Celery Executor Tracer Bullet

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land the minimum viable Celery-executor tracer bullet for `foundation.jobs` — the three core models, the Celery app + executor + task wrapper, the `ede jobs-worker` CLI entry, and a programmatic enqueue path. Slice ends when a pytest test can create an `ir.job.run` row, dispatch it through `CeleryExecutor` in eager mode, and observe `status=success` with output captured.

**Architecture:** Three new `ir.*` framework-metadata models (`ir.job`, `ir.job.run`, `ir.job.lock`) own the schema. A Celery app (`celery_app.py`) configured with JSON serializer + Redis broker hosts a single `execute_run` task. The task wrapper resolves a dotted-path target, bootstraps `Env`, invokes the target, writes the outcome. A thin `CeleryExecutor` implementing `Executor` protocol is the swap seam. `env.enqueue_job(...)` becomes the user-facing programmatic entry. No scheduler, no decorator, no admin UI in this slice — those land in Slices 2–4.

**Tech Stack:** Python 3.10+, SQLAlchemy, Alembic, **Celery 5.4+** (`celery>=5.4,<6`), **croniter 2.0+** (croniter is a Slice 2 dep but we add it now to avoid a second pyproject change), **redis-py 5.0+** (`redis>=5.0,<6`) + Redis server, pytest. CLI extends existing Click groups.

**Does NOT cover (split into next plans):**
- WS-J2 Scheduler thread + `JobsScheduler` (Slice 2)
- WS-J4 `@api.scheduled_job` / `@api.background_job` decorators + XML data path + boot reconciler (Slice 2)
- WS-J6 Retry policy with `send_task(eta=...)` re-enqueue + dead-letter notification dispatch (Slice 3)
- WS-J7 `env.job_progress()` thread-local plumbing (Slice 3)
- WS-J8 Settings → Technical → Jobs admin UI (Slice 4)
- WS-J9 RBAC seed + full ≥45-test suite — Slice 1 ships a single end-to-end smoke + per-model tests; the full suite lands across Slices 2-4
- WS-J10 XML-declared heartbeat first adopter + user walkthrough (Slice 4)

---

## File Structure

### New files (created in this slice)

| Path | Responsibility |
|---|---|
| `src/ede/foundation/jobs/__init__.py` | Empty package marker |
| `src/ede/foundation/jobs/__manifest__.py` | Module manifest (depends: base + presentation) |
| `src/ede/foundation/jobs/models/__init__.py` | Re-export model classes so loader picks them up |
| `src/ede/foundation/jobs/models/job.py` | `ir.job` model — definitions (with `source` enum) |
| `src/ede/foundation/jobs/models/job_run.py` | `ir.job.run` model — execution log (with `celery_task_id`) |
| `src/ede/foundation/jobs/models/job_lock.py` | `ir.job.lock` model — scheduler-side dedup |
| `src/ede/foundation/jobs/services/__init__.py` | Empty package marker |
| `src/ede/foundation/jobs/services/celery_app.py` | Celery app construction + JSON serializer + worker signal wiring |
| `src/ede/foundation/jobs/services/executor.py` | `Executor` protocol + `CeleryExecutor` implementation (the swap seam) |
| `src/ede/foundation/jobs/services/task_wrapper.py` | `execute_run` Celery task — resolves target, bootstraps Env, writes outcome |
| `src/ede/foundation/jobs/services/lock.py` | Acquire/release `ir.job.lock` rows (Postgres + SQLite fallback) |
| `src/ede/foundation/jobs/migrations/__init__.py` | Empty package marker |
| `src/ede/foundation/jobs/migrations/versions/<rev>_jobs_init.py` | Alembic migration for the three tables |
| `src/ede/cli/commands/jobs_worker.py` | `ede jobs-worker` Click command — thin wrapper around `celery worker` |
| `src/tests/foundation/jobs/__init__.py` | Test package marker |
| `src/tests/foundation/jobs/conftest.py` | Pytest fixtures — Celery in eager mode, Env helpers |
| `src/tests/foundation/jobs/test_models.py` | Per-model CRUD + uniqueness + enum validation |
| `src/tests/foundation/jobs/test_executor_eager.py` | End-to-end smoke: enqueue → run → status=success |

### Existing files modified

| Path | Change |
|---|---|
| `pyproject.toml` | Add `celery>=5.4,<6`, `croniter>=2.0`, `redis>=5.0,<6` to dependencies |
| `src/ede/foundation/settings.py` | Add `"jobs"` to `ACTIVE_MODULES`; add 10 new `JOBS_*` fields |
| `src/ede/core/env.py` | Add `env.enqueue_job(target, payload=None, run_at=None, priority=5, tenant_id=None)` method |
| `src/ede/cli/__init__.py` (or main CLI registrar) | Register `jobs_worker_command` |

---

## Pre-flight checks

- [ ] **Step P1: Confirm Redis is reachable on `redis://localhost:6379` for live tests** (not strictly needed for Slice 1 since we run Celery eager-mode in tests, but verify so live smoke at end of slice works):
    ```bash
    redis-cli -n 2 ping
    ```
    Expected: `PONG`. If absent on the dev box: `sudo apt install redis-server && sudo systemctl start redis-server`.

- [ ] **Step P2: Confirm the working tree is clean for the new module (no stray `src/ede/foundation/jobs/` from any earlier exploration):**
    ```bash
    test ! -d src/ede/foundation/jobs && echo "OK — green-field" || ls src/ede/foundation/jobs
    ```

---

## Task 1: Module scaffold + manifest + activation

**Files:**
- Create: `src/ede/foundation/jobs/__init__.py`
- Create: `src/ede/foundation/jobs/__manifest__.py`
- Create: `src/ede/foundation/jobs/models/__init__.py`
- Create: `src/ede/foundation/jobs/services/__init__.py`
- Create: `src/ede/foundation/jobs/migrations/__init__.py`
- Create: `src/ede/foundation/jobs/migrations/versions/__init__.py`
- Modify: `src/ede/foundation/settings.py:43` — append `"jobs"` to `ACTIVE_MODULES`

- [ ] **Step 1.1: Create the package directory tree**
    ```bash
    mkdir -p src/ede/foundation/jobs/{models,services,migrations/versions}
    touch src/ede/foundation/jobs/__init__.py \
          src/ede/foundation/jobs/models/__init__.py \
          src/ede/foundation/jobs/services/__init__.py \
          src/ede/foundation/jobs/migrations/__init__.py \
          src/ede/foundation/jobs/migrations/versions/__init__.py
    ```

- [ ] **Step 1.2: Write the manifest**
    Write `src/ede/foundation/jobs/__manifest__.py`:
    ```python
    {
        "name": "Background Jobs",
        "summary": "Cron + ad-hoc background work engine — schema, scheduler, Celery executor.",
        "description": """
    Foundation Background Jobs engine. Phase 1 delivers:
      - ir.job          Job definitions (decorator + XML + runtime sources)
      - ir.job.run      Per-execution log with progress, output, error capture
      - ir.job.lock     Scheduler-side dedup (SELECT FOR UPDATE SKIP LOCKED on Postgres)
      - JobsScheduler   Polls due ir.job rows, dispatches via Executor.submit
      - CeleryExecutor  Implements Executor protocol; Celery prefork pool with Redis broker
      - @api.scheduled_job / @api.background_job decorators + XML data path + reconciler
      - env.enqueue_job(target, payload) for ad-hoc submission
      - env.job_progress(pct, msg) for live progress reporting
      - ede jobs-worker CLI entry point + ede jobs ... operator sub-group
      - Settings → Technical → Jobs admin UI

    Slice 1 ships the schema + Celery executor + worker entry only.
    """,
        "author": "Platform",
        "category": "Foundation",
        "version": "0.1.0",
        "depends": ["base", "presentation"],
        "data": [],
    }
    ```

- [ ] **Step 1.3: Append `"jobs"` to `ACTIVE_MODULES` in `src/ede/foundation/settings.py:43`**
    Change the list from `[... "ai", "assistant"]` to `[... "ai", "assistant", "jobs"]`.
    Run the boot smoke once to confirm the loader picks it up cleanly:
    ```bash
    ede info --config ede.conf
    ```
    Expected: `jobs` appears in the loaded apps list. **No models registered yet** — the next tasks add them. Expected: no AttributeError, no ImportError.

---

## Task 2: External dependencies — celery, croniter, redis

**Files:**
- Modify: `pyproject.toml` — add three new entries to `dependencies` or the equivalent `[project.dependencies]` block

- [ ] **Step 2.1: Locate the existing dependencies list**
    ```bash
    grep -nE "^\s*\"(celery|croniter|redis|sqlalchemy|fastapi)" pyproject.toml | head -10
    ```

- [ ] **Step 2.2: Add three new deps** to the appropriate `dependencies` list. Pin majors:
    ```
    "celery>=5.4,<6",
    "croniter>=2.0",
    "redis>=5.0,<6",
    ```

- [ ] **Step 2.3: Install + confirm**
    ```bash
    pip install -e ".[dev]"
    python -c "import celery, croniter, redis; print(celery.__version__, croniter.__version__, redis.__version__)"
    ```
    Expected: three version strings, no ImportError.

---

## Task 3: `ir.job` model

**Files:**
- Create: `src/ede/foundation/jobs/models/job.py`
- Modify: `src/ede/foundation/jobs/models/__init__.py` — `from .job import Job`
- Test: `src/tests/foundation/jobs/test_models.py` (new file — partial; extended in Tasks 4-5)

- [ ] **Step 3.1: Write the failing test**
    Create `src/tests/foundation/jobs/__init__.py` (empty) and `src/tests/foundation/jobs/conftest.py`:
    ```python
    # src/tests/foundation/jobs/conftest.py
    import pytest

    pytest_plugins = ["src.tests.conftest"]
    ```
    Create `src/tests/foundation/jobs/test_models.py`:
    ```python
    # src/tests/foundation/jobs/test_models.py
    def test_ir_job_model_registers_and_creates(env_with_jobs):
        """ir.job should be in the registry and accept create with required fields."""
        env = env_with_jobs
        job_proxy = env.models["ir.job"]
        rec = job_proxy.create({
            "name": "test.heartbeat",
            "module_key": "foundation.jobs",
            "target": "ede.foundation.jobs.demo.heartbeat.tick",
            "kind": "scheduled",
            "cron": "*/2 * * * *",
            "source": "xml",
        })
        assert rec.name == "test.heartbeat"
        assert rec.active is True               # default
        assert rec.priority == 5                # default
        assert rec.retry_max_attempts == 3      # default
        assert rec.source == "xml"
    ```
    Add the `env_with_jobs` fixture to `src/tests/foundation/jobs/conftest.py`:
    ```python
    @pytest.fixture
    def env_with_jobs(in_memory_env):
        """Reuse the standard in-memory env fixture; ensures 'jobs' is in ACTIVE_MODULES."""
        return in_memory_env
    ```
    (Adjust the fixture name to match whatever the repo's standard env fixture is — confirm by `grep -rn "in_memory_env\|env_fixture\|env_proxy" src/tests/conftest.py | head -5`.)

- [ ] **Step 3.2: Run the failing test**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_models.py::test_ir_job_model_registers_and_creates
    ```
    Expected: FAIL with `KeyError: 'ir.job'` (model not yet registered).

- [ ] **Step 3.3: Write `src/ede/foundation/jobs/models/job.py`**
    ```python
    # -*- coding: utf-8 -*-
    """ir.job — Background job definition row."""
    from __future__ import annotations

    from ede.core import api
    from ede.core.kernel import fields
    from ede.core.kernel.model import DomainModel


    @api.model(
        "ir.job",
        description="Background job definition (cron / queued / manual)",
        record_name="name",
        default_order="priority asc, name asc",
        name_search_fields=["name", "module_key"],
    )
    class Job(DomainModel):
        """A registered background job — three sources may populate it.

        source=decorator → managed by the boot reconciler.
        source=xml       → loaded from data/*.xml by the standard EDE data loader.
        source=runtime   → admin-UI / programmatic creation; never touched by reconciler.
        """

        name = fields.Char(max_length=128, required=True, index=True, label="Name")
        module_key = fields.Char(max_length=64, required=True, index=True, label="Module")
        target = fields.Char(max_length=255, required=True, label="Target (dotted path)")

        kind = fields.Enum(
            selection=[
                ("scheduled", "Scheduled (cron)"),
                ("queued", "Queued (enqueue-only)"),
                ("manual", "Manual (button-only)"),
            ],
            default="queued",
            required=True,
            label="Kind",
        )
        cron = fields.Char(max_length=64, label="Cron expression")

        next_run_at_utc = fields.DateTime(label="Next Run (UTC)", index=True)
        last_run_at_utc = fields.DateTime(label="Last Run (UTC)")
        last_status = fields.Enum(
            selection=[
                ("success", "Success"),
                ("failed", "Failed"),
                ("dead_letter", "Dead Letter"),
                ("interrupted", "Interrupted"),
            ],
            label="Last Status",
        )

        retry_policy = fields.Enum(
            selection=[
                ("none", "None"),
                ("fixed", "Fixed delay"),
                ("exponential", "Exponential backoff"),
                ("linear", "Linear backoff"),
            ],
            default="exponential",
            required=True,
            label="Retry Policy",
        )
        retry_max_attempts = fields.Integer(default=3, required=True, label="Max Retry Attempts")
        retry_base_seconds = fields.Integer(default=60, required=True, label="Retry Base Seconds")

        priority = fields.Integer(default=5, required=True, label="Priority (0=highest, 9=lowest)")
        timeout_seconds = fields.Integer(default=600, required=True, label="Timeout Seconds")
        retry_on_interrupt = fields.Boolean(default=True, required=True, label="Retry on Interrupt")

        tenant_id = fields.Char(max_length=64, label="Tenant ID (null = system)", index=True)
        active = fields.Boolean(default=True, required=True, label="Active")

        source = fields.Enum(
            selection=[
                ("decorator", "Decorator"),
                ("xml", "XML data"),
                ("runtime", "Runtime"),
            ],
            default="runtime",
            required=True,
            label="Source",
            help="Where this ir.job row originated. The boot reconciler only manages source=decorator rows.",
        )

        description = fields.Char(multi_line=True, label="Description")
    ```

- [ ] **Step 3.4: Add to package re-exports**
    Write `src/ede/foundation/jobs/models/__init__.py`:
    ```python
    from .job import Job

    __all__ = ["Job"]
    ```

- [ ] **Step 3.5: Run the test — expect it to pass once a migration is in place**
    Since SQLAlchemy autogen needs a migration row to actually create the table, expect the test to **still fail** at this step with a "no such table: ir_job" error if your `env_with_jobs` fixture spins up a SQLite from scratch. That's acceptable here — Task 6 ships the migration. If your test fixture uses the registry metadata to auto-create tables (check via `grep -n "create_all" src/tests/conftest.py`), the test should pass.

    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_models.py::test_ir_job_model_registers_and_creates -v
    ```

    Expected outcomes:
    - If fixture uses `metadata.create_all()` → PASS now.
    - If fixture relies on alembic migrations → FAIL; will pass after Task 6.

    Either is fine; **document which case applies in a one-line code comment in `conftest.py`**.

---

## Task 4: `ir.job.run` model

**Files:**
- Create: `src/ede/foundation/jobs/models/job_run.py`
- Modify: `src/ede/foundation/jobs/models/__init__.py` — add `JobRun`
- Test: extend `src/tests/foundation/jobs/test_models.py`

- [ ] **Step 4.1: Write the failing test**
    Append to `src/tests/foundation/jobs/test_models.py`:
    ```python
    def test_ir_job_run_model_creates_with_defaults(env_with_jobs):
        env = env_with_jobs
        job = env.models["ir.job"].create({
            "name": "test.heartbeat-2",
            "module_key": "foundation.jobs",
            "target": "ede.foundation.jobs.demo.heartbeat.tick",
            "kind": "scheduled",
            "source": "xml",
        })
        run = env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "pending",
            "payload": {"x": 1},
        })
        assert run.status == "pending"
        assert run.payload == {"x": 1}
        assert run.celery_task_id is None
        assert run.attempt_number == 1
        assert run.parent_run_id is None
    ```

- [ ] **Step 4.2: Run — expect failure** (`KeyError: 'ir.job.run'`):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_models.py::test_ir_job_run_model_creates_with_defaults
    ```

- [ ] **Step 4.3: Write `src/ede/foundation/jobs/models/job_run.py`**
    ```python
    # -*- coding: utf-8 -*-
    """ir.job.run — Append-only execution log."""
    from __future__ import annotations

    from ede.core import api
    from ede.core.kernel import fields
    from ede.core.kernel.model import DomainModel


    @api.model(
        "ir.job.run",
        description="Per-execution log of an ir.job",
        record_name="status",
        default_order="created_at_utc desc",
    )
    class JobRun(DomainModel):
        """One row per execution attempt of a job.

        Retry chain reconstructable via parent_run_id. Terminal statuses (success,
        dead_letter, interrupted, timed_out) never transition further; failed may
        re-enter via a new child row with attempt_number = parent + 1.
        """

        job_id = fields.Reference("ir.job", on_delete="cascade", required=True, index=True, label="Job")
        attempt_number = fields.Integer(default=1, required=True, label="Attempt Number")
        parent_run_id = fields.Reference("ir.job.run", on_delete="set null", label="Parent Run")

        queued_at_utc = fields.DateTime(label="Queued (UTC)")
        started_at_utc = fields.DateTime(label="Started (UTC)")
        finished_at_utc = fields.DateTime(label="Finished (UTC)")

        status = fields.Enum(
            selection=[
                ("pending", "Pending"),
                ("running", "Running"),
                ("success", "Success"),
                ("failed", "Failed"),
                ("dead_letter", "Dead Letter"),
                ("interrupted", "Interrupted"),
                ("timed_out", "Timed Out"),
            ],
            default="pending",
            required=True,
            index=True,
            label="Status",
        )

        payload = fields.JSON(label="Payload")
        output = fields.JSON(label="Output")
        error_summary = fields.Char(multi_line=True, label="Error Summary")
        error_traceback = fields.Char(multi_line=True, label="Error Traceback")

        progress_pct = fields.Decimal(precision=5, scale=2, label="Progress %")
        progress_message = fields.Char(max_length=255, label="Progress Message")

        worker_id = fields.Char(max_length=64, label="Worker ID")
        celery_task_id = fields.Char(max_length=64, index=True, label="Celery Task ID")
        duration_seconds = fields.Integer(label="Duration (s)")
    ```

- [ ] **Step 4.4: Update model package re-exports**
    Edit `src/ede/foundation/jobs/models/__init__.py`:
    ```python
    from .job import Job
    from .job_run import JobRun

    __all__ = ["Job", "JobRun"]
    ```

- [ ] **Step 4.5: Run test — expect pass** (subject to same fixture caveat as Step 3.5):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_models.py::test_ir_job_run_model_creates_with_defaults -v
    ```

---

## Task 5: `ir.job.lock` model

**Files:**
- Create: `src/ede/foundation/jobs/models/job_lock.py`
- Modify: `src/ede/foundation/jobs/models/__init__.py` — add `JobLock`
- Test: extend `src/tests/foundation/jobs/test_models.py`

- [ ] **Step 5.1: Write the failing test**
    Append to `src/tests/foundation/jobs/test_models.py`:
    ```python
    def test_ir_job_lock_unique_on_lock_key(env_with_jobs):
        env = env_with_jobs
        env.models["ir.job.lock"].create({
            "lock_key": "job:test-foo",
            "worker_id": "hostA:123",
        })
        with pytest.raises(Exception):                # IntegrityError or framework variant
            env.models["ir.job.lock"].create({
                "lock_key": "job:test-foo",          # duplicate
                "worker_id": "hostB:456",
            })
    ```
    Add `import pytest` at the top of the test file if absent.

- [ ] **Step 5.2: Run — expect failure** (`KeyError: 'ir.job.lock'`):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_models.py::test_ir_job_lock_unique_on_lock_key
    ```

- [ ] **Step 5.3: Write `src/ede/foundation/jobs/models/job_lock.py`**
    ```python
    # -*- coding: utf-8 -*-
    """ir.job.lock — Scheduler-side dedup row."""
    from __future__ import annotations

    from ede.core import api
    from ede.core.kernel import fields
    from ede.core.kernel.model import DomainModel


    @api.model(
        "ir.job.lock",
        description="Scheduler-side concurrency control (one row per active claim)",
        record_name="lock_key",
        default_order="acquired_at_utc desc",
    )
    class JobLock(DomainModel):
        """A claimed lock for a given lock_key.

        The scheduler acquires before send_task. On Postgres uses
        SELECT FOR UPDATE SKIP LOCKED. SQLite falls back to in-process
        threading.Lock — single-worker dev only.
        """

        lock_key = fields.Char(max_length=128, required=True, index=True, label="Lock Key")
        worker_id = fields.Char(max_length=64, required=True, label="Worker ID")
        acquired_at_utc = fields.DateTime(required=True, label="Acquired (UTC)")
        expires_at_utc = fields.DateTime(required=True, label="Expires (UTC)")
    ```

- [ ] **Step 5.4: Update model package re-exports**
    Final `src/ede/foundation/jobs/models/__init__.py`:
    ```python
    from .job import Job
    from .job_run import JobRun
    from .job_lock import JobLock

    __all__ = ["Job", "JobRun", "JobLock"]
    ```

- [ ] **Step 5.5: Run all three model tests — expect PASS**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_models.py -v
    ```

---

## Task 6: Alembic migration for the three tables

**Files:**
- Create: `src/ede/foundation/jobs/migrations/versions/<rev>_jobs_init.py` (autogen produces filename)

> **Important: Alembic migration discipline.** This repo uses per-app version locations. Before generating, invoke the `writing-alembic-migrations` skill (per CLAUDE.md "skill-list" frontmatter) for the constraints — particularly the 63-character Postgres identifier limit on FK/index names and the no-ALTER-constraint-on-SQLite rule. If the autogenerated names exceed 63 chars, hand-name them with the pattern `ix_<table>_<col>` / `fk_<table>_<col>` / `uq_<table>_<col>`.

- [ ] **Step 6.1: Run autogen for the jobs app**
    ```bash
    ede migrate generate -m "jobs_init" --app jobs --config ede.conf
    ```
    Expected: a new file `src/ede/foundation/jobs/migrations/versions/<rev>_jobs_init.py`. If `resolving-alembic-multi-heads` errors trigger ("Can't locate revision identified by"), follow that skill's resolution.

- [ ] **Step 6.2: Inspect the generated migration**
    Open the file. Confirm three tables (`ir_job`, `ir_job_run`, `ir_job_lock`) with columns matching the model fields + auto-fields (`dbid`, `record_uuid`, `created_at_utc`, `updated_at_utc`, `revision`, `created_uid`, `updated_uid`). Confirm:
    - `ir_job.name` has a unique index (the framework auto-attaches via `index=True` — check the autogen output)
    - `ir_job_lock.lock_key` has a unique index
    - All FK columns target `record_uuid` of the parent table, not `dbid`
    - All identifier names ≤ 63 characters

- [ ] **Step 6.3: Hand-name constraints if any exceed 63 chars**
    If autogen produced e.g. `fk_ir_job_run_parent_run_id_ir_job_run_record_uuid` (> 63), rename to `fk_ir_job_run_parent_run_id`. Use `op.create_foreign_key("fk_ir_job_run_parent_run_id", "ir_job_run", "ir_job_run", ["parent_run_id"], ["record_uuid"])` pattern.

- [ ] **Step 6.4: Apply migration to a scratch SQLite tenant**
    ```bash
    ede migrate upgrade -t scratch_jobs --config ede.conf
    ```
    Expected: completes without error. Three new tables in the tenant DB.

- [ ] **Step 6.5: Run the model tests against a real migrated DB**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_models.py -v
    ```
    Expected: all PASS.

---

## Task 7: Add `JOBS_*` settings to FoundationSettings

**Files:**
- Modify: `src/ede/foundation/settings.py` — add 10 new fields after the existing `JWT_*` block

- [ ] **Step 7.1: Write the failing test**
    Create `src/tests/foundation/jobs/test_settings.py`:
    ```python
    from ede.foundation.settings import FoundationSettings


    def test_jobs_settings_defaults():
        s = FoundationSettings()
        assert s.JOBS_ENABLED is True
        assert s.JOBS_SCHEDULER_TICK_SECONDS == 10
        assert s.JOBS_GRACEFUL_TIMEOUT_SECONDS == 30
        assert s.JOBS_DEFAULT_RETRY_POLICY == "exponential"
        assert s.JOBS_DEFAULT_TIMEOUT_SECONDS == 600
        assert s.JOBS_CELERY_BROKER_URL == "redis://localhost:6379/2"
        assert s.JOBS_CELERY_DEFAULT_QUEUE == "ede.jobs.default"
        assert s.JOBS_CELERY_PREFORK_CONCURRENCY == 4
        assert s.JOBS_CELERY_TASK_SERIALIZER == "json"
        assert s.JOBS_CELERY_VISIBILITY_TIMEOUT_SECONDS == 3600
    ```

- [ ] **Step 7.2: Run — expect failure** (AttributeError):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_settings.py
    ```

- [ ] **Step 7.3: Add settings fields**
    Edit `src/ede/foundation/settings.py`. After the last existing field (likely under the JWT section), add a `# ── Background Jobs Engine ──────────────────────────────────────────────` block:
    ```python
    # ── Background Jobs Engine ──────────────────────────────────────────────
    JOBS_ENABLED: bool = True
    JOBS_SCHEDULER_TICK_SECONDS: int = 10
    JOBS_GRACEFUL_TIMEOUT_SECONDS: int = 30
    JOBS_DEFAULT_RETRY_POLICY: str = "exponential"
    JOBS_DEFAULT_TIMEOUT_SECONDS: int = 600

    JOBS_CELERY_BROKER_URL: str = "redis://localhost:6379/2"
    JOBS_CELERY_DEFAULT_QUEUE: str = "ede.jobs.default"
    JOBS_CELERY_PREFORK_CONCURRENCY: int = 4
    JOBS_CELERY_TASK_SERIALIZER: str = "json"
    JOBS_CELERY_VISIBILITY_TIMEOUT_SECONDS: int = 3600
    ```

- [ ] **Step 7.4: Run — expect PASS**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_settings.py -v
    ```

---

## Task 8: `Executor` protocol

**Files:**
- Create: `src/ede/foundation/jobs/services/executor.py`
- Test: `src/tests/foundation/jobs/test_executor_protocol.py`

- [ ] **Step 8.1: Write the failing test**
    Create `src/tests/foundation/jobs/test_executor_protocol.py`:
    ```python
    from ede.foundation.jobs.services.executor import Executor


    class _Dummy:
        """A class that satisfies the Executor protocol via structural typing."""

        def submit(self, env, run):
            pass

        def submit_retry(self, env, run, eta):
            pass

        def graceful_shutdown(self, timeout_seconds):
            pass


    def test_executor_protocol_is_satisfied_by_dummy():
        ex: Executor = _Dummy()
        assert hasattr(ex, "submit")
        assert hasattr(ex, "submit_retry")
        assert hasattr(ex, "graceful_shutdown")
    ```

- [ ] **Step 8.2: Run — expect failure** (ModuleNotFoundError):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_executor_protocol.py
    ```

- [ ] **Step 8.3: Write `src/ede/foundation/jobs/services/executor.py` — protocol only for now (CeleryExecutor lands in Task 11)**
    ```python
    # -*- coding: utf-8 -*-
    """Executor — the swap seam between the engine and the actual task runtime.

    Slice 1 ships one implementation (CeleryExecutor in task_wrapper.py). Future
    phases may add a ThreadPoolExecutor or RiverExecutor without touching the
    scheduler, decorator API, schema, UI, or CLI.
    """
    from __future__ import annotations

    from datetime import datetime
    from typing import Protocol, runtime_checkable, TYPE_CHECKING

    if TYPE_CHECKING:
        from ede.core.env import Env
        from ede.core.orm.recordset import RecordSet


    @runtime_checkable
    class Executor(Protocol):
        """Protocol every job executor must satisfy."""

        def submit(self, env: "Env", run: "RecordSet") -> None:
            """Hand a pending ir.job.run to the executor.

            Implementation MUST set ir.job.run.celery_task_id (or equivalent
            backend correlation id) on the row before returning, so the admin
            UI and the startup reconciler can correlate.
            """

        def submit_retry(self, env: "Env", run: "RecordSet", eta: datetime) -> None:
            """Schedule a retry attempt for the given run at eta (UTC-aware datetime)."""

        def graceful_shutdown(self, timeout_seconds: int) -> None:
            """Drain in-flight tasks up to timeout_seconds; called on SIGTERM."""
    ```

- [ ] **Step 8.4: Run — expect PASS**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_executor_protocol.py -v
    ```

---

## Task 9: Celery app construction

**Files:**
- Create: `src/ede/foundation/jobs/services/celery_app.py`
- Test: `src/tests/foundation/jobs/test_celery_app.py`

- [ ] **Step 9.1: Write the failing test**
    Create `src/tests/foundation/jobs/test_celery_app.py`:
    ```python
    from ede.foundation.jobs.services.celery_app import build_celery_app
    from ede.foundation.settings import FoundationSettings


    def test_build_celery_app_with_json_serializer_and_no_result_backend():
        settings = FoundationSettings()
        app = build_celery_app(settings)

        assert app.conf.task_serializer == "json"
        assert app.conf.accept_content == ["json"]
        assert app.conf.result_backend is None
        assert app.conf.task_acks_late is True
        assert app.conf.worker_prefetch_multiplier == 1
        assert app.conf.broker_url == settings.JOBS_CELERY_BROKER_URL
        assert app.conf.task_default_queue == settings.JOBS_CELERY_DEFAULT_QUEUE


    def test_build_celery_app_eager_mode_for_tests():
        settings = FoundationSettings()
        app = build_celery_app(settings, eager=True)
        assert app.conf.task_always_eager is True
        assert app.conf.task_eager_propagates is True
    ```

- [ ] **Step 9.2: Run — expect failure** (ModuleNotFoundError):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_celery_app.py
    ```

- [ ] **Step 9.3: Write `src/ede/foundation/jobs/services/celery_app.py`**
    ```python
    # -*- coding: utf-8 -*-
    """Celery app construction for foundation.jobs."""
    from __future__ import annotations

    from typing import TYPE_CHECKING

    from celery import Celery

    if TYPE_CHECKING:
        from ede.foundation.settings import FoundationSettings


    def build_celery_app(settings: "FoundationSettings", *, eager: bool = False) -> Celery:
        """Construct the Celery app from FoundationSettings.

        eager=True is for unit tests — tasks run synchronously in the calling
        thread, no broker required.
        """
        app = Celery("ede.jobs", broker=settings.JOBS_CELERY_BROKER_URL)
        app.conf.update(
            task_serializer=settings.JOBS_CELERY_TASK_SERIALIZER,
            accept_content=[settings.JOBS_CELERY_TASK_SERIALIZER],
            result_backend=None,                          # ir.job.run row is the result
            task_acks_late=True,
            worker_prefetch_multiplier=1,
            task_default_queue=settings.JOBS_CELERY_DEFAULT_QUEUE,
            broker_transport_options={
                "visibility_timeout": settings.JOBS_CELERY_VISIBILITY_TIMEOUT_SECONDS,
            },
        )
        if eager:
            app.conf.update(
                task_always_eager=True,
                task_eager_propagates=True,
            )
        return app


    # Module-level singleton — constructed lazily on first import in the worker
    # process; the eager-mode app is constructed per-test in the conftest fixture.
    _celery_app: Celery | None = None


    def get_celery_app(settings: "FoundationSettings") -> Celery:
        """Return the module-level singleton, building on first call."""
        global _celery_app
        if _celery_app is None:
            _celery_app = build_celery_app(settings)
        return _celery_app
    ```

- [ ] **Step 9.4: Run — expect PASS**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_celery_app.py -v
    ```

---

## Task 10: `execute_run` task wrapper

**Files:**
- Create: `src/ede/foundation/jobs/services/task_wrapper.py`
- Test: deferred to Task 14 (end-to-end smoke covers the wrapper in eager mode)

- [ ] **Step 10.1: Write `src/ede/foundation/jobs/services/task_wrapper.py`**
    ```python
    # -*- coding: utf-8 -*-
    """execute_run — the single Celery task that runs any ir.job target.

    The scheduler / executor never registers per-target Celery tasks. Instead,
    one task (execute_run) dispatches by looking up ir.job.run.job_id.target
    at runtime. This keeps Celery's task registry minimal and the boot path
    cheap.
    """
    from __future__ import annotations

    import importlib
    import os
    import socket
    import traceback
    from datetime import datetime, timezone
    from typing import Any, Callable

    from celery import Celery, Task

    from .lock import release_lock


    def _now_utc() -> datetime:
        return datetime.now(tz=timezone.utc)


    def _resolve_dotted_path(path: str) -> Callable:
        """Resolve 'pkg.module.fn' or 'pkg.module:fn' to a callable."""
        if ":" in path:
            module_path, attr = path.rsplit(":", 1)
        else:
            module_path, attr = path.rsplit(".", 1)
        module = importlib.import_module(module_path)
        return getattr(module, attr)


    def _serialise_output(output: Any) -> Any:
        """Output must already be JSON-serialisable (we use task_serializer=json)."""
        if output is None or isinstance(output, (dict, list, str, int, float, bool)):
            return output
        return {"__repr__": repr(output)[:1000]}


    def register_execute_run_task(celery_app: Celery, bootstrap_env_fn: Callable) -> Task:
        """Bind the execute_run task to the given Celery app.

        bootstrap_env_fn(run_id) → returns a fresh Env scoped to the run's tenant.
        Wiring the bootstrap as a callable parameter (rather than importing it
        at module level) keeps the Celery app free of foundation-boot coupling
        and makes the test fixture trivial.
        """

        @celery_app.task(name="ede.jobs.execute_run", bind=True, ignore_result=True)
        def execute_run(self, run_id: int) -> None:
            env = bootstrap_env_fn(run_id)
            run_proxy = env.models["ir.job.run"]
            run = run_proxy.browse(run_id).ensure_one()
            job = run.job_id

            run.write({
                "status": "running",
                "started_at_utc": _now_utc(),
                "worker_id": f"{socket.gethostname()}:{os.getpid()}",
                "celery_task_id": self.request.id,
            })

            started = _now_utc()
            try:
                target_fn = _resolve_dotted_path(job.target)
                output = target_fn(env, payload=run.payload or {})
                duration = int((_now_utc() - started).total_seconds())
                run.write({
                    "status": "success",
                    "finished_at_utc": _now_utc(),
                    "output": _serialise_output(output),
                    "duration_seconds": duration,
                })
                env.emit("ir.job.run.completed", {"run_id": run_id, "job_name": job.name})
            except Exception as exc:                                # noqa: BLE001 — catch-all
                run.write({
                    "status": "failed",                              # Slice 3 adds retry chain here
                    "finished_at_utc": _now_utc(),
                    "error_summary": f"{type(exc).__name__}: {exc}",
                    "error_traceback": traceback.format_exc(),
                    "duration_seconds": int((_now_utc() - started).total_seconds()),
                })
                env.emit("ir.job.run.failed", {"run_id": run_id, "job_name": job.name})
            finally:
                release_lock(env, lock_key=f"job:{job.name}")

        return execute_run
    ```

- [ ] **Step 10.2: No standalone test in this task** — covered by the end-to-end test in Task 14.

---

## Task 11: Lock acquire / release helpers + `CeleryExecutor`

**Files:**
- Create: `src/ede/foundation/jobs/services/lock.py`
- Append to: `src/ede/foundation/jobs/services/executor.py` — add `CeleryExecutor` class
- Test: `src/tests/foundation/jobs/test_lock.py`

- [ ] **Step 11.1: Write the failing lock test**
    Create `src/tests/foundation/jobs/test_lock.py`:
    ```python
    from datetime import timedelta

    from ede.foundation.jobs.services.lock import acquire_lock, release_lock


    def test_acquire_lock_succeeds_on_fresh_key(env_with_jobs):
        env = env_with_jobs
        ok = acquire_lock(env, lock_key="job:test-a", worker_id="w1", timeout_seconds=60)
        assert ok is True


    def test_acquire_lock_fails_on_taken_key(env_with_jobs):
        env = env_with_jobs
        assert acquire_lock(env, lock_key="job:test-b", worker_id="w1", timeout_seconds=60) is True
        # second attempt without release → returns False
        assert acquire_lock(env, lock_key="job:test-b", worker_id="w2", timeout_seconds=60) is False


    def test_release_lock_allows_reacquire(env_with_jobs):
        env = env_with_jobs
        acquire_lock(env, lock_key="job:test-c", worker_id="w1", timeout_seconds=60)
        release_lock(env, lock_key="job:test-c")
        assert acquire_lock(env, lock_key="job:test-c", worker_id="w2", timeout_seconds=60) is True
    ```

- [ ] **Step 11.2: Run — expect failure** (ModuleNotFoundError):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_lock.py
    ```

- [ ] **Step 11.3: Write `src/ede/foundation/jobs/services/lock.py`**
    ```python
    # -*- coding: utf-8 -*-
    """ir.job.lock acquire / release helpers.

    Slice 1 ships the application-level path (INSERT-or-fail keyed by unique
    index on lock_key). Slice 2 adds the Postgres SELECT FOR UPDATE SKIP LOCKED
    fast path inside the scheduler's tick.
    """
    from __future__ import annotations

    from datetime import datetime, timedelta, timezone


    def _now_utc() -> datetime:
        return datetime.now(tz=timezone.utc)


    def acquire_lock(env, *, lock_key: str, worker_id: str, timeout_seconds: int) -> bool:
        """Acquire a named lock; return True if acquired, False if already held."""
        lock_proxy = env.models["ir.job.lock"]

        # Reap stale locks first (expired)
        now = _now_utc()
        stale = lock_proxy.search([("lock_key", "=", lock_key), ("expires_at_utc", "<=", now)])
        for s in stale:
            s.delete()

        # If a live lock exists, fail
        existing = lock_proxy.search([("lock_key", "=", lock_key)])
        if existing:
            return False

        try:
            lock_proxy.create({
                "lock_key": lock_key,
                "worker_id": worker_id,
                "acquired_at_utc": now,
                "expires_at_utc": now + timedelta(seconds=timeout_seconds + 30),
            })
            return True
        except Exception:                                  # IntegrityError race
            return False


    def release_lock(env, *, lock_key: str) -> None:
        """Release the lock for lock_key. No-op if absent."""
        lock_proxy = env.models["ir.job.lock"]
        rows = lock_proxy.search([("lock_key", "=", lock_key)])
        for r in rows:
            r.delete()
    ```

- [ ] **Step 11.4: Run lock tests — expect PASS**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_lock.py -v
    ```

- [ ] **Step 11.5: Append `CeleryExecutor` to `src/ede/foundation/jobs/services/executor.py`**
    Add to the bottom of that file:
    ```python
    # ── Concrete implementations ─────────────────────────────────────────

    from datetime import datetime as _dt
    from typing import TYPE_CHECKING as _TC

    if _TC:
        from celery import Celery


    class CeleryExecutor:
        """Submits jobs to the foundation.jobs Celery app's execute_run task."""

        def __init__(self, celery_app: "Celery", *, default_queue: str):
            self._app = celery_app
            self._default_queue = default_queue

        def _queue_for(self, priority: int) -> str:
            # priority 0-9 → per-priority queue; fallback to default
            if 0 <= priority <= 9:
                return f"ede.jobs.p{priority}"
            return self._default_queue

        def submit(self, env, run) -> None:
            queue = self._queue_for(run.job_id.priority)
            async_result = self._app.send_task(
                "ede.jobs.execute_run",
                args=[run.dbid],
                queue=queue,
            )
            run.write({"celery_task_id": async_result.id, "queued_at_utc": _dt.utcnow()})

        def submit_retry(self, env, run, eta) -> None:
            queue = self._queue_for(run.job_id.priority)
            async_result = self._app.send_task(
                "ede.jobs.execute_run",
                args=[run.dbid],
                queue=queue,
                eta=eta,
            )
            run.write({"celery_task_id": async_result.id})

        def graceful_shutdown(self, timeout_seconds: int) -> None:
            # Celery worker handles SIGTERM itself via worker_shutdown_timeout.
            # This method is a no-op from the scheduler-process side; left as
            # a hook for the supervisor to call.
            return None
    ```

- [ ] **Step 11.6: Re-run protocol test to confirm CeleryExecutor satisfies the structural type**
    Append to `src/tests/foundation/jobs/test_executor_protocol.py`:
    ```python
    def test_celery_executor_satisfies_protocol():
        from ede.foundation.jobs.services.executor import CeleryExecutor, Executor
        from ede.foundation.jobs.services.celery_app import build_celery_app
        from ede.foundation.settings import FoundationSettings

        app = build_celery_app(FoundationSettings(), eager=True)
        ex: Executor = CeleryExecutor(app, default_queue="ede.jobs.default")
        assert hasattr(ex, "submit")
        assert hasattr(ex, "submit_retry")
        assert hasattr(ex, "graceful_shutdown")
    ```
    Run:
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_executor_protocol.py -v
    ```
    Expected: PASS.

---

## Task 12: `env.enqueue_job(...)` programmatic entry point

**Files:**
- Modify: `src/ede/core/env.py` — add `enqueue_job` method
- Test: `src/tests/foundation/jobs/test_enqueue_job.py`

- [ ] **Step 12.1: Write the failing test**
    Create `src/tests/foundation/jobs/test_enqueue_job.py`:
    ```python
    def test_env_enqueue_job_creates_ir_job_run_and_returns_handle(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor                  # fixture set up in conftest, see Step 12.4

        # Pre-register an ir.job definition (decorator path lands in Slice 2; for
        # Slice 1 we create it directly via the runtime source).
        job = env.models["ir.job"].create({
            "name": "test.echo",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
        })

        run = env.enqueue_job(
            target="src.tests.foundation.jobs.targets.echo_target",
            payload={"msg": "hello"},
            priority=5,
        )
        assert run is not None
        assert run.status == "pending"
        assert run.payload == {"msg": "hello"}
        assert run.job_id.id == job.id or run.job_id.name == job.name
        # In eager mode the celery_task_id will be set after submit; confirm:
        assert run.celery_task_id is not None
    ```

- [ ] **Step 12.2: Create the test target module**
    ```bash
    mkdir -p src/tests/foundation/jobs
    touch src/tests/foundation/jobs/targets.py
    ```
    Write `src/tests/foundation/jobs/targets.py`:
    ```python
    """Test target callables for the executor smoke."""

    def echo_target(env, payload=None):
        return {"echoed": payload}


    def raise_target(env, payload=None):
        raise RuntimeError("intentional failure")
    ```

- [ ] **Step 12.3: Run — expect failure** (AttributeError on `env.enqueue_job`):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_enqueue_job.py
    ```

- [ ] **Step 12.4: Wire `env.enqueue_job()` on the `Env` class**
    Edit `src/ede/core/env.py`. Locate the `Env` class and add at an appropriate place (near other public methods like `dispatch`, `emit`):
    ```python
    def enqueue_job(
        self,
        *,
        target: str,
        payload: dict | None = None,
        run_at=None,                              # datetime, UTC
        priority: int = 5,
        tenant_id: str | None = None,
    ):
        """Submit ad-hoc background work via the registered job executor.

        Creates an ir.job.run row (status=pending) and hands it to the executor.
        Requires the foundation.jobs module to be loaded and an executor to be
        registered on the Env (Slice 1: registered in the env_with_jobs_and_executor
        fixture; full wiring lands in Task 13 / Slice 2's scheduler startup).
        """
        if not hasattr(self, "_jobs_executor") or self._jobs_executor is None:
            raise RuntimeError(
                "env.enqueue_job called but no jobs executor is registered on this Env. "
                "Set env._jobs_executor before calling, or use a fixture that does."
            )

        # Resolve the ir.job definition by target. The decorator path (Slice 2)
        # populates this for code-defined jobs; for ad-hoc / runtime use the
        # caller is expected to have created the ir.job row already.
        job_proxy = self.models["ir.job"]
        matching = job_proxy.search([("target", "=", target)], limit=1)
        if not matching:
            raise RuntimeError(
                f"env.enqueue_job: no ir.job row with target='{target}'. "
                f"Create the ir.job definition first (decorator, XML, or runtime)."
            )
        job = matching[0]

        from datetime import datetime, timezone

        run_proxy = self.models["ir.job.run"]
        run = run_proxy.create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "pending",
            "payload": payload or {},
            "queued_at_utc": datetime.now(tz=timezone.utc),
        })

        self._jobs_executor.submit(self, run)
        return run
    ```

- [ ] **Step 12.5: Add the `env_with_jobs_and_executor` fixture**
    Append to `src/tests/foundation/jobs/conftest.py`:
    ```python
    import pytest

    @pytest.fixture
    def env_with_jobs_and_executor(env_with_jobs):
        """env with a Celery executor in eager mode wired in + execute_run registered."""
        from ede.foundation.jobs.services.celery_app import build_celery_app
        from ede.foundation.jobs.services.executor import CeleryExecutor
        from ede.foundation.jobs.services.task_wrapper import register_execute_run_task
        from ede.foundation.settings import FoundationSettings

        settings = FoundationSettings()
        app = build_celery_app(settings, eager=True)

        # Eager mode → bootstrap_env_fn just returns the existing test env
        register_execute_run_task(app, bootstrap_env_fn=lambda run_id: env_with_jobs)

        executor = CeleryExecutor(app, default_queue=settings.JOBS_CELERY_DEFAULT_QUEUE)
        env_with_jobs._jobs_executor = executor
        return env_with_jobs
    ```

- [ ] **Step 12.6: Run — expect PASS**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_enqueue_job.py -v
    ```

---

## Task 13: `ede jobs-worker` CLI entry point

**Files:**
- Create: `src/ede/cli/commands/jobs_worker.py`
- Modify: the existing CLI registrar (find via `grep -rn "worker_command\|add_command" src/ede/cli/__init__.py src/ede/cli/main.py 2>/dev/null | head -10`)
- Test: `src/tests/foundation/jobs/test_jobs_worker_cli.py`

- [ ] **Step 13.1: Write the failing CLI smoke test**
    Create `src/tests/foundation/jobs/test_jobs_worker_cli.py`:
    ```python
    from click.testing import CliRunner

    from ede.cli.commands.jobs_worker import jobs_worker_command


    def test_jobs_worker_help_runs():
        """Smoke — `ede jobs-worker --help` should not crash."""
        runner = CliRunner()
        result = runner.invoke(jobs_worker_command, ["--help"])
        assert result.exit_code == 0
        assert "jobs-worker" in result.output or "Celery" in result.output
    ```

- [ ] **Step 13.2: Run — expect failure** (ModuleNotFoundError):
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_jobs_worker_cli.py
    ```

- [ ] **Step 13.3: Write `src/ede/cli/commands/jobs_worker.py`**
    ```python
    # -*- coding: utf-8 -*-
    """ede jobs-worker — thin wrapper around `celery worker` with EDE settings preloaded."""
    from __future__ import annotations

    import sys

    import click

    from ede.core.boot import BootInput, boot_runtime_and_environment


    @click.command("jobs-worker")
    @click.option(
        "--foundation-settings",
        default="ede.foundation.settings",
        show_default=True,
        help="Python import path of foundation settings module.",
    )
    @click.option(
        "--foundation-root",
        default="ede.foundation",
        show_default=True,
    )
    @click.option(
        "--config",
        "config_path",
        default=None,
        type=click.Path(exists=True, dir_okay=False),
        help="Path to ede.conf config file.",
    )
    @click.option(
        "--concurrency",
        default=None,
        type=int,
        help="Override JOBS_CELERY_PREFORK_CONCURRENCY.",
    )
    @click.option(
        "--queues",
        "-Q",
        default=None,
        help="Comma-separated queue names; defaults to ede.jobs.p0..p9 + default.",
    )
    def jobs_worker_command(
        foundation_settings: str,
        foundation_root: str,
        config_path: str | None,
        concurrency: int | None,
        queues: str | None,
    ) -> None:
        """Run the EDE jobs Celery worker (prefork pool).

        Boots the EDE foundation registry first (so model + target lookups work
        inside tasks), then hands off to Celery's CLI with our app loaded.
        """
        boot_output = boot_runtime_and_environment(
            inp=BootInput(
                foundation_settings_module_path=foundation_settings,
                foundation_root_package=foundation_root,
                config_path=config_path,
            )
        )
        settings = boot_output.foundation_settings_module

        from ede.foundation.jobs.services.celery_app import build_celery_app
        from ede.foundation.jobs.services.task_wrapper import register_execute_run_task

        app = build_celery_app(settings, eager=False)

        # Bootstrap env per task — for Slice 1 the env_for_run callable is a
        # placeholder that re-uses the boot environment with tenant context
        # applied; Slice 2 swaps in a richer per-tenant builder.
        def _bootstrap_env_for_run(run_id: int):
            # TODO Slice 2: derive tenant from ir.job.run.tenant_id; for Slice 1
            # we return the shared boot environment unscoped.
            return boot_output.environment

        register_execute_run_task(app, bootstrap_env_fn=_bootstrap_env_for_run)

        # Build argv for Celery worker
        concurrency = concurrency or settings.JOBS_CELERY_PREFORK_CONCURRENCY
        if queues is None:
            queues = "ede.jobs.default," + ",".join(f"ede.jobs.p{p}" for p in range(10))
        argv = [
            "worker",
            "--loglevel=INFO",
            f"--concurrency={concurrency}",
            f"--queues={queues}",
            "--pool=prefork",
        ]

        # Hand off to Celery
        app.worker_main(argv=argv)
    ```

- [ ] **Step 13.4: Register the command with the main `ede` CLI**
    Find where `worker_command` and `server_command` are registered:
    ```bash
    grep -rn "worker_command\|server_command\|add_command\|cli\.command" src/ede/cli/__init__.py src/ede/cli/main.py 2>/dev/null | head -20
    ```
    Add the equivalent import + `cli.add_command(jobs_worker_command)` call in the same place. If unsure, run `ede --help` and confirm the new command appears.

- [ ] **Step 13.5: Run CLI smoke test — expect PASS**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/test_jobs_worker_cli.py -v
    ```

- [ ] **Step 13.6: Live smoke — confirm worker boots against real Redis (optional but recommended)**
    In one terminal:
    ```bash
    ede jobs-worker --config ede.conf --concurrency 1
    ```
    Expected: Celery banner appears, "ready" log line, then waits. Ctrl-C to exit.
    If Redis isn't running, error is reported clearly; that's acceptable for Slice 1 — the smoke is "does it boot cleanly with Redis up?"

---

## Task 14: End-to-end smoke test in eager mode

**Files:**
- Create: `src/tests/foundation/jobs/test_executor_eager.py`

- [ ] **Step 14.1: Write the end-to-end test**
    Create `src/tests/foundation/jobs/test_executor_eager.py`:
    ```python
    """End-to-end smoke: enqueue → execute_run task wrapper runs → status=success.

    Uses Celery eager mode so no broker is needed. Exercises the full path:
    env.enqueue_job → CeleryExecutor.submit → execute_run task wrapper →
    target callable → outcome write.
    """


    def test_enqueue_runs_target_and_writes_success(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor

        env.models["ir.job"].create({
            "name": "test.echo-e2e",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
        })

        run = env.enqueue_job(
            target="src.tests.foundation.jobs.targets.echo_target",
            payload={"msg": "ping"},
        )

        # In eager mode, send_task runs the task body synchronously before returning.
        # The run row should now be terminal.
        run_after = env.models["ir.job.run"].browse(run.id).ensure_one()
        assert run_after.status == "success", f"expected success, got {run_after.status} (err={run_after.error_summary})"
        assert run_after.output == {"echoed": {"msg": "ping"}}
        assert run_after.celery_task_id is not None
        assert run_after.worker_id is not None
        assert run_after.duration_seconds is not None and run_after.duration_seconds >= 0
        assert run_after.finished_at_utc is not None


    def test_enqueue_runs_target_and_records_failure(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor

        env.models["ir.job"].create({
            "name": "test.raise-e2e",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.raise_target",
            "kind": "queued",
            "source": "runtime",
        })

        run = env.enqueue_job(
            target="src.tests.foundation.jobs.targets.raise_target",
            payload={},
        )

        run_after = env.models["ir.job.run"].browse(run.id).ensure_one()
        assert run_after.status == "failed", f"expected failed, got {run_after.status}"
        assert "RuntimeError" in (run_after.error_summary or "")
        assert "intentional failure" in (run_after.error_summary or "")
        assert run_after.error_traceback is not None
    ```

- [ ] **Step 14.2: Run the full jobs test suite**
    ```bash
    ./run_tests.sh src/tests/foundation/jobs/ -v
    ```
    Expected:
    - `test_models.py` — 3 PASS
    - `test_settings.py` — 1 PASS
    - `test_executor_protocol.py` — 2 PASS
    - `test_celery_app.py` — 2 PASS
    - `test_lock.py` — 3 PASS
    - `test_enqueue_job.py` — 1 PASS
    - `test_jobs_worker_cli.py` — 1 PASS
    - `test_executor_eager.py` — 2 PASS
    Total: **15 tests, all PASS.**

- [ ] **Step 14.3: Run the full repo test suite — confirm no regressions**
    ```bash
    ./run_tests.sh
    ```
    Expected: prior baseline test count + 15 = new baseline; all green. No regressions in other foundation modules.

---

## Slice 1 Acceptance Gate

- [ ] **Step A1: All 15 new tests pass.**
- [ ] **Step A2: Full repo test suite passes — no regressions.**
- [ ] **Step A3: `ede info --config ede.conf` lists `jobs` among loaded modules.**
- [ ] **Step A4: `ede jobs-worker --help` prints clean help.**
- [ ] **Step A5: `ede jobs-worker --config ede.conf --concurrency 1` boots cleanly against a local Redis (manual live smoke — exit on Ctrl-C).**
- [ ] **Step A6: Schema present in a migrated tenant DB:**
    ```bash
    ede migrate upgrade -t scratch_jobs --config ede.conf
    sqlite3 <tenant-db-path> ".tables" | grep -E "ir_job|ir_job_run|ir_job_lock"
    ```
    Expected: all three tables present.
- [ ] **Step A7: Pause — wait for user `commit` instruction before staging anything per CLAUDE.md rule.**

When all 7 acceptance steps are green, Slice 1 is done. The next plan (Slice 2) lands the scheduler thread, the decorator API, the XML data path, and the boot reconciler — making cron-driven jobs actually tick.

---

## Self-Review (for the plan author)

**1. Spec coverage (Slice 1 scope only):**
- ✅ WS-J1 — models (Tasks 3 / 4 / 5) + migration (Task 6)
- ✅ WS-J3 — Celery executor (Tasks 8 / 9 / 10 / 11)
- ✅ WS-J5 — `ede jobs-worker` entry point (Task 13)
- ✅ Settings (Task 7)
- ✅ Dependency wiring (Task 2)
- ✅ `env.enqueue_job()` programmatic entry (Task 12)
- ✅ End-to-end smoke proving the tracer bullet (Task 14)

Out-of-scope items (deferred to Slices 2-4) are explicitly listed at the top of the plan.

**2. Placeholder scan:** Clean — no TBD / TODO / "implement later" in plan steps. There IS one `TODO Slice 2:` comment in the executable code (Task 13.3) marking a known-deferred behaviour — that's intentional documentation, not a placeholder.

**3. Type consistency:** `Executor` protocol methods (`submit`, `submit_retry`, `graceful_shutdown`) match across Task 8 (protocol), Task 11.5 (CeleryExecutor implementation), and Task 12.4 (env.enqueue_job consumer). `register_execute_run_task` signature matches Task 10 (definition) and Task 13.3 (consumer) and Task 12.5 (fixture).

---

## Execution Handoff

**Plan complete and saved to** `docs/superpowers/plans/2026-05-18-foundation-jobs-phase-1-slice-1.md`.

Two execution options:

**1. Subagent-Driven (recommended)** — dispatch a fresh subagent per task, review between tasks, fast iteration. Best for a 14-task plan like this where each task is well-scoped.

**2. Inline Execution** — execute tasks in this session, batch with checkpoints. Better if you want to be hands-on through every step.

Which approach?
