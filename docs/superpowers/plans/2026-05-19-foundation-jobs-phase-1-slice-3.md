# Foundation.jobs Phase 1 — Slice 3: Retry Policy + Dead-Letter + Progress Reporting

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the foundation.jobs engine production-resilient. Ship four retry policies (none/fixed/exponential/linear with ±20% jitter), the dead-letter terminal state with `notification.send` dispatch on exhausted retries, and `env.job_progress(pct, message)` thread-local plumbing for live progress reporting from inside long-running job targets.

**Architecture:** A new `services/retry_policy.py` computes the next-attempt delay as a pure function of `(policy, attempt_number, base_seconds)`. The existing `execute_run` Celery task wrapper extends its `except` branch to consult the policy: if attempts remain, INSERT a child `ir.job.run` with `parent_run_id` set + `attempt_number = parent+1` and re-dispatch via `self.apply_async(args=[new_run_id], eta=now+delay, queue=...)`; if exhausted, mark the run `dead_letter`, emit `ir.job.run.dead_letter`, and dispatch `notification.send` (loose-coupled — only fires if the command is registered). A new `services/progress.py` owns a `ContextVar`-backed thread-local; the task wrapper sets it before invoking the target callable and clears it in `finally`; `env.job_progress(percent, message)` reads the context var and writes through to `ir.job.run.progress_pct` + `progress_message`, no-op outside a job context.

**Tech Stack:** Python 3.10+ (stdlib `contextvars`, `random` for jitter, `datetime` + `timedelta` for eta math), Celery 5.4+ (`apply_async(eta=...)` honors eta in eager mode it ignores eta and fires immediately — convenient for tests), existing Slice 1/2 surface (`CeleryExecutor`, `JobsScheduler`, `JOB_REGISTRY`, `reconciler`), existing `foundation.notifications` (`notification.send` command — Slice 3 only consumes; no dependency on its module-load order).

**Does NOT cover (Slice 4):**
- Settings → Technical → Jobs admin UI (dashboard / definitions / run history / dead-letter recovery)
- RBAC seed (`jobs.admin` / `jobs.operator` / `jobs.viewer`)
- `data/notification_templates.xml` (the actual stuck-job + dead-letter email/web/in-app templates)
- `data/example_jobs.xml` (XML-declared heartbeat first-adopter)
- User-walkthrough acceptance gate → Phase 1 ✅ flip → OneMaster Phase 1 unblock

---

## File Structure

### New files

| Path | Responsibility |
|---|---|
| `src/ede/foundation/jobs/services/retry_policy.py` | `compute_retry_delay_seconds(policy, attempt_number, base_seconds, *, jitter_pct=0.2) -> int`; pure function, no I/O, no env access |
| `src/ede/foundation/jobs/services/progress.py` | `ContextVar`-backed thread-local: `set_progress_context(run_id)`, `clear_progress_context()`, `write_progress(env, percent, message)` |
| `src/tests/foundation/jobs/test_retry_policy.py` | 6 tests — each policy + jitter range + boundary |
| `src/tests/foundation/jobs/test_progress.py` | 4 tests — no-op outside context, set/clear, write-through to row, throttle-friendly (caller-controlled) |
| `src/tests/foundation/jobs/test_dead_letter.py` | 4 tests — exhausted retries marks `dead_letter`, emits event, dispatches notification, graceful when notification.send unregistered |
| `src/tests/foundation/jobs/test_retry_chain_e2e.py` | 2 tests — full retry chain via `parent_run_id`; eventual-success path |

### Existing files modified

| Path | Change |
|---|---|
| `src/ede/foundation/jobs/services/task_wrapper.py` | Replace flat `except` branch with `_handle_failure(env, run, exc, self)` that consults retry policy, creates child run or marks dead_letter. Wrap target call in `set_progress_context(run.id)` / `clear_progress_context()`. |
| `src/ede/core/env.py` | Add `env.job_progress(percent, message=None) -> None` method — reads progress context, writes through to `ir.job.run.progress_pct` + `progress_message`, no-op when no context. |

---

## Pre-flight

- [ ] **P1: Confirm Slice 2 is at HEAD.**
    ```bash
    git log --oneline -1 src/ede/foundation/jobs/services/reconciler.py
    ```
    Expected: shows commit `f445e3f` (Slice 2) or a later jobs-touching commit. If absent, Slice 2 wasn't merged — stop and fix.

- [ ] **P2: Confirm the existing jobs suite is green.**
    ```bash
    pytest src/tests/foundation/jobs/ -v
    ```
    Expected: 44 passed in ~1-3s. If anything is red, fix before touching Slice 3.

- [ ] **P3: Read the existing task_wrapper to know the current shape.**
    ```bash
    sed -n '1,120p' src/ede/foundation/jobs/services/task_wrapper.py
    ```
    Note: `register_execute_run_task(celery_app, bootstrap_env_fn) -> Task`. The task body is `def execute_run(self, run_id): ...`. `self` is the bound Celery Task instance; `self.app` is the Celery app; `self.request.id` is the current Celery task id; `self.apply_async(args=[...], eta=..., queue=...)` re-dispatches the same task with new args.

---

## Task 1: Retry policy helper

**Files:**
- Create: `src/ede/foundation/jobs/services/retry_policy.py`
- Test: `src/tests/foundation/jobs/test_retry_policy.py`

