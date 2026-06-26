# CatalogIQ — Mimari

## Genel Bakış

CatalogIQ, URL tabanlı ürün verilerini keşfeden, çıkaran, normalize eden ve hedef sistemlere entegre eden bir veri entegrasyon platformudur.

## Veri Akışı

```
SourceSite (config)
    │
    ▼
[SiteIntelligenceService] — platform tespiti, feed/sitemap/API keşfi
    │
    ▼
[DataAccessDecisionEngine] — öncelik zinciri
    API → Feed → Sitemap → JSON-LD → StaticHTML → JsRender
    │
    ▼
DiscoverUrlConsumer ← DiscoverUrlCommand (MassTransit)
    │  (sitemap/feed/HTML'den ProductUrl'ler çıkarır)
    │  RecrawlAfter > now → skip (fresh products)
    ▼
ExtractProductConsumer ← ExtractProductCommand
    │  (HTTP veya Playwright ile HTML fetch)
    │  RecrawlAfter > now → skip
    ▼
NormalizeProductConsumer ← NormalizeProductCommand
    │  (fiyat, slug, stok, kategori, kalite skoru)
    │  RecrawlAfter = NextRecrawlAt() set edilir
    ▼
ProcessMediaConsumer ← ProcessMediaCommand
    │  (resim URL deduplikasyon)
    ▼
IntegrateProductConsumer ← IntegrateProductCommand
    │  (REST/webhook/Excel export)
    ▼
ExternalSystem (Ecom, ERP, vb.)
```

## Background Services (Workers)

| Servis | Açıklama | Periyot |
|--------|----------|---------|
| `RobotsTxtRefreshService` | Stale robots.txt dosyalarını yeniler | 24 saat |
| `CrawlSchedulerService` | Zamanlanmış crawl'ları tetikler | 1 dakika polling |
| `FailedExtractionRetryService` | Başarısız/stuck extraction'ları yeniden kuyruğa alır | 30 saniye |
| `DataQualityCheckService` | Eksik alan olan ürünleri yeniden kuyruğa alır | 2 dakika |
| `NightlyRecrawlService` | Gece 01:00-06:00 UTC, RecrawlAfter <= now ürünleri yeniden tarar | Gece penceresi |
| `FailedIntegrationRetryService` | 15+ dk bekleyen IntegrationJob'ları kurtarır | 10 dakika |

## Akıllı Yeniden Tarama (Sprint 35)

- `NormalizedProduct.RecrawlAfter`: DateTime? — ürünün "fresh" olduğu süre sonu
- `NormalizeProductConsumer`: her normalizasyonda `RecrawlAfter = bugün/yarın 01:00 UTC` set edilir
- `ExtractProductConsumer`: `RecrawlAfter > UtcNow` ise extraction atlanır (~50x hız)
- `DiscoverUrlConsumer`: `RecrawlAfter > UtcNow` olan ürünler için yeni ProductUrl oluşturulmaz
- `NightlyRecrawlService`: 01:00-06:00 UTC arası `RecrawlAfter <= now` ürünleri kuyruğa alır

## Sistem Sürekliliği (Sprint 37)

### Windows Services
```
CatalogIQ.Api     — ASP.NET Core Web API (http://localhost:5163)
CatalogIQ.Workers — Worker Service (MassTransit consumers)
```

Her iki servis:
- `start/networkon` trigger: ağ bağlantısı kurulduğunda otomatik başlar
- `stop/networkoff`: ağ kesildiğinde durur
- Failure recovery: restart/5s, restart/15s, restart/60s
- `ASPNETCORE_ENVIRONMENT=Development`

### Docker Container Restart Policies
```yaml
restart: unless-stopped  # postgres, rabbitmq, redis
```

### Task Scheduler Görevleri
| Görev | Tetikleyici | Çalıştıran |
|-------|-------------|------------|
| CatalogIQ Watchdog | Her 5 dk + boot | SYSTEM |
| CatalogIQ Frontend | Kullanıcı girişi | Mevcut kullanıcı |

### Watchdog (`scripts/watchdog.ps1`)
1. Docker container'ları kontrol et → gerekirse `docker compose up -d`
2. RabbitMQ sağlıklı değilse bu turu atla
3. Workers servisi çalışmıyorsa başlat
4. Api servisi çalışmıyorsa başlat + health endpoint doğrula
5. Frontend port 3003'te dinlemiyorsa `npm run dev` başlat
6. `logs/watchdog.log` dosyasına kaydet

## Entity'ler (Ana Tablolar)

| Entity | Açıklama |
|--------|----------|
| `SourceSite` | Taranan web sitesi konfigürasyonu |
| `SourceUrlRequest` | Crawl isteği (Discovery job container) |
| `ProductUrl` | Keşfedilen ürün URL'si |
| `ExtractionResult` | Ham çıkarım verisi |
| `NormalizedProduct` | Normalize edilmiş ürün (CanonicalUrl+SourceSiteId unique) |
| `ProductChangeLog` | Ürün alan değişikliği geçmişi |
| `IntegrationTarget` | Hedef sistem konfigürasyonu (REST/webhook/Excel) |
| `FieldMapping` | Kaynak→hedef alan eşleştirmesi |
| `IntegrationJob` | Entegrasyon iş kaydı (Pending→Queued→Running→Completed/Failed/DeadLetter) |
| `ExtractionRule` | Site'e özgü XPath/CSS seçici kuralları |
| `CrawlSchedule` | Periyodik crawl zamanlaması (IntervalMinutes, varsayılan 1440) |
| `SystemJob` | Background job durumu takibi |
| `ComplianceProfile` | robots.txt + opt-out konfigürasyonu |
| `SiteIntelligenceProfile` | Platform tespiti, feed/sitemap bayrakları, risk skoru |
| `DataAccessCandidate` | Keşfedilen veri erişim yöntemleri |

## SiteProfileType Enum

| Değer | Tip |
|-------|-----|
| 0 | Generic |
| 1 | WooCommerce |
| 2 | Magento |
| 3 | Custom |
| 4 | Shopify |
| 5 | PrestaShop |
| 6 | BigCommerce |
| 7 | OpenCart |

## Migration Dizinleri

- `src/CatalogIQ.Infrastructure/Migrations/` — eski (InitialCreate..AddSystemJobs)
- `src/CatalogIQ.Infrastructure/Persistence/Migrations/` — yeni (Sprint 31+)
