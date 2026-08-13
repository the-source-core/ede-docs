# Logistics Vertical — Three-Domain Structure & NVOCC Enablement

> **Status:** Accepted structure — execution plan. Hand-authored architecture guidance; not auto-maintained from the roadmap.
> **Decision:** A **tiered domain** layout — one base domain holding the common modules, two product domains (freight forwarding, NVOCC) depending on it.
> **Source scope:** SOW — *NVOCC / Shipping Agency Masters, Admin & Control Foundation* (BRD N1, 33 deliverables).
> **Companions:** [Model & View Extension SDK](foundation-base-extensions.md) · [Module Integration Pattern](module-integration-pattern.md) · [CLAUDE.md](../CLAUDE.md).

---

## 1. The structure

Three domain packages under `src/domains/`. Common modules live in the base domain; each product domain holds only what is genuinely its own.

```
BASE DOMAIN
┌────────────────────────────────────────────────────────────────────┐
│  src/domains/logistics_base/                                       │
│  masters · equipment_control · pricing · sales_crm · booking ·     │
│  shipments · operations_execution · profitability · shared shapes  │
└──────────────────────────┬──────────────────────┬──────────────────┘
                           │ depends on           │ depends on
PRODUCT DOMAINS            │                      │
┌──────────────────────────┴─────────┐  ┌─────────┴──────────────────┐
│ src/domains/                       │  │ src/domains/               │
│   logistics_freight_forwarding/    │  │   logistics_nvocc_agency/         │
│ consolidation · documentation ·    │  │ agency                     │
│ tracking                           │  │                            │
└────────────────────────────────────┘  └────────────────────────────┘
                  ╳  never depend on each other  ╳
```

App keys are derived from folder position, so they read exactly as intended:

| Package | App keys |
|---|---|
| `src/domains/logistics_base/` | `logistics_base.masters`, `logistics_base.equipment_control`, … |
| `src/domains/logistics_freight_forwarding/` | `logistics_freight_forwarding.shipments`, `logistics_freight_forwarding.consolidation`, … |
| `src/domains/logistics_nvocc_agency/` | `logistics_nvocc_agency.agency_masters` |

### Activation

```python
# src/domains/settings.py — base domain MUST come first (see §2)
ACTIVE_DOMAINS = ["logistics_base", "logistics_freight_forwarding", "logistics_nvocc_agency"]   # full-service customer
ACTIVE_DOMAINS = ["logistics_base", "logistics_freight_forwarding"]                      # freight forwarding only
ACTIVE_DOMAINS = ["logistics_base", "logistics_nvocc_agency"]                                   # NVOCC / agency only
```

### This needs no framework change — verified

The loader already supports it as-is:

- `load_domains()` walks `ACTIVE_DOMAINS` **in order**, and `_validate_depends` requires a declared dependency to be **already loaded**. So listing `logistics_base` first is the whole ordering requirement.
- Cross-domain `depends` entries are accepted — they must be fully-qualified app keys, e.g. `"logistics_base.masters"`. Bare folder names are rejected.
- `load_app(domain_type=…, root_package=…)` is fully parameterised; nothing is hardcoded to a single domain.

Probed directly against `ModuleLoader._validate_depends`:

| Case | Result |
|---|---|
| `logistics_nvocc_agency.agency_masters` → `logistics_base.masters`, base loaded first | ✅ **Accepted** — the structure works today |
| Same dependency, base **not** loaded yet (wrong `ACTIVE_DOMAINS` order) | ✅ **Rejected**, with a clear message naming the missing key — ordering is self-enforcing |
| `logistics_nvocc_agency.agency_masters` → `logistics_freight_forwarding.shipments` (peer coupling) | ⚠️ **Accepted** — the loader does **not** block it |

So the structure is a **packaging and governance** change, not a kernel change — and the third row is why §2's boot guard is not optional.

### Naming