- [ ] **Step 1.1: Failing test first**

    Create `src/tests/foundation/jobs/test_retry_policy.py`:
    ```python
    """Tests for the retry policy helper — pure delay math."""
    import pytest

    from ede.foundation.jobs.services.retry_policy import (
        compute_retry_delay_seconds,
        InvalidRetryPolicy,
        RETRY_POLICIES,
    )


    def test_policy_none_signals_no_retry():
        """policy='none' returns None (caller treats as 'do not retry')."""
        assert compute_retry_delay_seconds(policy="none", attempt_number=1, base_seconds=60) is None


    def test_policy_fixed_returns_base_seconds():
        """policy='fixed' returns base_seconds regardless of attempt_number."""
        assert compute_retry_delay_seconds(policy="fixed", attempt_number=1, base_seconds=60) == 60
        assert compute_retry_delay_seconds(policy="fixed", attempt_number=5, base_seconds=60) == 60
        assert compute_retry_delay_seconds(policy="fixed", attempt_number=99, base_seconds=30) == 30


    def test_policy_linear_scales_with_attempt():
        """policy='linear' returns base × attempt."""
        assert compute_retry_delay_seconds(policy="linear", attempt_number=1, base_seconds=60) == 60
        assert compute_retry_delay_seconds(policy="linear", attempt_number=2, base_seconds=60) == 120
        assert compute_retry_delay_seconds(policy="linear", attempt_number=3, base_seconds=60) == 180


    def test_policy_exponential_doubles_each_attempt_within_jitter():
        """policy='exponential' returns base × 2^(attempt-1), with ±jitter_pct random jitter."""
        # attempt=1: base × 2^0 = 60, ±20% → [48, 72]
        for _ in range(20):
            d = compute_retry_delay_seconds(policy="exponential", attempt_number=1, base_seconds=60)
            assert 48 <= d <= 72, f"attempt=1 delay {d} outside [48, 72]"
        # attempt=3: base × 2^2 = 240, ±20% → [192, 288]
        for _ in range(20):
            d = compute_retry_delay_seconds(policy="exponential", attempt_number=3, base_seconds=60)
            assert 192 <= d <= 288, f"attempt=3 delay {d} outside [192, 288]"


    def test_policy_exponential_zero_jitter_is_deterministic():
        """When jitter_pct=0, exponential math is exact."""
        assert compute_retry_delay_seconds(
            policy="exponential", attempt_number=1, base_seconds=60, jitter_pct=0
        ) == 60
        assert compute_retry_delay_seconds(
            policy="exponential", attempt_number=2, base_seconds=60, jitter_pct=0
        ) == 120
        assert compute_retry_delay_seconds(
            policy="exponential", attempt_number=4, base_seconds=60, jitter_pct=0
        ) == 480


    def test_invalid_policy_raises():
        with pytest.raises(InvalidRetryPolicy):
            compute_retry_delay_seconds(policy="quadratic", attempt_number=1, base_seconds=60)


    def test_retry_policies_constant_lists_four_supported_policies():
        assert set(RETRY_POLICIES) == {"none", "fixed", "exponential", "linear"}
    ```

- [ ] **Step 1.2: Run — expect ModuleNotFoundError**
    ```bash
    pytest src/tests/foundation/jobs/test_retry_policy.py -v
    ```

- [ ] **Step 1.3: Write the helper**

    Create `src/ede/foundation/jobs/services/retry_policy.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Retry policy — pure delay math.

    Four policies supported on ir.job.retry_policy. The execute_run task
    wrapper calls compute_retry_delay_seconds(policy, attempt_number,
    base_seconds) when a job target raises. None signals "do not retry"
    (the caller marks the run dead_letter immediately); any integer
    return value is the seconds to wait before the next attempt fires.
    """
    from __future__ import annotations

    import random
    from typing import Optional


    RETRY_POLICIES = ("none", "fixed", "exponential", "linear")


    class InvalidRetryPolicy(ValueError):
        """Raised when an unknown policy string is passed to compute_retry_delay_seconds."""


    def compute_retry_delay_seconds(
        *,
        policy: str,
        attempt_number: int,
        base_seconds: int,
        jitter_pct: float = 0.2,
    ) -> Optional[int]:
        """Return seconds-until-next-attempt for the given policy.

        Returns None when policy='none' (caller MUST treat as "do not retry").
        Returns an integer >= 0 for all other policies.

        attempt_number is the CURRENT (just-failed) attempt's number — the
        helper returns the delay to use BEFORE the next attempt fires.

        jitter_pct only affects 'exponential'. Set to 0 in tests for
        deterministic delays.
        """
        if policy not in RETRY_POLICIES:
            raise InvalidRetryPolicy(f"unknown retry policy {policy!r}; expected one of {RETRY_POLICIES}")

        if policy == "none":
            return None
        if policy == "fixed":
            return int(base_seconds)
        if policy == "linear":
            return int(base_seconds * attempt_number)
        # exponential
        raw = base_seconds * (2 ** (attempt_number - 1))
        if jitter_pct <= 0:
            return int(raw)
        delta = raw * jitter_pct
        return int(raw + random.uniform(-delta, delta))
    ```

- [ ] **Step 1.4: Run — expect 7 PASS**
    ```bash
    pytest src/tests/foundation/jobs/test_retry_policy.py -v
    ```

- [ ] **Step 1.5: Lint**
    ```bash
    ruff check src/ede/foundation/jobs/services/retry_policy.py src/tests/foundation/jobs/test_retry_policy.py
    ```
    Expected: clean.

---

## Task 2: Progress reporting (thread-local + env method)

**Files:**
- Create: `src/ede/foundation/jobs/services/progress.py`
- Modify: `src/ede/core/env.py` — add `env.job_progress(...)` method
- Test: `src/tests/foundation/jobs/test_progress.py`

