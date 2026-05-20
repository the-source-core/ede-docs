# Foundation.jobs Phase 1 — Slice 2: Scheduler + Decorator + XML Data Path + Boot Reconciler

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `foundation.jobs` cron-driven. Ship the `JobsScheduler` thread that polls `ir.job` for due scheduled rows and dispatches via the executor; the `@api.scheduled_job` / `@api.background_job` decorators; the XML `<record model="ir.job">` data path; and the source-aware boot reconciler that lets decorator / XML / runtime rows coexist without clobbering each other.

**Architecture:** Three population paths feed one schema (`ir.job` with `source` enum from Slice 1). Decorators register callables in module-scoped `JOB_REGISTRY` lists at import time; the boot reconciler runs after `loader.load_app()` per app and reconciles those entries against `source=decorator` rows (insert new, update changed, mark removed `active=False` — never hard-delete because `ir.job.run` history FKs back). XML rows load via the standard EDE data loader unchanged. Runtime rows are admin-UI / programmatic creates. The reconciler **never touches** `source=xml` or `source=runtime` rows. The `JobsScheduler` thread lives inside `ede worker`, ticks every `JOBS_SCHEDULER_TICK_SECONDS` (default 10), acquires `ir.job.lock` per due job, creates an `ir.job.run` (pending), and hands it to the Slice 1 `CeleryExecutor.submit`. `croniter` computes `next_run_at_utc` from cron + last run.

**Tech Stack:** Python 3.10+, `croniter>=2.0` (already installed), Slice 1 services (`CeleryExecutor`, `lock.acquire_lock` / `release_lock`, `Executor` Protocol), pytest. CLI extends existing `ede worker` command.

**Does NOT cover (split into Slice 3 / 4):**
- Retry policy with `executor.submit_retry(eta=...)` re-enqueue + dead-letter notification dispatch (Slice 3)
- `env.job_progress(pct, msg)` thread-local plumbing (Slice 3)
- Settings → Technical → Jobs admin UI + RBAC seed + XML-declared heartbeat first-adopter + walkthrough (Slice 4)

---

## File Structure

### New files

| Path | Responsibility |
|---|---|
| `src/ede/foundation/jobs/services/cron.py` | `croniter` wrapper: `compute_next_run_at(cron_expr, from_utc) -> datetime` + cron-string validation |
| `src/ede/foundation/jobs/services/scheduler.py` | `JobsScheduler` class — `tick(env)` polls due jobs + dispatches; `run_forever(env, stop_event)` is the thread body |
| `src/ede/foundation/jobs/services/job_registry.py` | Module-level `JOB_REGISTRY` list of `JobSpec` dataclasses + `register_scheduled_job(spec)` + `register_background_job(spec)` + `iter_registered_jobs()` |
| `src/ede/foundation/jobs/services/reconciler.py` | `reconcile_decorator_jobs(env)` — INSERT / UPDATE / deactivate `source=decorator` rows from `JOB_REGISTRY` |
| `src/tests/foundation/jobs/test_cron.py` | Cron next-run math + invalid-cron rejection |
| `src/tests/foundation/jobs/test_scheduler.py` | `tick` picks due jobs, skips locked, advances next_run_at, dispatches via executor |
| `src/tests/foundation/jobs/test_decorators.py` | `@api.scheduled_job` registers + `@api.background_job` marks; both store metadata correctly |
| `src/tests/foundation/jobs/test_reconciler.py` | Boot reconciler INSERTs new, UPDATEs changed metadata, deactivates removed, **never touches xml/runtime rows**, raises on cross-source name collision |
| `src/tests/foundation/jobs/test_xml_data_path.py` | `<record model="ir.job">` loaded by data loader gets `source=xml`; reconciler ignores it on next boot |
| `src/tests/foundation/jobs/test_scheduler_e2e.py` | End-to-end: decorate a target → boot loads it → scheduler ticks → run completes via Celery eager → status=success |
| `src/ede/foundation/jobs/data/test_jobs.xml` | (test-only fixture) sample `ir.job` XML record used by `test_xml_data_path.py` — wired only for the test, NOT in production manifest yet (that ships in Slice 4 with the heartbeat) |

### Existing files modified

| Path | Change |
|---|---|
| `src/ede/core/api.py` | Export `scheduled_job` + `background_job` decorators (re-exports from `ede.foundation.jobs.services.job_registry`) |
| `src/ede/core/loader.py` | After `load_app(app_key)` completes for the `jobs` app, trigger the boot reconciler against the loaded `JOB_REGISTRY` |
| `src/ede/cli/commands/worker.py` | Spawn the `JobsScheduler` thread + supervisor when `--no-jobs` is NOT set (default behaviour: scheduler runs) |

---

## Pre-flight

- [ ] **P1: Verify Slice 1 is at HEAD.**
    ```bash
    git log --oneline -1 src/ede/foundation/jobs/services/executor.py
    ```
    Expected: shows commit `843ce2a` (Slice 1) or later. If absent, Slice 1 wasn't merged — stop and fix.

- [ ] **P2: Confirm the existing jobs suite is green.**
    ```bash
    pytest src/tests/foundation/jobs/ -v
    ```
    Expected: 15 passed in ~1s. If anything is red, fix before touching Slice 2.

- [ ] **P3: Read Slice 1's services for context.**
    Skim `src/ede/foundation/jobs/services/executor.py`, `task_wrapper.py`, `lock.py`, and `src/tests/foundation/jobs/conftest.py`. Slice 2 reuses all of them — `CeleryExecutor.submit(env, run)`, `acquire_lock`, the `env_with_jobs_and_executor` fixture.

---

## Task 1: Cron helper (`croniter` wrapper)

**Files:**
- Create: `src/ede/foundation/jobs/services/cron.py`
- Test: `src/tests/foundation/jobs/test_cron.py`

- [ ] **Step 1.1: Failing test first**

    Create `src/tests/foundation/jobs/test_cron.py`:
    ```python
    """Tests for the cron-expression helper."""
    from datetime import datetime, timezone

    import pytest

    from ede.foundation.jobs.services.cron import (
        compute_next_run_at,
        validate_cron_expression,
        InvalidCronExpression,
    )


    def test_compute_next_run_at_every_2_minutes():
        now = datetime(2026, 5, 18, 12, 0, 30, tzinfo=timezone.utc)
        nxt = compute_next_run_at("*/2 * * * *", from_utc=now)
        assert nxt == datetime(2026, 5, 18, 12, 2, 0, tzinfo=timezone.utc)


    def test_compute_next_run_at_hourly_on_minute_15():
        now = datetime(2026, 5, 18, 12, 0, 0, tzinfo=timezone.utc)
        nxt = compute_next_run_at("15 * * * *", from_utc=now)
        assert nxt == datetime(2026, 5, 18, 12, 15, 0, tzinfo=timezone.utc)


    def test_compute_next_run_at_returns_utc_aware_datetime():
        now = datetime(2026, 5, 18, 12, 0, 0, tzinfo=timezone.utc)
        nxt = compute_next_run_at("0 * * * *", from_utc=now)
        assert nxt.tzinfo is not None
        assert nxt.tzinfo.utcoffset(nxt).total_seconds() == 0


    def test_validate_cron_expression_accepts_valid():
        validate_cron_expression("*/5 * * * *")
        validate_cron_expression("0 0 * * 0")
        validate_cron_expression("30 2 1 * *")


    def test_validate_cron_expression_rejects_garbage():
        with pytest.raises(InvalidCronExpression):
            validate_cron_expression("not a cron")
        with pytest.raises(InvalidCronExpression):
            validate_cron_expression("* * * *")           # 4 fields, not 5
        with pytest.raises(InvalidCronExpression):
            validate_cron_expression("60 * * * *")        # minute > 59
    ```

