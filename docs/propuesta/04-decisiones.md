# 4. Decisiones de Diseño

← [Índice](../propuesta-arquitectura-hurto-vehiculos.md)

Cada decisión se presenta en formato ADR resumido: contexto, decisión, alternativas, consecuencias. Los ADRs de cada pilar con mayor detalle viven en las carpetas de especificación correspondientes.

---

### ADR-001 · Patrón Edge-First con Store-and-Forward

- **Contexto.** La energía falla hasta 4 h y la batería dura 1 h; GSM no es continua. No se puede perder evidencia de tránsito.
- **Decisión.** Cada dispositivo ejecuta un agente autónomo que persiste cada evento en disco (SQLite WAL) antes de intentar enviarlo. Reintenta indefinidamente con backoff exponencial. Cada evento lleva un `event_id` idempotente.
- **Alternativas.**
  - *Streaming directo sin buffer:* descartada, pierde datos en cortes.
  - *Buffer en RAM:* descartada, no sobrevive a corte de energía.
- **Consecuencias.** El sistema central debe deduplicar (lo hace el Deduplicator). El disco de 16 GB acomoda meses de metadatos y semanas de imágenes con rotación.

### ADR-002 · MQTT 5 como Protocolo Principal de Telemetría

- **Contexto.** Conexión GSM intermitente, payloads pequeños y frecuentes, miles a millones de dispositivos.
- **Decisión.** MQTT 5 con QoS 1, sesiones persistentes, keep-alive 60 s, autenticación mTLS por device. Broker EMQX en cluster.
- **Alternativas.**
  - *HTTP/2 o REST:* mayor overhead por conexión, sin QoS nativa.
  - *gRPC streaming:* sufre en GSM intermitente.
  - *AMQP:* más pesado, optimizado para escenarios distintos.
- **Consecuencias.** Se opera un broker dedicado con su propia HA. Las imágenes no viajan por MQTT — se separan en HTTPS para no saturar el broker.

### ADR-003 · Imágenes vía URLs Pre-firmadas a Object Storage

- **Contexto.** Las imágenes pesan dos a tres órdenes de magnitud más que la metadata. Saturar el broker con ellas es contraproducente.
- **Decisión.** El agente solicita una URL pre-firmada al Upload Service y hace `PUT` directo al object storage (S3-API). En el evento MQTT viaja sólo la URI resultante.
- **Alternativas.**
  - *Subir por MQTT:* satura el broker.
  - *Subir por HTTP a un backend:* cuello de botella y duplica tráfico.
- **Consecuencias.** El dispositivo necesita acceso de red al object storage. Si no es factible (NAT, regulaciones), se interpone un proxy de subida.

### ADR-004 · Kubernetes como Plataforma de Cómputo

- **Contexto.** Necesidad de escalar dinámicamente, alta disponibilidad, portabilidad entre nubes.
- **Decisión.** Kubernetes en la nube elegida (EKS / GKE / AKS) y RKE2/Rancher para on-prem. Todos los servicios se empaquetan como contenedores con manifiestos Helm.
- **Alternativas.**
  - *FaaS / Serverless:* acoplado a la nube.
  - *VMs con autoscaling:* peor utilización, operación más manual.
  - *Nomad:* menos ecosistema.
- **Consecuencias.** Se asume el costo operativo de K8s. Se gana la unidad de despliegue entre cualquier infraestructura.

### ADR-005 · Arquitectura Hexagonal para Servicios Sensibles a la Nube

- **Contexto.** Requisito explícito de evitar acoplamiento con un proveedor cloud.
- **Decisión.** Encapsular en puertos/adaptadores todo lo que toca cloud: object storage, KMS, secret manager, colas administradas. Implementaciones intercambiables: S3 ↔ GCS ↔ Azure Blob ↔ MinIO.
- **Alternativas.**
  - *SDK cloud directo:* lock-in, costoso migrar.
  - *Sólo OSS:* pierde ventajas managed.
- **Consecuencias.** Mayor esfuerzo inicial. Migración entre nubes factible en semanas, no meses.

### ADR-006 · Anti-Corruption Layer por País + Modelo Canónico Versionado

- **Contexto.** Cada policía tiene formato y tecnología distintos para vehículos hurtados.
- **Decisión.** Por cada país, un microservicio adapter dedicado. El modelo canónico tiene los campos obligatorios y una propiedad `extensions: jsonb`. El esquema está versionado (Avro + Schema Registry).
- **Alternativas.**
  - *Imponer un único formato a las policías:* no factible institucionalmente.
  - *Adaptador genérico parametrizable:* nunca cubre los casos reales (CDC, SOAP heredado, drops de archivo).
