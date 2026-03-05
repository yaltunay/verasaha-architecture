# Gizlilik Odaklı Tasarım (Türkçe)

Bu doküman yalnızca mimari referans amacıyla hazırlanmıştır ve hukuki danışmanlık veya resmi bir güvenlik sertifikası niteliği taşımaz.

---

## Gizlilik Öncelikli Mimari

VeraSaha, gizliliği tasarım aşamasından itibaren dikkate alır: veri minimizasyonu, erişim kontrolü, şeffaflık ve denetim izi mimari kararlarla desteklenir. Kişisel veri yalnızca gerekli bağlamda (kiracı ve kullanıcı kapsamında) işlenir; cross-tenant veya yetkisiz erişim yapısal olarak engellenir.

---

## Kiracı İzolasyonu

- **X-Tenant-Key** ve **JWT tenantId** eşleşmesi ile her istek tek bir kiracı bağlamında işlenir.
- **EF Core global query filter** ile veritabanı sorguları otomatik olarak TenantId ile filtrelenir; uygulama kodu yanlışlıkla başka kiracı verisi döndüremez.
- Sync ve bildirim kuyrukları tenant ve kullanıcı kapsamında; bir kiracının verisi diğerine sızmaz.

---

## Güvenli Yüklemeler

- Dosya yükleme uç noktaları **JWT ve X-Tenant-Key** ile korunur; yalnızca kimliği doğrulanmış ve doğru kiracı bağlamındaki kullanıcılar yükleyebilir.
- **Kiracı depolama kotası** ile aşırı veya kötüye kullanım sınırlanır; **dosya türü whitelist** ve (yapılandırıldığında) **magic number** kontrolü ile risk azaltılır.
- Yüklenen dosyalar kiracı ve kullanıcı bağlamında depolanır; erişim yalnızca imzalı URL ile sağlanır.

---

## İmzalı URL’ler

- Dosya indirme **imzalı URL** ile yapılır; URL’de tenantId, süre (expiry) ve HMAC imzası bulunur.
- CDN her istekte süreyi ve imzayı doğrular; geçersiz veya süresi dolmuş URL 403 döner. Doğrudan liste veya kimlik doğrulamasız erişim yoktur; gizlilik ve veri minimizasyonu korunur.

---

## Denetim Loglama

- Tüm varlık değişiklikleri **AuditLog** tablosuna yazılır; kim, ne zaman, hangi varlık üzerinde işlem yaptığı kayıt altına alınır.
- Hassas alan değerleri (parola, token, tam PII) audit kaydına eklenmez; denetim ve uyumluluk için gerekli minimum bilgi tutulur.

---

## Log Maskeleme (Redaction)

- Uygulama loglarında **hassas veri** (parola, token, PII) yazılmaz; redaction/maskeleme politikası uygulanır.
- Yapılandırılmış logda CorrelationId, TenantId, UserId bağlam bilgisi kullanılır; teşhis ve denetim mümkün iken gizlilik korunur.
