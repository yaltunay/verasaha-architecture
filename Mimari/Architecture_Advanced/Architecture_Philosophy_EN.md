This document describes the architectural approach of the VeraSaha platform and is intended for technical reference only. It does not constitute legal advice or a formal security certification.

## System goals

The VeraSaha platform targets field operations and meeting management for multiple organizations (tenants) from a single deployment. The primary goals are strict tenant isolation, predictable and resilient sync for mobile and web clients, secure file handling, and an operational model that can be reasoned about and automated. Business functionality is built on top of these foundations rather than bolted on afterwards. The architecture is intentionally modest in its technology choices but opinionated in how those choices are composed to deliver production-grade behavior.

## Design constraints

Key constraints shape the design: a single public API host (`api.verasaha.com`) to simplify CORS, certificates, and rate limiting; a multi-tenant data model that prefers logical isolation (TenantId + global filters) over per-tenant databases; offline-first usage patterns that assume weak or intermittent connectivity; and a lean operational footprint where a small team can understand, deploy, and support the system. These constraints push the design toward clear boundaries (reverse proxy vs API vs storage), simple data flows, and minimal "magic" in the runtime behavior.

## SaaS architecture principles

The system follows standard SaaS principles: one codebase serving many tenants, tenant-aware data models, environment separation (dev/test/prod) with configuration as code, and consistent API contracts across tenants. Features are designed to be tenant-agnostic at the code level but tenant-scoped at runtime, so that behavior is repeatable while data remains isolated. Backwards compatibility is treated as a default requirement for public APIs; changes are introduced behind version-tolerant contracts rather than through breaking modifications wherever possible.

## Multi-tenant philosophy

Multi-tenancy is treated as a core axis of the design, not a detail. Every request is processed in an explicit tenant context via `X-Tenant-Key` and `tenantId` in JWTs; the data model carries TenantId on tenant-scoped entities; EF Core global query filters enforce tenant predicates; background workers (SyncInbox, NotificationOutbox) and file access via signed URLs are tenant-aware. The philosophy is that cross-tenant data exposure must be made structurally difficult, requiring multiple independent failures rather than a single missed `WHERE` clause.

## Security-first mindset

Security is approached as a set of invariants rather than a checklist. Invariants such as "no cross-tenant data access", "no unauthenticated access to tenant data", and "no secrets in code or logs" drive detailed design: header-based tenant resolution, JWT tenant binding, strict TLS termination at the reverse proxy, signed CDN URLs, and structured logging with redaction. The system prefers mechanisms that are easy to enforce and audit (headers, claims, global filters) over ad-hoc checks scattered through handlers.

## Privacy-by-design mindset

Privacy considerations are integrated into the technical model: tenant and user scoping, audit logs without sensitive field values, signed URLs instead of public file listings, and secrets handled as configuration rather than data. The architecture is designed to make it easier to implement retention and data subject request processes by keeping data localized (per-tenant) and traceable (correlation IDs, AuditLog, SyncInbox history). Privacy is treated as alignment between data flows and business intent, not only as an after-the-fact compliance exercise.

## Operational maturity

Operational maturity focuses on observability, controlled failure modes, and runbook-friendly behavior. Structured logs, metrics, and traces provide the raw signals; rate limiting, health probes, and isolation between API, workers, and storage provide control points. The system favors deterministic behavior under load (e.g. returning 429s, enforcing quotas) over trying to be "too helpful" and silently degrading. Production readiness is defined not just by uptime, but by how easily incidents can be triaged and corrected with the available signals.

## Trade-off thinking

The architecture is explicit about trade-offs: a single database plus logical isolation is chosen over per-tenant databases; signed URLs and CDN-based delivery are chosen over fully dynamic stream-per-request; Outbox and eventual consistency are chosen over synchronous, in-request notification sending. These trade-offs prioritize operability, cost, and conceptual simplicity while accepting bounded downsides such as link sharing until expiry or eventual notification delivery. The guiding principle is to surface trade-offs clearly, document them in ADRs, and allow for evolution when product or compliance needs change.

