# specs: almacenamiento-lectura

---

## CA-01: Inserción idempotente de evento en vehicle_events (PostgreSQL)

**Dado** que la tabla `vehicle_events` ya contiene un registro con `event_id = "E-001"` y `country_code = "CO"`
**Cuando** el Enrichment Service proyecta el mismo evento (reintento at-least-once de Kafka) con el mismo `event_id`
**Entonces** la base de datos rechaza la inserción duplicada sin error de aplicación (clave primaria o constraint UNIQUE sobre `event_id`); el registro original no se modifica; la proyección registra en su métrica `projection.duplicate_skipped` y avanza al siguiente evento

## CA-02: Particionamiento por country_code en vehicle_events

**Dado** que la tabla `vehicle_events` está particionada por `country_code` y existen particiones para `"CO"` y `"MX"`
**Cuando** el Enrichment Service inserta un evento con `country_code = "CO"`
**Entonces** el registro se almacena en la partición `vehicle_events_co`; una consulta con `WHERE country_code = 'MX'` no escanea la partición colombiana (partition pruning verificable por EXPLAIN)

## CA-03: Búsqueda espacial PostGIS con índice en vehicle_events

**Dado** que el índice espacial (GiST o SP-GiST) sobre la columna `location geometry(Point, 4326)` está activo en `vehicle_events`
**Cuando** el Alert Service ejecuta una consulta geoespacial con `ST_DWithin(location, ST_MakePoint(lon, lat)::geography, radio_metros)`
**Entonces** el query planner utiliza el índice espacial (no seq-scan); el tiempo de respuesta para un radio de 10 km sobre un dataset de 1 M de registros es inferior a 200 ms

## CA-04: Búsqueda exacta de placa en vehicle_events (PostgreSQL)

**Dado** que existe un índice B-tree sobre `plate_normalized` en la partición `vehicle_events_co`
**Cuando** se ejecuta `SELECT * FROM vehicle_events WHERE plate_normalized = 'ABC123X' AND country_code = 'CO' ORDER BY timestamp_capture DESC`
**Entonces** el query planner usa el índice; se retornan todos los eventos de esa placa en orden cronológico descendente; el tiempo de respuesta p95 es inferior a 50 ms para hasta 500 eventos por placa

## CA-05: Registro de vehículo en stolen_vehicles con schema_version

**Dado** que el Canonical Vehicles Service recibe una actualización de vehículo hurtado con datos extendidos en el campo `extensions` JSONB
**Cuando** se inserta o actualiza el registro en `stolen_vehicles`
**Entonces** el campo `schema_version` se incrementa; el campo `extensions` almacena el JSON adicional sin alterar las columnas canónicas (`plate`, `country_code`, `theft_date`, `brand`, `class`, `line`, `color`, `model`, `owner_dni_hash`, `status`); el registro anterior queda trazable en `audit_log`

## CA-06: Escritura de entrada en audit_log (append-only, 7 años)

**Dado** que se realiza cualquier cambio de estado en `incidents` o `stolen_vehicles`
**Cuando** el servicio correspondiente registra el evento en `audit_log`
**Entonces** se inserta una fila con `entity_type`, `entity_id`, `country_code`, `event_type`, `actor_id`, `timestamp`, y `payload` JSONB; ninguna instrucción UPDATE o DELETE es posible sobre `audit_log` (regla de base de datos o política RLS); los registros con `timestamp` superior a 7 años pueden archivarse pero no eliminarse antes de ese plazo

## CA-07: Conector Debezium CDC publica cambios de vehicle_events al tópico pg.events.cdc

**Dado** que el conector Debezium está configurado con acceso al slot de replicación de PostgreSQL y el tópico `pg.events.cdc` existe con replication factor 3
**Cuando** el Enrichment Service inserta una fila en `vehicle_events`
**Entonces** Debezium detecta el cambio en el WAL y publica un mensaje en `pg.events.cdc` con el payload CDC (before/after + metadata de LSN) dentro de los 30 s siguientes a la inserción; ClickHouse ingiere el mensaje vía Kafka Engine Table sin pérdida de datos

## CA-08: Búsqueda fuzzy de placa en OpenSearch con SLO < 300 ms

**Dado** que existe un documento indexado en `vehicle-events-co-2026-05` con `plate_normalized = "ABC123X"`
**Cuando** la API de búsqueda ejecuta una query con fuzziness AUTO sobre el término `"ABC123"` (un carácter de diferencia)
**Entonces** el documento es devuelto en el resultado con score de relevancia; la latencia de la búsqueda p95 es inferior a 300 ms; la búsqueda exacta por `plate_normalized.keyword` retorna el resultado en menos de 50 ms

