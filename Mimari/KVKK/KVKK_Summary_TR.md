# KVKK Özeti (Türkçe)

Bu doküman yalnızca mimari referans amacıyla hazırlanmıştır ve hukuki danışmanlık veya resmi bir güvenlik sertifikası niteliği taşımaz.

---

## Genel Bakış

VeraSaha mimarisi, kişisel verilerin korunması ve KVKK ile uyumluluk hedeflerini destekleyecek şekilde tasarlanmıştır. Çok kiracılı veri ayrımı, denetim loglama, güvenli dosya erişimi, gizlilik izolasyonu, saklama süresi farkındalığı ve veri sahibi taleplerine (DSR) hazırlık bu özetin kapsamındadır.

---

## Çok Kiracılı Veri Ayrımı

- Tüm kiracıya ait veriler **TenantId** ile ayrılır; EF Core global query filter ile sorgular otomatik filtrelenir.
- API isteklerinde **X-Tenant-Key** zorunludur; JWT içindeki **tenantId** claim’i ile eşleşmezse 403 döner. Cross-tenant veri erişimi uygulama ve veri katmanında engellenir.
- Sync ve bildirim (SyncInbox, NotificationOutbox) tenant ve kullanıcı kapsamında işlenir; kiracılar arası veri karışmaz.

---

## Denetim Loglama (Audit Logging)

- Tüm varlık değişiklikleri **AuditLog** tablosuna yazılır: TenantId, UserId, Action, EntityName, EntityId, CorrelationId, TimestampUtc vb.
- Loglara **hassas veri değeri** yazılmaz; yalnızca kim, ne zaman, hangi varlık üzerinde işlem yaptığı kaydedilir. Denetim izi ve olay incelemesi için kullanılır; KVKK açısından işlem kayıtları desteklenir.

---

## Güvenli Dosya Erişimi

- Dosya erişimi **imzalı URL** ile sağlanır; tenantId, expiry ve HMAC imzası CDN tarafından doğrulanır.
- Kimlik doğrulamasız liste veya dizin erişimi yok; sadece API tarafından üretilen, süre sınırlı imzalı URL’ler geçerlidir. Veri minimizasyonu ve erişim kontrolü mimari seviyede uygulanır.

---

## Gizlilik İzolasyonu (Secrets)

- JWT secret, veritabanı bağlantı bilgisi, CDN imza anahtarı vb. **kaynak kodda veya logda** tutulmaz; .env veya ortam değişkenleri ile yönetilir.
- Production’da secret’lar güvenli secret store veya CI/CD ile enjekte edilir. Kişisel veri işleme süreçlerinde kullanılan sistem erişim bilgileri bu sayede korunur.

---

## Saklama Süresi Farkındalığı (Retention)

- Mimari, **audit log** ve **işlem kayıtları** için saklama süresi politikasının uygulanabileceği yapıyı sunar (örn. zaman damgalı kayıtlar, TenantId ile filtreleme).
- Veri silme veya anonimleştirme işlemleri iş kuralları ve operasyonel süreçlerle tanımlanır; teknik altyapı zaman bazlı temizlik veya arşivleme için uygundur.

---

## DSR Hazırlığı (Veri Sahibi Talepleri)

- **Erişim / düzeltme / silme / taşınabilirlik** talepleri için: TenantId ve UserId ile kapsamlı veri eşlemesi mümkündür; AuditLog ve tenant-scoped sorgular veri konumunu ve işlem geçmişini takip etmeyi destekler.
- CorrelationId ve yapılandırılmış log ile belirli bir işlem veya kullanıcı oturumunun izlenmesi kolaylaştırılır; DSR yanıt süreçleri bu verilerle desteklenebilir.

---

## Operasyonel Hazırlık (Hukuki Danışmanlık Değildir)

Bu bölüm uygulama durumunu ve operasyonel süreçler için placeholder’ları açıklar. Hukuki danışmanlık niteliği taşımaz; saklama süresi ve DSR süreçleri hukuki ve uyumluluk incelemesi ile tanımlanmalıdır.

### Uygulanan vs planlanan

| Alan | Uygulandı | Planlanan / placeholder |
|------|-----------|---------------------------|
| Kiracı veri ayrımı | Evet (TenantId, global filter, X-Tenant-Key + JWT) | — |
| Denetim loglama | Evet (AuditLog tablosu, hassas değer yok) | Saklama süresi politika ile belirlenecek |
| Güvenli dosya erişimi | Evet (imzalı URL, liste yok) | — |
| Secret izolasyonu | Evet (.env, kod/logda secret yok) | Rotasyon prosedürü dokümante edilecek |
| DSR veri eşlemesi | Teknik yetenek (TenantId, UserId, AuditLog) | Export/silme runbook’ları hukuki inceleme gerektirir |

### Saklama takvimi (placeholder)

| Veri kategorisi | Saklama (örnek) | Sahip / inceleme |
|-----------------|------------------|-------------------|
| Audit log | Politika ile tanımlanacak | Hukuk / KVK |
| Kullanıcı/kiracı verisi | Politika ile tanımlanacak | Hukuk / KVK |
| SyncInbox / Outbox | Operasyonel; temizleme politikası TBD | Operasyon |

*Gerçek saklama süresi veri sorumlusu tarafından hukuki ve KVKK uyumluluk incelemesi ile belirlenmelidir.*

### DSR süreci (placeholder)

- **Export (erişim / taşınabilirlik):** TenantId + UserId sorguları ve AuditLog ile teknik olarak desteklenir. Kesin format, kapsam ve teslim süreci taahhüt öncesi **hukuki inceleme gerektirir**.
- **Silme (erasure):** Kiracı/kullanıcı bazında teknik silme veya anonimleştirme uygulanabilir; kademeli silme ve saklama istisnaları (örn. yasal saklama) **hukuki inceleme gerektirir**.
- **Düzeltme:** Normal güncelleme akışları ile desteklenir; denetim izi mevcuttur. Süreç ve süreler **hukuki inceleme gerektirir**.

*Bu metni uyumluluk kontrol listesi olarak kullanmayın; DSR prosedürleri ve saklama için hukuk ve KVK ile çalışın.*
