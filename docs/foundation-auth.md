<!-- AUTO-MAINTAINED-BY: syncing-roadmap-to-docs -->
# Foundation Auth — Implementation Docs

**Module:** `foundation.auth` (`src/ede/foundation/auth/`)
**Roadmap:** [roadmap/foundation/auth/README.md](../roadmap/foundation/auth/README.md)
**Status:** ✅ Delivered (baseline — pre-roadmap)
**Layer:** Foundation engine — platform substrate

> Source of truth is the roadmap. This doc reflects the *current built state* — what is shipped, what is partial, what gaps remain, what configuration it introduces, and how a developer or end user interacts with it. Auto-maintained by the `syncing-roadmap-to-docs` skill.

---

## 1. Functional Understanding

### What it is
<!-- SYNC-BLOCK: what -->
`foundation.auth` is the authentication and session-management engine for the EDE platform. It owns the JWT round-trip, the `ir.session` ledger, and the FastAPI middleware that converts a Bearer token into a `principal` mapping the rest of the platform can rely on. It exposes five REST endpoints (`/api/auth/login`, `/logout`, `/refresh`, `/me`, `/switch-organization`), the `JwtService` for issuing and decoding access tokens, and the `SessionService` for the server-side session ledger with refresh-token rotation and revocation.
<!-- /SYNC-BLOCK -->

### Why it exists
<!-- SYNC-BLOCK: why -->
The platform needs exactly one auth surface so every domain controller can trust `request.state.principal` without re-implementing JWT issuance, refresh rotation, or session revocation. Building it once in foundation removes the temptation for each domain to roll its own token handling — and makes it possible for `foundation.security` to layer organization scoping, RBAC, and record rules on top of a single, well-defined principal.
<!-- /SYNC-BLOCK -->

### How a user / consumer interacts with it
<!-- SYNC-BLOCK: how -->
- **End-user UX entry points** — the React webclient login screen calls `POST /api/auth/login`; the org switcher calls `POST /api/auth/switch-organization`; logout calls `POST /api/auth/logout`. The user never sees `ir.session` directly.
- **Programmatic entry points for other modules** — any controller reads `request.state.principal` and forwards it to `env.with_principal(principal)`. Any service that needs to mint a token uses `JwtService.from_settings(settings)`. Any logout/security flow calls `SessionService.revoke_session()` or `revoke_all_sessions()`.
- **Integration boundary** — produces: JWT access tokens, `ir.session` ledger rows, the `Env.principal` mapping injected on each request. Consumes: `res.user` (from `foundation.base`) for identity and the stored password hash.
<!-- /SYNC-BLOCK -->

## 2. Technical Implementation

### Architecture at a glance
<!-- SYNC-BLOCK: architecture -->
```
[Client]
   │  Authorization: Bearer <access_token>
   ▼
[AuthMiddleware]  (src/ede/core/adapters/http/fastapi/auth_middleware.py)
   │  JwtService.decode_token() → claims
   │  SessionService validate → ir.session row not revoked, not expired
   ▼
request.state.principal = { user_id, tenant_id, active_organization_id, ... }
   │
   ▼
[Route handler]
   env.with_principal(principal) → downstream dispatch
```

Login/refresh write to `ir.session` (refresh token stored as SHA-256 hash, never plaintext). Logout flips `revoked=True`. Refresh rotates: old refresh token marked revoked, new pair issued.
<!-- /SYNC-BLOCK -->

### Models
<!-- SYNC-BLOCK: models -->
| Model Key | Purpose | Source File |
|---|---|---|
| `ir.session` | Server-side session ledger: `user_id`, `tenant_id`, `refresh_token_hash` (SHA-256), `expires_at`, `revoked`, `active_organization_id`. | [src/ede/foundation/auth/models/session.py](../src/ede/foundation/auth/models/session.py) |
<!-- /SYNC-BLOCK -->

### Services & key code paths
<!-- SYNC-BLOCK: services -->
| Service / Class | Responsibility | Source File |
|---|---|---|
| `JwtService` | PyJWT 2.9+ wrapper. `encode_access_token`, `decode_token`, `is_valid`, `from_settings()` factory. | [src/ede/core/services/auth/jwt_service.py](../src/ede/core/services/auth/jwt_service.py) |
| `SessionService` | `create_session`, `refresh_session`, `revoke_session`, `revoke_all_sessions`, `switch_active_organization`. | [src/ede/foundation/auth/services/session_service.py](../src/ede/foundation/auth/services/session_service.py) |
| `AuthMiddleware` | Decodes Bearer JWT, validates `ir.session`, injects `request.state.principal`. | [src/ede/core/adapters/http/fastapi/auth_middleware.py](../src/ede/core/adapters/http/fastapi/auth_middleware.py) |
| `verify_password` + hash utilities | Password hashing and verification. Exported from `res.user`. | [src/ede/foundation/base/models/user.py](../src/ede/foundation/base/models/user.py) |
<!-- /SYNC-BLOCK -->

