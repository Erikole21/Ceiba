# Terraform — Pilar 4: Sincronización de Países

**Change:** `sincronizacion-paises`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Estructura de Módulos

```
terraform/
├── modules/
│   ├── kafka-topics/             # Tópicos Kafka del Pilar 4
│   ├── schema-registry/          # Schema Registry: subjects y compatibilidad
│   ├── postgresql-canonical/     # Tabla canonical_vehicles y particiones
│   ├── redis-hotlist/            # Configuración Redis para lista roja
│   ├── mqtt-topics-bf/           # Topics MQTT del Bloom filter
│   └── minio-bf-storage/         # Bucket MinIO para versiones de BF
├── environments/
│   ├── staging/
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       └── terraform.tfvars
└── README.md                     # Este documento
```

---

## 2. Módulo `kafka-topics`

Crea y configura los cuatro tópicos Kafka del Pilar 4.

### 2.1 Variables de entrada

| Variable | Tipo | Descripción | Default |
|---|---|---|---|
| `kafka_bootstrap_servers` | `string` | Bootstrap servers de Kafka | — (obligatorio) |
| `replication_factor` | `number` | Factor de replicación | `3` |
| `events_topic_partitions` | `number` | Particiones de `stolen.vehicles.events` | `24` |
| `canonical_topic_partitions` | `number` | Particiones de `stolen.vehicles.canonical` | `24` |
| `dlq_topic_partitions` | `number` | Particiones de los DLQs | `6` |
| `events_retention_ms` | `number` | Retención de `stolen.vehicles.events` en ms | `604800000` (7d) |
| `canonical_retention_ms` | `number` | Retención de `stolen.vehicles.canonical` en ms | `2592000000` (30d) |
| `dlq_retention_ms` | `number` | Retención de los DLQs en ms | `1209600000` (14d) |

### 2.2 Variables de salida

| Output | Descripción |
|---|---|
| `events_topic_name` | Nombre del tópico de eventos (`stolen.vehicles.events`) |
| `canonical_topic_name` | Nombre del tópico canónico |
| `events_dlq_topic_name` | Nombre del DLQ de eventos |
| `canonical_dlq_topic_name` | Nombre del DLQ canónico |

### 2.3 Ejemplo de uso

```hcl
module "kafka_topics" {
  source = "./modules/kafka-topics"

  kafka_bootstrap_servers   = var.kafka_bootstrap_servers
  replication_factor        = 3
  events_topic_partitions   = 24
  canonical_topic_partitions = 24
  dlq_topic_partitions      = 6
  events_retention_ms       = 604800000
  canonical_retention_ms    = 2592000000
  dlq_retention_ms          = 1209600000
}
```

### 2.4 Recursos Terraform creados

```hcl
resource "kafka_topic" "stolen_vehicles_events" {
  name               = "stolen.vehicles.events"
  replication_factor = var.replication_factor
  partitions         = var.events_topic_partitions

  config = {
    "min.insync.replicas" = "2"
    "retention.ms"        = tostring(var.events_retention_ms)
    "cleanup.policy"      = "delete"
    "compression.type"    = "lz4"
  }
}

resource "kafka_topic" "stolen_vehicles_canonical" {
  name               = "stolen.vehicles.canonical"
  replication_factor = var.replication_factor
  partitions         = var.canonical_topic_partitions

  config = {
    "min.insync.replicas" = "2"
    "retention.ms"        = tostring(var.canonical_retention_ms)
    "cleanup.policy"      = "delete"
    "compression.type"    = "lz4"
  }
}

# DLQs — análogos con menos particiones
resource "kafka_topic" "stolen_vehicles_events_dlq" { ... }
resource "kafka_topic" "stolen_vehicles_canonical_dlq" { ... }
```

---

## 3. Módulo `schema-registry`

Registra el schema Avro inicial y configura la compatibilidad.

### 3.1 Variables de entrada

| Variable | Tipo | Descripción | Default |
|---|---|---|---|
| `schema_registry_url` | `string` | URL del Schema Registry | — (obligatorio) |
| `schema_file_path` | `string` | Ruta al archivo .avsc del schema canónico | `"./schemas/stolen-vehicle-event-v1.avsc"` |
| `compatibility_level` | `string` | Nivel de compatibilidad | `"BACKWARD"` |

### 3.2 Variables de salida

| Output | Descripción |
|---|---|
| `events_schema_id` | ID del schema registrado en el Schema Registry |
| `events_subject_name` | Nombre del subject (`stolen.vehicles.events-value`) |

### 3.3 Recursos creados

```hcl
resource "schemaregistry_schema" "stolen_vehicles_events" {
  subject            = "stolen.vehicles.events-value"
  schema             = file(var.schema_file_path)
  schema_type        = "AVRO"
  compatibility_level = var.compatibility_level
}
```

---

## 4. Módulo `postgresql-canonical`

Ejecuta el DDL de la tabla `canonical_vehicles` y crea las particiones para los países configurados.

### 4.1 Variables de entrada

