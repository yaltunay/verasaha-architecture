# Operasyon Özeti (Türkçe)

VeraSaha platformu için üst düzey operasyon özeti; İngilizce özet ile uyumludur.

---

## Özel Domain Desteği

- Aynı web uygulaması verasaha.com, tenant subdomain'leri ve onaylı özel domain'leri sunar; kiracı başına build veya dağıtım yoktur.
- Özel domain'ler DNS → aynı reverse proxy, TLS sağlama ve host'u aynı web upstream'e eşleme ile devreye alınır (ayrı dağıtım değildir).
- Kiracı bağlamı çalışma anında (host/bootstrap) çözülür; model load balancer dostu ve yatay ölçeklenebilirdir.
