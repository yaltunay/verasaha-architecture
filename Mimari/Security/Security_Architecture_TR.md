# Güvenlik Mimarisi (Türkçe)

Bu doküman yalnızca mimari referans amacıyla hazırlanmıştır ve hukuki danışmanlık veya resmi bir güvenlik sertifikası niteliği taşımaz.

---

## Güvenlik Mimarisi Genel Bakış

VeraSaha, çok kiracılı SaaS platformunda güvenlik; kimlik doğrulama, kiracı izolasyonu, API kontrolleri, dosya güvenliği, gizlilik yönetimi ve altyapı güvenliği ile katmanlı olarak ele alınır. Tüm korumalı uç noktalar JWT ve X-Tenant-Key ile zorunlu tenant bağlamında çalışır; cross-tenant veri erişimi yapısal olarak engellenir.

---

## Güvenlik Değişmezleri (Asla Bozulmamalı)

- **Çapraz kiracı veri erişimi yok:** A kiracısı bağlamındaki bir istek B kiracısının verisini asla okuyamaz veya yazamaz. Global query filter, JWT tenant bağlama ve kiracı kapsamında SyncInbox/NotificationOutbox ile zorunlu kılınır.
- **Kiracı verisine kimlik doğrulamasız erişim yok:** Korumalı uç noktalar geçerli JWT gerektirir; anonim erişim yalnızca discover/login. Dosya erişimi yalnızca imzalı URL ile (genel liste yok).
- **Kod veya logda secret yok:** Parolalar, token’lar, bağlantı dizeleri ve imza anahtarları asla commit veya log’a yazılmaz; redaction uygulanır.
- **Tenant bağlama katıdır:** JWT `tenantId` X-Tenant-Key ile eşleşmek zorunda; eşleşmezse → 403. Rezerve başlık değerleri reddedilir.
- **Mutasyonlar için denetim izi:** Her varlık değişikliği kaydedilir (kim, ne zaman, hangi varlık); audit’te hassas alan değeri yok.

Bu değişmezlerin ihlali hem güvenliği hem KVKK uyumlu veri işlemeyi zedeler; mimari değişiklik bunları korumalıdır.

---

## AuthN / AuthZ Modeli

- **Kimlik doğrulama (AuthN):** JWT Bearer token; login sonrası verilir (discover ve login uç noktaları AllowAnonymous). Claim’ler: `sub` (userId), `tenantId`, `exp`, `iss`. Başlangıçta geçersiz JWT yapılandırmasında fail-fast.
- **Yetkilendirme (AuthZ):** Kiracı kapsamında: istek, X-Tenant-Key’den çözülen kiracı için yalnızca JWT `tenantId` eşleşiyorsa yetkilidir. Rol/izin kontrolleri (varsa) tenant bağlamadan sonra uygulanır. 401 = kimlik yok veya geçersiz; 403 = kimlik geçerli ama izin yok (örn. yanlış kiracı).
- **Token’ın çapraz kiracı kullanımı yok:** A kiracısı için verilen token, B kiracısı için X-Tenant-Key ile kullanılamaz; 403 döner.

---

## Kiracı İzolasyon Modeli

- **İstek başına açık kiracı:** X-Tenant-Key başlığı kiracı anahtarını taşır (örn. subdomain); TenantId’ye çözülür; doğrulanır (rezerve/bilinmeyen/pasif → 400/404).
- **Veri katmanı:** EF Core global query filter tüm kiracı kapsamında sorgulara TenantId ekler; uygulama kodunda opt-out yok.
- **Senkron ve bildirimler:** SyncInbox anahtarı (TenantId, UserId, OpId); NotificationOutbox kiracı kapsamında; worker’lar yalnızca kiracı bağlamında işler. Çapraz kiracı okuma veya gönderim yok.
- **Dosyalar:** İmzalı URL tenantId içerir; CDN doğrular; bir kiracı diğerinin kaynağı için geçerli URL üretemez.

---

## Dosya Güvenliği Modeli

