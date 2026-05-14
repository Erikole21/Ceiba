# specs: identidad-seguridad

---

## CA-01: Autenticación de usuario mediante federación LDAP y emisión de JWT con claims canónicos

**Dado** que el realm `CO` de Keycloak tiene configurado el User Storage SPI — LDAP apuntando al Active Directory de la Policía Nacional de Colombia, con mapeo de atributo `ldap_role` → `role` y `ldap_country` → `country_code`
**Cuando** un agente policial ingresa sus credenciales LDAP (username / password) en la aplicación web
**Entonces** Keycloak autentica al usuario contra el LDAP, genera un token de acceso JWT firmado con la clave del realm, el token incluye los claims `country_code = "CO"`, `role = "officer"`, `zone = "<zona_asignada>"`, `sub`, `iss`, `exp`, `iat`, `jti`; la firma es verificable contra el JWKS endpoint `/realms/CO/protocol/openid-connect/certs`

## CA-02: Autenticación de usuario mediante federación JDBC y emisión de JWT con claims canónicos

**Dado** que el realm `MX` de Keycloak tiene configurado el User Storage SPI — JDBC conectado a la base de datos relacional de la Secretaría de Seguridad de México (MySQL), con query configurada para lookup por username y validación de hash de contraseña, y mapeo de columna `perfil` → `role`
**Cuando** un supervisor ingresa sus credenciales en la aplicación web
**Entonces** Keycloak consulta la BD relacional, valida la contraseña, genera un JWT con `country_code = "MX"`, `role = "supervisor"`, `zone` correcta; el token es verificable contra el JWKS endpoint del realm `MX`; no se modifica ningún registro en la base de datos del país (solo lectura)

## CA-03: Autenticación de usuario mediante SPI custom SOAP/REST y caché de credenciales

**Dado** que el realm `VE` de Keycloak tiene registrado un User Storage SPI custom que delega la autenticación a un servicio SOAP legacy de Venezuela, y la caché de credenciales tiene un TTL de 5 minutos
**Cuando** un usuario se autentica exitosamente por primera vez y, dentro del período de TTL, realiza una segunda autenticación con las mismas credenciales
**Entonces** la primera autenticación invoca el servicio SOAP y almacena el resultado en caché; la segunda autenticación se resuelve desde la caché local sin invocar el servicio upstream; en ambos casos el JWT resultante contiene `country_code = "VE"` con los claims correctos; si el TTL expira, la siguiente autenticación vuelve a invocar el servicio SOAP

## CA-04: Delegación de autenticación a IdP externo vía SAML 2.0

**Dado** que el realm `AR` tiene configurado un Identity Provider de Keycloak apuntando al IdP SAML 2.0 de la Policía Federal Argentina, con mapeo de atributo de aserción `Perfil` → `role` y `Pais` → `country_code`
**Cuando** un usuario inicia sesión y es redirigido al IdP argentino para autenticarse
**Entonces** Keycloak consume la aserción SAML, mapea los atributos al modelo de usuario interno, emite un JWT con `country_code = "AR"` y el `role` correspondiente; el token es válido para acceder a los servicios internos del sistema; el claim `country_code` no puede ser sobrescrito por la aserción si difiere del valor fijo del realm

## CA-05: Aislamiento de realm — usuario de un país no puede autenticarse en otro realm

**Dado** que el usuario `jperez` existe únicamente en el realm `CO` de Keycloak
**Cuando** se intenta autenticar al usuario `jperez` usando el endpoint de token del realm `MX` (p.ej. `/realms/MX/protocol/openid-connect/token`)
**Entonces** Keycloak retorna HTTP 401 (credenciales inválidas) para el realm `MX`; no se emite ningún token; el evento se registra como fallo de autenticación en el event log del realm `MX`; no existe ningún mecanismo de "cross-realm federation" que permita acceso cruzado

## CA-06: Claim country_code en JWT fijado por el realm — no modificable por el usuario

**Dado** que el realm `CO` tiene configurado `country_code = "CO"` como atributo fijo de realm
**Cuando** un usuario autenticado en el realm `CO` intenta solicitar un token con un claim `country_code` diferente (p.ej. mediante un request manipulado al endpoint de token)
**Entonces** Keycloak ignora cualquier valor de `country_code` proporcionado en la solicitud y emite el token con `country_code = "CO"` fijo del realm; el API Gateway valida el token con el JWKS del realm `CO` y confía en el claim; ningún servicio interno acepta tokens con `country_code` diferente al del realm que los emitió

