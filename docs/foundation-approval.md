<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Approval Workflow Engine — Implementation Docs

**Module:** `foundation.approval` (`src/ede/foundation/approval/`)
**Roadmap:** [roadmap/foundation/approval/README.md](../roadmap/foundation/approval/README.md)
**Status:** ✅ Phase 1 Delivered 2026-05-10 — engine, inbox, case detail page, notification dispatch + 24 templates, pricing-rate subject-lock adoption shipped and verified end-to-end via the 8-step pricing.rate browser walkthrough.
**Layer:** Foundation engine

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
A **domain-agnostic, multi-step approval workflow engine** that any EDE domain can plug into. Domains define *what* needs approval; the approval engine handles *how* it flows through approvers, SLAs, escalations, delegations, and decision audit trails. The engine is shipped as `foundation.approval` and consumed by every EDE domain that needs structured sign-offs.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
Approval flows are a recurring cross-domain concern: pricing rate sign-offs, procurement POs, expense claims, HR leave, finance journal postings — each one needs maker-checker, multi-step sign-offs, SLA-tracked approvals, delegation, and escalation. Building this once at the platform layer prevents every domain from re-implementing the same state machine, audit ledger, escalation worker, and rule engine. **Domains never build their own approval engine** — they reuse the primitives this module already provides (`ir.approval.case`, `flow.template`, `policy.set`, `escalation.policy`, decision ledger, SLA worker).
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
**Consuming-domain integration checklist** (from roadmap):
- **Manifest** — depend on `foundation.approval` in `__manifest__.py`.
- **Configuration data** — seed flow templates, policy sets, rules, and escalation policies as CSV/XML.
- **Submit trigger** — on the domain command that needs approval, dispatch `ir.approval.case.request` with `domain`, `resource_model`, `resource_id`, `input_data`.
- **Subject status field** — add a `pending_approval` state on the subject model.
- **Listen for decision** — subscribe to `approval.case.approved` / `approval.case.rejected` / `approval.case.returned` / `approval.case.cancelled` / `approval.case.recalled` events for `resource_model == "<your model>"` and transition subject status accordingly.
- **Lock subject** — call `register_subject_lock_on(env, model_key="<your.model>", locked_fields=[...], locked_states=("PENDING","APPROVED"))` from your domain's registration hook ([`subject_lock.py`](../src/ede/foundation/approval/services/subject_lock.py)). The helper installs `pre.{model}.update` / `pre.{model}.delete` hooks that veto edits while a related case is locked.
- **RBAC** — define approver roles and permission keys; reference them in `flow.step.def.approver_role`. Add `<domain>.recall` to permissions if recall is supported.
- **Notification templates** — approval ships its own templates for `approval.task.assigned`, `approval.case.{approved,rejected,returned,cancelled,recalled}`, `approval.task.{delegated,sla_breach}` per transport ([`data/notification_templates.xml`](../src/ede/foundation/approval/data/notification_templates.xml)). Consuming domains register additional templates only if they want domain-specific copy.
- **Tests** — verify submit → request → decide → status update path end-to-end.

**End-user UX entry points:**
- **Approvals** root app in the app-switcher (`category=application`, sequence 30) — opens the "My Approvals" inbox by default. Backed by `approval.menu_root_app` + `approval.action_inbox` (`client_component="approval.inbox"`).
- App-switcher icon shows a numeric badge equal to the user's pending tasks (sourced from `pending_approval_count` on `/api/web/bootstrap` and refreshed on every `web.client.notification` event with `event_key.startsWith("approval.")`).
- "All Cases" leaf under the same root for case-level browsing; admin "Configurations" sub-menu groups Cases / Flow Templates / Policy Sets for ops users.
- Per-user pending-task list also exposed programmatically via `GET /api/approval/cases/{id}/inbox`.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Domain]                       [foundation.approval]
─────────                      ─────────────────────
pricing.rate.submit  ──►  ir.approval.case.request
                                    │
                                    ▼
                          policy.set + rule.match
                                    │
                                    ▼
                          flow.template (steps)
                                    │
                                    ▼
                          ir.approval.task (per approver)
                                    │
                          (decide / delegate / escalate)
                                    │
                                    ▼