- [ ] **Step 2.1: Failing tests first**

    Create `src/tests/foundation/jobs/test_progress.py`:
    ```python
    """Tests for env.job_progress + the progress thread-local."""
    import pytest

    from ede.foundation.jobs.services.progress import (
        get_current_run_id,
        set_progress_context,
        clear_progress_context,
    )


    def test_no_context_returns_none():
        clear_progress_context()       # paranoia
        assert get_current_run_id() is None


    def test_set_then_get():
        set_progress_context("run-uuid-abc")
        try:
            assert get_current_run_id() == "run-uuid-abc"
        finally:
            clear_progress_context()


    def test_clear_resets_to_none():
        set_progress_context("run-uuid-xyz")
        clear_progress_context()
        assert get_current_run_id() is None


    def test_env_job_progress_no_op_outside_context(env_with_jobs_and_executor):
        """Calling env.job_progress() outside a job task body is safe (no-op)."""
        env = env_with_jobs_and_executor
        clear_progress_context()
        # No exception, no DB write
        env.job_progress(percent=42.5, message="should be ignored")


    def test_env_job_progress_writes_through_in_context(env_with_jobs_and_executor):
        """When set_progress_context is active, env.job_progress writes to ir.job.run."""
        env = env_with_jobs_and_executor

        # Set up an ir.job + ir.job.run we can target
        env.models["ir.job"].create({
            "name": "progress-test.job",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
        })
        job = env.models["ir.job"].search([("name", "=", "progress-test.job")])[0]
        run = env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "running",
            "payload": {},
        })

        set_progress_context(run.id)
        try:
            env.job_progress(percent=37.5, message="upserted 12000/32000")
        finally:
            clear_progress_context()

        refreshed = env.models["ir.job.run"].browse(run.id)
        # Decimal field may come back as Decimal or float — coerce
        assert float(refreshed.progress_pct) == 37.5
        assert refreshed.progress_message == "upserted 12000/32000"
    ```

- [ ] **Step 2.2: Run — expect ModuleNotFoundError**
    ```bash
    pytest src/tests/foundation/jobs/test_progress.py -v
    ```

- [ ] **Step 2.3: Write the progress helper**

    Create `src/ede/foundation/jobs/services/progress.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Progress reporting thread-local + write-through helper.

    The execute_run task wrapper sets the context before invoking the
    target callable and clears it after (in finally). Inside the target,
    env.job_progress(percent, message) reads the context and writes to
    the row. Outside any job context the call is a safe no-op so the
    target callable can be invoked both inside and outside a worker
    without conditional code.
    """
    from __future__ import annotations

    from contextvars import ContextVar
    from typing import Optional


    _CURRENT_RUN_ID: ContextVar[Optional[str]] = ContextVar(
        "ede.foundation.jobs.current_run_id", default=None
    )


    def set_progress_context(run_id: str) -> None:
        """Bind the current task's ir.job.run UUID to the context-local slot."""
        _CURRENT_RUN_ID.set(run_id)


    def clear_progress_context() -> None:
        """Reset the context-local slot. Always called in the task's finally block."""
        _CURRENT_RUN_ID.set(None)


    def get_current_run_id() -> Optional[str]:
        """Return the currently-bound ir.job.run UUID, or None if outside a job."""
        return _CURRENT_RUN_ID.get()


    def write_progress(env, *, percent: float, message: Optional[str] = None) -> None:
        """Write percent + message to the currently-bound ir.job.run row.

        No-op when no context is bound (safe to call from any code path).
        Each call is one UPDATE; callers throttle their own frequency.
        """
        run_id = _CURRENT_RUN_ID.get()
        if run_id is None:
            return
        run_proxy = env.models["ir.job.run"]
        run = run_proxy.browse(run_id)
        values = {"progress_pct": float(percent)}
        if message is not None:
            values["progress_message"] = str(message)[:255]
        run.write(values)
    ```

- [ ] **Step 2.4: Add `env.job_progress` to `Env`**

    Open `src/ede/core/env.py`. Locate the existing `enqueue_job` method (added in Slice 1). Add immediately after it:

    ```python
    def job_progress(self, *, percent: float, message: str | None = None) -> None:
        """Report progress from inside a job target callable.

        No-op when called outside a job context (e.g. during a unit test
        or from a CRUD handler). When inside a job, writes
        percent + message to the currently-bound ir.job.run row.
        """
        from ede.foundation.jobs.services.progress import write_progress
        write_progress(self, percent=percent, message=message)
    ```

    **NOTE on the inline import:** this is the legal CLAUDE.md exception for "optional-SDK guarded inside a method body" — `ede.foundation.jobs` is loaded only when `ACTIVE_MODULES` includes `"jobs"`, and `ede.core.env` is core (always loaded). The inline import keeps `ede.core.env` independent of `ede.foundation.jobs` at import time. **However**, in our codebase `foundation.jobs` is already in the default `ACTIVE_MODULES`, so a top-level `try: ... except ImportError: ...` is cleaner. Use this pattern instead:

    ```python
    # At the top of env.py with other imports:
    try:
        from ede.foundation.jobs.services.progress import write_progress as _write_job_progress
    except ImportError:                                            # foundation.jobs not loaded
        _write_job_progress = None


    # On the Env class:
    def job_progress(self, *, percent: float, message: str | None = None) -> None:
        """Report progress from inside a job target callable.

        No-op outside a job context.
        """
        if _write_job_progress is None:
            return
        _write_job_progress(self, percent=percent, message=message)
    ```

    Verify the import succeeds:
    ```bash
    python -c "from ede.core.env import Env; print(hasattr(Env, 'job_progress'))"
    ```
    Expected: `True`.

- [ ] **Step 2.5: Run — expect 5 PASS**
    ```bash
    pytest src/tests/foundation/jobs/test_progress.py -v
    ```

- [ ] **Step 2.6: Lint**
    ```bash
    ruff check src/ede/foundation/jobs/services/progress.py src/ede/core/env.py src/tests/foundation/jobs/test_progress.py
    ```

---

## Task 3: Task wrapper — wire retry-on-failure into execute_run

**Files:**
- Modify: `src/ede/foundation/jobs/services/task_wrapper.py` — replace flat `except` with retry-aware branch

