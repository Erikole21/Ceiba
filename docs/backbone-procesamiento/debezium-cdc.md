# Backbone de Procesamiento — Debezium CDC (Cold Path)

**Componente:** backbone-procesamiento → Debezium CDC  
**Versión del documento:** 1.0  
**Última actualización:** 2026-05-13

---

## 1. Responsabilidad

El conector Debezium es el componente del cold path que captura los cambios del WAL (Write-Ahead Log) de PostgreSQL y los publica al tópico `pg.events.cdc`. Este tópico es consumido por ClickHouse vía Kafka Engine Table para alimentar la capa analítica del sistema.

**Garantías:**
- Captura los cambios en la tabla `vehicle_events` dentro de los **30 segundos** siguientes a la inserción (CA-17).
- No impacta el hot path: opera sobre el slot de replicación de PostgreSQL sin bloquear las escrituras operacionales.
- El lag del conector es monitoreado; si supera 60 s se activa una alerta operacional (CR-08).
- Un lag elevado de Debezium no afecta la disponibilidad del hot path ni del pipeline de alertas.

---

## 2. Tablas Monitoreadas

| Tabla | Operaciones capturadas | Particionada por |
|---|---|---|
| `vehicle_events` | INSERT | `country_code` |
| `stolen_vehicles` | INSERT, UPDATE, DELETE | `country_code` |
| `alerts` | INSERT, UPDATE | `country_code` |
| `incidents` | INSERT, UPDATE | `country_code` |

> La tabla `incident_audit` no se captura por Debezium: es append-only y su contenido se consulta directamente desde PostgreSQL. ClickHouse no necesita los registros de auditoría para analítica.

---

## 3. Configuración del Conector Kafka Connect

El conector Debezium se configura como un **Kafka Connect Source Connector** (modo standalone o distribuido). Se usa la distribución distribuida (Kafka Connect cluster) en producción por HA.

```json
{
  "name": "debezium-pg-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-primary",
    "database.port": "5432",
    "database.user": "debezium_user",
    "database.password": "${DEBEZIUM_DB_PASSWORD}",
    "database.dbname": "antihurto",
    "database.server.name": "antihurto-pg",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot_events",
    "publication.name": "debezium_publication",
    "table.include.list": "public.vehicle_events,public.stolen_vehicles,public.alerts,public.incidents",
    "heartbeat.interval.ms": "10000",
    "heartbeat.action.query": "UPDATE debezium_heartbeat SET last_heartbeat = now() WHERE id = 1",
    "topic.prefix": "pg",
    "topic.naming.strategy": "io.debezium.schema.DefaultTopicNamingStrategy",
    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.route.renames": "after.country_code:country_code",
    "key.converter": "io.confluent.connect.avro.AvroConverter",
    "key.converter.schema.registry.url": "http://schema-registry:8081",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "producer.override.acks": "all",
    "producer.override.enable.idempotence": "true",
    "snapshot.mode": "initial",
    "decimal.handling.mode": "precise",
    "time.precision.mode": "adaptive_time_microseconds",
    "tombstones.on.delete": "true",
    "max.batch.size": "2048",
    "max.queue.size": "16384",
    "poll.interval.ms": "100"
  }
}
```

### 3.1 Rol de Base de Datos para Debezium

```sql
-- Crear rol con privilegios mínimos necesarios para replicación
CREATE ROLE debezium_user WITH LOGIN REPLICATION PASSWORD 'gestión_via_vault';
GRANT CONNECT ON DATABASE antihurto TO debezium_user;
GRANT USAGE ON SCHEMA public TO debezium_user;
GRANT SELECT ON TABLE vehicle_events, stolen_vehicles, alerts, incidents TO debezium_user;

-- Publicación lógica de PostgreSQL
CREATE PUBLICATION debezium_publication FOR TABLE
    vehicle_events, stolen_vehicles, alerts, incidents;

-- Tabla de heartbeat (necesaria para el monitoreo de lag cuando no hay actividad en las tablas)
CREATE TABLE debezium_heartbeat (
    id            INT PRIMARY KEY DEFAULT 1,
    last_heartbeat TIMESTAMPTZ NOT NULL DEFAULT now()
);
INSERT INTO debezium_heartbeat VALUES (1, now());
GRANT UPDATE ON debezium_heartbeat TO debezium_user;
```

### 3.2 Parámetros de PostgreSQL requeridos

```sql
-- postgresql.conf
wal_level = logical
max_replication_slots = 5       -- al menos 1 por conector + margen para DR
max_wal_senders = 10
wal_sender_timeout = 60000      -- 60 s
```

---

## 4. Formato CDC — Envelope Debezium

Cada mensaje en `pg.events.cdc` sigue el formato estándar del envelope Debezium:

