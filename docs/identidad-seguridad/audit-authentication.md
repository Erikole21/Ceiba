# Auditoría de Autenticación

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Tipos de eventos auditados por Keycloak

Keycloak registra los siguientes tipos de eventos de autenticación en el event log:

| Tipo de evento | Descripción |
|---|---|
| `LOGIN` | Autenticación exitosa de un usuario |
| `LOGIN_ERROR` | Intento de autenticación fallido (credenciales incorrectas, cuenta bloqueada, etc.) |
| `LOGOUT` | Cierre de sesión explícito del usuario |
| `LOGOUT_ERROR` | Error durante el proceso de logout |
| `REFRESH_TOKEN` | Renovación exitosa del access token mediante refresh token |
| `REFRESH_TOKEN_ERROR` | Intento de renovación de token fallido (token expirado, reutilización detectada) |
| `REFRESH_TOKEN_REUSE_DETECTED` | Se detectó reutilización de un refresh token ya invalidado (posible replay attack) |
| `REGISTER` | Registro de un nuevo usuario (si aplica por realm) |
| `REGISTER_ERROR` | Error durante el registro |
| `RESET_PASSWORD` | Restablecimiento de contraseña exitoso |
| `RESET_PASSWORD_ERROR` | Error durante el restablecimiento de contraseña |
| `UPDATE_PASSWORD` | Actualización de contraseña exitosa |
| `UPDATE_TOTP` | Configuración de TOTP exitosa |
| `REMOVE_TOTP` | Eliminación de TOTP |
| `CODE_TO_TOKEN` | Intercambio de authorization code por token (flujo OAuth) |
| `CODE_TO_TOKEN_ERROR` | Error en el intercambio de authorization code |
| `CLIENT_LOGIN` | Autenticación exitosa de un service account (client_credentials) |
| `CLIENT_LOGIN_ERROR` | Error en la autenticación de un service account |
| `TOKEN_EXCHANGE` | Intercambio de token |
| `IDENTITY_PROVIDER_LOGIN` | Autenticación vía IdP externo (SAML, OIDC) |
| `IDENTITY_PROVIDER_FIRST_LOGIN` | Primera autenticación de un usuario via IdP externo |
| `INTROSPECT_TOKEN` | Introspección de token |

---

## 2. Schema del evento exportado

Todos los eventos se exportan con el siguiente esquema:

| Campo | Tipo | Descripción | Ejemplo |
|---|---|---|---|
| `id` | `string` (UUID v4) | Identificador único del evento | `"a1b2c3d4-..."` |
| `time` | `number` (Unix ms) | Timestamp del evento en milisegundos | `1748000000000` |
| `type` | `string` | Tipo de evento de la lista anterior | `"LOGIN"` |
| `realmId` | `string` | ID del realm (= `country_code`) | `"CO"` |
| `clientId` | `string` | Client ID del cliente de Keycloak | `"web-app"` |
| `userId` | `string` (UUID) | ID del usuario en el realm (nulo para CLIENT_LOGIN) | `"a1b2c3d4-..."` |
| `sessionId` | `string` (UUID) | ID de la sesión SSO | `"f7e8d9c0-..."` |
| `ipAddress` | `string` | Dirección IP del cliente | `"190.122.45.67"` |
| `error` | `string` | Código de error si aplica (nulo en eventos exitosos) | `"invalid_user_credentials"` |
| `details` | `object` (JSONB) | Detalles adicionales según el tipo de evento | `{"username": "jperez", "reason": "..."}` |

---

## 3. Exportador de eventos — Event Listener SPI

Keycloak no exporta eventos de forma nativa a sistemas externos. Se implementa un **Event Listener SPI** custom que captura todos los eventos del Event Store de Keycloak y los reenvía al log centralizado.

### 3.1 Exportación a OpenSearch

```java
// EventListenerProvider implementation (pseudocódigo)
public class AuditEventListenerProvider implements EventListenerProvider {
    @Override
    public void onEvent(Event event) {
        AuditEvent auditEvent = AuditEvent.builder()
            .id(UUID.randomUUID().toString())
            .time(event.getTime())
            .type(event.getType().name())
            .realmId(event.getRealmId())
            .clientId(event.getClientId())
            .userId(event.getUserId())
            .sessionId(event.getSessionId())
            .ipAddress(event.getIpAddress())
            .error(event.getError())
            .details(event.getDetails())
            .build();
        // Enviar a OpenSearch vía buffer asíncrono
        opensearchClient.index(auditEvent);
    }
}
```

