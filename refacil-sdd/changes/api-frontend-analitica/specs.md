# specs: api-frontend-analitica

---

## CA-01: API Gateway valida JWT y rechaza solicitudes sin token

**Dado** que un cliente envía una solicitud HTTP a cualquier endpoint del sistema sin cabecera `Authorization`
**Cuando** el API Gateway recibe la solicitud
**Entonces** el Gateway responde con HTTP 401 y un cuerpo de error estándar (`{"error":"unauthorized","message":"missing or invalid JWT"}`); ninguna solicitud sin JWT válido alcanza un microservicio de dominio; la métrica `gateway.auth.rejected` se incrementa

## CA-02: API Gateway extrae claims y los propaga como headers internos

**Dado** que un oficial con JWT válido que contiene `country_code = "CO"`, `role = "officer"` y `zone = "ZONA-NORTE"` realiza una solicitud a `GET /v1/plates/ABC123X/events`
**Cuando** el API Gateway valida el token contra el JWKS endpoint de Keycloak
**Entonces** el Gateway reenvía la solicitud a search-service con las cabeceras internas `X-Country-Code: CO`, `X-Role: officer` y `X-Zone: ZONA-NORTE`; search-service filtra todos los resultados usando `country_code = "CO"` sin revalidar el JWT; ningún dato de otro país aparece en la respuesta

## CA-03: Rate-limiting por tenant+endpoint bloquea exceso de solicitudes

**Dado** que el rate-limiting está configurado en 100 solicitudes por minuto por tenant para el endpoint `GET /v1/plates/{plate}/events`, y el tenant `"CO"` ha enviado exactamente 100 solicitudes en los últimos 60 s
**Cuando** el tenant `"CO"` envía la solicitud número 101 en el mismo período
**Entonces** el API Gateway responde con HTTP 429 (`{"error":"too_many_requests"}`), con las cabeceras `Retry-After` y `X-RateLimit-Remaining: 0`; el contador de Redis para `ratelimit:CO:/v1/plates/events:{window}` refleja el bloqueo; el tenant `"MX"` en el mismo período no se ve afectado

## CA-04: search-service retorna eventos paginados de placa dentro del SLO

**Dado** que la placa `"ABC123X"` tiene 150 eventos en OpenSearch y PostgreSQL para el país `"CO"`, distribuidos en los últimos 30 días
**Cuando** un oficial autenticado solicita `GET /v1/plates/ABC123X/events?country=CO&page=1&pageSize=20`
**Entonces** search-service retorna 20 eventos ordenados por `timestamp_capture` descendente; cada evento incluye `timestamp`, `lat`, `lon`, `device_id`, `confidence`, `thumbnail_url` (URL pre-firmada GET con TTL de 5 min) y `country`/`city`; la respuesta incluye metadatos de paginación (`totalCount`, `page`, `pageSize`, `totalPages`); el tiempo de respuesta p95 es inferior a 300 ms

## CA-05: search-service aplica búsqueda fuzzy en OpenSearch antes de enriquecer en PostgreSQL

**Dado** que la placa normalizada en OpenSearch es `"ABC123X"` y el usuario busca `"ABC123"` (un carácter de diferencia)
**Cuando** search-service ejecuta la consulta con `fuzziness: AUTO` en el índice `vehicle-events-co-{YYYY-MM}`
**Entonces** OpenSearch retorna los `event_id` correspondientes con score de relevancia; search-service enriquece esos event_ids con coordenadas PostGIS y metadatos de PostgreSQL; la respuesta final incluye los eventos enriquecidos con thumbnail_url pre-firmada; el tiempo de búsqueda fuzzy p95 es inferior a 300 ms

## CA-06: search-service genera URLs pre-firmadas de lectura con TTL de 5 minutos

**Dado** que un evento tiene `thumbnail_uri = "s3://bucket/CO/thumbnails/E-001.jpg"` almacenado en PostgreSQL
**Cuando** search-service construye la respuesta de ese evento
**Entonces** el campo `thumbnail_url` en la respuesta contiene una URL pre-firmada GET válida para el proveedor de object storage configurado (MinIO o S3-compatible); la URL expira exactamente a los 300 s desde su generación; el puerto hexagonal `ObjectStoragePort` abstrae la implementación concreta del proveedor

