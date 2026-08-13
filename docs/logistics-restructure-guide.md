# Logistics Vertical Restructure — Developer Guide & Change Log

> **Audience:** anyone with in-flight work branched off the *old* single-`logistics`-domain
> structure, and anyone writing new logistics code during or after the restructure.
> **Branch:** `main-multi_domain_restructuring-dharmang`
> **Companions:** [approach](nvocc-domain-enablement.md) · work streams under
> `roadmap/logistics/report/work-streams/`

**Read §2 before rebasing.** It is the whole "what do I have to change?" answer, and for most
branches the answer is *very little* — model keys do not change, so the vast majority of code is
untouched.

---

## 1. What is happening, in one paragraph

The single `logistics` domain is being split into a **tiered** vertical: one **base** domain holding
what every logistics product needs, and **product** domains for each business line. Products may
depend on the base; they may **never** depend on each other. This is what lets a customer buy
freight forwarding, NVOCC, or both, and lets each ship on its own cadence.

```
                        logistics_base            ← shared tier
                       ▲              ▲
        logistics_freight_forwarding   logistics_nvocc_agency     ← product tier
                       ╳  never depend on each other  ╳
```

---

## 2. What you must change in in-flight work

### 2.1 Almost certainly nothing: model keys are unchanged

**Model keys and app keys are independent.** The key is declared in the decorator; the app key is
derived from folder position. Moving a module changes the app key and **nothing** about model
identity.

So all of this keeps working, untouched:

| Still valid | Why |
|---|---|
| `@api.model("logistics.equipment")` and every other `logistics.*` key | Keys were deliberately not renamed |
| `env.models["logistics.facility.master"]` | Same |
| `fields.Reference("logistics.unlocode.master")` | Same |
| Domain strings, command names, hook keys `pre.logistics.*.create` | Same |
| XML ids — `masters.some_record`, `<extend ref="masters.view_id">` | XML id module is the **short app name**, which does not change on a domain move |
| Data CSV filenames (`logistics.equipment.type.master.csv`) | Filenames encode the *model* key |
| Alembic revision ids, `down_revision` chains | Revision identity is content, not path |

That last group is why this restructure is affordable: **284 references across 72 files, plus 32
seed files, needed no edit at all.**

### 2.2 What does change

Three mechanical things, all keyed to the module's **new home**:

| # | Surface | Old | New |
|---|---|---|---|
| 1 | Filesystem path | `src/domains/logistics/<module>/` | `src/domains/<new_domain>/<module>/` |
| 2 | Manifest `depends` | `"logistics.<module>"` | `"<new_domain>.<module>"` |
| 3 | Python import | `from domains.logistics.<module>…` | `from domains.<new_domain>.<module>…` |
| 4 | Path literals in tests | `"src/domains/logistics/<module>/…"` | `"src/domains/<new_domain>/<module>/…"` |

`<new_domain>` is `logistics_base` for the 8 shared modules and
`logistics_freight_forwarding` for the 3 product ones — see the table in §4.

Plus, inside a moved module only: the `EDE-App-Key:` migration header (metadata; no schema
operation is ever touched).

### 2.3 Rebase recipe

After rebasing onto the restructure branch, run this from the repo root. It is safe to run more
than once, and it only rewrites the three surfaces above:

```bash
BASE='masters\|equipment_operations\|pricing\|sales_crm\|booking\|shipments\|operations_execution\|profitability'
PROD='consolidation\|documentation\|tracking'

# 1. manifest depends
grep -rl "\"logistics\.\($BASE\)\"" src/ --include=__manifest__.py \
  | xargs -r sed -i "s/\"logistics\.\($BASE\)\"/\"logistics_base.\1\"/g"
grep -rl "\"logistics\.\($PROD\)\"" src/ --include=__manifest__.py \
  | xargs -r sed -i "s/\"logistics\.\($PROD\)\"/\"logistics_freight_forwarding.\1\"/g"

# 2. python imports
grep -rl "domains\.logistics\.\($BASE\)" src/ --include=*.py \
  | xargs -r sed -i "s/domains\.logistics\.\($BASE\)/domains.logistics_base.\1/g"
grep -rl "domains\.logistics\.\($PROD\)" src/ --include=*.py \
  | xargs -r sed -i "s/domains\.logistics\.\($PROD\)/domains.logistics_freight_forwarding.\1/g"

# 3. hardcoded path literals in tests
grep -rl "src/domains/logistics/\($BASE\)" src/ --include=*.py \
  | xargs -r sed -i "s#src/domains/logistics/\($BASE\)#src/domains/logistics_base/\1#g"
grep -rl "src/domains/logistics/\($PROD\)" src/ --include=*.py \
  | xargs -r sed -i "s#src/domains/logistics/\($PROD\)#src/domains/logistics_freight_forwarding/\1#g"

# 4. verify — nothing may remain under the retired domain
grep -rn 'domains\.logistics\.\|src/domains/logistics/' src/ --include=*.py \
  | grep -v logistics_base | grep -v logistics_freight_forwarding
```

