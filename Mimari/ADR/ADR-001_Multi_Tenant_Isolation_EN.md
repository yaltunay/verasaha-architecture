# ADR-001 Multi-Tenant Isolation Model

This document records architectural decisions for the VeraSaha platform and is intended for technical reference only.  
It does not constitute legal advice or a formal security certification.

## Status

Accepted

## Context

VeraSaha is a multi-tenant SaaS platform serving multiple organizations (tenants) from a single deployment. We need to guarantee that each tenant’s data is strictly isolated: no cross-tenant data access, no accidental leakage via queries or background jobs, and clear boundaries for sync, notifications, and file access.

## Decision Drivers

- **Strong tenant isolation guarantees:** The architecture must prevent cross-tenant data access by design at both the application and data layers to support security and privacy (e.g. KVKK) expectations.
- **Consistent, enforceable query model:** EF Core global query filters provide a single, central mechanism to enforce TenantId predicates on all tenant-scoped queries, reducing the chance of developer mistakes.
- **Operational simplicity at scale:** A single database with strict logical isolation keeps operations, backups, and migrations simpler than per-tenant databases or schemas while still supporting strong isolation.

## Decision

We adopt a **multi-tenant isolation model** with the following elements:

- **SaaS multi-tenant architecture:** One logical application serves all tenants. Tenant identity is explicit on every request and in every data access path. All tenant-scoped entities carry a `TenantId` and are never shared across tenants.

- **Tenant data separation:** All tenant-scoped data is stored with a `TenantId` column. Read and write paths always operate in the context of a single tenant. No API or worker may return or process data for more than one tenant in a single operation.

- **Global query filters:** EF Core global query filters are applied to tenant-scoped entities. Queries automatically include a `TenantId` predicate derived from the current request context. Application code cannot accidentally omit the tenant filter; cross-tenant result sets are prevented at the data layer.

- **Tenant-bound operations:** Sync (changes/apply), notifications (Outbox), and file access are scoped by tenant (and user where applicable). SyncInbox uses (TenantId, UserId, OpId); NotificationOutbox uses (TenantId, IdempotencyKey). Workers process rows in a tenant-aware manner so that no cross-tenant data is exposed or sent.

## Alternatives Considered

- **Separate database per tenant:** Would provide strong isolation but increase operational cost, migration complexity, and scaling overhead. Rejected in favour of a single database with strict logical isolation and global filters.

- **Schema-per-tenant:** Similar trade-offs to database-per-tenant; migration and tooling become more complex. Rejected.

- **Application-only tenant checks:** Relying only on application code to add tenant filters is error-prone. Rejected in favour of global query filters as a safety net.

## Consequences

- **Positive:** Strong logical isolation with a single database; global filters reduce the risk of developer error and support security/KVKK posture.
- **Positive:** Simpler operations and backups; tenant context is consistent across API, sync, and workers.
- **Negative:** All tenant-scoped code must run with a clear tenant context; misconfiguration can lead to empty results or errors (preferred over silent cross-tenant exposure).
- **Negative:** Performance and indexing must account for `TenantId` in tenant-scoped tables; schema and queries are tenant-aware by design.

## When to Revisit

Revisit if: regulatory or contractual requirements demand physical separation (e.g. DB-per-tenant or region); tenant scale or compliance audits show a need for stronger isolation guarantees; or multi-region deployment requires tenant-to-region binding.
