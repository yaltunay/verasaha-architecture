# Tehdit Modeli (Türkçe)

Bu doküman yalnızca mimari referans amacıyla hazırlanmıştır ve hukuki danışmanlık veya resmi bir güvenlik sertifikası niteliği taşımaz.

---

## Tehdit Tablosu

| Tehdit | Saldırı Senaryosu | Azaltma (VeraSaha Mimarisi) |
|--------|-------------------|-----------------------------|
| **Çapraz kiracı veri erişimi** | Bir kiracı başka kiracının verisine API veya veritabanı üzerinden erişmeye çalışır. | X-Tenant-Key zorunluluğu; JWT tenantId claim ile eşleşme (eşleşmezse 403); EF Core global query filter ile tüm sorgular TenantId ile filtrelenir; Sync/Outbox tenant-scoped. |
| **Token hırsızlığı** | JWT çalınıp başka bir kiracı veya kullanıcı adına kullanılır. | JWT içindeki tenantId ile X-Tenant-Key zorunlu eşleşmesi; token başka tenant bağlamında kabul edilmez. Kısa ömürlü token ve HTTPS ile risk azaltılır. |
| **Replay saldırıları** | Aynı istek tekrar gönderilerek işlem tekrarlanır. | Sync tarafında (TenantId, UserId, OpId) idempotency; aynı OpId ile tekrar istek önceki sonucu döner. Mutasyonlar çift uygulanmaz. |
| **Doğrudan dosya erişimi** | CDN veya depolama URL’i tahmin edilerek kimlik doğrulamasız dosya indirilir. | Dosya erişimi yalnızca imzalı URL ile; tenantId + expiry + HMAC doğrulanır. Liste veya dizin erişimi yok; geçersiz/süresi dolmuş URL 403. |
| **Loglarda hassas veri açığa çıkması** | Parola, token veya PII loglara yazılır ve yetkisiz erişime açılır. | Redaction/maskeleme politikası; AuditLog’da hassas alan değeri saklanmaz; yapılandırılmış logda secret yazılmaz. |
| **API kötüye kullanımı / brute force** | Login veya yazma uç noktalarına yoğun istek atılır. | Auth (discover/login) IP bazlı rate limiting; yazma için kimliği doğrulanmış kullanıcı veya tenant+IP bazlı limit; DDoS ve brute force etkisi azaltılır. |
| **Yetkisiz yüklemeler** | Kötü niyetli veya aşırı dosya yükleme; yanlış tür veya sahte uzantı. | Korunan uç noktada JWT + X-Tenant-Key zorunlu; kiracı depolama kotası (aşımda 409); dosya türü whitelist ve magic number sniffing. |
| **CDN link paylaşımı** | İmzalı URL paylaşılarak dosyaya süre dolana kadar erişim sağlanır. | URL’de expiry (Unix timestamp) zorunlu; süre dolunca 403. Paylaşım tasarım gereği sınırlı süreli; kalıcı public link yok. |
| **Ayrıcalık yükseltme** | Kullanıcı başka rol veya tenant yetkisi elde etmeye çalışır. | JWT’de tenantId ve rol bilgisi; sunucu tarafında her istekte tenant ve yetki kontrolü; EF filter ile veri katmanında tenant izolasyonu. |
| **Gizlilik (secret) ifşası** | JWT secret, DB şifresi veya CDN imza anahtarı kaynak kodda veya logda görünür. | Secret’lar .env/ortam değişkenleri ile; kaynak kodda sabit yok; loglara yazılmaz; production’da güvenli secret store/CI ile enjekte edilir. |

---

## Özet

VeraSaha mimarisi; çok kiracılı izolasyon (X-Tenant-Key + JWT tenant binding + global query filter), imzalı CDN URL’leri, rate limiting, idempotent sync, log redaction ve secret’ların ortam dışında tutulması ile yukarıdaki tehditlere karşı savunma katmanları sunar. Bu belge mimari kararları özetler; resmi güvenlik sertifikası veya hukuki tavsiye yerine geçmez.
