# 3. Componentes Principales

← [Índice](../propuesta-arquitectura-hurto-vehiculos.md)

---

## 3.1 Borde

| Componente | Responsabilidad | Tecnología | Especificación |
|---|---|---|---|
| **Agente Edge** | Captura, normaliza, encola y reenvía eventos; aplica Bloom filter local; orquesta subida de imágenes; expone health y ejecuta OTA | Binario Go estático (~30 MB), systemd service | [overview.md](../agente-borde/overview.md) |
| **ANPR Adapter (SPI)** | Abstracción para leer placas desde el componente ANPR del dispositivo (REST local, socket UNIX, watch file, named pipe) | Interfaz Go con implementaciones intercambiables | [anpr-adapter-spi.md](../agente-borde/anpr-adapter-spi.md) |
| **Queue Manager** | Persiste eventos pendientes; sobrevive a cortes de energía; orden y prioridad (alertas primero) | SQLite con modo WAL + `synchronous=NORMAL` | [queue-manager.md](../agente-borde/queue-manager.md) |
| **Image Store local** | Buffer rotativo de imágenes; política configurable (ej. 7 días o 10 GB) | Filesystem ext4, rotación por tamaño y antigüedad | [image-store.md](../agente-borde/image-store.md) |
| **Bloom Filter** | Filtrado probabilístico local de hot-list por país, FP ~1 %, < 1 MB para 1 M placas | Implementación Go en memoria + persistencia | [bloom-filter.md](../agente-borde/bloom-filter.md) |
| **Health Beacon** | Reporta CPU, RAM, disco, batería, cobertura GSM, drift NTP | Métricas comprimidas vía MQTT (msgpack) | [health-beacon.md](../agente-borde/health-beacon.md) |
| **Config/OTA Manager** | Recibe configuración remota y actualizaciones firmadas | Suscripción a topic MQTT retained; verificación de firma cosign | [config-ota-manager.md](../agente-borde/config-ota-manager.md) |
| **Watchdog** | Reinicia el agente ante hangs o consumo anómalo de recursos | Proceso systemd con `WatchdogSec` | [watchdog.md](../agente-borde/watchdog.md) |

---

## 3.2 Ingestión

| Componente | Responsabilidad | Tecnología | Especificación |
|---|---|---|---|
| **MQTT Broker** | Recibir telemetría de dispositivos con QoS 1, autenticación mTLS, ACL por device | EMQX 5.x cluster (3 nodos, anti-afinidad por AZ) | [emqx-cluster.md](../ingestion-mqtt/emqx-cluster.md) |
| **Upload Service** | Emitir URLs pre-firmadas (TTL 5 min) para subida directa de imágenes; modo proxy para NAT/APN | Servicio Go; arquitectura hexagonal con `ObjectStoragePort` | [upload-service.md](../ingestion-mqtt/upload-service.md) |
| **Object Storage** | Almacenar imágenes (full + thumbnail) con lifecycle policy y SSE-KMS | S3-API: AWS S3, GCS, Azure Blob, o MinIO | [object-storage.md](../ingestion-mqtt/object-storage.md) |
| **MQTT→Kafka Bridge** | Mover mensajes del broker a Kafka manteniendo orden por `device_id` | EMQX Rule Engine + Kafka Data Bridge nativo | [bridge.md](../ingestion-mqtt/bridge.md) |
| **Seguridad de ingestión** | mTLS con OCSP/CRL, ACL por `device_id`/`country_code`, cifrado en tránsito y en reposo | TLS 1.3, Vault PKI, SSE-KMS | [security.md](../ingestion-mqtt/security.md) |

---

## 3.3 Backbone y Procesamiento

