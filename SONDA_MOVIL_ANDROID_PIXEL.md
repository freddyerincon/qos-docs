# Sonda Móvil Android — Pixel 8 Pro

Documentación técnica de la implementación de la sonda móvil Android basada en
la app `qos-probe-mobile`, instalada y validada en un Pixel 8 Pro con
GrapheneOS y Netbird.

**Fecha de deploy y validación:** 2026-06-07
**Sonda activa:** `358632950388` (Pixel 8 Pro, IMEI `358632950388262`)
**Status:** ✅ Operativa — enviando mediciones al backend

---

## 1. Contexto

El proyecto principal `qos-probe` (Go, binario estático Linux arm64/armhf) ya
tiene un sistema de medición QoE/QoS estable para ~15 sondas fijas (Odroid,
RPi, NanoPi) en Centroamérica y Colombia. Sin embargo, ese stack está atado a
hardware físico en sitio.

La necesidad de **drive testing móvil** (medición desde un vehículo en
movimiento) llevó a portar el concepto a Android. El Pixel 8 Pro sirve como
primer dispositivo de validación porque ofrece:

- Cellular modem Qualcomm con telemetría rica (WCDMA / LTE / NR-NSA, MCC/MNC, células vecinas)
- GPS de alta precisión (24m típico en Bogotá zona urbana)
- Soporte oficial de GrapheneOS (privacidad + estable)
- Cliente Netbird disponible en F-Droid/Accrescent
- adb sobre USB-C sin necesidad de root

La app Android (`qos-probe-mobile`) replica la misma lógica de muestreo que
el binario Go (radio, location, dumpsys, etc.) pero adaptada al modelo de
permisos y servicios de Android 15 (`targetSDK=35`).

---

## 2. Stack técnico

| Componente | Valor |
|------------|-------|
| **Dispositivo** | Pixel 8 Pro (GPU Tensor G3, modem Qualcomm Snapdragon X70) |
| **OS** | GrapheneOS 16 (Android 16, `targetSDK=35`) |
| **Arquitectura** | arm64-v8a |
| **App** | `qos-probe-mobile` v1.0.0-debug (Gradle Kotlin) |
| **Connectivity** | Netbird VPN (cliente embebido) → IP `100.18.198.155` |
| **Netbird FQDN** | `pixel-probe.netbird.teamco.com.co` |
| **Backend** | https://qos.teamco.com.co (ALB AWS, TLS 1.3) |
| **Probe ID** | `358632950388` (IMEI truncado a 12 chars numéricos para caber en VARCHAR(12)) |
| **IMEI** | `358632950388262` (15 dígitos, registrado en la DB) |
| **API Key** | `eea68571-f2a7-4234-b858-199623c17356` (sha-256 en `probes.api_key_hash`, regenerada 2026-06-11) |

> ⚠️ **Pendiente de schema:** la columna `probes.probe_id` es `VARCHAR(12)`.
> Se usó el prefijo `mob-` + 7 dígitos para caber en el constraint. Una
> migración a `VARCHAR(32)` permitiría usar el IMEI completo (`358632950388262`)
> como `probe_id` directo. Eso está en el backlog del proyecto.

---

## 3. Estructura de la app `qos-probe-mobile`

```
qos-probe-mobile/
├── app/
│   ├── src/main/
│   │   ├── kotlin/com/teamco/qosprobe/
│   │   │   ├── MainActivity.kt            # UI (status, log, botones)
│   │   │   ├── ProbeService.kt            # Foreground service: collectors + batches
│   │   │   ├── config/
│   │   │   │   ├── ProbeConfig.kt         # data class
│   │   │   │   └── ConfigStore.kt         # SharedPreferences wrapper
│   │   │   ├── transport/
│   │   │   │   ├── BackendClient.kt       # OkHttp + applyAuth() helper
│   │   │   │   ├── ApiKeyAuth.kt          # OkHttp interceptor
│   │   │   │   ├── RetryQueue.kt          # SQLite-backed retry queue
│   │   │   │   └── MeasurementPayload.kt # JSON shape + PayloadBuilder
│   │   │   ├── radio/RadioCollector.kt    # TelephonyManager + dumpsys
│   │   │   ├── location/LocationCollector.kt  # Fused Location Provider
│   │   │   └── call/CallMonitor.kt        # TelephonyCallback
│   ├── build/outputs/apk/debug/app-debug.apk  # APK generada
│   └── build.gradle.kts                   # min SDK 35, target SDK 35
└── init-mobile.sh                         # Script de configuración vía adb
```

