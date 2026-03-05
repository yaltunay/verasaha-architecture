# Privacy by Design (English)

*This document is provided for architectural reference only and does not constitute legal advice or a formal security certification.*

---

## Privacy-First Architecture

VeraSaha considers privacy from the design stage: data minimization, access control, transparency, and auditability are supported by architectural decisions. Personal data is processed only in the required context (tenant and user scope); cross-tenant or unauthorized access is structurally prevented.

---

## Tenant Isolation

- Every request is processed in a single tenant context via **X-Tenant-Key** and **JWT tenantId** matching.
- **EF Core global query filter** ensures database queries are automatically filtered by TenantId; application code cannot accidentally return another tenant’s data.
- Sync and notification queues are tenant- and user-scoped; one tenant’s data does not leak to another.

---

## Secure Uploads

- File upload endpoints are protected by **JWT and X-Tenant-Key**; only authenticated users in the correct tenant context can upload.
- **Tenant storage quota** limits abuse and overuse; **file type whitelist** and (when configured) **magic number** checks reduce risk.
- Uploaded files are stored in tenant and user context; access is only via signed URLs.

---

## Signed URLs

- File download is performed via **signed URLs**; each URL contains tenantId, expiry, and HMAC signature.
- The CDN validates expiry and signature on every request; invalid or expired URLs return 403. There is no direct listing or unauthenticated access; privacy and data minimization are preserved.

---

## Audit Logging

- All entity changes are written to an **AuditLog** table; who performed which action on which entity and when is recorded.
- Sensitive field values (passwords, tokens, full PII) are not stored in audit records; only the minimum information needed for audit and compliance is retained.

---

## Log Redaction

- **Sensitive data** (passwords, tokens, PII) is not written to application logs; a redaction/masking policy is applied.
- Structured logging uses CorrelationId, TenantId, and UserId for context; diagnostics and audit remain possible while privacy is protected.
