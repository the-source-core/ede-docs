# RMS Phase 1 Core — Design Spec

**Date:** 2026-05-07  
**Module:** `logistics.pricing` (sub-package of logistics domain)  
**Scope:** Phase 1 Core — models + margin engine + approval wiring + XML views + RBAC seeds  
**Deferred:** Excel upload, dedicated CRM rate-search endpoint, reports/dashboard, rate versioning (Phase 2)  
**Approach:** Pure EDE pattern — hooks + domain commands, no service layer

---

## 1. What We Are Building

The Rate Management System (RMS) Phase 1 Core gives a freight forwarding team the ability to:

- Create buy and sell rates with multi-modal charge lines
- Automatically calculate margin when charge lines are saved
- Submit rates for approval; auto-approve when margin is healthy, route to `ir.approval.case` when margin is below threshold
- Enforce buy-rate visibility and margin visibility via RBAC
- Manage rates through a full status lifecycle (draft → approved → suspended/expired)

This is the pricing foundation that Sales CRM will consume for quotation. No CRM-specific endpoint is built here — CRM will use standard `ede.search` on `pricing.rate` filtered to `status=approved`.

---

## 2. Module Structure

```
src/domains/logistics/pricing/
├── __manifest__.py
├── __init__.py
├── models/
│   ├── __init__.py
│   ├── rate.py              # pricing.rate + pricing.rate.line (EXISTS — no changes needed)
│   ├── margin_rule.py       # pricing.margin.rule (NEW)
│   └── rate_version.py      # pricing.rate.version stub (NEW)
├── controllers/
│   └── rate.py              # HTTP routes
├── views/
│   ├── pricing.rate.list.xml
│   └── pricing.rate.form.xml
├── data/
│   ├── pricing_roles.xml
│   └── ir.rbac.permission.csv
└── migrations/
    └── versions/            # single Alembic migration for all pricing tables
```

**Activation:** `src/domains/logistics/settings.py`
```python
ACTIVE_MODULES: list = ["masters", "pricing"]
```

---

## 3. Models

### 3.1 `pricing.rate` + `pricing.rate.line` — No Changes

Both models exist in `models/rate.py` (333 lines). They are complete per the roadmap field checklist. No edits required.

The existing model already declares:
- `margin_rule_id = fields.Reference("pricing.margin.rule", ...)` — resolves once `pricing.margin.rule` exists
- `version_ids = fields.One2Many("pricing.rate.version", "rate_id", ...)` — resolves once stub exists
- `calculated_margin_amount`, `calculated_margin_percent`, `margin_risk_level` — written by the margin engine

### 3.2 `pricing.margin.rule` (NEW — `models/margin_rule.py`)

Minimum margin configuration. Most specific rule (highest `priority`) wins when multiple rules match a rate.

| Field | Type | Notes |
|---|---|---|
| `code` | Char | Unique short code |
| `name` | Char | Display name |
| `rule_type` | Enum | `minimum` (blocks approval if breached) / `target` (warning only) |
| `margin_basis` | Enum | `percentage \| fixed_amount \| per_container \| per_kg \| per_cbm \| per_shipment` |
| `margin_value` | Decimal | Threshold — e.g. 5.0 for 5%, 500 for $500 fixed |
| `currency_id` | Reference → `res.currency` | For `fixed_amount` basis only |
| `customer_id` | Reference → `logistics.contact.member` | Optional applicability scope |
| `branch_id` | Reference → `ir.org.unit` | Optional applicability scope |
| `entity_id` | Reference → `ir.org.unit` | Optional applicability scope |
| `trade_lane_id` | Reference → `logistics.trade.lane` | Optional applicability scope |
| `mode_id` | Reference → `logistics.transport.mode.master` | Optional applicability scope |
| `product_id` | Reference → `logistics.product.master` | Optional applicability scope |
| `salesperson_id` | Reference → `res.user` | Optional applicability scope |
| `priority` | Integer | Higher = evaluated first. Default 10. |
| `active` | Boolean | Default True |

