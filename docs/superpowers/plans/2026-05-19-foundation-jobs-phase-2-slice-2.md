# Foundation.jobs Phase 2 — Slice 2: Auto-Pool + Dependency Graph + Tenant Concurrency

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three operational primitives on top of Phase 2 Slice 1's foundation: a `JOBS_RUNNER_POOL_AUTOSIZE` setting that defaults Celery prefork concurrency from CPU count when enabled; an `ir.job.requires` Many-to-Many self-link that lets a job declare "I run only after job X succeeded in the same scheduling window"; and a per-tenant concurrency cap on `ir.job` (`tenant_concurrency_limit`) that the scheduler honours.

**Architecture:** Three independent additions, deliverable as one slice. **Auto-pool** lives in settings + `ede jobs-worker` startup logic — when `JOBS_RUNNER_POOL_AUTOSIZE=True`, replace the static `JOBS_CELERY_PREFORK_CONCURRENCY` with `multiprocessing.cpu_count()` clamped to `[1, 32]`. **Dependency graph** introduces `ir.job.requires_ids = fields.Many2Many("ir.job", relation="ir_job_requires", ...)` self-link + Alembic migration; the scheduler's `tick()` adds a gate that skips a due job if any of its `requires_ids` have NOT succeeded in the current window (calendar day UTC). **Tenant concurrency** adds `ir.job.tenant_concurrency_limit = fields.Integer(default=0)` (0 = unlimited) + a scheduler-side count of currently-running `ir.job.run` rows scoped to the job's tenant; skip dispatch when the count meets the limit.

**Tech Stack:** Python stdlib (`multiprocessing.cpu_count`), Alembic (one migration for the 2 new fields + the M2M join table), existing Slice 1 surface (FakeClock for scheduler tests, event constants), existing Phase 1 surface (`ir.job` schema + `JobsScheduler.tick()` + `ir.job.run.status`/`finished_at_utc`). NO new external dependencies.

**Does NOT cover (Slice 3):**
- WS-J15 Prometheus metrics endpoint
- WS-J16 Stuck-job reaper
- WS-J17 Dead-letter recovery UI (Retry button + bulk action)
- Phase 2 ✅ flip (that lands at the end of Slice 3)

---

## File Structure

### New files

| Path | Responsibility |
|---|---|
| `src/ede/foundation/jobs/migrations/versions/<rev>_phase2_slice2.py` | Alembic migration: add `requires_ids` M2M join table + `tenant_concurrency_limit` Integer column on `ir.job` |
| `src/tests/foundation/jobs/test_pool_autosize.py` | `compute_prefork_concurrency(settings)` math — autosize on/off, clamping |
| `src/tests/foundation/jobs/test_dependency_graph.py` | Scheduler skips a due job when its `requires_ids` haven't succeeded today; dispatches when they have; doesn't deadlock when a job requires itself (defensive — should be rejected) |
| `src/tests/foundation/jobs/test_tenant_concurrency.py` | Scheduler skips dispatch when a tenant's running-job count meets the cap; dispatches when under |

### Existing files modified

| Path | Change |
|---|---|
| `src/ede/foundation/jobs/models/job.py` | Add `requires_ids = fields.Many2Many(...)` self-link + `tenant_concurrency_limit = fields.Integer(default=0)` |
| `src/ede/foundation/jobs/services/scheduler.py` | `tick()` adds two gates: dependency-graph check + tenant-concurrency check; both inserted between the lock-acquire and the executor.submit |
| `src/ede/foundation/settings.py` | Add `JOBS_RUNNER_POOL_AUTOSIZE: bool = False` |
| `src/ede/cli/commands/jobs_worker.py` | When booting Celery, if `settings.JOBS_RUNNER_POOL_AUTOSIZE` is True, override `--concurrency` with `compute_prefork_concurrency(settings)` |
| `src/ede/foundation/jobs/services/celery_app.py` (or new `services/pool.py`) | New `compute_prefork_concurrency(settings) -> int` helper |

---

## Pre-flight

- [ ] **P1: Confirm Phase 2 Slice 1 is at HEAD.**
    ```bash
    git log --oneline -1 src/ede/foundation/jobs/services/clock.py
    ```
    Expected: shows `00b7a58` (Phase 2 Slice 1) or later jobs commit.

- [ ] **P2: Confirm jobs suite is green.**
    ```bash
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ```
    Expected: 89 passed.

