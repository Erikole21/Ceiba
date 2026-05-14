# Backbone de Procesamiento — Enrichment Service

**Componente:** backbone-procesamiento → Enrichment Service  
**Versión del documento:** 1.0  
**Última actualización:** 2026-05-13

---

## 1. Responsabilidad

El Enrichment Service es el segundo componente del hot path. Consume eventos únicos del tópico `vehicle.events.deduped` y los transforma en eventos enriquecidos que contienen toda la información necesaria para el cotejo y la presentación al oficial de policía.

**Transformaciones que realiza:**
1. **Geocodificación inversa:** traduce coordenadas GPS (`lat`, `lon`) a dirección legible (`street`, `city`, `country`). Ver [adr-geocoding-strategy.md](./adr-geocoding-strategy.md).
2. **Normalización de placa por país:** aplica reglas específicas por `country_code` para normalizar `plate_raw` a `plate_normalized` (mayúsculas, sin separadores, formato canónico).
3. **Evaluación de calidad de imagen:** calcula `image_confidence_score` (0.0–1.0) e `image_blur_flag` (booleano) para la imagen referenciada en `image_uri`.
4. **Proyección a PostgreSQL:** inserta el evento enriquecido en la tabla `vehicle_events` con columna `location` tipo `GEOGRAPHY(POINT, 4326)`.
5. **Proyección a OpenSearch:** indexa el documento enriquecido en `vehicle-events-{cc}-{YYYY-MM}` (operación best-effort).

---

## 2. Arquitectura Interna

El Enrichment Service sigue el patrón de **puertos y adaptadores hexagonales** (ADR-005). El núcleo de dominio no tiene dependencias directas con Kafka, PostgreSQL, OpenSearch ni el servicio de geocodificación; todas las integraciones ocurren a través de puertos.

```mermaid
flowchart TB
    subgraph EXTERNAL[Infraestructura externa]
        K2([vehicle.events.deduped])
        K3([vehicle.events.enriched])
        DLQ([vehicle.events.enriched.dlq])
        PG[(PostgreSQL)]
        OS[(OpenSearch)]
        GEO[Nominatim Sidecar\ndataset local]
    end

    subgraph ADAPTERS_IN[Adaptadores de entrada]
        KAFKA_IN[KafkaConsumerAdapter\nAvro deserializer]
    end

    subgraph DOMAIN[Dominio — EnrichmentService]
        ENR_UC[EnrichmentUseCase\norquesta todas las etapas]
        GEO_PORT[GeocodingPort\ninterface]
        PLATE_PORT[PlateNormalizationPort\ninterface]
        IMG_PORT[ImageQualityPort\ninterface]
        EVENTS_PORT[VehicleEventsRepositoryPort\ninterface]
        INDEX_PORT[EventsIndexPort\ninterface]
        KAFKA_OUT_PORT[EventsKafkaProducerPort\ninterface]
    end

    subgraph ADAPTERS_OUT[Adaptadores de salida]
        GEO_ADAPTER[LocalNominatimAdapter\nHTTP sidecar timeout 200ms]
        PLATE_ADAPTER[PlateNormalizerAdapter\nreglas por country_code]
        IMG_ADAPTER[ImageQualityAdapter\nS3 fetch + análisis]
        PG_ADAPTER[PostgreSQLAdapter\nSQL + PostGIS]
        OS_ADAPTER[OpenSearchAdapter\nHTTP client]
        KAFKA_ADAPTER[KafkaProducerAdapter\nAvro serializer]
    end

    K2 --> KAFKA_IN --> ENR_UC
    ENR_UC --> GEO_PORT --> GEO_ADAPTER --> GEO
    ENR_UC --> PLATE_PORT --> PLATE_ADAPTER
    ENR_UC --> IMG_PORT --> IMG_ADAPTER
    ENR_UC --> EVENTS_PORT --> PG_ADAPTER --> PG
    ENR_UC --> INDEX_PORT --> OS_ADAPTER --> OS
    ENR_UC --> KAFKA_OUT_PORT --> KAFKA_ADAPTER --> K3
    KAFKA_ADAPTER -.DLQ.-> DLQ
```

