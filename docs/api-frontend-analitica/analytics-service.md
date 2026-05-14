# analytics-service — Especificación

**Componente:** `api-frontend-analitica`  
**Versión del documento:** 1.0  
**OpenAPI:** [openapi/analytics-service.yaml](./openapi/analytics-service.yaml)

---

## 1. Responsabilidad

El `analytics-service` expone los tres endpoints de analítica para el perfil de analista: heatmap H3, tendencias de hurto y rutas de escape frecuentes. Consulta exclusivamente las vistas materializadas de ClickHouse definidas en [`almacenamiento-lectura/clickhouse-h3-views.md`](../almacenamiento-lectura/clickhouse-h3-views.md).

El servicio implementa una estrategia de detección de staleness en las vistas materializadas y comunica degradación de SLO al cliente mediante el header `X-SLO-Degraded: true` y el campo `data_freshness_warning` en la respuesta.

---

## 2. Endpoint — GET /v1/analytics/heatmap

Retorna densidad de eventos de hurto por celda hexagonal H3 (resolución 8, ~0.74 km²).

### Parámetros

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `country` | string | Sí | `country_code` del tenant. |
| `from` | string (ISO 8601) | No | Inicio del rango. Default: 7 días atrás. |
| `to` | string (ISO 8601) | No | Fin del rango. Default: ahora. |
| `city` | string | No | Filtrar por ciudad. |
| `min_events` | integer | No | Umbral mínimo de eventos para incluir la celda. Default: 1. |
| `resolution` | integer | No | Resolución H3 (7–9). Default: 8. |

### Respuesta — 200 OK

```json
{
  "data": [
    {
      "h3_index": "88193ad9d3fffff",
      "event_count": 47,
      "avg_confidence": 0.891,
      "lat_centroid": 4.710989,
      "lon_centroid": -74.072092,
      "city": "Bogotá"
    },
    {
      "h3_index": "88193ad9dbfffff",
      "event_count": 31,
      "avg_confidence": 0.875,
      "lat_centroid": 4.695412,
      "lon_centroid": -74.083201,
      "city": "Bogotá"
    }
  ],
  "total_cells": 142,
  "total_events": 3847,
  "country": "CO",
  "from": "2026-05-06T00:00:00Z",
  "to": "2026-05-13T23:59:59Z",
  "query_time_ms": 1240,
  "data_freshness_warning": null
}
```

### Query SQL en ClickHouse

```sql
-- Vista: vehicle_events_h3_hourly
SELECT
    h3_index_8 AS h3_index,
    count() AS event_count,
    avg(confidence) AS avg_confidence,
    avg(lat) AS lat_centroid,
    avg(lon) AS lon_centroid,
    city
FROM vehicle_events_h3_hourly
WHERE
    country_code = {country: String}
    AND event_hour >= {from: DateTime}
    AND event_hour <= {to: DateTime}
    {city_filter}
GROUP BY h3_index_8, city
HAVING event_count >= {min_events: UInt32}
ORDER BY event_count DESC
SETTINGS max_execution_time = 30
```

---

## 3. Endpoint — GET /v1/analytics/trends

Retorna el conteo diario de hurtos para análisis de tendencias.

### Parámetros

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `country` | string | Sí | `country_code` del tenant. |
| `from` | string (ISO 8601) | No | Inicio del rango. Default: 30 días atrás. |
| `to` | string (ISO 8601) | No | Fin del rango. Default: ahora. |
| `city` | string | No | Filtrar por ciudad específica. |
| `granularity` | enum | No | `day` \| `week` \| `month`. Default: `day`. |

### Respuesta — 200 OK

```json
{
  "data": [
    {
      "date": "2026-05-13",
      "theft_count": 23,
      "city": "Bogotá"
    },
    {
      "date": "2026-05-12",
      "theft_count": 18,
      "city": "Bogotá"
    }
  ],
  "total_period": 412,
  "country": "CO",
  "granularity": "day",
  "from": "2026-04-13",
  "to": "2026-05-13",
  "query_time_ms": 892,
  "data_freshness_warning": null
}
```

### Query SQL en ClickHouse

```sql
-- Tabla: vehicle_events_daily (MergeTree, particionada por toYYYYMM)
SELECT
    toDate(event_ts) AS date,
    count() AS theft_count,
    city
FROM vehicle_events_daily
WHERE
    country_code = {country: String}
    AND is_stolen = true
    AND event_ts >= {from: DateTime}
    AND event_ts <= {to: DateTime}
    {city_filter}
GROUP BY date, city
ORDER BY date DESC
SETTINGS max_execution_time = 30
```

---

## 4. Endpoint — GET /v1/analytics/routes

Retorna las rutas de escape más frecuentes de vehículos hurtados (secuencias de celdas H3 recurrentes).

### Parámetros

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `country` | string | Sí | `country_code` del tenant. |
| `from` | string (ISO 8601) | No | Inicio del rango. Default: 30 días atrás. |
| `to` | string (ISO 8601) | No | Fin del rango. |
| `city` | string | No | Filtrar por ciudad. |
| `min_frequency` | integer | No | Frecuencia mínima de aparición de la ruta. Default: 3. |
| `max_routes` | integer | No | Máximo de rutas a retornar. Default: 10. |

