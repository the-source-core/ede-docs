# EDE Framework — HTTP Layer

## Overview

The HTTP layer is thin: it translates HTTP → Command and JSON ← result. Business logic
belongs in DomainModel handlers, not controllers.

```
HTTP Request
  └─► FastAPI (ASGI)
        └─► AuthMiddleware   (JWT → principal)
              └─► LogMiddleware  (timing/logging)
                    └─► FastApiHttpAdapter
                          └─► RouteController.method()
                                └─► env.dispatch(Command(...))
                                      └─► DomainModel handler
                                            └─► return dict
                                  ◄─── JSONResponse
```

---

## RouteController

`src/ede/core/services/http/controller.py`

All HTTP controllers inherit `RouteController`:

```python
from ede.core.services.http.controller import RouteController
from ede.core.services.http.decorators import route_config, route
from ede.core import api

@api.route_config(prefix="/api/shipments", tags=["Shipments"])
class ShipmentController(RouteController):
    # self.env is injected by FastApiHttpAdapter before each call

    @api.route("/", methods=["GET"], auth="user")
    def list_shipments(self) -> list:
        result = self.env.dispatch(Command(
            name="ede.search",
            payload={"domain": [("status", "!=", "cancelled")]},
            model_key="logistics.shipment",
        ))
        return result

    @api.route("/{id}", methods=["GET"], auth="user")
    def get_shipment(self, id: str) -> dict:
        result = self.env.dispatch(Command(
            name="ede.read_one",
            payload={},
            model_key="logistics.shipment",
            model_id=id,
        ))
        return result

    @api.route("/", methods=["POST"], auth="user")
    def create_shipment(self, body: dict) -> dict:
        return self.env.dispatch(Command(
            name="shipment.create",
            payload=body,
        ))
```

`RouteController` is a dataclass with a single attribute:
```python
@dataclass
class RouteController:
    env: Env
```

The adapter instantiates a new controller instance per request and injects the
request-scoped `Env`.

---

## Route Decorators

### `@api.route_config` — Controller-Level

```python
@api.route_config(
    prefix="/api/shipments",   # URL prefix for all routes in this controller
    tags=["Shipments"],        # OpenAPI tags
    version=None,              # Optional API version string
    description=None,          # Optional description for OpenAPI
)
```

This is equivalent to `@route_config` from `ede.core.services.http.decorators`.

### `@api.route` — Method-Level

```python
@api.route(
    path="/",                  # Path relative to controller prefix
    methods=["GET"],           # HTTP methods. Default: ["GET"]
    name=None,                 # Optional route name (for URL reversal)
    tags=None,                 # Optional additional tags (merged with controller tags)
    request_type="json",       # Request body type: "json" (default) | "form"
    auth="user",               # Auth level: "public" | "user" | "portal" | "application"
)
```

### Auth Levels

| Level | Enforcement |
|---|---|
| `"public"` | No authentication required |
| `"user"` | Valid JWT required. Principal injected into `env.principal`. |
| `"portal"` | Valid JWT + portal-level access |
| `"application"` | Valid JWT + application/API key level access |

The `auth` value is registered with the route and enforced by `AuthMiddleware`.

---

## HttpRouteRegistry

`src/ede/core/services/http/registry.py`

The `HttpRouteRegistry` is populated by `HttpRouteScanner` during boot. It holds all
discovered routes as `HttpRouteSpec` objects.

```python
@dataclass(frozen=True)
class HttpRouteSpec:
    controller_cls: Type[RouteController]
    handler_name: str
    meta: RouteMeta     # path, methods, name, tags, auth

    def stable_key(self) -> str:
        return f"{self.controller_cls.__module__}.{self.controller_cls.__name__}:{self.handler_name}"
```

Duplicate `(method, path)` combinations raise an error at registration time.

---

## HttpRouteScanner

`src/ede/core/services/http/scanner.py`

