# Backbone de Procesamiento — Especificación de Tópicos Kafka

**Componente:** backbone-procesamiento  
**Versión del documento:** 1.0  
**Última actualización:** 2026-05-13

---

## 1. Principios de Diseño

- **Particionamiento por `country_code`** como discriminador de tenant primario; el `device_id` se usa como clave de partición compuesta para garantizar orden por dispositivo dentro de cada país.
- **Replication factor 3** en todos los tópicos del pipeline con `min.insync.replicas=2` (tolerancia a fallo de un broker sin pérdida de durabilidad).
- **Dead-Letter Topics (DLQ)** para cada tópico que puede rechazar mensajes; los DLQ tienen retención mínima de 7 días para análisis forense.
- **Schema Registry** obligatorio para todos los tópicos: los productores registran el schema Avro antes de publicar; los consumidores validan el schema antes de deserializar. Se usa el Schema Registry de Confluent (o Apicurio como alternativa portable).
- **Retention time** diseñado para permitir replay de eventos durante la ventana de disponibilidad garantizada.
- Los nombres de tópico usan puntos como separadores (convenio del ecosistema Kafka).

---

## 2. Tabla Resumen de Tópicos

| Tópico | Particiones | Repl. Factor | min.insync | Retención | DLQ asociado |
|---|---|---|---|---|---|
| `vehicle.events.raw` | 24 | 3 | 2 | 24 h | `vehicle.events.raw.dlq` |
| `vehicle.events.deduped` | 24 | 3 | 2 | 48 h | `vehicle.events.deduped.dlq` |
| `vehicle.events.enriched` | 24 | 3 | 2 | 48 h | `vehicle.events.enriched.dlq` |
| `vehicle.alerts` | 12 | 3 | 2 | 7 días | `vehicle.alerts.dlq` |
| `stolen.vehicles.events` | 12 | 3 | 2 | 30 días | `stolen.vehicles.events.dlq` |
| `pg.events.cdc` | 12 | 3 | 2 | 7 días | `pg.events.cdc.dlq` |

> Las particiones se calculan para 5 000 eventos/seg máximo en todos los países combinados y 3 brokers de cómputo estándar (8 vCPU / 32 GB RAM). Revisar el número de particiones antes de añadir nuevos países con volumen significativo.

---

## 3. Especificación Detallada por Tópico

### 3.1 `vehicle.events.raw`

**Descripción:** Eventos crudos producidos por el bridge EMQX → Kafka. Cada mensaje corresponde a un evento MQTT publicado por un agente de borde. Es la fuente primaria del pipeline; puede contener duplicados (reintentos QoS 1).

| Propiedad | Valor |
|---|---|
| **Particiones** | 24 |
| **Replication factor** | 3 |
| **min.insync.replicas** | 2 |
| **Retención** | 24 h (`retention.ms = 86400000`) |
| **Cleanup policy** | delete |
| **Clave de partición** | `country_code + ":" + device_id` (hash Murmur2 sobre la clave compuesta) |
| **Productor** | EMQX Rule Engine (bridge nativo Kafka) |
| **Consumidor(es)** | Deduplicator (Kafka Streams, consumer group `deduplicator-cg`) |
| **Schema Avro** | `VehicleEventRaw` v1 — ver Schema Registry |
| **DLQ** | `vehicle.events.raw.dlq` |

**Condiciones de redirección al DLQ:**
- `event_id` ausente o nulo (CR-01).
- `country_code` ausente o vacío (CR-10).
- Mensaje no parseable o schema no válido.

**Payload de ejemplo (JSON representation del Avro):**
```json
{
  "event_id":     "550e8400-e29b-41d4-a716-446655440000",
  "country_code": "CO",
  "device_id":    "dev-co-001",
  "plate_raw":    "abc 123-x",
  "lat":          4.7109,
  "lon":          -74.0721,
  "confidence":   97.5,
  "image_uri":    "s3://antihurto-co/raw/2026/05/13/550e8400.jpg",
  "event_ts":     "2026-05-13T14:30:00.000Z",
  "clock_uncertain": false
}
```

---

### 3.2 `vehicle.events.deduped`

**Descripción:** Eventos únicos después del Deduplicator. Se garantiza que cada `event_id` aparece exactamente una vez en este tópico (dentro de la ventana de 24 h del estado RocksDB). El payload es idéntico al del evento crudo que lo originó, preservando todos los campos originales.

