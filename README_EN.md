# VeraSaha Architecture

Enterprise-grade multi-tenant SaaS architecture case study.

---

## What this repo is / is not

- **This repo is:** Architecture and operations **documentation only**. It documents design decisions, trade-offs, production hardening, security posture, and privacy-aware design for the VeraSaha platform. No application source code, config files, or infrastructure scripts are stored here.
- **This repo is not:** A codebase, a demo app, or legal/security certification. It is a technical portfolio and reference for system design.

---

## Host Topology

| Host | Role |
|------|------|
| **verasaha.com** | Landing (Next.js); discovery and login entry. |
| **app.verasaha.com** | Admin / panel (Angular). |
| **api.verasaha.com** | Single backend API; tenant context via `X-Tenant-Key` header. |
| **cdn.verasaha.com** | File serving via signed URLs (HMAC); no anonymous access. |
| **{tenant}.verasaha.com** | Tenant-specific UI hosts (e.g. firma.verasaha.com). |
| **Mobile (Flutter)** | Offline-first sync client. |

**Reserved subdomains:** `www`, `api`, `admin`, `app`, `debug`, `test`.  
**Reverse proxy:** Nginx (or NPM) + Let's Encrypt for TLS.

The same web application can serve the landing domain, tenant subdomains, and approved custom domains. Tenant-specific behavior is resolved at runtime; it does not require separate builds.

---

## Key Guarantees

- **Strict tenant isolation:** EF Core global query filters + tenant-scoped SyncInbox/NotificationOutbox; no cross-tenant data in normal flows.
- **Strict JWT tenant binding:** Token `tenantId` must match `X-Tenant-Key`; mismatch → 403.
- **Rate limiting:** Auth (discover/login) IP-based; write traffic per user or tenant+IP; reduces brute force and abuse.
- **Audit log + correlation:** All mutations logged (who, when, which entity); CorrelationId on every request for tracing.
- **Secure uploads + quota:** File type whitelist, optional magic-number check; tenant storage quota enforced (409 on exceed).
- **Offline sync idempotency:** SyncInbox (TenantId, UserId, OpId) ensures apply is idempotent; retries safe.
- **Outbox notifications:** NotificationOutbox + worker; at-least-once dispatch, retry/backoff; no sync FCM in request path.

---

## How to read the docs

1. **Start here:** [Mimari/Architecture_Summary/EN.md](Mimari/Architecture_Summary/EN.md) for the big picture.
2. **Deep dive:** [Mimari/Architecture_Detailed/EN.md](Mimari/Architecture_Detailed/EN.md) for rationale, trade-offs, and rejected alternatives.
3. **Security:** [Mimari/Security/Security_Architecture_EN.md](Mimari/Security/Security_Architecture_EN.md) and [Mimari/Security/Threat_Model_EN.md](Mimari/Security/Threat_Model_EN.md).
4. **Privacy / KVKK:** [Mimari/KVKK/KVKK_Summary_EN.md](Mimari/KVKK/KVKK_Summary_EN.md) and [Mimari/KVKK/Data_Protection_Model_EN.md](Mimari/KVKK/Data_Protection_Model_EN.md).
5. **Decisions:** [Mimari/ADR/ADR_INDEX_EN.md](Mimari/ADR/ADR_INDEX_EN.md) for Architecture Decision Records.

---

## Documentation (under Mimari/)

| Area | Summary | Detailed / Other |
|------|---------|-------------------|
| **Architecture** | [Architecture_Summary/EN.md](Mimari/Architecture_Summary/EN.md), [TR.md](Mimari/Architecture_Summary/TR.md) | [Architecture_Detailed/](Mimari/Architecture_Detailed/) |
| **Security** | [Security_Architecture_EN.md](Mimari/Security/Security_Architecture_EN.md), [TR](Mimari/Security/Security_Architecture_TR.md) | [Threat_Model_EN.md](Mimari/Security/Threat_Model_EN.md), [TR](Mimari/Security/Threat_Model_TR.md) |
| **KVKK / Privacy** | [KVKK_Summary_EN.md](Mimari/KVKK/KVKK_Summary_EN.md), [TR](Mimari/KVKK/KVKK_Summary_TR.md) | [Data_Protection_Model_EN.md](Mimari/KVKK/Data_Protection_Model_EN.md), [TR](Mimari/KVKK/Data_Protection_Model_TR.md) |
| **ADR** | [ADR_INDEX_EN.md](Mimari/ADR/ADR_INDEX_EN.md), [TR](Mimari/ADR/ADR_INDEX_TR.md) | Individual ADR-001 … ADR-007 |

Also: Mimari/Functional_*, Mimari/Operations_*, Mimari/Diagrams/, Mimari/Portfolio_OnePager_*.md.

---

## Architecture Layers

The documentation is organized into: **Architecture** (system design, topology, patterns), **Functional** (capabilities, features), **Operations** (deployment, observability, production readiness), **Security** (auth, isolation, threat model), **KVKK / Privacy** (data protection, DSR awareness), and **ADR** (recorded decisions). These aim to show production-grade SaaS thinking with security, operational maturity, and privacy-aware design.

---

## Author

Yücel ALTUNAY