---

## Task 1: Auto-pool helper + setting

**Files:**
- Create: `src/ede/foundation/jobs/services/pool.py`
- Modify: `src/ede/foundation/settings.py` — add `JOBS_RUNNER_POOL_AUTOSIZE: bool = False`
- Create: `src/tests/foundation/jobs/test_pool_autosize.py`

- [ ] **Step 1.1: Failing test first**

    Create `src/tests/foundation/jobs/test_pool_autosize.py`:
    ```python
    """Tests for compute_prefork_concurrency — pure math, no I/O."""
    from unittest.mock import patch

    from ede.foundation.jobs.services.pool import compute_prefork_concurrency
    from ede.foundation.settings import FoundationSettings


    def test_autosize_disabled_returns_static_setting():
        """JOBS_RUNNER_POOL_AUTOSIZE=False → returns JOBS_CELERY_PREFORK_CONCURRENCY verbatim."""
        s = FoundationSettings()
        s.JOBS_RUNNER_POOL_AUTOSIZE = False
        s.JOBS_CELERY_PREFORK_CONCURRENCY = 7
        assert compute_prefork_concurrency(s) == 7


    def test_autosize_enabled_uses_cpu_count():
        """JOBS_RUNNER_POOL_AUTOSIZE=True → returns multiprocessing.cpu_count()."""
        s = FoundationSettings()
        s.JOBS_RUNNER_POOL_AUTOSIZE = True
        with patch("multiprocessing.cpu_count", return_value=16):
            assert compute_prefork_concurrency(s) == 16


    def test_autosize_clamps_low():
        """cpu_count() < 1 → clamped to 1."""
        s = FoundationSettings()
        s.JOBS_RUNNER_POOL_AUTOSIZE = True
        with patch("multiprocessing.cpu_count", return_value=0):
            assert compute_prefork_concurrency(s) == 1


    def test_autosize_clamps_high():
        """cpu_count() > 32 → clamped to 32 (avoid runaway pools on big servers)."""
        s = FoundationSettings()
        s.JOBS_RUNNER_POOL_AUTOSIZE = True
        with patch("multiprocessing.cpu_count", return_value=128):
            assert compute_prefork_concurrency(s) == 32


    def test_default_setting_is_false():
        """Autosize is opt-in — default off so production behaviour is stable."""
        assert FoundationSettings().JOBS_RUNNER_POOL_AUTOSIZE is False
    ```

    Run — expect ModuleNotFoundError + AttributeError:
    ```bash
    pytest src/tests/foundation/jobs/test_pool_autosize.py -v
    ```

- [ ] **Step 1.2: Add the setting**

    Open `src/ede/foundation/settings.py`. Find the existing `JOBS_CELERY_PREFORK_CONCURRENCY: int = 4` line (in the `# ── Background Jobs Engine ──` block). Add immediately after it:
    ```python
    JOBS_RUNNER_POOL_AUTOSIZE: bool = False
    ```

- [ ] **Step 1.3: Write the helper**

    Create `src/ede/foundation/jobs/services/pool.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Auto-sized worker pool — computes the actual prefork concurrency.

    When JOBS_RUNNER_POOL_AUTOSIZE is False (default), returns the static
    JOBS_CELERY_PREFORK_CONCURRENCY setting. When True, returns
    multiprocessing.cpu_count() clamped to [1, 32] to avoid runaway pools
    on large servers.
    """
    from __future__ import annotations

    import multiprocessing
    from typing import TYPE_CHECKING

    if TYPE_CHECKING:
        from ede.foundation.settings import FoundationSettings


    _MIN_CONCURRENCY = 1
    _MAX_CONCURRENCY = 32


    def compute_prefork_concurrency(settings: "FoundationSettings") -> int:
        """Return the actual prefork concurrency value for `ede jobs-worker`.

        Respects JOBS_RUNNER_POOL_AUTOSIZE — when False, returns the static
        JOBS_CELERY_PREFORK_CONCURRENCY value verbatim. When True, computes
        from multiprocessing.cpu_count() clamped to [1, 32].
        """
        if not settings.JOBS_RUNNER_POOL_AUTOSIZE:
            return int(settings.JOBS_CELERY_PREFORK_CONCURRENCY)

        cpu = multiprocessing.cpu_count()
        if cpu < _MIN_CONCURRENCY:
            return _MIN_CONCURRENCY
        if cpu > _MAX_CONCURRENCY:
            return _MAX_CONCURRENCY
        return cpu
    ```

