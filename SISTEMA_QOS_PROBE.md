# 📡 Sistema de Sondas de Medición QoE/QoS — Documentación Técnica Completa

> **Versión del documento:** 1.0  
> **Fecha:** 2026-06-06  
> **Fuentes:** `github.com/freddyerincon/qos-probe` y `github.com/freddyerincon/qos-api-server` (repos locales en `~/workspace/qos-probe/` y `~/workspace/qos-api-server/`).  
> **Versión del binario en producción:** `1.1.6-20260605-test` (rama test, en uso).  
> **Versión de la API en producción:** commit `b281a77+` (EC2 `54.84.140.23`).

---

## 1. Visión general

El sistema de sondas QoE/QoS es una plataforma distribuida de medición de calidad de servicio y calidad de experiencia para operadores de telecomunicaciones y reguladores. La arquitectura es la siguiente:

```
┌─────────────────┐     HTTPS     ┌──────────────────┐
│  SONDA (Go)     │ ────────────► │  API SERVER (Go) │
│  /usr/local/bin │   Bearer      │  qos.teamco.com.co│
│  qos-probe      │   X-Probe-Key │  ALB AWS + EC2   │
└─────────────────┘               └──────────────────┘
         │                                 │
         │ ciclo local                     │
         ▼                                 ▼
   binarios embebidos                ┌──────────────┐
   • yt-dlp (python)                 │ RDS Postgres │
   • chromium (no)                   │  qosprobe    │
   • NUT (upsc)                      └──────────────┘
   • yt-dlp + Python                        │
                                           ▼
                                    ┌──────────────┐
                                    │ Panel Web    │
                                    │ (Angular)    │
                                    │ Grafana      │
                                    │ Telegram Bot │
                                    └──────────────┘
```

**Componentes:**
- **`qos-probe`** — binario Go embebido en cada sonda. Recolecta mediciones, las reporta al API.
- **`qos-api-server`** — backend HTTP/JSON. Recibe datos, los persiste, evalúa alarmas, expone panel RBAC.
- **RDS PostgreSQL** — base de datos principal (Tabla `measurements`, `keepalives`, `probes`, etc).
- **Cloudflare Workers** — CDN `cdn-test.teamco.com.co` para pruebas de velocidad + `qos-probe-releases` y `qos-test-releases` para auto-update.
- **Telegram Bot** — alarmas en tiempo real + AI chat.
- **Netbird** — VPN para acceso SSH remoto a las sondas.

---

## 2. Anatomía del binario `qos-probe`

### 2.1 Estructura del repositorio

```
qos-probe/
├── cmd/qos-probe/main.go              # 439 líneas — entry point
├── internal/
│   ├── api/                           # cliente HTTP al API server
│   │   ├── client.go                  # 9KB — POST keepalive/measurements/test-reports
│   │   └── tunnel.go
│   ├── cfmtls/
│   ├── config/                        # carga config del servidor
│   │   └── config.go                  # Define YouTubeCfg, YouTubeRealCfg, etc.
│   ├── errors/
│   ├── keepalive/                     # 13KB — loop de keepalive
│   │   └── keepalive.go
│   ├── logger/
│   ├── netbind/                       # SO_BINDTODEVICE por interfaz
│   ├── netcheck/                      # DPI, TLS, DNS hijack detection
│   ├── network/                       # WiFi manager, interface detection
│   ├── probe/                         # info de la sonda
│   │   ├── probe.go                   # GetProbeID, DetectISP, system stats
│   │   ├── isp.go
│   │   ├── location.go                # Apple/Google geoloc BSSID
│   │   ├── dns_watchdog.go
│   │   └── bigdatacloud.go
│   ├── retry/                         # cola de mediciones pendientes
│   ├── runner/                        # 1982 líneas — orquestador del ciclo
│   │   └── runner.go
│   ├── schedule/                      # franjas horarias
│   ├── sensors/                       # BME280, hwmon, UPS
│   │   ├── bme280.go
│   │   ├── ups.go                     # NUT + GPIO fallback
│   │   └── hwmon.go
│   ├── tests/                         # 24 archivos — todas las pruebas
│   │   ├── blocked.go
│   │   ├── dns.go
│   │   ├── hping.go
│   │   ├── http_speed.go              # speed test HTTP multi-hilo
│   │   ├── iperf3.go
│   │   ├── isp_detection.go
│   │   ├── multi_cdn.go               # speed test paralelo a 4 CDNs
│   │   ├── ndt7.go
│   │   ├── ntp.go
│   │   ├── ping.go
│   │   ├── tcp_port.go
│   │   ├── throttling.go
│   │   ├── traceroute.go
│   │   ├── udpping.go
│   │   ├── voip.go
│   │   ├── web.go
│   │   ├── webpage_load.go            # Chromium headless — versión nueva
│   │   ├── wifiscan.go
│   │   ├── wifistats.go
│   │   ├── youtube.go                 # yt-dlp - 14KB
│   │   ├── youtube_real.go            # Playwright Chromium - 9KB
│   │   ├── registry.go                # Registry de pruebas
│   │   └── scripts/youtube_qoe.py     # Python embebido via go:embed
│   ├── tunnel/                        # SSH reverse tunnel
│   ├── updater/
│   │   ├── updater.go                 # 24KB — auto-update desde CF Worker
│   │   └── crashguard.go              # detecta crash y rollback
│   └── watchdog/                      # watchdog interno, reinicio 12h
└── configs/
```

### 2.2 `main.go` — flujo de inicialización (línea por línea)

```
1.  Configura logging con rotation (lumberjack, 7 días, gzip, /var/log/qos-probe/qos-probe.log)
2.  Logger JSON estructurado
3.  tests.InstallYouTubeScript()              → extrae youtube_qoe.py embebido via go:embed
4.  Lee QOS_API_URL, sale si no existe
5.  crashguard: detecta si el run anterior crasheó → rollback automático
6.  Crea contexto cancelable por SIGINT/SIGTERM
7.  probeID = MAC de eth0 sin ':'  (ej: "001e0657089f")
8.  api.NewClient()                            → lee /etc/qos-probe/api_key.conf
9.  probe.DetectISP()                          → ip-api.com + MaxMind ASN DB local
10. cfgMgr.Load()                              → GET /api/v1/probes/{id}/config
11. BigDataCloud init                          → geoloc desagregada (gratis, sin key)
12. Google Geolocation API key desde env
13. BME280 init si environmental_monitor.enabled
14. UPS name desde config
15. retry.NewQueue()                           → cola de mediciones pendientes
16. keepalive.NewService()                     → loop 60s de keepalive
17. tunnel.EnsureKeyPair()                     → Ed25519 para SSH tunnel
18. apiClient.RegisterSSHKey()                 → POST /api/v1/probes/{id}/ssh-key
19. updater.NewUpdater()                       → poll CF Worker para nuevas versiones
20. probe.DetectLocationByBSSID()              → Google Geoloc API (BSSID WiFi)
21. ISP refresh ticker cada 12h
22. watchdog.New(12h)                          → reinicio periódico
23. DNS Watchdog                                → recupera systemd-resolved si se cuelga
24. runner.NewRunner() + r.Run(ctx)            → BUCLE PRINCIPAL
25. crashguard.MarkCleanShutdown()              → SIGTERM no es crash
```

### 2.3 Identidad de la sonda

```go
// internal/probe/probe.go
func GetProbeID() (string, error) {
    iface, _ := net.InterfaceByName("eth0")
    mac := iface.HardwareAddr.String()        // "00:1e:06:57:08:9f"
    id := strings.ReplaceAll(mac, ":", "")    // "001e0657089f"
    return id, nil
}
```

**ID = MAC de `eth0` sin dos puntos.** Esto es lo que aparece como `probe_id` en todas las tablas.