Watch for **segmented** path constructions the slash-form sed cannot see — these bit us twice:

```python
Path(__file__).resolve().parents[3] / "domains" / "logistics" / "pricing" / ...
```

Grep them separately: `grep -rn '"domains" */ *"logistics"' src/ --include=*.py`.

Then: `./run_tests.sh`. A boot failure naming a dependency that "is not loaded yet" means a manifest
still points at the old app key.

### 2.4 New code written during the restructure

- **Put new shared logistics capability in `logistics_base`.** When unsure, choose the base:
  over-inclusion is harmless, whereas a product-domain module the *other* product later needs
  leaves you blocked — or tempted into peer coupling, which now fails at boot.
- **Never add a dependency between two product domains.** `validate_domain_tiers` rejects it at
  boot (WS1). If two products need the same thing, it belongs in the base.
- **Keep using `logistics.*` model keys** for anything that lives in the base tier. Product-specific
  *new* models take the product prefix (`logistics_ff.*`, `nvocc.*`). Existing keys are
  grandfathered — do not mass-rename.

---

## 3. Production impact — none, and why

Audited before any module moved. An existing tenant upgrades on the normal path with **no data
migration**:

| Surface | Behaviour |
|---|---|
| Tables / schema | Unchanged — model keys unchanged means table names unchanged |
| Alembic history | Revision ids are content identity; `version_locations` is rebuilt from the registry at runtime, and nothing reads the `EDE-App-Key` header. Moving a migration file between app folders is transparent |
| `ir.model.app_key` | Stores the **short** app name (`masters`), not the full key. Upserted by external id `(short_app_name, model_key)`, which is stable → **updated in place** by `RegistrySync` during `ede migrate upgrade` |
| `ir.data.reference` (XML ids) | `module` = short app name → unchanged → menus, actions, views, RBAC rows all keep resolving |

**The one thing that would break it: renaming a module.** That changes the short app name, orphaning
every `<old_module>.*` XML id — and orphan cleanup deletes orphaned records that have no inbound
FKs. That is data loss, not untidiness. For this reason `equipment_operations` **keeps its name**;
the previously-floated rename to `equipment_control` was dropped.

> **Rule for the rest of this restructure, and after it:** moving a module between domains is free.
> *Renaming* a module is a data migration. Do not rename without one.

---

## 4. Change log

Status: ✅ landed · 🟡 in progress · ⬜ not started

| WS | Bucket | Change | Status |
|---|---|---|---|
| 1 | Platform | Domain tier guard | ✅ `162099f8` |
| 2 | Platform | Keyed abstract registration | ✅ `a3a986db` |
| 3 | Migration | `logistics_base` created; `masters` moved | ✅ `56009ccd` |
| 4 | Migration | 5 core modules moved | ✅ `e823c511` |
| 5 | Migration | 2 engine modules moved | ✅ `8b46725e` |
| 6 | Rename | `logistics` → `logistics_freight_forwarding` | ✅ `5a80dc89` |
| 7 | Migration | Shared shapes — `mixin.charge.line`, `mixin.party.line` | ✅ |
| 8 | NVOCC | `logistics_nvocc_agency` domain + `agency` module | ✅ Features 01–05; 06 🟡 |
| 9 | Migration (base) | Container control — BRD N1 / SOW D2.1–D2.8 | 🟡 |

**Streams 1–7 have landed, and WS8 + WS9 are built.** The vertical is now tiered,
the WS1 guard is enforcing it against the real structure, and the second product
domain exists and boots on the base alone.

### Moved modules — old app key → new app key

