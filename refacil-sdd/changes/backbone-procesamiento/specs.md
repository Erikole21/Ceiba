# specs: backbone-procesamiento

---

## CA-01: Deduplicación de evento duplicado dentro de la ventana de 24 h

**Dado** que el Deduplicator ha procesado un evento con `event_id = "E-001"` y ese `event_id` permanece en el estado RocksDB
**Cuando** el mismo `event_id` llega nuevamente al tópico `vehicle.events.raw` dentro de las 24 h siguientes (reintento QoS 1 del agente de borde)
**Entonces** el Deduplicator descarta el duplicado sin producirlo a `vehicle.events.deduped` y registra un contador de métrica `deduplicator_duplicates_discarded_total`

## CA-02: Evento único pasa la deduplicación y avanza al pipeline

**Dado** un evento con `event_id = "E-002"` que nunca ha sido procesado
**Cuando** llega al tópico `vehicle.events.raw`
**Entonces** el Deduplicator lo produce a `vehicle.events.deduped` preservando todos los campos originales del payload y usando `device_id` como clave de partición Kafka

## CA-03: Enriquecimiento con geocodificación inversa

**Dado** un evento en `vehicle.events.deduped` con campos `lat` y `lon` válidos
**Cuando** el Enrichment Service procesa el evento
**Entonces** el evento enriquecido contiene los campos `street`, `city` y `country` resueltos por geocodificación inversa; el campo `country_code` del resultado es consistente con el `country_code` del evento original

## CA-04: Normalización de placa por país

**Dado** un evento con `plate = "abc 123-x"` y `country_code = "CO"`
**Cuando** el Enrichment Service aplica la normalización de placa para Colombia
**Entonces** el campo `plate_normalized` del evento enriquecido es `"ABC123X"` (mayúsculas, sin separadores, sin espacios)

## CA-05: Evaluación de calidad de imagen

**Dado** un evento con referencia a una imagen en object storage
**Cuando** el Enrichment Service evalúa la calidad de la imagen
**Entonces** el evento enriquecido contiene `image_confidence_score` (0.0–1.0) y `image_blur_flag` (booleano); si `image_blur_flag = true` se registra en la métrica `enrichment_image_blur_detected_total`

## CA-06: Proyección de evento enriquecido a PostgreSQL con PostGIS

**Dado** un evento enriquecido procesado por el Enrichment Service
**Cuando** se ejecuta la proyección a PostgreSQL
**Entonces** se inserta una fila en la tabla de eventos con la columna `location` de tipo `GEOGRAPHY(POINT, 4326)` con las coordenadas del evento, el campo `country_code` como discriminador de tenant y todos los campos del evento enriquecido

## CA-07: Proyección de evento enriquecido a OpenSearch

**Dado** un evento enriquecido con `country_code = "CO"` y timestamp `2026-05-12T...`
**Cuando** el Enrichment Service escribe en el índice OpenSearch
**Entonces** el documento se indexa en `vehicle-events-co-2026-05` (índice rotatorio por país + mes) con los campos `plate_normalized`, `timestamp`, `device_id`, `country_code` y `event_id` buscables; la búsqueda exacta y fuzzy por `plate_normalized` devuelve el documento en menos de 200 ms

## CA-08: Match positivo en Redis produce alerta

**Dado** un evento en `vehicle.events.enriched` con `plate_normalized = "ABC123X"` y `country_code = "CO"`, y una entrada `stolen:CO:ABC123X` presente en Redis
**Cuando** el Matcher Service ejecuta `GET stolen:CO:ABC123X`
**Entonces** el Matcher produce un evento de alerta al tópico `vehicle.alerts` con los campos `event_id`, `plate_normalized`, `country_code`, `device_id`, `location` y `timestamp`

## CA-09: Miss en Redis no genera alerta

**Dado** un evento en `vehicle.events.enriched` cuya placa no existe en la lista roja de Redis
**Cuando** el Matcher Service ejecuta `GET stolen:{country_code}:{plate_normalized}` y obtiene respuesta nula
**Entonces** el Matcher no produce ningún mensaje a `vehicle.alerts` y avanza al siguiente evento sin latencia adicional

## CA-10: Fallback del Matcher ante indisponibilidad de Redis

**Dado** que Redis no está disponible (timeout o error de conexión)
**Cuando** el Matcher Service intenta ejecutar el cotejo para un evento
**Entonces** el Matcher usa el estado local de Kafka Streams como caché secundaria de la lista roja; si la placa está en el estado local, produce la alerta; si no está, no produce alerta; en ningún caso falla de forma abierta omitiendo el cotejo (fail closed)

## CA-11: Alerta persistida en PostgreSQL y enviada al WebSocket Gateway