## CA-07: alerts-service enruta alertas Kafka solo a oficiales de la zona correcta

**Dado** que el tópico Kafka `vehicle.alerts` contiene una alerta con `zone = "ZONA-NORTE"` y `country_code = "CO"`, y hay dos oficiales conectados por WebSocket: oficial A con `zone = "ZONA-NORTE"` y `country_code = "CO"`, y oficial B con `zone = "ZONA-SUR"` y `country_code = "CO"`
**Cuando** alerts-service consume la alerta desde Kafka
**Entonces** el Gateway WebSocket entrega la alerta únicamente al oficial A; el oficial B no recibe la alerta; el payload entregado incluye `plate`, `event_timestamp`, `lat`, `lon`, `thumbnail_url`, `device_id` y `alert_id`; la métrica `alerts.delivered` se incrementa con la etiqueta `zone = ZONA-NORTE`

## CA-08: alerts-service retorna alertas históricas pendientes por polling REST

**Dado** que un oficial que estuvo desconectado desde `2026-05-12T10:00:00Z` reconnecta a las `11:00:00Z`
**Cuando** el oficial solicita `GET /v1/alerts?since=2026-05-12T10:00:00Z&country=CO`
**Entonces** alerts-service retorna todas las alertas generadas en ese intervalo para `country_code = "CO"` y `zone` del oficial; la respuesta incluye el mismo payload que las alertas en tiempo real; los eventos se ordenan por `event_timestamp` ascendente; la respuesta está limitada a un máximo configurable de alertas (p.ej. 500) con indicador de truncamiento si aplica

## CA-09: incident-service persiste acción de recuperación y registra en audit_log

**Dado** que un oficial autenticado con rol `"officer"` y `country_code = "CO"` envía `POST /v1/incidents/ABC123X/recovery-action` con `{"action": "dispatch", "unit_id": "PATRULLA-12", "notes": "Avistado en Av. El Dorado"}`
**Cuando** incident-service procesa la solicitud
**Entonces** se crea o actualiza un registro en la tabla `incidents` de PostgreSQL con el nuevo estado y la acción registrada; se inserta una entrada inmutable en `audit_log` con `entity_type = "incident"`, `entity_id`, `event_type = "DISPATCH"`, `actor_id` del oficial y `timestamp`; el servicio emite un evento al tópico Kafka `incident.events` con el mismo payload; la respuesta HTTP es 201 con el estado actualizado del incidente

## CA-10: incident-service notifica a sincronizacion-paises al cerrar un caso

**Dado** que un incidente para la placa `"ABC123X"` en Colombia está en estado `"active"` y un supervisor envía `PATCH /v1/incidents/{id}/status` con `{"status": "recovered"}`
**Cuando** incident-service procesa el cierre del caso
**Entonces** el estado del incidente en PostgreSQL pasa a `"recovered"`; incident-service llama al puerto hexagonal `CountrySyncPort` para notificar a `sincronizacion-paises`; si el adaptador del país soporta write-back, el estado se propaga a la BD policial colombiana; si el adaptador no soporta write-back, la notificación se registra como `write_back_not_supported` en el audit_log sin generar error al usuario; la respuesta HTTP es 200 con el nuevo estado del incidente

## CA-11: analytics-service retorna mapa de calor H3 dentro del SLO

**Dado** que la vista materializada H3 de resolución 8 está pre-calculada para Colombia con datos de los últimos 7 días
**Cuando** un analista autenticado solicita `GET /v1/analytics/heatmap?country=CO&from=2026-05-05&to=2026-05-12&resolution=8`
**Entonces** analytics-service retorna un array de celdas H3 con `h3_index`, `event_count` y `avg_confidence`; los datos provienen de la vista materializada `vehicle_events_h3_hourly` de ClickHouse (no de la tabla raw); el tiempo de respuesta p95 es inferior a 3 s; analytics-service no expone credenciales de ClickHouse al cliente

## CA-12: analytics-service retorna tendencias de hurto por periodo configurado