- **Yükleme:** Yalnızca API; JWT + X-Tenant-Key zorunlu. Yazmadan önce kota kontrolü; dosya türü whitelist; isteğe bağlı magic number kontrolü. Dosyalar kiracı (ve kullanıcı) yolunda saklanır; doğrudan genel URL yok.
- **İndirme:** API stream yok. API kısa ömürlü imzalı URL üretir (tenantId + expiry + HMAC). CDN her istekte doğrular; geçersiz veya süresi dolmuş → 403. Dizin listesi veya anonim erişim yok.
- **Silme:** API uç noktası; kiracı ve sahiplik kapsamında; audit log’a yazılır.

---

## Gözlemlenebilirlik Güvenlik Olarak

- **Audit:** Her mutasyon → AuditLog (TenantId, UserId, Action, EntityName, EntityId, CorrelationId, TimestampUtc). Hassas alan değeri yok. Olay incelemesi ve uyumluluk; KVKK ile ilgili.
- **Korelasyon:** Her istekte CorrelationId; log ve isteğe bağlı trace’lerde taşınır. PII loglamadan istek seviyesi inceleme sağlar.
- **Redaction:** Parolalar, token’lar ve tam PII log’a yazılmaz. Teşhis için bağlam (CorrelationId, TenantId, UserId) ile yapılandırılmış loglama; gizlilik korunur.
- **Metrik/izleme:** /metrics korumalı (Admin veya dahili); metriklerde yüksek kardinaliteli tenant yok. Span’larda correlation_id, user_id olabilir; secret yok. Gözlemlenebilirlik kötüye kullanım ve anomali tespitini destekler.

---

## Operasyonel Kontroller

- **Rate limiting:** Auth (discover/login) IP bazlı sıkı limit; yazma (mutasyon) kimliği doğrulanmış kullanıcı veya tenant+IP bazlı. Brute force ve DDoS azaltılır; auth ve yazma ayrı partition’da.
- **Secret’lar:** .env veya ortam değişkenleri; production’da secret store veya CI/CD enjeksiyonu. .env versiyon kontrolünde yok. JWT ve CDN imza anahtarı rotasyonu planlanmalı; auth ve dosya erişimini etkiler.
- **Ortam ayrımı:** Development’ta DevTenant varsayılanı ve noop bildirim gönderici kullanılabilir; production’da varsayılan kiracı yok ve FCM/worker yapılandırmaya göre. Yapılandırma odaklı; production secret’ları dev’de yok.
- **Reverse proxy:** TLS kenarda; X-Forwarded-* güvenilir (istemci IP/şema); tutarlı CORS ve politika için tek API host.

---

## Kimlik Doğrulama (JWT Modeli)

- **JWT kullanımı:** Korunan API isteklerinde Bearer token ile kimlik doğrulama. Token içinde `tenantId`, `sub` (userId), `exp`, `iss` vb. claim'ler taşınır.
- **Tenant bağlama:** JWT'deki `tenantId` claim'i ile istek başlığındaki `X-Tenant-Key`'den çözülen tenant karşılaştırılır. Eşleşmezse 403 döner; token'ın başka tenant'ta kullanılması engellenir.
- **401 / 403 ayrımı:** Kimlik yok veya tenant claim eksik → 401. Kimlik var ancak tenant uyuşmazlığı → 403.
- **Auth uç noktaları:** `POST /api/auth/discover`, `POST /api/auth/login` AllowAnonymous; diğer tüm korumalı uç noktalarda JWT zorunludur.
- **Fail-fast:** JWT yapılandırması (secret, issuer, audience) hatalıysa uygulama başlamaz; production'da yanlış konfigürasyon riski azaltılır.

---

## Çok Kiracılı İzolasyon

- **X-Tenant-Key:** Korunan her istekte zorunlu; rezerve değerler (www, api, admin, app, debug, test) reddedilir; bilinmeyen veya pasif tenant 400/404.
- **EF Core global query filter:** Tüm tenant-scoped sorgular otomatik olarak `TenantId` ile filtrelenir; uygulama kodu tek başına cross-tenant veri döndüremez.
- **Sync ve Outbox:** SyncInbox ve NotificationOutbox tabloları (TenantId, UserId, OpId) ve (TenantId, IdempotencyKey) ile kiracı ve kullanıcı izolasyonu sağlar.
- **CDN imzası:** İmzalı URL'de tenantId ve HMAC bulunur; bir tenant başka tenant'ın dosyasına geçerli imza üretemez.

