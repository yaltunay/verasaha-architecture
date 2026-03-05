Bu doküman VeraSaha mimarisinin teknik yaklaşımını açıklamak amacıyla hazırlanmıştır ve yalnızca mimari referans niteliğindedir. Hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Dağıtım topolojisi

Platform; kenarda ters proxy, onun arkasında web arayüzleri (landing, admin, kiracı UI’ları), tek bir backend API, arka plan worker’lar, veritabanı, dosya depolama ve CDN/dosya sunum katmanı olacak şekilde ayrıştırılmıştır. Tüm dış HTTP(S) trafiği ters proxy’de sonlanır ve host’a göre ilgili servise yönlendirilir. API ve worker’lar, yapılandırmanın, secret’ların ve ağ sınırlarının açık olduğu bir ortamda çalışır; böylece dağıtımlar otomasyona uygun ve geri alınabilir hale gelir.

## Reverse proxy modeli

Reverse proxy (örn. Nginx/NPM) `verasaha.com`, `app.verasaha.com`, `api.verasaha.com` ve `cdn.verasaha.com` için tek kamu giriş noktasıdır. TLS’i sonlandırır, sertifikaları yönetir (örn. Let’s Encrypt) ve dahili HTTP üzerinden API ve diğer servislere iletir. Host tabanlı yönlendirme hem sorumluluk ayrımını hem de host başına politika tanımlamayı (CORS, rate limiting, cache) mümkün kılar; aynı zamanda basit bir zihinsel model sunar: proxy arkasındaki hiçbir bileşen doğrudan internete açılmaz.

## Secret yönetimi

JWT imzalama anahtarları, veritabanı bağlantı dizeleri, CDN imza anahtarı, FCM kimlik bilgileri gibi secret’lar, kaynak koda gömülmek veya loglanmak yerine ortam değişkenleri veya secret store’lar üzerinden enjekte edilir. Mimari, secret rotasyonunun kod değişikliği gerektirmeden yapılabilmesini ve ortamlar arasında farklı secret set’leri kullanılmasını varsayar. Yapılandırma dosyaları ve `.env` örnekleri gerçek değer içermez; yalnızca neye ihtiyaç olduğunu gösterir. Production ortamda secret’ların güvenli bir sağlayıcıdan veya CI/CD tarafından enjekte edilmesi beklenir.

## Gözlemlenebilirlik

Production olgunluğu, davranışı anlayacak kadar sinyal üretmeye bağlıdır. API ve worker’lar; CorrelationId, kiracı/kullanıcı bağlamı (uygun olduğunda) ve net olay adları içeren yapılandırılmış loglar üretir. Metrikler (örn. istek hızı, hata oranı, outbox derinliği) korumalı bir `/metrics` uç noktasından expose edilir ve metrik backend’i tarafından scrape edilir. Opsiyonel dağıtık izleme, controller–handler–worker akışlarını birbirine bağlar. Bu yapı taşları, “ne bozuldu?”, “hangi kiracı için?” ve “ne sıklıkta?” gibi soruların tahmin yerine veriyle cevaplanmasını sağlar.

## Hata yönetimi

Sistem, sessiz bozulma yerine açık hata ve sınırlı bozulmayı tercih eder. API uç noktaları; değişmezler ihlal edildiğinde veya kapasite aşıldığında anlamlı durum kodları (401/403, 409, 429, 5xx) döner. Senkronizasyon sözleşmeleri, kısmi hataların durumu bozmadan yeniden denemeye uygun olacak şekilde tasarlanır. Outbox ve SyncInbox, süreçler çöktüğünde işin kaybolmamasını sağlayan kalıcılık katmanları ekler. Rate limiting ve kotalarla birlikte bu yaklaşım, yerel hataların sistem geneline yayılma ihtimalini azaltır.

## Arka plan worker’lar

Arka plan worker’lar; NotificationOutbox’tan bildirim gönderimi gibi etkileşimli olmayan yükleri işler. Sorumlulukları istek/yanıt yolundan ayrıldığı için HTTP gecikmesi, yavaşlayan dış servislerden bağımsız kalır. Worker’lar, backoff’lu yeniden deneme ve dead-letter davranışları uygular; geçici sorunlar kalıcı veri kaybına yol açmaz, kalıcı sorunlar ise sessizce dönmek yerine manuel müdahale için yüzeye çıkarılır.

## Kota uygulaması

Kota uygulaması, tek bir kiracının paylaşılan kaynakları tüketmesini engeller. Depolama kotaları, bir kiracının ne kadar veri yükleyebileceğini sınırlar; kotayı aşma girişimleri net 4xx yanıtları ile sonuçlanır. Mimari, maliyet kontrolü ve kötüye kullanım senaryolarında hasarı sınırlamak için kota kontrollerinin API’de pahalı iş veya imzalı URL üretiminden önce yapılmasını öngörür.

## Rate limiting

Rate limiting en az iki seviyede uygulanır: brute-force ve kimlik bilgisi doldurma saldırılarına hassas olan kimlik doğrulama uç noktaları (`discover`, `login`) ve paylaşılan kaynakları ve veritabanı yükünü etkileyebilen yazma/mutasyon uç noktaları. Limitler IP veya tenant+kullanıcı bazında tanımlanır; meşru kullanım akışlarını korurken kötüye kullanımı kısıtlar. Sınırlar ters proxy’de ve/veya API katmanında uygulanabilir; kritik olan, kararların tutarlı ve gözlemlenebilir olmasıdır.

## Ölçekleme yaklaşımları

Mimari, öncelikle stateless bileşenlerin (ters proxy, API instance’ları, worker’lar) yük dengeleyici arkasında yatay çoğaltılması ve tek çok kiracılı veritabanında indeksleme/sorgu tasarımına özen gösterilmesiyle ölçeklenir. CDN tabanlı dosya sunumu, bant genişliği yoğun trafiği API’den uzaklaştırır. Kiracı izolasyonu mantıksal olduğu için yeni kiracılar eklemek yeni veritabanı tedarik etmeyi gerektirmez; ancak ağır kiracılar, indeks ayarlamaları, bölümlendirme stratejileri veya ileride sharding gibi evrimleri tetikleyebilir.

## Felaket kurtarma düşüncesi

Felaket kurtarma, çekirdek veriyi geri yüklemeye ve sistemi kontrollü bir sırayla ayağa kaldırmaya odaklanır. Veritabanı sistemin kayıt sistemi olarak görülür; yedeklenmeli ve geri yükleme testleri düzenli yapılmalıdır. Dosya depolama, gereken durumlarda dayanıklılık ve bölge farkındalığı ile yapılandırılmalıdır. Yapılandırma ve altyapı, kod veya belgeli script’ler ile ifade edilmelidir ki ortamlar yeniden yaratılabilsin. Gözlemlenebilirlik verileri (loglar, metrikler), olay zaman çizelgelerini yeniden kurmaya yetecek süre saklanmalıdır. Tasarım, ciddi bir olayda operatörlerin ad-hoc müdahaleler yerine bilinen adımlarla sistemi geri getirebilmesini hedefler.

