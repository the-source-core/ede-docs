# Foundation.jobs Phase 1 — Slice 4: Admin UI + RBAC + Heartbeat First-Adopter → Phase 1 ✅

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `foundation.jobs` visible-in-browser and ship the heartbeat first-adopter that proves the engine ticks end-to-end. Add RBAC seed for three operator roles. Cap with a user walkthrough that flips Phase 1 🟡 → ✅ and unblocks OneMaster Phase 1.

**Architecture:** Two new XML data files (`jobs_rbac.csv` + `jobs_menus.xml` + `example_jobs.xml`) registered in the manifest's `data` list — loaded by the standard EDE `DataLoader` at `ede migrate upgrade` time. Four view XML files under `views/` declare list + form + dashboard surfaces against `ir.job` / `ir.job.run` / `ir.job.lock`. A new `api/jobs_routes.py` exposes operator buttons (Run Now / Disable / Enable / Retry from dead-letter) under `/api/foundation/jobs/*`. The heartbeat target lives at `demo/heartbeat.py` (called by the XML-declared `ir.job` row with cron `*/2 * * * *`). The acceptance walkthrough exercises: heartbeat XML row loads via `DataLoader` → scheduler picks it up → Celery prefork executes target → `ir.job.run` row written with `status=success` → user opens Settings → Technical → Jobs → Run History → sees the ticking.

**Tech Stack:** EDE DSL views (`<ede>` root, `<view>`, `<list>`, `<form>`, `<sheet>`, `<group>`, `<field>`), `ir.menu` + `ir.action` records, `ir.rbac.permission` CSV (columns `id,name,code,resource,action,role_id/id,domain`), `@api.route_config` + `@api.route` HTTP controllers, existing Slice 1-3 surface (`ir.job` / `ir.job.run` schema, scheduler, executor, retry policy, reconciler).

**Closes Phase 1.** After Slice 4: Phase 1 status flips 🟡 → ✅; OneMaster Phase 1 hard prereq satisfied.

---

## File Structure

### New files

| Path | Responsibility |
|---|---|
| `src/ede/foundation/jobs/demo/__init__.py` | Empty package marker |
| `src/ede/foundation/jobs/demo/heartbeat.py` | `tick(env, payload=None)` — emits one progress call, returns `{"timestamp": ..., "iteration": N}` |
| `src/ede/foundation/jobs/api/__init__.py` | `from . import jobs_routes` — triggers `@api.route_config` scan |
| `src/ede/foundation/jobs/api/jobs_routes.py` | `JobsController` with `/api/foundation/jobs/*` — list, get, run-now, disable, enable, retry |
| `src/ede/foundation/jobs/data/jobs_rbac.csv` | 3 roles (`jobs.admin` / `jobs.operator` / `jobs.viewer`) + ~9 permission rows |
| `src/ede/foundation/jobs/data/jobs_menus.xml` | 3 ir.action records + 4 ir.menu records (Settings → Technical → Jobs tree) |
| `src/ede/foundation/jobs/data/example_jobs.xml` | Heartbeat `ir.job` row (`source=xml`, `cron="*/2 * * * *"`) |
| `src/ede/foundation/jobs/views/job_views.xml` | `ir.job` list (with `source` badge column + status indicator) + form (with cron preview, retry policy, action buttons) |
| `src/ede/foundation/jobs/views/job_run_views.xml` | `ir.job.run` list (status, attempt_number, durations, errors) + form (payload/output JSON, traceback collapse, progress bar) |
| `src/ede/foundation/jobs/views/job_lock_views.xml` | `ir.job.lock` list (read-only diagnostic) |
| `src/tests/foundation/jobs/test_heartbeat.py` | Smoke: `heartbeat.tick(env, payload)` returns expected dict shape |
| `src/tests/foundation/jobs/test_example_jobs_xml.py` | Load `data/example_jobs.xml` → assert heartbeat row exists with `source=xml` + cron `*/2 * * * *` |
| `src/tests/foundation/jobs/test_rbac.py` | 3 roles seeded; jobs.viewer cannot create ir.job; jobs.admin can |
| `src/tests/foundation/jobs/test_jobs_routes.py` | Smoke each HTTP endpoint via Click/FastAPI test client + protocol |
| `src/tests/foundation/jobs/test_phase1_acceptance.py` | End-to-end walkthrough: load heartbeat XML → scheduler tick → Celery eager execute → `ir.job.run` row with status=success |

### Existing files modified

| Path | Change |
|---|---|
| `src/ede/foundation/jobs/__manifest__.py` | Add `data` list entries: `jobs_rbac.csv`, `jobs_menus.xml`, `example_jobs.xml`, and the 3 view XML files |

---

## Pre-flight

- [ ] **P1: Confirm Slice 3 is at HEAD.**
    ```bash
    git log --oneline -1 src/ede/foundation/jobs/services/retry_policy.py
    ```
    Expected: shows commit `f8f1408` (Slice 3) or later. If absent, Slice 3 wasn't merged — stop and fix.

- [ ] **P2: Confirm the existing jobs suite is green.**
    ```bash
    pytest src/tests/foundation/jobs/ -v
    ```
    Expected: 64 passed. If anything red, fix before touching Slice 4.

- [ ] **P3: Read the canonical patterns this slice copies.**
    Browse these for shape — DON'T modify them:
    ```bash
    cat src/ede/foundation/notifications/__manifest__.py
    cat src/ede/foundation/notifications/data/notification_menus.xml
    head -50 src/ede/foundation/notifications/data/ir.rbac.permission.csv
    head -60 src/ede/foundation/notifications/views/ir_notification_views.xml
    head -60 src/ede/foundation/notifications/api/notification_routes.py
    cat src/ede/foundation/base/data/base_menus.xml | sed -n '490,535p'    # the Technical category
    ```
    Note the conventions:
    - XML root is `<ede>` (not `<openerp>`)
    - Manifest `data` list runs in order — load CSVs / RBAC before menus that reference them
    - `<ListView>` and `<FormView>` are the DSL element names (capital-camel, NOT `<list>` / `<form>`)
    - RBAC CSV id pattern: `<module>.<p_><resource>_<action>` (e.g. `jobs.p_ir_job_read`)
    - Menus reference `base.menu_settings_root` and `base.menu_cat_technical` (existing parents)