| Propiedad | Valor |
|---|---|
| **Particiones** | 24 |
| **Replication factor** | 3 |
| **min.insync.replicas** | 2 |
| **Retención** | 48 h (`retention.ms = 172800000`) |
| **Cleanup policy** | delete |
| **Clave de partición** | `device_id` (garantiza orden por dispositivo en el Enrichment Service) |
| **Productor** | Deduplicator (Kafka Streams, internal topology) |
| **Consumidor(es)** | Enrichment Service (consumer group `enrichment-cg`) |
| **Schema Avro** | `VehicleEventRaw` v1 (mismo schema que raw; no hay transformación en esta etapa) |
| **DLQ** | `vehicle.events.deduped.dlq` |

**Condiciones de redirección al DLQ:**
- `country_code` ausente (revalidación defensiva).
- Fallo de deserialización.

---

### 3.3 `vehicle.events.enriched`

**Descripción:** Eventos enriquecidos con geocodificación inversa (`street`, `city`, `country`), placa normalizada (`plate_normalized`), calidad de imagen (`image_confidence_score`, `image_blur_flag`) y estado de geocodificación (`geocoding_status`). Este tópico alimenta el Matcher Service (hot path) y sirve también como fuente de replay para re-indexar OpenSearch o re-enriquecer con nuevas versiones del Enrichment Service.

| Propiedad | Valor |
|---|---|
| **Particiones** | 24 |
| **Replication factor** | 3 |
| **min.insync.replicas** | 2 |
| **Retención** | 48 h (`retention.ms = 172800000`) |
| **Cleanup policy** | delete |
| **Clave de partición** | `country_code + ":" + plate_normalized` (optimiza el lookup en el Matcher) |
| **Productor** | Enrichment Service (consumer group `enrichment-cg` → produce tras enriquecer) |
| **Consumidor(es)** | Matcher Service (consumer group `matcher-cg`) |
| **Schema Avro** | `VehicleEventEnriched` v1 — ver Schema Registry |
| **DLQ** | `vehicle.events.enriched.dlq` |

**Condiciones de redirección al DLQ:**
- `plate_normalized` ausente (normalización imposible).
- Geocodificación retornó coordenadas inválidas y el evento no puede continuar (CR-03 casos extremos).
- `country_code` ausente.

**Campos adicionales respecto a `VehicleEventRaw`:**
```json
{
  "plate_normalized":        "ABC123X",
  "street":                  "Av. El Dorado",
  "city":                    "Bogotá",
  "country":                 "Colombia",
  "geocoding_status":        "OK",
  "image_confidence_score":  0.94,
  "image_blur_flag":         false,
  "enrichment_version":      1,
  "enriched_ts":             "2026-05-13T14:30:00.450Z"
}
```

---

### 3.4 `vehicle.alerts`

**Descripción:** Alertas generadas por el Matcher Service cuando confirma un match positivo entre `plate_normalized` y la lista roja en Redis. Cada mensaje representa un potencial avistamiento de un vehículo hurtado. El Alert Service consume este tópico para aplicar deduplicación de alertas por ventana de 5 min, persistir en PostgreSQL y enrutar al WebSocket Gateway.

| Propiedad | Valor |
|---|---|
| **Particiones** | 12 |
| **Replication factor** | 3 |
| **min.insync.replicas** | 2 |
| **Retención** | 7 días (`retention.ms = 604800000`) |
| **Cleanup policy** | delete |
| **Clave de partición** | `country_code + ":" + plate_normalized` |
| **Productor** | Matcher Service (consumer group `matcher-cg` → produce en caso de hit) |
| **Consumidor(es)** | Alert Service (consumer group `alert-service-cg`) |
| **Schema Avro** | `VehicleAlert` v1 — ver Schema Registry |
| **DLQ** | `vehicle.alerts.dlq` |

**Condiciones de redirección al DLQ:**
- Fallo de deserialización del payload de alerta.
- `location` inválida o ausente (impide routing geoespacial).

