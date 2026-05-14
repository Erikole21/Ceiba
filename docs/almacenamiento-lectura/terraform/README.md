# Terraform — Infraestructura del Almacenamiento de Lectura

**Componente:** Almacenamiento de lectura — aprovisionamiento IaC  
**Versión del documento:** 1.0  
**Referencia:** [helm/README.md](../helm/README.md) · [postgresql-ha.md](../postgresql-ha.md)

---

## 1. Estructura de Módulos

```
terraform/
├── modules/
│   ├── postgresql/          # Slot de replicación, extensiones, credenciales
│   ├── opensearch/          # Bootstrap cluster, plantillas ILM, credenciales
│   ├── clickhouse/          # Tablas, Kafka Engine Table, vistas H3
│   └── redis/               # Sentinel, credenciales, config
├── environments/
│   ├── dev/
│   │   └── main.tf
│   ├── staging/
│   │   └── main.tf
│   └── prod/
│       └── main.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

---

## 2. Módulo `postgresql`

### 2.1 Responsabilidades

- Crear el slot de replicación `debezium_slot_events`.
- Crear la publicación `debezium_pub_events` en PostgreSQL.
- Provisionar extensiones: `postgis`, `pg_partman`, `pgcrypto`, `btree_gist`.
- Crear roles y otorgar permisos.
- Rotar credenciales en Vault y actualizar el secret de Kubernetes.

### 2.2 Variables de Entrada

```hcl
variable "pg_host" {
  description = "Hostname del primario PostgreSQL (o balanceador de escritura)"
  type        = string
}

variable "pg_port" {
  description = "Puerto de PostgreSQL"
  type        = number
  default     = 5432
}

variable "pg_admin_user" {
  description = "Usuario administrador de PostgreSQL"
  type        = string
  default     = "postgres"
}

variable "pg_admin_password" {
  description = "Contraseña del usuario administrador"
  type        = string
  sensitive   = true
}

variable "pg_database" {
  description = "Nombre de la base de datos"
  type        = string
  default     = "antihurto"
}

variable "replication_slot_name" {
  description = "Nombre del slot de replicación para Debezium"
  type        = string
  default     = "debezium_slot_events"
}

variable "publication_name" {
  description = "Nombre de la publicación PostgreSQL"
  type        = string
  default     = "debezium_pub_events"
}

variable "enable_postgis" {
  description = "Instalar extensión PostGIS"
  type        = bool
  default     = true
}

variable "enable_pg_partman" {
  description = "Instalar extensión pg_partman"
  type        = bool
  default     = true
}

variable "vault_path" {
  description = "Path en Vault para almacenar las credenciales de PostgreSQL"
  type        = string
  default     = "secret/antihurto/postgresql"
}
```

### 2.3 Recursos Terraform

```hcl
# modules/postgresql/main.tf

