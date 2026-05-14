# Onboarding de un Nuevo País

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## Descripción general

Este documento describe el proceso ordenado para incorporar un nuevo país al sistema de identidad federada. El proceso aplica tanto al piloto inicial como a la incorporación de países adicionales en fases posteriores.

**Tiempo estimado por tipo de IdP:**

| Tipo de IdP del país | Tiempo estimado |
|---|---|
| LDAP / Active Directory | 2–4 días hábiles |
| Base de datos SQL (JDBC) | 2–4 días hábiles |
| SAML 2.0 | 3–5 días hábiles (incluye coordinación con el IdP del país) |
| OIDC | 2–4 días hábiles |
| SOAP/REST custom (SPI) | 5–10 días hábiles (incluye desarrollo del adaptador) |

---

## Pasos de onboarding

### Paso 1 — Crear el realm en Keycloak

**Responsable:** Operador de infraestructura de identidad
**Herramienta:** Keycloak Admin API / Terraform

```bash
# Via Terraform (recomendado para reproducibilidad)
# Añadir el nuevo país a la variable countries en el módulo keycloak
# terraform/main.tf:
# countries = ["CO", "MX", "AR", "VE"]  ← añadir "VE"

terraform plan -var='countries=["CO","MX","AR","VE"]'
terraform apply -var='countries=["CO","MX","AR","VE"]'
```

Verificar que el realm fue creado:

```bash
curl -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  "${KEYCLOAK_URL}/admin/realms/VE" | jq '.id'
```

---

### Paso 2 — Configurar clientes en el realm con secretos en Vault

**Responsable:** Operador de infraestructura de identidad
**Herramienta:** Terraform / Keycloak Admin API + Vault CLI

```bash
# Generar y almacenar el client_secret del adaptador de sincronización
CLIENT_SECRET=$(openssl rand -hex 32)
vault kv put secret/keycloak/clients/sync-adapter-ve \
  client_secret="${CLIENT_SECRET}"

# Terraform aplica la creación del cliente confidencial en el realm VE
```

---

### Paso 3 — Configurar el proveedor de federación

**Responsable:** Operador de infraestructura de identidad + Contraparte técnica del país
**Herramienta:** Keycloak Admin API / Terraform

Según el tipo de IdP del país:

- **LDAP:** ver [federation-ldap.md](./federation-ldap.md) — configurar `connectionUrl`, `bindDn`, `usersDn`.
- **JDBC:** ver [federation-jdbc.md](./federation-jdbc.md) — configurar `jdbcUrl`, queries.
- **SAML/OIDC:** ver [federation-saml-oidc.md](./federation-saml-oidc.md) — configurar endpoints, certificados.
- **SOAP/REST custom:** ver [federation-spi-custom.md](./federation-spi-custom.md) — desarrollar el adaptador, empaquetar el JAR.

---

### Paso 4 — Verificar el mapeo de atributos

**Responsable:** Operador de infraestructura de identidad + Contraparte técnica del país
**Herramienta:** Keycloak Admin Console / usuario de prueba del país

Verificar con un usuario de prueba que:

1. El usuario se autentica correctamente con las credenciales del sistema del país.
2. El claim `country_code` en el JWT tiene el valor correcto (ej: `"VE"`).
3. El claim `role` mapea correctamente a uno de los cinco roles canónicos.
4. El claim `zone` contiene un valor válido para el país.
5. Los claims `given_name`, `family_name`, `email` se populan si el IdP los provee.

```bash
# Obtener un token de prueba usando el service account sync-adapter-VE (client_credentials)
# Nota: web-app usa PKCE obligatorio y no soporta ROPC. Para verificación automática,
# usar el flujo client_credentials con el service account del adaptador de sincronización.
SYNC_SECRET=$(vault kv get -field=client_secret secret/keycloak/VE/sync-adapter-ve)

TOKEN=$(curl -s -X POST "${KEYCLOAK_URL}/realms/VE/protocol/openid-connect/token" \
  -d "grant_type=client_credentials&client_id=sync-adapter-ve&client_secret=${SYNC_SECRET}" \
  | jq -r '.access_token')

# Decodificar y verificar los claims del JWT
echo "${TOKEN}" | cut -d. -f2 | base64 -d 2>/dev/null | jq '{country_code, role, zone, sub, iss}'
```

> **Nota:** La verificación del flujo de usuario humano (`web-app`) debe realizarse manualmente en el navegador mediante el flujo Authorization Code + PKCE. El flujo ROPC (`grant_type=password`) está deshabilitado en `web-app` por diseño de seguridad (cliente público con PKCE obligatorio).

---

### Paso 5 — Crear la CA intermedia en Vault

**Responsable:** Operador de PKI / SRE senior
**Herramienta:** Vault CLI

