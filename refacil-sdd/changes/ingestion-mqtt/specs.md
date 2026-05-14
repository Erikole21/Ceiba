# specs: ingestion-mqtt

## CA-01: Conexión mTLS de dispositivo válido aceptada

**Dado** un dispositivo de borde con un certificado X.509 vigente emitido por la CA de Vault (ADR-010) configurado en su agente
**Cuando** el dispositivo establece una conexión MQTT 5 al clúster EMQX presentando dicho certificado
**Entonces** el broker acepta la conexión, crea o reanuda la sesión persistente del dispositivo, y el `device_id` extraído del campo CN o SAN del certificado queda asociado a esa sesión

## CA-02: Rechazo de conexión sin certificado de cliente

**Dado** un cliente que intenta conectarse al endpoint MQTT del clúster EMQX sin presentar certificado de cliente
**Cuando** se completa el handshake TLS
**Entonces** el broker cierra la conexión con error TLS durante el handshake, sin llegar a procesar el paquete MQTT CONNECT, y registra el intento en el log de auditoría

## CA-03: ACL — publicación en tópico propio permitida

**Dado** un dispositivo autenticado con `device_id = "D-001"` y `country_code = "CO"`
**Cuando** publica un mensaje en `devices/D-001/events` o `devices/D-001/health` o `devices/D-001/image-uri`
**Entonces** el broker acepta el mensaje, lo encola en la sesión del bridge y retorna PUBACK al dispositivo (QoS 1)

## CA-04: ACL — publicación en tópico ajeno rechazada

**Dado** un dispositivo autenticado con `device_id = "D-001"`
**Cuando** intenta publicar un mensaje en `devices/D-002/events` (tópico de otro dispositivo)
**Entonces** el broker rechaza el PUBLISH con código de razón MQTT 5 `0x87 Not Authorized`, desconecta al cliente y registra la violación en el log de auditoría

## CA-05: ACL — suscripción a tópicos de configuración y bloom-filter propios

**Dado** un dispositivo autenticado con `device_id = "D-001"` y `country_code = "CO"`
**Cuando** se suscribe a `devices/D-001/config`, `countries/CO/bloom-filter` y `devices/D-001/ota`
**Entonces** el broker confirma las tres suscripciones con QoS máximo concedido y entrega los mensajes retained existentes en esos tópicos

## CA-06: Entrega de evento al tópico Kafka con orden por dispositivo

**Dado** un dispositivo con `device_id = "D-001"` que publica diez eventos en secuencia en `devices/D-001/events`
**Cuando** el MQTT→Kafka Bridge procesa los mensajes
**Entonces** los diez mensajes aparecen en el tópico Kafka `vehicle.events.raw` en el mismo orden de publicación, todos asignados a la misma partición Kafka (determinada por la clave `device_id`)

## CA-07: Eventos de dispositivos distintos no comparten partición

**Dado** dos dispositivos `D-001` y `D-002` publicando eventos concurrentemente
**Cuando** el bridge los envía a Kafka
**Entonces** los eventos de `D-001` y los de `D-002` se asignan a particiones Kafka distintas (o pueden coincidir por hash, pero nunca se mezcla el orden relativo dentro de cada dispositivo)

## CA-08: Emisión de URL pre-firmada para dispositivo válido

**Dado** un dispositivo autenticado con certificado mTLS vigente que solicita al Upload Service una URL para subir una imagen
**Cuando** el Upload Service valida la identidad (verificación del certificado contra la CA de Vault o JWT emitido por Vault) y el `device_id` del certificado
**Entonces** el servicio retorna una URL pre-firmada S3-API con TTL de 5 minutos exactos, método HTTP PUT, y el path `images/{country_code}/{device_id}/{date}/{image_id}`

## CA-09: Subida directa de imagen vía URL pre-firmada

**Dado** un dispositivo que recibió una URL pre-firmada válida
**Cuando** realiza un HTTP PUT a esa URL con el binario de la imagen dentro del TTL de 5 minutos
**Entonces** el object storage almacena la imagen cifrada con clave KMS en la ruta especificada por la URL, retorna HTTP 200, y el dispositivo puede publicar el URI resultante en `devices/{device_id}/image-uri`

## CA-10: Fallback proxy-upload para dispositivos detrás de NAT/APN

**Dado** un dispositivo configurado con `upload_mode: proxy` que no puede alcanzar el object storage directamente
**Cuando** envía la imagen al endpoint proxy del Upload Service vía HTTP POST
**Entonces** el Upload Service recibe el binario, lo sube al object storage en nombre del dispositivo, y responde al dispositivo con el URI final de la imagen almacenada

## CA-11: Lifecycle policy aplicada en object storage

**Dado** que el object storage tiene configurada la lifecycle policy correspondiente
**Cuando** una imagen completa cumple 6 meses de antigüedad o una miniatura cumple 24 meses
**Entonces** el objeto es eliminado automáticamente por la política de ciclo de vida; no hay acción manual requerida

## CA-12: Sesión persistente reanudada tras reconexión

