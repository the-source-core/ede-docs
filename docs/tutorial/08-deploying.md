# 8. Deploying

*Tutorial chapter — coming soon.*

This chapter will cover containerised deployment, switching from SQLite to PostgreSQL, running the standalone event worker, multi-tenant gateway mode, and the production checklist (override `JWT_SECRET_KEY`, disable `DEBUG`, set `ENABLE_API_DOCS=false`, configure CORS).

In the meantime:

-   `DEPLOYMENT_DOCKER.md` (at the repo root) — Docker Compose flows, `docker-compose.yml`, `docker-compose.services.yml`, `docker-compose.gateway.yml`.
-   [Foundation Gateway](../foundation-gateway.md) — multi-tenant SaaS control plane (`ede serve gateway`, per-tenant DB provisioning, Traefik routing).
-   [Tenancy](../09-tenancy.md) — per-tenant database isolation, `Env.with_tenant_id` propagation, thread context.
-   [Authentication](../08-authentication.md) — JWT settings, session lifecycle, refresh-token rotation.

That concludes the tutorial track. From here, browse the **Architecture** and **Foundation Apps** sections to deepen specific topics.