**Dado** un evento de alerta en `vehicle.alerts`
**Cuando** el Alert Service lo procesa
**Entonces** se persiste un registro en la tabla de alertas de PostgreSQL con `country_code`, `plate_normalized`, `alert_timestamp`, `location` y estado `ACTIVE`; y se envía el payload de la alerta al WebSocket Gateway para distribución a los agentes de la zona

## CA-12: Deduplicación de alertas por placa y zona en ventana de 5 minutos

**Dado** que el Alert Service ya emitió una alerta para `plate_normalized = "ABC123X"` en la zona correspondiente a `country_code = "CO"` hace menos de 5 minutos
**Cuando** llega un segundo evento de alerta para la misma placa y misma zona
**Entonces** el Alert Service descarta la alerta duplicada sin producir un segundo push al WebSocket Gateway; el contador `alert_deduplication_suppressed_total` se incrementa

## CA-13: Entrega de alerta al agente policial en la zona correcta

**Dado** que un agente policial con sesión WebSocket activa tiene geocerca en la zona `zone_id = "Z-BOG-1"` de Colombia
**Cuando** el WebSocket Gateway recibe una alerta para `country_code = "CO"` con location dentro de `zone_id = "Z-BOG-1"`
**Entonces** el Gateway entrega el payload de la alerta a esa sesión WebSocket en tiempo real; los agentes de otras zonas o países no reciben la alerta

## CA-14: SLO hot path p95 < 2 s verificable por métricas

**Dado** que el pipeline completo (Deduplicator → Enrichment → Matcher → Alert → WebSocket) está operativo bajo carga normal
**Cuando** se mide el tiempo desde el `event_timestamp` del evento crudo hasta el timestamp de entrega de la alerta al WebSocket Gateway
**Entonces** el percentil 95 de esa latencia end-to-end es inferior a 2 000 ms; cada componente expone su latencia individual como métrica Prometheus

## CA-15: Creación de incidente y log de auditoría inmutable

**Dado** que una alerta fue procesada y el incident-service recibe una solicitud de creación de incidente
**Cuando** el incident-service persiste el incidente en PostgreSQL
**Entonces** se crea un registro de incidente con `state = CREATED` y una entrada en la tabla de auditoría con `event_type`, `state_from = null`, `state_to = CREATED`, `actor_id` y `timestamp`; la tabla de auditoría no permite updates ni deletes (solo inserts)

## CA-16: Propagación de vehículo recuperado a sincronizacion-paises

**Dado** que un incidente pasa al estado `RECOVERED` en el incident-service
**Cuando** el incident-service actualiza el estado del incidente
**Entonces** se produce un evento al tópico `stolen.vehicles.events` (o equivalente acordado) notificando el estado `RECOVERED` con `country_code`, `plate_normalized` e `incident_id`; el servicio `sincronizacion-paises` puede consumir este evento para actualizar el adaptador del país correspondiente

## CA-17: Cold path: Debezium publica cambios del WAL de PostgreSQL a Kafka

**Dado** que el conector Debezium está configurado con acceso al WAL de PostgreSQL y el tópico `pg.events.cdc` existe con replication factor 3
**Cuando** el Enrichment Service inserta una fila en la tabla de eventos de PostgreSQL
**Entonces** Debezium detecta el cambio en el WAL y publica un mensaje al tópico `pg.events.cdc` con el payload CDC (before/after) dentro de los 30 s siguientes a la inserción; ClickHouse ingiere el mensaje vía Kafka Engine Table

## CA-18: Aislamiento multi-tenant por country_code

**Dado** que el sistema tiene eventos de dispositivos de Colombia (`country_code = "CO"`) y Venezuela (`country_code = "VE"`) procesándose simultáneamente
**Cuando** un agente policial venezolano consulta alertas o eventos desde el WebSocket Gateway o el incident-service
**Entonces** solo recibe datos con `country_code = "VE"`; ningún dato colombiano es visible en el contexto venezolano; el filtrado por `country_code` ocurre en la capa de acceso a datos, no solo en la capa de presentación

---

## CR-01: Evento con event_id ausente o nulo rechazado en el Deduplicator

**Dado** un mensaje en `vehicle.events.raw` con el campo `event_id` ausente o con valor nulo
**Cuando** el Deduplicator intenta procesarlo
**Entonces** el mensaje se redirige a un dead-letter topic (`vehicle.events.raw.dlq`) con el motivo de rechazo anotado; no se produce a `vehicle.events.deduped` ni se eleva excepción no controlada

## CR-02: Fallo del servicio de geocodificación en el Enrichment Service

**Dado** que el servicio de geocodificación (API externa o dataset local) no está disponible o supera el timeout configurado
**Cuando** el Enrichment Service intenta enriquecer un evento con `lat`/`lon`
**Entonces** el evento se produce a `vehicle.events.enriched` con los campos `street`, `city`, `country` en blanco y `geocoding_status = "FAILED"`; el evento no se descarta; la métrica `enrichment_geocoding_failures_total` se incrementa; si el porcentaje de fallos supera el umbral configurado se activa una alerta operacional