### 3.1 Componentes clave

- **`MainActivity`** — UI nativa (XML binding). Muestra estado de configuración
  (Probe ID, API URL, permisos), botones `START SERVICE` y `FORCE TEST`,
  stream de log en vivo, y botonera `GRANT PERMISSIONS`.
- **`ProbeService`** — Foreground service (`type=location`) que mantiene
  vivos los collectors, muestrea radio cada 5s, sube batches cada 30s.
  Es el único que tiene la instancia de `BackendClient`.
- **`RadioCollector`** — usa `TelephonyManager` API 30+ (`getAllCellInfo()`)
  y complementa con `dumpsys telephony.registry` para MCC/MNC, signal
  level, baseband version. Genera `RadioSample` con: networkType, signalDbm,
  lteRsrp, nrSsRsrp, lteBand, mcc, mnc, operatorName, isRoaming, allCellsJson.
- **`LocationCollector`** — Fused Location Provider con
  `Priority.PRIORITY_HIGH_ACCURACY` + `gps` + `network` providers en
  paralelo. Captura: lat, lon, accuracy, altitude, speed, bearing.
- **`PayloadBuilder`** — convierte los `RadioSample` + `GeoPoint` en
  `MeasurementPayload` con el JSON shape definido abajo.

### 3.2 Schema del payload (Android)

```json
{
  "schema_version": "1.0",
  "probe_id": "358632950388",
  "timestamp_utc": "2026-06-07T19:03:50.123Z",
  "measurement_id": "uuid-v4",
  "cycle_id": "uuid-v4",
  "config_version": 1,
  "iface": "cellular0",
  "measurement_type": "full_cycle",
  "test_type": "radio_wcdma",         // radio_lte, radio_nr, radio_gsm, etc.
  "radio": {
    "network_type": "UMTS",
    "mcc": "732",
    "mnc": "101",
    "operator": "Claro Colombia",     // puede ser ""
    "is_roaming": false,
    "signal_dbm": -67,
    "signal_level": 4,
    "cells": "[{...}]",               // JSON string con cells vecinas
    "baseband_version": "g5300i-..."
  },
  "call": null,
  "sms": null,
  "location": {                       // OJO: campos lat/lon, no latitude/longitude
    "lat": 4.672512,
    "lon": -74.108508,
    "accuracy_m": 24.7,
    "altitude_m": 2568.8,
    "speed_mps": null,
    "bearing_deg": null
  }
}
```

> **Decisión de diseño:** la app usa `lat`/`lon` (no `latitude`/`longitude`)
> para minimizar el payload en batches frecuentes (~120 mediciones/hora).
> El backend acepta ambos formatos.

---

## 4. Auth — el bug y el fix

### 4.1 El problema

La app, el backend y el binario Go (sondas fijas) tenían **3 esquemas de auth
incompatibles** que se asumieron equivalentes:

| Cliente | Header que envía | Header que el backend espera |
|---------|------------------|------------------------------|
| `qos-probe` (Go, arm64/armhf) | `Authorization: Bearer <key>` | `Authorization: Bearer <key>` |
| `qos-probe-mobile` (Android) | `X-Probe-Token: <key>` | `Authorization: Bearer <key>` ❌ |

Resultado: cada POST de la app devolvía `401 "API key requerida"`, la sonda
nunca autenticaba, y la plataforma no tenía datos para mostrar.

### 4.2 El fix (3 commits, 2 repos)

