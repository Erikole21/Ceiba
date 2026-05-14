# SLOs y Observabilidad — API / Frontend / Analítica

**Componente:** `api-frontend-analitica`  
**Versión del documento:** 1.0

---

## 1. Tabla de SLOs Verificables por Servicio

| Servicio | Operación | SLO | Medición |
|---|---|---|---|
| **API Gateway** | Auth + routing overhead | p95 < 20 ms | `gateway_upstream_latency_seconds{quantile="0.95"}` |
| **search-service** | Búsqueda por matrícula (fuzzy) | p95 < 300 ms | `search_request_duration_seconds{quantile="0.95"}` |
| **search-service** | Trayectoria con thumbnails (≤ 50 eventos) | p95 < 800 ms | `search_trajectory_duration_seconds{quantile="0.95"}` |
| **search-service** | Tasa de error (5xx) | < 0.1 % en 5 min | `rate(search_errors_total[5m]) / rate(search_requests_total[5m])` |
| **alerts-service** | Latencia Kafka → WebSocket delivery | p95 < 2 s | `alerts_e2e_latency_seconds{quantile="0.95"}` |
| **alerts-service** | Tasa de entrega exitosa | > 99 % | `rate(alerts_delivered_total[5m]) / rate(alerts_received_total[5m])` |
| **incident-service** | POST recovery-action (escritura) | p95 < 500 ms | `incident_write_duration_seconds{quantile="0.95"}` |
| **analytics-service** | Heatmap H3 país (7 días) | p95 < 3 s | `analytics_query_duration_seconds{endpoint="heatmap",quantile="0.95"}` |
| **analytics-service** | Tendencias 30 días | p95 < 3 s | `analytics_query_duration_seconds{endpoint="trends",quantile="0.95"}` |
| **devices-service** | GET /v1/devices (paginado) | p95 < 200 ms | `devices_request_duration_seconds{quantile="0.95"}` |
| **vehicles-service** | GET /v1/vehicles (paginado) | p95 < 200 ms | `vehicles_request_duration_seconds{quantile="0.95"}` |

---

## 2. Métricas Prometheus por Servicio

### 2.1 API Gateway (Kong)

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `gateway_requests_total` | Counter | `status`, `route`, `method` | Total de requests por ruta y estado HTTP. |
| `gateway_upstream_latency_seconds` | Histogram | `route` | Latencia del upstream (buckets: 5ms, 10ms, 20ms, 50ms, 100ms, 200ms, 500ms). |
| `gateway_auth_rejected_total` | Counter | `reason` | Requests rechazados por autenticación JWT (`invalid_token`, `expired_token`, `missing_token`). |
| `gateway_authz_rejected_total` | Counter | `endpoint`, `required_role`, `actual_role` | Requests rechazados por autorización de rol (403 Forbidden). Incrementa cuando un usuario con rol insuficiente intenta acceder a un endpoint restringido (ej. `officer` en `/v1/vehicles`). |
| `gateway_jwks_cache_miss_total` | Counter | — | Requests al endpoint JWKS de Keycloak por cache miss (> 5 min). |
| `gateway_ratelimit_hit_total` | Counter | `route`, `country_code` | Requests rechazados por rate limit. |

### 2.2 search-service

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `search_requests_total` | Counter | `status`, `country_code` | Total de requests al endpoint. |
| `search_request_duration_seconds` | Histogram | `status`, `country_code` | Latencia total del endpoint. |
| `search_opensearch_query_duration_seconds` | Histogram | `country_code` | Latencia de la query a OpenSearch. |
| `search_postgres_query_duration_seconds` | Histogram | `country_code` | Latencia del SELECT a PostgreSQL. |
| `search_presign_duration_seconds` | Histogram | — | Latencia de generación de presigned URLs. |
| `search_circuit_breaker_state` | Gauge | `port` | Estado del circuit breaker (0=closed, 1=open, 0.5=half-open). |
| `search_errors_total` | Counter | `error_type`, `country_code` | Errores por tipo. |
| `search_results_empty_total` | Counter | `country_code` | Búsquedas sin resultados (plate_not_found). |
| `search_map_slo_exceeded_total` | Counter | `country_code` | Consultas de trayectoria que superaron el SLO de 800 ms (mapa con > 10 K eventos). |