### 2.4 Loop principal (`runner.Run`)

```go
// internal/runner/runner.go
func (r *Runner) Run(ctx context.Context) {
    fullInterval := cfg.MeasurementIntervalSec   // default 30 min
    quickInterval := cfg.QuickCheckIntervalSec   // default 5 min
    
    r.executeCycle(ctx, cfg)                     // ciclo completo al arrancar
    r.processRetryQueue(ctx)
    
    fullTicker := time.NewTicker(fullInterval)
    quickTicker := time.NewTicker(quickInterval)
    
    for {
        select {
        case <-fullTicker.C:                    // cada 30 min
            r.executeCycle(ctx, cfg)            // ciclo completo
            r.processRetryQueue(ctx)            // reintenta envíos fallidos
            r.retryQ.Cleanup()
        case <-quickTicker.C:                   // cada 5 min
            r.executeQuickCheck(ctx, cfg)       // quick check
        case <-ctx.Done():
            return
        }
    }
}
```

### 2.5 Ciclo completo (`executeCycleForInterface`)

**Fase 1 — Pre-validación (60s timeout)**
1. `ConnectivityCheck` (DNS + ping a gateway + ping a Google)
2. `FullNetworkCheck` (DPI, TLS fingerprint, DNS hijack, captive portal, transparent proxy)
3. Lectura de sensores (BME280/SHTC3/cpu-thermal)
4. Lectura de UPS (NUT/USB → GPIO fallback)
5. System health (uptime, CPU, RAM, disco)

**Fase 2 — Pruebas configuradas (orden en el código)**
1. `isp_detection` — siempre la primera, detecta IP pública + ISP + ASN
2. `latency_baseline` — ping a targets configurados (10s)
3. `throughput_single_stream` — NDT7 o HTTP download
4. `throughput_multi_stream` — iperf3 o multi-thread HTTP
5. `dns_performance` — resolución N×M samples
6. `blocked_sites` — chequeo de sitios bloqueados
7. `hping` — TCP ping
8. `udp_ping` — UDP ping
9. `traceroute`
10. `voip` — POLQA MOS
11. `ntp`
12. `http_access` — acceso HTTP
13. `wifi_scan` — ARPScan extendido
14. `wifi_stats` — RSSI, noise, SNR, bitrate
15. `youtube` (legacy, yt-dlp) — descarga de varias calidades
16. `youtube_real` (nuevo, Playwright Chromium) — reproducción real
17. `webpage_load` — carga de páginas con Chromium
18. `http_speed` — speed test HTTP multi-hilo
19. `multi_cdn` — speed test paralelo a 4 CDNs
20. `tcp_port` — probe TCP a 12 puertos

**Fase 3 — Detección de throttling** (si single < 50% de multi → flag)

**Fase 4 — Envío**
- `client.PostMeasurement(ctx, wrapper)` → `POST /api/v1/measurements`
- Si falla → `retryQ.Enqueue()`

### 2.6 Modo realtime vs batch

En `runner.go` línea 1955, `reportTestResult()`:

```go
func (r *Runner) reportTestResult(ctx, cfg, testType, ...) {
    if cfg.ReportingMode != "realtime" {
        return  // ← Modo batch: NO envía el reporte por prueba
    }
    // Modo realtime: envía TestReport al endpoint /api/v1/test-reports
    ...
}
```

**En producción, `cfg.ReportingMode` está en `"realtime"`** (default en `config.go:536`). Así que cada prueba también se envía por separado al endpoint `/api/v1/test-reports`.

### 2.7 Quick Check (`executeQuickCheck`)

Versión light (60s timeout). Solo:
1. Connectivity check (DNS + 2 pings)
2. Latency baseline (10s, primer target)
3. DNS performance (3 samples)
4. System health

`measurement_type = "quick_check"` (vs `"full_cycle"`).

---

## 3. Estructura de los mensajes

### 3.1 Keepalive → `POST /api/v1/keepalive`

Enviado cada 60s. **Múltiples instancias en BD** (no se agregan, son rows individuales).

```json
{
  "probe_id": "001e0657089f",
  "timestamp_utc": "2026-06-07T03:42:11Z",
  "event_type": "keepalive",                              // o "test_start"
  "current_test_type": null,
  "measurement_id": null,
  "public_ip": "181.61.10.50",
  "asn": 27855,
  "isp_name": "Somos Networks Colombia S.a.s. BIC",
  "interface_name": "eth0",                                // o "wlan0"
  "technology": "eth",                                     // o "wifi"
  "active_interfaces": ["eth0", "wlan0"],
  
  "temperatura_c": 36.4,
  "humedad_pct": 26.3,
  "presion_hpa": 0,
  "sensor_name": "cpu-thermal",
  "sensor_source": "thermal_zone",
  
  "ups_status": "OL",
  "ups_type": "gpio-ac-detector",
  "ups_load_pct": 0,
  "ups_battery_charge_pct": 0,
  "ups_battery_voltage_v": 0,
  "ups_battery_voltage_nominal_v": 0,
  "ups_battery_voltage_low_v": 0,
  "ups_battery_voltage_high_v": 0,
  "ups_runtime_remaining_min": 0,
  "ups_input_voltage_v": 0,
  "ups_output_voltage_v": 0,
  "ups_output_voltage_nominal_v": 0,
  "ups_output_frequency_hz": 0,
  "ups_output_frequency_nominal_hz": 0,
  "ups_temperature_c": 0,
  "ups_beeper_status": "",
  "ups_firmware_aux": "",
  
  "system_uptime_sec": 12345,
  "cpu_usage_pct": 5.2,
  "memory_usage_pct": 23.1,
  "disk_usage_pct": 47.8,
  "pending_measurements_count": 0,
  "config_version_loaded": 5,
  "software_version": "1.1.6-20260605-test",
  "location_lat": 4.672740,                               // 6 decimales garantizados
  "location_lon": -74.108365,
  
  "country_name": "Colombia",
  "country_code": "CO",
  "lvl1_name": "Bogotá D.C.",
  "lvl1_code": "DC",
  "lvl2_name": "Bogotá",
  "lvl3_name": "Suba",
  "postcode": "111111"
}
```

### 3.2 Medición → `POST /api/v1/measurements`

Enviado cada 30 min (ciclo completo) o 5 min (quick check).

**Wrapper:**
```json
{
  "measurement": {
    "schema_version": "1.0",
    "probe_id": "001e0657089f",
    "timestamp_utc": "2026-06-07T03:33:18Z",
    "measurement_id": "uuid-único-por-ciclo",
    "cycle_id": "20260607-033318-abc12345",
    "config_version": 5,
    "iface": "wlan0",
    "measurement_type": "full_cycle",                     // o "quick_check"
    
    "connectivity_check": {
      "dns_ok": true,
      "dns_latency_ms": 12.3,
      "gateway_ok": true,
      "gateway_latency_ms": 1.2,
      "google_ok": true,
      "google_latency_ms": 18.7
    },
    
    "latency_baseline": {
      "target_ip": "8.8.8.8",
      "latency_min_ms": 12.1,
      "latency_avg_ms": 14.5,
      "latency_max_ms": 18.2,
      "latency_p50_ms": 14.0,
      "latency_p95_ms": 17.0,
      "latency_p99_ms": 18.0,
      "latency_stddev_ms": 1.2,
      "jitter_ms": 2.1,
      "packet_loss_pct": 0.0,
      "packets_sent": 10,
      "packets_received": 10,
      "packets_timeout": 0,
      "packets_corrupt": 0,
      "packets_duplicate": 0
    },
    
    "throughput_single_stream": { ... },
    "throughput_multi_stream":  { ... },
    "dns_performance":          { ... },
    "blocked_sites":            { ... },
    "environmental_sensors":    { ... },
    "ups_status":               { ... },
    "system_health":            { ... },
    "throttling_detection":     { ... },
    "hping":                    { ... },
    "udp_ping":                 { ... },
    "streaming":                { ... },
    "voip":                     { ... },
    "ntp":                      { ... },
    "wifi_scan":                { ... },
    "wifi_stats":               { ... },
    "traceroute":               { ... },
    "http_access":              { ... },
    "youtube":                  { ... },                  // ← ESTA ES LA QUERY
    "youtube_real":             { ... },
    "http_speed":               { ... },
    "multi_cdn":                { ... },
    "tcp_port":                 { ... },
    "webpage_load":             { ... },
    
    "network_mode": "normal",                             // o "restricted"
    "blocked_ports": [],
    "network_diagnostic": { ... },
    "inactive_interfaces": [],
    
    "exit_code": 0,
    "failure_cause": null,
    "errors": []
  }
}
```

