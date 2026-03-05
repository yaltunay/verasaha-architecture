# Container View — VeraSaha (English)

C4 Level 2: Containers (deployable units). Mermaid.

---

## Diagram

```mermaid
flowchart TB
    subgraph Edge["Edge"]
        RP[Reverse Proxy\nNginx / NPM\nTLS termination\nHost-based routing]
    end

    subgraph Web["Web Applications"]
        Landing[Landing\nverasaha.com\nNext.js]
        App[Admin Panel\napp.verasaha.com\nAngular]
        TenantUI[Tenant UI\n{tenant}.verasaha.com]
    end

    subgraph Backend["Backend"]
        API[API\napi.verasaha.com\nASP.NET Core]
        Worker[Notification Worker\nOutbox processor]
    end

    subgraph Storage["Storage"]
        DB[(Database\nTenant-scoped\nEF Core)]
        Blob[Storage Volume\nBlob / Files\nTenant paths]
    end

    CDN[CDN / File Server\ncdn.verasaha.com\nSigned URL validation]
    Mobile[Mobile Client\nFlutter\nOffline-first]

    RP --> Landing
    RP --> App
    RP --> TenantUI
    RP --> API
    RP --> CDN

    API --> DB
    API --> Blob
    API --> CDN
    Worker --> DB
    Mobile --> API
    Mobile --> CDN
    TenantUI --> API
    TenantUI --> CDN
    App --> API
```

---

## Containers

- **Reverse proxy:** Nginx or NPM; TLS; routes to landing, app, API, CDN, tenant UI.
- **Landing:** verasaha.com; discovery and login start.
- **Admin panel:** app.verasaha.com.
- **Tenant UI:** Per-tenant web UI.
- **API:** Single backend; X-Tenant-Key; JWT; global query filters; SyncInbox, NotificationOutbox.
- **Notification worker:** Reads Outbox; sends to FCM (or noop in dev).
- **Database:** Tenant-scoped data; EF Core.
- **Storage volume:** Uploaded files; tenant-scoped paths; CDN serves via signed URLs.
- **CDN:** Validates signature and expiry; serves file bytes.
- **Mobile:** Flutter; offline-first; sync with API; signed URLs for files.
