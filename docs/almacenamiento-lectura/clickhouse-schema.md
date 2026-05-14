# ClickHouse — Schema Analítico

**Componente:** Almacenamiento de lectura — capa ClickHouse  
**Versión del documento:** 1.0  
**Referencia:** [adr-clickhouse-ingestion.md](./adr-clickhouse-ingestion.md) · [debezium-cdc.md](./debezium-cdc.md) · [clickhouse-h3-views.md](./clickhouse-h3-views.md)

---

## 1. Motor y Justificación

ClickHouse almacena los eventos de avistamiento en formato columnar para soportar las consultas analíticas del dashboard (Superset) y el mapa de calor H3. Se usa el motor `ReplicatedMergeTree` en producción (con ZooKeeper/ClickHouse Keeper para la coordinación de réplicas) y `MergeTree` para entornos de desarrollo de un solo nodo.

**Por qué `ReplacingMergeTree` en lugar de `MergeTree` puro:**

El canal CDC puede producir actualizaciones de eventos (ej. corrección de coordenadas). `ReplacingMergeTree` deduplica filas con la misma `(event_id, country_code)` conservando la versión más reciente, según `ingested_at`. La deduplicación ocurre en el fondo durante los merges; las consultas deben incluir `FINAL` o `WHERE _version = max(_version)` si se requiere exactitud absoluta en datos recientes.

---

## 2. Tabla Principal — `vehicle_events_daily`

```sql
CREATE TABLE vehicle_events_daily
(
    event_id            UUID            NOT NULL COMMENT 'UUID v5 del evento (mismo que PostgreSQL)',
    country_code        FixedString(2)  NOT NULL COMMENT 'ISO 3166-1 alpha-2',
    device_id           String          NOT NULL COMMENT 'ID del dispositivo de borde',
    plate_normalized    String          NOT NULL COMMENT 'Matrícula normalizada',
    event_ts            DateTime64(3, 'UTC') NOT NULL COMMENT 'Timestamp del evento',
    received_ts         DateTime64(3, 'UTC') NOT NULL COMMENT 'Timestamp de recepción en cloud',
    confidence          Float32         NOT NULL COMMENT 'Score de confianza ANPR (0-100)',
    lat                 Float64         NOT NULL COMMENT 'Latitud WGS-84',
    lon                 Float64         NOT NULL COMMENT 'Longitud WGS-84',
    h3_index_8          UInt64          NOT NULL COMMENT 'Índice H3 resolución 8 (~0.74 km²)',
    image_uri           Nullable(String) COMMENT 'URI del objeto en S3-API',
    thumbnail_uri       Nullable(String) COMMENT 'URI del thumbnail en S3-API',
    clock_uncertain     UInt8           NOT NULL DEFAULT 0 COMMENT '1 si drift NTP > umbral',
    image_unavailable   UInt8           NOT NULL DEFAULT 0 COMMENT '1 si imagen fue rotada',
    enrichment_version  UInt8           NOT NULL DEFAULT 1 COMMENT 'Versión del pipeline',
    cdc_op              String          NOT NULL DEFAULT 'c' COMMENT 'Operación CDC: c/u/d/r',
    source_lsn          String          COMMENT 'LSN PostgreSQL del evento CDC',
    ingested_at         DateTime64(3, 'UTC') NOT NULL DEFAULT now() COMMENT 'Timestamp de ingestión en ClickHouse'
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/vehicle_events_daily',
    '{replica}'
)
PARTITION BY (
    country_code,
    toYYYYMM(event_ts)
)
ORDER BY (
    country_code,
    toDate(event_ts),
    h3_index_8,
    plate_normalized,
    event_id
)
PRIMARY KEY (
    country_code,
    toDate(event_ts),
    h3_index_8
)
SETTINGS
    index_granularity = 8192,
    min_bytes_for_wide_part = 104857600,  -- 100 MB
    min_rows_for_wide_part = 1000000
TTL
    event_ts + INTERVAL 24 MONTH DELETE;

-- Comentario sobre la clave de ordenamiento:
-- (country_code, fecha, h3_index_8, plate_normalized, event_id)
-- Optimiza las consultas de analítica geoespacial (GROUP BY h3_index_8)
-- y las búsquedas por placa dentro de un país y fecha.
-- PRIMARY KEY es un prefijo de ORDER BY para el índice sparse de ClickHouse.
```

