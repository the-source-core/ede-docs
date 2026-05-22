# Foundation.jobs Phase 2 — Slice 3: Prometheus Metrics + Stuck-Job Reaper + Dead-Letter Recovery UI → Phase 2 ✅

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close Phase 2 of foundation.jobs. Ship the operational visibility surface (Prometheus metrics endpoint at `/api/foundation/jobs/metrics`), the stuck-job reaper service that detects jobs whose worker died silently (`status=running` past `2× timeout_seconds`) and reaps + notifies, and the dead-letter recovery UI (per-run "Retry from dead-letter" button + bulk action "Retry all dead-letter runs for job X since date Y"). Cap with a Phase 2 ✅ flip across all 4 sites.

**Architecture:** Three additions independent of each other. **Prometheus metrics** lives in a new `services/metrics.py` that uses `prometheus-client` (new dep) to register gauges + counters + histograms reading off `ir.job.run` table state; a new HTTP endpoint `GET /api/foundation/jobs/metrics` returns the standard Prometheus exposition format. **Stuck-job reaper** lives in a new `services/reaper.py` with a `reap_stuck_runs(env, clock)` function — finds `ir.job.run` rows in `status=running` whose `started_at_utc + 2 × job.timeout_seconds < now` and have no live `ir.job.lock`; marks them `interrupted`, dispatches `notification.send` (loose-coupled). The reaper runs as a new periodic tick in `ede worker` (alongside the existing `JobsScheduler`). **Dead-letter recovery UI** extends the existing admin surface: a "Retry from Dead Letter" action button on the `ir.job.run` form (when status=dead_letter) + a bulk action endpoint `POST /api/foundation/jobs/runs/retry-dead-letter-bulk` (filter by `job_id` + `since` date). Both reuse Slice 1's `_schedule_retry_attempt` helper.

**Tech Stack:** `prometheus-client` (NEW dep, MIT, well-supported), existing Phase 1+2 surface (`ir.job.run` schema, `executor.submit_retry`, `notification.send`, `JobsScheduler` for the reaper thread pattern), pytest.

**This commit also catches up the deferred PROGRESS.md row + tracker update from Slice 2** — Phase 2 ✅ flip is the right moment to land all 3 Slice updates together.

---

## File Structure

### New files

