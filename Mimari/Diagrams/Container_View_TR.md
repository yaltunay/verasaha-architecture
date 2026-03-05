# Container Görünümü — VeraSaha (Türkçe)

C4 Seviye 2: Konteynerler (dağıtılabilir birimler). Mermaid.

---

## Diyagram

```mermaid
flowchart TB
    subgraph Edge["Kenar"]
        RP[Reverse Proxy\nNginx / NPM\nTLS sonlandırma\nHost tabanlı yönlendirme]
    end

    subgraph Web["Web Uygulamaları"]
        Landing[Landing\nverasaha.com\nNext.js]
        App[Admin Panel\napp.verasaha.com\nAngular]
        TenantUI[Kiracı UI\n{tenant}.verasaha.com]
    end

    subgraph Backend["Backend"]
        API[API\napi.verasaha.com\nASP.NET Core]
        Worker[Bildirim Worker\nOutbox işleyici]
    end

    subgraph Storage["Depolama"]
        DB[(Veritabanı\nKiracı kapsamında\nEF Core)]
        Blob[Depolama Birimi\nBlob / Dosyalar\nKiracı yolları]
    end

    CDN[CDN / Dosya Sunucu\ncdn.verasaha.com\nİmzalı URL doğrulama]
    Mobile[Mobil İstemci\nFlutter\nÇevrimdışı öncelikli]

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

## Konteynerler

- **Reverse proxy:** Nginx veya NPM; TLS; landing, app, API, CDN, kiracı UI’a yönlendirme.
- **Landing:** verasaha.com; keşif ve giriş başlangıcı.
- **Admin panel:** app.verasaha.com.
- **Kiracı UI:** Kiracıya özel web arayüzü.
- **API:** Tek backend; X-Tenant-Key; JWT; global query filter’lar; SyncInbox, NotificationOutbox.
- **Bildirim worker:** Outbox’ı okur; FCM’e gönderir (veya dev’de noop).
- **Veritabanı:** Kiracı kapsamında veri; EF Core.
- **Depolama birimi:** Yüklenen dosyalar; kiracı yolları; CDN imzalı URL ile sunar.
- **CDN:** İmza ve süre doğrulama; dosya baytları sunar.
- **Mobil:** Flutter; çevrimdışı öncelikli; API ile senkron; dosyalar için imzalı URL.
