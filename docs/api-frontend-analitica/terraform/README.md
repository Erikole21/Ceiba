# Terraform — API / Frontend / Analítica

**Componente:** `api-frontend-analitica`  
**Versión del documento:** 1.0

---

## 1. Alcance de Terraform en este change

Terraform gestiona únicamente la **infraestructura externa al clúster**: el API Gateway (Kong), la base de datos de metadatos de Superset y los recursos de object storage de la Web App. Los **seis microservicios de dominio** (search-service, alerts-service, incident-service, analytics-service, devices-service, vehicles-service) se despliegan y gestionan exclusivamente mediante **Helm** (ver `helm/README.md`); no tienen módulos Terraform propios en este change.

---

## 2. Estructura de Módulos

```
terraform/api-frontend-analitica/
├── main.tf                  # Composición de módulos + providers
├── variables.tf             # Variables de entrada
├── outputs.tf               # Variables de salida
├── terraform.tfvars.example # Ejemplo de valores (NO incluir en git)
├── modules/
│   ├── api-gateway/         # Kong OSS: routes, plugins, TLS certs
│   ├── superset/            # Superset: metadata DB, ClickHouse connection
│   └── webapp/              # Web App: CDN/bucket de assets estáticos (si aplica)
└── environments/
    ├── cloud.tfvars         # Variables para cloud (EKS/GKE/AKS)
    └── onprem.tfvars        # Variables para on-prem
```

---

## 2. Módulo `api-gateway`

Provisiona los recursos necesarios para Kong OSS como API Gateway.

### Recursos

- **TLS Certificate:** cert-manager CRD o recurso cloud (ACM, GCP Certificate Manager).
- **Kong Routes y Plugins:** via Kong Admin API (provider `Kong`).
- **Redis** (para rate-limiting): creación del database Redis si se usa Redis Cluster gestionado.

### Variables de Entrada

| Variable | Tipo | Descripción | Ejemplo |
|---|---|---|---|
| `kong_admin_url` | string | URL de la Kong Admin API. | `http://kong-admin.anti-hurto:8001` |
| `keycloak_jwks_url` | string | URL del JWKS endpoint de Keycloak. | `https://keycloak.anti-hurto.internal/realms/co/...` |
| `keycloak_issuer` | string | Issuer del JWT. | `https://keycloak.anti-hurto.internal/realms/co` |
| `redis_host` | string | Host de Redis para rate-limiting. | `redis-master.default.svc.cluster.local` |
| `redis_port` | number | Puerto Redis. | `6379` |
| `redis_database` | number | Database Redis para rate-limiting. | `1` |
| `api_domain` | string | Dominio del API. | `api.anti-hurto.internal` |
| `tls_cert_arn` | string | ARN del cert (AWS) o nombre del cert K8s. | `arn:aws:acm:us-east-1:123:cert/abc` |
| `rate_limit_per_minute` | map(number) | Límites por endpoint. | `{ search = 300, alerts = 600 }` |
| `country_code` | string | country_code del deployment. | `CO` |

### Variables de Salida

| Variable | Descripción |
|---|---|
| `api_gateway_endpoint` | URL pública del API Gateway. |
| `kong_service_ids` | Mapa de service IDs de Kong por microservicio. |
| `tls_certificate_id` | ID del certificado TLS provisionado. |

### Ejemplo `main.tf` (módulo)

```hcl
# modules/api-gateway/main.tf
terraform {
  required_providers {
    kong = {
      source  = "Kong/kong"
      version = "~> 6.0"
    }
  }
}

provider "kong" {
  kong_admin_uri = var.kong_admin_url
}

# Servicio search-service
resource "kong_service" "search" {
  name     = "search-service"
  protocol = "http"
  host     = "search-service.anti-hurto.svc.cluster.local"
  port     = 8080
  path     = "/"
}

# Route para /v1/plates
resource "kong_route" "search_plates" {
  name         = "search-plates"
  service_id   = kong_service.search.id
  paths        = ["/v1/plates"]
  strip_path   = false
  protocols    = ["https"]
}

# Plugin JWT en la route
resource "kong_plugin" "jwt_search" {
  name      = "jwt"
  route_id  = kong_route.search_plates.id
  config_json = jsonencode({
    key_claim_name    = "sub"
    claims_to_verify  = ["exp", "nbf"]
    secret_is_base64  = false
  })
}

# Plugin rate-limiting
resource "kong_plugin" "rate_limit_search" {
  name      = "rate-limiting-advanced"
  route_id  = kong_route.search_plates.id
  config_json = jsonencode({
    limit       = [var.rate_limit_per_minute["search"]]
    window_size = [60]
    window_type = "sliding"
    strategy    = "redis"
    redis = {
      host     = var.redis_host
      port     = var.redis_port
      database = var.redis_database
    }
  })
}

# (Repetir para cada microservicio)
```

---

## 3. Módulo `superset`

Provisiona los recursos para Apache Superset.

### Recursos

- **PostgreSQL de metadata:** base de datos dedicada para el estado de Superset.
- **ClickHouse connection:** registro de la conexión en Superset via API.
- **Secrets K8s:** `superset-secrets` con `SECRET_KEY`, `SUPERSET_DB_PASS`, `CLICKHOUSE_PASSWORD`.

### Variables de Entrada