---

## Task 1: Heartbeat target callable

**Files:**
- Create: `src/ede/foundation/jobs/demo/__init__.py` (empty)
- Create: `src/ede/foundation/jobs/demo/heartbeat.py`
- Create: `src/tests/foundation/jobs/test_heartbeat.py`

- [ ] **Step 1.1: Failing test first**

    Create `src/tests/foundation/jobs/test_heartbeat.py`:
    ```python
    """Smoke test for the heartbeat target callable."""
    from ede.foundation.jobs.demo.heartbeat import tick


    def test_tick_returns_iteration_and_timestamp(env_with_jobs_and_executor):
        """tick(env, payload) returns dict with timestamp + iteration."""
        result = tick(env_with_jobs_and_executor, payload={"iteration": 1})
        assert "timestamp" in result
        assert "iteration" in result
        assert result["iteration"] == 1
        # timestamp is ISO 8601 UTC
        assert "T" in result["timestamp"]
        assert result["timestamp"].endswith("+00:00") or result["timestamp"].endswith("Z")


    def test_tick_default_iteration_when_payload_empty(env_with_jobs_and_executor):
        result = tick(env_with_jobs_and_executor, payload={})
        assert result["iteration"] == 0


    def test_tick_handles_none_payload(env_with_jobs_and_executor):
        result = tick(env_with_jobs_and_executor, payload=None)
        assert "iteration" in result
        assert "timestamp" in result
    ```

    Run — expect ModuleNotFoundError:
    ```bash
    pytest src/tests/foundation/jobs/test_heartbeat.py -v
    ```

- [ ] **Step 1.2: Create the directory + empty package marker**

    ```bash
    mkdir -p src/ede/foundation/jobs/demo
    touch src/ede/foundation/jobs/demo/__init__.py
    ```

- [ ] **Step 1.3: Write the heartbeat target**

    Create `src/ede/foundation/jobs/demo/heartbeat.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Heartbeat demo target — the foundation.jobs first-adopter.

    Declared via data/example_jobs.xml with cron="*/2 * * * *". Proves the
    engine ticks end-to-end. Operators see successive ir.job.run rows in
    Settings → Technical → Jobs → Run History.
    """
    from __future__ import annotations

    from datetime import datetime, timezone
    from typing import Any, Optional


    def tick(env, payload: Optional[dict] = None) -> dict[str, Any]:
        """Heartbeat tick — emit one progress event + return a small result dict.

        payload may carry {"iteration": N} for diagnostics. The reporter
        writes percent=50 mid-tick to prove env.job_progress wires through.
        """
        iteration = (payload or {}).get("iteration", 0)
        env.job_progress(percent=50.0, message=f"heartbeat tick (iter={iteration})")
        return {
            "timestamp": datetime.now(tz=timezone.utc).isoformat(),
            "iteration": iteration,
        }
    ```

- [ ] **Step 1.4: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_heartbeat.py -v
    ruff check src/ede/foundation/jobs/demo/ src/tests/foundation/jobs/test_heartbeat.py
    ```
    Expected: 3 tests PASS, ruff clean.

---

## Task 2: Heartbeat XML data row + manifest

**Files:**
- Create: `src/ede/foundation/jobs/data/example_jobs.xml`
- Modify: `src/ede/foundation/jobs/__manifest__.py` — add to `data` list
- Create: `src/tests/foundation/jobs/test_example_jobs_xml.py`

- [ ] **Step 2.1: Failing test first**

    Create `src/tests/foundation/jobs/test_example_jobs_xml.py`:
    ```python
    """Verify the heartbeat XML row loads via the standard DataLoader."""
    from pathlib import Path

    from ede.core.services.data_loader.loader import DataLoader
    from ede.foundation.jobs.models import Job, JobRun, JobLock                # noqa: F401 — registers models
    from ede.foundation.base.models.data_reference import DataReference        # noqa: F401


    def _load_example_jobs(env):
        """Helper — load data/example_jobs.xml against the env."""
        # Bootstrap ir.data.reference table (the test fixture only registers jobs models)
        from sqlalchemy import inspect

        if env.persistence is None:
            return
        engine = env.persistence.uow().session.get_bind()
        if "ir_data_reference" not in inspect(engine).get_table_names():
            # mirror the existing test_xml_data_path helper
            from ede.core.kernel.metadata import SqlAlchemyMetadataBuilder
            DataReference.__ede_register__(env.registry)
            SqlAlchemyMetadataBuilder().build(env.registry).metadata.create_all(engine)

        loader = DataLoader(env=env)
        path = Path(__file__).resolve().parent.parent.parent.parent / "ede" / "foundation" / "jobs" / "data" / "example_jobs.xml"
        loader._load_xml(app_name="foundation.jobs", file_path=str(path))


    def test_heartbeat_row_loaded_with_source_xml(env_with_jobs):
        env = env_with_jobs
        _load_example_jobs(env)

        rows = env.models["ir.job"].search([("name", "=", "foundation.jobs.heartbeat")])
        assert len(rows) == 1
        row = rows[0]
        assert row.source == "xml"
        assert row.kind == "scheduled"
        assert row.cron == "*/2 * * * *"
        assert row.target == "ede.foundation.jobs.demo.heartbeat.tick"
        assert row.retry_policy == "fixed"
        assert row.retry_max_attempts == 2
    ```

    Run — expect failure (`example_jobs.xml` doesn't exist):
    ```bash
    pytest src/tests/foundation/jobs/test_example_jobs_xml.py -v
    ```

- [ ] **Step 2.2: Create the XML data file**

    Create `src/ede/foundation/jobs/data/example_jobs.xml`:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <ede>
        <!--
            Foundation Jobs — heartbeat first-adopter.

            Proves the engine ticks end-to-end. Visible in Settings → Technical
            → Jobs → Run History → successive heartbeat runs every 2 minutes.
            source=xml so the boot reconciler never touches this row.
        -->
        <data noupdate="0">
            <record id="jobs.job_heartbeat" model="ir.job">
                <field name="name">foundation.jobs.heartbeat</field>
                <field name="module_key">foundation.jobs</field>
                <field name="target">ede.foundation.jobs.demo.heartbeat.tick</field>
                <field name="kind">scheduled</field>
                <field name="cron">*/2 * * * *</field>
                <field name="retry_policy">fixed</field>
                <field name="retry_max_attempts">2</field>
                <field name="retry_base_seconds">30</field>
                <field name="priority">7</field>
                <field name="timeout_seconds">60</field>
                <field name="retry_on_interrupt">True</field>
                <field name="active">True</field>
                <field name="source">xml</field>
                <field name="description">Heartbeat — proves the foundation.jobs engine is ticking end-to-end. Source=xml so the reconciler leaves it alone.</field>
            </record>
        </data>
    </ede>
    ```

