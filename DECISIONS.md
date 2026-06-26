# CatalogIQ — Teknik Kararlar

## Windows Service: PublishSingleFile kaldırıldı

**Karar:** `PublishSingleFile=true` yerine klasör yayımı kullanıldı.

**Neden:** 103MB single-file EXE, Windows Service LocalSystem hesabında ilk çalıştırmada native kütüphaneleri temp dizine çıkarmak için 30+ saniye harcıyor. Bu SCM'nin 30s `ServicesPipeTimeout`'unu aşıyor ve error 1053'e yol açıyor. Klasör yayımında 158KB launcher + ayrı DLL'ler; başlatma ~12s.

---

## PostgreSQL: MD5 Auth

**Karar:** `POSTGRES_HOST_AUTH_METHOD: md5`

**Neden:** Windows üzerinde Docker Desktop'ta PostgreSQL 17 varsayılan `scram-sha-256` auth, Npgsql ile uyumsuzluk yaşıyor. MD5 geçici çözüm.

---

## MassTransit: RequestedConnectionTimeout(8s)

**Karar:** Tüm RabbitMQ host konfigürasyonlarında `h.RequestedConnectionTimeout(TimeSpan.FromSeconds(8))`.

**Neden:** Varsayılan RabbitMQ.Client TCP bağlantı timeout 30 saniye. Windows Service SCM timeout (ServicesPipeTimeout) da 30 saniye. Eğer RabbitMQ başlangıçta erişilemezse 30s blocklanır ve error 1053 alınır. 8s timeout race condition'ı ortadan kaldırır.

---

## Migration: İki Dizin

**Karar:** Eski migration'lar `Migrations/`, yeni migration'lar `Persistence/Migrations/` dizininde.

**Neden:** Sprint orta noktasında dizin taşıması yapıldı. EF Core snapshot çakışmasını önlemek için `ConfigureWarnings(PendingModelChangesWarning.Ignore)` eklendi.

---

## Site Intelligence First

**Karar:** Her crawl öncesinde `POST /api/sources/{id}/inspect` çalıştırılır.

**Neden:** Kör crawl yerine önce site'i analiz et: platform tespiti (WooCommerce/Shopify/...), feed/sitemap/API varlığı, risk sinyalleri. Böylece en verimli veri erişim yöntemi seçilir (API > Feed > Sitemap > JSON-LD > HTML).

---

## RecrawlAfter: Gece 01:00 UTC

**Karar:** Yeni normalize edilen ürünler bugün veya yarın 01:00 UTC'ye kadar "fresh" sayılır.

**Neden:** Gündüz crawl yükünü minimize et. Gece 01:00-06:00 UTC penceresi düşük yük döneminde recrawl yapar. Böylece aynı ürün günde birden fazla kez çıkarılmaz.

---

## Windows Service: Network Trigger

**Karar:** `sc.exe triggerinfo start/networkon stop/networkoff`

**Neden:** Servisler sistem boot'unda değil, ağ bağlantısı kurulduğunda başlar. Docker Desktop ve diğer ağ bağımlı servislerin hazır olmasını bekler. Ağ kesildiğinde de durur (gereksiz yeniden bağlanma döngülerini önler).
