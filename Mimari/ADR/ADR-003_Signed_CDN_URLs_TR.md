# ADR-003 İmzalı CDN URL’leri

> Bu doküman VeraSaha platformu için alınan mimari kararları kayıt altına almak amacıyla hazırlanmıştır.  
> Yalnızca teknik referans niteliğindedir ve hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Status

Kabul edildi (Accepted)

## Context

Platformun kullanıcı yüklemelerini (örn. toplantı ekleri) saklaması ve sunması gerekmektedir. Tüm dosya içeriğini API üzerinden stream etmek bellek ve CPU tüketir ve iyi ölçeklenmez. Dosyaların kalıcı saklandığı, verimli sunulduğu ve yalnızca doğru kiracı bağlamında yetkili kullanıcılar tarafından erişildiği bir model gereklidir. Doğrudan depolama URL’leri veya kimlik doğrulamasız liste açığa çıkmamalıdır.

## Decision Drivers

1. **Ölçek ve performans:** API dosya baytı stream etmemeli; CDN bant genişliğini ve önbelleği üstlenir; erişim kiracı ve süre kapsamında kalmalı.
2. **Güvenlik:** Anonim veya uzun ömürlü URL yok; bir kiracı diğerinin dosyasına erişemez; imza kiracı ve kaynağı bağlar.
3. **Operasyonel basitlik:** Durum bilgisiz CDN doğrulaması (HMAC + expiry); istek başına API auth geri çağrısı yok.

## Decision

Dosya sunumu için **imzalı CDN URL’leri** kullanılmaktadır:

- **Sunucuda saklanan dosya yüklemeleri:** Yüklemeler API tarafından (JWT ve X-Tenant-Key ile) kabul edilir. Dosyalar kiracı ve kullanıcı kapsamında depolamada (örn. blob storage veya dosya sistemi) saklanır. API meta veriyi kaydeder; indirmede dosya baytlarını istemcilere stream etmez. Depolama API ve CDN arkasındadır; doğrudan açığa çıkmaz.

- **CDN sunum modeli:** Ayrı bir CDN host’u (örn. cdn.verasaha.com) dosya baytlarını sunar. API dosya içeriğini stream etmez. CDN her isteği imzalı URL ile doğrular; yalnızca geçerli ve süresi dolmamış imzaya sahip istekler sunulur. Bant genişliği CDN’e aktarılır ve önbellek/ölçekleme avantajı kullanılır.

- **İmzalı URL’ler:** API belirli dosyalar için kısa ömürlü imzalı URL’ler üretir. Her URL tenantId, expiry (Unix timestamp) ve paylaşılan gizli anahtar ile hesaplanan HMAC imzası içerir. CDN (veya depolama önündeki küçük bir doğrulama katmanı) her istekte imzayı ve süreyi doğrular. Geçersiz veya süresi dolmuş URL 403 döner. Dizin listesi veya kimlik doğrulamasız erişim desteklenmez.

- **Süre ve imza doğrulaması:** Expiry, linkin geçerli olduğu zaman penceresini (örn. dakikalar veya saatler) sınırlar. İmza doğrulaması, gizli anahtar olmadan URL’lerin üretilemeyeceğini ve kiracı ile kaynağın bağlı olduğunu garanti eder. Link paylaşımının etkisi azalır; kiracılar birbirinin kaynakları için geçerli imza üretemediğinden çapraz kiracı dosya erişimi engellenir.

## Alternatives Considered

- **API tüm dosya içeriğini stream eder:** Her indirme için API’nin okuması ve stream etmesi gerekirdi; gecikme, bellek ve CPU artardı. CDN tabanlı sunum tercih edildi.

- **Genel veya uzun ömürlü depolama URL’leri:** Dosyaları URL’e sahip herkese açar ve iptali zorlaştırırdı. Reddedildi.

- **CDN’e istek başına kimlik doğrulama:** CDN’in her istekte API veya auth servisine geri çağrı yapması gerekirdi; gecikme ve karmaşıklık ekler. İmzalı URL’ler kenarda durum bilgisiz doğrulama sağlar. İlk tasarım için reddedildi.

## Consequences

### Pozitif

- API dosya içeriğini stream etmediği için hafif kalır; dosya sunumu CDN üzerinden ölçeklenir ve bant genişliği CDN’e delege edilir.
- Erişim süre sınırlı ve kiracıya bağlıdır; imzalı URL yapısı URL üzerinde gereksiz hassas veri taşımadan dosya erişim kontrolü sağlar.

### Negatif

- CDN veya doğrulama katmanının imza gizli anahtarı ile senkron tutulması ve güvenli şekilde yönetilmesi gerekir; saat kayması (clock skew) süre doğrulamasını etkileyebilir.
- İmzalı linkler süre dolana kadar paylaşılabilir; bu kullanılabilirlik için bilinçli bir trade-off olmakla birlikte bazı kullanım senaryolarında ek politika veya denetim gerektirebilir.

## When to Revisit

Yeniden değerlendir: dosya erişiminin anında iptali istek başına auth veya iptal listeli token gerektirirse; veya düzenleyici gereklilik daha sıkı erişim kontrolü talep ederse.