- [ ] **Step 1.2: Run — expect ModuleNotFoundError**
    ```bash
    pytest src/tests/foundation/jobs/test_cron.py -v
    ```
    Expected: FAIL with `ModuleNotFoundError: No module named 'ede.foundation.jobs.services.cron'`.

- [ ] **Step 1.3: Write the cron helper**

    Create `src/ede/foundation/jobs/services/cron.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Cron expression helper — thin wrapper around `croniter`.

    Exists so the scheduler doesn't import croniter directly and so cron-string
    validation has one canonical entry point (decorator validation at registration
    time + XML data loader validation at row-create time + admin UI form validation
    all funnel through validate_cron_expression).
    """
    from __future__ import annotations

    from datetime import datetime, timezone

    from croniter import croniter, CroniterBadCronError


    class InvalidCronExpression(ValueError):
        """Raised when a cron string fails parse-time validation."""


    def validate_cron_expression(expr: str) -> None:
        """Raise InvalidCronExpression if expr is not a valid 5-field cron string."""
        if not isinstance(expr, str) or not expr.strip():
            raise InvalidCronExpression("cron expression must be a non-empty string")
        parts = expr.strip().split()
        if len(parts) != 5:
            raise InvalidCronExpression(
                f"cron expression must have exactly 5 fields, got {len(parts)}: {expr!r}"
            )
        try:
            croniter(expr, datetime.now(tz=timezone.utc))
        except (CroniterBadCronError, ValueError) as exc:
            raise InvalidCronExpression(f"invalid cron expression {expr!r}: {exc}") from exc


    def compute_next_run_at(cron_expr: str, *, from_utc: datetime) -> datetime:
        """Return the next UTC-aware datetime at which the cron fires after from_utc."""
        if from_utc.tzinfo is None:
            raise ValueError("from_utc must be a timezone-aware datetime")
        itr = croniter(cron_expr, from_utc)
        nxt = itr.get_next(datetime)
        # croniter returns naive datetime in some versions — force UTC
        if nxt.tzinfo is None:
            nxt = nxt.replace(tzinfo=timezone.utc)
        return nxt
    ```

- [ ] **Step 1.4: Run — expect PASS (5 tests)**
    ```bash
    pytest src/tests/foundation/jobs/test_cron.py -v
    ```
    Expected: 5 passed.

- [ ] **Step 1.5: Lint**
    ```bash
    ruff check src/ede/foundation/jobs/services/cron.py src/tests/foundation/jobs/test_cron.py
    ```
    Expected: clean.

---

## Task 2: `JobsScheduler` class — `tick()` + `run_forever()`

**Files:**
- Create: `src/ede/foundation/jobs/services/scheduler.py`
- Test: `src/tests/foundation/jobs/test_scheduler.py`

- [ ] **Step 2.1: Failing tests first**

    Create `src/tests/foundation/jobs/test_scheduler.py`:
    ```python
    """Tests for JobsScheduler.tick — due-job detection, lock acquisition, dispatch."""
    from datetime import datetime, timedelta, timezone
    from unittest.mock import MagicMock

    from ede.foundation.jobs.services.scheduler import JobsScheduler


    def _utc(*args):
        return datetime(*args, tzinfo=timezone.utc)


    def _now_minus(seconds: int) -> datetime:
        return datetime.now(tz=timezone.utc) - timedelta(seconds=seconds)


    def test_tick_dispatches_a_single_due_scheduled_job(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor
        env.models["ir.job"].create({
            "name": "scheduler-test.due-1",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/2 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),         # due 1 min ago
        })

        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor, tick_seconds=10)

        dispatched = scheduler.tick(env)

        assert dispatched == 1
        assert executor.submit.call_count == 1
        # The run row passed to submit() must be a pending ir.job.run linked to our job
        passed_env, passed_run = executor.submit.call_args.args
        assert passed_env is env
        assert passed_run.status == "pending"
        assert passed_run.job_id.name == "scheduler-test.due-1"


    def test_tick_skips_jobs_whose_next_run_is_in_the_future(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor
        env.models["ir.job"].create({
            "name": "scheduler-test.future-1",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "0 0 * * *",
            "source": "runtime",
            "next_run_at_utc": datetime.now(tz=timezone.utc) + timedelta(hours=1),
        })
        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor, tick_seconds=10)

        assert scheduler.tick(env) == 0
        executor.submit.assert_not_called()


    def test_tick_skips_inactive_jobs(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor
        env.models["ir.job"].create({
            "name": "scheduler-test.inactive",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/2 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
            "active": False,
        })
        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor, tick_seconds=10)

        assert scheduler.tick(env) == 0
        executor.submit.assert_not_called()


    def test_tick_skips_non_scheduled_kinds(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor
        env.models["ir.job"].create({
            "name": "scheduler-test.queued-only",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",                          # not "scheduled"
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
        })
        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor, tick_seconds=10)

        assert scheduler.tick(env) == 0


    def test_tick_advances_next_run_at_after_dispatch(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor
        job = env.models["ir.job"].create({
            "name": "scheduler-test.advance",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/2 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
        })
        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor, tick_seconds=10)

        scheduler.tick(env)

        refreshed = env.models["ir.job"].browse(job.id)
        assert refreshed.next_run_at_utc is not None
        assert refreshed.next_run_at_utc > datetime.now(tz=timezone.utc)


    def test_tick_does_not_dispatch_when_lock_already_held(env_with_jobs_and_executor):
        """Two scheduler ticks in quick succession must not double-dispatch the same job."""
        env = env_with_jobs_and_executor
        env.models["ir.job"].create({
            "name": "scheduler-test.locked",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/2 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
        })

        # Pre-acquire the lock to simulate another worker holding it
        from ede.foundation.jobs.services.lock import acquire_lock
        assert acquire_lock(env, lock_key="job:scheduler-test.locked", worker_id="other-worker", timeout_seconds=60) is True

        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor, tick_seconds=10)

        assert scheduler.tick(env) == 0
        executor.submit.assert_not_called()
    ```

- [ ] **Step 2.2: Run — expect ModuleNotFoundError**
    ```bash
    pytest src/tests/foundation/jobs/test_scheduler.py -v
    ```

