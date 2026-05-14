# Federación JDBC — User Storage SPI

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Descripción general

El User Storage SPI nativo de Keycloak con proveedor JDBC permite federar usuarios almacenados en bases de datos SQL relacionales. Esta modalidad aplica a los países cuya institución policial gestiona credenciales en una base de datos propia (MySQL, PostgreSQL, Oracle, SQL Server), sin directorio LDAP.

El SPI JDBC de Keycloak ejecuta queries SQL configurables para autenticar usuarios, buscar por username y listar usuarios. No modifica ningún dato en la base de datos del país.

---

## 2. Parámetros de configuración del proveedor JDBC

| Parámetro | Descripción | Ejemplo |
|---|---|---|
| `jdbcUrl` | URL de conexión JDBC | `jdbc:mysql://{host}:3306/policia_mx?useSSL=true&requireSSL=true` |
| `driverClass` | Clase del driver JDBC | Ver tabla de compatibilidad |
| `dbUser` | Usuario de la base de datos | `svc_keycloak_ro` |
| `dbPassword` | Contraseña del usuario | Obtenida de Vault: `vault kv get secret/keycloak/jdbc-mx/db-password` |
| `validationQuery` | Query de validación de conexión | `SELECT 1` (MySQL/PostgreSQL) o `SELECT 1 FROM DUAL` (Oracle) |
| `connectionPool.maxSize` | Tamaño máximo del pool de conexiones | `20` |
| `connectionPool.minIdle` | Conexiones mínimas en idle | `5` |
| `connectionPool.connectionTimeout` | Timeout de obtención de conexión (ms) | `30000` |
| `connectionPool.queryTimeout` | Timeout de query (ms) | `10000` |

### Credenciales en Vault

```bash
# Almacenar credencial de la BD del país en Vault KV v2
vault kv put secret/keycloak/jdbc-mx/db-password \
  password="<db_password>"

# La credencial es inyectada por Vault Agent sidecar en /vault/secrets/jdbc-mx-db-password
# Keycloak la lee desde configuración del provider o variable de entorno
```

---

## 3. Queries configurables

Las tres queries que el SPI JDBC ejecuta son completamente configurables:

### 3.1 Query de autenticación

Ejecutada para verificar las credenciales de un usuario. Debe devolver al menos una fila si las credenciales son válidas.

```sql
SELECT u.id
FROM usuarios u
WHERE u.username = ?
  AND u.password_hash = ?
  AND u.activo = 1
```

> **Nota:** el parámetro `?` para el password puede requerir transformación según el algoritmo de hash usado (ver §Compatibilidad con algoritmos de hash legacy).

### 3.2 Query de búsqueda por username

Ejecutada para obtener los atributos del usuario a mapear en Keycloak.

```sql
SELECT
    u.id            AS id,
    u.username      AS username,
    u.email         AS email,
    u.nombre        AS given_name,
    u.apellido      AS family_name,
    u.perfil        AS role,
    u.zona          AS zone
FROM usuarios u
WHERE u.username = ?
```

### 3.3 Query de listado de usuarios (paginado)

Ejecutada para poblar la vista de usuarios en la consola de administración de Keycloak.

```sql
SELECT
    u.id,
    u.username,
    u.email,
    u.nombre,
    u.apellido,
    u.perfil,
    u.zona
FROM usuarios u
WHERE u.activo = 1
ORDER BY u.username
LIMIT ? OFFSET ?
```

---

## 4. Compatibilidad de drivers JDBC

| Base de datos | Driver JDBC | `driverClass` |
|---|---|---|
| PostgreSQL | `postgresql-42.x.jar` | `org.postgresql.Driver` |
| MySQL / MariaDB | `mysql-connector-j-8.x.jar` | `com.mysql.cj.jdbc.Driver` |
| Oracle | `ojdbc11.jar` | `oracle.jdbc.OracleDriver` |
| SQL Server | `mssql-jdbc-12.x.jar` | `com.microsoft.sqlserver.jdbc.SQLServerDriver` |

Los drivers que no vienen incluidos en la imagen oficial de Keycloak se añaden al classpath mediante la configuración de providers en el Helm chart (montaje de JAR en `/opt/keycloak/providers/`).

