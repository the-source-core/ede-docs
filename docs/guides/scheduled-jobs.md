# Run work on a schedule or in the background

Register a cron job, enqueue ad-hoc background work, and report progress — all through the jobs runtime. The worker process drains the queue; your code just declares the work.

```python
from ede.core import api


from datetime import datetime, timedelta, timezone


@api.scheduled_job(name="blog.purge_drafts", cron="0 3 * * *")
def purge_old_drafts(env, payload=None) -> dict:
    cutoff = (datetime.now(timezone.utc) - timedelta(days=90)).replace(tzinfo=None)
    stale = env.models["blog.post"].search([
        ("published", "=", False),
        ("updated_at_utc", "<", cutoff),
    ])
    for post in stale:
        post.delete()
    return {"purged": len(stale)}
```

At boot the reconciler creates the matching `ir.job` row; the worker's scheduler dispatches the job every night at 03:00.

---

## 1. Schedule a recurring job

`@api.scheduled_job` decorates a free function that takes `(env, payload=None)` and runs on a cron cadence:

```python
@api.scheduled_job(name="blog.digest_tick", cron="*/15 * * * *")
def send_digest(env, payload=None) -> dict:
    ...
    return {"sent": n}
```

- `name` is the job's stable target identifier.
- `cron` is a standard 5-field expression.
- The function runs inside a worker-scoped `Env` — use `env.models[...]`, `env.dispatch(...)`, and `env.emit(...)` exactly as in a request.

## 2. Declare ad-hoc background work

For work triggered on demand rather than on a clock, use `@api.background_job`:

```python
@api.background_job(
    name="blog.reindex_posts",
    description="Rebuild the search index for an organization's posts.",
    timeout_seconds=600,
)
def reindex_posts(env, payload) -> dict:
    org_id = payload["organization_id"]
    ...
    return {"reindexed": count}
```

Enqueue it from anywhere with `env.enqueue_job`, passing the decorator's `name` as the `target`:

```python
env.enqueue_job(target="blog.reindex_posts", payload={"organization_id": org.id})
```

`enqueue_job` requires `foundation.jobs` to be loaded and the `ir.job` row to exist — the boot reconciler creates it from the decorator, so a decorated job is enqueueable as soon as the app is active.

## 3. Report progress from a long job

Inside any job, call `env.job_progress` to update the run's percent and status message:

```python
@api.background_job(name="blog.reindex_posts", description="...", timeout_seconds=600)
def reindex_posts(env, payload) -> dict:
    posts = env.models["blog.post"].search([])
    total = len(posts)
    for i, post in enumerate(posts):
        ... # index the post
        env.job_progress(percent=(i + 1) / total * 100, message=f"{i + 1}/{total}")
    return {"reindexed": total}
```

The percent and message surface on the `ir.job.run` row and in the Settings → Technical → Jobs admin UI.

## 4. Run the worker

A scheduled or enqueued job only executes when a worker is draining the queue. In development, run the server with an in-process worker:

```bash
ede serve --config ede.conf --with-worker
```

In production, run a dedicated worker process:

```bash
ede worker --config ede.conf
```

## 5. Verify

After boot, the decorated job appears as an `ir.job` row (Settings → Technical → Jobs). Trigger an ad-hoc run with `env.enqueue_job(...)` and watch the `ir.job.run` row move `pending → running → succeeded`, with the progress message updating as it goes.

## Reference

- Decorators: `src/ede/core/api.py` (`scheduled_job`, `background_job`)
- `env.enqueue_job` / `env.job_progress`: `src/ede/core/env.py`
- Job registry + reconciler: `src/ede/foundation/jobs/services/`
- Related: [Background Jobs](../features/jobs.md), [Command & Event Bus](../04-command-event-bus.md).
