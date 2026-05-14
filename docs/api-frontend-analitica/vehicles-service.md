# vehicles-service — Especificación

**Componente:** `api-frontend-analitica`  
**Versión del documento:** 1.0  
**OpenAPI:** [openapi/vehicles-service.yaml](./openapi/vehicles-service.yaml)

---

## 1. Responsabilidad

El `vehicles-service` gestiona la lista manual de vehículos hurtados en la tabla canónica `stolen_vehicles`. Complementa la sincronización automática desde los adapters de país: permite a un administrador agregar, listar y actualizar registros de forma manual (ej. reporte presencial, inclusión urgente).

**Restricción de rol:** todas las escrituras (`POST`, `PATCH`) requieren `role=admin`. Las lecturas (`GET`) permiten `admin` y `supervisor`.

---

## 2. Mapping al Schema Canónico `stolen_vehicles`

| Campo API | Campo PostgreSQL | Tipo | Descripción |
|---|---|---|---|
| `plate` | `plate` | string | Matrícula tal como fue registrada. |
| `plate_normalized` | `plate_normalized` | string | Matrícula normalizada (mayúsculas, sin guiones). Generado internamente. |
| `country_code` | `country_code` | string | Tenant (ISO 3166-1 alpha-2). Del JWT, no del body. |
| `theft_date` | `theft_date` | date | Fecha del hurto (YYYY-MM-DD). |
| `theft_location` | `theft_location` | string | Descripción del lugar de hurto. |
| `owner_dni_hash` | `owner_dni_hash` | string | HMAC-SHA256 con sal por país del DNI del propietario. El servicio calcula el hash; el API nunca expone el DNI en claro. |
| `brand` | `brand` | string | Marca (ej. Chevrolet). |
| `vehicle_class` | `vehicle_class` | string | Clase (automóvil, motocicleta, camión, etc.). |
| `line` | `line` | string | Línea o modelo (ej. Spark, NX, Tracker). |
| `color` | `color` | string | Color principal del vehículo. |
| `model_year` | `model_year` | integer | Año modelo. |
| `status` | `status` | enum | `stolen` \| `recovered` \| `cancelled`. |
| `schema_version` | `schema_version` | integer | Versión del schema canónico aplicado. Se incrementa en cada PATCH. |
| `source` | `source` | string | Origen del registro: `manual_admin` \| `country_adapter_{cc}`. |
| `extensions` | `extensions` | JSONB | Campos adicionales por país. |
| `created_at` | `created_at` | timestamp | Fecha de creación del registro. |
| `updated_at` | `updated_at` | timestamp | Última actualización. |

---

## 3. Endpoints

### GET /v1/vehicles

Retorna la lista paginada de vehículos hurtados del tenant.

**Roles permitidos:** `admin`, `supervisor`

#### Parámetros

| Parámetro | Tipo | Descripción |
|---|---|---|
| `status` | enum | Filtrar por `stolen`, `recovered`, `cancelled`. Default: `stolen`. |
| `brand` | string | Filtrar por marca. |
| `plate` | string | Búsqueda por matrícula (parcial o completa). |
| `limit` | integer | Default: 20. Máximo: 100. |
| `offset` | integer | Default: 0. |

#### Respuesta — 200 OK

```json
{
  "data": [
    {
      "plate": "ABC-123",
      "plate_normalized": "ABC123",
      "country_code": "CO",
      "theft_date": "2026-04-28",
      "theft_location": "Carrera 15 # 93-47, Chapinero, Bogotá",
      "brand": "Chevrolet",
      "vehicle_class": "automóvil",
      "line": "Tracker",
      "color": "Rojo",
      "model_year": 2019,
      "status": "stolen",
      "schema_version": 2,
      "source": "manual_admin",
      "created_at": "2026-04-28T10:15:00.000Z",
      "updated_at": "2026-05-02T08:30:00.000Z"
    }
  ],
  "total": 1847,
  "page": 1,
  "page_size": 20
}
```

---

### POST /v1/vehicles

Registra un nuevo vehículo hurtado manualmente.

**Roles permitidos:** `admin`

#### Request Body

```json
{
  "plate": "XYZ-789",
  "theft_date": "2026-05-13",
  "theft_location": "Autopista Norte km 15, Bogotá",
  "owner_dni": "79012345",
  "brand": "Toyota",
  "vehicle_class": "automóvil",
  "line": "Corolla",
  "color": "Blanco",
  "model_year": 2022,
  "extensions": {
    "soat_number": "CO-SOAT-2026-XYZ789",
    "internal_case_id": "BOG-2026-04521"
  }
}
```

