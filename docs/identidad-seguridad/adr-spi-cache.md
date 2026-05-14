# ADR — Estrategia de Caché en SPI Custom SOAP/REST

**ID:** ADR-spi-cache
**Estado:** Aprobado
**Fecha:** 2026-05-13
**Relacionado con:** [federation-spi-custom.md](./federation-spi-custom.md)

---

## Contexto

El SPI custom de Keycloak (`federation-spi-custom.md`) delega la autenticación a endpoints SOAP o REST de los sistemas legacy de cada país. Estos endpoints presentan características operacionales problemáticas:

- Disponibilidad variable: pueden tener ventanas de mantenimiento no anunciadas, timeouts frecuentes y tasas de error elevadas.
- Latencia alta: algunos servicios SOAP tienen tiempos de respuesta de 500 ms a 3 s.
- Sin SLA garantizado hacia el sistema.

Sin caché, cada intento de autenticación realiza una llamada sincrónica al endpoint upstream. Si el endpoint no está disponible, el usuario no puede autenticarse aunque sus credenciales sean correctas y estén vigentes. Además, en escenarios de alta carga, el SPI puede saturar el endpoint upstream con llamadas concurrentes.

El riesgo opuesto es el de una caché muy larga: si las credenciales de un usuario son revocadas en el sistema del país (baja del funcionario, compromiso de cuenta), el SPI seguiría autenticando con el resultado cacheado durante el período de TTL.

---

## Alternativas evaluadas

### Alternativa A — Sin caché

Cada autenticación realiza una llamada al endpoint upstream sin ningún tipo de almacenamiento de resultado.

**Ventajas:**

- Máxima seguridad: cualquier cambio de credenciales en el sistema upstream se refleja de inmediato.
- Implementación más simple.

**Desventajas:**

- Si el endpoint upstream no está disponible, la autenticación falla para todos los usuarios del realm, incluso aquellos con credenciales completamente válidas.
- El sistema queda indisponible para un país entero durante cualquier ventana de mantenimiento del IdP del país.
- No cumple el objetivo de disponibilidad del 99,9 % del sistema cuando los IdPs upstream tienen menor SLA.
- No protege contra saturación del endpoint upstream en picos de autenticación.

**Veredicto:** descartada. La dependencia directa del uptime del IdP del país para la disponibilidad del sistema es inaceptable.

### Alternativa B — Caché en memoria con TTL configurable (seleccionada)

Los resultados exitosos de autenticación se almacenan en memoria del proceso SPI con un TTL configurable (valor por defecto: 5 minutos). Durante el período de TTL, las autenticaciones con las mismas credenciales se resuelven desde la caché sin invocar el upstream.

**Ventajas:**

- Permite autenticar usuarios durante interrupciones cortas del IdP upstream (< TTL).
- Reduce la carga sobre los endpoints SOAP/REST en picos de autenticación.
- TTL configurable permite ajustar el equilibrio entre disponibilidad y riesgo de ventana de credenciales comprometidas.
- Implementación contenida en el SPI, sin dependencias externas adicionales.

**Desventajas:**

- Ventana de riesgo: si las credenciales de un usuario son revocadas en el upstream, el SPI continuará autenticando durante hasta TTL minutos (por defecto, 5 minutos).
- La caché no se comparte entre instancias del SPI (no es distribuida). En un cluster de Keycloak con múltiples pods, cada pod tiene su propia caché.
- Los fallos de autenticación no se cachean (el SPI siempre consulta el upstream para intentos fallidos).

**Veredicto:** seleccionada. La ventana de riesgo de 5 minutos es aceptable dado que: (a) la revocación de credenciales en sistemas policiales raramente es inmediata; (b) el TTL del access token de Keycloak (5 minutos) limita el impacto real; y (c) el beneficio de disponibilidad supera el riesgo.

### Alternativa C — Caché persistente en Redis

Los resultados de autenticación se almacenan en Redis, compartidos entre todas las instancias del SPI.

**Ventajas:**

- Caché compartida entre todos los pods de Keycloak: un usuario autenticado en cualquier pod usa la caché.
- Persistencia ante reinicios del pod.

**Desventajas:**

- Añade Redis como dependencia del plano de control de autenticación: si Redis no está disponible, el SPI no puede leer la caché y el comportamiento cae al caso "sin caché".
- Mayor complejidad operacional y de implementación.
- Las credenciales cacheadas en Redis representan un riesgo adicional si el Redis no está debidamente protegido (las credenciales deben almacenarse como hash, no en claro).
- El beneficio marginal sobre la caché en memoria (shared cache) no justifica la complejidad adicional para el perfil de carga previsto.

**Veredicto:** descartada. La complejidad añadida no justifica el beneficio para el caso de uso. Se puede reconsiderar si el perfil de carga demuestra que la caché por pod es insuficiente.

---

## Decisión

**Se adopta caché en memoria con TTL configurable en el SPI custom. El TTL por defecto es de 5 minutos.**

Parámetros de configuración:

| Parámetro | Valor por defecto | Descripción |
|---|---|---|
| `cache.enabled` | `true` | Activa/desactiva la caché |
| `cache.ttl_seconds` | `300` | TTL en segundos (5 minutos) |
| `cache.max_entries` | `10000` | Número máximo de entradas en caché por instancia |

La clave de caché es: `sha256(username + ":" + realm_id)`. Las credenciales nunca se almacenan en caché — solo el resultado de autenticación (éxito/fallo es solo para éxitos; los fallos no se cachean).

---

## Riesgo aceptado

El riesgo principal es la **ventana de autenticación con credenciales revocadas**: un usuario cuyas credenciales son revocadas en el sistema upstream puede continuar autenticándose durante hasta `cache.ttl_seconds` segundos (5 minutos por defecto).

Mitigaciones:

1. El acceso real a los datos está controlado también por la sesión de Keycloak (TTL de sesión) y por el TTL del access token (5 minutos por defecto). Una vez que el access token expira, el sistema necesita un nuevo token, lo que puede detectar el cambio de estado.
2. Keycloak puede forzar el cierre de sesiones activas vía la API de administración cuando se detecta una revocación (proceso operacional documentado).
3. El TTL es configurable: para realms con requisitos de seguridad más estrictos, se puede reducir a 1 o 2 minutos, aceptando mayor dependencia del uptime del upstream.

---

## Consecuencias

- El SPI custom implementa la lógica de caché en memoria como parte del componente (véase [federation-spi-custom.md](./federation-spi-custom.md)).
- El TTL se configura como parámetro del provider en la consola de administración de Keycloak o vía API.
- Las métricas `spi.cache.hits` y `spi.cache.misses` se exponen para observabilidad.
- En caso de fallo del upstream con caché expirada, el SPI retorna `UPSTREAM_IDP_UNAVAILABLE` sin bypass (véase `CR-01` en specs).

---

## Referencias

- [federation-spi-custom.md](./federation-spi-custom.md) — Especificación técnica del SPI custom
- [audit-authentication.md](./audit-authentication.md) — Auditoría de eventos de autenticación