Use **`logistics_base`**, not `logistics_foundation`. "Foundation" already names the platform layer (`src/ede/foundation/`, app keys `foundation.*`); a domain called `logistics_foundation` would read as if it lived there. `logistics_base` is unambiguous.

The underscore in the app key (`logistics_base.masters`) is correct and not a convention breach — the "no underscores inside a segment" rule governs **model keys**; app keys derive from folder names, and folder names are `snake_case`.

---

## 2. The one governance change — and how to make it enforceable

CLAUDE.md currently states the rule absolutely: *"Domains never depend on other domains."* This structure requires that to be re-expressed, because `logistics_freight_forwarding → logistics_base` is a domain→domain edge by the letter of it.

**The invariant that actually matters is unchanged: peer product domains must never couple.** Reformulate the rule around domain *tiers*:

| Tier | May depend on | Must never depend on |
|---|---|---|
| **Base domain** (`logistics_base`) | `foundation.*` only | Any product domain |
| **Product domain** (`logistics_freight_forwarding`, `logistics_nvocc_agency`) | `foundation.*`, any **base** domain, own siblings | Another **product** domain |

That keeps the graph acyclic and one-directional, keeps each product domain independently activatable, and states plainly the one edge that is now legal.

### Make it a guard, not a comment

The loader will happily accept `logistics_freight_forwarding → logistics_nvocc_agency` — it only checks load order. A documented rule that nothing enforces is the risk this structure introduces, so close it at boot. Everything needed is already in the registry: `AppSpec` carries `domain_type` and `depends`.

Declare the tiers:

```python
# src/domains/settings.py
ACTIVE_DOMAINS = ["logistics_base", "logistics_freight_forwarding", "logistics_nvocc_agency"]
BASE_DOMAINS   = ["logistics_base"]      # tier declaration — everything else is a product domain
```

Add a validator beside the existing ones:

```python
# src/ede/core/bootstrap.py — next to validate_extensions() / validate_delegated_declarations()
validate_domain_tiers(registry)
```

For every loaded domain app, for each declared dependency:

| Case | Verdict |
|---|---|
| dep is `foundation.*` | ✅ allow |
| dep's domain == own domain (sibling module) | ✅ allow |
| own domain is a **product** domain, dep's domain is a **base** domain | ✅ allow |
| own domain is a **base** domain, dep's domain is a product domain | ❌ **fail** — base must not depend on a product |
| both are product domains, and they differ | ❌ **fail** — peer coupling |

~25 lines, follows the established boot-validator pattern exactly (`validate_extensions`, `validate_delegated_declarations`, `validate_activity_components`), and turns the tier rule into something a mistake cannot pass. Worth landing in Phase 1 — it is the guard that makes the whole structure safe to hand to a team.

---

## 3. Which modules go in the base domain

**The test:** does the module serve both businesses with the *same semantics*? Yes → base. Divergent semantics → keep the **shape and the engine** in base, and the **concrete records** in each product domain.

**When unsure, put it in the base domain.** The two mistakes are not symmetric: a module in base that only one product uses is harmless over-inclusion, while a module in a product domain that the other product turns out to need leaves you blocked — or tempted into exactly the peer dependency the tier rule forbids.

**The graph constrains the answer.** Every module's declared dependencies were resolved transitively: putting a module in the base **forces its entire dependency closure into the base with it**. Only four modules are reverse-leaves — nothing depends on them — so only those can stay in a product domain without dragging others across.

