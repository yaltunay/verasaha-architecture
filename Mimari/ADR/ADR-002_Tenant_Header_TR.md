# ADR-002 Başlık Tabanlı Kiracı Çözümlemesi

> Bu doküman VeraSaha platformu için alınan mimari kararları kayıt altına almak amacıyla hazırlanmıştır.  
> Yalnızca teknik referans niteliğindedir ve hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Status

Kabul edildi (Accepted)

## Context

API tek bir host’tan (api.verasaha.com) sunulmaktadır. Tüm kiracılar aynı API host’unu kullandığı için kiracı, istek host’undan türetilemez. Her istek için kiracıyı güvenilir ve açık şekilde belirleyecek ve kimliği doğrulanmış çağrıların yalnızca doğru kiracı bağlamında çalışmasını sağlayacak bir yöntem gereklidir. Çapraz kiracı erişimin (örn. A kiracısı için verilen token’ın B kiracısı bağlamında kullanılması) önlenmesi gerekir.

## Decision Drivers

1. **Tek API host:** Tek host (api.verasaha.com) CORS, sertifika ve rate limiting'i basitleştirir; kiracı host'tan çıkarılamadığı için açık, istek seviyesinde mekanizma gerekir.
2. **Güvenlik ve denetim:** Başlık görünür ve denetlenebilir; JWT bağlama token'ın farklı kiracı bağlamında kullanılmasını engeller (403).
3. **Operasyonel basitlik:** Kiracı başına API dağıtımı veya host tabanlı kiracı çözümlemesi yok; tek politika yüzeyi.

## Decision

**Başlık tabanlı kiracı çözümlemesi** aşağıdaki tasarımla kullanılmaktadır:

- **X-Tenant-Key başlığı:** Korunan her API isteğinde `X-Tenant-Key` başlığı zorunludur. Değer kiracı anahtarıdır (örn. firma.verasaha.com için subdomain `firma`). İstemci bunu UI host’undan (örn. subdomain) alır ve her istekte gönderir. Rezerve değerler (www, api, admin, app, debug, test) reddedilir; bilinmeyen veya pasif kiracı 400/404 döner.

- **API host modeli:** Tüm API api.verasaha.com’dan sunulur. İstek host’undan kiracı çıkarılmaz. CORS ve güvenlik politikası tek noktada yönetilir; sertifika ve yönlendirme karmaşıklığı düşük kalır.

- **JWT tenant bağlama:** Kimliği doğrulanmış isteklerde API kiracıyı `X-Tenant-Key`’den çözümler ve JWT’deki `tenantId` claim’i ile karşılaştırır. Claim yoksa istek 401 ile reddedilir. Claim çözümlenen kiracı ile eşleşmezse istek 403 ile reddedilir. Bir kiracı için verilen token başka kiracı bağlamında kullanılamaz.

- **Çapraz kiracı erişiminin önlenmesi:** Zorunlu X-Tenant-Key, rezerve/bilinmeyen kiracılara karşı doğrulama ve JWT tenant bağlama birlikte (1) her korunan isteğin geçerli bir kiracı bağlamına sahip olmasını ve (2) token’daki kimliğin aynı kiracıya bağlı olmasını sağlar. Çapraz kiracı veri erişimi veya yetki kullanımı API katmanında engellenir.

## Alternatives Considered

- **Host’tan kiracı (örn. firma.verasaha.com/api):** Her kiracı subdomain’inde API sunmayı gerektirir; sertifika, yönlendirme ve rate limiting karmaşıklığı artar. Tek API host tercih edildi.

- **Yalnızca JWT’de kiracı:** İstemcinin token’da doğru kiracıyı göndermesi gerekirdi; sunucu yine de isteğin o kiracı için olduğunu doğrulayacak bir yola ihtiyaç duyardı. Başlık açık, denetlenebilir istek seviyesi kiracı sağlar. Reddedildi.

- **Query parametresi veya body:** Denetim için daha az görünür ve unutulması kolay; metadata için başlıklar standarttır. Reddedildi.

## Consequences

### Pozitif

- İstek başına net ve açık kiracı bağlamı sağlar; tek API host kullanımı operasyonu ve güvenlik politikalarının yönetimini basitleştirir.
- JWT tenant bağlama, token’ın yanlış kiracı bağlamında kullanılmasını engeller ve 401/403 ayrımı tanılama süreçlerini destekler.

### Negatif

- Tüm istemcilerin korunan isteklerde X-Tenant-Key göndermesi zorunludur; tenant UI’lardan API’ye erişim için CORS yapılandırmasının doğru yapılması gerekir.
- Geliştirme ortamında kullanılan opsiyonel DevTenant varsayılanı ile production davranışı arasında fark oluşabilir; bu farkın yanlışlıkla canlı ortama taşınmaması için dikkat gerektirir.

## When to Revisit

Yeniden değerlendir: düzenleyici veya müşteri gereksinimleri kiracının host'tan türetilmesini talep ederse; veya kiracı seviyesinde TLS veya WAF kuralları kiracı başına kenar yönlendirmeyi gerekli kılarsa.