### Commands
<!-- SYNC-BLOCK: commands -->
| Command | Trigger | Effect |
|---|---|---|
| _none specific to auth — auth flows are HTTP-first and route through `JwtService` / `SessionService` directly. CRUD on `ir.session` uses the generic `ede.*` commands._ | | |
<!-- /SYNC-BLOCK -->

### Events emitted
<!-- SYNC-BLOCK: events -->
| Event | When fired | Typical subscribers |
|---|---|---|
| _none specific to auth in the baseline — generic `ede.record.*` events fire for `ir.session` CRUD via `CrudKernel`._ | | |
<!-- /SYNC-BLOCK -->

### REST endpoints
<!-- SYNC-BLOCK: rest -->
| Method + Path | Purpose | Controller |
|---|---|---|
| `POST /api/auth/login` | Validate credentials, mint access + refresh tokens, create `ir.session` row. | [src/ede/foundation/auth/api/auth.py](../src/ede/foundation/auth/api/auth.py) |
| `POST /api/auth/logout` | Revoke the active `ir.session` row. | [src/ede/foundation/auth/api/auth.py](../src/ede/foundation/auth/api/auth.py) |
| `POST /api/auth/refresh` | Rotate refresh token, mint new access token. | [src/ede/foundation/auth/api/refresh.py](../src/ede/foundation/auth/api/refresh.py) |
| `GET /api/auth/me` | Return the resolved `principal` for the current Bearer token. | [src/ede/foundation/auth/api/me.py](../src/ede/foundation/auth/api/me.py) |
| `POST /api/auth/switch-organization` | Re-stamp `active_organization_id` on the session (added by `foundation.security` Phase 1). | [src/ede/foundation/auth/api/switch_organization.py](../src/ede/foundation/auth/api/switch_organization.py) |
<!-- /SYNC-BLOCK -->

### Lifecycle hooks
<!-- SYNC-BLOCK: hooks -->
| Hook key | Behavior |
|---|---|
| _none specific to auth in the baseline._ | |
<!-- /SYNC-BLOCK -->

### State machine
<!-- SYNC-BLOCK: state-machine -->
```
ir.session lifecycle:
    create (login)  ──►  active  ──► (logout / revoke_all) ──► revoked
                            │
                            └── (refresh) ──► refresh_token rotated, expires_at extended
                            │
                            └── (switch-organization) ──► active_organization_id updated
                            │
                            └── (expires_at past) ──► expired (rejected by middleware)
```
<!-- /SYNC-BLOCK -->

## 3. Configuration Introduced

> Captures every knob this module adds. Two parallel tracks — foundation `pydantic-settings` (env vars / `ede.conf`) vs `ir.config` runtime store. Empty rows are fine when the module introduces no config of that kind. Missing sections are an audit failure.

### Activation
<!-- SYNC-BLOCK: activation -->
- `ACTIVE_MODULES` entry (in `src/ede/foundation/settings.py`): `auth`
- `ACTIVE_DOMAINS` entry: _not applicable — foundation app_
- Manifest `depends`: `foundation.base`
<!-- /SYNC-BLOCK -->

### Foundation-level settings (`FoundationSettings` / env vars)
<!-- SYNC-BLOCK: foundation-settings -->
| Setting Key | Type | Default | Env Var | Purpose |
|---|---|---|---|---|
| `JWT_SECRET_KEY` | `str` | `"change-me-in-production"` | `EDE_JWT_SECRET_KEY` | Signing key for access + refresh tokens. **Must be overridden in production.** |
| `JWT_ALGORITHM` | `str` | `"HS256"` | `EDE_JWT_ALGORITHM` | JWT signing algorithm. |
| `JWT_ACCESS_TOKEN_EXPIRE_MINUTES` | `int` | `30` | `EDE_JWT_ACCESS_TOKEN_EXPIRE_MINUTES` | Lifetime of an issued access token. |
| `JWT_REFRESH_TOKEN_EXPIRE_DAYS` | `int` | `7` | `EDE_JWT_REFRESH_TOKEN_EXPIRE_DAYS` | Lifetime of an issued refresh token; controls `ir.session.expires_at`. |
<!-- /SYNC-BLOCK -->

### Runtime config (`ir.config` keys)
<!-- SYNC-BLOCK: ir-config -->
| Config Key | Scope | Type | Default | Purpose |
|---|---|---|---|---|
| _none_ | | | | |
<!-- /SYNC-BLOCK -->

### Declarative settings (XML `<settings>` panels)
<!-- SYNC-BLOCK: xml-settings -->
| Panel | File | Fields |
|---|---|---|
| _none_ | | |
<!-- /SYNC-BLOCK -->

### Seed data shipped
<!-- SYNC-BLOCK: seed-data -->
| Data file | What it seeds |
|---|---|
| _none_ | |
<!-- /SYNC-BLOCK -->