### 2.1 Particionamiento

El particionamiento compuesto `(country_code, toYYYYMM(event_ts))` crea particiones con el patrón `CO_202605`, `MX_202605`. Esta granularidad permite:

- **Eliminar particiones antiguas** directamente con `ALTER TABLE ... DROP PARTITION` (eficiente, O(1)).
- **Consultas por país y mes** con partition pruning en ambas dimensiones.
- **Isolamiento operacional por país** para mantenimiento independiente.

### 2.2 TTL de Retención

```sql
-- TTL de 24 meses sobre event_ts
-- ClickHouse elimina las filas que superan el TTL en el próximo merge.
-- Para retención configurable por país, se puede crear una tabla por país
-- o usar TTL condicional (feature disponible desde ClickHouse 22.x):
ALTER TABLE vehicle_events_daily MODIFY TTL
    event_ts + toIntervalMonth(
        if(country_code = 'CO', 36, 24)  -- Colombia: 36 meses; resto: 24 meses
    ) DELETE;
```

---

## 3. Kafka Engine Table — `vehicle_events_kafka`

Ver configuración completa en [debezium-cdc.md](./debezium-cdc.md) §5. Resumen:

```sql
CREATE TABLE vehicle_events_kafka
(
    event_id            String,
    country_code        FixedString(2),
    device_id           String,
    plate_normalized    String,
    event_ts            DateTime64(3, 'UTC'),
    received_ts         DateTime64(3, 'UTC'),
    confidence          Float32,
    location_x          Float64,
    location_y          Float64,
    image_uri           Nullable(String),
    thumbnail_uri       Nullable(String),
    clock_uncertain     UInt8,
    image_unavailable   UInt8,
    enrichment_version  UInt8,
    __op                String,
    __source_lsn        String,
    __source_ts_ms      UInt64,
    cdc_version         String
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka-0:9092,kafka-1:9092,kafka-2:9092',
    kafka_topic_list = 'pg.events.cdc',
    kafka_group_name = 'clickhouse-cdc-consumer',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 4,
    kafka_skip_broken_messages = 10;
```

---

## 4. Vista Materializada de Ingestión

```sql
-- Convierte el payload CDC a las columnas de vehicle_events_daily
CREATE MATERIALIZED VIEW vehicle_events_cdc_mv
TO vehicle_events_daily
AS
SELECT
    toUUID(event_id)                            AS event_id,
    country_code,
    device_id,
    plate_normalized,
    toDateTime64(event_ts, 3, 'UTC')            AS event_ts,
    toDateTime64(received_ts, 3, 'UTC')         AS received_ts,
    toFloat32(confidence)                       AS confidence,
    toFloat64(location_y)                       AS lat,
    toFloat64(location_x)                       AS lon,
    geoToH3(location_x, location_y, 8)         AS h3_index_8,
    image_uri,
    thumbnail_uri,
    toUInt8(clock_uncertain)                   AS clock_uncertain,
    toUInt8(image_unavailable)                 AS image_unavailable,
    toUInt8(enrichment_version)                AS enrichment_version,
    __op                                        AS cdc_op,
    __source_lsn                                AS source_lsn,
    now()                                       AS ingested_at
FROM vehicle_events_kafka
WHERE __op IN ('c', 'r', 'u');
-- Los eventos de delete ('d') se ignoran en la vista principal;
-- se propagan implícitamente cuando ReplacingMergeTree deduplica por (event_id, country_code)
-- y la fila con __op='d' no reemplaza a las anteriores (no se inserta).
```

---

## 5. Índices Adicionales (Skip Indexes)

```sql
-- Índice de datos de salto para plate_normalized (bloom filter probabilístico)
-- Reduce el número de gránulos a leer en consultas por matrícula
ALTER TABLE vehicle_events_daily ADD INDEX idx_plate_bloom
    plate_normalized
    TYPE bloom_filter(0.01)
    GRANULARITY 4;

-- Materializar el índice en los datos existentes
ALTER TABLE vehicle_events_daily MATERIALIZE INDEX idx_plate_bloom;
```

---

## 6. Query de Ejemplo — Analítica

### 6.1 Avistamientos por hora en un día específico (dashboard operacional)

