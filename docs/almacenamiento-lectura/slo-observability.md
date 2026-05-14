# SLOs y Observabilidad — Almacenamiento de Lectura

**Componente:** Almacenamiento de lectura — observabilidad  
**Versión del documento:** 1.0  
**Referencia:** [overview.md](./overview.md) · [slo-observability.md](./slo-observability.md)

---

## 1. Tabla de SLOs Verificables

| Almacén | Operación | SLO | Percentil | Expresión PromQL de verificación |
|---|---|---|---|---|
| **Redis** | GET `stolen:{cc}:{plate}` | < 1 ms | p99 | `histogram_quantile(0.99, rate(redis_command_duration_seconds_bucket{cmd="get"}[5m])) < 0.001` |
| **Redis** | SET `stolen:{cc}:{plate}` | < 2 ms | p99 | `histogram_quantile(0.99, rate(redis_command_duration_seconds_bucket{cmd="set"}[5m])) < 0.002` |
| **Redis** | Disponibilidad | 99.9 % | — | `avg_over_time(up{job="redis"}[30d]) >= 0.999` |
| **OpenSearch** | Búsqueda exacta por placa (`plate_normalized.keyword`) | < 50 ms | p95 | `histogram_quantile(0.95, rate(opensearch_search_latency_seconds_bucket{query_type="exact"}[5m])) < 0.05` |
| **OpenSearch** | Búsqueda fuzzy (fuzziness AUTO) | < 300 ms | p95 | `histogram_quantile(0.95, rate(opensearch_search_latency_seconds_bucket{query_type="fuzzy"}[5m])) < 0.3` |
| **PostgreSQL** | SELECT geoespacial (radio 10 km, 1 M registros) | < 200 ms | p95 | `histogram_quantile(0.95, rate(pg_query_duration_seconds_bucket{query_type="geo"}[5m])) < 0.2` |
| **OpenSearch** | Tasa de error de indexación | < 0.1 % | — | `rate(opensearch_index_errors_total[5m]) / rate(opensearch_index_requests_total[5m]) < 0.001` |
| **ClickHouse** | Dashboard 7 días | < 3 s | p95 | `histogram_quantile(0.95, rate(clickhouse_query_duration_seconds_bucket{query_kind="select"}[5m])) < 3` |
| **ClickHouse** | Mapa de calor H3 (país) | < 5 s | p95 | `histogram_quantile(0.95, rate(clickhouse_query_duration_seconds_bucket[5m])) < 5` |
| **Debezium CDC** | Lag WAL → Kafka | < 5 s | p99 | `debezium_metrics_MilliSecondsBehindSource{connector="debezium-vehicle-events"} / 1000 < 5` |
| **PostgreSQL** | SELECT por `event_id` | < 10 ms | p95 | `avg(rate(pg_stat_statements_total_exec_time_seconds[5m]) / rate(pg_stat_statements_calls_total[5m])) < 0.01` |
| **PostgreSQL** | SELECT trayectoria por placa | < 50 ms | p95 | `histogram_quantile(0.95, rate(pg_query_duration_seconds_bucket{query_type="trajectory"}[5m])) < 0.05` |
| **PostgreSQL** | Disponibilidad | 99.9 % | — | `avg_over_time(up{job="postgresql"}[30d]) >= 0.999` |

---

## 2. Métricas Prometheus por Almacén

### 2.1 Redis

```promql
# --- Latencia ---
# Latencia p99 de GETs (hot-list lookup)
histogram_quantile(0.99,
  rate(redis_command_duration_seconds_bucket{cmd="get"}[5m])
)

# --- Hitrate ---
# Porcentaje de GETs que encuentran la clave (hitrate de la hot-list)
rate(redis_keyspace_hits_total[5m])
/
(rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))

# --- Tamaño del dataset ---
# Memoria usada por Redis (bytes)
redis_memory_used_bytes

# Número total de claves
redis_db_keys{db="db0"}

# --- Conexiones ---
# Clientes conectados
redis_connected_clients

# --- Replicación ---
# Lag de réplica (bytes de offset pendientes)
redis_replication_offset - redis_replica_replication_offset

# --- Disponibilidad ---
up{job="redis"}
```

### 2.2 OpenSearch