## CA-09: Indexación de evento en índice rotatorio OpenSearch correcto

**Dado** que el Enrichment Service produce un evento con `country_code = "CO"` y `timestamp_capture = "2026-05-12T14:00:00Z"`
**Cuando** el evento se proyecta a OpenSearch
**Entonces** el documento se indexa en `vehicle-events-co-2026-05`; el alias `vehicle-events-co` apunta a ese índice y permite búsquedas transparentes; no se crea un documento en un índice de otro país ni de otro mes

## CA-10: Política ILM de OpenSearch — transición de hot a warm a cold

**Dado** que un índice `vehicle-events-co-2023-04` tiene una antigüedad superior a 3 meses
**Cuando** la política ILM evalúa el índice en su ciclo programado
**Entonces** el índice transiciona a fase warm (réplicas reducidas, merge forzado); a los 12 meses transiciona a fase cold (congelado); a los 24 meses el índice se elimina automáticamente; en ningún punto se eliminan datos antes de cumplir el plazo de su fase

## CA-11: Cotejo Redis sub-milisegundo para lista roja (hot path Matcher)

**Dado** que la clave `stolen:CO:ABC123X` existe en Redis con TTL activo (≤ 24 h)
**Cuando** el Matcher Service ejecuta `GET stolen:CO:ABC123X`
**Entonces** Redis retorna los metadatos del vehículo en menos de 1 ms (percentil 99 bajo carga normal); el Matcher produce un evento de alerta al tópico `vehicle.alerts`

## CA-12: TTL de entrada Redis expira tras 24 h y se renueva en cada CDC

**Dado** que la clave `stolen:CO:ABC123X` fue creada hace exactamente 24 h sin actividad
**Cuando** el TTL expira
**Entonces** Redis elimina la clave automáticamente; el Matcher obtiene nulo en el siguiente cotejo y no genera falsa alerta; si en ese momento el vehículo sigue en la lista robada, el Canonical Vehicles Service habrá renovado la clave antes de la expiración mediante el evento CDC correspondiente

## CA-13: Invalidación de Redis al retirar vehículo de la lista robada

**Dado** que el vehículo con placa `"ABC123X"` (Colombia) es retirado de la lista robada en la BD policial
**Cuando** el Canonical Vehicles Service recibe el evento CDC de `sincronizacion-paises`
**Entonces** el servicio ejecuta `DEL stolen:CO:ABC123X` en Redis; una consulta posterior del Matcher sobre esa clave retorna nulo; la entrada ya no genera alertas

## CA-14: Ingestión ClickHouse de evento desde pg.events.cdc

**Dado** que un mensaje de Debezium CDC está disponible en el tópico `pg.events.cdc` con `country_code = "CO"` y `timestamp_capture` dentro del mes actual
**Cuando** la Kafka Engine Table de ClickHouse consume el mensaje y la vista materializada procesa el dato
**Entonces** el registro aparece en `vehicle_events_daily` particionado en la partición `(toYYYYMM(timestamp_capture), country_code)`; la latencia de ingestión hasta disponibilidad de lectura en ClickHouse es inferior a 60 s

## CA-15: Consulta de hexbinning H3 (cold path) con SLO < 3 s

**Dado** que la vista materializada H3 de resolución 8 está pre-calculada para Colombia con datos de los últimos 7 días
**Cuando** `api-frontend-analitica` ejecuta una consulta de mapa de calor para un país + ventana de 7 días
**Entonces** ClickHouse retorna la agregación de eventos por celda H3 con el conteo correspondiente; el tiempo de respuesta p95 es inferior a 3 s; la consulta usa la vista materializada (no la tabla raw)

## CA-16: Particionamiento ClickHouse por country_code + mes

**Dado** que existen datos de Colombia (country_code = "CO") y México (country_code = "MX") en `vehicle_events_daily`
**Cuando** se ejecuta una consulta analítica filtrada por `country_code = 'CO'`
**Entonces** ClickHouse aplica partition pruning y no lee las partes de México; el query log confirma que solo se leen las partes de la partición colombiana

## CA-17: Aislamiento multi-tenant en todos los almacenes

