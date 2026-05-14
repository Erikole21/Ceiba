# Architecture

Plataforma distribuida edge-to-cloud de seis pilares:

1. **Edge resiliente** — agente Go binario (~30 MB) con cola SQLite WAL, Bloom filter de placas hurtadas y reintento idempotente. Soporta hasta 4 h sin red / 1 h sin energía.
2. **Comunicación segura** — MQTT 5 sobre TLS con mTLS por certificado de dispositivo (PKI propia). Imágenes vía URLs pre-firmadas a object storage.
3. **Cloud-agnostic** — Kubernetes, PostgreSQL, Kafka, Keycloak, Redis, OpenSearch, MinIO/S3. Hexagonal ports/adapters para servicios cloud-sensibles.
4. **Anti-Corruption Layer por país** — cada BD policial se sincroniza a modelo canónico via adaptador propio; cambios llegan a Kafka.
5. **Identidad federada** — Keycloak, realm por país; LDAP/DB nativo + SPI custom para SOAP/REST.
6. **Hot path / Cold path** — matching y alertas (p95 < 2 s); analítica diferida (Superset + ClickHouse + PostGIS).

### Componentes principales
- **Dispositivo borde**: cámara + ANPR local + GPS + GSM; agente Go gestiona cola, Bloom filter, MQTT uploader.
- **Ingestión**: MQTT Broker cluster (EMQX) + Upload Service (presigned URLs) → Kafka/Redpanda.
- **Procesamiento**: Deduplicator → Enrichment (Geo/Calidad) → Matcher → Alert Service.
- **Datos**: PostgreSQL+PostGIS (eventos), OpenSearch (búsqueda), ClickHouse (analítica), Redis (caché/sesiones).
- **API y Frontend**: API Gateway (REST + GraphQL), Web App (React + MapLibre), WebSocket (alertas en vivo).
- **Sincronización**: Adapters por país (ACL) con polling/CDC hacia BDs policiales.

Diagramas C4 L1, C4 L2 y diagrama de agente en `docs/propuesta-arquitectura-hurto-vehiculos.md`.
