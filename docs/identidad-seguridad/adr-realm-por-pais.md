# ADR — Realm por País en Keycloak como Barrera de Aislamiento

**ID:** ADR-realm-por-pais
**Estado:** Aprobado
**Fecha:** 2026-05-13
**Relacionado con:** ADR-011 (Multi-tenant por país)

---

## Contexto

El sistema opera en múltiples países simultáneamente. Cada país tiene su propia base de usuarios policiales, su propia fuente de identidad (LDAP, base de datos relacional, SOAP/REST, SAML/OIDC) y sus propios requisitos de soberanía de datos y cumplimiento regulatorio.

Keycloak soporta múltiples modelos de organización de usuarios. La elección del modelo tiene consecuencias técnicas y operacionales de largo alcance en:

- Aislamiento de sesiones y tokens entre países.
- Configuración de políticas de contraseñas y MFA por país.
- Federación con fuentes de identidad heterogéneas por país.
- Límite de exposición ante un compromiso de credenciales.

ADR-011 establece que `country_code` es el discriminador de tenant en el sistema. Este ADR extiende esa decisión al plano de identidad: cómo se implementa técnicamente el aislamiento en Keycloak.

---

## Alternativas evaluadas

### Alternativa A — Realm único con grupos por país

Todos los usuarios de todos los países se crean en un único realm de Keycloak. Los países se modelan como grupos o atributos.

**Ventajas:**

- Configuración inicial más simple.
- Claves de firma compartidas.
- SSO transparente entre todos los usuarios.

**Desventajas:**

- Un único punto de configuración: un error en la política de contraseñas afecta a todos los países.
- No existe aislamiento técnico real: un administrador del realm puede ver usuarios de cualquier país.
- La federación a múltiples fuentes de identidad (una por país) se vuelve compleja de gestionar en un solo realm.
- Las sesiones de usuarios de diferentes países comparten el mismo contexto de Keycloak.
- Un compromiso del realm compromete a todos los países.
- Los grupos no son una barrera de aislamiento técnico: son una construcción de autorización, no de autenticación.

**Veredicto:** descartada. Los grupos en un realm único no garantizan aislamiento técnico. Un claim `country_code` basado en grupos puede ser alterado mediante configuración incorrecta o un exploit de la capa de administración.

### Alternativa B — Realm por país (seleccionada)

Cada país tiene su propio realm en Keycloak. Por ejemplo: `CO`, `MX`, `AR`, `VE`, `BR`.

**Ventajas:**

- Aislamiento técnico real: cada realm tiene sus propias claves de firma, sus propios clientes, sus propios usuarios y su propia configuración de federación.
- El claim `country_code` se fija en la configuración del realm y no puede ser sobreescrito por el usuario ni por la fuente de identidad (véase [jwt-claims-schema.md](./jwt-claims-schema.md)).
- Un operador del realm `CO` no tiene visibilidad sobre los usuarios del realm `MX`.
- Cada realm puede tener su propio proveedor de identidad federado, política de contraseñas, configuración de MFA y TTL de sesión.
- Un compromiso de un realm no afecta a otros realms.
- Cumple explícitamente con ADR-011: el realm es la barrera técnica de aislamiento a nivel de identidad.

**Desventajas:**

- Mayor carga operacional: cada nuevo país requiere la creación y configuración de un nuevo realm.
- Keycloak tiene un límite práctico de realms por cluster (ver §Consecuencias).
- No hay SSO entre países por diseño (esto es intencional).

**Veredicto:** seleccionada.

### Alternativa C — Cluster independiente de Keycloak por país

Cada país despliega su propia instancia de Keycloak completamente separada.

**Ventajas:**

- Máximo aislamiento de infraestructura.
- Posibilidad de desplegar en región geográfica del país.

**Desventajas:**

- Costo operacional muy elevado: N clusters que operar, actualizar y monitorear.
- No existe beneficio técnico adicional sobre el modelo realm-por-país para la mayoría de los casos.
- La gestión centralizada de políticas de seguridad (actualizaciones de Keycloak, rotación de claves) se fragmenta.

**Veredicto:** descartada para el caso general. Se puede considerar para países con requisitos regulatorios de residencia de datos estrictos que exijan que incluso el plano de control de identidad resida en la región del país.

---

## Decisión

**Se adopta el modelo realm por país como barrera técnica de aislamiento de identidad, en cumplimiento de ADR-011.**

- Cada país incorporado al sistema recibe un realm dedicado en el cluster de Keycloak.
- El nombre del realm coincide con el código ISO 3166-1 alpha-2 del país (`CO`, `MX`, `AR`, etc.).
- El claim `country_code` se configura como atributo fijo del realm y se inyecta en todos los tokens emitidos, independientemente de la fuente de identidad federada.
- No existe ningún mecanismo de cross-realm federation que permita a un usuario de un realm autenticarse en otro.

### Justificación del rechazo de grupos en realm único

Los grupos de Keycloak son una construcción de autorización (asignación de roles y atributos), no una barrera de autenticación. Un error de configuración, un exploit de la API de administración o una mala asignación de permisos de administrador puede resultar en que un usuario de un país obtenga atributos de otro país. La separación a nivel de realm es técnicamente ineludible: el proceso de emisión de tokens está completamente encapsulado dentro del realm, y no existe ninguna operación de configuración que permita a un realm emitir tokens del contexto de otro realm.

---

## Consecuencias

### Consecuencias positivas

- El claim `country_code` en el JWT es criptográficamente vinculante: está firmado con la clave privada del realm correspondiente y no puede ser alterado sin invalidar la firma.
- Los servicios downstream (`almacenamiento-lectura`, `backbone-procesamiento`, `api-frontend-analitica`) pueden confiar en `country_code` sin validación adicional una vez verificada la firma JWT.
- Incorporar un nuevo país es un proceso operacional delimitado, documentado en [onboarding-nuevo-pais.md](./onboarding-nuevo-pais.md).

### Consecuencias y limitaciones a gestionar

- **Límite de realms por cluster de Keycloak:** el límite práctico documentado en la comunidad de Keycloak es de 50–100 realms por cluster antes de que el rendimiento del servidor de administración se degrade notablemente. Para la escala prevista en Latinoamérica (< 20 países), este límite no es un problema. Si el sistema escala a cobertura global (> 50 países), se evalúa un segundo cluster de Keycloak con los mismos manifiestos Helm.
- **Operación por realm:** cada realm requiere gestión individual: rotación de claves, backup, monitoring de eventos. El proceso de onboarding debe estar automatizado vía Terraform (véase [terraform/README.md](./terraform/README.md)).
- **Sin SSO entre países:** es una consecuencia aceptada. Los usuarios policiales operan en el contexto de su país; el SSO entre países no es un requisito del sistema.

---

## Referencias

- [keycloak-realm-model.md](./keycloak-realm-model.md) — Estructura detallada de realm, clientes y federación
- [jwt-claims-schema.md](./jwt-claims-schema.md) — Regla de fijación del claim `country_code` desde el realm
- [onboarding-nuevo-pais.md](./onboarding-nuevo-pais.md) — Proceso de creación de nuevo realm
- [multitenancy-security.md](./multitenancy-security.md) — Modelo de aislamiento técnico completo
- [terraform/README.md](./terraform/README.md) — Automatización de provisión de realms
