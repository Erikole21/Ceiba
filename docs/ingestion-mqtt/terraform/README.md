# Terraform — Provisión de Infraestructura para la Capa de Ingestión

**Pilar:** 2 — Comunicación Segura y Eficiente  
**ADR de referencia:** ADR-004 · ADR-005  
**Referencia:** [Visión General](../overview.md) · [object-storage.md](../object-storage.md) · [helm/README.md](../helm/README.md)  
**Última actualización:** 2026-05-13

---

## 1. Estructura de Módulos

La infraestructura de la capa de ingestión se organiza en cuatro módulos Terraform independientes. Cada módulo puede utilizarse de forma individual o como parte de la composición completa del Pilar 2.

```
terraform/
├── main.tf                    # Composición de todos los módulos
├── variables.tf               # Variables globales del pilar
├── outputs.tf                 # Outputs consolidados
├── modules/
│   ├── emqx-cluster/          # Módulo: clúster EMQX en K8s
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── upload-service/        # Módulo: recursos del Upload Service
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── object-storage/        # Módulo: bucket S3-API y lifecycle
│   │   ├── main.tf            # Adaptador hexagonal a nivel infra (ADR-005)
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── kms-key/               # Módulo: clave KMS para SSE
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
```

---

## 2. Módulo: emqx-cluster

Provisiona los recursos Kubernetes necesarios para el clúster EMQX: namespace, RBAC, Secrets de TLS y Kafka, NetworkPolicy y (opcionalmente) los Load Balancers del cloud.

### 2.1 Variables de Entrada

| Variable | Tipo | Descripción | Valor por defecto |
|---|---|---|---|
| `namespace` | `string` | Namespace Kubernetes para EMQX | `"mqtt"` |
| `node_count` | `number` | Número de nodos del clúster EMQX | `3` |
| `emqx_version` | `string` | Versión de EMQX a desplegar | `"5.6.0"` |
| `tls_cert_pem` | `string` (sensitive) | Certificado TLS del servidor EMQX (PEM) | — |
| `tls_key_pem` | `string` (sensitive) | Clave privada TLS del servidor (PEM) | — |
| `ca_chain_pem` | `string` (sensitive) | Bundle de CAs de confianza (PEM) | — |
| `kafka_bootstrap_hosts` | `string` | Bootstrap servers de Kafka | — |
| `kafka_sasl_username` | `string` (sensitive) | Usuario SASL para Kafka | — |
| `kafka_sasl_password` | `string` (sensitive) | Contraseña SASL para Kafka | — |
| `session_expiry_interval` | `string` | TTL de sesiones MQTT | `"2h"` |
| `enable_anti_affinity` | `bool` | Anti-afinidad por zona K8s | `true` |
| `load_balancer_enabled` | `bool` | Provisionar LoadBalancer cloud | `false` |
| `cloud_provider` | `string` | `aws`, `gcp`, `azure`, `onprem` | `"onprem"` |

### 2.2 Variables de Salida

| Output | Tipo | Descripción |
|---|---|---|
| `emqx_endpoint` | `string` | URL del endpoint MQTT (`mqtts://...`) |
| `emqx_service_name` | `string` | Nombre del Service K8s de EMQX |
| `namespace` | `string` | Namespace donde se despliega EMQX |

---

## 3. Módulo: upload-service

Provisiona los recursos Kubernetes del Upload Service: Deployment, Service, Ingress/LoadBalancer, Secret de configuración, y los permisos IAM/SA necesarios para acceder al object storage.

### 3.1 Variables de Entrada

| Variable | Tipo | Descripción | Valor por defecto |
|---|---|---|---|
| `namespace` | `string` | Namespace Kubernetes | `"mqtt"` |
| `image_tag` | `string` | Tag de la imagen Docker del servicio | — |
| `replica_count` | `number` | Número de réplicas | `3` |
| `storage_backend` | `string` | Adaptador de storage: `minio`, `s3`, `gcs`, `azure` | `"minio"` |
| `storage_bucket_name` | `string` | Nombre del bucket de destino | — |
| `storage_endpoint` | `string` | Endpoint del storage (para MinIO o custom) | `""` |
| `kms_key_id` | `string` | ID/ARN de la clave KMS | — |
| `vault_address` | `string` | URL de Vault PKI | — |
| `vault_pki_path` | `string` | Path del PKI en Vault | `"pki"` |
| `presign_ttl_seconds` | `number` | TTL de presigned URLs (siempre 300) | `300` |
| `tls_enabled` | `bool` | Habilitar TLS en el servicio | `true` |
| `tls_cert_pem` | `string` (sensitive) | Certificado TLS del servicio (PEM) | — |
| `tls_key_pem` | `string` (sensitive) | Clave privada TLS (PEM) | — |

### 3.2 Variables de Salida

| Output | Tipo | Descripción |
|---|---|---|
| `upload_service_endpoint` | `string` | URL del Upload Service |
| `service_account_name` | `string` | Service Account K8s con permisos de storage |

---

## 4. Módulo: object-storage