```promql
# --- Latencia de búsqueda ---
histogram_quantile(0.95,
  rate(opensearch_search_latency_seconds_bucket[5m])
)

# --- Tasa de indexación ---
rate(opensearch_index_requests_total[5m])

# --- Tasa de error de indexación ---
rate(opensearch_index_errors_total[5m])

# --- Tamaño de índices activos (por país) ---
opensearch_index_store_size_bytes{index=~"vehicle-events-.*"}

# --- Número de shards no asignados (alerta de cluster unhealthy) ---
opensearch_cluster_shards_unassigned

# --- Estado del cluster (0=green, 1=yellow, 2=red) ---
opensearch_cluster_status

# --- Uso de JVM heap (indica presión de memoria) ---
opensearch_jvm_memory_used_bytes / opensearch_jvm_memory_max_bytes
```

### 2.3 ClickHouse

```promql
# --- Latencia de queries SELECT ---
histogram_quantile(0.95,
  rate(clickhouse_query_duration_seconds_bucket{query_kind="select"}[5m])
)

# --- Queries activas (concurrencia actual) ---
clickhouse_metrics_Query

# --- Inserciones por segundo (ingestión CDC) ---
rate(clickhouse_profile_events_InsertedRows_total[5m])

# --- Tamaño de datos en disco ---
clickhouse_table_parts_bytes{table="vehicle_events_daily"}

# --- Número de partes activas (indicator de fragmentación, debe ser < 300 por tabla) ---
clickhouse_table_parts_count{table="vehicle_events_daily"}

# --- Lag del consumer Kafka (CDC) ---
# Calculado externamente por el exporter de Kafka Consumer Groups:
kafka_consumer_group_lag{group="clickhouse-cdc-consumer",topic="pg.events.cdc"}

# --- Merges activos ---
clickhouse_metrics_Merge
```

### 2.4 PostgreSQL

```promql
# --- Latencia de queries (vía pg_stat_statements) ---
rate(pg_stat_statements_total_exec_time_seconds[5m])
/
rate(pg_stat_statements_calls_total[5m])

# --- Conexiones activas ---
pg_stat_activity_count{state="active"}

# --- Replication lag (réplicas físicas) ---
pg_replication_lag_seconds{slot_type="physical"}

# --- Tamaño del slot CDC (bytes de WAL retenidos) ---
pg_replication_slot_pg_wal_lsn_diff{slot_name="debezium_slot_events"}

# --- Bloat de tablas (indica necesidad de VACUUM) ---
pg_stat_user_tables_n_dead_tup{relname="vehicle_events"}

# --- Hitrate del buffer cache ---
pg_stat_bgwriter_buffers_alloc_total

# --- Latencia de transacciones (percentil 95) ---
histogram_quantile(0.95,
  rate(pg_transaction_duration_seconds_bucket[5m])
)

# --- Tamaño total de la base de datos ---
pg_database_size_bytes{datname="antihurto"}
```

### 2.5 Debezium CDC

```promql
# --- Lag del conector (ms por detrás de la fuente) ---
debezium_metrics_MilliSecondsBehindSource{connector="debezium-vehicle-events"}

# --- Transacciones capturadas por segundo ---
rate(debezium_metrics_NumberOfCommittedTransactions{connector="debezium-vehicle-events"}[5m])

# --- Errores del conector ---
rate(debezium_metrics_NumberOfErrorsTotal{connector="debezium-vehicle-events"}[5m])

# --- Tamaño del slot de replicación PostgreSQL ---
pg_replication_slot_pg_wal_lsn_diff{slot_name="debezium_slot_events"}
```

### 2.6 Pipeline de Proyección (Enrichment Service)

