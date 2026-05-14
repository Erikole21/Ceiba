# Terraform — Módulos IaC para Identidad y PKI

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Estructura de módulos

```
terraform/
  identidad-seguridad/
    modules/
      keycloak-realm/         ← realms, clientes, roles, federación por país
      vault-pki/              ← root CA, CAs intermedias, roles de emisión
      vault-secrets/          ← KV v2, políticas, AppRole, Kubernetes auth
    main.tf                   ← orquestación de módulos
    variables.tf
    outputs.tf
    versions.tf
```

---

## 2. Módulo `keycloak-realm`

Usa el provider `mrparkers/keycloak` para gestionar la configuración de Keycloak via Terraform.

```hcl
# modules/keycloak-realm/main.tf

terraform {
  required_providers {
    keycloak = {
      source  = "mrparkers/keycloak"
      version = "~> 4.4"
    }
  }
}

# Crear el realm del país
resource "keycloak_realm" "country_realm" {
  realm                    = var.country_code
  enabled                  = true
  display_name             = "Sistema Anti-Hurto — ${var.country_code}"
  access_token_lifespan    = 300
  sso_session_idle_timeout = 1800
  sso_session_max_lifespan = 36000

  password_policy = "length(12) and upperCase(1) and lowerCase(1) and digits(1) and specialChars(1) and notUsername(undefined) and passwordHistory(5)"
}

# Hardcoded claim mapper para country_code
resource "keycloak_generic_protocol_mapper" "country_code_mapper" {
  realm_id        = keycloak_realm.country_realm.id
  client_scope_id = keycloak_openid_client_scope.base_scope.id
  name            = "country_code"
  protocol        = "openid-connect"
  protocol_mapper = "oidc-hardcoded-claim-mapper"
  config = {
    "claim.name"   = "country_code"
    "claim.value"  = var.country_code
    "jsonType.label" = "String"
    "access.token.claim" = "true"
    "id.token.claim"     = "true"
  }
}

# Roles del realm
resource "keycloak_role" "roles" {
  for_each = toset(["officer", "supervisor", "analyst", "admin", "auditor"])
  realm_id  = keycloak_realm.country_realm.id
  name      = each.key
}

# Herencia: supervisor hereda de officer
resource "keycloak_role" "supervisor_composite" {
  realm_id          = keycloak_realm.country_realm.id
  name              = "supervisor"
  composite_roles   = [keycloak_role.roles["officer"].id]
  depends_on        = [keycloak_role.roles]
}

# Cliente web-app (Authorization Code + PKCE)
resource "keycloak_openid_client" "web_app" {
  realm_id              = keycloak_realm.country_realm.id
  client_id             = "web-app"
  name                  = "Web Application"
  enabled               = true
  access_type           = "PUBLIC"
  standard_flow_enabled = true
  pkce_code_challenge_method = "S256"
  valid_redirect_uris   = ["https://${var.app_domain}/*"]
  web_origins           = ["https://${var.app_domain}"]
}

# Cliente sync-adapter (Client Credentials)
resource "keycloak_openid_client" "sync_adapter" {
  realm_id                     = keycloak_realm.country_realm.id
  client_id                    = "sync-adapter-${lower(var.country_code)}"
  name                         = "Sync Adapter ${var.country_code}"
  enabled                      = true
  access_type                  = "CONFIDENTIAL"
  service_accounts_enabled     = true
  standard_flow_enabled        = false
  client_secret                = data.vault_kv_secret_v2.sync_adapter_secret.data.client_secret
}
```

**Variables del módulo:**

| Variable | Tipo | Descripción |
|---|---|---|
| `country_code` | `string` | Código ISO 3166-1 alpha-2 del país (ej: `"CO"`) |
| `app_domain` | `string` | Dominio de la aplicación web |
| `keycloak_url` | `string` | URL base de Keycloak |
| `federation_type` | `string` | Tipo de federación: `ldap`, `jdbc`, `saml`, `oidc`, `spi` |
| `federation_config` | `map(string)` | Parámetros específicos de la federación |

