# ADR-003 Signed CDN URLs

*This document records architectural decisions for the VeraSaha platform and is intended for technical reference only.*

## Status

Accepted

## Context

The platform needs to store and serve user-uploaded files (e.g. meeting attachments). Streaming all file content through the API would consume significant memory and CPU and would not scale well. We need a model where files are stored durably, served efficiently, and accessed only by authorised users in the correct tenant context. Direct storage URLs or unauthenticated listing must not be exposed.

## Decision Drivers

1. **Scale and performance:** API must not stream file bytes; CDN offloads bandwidth and provides caching; file access must remain tenant- and time-scoped.
2. **Security:** No anonymous or long-lived URLs; tenant cannot access another tenant's files; signature binds tenant and resource.
3. **Operational simplicity:** Stateless CDN validation (HMAC + expiry); no per-request auth callback to API.

## Decision

We use **signed CDN URLs** for file delivery:

- **File uploads stored on server:** Uploads are accepted by the API (with JWT and X-Tenant-Key). Files are stored in tenant- and user-scoped storage (e.g. blob storage or filesystem). The API records metadata and does not stream file bytes to clients on download. Storage is behind the API and CDN; not directly exposed.

- **CDN serving model:** A dedicated CDN host (e.g. cdn.verasaha.com) serves file bytes. The API does not stream file content. The CDN validates each request using a signed URL; only requests with a valid, non-expired signature are served. This offloads bandwidth and benefits from CDN caching and scaling.

- **Signed URLs:** The API generates short-lived signed URLs for specific files. Each URL includes tenantId, expiry (Unix timestamp), and an HMAC signature computed with a shared secret. The CDN (or a small validation layer in front of storage) verifies the signature and expiry on every request. Invalid or expired URLs return 403. No directory listing or unauthenticated access is supported.

- **Expiration and signature validation:** Expiry limits the time window during which a link is valid (e.g. minutes to hours). Signature validation ensures that URLs cannot be forged without the secret and that tenant and resource are bound. This reduces the impact of link sharing and prevents cross-tenant file access when tenants cannot generate valid signatures for each other’s resources.

## Alternatives Considered

- **API streams all file content:** Would require the API to read and stream every download, increasing latency, memory, and CPU. Rejected in favour of CDN-based delivery.

- **Public or long-lived storage URLs:** Would expose files to anyone with the URL and complicate revocation. Rejected.

- **Per-request auth to CDN:** Would require the CDN to call back to the API or an auth service for every request, adding latency and complexity. Signed URLs allow stateless validation at the edge. Rejected for initial design.

## Consequences

- **Positive:** API stays lightweight; file delivery scales via CDN; access is time-limited and tenant-bound; supports security and KVKK data minimization.
- **Positive:** Upload quota and file-type checks remain in the API; no sensitive data in URLs beyond validation needs.
- **Negative:** CDN or validation layer must be deployed and kept in sync with the signing secret; clock skew must be considered for expiry.
- **Negative:** Link sharing is possible until expiry (accepted for usability); secret rotation requires coordination.

## When to Revisit

Revisit if: need for immediate revocation of file access requires per-request auth or short-lived tokens with revocation list; or regulatory requirement for stricter access control (e.g. no link sharing).