### 3.3 Test Report → `POST /api/v1/test-reports` (modo realtime)

Enviado **al terminar cada prueba** (no al final del ciclo). Esto es lo que tiene la granularidad fina.

```json
{
  "test_report": {
    "schema_version": "1.0",
    "probe_id": "001e0657089f",
    "measurement_id": "uuid-ciclo",
    "cycle_id": "20260607-033318-abc12345",
    "config_version": 5,
    "iface": "wlan0",
    "test_type": "youtube_real",                          // o "isp_detection", "multi_cdn", etc
    "test_status": "completed",                           // o "failed"
    "start_time_utc": "2026-06-07T03:33:00Z",
    "end_time_utc": "2026-06-07T03:33:45Z",
    "duration_ms": 45000,
    "test_data": { ... },                                 // resultado completo
    "error": ""
  }
}
```

### 3.4 Estructura de YouTube (`youtube_real`, Playwright/Chromium)

```json
{
  "timestamp_utc": "2026-06-07T03:33:45Z",
  "url": "https://www.youtube.com/watch?v=jNQXAC9IVRw",
  "requested_duration_sec": 30,
  "success": true,
  "error": "",
  
  "page_load_time_ms": 1250,
  "player_ready_time_ms": 2300,
  "first_frame_time_ms": 3100,
  
  "actual_playback_duration_sec": 28.5,
  "video_duration_sec": 19.0,
  
  "initial_quality": "720p",
  "final_quality": "1080p",
  "available_qualities": ["144p","240p","360p","480p","720p","1080p","1440p","2160p"],
  "quality_changes": [
    { "time_sec": 12.3, "from": "720p", "to": "1080p" }
  ],
  
  "buffering_events": [
    { "time_sec": 8.5, "duration_ms": 1500 }
  ],
  "total_buffering_time_ms": 1500,
  "buffering_count": 1,
  "rebuffering_ratio": 0.0526,
  
  "playback_mos_avg": 4.32,
  "playback_mos_first": 4.0,
  "playback_mos_stable": 4.5,
  
  "stalls_count": 1,
  "playing_time_ms": 28500,
  "stalls_time_ms": 1500,
  "max_stall_time_ms": 1500,
  
  "stalls": [
    { "time_sec": 8.5, "duration_ms": 1500 }
  ]
}
```

---

## 4. Servidor API — `qos-api-server`

### 4.1 Estructura

```
qos-api-server/
├── cmd/api-server/main.go                 # 752 líneas — router principal
├── internal/
│   ├── auth/                              # JWT manager
│   ├── config/                            # carga env vars
│   ├── database/
│   │   ├── db.go
│   │   ├── keepalives.go                  # 7KB — InsertKeepalive
│   │   ├── measurements_fix.go            # 22KB — InsertMeasurement
│   │   ├── measurements_old.go.bak
│   │   ├── probes.go                      # 15KB
│   │   ├── test_reports.go                # 6KB
│   │   ├── connectivity.go
│   │   ├── indicators.go
│   │   └── tunnel.go
│   ├── domain/
│   ├── email/                             # SMTP
│   ├── handlers/                          # 31 archivos
│   │   ├── admin.go
│   │   ├── ai_chat.go                     # 72KB — Gemini + Claude
│   │   ├── ai_telegram.go
│   │   ├── alarm_settings.go
│   │   ├── alarms.go                      # 13KB
│   │   ├── api_v2.go                      # 37KB — endpoints públicos
│   │   ├── auth.go                        # 16KB
│   │   ├── binary_report.go
│   │   ├── comparison.go                  # 13KB
│   │   ├── config.go                      # 9KB — sonda config endpoint
│   │   ├── config_templates.go
│   │   ├── connectivity.go
│   │   ├── default_config.go
│   │   ├── health.go
│   │   ├── indicators.go
│   │   ├── keepalive.go
│   │   ├── measurements.go                # 5KB — sonda endpoint
│   │   ├── notification_contacts.go
│   │   ├── organizations.go
│   │   ├── outage_scraper.go
│   │   ├── panel_measurements.go          # 23KB
│   │   ├── probes_rbac.go
│   │   ├── reports_sla.go
│   │   ├── telegram_links.go
│   │   ├── telegram_webhook.go
│   │   ├── test_reports.go                # 8KB
│   │   ├── tunnel.go                      # 25KB
│   │   ├── update_policy.go
│   │   ├── users.go
│   │   └── wifi_baseline.go
│   ├── middleware/
│   │   ├── auth.go                        # ProbeAuth (X-Probe-Key) + JWT
│   │   ├── gzip.go
│   │   ├── logging.go
│   │   └── ratelimit.go
│   ├── notification/                      # email + telegram + OpenWA
│   ├── repository/                        # acceso a BD
│   │   ├── alarm.go                       # 38KB
│   │   ├── config_template.go
│   │   ├── interfaces.go
│   │   ├── organization.go
│   │   ├── pghelper.go
│   │   ├── probe.go                       # 16KB
│   │   ├── refresh_token.go
│   │   └── user.go
│   ├── service/
│   │   ├── alarm.go                       # evaluador de alarmas
│   │   ├── auth.go
│   │   ├── cross_correlation.go
│   │   ├── offline_monitor.go
│   │   ├── outage_scraper.go
│   │   ├── power_outage_detector.go
│   │   ├── predictive_baseline.go
│   │   ├── score_evaluator.go
│   │   ├── sla.go
│   │   ├── tunnel.go
│   │   ├── version_monitor.go
│   │   ├── weekly_report.go
│   │   └── wifi_fingerprint.go
│   └── ticketing/
├── database/                              # 39 migraciones SQL
│   ├── migrate_*.sql                      # tablas, columnas, vistas
│   └── migrations/
└── docs/
```

### 4.2 Middlewares

| Middleware | Función |
|---|---|
| `probeAuth` | Valida `X-Probe-Key: <api_key>` de la sonda. Lee API key del header, busca probe por `api_key_hash` en BD, inyecta `probe_id` en contexto. |
| `adminAuth` | Valida `X-Admin-Key: <admin_api_key>` (header o body). Para endpoints admin. |
| `jwtAuth` | Valida JWT en `Authorization: Bearer`. Para panel web. |
| `requireSuperAdmin` | Requiere rol `superadmin` en JWT. |
| `requireOrgAdmin` | Requiere rol `org_admin` o `superadmin`. |
| `requireViewer` | Requiere rol `org_viewer`, `org_admin` o `superadmin`. |
| `rateLimiter` | 60 req/min por IP. |
| `logging` | Log estructurado de cada request. |
| `gzip` | Compresión gzip de respuestas. |

### 4.3 Endpoints de sondas (auth: `X-Probe-Key`)

