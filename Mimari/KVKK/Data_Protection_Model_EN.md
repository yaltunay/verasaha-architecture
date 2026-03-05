# Data Protection Model (English)

*This document is provided for architectural reference only and does not constitute legal advice or a formal security certification.*

---

## Architecture Flow Overview

User data is protected across the following layers:

```
Client
       ↓
Reverse Proxy (TLS)
       ↓
API (tenant enforcement)
       ↓
Database (tenant query filters)
       ↓
Storage (signed URL access)
```

---

## Client

- Requests are sent over **HTTPS**; TLS is terminated at the reverse proxy.
- Authenticated requests carry a **JWT** Bearer token and **X-Tenant-Key** header; if the client sends the wrong tenant header, the server returns 403. Tokens and tenant information should be stored securely on the client (e.g. secure storage / session).

---

## Reverse Proxy (TLS)

- **Nginx + Let’s Encrypt** provide TLS termination; api.verasaha.com, cdn.verasaha.com, and app.verasaha.com are served over HTTPS.
- Traffic is forwarded to the API over an encrypted channel; headers such as **X-Forwarded-For** and **X-Forwarded-Proto** are passed to the API with a trusted-proxy configuration. This layer provides transport security and correct client identification.

---

## API (Tenant Enforcement)

- Protected endpoints require **X-Tenant-Key**; reserved or invalid values return 400/404.
- **JWT** is validated; the **tenantId** claim is compared with the tenant resolved from X-Tenant-Key; on mismatch, the API returns 403. Thus user data is processed only in the correct tenant context.
- Rate limiting, CORS, and input validation reduce API abuse and unauthorized access. User data is protected at the API layer by tenant and authorization checks.

---

## Database (Tenant Query Filters)

- All tenant-scoped queries run with an EF Core **global query filter** on **TenantId**; even without explicit filters in application code, cross-tenant data is not returned.
- **AuditLog** records changes; sensitive field values are not stored. At the database layer, data is limited to rows belonging to the tenant; data integrity and isolation are enforced at this layer.

---

## Storage (Signed URL Access)

- File content is **not streamed through the API**; access is only via **signed URLs**.
- Signed URLs contain **tenantId**, **expiry**, and **HMAC**; the CDN validates each request. Invalid or expired URLs return 403; there is no direct storage URL or unauthenticated listing. User files are protected at the storage layer by tenant- and time-limited access.
