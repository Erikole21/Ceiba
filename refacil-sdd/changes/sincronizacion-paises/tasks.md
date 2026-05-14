# tasks: sincronizacion-paises

## Fase 1 — Fundamentos (modelo canónico y contrato Kafka)

- [x] T-01: Redactar `docs/sincronizacion-paises/overview.md` — descripción narrativa del Pilar 4, flujo completo desde la BD policial hasta el agente de borde, diagrama Mermaid de componentes y topología de despliegue on-prem vs. cloud [M]
- [x] T-02: Redactar `docs/sincronizacion-paises/canonical-model.md` — especificación autoritativa de los 9 campos mandatorios del caso de estudio, campos adicionales (`country_code`, `status`, `schema_version`, `extensions`), reglas de validación por campo y política de uso de `extensions` [M]
- [x] T-03: Redactar `docs/sincronizacion-paises/avro-schema.md` — schema JSON completo del evento canónico, reglas de compatibilidad backward en el Schema Registry, política de versionado minor/major y proceso de migración de adaptadores ante versión major [M]
- [x] T-04: Redactar `docs/sincronizacion-paises/kafka-topics.md` — tabla autoritativa de tópicos `stolen.vehicles.events` y `stolen.vehicles.canonical` con producers, consumers, clave de partición, replication factor, retención y dead-letter topics; integración con la tabla de tópicos de `backbone-procesamiento` [S]

## Fase 2 — Decisiones de diseño (ADRs)

- [x] T-05: Redactar `docs/sincronizacion-paises/adr-006-acl-canonical-model.md` — ADR-006 formal: opciones evaluadas, decisión de adaptador por país con modelo canónico, consecuencias y referencia a la propuesta principal [M]
- [x] T-06: Redactar `docs/sincronizacion-paises/adr-009-bloom-filter-edge.md` — ADR-009 formal: opciones evaluadas, parámetros fijos del BF (FP ≈ 1 %, < 1 MB para 1 M placas), protocolo de delta y política de full snapshot; referencia al contrato con `agente-borde` [M]

## Fase 3 — Marco de adaptadores

- [x] T-07: Redactar `docs/sincronizacion-paises/country-adapter-framework.md` — interfaz de contrato del adaptador, los cuatro modos de integración (CDC Debezium, JDBC polling, SOAP/REST programado, file drop watcher), lógica de checkpoint, política de idempotencia y métricas por adaptador [L]
- [x] T-08: Redactar `docs/sincronizacion-paises/sla-freshness.md` — SLAs de frescura por modo de integración, tabla con ventanas de retraso, impacto en la lista roja de Redis y proceso de comunicación del SLA al contraparte policial [S]
- [x] T-09: Redactar `docs/sincronizacion-paises/data-sovereignty.md` — política de soberanía de datos, diagrama de topología on-prem vs. cloud, reglas de qué datos pueden salir del territorio, configuración del campo `extensions` para datos autorizados, referencias legales por país y conexión con `identidad-seguridad` [M]

## Fase 4 — Especificaciones de servicios

- [x] T-10: Redactar `docs/sincronizacion-paises/postgresql-schema.md` — DDL de la tabla `canonical_vehicles` (columnas, tipos, restricciones, índices, particionamiento por `country_code`, columna JSONB `extensions`, trigger `updated_at`) con nota de coordinación con `almacenamiento-lectura` [M]
- [x] T-11: Redactar `docs/sincronizacion-paises/canonical-vehicles-service.md` — especificación del Canonical Vehicles Service: puertos hexagonales, flujo de procesamiento (validar → upsert → actualizar Redis → publicar en `stolen.vehicles.canonical`), proceso de reconciliación Redis, manejo del evento RECOVERED y métricas [L]
- [x] T-12: Redactar `docs/sincronizacion-paises/edge-distribution-service.md` — especificación del Edge Distribution Service: puerto `BloomFilterStorePort`, algoritmo de delta BF, política de full snapshot (> 7 días), esquema del payload MQTT, referencias al contrato con `agente-borde` e `ingestion-mqtt`, y métricas [L]

## Fase 5 — Guías operativas y evolución

- [x] T-13: Redactar `docs/sincronizacion-paises/schema-evolution.md` — guía de evolución del esquema canónico: campos opcionales (minor version), cambios en campos mandatorios (major version + plan de migración coordinada), uso de `extensions` para postergar promovimiento a mandatorio, proceso de deprecación [M]
- [x] T-14: Redactar `docs/sincronizacion-paises/country-onboarding-guide.md` — guía paso a paso de incorporación de un nuevo país: selección de modo, implementación del conector, mapeo de campos, validación con el contraparte policial, despliegue del contenedor; checklist de validación pre-producción [M]
- [x] T-15: Redactar `docs/sincronizacion-paises/slo-observability.md` — SLOs y observabilidad: métricas Prometheus por componente, dashboards de Grafana recomendados, alertas operacionales (lag Kafka, lag adaptador, fallos Redis, BF no publicados, dispositivos solicitando full snapshot) [M]

## Fase 6 — Infraestructura y cierre

- [x] T-16: Redactar `docs/sincronizacion-paises/helm/README.md` — guía de despliegue Helm: values configurables para el Canonical Vehicles Service, Edge Distribution Service y adaptador de referencia; instrucciones de instalación, actualización y rollback; notas sobre despliegue on-prem del adaptador de país [M]
- [x] T-17: Redactar `docs/sincronizacion-paises/terraform/README.md` — guía de módulos Terraform: infraestructura necesaria (tópicos Kafka, Schema Registry, tabla `canonical_vehicles`, claves Redis, tópicos MQTT, almacenamiento para versiones de BF); variables de entrada/salida por módulo [M]
- [x] T-18: Actualizar `docs/propuesta-arquitectura-hurto-vehiculos.md` — agregar referencias cruzadas en la sección del Pilar 4 hacia `docs/sincronizacion-paises/`; actualizar tabla de componentes de sincronización con enlaces a las especificaciones; no alterar diagramas Mermaid existentes ni atributos de calidad [S]
