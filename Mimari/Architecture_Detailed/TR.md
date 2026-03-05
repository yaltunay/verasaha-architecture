# Mimari Detay (Türkçe)

VeraSaha SaaS platformunun detaylı mimari dokümanı. Neden bu tasarım, trade-off'lar ve reddedilen alternatifler bu belgede özetlenir.

---

## 1. Alan ve Host Topolojisi (Detay)

**Neden tek API host?**  
Tüm API'nin api.verasaha.com'da toplanması, CORS ve güvenlik politikasını tek noktada yönetmeyi, sertifika yönetimini basitleştirmeyi ve istemcinin her zaman aynı origin'e istek atmasını sağlar. Diğer host'larda (örn. {tenant}.verasaha.com veya verasaha.com) /api yoluna yapılan istekler 404 ile reddedilir; mesaj: "API is served from api.verasaha.com." Bu sayede yanlış yapılandırılmış istemciler veya proxy'ler hemen fark edilir.

**Trade-off:** Tenant UI'ın kendi subdomain'inde (örn. firma.verasaha.com) çalışması, ancak API'nin farklı bir host'ta (api.verasaha.com) olması cross-origin isteklere yol açar. CORS doğru yapılandırıldığı sürece bu kabul edilir; güvenlik ve operasyon basitliği tercih edilir.

**Reddedilen alternatif:** Her tenant subdomain'inde API'yi de sunmak (örn. firma.verasaha.com/api). Bu, sertifika, routing ve rate limiting karmaşıklığını artırır; tek API host ile tek noktada yönetim tercih edilmiştir.

**Rezerve subdomain'ler** (www, api, admin, app, debug, test) tek bir kaynakta (ReservedSubdomains) tanımlıdır; hem X-Tenant-Key reddi hem de CORS origin kontrollerinde kullanılır. Bu isimler asla tenant anahtarı olarak kabul edilmez.

