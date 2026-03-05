# ADR-002 Header-Based Tenant Resolution

*This document records architectural decisions for the VeraSaha platform and is intended for technical reference only.*

## Status

Accepted

## Context

The API is served from a single host (api.verasaha.com). The tenant cannot be derived from the request host because all tenants use the same API host. We need a reliable, explicit way to identify the tenant for each request and to enforce that authenticated calls operate only in the correct tenant context. We must prevent cross-tenant access (e.g. a token issued for tenant A used with tenant B’s context).

## Decision Drivers

1. **Single API host:** One host (api.verasaha.com) simplifies CORS, certs, and rate limiting; tenant cannot be inferred from host, so an explicit request-level mechanism is required.
2. **Security and audit:** Header is visible and auditable; JWT binding prevents token use in a different tenant context (403).
3. **Operational simplicity:** No per-tenant API deployment or host-based routing for tenant resolution; one policy surface.

## Decision

We use **header-based tenant resolution** with the following design:

- **X-Tenant-Key header:** Every protected API request must include the `X-Tenant-Key` header. The value is the tenant key (e.g. subdomain such as `firma` for firma.verasaha.com). The client obtains this from the UI host (e.g. subdomain) and sends it on each request. Reserved values (www, api, admin, app, debug, test) are rejected; unknown or inactive tenants return 400/404.

- **API host model:** The entire API is served from api.verasaha.com. No tenant is inferred from the request host. CORS and security policy are managed in one place; certificate and routing complexity stay low.

- **JWT tenant binding:** For authenticated requests, the API resolves the tenant from `X-Tenant-Key` and compares it with the `tenantId` claim in the JWT. If the claim is missing, the request is rejected with 401. If the claim does not match the resolved tenant, the request is rejected with 403. A token issued for one tenant cannot be used in another tenant’s context.

- **Prevention of cross-tenant access:** The combination of mandatory X-Tenant-Key, validation against reserved/unknown tenants, and JWT tenant binding ensures that (1) every protected request has a valid tenant context and (2) the identity in the token is bound to that same tenant. Cross-tenant data access or privilege use is blocked at the API layer.

## Alternatives Considered

- **Tenant from host (e.g. firma.verasaha.com/api):** Would require serving the API on every tenant subdomain, increasing certificate, routing, and rate-limiting complexity. Rejected in favour of a single API host.

- **Tenant only in JWT:** Would require the client to send the correct tenant in the token; the server would still need a way to validate that the request is intended for that tenant. Header provides an explicit, auditable request-level tenant. Rejected.

- **Query parameter or body:** Less visible for auditing and easier to forget; headers are standard for request metadata. Rejected.

## Consequences

- **Positive:** Clear, explicit tenant per request; single API host simplifies operations and security; JWT binding prevents cross-tenant token misuse.
- **Positive:** 401/403 semantics support troubleshooting; header is auditable for security and KVKK-related traceability.
- **Negative:** All clients must send X-Tenant-Key on protected requests; CORS must be configured for cross-origin tenant UI to API.
- **Negative:** Development may use an optional DevTenant default when the header is omitted (Development only); must not leak to production.

## When to Revisit

Revisit if: regulatory or client requirements demand tenant-from-host (e.g. dedicated host per tenant); or a need for tenant-level TLS or WAF rules makes per-tenant edge routing necessary.
