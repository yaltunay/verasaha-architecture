# VeraSaha Mimarisi

Kurumsal seviyede çok kiracılı (multi-tenant) SaaS mimari vaka çalışması.

Bu depo yalnızca **mimari ve operasyon dokümantasyonu** içerir.  
Herhangi bir uygulama kodu, yapılandırma dosyası veya altyapı script’i içermez.

Bu repository, VeraSaha platformunun:

- Mimari tasarım kararlarını
- Güvenlik ve izolasyon stratejisini
- Trade-off analizlerini
- Production işletim modelini
- Reddedilen alternatifleri

dokümante etmek amacıyla oluşturulmuştur.

---

## 🎯 VeraSaha Nedir?

VeraSaha, saha operasyonları ve görüşme yönetimi için tasarlanmış çok kiracılı bir SaaS platformudur.

Temel yetenekleri:

- Çok kiracılı izolasyon (subdomain UX, API tarafında header tabanlı tenant çözümleme)
- Offline-first mobil senkronizasyon (idempotent apply + delta changes modeli)
- Güvenli dosya yükleme + kota yönetimi
- CDN üzerinden imzalı URL ile güvenli dosya sunumu
- Arka plan bildirim sistemi (Outbox pattern + worker)
- Gözlemlenebilirlik (structured logging, metrics, tracing)
- Reverse proxy + HTTPS production uyumluluğu

---

## 🏗 Güncel Sistem Topolojisi

- **verasaha.com** → Landing (Next.js) + root login/discovery
- **{tenant}.verasaha.com** → Tenant UI host’ları
- **app.verasaha.com** → Admin / Panel (Angular)
- **api.verasaha.com** → Tek backend API (tenant bilgisi `X-Tenant-Key` ile taşınır)
- **cdn.verasaha.com** → Dosya sunumu (imzalı URL doğrulamalı)
- **Mobile (Flutter)** → Offline-first senkronizasyon modeli

### Rezerve Subdomain’ler
`www`, `api`, `admin`, `app`, `debug`, `test`

### Reverse Proxy
Nginx + Let's Encrypt (TLS sonlandırma ve yönlendirme)

---

## 🔐 Çok Kiracılı İzolasyon Garantileri

Sistem izolasyonu yalnızca uygulama kodu ile değil, mimari seviyede sağlanır.

1. **API-Only Host Modeli**  
   Tüm backend trafiği `api.verasaha.com` üzerinden akar.

2. **Zorunlu Tenant Header**  
   Korunan uç noktalarda `X-Tenant-Key` başlığı zorunludur.

3. **Strict JWT Tenant Binding**  
   JWT içindeki tenantId claim’i ile header’dan çözülen tenant eşleşmek zorundadır.  
   - Claim yok → 401  
   - Claim mismatch → 403  

4. **EF Core Global Query Filter**  
   Tüm tenant-scoped veriler otomatik filtrelenir.

5. **Tenant-Scoped Inbox / Outbox Tabloları**  
   SyncInbox ve NotificationOutbox tenant izolasyonu sağlar.

6. **CDN İmzalı URL Modeli**  
   Dosya erişimi tenantId + expiry + HMAC imzası ile doğrulanır.

Cross-tenant veri erişimi yapısal olarak engellenmiştir.

---

## 🧠 Mimari Yaklaşım

### Onion Architecture
Domain katmanı dış bağımlılık içermez.  
Application katmanı use-case’leri orkestre eder.  
Persistence ve Infrastructure katmanları implementasyon detaylarını içerir.

### Domain Kuralları
- Meeting aggregate immutability
- Zorunlu değişiklik nedeni (change reason)
- ParticipationType (Planned / Actual)
- Devamsızlıkta AbsenceReason zorunluluğu

### Offline-First Senkronizasyon
- `/api/sync/changes`
- `/api/sync/apply`
- SyncInbox `(TenantId, UserId, OpId)`
- UTC-only zaman disiplini
- Tombstone modeli

### Dosya Stratejisi
- Whitelist + magic number sniffing
- Tenant storage quota enforcement
- API file streaming yok
- CDN signed URL

### Bildirim Stratejisi
- Outbox pattern
- Worker tabanlı dispatch
- Retry / backoff / dead-letter
- Development ortamında noop sender

---

## 📊 Gözlemlenebilirlik

- Structured logging (Serilog)
- CorrelationId middleware
- Sensitive data redaction
- Prometheus metrics (`/metrics` korumalı)
- `notifications.outbox.pending` gauge
- OpenTelemetry tracing  
  - Development: Console exporter  
  - Production: OTLP exporter (config-driven)

---

## ⚙️ Operasyonel Prensipler

- Fail-fast JWT yapılandırması
- Rate limiting (auth vs write partitioning)
- Upload abuse ve kota kontrolü
- AuditLog ile tüm mutasyonların kaydı
- UTC zaman standardı
- Loglarda secret bulunmaması

---

## 🎓 Bu Repository’nin Amacı

Bu repository:

- Gerçek bir SaaS sisteminin mimari tasarımını göstermek
- Çok kiracılı izolasyon stratejisini belgelemek
- Enterprise-grade karar süreçlerini görünür kılmak
- Teknik vaka çalışması sunmak

için oluşturulmuştur.

Bu bir demo proje değil; production odaklı bir mimari referanstır.

---

## 👤 Yazar

Yücel ALTUNAY