| Module | Tier | Forced by | Note |
|---|---|---|---|
| `masters` | **base** | — (safe leaf) | All 42 masters are needed by both — see §5. |
| `equipment_control` | **base** | needs `masters` | Today's `equipment_operations`. One shared container/equipment registry, so container numbers stay globally unique and one register report covers both businesses. |
| `pricing` | **base** | needs `masters` | Both businesses price; NVOCC pricing (BRD N2) builds on the same rate engine. |
| `sales_crm` | **base** | needs `masters`, `pricing` | Both sell, and quote → handover is the same flow. Owns `crm.*` keys. |
| `booking` | **base** | needs `equipment_control`, `sales_crm`, `pricing` | Both book capacity. Depended on by `shipments` and `profitability`. |
| `shipments` | **base** | needs `booking`, `sales_crm` | **Not optional** — `consolidation`, `documentation`, `operations_execution`, `profitability` and `tracking` all depend on it. |
| `operations_execution` | **base** | reverse-leaf, but needs `shipments` | Shared task / milestone / cutoff engine. |
| `profitability` | **base** | reverse-leaf, but needs `shipments` | Shared costing and margin. |
| `consolidation` | **`logistics_freight_forwarding`** | — | Co-load / consol as modelled today is freight-forwarding-specific. |
| `documentation` | **`logistics_freight_forwarding`** | needs `consolidation` | Reverse-leaf. Travels with `consolidation`. |
| `tracking` | **`logistics_freight_forwarding`** | needs `consolidation` | Reverse-leaf. Travels with `consolidation`. |
| `agency` | **`logistics_nvocc_agency`** | — | The only genuinely NVOCC-owned module: principal, entitlement, agency governance. |

Those last three move together or not at all, because `documentation` and `tracking` both depend on `consolidation`. Keeping them out also matches the SOW, where bill-of-lading, manifest and container-tracking *execution* are separate downstream specifications. Verified tier-valid: nothing in the base depends on any of the three.

### SOW Phase 2 is base work, not NVOCC work

The SOW's Phase 2 is titled *"Container & Equipment Control"* (D2.1–D2.8), which reads like an NVOCC module — it isn't. Every one of those deliverables targets a **base-domain** model:

| SOW | Target | Lands in |
|---|---|---|
| D2.1 Container type master | `logistics.equipment.type.master` | `logistics_base.masters` |
| D2.2 Ownership setup · D2.3 Container master · D2.8 Check digit | `logistics.equipment` | `logistics_base.equipment_control` |
| D2.4 Status lifecycle · D2.5 Movement crosswalk | equipment status masters, movement events | `logistics_base` |
| D2.6 Yard / CFS / depot · D2.7 Terminal / ICD / port | `logistics.facility.master`, `logistics.unlocode.master` | `logistics_base.masters` |

The freight product already ships equipment tracking today, and ISO check digits, ownership types and CODECO/COARRI event codes are container concerns, not agency concerns. **NVOCC owns exactly one slice of Phase 2:** `principal_id` on the equipment record, contributed via `@api.extend_model("logistics.equipment")` from `logistics_nvocc_agency.agency_masters` — because it references `nvocc.principal`, and the base can never point at a product domain.

Two calls remain genuinely yours, because they are business-semantics questions rather than graph-forced ones: whether **booking** should be one shared model or per-product concretes over a shared shape, and how much of **documentation** NVOCC will eventually need.

---

## 4. Model keys stay as they are

**Do not rename model keys when modules move between domains.** Model keys are declared in the decorator; app keys derive from folder position. They are independent, so relocating a module changes the app key and nothing about model identity.

Audited blast radius of the masters move:

| Measure | Count |
|---|---|
| Models owned by `masters` (all move to base) | **42** |
| Models owned by the whole current logistics domain | 162 |
| References to masters-owned keys from **outside** masters | **284** across **72 files** |
| Data seed files in `masters` | 32 |
| Migrations in `masters` | 13 |

Keeping the keys preserves all 284 references, all 32 seed files — including the CSVs, whose **filenames must exactly match the target model key** — every view, every stored dataset blueprint, and the on-demand trade-location sync, which targets `logistics.unlocode.master` by key.

### Prefix convention across the three tiers