| Module | Old | New | WS |
|---|---|---|---|
| `masters` | `logistics.masters` | `logistics_base.masters` | 3 |
| `equipment_operations` | `logistics.equipment_operations` | `logistics_base.equipment_operations` | 4 |
| `pricing` | `logistics.pricing` | `logistics_base.pricing` | 4 |
| `sales_crm` | `logistics.sales_crm` | `logistics_base.sales_crm` | 4 |
| `booking` | `logistics.booking` | `logistics_base.booking` | 4 |
| `shipments` | `logistics.shipments` | `logistics_base.shipments` | 4 |
| `operations_execution` | `logistics.operations_execution` | `logistics_base.operations_execution` | 5 |
| `profitability` | `logistics.profitability` | `logistics_base.profitability` | 5 |
| `consolidation` | `logistics.consolidation` | `logistics_freight_forwarding.consolidation` | 6 |
| `documentation` | `logistics.documentation` | `logistics_freight_forwarding.documentation` | 6 |
| `tracking` | `logistics.tracking` | `logistics_freight_forwarding.tracking` | 6 |

**Every module moved.** The `logistics` domain no longer exists — a boot
assertion in `test_loader_depends.py` now fails if any app reappears under it.

---

### WS1 — Domain tier guard ✅

**What.** `validate_domain_tiers(registry)` runs at boot beside `validate_extensions()` /
`validate_delegated_declarations()`. Tiers are declared by `BASE_DOMAINS` in
`src/domains/settings.py`.

**Why.** `ModuleLoader._validate_depends` only checks that a dependency is already *loaded* — it
does not care which domain it came from. Probed against the live loader: **a peer-domain dependency
is ACCEPTED**. Without the guard the tier rule would be a comment, and the first accidental coupling
would land green in CI.

**Affects you if:** you add a manifest `depends` entry across domains. The legal cross-domain edge
is product → base. Everything else raises at boot, naming the app, the dependency, and the rule.

**Files:** `src/ede/core/domain_tiers.py` (new) · `src/ede/core/bootstrap.py` ·
`src/domains/settings.py` · `src/tests/core/test_domain_tiers.py` (13 tests).

---

### WS2 — Keyed abstract registration ✅

**What.** `@api.model("mixin.x", abstract=True)` now works. `Registry` gained an abstract map
(`register_abstract` / `get_abstract` / `list_abstracts` / `list_models_inheriting`); the loader
routes abstract classes there instead of to `register_model`; `__ede_inherits__` is derived from the
MRO; `validate_abstract_lineage()` runs at boot.

**Why.** The parameter already existed and was fatal — the decorator set a model key and returned
early without registering, while the loader gated only on "has a key" and handed the class to
`register_model`, which rejects abstracts. It had **zero usages**, so nothing had exercised it.

**Affects you if:** you want to share a record *shape* across modules. Declare a keyed abstract in
the owning module and subclass it — fields, constraints **and** behaviour inherit normally.

Two things to know:

- **Lineage is derived, not declared.** There is deliberately no `inherit="mixin.x"` parameter: a
  key-only edge would carry fields but not methods, commands, hooks or `super()`, and shared
  behaviour is the point. Subclass in Python; the registry reads the lineage off the MRO.
- **`@api.extend_model` cannot target an abstract** — its validator resolves concrete targets only.
  A field every consumer needs goes *into* the mixin; a field for one consumer goes on that
  consumer's concrete.

**Files:** `src/ede/core/registry.py` · `src/ede/core/loader.py` · `src/ede/core/bootstrap.py` ·
`src/tests/core/test_keyed_abstracts.py` (16 tests).

---

### WS3 — `logistics_base` created, `masters` moved 🟡

**What.** New base domain `src/domains/logistics_base/` with `masters` (42 models, 12 views, 44 data
files, 13 migrations) relocated into it wholesale. `ACTIVE_DOMAINS` becomes
`["logistics_base", "logistics"]` — **base first**, because domain load order is settings order and
a dependency must already be loaded when declared. `BASE_DOMAINS = ["logistics_base"]` activates the
WS1 guard for real.

**Why masters, and why all of it.** All 42 masters were audited against both businesses and every
one is needed by both. Cherry-picking would create a second masters module and a seam in the place
hardest to change later.

