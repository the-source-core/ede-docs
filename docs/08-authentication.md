# EDE Framework — Authentication (JWT + Sessions)

## Overview

Authentication is implemented in two parts:

1. **JWT Service** (`src/ede/core/services/auth/jwt_service.py`) — stateless token encoding/decoding
2. **Session Model** (`src/ede/foundation/auth/models/session.py`) — stateful session tracking
3. **Session Service** (`src/ede/foundation/auth/services/session_service.py`) — session lifecycle
4. **Auth Middleware** (`src/ede/core/adapters/http/fastapi/auth_middleware.py`) — per-request enforcement

The combination provides **short-lived JWTs** (access tokens) with **long-lived refresh tokens**
(stored as SHA-256 hashes in DB), revocable at any time via the session record.

---

## JWT Configuration

`src/ede/foundation/settings.py`

```python
JWT_SECRET_KEY = "change-me-in-production"
JWT_ALGORITHM = "HS256"
JWT_ACCESS_TOKEN_EXPIRE_MINUTES = 30
JWT_REFRESH_TOKEN_EXPIRE_DAYS = 7
```

---

## JwtService

`src/ede/core/services/auth/jwt_service.py`

Stateless, reusable service for JWT operations.

### Factory

```python
jwt_service = JwtService.from_settings(settings)
# reads JWT_SECRET_KEY, JWT_ALGORITHM, JWT_ACCESS_TOKEN_EXPIRE_MINUTES
```

Or construct directly:
```python
jwt_service = JwtService(
    secret_key="my-secret",
    algorithm="HS256",
    access_token_expire_minutes=30,
)
```

### Encoding

```python
token = jwt_service.encode_access_token(
    user_id="uuid-1234",
    email="user@example.com",
    tenant_id="acme",
    session_id="uuid-session",   # Optional but recommended for revocation
)
```

JWT payload (claims):
```json
{
    "sub": "uuid-1234",
    "email": "user@example.com",
    "tenant_id": "acme",
    "sid": "uuid-session",
    "iat": 1700000000,
    "exp": 1700001800
}
```

### Decoding

```python
claims = jwt_service.decode_token(token)
# → dict with all JWT claims

# Raises:
#   ExpiredTokenError  — if token is expired
#   InvalidTokenError  — if signature invalid, malformed, or missing required claims
```

### Validation (non-raising)

```python
is_ok = jwt_service.is_valid(token)   # → bool
```

---

## ir.session Model

`src/ede/foundation/auth/models/session.py`

Tracks active sessions in the database per tenant.

| Field | Type | Notes |
|---|---|---|
| `user_id` | UUID | References `res.user.record_uuid` |
| `tenant_id` | Char | Tenant this session belongs to |
| `refresh_token_hash` | Char | SHA-256 hash of the refresh token |
| `expires_at` | DateTime | Refresh token expiry |
| `revoked` | Boolean | Explicitly revoked flag |
| `os` | Char | Client OS (optional, from User-Agent) |
| `browser` | Char | Client browser (optional, from User-Agent) |