---

## 3. Módulo `vault-pki`

```hcl
# modules/vault-pki/main.tf

# Mount root CA (solo se ejecuta una vez)
resource "vault_mount" "pki_root" {
  path                      = "pki-root"
  type                      = "pki"
  max_lease_ttl_seconds     = 315360000  # 10 años
}

resource "vault_pki_secret_backend_root_cert" "root_ca" {
  backend     = vault_mount.pki_root.path
  type        = "internal"
  common_name = "Sistema Anti-Hurto Vehicles Root CA"
  ttl         = "315360000"
  key_type    = "rsa"
  key_bits    = 4096
}

# Mount CA intermedia por país (en bucle via for_each)
resource "vault_mount" "pki_intermediate" {
  for_each              = toset(var.countries)
  path                  = "pki-${lower(each.key)}"
  type                  = "pki"
  max_lease_ttl_seconds = 157680000  # 5 años
}

resource "vault_pki_secret_backend_intermediate_cert_request" "csr" {
  for_each    = toset(var.countries)
  backend     = vault_mount.pki_intermediate[each.key].path
  type        = "internal"
  common_name = "Sistema Anti-Hurto Vehicles Intermediate CA ${each.key}"
  key_type    = "rsa"
  key_bits    = 4096
}

resource "vault_pki_secret_backend_root_sign_intermediate" "sign" {
  for_each    = toset(var.countries)
  backend     = vault_mount.pki_root.path
  csr         = vault_pki_secret_backend_intermediate_cert_request.csr[each.key].csr
  common_name = "Sistema Anti-Hurto Vehicles Intermediate CA ${each.key}"
  ttl         = "157680000"
}

resource "vault_pki_secret_backend_intermediate_set_signed" "set_signed" {
  for_each    = toset(var.countries)
  backend     = vault_mount.pki_intermediate[each.key].path
  certificate = vault_pki_secret_backend_root_sign_intermediate.sign[each.key].certificate
}

# Rol de emisión para dispositivos
resource "vault_pki_secret_backend_role" "device_role" {
  for_each         = toset(var.countries)
  backend          = vault_mount.pki_intermediate[each.key].path
  name             = "device-role"
  allow_any_name    = true
  enforce_hostnames = false
  allow_subdomains  = false
  # allow_any_name=true es necesario porque CN=device:{device_id} contiene ':' y no
  # es un hostname RFC 5280 válido. La restricción por país se impone via política Vault.
  key_type         = "rsa"
  key_bits         = 2048
  ttl              = "2160h"
  max_ttl          = "2160h"
}

# Política de revocación
resource "vault_pki_secret_backend_config_urls" "config_urls" {
  for_each                 = toset(var.countries)
  backend                  = vault_mount.pki_intermediate[each.key].path
  issuing_certificates     = ["${var.vault_addr}/v1/pki-${lower(each.key)}/ca"]
  crl_distribution_points  = ["${var.vault_addr}/v1/pki-${lower(each.key)}/crl"]
  ocsp_servers             = ["${var.vault_addr}/v1/pki-${lower(each.key)}/ocsp"]
}
```

---

## 4. Módulo `vault-secrets`

```hcl
# modules/vault-secrets/main.tf

# Mount KV v2
resource "vault_mount" "kv" {
  path    = "secret"
  type    = "kv"
  options = { version = "2" }
}

# Política por servicio
resource "vault_policy" "keycloak_policy" {
  name   = "keycloak-policy"
  policy = <<EOT
path "secret/data/keycloak/*" {
  capabilities = ["read"]
}
path "database/creds/keycloak-app" {
  capabilities = ["read"]
}
EOT
}

# AppRole por servicio
resource "vault_approle_auth_backend_role" "sync_adapter" {
  for_each       = toset(var.countries)
  role_name      = "sync-adapter-${lower(each.key)}"
  token_policies = ["sync-adapter-policy"]
  token_ttl      = 3600
  token_max_ttl  = 3600
}

# Kubernetes auth role por servicio
resource "vault_kubernetes_auth_backend_role" "events_service" {
  role_name                        = "events-service"
  bound_service_account_names      = ["events-service"]
  bound_service_account_namespaces = ["prod"]
  token_policies                   = ["events-policy"]
  token_ttl                        = 3600
}
```