**Affects you if:** your branch touches `src/domains/logistics/masters/**`, imports from it, or
declares `depends: ["logistics.masters"]`. Apply the §2.3 recipe.

**Not affected:** every `logistics.*` model key, every XML id (`masters.*`), every view
`<extend ref="masters.…">`, every data CSV, and the trade-location sync — all resolve by names that
did not change.

**Files:** `git mv src/domains/logistics/masters src/domains/logistics_base/masters` · 10 sibling
manifests repointed · 3 Python imports · 6 migration headers re-stamped ·
`src/domains/logistics_base/{__init__,settings}.py` (new) · `src/domains/{settings,logistics/settings}.py`.

**Verification:** full suite green (5868 pytest, 673 vitest). One stale assertion in
`test_loader_depends.py` hardcoded `logistics.masters`; updated to assert the new tier **and** that
the old key is gone.

---

### WS4 — 5 core modules moved ✅ `e823c511`

**What.** `equipment_operations`, `pricing`, `sales_crm`, `booking`, `shipments` → `logistics_base`,
in dependency order (`shipments` needs `booking` → `sales_crm` → `pricing` → `masters`, and
`equipment_operations`).

**Why.** These carry the shared commercial and operational spine — an NVOCC operator books capacity,
prices a lane and controls containers exactly as a forwarder does. Leaving any one behind would
block the rest, because the dependency closure is transitive.

**`equipment_operations` keeps its name.** The rename to `equipment_control` floated in the design
docs was **dropped** — see §3 for why it would have been destructive in production.

**Affects you if:** your branch touches any of the five. Apply the §2.3 recipe.

**Verification:** full suite green. Fallout was entirely stale path literals: 26 hardcoded
`src/domains/logistics/<module>/…` strings, plus two *segmented* `Path(...) / "domains" /
"logistics" / …` constructions the slash-form sed could not see.

---

### WS5 — 2 engine modules moved ✅ `8b46725e`

**What.** `operations_execution` and `profitability` → `logistics_base`. The base tier now owns 8
modules.

**Why.** Both are reverse-leaves — nothing depends on them — so they moved independently and last.
Job execution (milestones, tasks, cut-offs) and per-job margin are business-neutral; duplicating
either engine per product is exactly what the restructure exists to avoid.

**Affects you if:** your branch touches either. Apply the §2.3 recipe.

---

### WS6 — `logistics` → `logistics_freight_forwarding` ✅ `5a80dc89`

**What.** The remaining product domain is renamed. `ACTIVE_DOMAINS` is now
`["logistics_base", "logistics_freight_forwarding"]`.

**Why last.** It carries no capability — its value is that the product boundary is self-evident on
disk. Sequencing it last means a problem here reverts without giving up WS3–WS5.

**Wide but shallow:** no schema change, no data migration, no model-key changes. Only the
`domain_type` segment of three app keys moves, plus import paths and path literals. Views and XML
data ids are scoped by app *name*, which does not change.

**Affects you if:** your branch touches `consolidation`, `documentation` or `tracking`.

---

## 5. Final state

```
src/domains/
├── logistics_base/                    # BASE tier — 8 modules
│   ├── masters/  equipment_operations/  pricing/  sales_crm/
│   └── booking/  shipments/  operations_execution/  profitability/
├── logistics_freight_forwarding/      # PRODUCT tier — 3 modules
│   └── consolidation/  documentation/  tracking/
└── logistics_nvocc_agency/            # PRODUCT tier — 5 modules
    ├── agency_masters/                # nvocc.principal + entitlement + refusal log
    ├── agency_network/                # nvocc.principal.agent + res.partner ext
    ├── agency_numbering/              # nvocc.numbering.rule + logistics.charge ext
    ├── agency_integration/            # nvocc.integration.partner / .message.type
    └── agency_reports/                # registers + KPIs (data-only, no models)
```

```python
# src/domains/settings.py
ACTIVE_DOMAINS = ["logistics_base", "logistics_freight_forwarding", "logistics_nvocc_agency"]
BASE_DOMAINS   = ["logistics_base"]                                   # base FIRST
```

### 5.1 Module folder names are globally unique — this is load-bearing

`ir.data.reference.module` stores the **short** app name (the folder), not the full app key. Two
modules named `masters` in different domains would collide in that table. This is why the NVOCC
modules carry the `agency_` prefix instead of the shorter `masters` / `reports` that would otherwise
read better — `logistics_nvocc_agency.masters` would fight `logistics_base.masters`.

