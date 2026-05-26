<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Conversation Orchestrator (Converse) — Implementation Docs

**Module:** `foundation.converse` (`src/ede/foundation/converse/`)
**Roadmap:** [roadmap/foundation/converse/](../roadmap/foundation/converse/README.md)
**Status:** 🔴 Not Started — drafted 2026-05-26
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A conversation orchestrator — the engine that turns each inbound message on a `messaging.thread` into (1) an AI-driven intent classification + slot extraction step, (2) a slot-fill ask-back loop when required slots are missing, (3) an action — `compose_reply`, `dispatch_command`, `workflow_transition` (Phase 2), `request_approval` (Phase 2), or `handoff_to_human` — fired when the dialog state's required slots are present, and (4) a persistent `converse.instance` row that survives across messages, days, and reboots so the conversation has memory.

Flow-driven and declarative — each business use case ships a `converse.flow` row that names the intents to recognise, the slots to extract per intent, the ask-back templates, and the action graph. No code per flow. The engine ships once; flows are configuration.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Every consumer that wants "AI bot turns customer messages into ERP records" otherwise re-invents the same seven concerns: intent classifier, slot validator, ask-back template renderer, state machine across messages, action dispatcher with provenance, handoff-to-human escalation, and per-flow confidence threshold tuning. Seven consumers × the same seven concerns = forty-nine ad-hoc implementations that drift. The platform owns it once; consumers ship one `converse.flow` row + one skill pack + a handful of `@api.ai_tool`-decorated commands and get all seven for free.

**Sibling-not-stacked** with [`foundation.assistant`](foundation-assistant.md) — assistant is in-app + synchronous + on-screen + read-only by default; converse is external-party + asynchronous + channel-attached + write-mode by default. The two engines share `foundation.ai` underneath; their lifecycles, safety contracts, and UX surfaces differ.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX entry points** — Settings → Platform AI → Conversation Intake (admin toggles skill packs per organisation); Settings → Technical → Conversation Flows (read-only flow definitions, ops debug); Settings → Technical → Conversation Instances (read-only runtime tracker per `messaging.thread`); external party simply messages on the channel — no EDE UI.
- **Programmatic entry points for other modules** —
  - Author a `converse.flow` XML row in your module's `data/` directory using the `<converse-flow>` DSL (states + slots + transitions + action configs).
  - Author an `ai.prompt.template` for each state's slot-extract step.
  - Author a `converse.skill.pack` binding (flow + tool whitelist + channel kind + per-org enable M2M).
  - Annotate the commands your `dispatch_command` action needs with `@api.ai_tool(write=True, kind="proposer", description=...)`.
  - Subscribe to `converse.instance.advanced` / `converse.action.fired` for analytics or follow-on automation.
- **Integration boundary** — consumes `messaging.inbound_received` from `foundation.messaging`; calls `ai.invoke` from `foundation.ai`; dispatches commands through the function-calling bridge (read-only + Phase 4 write-mode); routes approvals through `ir.approval.case.request` (Phase 2). Never speaks to a provider directly. Never owns chatter wiring (messaging puts every turn into chatter already).
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Inbound message on messaging.thread]
                │
                ▼
   messaging.inbound_received event
                │
                ▼