## CR-03: Evento con coordenadas inválidas en el Enrichment Service

**Dado** un evento con `lat` o `lon` fuera del rango válido (p.ej. lat > 90 o lon > 180)
**Cuando** el Enrichment Service intenta la geocodificación
**Entonces** el evento avanza con `geocoding_status = "INVALID_COORDINATES"` y los campos de geocodificación en blanco; no se escribe una geometría PostGIS inválida en PostgreSQL

## CR-04: Redis disponible con entrada expirada — no genera alerta

**Dado** que Redis está disponible pero la clave `stolen:{country_code}:{plate_normalized}` expiró (vehículo retirado de la lista roja)
**Cuando** el Matcher ejecuta `GET stolen:CO:ABC123X`
**Entonces** Redis devuelve nulo y el Matcher no produce alerta, igual que en un miss normal

## CR-05: Alerta sin agentes activos en la zona

**Dado** que el Alert Service genera una alerta para una zona donde no hay agentes con sesión WebSocket activa
**Cuando** el WebSocket Gateway intenta entregar la alerta
**Entonces** la alerta se persiste en PostgreSQL con estado `ACTIVE` (un agente puede conectarse posteriormente y recuperarla); no se genera error ni excepción; la métrica `alert_delivery_no_active_agents_total` se incrementa

## CR-06: Reconexión de agente policial — recuperación de alertas pendientes

**Dado** que un agente policial perdió la conexión WebSocket y hay alertas `ACTIVE` no entregadas para su zona
**Cuando** el agente se reconecta al WebSocket Gateway
**Entonces** el Gateway entrega las alertas pendientes de la zona del agente desde PostgreSQL en el momento de la reconexión; el agente no pierde alertas generadas durante su desconexión dentro de la ventana de retención configurada

## CR-07: incident-service — transición de estado inválida rechazada

**Dado** que un incidente está en estado `RECOVERED`
**Cuando** el incident-service recibe una solicitud de transición hacia el estado `DISPATCHED` (retroceso inválido en el ciclo de vida)
**Entonces** el servicio rechaza la transición con error 409 Conflict, no modifica el estado del incidente ni escribe en el log de auditoría, y retorna el estado actual del incidente en la respuesta

## CR-08: Lag excesivo de Debezium — alerta operacional

**Dado** que el lag del conector Debezium (diferencia entre el LSN actual de PostgreSQL y el último LSN procesado por Debezium) supera el umbral configurado (por ejemplo, > 60 s)
**Cuando** el sistema de monitoreo evalúa la métrica `debezium_connector_lag_seconds`
**Entonces** se activa una alerta operacional al canal de monitoreo; el servicio de ClickHouse continúa ingiriendo los mensajes disponibles en `pg.events.cdc` sin pérdida de datos; el lag no afecta el hot path operacional

## CR-09: Partición Kafka con min.insync.replicas no satisfecha

**Dado** que el clúster Kafka pierde réplicas de forma que `min.insync.replicas=2` no puede satisfacerse para un tópico del pipeline
**Cuando** el Enrichment Service o el Matcher Service intentan producir mensajes
**Entonces** el productor recibe `NotEnoughReplicasException`; los mensajes se retienen en el buffer del productor con backoff exponencial; la alerta operacional de disponibilidad Kafka se activa; no se pierden mensajes durante una ventana de indisponibilidad inferior a la retención configurada del tópico

## CR-10: Evento con country_code ausente rechazado en todo el pipeline

**Dado** un evento en cualquier tópico del pipeline con el campo `country_code` ausente o vacío
**Cuando** cualquier componente del pipeline (Deduplicator, Enrichment, Matcher, Alert Service) intenta procesarlo
**Entonces** el mensaje se redirige al dead-letter topic correspondiente con el motivo `MISSING_COUNTRY_CODE`; ningún componente persiste datos sin discriminador de tenant

---

> **Nota de validación cruzada entre repositorios:** Este change establece contratos con `ingestion-mqtt` (upstream, aprobado), `almacenamiento-lectura` (downstream, pendiente), `api-frontend-analitica` (downstream, pendiente), `identidad-seguridad` (downstream, pendiente) y `sincronizacion-paises` (pendiente). Los esquemas de payload de `vehicle.events.raw`, `vehicle.alerts` y `pg.events.cdc`, así como el contrato del WebSocket Gateway y el esquema de tablas de PostgreSQL/OpenSearch/ClickHouse, deben acordarse formalmente antes de la implementación. Consultar `refacil-prereqs/BUS-CROSS-REPO.md`.