| Prefix | Owner | Meaning |
|---|---|---|
| `res.*` / `ir.*` | Platform | Universal resources / framework metadata |
| `logistics.*` | **`logistics_base`** | Shared across the logistics vertical |
| `mixin.*` | `logistics_base` | Keyed abstract shapes (§6) |
| `logistics_ff.*` | `logistics_freight_forwarding` | Freight-forwarding-specific, **new models only** |
| `nvocc.*` | `logistics_nvocc_agency` | NVOCC / agency-specific |

**Existing FF model keys are grandfathered.** `logistics.shipment`, `logistics.booking` and friends keep their keys even though they sit in a product domain — renaming them means a table rename plus FK repointing across ~10 child models plus invalidation of every stored reference naming the old key, for zero capability. New FF models take `logistics_ff.*`. The prefix then tells you the tier for everything written from here on.

> ⚠️ **CLAUDE.md amendment required**, in one change, before Phase 1 lands: the tier rule (§2), the vertical/mixin prefixes above, and the Model Placement Test rewritten to ask *"which tier?"* rather than *"platform or domain?"*.

---

## 5. What lifts into `logistics_base.masters`

All 42 masters, wholesale — auditing each against both businesses, every one is needed by both.

| Group | Models | Needed by NVOCC because |
|---|---|---|
| Trade locations | `unlocode.master` | Ports of load / discharge on every BL and manifest |
| Facilities | `facility.master`, `facility.zone.master`, `facility.type.master`, `facility.zone.type.master`, `chl.type.master` | Yard / CFS / depot / terminal / ICD — SOW D2.6, D2.7 |
| Equipment | `transport.mode.master`, `shipment.type.master`, `load.type.master`, `equipment.category/subcategory/identifier.type/type.master` | Container type + ISO mapping — SOW D2.1 |
| Operational states | `equipment.status.master`, `equipment.condition.master`, `usage.status.master`, `movement.event.type.master` | Container status lifecycle + movement mapping — SOW D2.4, D2.5 |
| Maintenance | `maintenance.type/status.master`, `inspection.type/result.status.master` | Repair yard, under-repair state, EIR inspection |
| Security | `seal.type.master`, `seal.change.reason.master` | Seal control on gate-in / gate-out |
| Documents | `document.type.master`, `reference.context.type.master` | BL / manifest / DO document typing |
| Products & trade | `incoterm.master`, `product.master`, `commodity.master`, `trade.lane` | Commodity and lane on every booking |
| Cargo | `package.type`, `haz.class` | Packing and dangerous-goods declaration |
| Parties | `carrier`, `carrier.agent`, `provider` | Principal's carrier network, agent network — SOW D3.3 |
| Vessels | `vessel`, `vessel.category` | Vessel call, manifest, voyage reference |
| Rules & misc | `service.mode`, `volumetric.rule`, `lane.restriction`, `sailing.schedule`, `vendor.service`, `tag` | Chargeable weight, routing restrictions, schedules, vendor roles |

Don't cherry-pick. If a master later proves genuinely FF-only, pushing it down into `logistics_freight_forwarding` is a small follow-up; splitting up front risks guessing wrong in the place hardest to change later.

**Also build here:** the **charge-code master**, which does not exist anywhere today (only transactional charge lines). Pricing, booking, shipments and documentation all need it, and the platform localization roadmap already names `logistics.charge` as its canonical tax-coding target.

---

## 6. Shared shapes — keyed abstract mixins

Where two product domains need the *same shape* but their own records, declare a keyed abstract in `logistics_base` and let each product domain subclass it.

### Framework support (verified)

- `AbstractDomainModel` — declaration-only base; **no table, no migration**.
- `@api.model(key, abstract=True)` — gives the abstract its own model key.
- Field **and** constraint collection walk the full MRO, so everything on the base flows into every concrete.
- Methods, `@api.on_command` and `@api.on_hook` inherit normally, including `super()`.
- Shipped precedents used by ~40 models: `Chatterable`, `Attachable`.

### ⚠️ The keyed path is broken today — fix it first