**Nota sobre `owner_dni`:** el campo se recibe en claro pero nunca se almacena en claro. El servicio calcula `HMAC-SHA256(owner_dni, salt_per_country)` y almacena el hash en `owner_dni_hash`. El `owner_dni` no aparece en ninguna respuesta.

#### Respuesta — 201 Created

```json
{
  "plate": "XYZ789",
  "plate_normalized": "XYZ789",
  "country_code": "CO",
  "status": "stolen",
  "schema_version": 1,
  "source": "manual_admin",
  "created_at": "2026-05-13T14:45:00.000Z"
}
```

#### Errores

| Código | `error` | Descripción |
|---|---|---|
| 400 | `invalid_plate_format` | Formato de matrícula no reconocido. |
| 400 | `missing_required_fields` | Faltan campos obligatorios del schema canónico. |
| 401 | `unauthorized` | JWT inválido. |
| 403 | `forbidden_role` | Solo `role=admin` puede crear registros. |
| 409 | `plate_already_exists` | La matrícula ya existe como `stolen` en el país. |

---

### PATCH /v1/vehicles/{id}

Actualiza un registro de vehículo hurtado. Permite cambiar `status`, agregar notas en `extensions`, o corregir campos.

**Roles permitidos:** `admin`

#### Path Parameters

| Parámetro | Tipo | Descripción |
|---|---|---|
| `id` | string | `plate_normalized` del vehículo (ej. `XYZ789`). |

#### Request Body

```json
{
  "status": "recovered",
  "extensions": {
    "recovery_date": "2026-05-13",
    "recovery_notes": "Recuperado por patrulla BOG-N-042 en Carrera 30."
  }
}
```

#### Respuesta — 200 OK

```json
{
  "plate_normalized": "XYZ789",
  "country_code": "CO",
  "status": "recovered",
  "schema_version": 2,
  "updated_at": "2026-05-13T15:00:00.000Z"
}
```

---

## 4. Registro en `audit_log`

Cada escritura (`POST`, `PATCH`) genera una entrada en `audit_log`:

```json
{
  "entry_id": "aud-veh-xyz789-001",
  "entity_type": "stolen_vehicle",
  "entity_id": "XYZ789",
  "country_code": "CO",
  "action": "create",
  "actor_id": "usr-admin-001",
  "actor_role": "admin",
  "ip_address": "10.0.1.50",
  "timestamp": "2026-05-13T14:45:00.000Z",
  "changes": {
    "status": { "from": null, "to": "stolen" },
    "plate": { "from": null, "to": "XYZ789" }
  }
}
```

---

## 5. Coordinación con Cache Redis

Al crear o actualizar un vehículo, el `vehicles-service` invalida/actualiza la clave Redis correspondiente:

```
# Al marcar un vehículo como stolen:
SET stolen:co:XYZ789 <vehicle_data_json>
EXPIRE stolen:co:XYZ789 86400  # TTL 24h

# Al marcar un vehículo como recovered:
DEL stolen:co:XYZ789
```

La clave sigue el schema `stolen:{country_code_lowercase}:{plate_normalized}` definido en [`almacenamiento-lectura/redis-schema.md`](../almacenamiento-lectura/redis-schema.md).

---

## 6. Métricas Prometheus

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `vehicles_request_duration_seconds` | Histogram | `method`, `endpoint`, `country_code` | Latencia de operaciones CRUD. |
| `vehicles_created_total` | Counter | `country_code`, `source` | Vehículos creados por origen. |
| `vehicles_updated_total` | Counter | `country_code`, `status` | Actualizaciones por estado destino. |
| `vehicles_redis_sync_total` | Counter | `country_code`, `operation` | Operaciones de sincronización Redis. |

---

## 7. Referencias

- [openapi/vehicles-service.yaml](./openapi/vehicles-service.yaml)
- [almacenamiento-lectura/postgresql-schema.md](../almacenamiento-lectura/postgresql-schema.md) — tabla `stolen_vehicles`
- [almacenamiento-lectura/redis-schema.md](../almacenamiento-lectura/redis-schema.md)
- [slo-observability.md](./slo-observability.md)
- [ADR-006 — Anti-Corruption Layer](../propuesta-arquitectura-hurto-vehiculos.md#adr-006--anti-corruption-layer-por-país--modelo-canónico-versionado)
