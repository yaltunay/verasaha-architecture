# Mimari Karar Kayıtları — İndeks (Türkçe)

Bu indeks, VeraSaha platformu için tutulan tüm Mimari Karar Kayıtlarını (ADR) listeler.

| ADR No   | Başlık                              | Kısa açıklama |
|----------|-------------------------------------|----------------|
| ADR-001  | Çok Kiracılı İzolasyon Modeli      | SaaS çok kiracılı mimari, kiracı veri ayrımı, global sorgu filtreleri, kiracıya bağlı işlemler. |
| ADR-002  | Başlık Tabanlı Kiracı Çözümlemesi   | X-Tenant-Key başlığı kullanımı, API host modeli, JWT tenant bağlama, çapraz kiracı erişiminin önlenmesi. |
| ADR-003  | İmzalı CDN URL’leri                | Sunucuda saklanan dosya yüklemeleri, CDN sunum modeli, imzalı URL’ler, süre ve imza doğrulaması. |
| ADR-004  | Bildirimler için Outbox Paterni     | Arka plan bildirimleri, güvenilir mesaj dağıtımı, yeniden deneme stratejisi, eventual consistency. |
| ADR-005  | Çevrimdışı Senkronizasyon Mimarisi | Çevrimdışı öncelikli mobil istemciler, delta senkronizasyonu, idempotent apply uç noktası, sync inbox paterni. |
| ADR-006  | Reverse Proxy Dağıtımı             | Reverse proxy katmanı, TLS sonlandırma, dahili servis açığa çıkarma, host tabanlı yönlendirme. |
| ADR-007  | Gözlemlenebilirlik Yığını          | Yapılandırılmış loglama, metrikler, izleme, merkezi gözlemlenebilirlik. |

---

## Doküman listesi

- [ADR-001_Multi_Tenant_Isolation_TR.md](ADR-001_Multi_Tenant_Isolation_TR.md)
- [ADR-002_Tenant_Header_TR.md](ADR-002_Tenant_Header_TR.md)
- [ADR-003_Signed_CDN_URLs_TR.md](ADR-003_Signed_CDN_URLs_TR.md)
- [ADR-004_Outbox_Pattern_TR.md](ADR-004_Outbox_Pattern_TR.md)
- [ADR-005_Offline_Sync_TR.md](ADR-005_Offline_Sync_TR.md)
- [ADR-006_Reverse_Proxy_TR.md](ADR-006_Reverse_Proxy_TR.md)
- [ADR-007_Observability_TR.md](ADR-007_Observability_TR.md)
