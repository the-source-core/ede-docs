# Auth — Login & Sessions

Authenticate users with JWT access tokens and rotating refresh tokens. The login / logout / refresh / `me` endpoints are wired and ready — you only configure the secret and set token TTLs.

```bash
# Log in
curl -X POST http://localhost:8000/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email": "admin@example.com", "password": "admin"}'

# Response
{
    "access_token": "eyJhbGciOiJI…",
    "refresh_token": "rt_3f8a…",
    "token_type": "bearer",
    "expires_in": 1800
}
```

The access token is a signed JWT with a short TTL (30 minutes by default). The refresh token is opaque and stored hashed in `ir.session`; rotate it through `/api/auth/refresh` to get a new access token without re-entering credentials.

---

## What you get

-   **Four endpoints**: `POST /api/auth/login`, `POST /api/auth/logout`, `POST /api/auth/refresh`, `GET /api/auth/me`.
-   **`JwtService`** — class-based, reusable: `encode_access_token()`, `decode_token()`, `is_valid()`, `from_settings(settings)`.
-   **`SessionService`** — class-based: `create_session()`, `refresh_session()`, `revoke_session()`, `revoke_all_sessions()`.
-   **`AuthMiddleware`** — extracts and verifies `Authorization: Bearer <token>` on every request, populates `request.state.principal`.
-   **`Env.principal`** — typed dict of the authenticated identity (`{"user_id": …, "tenant_id": …, "role": …}`) auto-cloned into every command handler.
-   **`ir.session`** model — persistent record of issued refresh tokens, with revocation.
-   **PyJWT** under the hood — symmetric (`HS256`) by default; switchable.

## How to use it

### Protect an HTTP route

Every route under a `RouteController` that doesn't opt out is authenticated. Inside the handler, `env.principal` is populated:

```python
from ede.core import api
from ede.core.services.http.controller import RouteController


@api.route_config(prefix="/api/blog")
class BlogController(RouteController):

    @api.route("/me/posts", methods=["GET"])
    def my_posts(self, request, env):
        author_uid = env.principal["user_id"]
        posts = env.models["blog.post"].search([
            ("author_id", "=", author_uid),
        ])
        return [p.read() for p in posts]
```

If the bearer token is missing or invalid, the middleware returns `401 Unauthorized` before your handler runs.

### Issue tokens manually (e.g. for tests or background work)

```python
from ede.core.services.auth.jwt_service import JwtService

svc = JwtService.from_settings(settings)
access = svc.encode_access_token({"sub": user.dbid, "tenant_id": "system"})
payload = svc.decode_token(access)   # raises on expired / invalid
```

### Rotate refresh tokens

```python
from ede.foundation.auth.services.session_service import SessionService

session_svc = SessionService.from_settings(settings)
new_access, new_refresh = session_svc.refresh_session(env, old_refresh_token)
```

The old refresh token is invalidated atomically — replay attacks raise `InvalidTokenError`.

### Revoke a session (logout)

```python
session_svc.revoke_session(env, refresh_token)
```

Subsequent uses of either the access or refresh token are rejected at the middleware.

### Verify a password directly

```python
from ede.foundation.base.models.user import verify_password

ok = verify_password(plain_input, user.password_hash)
```

## Configuration

Set in `ede.conf` (env-file syntax) or as environment variables:

| Setting | Default | What it controls |
|---|---|---|
| `JWT_SECRET_KEY` | `change-me-in-production` | HMAC signing key. **Override in production.** |
| `JWT_ALGORITHM` | `HS256` | JWT signing algorithm. |
| `JWT_ACCESS_TOKEN_EXPIRE_MINUTES` | `30` | Access-token TTL. |
| `JWT_REFRESH_TOKEN_EXPIRE_DAYS` | `7` | Refresh-token TTL. |

!!! warning "Production"
    Override `JWT_SECRET_KEY` before deployment. The default value is publicly known.

## How it composes with other features

-   **[Tenancy](../09-tenancy.md)** — the `tenant_id` claim in the JWT is what the tenancy middleware uses to pick the right per-tenant database.
-   **[Security & Authorization](security.md)** — `env.principal` is the input to the permission engine. Every command goes through both auth (am I logged in?) and authz (am I allowed to do this?).
-   **[Tutorial — Permissions & Security](../tutorial/06-permissions.md)** — step-by-step walk-through of gating your own model.

## Reference

| Concept | Where it lives |
|---|---|
| `JwtService` | `src/ede/core/services/auth/jwt_service.py` |
| `SessionService` | `src/ede/foundation/auth/services/session_service.py` |
| `AuthMiddleware` | `src/ede/core/adapters/http/fastapi/auth_middleware.py` |
| `ir.session` model | `src/ede/foundation/auth/models/session.py` |
| Token errors | `src/ede/core/services/auth/errors.py` (`TokenError`, `ExpiredTokenError`, `InvalidTokenError`) |
| HTTP endpoints | `src/ede/foundation/auth/api/` |
| Full reference | [Authentication architecture guide](../08-authentication.md) |
