# CatalogIQ — İlerleme

_Son güncelleme: 2026-06-30 (Adım 70)_

## Özet

| Sprint | Tarih | Konu |
|--------|-------|------|
| Sprint 1-3 | 2026-06-14 | Solution iskelet, Domain, EF, MassTransit, API, Frontend |
| Sprint 4-5 | 2026-06-14 | Normalization, Quality Score, Media, ExtractionRule |
| Sprint 6-7 | 2026-06-15 | Mapping Engine, Integration Engine, Playwright, Excel |
| Sprint 8 | 2026-06-15 | Category normalization, robots.txt, Stats, Dashboard |
| Sprint 9 | 2026-06-15 | Site Intelligence First mimari, DataAccessDecisionEngine |
| Sprint 10 | 2026-06-15 | Token bucket rate limiter, Crawl Safety Service |
| Sprint 11 | 2026-06-15 | Feed ingestion (RSS/Atom/Google Shopping), StartFromPlan |
| Sprint 12 | 2026-06-15 | ExtractionSource enum, Observability API + admin |
| Sprint 13 | 2026-06-16 | ExtractionPriority, CrawlSchedulerService, SystemJob |
| Sprint 14 | 2026-06-19 | Frontend ExtractionPriority entegrasyonu |
| Sprint 15-20 | 2026-06-19-20 | TechnicalAttributes, PriceSparkline, CategoryFetch, UX |
| Sprint 21 | 2026-06-20 | CatalogIQ → Ecom entegrasyon doğrulaması |
| Sprint 22 | 2026-06-20 | Ecom import: çoklu resim, stok, barcode, uzun açıklama |
| Sprint 23 | 2026-06-20 | Toplu entegrasyon, İş istatistikleri |
| Sprint 24 | 2026-06-21 | CategoryPath NULL düzeltmesi, breadcrumb/JSON-LD |
| Sprint 25 | 2026-06-21 | TitleCategoryInferrer (100+ kural, Türkçe oto parça) |
| Sprint 26-29 | 2026-06-21-22 | Gelişmiş filtreler, PATCH endpoint, inline edit, UX |
| Sprint 30-31 | 2026-06-22 | Crawler limitleri kaldırıldı, MaxPaginationPages |
| Sprint 32 | 2026-06-22 | Recrawled tracking, crawl sayacı |
| Sprint 33 | 2026-06-22 | IntervalMinutes, Zamanlamalar düzenle, silentRefresh |
| Sprint 34 | 2026-06-23 | Toplu retry, kapsamlı filtreler, ürün detayı expand |
| Sprint 35 | 2026-06-24 | Akıllı Yeniden Tarama (RecrawlAfter, NightlyRecrawlService) |
| Sprint 36 | 2026-06-25 | Integration pipeline fix: DeadLetter, orphan recovery, EF fix |
| Sprint 37 | 2026-06-26 | Sistem sürekliliği: Windows Services, Watchdog, docker restart |
| Sprint 38 | 2026-06-26 | Fiyat filtresi (≤/≥/=/↔), entegrasyon job reboot kurtarma fix |
| Sprint 39 | 2026-06-27 | Polly 8.4.2 HTTP retry (5x), consumer yeniden kuyruk mantığı kaldırıldı |
| Sprint 40 | 2026-06-29 | Boyner görsel fix: ExtractionEngine 3 bug, fix-missing-images endpoint |
| Sprint 41 | 2026-06-30 | Mükerrer ürün tespiti ve temizleme (içerik bazında duplikat) |
| Sprint 42 | 2026-06-30 | Toplu yeniden başlatma (Crawler İstekler) + DB unique index + race condition fix |

## Güncel Durum

- **Normalize Ürün:** 100.743 (onlineyedekparca.com + boyner.com.tr), avgQuality: 0.95 (DB unique index ile duplikat önlendi)
- **API:** http://localhost:5163 — Windows Service olarak çalışıyor (`dotnet.exe` ile)
- **Workers:** Windows Service olarak çalışıyor (Session 0)
- **Frontend:** http://localhost:3003
- **Docker:** postgres/rabbitmq/redis — `restart: unless-stopped`
- **DB:** `IX_NormalizedProducts_TenantId_SourceSiteId_CanonicalUrl` UNIQUE partial index aktif

## Test Siteleri

| Site | ID | Durum |
|------|-----|-------|
| onlineyedekparca.com | bc87db68-ebd1-4a35-8656-75ba5b422d72 | Aktif |
| ankpar.com | a1b2c3d4-e5f6-7890-abcd-ef1234567890 | Aktif |