**Dado** que ClickHouse contiene datos de hurtos para Colombia en los últimos 90 días
**Cuando** un analista solicita `GET /v1/analytics/trends?country=CO&city=Bogota&period=30d`
**Entonces** analytics-service retorna una serie temporal con `date`, `theft_count` y `country`/`city` para los últimos 30 días filtrados por ciudad; los valores reflejan los datos de ClickHouse agrupados por día; la respuesta incluye el período efectivo consultado; el tiempo de respuesta p95 es inferior a 3 s para el período de 30 días

## CA-13: Web App muestra la trayectoria completa de una placa en el mapa

**Dado** que search-service retorna 35 eventos para la placa `"ABC123X"` en Colombia
**Cuando** el oficial ve la vista de búsqueda de placa en la Web App
**Entonces** MapLibre GL renderiza un punto por evento en sus coordenadas lat/lon; cada punto tiene un popup con fecha/hora, ubicación y miniatura accesible desde la thumbnail_url; la lista lateral muestra los 35 eventos paginados; al hacer clic en un evento de la lista, el mapa centra y resalta el punto correspondiente; la miniatura se carga desde la URL pre-firmada directamente en el navegador del usuario

## CA-14: Web App recibe y muestra una alerta en tiempo real por WebSocket

**Dado** que el oficial está autenticado y tiene la vista de alertas en tiempo real abierta en el navegador
**Cuando** el WebSocket Gateway entrega una alerta con `plate = "XYZ789"` para la zona del oficial
**Entonces** aparece una tarjeta de alerta nueva en la UI sin necesidad de refrescar la página; la tarjeta muestra placa, timestamp, coordenadas textuales, miniatura del vehículo y botones de acción (despachar unidad, abrir incidente); el mapa resalta la ubicación de la alerta; el tiempo entre la producción del evento en Kafka y la aparición de la tarjeta en la UI es inferior a 2 s en condiciones normales de red

## CA-15: Web App aplica multi-tenancy y no muestra datos de otros países

**Dado** que el usuario autenticado tiene `country_code = "CO"` en su JWT
**Cuando** el usuario navega por cualquier vista de la Web App (búsqueda, alertas, incidentes, analítica)
**Entonces** todos los datos visibles corresponden únicamente a Colombia; no existe ningún selector de país que permita al usuario cambiar al contexto de `"MX"` u otro país; los endpoints de API son llamados siempre con el `country_code` del JWT del usuario; la UI no renderiza ningún dato que no coincida con `country_code = "CO"`

## CA-16: Web App renderiza capa de mapa de calor con Kepler.gl para el analista

**Dado** que analytics-service retorna datos H3 de hexbinning para Colombia (7 días)
**Cuando** un analista con rol `"analyst"` abre la vista analítica
**Entonces** Kepler.gl renderiza la capa de mapa de calor sobre MapLibre GL con las celdas H3 coloreadas según `event_count`; el analista puede ajustar la ventana temporal (7d / 30d / 90d) y el filtro de país desde la UI; el mapa actualiza la capa al cambiar el filtro; el embed o enlace a Superset está disponible en la misma vista para el acceso a dashboards BI detallados

## CA-17: devices-service gestiona ciclo completo de dispositivo y disparo de OTA

**Dado** que un administrador con rol `"admin"` y `country_code = "CO"` solicita `POST /v1/devices/{id}/ota` con el payload de versión de firmware
**Cuando** devices-service procesa la solicitud
**Entonces** el dispositivo queda registrado con `firmware_status = "pending_ota"`; devices-service registra la acción en el log de auditoría; la respuesta HTTP es 202 (aceptado, procesamiento asíncrono); el estado del dispositivo es consultable en `GET /v1/devices/{id}` con el campo `firmware_status` actualizado

## CA-18: vehicles-service actualiza la lista canónica de vehículos hurtados

**Dado** que un administrador con rol `"admin"` envía `PATCH /v1/vehicles/{id}` con `{"status": "recovered"}`
**Cuando** vehicles-service procesa la solicitud
**Entonces** la tabla `stolen_vehicles` de PostgreSQL actualiza el campo `status` del vehículo con `id = {id}`; se incrementa el campo `schema_version`; se inserta una entrada en `audit_log` con el cambio; la respuesta HTTP es 200 con el registro actualizado; ningún usuario con rol diferente a `"admin"` puede ejecutar esta operación (el API Gateway rechaza con 403)