| Variable | Tipo | Descripción | Ejemplo |
|---|---|---|---|
| `superset_admin_url` | string | URL de la Superset Admin API. | `https://superset.anti-hurto.internal` |
| `superset_db_host` | string | Host PostgreSQL metadata. | `pg-superset.default.svc.cluster.local` |
| `superset_db_name` | string | Nombre de la DB de metadata. | `superset` |
| `clickhouse_host` | string | Host ClickHouse. | `clickhouse.default.svc.cluster.local` |
| `clickhouse_database` | string | Database ClickHouse. | `anti_hurto` |
| `clickhouse_username` | string | Usuario de solo lectura. | `superset_ro` |
| `superset_secret_key` | string (sensitive) | SECRET_KEY de Superset. | — |
| `clickhouse_password` | string (sensitive) | Contraseña ClickHouse. | — |

### Variables de Salida

| Variable | Descripción |
|---|---|
| `superset_db_id` | ID de la conexión ClickHouse registrada en Superset. |
| `superset_url` | URL pública de Superset. |

### Ejemplo `main.tf` (módulo)

```hcl
# modules/superset/main.tf

# Crear Secret K8s con credentials
resource "kubernetes_secret" "superset_secrets" {
  metadata {
    name      = "superset-secrets"
    namespace = var.namespace
  }
  data = {
    SECRET_KEY        = var.superset_secret_key
    SUPERSET_DB_PASS  = var.superset_db_password
    CLICKHOUSE_PASSWORD = var.clickhouse_password
  }
  type = "Opaque"
}

# Crear base de datos de metadata de Superset en PostgreSQL
resource "postgresql_database" "superset_metadata" {
  name              = var.superset_db_name
  owner             = "superset"
  lc_collate        = "en_US.UTF-8"
  connection_limit  = 50
  allow_connections = true
}

# Nota: El registro de la conexión ClickHouse en Superset se realiza
# mediante el Superset CLI (superset db upgrade) en el Job K8s de
# inicialización. El provider de Superset para Terraform no está
# oficialmente soportado; se recomienda usar el Helm chart init job.
```

---

## 4. Módulo `webapp`

Provisiona los recursos para servir la Web App (si se usa CDN/bucket de assets).

### Recursos

- **Bucket de assets estáticos:** S3-compatible (AWS S3 / GCS / MinIO).
- **CDN** (opcional): distribución CloudFront / Cloud CDN (detrás de adapter hexagonal).

### Variables de Entrada

| Variable | Tipo | Descripción | Ejemplo |
|---|---|---|---|
| `webapp_bucket_name` | string | Nombre del bucket. | `anti-hurto-webapp-co` |
| `webapp_domain` | string | Dominio de la Web App. | `app.anti-hurto.internal` |
| `cdn_enabled` | bool | Habilitar CDN. | `false` (on-prem), `true` (cloud) |
| `cloud_provider` | string | `aws`, `gcp`, `azure`, `onprem`. | `aws` |

### Variables de Salida

| Variable | Descripción |
|---|---|
| `webapp_url` | URL pública de la Web App. |
| `webapp_bucket_id` | ID del bucket de assets. |

---

## 5. Composición Principal (`main.tf`)

```hcl
# terraform/api-frontend-analitica/main.tf

module "api_gateway" {
  source = "./modules/api-gateway"
  
  kong_admin_url        = var.kong_admin_url
  keycloak_jwks_url     = var.keycloak_jwks_url
  keycloak_issuer       = var.keycloak_issuer
  redis_host            = var.redis_host
  redis_port            = var.redis_port
  redis_database        = var.redis_database
  api_domain            = var.api_domain
  rate_limit_per_minute = var.rate_limit_per_minute
  country_code          = var.country_code
}

module "superset" {
  source = "./modules/superset"
  
  superset_admin_url     = var.superset_admin_url
  superset_db_host       = var.superset_db_host
  superset_db_name       = var.superset_db_name
  clickhouse_host        = var.clickhouse_host
  clickhouse_database    = var.clickhouse_database
  clickhouse_username    = var.clickhouse_username
  superset_secret_key    = var.superset_secret_key
  clickhouse_password    = var.clickhouse_password
  namespace              = var.namespace
  
  # Superset depende de que el API Gateway esté levantado
  depends_on = [module.api_gateway]
}

module "webapp" {
  source = "./modules/webapp"
  
  webapp_bucket_name = var.webapp_bucket_name
  webapp_domain      = var.webapp_domain
  cdn_enabled        = var.cdn_enabled
  cloud_provider     = var.cloud_provider
}
```

---

## 6. Orden de Provisioning

```
1. Kubernetes namespace y RBAC (pre-requisito)
2. Secrets K8s (superset-secrets, service-secrets) — módulo superset
3. PostgreSQL database de metadata Superset — módulo superset
4. API Gateway routes y plugins — módulo api-gateway
5. Superset K8s deployment (vía Helm, orquestado con null_resource) — módulo superset
6. Webapp bucket/CDN — módulo webapp (independiente)
```

---

## 7. Uso

```bash
# Inicializar
terraform init -backend-config="bucket=anti-hurto-tfstate" \
               -backend-config="key=api-frontend-analitica/terraform.tfstate"

# Planificar con variables de cloud
terraform plan -var-file="environments/cloud.tfvars" -out=tfplan

# Aplicar
terraform apply tfplan

# Destruir (con precaución)
terraform destroy -var-file="environments/cloud.tfvars"
```

---

## 8. Referencias

- [helm/README.md](../helm/README.md)
- [api-gateway.md](../api-gateway.md)
- [superset-integration.md](../superset-integration.md)
- [ADR-005 — Arquitectura Hexagonal](../../propuesta-arquitectura-hurto-vehiculos.md#adr-005--arquitectura-hexagonal-para-servicios-sensibles-a-la-nube)
