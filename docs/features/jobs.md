# Background Jobs

Run work outside the request cycle — on a cron schedule or enqueued on demand. Declare a job with a decorator and the engine handles scheduling, retries, timeouts, and an auditable run history.

```python
@api.scheduled_job(name="blog.purge_drafts", cron="0 2 * * *")
def purge_old_drafts(env, payload=None):
    stale = env.models["blog.post"].search([("state", "=", "draft")])
    stale.delete()
    return {"purged": len(stale)}
```

Every execution is recorded as an `ir.job.run` row with status, output, timing, and traceback — visible under Settings → Technical → Jobs.

---

## What you get

-   **`@api.scheduled_job`** — register a cron-driven job at import time.
-   **`@api.background_job`** — register a callable you enqueue on demand.
-   **`ir.job`** — the job definition (cron, retry policy, priority, timeout, active flag).
-   **`ir.job.run`** — append-only execution log (status, payload, output, error, progress, timing).
-   **`ir.job.lock`** — scheduler-side dedup so only one worker claims a due job.
-   **`env.enqueue_job(...)`** — enqueue a background job programmatically.
-   **`env.job_progress(...)`** — report progress from inside a running job.
-   **Retry policies** — `none` / `fixed` / `exponential` / `linear`, with a terminal dead-letter state.
-   **Workers** — `ede jobs-worker` (executor) and `ede worker` (in-process scheduler).
-   **Admin UI** — job definitions, run history, and locks under Settings → Technical → Jobs.
-   **Metrics** — a Prometheus endpoint exposing queue depth and run counts.

## How to use it

### Schedule a recurring job

The `cron` is a standard 5-field expression. The target receives `env` and an optional `payload`; its return value is stored as the run's output (must be JSON-serializable).

```python
@api.scheduled_job(
    name="blog.refresh_view_counts",
    cron="*/15 * * * *",
    retry_policy="exponential",
    retry_max_attempts=3,
    priority=5,
    timeout_seconds=300,
)
def refresh_view_counts(env, payload=None):
    ...
    return {"updated": n}
```

### Readable schedules

Raw 5-field cron strings are terse and easy to misread. `SchedulerCron` builds the same string from named parts and validates it at import time, so a bad schedule fails loudly at declaration instead of running on the wrong cadence:

```python
from ede.foundation.jobs import SchedulerCron

@api.scheduled_job(name="blog.reindex", cron=SchedulerCron.run_every(minutes=15))
def reindex(env, payload=None): ...

@api.scheduled_job(name="blog.purge_drafts", cron=SchedulerCron.daily(hour=2))
def purge(env, payload=None): ...

@api.scheduled_job(name="blog.weekly_digest", cron=SchedulerCron.weekly(weekday="mon", hour=9))
def digest(env, payload=None): ...
```

Every builder returns a plain cron string, so it drops straight into the `cron=` parameter and raw strings keep working:

| Call | Produces |
|---|---|
| `SchedulerCron.run_every(minutes=15)` | `*/15 * * * *` |
| `SchedulerCron.run_every(hours=2)` | `0 */2 * * *` |
| `SchedulerCron.hourly(minute=30)` | `30 * * * *` |
| `SchedulerCron.daily(hour=2)` | `0 2 * * *` |
| `SchedulerCron.weekly(weekday="mon-fri", hour=9)` | `0 9 * * 1-5` |
| `SchedulerCron.monthly(day=1)` | `0 0 1 * *` |
| `SchedulerCron.build(minute=[0, 30], hour=9)` | `0,30 9 * * *` |

`build(minute=, hour=, day=, month=, weekday=)` is the general escape hatch — each field takes an int, a list, or a raw fragment; `weekday` also accepts names and ranges (`"mon"`, `"mon-fri"`).

### Enqueue a background job on demand

Register the callable with `@api.background_job` (no cron), then enqueue it from a command handler, an event, or anywhere you hold an `env`:

```python
@api.background_job(name="blog.export_archive")
def export_archive(env, payload):
    author_id = payload["author_id"]
    ...

# elsewhere — fire it off without blocking the request
env.enqueue_job(target="blog.export_archive", payload={"author_id": author.id})
```

`enqueue_job` creates a pending `ir.job.run` and hands it to the executor; it returns immediately.

### Report progress from a long job

```python
@api.background_job(name="blog.reindex")
def reindex(env, payload):
    posts = env.models["blog.post"].search([])
    for i, post in enumerate(posts):
        ...
        env.job_progress(percent=100 * (i + 1) / len(posts), message=f"{i + 1}/{len(posts)}")
```

`job_progress` updates the live run row; it's a safe no-op when called outside a job, so shared utility code can call it freely.

### Declare a job in XML

Jobs can also be declared as data so an admin can tune them without a code change. Set `source` to `xml` so the boot reconciler leaves the row alone:

```xml
<record id="blog.job_nightly_digest" model="ir.job">
    <field name="name">blog.nightly_digest</field>
    <field name="module_key">blog</field>
    <field name="target">blog.jobs.send_digest</field>
    <field name="kind">scheduled</field>
    <field name="cron">0 7 * * *</field>
    <field name="retry_policy">fixed</field>
    <field name="source">xml</field>
</record>
```

### Run the workers

The executor consumes the queue; the scheduler dispatches due jobs.

```bash
ede jobs-worker --concurrency 4      # executor (prefork pool, consumes the broker)
ede worker                           # event drainer + in-process scheduler + stuck-job reaper
ede worker --no-jobs                 # event drainer only (skip scheduling)
```

### Recover failed runs

When retries are exhausted, a run lands in `dead_letter` and stays queryable. Retry it from the run form ("Retry from Dead Letter") or via the recovery endpoint; the new attempt links back through `parent_run_id`.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `JOBS_ENABLED` | `True` | Master switch for the scheduler and executor. |
| `JOBS_SCHEDULER_TICK_SECONDS` | `10` | How often the in-process scheduler checks for due jobs. |
| `JOBS_DEFAULT_RETRY_POLICY` | `exponential` | Fallback retry policy when a job row doesn't set one. |
| `JOBS_DEFAULT_TIMEOUT_SECONDS` | `600` | Fallback hard execution ceiling per run. |
| `JOBS_CELERY_BROKER_URL` | `redis://localhost:6379/2` | Broker the executor consumes. |
| `JOBS_CELERY_PREFORK_CONCURRENCY` | `4` | Child processes per `ede jobs-worker`. |
| `JOBS_REAPER_TICK_SECONDS` | `60` | How often the reaper marks crashed runs as interrupted. |

## Reference

-   Source: `src/ede/foundation/jobs/`
-   Decorators: `@api.scheduled_job`, `@api.background_job` — see [Commands & Events](../tutorial/04-commands-and-events.md).
-   Related: [Notifications](notifications.md) (dead-letter alerts), [Approval](approval.md) (SLA escalation runs as a scheduled job), [Datasets](dataset.md).
-   Metrics: `GET /api/foundation/jobs/metrics` (Prometheus text exposition).