`@api.model(key, abstract=True)` sets `__ede_model_key__` and returns early without registering. But the loader's model scan gates **only** on "does this class have a model key?" — there is no abstract guard anywhere in `loader.py` — so it calls `register_model()`, which raises:

```
InvalidHandler: Abstract model cannot be registered: ShipmentBaseProbe
```

Verified by probe. `abstract=True` has **zero usages** in the codebase — every abstract today subclasses `AbstractDomainModel` *without* a decorator, so it has no key and never reaches that path. Three small additive changes fix it, with no effect on any existing model:

1. `Registry` — add `_abstracts_by_key`, `register_abstract(cls)`, `get_abstract(key)`, `list_abstracts()`.
2. `ModuleLoader._register_models_from_module` — route classes whose `__ede_abstract__` is true to `register_abstract()`.
3. Boot — validate lineage in the same pass as `validate_extensions()`, and derive `__ede_inherits__` from the MRO (see below).

Approval-gated kernel change. Ship it with tests — the path is entirely uncovered.

### Derive the lineage; don't declare it

Don't make `inherit="mixin.key"` the *mechanism*. Field collection happens in the metaclass by walking the MRO, so a key-only edge with no Python inheritance would give you fields and nothing else — no methods, no commands, no hooks, no `super()`, and no IDE navigation. Since shared behaviour is the point, keep Python inheritance and **derive** the keyed lineage: at `register_model`, walk `cls.__mro__`, collect `__ede_model_key__` from every base whose `__ede_abstract__` is true, store as `__ede_inherits__`.

That gives the registry-queryable lineage ("which models inherit `mixin.package.measure`?") with zero authoring burden and no possibility of drift. If the team wants it visible at the call site, add `inherit=` as a boot-validated **assertion** — the same pattern `delegated=` follows.

### How a product domain consumes a base-domain mixin

The model key is for introspection. The inheritance is wired by a **plain Python import**, legal because the product domain declares the base domain in its manifest.

```python
# src/domains/logistics_base/masters/models/abstracts/package_measures.py
from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import AbstractDomainModel

@api.model("mixin.package.measure", abstract=True)
class PackageMeasureMixin(AbstractDomainModel):
    """Dimensions, weights, volume. No table — contributes columns to each consumer."""
    length           = fields.Decimal(precision=12, scale=3, label="Length")
    width            = fields.Decimal(precision=12, scale=3, label="Width")
    height           = fields.Decimal(precision=12, scale=3, label="Height")
    dimension_uom_id = fields.Reference("res.uom", on_delete="set null", label="Dimension UOM")
    gross_weight     = fields.Decimal(precision=14, scale=4, label="Gross Weight")
    tare_weight      = fields.Decimal(precision=14, scale=4, label="Tare Weight")
    weight_uom_id    = fields.Reference("res.uom", on_delete="set null", label="Weight UOM")
    volume           = fields.Decimal(precision=14, scale=4, label="Volume")
    volume_uom_id    = fields.Reference("res.uom", on_delete="set null", label="Volume UOM")

    # Shared behaviour — declared once, inherited by every consumer.
    def cbm(self): ...
    def chargeable_weight(self, volumetric_ratio): ...
```

Export it for one stable import path, and import it from the owning module's `models/__init__.py` so it registers even when no consumer is active:

```python
# .../masters/models/abstracts/__init__.py
from .package_measures import PackageMeasureMixin      # noqa: F401
```

A **product-domain** consumer — the freight-forwarding manifest line, which needs the same measurements as a base cargo line but keeps its own records:

```python
# src/domains/logistics_freight_forwarding/consolidation/models/manifest_line.py
from ede.core import api
from ede.core.kernel import fields
from domains.logistics_base.masters.models.abstracts import PackageMeasureMixin

@api.model("consolidation.manifest.line", description="Manifest Line")
class ManifestLine(PackageMeasureMixin):
    console_id  = fields.Reference("consolidation.console", on_delete="cascade", required=True)
    marks_nos   = fields.Char(max_length=256)
    package_qty = fields.Integer()
```

