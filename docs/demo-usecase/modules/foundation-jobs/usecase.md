# `foundation.jobs` — Demo Use-Case

**Module:** `ede.foundation.jobs`
**App key:** `foundation.jobs`
**Demo manifest entries** (target): _none initially_ (see below — depends on Phase 1 ship)
**Status:** 🔴 Not authored (module itself is 🔴 Not Started — see [roadmap/foundation/jobs/](../../../../roadmap/foundation/jobs/))

---

## Use-case

The Jobs engine itself ships **production** seeds for system-level scheduled jobs (registry sync reconciler, dead-letter reaper). Demo data here covers:

- One sample one-shot background job that has already completed (to exercise the "Run History" UI with realistic data).
- One sample scheduled job in `enabled=True` state owned by the demo admin so the dashboard isn't empty on first login.
- One sample dead-letter row (a failing fake job) so the recovery UI is exercisable.

This module's demo data lands AFTER Phase 1 ships — until then, this doc stays at 🔴 and the module's `__manifest__.py` does not declare a `demo: [...]` list.

## Records produced (planned, Phase 1)

### `demo/demo_job_run_history.xml`

| External ID | Model | State |
|---|---|---|
| `jobs.demo_run_sync_demo_data` | `ir.job.run` | status=`succeeded`, duration_ms=120, owner=`demo_user_admin` |
| `jobs.demo_run_failing_sample` | `ir.job.run` | status=`dead_letter`, error_excerpt="ConnectionError to demo-api.test:443" |

### `demo/demo_job_definitions.xml`

| External ID | Model | Notes |
|---|---|---|
| `jobs.demo_job_nightly_summary` | `ir.job` | cron=`0 2 * * *`, enabled=true, owner=`demo_user_admin` |

## Out of scope

- The platform's own scheduled jobs — production seeds.
- Multi-worker / distributed-lock contention demo — Phase 2.

## Dependencies

- `foundation.base` demo (owner users)
- Phase 1 of `foundation.jobs` itself shipping `ir.job` / `ir.job.run` / `ir.job.queue` models.

## Verification (once Phase 1 lands)

```
ede migrate upgrade -t demo --with-demo=foundation.jobs
```

Open Settings → Technical → Jobs → dashboard shows the demo nightly summary, run history shows one success + one dead-letter.

## Authoring order

1. Phase 1 ship of the module (gates this doc moving past 🔴).
2. `demo_job_definitions.xml` first (creates the job definition the run history then refers to).
3. `demo_job_run_history.xml`.

---

*Back to [demo-usecase index](../../README.md).*