- [ ] **Step 3.1: Read the current state of `task_wrapper.py`**

    ```bash
    sed -n '1,120p' src/ede/foundation/jobs/services/task_wrapper.py
    ```
    Confirm the current `except Exception as exc:` block writes `status="failed"` + emits `ir.job.run.failed` + releases lock.

- [ ] **Step 3.2: Add module-top imports + helper functions**

    Add to the top imports block of `task_wrapper.py` (alongside existing imports):
    ```python
    from datetime import timedelta

    from .retry_policy import compute_retry_delay_seconds
    ```

    Add these module-level helpers (above `register_execute_run_task`):
    ```python
    def _queue_for_priority(priority: int, default_queue: str = "ede.jobs.default") -> str:
        """Map ir.job.priority (0-9) to per-priority queue name. Fallback to default."""
        if isinstance(priority, int) and 0 <= priority <= 9:
            return f"ede.jobs.p{priority}"
        return default_queue


    def _schedule_retry_attempt(env, parent_run, task, *, delay_seconds: int):
        """Create a child ir.job.run + apply_async with eta = now + delay_seconds.

        parent_run.attempt_number must be < parent_run.job_id.retry_max_attempts
        (the caller has already gated). Returns the new child run row.
        """
        job = parent_run.job_id
        run_proxy = env.models["ir.job.run"]
        eta = _now_utc() + timedelta(seconds=delay_seconds)

        new_run = run_proxy.create({
            "job_id": job.id,
            "attempt_number": parent_run.attempt_number + 1,
            "parent_run_id": parent_run.id,
            "status": "pending",
            "payload": parent_run.payload,
            "queued_at_utc": _now_utc(),
        })

        queue = _queue_for_priority(job.priority)
        async_result = task.apply_async(args=[new_run.id], eta=eta, queue=queue)
        new_run.write({"celery_task_id": async_result.id})
        return new_run
    ```

- [ ] **Step 3.3: Replace the failure branch in `execute_run`**

    Locate the existing `except Exception as exc:` block inside the inner `execute_run` task. Replace it with:
    ```python
        except Exception as exc:                                # noqa: BLE001 — catch-all
            duration = int((_now_utc() - started).total_seconds())
            error_summary = f"{type(exc).__name__}: {exc}"
            error_traceback = traceback.format_exc()

            # Decide: retry or dead-letter?
            delay = compute_retry_delay_seconds(
                policy=job.retry_policy,
                attempt_number=run.attempt_number,
                base_seconds=job.retry_base_seconds,
            )
            attempts_remaining = (
                delay is not None
                and run.attempt_number < job.retry_max_attempts
            )

            if attempts_remaining:
                run.write({
                    "status": "failed",                          # intermediate failure
                    "finished_at_utc": _now_utc(),
                    "error_summary": error_summary,
                    "error_traceback": error_traceback,
                    "duration_seconds": duration,
                })
                env.emit("ir.job.run.failed", {
                    "run_id": run_id,
                    "job_name": job.name,
                    "attempt_number": run.attempt_number,
                    "will_retry": True,
                })
                _schedule_retry_attempt(env, run, self, delay_seconds=delay)
            else:
                # Dead-letter — retries exhausted or policy=none
                run.write({
                    "status": "dead_letter",
                    "finished_at_utc": _now_utc(),
                    "error_summary": error_summary,
                    "error_traceback": error_traceback,
                    "duration_seconds": duration,
                })
                env.emit("ir.job.run.dead_letter", {
                    "run_id": run_id,
                    "job_name": job.name,
                    "attempt_number": run.attempt_number,
                    "error_summary": error_summary,
                })
                _dispatch_dead_letter_notification(env, run, job)
    ```

    Add a stub `_dispatch_dead_letter_notification` function — full implementation lands in Task 4. For now:
    ```python
    def _dispatch_dead_letter_notification(env, run, job) -> None:
        """Loose-coupled notification dispatch on dead-letter — full body in Task 4."""
        return  # populated in Task 4
    ```

- [ ] **Step 3.4: Run the existing jobs suite — confirm no regressions**
    ```bash
    pytest src/tests/foundation/jobs/ -v
    ```
    Expected: 44 + 7 + 5 = 56 tests passing (Slice 1+2 existing + Task 1 retry_policy + Task 2 progress). The existing `test_executor_eager.py::test_enqueue_runs_target_and_records_failure` test currently expects `status="failed"` for `raise_target` with `retry_policy="exponential"` (the default). With Slice 3's retry kicking in, the target will be retried → expect `status="dead_letter"` after exhaustion OR `status="failed"` mid-chain (depending on the test's job config).

    Read the test:
    ```bash
    grep -A 20 "test_enqueue_runs_target_and_records_failure" src/tests/foundation/jobs/test_executor_eager.py
    ```

    If the test's `ir.job` row uses the default `retry_policy="exponential"` + `retry_max_attempts=3`, the target will be retried 3 times in eager mode (since eager ignores eta) → final attempt's status will be `dead_letter` not `failed`. **Update the test's assertion** to expect `status == "dead_letter"` AND verify the retry chain via `parent_run_id`. OR change the test's ir.job to use `retry_policy="none"` so no retry fires (still tests the immediate-failure path).

    Pick the simpler option (`retry_policy="none"`) for `test_enqueue_runs_target_and_records_failure`. The full retry chain gets its own dedicated test in Task 6.

- [ ] **Step 3.5: Lint**
    ```bash
    ruff check src/ede/foundation/jobs/services/task_wrapper.py
    ```

---

## Task 4: Dead-letter notification dispatch (loose-coupled)

**Files:**
- Modify: `src/ede/foundation/jobs/services/task_wrapper.py` — fill in `_dispatch_dead_letter_notification`
- Test: `src/tests/foundation/jobs/test_dead_letter.py`