┌────────────────────────────────────────────────────┐
│  ConverseService.handle_inbound(thread, message)   │
│                                                    │
│  1. Load/create converse.instance for              │
│     (thread, flow). Flow selection is per          │
│     channel via converse.skill.pack rows.          │
│  2. (Phase 2) If awaiting_approval_id set →        │
│     wait-template reply, do NOT re-extract.        │
│  3. AI extraction: ai.invoke with the current      │
│     state's prompt template + JSON-schema slots.   │
│  4. Merge + validate slots via shared AST.         │
│  5. Missing required slots → ask back via          │
│     messaging.send (priority-ordered).             │
│  6. All required slots present → fire action:      │
│        compose_reply    → messaging.send           │
│        dispatch_command → ai-tool bridge dispatch  │
│        workflow_transition → env.workflow.transition  (P2)
│        request_approval → ir.approval.case.request (P2)│
│        handoff_to_human → mark + add followers     │
│  7. Advance state per converse.transition.         │
│  8. Emit converse.instance.advanced event.         │
└────────────────────────────────────────────────────┘
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `converse.flow` | Declarative flow definition (`code`, `name`, `entry_state_id`, `confidence_threshold`, `is_active`) | `src/ede/foundation/converse/models/flow.py` (planned) |
| `converse.state` | One node in the flow's state graph; carries `prompt_template_id`, required/optional slots, `action_kind` + `action_config` JSON | `src/ede/foundation/converse/models/state.py` (planned) |
| `converse.transition` | Directed edge between states with `trigger_kind` (`slot_complete`, `intent_matches`, `confidence_below`, `slot_value_matches`, `unconditional`, `approval_approved` (P2), `approval_rejected` (P2)) + optional guard AST | `src/ede/foundation/converse/models/transition.py` (planned) |
| `converse.slot` | Typed slot spec (`text` / `integer` / `decimal` / `date` / `enum` / `reference`); validator AST; ask-back template | `src/ede/foundation/converse/models/slot.py` (planned) |
| `converse.instance` | Per-thread runtime tracker — current state, collected slot values, awaiting-slot / awaiting-approval flags, AI conversation link, confidence history | `src/ede/foundation/converse/models/instance.py` (planned) |
| `converse.turn.log` | Append-only audit ledger; one row per turn with state-before/after, slots extracted, action fired, action result, confidence, latency | `src/ede/foundation/converse/models/turn_log.py` (planned) |
| `converse.skill.pack` | Bundles flow + tool whitelist + channel scope + per-org enable | `src/ede/foundation/converse/models/skill_pack.py` (planned) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `ConverseService` | Top-level facade + inbound event subscriber (`messaging.inbound_received`) | `src/ede/foundation/converse/services/converse_service.py` (planned) |
| `TurnProcessor` | Single-turn engine — extract → validate → ask-back / fire-action → advance state → log | `src/ede/foundation/converse/services/turn_processor.py` (planned) |
| `SlotExtractor` | `ai.invoke` wrapper with strict JSON-schema enforcement for slot fill + confidence parsing | `src/ede/foundation/converse/services/slot_extractor.py` (planned) |
| `action_dispatchers` | One dispatcher per `action_kind` (`compose_reply`, `dispatch_command`, `handoff_to_human` in Phase 1; `request_approval`, `workflow_transition` added in Phase 2) | `src/ede/foundation/converse/services/action_dispatchers.py` (planned) |
| `approval_bridge` (Phase 2) | Listener on `approval.case.approved` / `approval.case.rejected` — resumes instances awaiting that case and fires the configured response template | `src/ede/foundation/converse/services/approval_bridge.py` (planned) |
| `boot_validator` | Boot-time validity check on every active flow (entry reachable, transitions terminate on declared states, prompts/tools resolvable, no orphans, no terminal-with-outgoing); fails boot loudly | `src/ede/foundation/converse/services/boot_validator.py` (planned) |
| `flow_parser` | XML → dataclasses → DB rows for the `<converse-flow>` DSL | `src/ede/foundation/converse/dsl/flow_parser.py` (planned) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| (no new commands — engine acts on inbound events) | Inbound event subscriber dispatches `ai.invoke`, `messaging.send`, `ir.approval.case.request` (Phase 2) | Engine actions go through existing commands; no new external command surface |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `converse.instance.created` | First inbound on a (thread, flow) tuple creates the instance | Analytics, custom routing |
| `converse.instance.advanced` | After each turn, when the state changes | Analytics, derived workflows |
| `converse.action.fired` | After every action dispatcher returns | Analytics, audit |
| `converse.handoff.triggered` | When the engine hands the thread to a human (low confidence / max-asks exhausted / explicit) | Notification escalation, follower-list updates |
| `converse.instance.awaiting_approval` (Phase 2) | After `request_approval` or `workflow_transition` sets `awaiting_approval_case_id` | Analytics |
| `converse.instance.approval_resolved` (Phase 2) | After the approval listener processes a case event | Analytics |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| (admin-debug routes only, exposed via the standard model CRUD on `converse.flow` / `converse.instance` / `converse.turn.log` / `converse.skill.pack`) | Engine surface is event-driven, not request-driven | `ConverseController` (admin debug — planned) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.converse.handle_inbound` | Last chance to redact PII from `inbound.body` before the AI sees it (Phase 1 ships the hookpoint; production-grade redaction in Phase 3) |
| `pre.converse.action.dispatch_command` | Last chance to veto or rewrite the command + payload before the bridge fires |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
`converse.instance` runtime state is defined entirely by the bound `converse.flow`'s state graph — there is no built-in state machine inside the model. Common shape (e.g. for the `quote_request_v1` flow):

```
[entry: intake] ──slot_complete──► [acknowledged] ──slot_complete──► [submit_for_approval]
                                                                              │
                                                                              ▼
                                                                   [awaiting_approval] (P2)
                                                                       │             │
                                                              ┌────────┘             └────────┐
                                                              ▼ approval_approved             ▼ approval_rejected
                                                       [approved]                       [rejected]
                                                              │                              │
                                                              ▼                              ▼
                                                          (terminal)                   (terminal)
