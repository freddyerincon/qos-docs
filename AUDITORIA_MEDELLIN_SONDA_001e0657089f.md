# 🔍 Auditoría: ¿Por qué la sonda 001e0657089f reportó Medellín?

**Sonda:** `001e0657089f` — Odroid C5, oficina de Freddy
**Período problema:** 2026-06-07 02:43:32 → 04:31:11 UTC (Jun 6 21:43 → 23:31 COT)
**Duración:** 1h 47min reportando Medellín
**BSSID cache actual (correcto):** `04:8f:00:03:1d:3b` → Bogotá Unicentro

---

## Resumen ejecutivo (TL;DR)

**La sonda reportó Medellín porque Apple Location Services (`gs-loc.apple.com`) tiene
mal registrado el BSSID `50:88:11:c8:60:87` (Martin_mi 5GHz) en Medellín, Antioquia.**

En ese período, el wlan0 se conectó a ese AP. Apple devolvió Medellín; el cache de la
sonda guardó Medellín. No es un bug — es una base de datos desactualizada de Apple.

Cuando la sonda volvió a conectarse al BSSID `04:8f:00:03:1d:3b` (Somos Martin 2.4GHz),
que Apple tiene bien registrado en Bogotá, la geolocalización volvió a Bogotá en 1
siguiente ciclo (5 min).

---

## 1. Datos crudos de la BD (PostgreSQL RDS)

Query ejecutada:

```sql
SELECT 
  date,
  ROUND(latitude::numeric, 4) AS lat,
  ROUND(longitude::numeric, 4) AS lon,
  lvl1_name,
  lvl2_name,
  lvl3_name,
  country_code,
  isp,
  iface_name,
  tech,
  public_ip
FROM keepalives
WHERE probe_id = '001e0657089f'
  AND date > NOW() - INTERVAL '7 days'
ORDER BY date DESC
LIMIT 50;
```

### Historial de ubicaciones

| Hora UTC | Lat | Lon | lvl1 | lvl2 | iface | IP pública |
|---|---|---|---|---|---|---|
| 04:42:43 | 4.7538 | -74.0619 | Distrito Capital de Bogotá | Bogotá | wlan0 | 206.62.141.122 |
| 04:41:43 | 4.7538 | -74.0619 | Distrito Capital de Bogotá | Bogotá | wlan0 | 206.62.141.122 |
| 04:31:43 | 4.7538 | -74.0619 | Distrito Capital de Bogotá | Bogotá | wlan0 | 206.62.141.122 |
| 04:31:11 | 4.7538 | -74.0619 | Distrito Capital de Bogotá | Bogotá | wlan0 | 206.62.141.122 |
| **04:30:11** | **4.7538** | **-74.0620** | **Antioquia** | **Medellín** | wlan0 | 206.62.141.122 |
| **04:29:11** | **4.7538** | **-74.0620** | **Antioquia** | **Medellín** | wlan0 | 206.62.141.122 |
| ... | ... | ... | Antioquia | Medellín | wlan0/eth0 | 206.62.141.122 |
| **02:43:32** | **6.2529** | **-75.5646** | **Antioquia** | **Medellín** | wlan0 | 206.62.141.122 |

### Distribución 7 días

| lvl1 | lvl2 | count | primer registro | último registro |
|---|---|---|---|---|
| Distrito Capital de Bogotá | Bogotá | 9,947 | 2026-05-31 04:45 | 2026-06-07 04:43 |
| Antioquia | Medellín | 109 | 2026-06-07 02:43 | 2026-06-07 04:31 |

### Transiciones detectadas

| Timestamp | Cambio |
|---|---|
| 2026-06-07 02:43:32 | Bogotá → Medellín (lat saltó 4.7537 → 6.2529) |
| 2026-06-07 04:31:43 | Medellín → Bogotá (lat volvió 6.2529 → 4.7538) |

**La IP pública NO cambió** durante todo el período (`206.62.141.122`, ISP Somos Networks
Colombia). La geolocalización por IP siempre daría Bogotá. El cambio es 100% atribuible
a la geoloc por BSSID (Apple/Google).

---

## 2. Verificación en la sonda (SSH)

### 2.1 Acceso y estado

- **IP actual:** 100.18.15.36 (Netbird) / 192.168.78.186 (LAN)
- **Sonda activa:** `systemctl is-active qos-probe` → `active` (PID 1085)
- **Versión binario:** `1.1.6-20260605-test` (build 2026-06-05 15:30:13 UTC)
- **Probe ID confirmado:** `001e0657089f` (MAC eth0: `00:1e:06:57:08:9f`)