**Rule resolution logic** (used in margin engine and submit command):
- Query all active rules ordered by `priority DESC`
- Return first rule where every non-null applicability field matches the rate
- If no rule matches: fall back to a hardcoded default (5% minimum, `percentage` basis)

### 3.3 `pricing.rate.version` Stub (NEW — `models/rate_version.py`)

Minimal stub — satisfies the FK reference from `pricing.rate.version_ids`. No views, no commands. Phase 2 fills it out with amendment workflow.

| Field | Type | Notes |
|---|---|---|
| `rate_id` | Reference → `pricing.rate` | cascade delete |
| `version_number` | Integer | Sequential per rate |
| `amendment_reason` | Char | Why this version was created |
| `snapshot_json` | Char (multi_line) | Full rate JSON at time of versioning |
| `versioned_by_id` | Reference → `res.user` | Who triggered versioning |
| `versioned_at` | DateTime | When versioned |

---

## 4. Margin Engine

### 4.1 Trigger

Three lifecycle hooks on `RateLine` (all call the same `_recalculate_margin` helper):
- `post.pricing.rate.line.create`
- `post.pricing.rate.line.update`
- `post.pricing.rate.line.delete`

### 4.2 Calculation

```
1. Load all sibling rate lines (same rate_id) from DB
2. Convert each line's unit_rate → rate header currency via res.exchange.rate
   (rate_date = today; fallback: use unit_rate as-is if no FX rate found)
3. total_charge = sum of all converted unit_rates
   (Phase 1: sell rate lines sum = sell_total; buy amounts entered on buy rate)
4. margin_amount  = calculated_margin_amount (stored, updated on each line change)
5. margin_percent = (margin_amount / total_charge × 100) if total_charge > 0 else 0
6. margin_risk_level:
     margin_percent ≥ target threshold  → "safe"
     margin_percent ≥ minimum threshold → "watch"
     margin_percent < minimum threshold → "risk"
     margin_amount < 0                  → "negative_risk"
7. Write back to pricing.rate header:
   rate_proxy.browse(rate_id).write({
       "calculated_margin_percent": margin_percent,
       "calculated_margin_amount": margin_amount,
       "margin_risk_level": risk_level,
   })
```

### 4.3 FX Conversion

- Use `res.exchange.rate` queried for `rate_date = today`, `from_currency = line.currency_id`, `to_currency = rate.currency_id`
- If no exchange rate found: log a warning, use line currency amount unchanged (safe fallback — no silent errors)

### 4.4 Margin Rule Resolution

`_resolve_margin_rule(rate, env)`:
1. If `rate.margin_rule_id` is set: use it directly
2. Otherwise query `pricing.margin.rule` ordered by `priority DESC`
3. Return first where all non-null scope fields match the rate's corresponding fields
4. Fallback: `{"rule_type": "minimum", "margin_basis": "percentage", "margin_value": 5.0}`

---

## 5. Rate Status Machine + Domain Commands

### 5.1 Status Transitions

```
draft
  ──[pricing.rate.submit]──► pending_approval   (when approval rule triggers)
  ──[pricing.rate.submit]──► approved            (auto-approve when no rule triggers)

pending_approval
  ──[pricing.rate.approve]─► approved
  ──[pricing.rate.reject]──► draft               (notification posted to timeline)

approved
  ──[pricing.rate.suspend]─► suspended
  ──(expiry check on read)─► expired             (when valid_to < today)

suspended
  ──[pricing.rate.approve]─► approved            (reinstate)

expired / archived: terminal in Phase 1
```

### 5.2 `pricing.rate.submit` Command

This is the core command. Runs entirely in-process.

