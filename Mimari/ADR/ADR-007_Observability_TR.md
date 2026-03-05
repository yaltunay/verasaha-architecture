# ADR-007 Gözlemlenebilirlik Yığını

> Bu doküman VeraSaha platformu için alınan mimari kararları kayıt altına almak amacıyla hazırlanmıştır.  
> Yalnızca teknik referans niteliğindedir ve hukuki danışmanlık veya resmi güvenlik sertifikasyonu değildir.

## Status

Kabul edildi (Accepted)

## Context

Platform production operasyonlarını desteklemelidir: arıza teşhisi, performans anlama ve kötüye kullanım veya anomali tespiti. API, worker’lar ve diğer bileşenlerde tutarlı çalışan, hassas veri açığa çıkarmayan ve sürdürülemez kardinalite (örn. boyutu patlayan kiracı başına metrikler) yaratmayan bir loglama, metrik ve izleme yaklaşımı gereklidir.

## Decision Drivers

1. **Olay müdahalesi ve denetim:** CorrelationId ve yapılandırılmış loglar istek seviyesi izlemeyi sağlar; AuditLog uyumluluk ve KVKK ile ilgili denetimi destekler; redaction gizliliği korur.
2. **Operasyonel görünürlük:** Metrikler ve isteğe bağlı tracing kapasite planlaması ve anomali tespitini destekler; /metrics bilgi sızıntısını önlemek için korumalıdır.
3. **Kardinalite kontrolü:** Metriklerde yüksek kardinaliteli kiracı etiketi yok; span'larda isteğe bağlı tenant_id ile gözlemlenebilirlik yığını yönetilebilir kalır.

## Decision

Aşağıdaki unsurlarla **merkezi gözlemlenebilirlik yığını** benimsenmiştir:

- **Yapılandırılmış loglama:** Uygulama logları yapılandırılmıştır (örn. Serilog ile JSON veya anahtar-değer çıktısı). Her log girişi correlation_id, tenant_id, user_id (mevcut olduğunda) ve diğer bağlamı içerebilir. Loglar merkezi log deposunda toplama ve sorgulama için uygundur. **Hassas veri** (parolalar, token’lar, tam PII) loglara yazılmaz; redaction veya maskeleme uygulanır; operasyonel görünürlük güvenlik veya gizliliği ihlal etmez.

- **Metrikler:** Temel operasyonel metrikler standart bir formatta (örn. Prometheus) açığa çıkarılır. Örnekler: istek sayıları, hata oranları, outbox kuyruk derinliği (örn. notifications.outbox.pending). Metrik uç noktası (örn. /metrics) korumalıdır (örn. yalnızca Admin JWT veya dahili ağ); kamuya açık scrape edilmez. **Kardinalite** kontrol altındadır: çok fazla zaman serisi yaratacak metriklerde tenant_id gibi yüksek kardinaliteli etiketler kullanılmaz; metrik yığını yönetilebilir kalır.

- **İzleme (Tracing):** Dağıtık izleme (örn. OpenTelemetry) bir isteğin servisler veya bileşenler arasında takibini sağlar. Trace ID’leri ve correlation_id mümkün olduğunda hizalanır. Span’lar correlation_id, user_id ve isteğe bağlı tenant_id taşıyabilir (veya kardinaliteyi sınırlamak için tenant_id atlanabilir). Geliştirme ortamında konsol exporter kullanılabilir; production’da trace’ler merkezi analiz için OTLP uç noktası veya benzerine export edilebilir.

- **Merkezi gözlemlenebilirlik:** Loglar, metrikler ve trace’ler merkezi sistemlere (örn. log toplama, Prometheus, trace backend) gönderilecek veya oradan scrape edilecek şekilde tasarlanmıştır. Bu olay müdahalesi, kapasite planlaması ve denetimi destekler. Backend seçimi (örn. Grafana, Jaeger, Elasticsearch) operasyonel bir konudur; mimari yapılandırılmış, hassas veri içermeyen ve kardinalite bilinçli sinyaller üretmeyi taahhüt eder.

## Alternatives Considered

- **Yalnızca yapılandırılmamış loglar:** Sorgulama ve ilişkilendirme zorlaşır; yapılandırılmış loglama tercih edildi.

- **Her metrikte tenant_id:** Metrik depolama ve sorgulama maliyetinde yüksek kardinalite yaratırdı. Reddedildi; tenant yalnızca gerekli ve kardinalitenin sınırlı olduğu yerde kullanılır.

- **İzleme yok:** Servisler veya bileşenler arası teşhisi zorlaştırırdı. Opsiyonel ama desteklenen izleme tercih edildi.

- **Hata ayıklama için hassas veri loglama:** Güvenlik ve uyumluluk riski. Reddedildi; redaction zorunludur.

## Consequences

### Pozitif

- Log, metrik ve trace için tek ve tutarlı bir yaklaşım sunar; correlation_id log ve trace’lerin olay incelemesi için birbirine bağlanmasını sağlar.
- Metrikler uyarı ve kapasite planlamasını destekler; redaction ve kardinalite kontrolleri güvenlik ve gözlemlenebilirlik yığınının ölçeklenebilirliğini korur.

### Negatif

- Log/metrik/trace backend’lerine operasyonel bağımlılık yaratır; enstrümantasyonun tüm bileşenlerde tutarlı uygulanması gerekir.
- Kiracı seviyesi hata ayıklama çoğu durumda yüksek kardinaliteli metrikler yerine log ve trace sorguları gerektirir; geliştirme ve production ortamlarında farklı exporter kullanılması (konsol vs OTLP) davranış farklarına yol açabilir.

## When to Revisit

Yeniden değerlendir: uyumluluk veya denetim sınırlı kiracı seviyesi metrikler gerektirirse; veya merkezi log/trace backend seçimi değişirse.
