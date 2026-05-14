# Backbone de Procesamiento — Alert Service

**Componente:** backbone-procesamiento → Alert Service  
**Versión del documento:** 1.0  
**Última actualización:** 2026-05-13

---

## 1. Responsabilidad

El Alert Service es el cuarto componente del hot path. Consume eventos del tópico `vehicle.alerts` (producidos por el Matcher Service) y:

1. **Deduplica alertas** por placa y zona en una ventana de 5 minutos para evitar notificar múltiples veces al mismo oficial por el mismo vehículo.
2. **Persiste la alerta** en la tabla `alerts` de PostgreSQL con estado `ACTIVE`.
3. **Determina la zona** de la alerta usando consultas geoespaciales PostGIS sobre las geocercas configuradas.
4. **Enruta la alerta** al WebSocket Gateway con el identificador de zona (`zone_id`).
5. **Gestiona alertas sin agentes activos:** persiste la alerta para que el WebSocket Gateway la entregue en el momento de reconexión.

---

## 2. Esquema de la Tabla `alerts`

La tabla `alerts` en PostgreSQL almacena el estado de vida de cada alerta. Ver también [postgresql-schema.md](./postgresql-schema.md) para el DDL completo.

```sql
CREATE TABLE alerts (
    alert_id          UUID          NOT NULL DEFAULT gen_random_uuid(),
    event_id          UUID          NOT NULL,
    country_code      CHAR(2)       NOT NULL,
    plate_normalized  TEXT          NOT NULL,
    device_id         TEXT          NOT NULL,
    zone_id           TEXT,                     -- NULL si no se pudo determinar la zona
    location          GEOGRAPHY(POINT, 4326) NOT NULL,
    alert_ts          TIMESTAMPTZ   NOT NULL,
    event_ts          TIMESTAMPTZ   NOT NULL,
    state             TEXT          NOT NULL DEFAULT 'ACTIVE',
                                               -- ACTIVE | DELIVERED | EXPIRED | SUPPRESSED
    image_uri         TEXT,
    thumbnail_uri     TEXT,
    confidence        NUMERIC(5,2),
    via_fallback      BOOLEAN       NOT NULL DEFAULT FALSE,
    stolen_vehicle    JSONB,                    -- Snapshot del vehículo hurtado
    delivered_at      TIMESTAMPTZ,              -- Timestamp de entrega al primer agente
    expires_at        TIMESTAMPTZ   NOT NULL,   -- alert_ts + retención configurada (default 24h)
    created_at        TIMESTAMPTZ   NOT NULL DEFAULT now(),

    PRIMARY KEY (alert_id, country_code),
    FOREIGN KEY (event_id, country_code) REFERENCES vehicle_events(event_id, country_code)
) PARTITION BY LIST (country_code);

-- Índices relevantes
CREATE INDEX idx_alerts_country_state    ON alerts (country_code, state)      WHERE state = 'ACTIVE';
CREATE INDEX idx_alerts_plate_country    ON alerts (plate_normalized, country_code, alert_ts DESC);
CREATE INDEX idx_alerts_zone_state       ON alerts (zone_id, state)            WHERE state = 'ACTIVE';
CREATE INDEX idx_alerts_location         ON alerts USING GIST (location);
CREATE INDEX idx_alerts_expires          ON alerts (expires_at)                WHERE state = 'ACTIVE';
```

**Estados de una alerta:**

| Estado | Descripción |
|---|---|
| `ACTIVE` | Alerta generada y pendiente de entrega o confirmación. |
| `DELIVERED` | Alerta entregada a al menos un agente activo en la zona. |
| `EXPIRED` | La alerta superó `expires_at` sin ser entregada (proceso de cleanup periódico). |
| `SUPPRESSED` | Alerta descartada por la deduplicación de ventana de 5 min. |

---

## 3. Algoritmo de Deduplicación por Ventana de 5 Minutos

La deduplicación impide que el mismo vehículo genere múltiples notificaciones a los agentes cuando pasa por varios dispositivos en un corto periodo de tiempo.

**Clave de deduplicación:** `country_code + ":" + plate_normalized + ":" + zone_id`

**Ventana:** 5 minutos desde la primera alerta por esa clave.

