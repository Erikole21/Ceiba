# PostgreSQL + PostGIS — Schema

**Componente:** Almacenamiento de lectura — capa PostgreSQL  
**Versión del documento:** 1.0  
**Referencia:** [adr-postgresql-partitioning.md](./adr-postgresql-partitioning.md) · [postgresql-ha.md](./postgresql-ha.md) · [debezium-cdc.md](./debezium-cdc.md)

> **Nota de coordinación:** Este documento es la **fuente de verdad** del schema PostgreSQL. El documento [docs/backbone-procesamiento/postgresql-schema.md](../backbone-procesamiento/postgresql-schema.md) referencia este schema para la perspectiva de escritura. Cualquier cambio de schema debe reflejarse en ambos documentos.

---

## 1. Extensiones Requeridas

```sql
-- Extensiones que deben estar activas antes de crear las tablas
CREATE EXTENSION IF NOT EXISTS postgis;          -- Tipos y funciones geoespaciales
CREATE EXTENSION IF NOT EXISTS pg_partman;       -- Gestión automática de particiones
CREATE EXTENSION IF NOT EXISTS pgcrypto;         -- Cifrado de campos PII (DNI propietario)
CREATE EXTENSION IF NOT EXISTS btree_gist;       -- Índices GiST en columnas escalares (para exclusion constraints)
```

---

## 2. Tabla `vehicle_events` — Eventos de Avistamiento

### 2.1 DDL

```sql
-- Tabla principal (particionada por country_code, ADR-PG-01)
CREATE TABLE vehicle_events (
    event_id            UUID            NOT NULL,          -- UUID v5: device_id+plate+monotonic_ts
    country_code        CHAR(2)         NOT NULL,          -- ISO 3166-1 alpha-2 (discriminador de tenant)
    device_id           TEXT            NOT NULL,          -- Identificador del dispositivo de borde
    plate_raw           TEXT            NOT NULL,          -- Matrícula tal como la leyó el ANPR
    plate_normalized    TEXT            NOT NULL,          -- Matrícula normalizada (mayúsculas, sin guiones)
    event_ts            TIMESTAMPTZ     NOT NULL,          -- Timestamp del evento (wall-clock del dispositivo)
    received_ts         TIMESTAMPTZ     NOT NULL DEFAULT now(), -- Timestamp de recepción en cloud
    confidence          NUMERIC(5,2)    NOT NULL,          -- Score de confianza ANPR (0.00-100.00)
    location            GEOGRAPHY(POINT, 4326),            -- Coordenadas GPS (PostGIS); NULL si coordenadas inválidas (CR-03)
    geocoding_status    TEXT            NOT NULL DEFAULT 'valid', -- valid | INVALID_COORDINATES | MISSING
    image_uri           TEXT,                              -- URI del objeto en S3-API (puede ser NULL si no se subió)
    thumbnail_uri       TEXT,                              -- URI del thumbnail en S3-API
    clock_uncertain     BOOLEAN         NOT NULL DEFAULT false, -- true si drift NTP > umbral
    image_unavailable   BOOLEAN         NOT NULL DEFAULT false, -- true si imagen fue rotada antes de subir
    enrichment_version  SMALLINT        NOT NULL DEFAULT 1,-- Versión del pipeline de enriquecimiento
    data_lifecycle_status TEXT          NOT NULL DEFAULT 'active', -- active | forgotten (right-to-be-forgotten, CA-19)
    extensions          JSONB,                             -- Campos adicionales por país
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT now(),
    CONSTRAINT vehicle_events_pkey PRIMARY KEY (event_id, country_code),
    CONSTRAINT vehicle_events_geocoding_status_check
        CHECK (geocoding_status IN ('valid', 'INVALID_COORDINATES', 'MISSING')),
    CONSTRAINT vehicle_events_lifecycle_status_check
        CHECK (data_lifecycle_status IN ('active', 'forgotten'))
) PARTITION BY LIST (country_code);

-- Partición Colombia
CREATE TABLE vehicle_events_co
    PARTITION OF vehicle_events
    FOR VALUES IN ('CO');

-- Partición México
CREATE TABLE vehicle_events_mx
    PARTITION OF vehicle_events
    FOR VALUES IN ('MX');

-- Partición Perú
CREATE TABLE vehicle_events_pe
    PARTITION OF vehicle_events
    FOR VALUES IN ('PE');

-- Partición para países en piloto (sin partición dedicada aún)
CREATE TABLE vehicle_events_other
    PARTITION OF vehicle_events
    DEFAULT;
```

