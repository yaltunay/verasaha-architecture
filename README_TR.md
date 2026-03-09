# VeraSaha Mimarisi

Kurumsal seviyede çok kiracılı (multi-tenant) SaaS mimari vaka çalışması.

---

## Bu repo ne / ne değil

- **Bu repo:** Yalnızca mimari ve operasyon **dokümantasyonudur**. VeraSaha platformu için tasarım kararları, trade-off’lar, production sertleştirme, güvenlik duruşu ve gizlilik odaklı tasarım burada dokümante edilir. Uygulama kaynak kodu, yapılandırma dosyası veya altyapı script’i bulunmaz.
- **Bu repo değildir:** Kod tabanı, demo uygulama veya hukuki/güvenlik sertifikası değildir. Sistem tasarımı için teknik portföy ve referans niteliğindedir.

---

## Host Topolojisi

| Host | Rol |
|------|-----|
| **verasaha.com** | Landing (Next.js); keşif ve giriş noktası. |
| **app.verasaha.com** | Admin / panel (Angular). |
| **api.verasaha.com** | Tek backend API; kiracı bağlamı `X-Tenant-Key` başlığı ile. |
| **cdn.verasaha.com** | İmzalı URL (HMAC) ile dosya sunumu; anonim erişim yok. |
| **{tenant}.verasaha.com** | Kiracıya özel UI host’ları (örn. firma.verasaha.com). |
| **Mobile (Flutter)** | Çevrimdışı öncelikli senkron istemcisi. |

**Rezerve subdomain’ler:** `www`, `api`, `admin`, `app`, `debug`, `test`.  
**Reverse proxy:** Nginx (veya NPM) + Let's Encrypt (TLS).

Aynı web uygulaması landing domainini, tenant subdomainlerini ve onaylı özel domainleri sunabilir. Tenant davranışı çalışma anında çözülür; ayrı build gerektirmez.

---

## Temel Garantiler

- **Katı kiracı izolasyonu:** EF Core global query filter’lar + kiracı kapsamında SyncInbox/NotificationOutbox; normal akışlarda çapraz kiracı veri yok.
- **Katı JWT tenant bağlama:** Token `tenantId` ile `X-Tenant-Key` eşleşmek zorunda; eşleşmezse → 403.
- **Rate limiting:** Auth (discover/login) IP bazlı; yazma trafiği kullanıcı veya tenant+IP bazlı; brute force ve kötüye kullanım azaltılır.
- **Audit log + correlation:** Tüm mutasyonlar loglanır (kim, ne zaman, hangi varlık); her istekte CorrelationId ile izleme.
- **Güvenli yüklemeler + kota:** Dosya türü whitelist, isteğe bağlı magic number kontrolü; kiracı depolama kotası (aşımda 409).
- **Çevrimdışı senkron idempotency:** SyncInbox (TenantId, UserId, OpId) ile apply idempotent; yeniden denemeler güvenli.
- **Outbox bildirimleri:** NotificationOutbox + worker; en az bir kez dağıtım, retry/backoff; istek yolunda senkron FCM yok.

---

## Dokümanları nasıl okursunuz

1. **Başlangıç:** [Mimari/Architecture_Summary/TR.md](Mimari/Architecture_Summary/TR.md) ile büyük resim.
2. **Derinlemesine:** [Mimari/Architecture_Detailed/TR.md](Mimari/Architecture_Detailed/TR.md) ile gerekçe, trade-off’lar ve reddedilen alternatifler.
3. **Güvenlik:** [Mimari/Security/Security_Architecture_TR.md](Mimari/Security/Security_Architecture_TR.md) ve [Mimari/Security/Threat_Model_TR.md](Mimari/Security/Threat_Model_TR.md).
4. **KVKK / Gizlilik:** [Mimari/KVKK/KVKK_Summary_TR.md](Mimari/KVKK/KVKK_Summary_TR.md) ve [Mimari/KVKK/Data_Protection_Model_TR.md](Mimari/KVKK/Data_Protection_Model_TR.md).
5. **Kararlar:** [Mimari/ADR/ADR_INDEX_TR.md](Mimari/ADR/ADR_INDEX_TR.md) ile Mimari Karar Kayıtları.

---

## Dokümantasyon (Mimari/ altında)

| Alan | Özet | Detay / Diğer |
|------|------|----------------|
| **Architecture** | [Architecture_Summary/TR.md](Mimari/Architecture_Summary/TR.md), [EN.md](Mimari/Architecture_Summary/EN.md) | [Architecture_Detailed/](Mimari/Architecture_Detailed/) |
| **Security** | [Security_Architecture_TR.md](Mimari/Security/Security_Architecture_TR.md), [EN](Mimari/Security/Security_Architecture_EN.md) | [Threat_Model_TR.md](Mimari/Security/Threat_Model_TR.md), [EN](Mimari/Security/Threat_Model_EN.md) |
| **KVKK / Privacy** | [KVKK_Summary_TR.md](Mimari/KVKK/KVKK_Summary_TR.md), [EN](Mimari/KVKK/KVKK_Summary_EN.md) | [Data_Protection_Model_TR.md](Mimari/KVKK/Data_Protection_Model_TR.md), [EN](Mimari/KVKK/Data_Protection_Model_EN.md) |
| **ADR** | [ADR_INDEX_TR.md](Mimari/ADR/ADR_INDEX_TR.md), [EN](Mimari/ADR/ADR_INDEX_EN.md) | ADR-001 … ADR-007 tek tek |

Ayrıca: Mimari/Functional_*, Mimari/Operations_*, Mimari/Diagrams/, Mimari/Portfolio_OnePager_*.md.

---

## Mimari Katmanlar

Dokümantasyon şu alanlara ayrılır: **Architecture** (sistem tasarımı, topoloji, kalıplar), **Functional** (yetenekler, özellikler), **Operations** (dağıtım, gözlemlenebilirlik, production hazırlığı), **Security** (kimlik doğrulama, izolasyon, tehdit modeli), **KVKK / Privacy** (veri koruma, DSR farkındalığı), **ADR** (kayıtlı kararlar). Amaç: güvenlik, operasyonel olgunluk ve gizlilik odaklı tasarım ile production seviyesinde SaaS düşüncesini göstermek.

---

## Yazar

Yücel ALTUNAY