**Dado** que hay eventos de Colombia (`country_code = "CO"`) y Venezuela (`country_code = "VE"`) almacenados simultáneamente en PostgreSQL, OpenSearch, ClickHouse y Redis
**Cuando** se ejecutan consultas con contexto de tenant `"VE"` (claim JWT o parámetro de API)
**Entonces** ninguna consulta retorna datos con `country_code = "CO"`; el aislamiento se aplica en la capa de acceso a datos mediante partition pruning, alias de índice OpenSearch, prefijo de clave Redis y cláusula WHERE en ClickHouse; no solo en la capa de presentación

## CA-18: Alta disponibilidad PostgreSQL con Patroni (failover automático)

**Dado** que la topología HA de PostgreSQL está configurada con Patroni (primario + al menos una réplica) o equivalente gestionado (RDS Multi-AZ)
**Cuando** el nodo primario falla
**Entonces** Patroni promueve la réplica más actualizada a primario en menos de 30 s; la aplicación reconecta sin intervención manual; el tiempo de indisponibilidad observado está dentro del SLA de 99,9 % mensual

## CA-19: Derecho al olvido — limpieza coordinada al retirar vehículo

**Dado** que un vehículo es retirado definitivamente de la lista robada y se activa el flujo de right-to-be-forgotten
**Cuando** la automatización de ciclo de vida ejecuta la limpieza en todos los almacenes
**Entonces** el documento correspondiente es eliminado de los índices OpenSearch activos; el registro en PostgreSQL `vehicle_events` es marcado con `data_lifecycle_status = 'forgotten'` (no eliminado, por retención de auditoría); la clave Redis es eliminada inmediatamente; el registro en `stolen_vehicles` pasa a `status = 'removed'`; ClickHouse marca el registro en la partición correspondiente para no proyectarlo en vistas nuevas

---

## CR-01: Evento sin event_id rechazado por constraint en PostgreSQL

**Dado** un mensaje del pipeline que llega sin el campo `event_id` (nulo o vacío)
**Cuando** el Enrichment Service intenta insertar la fila en `vehicle_events`
**Entonces** PostgreSQL rechaza la inserción con violación de NOT NULL o CHECK constraint; el servicio redirige el mensaje al dead-letter topic correspondiente con el motivo `MISSING_EVENT_ID`; no se persiste ningún registro huérfano

## CR-02: Evento sin country_code rechazado en todos los almacenes

**Dado** un evento que llega al pipeline de proyección sin el campo `country_code` (nulo o vacío)
**Cuando** cualquier servicio de proyección intenta almacenarlo (PostgreSQL, OpenSearch, ClickHouse o Redis)
**Entonces** la proyección rechaza la escritura con el motivo `MISSING_COUNTRY_CODE`; el mensaje se redirige al dead-letter topic; ningún almacén persiste datos sin discriminador de tenant (ADR-011)

## CR-03: Coordenadas fuera de rango no generan geometría PostGIS inválida

**Dado** un evento con `lat = 95.0` (fuera del rango válido -90 a 90) o `lon = 200.0`
**Cuando** el Enrichment Service intenta construir la geometría PostGIS y proyectarla
**Entonces** la inserción en `vehicle_events` se realiza con `location = NULL` y `geocoding_status = 'INVALID_COORDINATES'`; no se produce una excepción de geometría inválida en PostgreSQL; la métrica `projection.invalid_coordinates` se incrementa

## CR-04: Índice OpenSearch inexistente no bloquea la ingestión

**Dado** que la política ILM ha eliminado el índice `vehicle-events-co-2024-01` (datos más antiguos de 24 meses)
**Cuando** una búsqueda histórica intenta consultar ese índice directamente
**Entonces** OpenSearch retorna un error 404 controlado; la API de búsqueda responde con un mensaje semántico indicando que los datos superan el período de retención; no se produce un fallo silencioso ni se retornan datos parciales sin aviso

## CR-05: Fallo de ClickHouse durante ingestión no pierde mensajes Kafka

**Dado** que ClickHouse no está disponible temporalmente (restart, mantenimiento)
**Cuando** llegan mensajes al tópico `pg.events.cdc` durante la indisponibilidad
**Entonces** los mensajes permanecen disponibles en Kafka durante la ventana de retención configurada; al recuperarse ClickHouse, la Kafka Engine Table reanuda el consumo desde el último offset confirmado sin pérdida de datos; el lag de ingestión se monitorea y dispara alerta si supera el umbral configurado (p.ej. > 300 s)

## CR-06: TTL Redis expirado para vehículo aún en lista robada — renovación garantizada

