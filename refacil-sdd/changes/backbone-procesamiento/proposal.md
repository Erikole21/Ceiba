# proposal: backbone-procesamiento

## Objetivo

Los eventos crudos que llegan a Kafka desde los dispositivos de borde contienen duplicados (reintentos QoS 1), coordenadas sin geocodificar, placas sin normalizar y no están contrastados contra el listado de vehículos hurtados. Las alertas deben llegar a los agentes de policía en p95 < 2 s desde la captura, y las proyecciones analíticas deben alimentarse sin acoplar la ingestión operacional al cold path. Este change implementa el Pilar 6 (Hot path / Cold path separados) del Sistema Anti-Hurto de Vehículos: el backbone de procesamiento cloud que transforma los eventos crudos en datos deduplicados, enriquecidos, cotejados y alertados, proyectándolos simultáneamente a los stores de lectura especializados.

## Alcance

**Incluye:**
- **Deduplicator**: consumidor de `vehicle.events.raw` con ventana de deduplicación de 24 h usando Kafka Streams + estado RocksDB; produce a `vehicle.events.deduped`.
- **Enrichment Service**: enriquecimiento de eventos con geocodificación inversa (lat/lon → calle/ciudad/país), normalización de placa (mayúsculas, separadores eliminados, formato por país) y evaluación de calidad de imagen (puntaje de confianza, flag de desenfoque); proyección a PostgreSQL (tabla de eventos con columna geometry PostGIS) y OpenSearch (índice por país + mes); produce a `vehicle.events.enriched`.
- **Matcher Service**: cotejo contra lista roja en Redis (`GET stolen:{country_code}:{plate}`); en hit produce evento de alerta a `vehicle.alerts`; con fallback a estado local de Kafka Streams si Redis no está disponible.
- **Alert Service**: generación y persistencia de alertas en PostgreSQL; enrutamiento geoespacial a agentes activos en la zona; push a WebSocket Gateway; deduplicación de alertas por placa y zona en ventana de 5 minutos.
- **WebSocket Gateway**: conexiones persistentes con la aplicación móvil policial; filtrado de alertas por `country_code` + geocerca de la zona del agente.
- **incident-service**: ciclo de vida de la recuperación del vehículo hurtado: despacho de unidad, escalación, marcado como recuperado, propagación del estado a `sincronizacion-paises`; log de auditoría inmutable en PostgreSQL.
- **Proyecciones PostgreSQL**: escritura de eventos enriquecidos (Enrichment Service), alertas (Alert Service) e incidentes (incident-service); `country_code` como discriminador de tenant en todas las tablas.
- **Proyección OpenSearch**: indexación de eventos para búsqueda de placas (exacta + fuzzy); estrategia de índices rotatorios por país + mes.
- **Cold path vía Debezium CDC**: conector Debezium que lee el WAL de PostgreSQL y publica cambios en `pg.events.cdc`; ClickHouse consume vía Kafka Engine Table, desacoplando la analítica del hot path operacional.
- Documentación de decisiones de diseño: ADR sobre geocodificación síncrona vs. asíncrona, ADR sobre estrategia de fallback del Matcher, especificación de esquema de tópicos Kafka y SLOs del pipeline.
- Documentación operativa: Helm charts para Kubernetes, módulos Terraform, runbooks de SLO y monitoreo de lag de Debezium.

**Excluye:**
- La capa de ingestión MQTT→Kafka (`ingestion-mqtt`, aprobado): este change la consume pero no la modifica.
- El change `sincronizacion-paises` (pendiente): este change notifica sobre vehículos recuperados pero no lo implementa.
- Los schemas de stores de lectura (`almacenamiento-lectura`, pendiente): los esquemas de PostgreSQL, OpenSearch, ClickHouse y Redis deben acordarse con ese change antes de la implementación.
- La API REST/GraphQL para el frontend analítico (`api-frontend-analitica`, pendiente).
- La identidad federada y emisión de JWT de Keycloak (`identidad-seguridad`, pendiente): el WebSocket Gateway y el incident-service consumen JWT pero no implementan la emisión.
- El Bloom Filter Generator (distribución de filtro a dispositivos de borde).
- La implementación de adaptadores por país para BDs policiales.

## Justificación

Sin este backbone de procesamiento, los eventos crudos que llegan a `vehicle.events.raw` quedan sin procesar: no hay deduplicación, no hay cotejo contra el listado de hurtados y no hay alertas en tiempo real a los agentes de policía. El change `ingestion-mqtt` ya está aprobado y los datos fluyen hasta Kafka; este change es el paso desbloqueante que activa el valor operacional del sistema. La separación entre hot path (Deduplicator → Enrichment → Matcher → Alert → WebSocket, p95 < 2 s) y cold path (Debezium CDC → ClickHouse) es la decisión arquitectural central del Pilar 6 (ADR-008): permite escalar y operar ambos paths independientemente sin que la carga analítica impacte la latencia de alertas operacionales.

## Restricciones

- **SLO hot path**: p95 < 2 s desde la captura del evento en el dispositivo hasta la entrega de la alerta al agente; cada componente del pipeline debe exponer métricas de latencia individuales.
- **Ventana de deduplicación**: el Deduplicator debe detectar duplicados dentro de al menos 24 h por `event_id`.
- **Deduplicación de alertas**: misma placa, misma zona → máximo 1 alerta por ventana de 5 minutos por agente.
- **Geocodificación**: la decisión entre enriquecimiento síncrono y asíncrono debe documentarse y justificarse en un ADR del change, dado el impacto directo sobre el SLO de latencia.
- **Fallback del Matcher**: ante indisponibilidad de Redis, usar el estado local de Kafka Streams como caché secundaria (fail closed); nunca fail open en un sistema de seguridad pública.
- **Aislamiento multi-tenant**: `country_code` como discriminador en tablas, índices, tópicos y geocercas; ningún evento de un país debe ser visible desde el contexto de otro (ADR-011).
- **Replicación Kafka**: factor 3, `min.insync.replicas=2` para todos los tópicos del pipeline (ADR-012).
- **Arquitectura hexagonal**: acceso a Redis, servicio de geocodificación y escrituras a PostgreSQL/OpenSearch detrás de puertos/adaptadores (ADR-005).
- **Despliegue**: Kubernetes vía Helm; infraestructura aprovisionada con Terraform.
- **Lag de ClickHouse**: aceptable para el cold path; debe documentarse el lag esperado (segundos a decenas de segundos) y el procedimiento de monitoreo y alerta.
