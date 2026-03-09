# Operasyon Detay (Türkçe)

Operasyon ve production hazırlığı detayı; İngilizce belge ile uyumludur.

---

## Özel Domain Desteği

- **Aynı web uygulaması:** Tek web uygulaması/imaj landing, tenant subdomain'leri ve onaylı özel domain'leri sunar. Kiracı başına ayrı frontend build yoktur.
- **Devreye alma:** Özel domain'ler: DNS aynı reverse proxy/load balancer'a, TLS sağlama, host'u aynı web upstream'e eşleme. Ayrı uygulama dağıtımı değildir.
- **Load balancer:** Kiracı bağlamı çalışma anında host/bootstrap'tan; yatay ölçeklenebilir; sticky session veya kiracıya özel dağıtım gerekmez.