---

## 3. Puertos Hexagonales

### 3.1 `GeocodingPort`

```go
// GeocodingPort — geocodificación inversa (ver adr-geocoding-strategy.md)
type GeocodingPort interface {
    ReverseGeocode(ctx context.Context, lat, lon float64) (GeocodingResult, error)
}

type GeocodingResult struct {
    Street  string // Puede ser vacío si el dataset no resuelve la dirección exacta
    City    string
    Country string
    Status  string // "OK" | "FAILED" | "INVALID_COORDINATES"
}
```

**Implementaciones:**
- `LocalNominatimAdapter` (producción): sidecar HTTP con timeout 200 ms.
- `MockGeocodingAdapter` (tests).

### 3.2 `PlateNormalizationPort`

```go
// PlateNormalizationPort — normalización de placa por país
type PlateNormalizationPort interface {
    Normalize(ctx context.Context, plateRaw, countryCode string) (PlateNormResult, error)
}

type PlateNormResult struct {
    PlateNormalized string // Resultado normalizado (ej. "ABC123X")
    Status          string // "OK" | "UNKNOWN_FORMAT" | "NORMALIZATION_ERROR"
}
```

**Implementaciones:**
- `CountryPlateNormalizerAdapter` (producción): aplica reglas específicas por país.
- `MockPlateNormalizationAdapter` (tests).

**Reglas de normalización (CA-04):**
- Convertir a mayúsculas.
- Eliminar guiones, espacios, puntos y otros separadores.
- Validar el formato resultante contra la expresión regular del país.

| País | Formato entrada ejemplo | Resultado normalizado | Expresión regular |
|---|---|---|---|
| Colombia (CO) | `"abc 123-x"` | `"ABC123X"` | `^[A-Z]{3}[0-9]{3}[A-Z]$` (motos) o `^[A-Z]{3}[0-9]{3}$` (autos) |
| México (MX) | `"ABC-12-34"` | `"ABC1234"` | `^[A-Z]{3}[0-9]{4}$` |
| Venezuela (VE) | `"AB1-23C"` | `"AB123C"` | `^[A-Z]{2}[0-9]{3}[A-Z]$` |

> Las reglas de normalización por país se cargan desde un archivo de configuración versionado (`plate-rules.yaml`) para permitir agregar nuevos países sin recompilar el servicio.

### 3.3 `ImageQualityPort`

```go
// ImageQualityPort — evaluación de calidad de imagen
type ImageQualityPort interface {
    Evaluate(ctx context.Context, imageURI string) (ImageQualityResult, error)
}

type ImageQualityResult struct {
    ConfidenceScore float64 // 0.0 – 1.0
    BlurFlag        bool    // true si la imagen está borrosa
    Status          string  // "OK" | "IMAGE_UNAVAILABLE" | "EVALUATION_ERROR"
}
```

**Implementaciones:**
- `S3ImageQualityAdapter` (producción): descarga la imagen desde object storage, aplica análisis de nitidez (varianza del Laplaciano).
- `MockImageQualityAdapter` (tests): retorna valores predefinidos.

> La evaluación de calidad es **best-effort**: si la imagen no está disponible (`image_uri` nulo o `image_unavailable=true` en el payload original), el resultado es `ConfidenceScore=0.0`, `BlurFlag=false`, `Status="IMAGE_UNAVAILABLE"`. El evento no se bloquea.

### 3.4 `VehicleEventsRepositoryPort`

```go
// VehicleEventsRepositoryPort — escritura en PostgreSQL
type VehicleEventsRepositoryPort interface {
    InsertEvent(ctx context.Context, event EnrichedEvent) error
}
```

**Implementación:** `PostgreSQLVehicleEventsAdapter` (ver [postgresql-schema.md](./postgresql-schema.md) para el contrato de escritura).

### 3.5 `EventsIndexPort`

```go
// EventsIndexPort — indexación en OpenSearch
type EventsIndexPort interface {
    IndexEvent(ctx context.Context, event EnrichedEvent) error
}
```

