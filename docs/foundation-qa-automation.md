<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# QA Automation — Implementation Docs

**Module:** `foundation.qa-automation` (`src/ede/foundation/qa_automation/`)
**Roadmap:** [roadmap/foundation/qa-automation/](../roadmap/foundation/qa-automation/README.md)
**Status:** 🟡 In Progress (Phases 1, 2 & 3 ✅ Delivered 2026-05-13; Phases 4–8 🔴)
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A quality + use-case automation engine built on Playwright (open source, MIT). Provides four customer-visible capabilities: (1) browser-driven integration tests that exercise the live FastAPI backend through the real React webclient, runnable as a flag on `./run_tests.sh`; (2) a recorder — CLI (`ede e2e record`) and in-app ("Record Use Case" button) — that converts a live browsing session into a Playwright test and a structured use-case record; (3) a replay engine that re-runs the use-case library against any tenant, gates the ✅ Delivered flag on demo modules, and publishes polished demo videos to the docs; (4) a coverage reporter that maps every test to a BRS req ID and module use case, producing an HTML dashboard.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Today the EDE test surface stops at unit + frontend-component boundaries. A change that compiles, type-checks, and passes pytest can still ship a broken command contract between React and FastAPI — the two halves agree at the type level, disagree at the wire level. We also have no executable bridge between the prose in `docs/demo-usecase/modules/<m>/usecase.md` and the running tenant: the scenario exists as text and as XML demo data, but nothing automatically walks the demo through the UI to prove it works. `foundation.qa-automation` closes both gaps with a single open-source toolchain (Playwright driven from pytest) and layers in-app recording, automatic demo-video generation, and BRS req-coverage reporting on top.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **Developers** run `./run_tests.sh --e2e` for the full browser suite — collects from BOTH `src/tests/e2e/` (foundation infra tests) and `src/domains/<domain>/<module>/tests/e2e/` (module-specific use-case tests). Module tests co-locate with the module they exercise. While authoring, target a single file with `pytest src/domains/logistics/sales_crm/tests/e2e/test_create_lead.py -p ede.foundation.qa_automation.fixtures --headed`. Per-test artifacts (video + Playwright trace) land at `qa-report/artifacts/<sanitised-test-id>/{video.webm,trace.zip}`; JUnit at `qa-report/junit.xml`. Default `./run_tests.sh` (no `--e2e`) ignores the e2e trees so the regular suite is unaffected.
- **QA owners** click the "● Record" button in the React webclient (Phase 4, gated by the `qa.recorder` role). The system captures clicks / fills / navigations, generates a Playwright test, stubs the use-case doc, and tags the recording with a BRS req ID.
- **Customer support** triages bug reports by running `ede e2e import qa-bug-<id>` (Phase 8) — a customer-uploaded bundle becomes a failing regression test under `src/tests/e2e/regressions/`.
- **Sales / prospects** browse the live `demo.<host>` sandbox tenant (Phase 6); the recorded use-case library acts as a guided tour with video.
- **Roadmap delivery** uses `ede e2e replay --module=<module> --tenant=demo` (Phase 5) as the canonical pre-flip command before marking a feature ✅ Delivered — the replay must pass alongside the demo-data smoke load.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Author / QA / Customer]                  [foundation.qa-automation]                      [Output surfaces]
─────────────────────────                 ──────────────────────────                      ──────────────────
                                          ┌─────────────────────────────────┐
"Write a test"          ─►   src/tests/e2e/usecases/<module>/test_*.py      │
"Record in browser"     ─►   ede e2e record <name>      ─►  generates   ─►  │
                                          │                  test file +    │
                                          │                  usecase.md     │
"Record in app"         ─►   webclient ▶ "Record Use Case"                  │
                                          │                  → qa.recording │
                                          │                  → generates    │
                                          │                    test file    │
"Replay"                ─►   ede e2e replay --tenant=<x>                    │
"Customer bug"          ─►   ede e2e import <bug.zip>                       │
                                          └─────────────────────────────────┘
                                                       │
                          ┌────────────────────────────┼──────────────────────────────┐
                          ▼                            ▼                              ▼
                  pytest result                  HTML report                    Demo videos
                  (./run_tests.sh --e2e)         + trace viewer                 (auto-embedded
                  + JUnit XML for CI             + BRS coverage matrix          in usecase.md)
                                                 + module use-case grid