| Variable | Tipo | Descripción | Default |
|---|---|---|---|
| `db_host` | `string` | Host de PostgreSQL | — (obligatorio) |
| `db_port` | `number` | Puerto de PostgreSQL | `5432` |
| `db_name` | `string` | Nombre de la base de datos | `"antihurto"` |
| `db_user` | `string` | Usuario de PostgreSQL | — (obligatorio) |
| `db_password` | `string` (sensitive) | Contraseña de PostgreSQL — nunca se incluye en la URL; se pasa como variable de entorno `PGPASSWORD` al provisioner | — (obligatorio) |
| `active_countries` | `list(string)` | Lista de country_codes activos | `["CO", "VE"]` |
| `create_roles` | `bool` | Crear los roles de BD del Pilar 4 | `true` |

> **Seguridad:** No usar `database_url` completa como argumento posicional de `psql` — expone la contraseña en el process list y en Terraform state. Las variables `db_*` se pasan como variables de entorno `PG*` al `local-exec` provisioner.

### 4.2 Variables de salida

| Output | Descripción |
|---|---|
| `table_name` | Nombre de la tabla (`canonical_vehicles`) |
| `created_partitions` | Lista de particiones creadas |
| `roles_created` | Lista de roles DB creados |

### 4.3 Recursos creados

```hcl
# Ejecuta el DDL usando el provider postgresql
resource "postgresql_extension" "uuid_ossp" {
  name = "uuid-ossp"
}

# El DDL completo está en modules/postgresql-canonical/sql/ddl.sql
resource "null_resource" "canonical_vehicles_ddl" {
  triggers = {
    ddl_hash = filemd5("${path.module}/sql/ddl.sql")
  }

  provisioner "local-exec" {
    # Pasar credenciales por variables de entorno, no en la URL (evita exposición en process list y Terraform state)
    environment = {
      PGHOST     = var.db_host
      PGPORT     = var.db_port
      PGDATABASE = var.db_name
      PGUSER     = var.db_user
      PGPASSWORD = var.db_password
    }
    command = "psql -f ${path.module}/sql/ddl.sql"
  }
}

# Particiones (una por país activo)
resource "null_resource" "country_partitions" {
  for_each = toset(var.active_countries)

  triggers = {
    country_code = each.value
  }

  provisioner "local-exec" {
    environment = {
      PGHOST     = var.db_host
      PGPORT     = var.db_port
      PGDATABASE = var.db_name
      PGUSER     = var.db_user
      PGPASSWORD = var.db_password
    }
    command = <<-EOT
      psql -c "
        CREATE TABLE IF NOT EXISTS canonical_vehicles_${lower(each.value)}
        PARTITION OF canonical_vehicles
        FOR VALUES IN ('${each.value}');
      "
    EOT
  }
}
```

---

## 5. Módulo `redis-hotlist`

Configura las políticas de Redis para la lista roja del Pilar 4.

### 5.1 Variables de entrada

| Variable | Tipo | Descripción | Default |
|---|---|---|---|
| `redis_url` | `string` | URL de Redis | — (obligatorio) |
| `hot_list_ttl_seconds` | `number` | TTL de las claves `stolen:{cc}:{plate}` | `2592000` (30d) |
| `key_prefix` | `string` | Prefijo de las claves | `"stolen"` |

### 5.2 Notas

Las claves Redis (`stolen:{cc}:{plate}`) son creadas dinámicamente por el Canonical Vehicles Service, no por Terraform. Este módulo solo configura las políticas de Redis relevantes:

- Keyspace notifications (si se requiere para el EDS cuando consume desde Redis).
- Política de eviction: se recomienda `allkeys-lru` en caso de presión de memoria.
- Configuración del sentinel o cluster.

```hcl
# Configurar política de eviction en Redis
resource "redis_config" "eviction_policy" {
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"
  }
}
```

---

## 6. Módulo `mqtt-topics-bf`

Configura las ACLs en EMQX para los topics de Bloom filter del Pilar 4.

### 6.1 Variables de entrada

| Variable | Tipo | Descripción | Default |
|---|---|---|---|
| `emqx_api_url` | `string` | URL de la API de administración de EMQX | — (obligatorio) |
| `active_countries` | `list(string)` | Países con topic MQTT de BF | `["CO", "VE"]` |
| `edge_distribution_client_id` | `string` | Client ID del Edge Distribution Service | `"edge-distribution-service"` |

### 6.2 Recursos creados

```hcl
# ACL: Edge Distribution Service puede publicar en todos los topics de BF
resource "emqx_acl" "eds_publish_bf" {
  clientid  = var.edge_distribution_client_id
  topic     = "countries/+/bloom-filter"
  action    = "publish"
  access    = "allow"
}

# ACL: agentes de borde de cada país suscriben solo a su topic
resource "emqx_acl" "device_subscribe_bf" {
  for_each = toset(var.active_countries)

  topic     = "countries/${each.value}/bloom-filter"
  action    = "subscribe"
  access    = "allow"
  # username pattern: device-{country_code}-*
  username  = "device-${each.value}-\\S+"
}
```

---

## 7. Módulo `minio-bf-storage`

Crea el bucket MinIO/S3 para almacenar versiones del Bloom filter.

### 7.1 Variables de entrada