**Dado** que la clave `stolen:CO:ABC123X` está próxima a expirar (TTL < 10 % del valor inicial de 24 h)
**Cuando** el Canonical Vehicles Service recibe cualquier evento CDC relacionado con ese vehículo
**Entonces** el servicio ejecuta `SET stolen:CO:ABC123X <metadata> EX 86400` (renovando el TTL completo); la clave no expira mientras el vehículo permanezca en la lista robada; si no llega ningún evento CDC en 24 h, la expiración es el comportamiento seguro esperado (el siguiente cotejo del Matcher obtiene nulo)

## CR-07: Fallo de Redis no impide la proyección a PostgreSQL u OpenSearch

**Dado** que Redis no está disponible (timeout de conexión o error de red)
**Cuando** el servicio de proyección intenta actualizar una clave de la lista roja
**Entonces** la proyección registra el fallo en la métrica `redis.write.failures` y registra el evento en el dead-letter de Redis para reintento; la proyección a PostgreSQL y OpenSearch continúa sin interrupción; no se genera una excepción no controlada que detenga el pipeline

## CR-08: Partición PostgreSQL sin vacío genera bloat — alerta operacional

**Dado** que una partición de `vehicle_events` acumula filas de versiones antiguas (dead tuples) superior al umbral configurado de bloat (p.ej. > 20 %)
**Cuando** el sistema de monitoreo evalúa la métrica `pg_stat_user_tables.n_dead_tup`
**Entonces** se activa una alerta operacional al canal de monitoreo recomendando la ejecución de VACUUM ANALYZE; el runbook correspondiente documenta el procedimiento; la alerta no interrumpe el servicio pero previene degradación del rendimiento de consultas

## CR-09: Topología Redis Sentinel — failover sin pérdida de escrituras pendientes

**Dado** que la topología Redis Sentinel está activa con un primario y dos réplicas, y el primario falla
**Cuando** Sentinel elige una nueva réplica primaria
**Entonces** el failover se completa en menos de 10 s; las escrituras pendientes en el buffer del cliente se reintentan contra el nuevo primario; las claves `stolen:{country_code}:{plate_normalized}` de los últimos 30 s son eventualmente consistentes en el nuevo primario; ningún cotejo del Matcher obtiene un falso negativo durante el failover si la clave estaba presente antes del fallo

## CR-10: Consulta analítica ClickHouse sin pre-agregado — tiempo fuera de SLO

**Dado** que la vista materializada H3 no ha sido actualizada (primera ingestión después de un reinicio) y `api-frontend-analitica` solicita un mapa de calor de 30 días sobre la tabla raw
**Cuando** ClickHouse ejecuta la consulta sin usar la vista materializada
**Entonces** la consulta retorna resultados correctos pero la API registra en su métrica que el tiempo de respuesta superó el SLO de 3 s; el sistema expone este degradado como evento de observabilidad; la API puede opcionalmente devolver datos parciales con cabecera de advertencia `X-SLO-Degraded: true`

---

> **Nota de validación cruzada entre repositorios:** Este change define los contratos de escritura y lectura de los cuatro almacenes de datos del sistema. Los contratos establecidos aquí deben acordarse formalmente con los changes relacionados antes de la implementación. Consultar `refacil-prereqs/BUS-CROSS-REPO.md`.
>
> - **`backbone-procesamiento` (upstream, aprobado):** los schemas de `vehicle_events`, `stolen_vehicles`, `incidents` y `audit_log` en PostgreSQL; el mapping OpenSearch; la estructura de tabla `vehicle_events_daily` en ClickHouse; y la estructura de clave Redis `stolen:{country_code}:{plate_normalized}` son el contrato de escritura del pipeline de procesamiento. Cualquier cambio en estos schemas requiere notificación y re-validación con ese change.
> - **`sincronizacion-paises` (upstream, aprobado):** el schema canónico de `stolen_vehicles` y la estructura de clave Redis son el destino de escritura del Canonical Vehicles Service. Los campos opcionales en `extensions` JSONB permiten evolucionar el modelo sin romper el contrato base.
> - **`api-frontend-analitica` (downstream, pendiente):** los contratos de lectura de los cuatro almacenes (queries de ejemplo, índices disponibles, aliases OpenSearch, vistas ClickHouse, estructura de respuesta Redis) documentados en `design.md` de este change son la especificación de interfaz que ese change debe implementar.
> - **`identidad-seguridad` (downstream, pendiente):** el claim `country_code` del JWT emitido por Keycloak es el discriminador de tenant usado en todas las consultas a los cuatro almacenes; el mecanismo de propagación del claim hasta la capa de datos debe acordarse con ese change.
