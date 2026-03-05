# Architecture Detailed (English)

Detailed architecture document for the VeraSaha SaaS platform. This document summarises the rationale for this design, trade-offs, and rejected alternatives.

---

## 1. Domain and Host Topology (Detail)

**Why a single API host?**  
Hosting the entire API at api.verasaha.com allows CORS and security policy to be managed in one place, simplifies certificate management, and ensures the client always targets the same origin. Requests to /api on other hosts (e.g. {tenant}.verasaha.com or verasaha.com) are rejected with 404 and the message: "API is served from api.verasaha.com." Misconfigured clients or proxies are thus detected immediately.

**Trade-off:** The tenant UI runs on its own subdomain (e.g. company.verasaha.com) while the API is on a different host (api.verasaha.com), which results in cross-origin requests. This is acceptable as long as CORS is configured correctly; security and operational simplicity are preferred.

**Rejected alternative:** Serving the API on each tenant subdomain (e.g. company.verasaha.com/api). This increases certificate, routing, and rate-limiting complexity; a single API host with centralised management was chosen.

**Reserved subdomains** (www, api, admin, app, debug, test) are defined in a single source (ReservedSubdomains); used both for X-Tenant-Key rejection and CORS origin checks. These names are never accepted as tenant keys.

**Reverse proxy (Nginx + Let's Encrypt):** TLS termination and routing to api/cdn/app hosts are done at the Nginx layer; forwarded headers (X-Forwarded-*) are passed to the API and used with a trusted-proxy configuration.

---

## 2. Tenant Model – API-Only Host and JWT Binding (Detail)

**Why X-Tenant-Key?**  
Because the API is served on a single host (api.verasaha.com), the tenant cannot be derived from the request host. Tenant information is therefore carried in a header (X-Tenant-Key). The client takes the subdomain from the UI host (e.g. company.verasaha.com) and sends the same value as X-Tenant-Key.

**Protected endpoints:** All endpoints that are not [AllowAnonymous] are protected; X-Tenant-Key is required. AllowAnonymous endpoints (e.g. POST /api/auth/discover, POST /api/auth/login) do not require the header.

**Reserved host rules:** The X-Tenant-Key value is checked against the reserved list; invalid or unknown tenants return 400/404. Inactive tenants also return 404.

**JWT tenant binding:** For authenticated requests, the tenantId claim in the JWT is compared with the TenantId resolved from X-Tenant-Key. Missing claim → 401 (unauthorised); mismatch → 403 (forbidden). Thus use of a token in another tenant's context is blocked; multi-tenant isolation is enforced.

**401 vs 403:** No identity or missing tenant claim → 401. Identity present but token tenant does not match request tenant → 403.

**Development:** Optional DevTenant configuration can assign a default tenant when X-Tenant-Key is not sent (Development only). This is not used in Production.

---

## 3. Core Architectural Patterns (Detail)

**Onion Architecture:** The Domain layer has no external dependencies; business rules and aggregates live here. The Application layer uses CQRS/MediatR and interfaces; no database reference. Infrastructure (external services) and Persistence (EF Core, DbContext) contain application details. Presentation (Web API) is only HTTP endpoints. This enables testability and isolation of change.

**Meeting aggregate and immutability:** After a meeting is completed, certain fields cannot be changed; domain rules require a change reason. This supports audit trail and data integrity.

**ParticipationType (Planned / Actual):** Distinguishes whether a participant was planned in advance or added in person; reporting and notification rules apply accordingly.

**AbsenceReason:** A reason is required when a participant is absent; enforced by domain validation.

**AuditLog:** All entity changes are written to a central AuditLog table via SaveChanges override or interceptor (TenantId, UserId, Action, EntityName, EntityId, CorrelationId, TimestampUtc, etc.). Sensitive data is not written to logs; used for audit and incident review.

**CorrelationId middleware:** Each request is assigned a unique CorrelationId (from header or TraceIdentifier). Logs and traces use this ID for request correlation; critical for incident diagnosis.

**Rate limiting:** Strict IP-based limit for auth (discover/login); separate policy for writes (mutations) by authenticated user or tenant+IP. Reduces abuse and DDoS; auth and write traffic are evaluated in separate partitions.

---

## 4. Offline-First Synchronisation (Detail)

**Why SyncInbox (TenantId, UserId, OpId)?**  
When the client sends the same request again with the same OpId, the server does not re-apply the mutation; it returns the previously stored result. This prevents double application and data corruption on network failure or retries. The key (TenantId, UserId, OpId) also enforces tenant and user isolation.

**UTC discipline:** All timestamps are UTC; sent and received in ISO 8601 (Z). since, serverTimeUtc, occurredAtUtc are all UTC. Avoids timezone confusion and sync errors.

**Tombstone model:** Deleted records are returned in the changes response as a "deleted" marker (tombstone); the client stays in sync with local deletion.

**Rejected alternative:** Server file streaming or direct file serving is not part of the sync contract; files are served via CDN and signed URLs in a separate flow.

---

## 5. Files and CDN (Detail)

**Why does the API not stream files?**  
Streaming file content through the API would create memory and CPU load; undesirable especially in constrained RAM environments. The CDN serves files directly via signed URLs; the API is limited to upload, metadata, delete, and signed URL generation. CDN caching and scaling benefits are also used.

**Signed URL:** Contains tenantId, expiry (Unix timestamp), and HMAC signature; the CDN validates expiry and signature on every request. Invalid or expired URLs return 403. There is no direct listing or unauthenticated access.

**Quota and magic number:** Tenant storage quota is checked before upload; 409 on exceed. File type whitelist and (when configured) magic number sniffing reduce spoofed-extension risk; tenant storage usage is tracked.

---

## 6. Notifications (Detail)

**Outbox pattern:** Notifications are written to the NotificationOutbox table; committed in the same transaction as the domain change. A background worker reads rows and sends to FCM. Handlers/controllers do not call FCM directly; at-least-once enqueue and worker-side retry/backoff/dead-letter handling are ensured.

**IdempotencyKey:** Re-enqueueing the same logical event (e.g. the same meeting reminder) is prevented by a unique constraint on (TenantId, IdempotencyKey). Format is documented (e.g. meeting-reminder:…).

**DEV compatibility:** When FCM is disabled (default in Development), the worker uses NoopNotificationSender; in Production, if FCM is disabled the worker is not started (configuration-controlled).

---

## 7. Observability (Detail)

**Structured logging:** Serilog structured logging; CorrelationId, TenantId, UserId (if present) are added to log context. Sensitive fields are not written to logs (redaction).

**Metrics:** The /metrics endpoint is in Prometheus format; access is restricted to Admin JWT or internal network. notifications.outbox.pending gauge; no tenant tag (cardinality rule).

**Tracing:** OpenTelemetry; Console exporter in Development; when OTLP endpoint is configured in Production, that exporter is added. correlation_id, user_id, tenant_id (optional) can be added to spans; tenant_id can be disabled due to cardinality concerns.

**Why no secrets in logs?** Writing sensitive information (passwords, tokens, PII) to logs is a security and compliance risk; prevented by redaction and log policy.

---

## 8. Multi-Tenant Isolation Guarantees

- All tenant-scoped data access is filtered by **TenantId**; EF Core global query filter and repository layer comply.
- Sync changes/apply return or apply only the relevant tenant's data; JWT tenantId claim must match X-Tenant-Key.
- Notification worker processes in a tenant-based (or tenant-scoped batch) manner; no cross-tenant data leakage.
- CDN signature includes tenantId; one tenant cannot use another tenant's file URL without a valid signature.