| Variable | Tipo | Descripción | Default |
|---|---|---|---|
| `minio_endpoint` | `string` | Endpoint de MinIO o S3 | — (obligatorio) |
| `bucket_name` | `string` | Nombre del bucket | `"bloom-filters"` |
| `versioning_enabled` | `bool` | Habilitar versionado del bucket | `true` |
| `lifecycle_retention_days` | `number` | Días de retención de versiones antiguas | `30` |

### 7.2 Variables de salida

| Output | Descripción |
|---|---|
| `bucket_name` | Nombre del bucket creado |
| `bucket_arn` | ARN del bucket (si es S3 AWS) |

### 7.3 Recursos creados

```hcl
resource "minio_s3_bucket" "bloom_filters" {
  bucket = var.bucket_name
  acl    = "private"
}

resource "minio_s3_bucket_versioning" "bloom_filters" {
  bucket = minio_s3_bucket.bloom_filters.bucket

  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Suspended"
  }
}

resource "minio_s3_bucket_lifecycle_configuration" "bloom_filters" {
  bucket = minio_s3_bucket.bloom_filters.bucket

  rule {
    id     = "expire-old-bf-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = var.lifecycle_retention_days
    }
  }
}
```

---

## 8. Ejemplo de Composición en Environment

```hcl
# terraform/environments/production/main.tf

module "kafka_topics" {
  source = "../../modules/kafka-topics"

  kafka_bootstrap_servers   = var.kafka_bootstrap_servers
  replication_factor        = 3
}

module "schema_registry" {
  source = "../../modules/schema-registry"

  schema_registry_url = var.schema_registry_url
  schema_file_path    = "../../../schemas/stolen-vehicle-event-v1.avsc"
}

module "postgresql_canonical" {
  source = "../../modules/postgresql-canonical"

  database_url     = var.database_url
  active_countries = ["CO", "VE", "MX", "AR"]
}

module "redis_hotlist" {
  source = "../../modules/redis-hotlist"

  redis_url              = var.redis_url
  hot_list_ttl_seconds   = 2592000
}

module "mqtt_topics_bf" {
  source = "../../modules/mqtt-topics-bf"

  emqx_api_url                 = var.emqx_api_url
  active_countries             = ["CO", "VE", "MX", "AR"]
  edge_distribution_client_id  = "edge-distribution-service"
}

module "minio_bf_storage" {
  source = "../../modules/minio-bf-storage"

  minio_endpoint          = var.minio_endpoint
  bucket_name             = "bloom-filters"
  lifecycle_retention_days = 30
}
```

```hcl
# terraform/environments/production/terraform.tfvars
kafka_bootstrap_servers = "kafka-broker-1:9092,kafka-broker-2:9092,kafka-broker-3:9092"
schema_registry_url     = "http://schema-registry:8081"
database_url            = "postgresql://antihurto_canonical@pg-primary:5432/antihurto"
redis_url               = "redis://redis-sentinel:26379"
emqx_api_url            = "http://emqx:8081"
minio_endpoint          = "minio:9000"
```

---

## 9. Aplicación

```bash
# Inicializar proveedores
terraform -chdir=environments/production init

# Ver plan de cambios
terraform -chdir=environments/production plan \
  -var-file=terraform.tfvars \
  -out=tfplan

# Aplicar
terraform -chdir=environments/production apply tfplan

# Para un nuevo país (agregar PE a active_countries)
terraform -chdir=environments/production apply \
  -var-file=terraform.tfvars \
  -var='active_countries=["CO","VE","MX","AR","PE"]'
```

---

## 10. Proveedores Terraform Requeridos

| Proveedor | Versión | Módulo que lo usa |
|---|---|---|
| `mongey/kafka` | `~> 0.7` | `kafka-topics` |
| `arkiaconsulting/schemaregistry` | `~> 0.1` | `schema-registry` |
| `cyrilgdn/postgresql` | `~> 1.21` | `postgresql-canonical` |
| `minio/minio` | `~> 2.3` | `minio-bf-storage` |
| `hashicorp/null` | `~> 3.0` | `postgresql-canonical` |

```hcl
# terraform/environments/production/versions.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    kafka = {
      source  = "mongey/kafka"
      version = "~> 0.7"
    }
    schemaregistry = {
      source  = "arkiaconsulting/schemaregistry"
      version = "~> 0.1"
    }
    postgresql = {
      source  = "cyrilgdn/postgresql"
      version = "~> 1.21"
    }
    minio = {
      source  = "minio/minio"
      version = "~> 2.3"
    }
  }
}
```

---

## 11. Referencias

- [`helm/README.md`](../helm/README.md) — Despliegue de los servicios
- [`kafka-topics.md`](../kafka-topics.md) — Especificación de los tópicos
- [`postgresql-schema.md`](../postgresql-schema.md) — DDL de la tabla canonical_vehicles
- [`country-onboarding-guide.md`](../country-onboarding-guide.md) — Proceso de onboarding (incluye paso de provisión Terraform)
- [`docs/ingestion-mqtt/terraform/README.md`](../../ingestion-mqtt/terraform/README.md) — Terraform de la capa MQTT (complementario)