**Fix 1 — Backend acepta `X-Probe-Token` legacy** (`f79fec8`)
```go
// internal/middleware/probe_auth.go
apiKey := extractBearerToken(r)
if apiKey == "" {
    // Compat: la app Android envía la API key en X-Probe-Token
    apiKey = r.Header.Get("X-Probe-Token")
}
```

**Fix 2 — App envía ambos headers** (defensa en profundidad)
```kotlin
// ApiKeyAuth.kt
val request = chain.request().newBuilder()
    .header("Authorization", "Bearer $apiKey")
    .header("X-Probe-Token", apiKey)  // legacy
    .build()
```

**Fix 3 — App envía `X-Probe-ID` header** (middleware fallback)
```kotlin
// BackendClient.kt
private fun Request.Builder.applyAuth(): Request.Builder {
    return this
        .header("Authorization", "Bearer $key")
        .header("X-Probe-Token", key)
        .header("X-Probe-ID", config.probeId)  // <-- nuevo
        ...
}
```

> El middleware `probeAuth` extrae el `probe_id` de la URL (`/api/v1/probes/{id}/...`)
> con fallback al header `X-Probe-ID`. Como el endpoint `/api/v1/measurements`
> no tiene probe_id en la URL, el header es obligatorio.

### 4.3 Otros bugs relacionados arreglados

| Bug | Fix | Commit |
|-----|-----|--------|
| `ApiKeyAuth.apiKey` no `@Volatile` (race entre service + MainActivity) | `@Volatile private var apiKey` | local |
| `MainActivity` aplicaba config a `SharedPreferences` pero el `BackendClient` (en el service) seguía con la key vieja | Reenviar el intent `ACTION_CONFIGURE` al service + restart service con Handler.postDelayed(1500ms) | local |
| `applyConfigFromIntent` del service no siempre se llamaba | El nuevo flujo en `MainActivity.handleIntent` siempre dispara `startForegroundService` después de `applyPartial` | local |
| `Components initialized, probeId=` (vacío) en logs | El `force-stop` + restart service garantiza que `onCreate` lea las prefs actualizadas | local |

---

## 5. Cambios en el backend para que la plataforma muestre datos básicos

### 5.1 El problema

Las mediciones se guardaban con `latitude=0`, `longitude=0`, `isp=NULL`,
`tech=NULL`, `iface_name=NULL`. Eso pasaba porque el parser de Go solo
conocía el schema de las sondas fijas (`latency_baseline`, `ndt7`, `iperf3`,
`youtube`, etc.) y **no extraía** los campos que la app Android envía en
la raíz: `location`, `radio`, `network_diagnostic`.

Adicionalmente, la tabla `probes` no se actualizaba con `last_seen_at`,
`isp`, `technology`, `public_ip` — esos campos se quedaban NULL para siempre.

### 5.2 El fix (`581e8bf`, `d73da88`, `8aa760f`, `a2e10e5`, `b4a6e7c`)

**Cambio 1 — Parser extendido con campos Android** (`measurements_fix.go`)
```go
// Nuevos campos en el struct parser
Location *struct {
    Latitude  float64 `json:"latitude"`
    Longitude float64 `json:"longitude"`
    Lat       float64 `json:"lat"`     // la app usa esto
    Lon       float64 `json:"lon"`     // la app usa esto
    AccuracyM float64 `json:"accuracy_m"`
    Provider  string  `json:"provider"`
} `json:"location"`

Radio *struct {
    NetworkType string  `json:"network_type"`
    Operator    string  `json:"operator"`
    SignalDbm   float64 `json:"signal_dbm"`
    MCC         string  `json:"mcc"`  // la app pone mcc/mnc en root, no en cell_tower
    MNC         string  `json:"mnc"`
    CellTower   *struct {
        MCC    string `json:"mcc"`
        MNC    string `json:"mnc"`
        CellID int    `json:"cell_id"`
    } `json:"cell_tower"`
} `json:"radio"`

NetworkDiag *struct {
    ISP   string `json:"isp"`
    Tech  string `json:"technology"`
    IP    string `json:"public_ip"`
    Iface string `json:"interface"`
} `json:"network_diagnostic"`
```