### 3.2 Exportación a Loki

Para entornos donde OpenSearch se usa solo para eventos operacionales, los eventos de auditoría se exportan a Loki con etiquetas:

```
{realm="CO", event_type="LOGIN_ERROR", service="keycloak"}
```

### 3.3 Garantía de no pérdida de eventos

El exportador usa un buffer de escritura en memoria con persistencia en disco (o en un topic de Kafka) como respaldo. Si el log centralizado tiene una interrupción breve (< 5 minutos), los eventos se mantienen en el buffer y se envían al recuperarse la conexión. Esto cumple el requisito de que ningún evento de autenticación se pierda ante interrupciones breves del backend de logs.

---

## 4. Política de retención

Todos los eventos de autenticación se retienen durante **7 años** en el log centralizado, en cumplimiento de:

- Requisitos de auditoría de seguridad de las instituciones policiales.
- Regulaciones de datos personales aplicables en cada país (Ley 1581 en Colombia y equivalentes).

La retención se implementa mediante:

- **OpenSearch:** ILM (Index Lifecycle Management) con política de transición a almacenamiento de menor costo (cold tier) a los 90 días y eliminación a los 7 años.
- **Loki:** retention policy configurada en el ruler: `7y`.
- **S3/Object Storage:** archivado de eventos exportados con S3 Object Lock para inmutabilidad durante 7 años.

---

## 5. Dashboard de monitoreo (Grafana/Kibana)

### Panel principal de autenticación

| Métrica | Visualización | Descripción |
|---|---|---|
| Tasa de fallos de autenticación por país | Time series (1h rolling) | `COUNT(event_type=LOGIN_ERROR) / COUNT(event_type IN [LOGIN, LOGIN_ERROR])` |
| Top 10 usuarios con más fallos | Table | Ordenados por número de `LOGIN_ERROR` en las últimas 24 h |
| Mapa de distribución por IP | Geo heatmap | Distribución geográfica de los intentos de autenticación |
| Fallos por tipo de error | Bar chart | `invalid_user_credentials`, `account_disabled`, `UPSTREAM_IDP_UNAVAILABLE`, etc. |
| Autenticaciones por tipo de fuente | Pie chart | LDAP, JDBC, SPI custom, SAML, OIDC |
| Sesiones activas por país | Gauge | Número de sesiones SSO activas por realm |

---

## 6. Alertas de seguridad

| Alerta | Condición | Canal | Severidad |
|---|---|---|---|
| Brute force por usuario | `> 10 LOGIN_ERROR` del mismo `userId` en 5 minutos | Canal de seguridad (Slack/PagerDuty) | Alta |
| Brute force por IP | `> 50 LOGIN_ERROR` desde la misma `ipAddress` en 5 minutos | Canal de seguridad | Alta |
| Replay de refresh token | Cualquier evento `REFRESH_TOKEN_REUSE_DETECTED` | Canal de seguridad | Crítica |
| IdP upstream no disponible | `> 20 LOGIN_ERROR` con error `UPSTREAM_IDP_UNAVAILABLE` en 5 minutos | Canal de operaciones | Alta |
| Tasa de error > 20 % | Tasa de fallos de autenticación > 20 % en 15 minutos para un realm | Canal de operaciones | Media |
| Token con `country_code` ausente | Cualquier solicitud al API Gateway rechazada por `country_code` ausente | Canal de seguridad | Alta |

### Ejemplo de alerta de brute force en Grafana

```yaml
# Regla de alerta Grafana/Loki
- alert: BruteForceByUser
  expr: |
    sum by (userId, realmId) (
      count_over_time(
        {service="keycloak", event_type="LOGIN_ERROR"}[5m]
      )
    ) > 10
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "Posible brute force detectado"
    description: "Usuario {{ $labels.userId }} en realm {{ $labels.realmId }}: más de 10 fallos en 5 minutos"
```
