# Backbone de Procesamiento — SLOs y Observabilidad

**Componente:** backbone-procesamiento  
**Versión del documento:** 1.0  
**Última actualización:** 2026-05-13

---

## 1. SLO Principal — Hot Path p95 < 2 s

### 1.1 Definición

| SLO | Valor objetivo | Ventana de medición | Métrica primaria |
|---|---|---|---|
| Latencia end-to-end p95 | < 2 000 ms | 5 minutos (producción), 1 min (pre-prod) | `hot_path_e2e_latency_seconds` p95 |
| Latencia end-to-end p99 | < 5 000 ms | 5 minutos | `hot_path_e2e_latency_seconds` p99 |
| Disponibilidad del pipeline | ≥ 99.9 % | 30 días | `hot_path_availability_ratio` |

**Definición de latencia end-to-end:** tiempo desde `event_ts` del evento crudo (timestamp de captura en el dispositivo de borde) hasta el timestamp de entrega de la alerta al WebSocket Gateway.

### 1.2 Presupuesto de Error por Componente

La suma de los presupuestos p95 por etapa debe ser ≤ 2 000 ms. Distribución detallada:

| Etapa | Componente | Presupuesto p95 | Presupuesto p99 | Métrica |
|---|---|---|---|---|
| Red Agente → EMQX | Red GSM/LAN | ≤ 150 ms | ≤ 500 ms | `mqtt_publish_latency_seconds` |
| Bridge EMQX → Kafka | EMQX Rule Engine | ≤ 50 ms | ≤ 200 ms | `emqx_kafka_bridge_latency_seconds` |
| Deduplicación | Deduplicator | ≤ 100 ms | ≤ 300 ms | `deduplicator_processing_duration_seconds` |
| Enriquecimiento | Enrichment Service | ≤ 300 ms | ≤ 800 ms | `enrichment_processing_duration_seconds` |
| Cotejo Redis | Matcher Service | ≤ 50 ms | ≤ 150 ms | `matcher_redis_duration_seconds` |
| Procesamiento Matcher | Matcher Service | ≤ 100 ms | ≤ 300 ms | `matcher_processing_duration_seconds` |
| Alert Service | Alert Service | ≤ 200 ms | ≤ 500 ms | `alert_processing_duration_seconds` |
| Entrega WebSocket | WebSocket Gateway | ≤ 100 ms | ≤ 250 ms | `ws_delivery_duration_seconds` |
| **Margen de seguridad** | — | **950 ms** | — | — |
| **Total** | | **≤ 2 000 ms** | **≤ 4 000 ms** | |

> El margen de 950 ms acomoda jitter de red, GC pauses, contención en particiones Kafka y variabilidad entre países. Si el p95 supera 1 500 ms de forma sostenida, el SRE debe investigar el componente con mayor contribución usando el dashboard `backbone-slo-budget`.

### 1.3 Cálculo de la Métrica `hot_path_e2e_latency_seconds`

La métrica end-to-end se calcula en el WebSocket Gateway como:

```
hot_path_e2e_latency_seconds = ws_delivery_timestamp - event_ts
```

Donde `event_ts` es el timestamp de captura extraído del payload de la alerta (propagado desde el evento original a través de todo el pipeline).

```yaml
# PromQL para calcular el p95 de la latencia end-to-end
histogram_quantile(0.95,
  rate(hot_path_e2e_latency_seconds_bucket[5m])
)
```

---

## 2. Métricas Prometheus por Componente

### 2.1 Deduplicator

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `deduplicator_events_processed_total` | Counter | `country_code` | Total de eventos procesados. |
| `deduplicator_duplicates_discarded_total` | Counter | `country_code` | Duplicados descartados. |
| `deduplicator_events_forwarded_total` | Counter | `country_code` | Eventos únicos producidos a `vehicle.events.deduped`. |
| `deduplicator_dlq_messages_total` | Counter | `country_code`, `reason` | Mensajes al DLQ. |
| `deduplicator_processing_duration_seconds` | Histogram | `country_code` | Latencia de procesamiento. |
| `deduplicator_state_store_size_bytes` | Gauge | — | Tamaño del state store RocksDB. |
| `deduplicator_kafka_consumer_lag` | Gauge | `country_code`, `partition` | Lag del consumer group. |