### 2.2 Índices

```sql
-- IDX-VE-01: Búsqueda por matrícula normalizada + país (patrón de acceso más frecuente)
CREATE INDEX idx_ve_plate_country
    ON vehicle_events (country_code, plate_normalized);
-- Query de ejemplo:
-- SELECT * FROM vehicle_events
-- WHERE country_code = 'CO' AND plate_normalized = 'ABC123'
-- ORDER BY event_ts DESC LIMIT 50;

-- IDX-VE-02: Trayectoria temporal de un vehículo
CREATE INDEX idx_ve_plate_ts
    ON vehicle_events (country_code, plate_normalized, event_ts DESC);
-- Query de ejemplo:
-- SELECT event_id, event_ts, location, image_uri
-- FROM vehicle_events
-- WHERE country_code = 'CO' AND plate_normalized = 'ABC123'
--   AND event_ts BETWEEN '2026-01-01' AND '2026-03-31'
-- ORDER BY event_ts DESC;

-- IDX-VE-03: Búsqueda geoespacial por radio (PostGIS)
-- Solo aplica a filas con location IS NOT NULL y geocoding_status = 'valid'
CREATE INDEX idx_ve_location
    ON vehicle_events USING GIST (location)
    WHERE location IS NOT NULL;
-- Query de ejemplo (avistamientos en radio de 10 km de una coordenada, sobre 1 M de registros):
-- SELECT event_id, plate_normalized, event_ts
-- FROM vehicle_events
-- WHERE country_code = 'CO'
--   AND geocoding_status = 'valid'
--   AND ST_DWithin(
--         location,
--         ST_MakePoint(-74.0721, 4.7109)::geography,
--         10000  -- metros (10 km)
--       )
-- ORDER BY event_ts DESC;
-- Resultado esperado: p95 < 200 ms con el índice GiST activo.
-- Para coordenadas inválidas (lat > 90 o lon > 180), el Enrichment Service inserta
-- location = NULL y geocoding_status = 'INVALID_COORDINATES' sin lanzar excepción PostGIS.
-- La métrica projection.invalid_coordinates se incrementa en esos casos.

-- IDX-VE-04: Búsqueda por device_id (para diagnóstico de dispositivos)
CREATE INDEX idx_ve_device
    ON vehicle_events (country_code, device_id, event_ts DESC);
-- Query de ejemplo:
-- SELECT event_id, plate_normalized, event_ts, confidence
-- FROM vehicle_events
-- WHERE country_code = 'CO' AND device_id = 'dev-co-001'
--   AND event_ts > now() - INTERVAL '24 hours'
-- ORDER BY event_ts DESC;

-- IDX-VE-05: Cobertura para received_ts (monitoreo de lag de ingestión)
CREATE INDEX idx_ve_received
    ON vehicle_events (country_code, received_ts DESC);
-- Query de ejemplo:
-- SELECT count(*) FROM vehicle_events
-- WHERE country_code = 'CO' AND received_ts > now() - INTERVAL '5 minutes';
```

---

## 3. Tabla `stolen_vehicles` — Vehículos Hurtados Canónicos

### 3.1 DDL