```promql
# --- CA-01: Eventos duplicados ignorados (ON CONFLICT DO NOTHING) ---
# Contador: projection.duplicate_skipped{country_code="CO"}
rate(projection_duplicate_skipped_total[5m])

# --- CR-03: Coordenadas inválidas en eventos entrantes ---
# Contador: projection.invalid_coordinates{country_code="CO"}
rate(projection_invalid_coordinates_total[5m])

# --- CR-02/CR-01: Eventos rechazados y enviados al dead-letter ---
# Contador: projection.dead_letter{reason="MISSING_EVENT_ID"} / {reason="MISSING_COUNTRY_CODE"}
rate(projection_dead_letter_total{reason=~"MISSING_EVENT_ID|MISSING_COUNTRY_CODE"}[5m])

# --- CR-07: Fallos de escritura en Redis (hot-list) ---
# Contador: redis.write.failures{operation="stolen_list_set"}
rate(redis_write_failures_total{operation="stolen_list_set"}[5m])

# --- CR-10: Consultas ClickHouse sin pre-agregado que superan el SLO ---
# Contador: clickhouse.query.slo_degraded{query_kind="heatmap_raw"}
rate(clickhouse_query_slo_degraded_total{query_kind="heatmap_raw"}[5m])
```

---

## 3. Dashboards Grafana Recomendados

| Dashboard | Paneles principales | Fuente de datos |
|---|---|---|
| **Storage Overview** | Estado de los 4 almacenes (verde/amarillo/rojo), throughput agregado, SLOs en tiempo real | Prometheus |
| **Redis Operations** | Hitrate, latencia GET p99, memoria usada, conexiones, lag de réplica | Prometheus + Redis Exporter |
| **OpenSearch Health** | Estado cluster, latencia de búsqueda p95, tasa de indexación, tamaño de índices, JVM heap | Prometheus + OpenSearch Exporter |
| **ClickHouse Analytics** | Latencia de queries p95, inserciones/s, tamaño de partes, merges activos, lag Kafka | Prometheus + ClickHouse Exporter |
| **PostgreSQL Operations** | Latencia de queries p95, conexiones activas, bloat, hitrate buffer, tamaño BD, lag réplica | Prometheus + PostgreSQL Exporter (pg_exporter) |
| **CDC Pipeline** | Lag Debezium (ms), tasa de transacciones CDC, errores, tamaño slot WAL | Prometheus |
| **Data Lifecycle** | Tamaño por partición/país, proyección de crecimiento 6 meses, próximas eliminaciones | Prometheus + consultas directas |

---

## 4. Alertas Operacionales