**Dado** un dispositivo con sesión persistente establecida que pierde la conectividad GSM temporalmente
**Cuando** el dispositivo se reconecta al clúster EMQX antes de que expire el `session_expiry_interval` configurado
**Entonces** el broker reanuda la sesión existente, entrega los mensajes QoS 1 pendientes (retained en la sesión), y el dispositivo no pierde eventos publicados durante la desconexión si fueron persistidos por el agente localmente

## CA-13: Rechazo de certificado revocado vía OCSP

**Dado** un dispositivo cuyo certificado fue revocado en Vault y la revocación está reflejada en el endpoint OCSP
**Cuando** el dispositivo intenta reconectarse al clúster EMQX
**Entonces** el broker rechaza la conexión durante el handshake TLS tras consultar OCSP, registra el evento y no permite acceso a ningún tópico

## CA-14: Cifrado server-side en object storage verificable

**Dado** que el object storage tiene cifrado server-side configurado con clave KMS
**Cuando** se almacena cualquier objeto (imagen o miniatura)
**Entonces** el objeto queda cifrado en reposo con la clave KMS; la API S3-API del objeto retorna el header `x-amz-server-side-encryption` (o equivalente del proveedor) con valor no vacío

## CA-15: Health beacon del bridge expuesto como métrica

**Dado** que el MQTT→Kafka Bridge está en operación
**Cuando** el sistema de monitoreo consulta las métricas del bridge
**Entonces** se observan al menos: mensajes recibidos por segundo desde EMQX, mensajes publicados por segundo a Kafka, lag de procesamiento en milisegundos y tasa de errores de publicación Kafka

---

## CR-01: Expiración de URL pre-firmada

**Dado** un dispositivo que recibió una URL pre-firmada válida con TTL de 5 minutos
**Cuando** intenta realizar el HTTP PUT a esa URL pasados los 5 minutos desde su emisión
**Entonces** el object storage rechaza la operación con HTTP 403 Forbidden; el dispositivo debe solicitar una nueva URL al Upload Service

## CR-02: Rechazo de upload sin identidad validada

**Dado** un cliente sin certificado mTLS válido ni JWT vigente que solicita una URL pre-firmada al Upload Service
**Cuando** el servicio procesa la solicitud
**Entonces** el Upload Service retorna HTTP 401 Unauthorized sin emitir ninguna URL, y registra el intento fallido

## CR-03: Saturación del canal GSM — mensaje MQTT descartado por payload excesivo

**Dado** un dispositivo que intenta publicar un mensaje en `devices/{device_id}/events` con un payload superior al límite configurado en EMQX (por ejemplo, > 10 KB para metadatos de evento)
**Cuando** el broker recibe el PUBLISH
**Entonces** el broker rechaza el mensaje con código de razón MQTT 5 `0x95 Packet Too Large` y no lo encola; el dispositivo es notificado y debe enviar la imagen por el canal de presigned URL

## CR-04: Nodo EMQX caído — alta disponibilidad del clúster

**Dado** un clúster EMQX de 3 nodos en operación con dispositivos conectados
**Cuando** uno de los nodos falla (caída de proceso o pérdida de red)
**Entonces** los dispositivos conectados a ese nodo se reconectan automáticamente a alguno de los otros dos nodos dentro del tiempo de keepalive MQTT configurado; la entrega de mensajes QoS 1 continúa sin pérdida para sesiones persistentes

## CR-05: Agotamiento de sesiones EMQX por dispositivos offline

**Dado** un escenario con miles de dispositivos offline con sesiones persistentes acumuladas
**Cuando** el `session_expiry_interval` configurado expira para una sesión
**Entonces** EMQX libera los recursos de esa sesión (suscripciones, mensajes pendientes) automáticamente; el consumo de memoria del clúster se mantiene dentro del presupuesto operativo definido en el runbook

## CR-06: Bridge MQTT→Kafka — Kafka no disponible temporalmente

**Dado** que el clúster Kafka está temporalmente inaccesible (mantenimiento, fallo de red)
**Cuando** el MQTT→Kafka Bridge intenta publicar un evento
**Entonces** el bridge aplica backoff exponencial y reintenta; los mensajes QoS 1 permanecen en la sesión EMQX (sin PUBACK al broker hasta confirmación Kafka) o en el buffer local del bridge según la implementación elegida; no se pierden eventos durante una indisponibilidad de Kafka inferior al `session_expiry_interval`

## CR-07: Tormenta de reconexiones simultáneas

**Dado** un escenario en que una pérdida masiva de conectividad GSM hace que un gran número de dispositivos intenten reconectarse simultáneamente al clúster EMQX
**Cuando** se recupera la conectividad
**Entonces** el clúster EMQX acepta reconexiones sin degradación de servicio para los dispositivos ya conectados; el runbook de escalado define el umbral de reconexiones simultáneas por nodo y las métricas de alerta

## CR-08: Adaptador de object storage — proveedor no disponible

**Dado** que el proveedor de object storage configurado (MinIO u otro) no está disponible
**Cuando** el Upload Service intenta completar un proxy-upload o verificar disponibilidad
**Entonces** el servicio retorna HTTP 503 Service Unavailable al dispositivo con un header `Retry-After`; no almacena datos parciales ni expone detalles internos del error al cliente