1. Guard: rate must be in `draft` status
2. Validate `valid_from < valid_to`
3. Validate at least one charge line exists
4. Validate all lines with `is_mandatory=True` are present (by checking required `product_id` codes per mode — configurable via `ir.config` in Phase 2; Phase 1: skip mandatory check, just check at least one line exists)
5. Generate `rate_number` from `ir.sequence("RATE")` if not already set
6. Run `_resolve_margin_rule(rate)` → get threshold
7. Evaluate: if `margin_risk_level in ("risk", "negative_risk")` and `rule_type == "minimum"`:
   - Create `ir.approval.case(domain="pricing", resource_model="pricing.rate", resource_id=rate.record_uuid, input_snapshot={"margin_percent": ..., "margin_amount": ...})`
   - Set `status = "pending_approval"`
8. Else: set `status = "approved"` (auto-approve — no approval overhead for healthy rates)

### 5.3 `pricing.rate.approve` Command

- Guard: caller must have `pricing.rate:approve` permission
- Guard: caller must not be the rate's `rate_owner_user_id` (maker-checker)
- Set `status = "approved"`
- If linked `ir.approval.case` exists: resolve it as approved

### 5.4 `pricing.rate.reject` Command

- Guard: caller must have `pricing.rate:approve` permission
- Set `status = "draft"`
- Post `communication.message(type="notification")` to rate record with rejection reason
- If linked `ir.approval.case` exists: resolve it as rejected

### 5.5 `pricing.rate.suspend` Command

- Guard: `status == "approved"`
- Guard: caller must have `pricing.rate:suspend` permission
- Set `status = "suspended"`

---

## 6. RBAC Roles & Permissions

### 6.1 Roles (seeded via `pricing_roles.xml`)

| Role Code | Role Name | Description |
|---|---|---|
| `pricing.executive` | Pricing Executive | Create/submit/manage buy & sell rates. Sees buy amounts. Procurement/pricing team. |
| `pricing.approver` | Pricing Approver | Approve/reject rates. Sees buy amounts + margin. Cannot approve own rates (maker-checker). |

Standard logistics roles (`logistics_viewer`, `logistics_supervisor`, `logistics_admin`) get permissions on `pricing.rate` without needing new role records.

### 6.2 Key Permissions (`ir.rbac.permission.csv`)

| Permission Key | Description | Roles |
|---|---|---|
| `pricing.rate:create` | Create new rates | pricing.executive, pricing.approver, logistics_admin |
| `pricing.rate:submit` | Submit rate for approval | pricing.executive, pricing.approver, logistics_admin |
| `pricing.rate:approve` | Approve / reject rates | pricing.approver, logistics_admin |
| `pricing.rate:suspend` | Suspend approved rates | pricing.approver, logistics_admin |
| `pricing.rate:view_buy_amount` | See buy-side unit_rate values | pricing.executive, pricing.approver, logistics_admin |
| `pricing.rate:view_margin` | See margin amount and % | pricing.approver, logistics_supervisor, logistics_admin |
| `pricing.rate:read` | Read approved sell rates | All roles including logistics_viewer |

### 6.3 Field-Level Scrubbing

Enforced in the HTTP controller (not the model layer):
- Missing `pricing.rate:view_buy_amount` → strip `unit_rate` from buy-side lines in API response
- Missing `pricing.rate:view_margin` → strip `calculated_margin_amount`, `calculated_margin_percent` from rate header in API response

---

## 7. XML Views

### 7.1 List View (`pricing.rate.list.xml`)

Columns: `rate_number`, `rate_type`, `mode_id`, `origin_location_id → destination_location_id`, `vendor_id / customer_id`, `valid_from`, `valid_to`, `currency_id`, `calculated_margin_percent` (RBAC-gated), `status`

Default filter: `status = approved` (pricing team sees all; sales sees only approved sell rates via RBAC)

### 7.2 Form View (`pricing.rate.form.xml`)

**Statusbar:** `draft → pending_approval → approved → suspended`