- [ ] **Step 4.1: Failing tests first**

    Create `src/tests/foundation/jobs/test_dead_letter.py`:
    ```python
    """Tests for the dead-letter terminal state + notification dispatch."""
    from unittest.mock import MagicMock

    import pytest

    from ede.foundation.jobs.services.job_registry import _clear_registry_for_tests


    @pytest.fixture(autouse=True)
    def reset_registry():
        _clear_registry_for_tests()
        yield
        _clear_registry_for_tests()


    def test_dead_letter_marks_run_status_dead_letter(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor

        env.models["ir.job"].create({
            "name": "dl-test.always-fails",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.raise_target",
            "kind": "queued",
            "source": "runtime",
            "retry_policy": "none",         # immediate dead-letter
            "retry_max_attempts": 1,
        })

        run = env.enqueue_job(
            target="src.tests.foundation.jobs.targets.raise_target",
            payload={},
        )

        run_after = env.models["ir.job.run"].browse(run.id)
        assert run_after.status == "dead_letter"
        assert "intentional failure" in (run_after.error_summary or "")
        assert run_after.error_traceback is not None
        assert run_after.finished_at_utc is not None


    def test_dead_letter_emits_event(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor
        events_captured = []
        # Patch the env's emit method to capture dead_letter events.
        original_emit = env.emit

        def _capturing_emit(event_name, payload):
            if event_name == "ir.job.run.dead_letter":
                events_captured.append((event_name, dict(payload)))
            return original_emit(event_name, payload)

        env.emit = _capturing_emit

        try:
            env.models["ir.job"].create({
                "name": "dl-test.event",
                "module_key": "foundation.jobs",
                "target": "src.tests.foundation.jobs.targets.raise_target",
                "kind": "queued",
                "source": "runtime",
                "retry_policy": "none",
            })
            env.enqueue_job(
                target="src.tests.foundation.jobs.targets.raise_target",
                payload={},
            )
        finally:
            env.emit = original_emit

        assert len(events_captured) == 1
        name, payload = events_captured[0]
        assert name == "ir.job.run.dead_letter"
        assert payload["job_name"] == "dl-test.event"
        assert "intentional failure" in payload["error_summary"]


    def test_dead_letter_dispatches_notification_when_registered(env_with_jobs_and_executor):
        env = env_with_jobs_and_executor
        dispatch_log = []

        # Register a stub notification.send handler on the env's command bus
        original_dispatch = env.dispatch

        def _stub_dispatch(cmd):
            if cmd.name == "notification.send":
                dispatch_log.append(dict(cmd.payload))
                return {"ok": True}
            return original_dispatch(cmd)

        env.dispatch = _stub_dispatch

        try:
            env.models["ir.job"].create({
                "name": "dl-test.notification",
                "module_key": "foundation.jobs",
                "target": "src.tests.foundation.jobs.targets.raise_target",
                "kind": "queued",
                "source": "runtime",
                "retry_policy": "none",
            })
            env.enqueue_job(
                target="src.tests.foundation.jobs.targets.raise_target",
                payload={},
            )
        finally:
            env.dispatch = original_dispatch

        assert len(dispatch_log) == 1
        sent = dispatch_log[0]
        assert sent["event_key"] == "ir.job.run.dead_letter"
        assert "dl-test.notification" in str(sent)


    def test_dead_letter_dispatch_silent_when_notification_unregistered(env_with_jobs_and_executor):
        """If notification.send is not registered, the dispatch attempt logs but doesn't crash."""
        env = env_with_jobs_and_executor
        # The test env doesn't register notification.send — proves graceful degradation.

        env.models["ir.job"].create({
            "name": "dl-test.no-notif",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.raise_target",
            "kind": "queued",
            "source": "runtime",
            "retry_policy": "none",
        })

        # Should not raise
        run = env.enqueue_job(
            target="src.tests.foundation.jobs.targets.raise_target",
            payload={},
        )
        run_after = env.models["ir.job.run"].browse(run.id)
        assert run_after.status == "dead_letter"
    ```

- [ ] **Step 4.2: Run — expect failure (dispatch is currently a no-op stub)**
    ```bash
    pytest src/tests/foundation/jobs/test_dead_letter.py -v
    ```

- [ ] **Step 4.3: Fill in `_dispatch_dead_letter_notification`**

    Replace the stub in `src/ede/foundation/jobs/services/task_wrapper.py` with:
    ```python
    def _dispatch_dead_letter_notification(env, run, job) -> None:
        """Loose-coupled notification dispatch on dead-letter.

        Dispatches a notification.send command if the command is registered
        on the env's CommandBus. If notification.send is not registered
        (foundation.notifications not loaded, or this is a unit test
        without the engine), logs a warning and returns — never raises.
        """
        from ede.core.types import Command                      # local to dodge core/env circular at import time

        payload = {
            "event_key": "ir.job.run.dead_letter",
            "subject": f"Job '{job.name}' dead-lettered after {run.attempt_number} attempts",
            "body": (
                f"Job {job.name!r} (target={job.target}) exhausted retry attempts.\n"
                f"Final attempt #{run.attempt_number} status: dead_letter.\n"
                f"Error: {run.error_summary or '(unknown)'}\n"
                f"Run ID: {run.id}\n"
            ),
            "module_key": job.module_key,
            "context": {
                "job_id": job.id,
                "run_id": run.id,
                "attempt_number": run.attempt_number,
                "error_summary": run.error_summary,
            },
        }

        try:
            env.dispatch(Command(name="notification.send", payload=payload))
        except Exception as exc:                                # noqa: BLE001
            logger.warning(
                "dead-letter notification dispatch failed for job %r run %r: %s",
                job.name, run.id, exc,
            )
    ```

    **Note:** the `from ede.core.types import Command` lives inside the function body — this is intentional because `task_wrapper.py` is loaded at boot via the foundation.jobs registry, and `ede.core.types` is shallow / cheap to re-import. **However**, CLAUDE.md disallows inline imports inside functions except for optional-SDK guards. Move it to the module top instead:
    ```python
    # At the top of task_wrapper.py:
    from ede.core.types import Command
    ```
    and drop the inline line. Verify `task_wrapper.py` already imports from `ede.core.*` elsewhere (it imports `release_lock` from `.lock` which is `ede.foundation.jobs.services.lock` — adjacent, not core). Add the `from ede.core.types import Command` at module top.

