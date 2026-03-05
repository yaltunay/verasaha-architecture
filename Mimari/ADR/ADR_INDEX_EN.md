# Architecture Decision Records — Index (English)

This index lists all Architecture Decision Records (ADRs) for the VeraSaha platform.

| ADR ID   | Title                           | Short description |
|----------|----------------------------------|-------------------|
| ADR-001  | Multi-Tenant Isolation Model     | SaaS multi-tenant architecture, tenant data separation, global query filters, tenant-bound operations. |
| ADR-002  | Header-Based Tenant Resolution  | Use of X-Tenant-Key header, API host model, JWT tenant binding, prevention of cross-tenant access. |
| ADR-003  | Signed CDN URLs                 | File uploads stored on server, CDN serving model, signed URLs, expiration and signature validation. |
| ADR-004  | Outbox Pattern for Notifications | Background notifications, reliable message dispatch, retry strategy, eventual consistency. |
| ADR-005  | Offline Sync Architecture       | Offline-first mobile clients, delta synchronization, idempotent apply endpoint, sync inbox pattern. |
| ADR-006  | Reverse Proxy Deployment        | Reverse proxy layer, TLS termination, internal service exposure, host-based routing. |
| ADR-007  | Observability Stack             | Structured logging, metrics, tracing, centralized observability. |

---

## Document list

- [ADR-001_Multi_Tenant_Isolation_EN.md](ADR-001_Multi_Tenant_Isolation_EN.md)
- [ADR-002_Tenant_Header_EN.md](ADR-002_Tenant_Header_EN.md)
- [ADR-003_Signed_CDN_URLs_EN.md](ADR-003_Signed_CDN_URLs_EN.md)
- [ADR-004_Outbox_Pattern_EN.md](ADR-004_Outbox_Pattern_EN.md)
- [ADR-005_Offline_Sync_EN.md](ADR-005_Offline_Sync_EN.md)
- [ADR-006_Reverse_Proxy_EN.md](ADR-006_Reverse_Proxy_EN.md)
- [ADR-007_Observability_EN.md](ADR-007_Observability_EN.md)