| Componente | Responsabilidad | Tecnología | Especificación |
|---|---|---|---|
| **Event Bus** | Backbone durable para eventos. 6 tópicos del pipeline con replication factor 3 y `min.insync.replicas=2`. | Apache Kafka (alt: Redpanda) | [kafka-topics.md](../backbone-procesamiento/kafka-topics.md) |
| **Deduplicator** | Elimina reenvíos idempotentes usando `event_id` único; ventana de 24 h en state store RocksDB | Kafka Streams con state store RocksDB | [deduplicator.md](../backbone-procesamiento/deduplicator.md) |
| **Enrichment Service** | Geocodifica coordenadas (dataset local, timeout 200 ms), normaliza placa por país, evalúa calidad de imagen; proyecta a PostgreSQL y OpenSearch | Servicio Go / NestJS; arquitectura hexagonal | [enrichment-service.md](../backbone-procesamiento/enrichment-service.md) |
| **Matcher Service** | Coteja placa normalizada contra lista roja Redis; fallback fail closed con Kafka Streams local state store si Redis no disponible | Servicio Go consumidor de Kafka + Redis lookup + GlobalKTable | [matcher-service.md](../backbone-procesamiento/matcher-service.md) |
| **Alert Service** | Genera, persiste y rutea alertas a oficiales por zona PostGIS; deduplicación por ventana 5 min | Go / NestJS + PostgreSQL + gRPC al WebSocket Gateway | [alert-service.md](../backbone-procesamiento/alert-service.md) |
| **WebSocket Gateway** | Sesiones WebSocket de agentes policiales (JWT Keycloak); filtro por `country_code` + geocerca | Go / NestJS; Redis session store | [websocket-gateway.md](../backbone-procesamiento/websocket-gateway.md) |
| **incident-service** | Máquina de estados del ciclo de recuperación (CREATED→DISPATCHED→ON_SCENE→RECOVERED→CLOSED); log de auditoría append-only | Go / NestJS + PostgreSQL | [incident-service.md](../backbone-procesamiento/incident-service.md) |
| **Debezium CDC** | Captura cambios del WAL de PostgreSQL y los publica a `pg.events.cdc`; ClickHouse ingiere vía Kafka Engine Table | Debezium 2.x + Kafka Connect cluster | [debezium-cdc.md](../backbone-procesamiento/debezium-cdc.md) |

> **Vista completa del backbone:** [`docs/backbone-procesamiento/overview.md`](../backbone-procesamiento/overview.md)

---

## 3.4 Adaptación por País (Anti-Corruption Layer)

| Componente | Responsabilidad | Tecnología | Especificación |
|---|---|---|---|
| **Country Adapter** | Conector específico por país; mapea su BD policial al modelo canónico; publica eventos | Contenedor independiente por país; modos: CDC (Debezium), polling JDBC, SOAP/REST cliente, file watcher | [country-adapter-framework.md](../sincronizacion-paises/country-adapter-framework.md) |
| **Schema Registry** | Versiona el modelo canónico de eventos y vehículos hurtados | Confluent Schema Registry o Apicurio | [avro-schema.md](../sincronizacion-paises/avro-schema.md) |
| **Canonical Vehicles Service** | Procesa eventos canónicos; hace upsert en PostgreSQL; actualiza lista roja en Redis | NestJS / Go; arquitectura hexagonal | [canonical-vehicles-service.md](../sincronizacion-paises/canonical-vehicles-service.md) |
| **Edge Distribution Service** | Genera y distribuye Bloom filters por país a los agentes de borde | Go; MQTT retained | [edge-distribution-service.md](../sincronizacion-paises/edge-distribution-service.md) |

> **Vista completa:** [`docs/sincronizacion-paises/overview.md`](../sincronizacion-paises/overview.md)

---

## 3.5 Almacenamiento de Lectura

| Componente | Responsabilidad | Tecnología | Especificación |
|---|---|---|---|
| **PostgreSQL + PostGIS** | Eventos consultables, vehículos hurtados, usuarios, configuración; consultas geoespaciales | PostgreSQL 16 (RDS / CloudSQL / Aurora / Patroni HA on-prem) | [schema](../almacenamiento-lectura/postgresql-schema.md) · [HA](../almacenamiento-lectura/postgresql-ha.md) |
| **OpenSearch** | Búsqueda rápida por matrícula, búsqueda fuzzy, agregaciones temporales | OpenSearch 2.x (alt: Elasticsearch OSS) | [schema](../almacenamiento-lectura/opensearch-schema.md) · [ILM](../almacenamiento-lectura/opensearch-ilm.md) |
| **ClickHouse** | Analítica columnar masiva: heatmaps, rutas, tendencias por ciudad/país | ClickHouse cluster con sharding por país | [schema](../almacenamiento-lectura/clickhouse-schema.md) · [H3 views](../almacenamiento-lectura/clickhouse-h3-views.md) |
| **Redis** | Cache de hot-list de placas, sesiones, rate-limiting, pub/sub de alertas | Redis Sentinel o Cluster | [schema](../almacenamiento-lectura/redis-schema.md) · [ADR topología](../almacenamiento-lectura/adr-redis-topology.md) |

