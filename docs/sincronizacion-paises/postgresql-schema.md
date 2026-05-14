# Esquema PostgreSQL — canonical_vehicles

**Change:** `sincronizacion-paises`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Propósito

Este documento especifica el DDL completo de la tabla `canonical_vehicles`, sus índices, la estrategia de particionamiento por `country_code`, la columna JSONB `extensions` y el trigger `updated_at`. Es la fuente de verdad para el esquema de datos del Pilar 4 en PostgreSQL.

**Nota de coordinación:** Esta tabla coexiste con las tablas de eventos de tránsito definidas en [`docs/almacenamiento-lectura/postgresql-schema.md`](../almacenamiento-lectura/postgresql-schema.md). Los cambios a `canonical_vehicles` deben coordinarse con el equipo de `almacenamiento-lectura` cuando afecten índices o particionamiento usado por el Matcher Service.

---

## 2. DDL — Tabla `canonical_vehicles`

```sql
-- Extensión para UUID (si no está habilitada)
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Enum de estado del vehículo
CREATE TYPE vehicle_status AS ENUM ('STOLEN', 'RECOVERED');

-- Tabla principal (particionada por country_code)
CREATE TABLE canonical_vehicles (
    -- Identificadores
    id                  UUID            NOT NULL DEFAULT uuid_generate_v4(),
    country_code        CHAR(2)         NOT NULL,
    plate               VARCHAR(20)     NOT NULL,

    -- Estado
    status              vehicle_status  NOT NULL DEFAULT 'STOLEN',

    -- 9 campos mandatorios del modelo canónico
    stolen_date         TIMESTAMPTZ     NOT NULL,
    stolen_location     VARCHAR(500)    NOT NULL,
    owner_id            VARCHAR(50)     NOT NULL,
    brand               VARCHAR(100)    NOT NULL,
    class               VARCHAR(50)     NOT NULL,
    line                VARCHAR(100)    NOT NULL,
    color               VARCHAR(50)     NOT NULL,
    model               CHAR(4)         NOT NULL CHECK (model ~ '^\d{4}$'),

    -- Metadatos de origen
    source_system       VARCHAR(100)    NOT NULL,
    schema_version      VARCHAR(10)     NOT NULL,
    last_event_id       UUID            NOT NULL,

    -- Extensiones por país (campos adicionales autorizados)
    extensions          JSONB           NOT NULL DEFAULT '{}',

    -- Timestamps de control
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    recovered_at        TIMESTAMPTZ,    -- Se puebla cuando status = RECOVERED

    -- Restricciones
    CONSTRAINT pk_canonical_vehicles PRIMARY KEY (country_code, plate),
    CONSTRAINT chk_country_code CHECK (country_code ~ '^[A-Z]{2}$'),
    CONSTRAINT chk_plate_not_empty CHECK (LENGTH(TRIM(plate)) > 0),
    CONSTRAINT chk_recovered_at CHECK (
        (status = 'RECOVERED' AND recovered_at IS NOT NULL)
        OR (status = 'STOLEN' AND recovered_at IS NULL)
    )
) PARTITION BY LIST (country_code);
```

---

## 3. Particiones por País

Cada país activo tiene su propia partición. Las particiones se crean durante el onboarding del país.

```sql
-- Partición Colombia
CREATE TABLE canonical_vehicles_co
    PARTITION OF canonical_vehicles
    FOR VALUES IN ('CO');

-- Partición Venezuela
CREATE TABLE canonical_vehicles_ve
    PARTITION OF canonical_vehicles
    FOR VALUES IN ('VE');

-- Partición México
CREATE TABLE canonical_vehicles_mx
    PARTITION OF canonical_vehicles
    FOR VALUES IN ('MX');

-- Partición Argentina
CREATE TABLE canonical_vehicles_ar
    PARTITION OF canonical_vehicles
    FOR VALUES IN ('AR');

-- Partición DEFAULT (captura países en fase de onboarding)
CREATE TABLE canonical_vehicles_default
    PARTITION OF canonical_vehicles
    DEFAULT;
```

