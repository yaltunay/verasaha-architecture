# System Context — VeraSaha (English)

C4 Level 1: System Context. Text diagram using Mermaid.

---

## Diagram

```mermaid
flowchart LR
    subgraph Users
        U1[Field User]
        U2[Admin]
        U3[Tenant User]
    end

    subgraph VeraSaha["VeraSaha Platform"]
        direction TB
        RP[Reverse Proxy\nNginx/NPM + TLS]
        RP --> Landing[verasaha.com\nLanding + Login]
        RP --> App[app.verasaha.com\nPanel]
        RP --> API[api.verasaha.com\nBackend API]
        RP --> CDN[cdn.verasaha.com\nSigned URLs]
        RP --> TenantUI["{tenant}.verasaha.com\nTenant UI"]
    end

    Mobile[Mobile App\nFlutter - Offline Sync]

    U1 --> Mobile
    U1 --> TenantUI
    U2 --> App
    U3 --> TenantUI
    Mobile --> API
    TenantUI --> API
    App --> API
    TenantUI --> CDN
    Mobile --> CDN
```

---

## Elements

- **Users:** Field user (mobile + tenant UI), admin (panel), tenant user (browser).
- **Reverse proxy:** Nginx or NPM; TLS termination; routes by host to landing, app, API, CDN, tenant UI.
- **Landing:** verasaha.com — discovery and login entry.
- **Panel:** app.verasaha.com — admin/management.
- **Backend API:** api.verasaha.com — single API; X-Tenant-Key + JWT.
- **CDN:** cdn.verasaha.com — file serving via signed URLs only.
- **Tenant UI:** {tenant}.verasaha.com — tenant-specific web UI.
- **Mobile:** Flutter client; offline-first; sync with API.