```

The core engine (`src/ede/core/engines/playwright_runner/`) is renderer-agnostic — same engine drives unit-mode (pytest), record-mode (CLI codegen), in-app mode (WebClient EventBus → step list), and replay-mode (DB-stored use case → Playwright actions). The foundation shell (`src/ede/foundation/qa_automation/`) layers the ORM models, admin UI, recorder API, and `ede e2e` CLI on top.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `qa.usecase` | Named browser-driven flow tied to BRS req IDs + module — name, description, status, BRS req IDs (JSON), test_file_path, test_class_name, owner, last_run_status, last_run_at_utc, active | [src/ede/foundation/qa_automation/models/usecase.py](../src/ede/foundation/qa_automation/models/usecase.py) (✅ Phase 2) |
| `qa.usecase.step` | Ordered action inside a use case — sequence, action enum, selector, payload (JSON), description | [src/ede/foundation/qa_automation/models/usecase.py](../src/ede/foundation/qa_automation/models/usecase.py) (✅ Phase 2 — schema; rows populated by Phase-4 recorder) |
| `qa.recording` | Raw capture session created by the in-app recorder, awaiting promotion to a `qa.usecase` | (Phase 4 — not yet authored) |
| `qa.run` | Replay execution — tenant, started_at, finished_at, status, video URL, trace URL, JUnit blob | (Phase 5 — not yet authored) |
| `qa.run.result` | Per-test results within a run — links to `qa.usecase`, captures duration / status / failure trace | (Phase 5 — not yet authored) |
| `qa.brs.requirement` | BRS req ID with description + module + owner, joined to `qa.usecase` for coverage | (Phase 7 — not yet authored) |
| `qa.bug.report` | Customer-uploaded bug repro bundle (HAR + console + trace) awaiting promotion | (Phase 8 — not yet authored) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `PlaywrightRunner` | Wraps `pytest-playwright`; spins up Chromium / Firefox / WebKit, attaches video + trace recorders | `src/ede/core/engines/playwright_runner/runner.py` (Phase 1) |
| `ede.foundation.qa_automation.fixtures` | Pytest-plugin module that ships the Phase 1 fixture stack (`frontend_build`, `live_server`, `seed_admin`, `with_demo_data`, `authenticated_page`). Loaded via `pytest_plugins` in every e2e conftest, plus `-p` flag from `./run_tests.sh --e2e`. | `src/ede/foundation/qa_automation/fixtures.py` (Phase 1) |
| `recorder.scaffold` (pure transformer) + `recorder.codegen_bridge` (subprocess shell) | `recorder.scaffold.build_test_file(...)` deterministically transforms raw `playwright codegen` output into an EDE-scaffolded test (class wrap, `pytestmark = pytest.mark.e2e`, `@pytest.mark.qa_module/.brs` decorators, `page = authenticated_page` body alias, initial-goto strip). `CodegenBridge.record(...)` invokes `playwright codegen --target=python-pytest`, captures the temp output, post-processes through the scaffold, writes the file to its conventional path, and appends a stub to `docs/demo-usecase/modules/<layer>-<short>/usecase.md`. | [src/ede/foundation/qa_automation/recorder/scaffold.py](../src/ede/foundation/qa_automation/recorder/scaffold.py) + [src/ede/foundation/qa_automation/recorder/codegen_bridge.py](../src/ede/foundation/qa_automation/recorder/codegen_bridge.py) (✅ Phase 3) |
| `InBrowserRecorder` | Listens on the WebClient EventBus, deduplicates synthetic React events, emits canonical step list | `src/ede/core/engines/playwright_runner/inbrowser_recorder.py` (Phase 4) |
| `ReplayEngine` | Loads a `qa.usecase` from DB, hydrates as Playwright actions, executes, persists `qa.run` | `src/ede/core/engines/playwright_runner/replay_engine.py` (Phase 5) |
| `EdeQaReporter` | Custom pytest reporter — emits the EDE HTML dashboard (module grid + BRS matrix) | `src/ede/core/engines/playwright_runner/reporter.py` (Phase 7) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `qa.usecase.create` | Recorder save / direct admin | Persists a `qa.usecase` + ordered `qa.usecase.step` rows |
| `qa.usecase.replay` | `ede e2e replay` CLI / admin button | Spawns replay engine against target tenant, persists `qa.run` |
| `qa.recording.promote` | In-app recorder "Save as Use Case" | Converts a `qa.recording` into a `qa.usecase` + writes test file to disk |
| `qa.bug.import` | `ede e2e import` CLI | Converts a bug bundle into a failing regression test |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `qa.run.completed` | Replay finishes (pass or fail) | Roadmap-row updater (✅-flip gate), notifications |
| `qa.usecase.recorded` | New usecase persisted from recorder | Doc-sync (appends to `usecase.md`) |
| `qa.bug.imported` | Customer bug bundle promoted to regression test | Notifications to the assigned dev |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/qa/recording/start` | Begin an in-app recording session | (Phase 4 — not yet authored) |
| `POST /api/qa/recording/append` | Append a captured step from the webclient | (Phase 4 — not yet authored) |
| `POST /api/qa/recording/stop` | Finalize a recording, return preview payload | (Phase 4 — not yet authored) |
| `POST /api/qa/usecase/{id}/replay` | Trigger a replay run | (Phase 5 — not yet authored) |
| `GET /api/qa/run/{id}/trace` | Stream Playwright trace zip | (Phase 5 — not yet authored) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.qa.usecase.delete` | Veto deletion if any `qa.run.result` references the use case (forces archive instead) |
| `post.qa.run.create` | Updates `qa.usecase.last_run_status` for fast dashboard rendering |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
**`qa.recording`**: `draft → ready → promoted` (terminal: promoted-to-usecase OR discarded)
**`qa.run`**: `queued → running → passed | failed | errored`
**`qa.bug.report`**: `received → triaged → imported → fixed | rejected`
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `qa_automation`
- `ACTIVE_DOMAINS` entry: _not applicable — foundation engine_
- Manifest `depends`: `["base", "auth", "presentation"]`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `QA_E2E_ENABLED` | `bool` | `False` | `QA_E2E_ENABLED` | Master switch — when false, `--e2e` flag is a no-op, the recorder button is hidden |
| `QA_E2E_BROWSERS` | `list[str]` | `["chromium"]` | `QA_E2E_BROWSERS` | Browsers to install + run against (`chromium`, `firefox`, `webkit`) |
| `QA_E2E_HEADLESS` | `bool` | `True` | `QA_E2E_HEADLESS` | Run browsers headless (CI) vs headed (demo-video generation) |
| `QA_E2E_VIDEO_DIR` | `str` | `qa-report/videos` | `QA_E2E_VIDEO_DIR` | Where recorded `.webm` videos land |
| `QA_E2E_TRACE_DIR` | `str` | `qa-report/traces` | `QA_E2E_TRACE_DIR` | Where Playwright trace zips land |
| `QA_RECORDER_ROLE` | `str` | `qa.recorder` | `QA_RECORDER_ROLE` | Role required to see the in-app "Record" button (Phase 4) |
| `QA_SANDBOX_RESET_CRON` | `str` | `0 3 * * *` | `QA_SANDBOX_RESET_CRON` | Cron for nightly sandbox reset (Phase 6 backend MVP — cron implementation deferred to E04 sandbox infra; setting reserved on `FoundationSettings`). |
| `QA_SANDBOX_TENANT_KEY` | `str` | `qa-sandbox` | `QA_SANDBOX_TENANT_KEY` | Reserved tenant key for the public demo sandbox (Phase 6 backend MVP — same E04-deferred caveat). |
| `QA_VIDEO_FFMPEG_PATH` | `str` | `ffmpeg` | `QA_VIDEO_FFMPEG_PATH` | ffmpeg binary path used by the Phase 6 polish pipeline. Override when ffmpeg is not on `$PATH`. |
| `QA_VIDEO_RESOLUTION` | `str` | `1280x720` | `QA_VIDEO_RESOLUTION` | Output resolution for the polish pipeline's title / closing card PNGs and the final concat (matches the standard QA_E2E_WINDOW for visual continuity). |
| `QA_E2E_WINDOW` | `str` | `1280x720` | `QA_E2E_WINDOW` | Standard window + viewport + video-recording size; runtime override per `./run_tests.sh --e2e --headed` invocation. Same value for headed and headless (ensures video.webm dimensions are byte-stable across CI ↔ developer machines and snapshot baselines). |
| `QA_TTS_ENGINE` | `str` | `kokoro` | `QA_TTS_ENGINE` | TTS engine for narrated demo videos: `kokoro` \| `sherpa` \| `none` (Enhancement 02 — anchored on Phase 6). Default is `kokoro-onnx` (Apache 2.0, ~82M params, CPU-only via ONNX Runtime). `sherpa` selects the `sherpa-onnx` adapter as the fallback if Kokoro stalls. `none` short-circuits and leaves the silent Phase 6 video untouched. |
| `QA_TTS_VOICE_DIR` | `str` | `~/.cache/ede/kokoro` | `QA_TTS_VOICE_DIR` | Cache directory for downloaded Kokoro ONNX model + voice-pack files (Enhancement 02). First-run synthesis downloads `kokoro-v1.0.onnx` + `voices-v1.0.bin` (~80 MB quantized total); subsequent runs reuse the cache. |
| `QA_TTS_DEFAULT_VOICE` | `str` | `af_sarah` | `QA_TTS_DEFAULT_VOICE` | Fallback Kokoro voice when `qa.usecase.voice` is blank and `narration_lang` doesn't resolve a locale match (Enhancement 02). Kokoro naming: `<lang>_<speaker>` — `af_` American Female, `am_` American Male, `bf_`/`bm_` British. |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `qa.coverage_required` | tenant | `bool` | `False` | When true, `ede e2e gate` exits non-zero on any missing/red qa.run.result for the module; false → warning only (Phase 5 ✅) |
| `qa.run.retention_days` | tenant | `int` | `90` | Auto-archive qa.run rows older than N days (Phase 5 — cleanup job deferred to E04) |
| `qa.bug_report.public_url` | tenant | `str` | _empty_ | Public URL where customers land to upload bug bundles (Phase 8) |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| QA — Recording | `src/ede/foundation/qa_automation/views/settings_qa.xml` (Phase 4) | recorder role, allowed origins, retention |
| QA — Replay | `src/ede/foundation/qa_automation/views/settings_qa_replay.xml` (Phase 5) | default tenant, video / trace retention, ✅-flip gate toggle |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `src/ede/foundation/qa_automation/data/qa_role.xml` (Phase 4) | `qa.recorder` role + default binding to admin users |
| `src/ede/foundation/qa_automation/demo/demo_qa_usecase.xml` (Phase 2) | A handful of reference use cases (sales-crm lead-to-quote, approval flow) that double as smoke tests |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Playwright stack + fixtures + reference test | ✅ Delivered 2026-05-13 | [phase-1-implementation.md](../roadmap/foundation/qa-automation/phase-1-implementation.md) |
| Phase 2 | Foundation primitive coverage + use-case test library | ✅ Delivered 2026-05-13 | [phase-2-implementation.md](../roadmap/foundation/qa-automation/phase-2-implementation.md) |
| Phase 3 | Recorder CLI (`ede e2e record`) | ✅ Delivered 2026-05-13 | [phase-3-implementation.md](../roadmap/foundation/qa-automation/phase-3-implementation.md) |
| Phase 4 | In-app recorder + recording library UI | ✅ Delivered 2026-05-13 | [phase-4-implementation.md](../roadmap/foundation/qa-automation/phase-4-implementation.md) |
| Phase 5 | Replay engine + ✅-flip gate | ✅ Delivered 2026-05-13 (backend MVP — frontend list view deferred to E04) | [phase-5-implementation.md](../roadmap/foundation/qa-automation/phase-5-implementation.md) |
| Phase 6 | Live automated demo (video + sandbox tenant) | ✅ Delivered 2026-05-13 (backend MVP — sandbox tenant infra deferred to E04) | [phase-6-implementation.md](../roadmap/foundation/qa-automation/phase-6-implementation.md) |
| Phase 7 | BRS coverage reporter + roadmap cross-link | 🔴 Not Started | [phase-7-implementation.md](../roadmap/foundation/qa-automation/phase-7-implementation.md) |
| Phase 8 | Customer bug reproducer | 🔴 Not Started | [phase-8-implementation.md](../roadmap/foundation/qa-automation/phase-8-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Playwright + pytest e2e harness (Phase 1) | _none — pure infrastructure_ | [src/ede/foundation/qa_automation/fixtures.py](../src/ede/foundation/qa_automation/fixtures.py) (pytest plugin: `frontend_build`, `live_server`, `seed_admin`, `with_demo_data` stub, `authenticated_page` w/ per-test video+trace, `unauthenticated_page`, `page_as_user` factory) · [src/tests/e2e/conftest.py](../src/tests/e2e/conftest.py) · [src/domains/logistics/sales_crm/tests/e2e/conftest.py](../src/domains/logistics/sales_crm/tests/e2e/conftest.py) (both declare `pytest_plugins = ("ede.foundation.qa_automation.fixtures",)`) · [src/domains/logistics/sales_crm/tests/e2e/test_create_lead.py](../src/domains/logistics/sales_crm/tests/e2e/test_create_lead.py) (reference test, `1 passed in 48.6s`) · [run_tests.sh](../run_tests.sh) (`--e2e` branch + default `--ignore-glob` for e2e trees) · [pyproject.toml](../pyproject.toml) (`e2e` extras, `e2e`/`brs`/`qa_module` pytest markers) · [.github/workflows/qa-e2e.yml](../.github/workflows/qa-e2e.yml) (PR job) · 5 `QA_E2E_*` settings on [FoundationSettings](../src/ede/foundation/settings.py) | [phase-1-implementation.md](../roadmap/foundation/qa-automation/phase-1-implementation.md) |
| Foundation primitive coverage + use-case test library (Phase 2) | `qa.usecase`, `qa.usecase.step` | [src/ede/foundation/qa_automation/models/usecase.py](../src/ede/foundation/qa_automation/models/usecase.py) (2 ORM models) · [src/ede/foundation/qa_automation/migrations/versions/f3a8e91b7c24_qa_automation_initial.py](../src/ede/foundation/qa_automation/migrations/versions/f3a8e91b7c24_qa_automation_initial.py) (Alembic) · [src/ede/foundation/qa_automation/demo/demo_qa_usecase.xml](../src/ede/foundation/qa_automation/demo/demo_qa_usecase.xml) (9 qa.usecase rows) · [src/tests/e2e/foundation/auth/test_login.py](../src/tests/e2e/foundation/auth/test_login.py) · [src/tests/e2e/foundation/base/test_members_directory_crud.py](../src/tests/e2e/foundation/base/test_members_directory_crud.py) · [src/tests/e2e/foundation/base/test_archived_toggle.py](../src/tests/e2e/foundation/base/test_archived_toggle.py) · [src/tests/e2e/foundation/base/test_rbac_denied.py](../src/tests/e2e/foundation/base/test_rbac_denied.py) · [src/tests/e2e/foundation/workflow/test_transition_via_statusbar.py](../src/tests/e2e/foundation/workflow/test_transition_via_statusbar.py) · [src/tests/e2e/foundation/presentation/test_kanban_drag.py](../src/tests/e2e/foundation/presentation/test_kanban_drag.py) · [src/tests/e2e/foundation/approval/test_approve_case.py](../src/tests/e2e/foundation/approval/test_approve_case.py) · [src/tests/e2e/foundation/communication/test_chatter_post.py](../src/tests/e2e/foundation/communication/test_chatter_post.py) · [src/tests/e2e/foundation/notifications/test_inbox_bell.py](../src/tests/e2e/foundation/notifications/test_inbox_bell.py). Run: **11 passed in 71.6s**. | [phase-2-implementation.md](../roadmap/foundation/qa-automation/phase-2-implementation.md) |
| Recorder CLI (`ede e2e record`) (Phase 3) | _none — pure tooling_ | [src/ede/cli/commands/e2e.py](../src/ede/cli/commands/e2e.py) (Click group `ede e2e` w/ `record` + `import` subcommands, registered in [src/ede/cli/main.py](../src/ede/cli/main.py)) · [src/ede/foundation/qa_automation/recorder/scaffold.py](../src/ede/foundation/qa_automation/recorder/scaffold.py) (pure codegen→EDE-scaffold transformer) · [src/ede/foundation/qa_automation/recorder/codegen_bridge.py](../src/ede/foundation/qa_automation/recorder/codegen_bridge.py) (subprocess shell around `playwright codegen --target=python-pytest`, writes test files + appends usecase.md stub) · [src/tests/foundation/qa_automation/test_recorder_scaffold.py](../src/tests/foundation/qa_automation/test_recorder_scaffold.py) (23 unit-test cases). Pre-auth seeding + trace-zip ingestion + URL-based module auto-detection intentionally deferred. | [phase-3-implementation.md](../roadmap/foundation/qa-automation/phase-3-implementation.md) |
| Deterministic-ID Seeder Primitive (Enhancement 01) | _none — test-only fixture_ | [src/ede/foundation/qa_automation/fixtures.py](../src/ede/foundation/qa_automation/fixtures.py) (`seed_deterministic` function-scoped fixture that bypasses `generic_repo.create()`'s auto-UUID via direct `session.execute(insert)`) · [src/tests/e2e/foundation/base/test_members_list_visual.py](../src/tests/e2e/foundation/base/test_members_list_visual.py) (first user — UUID round-trip + baseline screenshot capture). | [enhancements/01-deterministic-id-seeder-and-visual-regression.md](../roadmap/foundation/qa-automation/enhancements/01-deterministic-id-seeder-and-visual-regression.md) |
| Pixel-Diff Matcher (Enhancement 03) | _none — test-only helper_ | [src/ede/foundation/qa_automation/fixtures.py](../src/ede/foundation/qa_automation/fixtures.py) (`assert_visual_match(page_or_locator, baseline_path, *, max_diff_ratio=0.01, tolerance_per_channel=5, full_page=True)` hand-rolled PIL comparator — first run captures, subsequent runs do per-channel pixel diff; failures emit a side-by-side baseline-current-diff triptych under `qa-report/visual-diffs/`) · [src/tests/e2e/foundation/base/test_members_list_visual.py](../src/tests/e2e/foundation/base/test_members_list_visual.py) (re-pointed from bare `page.screenshot()` to `assert_visual_match(page, baseline_path)`) · `.gitignore` carves out `qa-report/snapshots/` from the parent `qa-report/*` ignore so baselines are committed while artifacts/junit/visual-diffs stay ignored. **Closes Phase 2 acceptance criterion 7 ("Visual snapshots committed") — flipped ☐ → ✅.** | [enhancements/03-pixel-diff-matcher.md](../roadmap/foundation/qa-automation/enhancements/03-pixel-diff-matcher.md) |
| Replay engine + ✅-flip gate (Phase 5, backend MVP) | `qa.run`, `qa.run.result` | [src/ede/foundation/qa_automation/models/run.py](../src/ede/foundation/qa_automation/models/run.py) (`QaRun` w/ `module`/`tenant`/`commit_sha`/`status`/`summary`/`junit_xml_path` + `QaRunResult` w/ `usecase_id`/`test_id`/`status`/`duration_ms`/`failure_message`/`failure_trace`) · [src/ede/foundation/qa_automation/migrations/versions/c58efb0a46d8_qa_automation_phase_5_qa_run_and_qa_run_.py](../src/ede/foundation/qa_automation/migrations/versions/c58efb0a46d8_qa_automation_phase_5_qa_run_and_qa_run_.py) · [src/ede/foundation/qa_automation/replay/](../src/ede/foundation/qa_automation/replay/) (`engine.py` ReplayEngine — subprocess pytest + JUnit parse + qa.run persistence + `qa.run.completed` event emission · `junit_parser.py` minimal pytest-shaped XML reader · `gate.py` `check_module_gate(env, module, freshness_hours=24)` → `GateOutcome`) · [src/ede/cli/commands/e2e.py](../src/ede/cli/commands/e2e.py) (`ede e2e replay --module --tenant --usecase --against-commit` + `ede e2e gate --module --tenant --freshness-hours`). Smoke: `ede e2e replay --module=foundation.auth --tenant=devqa` → 2 passed, qa.run row created, `ede e2e gate` flips ⚠️ → ✅. **Frontend `/wc/qa/runs` list view + downstream event subscribers (notifications, roadmap-row updater) carved as Enhancement 04 candidate.** | [phase-5-implementation.md](../roadmap/foundation/qa-automation/phase-5-implementation.md) |
| Live automated demo — video polish pipeline (Phase 6, backend MVP) | _none — pure tooling_ | [src/ede/foundation/qa_automation/polish/](../src/ede/foundation/qa_automation/polish/) — three modules: `cards.py` renders 1280×720 title + closing PNGs via PIL with `DejaVuSans` font (use-case name + module + 12-char commit SHA on title; "Powered by EDE" + "Enterprise Digital Engine" tagline on closing; indigo accent stripe matching the EDE webclient theme tokens — slate-900 background, slate-400 muted text); `pipeline.py` ships `polish_video_for_usecase(raw_video_path, usecase_name, module, commit_sha, ...) → PolishResult` (single use case) + `polish_run_videos(env, run_uuid) → PolishRunSummary` (walks every qa.run.result, finds video.webm under qa-report/artifacts/<sanitised-test-id>/, polishes, attaches to per-module usecase.md). ffmpeg concat: `[title:3s][raw video][closing:3s] → h264/yuv420p mp4 @ CRF 23, preset fast, +faststart`. Per-module videos land at `docs/demo-usecase/modules/<m>/videos/<slug>.mp4`; per-module `usecase.md` gains a `<!-- SYNC-BLOCK: videos -->` section listing every polished video as `<video controls>` markdown. Two new CLI surfaces in [src/ede/cli/commands/e2e.py](../src/ede/cli/commands/e2e.py): `ede e2e replay --polish` flag (auto-polishes after a replay completes) + standalone `ede e2e polish --run=<uuid>` (re-polish without re-running pytest). Four new foundation settings (`QA_VIDEO_FFMPEG_PATH` / `QA_VIDEO_RESOLUTION` / `QA_SANDBOX_TENANT_KEY` / `QA_SANDBOX_RESET_CRON`). **Smoke verified**: ran `polish_video_for_usecase` against a captured `auth/test_login` video.webm → produced an 11.4 s h264 mp4 (≈3 s title + ~5 s replay + 3 s closing) with both cards visually correct. **Sandbox tenant infrastructure deferred to E04 candidate** — Traefik routing + nightly cron + Docker reset + public URL + `/wc/demo` Tour mode is a multi-day infra workstream that ships separately. Per-step caption overlays (mentioned in the original Phase 6 plan) also deferred to E04 — they need per-step timing data from trace.zip that Phase 5 doesn't currently surface. | [phase-6-implementation.md](../roadmap/foundation/qa-automation/phase-6-implementation.md) |
| In-app recorder + recording library UI (Phase 4) | `qa.recording`, `qa.recording.step` | [src/ede/foundation/qa_automation/models/recording.py](../src/ede/foundation/qa_automation/models/recording.py) (`QaRecording` w/ `state` lifecycle `recording → saved → promoted | discarded` + `recorded_by_uid` / `module` / `base_url` / `initial_path` / `viewport_width` / `viewport_height` / `brs_requirement_ids` / `promoted_usecase_id` back-link + `QaRecordingStep` w/ `action` enum / `selector` / `payload` JSON / `description` / `captured_at_utc`) · [src/ede/foundation/qa_automation/migrations/versions/ad0bd80e421a_qa_automation_phase_4_recording_models.py](../src/ede/foundation/qa_automation/migrations/versions/ad0bd80e421a_qa_automation_phase_4_recording_models.py) · [src/ede/foundation/qa_automation/controllers.py](../src/ede/foundation/qa_automation/controllers.py) (HTTP surface: `POST /api/qa/recordings` create · `POST /api/qa/recordings/<id>/step` append · `POST /api/qa/recordings/<id>/finalize` save · `POST /api/qa/recordings/<id>/discard` · `POST /api/qa/recordings/<id>/promote` · `GET /api/qa/recordings` list · `GET /api/qa/recordings/<id>` detail) · [src/ede/foundation/qa_automation/recorder/inbrowser.py](../src/ede/foundation/qa_automation/recorder/inbrowser.py) (`promote_recording` — walks step rows, renders to codegen-shaped body, feeds Phase 3's `recorder.scaffold.build_test_file` for the EDE pytest wrap, writes the test file, creates `qa.usecase` row, appends stub to per-module usecase.md) · [src/ede/foundation/qa_automation/data/qa_role.xml](../src/ede/foundation/qa_automation/data/qa_role.xml) (`qa.recorder` + `qa.viewer` roles, both inheriting `internal_user`; admin default-bound to `qa.recorder`) · [src/ede/foundation/qa_automation/data/qa_menus.xml](../src/ede/foundation/qa_automation/data/qa_menus.xml) (new top-level QA Automation app under system category, 3 leaves: Use Cases / Recordings / Runs) · React webclient: [src/frontend/src/workspace/qa-recorder/](../src/frontend/src/workspace/qa-recorder/) — `qaRecorderService.ts` fetch wrappers · `captureSelectors.ts` pure selector heuristics (role+name → text → label → placeholder → CSS path) · `QaRecorderContext.tsx` state machine + global capture-phase click/change/navigation listeners (filtered by `[data-qa-recorder-ui]`) · `QaRecorderUi.tsx` floating Record button (idle) + control bar (recording, w/ Stop & Save + Discard) + Promote dialog (saved, w/ slug + module + BRS inputs) + Promoted / Discarded / Error banners; wired into [`WorkspaceClient.tsx`](../src/frontend/src/workspace/components/WorkspaceClient.tsx) so the button + overlay are visible across all authenticated views. Smokes: `python -c "from ede.foundation.qa_automation import controllers"` clean · `ede migrate upgrade -t devqa` applies migration + loads 3 role rows + 7 menu rows · `bun run build` 2861 modules clean · `bun run test` 494 vitest passed. **Known limitations** (deferred to follow-up enhancements): Promote action on auto-rendered FormView for OLDER saved recordings (current overlay covers just-recorded → promote happy-path); replay-preview on detail page (needs Phase 5 frame-by-frame render); UI step editing; role-gated Record button visibility (backend RBAC is authoritative, frontend button shown to all authenticated users until `/api/auth/me` exposes roles). | [phase-4-implementation.md](../roadmap/foundation/qa-automation/phase-4-implementation.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| No browser-driven test coverage on EDE today | 🟠 High | [roadmap/foundation/qa-automation/README.md](../roadmap/foundation/qa-automation/README.md) |
| Recorded use cases / live demos not yet possible | 🟠 High | [roadmap/foundation/qa-automation/README.md](../roadmap/foundation/qa-automation/README.md) |
| BRS req coverage matrix not yet tracked | 🟢 Low backlog | [roadmap/foundation/qa-automation/README.md](../roadmap/foundation/qa-automation/README.md) |
| TTS Narration via Kokoro — path-1 per-test card-window MVP design pinned; pyproject `tts` extras + 4 nullable additive fields on `qa.usecase` + 3 `QA_TTS_*` settings + new `polish/narrator.py` + `ede e2e replay --with-audio` flag pending implementation | 🟡 In Progress 2026-05-13 | [enhancements/02-tts-narration-via-piper.md](../roadmap/foundation/qa-automation/enhancements/02-tts-narration-via-piper.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _to be populated as the module is built out_

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- **Phase 1 (✅ Delivered 2026-05-13):** added the `e2e` extras group to `pyproject.toml` (`playwright>=1.45,<2.0`, `pytest-playwright>=0.5`). One-time setup: `pip install -e ".[dev,e2e]" && python -m playwright install chromium`. `./run_tests.sh --e2e` runs the suite against a freshly-migrated **PostgreSQL** tenant DB (unique `e2e_qa_<hex>` per session, dropped on teardown). SQLite is not supported because hand-written migrations in `logistics.masters` use ALTER TABLE ADD CONSTRAINT outside `batch_alter_table`. Per-test artifacts land at `qa-report/artifacts/<sanitised-test-id>/{video.webm,trace.zip}`; JUnit XML at `qa-report/junit.xml`. The fixture stack lives in `src/ede/foundation/qa_automation/fixtures.py` and is shared across `src/tests/e2e/` (foundation) and `src/domains/<domain>/<module>/tests/e2e/` (module) via `pytest_plugins`. Five new `QA_E2E_*` settings ship on `FoundationSettings`. **No ORM migrations** — Phase 1 is pure infrastructure (no records, demo-data gate documented as N/A in the phase Verification section). `qa-report/` is gitignored.
- **Phase 2 (✅ Delivered 2026-05-13):** introduces the first migration under `src/ede/foundation/qa_automation/migrations/` (`qa.usecase`, `qa.usecase.step`, Alembic `f3a8e91b7c24` anchored on foundation.base head `03658b1fa089`), registers pytest markers `brs` + `qa_module`, and ships 9 **foundation primitive** tests / 10 test methods across Tier 1 (auth login + bad creds, Members directory CRUD via res.partner, archived toggle revealing soft-archived rows, RBAC denied: backend `PermissionDeniedError` + UI app-filter) + Tier 2 (workflow drop-lead via FormView statusbar w/ "Continue" confirm, kanban drag → workflow transition on `crm.opportunity` via @dnd-kit pointer events, Approvals app inbox renders + `ir.approval.case` query, chatter post → `communication.message` DB row, notifications bell shows seeded `ir.notification` + "Mark all as read" sets `read_at_utc`). **11 passed in 71.6s**. **Scope pivot**: domain flow tests (sales-crm lead→quote, pricing rate query / margin override) deferred until those modules ship their own demo data — foundation primitives stress every consumer module implicitly. Visual regression snapshots deferred to Enhancement 01 — record_uuid drifts session-to-session under the e2e-tenant-per-run model, needs a deterministic-id seeder first. Demo data ships 9 `qa.usecase` rows in `demo/demo_qa_usecase.xml` so `--with-demo=foundation.qa_automation` produces a populated Use Case Library (smoke-tested: `demo_load: 9 created`).
- **Phase 3 (✅ Delivered 2026-05-13):** new `ede e2e` Click group with two subcommands. `ede e2e record <module>/<slug>` wraps `playwright codegen --target=python-pytest` against a running EDE server (operator runs `ede serve --with-worker` in another terminal first; the browser session is interactive, the operator records their flow + closes the window). `recorder.scaffold.build_test_file(...)` deterministically transforms the codegen output into an EDE-scaffolded pytest file (class wrap w/ `Test<PascalCase>:`, `pytestmark = pytest.mark.e2e`, `@pytest.mark.qa_module("<module-key>")` + optional `@pytest.mark.brs(...)`, `page = authenticated_page` body alias, drops the initial `page.goto(<base>/wc/)` since the fixture lands there). `recorder.codegen_bridge` is the thin subprocess shell that routes the test file to `src/tests/e2e/foundation/<short>/` or `src/domains/<dom>/<mod>/tests/e2e/`, auto-creates `__init__.py` markers, and appends a stub under `## Recorded e2e tests` in the per-module usecase.md (creating the file from a minimal scaffold if missing). Companion `ede e2e import recording.json` for external JSON bundles (trace-zip ingestion deferred to Phase 8 with a clear forward-pointer error). Pure scaffold transformer is unit-tested via 23 pytest cases in `src/tests/foundation/qa_automation/test_recorder_scaffold.py` — slug validation, class+function name generation, decorator emission, body indentation, initial-goto stripping, fixture-parameter rewrite, playwright-import removal, AST parse-clean check on the emitted file, and usecase.md stub formatting. The live recording leg is inherently human-driven (codegen requires a real browser session) — documented as manual smoke. **Pre-authenticated browser-context seeding deferred** (codegen records the login form, which is itself e2e-tested by `auth/test_login.py`); **module-from-URL auto-detection deferred** (v1 requires `--module` or the `<module>/<slug>` slug form).
- **Phase 5** adds the ✅-flip gate to the roadmap-delivery flow — opt-in via `qa.coverage_required` until the module stabilises.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `qa.recorder` | Visibility of the in-app "Record" button; CRUD on `qa.recording`; promote to `qa.usecase`; replay |
| `qa.viewer` | Read-only on `qa.usecase` / `qa.run` / `qa.run.result`; view coverage dashboards |
| (admin) | Full access; can change recorder role binding |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [foundation.auth](foundation-auth.md) — `ir.session` + JWT, used by the `authenticated_page` fixture
- [foundation.presentation](foundation-presentation.md) — the React webclient under test; recorder hooks the EventBus
- [docs/demo-usecase/README.md](demo-usecase/README.md) — unifying scenario every recorded use case must align with
- [.claude/skills/authoring-e2e-tests/SKILL.md](../.claude/skills/authoring-e2e-tests/SKILL.md) — the canonical authoring workflow
- [.claude/skills/preparing-demo-data/SKILL.md](../.claude/skills/preparing-demo-data/SKILL.md) — companion gate that Phase 5 extends
<!-- /SYNC-BLOCK -->

### Future Directions (Parked — good-to-have, not actively planned)
<!-- HAND-AUTHORED — preserved across syncs -->
Captured here so the option space stays visible. These are NOT on the active roadmap and have no committed phase number; they surface as concrete plans when a consumer commits to consuming them.

- **AI-Generated User Documentation & Feature Guides** — turn the existing replay artifacts (per-step screenshots, `trace.zip`, `qa.usecase.step.description`) into a multi-output documentation pipeline: static markdown user guides, video walkthroughs (already covered by Phase 6 + E02), interactive in-app tours, AI-curated troubleshooting / common-mistakes sections, and localized translations. The e2e infrastructure is the ground truth (real screenshots + replay gate confirms the flow actually works); an LLM provides the voice (titles, prose, narration script, translations). The same `qa.usecase.step.description` field becomes multi-duty across video captions, TTS narration script (Enhancement 02 — Kokoro), and user-doc prose. Likely shape: a new `foundation.docs-generator` module that consumes the qa-automation replay output, gated through `foundation.ai`. See [roadmap/foundation/qa-automation/README.md § Future Directions](../roadmap/foundation/qa-automation/README.md#future-directions-parked--good-to-have-not-actively-planned).
- **`foundation.ai`** — the broader future module this depends on. A general-purpose AI provider substrate (swappable LLM, embeddings, vision, TTS adapters) modelled like `foundation.persistence` (sqlalchemy / null providers today). Would serve docs-generation as its first consumer, with future consumers including in-app "ask the docs" search, automatic test-failure triage, code-explainer for the model registry, and BRS req-to-test auto-suggestion (closes a deferred Phase 7 gap). Designed engine-agnostic so an OSS local-LLM swap remains possible. **No active roadmap entry yet** — surfaces when a concrete consumer commits.

---

*Last sync: 2026-05-13 (Phases 1, 2, 3, 4, 5 & 6 ✅ Delivered + Enhancement 01 ✅ Delivered + Enhancement 02 🟡 In Progress (path-1 per-test card-window narration MVP design pinned) + Enhancement 03 ✅ Delivered + **Phase 6 ✅ Delivered (backend MVP) — Live Automated Demo (video polish)**: new `src/ede/foundation/qa_automation/polish/` package — `cards.py` renders 1280×720 title + closing PNGs via PIL (use-case name + module + commit SHA on title; "Powered by EDE" on closing; indigo accent stripe matching webclient theme); `pipeline.py` ships `polish_video_for_usecase` + `polish_run_videos`; ffmpeg concat `[title:3s][raw video][closing:3s] → h264/yuv420p mp4`. Polished videos land at `docs/demo-usecase/modules/<m>/videos/<slug>.mp4` and auto-attach to the per-module `usecase.md` via a new `<!-- SYNC-BLOCK: videos -->` section. Two new CLI surfaces: `ede e2e replay --polish` (auto-polish after replay) + standalone `ede e2e polish --run=<uuid>` (re-polish without re-running pytest). Four new foundation settings (`QA_VIDEO_FFMPEG_PATH` / `QA_VIDEO_RESOLUTION` / `QA_SANDBOX_TENANT_KEY` / `QA_SANDBOX_RESET_CRON`). Smoke verified: ran `polish_video_for_usecase` against a captured auth/test_login video.webm → produced an 11.4 s h264 mp4 with both cards visually correct. Sandbox tenant infrastructure (Traefik + cron + Docker + public URL + `/wc/demo` Tour mode) carved as E04 candidate; per-step caption overlays also deferred to E04 (need per-step timing from trace.zip Phase 5 doesn't currently surface). **Phase 4 ✅ Delivered — In-App Recorder**: `qa.recording` + `qa.recording.step` models + Alembic + new `controllers.py` HTTP surface (7 endpoints) + `recorder/inbrowser.py` `promote_recording` pipeline + 3 RBAC role rows (qa.recorder + qa.viewer + admin binding) + 7 menu rows (QA Automation top-level app + 3 leaves) + React `qa-recorder/` package (service + selector heuristics + state-machine context + floating UI) wired into `WorkspaceClient`. Smokes: backend imports clean · migration + seed load on devqa · `bun run build` 2861 modules clean · 494 vitest passed. Live human-driven recording smoke remains the operator's responsibility per the roadmap's "inherently human-driven" caveat. **Phase 5 ✅ Delivered (backend MVP) — Replay Engine + ✅-flip Gate**. `qa.run` + `qa.run.result` ORM models + Alembic migration `c58efb0a46d8` + `src/ede/foundation/qa_automation/replay/` engine package (subprocess pytest + JUnit XML parser + gate logic) + 2 new CLI subcommands under `ede e2e` group (`replay` for executing tagged use cases, `gate` for ✅-flip enforcement) + 2 new `ir.config` keys (`qa.coverage_required` soft/hard gate toggle, `qa.run.retention_days` cleanup horizon) + `qa.run.completed` event hook for downstream subscribers. Smoke: `ede e2e replay --module=foundation.auth --tenant=devqa` → 2 passed, qa.run row created, `ede e2e gate` clears. Frontend `/wc/qa/runs` list view + downstream event subscribers (notifications, roadmap-row updater) carved as E04 candidate. — Enhancement 03 ✅ Delivered — Pixel-Diff Matcher closes Phase 2's outstanding "Visual snapshots committed" acceptance checkbox by adding a hand-rolled PIL comparator helper `assert_visual_match(page_or_locator, baseline_path, *, max_diff_ratio, tolerance_per_channel)` in [src/ede/foundation/qa_automation/fixtures.py](../src/ede/foundation/qa_automation/fixtures.py). First-run captures the baseline; subsequent runs do per-channel pixel diff with configurable tolerance; failure writes a baseline-current-diff triptych PNG under `qa-report/visual-diffs/<test-id>.png`. No new ORM models, no new settings, no new foundation deps (PIL already transitive). `qa-report/snapshots/` carved out of `.gitignore` so baselines are committed. Phase 2 acceptance criterion 7 flips ✅ on landing. — Phase 1+2: 11 e2e passed → 12 with E01 + `qa.usecase` models + 9 demo rows; Phase 3: `ede e2e record` CLI shipped + 23-case unit suite on the pure scaffold transformer; Enhancement 01: deterministic-ID seeder primitive unblocks visual regression; Enhancement 02: TTS narration via **Kokoro** anchored on Phase 6 — **2026-05-13 pivoted to path-1 per-test card-window narration MVP**: title card narrates `qa.usecase.description`, each section card narrates the matching test method's first-line docstring (fallback humanised method name), closing card optionally narrates a fixed sign-off; raw method video stays silent. Trades raw-action narration coverage (deferred to a follow-up enhancement once trace.zip step-extraction lands) for zero new instrumentation + perfect a/v sync via paired `anullsrc` silent audio segments. Schema delta: 4 nullable additive columns on `qa.usecase` (`voice` / `audio_enabled` / `narration_lang` / `slow_on_overrun` — last column reserved for a future per-step enhancement). `qa.usecase.step.narration_override` dropped from MVP scope — that field belongs to per-step narration. Single additive Alembic, no FK touches, no demo-data delta. Three `QA_TTS_*` foundation settings already propagated into the foundation-settings table above. New optional `tts` extras group in `pyproject.toml` (`kokoro-onnx>=0.4` + `sherpa-onnx>=1.0`, both Apache 2.0 CPU-only); new `polish/narrator.py` module; `ede e2e replay --with-audio` flag wires the polish pipeline's ffmpeg invocation to mux audio when Kokoro is available; `usecase.md` `<!-- SYNC-BLOCK: videos -->` block prefers `demo.mp4` over the silent polished mp4 when both exist). To refresh, invoke the `syncing-roadmap-to-docs` skill.*
