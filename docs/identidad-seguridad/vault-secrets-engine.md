# Vault Secrets Engine — Gestión de Secretos de Infraestructura

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Inventario de secretos gestionados por Vault

### 1.1 KV v2 — Secretos estáticos

| Path en Vault | Servicio consumidor | Contenido | Rotación |
|---|---|---|---|
| `secret/keycloak/db` | Keycloak | `username`, `password` (PostgreSQL backend de Keycloak) | Manual, 90 días |
| `secret/keycloak/ldap-{country_code}/bind-credential` | Keycloak (LDAP SPI) | `password` de la cuenta de servicio LDAP del país | Manual, según política del país |
| `secret/keycloak/jdbc-{country_code}/db-password` | Keycloak (JDBC SPI) | `password` del usuario de la BD del país | Manual, según política del país |
| `secret/keycloak/clients/{client_id}` | Keycloak admin | `client_secret` de clientes confidenciales (sync-adapter, superset, etc.) | Automática, 90 días vía Terraform |
| `secret/keycloak/idp-{country_code}-saml/signing-cert` | Keycloak (SAML IdP) | Certificado público del IdP SAML del país | Al renovar el cert del IdP del país |
| `secret/keycloak/idp-{country_code}-oidc/client-secret` | Keycloak (OIDC IdP) | `client_secret` del cliente OIDC en el IdP del país | Según política del IdP del país |
| `secret/minio/access-key` | Upload Service, servicios internos | `access_key_id`, `secret_access_key` | 90 días |
| `secret/kafka/{service_name}` | Productores/consumidores Kafka | `username`, `password` (SASL/SCRAM) | 90 días |
| `secret/opensearch/{service_name}` | Servicios de búsqueda | `username`, `password` | 90 días |
| `secret/api-keys/{service_name}` | Integraciones externas | `api_key` | Según proveedor |

### 1.2 Database Secrets Engine — Secretos dinámicos

Vault genera credenciales de base de datos con TTL corto mediante el Database Secrets Engine. Esto elimina credenciales estáticas de larga duración.

| Mount | Base de datos | Lease (aplicaciones) | Lease (migraciones) |
|---|---|---|---|
| `database/keycloak` | PostgreSQL (Keycloak backend) | `24h` | `1h` |
| `database/events` | PostgreSQL (eventos operacionales) | `24h` | `1h` |
| `database/clickhouse` | ClickHouse | `24h` | `1h` |

```bash
# Configurar el Database Secrets Engine para PostgreSQL de eventos
vault write database/config/events-pg \
  plugin_name=postgresql-database-plugin \
  allowed_roles="events-app,events-migration" \
  connection_url="postgresql://{{username}}:{{password}}@events-pg:5432/events?sslmode=require" \
  username="vault_admin" \
  password="${VAULT_PG_PASSWORD}"

# Rol para aplicaciones (24h)
vault write database/roles/events-app \
  db_name=events-pg \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="24h" \
  max_ttl="24h"

# Rol para migraciones (1h)
vault write database/roles/events-migration \
  db_name=events-pg \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="1h"
```

---

## 2. Vault Agent Sidecar — inyección de secretos

**Regla fundamental:** los secretos nunca se inyectan como variables de entorno. Siempre se escriben como archivos en `/vault/secrets/`.

### 2.1 Configuración del template de Vault Agent

```hcl
# vault-agent-config.hcl — ejemplo para el servicio de eventos
vault {
  address = "https://vault.sistema.com"
}

auto_auth {
  method "kubernetes" {
    mount_path = "auth/kubernetes"
    config = {
      role = "events-service"
    }
  }
}

template {
  source      = "/vault/templates/events-db.ctmpl"
  destination = "/vault/secrets/events-db"
  perms       = 0400
  command     = "kill -HUP 1"   # recarga el proceso del servicio al rotar secretos
}
```

```
{{- with secret "database/creds/events-app" -}}
DB_URL=postgresql://{{ .Data.username }}:{{ .Data.password }}@events-pg:5432/events?sslmode=require
{{- end }}
```

El servicio lee la configuración desde `/vault/secrets/events-db` como archivo de configuración, no como variable de entorno de shell.

### 2.2 Inyección mediante anotaciones de Kubernetes

```yaml
# Anotaciones en el pod para activar el Vault Agent sidecar
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "events-service"
  vault.hashicorp.com/agent-inject-secret-events-db: "database/creds/events-app"
  vault.hashicorp.com/agent-inject-template-events-db: |
    {{- with secret "database/creds/events-app" -}}
    DB_URL=postgresql://{{ .Data.username }}:{{ .Data.password }}@events-pg:5432/events
    {{- end }}
```

---

## 3. Tabla de políticas: servicio → secretos accesibles

