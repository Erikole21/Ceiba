# Almacenamiento de Lectura — Visión General

**Componente:** Capa de persistencia y almacenamiento de lectura  
**Versión del documento:** 1.0  
**Referencia arquitectural:** [Propuesta de Arquitectura §3.5](../propuesta-arquitectura-hurto-vehiculos.md#35-almacenamiento-de-lectura) · Pilar 6 — Hot/Cold path separados

---

## 1. Propósito

La capa de almacenamiento de lectura define los **cuatro almacenes especializados** que materializan las proyecciones del Sistema Anti-Hurto de Vehículos siguiendo el patrón CQRS pragmático (ADR-008). Cada almacén está optimizado para un tipo de acceso distinto:

| Almacén | Caso de uso principal |
|---|---|
| **PostgreSQL + PostGIS** | Consultas operacionales, trayectorias geoespaciales, vehículos hurtados canónicos, audit log |
| **OpenSearch** | Búsqueda de matrículas (exacta + fuzzy), filtros temporales, agregaciones por país |
| **ClickHouse** | Analítica columnar masiva, heatmaps H3, tendencias por zona/ciudad/país |
| **Redis** | Cache de hot-list de placas hurtadas, sesiones, rate-limiting, pub/sub de alertas |

Este documento establece el **contrato de persistencia**: lo que `backbone-procesamiento` escribe y `api-frontend-analitica` lee.

> **Capa de exposición:** la implementación de los contratos de lectura aquí definidos está en [`docs/api-frontend-analitica/`](../api-frontend-analitica/overview.md). Los puertos hexagonales (ADR-005) que implementan esos contratos son:
>
> | Puerto | Microservicio implementador | Almacén |
> |---|---|---|
> | `EventsIndexPort` | `search-service` ([spec](../api-frontend-analitica/search-service.md)) | OpenSearch |
> | `EventsRepositoryPort` | `search-service` ([spec](../api-frontend-analitica/search-service.md)) | PostgreSQL + PostGIS |
> | `ObjectStoragePort` | `search-service` ([spec](../api-frontend-analitica/search-service.md)) | Object Storage (S3-API / MinIO) |
> | `AlertsKafkaPort` | `alerts-service` ([spec](../api-frontend-analitica/alerts-service.md)) | Kafka `vehicle.alerts` |
> | `AlertsStatePort` | `alerts-service` ([spec](../api-frontend-analitica/alerts-service.md)) | Redis (estado conexiones WS) |
> | `AnalyticsRepositoryPort` | `analytics-service` ([spec](../api-frontend-analitica/analytics-service.md)) | ClickHouse vistas materializadas |
> | `CountrySyncPort` | `incident-service` ([spec](../api-frontend-analitica/incident-service.md)) | Kafka / adapter del país (write-back) |

---

## 2. Diagrama de Almacenes y Relaciones

```mermaid
flowchart LR
    subgraph BACKBONE["backbone-procesamiento (escrituras)"]
        ENR[Enrichment Service]
        MATCH[Matcher Service]
        CANON[Canonical Vehicles Service]
        ALERT[Alert Service]
    end

    subgraph STORES["Almacenamiento de Lectura"]
        PG[(PostgreSQL\n+ PostGIS\nevents / vehicles\nincidents / audit)]
        OS[(OpenSearch\nvehicle-events\n-{country}-{YYYY-MM})]
        CH[(ClickHouse\nvehicle_events_daily\nvehicle_events_h3_hourly)]
        RD[(Redis\nstolen:{cc}:{plate}\nttl=24h)]
    end

    subgraph READERS["api-frontend-analitica (lecturas)"]
        SS[search-service]
        AS[analytics-service]
        MS[Matcher Service\nlookup]
        WS[WebSocket\nAlert gateway]
    end

    subgraph CDC["CDC Pipeline"]
        DBZ[Debezium\nConector WAL]
        KAF[Kafka\nvehicle.events.enriched\npg.events.cdc]
    end

    ENR -->|INSERT vehicle_events| PG
    ENR -->|index vehicle-events-*| OS
    ENR -->|publish vehicle.events.enriched| KAF
    CANON -->|upsert stolen_vehicles| PG
    CANON -->|SET stolen:{cc}:{plate}| RD
    ALERT -->|INSERT incidents| PG
    ALERT -->|pub/sub alerts channel| RD

    PG -->|WAL stream| DBZ
    DBZ -->|pg.events.cdc| KAF
    KAF -->|Kafka Engine Table| CH

    SS -->|SELECT + PostGIS| PG
    SS -->|query vehicle-events-*| OS
    AS -->|SELECT aggregations| CH
    MS -->|GET stolen:{cc}:{plate}| RD
    WS -->|SUBSCRIBE alerts| RD
```

**Flujo de escritura:** El `Enrichment Service` produce el evento enriquecido y lo proyecta simultáneamente a PostgreSQL (estado consultable con coordenadas PostGIS) y OpenSearch (índice de búsqueda). Paralelamente, publica en el tópico Kafka `vehicle.events.enriched`. Debezium captura el WAL de PostgreSQL y produce el tópico `pg.events.cdc` que ClickHouse consume mediante Kafka Engine Table.

**Flujo de lectura:** El `search-service` combina OpenSearch (descubrimiento por placa) con PostgreSQL (coordenadas y metadatos). El `analytics-service` consulta ClickHouse para dashboards y mapas de calor. El `Matcher Service` realiza lookups de sub-milisegundo en Redis. Las alertas en tiempo real se propagan vía Redis pub/sub al WebSocket gateway.

---

## 3. Tabla de SLOs por Almacén

| Almacén | Operación | SLO | Percentil | Herramienta de medición |
|---|---|---|---|---|
| **Redis** | GET `stolen:{cc}:{plate}` | < 1 ms | p99 | Prometheus `redis_command_duration_seconds` |
| **Redis** | SET / DEL | < 2 ms | p99 | Prometheus `redis_command_duration_seconds` |
| **OpenSearch** | Búsqueda exacta por placa (`plate_normalized.keyword`) | < 50 ms | p95 | Prometheus `opensearch_search_latency_seconds` |
| **OpenSearch** | Búsqueda fuzzy (fuzziness AUTO) | < 300 ms | p95 | Prometheus `opensearch_search_latency_seconds` |
| **ClickHouse** | Dashboard 7 días (1 ciudad) | < 3 s | p95 | Prometheus `clickhouse_query_duration_seconds` |
| **ClickHouse** | Mapa de calor H3 (país) | < 5 s | p95 | Prometheus `clickhouse_query_duration_seconds` |
| **PostgreSQL** | SELECT por `event_id` (PK) | < 10 ms | p95 | Prometheus `pg_query_duration_seconds` |
| **PostgreSQL** | SELECT trayectoria por placa | < 50 ms | p95 | Prometheus `pg_query_duration_seconds` |
| **PostgreSQL** | SELECT geoespacial (radio 10 km, 1 M registros) | < 200 ms | p95 | Prometheus `pg_query_duration_seconds` |
| **Debezium CDC** | Lag WAL → ClickHouse | < 5 s | p99 | Prometheus `debezium_source_record_lag` |

---

## 4. Decisiones ADR Aplicadas

| ADR | Decisión | Impacto en este componente |
|---|---|---|
| **ADR-005** | Arquitectura hexagonal para servicios sensibles a la nube | PostgreSQL, OpenSearch, ClickHouse y Redis están detrás de puertos/adaptadores en los servicios consumidores (`StolenVehiclesPort`, `VehicleListPort`); permiten sustitución sin reescribir lógica de dominio |
| **ADR-008** | CQRS pragmático con eventos como fuente | Kafka es la fuente de verdad; los cuatro almacenes son proyecciones materializadas. No hay event-sourcing puro: se persiste estado materializado. La consistencia entre proyecciones es eventual |
| **ADR-011** | Multi-tenant por país con aislamiento de datos | `country_code` como discriminador en todas las tablas, índices, tópicos y claves Redis. Particionamiento LIST por `country_code` en PostgreSQL; índices rodantes `vehicle-events-{country_code}-{YYYY-MM}` en OpenSearch; sharding por país en ClickHouse |

Adicionalmente:

- **ADR-PG-01** (local): Particionamiento LIST por `country_code` como primer nivel en PostgreSQL (ver [adr-postgresql-partitioning.md](./adr-postgresql-partitioning.md)).
- **ADR-CH-01** (local): CDC Debezium como canal primario de ingestión en ClickHouse (ver [adr-clickhouse-ingestion.md](./adr-clickhouse-ingestion.md)).
- **ADR-RD-01** (local): Redis Sentinel como topología inicial (ver [adr-redis-topology.md](./adr-redis-topology.md)).

---

## 5. Glosario de Componentes

| Término | Definición |
|---|---|
| **vehicle_events** | Tabla PostgreSQL particionada que almacena cada avistamiento de matrícula enriquecido (coordenadas PostGIS, device_id, confidence, image_uri). Particionada por `country_code`. |
| **stolen_vehicles** | Tabla PostgreSQL canónica de vehículos hurtados; un registro por matrícula por país. Incluye los 9 campos obligatorios + `extensions` JSONB + `schema_version`. |
| **incidents** | Tabla PostgreSQL con el ciclo de vida de recuperación: `open → in_progress → recovered / closed`. |
| **audit_log** | Tabla PostgreSQL append-only que registra cada acceso a datos de placa o vehículo hurtado. Inmutable por política de FK y trigger. |
| **vehicle-events-{country_code}-{YYYY-MM}** | Índice OpenSearch rodante por país y mes. Contiene los mismos eventos que PostgreSQL pero optimizados para búsqueda full-text y fuzzy. |
| **vehicle_events_daily** | Tabla ClickHouse (MergeTree / ReplicatedMergeTree) para analítica diaria. Particionada por mes (`toYYYYMM(event_ts)`). Motor columnar. |
| **vehicle_events_h3_hourly** | Vista materializada ClickHouse que agrega conteos por celda H3 resolución 8 (≈ 0.74 km²) y hora. Base de los heatmaps. |
| **stolen:{cc}:{plate}** | Clave Redis de la hot-list. `cc` = `country_code` en minúsculas; `plate` = matrícula normalizada (mayúsculas, sin guiones). TTL = 24 h. |
| **pg.events.cdc** | Tópico Kafka producido por Debezium desde el WAL de PostgreSQL. Formato payload CDC con `before`/`after` + metadatos LSN. |
| **vehicle.events.enriched** | Tópico Kafka producido por el `Enrichment Service` con el evento completamente enriquecido. Consumido por proyectores de OpenSearch y ClickHouse directo. |
| **StolenVehiclesPort** | Puerto hexagonal (ADR-005) implementado por Redis; permite sustituir Redis por otro almacén de cache sin afectar el Matcher Service. |
| **VehicleListPort** | Puerto hexagonal que abstrae el acceso a la lista canónica de vehículos hurtados en PostgreSQL. |
| **country_code** | Código ISO 3166-1 alpha-2 del país (ej. `CO`, `MX`, `PE`). Discriminador de tenant en todos los almacenes. |
| **h3_index_8** | Índice H3 de resolución 8 que representa la celda hexagonal de aproximadamente 0.74 km² donde ocurrió el evento. |
| **ILM** | Index Lifecycle Management de OpenSearch. Define las fases hot/warm/cold/delete de cada índice rodante. |
| **pg_partman** | Extensión PostgreSQL que automatiza la creación y el mantenimiento de particiones por tiempo y lista. |
| **schema_version** | Campo entero en `stolen_vehicles` que indica la versión del modelo canónico con el que fue sincronizado el registro. |

---

## 6. Índice de Documentos de Este Componente

| Documento | Contenido |
|---|---|
| [adr-postgresql-partitioning.md](./adr-postgresql-partitioning.md) | ADR de estrategia de particionamiento PostgreSQL |
| [adr-clickhouse-ingestion.md](./adr-clickhouse-ingestion.md) | ADR de canal de ingestión ClickHouse |
| [adr-redis-topology.md](./adr-redis-topology.md) | ADR de topología Redis Sentinel vs. Cluster |
| [postgresql-schema.md](./postgresql-schema.md) | DDL descriptivo de las cuatro tablas, índices y ejemplos |
| [postgresql-ha.md](./postgresql-ha.md) | Alta disponibilidad con Patroni y equivalentes gestionados |
| [debezium-cdc.md](./debezium-cdc.md) | Conector Debezium, tópico CDC, payload, Kafka Engine Table |
| [opensearch-schema.md](./opensearch-schema.md) | Mapping del índice, búsqueda fuzzy, aliases, queries de ejemplo |
| [opensearch-ilm.md](./opensearch-ilm.md) | Política ILM, plantillas de índice, rollover, runbook |
| [clickhouse-schema.md](./clickhouse-schema.md) | Tabla principal, Kafka Engine Table, vista materializada, TTL |
| [clickhouse-h3-views.md](./clickhouse-h3-views.md) | Vista H3 horaria, queries de heatmap, retención |
| [redis-schema.md](./redis-schema.md) | Claves, TTL, usos secundarios, puertos hexagonales |
| [data-lifecycle.md](./data-lifecycle.md) | Retención por almacén, right-to-be-forgotten, residencia regional |
| [slo-observability.md](./slo-observability.md) | SLOs verificables, métricas Prometheus, dashboards Grafana |
| [helm/README.md](./helm/README.md) | Values Helm por almacén, cloud vs. on-prem, operación |
| [terraform/README.md](./terraform/README.md) | Módulos Terraform, orden de provisioning, variables |