```

Handoff terminates the flow short-circuit at any point: the instance stops processing further inbounds until cleared.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` (in `src/ede/foundation/settings.py`): add `"converse"` after `"messaging"`.
- `ACTIVE_DOMAINS`: n/a.
- Manifest `depends`: `["base", "ai", "messaging", "communication"]` (Phase 2 adds `"approval"` and `"workflow"`).
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `CONVERSE_ENABLED` | `bool` | `True` | `EDE_CONVERSE_ENABLED` | Hard kill-switch for the entire engine |
| `CONVERSE_DEFAULT_CONFIDENCE_THRESHOLD` | `float` | `0.6` | `EDE_CONVERSE_DEFAULT_CONFIDENCE_THRESHOLD` | Default minimum AI confidence; below this the engine asks back rather than firing an action |
| `CONVERSE_MAX_SLOT_ASK_BACKS` | `int` | `3` | `EDE_CONVERSE_MAX_SLOT_ASK_BACKS` | Maximum times the engine asks the same slot before triggering handoff-to-human |
| `CONVERSE_DEFAULT_FALLBACK_ROLE` | `str` | `"sales_ops"` | `EDE_CONVERSE_DEFAULT_FALLBACK_ROLE` | Default RBAC role to subscribe to the thread when `handoff_to_human` fires |
| `CONVERSE_TURN_TIMEOUT_SECONDS` | `int` | `15` | `EDE_CONVERSE_TURN_TIMEOUT_SECONDS` | Max wall-clock time per turn (AI call + action); beyond this the engine fires an apology reply + handoff |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `converse.skill_pack.{code}.enabled` | org | Boolean | (per skill pack default) | Per-org toggle for any seeded skill pack |
| `converse.approval_wait_response_template` | org | Char | `"Still with our team — I'll let you know shortly."` | Phase 2: response sent when a customer texts mid-approval |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| Settings → Platform AI → Conversation Intake | `src/ede/foundation/converse/data/converse_settings.xml` (planned) | Per-skill-pack enable toggles · `converse.approval_wait_response_template` (Phase 2) |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| `data/ir.rbac.permission.csv` (planned) | 5 permissions: `converse.flow.read`, `converse.flow.manage`, `converse.instance.read`, `converse.instance.manage`, `converse.skill_pack.manage` |
| `data/converse_menus.xml` (planned) | Settings → Platform AI → Conversation Intake leaf + Settings → Technical → Conversation Flows / Conversation Instances ops-debug leaves |
| `data/flow_quote_request_v1.xml` (planned) | The freight quote intake flow — owned by [`uc.freight-quote-via-telegram`](../roadmap/usecases/freight-quote-via-telegram.md) |
| `data/prompt_quote_request.xml` (planned) | `ai.prompt.template` for the slot-extract step |
| `data/skill_pack_freight_quote.xml` (planned) | The `freight_quote_intake_v1` skill pack |
| `data/reply_templates_quote_request.xml` (planned) | Reply templates (draft-acknowledged in Phase 1; approved + rejected + submitted-acknowledged in Phase 2) |
| `demo/demo_freight_quote_via_telegram.xml` (planned) | Demo Telegram channel + identity + partner + transcript |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Engine + Slot-Fill Loop + Quote-Request Reference Flow | 🔴 Not Started | [phase-1-implementation.md](../roadmap/foundation/converse/phase-1-implementation.md) |
| Phase 2 | Approval Round-Trip + Workflow Action + Closed-Loop Reference Flow | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/converse/phase-2-implementation.md) |
| Phase 3 (planned) | Visual Flow Editor + Multi-Language + More Action Kinds | 🔴 Not Started | (drafted in README) |
| Phase 4 (planned) | Cross-Channel Identity + Operator Override + Long-Running Plans | 🔴 Not Started | (drafted in README) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| _none yet_ | | | |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| Entire module is 🔴 Not Started — Phase 1 drafted but not shipped | 🔴 | [phase-1-implementation.md](../roadmap/foundation/converse/phase-1-implementation.md) |
| Approval round-trip back to channel (closes the customer-visible loop) | 🔴 (Phase 2) | [phase-2-implementation.md](../roadmap/foundation/converse/phase-2-implementation.md) |
| Open: AI write-mode `provenance-only` policy for asynchronous external-party-driven dialogs | 🔴 (blocks Phase 1 closed-loop) | [`uc.freight-quote-via-telegram` open questions](../roadmap/usecases/freight-quote-via-telegram.md#open-questions) |
| Visual flow editor | 🔴 (Phase 3) | [README](../roadmap/foundation/converse/README.md) |
| Multi-language detect + reply-language selection | 🔴 (Phase 3) | [README](../roadmap/foundation/converse/README.md) |
| Cross-channel identity merge | 🔴 (Phase 4) | [README](../roadmap/foundation/converse/README.md) |
| Operator override of an in-flight dialog | 🔴 (Phase 4) | [README](../roadmap/foundation/converse/README.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- _Populated as the first cross-engine use case ships._

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Phase 1 ships a single Alembic revision creating the seven `converse.*` tables + the `converse.skill.pack ↔ res.organization` M2M join.
- Phase 2 ships **zero new migrations** — Phase 1 declares the relevant Enum values up front (including Phase 2 values) so Phase 2 is pure code + data.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| `internal_user` | `converse.flow.read`, `converse.instance.read` |
| `system_admin` | `converse.flow.manage`, `converse.instance.manage`, `converse.skill_pack.manage` |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [`foundation.messaging`](foundation-messaging.md) — inbound + outbound transport (hard prereq)
- [`foundation.ai`](foundation-ai.md) — provider, tools, prompts, cost, write-mode provenance (hard prereq)
- [`foundation.approval`](foundation-approval.md) — sign-off engine (Phase 2 soft prereq)
- [`foundation.workflow`](foundation-workflow.md) — shared AST evaluator (soft prereq) + Phase 2 `workflow_transition` action
- [`foundation.communication`](foundation-communication.md) — chatter mirror (via messaging)
- [`foundation.assistant`](foundation-assistant.md) — sibling in-app surface (different lifecycle)
- Use case driver: [`uc.freight-quote-via-telegram`](../roadmap/usecases/freight-quote-via-telegram.md)
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-26. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
