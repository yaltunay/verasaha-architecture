# ADR-006 Reverse Proxy Dağıtımı

> Bu doküman VeraSaha platformu için alınan mimari kararları kayıt altına almak amacıyla hazırlanmıştır.  
> Yalnızca teknik referans niteliğindedir ve hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Status

Kabul edildi (Accepted)

## Context

Platform birden fazla host sunar: api.verasaha.com (API), cdn.verasaha.com (dosya sunumu), app.verasaha.com (admin/panel) ve tenant subdomain’leri (UI). TLS sonlandırma, sertifikaların tek noktadan yönetimi ve dış isteklerin uygun dahili servise yönlendirilmesi gereklidir. Dahili servisler doğrudan internete açılmamalı; reverse proxy HTTP/HTTPS trafiği için tek kamu giriş noktası olmalıdır.

## Decision Drivers

1. **TLS ve sertifikalar:** Proxy'de merkezi sonlandırma; Let's Encrypt veya sertifika yönetimi tek yerde; backend'ler kamuya açık değil.
2. **Yönlendirme ve güvenlik:** Host tabanlı yönlendirme (landing, API, CDN, app, kiracı UI); X-Forwarded-* ile doğru istemci IP ve şema (rate limiting, audit).
3. **Operasyonel basitlik:** Sertleştirilecek ve izlenecek tek kenar; dahili servisler yalnızca localhost veya özel ağda dinler.

## Decision

Tüm HTTP sunan bileşenlerin önünde **reverse proxy katmanı** kullanılmaktadır:

- **Reverse proxy katmanı:** Bir reverse proxy (örn. Nginx) HTTPS için tek kamu giriş noktasıdır. İstemci bağlantılarını kabul eder, TLS’i sonlandırır ve yapılandırmaya göre istekleri uygun backend’e (API, CDN veya app host) iletir. Backend’ler localhost veya özel ağda dinleyebilir; TLS işlemesi veya internetten doğrudan erişilebilir olması gerekmez.

- **TLS sonlandırma:** TLS (HTTPS) reverse proxy’de sonlandırılır. Sertifikalar (örn. Let’s Encrypt) proxy’de yönetilir. Backend’ler düz HTTP alır (genelde güvenilir ağ veya loopback üzerinden). Bu sertifika yenilemeyi merkezileştirir ve her backend’de yapılandırma ve saldırı yüzeyini azaltır.

- **Dahili servis açığa çıkarma:** Dahili servisler (API, CDN doğrulama, app sunucusu) genel portlarda açığa çıkmaz. Localhost veya özel IP’ye bağlıdır ve yalnızca reverse proxy tarafından erişilir. Bu açığa çıkmayı sınırlar ve güvenlik duvarı kurallarının dahili host’lara doğrudan erişimi kısıtlamasına olanak tanır.

- **Host tabanlı yönlendirme:** Proxy host (ve isteğe bağlı path) ile yönlendirir. Örn. api.verasaha.com → API backend, cdn.verasaha.com → CDN veya dosya doğrulama servisi, app.verasaha.com → admin uygulaması, {tenant}.verasaha.com → tenant UI. Kamu host adları ile backend servisleri arasında net eşleme sağlanır ve host başına CORS ve güvenlik politikası desteklenir.

## Alternatives Considered

- **Her backend’de TLS:** Her servisin kendi sertifikası ve yenilemesi gerekirdi; yapılandırma ve operasyonel yük artardı. Sertifika sonlandırmanın proxy’de merkezileştirilmesi tercih edildi.

- **Reverse proxy yok (backend’lerin doğrudan açığa çıkması):** Her backend’in kamuya açık ve TLS işlemesi gerekirdi; kenarda WAF, rate limiting veya merkezi loglama eklemek zorlaşırdı. Reddedildi.

- **Yalnızca bulut yük dengeleyici:** Bulut LB benzer rolü üstlenebilir; karar kavramsal ve çoğu zaman gerçek uygulama olarak reverse proxy (örn. Nginx) kullanmaktır. Proxy eşdeğer özelliklere sahip yönetimli bir LB ile değiştirilirse aynı ilkeler geçerlidir.

## Consequences

### Pozitif

- TLS ve sertifikalar için tek bir katmanda yönetim sağlar; dahili servislerin doğrudan internete açılmaması güvenlik yüzeyini daraltır.
- Host tabanlı yönlendirme, landing, API, CDN, app ve kiracı UI’lar arasında net bir eşleme sağlar; X-Forwarded-* başlıkları doğru istemci IP ve protokol bilgisini backend’lere taşıyabilir.

### Negatif

- Reverse proxy tek hata noktası hâline gelir; yüksek erişilebilirlik ve ölçek için proxy katmanının da yedekli ve izleniyor olması gerekir.
- Proxy yapılandırmasının backend dağıtımı ile senkron tutulmaması veya güvenilir proxy ayarlarının yanlış yapılması, istemci IP/şema bilgisinin yanlış yorumlanmasına ve güvenlik/politika sorunlarına yol açabilir.

## When to Revisit

Yeniden değerlendir: ölçek veya uyumluluk proxy önünde yönetimli LB veya WAF gerektirirse; veya çok bölgeli dağıtım bölgeye özel kenar yönlendirme gerektirirse.