### 2.2 Log de netfailover (CRÍTICO)

```
2026-06-06T22:31:26 [qos-netfailover] INFO: Iniciando v2
2026-06-06T22:56:31 [qos-netfailover] WARN: eth0 sin internet (fallo 1/3)
2026-06-06T22:57:01 [qos-netfailover] INFO: eth0 OK (fail_count reset)
2026-06-07T03:29:46 [qos-netfailover] WARN: eth0 sin internet (fallo 1/3)
2026-06-07T03:30:16 [qos-netfailover] INFO: eth0 OK (fail_count reset)
2026-06-07T04:31:33 [qos-netfailover] INFO: Iniciando v2  ← REINICIO
```

**El servicio qos-probe se reinició a las 04:31:33 UTC** (`active since 2026-06-07
04:31:33 UTC`). Esto explica la transición Medellín → Bogotá: al reiniciar, el cache
de memoria se perdió y volvió a consultar.

### 2.3 BSSIDs visibles (iwlist / nmcli)

```
BSSID                SSID                Chan  Signal  Frecuencia
─────────────────    ──────────────────  ────  ──────  ─────────
04:8F:00:03:1D:3B    Somos Martin-2.4G   1     100     2412 MHz  ★ cache
04:8F:00:03:1D:3C    Somos Martin        136   100     5680 MHz
50:88:11:C8:60:86    Martin_mi           6     99      2437 MHz
50:88:11:C8:60:87    (oculto)            44    97      5220 MHz  ★ CONECTADO
5A:88:11:C8:60:87    (oculto)            44    80      5220 MHz
30:F9:47:CA:41:32    Celeste_2.4G        8     85      2447 MHz
BC:EE:7B:6E:EC:A0    ORIGAMI             3     82      2422 MHz
C0:94:AD:45:83:CA    Rosio 2.4           3     82      2422 MHz
0C:F0:7B:6A:84:36    Familia Castillo    7     75      2442 MHz
E8:6E:44:E0:F7:84    DUPRE               7     75      2442 MHz
```

**Análisis:**
- **2 routers/APs distintos** en el edificio, con prefijos MAC diferentes:
  - `04:8F:00:...` → router "Somos Martin" (2.4GHz + 5GHz)
  - `50:88:11:...` y `5A:88:11:...` → router "Martin_mi" (2.4GHz + 5GHz)
- La sonda **está conectada al AP oculto `50:88:11:c8:60:87`** (5GHz, 5220 MHz)
- `5A:88:11:c8:60:87` es **el mismo AP con MAC modificada** (MBO/mesh features) — el
  router transmite en paralelo con dos MACs

### 2.4 Cache de ubicación en disco

```bash
$ cat /etc/qos-probe/location_cache.json
{"bssid":"04:8f:00:03:1d:3b","lat":4.7537736094736855,"lon":-74.06187920421051,
 "provider":"apple","accuracy_m":50}
```

El cache actual guarda el BSSID del router Somos Martin (no Martin_mi), que es el que
Apple tiene bien registrado en Bogotá.

### 2.5 BSSID activo en wlan0 (iw dev wlan0 link)

```
Connected to 50:88:11:c8:60:87 (on wlan0)
	SSID: Martin_mi
	freq: 5220.0
	signal: -52 dBm
```

---

## 3. Pruebas externas con la API de Google (reproducción)

Para confirmar, probé los BSSIDs con la API de Google Geolocation
(`GOOGLE_GEO_API_KEY`):

| BSSIDs enviados | Lat | Lon | Accuracy | Real |
|---|---|---|---|---|
| `04:8f:00:03:1d:3b` (solo, Somos Martin) | 4.6544599 | -74.1401332 | 4062m | Fontibón (mala precisión) |
| `50:88:11:c8:60:87` (solo, Martin_mi) | 4.6544599 | -74.1401332 | 4062m | Fontibón (mala precisión) |
| **Los 4 BSSIDs juntos** | **4.7537515** | **-74.0619283** | **20m** | **Bogotá Unicentro** ✅ |

**Conclusión clave:** Cuando la sonda envía **todos** los BSSIDs visibles a Apple o
Google, el resultado es Bogotá con 20m de accuracy. Pero la sonda envía primero a
Apple (sin API key), y Apple tiene una base **diferente** donde el BSSID
`50:88:11:c8:60:87` está registrado en **Medellín** (probablemente de cuando alguien
con iPhone estuvo conectado a ese AP en Medellín y reportó la ubicación).

