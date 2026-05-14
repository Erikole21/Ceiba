# design: ingestion-mqtt

## Archivos a crear

| Ruta | Propósito |
|------|-----------|
| `docs/ingestion-mqtt/overview.md` | Descripción narrativa de la capa de ingestión: flujo completo device→EMQX→Bridge→Kafka y device→UploadService→ObjectStorage, decisiones ADR aplicadas, diagrama Mermaid de componentes |
| `docs/ingestion-mqtt/emqx-cluster.md` | Especificación del clúster EMQX: configuración de mTLS, ACL por `device_id` y `country_code`, esquema completo de tópicos, `session_expiry_interval`, keepalive, límites de payload, dimensionamiento de nodos y parámetros de HA |
| `docs/ingestion-mqtt/topic-schema.md` | Esquema autoritativo de tópicos MQTT: tabla de tópicos, dirección, contenido, QoS, retained flag, restricciones ACL y ejemplos de payload por tópico |
| `docs/ingestion-mqtt/upload-service.md` | Especificación del Upload Service: API REST (endpoints, contratos de request/response), flujo de validación de identidad, emisión de presigned URL, modo proxy-upload, arquitectura hexagonal con puerto `ObjectStoragePort` e implementaciones adaptadoras |
| `docs/ingestion-mqtt/object-storage.md` | Especificación del object storage: estructura de paths, política de ciclo de vida, configuración de cifrado KMS, replicación Multi-AZ, política de bucket, adaptadores soportados (MinIO, AWS S3, GCS, Azure Blob) |
| `docs/ingestion-mqtt/bridge.md` | Especificación y decisión ADR del MQTT→Kafka Bridge: justificación de la opción elegida (Kafka Connect vs. bridge Go custom), configuración del conector/bridge, garantía de orden por `device_id`, manejo de backpressure y reintentos, métricas de monitoreo |
| `docs/ingestion-mqtt/security.md` | Especificación de seguridad de la capa de ingestión: flujo mTLS completo, validación OCSP contra Vault, política de revocación CRL, rotación de certificados de dispositivo sin downtime, restricciones de ACL, cifrado en tránsito y en reposo |
| `docs/ingestion-mqtt/operations.md` | Runbook operativo: procedimiento de escalado de nodos EMQX, gestión de tormenta de reconexiones, rotación de certificados CA, monitoreo de lag Kafka, alertas críticas y procedimientos de recuperación |
| `docs/ingestion-mqtt/helm/README.md` | Guía de despliegue con Helm: valores configurables para EMQX, Upload Service, MinIO y Bridge; instrucciones de instalación, actualización y rollback en Kubernetes |
| `docs/ingestion-mqtt/terraform/README.md` | Guía de módulos Terraform: estructura de módulos para infraestructura cloud (object storage, networking, KMS), variables de entrada/salida, ejemplos de uso por proveedor |
| `docs/ingestion-mqtt/adr-bridge-decision.md` | ADR específico del change: decisión entre Kafka Connect + MQTT source connector vs. bridge Go custom, criterios evaluados (latencia, mantenimiento, ops, ordering guarantee), decisión tomada y justificación |

## Archivos a modificar

| Ruta | Cambios |
|------|---------|
| `docs/propuesta-arquitectura-hurto-vehiculos.md` | Agregar referencias cruzadas en la sección del Pilar 2 (Comunicación segura) hacia `docs/ingestion-mqtt/`; actualizar la tabla de componentes de la capa de ingestión (EMQX, Upload Service, Bridge) con enlaces a las especificaciones detalladas; no alterar diagramas Mermaid existentes ni atributos de calidad |

## Archivos fuera de alcance (doNotTouch)

- `refacil-sdd/` — artefactos SDD gestionados por la metodología
- `.claude/`, `.cursor/` — configuración de agentes
- `AGENTS.md` — índice del proyecto; solo se modifica en el change `setup` o por instrucción explícita del autor
- `.agents/` — documentación de sesión; no se modifica en este change
- `refacil-sdd/changes/agente-borde/` — change aprobado; no se modifica

## Decisión de diseño: MQTT→Kafka Bridge (adr-bridge-decision.md)

Esta es la única decisión de diseño pendiente que requiere elección explícita. Se analizan dos opciones:

**Opción A — Kafka Connect + MQTT source connector**
- Ventajas: menor código a mantener, operación declarativa via API Kafka Connect, reintentos y dead-letter queue nativos, integración con Kafka ecosystem (Schema Registry, SMTs).
- Desventajas: latencia adicional ~100 ms por el poll-based connector, dependencia de un conector de terceros (p.ej. Confluent MQTT Connector o plugin EMQX Kafka Bridge nativo), complejidad de configuración de ordering por `device_id` vía SMT o partitioner custom.
- Ordering: se garantiza configurando la clave del mensaje Kafka = `device_id` mediante un Single Message Transform (SMT) o configuración nativa del conector EMQX.