pricing.rate.approve  ◄──  ir.approval.case.{APPROVED|REJECTED}
```
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.approval.case` | Single approval lifecycle record (state machine root) | [src/ede/foundation/approval/models/approval_case.py](../src/ede/foundation/approval/models/approval_case.py) |
| `ir.approval.task` | Per-approver task with SLA deadline and delegation depth | [src/ede/foundation/approval/models/](../src/ede/foundation/approval/models/) |
| `ir.approval.flow.template` | Multi-step approver chain definition | [src/ede/foundation/approval/models/](../src/ede/foundation/approval/models/) |
| `ir.approval.flow.step.def` | Step within a flow template (serial/parallel, ALL/ANY join) | [src/ede/foundation/approval/models/](../src/ede/foundation/approval/models/) |
| `ir.approval.policy.set` | Domain-scoped grouping of rules | [src/ede/foundation/approval/models/](../src/ede/foundation/approval/models/) |
| `ir.approval.rule` | AST expression rule that selects the flow template | [src/ede/foundation/approval/models/](../src/ede/foundation/approval/models/) |
| `ir.approval.escalation.policy` | SLA breach handling (escalate / auto-approve / auto-reject) | [src/ede/foundation/approval/models/](../src/ede/foundation/approval/models/) |
| `ir.approval.delegation.policy` | Out-of-office / branch-scope delegation rules | [src/ede/foundation/approval/models/](../src/ede/foundation/approval/models/) |
| `ir.approval.decision` | Immutable decision ledger entry | [src/ede/foundation/approval/models/audit.py](../src/ede/foundation/approval/models/audit.py) |
| `ir.approval.event.log` | Append-only event audit stream | [src/ede/foundation/approval/models/audit.py](../src/ede/foundation/approval/models/audit.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `ApprovalService` | Orchestrator + AST rule evaluator (case lifecycle, flow execution, decision recording) | [src/ede/foundation/approval/services/](../src/ede/foundation/approval/services/) |
| `AssignmentService` | Resolves approver role → users → creates tasks | [src/ede/foundation/approval/services/](../src/ede/foundation/approval/services/) |
| `SlaWorker` | Tick-based escalation: queries overdue tasks on `ede.worker.tick` and dispatches escalations | [src/ede/foundation/approval/services/](../src/ede/foundation/approval/services/) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| `ir.approval.case.request` | Consuming domain dispatches with `domain`, `resource_model`, `resource_id`, `input_data` | Creates case, matches rule, instantiates flow, opens first step's tasks |
| `ir.approval.case.decide` | Approver clicks Approve/Reject/Return | Records decision, advances/closes case, emits state-transition event |
| `ir.approval.case.delegate` | Approver delegates a task | Reassigns task, increments delegation depth, audit-logs |
| `ir.approval.case.escalate` | `SlaWorker` on SLA breach (or manual) | Applies escalation policy (escalate / auto-approve / auto-reject) |
| `ir.approval.case.cancel` | Requester or admin cancels | Transitions case → CANCELLED |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| `approval.task.assigned` | `AssignmentService` spawns a task for an approver | Notification engine, in-app bell, inbox refresh |
| `approval.task.delegated` | Approver delegates a task to another user | Notification engine; original + new assignee notified |
| `approval.task.sla_breach` | `SlaWorker` detects a task past its deadline | Escalation policy executor, notifications |
| `approval.case.approved` | `ApprovalService.decide_case` transitions case to APPROVED | Originating domain (transition subject status), notifications |
| `approval.case.rejected` | Case transitions to REJECTED | Originating domain, notifications |
| `approval.case.returned` | Case transitions to RETURNED (sent back to requester) | Originating domain, notifications |
| `approval.case.cancelled` | Requester or admin cancels the case | Originating domain, notifications |
| `approval.case.recalled` | Admin with `approval.recall` permission recalls an APPROVED case | Originating domain (revert subject), notifications |

All seven `approval.case.*` and three `approval.task.*` events are emitted via `ApprovalService._emit_case_event` / corresponding hooks in `assignment_service.py` and `sla_worker.py`. Notification dispatch is decoupled — services dispatch a `notification.send` Command per event; the dispatcher in `foundation.notifications` resolves recipients, renders templates, and persists `ir.notification` rows.
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `* /api/approval/cases` | CRUD on cases + `decide` / `delegate` / `cancel` / `recall` actions | [src/ede/foundation/approval/api/approval_routes.py](../src/ede/foundation/approval/api/approval_routes.py) |
| `* /api/approval/tasks/{id}` | Per-task read / action endpoints | [src/ede/foundation/approval/api/approval_routes.py](../src/ede/foundation/approval/api/approval_routes.py) |
| `GET /api/approval/cases/{id}/inbox` | Per-user pending-task list — consumed by [`ApprovalInboxView.tsx`](../src/frontend/src/workspace/views/client/ApprovalInboxView.tsx) | [src/ede/foundation/approval/api/approval_routes.py](../src/ede/foundation/approval/api/approval_routes.py) |

The inbox view polls every 30s and additionally refreshes (debounced 500ms) on any `web.client.notification` event whose `event_key` starts with `approval.`. The pending-task count is computed at boot in `bootstrap.py:262-287` and surfaced as a numeric badge on the Approvals app icon in `AppSwitcher.tsx`.
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| `pre.<subject_model>.update` (registered per consumer) | Installed by `register_subject_lock_on(env, model_key=..., locked_fields=[...], locked_states=("PENDING","APPROVED"))`. Looks up the latest `ir.approval.case` for `(resource_model, resource_id)` and raises `ValueError` listing offending fields if the case is in a locked state. |
| `pre.<subject_model>.delete` (registered per consumer) | Same registration call also vetoes deletes while the subject is in a locked-case state. |

Helper lives in [`services/subject_lock.py`](../src/ede/foundation/approval/services/subject_lock.py). Two entry points: `make_subject_lock_hooks(...)` returns ready-decorated hook callables for static `@api.on_hook` registration; `register_subject_lock_on(env, ...)` registers programmatically against an existing registry at boot time.
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
```
DRAFT ──► PENDING ──► APPROVED ──► RECALLED
                 ├──► REJECTED
                 ├──► RETURNED
                 └──► CANCELLED
```
Implemented in `approval_case.py:12–125`. The `RECALLED` transition (APPROVED → RECALLED) requires the actor to hold the `approval.recall` permission and writes an `ir.approval.decision` ledger row with `decision="RECALL"`.
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> ⚠ **Roadmap predates the Configuration Introduced discipline (added 2026-05-08).** Backfill needed in `roadmap/foundation/approval/README.md` and feature files via the `roadmap-driven-delivery` skill. Until then, this section reflects no declared knobs even though the engine ships RBAC seed data and approval menus that may qualify.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `approval`
- `ACTIVE_DOMAINS` entry (in `src/domains/settings.py`): n/a (foundation app)
- Manifest `depends`: `["foundation.notifications"]` (per [src/ede/foundation/approval/__manifest__.py](../src/ede/foundation/approval/__manifest__.py))
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| `approval.team_role.default_resolution` | system | selection | `primary_only` | Default `resolution_strategy` for TEAM_ROLE steps that leave it unset (`primary_only` / `all_parallel`) |
| `approval.team_role.default_escalation` | system | selection | `next_in_sequence` | Default `escalation_strategy` (`none` / `next_in_sequence` / `walk_up_hierarchy`) |
| `approval.team_role.default_max_escalations` | system | integer | `3` | Default cap on escalation walks before a TEAM_ROLE case stalls |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| Settings → Approvals → Team-Role Routing Defaults | [views/approval_team_role_settings.xml](../src/ede/foundation/approval/views/approval_team_role_settings.xml) | `approval.team_role.default_resolution`, `approval.team_role.default_escalation`, `approval.team_role.default_max_escalations` |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| [`data/ir.rbac.permission.csv`](../src/ede/foundation/approval/data/ir.rbac.permission.csv) | Permission registry entries — case/task/decision/event-log read, plus `approval.recall`. |
| [`data/notification_templates.xml`](../src/ede/foundation/approval/data/notification_templates.xml) | 8 events × 3 transports (email / web / in_app) = 24 `ir.notification.template` rows for `approval.task.{assigned,delegated,sla_breach}` + `approval.case.{approved,rejected,returned,cancelled,recalled}`. |
| [`data/approval_menus.xml`](../src/ede/foundation/approval/data/approval_menus.xml) | Approvals app-switcher entry (`menu_root_app`, `category=application`), inbox client action (`approval.action_inbox`, `client_component="approval.inbox"`), All Cases leaf, admin Configurations sub-menu. |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Production-Ready Core + Standalone App + Notification Integration | ✅ Delivered 2026-05-10 | [phase-1-implementation.md](../roadmap/foundation/approval/phase-1-implementation.md) |
| Phase 2 | Operational Power Features | 🔴 Not Started | [phase-2-implementation.md](../roadmap/foundation/approval/phase-2-implementation.md) |
| Phase 3 | Cross-Domain & Analytics | 🔴 Not Started | [phase-3-implementation.md](../roadmap/foundation/approval/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| Team-Role Routing (Enh 01) — `TEAM_ROLE` assignment kind resolving the record's-team role-holder via shared `TeamRoleService`; PRIMARY_ONLY/ALL_PARALLEL resolution; NEXT_IN_SEQUENCE/WALK_UP_HIERARCHY/NONE escalation; zero-assignee→immediate walk-up at create; `max_escalations` cap → stall; supersede-not-delete + `escalation_depth`; save-time validators; `step_assigned`/`escalated`/`stalled` audit rows | `ir.approval.flow.template` (+`subject_model_key`/`subject_team_field`), `ir.approval.flow.step.def` (+`TEAM_ROLE`/`resolution_strategy`/`escalation_strategy`/`max_escalations`), `ir.approval.task` (+`escalation_depth`/`superseded_by`) | [services/team_role_routing.py](../src/ede/foundation/approval/services/team_role_routing.py), wired in [services/approval_service.py](../src/ede/foundation/approval/services/approval_service.py) (`_create_step_tasks`, `_escalate_team_role`); validators on [models/flow_template.py](../src/ede/foundation/approval/models/flow_template.py); Alembic `a1f4c9d27b03`; 28 tests in [test_team_role_routing.py](../src/tests/foundation/approval/test_team_role_routing.py); guide [docs/foundation-approval-team-role-routing.md](./foundation-approval-team-role-routing.md) | [Enh 01](../roadmap/foundation/approval/enhancements/01-team-role-routing.md) |
| `ApprovalCaseView` full-page case detail | n/a | [src/frontend/src/workspace/views/client/ApprovalCaseView.tsx](../src/frontend/src/workspace/views/client/ApprovalCaseView.tsx) — top-bar status pill + back, left-rail subject snapshot, main-pane step tree (parallel-approver call-out + per-task assignee/state/SLA + "you" highlight), right-rail decision timeline, ActionBar (Approve/Reject/Return/Delegate) gated on having a PENDING task. New `approval.action_case` (path `approval-case`, `client_component="approval.case"`) in [data/approval_menus.xml](../src/ede/foundation/approval/data/approval_menus.xml). ClientActionRegistry + WorkspaceActionController extended to forward URL `param`. Inbox "Open" navigates here via `useNavigate`; in-place drawer removed. | [Phase 1 B3](../roadmap/foundation/approval/phase-1-implementation.md#b3-case-detail-view--approvalcaseview-react-component) |
| Pricing-rate subject-lock adoption | n/a | [src/domains/logistics/pricing/models/rate.py](../src/domains/logistics/pricing/models/rate.py) — module-level `make_subject_lock_hooks(model_key="pricing.rate", locked_fields=[16 fields per spec], …)` plus custom parent-walk hook for `pricing.rate.line` create/update/delete. 21 unit tests in [src/tests/pricing/test_subject_lock.py](../src/tests/pricing/test_subject_lock.py). | [Phase 1 A2 step 2](../roadmap/foundation/approval/phase-1-implementation.md#a2-reusable-subject-lock-helper) |
| Models (10) | `ir.approval.case`, `ir.approval.task`, `ir.approval.flow.template`, `ir.approval.flow.step.def`, `ir.approval.policy.set`, `ir.approval.rule`, `ir.approval.escalation.policy`, `ir.approval.delegation.policy`, `ir.approval.decision`, `ir.approval.event.log` | [src/ede/foundation/approval/models/](../src/ede/foundation/approval/models/) | [README — ✅ Built](../roadmap/foundation/approval/README.md#-built) |
| Services | n/a | `ApprovalService`, `AssignmentService`, `SlaWorker`, `RuleContextBuilder` ([rule_context.py](../src/ede/foundation/approval/services/rule_context.py)), `subject_lock` helpers ([subject_lock.py](../src/ede/foundation/approval/services/subject_lock.py)) | [README — ✅ Built](../roadmap/foundation/approval/README.md#-built) |
| REST API | n/a | [src/ede/foundation/approval/api/approval_routes.py](../src/ede/foundation/approval/api/approval_routes.py) | [README — ✅ Built](../roadmap/foundation/approval/README.md#-built) |
| Commands | `ir.approval.case.{request, decide, delegate, escalate, cancel, recall}` | [src/ede/foundation/approval/models/approval_case.py](../src/ede/foundation/approval/models/approval_case.py), [services/approval_service.py](../src/ede/foundation/approval/services/approval_service.py) | [Phase 1 A1, A5](../roadmap/foundation/approval/phase-1-implementation.md#a5-implement-recall-command--handler) |
| State machine | `ir.approval.case` | `approval_case.py:12–125` (DRAFT → PENDING → APPROVED → RECALLED \| REJECTED \| RETURNED \| CANCELLED) | [README — ✅ Built](../roadmap/foundation/approval/README.md#-built) |
| Lifecycle event emission | n/a | `ApprovalService._emit_case_event` covers approved / rejected / returned / cancelled / recalled ([approval_service.py:782-795](../src/ede/foundation/approval/services/approval_service.py)) | [Phase 1 A1](../roadmap/foundation/approval/phase-1-implementation.md#a1-emit-lifecycle-events-on-case-state-change) |
| Subject-lock helper | n/a | [services/subject_lock.py](../src/ede/foundation/approval/services/subject_lock.py) — `make_subject_lock_hooks` + `register_subject_lock_on` (185 lines, 4 tests) | [Phase 1 A2](../roadmap/foundation/approval/phase-1-implementation.md#a2-reusable-subject-lock-helper) |
| Rule-context schema | n/a | [services/rule_context.py](../src/ede/foundation/approval/services/rule_context.py) — `subject.*`, `requester.*`, `org.*`, `now.*`, `input.*` namespaces (143 lines, 7 tests) | [Phase 1 A3](../roadmap/foundation/approval/phase-1-implementation.md#a3-standardize-rule-context-schema) |
| Notification dispatch | n/a | `ApprovalService._dispatch_notification` ([approval_service.py:818-841](../src/ede/foundation/approval/services/approval_service.py)) — replaces direct `NotificationService` calls with `Command("notification.send", ...)` | [Phase 1 A4](../roadmap/foundation/approval/phase-1-implementation.md#a4-switch-to-notificationsend-command-dispatch) |
| Principal binding | n/a | `_require_principal_user_id` guard; all decider/requester identity reads from `env.principal` | [Phase 1 A6](../roadmap/foundation/approval/phase-1-implementation.md#a6-principal-binding) |
| Audit trail | `ir.approval.decision`, `ir.approval.event.log` | [src/ede/foundation/approval/models/audit.py](../src/ede/foundation/approval/models/audit.py) | [README — ✅ Built](../roadmap/foundation/approval/README.md#-built) |
| Test suite | n/a | 31 tests in [src/tests/foundation/approval/](../src/tests/foundation/approval/) — `test_case_lifecycle` (5), `test_parallel_steps` (2), `test_sla_escalation` (2), `test_delegation` (2), `test_recall` (3), `test_rule_context` (7), `test_subject_lock` (4), `test_notification_dispatch` (4), `test_principal_guard` (2) | [Phase 1 A7](../roadmap/foundation/approval/phase-1-implementation.md#a7-test-suite--full-lifecycle) |
| Consumer integration guide | n/a | [docs/foundation-approval-engine-guide.md](./foundation-approval-engine-guide.md) | [Phase 1 A8](../roadmap/foundation/approval/phase-1-implementation.md#a8-consumer-integration-guide) |
| RBAC seed | n/a | [data/ir.rbac.permission.csv](../src/ede/foundation/approval/data/ir.rbac.permission.csv) (case/task/decision/event-log + `approval.recall`) | [README — ✅ Built](../roadmap/foundation/approval/README.md#-built) |
| Approvals root app | n/a | `approval.menu_root_app` (`category=application`, sequence 30) in [data/approval_menus.xml](../src/ede/foundation/approval/data/approval_menus.xml) | [Phase 1 B1](../roadmap/foundation/approval/phase-1-implementation.md#b1-promote-approval-to-root-app) |
| "My Approvals" inbox page | n/a | `approval.inbox` client action — [`InboxPage.tsx`](../src/ede/foundation/approval/frontend/components/InboxPage.tsx): Triage × Dashboard focus card (SLA + domain chips, requester meta, stat grid from `input_snapshot`, workflow approval chain + heartbeat), pager + KPI strip, sticky 1fr/360px layout with a right rail (Out today bulk-delegate · Your queue navigator · Common delegates · queue rules), inline approve/reject/comment/delegate, SSE refresh + keyboard nav; [`ApprovalService.ts`](../src/frontend/src/core/services/ApprovalService.ts) HTTP client | [Approval Enhancement 02](../roadmap/foundation/approval/enhancements/02-approval-inbox-frontend2.md) (frozen [mockup](../roadmap/foundation/approval/mockups/approval-inbox-frozen.html)) |
| Pending-task badge | n/a | `pending_approval_count` end-to-end: [bootstrap.py:262-287](../src/ede/foundation/base/api/bootstrap.py) → `WorkspaceService.ts` → `AppSwitcher.tsx` numeric badge | [Phase 1 B4](../roadmap/foundation/approval/phase-1-implementation.md#b4-bootstrap-inbox-into-webclient-context) |
| Notification template fixtures | `ir.notification.template` × 24 rows | [data/notification_templates.xml](../src/ede/foundation/approval/data/notification_templates.xml) — 8 events × 3 transports (email / web / in_app) | [Phase 1 C1](../roadmap/foundation/approval/phase-1-implementation.md#c1-register-approval-notification-template-fixtures) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| `approval.case` client-action absent in `src/frontend` — single-case detail view (drill from inbox or deep-link via `case_id`). Shares timeline + action-bar primitives with Enh 02; ships fresh, not a port. | 🔴 | [Approval Enhancement 03](../roadmap/foundation/approval/enhancements/03-approval-case-frontend2.md) |
| Bell click-through `action_url` for approval notifications lacks org slug — currently `/wc/approvals/case/{case_id}`, which the SPA router can't resolve to the new `approval-case` action without an `orgSlug`. Workaround: supported entry path is the Approvals app icon → inbox → card → case detail. | 🟡 | [Phase 2](../roadmap/foundation/approval/phase-2-implementation.md) |
| No bulk-decide endpoint or UI — approvers with many pending tasks must decide one-by-one. | 🟡 | [Phase 2](../roadmap/foundation/approval/phase-2-implementation.md) |
| No delegation auto-resolution — manual user-pick only; no OOO / calendar-aware auto-delegation. | 🟡 | [Phase 2](../roadmap/foundation/approval/phase-2-implementation.md) |
| No conditional steps — `flow.step.def` is sequential/parallel only; no "skip if amount < X" branch logic. | 🟡 | [Phase 2](../roadmap/foundation/approval/phase-2-implementation.md) |
| No analytics views — no avg-time-to-approve, % escalated, top bottleneck approver dashboards. | 🟢 | [Phase 3](../roadmap/foundation/approval/phase-3-implementation.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- **Don't build a domain-specific approval engine.** If your domain (procurement, expense, HR, finance, pricing) needs sign-offs, the answer is `ir.approval.case.request` + a flow template + a policy rule — never a `domain.X.approval` model. Re-implementing the state machine, decision ledger, or SLA worker inside a domain is the single biggest anti-pattern this module exists to prevent.
- **Subject-lock is your responsibility today.** Until the Phase 1 `subject_lock` helper lands, your domain must register its own `pre.{subject}.write` / `pre.{subject}.delete` lifecycle hooks to block edits while the related case is in PENDING / APPROVED. Don't wait for the helper — ship the hook now and migrate later.
- **Don't rely on `approval.case.approved` events firing yet.** The wiring is partial (only `SlaWorker` emits notification events reliably). Until Phase 1 closes the "no event published on case state transition" gap, your consuming domain must poll case state or hook into the decision ledger directly. Track this gap before designing your integration.
- **`input_data` is ad-hoc — pick a stable shape now.** No common rule-context schema exists yet. When you wire a new domain, agree on a `subject.*` / `requester.*` / `org.*` key layout up front so your AST rule expressions don't break when the schema is formalized in Phase 1.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
_none_ — no breaking changes shipped yet.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Permission | Granted to | Effect |
|---|---|---|
| Approval read access | All authenticated users | Read on `ir.approval.case` / `ir.approval.task` / `ir.approval.decision` / `ir.approval.event.log`; seeded via [data/ir.rbac.permission.csv](../src/ede/foundation/approval/data/ir.rbac.permission.csv). |
| `approval.recall` | Designated admin role | Permits `POST /api/approval/cases/{id}/recall` on APPROVED cases. Enforced by `_require_principal_user_id` + `principal.has_permission("approval.recall")` guard in `ApprovalService.recall_case`. |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [foundation.notifications](../roadmap/foundation/notifications/README.md) — Phase 1 hard prerequisite; approval consumes the notifications engine for bell + email + in-app delivery.
- [logistics.pricing](../roadmap/logistics/pricing/README.md) — first real consumer; see [Pricing Rate Approval (Phase 1)](../roadmap/logistics/pricing/phase-1/04-rate-approval.md) and [Approved-State Immutability](../roadmap/logistics/pricing/phase-1/01-rate-master.md) for the subject-lock example.
- [Foundation Model Naming](./foundation-model-naming.md) — `ir.*` vs `res.*` conventions used throughout this module.

### UX Design Artifacts
- [Approval Inbox — frozen mockup](../roadmap/foundation/approval/mockups/approval-inbox-frozen.html) ✅ Frozen 2026-06-05 — single source of truth for the `approval.inbox` client-action layout (Triage × Dashboard hybrid). Implementation must match. See [`roadmap/foundation/approval/mockups/README.md`](../roadmap/foundation/approval/mockups/README.md) for the full spec (theme, layout split, focus card anatomy, horizontal workflow chain + heartbeat animation, sidebar order, sub-component reuse notes for `<FocusPager>` / `<WorkflowChain>` / `.is-pulsing`).
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-06-06 (two new frontend2 client-action rebuild enhancements added to Known Gaps as 🔴 Not Started — **Enhancement 02 `approval.inbox` Client Action** at [`roadmap/foundation/approval/enhancements/02-approval-inbox-frontend2.md`](../roadmap/foundation/approval/enhancements/02-approval-inbox-frontend2.md) and **Enhancement 03 `approval.case` Client Action** at [`roadmap/foundation/approval/enhancements/03-approval-case-frontend2.md`](../roadmap/foundation/approval/enhancements/03-approval-case-frontend2.md). Both authored as part of the [legacy-frontend deprecation track](../roadmap/foundation/presentation/frontend-ui-revamp/legacy-deprecation.md) — the deprecation cutover deletes `src/frontend` without porting these UIs; they ship fresh in `frontend` afterward, not as cutover blockers. Enhancement 02 implements the [frozen mockup](../roadmap/foundation/approval/mockups/approval-inbox-frozen.html) (Triage × Dashboard hybrid). Enhancement 03 ships the single-case detail surface, sharing timeline + action-bar primitives with Enh 02. No backend changes — both consume the Phase 2 engine as-is. Previous sync: 2026-06-05 (UX design artifact added — **Approval Inbox layout FROZEN** at [`roadmap/foundation/approval/mockups/approval-inbox-frozen.html`](../roadmap/foundation/approval/mockups/approval-inbox-frozen.html). Triage × Dashboard hybrid: KPI strip + sticky focus card (with pager strip, chips, horizontal approval-chain workflow visualization, heartbeat animation on active node, header-right action group with Approve/Reject text buttons + Comment/Delegate icon buttons) + natural-flow right sidebar (Out today? → Your queue → Rules → Common delegates). Ocean-blue accent theme. Implementation must match — dev team picks this up when WS 6.1 (frontend-ui-revamp) builds the `approval.inbox` client-action atop the now-shipped Enhancement 08 SDK. Previous sync: 2026-05-31 (Enhancement 01 Team-Role Routing ✅ Delivered — moved from Known Gaps into Built Capabilities; added the 3 `approval.team_role.default_*` `ir.config` keys + the Team-Role Routing Defaults settings panel to Section 3; the `TEAM_ROLE` assignment kind resolves the record's-team role-holder via the shared `TeamRoleService`, with PRIMARY_ONLY/ALL_PARALLEL resolution, NEXT_IN_SEQUENCE/WALK_UP_HIERARCHY/NONE escalation, zero-assignee→immediate walk-up at create, `max_escalations` cap, supersede-not-delete + `escalation_depth`, save-time validators, and `step_assigned`/`escalated`/`stalled` audit rows; reconciled to the delivered schema — `approver_ref` holds the team-role code, Char `subject_model_key`/`subject_team_field` replace the spec's `ir.model`/`ir.model.field` References; Alembic `a1f4c9d27b03`; full guide docs/foundation-approval-team-role-routing.md. Prior sync: 2026-05-31 — flipped 🔴 → 🟡 In Progress. Prior sync: 2026-05-25 (Enhancement 01 Team-Role Routing added to Known Gaps — new `TEAM_ROLE` assignment kind on `ir.approval.flow.step` + `subject_model_id` (→ `ir.model`) and `subject_team_field_id` (→ `ir.model.field`) bindings on `ir.approval.flow.template` + sequence-aware escalation strategies `NEXT_IN_SEQUENCE` / `WALK_UP_HIERARCHY` / `NONE` + per-step `sla_hours` + `max_escalations` cap. Engine reads `record.team_id` via template-bound team-field and calls shared `TeamRoleService.resolve` (from `foundation.base` Enh 06 — hard prereq). Zero-assignee at case-create triggers immediate escalation rather than 24h-wait stall. Existing `ROLE` and `USER` kinds stay alive permanently — backward-compatible; no forced migration of existing pricing.rate flows. Sibling of `foundation.workflow` Enh 01 (same `TeamRoleService` consumer). First user-visible consumer is `logistics.sales-crm` Enh 09; eventual sales-crm Phase 2 · 02 Pricing Approval rewrite builds multi-step chains on top. Prior sync: 2026-05-10 — Phase 1 ✅ Delivered.) To refresh, invoke the syncing-roadmap-to-docs skill.*