**Payload de ejemplo:**
```json
{
  "alert_id":          "77a1b2c3-d4e5-6789-abcd-ef0123456789",
  "event_id":          "550e8400-e29b-41d4-a716-446655440000",
  "country_code":      "CO",
  "plate_normalized":  "ABC123X",
  "device_id":         "dev-co-001",
  "location": {
    "lat": 4.7109,
    "lon": -74.0721
  },
  "event_ts":          "2026-05-13T14:30:00.000Z",
  "alert_ts":          "2026-05-13T14:30:00.700Z",
  "image_uri":         "s3://antihurto-co/events/2026/05/13/550e8400.jpg",
  "thumbnail_uri":     "s3://antihurto-co/events/2026/05/13/550e8400_thumb.jpg",
  "confidence":        97.5,
  "stolen_vehicle": {
    "brand":   "Chevrolet",
    "line":    "Spark",
    "color":   "Rojo",
    "stolen_at": "2025-11-01T08:00:00Z"
  }
}
```

---

### 3.5 `stolen.vehicles.events`

**Descripción:** Tópico de sincronización de la lista roja de vehículos hurtados. Los Country Adapters publican eventos de tipo `STOLEN`, `RECOVERED` y `UPDATED` cuando detectan cambios en las bases policiales por país. El incident-service también publica a este tópico cuando un vehículo pasa al estado `RECOVERED`. El Canonical Vehicles Service consume este tópico para hacer upsert en PostgreSQL y actualizar Redis.

| Propiedad | Valor |
|---|---|
| **Particiones** | 12 |
| **Replication factor** | 3 |
| **min.insync.replicas** | 2 |
| **Retención** | 30 días (`retention.ms = 2592000000`) |
| **Cleanup policy** | delete |
| **Clave de partición** | `country_code + ":" + plate_normalized` |
| **Productor(es)** | Country Adapters (uno por país); incident-service (evento `RECOVERED`) |
| **Consumidor(es)** | Canonical Vehicles Service (consumer group `canonical-vehicles-cg`); Edge Distribution Service (consumer group `edge-dist-cg`) |
| **Schema Avro** | `StolenVehicleEvent` v1 — ver Schema Registry |
| **DLQ** | `stolen.vehicles.events.dlq` |

**Payload de ejemplo (evento STOLEN):**
```json
{
  "event_type":       "STOLEN",
  "country_code":     "CO",
  "plate_normalized": "XYZ789A",
  "stolen_at":        "2026-05-10T09:30:00Z",
  "stolen_location":  "Cali, Valle del Cauca",
  "owner_id_hash":    "hmac:sha256:abc123...",
  "brand":            "Toyota",
  "vehicle_class":    "Automóvil",
  "line":             "Corolla",
  "color":            "Blanco",
  "model_year":       2022,
  "schema_version":   1,
  "source_country_id": "CO-001",
  "extensions":       {}
}
```

---

### 3.6 `pg.events.cdc`

**Descripción:** Tópico de captura de cambios del WAL de PostgreSQL (Change Data Capture), producido exclusivamente por el conector Debezium. Contiene operaciones INSERT, UPDATE y DELETE sobre las tablas monitoreadas (`vehicle_events`, `stolen_vehicles`, `alerts`, `incidents`). ClickHouse lo consume vía Kafka Engine Table para alimentar el cold path analítico. El lag esperado es < 30 s bajo carga normal.

| Propiedad | Valor |
|---|---|
| **Particiones** | 12 |
| **Replication factor** | 3 |
| **min.insync.replicas** | 2 |
| **Retención** | 7 días (`retention.ms = 604800000`) |
| **Cleanup policy** | delete |
| **Clave de partición** | Asignado por Debezium: `country_code` extraído del payload CDC |
| **Productor** | Debezium connector (Kafka Connect) |
| **Consumidor(es)** | ClickHouse Kafka Engine Table (consumer group `clickhouse-cdc-cg`) |
| **Schema Avro** | Debezium envelope `VehicleEventCDC` v1 — ver Schema Registry |
| **DLQ** | `pg.events.cdc.dlq` |

**Formato del envelope CDC (Debezium):**
```json
{
  "before": null,
  "after": {
    "event_id":          "550e8400-e29b-41d4-a716-446655440000",
    "country_code":      "CO",
    "plate_normalized":  "ABC123X",
    "event_ts":          1747144200000,
    "location":          "0101000020E6100000...",
    "enrichment_version": 1
  },
  "op": "c",
  "ts_ms": 1747144201234,
  "source": {
    "table":  "vehicle_events",
    "db":     "antihurto",
    "lsn":    12345678
  }
}
```

---

## 4. Dead-Letter Topics — Configuración Común

