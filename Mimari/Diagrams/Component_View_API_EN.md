# Component View — API (English)

C4 Level 3: Main components inside the API container. Mermaid.

---

## Diagram

```mermaid
flowchart TB
    subgraph API["API - api.verasaha.com"]
        subgraph Presentation["Presentation"]
            AuthC[Auth Controller\ndiscover, login]
            SyncC[Sync Controller\nchanges, apply]
            FileC[File Controller\nupload, signed URL]
            ApiC[API Controllers\nCRUD, tenant-scoped]
        end

        subgraph Application["Application"]
            MediatR[MediatR / CQRS]
            TenantMW[Tenant Middleware\nX-Tenant-Key, JWT]
            CorrMW[CorrelationId Middleware]
            RateLimit[Rate Limiter\nauth vs write]
        end

        subgraph Domain["Domain"]
            DomainModel[Domain Model\nMeeting, etc.]
            AuditLog[AuditLog\ninterceptor]
        end

        subgraph Persistence["Persistence"]
            DbContext[DbContext\nGlobal query filter]
            SyncInbox[SyncInbox\nTenantId, UserId, OpId]
            NotifOutbox[NotificationOutbox]
        end

        subgraph Infrastructure["Infrastructure"]
            FileStore[File Store\ntenant paths]
            Signing[URL Signing\nHMAC, expiry]
        end
    end

    TenantMW --> MediatR
    CorrMW --> TenantMW
    RateLimit --> AuthC
    AuthC --> MediatR
    SyncC --> MediatR
    FileC --> MediatR
    ApiC --> MediatR
    MediatR --> DomainModel
    MediatR --> DbContext
    MediatR --> SyncInbox
    MediatR --> NotifOutbox
    MediatR --> FileStore
    MediatR --> Signing
    DbContext --> AuditLog
```

---

## Components (concise)

- **Presentation:** Auth (discover, login), Sync (changes, apply), File (upload, signed URL), API CRUD; all tenant-scoped where applicable.
- **Middleware:** CorrelationId; Tenant resolution (X-Tenant-Key) and JWT tenant binding; rate limiting (auth vs write).
- **Application:** MediatR/CQRS; use cases orchestrate domain and persistence.
- **Domain:** Aggregates (e.g. Meeting); AuditLog via SaveChanges interceptor.
- **Persistence:** DbContext with global TenantId filter; SyncInbox (idempotency); NotificationOutbox.
- **Infrastructure:** File store (tenant paths); signed URL generation (HMAC, expiry) for CDN.