- [ ] **Step 2.3: Write `JobsScheduler`**

    Create `src/ede/foundation/jobs/services/scheduler.py`:
    ```python
    # -*- coding: utf-8 -*-
    """JobsScheduler — polls ir.job for due scheduled rows and dispatches via the executor.

    Slice 2 ships the in-process scheduler thread that lives inside `ede worker`.
    It does NOT execute work itself — execution is the Celery executor's job.
    The scheduler's responsibilities are:
        1. Find due ir.job rows (kind=scheduled, active=True, next_run_at_utc <= now).
        2. Acquire ir.job.lock to dedup against parallel schedulers in other workers.
        3. Create an ir.job.run (status=pending) for the upcoming execution.
        4. Hand the run to executor.submit(env, run).
        5. Advance ir.job.next_run_at_utc to the next cron firing time.
    """
    from __future__ import annotations

    import logging
    import socket
    import os
    import threading
    from datetime import datetime, timezone
    from typing import TYPE_CHECKING

    from .cron import compute_next_run_at
    from .lock import acquire_lock, release_lock

    if TYPE_CHECKING:
        from ede.core.env import Env
        from ede.core.orm.recordset import RecordSet
        from .executor import Executor

    logger = logging.getLogger(__name__)


    def _now_utc() -> datetime:
        return datetime.now(tz=timezone.utc)


    def _worker_id() -> str:
        return f"{socket.gethostname()}:{os.getpid()}:scheduler"


    class JobsScheduler:
        """Polls ir.job rows for due scheduled jobs and dispatches to the executor."""

        def __init__(self, *, executor: "Executor", tick_seconds: int = 10):
            self.executor = executor
            self.tick_seconds = tick_seconds
            self._stop = threading.Event()

        def tick(self, env: "Env") -> int:
            """One scheduler tick. Returns number of jobs dispatched."""
            now = _now_utc()
            due_jobs = env.models["ir.job"].search(
                [
                    ("kind", "=", "scheduled"),
                    ("active", "=", True),
                    ("next_run_at_utc", "<=", now),
                ],
                limit=100,
                order="priority asc, next_run_at_utc asc",
            )

            dispatched = 0
            for job in due_jobs:
                lock_key = f"job:{job.name}"
                if not acquire_lock(
                    env,
                    lock_key=lock_key,
                    worker_id=_worker_id(),
                    timeout_seconds=job.timeout_seconds,
                ):
                    logger.debug("scheduler: skipping %s — lock held", job.name)
                    continue

                try:
                    run = self._create_pending_run(env, job)
                    self.executor.submit(env, run)
                    self._advance_next_run_at(env, job, from_utc=now)
                    dispatched += 1
                except Exception:                                      # noqa: BLE001
                    # Catch-all so one bad job can't kill the tick loop.
                    logger.exception("scheduler: failed to dispatch %s", job.name)
                    # Release the lock so a future tick can retry.
                    release_lock(env, lock_key=lock_key)
            return dispatched

        def run_forever(self, env: "Env") -> None:
            """Long-lived loop — call once from the scheduler thread."""
            logger.info("JobsScheduler starting (tick_seconds=%s)", self.tick_seconds)
            while not self._stop.is_set():
                try:
                    self.tick(env)
                except Exception:                                      # noqa: BLE001
                    logger.exception("scheduler tick failed")
                self._stop.wait(self.tick_seconds)
            logger.info("JobsScheduler stopped")

        def stop(self) -> None:
            """Signal run_forever to exit at the next tick boundary."""
            self._stop.set()

        # ── Internals ────────────────────────────────────────────────────

        def _create_pending_run(self, env: "Env", job: "RecordSet") -> "RecordSet":
            return env.models["ir.job.run"].create(
                {
                    "job_id": job.id,
                    "attempt_number": 1,
                    "status": "pending",
                    "payload": {},
                    "queued_at_utc": _now_utc(),
                }
            )

        def _advance_next_run_at(self, env: "Env", job: "RecordSet", *, from_utc: datetime) -> None:
            if not job.cron:
                return
            next_at = compute_next_run_at(job.cron, from_utc=from_utc)
            job.write({
                "next_run_at_utc": next_at,
                "last_run_at_utc": from_utc,
            })
    ```

- [ ] **Step 2.4: Run scheduler tests — expect 6 PASS**
    ```bash
    pytest src/tests/foundation/jobs/test_scheduler.py -v
    ```

- [ ] **Step 2.5: Lint**
    ```bash
    ruff check src/ede/foundation/jobs/services/scheduler.py src/tests/foundation/jobs/test_scheduler.py
    ```

---

## Task 3: `JOB_REGISTRY` + `@api.scheduled_job` / `@api.background_job` decorators

**Files:**
- Create: `src/ede/foundation/jobs/services/job_registry.py`
- Modify: `src/ede/core/api.py` — add the two re-exports
- Test: `src/tests/foundation/jobs/test_decorators.py`

- [ ] **Step 3.1: Failing test first**

    Create `src/tests/foundation/jobs/test_decorators.py`:
    ```python
    """Tests for @api.scheduled_job and @api.background_job decorator surface."""
    import pytest

    from ede.core import api
    from ede.foundation.jobs.services.job_registry import (
        JOB_REGISTRY,
        JobSpec,
        iter_registered_jobs,
        _clear_registry_for_tests,
    )


    @pytest.fixture(autouse=True)
    def reset_registry():
        _clear_registry_for_tests()
        yield
        _clear_registry_for_tests()


    def test_scheduled_job_decorator_registers_spec():
        @api.scheduled_job(
            name="test.decorated.ping",
            cron="*/5 * * * *",
            retry_policy="exponential",
            retry_max_attempts=5,
            priority=4,
            description="ping",
        )
        def ping(env, payload=None):
            return "pong"

        specs = list(iter_registered_jobs())
        assert len(specs) == 1
        spec = specs[0]
        assert isinstance(spec, JobSpec)
        assert spec.name == "test.decorated.ping"
        assert spec.kind == "scheduled"
        assert spec.cron == "*/5 * * * *"
        assert spec.retry_policy == "exponential"
        assert spec.retry_max_attempts == 5
        assert spec.priority == 4
        assert spec.description == "ping"
        assert spec.target.endswith("ping")          # dotted path captured


    def test_scheduled_job_decorator_returns_callable_unchanged():
        @api.scheduled_job(name="test.identity", cron="0 * * * *")
        def my_target(env, payload=None):
            return "hello"

        assert my_target.__name__ == "my_target"
        assert my_target(env=None, payload={}) == "hello"


    def test_background_job_decorator_marks_callable():
        @api.background_job(name="test.bg")
        def my_bg(env, payload=None):
            return "ok"

        specs = list(iter_registered_jobs())
        assert len(specs) == 1
        assert specs[0].kind == "queued"
        assert specs[0].cron is None


    def test_scheduled_job_rejects_invalid_cron():
        from ede.foundation.jobs.services.cron import InvalidCronExpression
        with pytest.raises(InvalidCronExpression):
            @api.scheduled_job(name="test.bad-cron", cron="not a cron")
            def bad(env, payload=None):
                pass


    def test_scheduled_job_requires_name():
        with pytest.raises(ValueError, match="name"):
            @api.scheduled_job(name="", cron="*/5 * * * *")
            def no_name(env, payload=None):
                pass


    def test_duplicate_name_in_same_process_raises():
        @api.scheduled_job(name="test.dup", cron="*/5 * * * *")
        def a(env, payload=None):
            pass

        with pytest.raises(ValueError, match="already registered"):
            @api.scheduled_job(name="test.dup", cron="*/5 * * * *")
            def b(env, payload=None):
                pass
    ```

- [ ] **Step 3.2: Run — expect ImportError / ModuleNotFoundError**
    ```bash
    pytest src/tests/foundation/jobs/test_decorators.py -v
    ```