- [ ] **Step 1.4: Wire into `ede jobs-worker`**

    Open `src/ede/cli/commands/jobs_worker.py`. Find where `--concurrency` is set (likely from `settings.JOBS_CELERY_PREFORK_CONCURRENCY`). Add module-top import:
    ```python
    from ede.foundation.jobs.services.pool import compute_prefork_concurrency
    ```

    Replace the existing concurrency computation with:
    ```python
    concurrency = concurrency or compute_prefork_concurrency(settings)
    ```

    (The `concurrency` variable is presumably the Click option's value — keep allowing CLI override; only fall back to the auto-sized value when not provided.)

- [ ] **Step 1.5: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_pool_autosize.py -v
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ruff check src/ede/foundation/jobs/services/pool.py src/ede/foundation/settings.py src/ede/cli/commands/jobs_worker.py src/tests/foundation/jobs/test_pool_autosize.py
    ```
    Expected: 5 new pool tests PASS, full jobs suite 94 (89 prior + 5 new), ruff clean.

---

## Task 2: Schema additions — `requires_ids` M2M + `tenant_concurrency_limit`

**Files:**
- Modify: `src/ede/foundation/jobs/models/job.py` — add 2 new fields
- Create: `src/ede/foundation/jobs/migrations/versions/<rev>_phase2_slice2.py`

- [ ] **Step 2.1: Add the fields to the model**

    Open `src/ede/foundation/jobs/models/job.py`. Find the existing field declarations (after `active`, before `source`, or wherever fits the existing grouping). Add:
    ```python
    requires_ids = fields.Many2Many(
        "ir.job",
        relation="ir_job_requires",
        column1="job_id",
        column2="requires_job_id",
        label="Requires",
        help="Jobs that must have succeeded in the current scheduling window before this job runs.",
    )
    tenant_concurrency_limit = fields.Integer(
        default=0,
        required=True,
        label="Tenant Concurrency Limit (0 = unlimited)",
        help="Max simultaneous runs of THIS job scoped to its tenant. 0 disables the limit.",
    )
    ```

    **Verify the `fields.Many2Many` signature** — the EDE field-API uses `Many2Many(target_model_key, relation=..., column1=..., column2=...)`. Sample another model using M2M to confirm exact kwargs:
    ```bash
    grep -rnE "fields\.Many2Many\(" src/ede/foundation/ src/ede/core/kernel/fields.py | head -5
    ```

    If the kwargs differ, adjust.

- [ ] **Step 2.2: Generate the migration**

    ```bash
    ede migrate generate -m "phase2_slice2_dependency_graph_and_tenant_concurrency" --app jobs --config ede.conf
    ```

    Expected: a new file `src/ede/foundation/jobs/migrations/versions/<rev>_phase2_slice2_dependency_graph_and_tenant_concurrency.py`.

    **If `ede migrate generate` errors with multi-head**, follow the `resolving-alembic-multi-heads` skill — likely you need to set `down_revision = "3438ba0d57d1"` manually (Phase 1's `jobs_init.py`).

    Inspect the generated migration. It should:
    - `op.create_table("ir_job_requires", ...)` with two columns `job_id` + `requires_job_id`, both FK → `ir_job.record_uuid` with CASCADE delete
    - `op.add_column("ir_job", sa.Column("tenant_concurrency_limit", sa.Integer(), ..., default=0))`
    - Constraint names ≤ 63 chars (use `op.f(...)` per the `writing-alembic-migrations` skill if autogen produces longer ones)

    If autogen omitted the FK names or used something incorrect, hand-name them. The migration must work on Postgres (production) — SQLite test fixture uses `metadata.create_all` instead.

- [ ] **Step 2.3: Apply + verify**

    Apply to a scratch Postgres tenant if available:
    ```bash
    ede migrate upgrade -t scratch_phase2_s2 --config ede.conf 2>&1 | tail -5
    ```

    Or rely on the existing test suite's `metadata.create_all` path — if all 89 jobs tests still pass after the field additions, the schema is correct.

    Run:
    ```bash
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ```
    Expected: all 94 tests still pass (the 5 from Task 1 + 89 prior).

- [ ] **Step 2.4: Lint**

    ```bash
    ruff check src/ede/foundation/jobs/models/job.py src/ede/foundation/jobs/migrations/versions/
    ```

---

## Task 3: Dependency-graph gate in scheduler

**Files:**
- Modify: `src/ede/foundation/jobs/services/scheduler.py` — add gate after lock acquire
- Create: `src/tests/foundation/jobs/test_dependency_graph.py`

- [ ] **Step 3.1: Failing tests first**

    Create `src/tests/foundation/jobs/test_dependency_graph.py`:
    ```python
    """Scheduler dependency-graph gate — skip if requires_ids haven't succeeded today."""
    from datetime import datetime, timedelta, timezone
    from unittest.mock import MagicMock

    from ede.foundation.jobs.services.clock import FakeClock
    from ede.foundation.jobs.services.scheduler import JobsScheduler


    def _now_minus(seconds: int) -> datetime:
        return datetime.now(tz=timezone.utc) - timedelta(seconds=seconds)


    def test_job_with_satisfied_requires_dispatches(env_with_jobs_and_executor):
        """Required job succeeded today → dependent job dispatches normally."""
        env = env_with_jobs_and_executor

        # Create required job + a successful run today
        required = env.models["ir.job"].create({
            "name": "dep-test.required",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/10 * * * *",
            "source": "runtime",
        })
        env.models["ir.job.run"].create({
            "job_id": required.id,
            "attempt_number": 1,
            "status": "success",
            "finished_at_utc": datetime.now(tz=timezone.utc),
        })

        # Create dependent job that's due now, with requires_ids pointing at required
        dependent = env.models["ir.job"].create({
            "name": "dep-test.dependent",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/10 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
            "requires_ids": [required.id],
        })

        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor)
        assert scheduler.tick(env) == 1
        executor.submit.assert_called_once()


    def test_job_with_unsatisfied_requires_skips(env_with_jobs_and_executor):
        """Required job has NOT succeeded today → dependent job skipped."""
        env = env_with_jobs_and_executor

        required = env.models["ir.job"].create({
            "name": "dep-test.required-no-run",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/10 * * * *",
            "source": "runtime",
        })
        # NO successful ir.job.run for required today

        env.models["ir.job"].create({
            "name": "dep-test.dependent-blocked",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/10 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
            "requires_ids": [required.id],
        })

        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor)
        assert scheduler.tick(env) == 0           # NOT dispatched — required failed/missing
        executor.submit.assert_not_called()


    def test_job_with_no_requires_dispatches(env_with_jobs_and_executor):
        """Empty requires_ids → no gate, dispatches normally."""
        env = env_with_jobs_and_executor

        env.models["ir.job"].create({
            "name": "dep-test.no-requires",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/10 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
            # requires_ids omitted — defaults to empty
        })

        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor)
        assert scheduler.tick(env) == 1
        executor.submit.assert_called_once()
    ```

    Run — expect failures (requires_ids field exists from Task 2 but scheduler doesn't gate on it yet):
    ```bash
    pytest src/tests/foundation/jobs/test_dependency_graph.py -v
    ```

    Specifically `test_job_with_unsatisfied_requires_skips` will fail with `dispatched == 1` (it WILL dispatch because no gate exists yet). Others may pass coincidentally.

- [ ] **Step 3.2: Add the gate in scheduler.tick()**

    Open `src/ede/foundation/jobs/services/scheduler.py`. In the `tick(env)` loop, find the spot AFTER `acquire_lock` succeeds but BEFORE `self._create_pending_run(env, job)` + `self.executor.submit(env, run)`. Insert:

    ```python
                # Dependency-graph gate — skip if any required job hasn't succeeded today
                if not self._requires_satisfied(env, job, now=now):
                    logger.debug("scheduler: skipping %s — requires not satisfied", job.name)
                    release_lock(env, lock_key=lock_key)
                    continue
    ```

    Then add the helper method on `JobsScheduler`:
    ```python
        def _requires_satisfied(self, env: "Env", job: "RecordSet", *, now: datetime) -> bool:
            """Return True if every required job has at least one successful ir.job.run
            with finished_at_utc within today's calendar day (UTC)."""
            requires = job.requires_ids
            if not requires:
                return True

            # Compute the start of today in UTC
            today_start = now.replace(hour=0, minute=0, second=0, microsecond=0)

            run_proxy = env.models["ir.job.run"]
            for required_job in requires:
                successes = run_proxy.search([
                    ("job_id", "=", required_job.id),
                    ("status", "=", "success"),
                    ("finished_at_utc", ">=", today_start),
                ], limit=1)
                if not successes:
                    return False
            return True
    ```

    **Critical**: the `release_lock(env, lock_key=lock_key)` call when skipping is essential — otherwise the lock leaks and the job is blocked indefinitely until the lock expires.

    **Search domain caveat**: `finished_at_utc` is a DateTime field; EDE stores it. The search `(">=", today_start)` should work if the DB column is comparable. If the test fails with a type-comparison error, convert `today_start` to ISO string or check how other code compares DateTime fields in search domains. The Slice 1 implementer noted EDE returns DateTime fields as ISO strings via RecordSet attribute access — but search domains are compiled to SQL where datetimes are passed natively. Should be fine.

- [ ] **Step 3.3: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_dependency_graph.py -v
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ruff check src/ede/foundation/jobs/services/scheduler.py src/tests/foundation/jobs/test_dependency_graph.py
    ```

    Expected: 3 dependency tests PASS; full jobs suite 97 (89 prior + 5 pool + 3 dep + 0 net schema); ruff clean.

    If `test_job_with_satisfied_requires_dispatches` fails because the M2M field doesn't accept a list of IDs in create payload, sample how other tests create records with M2M values and adjust the test fixture syntax.