## CA-07: JWKS endpoint disponible por realm para validación de tokens

**Dado** que Keycloak está desplegado con el realm `CO` activo
**Cuando** el API Gateway realiza una solicitud GET a `/realms/CO/protocol/openid-connect/certs`
**Entonces** Keycloak responde con HTTP 200 y un payload JSON con el conjunto de claves públicas JWK vigentes para el realm `CO`; el API Gateway puede verificar la firma del access token usando esas claves; la rotación de claves del realm actualiza el JWKS endpoint sin interrupción del servicio (las claves anteriores permanecen durante el período de transición configurado)

## CA-08: Bootstrap de certificado de dispositivo con token de un solo uso (Vault PKI)

**Dado** que un nuevo dispositivo `device-001` con `country_code = "CO"` ha recibido un token de bootstrap de un solo uso aprovisionado por el operador, y el mount `pki-co` está activo en Vault con la CA intermedia para Colombia
**Cuando** el agente de borde presenta el token de bootstrap al endpoint de Vault para solicitar su primer certificado
**Entonces** Vault emite un certificado X.509 con `CN=device:device-001`, `OU=country:CO`, TTL 90 días, firmado por la CA intermedia `pki-co`; el token de bootstrap es invalidado inmediatamente después del intercambio y no puede reutilizarse; el dispositivo almacena el certificado y puede establecer conexiones mTLS con EMQX

## CA-09: Renovación automática de certificado de dispositivo (método cert auth)

**Dado** que el dispositivo `device-001` tiene un certificado vigente a 14 días de su expiración y puede conectarse a Vault
**Cuando** el agente de borde genera una CSR y la presenta a Vault junto con el certificado vigente como credencial de autenticación (método `cert` auth)
**Entonces** Vault verifica que el certificado presentado es válido, no está revocado, y fue emitido por la CA `pki-co`; emite un nuevo certificado con el mismo `CN` y `OU`, TTL 90 días desde la fecha de emisión; el dispositivo reemplaza su certificado anterior; la conexión mTLS con EMQX se puede restablecer con el nuevo certificado sin intervención manual

## CA-10: Revocación de certificado de dispositivo y verificación CRL por EMQX

**Dado** que el dispositivo `device-002` ha sido comprometido o dado de baja y se ejecuta `vault pki revoke` con el número de serie de su certificado
**Cuando** EMQX recibe una nueva solicitud de conexión mTLS desde `device-002` presentando el certificado revocado
**Entonces** EMQX descarga o verifica la CRL actualizada desde el endpoint PKI de Vault; rechaza la conexión con error TLS indicando que el certificado está revocado; el dispositivo no puede publicar ni suscribirse hasta que obtenga un nuevo certificado por el proceso de re-bootstrap; el rechazo queda registrado en los logs de EMQX

## CA-11: EMQX aplica ACL por device_id extraído del CN del certificado

**Dado** que el dispositivo `device-003` con `CN=device:device-003` establece una conexión mTLS exitosa con EMQX
**Cuando** el dispositivo intenta publicar en el tópico `devices/device-003/events` (permitido) y también intenta publicar en `devices/device-999/events` (no permitido — pertenece a otro dispositivo)
**Entonces** EMQX permite la publicación en `devices/device-003/events`; rechaza la publicación en `devices/device-999/events` con código de error MQTT de autorización; la ACL es evaluada por EMQX usando el CN del certificado como `device_id` sin consultar un servicio externo en el camino caliente de publicación

## CA-12: API Gateway valida JWT y extrae claims para enrutamiento RBAC

**Dado** que un usuario con `role = "analyst"` porta un access token válido y realiza una solicitud GET al endpoint de búsqueda operacional de placas (endpoint restringido a roles `officer` y `supervisor`)
**Cuando** el API Gateway recibe la solicitud y valida el token contra el JWKS endpoint del realm correspondiente
**Entonces** el token es criptográficamente válido (firma, `exp`, `iss`); el API Gateway extrae los claims `country_code`, `role`, `zone` y los propaga como headers internos (`X-Country-Code`, `X-Role`, `X-Zone`); como el `role = "analyst"` no está autorizado para ese endpoint, el API Gateway retorna HTTP 403 sin reenviar la solicitud al servicio interno

## CA-13: Service accounts para adaptadores de sincronización de países (RBAC sin usuario humano)

