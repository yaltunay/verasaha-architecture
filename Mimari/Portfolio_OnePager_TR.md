# VeraSaha — Mimari Tek Sayfa (Türkçe)

Bu doküman yalnızca portföy ve teknik referans içindir. Hukuki veya güvenlik sertifikası niteliği taşımaz.

---

## Sistem hedefleri ve kısıtlar

VeraSaha, saha operasyonları ve toplantı yönetimi için **çok kiracılı bir SaaS platformudur**. Hedefler: katı kiracı izolasyonu, çevrimdışı öncelikli mobil senkron, güvenli dosya işleme ve production odaklı operasyon (TLS, rate limiting, audit, gözlemlenebilirlik). Kısıtlar: operasyonel basitlik için tek API host (api.verasaha.com); kiracı bağlamı başlık (X-Tenant-Key) ve JWT bağlama ile; API, DB veya worker seviyesinde çapraz kiracı veri erişimi yok.

---

## Topoloji (açıklama)

- **Kenar:** Nginx (veya NPM) reverse proxy; TLS sonlandırma; host tabanlı yönlendirme (landing, API, CDN, app, tenant UI’lar).
- **verasaha.com:** Landing + keşif/giriş başlangıcı.
- **app.verasaha.com:** Admin panel.
- **api.verasaha.com:** Tek backend API; tüm korumalı çağrılar X-Tenant-Key taşır; JWT tenantId eşleşmek zorunda.
- **cdn.verasaha.com:** İmzalı URL (tenantId + expiry + HMAC) ile dosya sunumu; liste veya anonim erişim yok.
- **{tenant}.verasaha.com:** Kiracıya özel web UI.
- **Mobil:** Çevrimdışı öncelikli Flutter istemcisi; changes/apply ve SyncInbox idempotency ile senkron.
- **Backend:** Veritabanı (global query filter ile kiracı kapsamında); yüklemeler için blob/depolama; bildirimler için NotificationOutbox + worker.

---

## Production seviyesini sağlayan unsurlar (8–12 madde)

- Katı kiracı izolasyonu (EF Core global filter’lar, kiracı kapsamında SyncInbox/NotificationOutbox).
- JWT tenant bağlama (401/403 semantiği; token başka kiracı bağlamında kullanılamaz).
- Rate limiting (auth IP bazlı; yazma kullanıcı veya tenant+IP bazlı).
- Tüm mutasyonlar için audit log + her istekte CorrelationId.
- Güvenli yüklemeler: tür whitelist, isteğe bağlı magic number kontrolü, kiracı depolama kotası.
- Çevrimdışı senkron idempotency (SyncInbox); yeniden denemeler çift uygulama yapmaz.
- Bildirimler için Outbox paterni (en az bir kez, worker, retry/backoff).
- İmzalı CDN URL’leri (süre sınırlı, HMAC); API dosya stream’i yok.
- Secret’lar ortam değişkeni / secret store ile; kod veya logda secret yok.
- Redaction’lı yapılandırılmış loglama; metrikler (/metrics korumalı); isteğe bağlı tracing.
- Reverse proxy + TLS; CORS ve politika tutarlılığı için tek API host.
- KVKK farkında tasarım: kiracı veri ayrımı, audit, saklama süresi için hazır, DSR destekli veri eşlemesi.

---

## Öne çıkan mühendislik trade-off’ları

- **Tek API host vs kiracı başına host:** Tek host CORS, sertifika ve rate limiting’i basitleştirir; karşılığında tenant UI’lardan cross-origin istekler ve her çağrıda zorunlu X-Tenant-Key.
- **Mantıksal kiracı izolasyonu vs kiracı başına DB:** Tek DB ve global filter’lar operasyon ve yedekleme karmaşıklığını azaltır; izolasyon mantıksal olup doğru kiracı bağlamı ve filter’lara bağlıdır.
- **İmzalı URL vs CDN’e istek başına auth:** İmzalı URL’ler durum bilgisiz CDN doğrulaması ve ölçek sağlar; karşılığı süre dolana kadar link paylaşımı ve imza secret’ının güvenli tutulması.
- **Outbox vs doğrudan FCM:** Outbox kalıcılık sağlar ve istek yolunu FCM’den ayırır; karşılığı eventual consistency ve worker’ın çalıştırılıp izlenmesi.

---

## Sonra ne iyileştirilir

- **Resmi DSR runbook’ları:** Export/silme süreçlerini hukuki incelemeyle dokümante etmek; saklama takvimi ve otomasyonu netleştirmek.
- **Kiracı seviyesi metrikler (sınırlı):** Kardinalitenin kabul edilebilir olduğu yerlerde destek ve kapasite için sınırlı kiracı metrikleri.
- **Senkron için çakışma çözümü:** Çevrimdışı istemcilerden aynı varlık üzerinde eşzamanlı düzenlemeler için açık kurallar veya CRDT’ler.
- **Secret rotasyonu:** JWT ve CDN imza anahtarları için kesinti olmadan rotasyon dokümantasyonu ve otomasyonu.
- **WAF / bot koruması:** Rate limiting ötesinde yaygın saldırı kalıpları için kenar kuralları veya WAF.

*Yukarıdakilerin her biri KVKK veya güvenlik etkisi yaratabilir; buna göre gözden geçirilmelidir.*