**Action buttons (conditional visibility):**

| Button | Label | Visible when | Command |
|---|---|---|---|
| Submit | Submit for Approval | `status == draft` | `pricing.rate.submit` |
| Approve | Approve | `status == pending_approval` | `pricing.rate.approve` |
| Reject | Reject | `status == pending_approval` | `pricing.rate.reject` |
| Suspend | Suspend | `status == approved` | `pricing.rate.suspend` |

**Form sections:**

1. **Classification** — `mode_id`, `product_id`, `rate_type`, `rate_source`, `currency_id`, `contract_number`, `quote_reference`
2. **Route** — `origin_location_id`, `destination_location_id`, `origin_country_id`, `destination_country_id`, `trade_lane_id`
3. **Party** — `vendor_id`, `customer_id`, `commodity_id`, `entity_id`, `branch_id`, `rate_owner_user_id`
4. **Validity** — `valid_from`, `valid_to`, `renewal_status`
5. **Charge Lines (O2M inline)** — `product_id`, `uom_id`, `container_type_id`, `unit_rate`, `currency_id`, `minimum_amount`, `maximum_amount`, `slab_from`, `slab_to`, `is_mandatory`, `is_customer_visible`, `payment_responsibility`, `remarks`
6. **Margin & Governance** (section hidden for roles without `pricing.rate:view_margin`) — `calculated_margin_amount`, `calculated_margin_percent`, `margin_risk_level`, `margin_rule_id`, `is_global`
7. **Chatter / Timeline** — `communication.message` thread (approval decisions, notifications, activities)

---

## 8. Migration

Single Alembic migration creates four tables:

| Table | Columns |
|---|---|
| `pricing_rate` | All fields from `pricing.rate` model |
| `pricing_rate_line` | All fields from `pricing.rate.line` model, FK → `pricing_rate.record_uuid` |
| `pricing_margin_rule` | All fields from `pricing.margin.rule` model |
| `pricing_rate_version` | Stub fields from `pricing.rate.version`, FK → `pricing_rate.record_uuid` |

Indexes: `pricing_rate(status)`, `pricing_rate(valid_from, valid_to)`, `pricing_rate(rate_number)` unique, `pricing_rate_line(rate_id)`.

---

## 9. Out of Scope for This Implementation

- Excel/CSV bulk upload (Phase 1 roadmap item 05 — deferred)
- Reports and dashboard (Phase 1 roadmap item 06 — deferred)
- Dedicated CRM rate-search endpoint (deferred to Sales CRM implementation)
- `pricing.rate.version` amendment workflow — stub only, Phase 2
- Multi-branch rate scoping — Phase 2
- Expiry cron job — Phase 1 checks expiry on read only
- `pricing.spot.request`, `pricing.contract`, `pricing.renewal.rule` — Phase 2/3

---

## 10. Acceptance Criteria

- [ ] `pricing.rate` can be created with at least one charge line
- [ ] `rate_number` auto-generated from `ir.sequence("RATE")` on submit
- [ ] Margin recalculates automatically when a charge line is added, edited, or deleted
- [ ] Submit on a below-minimum-margin rate creates `ir.approval.case` and sets status `pending_approval`
- [ ] Submit on a healthy-margin rate auto-approves (no manual step)
- [ ] Rate owner cannot approve their own rate (maker-checker guard)
- [ ] Approved rate sets status to `approved`; rejected rate returns to `draft` with notification
- [ ] Sales user (`logistics_viewer`) cannot see `unit_rate` on buy-side lines in API response
- [ ] Sales user cannot see `calculated_margin_amount/percent` in API response
- [ ] Expired rates (`valid_to < today`) blocked from use in quotation (status check on read)
- [ ] Module activates cleanly: `ede serve` starts without errors with `pricing` in `ACTIVE_MODULES`
- [ ] Alembic migration runs cleanly on fresh DB
