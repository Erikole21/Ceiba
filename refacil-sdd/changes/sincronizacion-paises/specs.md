# specs: sincronizacion-paises

---

## CA-01: Publicación de evento canónico por adaptador CDC

**Dado** que un adaptador de país en modo CDC (Debezium conectado a la BD policial) detecta una nueva fila con `plate = "ABC123"` y `country_code = "CO"` en la BD policial
**Cuando** el adaptador mapea la fila al modelo canónico y publica en Kafka
**Entonces** aparece en el tópico `stolen.vehicles.events` un mensaje con clave `CO:ABC123`, los 9 campos mandatorios presentes y no nulos (`plate`, `stolen_date`, `stolen_location`, `owner_id`, `brand`, `class`, `line`, `color`, `model`), `status = STOLEN`, `schema_version` igual a la versión activa del Schema Registry y `country_code = "CO"`; la latencia desde el cambio en la BD policial hasta la publicación en Kafka es inferior a 1 s en condiciones normales

## CA-02: Publicación de evento canónico por adaptador JDBC polling

**Dado** que un adaptador en modo JDBC polling ejecuta su consulta periódica y detecta una fila nueva en la BD policial de Venezuela (`country_code = "VE"`)
**Cuando** el adaptador completa el ciclo de polling y mapea el resultado
**Entonces** el mensaje correspondiente aparece en `stolen.vehicles.events` dentro de la ventana de polling configurada (máximo 15 min), con la clave `VE:{plate}` y todos los campos del modelo canónico debidamente poblados; el adaptador registra la marca de tiempo del último registro procesado para evitar re-procesar duplicados en el siguiente ciclo

## CA-03: Publicación de evento canónico por adaptador SOAP/REST programado

**Dado** que un adaptador en modo SOAP/REST ejecuta su solicitud programada a la API policial de un país y recibe una lista de vehículos hurtados nuevos o actualizados
**Cuando** el adaptador itera la respuesta y mapea cada registro al modelo canónico
**Entonces** cada registro genera un mensaje independiente en `stolen.vehicles.events` con la clave `{country_code}:{plate}`; el adaptador registra la marca de tiempo de la última consulta exitosa para construir el delta en la próxima ejecución

## CA-04: Publicación de evento canónico por adaptador file drop watcher

**Dado** que el adaptador file drop watcher detecta un archivo CSV o XML nuevo en el directorio o SFTP configurado para un país
**Cuando** el adaptador parsea el archivo y mapea cada registro al modelo canónico
**Entonces** cada fila del archivo genera un mensaje en `stolen.vehicles.events`; el archivo procesado se marca como consumido (movido a directorio de archivado o registro en BD de control) para evitar re-ingesta en el siguiente ciclo de monitoreo

## CA-05: Validación de esquema Avro por el Canonical Vehicles Service

**Dado** que el Canonical Vehicles Service recibe un mensaje en `stolen.vehicles.events`
**Cuando** deserializa el payload usando el Schema Registry (compatibilidad backward)
**Entonces** si el esquema es válido, el mensaje avanza al procesamiento de upsert; si el esquema es inválido (campo mandatorio ausente, tipo incompatible o versión desconocida), el mensaje se redirige al dead-letter topic `stolen.vehicles.events.dlq` con el motivo de rechazo anotado, sin elevar excepción no controlada

## CA-06: Upsert en tabla canonical_vehicles de PostgreSQL

**Dado** que el Canonical Vehicles Service recibe un evento canónico válido con `plate = "ABC123"` y `country_code = "CO"` y `status = STOLEN`
**Cuando** ejecuta el upsert en la tabla `canonical_vehicles`
**Entonces** si no existe fila con esa combinación `(country_code, plate)`, se inserta una fila nueva; si ya existe, se actualiza con los campos del evento; en ambos casos el campo `updated_at` refleja el timestamp del evento; la operación es idempotente (ejecutar el mismo evento dos veces produce el mismo estado final)

## CA-07: Actualización de la lista roja en Redis

**Dado** que el Canonical Vehicles Service completó el upsert de un vehículo con `status = STOLEN`
**Cuando** actualiza la lista roja en Redis
**Entonces** la clave `stolen:{country_code}:{plate}` existe en Redis con el TTL configurado (valor de referencia: 30 días, ajustable por configuración); el Matcher Service (change `backbone-procesamiento`) puede ejecutar `GET stolen:CO:ABC123` y obtener respuesta no nula para esa placa

## CA-08: Propagación del estado RECOVERED desde el incident-service

**Dado** que el incident-service (`backbone-procesamiento`) publica un evento con `status = RECOVERED`, `plate = "ABC123"` y `country_code = "CO"` en el tópico `stolen.vehicles.events`
**Cuando** el Canonical Vehicles Service lo procesa
**Entonces** la fila correspondiente en `canonical_vehicles` se actualiza con `status = RECOVERED`; la clave `stolen:CO:ABC123` se elimina de Redis (o expira inmediatamente); el Edge Distribution Service recibe notificación para regenerar el Bloom filter del país y publicar el delta