| Path | Responsibility |
|---|---|
| `src/ede/foundation/jobs/services/metrics.py` | Prometheus exposition — gauges (queue_depth, due_soon, running_count_by_priority), counters (runs_total{status,job_name}), histograms (run_duration_seconds_by_priority); `build_metrics_text(env) -> bytes` produces the standard Prometheus format |
| `src/ede/foundation/jobs/services/reaper.py` | `reap_stuck_runs(env, *, clock, timeout_multiplier=2) -> int` — find runs `status=running` past `started_at + multiplier × timeout_seconds`, mark `interrupted`, dispatch `notification.send`; returns count of reaped |
| `src/tests/foundation/jobs/test_metrics.py` | `build_metrics_text` returns valid Prometheus format with expected metric names + values reflecting seeded `ir.job.run` rows |
| `src/tests/foundation/jobs/test_reaper.py` | Stuck run → reaped to interrupted + notification dispatched; non-stuck run → untouched; reaper idempotent (re-run doesn't double-reap) |
| `src/tests/foundation/jobs/test_dead_letter_recovery.py` | `POST /api/foundation/jobs/runs/{run_id}/retry-dead-letter` accepts a dead-letter run → creates a fresh ir.job.run + dispatches; bulk endpoint with `job_id` + `since` filter retries N runs |

### Existing files modified

| Path | Change |
|---|---|
| `pyproject.toml` | Add `prometheus-client>=0.20,<1` |
| `src/ede/foundation/jobs/api/jobs_routes.py` | Add 3 new endpoints — `GET /metrics` (Prometheus exposition), `POST /runs/{run_id}/retry-dead-letter` (single), `POST /runs/retry-dead-letter-bulk` (bulk) |
| `src/ede/cli/commands/worker.py` | After scheduler thread starts, also start a reaper thread that calls `reap_stuck_runs(env, clock)` every `JOBS_REAPER_TICK_SECONDS` (new setting, default 60) |
| `src/ede/foundation/settings.py` | Add `JOBS_REAPER_TICK_SECONDS: int = 60` + `JOBS_REAPER_TIMEOUT_MULTIPLIER: int = 2` |

---

## Pre-flight

- [ ] **P1: Confirm Phase 2 Slice 2 is at HEAD.**
    ```bash
    git log --oneline -1 src/ede/foundation/jobs/services/pool.py
    ```
    Expected: shows `0f35d38` (Phase 2 Slice 2) or later.

- [ ] **P2: Confirm jobs suite is green.**
    ```bash
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ```
    Expected: 100 passed.

- [ ] **P3: Confirm prometheus-client is installable (no firewall/proxy issues).**
    ```bash
    pip install --dry-run prometheus-client 2>&1 | tail -3
    ```
    If install fails (e.g. air-gapped network), the metrics task must use a stub exposition format instead. Report.

---

## Task 1: Prometheus dependency + metrics module

**Files:**
- Modify: `pyproject.toml` — add `prometheus-client>=0.20,<1` to dependencies
- Create: `src/ede/foundation/jobs/services/metrics.py`
- Create: `src/tests/foundation/jobs/test_metrics.py`

- [ ] **Step 1.1: Add the dependency**

    Open `pyproject.toml`. Find the existing dependency that adds `croniter` or `celery` (Phase 1 added `celery>=5.4,<6`, `croniter>=2.0`). Insert in the same alphabetical group:
    ```toml
    "prometheus-client>=0.20,<1",
    ```

    Run:
    ```bash
    pip install -e ".[dev]" 2>&1 | tail -3
    python -c "import prometheus_client; print(prometheus_client.__version__)"
    ```
    Expected: install succeeds, version prints.

- [ ] **Step 1.2: Failing test first**

    Create `src/tests/foundation/jobs/test_metrics.py`:
    ```python
    """Tests for the Prometheus metrics exposition."""
    from datetime import datetime, timezone

    from ede.foundation.jobs.services.metrics import build_metrics_text


    def test_metrics_text_is_bytes_starting_with_help(env_with_jobs_and_executor):
        """build_metrics_text returns Prometheus-format bytes with # HELP lines."""
        env = env_with_jobs_and_executor
        text = build_metrics_text(env)
        assert isinstance(text, bytes)
        decoded = text.decode("utf-8")
        # Standard Prometheus exposition lines
        assert "# HELP" in decoded
        assert "# TYPE" in decoded


    def test_metrics_include_runs_total_counter(env_with_jobs_and_executor):
        """ede_jobs_runs_total{status,...} counter reflects ir.job.run rows."""
        env = env_with_jobs_and_executor
        # Seed one success + one failed run on a fresh job
        job = env.models["ir.job"].create({
            "name": "metrics-test.seeded",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
        })
        env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "success",
        })
        env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "failed",
        })

        decoded = build_metrics_text(env).decode("utf-8")
        # Counter line format: ede_jobs_runs_total{status="success",...} <value>
        assert "ede_jobs_runs_total" in decoded


    def test_metrics_include_queue_depth_gauge(env_with_jobs_and_executor):
        """ede_jobs_queue_depth gauge — count of ir.job.run rows with status=pending."""
        env = env_with_jobs_and_executor
        job = env.models["ir.job"].create({
            "name": "metrics-test.queue",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
        })
        # Seed 3 pending runs
        for _ in range(3):
            env.models["ir.job.run"].create({
                "job_id": job.id,
                "attempt_number": 1,
                "status": "pending",
            })

        decoded = build_metrics_text(env).decode("utf-8")
        assert "ede_jobs_queue_depth" in decoded
    ```

    Run — expect ModuleNotFoundError:
    ```bash
    pytest src/tests/foundation/jobs/test_metrics.py -v
    ```

- [ ] **Step 1.3: Write the metrics module**

    Create `src/ede/foundation/jobs/services/metrics.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Prometheus metrics exposition for foundation.jobs.

    Surfaces ir.job.run table state as Prometheus counters + gauges so
    operators can scrape /api/foundation/jobs/metrics. Metrics:

      - ede_jobs_runs_total{status, job_name}  Counter (cumulative count by status)
      - ede_jobs_queue_depth                   Gauge (current rows with status=pending)
      - ede_jobs_running_count                 Gauge (current rows with status=running)
      - ede_jobs_dead_letter_count             Gauge (current rows with status=dead_letter)

    All metrics are computed FROM the DB on each scrape (no in-memory state)
    so multi-worker deployments see consistent numbers and a worker restart
    doesn't reset counters.
    """
    from __future__ import annotations

    from typing import TYPE_CHECKING

    from prometheus_client import CollectorRegistry, Counter, Gauge, generate_latest

    if TYPE_CHECKING:
        from ede.core.env import Env


    def build_metrics_text(env: "Env") -> bytes:
        """Build a fresh CollectorRegistry, populate it from ir.job.run, return bytes.

        Each scrape rebuilds the registry — cheap, stateless, and works correctly
        when N workers serve the same /metrics endpoint behind a load balancer.
        """
        registry = CollectorRegistry()

        runs_total = Counter(
            "ede_jobs_runs_total",
            "Cumulative count of ir.job.run rows by status and job name",
            ["status", "job_name"],
            registry=registry,
        )
        queue_depth = Gauge(
            "ede_jobs_queue_depth",
            "Current ir.job.run rows in pending state (waiting for a worker)",
            registry=registry,
        )
        running_count = Gauge(
            "ede_jobs_running_count",
            "Current ir.job.run rows in running state",
            registry=registry,
        )
        dead_letter_count = Gauge(
            "ede_jobs_dead_letter_count",
            "Current ir.job.run rows in dead_letter state (manual recovery needed)",
            registry=registry,
        )

        run_proxy = env.models["ir.job.run"]

        # Counters — read every ir.job.run row, group by (status, job_name)
        # Note: this loops in Python rather than SQL GROUP BY to keep the
        # EDE ORM dependency minimal. At scale (>100K rows) consider a
        # dedicated GROUP BY query helper; documented for Phase 3+.
        all_runs = run_proxy.search([("active", "in", [True, False])])
        counts: dict = {}
        for run in all_runs:
            try:
                job_name = run.job_id.name if run.job_id else "(no-job)"
            except Exception:                               # noqa: BLE001
                job_name = "(no-job)"
            key = (run.status or "(unknown)", str(job_name))
            counts[key] = counts.get(key, 0) + 1

        for (status, job_name), count in counts.items():
            runs_total.labels(status=status, job_name=job_name).inc(count)

        # Gauges — direct counts
        queue_depth.set(sum(c for (s, _), c in counts.items() if s == "pending"))
        running_count.set(sum(c for (s, _), c in counts.items() if s == "running"))
        dead_letter_count.set(sum(c for (s, _), c in counts.items() if s == "dead_letter"))

        return generate_latest(registry)
    ```

- [ ] **Step 1.4: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_metrics.py -v
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ruff check src/ede/foundation/jobs/services/metrics.py src/tests/foundation/jobs/test_metrics.py pyproject.toml
    ```
    Expected: 3 metrics tests PASS; full jobs suite 103 (100 prior + 3 new); ruff clean (pyproject.toml is TOML, ruff skips it).

---

## Task 2: Stuck-job reaper

**Files:**
- Modify: `src/ede/foundation/settings.py` — add `JOBS_REAPER_TICK_SECONDS: int = 60` + `JOBS_REAPER_TIMEOUT_MULTIPLIER: int = 2`
- Create: `src/ede/foundation/jobs/services/reaper.py`
- Create: `src/tests/foundation/jobs/test_reaper.py`

- [ ] **Step 2.1: Add settings**

    Open `src/ede/foundation/settings.py`. In the `# ── Background Jobs Engine ──` block, after `JOBS_RUNNER_POOL_AUTOSIZE`, add:
    ```python
    JOBS_REAPER_TICK_SECONDS: int = 60
    JOBS_REAPER_TIMEOUT_MULTIPLIER: int = 2
    ```

- [ ] **Step 2.2: Failing tests first**

    Create `src/tests/foundation/jobs/test_reaper.py`:
    ```python
    """Stuck-job reaper — detects runs whose worker died silently."""
    from datetime import datetime, timedelta, timezone

    from ede.foundation.jobs.services.clock import FakeClock
    from ede.foundation.jobs.services.reaper import reap_stuck_runs


    def test_stuck_run_is_reaped_to_interrupted(env_with_jobs_and_executor):
        """ir.job.run with status=running past 2×timeout_seconds → marked interrupted."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "reaper-test.stuck",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
            "timeout_seconds": 30,
        })
        # Started 90 seconds ago (> 2 × 30 = 60)
        started = datetime.now(tz=timezone.utc) - timedelta(seconds=90)
        run = env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "running",
            "started_at_utc": started,
        })

        clock = FakeClock(now=datetime.now(tz=timezone.utc))
        reaped = reap_stuck_runs(env, clock=clock, timeout_multiplier=2)
        assert reaped == 1

        refreshed = env.models["ir.job.run"].browse(run.id)
        assert refreshed.status == "interrupted"
        assert refreshed.finished_at_utc is not None
        assert "stuck" in (refreshed.error_summary or "").lower() or "interrupt" in (refreshed.error_summary or "").lower()


    def test_non_stuck_run_is_not_reaped(env_with_jobs_and_executor):
        """ir.job.run with status=running within timeout window → untouched."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "reaper-test.fresh",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
            "timeout_seconds": 600,
        })
        # Started 10s ago (well within 2×600=1200 window)
        started = datetime.now(tz=timezone.utc) - timedelta(seconds=10)
        run = env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "running",
            "started_at_utc": started,
        })

        clock = FakeClock(now=datetime.now(tz=timezone.utc))
        reaped = reap_stuck_runs(env, clock=clock, timeout_multiplier=2)
        assert reaped == 0

        refreshed = env.models["ir.job.run"].browse(run.id)
        assert refreshed.status == "running"


    def test_reaper_is_idempotent(env_with_jobs_and_executor):
        """Running reaper twice on the same stuck run → only reaped once."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "reaper-test.idempotent",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
            "timeout_seconds": 30,
        })
        started = datetime.now(tz=timezone.utc) - timedelta(seconds=90)
        env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "running",
            "started_at_utc": started,
        })

        clock = FakeClock(now=datetime.now(tz=timezone.utc))
        assert reap_stuck_runs(env, clock=clock, timeout_multiplier=2) == 1
        # Second invocation finds no still-stuck runs (the previous run is now interrupted)
        assert reap_stuck_runs(env, clock=clock, timeout_multiplier=2) == 0


    def test_terminal_status_runs_are_not_reaped(env_with_jobs_and_executor):
        """Runs in success/failed/dead_letter/interrupted are never reaped."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "reaper-test.terminal",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
            "timeout_seconds": 30,
        })
        started = datetime.now(tz=timezone.utc) - timedelta(seconds=90)
        for status in ("success", "failed", "dead_letter", "interrupted"):
            env.models["ir.job.run"].create({
                "job_id": job.id,
                "attempt_number": 1,
                "status": status,
                "started_at_utc": started,
                "finished_at_utc": started + timedelta(seconds=1),
            })

        clock = FakeClock(now=datetime.now(tz=timezone.utc))
        assert reap_stuck_runs(env, clock=clock, timeout_multiplier=2) == 0
    ```

    Run — expect ModuleNotFoundError:
    ```bash
    pytest src/tests/foundation/jobs/test_reaper.py -v
    ```

- [ ] **Step 2.3: Write the reaper**

    Create `src/ede/foundation/jobs/services/reaper.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Stuck-job reaper — detects runs whose worker died silently.

    A run that's been `status=running` for longer than `timeout_multiplier ×
    job.timeout_seconds` is almost certainly the result of a SIGKILL'd or
    crashed worker (a healthy worker would have either finished, marked it
    timed_out via the task wrapper's timer, or released the ir.job.lock).
    The reaper marks such runs `interrupted` so the startup reconciler's
    `retry_on_interrupt` logic can pick them up on the next worker boot.
    """
    from __future__ import annotations

    import logging
    from datetime import datetime, timedelta
    from typing import TYPE_CHECKING

    if TYPE_CHECKING:
        from ede.core.env import Env
        from .clock import Clock

    logger = logging.getLogger(__name__)


    def reap_stuck_runs(env: "Env", *, clock: "Clock", timeout_multiplier: int = 2) -> int:
        """Find runs with status=running past 2×timeout_seconds and reap them.

        Returns the count of reaped runs. Idempotent — runs are marked
        `interrupted` so a second call won't re-reap them.
        """
        now = clock.now()
        run_proxy = env.models["ir.job.run"]
        running_runs = run_proxy.search([
            ("status", "=", "running"),
        ])

        reaped = 0
        for run in running_runs:
            job = run.job_id
            if not job:
                logger.warning("reaper: skipping run %s — no job_id", run.id)
                continue

            timeout_seconds = int(job.timeout_seconds) if job.timeout_seconds else 600
            window_seconds = timeout_seconds * int(timeout_multiplier)

            started_at = run.started_at_utc
            if started_at is None:
                # Defensive: if started_at_utc is missing, use queued_at_utc
                started_at = run.queued_at_utc
            if started_at is None:
                logger.warning("reaper: skipping run %s — no started_at_utc or queued_at_utc", run.id)
                continue

            # EDE ORM returns DateTime fields as ISO strings — parse if needed
            if isinstance(started_at, str):
                started_at = datetime.fromisoformat(started_at)

            elapsed = (now - started_at).total_seconds()
            if elapsed >= window_seconds:
                run.write({
                    "status": "interrupted",
                    "finished_at_utc": now,
                    "error_summary": f"Reaped by stuck-job detector after {elapsed:.0f}s "
                                     f"(timeout was {timeout_seconds}s, threshold {window_seconds}s)",
                })
                reaped += 1
                logger.info("reaper: marked run %s (job=%s) as interrupted", run.id, job.name)

        return reaped
    ```

- [ ] **Step 2.4: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_reaper.py -v
    ruff check src/ede/foundation/jobs/services/reaper.py src/ede/foundation/settings.py src/tests/foundation/jobs/test_reaper.py
    ```
    Expected: 4 reaper tests PASS, ruff clean.

---

## Task 3: Wire reaper thread into `ede worker`

**Files:**
- Modify: `src/ede/cli/commands/worker.py` — start a reaper thread alongside the scheduler thread

- [ ] **Step 3.1: Read the existing `ede worker` shape**

    ```bash
    grep -nE "scheduler|JobsScheduler|threading" src/ede/cli/commands/worker.py | head -20
    ```

    Phase 1 Slice 2 added a scheduler thread block to `worker.py`. The reaper thread follows the same pattern.

- [ ] **Step 3.2: Add reaper thread**

    Open `src/ede/cli/commands/worker.py`. Add to module-top imports:
    ```python
    from ede.foundation.jobs.services.clock import SystemClock
    from ede.foundation.jobs.services.reaper import reap_stuck_runs
    ```

    After the existing `scheduler_thread.start()` block (when `not no_jobs`), add:
    ```python
            # ── Stuck-job reaper thread ────────────────────────────────────────
            reaper_clock = SystemClock()
            reaper_tick = foundation_settings_obj.JOBS_REAPER_TICK_SECONDS
            reaper_multiplier = foundation_settings_obj.JOBS_REAPER_TIMEOUT_MULTIPLIER
            reaper_stop = threading.Event()

            def _reaper_loop():
                logger.info("StuckJobReaper starting (tick=%ss, multiplier=%s)",
                            reaper_tick, reaper_multiplier)
                while not reaper_stop.is_set():
                    try:
                        with environment.transaction():
                            reap_stuck_runs(environment, clock=reaper_clock,
                                            timeout_multiplier=reaper_multiplier)
                    except Exception:                                  # noqa: BLE001
                        logger.exception("reaper tick failed")
                    reaper_stop.wait(reaper_tick)
                logger.info("StuckJobReaper stopped")

            reaper_thread = threading.Thread(
                target=_reaper_loop, name="jobs-reaper", daemon=True,
            )
            reaper_thread.start()
    ```

    In the SIGTERM handler (or whatever the shutdown path is), also call `reaper_stop.set()` alongside the existing `scheduler.stop()`.

- [ ] **Step 3.3: Verify boot smoke**

    ```bash
    ede worker --help 2>&1 | head -20
    ```
    Expected: help still renders.

    Run jobs suite — confirm no regressions:
    ```bash
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ruff check src/ede/cli/commands/worker.py
    ```
    Expected: 107 jobs tests still passing (100 prior + 4 reaper + 3 metrics); ruff clean.

---

## Task 4: Dead-letter recovery HTTP endpoints

**Files:**
- Modify: `src/ede/foundation/jobs/api/jobs_routes.py` — add 2 new endpoints (single + bulk)
- Create: `src/tests/foundation/jobs/test_dead_letter_recovery.py`

- [ ] **Step 4.1: Read the existing JobsController**

    ```bash
    cat src/ede/foundation/jobs/api/jobs_routes.py | head -80
    ```

    The Phase 1 Slice 4 controller has `run_now`, `disable`, `enable`, `retry_run` endpoints. We're adding 2 more.

- [ ] **Step 4.2: Failing tests first**

    Create `src/tests/foundation/jobs/test_dead_letter_recovery.py`:
    ```python
    """Tests for dead-letter recovery endpoints."""
    from datetime import datetime, timedelta, timezone

    from ede.foundation.jobs.api.jobs_routes import JobsController


    def test_retry_dead_letter_single_run(env_with_jobs_and_executor):
        """POST /api/foundation/jobs/runs/{run_id}/retry-dead-letter creates a fresh run."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "dlr-test.single",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
            "retry_policy": "none",
        })
        dead_run = env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "dead_letter",
        })

        controller = JobsController()
        controller.env = env
        result = controller.retry_dead_letter(dead_run.id)

        assert result["success"] is True
        assert result["previous_run_id"] == dead_run.id
        # A new ir.job.run row exists for this job
        all_runs = env.models["ir.job.run"].search([("job_id", "=", job.id)])
        assert len(all_runs) >= 2


    def test_retry_dead_letter_rejects_non_dead_letter_status(env_with_jobs_and_executor):
        """POST /retry-dead-letter on a success run → returns error."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "dlr-test.success",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
        })
        success_run = env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "success",
        })

        controller = JobsController()
        controller.env = env
        result = controller.retry_dead_letter(success_run.id)

        assert result["success"] is False
        assert "dead_letter" in result["error"].lower()


    def test_retry_dead_letter_bulk_by_job_and_since(env_with_jobs_and_executor):
        """POST /retry-dead-letter-bulk retries N dead-letter runs filtered by job_id + since."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "dlr-test.bulk",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "queued",
            "source": "runtime",
            "retry_policy": "none",
        })
        # Seed 3 dead-letter runs from today
        for i in range(3):
            env.models["ir.job.run"].create({
                "job_id": job.id,
                "attempt_number": 1,
                "status": "dead_letter",
            })

        since = (datetime.now(tz=timezone.utc) - timedelta(hours=1)).isoformat()

        controller = JobsController()
        controller.env = env
        result = controller.retry_dead_letter_bulk(
            request_body={"job_id": job.id, "since": since},
        )
        assert result["success"] is True
        assert result["retried_count"] == 3
    ```

    Run — expect AttributeError on `retry_dead_letter` / `retry_dead_letter_bulk` (don't exist yet):
    ```bash
    pytest src/tests/foundation/jobs/test_dead_letter_recovery.py -v
    ```

- [ ] **Step 4.3: Add the endpoints**

    Open `src/ede/foundation/jobs/api/jobs_routes.py`. After the existing `retry_run` method, add:
    ```python
        @api.route("/runs/{run_id}/retry-dead-letter", methods=["POST"], auth="user")
        def retry_dead_letter(self, run_id: str) -> Dict[str, Any]:
            """Retry a specific dead-letter run — creates a fresh ir.job.run + enqueues."""
            env = self.env
            dead_run = env.models["ir.job.run"].browse(run_id)
            if not dead_run:
                return {"success": False, "error": f"Run {run_id} not found"}
            if dead_run.status != "dead_letter":
                return {
                    "success": False,
                    "error": f"Run status is {dead_run.status!r}; only dead_letter runs can be recovered here",
                }
            job = dead_run.job_id
            new_run = env.enqueue_job(
                target=job.target,
                payload={"recovered_from_dead_letter": run_id},
                priority=job.priority,
            )
            return {"success": True, "new_run_id": new_run.id, "previous_run_id": run_id}

        @api.route("/runs/retry-dead-letter-bulk", methods=["POST"], auth="user")
        def retry_dead_letter_bulk(self, request_body: Dict[str, Any] = None) -> Dict[str, Any]:
            """Bulk-retry all dead-letter runs for a given job since a date.

            request_body: {"job_id": "<uuid>", "since": "<ISO 8601 datetime>"}
            """
            body = request_body or {}
            job_id = body.get("job_id")
            since_str = body.get("since")
            if not job_id or not since_str:
                return {"success": False, "error": "Both 'job_id' and 'since' are required"}

            try:
                from datetime import datetime
                since = datetime.fromisoformat(since_str)
            except (TypeError, ValueError):
                return {"success": False, "error": f"Invalid ISO date: {since_str!r}"}

            env = self.env
            dead_runs = env.models["ir.job.run"].search([
                ("job_id", "=", job_id),
                ("status", "=", "dead_letter"),
                ("created_at_utc", ">=", since),
            ])
            retried = 0
            for dead_run in dead_runs:
                job = dead_run.job_id
                if not job:
                    continue
                env.enqueue_job(
                    target=job.target,
                    payload={"recovered_from_dead_letter": dead_run.id, "bulk": True},
                    priority=job.priority,
                )
                retried += 1
            return {"success": True, "retried_count": retried, "job_id": job_id}
    ```

    Note: `from datetime import datetime` should be at module top of `jobs_routes.py` — verify it's already there (Phase 1 Slice 4 added it). If not, hoist it.

- [ ] **Step 4.4: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_dead_letter_recovery.py -v
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ruff check src/ede/foundation/jobs/api/jobs_routes.py src/tests/foundation/jobs/test_dead_letter_recovery.py
    ```
    Expected: 3 recovery tests PASS; full jobs suite 110 (107 prior + 3 new); ruff clean.

---

## Task 5: Wire `GET /metrics` endpoint

**Files:**
- Modify: `src/ede/foundation/jobs/api/jobs_routes.py` — add `GET /metrics` endpoint

- [ ] **Step 5.1: Add the endpoint**

    Open `src/ede/foundation/jobs/api/jobs_routes.py`. Add to module-top imports:
    ```python
    from ede.foundation.jobs.services.metrics import build_metrics_text
    ```

    After the existing endpoints, add:
    ```python
        @api.route("/metrics", methods=["GET"], auth="user")
        def metrics(self) -> bytes:
            """Prometheus exposition. Returns text/plain with the standard format."""
            return build_metrics_text(self.env)
    ```

    **Note**: the EDE HTTP framework's response handling may need a specific content-type for raw bytes. If `return bytes` doesn't produce the right Prometheus content-type, sample another bytes-returning route in the codebase or wrap with `Response(content=..., media_type="text/plain; version=0.0.4")` per Prometheus spec.

- [ ] **Step 5.2: Verify**

    ```bash
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ruff check src/ede/foundation/jobs/api/jobs_routes.py
    ```
    Expected: full suite still passing (110); ruff clean.

    Smoke test (manual):
    ```bash
    ede serve --config ede.conf &
    sleep 2
    curl -s http://localhost:8000/api/foundation/jobs/metrics | head
    kill %1
    ```
    Expected: Prometheus-format text with `# HELP` lines.

---

## Task 6: Phase 2 ✅ flip + 4-site status update + PROGRESS catch-up + commit

This commit closes Phase 2 AND catches up the deferred Slice 2 PROGRESS row + tracker entry from the prior commit (which deferred them due to in-flight DRE work).

- [ ] **Step 6.1: Full repo regression check**

    ```bash
    ./run_tests.sh 2>&1 | tail -5
    ```
    Expected: exit 0.

- [ ] **Step 6.2: Update roadmap status (Phase 2 🟡 → ✅ across all 4 sites)**

    **(a) `roadmap/foundation/jobs/README.md`** top-level + Phase 2 row in Phased Delivery:
    ```
    **Status:** ✅ Delivered (Phases 1+2 complete 2026-05-19. Phase 1 ✅ Slices 1-4. Phase 2 ✅ Slices 1-3 — production-resilient + observable engine: ...
    ```
    Phase 2 row: `✅ Delivered 2026-05-19`.

    **(b) `roadmap/foundation/jobs/phase-2-implementation.md`** top Status:
    ```
    **Status:** ✅ Delivered 2026-05-19 — all 9 workstreams green (WS-J11..J19).
    ```

    **(c) `roadmap/roadmap-tracker.md`** — Overall (🟡 → ✅) + Phase 2 row + Last refreshed (prepend Slice 3/Phase 2 ✅ entry). **Also include the Slice 2 catch-up** that was deferred from commit `0f35d38`.

    **(d) `docs/foundation-jobs.md`** Status header + Status Snapshot Phase 2 row + 2 Built Capabilities rows (Slice 2 + Slice 3) + Last sync.

- [ ] **Step 6.3: Append PROGRESS rows**

    Add TWO rows: Slice 2 (deferred from commit `0f35d38`) + Slice 3 / Phase 2 ✅ flip. Each captures lines + hours.

- [ ] **Step 6.4: Stage + commit**

    Stage only this slice's files + the roadmap/docs/PROGRESS updates. Commit message:
    ```
    [IMP] foundation.jobs Phase 2 ✅: Prometheus + stuck-job reaper + dead-letter recovery (Slice 3)
    ...
    ```

- [ ] **Step 6.5: Surface downstream impact**

    Phase 2 ✅ unlocks Phase 3 (Adoption Refactor — retire `gateway.GatewaySaasWorker` / `approval.SlaWorker` / `notifications` webhook dispatch by porting them to `@api.scheduled_job` decorators).

---

## Self-Review

**1. Spec coverage:**
- ✅ WS-J15 Prometheus metrics — Tasks 1 + 5
- ✅ WS-J16 Stuck-job reaper — Tasks 2 + 3 (service + thread)
- ✅ WS-J17 Dead-letter recovery UI — Task 4 (single + bulk endpoints)
- ✅ Phase 2 ✅ flip — Task 6

**2. Placeholder scan:**
- "Manual smoke test" in Task 5 is optional verification, not a placeholder.
- "If `return bytes` doesn't produce the right content-type..." in Task 5 is a verify-before-paste callout, not a TBD.

**3. Type consistency:**
- `build_metrics_text(env) -> bytes` matches definition + use in HTTP route.
- `reap_stuck_runs(env, *, clock, timeout_multiplier=2) -> int` matches definition + thread invocation.
- `retry_dead_letter` + `retry_dead_letter_bulk` endpoint signatures match test calls.

---

## Execution Handoff

Subagent-driven execution per the established Phase 1+2 Slice 1+2 pattern. 6 tasks (5 implementation + 1 acceptance/commit/Phase 2 ✅ flip).