```
PARA cada mensaje m en vehicle.alerts:
    zone_id := determinarZona(m.location, m.country_code)
    clave_dedup := m.country_code + ":" + m.plate_normalized + ":" + zone_id

    # Buscar alerta activa reciente para la misma clave (ventana 5 min)
    existente := SELECT alert_id FROM alerts
                 WHERE country_code     = m.country_code
                   AND plate_normalized = m.plate_normalized
                   AND zone_id          = zone_id
                   AND state IN ('ACTIVE', 'DELIVERED')
                   AND alert_ts >= now() - INTERVAL '5 minutes'
                 LIMIT 1

    SI existente NO ES nulo:
        # Alerta duplicada — descartar sin notificar
        INSERT INTO alerts (..., state='SUPPRESSED') VALUES (...)
        incrementar contador alert_deduplication_suppressed
        CONTINUAR

    # Primera alerta para esta clave en la ventana → persiste y enruta
    INSERT INTO alerts (..., state='ACTIVE') VALUES (...)
    enviarAlWebSocketGateway(alert, zone_id)
```

> La consulta de deduplicación usa el índice `idx_alerts_zone_state` (combinado con el filtro `alert_ts >= now() - 5 min`) para garantizar latencia < 10 ms.

---

## 4. Determinación de Zona con PostGIS

El Alert Service determina el `zone_id` (geocerca) de la alerta consultando la tabla de zonas geoespaciales:

```sql
-- Tabla de zonas (definida en almacenamiento-lectura)
CREATE TABLE alert_zones (
    zone_id       TEXT        NOT NULL,
    country_code  CHAR(2)     NOT NULL,
    zone_name     TEXT        NOT NULL,
    boundary      GEOGRAPHY(POLYGON, 4326) NOT NULL,
    active        BOOLEAN     NOT NULL DEFAULT TRUE,
    PRIMARY KEY (zone_id, country_code)
);

CREATE INDEX idx_alert_zones_boundary ON alert_zones USING GIST (boundary);
CREATE INDEX idx_alert_zones_country  ON alert_zones (country_code) WHERE active = TRUE;
```

**Query de determinación de zona:**
```sql
SELECT zone_id
FROM alert_zones
WHERE country_code = $1
  AND active = TRUE
  AND ST_Within(
        ST_MakePoint($3, $2)::geography::geometry,
        boundary::geometry
      )
LIMIT 1;
```

Si la consulta retorna vacío (la ubicación del evento no está dentro de ninguna geocerca configurada), el Alert Service asigna `zone_id = NULL` y la alerta se enruta a todos los agentes del `country_code` (fallback de última instancia).

---

## 5. Enrutamiento al WebSocket Gateway

El Alert Service envía la alerta al WebSocket Gateway mediante una llamada gRPC interna (o HTTP/2 sobre la red del cluster Kubernetes). El Gateway es responsable de filtrar las sesiones activas por `country_code` y `zone_id`.

**Interface de llamada al WebSocket Gateway:**
```go
// AlertsGatewayPort — enrutamiento de alertas al WebSocket Gateway
type AlertsGatewayPort interface {
    // SendAlert entrega la alerta al Gateway para distribución.
    // Retorna nil si la llamada fue aceptada (no garantiza entrega al agente).
    SendAlert(ctx context.Context, alert Alert) error
}
```

**Timeout de la llamada al Gateway:** 200 ms. Si el Gateway no responde en ese tiempo, el Alert Service registra el fallo y continúa (la alerta ya está persistida en PostgreSQL con estado `ACTIVE`; el Gateway la recuperará en la próxima reconexión del agente).

---

## 6. Alertas sin Agentes Activos (CR-05)

Si el WebSocket Gateway informa que no hay agentes activos en la zona de la alerta (o si el Gateway no está disponible):
- La alerta se persiste en PostgreSQL con estado `ACTIVE`.
- Se incrementa la métrica `alert_delivery_no_active_agents_total`.
- No se genera error ni excepción en el Alert Service.
- Cuando un agente se conecta al WebSocket Gateway, el Gateway consulta las alertas `ACTIVE` pendientes para su zona y las entrega en el momento de la reconexión (ver [websocket-gateway.md](./websocket-gateway.md)).

---

## 7. Flujo de Secuencia Completo

