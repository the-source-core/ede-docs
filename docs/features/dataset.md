# Dataset & Metric Engine

Turn a declarative JSON spec into a safe, tenant-scoped SQL query and a renderer-agnostic JSON result. Two surfaces flow into the same compiler: code-authored metrics for invariant business truths, and a low-code Blueprint UI for ad-hoc datasets — both produce the same `DatasetResult` envelope.

```python
from ede.core import api

@api.metric("base.partner_count")
class PartnerCount:
    name = "Active Partner Count"
    spec = {
        "model": "res.partner",
        "alias": "p",
        "fields": ["count(p.record_uuid):n"],
    }
    result_mode = "scalar"
    value_field = "n"
    default_value = 0
```

```bash
curl -X POST /api/metric/base.partner_count/run \
    -H "Authorization: Bearer $TOKEN" \
    -d '{}'
```

The HTTP route compiles the spec to SQL, executes it in the caller's tenant, and returns a `DatasetResult` JSON envelope ready for any consumer (reporting, dashboards, documents, the future MCP server).

---

## What you get

-   **`@api.metric("key")`** — register a code-authored metric. The class holds the JSON spec; the decorator freezes it into a process-wide registry on import.
-   **`DatasetCompiler`** — converts a JSON spec into parameterized SQL. Handles joins, multi-aggregate selects, GROUP BY, ORDER BY, domain filters, tenant scoping, and the `active` flag.
-   **`DatasetResult`** envelope (`CONTRACT_VERSION = 1`) — TypedDict shape with `meta`, `schema`, and `rows`. Stable across consumers and versions.
-   **`@api.metric(... engine="formula")`** — derive a metric from other metrics. Declare `depends_on=[...]` and an `expr="{{a}} + {{b}}"` template; the formula engine evaluates the expression against a shared safe AST evaluator.
-   **`@api.metric(... engine="plan")`** — multi-spec assembly with an optional `post_process` callable. Run several specs in one pass, combine their rows in Python.
-   **DAG cycle detection at registration time** — circular `depends_on` declarations raise on import, not at first run.
-   **Per-run metric cache** — repeated lookups of the same metric inside one request reuse a deep-copied result. Toggle with `METRIC_CACHE_ENABLED`.
-   **`ir.dataset.blueprint` + 5 child models** — low-code authoring surface. Pick a base model, add joins, declare aggregates and groups; the form emits the same JSON spec a Python metric writes by hand.
-   **HTTP routes** — `POST /api/dataset/run`, `POST /api/metric/{key}/run`, `GET /api/metric/list`. Auth-protected; honors the caller's RBAC and tenant.
-   **Settings → Technical → Datasets** admin UI — Blueprint list, form, and live SQL preview on save.

## How to use it

### Register a metric

Place metric classes under any module's `tools/metrics/` (or wherever your module imports run); the `@api.metric` decorator registers them at import time.

```python
@api.metric("base.organization_count")
class OrganizationCount:
    name = "Active Organization Count"
    spec = {
        "model": "res.organization",
        "alias": "o",
        "fields": ["count(o.record_uuid):n"],
    }
    result_mode = "scalar"
    value_field = "n"
    default_value = 0
```

The `spec` is the canonical JSON contract — the same shape a Blueprint UI produces on save. `result_mode="scalar"` returns a single value with `value_field` naming the column; the default mode is `"rows"` and returns the full row list.

### Run a metric

```python
result = env.dispatch(Command(
    name="ede.metric.run",
    payload={"key": "base.partner_count"},
))
# result is a DatasetResult dict: meta, schema, rows
```

Or over HTTP:

```http
POST /api/metric/base.partner_count/run
Authorization: Bearer <token>
Content-Type: application/json

{}
```

The response is a `DatasetResult` envelope. The compiler scopes the query to the caller's `tenant_id` automatically — never put `tenant_id` in the spec by hand.

### Compose metrics with the formula engine

```python
@api.metric("base.entity_total")
class EntityTotal:
    name = "Total Entities (Partners + Organizations)"
    engine = "formula"
    depends_on = ["base.partner_count", "base.organization_count"]
    expr = "{{base.partner_count}} + {{base.organization_count}}"
    result_mode = "scalar"
```

At run time the executor evaluates each dependency, substitutes its scalar value into the `expr` template, then evaluates the expression with the shared safe AST evaluator (no `eval`, no globals, restricted to numeric ops + a curated function set). Cycle detection at registration prevents an `a → b → a` chain from ever loading.