**Reverse proxy (Nginx + Let's Encrypt):** TLS sonlandırma ve api/cdn/app host'larına yönlendirme kavramsal olarak Nginx ile yapılır; forwarded headers (X-Forwarded-*) API'ye iletilir ve güvenilir proxy yapılandırması ile kullanılır.

---

## 2. Kiracı Modeli – API-Only Host ve JWT Bağlama (Detay)

**Neden X-Tenant-Key?**  
API tek host'ta (api.verasaha.com) sunulduğu için istek host'undan tenant çıkarılamaz. Tenant bilgisi zorunlu olarak başlıkla (X-Tenant-Key) taşınır. İstemci, UI host'undan (örn. firma.verasaha.com) subdomain'i (firma) alıp aynı değeri X-Tenant-Key olarak gönderir.

**Korunan uç noktalar:** [AllowAnonymous] olmayan tüm uç noktalar korumalıdır; X-Tenant-Key zorunludur. AllowAnonymous olanlar (örn. POST /api/auth/discover, POST /api/auth/login) başlık gerektirmez.

**Rezerve host kuralları:** X-Tenant-Key değeri rezerve listeyle karşılaştırılır; geçersiz veya bilinmeyen kiracı 400/404 ile reddedilir. Aktif olmayan kiracı da 404 döner.

**JWT tenant binding:** Kimliği doğrulanmış isteklerde JWT'deki tenantId claim'i ile X-Tenant-Key'den çözülen TenantId karşılaştırılır. Claim yoksa 401 (yetkisiz); eşleşmezse 403 (yasak). Böylece token'ın başka bir tenant'ta kullanılması engellenir; çok kiracılı izolasyon garanti altına alınır.

**401 vs 403:** Kimlik yok veya tenant claim eksik → 401. Kimlik var ancak token'ın tenant'ı ile istek tenant'ı uyuşmuyor → 403.

**Development:** Opsiyonel DevTenant yapılandırması ile X-Tenant-Key gönderilmeden varsayılan bir tenant atanabilir (yalnızca Development). Production'da bu yoktur.

---

## 3. Çekirdek Mimari Kalıplar (Detay)

**Onion Architecture:** Domain katmanı dış bağımlılık içermez; iş kuralları ve aggregate'ler burada. Application katmanı CQRS/MediatR ve arayüzler; veritabanı referansı yok. Infrastructure (dış servisler) ve Persistence (EF Core, DbContext) uygulama detaylarını içerir. Presentation (Web API) yalnızca HTTP uç noktalarıdır. Bu sayede test edilebilirlik ve değişim izolasyonu sağlanır.

**Meeting aggregate ve immutability:** Toplantı tamamlandıktan sonra belirli alanlar değiştirilemez; domain kuralları ile zorunlu değişiklik nedeni (reason) istenir. Bu, denetim izi ve veri bütünlüğü için tercih edilir.

**ParticipationType (Planned / Actual):** Katılımcının önceden planlanmış mı yoksa fiilen (yerinde) eklenmiş mi olduğunu ayırt eder; raporlama ve bildirim kuralları buna göre uygulanır.

**AbsenceReason:** Devamsızlık durumunda neden alanı zorunludur; domain doğrulaması ile uygulanır.

**AuditLog:** SaveChanges override veya interceptor ile tüm entity değişiklikleri merkezi AuditLog tablosuna yazılır (TenantId, UserId, Action, EntityName, EntityId, CorrelationId, TimestampUtc, vb.). Loglara hassas veri yazılmaz; denetim ve olay incelemesi için kullanılır.

**CorrelationId middleware:** Her isteğe benzersiz bir CorrelationId atanır (başlıktan veya TraceIdentifier). Log ve trace'lerde bu ID ile istek takibi yapılır; olay teşhisinde kritiktir.

**Rate limiting:** Auth (discover/login) IP bazlı sıkı limit; yazma (mutasyon) için kimliği doğrulanmış kullanıcı veya tenant+IP bazlı ayrı politika. Kötüye kullanım ve DDoS azaltılır; auth ve write trafiği ayrı partition'larda değerlendirilir.

---

## 4. Çevrimdışı Öncelikli Senkronizasyon (Detay)

**Neden SyncInbox (TenantId, UserId, OpId)?**  
İstemci aynı OpId ile tekrar istek gönderdiğinde sunucu yeniden mutasyon uygulamaz; önceden kaydedilmiş sonucu döner. Ağ kesintisi veya yeniden denemelerde çift uygulama ve veri bozulması önlenir. Anahtar (TenantId, UserId, OpId) kiracı ve kullanıcı izolasyonunu da sağlar.

**UTC disiplini:** Tüm zaman damgaları UTC; ISO 8601 (Z) ile gönderilir ve alınır. since, serverTimeUtc, occurredAtUtc hep UTC'dir. Saat dilimi karışıklığı ve senkronizasyon hataları önlenir.

**Tombstone modeli:** Silinen kayıtlar için "silindi" bilgisi (tombstone) changes yanıtında döner; istemci yerel silme ile senkron kalır.

**Alternatif reddedilen:** Sunucunun dosya stream'i veya doğrudan dosya sunması senkron contract'ta yok; dosya CDN ve imzalı URL ile ayrı akıştadır.

---

## 5. Dosya ve CDN (Detay)

**Neden API dosya stream etmez?**  
Dosya içeriğinin API üzerinden akması bellek ve CPU yükü oluşturur; özellikle kısıtlı RAM ortamında istenmez. CDN imzalı URL ile dosyayı doğrudan sunar; API sadece yükleme, meta, silme ve imzalı URL üretimi ile sınırlı kalır. Ayrıca CDN cache ve ölçekleme avantajı kullanılır.

**İmzalı URL:** tenantId, expiry (Unix timestamp) ve HMAC imzası içerir; CDN her istekte süre ve imzayı doğrular. Geçersiz veya süresi dolmuş URL 403 döner. Doğrudan liste veya kimlik doğrulamasız erişim yoktur.

**Kota ve magic number:** Kiracı depolama kotası yükleme öncesi kontrol edilir; aşımda 409. Dosya türü whitelist ve (yapılandırıldığında) magic number sniffing ile sahte uzantı riski azaltılır; tenant depolama kullanımı takip edilir.

---

## 6. Bildirimler (Detay)

**Outbox pattern:** Bildirimler NotificationOutbox tablosuna yazılır; aynı transaction içinde domain değişikliği ile birlikte commit edilir. Arka planda worker satırları alır, FCM'e gönderir. Handlers/controller'lar FCM'i doğrudan çağırmaz; en az bir kez kuyruğa alınma ve worker tarafında retry/backoff/dead-letter yönetimi sağlanır.

**IdempotencyKey:** Aynı mantıksal olayın (örn. aynı toplantı hatırlatıcısı) tekrar kuyruğa alınması (TenantId, IdempotencyKey) unique kısıtı ile engellenir. Format dokümante edilir (örn. meeting-reminder:…).

**DEV uyumluluğu:** FCM kapalıyken (Development'ta varsayılan) worker NoopNotificationSender kullanır; Production'da FCM kapalıysa worker başlatılmaz (yapılandırma ile kontrol).

---

## 7. Gözlemlenebilirlik (Detay)

**Yapılandırılmış loglama:** Serilog ile yapılandırılmış log; CorrelationId, TenantId, UserId (varsa) log bağlamına eklenir. Hassas alanlar maskeleyici (redaction) ile log'a yazılmaz.

**Metrics:** /metrics uç noktası Prometheus formatında; erişim Admin JWT veya dahili ağ ile kısıtlıdır. notifications.outbox.pending gauge; tenant etiketi yok (kardinalite kuralı).

**Tracing:** OpenTelemetry; Development'ta Console exporter; Production'da OTLP endpoint yapılandırıldığında exporter eklenir. Span'lara correlation_id, user_id, tenant_id (isteğe bağlı) eklenebilir; tenant_id kardinalite endişesi ile devre dışı bırakılabilir.

**Neden log'da secret yok?** Hassas bilgilerin (parola, token, PII) log'a düşmesi güvenlik ve uyumluluk riskidir; redaction ve log politikası ile önlenir.

---

## 8. Çok Kiracılı İzolasyon Garantileri

- Tüm tenant-scoped veri erişimi **TenantId** ile filtrelenir; EF Core global query filter ve repository katmanı buna uyar.
- Sync changes/apply yalnızca ilgili tenant'ın verisini döner veya uygular; JWT tenantId claim'i ile X-Tenant-Key eşleşmesi zorunludur.
- Notification worker tenant bazlı (veya tenant-scoped batch) işler; cross-tenant veri sızıntısı olmaz.
- CDN imzası tenantId içerir; bir tenant başka tenant'ın dosya URL'ini geçerli imza olmadan kullanamaz.
