# ClickHouse — Vista H3 Horaria y Mapas de Calor

**Componente:** Almacenamiento de lectura — capa ClickHouse  
**Versión del documento:** 1.0  
**Referencia:** [clickhouse-schema.md](./clickhouse-schema.md) · [slo-observability.md](./slo-observability.md)

---

## 1. Propósito

La vista materializada `vehicle_events_h3_hourly` agrega los eventos de avistamiento por celda H3 (resolución 8), país y hora. Es la fuente de datos principal para:

- **Mapas de calor** en Superset (Kepler.gl): densidad de tráfico por zona.
- **Análisis de rutas frecuentes**: zonas con alta recurrencia de vehículos hurtados.
- **Tendencias horarias**: patrones de actividad durante el día.

La pre-agregación por hora reduce drásticamente el volumen de datos que el dashboard debe procesar, permitiendo cumplir el SLO de p95 < 3 s en consultas de 7 días.

---

## 2. Justificación de Resolución H3 8

H3 es el sistema de indexación geoespacial hexagonal de Uber. La resolución determina el tamaño de cada celda hexagonal:

| Resolución | Área promedio | Uso recomendado |
|---|---|---|
| H3-6 | ≈ 36.1 km² | Análisis a nivel de ciudad / municipio |
| H3-7 | ≈ 5.2 km² | Barrios grandes |
| **H3-8** | **≈ 0.74 km²** | **Sectores urbanos — seleccionado** |
| H3-9 | ≈ 0.1 km² | Manzanas / intersecciones |
| H3-10 | ≈ 0.015 km² | Cuadras específicas |

**Justificación de H3-8:**

- **Granularidad operacional:** 0.74 km² corresponde a un radio de ~480 m, suficiente para identificar zonas de concentración de hurtos sin comprometer la privacidad individual.
- **Eficiencia de almacenamiento:** Con resolución 8, una ciudad como Bogotá (1 600 km²) se divide en ~2 200 celdas. Con resolución 9 serían ~15 000 celdas — cinco veces más filas en la vista materializada.
- **Compatibilidad con Kepler.gl:** El mapa de calor H3 en Superset/Kepler.gl funciona óptimamente entre resolución 6 y 9. H3-8 produce visualizaciones legibles a niveles de zoom 11–15 (escala de barrio a ciudad).
- **Rendimiento de consulta:** Con H3-8 el número de celdas por query es manejable para filtros de radio (un radio de 5 km contiene ~90 celdas H3-8 vs ~640 celdas H3-9).

---

## 3. Vista Materializada `vehicle_events_h3_hourly`

```sql
CREATE MATERIALIZED VIEW vehicle_events_h3_hourly
ENGINE = SummingMergeTree()
PARTITION BY (country_code, toYYYYMM(hour_ts))
ORDER BY (country_code, h3_index_8, hour_ts)
POPULATE AS
SELECT
    country_code,
    h3_index_8,
    toStartOfHour(event_ts)              AS hour_ts,
    count()                              AS event_count,
    uniq(plate_normalized)               AS unique_plates,
    uniq(device_id)                      AS active_devices,
    sum(if(confidence >= 90.0, 1, 0))   AS high_confidence_count,
    avg(confidence)                      AS avg_confidence,
    min(event_ts)                        AS first_event_ts,
    max(event_ts)                        AS last_event_ts
FROM vehicle_events_daily
GROUP BY
    country_code,
    h3_index_8,
    toStartOfHour(event_ts);

-- Nota: SummingMergeTree suma automáticamente las columnas numéricas
-- al fusionar partes con la misma clave ORDER BY.
-- Columnas no numéricas (first_event_ts, last_event_ts) requieren
-- post-procesamiento con min()/max() en las queries.
```

### 3.1 Política de Retención

La vista `vehicle_events_h3_hourly` tiene **retención indefinida** (sin TTL). Los datos agregados por hora y celda H3 tienen un volumen mucho menor que los eventos individuales (compresión de varios órdenes de magnitud) y son valiosos para análisis de tendencias a largo plazo (patrones estacionales, evolución de zonas de riesgo año a año).

Si en el futuro se requiere limitar el almacenamiento:

```sql
-- Agregar TTL de retención a largo plazo (ejemplo: 10 años)
ALTER TABLE vehicle_events_h3_hourly MODIFY TTL
    hour_ts + INTERVAL 10 YEAR DELETE;
```

---

## 4. Queries de Ejemplo

### 4.1 Dashboard — Mapa de calor de los últimos 7 días (por país)

```sql
SELECT
    h3_index_8,
    sum(event_count)        AS total_events,
    sum(unique_plates)      AS total_unique_plates,   -- aproximación: suma de estimaciones por hora
    avg(avg_confidence)     AS mean_confidence
FROM vehicle_events_h3_hourly
WHERE
    country_code = 'CO'
    AND hour_ts >= now() - INTERVAL 7 DAY
GROUP BY h3_index_8
ORDER BY total_events DESC
LIMIT 500;
```

> **Nota sobre `unique_plates`:** La columna almacena la estimación de cardinalidad por celda-hora generada por `uniq()`. Al usar `SummingMergeTree`, la operación correcta al agregar múltiples filas es `sum(unique_plates)`, que produce la suma de estimaciones individuales (ligero sobreconteo por solapamiento entre horas). Para cardinalidad exacta entre períodos se requeriría `AggregatingMergeTree` con `uniqState()`/`uniqMerge()`.

