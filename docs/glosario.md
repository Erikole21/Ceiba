# Glosario — Sistema Anti-Hurto de Vehículos

Términos técnicos usados a lo largo de los pilares y changes del SDD.
Cuando aparezca un término nuevo que requiera explicación, agregarlo aquí antes de continuar con el siguiente pilar o change.

---

## Arquitectura general

**Dispositivo de borde (edge device)**
Cualquier dispositivo físico instalado en el mundo real (en este caso, en las calles) que forma parte del sistema pero opera lejos del centro de procesamiento (la nube). Tiene recursos limitados (CPU, RAM, disco, batería) y conectividad intermitente. Se llama "de borde" porque está en el extremo de la red, donde el sistema toca la realidad.

**Edge-first / Store-and-forward**
Patrón de diseño donde el dispositivo de borde es autónomo: guarda los datos localmente antes de intentar enviarlos. Si se cae la red o la energía, los datos sobreviven en disco y se transmiten cuando las condiciones lo permiten. Garantiza que no se pierde evidencia ante fallos.

**Hot path**
El camino rápido de procesamiento: desde que se captura una placa hasta que se genera una alerta al oficial. El objetivo es que todo ocurra en menos de 2 segundos.

**Cold path**
El camino lento: analítica, reportes, dashboards históricos, sincronización de bases de datos. No tiene restricciones de tiempo real pero maneja volúmenes masivos de datos.

**Cloud-agnostic**
Diseño que no depende de un proveedor de nube específico (AWS, Azure, GCP). Se logra usando tecnologías open-source portables (Kubernetes, Kafka, PostgreSQL, MinIO) y envolviendo cualquier servicio específico de la nube detrás de una capa de abstracción (puerto/adaptador).

**Multi-tenant**
Un solo sistema que sirve a múltiples "inquilinos" (en este caso, países) de forma aislada. Los datos de Colombia no se mezclan con los de Venezuela, aunque corran en la misma infraestructura.

**ACL — Anti-Corruption Layer** *(patrón de arquitectura)*
Capa de adaptación que protege al núcleo del sistema de los cambios o inconsistencias de sistemas externos. En este proyecto, cada país tiene un adapter que traduce su base policial al modelo canónico interno. No confundir con ACL de broker MQTT (ver abajo).

**ACL — Access Control List** *(seguridad de broker MQTT)*
Lista de reglas que define qué clientes MQTT pueden publicar o suscribirse a qué tópicos. En este proyecto, cada dispositivo solo puede publicar en `devices/{device_id}/*` y suscribirse a sus propios tópicos de configuración y bloom-filter. Una violación produce desconexión inmediata.

**Modelo canónico**
El formato estándar interno al que se traducen todos los datos externos. Define los 9 campos obligatorios de un vehículo hurtado más un campo `extensions` para datos adicionales por país.

**CQRS (Command Query Responsibility Segregation)**
Patrón que separa las escrituras (commands) de las lecturas (queries). Los eventos se escriben en Kafka y luego se proyectan a bases especializadas para lectura: PostgreSQL para consultas operacionales, OpenSearch para búsqueda, ClickHouse para analítica.

**mTLS (mutual TLS)**
Variante de TLS donde ambas partes se autentican mutuamente con certificado. El dispositivo prueba su identidad ante el broker, y el broker prueba la suya ante el dispositivo. Garantiza que solo dispositivos autorizados puedan conectarse.

**PKI (Public Key Infrastructure)**
Infraestructura para emitir, distribuir, renovar y revocar certificados digitales. En este proyecto se usa Vault PKI para emitir un certificado único por dispositivo.

**OTA (Over The Air)**
Actualización remota de software sin intervención física. El dispositivo descarga el nuevo binario, verifica su firma digital y se actualiza solo.

---

## Subsistemas del agente de borde

**ANPR (Automatic Number Plate Recognition)**
Componente que lee placas de vehículos en tiempo real usando la cámara. Ya viene instalado en el dispositivo — el agente no lo reemplaza, lo consume.

**ANPR Adapter SPI**
Capa de traducción entre el agente y el componente ANPR. Como cada fabricante expone la lectura de placas de forma diferente (HTTP, socket UNIX, archivo, named pipe), el Adapter SPI define un contrato fijo y tiene una implementación por cada variante. SPI (*Service Provider Interface*): un patrón donde el contrato está fijo pero las implementaciones son intercambiables.