### 2.2 Enrichment Service

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `enrichment_events_processed_total` | Counter | `country_code` | Total procesados. |
| `enrichment_events_forwarded_total` | Counter | `country_code` | Producidos a `vehicle.events.enriched`. |
| `enrichment_dlq_messages_total` | Counter | `country_code`, `reason` | Mensajes al DLQ. |
| `enrichment_processing_duration_seconds` | Histogram | `country_code` | Latencia total. |
| `enrichment_geocoding_duration_seconds` | Histogram | `country_code` | Latencia de geocodificación. |
| `enrichment_geocoding_failures_total` | Counter | `country_code`, `status` | Fallos de geocodificación (`FAILED`, `INVALID_COORDINATES`). |
| `enrichment_plate_normalization_errors_total` | Counter | `country_code` | Errores de normalización. |
| `enrichment_image_blur_detected_total` | Counter | `country_code` | Imágenes con `blur_flag=true`. |
| `enrichment_postgres_write_duration_seconds` | Histogram | `country_code` | Latencia de escritura PostgreSQL. |
| `enrichment_postgres_write_errors_total` | Counter | `country_code` | Errores de escritura PostgreSQL. |
| `enrichment_opensearch_index_errors_total` | Counter | `country_code` | Errores de indexación OpenSearch. |
| `enrichment_kafka_consumer_lag` | Gauge | `country_code`, `partition` | Lag del consumer group. |

### 2.3 Matcher Service

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `matcher_events_processed_total` | Counter | `country_code` | Total procesados. |
| `matcher_hits_total` | Counter | `country_code` | Matches positivos. |
| `matcher_misses_total` | Counter | `country_code` | Misses. |
| `matcher_fallback_used_total` | Counter | `country_code` | Eventos con fallback a estado local. |
| `matcher_fallback_hits_total` | Counter | `country_code` | Hits en estado local. |
| `matcher_redis_duration_seconds` | Histogram | `country_code` | Latencia Redis. |
| `matcher_redis_errors_total` | Counter | `country_code` | Errores Redis. |
| `matcher_redis_timeouts_total` | Counter | `country_code` | Timeouts Redis. |
| `matcher_processing_duration_seconds` | Histogram | `country_code` | Latencia total. |
| `matcher_kafka_consumer_lag` | Gauge | `country_code`, `partition` | Lag del consumer group. |

### 2.4 Alert Service

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `alert_events_processed_total` | Counter | `country_code` | Total procesados. |
| `alert_active_total` | Counter | `country_code` | Alertas ACTIVE persistidas. |
| `alert_deduplication_suppressed_total` | Counter | `country_code` | Alertas suprimidas por ventana 5 min. |
| `alert_delivery_no_active_agents_total` | Counter | `country_code` | Sin agentes activos en la zona. |
| `alert_gateway_send_duration_seconds` | Histogram | `country_code` | Latencia de envío al Gateway. |
| `alert_gateway_errors_total` | Counter | `country_code` | Fallos de comunicación con Gateway. |
| `alert_processing_duration_seconds` | Histogram | `country_code` | Latencia total. |
| `alert_kafka_consumer_lag` | Gauge | `country_code`, `partition` | Lag del consumer group. |

### 2.5 WebSocket Gateway

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `ws_active_sessions` | Gauge | `country_code` | Sesiones activas. |
| `ws_alerts_delivered_total` | Counter | `country_code` | Alertas entregadas (tiempo real + reconexión). |
| `ws_alerts_no_active_agents_total` | Counter | `country_code` | Sin agentes activos. |
| `ws_delivery_duration_seconds` | Histogram | `country_code` | Latencia de entrega al WebSocket. |
| `ws_auth_failures_total` | Counter | `country_code` | Fallos de autenticación JWT. |
| `hot_path_e2e_latency_seconds` | Histogram | `country_code` | **Latencia end-to-end** (métrica primaria del SLO). |

### 2.6 Debezium CDC

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `debezium_connector_lag_ms` | Gauge | `table` | Lag del conector en milisegundos. |
| `debezium_events_seen_total` | Counter | `table`, `op` | Eventos procesados por tabla y operación. |
| `debezium_events_skipped_total` | Counter | `table` | Eventos omitidos (errores). |
| `debezium_lsn_current` | Gauge | — | LSN actual procesado. |

---

## 3. Dashboards Grafana Recomendados

### 3.1 Dashboard: `backbone-slo-budget`

**Propósito:** monitoreo principal del SLO p95 < 2 s y distribución del presupuesto de latencia.

**Paneles sugeridos:**
- **Panel 1:** Gauge — Latencia end-to-end p95 actual vs. objetivo 2 000 ms.
- **Panel 2:** Gauge — Latencia end-to-end p99 actual vs. objetivo 5 000 ms.
- **Panel 3:** Heatmap — Distribución de latencia end-to-end por hora de los últimos 7 días.
- **Panel 4:** Stacked bar — Contribución de cada componente al p95 en los últimos 15 minutos.
- **Panel 5:** Timeseries — Error budget burn rate (fracción del presupuesto de error consumido).
- **Panel 6:** Stat — Eventos procesados/seg por country_code.