Called during app loading. Scans imported modules for `RouteController` subclasses:

1. Iterates `sys.modules` matching the app's module prefix
2. For each module, finds all `RouteController` subclasses
3. For each subclass, scans methods decorated with `@api.route`
4. Merges controller-level `prefix` + `tags` with method-level `path` + `tags`
5. Registers the `HttpRouteSpec` into `HttpRouteRegistry`

Path merging rules:
- `prefix="/api/shipments"` + `path="/"` → `/api/shipments/`
- `prefix="/api/shipments"` + `path="/{id}"` → `/api/shipments/{id}`
- No double slashes

---

## FastApiHttpAdapter

`src/ede/core/adapters/http/fastapi/fastapi.py`

Mounts the `HttpRouteRegistry` onto a FastAPI application:

1. For each `HttpRouteSpec`, creates a FastAPI route handler closure
2. The closure:
   a. Resolves `tenant_id` (from Host header or `X-Tenant-Id` header)
   b. Retrieves `request.state.principal` (set by `AuthMiddleware`)
   c. Clones `Env` with `with_tenant_id()` and `with_principal()`
   d. Instantiates the controller class with the request-scoped `Env`
   e. Extracts path parameters, query parameters, and JSON body
   f. Calls `controller.method(**kwargs)`
   g. Returns `JSONResponse(result)`

3. Adds all middlewares via `add_all_middlewares(app, settings=settings)`

---

## Middleware Stack

`src/ede/core/adapters/http/fastapi/middlewares.py`

```python
add_all_middlewares(app, settings=settings, env=base_env)
```

Middleware is applied in this order (innermost first):
1. `AuthMiddleware` — JWT validation, principal injection (runs closest to handler)
2. `LogMiddleware` — request timing and structured logging (outermost)

### AuthMiddleware

`src/ede/core/adapters/http/fastapi/auth_middleware.py`

**Always bypassed** for:
- Paths starting with `/wc` (web client static assets)
- Paths starting with `/assets`
- `/health` endpoint
- Any path not starting with `/api` or `/public`

**For all other paths**:
1. Looks up auth requirement via `get_route_auth(method, path)` (default: `"user"`)
2. `"public"` routes pass through immediately
3. `"user"` / `"portal"` / `"application"` routes:
   a. Extract `Authorization: Bearer <token>` header
   b. Decode JWT via `JwtService.decode_token(token)` → claims dict
   c. Validate session is still active via DB lookup (`ir.session` check)
   d. Set `request.state.principal = claims`
   e. If validation fails: return `401 Unauthorized`

**Principal shape**:
```python
{
    "auth_type": "user",
    "user_id": "uuid-...",
    "email": "user@example.com",
    "tenant_id": "acme",
    "session_id": "uuid-...",
    "identity": "user@example.com",
}
```

### Route Auth Registration

Routes are registered with their auth level during boot:

```python
register_route_auth("GET", "/api/shipments/", "user")
register_route_auth("POST", "/api/shipments/", "user")
register_route_auth("GET", "/api/health", "public")
```

`get_route_auth(method, path)` looks up the registered level. Returns `"user"` if not found
(fail secure).

---

## Request Context in Controllers

Inside a controller method:
- `self.env` — request-scoped `Env` (tenant + principal already set)
- `self.env.tenant_id` — resolved tenant ID
- `self.env.principal` — dict with user claims (or `None` for public routes)

```python
@api.route("/me", methods=["GET"], auth="user")
def get_current_user(self) -> dict:
    principal = self.env.principal
    user = self.env.models["res.user"].browse(principal["user_id"])
    return user.read()[0]
```

---

## RouteMeta Reference

```python
@dataclass
class RouteMeta:
    path: str             # relative path, e.g. "/{id}"
    methods: list[str]    # e.g. ["GET", "POST"]
    name: str | None      # optional route name
    tags: list[str]       # merged from controller + method
    request_type: str     # "json" | "form"
    auth: str             # "public" | "user" | "portal" | "application"
```