**Implementación:** `OpenSearchEventsAdapter` (ver [opensearch-schema.md](./opensearch-schema.md) para el contrato de indexación).

### 3.6 `EventsKafkaProducerPort`

```go
// EventsKafkaProducerPort — producción a vehicle.events.enriched y DLQ
type EventsKafkaProducerPort interface {
    PublishEnriched(ctx context.Context, event EnrichedEvent) error
    PublishDLQ(ctx context.Context, originalMsg []byte, reason string) error
}
```

---

## 4. Flujo de Procesamiento

```mermaid
sequenceDiagram
    autonumber
    participant K2  as vehicle.events.deduped
    participant ENR as EnrichmentUseCase
    participant GEO as GeocodingPort
    participant PLT as PlateNormalizationPort
    participant IMG as ImageQualityPort
    participant PG  as VehicleEventsRepositoryPort
    participant OS  as EventsIndexPort
    participant K3  as vehicle.events.enriched
    participant DLQ as vehicle.events.enriched.dlq

    K2  ->> ENR: mensaje VehicleEventRaw (Avro)
    ENR ->> ENR: validar country_code (no vacío)
    alt country_code ausente
        ENR ->> DLQ: producir con dlq.reason=MISSING_COUNTRY_CODE
    end
    ENR ->> GEO: ReverseGeocode(lat, lon) [timeout 200ms]
    GEO -->> ENR: GeocodingResult {street, city, country, status}
    ENR ->> PLT: Normalize(plate_raw, country_code)
    PLT -->> ENR: PlateNormResult {plate_normalized, status}
    ENR ->> IMG: Evaluate(image_uri) [best-effort, async si >100ms]
    IMG -->> ENR: ImageQualityResult {confidence_score, blur_flag, status}
    ENR ->> ENR: construir EnrichedEvent
    par Proyecciones paralelas
        ENR ->> PG: InsertEvent(enrichedEvent) [ON CONFLICT DO NOTHING]
    and
        ENR ->> OS: IndexEvent(enrichedEvent) [best-effort, reintentos x3]
    end
    ENR ->> K3: PublishEnriched(enrichedEvent)
    K3  -->> ENR: ack (offset confirmado)
```

> Las proyecciones a PostgreSQL y OpenSearch ocurren **en paralelo** mediante goroutines concurrentes con un contexto de timeout de 500 ms para cada una. Un fallo de PostgreSQL cancela la producción a Kafka (el mensaje se reintentará). Un fallo de OpenSearch no cancela la producción (OpenSearch es best-effort).

---

## 5. Manejo de Fallos

### 5.1 Fallo de Geocodificación

Si la geocodificación falla (timeout, servicio no disponible, coordenadas inválidas):
- El evento avanza con `geocoding_status = "FAILED"` o `"INVALID_COORDINATES"`.
- Los campos `street`, `city`, `country` quedan vacíos.
- Se incrementa `enrichment_geocoding_failures_total` con label `status` (CA-03, CR-02, CR-03).
- Si la tasa supera el 5 % en ventana de 5 min, se activa la alerta `enrichment_geocoding_failure_rate_high`.

### 5.2 Fallo de Normalización de Placa

Si la normalización falla (formato desconocido, error interno):
- El evento avanza con `plate_normalized = plate_raw.toUpperCase()` (normalización mínima).
- Se registra `enrichment_plate_normalization_errors_total` con label `country_code`.
- Si `plate_normalized` no puede determinarse, el evento se redirige al DLQ con `dlq.reason=PLATE_NORMALIZATION_ERROR`.

### 5.3 Fallo de Evaluación de Imagen

Si la imagen no está disponible o la evaluación falla:
- El evento avanza con `image_confidence_score = 0.0`, `image_blur_flag = false`, `image_quality_status = "IMAGE_UNAVAILABLE"` o `"EVALUATION_ERROR"`.
- El evento no se bloquea; la imagen es opcional para el hot path.

### 5.4 Fallo de Escritura en PostgreSQL