```python
# src/domains/logistics_freight_forwarding/consolidation/__manifest__.py
"depends": ["foundation.base", "logistics_base.masters", "logistics_base.shipments"],
```

The concrete's table carries its own columns **plus** all nine mixin columns. The mixin gets no table and no migration. A base-domain consumer — `logistics.shipment.cargo.line` — inherits the identical shape from the identical declaration, which is the whole point.

### Composition and collisions (verified)

- Mixin fields merge into the concrete's field set.
- On a field-name collision between two mixins, **the first base in the list wins** — standard Python precedence, and it happens **silently**. Coordinate or prefix names across mixins.
- The concrete's own declaration overrides any mixin's.

### "Extending" a mixin — three different cases

| You want to… | Do this | Not this |
|---|---|---|
| Add product-specific fields alongside the mixin's | Declare them **on the concrete** | — |
| Add a field every consumer of the mixin needs | Edit the mixin in `logistics_base` — you own that module | `@api.extend_model` — it cannot target an abstract |
| Add a field to a **concrete** model owned by another module | `@api.extend_model("that.model.key")` + your own migration | Editing their model file |

`@api.extend_model` never applies to a mixin: its validator resolves targets from the registered-*model* map, which abstracts never enter. The extend SDK is for models owned by modules you shouldn't edit — `logistics_base` is yours, so you edit it.

### Mixin for shapes; shared concrete for registries

A mixin **copies columns into each consumer's own table**. That is right for a repeating *shape* (measures, party lines, charge lines, addresses). It is wrong for a shared *registry* — two concretes means two tables, so identity is not unique across the platform and no single query spans both.

So: `equipment_control` lives in `logistics_base` as a **concrete** model that both product domains reference (§3), and each product domain adds its own fields via `@api.extend_model`. Reserve mixins for shapes.

Where something must FK to "either kind" of a per-domain record, an abstract cannot help — a FK cannot target one. Use **delegation** (`delegated={parent_key: fk_field}`, already shipped and used by `res.organization`, `res.user`, `logistics.carrier`): a shared concrete parent that each product concrete delegates to, so the child FKs the parent. Costs one extra row and a join; methods still don't inherit, so pair it with an abstract base for behaviour.

### Which shapes to extract, and when

Extract an abstract **only when the second consumer actually exists**. A base pulled up for a hypothetical consumer that then diverges is worse than two honest copies.

| Candidate | Extract when | Confidence |
|---|---|---|
| `mixin.package.measure` | Phase 2 — both need dimensions/weights | High |
| `mixin.charge.line` | With the shared charge-code master | High |
| `mixin.party.line` | With the second consumer's party lines | High |
| `mixin.cargo.line` | When NVOCC packing lines are declared | Medium |
| `mixin.shipment.base` | If NVOCC declares its own shipment concrete | Medium |
| `mixin.booking.base` | Defer — pending the booking-tier call (§3) | Low |

---

## 7. Execution plan

### Phase 0 — Governance (no code)

- [ ] Amend CLAUDE.md in one change: the domain-tier rule (§2), the prefix table (§4), the Model Placement Test rewritten around tiers.
- [ ] Confirm the four tier calls that are business decisions: booking, shipments, consolidation, documentation (§3).
- [ ] Approve the keyed-abstract kernel fix (§6) — blocks Phase 2.
- [ ] Confirm the four SOW deliverables that are not NVOCC work — tax/TDS, finance mapping, regional framework, country packs — are sequenced ahead or descoped under the SOW's change-management clause.

### Phase 1 — Stand up the three domains

