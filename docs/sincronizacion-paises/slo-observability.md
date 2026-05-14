# SLOs y Observabilidad — Pilar 4

**Change:** `sincronizacion-paises`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. SLOs del Pilar 4

| SLO | Objetivo | Ventana | Componente |
|---|---|---|---|
| Latencia CDC BD → Kafka | P99 ≤ 1 s desde cambio en BD policial hasta publicación en stolen.vehicles.events (CA-01) | 7 días rolling | Adaptador CDC |
| Frescura Redis (modo CDC) | P99 ≤ 30 s desde cambio en BD hasta placa en Redis | 7 días rolling | Adaptador CDC + Canonical Vehicles Service |
| Frescura Redis (modo JDBC 15 min) | P99 ≤ 16 min desde cambio en BD hasta placa en Redis | 7 días rolling | Adaptador JDBC + Canonical Vehicles Service |
| Frescura BF en borde | P99 ≤ versión Redis + 5 min | 7 días rolling | Edge Distribution Service + MQTT |
| Disponibilidad de ingesta | 99.9 % (adaptadores operativos sin alertas activas) | 30 días rolling | Adaptadores de país |
| Lag consumer Canonical Vehicles Service | Alerta si lag > 10 000 mensajes | En tiempo real | Canonical Vehicles Service |
| Tamaño BF | FP < 1 %, tamaño < 1 MB | Cada generación | Edge Distribution Service |
| Tiempo de generación BF | P95 < 60 s para cualquier país | 7 días rolling | Edge Distribution Service |

---

## 2. Métricas Prometheus por Componente

### 2.1 Adaptador de País

```yaml
# Métricas de ingesta
- adapter_records_fetched_total{country_code, mode}           # Counter
- adapter_records_published_total{country_code, mode}         # Counter
- adapter_records_failed_total{country_code, mode, reason}    # Counter
- adapter_cycle_duration_seconds{country_code, mode}          # Histogram
  # buckets: [0.1, 0.5, 1, 5, 15, 30, 60, 120, 300]

# Métricas de salud
- adapter_last_successful_sync_timestamp{country_code, mode}  # Gauge (Unix ts)
- adapter_checkpoint_lag_seconds{country_code, mode}          # Gauge
- adapter_connectivity_failures_total{country_code, mode}     # Counter

# Métricas de calidad
- adapter_validation_failures_total{country_code, mode, field} # Counter
- adapter_kafka_produce_errors_total{country_code, mode}       # Counter
```

### 2.2 Canonical Vehicles Service

```yaml
# Métricas de consumo
- canonical_vehicles_consumer_lag{country_code, partition}         # Gauge
- canonical_vehicles_events_processed_total{country_code, status}  # Counter
- canonical_vehicles_events_dlq_total{country_code, error_code}    # Counter
- canonical_vehicles_rejected_invalid_country_total{country_code}  # Counter
- canonical_vehicles_processing_duration_seconds{country_code}     # Histogram
  # buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5]

# Métricas de persistencia
- canonical_vehicles_upsert_duration_seconds{country_code}         # Histogram
- canonical_vehicles_pg_errors_total{country_code}                 # Counter

# Métricas Redis
- canonical_vehicles_redis_failures_total{country_code}            # Counter
- canonical_vehicles_redis_retries_total{country_code}             # Counter
- canonical_vehicles_redis_set_duration_seconds{country_code}      # Histogram

# Métricas de negocio
- canonical_vehicles_active_stolen_count{country_code}             # Gauge
- canonical_vehicles_recovered_total{country_code}                 # Counter
- canonical_vehicles_recovered_not_found_total{country_code}       # Counter
- canonical_vehicles_dedup_skipped_total{country_code}             # Counter
- canonical_vehicles_reconciliation_records_total{country_code}    # Gauge
```

### 2.3 Edge Distribution Service

