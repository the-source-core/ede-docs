# `foundation.ai_insights` — Demo Use-Case

**Module:** `ede.foundation.ai_insights`
**App key:** `foundation.ai_insights`
**Demo manifest entries:** `demo/demo_ai_insights.xml`
**Status:** ✅ Authored (Phase 2) — demo decision profile ships with the `<decision/>` cockpit surface.

---

## Use-case

`foundation.ai_insights` turns any record into a **deterministic, explainable decision**. The headline UI feature is the form-view **`<decision/>` cockpit card** — so the demo must ship at least one **decision profile** an admin can immediately see in action:

1. **Decision Profiles admin** — `Settings → AI Insights → Decision Profiles` shows the seeded **"Consolidation Load Risk"** base profile over `logistics.shipment`. An admin opens it, sees the three factors (weight · package count · SI/VGM dependency) with weights, severity bands, and the recommended actions, and can edit them on screen (no redeploy) or **Clone as New Version** to roll the config forward.
2. **`<decision/>` cockpit** — when a `<decision/>` element is placed on the shipment form, the card evaluates the open shipment and renders the score ring · severity chip · ranked factors · confirm-to-act buttons. (The demo ships the profile; placing the element on the shipment form is a consumer-module choice.)
3. **Decision Log** — `Settings → AI Insights → Decision Log` shows the immutable audit ledger: every evaluation with score, severity, ranked factors, inputs, and config version.

The decision is **deterministic** — the optional AI-narration layer (Phase 4) only explains it; it never changes the score.

## Records produced

### `demo/demo_ai_insights.xml`

| External ID | Model | Notes |
|---|---|---|
| `ai_insights.demo_decision_shipment_load` | `ir.insight.decision` | Base profile (no org), `model_key=logistics.shipment`, `score_mode=weighted_sum`, bands 0.30 / 0.70. |
| `ai_insights.demo_factor_weight` | `ir.insight.decision.factor` | `field` → `consolidation_total_weight`, weight 2.0, normalize 0–30000, higher-is-riskier. |
| `ai_insights.demo_factor_packages` | `ir.insight.decision.factor` | `field` → `consolidation_total_packages`, weight 1.0, normalize 0–500. |
| `ai_insights.demo_factor_vgm` | `ir.insight.decision.factor` | `field` → `si_vgm_dependency` (boolean), weight 1.0. |
| `ai_insights.demo_action_review` | `ir.insight.decision.action` | HIGH → event `logistics.shipment.load_review_requested`, requires confirm. |
| `ai_insights.demo_action_audit` | `ir.insight.decision.action` | ANY → event `logistics.shipment.load_risk_noted`, no confirm. |

All factors point at **real `logistics.shipment` fields** so `insights.evaluate` runs end-to-end. Actions use the **event** kind (no handler required) so the demo is inert until a consumer subscribes — safe to ship in any tenant.

## Smoke test

```bash
ede migrate upgrade -t <tenant> --with-demo=foundation.ai_insights --config <conf>
```

Verified 2026-07-01 on a fresh PostgreSQL tenant: **6 demo rows** created (1 profile · 3 factors · 2 actions), idempotent on re-run; views + menus load; RelaxNG validation clean.
