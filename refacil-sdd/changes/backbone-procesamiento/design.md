# design: backbone-procesamiento

## Archivos a crear

| Ruta | Propósito |
|------|-----------|
| `docs/backbone-procesamiento/overview.md` | Descripción narrativa del backbone de procesamiento: flujo completo `vehicle.events.raw` → Deduplicator → Enrichment → Matcher → Alert → WebSocket; cold path vía Debezium → ClickHouse; diagrama Mermaid de componentes; SLO hot path p95 < 2 s; decisiones ADR aplicadas |
| `docs/backbone-procesamiento/deduplicator.md` | Especificación del Deduplicator: esquema del estado RocksDB, clave de deduplicación (`event_id`), ventana de 24 h, política de retención del estado, garantías de ordering, configuración de Kafka Streams, métricas expuestas (`deduplicator.duplicates.discarded`, lag de procesamiento), dead-letter topic (`vehicle.events.raw.dlq`) |
| `docs/backbone-procesamiento/enrichment-service.md` | Especificación del Enrichment Service: flujo de enriquecimiento (geocodificación inversa, normalización de placa, evaluación de calidad de imagen); decisión síncrono vs. asíncrono (ADR del change); puertos hexagonales: `GeocodingPort`, `ImageQualityPort`, `EventsRepositoryPort` (PostgreSQL), `EventsIndexPort` (OpenSearch); estrategia de índices OpenSearch por país + mes; schema PostGIS en PostgreSQL; manejo de fallos de geocodificación (CR-02) |
| `docs/backbone-procesamiento/adr-geocoding-strategy.md` | ADR específico del change: decisión entre geocodificación síncrona y asíncrona en el hot path. Criterios: impacto sobre SLO p95 < 2 s, consistencia del evento enriquecido, complejidad operacional. Decisión: geocodificación síncrona con timeout estricto (p.ej. 200 ms) usando dataset local (Nominatim self-hosted o GeoNames) como primera opción, con fallback a campos en blanco y `geocoding_status = "FAILED"`. Justificación: mantiene el evento cohesivo, evita un segundo topic y un join posterior, y el timeout estricto preserva el SLO |
| `docs/backbone-procesamiento/matcher-service.md` | Especificación del Matcher Service: protocolo Redis (`GET stolen:{country_code}:{plate_normalized}`), estructura de la clave, TTL de entradas en la lista roja; estrategia de fallback ante indisponibilidad de Redis (estado local Kafka Streams); política fail closed; puerto hexagonal `StolenVehiclesPort`; latencia objetivo (< 5 ms en condiciones normales); métricas: `matcher.hits`, `matcher.misses`, `matcher.redis_fallback.activations` |
| `docs/backbone-procesamiento/adr-matcher-fallback.md` | ADR específico del change: estrategia fail closed vs. fail open en el Matcher ante indisponibilidad de Redis. Decisión: fail closed con Kafka Streams local state store como caché secundaria. Justificación: en un sistema de seguridad pública, omitir un cotejo (fail open) introduce riesgo de falsos negativos inaceptables; el estado local de Kafka Streams puede mantenerse sincronizado con el tópico `stolen.vehicles.events` como fuente de verdad |
| `docs/backbone-procesamiento/alert-service.md` | Especificación del Alert Service: esquema de la tabla de alertas en PostgreSQL (`alert_id`, `event_id`, `plate_normalized`, `country_code`, `location`, `alert_timestamp`, `state`, `zone_id`); algoritmo de deduplicación de alertas (ventana de 5 minutos por placa + zona); enrutamiento geoespacial a agentes activos (consulta PostGIS contra tabla de sesiones activas); integración con WebSocket Gateway (push interno); dead-letter y manejo de alertas no entregadas; métricas: `alert.deduplication.suppressed`, `alert.delivery.no_active_agents` |
| `docs/backbone-procesamiento/websocket-gateway.md` | Especificación del WebSocket Gateway: protocolo (Socket.IO sobre HTTPS o WS nativo); autenticación de conexión (validación JWT emitido por Keycloak, referencia a `identidad-seguridad`); registro de sesiones activas (Redis o tabla PostgreSQL de sesiones); filtrado de alertas por `country_code` + geocerca de zona del agente; entrega de alertas pendientes en reconexión (CR-06); esquema del payload de alerta enviado al cliente; métricas: conexiones activas, alertas entregadas/s, latencia de push |
| `docs/backbone-procesamiento/incident-service.md` | Especificación del incident-service: máquina de estados del incidente (DETECTED → DISPATCHED → ON_SCENE → RECOVERED / CANCELLED); esquema de la tabla de incidentes y tabla de auditoría (append-only); reglas de transición válidas e inválidas (CR-07); integración con Alert Service (recibe referencia a alerta); producción de evento `RECOVERED` al tópico `stolen.vehicles.events` para `sincronizacion-paises`; puerto hexagonal `IncidentRepositoryPort` (PostgreSQL); API interna (REST o gRPC) del servicio |
| `docs/backbone-procesamiento/debezium-cdc.md` | Especificación del conector Debezium: tablas monitoreadas en PostgreSQL (tabla de eventos enriquecidos), configuración del conector Kafka Connect, tópico de salida `pg.events.cdc`, clave de mensaje (`event_id`), formato del payload CDC (before/after), configuración de ClickHouse Kafka Engine Table para consumo; lag esperado y monitoreo (CR-08); política de reintentos y dead-letter del conector |
| `docs/backbone-procesamiento/kafka-topics.md` | Especificación autoritativa de tópicos Kafka del backbone: tabla de tópicos (`vehicle.events.raw`, `vehicle.events.deduped`, `vehicle.events.enriched`, `vehicle.alerts`, `pg.events.cdc`, `stolen.vehicles.events`), producer/consumer de cada uno, clave de partición, replication factor, `min.insync.replicas`, retención, dead-letter topics, schema (Avro o JSON Schema con referencia a Schema Registry) |
| `docs/backbone-procesamiento/postgresql-schema.md` | Especificación del esquema de tablas PostgreSQL del backbone: tabla `events` (con columna PostGIS `location`), tabla `alerts`, tabla `incidents`, tabla `incident_audit` (append-only); índices por `country_code`, `plate_normalized`, `timestamp`; particionamiento por `country_code`; notas de coordinación con `almacenamiento-lectura` |
| `docs/backbone-procesamiento/opensearch-schema.md` | Especificación del esquema de índices OpenSearch: nombre del índice (`events-{country_code}-{YYYY-MM}`), mapping de campos, política de ILM (rollover), estrategia de búsqueda exacta y fuzzy por `plate_normalized`; notas de coordinación con `almacenamiento-lectura` |
| `docs/backbone-procesamiento/slo-observability.md` | Especificación de SLOs y observabilidad del backbone: presupuesto de error por componente para cumplir p95 < 2 s end-to-end; métricas Prometheus por componente; dashboards de Grafana recomendados; alertas operacionales (lag Kafka, lag Debezium, tasa de fallos de geocodificación, fallos Redis, alertas suprimidas, incidentes sin despacho) |
| `docs/backbone-procesamiento/helm/README.md` | Guía de despliegue con Helm: values configurables para cada microservicio (Deduplicator, Enrichment, Matcher, Alert, WebSocket Gateway, incident-service, Debezium Connector); instrucciones de instalación, actualización y rollback en Kubernetes |
| `docs/backbone-procesamiento/terraform/README.md` | Guía de módulos Terraform: infraestructura necesaria para el backbone (topics Kafka, credenciales Redis, extensión PostGIS en PostgreSQL, índices OpenSearch, Debezium Kafka Connect cluster); variables de entrada/salida por módulo |