### 2.3 alerts-service

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `ws_active_connections` | Gauge | `country_code` | Conexiones WebSocket activas por país. |
| `alerts_received_total` | Counter | `country_code`, `zone` | Alertas recibidas de Kafka. |
| `alerts_delivered_total` | Counter | `country_code`, `zone` | Alertas entregadas vía WebSocket. |
| `alerts_no_subscribers_total` | Counter | `country_code`, `zone` | Alertas sin oficiales conectados en la zona. |
| `alerts_e2e_latency_seconds` | Histogram | `country_code` | Latencia Kafka produced_at → WebSocket emit. |
| `ws_disconnections_total` | Counter | `country_code`, `reason` | Desconexiones WebSocket. |
| `kafka_consumer_lag` | Gauge | `topic`, `partition` | Lag del consumer group `alerts-service-group`. |

### 2.4 incident-service

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `incidents_created_total` | Counter | `country_code` | Incidentes creados. |
| `incidents_by_status` | Gauge | `country_code`, `status` | Incidentes activos por estado. |
| `incident_write_duration_seconds` | Histogram | `operation` | Latencia de escritura (create, update). |
| `incident_country_sync_total` | Counter | `country_code`, `status` | Resultados del write-back al país. |
| `incident_state_transition_total` | Counter | `from_status`, `to_status` | Conteo de transiciones de estado. |

### 2.5 analytics-service

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `analytics_query_duration_seconds` | Histogram | `endpoint`, `country_code` | Latencia por endpoint (heatmap, trends, routes). |
| `analytics_clickhouse_query_duration_seconds` | Histogram | `query_type`, `country_code` | Latencia de la query ClickHouse interna. |
| `analytics_slo_degraded_total` | Counter | `country_code` | Veces que se detectó staleness > 60 min en vistas materializadas (nombre canónico; alias: `analytics_stale_data_total`). |
| `analytics_errors_total` | Counter | `endpoint`, `error_type` | Errores por endpoint. |
| `analytics_circuit_breaker_state` | Gauge | — | Estado del circuit breaker hacia ClickHouse. |

### 2.6 devices-service y vehicles-service

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `devices_total` | Gauge | `country_code`, `status` | Total de dispositivos por estado. |
| `devices_request_duration_seconds` | Histogram | `method`, `endpoint` | Latencia CRUD dispositivos. |
| `devices_ota_initiated_total` | Counter | `country_code` | OTAs iniciadas. |
| `devices_ota_completed_total` | Counter | `country_code`, `result` | OTAs completadas (success/failed). |
| `vehicles_request_duration_seconds` | Histogram | `method`, `country_code` | Latencia CRUD vehículos. |
| `vehicles_created_total` | Counter | `country_code`, `source` | Vehículos creados por origen. |
| `vehicles_redis_sync_total` | Counter | `country_code`, `operation` | Operaciones de sincronización Redis. |

---

## 3. Dashboards Grafana Recomendados

### 3.1 Dashboard: API Gateway Overview

**Panels:**
- Total requests/sec por ruta (line chart).
- HTTP status code distribution (pie).
- p95 latency por ruta (gauge).
- Auth rejections rate (counter).
- JWKS cache misses (counter).
- Rate limit hits por país (heatmap temporal).

### 3.2 Dashboard: search-service SLOs

**Panels:**
- SLO burn rate: latencia p95 vs. objetivo 300 ms (gauge + threshold).
- Error rate (5xx) vs. SLO 0.1 % (gauge).
- Distribución de latencia por cuantil (heatmap de histograma).
- Circuit breaker state por puerto (state timeline).
- Búsquedas sin resultados vs. con resultados (stacked bar).
- Latencia por subcomponente: OpenSearch, PostgreSQL, presign (stacked bar).

### 3.3 Dashboard: alerts-service Realtime

**Panels:**
- WebSocket active connections por país (live gauge).
- Alertas recibidas vs. entregadas en los últimos 10 min (counter diff).
- Latencia e2e de alertas (histogram heatmap).
- Kafka consumer lag (line chart, SLO < 1000).
- Desconexiones WebSocket por razón (counter).
- Alertas sin suscriptores (indica zonas sin oficiales conectados).

### 3.4 Dashboard: Analytics Service Performance

**Panels:**
- p95 latency por endpoint (bar chart vs. SLO 3 s).
- Stale data detections (counter).
- ClickHouse query duration por tipo (box plot).
- Error rate por endpoint.

---

## 4. Alertas Operacionales

### 4.1 Alertas Críticas (PagerDuty)