```json
{
  "schema": { ... },
  "payload": {
    "before": null,
    "after": {
      "event_id":          "550e8400-e29b-41d4-a716-446655440000",
      "country_code":      "CO",
      "device_id":         "dev-co-001",
      "plate_raw":         "abc 123-x",
      "plate_normalized":  "ABC123X",
      "event_ts":          1747144200000000,
      "received_ts":       1747144200200000,
      "confidence":        97.50,
      "location":          "0101000020E6100000AE47E17A14E25240A4703D0A57C14140",
      "image_uri":         "s3://antihurto-co/events/2026/05/13/550e8400.jpg",
      "thumbnail_uri":     "s3://antihurto-co/events/2026/05/13/550e8400_thumb.jpg",
      "clock_uncertain":   false,
      "image_unavailable": false,
      "enrichment_version": 1,
      "extensions":        "{}"
    },
    "source": {
      "version":   "2.5.0.Final",
      "connector": "postgresql",
      "name":      "antihurto-pg",
      "ts_ms":     1747144201234,
      "snapshot":  "false",
      "db":        "antihurto",
      "sequence":  "[null,\"12345678\"]",
      "schema":    "public",
      "table":     "vehicle_events",
      "txId":      789012,
      "lsn":       12345678,
      "xmin":      null
    },
    "op": "c",
    "ts_ms": 1747144201234
  }
}
```

**Valores del campo `op`:**
- `c` — CREATE (INSERT)
- `u` — UPDATE
- `d` — DELETE
- `r` — READ (snapshot inicial)

---

## 5. ClickHouse Kafka Engine Table

ClickHouse consume el tópico `pg.events.cdc` mediante una Kafka Engine Table y una vista materializada que inserta en la tabla de almacenamiento final.

```sql
-- Tabla de ingestión Kafka Engine (efímera)
CREATE TABLE vehicle_events_kafka_queue (
    event_id          String,
    country_code      FixedString(2),
    device_id         String,
    plate_raw         String,
    plate_normalized  String,
    event_ts          DateTime64(6, 'UTC'),
    received_ts       DateTime64(6, 'UTC'),
    confidence        Float64,
    location_wkb      String,           -- WKB hexadecimal de PostGIS
    clock_uncertain   Bool,
    image_uri         String,
    thumbnail_uri     String,
    enrichment_version UInt8
) ENGINE = Kafka
SETTINGS
    kafka_broker_list    = 'kafka-broker-1:9092,kafka-broker-2:9092,kafka-broker-3:9092',
    kafka_topic_list     = 'pg.events.cdc',
    kafka_group_name     = 'clickhouse-cdc-cg',
    kafka_format         = 'AvroConfluent',
    kafka_schema_registry_url = 'http://schema-registry:8081',
    kafka_num_consumers  = 4,
    kafka_max_block_size = 65536;

-- Vista materializada que mueve datos a la tabla de almacenamiento
CREATE MATERIALIZED VIEW vehicle_events_cdc_mv TO vehicle_events AS
SELECT
    event_id,
    country_code,
    device_id,
    plate_raw,
    plate_normalized,
    event_ts,
    received_ts,
    confidence,
    -- Convertir WKB hex a coordenadas lat/lon (función UDF o via processing en ClickHouse)
    0.0  AS lat,
    0.0  AS lon,
    clock_uncertain,
    image_uri,
    thumbnail_uri,
    enrichment_version,
    now() AS ingested_at
FROM vehicle_events_kafka_queue
WHERE JSONExtractString(payload, 'op') IN ('c', 'r')  -- solo inserts y snapshots
  AND JSONExtractString(payload, 'after') IS NOT NULL;
```

> La conversión de coordenadas WKB PostGIS → lat/lon en ClickHouse se implementa mediante una función UDF en Python o mediante un proceso de transformación intermedio. El campo `location` de PostGIS se deserializa en el Enrichment Service y se almacena como WKB hexadecimal en el mensaje CDC.

---

## 6. Lag Esperado y SLO del Cold Path

| Etapa | Lag esperado p95 | Notas |
|---|---|---|
| WAL PostgreSQL → Debezium | < 5 s | Configuración `poll.interval.ms=100`, heartbeat activo |
| Debezium → `pg.events.cdc` | < 5 s | Bajo carga normal (< 1 000 eventos/seg) |
| `pg.events.cdc` → ClickHouse (Kafka Engine) | < 20 s | Depende del `max_block_size` de la Kafka Engine Table |
| **Total cold path** | **< 30 s** | CA-17 |