## Archivos a modificar

| Ruta | Cambios |
|------|---------|
| `docs/propuesta-arquitectura-hurto-vehiculos.md` | Agregar referencias cruzadas en la sección del Pilar 6 (Hot path / Cold path) hacia `docs/backbone-procesamiento/`; actualizar la tabla de componentes de procesamiento (Deduplicator, Enrichment, Matcher, Alert Service, WebSocket Gateway, incident-service) con enlaces a las especificaciones detalladas; no alterar diagramas Mermaid existentes ni atributos de calidad |

## Archivos fuera de alcance (doNotTouch)

- `refacil-sdd/` — artefactos SDD gestionados por la metodología
- `.claude/`, `.cursor/` — configuración de agentes
- `AGENTS.md` — índice del proyecto; solo se modifica en el change `setup` o por instrucción explícita del autor
- `.agents/` — documentación de sesión; no se modifica en este change
- `refacil-sdd/changes/agente-borde/` — change aprobado; no se modifica
- `refacil-sdd/changes/ingestion-mqtt/` — change aprobado; no se modifica
- `docs/agente-borde/` — especificaciones del change `agente-borde`; no se modifican
- `docs/ingestion-mqtt/` — especificaciones del change `ingestion-mqtt`; no se modifican
- `package-lock.json` — no aplica (repo de documentación)

## Patrones y convenciones detectados

- **Idioma:** español formal orientado a audiencia técnica senior (regla `Always` en `.agents/summary.md`); identificadores, rutas y nombres de archivo en inglés.
- **Diagramas:** Mermaid para todo diagrama nuevo; usar secuencia para flujos request/response y flowchart/graph para topología de componentes y máquinas de estado, igual que en `docs/agente-borde/` y `docs/ingestion-mqtt/`.
- **Terminología canónica:** "agente de borde", "hot path / cold path", "lista roja", "geocodificación inversa", "normalización de placa", "log de auditoría inmutable", "discriminador de tenant"; mantener consistencia con la propuesta principal, `docs/agente-borde/` y `docs/ingestion-mqtt/`.
- **Referencias cruzadas ADR:** toda decisión técnica debe mencionar explícitamente el ADR que la origina (ADR-005, ADR-008, ADR-011, ADR-012); los ADRs nuevos del change (`adr-geocoding-strategy.md`, `adr-matcher-fallback.md`) deben indicar que complementan ADR-008.
- **Estructura de sección:** encabezados H2/H3 sin numeración propia en documentos bajo `docs/backbone-procesamiento/`; la numeración existe solo en la propuesta principal.
- **Hexagonal:** cada microservicio con dependencias externas (Redis, geocodificación, PostgreSQL, OpenSearch) debe definir sus puertos como interfaces y sus adaptadores como implementaciones intercambiables, igual patrón que el `ObjectStoragePort` del Upload Service en `ingestion-mqtt`.
- **Privacidad:** los documentos de esta capa deben explicitar que se procesan placas, coordenadas y referencias a imágenes de vehículos (no rostros ni biometría); `country_code` como barrera de aislamiento de datos entre países (ADR-011).
- **Multi-tenant:** `country_code` como partition key en Kafka, como campo de filtrado en Redis, como discriminador en PostgreSQL e índice en OpenSearch.