- [ ] **Step 3.3: Write `job_registry.py`**

    Create `src/ede/foundation/jobs/services/job_registry.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Job registry — module-level list of decorator-declared JobSpec rows.

    @api.scheduled_job and @api.background_job append to JOB_REGISTRY at import
    time. The boot reconciler walks it after each app loads and reconciles with
    source=decorator ir.job rows.

    The registry is process-global — a single source of truth across all loaded
    apps in this Python process.
    """
    from __future__ import annotations

    from dataclasses import dataclass, field
    from typing import Any, Callable, Iterator, List, Optional

    from .cron import validate_cron_expression


    @dataclass(frozen=True)
    class JobSpec:
        """Captured metadata for a decorator-declared job."""

        name: str
        target: str                                                    # dotted python path
        kind: str                                                      # "scheduled" | "queued"
        cron: Optional[str] = None
        retry_policy: str = "exponential"
        retry_max_attempts: int = 3
        retry_base_seconds: int = 60
        priority: int = 5
        timeout_seconds: int = 600
        retry_on_interrupt: bool = True
        description: str = ""
        module_key: str = ""                                            # filled by decorator from caller's __module__


    JOB_REGISTRY: List[JobSpec] = []
    _NAMES_SEEN: set[str] = set()


    def _clear_registry_for_tests() -> None:
        """ONLY for tests — reset the process-global registry."""
        JOB_REGISTRY.clear()
        _NAMES_SEEN.clear()


    def iter_registered_jobs() -> Iterator[JobSpec]:
        """Yield every JobSpec currently in the registry."""
        yield from JOB_REGISTRY


    def _register(spec: JobSpec) -> None:
        if spec.name in _NAMES_SEEN:
            raise ValueError(
                f"ir.job name {spec.name!r} is already registered by another decorator. "
                f"Job names must be process-globally unique."
            )
        _NAMES_SEEN.add(spec.name)
        JOB_REGISTRY.append(spec)


    def _dotted_target(fn: Callable) -> str:
        module = getattr(fn, "__module__", None) or ""
        qualname = getattr(fn, "__qualname__", None) or getattr(fn, "__name__", "")
        if not module:
            return qualname
        return f"{module}.{qualname}"


    def scheduled_job(
        *,
        name: str,
        cron: str,
        retry_policy: str = "exponential",
        retry_max_attempts: int = 3,
        retry_base_seconds: int = 60,
        priority: int = 5,
        timeout_seconds: int = 600,
        retry_on_interrupt: bool = True,
        description: str = "",
    ) -> Callable[[Callable], Callable]:
        """Decorator — register a cron-driven scheduled job."""
        if not name:
            raise ValueError("scheduled_job requires a non-empty name")
        validate_cron_expression(cron)

        def _decorator(fn: Callable) -> Callable:
            spec = JobSpec(
                name=name,
                target=_dotted_target(fn),
                kind="scheduled",
                cron=cron,
                retry_policy=retry_policy,
                retry_max_attempts=retry_max_attempts,
                retry_base_seconds=retry_base_seconds,
                priority=priority,
                timeout_seconds=timeout_seconds,
                retry_on_interrupt=retry_on_interrupt,
                description=description,
                module_key=getattr(fn, "__module__", "") or "",
            )
            _register(spec)
            return fn

        return _decorator


    def background_job(
        *,
        name: str,
        description: str = "",
        priority: int = 5,
        timeout_seconds: int = 600,
        retry_policy: str = "exponential",
        retry_max_attempts: int = 3,
        retry_base_seconds: int = 60,
        retry_on_interrupt: bool = True,
    ) -> Callable[[Callable], Callable]:
        """Decorator — marker for a callable expected to be invoked via env.enqueue_job()."""
        if not name:
            raise ValueError("background_job requires a non-empty name")

        def _decorator(fn: Callable) -> Callable:
            spec = JobSpec(
                name=name,
                target=_dotted_target(fn),
                kind="queued",
                cron=None,
                retry_policy=retry_policy,
                retry_max_attempts=retry_max_attempts,
                retry_base_seconds=retry_base_seconds,
                priority=priority,
                timeout_seconds=timeout_seconds,
                retry_on_interrupt=retry_on_interrupt,
                description=description,
                module_key=getattr(fn, "__module__", "") or "",
            )
            _register(spec)
            return fn

        return _decorator
    ```

- [ ] **Step 3.4: Re-export from `ede.core.api`**

    Open `src/ede/core/api.py`. Find the existing decorator exports (e.g. `model`, `on_command`, `on_event`, `on_hook`, `ai_tool`, `metric`, `extend_model`). Match the style and add:

    ```python
    from ede.foundation.jobs.services.job_registry import (
        scheduled_job,
        background_job,
    )

    __all__ = [..., "scheduled_job", "background_job"]
    ```

    **Important:** `ede.foundation.jobs.services.job_registry` imports from `ede.foundation.jobs.services.cron`, which imports `croniter`. If `ede.core.api` is imported very early in the boot chain (before `ede.foundation` is on `sys.path`), this could circular-import. **Test by running**:
    ```bash
    python -c "from ede.core import api; print(api.scheduled_job, api.background_job)"
    ```
    Expected: prints two callables. If ImportError mentions a circular dependency, lazy-import the names inside `api.py` (move the `from ...job_registry import ...` to the bottom of the file or behind a property). Report whichever approach you took.

- [ ] **Step 3.5: Run decorator tests — expect 6 PASS**
    ```bash
    pytest src/tests/foundation/jobs/test_decorators.py -v
    ```

- [ ] **Step 3.6: Lint**
    ```bash
    ruff check src/ede/foundation/jobs/services/job_registry.py src/ede/core/api.py src/tests/foundation/jobs/test_decorators.py
    ```

---

## Task 4: Boot reconciler

**Files:**
- Create: `src/ede/foundation/jobs/services/reconciler.py`
- Test: `src/tests/foundation/jobs/test_reconciler.py`

- [ ] **Step 4.1: Failing test first**

    Create `src/tests/foundation/jobs/test_reconciler.py`:
    ```python
    """Tests for the boot-time reconciler — source-aware INSERT/UPDATE/deactivate."""
    import pytest

    from ede.foundation.jobs.services.job_registry import (
        JobSpec,
        JOB_REGISTRY,
        _clear_registry_for_tests,
    )
    from ede.foundation.jobs.services.reconciler import reconcile_decorator_jobs


    @pytest.fixture(autouse=True)
    def reset_registry():
        _clear_registry_for_tests()
        yield
        _clear_registry_for_tests()


    def _push(spec):
        JOB_REGISTRY.append(spec)


    def test_inserts_new_decorator_row(env_with_jobs):
        env = env_with_jobs
        _push(JobSpec(
            name="r.insert-1",
            target="pkg.mod.fn",
            kind="scheduled",
            cron="*/5 * * * *",
            module_key="foundation.jobs",
        ))

        reconcile_decorator_jobs(env, module_keys={"foundation.jobs"})

        rows = env.models["ir.job"].search([("name", "=", "r.insert-1")])
        assert len(rows) == 1
        row = rows[0]
        assert row.source == "decorator"
        assert row.cron == "*/5 * * * *"
        assert row.target == "pkg.mod.fn"
        assert row.active is True
        assert row.next_run_at_utc is not None       # reconciler seeds the first firing time


    def test_updates_existing_decorator_row_when_metadata_changes(env_with_jobs):
        env = env_with_jobs
        # Initial registration
        _push(JobSpec(name="r.update-1", target="pkg.mod.fn", kind="scheduled",
                     cron="0 * * * *", priority=5, module_key="foundation.jobs"))
        reconcile_decorator_jobs(env, module_keys={"foundation.jobs"})

        # Second pass — metadata changed
        _clear_registry_for_tests()
        _push(JobSpec(name="r.update-1", target="pkg.mod.fn", kind="scheduled",
                     cron="*/15 * * * *", priority=3, module_key="foundation.jobs"))
        reconcile_decorator_jobs(env, module_keys={"foundation.jobs"})

        rows = env.models["ir.job"].search([("name", "=", "r.update-1")])
        assert len(rows) == 1
        assert rows[0].cron == "*/15 * * * *"
        assert rows[0].priority == 3


    def test_deactivates_decorator_row_when_decorator_removed(env_with_jobs):
        env = env_with_jobs
        _push(JobSpec(name="r.gone", target="pkg.mod.fn", kind="scheduled",
                     cron="*/5 * * * *", module_key="foundation.jobs"))
        reconcile_decorator_jobs(env, module_keys={"foundation.jobs"})
        assert env.models["ir.job"].search([("name", "=", "r.gone")])[0].active is True

        # Decorator removed in the next boot
        _clear_registry_for_tests()
        reconcile_decorator_jobs(env, module_keys={"foundation.jobs"})

        rows = env.models["ir.job"].search([("name", "=", "r.gone")])
        assert len(rows) == 1                        # never hard-deleted — history FKs back
        assert rows[0].active is False


    def test_never_touches_xml_source_row(env_with_jobs):
        env = env_with_jobs
        # Pre-seed an XML-source row that doesn't match any decorator
        env.models["ir.job"].create({
            "name": "r.xml-row",
            "module_key": "foundation.jobs",
            "target": "pkg.mod.xml_fn",
            "kind": "scheduled",
            "cron": "*/2 * * * *",
            "source": "xml",
        })

        reconcile_decorator_jobs(env, module_keys={"foundation.jobs"})

        row = env.models["ir.job"].search([("name", "=", "r.xml-row")])[0]
        assert row.source == "xml"
        assert row.active is True                    # NEVER deactivated by reconciler


    def test_never_touches_runtime_source_row(env_with_jobs):
        env = env_with_jobs
        env.models["ir.job"].create({
            "name": "r.runtime-row",
            "module_key": "foundation.jobs",
            "target": "pkg.mod.rt_fn",
            "kind": "queued",
            "source": "runtime",
        })

        reconcile_decorator_jobs(env, module_keys={"foundation.jobs"})

        row = env.models["ir.job"].search([("name", "=", "r.runtime-row")])[0]
        assert row.source == "runtime"
        assert row.active is True


    def test_decorator_name_collision_with_xml_logs_and_skips(env_with_jobs, caplog):
        """If a decorator tries to register a name that already exists as source=xml,
        the reconciler logs a warning and skips — data and admin intent always beat code.
        """
        env = env_with_jobs
        env.models["ir.job"].create({
            "name": "r.conflict",
            "module_key": "foundation.jobs",
            "target": "pkg.mod.xml_fn",
            "kind": "scheduled",
            "cron": "0 * * * *",
            "source": "xml",
        })

        _push(JobSpec(name="r.conflict", target="pkg.mod.decorator_fn", kind="scheduled",
                     cron="*/15 * * * *", module_key="foundation.jobs"))

        import logging
        with caplog.at_level(logging.WARNING, logger="ede.foundation.jobs.services.reconciler"):
            reconcile_decorator_jobs(env, module_keys={"foundation.jobs"})

        # XML row unchanged
        row = env.models["ir.job"].search([("name", "=", "r.conflict")])[0]
        assert row.source == "xml"
        assert row.target == "pkg.mod.xml_fn"
        assert row.cron == "0 * * * *"
        # Warning logged
        assert any("r.conflict" in rec.message and "xml" in rec.message.lower() for rec in caplog.records)


    def test_module_keys_filter_scopes_reconciler(env_with_jobs):
        """If module_keys is passed, only specs whose module_key starts with one of them are reconciled."""
        env = env_with_jobs
        _push(JobSpec(name="r.in-scope", target="ede.foundation.jobs.demo.x", kind="scheduled",
                     cron="*/5 * * * *", module_key="ede.foundation.jobs.demo"))
        _push(JobSpec(name="r.out-of-scope", target="ede.foundation.other.y", kind="scheduled",
                     cron="*/5 * * * *", module_key="ede.foundation.other"))

        reconcile_decorator_jobs(env, module_keys={"ede.foundation.jobs"})

        assert env.models["ir.job"].search([("name", "=", "r.in-scope")])
        assert not env.models["ir.job"].search([("name", "=", "r.out-of-scope")])
    ```

