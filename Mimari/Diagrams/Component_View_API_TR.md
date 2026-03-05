# Bileşen Görünümü — API (Türkçe)

C4 Seviye 3: API konteyneri içindeki ana bileşenler. Mermaid.

---

## Diyagram

```mermaid
flowchart TB
    subgraph API["API - api.verasaha.com"]
        subgraph Presentation["Sunum"]
            AuthC[Auth Controller\ndiscover, login]
            SyncC[Sync Controller\nchanges, apply]
            FileC[File Controller\nyükleme, imzalı URL]
            ApiC[API Controller'lar\nCRUD, kiracı kapsamında]
        end

        subgraph Application["Uygulama"]
            MediatR[MediatR / CQRS]
            TenantMW[Tenant Middleware\nX-Tenant-Key, JWT]
            CorrMW[CorrelationId Middleware]
            RateLimit[Rate Limiter\nauth vs write]
        end

        subgraph Domain["Domain"]
            DomainModel[Domain Model\nMeeting, vb.]
            AuditLog[AuditLog\ninterceptor]
        end

        subgraph Persistence["Kalıcılık"]
            DbContext[DbContext\nGlobal query filter]
            SyncInbox[SyncInbox\nTenantId, UserId, OpId]
            NotifOutbox[NotificationOutbox]
        end

        subgraph Infrastructure["Altyapı"]
            FileStore[Dosya Deposu\nkiracı yolları]
            Signing[URL İmzalama\nHMAC, expiry]
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

## Bileşenler (kısa)

- **Sunum:** Auth (discover, login), Sync (changes, apply), File (yükleme, imzalı URL), API CRUD; uygulanabilir yerde kiracı kapsamında.
- **Middleware:** CorrelationId; Kiracı çözümleme (X-Tenant-Key) ve JWT tenant bağlama; rate limiting (auth vs write).
- **Uygulama:** MediatR/CQRS; use case’ler domain ve kalıcılığı orkestre eder.
- **Domain:** Aggregate’ler (örn. Meeting); SaveChanges interceptor ile AuditLog.
- **Kalıcılık:** Global TenantId filter’lı DbContext; SyncInbox (idempotency); NotificationOutbox.
- **Altyapı:** Dosya deposu (kiracı yolları); CDN için imzalı URL üretimi (HMAC, expiry).