**Opción B — Bridge Go custom**
- Ventajas: latencia < 10 ms, control total sobre la lógica de particionado, sin dependencias externas de terceros.
- Desventajas: código a mantener, requiere implementar backoff, dead-letter, health checks y métricas desde cero.

**Decisión recomendada: Opción A (Kafka Connect + conector MQTT nativo de EMQX)**

EMQX 5.x incluye un Data Bridge nativo hacia Kafka que opera a nivel de reglas (Rule Engine), lo que elimina la dependencia de un conector de terceros y reduce la latencia adicional a < 50 ms. La clave de partición Kafka se configura directamente como `${clientid}` (equivalente a `device_id`) en la regla de bridge. Esta opción minimiza el código a mantener, es operable de forma declarativa y satisface el requisito de ordering. La decisión debe documentarse en `docs/ingestion-mqtt/adr-bridge-decision.md`.

## Patrones y convenciones detectados

- **Idioma:** español formal orientado a audiencia técnica senior (regla `Always` en `.agents/summary.md`); identificadores, rutas y nombres de archivo en inglés.
- **Diagramas:** Mermaid para todo diagrama nuevo; usar secuencia para flujos request/response y flowchart/graph para topología de componentes, igual que en `docs/agente-borde/`.
- **Terminología canónica:** "agente de borde", "clúster EMQX", "bridge MQTT→Kafka", "URL pre-firmada", "cifrado en reposo", "ACL por país"; mantener consistencia con la propuesta principal y con `docs/agente-borde/`.
- **Referencias cruzadas ADR:** toda decisión técnica debe mencionar explícitamente el ADR que la origina (ADR-002, ADR-003, ADR-005, ADR-010, ADR-011).
- **Estructura de sección:** encabezados H2/H3 sin numeración propia en documentos bajo `docs/ingestion-mqtt/`; la numeración existe solo en la propuesta principal.
- **Privacidad:** los documentos de esta capa deben explicitar que el object storage almacena imágenes de vehículos (no rostros ni biometría), con retención máxima de 24 meses y acceso restringido por `country_code`.
- **Hexagonal:** el Upload Service expone `ObjectStoragePort` como interfaz; los adaptadores (MinIO, S3, GCS, Azure Blob) son implementaciones intercambiables sin modificar el núcleo — igual patrón que el ANPR Adapter SPI de `agente-borde`.

## Dependencias entre tareas

1. `docs/ingestion-mqtt/overview.md` debe redactarse primero — establece el glosario, el flujo de datos de la capa y el diagrama de componentes que referencian todos los demás documentos.
2. `docs/ingestion-mqtt/topic-schema.md` debe completarse antes de `docs/ingestion-mqtt/emqx-cluster.md` y antes de `docs/ingestion-mqtt/bridge.md`, ya que el esquema de tópicos es el contrato compartido entre el broker, el agente de borde y el bridge.
3. `docs/ingestion-mqtt/adr-bridge-decision.md` debe redactarse antes de `docs/ingestion-mqtt/bridge.md` — la especificación del bridge depende de la decisión documentada en el ADR.
4. `docs/ingestion-mqtt/upload-service.md` y `docs/ingestion-mqtt/object-storage.md` pueden redactarse en paralelo entre sí, pero ambos deben completarse antes de `docs/ingestion-mqtt/security.md` (la sección de seguridad del upload cubre ambos componentes).
5. `docs/ingestion-mqtt/helm/README.md` y `docs/ingestion-mqtt/terraform/README.md` dependen de que los componentes estén especificados; son las últimas tareas de documentación.
6. La modificación de `docs/propuesta-arquitectura-hurto-vehiculos.md` es el paso final, una vez consolidados todos los documentos de `docs/ingestion-mqtt/`.

## Nota de validación cruzada entre repositorios

Este change establece contratos con componentes especificados en otros changes. Consultar `refacil-prereqs/BUS-CROSS-REPO.md` para el proceso de acuerdo formal:

- **`agente-borde` (upstream, aprobado):** el esquema de tópicos MQTT (`topic-schema.md`), el formato de payload de `devices/{device_id}/events` y el protocolo de solicitud de URL pre-firmada al Upload Service deben ser consistentes con lo especificado en `docs/agente-borde/uploader.md`. Cualquier cambio en el contrato de la URL pre-firmada (endpoint, método HTTP, headers requeridos, TTL, formato de respuesta) requiere notificación al change `agente-borde`.
- **`backbone-procesamiento` (downstream, pendiente):** el tópico Kafka `vehicle.events.raw`, el esquema del mensaje (incluyendo la clave de partición `device_id`), el formato del payload y el número de particiones deben acordarse con ese change antes de la implementación del bridge.
- **`identidad-seguridad` (pendiente):** el mecanismo de aprovisionamiento del certificado mTLS en el dispositivo, el endpoint OCSP de Vault, el intervalo de rotación (TTL 90 días) y el formato del CN/SAN del certificado son contratos que este change consume; cualquier cambio debe notificarse a este change.