- [ ] **Step 4.4: Run dead-letter tests — expect 4 PASS**
    ```bash
    pytest src/tests/foundation/jobs/test_dead_letter.py -v
    ```

- [ ] **Step 4.5: Run full jobs suite**
    ```bash
    pytest src/tests/foundation/jobs/ -v
    ```
    Expected: previous count + 4 = ~60 tests passing.

- [ ] **Step 4.6: Lint**
    ```bash
    ruff check src/ede/foundation/jobs/services/task_wrapper.py src/tests/foundation/jobs/test_dead_letter.py
    ```

---

## Task 5: Wire progress thread-local into the task wrapper

**Files:**
- Modify: `src/ede/foundation/jobs/services/task_wrapper.py` — call set/clear around target invocation

- [ ] **Step 5.1: Failing test first (add to test_progress.py)**

    Append to `src/tests/foundation/jobs/test_progress.py`:
    ```python
    def test_progress_context_set_during_task_execution(env_with_jobs_and_executor):
        """During execute_run, env.job_progress() writes to ir.job.run.progress_pct.

        This is the integration test — uses a real target that calls
        env.job_progress mid-execution.
        """
        env = env_with_jobs_and_executor

        env.models["ir.job"].create({
            "name": "progress-mid-task.test",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.progress_reporting_target",
            "kind": "queued",
            "source": "runtime",
            "retry_policy": "none",
        })

        run = env.enqueue_job(
            target="src.tests.foundation.jobs.targets.progress_reporting_target",
            payload={"total": 4},
        )

        # The target ran in eager mode → terminal state
        run_after = env.models["ir.job.run"].browse(run.id)
        assert run_after.status == "success", (
            f"expected success, got {run_after.status} err={run_after.error_summary}"
        )
        # The target reported progress 4 times (one per loop iteration). Last
        # write wins — we check the final value matches what the target ended on.
        assert float(run_after.progress_pct) == 100.0
        assert run_after.progress_message == "done"


    def test_progress_context_cleared_after_task_execution(env_with_jobs_and_executor):
        """After a job finishes, the context-local is cleared — calling env.job_progress
        from outside (in the same thread) is a no-op again."""
        from ede.foundation.jobs.services.progress import get_current_run_id

        env = env_with_jobs_and_executor

        env.models["ir.job"].create({
            "name": "progress-cleared.test",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
            "retry_policy": "none",
        })
        env.enqueue_job(
            target="src.tests.foundation.jobs.targets.echo_target",
            payload={},
        )

        assert get_current_run_id() is None
    ```

    Move the `from ede.foundation.jobs.services.progress import get_current_run_id` to the module-top imports of `test_progress.py`.

- [ ] **Step 5.2: Add the test target callable**

    Append to `src/tests/foundation/jobs/targets.py`:
    ```python
    def progress_reporting_target(env, payload=None):
        """Test target that reports progress mid-execution via env.job_progress.

        Iterates from 1..total, reports percent at each step. Returns total
        when done.
        """
        total = (payload or {}).get("total", 4)
        for i in range(1, total + 1):
            pct = (i / total) * 100.0
            msg = "done" if i == total else f"{i}/{total}"
            env.job_progress(percent=pct, message=msg)
        return {"completed": total}
    ```

