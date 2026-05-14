# Backbone de Procesamiento — PostgreSQL Schema (Perspectiva de Escritura)

**Componente:** backbone-procesamiento — escritura a PostgreSQL  
**Versión del documento:** 1.0

> **Nota de coordinación:** Este documento describe la perspectiva de escritura del `backbone-procesamiento` sobre PostgreSQL. La **fuente de verdad del schema** (DDL completo, particionamiento, índices, roles, extensiones) está en:
>
> **[docs/almacenamiento-lectura/postgresql-schema.md](../almacenamiento-lectura/postgresql-schema.md)**
>
> Todo cambio de schema debe realizarse en ese documento y luego reflejarse aquí. No duplicar DDL en este archivo.

---

## 1. Responsabilidades de Escritura del Backbone

El backbone de procesamiento escribe en PostgreSQL a través de tres servicios:

| Servicio | Tabla(s) escritas | Operación |
|---|---|---|
| **Enrichment Service** | `vehicle_events` | INSERT (un registro por evento enriquecido) |
| **Canonical Vehicles Service** | `stolen_vehicles` | INSERT + UPDATE (upsert por `country_code, plate_normalized`) |
| **incident-service** | `incidents` | INSERT + UPDATE (ciclo de vida de recuperación) |
| **API Gateway (interceptor)** | `audit_log` | INSERT (registro inmutable de accesos) |

---

## 2. Contrato de Escritura — `vehicle_events`

El `Enrichment Service` produce el evento enriquecido desde el tópico Kafka `vehicle.events.enriched` e inserta en `vehicle_events`. Los campos requeridos y su origen:

| Campo | Tipo | Origen | Notas |
|---|---|---|---|
| `event_id` | UUID | Agente de borde (UUID v5) | Clave de deduplicación; el Deduplicator garantiza unicidad antes de llegar al Enrichment |
| `country_code` | CHAR(2) | Topic partition key / JWT del dispositivo | Discriminador de tenant (ADR-011) |
| `device_id` | TEXT | Payload MQTT | Identificador del dispositivo de borde |
| `plate_raw` | TEXT | Payload MQTT | Matrícula original del ANPR |
| `plate_normalized` | TEXT | Collector (agente de borde) | Mayúsculas, sin guiones; normalización idempotente |
| `event_ts` | TIMESTAMPTZ | Payload MQTT (wall-clock del dispositivo) | Puede tener drift si `clock_uncertain=true` |
| `confidence` | NUMERIC(5,2) | Payload MQTT | Score 0.00–100.00 del ANPR |
| `location` | GEOGRAPHY(POINT) | Enrichment (conversión GPS lat/lon → PostGIS) | `ST_MakePoint(lon, lat)::geography` |
| `image_uri` | TEXT | Upload Service (presigned PUT) | Puede ser NULL si no se subió la imagen |
| `thumbnail_uri` | TEXT | Upload Service | Puede ser NULL |
| `clock_uncertain` | BOOLEAN | Payload MQTT | `true` si drift NTP > 5 s en el momento de captura |
| `image_unavailable` | BOOLEAN | Payload MQTT | `true` si imagen fue rotada antes de subir |
| `enrichment_version` | SMALLINT | Enrichment Service | Versión del pipeline; permite replay con filtro de versión |
| `extensions` | JSONB | Enrichment Service | Campos adicionales por país (ej. zona, operador de cámara) |

```sql
-- Fragmento de INSERT del Enrichment Service (pseudocódigo)
INSERT INTO vehicle_events (
    event_id, country_code, device_id,
    plate_raw, plate_normalized,
    event_ts, received_ts, confidence,
    location, image_uri, thumbnail_uri,
    clock_uncertain, image_unavailable,
    enrichment_version, extensions
)
VALUES (
    $1::uuid, $2, $3,
    $4, $5,
    $6::timestamptz, now(), $7::numeric,
    ST_MakePoint($8, $9)::geography, $10, $11,
    $12::boolean, $13::boolean,
    $14::smallint, $15::jsonb
)
ON CONFLICT (event_id, country_code) DO NOTHING;  -- Idempotente: si ya existe, ignorar
```

---

## 3. Contrato de Escritura — `stolen_vehicles`

El `Canonical Vehicles Service` hace upsert de vehículos hurtados cuando recibe eventos del tópico Kafka `stolen.vehicles.events` (producido por los Country Adapters):

```sql
-- Upsert idempotente con schema_version
INSERT INTO stolen_vehicles (
    country_code, plate_normalized,
    stolen_at, stolen_location, owner_id_hash,
    brand, vehicle_class, line, color, model_year,
    schema_version, source_country_id, status,
    extensions, synced_at, synced_by_adapter
)
VALUES (
    $1, $2,
    $3::timestamptz, $4, $5,
    $6, $7, $8, $9, $10::smallint,
    $11::smallint, $12, 'stolen',
    $13::jsonb, now(), $14
)
ON CONFLICT (country_code, plate_normalized)
DO UPDATE SET
    stolen_at         = EXCLUDED.stolen_at,
    stolen_location   = EXCLUDED.stolen_location,
    owner_id_hash     = EXCLUDED.owner_id_hash,
    brand             = EXCLUDED.brand,
    vehicle_class     = EXCLUDED.vehicle_class,
    line              = EXCLUDED.line,
    color             = EXCLUDED.color,
    model_year        = EXCLUDED.model_year,
    schema_version    = EXCLUDED.schema_version,
    source_country_id = EXCLUDED.source_country_id,
    status            = 'stolen',
    extensions        = EXCLUDED.extensions,
    synced_at         = now(),
    synced_by_adapter = EXCLUDED.synced_by_adapter;
```

---

## 4. Coordinación con Almacenamiento de Lectura

- **Debezium CDC:** Los INSERT en `vehicle_events` son capturados automáticamente por el slot de replicación `debezium_slot_events` y propagados a ClickHouse. El Enrichment Service no debe notificar a ClickHouse directamente.
- **Escritura concurrente:** El role `app_writer` tiene permisos de INSERT/UPDATE en `vehicle_events`, `stolen_vehicles` e `incidents`. No tiene permisos de SELECT (la lectura se hace exclusivamente con `app_reader` o a través del `search-service`).
- **Consistencia:** La proyección a OpenSearch ocurre en paralelo a la escritura en PostgreSQL (el Enrichment Service escribe en ambos en el mismo ciclo de procesamiento del evento Kafka). La consistencia entre PostgreSQL y OpenSearch es eventual; en caso de divergencia, PostgreSQL es la fuente de verdad.

Ver el schema completo, índices y política de retención en [docs/almacenamiento-lectura/postgresql-schema.md](../almacenamiento-lectura/postgresql-schema.md).
