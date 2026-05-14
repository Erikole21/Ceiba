# search-service — Especificación

**Componente:** `api-frontend-analitica`  
**Versión del documento:** 1.0  
**OpenAPI:** [openapi/search-service.yaml](./openapi/search-service.yaml)

---

## 1. Responsabilidad

El `search-service` orquesta la búsqueda de eventos de avistamiento de un vehículo por matrícula. Combina:

- **OpenSearch** para descubrimiento rápido por matrícula (exacto + fuzzy).
- **PostgreSQL+PostGIS** para obtener coordenadas, metadatos enriquecidos y estado de incidente.
- **Object Storage (S3-API / MinIO)** para generar presigned URLs de thumbnails con TTL 5 min.

Es el servicio que materializa el flujo del Read Path descrito en §2.6 de la propuesta de arquitectura.

---

## 2. Diagrama de Secuencia de Orquestación

```mermaid
sequenceDiagram
    autonumber
    participant GW as API Gateway
    participant SS as search-service
    participant EI as EventsIndexPort\n(OpenSearch adapter)
    participant ER as EventsRepositoryPort\n(PostgreSQL adapter)
    participant OS as ObjectStoragePort\n(S3/MinIO adapter)

    GW->>SS: GET /v1/plates/{plate}/events\nX-Country-Code: CO\nX-Role: officer
    SS->>SS: Normalizar placa (mayúsculas, sin guiones)\nABC-123 → ABC123
    SS->>EI: query plate_normalized=ABC123\ncountry_code=CO\nfrom, to, limit, offset
    Note over EI: OpenSearch query:\nexact match + fuzzy AUTO\nindex: vehicle-events-co-2026-05
    EI-->>SS: [event_id list, total hits]
    alt No event_ids found (total hits = 0, offset = 0)
        SS-->>GW: 404 Not Found\n{"error":"plate_not_found","plate":"ABC123","country":"CO"}
    else No event_ids found (total hits = 0, offset > 0)
        SS-->>GW: 200 OK\n{"data":[],"total":0,"page":N,"page_size":20,"pages":0}
    end
    SS->>ER: SELECT * FROM vehicle_events\nWHERE event_id IN (...)\nAND country_code='CO'
    Note over ER: PostgreSQL+PostGIS:\nST_AsGeoJSON(coordinates)\ntimestamp, device_id, confidence, image_uri
    ER-->>SS: [VehicleEvent records]
    loop Para cada event con image_uri
        SS->>OS: generatePresignedGetUrl(image_uri, ttl=5min)
        OS-->>SS: presigned_url (expires_at)
    end
    SS->>SS: Construir respuesta paginada\nOrdenar por timestamp DESC
    SS-->>GW: 200 OK\nPlateEventsResponse JSON
```

---

## 3. Endpoint

### GET /v1/plates/{plate}/events

Retorna la lista paginada de eventos de avistamiento de la matrícula especificada.

#### Parámetros

| Parámetro | Tipo | Ubicación | Requerido | Descripción |
|---|---|---|---|---|
| `plate` | string | path | Sí | Matrícula a buscar. Se normaliza internamente (mayúsculas, sin guiones). Ej: `ABC123` o `ABC-123`. |
| `from` | string (ISO 8601) | query | No | Inicio del rango de fechas. Ej: `2026-05-01T00:00:00Z`. Default: 30 días atrás. |
| `to` | string (ISO 8601) | query | No | Fin del rango de fechas. Ej: `2026-05-13T23:59:59Z`. Default: ahora. |
| `country` | string | query | No | Override de `country_code`. Solo `role=admin` puede especificar un país distinto al del token. |
| `device_id` | string | query | No | Filtrar por dispositivo específico. |
| `min_confidence` | number | query | No | Confianza mínima del ANPR (0.0–1.0). Default: `0.6`. |
| `limit` | integer | query | No | Elementos por página. Default: `20`. Máximo: `100`. |
| `offset` | integer | query | No | Desplazamiento para paginación. Default: `0`. |

#### Respuesta exitosa — 200 OK