terraform {
  required_providers {
    postgresql = {
      source  = "cyrilgdn/postgresql"
      version = "~> 1.21"
    }
    vault = {
      source  = "hashicorp/vault"
      version = "~> 3.20"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }
}

# Extensiones
resource "postgresql_extension" "postgis" {
  count    = var.enable_postgis ? 1 : 0
  name     = "postgis"
  database = var.pg_database
}

resource "postgresql_extension" "pg_partman" {
  count    = var.enable_pg_partman ? 1 : 0
  name     = "pg_partman"
  database = var.pg_database
}

resource "postgresql_extension" "pgcrypto" {
  name     = "pgcrypto"
  database = var.pg_database
}

resource "postgresql_extension" "btree_gist" {
  name     = "btree_gist"
  database = var.pg_database
}

# Slot de replicación para Debezium
resource "postgresql_replication_slot" "debezium" {
  name   = var.replication_slot_name
  plugin = "pgoutput"
}

# Roles
resource "postgresql_role" "app_writer" {
  name     = "app_writer"
  login    = true
  password = random_password.app_writer.result
}

resource "postgresql_role" "app_reader" {
  name     = "app_reader"
  login    = true
  password = random_password.app_reader.result
}

resource "postgresql_role" "debezium_replication" {
  name        = "debezium_replication"
  login       = true
  replication = true
  password    = random_password.debezium.result
}

# Contraseñas aleatorias
resource "random_password" "app_writer" {
  length  = 32
  special = false
}

resource "random_password" "app_reader" {
  length  = 32
  special = false
}

resource "random_password" "debezium" {
  length  = 32
  special = false
}

# Credenciales en Vault
resource "vault_kv_secret_v2" "pg_credentials" {
  mount = "secret"
  name  = "antihurto/postgresql"

  data_json = jsonencode({
    app_writer_password     = random_password.app_writer.result
    app_reader_password     = random_password.app_reader.result
    debezium_password       = random_password.debezium.result
  })
}

# Secret de Kubernetes sincronizado desde Vault (via External Secrets Operator)
resource "kubernetes_manifest" "pg_external_secret" {
  manifest = {
    apiVersion = "external-secrets.io/v1beta1"
    kind       = "ExternalSecret"
    metadata = {
      name      = "pg-credentials"
      namespace = "antihurto-storage"
    }
    spec = {
      refreshInterval = "1h"
      secretStoreRef = {
        name = "vault-backend"
        kind = "ClusterSecretStore"
      }
      target = {
        name = "pg-credentials"
      }
      data = [
        {
          secretKey = "app-password"
          remoteRef = {
            key      = "antihurto/postgresql"
            property = "app_writer_password"
          }
        }
      ]
    }
  }
}
```

### 2.4 Outputs

```hcl
# modules/postgresql/outputs.tf

output "replication_slot_name" {
  description = "Nombre del slot de replicación creado"
  value       = postgresql_replication_slot.debezium.name
}

output "app_writer_role" {
  description = "Nombre del rol de escritura"
  value       = postgresql_role.app_writer.name
}

output "vault_secret_path" {
  description = "Path en Vault donde se almacenaron las credenciales"
  value       = vault_kv_secret_v2.pg_credentials.path
}
```

---

## 3. Módulo `opensearch`

### 3.1 Responsabilidades

- Bootstrap del cluster (verificar salud).
- Crear la plantilla de índice `vehicle-events-template`.
- Crear la política ILM `vehicle-events-lifecycle`.
- Crear el repositorio de snapshots S3.
- Crear usuarios y roles de seguridad OpenSearch.

### 3.2 Variables de Entrada

```hcl
variable "opensearch_endpoint" {
  description = "URL del endpoint OpenSearch (incluye puerto)"
  type        = string
}

variable "opensearch_admin_user" {
  type      = string
  default   = "admin"
}

variable "opensearch_admin_password" {
  type      = string
  sensitive = true
}

variable "s3_snapshot_bucket" {
  description = "Bucket S3 para snapshots automáticos"
  type        = string
  default     = "antihurto-opensearch-snapshots"
}

variable "ilm_hot_days" {
  description = "Días en fase hot antes de transición a warm"
  type        = number
  default     = 90
}

variable "ilm_delete_days" {
  description = "Días totales antes de eliminar el índice"
  type        = number
  default     = 730
}
```

### 3.3 Recursos Terraform

```hcl
# modules/opensearch/main.tf

terraform {
  required_providers {
    opensearch = {
      source  = "opensearch-project/opensearch"
      version = "~> 2.2"
    }
  }
}

# Plantilla de índice
resource "opensearch_index_template" "vehicle_events" {
  name = "vehicle-events-template"
  body = file("${path.module}/templates/vehicle-events-template.json")
}

# Política ILM
resource "opensearch_ism_policy" "vehicle_events_lifecycle" {
  policy_id = "vehicle-events-lifecycle"
  body      = templatefile("${path.module}/templates/vehicle-events-ilm.json", {
    hot_days    = var.ilm_hot_days
    delete_days = var.ilm_delete_days
  })
}

# Repositorio de snapshots
resource "opensearch_snapshot_repository" "s3" {
  name = "antihurto-s3-repo"
  type = "s3"
  settings = {
    bucket    = var.s3_snapshot_bucket
    region    = var.aws_region
    base_path = "vehicle-events"
  }
}

# Roles de seguridad
resource "opensearch_role" "search_reader" {
  role_name = "search_reader"
  index_permissions {
    index_patterns  = ["vehicle-events-*"]
    allowed_actions = ["read", "search", "get"]
  }
}

resource "opensearch_role" "search_writer" {
  role_name = "search_writer"
  index_permissions {
    index_patterns  = ["vehicle-events-*"]
    allowed_actions = ["create_index", "index", "bulk"]
  }
}
```

---

## 4. Módulo `clickhouse`

### 4.1 Responsabilidades

- Crear la tabla `vehicle_events_daily` (MergeTree/ReplicatedMergeTree).
- Crear la Kafka Engine Table `vehicle_events_kafka`.
- Crear la vista materializada `vehicle_events_cdc_mv`.
- Crear la vista materializada `vehicle_events_h3_hourly`.
- Configurar TTL de retención.

### 4.2 Variables de Entrada

```hcl
variable "clickhouse_host" {
  description = "Host del cluster ClickHouse (o balanceador)"
  type        = string
}

variable "clickhouse_port" {
  description = "Puerto HTTP de ClickHouse"
  type        = number
  default     = 8123
}

variable "clickhouse_user" {
  type    = string
  default = "default"
}

variable "clickhouse_password" {
  type      = string
  sensitive = true
}

variable "kafka_brokers" {
  description = "Lista de brokers Kafka separados por coma"
  type        = string
  default     = "kafka-0:9092,kafka-1:9092,kafka-2:9092"
}

variable "retention_months" {
  description = "Meses de retención para vehicle_events_daily"
  type        = number
  default     = 24
}

variable "replication_enabled" {
  description = "Usar ReplicatedMergeTree en lugar de MergeTree"
  type        = bool
  default     = true
}
```

### 4.3 Recursos Terraform

```hcl
# modules/clickhouse/main.tf

terraform {
  required_providers {
    clickhouse = {
      source  = "ClickHouse/clickhouse"
      version = "~> 0.3"
    }
  }
}

resource "clickhouse_table" "vehicle_events_daily" {
  database = "default"
  name     = "vehicle_events_daily"
  engine   = var.replication_enabled ? "ReplicatedMergeTree('/clickhouse/tables/{shard}/vehicle_events_daily', '{replica}')" : "MergeTree()"

  columns = file("${path.module}/sql/vehicle_events_daily_columns.sql")

  partition_by = "(country_code, toYYYYMM(event_ts))"
  order_by     = "(country_code, toDate(event_ts), h3_index_8, plate_normalized, event_id)"
  primary_key  = "(country_code, toDate(event_ts), h3_index_8)"
  ttl          = "event_ts + INTERVAL ${var.retention_months} MONTH DELETE"
}

resource "clickhouse_table" "vehicle_events_kafka" {
  database = "default"
  name     = "vehicle_events_kafka"
  engine   = "Kafka"
  engine_params = {
    kafka_broker_list     = var.kafka_brokers
    kafka_topic_list      = "pg.events.cdc"
    kafka_group_name      = "clickhouse-cdc-consumer"
    kafka_format          = "JSONEachRow"
    kafka_num_consumers   = "4"
    kafka_skip_broken_messages = "10"
  }
  columns = file("${path.module}/sql/vehicle_events_kafka_columns.sql")
}

resource "clickhouse_view" "vehicle_events_cdc_mv" {
  database    = "default"
  name        = "vehicle_events_cdc_mv"
  is_materialized = true
  to_table    = "vehicle_events_daily"
  query       = file("${path.module}/sql/vehicle_events_cdc_mv.sql")

  depends_on = [
    clickhouse_table.vehicle_events_daily,
    clickhouse_table.vehicle_events_kafka
  ]
}

resource "clickhouse_view" "vehicle_events_h3_hourly" {
  database    = "default"
  name        = "vehicle_events_h3_hourly"
  is_materialized = true
  engine      = "SummingMergeTree()"
  order_by    = "(country_code, h3_index_8, hour_ts)"
  query       = file("${path.module}/sql/vehicle_events_h3_hourly.sql")

  depends_on = [clickhouse_table.vehicle_events_daily]
}
```

---

## 5. Módulo `redis`

### 5.1 Responsabilidades

- Crear credenciales de Redis en Vault.
- Sincronizar el secret de Kubernetes vía External Secrets Operator.
- Configurar ACLs de usuario (`app_matcher`, `app_session`, `app_rate_limit`).

### 5.2 Variables de Entrada

```hcl
variable "redis_sentinel_hosts" {
  description = "Lista de hosts Sentinel (host:port)"
  type        = list(string)
  default     = ["redis-sentinel-0:26379", "redis-sentinel-1:26379", "redis-sentinel-2:26379"]
}

variable "redis_master_set" {
  description = "Nombre del master set en Sentinel"
  type        = string
  default     = "antihurto-master"
}

variable "vault_path" {
  description = "Path en Vault para credenciales Redis"
  type        = string
  default     = "secret/antihurto/redis"
}
```

### 5.3 Recursos Terraform

```hcl
# modules/redis/main.tf

resource "random_password" "redis_password" {
  length  = 32
  special = false
}

resource "vault_kv_secret_v2" "redis_credentials" {
  mount = "secret"
  name  = "antihurto/redis"

  data_json = jsonencode({
    password = random_password.redis_password.result
  })
}

resource "kubernetes_manifest" "redis_external_secret" {
  manifest = {
    apiVersion = "external-secrets.io/v1beta1"
    kind       = "ExternalSecret"
    metadata = {
      name      = "redis-credentials"
      namespace = "antihurto-storage"
    }
    spec = {
      refreshInterval = "1h"
      secretStoreRef = { name = "vault-backend", kind = "ClusterSecretStore" }
      target         = { name = "redis-credentials" }
      data = [{
        secretKey = "redis-password"
        remoteRef = { key = "antihurto/redis", property = "password" }
      }]
    }
  }
}
```

### 5.4 Outputs

```hcl
output "redis_secret_path" {
  description = "Path en Vault con las credenciales Redis"
  value       = vault_kv_secret_v2.redis_credentials.path
}

output "sentinel_connection_string" {
  description = "String de conexión Sentinel para los servicios"
  value = "sentinel://${join(",", var.redis_sentinel_hosts)}/${var.redis_master_set}"
}
```

---

## 6. Orden de Aprovisionamiento

El orden correcto de ejecución de los módulos Terraform es:

```mermaid
graph TD
    VAULT[Vault PKI\n(prerequisito externo)] --> PG[módulo postgresql]
    VAULT --> RD[módulo redis]
    PG --> OS[módulo opensearch]
    PG --> CH[módulo clickhouse]
    CH -->|depende de Kafka existente| KAFKA[Kafka Cluster\n(prerequisito externo)]
    OS --> DEPLOY[Despliegue Helm\naplicar charts]
    CH --> DEPLOY
    RD --> DEPLOY
    DEPLOY --> CONNECTORS[Registrar conector Debezium\nvía API Kafka Connect]
```

### 6.1 Comandos de Aplicación

```bash
# 1. Inicializar Terraform
terraform init

# 2. Planificar (siempre antes de apply en producción)
terraform plan -var-file=environments/prod/terraform.tfvars -out=prod.plan

# 3. Aplicar en orden de módulos
terraform apply -target=module.postgresql prod.plan
terraform apply -target=module.redis prod.plan
terraform apply -target=module.opensearch prod.plan
terraform apply -target=module.clickhouse prod.plan

# 4. Aplicar el resto (outputs, secrets de K8s)
terraform apply prod.plan
```

---

## 7. Variables de Entrada Principales

```hcl
# variables.tf (raíz)

variable "environment" {
  description = "Entorno de despliegue: dev | staging | prod"
  type        = string
}

variable "country_codes" {
  description = "Lista de country_codes para los que se crean particiones"
  type        = list(string)
  default     = ["CO", "MX", "PE"]
}

variable "aws_region" {
  description = "Región AWS (solo si se usan servicios gestionados)"
  type        = string
  default     = ""
}

variable "pg_host" { type = string }
variable "opensearch_endpoint" { type = string }
variable "clickhouse_host" { type = string }
variable "kafka_brokers" { type = string }

variable "vault_address" {
  description = "URL de Vault para la gestión de secretos"
  type        = string
}
```

---

## 8. Outputs Globales

```hcl
# outputs.tf (raíz)

output "pg_replication_slot" {
  value = module.postgresql.replication_slot_name
}

output "os_index_template" {
  value = "vehicle-events-template (creado)"
}

output "ch_tables" {
  value = [
    "vehicle_events_daily",
    "vehicle_events_kafka",
    "vehicle_events_h3_hourly"
  ]
}

output "redis_sentinel_connection" {
  value = module.redis.sentinel_connection_string
}

output "vault_secrets" {
  value = {
    postgresql = module.postgresql.vault_secret_path
    redis      = module.redis.redis_secret_path
  }
}
```