Si el INSERT en PostgreSQL falla (error de conexión, timeout):
- El Enrichment Service no confirma el offset Kafka del mensaje.
- Kafka Streams reintenta el procesamiento con backoff exponencial.
- Si persiste tras 3 reintentos, el mensaje se envía al DLQ con `dlq.reason=POSTGRES_WRITE_ERROR`.
- Se activa la alerta `enrichment_postgres_write_errors_high`.

### 5.5 Fallo de Indexación en OpenSearch

Si el INDEX en OpenSearch falla:
- El evento igual se produce a `vehicle.events.enriched` (OpenSearch es best-effort).
- Se reintenta hasta 3 veces con backoff exponencial (500 ms, 1 s, 2 s).
- Si persiste, se registra `opensearch_index_errors_total` y se activa la alerta si la tasa supera 0.1 %.

### 5.6 Fallo del Schema Registry

Si el Schema Registry no está disponible:
- El Enrichment Service no puede deserializar los mensajes de entrada.
- El consumo se pausa y se activa la alerta `enrichment_schema_registry_unavailable`.

---

## 6. Schema del Evento Enriquecido (`VehicleEventEnriched`)

```json
{
  "event_id":              "550e8400-e29b-41d4-a716-446655440000",
  "country_code":          "CO",
  "device_id":             "dev-co-001",
  "plate_raw":             "abc 123-x",
  "plate_normalized":      "ABC123X",
  "lat":                   4.7109,
  "lon":                   -74.0721,
  "street":                "Av. El Dorado",
  "city":                  "Bogotá",
  "country":               "Colombia",
  "geocoding_status":      "OK",
  "event_ts":              "2026-05-13T14:30:00.000Z",
  "received_ts":           "2026-05-13T14:30:00.200Z",
  "enriched_ts":           "2026-05-13T14:30:00.450Z",
  "confidence":            97.5,
  "image_uri":             "s3://antihurto-co/events/2026/05/13/550e8400.jpg",
  "thumbnail_uri":         "s3://antihurto-co/events/2026/05/13/550e8400_thumb.jpg",
  "image_confidence_score": 0.94,
  "image_blur_flag":       false,
  "image_quality_status":  "OK",
  "clock_uncertain":       false,
  "image_unavailable":     false,
  "enrichment_version":    1,
  "extensions":            {}
}
```

**Clave de partición en `vehicle.events.enriched`:** `country_code + ":" + plate_normalized` (facilita el lookup en el Matcher).

---

## 7. Métricas Prometheus

| Métrica | Tipo | Descripción |
|---|---|---|
| `enrichment_events_processed_total` | Counter | Total de eventos procesados. |
| `enrichment_events_forwarded_total` | Counter | Eventos producidos exitosamente a `vehicle.events.enriched`. |
| `enrichment_dlq_messages_total` | Counter | Mensajes enviados al DLQ; label `reason`. |
| `enrichment_processing_duration_seconds` | Histogram | Latencia total de enriquecimiento. Objetivo p95 < 300 ms. |
| `enrichment_geocoding_duration_seconds` | Histogram | Latencia de la geocodificación inversa. Objetivo p95 < 200 ms. |
| `enrichment_geocoding_failures_total` | Counter | Fallos de geocodificación; label `status` (`FAILED`, `INVALID_COORDINATES`). |
| `enrichment_geocoding_failure_rate` | Gauge | Porcentaje de fallos en ventana móvil de 5 min. Alerta si > 5 %. |
| `enrichment_plate_normalization_errors_total` | Counter | Errores de normalización de placa; label `country_code`. |
| `enrichment_image_quality_evaluations_total` | Counter | Evaluaciones de calidad de imagen; label `status`. |
| `enrichment_image_blur_detected_total` | Counter | Imágenes con `blur_flag = true` (CA-05). |
| `enrichment_postgres_write_duration_seconds` | Histogram | Latencia de escritura en PostgreSQL. Objetivo p95 < 100 ms. |
| `enrichment_postgres_write_errors_total` | Counter | Errores de escritura en PostgreSQL. |
| `enrichment_opensearch_index_duration_seconds` | Histogram | Latencia de indexación en OpenSearch. |
| `enrichment_opensearch_index_errors_total` | Counter | Errores de indexación en OpenSearch. |
| `enrichment_kafka_consumer_lag` | Gauge | Lag del consumer group `enrichment-cg` en `vehicle.events.deduped`. |