```json
{
  "data": [
    {
      "event_id": "evt-2026-05-13-co-abc123-001",
      "plate": "ABC123",
      "plate_normalized": "ABC123",
      "timestamp": "2026-05-13T14:32:07.123Z",
      "lat": 4.710989,
      "lon": -74.072092,
      "device_id": "dev-bog-norte-042",
      "confidence": 0.94,
      "thumbnail_url": "https://storage.anti-hurto.internal/thumbnails/evt-2026-05-13-co-abc123-001.jpg?X-Amz-Expires=300&X-Amz-Signature=...",
      "thumbnail_expires_at": "2026-05-13T14:37:07.123Z",
      "country": "CO",
      "city": "Bogotá",
      "address": "Calle 72 con Carrera 11, Bogotá",
      "is_stolen": true,
      "incident_id": "inc-abc123-2026-05"
    }
  ],
  "total": 47,
  "page": 1,
  "page_size": 20,
  "pages": 3,
  "plate": "ABC123",
  "query_time_ms": 187
}
```

#### Schema de `PlateEvent`

| Campo | Tipo | Descripción |
|---|---|---|
| `event_id` | string | Identificador único del evento. Formato: `evt-{YYYY-MM-DD}-{cc}-{plate}-{seq}`. |
| `plate` | string | Matrícula tal como fue detectada por el ANPR. |
| `plate_normalized` | string | Matrícula normalizada (mayúsculas, sin guiones, sin espacios). |
| `timestamp` | string (ISO 8601) | Timestamp de captura del evento (UTC). |
| `lat` | number | Latitud en grados decimales (WGS84). |
| `lon` | number | Longitud en grados decimales (WGS84). |
| `device_id` | string | Identificador del dispositivo captura. |
| `confidence` | number | Confianza del ANPR (0.0–1.0). |
| `thumbnail_url` | string (URL) | URL pre-firmada del thumbnail del vehículo (TTL 5 min). Nulo si imagen no disponible. |
| `thumbnail_expires_at` | string (ISO 8601) | Timestamp de expiración de la URL pre-firmada. |
| `country` | string | `country_code` del tenant (ISO 3166-1 alpha-2). |
| `city` | string | Ciudad geocodificada del evento. |
| `address` | string | Dirección aproximada geocodificada. Puede ser nulo. |
| `is_stolen` | boolean | `true` si el vehículo está en la lista de hurtados al momento de la consulta. |
| `incident_id` | string | ID del incidente activo asociado (nulo si no hay incidente). |

#### Códigos de Error

| Código HTTP | `error` | Descripción |
|---|---|---|
| 400 | `invalid_plate_format` | La matrícula no tiene un formato reconocible. |
| 400 | `invalid_date_range` | El rango `from`/`to` es inválido (from > to, rango > 365 días). |
| 401 | `unauthorized` | JWT ausente, inválido o expirado. |
| 403 | `forbidden_country` | El usuario no tiene acceso al `country_code` solicitado. |
| 404 | `plate_not_found` | La matrícula no tiene eventos registrados en el sistema para el país dado (OpenSearch retorna 0 resultados en la primera consulta, offset=0). Si la placa existe pero no hay eventos en el rango de fechas/filtros aplicados con offset > 0, se retorna 200 con array vacío. |
| 429 | `rate_limit_exceeded` | Rate limit excedido. Ver header `Retry-After`. |
| 500 | `internal_error` | Error interno. Ver `X-Trace-ID` para diagnóstico. |
| 503 | `index_unavailable` | OpenSearch o PostgreSQL no disponibles (circuit breaker abierto). |

---

## 4. Puertos Hexagonales

### 4.1 `EventsIndexPort` (OpenSearch)

```typescript
interface EventsIndexPort {
  searchByPlate(params: {
    plate_normalized: string;
    country_code: string;
    from: Date;
    to: Date;
    device_id?: string;
    min_confidence?: number;
    limit: number;
    offset: number;
  }): Promise<{ event_ids: string[]; total: number }>;
}
```

**Adapter `OpenSearchEventsIndexAdapter`:** ejecuta una query combinada exacta+fuzzy contra el índice `vehicle-events-{country_code}-{YYYY-MM}`. Normaliza la placa antes del query. Usa `multi_match` con `type: best_fields` y `fuzziness: AUTO`.

