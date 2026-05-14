# Health Beacon

**Subsistema:** Health Beacon  
**Responsabilidad:** Publicación periódica del estado operacional del dispositivo hacia el cloud  
**Referencia arquitectural:** [Visión General](./overview.md) · [Propuesta §3.1](../propuesta-arquitectura-hurto-vehiculos.md#31-borde)

---

## 1. Propósito

El Health Beacon publica un snapshot del estado del dispositivo cada 60 segundos en el topic `devices/<device_id>/health`. Permite al cloud detectar dispositivos degradados, con recursos críticos o con problemas de conectividad antes de que fallen completamente.

El payload se codifica en **MessagePack (msgpack)** para reducir el tamaño del mensaje y el consumo de banda GSM frente a JSON sin formato.

---

## 2. Topic de Publicación

| Propiedad | Valor |
|---|---|
| Topic | `devices/<device_id>/health` |
| QoS | 1 |
| Retain | `false` |
| Intervalo | 60 segundos (configurable vía [Config/OTA Manager](./config-ota-manager.md)) |
| Codificación | MessagePack (msgpack v2) |
| Tamaño típico del payload | ~180–320 bytes (comprimido) vs ~600–900 bytes (JSON) |

---

## 3. Esquema del Payload

### 3.1 Estructura Go

```go
// Package health define el payload del Health Beacon.
package health

// Beacon es el payload completo publicado en devices/<device_id>/health.
type Beacon struct {
    // --- Identificación ---
    DeviceID    string `msgpack:"did"`   // Identificador del dispositivo
    CountryCode string `msgpack:"cc"`    // Código ISO del país
    Timestamp   int64  `msgpack:"ts"`    // UNIX ms en el momento de la publicación
    AgentVersion string `msgpack:"av"`  // Versión semántica del binario del agente

    // --- Recursos del sistema ---
    CPUPercent  float32 `msgpack:"cpu"` // % de uso de CPU del proceso (promedio 10 s)
    RSSMegabytes float32 `msgpack:"rss"` // RSS del proceso en MB
    DiskFreeGB  float32 `msgpack:"dsk"` // Espacio libre en /var/agent en GB
    BatteryPercent int8 `msgpack:"bat"` // Nivel de batería de respaldo en % (-1 si no disponible)

    // --- Conectividad ---
    RSSIdbm     int8   `msgpack:"rssi"` // RSSI de la señal GSM en dBm (típico: -50 a -110)
    NTPDriftMs  int32  `msgpack:"ntp"`  // Drift NTP en milisegundos (positivo o negativo)

    // --- Cola de eventos ---
    QueuePendingTotal int32 `msgpack:"qpt"` // Total de eventos pending en cola
    QueuePendingHigh  int16 `msgpack:"qph"` // Eventos pending con prioridad HIGH
    OldestPendingS    int32 `msgpack:"qoa"` // Antigüedad del evento pending más viejo (segundos)

    // --- Campos de alerta (booleanos, omitidos si false) ---
    AlertDiskLow        bool `msgpack:"a_dsk,omitempty"`  // Espacio < 500 MB (CR-02)
    AlertANPRUnavailable bool `msgpack:"a_anpr,omitempty"` // ANPR no alcanzable (CR-01)
    AlertBloomUnavailable bool `msgpack:"a_bf,omitempty"`  // Bloom filter ausente (CR-05)
    AlertClockDrift     bool `msgpack:"a_ntp,omitempty"`  // Drift NTP > umbral (CR-06)
    AlertCertExpired    bool `msgpack:"a_cert,omitempty"` // Certificado mTLS vencido (CR-04)
    AlertOTAError       bool `msgpack:"a_ota,omitempty"`  // Error en OTA reciente (CR-03)
    OTAErrorDetail      string `msgpack:"a_ota_msg,omitempty"` // Detalle del error OTA
    AlertConfigLocked   bool `msgpack:"a_locked,omitempty"` // Modo config locked activo (ver security.md §4)

    // --- Campos de diagnóstico adicionales ---
    UptimeSeconds      int64  `msgpack:"up"`  // Segundos desde el arranque del agente
    LastEventDelivered int64  `msgpack:"led"` // UNIX ms del último evento delivered
    LastHealthSentAt   int64  `msgpack:"lhs"` // UNIX ms del envío anterior de health beacon
}
```

### 3.2 Ejemplo de Payload en Representación JSON (antes de codificación msgpack)

```json
{
  "did": "CO-BOG-DEV-00142",
  "cc": "CO",
  "ts": 1715000000000,
  "av": "1.4.2",
  "cpu": 2.3,
  "rss": 38.5,
  "dsk": 11.2,
  "bat": 87,
  "rssi": -73,
  "ntp": 142,
  "qpt": 12,
  "qph": 3,
  "qoa": 47,
  "up": 86400,
  "led": 1714999995000,
  "lhs": 1714999940000
}
```

En este ejemplo no hay alertas activas, por lo que los campos `a_*` se omiten completamente del payload msgpack (ahorro de espacio).

### 3.3 Ejemplo de Payload con Alertas

```json
{
  "did": "CO-BOG-DEV-00142",
  "cc": "CO",
  "ts": 1715000060000,
  "av": "1.4.2",
  "cpu": 1.8,
  "rss": 41.0,
  "dsk": 0.3,
  "bat": 12,
  "rssi": -95,
  "ntp": 5800,
  "qpt": 8420,
  "qph": 14,
  "qoa": 14400,
  "a_dsk": true,
  "a_ntp": true,
  "a_locked": true,
  "up": 172800,
  "led": 1714985000000,
  "lhs": 1715000000000
}
```

Aquí `a_dsk: true` indica disco bajo, `a_ntp: true` indica drift NTP > 5 s, y `a_locked: true` indica que el agente detectó una modificación no autorizada de configuración o binario (ver [security.md §4](./security.md#4-modo-config-locked-ante-tampering)).

---

## 4. Recolección de Métricas

### 4.1 CPU del Proceso

El Health Beacon mide el CPU del proceso usando `/proc/<pid>/stat` (Linux):

```go
func processCPUPercent(pid int, interval time.Duration) float32 {
    before := readProcStat(pid)
    time.Sleep(interval)
    after := readProcStat(pid)
    // Calcular diferencia de jiffies / tiempo transcurrido / Hz del sistema
    return float32((after.utime + after.stime - before.utime - before.stime) * 100 /
        (interval.Seconds() * float64(clkTck)))
}
```

La medición es el promedio de los últimos 10 segundos.

### 4.2 RSS del Proceso

```go
func processRSSMB(pid int) float32 {
    data, _ := os.ReadFile(fmt.Sprintf("/proc/%d/status", pid))
    // Buscar línea "VmRSS: <kb> kB"
    return float32(parseKB(data, "VmRSS")) / 1024.0
}
```

### 4.3 Espacio Libre en Disco

```go
freeBytes, _ := checkDiskFree("/var/agent")
freeGB := float32(freeBytes) / (1024 * 1024 * 1024)
```

### 4.4 Nivel de Batería

Se lee desde el archivo de sysfs del controlador de batería:

```
/sys/class/power_supply/BAT0/capacity   ← % entero (0-100)
```

Si el archivo no existe o no es legible, `bat = -1`.

### 4.5 RSSI GSM

Se obtiene interrogando el módem GSM via AT commands o leyendo el archivo de estado del módulo:

```
AT+CSQ  → respuesta: +CSQ: <rssi>,<ber>
```

La conversión de unidades CSQ a dBm: `rssi_dbm = -113 + (csq_value * 2)`.

### 4.6 Drift NTP

```go
func ntpDriftMs() int32 {
    // Leer offset de chrony (preferido) o ntpq
    out, _ := exec.Command("chronyc", "tracking").Output()
    // Parsear línea "System time     : <offset> seconds"
    return int32(parseOffset(out) * 1000)
}
```

Si `chronyc` no está disponible, se usa `ntpq -p` como alternativa.

---

## 5. Lógica de Publicación

### 5.1 Goroutine del Health Beacon

```go
func (h *HealthBeacon) Run(ctx context.Context) {
    ticker := time.NewTicker(h.interval) // por defecto 60 s
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            beacon := h.collect()
            payload, _ := msgpack.Marshal(beacon)
            h.mqtt.Publish("devices/"+h.deviceID+"/health", 1, false, payload)
            h.lastSentAt = time.Now()
        case <-ctx.Done():
            return
        }
    }
}
```

### 5.2 Publicación Fuera de Ciclo ante Alerta Crítica

El Health Beacon también puede publicar fuera del ciclo de 60 s cuando un subsistema reporta una condición crítica inmediata:

| Condición | Subsistema que notifica | Publicación inmediata |
|---|---|---|
| Error OTA (firma inválida) | Config/OTA Manager | Sí (con `a_ota: true`) |
| Certificado mTLS vencido detectado | Uploader | En el próximo ciclo (60 s) |
| Disco bajo (< 500 MB) | Queue Manager | En el próximo ciclo |
| Config locked activado | Security (verificación interna) | En el próximo ciclo (con `a_locked: true`) |

Las alertas críticas de OTA se publican inmediatamente porque el operador necesita saberlo antes del siguiente heartbeat ordinario.

---

## 6. Compresión: msgpack vs CBOR

La elección de codificación es msgpack v2 (CA-08):

| Formato | Tamaño típico (sin alertas) | Tamaño típico (con 3 alertas) | Librería Go |
|---|---|---|---|
| JSON sin formato | ~600 bytes | ~700 bytes | `encoding/json` |
| JSON comprimido (gzip) | ~220 bytes | ~250 bytes | `compress/gzip` |
| msgpack | ~180 bytes | ~210 bytes | `github.com/vmihailov/msgpack` |
| CBOR | ~175 bytes | ~205 bytes | `github.com/fxamacker/cbor` |

Se selecciona **msgpack** por:

- Madurez de la librería Go (`vmihailov/msgpack` es ampliamente usada en producción).
- Diferencia marginal respecto a CBOR (~5 bytes).
- Tooling de diagnóstico más extendido (conversores online, soporte en Wireshark).

---

## 7. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [Queue Manager](./queue-manager.md) | Provee `QueueStats` (eventos pending, antigüedad, alerta de disco) |
| [Image Store](./image-store.md) | Provee estadísticas de imágenes almacenadas |
| [Uploader](./uploader.md) | Publica el payload en el topic MQTT; comparte la sesión MQTT |
| [ANPR Adapter SPI](./anpr-adapter-spi.md) | Notifica flag `anpr_unavailable` cuando el adapter falla |
| [Bloom Filter](./bloom-filter.md) | Notifica flag `bloom_unavailable` cuando el filtro no está disponible |
| [Config/OTA Manager](./config-ota-manager.md) | Notifica flag `ota_error` tras falla de verificación de firma |
| [Visión General](./overview.md) | Restricciones de footprint y glosario canónico |