- [ ] **Step 5.3: Run — expect failure (progress writes through but task wrapper doesn't set context yet)**
    ```bash
    pytest src/tests/foundation/jobs/test_progress.py -v
    ```
    Both new tests should FAIL — the integration test will show `progress_pct = None` because the context wasn't set during execution.

- [ ] **Step 5.4: Wire set/clear into the task wrapper**

    Open `src/ede/foundation/jobs/services/task_wrapper.py`. Add to the module-top imports:
    ```python
    from .progress import set_progress_context, clear_progress_context
    ```

    Inside `execute_run`, find where the target callable is invoked. The current shape is roughly:
    ```python
        run.write({"status": "running", ...})
        started = _now_utc()
        try:
            target_fn = _resolve_dotted_path(job.target)
            output = target_fn(env, payload=run.payload or {})
            ...
    ```

    Change to:
    ```python
        run.write({"status": "running", ...})
        started = _now_utc()
        set_progress_context(run_id)
        try:
            target_fn = _resolve_dotted_path(job.target)
            output = target_fn(env, payload=run.payload or {})
            ...
        # except-branch + finally stay as they are
        finally:
            clear_progress_context()
            release_lock(env, lock_key=f"job:{job.name}")
    ```

    The clear MUST be in `finally` so even an exception leaves the context-local empty. The release_lock call was already in finally; just append clear_progress_context BEFORE release_lock (order shouldn't matter, but conceptually progress is "task scoped" and lock is "infrastructure scoped").

- [ ] **Step 5.5: Run — expect 7 PASS (5 prior + 2 new)**
    ```bash
    pytest src/tests/foundation/jobs/test_progress.py -v
    ```

- [ ] **Step 5.6: Run full jobs suite + lint**
    ```bash
    pytest src/tests/foundation/jobs/ -v
    ruff check src/ede/foundation/jobs/services/task_wrapper.py src/tests/foundation/jobs/test_progress.py src/tests/foundation/jobs/targets.py
    ```
    Expected: prior count + 2 = ~62 tests passing. Ruff clean.

---

## Task 6: End-to-end retry chain test

**Files:**
- Create: `src/tests/foundation/jobs/test_retry_chain_e2e.py`
- Modify: `src/tests/foundation/jobs/targets.py` — add a deterministic flaky target

- [ ] **Step 6.1: Add the flaky target**

    Append to `src/tests/foundation/jobs/targets.py`:
    ```python
    # Process-global counter for the flaky target. Tests must reset before use.
    _flaky_counter = {"count": 0}


    def reset_flaky_target() -> None:
        """Test helper — reset the flaky target's counter between tests."""
        _flaky_counter["count"] = 0


    def flaky_target_succeeds_on_attempt(env, payload=None):
        """Test target that fails the first N-1 times then succeeds.

        payload = {"succeed_on_attempt": 3} → fails on attempts 1+2, succeeds on 3.
        Reads attempt_number from the implicit run context (via the
        progress thread-local — peeks ir.job.run.attempt_number).
        """
        from ede.foundation.jobs.services.progress import get_current_run_id

        _flaky_counter["count"] += 1
        succeed_on = (payload or {}).get("succeed_on_attempt", 3)

        run_id = get_current_run_id()
        if run_id is None:
            # Outside a job context, just succeed (safe for unit tests)
            return {"ok": True}

        run = env.models["ir.job.run"].browse(run_id)
        if run.attempt_number < succeed_on:
            raise RuntimeError(f"flaky failure on attempt {run.attempt_number}")
        return {"ok": True, "succeeded_on_attempt": run.attempt_number}
    ```

- [ ] **Step 6.2: Write the e2e tests**

    Create `src/tests/foundation/jobs/test_retry_chain_e2e.py`:
    ```python
    """End-to-end retry chain tests — full path from enqueue through Celery eager retries."""
    from src.tests.foundation.jobs.targets import reset_flaky_target


    def test_target_fails_3_times_then_dead_letters(env_with_jobs_and_executor):
        """raise_target with retry_policy=fixed, max_attempts=3 → 3 failed retries → dead_letter."""
        env = env_with_jobs_and_executor

        env.models["ir.job"].create({
            "name": "retry-chain.exhausts",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.raise_target",
            "kind": "queued",
            "source": "runtime",
            "retry_policy": "fixed",
            "retry_base_seconds": 0,         # eager mode ignores eta but be explicit
            "retry_max_attempts": 3,
        })

        run = env.enqueue_job(
            target="src.tests.foundation.jobs.targets.raise_target",
            payload={},
        )

        # Eager mode: all 3 retries fired synchronously. Find all runs for this job.
        job = env.models["ir.job"].search([("name", "=", "retry-chain.exhausts")])[0]
        all_runs = env.models["ir.job.run"].search(
            [("job_id", "=", job.id)], order="attempt_number asc",
        )
        # We expect EXACTLY 3 runs: attempt 1 failed, attempt 2 failed, attempt 3 dead_letter
        # (retry_max_attempts=3 means up to 3 total attempts; on the 3rd failure no further retry).
        assert len(all_runs) == 3
        assert [r.attempt_number for r in all_runs] == [1, 2, 3]
        assert all_runs[0].status == "failed"
        assert all_runs[1].status == "failed"
        assert all_runs[2].status == "dead_letter"

        # Retry chain reconstructable via parent_run_id
        assert not all_runs[0].parent_run_id           # root
        assert all_runs[1].parent_run_id.id == all_runs[0].id
        assert all_runs[2].parent_run_id.id == all_runs[1].id


    def test_target_succeeds_on_third_attempt(env_with_jobs_and_executor):
        """flaky target that succeeds on attempt 3 → 2 failed + 1 success in the chain."""
        env = env_with_jobs_and_executor
        reset_flaky_target()

        env.models["ir.job"].create({
            "name": "retry-chain.eventual-success",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.flaky_target_succeeds_on_attempt",
            "kind": "queued",
            "source": "runtime",
            "retry_policy": "fixed",
            "retry_base_seconds": 0,
            "retry_max_attempts": 5,
        })

        env.enqueue_job(
            target="src.tests.foundation.jobs.targets.flaky_target_succeeds_on_attempt",
            payload={"succeed_on_attempt": 3},
        )

        job = env.models["ir.job"].search([("name", "=", "retry-chain.eventual-success")])[0]
        all_runs = env.models["ir.job.run"].search(
            [("job_id", "=", job.id)], order="attempt_number asc",
        )
        assert len(all_runs) == 3
        assert [r.attempt_number for r in all_runs] == [1, 2, 3]
        assert all_runs[0].status == "failed"
        assert all_runs[1].status == "failed"
        assert all_runs[2].status == "success"
        assert all_runs[2].output == {"ok": True, "succeeded_on_attempt": 3}
    ```

    Move the `from src.tests.foundation.jobs.targets import reset_flaky_target` to module top of `test_retry_chain_e2e.py`.

- [ ] **Step 6.3: Run — expect both PASS in eager mode**
    ```bash
    pytest src/tests/foundation/jobs/test_retry_chain_e2e.py -v
    ```

    If something fails:
    - **`len(all_runs) != 3`** → the retry chain didn't fire all attempts. Check that `attempts_remaining = run.attempt_number < job.retry_max_attempts` (NOT `<=`) — the gate is "have we already done max_attempts? then dead-letter".
    - **All status == "failed", none "dead_letter"** → the final-attempt branch isn't triggering. Verify the condition above.
    - **`parent_run_id` is None on retry runs** → `_schedule_retry_attempt` isn't setting it. Check the `create` call.

- [ ] **Step 6.4: Run full suite**
    ```bash
    pytest src/tests/foundation/jobs/ -v
    ```
    Expected: ~64 tests passing.

- [ ] **Step 6.5: Lint**
    ```bash
    ruff check src/tests/foundation/jobs/test_retry_chain_e2e.py src/tests/foundation/jobs/targets.py
    ```

---

## Task 7: Acceptance gate + roadmap status flip + commit

- [ ] **Step 7.1: Full repo regression check**

    ```bash
    ./run_tests.sh
    ```
    Expected: exit 0, prior baseline + ~20 new = new baseline, all green.

- [ ] **Step 7.2: Boot smoke**

    ```bash
    ede info --config ede.conf 2>&1 | grep -iE "foundation.jobs|reconciler" | head -3
    ```
    Expected: shows `foundation.jobs (Background Jobs)`; no reconciler line (Slice 2 fix already removed it from info).

- [ ] **Step 7.3: Flip the roadmap status — 4 sites**

    All four still 🟡 (Phase 1 incomplete — Slice 4 remains). Update the parenthetical:

    **(a) `roadmap/foundation/jobs/README.md`** — top Status header:
    ```
    **Status:** 🟡 In Progress (Slice 1 ✅ 2026-05-18 + Slice 2 ✅ 2026-05-19 + Slice 3 ✅ 2026-05-19 — schema + Celery executor + jobs-worker CLI + cron-driven scheduler + decorators + XML data path + boot reconciler + retry policy + dead-letter + env.job_progress; Slice 4 remains: admin UI + RBAC + heartbeat first adopter + walkthrough)
    ```

    And the Phased Delivery table Phase 1 row:
    ```
    | [Phase 1](./phase-1-implementation.md) | ... | ~3 weeks | 🟡 In Progress (Slice 1+2+3 ✅; Slice 4 remains) |
    ```

    **(b) `roadmap/foundation/jobs/phase-1-implementation.md`** — top Status header:
    ```
    **Status:** 🟡 In Progress (Slices 1+2+3 ✅ — WS-J1 + WS-J2 + parts of WS-J3 + WS-J4 + parts of WS-J5 + WS-J6 + WS-J7; remaining: WS-J8 admin UI, WS-J9 RBAC seed, WS-J10 heartbeat first adopter walkthrough)
    ```

    **(c) `roadmap/roadmap-tracker.md`** — Overall line + Phase 1 row + Last refreshed (prepend Slice 3 entry, demote prior Slice 2 mention to "Prior 2026-05-19 refresh").

    **(d) `docs/foundation-jobs.md`** — Status header + Status Snapshot row + add a Built Capabilities row for Slice 3. Bump `Last sync` to today.

- [ ] **Step 7.4: Append PROGRESS.md row**

    Append a new row after the Slice 2 post-delivery row, dated 2026-05-19, theme: `foundation.jobs Phase 1 Slice 3 🟡 — retry policy + dead-letter + env.job_progress`. Capture: lines added (~600-800), hours (~1 design + 2 dev + 0.5 review = ~3.5 total).

- [ ] **Step 7.5: Stage + commit**

    Stage only Slice 3 files + the roadmap / docs / PROGRESS flips. Leave any unrelated in-flight work alone. Commit message format:
    ```
    [IMP] foundation.jobs Phase 1 Slice 3 (🟡): retry policy + dead-letter + env.job_progress

    Third slice of Phase 1 — the engine is now production-resilient.
    ...
    ```

- [ ] **Step 7.6: Pause for explicit user `commit` instruction** (CLAUDE.md hard rule — controller, NOT the implementer, runs `git commit`).

---

## Self-Review

**1. Spec coverage:**
- ✅ Retry policy (Task 1) — 4 policies + jitter + invalid-policy guard
- ✅ Progress reporting (Task 2) — context-local + env.job_progress + write_progress
- ✅ Task wrapper retry-on-failure (Task 3) — child run + apply_async with eta
- ✅ Dead-letter notification (Task 4) — loose-coupled dispatch + graceful when unregistered
- ✅ Progress thread-local wiring (Task 5) — set before target, clear in finally
- ✅ E2E retry chain (Task 6) — exhausts + eventual success paths
- ✅ Acceptance gate + 4-site flip + commit (Task 7)

**2. Placeholder scan:**
- `_dispatch_dead_letter_notification` stub in Task 3 is intentionally a no-op pending Task 4 — this is a documented two-task split, not a placeholder. Task 4 fills it in.
- One "NOTE on the inline import" callout in Task 2 documents the legal CLAUDE.md exception case AND provides the canonical try/except-at-module-top alternative — pick one. Both are concrete, not "TODO".
- All test bodies are complete code, not "write tests for X".

**3. Type consistency:**
- `compute_retry_delay_seconds(policy, attempt_number, base_seconds, *, jitter_pct=0.2) -> Optional[int]` consistent between definition (Task 1) and consumption (Task 3).
- `set_progress_context(run_id: str) -> None` / `clear_progress_context() -> None` / `write_progress(env, *, percent, message=None) -> None` consistent across Task 2 (definition) + Task 5 (wiring).
- `env.job_progress(*, percent, message=None) -> None` consistent across Task 2 (definition) + Task 5 tests (consumption).
- `_schedule_retry_attempt(env, parent_run, task, *, delay_seconds)` matches its single call site in Task 3 's failure branch.

**4. Ordering:**
- Task 1 → Task 2 → Task 3 → Task 4 → Task 5 → Task 6 → Task 7. Tasks 1+2 are independent; Task 3 depends on Task 1; Task 4 extends Task 3; Task 5 depends on Task 2's helpers but extends Task 3's wrapper; Task 6 depends on Task 5 (because the flaky target reads `get_current_run_id()`); Task 7 closes the slice.

---

## Execution Handoff

**Plan complete and saved to** `docs/superpowers/plans/2026-05-19-foundation-jobs-phase-1-slice-3.md`.

Two execution options:
**1. Subagent-Driven (recommended)** — same pattern that delivered Slices 1+2. ~8-12 subagent invocations including review loops.
**2. Inline execution with checkpoints** — slower but more visible.

Which approach?
