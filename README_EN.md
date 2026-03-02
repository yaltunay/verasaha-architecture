# VeraSaha Architecture

Enterprise-grade multi-tenant SaaS architecture case study.

This repository contains **architecture & operations documentation only** (no source code).  
It documents the design decisions, trade-offs, and production hardening strategy of the VeraSaha platform.

---

## 🎯 What is VeraSaha?

VeraSaha is a multi-tenant SaaS platform designed for field operations and meeting management, with:

- Multi-tenant isolation (subdomain UX, header-based tenancy on API)
- Offline-first mobile sync (idempotent apply + delta changes)
- Secure uploads + quota enforcement
- CDN-based secure file serving (signed URLs)
- Background notifications (Outbox pattern + worker)
- Observability (structured logs, metrics, tracing)
- Reverse proxy + HTTPS production readiness

---

## 🏗 Host Topology (Current)

- **verasaha.com** → Landing (Next.js) + root login/discovery
- **{tenant}.verasaha.com** → Tenant UI hosts
- **app.verasaha.com** → Admin/Panel (Angular)
- **api.verasaha.com** → Single backend API (tenant via `X-Tenant-Key`)
- **cdn.verasaha.com** → File serving via signed URLs (HMAC)
- **Mobile (Flutter)** → Offline-first sync

**Reserved subdomains:** `www`, `api`, `admin`, `app`, `debug`, `test`  
**Reverse proxy:** Nginx + Let's Encrypt

---

## 🔐 Multi-Tenant Isolation (Guarantees)

Isolation is enforced structurally:

1. **API host model**: `api.verasaha.com`
2. **Mandatory tenant header** on protected endpoints: `X-Tenant-Key`
3. **Strict JWT tenant binding** (401/403 enforcement)
4. **EF Core global query filters**
5. **Tenant-scoped inbox/outbox tables** (idempotency + background processing)
6. **CDN signed URLs** include tenant context and signature checks

---

## 📁 Documentation

All documents live under:

Folders are bilingual (**TR/EN**) and split into summary vs detailed:

- Architecture_Summary
- Architecture_Detailed
- Functional_Summary
- Functional_Detailed
- Operations_Summary
- Operations_Detailed

---

## ✅ How to Use

- Read **Architecture_Summary** first for the big picture.
- Use **Architecture_Detailed** for decision rationales and trade-offs.
- Use **Operations_*** for production readiness and incident workflows.

---

## 👤 Author

Yücel ALTUNAY
