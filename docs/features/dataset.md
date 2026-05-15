# Datasets & Metrics

Declarative compute layer for analytics. Define a **dataset** as a blueprint of source models, joins, filters, and metrics; the engine resolves it to a single denormalised result and caches it per run.

```python
# Read a dataset (one HTTP round-trip; the engine plans + executes the query).
result = env.dispatch(Command("dataset.read", payload={
    "dataset": "sales.pipeline_health",
    "filters": {"date_range": ["2026-04-01", "2026-04-30"]},
}))

# `result.rows` — list of dict rows; `result.schema` — column metadata.
for row in result.rows:
    print(row["stage"], row["weighted_value"])
```

Metrics referenced by datasets share the formula engine, AST evaluator, and per-run cache — define a metric once, reuse it across dashboards, reports, and document templates.

---

## What you get

-   **`dataset.blueprint`** — model definition: source, joins, filters, metric references, schema.
-   **`metric.definition`** — a single computed value (sum, avg, count, formula) with `depends_on` tracking.
-   **`dataset.read`** command — entry point for any consumer.
-   **Formula engine** — restricted AST evaluator with a closed function whitelist (numeric set for metrics).
-   **DAG cycle detection** — metric dependencies are checked at blueprint save time.
-   **Per-run cache** — every metric in a request resolves at most once.
-   **Universal result contract** — `DatasetResult` JSON shape consumed by the React grid, document `<rows datasource>`, HTTP / SSE / WS / webhook / export.

## How to use it

### Define a metric

```xml
<metric id="weighted_value">
    <field name="name">Weighted Pipeline Value</field>
    <field name="kind">formula</field>
    <field name="formula">value * (probability_percent / 100)</field>
    <field name="depends_on">["value", "probability_percent"]</field>
</metric>
```

### Define a dataset blueprint

```xml
<dataset id="sales.pipeline_health">
    <field name="source_model">crm.opportunity</field>
    <field name="filters">[("state","not in",["won","lost"])]</field>
    <field name="schema" eval="[
        {'name': 'stage',          'kind': 'string'},
        {'name': 'stage_id',       'kind': 'reference'},
        {'name': 'value',          'kind': 'monetary'},
        {'name': 'probability_percent', 'kind': 'percentage'},
        {'name': 'weighted_value', 'kind': 'monetary', 'metric_id': 'weighted_value'},
    ]"/>
</dataset>
```

### Read from the React client

The same blueprint feeds a list view, a dashboard widget, a PDF report, or a CSV export — change the consumer, not the data layer.

```javascript
const result = await api.command("dataset.read", {
    dataset: "sales.pipeline_health",
    filters: { date_range: [from, to] },
});
```

### Add a runtime filter

```python
env.dispatch(Command("dataset.read", payload={
    "dataset": "sales.pipeline_health",
    "filters": {
        "date_range": ["2026-04-01", "2026-04-30"],
        "owner_id":   user.id,
    },
}))
```

The blueprint declares which filter keys it accepts; unknown keys are rejected.

## Reference

-   Source: `src/ede/foundation/dataset/`
-   Formula engine: `src/ede/core/engines/formula/safe_eval.py`.
-   Result contract: `src/ede/core/engines/integration/contract.py`.
