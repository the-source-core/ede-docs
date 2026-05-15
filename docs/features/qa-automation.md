# QA Automation

Browser-driven end-to-end tests via Playwright, with use-case recording and deterministic replay against demo tenants. Record a flow once in your browser, replay it on every commit.

```bash
# Record a flow against a running dev server
ede e2e record --usecase blog.publish-post

# Replay it as part of the test suite
./run_tests.sh e2e
```

The recorder captures every click, fill, and assertion, normalizes timestamps and IDs, and emits a deterministic replay file that lives alongside the rest of your tests.

---

## What you get

-   **Playwright plumbing** — Chromium + Firefox + WebKit, headless or headed.
-   **`qa.usecase`** + **`qa.usecase.step`** models — recorded use cases are first-class records.
-   **`ede e2e record`** CLI — interactive recording into a 23-case scaffold transformer.
-   **`seed_deterministic`** primitive — byte-stable test fixtures so replays don't drift on rerun.
-   **9 foundation-primitive e2e tests** — login, navigation, list-view, form-view, kanban, search, create/read/update/delete, chatter.
-   **Visual-regression matcher** — pixel-diff against committed baselines under `qa-report/snapshots/`.
-   **CI integration** — the `qa-e2e.yml` workflow runs the suite on every PR.

## How to use it

### Record a use case

```bash
ede serve --with-worker &
ede e2e record --usecase blog.publish-post
```

A Chromium window opens. Perform the flow you want to test; the recorder writes a `qa.usecase` record + steps to your dev DB.

Stop the recorder when done — the use case is now persisted and ready to export as a replay script.

### Replay all use cases

```bash
./run_tests.sh e2e
```

Playwright spins up a fresh demo tenant per run (the `seed_deterministic` primitive guarantees stable IDs), executes each step, and compares the result against the recorded assertions.

### Replay a single use case

```bash
pytest src/tests/e2e/usecases/blog/test_publish_post.py
```

### Lock visual baselines

When a UI change is intentional and visual diffs flip:

```bash
./run_tests.sh e2e --update-snapshots
```

Commits the new baseline under `qa-report/snapshots/`.

### Write a use case by hand

```python
from ede.foundation.qa_automation.testing import e2e_test


@e2e_test("blog.search-filter")
def test_search_by_author(page, env, demo_tenant):
    page.goto(demo_tenant.url + "/wc")
    page.fill('input[name="search"]', "alice")
    page.click('button[data-action="apply-filter"]')
    page.wait_for_selector("text=Showing 3 posts")
```

## How it composes with other features

-   **[Demo Data Loader](../tutorial/01-your-first-module.md)** — `--with-demo=<app>` is how the e2e harness seeds tenants.
-   **[Tutorial — Your First Module](../tutorial/01-your-first-module.md)** — the example tests use the same module shape you build there.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `E2E_HEADED` | `false` | Run with visible browser. |
| `E2E_BROWSER` | `chromium` | One of `chromium`, `firefox`, `webkit`. |
| `E2E_BASE_URL` | `http://localhost:8000` | Where the replayed flows hit. |

## Reference

| Concept | Where it lives |
|---|---|
| `qa.usecase`, `qa.usecase.step` | `src/ede/foundation/qa_automation/models/` |
| Recorder CLI | `src/ede/cli/e2e.py` |
| Playwright runner | `src/ede/core/engines/playwright_runner/` |
| Tests | `src/tests/e2e/` |
| `seed_deterministic` | `src/ede/foundation/qa_automation/testing/seed.py` |