---

## Task 4: Tenant concurrency gate in scheduler

**Files:**
- Modify: `src/ede/foundation/jobs/services/scheduler.py` — add second gate after dep gate
- Create: `src/tests/foundation/jobs/test_tenant_concurrency.py`

- [ ] **Step 4.1: Failing tests first**

    Create `src/tests/foundation/jobs/test_tenant_concurrency.py`:
    ```python
    """Scheduler tenant-concurrency gate — skip when tenant's running-job count meets the cap."""
    from datetime import datetime, timedelta, timezone
    from unittest.mock import MagicMock

    from ede.foundation.jobs.services.scheduler import JobsScheduler


    def _now_minus(seconds: int) -> datetime:
        return datetime.now(tz=timezone.utc) - timedelta(seconds=seconds)


    def test_dispatch_proceeds_when_limit_is_zero(env_with_jobs_and_executor):
        """tenant_concurrency_limit=0 (default) → unlimited; dispatch proceeds."""
        env = env_with_jobs_and_executor

        env.models["ir.job"].create({
            "name": "tc-test.unlimited",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/10 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
            "tenant_id": "tenant-A",
            # tenant_concurrency_limit defaults to 0
        })

        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor)
        assert scheduler.tick(env) == 1


    def test_dispatch_skipped_when_running_count_meets_limit(env_with_jobs_and_executor):
        """tenant_concurrency_limit=1 + 1 running run for this job → skip."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "tc-test.capped",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/10 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
            "tenant_id": "tenant-B",
            "tenant_concurrency_limit": 1,
        })
        # Seed 1 currently-running run for this job
        env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "running",
        })

        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor)
        assert scheduler.tick(env) == 0           # NOT dispatched
        executor.submit.assert_not_called()


    def test_dispatch_proceeds_when_under_limit(env_with_jobs_and_executor):
        """tenant_concurrency_limit=2 + 1 running run for this job → dispatch."""
        env = env_with_jobs_and_executor

        job = env.models["ir.job"].create({
            "name": "tc-test.under-cap",
            "module_key": "foundation.jobs",
            "target": "src.tests.foundation.jobs.targets.echo_target",
            "kind": "scheduled",
            "cron": "*/10 * * * *",
            "source": "runtime",
            "next_run_at_utc": _now_minus(60),
            "tenant_id": "tenant-C",
            "tenant_concurrency_limit": 2,
        })
        env.models["ir.job.run"].create({
            "job_id": job.id,
            "attempt_number": 1,
            "status": "running",
        })

        executor = MagicMock(spec_set=["submit", "submit_retry", "graceful_shutdown"])
        scheduler = JobsScheduler(executor=executor)
        assert scheduler.tick(env) == 1
    ```

    Run — expect the cap test to fail (no gate yet):
    ```bash
    pytest src/tests/foundation/jobs/test_tenant_concurrency.py -v
    ```