### 4.2 `EventsRepositoryPort` (PostgreSQL+PostGIS)

```typescript
interface EventsRepositoryPort {
  findByEventIds(params: {
    event_ids: string[];
    country_code: string;
  }): Promise<VehicleEvent[]>;
}
```

**Adapter `PostgreSQLEventsRepositoryAdapter`:** ejecuta `SELECT` con `WHERE event_id = ANY($1) AND country_code = $2`. Incluye `ST_X(coordinates)` y `ST_Y(coordinates)` para lat/lon.

### 4.3 `ObjectStoragePort` (S3-API / MinIO)

```typescript
interface ObjectStoragePort {
  generatePresignedGetUrl(params: {
    object_key: string;
    ttl_seconds: number;
  }): Promise<{ url: string; expires_at: Date }>;
}
```

**Adapter `S3ObjectStorageAdapter`:** usa AWS SDK v3 `getSignedUrl` con TTL 300 s (5 min). Configurable para MinIO con endpoint override. El `object_key` se extrae del campo `image_uri` del evento (`s3://bucket/path/to/thumbnail.jpg` → `path/to/thumbnail.jpg`).

---

## 5. Circuit Breaker

Cada puerto tiene un circuit breaker configurado:

| Puerto | Umbral de fallo | Timeout de apertura | Half-open probe |
|---|---|---|---|
| `EventsIndexPort` | 5 errores en 10 s | 30 s | 1 request cada 10 s |
| `EventsRepositoryPort` | 5 errores en 10 s | 30 s | 1 request cada 10 s |
| `ObjectStoragePort` | 3 errores en 10 s | 60 s | 1 request cada 30 s |

Si `EventsIndexPort` o `EventsRepositoryPort` están en estado abierto, el servicio retorna `503 Service Unavailable`. Si `ObjectStoragePort` está abierto, el servicio retorna los eventos sin `thumbnail_url` (degraded mode — `thumbnail_url: null`).

---

## 6. SLO y Métricas Prometheus

**SLOs:**
- Búsqueda por matrícula (fuzzy): p95 < 300 ms.
- Carga de trayectoria con thumbnails (≤ 50 eventos): p95 < 800 ms.
- Tasa de error (5xx): < 0.1 % en ventana de 5 min.

**Métricas Prometheus:**

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `search_request_duration_seconds` | Histogram | `status`, `country_code` | Latencia total del endpoint por estado HTTP. |
| `search_opensearch_query_duration_seconds` | Histogram | `country_code` | Latencia de la query a OpenSearch. |
| `search_postgres_query_duration_seconds` | Histogram | `country_code` | Latencia del SELECT a PostgreSQL. |
| `search_presign_duration_seconds` | Histogram | — | Latencia de generación de presigned URLs. |
| `search_circuit_breaker_state{port}` | Gauge | `port` | Estado del circuit breaker: 0=closed, 1=open, 0.5=half-open. |
| `search_errors_total{error_type}` | Counter | `error_type`, `country_code` | Errores por tipo. |
| `search_results_empty_total` | Counter | `country_code` | Búsquedas sin resultados. |
| `search_map_slo_exceeded_total` | Counter | `country_code` | Consultas de trayectoria que superaron el SLO de 800 ms. |

---

## 7. Ejemplo Completo de Request/Response

### Request