## 4. Developer & User Notes

### Status Snapshot
<!-- SYNC-BLOCK: status-snapshot -->
| Phase | Title | Status | Roadmap |
|---|---|---|---|
| Phase 1 | Baseline — JWT + Session + Middleware + 5 REST endpoints | ✅ Delivered (baseline) | [roadmap/foundation/auth/README.md](../roadmap/foundation/auth/README.md) |
<!-- /SYNC-BLOCK -->

### Built Capabilities
> Populated only when a feature reaches ✅ Delivered in the roadmap.

<!-- SYNC-BLOCK: built -->
| Feature | Model Keys | Key Files | Roadmap Source |
|---|---|---|---|
| JWT issuance & decode | _none_ | [src/ede/core/services/auth/jwt_service.py](../src/ede/core/services/auth/jwt_service.py) | [roadmap/foundation/auth/README.md](../roadmap/foundation/auth/README.md) |
| Session ledger | `ir.session` | [src/ede/foundation/auth/models/session.py](../src/ede/foundation/auth/models/session.py), [src/ede/foundation/auth/services/session_service.py](../src/ede/foundation/auth/services/session_service.py) | [roadmap/foundation/auth/README.md](../roadmap/foundation/auth/README.md) |
| Bearer-token middleware | _none_ | [src/ede/core/adapters/http/fastapi/auth_middleware.py](../src/ede/core/adapters/http/fastapi/auth_middleware.py) | [roadmap/foundation/auth/README.md](../roadmap/foundation/auth/README.md) |
| Auth REST endpoints (5) | `ir.session` | [src/ede/foundation/auth/api/](../src/ede/foundation/auth/api/) | [roadmap/foundation/auth/README.md](../roadmap/foundation/auth/README.md) |
| Password hashing | _none — exported from `res.user`_ | [src/ede/foundation/base/models/user.py](../src/ede/foundation/base/models/user.py) | [roadmap/foundation/auth/README.md](../roadmap/foundation/auth/README.md) |
<!-- /SYNC-BLOCK -->

### Known Gaps
> Mirrored from gap tables in the roadmap (🔴/🟠/🟡 entries).

<!-- SYNC-BLOCK: gaps -->
| Gap | Severity | Roadmap Reference |
|---|---|---|
| _none yet — security hardening (rate-limit on login, password-reset flow, 2FA) is roadmapped under `foundation.security`, not here._ | | [roadmap/foundation/security/README.md](../roadmap/foundation/security/README.md) |
<!-- /SYNC-BLOCK -->

### Things developers commonly get wrong
<!-- HAND-AUTHORED — preserved across syncs -->
- Do not delete `ir.session` rows to "log out" — call `SessionService.revoke_session()` so the revocation is auditable and consistent with refresh-token semantics.
- Do not call PyJWT directly — always go through `JwtService.from_settings(settings)` so algorithm, secret, and expiry stay in sync with foundation settings.
- The `active_organization_id` column on `ir.session` and the `/api/auth/switch-organization` endpoint are part of `foundation.security` Phase 1 — they live in `foundation.auth` code but are tracked under the security roadmap.
- `verify_password` lives on `res.user` (in `foundation.base`), not in `foundation.auth`. Auth authenticates against base; base owns the hash.

### Migration / upgrade notes
<!-- SYNC-BLOCK: migration -->
- Baseline `ir.session` schema ships with `foundation.auth`'s own migrations under [src/ede/foundation/auth/migrations/](../src/ede/foundation/auth/migrations/).
- `active_organization_id` column on `ir.session` was added by a later `foundation.security` Phase 1 migration; see the security roadmap for that migration revision.
- Production must override `JWT_SECRET_KEY` before first deploy. Booting with the default secret is treated as a misconfiguration.
<!-- /SYNC-BLOCK -->

### Permissions / RBAC
<!-- SYNC-BLOCK: rbac -->
| Role | Permissions |
|---|---|
| Anonymous | `POST /api/auth/login`, `POST /api/auth/refresh` only. |
| Authenticated user | `POST /api/auth/logout`, `GET /api/auth/me`, `POST /api/auth/switch-organization` for their own session. |
| _Role-based access on application records is owned by_ `foundation.security` _Phases 4–5._ | |
<!-- /SYNC-BLOCK -->

### Related modules
<!-- SYNC-BLOCK: related -->
- [Foundation Security](foundation-security.md) — primary consumer + extender (active-org claim, switch-organization endpoint).
- [Foundation Base](foundation-base.md) — owns `res.user` and the password hash that `foundation.auth` authenticates against.
- Legacy guide: [docs/08-authentication.md](08-authentication.md).
<!-- /SYNC-BLOCK -->

---

*Last sync: 2026-05-14. To refresh, invoke the `syncing-roadmap-to-docs` skill.*
