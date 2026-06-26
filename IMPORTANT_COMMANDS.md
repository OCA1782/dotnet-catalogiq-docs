# CatalogIQ — Önemli Komutlar

## Docker

```bash
# Altyapıyı başlat
docker compose up -d

# Durum kontrolü
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Loglar
docker logs catalogiq-rabbitmq
docker logs catalogiq-postgres
```

## Windows Services

```powershell
# Durum sorgula
sc.exe query "CatalogIQ.Workers"
sc.exe query "CatalogIQ.Api"

# Başlat / Durdur
sc.exe start "CatalogIQ.Workers"
sc.exe stop "CatalogIQ.Workers"
sc.exe start "CatalogIQ.Api"
sc.exe stop "CatalogIQ.Api"

# Tüm sistem durumu
sc.exe query "CatalogIQ.Workers" | Select-String "STATE"
sc.exe query "CatalogIQ.Api" | Select-String "STATE"
```

## Build & Publish

```bash
# Workers publish (Windows Service binary)
dotnet publish src/CatalogIQ.Workers -c Release --runtime win-x64 --self-contained true -o publish/Workers

# API publish (Windows Service binary)
dotnet publish src/CatalogIQ.Api -c Release --runtime win-x64 --self-contained true -o publish/Api

# Tüm çözüm build
dotnet build
```

## EF Core Migrations

```bash
# Yeni migration ekle (Persistence dizinine)
dotnet ef migrations add <MigrationName> \
  --project src/CatalogIQ.Infrastructure \
  --startup-project src/CatalogIQ.Api \
  --output-dir Persistence/Migrations

# Migration uygula
dotnet ef database update \
  --project src/CatalogIQ.Infrastructure \
  --startup-project src/CatalogIQ.Api
```

## Watchdog

```powershell
# Manuel çalıştır
powershell.exe -ExecutionPolicy Bypass -File "scripts\watchdog.ps1"

# Log izle
Get-Content "logs\watchdog.log" -Tail 20 -Wait
```

## Task Scheduler

```powershell
# CatalogIQ görevlerini listele
Get-ScheduledTask -TaskPath "\CatalogIQ\" | Select-Object TaskName, State

# Watchdog'u hemen çalıştır
Start-ScheduledTask -TaskPath "\CatalogIQ\" -TaskName "CatalogIQ Watchdog"
```

## Veritabanı

```bash
# PostgreSQL bağlan
docker exec -it catalogiq-postgres psql -U catalogiq -d catalogiq

# Normalize ürün sayısı
docker exec -it catalogiq-postgres psql -U catalogiq -d catalogiq \
  -c "SELECT COUNT(*) FROM \"NormalizedProducts\" WHERE \"IsDeleted\"=false;"
```

## API Health Check

```bash
curl http://localhost:5163/health
```

## Frontend

```bash
cd admin
npm run dev    # http://localhost:3003
npm run build  # Production build
```