Provisiona el bucket de object storage con lifecycle policy, cifrado SSE-KMS y bucket policy. Este módulo implementa el patrón del adaptador hexagonal (ADR-005) a nivel de infraestructura: la misma interfaz de variables de entrada funciona para AWS S3, GCS, Azure Blob y MinIO.

### 4.1 Variables de Entrada

| Variable | Tipo | Descripción | Valor por defecto |
|---|---|---|---|
| `provider` | `string` | Proveedor: `aws`, `gcp`, `azure`, `minio` | `"minio"` |
| `bucket_name` | `string` | Nombre del bucket | — |
| `region` | `string` | Región del bucket (para cloud providers) | `""` |
| `country_code` | `string` | Código ISO del país (para naming y tagging) | — |
| `kms_key_id` | `string` | ID/ARN/URI de la clave KMS | — |
| `full_images_expiry_days` | `number` | Días de retención de imágenes completas | `180` |
| `thumbnails_expiry_days` | `number` | Días de retención de thumbnails | `730` |
| `enable_cross_region_replication` | `bool` | Habilitar replicación cross-region para DR | `false` |
| `replication_destination_bucket` | `string` | Bucket destino de replicación (si DR habilitado) | `""` |
| `replication_destination_kms_key_id` | `string` | Clave KMS del bucket destino | `""` |
| `enable_versioning` | `bool` | Habilitar versionado de objetos | `false` |
| `minio_endpoint` | `string` | Endpoint de MinIO (solo para provider=minio) | `""` |
| `minio_access_key` | `string` (sensitive) | Access key de MinIO | `""` |
| `minio_secret_key` | `string` (sensitive) | Secret key de MinIO | `""` |

### 4.2 Variables de Salida

| Output | Tipo | Descripción |
|---|---|---|
| `bucket_arn` | `string` | ARN/URI del bucket creado |
| `bucket_endpoint` | `string` | Endpoint S3-API del bucket |
| `lifecycle_rule_ids` | `list(string)` | IDs de las reglas de lifecycle aplicadas |

### 4.3 Implementación como Adaptador Hexagonal (ADR-005)

El módulo `object-storage` es la manifestación del principio hexagonal (ADR-005) a nivel de infraestructura. Internamente el módulo usa `count` y `for_each` junto con bloques `provider` condicionales para crear los recursos del proveedor seleccionado. El código de la aplicación (Upload Service) no necesita cambiar al cambiar de proveedor; solo cambia la variable `provider` en Terraform.

```hcl
# Fragmento ilustrativo de la lógica de selección de proveedor
resource "aws_s3_bucket" "evidence" {
  count  = var.provider == "aws" ? 1 : 0
  bucket = var.bucket_name
  tags   = local.common_tags
}

resource "google_storage_bucket" "evidence" {
  count    = var.provider == "gcp" ? 1 : 0
  name     = var.bucket_name
  location = var.region
  labels   = local.common_labels
}

# Para MinIO se usa el provider minio/minio
resource "minio_s3_bucket" "evidence" {
  count  = var.provider == "minio" ? 1 : 0
  bucket = var.bucket_name
}
```

---

## 5. Módulo: kms-key

Provisiona la clave KMS para SSE-KMS del object storage. Abstracto sobre el proveedor de KMS.

### 5.1 Variables de Entrada

| Variable | Tipo | Descripción | Valor por defecto |
|---|---|---|---|
| `provider` | `string` | Proveedor KMS: `aws`, `gcp`, `azure`, `vault` | `"vault"` |
| `key_alias` | `string` | Alias/nombre de la clave | `"ceiba-evidence-key"` |
| `country_code` | `string` | País (para tagging y naming) | — |
| `rotation_enabled` | `bool` | Habilitar rotación automática anual | `true` |
| `vault_address` | `string` | URL de Vault (para provider=vault) | `""` |
| `vault_transit_path` | `string` | Path del Transit Engine en Vault | `"transit"` |

### 5.2 Variables de Salida

| Output | Tipo | Descripción |
|---|---|---|
| `key_id` | `string` | ID/ARN/URI de la clave creada |
| `key_alias` | `string` | Alias de la clave para referencia |

---

## 6. Ejemplos de Uso

### 6.1 AWS

```hcl
# main.tf — Despliegue en AWS

module "kms_key" {
  source       = "./modules/kms-key"
  provider     = "aws"
  key_alias    = "ceiba-evidence-co"
  country_code = "CO"
  rotation_enabled = true
}

module "object_storage" {
  source       = "./modules/object-storage"
  provider     = "aws"
  bucket_name  = "ceiba-evidence-co"
  region       = "us-east-1"
  country_code = "CO"
  kms_key_id   = module.kms_key.key_id

  full_images_expiry_days = 180
  thumbnails_expiry_days  = 730

  enable_cross_region_replication    = true
  replication_destination_bucket     = "ceiba-evidence-co-dr"
  replication_destination_kms_key_id = var.dr_kms_key_id
}

module "upload_service" {
  source           = "./modules/upload-service"
  namespace        = "mqtt"
  image_tag        = "1.2.0"
  replica_count    = 3
  storage_backend  = "s3"
  storage_bucket_name = module.object_storage.bucket_arn
  kms_key_id       = module.kms_key.key_id
  vault_address    = var.vault_address
  presign_ttl_seconds = 300
  tls_enabled      = true
  tls_cert_pem     = var.upload_service_tls_cert
  tls_key_pem      = var.upload_service_tls_key
}
```

