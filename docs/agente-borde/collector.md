# Collector

**Subsistema:** Collector  
**Responsabilidad:** Normalizar la lectura cruda del ANPR en el evento canónico del agente  
**Referencia arquitectural:** [Visión General](./overview.md) · [ANPR Adapter SPI](./anpr-adapter-spi.md)

---

## 1. Propósito

El Collector recibe `PlateReading` del [ANPR Adapter SPI](./anpr-adapter-spi.md), adjunta el contexto del dispositivo (coordenadas GPS, timestamps dual), aplica el filtro de `confidence_min` y produce un `NormalizedEvent` con `event_id` idempotente. Es el punto donde la lectura cruda del hardware se convierte en el evento canónico que el resto del agente procesa y persiste.

---

## 2. Esquema del Evento Normalizado (`NormalizedEvent`)

El `NormalizedEvent` es la estructura Go central que fluye del Collector al Bloom Filter, al Queue Manager y finalmente al Uploader.

### 2.1 Definición Go

```go
// Package collector define el evento normalizado que produce el Collector.
package collector

import "time"

// NormalizedEvent es el evento canónico producido por el Collector para cada
// lectura de matrícula que supera el umbral de confianza.
type NormalizedEvent struct {
    // --- Identificación ---

    // EventID es el identificador único e idempotente del evento.
    // Se calcula como UUID v5 sobre el namespace del agente y la clave compuesta
    // device_id + plate + monotonic_ns. Ver §3 para el algoritmo completo.
    EventID string `json:"event_id"`

    // DeviceID es el identificador único del dispositivo (string opaco, asignado en provisioning).
    // Ejemplo: "CO-BOG-DEV-00142"
    DeviceID string `json:"device_id"`

    // CountryCode es el código ISO 3166-1 alpha-2 del país donde opera el dispositivo.
    // Ejemplo: "CO", "MX", "PE"
    CountryCode string `json:"country_code"`

    // --- Matrícula ---

    // Plate es la matrícula detectada, normalizada a mayúsculas sin espacios.
    // Ejemplo: "ABC123", "XYZ-999"
    Plate string `json:"plate"`

    // Confidence es el índice de confianza reportado por el componente ANPR [0.0, 1.0].
    Confidence float64 `json:"confidence"`

    // --- Timestamps ---

    // WallClockUTC es el timestamp de la captura según el reloj wall-clock del sistema,
    // en UTC, con resolución de milisegundos. Puede verse afectado por drift NTP.
    WallClockUTC time.Time `json:"wall_clock_utc"`

    // MonotonicNs es el valor del reloj monotónico del proceso Go en el momento de
    // la captura (nanosegundos desde arranque del proceso). No puede saltar hacia
    // atrás; se usa para ordenamiento relativo y para el cálculo del event_id.
    // Nota: al reiniciar el proceso el reloj monotónico se resetea; el event_id
    // sigue siendo idempotente porque el restart implica un nuevo device_id temporal
    // o bien el mismo timestamp_monotónico no se repite en la misma sesión.
    MonotonicNs int64 `json:"monotonic_ns"`

    // ClockUncertain indica que el drift NTP supera el umbral configurado
    // (por defecto 5 segundos) en el momento de la captura. Cuando es true,
    // el campo wall_clock_utc no es confiable para ordenamiento absoluto,
    // pero monotonic_ns sigue siendo válido para ordenamiento relativo.
    ClockUncertain bool `json:"clock_uncertain"`

    // --- Localización ---

    // Latitude es la latitud GPS del dispositivo en el momento de la captura.
    // Valor 0.0 si el GPS no tiene fix; en ese caso gps_fix es false.
    Latitude float64 `json:"latitude"`

    // Longitude es la longitud GPS del dispositivo en el momento de la captura.
    Longitude float64 `json:"longitude"`

    // GpsFix indica si el dispositivo tenía un fix GPS válido al momento de captura.
    GpsFix bool `json:"gps_fix"`

    // Altitude es la altitud en metros sobre el nivel del mar (opcional, 0 si no disponible).
    Altitude float64 `json:"altitude,omitempty"`

    // --- Imagen ---

    // ImagePath es la ruta en el filesystem local a la imagen capturada.
    // Vacío si el componente ANPR no produce imagen.
    ImagePath string `json:"image_path,omitempty"`

    // ImageURI es la URI en el object storage cloud una vez que la imagen fue subida
    // exitosamente. Vacío hasta que el Uploader completa la subida.
    // Ejemplo: "s3://ceiba-evidence-co/events/2024/CO-BOG-DEV-00142/abc123.jpg"
    ImageURI string `json:"image_uri,omitempty"`

    // ImageUnavailable indica que la imagen local fue eliminada por la política de
    // rotación antes de ser subida al cloud. Cuando es true, image_uri estará vacío
    // y el evento se publica igualmente. (CR-07)
    ImageUnavailable bool `json:"image_unavailable,omitempty"`

    // --- Prioridad y estado (uso interno del Queue Manager) ---

    // Priority es la prioridad del evento en la cola: "HIGH" si hubo hit en el
    // Bloom filter, "NORMAL" en caso contrario.
    Priority string `json:"priority"`

    // BloomHit indica si el Bloom filter detectó coincidencia con la lista de hurtados.
    BloomHit bool `json:"bloom_hit"`

    // --- Metadatos de transmisión (gestionados por Queue Manager / Uploader) ---

    // State es el estado de transmisión del evento: "pending", "in_flight", "delivered".
    // No se incluye en el payload MQTT final.
    State string `json:"state,omitempty"`
}
```

