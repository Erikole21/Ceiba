# devices-service — Especificación

**Componente:** `api-frontend-analitica`  
**Versión del documento:** 1.0  
**OpenAPI:** [openapi/devices-service.yaml](./openapi/devices-service.yaml)

---

## 1. Responsabilidad

El `devices-service` gestiona el inventario de dispositivos de captura. Permite al administrador registrar, consultar y actualizar dispositivos, y disparar actualizaciones OTA de forma asíncrona.

El acceso está restringido por `country_code` del token JWT: un administrador de Colombia (`country_code=CO`) sólo puede ver y gestionar dispositivos de Colombia.

---

## 2. Modelo de Dispositivo

| Campo | Tipo | Descripción |
|---|---|---|
| `device_id` | string | Identificador único del dispositivo. Formato: `dev-{country_code}-{location}-{seq}`. |
| `country_code` | string | Tenant (ISO 3166-1 alpha-2). Discriminador multi-tenant. |
| `serial_number` | string | Número de serie del hardware. |
| `location_name` | string | Nombre descriptivo de la ubicación (ej. "Calle 72 con Carrera 11, Bogotá"). |
| `lat` | number | Latitud de instalación fija del dispositivo. |
| `lon` | number | Longitud de instalación fija del dispositivo. |
| `zone` | string | Zona policial asociada (ej. `BOG-NORTE`). |
| `status` | enum | `active` \| `inactive` \| `pending_registration` \| `pending_ota` \| `error`. |
| `firmware_version` | string | Versión del firmware actual (semver). |
| `last_seen_at` | string (ISO 8601) | Última señal de heartbeat recibida. |
| `last_event_at` | string (ISO 8601) | Último evento de placa procesado. |
| `anpr_model` | string | Modelo del componente ANPR (ej. `Hikvision-DS-MH1823-1`). |
| `connectivity` | string | Tipo de conectividad: `gsm_4g`, `gsm_3g`, `wifi`, `fiber`. |
| `battery_level` | number | Nivel de batería de respaldo (0–100). Nulo si no aplica. |
| `created_at` | string (ISO 8601) | Fecha de registro en el sistema. |
| `updated_at` | string (ISO 8601) | Última actualización del registro. |

---

## 3. Endpoints

### GET /v1/devices

Retorna la lista paginada de dispositivos del tenant.

**Roles permitidos:** `admin`, `supervisor`

#### Parámetros

| Parámetro | Tipo | Descripción |
|---|---|---|
| `status` | string | Filtrar por estado. |
| `zone` | string | Filtrar por zona. |
| `limit` | integer | Default: 20. Máximo: 100. |
| `offset` | integer | Default: 0. |

#### Respuesta — 200 OK

```json
{
  "data": [
    {
      "device_id": "dev-co-bog-norte-042",
      "country_code": "CO",
      "serial_number": "SN-20240315-042",
      "location_name": "Calle 72 con Carrera 11, Chapinero, Bogotá",
      "lat": 4.710989,
      "lon": -74.072092,
      "zone": "BOG-NORTE",
      "status": "active",
      "firmware_version": "2.4.1",
      "last_seen_at": "2026-05-13T14:30:00.000Z",
      "last_event_at": "2026-05-13T14:28:45.000Z",
      "anpr_model": "Hikvision-DS-MH1823-1",
      "connectivity": "gsm_4g",
      "battery_level": 87,
      "created_at": "2024-03-15T09:00:00.000Z",
      "updated_at": "2026-05-13T14:30:00.000Z"
    }
  ],
  "total": 342,
  "page": 1,
  "page_size": 20
}
```

---

### POST /v1/devices

Registra un nuevo dispositivo.

**Roles permitidos:** `admin`

#### Request Body

```json
{
  "serial_number": "SN-20260513-099",
  "location_name": "Autopista Norte con Calle 183, Usaquén, Bogotá",
  "lat": 4.748921,
  "lon": -74.045832,
  "zone": "BOG-NORTE",
  "anpr_model": "Hikvision-DS-MH1823-1",
  "connectivity": "gsm_4g"
}
```