```sql
-- Tabla canónica de vehículos hurtados (no particionada: cardinalidad baja, acceso por PK)
CREATE TABLE stolen_vehicles (
    country_code        CHAR(2)         NOT NULL,          -- ISO 3166-1 alpha-2
    plate_normalized    TEXT            NOT NULL,          -- Matrícula normalizada (sin guiones)
    -- Campos obligatorios del modelo canónico (9 campos requeridos)
    stolen_at           TIMESTAMPTZ     NOT NULL,          -- Fecha y hora del hurto
    stolen_location     TEXT            NOT NULL,          -- Lugar del hurto (texto libre del adapter)
    owner_id_hash       TEXT            NOT NULL,          -- HMAC-SHA256(DNI propietario, sal_por_país)
    brand               TEXT            NOT NULL,          -- Marca del vehículo
    vehicle_class       TEXT            NOT NULL,          -- Clase: automóvil, motocicleta, camión, etc.
    line                TEXT            NOT NULL,          -- Línea/modelo específico
    color               TEXT            NOT NULL,          -- Color del vehículo
    model_year          SMALLINT        NOT NULL,          -- Año del modelo
    -- Control de sincronización y versiones
    schema_version      SMALLINT        NOT NULL DEFAULT 1,-- Versión del modelo canónico
    source_country_id   TEXT,                              -- ID del registro en la BD policial origen
    status              TEXT            NOT NULL DEFAULT 'stolen', -- stolen | recovered | cancelled
    status_changed_at   TIMESTAMPTZ,                       -- Timestamp del último cambio de estado
    -- Campos adicionales por país (extensible sin cambio de schema)
    extensions          JSONB,                             -- {"cedula_extra": "...", "descripcion": "..."}
    -- Metadatos de sincronización
    synced_at           TIMESTAMPTZ     NOT NULL DEFAULT now(),
    synced_by_adapter   TEXT            NOT NULL,          -- Nombre del country adapter
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT now(),
    CONSTRAINT stolen_vehicles_pkey PRIMARY KEY (country_code, plate_normalized),
    CONSTRAINT stolen_vehicles_status_check
        CHECK (status IN ('stolen', 'recovered', 'cancelled'))
);

-- Trigger para actualizar updated_at automáticamente
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_stolen_vehicles_updated_at
    BEFORE UPDATE ON stolen_vehicles
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### 3.2 Índices

```sql
-- IDX-SV-01: Acceso por clave primaria (implícito en PRIMARY KEY)
-- Query de ejemplo:
-- SELECT * FROM stolen_vehicles
-- WHERE country_code = 'CO' AND plate_normalized = 'ABC123';

-- IDX-SV-02: Vehículos hurtados activos por país (para sincronización con Redis y Bloom filter)
CREATE INDEX idx_sv_status_country
    ON stolen_vehicles (country_code, status)
    WHERE status = 'stolen';
-- Query de ejemplo:
-- SELECT plate_normalized FROM stolen_vehicles
-- WHERE country_code = 'CO' AND status = 'stolen'
-- ORDER BY synced_at DESC;

-- IDX-SV-03: Búsqueda por adapter y timestamp (para replay y auditoría de sincronización)
CREATE INDEX idx_sv_adapter_sync
    ON stolen_vehicles (synced_by_adapter, synced_at DESC);
-- Query de ejemplo:
-- SELECT plate_normalized, synced_at, status FROM stolen_vehicles
-- WHERE synced_by_adapter = 'adapter-co-v2' AND synced_at > '2026-01-01';

-- IDX-SV-04: Búsqueda en extensions JSONB (para consultas por campos específicos de cada país)
CREATE INDEX idx_sv_extensions
    ON stolen_vehicles USING GIN (extensions);