```sql
SELECT
    country_code,
    toStartOfHour(event_ts)    AS hour,
    count()                    AS event_count,
    uniq(plate_normalized)     AS unique_plates,
    avg(confidence)            AS avg_confidence
FROM vehicle_events_daily
WHERE
    country_code = 'CO'
    AND toDate(event_ts) = '2026-05-13'
GROUP BY country_code, hour
ORDER BY hour ASC;
```

### 6.2 Top 10 dispositivos con más eventos en la última semana

```sql
SELECT
    device_id,
    count()         AS total_events,
    uniq(plate_normalized) AS unique_plates
FROM vehicle_events_daily
WHERE
    country_code = 'CO'
    AND event_ts >= now() - INTERVAL 7 DAY
GROUP BY device_id
ORDER BY total_events DESC
LIMIT 10;
```

### 6.3 Distribución de confidence score (diagnóstico de calidad ANPR)

```sql
SELECT
    country_code,
    floor(confidence / 10) * 10   AS confidence_bucket,
    count()                        AS count,
    count() * 100.0 / sum(count()) OVER (PARTITION BY country_code) AS pct
FROM vehicle_events_daily
WHERE
    country_code IN ('CO', 'MX')
    AND event_ts >= now() - INTERVAL 30 DAY
GROUP BY country_code, confidence_bucket
ORDER BY country_code, confidence_bucket;
```

---

## 7. Gestión de Offsets Kafka y Resiliencia ante Reinicios (CR-05)

La Kafka Engine Table `vehicle_events_kafka` usa un **consumer group de Kafka** para gestionar los offsets. ClickHouse confirma (commit) el offset de cada mensaje **después de que la vista materializada lo insertó correctamente** en `vehicle_events_daily`.

### 7.1 Comportamiento ante Reinicio de ClickHouse

```
1. ClickHouse se reinicia (maintenance, OOM, crash).
2. La Kafka Engine Table se reconecta al broker con group_id = 'clickhouse-cdc-consumer'.
3. Kafka devuelve el último offset confirmado (committed offset).
4. ClickHouse reanuda el consumo desde ese offset: ningún mensaje se pierde.
5. Mensajes publicados en pg.events.cdc durante la indisponibilidad de ClickHouse
   permanecen en Kafka hasta que se consumen (retención configurada en el tópico).
```

**Por qué no se pierden mensajes (CR-05):**

- Los offsets se almacenan en Kafka, no en ClickHouse. Un reinicio de ClickHouse no afecta los offsets.
- La retención del tópico `pg.events.cdc` está configurada por tiempo (ej. 7 días) y por tamaño (ej. 50 GB), suficiente para absorber indisponibilidades de horas.
- Si `kafka_skip_broken_messages = 10`, hasta 10 mensajes malformados por batch se saltan; esto no produce pérdida de datos válidos (solo mensajes corruptos).

### 7.2 Monitoreo del Lag de Ingestión

```bash
# Ver el lag actual del consumer group de ClickHouse
kafka-consumer-groups.sh --bootstrap-server kafka-0:9092 \
  --group clickhouse-cdc-consumer \
  --describe

# Métrica Prometheus (exportada por Kafka Exporter):
kafka_consumer_group_lag{group="clickhouse-cdc-consumer",topic="pg.events.cdc"}

# Alerta: si el lag supera 300 s (5 min) de datos, se activa DebeziumHighLag
# (ver slo-observability.md §4)
```

### 7.3 Recuperación ante Lag Excesivo

Si ClickHouse estuvo caído más tiempo que la retención de Kafka, los mensajes se pierden irrecuperablemente desde Kafka. En ese caso se puede re-indexar desde el WAL de PostgreSQL vía un nuevo slot de replicación Debezium (procedure documentado en [debezium-cdc.md](./debezium-cdc.md) §9).

---

## 8. Operaciones de Mantenimiento

```sql
-- Ver el tamaño por partición
SELECT
    partition,
    formatReadableSize(sum(bytes_on_disk)) AS size_on_disk,
    sum(rows)                               AS total_rows,
    count()                                 AS parts_count
FROM system.parts
WHERE table = 'vehicle_events_daily' AND active = 1
GROUP BY partition
ORDER BY partition;

-- Eliminar manualmente una partición antigua (después del TTL)
ALTER TABLE vehicle_events_daily DROP PARTITION ('CO', 202401);

-- Optimizar (forzar merge) una partición en mantenimiento planificado
OPTIMIZE TABLE vehicle_events_daily PARTITION ('CO', 202602) FINAL;
```