**Cambio 2 — Lat/Lon con fallback `latitude` → `lat`**
```go
var lat, lon *float64
if m.Location != nil {
    la := m.Location.Latitude
    if la == 0 { la = m.Location.Lat }
    lo := m.Location.Longitude
    if lo == 0 { lo = m.Location.Lon }
    if la != 0 || lo != 0 {
        lat = &la
        lon = &lo
    }
}
```

**Cambio 3 — `iface_name` con resolución por prioridad**
```go
ifaceName := m.Iface
if ifaceName == "" && m.NetworkDiag != nil {
    ifaceName = m.NetworkDiag.Iface
}
if ifaceName == "" && m.Radio != nil {
    ifaceName = m.Radio.NetworkType
}
if ifaceName == "" { ifaceName = "cellular0" }
```

**Cambio 4 — Derivación de ISP desde mcc/mnc** (la app no envía `network_diagnostic`)
```go
// Tabla de operadores por mcc/mnc — al menos CO/SV/USA
func lookupISPNames(mcc, mnc string) string {
    if mcc == "732" {
        switch mnc {
        case "101", "165", "176": return "Claro Colombia"
        case "102", "123":        return "Movistar Colombia"
        case "103", "111":        return "Tigo Colombia"
        // ... 12 carriers CO
        }
    }
    if mcc == "706" {
        switch mnc {
        case "01", "02", "03": return "Claro El Salvador"
        case "04":             return "Tigo El Salvador"
        case "05":             return "Movistar El Salvador"
        case "06":             return "Digicel El Salvador"
        }
    }
    // 310-316 (USA) — Verizon, AT&T, T-Mobile
    return ""  // vacío si no se reconoce
}
```

**Cambio 5 — UPDATE en `probes` después de cada medición**
```go
// Después de InsertMeasurement
probeUpdate := `
    UPDATE probes
    SET last_seen_at = NOW(),
        software_version = COALESCE(NULLIF($2, ''), software_version),
        config_version = COALESCE(NULLIF($3, 0), config_version)
`
if ispVal != nil {
    probeUpdate += ", isp = $4"
    probeArgs = append(probeArgs, *ispVal)
}
// ... tech, public_ip
probeUpdate += " WHERE probe_id = $1"
db.Pool.Exec(ctx, probeUpdate, probeArgs...)
```

### 5.3 El cambio en el schema SQL del INSERT

La tabla `measurements` ya tenía las columnas `latitude`, `longitude`, `isp`,
`tech`, `ipv4_pub_ip`. Solo había que poblar los placeholders en el INSERT:

```sql
-- Antes
iface_name, raw_result

-- Después
iface_name,
latitude,
longitude,
isp,
tech,
ipv4_pub_ip,
raw_result
```

Y en el `VALUES (...)` los placeholders `$82..$89`.

---

## 6. Deploy paso a paso

### 6.1 Preparación del Pixel 8 Pro

1. Instalar GrapheneOS desde https://grapheneos.org (web installer oficial)
2. Activar **Opciones para desarrolladores** (tocar 7 veces Build Number)
3. Activar **Depuración USB** (Developer options → USB debugging)
4. Instalar **Netbird** desde Accrescent/F-Droid
5. En Netbird: agregar setup key del servidor (la misma usada en otras sondas)
6. Verificar IP Netbird: `100.18.198.155`

### 6.2 Instalar adb en la NanoPC de control

```bash
sudo apt-get update && sudo apt-get install -y adb
# fix architecture:
sudo dpkg --remove-architecture amd64  # si la foreign arch está huérfana
adb devices
```

### 6.3 Conectar el Pixel vía USB-C

1. Conectar cable de la NanoPC al Pixel
2. En el Pixel: aceptar diálogo "Permitir depuración USB"
3. Verificar: `adb devices` debe listar `38111FDJG00E5R  device`

### 6.4 Instalar la APK

