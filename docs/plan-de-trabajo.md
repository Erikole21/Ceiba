# Plan de Trabajo — Changes SDD

Registro de los changes identificados para la propuesta de arquitectura del Sistema Anti-Hurto de Vehículos.
Se trabajan **pilar a pilar**: primero se entiende el pilar, se identifican los changes que lo componen, y se crea cada uno con `/refacil:propose`.

---

## Estado de changes

| # | Change | Pilar | Alcance resumido | Propose | Apply |
|---|--------|-------|-----------------|---------|-------|
| 1 | `agente-borde` | 1 · Edge resiliente | Agente Go en el dispositivo: ANPR SPI, SQLite WAL, Bloom filter, MQTT uploader, OTA, watchdog | ✅ Aprobado | ✅ Implementado |
| 2 | `ingestion-mqtt` | 2 · Comunicación segura | MQTT broker EMQX, Upload Service (URLs pre-firmadas), MinIO/S3, bridge Kafka | ✅ Aprobado | ✅ Implementado |
| 3 | `backbone-procesamiento` | 6 · Hot/Cold path | Kafka, Deduplicator, Enrichment, Matcher, Alert Service, WebSocket, incident-service | ✅ Aprobado | ✅ Implementado |
| 4 | `sincronizacion-paises` | 4 · ACL por país | Adapters por país, Canonical Service, modelo canónico versionado, Edge Distribution (delta BF) | ✅ Aprobado | ⏳ Pendiente |
| 5 | `almacenamiento-lectura` | 6 · Hot/Cold path | PostgreSQL+PostGIS, OpenSearch, ClickHouse, Redis | ✅ Aprobado | ✅ Implementado |
| 6 | `api-frontend-analitica` | 6 · Hot/Cold path | API Gateway, React+MapLibre, search-service, Superset, H3/Kepler.gl | ✅ Aprobado | ✅ Implementado |
| 7 | `identidad-seguridad` | 5 · Identidad federada | Keycloak (realm por país), PKI propia (Vault), mTLS, multi-tenant | ✅ Aprobado | ✅ Implementado |

> El Pilar 3 (Cloud-agnostic) es **transversal** — no genera un change propio; sus decisiones (Kubernetes, hexagonal, OSS portable) se aplican como restricciones de diseño en todos los changes anteriores.

---

## Fase de implementación

Todos los proposes están aprobados. La fase siguiente es implementar los documentos de diseño detallado con `/refacil:apply`. Total: **123 tareas** en los 7 changes (ver columna Apply en la tabla de estado).

| # | Change | Tareas |
|---|--------|--------|
| 1 | `agente-borde` | 13 |
| 2 | `ingestion-mqtt` | 12 |
| 3 | `backbone-procesamiento` | 17 |
| 4 | `sincronizacion-paises` | 18 |
| 5 | `almacenamiento-lectura` | 18 |
| 6 | `api-frontend-analitica` | 24 |
| 7 | `identidad-seguridad` | 21 |

**Orden sugerido por dependencias:**
1. `almacenamiento-lectura` — define los schemas que todos los demás leen
2. `identidad-seguridad` — define el contrato JWT que `api-frontend-analitica` consume
3. `api-frontend-analitica` — depende de los dos anteriores
4. Los 4 primeros changes pueden trabajarse en cualquier orden

Para retomar: ver columna **Apply** en la tabla de estado y ejecutar `/refacil:apply <nombre>` sobre el primer change con `⏳ Pendiente`.

---

## Modo de trabajo acordado

1. Entender el pilar desde el problema (análisis ya hecho para Pilar 1).
2. Identificar qué changes nacen de ese pilar.
3. Ejecutar `/refacil:propose <change>` para cada uno.
4. Revisar artefactos (proposal, specs, design, tasks) y aprobar.
5. Pasar al siguiente pilar.

---

## Progreso por pilar

### Pilar 1 — Edge resiliente con store-and-forward
- **Análisis:** completo
- **Changes:** `agente-borde`
- **Estado:** ✅ Aprobado

### Pilar 2 — Comunicación segura y eficiente
- **Análisis:** completo
- **Changes:** `ingestion-mqtt`
- **Estado:** ✅ Aprobado

### Pilar 3 — Cloud-agnostic por diseño *(transversal)*
- **Análisis:** completo (incluido en pilares anteriores)
- **Changes:** ninguno propio — restricciones de diseño en todos

### Pilar 4 — Anti-Corruption Layer por país
- **Análisis:** completo
- **Changes:** `sincronizacion-paises`
- **Estado:** ✅ Aprobado

### Pilar 5 — Identidad federada con Keycloak
- **Análisis:** completo
- **Changes:** `identidad-seguridad`
- **Estado:** ✅ Aprobado

### Pilar 6 — Hot path / Cold path separados
- **Análisis:** completo
- **Changes:** `backbone-procesamiento`, `almacenamiento-lectura`, `api-frontend-analitica`
- **Estado:** ✅ Aprobado

---

## Cómo retomar

1. Abrir este archivo para ver el estado actual.
2. **Fase propose** (completa): todos los changes tienen artefactos aprobados en `refacil-sdd/changes/<nombre>/`.
3. **Fase apply** (siguiente): identificar el primer change con estado `⏳ Pendiente` en la tabla de implementación y ejecutar `/refacil:apply <nombre>`.
4. Al completar el apply de un change, actualizar su estado en la tabla de implementación a `✅ Implementado`.
