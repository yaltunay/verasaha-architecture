# Architecture Summary (English)

This file provides a high-level overview of the VeraSaha architecture, aligned with the Turkish summary.

---

## Custom Domain Support

- **Same web app:** A single web application (same build/image) can serve verasaha.com (landing), tenant subdomains (e.g. tenantA.verasaha.com), and approved custom domains (e.g. saha.firma.com). No separate frontend build or deployment per tenant is required.
- **Host roles:** The web host is only the UX/access layer. API is always api.verasaha.com; file serving is always cdn.verasaha.com.
- **Tenant bootstrap:** For tenant subdomains, tenant key is derived from host or confirmed via bootstrap. For custom domains, the frontend resolves host → tenant using a bootstrap mechanism; then sends X-Tenant-Key to api.verasaha.com.
- **Reverse proxy / DNS:** Custom domains are onboarded by DNS to the same reverse proxy, TLS provisioning, and host mapping to the same web upstream—not by separate app deployments.
- **Load balancer:** The model is compatible with horizontal scaling; tenant context is resolved at runtime from host/bootstrap, not from tenant-specific builds.
