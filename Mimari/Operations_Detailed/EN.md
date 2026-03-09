# Operations Detailed (English)

Operations and production readiness in detail, aligned with the Turkish document.

---

## Custom Domain Support

- **Same web app:** One web application/image serves landing, tenant subdomains, and approved custom domains. No separate frontend build per tenant.
- **Onboarding:** Custom domains: DNS to same reverse proxy/load balancer, TLS provisioning, host mapping to same web upstream. Not separate app deployments.
- **Load balancer:** Tenant context from host/bootstrap at runtime; horizontally scalable; no sticky session or tenant-specific deployment required.