- [ ] **Step 4.2: Run — expect ModuleNotFoundError**
    ```bash
    pytest src/tests/foundation/jobs/test_reconciler.py -v
    ```

- [ ] **Step 4.3: Write the reconciler**

    Create `src/ede/foundation/jobs/services/reconciler.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Boot-time reconciler for decorator-source ir.job rows.

    Runs after loader.load_app(...) for each app that contributes decorators.
    Walks JOB_REGISTRY and ensures every spec has a matching source=decorator
    ir.job row with up-to-date metadata. Specs that disappear (e.g. the
    decorator was removed) get their row marked active=False — never hard-
    deleted, because ir.job.run history FKs back.

    The reconciler NEVER touches source=xml or source=runtime rows.
    """
    from __future__ import annotations

    import logging
    from dataclasses import asdict
    from datetime import datetime, timezone
    from typing import Iterable, Optional, Set, TYPE_CHECKING

    from .cron import compute_next_run_at
    from .job_registry import JobSpec, iter_registered_jobs

    if TYPE_CHECKING:
        from ede.core.env import Env

    logger = logging.getLogger(__name__)

    # Fields the reconciler is allowed to overwrite on source=decorator rows.
    _RECONCILED_FIELDS = (
        "module_key",
        "target",
        "kind",
        "cron",
        "retry_policy",
        "retry_max_attempts",
        "retry_base_seconds",
        "priority",
        "timeout_seconds",
        "retry_on_interrupt",
        "description",
    )


    def _now_utc() -> datetime:
        return datetime.now(tz=timezone.utc)


    def _spec_to_row_values(spec: JobSpec) -> dict:
        return {
            "name": spec.name,
            "module_key": spec.module_key or "foundation.jobs",
            "target": spec.target,
            "kind": spec.kind,
            "cron": spec.cron,
            "retry_policy": spec.retry_policy,
            "retry_max_attempts": spec.retry_max_attempts,
            "retry_base_seconds": spec.retry_base_seconds,
            "priority": spec.priority,
            "timeout_seconds": spec.timeout_seconds,
            "retry_on_interrupt": spec.retry_on_interrupt,
            "description": spec.description,
            "source": "decorator",
            "active": True,
        }


    def _in_scope(spec: JobSpec, module_keys: Optional[Set[str]]) -> bool:
        if module_keys is None:
            return True
        mk = spec.module_key or ""
        return any(mk == k or mk.startswith(k + ".") for k in module_keys)


    def reconcile_decorator_jobs(
        env: "Env",
        *,
        module_keys: Optional[Iterable[str]] = None,
    ) -> dict:
        """Reconcile JOB_REGISTRY entries against source=decorator ir.job rows.

        If module_keys is provided, only specs whose module_key matches one of
        those prefixes are considered (used by the loader to scope per-app boots).
        Returns {"inserted": n, "updated": n, "deactivated": n, "skipped": n}.
        """
        keys_set = set(module_keys) if module_keys is not None else None
        specs = [s for s in iter_registered_jobs() if _in_scope(s, keys_set)]
        spec_names = {s.name for s in specs}

        job_proxy = env.models["ir.job"]
        inserted = updated = deactivated = skipped = 0

        for spec in specs:
            existing = job_proxy.search([("name", "=", spec.name)], limit=1)
            if not existing:
                values = _spec_to_row_values(spec)
                if spec.cron:
                    values["next_run_at_utc"] = compute_next_run_at(spec.cron, from_utc=_now_utc())
                job_proxy.create(values)
                inserted += 1
                continue

            row = existing[0]
            if row.source != "decorator":
                logger.warning(
                    "decorator job %r conflicts with existing source=%s row — skipping "
                    "(data and admin intent beat code)",
                    spec.name, row.source,
                )
                skipped += 1
                continue

            patch = {
                f: getattr(spec, f) if f != "module_key" else (spec.module_key or "foundation.jobs")
                for f in _RECONCILED_FIELDS
            }
            patch["active"] = True
            # Only emit the write if anything actually changed.
            current = {f: getattr(row, f) for f in patch}
            if current != patch:
                row.write(patch)
                updated += 1

        # Deactivate decorator rows that are no longer in the registry.
        # We have to do this per scope — only consider rows whose module_key matches
        # the keys we're reconciling. Otherwise reconciling app A would wipe app B's rows.
        if keys_set is None:
            existing_decorator_rows = job_proxy.search([("source", "=", "decorator")])
        else:
            existing_decorator_rows = []
            for k in keys_set:
                existing_decorator_rows.extend(
                    job_proxy.search([
                        ("source", "=", "decorator"),
                        ("module_key", "in", [k] + [f"{k}.{suffix}" for suffix in ()]),  # exact-match list
                    ])
                )
                # Also include startswith matches via a second search (the ORM doesn't
                # have a `like` operator in this codebase — confirm in the implementer
                # step and adjust if needed).
                existing_decorator_rows.extend(
                    [r for r in job_proxy.search([("source", "=", "decorator")])
                     if (r.module_key or "").startswith(k + ".")]
                )
        # Dedup by record id
        seen_ids = set()
        unique_rows = []
        for r in existing_decorator_rows:
            if r.id not in seen_ids:
                seen_ids.add(r.id)
                unique_rows.append(r)

        for row in unique_rows:
            if row.name not in spec_names and row.active:
                row.write({"active": False})
                deactivated += 1

        logger.info(
            "reconciler: %d inserted, %d updated, %d deactivated, %d skipped",
            inserted, updated, deactivated, skipped,
        )
        return {"inserted": inserted, "updated": updated, "deactivated": deactivated, "skipped": skipped}
    ```

    **Notes for the implementer:**
    1. The `module_keys` filter logic above is verbose because EDE's search may not have a `like` / `startswith` operator. **Check the ORM API first** (`grep -rn "startswith\|like\|ilike" src/ede/core/orm/`). If a `like` operator exists, replace the verbose loop with `[("module_key", "like", f"{k}%")]`. If not, the in-Python filter (the `[r for r in ... if ...]` list comprehension) is fine — fix any duplication.
    2. Initial `next_run_at_utc` is set on INSERT only. UPDATE deliberately doesn't touch it — so a cron change mid-flight doesn't reset the schedule. If you prefer cron-change-resets-schedule semantics, recompute in the UPDATE path too.

