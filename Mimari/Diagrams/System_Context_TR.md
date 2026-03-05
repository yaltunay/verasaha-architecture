# Sistem Bağlamı — VeraSaha (Türkçe)

C4 Seviye 1: Sistem bağlamı. Mermaid ile metin diyagramı.

---

## Diyagram

```mermaid
flowchart LR
    subgraph Users["Kullanıcılar"]
        U1[Saha Kullanıcısı]
        U2[Admin]
        U3[Kiracı Kullanıcısı]
    end

    subgraph VeraSaha["VeraSaha Platformu"]
        direction TB
        RP[Reverse Proxy\nNginx/NPM + TLS]
        RP --> Landing[verasaha.com\nLanding + Giriş]
        RP --> App[app.verasaha.com\nPanel]
        RP --> API[api.verasaha.com\nBackend API]
        RP --> CDN[cdn.verasaha.com\nİmzalı URL]
        RP --> TenantUI["{tenant}.verasaha.com\nKiracı UI"]
    end

    Mobile[Mobil Uygulama\nFlutter - Çevrimdışı Senkron]

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

## Öğeler

- **Kullanıcılar:** Saha kullanıcısı (mobil + kiracı UI), admin (panel), kiracı kullanıcısı (tarayıcı).
- **Reverse proxy:** Nginx veya NPM; TLS sonlandırma; host’a göre landing, app, API, CDN, kiracı UI’a yönlendirme.
- **Landing:** verasaha.com — keşif ve giriş noktası.
- **Panel:** app.verasaha.com — yönetim.
- **Backend API:** api.verasaha.com — tek API; X-Tenant-Key + JWT.
- **CDN:** cdn.verasaha.com — yalnızca imzalı URL ile dosya sunumu.
- **Kiracı UI:** {tenant}.verasaha.com — kiracıya özel web arayüzü.
- **Mobil:** Flutter istemcisi; çevrimdışı öncelikli; API ile senkron.