- [ ] Create `src/domains/logistics_base/`, `src/domains/logistics_freight_forwarding/`, `src/domains/logistics_nvocc_agency/`, each with `__init__.py` + `settings.py`.
- [ ] Move the common modules into `logistics_base` per §3; move `shipments` / `consolidation` into `logistics_freight_forwarding`.
- [ ] **Leave every model key unchanged.**
- [ ] Re-header the affected migrations to their new app keys; update every manifest `depends` to the new fully-qualified keys.
- [ ] `ACTIVE_DOMAINS = ["logistics_base", "logistics_freight_forwarding"]` — base first.
- [ ] **Land `validate_domain_tiers`** (§2) plus `BASE_DOMAINS` in domain settings.
- **Verify:** full suite · boot · `migrate upgrade` on a scratch tenant · a generated migration for every moved module is **empty** · the tier guard rejects a deliberately-planted peer dependency.

### Phase 2 — Kernel fix + shared shapes

- [ ] Registry abstract map, loader routing, boot lineage validation, derived `__ede_inherits__` — with tests.
- [ ] Extract the first mixins into `logistics_base` with `mixin.*` keys; refactor existing models to inherit them, keeping field sets identical.
- **Verify:** a generated migration is **empty**. A non-empty diff means a field changed during pull-up.

### Phase 3 — Build NVOCC

- [ ] Add `logistics_nvocc_agency` to `ACTIVE_DOMAINS`; scaffold `agency` per the SOW.
- [ ] SOW Phase 2 (D2.1–D2.8) is built in `logistics_base.equipment_control` + `logistics_base.masters`, **not** in NVOCC. NVOCC contributes only the principal link via `@api.extend_model("logistics.equipment")`.
- [ ] The principal scope dimension — still the one other approval-gated platform change.
- **Verify:** full suite · three activation combinations each boot clean (base+ff, base+nvocc, all three) · demo data smoke-tested per module · cross-principal access tests.

---

## 8. Do

| # | Rule | Why |
|---|---|---|
| 1 | List `logistics_base` **first** in `ACTIVE_DOMAINS` | Dependency validation requires a dep to be already loaded; domain load order is settings order. |
| 2 | Name it `logistics_base`, not `logistics_foundation` | "Foundation" already names the platform layer; the second name reads as if it lived there. |
| 3 | Express the rule as **tiers** and **enforce it at boot** | The loader accepts peer coupling silently. `validate_domain_tiers` turns the rule into something a mistake cannot pass. |
| 4 | **Keep model keys unchanged** when modules move | Preserves 284 references, 32 seed files (CSV names encode keys), every view, every blueprint, the trade-location sync. |
| 5 | Default an ambiguous module to the **base** domain | Over-inclusion is harmless; a product-domain module the other product needs leaves you blocked or tempted into peer coupling. |
| 6 | Move `masters` **wholesale** | All 42 are needed by both. Push a genuinely FF-only master down later. |
| 7 | Put `equipment_control` in the base domain as a **shared concrete** | One container registry — globally unique numbers, one register report. Each product adds its own fields via `extend_model`. |
| 8 | Use **mixins for shapes**, shared concretes for **registries** | A mixin copies columns per table; two tables means identity isn't unique and no query spans both. |
| 9 | Fix the three keyed-abstract kernel gaps before any mixin ships | The path is declarable but crashes at boot, and has zero test coverage. |
| 10 | **Derive** the keyed lineage from the MRO | Registry-queryable lineage, full method inheritance, drift impossible. |
| 11 | Verify Phases 1 and 2 by asserting the generated migration is **empty** | The only mechanical proof that a move or a pull-up changed no schema. |
| 12 | Test **three activation combinations** every phase | A tiered split is only real if each product domain boots on the base alone. |
| 13 | Build the shared **charge-code master** in `logistics_base` | It exists nowhere today, and four modules need it. |
| 14 | Extract a mixin only when the **second consumer exists** | A wrong shared base is harder to remove than a duplicate is to merge. |

## 9. Don't