---

## API Güvenlik Kontrolleri

- **CORS:** Sadece izin verilen origin'ler (tenant subdomain'leri, app.verasaha.com, vb.) kabul edilir; tek API host (api.verasaha.com) üzerinden merkezi yönetim.
- **Rate limiting:** Auth (discover/login) IP bazlı sıkı limit; yazma (mutasyon) için kimliği doğrulanmış kullanıcı veya tenant+IP bazlı ayrı politika. Brute force ve DDoS azaltılır.
- **Rezerve subdomain kuralları:** X-Tenant-Key rezerve listeyle kontrol edilir; bu isimler asla tenant olarak kabul edilmez.

---

## Dosya Depolama Güvenliği (İmzalı CDN URL’leri)

- **API dosya stream etmez:** Dosya içeriği API üzerinden akmaz; bellek ve CPU yükü ile doğrudan dosya sunma riski önlenir.
- **İmzalı URL:** tenantId, expiry (Unix timestamp) ve HMAC imzası içerir; CDN her istekte süre ve imzayı doğrular. Geçersiz veya süresi dolmuş URL 403 döner.
- **Doğrudan liste yok:** Kimlik doğrulamasız dosya listesi veya dizin erişimi yok; erişim yalnızca API tarafından üretilen imzalı URL ile sağlanır.
- **Kota ve tür kontrolü:** Kiracı depolama kotası yükleme öncesi kontrol edilir; aşımda 409. Dosya türü whitelist ve (yapılandırıldığında) magic number sniffing ile sahte uzantı riski azaltılır.

---

## Gizlilik Yönetimi (.env)

- **Ortam değişkenleri:** Hassas yapılandırma (JWT secret, DB connection string, CDN signing key, FCM vb.) .env veya ortam değişkenleri ile sağlanır; kaynak kodda sabit değer tutulmaz.
- **Üretim:** Production'da secret'lar güvenli bir secret store veya CI/CD ile enjekte edilir; .env dosyası versiyon kontrolüne alınmaz.

---

## Loglama Güvenliği (Maskeleme)

- **Hassas veri loglanmaz:** Parola, token, tam PII log'a yazılmaz; redaction/maskeleme politikası uygulanır.
- **Yapılandırılmış log:** Serilog ile CorrelationId, TenantId, UserId (varsa) bağlama eklenir; içerik denetim ve teşhis için uygun, gizlilik korunur.
- **AuditLog:** Entity değişiklikleri AuditLog tablosuna yazılır; hassas alan değerleri saklanmaz; sadece kim, ne zaman, hangi varlık üzerinde işlem yaptığı kaydedilir.

---

## Gözlemlenebilirlik Güvenliği (Metrik + İzleme)

- **Metrics:** `/metrics` Prometheus formatında; erişim Admin JWT veya dahili ağ ile kısıtlıdır; public erişim yok.
- **Tenant kardinalitesi:** Metriklerde tenant etiketi (örn. notifications.outbox.pending) kardinalite patlamasını önlemek için kullanılmaz veya sınırlı tutulur.
- **Tracing:** OpenTelemetry span'larına correlation_id, user_id eklenebilir; tenant_id isteğe bağlı veya devre dışı; hassas veri span'lara yazılmaz.

---

## Altyapı Güvenliği (Docker + Reverse Proxy + TLS)

- **Reverse proxy (Nginx + Let's Encrypt):** TLS sonlandırma reverse proxy'de yapılır; api.verasaha.com, cdn.verasaha.com, app.verasaha.com HTTPS ile sunulur.
- **Forwarded headers:** X-Forwarded-For, X-Forwarded-Proto API'ye iletilir; güvenilir proxy yapılandırması ile kullanılır; rate limiting ve loglama doğru client bilgisi ile yapılır.
- **Docker:** Uygulama konteyner içinde çalışabilir; ağ ve volume izolasyonu ile production dağıtımı desteklenir.
- **Tek API host:** Tüm API trafiği api.verasaha.com üzerinden geçer; CORS ve güvenlik politikası tek noktada yönetilir.