### 2.2 Payload MQTT Publicado

El payload que el Uploader publica en el topic `devices/<device_id>/events` es un subconjunto del `NormalizedEvent` serializado en JSON. Los campos de uso interno (`state`, `image_path`) no se incluyen:

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "device_id": "CO-BOG-DEV-00142",
  "country_code": "CO",
  "plate": "ABC-123",
  "confidence": 0.94,
  "wall_clock_utc": "2024-05-07T14:32:11.421Z",
  "monotonic_ns": 4837291847312,
  "clock_uncertain": false,
  "latitude": 4.710989,
  "longitude": -74.072092,
  "gps_fix": true,
  "altitude": 2625.0,
  "image_uri": "s3://ceiba-evidence-co/events/2024/CO-BOG-DEV-00142/550e8400.jpg",
  "bloom_hit": true,
  "priority": "HIGH"
}
```

---

## 3. Algoritmo de Construcción del `event_id` (UUID v5)

El `event_id` es un **UUID v5** que garantiza idempotencia: el mismo evento, generado múltiples veces (por ejemplo tras un reinicio del agente que reintenta enviar la misma lectura), produce siempre el mismo identificador.

### 3.1 Parámetros del UUID v5

| Parámetro | Valor |
|---|---|
| Namespace UUID | `6ba7b810-9dad-11d1-80b4-00c04fd430c8` (DNS namespace de RFC 4122, reutilizado como base canónica del proyecto) |
| Nombre (input) | `<device_id>:<plate_normalized>:<monotonic_ns>` |

### 3.2 Construcción del campo `name`

```
plate_normalized = upper(trim(plate))
name = device_id + ":" + plate_normalized + ":" + strconv.FormatInt(monotonic_ns, 10)

// Ejemplo:
// device_id      = "CO-BOG-DEV-00142"
// plate          = "abc-123"  → plate_normalized = "ABC-123"
// monotonic_ns   = 4837291847312
// name           = "CO-BOG-DEV-00142:ABC-123:4837291847312"
```

### 3.3 Cálculo del UUID v5

```go
import (
    "github.com/google/uuid"
    "strings"
    "strconv"
    "fmt"
)

var uuidNamespace = uuid.MustParse("6ba7b810-9dad-11d1-80b4-00c04fd430c8")