-- Query de ejemplo:
-- SELECT plate_normalized FROM stolen_vehicles
-- WHERE country_code = 'CO' AND extensions @> '{"color_secundario": "negro"}';
```

---

## 4. Tabla `incidents` — Ciclo de Vida de Recuperación

### 4.1 DDL

```sql
CREATE TABLE incidents (
    incident_id         UUID            NOT NULL DEFAULT gen_random_uuid(),
    country_code        CHAR(2)         NOT NULL,
    plate_normalized    TEXT            NOT NULL,
    -- Estado del incidente
    status              TEXT            NOT NULL DEFAULT 'open',
    -- Ciclo de vida: open → in_progress → recovered | closed
    opened_at           TIMESTAMPTZ     NOT NULL DEFAULT now(),
    in_progress_at      TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    -- Datos operacionales
    assigned_unit       TEXT,                              -- Unidad policial asignada
    assigned_officer_id TEXT,                              -- ID del oficial asignado (opaco, de Keycloak)
    last_known_event_id UUID,                              -- FK al último event_id en vehicle_events
    last_known_location GEOGRAPHY(POINT, 4326),            -- Última ubicación conocida del vehículo
    last_known_ts       TIMESTAMPTZ,                       -- Timestamp de la última detección
    resolution_type     TEXT,                              -- recovered | not_found | cancelled | error
    resolution_notes    TEXT,                              -- Notas del oficial al cerrar
    -- Metadatos
    created_by          TEXT            NOT NULL,          -- ID del oficial que abrió el incidente (Keycloak)
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT now(),
    CONSTRAINT incidents_pkey PRIMARY KEY (incident_id),
    CONSTRAINT incidents_status_check
        CHECK (status IN ('open', 'in_progress', 'recovered', 'closed')),
    CONSTRAINT incidents_resolution_check
        CHECK (resolution_type IS NULL OR
               resolution_type IN ('recovered', 'not_found', 'cancelled', 'error'))
);

CREATE TRIGGER trg_incidents_updated_at
    BEFORE UPDATE ON incidents
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### 4.2 Índices

```sql
-- IDX-INC-01: Incidentes abiertos por país (vista operacional del dashboard policial)
CREATE INDEX idx_inc_status_country
    ON incidents (country_code, status, opened_at DESC)
    WHERE status IN ('open', 'in_progress');
-- Query de ejemplo:
-- SELECT incident_id, plate_normalized, status, last_known_location, last_known_ts
-- FROM incidents
-- WHERE country_code = 'CO' AND status IN ('open', 'in_progress')
-- ORDER BY opened_at DESC;

-- IDX-INC-02: Historial por matrícula
CREATE INDEX idx_inc_plate
    ON incidents (country_code, plate_normalized, opened_at DESC);
-- Query de ejemplo:
-- SELECT * FROM incidents
-- WHERE country_code = 'CO' AND plate_normalized = 'ABC123'
-- ORDER BY opened_at DESC;

-- IDX-INC-03: Incidentes asignados a un oficial
CREATE INDEX idx_inc_officer
    ON incidents (assigned_officer_id, status)
    WHERE assigned_officer_id IS NOT NULL;
```

---

## 5. Tabla `audit_log` — Log de Auditoría Inmutable

### 5.1 DDL

```sql
-- Tabla append-only: no se permiten UPDATE ni DELETE (enforced por trigger)
CREATE TABLE audit_log (
    audit_id            BIGSERIAL       NOT NULL,
    country_code        CHAR(2)         NOT NULL,
    actor_id            TEXT            NOT NULL,          -- ID del usuario (Keycloak sub)
    actor_ip            INET            NOT NULL,          -- IP del cliente en el momento del acceso
    event_type          TEXT            NOT NULL,          -- search_plate | view_event | view_incident | rtbf | etc.
    entity_type         TEXT            NOT NULL,          -- plate | event | incident | vehicle
    entity_id           TEXT            NOT NULL,          -- plate_normalized o event_id o incident_id
    payload             JSONB,                             -- Parámetros de la consulta (sin PII sensible)
    result_count        INTEGER,                           -- Número de registros retornados
    duration_ms         INTEGER,                           -- Duración de la operación en ms
    logged_at           TIMESTAMPTZ     NOT NULL DEFAULT now(),
    CONSTRAINT audit_log_pkey PRIMARY KEY (audit_id)
);

-- Trigger para prevenir modificaciones (inmutabilidad)
CREATE OR REPLACE FUNCTION prevent_audit_modifications()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'El audit_log es inmutable. No se permiten UPDATE ni DELETE.';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_no_update
    BEFORE UPDATE ON audit_log
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modifications();

CREATE TRIGGER trg_audit_no_delete
    BEFORE DELETE ON audit_log
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modifications();
```