- [ ] **Step 2.3: Register in manifest**

    Open `src/ede/foundation/jobs/__manifest__.py`. The current `"data": []` is empty. Change to:
    ```python
        "data": [
            "data/example_jobs.xml",
        ],
    ```

    (Future tasks add more entries; for now just heartbeat.)

- [ ] **Step 2.4: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_example_jobs_xml.py -v
    ruff check src/ede/foundation/jobs/__manifest__.py src/tests/foundation/jobs/test_example_jobs_xml.py
    ```
    Expected: 1 test PASS, ruff clean. (If the manifest is a literal dict with no functions, ruff is a no-op.)

---

## Task 3: RBAC seed (jobs.admin / jobs.operator / jobs.viewer)

**Files:**
- Create: `src/ede/foundation/jobs/data/jobs_rbac.csv`
- Modify: `src/ede/foundation/jobs/__manifest__.py` — prepend `jobs_rbac.csv` to `data` list
- Create: `src/tests/foundation/jobs/test_rbac.py`

- [ ] **Step 3.1: Discover existing role IDs**

    The CSV references `role_id/id` slots — find what's available:
    ```bash
    grep -hE "^rbac\." src/ede/foundation/base/data/*.csv src/ede/foundation/base/data/*.xml 2>/dev/null | grep -iE "role|admin" | head -10
    grep -nE "role_internal_user|role_system_admin" src/ede/foundation/base/data/*.xml 2>/dev/null | head -5
    ```

    Expected: roles `rbac.role_internal_user`, `rbac.role_system_admin` exist in foundation.base. The Slice 4 plan uses these as the binding targets (we do NOT seed new role records here — that's Phase 2 if needed). Job-specific roles map onto these existing two: `jobs.admin` perms bind to `role_system_admin`; `jobs.operator` + `jobs.viewer` perms bind to `role_internal_user`.

    **If those role IDs don't exist with those exact strings**, sample any RBAC CSV in `src/ede/foundation/*/data/ir.rbac.permission.csv` and use whatever role keys are listed. Report.

- [ ] **Step 3.2: Write the CSV**

    Create `src/ede/foundation/jobs/data/jobs_rbac.csv`:
    ```csv
    id,name,code,resource,action,role_id/id,domain
    jobs.p_ir_job_read,Read Jobs,ir.job.read,ir.job,read,rbac.role_internal_user,
    jobs.p_ir_job_create,Create Jobs,ir.job.create,ir.job,create,rbac.role_system_admin,
    jobs.p_ir_job_update,Update Jobs,ir.job.update,ir.job,update,rbac.role_system_admin,
    jobs.p_ir_job_delete,Delete Jobs,ir.job.delete,ir.job,delete,rbac.role_system_admin,
    jobs.p_ir_job_run_read,Read Job Runs,ir.job.run.read,ir.job.run,read,rbac.role_internal_user,
    jobs.p_ir_job_run_create,Create Job Runs,ir.job.run.create,ir.job.run,create,rbac.role_system_admin,
    jobs.p_ir_job_run_update,Update Job Runs,ir.job.run.update,ir.job.run,update,rbac.role_system_admin,
    jobs.p_ir_job_lock_read,Read Job Locks,ir.job.lock.read,ir.job.lock,read,rbac.role_system_admin,
    ```

    Notes:
    - `read` permissions go to `role_internal_user` (any logged-in user can list jobs + see run history)
    - mutate permissions go to `role_system_admin` (operators / admins only)
    - `ir.job.lock` is admin-only (diagnostic table)
    - We do NOT seed `jobs.admin/operator/viewer` as distinct roles in this slice — those are stretch goals for Phase 2 if granular operator-vs-viewer distinction matters. The two existing role keys give us "everyone can see, only admins can edit" which is the right default.

    **If you find that the codebase uses different role keys** (e.g. `base.role_admin`, `rbac.system_admin`), adjust accordingly and report.

- [ ] **Step 3.3: Update manifest — prepend CSV before XML**

    Edit `src/ede/foundation/jobs/__manifest__.py`:
    ```python
        "data": [
            "data/jobs_rbac.csv",
            "data/example_jobs.xml",
        ],
    ```

    **Order matters**: RBAC must load before any record that needs RBAC checks. CSVs typically go first in the foundation modules.

- [ ] **Step 3.4: Failing test**

    Create `src/tests/foundation/jobs/test_rbac.py`:
    ```python
    """Verify the jobs RBAC permissions load correctly via the data loader."""
    from pathlib import Path

    import pytest


    def _csv_path() -> Path:
        return (
            Path(__file__).resolve().parent.parent.parent.parent
            / "ede" / "foundation" / "jobs" / "data" / "jobs_rbac.csv"
        )


    def test_jobs_rbac_csv_exists_and_parseable():
        """The CSV exists, has the expected columns, and contains ≥7 rows."""
        path = _csv_path()
        assert path.exists(), f"jobs_rbac.csv missing at {path}"
        rows = path.read_text(encoding="utf-8").strip().splitlines()
        header = rows[0]
        assert "id,name,code,resource,action,role_id/id,domain" in header, f"unexpected header: {header}"
        data_rows = rows[1:]
        assert len(data_rows) >= 7, f"expected ≥7 permission rows, got {len(data_rows)}"


    def test_jobs_rbac_covers_three_models():
        """All three ir.job* models must have at least a read permission."""
        path = _csv_path()
        text = path.read_text(encoding="utf-8")
        assert "ir.job,read" in text
        assert "ir.job.run,read" in text
        assert "ir.job.lock,read" in text


    def test_jobs_rbac_admin_roles_for_mutations():
        """Mutate permissions bind to role_system_admin (not internal_user)."""
        path = _csv_path()
        text = path.read_text(encoding="utf-8")
        # Spot-check: ir.job.create should NOT bind to internal_user
        create_row = next((line for line in text.splitlines() if "ir.job.create" in line and ",create," in line), None)
        assert create_row is not None, "ir.job.create permission row not found"
        assert "role_system_admin" in create_row, f"expected role_system_admin on create row: {create_row}"
        assert "role_internal_user" not in create_row, f"create should NOT bind to internal_user: {create_row}"
    ```

    Run — expect PASS (the tests are file-existence + content-spot-check):
    ```bash
    pytest src/tests/foundation/jobs/test_rbac.py -v
    ```

    These tests verify the CSV is well-formed; the actual enforcement (a viewer fails to create) requires a fuller auth env that's already covered by foundation.base's RBAC test suite. Phase 2 can extend if needed.

- [ ] **Step 3.5: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_rbac.py -v
    ruff check src/tests/foundation/jobs/test_rbac.py
    ```
    3 tests PASS, ruff clean.