**Collector**
Subsistema que toma la placa cruda del ANPR Adapter y construye un evento estructurado completo: agrega coordenadas GPS, timestamps (wall-clock UTC + monotónico), ID del dispositivo, y genera un identificador único e idempotente para el evento.

**Queue Manager**
Cola persistente en disco (SQLite WAL). Guarda cada evento antes de intentar enviarlo. Garantiza que un corte de energía no pierda datos. Ordena los eventos por prioridad: los hits del Bloom filter van primero.

**Image Store**
Buffer local de imágenes en el filesystem del dispositivo. Almacena las fotos del vehículo mientras espera subida. Tiene política de rotación automática (7 días o 10 GB, lo que ocurra primero) para no agotar el disco.

**Bloom Filter**
Estructura de datos probabilística y compacta (< 1 MB) que permite consultar localmente si una placa está en la lista de vehículos hurtados, sin necesidad de conexión a la nube. Puede tener falsos positivos (~1%) — puede decir "sí está" cuando no, pero nunca dice "no está" cuando sí. La confirmación definitiva siempre la hace la nube.

**Uploader**
Subsistema que lee los eventos de la cola y los transmite a la nube: metadatos por MQTT y fotos por HTTP (presigned URL). Reintenta hasta recibir confirmación de entrega (PUBACK). Un evento solo se marca como enviado cuando la nube lo confirmó.

**Health Beacon**
Reporte periódico (cada 60 segundos por defecto) del estado de salud del dispositivo: CPU, RAM, disco, batería, señal GSM, drift de reloj. Si deja de llegar, el sistema sabe que el dispositivo tiene un problema.

**Config/OTA Manager**
Subsistema que recibe configuración remota y actualizaciones de binario (OTA) vía MQTT. Verifica la firma digital del binario antes de aplicarlo. Permite cambiar parámetros del agente sin tocar físicamente el dispositivo.

**Watchdog**
Guardian que monitorea que el proceso del agente esté vivo y funcionando. Si detecta un cuelgue, deadlock o consumo anómalo de memoria, le pide al sistema operativo (systemd) que reinicie el proceso automáticamente.

---

## Ingestión y comunicación

**MQTT**
Protocolo de mensajería ligero diseñado para conexiones inestables y dispositivos con recursos limitados. Usa un modelo publicador/suscriptor y soporta QoS (calidad de servicio) para garantizar entrega. Ideal para GSM intermitente.

**QoS 1 (Quality of Service 1)**
Nivel de garantía de entrega en MQTT: el mensaje se entrega al menos una vez. El publicador retiene el mensaje hasta recibir confirmación (PUBACK) del broker. Puede haber duplicados, que el sistema deduplicará en la nube.

**EMQX**
Broker MQTT open-source de alta disponibilidad. Es el servidor al que todos los dispositivos se conectan para publicar sus eventos.

**Presigned URL**
URL temporal generada por el sistema de almacenamiento que permite subir (o descargar) un archivo directamente sin pasar por un servidor intermediario. El dispositivo la solicita, recibe la URL, y hace el PUT de la imagen directo al object storage.

**Object Storage (S3-API)**
Almacenamiento de objetos (archivos) accesible por API compatible con S3 de Amazon. En este proyecto puede ser AWS S3, GCS, Azure Blob o MinIO (on-prem). Las imágenes de los vehículos se guardan aquí.

**Upload Service**
Microservicio cloud que valida la identidad del dispositivo y emite URLs pre-firmadas para que el dispositivo suba imágenes directamente al object storage. También soporta modo proxy-upload para dispositivos que no pueden alcanzar el object storage directamente (NAT/APN corporativa).

**MQTT→Kafka Bridge**
Componente que mueve los mensajes del broker EMQX al topic Kafka `vehicle.events.raw`. En este proyecto se implementa usando el EMQX Rule Engine con un Kafka Data Bridge nativo (EMQX 5.x), lo que garantiza el orden por `device_id` configurando la clave de partición Kafka como `${clientid}`.

