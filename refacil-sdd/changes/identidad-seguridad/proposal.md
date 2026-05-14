# proposal: identidad-seguridad

## Objetivo

El Sistema Anti-Hurto de Vehículos federa instituciones policiales de múltiples países, cada una con su propio mecanismo de autenticación (LDAP, base de datos relacional, SOAP legacy, REST propietario, o un IdP corporativo SAML/OIDC). Este change define la capa de identidad federada y seguridad que resuelve esa heterogeneidad: Keycloak actúa como broker de identidad con un realm por país, normaliza cualquier fuente de autenticación externa a tokens JWT/OIDC estándar con claims `country_code`, `role` y `zone`, y proporciona al mismo tiempo la PKI de dispositivos vía HashiCorp Vault para autenticación criptográfica mTLS de los agentes de borde. El resultado es una capa de identidad unificada y cloud-agnostic que habilita el RBAC fino por país/rol y el aislamiento multi-tenant requerido por ADR-011.

## Alcance

**Incluye:**

- **Keycloak — broker de identidad:**
  - Despliegue de Keycloak en Kubernetes con backend PostgreSQL (Patroni on-prem o BD gestionada — mismo Helm chart, values distintos)
  - Modelo de realm por país: namespace aislado con usuarios, roles, clientes, políticas de contraseña, configuración de sesión y proveedores de federación propios
  - Clientes registrados por realm: `web-app`, `api-gateway`, `superset`, servicios internos
  - Estructura canónica del JWT con claims obligatorios: `country_code` (ISO 3166-1 alpha-2), `role` (uno de `officer`, `supervisor`, `analyst`, `admin`, `auditor`), `zone` (zona geográfica interna del país), más claims OIDC estándar (`sub`, `iss`, `exp`, `iat`, `jti`)
  - JWKS endpoint por realm: `/realms/{country_code}/protocol/openid-connect/certs`
  - SSO entre aplicaciones internas del mismo realm
  - MFA configurable por realm (TOTP, push); herencia de política de MFA del IdP del país cuando aplique
  - Gestión de sesiones: TTL de token configurable por realm; rotación de refresh token
  - Exportación de eventos de auditoría de autenticación a log centralizado (OpenSearch o Loki) con retención de 7 años

- **Proveedores de federación nativos (LDAP y JDBC):**
  - User Storage SPI — LDAP: proveedor nativo de Keycloak; conexión al LDAP/Active Directory de la institución policial; mapeo de atributos LDAP a atributos de usuario Keycloak incluyendo `country_code` y `role`; federación de solo lectura (sin write-back)
  - User Storage SPI — JDBC: proveedor nativo de base de datos; conexión a la BD relacional policial (PostgreSQL, MySQL, Oracle, SQL Server); query customizable de búsqueda y validación de credenciales; mapeo de columnas a atributos de usuario
  - Documentación de configuración por tipo: URL de conexión, credenciales de bind/acceso (almacenadas en Vault), estrategia de sincronización (on-demand vs. periódica), mapeo de atributos

- **SPI custom — federación SOAP/REST:**
  - Implementación personalizada de Keycloak User Storage SPI para instituciones con servicios SOAP legacy o REST propietarios
  - Empaquetado como contenedor desplegable registrado como extensión de Keycloak
  - Adaptador por país: cada país con IdP no estándar obtiene su propia implementación SPI
  - Contrato del adaptador: recibe username/password, invoca el endpoint de autenticación del país, valida la respuesta, mapea al modelo de usuario Keycloak con claims `country_code`, `role`, `zone`
  - Caché local de credenciales con TTL configurable (p.ej. 5 min) para evitar sobrecarga de sistemas legacy
  - Manejo de errores: fallo del IdP upstream → fallo de autenticación sin bypass; backoff exponencial en reintentos
  - Despliegue como sidecar o pod separado registrado como provider de Keycloak