| Método | Ruta | Handler | Descripción |
|---|---|---|---|
| GET  | `/health` | `health` | Health check (sin auth) |
| POST | `/api/v1/keepalive` | `keepalive` | Recibe keepalive cada 60s |
| POST | `/api/v1/measurements` | `measurements` | Recibe ciclo completo o quick check |
| POST | `/api/v1/test-reports` | `test_reports` | Recibe reporte individual de prueba (realtime) |
| GET  | `/api/v1/probes/{id}/config` | `config` | Devuelve config con ETag para 304 |
| POST | `/api/v1/probes/{id}/binary-report` | `binary_report` | Crash/rollback report del updater |
| POST | `/api/v1/probes/{id}/ssh-key` | `tunnel` | Registra clave SSH pública |
| GET  | `/api/v1/probes/{id}/commands` | `tunnel` | Polling de comandos SSH tunnel |
| POST | `/api/v1/probes/{id}/commands/{cmd_id}/ack` | `tunnel` | ACK de comando ejecutado |
| POST | `/api/v1/probes/{id}/update-events` | `tunnel` | Eventos de updater |

### 4.4 Endpoints Admin (auth: `X-Admin-Key` o JWT)

| Método | Ruta | Handler | Auth |
|---|---|---|---|
| GET/POST | `/api/v1/admin/probes` | `admin` | Admin Key |
| GET/POST | `/api/v1/admin/probes/{id}/*` | `admin` | Admin Key |
| GET | `/api/v1/admin/probes/{id}/config` | `adminConfigGet` | JWT org_admin |
| PUT | `/api/v1/admin/probes/{id}/config` | `configUpdate` | JWT org_admin |
| GET | `/api/v1/admin/probes/{id}/update-policy` | `updatePolicy` | JWT org_admin |
| PUT | `/api/v1/admin/probes/{id}/update-policy` | `updatePolicy` | JWT org_admin |
| PUT | `/api/v1/admin/orgs/{org_id}/update-policy` | `bulkUpdatePolicy` | JWT org_admin |

### 4.5 Endpoints Panel (auth: JWT)

#### Auth (público)
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/auth/login` | Login con email+password → JWT access + refresh |
| POST | `/auth/refresh` | Renueva access token |
| POST | `/auth/logout` | Revoca refresh token |
| POST | `/auth/change-password` | Cambia password (JWT) |
| POST | `/auth/switch-org` | Cambia org activa (multi-tenant) |
| GET  | `/me/organizations` | Lista orgs del usuario |

#### Organizaciones
| Método | Ruta | Rol |
|---|---|---|
| GET    | `/orgs` | superadmin |
| POST   | `/orgs` | superadmin |
| GET    | `/orgs/{id}` | org_admin+ |
| PUT    | `/orgs/{id}` | org_admin+ |

#### Usuarios
| Método | Ruta | Rol |
|---|---|---|
| POST   | `/orgs/{org_id}/users` | superadmin |
| GET    | `/orgs/{org_id}/users` | org_admin+ |
| PUT    | `/orgs/{org_id}/users/{user_id}` | superadmin |
| DELETE | `/orgs/{org_id}/users/{user_id}` | superadmin |
| POST   | `/orgs/{org_id}/users/{user_id}/reset-password` | org_admin |
| POST   | `/users/me/telegram-link` | viewer+ |

#### Sondas (panel RBAC)
| Método | Ruta | Rol |
|---|---|---|
| GET    | `/probes` | viewer+ |
| GET    | `/probes/{id}` | viewer+ |
| POST   | `/probes/{id}/orgs` | superadmin |
| DELETE | `/probes/{id}/orgs/{org_id}` | superadmin |
| GET    | `/probes/{id}/orgs` | superadmin |
| PUT    | `/probes/{id}/org` | superadmin |
| POST   | `/probes/{id}/assign` | org_admin |
| DELETE | `/probes/{id}/assign/{user_id}` | org_admin |
| PUT    | `/probes/{id}/name` | org_admin |
| PUT    | `/probes/{id}/active` | org_admin |

#### Alarmas
| Método | Ruta | Rol |
|---|---|---|
| POST/GET/PUT/DELETE | `/orgs/{org_id}/alarm-rules` | org_admin+ |
| GET    | `/orgs/{org_id}/alarm-rules/{id}` | org_admin+ |
| GET    | `/orgs/{org_id}/alarms` | viewer+ |
| GET    | `/orgs/{org_id}/alarms/history` | viewer+ |
| GET    | `/orgs/{org_id}/alarms/stats` | viewer+ |
| GET    | `/probes/{probe_id}/alarms` | viewer+ |
| GET    | `/orgs/{org_id}/alarm-settings` | superadmin |
| PUT    | `/orgs/{org_id}/alarm-settings` | superadmin |

#### Config Templates
| Método | Ruta | Rol |
|---|---|---|
| POST/GET | `/orgs/{org_id}/config-templates` | org_admin |
| GET/PUT/DELETE | `/orgs/{org_id}/config-templates/{id}` | org_admin |
| POST | `/orgs/{org_id}/config-templates/assign` | org_admin |

#### Notificación
| Método | Ruta | Rol |
|---|---|---|
| GET/POST/PUT/DELETE | `/orgs/{org_id}/contacts` | org_admin |
| GET | `/orgs/{org_id}/alarm-rules/{id}/contacts` | viewer+ |
| POST/DELETE | `/orgs/{org_id}/alarm-rules/{id}/contacts/{cid}` | org_admin |

#### SLA y Reportes
| Método | Ruta | Rol |
|---|---|---|
| GET/PUT | `/orgs/{org_id}/report-config` | org_admin |
| POST | `/orgs/{org_id}/report-config/send` | org_admin |
| POST/GET | `/orgs/{org_id}/sla-contracts` | org_admin / viewer+ |
| GET | `/orgs/{org_id}/sla-contracts/{cid}/reports` | viewer+ |
| POST | `/orgs/{org_id}/sla-contracts/{cid}/calculate` | org_admin |

#### Comparativa
| Método | Ruta | Rol |
|---|---|---|
| GET | `/orgs/{org_id}/comparison/probes` | viewer+ |
| GET | `/orgs/{org_id}/comparison/clusters` | viewer+ |
| GET | `/orgs/{org_id}/comparison/clusters/{cluster}` | viewer+ |
| GET | `/orgs/{org_id}/comparison/isps` | viewer+ |

#### Indicadores sintéticos
| Método | Ruta | Rol |
|---|---|---|
| POST/GET | `/orgs/{org_id}/indicators` | org_admin / viewer+ |
| GET | `/orgs/{org_id}/indicators/{id}` | viewer+ |
| GET | `/orgs/{org_id}/indicators/{id}/scores/geo` | viewer+ |
| GET | `/orgs/{org_id}/indicators/{id}/scores/{probe_id}` | viewer+ |
| GET | `/orgs/{org_id}/indicators/{id}/summary` | viewer+ |

#### WiFi fingerprinting
| Método | Ruta | Rol |
|---|---|---|
| GET | `/orgs/{org_id}/probes/{probe_id}/wifi/profile` | viewer+ |
| GET | `/orgs/{org_id}/probes/{probe_id}/wifi/correlation` | viewer+ |
| GET | `/orgs/{org_id}/probes/{probe_id}/wifi/history` | viewer+ |
| GET | `/orgs/{org_id}/probes/{probe_id}/baseline` | viewer+ |
| GET | `/orgs/{org_id}/predictive-alerts` | viewer+ |
| POST | `/orgs/{org_id}/predictive-alerts/{id}/resolve` | org_admin |

#### API v2 (consumo externo de datos, JWT)
| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/v2/probes` | Lista de sondas |
| GET | `/api/v2/probes/{id}` | Detalle de sonda |
| GET | `/api/v2/probes/{id}/measurements` | Mediciones de una sonda |
| GET | `/api/v2/probes/{id}/keepalives` | Keepalives de una sonda |
| GET | `/api/v2/probes/{id}/connectivity` | Conectividad |
| GET | `/api/v2/probes/{id}/alarms` | Alarmas |
| GET | `/api/v2/clusters` | Clusters de sondas |
| GET | `/api/v2/summary` | Resumen general |
| GET | `/api/v2/redundancy` | Redundancia |