- [ ] **Step 4.2: Add the gate in scheduler.tick()**

    Open `src/ede/foundation/jobs/services/scheduler.py`. After the dependency-graph gate (from Task 3), add:
    ```python
                # Tenant-concurrency gate — skip when this job's running count meets the cap
                if not self._under_tenant_limit(env, job):
                    logger.debug("scheduler: skipping %s — tenant concurrency limit reached", job.name)
                    release_lock(env, lock_key=lock_key)
                    continue
    ```

    Add the helper method:
    ```python
        def _under_tenant_limit(self, env: "Env", job: "RecordSet") -> bool:
            """Return True if this job's running-run count is below tenant_concurrency_limit.

            Limit of 0 (default) means unlimited — always returns True.
            Counts runs across ALL ir.job rows for this job's tenant_id (currently
            scoped per-job; cross-job tenant limits can land in a future phase).
            """
            limit = job.tenant_concurrency_limit
            if not limit or int(limit) <= 0:
                return True

            run_proxy = env.models["ir.job.run"]
            running = run_proxy.search([
                ("job_id", "=", job.id),
                ("status", "=", "running"),
            ])
            # Use len(); some ORMs lack a count() helper, len() works on RecordSet
            return len(running) < int(limit)
    ```

    Note: Slice 2 scopes the count to THIS job's running runs (not all jobs for the tenant). A broader "across all jobs in tenant" interpretation is possible but adds complexity; per-job is the most useful default for the typical "throttle one heavy job per tenant" use case.