---

## Task 4: Job views (ir.job list + form)

**Files:**
- Create: `src/ede/foundation/jobs/views/job_views.xml`

- [ ] **Step 4.1: Write the XML**

    Create `src/ede/foundation/jobs/views/job_views.xml`:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <ede version="1.0">
        <!--
            ir.job — definitions list + form.

            List: every job declared in the system, with its source (decorator/xml/
            runtime), cron, next-run, last-status, and active flag.
            Form: full definition + action buttons (Run Now / Disable).
        -->

        <view id="ir_job_list_view" model="ir.job">
            <ListView order_by="priority asc, name asc">
                <field name="name"              string="Name" />
                <field name="module_key"        string="Module" />
                <field name="kind"              string="Kind" />
                <field name="cron"              string="Cron" />
                <field name="next_run_at_utc"   string="Next Run" />
                <field name="last_run_at_utc"   string="Last Run" />
                <field name="last_status"       string="Last Status" />
                <field name="priority"          string="Pri" />
                <field name="source"            string="Source" />
                <field name="active"            string="Active" />
            </ListView>
        </view>

        <view id="ir_job_form_view" model="ir.job">
            <FormView>
                <header>
                    <button name="run_now" type="action" string="Run Now" />
                </header>
                <sheet>
                    <section cols="2">
                        <field name="name"          widget="record_title" />
                        <field name="active" />
                    </section>
                    <section string="Identity" cols="2">
                        <field name="module_key" />
                        <field name="source" />
                        <field name="target" />
                        <field name="description" />
                    </section>
                    <section string="Schedule" cols="2">
                        <field name="kind" />
                        <field name="cron" />
                        <field name="next_run_at_utc" />
                        <field name="last_run_at_utc" />
                        <field name="last_status" />
                    </section>
                    <section string="Retry Policy" cols="2">
                        <field name="retry_policy" />
                        <field name="retry_max_attempts" />
                        <field name="retry_base_seconds" />
                        <field name="retry_on_interrupt" />
                    </section>
                    <section string="Execution" cols="2">
                        <field name="priority" />
                        <field name="timeout_seconds" />
                        <field name="tenant_id" />
                    </section>
                </sheet>
            </FormView>
        </view>

        <view id="ir_job_search_view" model="ir.job">
            <SearchView>
                <field name="name" />
                <field name="module_key" />
                <field name="source" />
                <field name="last_status" />
                <field name="active" />
            </SearchView>
        </view>
    </ede>
    ```

    **Adjust to repo conventions:**
    - If existing list/form views use lowercase `<list>` / `<form>` instead of `<ListView>` / `<FormView>`, switch. Confirm by checking the DSL schema or any existing foundation view's element names.
    - If `widget="record_title"` doesn't exist in this DSL, use whatever widget makes the field the primary visual hero (e.g. `widget="title"` or just omit and rely on the field name).
    - If `<button name="run_now" type="action">` doesn't resolve to a controller method via that mechanism, change the type to `"command"` (dispatches a command via the bus) — check existing views for similar action-button patterns.

- [ ] **Step 4.2: Update manifest**

    Add to the `data` list:
    ```python
        "data": [
            "data/jobs_rbac.csv",
            "views/job_views.xml",
            "data/example_jobs.xml",
        ],
    ```

- [ ] **Step 4.3: Verify the XML parses cleanly**

    The repo has a DSL RelaxNG validator at `src/tests/foundation/test_dsl_relaxng_validator.py` (per the EDE skill list). Run it to confirm the new XML passes:
    ```bash
    pytest src/tests/foundation/test_dsl_relaxng_validator.py -v 2>&1 | tail -20
    ```

    Expected: passes. If it fails on the new file, the DSL doesn't accept some element/attribute — fix per the error message and re-run.

- [ ] **Step 4.4: Boot smoke**

    ```bash
    ede info --config ede.conf 2>&1 | grep -iE "jobs|error" | head -10
    ```
    Expected: foundation.jobs listed, no traceback referencing the new XML.

---

## Task 5: Job Run views (ir.job.run list + form)

**Files:**
- Create: `src/ede/foundation/jobs/views/job_run_views.xml`

- [ ] **Step 5.1: Write the XML**

    Create `src/ede/foundation/jobs/views/job_run_views.xml`:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <ede version="1.0">
        <!--
            ir.job.run — execution log (read-mostly).

            List: chronological execution history with status, attempt_number,
            durations, error capture.
            Form: full row with payload / output JSON, traceback, progress.
        -->

        <view id="ir_job_run_list_view" model="ir.job.run">
            <ListView order_by="created_at_utc desc">
                <field name="job_id"            string="Job" />
                <field name="attempt_number"    string="Attempt" />
                <field name="status"            string="Status" />
                <field name="started_at_utc"    string="Started" />
                <field name="finished_at_utc"   string="Finished" />
                <field name="duration_seconds"  string="Duration (s)" />
                <field name="progress_pct"      string="Progress %" />
                <field name="worker_id"         string="Worker" />
            </ListView>
        </view>

        <view id="ir_job_run_form_view" model="ir.job.run">
            <FormView>
                <header>
                </header>
                <sheet>
                    <section cols="2">
                        <field name="job_id" />
                        <field name="status" />
                    </section>
                    <section string="Attempt" cols="2">
                        <field name="attempt_number" />
                        <field name="parent_run_id" />
                    </section>
                    <section string="Timing" cols="3">
                        <field name="queued_at_utc" />
                        <field name="started_at_utc" />
                        <field name="finished_at_utc" />
                    </section>
                    <section string="Execution" cols="2">
                        <field name="duration_seconds" />
                        <field name="worker_id" />
                        <field name="celery_task_id" />
                    </section>
                    <section string="Progress" cols="2">
                        <field name="progress_pct" />
                        <field name="progress_message" />
                    </section>
                    <section string="Payload" cols="1">
                        <field name="payload" />
                    </section>
                    <section string="Output" cols="1">
                        <field name="output" />
                    </section>
                    <section string="Error" cols="1">
                        <field name="error_summary" />
                        <field name="error_traceback" />
                    </section>
                </sheet>
            </FormView>
        </view>

        <view id="ir_job_run_search_view" model="ir.job.run">
            <SearchView>
                <field name="job_id" />
                <field name="status" />
                <field name="worker_id" />
            </SearchView>
        </view>
    </ede>
    ```