**Dado** que el adaptador de sincronización de Colombia (`sincronizacion-paises`) opera como service account registrado como cliente confidencial en el realm `CO` de Keycloak
**Cuando** el adaptador solicita un token de acceso usando el flujo `client_credentials` con su `client_id` y `client_secret`
**Entonces** Keycloak emite un JWT con `country_code = "CO"`, `role = "admin"` (scope de sincronización), `sub = "sync-adapter-co"`; el token no contiene `zone` (no aplica para service accounts); los servicios internos autorizan las operaciones del adaptador basados en ese token; el `client_secret` es inyectado vía Vault Agent sidecar (nunca en variables de entorno ni en el repositorio)

## CA-14: SSO entre aplicaciones del mismo realm

**Dado** que un usuario ya autenticado en la aplicación web (realm `CO`) tiene una sesión SSO activa en Keycloak
**Cuando** el mismo usuario accede a Superset (dashboard analítico) registrado como cliente OIDC en el realm `CO`
**Entonces** Keycloak reconoce la sesión SSO activa y emite un nuevo access token para Superset sin solicitar credenciales al usuario; el token para Superset contiene los mismos claims `country_code`, `role`, `zone`; si la sesión SSO ha expirado, Keycloak redirige al usuario al flujo de autenticación

## CA-15: MFA TOTP aplicado en el realm cuando está configurado

**Dado** que el realm `BR` tiene habilitado MFA obligatorio con TOTP para todos los roles excepto `officer`
**Cuando** un usuario con `role = "supervisor"` completa correctamente el paso de contraseña en el flujo de autenticación
**Entonces** Keycloak exige el segundo factor (código TOTP válido de la app de autenticación) antes de emitir el token; si el código TOTP es incorrecto, Keycloak rechaza la autenticación con HTTP 401 y registra el intento fallido; si el código es correcto, emite el JWT con todos los claims canónicos

## CA-16: Rotación de refresh token y expiración de sesión

**Dado** que el realm `CO` tiene configurado TTL de access token de 5 minutos y rotación de refresh token activa, y el usuario tiene un refresh token activo
**Cuando** el cliente presenta el refresh token para obtener un nuevo access token
**Entonces** Keycloak emite un nuevo access token y un nuevo refresh token (rotación); el refresh token anterior queda inmediatamente inválido; si el cliente intenta usar el refresh token anterior nuevamente, Keycloak lo rechaza y cierra la sesión completa del usuario (detección de reutilización de refresh token)

## CA-17: Exportación de eventos de auditoría de autenticación al log centralizado

**Dado** que Keycloak tiene configurado el exportador de eventos hacia el log centralizado (OpenSearch o Loki) para el realm `CO`
**Cuando** ocurre cualquier evento de autenticación (login exitoso, fallo de login, refresh de token, revocación de token, logout)
**Entonces** el evento es exportado al log centralizado dentro de los 30 s siguientes al evento, con los campos `realm_id`, `event_type`, `user_id` (o `client_id` para service accounts), `ip_address`, `timestamp`, `error` (si aplica); los eventos son consultables en OpenSearch o Loki y se retienen por al menos 7 años; ningún evento de autenticación se pierde aunque el log centralizado tenga una interrupción breve (buffer de escritura)

## CA-18: Vault HA con Raft — disponibilidad del 99,9 % y auto-unseal

**Dado** que Vault está desplegado en Kubernetes con topología HA (3 nodos Raft) y auto-unseal configurado con cloud KMS
**Cuando** el nodo líder de Vault falla (restart, fallo de pod, mantenimiento)
**Entonces** Raft elige un nuevo líder en menos de 30 s; el nuevo líder completa el auto-unseal usando el KMS configurado sin intervención manual; las operaciones de emisión de certificados y lectura de secretos se reanudan automáticamente; el tiempo de indisponibilidad es inferior al SLA del 99,9 % mensual

---

## CR-01: Fallo del IdP upstream en SPI custom — autenticación rechazada sin bypass

**Dado** que el servicio SOAP de autenticación de Venezuela está caído (timeout de conexión o HTTP 5xx)
**Cuando** un usuario del realm `VE` intenta autenticarse y la caché de credenciales del SPI custom ha expirado para ese usuario
**Entonces** el SPI custom retorna fallo de autenticación a Keycloak; Keycloak responde con HTTP 401 al cliente; no existe ningún modo de bypass que permita acceder al sistema sin validación upstream; el error se registra en los logs del SPI con el motivo `UPSTREAM_IDP_UNAVAILABLE`; el backoff exponencial evita saturar el servicio SOAP durante la recuperación

