# specs: agente-borde

## CA-01: captura de placa vía ANPR Adapter SPI

**Dado** un dispositivo en operación con el componente ANPR local activo y el agente Go iniciado con la implementación de adapter configurada (REST local, socket UNIX, watch file o named pipe)
**Cuando** el ANPR detecta una matrícula con confidence >= umbral configurado
**Entonces** el Collector recibe la placa, le adjunta coordenadas GPS actuales, timestamp wall-clock UTC y timestamp monotónico del proceso, y produce un evento normalizado con `event_id` idempotente (UUID v5 sobre device_id + placa + timestamp_monotónico)

## CA-02: persistencia antes de envío (store-and-forward)

**Dado** un evento normalizado generado por el Collector
**Cuando** el Queue Manager recibe el evento
**Entonces** el evento se escribe en la base SQLite WAL (`synchronous=NORMAL`) y se confirma la escritura (fsync implícito del WAL) antes de que el Uploader intente cualquier transmisión; si el proceso termina abruptamente después de la escritura, el evento permanece en la cola al reiniciar

## CA-03: modo de captura configurable (`upload_mode`)

**Dado** el agente con el parámetro `upload_mode` configurado (recibido vía Config/OTA Manager; valor por defecto: `stolen_only`)
**Cuando** el Collector produce un evento normalizado y el Queue Manager consulta el Bloom filter
**Entonces**:
- Si `upload_mode: stolen_only` (por defecto): solo los eventos con **hit en el Bloom filter** se encolan para transmisión al cloud; los eventos sin hit se descartan localmente. El hit determina tanto la selección como la prioridad `HIGH`.
- Si `upload_mode: all`: **todos los eventos** se encolan para transmisión; el Bloom filter solo determina la prioridad (`HIGH` para hit, `NORMAL` para miss). No se descarta ningún evento.
- El cambio de `upload_mode` se aplica a los eventos nuevos inmediatamente al recibir la configuración; los eventos ya encolados no se reenvían ni eliminan retroactivamente.

## CA-04: transmisión de metadatos por MQTT QoS 1

**Dado** el agente con conectividad GSM disponible y sesión MQTT persistente activa con el broker EMQX (mTLS con certificado de dispositivo)
**Cuando** el Uploader procesa un evento de la cola
**Entonces** publica el payload JSON del evento en el topic `devices/<device_id>/events` con QoS 1; aguarda el PUBACK del broker; si no lo recibe en el timeout configurado, reintenta con backoff exponencial; el evento permanece en la cola SQLite hasta recibir confirmación de entrega

## CA-05: subida de imagen por URL pre-firmada

**Dado** una imagen capturada por el dispositivo y almacenada en el Image Store local
**Cuando** el Uploader tiene conectividad y el evento asociado está pendiente de subida de imagen
**Entonces** solicita una URL pre-firmada al Upload Service cloud, realiza `HTTP PUT` directo al object storage S3-API con la imagen, y actualiza el evento en SQLite con el campo `image_uri` antes de publicar el evento MQTT definitivo; la imagen local se conserva hasta que la política de rotación la elimine

## CA-06: rotación del Image Store local

**Dado** el Image Store local con imágenes acumuladas
**Cuando** el total de imágenes supera 10 GB o la imagen más antigua supera 7 días de antigüedad (lo que ocurra primero)
**Entonces** el Image Store elimina las imágenes más antiguas en lotes hasta quedar dentro de los límites de política; las imágenes ya subidas con éxito se eliminan primero; el proceso de eliminación no bloquea la captura de nuevas imágenes

## CA-07: actualización incremental del Bloom filter

**Dado** el agente suscrito al topic MQTT retained `countries/<country_code>/bloom-filter/delta`
**Cuando** el cloud publica un delta del Bloom filter del país
**Entonces** el agente aplica el delta en memoria sin descargar el filtro completo, persiste el filtro actualizado en disco (archivo binario), y confirma la actualización vía log estructurado; la sincronización inicial completa desde cero debe completarse en menos de 30 segundos sobre GSM 3G

## CA-08: publicación de Health Beacon

**Dado** el agente en operación normal
**Cuando** transcurre el intervalo de heartbeat configurado (por defecto 60 segundos)
**Entonces** el Health Beacon publica en el topic `devices/<device_id>/health` un payload con: porcentaje de CPU, RSS en MB, uso de disco en GB, nivel de batería en %, nivel de señal GSM (RSSI), drift NTP en ms, y cantidad de eventos pendientes en cola; el payload usa compresión (msgpack o CBOR) para reducir uso de banda GSM

## CA-09: aplicación de actualización OTA

**Dado** el Config/OTA Manager suscrito al topic MQTT retained `devices/<device_id>/ota` o `fleet/<country_code>/ota`
**Cuando** se publica un manifiesto OTA con URL de descarga del nuevo binario y firma cosign
**Entonces** el agente descarga el binario en una ruta temporal, verifica la firma cosign con la clave pública embebida en el agente actual, reemplaza el binario activo solo si la verificación es exitosa, y solicita reinicio vía systemd; si la firma no es válida, descarta el archivo temporal, reporta el error en `devices/<device_id>/health` y continúa operando con el binario actual

## CA-10: watchdog y auto-recuperación

**Dado** el agente corriendo como servicio systemd con `WatchdogSec` configurado
**Cuando** el proceso principal deja de emitir el keepalive de watchdog (por hang, deadlock o consumo de memoria > umbral) durante el intervalo configurado
**Entonces** systemd reinicia el proceso; al reiniciar, el agente retoma la cola SQLite desde el último evento no confirmado y recupera el Bloom filter desde disco sin pérdida de datos

