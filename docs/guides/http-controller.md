# Wire a new HTTP controller

Expose a custom REST endpoint that authenticates the caller, resolves the tenant, and delegates to the command bus — all from one decorated class.

```python
from ede.core import api
from ede.core.services.http.controller import RouteController
from ede.core.types import Command


@api.route_config(prefix="/api/blog", tags=["blog"])
class BlogController(RouteController):

    @api.route("/posts/{post_id}/publish", methods=["POST"], auth="user")
    def publish_post(self, post_id: str) -> dict:
        env = self.env
        result = env.dispatch(Command("blog.post.publish", payload={"post_uuid": post_id}))
        return {"published": True, "post_id": post_id}
```

Routes are scanned from imported modules at boot and mounted on the FastAPI adapter. No manual router registration — importing the module is what registers it.

---

## 1. Subclass `RouteController` and configure the prefix

`@api.route_config` sets the shared URL prefix and OpenAPI tags for every route on the class:

```python
@api.route_config(prefix="/api/blog", tags=["blog"])
class BlogController(RouteController):
    ...
```

`RouteController` gives each handler `self.env` — a request-scoped `Env` already carrying the resolved tenant and the authenticated principal.

## 2. Declare each endpoint with `@api.route`

```python
@api.route("/posts", methods=["GET"], auth="user")
def list_posts(self, query_params: dict = None) -> dict:
    posts = self.env.models["blog.post"].search([("published", "=", True)])
    return {"items": posts.read(["title", "published"])}
```

- `methods` is the HTTP verb list (`["GET"]`, `["POST"]`, …).
- `auth="user"` requires a valid JWT; the decoded principal lands on `self.env.principal`. Use `auth="public"` for unauthenticated endpoints.
- Path parameters (`{post_id}`) and `query_params` / `body` arrive as handler arguments.

## 3. Read directly, mutate through the bus

Reads go straight through the model proxy — no command dispatch:

```python
post = self.env.models["blog.post"].browse(post_id).read()[0]
```

Mutations go through `env.dispatch` so lifecycle hooks, events, and RBAC all fire:

```python
self.env.dispatch(Command("blog.post.publish", payload={"post_uuid": post_id}))
```

`Command.payload` is required — pass `payload={}` when there's nothing to send. See [Command & Event Bus](../04-command-event-bus.md) for the full dispatch contract.

## 4. Make sure the module is imported at boot

Controllers register by import. List the controller's package in the app so it loads when the app activates — the scanner finds every `RouteController` in imported modules and mounts its routes. Activate the app via `ACTIVE_MODULES` / `ACTIVE_DOMAINS`; there is no auto-discovery.

## 5. Verify

With the server running, the endpoint appears in the OpenAPI schema and responds:

```bash
curl -X POST http://localhost:8002/api/blog/posts/<uuid>/publish \
     -H "Authorization: Bearer <access-token>"
```

If a route doesn't show up, the controller's module isn't being imported — check the app is in `ACTIVE_MODULES` / `ACTIVE_DOMAINS` and that the package imports the controller.

## Reference

- `RouteController`: `src/ede/core/services/http/controller.py`
- Route decorators: `src/ede/core/services/http/decorators.py`
- FastAPI adapter: `src/ede/core/adapters/http/`
- Related: [HTTP Layer](../07-http-layer.md), [Authentication](../08-authentication.md), [Auth — Login & Sessions](../features/auth.md).