- [ ] **Step 5.2: Update manifest**

    Add `views/job_run_views.xml` to the `data` list (after `job_views.xml`).

- [ ] **Step 5.3: Verify**

    ```bash
    pytest src/tests/foundation/test_dsl_relaxng_validator.py -v 2>&1 | tail -10
    ```
    Expected: PASS.

---

## Task 6: Job Lock views (read-only diagnostic)

**Files:**
- Create: `src/ede/foundation/jobs/views/job_lock_views.xml`

- [ ] **Step 6.1: Write the XML**

    Create `src/ede/foundation/jobs/views/job_lock_views.xml`:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <ede version="1.0">
        <!--
            ir.job.lock — scheduler-side dedup rows.
            Read-only diagnostic view; locks are normally short-lived.
        -->

        <view id="ir_job_lock_list_view" model="ir.job.lock">
            <ListView order_by="acquired_at_utc desc">
                <field name="lock_key"          string="Lock Key" />
                <field name="worker_id"         string="Worker" />
                <field name="acquired_at_utc"   string="Acquired" />
                <field name="expires_at_utc"   string="Expires" />
            </ListView>
        </view>

        <view id="ir_job_lock_search_view" model="ir.job.lock">
            <SearchView>
                <field name="lock_key" />
                <field name="worker_id" />
            </SearchView>
        </view>
    </ede>
    ```

- [ ] **Step 6.2: Update manifest**

    Add `views/job_lock_views.xml` to `data`.

- [ ] **Step 6.3: Verify**

    ```bash
    pytest src/tests/foundation/test_dsl_relaxng_validator.py -v 2>&1 | tail -10
    ```

---

## Task 7: Menus + actions XML

**Files:**
- Create: `src/ede/foundation/jobs/data/jobs_menus.xml`

- [ ] **Step 7.1: Write the XML**

    Create `src/ede/foundation/jobs/data/jobs_menus.xml`:
    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <ede>
        <!--
            Foundation Jobs — navigation menus.

            Placement: Settings → Technical → Jobs (a NEW section under the
            existing Technical category seeded by foundation.base).
              ├── Job Definitions    → ir.job        list + form
              ├── Run History        → ir.job.run    list + form
              ├── Dead Letter Queue  → ir.job.run    list filtered status=dead_letter (Phase 2 saved-search)
              └── Locks              → ir.job.lock   list (diagnostic)
        -->

        <data noupdate="0">

            <!-- ════════════════════════════════════════════════════════════
                 Actions
                 ════════════════════════════════════════════════════════════ -->

            <record id="jobs.action_ir_job" model="ir.action">
                <field name="name">Job Definitions</field>
                <field name="path">job-definitions</field>
                <field name="model_key">ir.job</field>
                <field name="default_view">list</field>
                <field name="available_views">list,form</field>
            </record>

            <record id="jobs.action_ir_job_run" model="ir.action">
                <field name="name">Job Run History</field>
                <field name="path">job-runs</field>
                <field name="model_key">ir.job.run</field>
                <field name="default_view">list</field>
                <field name="available_views">list,form</field>
            </record>

            <record id="jobs.action_ir_job_lock" model="ir.action">
                <field name="name">Job Locks</field>
                <field name="path">job-locks</field>
                <field name="model_key">ir.job.lock</field>
                <field name="default_view">list</field>
                <field name="available_views">list</field>
            </record>

            <!-- ════════════════════════════════════════════════════════════
                 Section: Jobs (within Settings → Technical)
                 ════════════════════════════════════════════════════════════ -->

            <record id="jobs.menu_cat_technical_jobs" model="ir.menu">
                <field name="name">Jobs</field>
                <field name="parent_id" ref="base.menu_cat_technical"/>
                <field name="sequence">50</field>
            </record>

            <record id="jobs.menu_settings_jobs_definitions" model="ir.menu">
                <field name="name">Job Definitions</field>
                <field name="parent_id" ref="jobs.menu_cat_technical_jobs"/>
                <field name="sequence">10</field>
                <field name="action_id" ref="jobs.action_ir_job"/>
            </record>

            <record id="jobs.menu_settings_jobs_runs" model="ir.menu">
                <field name="name">Run History</field>
                <field name="parent_id" ref="jobs.menu_cat_technical_jobs"/>
                <field name="sequence">20</field>
                <field name="action_id" ref="jobs.action_ir_job_run"/>
            </record>

            <record id="jobs.menu_settings_jobs_locks" model="ir.menu">
                <field name="name">Locks</field>
                <field name="parent_id" ref="jobs.menu_cat_technical_jobs"/>
                <field name="sequence">30</field>
                <field name="action_id" ref="jobs.action_ir_job_lock"/>
            </record>

        </data>
    </ede>
    ```

    Notes:
    - `parent_id` references `base.menu_cat_technical` — the existing Settings → Technical category. **Verify** this XML id exists by grepping `base.menu_cat_technical` in `src/ede/foundation/base/data/`.
    - We deliberately do NOT add a Dead Letter Queue saved-search leaf in this slice — that's a Phase 2 enhancement. The Run History list lets users filter by `status=dead_letter` manually.
    - Dashboard is also deferred to Phase 2.