```bash
# Montar el nuevo mount de PKI para el país
vault secrets enable -path=pki-ve pki
vault secrets tune -max-lease-ttl=43800h pki-ve

# Generar CSR de la CA intermedia del nuevo país
vault write pki-ve/intermediate/generate/internal \
  common_name="Sistema Anti-Hurto Vehicles Intermediate CA VE" \
  key_type=rsa key_bits=4096 \
  -format=json | jq -r '.data.csr' > pki-ve.csr

# Firmar con la root CA
vault write pki-root/root/sign-intermediate \
  csr=@pki-ve.csr \
  common_name="Sistema Anti-Hurto Vehicles Intermediate CA VE" \
  ttl=43800h \
  -format=json | jq -r '.data.certificate' > pki-ve-signed.crt

# Importar el certificado firmado
vault write pki-ve/intermediate/set-signed certificate=@pki-ve-signed.crt

# Crear el rol de emisión para dispositivos del nuevo país
vault write pki-ve/roles/device-role \
  allow_any_name=true \
  enforce_hostnames=false \
  key_type="rsa" key_bits=2048 \
  ttl="2160h" max_ttl="2160h"
# Nota: allow_any_name=true es necesario porque CN=device:{device_id} contiene ':'
# y no es un hostname RFC 5280 válido. Ver vault-pki-device.md §5.
```

---

### Paso 6 — Emitir el certificado de CA firmado por la root CA

(Completado en el paso anterior como parte de la configuración del mount PKI.)

Verificar:

```bash
# Verificar que la CA intermedia está correctamente firmada
curl -s "${VAULT_ADDR}/v1/pki-ve/ca/pem" | openssl x509 -noout -subject -issuer
```

---

### Paso 7 — Registrar la CA del nuevo país en EMQX

**Responsable:** Operador de infraestructura
**Herramienta:** kubectl / Helm

La CA intermedia del nuevo país debe añadirse al bundle de CAs de EMQX:

```bash
# Descargar el certificado de la CA intermedia del nuevo país
vault read -field=certificate pki-ve/cert/ca > pki-ve-ca.pem

# Añadir al ConfigMap de EMQX que contiene el bundle de CAs
# El bundle se regenera y el pod de EMQX se reinicia con rolling update
kubectl create configmap emqx-ca-bundle \
  --from-file=ca-bundle.pem=./ca-bundle-updated.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

### Paso 8 — Bootstrap del primer dispositivo del nuevo país

**Responsable:** Operador de campo + Operador de PKI
**Herramienta:** Vault CLI + agente de borde

Seguir el proceso descrito en [vault-pki-device.md](./vault-pki-device.md) §2. Para el primer dispositivo del nuevo país:

```bash
# Crear bootstrap token para el primer dispositivo de Venezuela
vault token create \
  -policy="bootstrap-ve" \
  -use-limit=1 \
  -ttl=24h \
  -display-name="bootstrap-device-ve-001" \
  -format=json | jq -r '.auth.client_token'
```

---

### Paso 9 — Verificar el flujo de autenticación end-to-end

**Responsable:** QA / Operador de infraestructura
**Herramienta:** curl, mqtt-cli, navegador

1. Verificar autenticación de usuario humano: login en `web-app` con credenciales del IdP del país → JWT con claims correctos.
2. Verificar que el API Gateway acepta el token y aplica RBAC correctamente.
3. Verificar autenticación del dispositivo: bootstrap → cert → conexión mTLS a EMQX → publicación en tópico permitido (`devices/{device_id}/events`).
4. Verificar rechazo de tópico no permitido (ACL): el dispositivo intenta publicar en `devices/{otro_device_id}/events` y debe recibir `PUBACK` con código de error `0x87 Not Authorized` (MQTT 5) o ser desconectado (MQTT 3.1.1); el rechazo debe aparecer en los logs de EMQX con el CN del dispositivo y el tópico denegado. Si en cambio se recibe un error TLS en el handshake (antes de cualquier CONNECT), indica que el bundle de CAs de EMQX no se actualizó correctamente en el paso 7 — revisar el ConfigMap `emqx-ca-bundle`.

---

### Paso 10 — Habilitar la exportación de eventos de auditoría

**Responsable:** Operador de infraestructura
**Herramienta:** Keycloak Admin API

```bash
# Configurar el Event Listener SPI de auditoría para el nuevo realm
curl -X PUT \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"eventsEnabled": true, "eventsExpiration": 0, "eventsListeners": ["jboss-logging", "audit-opensearch-exporter"]}' \
  "${KEYCLOAK_URL}/admin/realms/VE/events/config"
```

---

### Paso 11 — Actualizar la documentación del país

**Responsable:** Operador / Technical writer
**Herramienta:** git + editor

Documentar en el repositorio:

- Tipo de IdP y parámetros de federación del país.
- Mapeo de atributos específico del país.
- Contacto técnico de contraparte del país.
- Cualquier desviación de la configuración estándar.
