# ADR-005 Çevrimdışı Senkronizasyon Mimarisi

> Bu doküman VeraSaha platformu için alınan mimari kararları kayıt altına almak amacıyla hazırlanmıştır.  
> Yalnızca teknik referans niteliğindedir ve hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Status

Kabul edildi (Accepted)

## Context

Mobil istemciler (örn. Flutter) bağlantının kesintili olduğu ortamlarda çalışır. Kullanıcılar veriyi çevrimdışı görüntüleyebilmeli ve değişiklik yapabilmeli; bağlantı sağlandığında değişiklikler senkronize edilmelidir. Çevrimdışı öncelikli istemcileri destekleyen, mutasyonların çift uygulanmasını önleyen ve çakışan veya kayıp güncelleme olmadan istemci ile sunucuyu zamanla tutarlı tutan bir senkron modeli gereklidir.

## Decision Drivers

1. **Çevrimdışı öncelikli UX:** Saha kullanıcıları bağlantı olmadan çalışabilmeli; sync delta ve idempotent apply ile yeniden denemeler güvenli olmalı.
2. **Veri bütünlüğü:** Aynı OpId çift uygulama yapmamalı; SyncInbox (TenantId, UserId, OpId) idempotency ve kiracı izolasyonu sağlar.
3. **Operasyonel netlik:** Tek senkron sözleşmesi (changes + apply); UTC ve tombstone modeli istemci ile sunucuyu hizalı tutar.

## Decision

Aşağıdaki unsurlarla **çevrimdışı senkronizasyon mimarisi** benimsenmiştir:

- **Çevrimdışı öncelikli mobil istemciler:** İstemciler yerel bir depo tutar ve çevrimiçi olmadan veri okuyup yazabilir. Tüm mutasyonlar yerel kaydedilir ve mümkün olduğunda sunucuya gönderilir. UI yerel veriden çalışabilir; senkron arka planda çalışır. Sunucu bir **changes** uç noktası (belirli bir zamandan bu yana değişikliklerin deltası) ve bir **apply** uç noktası (istemci mutasyon batch’i gönderir) sunar.

- **Delta senkronizasyonu:** Sunucunun **changes** uç noktası yalnızca istemcinin `since` parametresinden (örn. serverTimeUtc) bu yana oluşturulan, güncellenen veya silinen varlıkları (veya ilgili alanları) döner. Tombstone’lar (silindi işaretleri) istemcinin yerel kopyaları kaldırması için dahil edilir. Bu yük boyutunu azaltır ve artımlı senkron sağlar. Tüm zaman damgaları UTC’dir (ISO 8601).

- **Idempotent apply uç noktası:** İstemci her işlem için benzersiz bir **OpId** ile mutasyonlar gönderir. Sunucu uygulanan işlemleri (TenantId, UserId, OpId) anahtarlı **SyncInbox** tablosunda saklar. İstemci aynı OpId ile yeniden denerse (örn. ağ hatası sonrası) sunucu mutasyonu tekrar uygulamaz; önceden saklanan sonucu döner. Bu çift uygulamayı ve veri bozulmasını önler ve yeniden denemeleri güvenli kılar.

- **Sync inbox paterni:** SyncInbox hem idempotency hem kiracı/kullanıcı izolasyonu sağlar. Yalnızca ilgili kiracı ve kullanıcı kendi işlemlerini görebilir veya tekrarlayabilir. Sunucu gerektiğinde mutasyonları sırayla uygular ve istemcinin uzlaştırması için başarı veya yapılandırılmış hata döner.

## Alternatives Considered

- **Yalnızca her zaman çevrimiçi:** Bağlantının zayıf olduğu saha kullanıcı ihtiyacını karşılamazdı. Reddedildi.

- **Idempotency olmadan son yazan kazanır:** Yeniden denemede çift uygulama ve tutarsız durum riski oluştururdu. Reddedildi.

- **Sunucu otoriteli çakışma çözümü (örn. CRDT):** Belirli varlıklar için eklenebilir; ilk tasarım için istemci OpId idempotency ve gerektiğinde sıralı apply ile mutasyon uygular. Daha karmaşık çakışma yönetimi sonradan eklenebilir.

## Consequences

### Pozitif

- Çevrimdışı öncelikli UX sağlar; istemciler bağlantı olmasa bile çalışabilir ve OpId tabanlı idempotent apply ile yeniden denemeler güvenlidir.
- Delta senkronizasyonu bant genişliğini azaltır; SyncInbox kiracı ve kullanıcı izolasyonunu korurken UTC + tombstone modeli istemci ile sunucuyu hizalı tutar.

### Negatif

- İstemcilerin OpId üretmesi ve saklaması, sunucunun ise SyncInbox tablosunu yönetmesi gerekir; bu da ek depolama ve operasyonel karmaşıklık getirir.
- “Son apply kazanır” dışındaki gelişmiş çakışma çözümü bu mimarinin dışında bırakılmıştır; yüksek çakışma oranlarında ek tasarım ve ürün kararları gerektirebilir.

## When to Revisit

Yeniden değerlendir: çakışma oranı veya ürün CRDT veya sunucu otoriteli çakışma çözümünü gerektirirse; veya SyncInbox saklama/kardinalite operasyonel sorun olursa.