---

## 5. Orquestación principal con variable `countries`

```hcl
# main.tf

variable "countries" {
  description = "Lista de países (códigos ISO 3166-1 alpha-2) a provisionar"
  type        = list(string)
  default     = ["CO", "MX", "AR"]
}

module "keycloak_realms" {
  for_each        = toset(var.countries)
  source          = "./modules/keycloak-realm"
  country_code    = each.key
  app_domain      = var.app_domain
  keycloak_url    = var.keycloak_url
  federation_type = var.federation_config[each.key].type
  federation_config = var.federation_config[each.key].params
}

module "vault_pki" {
  source     = "./modules/vault-pki"
  countries  = var.countries
  vault_addr = var.vault_addr
}

module "vault_secrets" {
  source     = "./modules/vault-secrets"
  countries  = var.countries
}
```

---

## 6. Outputs — JWKS y OCSP endpoints por país

```hcl
# outputs.tf

output "jwks_endpoints" {
  description = "JWKS endpoints por país para validación de tokens"
  value = {
    for country in var.countries :
    country => "${var.keycloak_url}/realms/${country}/protocol/openid-connect/certs"
  }
}

output "ocsp_endpoints" {
  description = "OCSP endpoints por país para verificación de certificados de dispositivo"
  value = {
    for country in var.countries :
    country => "${var.vault_addr}/v1/pki-${lower(country)}/ocsp"
  }
}

output "crl_endpoints" {
  description = "CRL endpoints por país"
  value = {
    for country in var.countries :
    country => "${var.vault_addr}/v1/pki-${lower(country)}/crl"
  }
}
```

Ejemplo de salida para `countries = ["CO", "MX", "AR"]`:

```json
{
  "jwks_endpoints": {
    "CO": "https://auth.sistema.com/realms/CO/protocol/openid-connect/certs",
    "MX": "https://auth.sistema.com/realms/MX/protocol/openid-connect/certs",
    "AR": "https://auth.sistema.com/realms/AR/protocol/openid-connect/certs"
  },
  "ocsp_endpoints": {
    "CO": "https://vault.sistema.com/v1/pki-co/ocsp",
    "MX": "https://vault.sistema.com/v1/pki-mx/ocsp",
    "AR": "https://vault.sistema.com/v1/pki-ar/ocsp"
  }
}
```

---

## 7. Orden de provisión

El orden de aplicación de Terraform está determinado por las dependencias entre módulos. El orden correcto es:

1. **Vault init** — `vault operator init` (manual, una sola vez, antes de ejecutar Terraform).
2. **`vault-secrets`** — Mounts KV v2, políticas, AppRole, Kubernetes auth. Sin dependencias de otros módulos.
3. **Keycloak PostgreSQL** — La BD de Keycloak debe estar activa antes de que Terraform configure los realms.
4. **`keycloak-realm`** — Realms, clientes, roles, federación. Depende de que Keycloak esté activo y la BD disponible.
5. **`vault-pki`** — Root CA, CAs intermedias, roles de emisión. Depende de que Vault esté activo y con KV configurado.
6. **EMQX mTLS config** — Actualización del bundle de CAs en EMQX. Depende de que las CAs intermedias estén emitidas por `vault-pki`.

```bash
# Ejecutar en orden
terraform apply -target=module.vault_secrets
terraform apply -target=module.keycloak_realms
terraform apply -target=module.vault_pki
# Luego aplicar la configuración de EMQX (módulo separado del change ingestion-mqtt)
```