**Umbral de alerta operacional (CR-08):**
- Si el lag de Debezium supera **60 segundos** (medido como diferencia entre el LSN actual de PostgreSQL y el último LSN procesado por el conector): alerta `debezium_connector_lag_high`.
- El hot path continúa operando con normalidad; el lag del cold path no lo afecta.

---

## 7. Monitoreo y Dead-Letter

### 7.1 Métricas JMX de Kafka Connect

Debezium expone métricas JMX que se recolectan con `jmx_exporter` de Prometheus:

| Métrica JMX | Métrica Prometheus | Descripción |
|---|---|---|
| `debezium.postgres.connector.metrics:MilliSecondsBehindSource` | `debezium_connector_lag_ms` | Lag del conector en milisegundos. Alerta si > 60 000. |
| `debezium.postgres.connector.metrics:NumberOfEventsFiltered` | `debezium_events_filtered_total` | Eventos filtrados (tablas no monitoreadas). |
| `debezium.postgres.connector.metrics:TotalNumberOfEventsSeen` | `debezium_events_seen_total` | Total de eventos procesados por el conector. |
| `debezium.postgres.connector.metrics:NumberOfSkippedEvents` | `debezium_events_skipped_total` | Eventos omitidos (errores). Alerta si > 0. |
| `debezium.postgres.connector.metrics:SourceEventPosition` | `debezium_lsn_current` | LSN actual procesado por el conector. |

### 7.2 Dead-Letter Topic — `pg.events.cdc.dlq`

Si el conector Debezium encuentra un error al procesar o serializar un evento WAL:
- El mensaje se envía a `pg.events.cdc.dlq` con los headers de diagnóstico (causa del error, tabla de origen, LSN).
- El conector continúa procesando eventos subsecuentes (configuración `errors.tolerance=all`).
- Se incrementa la métrica `debezium_events_skipped_total`.
- Si la tasa de mensajes en el DLQ supera el 0.1 % en 5 minutos, se activa la alerta `debezium_dlq_rate_high`.

```json
{
  "name": "debezium-pg-connector",
  "config": {
    "...": "...",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "pg.events.cdc.dlq",
    "errors.deadletterqueue.topic.replication.factor": "3",
    "errors.deadletterqueue.context.headers.enable": "true"
  }
}
```

### 7.3 Alerta de Lag Elevado (CR-08)

```yaml
# Regla Prometheus para alerta de lag Debezium
- alert: DebeziumConnectorLagHigh
  expr: debezium_connector_lag_ms > 60000
  for: 2m
  labels:
    severity: warning
    component: debezium-cdc
  annotations:
    summary: "Debezium connector lag excede 60 segundos"
    description: >
      El lag del conector Debezium es {{ $value | humanizeDuration }}.
      ClickHouse puede estar recibiendo datos con retraso.
      El hot path NO está afectado.
    runbook_url: "https://wiki.antihurto/runbooks/debezium-lag"
```

---

## 8. Verificación Post-Despliegue (CA-17)

Para verificar que el conector funciona correctamente tras el despliegue:

```bash
# 1. Verificar estado del conector
curl http://kafka-connect:8083/connectors/debezium-pg-connector/status

# 2. Insertar un evento de prueba en PostgreSQL y verificar que aparece en pg.events.cdc
kafka-console-consumer \
  --bootstrap-server kafka-broker-1:9092 \
  --topic pg.events.cdc \
  --from-beginning \
  --max-messages 1

# 3. Verificar lag del consumer group de ClickHouse
kafka-consumer-groups \
  --bootstrap-server kafka-broker-1:9092 \
  --describe \
  --group clickhouse-cdc-cg
```

---

## 9. Coordinación con Almacenamiento de Lectura

El conector Debezium del backbone-procesamiento se coordina con el módulo `almacenamiento-lectura`. En particular:
- El slot de replicación `debezium_slot_events` es compartido con el conector de replicación lógica para DR.
- El schema DDL de las tablas monitoreadas se define en [docs/almacenamiento-lectura/postgresql-schema.md](../almacenamiento-lectura/postgresql-schema.md) como fuente de verdad.
- El schema de ClickHouse (`vehicle_events` en ClickHouse) se define en [docs/almacenamiento-lectura/clickhouse-schema.md](../almacenamiento-lectura/clickhouse-schema.md).

---

## 10. Criterios de Aceptación Cubiertos

| CA/CR | Verificación |
|---|---|
| CA-17: Debezium publica cambios del WAL en < 30 s | `debezium_connector_lag_ms` < 30 000 bajo carga normal; mensaje visible en `pg.events.cdc` dentro de 30 s. |
| CR-08: Lag > 60 s activa alerta operacional | Alerta `DebeziumConnectorLagHigh` disparada al superar 60 000 ms por > 2 min; ClickHouse continúa sin pérdida de datos. |