**Procedimiento para agregar un nuevo país:**

```sql
-- Ejecutar durante el onboarding del país {CC}
CREATE TABLE canonical_vehicles_{cc_lower}
    PARTITION OF canonical_vehicles
    FOR VALUES IN ('{CC}');
```

---

## 4. Índices

Los índices se crean sobre cada partición. PostgreSQL crea índices de partición automáticamente si se crean sobre la tabla particionada principal.

```sql
-- Índice para lookups por plate dentro de un país (usado por el Matcher Service)
-- La PRIMARY KEY ya cubre (country_code, plate), pero se agrega índice inverso
-- para búsquedas por plate sin country_code conocido (casos de uso del search-service)
CREATE INDEX idx_canonical_vehicles_plate
    ON canonical_vehicles (plate, country_code);

-- Índice para filtrar por status (usado por el Edge Distribution Service al
-- construir el BF: SELECT plate FROM canonical_vehicles WHERE status = 'STOLEN')
CREATE INDEX idx_canonical_vehicles_status
    ON canonical_vehicles (country_code, status)
    INCLUDE (plate);

-- Índice GIN para búsqueda dentro de extensions (soporte a consultas analíticas)
CREATE INDEX idx_canonical_vehicles_extensions
    ON canonical_vehicles USING GIN (extensions jsonb_path_ops);

-- Índice para consultas por rango de stolen_date (reporting, purge de datos viejos)
CREATE INDEX idx_canonical_vehicles_stolen_date
    ON canonical_vehicles (country_code, stolen_date DESC);

-- Índice para el source_system (auditoría y diagnóstico por adaptador)
CREATE INDEX idx_canonical_vehicles_source_system
    ON canonical_vehicles (country_code, source_system);
```

---

## 5. Trigger `updated_at`

```sql
-- Función del trigger
CREATE OR REPLACE FUNCTION fn_update_canonical_vehicles_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger sobre la tabla particionada
CREATE TRIGGER trg_canonical_vehicles_updated_at
    BEFORE UPDATE ON canonical_vehicles
    FOR EACH ROW
    EXECUTE FUNCTION fn_update_canonical_vehicles_timestamp();
```

**Nota:** En PostgreSQL, los triggers sobre tablas particionadas se propagan automáticamente a todas las particiones a partir de la versión 13. Para versiones anteriores, el trigger debe crearse en cada partición individualmente.

---

## 6. Operación de Upsert

El Canonical Vehicles Service ejecuta el siguiente upsert (CA-06):

```sql
INSERT INTO canonical_vehicles (
    country_code, plate, status, stolen_date, stolen_location,
    owner_id, brand, class, line, color, model,
    source_system, schema_version, last_event_id, extensions
)
VALUES (
    $1, $2, $3, $4, $5,
    $6, $7, $8, $9, $10, $11,
    $12, $13, $14, $15
)
ON CONFLICT (country_code, plate) DO UPDATE SET
    status          = EXCLUDED.status,
    stolen_date     = EXCLUDED.stolen_date,
    stolen_location = EXCLUDED.stolen_location,
    owner_id        = EXCLUDED.owner_id,
    brand           = EXCLUDED.brand,
    class           = EXCLUDED.class,
    line            = EXCLUDED.line,
    color           = EXCLUDED.color,
    model           = EXCLUDED.model,
    source_system   = EXCLUDED.source_system,
    schema_version  = EXCLUDED.schema_version,
    last_event_id   = EXCLUDED.last_event_id,
    extensions      = EXCLUDED.extensions,
    recovered_at    = CASE
                        WHEN EXCLUDED.status = 'RECOVERED' THEN NOW()
                        ELSE NULL
                      END
-- El trigger fn_update_canonical_vehicles_timestamp actualiza updated_at automáticamente
RETURNING id, country_code, plate, status, updated_at;
```

La operación es idempotente: ejecutar el mismo evento dos veces produce el mismo estado final (CA-06).