func buildEventID(deviceID, plate string, monotonicNs int64) string {
    plataNorm := strings.ToUpper(strings.TrimSpace(plate))
    name := fmt.Sprintf("%s:%s:%s", deviceID, plataNorm, strconv.FormatInt(monotonicNs, 10))
    return uuid.NewSHA1(uuidNamespace, []byte(name)).String()
}
```

### 3.4 Propiedades de Idempotencia

- El mismo `(device_id, plate, monotonic_ns)` siempre produce el mismo `event_id`. Esto permite que el cloud duplique-elimine reenvíos sin lógica especial en el agente.
- El `monotonic_ns` es el reloj monotónico del **proceso** al momento de la lectura. Como el reloj monotónico no retrocede dentro de una sesión de proceso, dos lecturas diferentes en la misma sesión nunca comparten el mismo `monotonic_ns`.
- Al reiniciar el proceso, el reloj monotónico se resetea a 0. Sin embargo, si el agente reintenta enviar un evento ya persistido en SQLite, ese evento ya tiene su `event_id` calculado y almacenado; no se recalcula.

---

## 4. Reglas de Timestamp Dual

El Collector captura dos timestamps en el momento de normalizar el evento:

| Campo | Reloj usado | Propósito | Afectado por NTP |
|---|---|---|---|
| `wall_clock_utc` | `time.Now().UTC()` (wall-clock del sistema) | Timestamp absoluto; correlación con otros dispositivos y fuentes externas | Sí — puede saltar por ajuste NTP |
| `monotonic_ns` | `time.Now().UnixNano()` (epoch wall-clock en ns; capturado junto a `wall_clock_utc` en la misma llamada `time.Now()`, lo que preserva el orden monotónico dentro del proceso) | Ordenamiento relativo dentro del mismo proceso; cálculo del `event_id` | No |

**Captura sincronizada:** Los dos valores se capturan en la misma instrucción Go para minimizar la divergencia entre ambos relojes:

```go
now := time.Now()  // Go captura wall-clock y monotónico en una sola llamada
event.WallClockUTC = now.UTC()
event.MonotonicNs  = now.UnixNano()
```

### 4.1 Campo `clock_uncertain`

El Collector consulta el drift NTP actual del sistema antes de construir el evento. Si el drift supera el umbral configurado (`ntp_drift_threshold_s`, por defecto `5.0` segundos), establece `ClockUncertain = true`.

```go
// Lógica de evaluación del campo clock_uncertain
driftMs := getNTPDriftMs()  // Lee /run/chrony/drift o sondea chronyc/ntpq
if math.Abs(float64(driftMs)) > float64(cfg.NTPDriftThresholdMs) {
    event.ClockUncertain = true
    log.Warn("clock_drift_warning",
        "drift_ms", driftMs,
        "event_id", event.EventID,
    )
}
```

El campo `clock_uncertain` no bloquea el procesamiento del evento; el evento se encola y se transmite normalmente. El cloud puede usar este flag para aplicar correcciones o filtros adicionales (CR-06).

---

## 5. Integración con `upload_mode`

El Collector consulta el parámetro `upload_mode` al procesar cada evento (CA-03):

| `upload_mode` | Bloom hit | Acción |
|---|---|---|
| `stolen_only` | `true` | El evento se encola con prioridad `HIGH` |
| `stolen_only` | `false` | El evento se descarta (no se persiste en SQLite) |
| `all` | `true` | El evento se encola con prioridad `HIGH` |
| `all` | `false` | El evento se encola con prioridad `NORMAL` |

**Excepción — modo degradado del Bloom filter:** Si el Bloom filter no está disponible (ver [Bloom Filter §6](./bloom-filter.md#6-modo-degradado)), el Collector trata todos los eventos como `bloom_hit = false`. En `upload_mode: stolen_only` bajo modo degradado, **no se descarta ningún evento** — todos se encolan con prioridad `NORMAL`. Esta política es conservadora: evita perder evidencia ante la imposibilidad de verificar el filtro. El Collector registra una advertencia en el log (`"event": "collector_degraded_mode"`) por cada evento procesado en este estado.

El cambio de `upload_mode` se aplica al siguiente evento procesado sin reiniciar el agente (CA-14).

---

## 6. Diagrama de Flujo del Collector

```mermaid
flowchart TD
    A[PlateReading recibida del Adapter SPI] --> B{confidence >= confidence_min?}
    B -- No --> DISC[Descarta lectura\nlog DEBUG]
    B -- Sí --> C[Captura now = time.Now()]
    C --> D[Calcula event_id\nUUID v5]
    D --> E[Adjunta GPS coords]
    E --> F{drift NTP > umbral?}
    F -- Sí --> G[clock_uncertain = true\nlog WARN]
    F -- No --> H[clock_uncertain = false]
    G --> I[Consulta Bloom Filter]
    H --> I
    I --> J{bloom_hit?}
    J -- Sí --> K[priority = HIGH\nbloom_hit = true]
    J -- No --> L{upload_mode = all?}
    L -- Sí --> M[priority = NORMAL\nbloom_hit = false]
    L -- No --> N{Bloom disponible?}
    N -- Sí --> DISC2[Descarta evento\nno se persiste]
    N -- No --> M2[priority = NORMAL\nlog WARN degraded]
    K --> QUE[Envía NormalizedEvent\nal Queue Manager]
    M --> QUE
    M2 --> QUE
```

---

## 7. Registro Estructurado

El Collector emite los siguientes eventos de log (formato JSON, nivel configurable):

| Nivel | Evento (`event`) | Campos adicionales |
|---|---|---|
| `INFO` | `event_normalized` | `event_id`, `plate`, `confidence`, `priority`, `bloom_hit` |
| `DEBUG` | `plate_discarded_low_confidence` | `plate`, `confidence`, `threshold` |
| `WARN` | `clock_drift_warning` | `drift_ms`, `event_id`, `threshold_ms` |
| `WARN` | `collector_degraded_mode` | `event_id`, `plate`, `upload_mode` |
| `DEBUG` | `event_discarded_stolen_only` | `event_id`, `plate` |

---

## 8. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [ANPR Adapter SPI](./anpr-adapter-spi.md) | Produce `PlateReading` consumida por el Collector |
| [Bloom Filter](./bloom-filter.md) | Determina `bloom_hit` y `priority` para cada evento |
| [Queue Manager](./queue-manager.md) | Persiste el `NormalizedEvent` antes de cualquier transmisión |
| [Config/OTA Manager](./config-ota-manager.md) | Actualiza `upload_mode` y `confidence_min` en caliente |
| [Health Beacon](./health-beacon.md) | Usa `clock_uncertain` para reportar `clock_drift_warning` |
| [Uploader](./uploader.md) | Lee el `NormalizedEvent` de la cola y lo publica por MQTT |
| [Visión General — Glosario](./overview.md#6-glosario-canónico) | Definiciones de `event_id`, `NormalizedEvent`, `clock_uncertain` |
| [ADRs Locales](./adr-local.md) | Justificación del UUID v5 como mecanismo de idempotencia |
