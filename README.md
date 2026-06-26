> **ÖNEMLİ NOT:** GİT'TE HİÇ BİR ŞEKİLDE CLAUDE İLE İLGİLİ BİLGİ-DETAY-LOG VS İSTEMİYORUM. DİKKAT ET.

# CatalogIQ

URL tabanlı ürün veri çıkarımı, normalizasyon ve entegrasyon platformu. Backend **.NET 10 Worker Service + Web API**, frontend **Next.js 16** admin paneli.

_Son güncelleme: 2026-06-26_

---

## Tech Stack

| Katman | Teknoloji |
|--------|-----------|
| Backend API | .NET 10, ASP.NET Core Web API |
| Worker | .NET 10 Worker Service (MassTransit consumers) |
| ORM | Entity Framework Core 10 + PostgreSQL 17 |
| Mesaj kuyruğu | MassTransit 8 + RabbitMQ |
| Önbellekleme | Redis 7 (StackExchange.Redis) |
| HTML Parse | HtmlAgilityPack |
| JS Render | Microsoft.Playwright |
| Excel | ClosedXML |
| API Docs | Scalar |
| Admin UI | Next.js 16, TypeScript, Tailwind CSS |
| Veritabanı | PostgreSQL 17 (Docker, port 5436) |
| Auth (MD5) | md5 — Windows Docker SCRAM uyumsuzluğu geçici çözümü |

---

## Proje Yapısı (12 Proje)

```
CatalogIQ.sln
├── src/
│   ├── CatalogIQ.Api           — ASP.NET Core Web API (Scalar, /health, CORS)
│   ├── CatalogIQ.Workers       — Worker Service (MassTransit consumers, background services)
│   ├── CatalogIQ.Domain        — Entity'ler, enum'lar, value object'ler
│   ├── CatalogIQ.SharedKernel  — Paylaşılan tip/enum'lar (ExtractionSource, SiteProfileType...)
│   ├── CatalogIQ.Infrastructure — EF Core, AppDbContext, MassTransit config, Redis
│   ├── CatalogIQ.Compliance    — RobotsTxt parse, compliance gate
│   ├── CatalogIQ.Discovery     — Sitemap/feed/HTML URL keşfi, rate limiter
│   ├── CatalogIQ.Extraction    — HTML/JSON-LD/OG/microdata extraction, Playwright
│   ├── CatalogIQ.Normalization — Fiyat/slug/stok/kategori normalizasyon, kalite skoru
│   ├── CatalogIQ.Media         — Resim URL toplama ve tekrar kaldırma
│   ├── CatalogIQ.Mapping       — Field mapping engine (dönüşüm ifadeleri)
│   └── CatalogIQ.Integration   — REST/webhook/Excel entegrasyon motoru
└── admin/                      — Next.js 16 admin paneli (http://localhost:3003)
```

---

## Servisler & Port'lar

| Servis | URL/Port |
|--------|----------|
| CatalogIQ API | http://localhost:5163 |
| API Docs (Scalar) | http://localhost:5163/scalar |
| Admin Panel | http://localhost:3003 |
| PostgreSQL | localhost:5436 |
| RabbitMQ | localhost:5672 (mgmt: 15672) |
| Redis | localhost:6379 |

---

## Başlatma

### Otomatik (Windows Service + Task Scheduler)
Sistem ağ bağlantısı kurduğunda **CatalogIQ.Api** ve **CatalogIQ.Workers** servisleri otomatik başlar.
Her 5 dakikada **CatalogIQ Watchdog** task'ı çalışarak tüm bileşenleri kontrol eder.

### Manuel
```cmd
docker compose up -d          # Altyapı (Postgres, RabbitMQ, Redis)
sc.exe start "CatalogIQ.Api"  # API servisi
sc.exe start "CatalogIQ.Workers" # Workers servisi
cd admin && npm run dev       # Frontend
```

---

## Mimari İlke

**"Site Intelligence First"** — Önce site incele (`/api/sources/{id}/inspect`) → data-access-map oluştur → sonra crawl başlat.
Feed/sitemap'ten yapısal veri varsa HTTP extraction atlanır, direkt ExtractionResult oluşturulur.

---

## CatalogIQ → Ecom Entegrasyon

- **IntegrationTarget ID:** cb5ccc45-6766-4bac-96c3-80653036b945 ("Ecom Urunler")
- **EndpointUrl:** http://localhost:5124/api/products/import
- **ApiKey:** catalogiq-dev-key-2026
- **8 FieldMapping:** title→name, sku→sku, priceCurrent→price, brand→brandName, categoryPath→categoryName, firstImageUrl→imageUrl, shortDescription→shortDescription, currency→currency