```http
GET /v1/plates/ABC-123/events?from=2026-05-01T00:00:00Z&to=2026-05-13T23:59:59Z&limit=2
Host: api.anti-hurto.internal
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

(Después del API Gateway, llega al servicio con headers adicionales:)
```http
GET /v1/plates/ABC-123/events?from=2026-05-01T00:00:00Z&to=2026-05-13T23:59:59Z&limit=2
X-Country-Code: CO
X-Role: officer
X-Zone: BOG-NORTE
X-User-Id: usr-pol-001
X-Trace-ID: 4bf92f3577b34da6a3ce929d0e0e4736
```

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Trace-ID: 4bf92f3577b34da6a3ce929d0e0e4736
Cache-Control: no-store

{
  "data": [
    {
      "event_id": "evt-2026-05-13-co-abc123-004",
      "plate": "ABC-123",
      "plate_normalized": "ABC123",
      "timestamp": "2026-05-13T14:32:07.123Z",
      "lat": 4.710989,
      "lon": -74.072092,
      "device_id": "dev-bog-norte-042",
      "confidence": 0.94,
      "thumbnail_url": "https://minio.anti-hurto.internal/thumbnails/2026/05/13/abc123-004.jpg?X-Amz-Expires=300&X-Amz-Signature=abc123xyz",
      "thumbnail_expires_at": "2026-05-13T14:37:07.123Z",
      "country": "CO",
      "city": "Bogotá",
      "address": "Calle 72 # 11-23, Chapinero, Bogotá",
      "is_stolen": true,
      "incident_id": "inc-abc123-2026-05"
    },
    {
      "event_id": "evt-2026-05-13-co-abc123-003",
      "plate": "ABC123",
      "plate_normalized": "ABC123",
      "timestamp": "2026-05-13T13:18:44.001Z",
      "lat": 4.688391,
      "lon": -74.054207,
      "device_id": "dev-bog-centro-017",
      "confidence": 0.91,
      "thumbnail_url": "https://minio.anti-hurto.internal/thumbnails/2026/05/13/abc123-003.jpg?X-Amz-Expires=300&X-Amz-Signature=def456uvw",
      "thumbnail_expires_at": "2026-05-13T14:37:07.123Z",
      "country": "CO",
      "city": "Bogotá",
      "address": "Carrera 10 # 16-41, La Candelaria, Bogotá",
      "is_stolen": true,
      "incident_id": "inc-abc123-2026-05"
    }
  ],
  "total": 47,
  "page": 1,
  "page_size": 2,
  "pages": 24,
  "plate": "ABC123",
  "query_time_ms": 187
}
```

---

## 8. Estrategia de Degradación de SLO — Mapa con Alto Volumen de Eventos

Cuando una matrícula tiene más de **10.000 eventos** activos en el período consultado, el tiempo de construcción de la respuesta puede superar el SLO de 800 ms para render de mapa. El servicio aplica la siguiente estrategia:

### 8.1 Truncamiento Paginado

La respuesta **nunca** contiene más eventos que `pageSize` (máximo 100 por página). El cliente es responsable de paginar usando `offset` para obtener el historial completo. No se devuelven más de `pageSize` eventos en una única respuesta.

### 8.2 Agrupación Geoespacial (Clustering)

Cuando `total > 10.000` y se detecta que el tiempo de consulta supera 600 ms (umbral preventivo), el servicio activa una estrategia de simplificación:

1. Los event_ids de OpenSearch se truncan a los primeros 500 por relevancia de score.
2. Las coordenadas de PostgreSQL+PostGIS se agrupan en celdas H3 de resolución 9 (~0.1 km²): se retorna un representante por celda con el timestamp más reciente.
3. El campo `clustered: true` se incluye en la respuesta para que la Web App renderice los puntos como clusters visuales en MapLibre GL.

### 8.3 Cabecera `X-SLO-Degraded`

Si el tiempo de respuesta total supera los **800 ms**, la respuesta incluye:

```http
X-SLO-Degraded: true
X-SLO-Degraded-Reason: map_event_threshold_exceeded
```

Y el body incluye el campo `slo_degraded: true`:

```json
{
  "data": [...],
  "total": 10842,
  "page": 1,
  "page_size": 20,
  "pages": 543,
  "plate": "ABC123",
  "query_time_ms": 1043,
  "slo_degraded": true
}
```

### 8.4 Métrica Prometheus

Cada respuesta que supera el umbral de 800 ms incrementa:
```
search_map_slo_exceeded_total{country_code="CO"}
```

---

## 9. Referencias

- [openapi/search-service.yaml](./openapi/search-service.yaml)
- [almacenamiento-lectura/opensearch-schema.md](../almacenamiento-lectura/opensearch-schema.md)
- [almacenamiento-lectura/postgresql-schema.md](../almacenamiento-lectura/postgresql-schema.md)
- [slo-observability.md](./slo-observability.md)
- [ADR-005 — Arquitectura Hexagonal](../propuesta-arquitectura-hurto-vehiculos.md#adr-005--arquitectura-hexagonal-para-servicios-sensibles-a-la-nube)
