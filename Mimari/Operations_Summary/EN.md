# Operations Summary (English)

High-level operations overview for the VeraSaha platform, aligned with the Turkish summary.

---

## Custom Domain Support

- The same web application serves verasaha.com, tenant subdomains, and approved custom domains; no per-tenant build or deployment.
- Custom domains are onboarded via DNS → same reverse proxy, TLS provisioning, and host mapping to the same web upstream (not separate deployments).
- Tenant context is resolved at runtime (host/bootstrap); the model is load-balancer friendly and horizontally scalable.