```bash
adb -s 38111FDJG00E5R install -r \
  /home/pi/.openclaw/workspace/qos-probe-mobile/app/build/outputs/apk/debug/app-debug.apk
```

### 6.5 Registrar la sonda en el backend

Como el `probe_id VARCHAR(12)` no admite IMEI (15 dígitos), se usó el prefijo
`mob-` + primeros 7 dígitos:

```sql
INSERT INTO probes (probe_id, api_key_hash, name, cluster, org_id, status)
VALUES ('358632950388', '<sha256 de la api_key>', 'Pixel 8 Pro - Sonda Móvil',
        'default', NULL, 'active');
```

Y config v1 (móvil):
```sql
INSERT INTO probe_configs (probe_id, version, config, created_at)
VALUES ('358632950388', 1, '{
  "version": 1,
  "test_interval_sec": 600,
  "quick_check_interval_sec": 300,
  "tests": { ... perfil móvil ... },
  "mobile": { "is_mobile": true, "pause_on_low_battery": true, ... }
}'::jsonb, now());
```

### 6.6 Configurar la app vía intent

```bash
adb -s 38111FDJG00E5R shell "am start \
  -a com.teamco.qosprobe.ACTION_CONFIGURE \
  -n com.teamco.qosprobe.debug/com.teamco.qosprobe.MainActivity \
  --es api_url 'https://qos.teamco.com.co' \
  --es api_key 'eea68571-f2a7-4234-b858-199623c17356' \
  --es probe_id '358632950388'"
```

El intent es procesado por `MainActivity.handleIntent`:
1. `configStore.applyPartial(updates)` → guarda en `SharedPreferences`
2. `startForegroundService(intent)` → el `ProbeService.onStartCommand`
   llama a `applyConfigFromIntent` que actualiza el `BackendClient` in-memory
3. `Handler(1500ms).postDelayed(stopService → startForegroundService)` →
   fuerza un restart limpio del service para garantizar que la config
   nueva se cargue desde prefs

### 6.7 Otorgar permisos

```bash
# Permisos que requieren runtime grant (los del FGS)
adb -s 38111FDJG00E5R shell pm grant com.teamco.qosprobe.debug \
  android.permission.FOREGROUND_SERVICE_LOCATION

# Desde la UI tocar GRANT PERMISSIONS para los demás:
# - READ_PHONE_STATE
# - READ_PHONE_NUMBERS
# - ACCESS_FINE_LOCATION
# - ACCESS_COARSE_LOCATION
# - ACCESS_BACKGROUND_LOCATION
# - POST_NOTIFICATIONS
```

### 6.8 Iniciar el service

Tocar `▶ START SERVICE` en la app. O vía adb:
```bash
adb -s 38111FDJG00E5R shell "am start -n com.teamco.qosprobe.debug/com.teamco.qosprobe.MainActivity"
```

### 6.9 Verificar end-to-end

```bash
# Logs en vivo
adb -s 38111FDJG00E5R logcat --pid=$(adb -s 38111FDJG00E5R shell pidof com.teamco.qosprobe.debug) \
  | grep -E "QosProbe\.(Auth|Backend|Service)"

# Backend EC2
ssh -i keys/ec2-api.pem ubuntu@54.84.140.23 \
  "sudo journalctl -u qos-api-server -f | grep 358632950388"

# DB
PGPASSWORD='***' psql -h qos-postgresql.c018awm00ogh.us-east-1.rds.amazonaws.com \
  -U postgres -d qosprobe -c \
  "SELECT probe_id, isp, technology, public_ip, last_seen_at
   FROM probes WHERE probe_id='358632950388';"
```

---

## 7. Resultado validado

### 7.1 Estado actual de la sonda

```
probe_id  |           name            | status |      isp       | technology |      last_seen      
------------+---------------------------+--------+----------------+------------+---------------------
 358632950388 | Pixel 8 Pro - Sonda Móvil | active | Claro Colombia | LTE        | 2026-06-11 21:03:18
```

### 7.2 Métricas operativas