- [ ] **Step 4.4: Run reconciler tests — expect 7 PASS**
    ```bash
    pytest src/tests/foundation/jobs/test_reconciler.py -v
    ```

- [ ] **Step 4.5: Lint**
    ```bash
    ruff check src/ede/foundation/jobs/services/reconciler.py src/tests/foundation/jobs/test_reconciler.py
    ```

---

## Task 5: Loader integration — call the reconciler at boot

**Files:**
- Modify: `src/ede/core/loader.py` — invoke `reconcile_decorator_jobs` after the jobs app loads
- Test: extend `src/tests/foundation/jobs/test_reconciler.py` with one integration test that simulates a real loader boot

- [ ] **Step 5.1: Read the loader**

    Open `src/ede/core/loader.py`. Find the `load_app(app_key)` method (or equivalent — the entry point that imports a foundation app and registers its models/handlers/events). Look at what happens after each app loads — there's likely a registry-finalize step or a post-load hook.

    **Decide:** the reconciler can hook in at one of two points:
    - **Per-app** — after each `load_app(...)` returns, if `app_key == "jobs"` is in the load chain, run the reconciler scoped to that module's keys. (cleaner, scales)
    - **Once at end of boot** — `bootstrap_environment` calls the reconciler after `loader.load_all()` finishes, scoped to ALL `module_keys`. (simpler, fine for Slice 2)

    **Recommendation:** per-app, hooked into `load_app("jobs")` specifically. The jobs app itself contains no decorators in Slice 2; the reconciler is shared infrastructure consumed by future apps (Slice 4 heartbeat, later by domain modules). Hook the call inside `load_app` such that ALL apps' decorators are reconciled together once `jobs` itself has loaded — this is a one-time boot call, not a per-app dance.

    **Simpler option** if `load_app` doesn't have a natural seam: hook it into `bootstrap_environment` after `loader.load_all()` completes, before returning the env. The reconciler reads `JOB_REGISTRY` (populated by every app's import chain) and writes to `env.models["ir.job"]`. Pick whichever produces the cleanest diff.

