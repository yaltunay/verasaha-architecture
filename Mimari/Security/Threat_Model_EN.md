# Threat Model (English)

*This document is provided for architectural reference only and does not constitute legal advice or a formal security certification.*

---

## Threat Table

| Threat | Attack Scenario | Mitigation (VeraSaha Architecture) |
|--------|-----------------|-------------------------------------|
| **Cross-tenant data access** | An attacker attempts to access another tenant’s data via the API or database. | Mandatory X-Tenant-Key; JWT tenantId claim must match (403 on mismatch); EF Core global query filter ensures all queries are scoped by TenantId; Sync/Outbox are tenant-scoped. |
| **Token theft** | A JWT is stolen and used in another tenant’s or user’s context. | JWT tenantId must match X-Tenant-Key; token is rejected in a different tenant context. Short-lived tokens and HTTPS reduce exposure. |
| **Replay attacks** | The same request is replayed to repeat an operation. | Sync idempotency via (TenantId, UserId, OpId); duplicate OpId returns the previous result; mutations are not applied twice. |
| **Direct file access** | Storage or CDN URLs are guessed to download files without authentication. | File access only via signed URLs; tenantId, expiry, and HMAC are validated. No listing or directory access; invalid or expired URL returns 403. |
| **Sensitive data exposure in logs** | Passwords, tokens, or PII are written to logs and become accessible. | Redaction/masking policy; AuditLog does not store sensitive field values; structured logging excludes secrets. |
| **API abuse / brute force** | High volume of requests against login or write endpoints. | IP-based rate limiting for auth (discover/login); per-user or tenant+IP limits for writes; reduces impact of DDoS and brute force. |
| **Unauthorized uploads** | Malicious or excessive uploads; wrong file type or spoofed extension. | JWT and X-Tenant-Key required on protected upload endpoints; tenant storage quota (409 on exceed); file type whitelist and magic-number sniffing. |
| **CDN link sharing** | Signed URL is shared so anyone can access the file until expiry. | URL includes mandatory expiry (Unix timestamp); after expiry, CDN returns 403. Sharing is intentionally time-limited; no permanent public links. |
| **Privilege escalation** | User attempts to gain another role or tenant’s privileges. | JWT carries tenantId and role; server validates tenant and permissions on every request; EF filter enforces tenant isolation at the data layer. |
| **Secrets exposure** | JWT secret, DB password, or CDN signing key appears in source or logs. | Secrets supplied via .env/environment variables; no hardcoded secrets in source; not written to logs; in production, injected via secure secret store or CI. |

---

## Summary

The VeraSaha architecture provides defensive layers against the above threats through multi-tenant isolation (X-Tenant-Key, JWT tenant binding, global query filter), signed CDN URLs, rate limiting, idempotent sync, log redaction, and keeping secrets out of code and logs. This document summarizes architectural decisions; it is not a formal security certification or legal advice.
