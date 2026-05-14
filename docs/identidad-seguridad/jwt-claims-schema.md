# Schema de Claims JWT Canónicos

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Algoritmo de firma

Todos los tokens de acceso emitidos por Keycloak en este sistema usan **RS256** (RSA PKCS#1 v1.5 con SHA-256). El par de claves por realm es RSA de mínimo **2048 bits** (recomendado 4096 bits para realms de producción).

La clave pública de verificación se expone en el endpoint JWKS canónico de cada realm:

```
GET https://{keycloak_host}/realms/{country_code}/protocol/openid-connect/certs
```

Ejemplos:

- `https://auth.sistema.com/realms/CO/protocol/openid-connect/certs`
- `https://auth.sistema.com/realms/MX/protocol/openid-connect/certs`

El campo `kid` en el header del JWT identifica la clave de verificación dentro del conjunto JWKS. Durante la rotación de claves, el JWKS expone tanto la clave nueva como la anterior durante el período de transición configurado.

---

## 2. Claims obligatorios

| Claim | Tipo | Formato | Restricciones | Ejemplo |
|---|---|---|---|---|
| `sub` | `string` | UUID v4 o identificador único del usuario en el realm | Inmutable, único por realm | `"a1b2c3d4-e5f6-7890-abcd-ef1234567890"` |
| `iss` | `string` | URL del realm emisor | Debe coincidir con el realm del `country_code` | `"https://auth.sistema.com/realms/CO"` |
| `exp` | `number` | Unix timestamp (segundos) | Mayor al tiempo actual; TTL configurado por realm | `1748000000` |
| `iat` | `number` | Unix timestamp (segundos) | Igual al momento de emisión | `1747999700` |
| `jti` | `string` | UUID v4 | Único por token; usado para detección de replay | `"f7e8d9c0-b1a2-3456-cdef-890123456789"` |
| `country_code` | `string` | ISO 3166-1 alpha-2 en mayúsculas | Fijado por configuración del realm; no puede ser sobreescrito por el usuario | `"CO"` |
| `role` | `string` | Enum: `officer`, `supervisor`, `analyst`, `admin`, `auditor` | Exactamente uno de los cinco valores; mapeado desde la fuente de identidad | `"officer"` |
| `zone` | `string` | Identificador de zona geográfica del país | Asignado por la fuente de identidad; puede ser string libre o código según el país | `"bogota-norte"` |

### Regla de fijación de `country_code`

El claim `country_code` **nunca** proviene del usuario, de la sesión HTTP, ni de la fuente de identidad federada. Se configura como atributo de sesión fijo en el mapper de Keycloak asociado al realm:

```
Mapper type: Hardcoded claim
Token Claim Name: country_code
Claim value: CO   ← valor fijo en la configuración del realm
```

Cualquier intento de proporcionar un `country_code` diferente en la solicitud de token es ignorado por Keycloak. El API Gateway valida que el claim `country_code` esté presente y no sea nulo; cualquier token sin este claim es rechazado con HTTP 401.

---

## 3. Claims opcionales

| Claim | Tipo | Descripción | Presente en |
|---|---|---|---|
| `given_name` | `string` | Nombre del usuario | Tokens de usuario humano cuando la fuente de identidad lo provee |
| `family_name` | `string` | Apellido del usuario | Tokens de usuario humano cuando la fuente de identidad lo provee |
| `email` | `string` | Correo electrónico | Tokens de usuario humano cuando la fuente de identidad lo provee |
| `preferred_username` | `string` | Nombre de usuario en el realm | Tokens de usuario humano |
| `azp` | `string` | Client ID del cliente autorizado | Todos los tokens |
| `session_state` | `string` | Identificador de sesión SSO | Tokens de usuario humano (no en client_credentials) |
| `scope` | `string` | Scopes OAuth 2.0 concedidos | Todos los tokens |

---

## 4. Ejemplos de payload JWT decodificado por rol

### 4.1 Oficial (`officer`)

```json
{
  "sub": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "iss": "https://auth.sistema.com/realms/CO",
  "exp": 1748000300,
  "iat": 1748000000,
  "jti": "f7e8d9c0-b1a2-3456-cdef-890123456789",
  "azp": "web-app",
  "scope": "openid profile",
  "country_code": "CO",
  "role": "officer",
  "zone": "bogota-norte",
  "given_name": "Juan",
  "family_name": "Pérez",
  "email": "jperez@policia.co",
  "preferred_username": "jperez",
  "session_state": "9d8e7f6a-5b4c-3d2e-1f0a-bc9876543210"
}
```

### 4.2 Supervisor (`supervisor`)

```json
{
  "sub": "b2c3d4e5-f6a7-8901-bcde-f01234567891",
  "iss": "https://auth.sistema.com/realms/MX",
  "exp": 1748000300,
  "iat": 1748000000,
  "jti": "e6d7c8b9-a0f1-2345-bcde-789012345678",
  "azp": "web-app",
  "scope": "openid profile",
  "country_code": "MX",
  "role": "supervisor",
  "zone": "cdmx-sur",
  "given_name": "María",
  "family_name": "García",
  "email": "mgarcia@sspc.gob.mx",
  "preferred_username": "mgarcia"
}
```

### 4.3 Analista (`analyst`)

```json
{
  "sub": "c3d4e5f6-a7b8-9012-cdef-012345678902",
  "iss": "https://auth.sistema.com/realms/AR",
  "exp": 1748000300,
  "iat": 1748000000,
  "jti": "d5c6b7a8-9f0e-1234-abcd-678901234567",
  "azp": "superset",
  "scope": "openid profile",
  "country_code": "AR",
  "role": "analyst",
  "zone": "argentina-nacional",
  "given_name": "Carlos",
  "family_name": "López",
  "preferred_username": "clopez"
}
```

### 4.4 Administrador (`admin`)

```json
{
  "sub": "d4e5f6a7-b8c9-0123-defa-123456789013",
  "iss": "https://auth.sistema.com/realms/CO",
  "exp": 1748000300,
  "iat": 1748000000,
  "jti": "c4b5a6f7-8e9d-0123-abcd-567890123456",
  "azp": "admin-console",
  "scope": "openid profile admin",
  "country_code": "CO",
  "role": "admin",
  "zone": "colombia-nacional",
  "preferred_username": "admin.co"
}
```

### 4.5 Auditor (`auditor`)

```json
{
  "sub": "e5f6a7b8-c9d0-1234-efab-234567890124",
  "iss": "https://auth.sistema.com/realms/CO",
  "exp": 1748000300,
  "iat": 1748000000,
  "jti": "b3a4f5e6-7d8c-9012-abcd-456789012345",
  "azp": "audit-app",
  "scope": "openid profile audit",
  "country_code": "CO",
  "role": "auditor",
  "zone": "colombia-nacional",
  "preferred_username": "auditor.co"
}
```

---

## 5. Tokens de service account (`client_credentials`)

Los adaptadores de sincronización de países y otros servicios internos usan el flujo `client_credentials`. En estos tokens:

- `sub` es el `client_id` del cliente de Keycloak (no un UUID de usuario).
- `zone` no aplica para service accounts y puede estar ausente.
- `role` refleja el scope del servicio (por ejemplo, `admin` para el adaptador de sincronización).

```json
{
  "sub": "sync-adapter-co",
  "iss": "https://auth.sistema.com/realms/CO",
  "exp": 1748003600,
  "iat": 1748000000,
  "jti": "a2b3c4d5-e6f7-0123-abcd-345678901234",
  "azp": "sync-adapter-co",
  "scope": "sync",
  "country_code": "CO",
  "role": "admin"
}
```

---

## 6. Notas de contrato con otros módulos

### `almacenamiento-lectura`

El claim `country_code` (string ISO 3166-1 alpha-2, obligatorio, no nulo) es el discriminador de tenant usado en:

- Particionamiento de tablas PostgreSQL (`events`, `stolen_vehicles`).
- Aliases de índice en OpenSearch (`events-{country_code}-*`).
- Prefijos de clave en Redis (`stolen:{country_code}:*`).

Cualquier cambio en el tipo, formato o nombre del claim requiere coordinación con el módulo `almacenamiento-lectura`.

### `api-frontend-analitica`

El API Gateway valida el token contra el JWKS del realm indicado en `iss` e inyecta los claims como headers internos:

- `X-Country-Code: {country_code}`
- `X-Role: {role}`
- `X-Zone: {zone}`

Los servicios internos confían en estos headers y no validan el JWT directamente. La responsabilidad de validación es exclusivamente del API Gateway.

### `backbone-procesamiento`

Los headers internos `X-Country-Code`, `X-Role`, `X-Zone` son consumidos por los servicios del backbone para aplicar RBAC en operaciones de escritura y para enriquecer eventos con el contexto del usuario que los originó.