---

## 7. Consultas Frecuentes

### 7.1 Obtener todas las placas STOLEN de un país (para construir el BF)

```sql
SELECT plate
FROM canonical_vehicles
WHERE country_code = $1
  AND status = 'STOLEN';
-- Usa idx_canonical_vehicles_status (INCLUDE plate)
-- Para 1M de placas: estimado < 2 s en hardware estándar
```

### 7.2 Verificar existencia de una placa (Canonical Vehicles Service - evento RECOVERED)

```sql
SELECT id, status
FROM canonical_vehicles
WHERE country_code = $1
  AND plate = $2;
-- Usa PRIMARY KEY (country_code, plate) — O(log n)
```

### 7.3 Reconciliación Redis (leer todos los registros STOLEN para re-poblar)

```sql
SELECT country_code, plate
FROM canonical_vehicles
WHERE status = 'STOLEN'
ORDER BY country_code, plate;
-- Usa idx_canonical_vehicles_status con scan por partición
```

---

## 8. Control de Acceso (Row Level Security)

El campo `owner_id` contiene datos personales (ver [`data-sovereignty.md`](./data-sovereignty.md)). Se implementa control de acceso a nivel de rol:

```sql
-- Rol para el Canonical Vehicles Service (lectura/escritura sin restricción)
GRANT SELECT, INSERT, UPDATE ON canonical_vehicles TO role_canonical_service;

-- Rol para el Edge Distribution Service (solo SELECT de plate + status)
GRANT SELECT (country_code, plate, status, updated_at) ON canonical_vehicles TO role_edge_distribution;

-- Rol para analítica (sin acceso a owner_id)
GRANT SELECT (
    id, country_code, plate, status, stolen_date, stolen_location,
    brand, class, line, color, model, source_system, schema_version,
    extensions, created_at, updated_at
) ON canonical_vehicles TO role_analytics;

-- Rol para el Matcher Service (solo plate + status por índice)
GRANT SELECT (country_code, plate, status) ON canonical_vehicles TO role_matcher_service;
```

La columna `owner_id` solo es accesible para `role_canonical_service` y `role_auditor`.

---

## 9. Política de Retención y Purge

Los registros con `status = 'RECOVERED'` y `recovered_at` anterior a N días (configurable, valor de referencia: 365 días) pueden archivarse y eliminarse de la tabla activa.

```sql
-- Job de purge (ejecutar periódicamente, p.ej. mensual)
DELETE FROM canonical_vehicles
WHERE status = 'RECOVERED'
  AND recovered_at < NOW() - INTERVAL '365 days';
-- Ejecutar en batches para evitar lock prolongado:
-- DELETE ... WHERE ctid IN (SELECT ctid FROM canonical_vehicles WHERE ... LIMIT 10000)
```

---

## 10. Estimación de Tamaño

| País | Registros STOLEN estimados | Tamaño estimado tabla (sin índices) | Con índices |
|---|---|---|---|
| Colombia | ~500 000 | ~300 MB | ~800 MB |
| Venezuela | ~200 000 | ~120 MB | ~320 MB |
| México | ~1 000 000 | ~600 MB | ~1.6 GB |
| 10 países (promedio 200K c/u) | ~2 000 000 | ~1.2 GB | ~3.2 GB |

La partición por `country_code` permite manejar cada partición de forma independiente (vacuum, índices, backup).

---

## 11. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [`canonical-model.md`](./canonical-model.md) | Definición semántica de cada columna |
| [`canonical-vehicles-service.md`](./canonical-vehicles-service.md) | Servicio que ejecuta el upsert |
| [`edge-distribution-service.md`](./edge-distribution-service.md) | Consume `plate` y `status` para construir el BF |
| [`data-sovereignty.md`](./data-sovereignty.md) | Política de acceso a `owner_id` |
| [`docs/almacenamiento-lectura/postgresql-schema.md`](../almacenamiento-lectura/postgresql-schema.md) | Esquema de eventos de tránsito (tabla diferente) |