| Don't | Why it breaks | Instead |
|---|---|---|
| Let `logistics_freight_forwarding` and `logistics_nvocc_agency` reference each other | The one edge the tier rule exists to forbid — and the loader accepts it silently. | Lift the shared concept into `logistics_base`. |
| Let `logistics_base` depend on a product domain | Inverts the tier and creates a cycle in everything but name. | Base depends on `foundation.*` only. |
| Ship the tier rule as documentation only | Nothing enforces it; the first mistake lands in `main` green. | `validate_domain_tiers` at boot, Phase 1. |
| Put `logistics_base` after a product domain in `ACTIVE_DOMAINS` | Dependency validation fails — the dep isn't loaded yet. | Base first. |
| Rename model keys because a module changed domain | Keys and app keys are independent. Renaming churns 284 references and 32 seed files for zero gain. | Keys stay; new models take the new tier prefix. |
| Rename existing FF keys (`logistics.shipment` → `logistics_freight_forwarding.shipment`) | Table rename + FK repointing across ~10 children + invalidation of every stored reference naming the old key. | Grandfather them. New FF models use `logistics_ff.*`. |
| Fork a second container registry in NVOCC | Container numbers stop being globally unique and no report spans both businesses. | One shared concrete in `logistics_base.equipment_control`. |
| Give a mixin a table or a model-level migration | It stops being abstract. | `abstract=True`, no table, no migration. |
| Point a FK at an abstract | Cannot resolve — abstracts never enter the registered-model map. | A shared concrete parent + `delegated=`, or keep the child in the domain. |
| `@api.extend_model` a mixin | Its validator resolves concrete targets only. | Edit the mixin (you own the base), or extend each concrete. |
| Overload `extend_model` to mint a new model | Extend-in-place and declare-new-model have opposite migration consequences; the call site becomes unreadable. | `extend_model` extends; `model` declares. |
| Make `inherit="mixin.key"` the inheritance mechanism | Fields would cross but not methods, commands, hooks or `super()` — and behaviour was the point. | Python inheritance; derive the lineage. Optional `inherit=` as a validated assertion. |
| Ship a keyed abstract before the kernel fix | `InvalidHandler: Abstract model cannot be registered` at boot. | Land the registry + loader + validation fixes with tests. |
| Cherry-pick which masters move | Two masters modules and a new seam where it is hardest to change. | Move wholesale. |
| Ship Phase 1 with a non-empty generated migration | The move silently changed schema. | Fix the model, regenerate, re-verify. |

---

## 10. Open decisions

| # | Question | Owner |
|---|---|---|
| 1 | CLAUDE.md amendment — approve the tier rule and prefix wording (§2, §4). Blocks Phase 1. | Eng Lead |
| 2 | Tier calls for **booking**, **shipments**, **consolidation**, **documentation** (§3). Business semantics, not architecture. | Product Owner |
| 3 | Keyed-abstract kernel fix — approve the three additive changes (§6). Blocks Phase 2. | Eng Lead |
| 4 | Accept `inherit=` as an optional validated assertion, or rely purely on derived `__ede_inherits__`? | Eng Lead |
| 5 | Principal scope dimension — generalise the scope mechanism, or accept the record-rule fallback? | Eng Lead |
| 6 | Tax/TDS, finance mapping, regional framework, country packs — sequence platform localization ahead, or descope those four SOW deliverables? | Product Owner |
| 7 | Finance-mapping layer, against the standing decision that this platform holds no accounting domain. | Eng Lead + PO |

---

## Related

- [Model & View Extension SDK](foundation-base-extensions.md) — extension mechanics; cannot target abstracts.
- [Module Integration Pattern](module-integration-pattern.md) — producer-owns-the-contract; applies **within** each domain.
- [Company Scope](foundation-company-scope.md) · [Active Organization](platform-08-active-organization-and-company-scope.md) — the dimension principal scoping must mirror.
- [`foundation.l10n` roadmap](../roadmap/foundation/l10n/README.md) — prerequisite for the regional deliverables.
- [CLAUDE.md](../CLAUDE.md) — placement test, naming rules, platform-change approval gate.
