# EDE Framework — Documentation Index

**EDE (Enterprise Digital Engine)** — a Domain-Driven Design platform kernel for Python.

This documentation covers the entire backend codebase: architecture, conventions,
implementation details, and developer guides.

---

## Contents

| # | Document | Description |
|---|---|---|
| 01 | [Overview](01-overview.md) | What EDE is, five-layer model, request/event flow, tech stack |
| 02 | [App Structure & Loading](02-app-structure.md) | Source tree, app layout, manifest, boot sequence, Registry, Env |
| 03 | [Domain Model & Fields](03-domain-model.md) | DomainModel, auto-fields, field types, computed fields, FieldSpec |
| 04 | [Command & Event Bus](04-command-event-bus.md) | Command dispatch, CommandBus, Events, EventQueue, EventWorker, retries, field change tracking |
| 05 | [ORM Layer](05-orm-layer.md) | ModelProxy, RecordSet, relational commands, transactions |
| 06 | [Persistence](06-persistence.md) | Providers, SQLAlchemy adapter, schema specs, domain filter DSL |
| 07 | [HTTP Layer](07-http-layer.md) | RouteController, route decorators, FastApiHttpAdapter, middleware, Web Push (SSE) |
| 08 | [Authentication](08-authentication.md) | JWT, sessions (ir.session), SessionService, AuthMiddleware |
| 09 | [Tenancy](09-tenancy.md) | Per-tenant databases, Env cloning, tenant resolution, thread context |
| 10 | [Presentation DSL](10-presentation-dsl.md) | View XML format, DslParser, ViewRegistry, RenderPlan |
| 11 | [Foundation Apps](11-foundation-apps.md) | base, auth, presentation — built-in models, routes, conventions |
| 12 | [Migrations](12-migrations.md) | Alembic per-app version locations, multi-tenant upgrade |
| 13 | [Developer Guide](13-dev-guide.md) | Step-by-step: adding domains, apps, models, commands, routes |
| 14 | [Lifecycle Hooks](14-lifecycle-hooks.md) | `pre.*` / `post.*` hooks, hook key derivation, `cmd.record` semantics, field tracking hooks, examples |
| 15 | [Command & Event Usage Guide](15-command-event-guide.md) | When to use `on_command`, `on_event`, `on_event(track_fields)`, and `web.client.*` push events |
| 16 | [Connector Framework](16-connector-framework.md) | Service-agnostic connector architecture: `ConnectorProvider` ABC, `ConnectorRegistry`, `ir.connector` model, `ConnectorService`, adding new providers and categories |
| 17 | [Storage Module](17-storage-module.md) | `foundation.storage`: `StorageBackend` ABC, `LocalFilesystemBackend`, `DocumentService`, `KeyBuilder`, `StorageRouter`, document + version models, upload/download API |

---

## Module Implementation Docs (auto-maintained from roadmap)

These reflect the *current built state* of each module — what is shipped, what is partial, what gaps remain, what configuration each introduces. They are auto-maintained by the `syncing-roadmap-to-docs` skill from the corresponding roadmap files.

### Platform-Wide SDKs (read these BEFORE touching a base model)

Before adding fields to or patching views on a model your module doesn't own:

- **[Model & View Extension SDK](foundation-base-extensions.md)** — `@api.extend_model("target.key")` decorator for cross-module field additions + `<extend ref="<prefix>.<view_id>">` for view inheritance. Shipped in `foundation.base` Phase 2 on 2026-05-18. Includes soft-FK degradation so test fixtures that register only the base model continue to pass. **Required for** consuming modules like `foundation.assistant`, `foundation.l10n`, `foundation.marketplace`, future country / tenant packs.
- **[Module Integration Pattern](module-integration-pattern.md)** — the *pattern* layered on the SDK: how a producer module (e.g. `logistics.sales_crm`) and its downstream consumers (`logistics.booking`, future `logistics.shipments`) integrate through a boundary object (the `crm.handover` Y-fork) **without a middle integration module**. Producer owns the contract + neutral routing slot; each consumer owns its own FK + command + button; back-channel is events, never a reverse import. Covers the three invariants that keep it acyclic and when a bridge module *is* warranted.
- **[Platform Implementation Rules](../roadmap/platform/00-execution-rules.md)** — Rule 1's "Cross-App Field & View Additions" sub-section formalises the SDK as non-negotiable for cross-module schema changes. Editing a base module's source to add a downstream module's field is rejected at code-review time.

### Foundation engines