### Respuesta — 200 OK

```json
{
  "data": [
    {
      "route_id": "route-co-bog-001",
      "segment_sequence": [
        {
          "order": 1,
          "h3_index": "88193ad9d3fffff",
          "lat": 4.710989,
          "lon": -74.072092,
          "avg_time_in_cell_minutes": 4.2
        },
        {
          "order": 2,
          "h3_index": "88193ad9dbfffff",
          "lat": 4.695412,
          "lon": -74.083201,
          "avg_time_in_cell_minutes": 2.8
        }
      ],
      "frequency": 12,
      "avg_total_duration_minutes": 22.5,
      "city": "Bogotá"
    }
  ],
  "total_routes": 8,
  "country": "CO",
  "query_time_ms": 2340,
  "data_freshness_warning": null
}
```

### Query SQL en ClickHouse

```sql
-- Análisis de rutas: secuencias de H3 para placas hurtadas
WITH plate_sequences AS (
    SELECT
        plate_normalized,
        groupArray(h3_index_8) AS h3_sequence,
        count() AS sequence_length
    FROM (
        SELECT
            plate_normalized,
            h3_index_8,
            event_ts
        FROM vehicle_events_daily
        WHERE
            country_code = {country: String}
            AND is_stolen = true
            AND event_ts >= {from: DateTime}
            AND event_ts <= {to: DateTime}
            {city_filter}
        ORDER BY plate_normalized, event_ts
    )
    GROUP BY plate_normalized
    HAVING sequence_length >= 2
)
SELECT
    h3_sequence,
    count() AS frequency,
    avg(sequence_length) AS avg_length
FROM plate_sequences
GROUP BY h3_sequence
HAVING frequency >= {min_frequency: UInt32}
ORDER BY frequency DESC
LIMIT {max_routes: UInt32}
SETTINGS max_execution_time = 60
```

---

## 5. Puerto Hexagonal `AnalyticsRepositoryPort`

```typescript
interface AnalyticsRepositoryPort {
  queryHeatmap(params: HeatmapParams): Promise<HeatmapResult>;
  queryTrends(params: TrendsParams): Promise<TrendsResult>;
  queryRoutes(params: RoutesParams): Promise<RoutesResult>;
  checkDataFreshness(country_code: string): Promise<DataFreshnessInfo>;
}

interface DataFreshnessInfo {
  last_updated_at: Date;
  lag_minutes: number;
  is_stale: boolean;  // true si lag > 60 min
}
```

**Adapter `ClickHouseAnalyticsAdapter`:** usa el cliente HTTP de ClickHouse. Configura `max_execution_time` y `max_memory_usage` por query. Implementa `checkDataFreshness` via:

```sql
SELECT max(event_hour) AS last_updated, now() - max(event_hour) AS lag_seconds
FROM vehicle_events_h3_hourly
WHERE country_code = {country: String}
```

---

## 6. Detección de Staleness en Vistas Materializadas

Si el lag de actualización de las vistas materializadas supera 60 minutos (ej. por retraso en el pipeline CDC), el servicio:

1. Incluye `data_freshness_warning` en el cuerpo de la respuesta:
   ```json
   "data_freshness_warning": {
     "last_updated_at": "2026-05-13T13:00:00Z",
     "lag_minutes": 92,
     "message": "Los datos pueden estar desactualizados. Última actualización hace 92 minutos."
   }
   ```
2. Añade el header `X-SLO-Degraded: true` en la respuesta HTTP.
3. Registra el evento en Prometheus: `analytics_slo_degraded_total{country_code}` (también expuesto como `analytics_stale_data_total` por compatibilidad con dashboards anteriores).

Los datos se sirven de todas formas (best-effort). No se retorna error.

---

## 7. SLO y Métricas Prometheus

**SLOs:**
- Heatmap H3 (país, 7 días): p95 < 3 s.
- Tendencias diarias (30 días): p95 < 3 s.
- Rutas frecuentes (30 días): p95 < 5 s.

**Métricas Prometheus:**

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `analytics_query_duration_seconds` | Histogram | `endpoint`, `country_code` | Latencia por endpoint. |
| `analytics_clickhouse_query_duration_seconds` | Histogram | `query_type`, `country_code` | Latencia de la query ClickHouse. |
| `analytics_slo_degraded_total` | Counter | `country_code` | Veces que se detectó staleness > 60 min (nombre canónico; alias: `analytics_stale_data_total`). |
| `analytics_errors_total` | Counter | `endpoint`, `error_type` | Errores por endpoint. |
| `analytics_circuit_breaker_state` | Gauge | — | Estado del circuit breaker hacia ClickHouse. |

---

## 8. Referencias

- [openapi/analytics-service.yaml](./openapi/analytics-service.yaml)
- [almacenamiento-lectura/clickhouse-h3-views.md](../almacenamiento-lectura/clickhouse-h3-views.md)
- [almacenamiento-lectura/clickhouse-schema.md](../almacenamiento-lectura/clickhouse-schema.md)
- [superset-integration.md](./superset-integration.md)
- [slo-observability.md](./slo-observability.md)