---

## CR-01: Solicitud con JWT expirado es rechazada por el API Gateway

**Dado** que un cliente envía una solicitud con un JWT cuyo claim `exp` es anterior al timestamp actual
**Cuando** el API Gateway valida el token contra el JWKS endpoint de Keycloak
**Entonces** el Gateway responde con HTTP 401 (`{"error":"token_expired"}`); la solicitud no alcanza ningún microservicio de dominio; el cliente debe obtener un nuevo JWT antes de reintentar

## CR-02: search-service retorna 404 semántico cuando no hay eventos para la placa

**Dado** que la placa `"ZZZ999"` no existe en OpenSearch ni en PostgreSQL para `country_code = "CO"`
**Cuando** un oficial solicita `GET /v1/plates/ZZZ999/events?country=CO`
**Entonces** search-service responde con HTTP 404 y un cuerpo semántico (`{"error":"plate_not_found","plate":"ZZZ999","country":"CO"}`); no se retorna un array vacío sin mensaje de contexto; la métrica `search.plate_not_found` se incrementa; el tiempo de respuesta sigue siendo inferior a 300 ms

## CR-03: search-service maneja fallo de OpenSearch sin bloquear la respuesta

**Dado** que OpenSearch no está disponible (timeout de conexión)
**Cuando** search-service intenta ejecutar la búsqueda fuzzy de placa
**Entonces** el servicio responde con HTTP 503 (`{"error":"search_unavailable","retry_after":30}`); no propaga la excepción sin manejar; la métrica `search.opensearch_errors` se incrementa; el circuit breaker del adaptador hexagonal `EventsIndexPort` registra el fallo; PostgreSQL no es consultado en ese intento (depende del resultado de OpenSearch)

## CR-04: alerts-service no entrega alertas a oficiales de zona incorrecta o país diferente

**Dado** que una alerta con `country_code = "MX"` y `zone = "ZONA-CENTRO"` llega al tópico `vehicle.alerts`
**Cuando** alerts-service procesa la alerta
**Entonces** la alerta no se entrega a ningún oficial con `country_code = "CO"`, independientemente de su zona; la alerta no se entrega a oficiales de México con `zone` diferente a `"ZONA-CENTRO"`; el Gateway WebSocket mantiene el registro de `country_code` + `zone` por cada conexión activa como criterio de enrutamiento

## CR-05: incident-service rechaza acción inválida sobre un incidente cerrado

**Dado** que el incidente con `id = "INC-001"` está en estado `"recovered"` (caso cerrado)
**Cuando** un oficial envía `POST /v1/incidents/ABC123X/recovery-action` con `{"action": "dispatch"}` referenciando ese incidente
**Entonces** incident-service responde con HTTP 409 (`{"error":"conflict","message":"incident already closed","incident_id":"INC-001","current_status":"recovered"}`); el registro en PostgreSQL no se modifica; no se emite ningún evento al tópico `incident.events`; el audit_log no registra la acción rechazada

## CR-06: analytics-service devuelve advertencia cuando la vista H3 está desactualizada

**Dado** que la vista materializada H3 de ClickHouse no ha sido actualizada en las últimas 2 h (p.ej. primer arranque tras reinicio) y analytics-service recibe una solicitud de mapa de calor
**Cuando** analytics-service detecta que la antigüedad de los datos supera el umbral configurado
**Entonces** el servicio retorna los datos disponibles (posiblemente parciales) con la cabecera `X-SLO-Degraded: true` y el campo `data_freshness_warning` en el cuerpo de la respuesta; no retorna HTTP 503 salvo que no haya datos en absoluto; la métrica `analytics.slo_degraded` se incrementa

## CR-07: WebSocket Gateway gestiona desconexión abrupta del oficial sin perder alertas