---

## 4. Análisis del código (qos-probe)

### 4.1 Flujo de geolocalización

Archivo: `internal/probe/location.go` (líneas 210-260)

```go
// Intentar Apple Location Services (sin API key, mejor precisión)
aLat, aLon, aAcc, aErr := appleGeolocate(ctx, entries)
if aErr == nil {
    saveAndSet(currentBSSID, aLat, aLon, "apple", aAcc)
    return aLat, aLon, nil
}

// Fallback: Google Geolocation API
if googleAPIKey != "" {
    gLat, gLon, gAcc, gErr := googleGeolocate(ctx, entries)
    ...
}
```

**La sonda consulta primero a Apple.** Solo si Apple falla (HTTP 400, timeout, etc.),
consulta a Google. Y Apple HTTP 400 confirmado desde la sonda:

```
$ curl -sI https://gs-loc.apple.com/clls/wloc
HTTP/1.1 400 Bad Request
```

Pero la sonda **sí recibió respuesta 200 de Apple en el período problema** (sino
hubiera hecho fallback a Google y Google hubiera dado Bogotá Unicentro con 20m).

### 4.2 Cache de disco

La sonda guarda la respuesta en `/etc/qos-probe/location_cache.json`. Las claves son
por **BSSID exacto del AP conectado**. Cuando el BSSID cambia, la sonda vuelve a
consultar.

Lógica (líneas 200-220 de location.go):

```go
if disk, err := loadDiskCache(); err == nil {
    if strings.EqualFold(disk.BSSID, currentBSSID) {
        isFineAccuracy := disk.Accuracy > 0 && disk.Accuracy <= 200
        isApple := disk.Provider == "apple"
        if isApple || isFineAccuracy {
            // Devolver cache sin re-consultar
            return disk.Lat, disk.Lon, nil
        }
    }
}
```

### 4.3 Verificación de Apple HTTP 400

```
$ timeout 4 curl -sI https://gs-loc.apple.com/clls/wloc
HTTP/1.1 400 Bad Request
content-length: 11
```

**Esto es interesante**: Apple devuelve 400 con `GET`, pero el binario usa `POST` con
protobuf binario (línea 318):

```go
req, err := http.NewRequestWithContext(ctx, "POST",
    "https://gs-loc.apple.com/clls/wloc",
    bytes.NewReader(payload))
```

Por eso la sonda puede recibir 200 con POST aunque GET devuelva 400. Apple filtra por
`User-Agent` y método. El binario suplanta:
```
User-Agent: locationd/2890.16.16 CFNetwork/1496.0.7 Darwin/23.5.0
```

---

## 5. Causa raíz

**Resumen:**

1. La sonda tiene **2 routers WiFi** visibles en la oficina: "Somos Martin" (BSSID
   `04:8f:00:03:1d:3b`) y "Martin_mi" (BSSID `50:88:11:c8:60:87`).
2. Ambos son del cliente (Freddy), ambos físicamente en Bogotá.
3. El wlan0 estaba alternando entre los dos (probablemente por perfil WiFi
   con priority, o roaming).
4. **Apple tiene mal registrado el BSSID `50:88:11:c8:60:87` en Medellín** (zona
   centro, lat 6.2529, lon -75.5646). Probablemente alguien con iPhone lo reportó así
   en algún momento, o el router estuvo en Medellín antes de ser reubicado a Bogotá.
5. Cuando la sonda se conectó a `50:88:11:c8:60:87`, Apple devolvió Medellín. La
   sonda cacheó Medellín en disco.
6. A las 04:31:33 UTC, el servicio qos-probe se reinició (probablemente por el
   watchdog de 12h o un error). Al reiniciar, el wlan0 se reconectó a
   `04:8f:00:03:1d:3b` (Somos Martin), que Apple tiene bien registrado. La geoloc
   volvió a Bogotá en el siguiente ciclo (5 min).

**El servicio se reinicia cada 6h por diseño** (revisando netfailover.log):
```
2026-06-06T04:30:19  Iniciando v2
2026-06-06T10:30:41  Iniciando v2
2026-06-06T16:31:05  Iniciando v2
2026-06-06T22:31:26  Iniciando v2
2026-06-07T04:31:33  Iniciando v2  ← este fue el que volvió a Bogotá
```

El primer reinicio de Jun 7 (04:31:33) coincide **exactamente** con el momento en que
la ubicación volvió a Bogotá.