Este query produce una tabla con hasta 500 celdas H3 ordenadas por densidad de eventos, lista para renderizar en Kepler.gl como mapa de calor hexagonal.

### 4.2 Dashboard — Evolución horaria en una zona específica (últimas 24 h)

```sql
-- Primero: obtener las celdas H3-8 vecinas de una coordenada (radio ~1.5 km ≈ 2 anillos H3-8)
-- Esto se hace en la capa de aplicación con la librería H3:
-- const center = h3.latLngToCell(4.7109, -74.0721, 8);      => "8834..."
-- const ring2  = h3.gridDisk(center, 2);                    => array de ~19 celdas

-- En ClickHouse, buscar por el array de celdas H3:
SELECT
    toStartOfHour(hour_ts)  AS hour,
    sum(event_count)         AS events,
    sum(unique_plates)       AS plates   -- suma de estimaciones por hora (SummingMergeTree)
FROM vehicle_events_h3_hourly
WHERE
    country_code = 'CO'
    AND h3_index_8 IN (
        -- Celdas generadas por h3.gridDisk(center, 2) en la capa de aplicación:
        613917667289489407,
        613917667289489151,
        613917667289489919
        -- ... hasta ~19 celdas
    )
    AND hour_ts >= now() - INTERVAL 24 HOUR
GROUP BY hour
ORDER BY hour ASC;
```

### 4.3 Análisis de Tendencias — Top zonas por mes (comparativa)

```sql
SELECT
    h3_index_8,
    toYYYYMM(hour_ts)       AS month,
    sum(event_count)         AS total_events
FROM vehicle_events_h3_hourly
WHERE
    country_code = 'CO'
    AND hour_ts >= toStartOfMonth(now() - INTERVAL 6 MONTH)
GROUP BY h3_index_8, month
ORDER BY month, total_events DESC;
```

### 4.4 Detección de Zonas de Alta Actividad de Vehículos Hurtados

```sql
-- Cruzar con la tabla principal para filtrar solo eventos con plate en la hot-list.
-- (Requiere join con vehicle_events_daily o con un dictado de stolen_vehicles en ClickHouse)
SELECT
    ve.h3_index_8,
    sum(ve.event_count) AS stolen_vehicle_sightings
FROM vehicle_events_h3_hourly ve
-- Filtrar por matrículas que aparecen en stolen_vehicles (tabla Dictionary o join)
INNER JOIN (
    SELECT DISTINCT plate_normalized
    FROM vehicle_events_daily
    WHERE country_code = 'CO'
      AND event_ts >= now() - INTERVAL 30 DAY
      AND plate_normalized IN (
          -- Lista de placas hurtadas activa (precargada desde PostgreSQL vía Dictionary)
          -- En producción, esto se gestiona como un ClickHouse Dictionary
          SELECT plate_normalized FROM antihurto_pg_dict WHERE country_code = 'CO'
      )
) stolen ON true
WHERE
    ve.country_code = 'CO'
    AND ve.hour_ts >= now() - INTERVAL 30 DAY
GROUP BY ve.h3_index_8
ORDER BY stolen_vehicle_sightings DESC
LIMIT 100;
```

### 4.5 Exportar para Kepler.gl (formato GeoJSON aproximado)

```sql
-- Superset / Apache ECharts pueden renderizar H3 directamente si se pasa el índice.
-- Si se necesita GeoJSON, la conversión H3 → Polygon se hace en la capa de aplicación.
SELECT
    h3_index_8,
    h3ToString(h3_index_8)  AS h3_cell_id,  -- Representación string del índice H3
    sum(event_count)         AS events
FROM vehicle_events_h3_hourly
WHERE
    country_code = 'CO'
    AND hour_ts >= now() - INTERVAL 7 DAY
GROUP BY h3_index_8
ORDER BY events DESC;
```

---

## 5. Configuración de Superset para Mapa de Calor H3

```json
// Configuración del chart H3 Hexagon Layer en Superset (DeckGL H3)
{
  "viz_type": "deck_hex",
  "spatial": {
    "type": "h3",
    "key": "h3_cell_id"
  },
  "metric": {
    "aggregate": "SUM",
    "column": { "column_name": "events" }
  },
  "radius": 8,
  "extruded": true,
  "color_picker": {
    "r": 255, "g": 80, "b": 0, "a": 1
  },
  "linear_color_scheme": "oranges"
}
```

---

## 6. Mantenimiento y Monitoreo

```sql
-- Verificar el tamaño de la vista materializada
SELECT
    formatReadableSize(sum(bytes_on_disk)) AS size,
    sum(rows)                               AS total_rows,
    count()                                 AS parts
FROM system.parts
WHERE table = 'vehicle_events_h3_hourly' AND active = 1;

-- Verificar lag de la vista materializada (comparar con vehicle_events_daily)
SELECT
    max(hour_ts)        AS latest_hour_in_h3_view,
    now()               AS current_time,
    dateDiff('minute', max(hour_ts), now()) AS lag_minutes
FROM vehicle_events_h3_hourly;

-- Repoblar la vista si hay datos faltantes (POPULATE actualiza desde vehicle_events_daily)
-- PRECAUCIÓN: Esta operación puede tardar minutos/horas en tablas grandes.
-- Ejecutar en horario de baja carga.
ALTER TABLE vehicle_events_h3_hourly MATERIALIZE;
```