### 5.2 Índices

```sql
-- IDX-AL-01: Auditoría por actor y tiempo (investigación de uso sospechoso)
CREATE INDEX idx_al_actor_time
    ON audit_log (actor_id, logged_at DESC);
-- Query de ejemplo:
-- SELECT event_type, entity_type, entity_id, logged_at
-- FROM audit_log
-- WHERE actor_id = 'keycloak-user-uuid' AND logged_at > now() - INTERVAL '7 days'
-- ORDER BY logged_at DESC;

-- IDX-AL-02: Auditoría por entidad (quién accedió a una placa específica)
CREATE INDEX idx_al_resource
    ON audit_log (country_code, entity_type, entity_id, logged_at DESC);
-- Query de ejemplo:
-- SELECT actor_id, actor_ip, event_type, logged_at
-- FROM audit_log
-- WHERE country_code = 'CO' AND entity_type = 'plate' AND entity_id = 'ABC123'
-- ORDER BY logged_at DESC;

-- IDX-AL-03: Auditoría por país y tiempo (reporte de cumplimiento)
CREATE INDEX idx_al_country_time
    ON audit_log (country_code, logged_at DESC);
```

---

## 6. Política de Retención y pg_partman

La retención de datos se gestiona con `pg_partman` y se documenta detalladamente en [data-lifecycle.md](./data-lifecycle.md). Configuración de referencia:

```sql
-- Configurar pg_partman para sub-particionamiento de vehicle_events_co por mes
-- (se activa cuando la partición supera 500 GB, según ADR-PG-01)
SELECT partman.create_parent(
    p_parent_table := 'public.vehicle_events_co',
    p_control := 'event_ts',
    p_type := 'range',
    p_interval := '1 month',
    p_premake := 3,          -- Pre-crear 3 meses futuros
    p_retention := '24 months', -- Retener 24 meses
    p_retention_keep_table := false -- Eliminar particiones antiguas
);

-- Ejecutar mantenimiento periódico (cron vía pg_cron o job externo)
SELECT partman.run_maintenance(p_analyze := true);
```

---

## 7. Configuración de Roles y Permisos

```sql
-- Role de escritura (usado por backbone-procesamiento: Enrichment, Canonical Vehicles)
CREATE ROLE app_writer LOGIN PASSWORD '***';
GRANT INSERT, UPDATE ON vehicle_events, stolen_vehicles, incidents TO app_writer;
GRANT INSERT ON audit_log TO app_writer;
GRANT USAGE, SELECT ON SEQUENCE audit_log_audit_id_seq TO app_writer;

-- Role de lectura (usado por api-frontend-analitica: search-service, analytics-service)
CREATE ROLE app_reader LOGIN PASSWORD '***';
GRANT SELECT ON vehicle_events, stolen_vehicles, incidents, audit_log TO app_reader;

-- Role de Debezium (solo lectura del WAL, sin acceso a datos)
CREATE ROLE debezium_replication REPLICATION LOGIN PASSWORD '***';
GRANT CONNECT ON DATABASE antihurto TO debezium_replication;

-- Role de pg_partman (mantenimiento de particiones)
CREATE ROLE partman_maintenance LOGIN PASSWORD '***';
GRANT ALL ON TABLE partman.part_config TO partman_maintenance;
```

---

## 8. Notas de Coordinación