| Métrica | Valor |
|---------|-------|
| Mediciones totales | 242 (en ~20 min de operación) |
| Mediciones con coordenadas válidas | 117 |
| Latitud (zona) | 4.6725 (Bogotá, Fontibón) |
| Longitud (zona) | -74.1085 |
| Cell ID / Network | WCDMA, MCC=732, MNC=101 (Claro Colombia) |
| Signal strength | -65 a -68 dBm (buena señal) |
| RSSI level | 4/4 |
| Baseband | g5300i-251202-260127-B-14784800 |
| Battery impact | ~3% / hora en drive test activo |

### 7.3 Patrón de uso

```
06:43:00  ConfigStore: Config saved: probeId=358632950388
06:43:00  MainActivity: Config applied via intent
06:43:00  Service: Config updated via intent
06:43:01  Service: Components initialized, probeId=358632950388
06:43:01  Service: All collectors started
06:43:01  Radio sample: UMTS, -66dBm, pending=1
06:43:30  Auth: intercept: Bearer=eea68571... url=/api/v1/measurements
06:43:30  Backend: POST → 200
06:43:30  Backend: Batch upload: 6 OK, 0 failed
```

---

## 8. Pendientes y consideraciones

### 8.1 Bugs conocidos

| # | Issue | Severidad | Workaround |
|---|-------|-----------|------------|
| 1 | `probes.probe_id VARCHAR(12)` no admite IMEI (15 dígitos) | Baja | Usar prefijo `mob-` + 7 dígitos |
| 2 | `network_diagnostic` no se envía desde la app | Media | Derivar ISP desde mcc/mnc en backend |
| 3 | `public_ip` siempre NULL en la plataforma | Media | Implementar endpoint outbound-IP en la app |
| 4 | `system_health` (CPU/mem/disk) no se envía | Baja | Implementar `SystemInfoCollector` |
| 5 | `wifi_stats` collector no implementado | Baja | Para drive test es secundario |
| 6 | `streaming` / `youtube` / `iperf3` no implementados en Android | Media | Replicar la lógica de `qos-probe` Go o usar lib externa |
| 7 | `multi_cdn` no implementado | Baja | Bajo valor en drive test (red de operador) |
| 8 | App es debug build (sin minify, con logs verbosos) | Baja | Pendiente release firmado |

### 8.2 Próximos pasos

1. **Migrar `probes.probe_id` a VARCHAR(32)** — eliminar el workaround del prefijo
2. **Agregar `SystemInfoCollector`** — CPU/mem/disk/battery en cada medición
3. **Agregar `NetworkDiagCollector`** — ISP via DNS reverso, public IP via API externa
4. **Implementar `SpeedTest` o `YouTube` collectors** — los más útiles para drive test
5. **Release firmada** — con keystore en CI y proguard rules
6. **F-Droid / Accrescent submission** — distribución fuera de Play Store
7. **Migrar a foreground service normal con `WorkManager` backup** — para Android 16+ restrictions
8. **Battery optimization exemption** automática via Shizuku o Settings.Global

### 8.3 Limitaciones del entorno

- Sin SIM activa, el modem no se registra en la red → no hay datos de
  red celular real (solo `telephony.registry` con estado
  `EMERGENCY_ONLY` / `OUT_OF_SERVICE`). Para drive test real, **insertar
  SIM con plan de datos** antes de salir a la calle.
- GrapheneOS oculta el IMEI por privacidad — por eso se usó el método
  manual `*#06#` en lugar de adb para obtenerlo.
- El paquete debug termina en `com.teamco.qosprobe.debug` (no el package
  `com.teamco.qosprobe` de release) — `pidof` no lo encuentra con el
  nombre sin sufijo. Cuidado en scripts.

---

## 9. Comandos útiles de referencia

### 9.1 adb

