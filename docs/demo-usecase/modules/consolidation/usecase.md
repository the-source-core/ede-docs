# Consolidation — Demo Use Case

> Part of the unifying demo scenario in [docs/demo-usecase/README.md](../../README.md).
> Scope: **Phase 1 config-master slice** (features 04 `consolidation.policy` + 08
> `consolidation.type`). The transactional consolidation flow (console creation,
> pool search, manifests) is blocked on Shipments Phase 2 and is not demoed yet.

## What the demo shows

The freight forwarder's **Mumbai branch** (`base.default_organization`) runs
consolidations on the INMUM→SGSIN lane. Two reference layers are demonstrated:

1. **Standard seed (all tenants)** — the three tenant-wide grouping policies
   (`GROUP-BY-POD`, `GROUP-BY-CUSTOMER`, `GROUP-BY-CARRIER`) and the eight
   consolidation-type defaults (`consolidation.type`) that map each pattern
   (export / import-deconsole / LCL / air / road / rail / co-load / multimodal)
   to its mode, equipment family, manifest doc-type, and deconsolidation flag.

2. **Branch-scoped demo policies** — two `consolidation.policy` rows owned by the
   Mumbai branch:
   - `DEMO-SGSIN-LCL` — destination-grouping for Singapore-bound LCL groupage,
     with a tighter 36-hour cut-off and a 10% capacity buffer.
   - `DEMO-AIR-CONSOLE` — customer-grouping for air consoles with a 12-hour
     cut-off.

## Behaviour exercised

- **Scope resolution (BR-PM-01):** `consolidation.policy.resolve(env, group_by="destination",
  organization_id=<mumbai>, transport_mode_id=<sea>)` returns `DEMO-SGSIN-LCL`
  (org-specific) over the tenant-wide `GROUP-BY-POD` default — the org-scoped row
  wins on scope specificity / priority.
- **Policy-level overrides (BR-PM-02):** the demo rows carry their own
  `capacity_buffer_pct` / `cutoff_hours_before_departure`, illustrating how a
  policy overrides the org `ir.config` defaults for consoles created under it
  (enforcement lands with the console — feature 07).
- **Soft archive:** policies are archivable via `active`; archived rows stop
  matching but remain referenced by historical consoles.

## Records seeded

| Model | Standard seed | Demo (branch-scoped) |
|---|---|---|
| `consolidation.type` | 8 type defaults | — |
| `consolidation.policy` | 3 tenant-wide policies | 2 Mumbai-branch policies |

Smoke: `ede migrate upgrade -t <tenant> --with-demo=logistics.consolidation`.