#### AI Chat
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/ai/chat` | Chat con AI (Gemini o Claude) |
| GET  | `/api/v1/ai/logs` | Logs de AI chat |
| POST | `/api/v1/ai/feedback` | Feedback de AI |
| POST | `/webhooks/ai-telegram` | Webhook Telegram bot AI |

#### Panel mediciones (widgets personalizados)
| Método | Ruta | Descripción |
|---|---|---|
| GET  | `/panel/measurements` | Mediciones |
| GET  | `/panel/probe-status` | Estado de sondas |
| POST | `/panel/widgets/query` | Query de widget |
| GET/POST | `/panel/widgets` | CRUD widgets |
| GET/PUT/DELETE | `/panel/widgets/{id}` | Widget por ID |

#### Cortes de energía programados
| Método | Ruta | Rol |
|---|---|---|
| GET    | `/admin/planned-outages` | org_admin |
| POST   | `/admin/planned-outages` | superadmin |
| POST   | `/admin/planned-outages/scrape` | superadmin |
| GET    | `/admin/planned-outages/scraper-status` | superadmin |

#### Túneles SSH (operador)
| Método | Ruta | Rol |
|---|---|---|
| POST | `/api/v1/probes/{id}/tunnel/connect` | org_admin |
| POST | `/api/v1/probes/{id}/tunnel/disconnect` | org_admin |
| GET  | `/api/v1/probes/{id}/tunnel/status` | viewer+ |
| GET  | `/api/v1/probes/{id}/connectivity-timeline` | viewer+ |
| POST | `/api/v1/probes/{id}/netbird/{up,down,restart}` | org_admin |
| POST | `/api/v1/probes/{id}/probe/restart` | org_admin |
| POST | `/api/v1/probes/{id}/system/reboot` | org_admin |
| POST | `/api/v1/probes/{id}/tailscale/uninstall` | org_admin |
| POST | `/api/v1/probes/{id}/asn-db/update` | org_admin |
| POST | `/api/v1/probes/{id}/update-binary` | org_admin |

#### Telegram
| Método | Ruta | Rol |
|---|---|---|
| POST | `/webhooks/telegram` | Webhook del bot |
| POST | `/orgs/{org_id}/telegram/link` | Generar link de vinculacion |
| GET  | `/orgs/{org_id}/telegram/subscriptions` | Listar suscripciones |
| DELETE | `/orgs/{org_id}/telegram/subscriptions/{id}` | Eliminar suscripcion |

---

## 5. Tablas de la base de datos (PostgreSQL en RDS)

### 5.1 Tablas de mediciones

#### `measurements` (la principal, ~140 columnas)

```sql
-- Identificación
id                     UUID              PK
probe_id               VARCHAR(12)       NOT NULL       -- "001e0657089f"
measurement_id         UUID              NOT NULL       -- UUID único del ciclo
cycle_id               VARCHAR(100)                     -- "20260607-033318-abc12345"
tag                    VARCHAR(100)                     -- opcional
program                VARCHAR(50)       NOT NULL       -- "ping-test" | "ndt7" | "youtube" | "multi-cdn" | etc
task_name              VARCHAR(100)                     -- "youtube_qoe" | "isp_detection" | "tcp_port" | etc
cluster                VARCHAR(50)                      -- agrupación lógica de sondas
config_version         BIGINT                           -- versión de config que la sonda tenía
software_version       VARCHAR(20)                      -- "1.1.6-20260605-test"

-- Tiempos
date                   TIMESTAMPTZ        NOT NULL
date_end               TIMESTAMPTZ
local_hours            DOUBLE PRECISION                 -- hora local de la sonda
hours                  DOUBLE PRECISION                 -- hora UTC

-- Red
iface_name             VARCHAR(20)                      -- "eth0" | "wlan0"
isp                    VARCHAR(100)
tech                   VARCHAR(20)                      -- "eth" | "wifi" | "4g"
tech_ho                VARCHAR(20)                      -- handover
latitude               DOUBLE PRECISION
longitude              DOUBLE PRECISION
target_ip              INET                             -- IP objetivo de la prueba
ipv4_pub_ip            INET                             -- IP pública de la sonda
ipv6_pub_ip            INET
ipv4_dns1, ipv4_dns2   INET
ipv6_dns1, ipv6_dns2   INET

-- Estado
exit_code              INTEGER                          -- 0 = OK
failure_cause          TEXT
errors                 TEXT[]                           -- array de strings
connectivity           JSONB                            -- {dns_ok, gateway_ok, google_ok, ...}

-- Latencia (ping, hping, udp_ping)
latency_min_ms         DOUBLE PRECISION
latency_avg_ms         DOUBLE PRECISION
latency_max_ms         DOUBLE PRECISION
latency_p50_ms         DOUBLE PRECISION
latency_p95_ms         DOUBLE PRECISION
latency_p99_ms         DOUBLE PRECISION
latency_stddev_ms      DOUBLE PRECISION
jitter_ms              DOUBLE PRECISION
packet_loss_pct        DOUBLE PRECISION
tx_packets, rx_packets INTEGER
timeout_packets        INTEGER
corrupt_packets        INTEGER
duplicate_packets      INTEGER

-- Speed (NDT7 o iperf3)
avg_speed_dl_mbps      DOUBLE PRECISION
avg_speed_ul_mbps      DOUBLE PRECISION
p50_speed_dl_mbps      DOUBLE PRECISION
p90_speed_dl_mbps      DOUBLE PRECISION
p10_speed_dl_mbps      DOUBLE PRECISION
max_speed_dl_mbps      DOUBLE PRECISION
min_speed_dl_mbps      DOUBLE PRECISION
p50_speed_ul_mbps      DOUBLE PRECISION
p90_speed_ul_mbps      DOUBLE PRECISION
max_speed_ul_mbps      DOUBLE PRECISION
min_speed_ul_mbps      DOUBLE PRECISION
time_dl_ms, time_ul_ms DOUBLE PRECISION
bytes_dl, bytes_ul     BIGINT
accomplishment_dl_pct  DOUBLE PRECISION
accomplishment_ul_pct  DOUBLE PRECISION
avg_ux_speed_dl_mbps   DOUBLE PRECISION   -- multi-user experience
avg_ux_speed_ul_mbps   DOUBLE PRECISION
streams_count          INTEGER
retransmit_pct         DOUBLE PRECISION

-- DNS
dns_lookup_ms          DOUBLE PRECISION
dns_server             VARCHAR
dns_resolved_ip        INET
dns_attempts           INTEGER
dns_reattempts         INTEGER
dns_success            BOOLEAN

-- Web (HTTP)
load_time_ms           DOUBLE PRECISION
connection_time_ms     DOUBLE PRECISION
num_resources          INTEGER
failed_requests        INTEGER
data_length_bytes      BIGINT
encoded_data_length_bytes BIGINT
session_duration_ms    DOUBLE PRECISION
completeness_pct       DOUBLE PRECISION
dom_content_event_ms   DOUBLE PRECISION

