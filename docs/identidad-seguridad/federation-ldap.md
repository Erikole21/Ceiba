# Federación LDAP — User Storage SPI

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Descripción general

El User Storage SPI nativo de Keycloak permite federar usuarios desde un directorio LDAP (Active Directory, OpenLDAP, etc.) sin necesidad de importación previa. Keycloak consulta el LDAP en tiempo de autenticación y mapea los atributos del directorio a los claims canónicos del JWT del sistema.

Esta federación aplica a los países cuya institución policial usa un directorio LDAP o Active Directory como fuente de identidad (caso Colombia — Policía Nacional, entre otros).

---

## 2. Parámetros de configuración del proveedor LDAP

| Parámetro Keycloak | Descripción | Ejemplo |
|---|---|---|
| `vendor` | Tipo de servidor LDAP | `Active Directory` o `Other` (OpenLDAP) |
| `connectionUrl` | URL de conexión al LDAP | `ldaps://ldap.policia.co:636` |
| `bindDn` | Distinguished Name de la cuenta de servicio de lectura | `cn=svc-keycloak,ou=ServiceAccounts,dc=policia,dc=co` |
| `bindCredential` | Contraseña de la cuenta de servicio | Obtenida de Vault: `vault kv get secret/keycloak/ldap-co/bind-credential` |
| `usersDn` | DN base para búsqueda de usuarios | `ou=Users,dc=policia,dc=co` |
| `userObjectClasses` | Clases de objeto para filtrar usuarios | `person, organizationalPerson, user` |
| `usernameLdapAttribute` | Atributo LDAP que corresponde al username de Keycloak | `sAMAccountName` (AD) o `uid` (OpenLDAP) |
| `uuidLdapAttribute` | Atributo LDAP que se usa como identificador único | `objectGUID` (AD) o `entryUUID` (OpenLDAP) |
| `searchScope` | Profundidad de búsqueda | `2` (SUBTREE) |
| `pagination` | Paginación LDAP para directorios grandes | `true`, `pageSize=500` |
| `readTimeout` | Timeout de operación LDAP en milisegundos | `10000` |
| `connectionPooling` | Pool de conexiones LDAP | `true` |

### Credenciales en Vault

La contraseña del `bindDn` nunca se almacena en texto claro en la configuración de Keycloak. Se inyecta mediante Vault Agent sidecar y se referencia desde la configuración del proveedor LDAP:

```bash
# Guardar credencial en Vault
vault kv put secret/keycloak/ldap-co/bind-credential \
  password="<bind_password>"

# Vault Agent inyecta la credencial en /vault/secrets/ldap-co-bind
# Keycloak la lee desde variables de entorno o archivos según la configuración del provider
```

---

## 3. Estrategia de sincronización

| Modo | Descripción | Uso recomendado |
|---|---|---|
| **On-demand (lazy loading)** | El usuario se importa a Keycloak la primera vez que se autentica. Los atributos se actualizan en cada autenticación. | **Recomendado para la mayoría de los casos.** Minimiza la carga inicial y mantiene los datos frescos. |
| **Periodic full sync** | Importa todos los usuarios del directorio LDAP periódicamente (por defecto cada 24 h). | Útil si se necesitan buscar usuarios en la consola de administración antes de que se hayan autenticado. |
| **Periodic changed users sync** | Importa solo los usuarios modificados desde la última sincronización. | Complemento al full sync para mantener datos actualizados con menor carga. |

**Justificación de on-demand como modo primario:** dado que la fuente de verdad para los atributos de usuario es el directorio LDAP, la importación periódica introduce una latencia de actualización innecesaria. Con on-demand, los atributos siempre reflejan el estado actual del directorio en el momento de la autenticación. La importación periódica se configura adicionalmente para soporte de búsqueda administrativa.

---

## 4. Mapeo de atributos canónico

| Atributo LDAP | Atributo Keycloak | Claim JWT | Tipo | Descripción |
|---|---|---|---|---|
| `sAMAccountName` / `uid` | `username` | `preferred_username` | `string` | Identificador de usuario |
| `mail` | `email` | `email` | `string` | Correo electrónico |
| `givenName` | `firstName` | `given_name` | `string` | Nombre |
| `sn` (surname) | `lastName` | `family_name` | `string` | Apellido |
| `ldapRole` / `extensionAttribute1` | `role` | `role` | `string` | Rol en el sistema (officer, supervisor, etc.) |
| `ldapCountry` / `extensionAttribute2` | `country_code_ldap` | — | `string` | Atributo del directorio; el claim `country_code` se fija por realm, no desde este atributo |
| `department` | `zone` | `zone` | `string` | Zona geográfica asignada al usuario |
| `objectGUID` / `entryUUID` | `LDAP_ENTRY_UUID` | — | `string` | Identificador único del directorio; no se expone en JWT |

