# CatalogIQ — Değişiklik Günlüğü

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