-- Streaming/Video (YouTube, etc)
load_player_time_ms    DOUBLE PRECISION
play_start_time_ms     DOUBLE PRECISION
buffering_time_ms      DOUBLE PRECISION
stalls_count           INTEGER
stalls_time_ms         DOUBLE PRECISION
max_stall_time_ms      DOUBLE PRECISION
avg_resolution         VARCHAR                          -- "1280x720" etc
quality                VARCHAR                          -- "720p" | "1080p"
qualities_counter      INTEGER
playing_time_ms        DOUBLE PRECISION
playback_mos_avg       DOUBLE PRECISION
playback_mos_first     DOUBLE PRECISION
playback_mos_stable    DOUBLE PRECISION

-- VoIP
call_setup_time_ms     DOUBLE PRECISION
call_time_ms           DOUBLE PRECISION
drop_call              BOOLEAN
polqa_mos_origin       DOUBLE PRECISION
polqa_mos_destination  DOUBLE PRECISION

-- Traceroute
hops_counter           INTEGER
fastest_hop_ms         DOUBLE PRECISION
slowest_hop_ms         DOUBLE PRECISION
traceroute_events      JSONB

-- WiFi scan
wifi_devices_2g        INTEGER
wifi_devices_5g        INTEGER
wifi_devices_6g        INTEGER
wifi_scan_results      JSONB

-- Latencia bajo carga
latency_loaded_avg_ms  DOUBLE PRECISION
latency_baseline_avg_ms DOUBLE PRECISION
latency_degradation_pct DOUBLE PRECISION
buffer_bloat_detected  BOOLEAN

-- Throttling
throttling_ratio       DOUBLE PRECISION
throttling_detected    BOOLEAN

-- Sitios bloqueados
blocked_sites_results  JSONB
blocked_total_checked  INTEGER
blocked_compliant      INTEGER
blocked_violations     INTEGER

-- NTP
ntp_offset_ms          DOUBLE PRECISION
ntp_synced             BOOLEAN

-- Sensores ambientales
temperatura_c          DOUBLE PRECISION
humedad_pct            DOUBLE PRECISION
presion_hpa            DOUBLE PRECISION

-- UPS
ups_status             VARCHAR(10)                      -- "OL" | "OB" | "CHRG"
ups_battery_charge_pct DOUBLE PRECISION
ups_battery_voltage_v  DOUBLE PRECISION
ups_runtime_min        DOUBLE PRECISION
ups_temperature_c      DOUBLE PRECISION
ups_load_pct           DOUBLE PRECISION

-- Sistema
system_uptime_sec      BIGINT
cpu_usage_pct          DOUBLE PRECISION
memory_usage_pct       DOUBLE PRECISION
disk_usage_pct         DOUBLE PRECISION

-- Multi-CDN
waterfall              JSONB                            -- time-series del speed test
raw_result             JSONB                            -- JSON completo recibido
cdn_fastest            TEXT
cdn_slowest            TEXT
cdn_ratio              DOUBLE PRECISION
cdn_discrimination     BOOLEAN
cdns_tested            INTEGER
cdns_successful        INTEGER
cdn_total_throughput_mbps DOUBLE PRECISION
```

**Particionado:** `measurements` está particionada por mes (`measurements_2026_01`, `measurements_2026_02`, ...). Partición por defecto: `measurements_default`. Trigger automático al insertar.

#### `test_reports` (modo realtime, granularidad fina por prueba)

```sql
id                    UUID           PK
probe_id              VARCHAR(12)    NOT NULL
measurement_id        UUID
cycle_id              VARCHAR(100)
config_version        BIGINT
test_type             VARCHAR(50)    -- "isp_detection", "youtube_real", "multi_cdn", etc
test_status           VARCHAR(20)    -- "completed" | "failed"
start_time, end_time  TIMESTAMPTZ
duration_ms           BIGINT
error_message         TEXT
raw_result            JSONB          -- resultado completo de la prueba
```

Particionado por mes.

#### `keepalives` (heartbeat cada 60s)

```sql
id              UUID         PK
probe_id        VARCHAR(12)  NOT NULL
date            TIMESTAMPTZ  NOT NULL
hours           DOUBLE PRECISION
local_hours     DOUBLE PRECISION
event_type      VARCHAR(20)  -- "keepalive" | "test_start" | "test_end"
task_id         UUID
program         VARCHAR(50)  -- nombre de la prueba activa
iface_name      VARCHAR(20)  -- "eth0" | "wlan0"
isp             VARCHAR(100)
provider_id     VARCHAR
tech            VARCHAR(20)
latitude        DOUBLE PRECISION
longitude       DOUBLE PRECISION
public_ip       INET
asn             INTEGER
temperatura_c   DOUBLE PRECISION
humedad_pct     DOUBLE PRECISION
presion_hpa     DOUBLE PRECISION
ups_status      VARCHAR(10)
```

Particionado por mes.

#### `binary_reports` (crashes y rollbacks del updater)
#### `probe_update_events` (eventos de update)

### 5.2 Tablas de sondas

#### `probes` (estado actual de cada sonda)

```sql
id                  UUID                PK
probe_id            VARCHAR(12)         UNIQUE NOT NULL  -- "001e0657089f"
name                VARCHAR(100)                         -- alias humano
is_active           BOOLEAN
config_version      BIGINT
last_seen           TIMESTAMPTZ
api_key_hash        VARCHAR(64)         -- SHA256 de la API key
current_org_id      UUID
lat, lon            DOUBLE PRECISION
isp, asn, public_ip
cluster             VARCHAR(50)
has_ups             BOOLEAN
telegram_chat_id    BIGINT
software_version    VARCHAR(20)
```

#### `probe_configs` (histórico de configs)
#### `probe_assignments` (asignación a usuarios)
#### `probe_org_assignments` (multi-tenancy)
#### `probe_ssh_keys` (claves SSH para tunnel)
#### `probe_baselines` (baseline predictivo)
#### `probe_scores` (score sintético por sonda)

### 5.3 Tablas WiFi

#### `wifi_stats_measurements` (parámetros RF por medición)
#### `wifi_device_history` (histórico de BSSID)
#### `wifi_device_hourly` (agregado por hora)
#### `wifi_scan_hourly`

### 5.4 Tablas de alarmas

#### `alarm_rules` (reglas definidas por org)

```sql
id                  UUID         PK
org_id              UUID
name                VARCHAR
metric              VARCHAR                       -- "latency_avg_ms" | "packet_loss_pct" | etc
comparison_op       VARCHAR                       -- "gt" | "lt" | "between" | "outside"
threshold_value     DOUBLE PRECISION
threshold_value_2   DOUBLE PRECISION              -- para "between"
duration_minutes    INTEGER                       -- cuánto tiempo debe estar en alarma
scope               VARCHAR                       -- "probe" | "cluster" | "org"
scope_id            UUID
min_severity        VARCHAR                       -- "warning" | "critical"
notification_emails TEXT[]
notification_telegram_chat_ids BIGINT[]
show_in_panel       BOOLEAN                       -- visible en panel
```

#### `alarm_states` (estado actual de cada alarma)
#### `alarm_history` (histórico de disparos)
#### `alarm_org_settings` (cuotas por org)
#### `alarm_rule_contacts` (contactos personalizados)

### 5.5 Tablas RBAC

#### `organizations`
#### `users`
#### `user_org_memberships`
#### `refresh_tokens`
#### `panel_widgets` (widgets personalizados del dashboard)

### 5.6 Tablas de reportes

#### `org_config_templates` (templates de config)
#### `org_notification_contacts`
#### `org_report_configs`
#### `sla_contracts`
#### `sla_reports`
#### `report_history`
#### `synthetic_indicators` (KPI sintéticos)
#### `indicator_kpis`

### 5.7 Tablas UPS

#### `ups_events` (eventos de cambio de estado)
#### Vista materializada: `ups_latest` (estado actual, refresh cada 60s)

### 5.8 Tablas de túneles

#### `tunnel_commands` (comandos pendientes para sonda)
#### `tunnel_sessions` (sesiones activas)
#### `outage_scraper_state`

### 5.9 Tablas varias

#### `planned_outages` (cortes programados)
#### `predictive_alerts` (alertas predictivas)
#### `telegram_link_codes`, `telegram_invite_codes`, `telegram_subscriptions`
#### `error_codes` (catálogo)
#### `software_versions` (catálogo de versiones)
#### `ai_chat_logs` (logs del AI)
#### `test_reports_hourly`, `measurements_hourly`, `wifi_device_hourly` (agregados)

### 5.10 Tablas de retry

#### `retry_queue` (mediciones pendientes de reenvío)
#### `measurements_hourly_inc`, `test_reports_hourly_inc` (incrementales)

---

## 6. Diagrama de flujo de datos

```
┌─────────────┐
│   SONDA     │
│  (Go bin)   │
└──────┬──────┘
       │
       │ cada 60s
       ▼
