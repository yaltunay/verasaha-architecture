# Security Architecture (English)

*This document is provided for architectural reference only and does not constitute legal advice or a formal security certification.*

---

## Security Architecture Overview

VeraSaha addresses security in a multi-tenant SaaS platform through layered controls: authentication, tenant isolation, API security, file security, secrets management, and infrastructure security. All protected endpoints operate in a mandatory tenant context via JWT and X-Tenant-Key; cross-tenant data access is structurally prevented.

---

## Security Invariants (Must Never Break)

- **No cross-tenant data access:** A request in tenant A’s context must never read or write tenant B’s data. Enforced by global query filter, JWT tenant binding, and tenant-scoped SyncInbox/NotificationOutbox.
- **No unauthenticated access to tenant data:** Protected endpoints require valid JWT; anonymous access only for discover/login. File access only via signed URLs (no public listing).
- **No secrets in code or logs:** Passwords, tokens, connection strings, and signing keys are never committed or logged; redaction applied.
- **Tenant binding is strict:** JWT `tenantId` must match X-Tenant-Key; mismatch → 403. Reserved header values rejected.
- **Audit trail for mutations:** Every entity change is recorded (who, when, which entity); no sensitive field values in audit.

Violating these invariants would compromise both security and KVKK-aligned data handling; any architectural change must preserve them.

---

## AuthN / AuthZ Model

- **Authentication (AuthN):** JWT Bearer token; issued after login (discover + login endpoints AllowAnonymous). Claims: `sub` (userId), `tenantId`, `exp`, `iss`. Fail-fast on invalid JWT config at startup.
- **Authorization (AuthZ):** Tenant-scoped: request is authorized for the tenant resolved from X-Tenant-Key only if JWT `tenantId` matches. Role/permission checks (if used) are applied after tenant binding. 401 = no or invalid identity; 403 = identity valid but not allowed (e.g. wrong tenant).
- **No cross-tenant use of token:** A token issued for tenant A cannot be used with X-Tenant-Key for tenant B; 403 returned.

---

## Tenant Isolation Model

- **Explicit tenant per request:** X-Tenant-Key header carries tenant key (e.g. subdomain); resolved to TenantId; validated (reserved/unknown/inactive → 400/404).
- **Data layer:** EF Core global query filter injects TenantId into all tenant-scoped queries; no opt-out in application code.
- **Sync and notifications:** SyncInbox key (TenantId, UserId, OpId); NotificationOutbox tenant-scoped; workers process only within tenant context. No cross-tenant reads or sends.
- **Files:** Signed URL includes tenantId; CDN validates; one tenant cannot generate a valid URL for another’s resource.

---

## File Security Model

- **Upload:** API only; JWT + X-Tenant-Key required. Quota check before write; file type whitelist; optional magic-number check. Files stored under tenant (and user) path; no direct public URL.
- **Download:** No API streaming. API issues short-lived signed URL (tenantId + expiry + HMAC). CDN validates on each request; invalid or expired → 403. No directory listing or anonymous access.
- **Deletion:** API endpoint; tenant- and ownership-scoped; audit logged.

---

## Observability as Security

- **Audit:** Every mutation → AuditLog (TenantId, UserId, Action, EntityName, EntityId, CorrelationId, TimestampUtc). No sensitive field values. Supports incident review and compliance; KVKK-relevant.
- **Correlation:** CorrelationId on every request; propagated in logs and optional traces. Enables request-level investigation without logging PII.
- **Redaction:** Passwords, tokens, and full PII never in logs. Structured logging (Serilog) with context (CorrelationId, TenantId, UserId) for diagnostics while preserving privacy.
- **Metrics/tracing:** /metrics protected (Admin or internal); no high-cardinality tenant in metrics. Spans may carry correlation_id, user_id; no secrets. Observability supports detection of abuse and anomalies.

---

## Operational Controls

- **Rate limiting:** Auth (discover/login) IP-based strict limit; write (mutations) per authenticated user or tenant+IP. Reduces brute force and DDoS; auth and write partitioned.
- **Secrets:** .env or environment variables; production via secret store or CI/CD injection. No .env in version control. Rotation of JWT and CDN signing key should be planned; impacts auth and file access.
- **Environment separation:** Development may use DevTenant default and noop notification sender; production has no default tenant and FCM/worker as configured. Config-driven; no production secrets in dev.
- **Reverse proxy:** TLS at edge; X-Forwarded-* trusted for client IP/scheme; single API host for consistent CORS and policy.

