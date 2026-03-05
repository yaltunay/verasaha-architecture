# ADR-001 Çok Kiracılı İzolasyon Modeli

> Bu doküman VeraSaha platformu için alınan mimari kararları kayıt altına almak amacıyla hazırlanmıştır.  
> Yalnızca teknik referans niteliğindedir ve hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Status

Kabul edildi (Accepted)

## Context

VeraSaha, tek bir dağıtımdan birden fazla organizasyonu (kiracı) sunan çok kiracılı bir SaaS platformudur. Her kiracının verisinin kesin biçimde izole olması gerekir: çapraz kiracı veri erişimi olmamalı, sorgular veya arka plan işleriyle yanlışlıkla sızıntı olmamalı; senkron, bildirim ve dosya erişimi için net sınırlar tanımlanmalıdır.

## Decision Drivers

1. **Uyumluluk ve güven:** Kiracı veri izolasyonu SaaS ve KVKK için vazgeçilmezdir; bir kiracı diğerinin verisini asla görmemelidir.
2. **Operasyonel basitlik:** Tek veritabanı ve katı mantıksal izolasyon, kiracı başına DB veya şema yaklaşımından daha kolay işletilir.
3. **Savunma derinliği:** Global query filter’lar geliştirici hatası riskini azaltır; izolasyon yalnızca uygulama kodu değil veri katmanında da uygulanır.

## Decision

Aşağıdaki unsurlarla **çok kiracılı izolasyon modeli** benimsenmiştir:

- **SaaS çok kiracılı mimari:** Tek bir mantıksal uygulama tüm kiracıları sunar. Kiracı kimliği her istekte ve her veri erişim yolunda açıkça belirtilir. Tüm kiracı kapsamındaki varlıklar `TenantId` taşır ve kiracılar arasında paylaşılmaz.

- **Kiracı veri ayrımı:** Kiracı kapsamındaki tüm veriler `TenantId` sütunu ile saklanır. Okuma ve yazma yolları her zaman tek bir kiracı bağlamında çalışır. Hiçbir API veya worker tek bir işlemde birden fazla kiracının verisini döndürmez veya işlemez.

- **Global sorgu filtreleri:** EF Core global query filter’lar kiracı kapsamındaki varlıklara uygulanır. Sorgular, mevcut istek bağlamından türetilen bir `TenantId` koşulunu otomatik içerir. Uygulama kodu yanlışlıkla kiracı filtresini atlayamaz; çapraz kiracı sonuç kümeleri veri katmanında engellenir.

- **Kiracıya bağlı işlemler:** Senkron (changes/apply), bildirimler (Outbox) ve dosya erişimi kiracı (ve gerektiğinde kullanıcı) kapsamındadır. SyncInbox (TenantId, UserId, OpId); NotificationOutbox (TenantId, IdempotencyKey) kullanır. Worker’lar satırları kiracı farkındalığı ile işler; çapraz kiracı veri açığa çıkmaz veya gönderilmez.

## Alternatives Considered

- **Kiracı başına ayrı veritabanı:** Güçlü izolasyon sağlardı ancak operasyonel maliyet, migrasyon karmaşıklığı ve ölçekleme yükünü artırırdı. Tek veritabanı ve katı mantıksal izolasyon + global filter tercih edildi.

- **Kiracı başına şema:** Kiracı başına veritabanına benzer trade-off’lar; migrasyon ve araçlar karmaşıklaşır. Reddedildi.

- **Yalnızca uygulama seviyesinde kiracı kontrolü:** Kiracı filtresini yalnızca uygulama koduna bırakmak hata riski taşır. Güvenlik ağı olarak global query filter tercih edildi.

## Consequences

### Pozitif

- Tek veritabanı ile güçlü mantıksal izolasyon sağlanır; operasyon ve yedekleme süreçleri kiracı başına DB veya şemaya göre daha basittir.
- Global query filter’lar geliştirici hatası riskini azaltır ve kiracı bağlamının API, senkron ve worker bileşenlerinde tutarlı olmasını sağlar.

### Negatif

- Tüm kiracı kapsamındaki kod net bir kiracı bağlamında çalışmalıdır; yanlış yapılandırma (örneğin bağlamda kiracı eksikliği) boş sonuç veya hata üretebilir ve bu durumun doğru yönetilmesi gerekir.
- Performans ve indeksleme tasarımı, TenantId içeren tablolarda ek karmaşıklık getirir; ölçek arttıkça sorgu ve indeks stratejileri dikkatle gözden geçirilmelidir.

## When to Revisit

Yeniden değerlendir: düzenleyici veya sözleşmesel gereksinimler fiziksel ayrım (örn. kiracı başına DB veya bölge) talep ederse; kiracı ölçeği veya uyumluluk denetimleri daha güçlü izolasyon garantisi ihtiyacı gösterirse; veya çok bölgeli dağıtım kiracı–bölge bağlaması gerektirirse.