---

## URL Patterns

| Pattern | Method | Purpose | Auth |
|---|---|---|---|
| `/api/{model}` | GET | List records | user |
| `/api/{model}` | POST | Create record | user |
| `/api/{model}/{id}` | GET | Read one record | user |
| `/api/{model}/{id}` | PUT | Update record | user |
| `/api/{model}/{id}` | DELETE | Delete record | user |
| `/api/auth/login` | POST | Login → JWT | public |
| `/api/auth/logout` | POST | Invalidate session | user |
| `/api/auth/refresh` | POST | Refresh JWT | public |
| `/api/auth/me` | GET | Current user | user |
| `/health` | GET | Health check | public |

| `/api/web-push/events` | GET | SSE push stream | user |

---

## Web Push (SSE)

EDE ships a Server-Sent Events endpoint that allows the backend to push messages
to all connected browser tabs in real time.

### Endpoint

```
GET /api/web-push/events
Authorization: Bearer <access_token>
```

The browser opens a persistent `fetch`-based SSE connection (not the native
`EventSource` API, which cannot send `Authorization` headers). The server streams
`data: {...}\n\n` lines. A `": keepalive"` comment is sent every 25 seconds to
prevent proxy timeouts on idle connections.

### Controller

`src/ede/foundation/presentation/api/web_push.py` — `WebPushController`

```python
@api.route_config(prefix="", tags=["foundation.presentation.web_push"], version="v1")
class WebPushController(RouteController):

    @api.route("/api/web-push/events", methods=["GET"], request_type="sse")
    async def stream_events(self, is_disconnected) -> AsyncGenerator[str, None]:
        push_reg = getattr(self.env, "web_push_registry", None)
        tenant_id = self.env.tenant_id or ""
        entry = push_reg.subscribe(tenant_id)
        _, q = entry
        try:
            yield 'data: {"type":"connected"}\n\n'
            while True:
                if await is_disconnected():
                    break
                try:
                    msg = await asyncio.wait_for(q.get(), timeout=25.0)
                    yield f"data: {json.dumps(msg)}\n\n"
                except asyncio.TimeoutError:
                    yield ": keepalive\n\n"
        finally:
            push_reg.unsubscribe(tenant_id, entry)
```

The controller is an **async generator function** (`request_type="sse"`). The
FastAPI adapter detects this via `inspect.isasyncgenfunction()` and wraps the
result in a `StreamingResponse` with `media_type="text/event-stream"`.

`is_disconnected` is injected by the HTTP adapter from `request.is_disconnected` —
the controller itself has no FastAPI dependency.

### WebPushRegistry

`src/ede/core/services/push/web_push_registry.py`

Thread-safe registry that maps `tenant_id → [(event_loop, asyncio.Queue)]`.

| Method | Description |
|---|---|
| `subscribe(tenant_id)` | Called from the async SSE handler; registers a new queue |
| `unsubscribe(tenant_id, entry)` | Called in `finally`; removes the queue |
| `broadcast(tenant_id, message)` | Called from the sync EventWorker thread; uses `loop.call_soon_threadsafe` to bridge into each subscriber's asyncio queue |

Injected into `Env` at boot: `base_env.web_push_registry = WebPushRegistry()`.

### Push event handlers

`src/ede/foundation/presentation/events/web_client_events.py`

```python
@api.on_event("web.client.reload")
def handle_web_client_reload(event, env) -> None:
    push_reg = getattr(env, "web_push_registry", None)
    if push_reg:
        push_reg.broadcast(event.tenant_id or "", {
            "type": "web.client.reload",
            **dict(event.payload),
        })

@api.on_event("web.client.logout")
def handle_web_client_logout(event, env) -> None: ...
```

Any backend code can emit `web.client.reload` or `web.client.logout` via
`env.emit("web.client.reload", {...})` and the push handler will broadcast
to all connected tabs for that tenant.

