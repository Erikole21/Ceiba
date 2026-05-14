# 1. Resumen de la Propuesta

← [Índice](../propuesta-arquitectura-hurto-vehiculos.md)

---

Se propone una **plataforma distribuida edge-to-cloud** que captura eventos de tránsito en dispositivos de borde con recursos limitados, los sincroniza de forma confiable hacia una nube portable, los enriquece y correlaciona contra bases de datos heterogéneas de vehículos hurtados por país, y los expone a usuarios policiales mediante consultas, mapas, alertas y analítica.

La arquitectura se sostiene sobre seis pilares:

1. **Edge resiliente con store-and-forward.** Cada dispositivo opera de forma autónoma con cola persistente local, generación de alertas locales contra Bloom-filters de placas hurtadas, y reintento idempotente cuando recupera energía o conectividad. Hasta 4 h sin red y hasta 1 h sin energía son escenarios cubiertos. *Esto responde al requisito de garantizar la recolección y sincronización de datos ante fallos de energía y comunicaciones, dado el hardware limitado (2 cores, 1 GB RAM, 16 GB disco) con batería de respaldo y GSM intermitente.*

   > **Especificación detallada — Pilar 1:** [`docs/agente-borde/`](../agente-borde/overview.md)

2. **Comunicación segura y eficiente.** MQTT 5 sobre TLS con autenticación mutua mTLS por certificado de dispositivo emitido por una PKI propia. Imágenes vía URLs pre-firmadas a object storage para no saturar el broker. *Esto responde al requisito de garantizar una comunicación segura entre los dispositivos y el sistema, evitando información de dispositivos o personas no autorizadas, con un protocolo adecuado para conexiones GSM intermitentes y payloads de distintos tamaños.*

   > **Especificación detallada — Pilar 2:** [`docs/ingestion-mqtt/`](../ingestion-mqtt/overview.md)

3. **Cloud-agnostic por diseño.** Stack basado en Kubernetes, PostgreSQL, Kafka, Keycloak, Redis, OpenSearch, MinIO/S3-API, ClickHouse. Servicios sensibles a la nube quedan detrás de puertos/adaptadores hexagonales para permitir migración o despliegue on-prem. *Esto responde al requisito de evitar el acoplamiento con un proveedor cloud, poder cambiar de proveedor o montar una nube privada, y escalar dinámicamente desde Latinoamérica hasta cobertura global.*

4. **Anti-Corruption Layer por país.** Cada base de datos policial se sincroniza a un modelo canónico mediante un adaptador específico; cambios se propagan como eventos al núcleo. *Esto responde al requisito de integrar las bases de datos de vehículos hurtados de cada país, que pueden tener diferente formato y tecnología, unificándolas bajo los 9 campos canónicos requeridos.*

   > **Especificación detallada — Pilar 4:** [`docs/sincronizacion-paises/`](../sincronizacion-paises/overview.md)

5. **Identidad federada con Keycloak como broker.** Un realm por país como barrera técnica de aislamiento multi-tenant; federación nativa de LDAP y bases de datos, SPI custom para SOAP/REST y delegación a IdPs SAML/OIDC del país; tokens JWT con claims canónicos `country_code`/`role`/`zone`; cinco roles RBAC con scope geográfico por zona; PKI de dispositivos via HashiCorp Vault con certificados X.509 TTL 90 días y mTLS obligatorio en EMQX. *Esto responde al requisito de integrar el sistema con los mecanismos de autenticación heterogéneos de las instituciones policiales.*

   > **Especificación detallada — Pilar 5:** [`docs/identidad-seguridad/`](../identidad-seguridad/overview.md)

6. **Hot path / Cold path separados.** Ruta caliente para matching y alertas (objetivo p95 < 2 s desde captura). Ruta fría para analítica y dashboards (Superset, ClickHouse, PostGIS). *Esto responde al doble requisito: (a) buscar una matrícula y visualizar todos los avistamientos con prueba visual; y (b) identificar tendencias mediante análisis de datos.*

   > **Especificación detallada — Pilar 6:** [`docs/backbone-procesamiento/`](../backbone-procesamiento/overview.md) · [`docs/api-frontend-analitica/`](../api-frontend-analitica/overview.md)

El resultado: un sistema **portable**, **escalable horizontalmente por país y por región**, **alta disponibilidad (objetivo 99.9 %)**, **tolerante a fallos de borde**, **seguro de extremo a extremo**, que **cierra el ciclo operacional**: desde la detección en la calle hasta la recuperación del vehículo y la actualización de la base policial.