Sessions are per-tenant (stored in the tenant's own database).

---

## SessionService

`src/ede/foundation/auth/services/session_service.py`

Manages session lifecycle.

### Factory

```python
session_service = SessionService.from_settings(settings)
```

### Create Session (Login)

```python
result = session_service.create_session(
    env=env,
    user_id="uuid-1234",
    tenant_id="acme",
)
# result = {
#     "access_token": "eyJ...",
#     "refresh_token": "raw-uuid-refresh-token",
#     "token_type": "Bearer",
#     "expires_in": 1800,   # seconds
# }
```

Internally:
1. Generates a random refresh token (UUID4)
2. Hashes it (SHA-256) → stores in `ir.session`
3. Calls `jwt_service.encode_access_token(session_id=session.id, ...)`
4. Returns access + refresh tokens

### Refresh Session

```python
result = session_service.refresh_session(
    env=env,
    refresh_token="raw-refresh-token",
    tenant_id="acme",
)
# Returns new access_token + new refresh_token (rotation)
# Raises if session not found, expired, or revoked
```

Refresh token rotation: each refresh invalidates the old token and issues a new one.

### Revoke Session (Logout)

```python
session_service.revoke_session(env=env, session_id="uuid-session", tenant_id="acme")
# Sets ir.session.revoked = True
```

### Revoke All Sessions

```python
session_service.revoke_all_sessions(env=env, user_id="uuid-1234", tenant_id="acme")
# Marks all active sessions for this user as revoked
```

---

## Auth Endpoints

`src/ede/foundation/auth/api/controllers.py`

| Endpoint | Method | Auth | Purpose |
|---|---|---|---|
| `/api/auth/login` | POST | public | Login → access + refresh tokens |
| `/api/auth/logout` | POST | user | Revoke current session |
| `/api/auth/refresh` | POST | public | Get new access token via refresh token |
| `/api/auth/me` | GET | user | Return current user details |

### Login Request/Response

```
POST /api/auth/login
Content-Type: application/json

{
    "email": "admin@example.com",
    "password": "secret",
    "tenant_id": "acme"
}
```

```json
{
    "access_token": "eyJ...",
    "refresh_token": "9f3c7a...",
    "token_type": "Bearer",
    "expires_in": 1800
}
```

### Refresh Request

```
POST /api/auth/refresh
Content-Type: application/json

{
    "refresh_token": "9f3c7a...",
    "tenant_id": "acme"
}
```

### Using the Token

```
GET /api/shipments/
Authorization: Bearer eyJ...
X-Tenant-Id: acme
```

---

## AuthMiddleware Flow

`src/ede/core/adapters/http/fastapi/auth_middleware.py`

Per-request flow for protected routes:

```
Request arrives
  │
  ├── Path in bypass list? (/wc, /assets, /health, non-/api paths)
  │     → pass through (no auth)
  │
  ├── Look up auth level for (method, path) → "public" | "user" | ...
  │     └── "public" → pass through
  │
  └── "user" / "portal" / "application":
        ├── Extract Authorization: Bearer <token>
        │     └── Missing → 401
        │
        ├── jwt_service.decode_token(token)
        │     └── ExpiredTokenError → 401 {"detail": "Token expired"}
        │     └── InvalidTokenError → 401 {"detail": "Invalid token"}
        │
        ├── _check_session_active(session_id, tenant_id)
        │     └── Query ir.session by session_id
        │     └── session.revoked=True → 401 {"detail": "Session revoked"}
        │     └── session not found → 401 {"detail": "Session not found"}
        │
        ├── Set request.state.principal = {
        │       "auth_type": "user",
        │       "user_id": claims["sub"],
        │       "email": claims["email"],
        │       "tenant_id": claims["tenant_id"],
        │       "session_id": claims["sid"],
        │       "identity": claims["email"],
        │   }
        │
        └── pass to next handler
```

---

## Accessing Principal in Controllers

```python
@api.route("/me", methods=["GET"], auth="user")
def get_me(self) -> dict:
    principal = self.env.principal
    # {
    #     "auth_type": "user",
    #     "user_id": "uuid-1234",
    #     "email": "user@example.com",
    #     "tenant_id": "acme",
    #     "session_id": "uuid-session",
    # }
    user = self.env.models["res.user"].browse(principal["user_id"])
    return user.read()[0]
```

---

## Error Types

`src/ede/core/services/auth/errors.py`

```python
class TokenError(EdeError): ...          # Base class for token errors
class ExpiredTokenError(TokenError): ... # JWT exp claim has passed
class InvalidTokenError(TokenError): ... # Bad signature, malformed, missing claims
```

---

## Why JWT + Server-Side Sessions?

### Pure JWTs are not enough for enterprise use

A pure stateless JWT cannot be revoked. If an employee is terminated or a session is
compromised, you must wait for the token to expire — up to 30 minutes. For enterprise
applications that need immediate revocation (role changes, admin lockout, compliance
requirements), this is unacceptable.

### Pure server-side sessions don't scale horizontally

Storing session state in memory or a single DB means every request must hit the same
node. This limits horizontal scaling and creates a single point of failure.

### EDE's hybrid approach

EDE uses **short-lived JWTs** (30 minutes, default) for stateless, low-latency request
authentication, combined with **server-side session records** (`ir.session`) for:
- Immediate revocation (mark `revoked=True` → next request fails)
- Refresh token rotation (each use issues a new refresh token, invalidating the old)
- Multi-device session management (revoke all sessions for a user with one call)
- Audit trail (which device, browser, OS, when)

This is the approach used by production SaaS systems: stateless on the hot path,
stateful for security operations.

### Sessions are per-tenant

`ir.session` records are stored in the tenant's own database. Session data never
crosses tenant boundaries. A session revocation in tenant `"acme"` has no effect
on tenant `"beta"`.

---

## Security Notes

1. **Store secrets in env vars** — never commit `JWT_SECRET_KEY` to source control.
   Use `.env` file or container secrets in production.

2. **Refresh token rotation** — each use of a refresh token issues a new one and invalidates
   the old. This limits the window for refresh token theft.

3. **Session revocation** — `AuthMiddleware` checks `ir.session` on every request for
   protected routes. Revoking a session immediately denies further access, even if the
   JWT hasn't expired yet.

4. **Short-lived access tokens** — 30 minutes by default. Adjust
   `JWT_ACCESS_TOKEN_EXPIRE_MINUTES` based on security requirements.

5. **Password hashing** — `verify_password(plain, stored_hash)` is exported from
   `ede.foundation.base.models.user`. Uses bcrypt. Never store plain-text passwords.