## CR-02: Token de bootstrap de dispositivo reutilizado — rechazo por Vault

**Dado** que un token de bootstrap de un solo uso ya fue utilizado para emitir el primer certificado del dispositivo `device-004`
**Cuando** un atacante o un proceso duplicado intenta usar el mismo token de bootstrap para solicitar otro certificado
**Entonces** Vault rechaza la solicitud con error de token inválido o expirado; no se emite ningún certificado nuevo; el intento queda registrado en los audit logs de Vault; el operador puede detectar el intento de reutilización mediante la alerta correspondiente

## CR-03: Certificado de dispositivo expirado — rechazo por EMQX sin excepción silenciosa

**Dado** que el certificado del dispositivo `device-005` expiró hace 1 hora (el agente no renovó en tiempo)
**Cuando** el dispositivo intenta establecer una conexión mTLS con EMQX presentando el certificado expirado
**Entonces** EMQX rechaza el handshake TLS con error de certificado expirado; el dispositivo no puede publicar datos; el rechazo es explícito en los logs de EMQX; el dispositivo debe iniciar el proceso de re-bootstrap con un nuevo token de un solo uso; no existe modo de "grace period" que permita conexiones con certificados expirados

## CR-04: JWT con claim country_code ausente o nulo rechazado por el API Gateway

**Dado** que un token de acceso fue emitido por un realm mal configurado (sin claim `country_code`) o es un token manipulado
**Cuando** el API Gateway recibe una solicitud con ese token y lo valida contra el JWKS endpoint
**Entonces** aunque la firma criptográfica sea válida, el API Gateway rechaza la solicitud con HTTP 401 o 403 indicando claim obligatorio ausente; no propaga la solicitud a ningún servicio interno; el evento queda registrado como anomalía de seguridad; no existe fallback que asuma un `country_code` por defecto

## CR-05: Validación de firma JWT — token manipulado rechazado por API Gateway

**Dado** que un atacante modifica el payload de un JWT (p.ej. cambia `role` de `analyst` a `admin`) manteniendo el formato base64
**Cuando** el API Gateway valida el token contra el JWKS endpoint del realm correspondiente
**Entonces** la validación de firma falla (el payload modificado no corresponde a la firma original); el API Gateway rechaza la solicitud con HTTP 401; ninguna solicitud con firma inválida llega a los servicios internos; el evento se registra como intento de acceso con token inválido

## CR-06: Intento de publicación MQTT de dispositivo en tópico de otro dispositivo — rechazo ACL

**Dado** que el dispositivo `device-006` (CN = `device:device-006`) está conectado a EMQX mediante mTLS
**Cuando** el dispositivo intenta publicar un mensaje en el tópico `devices/device-007/events` (perteneciente a otro dispositivo)
**Entonces** EMQX evalúa la ACL basada en el CN del certificado; rechaza la publicación con código de error MQTT de autorización (PUBACK con código de error, o desconexión según la versión MQTT); el dispositivo `device-006` continúa conectado y puede publicar en sus propios tópicos; el rechazo queda registrado en los logs de EMQX con el CN del dispositivo y el tópico denegado

## CR-07: Fallo de Vault no impide lectura de secretos ya inyectados en pods activos

**Dado** que Vault Agent sidecar ya inyectó las credenciales de base de datos en el pod de un servicio interno antes de la indisponibilidad de Vault
**Cuando** Vault entra en estado de fallo (todos los nodos caídos o sellados)
**Entonces** los pods con credenciales ya inyectadas continúan funcionando hasta que los secretos expiren; los nuevos pods que intenten arrancar no pueden obtener secretos e impiden su propio inicio (fail-closed); el servicio de monitoreo emite alerta de Vault no disponible; cuando Vault se recupera, los pods nuevos obtienen sus secretos y arrancan normalmente

## CR-08: Mapeo de atributos LDAP incompleto — autenticación rechazada sin claims parciales

**Dado** que el directorio LDAP de un país devuelve un usuario que no tiene el atributo LDAP mapeado a `role` (campo ausente en el directorio)
**Cuando** el usuario intenta autenticarse a través del User Storage SPI — LDAP
**Entonces** Keycloak puede completar la autenticación de credenciales pero no puede emitir un token con el claim `role` requerido; Keycloak rechaza la emisión del token con error indicando atributo obligatorio ausente; el evento se registra como `MISSING_ROLE_CLAIM`; no se emite ningún token parcial que pueda llegar a los servicios internos sin el claim `role`