| Doc | Module | Roadmap |
|---|---|---|
| [AI Capability Layer](foundation-ai.md) | `foundation.ai` (Phases 1 / 2 / 4 ✅ Delivered + Enhancement 01 ✅ — provider abstraction (Anthropic + OpenAI) + `@api.ai_tool` decorator (read-only + opt-in write-mode) + tool/prompt registries + `ai.conversation` primitives + function-calling bridge with draft→preview→confirm flow for writes + cost/audit + per-user quotas + rate limits + PII redaction + prompt-injection detector + output validators + data-residency + provider routing + cost-threshold alerts + write-mode approval-engine integration + undo window + 8 HTTP routes + 19 RBAC perms + **standalone "AI" app in the app-switcher** + 23 DSL views + `read_schema` reference adopter; **Phase 3 🔴 ON HOLD pending `foundation.jobs` Phase 1**; HARD prereq for `foundation.assistant` + `foundation.mcp`) | [roadmap/foundation/ai/](../roadmap/foundation/ai/README.md) |
| [AI Assistant (Chat Companion to Actions)](foundation-assistant.md) | `foundation.assistant` (Phase 1 ✅ Delivered 2026-05-17 end-to-end — all 12 WS shipped: 4 models + Alembic migration + 5 tools on `assistant.tool` carrier + `AssistantTurnService` + 5 HTTP routes + 4 settings + RBAC + AssistantPanel React side-slide + reducer dispatcher `applyAssistantOps` in `WorkspaceActionController` + ✨ provenance marker on `ControlPanelSearchView` chips/groupby pills + `SMOKE_TEST.md` manual gate; 48 pytest + 26 vitest green; Enhancements 01 ✅ Org-Level Default Provider + 02 ✅ Streaming + Per-Org Preferences + Chat UX Polish + 03 ✅ Action-Anchored Session History + Phase 2 ✅ Full Read-Only Surface + Action Buttons + Skill Packs (all 7 WS — 5 new proposer tools, ActionButton wiring, 3 seeded skill packs, AssistantInsightPreview, assistant.user_preference model; 2264 pytest + 556 vitest total); Phases 3 / 4 🔴 — configurations & properties assistant in Phase 3; gated write-mode + domain skill packs in Phase 4) | [roadmap/foundation/assistant/](../roadmap/foundation/assistant/README.md) |
| [Approval Workflow Engine](foundation-approval.md) | `foundation.approval` (Phase 1 ✅) | [roadmap/foundation/approval/](../roadmap/foundation/approval/README.md) |
| [Auth (JWT + Session + Middleware)](foundation-auth.md) | `foundation.auth` (Phase 1 ✅ Delivered baseline — pre-roadmap) | [roadmap/foundation/auth/](../roadmap/foundation/auth/README.md) |
| [Background Jobs Engine](foundation-jobs.md) | `foundation.jobs` (**Phases 1+2 ✅ Delivered 2026-05-19** — `ir.job` / `ir.job.run` / `ir.job.lock` + Celery executor + cron scheduler + `@api.scheduled_job` + retry / dead-letter + `env.job_progress` + multi-worker distributed locking + Prometheus metrics + stuck-job reaper + `ir.job.requires` dependency graph + admin UI; **Phase 3 ✅ Delivered 2026-05-28** — approval `SlaWorker` retired onto the `approval.sla_tick` `@api.scheduled_job` (WS-J20 + J22 + J23; WS-J21 notifications-webhook deferred — no webhook surface yet); **Phase 4 🔴** — gateway `GatewaySaasWorker` retirement, split out because provisioning is destructive, gated on the Postgres multi-worker contention proof; HARD prereq for the `foundation.servora_sync` Servora pull worker) | [roadmap/foundation/jobs/](../roadmap/foundation/jobs/README.md) |
| [Base (Platform Substrate)](foundation-base.md) | `foundation.base` (Phase 1 ✅ Delivered baseline — `res.*` cross-domain masters + `ir.*` platform metadata + health-check; root of the foundation dependency graph. **Phase 2 — Model & View Extension SDK ✅ Delivered 2026-05-18** — `@api.extend_model` decorator + `<extend ref="...">` view inheritance + soft-FK metadata builder + boot validator + `ir.model.extension` registry mirror + admin UI + [developer guide](foundation-base-extensions.md) + 40 pytest cases. Hard prereq for `foundation.l10n` Phase 1, `foundation.marketplace` Phase 1, and the parked `foundation.assistant` org-level default-provider — all three now unblocked. — `@api.extend_model` decorator + view inheritance + registry merge + boot validator + `ir.model.extension` mirror; net-new kernel-level substrate consumed by `foundation.l10n` / `foundation.assistant` / `foundation.customization` / `foundation.marketplace` / every future module that needs to extend a base model.) | [roadmap/foundation/base/](../roadmap/foundation/base/README.md) |
| [BI Dashboard Engine](foundation-dashboard.md) | `foundation.dashboard` (all 3 phases 🔴 — KPIs + widgets + rule-based insight surface; AI is future scope under internal MCP server) | [roadmap/foundation/dashboard/](../roadmap/foundation/dashboard/README.md) |
| [Communication (Record-Level Chatter Engine)](foundation-communication.md) | `foundation.communication` (Phase 1 ✅ Delivered 2026-05-13 — `CommunicationService` (11 methods routed via `Command`) + `/api/chatter/*` REST (10 endpoints, RBAC via `AuthorizationService.can`) + 7 React chatter components + `<activity>` DSL with strict-reject parser + `Chatterable` mixin (`res.partner` extends it; `crm.lead`/`crm.opportunity`/`crm.handover` also adopters) + message immutability hooks + activity→timeline auto-post; **+26 new pytest cases; 1808 pytest + 494 vitest green**; unblocks `logistics.equipment-operations` Phase 1. Phases 2 (identity centralization + notifications/email bridges + auto-follow + mentions) and 3 (attachments + rich text + threads + mail-in + SLA + scheduled) 🔴) | [roadmap/foundation/communication/](../roadmap/foundation/communication/README.md) |
| [Connectors Framework](foundation-connectors.md) | `foundation.connectors` (✅ Delivered — baseline / pre-roadmap — service-agnostic connector framework: `ir.connector` + `ir.connector.param` models, lifecycle verbs (`test_connection`/`activate`/`deactivate`/`import_config`/`get_config`), `ConnectorService` coordination layer, `/api/connectors/*` admin HTTP API, Settings → Integrations → Connectors UI, single-default-per-(category, org) invariant; consumed by `foundation.email` and `foundation.storage` — connector *kinds* (SMTP/S3/GCS/Drive/…) register from their consumer module against the kernel `connector_registry`, never inside this module) | [roadmap/foundation/connectors/](../roadmap/foundation/connectors/README.md) |
| [Conversation Orchestrator (Converse)](foundation-converse.md) | `foundation.converse` (Phase 1 🔴 + Phase 2 🔴 — drafted 2026-05-26 — AI-driven dialog orchestrator that turns inbound messages on a `messaging.thread` into intent classification + slot-fill ask-back loop + action graph (`compose_reply`/`dispatch_command`/`workflow_transition`/`request_approval`/`handoff_to_human`); flow-driven and declarative via `converse.flow` XML rows; sibling-not-stacked with `foundation.assistant` (different lifecycle: external-party + asynchronous + write-mode vs assistant's in-app + synchronous + read-only). Hard prereqs: `foundation.messaging` Phase 1 + `foundation.ai` Phase 4 (with `provenance-only` write-mode open question resolved). First in-tree consumer use case: [uc.freight-quote-via-telegram](../roadmap/usecases/freight-quote-via-telegram.md)) | [roadmap/foundation/converse/](../roadmap/foundation/converse/README.md) |
| [Country Localization SDK (l10n)](foundation-l10n.md) | `foundation.l10n` (all 3 phases 🔴 — **restructured 2026-05-17**: the generic `@api.extend_model` decorator + view inheritance moved to `foundation.base` Phase 2; l10n now layers `country_scope("CC")` predicate + per-org activation on top. Phase 1 narrowed to country-scope predicate + `res.organization.localization_pack_ids` + test-fixture pack; Phase 2 ships `localizations.in` HSN/GST/PAN/GSTIN; Phase 3 adds US/EU/AE packs + reporting/document integration. Hard prereq: `foundation.base` Phase 2.) | [roadmap/foundation/l10n/](../roadmap/foundation/l10n/README.md) |
| [Customization (Properties + Persistent Model Registry)](foundation-customization.md) | `foundation.customization` (Phase 1 ✅) | [roadmap/foundation/customization/](../roadmap/foundation/customization/README.md) |
| [Dataset & Metric Engine](foundation-dataset.md) | `foundation.dataset` (Phase 1 ✅ Delivered 2026-05-13 — substrate + Blueprint admin form + HTTP + RBAC + demo; Phase 2 ✅ Delivered 2026-05-13 — plan engine + formula engine + DAG cycle detection + shared safe AST evaluator + per-run cache + 3 new demo metrics; Phase 3 🔴) | [roadmap/foundation/dataset/](../roadmap/foundation/dataset/README.md) |
| [Document Report Engine (DRE)](foundation-document.md) | `foundation.document` (Phase 1 ✅ Delivered 2026-05-27 · Phase 2 🔴 · Phase 3 🔴 · Enhancement 01 Attachment Substrate ✅ Delivered 2026-05-27 — DML v1.1 parser + two-stage pipeline (Stage 1: DML → resolved `.xml`; Stage 2: independent per-format engines) + pixel-perfect PDF via ReportLab Canvas (Phase 1) + HTML + DOCX engines (Phase 2; **no print — out of scope**) + records-as-data inheritance (`parent_id` + `inherit_mode` + `domain`) + `ir.document.type` taxonomy + React drag-drop designer (Phase 3); implements platform `ReportEngine` Protocol — DRE is one of N registered engines via `ir.action.report` + `ede.report.run`) | [roadmap/foundation/document/](../roadmap/foundation/document/README.md) |
| [Email (Outbound Queue + Templates + Transport Plug-board)](foundation-email.md) | `foundation.email` (✅ Delivered baseline — pre-roadmap — `mail.outbox` + `mail.template` + `EmailRouter` queue drain + connector-based transports (Gmail OAuth2 today; SMTP / SendGrid / Mailgun via same contract) + `/api/email/*` REST + Email app in app-switcher) | [roadmap/foundation/email/](../roadmap/foundation/email/README.md) |
| [Extension Marketplace](foundation-marketplace.md) | `foundation.marketplace` (all 4 phases 🔴 — third-party extension lifecycle; **HARD prereq on `foundation.l10n` Phase 1** for shared `ScopeEvaluator`; curated marketplace + metadata activation, no code download at click-time; Phase 4 onboards first partner AEB Customs end-to-end) | [roadmap/foundation/marketplace/](../roadmap/foundation/marketplace/README.md) |
| [Gateway (Multi-Tenant SaaS Control Plane)](foundation-gateway.md) | `foundation.gateway` (✅ Delivered baseline — pre-roadmap; `gateway.tenant` + `gateway.shared_pool` lifecycle + `GatewaySaasWorker` polling provisioner + `TraefikRoutePublisher` (Redis push) + per-tenant `MigrationRunner` + dual-port `ede serve gateway` (`:8000` app, `:8001` admin SPA at `/wc/saas-manager/**`) + `/api/gateway/{tenants,pools,traefik,public}` admin API + dedicated SaaS-manager React build variant via `bun run build:gateway`. NOT in default `ACTIVE_MODULES`. Known gaps: per-tenant quotas, billing hooks, DB backup, SSO federation 🟠) | [roadmap/foundation/gateway/](../roadmap/foundation/gateway/README.md) |
| [Internationalization (i18n)](foundation-i18n.md) | `foundation.i18n` (Phase 1 ✅) | [roadmap/foundation/i18n/](../roadmap/foundation/i18n/README.md) |
| [Import / Export Engine](foundation-import-export.md) | `foundation.import_export` (all 3 phases 🔴 — drafted 2026-05-27 — template-driven bulk file engine: 3 authoring modes (`@api.io_template` decorator + `<record model="ir.io.template">` XML + admin UI) coexist on one schema; Phase 1 = Excel + CSV parse → validate → preview → commit pipeline with 8 built-in validators (`required`/`regex`/`range`/`enum`/`iso_4217_currency`/`lookup`/`date`/`formula`) + auto-injected Import button on ListView/KanbanView + `<button special="io_import">` DSL hook + first adopter `pricing.rate.fcl.upload` re-scoping `logistics.pricing` Phase 1 Feature 05; Phase 2 = async via `foundation.jobs` + per-row approval mode + export direction + Download Blank Template button; Phase 3 = multi-sheet templates + scheduled SFTP/Drive sources + email-attachment imports + per-row inline edit + run rollback + UPDATE diff preview + template versioning + JSON cross-tenant export + retention sweeper) | [roadmap/foundation/import_export/](../roadmap/foundation/import_export/README.md) |
| [MCP (Model Context Protocol Server)](foundation-mcp.md) | `foundation.mcp` (all 5 phases 🔴 — publishes `foundation.ai`'s tool/prompt/conversation registries over MCP to external AI clients (Claude Desktop / IDE plugins / third-party agents); HTTP-SSE transport in Phase 1, stdio in Phase 3, gated write-mode in Phase 4, capability negotiation in Phase 5; sibling of `foundation.assistant`, not stacked) | [roadmap/foundation/mcp/](../roadmap/foundation/mcp/README.md) |
| [Messaging Channel Engine](foundation-messaging.md) | `foundation.messaging` (Phase 1 🔴 — drafted 2026-05-26 — provider-agnostic two-way conversation transport over messaging apps; Telegram first (Phase 1), WhatsApp Cloud + Messenger + Twilio SMS + Webchat in later phases; mirrors `foundation.email` in shape (connector plug-board + provider contract) but for stateful threaded conversations; owns inbound webhook + outbound send + threading + `(channel_kind, external_id) → res.partner` identity resolution + chatter mirror; hard prereq for `foundation.converse`. First in-tree consumer use case: [uc.freight-quote-via-telegram](../roadmap/usecases/freight-quote-via-telegram.md)) | [roadmap/foundation/messaging/](../roadmap/foundation/messaging/README.md) |
| [Notifications Engine](foundation-notifications.md) | `foundation.notifications` (Phase 1 ✅) | [roadmap/foundation/notifications/](../roadmap/foundation/notifications/README.md) |
| [Presentation (View DSL & Web Client)](foundation-presentation.md) | `foundation.presentation` (Phase 1 ✅ — KanbanView shipped 2026-05-11; ListView/FormView/SearchView already shipped) | [roadmap/foundation/presentation/](../roadmap/foundation/presentation/README.md) |
| [QA Automation Engine](foundation-qa-automation.md) | `foundation.qa-automation` (Phases 1, 2 & 3 ✅ Delivered + Enhancement 01 ✅ 2026-05-13 — Playwright plumbing + 9 foundation-primitive e2e tests + `qa.usecase` + `qa.usecase.step` models + `ede e2e record` CLI w/ 23-case scaffold-transformer unit suite + `seed_deterministic` primitive for byte-stable test fixtures (**12 e2e passed in 76.9s**); Phases 4–8 🔴) | [roadmap/foundation/qa-automation/](../roadmap/foundation/qa-automation/README.md) |
| [Reporting Engine](foundation-reporting.md) | `foundation.reporting` (all 3 phases 🔴 — statement-mode grid with per-line `date_scope`; Phase 2 live + write-back + pivot; Phase 3 sharing + snapshots) | [roadmap/foundation/reporting/](../roadmap/foundation/reporting/README.md) |
| [Storage (Document Store + Pluggable Backends)](foundation-storage.md) | `foundation.storage` (✅ Delivered — baseline, pre-roadmap) — `storage.folder` + `storage.document` + `storage.document_version` + `StorageRouter` (local FS default; cloud via `foundation.connectors` — Google Drive shipped) + HTTP API + RBAC seed + Documents app | [roadmap/foundation/storage/](../roadmap/foundation/storage/README.md) |
| [Security & Authorization (Active-Org + Company-Scope + Record Rules)](foundation-security.md) | `foundation.security` ✅ **All 5 phases Delivered 2026-05-13** — Phase 1 active-organization JWT/session/principal propagation + `Env.active_organization_id` slot + 5 new `$principal.*` ABAC variables + `POST /api/auth/switch-organization` + frontend wiring; Phase 2 `@api.model(company_scope="strict\|optional\|multi")` decorator opt-in + post-init field auto-injection + `apply_company_filter` at all 8 ORM read callsites + `env.with_company_test` clone + `register_company_scope_hooks` pre-create stamping + `ir.model.company_scope` registry column + `res.partner` proof-of-life; Phase 3 allowed-org write guard (pre-create + pre-update hooks) + `res.organization.reassign` permission + 3 standardized `PermissionDeniedError.reason` codes; Phase 4 `ir.rbac.decision.log.{active,target}_organization_id` + `ir.rbac.binding.change.log.scope_organization_id` columns + indexes + admin views with "Cross-Org Attempts" saved search + `SECURITY_DECISION_LOG_RETAIN_DAYS` setting; Phase 5 `ir.rbac.record.rule` record-rule engine + `RecordRuleEngine` composing `GLOBAL AND ((ROLE_A_OR_BLOCK) OR (ROLE_B_OR_BLOCK))` + `apply_record_rules_filter` at all 8 callsites + `AuthorizationService._enforce_record_rules` per-record gate (`record_rule_violation` reason) + admin UI. Developer guides: [company_scope](foundation-company-scope.md) · [record rules](foundation-record-rules.md) | [roadmap/foundation/security/](../roadmap/foundation/security/README.md) |
| [Workflow Engine](foundation-workflow.md) | `foundation.workflow` (Phase 1 ✅, Phase 2 ✅, Phase 3 🔴) | [roadmap/foundation/workflow/](../roadmap/foundation/workflow/README.md) |

### Domain modules

Domain docs co-locate with the domain — they live under `src/domains/<domain>/docs/`, not here:

| Doc | Module | Roadmap |
|---|---|---|
| [Rate Management System (RMS)](../src/domains/logistics/docs/pricing.md) | `pricing.*` (logistics) | [roadmap/logistics/pricing/](../roadmap/logistics/pricing/README.md) |
| [Sales & CRM](../src/domains/logistics/docs/sales-crm.md) | `crm.*` (logistics, planned) | [roadmap/logistics/sales-crm/](../roadmap/logistics/sales-crm/README.md) |
| [Servora Integration — External Reference-Data Platform](../src/domains/onemaster/docs/onemaster.md) | `foundation.servora_sync` (the in-house OneMaster hub is **retired**; reference master data — geography, ports, airports, countries, currencies, FX — is now served by **Servora**, a separate self-hosted platform in its own repo. EDE integrates as a **consumer** over HTTPS via `foundation.servora_sync`; all 3 integration phases 🔴 Not Started — geography path unblocked today, financial path waits on Servora's FX connector) | [roadmap/onemaster/](../roadmap/onemaster/README.md) |

> Roadmap is the source of truth. Status, configuration, and built-vs-planned reflected in these docs are mirrored from the roadmap module READMEs and feature files. To refresh, edit the roadmap and invoke the `syncing-roadmap-to-docs` skill.

### Platform changes

Status-bearing platform-wide changes (kernel/ORM/cross-cutting) are mirrored from `roadmap/platform/<NN>-<topic>.md` into `docs/platform-<NN>-<topic>.md`.

| Doc | Change | Roadmap |
|---|---|---|
| [ORM Soft-Archive Auto-Filter](platform-01-orm-active-filter.md) | `active=False` rows hidden by default for any model declaring an `active` Boolean field; frontend "Archived" toggle | [roadmap/platform/01-orm-active-filter.md](../roadmap/platform/01-orm-active-filter.md) |
| [Computed Field Runtime](platform-02-compute-field-runtime.md) ✅ | Kernel runtime for `fields.Field(method=..., depends_on=..., store=...)` — synchronous-in-UoW recompute on dependency writes. Phase 1: same-record + O2M deps, transitive cascading, cycle detection. Phase 2: `pricing.rate`'s 4 hook-driven fields retrofitted. Phase 3: M2O dotted paths + unstored (`store=False`) computes evaluated on `RecordSet.read()`. | [roadmap/platform/02-compute-field-runtime.md](../roadmap/platform/02-compute-field-runtime.md) |
| [Demo Data Loader](platform-03-demo-data-loader.md) ✅ | `ede migrate upgrade --with-demo=all|<app_keys>` loads each app's `demo/*.xml` after the main data load. New `demo:` manifest key; new `is_demo` Boolean on `ir.data.reference` so demo rows are excluded from orphan cleanup and (future) purgeable independently of production seeds. Opt-in per run, per app. Delivered 2026-05-12 — 1531 pytest + 445 vitest green; 19 new pytest tests. | [roadmap/platform/03-demo-data-loader.md](../roadmap/platform/03-demo-data-loader.md) |
| [Demo Data Rollout — Foundation](platform-04-demo-data-foundation-rollout.md) 🔴 | Retrofits the demo channel onto delivered foundation modules (`base`, `i18n`, `customization`, `approval`, `communication`, `notifications`) so `--with-demo=all` produces a coherent demo tenant matching the unifying scenario in `docs/demo-usecase/`. ~7 demo XML files + manifest `demo: [...]` additions; no platform code. Pre-req for the logistics demo rollout. | [roadmap/platform/04-demo-data-foundation-rollout.md](../roadmap/platform/04-demo-data-foundation-rollout.md) |
| [Engine Substrate](platform-05-engine-substrate.md) 🔴 | New top-level kernel directory `src/ede/core/engines/` at peer level with `kernel/orm/bus/services/` for renderer-agnostic computational engines (dataset compiler, metric registry, formula evaluator, chip / period kernel, integration spine, report / document / dashboard engines). Plus decorator-surface additions (`@api.metric`, `@api.chip`, `@api.kpi`) to `src/ede/core/api.py`. Each phase lands alongside its consumer foundation module wave (W1-W7). | [roadmap/platform/05-engine-substrate.md](../roadmap/platform/05-engine-substrate.md) |
| [Universal Result JSON Contract](platform-06-universal-json-contract.md) 🔴 | Single canonical JSON contract — `DatasetResult` / `ReportResult` / `WidgetResult` / `KpiResult` `TypedDict` family with type-aware `schema[].kind` column descriptors — that every reporting / analytics / KPI / export engine produces and every consumer (React grid, document `<rows datasource>`, HTTP / SSE / WS / webhook / export, future MCP) reads. Lives in `src/ede/core/engines/integration/contract.py`. Versioned via `meta.contract_version`; backwards-compatible additions don't bump. | [roadmap/platform/06-universal-json-contract.md](../roadmap/platform/06-universal-json-contract.md) |
| [Shared Safe AST Evaluator](platform-07-shared-safe-ast-evaluator.md) 🔴 | AST-restricted evaluator at `src/ede/core/engines/formula/safe_eval.py` with closed node-type whitelist + closed function whitelist. Two consumers, one implementation: metric formula engine (`function_set="numeric"`) and DML `<var formula="...">` engine (`function_set="full"` — adds string + date + format functions). One inventory of allowed inputs, one place to harden when a Python AST CVE drops. | [roadmap/platform/07-shared-safe-ast-evaluator.md](../roadmap/platform/07-shared-safe-ast-evaluator.md) |
| [Active-Organization Context + `company_scope` ORM Opt-In](platform-08-active-organization-and-company-scope.md) 🔴 | Kernel-level cross-cutting design for `foundation.security` Phases 1–4: `Env.active_organization_id` + `with_active_organization_id` clone; `@api.model(company_scope="strict\|optional\|multi"\|None)` decorator with auto-injection of `organization_id` (strict/optional) or `organization_ids` M2M (multi); `apply_company_filter` helper alongside `apply_active_filter`; `register_company_scope_hooks` registrar mirroring `register_authorization_hooks`; 5 new `$principal.*` keys (`active_organization_id`, `allowed_organization_ids`, split `org_ids` / `branch_ids` / `department_ids`); audit-log column additions on `ir.rbac.decision.log` + `ir.rbac.binding.change.log`. Delivery sequenced through `foundation.security` W1–W4. | [roadmap/platform/08-active-organization-and-company-scope.md](../roadmap/platform/08-active-organization-and-company-scope.md) |
| [Monetary Field Type](platform-09-monetary-field-type.md) ✅ | New kernel `fields.Monetary(currency_field=...)` — subclass of `Decimal` keeping `field_type="decimal"` (no amount-column migration). `currency_field` defaults to `currency_id`; absent → plain decimal (explicit-but-missing fails fast at boot). Auto-applies the `monetary` widget via a `default_widget` field-spec hint; bare `<field/>` renders as money, explicit `widget=` still overrides. Output is **currency-driven**: `res.currency` gains `symbol_position` / `decimal_separator` / `thousand_separator` (additive migration; `symbol` + `decimal_precision` reused) and `MonetaryField.tsx` formats per the linked currency row. Spans `ede.core.kernel` + `foundation.base` + `foundation.presentation`. | [roadmap/platform/09-monetary-field-type.md](../roadmap/platform/09-monetary-field-type.md) |
| [`@api.autovacuum` + Transient Models](platform-11-autovacuum-and-transient-models.md) ✅ | Kernel `@api.autovacuum` cleanup decorator + registry collection + `__ede_transient__` model-kind (auto TTL vacuum via `created_at_utc`), swept daily by a config-defined `ir.autovacuum.gc` `ir.job` — plain target in `foundation.base`, XML job (not `@api.scheduled_job`) because base imports during `ede.core.api`'s mid-load. First consumer: `foundation.base` Enh 10 (admin password-reset wizard). | [roadmap/platform/11-autovacuum-and-transient-models.md](../roadmap/platform/11-autovacuum-and-transient-models.md) |

### Platform-wide rules

Files under [roadmap/platform/](../roadmap/platform/) prefixed `00-` (e.g. [00-execution-rules.md](../roadmap/platform/00-execution-rules.md)) are *rules* documents, not module roadmaps — they have no status / phases / built table. They are read directly from `roadmap/platform/` and are deliberately not mirrored into `docs/`. The cross-module status index lives at [roadmap/roadmap-tracker.md](../roadmap/roadmap-tracker.md).

---

## Quick Reference

### Adding a New Feature Checklist

```
1. Domain settings   → src/domains/settings.py          ACTIVE_DOMAINS = [...]
2. App settings      → src/domains/<domain>/settings.py  ACTIVE_MODULES = [...]
3. App layout        → src/domains/<domain>/<app>/
                         __manifest__.py
                         __init__.py
                         models/__init__.py
                         models/<model>.py
                         api/controllers.py
                         views/<model>.xml
                         migrations/versions/
4. Model             → @api.model("domain.name") class MyModel(DomainModel)
5. Commands          → @api.on_command("app.verb") def handler(self, cmd): ...
6. Hooks (optional)  → @api.on_hook("pre.app.verb") def guard(self, cmd): ...
7. Events            → @api.on_event("app.past") def handler(event, env): ...
8. Controller        → @api.route_config(prefix=...) class MyController(RouteController)
9. Views             → XML files declared in __manifest__ "data" key
10. Migration        → ede migrate generate -m "description"
11. Apply            → ede migrate upgrade -t <tenant>
```

### Key Naming Rules

| Thing | Rule | Example |
|---|---|---|
| App key | `{domain_type}.{folder}` (auto-derived) | `logistics.shipment` |
| Model key | `{domain_type}.{name}` | `res.user`, `logistics.shipment` |
| Table name | model key with `.` → `_` | `logistics_shipment` |
| Command | `{app_key}.{verb}` | `shipment.create` |
| Event | `{app_key}.{past_tense}` | `shipment.confirmed` |
| View ID | dotted, maps to filename | `logistics.shipment.list` → `logistics_shipment.xml` |
| Manifest author | Always `"THE_BLACK_BOX"` | — |

### Key File Locations

| What | Where |
|---|---|
| Public API decorators | `src/ede/core/api.py` |
| Runtime context (Env) | `src/ede/core/env.py` |
| Registry | `src/ede/core/registry.py` |
| DomainModel base | `src/ede/core/kernel/model.py` |
| Field descriptors | `src/ede/core/kernel/fields.py` |
| CommandBus (+ lifecycle hooks) | `src/ede/core/bus/command_bus.py` |
| EventWorker | `src/ede/core/bus/worker.py` |
| RecordSet | `src/ede/core/orm/recordset.py` |
| ModelProxy | `src/ede/core/orm/model_proxy.py` |
| AuthMiddleware | `src/ede/core/adapters/http/fastapi/auth_middleware.py` |
| JwtService | `src/ede/core/services/auth/jwt_service.py` |
| WebPushRegistry (SSE) | `src/ede/core/services/push/web_push_registry.py` |
| Foundation settings | `src/ede/foundation/settings.py` |
| Domain settings | `src/domains/settings.py` |
| ConnectorProvider ABC | `src/ede/core/connectors/interfaces.py` |
| ConnectorRegistry (singleton) | `src/ede/core/connectors/registry.py` |
| ir.connector model | `src/ede/foundation/connectors/models/connector.py` |
| ir.connector.param model | `src/ede/foundation/connectors/models/connector_param.py` |
| StorageBackend ABC | `src/ede/core/services/storage/interfaces.py` |
| LocalFilesystemBackend | `src/ede/core/adapters/storage/local_fs.py` |
| DocumentService | `src/ede/core/services/storage/document_service.py` |
| KeyBuilder | `src/ede/core/services/storage/key_builder.py` |
| StorageRouter | `src/ede/foundation/storage/services/storage_router.py` |
| ConnectorService | `src/ede/foundation/connectors/services/connector_service.py` |

### CLI

```bash
ede serve                          # start server
ede serve --with-worker            # server + event worker (dev)
ede worker                         # standalone event worker
ede migrate generate -m "msg"      # generate migrations
ede migrate upgrade -t <tenant>    # upgrade tenant DB
ede info                           # show loaded apps
```

---

## Architecture Principles

1. **Dependencies flow downward only** — Domain never imports from HTTP or Persistence.
2. **Explicit activation** — no auto-discovery; every app must be listed in settings.
3. **Protocol-driven infrastructure** — persistence, events, and auth are contracts, not
   implementations. Swap implementations without touching domain code.
4. **Immutable Env propagation** — per-request context is a shallow clone of the base Env;
   no global mutable state.
5. **Per-tenant isolation** — each tenant has its own database; no shared state between tenants.
6. **Command → Event flow** — mutations are synchronous commands; notifications are
   asynchronous events with retry.
7. **Registry as single source of truth** — all models, commands, events, and providers are
   indexed in the Registry at boot; nothing discovered at runtime.