### 6.2 GCP

```hcl
# main.tf — Despliegue en GCP

module "kms_key" {
  source       = "./modules/kms-key"
  provider     = "gcp"
  key_alias    = "ceiba-evidence-co"
  country_code = "CO"
}

module "object_storage" {
  source       = "./modules/object-storage"
  provider     = "gcp"
  bucket_name  = "ceiba-evidence-co"
  region       = "southamerica-east1"
  country_code = "CO"
  kms_key_id   = module.kms_key.key_id

  full_images_expiry_days = 180
  thumbnails_expiry_days  = 730
}

module "upload_service" {
  source           = "./modules/upload-service"
  storage_backend  = "gcs"
  storage_bucket_name = module.object_storage.bucket_arn
  kms_key_id       = module.kms_key.key_id
  # ... resto de variables
}
```

### 6.3 On-prem (MinIO)

```hcl
# main.tf — Despliegue on-prem con MinIO y Vault Transit

module "kms_key" {
  source              = "./modules/kms-key"
  provider            = "vault"
  key_alias           = "ceiba-evidence-key"
  country_code        = "CO"
  vault_address       = "https://vault.internal:8200"
  vault_transit_path  = "transit"
}

module "object_storage" {
  source         = "./modules/object-storage"
  provider       = "minio"
  bucket_name    = "ceiba-evidence-co"
  country_code   = "CO"
  kms_key_id     = module.kms_key.key_id
  minio_endpoint = "https://minio.internal:9000"
  minio_access_key = var.minio_access_key  # Desde Vault Secrets
  minio_secret_key = var.minio_secret_key  # Desde Vault Secrets

  full_images_expiry_days = 180
  thumbnails_expiry_days  = 730
}

module "emqx_cluster" {
  source              = "./modules/emqx-cluster"
  namespace           = "mqtt"
  node_count          = 3
  kafka_bootstrap_hosts = var.kafka_bootstrap_hosts
  kafka_sasl_username = var.kafka_sasl_username
  kafka_sasl_password = var.kafka_sasl_password
  session_expiry_interval = "2h"
  enable_anti_affinity = true
  cloud_provider      = "onprem"
  tls_cert_pem        = var.emqx_tls_cert
  tls_key_pem         = var.emqx_tls_key
  ca_chain_pem        = var.ca_chain_pem
}

module "upload_service" {
  source           = "./modules/upload-service"
  storage_backend  = "minio"
  storage_bucket_name = module.object_storage.bucket_arn
  storage_endpoint = "https://minio.internal:9000"
  kms_key_id       = module.kms_key.key_id
  vault_address    = "https://vault.internal:8200"
  presign_ttl_seconds = 300
  # ... resto de variables
}
```

---

## 7. Notas Sobre el Módulo object-storage como Adaptador Hexagonal

El módulo `object-storage` materializa el principio de arquitectura hexagonal (ADR-005) en la capa de infraestructura. La misma interfaz de variables de entrada (`provider`, `bucket_name`, `kms_key_id`, `full_images_expiry_days`, `thumbnails_expiry_days`) funciona para AWS S3, GCS, Azure Blob y MinIO.

**Beneficios:**

- Un cambio de proveedor de cloud requiere modificar únicamente la variable `provider` y las credenciales. La lógica de la aplicación (Upload Service) y la estructura de paths del bucket no cambian.
- Las lifecycle policies se expresan en términos del dominio (`full_images_expiry_days = 180`) en lugar de en términos del proveedor (`aws_s3_bucket_lifecycle_configuration` vs. `google_storage_bucket_lifecycle_rule`). El módulo traduce internamente.
- El mismo pipeline de CI/CD puede desplegar en cualquier proveedor cambiando el archivo `terraform.tfvars`.

**Limitación:** la interoperabilidad S3 de GCS y Azure Blob tiene restricciones (ver [object-storage.md §6](../object-storage.md)). El módulo incluye validaciones que emiten advertencias si se usa un proveedor con funcionalidades parcialmente soportadas.

---

## 8. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [object-storage.md](../object-storage.md) | Especificación de bucket, lifecycle y cifrado que este módulo provisiona |
| [emqx-cluster.md](../emqx-cluster.md) | Parámetros de configuración que el módulo emqx-cluster configura |
| [helm/README.md](../helm/README.md) | Despliegue de la aplicación sobre la infraestructura provisionada por Terraform |
| [`docs/identidad-seguridad/terraform/README.md`](../../identidad-seguridad/terraform/README.md) | Módulo Terraform de Vault PKI (prerequisito para los certificados) |
| [ADR-005](../../propuesta-arquitectura-hurto-vehiculos.md#adr-005--arquitectura-hexagonal-para-servicios-sensibles-a-la-nube) | Decisión de arquitectura hexagonal que este módulo implementa |