```promql
# Panel 1: p95 end-to-end
histogram_quantile(0.95,
  sum(rate(hot_path_e2e_latency_seconds_bucket[5m])) by (le)
)

# Panel 4: Contribución Enrichment Service
histogram_quantile(0.95,
  sum(rate(enrichment_processing_duration_seconds_bucket[5m])) by (le)
)
```

### 3.2 Dashboard: `backbone-pipeline-health`

**Propósito:** salud operacional del pipeline (throughput, errores, lags).

**Paneles sugeridos:**
- Throughput por tópico Kafka (mensajes/seg).
- Lag por consumer group (Deduplicator, Enrichment, Matcher, Alert Service).
- Tasa de errores por componente.
- DLQ rate (mensajes/min en cada dead-letter topic).
- Tasa de duplicados descartados por el Deduplicator.
- Tasa de geocodificación fallida.
- Tasa de uso del fallback Redis en el Matcher.
- Tasa de alertas suprimidas por deduplicación de 5 min.

### 3.3 Dashboard: `backbone-websocket`

**Propósito:** monitoreo de sesiones WebSocket y entrega de alertas.

**Paneles sugeridos:**
- Sesiones activas por `country_code` y `zone_id`.
- Tasa de alertas entregadas vs. sin agentes activos.
- Latencia de entrega al WebSocket (p50, p95, p99).
- Tasa de fallos de autenticación JWT.
- Número de reconexiones por hora.
- Alertas pendientes no entregadas (estado ACTIVE con > 5 min de antigüedad).

### 3.4 Dashboard: `backbone-debezium-cold-path`

**Propósito:** monitoreo del cold path (Debezium + ClickHouse).

**Paneles sugeridos:**
- Lag del conector Debezium por tabla (ms).
- Tasa de eventos CDC procesados por tabla.
- Lag del consumer group de ClickHouse (`clickhouse-cdc-cg`).
- Mensajes en `pg.events.cdc.dlq`.
- Últimos LSN procesados por Debezium.

---

## 4. Alertas Operacionales

### 4.1 Alertas de Latencia del Hot Path

```yaml
- alert: HotPathP95Above1500ms
  expr: |
    histogram_quantile(0.95,
      sum(rate(hot_path_e2e_latency_seconds_bucket[5m])) by (le)
    ) > 1.5
  for: 3m
  labels:
    severity: warning
    component: hot-path
  annotations:
    summary: "Hot path p95 supera 1 500 ms — riesgo de incumplir SLO"
    runbook_url: "https://wiki.antihurto/runbooks/hot-path-latency"

- alert: HotPathP95Above2000ms
  expr: |
    histogram_quantile(0.95,
      sum(rate(hot_path_e2e_latency_seconds_bucket[5m])) by (le)
    ) > 2.0
  for: 1m
  labels:
    severity: critical
    component: hot-path
  annotations:
    summary: "SLO VIOLADO: Hot path p95 supera 2 s"
    runbook_url: "https://wiki.antihurto/runbooks/hot-path-slo-violated"
```

### 4.2 Alertas de Lag Kafka

```yaml
- alert: KafkaLagDeduplicatorHigh
  expr: deduplicator_kafka_consumer_lag > 10000
  for: 2m
  labels:
    severity: warning
    component: deduplicator
  annotations:
    summary: "Lag del Deduplicator supera 10 000 mensajes"

- alert: KafkaLagEnrichmentHigh
  expr: enrichment_kafka_consumer_lag > 5000
  for: 2m
  labels:
    severity: warning
    component: enrichment-service
  annotations:
    summary: "Lag del Enrichment Service supera 5 000 mensajes"

- alert: KafkaLagMatcherHigh
  expr: matcher_kafka_consumer_lag > 5000
  for: 2m
  labels:
    severity: warning
    component: matcher-service
  annotations:
    summary: "Lag del Matcher Service supera 5 000 mensajes"

- alert: KafkaLagAlertServiceHigh
  expr: alert_kafka_consumer_lag > 1000
  for: 2m
  labels:
    severity: critical
    component: alert-service
  annotations:
    summary: "Lag del Alert Service supera 1 000 mensajes — alertas en riesgo"
```

### 4.3 Alerta de Lag Debezium

```yaml
- alert: DebeziumConnectorLagHigh
  expr: debezium_connector_lag_ms > 60000
  for: 2m
  labels:
    severity: warning
    component: debezium-cdc
  annotations:
    summary: "Lag Debezium > 60 s — cold path puede estar retrasado"
    description: "El hot path NO está afectado. ClickHouse puede tener datos desactualizados."
```

### 4.4 Alertas de Geocodificación