- **Identity Provider SPI — delegación SAML/OIDC:**
  - Para instituciones con IdP propio compatible con SAML 2.0 u OIDC: Keycloak actúa como Service Provider / cliente OIDC y delega la autenticación al IdP del país
  - Mapeo de aserciones/tokens del IdP del país al formato interno de token Keycloak con claims obligatorios
  - Camino de federación opcional: no todos los países lo utilizan

- **PKI de dispositivos — HashiCorp Vault:**
  - Vault PKI Engine como CA para certificados de dispositivos
  - Una CA intermedia por país (mount `pki-{country_code}`) bajo una CA raíz común
  - Certificado por dispositivo: `CN=device:{device_id}`, `OU=country:{country_code}`, TTL 90 días
  - Proceso de bootstrap: el dispositivo recibe un token de bootstrap de un solo uso; lo intercambia por su primer certificado vía Vault token auth; el token es invalidado inmediatamente
  - Renovación automática: el agente de borde genera una CSR 14 días antes de la expiración, presenta el certificado actual + CSR a Vault, recibe el nuevo certificado (método `cert` auth — mTLS con el certificado vigente)
  - Revocación: `vault pki revoke` por `device_id`; CRL publicada en el endpoint PKI de Vault; EMQX verifica la CRL en cada conexión; endpoint OCSP disponible para verificación en tiempo real
  - Despliegue de Vault en Kubernetes (HA con Raft storage); auto-unseal con cloud KMS (AWS KMS, GCP Cloud KMS, Azure Key Vault) o HSM on-prem (PKCS#11) — adaptador hexagonal `VaultUnsealPort`
  - Vault Secrets Engine para todos los secretos de aplicación: credenciales de BD, API keys, credenciales S3, credenciales Kafka; políticas de rotación automática; Vault Agent sidecar para inyección de secretos en pods Kubernetes

- **RBAC — roles detallados por país:**
  - Cinco roles definidos a nivel de realm:
    - `officer`: búsqueda de placas, visualización de eventos en mapa, alertas en vivo, gestión de incidentes; scope = `country_code` + `zone` propios
    - `supervisor`: todos los permisos de `officer` + visualización de incidentes de otros agentes del mismo `country_code`; puede escalar incidentes
    - `analyst`: acceso de solo lectura a dashboards analíticos (mapas de calor, tendencias, rutas); sin acceso a eventos individuales ni placas; scope = `country_code` propio
    - `admin`: gestión de dispositivos (CRUD, OTA), administración de lista canónica de vehículos hurtados, configuración de país; scope = `country_code` propio
    - `auditor`: acceso de solo lectura a logs de auditoría; sin modificación de datos; scope = `country_code` propio
  - Asignación de roles gestionada en el realm de Keycloak; herencia desde el proveedor de federación mediante mapeo de atributos cuando sea posible
  - API Gateway aplica enrutamiento basado en rol: rol no autorizado para un endpoint → HTTP 403

- **mTLS para dispositivos:**
  - EMQX configurado con mTLS: requiere certificado de cliente firmado por la CA intermedia del país
  - ACL por dispositivo: el dispositivo solo puede publicar en `devices/{device_id}/#` y suscribirse a `config/{device_id}/#` y `bloomfilter/{country_code}/#`
  - ACL aplicada por EMQX usando el campo CN del certificado como `device_id`
  - Upload Service valida la identidad del dispositivo vía certificado o mediante JWT de corta duración emitido por Vault (flujo alternativo cuando mTLS a object storage no es factible)

- **Seguridad multi-tenant:**
  - Realm por país como barrera dura: usuarios del realm "CO" no pueden autenticarse contra el realm "MX"
  - El claim `country_code` en el JWT es fijado por Keycloak desde la configuración del realm — los usuarios no pueden asignarse un país diferente
  - Los servicios internos confían en el `country_code` propagado por el API Gateway; no revalidan de forma independiente
  - Auditoría: todos los eventos de autenticación (login, refresh, fallo, revocación) exportados al log centralizado con retención de 7 años

- **Documentación de despliegue y operaciones:**
  - Helm charts para Keycloak + PostgreSQL y para Vault; mismos manifiestos para cloud y on-prem mediante values distintos
  - ADRs específicos: elección de Vault como PKI de dispositivos, modelo realm-por-país en Keycloak, estrategia de caché en SPI custom, modelo de auto-unseal hexagonal
  - Runbooks: incorporación de un nuevo país (onboarding de realm), bootstrap de primer dispositivo, revocación de certificado de dispositivo, rotación de CA intermedia, failover de Keycloak y Vault

**Excluye:**

- La implementación de los adaptadores de federación SOAP/REST de cada país (son artefactos de código que viven en repositorios de implementación, fuera de este repo de documentación); este change define el contrato y la arquitectura del SPI
- La implementación concreta de los clientes del API Gateway para validar el JWT (`api-frontend-analitica`, propuesto): ese change consume el JWKS endpoint definido aquí
- La configuración operacional de EMQX más allá de los parámetros mTLS y ACL (responsabilidad de `ingestion-mqtt`, aprobado): este change define el esquema de certificados y la política de ACL como contrato
- Los adaptadores de sincronización de vehículos hurtados por país (`sincronizacion-paises`, aprobado): los service accounts usados por esos adaptadores se documentan como entidades Keycloak, pero la lógica de sincronización no es parte de este change
- Los schemas de almacenamiento de datos y el uso del claim `country_code` como discriminador de tenant en los cuatro almacenes (`almacenamiento-lectura`, aprobado): ese change define el mecanismo de aplicación; este change define el contrato del claim
- Servicios de identidad gestionados por nube (Cognito, Azure AD B2C, Okta) como solución principal: pueden usarse detrás del adaptador hexagonal `IdentityProviderPort` pero no son la decisión de arquitectura base

## Justificación

Sin una capa de identidad federada, cada servicio interno debería integrarse directamente con cada IdP policial de cada país, multiplicando el número de integraciones por el número de países y creando un acoplamiento prohibitivo. Keycloak como broker normaliza esa complejidad: los servicios internos solo conocen JWT/OIDC estándar, mientras que la heterogeneidad de fuentes de autenticación queda encapsulada en los proveedores de federación por país. Los changes aprobados `agente-borde`, `ingestion-mqtt`, `almacenamiento-lectura` y `backbone-procesamiento` dependen de contratos definidos aquí — certificados de dispositivo para mTLS, claims JWT para RBAC y discriminación de tenant — que deben estar especificados antes de que `api-frontend-analitica` pueda completar su diseño y cualquier implementación pueda comenzar. Este change cierra ese bloqueo arquitectural.

## Restricciones

- **Cloud-agnostic:** Keycloak en Kubernetes (mismo despliegue en cloud u on-prem); Vault en Kubernetes con Raft storage; ningún servicio de identidad gestionado por nube sin adaptador hexagonal
- **Vault auto-unseal hexagonal (ADR-005):** el mecanismo de unseal debe poder intercambiarse entre cloud KMS y HSM on-prem sin cambios en la lógica de la aplicación; adaptador `VaultUnsealPort` define el contrato
- **Zero secrets en git:** todos los secretos (credenciales de BD, bind LDAP, client secrets de Keycloak, credenciales de Vault) inyectados vía Vault Agent sidecar; prohibido almacenar secretos en variables de entorno en repositorios de código
- **Aislamiento de realm obligatorio (ADR-011):** un usuario de un país no puede obtener tokens válidos para otro país bajo ninguna circunstancia; la separación de realm es la barrera técnica que garantiza esto
- **TTL de certificado de dispositivo:** 90 días máximo; la renovación automática debe iniciarse con al menos 14 días de antelación para tolerar períodos de desconexión del agente de borde (el agente soporta hasta 4 h sin red según `agente-borde`)
- **Retención de auditoría de autenticación:** 7 años, consistente con la política de `audit_log` en `almacenamiento-lectura`
- **Caché SPI custom:** máximo 5 minutos para no mantener credenciales comprometidas activas más de ese tiempo si la fuente upstream las invalida
- **Sin write-back a IdPs externos:** la federación LDAP y JDBC es de solo lectura; Keycloak no modifica directorios ni bases de datos policiales externas