---

## 8. Configuración del Servicio

```yaml
# enrichment-service config (variables de entorno / ConfigMap Kubernetes)
KAFKA_BOOTSTRAP_SERVERS: "kafka-broker-1:9092,kafka-broker-2:9092,kafka-broker-3:9092"
KAFKA_GROUP_ID: "enrichment-cg"
KAFKA_INPUT_TOPIC: "vehicle.events.deduped"
KAFKA_OUTPUT_TOPIC: "vehicle.events.enriched"
KAFKA_DLQ_TOPIC: "vehicle.events.enriched.dlq"
SCHEMA_REGISTRY_URL: "http://schema-registry:8081"

GEOCODING_SIDECAR_URL: "http://localhost:8765"
GEOCODING_TIMEOUT_MS: "200"

POSTGRES_DSN: "postgresql://app_writer:${POSTGRES_PASSWORD}@postgres-primary:5432/antihurto"
POSTGRES_MAX_CONNECTIONS: "20"
POSTGRES_CONNECT_TIMEOUT_MS: "5000"
POSTGRES_WRITE_TIMEOUT_MS: "3000"

OPENSEARCH_URL: "http://opensearch-cluster:9200"
OPENSEARCH_MAX_RETRIES: "3"
OPENSEARCH_RETRY_BACKOFF_MS: "500"

PLATE_RULES_CONFIG_PATH: "/config/plate-rules.yaml"

IMAGE_QUALITY_ENABLED: "true"
IMAGE_QUALITY_TIMEOUT_MS: "500"
S3_ENDPOINT: "https://s3.amazonaws.com"

ENRICHMENT_VERSION: "1"
METRICS_PORT: "9090"
HEALTH_PORT: "8080"
```

---

## 9. Despliegue en Kubernetes

```yaml
# Fragmento del Deployment (ver helm/README.md para values completos)
replicas: 3
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"
livenessProbe:
  httpGet:
    path: /health/liveness
    port: 8080
  initialDelaySeconds: 30
readinessProbe:
  httpGet:
    path: /health/readiness
    port: 8080
  initialDelaySeconds: 10
```

El Enrichment Service es **stateless** (no tiene state store local); puede escalarse horizontalmente aumentando `replicas`. El número máximo de réplicas productivas está limitado por el número de particiones de `vehicle.events.deduped` (24).

---

## 10. Criterios de Aceptación Cubiertos

| CA/CR | Verificación |
|---|---|
| CA-03: Geocodificación inversa completa | El evento enriquecido contiene `street`, `city`, `country` y `geocoding_status=OK` para coordenadas válidas. |
| CA-04: Normalización de placa CO `"abc 123-x"` → `"ABC123X"` | `PlateNormalizationPort.Normalize("abc 123-x", "CO")` retorna `"ABC123X"`. |
| CA-05: Evaluación de calidad de imagen | El evento contiene `image_confidence_score` (0.0–1.0) y `image_blur_flag`; si `blur_flag=true` se registra métrica. |
| CA-06: Proyección a PostgreSQL con PostGIS | INSERT en `vehicle_events` con `location = ST_MakePoint(lon, lat)::geography` y `country_code`. |
| CA-07: Proyección a OpenSearch | Documento indexado en `vehicle-events-co-2026-05` con campos buscables en < 200 ms (búsqueda tras refresh). |
| CR-02: Fallo de geocodificación — evento avanza | `geocoding_status="FAILED"`, campos en blanco, métrica incrementada, evento producido a `vehicle.events.enriched`. |
| CR-03: Coordenadas inválidas — `geocoding_status="INVALID_COORDINATES"` | No se escribe geometría PostGIS inválida; `location` se persiste como NULL. |
| CR-10: `country_code` ausente → DLQ | Header `dlq.reason=MISSING_COUNTRY_CODE`. |
