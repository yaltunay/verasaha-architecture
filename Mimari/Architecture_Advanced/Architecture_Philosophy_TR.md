Bu doküman VeraSaha mimarisinin teknik yaklaşımını açıklamak amacıyla hazırlanmıştır ve yalnızca mimari referans niteliğindedir. Hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Sistem hedefleri

VeraSaha, saha operasyonları ve toplantı yönetimi için çok kiracılı bir SaaS platformudur. Temel hedefler: katı kiracı izolasyonu, web ve mobil istemciler için öngörülebilir ve dayanıklı senkronizasyon, güvenli dosya işleme ve küçük bir ekibin anlayıp işletebileceği bir operasyonel modeldir. İşlevsel özellikler bu temellerin üzerine inşa edilir; güvenlik, gizlilik ve operasyonel olgunluk sonradan eklenen katmanlar değil, ilk tasarımın parçasıdır.

## Tasarım kısıtları

Mimariyi belirleyen temel kısıtlar vardır: CORS, sertifika ve rate limiting’i basitleştirmek için tek bir API host (`api.verasaha.com`); kiracı başına veritabanı yerine TenantId + global filtrelerle mantıksal izolasyon; zayıf veya kesintili bağlantı varsayımıyla çevrimdışı öncelikli kullanım; ve sınırlı operasyon ekibiyle sürdürülebilir bir dağıtım modeli. Bu kısıtlar, ters proxy – API – depolama gibi katmanlar arasında net sınırlar, basit veri akışları ve çalıştırma zamanında minimum “büyü” kullanımını teşvik eder.

## SaaS mimari prensipleri

Sistem, klasik SaaS prensiplerini izler: tek kod tabanı, çok kiracı; kiracı farkındalığı olan veri modeli; ortam ayrımı (dev/test/prod) ve yapılandırmanın kodla yönetilmesi; tüm kiracılar için tutarlı API sözleşmeleri. Özellikler kod seviyesinde kiracıdan bağımsız, çalışma zamanında ise kiracı kapsamlı olacak şekilde tasarlanır; böylece davranış tekrar edilebilir, veri ise ayrıştırılmış kalır. Geriye dönük uyumluluk, halka açık API’ler için varsayılan kabul edilir; değişiklikler mümkün olduğunca kırıcı olmayan, versiyon toleranslı sözleşmelerle sunulur.

## Çok kiracılı felsefe

Çok kiracılılık mimarinin merkezi eksenlerinden biridir; ayrıntı değil. Her istek `X-Tenant-Key` ve JWT içindeki `tenantId` üzerinden açık bir kiracı bağlamında işlenir; veri modeli kiracı kapsamlı varlıklarda TenantId taşır; EF Core global query filter’ları tüm sorgulara kiracı koşulu ekler; arka plan işlemleri (SyncInbox, NotificationOutbox) ve imzalı URL ile dosya erişimi kiracı farkındalığıyla tasarlanır. Felsefe, çapraz kiracı veri açığa çıkmasının tek bir unutulmuş `WHERE` koşuluna değil, birden fazla bağımsız korumanın aynı anda bozulmasına bağlı olması gerektiğidir.

## Güvenlik-öncelikli bakış

Güvenlik, kontrol listesi değil değişmezler (invariant) seti olarak ele alınır. “Çapraz kiracı veri erişimi yok”, “Kimlik doğrulamasız kiracı verisi yok”, “Kod veya logda secret yok” gibi prensipler; başlık tabanlı kiracı çözümlemesi, JWT tenant bağlama, ters proxy’de TLS sonlandırma, imzalı CDN URL’leri ve redaction’lı yapılandırılmış loglama gibi detayları yönlendirir. Sistem, handler’lara dağılmış noktasal kontroller yerine, denetlenebilir ve tekrarlanabilir mekanizmaları (başlıklar, claim’ler, global filtreler) tercih eder.

## Gizlilik odaklı tasarım

Gizlilik, teknik modelin içine gömülüdür: kiracı ve kullanıcı kapsamı, hassas alan değeri içermeyen denetim logları, açık dizin listeleri yerine imzalı URL’ler ve secret’ların veri değil yapılandırma olarak ele alınması. Mimari, saklama süresi ve veri sahibi talepleri süreçlerinin uygulanmasını kolaylaştırmak için veriyi yerelleştirir (kiracı bazında) ve izlenebilir kılar (CorrelationId, AuditLog, SyncInbox geçmişi). Gizlilik, sadece uyumluluk kontrol listesi değil; veri akışları ile iş amacı arasındaki hizalanma olarak görülür.

## Operasyonel olgunluk

Operasyonel olgunluk; gözlemlenebilirlik, kontrollü hata durumları ve runbook’larla yönetilebilir davranış etrafında şekillenir. Yapılandırılmış loglar, metrikler ve izler gözlem sinyallerini sağlar; rate limiting, sağlık kontrolleri ve API–worker–depolama ayrımı kontrol noktalarını oluşturur. Sistem, yük altında “çok yardımsever” davranmak yerine (sessizce bozulmak yerine) deterministik davranışları (429 döndürmek, kota uygulamak) tercih eder. Production olgunluğu yalnızca uptime ile değil, eldeki sinyallerle olayların ne kadar hızlı teşhis ve onarılabildiğiyle tanımlanır.

## Trade-off düşüncesi

Mimari, trade-off’ları görünür kılar: kiracı başına DB yerine tek DB + mantıksal izolasyon; her istekte stream yerine imzalı URL + CDN sunumu; istek içinde senkron bildirim yerine Outbox ve eventual consistency tercih edilir. Bu tercihler; işletilebilirlik, maliyet ve kavramsal sadeliği, örneğin süre dolana kadar link paylaşımı veya bildirimlerin zamanla teslimi gibi sınırlı olumsuzluklar karşılığında öncelemektedir. Yol gösteren ilke, trade-off’ları açıkça ifade etmek, ADR’lerde dokümante etmek ve ürün veya uyumluluk ihtiyaçları değiştiğinde evrime izin vermektir.