### 5.2 Why there is no `agency_access` module

The obvious seam — split entitlement + `principal_context` away from the principal master — does
**not** hold. `nvocc.principal` calls `principal_context` for its `switch` / `context` commands,
while `nvocc.principal.entitlement` carries an FK back to the principal: a cycle. It cannot be
broken by contributing the two commands from the other side, because `@api.extend_model` merges
**fields only** (`Registry._merge_extension_into_base` returns early without
`__ede_extension_fields__`). Separating them needs a framework change, so they ship as one module
and the coupling is documented in `agency_masters/__init__.py` rather than hidden behind a boundary
that does not hold.

### 5.3 Splitting a module IS the rename case — measured, not assumed

The domain rename `logistics_nvocc` → `logistics_nvocc_agency` was free (the short app name never
appears in it). The **module split was not**, and a dry run against an already-seeded dev tenant
showed exactly why:

| Symptom | Count on `devqa` |
|---|---|
| Refs stranded under the retired `agency` module | 403 |
| Refs created under the five new module names | 83 |
| `ir_model` rows still stamped `app_key='agency'` | 7 |

Two distinct failure modes follow, and both are generic to any module rename:

1. **`RegistrySync` does not re-stamp `app_key` when a model changes module.** The seven `ir.model`
   rows kept `agency`, so their external ids stayed `agency.model_nvocc_principal`. Every
   `agency_reports` blueprint then failed with
   `Cannot resolve ref='agency_masters.model_nvocc_principal'`.
2. **The 403 stranded refs are orphan-cleanup candidates.** Cleanup deletes an orphaned record with
   no inbound FK — the data-loss path this guide has warned about since §3.

On `devqa` the cleanup did not actually run, because the *pre-existing* `documentation.*` dangling-ref
bug aborts the data load first (`orphan_cleanup: skipped (data load reported errors)`). A broken
tenant was accidentally shielded by another break; do not rely on that.

**Consequence:** a tenant that has already seeded `agency.*` needs a re-seed or a written data
migration. This split was done pre-production precisely so the answer could be "re-seed the dev
tenants". The window closes on the first customer seed.

The migration files themselves needed nothing: the split is schema-neutral (same tables, columns and
FKs), so the two nvocc revisions simply moved into `agency_masters/migrations/versions/` as **pure
renames**, byte-identical. Revision count before and after: 236 → 236.

### 5.4 ⚠ TWO ITEMS MUST BE CLOSED BEFORE THIS REACHES ANY SEEDED TENANT

Both were found by the pre-commit review. Neither affects a **fresh** tenant, and neither is a defect
in the committed code — they are deployment blockers. Do not merge to a branch that any real tenant
tracks until they are resolved.

**(a) A plain `migrate upgrade` on a seeded tenant DELETES the agency RBAC rows — a lockout.**
The chain is exact, and the dangerous part is that nothing fails loudly:

1. Roles and permissions kept their natural keys (`agency_viewer`, `nvocc.principal.read`) while
   their external ids moved from `agency.*` to `agency_masters.*` / `agency_network.*` / …
2. The loader hits the `unique=True` on `ir.rbac.role.code` / `ir.rbac.permission.code`, calls
   `_recover_existing_record`, and binds the **new** external id to the **existing** `record_uuid`.
   The load reports success.
3. Orphan cleanup computes orphans by `(module, name)`, not by `record_id`. The stale `agency.p_*`
   refs are therefore orphans pointing at the records the new ids just adopted. `ir.rbac.permission`
   is not in `_PROTECTED_MODELS` and nothing holds an inbound FK to a permission row → **all 24
   agency permission rows are deleted.**

`AuthorizationService` is fail-closed, so the result is a lockout, not an escalation — every agency
user except the `system_admin` role code is denied every agency action. Two acceptable fixes: ship a
data migration that re-stamps `ir.data.reference.module` from `agency` to the new owning module
*before* the first post-split upgrade (this makes the split lossless), or state in the runbook that
affected tenants are **rebuilt, not upgraded** — and note that runtime-created role bindings are not
restored by a data-file re-seed.

