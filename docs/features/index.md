# Features

EDE ships a complete platform out of the box — every page in this section documents one capability you can use today. Pick a card to jump straight to the code.

## Identity & Access

| Feature | What it gives you |
|---|---|
| [Auth — Login & Sessions](auth.md) | JWT access + refresh tokens, the `/api/auth/*` endpoints, `JwtService`, `SessionService`. |
| [Security & Authorization](security.md) | Role + permission engine, active-organization context, company scope, allowed-org write guards. |
| [Record Rules](record-rules.md) | Per-record visibility filters declared as data, layered with the permission engine. |

## Data & Productivity

| Feature | What it gives you |
|---|---|
| [Base — Platform Substrate](base.md) | The `res.*` cross-domain masters: country, city, currency, partner, user, organization, UOM, language. |
| [Connectors](connectors.md) | One contract for external systems — SMTP, S3, Google Drive, Gmail OAuth2, custom. Used by Email + Storage. |
| [Customization (Properties)](customization.md) | User-defined fields on any record without writing code or migrations. |
| [Communication (Chatter)](communication.md) | Per-record message thread, activity timeline, mentions; embeds on any model with one mixin. |
| [Notifications](notifications.md) | In-app + email push, opt-in registry, per-channel routing. |
| [Email (Outbound Queue)](email.md) | Queued send with retry, templates, transport plug-board (SMTP / SendGrid / Mailgun / Gmail OAuth2). |
| [Document Storage](storage.md) | File upload + versioned documents + pluggable backends (local FS shipped, Google Drive via connector). |
| [Datasets & Metrics](dataset.md) | Declarative `metric` + `dataset` definitions, formula engine, caching. |

## Workflow & Automation

| Feature | What it gives you |
|---|---|
| [Approval Workflows](approval.md) | Multi-step approval routing on any record, configurable per company / per model. |
| [Workflow Engine](workflow.md) | Declarative state machines + automated transitions, with the workflow DSL. |
| [QA Automation](qa-automation.md) | Playwright-driven browser e2e + use-case recording + replay against demo tenants. |

## Presentation & Localization

| Feature | What it gives you |
|---|---|
| [Presentation (View DSL)](presentation.md) | XML declarative list / form / kanban / search views rendered by the React web client. |
| [Internationalization](i18n.md) | UI string catalogues + record translations, per-user language. |

## Multi-Tenancy

| Feature | What it gives you |
|---|---|
| [Multi-Tenant Gateway](gateway.md) | SaaS control plane: per-tenant DB provisioning, Traefik routing, admin SPA. |

---

Don't see what you need? The full architecture reference is in the [Architecture](../01-overview.md) section, and the [Tutorial](../tutorial/00-installation.md) covers everything you need to build your first feature on top of the framework.