**EMQX Rule Engine**
Motor de reglas integrado en EMQX 5.x que permite definir acciones sobre los mensajes MQTT (filtrar, transformar, reenviar). En este proyecto se usa para enrutar eventos de tópicos de dispositivos hacia Kafka mediante el Kafka Data Bridge nativo, eliminando la necesidad de un conector de terceros.

---

## Procesamiento

**Kafka**
Plataforma de streaming distribuida. Actúa como el backbone de eventos del sistema: todos los eventos (capturas, hurtos sincronizados, alertas) pasan por Kafka antes de ser procesados. Permite replay, múltiples consumidores y retención duradera.

**Deduplicator**
Servicio que elimina eventos duplicados que llegan a Kafka. Como el Uploader puede reenviar el mismo evento más de una vez (QoS 1), el Deduplicator los identifica por `event_id` y descarta los repetidos.

**Enrichment Service**
Servicio que enriquece cada evento con contexto adicional: geocodificación de coordenadas (convertir lat/lon a nombre de calle/ciudad), normalización de la placa, calidad de la imagen (confidence, blur).

**Matcher Service**
Servicio que confirma si la placa de un evento está en la lista canónica de vehículos hurtados, consultando Redis. Si hay match, dispara una alerta.

**Kafka Streams**
Librería de procesamiento de streams sobre Apache Kafka. Permite construir aplicaciones que consumen, transforman y producen eventos en Kafka con estado local (por ejemplo, tablas RocksDB para deduplicación). El Deduplicator de este proyecto usa Kafka Streams con un state store RocksDB para identificar y descartar eventos duplicados por `event_id`.

**Alert Service**
Servicio que recibe los matches confirmados del Matcher Service, genera la alerta, la persiste y la rutea a los oficiales de la zona donde se detectó el vehículo. Se comunica con el WebSocket Gateway para el push en tiempo real.

**WebSocket Gateway**
Componente que mantiene conexiones WebSocket persistentes con los clientes (app policial) y entrega alertas en tiempo real cuando el Alert Service las genera. Los oficiales reciben la notificación con ubicación y thumbnail del vehículo sin necesidad de hacer polling.

**incident-service**
Servicio que gestiona el ciclo de vida de la recuperación de un vehículo: despacho de unidad al último punto conocido, escalación, cierre de caso y propagación del estado "recuperado" al sistema del país correspondiente. Persiste el historial en PostgreSQL con audit log.

**CDC (Change Data Capture)**
Técnica para capturar los cambios en una base de datos (inserts, updates, deletes) en tiempo real, leyendo el log de transacciones (WAL en PostgreSQL). En este proyecto, Debezium usa CDC para sincronizar datos de PostgreSQL hacia ClickHouse.

**Debezium**
Plataforma open-source de CDC que lee el WAL de PostgreSQL y publica los cambios como eventos en Kafka. En este proyecto se usa para alimentar ClickHouse con datos ya enriquecidos y validados desde PostgreSQL.

**Canonical Vehicles Service**
Servicio que mantiene la tabla canónica de vehículos hurtados a partir de los eventos publicados por los adapters de cada país. Actualiza PostgreSQL y Redis (hot-list para el Matcher) cuando llega un cambio de la BD policial.

**Edge Distribution Service**
Servicio que calcula los deltas del Bloom filter por país y los distribuye a los dispositivos vía MQTT retained topic. Cuando se agrega o elimina una placa de la lista canónica, regenera solo la diferencia (no el filtro completo) para minimizar el tráfico GSM.

---

## Almacenamiento

**PostgreSQL + PostGIS**
Base de datos relacional con extensión geoespacial. Almacena eventos consultables con soporte para consultas de ubicación (radio de búsqueda, trayectoria en mapa).

**OpenSearch**
Motor de búsqueda y análisis. Permite buscar una placa rápidamente con fuzzy matching (tolera errores de tipeo) y agregaciones temporales.

**ClickHouse**
Base de datos columnar de alto rendimiento para analítica masiva: heatmaps de zonas de hurto, rutas de escape, tendencias por ciudad. Opera sobre volúmenes enormes en segundos.

**Redis**
Base de datos en memoria. Se usa como caché de la lista de vehículos hurtados (para que el Matcher responda en microsegundos), sesiones de usuario y pub/sub de alertas.

