# proposal: almacenamiento-lectura

## Objetivo

El Sistema Anti-Hurto de Vehículos tiene dos cargas de lectura fundamentalmente distintas que no pueden ser servidas por un único almacén de datos: el hot path operacional (búsqueda de placa por agente policial, SLO p95 < 300 ms; cotejo del Matcher contra lista roja hurtada, sub-milisegundo) y el cold path analítico (mapas de calor de zonas de hurto, tendencias por ciudad/país, SLO p95 < 3 s para dashboards de 7 días). Este change define la capa de persistencia y almacenamiento de lectura: los schemas, particionamientos, estrategias de índice, proyecciones y ciclos de vida de datos que sustentan los cuatro almacenes especializados — PostgreSQL+PostGIS, OpenSearch, ClickHouse y Redis — estableciendo el contrato que `backbone-procesamiento` escribe y `api-frontend-analitica` lee.

## Alcance

**Incluye:**
- **PostgreSQL + PostGIS**: esquema de la tabla `vehicle_events` (UUID idempotente, PostGIS point, campos de auditoría), tabla canónica `stolen_vehicles`, tabla `incidents` con ciclo de vida completo, tabla `audit_log` append-only con retención de 7 años; particionamiento por `country_code` (ADR-011); índices por `plate_normalized`, `(country_code, timestamp_capture)` e índice espacial PostGIS; estrategia HA con Patroni on-prem y equivalente gestionado (RDS Multi-AZ / CloudSQL, mismo Helm chart con values distintos); configuración del conector Debezium CDC sobre el WAL de PostgreSQL para alimentar ClickHouse.
- **OpenSearch**: diseño de índices rotatorios `vehicle-events-{country_code}-{YYYY-MM}`; mapping de campos (`plate_normalized` como keyword + analyzed para búsqueda fuzzy, `geo_point` para coordenadas, etc.); configuración de búsqueda fuzzy (fuzziness AUTO, max expansions); política ILM: hot 0–3 meses, warm 3–12 meses, cold 12–24 meses, eliminación > 24 meses; aliases por país para aislamiento multi-tenant.
- **ClickHouse**: tabla materializada columnar `vehicle_events_daily` particionada por `toYYYYMM(timestamp_capture)` y `country_code`; vistas materializadas de hexbinning H3 (resolución 8, aproximadamente 0,74 km²) pre-agregadas por hora para mapas de calor; estrategia de sharding por `country_code` + mes para distribución balanceada; retención de 24 meses para datos raw, vistas agregadas mantenidas indefinidamente; ingestión vía Kafka desde el topic de Debezium CDC o consumidor directo desde el topic de Enrichment.
- **Redis**: estructura de clave `stolen:{country_code}:{plate_normalized}` con TTL 24 h para la lista roja del Matcher; topología Redis Sentinel (99,9 % HA) con opción de migración a Redis Cluster (> 50 GB de dataset); estrategia de invalidación de caché desencadenada por el Canonical Vehicles Service ante cada cambio en la lista robada; usos secundarios: tokens de sesión (TTL configurado por Keycloak), contadores de rate-limiting con ventana deslizante y pub/sub para fan-out de alertas.
- **Ciclo de vida y multi-tenancy (ADR-011)**: documento de política de retención unificada por almacén; estrategia de derecho al olvido (right-to-be-forgotten) cuando un vehículo es retirado de la lista robada; opción de residencia regional por namespace de `country_code` en clústeres K8s regionales.
- Documentación de decisiones de diseño: ADR sobre topología Redis (Sentinel vs. Cluster), ADR sobre estrategia de ingestión en ClickHouse (CDC vía Debezium vs. consumidor Kafka directo), ADR sobre particionamiento de `vehicle_events` en PostgreSQL.
- Documentación operativa: Helm charts para cada almacén (mismos manifiestos para cloud y on-prem con values distintos), runbooks de mantenimiento de índices (vacuum PostgreSQL, rollover OpenSearch), módulos Terraform, monitoreo de retraso de ingestión ClickHouse.