> **Nota sobre `country_code`:** el atributo LDAP del país es informativo y puede mapearse a un atributo de usuario de Keycloak para auditoría, pero el claim `country_code` en el JWT siempre se fija por la configuración del realm (Hardcoded claim mapper). Esto garantiza que un usuario del directorio LDAP no pueda obtener un `country_code` diferente al del realm en el que se autentica.

---

## 5. Configuración SSL/TLS

- La conexión al LDAP siempre usa **LDAPS** (puerto 636) con TLS 1.2 o superior.
- El certificado del servidor LDAP debe estar firmado por una CA reconocida o importada al truststore de Keycloak.
- Se rechaza cualquier certificado inválido, expirado o autofirmado no importado explícitamente.
- Configuración del truststore en la imagen Docker de Keycloak:

```bash
# Importar certificado CA del LDAP al truststore de Keycloak
keytool -importcert \
  -keystore /opt/keycloak/conf/truststore.jks \
  -storepass changeit \
  -alias ldap-ca-co \
  -file /vault/secrets/ldap-ca-co.crt \
  -noprompt
```

---

## 6. Política de solo lectura

El proveedor LDAP de Keycloak se configura en modo **read-only** (`editMode=READ_ONLY`):

- Keycloak nunca escribe ni modifica atributos en el directorio LDAP del país.
- Los cambios de contraseña realizados desde la interfaz de Keycloak no se propagan al LDAP (esto previene modificaciones no autorizadas en el directorio policial).
- Si un país requiere gestión de contraseñas desde el sistema, se implementa un flujo específico con aprobación explícita de la contraparte técnica del país.

---

## 7. Comportamiento ante fallo del LDAP

Keycloak gestiona el fallo del LDAP según la configuración del proveedor:

1. **Usuarios importados con caché:** si el usuario fue importado previamente y tiene datos en la base de datos de Keycloak (`editMode=READ_ONLY` + `importEnabled=true`), Keycloak puede autenticar con los datos cacheados. La contraseña sigue siendo validada contra el LDAP (si el LDAP no responde, la autenticación falla).
2. **Sin caché de contraseñas:** Keycloak no almacena contraseñas de los usuarios LDAP. Si el LDAP no está disponible, la autenticación falla con error `LDAP_CONNECTION_ERROR` registrado en el event log.
3. **Failover entre nodos LDAP:** si el Active Directory tiene múltiples Domain Controllers, se configura la URL de conexión con múltiples hosts:

```
ldaps://dc1.policia.co:636 ldaps://dc2.policia.co:636
```

Keycloak intenta el primer host y, si falla, pasa al siguiente.

---

## 8. Comportamiento ante atributo de rol ausente en LDAP

Si el atributo LDAP mapeado al claim `role` (p.ej. `extensionAttribute1` → `role`) está ausente o vacío en la entrada del usuario, Keycloak aplica la siguiente política:

1. La autenticación de credenciales puede completarse exitosamente (el LDAP valida el `bind` con username/password).
2. Al intentar emitir el token de acceso, Keycloak detecta que el mapper de atributo no pudo poblar el claim `role` (atributo LDAP ausente o vacío).
3. Keycloak rechaza la emisión del token con error `MISSING_ROLE_CLAIM` registrado en el event log del realm.
4. No se emite ningún token parcial. El cliente recibe HTTP 401 con indicación de atributo obligatorio ausente.
5. El operador debe revisar la entrada LDAP del usuario y asegurar que el atributo mapeado a `role` contiene uno de los cinco valores canónicos: `officer`, `supervisor`, `analyst`, `admin`, `auditor`.

> **Configuración requerida:** el mapper de atributo en Keycloak debe tener `Is Mandatory = ON` para el atributo `role`. Con esta configuración, Keycloak garantiza que ningún token sin `role` sea emitido.

---

## 9. Verificación de la federación

```bash
# Verificar conexión LDAP desde la consola de administración de Keycloak
# POST /admin/realms/CO/components/{ldap_provider_id}/test-connection

# Listar usuarios federados (importados on-demand)
# GET /admin/realms/CO/users?search=jperez

# Forzar sincronización completa
# POST /admin/realms/CO/user-storage/{ldap_provider_id}/sync?action=triggerFullSync
```