## CA-09: Publicación de delta de Bloom filter en MQTT retained

**Dado** que el Edge Distribution Service detecta un cambio en la lista canónica de placas hurtadas activas para `country_code = "CO"` (alta o baja de placas)
**Cuando** genera la versión nueva del Bloom filter y calcula el delta respecto a la versión anterior
**Entonces** publica en el tópico MQTT `countries/CO/bloom-filter` (retained, QoS 1) un payload que incluye: versión nueva del BF, delta binario comprimido, checksum y timestamp; el agente de borde que ya posee la versión anterior puede reconstruir el BF actualizado aplicando el delta sin requerir descarga completa

## CA-10: Descarga completa del Bloom filter ante versión desactualizada del dispositivo

**Dado** que un dispositivo de borde se reconecta tras un período de desconexión superior a 7 días y su versión local de BF es anterior al umbral configurable
**Cuando** el Edge Distribution Service detecta la solicitud de sincronización con una versión demasiado antigua (o el dispositivo lo indica en su suscripción)
**Entonces** el Edge Distribution Service publica en el tópico MQTT correspondiente un payload de descarga completa del BF (full snapshot) en lugar del delta; el dispositivo reemplaza su BF local por la versión completa recibida

## CA-11: Parámetros del Bloom filter satisfacen las restricciones del caso de estudio

**Dado** que el Edge Distribution Service construye el Bloom filter para un país con hasta 1 000 000 de placas hurtadas activas
**Cuando** serializa el BF para distribución
**Entonces** el tamaño serializado del BF completo es inferior a 1 MB y la tasa de falsos positivos medida es inferior al 1 %

## CA-12: Aislamiento multi-tenant — datos de un país no visibles en otro

**Dado** que el sistema sincroniza simultáneamente datos de Colombia (`country_code = "CO"`) y Venezuela (`country_code = "VE"`)
**Cuando** el Canonical Vehicles Service o el Edge Distribution Service procesan un evento para `CO`
**Entonces** los registros, las claves Redis y los tópicos MQTT de `VE` no son modificados ni leídos; la partición Kafka `stolen.vehicles.events` usa `country_code` como parte de la clave; la tabla `canonical_vehicles` incluye `country_code` como discriminador de tenant en cada consulta y en el índice primario

## CA-13: Incorporación de un nuevo país mediante el onboarding guide

**Dado** que un operador sigue el proceso documentado en el onboarding guide para incorporar un nuevo país con modo de integración JDBC polling
**Cuando** completa los cuatro pasos (implementar el conector, definir el mapeo de campos, validar con el contraparte policial, desplegar el contenedor del adaptador)
**Entonces** los primeros eventos canónicos del nuevo país aparecen en `stolen.vehicles.events` dentro del período de polling configurado, sin modificar ningún componente del núcleo (Canonical Vehicles Service, Edge Distribution Service, Schema Registry)

## CA-14: Extensiones por país en el campo extensions (JSONB)

**Dado** que la BD policial de un país provee un campo adicional no contemplado en el modelo canónico de 9 campos (p.ej. `numero_motor`)
**Cuando** el adaptador del país mapea el registro y no encuentra campo canónico correspondiente
**Entonces** el campo adicional se almacena en `extensions` como entrada JSONB (`{"numero_motor": "XYZ123"}`); el evento se serializa correctamente contra el esquema Avro sin requerir una versión mayor del esquema; el Canonical Vehicles Service persiste `extensions` en la columna JSONB de `canonical_vehicles`

## CA-15: Disponibilidad del Canonical Vehicles Service ante lag de Kafka

**Dado** que el lag de consumo del tópico `stolen.vehicles.events` supera el umbral de alerta configurado (valor de referencia: > 10 000 mensajes no procesados)
**Cuando** el sistema de monitoreo evalúa la métrica `canonical_vehicles_consumer.lag`
**Entonces** se activa una alerta operacional; el servicio continúa procesando mensajes en orden sin pérdida de datos; el SLA de frescura de la lista roja en Redis se ve afectado proporcionalmente al lag acumulado y este impacto queda documentado en las métricas

---

## CR-01: Mensaje en stolen.vehicles.events con campo mandatorio ausente

**Dado** un mensaje en `stolen.vehicles.events` con el campo `plate` ausente (o cualquier otro de los 9 campos mandatorios del modelo canónico)
**Cuando** el Canonical Vehicles Service intenta deserializarlo y validarlo
**Entonces** el mensaje se redirige al dead-letter topic `stolen.vehicles.events.dlq` con el código de error `MISSING_MANDATORY_FIELD` y el nombre del campo faltante; no se inserta nada en `canonical_vehicles` ni se modifica Redis; no se eleva excepción no controlada

