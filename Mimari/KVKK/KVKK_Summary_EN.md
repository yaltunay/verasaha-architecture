# KVKK Summary (English)

*This document is provided for architectural reference only and does not constitute legal advice or a formal security certification.*

---

## Overview

The VeraSaha architecture is designed to support personal data protection and alignment with privacy regulations (e.g. KVKK). This summary covers multi-tenant data separation, audit logging, secure file access, secrets isolation, retention awareness, and readiness for data subject requests (DSR).

---

## Multi-Tenant Data Separation

- All tenant data is separated by **TenantId**; EF Core global query filter ensures queries are automatically scoped.
- **X-Tenant-Key** is required on API requests; if it does not match the **tenantId** claim in the JWT, the API returns 403. Cross-tenant data access is prevented at the application and data layers.
- Sync and notification processing (SyncInbox, NotificationOutbox) are tenant- and user-scoped; no mixing of tenant data.

---

## Audit Logging

- All entity changes are written to an **AuditLog** table: TenantId, UserId, Action, EntityName, EntityId, CorrelationId, TimestampUtc, etc.
- **Sensitive field values** are not written to the audit log; only who performed which action on which entity and when is recorded. This supports audit trails and incident review and helps meet requirements for processing records under privacy regulations.

---

## Secure File Access

- File access is provided only via **signed URLs**; tenantId, expiry, and HMAC signature are validated by the CDN.
- There is no unauthenticated listing or directory access; only time-limited signed URLs issued by the API are valid. Data minimization and access control are applied at the architecture level.

---

## Secrets Isolation

- JWT secret, database connection details, CDN signing key, etc. are **not** stored in source code or logs; they are managed via .env or environment variables.
- In production, secrets are injected via a secure secret store or CI/CD. This protects system access credentials used in personal data processing.

---

## Retention Awareness

- The architecture supports applying **retention policy** to audit and processing records (e.g. timestamped records, filtering by TenantId).
- Data deletion or anonymization is defined by business rules and operational procedures; the technical base is suitable for time-based purge or archiving.

---

## DSR Readiness (Data Subject Requests)

- For **access, rectification, erasure, portability** requests: Data can be mapped comprehensively by TenantId and UserId; AuditLog and tenant-scoped queries support locating data and processing history.
- CorrelationId and structured logging make it easier to trace a given operation or user session; DSR response processes can be supported with this data.

---

## Operational Readiness (Not Legal Advice)

This section describes implementation status and placeholders for operational procedures. It does not constitute legal advice; retention and DSR processes must be defined with legal and compliance review.

### Implemented vs planned

| Area | Implemented | Planned / placeholder |
|------|-------------|------------------------|
| Tenant data separation | Yes (TenantId, global filter, X-Tenant-Key + JWT) | — |
| Audit logging | Yes (AuditLog table, no sensitive values) | Retention period to be set by policy |
| Secure file access | Yes (signed URLs, no listing) | — |
| Secrets isolation | Yes (.env, no secrets in code/logs) | Rotation procedure to be documented |
| DSR data mapping | Technical capability (TenantId, UserId, AuditLog) | Export/delete runbooks require legal review |

### Retention schedule (placeholder)

| Data category | Retention (example) | Owner / review |
|---------------|---------------------|-----------------|
| Audit log | To be defined by policy | Legal / DPO |
| User/tenant data | To be defined by policy | Legal / DPO |
| SyncInbox / Outbox | Operational; purge policy TBD | Operations |

*Actual retention must be set by the data controller with legal and KVKK compliance review.*

### DSR process (placeholder)

- **Export (access / portability):** Technically supported by TenantId + UserId queries and AuditLog. Exact format, scope, and delivery process **require legal review** before commitment.
- **Erasure (delete):** Technical deletion or anonymization can be implemented per tenant/user; cascade and retention exceptions (e.g. legal hold) **require legal review**.
- **Rectification:** Supported by normal update flows; audit trail exists. Process and timelines **require legal review**.

*Do not treat this as a compliance checklist; engage legal and DPO for DSR procedures and retention.*