- [ ] **Step 5.2: Wire the reconciler call**

    Add the call in whichever location you chose. Example for `bootstrap_environment`:
    ```python
    # In src/ede/core/bootstrap.py (or wherever bootstrap_environment lives)

    def bootstrap_environment(...):
        ...
        loader.load_all(...)

        # Foundation.jobs reconciler — populates source=decorator rows in ir.job
        # from the @api.scheduled_job / @api.background_job decorator registry.
        # Conditional: only run if the jobs app is loaded (so non-jobs deployments
        # don't try to import foundation.jobs services).
        if "jobs" in {app.key for app in registry.iter_apps()}:
            from ede.foundation.jobs.services.reconciler import reconcile_decorator_jobs
            with env.transaction():
                reconcile_decorator_jobs(env)

        return env
    ```

    Adjust based on the actual `bootstrap_environment` signature + how it gets an env. The key constraints:
    1. The reconciler call must be **inside a transaction** (`env.transaction()`) so all INSERT/UPDATE/deactivate happen atomically.
    2. The conditional check on `"jobs" in active_apps` is required so non-jobs deployments (if any) don't crash on the import.
    3. Imports go at the **module top** if `bootstrap.py` always has `foundation.jobs` available; otherwise an inline import behind the `if` is acceptable as a hard-dep guard (one of CLAUDE.md's two legal inline-import exceptions).

- [ ] **Step 5.3: Add an integration test**

    Append to `src/tests/foundation/jobs/test_reconciler.py`:
    ```python
    def test_full_boot_reconciles_decorators_into_env(env_with_jobs):
        """After loader runs, decorators registered via @api.scheduled_job appear as ir.job rows."""
        # The env_with_jobs fixture already simulates a booted env. We simulate the
        # decorator-import side by pushing a JobSpec into the registry, then call
        # reconcile_decorator_jobs directly (which is what bootstrap_environment does
        # internally).
        from ede.foundation.jobs.services.job_registry import JobSpec, JOB_REGISTRY

        JOB_REGISTRY.append(JobSpec(
            name="boot.smoke",
            target="ede.tests.foundation.jobs.targets.echo_target",
            kind="scheduled",
            cron="*/10 * * * *",
            module_key="foundation.jobs.demo",
        ))

        from ede.foundation.jobs.services.reconciler import reconcile_decorator_jobs
        reconcile_decorator_jobs(env_with_jobs)

        rows = env_with_jobs.models["ir.job"].search([("name", "=", "boot.smoke")])
        assert len(rows) == 1
        assert rows[0].source == "decorator"
        assert rows[0].cron == "*/10 * * * *"
        assert rows[0].next_run_at_utc is not None
    ```

- [ ] **Step 5.4: Run reconciler tests (now 8 PASS)**
    ```bash
    pytest src/tests/foundation/jobs/test_reconciler.py -v
    ```

- [ ] **Step 5.5: Boot smoke — confirm `ede info` still loads cleanly**
    ```bash
    ede info --config ede.conf
    ```
    Expected: prints loaded apps including `foundation.jobs`, no traceback. If `reconcile_decorator_jobs` fires during boot, you should see one INFO log line `reconciler: 0 inserted, 0 updated, 0 deactivated, 0 skipped` (because no decorators are declared yet in production — Slice 4 ships the first one).

---

## Task 6: XML data path — confirm it works + add regression test

**Files:**
- Create: `src/ede/foundation/jobs/data/test_jobs.xml` (test fixture — wired only into the test setup, NOT in the main manifest)
- Create: `src/tests/foundation/jobs/test_xml_data_path.py`

**Important:** the standard EDE data loader should handle `<record model="ir.job">` automatically — this task is verification, not new code. If the test fails because the data loader rejects the model or mis-handles a field, that's a bug to surface to the controller (NOT something this task tries to fix in the loader).

- [ ] **Step 6.1: Create the test fixture**

    Create `src/ede/foundation/jobs/data/test_jobs.xml`:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <openerp>
        <data noupdate="0">
            <record id="job_xml_test_fixture" model="ir.job">
                <field name="name">test.xml-data-path.fixture</field>
                <field name="module_key">foundation.jobs.test</field>
                <field name="target">src.tests.foundation.jobs.targets.echo_target</field>
                <field name="kind">scheduled</field>
                <field name="cron">*/30 * * * *</field>
                <field name="priority">5</field>
                <field name="retry_policy">fixed</field>
                <field name="retry_max_attempts">2</field>
                <field name="source">xml</field>
                <field name="description">Test fixture for XML data path coverage.</field>
            </record>
        </data>
    </openerp>
    ```

    **Do NOT add this file to `__manifest__.py`'s `data` list** — it's a test fixture, not production data. The test loads it explicitly.

- [ ] **Step 6.2: Write the failing test**

    Create `src/tests/foundation/jobs/test_xml_data_path.py`:
    ```python
    """Verify XML <record model="ir.job"> rows load via the standard EDE data loader.

    This is a regression test — it confirms the XML data path Slice 2 advertises
    works end-to-end. No new code should be needed in foundation.jobs services to
    make this pass; the data loader is shared infrastructure.
    """
    from pathlib import Path


    def test_xml_record_loads_into_ir_job_with_source_xml(env_with_jobs):
        """Load the test_jobs.xml fixture and verify the row lands with source=xml."""
        # Find the right data-loader entry point. Patterns to try (in order):
        #   1. env.data_loader.load_file(absolute_path)
        #   2. from ede.core.services.data_loader import load_xml_file; load_xml_file(env, path)
        #   3. from ede.foundation.base.services.data_loader import ...; some variant
        #
        # The implementer should grep `grep -rnE "def load_(file|xml|data)" src/ede/core/ src/ede/foundation/base/`
        # to find the canonical loader and adjust the call below.

        fixture_path = Path("src/ede/foundation/jobs/data/test_jobs.xml").resolve()
        from ede.core.services.data_loader.xml_loader import load_xml_file   # adjust if path differs
        load_xml_file(env_with_jobs, str(fixture_path))

        rows = env_with_jobs.models["ir.job"].search([("name", "=", "test.xml-data-path.fixture")])
        assert len(rows) == 1
        row = rows[0]
        assert row.source == "xml"
        assert row.cron == "*/30 * * * *"
        assert row.kind == "scheduled"
        assert row.target == "src.tests.foundation.jobs.targets.echo_target"
        assert row.module_key == "foundation.jobs.test"


    def test_reconciler_does_not_touch_xml_loaded_row(env_with_jobs):
        """After XML-loading a row, running the reconciler must NOT deactivate it."""
        from ede.foundation.jobs.services.job_registry import _clear_registry_for_tests
        from ede.foundation.jobs.services.reconciler import reconcile_decorator_jobs

        # Load the XML fixture first
        fixture_path = Path("src/ede/foundation/jobs/data/test_jobs.xml").resolve()
        from ede.core.services.data_loader.xml_loader import load_xml_file
        load_xml_file(env_with_jobs, str(fixture_path))

        # Now run the reconciler (decorator registry is empty — would normally deactivate everything)
        _clear_registry_for_tests()
        reconcile_decorator_jobs(env_with_jobs)

        row = env_with_jobs.models["ir.job"].search([("name", "=", "test.xml-data-path.fixture")])[0]
        assert row.active is True                    # XML row survives
        assert row.source == "xml"
    ```

- [ ] **Step 6.3: Run — likely needs an import path adjustment**

    ```bash
    pytest src/tests/foundation/jobs/test_xml_data_path.py -v
    ```

    First run will fail with ImportError on the `load_xml_file` path (the test guesses). Run:
    ```bash
    grep -rnE "def load_(file|xml|data)|def load_module_data" src/ede/core/ src/ede/foundation/base/ 2>/dev/null | head -20
    ```
    to find the actual loader function, then fix the import path in BOTH tests. Re-run. Expected: 2 PASS.

- [ ] **Step 6.4: Lint**
    ```bash
    ruff check src/tests/foundation/jobs/test_xml_data_path.py
    ```

---

## Task 7: `ede worker` scheduler thread integration

**Files:**
- Modify: `src/ede/cli/commands/worker.py` — add the `JobsScheduler` thread + `--no-jobs` flag + supervisor + graceful shutdown
- Test: add a smoke test confirming `--no-jobs` skips the scheduler init path (the live thread isn't tested in pytest)

- [ ] **Step 7.1: Read the existing `ede worker`**

    Open `src/ede/cli/commands/worker.py`. Slice 1 didn't touch it. The existing command spins up the event-queue drainer (`EventWorker`). You're adding:
    - A `--no-jobs` flag (default: jobs scheduler IS started).
    - A `JobsScheduler` instance + thread + supervisor logic (catch dead threads + exit 2 so the orchestrator restarts).
    - SIGTERM handler that calls `scheduler.stop()` and waits up to `JOBS_GRACEFUL_TIMEOUT_SECONDS` for the in-flight tick to finish.

- [ ] **Step 7.2: Implement**

    Roughly:
    ```python
    # src/ede/cli/commands/worker.py (additions, not full rewrite)

    import signal
    import sys
    import threading
    import time

    import click

    from ede.foundation.jobs.services.celery_app import build_celery_app
    from ede.foundation.jobs.services.executor import CeleryExecutor
    from ede.foundation.jobs.services.scheduler import JobsScheduler


    @click.option("--no-jobs", is_flag=True, default=False,
                  help="Skip the JobsScheduler thread (event-drainer-only mode).")
    def worker_command(..., no_jobs: bool, ...):
        ...
        # existing event-drainer setup ...

        scheduler_thread: threading.Thread | None = None
        scheduler: JobsScheduler | None = None
        if not no_jobs:
            settings = boot_output.foundation_settings_module
            celery_app = build_celery_app(settings)
            executor = CeleryExecutor(celery_app, default_queue=settings.JOBS_CELERY_DEFAULT_QUEUE)
            scheduler = JobsScheduler(executor=executor, tick_seconds=settings.JOBS_SCHEDULER_TICK_SECONDS)

            scheduler_thread = threading.Thread(
                target=scheduler.run_forever,
                args=(boot_output.environment,),
                name="jobs-scheduler",
                daemon=True,
            )
            scheduler_thread.start()

        # ── SIGTERM handler ─────────────────────────────────────────
        def _shutdown(signum, frame):
            if scheduler is not None:
                scheduler.stop()
            # existing event-drainer shutdown logic ...
            sys.exit(0)

        signal.signal(signal.SIGTERM, _shutdown)
        signal.signal(signal.SIGINT, _shutdown)

        # ── Supervisor — if scheduler thread dies unexpectedly, exit non-zero ───
        try:
            while True:
                time.sleep(5)
                if scheduler_thread is not None and not scheduler_thread.is_alive():
                    click.echo("CRITICAL: jobs-scheduler thread died — exiting", err=True)
                    sys.exit(2)
                # existing event-drainer health check (if any)
        except KeyboardInterrupt:
            _shutdown(None, None)
    ```

    Adapt to whatever the existing `worker_command` actually looks like — match its style, don't rewrite the event-drainer wiring. **Imports go at module top** (signal, sys, threading, time, the three foundation.jobs imports).

- [ ] **Step 7.3: Smoke-test that `ede worker --help` still works + the new flag is present**

    Create `src/tests/foundation/jobs/test_worker_cli_jobs_flag.py`:
    ```python
    """Smoke: confirm `ede worker --help` advertises the --no-jobs flag added in Slice 2."""
    from click.testing import CliRunner

    from ede.cli.commands.worker import worker_command


    def test_worker_help_includes_no_jobs_flag():
        runner = CliRunner()
        result = runner.invoke(worker_command, ["--help"])
        assert result.exit_code == 0
        assert "--no-jobs" in result.output
        assert "scheduler" in result.output.lower() or "jobs" in result.output.lower()
    ```

    Run:
    ```bash
    pytest src/tests/foundation/jobs/test_worker_cli_jobs_flag.py -v
    ```
    Expected: PASS.

- [ ] **Step 7.4: Lint**
    ```bash
    ruff check src/ede/cli/commands/worker.py src/tests/foundation/jobs/test_worker_cli_jobs_flag.py
    ```

- [ ] **Step 7.5: (Optional) Manual live smoke**

    With Redis running locally:
    ```bash
    ede worker --config ede.conf
    ```
    Expected:
    - Boot logs include `JobsScheduler starting (tick_seconds=10)`.
    - The existing event-drainer logs appear as before.
    - Ctrl-C exits cleanly with `JobsScheduler stopped`.

    Then:
    ```bash
    ede worker --config ede.conf --no-jobs
    ```
    Expected: NO `JobsScheduler starting` line; event-drainer-only output.

---

## Task 8: End-to-end smoke — decorator → boot → scheduler tick → Celery execution

**Files:**
- Create: `src/tests/foundation/jobs/test_scheduler_e2e.py`

- [ ] **Step 8.1: Write the e2e test**

    Create `src/tests/foundation/jobs/test_scheduler_e2e.py`:
    ```python
    """End-to-end smoke: decorator → reconciler → scheduler.tick → Celery eager → success.

    Validates the full Slice 2 path in one test.
    """
    from datetime import datetime, timedelta, timezone

    import pytest

    from ede.foundation.jobs.services.job_registry import (
        JOB_REGISTRY,
        JobSpec,
        _clear_registry_for_tests,
    )
    from ede.foundation.jobs.services.reconciler import reconcile_decorator_jobs
    from ede.foundation.jobs.services.scheduler import JobsScheduler


    @pytest.fixture(autouse=True)
    def reset_registry():
        _clear_registry_for_tests()
        yield
        _clear_registry_for_tests()


    def test_decorated_job_reconciles_and_scheduler_dispatches_through_celery(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor

        # Simulate the decorator import — push the spec onto the registry
        JOB_REGISTRY.append(JobSpec(
            name="e2e.scheduler.echo",
            target="src.tests.foundation.jobs.targets.echo_target",
            kind="scheduled",
            cron="*/1 * * * *",
            module_key="foundation.jobs.demo",
            description="e2e smoke",
        ))

        # Reconciler creates the ir.job row (source=decorator, with next_run_at)
        reconcile_decorator_jobs(env)
        job = env.models["ir.job"].search([("name", "=", "e2e.scheduler.echo")])[0]
        assert job.source == "decorator"

        # Force the job due NOW (reconciler set next_run_at to a future cron firing)
        job.write({"next_run_at_utc": datetime.now(tz=timezone.utc) - timedelta(seconds=30)})

        # Scheduler tick dispatches through the eager Celery executor
        scheduler = JobsScheduler(executor=env._jobs_executor, tick_seconds=10)
        dispatched = scheduler.tick(env)
        assert dispatched == 1

        # Find the freshly-created run — eager mode means it ran synchronously
        runs = env.models["ir.job.run"].search([("job_id", "=", job.id)])
        assert len(runs) == 1
        run = runs[0]
        assert run.status == "success", (
            f"expected success, got {run.status} (err={run.error_summary})"
        )
        assert run.output == {"echoed": {}}          # echo_target returns {"echoed": payload}
        assert run.celery_task_id is not None
        assert run.worker_id is not None

        # next_run_at advanced
        refreshed = env.models["ir.job"].browse(job.id)
        assert refreshed.next_run_at_utc > datetime.now(tz=timezone.utc)
        assert refreshed.last_run_at_utc is not None
    ```

- [ ] **Step 8.2: Run — expect PASS**
    ```bash
    pytest src/tests/foundation/jobs/test_scheduler_e2e.py -v
    ```

- [ ] **Step 8.3: Lint**
    ```bash
    ruff check src/tests/foundation/jobs/test_scheduler_e2e.py
    ```

---

## Task 9: Slice 2 Acceptance Gate

- [ ] **Step A1: Full jobs suite green**
    ```bash
    pytest src/tests/foundation/jobs/ -v
    ```
    Expected: previous 15 + new (~25-28: 5 cron + 6 scheduler + 6 decorators + 8 reconciler + 2 xml + 1 worker-cli + 1 e2e) — total ~38-43 jobs tests, ALL passing.

- [ ] **Step A2: Full repo no regressions**
    ```bash
    ./run_tests.sh
    ```
    Expected: prior baseline + ~25 new = new total, all green (modulo any pre-existing unrelated failures the controller has noted).

- [ ] **Step A3: Boot smoke**
    ```bash
    ede info --config ede.conf
    ```
    Expected: foundation.jobs listed; one INFO log line from the reconciler showing `0 inserted, 0 updated, 0 deactivated, 0 skipped` (no decorators in production yet — that ships in Slice 4).

- [ ] **Step A4: Worker boots with scheduler thread**
    Manual one-liner (if Redis is up):
    ```bash
    ede worker --config ede.conf
    ```
    Confirm `JobsScheduler starting` appears in the logs, then Ctrl-C.

- [ ] **Step A5: `--no-jobs` opt-out**
    ```bash
    ede worker --config ede.conf --no-jobs
    ```
    Confirm the `JobsScheduler starting` line does NOT appear.

- [ ] **Step A6: Pause for `commit` instruction** (CLAUDE.md hard rule).

---

## Status flip for Slice 2

When all acceptance steps green, Slice 2 keeps Phase 1 at 🟡 (Slices 3 + 4 still pending). Update:

- `roadmap/foundation/jobs/README.md` — top Status note ("Slices 1-2 ✅; Slices 3-4 remain")
- `roadmap/foundation/jobs/phase-1-implementation.md` — top Status note
- `roadmap/roadmap-tracker.md` — Overall + Phase 1 row + Last refreshed (include the Slice 2 changelog)
- `docs/foundation-jobs.md` — Status header + Built Capabilities table (add Slice 2 row)

Do not advance to ✅ — that's reserved for end of Slice 4 (the heartbeat walkthrough).

---

## Self-Review

**1. Spec coverage:**
- ✅ Cron helper (Task 1)
- ✅ `JobsScheduler` with `tick` + `run_forever` (Task 2)
- ✅ `@api.scheduled_job` / `@api.background_job` (Task 3)
- ✅ Boot reconciler — source-aware + module_key scoping (Task 4)
- ✅ Loader hookup (Task 5)
- ✅ XML data path verification (Task 6)
- ✅ `ede worker` scheduler thread + `--no-jobs` + supervisor (Task 7)
- ✅ End-to-end smoke (Task 8)
- ✅ Acceptance gate (Task 9)

Out-of-scope items (Slices 3 + 4) explicitly listed at the top of the plan.

**2. Placeholder scan:**
- One "TBD-style" callout in Task 4 about `like`/`startswith` ORM operator support — but it's a real "verify the API before pasting" instruction with a fallback explicitly written out, not a vague TODO. Same in Task 6 for the XML loader import path. Both are intentional verification points, not placeholders.

**3. Type consistency:**
- `JobSpec` dataclass fields match the kwargs of `scheduled_job` / `background_job` and the row values written by the reconciler.
- `reconcile_decorator_jobs(env, *, module_keys=None)` signature matches in Task 4 (declaration), Task 5 (loader call site), and Task 8 (e2e test).
- `JobsScheduler(executor=..., tick_seconds=...).tick(env) -> int` matches across Tasks 2, 7 (worker integration), and 8 (e2e).

---

## Execution Handoff

**Plan complete and saved to** `docs/superpowers/plans/2026-05-18-foundation-jobs-phase-1-slice-2.md`.

Two execution options:

**1. Subagent-Driven (recommended)** — dispatch a fresh subagent per task, review between tasks, fast iteration. Same pattern that delivered Slice 1.

**2. Inline Execution** — execute tasks in this session, batch with checkpoints.

Which approach?