> **Vista completa:** [`docs/almacenamiento-lectura/overview.md`](../almacenamiento-lectura/overview.md)

---

## 3.6 API, Frontend y Analítica

| Componente | Responsabilidad | Tecnología | Especificación |
|---|---|---|---|
| **API Gateway** | Punto único de entrada; rate-limiting, autenticación JWT/JWKS, routing, TLS, audit log | Kong OSS (DB-less mode) | [api-gateway.md](../api-frontend-analitica/api-gateway.md) |
| **search-service** | Búsqueda de eventos por matrícula (exacta + fuzzy); orquesta OpenSearch + PostgreSQL + presigned URLs | NestJS o Go; puertos hexagonales | [search-service.md](../api-frontend-analitica/search-service.md) |
| **alerts-service** | Distribución de alertas en tiempo real a oficiales de la zona via WebSocket | Socket.IO + Redis adapter | [alerts-service.md](../api-frontend-analitica/alerts-service.md) |
| **incident-service** | Ciclo de vida de recuperación (created → dispatched → escalated → recovered → closed) | NestJS o Go | [incident-service.md](../api-frontend-analitica/incident-service.md) |
| **analytics-service** | Heatmaps H3, tendencias de hurto, rutas de escape | NestJS o Go; ClickHouse | [analytics-service.md](../api-frontend-analitica/analytics-service.md) |
| **devices-service** | CRUD de dispositivos; OTA asíncrono | NestJS o Go; multi-tenant por `country_code` | [devices-service.md](../api-frontend-analitica/devices-service.md) |
| **vehicles-service** | CRUD manual de vehículos hurtados (role=admin); coordina cache Redis | NestJS o Go | [vehicles-service.md](../api-frontend-analitica/vehicles-service.md) |
| **Web App** | Mapa de trayectorias, búsqueda por placa, alertas en vivo, gestión de recuperación, analítica, admin | React + TypeScript + MapLibre GL JS + Kepler.gl + Socket.IO | [webapp-architecture.md](../api-frontend-analitica/webapp-architecture.md) |
| **BI / Analytics** | Dashboards interactivos con Row Level Security por país | Apache Superset OSS + ClickHouse | [superset-integration.md](../api-frontend-analitica/superset-integration.md) |

> **Vista completa:** [`docs/api-frontend-analitica/overview.md`](../api-frontend-analitica/overview.md)

---

## 3.7 Identidad

| Componente | Responsabilidad | Tecnología | Especificación |
|---|---|---|---|
| **Keycloak** | Broker de identidad; un realm por país; emisión de JWT; SSO web | Keycloak 24+ sobre K8s con Postgres dedicado | [keycloak-realm-model.md](../identidad-seguridad/keycloak-realm-model.md) |
| **Federation Providers** | LDAP nativo, JDBC nativo, SPI custom para SOAP/REST, delegación SAML/OIDC | Keycloak SPIs (User Storage SPI / Identity Provider SPI) | [federation-ldap.md](../identidad-seguridad/federation-ldap.md) · [federation-spi-custom.md](../identidad-seguridad/federation-spi-custom.md) |
| **Device PKI** | Emisión, rotación y revocación de certificados de dispositivo | HashiCorp Vault PKI (alt: step-ca) | [vault-pki-device.md](../identidad-seguridad/vault-pki-device.md) |

> **Vista completa:** [`docs/identidad-seguridad/overview.md`](../identidad-seguridad/overview.md)

---

## 3.8 Observabilidad y Operación

| Componente | Responsabilidad | Tecnología |
|---|---|---|
| **Métricas** | Recolección y alerting de métricas de plataforma y negocio | Prometheus + Alertmanager + Grafana |
| **Logs** | Logs estructurados centralizados | Loki o OpenSearch (logs) |
| **Trazas** | Tracing distribuido extremo a extremo | OpenTelemetry SDK + Tempo o Jaeger |
| **Secrets** | Custodia de secretos y rotación | Vault (cloud-agnostic) |
| **GitOps** | Despliegue declarativo y reproducible | ArgoCD + Helm/Kustomize |
| **IaC** | Provisión de infraestructura | Terraform con módulos por proveedor |