```yaml
# Métricas de generación BF
- edge_distribution_bf_generated_total{country_code}               # Counter
- edge_distribution_bf_full_snapshot_total{country_code}           # Counter
- edge_distribution_bf_generation_duration_seconds{country_code}   # Histogram
  # buckets: [1, 5, 10, 30, 60, 120]

# Métricas de distribución
- edge_distribution_mqtt_publish_total{country_code, type}         # Counter
- edge_distribution_mqtt_publish_errors_total{country_code}        # Counter
- edge_distribution_mqtt_publish_duration_seconds{country_code}    # Histogram

# Métricas de tamaño y calidad
- edge_distribution_bf_element_count{country_code}                 # Gauge
- edge_distribution_bf_size_bytes{country_code}                    # Gauge
- edge_distribution_bf_compressed_size_bytes{country_code}         # Gauge
- edge_distribution_bf_capacity_ratio{country_code}                # Gauge (0..1)

# Métricas de almacenamiento
- edge_distribution_storage_store_errors_total{country_code}       # Counter
- edge_distribution_storage_store_duration_seconds{country_code}   # Histogram
```

---

## 3. Consultas PromQL de SLO

### 3.1 Lag del consumer de Canonical Vehicles Service

```promql
# Lag actual por country_code (alerta si > 10000)
canonical_vehicles_consumer_lag{country_code="CO"}

# Porcentaje del tiempo que el lag estuvo bajo el umbral (7 días)
(
  count_over_time(canonical_vehicles_consumer_lag{country_code="CO"}[7d] <= 10000)
  /
  count_over_time(canonical_vehicles_consumer_lag{country_code="CO"}[7d])
) * 100
```

### 3.2 Tasa de error del adaptador

```promql
# Tasa de fallos de publicación por minuto
rate(adapter_records_failed_total{country_code="CO"}[5m])

# Error ratio (fallos / total intentos)
rate(adapter_records_failed_total{country_code="CO"}[5m])
/
rate(adapter_records_fetched_total{country_code="CO"}[5m])
```

### 3.3 Tiempo desde última sincronización exitosa

```promql
# Retraso del adaptador en segundos (staleness)
time() - adapter_last_successful_sync_timestamp{country_code="CO"}

# Alerta: retraso > SLA acordado (ejemplo 900s = 15 min para modo JDBC)
time() - adapter_last_successful_sync_timestamp{country_code="CO", mode="JDBC"} > 900
```

### 3.4 Capacidad del Bloom Filter

```promql
# Ratio de capacidad (alerta si > 90%)
edge_distribution_bf_capacity_ratio{country_code="CO"} > 0.9

# Tamaño comprimido (alerta si > 900KB = 921600 bytes)
edge_distribution_bf_compressed_size_bytes{country_code="CO"} > 921600
```

---

## 4. Dashboards Grafana Recomendados

### 4.1 Dashboard: Pilar 4 — Vista General

**Filas:**

1. **Estado por País** — tabla con todos los países activos, semáforo de estado:
   - Verde: último sync < SLA, lag < 1000, BF publicado en < 2 min.
   - Amarillo: último sync entre SLA y 2×SLA, lag 1000–10000.
   - Rojo: adaptador caído, lag > 10000, BF no publicado en > 30 min.

2. **Throughput de Ingesta** — gráfico de series de tiempo:
   - `rate(adapter_records_published_total[5m])` por country_code.
   - `rate(adapter_records_failed_total[5m])` por country_code.

3. **Lag del Canonical Vehicles Service** — gráfico de series:
   - `canonical_vehicles_consumer_lag` por country_code.
   - Línea de alerta en 10 000.

4. **Estado del Bloom Filter** — panel por país:
   - Número de elementos activos.
   - Tamaño del BF (comprimido).
   - Tiempo desde última generación.

### 4.2 Dashboard: Detalle por País

Variables: `country_code` (dropdown con los países activos).

**Paneles:**

