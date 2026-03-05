# ADR-004 Bildirimler için Outbox Paterni

> Bu doküman VeraSaha platformu için alınan mimari kararları kayıt altına almak amacıyla hazırlanmıştır.  
> Yalnızca teknik referans niteliğindedir ve hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Status

Kabul edildi (Accepted)

## Context

Platform, alan olayları gerçekleştiğinde (örn. toplantı hatırlatıcısı) arka plan bildirimleri (örn. FCM ile push) göndermelidir. İstek işleyen koddan doğrudan göndermek HTTP yanıt süresini dış servis gecikmesine bağlar ve işlem commit edildikten sonra süreç çökerse veya dış çağrı başarısız olursa bildirimin kaybolma riski oluşturur. Güvenilir, en az bir kez teslimat ve alan kalıcılığı ile bildirim dağıtımı arasında net ayrım gereklidir.

## Decision Drivers

1. **Güvenilirlik:** İşlem çökerse veya FCM commit sonrası başarısız olursa bildirimler kaybolmamalı; aynı transaction'da Outbox'a yazım dayanıklılık sağlar.
2. **İstek yolu ayrımı:** HTTP yanıtı FCM beklememeli; arka plan worker retry, backoff ve dead-letter yönetir.
3. **Tutarlılık:** Idempotency key ile en az bir kez teslimat; eventual consistency hatırlatıcılar için kabul edilebilir.

## Decision

**Bildirimler için Outbox paterni** kullanılmaktadır:

- **Arka plan bildirimleri:** Bildirimler controller veya handler’lardan senkron gönderilmez. Bildirim tetikleyen bir alan olayı oluştuğunda, domain değişikliği ile **aynı veritabanı transaction’ında** **NotificationOutbox** tablosuna bir satır yazılır. HTTP yanıtı FCM veya başka bir dış servisi beklemeden tamamlanabilir.

- **Güvenilir mesaj dağıtımı:** Arka planda bir worker (veya zamanlanmış iş) NotificationOutbox’tan satırları okur ve FCM’e (veya geliştirme ortamında noop sender) gönderir. Worker başarısız gönderimleri yeniden deneyebilir, backoff uygulayabilir ve kalıcı başarısız öğeleri dead-letter yoluna taşıyabilir. Handler ve controller’lar FCM’i doğrudan çağırmaz; teslimat istek yolundan ayrılmıştır.

- **Yeniden deneme stratejisi:** Worker geçici hatalar için backoff ile yeniden deneme uygular. Idempotency key’ler (örn. TenantId + IdempotencyKey) aynı mantıksal olayın iki kez kuyruğa alınmasını önler ve güvenli yeniden denemelere izin verir. Hedef en az bir kez teslimattır; çoğaltma baskılama tüketicide veya mümkün olduğunda idempotent bildirim semantiği ile ele alınır.

- **Eventual consistency:** Bildirimler domain değişikliği commit edildikten sonra zamanla gönderilir. Anında teslimat için güçlü garanti yoktur; sistem bildirim kaybetmemeyi katı gerçek zamanlı teslimata tercih eder. Hatırlatıcılar ve benzeri kullanım senaryoları için bu kabul edilebilir.

## Alternatives Considered

- **İstek handler’dan doğrudan gönder:** Yanıtı FCM’e bağlar ve commit sonrası gönderim öncesi süreç sonlanırsa bildirimin kaybolma riski oluşturur. Reddedildi.

- **Yalnızca dış mesaj kuyruğu:** Altyapı ve operasyonel karmaşıklık ekler; Outbox “kuyruğa al” adımını domain yazımı ile aynı transaction’da tutar, çift yazma ve tutarlılık sorunlarını önler. Ölçekleme için dış kuyruk sonradan eklenebilir. İlk tasarım için reddedildi.

- **Handler’dan fire-and-forget:** Kalıcılık yok; commit sonrası çökme veya hata bildirimi kaybettirir. Reddedildi.

## Consequences

### Pozitif

- Transaction commit edildikten sonra bildirimler Outbox tablosunda kalıcıdır; istek yolu FCM gecikmesinden bağımsız ve daha hızlı çalışır.
- Yeniden deneme ve dead-letter işleme tek bir worker bileşeninde merkezileştirilir; aynı transaction içinde “domain kaydedildi ama bildirim kayboldu” boşluğu mimari olarak kapatılır.

### Negatif

- Bildirimler eventual consistency ile teslim edilir; zamanında veya sıralı teslimat beklentileri ürün seviyesinde doğru yönetilmelidir.
- Worker süreçlerinin çalışır durumda tutulması ve IdempotencyKey tasarımının iyi dokümante edilmesi gerekir; aksi halde kuyrukta çoğaltmalar veya birikmiş işler operasyonel risk oluşturabilir.

## When to Revisit

Yeniden değerlendir: güçlü gerçek zamanlı teslimat gerekirse; veya ölçek DB Outbox yerine özel mesaj broker (RabbitMQ, SQS) gerektirirse.
