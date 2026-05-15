# `logistics.masters` — Demo Use-Case

**Module:** `domains.logistics.masters`
**App key:** `logistics.masters`
**Demo manifest entries** (target): `demo/demo_ports.xml`, `demo/demo_equipment.xml`, `demo/demo_commodities.xml`
**Status:** 🔴 Not yet authored

---

## Use-case

Seed the **logistics reference data** the scenario lane (Mumbai → Singapore) depends on:

- Two demo ports — Mumbai (`INMUM`) and Singapore (`SGSIN`) — as `logistics.unlocode.master` rows (the UN/LOCODE master).
- One demo equipment line — a 20 ft Standard container — referenced by the pricing demo and (later) booking demo.
- One demo commodity — General Cargo / FAK — for the rate to apply to.

Production seeds already ship the **catalogue** of UN/LOCODEs, equipment categories, transport modes, etc. Demo only adds these *example* rows that bind to the scenario; this keeps the demo footprint small and predictable.

## Records produced

### `demo/demo_ports.xml`

| External ID | Model | Notes |
|---|---|---|
| `masters.demo_port_inmum` | `logistics.unlocode.master` | code=`INMUM`, name="Mumbai", country=IN, function="port" |
| `masters.demo_port_sgsin` | `logistics.unlocode.master` | code=`SGSIN`, name="Singapore", country=SG, function="port" |

### `demo/demo_equipment.xml`

| External ID | Model | Notes |
|---|---|---|
| `masters.demo_equipment_20gp` | `logistics.equipment.master` | code=`20GP`, category=`dry`, size_ft=20 |

### `demo/demo_commodities.xml`

| External ID | Model | Notes |
|---|---|---|
| `masters.demo_commodity_fak` | `logistics.commodity.master` | code=`FAK`, name="Freight All Kinds" |

## Out of scope

- Standard UN/LOCODE catalogue — production seed (`data/logistics.unlocode.master.csv` if/when added).
- Equipment categories / sub-categories — production seeds.
- Incoterms / charge codes — production seeds.
- Trade-lane / route master rows — Phase 2 of masters.

## Dependencies

- `foundation.base` — `res.country` rows IN and SG already seeded in production data.

## Verification

```
ede migrate upgrade -t demo --with-demo=logistics.masters
```

`select code, name from logistics_unlocode_master where code in ('INMUM','SGSIN');` returns both rows. Pricing + sales-CRM demo files now have valid lane endpoints to `ref=`.

## Authoring order

1. Ports first — pricing/sales-CRM demos all `ref=` them.
2. Equipment + commodities can load in any order after ports.

Manifest `demo: [...]` order:
```python
"demo": [
    "demo/demo_ports.xml",
    "demo/demo_equipment.xml",
    "demo/demo_commodities.xml",
],
```

---

*Back to [demo-usecase index](../../README.md).*