```yaml
- alert: EnrichmentGeocodingFailureRateHigh
  expr: |
    rate(enrichment_geocoding_failures_total[5m])
    / rate(enrichment_events_processed_total[5m]) > 0.05
  for: 5m
  labels:
    severity: warning
    component: enrichment-service
  annotations:
    summary: "Tasa de fallos de geocodificación > 5 %"
    description: "El sidecar Nominatim puede estar degradado o el dataset desactualizado."
```

### 4.5 Alerta de Redis No Disponible

```yaml
- alert: MatcherRedisFallbackRateHigh
  expr: |
    rate(matcher_fallback_used_total[5m])
    / rate(matcher_events_processed_total[5m]) > 0.05
  for: 2m
  labels:
    severity: critical
    component: matcher-service
  annotations:
    summary: "Matcher usando fallback local > 5 % — Redis posiblemente no disponible"
    description: >
      El cotejo continúa usando el estado local (fail closed). Sin embargo, la
      lista roja local puede estar levemente desactualizada. Verificar Redis Sentinel.
    runbook_url: "https://wiki.antihurto/runbooks/redis-unavailable"
```

### 4.6 Alerta de Alertas Suprimidas

```yaml
- alert: AlertSuppressionRateHigh
  expr: |
    rate(alert_deduplication_suppressed_total[5m])
    / (rate(alert_active_total[5m]) + rate(alert_deduplication_suppressed_total[5m])) > 0.30
  for: 5m
  labels:
    severity: warning
    component: alert-service
  annotations:
    summary: "Tasa de alertas suprimidas > 30 % — posible tormenta de alertas"
    description: >
      Una tasa alta de supresión puede indicar que muchos dispositivos ven el mismo
      vehículo en la misma zona. Esto es normal en operaciones de alto tráfico, pero
      puede indicar un loop en el pipeline si es inusualmente alto.
```

### 4.7 Alerta de DLQ Rate

```yaml
- alert: KafkaDLQRateHigh
  expr: |
    sum(rate(deduplicator_dlq_messages_total[5m])
      + rate(enrichment_dlq_messages_total[5m])
      + rate(alert_events_processed_total[5m])) by (country_code)
    > 0
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "Mensajes siendo enviados a DLQ — revisar logs del componente"
```

---

## 5. Recording Rules Prometheus

```yaml
# Grupo de reglas de recording para el backbone
groups:
  - name: backbone.recording_rules
    interval: 15s
    rules:
      - record: backbone:hot_path_e2e_p95:5m
        expr: |
          histogram_quantile(0.95,
            sum(rate(hot_path_e2e_latency_seconds_bucket[5m])) by (le)
          )

      - record: backbone:hot_path_e2e_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(hot_path_e2e_latency_seconds_bucket[5m])) by (le)
          )

      - record: backbone:events_per_second:5m
        expr: |
          sum(rate(deduplicator_events_processed_total[5m])) by (country_code)

      - record: backbone:alert_rate:5m
        expr: |
          sum(rate(alert_active_total[5m])) by (country_code)

      - record: backbone:geocoding_failure_rate:5m
        expr: |
          sum(rate(enrichment_geocoding_failures_total[5m])) by (country_code)
          / sum(rate(enrichment_events_processed_total[5m])) by (country_code)

      - record: backbone:matcher_fallback_rate:5m
        expr: |
          sum(rate(matcher_fallback_used_total[5m])) by (country_code)
          / sum(rate(matcher_events_processed_total[5m])) by (country_code)
```

---

## 6. Trazabilidad Distribuida (OpenTelemetry)

Cada componente del backbone propaga el `trace_id` del evento original (iniciado en el agente de borde o en el Bridge EMQX→Kafka) mediante headers Kafka. El `trace_id` se incluye en todos los logs estructurados y en los spans de OpenTelemetry.

**Instrumentación mínima por componente:**
- Span por mensaje Kafka consumido y producido.
- Atributo `event.id`, `country_code`, `plate_normalized` en cada span.
- Exportación a Tempo o Jaeger via OpenTelemetry Collector.

Esta instrumentación permite reconstruir el camino completo de un evento sospechoso en el dashboard de trazas: desde el MQTT broker hasta la entrega al WebSocket Gateway.

---

## 7. Referencias

| Documento | Descripción |
|---|---|
| [overview.md](./overview.md) | Presupuesto de latencia por etapa (Sección 4). |
| [deduplicator.md](./deduplicator.md) | Métricas del Deduplicator. |
| [enrichment-service.md](./enrichment-service.md) | Métricas del Enrichment Service. |
| [matcher-service.md](./matcher-service.md) | Métricas del Matcher Service. |
| [alert-service.md](./alert-service.md) | Métricas del Alert Service. |
| [websocket-gateway.md](./websocket-gateway.md) | Métrica `hot_path_e2e_latency_seconds`. |
| [debezium-cdc.md](./debezium-cdc.md) | Métricas Debezium CDC. |
