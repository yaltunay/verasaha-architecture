Bu doküman VeraSaha mimarisinin teknik yaklaşımını açıklamak amacıyla hazırlanmıştır ve yalnızca mimari referans niteliğindedir. Hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## API hata senaryoları

API, yaygın hata durumlarının açık ve teşhis edilebilir olacağı şekilde tasarlanmıştır. Kimlik doğrulama ve yetkilendirme hataları (401/403), iş kuralı çakışmalarından (409) ve aşırı yük veya altyapı sorunlarından (429/5xx) ayrıştırılır. Veritabanı veya depolama gibi bağımlılıklar kullanılamaz olduğunda API, belirsizce beklemek yerine hızlıca hata vermeli ve istemcilere ve loglara net sinyaller üretmelidir. Kiracı çözümlemesi hataları (geçersiz veya eksik `X-Tenant-Key`), her korunan çağrının bir kiracı beyan etmesi gerektiği prensibini korumak için istemci hatası olarak ele alınır.

## Senkronizasyon çakışma yönetimi

Senkronizasyon, `changes` ve `apply` uç noktalarına dayanan çevrimdışı öncelikli bir modelle çalışır. Zaman damgaları ve tombstone’lar çakışmaları en aza indirir, ancak tüm yarış durumları engellenemez. Temel model, belirli bir kayıt için “son başarılı apply kazanır” yaklaşımıdır; daha sıkı semantik gerektiğinde domain kurallarının eklenmesi beklenir. Değişiklikler SyncInbox ve denetim logları üzerinden kaydedildiğinden, operatörler güncelleme dizilerini yeniden kurabilir ve istisnai durumlarda veriyi kontrollü biçimde düzeltebilir.

## İdempotent işlemler

İdempotency, senkronizasyon ve arka plan süreçleri için temel bir güvenilirlik aracıdır. `(TenantId, UserId, OpId)` ile anahtarlanan SyncInbox tablosu, aynı işlemin tekrar gönderilmesinin mutasyonları ikinci kez uygulamamasını garanti eder. Bu, ağ kesintileri, istemci yeniden başlatmaları ve geçici API hataları boyunca yeniden denemeleri güvenli kılar. Benzer kalıplar, bildirim batch’leri için idempotency key’leriyle NotificationOutbox’ta da kullanılır. İdempotency, yalnızca veri bütünlüğünü korumakla kalmaz, istemci hata yönetimini de kolaylaştırır: istemciler, önceki denemenin tam nerede başarısız olduğunu bilmeden istekleri tekrar edebilir.

## Yeniden deneme stratejileri

Yeniden denemeler, değer kattıkları yerlerde uygulanır: arka plan worker’lar ve istemciler, geçici hatalar (timeout’lar, kısa süreli 5xx yanıtları) için artan bekleme süreli (exponential backoff) retry uygular. Mimari, kör, sık döngülü retry’leri teşvik etmez; bunun yerine retry’lerin oranı sınırlandırılır ve toplam deneme sayısı makul tutulur. Senkron API çağrılarında, tekrarlanan 4xx hataları sonlandırıcı kabul edilmeli, yalnızca gerçekten geçici görünen koşullarda retry yapılmalıdır. Geçici ve kalıcı hataların ayrımı, kısmi kesintiler sırasında kendi kendine DDoS yaratmamak için kritiktir.

## Bildirim yeniden denemeleri

NotificationOutbox, teslimatı domain yazımından ayırmak ve güvenilir retry desteği sağlamak için tasarlanmıştır. Worker’lar Outbox’tan kayıtları okur, teslimatı dener (örn. FCM’e) ve başarısızlık durumunda backoff ile sonraki denemeleri planlar. Belirli sayıda deneme sonrası veya belirli hata türlerinde (örn. geçersiz token) kayıtlar dead-letter kuyruğuna taşınabilir veya kalıcı olarak başarısız işaretlenebilir. Böylece bildirim altyapısındaki geçici kesintiler sessiz veri kaybına dönüşmez; problemli mesajlar incelenip manuel olarak ele alınabilir.

## Rate limit korumaları

Rate limiting, hem altyapıyı hem kiracıları aşırı yük ve kötüye kullanımdan korur. Limitler aşıldığında sistem 429 veya benzeri net durum kodlarıyla yanıt verir ve istemcilere geri çekilmeleri gerektiğini sinyaller. Rate limiting’in kendisi de retry’leri tetikleyebileceği için, thrash etkisini önlemek adına; örneğin Retry-After başlıkları veya istemci tarafı konvansiyonlarıyla, tekrar deneme davranışının yönlendirilmesi önemlidir. Doğru yapılandırılmış limitler, kısmi kesinti veya “sıcak” kiracılar senaryolarında çekirdek servislerin herkes için değil, ağırlıklı olarak etkilenmiş aktörler için bozulmasını sağlar.

## Circuit breaker bakışı

Belirli bir kütüphaneye bağlı olmadan, mimari “circuit breaker” tarzı düşünmeyi teşvik eder: bir aşağı akış sistemi kalıcı olarak başarısız oluyorsa, istemciler ve worker’lar tam trafik göndermeye devam etmek yerine hızlıca hata vermeli veya davranışı degrade etmelidir. Örneğin kritik olmayan entegrasyonların geçici olarak atlanması, sync frekansının azaltılması veya ağır arka plan işlerinin ertelenmesi bu kapsamdadır. Amaç, etkiyi yerelleştirmek ve operatörlere sorunları düzeltmeleri için zaman tanırken sistemin sürekli olarak hataları büyütmesini engellemektir.

## Gözlemlenebilirlik ile tespit

Güvenilirlik, anormalliklerin erken tespitine dayanır. Yapılandırılmış loglar, metrikler ve izler; hata oranları, gecikme sıçramaları, kuyruk derinlikleri ve olağandışı erişim desenleri için alarm üretmenin temelini sağlar. CorrelationId ve kiracı farkındalığı, sorunların küresel mi, belirli kiracılara mı yoksa belirli özelliklere mi özgü olduğunu anlamayı kolaylaştırır. Tasarım, alarmların gerçekten eyleme dönük olmasını; panolar ve sorguların “bir şeyler ters gidiyor” gözleminden “şu bileşen, şu nedenle bozuldu” sonucuna köprü kurabilmesini hedefler.

## Manuel kurtarma prosedürleri

Tüm hata senaryoları tamamen otomatikleştirilemez. Bu nedenle mimari, manuel müdahalelerin güvenli ve tekrarlanabilir olabilmesi için izlenebilirlik (AuditLog, SyncInbox, NotificationOutbox) ve net değişmezler etrafında şekillenir. Örnek prosedürler arasında belirli senkronizasyon işlemlerinin yeniden oynatılması veya iptali, takılı kalmış bildirimlerin yeniden kuyruklanması, problemli kiracıların geçici olarak devre dışı bırakılması veya sınırlı kapsamda yedekten veri geri yüklenmesi sayılabilir. Dokümante runbook’lar ve ADR’ler, bu müdahaleleri tasarlamak ve uygulamak için gerekli bağlamı sağlar.