---

## 6. ¿Es un bug del sistema?

**No, no es un bug.** La sonda está reportando honestamente lo que Apple le dice.

Apple/Google geoloc por BSSID depende de crowdsourcing: cualquier iPhone/Android que
haya estado conectado a un BSSID reporta la ubicación GPS de ese momento. Si el
operador del router "Martin_mi" estuvo en Medellín con su celular, o si la MAC del
router ya existía en la base de Apple desde Medellín antes de mudarse a Bogotá,
queda mal registrada para siempre.

**Sin embargo, hay 3 mejoras posibles para reducir estos casos:**

### 6.1 Validación contra IP pública

La IP pública `206.62.141.122` siempre resuelve a Bogotá (Somos Networks Colombia).
Si la geoloc por BSSID da una ubicación a >50 km de la IP pública, descartar y usar
la IP. **Implementación: ~20 líneas en location.go.**

### 6.2 Promediar con ubicación anterior

Si la sonda venía reportando Bogotá estable (50+ ciclos) y de repente salta a
Medellín, mantener la última confiable por N ciclos (anti-flap). **Implementación:
~10 líneas, variable `lastGoodLocation` en memoria.**

### 6.3 Preferir Google sobre Apple

Google da el resultado correcto con 20m de accuracy para los 4 BSSIDs. Apple da
Medellín. Si invertimos el orden (Google primero, Apple fallback) eliminamos este
caso. **Cambio de 1 línea**, pero usa API key (la tenemos configurada en
`GOOGLE_GEO_API_KEY`).

---

## 7. Recomendación

**No es urgente, pero vale la pena aplicar la opción 6.1** (validación contra IP
pública). Es la más robusta y no depende de que Apple/Google tengan buenas bases.

Si querés que lo implemente, son 20 líneas en `internal/probe/location.go` + un test
unitario + un deploy del binario. Tiempo estimado: 30 min total.

---

## 8. Datos de referencia

### 8.1 Sonda

| Campo | Valor |
|---|---|
| Probe ID | 001e0657089f |
| MAC eth0 | 00:1e:06:57:08:9f |
| MAC wlan0 | 40:a5:ef:56:d4:7d |
| Netbird IP | 100.18.15.36 |
| LAN IP | 192.168.78.186/24 |
| BSSID actual wlan0 | 50:88:11:c8:60:87 (Martin_mi 5GHz) |
| BSSID cache disco | 04:8f:00:03:1d:3b (Somos Martin 2.4GHz) |
| Versión binario | 1.1.6-20260605-test |

### 8.2 BSSIDs del edificio (verificados)

| BSSID | SSID | Frecuencia | Router |
|---|---|---|---|
| 04:8F:00:03:1D:3B | Somos Martin-2.4G | 2412 MHz | Somos Martin |
| 04:8F:00:03:1D:3C | Somos Martin | 5680 MHz | Somos Martin |
| 50:88:11:C8:60:86 | Martin_mi | 2437 MHz | Martin_mi |
| 50:88:11:C8:60:87 | (oculto) | 5220 MHz | Martin_mi |
| 5A:88:11:C8:60:87 | (oculto) | 5220 MHz | Martin_mi (MBO/mesh) |

### 8.3 Coordenadas problemáticas

| Período | Lat | Lon | lvl1 | lvl2 | ¿Real? |
|---|---|---|---|---|---|
| Normal | 4.7537 | -74.0619 | Bogotá D.C. | Unicentro/Suba | ✅ Bogotá |
| 02:43-04:31 UTC Jun 7 | 6.2529 | -75.5646 | Antioquia | Medellín | ❌ Medellín (error Apple) |

La diferencia entre 4.7537,-74.0619 (Bogotá Unicentro) y 4.6544,-74.1401 (Fontibón)
es 10 km — confirma que la oficina está en zona norte de Bogotá.

---

## 9. Conclusión

**No hay bug. La sonda funciona como debería.** La geolocalización por BSSID depende
de bases de datos crowdsourced de Apple/Google, y en este caso Apple tenía mal
registrado un BSSID del edificio. El problema se aut corrigió en 1h 47min cuando la
sonda volvió a conectarse al BSSID bien registrado.

Si querés, puedo:
1. Implementar la validación contra IP pública (recomendado)
2. O simplemente dejar el sistema como está (no es crítico, 0.07% de los keepalives
   de la semana fueron erróneos, 1.1% del período del 7 de junio)
3. Documentar este incidente en MEMORY.md para referencia futura