```yaml
# alertmanager-rules.yaml — reglas de alerta para almacenamiento de lectura

groups:
  - name: almacenamiento-lectura
    rules:

      # Redis
      - alert: RedisHighLatency
        expr: |
          histogram_quantile(0.99,
            rate(redis_command_duration_seconds_bucket{cmd="get"}[5m])
          ) > 0.005
        for: 2m
        labels:
          severity: warning
          component: redis
        annotations:
          summary: "Redis GET p99 > 5 ms (SLO: < 1 ms)"
          description: "La latencia p99 de Redis GET supera 5 ms. Verificar carga del primario y red."

      - alert: RedisDown
        expr: up{job="redis"} == 0
        for: 1m
        labels:
          severity: critical
          component: redis
        annotations:
          summary: "Redis no disponible"
          description: "El primario Redis está caído. Verificar estado de Sentinel."

      - alert: RedisLowHitRate
        expr: |
          rate(redis_keyspace_hits_total[5m])
          / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m]))
          < 0.80
        for: 5m
        labels:
          severity: warning
          component: redis
        annotations:
          summary: "Hitrate Redis < 80 %"
          description: "El hitrate de la hot-list Redis es bajo. Puede indicar que la recarga no completó."

      # OpenSearch
      - alert: OpenSearchHighExactLatency
        expr: |
          histogram_quantile(0.95,
            rate(opensearch_search_latency_seconds_bucket{query_type="exact"}[5m])
          ) > 0.1
        for: 2m
        labels:
          severity: warning
          component: opensearch
        annotations:
          summary: "OpenSearch búsqueda exacta p95 > 100 ms (SLO: < 50 ms)"
          description: "La latencia de búsqueda exacta por plate_normalized.keyword supera el umbral."

      - alert: OpenSearchHighFuzzyLatency
        expr: |
          histogram_quantile(0.95,
            rate(opensearch_search_latency_seconds_bucket{query_type="fuzzy"}[5m])
          ) > 0.5
        for: 5m
        labels:
          severity: warning
          component: opensearch
        annotations:
          summary: "OpenSearch búsqueda fuzzy p95 > 500 ms (SLO: < 300 ms)"

      - alert: OpenSearchClusterRed
        expr: opensearch_cluster_status == 2
        for: 1m
        labels:
          severity: critical
          component: opensearch
        annotations:
          summary: "Cluster OpenSearch en estado RED"
          description: "Hay shards sin asignar. Verificar nodos y espacio en disco."

      - alert: OpenSearchIndexSizeLarge
        expr: |
          opensearch_index_store_size_bytes{index=~"vehicle-events-.*"}
          > 50 * 1024 * 1024 * 1024
        for: 30m
        labels:
          severity: warning
          component: opensearch
        annotations:
          summary: "Índice OpenSearch supera 50 GB — candidato a rollover"

      # ClickHouse
      - alert: ClickHouseHighQueryLatency
        expr: |
          histogram_quantile(0.95,
            rate(clickhouse_query_duration_seconds_bucket{query_kind="select"}[5m])
          ) > 10
        for: 5m
        labels:
          severity: warning
          component: clickhouse
        annotations:
          summary: "ClickHouse query p95 > 10 s (SLO analítico: < 3 s)"

      - alert: ClickHouseHighPartCount
        expr: |
          clickhouse_table_parts_count{table="vehicle_events_daily"} > 300
        for: 10m
        labels:
          severity: warning
          component: clickhouse
        annotations:
          summary: "ClickHouse: demasiadas partes en vehicle_events_daily (> 300)"
          description: "Puede indicar merges lentos o ingesta excesiva. Verificar OPTIMIZE TABLE."

      # CR-10: Consultas ClickHouse sin pre-agregado que superan el SLO de 3 s
      - alert: ClickHouseSLODegraded
        expr: |
          rate(clickhouse_query_slo_degraded_total{query_kind="heatmap_raw"}[5m]) > 0
        for: 1m
        labels:
          severity: warning
          component: clickhouse
        annotations:
          summary: "ClickHouse: consulta de heatmap sin pre-agregado supera SLO de 3 s (CR-10)"
          description: |
            Una consulta de mapa de calor se ejecutó sobre la tabla raw sin usar la vista materializada.
            Puede ocurrir cuando vehicle_events_h3_hourly no está actualizada (primer arranque tras reinicio).
            El analytics-service debe agregar la cabecera X-SLO-Degraded: true en la respuesta al cliente.

      # Debezium CDC
      - alert: DebeziumHighLag
        expr: |
          debezium_metrics_MilliSecondsBehindSource{connector="debezium-vehicle-events"} > 30000
        for: 5m
        labels:
          severity: warning
          component: debezium
        annotations:
          summary: "Debezium CDC lag > 30 s"
          description: "El conector CDC está acumulando lag. Verificar WAL de PostgreSQL y conectividad Kafka."

      - alert: DebeziumConnectorDown
        expr: |
          debezium_metrics_Connected{connector="debezium-vehicle-events"} == 0
        for: 2m
        labels:
          severity: critical
          component: debezium
        annotations:
          summary: "Conector Debezium desconectado"
          description: "El CDC entre PostgreSQL y ClickHouse está interrumpido. ClickHouse dejará de recibir datos."

      - alert: PostgreSQLWALSlotGrowing
        expr: |
          pg_replication_slot_pg_wal_lsn_diff{slot_name="debezium_slot_events"}
          > 10 * 1024 * 1024 * 1024
        for: 10m
        labels:
          severity: critical
          component: postgresql
        annotations:
          summary: "Slot WAL de Debezium retiene > 10 GB"
          description: "El slot debezium_slot_events está acumulando WAL. Riesgo de agotar disco. Verificar estado del conector."

      # PostgreSQL
      - alert: PostgreSQLHighReplicationLag
        expr: |
          pg_replication_lag_seconds{slot_type="physical"} > 30
        for: 5m
        labels:
          severity: warning
          component: postgresql
        annotations:
          summary: "Réplica PostgreSQL con lag > 30 s"

      - alert: PostgreSQLHighTableBloat
        expr: |
          pg_stat_user_tables_n_dead_tup{relname="vehicle_events"} > 10000000
        for: 30m
        labels:
          severity: warning
          component: postgresql
        annotations:
          summary: "Alta cantidad de tuplas muertas en vehicle_events (> 10 M)"
          description: "VACUUM puede estar atrasado. Verificar autovacuum y ejecutar VACUUM ANALYZE manualmente."
```