## CR-09: Token de acceso expirado — rechazo explícito y no autenticación silenciosa

**Dado** que un cliente porta un access token cuyo `exp` fue hace 10 minutos
**Cuando** el cliente presenta el token expirado al API Gateway
**Entonces** el API Gateway detecta la expiración en la validación del claim `exp`; retorna HTTP 401 con indicación de token expirado; el cliente debe obtener un nuevo token usando el refresh token (si está vigente) o iniciar un nuevo flujo de autenticación; ninguna solicitud con token expirado llega a los servicios internos

## CR-10: Rotación de clave de firma del realm — tokens emitidos antes de la rotación siguen siendo válidos durante la transición

**Dado** que el realm `CO` rota su par de claves de firma (operación de mantenimiento programada) y el JWKS endpoint ahora expone tanto la clave nueva como la anterior durante el período de transición
**Cuando** un cliente presenta un access token válido firmado con la clave anterior (aún dentro de su TTL)
**Entonces** el API Gateway encuentra la clave anterior en el JWKS endpoint (identificada por `kid`), valida la firma correctamente, y autoriza la solicitud; los tokens emitidos con la clave nueva también son validados correctamente; no existe un período de interrupción del servicio durante la rotación de claves; una vez que todos los tokens antiguos hayan expirado, la clave anterior se elimina del JWKS endpoint

## CR-11: Certificado de dispositivo con CA desconocida rechazado por EMQX

**Dado** que un dispositivo no autorizado intenta conectarse a EMQX presentando un certificado autofirmado o firmado por una CA no reconocida (no es ninguna de las CAs intermedias del sistema)
**Cuando** EMQX valida el certificado de cliente durante el handshake TLS
**Entonces** EMQX rechaza la conexión con error TLS de CA no confiable; el handshake falla antes de que se procese ningún mensaje MQTT; el intento de conexión queda registrado en los logs de EMQX con la información del certificado presentado para análisis forense

---

> **Nota de validación cruzada entre repositorios:** Este change define los contratos de identidad que consumen múltiples changes del sistema. Los contratos establecidos aquí deben acordarse formalmente con los changes relacionados antes de la implementación. Consultar `refacil-prereqs/BUS-CROSS-REPO.md`.
>
> - **`agente-borde` (upstream, aprobado):** el proceso de bootstrap de certificado (token de un solo uso → primer cert), el método de renovación (`cert` auth con mTLS), y el TTL de 90 días son contratos que el agente de borde debe implementar. Cualquier cambio en el flujo de bootstrap o renovación requiere coordinación con ese change.
> - **`ingestion-mqtt` (upstream, aprobado):** la configuración de mTLS de EMQX (CA cert de cada país para validación de clientes), la política de ACL por `device_id` extraído del CN, y la verificación de CRL son contratos definidos aquí que `ingestion-mqtt` debe implementar en su configuración de EMQX. La estructura de tópicos (`devices/{device_id}/#`, `config/{device_id}/#`, `bloomfilter/{country_code}/#`) es parte de ese contrato.
> - **`almacenamiento-lectura` (upstream, aprobado):** el claim `country_code` del JWT es el discriminador de tenant usado en todos los almacenes de datos. El schema del claim (`country_code`: string ISO 3166-1 alpha-2, obligatorio, fijado por el realm) es el contrato que ese change usa para el particionamiento, aliases de índice y prefijos de clave. Cualquier cambio en el claim debe acordarse con ese change.
> - **`backbone-procesamiento` (upstream, aprobado):** los claims `country_code`, `role` y `zone` propagados por el API Gateway como headers internos son consumidos por los servicios del backbone para aplicar RBAC en sus operaciones. El nombre y formato de los headers internos (`X-Country-Code`, `X-Role`, `X-Zone`) debe acordarse con ese change.
> - **`api-frontend-analitica` (downstream, propuesto):** el JWKS endpoint por realm, la estructura del JWT (claims, algoritmo de firma, TTL), y el flujo OIDC para la Web App son contratos que ese change debe implementar en el API Gateway y en la lógica de autenticación del frontend. El formato de claims y los endpoints de Keycloak documentados aquí son la especificación de interfaz.
