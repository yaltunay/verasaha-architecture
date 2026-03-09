# Mimari Özet (Türkçe)

VeraSaha mimarisinin üst düzey özeti; İngilizce özet ile uyumludur.

---

## Özel Domain Desteği

- **Aynı web uygulaması:** Tek bir web uygulaması (aynı build/imaj) verasaha.com (landing), tenant subdomain'leri (örn. tenantA.verasaha.com) ve onaylı özel domain'leri (örn. saha.firma.com) sunabilir. Kiracı başına ayrı frontend build veya dağıtım gerekmez.
- **Host rolleri:** Web host yalnızca UX/erişim katmanıdır. API her zaman api.verasaha.com; dosya sunumu her zaman cdn.verasaha.com'dur.
- **Kiracı bootstrap:** Subdomain'lerde tenant anahtarı host'tan türetilir veya bootstrap ile doğrulanır. Özel domain'de frontend, host → tenant çözümlemesi için bootstrap mekanizması kullanır; ardından api.verasaha.com'a X-Tenant-Key gönderir.
- **Reverse proxy / DNS:** Özel domain'ler, aynı reverse proxy'ye DNS, TLS sağlama ve host'u aynı web upstream'e eşleme ile devreye alınır—ayrı uygulama dağıtımı değildir.
- **Load balancer:** Model yatay ölçekleme ile uyumludur; kiracı bağlamı çalışma anında host/bootstrap ile çözülür, kiracıya özel build'den değil.