┌──────────────────────────────────┐
│  POST /api/v1/keepalive          │──► INSERT INTO keepalives
└──────────────────────────────────┘
       │
       │ cada 5 min
       ▼
┌──────────────────────────────────┐
│  POST /api/v1/measurements       │──► INSERT INTO measurements
│  (measurement_type=quick_check)  │     (full JSON en raw_result)
└──────────────────────────────────┘
       │
       │ cada 30 min
       ▼
┌──────────────────────────────────┐
│  POST /api/v1/measurements       │──► INSERT INTO measurements
│  (measurement_type=full_cycle)   │     + post-hook: AlarmEvaluator
└──────────────────────────────────┘
       │
       │ durante el ciclo (realtime)
       ▼
┌──────────────────────────────────┐
│  POST /api/v1/test-reports       │──► INSERT INTO test_reports
│  (1 por prueba completada)       │     + post-hook: WiFiFingerprint
└──────────────────────────────────┘     + ISP change detector
       │
       │ cada vez que arranca
       ▼
┌──────────────────────────────────┐
│  GET /api/v1/probes/{id}/config  │──► Effective config (template o legacy)
└──────────────────────────────────┘
       │
       │ (background goroutines)
       ▼
┌──────────────────────────────────┐
│  Auto-updater                    │──► GET qos-probe-releases CF Worker
│  - check cada 1h                 │
│  - descarga binario si nueva     │──► POST /api/v1/probes/{id}/binary-report
│  - swap + restart                │     (success/failure)
│  - crashguard en cada arranque   │
└──────────────────────────────────┘
```

---

## 7. Ciclo de update de binario (auto-update)

1. Sonda inicia → `crashguard.CheckAndMark()` verifica si el run anterior crasheó.
   - Si sí: descarga la versión anterior, rollback, exit 1.
   - Si no: continúa.

2. `cfgMgr.Load()` → `GET /api/v1/probes/{id}/config` con `If-None-Match: <etag>`.
   - 200 OK + nuevo config → actualiza local.
   - 304 Not Modified → mantiene config actual.

3. Cada 1h, `updater.Start(ctx)` chequea nueva versión:
   - `GET https://releases.teamco.com.co/latest-stable` (CF Worker)
   - Si `remoteVersion != currentVersion` Y `remoteVersion > currentVersion` (semver):
     - Descarga `https://releases.teamco.com.co/qos-probe-arm64-VERSION`
     - Verifica SHA256
     - `cp /usr/local/bin/qos-probe /usr/local/bin/qos-probe.backup`
     - `mv /tmp/qos-probe-new /usr/local/bin/qos-probe`
     - `systemctl restart qos-probe`

4. Próximo arranque → `crashguard.CheckAndMark()`.
   - Si arranca OK y corre 5 min → `crashguard.MarkHealthy()` → borra backup.
   - Si crashea → `crashguard.CheckAndMark()` detecta marker → rollback a backup.

**Bug conocido:** El updater NO compara semver, solo verifica `remoteVersion != currentVersion`. Causa downgrades si se sube una versión inferior. Pendiente fix.

---

## 8. Bug identificado: `task_name` vacío en mediciones

**Síntoma observado en producción (sonda `001e0657089f`):**
- En las últimas 24h, 126,940 mediciones en `measurements` con `program='ping-test'`.
- Solo 52 mediciones con `program='measurement'` (genérico).
- CERO con `program='youtube'`, `'multi-cdn'`, `'ndt7'`, etc.
- La columna `task_name` está siempre `NULL`.

**Causa raíz:**

El handler `MeasurementHandler.ServeHTTP` en `internal/handlers/measurements.go` y `InsertMeasurement` en `internal/database/measurements_fix.go` calculan `program` a partir de las **claves presentes en el JSON** mediante `detectProgram(raw)`:

```go
func detectProgram(raw json.RawMessage) string {
    mapping := []struct{ key, prog string }{
        {"latency_baseline", "ping-test"},        // ← primero
        {"throughput_single_stream", "ndt7"},
        ...
        {"youtube_real", "youtube"},
        ...
    }
    for _, m := range mapping {
        if _, ok := keys[m.key]; ok {
            return m.prog   // ← retorna el PRIMER match
        }
    }
    return "measurement"
}
```

**Como el payload de un ciclo completo contiene TODAS las claves juntas** (`latency_baseline` + `throughput_single_stream` + `dns_performance` + `youtube_real` + `multi_cdn` + ...), el primer match es siempre `latency_baseline` → `ping-test`. Por eso **todas las mediciones full_cycle aparecen como `program='ping-test'`** y `task_name` NULL.

**Fix propuesto (Opción A — recomendada):**

Insertar **una fila por cada prueba presente** en el JSON, no una sola fila. Cada fila tendrá `program` y `task_name` correctos:

```go
// Pseudocódigo
for key, prog := range programMapping {
    if _, ok := keys[key]; ok {
        // Insertar nueva fila con program=prog, task_name=key
        // (la fila completa se duplica, pero el `task_name` distingue)
    }
}
```

**Fix propuesto (Opción B — más simple):**

Mantener una sola fila por ciclo, pero agregar un campo `task_names` (array) que liste todas las pruebas que estaban en el JSON. El frontend puede entonces filtrar.

**O confirmar si el modo `realtime` está activo** — si lo está, los detalles por prueba **sí** están en la tabla `test_reports`. Verifiqué que en las últimas 24h `test_reports` tiene datos. Entonces el dashboard debería consultar `test_reports` para ver YouTube, no `measurements`.

---

## 9. Configuración de la sonda

La sonda recibe config del servidor (`GET /api/v1/probes/{id}/config`):

