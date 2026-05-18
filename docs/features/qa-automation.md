# QA Automation

Record a browser flow once, replay it on every commit. The recorder captures clicks, fills, and assertions against a running dev server; the replay engine runs the same sequence on Playwright in CI and gates the merge.

```bash
# Start a dev server with the worker
ede serve --with-worker &

# Record a flow into a use case
ede e2e record --usecase blog.publish-post

# Replay everything in CI
ede e2e replay --tenant qa-sandbox
```

A use case is a persistent record (`qa.usecase` + `qa.usecase.step`) — not a file on disk. The recorder writes steps as you click; the replay engine reads them back, runs them through Playwright, and stores results in `qa.run` / `qa.run.result`.

---

## What you get

-   **`ede e2e` CLI** — five subcommands: `record`, `import`, `replay`, `polish`, `gate`. All routed through the framework's Click group.
-   **`qa.recording` / `qa.recording.step`** — the in-browser recorder's live capture target. Rows finalise into use cases.
-   **`qa.usecase` / `qa.usecase.step`** — frozen, replayable scenarios. Promoted from a recording once you're happy with it.
-   **`qa.run` / `qa.run.result`** — one row per replay run plus one row per use-case result inside it. CI consumes these.
-   **HTTP API** — `POST /api/qa/recordings`, `POST /api/qa/recordings/{id}/step`, `POST /api/qa/recordings/{id}/finalize`, `POST /api/qa/recordings/{id}/promote`, plus reads. The in-browser recorder is a thin client over this surface.
-   **Replay gate** — `ede e2e gate --module <key>` reads the latest `qa.run` for a module and blocks merge if any use case failed within a freshness window.
-   **Polish pipeline** — `ede e2e polish` post-processes a run into a deliverable video (card titles, narration cards, ffmpeg-stitched MP4) for sharing.
-   **`seed_deterministic` pytest fixture** — byte-stable demo seeding so replay diffs come from your code change, not from `created_at` jitter.
-   **`activity.type` integration** — replay results are posted as chatter messages on the parent module's record so the audit lives next to the code under review.
-   **CI workflow** — `.github/workflows/qa-e2e.yml` runs the suite on every PR.

## How to use it

### Record a use case

```bash
ede serve --with-worker &
ede e2e record --usecase blog.publish-post
```

Chrome opens. Perform the flow you want to test. The recorder writes a `qa.recording` + ordered `qa.recording.step` rows to the dev DB as you click.

Close the recorder window when done. Finalize and promote the recording to a use case:

```bash
ede e2e record --finalize <recording_uuid>
ede e2e record --promote  <recording_uuid> --usecase blog.publish-post
```

Promoting freezes the steps into `qa.usecase` / `qa.usecase.step` — replays will run exactly these steps.

### Import an existing Playwright script

```bash
ede e2e import --file path/to/test_flow.py --usecase blog.search-filter
```

The Playwright codegen-format script is parsed into `qa.usecase.step` rows so it joins the same replay pipeline.

### Replay a use case

```bash
ede e2e replay --tenant qa-sandbox --usecase blog.publish-post
```

The replay engine spins up the demo tenant defined by `QA_SANDBOX_TENANT_KEY`, runs each step through Playwright, records timings + screenshots, and writes a `qa.run.result` per use case.

Omit `--usecase` to replay everything that targets the tenant.

### Gate a merge on replay freshness

```bash
ede e2e gate --module foundation.communication --freshness-hours 24
```

The gate exits non-zero if the latest `qa.run` for `foundation.communication` failed, or if no run inside the freshness window exists. Wire it into CI as the last step before merge.

### Polish a passing run into a sharable video

```bash
ede e2e polish --run <run_uuid>
```

Generates title and result cards, stitches them around the replay video with ffmpeg, and uploads the result for stakeholders.

### Use `seed_deterministic` in a pytest test

```python
def test_my_flow(seed_deterministic, authenticated_page, live_server):
    seed_deterministic("res.partner", {
        "name": "Alice",
        "email": "alice@example.com",
    })
    authenticated_page.goto(live_server.url + "/wc")
    # ... assertions
```

`seed_deterministic` writes rows with stable `record_uuid`s derived from a hash of the payload — re-running the test against a clean tenant produces the same UUIDs, so screenshot diffs and DB-state assertions are byte-stable.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `ENABLE_QA_AUTOMATION` | `False` | Master switch — register the module's models, routes, and CLI. |
| `QA_E2E_ENABLED` | `False` | Toggle replay execution. Off in production. |
| `QA_E2E_BROWSERS` | `["chromium"]` | Browsers to run replays against. Add `"firefox"` or `"webkit"` for cross-browser. |
| `QA_E2E_HEADLESS` | `True` | Run headless; set `False` to see the browser when developing locally. |
| `QA_E2E_VIDEO_DIR` | `qa-report/videos` | Where Playwright writes video recordings of each run. |
| `QA_E2E_TRACE_DIR` | `qa-report/traces` | Where Playwright writes trace files for failed replays. |
| `QA_VIDEO_FFMPEG_PATH` | `ffmpeg` | Binary used by the polish pipeline. |
| `QA_VIDEO_RESOLUTION` | `1280x720` | Video size for polished output. |
| `QA_VIDEO_SLOWMO_MS` | `700` | Per-action slowmo when generating sharable videos. |
| `QA_SANDBOX_TENANT_KEY` | `qa-sandbox` | Tenant key the replay engine targets. |

## How it composes with other features

-   [Commands & events](../tutorial/04-commands-and-events.md) — the in-browser recorder dispatches `qa.recording.step.create` through the same command bus as any other write.
-   [Communication (Chatter)](communication.md) — replay results post a chatter message on the related module record so reviewers see them in context.
-   [Permissions](security.md) — recordings and runs are RBAC-gated; only QA roles can promote a recording to a frozen use case.

## Reference

-   Models: `src/ede/foundation/qa_automation/models/{recording,usecase,run}.py`
-   HTTP controller: `src/ede/foundation/qa_automation/controllers.py` (prefix `/api/qa`)
-   In-browser recorder: `src/ede/foundation/qa_automation/recorder/{inbrowser,codegen_bridge,scaffold}.py`
-   Replay engine: `src/ede/foundation/qa_automation/replay/{engine,gate,junit_parser}.py`
-   Polish pipeline: `src/ede/foundation/qa_automation/polish/{cards,pipeline}.py`
-   Pytest fixtures (`seed_deterministic`): `src/ede/foundation/qa_automation/fixtures.py`
-   CLI: `src/ede/cli/commands/e2e.py`
-   CI workflow: `.github/workflows/qa-e2e.yml`