**Patroni**
Solución open-source de alta disponibilidad para PostgreSQL. Gestiona un clúster con un nodo primario y una o más réplicas, detecta fallos del primario y promueve automáticamente la réplica más actualizada en segundos. Usa un DCS (etcd, Consul o ZooKeeper) como árbitro para evitar split-brain. En este proyecto es la opción HA on-prem; en cloud se usa el equivalente gestionado (RDS Multi-AZ, CloudSQL HA) con el mismo Helm chart parametrizado.

**ILM (Index Lifecycle Management)**
Política de ciclo de vida de índices en OpenSearch/Elasticsearch. Define fases automáticas: hot (escritura activa, réplicas completas), warm (solo lectura, réplicas reducidas, merge forzado), cold (congelado, acceso esporádico) y delete (eliminación al cumplir el plazo de retención). En este proyecto gestiona los índices rotatorios `vehicle-events-{country_code}-{YYYY-MM}` con retención de 24 meses.

**Schema Registry (Avro)**
Servicio que almacena y versiona los esquemas de los mensajes Kafka (en formato Avro). Garantiza que productores y consumidores usen el mismo contrato de datos y permite evolucionar los esquemas de forma compatible hacia atrás.

---

## API, Frontend y Analítica

**API Gateway**
Punto único de entrada para todas las solicitudes de los clientes (web app, móvil). Centraliza rate-limiting, validación de JWT (contra Keycloak), routing hacia los microservicios internos y observabilidad. En este proyecto puede ser Kong, Envoy o NGINX Ingress; si se usa un servicio cloud (AWS API GW), queda detrás de un adaptador hexagonal.

**search-service**
Microservicio que orquesta la búsqueda de una matrícula: consulta OpenSearch para obtener los `event_id` relevantes, luego PostgreSQL para los datos enriquecidos con coordenadas, y finalmente genera las URLs pre-firmadas de thumbnails desde el object storage. Devuelve una lista paginada de avistamientos lista para renderizar en el mapa.

**MapLibre GL**
Librería open-source de renderizado de mapas vectoriales en el navegador. Reemplaza a Mapbox GL con licencia libre. Se usa en la Web App para mostrar la trayectoria del vehículo (cada avistamiento como un punto en el mapa con información de fecha/hora y thumbnail).

**Apache Superset**
Plataforma de BI open-source para crear dashboards interactivos. En este proyecto se conecta a ClickHouse para visualizar tendencias de hurto, zonas peligrosas y rutas de escape. Cloud-agnostic: se despliega como contenedor en Kubernetes.

**H3 (sistema de índice geoespacial hexagonal)**
Sistema de indexación geoespacial de Uber que divide el mundo en celdas hexagonales de distintas resoluciones. Se usa para agrupar eventos de tránsito en zonas (hexbinning) y construir heatmaps de densidad de hurtos. ClickHouse tiene funciones nativas de H3.

**SLO (Service Level Objective)**
Objetivo de nivel de servicio: la meta interna de rendimiento o disponibilidad que el sistema debe cumplir. En este proyecto cada operación crítica tiene su SLO definido (p.ej. búsqueda de placa p95 < 300 ms, alerta end-to-end p95 < 2 s, dashboard analítico p95 < 3 s). Diferente de SLA (acuerdo contractual con el cliente): el SLO es la meta de ingeniería que permite cumplir el SLA con margen.

**rate-limiting**
Control del número máximo de solicitudes que un cliente puede hacer a la API en un período de tiempo. Protege los servicios de abuso, ataques de fuerza bruta y picos involuntarios. En este proyecto se implementa en el API Gateway (Kong/Envoy) con ventana deslizante por tenant y endpoint; Redis almacena los contadores con expiración automática.

**Kepler.gl**
Herramienta de visualización geoespacial open-source (también de Uber). Se integra en la Web App para capas analíticas avanzadas: heatmaps de hurtos, rutas recurrentes de escape, animaciones temporales de eventos.

---

## Identidad

**Keycloak**
Servidor de identidad open-source. Gestiona autenticación y autorización. En este proyecto actúa como broker central: cada país tiene su propio "realm" y Keycloak federa con los sistemas de autenticación de cada policía (LDAP, bases de datos, SOAP, REST).