- Registros fetched vs. published vs. failed (gráfico de series).
- Checkpoint lag (gauge).
- Distribución de duración de ciclos (heatmap histograma).
- DLQ: mensajes por error_code (tabla).
- Reconciliaciones Redis (timeline).
- Generaciones de BF: delta vs. full snapshot (gráfico de barras).

### 4.3 Dashboard: SLO Frescura

- Ventana: 7 días rolling.
- Para cada país y modo: porcentaje del tiempo dentro del SLA.
- Burn rate de error budget (si el SLO de 99.9% disponibilidad de ingesta se usa como presupuesto de error).

---

## 5. Alertas Operacionales

### 5.1 Alertas críticas (severity: critical — pagerduty)

| Alerta | Condición | Acción |
|---|---|---|
| `adapter_connectivity_down` | `rate(adapter_connectivity_failures_total[5m]) > 0` AND `adapter_last_successful_sync_timestamp` no actualizado en 15 min | Contactar contraparte policial del país; revisar VPN / credenciales |
| `canonical_vehicles_consumer_lag_critical` | `canonical_vehicles_consumer_lag > 50000` durante 5 min | Escalar con SRE; revisar recursos del Canonical Vehicles Service |
| `edge_distribution_bf_not_published` | `edge_distribution_bf_generated_total` no incrementa en 30 min cuando hay cambios | Revisar EDS: conectividad con MQTT y MinIO |

### 5.2 Alertas de advertencia (severity: warning — slack)

| Alerta | Condición | Acción |
|---|---|---|
| `adapter_cdc_publish_latency` | `histogram_quantile(0.99, rate(adapter_cycle_duration_seconds_bucket{mode="CDC"}[5m])) > 1` por > 5 min | Revisar throughput del adaptador CDC; posible saturación de recursos o lentitud en la BD policial (CA-01) |
| `adapter_sla_breach` | `time() - adapter_last_successful_sync_timestamp > sla_threshold` por > 5 min | Investigar causa; comunicar impacto al equipo de operaciones |
| `canonical_vehicles_consumer_lag_warning` | `canonical_vehicles_consumer_lag > 10000` durante 5 min | Revisar velocidad de procesamiento; considerar aumentar replicas |
| `canonical_vehicles_dlq_spike` | `rate(canonical_vehicles_events_dlq_total[5m]) > 10` | Revisar mensajes en DLQ; posible problema en un adaptador |
| `edge_distribution_bf_capacity_warning` | `edge_distribution_bf_capacity_ratio > 0.9` | Planificar ajuste de parámetros del BF en próxima versión |
| `canonical_vehicles_redis_failures` | `canonical_vehicles_redis_failures_total` incrementa > 5 en 1 min | Verificar disponibilidad de Redis; activar reconciliación si Redis vuelve |
| `adapter_validation_failures_spike` | `rate(adapter_validation_failures_total[5m]) > 5` | Revisar si hubo cambio en el esquema de la BD policial |

### 5.3 Configuración de alertas en Prometheus