- [ ] **Step 4.3: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_tenant_concurrency.py -v
    pytest src/tests/foundation/jobs/ -v 2>&1 | tail -3
    ruff check src/ede/foundation/jobs/services/scheduler.py src/tests/foundation/jobs/test_tenant_concurrency.py
    ```

    Expected: 3 tenant-concurrency tests PASS; full jobs suite 100 (89 prior + 5 pool + 3 dep + 3 tc); ruff clean.

---

## Task 5: Acceptance + roadmap status update + commit

- [ ] **Step 5.1: Full repo regression check**

    ```bash
    ./run_tests.sh 2>&1 | tail -3
    ```
    Expected: exit 0.

- [ ] **Step 5.2: Update roadmap status (Phase 2 stays 🟡 — Slice 3 still pending)**

    Edit the 4 status sites to note "Slices 1+2 ✅":
    - `roadmap/foundation/jobs/phase-2-implementation.md` top Status
    - `roadmap/foundation/jobs/README.md` Phase 2 row in Phased Delivery
    - `roadmap/roadmap-tracker.md` Phase 2 row + prepend Last refreshed entry
    - `docs/foundation-jobs.md` Status header + Phase 2 row in Status Snapshot + Built Capabilities row + Last sync

- [ ] **Step 5.3: Append PROGRESS row**

    Dated 2026-05-19, theme: `foundation.jobs Phase 2 Slice 2 🟡 — auto-pool + dependency graph + tenant concurrency`. ~1000 lines, ~2.5-3 hrs.

- [ ] **Step 5.4: Stage + commit**

    Stage only the new files + modified production files + roadmap/docs/PROGRESS. Commit with the standard `[IMP] foundation.jobs Phase 2 Slice 2 (🟡): ...` shape.

---

## Self-Review

**1. Spec coverage:**
- ✅ WS-J12 auto-sized pool — Task 1
- ✅ WS-J13 dependency graph — Tasks 2 (schema) + 3 (scheduler gate)
- ✅ WS-J14 per-tenant concurrency — Tasks 2 (schema) + 4 (scheduler gate)

**2. Placeholder scan:**
- The "If `ede migrate generate` errors..." callout in Task 2 is a fallback instruction with explicit code, not a TBD.
- The "Search domain caveat" callout in Task 3 is verification-before-paste guidance, not a placeholder.

**3. Type consistency:**
- `compute_prefork_concurrency(settings) -> int` matches definition + use in CLI command.
- `_requires_satisfied(env, job, *, now: datetime) -> bool` + `_under_tenant_limit(env, job) -> bool` methods consistent.
- Schema field names (`requires_ids`, `tenant_concurrency_limit`) consistent across model + migration + scheduler + tests.

---

## Execution Handoff

Subagent-driven execution per the established pattern. 5 tasks (4 implementation + 1 acceptance/commit).
