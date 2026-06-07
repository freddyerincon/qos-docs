# QoS Probe — Documentación

Documentación técnica del sistema de sondas de medición QoE/QoS.

## Contenido

- [SISTEMA_QOS_PROBE.md](./SISTEMA_QOS_PROBE.md) — Documento principal (13 secciones, ~53KB)
- [SONDA_MOVIL_ANDROID_PIXEL.md](./SONDA_MOVIL_ANDROID_PIXEL.md) — Sonda móvil Android (Pixel 8 Pro + GrapheneOS) — deploy, bugs y fixes
- [AUDITORIA_MEDELLIN_SONDA_001e0657089f.md](./AUDITORIA_MEDELLIN_SONDA_001e0657089f.md) — Incidente Medellín y validación BSSID vs IP pública

## Estado del proyecto

- **Sondas fijas activas:** 13+ (Odroid XU4, C5, NanoPi, Pi5) en CO / SV
- **Sonda móvil activa:** `mob-358632` (Pixel 8 Pro, GrapheneOS, Netbird)
- **Binario en producción:** `1.1.6-20260605-test`
- **API en producción:** EC2 `54.84.140.23` (commit `b4a6e7c+`)
- **BD:** RDS PostgreSQL `qosprobe` en `qos-postgresql.c018awm00ogh.us-east-1.rds.amazonaws.com`