| Servicio | Política Vault | Secretos accesibles |
|---|---|---|
| `keycloak` | `keycloak-policy` | `secret/keycloak/db`, `database/creds/keycloak-app`, `secret/keycloak/ldap-*/bind-credential`, `secret/keycloak/idp-*` |
| `sync-adapter-{country_code}` | `sync-adapter-policy` | `secret/keycloak/clients/sync-adapter-{cc}`, `secret/kafka/sync-adapter-{cc}` |
| `upload-service` | `upload-service-policy` | `secret/minio/access-key`, `secret/keycloak/clients/upload-service` |
| `enrichment-service` | `enrichment-policy` | `database/creds/events-app`, `secret/opensearch/enrichment` |
| `matcher-service` | `matcher-policy` | `secret/kafka/matcher`, `secret/redis/matcher` |
| `alert-service` | `alert-policy` | `secret/kafka/alert`, `database/creds/events-app` |
| `search-service` | `search-policy` | `secret/opensearch/search`, `database/creds/events-app` |
| `analytics-service` | `analytics-policy` | `database/creds/events-app`, `secret/clickhouse/analytics` |
| `api-gateway` | `api-gateway-policy` | `secret/keycloak/clients/api-gateway` |

---

## 4. Autenticación de servicios internos con Vault

### 4.1 AppRole — para servicios fuera de Kubernetes

Servicios que no corren en Kubernetes (scripts de inicialización, operadores externos) usan AppRole:

```bash
# Crear AppRole para un servicio
vault auth enable approle
vault write auth/approle/role/sync-adapter-co \
  secret_id_ttl=10m \
  token_num_uses=1 \
  token_ttl=20m \
  token_max_ttl=30m \
  policies="sync-adapter-policy"

# Obtener role_id (estático, se puede almacenar en la configuración del servicio)
vault read auth/approle/role/sync-adapter-co/role-id

# Obtener secret_id (dinámico, un solo uso, se genera por cada inicio del servicio)
vault write -f auth/approle/role/sync-adapter-co/secret-id
```

### 4.2 Kubernetes auth — para pods en el cluster

```bash
# Configurar el método de autenticación Kubernetes
vault auth enable kubernetes
vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Crear rol para el servicio de eventos
vault write auth/kubernetes/role/events-service \
  bound_service_account_names=events-service \
  bound_service_account_namespaces=prod \
  policies=events-policy \
  ttl=1h
```

Los pods usan el service account de Kubernetes como credencial para obtener tokens de Vault, sin necesidad de gestionar secretos adicionales para la autenticación con Vault.

---

## 5. Comportamiento ante indisponibilidad de Vault (fail-closed)

Cuando Vault entra en estado de fallo (todos los nodos sellados, caídos o partición de red), el comportamiento del sistema es **fail-closed**:

### Pods activos (secretos ya inyectados)

- Los pods que ya están corriendo **continúan funcionando** con los secretos que Vault Agent sidecar inyectó previamente como archivos en `/vault/secrets/`.
- Los secretos de tipo KV son estáticos (se inyectan una vez al arrancar el pod); no requieren reconexión a Vault durante la vida del pod.
- Las credenciales dinámicas del Vault Database Secrets Engine (lease de 24 h) **expiran** cuando Vault no puede renovarlas. Cuando el lease expira, la conexión a la BD falla; el pod debe reiniciarse para obtener nuevas credenciales al recuperarse Vault.

### Nuevos pods (fail-closed)

- El init-container de Vault Agent **bloquea el arranque** del pod hasta obtener los secretos de Vault.
- Si Vault no responde, el init-container falla repetidamente; Kubernetes aplica CrashLoopBackOff al pod.
- **Ningún pod nuevo arranca** mientras Vault está caído. Esto garantiza que ningún servicio se inicie sin secretos válidos (política fail-closed).

### Alertas de monitoreo

La indisponibilidad de Vault dispara alertas automáticas monitoreando las métricas de Vault:

```bash
# Verificar estado de disponibilidad (retorna 200 si activo y unsealed)
curl -s "${VAULT_ADDR}/v1/sys/health" | jq '{initialized, sealed, standby}'

# Métricas clave a monitorear
# vault.core.unsealed == 1  → Vault disponible
# vault.raft.leader       → nodo líder activo en el cluster
```

La alerta debe activarse si `vault.core.unsealed == 0` por más de 60 s en cualquier nodo, y notificar al canal de operaciones correspondiente.

### Recuperación

Cuando Vault se recupera (unseal automático vía KMS o manual con Shamir):

1. Los pods activos con credenciales dinámicas expiradas se reinician y obtienen nuevas credenciales.
2. Los nuevos pods pueden arrancar normalmente; el init-container de Vault Agent obtiene los secretos.
3. No se requiere intervención manual en los pods (Kubernetes gestiona el reinicio vía liveness/readiness probes).