**Dado** que un oficial pierde la conexión WebSocket mientras hay alertas nuevas en el tópico `vehicle.alerts` para su zona
**Cuando** el oficial reconecta y solicita `GET /v1/alerts?since={ultima_conexion}&country=CO`
**Entonces** alerts-service retorna todas las alertas perdidas durante la desconexión, ordenadas por `event_timestamp`; el Gateway WebSocket libera los recursos de la conexión anterior al detectar el cierre; no se generan duplicados de alerta al reconectar

## CR-08: vehicles-service impide escritura por roles no autorizados

**Dado** que un usuario con rol `"officer"` intenta `POST /v1/vehicles` o `PATCH /v1/vehicles/{id}`
**Cuando** el API Gateway evalúa el claim `role` del JWT
**Entonces** el Gateway responde con HTTP 403 (`{"error":"forbidden","required_role":"admin"}`); la solicitud no alcanza vehicles-service; la métrica `gateway.authz.rejected` se incrementa con la etiqueta `endpoint=/v1/vehicles`

## CR-09: search-service supera el SLO de mapa cuando se excede el umbral de eventos

**Dado** que una placa tiene más de 10 K eventos activos en PostgreSQL+PostGIS para el período consultado
**Cuando** search-service construye la respuesta del mapa
**Entonces** el servicio aplica una estrategia de truncamiento o agrupación geoespacial para reducir los puntos enviados al cliente a un máximo configurable (p.ej. 10 K por página); si el tiempo de respuesta supera los 800 ms, la respuesta incluye la cabecera `X-SLO-Degraded: true`; la métrica `search.map_slo_exceeded` se incrementa; nunca se devuelven más de `pageSize` eventos en una única respuesta

## CR-10: API Gateway maneja fallo del JWKS endpoint de Keycloak con degradado controlado

**Dado** que el JWKS endpoint de Keycloak no está disponible temporalmente
**Cuando** el API Gateway intenta validar un JWT recibido
**Entonces** el Gateway utiliza las claves públicas en caché (TTL configurable, recomendado 5 min) si las tiene disponibles y el JWT no está expirado; si el caché de claves está vacío o expirado, responde con HTTP 503 (`{"error":"auth_service_unavailable"}`); no se omite la validación de JWT bajo ninguna circunstancia; la métrica `gateway.jwks_cache_miss` se incrementa cuando se usa el caché

---

> **Nota de validación cruzada entre repositorios:** Este change consume contratos de lectura definidos en `almacenamiento-lectura` y produce contratos de notificación hacia `sincronizacion-paises`. Los criterios de este change dependen de los schemas, índices y estructuras de clave definidos en esos changes. Consultar `refacil-prereqs/BUS-CROSS-REPO.md` para el proceso de acuerdo formal.
>
> - **`almacenamiento-lectura` (upstream, aprobado):** los contratos de lectura de OpenSearch (búsqueda fuzzy sobre `vehicle-events-{country_code}-{YYYY-MM}`, aliases por país), PostgreSQL+PostGIS (tablas `vehicle_events`, `incidents`, `audit_log`, `stolen_vehicles` con sus índices), ClickHouse (vistas materializadas H3 `vehicle_events_h3_hourly`) y Redis (estructura de clave `stolen:{country_code}:{plate_normalized}`) son la especificación de interfaz que los adaptadores hexagonales de este change deben implementar. Cualquier cambio en esos contratos requiere re-validación con este change.
> - **`backbone-procesamiento` (upstream, aprobado):** el schema del payload del tópico `vehicle.alerts` consumido por alerts-service debe ser acordado formalmente; cualquier cambio en el schema del mensaje requiere notificación y re-validación.
> - **`sincronizacion-paises` (upstream, aprobado):** la interfaz del `CountrySyncPort` que incident-service usa para notificar cierres de caso debe ser acordada con ese change; el comportamiento ante adaptadores de país sin soporte de write-back debe estar documentado explícitamente en la interfaz.
> - **`identidad-seguridad` (downstream, pendiente):** el schema del JWT (claims `country_code`, `role`, `zone`) y la URL del JWKS endpoint son el contrato entre Keycloak y el API Gateway de este change. Cualquier cambio en los nombres de claims o en la estructura del token requiere re-validación formal con este change antes de la implementación de `identidad-seguridad`.
