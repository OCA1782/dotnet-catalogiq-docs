# CatalogIQ — Değişiklik Günlüğü

## [Sprint 44] — 2026-07-01

### Eklendi
- **`StalledDiscoveryRetryService`** (Sprint 43): Her 5 dk'da Queued (mesaj kaybolmuş) ve Running-takılı DiscoveryJob'ları tespit eder; `DiscoverUrlCommand` yeniden yayınlar; `AttemptCount >= 3` olanları otomatik Failed'a çeker
- **`DiscoverUrlConsumer` idempotency guard**: Completed/Cancelled/Failed statüsündeki job için gelen duplicate mesaj sessizce atlanır
- **`POST /api/source-requests/recover-stalled`**: Admin endpoint — 10 dk'dan uzun Queued kalan job'ları kurtarır (manuel tetikleme)
- **Workers rolling dosya loglama**: `Serilog.Sinks.File` sink eklendi → `logs/workers-{tarih}.log` (14 gün tutulur)

### Değiştirildi
- **`CatalogIQ.Workers` Windows Service**: `DEMAND_START` → `DELAYED AUTO_START`; `FAILURE_ACTIONS_ON_NONCRASH_FAILURES=TRUE` (exit code 0'da da recovery tetiklenir)
- **`CatalogIQ.Api` Windows Service**: `DEMAND_START` → `DELAYED AUTO_START`; `FAILURE_ACTIONS_ON_NONCRASH_FAILURES=TRUE`
- **`scripts/watchdog.ps1`**: RabbitMQ yönetim API kontrolü eklendi (`http://localhost:15672/api/connections`) — 0 bağlantı tespitinde `docker restart catalogiq-rabbitmq`; Docker NAT kırılması artık otomatik çözülür
- **Task Scheduler watchdog görevi**: `SYSTEM` → `fpen` Interactive (Docker erişimi için); `StopAtDurationEnd=false`; 3 tetikleyici (5dk/boot/login); `DisallowStartIfOnBatteries=false`

### Kök Neden (Düzeltilen)
- Docker Desktop NAT bug (Windows): TCP handshake tamamlanıyor ama AMQP handshake başlamıyor → `SocketException 10053` — container `healthy` görünse de AMQP bağlantısı sıfıra düşüyor. Önceki watchdog yalnızca container health kontrolü yapıyordu, bağlantı sayısı kontrolü yoktu.
- Watchdog SYSTEM olarak çalışıyordu → Docker pipe erişimi yok → görev sessizce başarısız oluyordu, log yazılmıyordu.

---

## [Sprint 42] — 2026-06-30

### Eklendi
- **`POST /api/source-requests/bulk-restart`**: Seçili Failed/Completed/Cancelled statüsündeki istekleri toplu yeniden başlatır; her istek için yeni DiscoveryJob oluşturulur ve DiscoverUrlCommand yayınlanır; `{ restarted, skipped }` dönüşü
- **Frontend `api.ts`**: `sourceRequests.bulkRestart(ids)` metodu eklendi
- **Crawler İstekler ekranı**: `handleBulkRestart()` fonksiyonu ve "Seçilenleri Yeniden Başlat" butonu (yeşil, RotateCcw ikonu) bulk action bar'a eklendi
- **`NormalizedProductConfiguration.cs`**: `(TenantId, SourceSiteId, CanonicalUrl)` unique partial index — `WHERE CanonicalUrl IS NOT NULL AND IsDeleted = false`
- **EF Migration `20260630171712_AddNormalizedProductCanonicalUrlUniqueIndex`**: index oluşturuldu ve uygulandı

### Değiştirildi
- **`NormalizeProductConsumer.cs`**: Mevcut kayıt sorgusu `.IgnoreQueryFilters()` ile güncellendi — soft-deleted ürünler de CanonicalUrl bazında bulunur ve güncellenir
- **`NormalizeProductConsumer.cs`**: `SaveChangesAsync()` try/catch ile sarıldı; `DbUpdateException` + PostgreSQL `23505` (unique_violation) yakalanıyor — ChangeTracker temizlenir, winner bulunur, tüm alanlar uygulanır, `IsDeleted = false` set edilir (race condition kurtarma)

---

## [Sprint 41] — 2026-06-30

### Eklendi
- **`GET /api/normalized-products/quality-summary`**: `contentDuplicateCount` ve `contentDuplicateExtraCount` alanları — aynı site içinde başlık+açıklama+fiyat+stok birebir aynı olan grup sayısı ve ekstra kayıt sayısı
- **`GET /api/normalized-products`**: `contentDuplicatesOnly=true` query parametresi — içerik duplikatı olan ürünleri filtreler (raw SQL, EXISTS subquery)
- **`POST /api/normalized-products/remove-content-duplicates`**: PostgreSQL `DISTINCT ON` ile her gruptan en yüksek kalite skoru korunur, diğerleri soft-delete edilir
- **Normalize Ürünler — "Mükerrer" kart**: Quality özet kartlarına 5. kart olarak eklendi; duplikat grup sayısını ve ekstra kayıt sayısını gösterir; tıklandığında filtre uygulanır
- **Normalize Ürünler — "Mükerrer" chip**: Eksik Alan bölümüne filtre chip eklendi
- **Normalize Ürünler — "Mükerrerleri Temizle" butonu**: Onay modalı ile duplikatları temizler

### Operasyonel
- 72 duplikat kayıt (`onlineyedekparca.com` — farklı URL varyantları: `/urun/xx` vs `/urun/xx-1`) temizlendi
- Normalize ürün sayısı: 101.815 → 100.743

---

## [Sprint 40] — 2026-06-29

### Düzeltildi (3 kritik bug)
- **`ExtractImagesFromJsonLd()`**: `OfType<JsonValue>()` → `.Select()` ile hem string hem `ImageObject` (`{url/contentUrl:...}`) handle edildi — Boyner gibi JSON-LD'de `[{@type:ImageObject}]` kullanan siteler artık görsel topluyor
- **`GenericRules`/`ShopifyRules`**: `"image"` (tekil) → `"images"` (çoğul) rename; `ApplyRules()` uyumlu hale getirildi, seçiciler çoğaltıldı
- **Image HTML fallback**: JSON-LD/rules başarılı ama images boş ise `CollectImages()` (og:image + HTML selectors) devreye giriyor
- **CS8625 nullable uyarıları**: `GetMeta()`, `GetMicrodata()` null-safe; 13 → 7 warning

### Eklendi
- **`POST /api/source-sites/{id}/fix-missing-images`**: ImageUrls boş ürün tespiti, RecrawlAfter sıfırlama (freshness bypass), kuyruğa alma; `{found, queued, message}` dönüşü
- **`LazyLoadAttrs`**: `data-image-src`, `data-zoom-src`, `data-large`, `data-hi-res`, `data-srcset`
- **`CollectImages()` XPath**: `picture//img`, gallery/slider/swiper/carousel, product-image/product-photo/product-media div'leri
- **`UrlClassifier`**: `SitemapPattern` subdomain desteği (`//sitemap.`); `ProductSuffixPattern` `-p-12345` (Boyner/Trendyol URL formatı)

### Operasyonel
- Boyner site: `ProfileType` WooCommerce → Generic; `ExtractionPriority` HttpFirst → PlaywrightFirst
- 918 Boyner ürünü `fix-missing-images` ile yeniden extraction kuyruğuna alındı
- `CatalogIQ.Api` Windows Service `ImagePath` güncellendi: `dotnet.exe CatalogIQ.Api.dll` (framework aphost sorunu kalıcı çözüldü)

---

## [Sprint 39] — 2026-06-27

### Eklendi
- `Polly 8.4.2` — `CatalogIQ.Integration.csproj`'a bağımlılık olarak eklendi
- `IntegrationEngine._httpRetryPipeline`: `ResiliencePipeline<HttpResponseMessage>` — 5 retry, üstel backoff (2s→4s→8s→16s→32s + jitter), yalnızca `HttpRequestException` + HTTP 5xx

### Değiştirildi
- `IntegrationEngine.SendRestApiAsync`: HTTP çağrısı `_httpRetryPipeline.ExecuteAsync` içine alındı
- `IntegrationEngine.SendWebhookAsync`: Her retry denemesinde `HttpRequestMessage` yeniden oluşturulur; pipeline içine alındı
- `IntegrateProductConsumer`: Transient hata yeniden kuyruk mantığı (`Queued`/`DeadLetter` geçişi) kaldırıldı; artık her başarısız job `Failed` olarak işaretleniyor (aktif kuyruktan çıkar)

---

## [Sprint 38] — 2026-06-26

### Eklendi
- `GET /api/normalized-products`: `priceMin` (≥) ve `priceMax` (≤) query parametreleri
- `api.ts`: `normalizedProducts.list()` → `priceMin` / `priceMax` parametreleri
- Normalize Ürünler sayfası — Fiyat Filtresi UI: mod seçici (≤ En Fazla / ≥ En Az / = Tam Fiyat / ↔ Aralık), aralık modunda iki input

### Düzeltildi
- `FailedIntegrationRetryService`: `staleRetried` sorgusu — reboot sonrası kaybolan RabbitMQ retry mesajları artık kurtarılıyor
- `IntegrateProductConsumer`: Transient retry yolunda `StartedAt = null` set edildi

---

## [Sprint 37] — 2026-06-26

### Eklendi
- `CatalogIQ.Api` Windows Service (`UseWindowsService()` + `Microsoft.Extensions.Hosting.WindowsServices`)
- `docker-compose.yml`: tüm servislere `restart: unless-stopped`
- `scripts/watchdog.ps1`: Docker/Services/Frontend sağlık kontrolü + otomatik yeniden başlatma
- Task Scheduler: `\CatalogIQ\CatalogIQ Watchdog` (her 5 dk, SYSTEM)
- Task Scheduler: `\CatalogIQ\CatalogIQ Frontend` (kullanıcı girişinde)
- `ServicesPipeTimeout` registry → 180000ms (reboot gerekli)

### Değiştirildi
- `InfrastructureServiceExtensions.cs`: MassTransit `RequestedConnectionTimeout(8s)` — SCM timeout race condition önlendi
- `CatalogIQ.Workers` publish: `PublishSingleFile=true` kaldırıldı → 103MB → 158KB launcher; Windows Service ilk başlatma ~12s
- `CatalogIQ.Workers` Windows Service: `start/networkon` trigger, failure recovery, env=Development

---

## [Sprint 36] — 2026-06-25

### Düzeltildi
- `IntegrateProductConsumer`: `AttemptCount >= MaxAttempts` durumunda `JobStatus.DeadLetter` geçişi eklendi
- `InfrastructureServiceExtensions`: `ConfigureWarnings(PendingModelChangesWarning.Ignore)` her iki `AddDbContext` çağrısına eklendi
- `EF Migration 20260625173305_AddRecrawlAfterAndMaxPagination`: idempotent `DO $$ IF NOT EXISTS` SQL

### Eklendi
- `FailedIntegrationRetryService`: 15+ dk bekleyen orphan IntegrationJob'ları kurtarır

---

## [Sprint 35] — 2026-06-24

### Eklendi
- `NormalizedProduct.RecrawlAfter` (DateTime?) — "fresh" ürün işaretleme
- `NightlyRecrawlService` — gece 01:00-06:00 UTC yeniden tarama
- Migration `20260624000001_AddRecrawlAfter`
- `FailedExtractionRetryService` step-0a/0b — RecrawlAfter aware

### Değiştirildi
- `ExtractProductConsumer`: `RecrawlAfter > UtcNow` → extraction atlanır
- `DiscoverUrlConsumer`: `RecrawlAfter > UtcNow` → ProductUrl oluşturulmaz
- `NormalizeProductConsumer`: normalizasyonda `RecrawlAfter` set edilir
- `DiscoverUrlConsumer finalization bug`: `createdUrlCount = result.ProductUrls.Count - skippedKnown`

---

## [Sprint 34] — 2026-06-23

### Eklendi
- `POST /api/integration-jobs/bulk-retry`
- `ListIntegrationJobs`: `search`, `dateFrom`, `dateTo`, `httpStatus`, `sortBy` parametreleri
- Crawler İstekler: `POST /api/source-requests/bulk-cancel` + `bulk-delete`

---

## [Sprint 33] — 2026-06-22

### Değiştirildi
- `CrawlSchedule.IntervalHours` → `IntervalMinutes` (varsayılan 1440)
- Migration `20260622000002_AddIntervalMinutes`

---

## [Sprint 31] — 2026-06-22

### Değiştirildi
- `MaxUrlsPerRequest = 3000` kısıtı kaldırıldı
- `MaxPaginationPages` 50 → 500 (varsayılan)
- Sitemap index `.Take(10)` → `.Take(100)`
- `SourceUrlRequest.MaxPaginationPages` yeni alan
- Migration `20260622000001_AddMaxPaginationPages`