## CA-11: operación sin conectividad (modo offline)

**Dado** un dispositivo sin conectividad GSM
**Cuando** el Uploader detecta que el broker MQTT no es alcanzable
**Entonces** el agente continúa capturando placas, normalizando eventos y persistiéndolos en la cola SQLite; la cola acumula hasta agotar la capacidad de disco según la política de rotación; cuando la conectividad se restablece, el Uploader retransmite en orden de prioridad (HIGH primero, luego NORMAL en FIFO) sin duplicar eventos ya confirmados

## CA-12: recuperación tras pérdida de energía abrupta

**Dado** un dispositivo que sufre un corte de energía abrupto mientras hay eventos en cola
**Cuando** el dispositivo recupera energía y systemd inicia el agente
**Entonces** el agente lee la cola SQLite WAL sin corrupción, retoma el procesamiento desde el primer evento no confirmado, y continúa la operación normal; el tiempo máximo de recuperación desde el arranque del sistema hasta el primer evento transmitido no excede 30 segundos en hardware de referencia

## CA-13: footprint de recursos

**Dado** el agente en operación sostenida (más de 1 hora) bajo carga normal (una captura cada 5 segundos en promedio)
**Cuando** se mide el consumo de recursos del proceso
**Entonces** el uso de CPU no supera el 5% sostenido en los 2 cores del dispositivo, el RSS del proceso no supera 50 MB, y el binario estático ocupa aproximadamente 30 MB en disco

---

## CA-14: cambio de `upload_mode` sin redeploy

**Dado** el agente en operación con `upload_mode: stolen_only`
**Cuando** el Config/OTA Manager recibe una actualización de configuración remota con `upload_mode: all` (o viceversa) vía el topic MQTT retained `devices/<device_id>/config` o `fleet/<country_code>/config`
**Entonces** el agente aplica el nuevo valor en el siguiente ciclo de captura sin reiniciarse; registra el cambio en el log estructurado con el valor anterior, el nuevo valor y el timestamp de aplicación; el cambio de `stolen_only` a `all` requiere que la configuración incluya un campo `authorized_by` no vacío (nombre o ID del administrador que autorizó el modo de captura masiva)

---

## CR-01: adapter SPI no disponible

**Dado** el agente iniciado con una implementación de adapter configurada
**Cuando** el componente ANPR local no es alcanzable (socket cerrado, archivo no disponible, endpoint REST no responde)
**Entonces** el agente registra el error en el log estructurado, reporta el estado `anpr_unavailable` en el siguiente Health Beacon, y reintenta la conexión con backoff exponencial sin terminar el proceso; no se generan eventos con datos de placa inválidos o vacíos

## CR-02: cola SQLite próxima al límite de disco

**Dado** el agente en modo offline con la cola SQLite acumulando eventos
**Cuando** el espacio disponible en disco cae por debajo del umbral de seguridad configurado (por defecto 500 MB libres)
**Entonces** el agente deja de aceptar nuevas imágenes en el Image Store, reporta `disk_low` en el Health Beacon, y elimina primero las imágenes locales más antiguas ya subidas para liberar espacio; nunca descarta eventos de metadatos (payload MQTT) que aún no han sido confirmados por el broker

## CR-03: firma OTA inválida

**Dado** el Config/OTA Manager procesando un manifiesto OTA
**Cuando** la verificación cosign del binario descargado falla (firma inválida, clave desconocida o archivo corrupto)
**Entonces** el agente descarta el binario temporal, no reemplaza el binario activo, publica un evento de error en `devices/<device_id>/health` con el campo `ota_error`, y continúa operando con el binario actual sin reiniciarse

## CR-04: certificado mTLS vencido o rechazado por el broker

**Dado** el Uploader intentando establecer conexión MQTT con el broker
**Cuando** el certificado de dispositivo ha vencido o el broker rechaza la conexión por revocación
**Entonces** el agente no transmite datos, registra el error de autenticación en el log local, reporta el estado `cert_expired` vía Health Beacon cuando recupere conectividad con certificado válido, y la cola SQLite continúa acumulando eventos de forma local durante este período

## CR-05: Bloom filter ausente o corrupto al arrancar

**Dado** el agente iniciando cuando el archivo de Bloom filter en disco está ausente o corrupto
**Cuando** el Bloom Filter intenta cargar el filtro desde disco al arrancar
**Entonces** el agente arranca sin Bloom filter activo (modo degradado), todos los eventos se tratan como prioridad NORMAL, el Collector registra una advertencia en el log indicando operación sin filtro, y el agente solicita una sincronización completa del filtro al suscribirse al topic MQTT retained al recuperar conectividad

## CR-06: drift NTP excesivo

**Dado** el Health Beacon midiendo el drift del reloj del dispositivo
**Cuando** el drift NTP supera el umbral configurado (por defecto 5 segundos)
**Entonces** el Health Beacon publica el campo `ntp_drift_ms` con el valor real y eleva la alerta `clock_drift_warning` en el payload; el timestamp monotónico del evento sigue siendo válido para ordenamiento relativo, pero el timestamp wall-clock se marca con la bandera `clock_uncertain: true` en el payload MQTT del evento

## CR-07: imagen eliminada antes de ser subida

**Dado** el Uploader intentando subir una imagen referenciada por un evento en cola
**Cuando** la imagen local ha sido eliminada por la política de rotación antes de ser subida
**Entonces** el evento MQTT se publica igualmente con el campo `image_uri` ausente y el campo `image_unavailable: true`; el evento no se bloquea ni se descarta por falta de imagen; el cloud procesa el evento de metadatos de forma independiente