- **Consecuencias.** Costo de mantenimiento por país. A cambio, el núcleo está aislado de cualquier cambio de esquema externo.

> **ADR-006 detallado:** [`docs/sincronizacion-paises/adr-006-acl-canonical-model.md`](../sincronizacion-paises/adr-006-acl-canonical-model.md)

### ADR-007 · Keycloak como Broker de Identidad

- **Contexto.** Las policías usan LDAP, DBs, SOAP y REST para autenticación. El sistema debe integrarse con todas.
- **Decisión.** Un Keycloak central con un *realm* por país. Federación nativa para LDAP y DB. SPI custom para SOAP/REST. Las apps reciben JWT estándar OIDC.
- **Alternativas.**
  - *Construir OIDC propio:* reinventar la rueda.
  - *Cognito / Azure AD B2C:* lock-in, menos flexible para federaciones heterogéneas.
- **Consecuencias.** Keycloak es una pieza crítica con su propia operación. Se gana un único punto de federación y SSO real.

### ADR-008 · CQRS Pragmático con Eventos como Fuente

- **Contexto.** Alta tasa de eventos, necesidad de lectura optimizada y analítica.
- **Decisión.** Los eventos en Kafka son el log de cambios. Servicios independientes proyectan a Postgres (consultas operacionales), OpenSearch (búsqueda), ClickHouse (analítica). Sin event-sourcing puro.
- **Alternativas.**
  - *Una sola DB para todo:* no rinde tanto en transaccional como en analítica.
  - *Event-sourcing puro:* exceso de complejidad para este caso.
- **Consecuencias.** Consistencia eventual entre proyecciones. Lecturas mucho más rápidas.

### ADR-009 · Bloom Filter en Borde para Hot-Path de Alertas

- **Contexto.** Un dispositivo puede ver miles de placas al día, pero sólo una fracción corresponde a vehículos hurtados.
- **Decisión.** Cada dispositivo mantiene un Bloom filter de las placas hurtadas de su país (< 1 MB para 1 M placas con FP < 1 %). El filtro se actualiza incrementalmente vía MQTT retained topic.
- **Alternativas.**
  - *Lista completa en disco:* costosa de sincronizar y consultar.
  - *Sólo cloud-side matching:* añade latencia y depende de conectividad.
- **Consecuencias.** La confirmación canónica siempre ocurre en cloud para evitar falsos positivos del BF.

> **ADR-009 detallado:** [`docs/sincronizacion-paises/adr-009-bloom-filter-edge.md`](../sincronizacion-paises/adr-009-bloom-filter-edge.md)

### ADR-010 · PKI Propia con Certificados Cortos para Dispositivos

- **Contexto.** Necesidad de identidad fuerte por dispositivo, revocación eficiente, rotación segura.
- **Decisión.** Vault PKI emite un certificado por dispositivo con TTL 90 días. El agente rota automáticamente antes del vencimiento. Provisioning inicial vía bootstrap token de un solo uso.
- **Alternativas.**
  - *API keys estáticas:* débiles, difíciles de revocar masivamente.
  - *Cert long-lived sin rotación:* riesgo si una unidad es comprometida.
- **Consecuencias.** Operación PKI no trivial pero estándar. Revocación inmediata por device sin afectar a otros.

### ADR-011 · Multi-tenant por País con Aislamiento de Datos

- **Contexto.** Soberanía de datos, regulaciones por país, separación de operación.
- **Decisión.** `country_code` como discriminador de tenant en todas las tablas y topics. Posibilidad de despliegues regionales separados. Keycloak con un realm por país. Bloom filters segmentados por país.
- **Alternativas.**
  - *Tenant único global:* ignora regulación local.
  - *Despliegue completamente separado por país:* multiplica costos de operación.
- **Consecuencias.** Diseño multi-tenant disciplinado desde el inicio.

### ADR-012 · Stack de Streaming sobre Kafka (con opción Redpanda)

- **Contexto.** Necesidad de backbone durable, alto throughput, compatible cloud-agnostic.
- **Decisión.** Apache Kafka como referencia; se permite Redpanda en clústeres on-prem por simplicidad operacional (sin ZooKeeper, mismo API).
- **Alternativas.**
  - *AWS Kinesis / Google Pub-Sub:* lock-in.
  - *RabbitMQ:* no diseñado para replay masivo ni retention prolongado.
- **Consecuencias.** Se gana ecosistema (Connect, Streams, Schema Registry, Debezium). Operación no trivial — se mitiga con managed (MSK, Confluent Cloud) detrás de la abstracción cuando aplica.