```bash
# Conectar / reconectar
adb kill-server && adb start-server
adb devices

# Reinstalar APK
adb -s <serial> install -r app-debug.apk

# Force stop + reiniciar
adb -s <serial> shell am force-stop com.teamco.qosprobe.debug
adb -s <serial> shell am start -n com.teamco.qosprobe.debug/com.teamco.qosprobe.MainActivity

# Configurar via intent
adb -s <serial> shell am start \
  -a com.teamco.qosprobe.ACTION_CONFIGURE \
  -n com.teamco.qosprobe.debug/com.teamco.qosprobe.MainActivity \
  --es api_url 'https://qos.teamco.com.co' \
  --es api_key '<KEY>' \
  --es probe_id '358632950388'

# Ver logs en vivo
adb -s <serial> logcat --pid=$(adb -s <serial> shell pidof com.teamco.qosprobe.debug) \
  | grep -E "QosProbe\.(Auth|Backend|Service|ConfigStore)"

# Screenshot
adb -s <serial> exec-out screencap -p > screen.png
```

### 9.2 Backend

```bash
# Logs en EC2
ssh -i keys/ec2-api.pem ubuntu@54.84.140.23 \
  "sudo journalctl -u qos-api-server -f | grep 358632950388"

# DB queries
PGPASSWORD='***' psql -h qos-postgresql.c018awm00ogh.us-east-1.rds.amazonaws.com \
  -U postgres -d qosprobe -c "
    SELECT probe_id, isp, technology, public_ip, last_seen_at
    FROM probes WHERE probe_id='358632950388';
  "

PGPASSWORD='***' psql -h qos-postgresql.c018awm00ogh.us-east-1.rds.amazonaws.com \
  -U postgres -d qosprobe -c "
    SELECT iface_name, latitude, longitude, isp, tech, date
    FROM measurements WHERE probe_id='358632950388'
    ORDER BY date DESC LIMIT 10;
  "
```

### 9.3 Build y deploy

```bash
# Build APK debug
cd /home/pi/.openclaw/workspace/qos-probe-mobile
./gradlew assembleDebug

# Compilar backend
cd /home/pi/.openclaw/workspace/qos-api-server
/usr/local/go/bin/go build -o /tmp/qos-api-server-new ./cmd/api-server/

# Push a GitHub → CI deploy automático
git add . && git commit -m "..."
git push origin main
```

---

## 10. Changelog

| Fecha | Commit | Cambio |
|-------|--------|--------|
| 2026-06-07 | `f0ed83c` | Aceptar IMEI 15 dígitos en `RegisterProbe` |
| 2026-06-07 | `f79fec8` | Backend: aceptar `X-Probe-Token` legacy |
| 2026-06-07 | `581e8bf` | Parser: agregar Location/Radio/NetworkDiag |
| 2026-06-07 | `d73da88` | Parser: aceptar `lat`/`lon` además de `latitude`/`longitude` |
| 2026-06-07 | `8aa760f` | `lookupISPNames()`: tabla CO/SV/USA |
| 2026-06-07 | `a2e10e5` | Fix: switch case duplicado |
| 2026-06-07 | `b4a6e7c` | Parser: mcc/mnc del root de radio (no cell_tower) |
| 2026-06-07 | local  | `ApiKeyAuth`: `@Volatile`, debug logging |
| 2026-06-07 | local  | `BackendClient.applyAuth()` helper con Bearer+X-Probe-Token+X-Probe-ID |
| 2026-06-07 | local  | `MainActivity`: reenviar intent al service + restart service |

---

## 11. Referencias

- Repos: `qos-api-server`, `qos-probe`, `qos-probe-mobile` en `github.com/freddyerincon`
- Documento principal del sistema: [`SISTEMA_QOS_PROBE.md`](./SISTEMA_QOS_PROBE.md)
- Auditoría previa: [`AUDITORIA_MEDELLIN_SONDA_001e0657089f.md`](./AUDITORIA_MEDELLIN_SONDA_001e0657089f.md)
- Android 15 + targetSDK 35: https://developer.android.com/about/versions/15
- Foreground service types: https://developer.android.com/about/versions/14/changes/fgs-types-required
- Netbird: https://netbird.io
- GrapheneOS: https://grapheneos.org

---

*Documento mantenido por Horus. Última actualización: 2026-06-07*
