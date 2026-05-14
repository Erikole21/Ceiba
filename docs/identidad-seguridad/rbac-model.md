# Modelo RBAC — Roles y Permisos

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Definición de los cinco roles

| Rol | Descripción | Scope de datos | Herencia |
|---|---|---|---|
| `officer` | Oficial de policía operativo | Solo su `zone` (eventos y alertas en vivo) | — |
| `supervisor` | Supervisor policial | Su `zone` + gestión de su equipo | Hereda permisos de `officer` |
| `analyst` | Analista de inteligencia | Todo el `country_code` (lectura) | — |
| `admin` | Administrador del sistema del país | Todo el `country_code` (lectura + escritura de configuración) | — |
| `auditor` | Auditor de seguridad | Todo el `country_code` (solo lectura de audit logs y métricas) | — |

### Herencia de roles

`supervisor` hereda todos los permisos de `officer`. Es decir, un supervisor puede realizar cualquier acción que un oficial puede realizar dentro de su zona, más acciones adicionales de supervisión.

---

## 2. Tabla de permisos por rol y recurso

| Recurso / Operación | `officer` | `supervisor` | `analyst` | `admin` | `auditor` |
|---|---|---|---|---|---|
| Buscar eventos por placa (propia zona) | Lectura | Lectura | — | Lectura | — |
| Buscar eventos por placa (todo el país) | — | — | Lectura | Lectura | — |
| Ver trayectoria en mapa (propia zona) | Lectura | Lectura | — | Lectura | — |
| Ver trayectoria en mapa (todo el país) | — | — | Lectura | Lectura | — |
| Ver imagen/prueba visual de eventos | Lectura | Lectura | Lectura | Lectura | — |
| Recibir alertas en tiempo real (propia zona) | Lectura | Lectura | — | — | — |
| Gestionar recuperación de vehículo | Escritura | Escritura | — | — | — |
| Escalar incidente | — | Escritura | — | — | — |
| Cerrar incidente / marcar recuperado | — | Escritura | — | — | — |
| Ver dashboards analíticos (país) | — | — | Lectura | Lectura | Lectura |
| Ver heatmaps y tendencias (país) | — | — | Lectura | Lectura | Lectura |
| Gestionar usuarios del país | — | — | — | Escritura | — |
| Configurar dispositivos | — | — | — | Escritura | — |
| Ver audit logs | — | — | — | — | Lectura |
| Ver métricas de autenticación | — | — | — | Escritura | Lectura |
| Exportar datos (CSV/Excel) | — | — | Lectura | Lectura | — |

---

## 3. Reglas de scope geográfico

### Regla de zona para `officer` y `supervisor`

Los roles `officer` y `supervisor` pueden ver únicamente los datos de **eventos en vivo y alertas** correspondientes a su `zone`. Esta restricción se aplica en:

1. El API Gateway: filtra las solicitudes de eventos en vivo usando el claim `zone` del JWT.
2. El WebSocket Gateway: suscribe al oficial únicamente a los canales de alerta de su zona.
3. Los servicios internos: verifican el header `X-Zone` para filtrar resultados.

### Regla de país para `analyst`, `admin` y `auditor`

Los roles `analyst`, `admin` y `auditor` tienen acceso de lectura a **todos los datos de su `country_code`** sin restricción de zona. La restricción a un solo país se aplica mediante el claim `country_code` del JWT.

### Regla de aislamiento inter-país

Ningún rol tiene acceso a datos de un `country_code` diferente al del realm que emitió su token. Esta garantía la provee la arquitectura de realm-per-country (ver [adr-realm-por-pais.md](./adr-realm-por-pais.md)).

---

## 4. Mapeo de roles a scopes OAuth 2.0

| Rol | Scopes OAuth 2.0 |
|---|---|
| `officer` | `openid profile events:read:zone alerts:read:zone recovery:write` |
| `supervisor` | `openid profile events:read:zone alerts:read:zone recovery:write incidents:write` |
| `analyst` | `openid profile events:read:country analytics:read` |
| `admin` | `openid profile events:read:country analytics:read config:write users:write` |
| `auditor` | `openid profile audit:read metrics:read` |

---

## 5. Configuración del API Gateway — Enforcement de RBAC

La siguiente tabla define las reglas de autorización que el API Gateway (Kong/Envoy) aplica en cada endpoint:

| Método HTTP | Patrón de ruta | Roles permitidos | Filtro adicional |
|---|---|---|---|
| `GET` | `/v1/plates/{plate}/events` | `officer`, `supervisor`, `analyst`, `admin` | `officer`/`supervisor`: filtra por `zone`; `analyst`/`admin`: sin filtro |
| `GET` | `/v1/plates/{plate}/map` | `officer`, `supervisor`, `analyst`, `admin` | Igual al anterior |
| `GET` | `/v1/alerts/live` | `officer`, `supervisor` | Filtra por `zone` del JWT |
| `POST` | `/v1/incidents/{id}/recovery-action` | `officer`, `supervisor` | Solo incidentes de su `zone` |
| `POST` | `/v1/incidents/{id}/escalate` | `supervisor` | Solo incidentes de su `zone` |
| `PATCH` | `/v1/incidents/{id}/close` | `supervisor` | Solo incidentes de su `zone` |
| `GET` | `/v1/analytics/*` | `analyst`, `admin`, `auditor` | Sin filtro de zona |
| `GET` | `/v1/heatmaps/*` | `analyst`, `admin`, `auditor` | Sin filtro de zona |
| `POST` | `/v1/admin/users` | `admin` | Solo usuarios del mismo `country_code` |
| `PUT` | `/v1/admin/devices/*` | `admin` | Solo dispositivos del mismo `country_code` |
| `GET` | `/v1/audit/events` | `auditor` | Solo audit logs del mismo `country_code` |
| `GET` | `/v1/metrics/*` | `admin`, `auditor` | Solo métricas del mismo `country_code` |
| `GET` | `/v1/export/*` | `analyst`, `admin` | Solo datos del mismo `country_code` |

### Nota de responsabilidad de enforcement

**El API Gateway es el único punto de enforcement de RBAC para solicitudes HTTP externas.** Los servicios internos confían en los headers `X-Country-Code`, `X-Role`, `X-Zone` inyectados por el API Gateway y aplican filtros de datos adicionales (por zona, por país) basados en esos headers. Los servicios internos no validan el JWT directamente; esa responsabilidad es exclusiva del API Gateway.

---

## 6. Ejemplo de verificación de autorización en el API Gateway

```bash
# Solicitud de un oficial (role=officer, zone=bogota-norte)
# Intenta acceder a endpoint de analítica (solo analyst/admin/auditor)
curl -H "Authorization: Bearer <officer_token>" \
     https://api.sistema.com/v1/analytics/trends

# Respuesta esperada del API Gateway:
# HTTP/1.1 403 Forbidden
# {"error": "insufficient_scope", "required": ["analyst", "admin", "auditor"], "current": "officer"}

# Solicitud del mismo oficial a endpoint de su zona (permitido)
curl -H "Authorization: Bearer <officer_token>" \
     "https://api.sistema.com/v1/alerts/live?zone=bogota-norte"

# Respuesta esperada:
# HTTP/1.1 200 OK
# Solo se devuelven alertas de la zona bogota-norte
```