Todos los DLQ comparten la siguiente configuración base:

| Propiedad | Valor |
|---|---|
| **Replication factor** | 3 |
| **min.insync.replicas** | 2 |
| **Retención** | 7 días |
| **Particiones** | Mismas que el tópico padre |
| **Cleanup policy** | delete |

El mensaje en el DLQ incluye los **headers Kafka** originales más los siguientes headers adicionales:
- `dlq.reason`: descripción textual del motivo de rechazo (ej. `MISSING_EVENT_ID`, `MISSING_COUNTRY_CODE`, `SCHEMA_VALIDATION_FAILED`).
- `dlq.source.topic`: nombre del tópico de origen.
- `dlq.source.partition`: partición de origen.
- `dlq.source.offset`: offset de origen.
- `dlq.timestamp`: timestamp del rechazo (ISO 8601).
- `dlq.component`: nombre del componente que rechazó el mensaje (ej. `deduplicator`, `enrichment-service`).

---

## 5. Schema Registry

El Schema Registry es la autoridad para los contratos de datos entre productores y consumidores Kafka. Reglas de compatibilidad:

| Schema | Compatibilidad | Notas |
|---|---|---|
| `VehicleEventRaw` | `BACKWARD` | Los consumidores con schema v(n) deben poder leer mensajes con schema v(n-1) |
| `VehicleEventEnriched` | `BACKWARD` | Idem |
| `VehicleAlert` | `FULL` | Tanto productores como consumidores deben soportar versiones adyacentes |
| `StolenVehicleEvent` | `FORWARD` | Nuevos campos en el productor deben ser tolerados por consumidores antiguos |
| `VehicleEventCDC` | `NONE` | Gestionado por Debezium; no se modifica manualmente |

> **Regla operacional:** nunca se elimina un campo de un schema sin incrementar la versión principal y comunicar a todos los consumidores. El Schema Registry está configurado con `DELETE_SCHEMA_VERSIONS=false` en producción.

---

## 6. Configuración de Productores

Todos los productores del backbone usan la siguiente configuración base:

```properties
acks=all
retries=2147483647
retry.backoff.ms=100
retry.backoff.max.ms=5000
max.in.flight.requests.per.connection=5
enable.idempotence=true
compression.type=lz4
linger.ms=5
batch.size=65536
```

> `retry.backoff.ms=100` y `retry.backoff.max.ms=5000` configuran el backoff exponencial del productor ante errores recuperables como `NotEnoughReplicasException`: el tiempo de espera entre reintentos comienza en 100 ms y se duplica hasta un máximo de 5 000 ms, evitando tormentas de reintentos durante recuperación de réplicas (CR-09).

> `enable.idempotence=true` combinado con `acks=all` garantiza entrega exactly-once a nivel de productor. La semántica exactly-once end-to-end (transacciones Kafka Streams) está habilitada en el Deduplicator.

---

## 7. Configuración de Consumidores

Todos los consumidores del backbone usan la siguiente configuración base:

```properties
auto.offset.reset=earliest
enable.auto.commit=false
isolation.level=read_committed
max.poll.records=500
```

> `enable.auto.commit=false` es obligatorio para garantizar que el offset solo se confirma después de que el mensaje fue procesado exitosamente (incluida la escritura en PostgreSQL/OpenSearch/Redis). `isolation.level=read_committed` evita consumir mensajes de transacciones abortadas.

---

## 8. Alertas Operacionales por Tópico

| Condición | Umbral | Alerta |
|---|---|---|
| Consumer lag `deduplicator-cg` en `vehicle.events.raw` | > 10 000 mensajes | `kafka_lag_deduplicator_high` |
| Consumer lag `enrichment-cg` en `vehicle.events.deduped` | > 5 000 mensajes | `kafka_lag_enrichment_high` |
| Consumer lag `matcher-cg` en `vehicle.events.enriched` | > 5 000 mensajes | `kafka_lag_matcher_high` |
| Consumer lag `alert-service-cg` en `vehicle.alerts` | > 1 000 mensajes | `kafka_lag_alert_service_high` |
| Tasa de producción a cualquier DLQ | > 1 % del tópico padre | `kafka_dlq_rate_high` |
| `min.insync.replicas` no satisfecho | cualquier tópico del pipeline | `kafka_isr_underreplicated` |

Ver métricas Prometheus completas en [slo-observability.md](./slo-observability.md).