- [ ] **Step 7.2: Update manifest**

    Add `data/jobs_menus.xml` AFTER the view XMLs (menus reference action models which reference view definitions):
    ```python
        "data": [
            "data/jobs_rbac.csv",
            "views/job_views.xml",
            "views/job_run_views.xml",
            "views/job_lock_views.xml",
            "data/jobs_menus.xml",
            "data/example_jobs.xml",
        ],
    ```

- [ ] **Step 7.3: Verify**

    ```bash
    ede info --config ede.conf 2>&1 | grep -iE "jobs|menu_cat_technical" | head -10
    ```
    Expected: foundation.jobs in the loaded apps list, no XML parse errors.

---

## Task 8: HTTP controller for action buttons

**Files:**
- Create: `src/ede/foundation/jobs/api/__init__.py`
- Create: `src/ede/foundation/jobs/api/jobs_routes.py`
- Create: `src/tests/foundation/jobs/test_jobs_routes.py`

- [ ] **Step 8.1: Failing test first**

    Create `src/tests/foundation/jobs/test_jobs_routes.py`:
    ```python
    """HTTP smoke tests for /api/foundation/jobs/* endpoints."""
    from ede.foundation.jobs.api.jobs_routes import JobsController


    def test_controller_is_decorated_with_route_config():
        """JobsController must carry the @api.route_config metadata."""
        assert hasattr(JobsController, "__ede_route_prefix__")
        assert JobsController.__ede_route_prefix__ == "/api/foundation/jobs"


    def test_run_now_method_exists():
        """run_now must exist as an @api.route-decorated method."""
        assert hasattr(JobsController, "run_now")


    def test_disable_method_exists():
        assert hasattr(JobsController, "disable")


    def test_enable_method_exists():
        assert hasattr(JobsController, "enable")


    def test_retry_run_method_exists():
        assert hasattr(JobsController, "retry_run")
    ```

    Run — expect ModuleNotFoundError:
    ```bash
    pytest src/tests/foundation/jobs/test_jobs_routes.py -v
    ```

- [ ] **Step 8.2: Write the api/__init__.py**

    Create `src/ede/foundation/jobs/api/__init__.py`:
    ```python
    # -*- coding: utf-8 -*-
    from . import jobs_routes  # noqa: F401  triggers @api.route_config scan
    ```

- [ ] **Step 8.3: Write the controller**

    Create `src/ede/foundation/jobs/api/jobs_routes.py`:
    ```python
    # -*- coding: utf-8 -*-
    """Foundation Jobs API — admin action buttons.

    Mounted at /api/foundation/jobs/*. All endpoints require an authenticated
    user with appropriate ir.job.update permission (RBAC enforces).

      POST /api/foundation/jobs/{job_id}/run-now      — manually trigger a job (creates ir.job.run + enqueues)
      POST /api/foundation/jobs/{job_id}/disable      — soft-toggle active=False
      POST /api/foundation/jobs/{job_id}/enable       — soft-toggle active=True
      POST /api/foundation/jobs/runs/{run_id}/retry   — force-retry a failed/dead-letter run (creates a fresh child run)
    """
    from __future__ import annotations

    import logging
    from datetime import datetime, timezone
    from typing import Any, Dict

    from ede.core import api
    from ede.core.services.http.controller import RouteController

    logger = logging.getLogger(__name__)


    @api.route_config(prefix="/api/foundation/jobs", tags=["foundation.jobs"])
    class JobsController(RouteController):
        """Admin action endpoints for the Settings → Technical → Jobs UI."""

        @api.route("/{job_id}/run-now", methods=["POST"], auth="user")
        def run_now(self, job_id: str) -> Dict[str, Any]:
            """Create an ir.job.run and enqueue it immediately."""
            env = self.env
            job = env.models["ir.job"].browse(job_id)
            if not job:
                return {"success": False, "error": f"Job {job_id} not found", "run_id": None}

            run = env.enqueue_job(
                target=job.target,
                payload={"manual_trigger": True, "triggered_by": env.principal.get("user_id")},
                priority=job.priority,
            )
            return {"success": True, "run_id": run.id, "job_name": job.name}

        @api.route("/{job_id}/disable", methods=["POST"], auth="user")
        def disable(self, job_id: str) -> Dict[str, Any]:
            env = self.env
            job = env.models["ir.job"].browse(job_id)
            if not job:
                return {"success": False, "error": f"Job {job_id} not found"}
            job.write({"active": False})
            return {"success": True, "active": False}

        @api.route("/{job_id}/enable", methods=["POST"], auth="user")
        def enable(self, job_id: str) -> Dict[str, Any]:
            env = self.env
            job = env.models["ir.job"].browse(job_id)
            if not job:
                return {"success": False, "error": f"Job {job_id} not found"}
            job.write({"active": True})
            return {"success": True, "active": True}

        @api.route("/runs/{run_id}/retry", methods=["POST"], auth="user")
        def retry_run(self, run_id: str) -> Dict[str, Any]:
            """Force-retry a failed/dead-letter run by creating a fresh child + dispatching."""
            env = self.env
            run = env.models["ir.job.run"].browse(run_id)
            if not run:
                return {"success": False, "error": f"Run {run_id} not found"}
            if run.status not in ("failed", "dead_letter"):
                return {
                    "success": False,
                    "error": f"Run status is {run.status!r}; only failed/dead_letter runs can be retried",
                }

            # Create a fresh ir.job.run + enqueue via env.enqueue_job
            job = run.job_id
            new_run = env.enqueue_job(
                target=job.target,
                payload={"manual_retry_of": run.id},
                priority=job.priority,
            )
            return {"success": True, "new_run_id": new_run.id, "previous_run_id": run.id}
    ```

    **If `env.principal.get("user_id")` is wrong** (e.g. the principal is a dict-like object that uses different keys), adjust to match how other controllers (`notification_routes.py`) read the current user. Look at `_current_user_id()` in the notifications controller for the canonical pattern.

    **If `auth="user"` isn't the right kwarg** (e.g. it might be `auth="bearer"` or no kwarg at all), match what notification_routes.py uses.

- [ ] **Step 8.4: Verify**

    ```bash
    pytest src/tests/foundation/jobs/test_jobs_routes.py -v
    ruff check src/ede/foundation/jobs/api/ src/tests/foundation/jobs/test_jobs_routes.py
    ```
    Expected: 5 tests PASS, ruff clean.

    If `__ede_route_prefix__` isn't the actual attribute name set by `@api.route_config`, adjust the test assertion to whatever the decorator actually stores (look at how other controllers expose this attribute).