```json
{
  "version": 5,
  "schema_version": "1.1",
  "measurement_interval_sec": 1800,           // 30 min
  "quick_check_interval_sec": 300,            // 5 min
  "delay_between_tests_sec": 10,
  "reporting_mode": "realtime",              // "realtime" | "batch"
  
  "auto_update_enabled": true,
  "allow_test_binary": false,
  
  "expected_download_mbps": 50,
  "expected_upload_mbps": 10,
  
  "connectivity_check": {
    "enabled": true,
    "targets": ["192.168.1.1", "8.8.8.8"],
    "timeout_sec": 5
  },
  
  "latency_baseline": {
    "enabled": true,
    "targets": ["8.8.8.8", "1.1.1.1"],
    "duration_sec": 60,
    "packet_size_bytes": 56,
    "interval_ms": 1000,
    "schedule": null
  },
  
  "throughput_single_stream": { ... },
  "throughput_multi_stream": { ... },
  "dns_performance": { ... },
  "blocked_sites": { ... },
  "hping": { ... },
  "udp_ping": { ... },
  "traceroute": { ... },
  "voip": { ... },
  "ntp": { ... },
  "wifi_scan": { ... },
  "wifi_stats": { ... },
  "http_access": { ... },
  
  "youtube": {
    "enabled": true,
    "video_url": "",
    "video_ids": ["jNQXAC9IVRw"],
    "test_qualities": ["360", "720", "1080"],
    "download_size_mb": 2.0,
    "timeout_sec": 120,
    "schedule": null
  },
  
  "youtube_real": {
    "enabled": true,
    "video_url": "",
    "urls": [],
    "duration_sec": 30,
    "timeout_sec": 120,
    "schedule": null
  },
  
  "http_speed": { ... },
  "multi_cdn": {
    "enabled": true,
    "targets": [
      { "name": "Google",   "url": "https://...google...", "enabled": true, "probe_token": "" },
      { "name": "Cloudflare", "url": "https://...cloudflare...", "enabled": true },
      { "name": "OVH",      "url": "https://...ovh...", "enabled": true },
      { "name": "Hetzner",  "url": "https://...hetzner...", "enabled": true },
      { "name": "Datapacket", "url": "https://...datapacket...", "enabled": true },
      { "name": "Teamco",   "url": "https://cdn-test.teamco.com.co/...", "enabled": true, "probe_token": "..." }
    ],
    "duration_sec": 15,
    "timeout_sec": 30,
    "streams": 4,
    "pings_per_cdn": 3,
    "ping_timeout_sec": 5,
    "max_cdns": 5
  },
  
  "tcp_port": {
    "enabled": true,
    "target": "8.8.8.8",
    "ports": [
      { "port": 80, "protocol": "tcp", "description": "HTTP", "enabled": true },
      { "port": 443, "protocol": "tcp", "description": "HTTPS", "enabled": true },
      ...
    ],
    "attempts": 3,
    "timeout_sec": 5
  },
  
  "webpage_load": {
    "enabled": false,
    "urls": [],
    "timeout_sec": 60,
    "chromium_path": "/usr/bin/chromium",
    "user_agent": "Mozilla/5.0 ...",
    "wait_after_load_ms": 3000
  },
  
  "environmental_monitor": {
    "enabled": false,
    "i2c_bus": 1,
    "i2c_address": "0x76"
  },
  
  "ups_monitoring": {
    "enabled": true,
    "ups_name": "cdp@localhost"
  },
  
  "wifi": {
    "ssid": "",
    "password": "",
    "retry_enabled": true,
    "retry_minutes": 5
  }
}
```

---

## 10. Variables de entorno de la sonda

| Variable | Default | Descripción |
|---|---|---|
| `QOS_API_URL` | (requerido) | URL del API server |
| `QOS_API_KEY` | (en /etc/qos-probe/api_key.conf) | API key de la sonda |
| `QOS_S3_URL` | `https://releases.teamco.com.co` | URL del CF Worker para releases stable |
| `QOS_S3_TEST_URL` | `https://test-releases.teamco.com.co` | URL del CF Worker para releases test |
| `GOOGLE_GEO_API_KEY` | (vacío) | API key de Google Geolocation para BSSID |
| `CF_PROBE_TOKEN` | (vacío) | Token HMAC para `cdn-test.teamco.com.co` |
| `WATCHDOG_RESTART_HOURS` | 12 | Período de reinicio automático |
| `NOTIFY_SOCKET` | (vacío) | Socket de systemd para watchdog sd_notify |

## 11. Variables de entorno del API server

| Variable | Default | Descripción |
|---|---|---|
| `PORT` | 8080 | Puerto HTTP |
| `DATABASE_URL` | (requerido) | PostgreSQL connection string |
| `JWT_SECRET` | (requerido) | Secret para firmar JWT |
| `ADMIN_API_KEY` | (requerido) | API key para endpoints admin |
| `RATE_LIMIT_PER_MINUTE` | 60 | Rate limit por IP |
| `ALARM_DEFAULT_QUOTA_DAILY` | 100 | |
| `ALARM_DEFAULT_QUOTA_WEEKLY` | 500 | |
| `ALARM_DEFAULT_QUOTA_MONTHLY` | 2000 | |
| `ALARM_DEFAULT_MAX_RECIPIENTS` | 5 | |
| `ALARM_DEFAULT_MAX_RULES` | 20 | |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM` | (vacío) | Para emails |
| `TELEGRAM_BOT_TOKEN` | (vacío) | Bot de alarmas |
| `TELEGRAM_BOT_USERNAME` | (vacío) | Username del bot |
| `TELEGRAM_AI_BOT_TOKEN` | (vacío) | Bot de AI chat |
| `TELEGRAM_WEBHOOK_PATH` | `/webhooks/telegram` | Path del webhook |
| `TELEGRAM_ADMIN_CHAT_IDS` | (vacío) | Chat IDs admin (separados por coma) |
| `OPENWA_BASE_URL`, `OPENWA_API_KEY`, `OPENWA_SESSION_ID` | (vacío) | WhatsApp self-hosted |
| `GEMINI_API_KEY` | (vacío) | Google Gemini para AI |
| `ANTHROPIC_API_KEY` | (vacío) | Anthropic Claude para AI |
| `PANEL_URL` | (vacío) | URL del panel (para links en notificaciones) |
| `TERMINAL_SERVER_URL` | (vacío) | URL del terminal server para SSH tunnels |
| `TERMINAL_SERVER_API_KEY` | (vacío) | API key del terminal server |
| `WSTUNNEL_URL` | (vacío) | WebSocket tunnel server (alternativa a autossh) |

---

## 12. Servicios de fondo en el API server

| Servicio | Intervalo | Función |
|---|---|---|
| `Tunnel Monitor` | 60s | Monitorea túneles SSH activos, limpia sesiones stale >3min |
| `Offline Monitor` | 2min | Detecta sondas que dejaron de reportar >5min |
| `Aggregate Evaluator` | 60s | Evalúa reglas de alarma con `aggregate_fn` (no por medición) |
| `Score Evaluator` | 60min | Calcula scores sintéticos por sonda |
| `Report Service` | semanal | Genera y envía reportes semanales por email/Telegram |
| `Cross Correlation` | variable | Análisis de causa raíz cruzado |
| `Baseline Service` | variable | Refresca baselines predictivos |
| `Power Outage Detector` | 2min | Detecta cortes eléctricos correlacionando UPS de cluster |
| `Version Monitor` | variable | Detecta sondas con versiones desactualizadas |
| `Outage Scraper` | 24h | Scrapea web de operadores para cortes programados |
| `UPS Latest Refresh` | 60s | `REFRESH MATERIALIZED VIEW CONCURRENTLY ups_latest` |

---

## 13. Conclusión y bug activo

El sistema tiene una arquitectura sólida: el binario Go está bien estructurado, el API server tiene RBAC + multi-tenancy, hay 60+ tablas en BD, monitoreo UPS, túneles SSH, AI chat, predictive alerts, etc.

**Bug activo a resolver (de la auditoría 2026-06-06):**

En producción, las pruebas individuales (YouTube, multi-CDN, NDT7, etc) **no se ven reflejadas en la tabla `measurements`** porque el `detectProgram()` del backend solo asigna UN programa por medición, y siempre retorna el primero del orden de prioridad (`latency_baseline` → `ping-test`).

La información granular **sí está disponible en `test_reports`** (modo realtime activo), pero el panel web probablemente está consultando `measurements` y por eso no muestra esas pruebas.

**Opciones para resolver:**
1. **Cambiar el dashboard** para leer de `test_reports` en lugar de `measurements` (más rápido, sin deploy de backend).
2. **Insertar N filas** en `measurements` por cada prueba (cambio de schema, deploy API server).
3. **Agregar columna `task_names TEXT[]`** a `measurements` y popularla con todas las pruebas presentes (deploy API server con migración).

La opción 1 es la más rápida. La opción 2 es la más limpia arquitectónicamente.