- **backbone-procesamiento:** El `Enrichment Service` escribe en `vehicle_events` con el role `app_writer`. El `Canonical Vehicles Service` hace upsert en `stolen_vehicles`. El `incident-service` escribe en `incidents`. Todos los accesos de usuario quedan registrados automáticamente en `audit_log` vía el interceptor del API Gateway.
- **sincronizacion-paises:** El `Canonical Vehicles Service` (parte de ese change) escribe en `stolen_vehicles` usando el campo `synced_by_adapter` para identificar el adaptador de país de origen. El schema canónico aquí definido — incluyendo `extensions JSONB` y `schema_version` — es el contrato de escritura que ese change debe respetar. Cualquier campo nuevo en un país específico debe ir en `extensions`, no en columnas nuevas.
- **identidad-seguridad:** El claim `country_code` del JWT emitido por Keycloak es el discriminador de tenant propagado hasta la capa de datos. El `search-service` lo extrae del token y lo usa en cada consulta como `WHERE country_code = :cc` y en el `SET LOCAL` de la transacción. La configuración concreta del realm y la emisión del claim son responsabilidad de ese change; este schema solo define el requisito.
- **api-frontend-analitica:** El `search-service` usa el role `app_reader` con `SET LOCAL country_code = 'CO'` para reforzar el aislamiento de tenant en cada transacción.
- **PostGIS:** La columna `location` usa tipo `GEOGRAPHY(POINT, 4326)` (no `GEOMETRY`) para que `ST_DWithin` opere en metros reales sin necesidad de proyección previa. La columna es **nullable**: eventos con coordenadas inválidas se almacenan con `location = NULL` y `geocoding_status = 'INVALID_COORDINATES'`.

---

## 9. Validación de Eventos en el Pipeline — Rechazo y Dead-Letter (CR-01, CR-02, CR-03)

Las validaciones de integridad ocurren en el **Enrichment Service** antes de proyectar a PostgreSQL. Los eventos rechazados no se persisten; se redirigen al dead-letter topic correspondiente.

| Criterio | Condición de rechazo | Dead-letter topic | Código de motivo |
|---|---|---|---|
| **CR-01** | `event_id` nulo o vacío | `vehicle.events.dlq` | `MISSING_EVENT_ID` |
| **CR-02** | `country_code` nulo o vacío | `vehicle.events.dlq` | `MISSING_COUNTRY_CODE` |
| **CR-03** | `lat` fuera de rango [-90, 90] o `lon` fuera de [-180, 180] | No se rechaza; se inserta con `location = NULL`, `geocoding_status = 'INVALID_COORDINATES'` | métrica `projection.invalid_coordinates` |

**Comportamiento en PostgreSQL ante `event_id` nulo (CR-01):**

```sql
-- El constraint NOT NULL sobre event_id garantiza que PostgreSQL rechaza la inserción
-- si el Enrichment Service omite la validación previa:
ALTER TABLE vehicle_events ALTER COLUMN event_id SET NOT NULL;
-- La constraint PRIMARY KEY (event_id, country_code) también previene duplicados.
```

**Comportamiento ante duplicados (CA-01):**

```sql
-- El Enrichment Service usa INSERT ... ON CONFLICT DO NOTHING para idempotencia:
INSERT INTO vehicle_events (event_id, country_code, ...)
VALUES ('uuid-del-evento', 'CO', ...)
ON CONFLICT (event_id, country_code) DO NOTHING;
-- Cuando DO NOTHING se activa, el servicio incrementa la métrica:
-- projection.duplicate_skipped{country_code="CO"} += 1
```

La métrica `projection.duplicate_skipped` (contador Prometheus con label `country_code`) permite detectar reinicios del pipeline que estén reproduciendo eventos ya procesados.

**Comportamiento ante coordenadas inválidas (CR-03):**

```python
# Pseudocódigo del Enrichment Service
if not (-90 <= lat <= 90 and -180 <= lon <= 180):
    location = None
    geocoding_status = 'INVALID_COORDINATES'
    metrics.increment('projection.invalid_coordinates', country_code=event.country_code)
else:
    location = f'POINT({lon} {lat})'
    geocoding_status = 'valid'
```

Los eventos con `geocoding_status = 'INVALID_COORDINATES'` se almacenan normalmente en PostgreSQL y OpenSearch (sin campo de coordenadas); no bloquean el pipeline.