## Decisiones de diseño pendientes de documentación en ADR

Dos decisiones deben formalizarse como ADR durante la implementación de los artefactos:

1. **Geocodificación síncrona vs. asíncrona** (`adr-geocoding-strategy.md`): la opción asíncrona (escribir primero, enriquecer después) simplifica el hot path pero requiere un segundo topic y un join posterior; la opción síncrona con timeout estricto mantiene el evento cohesivo y es la opción recomendada con dataset de geocodificación local para evitar dependencia de API externa en el hot path.

2. **Fallback del Matcher** (`adr-matcher-fallback.md`): fail closed con Kafka Streams local state store alimentado por el tópico `stolen.vehicles.events` es la única opción aceptable en un sistema de seguridad pública; fail open introduce riesgo de falsos negativos inaceptable.

## Dependencias entre tareas

1. `docs/backbone-procesamiento/overview.md` y `docs/backbone-procesamiento/kafka-topics.md` deben redactarse primero — el overview establece el glosario y el flujo; el esquema de tópicos es el contrato compartido que todos los demás documentos referencian.
2. Los ADRs (`adr-geocoding-strategy.md`, `adr-matcher-fallback.md`) deben completarse antes de las especificaciones de los servicios que dependen de esas decisiones (`enrichment-service.md`, `matcher-service.md`).
3. `docs/backbone-procesamiento/postgresql-schema.md` y `docs/backbone-procesamiento/opensearch-schema.md` deben completarse antes de `docs/backbone-procesamiento/enrichment-service.md` y `docs/backbone-procesamiento/alert-service.md` (las especificaciones de servicio referencian el esquema de datos).
4. `docs/backbone-procesamiento/alert-service.md` debe completarse antes de `docs/backbone-procesamiento/websocket-gateway.md` y `docs/backbone-procesamiento/incident-service.md` (ambos dependen del contrato de alerta).
5. `docs/backbone-procesamiento/debezium-cdc.md` puede redactarse en paralelo con los servicios del hot path, ya que su única dependencia es `postgresql-schema.md`.
6. `docs/backbone-procesamiento/slo-observability.md` se redacta una vez que todas las especificaciones de servicio están consolidadas (referencia las métricas de cada componente).
7. `docs/backbone-procesamiento/helm/README.md` y `docs/backbone-procesamiento/terraform/README.md` son las últimas tareas de documentación.
8. La modificación de `docs/propuesta-arquitectura-hurto-vehiculos.md` es el paso final.

## Nota de validación cruzada entre repositorios

Este change establece o consume contratos con otros changes del sistema. Consultar `refacil-prereqs/BUS-CROSS-REPO.md` para el proceso de acuerdo formal:

- **`ingestion-mqtt` (upstream, aprobado):** el esquema del mensaje en `vehicle.events.raw`, la clave de partición `device_id`, los campos `event_id`, `plate`, `lat`, `lon`, `country_code`, `device_id` y `timestamp` son el contrato de entrada del Deduplicator. Cualquier cambio en ese esquema requiere notificación a este change.
- **`almacenamiento-lectura` (downstream, pendiente):** los esquemas de las tablas PostgreSQL (incluida la columna PostGIS), los índices OpenSearch y la Kafka Engine Table de ClickHouse deben acordarse con ese change antes de la implementación. Este change propone los esquemas; `almacenamiento-lectura` los valida y puede introducir restricciones adicionales.
- **`api-frontend-analitica` (downstream, pendiente):** el esquema del payload de alerta enviado por el WebSocket Gateway al cliente móvil policial y el contrato de la API de consulta de incidentes deben acordarse con ese change.
- **`identidad-seguridad` (downstream, pendiente):** el WebSocket Gateway valida JWT emitidos por Keycloak; el incident-service puede requerir autorización por rol. El mecanismo de validación (endpoint JWKS, claims requeridos) debe acordarse con ese change.
- **`sincronizacion-paises` (pendiente):** el incident-service produce eventos al tópico `stolen.vehicles.events` cuando un vehículo pasa a estado `RECOVERED`; el esquema de ese mensaje debe acordarse con el change que implementa el Canonical Vehicles Service.