### Compose metrics with the plan engine

```python
@api.metric("base.entity_summary")
class EntitySummary:
    name = "Entities by type"
    engine = "plan"
    depends_on = ["base.partner_count", "base.organization_count"]

    @staticmethod
    def post_process(results):
        return {
            "rows": [
                {"kind": "partner", "n": results["base.partner_count"]["rows"][0]["n"]},
                {"kind": "organization", "n": results["base.organization_count"]["rows"][0]["n"]},
            ],
        }
```

The plan engine runs each `depends_on` metric in one pass, then hands the dict of `DatasetResult`s to your `post_process`. Use it whenever a single user-visible result needs data from multiple compiled specs.

### Author a Blueprint (no Python)

Navigate to **Settings → Technical → Datasets → New**. Pick a base model, add Field rows (the SELECT list — optionally with an alias and aggregate), Connection rows (JOINs), Group rows (GROUP BY), and Sort rows (ORDER BY). The form previews the compiled SQL on save. Locking a Blueprint freezes its spec and exposes it through the same `ede.dataset.run` command Python metrics use.

### List the metric registry

```http
GET /api/metric/list
```

Returns `[{"key": "base.partner_count", "name": "Active Partner Count", "engine": "dataset"}, ...]` — useful for any consumer that needs to enumerate the registry (dashboard pickers, the future MCP server, documentation).

## JSON spec shape

A minimal `dataset` spec:

```json
{
    "model": "res.partner",
    "alias": "p",
    "fields": ["p.name", "count(p.record_uuid):n"],
    "domain": [["active", "=", true]],
    "groups": ["p.name"],
    "sorts": [{"field": "n", "direction": "desc"}]
}
```

Field selectors accept dotted-path traversal (`p.organization_id.name`) and aggregate functions (`count`, `sum`, `avg`, `min`, `max`, `count_distinct`). The compiler validates every reference against the registry — unknown fields raise `DatasetCompileError` before any SQL runs.

The `DatasetResult` returned has:

```json
{
    "meta": {"contract_version": 1, "model": "res.partner", "row_count": 1},
    "schema": {"columns": [{"name": "n", "kind": "scalar", "type": "integer"}]},
    "rows": [{"n": 42}]
}
```

`contract_version` is the explicit envelope version — consumers can refuse a `DatasetResult` from an incompatible future engine.

## Configuration

| Setting | Default | What it controls |
|---|---|---|
| `DATASET_DEFAULT_QUERY_TIMEOUT_SECONDS` | `30` | Hard timeout on compiled SQL execution. Raises `DatasetTimeoutError`. |
| `DATASET_MAX_RESULT_ROWS` | `100000` | Cap on rows returned by any single run. Raises `DatasetRowLimitExceeded`. |
| `METRIC_CACHE_ENABLED` | `True` | Per-run deep-copy cache for repeated metric lookups in one request. |

## How it composes with other features

-   [Commands & events](../tutorial/04-commands-and-events.md) — `ede.metric.run` and `ede.dataset.run` are first-class commands; you can dispatch them from any controller, hook, or worker.
-   [Permissions](security.md) — 4 RBAC roles ship out of the box (`dataset.viewer`, `dataset.author`, `dataset.publisher`, `dataset.admin`); domain teams attach record rules to scope rows.
-   [Form views](../tutorial/03-views-and-web-client.md) — the Blueprint admin form is a normal `<FormView>` with `<notebook>` tabs; the live SQL preview is just a computed field.

## Reference

-   Foundation shell (models, controllers, views, RBAC): `src/ede/foundation/dataset/`
-   Compiler: `src/ede/core/engines/dataset/compiler.py`, `field_resolver.py`, `expressions.py`
-   Metric registry & decorator: `src/ede/core/engines/metric/registry.py`, `decorator.py`
-   Formula engine: `src/ede/core/engines/metric/formula_engine.py`
-   Plan engine: `src/ede/core/engines/metric/plan_engine.py`
-   DAG cycle detector: `src/ede/core/engines/metric/dag.py`
-   Per-run cache: `src/ede/core/engines/metric/cache.py`
-   Shared safe AST evaluator: `src/ede/core/engines/formula/safe_eval.py`
-   JSON contract: `src/ede/core/engines/integration/contract.py`
-   HTTP routes: `src/ede/foundation/dataset/controllers.py`
-   Built-in demo metrics: `src/ede/foundation/dataset/tools/metrics/base_metrics.py`
