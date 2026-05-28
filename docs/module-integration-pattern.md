# Module Integration Pattern — Producer Owns the Contract, Consumers Extend

> **Status:** Architecture guidance (hand-authored). Not auto-maintained from the roadmap.
> **Companion:** [Model & View Extension SDK](foundation-base-extensions.md) — the *mechanics*. This doc is the *pattern* that sits on top of them.

How do two apps integrate in EDE — one that **produces** work and one (or several) that **process** it — without a third "integration" module to glue them together? This guide answers that, using the sales→operations handover as the canonical example.

The short version: **there is no middle integration module.** One module owns a boundary object (the contract); each downstream consumer reaches into it on its own terms, owns its own hookup, and signals back through events. The dependency graph stays acyclic and one-directional, so consumers are independently activatable and adding a new one never touches the producer or its siblings.

> **⛔ Governance — hard rule.** Introducing a middle "integration" / "bridge" module between two apps requires **explicit user approval**, and is to be **avoided by default**. See the binding rule in [CLAUDE.md → What NOT To Do](../CLAUDE.md). Always present the producer-owns-the-contract solution **first**; only escalate to a bridge for the two narrow cases in [When a separate integration module *is* warranted](#when-a-separate-integration-module-is-warranted), and only with explicit approval.

---

## The shape

```
┌──────────────────────────────────────────────────────────────┐
│  PRODUCER  (logistics.sales_crm)                              │
│  Owns crm.handover — the boundary object / integration        │
│  contract. Terminates here: produces the operational brief,   │
│  then stops. Knows nothing about who consumes it.             │
└───────────────▲───────────────────────▲──────────────────────┘
                │ depends on            │ depends on
   ┌────────────┴───────────┐  ┌────────┴────────────────┐
   │ CONSUMER               │  │ CONSUMER (future)        │
   │ logistics.booking      │  │ logistics.shipments      │
   │ • extend_model: own FK │  │ • extend_model: own FK   │
   │ • own command          │  │ • own command            │
   │ • own button (extend   │  │ • own button (extend     │
   │   ref on the form)     │  │   ref on the form)       │
   └────────────────────────┘  └──────────────────────────┘
        events ▲                          events ▲
        └──────┴──── back-channel (consumer → producer) ───┘
                    NEVER a reverse import
```

- **Producer** (`logistics.sales_crm`) owns `crm.handover`. It is the sales→ops boundary record — the "operational brief" produced when a quote is accepted. sales_crm depends on **neither** booking nor shipment.
- **Consumers** (`logistics.booking`, future `logistics.shipments`) each depend on `sales_crm` and own their entire side of the integration:
  - a **field** contributed via [`@api.extend_model("crm.handover")`](foundation-base-extensions.md) — their own FK back to the spawned record;
  - a **command** (`logistics.booking.create_from_handover`) that reads the handover and spawns their record;
  - a **button** on the handover form, wired via `<extend ref="sales_crm.crm_handover_form_view">` from the consumer's own view file.
- **No bridge module.** There is no `sales_ops_integration` app. The handover *is* the contract; each consumer is a self-contained additive slice.

The handover is singular. "handover → booking" and "handover → shipment" are **not two integration objects** — they are two **paths of the same handover**, recorded by a routing field. This is a Y-fork, not two pipes.

---

## Why no middle module

A bridge module would have to depend on the producer **and** every consumer — a fan-in hub. That:

- **Couples releases.** Touching booking's conversion would force a bridge release that also carries shipment.
- **Creates ownership ambiguity.** Three modules co-owning one model's behaviour is harder to reason about than one owner + thin per-consumer extensions.
- **Fights the framework grain.** EDE ships `@api.extend_model` + `<extend ref>` precisely so a consumer can reach into an owner cleanly. A glue module re-introduces the very coupling the SDK removes.

The per-consumer pattern instead keeps each integration **independently activatable**: drop `booking` from `ACTIVE_MODULES` and its handover field, button, and command simply don't load — the "booking" fork just isn't offered. Clean degradation, zero edits elsewhere.

---

## The three rules that keep it valid

The "no middle module" property is not free — it holds **only while three invariants hold**. Break any one and you either need events or a real bridge.

### Rule 1 — The dependency graph stays acyclic and one-directional

Consumers depend on the producer, never the reverse. The producer must never `import` or `depends`-on a consumer. The moment sales_crm needs to *read* booking, you have a cycle — and the pattern collapses. Keep all reverse signalling on **events** (Rule 3).

### Rule 2 — The producer owns the neutral routing slot + its invariant; consumers own only their own value

The routing decision ("which fork did this handover take?") is a property of the **handover**, so the slot and its mutual-exclusion invariant ("once routed, frozen") belong to the **producer** (sales_crm) — not to a consumer.

But the producer must not know downstream *identities* (it can't depend on them). So the slot carries **no hardcoded downstream vocabulary**. Two correct shapes:

- A neutral string/slot the producer owns, into which each consumer writes its **own** code (`"booking"`, `"shipment"`); the producer enforces only "non-null ⇒ frozen".
- **Preferred:** a **Reference to a small master** (per the [master-over-enum principle](foundation-model-naming.md)), where each consumer **seeds its own row**. The producer owns the field + freeze guard; consumers own their seed row + their FK + their button + their command. No consumer ever declares a sibling's value.

> **Anti-pattern to avoid:** a consumer declaring the *whole* vocabulary. If `logistics.booking`'s extension defines `path_taken` with **both** `"booking"` and `"shipment"` values, then booking "knows about" shipment — exactly the cross-module leak the architecture exists to prevent. When shipment ships it collides or duplicates. The fix is Rule 2: move the slot to the owner, make it master-backed, let each consumer seed only its own row.

### Rule 3 — The back-channel is events, never a reverse import

When the producer wants to *reflect downstream progress* (e.g. show "booking confirmed / in transit" on the handover), do **not** have sales_crm read booking — that violates Rule 1. Instead:

- The consumer **emits events** as its record progresses.
- A neutral `fulfillment_status` on the handover is advanced **either** by the consumer writing back (consumer → producer write is still one-directional, and fine) **or** by an event subscriber.

A consumer *writing into* the producer's record is acyclic and allowed. The producer *reading* a consumer's model is a cycle and forbidden.

---

## What each side ships (checklist)

**Producer (`sales_crm`) — once:**
- [ ] The boundary model (`crm.handover`) with all the data a consumer needs to spawn its record (parties, route, cargo, schedule, charges, commercial refs).
- [ ] A neutral routing slot (master-backed Reference) + the "frozen once set" guard (mutual exclusion).
- [ ] A neutral `fulfillment_status` slot if downstream progress must surface on the handover.
- [ ] *(When a second consumer arrives)* a canonical reader — e.g. `handover.to_operational_payload()` — so consumers map from **one** contract instead of duplicating the field-copy logic. Don't pre-build it for a single consumer.

**Each consumer (`booking`, future `shipments`) — additively, inside itself:**
- [ ] `@api.extend_model("crm.handover")` contributing **only its own FK** (`booking_id`).
- [ ] Its seed row in the routing master (its own value only).
- [ ] Its `…create_from_handover` command — validates the freeze guard + producer-side preconditions, spawns the record, stamps the routing slot + FK.
- [ ] Its button on the handover form via `<extend ref="sales_crm.crm_handover_form_view">` from the consumer's own view file.
- [ ] Events as its record progresses (so the producer's `fulfillment_status` can advance without a reverse import).

---

## Canonical example — the handover Y-fork

Files, as wired today:

| Concern | Owner | Where |
|---|---|---|
| `crm.handover` model | `logistics.sales_crm` | [`models/handover.py`](../src/domains/logistics/sales_crm/models/handover.py) |
| Y-fork FK + path field | `logistics.booking` | [`models/handover_extension.py`](../src/domains/logistics/booking/models/handover_extension.py) — `@api.extend_model("crm.handover")` |
| Conversion command | `logistics.booking` | [`models/booking.py`](../src/domains/logistics/booking/models/booking.py) — `logistics.booking.create_from_handover` |
| Mutual exclusion (BR-BK-CONV-02) | enforced in the command | rejects when the handover is already routed |
| "Create Booking" button | **not yet wired** | target: `<extend ref="sales_crm.crm_handover_form_view">` from booking |

### Current state vs. target shape

The pattern above is the **target**. Two gaps exist in the handover implementation today, both worth closing before `logistics.shipments` arrives:

1. **Routing slot ownership (Rule 2).** `path_taken` is currently declared on **booking's** extension and hardcodes *both* `"booking"` and `"shipment"` values. Target: move the neutral slot + freeze guard onto `crm.handover` (owned by sales_crm), make it master-backed, and have each consumer seed only its own row.
2. **Missing button (UI gap).** `create_from_handover` exists but no handover view dispatches it — the Y-fork is backend-only. Target: wire the button via `<extend ref>` from booking.

Closing both is a producer-side substrate refactor + a consumer-side button slice. After that, adding shipment is a **pure additive slice** with zero changes to sales_crm or booking.

---

## When a separate integration module *is* warranted

The "no middle module" rule is specific to this topology (one producer → N consumers, acyclic). A dedicated bridge earns its keep only when:

- **Cross-domain peers that must not depend on each other** — e.g. `logistics` ↔ `finance`. Neither may own the extension (it would create a cross-domain dependency, forbidden by [CLAUDE.md](../CLAUDE.md)). The shared concept lifts to `foundation.base` as a `res.*` master, or — rarely — into a bridge that depends on both. *(booking / shipments / sales_crm are all within `logistics`, so this does not apply.)*
- **Integration with its own persistent lifecycle and state** — a reconciliation ledger, a cross-module orchestration saga with its own status machine, audit trail, and compensation logic. When the integration *itself* is a stateful entity owned by neither side, give it a home. *(The handover's state lives fine on the handover, so this does not apply yet.)*

If neither holds, a bridge module is pure overhead. Default to producer-owns-the-contract.

---

## Related

- [Model & View Extension SDK](foundation-base-extensions.md) — the `@api.extend_model` + `<extend ref>` mechanics this pattern is built on.
- [Foundation Model Naming](foundation-model-naming.md) — `res.*` vs domain models; master-over-enum guidance for the routing slot.
- [Command & Event Usage Guide](15-command-event-guide.md) — when to use commands vs events; the back-channel in Rule 3.
- [Platform Implementation Rules](../roadmap/platform/00-execution-rules.md) — cross-app field/view additions are non-negotiable via the SDK.