### SSE request_type adapter contract

When `request_type="sse"` is declared on a route:

1. The adapter passes `is_disconnected=request.is_disconnected` as a kwarg.
2. The handler method must be an **async generator function** (contains `yield`).
3. `_handle_request_dispatcher_sse` wraps the generator in `StreamingResponse`.

---

## Structured Request Logging

Every request produces **two log lines** from different loggers:

```
ede.http.tenant | [dharmangsoni] PresentationController:action_records  action_slug=lm-countries
ede.http.access | GET /api/web/lm-countries/records -> 200 (22.63ms) client=127.0.0.1 request_id=2dbf5940c3864f3e8182a751e66dc15e
```

| Logger | Source | What it shows |
|--------|--------|---------------|
| `ede.http.tenant` | `handler.py` — emitted at handler entry | Resolved tenant, controller:method, path params |
| `ede.http.access` | `log_middleware.py` — emitted after response | HTTP method, path, status code, elapsed ms, client IP, request ID |

The two lines are intentionally distinct: `ede.http.tenant` leads with `[tenant]` (routing context), while `ede.http.access` leads with `METHOD /path` (HTTP outcome). This lets you grep each logger independently.

**`ede.http.tenant` format:**
```
[{tenant_id}] {ControllerClass}:{handler_method}  {path_param}={value} ...
```

**`ede.http.access` format (normal):**
```
{METHOD} {/path} -> {status} ({ms}ms) client={ip} request_id={id}
```

**`ede.http.access` format (exception):**
```
{METHOD} {/path} -> {status} ({ms}ms) client={ip} request_id={id} error={ExceptionClass}
```

Error-level logs on `ede.http.tenant` fire for handler exceptions (business rule violations, tenant DB not found, uncaught errors) and include `tenant_id`, `route`, and full traceback.

---

## Controller Best Practices

1. **Controllers are thin adapters** — no business logic. Translate HTTP → Command → return.
2. **Use `env.dispatch(Command(...))` for named actions** — not raw ORM calls in controllers.
3. **For CRUD operations**, dispatch `ede.create`, `ede.read_one`, `ede.search`, etc.
4. **Never import HTTP-specific types into DomainModel** — the domain layer must stay clean.
5. **One controller per app** is recommended, though multiple are fine.
6. **Path parameters** map to FastAPI path params (e.g. `{id}` → `id: str`).
7. **Request body** is injected as `body: dict` when `request_type="json"`.

---

## Why the HTTP Layer Is Designed This Way

### Thin controllers protect the domain

A controller that contains `if record.status == "pending" and invoice_count > 0: raise ...`
is a controller that can't be tested without a running HTTP server, can't be reused from a
CLI command, and can't be moved without dragging that logic with it. In EDE, that logic
belongs in a command handler or a hook. The controller's only job is to know *which command
to dispatch* and *what to pass as payload*.

### `env.dispatch()` is the stable contract

When a controller calls `env.dispatch(Command("shipment.confirm", ...))`, it is calling
a contract — not an implementation. The command handler can change its implementation,
be replaced, or have new hooks attached, without the controller changing at all. This
is what makes the system surviveble: HTTP and domain evolve independently.

### Middleware as transparent cross-cutting concerns

Authentication, logging, and tenant resolution are middleware — they run on every request
without any controller knowing about them. The controller receives a pre-populated `env`
with `tenant_id` and `principal` already set. Adding a new cross-cutting concern (rate
limiting, request tracing) adds a new middleware and nothing else changes.

### SSE as a first-class HTTP pattern

The `request_type="sse"` route type shows that the HTTP layer is not just REST. Streaming
responses, long-lived connections, and push patterns are first-class citizens. The domain
never knows how its events reach the browser — it just calls `env.emit("web.client.reload")`.
The HTTP layer handles the transport.