---

## 5. Mapeo de columnas a atributos Keycloak y claims JWT

| Columna SQL | Atributo Keycloak | Claim JWT | Notas |
|---|---|---|---|
| `id` | `FEDERATION_ID` | — | Identificador único en la BD; no se expone en JWT |
| `username` | `username` | `preferred_username` | Nombre de usuario |
| `email` | `email` | `email` | Correo electrónico |
| `nombre` / `given_name` | `firstName` | `given_name` | Nombre; el nombre de columna varía por BD |
| `apellido` / `family_name` | `lastName` | `family_name` | Apellido |
| `perfil` / `role` | `role` | `role` | Rol en el sistema; debe mapearse a los cinco roles canónicos |
| `zona` / `zone` | `zone` | `zone` | Zona geográfica |

---

## 6. Pool de conexiones y timeouts

El SPI JDBC de Keycloak usa un pool de conexiones interno. Consideraciones para bases de datos de países con capacidad limitada:

- **`connectionPool.maxSize`:** limitar según la capacidad del servidor de la BD del país. Un valor de 20 conexiones máximas es adecuado para la mayoría de los casos.
- **`connectionPool.connectionTimeout`:** 30 s es el máximo recomendado. Si la BD tarda más de 30 s en responder, la autenticación falla con error de timeout.
- **`connectionPool.queryTimeout`:** 10 s. Las queries de autenticación deben completarse en menos de 10 s; si no, hay un problema de índices en la BD del país que debe resolverse.
- **SSL obligatorio:** todas las conexiones JDBC usan SSL/TLS. Los parámetros `useSSL=true&requireSSL=true` deben estar presentes en la JDBC URL para MySQL/MariaDB.

---

## 7. Política de solo lectura

El proveedor JDBC de Keycloak se configura en modo **read-only** (`editMode=READ_ONLY`):

- Keycloak nunca ejecuta sentencias `INSERT`, `UPDATE` ni `DELETE` en la base de datos del país.
- Los cambios de contraseña iniciados desde la interfaz de Keycloak son bloqueados por la configuración del proveedor; no se propagan a la BD policial.
- El usuario de base de datos configurado en `dbUser` debe tener únicamente permisos `SELECT` en las tablas de usuarios. Se recomienda crear un usuario dedicado de solo lectura (p.ej. `svc_keycloak_ro`) para garantizar el aislamiento a nivel de BD.

```bash
# Ejemplo de creación de usuario de solo lectura en PostgreSQL
CREATE USER svc_keycloak_ro WITH PASSWORD '<password>';
GRANT CONNECT ON DATABASE policia_mx TO svc_keycloak_ro;
GRANT SELECT ON usuarios TO svc_keycloak_ro;
```

---

## 8. Compatibilidad con algoritmos de hash legacy y recomendación de migración

Algunas bases de datos policiales almacenan contraseñas con algoritmos de hash obsoletos. La política del sistema respecto a estos algoritmos es:

| Algoritmo | Soporte | Recomendación |
|---|---|---|
| **PBKDF2-SHA256** | Soportado nativamente | Sin acción requerida |
| **bcrypt** | Soportado con query custom | Sin acción requerida |
| **SHA-256** | Soportado con query custom | Migración a PBKDF2-SHA256 recomendada en 12 meses |
| **SHA-1** | **Deprecado.** Soportado temporalmente con query custom | Migración obligatoria antes del go-live en producción |
| **MD5** | **Deprecado.** Soportado temporalmente con query custom | Migración obligatoria antes del go-live en producción |

Para los algoritmos SHA-1 y MD5, la query de autenticación debe transformar el password antes de comparar:

```sql
-- Ejemplo para MD5 (solo como solución temporal de migración)
SELECT u.id
FROM usuarios u
WHERE u.username = ?
  AND u.password_hash = MD5(?)
  AND u.activo = 1
```

La migración recomendada es: en el próximo login exitoso del usuario, re-hashear la contraseña con PBKDF2-SHA256 y actualizar el registro en la BD del país (requiere un campo adicional de hash moderno). Este proceso debe coordinarse con la contraparte técnica del país.