## CR-02: Mensaje con country_code ausente o no registrado

**Dado** un mensaje en `stolen.vehicles.events` sin el campo `country_code` o con un valor no registrado en la configuración del sistema
**Cuando** el Canonical Vehicles Service lo procesa
**Entonces** el mensaje se redirige a `stolen.vehicles.events.dlq` con el código de error `INVALID_COUNTRY_CODE`; no se modifica ningún dato del sistema; se incrementa la métrica `canonical_vehicles.rejected.invalid_country`

## CR-03: Fallo de conectividad del adaptador con la BD policial

**Dado** que un adaptador de país (cualquier modo) pierde la conectividad con la BD policial
**Cuando** el adaptador intenta ejecutar la siguiente consulta o escuchar el stream CDC
**Entonces** el adaptador registra el error, activa el mecanismo de reintento con backoff exponencial configurado y expone la métrica `adapter.connectivity.failures`; la última marca de tiempo de sincronización exitosa queda disponible para calcular el retraso acumulado; no se publican mensajes parciales ni corruptos en Kafka

## CR-04: Incompatibilidad de esquema Avro en el Schema Registry

**Dado** que un adaptador intenta publicar un mensaje con una versión de esquema incompatible con las reglas de compatibilidad backward del Schema Registry
**Cuando** el producer Avro del adaptador valida el esquema antes de publicar
**Entonces** el Schema Registry rechaza el registro del nuevo esquema; el adaptador registra el error y detiene la publicación hasta que el operador resuelva la incompatibilidad; no se publican mensajes con esquema no registrado

## CR-05: Dispositivo de borde con BF muy desactualizado solicita delta inexistente

**Dado** que un dispositivo de borde tiene una versión del Bloom filter con antigüedad superior a 7 días y solicita sincronización por delta
**Cuando** el Edge Distribution Service evalúa la solicitud
**Entonces** el servicio detecta que la versión del dispositivo es demasiado antigua para calcular un delta válido y publica un payload de descarga completa (full snapshot) en el tópico MQTT correspondiente; el dispositivo actualiza su BF local desde el snapshot sin aplicar deltas intermedios

## CR-06: Evento RECOVERED con placa inexistente en canonical_vehicles

**Dado** que el incident-service produce un evento `RECOVERED` para `plate = "XYZ999"` y `country_code = "CO"`, pero esa placa no existe en la tabla `canonical_vehicles`
**Cuando** el Canonical Vehicles Service procesa el evento
**Entonces** el servicio registra una advertencia operacional (`canonical_vehicles.recovered.not_found`), no lanza excepción, no inserta ninguna fila nueva con `status = RECOVERED` y descarta el evento sin modificar Redis ni el BF; el evento se registra en el log de auditoría con estado `ORPHAN_RECOVERED`

## CR-07: Fallo de Redis durante la actualización de la lista roja

**Dado** que Redis no está disponible cuando el Canonical Vehicles Service intenta actualizar la clave `stolen:{country_code}:{plate}`
**Cuando** el servicio ejecuta el SET o DEL en Redis
**Entonces** el servicio registra el fallo, incrementa la métrica `canonical_vehicles.redis.failures`, reintenta con backoff exponencial hasta el número máximo de intentos configurado; el upsert en PostgreSQL ya fue confirmado, por lo que el estado persistente es consistente; al recuperarse Redis, el servicio puede re-sincronizar la lista roja desde `canonical_vehicles` mediante un proceso de reconciliación

## CR-08: Soberanía de datos — datos crudos de la BD policial no salen del territorio

**Dado** que el adaptador de un país opera en modo on-premises dentro del territorio nacional
**Cuando** el adaptador mapea un registro de la BD policial al modelo canónico y produce el mensaje Kafka
**Entonces** el payload del mensaje contiene exclusivamente los campos del modelo canónico; ningún campo crudo no mapeado de la BD policial se incluye en el payload de Kafka (los datos adicionales van a `extensions` si y solo si el operador del país los autoriza explícitamente en la configuración del adaptador)

---

> **Nota de validación cruzada entre repositorios:** Este change establece o consume contratos con `backbone-procesamiento` (aprobado, produce eventos `RECOVERED`), `ingestion-mqtt` (aprobado, el Edge Distribution Service publica en MQTT hacia los agentes de borde), `agente-borde` (aprobado, consume el delta del BF vía MQTT retained), `identidad-seguridad` (pendiente, credenciales de adaptadores vía Vault) y `almacenamiento-lectura` (pendiente, esquema de `canonical_vehicles` para analítica). Los contratos de payload deben acordarse formalmente antes de la implementación. Consultar `refacil-prereqs/BUS-CROSS-REPO.md`.