#### Respuesta — 201 Created

```json
{
  "device_id": "dev-co-bog-norte-099",
  "country_code": "CO",
  "status": "pending_registration",
  "created_at": "2026-05-13T14:40:00.000Z"
}
```

El dispositivo queda en `pending_registration` hasta que el agente edge complete el bootstrap con el token PKI. El campo `status` se actualiza a `active` cuando se recibe el primer heartbeat.

---

### GET /v1/devices/{id}

Retorna los detalles de un dispositivo específico.

**Roles permitidos:** `admin`, `supervisor`

#### Respuesta — 200 OK

Schema igual al elemento de `GET /v1/devices`, con campos adicionales:

```json
{
  "device_id": "dev-co-bog-norte-042",
  "...": "...",
  "ota_history": [
    {
      "ota_id": "ota-dev-co-bog-norte-042-001",
      "target_version": "2.4.1",
      "requested_at": "2026-04-10T10:00:00Z",
      "completed_at": "2026-04-10T10:05:30Z",
      "status": "completed"
    }
  ]
}
```

---

### POST /v1/devices/{id}/ota

Dispara una actualización de firmware OTA de forma asíncrona.

**Roles permitidos:** `admin`

#### Request Body

```json
{
  "target_version": "2.5.0",
  "notes": "Actualización de seguridad: parche CVE-2026-1234"
}
```

#### Respuesta — 202 Accepted

```json
{
  "ota_id": "ota-dev-co-bog-norte-042-002",
  "device_id": "dev-co-bog-norte-042",
  "target_version": "2.5.0",
  "status": "pending_ota",
  "requested_at": "2026-05-13T14:42:00.000Z",
  "estimated_completion": "2026-05-13T14:47:00.000Z"
}
```

El `device_status` se actualiza a `pending_ota`. El agente edge recibe la instrucción OTA vía MQTT topic retained `config/{device_id}/ota`. Al completar, el agente publica en `health/{device_id}` y el status vuelve a `active`.

#### Errores de OTA

| Código | `error` | Descripción |
|---|---|---|
| 400 | `invalid_version` | El `target_version` no existe en el repositorio de firmware. |
| 409 | `ota_already_pending` | Ya hay una OTA pendiente para este dispositivo. |
| 404 | `device_not_found` | Dispositivo no encontrado. |
| 403 | `device_not_in_country` | El dispositivo no pertenece al `country_code` del admin. |

---

## 4. Aislamiento Multi-Tenant

Todas las queries a PostgreSQL incluyen `WHERE country_code = $X` donde `$X` proviene del header `X-Country-Code` inyectado por el API Gateway. No existe ninguna ruta que permita acceder a dispositivos de otro país.

```sql
-- Ejemplo de query protegida
SELECT * FROM devices
WHERE country_code = $1
AND status = $2
ORDER BY created_at DESC
LIMIT $3 OFFSET $4;
-- $1 = X-Country-Code header (nunca del query param directo)
```

---

## 5. Métricas Prometheus

| Métrica | Tipo | Labels | Descripción |
|---|---|---|---|
| `devices_total` | Gauge | `country_code`, `status` | Total de dispositivos por estado. |
| `devices_request_duration_seconds` | Histogram | `method`, `endpoint` | Latencia de operaciones CRUD. |
| `devices_ota_initiated_total` | Counter | `country_code` | OTAs iniciadas. |
| `devices_ota_completed_total` | Counter | `country_code`, `result` | OTAs completadas (success/failed). |
| `devices_heartbeat_missing_total` | Counter | `country_code` | Dispositivos sin heartbeat en > 5 min. |

---

## 6. Referencias

- [openapi/devices-service.yaml](./openapi/devices-service.yaml)
- [almacenamiento-lectura/postgresql-schema.md](../almacenamiento-lectura/postgresql-schema.md)
- [slo-observability.md](./slo-observability.md)
- [ADR-011 — Multi-tenant](../propuesta-arquitectura-hurto-vehiculos.md#adr-011--multi-tenant-por-país-con-aislamiento-de-datos)