**OIDC (OpenID Connect)**
Protocolo estándar de autenticación sobre OAuth 2.0. Las apps reciben un JWT firmado que prueba quién es el usuario y qué puede hacer.

**JWT (JSON Web Token)**
Token compacto y firmado digitalmente que contiene la identidad y permisos del usuario. Se valida criptográficamente sin necesidad de consultar la base de datos en cada request.

**LDAP**
Protocolo para directorios de usuarios (el más común en instituciones policiales y empresas). Keycloak puede federar con LDAP nativamente.

**RBAC (Role-Based Access Control)**
Control de acceso basado en roles: oficial, supervisor, analista, administrador, auditor. Cada rol tiene permisos distintos y pueden ser restringidos por país/ciudad.

**realm**
Espacio de configuración aislado dentro de Keycloak. Cada realm tiene sus propios usuarios, roles, clientes, políticas de contraseña y proveedores de identidad federados. En este proyecto hay un realm por país, lo que garantiza que las credenciales, sesiones y configuraciones de Colombia no interfieren con las de Venezuela ni con las de ningún otro país.

**SSO (Single Sign-On)**
Inicio de sesión único: el usuario se autentica una vez y accede a todas las aplicaciones del ecosistema sin volver a ingresar credenciales. Keycloak lo implementa mediante sesiones OIDC compartidas entre clientes del mismo realm.

**MFA (Multi-Factor Authentication)**
Autenticación multifactor: además de la contraseña, el usuario debe verificar su identidad con un segundo factor (OTP, TOTP, push a dispositivo móvil). En este proyecto es configurable por realm; cuando la institución policial lo exige, Keycloak lo aplica sin cambios en las aplicaciones.

**SPI (Service Provider Interface) — contexto Keycloak**
Mecanismo de extensión de Keycloak que permite agregar implementaciones personalizadas de federación de identidad. Las más relevantes en este proyecto son: **User Storage SPI** (conecta Keycloak a una fuente de usuarios externa como una BD policial o un servicio SOAP) e **Identity Provider SPI** (permite federar con sistemas que no hablan OIDC ni SAML estándar). El SPI define el contrato; cada country adapter implementa el proveedor concreto. No confundir con el ANPR Adapter SPI del agente de borde.

**SAML (Security Assertion Markup Language)**
Protocolo de federación de identidad basado en XML, muy común en sistemas institucionales legados. Permite que una institución policial actúe como Identity Provider (IdP) y delegue el inicio de sesión a Keycloak mediante assertions firmadas. Alternativa a OIDC cuando la policía no soporta el protocolo moderno.

**HashiCorp Vault**
Plataforma open-source para gestión centralizada de secretos y PKI. En este proyecto cumple dos roles: (1) emitir, renovar y revocar certificados de dispositivo (Vault PKI Engine) con TTL de 90 días; (2) custodia de secretos de aplicación (claves de API, credenciales de base de datos) con auto-rotación. Cloud-agnostic: se despliega en Kubernetes igual en cualquier nube o on-prem.

**CSR (Certificate Signing Request)**
Solicitud de firma de certificado: el dispositivo genera un par de claves asimétricas, crea un CSR con su identidad (device_id, país) y lo envía a la CA (Vault PKI). La CA devuelve el certificado firmado. En este proyecto el agente genera el CSR automáticamente antes del vencimiento del certificado actual, renovando su identidad sin intervención humana.

**CRL / OCSP**
Mecanismos de revocación de certificados. **CRL (Certificate Revocation List)**: lista periódica publicada por la CA con los certificados revocados antes de su vencimiento. **OCSP (Online Certificate Status Protocol)**: consulta en tiempo real al responder de la CA sobre si un certificado específico sigue siendo válido. Se usan cuando un dispositivo es comprometido o dado de baja para invalidar su certificado de inmediato.

**Bootstrap token**
Token de un solo uso que permite a un dispositivo nuevo intercambiarlo por su primer certificado operativo emitido por Vault PKI. El fabricante lo provisiona en el dispositivo de fábrica (o el operador lo inyecta en el primer arranque). Una vez usado, el token se invalida y el dispositivo usa el ciclo normal de renovación por CSR.
