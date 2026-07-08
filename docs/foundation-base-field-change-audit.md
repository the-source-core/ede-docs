# Field-Change Audit Log — `ir.field.change.log`

## What this is

`ir.field.change.log` is the platform's generic, append-only **"who changed what field, on which record, when"** trail. Any model opts in by listing the fields it wants audited; the framework then persists one immutable row per audited-field mutation — no per-domain audit table, no hand-wired event handler, no bespoke history UI.

It is the persistence counterpart to the existing `__ede_track_fields__` event pipeline: tracking *emits a transient event*, auditing *persists a durable row*. The two are independent and composable.

## Three-line opt-in

```python
@api.model("crm.quote.version")
class QuoteVersion(DomainModel):
    total_sell_amount = fields.Decimal()
    version_status = fields.Char(max_length=30)

    __ede_audit_fields__: ClassVar[List[str]] = ["total_sell_amount", "version_status"]
```

That's it. On every `ede.update` that changes `total_sell_amount` or `version_status`, the framework writes an `ir.field.change.log` row capturing the old value, new value, the acting user, the active organization, the triggering command, and a UTC timestamp. No further wiring.

## `__ede_track_fields__` vs `__ede_audit_fields__`

| Attribute | Effect | When to use |
|---|---|---|
| `__ede_track_fields__` | Emits a transient `{model}.field_changed` **event** (consumed in-process by `@api.on_event` handlers; not persisted). | Live reactions — reload a client, recompute a cache, notify a subscriber. |
| `__ede_audit_fields__` | Persists a durable `ir.field.change.log` **row**. | Commercial / regulatory / compliance history you must be able to show later. |

A field may appear in one, both, or neither — they are independent metadata. If neither is set, no events fire and no rows are written: **zero overhead for non-audited models.**

## When to opt in (and when not)

Opt in for fields whose change history has business or regulatory meaning: monetary amounts, statuses that gate a workflow, exchange-rate snapshots, contractual terms, anything a customer or auditor might dispute later.

Do **not** opt in for high-churn operational fields (progress counters, last-seen timestamps, cache columns) — the audit log is not a change-data-capture firehose, and every audited write costs one insert.

## How it works

1. A shared `pre.{model}.update` hook captures the prior DB values of the audited (and tracked) fields onto the command.
2. A synchronous `post.{model}.update` hook — auto-wired by `register_model()` when `__ede_audit_fields__` is non-empty — diffs the payload against those prior values and writes one row per field that actually changed.
3. The write goes through a **direct repo** that reuses the mutating command's unit of work. It bypasses the command bus (no recursion, no lifecycle-hook re-entry) and, because it shares the business write's transaction, the audit row commits or rolls back atomically with the change it records.

Capture is synchronous and in-transaction — not routed through the async event queue — so every row carries the true acting `changed_by_user_id` (`env.principal`), `organization_id` (`env.active_organization_id`), and `command_name` (`cmd.name`), none of which survive a trip through a background worker.

## The stored row

| Column | Meaning |
|---|---|
| `model_key` | The audited model (e.g. `crm.quote.version`). |
| `target_record_uuid` | The audited record's `record_uuid`. (Named `target_*` to avoid colliding with the audit row's own auto `record_uuid`; the HTTP API and `<audit-trail/>` widget speak plain `record_uuid`.) |
| `field_name` | The field that changed. |
| `old_value_json` / `new_value_json` | JSON-serialized prior / new value. |
| `changed_by_user_id` | Reference to `res.user` — the acting principal (null for system writes). |
| `changed_at_utc` | Capture time (UTC). |
| `command_name` | The triggering command (`ede.update`, or a domain command). |
| `organization_id` | Reference to `res.organization` — the active org at write time. |

### FK / value serialization contract

Reference (FK) fields serialize the target's **`record_uuid` string**, never its `dbid` — the ORM stores and returns FKs as UUIDs, so the captured value is a stable external identity. `Decimal` values serialize to their exact textual form (no float rounding), and `datetime` / `date` / `UUID` values to their ISO / string form. Primitives pass through unchanged.

## Append-only enforcement

`ir.field.change.log` is immutable. `pre.ir.field.change.log.update` and `pre.ir.field.change.log.delete` hooks veto every command-bus write — there is no update or delete API. The only two legitimate producers are the framework's own capture path (create) and the retention prune sweep (a direct-repo hard delete that bypasses the command bus).

## Surfacing history in a form — `<audit-trail/>`

Drop the self-closing element into any form view (typically a notebook page):

```xml
<form>
    <notebook>
        <page label="Audit Trail">
            <audit-trail/>
        </page>
    </notebook>
</form>
```

Optional attributes: `title` (header override) and `limit` (initial page size). The React widget fetches `GET /api/foundation/base/field-change-log?model_key=…&record_uuid=…` for the loaded record and renders a newest-first table (when · field · from → to · by).

**RBAC:** the endpoint gates on `read` of the *parent model* — if you can read the record, you can read its audit. No separate audit permission to grant.

## Retention

A weekly scheduled job (`ir.field.change.log.prune`, Sunday 03:00) hard-deletes rows older than the retention window. Two system-scope `ir.config` keys, surfaced under **Settings → General → Field-Change Audit**, govern it:

| Key | Default | Purpose |
|---|---|---|
| `ir.field.change.log.retention_days` | `2555` (~7 years, SOX / GDPR) | Maximum row age before the sweep removes it. |
| `ir.field.change.log.retention_enforce` | `true` | When `false`, the sweep is a no-op — audit grows unbounded (forensic / compliance hold). |

### Soft-degrade when the jobs engine is absent

The prune is a config-defined `ir.job` owned by `foundation.jobs` (the module that owns the scheduler), pointing at a target that lives in `foundation.base`. If `foundation.jobs` is not active in a deployment, the job simply never fires and the audit log grows unbounded — the same soft-degradation other platform sweeps use. Auditing itself does not depend on the jobs engine.

## Consumer adoption checklist

1. Add `__ede_audit_fields__ = [...]` to the model — list only the fields whose history matters.
2. Add an `<audit-trail/>` element to the form view (usually a dedicated notebook page).
3. Nothing else — the migration for `ir.field.change.log` ships in `foundation.base`; the capture hook, endpoint, retention job, and RBAC are all platform-provided.