**(b) Migration ownership does not match module ownership.**
`8fbf2d8f0211` now sits in `agency_masters` and creates *every* sibling's tables
(`nvocc_principal_agent`, `nvocc_numbering_rule`, `nvocc_integration_partner`, …) plus the
`res_partner` and `logistics_charge` extension columns that `agency_network` / `agency_numbering`
declare. Version locations are collected per **loaded** app, so an agency-masters-only tenant gets
four foreign tables and 15 foreign columns — which falsifies the activation-independence claim in
`agency_integration`'s own manifest, and breaks CLAUDE.md's rule that an extension column ships from
the module that declares it.

Keeping the revisions as pure renames was correct for *this* commit (splitting them by hand is
forbidden). The real fix is to regenerate: scope `8fbf2d8f0211` to `agency_masters`' own tables plus
`logistics_equipment.principal_id`, then
`ede migrate generate --app logistics_nvocc_agency.agency_network` (and `…_numbering`,
`…_integration`) against the reference DB. That invalidates existing alembic state, which is exactly
why it pairs with the re-seed in (a) — and why doing it pre-production is the cheap moment.

**Guardrails now enforced at boot:**

| Guard | Rejects |
|---|---|
| `validate_domain_tiers` | a base domain depending on a product domain; two product domains depending on each other |
| `validate_abstract_lineage` | a concrete inheriting a `mixin.*` key whose module never loaded |
| `test_loader_depends.py` | any app reappearing under the retired `logistics` domain |

**Verification across all six streams:** 5868 pytest + 673 vitest green.

---

### WS7 — Shared shapes (mixins) ✅

**What.** Two keyed abstracts in `logistics_base/masters/models/abstracts/`, extracted *out of*
models that already shipped:

| Mixin | Consumers | Fields |
|---|---|---|
| `mixin.charge.line` | `logistics.booking.charge`, `logistics.shipment.charge` | 10 — quantity, buy/sell unit rate, buy/sell amount, margin, currency, sequence, active, notes — plus `computed_margin()` / `margin_is_consistent()` |
| `mixin.party.line` | `logistics.booking.party`, `logistics.shipment.party` | 5 — partner, role, address, active, notes |

**Scoped by measurement, not by the plan.** The design doc named three mixins; the code said
otherwise:

- **`mixin.package.measure` was NOT built.** Only two models carry measurement fields
  (`crm.quote.package`, `logistics.shipment.cargo.line`) and they overlap on 3 fields, of which
  only `gross_weight` truly aligns — one has `volumetric_weight`, the other `net_weight`. That is a
  forced abstraction, so the plan's own rule applies: extract only when a real second consumer exists.
- **Three charge fields were deliberately left on the concretes.** `charge_code`,
  `charge_description` and `charge_category` share a *name* across booking and shipment but not a
  *definition* — `Char(64)` vs `Char(32)`, `Char(500)` vs `Char(256)`, and 9-value-required vs
  8-value-optional enums with different members. Pulling them up would have picked one definition
  and silently re-shaped the other's **live column**.
- **`logistics.document.charge` does not inherit.** It shares 2 fields with the others; it is a
  document display line (`amount` / `description` / `prepaid_or_collect`), not a commercial line.
- **`crm.handover.party` does not inherit.** Its `notes` is `Char(max_length=500, multi_line=True)`
  against an unbounded `Char(multi_line=True)` on both consumers, and it has no `active`.

**Affects you if:** you add a field to a booking or shipment charge/party line. Ask first whether it
belongs on the shared shape (every consumer needs it → into the mixin) or on the one concrete
(product- or record-specific → on that class). Remember `@api.extend_model` cannot target an
abstract.

**No schema change.** Guarded by `src/tests/domains/test_shared_shapes_no_schema_drift.py`, which
pins each concrete's **pre-refactor** field set and asserts the divergent definitions still differ —
so a future "tidy-up" that normalises `Char(64)`/`Char(32)` fails the build instead of the database.

**Verification:** full suite green (5886 pytest, 673 vitest); 18 new tests.

---

### Follow-up — `logistics.charge` shipped ✅

The restructure analysis surfaced a **pre-existing** gap: there was no charge-code master
anywhere, only transactional charge lines with a free-text `charge_code`. It has now been built in
`logistics_base.masters` as **`logistics.charge`** — 22 seeded standard freight codes, list/form/
search views, an action and a Commerce menu entry.