**Excluye:**
- El pipeline de procesamiento que escribe en estos almacenes (`backbone-procesamiento`, aprobado): este change define los schemas que ese pipeline usa pero no reimplementa la lógica de procesamiento.
- La sincronización de vehículos hurtados desde BDs policiales hacia `stolen_vehicles` y Redis (`sincronizacion-paises`, aprobado): este change define la tabla canónica y la estructura de clave Redis, pero no el proceso de sincronización.
- La capa de API y frontend analítico que lee de estos almacenes (`api-frontend-analitica`, pendiente): los contratos de lectura quedan documentados aquí como especificación de interfaz, pero no se implementan endpoints.
- La emisión de JWT y la gestión de tenants en Keycloak (`identidad-seguridad`, pendiente): el claim `country_code` del JWT determina el scope de tenant en las consultas, pero la emisión y configuración no son parte de este change.
- Object storage para imágenes y miniaturas (MinIO/S3): los campos `image_uri` y `thumbnail_uri` se referencian en `vehicle_events`, pero la gestión del object storage es responsabilidad de `ingestion-mqtt`.
- El agente de borde, la ingestión MQTT y el Deduplicator: componentes upstream fuera del alcance de este change.

## Justificación

Sin esquemas y estrategias de almacenamiento definidos, `backbone-procesamiento` no puede finalizar sus proyecciones a PostgreSQL, OpenSearch y ClickHouse, y `api-frontend-analitica` no puede construir sus adaptadores de lectura. Los SLOs de latencia son incompatibles entre sí y con un almacén unificado: sub-milisegundo para el Matcher (Redis), p95 < 300 ms para búsqueda operacional (OpenSearch), y p95 < 3 s para analítica diferida (ClickHouse). La elección de cada motor es una decisión arquitectural del Pilar 6 (ADR-008 CQRS pragmático) que debe estar documentada con su justificación antes de la implementación. Adicionalmente, el requisito de multi-tenancy por `country_code` desde el primer día (ADR-011) y la portabilidad cloud-agnostic (ADR-005) imponen restricciones de diseño transversales a los cuatro almacenes que deben quedar explícitas en un único artefacto de referencia. Este change cierra la brecha entre el backbone de procesamiento aprobado y la capa de lectura pendiente.

## Restricciones

- **SLO hot path — búsqueda operacional**: OpenSearch p95 < 300 ms para búsqueda de placa (UI de agente), p95 < 800 ms para mapa con hasta 10 K eventos simultáneos.
- **SLO hot path — Matcher**: cotejo Redis `GET stolen:{country_code}:{plate_normalized}` en condiciones normales < 1 ms; con fallback local de Kafka Streams (ver `backbone-procesamiento`) < 5 ms.
- **SLO cold path**: agregación ClickHouse con hexbinning H3 para dashboard de 7 días p95 < 3 s.
- **Cloud-agnostic**: PostgreSQL portable (Patroni on-prem / RDS Multi-AZ / CloudSQL), OpenSearch OSS (sin servicios propietarios de AWS), ClickHouse OSS, Redis OSS; ningún servicio vinculado a un único proveedor de nube.
- **Hexagonal (ADR-005)**: adaptadores de conexión a bases de datos gestionadas (connection strings, credenciales de IAM, certificados TLS) detrás de puertos parametrizados; mismos Helm charts para cloud y on-prem mediante values distintos.
- **Multi-tenant desde el primer día (ADR-011)**: `country_code` como discriminador en todas las tablas, índices, prefijos de clave Redis y tópicos CDC; ningún dato de un país puede ser accesible desde el contexto de otro.
- **Idempotencia**: `event_id` UUID como clave primaria idempotente en `vehicle_events`; inserciones repetidas del pipeline (reintentos por at-least-once Kafka) no generan duplicados.
- **Retención operable**: la política de retención debe automatizarse con herramientas estándar: pg_partman para PostgreSQL, políticas ILM para OpenSearch, TTL nativo de Redis, TTL de tabla ClickHouse; sin scripts ad-hoc manuales.
- **Despliegue**: Kubernetes vía Helm; infraestructura aprovisionada con Terraform; mismos manifiestos cloud y on-prem.