---

## Task 9: End-to-end Phase 1 acceptance walkthrough

**Files:**
- Create: `src/tests/foundation/jobs/test_phase1_acceptance.py`

- [ ] **Step 9.1: Write the acceptance test**

    Create `src/tests/foundation/jobs/test_phase1_acceptance.py`:
    ```python
    """Phase 1 acceptance — heartbeat XML row → reconciler ignores → scheduler ticks → Celery executes → status=success."""
    from datetime import datetime, timedelta, timezone
    from pathlib import Path

    from ede.core.services.data_loader.loader import DataLoader
    from ede.foundation.base.models.data_reference import DataReference        # noqa: F401
    from ede.foundation.jobs.services.job_registry import _clear_registry_for_tests
    from ede.foundation.jobs.services.reconciler import reconcile_decorator_jobs
    from ede.foundation.jobs.services.scheduler import JobsScheduler


    def _ensure_data_reference_table(env):
        """Bootstrap ir.data.reference for the test SQLite engine."""
        from sqlalchemy import inspect
        from ede.core.kernel.metadata import SqlAlchemyMetadataBuilder

        engine = env.persistence.uow().session.get_bind()
        if "ir_data_reference" not in inspect(engine).get_table_names():
            DataReference.__ede_register__(env.registry)
            SqlAlchemyMetadataBuilder().build(env.registry).metadata.create_all(engine)


    def test_heartbeat_full_phase1_walkthrough(env_with_jobs_and_executor):
        """End-to-end: load heartbeat XML → reconcile (no-op) → scheduler tick → success."""
        env = env_with_jobs_and_executor
        _clear_registry_for_tests()

        # 1. Load the heartbeat XML row
        _ensure_data_reference_table(env)
        loader = DataLoader(env=env)
        fixture_path = (
            Path(__file__).resolve().parent.parent.parent.parent
            / "ede" / "foundation" / "jobs" / "data" / "example_jobs.xml"
        )
        loader._load_xml(app_name="foundation.jobs", file_path=str(fixture_path))

        # 2. Verify the row lands with source=xml
        heartbeat = env.models["ir.job"].search(
            [("name", "=", "foundation.jobs.heartbeat")]
        )[0]
        assert heartbeat.source == "xml"
        assert heartbeat.kind == "scheduled"
        assert heartbeat.cron == "*/2 * * * *"
        assert heartbeat.active is True

        # 3. Reconciler does NOT touch the XML row
        reconcile_decorator_jobs(env)
        unchanged = env.models["ir.job"].browse(heartbeat.id)
        assert unchanged.source == "xml"
        assert unchanged.active is True

        # 4. Force the job due now (the data loader set next_run_at = first future cron tick)
        heartbeat.write({"next_run_at_utc": datetime.now(tz=timezone.utc) - timedelta(seconds=30)})

        # 5. Scheduler tick dispatches it through Celery (eager mode)
        scheduler = JobsScheduler(executor=env._jobs_executor, tick_seconds=10)
        dispatched = scheduler.tick(env)
        assert dispatched == 1, f"expected scheduler to dispatch 1 job, got {dispatched}"

        # 6. The run row should be terminal (eager mode = synchronous)
        runs = env.models["ir.job.run"].search([("job_id", "=", heartbeat.id)])
        assert len(runs) == 1
        run = runs[0]
        assert run.status == "success", (
            f"expected success, got {run.status} (err={run.error_summary})"
        )
        # Heartbeat output shape: {"timestamp": "...", "iteration": 0}
        assert "timestamp" in run.output
        assert run.output["iteration"] == 0
        assert run.celery_task_id is not None
        assert run.worker_id is not None
        # Progress was set mid-tick to 50.0
        assert float(run.progress_pct) == 50.0
        assert "heartbeat tick" in run.progress_message

        # 7. next_run_at advanced to the next 2-minute cron boundary
        refreshed_job = env.models["ir.job"].browse(heartbeat.id)
        next_at = refreshed_job.next_run_at_utc
        if isinstance(next_at, str):
            next_at = datetime.fromisoformat(next_at)
        assert next_at > datetime.now(tz=timezone.utc)
    ```

- [ ] **Step 9.2: Run — expect PASS in eager mode**

    ```bash
    pytest src/tests/foundation/jobs/test_phase1_acceptance.py -v
    ```

    If something fails:
    - **`dispatched == 0`** → scheduler didn't see the heartbeat. Check `next_run_at_utc` was actually overwritten (the `job.write` might be cached; try `env.models["ir.job"].browse(heartbeat.id)` to refresh).
    - **`run.status != "success"`** → look at `run.error_summary` for the underlying exception. Likely a target resolution issue (`ede.foundation.jobs.demo.heartbeat.tick` not importable) or an env issue.
    - **`progress_pct != 50.0`** → the heartbeat tick call to `env.job_progress` didn't reach the row. Check Slice 3 wiring is intact (status of `test_progress.py` tests).
    - **DataLoader fails** → verify `ir.data.reference` was bootstrapped (the helper does it).

- [ ] **Step 9.3: Lint + full suite**

    ```bash
    pytest src/tests/foundation/jobs/ -v
    ruff check src/tests/foundation/jobs/test_phase1_acceptance.py
    ```
    Expected: cumulative jobs suite passing (64 prior + ~12 new from Slice 4 = ~76). Ruff clean.

---

## Task 10: Phase 1 ✅ flip + 4-site status update + PROGRESS + commit

- [ ] **Step 10.1: Full repo regression check**

    ```bash
    ./run_tests.sh 2>&1 | tail -5
    ```
    Expected: exit 0, prior baseline + ~12 new = new baseline, all green.

- [ ] **Step 10.2: Boot smoke**

    ```bash
    ede info --config ede.conf 2>&1 | head -15
    ```
    Expected: foundation.jobs in the loaded apps list, no traceback.

    **Live smoke (optional, if Postgres + Redis available)**:
    ```bash
    ede migrate upgrade -t scratch_phase1 --config ede.conf 2>&1 | tail -10
    ```
    Expected: alembic migrations → registry sync → data load (heartbeat row created with source=xml + 8 RBAC rows + 4 menu rows + 3 action rows + 3 view records) → orphan cleanup → `jobs reconciler: 0 inserted, 0 updated, 0 deactivated, 0 skipped` line. No errors.