Two design points worth knowing if you touch it:

- **`category` is the superset of both consumers' vocabularies.** `logistics.booking.charge` has
  `transit` + `surcharge`; `logistics.shipment.charge` has `handling`. The master carries all of
  them so a code maps from either side without loss — pinned by a test asserting each consumer's
  selection is a subset of the master's.
- **`basis` is not an amount.** It records *how* a charge is derived (per container, per kg,
  percentage…). The amount stays on the transactional line, which is a frozen per-deal value.

It is also the model a localization pack extends for country tax coding (HSN / SAC / CN) — extend
`logistics.charge`, don't add tax fields to charge lines.

> **Not yet applied to any tenant.** The migration (`1916c677587b`) is generated and committed but
> has not been run anywhere. Note the `devqa` tenant is already behind on migrations independently
> of this change.
### WS8 — `logistics_nvocc_agency` domain + `agency` module ✅ (Features 01–05)

**What.** The second **product** domain, built against BRD N1 (NVOCC / Shipping Agency Masters,
Admin & Control Foundation). Six `nvocc.*` models — `principal`, `principal.entitlement`,
`principal.agent`, `numbering.rule`, `integration.partner`, `integration.message.type` — plus three
contributions onto models this module does not own, all via `@api.extend_model`: `res.partner`
(NVOCC/agency customer + vendor attributes), `logistics.product.master` (agency billing flags),
and `logistics.equipment` (`principal_id`).

**The tier rule, proven mechanically.** `principal_id` references `nvocc.principal`, so it cannot be
declared in the base. Its migration is a **single `add_column` + `create_foreign_key` shipping from
`logistics_nvocc_agency.agency_masters`** — the one legal downward-pointing field, contributed the one legal way.
Freight-only activation gets container control with no NVOCC column and no NVOCC dependency; there
is a test for each of the three activation combinations.

**Affects you if:** you add a principal-controlled model. Tag it with `principal_id` and use
`agency/services/principal_context.py` — do not reinvent the entitlement lookup.

**⚠ Two naming traps this stream hit, both now guarded by tests:**

1. **`principal` is the authenticated user.** `Env.principal` already means "who is logged in".
   Carrier identity is `principal_id` / `active_principal_id` / `allowed_principal_ids`, never a bare
   `principal`. The collision would be silent and security-relevant.
2. **Delegation drains a child's columns.** `@api.model(..., delegated={"res.partner": "partner_id"})`
   is *parent-priority*: any field name that also exists on `res.partner` routes to the partner row
   and the child's own column is never written. Verified in a live tenant —
   `res_organization.code` and `.name` are NULL for exactly this reason. `nvocc.principal` therefore
   uses **`principal_code`** (not `code`) and **`is_active`** (not `active`), the latter matching the
   `res.organization` precedent. A `code` field would have left a unique constraint guarding an
   always-NULL column *and* overwritten the party's own code.

### WS9 — Container control 🟡 (base tier, not NVOCC)

**What.** The SOW files this under "NVOCC Phase 2"; every deliverable targets a base-domain model, so
it landed in `logistics_base` and **both** products get it. Shipped: 9 new columns on
`logistics.equipment` (check digit, ownership link, lease reference, reefer attributes, CSC validity);
`logistics.equipment.ownership.type.master` (7 seeded types) and `logistics.movement.event.code.map`
(versioned, partner-aware crosswalk) in `masters`; an ISO 6346 validator behind an `ir.config`
switch that **defaults off** so tenants with legacy container numbers upgrade safely; a
`blocks_release` flag on the status master plus a new `ON_HOLD` status enforcing BR-NAM-07/08; and
3 new facility types.

**Delivered 2026-08-04:** the ISO 6346 import validator (NAM-21) + a container-fleet import template, and the 4 container register datasets (NAM-27) — every blueprint compiled *and executed* against a live tenant.

**A design note worth carrying forward.** The obvious way to enforce "an unavailable container cannot
be allocated" was the existing `is_assignable_state` flag. That flag is also false for `IN_USE` and
`IN_TRANSIT`, so gating on it would have forbidden legitimate *forward* booking of a unit that is
merely busy today — concurrency is already handled by the window-overlap rule. The narrower
`blocks_release` flag exists because reusing the tidier-looking one would have quietly broken
forward booking.