```yaml
# prometheus-alerts-api-layer.yaml
groups:
  - name: api-frontend-analitica-critical
    rules:
      - alert: SearchLatencyExceedsSLO
        expr: |
          histogram_quantile(0.95, 
            rate(search_request_duration_seconds_bucket[5m])
          ) > 0.3
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "search-service p95 > 300 ms durante 5 min"
          description: "p95 actual: {{ $value | humanizeDuration }}. SLO: 300ms."
          runbook: "https://wiki.anti-hurto.internal/runbooks/search-latency"

      - alert: AlertsKafkaLagHigh
        expr: kafka_consumer_lag{topic="vehicle.alerts"} > 1000
        for: 30s
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Lag del consumer de alertas > 1000 mensajes"
          description: "Lag actual: {{ $value }} mensajes. Los oficiales pueden estar recibiendo alertas con retraso > 2s."
          runbook: "https://wiki.anti-hurto.internal/runbooks/kafka-consumer-lag"

      - alert: GatewayJWKSUnavailable
        expr: increase(gateway_jwks_cache_miss_total[5m]) > 0 and gateway_auth_rejected_total > 0
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "JWKS endpoint de Keycloak no responde"
          description: "Cache miss de JWKS con rechazos de autenticación. Usuarios no pueden acceder."

      - alert: WSConnectionsDrasticDrop
        expr: |
          (ws_active_connections - ws_active_connections offset 5m) / ws_active_connections offset 5m < -0.5
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Caída drástica de conexiones WebSocket (> 50% en 5 min)"

  - name: api-frontend-analitica-warning
    rules:
      - alert: HighErrorRateSearchService
        expr: |
          rate(search_errors_total{error_type=~"5.."}[1m]) 
          / rate(search_requests_total[1m]) > 0.05
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Error rate > 5% en search-service en los últimos 60s"

      - alert: AlertsNoSubscribersHigh
        expr: rate(alerts_no_subscribers_total[5m]) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Alertas sin suscriptores: puede indicar zonas sin cobertura policial"

      - alert: AnalyticsStaleData
        expr: analytics_slo_degraded_total > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Datos de analítica desactualizados (lag > 60 min)"
          description: "Las vistas materializadas de ClickHouse no se actualizan correctamente."
```

---

## 5. OpenTelemetry y Propagación de X-Trace-ID

### 5.1 Configuración del SDK OpenTelemetry

Todos los microservicios configuran el SDK de OpenTelemetry para Node.js/Go:

```typescript
// src/observability/tracing.ts (NestJS)
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,  // Tempo / Jaeger
  }),
  instrumentations: [new HttpInstrumentation()],
  serviceName: process.env.SERVICE_NAME,  // 'search-service', etc.
});

sdk.start();
```

### 5.2 Propagación del Trace ID

El API Gateway (Kong) inyecta el header `X-Trace-ID` con el trace ID de W3C TraceContext (`traceparent`). Todos los microservicios:

1. Leen el `traceparent` header y lo propagan a sus llamadas downstream (OpenSearch, PostgreSQL, Redis).
2. Incluyen el trace ID en todos los logs estructurados JSON.
3. Retornan el trace ID en el header `X-Trace-ID` de la respuesta HTTP (para correlación por el cliente).

```typescript
// Middleware de logging estructurado
app.use((req, res, next) => {
  const traceId = req.headers['x-trace-id'] || generateTraceId();
  res.setHeader('X-Trace-ID', traceId);
  // Inyectar en el contexto de logging
  logger.child({ trace_id: traceId, service: SERVICE_NAME });
  next();
});
```

### 5.3 Formato de Log Estructurado

```json
{
  "timestamp": "2026-05-13T14:32:07.123Z",
  "level": "info",
  "service": "search-service",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "country_code": "CO",
  "user_id": "usr-pol-001",
  "message": "Plate events query completed",
  "plate_normalized": "ABC123",
  "results": 47,
  "opensearch_latency_ms": 45,
  "postgres_latency_ms": 23,
  "presign_latency_ms": 12,
  "total_latency_ms": 187
}
```

---

## 6. Referencias

- [api-gateway.md §9 — Métricas del Gateway](./api-gateway.md)
- [search-service.md §6 — Métricas](./search-service.md)
- [alerts-service.md §8 — Métricas](./alerts-service.md)
- [incident-service.md §7 — Métricas](./incident-service.md)
- [analytics-service.md §7 — Métricas](./analytics-service.md)
- [almacenamiento-lectura/slo-observability.md](../almacenamiento-lectura/slo-observability.md)