```mermaid
sequenceDiagram
    autonumber
    participant K4   as vehicle.alerts
    participant ALS  as Alert Service
    participant PG   as PostgreSQL
    participant WS   as WebSocket Gateway

    K4   ->> ALS: VehicleAlert (Avro)
    ALS  ->> PG:  SELECT zone_id WHERE ST_Within(location, boundary) [< 10 ms]
    PG   -->> ALS: zone_id (o NULL)
    ALS  ->> PG:  SELECT existente WHERE plate + zone + 5min [idx]
    alt Duplicado en ventana 5 min
        PG   -->> ALS: alert_id existente
        ALS  ->> PG:  INSERT alerts state=SUPPRESSED
        ALS  ->> ALS: incrementar alert_deduplication_suppressed
    else Primera alerta en ventana
        PG   -->> ALS: vacío
        ALS  ->> PG:  INSERT alerts state=ACTIVE
        ALS  ->> WS:  SendAlert(alert, zone_id) [timeout 200ms]
        alt Gateway responde — agentes activos
            WS   -->> ALS: OK (agentes notificados)
            ALS  ->> PG:  UPDATE alerts SET state=DELIVERED
        else Gateway responde — sin agentes activos
            WS   -->> ALS: NO_AGENTS
            ALS  ->> PG:  alerta queda ACTIVE (para reconexión)
            ALS  ->> ALS: incrementar alert_delivery_no_active_agents
        else Gateway no disponible (timeout)
            WS   -->> ALS: timeout
            ALS  ->> ALS: registrar fallo (alerta ACTIVE para reconexión)
        end
    end
    ALS  ->> K4:  confirmar offset
```

---

## 8. Métricas Prometheus

| Métrica | Tipo | Descripción |
|---|---|---|
| `alert_events_processed_total` | Counter | Total de eventos de alerta procesados. |
| `alert_persisted_total` | Counter | Alertas persistidas en PostgreSQL (ACTIVE + SUPPRESSED). |
| `alert_active_total` | Counter | Alertas persistidas con estado ACTIVE. |
| `alert_deduplication_suppressed_total` | Counter | Alertas descartadas por deduplicación de ventana 5 min (CA-12). |
| `alert_delivery_no_active_agents_total` | Counter | Alertas donde no había agentes activos en la zona (CR-05). |
| `alert_gateway_send_duration_seconds` | Histogram | Latencia de la llamada al WebSocket Gateway. Objetivo p95 < 200 ms. |
| `alert_gateway_errors_total` | Counter | Fallos de comunicación con el WebSocket Gateway. |
| `alert_zone_resolution_duration_seconds` | Histogram | Latencia de la consulta PostGIS de zona. Objetivo p95 < 10 ms. |
| `alert_zone_not_found_total` | Counter | Alertas cuya ubicación no corresponde a ninguna geocerca configurada. |
| `alert_processing_duration_seconds` | Histogram | Latencia total (consume → persistir → enviar al Gateway). Objetivo p95 < 200 ms. |
| `alert_kafka_consumer_lag` | Gauge | Lag del consumer group `alert-service-cg` en `vehicle.alerts`. |

---

## 9. Configuración del Servicio

```yaml
KAFKA_BOOTSTRAP_SERVERS: "kafka-broker-1:9092,kafka-broker-2:9092,kafka-broker-3:9092"
KAFKA_GROUP_ID: "alert-service-cg"
KAFKA_INPUT_TOPIC: "vehicle.alerts"
SCHEMA_REGISTRY_URL: "http://schema-registry:8081"

POSTGRES_DSN: "postgresql://app_writer:${POSTGRES_PASSWORD}@postgres-primary:5432/antihurto"
POSTGRES_MAX_CONNECTIONS: "20"

WEBSOCKET_GATEWAY_GRPC_ADDR: "websocket-gateway:9001"
WEBSOCKET_GATEWAY_TIMEOUT_MS: "200"

ALERT_DEDUP_WINDOW_MINUTES: "5"
ALERT_EXPIRY_HOURS: "24"

METRICS_PORT: "9090"
HEALTH_PORT: "8080"
```

---

## 10. Criterios de Aceptación Cubiertos

| CA/CR | Verificación |
|---|---|
| CA-11: Alerta persistida y enviada al Gateway | INSERT en `alerts` con `state=ACTIVE`; llamada al WebSocket Gateway dentro del timeout. |
| CA-12: Deduplicación por ventana 5 min | Segunda alerta para misma placa + zona en < 5 min → `state=SUPPRESSED`; métrica `alert_deduplication_suppressed_total`. |
| CA-13: Entrega a agente en zona correcta | Filtro por `zone_id` desde la consulta PostGIS; agentes de otras zonas no reciben la alerta. |
| CA-14: SLO p95 < 2 s | Presupuesto Alert Service: 200 ms. Medido por `alert_processing_duration_seconds`. |
| CR-05: Sin agentes activos — alerta persiste | Estado `ACTIVE` en PostgreSQL; métrica `alert_delivery_no_active_agents_total`; sin error. |