---

## Authentication (JWT Model)

- **JWT usage:** Protected API requests use Bearer token authentication. Tokens carry claims such as `tenantId`, `sub` (userId), `exp`, and `iss`.
- **Tenant binding:** The `tenantId` claim in the JWT is compared with the tenant resolved from the request header `X-Tenant-Key`. On mismatch, the API returns 403; use of a token in another tenant’s context is blocked.
- **401 vs 403:** No identity or missing tenant claim → 401. Valid identity but tenant mismatch → 403.
- **Auth endpoints:** `POST /api/auth/discover` and `POST /api/auth/login` are AllowAnonymous; all other protected endpoints require JWT.
- **Fail-fast:** Invalid JWT configuration (secret, issuer, audience) prevents application startup; misconfiguration in production is reduced.

---

## Multi-Tenant Isolation

- **X-Tenant-Key:** Required on every protected request; reserved values (www, api, admin, app, debug, test) are rejected; unknown or inactive tenants return 400/404.
- **EF Core global query filter:** All tenant-scoped queries are automatically filtered by `TenantId`; application code cannot return cross-tenant data through normal queries.
- **Sync and Outbox:** SyncInbox and NotificationOutbox are scoped by (TenantId, UserId, OpId) and (TenantId, IdempotencyKey) to enforce tenant and user isolation.
- **CDN signature:** Signed URLs include tenantId and HMAC; one tenant cannot generate a valid signature for another tenant’s files.

---

## API Security Controls

- **CORS:** Only allowed origins (tenant subdomains, app.verasaha.com, etc.) are accepted; single API host (api.verasaha.com) allows central policy management.
- **Rate limiting:** Strict IP-based limits for auth (discover/login); separate policies for authenticated users or tenant+IP for write (mutation) traffic. Reduces brute force and DDoS impact.
- **Reserved subdomain rules:** X-Tenant-Key is validated against a reserved list; those names are never accepted as tenant keys.

---

## File Storage Security (Signed CDN URLs)

- **No API file streaming:** File content does not flow through the API; this avoids memory/CPU load and direct file-serving exposure.
- **Signed URLs:** Each URL contains tenantId, expiry (Unix timestamp), and HMAC signature; the CDN validates expiry and signature on every request. Invalid or expired URLs return 403.
- **No anonymous listing:** No unauthenticated file or directory listing; access is only via signed URLs issued by the API.
- **Quota and type checks:** Tenant storage quota is checked before upload; excess returns 409. File type whitelist and (when configured) magic-number sniffing reduce spoofed-extension risk.

---

## Secrets Management (.env)

- **Environment variables:** Sensitive configuration (JWT secret, DB connection string, CDN signing key, FCM, etc.) is supplied via .env or environment variables; no hardcoded secrets in source.
- **Production:** In production, secrets are injected via a secure secret store or CI/CD; .env files are not committed to version control.

---

## Logging Security (Redaction)

- **No sensitive data in logs:** Passwords, tokens, and full PII are not written to logs; a redaction/masking policy is applied.
- **Structured logging:** Serilog adds CorrelationId, TenantId, and UserId (when present) to log context; content supports audit and diagnostics while preserving privacy.
- **AuditLog:** Entity changes are written to an AuditLog table; sensitive field values are not stored; only who, when, and which entity was changed are recorded.

---

## Observability Security (Metrics + Tracing)

- **Metrics:** The `/metrics` endpoint is in Prometheus format; access is restricted to Admin JWT or internal network; no public access.
- **Tenant cardinality:** Metrics avoid high-cardinality tenant labels (e.g. notifications.outbox.pending) where they would cause cardinality explosion.
- **Tracing:** OpenTelemetry spans may include correlation_id and user_id; tenant_id is optional or disabled; no sensitive data is written to spans.

---

## Infrastructure Security (Docker + Reverse Proxy + TLS)

- **Reverse proxy (Nginx + Let's Encrypt):** TLS is terminated at the reverse proxy; api.verasaha.com, cdn.verasaha.com, and app.verasaha.com are served over HTTPS.
- **Forwarded headers:** X-Forwarded-For and X-Forwarded-Proto are passed to the API and used with a trusted-proxy configuration so rate limiting and logging use correct client information.
- **Docker:** The application can run in containers; network and volume isolation support production deployment.
- **Single API host:** All API traffic goes through api.verasaha.com; CORS and security policy are managed in one place.
