# Veri Koruma Modeli (Türkçe)

Bu doküman yalnızca mimari referans amacıyla hazırlanmıştır ve hukuki danışmanlık veya resmi bir güvenlik sertifikası niteliği taşımaz.

---

## Mimari Akış Özeti

Kullanıcı verisinin korunması aşağıdaki katmanlar boyunca sağlanır:

```
İstemci (Client)
       ↓
Ters Proxy (TLS)
       ↓
API (kiracı zorunluluğu)
       ↓
Veritabanı (tenant sorgu filtreleri)
       ↓
Depolama (imzalı URL erişimi)
```

---

## İstemci (Client)

- İstekler **HTTPS** üzerinden gönderilir; TLS ile şifreleme ters proxy’de sonlandırılır.
- Kimliği doğrulanmış isteklerde **JWT** Bearer token ve **X-Tenant-Key** başlığı taşınır; istemci yanlış tenant başlığı ile istek atarsa sunucu 403 döner. Token ve tenant bilgisi istemci tarafında güvenli saklama (örn. güvenli depolama / session) ile korunmalıdır.

---

## Ters Proxy (TLS)

- **Nginx + Let’s Encrypt** ile TLS sonlandırma yapılır; api.verasaha.com, cdn.verasaha.com, app.verasaha.com HTTPS ile sunulur.
- Trafik şifreli kanal üzerinden API’ye iletilir; **X-Forwarded-For**, **X-Forwarded-Proto** gibi başlıklar güvenilir proxy yapılandırması ile API’ye aktarılır. Bu katmanda aktarım güvenliği ve doğru istemci bilgisi sağlanır.

---

## API (Kiracı Zorunluluğu)

- Korunan uç noktalarda **X-Tenant-Key** zorunludur; rezerve veya geçersiz değer 400/404 döner.
- **JWT** doğrulanır; **tenantId** claim’i ile X-Tenant-Key’den çözülen tenant karşılaştırılır; eşleşmezse 403. Böylece kullanıcı verisi yalnızca doğru kiracı bağlamında işlenir.
- Rate limiting, CORS ve input doğrulama ile API kötüye kullanımı ve yetkisiz erişim azaltılır. Kullanıcı verisi API katmanında tenant ve yetki kontrolleri ile korunur.

---

## Veritabanı (Tenant Sorgu Filtreleri)

- Tüm tenant-scoped sorgular **EF Core global query filter** ile **TenantId** filtresi uygulanarak çalışır; uygulama kodu açıkça filtre eklemeden bile cross-tenant veri dönmez.
- **AuditLog** ile değişiklikler kayıt altına alınır; hassas alan değeri saklanmaz. Veritabanı katmanında veri yalnızca ilgili kiracıya ait satırlarla sınırlıdır; veri bütünlüğü ve izolasyon bu katmanda garanti edilir.

---

## Depolama (İmzalı URL Erişimi)

- Dosya içeriği **API üzerinden stream edilmez**; erişim yalnızca **imzalı URL** ile sağlanır.
- İmzalı URL’de **tenantId**, **expiry** ve **HMAC** bulunur; CDN her istekte doğrulama yapar. Geçersiz veya süresi dolmuş URL 403 döner; doğrudan depolama URL’i veya kimlik doğrulamasız liste yoktur. Kullanıcı dosyaları depolama katmanında tenant ve süre sınırlı erişim ile korunur.
