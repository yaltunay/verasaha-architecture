# VeraSaha — Architecture One-Pager (English)

*This document is for portfolio and technical reference only. It does not constitute legal or security certification.*

---

## System goals and constraints

VeraSaha is a **multi-tenant SaaS platform** for field operations and meeting management. Goals: strict tenant isolation, offline-first mobile sync, secure file handling, and production-ready operations (TLS, rate limiting, audit, observability). Constraints: single API host (api.verasaha.com) for operational simplicity; tenant context via header (X-Tenant-Key) and JWT binding; no cross-tenant data access at API, DB, or worker level.

---

## Topology (description)

- **Edge:** Nginx (or NPM) reverse proxy; TLS termination; host-based routing to landing, API, CDN, app, and tenant UIs.
- **verasaha.com:** Landing + discovery/login start.
- **app.verasaha.com:** Admin panel.
- **api.verasaha.com:** Single backend API; all protected calls carry X-Tenant-Key; JWT tenantId must match.
- **cdn.verasaha.com:** File serving via signed URLs (tenantId + expiry + HMAC); no listing or anonymous access.
- **{tenant}.verasaha.com:** Tenant-specific web UI.
- **Mobile:** Offline-first Flutter client; sync via changes/apply and SyncInbox idempotency.
- **Backend:** Database (tenant-scoped with global query filters); blob/storage for uploads; NotificationOutbox + worker for push.

---

## What makes it production-grade (8–12 bullets)

- Strict tenant isolation (EF Core global filters, tenant-scoped SyncInbox/NotificationOutbox).
- JWT tenant binding (401/403 semantics; token cannot be used in another tenant context).
- Rate limiting (auth IP-based; write per user or tenant+IP).
- Audit log for all mutations + CorrelationId on every request.
- Secure uploads: type whitelist, optional magic-number check, tenant storage quota.
- Offline sync idempotency (SyncInbox) so retries do not double-apply.
- Outbox pattern for notifications (at-least-once, worker, retry/backoff).
- Signed CDN URLs (time-limited, HMAC); no API file streaming.
- Secrets via environment / secret store; no secrets in code or logs.
- Structured logging with redaction; metrics (/metrics protected); optional tracing.
- Reverse proxy + TLS; single API host for CORS and policy consistency.
- KVKK-aware design: tenant data separation, audit, retention-ready, DSR-supportive data mapping.

---

## Notable engineering trade-offs

- **Single API host vs tenant-per-host:** One host simplifies CORS, certs, and rate limiting; trade-off is cross-origin requests from tenant UIs and mandatory X-Tenant-Key on every call.
- **Logical tenant isolation vs DB-per-tenant:** Single DB with global filters reduces ops and backup complexity; isolation is logical and depends on correct tenant context and filters.
- **Signed URLs vs per-request auth to CDN:** Signed URLs allow stateless CDN validation and scale; trade-off is link sharing until expiry and need to keep signing secret secure.
- **Outbox vs direct FCM:** Outbox gives durability and decouples request path from FCM; trade-off is eventual consistency and need to run and monitor the worker.

---

## What I would improve next

- **Formal DSR runbooks:** Document export/delete procedures with legal review; clarify retention schedule and automation.
- **Tenant-level metrics (bounded):** Where cardinality is acceptable, add limited tenant-scoped metrics for support and capacity.
- **Conflict resolution for sync:** Define explicit rules or CRDTs for concurrent edits on the same entity from offline clients.
- **Secrets rotation:** Document and automate rotation for JWT and CDN signing keys without downtime.
- **WAF / bot protection:** Add edge rules or WAF for common attack patterns beyond rate limiting.

*Any of the above may have KVKK or security implications and should be reviewed accordingly.*