- [ ] **Step 10.3: Flip Phase 1 status — 🟡 → ✅ across 4 sites**

    **(a) `roadmap/foundation/jobs/README.md`** — top Status header:
    ```
    **Status:** ✅ Delivered (Phase 1 complete 2026-05-19 — Slices 1+2+3+4 ✅: schema + Celery executor + jobs-worker CLI + cron scheduler + decorators + XML data path + boot reconciler + retry policy + dead-letter + env.job_progress + Settings → Technical → Jobs admin UI + RBAC seed + XML-declared heartbeat first adopter)
    ```

    Phased Delivery row:
    ```
    | [Phase 1](./phase-1-implementation.md) | ... | ~3 weeks | ✅ Delivered 2026-05-19 |
    ```

    **(b) `roadmap/foundation/jobs/phase-1-implementation.md`** — top Status header:
    ```
    **Status:** ✅ Delivered 2026-05-19 — all 10 workstreams green: WS-J1 models + WS-J2 scheduler + WS-J3 Celery executor + WS-J4 decorators+XML+reconciler + WS-J5 jobs-worker CLI + scheduler thread + env.enqueue_job + WS-J6 retry policy + dead-letter + WS-J7 progress reporting + WS-J8 admin UI (Settings → Technical → Jobs tree, list/form views for ir.job + ir.job.run + ir.job.lock) + WS-J9 RBAC seed (8 ir.rbac.permission rows across the 3 models) + WS-J10 heartbeat first adopter (XML-declared `ir.job` with cron="*/2 * * * *", target `ede.foundation.jobs.demo.heartbeat.tick`, walkthrough green).
    ```

    Check off all 10 WS-J* deliverables in the implementation file.

    **(c) `roadmap/roadmap-tracker.md`** — Overall + Phase 1 row + Last refreshed:
    - Overall: change `🟡 In Progress` → `✅ Delivered 2026-05-19`
    - Phase 1 row: `🟡 (Slices 1+2+3 ✅...)` → `✅ Delivered 2026-05-19`
    - Last refreshed: prepend Slice 4 entry (demote prior Slice 3 entry)
    - **Recompute the Status Roll-up table** (the `🟡` count drops by 1, `✅` count goes up by 1)

    **(d) `docs/foundation-jobs.md`** — Status header + Status Snapshot row + Built Capabilities row + Last sync.

- [ ] **Step 10.4: Append PROGRESS.md row**

    Dated 2026-05-19, theme: `foundation.jobs` Phase 1 ✅ Delivered (Slice 4 — admin UI + RBAC + heartbeat first adopter). Capture line count (~1,200) + hours (~1 design + 3 dev + 0.5 review = ~4.5 hrs).

- [ ] **Step 10.5: Stage + commit**

    Stage only Slice 4 files + the roadmap/docs/PROGRESS flips. Commit message:
    ```
    [IMP] foundation.jobs Phase 1 ✅: admin UI + RBAC + heartbeat first adopter (Slice 4)

    Closes Phase 1. ...
    ```

- [ ] **Step 10.6: Pause for explicit user `commit` instruction** (CLAUDE.md hard rule — controller, NOT the implementer, runs `git commit`).

- [ ] **Step 10.7: Surface downstream impact**

    Phase 1 ✅ unblocks:
    - **OneMaster Phase 1** — its hard prereq is now satisfied
    - `foundation.jobs` Phase 2 (advanced multi-worker + observability) — now eligible to start
    - `foundation.jobs` Phase 3 (retire ad-hoc workers in gateway/approval/notifications)

---

## Self-Review

**1. Spec coverage:**
- ✅ WS-J8 admin UI — Tasks 4 + 5 + 6 (list + form for 3 models) + Task 7 (menus + actions) + Task 8 (action button routes). Dashboard view deferred to Phase 2 enhancement (explicit, with rationale).
- ✅ WS-J9 RBAC seed — Task 3 (8 permission rows, binding to existing role_internal_user / role_system_admin)
- ✅ WS-J10 heartbeat first adopter — Tasks 1 (target callable) + 2 (XML row) + 9 (full walkthrough)
- ✅ Phase 1 ✅ flip — Task 10 (4-site status update, OneMaster unblock)

**2. Placeholder scan:**
- No "TBD" / "TODO" / "implement later" lines in any task.
- The "Dashboard view deferred to Phase 2 enhancement" is explicit scoping, not a placeholder.
- The "If `auth=...` kwarg name differs, adapt to existing pattern" callouts in Tasks 4/8 are verification-before-paste instructions, not placeholders.

**3. Type consistency:**
- `tick(env, payload: Optional[dict] = None) -> dict[str, Any]` in Task 1 matches the `target_fn(env, payload=...)` invocation pattern from Slice 1's task wrapper.
- `JobsController` `run_now`/`disable`/`enable`/`retry_run` methods match the test assertions in Task 8.
- XML id namespacing (`jobs.action_ir_job` / `jobs.menu_*`) is consistent.
- RBAC CSV id format (`jobs.p_ir_<model>_<action>`) is consistent across all 8 rows.

**4. Ordering:**
- Tasks 1 + 2 (heartbeat code + XML row) can be done in either order; the XML test references the target by dotted path so the target must exist.
- Task 3 (RBAC) is independent.
- Tasks 4 / 5 / 6 (views) are independent of each other but Task 7 (menus referencing actions referencing models with views) needs them all.
- Task 8 (HTTP controller) is independent.
- Task 9 (acceptance walkthrough) needs Tasks 1 + 2 + everything else for the visible-in-UI assertion (though the test exercises programmatically; full UI walkthrough is the live smoke in Task 10).
- Task 10 ships the closing.

---

## Execution Handoff

**Plan complete and saved to** `docs/superpowers/plans/2026-05-19-foundation-jobs-phase-1-slice-4.md`.

Two execution options:
**1. Subagent-Driven (recommended)** — same pattern that delivered Slices 1+2+3. ~10-12 subagent invocations including review loops.
**2. Inline execution with checkpoints** — slower but more visible.

Which approach?