```yaml
# prometheus/rules/sincronizacion-paises.yaml
groups:
  - name: sincronizacion-paises.critical
    rules:
      - alert: AdapterConnectivityDown
        expr: |
          (time() - adapter_last_successful_sync_timestamp) > 900
        for: 5m
        labels:
          severity: critical
          component: adapter
        annotations:
          summary: "Adaptador {{ $labels.country_code }} sin sincronización exitosa en > 15 min"
          description: "El adaptador {{ $labels.country_code }} (modo: {{ $labels.mode }}) no ha completado una sincronización exitosa en {{ $value | humanizeDuration }}."
          runbook: "https://docs.antihurto.internal/runbooks/adapter-connectivity"

      - alert: CanonicalVehiclesConsumerLagCritical
        expr: |
          canonical_vehicles_consumer_lag > 50000
        for: 5m
        labels:
          severity: critical
          component: canonical-vehicles-service
        annotations:
          summary: "Lag crítico en canonical-vehicles-service para {{ $labels.country_code }}"
          description: "El lag del consumer supera 50,000 mensajes para {{ $labels.country_code }}."

  - name: sincronizacion-paises.warning
    rules:
      - alert: CanonicalVehiclesConsumerLagWarning
        expr: |
          canonical_vehicles_consumer_lag > 10000
        for: 5m
        labels:
          severity: warning
          component: canonical-vehicles-service
        annotations:
          summary: "Lag elevado en canonical-vehicles-service para {{ $labels.country_code }}"
          description: "El lag del consumer supera 10,000 mensajes para {{ $labels.country_code }}."

      - alert: BloomFilterCapacityWarning
        expr: |
          edge_distribution_bf_capacity_ratio > 0.9
        for: 5m
        labels:
          severity: warning
          component: edge-distribution-service
        annotations:
          summary: "BF de {{ $labels.country_code }} al {{ $value | humanizePercentage }} de capacidad"
          description: "El Bloom filter de {{ $labels.country_code }} tiene {{ $value | humanizePercentage }} de su capacidad máxima. Planificar ajuste de parámetros."
```

---

## 6. Logs Estructurados

Todos los componentes del Pilar 4 emiten logs JSON con los siguientes campos estándar:

```json
{
  "timestamp": "2026-01-15T14:31:05.123Z",
  "level": "INFO",
  "service": "canonical-vehicles-service",
  "country_code": "CO",
  "event_id": "f47ac10b-...",
  "message": "Upsert completed",
  "duration_ms": 45,
  "trace_id": "abc123..."
}
```

Los eventos de audit (acceso a `owner_id`, eventos RECOVERED, reconciliaciones) se emiten con nivel `AUDIT` y se envían a un stream separado de logs de auditoría (inmutable).

---

## 7. Trazas Distribuidas (OpenTelemetry)

El trace distribuido de un evento desde el adaptador hasta el BF en el borde incluye los siguientes spans:

```
adapter.ingest_cycle                 [adaptador]
  adapter.fetch_records              [adaptador → BD policial]
  adapter.map_to_canonical           [adaptador]
  adapter.kafka_publish              [adaptador → Kafka]

canonical_vehicles.process_event     [Canonical Vehicles Service]
  canonical_vehicles.deserialize     [CVS → Schema Registry]
  canonical_vehicles.validate        [CVS]
  canonical_vehicles.pg_upsert       [CVS → PostgreSQL]
  canonical_vehicles.redis_set       [CVS → Redis]
  canonical_vehicles.kafka_publish   [CVS → stolen.vehicles.canonical]
  canonical_vehicles.notify_eds      [CVS → Edge Distribution Service]

edge_distribution.generate_bf        [Edge Distribution Service]
  edge_distribution.pg_query         [EDS → PostgreSQL]
  edge_distribution.build_bf         [EDS]
  edge_distribution.calculate_delta  [EDS]
  edge_distribution.store_version    [EDS → MinIO]
  edge_distribution.mqtt_publish     [EDS → MQTT Broker]
```

Exportar a **Tempo** (o Jaeger) mediante el SDK de OpenTelemetry. El `trace_id` se propaga como header Kafka (`X-Trace-Id`) entre componentes.

---

## 8. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [`canonical-vehicles-service.md`](./canonical-vehicles-service.md) | Métricas detalladas del CVS |
| [`edge-distribution-service.md`](./edge-distribution-service.md) | Métricas detalladas del EDS |
| [`country-adapter-framework.md`](./country-adapter-framework.md) | Métricas detalladas de los adaptadores |
| [`sla-freshness.md`](./sla-freshness.md) | SLAs que estas métricas monitorizan |
| [`docs/backbone-procesamiento/slo-observability.md`](../backbone-procesamiento/slo-observability.md) | Observabilidad del backbone (complementaria) |
