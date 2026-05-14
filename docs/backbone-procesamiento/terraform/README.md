# Backbone de Procesamiento — Módulos Terraform

**Componente:** backbone-procesamiento  
**Versión del documento:** 1.0  
**Última actualización:** 2026-05-13

---

## 1. Estructura de Módulos

Los módulos Terraform del backbone de procesamiento provisionan la infraestructura necesaria para que los microservicios operen. Están organizados por recurso gestionado:

```
terraform/backbone-procesamiento/
├── main.tf                    # Composición de módulos
├── variables.tf               # Variables de entrada del módulo raíz
├── outputs.tf                 # Salidas del módulo raíz
├── versions.tf                # Restricciones de versión de Terraform y providers
├── modules/
│   ├── kafka-topics/          # Tópicos Kafka del pipeline
│   ├── redis-credentials/     # Credenciales Redis (via Vault)
│   ├── postgresql-extensions/ # Extensiones PostGIS + roles
│   ├── opensearch-templates/  # Index templates e ILM OpenSearch
│   └── debezium-connect/      # Kafka Connect cluster + conector Debezium
└── environments/
    ├── staging/
    │   ├── main.tf
    │   └── terraform.tfvars
    └── production/
        ├── main.tf
        └── terraform.tfvars
```

---

## 2. Módulo: `kafka-topics`

Provisiona los 6 tópicos del pipeline con sus configuraciones de replication factor, retención y dead-letter topics.

### Variables de entrada

| Variable | Tipo | Default | Descripción |
|---|---|---|---|
| `kafka_bootstrap_servers` | `string` | — | Lista de brokers Kafka (obligatorio). |
| `replication_factor` | `number` | `3` | Factor de replicación para todos los tópicos. |
| `min_insync_replicas` | `number` | `2` | Mínimo de réplicas sincronizadas. |
| `raw_events_partitions` | `number` | `24` | Particiones para `vehicle.events.raw`. |
| `enriched_events_partitions` | `number` | `24` | Particiones para `vehicle.events.deduped` y `vehicle.events.enriched`. |
| `alerts_partitions` | `number` | `12` | Particiones para `vehicle.alerts`. |
| `stolen_vehicles_partitions` | `number` | `12` | Particiones para `stolen.vehicles.events`. |
| `cdc_partitions` | `number` | `12` | Particiones para `pg.events.cdc`. |
| `dlq_retention_ms` | `number` | `604800000` | Retención de los DLQ en ms (default 7 días). |

### Recursos creados

```hcl
# modules/kafka-topics/main.tf
resource "confluent_kafka_topic" "vehicle_events_raw" {
  kafka_cluster { id = var.kafka_cluster_id }
  topic_name         = "vehicle.events.raw"
  partitions_count   = var.raw_events_partitions
  rest_endpoint      = var.kafka_rest_endpoint

  config = {
    "cleanup.policy"             = "delete"
    "retention.ms"               = "86400000"    # 24 h
    "replication.factor"         = tostring(var.replication_factor)
    "min.insync.replicas"        = tostring(var.min_insync_replicas)
    "compression.type"           = "lz4"
  }
}

resource "confluent_kafka_topic" "vehicle_events_raw_dlq" {
  kafka_cluster { id = var.kafka_cluster_id }
  topic_name       = "vehicle.events.raw.dlq"
  partitions_count = var.raw_events_partitions
  config = {
    "cleanup.policy"      = "delete"
    "retention.ms"        = tostring(var.dlq_retention_ms)
    "replication.factor"  = tostring(var.replication_factor)
    "min.insync.replicas" = tostring(var.min_insync_replicas)
  }
}

# ... recursos similares para los 5 tópicos restantes y sus DLQs
```

### Salidas

| Output | Descripción |
|---|---|
| `topic_names` | Map con los nombres de todos los tópicos creados. |
| `dlq_topic_names` | Map con los nombres de todos los DLQ creados. |

---

## 3. Módulo: `redis-credentials`

Provisiona las credenciales de acceso a Redis para los microservicios del backbone, usando Vault como gestor de secretos.

### Variables de entrada

| Variable | Tipo | Default | Descripción |
|---|---|---|---|
| `vault_addr` | `string` | — | URL del servidor Vault. |
| `vault_namespace` | `string` | `""` | Namespace de Vault (si aplica). |
| `redis_sentinel_addr` | `string` | — | Dirección del Redis Sentinel. |
| `redis_master_name` | `string` | `"mymaster"` | Nombre del master en Sentinel. |
| `service_names` | `list(string)` | `["matcher-service", "alert-service", "websocket-gateway"]` | Servicios que necesitan acceso a Redis. |

### Recursos creados

```hcl
# modules/redis-credentials/main.tf

# Política de Vault para acceso a secretos Redis del backbone
resource "vault_policy" "backbone_redis" {
  name = "backbone-redis-read"
  policy = <<EOT
path "secret/data/backbone/redis/*" {
  capabilities = ["read"]
}
EOT
}

# Secretos Redis almacenados en Vault KV v2
resource "vault_kv_secret_v2" "redis_config" {
  mount = "secret"
  name  = "backbone/redis/config"

  data_json = jsonencode({
    sentinel_addr = var.redis_sentinel_addr
    master_name   = var.redis_master_name
    password      = random_password.redis_password.result
  })
}

# ExternalSecret (via ESO) para inyectar los secretos en Kubernetes
resource "kubernetes_manifest" "redis_external_secret" {
  for_each = toset(var.service_names)

  manifest = {
    apiVersion = "external-secrets.io/v1beta1"
    kind       = "ExternalSecret"
    metadata = {
      name      = "${each.value}-redis-secret"
      namespace = "backbone-procesamiento"
    }
    spec = {
      refreshInterval = "1h"
      secretStoreRef = {
        name = "vault-backend"
        kind = "ClusterSecretStore"
      }
      target = {
        name           = "${each.value}-redis"
        creationPolicy = "Owner"
      }
      data = [
        {
          secretKey = "REDIS_PASSWORD"
          remoteRef = {
            key      = "backbone/redis/config"
            property = "password"
          }
        }
      ]
    }
  }
}
```

### Salidas

| Output | Descripción |
|---|---|
| `vault_policy_name` | Nombre de la política de Vault creada. |
| `redis_secret_names` | Map de nombres de ExternalSecrets creados por servicio. |

---

## 4. Módulo: `postgresql-extensions`

Habilita la extensión PostGIS, crea los roles de acceso y configura la publicación lógica para Debezium.

### Variables de entrada

| Variable | Tipo | Default | Descripción |
|---|---|---|---|
| `postgres_host` | `string` | — | Host del servidor PostgreSQL. |
| `postgres_port` | `number` | `5432` | Puerto PostgreSQL. |
| `postgres_database` | `string` | `"antihurto"` | Nombre de la base de datos. |
| `postgres_admin_user` | `string` | `"postgres"` | Usuario administrador (para aplicar extensiones). |
| `debezium_user_password` | `string` | — | Contraseña para el usuario `debezium_user` (sensible, inyectado desde Vault). |
| `app_writer_password` | `string` | — | Contraseña para el rol `app_writer`. |
| `app_reader_password` | `string` | — | Contraseña para el rol `app_reader`. |

### Recursos creados

```hcl
# modules/postgresql-extensions/main.tf

resource "postgresql_extension" "postgis" {
  name     = "postgis"
  database = var.postgres_database
}

resource "postgresql_extension" "uuid_ossp" {
  name     = "uuid-ossp"
  database = var.postgres_database
}

resource "postgresql_role" "app_writer" {
  name     = "app_writer"
  login    = true
  password = var.app_writer_password
}

resource "postgresql_role" "app_reader" {
  name     = "app_reader"
  login    = true
  password = var.app_reader_password
}

resource "postgresql_role" "debezium_user" {
  name        = "debezium_user"
  login       = true
  replication = true
  password    = var.debezium_user_password
}

resource "postgresql_grant" "app_writer_grants" {
  database    = var.postgres_database
  role        = postgresql_role.app_writer.name
  schema      = "public"
  object_type = "table"
  privileges  = ["INSERT", "UPDATE"]
  objects     = ["vehicle_events", "stolen_vehicles", "alerts", "incidents", "incident_audit"]
}

resource "postgresql_grant" "app_reader_grants" {
  database    = var.postgres_database
  role        = postgresql_role.app_reader.name
  schema      = "public"
  object_type = "table"
  privileges  = ["SELECT"]
  objects     = ["vehicle_events", "stolen_vehicles", "alerts", "incidents", "alert_zones"]
}

# La publicación lógica para Debezium se aplica via null_resource + psql
# ya que el provider postgresql no soporta CREATE PUBLICATION directamente
resource "null_resource" "debezium_publication" {
  triggers = { hash = sha256(join(",", ["vehicle_events", "stolen_vehicles", "alerts", "incidents"])) }

  provisioner "local-exec" {
    command = <<EOF
psql postgresql://${var.postgres_admin_user}@${var.postgres_host}:${var.postgres_port}/${var.postgres_database} \
  -c "CREATE PUBLICATION debezium_publication FOR TABLE vehicle_events, stolen_vehicles, alerts, incidents;"
EOF
  }
}
```

### Salidas

| Output | Descripción |
|---|---|
| `postgis_version` | Versión de PostGIS instalada. |
| `roles_created` | Lista de roles creados (`app_writer`, `app_reader`, `debezium_user`). |

---

## 5. Módulo: `opensearch-templates`

Crea las index templates y políticas ILM en OpenSearch para los índices del backbone.

### Variables de entrada

| Variable | Tipo | Default | Descripción |
|---|---|---|---|
| `opensearch_url` | `string` | — | URL del clúster OpenSearch. |
| `opensearch_username` | `string` | — | Usuario de administración. |
| `country_codes` | `list(string)` | `["co", "ve", "mx"]` | Códigos de país en minúsculas para crear index patterns. |
| `ilm_hot_phase_max_age` | `string` | `"30d"` | Duración de la fase hot en ILM. |
| `ilm_warm_phase_enabled` | `bool` | `true` | Habilitar fase warm en ILM. |
| `ilm_delete_after` | `string` | `"730d"` | Eliminación del índice (24 meses). |
| `number_of_shards` | `number` | `2` | Shards por índice. |
| `number_of_replicas` | `number` | `1` | Réplicas por shard. |

### Recursos creados

```hcl
# modules/opensearch-templates/main.tf

# Index template para vehicle-events-*
resource "opensearch_index_template" "vehicle_events" {
  name = "vehicle-events-template"

  body = jsonencode({
    index_patterns = ["vehicle-events-*"]
    template = {
      settings = {
        number_of_shards   = var.number_of_shards
        number_of_replicas = var.number_of_replicas
        "index.lifecycle.name" = "vehicle-events-lifecycle"
      }
      mappings = {
        properties = {
          event_id          = { type = "keyword" }
          country_code      = { type = "keyword" }
          device_id         = { type = "keyword" }
          plate_raw         = { type = "text" }
          plate_normalized  = { type = "keyword" }
          event_ts          = { type = "date" }
          received_ts       = { type = "date" }
          confidence        = { type = "float" }
          location          = { type = "geo_point" }
          geocoding_status  = { type = "keyword" }
          street            = { type = "text", analyzer = "spanish" }
          city              = { type = "keyword" }
          country           = { type = "keyword" }
          image_uri         = { type = "keyword", index = false }
          thumbnail_uri     = { type = "keyword", index = false }
          clock_uncertain   = { type = "boolean" }
          enrichment_version = { type = "short" }
        }
      }
    }
  })
}

# Política ILM para gestión del ciclo de vida de los índices
resource "opensearch_index_lifecycle_policy" "vehicle_events" {
  name = "vehicle-events-lifecycle"

  body = jsonencode({
    policy = {
      phases = {
        hot = {
          min_age = "0ms"
          actions = {
            rollover = {
              max_age  = var.ilm_hot_phase_max_age
              max_size = "50gb"
            }
          }
        }
        warm = {
          min_age = "30d"
          actions = {
            forcemerge = { max_num_segments = 1 }
            shrink     = { number_of_shards = 1 }
          }
        }
        delete = {
          min_age = var.ilm_delete_after
          actions = { delete = {} }
        }
      }
    }
  })
}
```

### Salidas

| Output | Descripción |
|---|---|
| `template_name` | Nombre del index template creado. |
| `ilm_policy_name` | Nombre de la política ILM creada. |

---

## 6. Módulo: `debezium-connect`

Configura el clúster Kafka Connect y registra el conector Debezium PostgreSQL.

### Variables de entrada

| Variable | Tipo | Default | Descripción |
|---|---|---|---|
| `kafka_connect_url` | `string` | — | URL del API REST de Kafka Connect. |
| `connector_name` | `string` | `"debezium-pg-connector"` | Nombre del conector. |
| `postgres_hostname` | `string` | — | Host de PostgreSQL. |
| `postgres_database` | `string` | `"antihurto"` | Base de datos a monitorear. |
| `postgres_slot_name` | `string` | `"debezium_slot_events"` | Nombre del slot de replicación lógica. |
| `postgres_publication_name` | `string` | `"debezium_publication"` | Nombre de la publicación PostgreSQL. |
| `schema_registry_url` | `string` | — | URL del Schema Registry. |
| `output_topic_prefix` | `string` | `"pg"` | Prefijo del tópico CDC. |
| `heartbeat_interval_ms` | `number` | `10000` | Intervalo del heartbeat Debezium. |
| `errors_tolerance` | `string` | `"all"` | Tolerancia a errores (`none` o `all`). |

### Recursos creados

```hcl
# modules/debezium-connect/main.tf

resource "kafka_connect_connector" "debezium_pg" {
  name = var.connector_name

  config = {
    "connector.class"                     = "io.debezium.connector.postgresql.PostgresConnector"
    "database.hostname"                   = var.postgres_hostname
    "database.port"                       = "5432"
    "database.user"                       = "debezium_user"
    "database.password"                   = var.postgres_password   # sensible, via Vault
    "database.dbname"                     = var.postgres_database
    "database.server.name"                = "antihurto-pg"
    "plugin.name"                         = "pgoutput"
    "slot.name"                           = var.postgres_slot_name
    "publication.name"                    = var.postgres_publication_name
    "table.include.list"                  = "public.vehicle_events,public.stolen_vehicles,public.alerts,public.incidents"
    "heartbeat.interval.ms"               = tostring(var.heartbeat_interval_ms)
    "topic.prefix"                        = var.output_topic_prefix
    "key.converter"                       = "io.confluent.connect.avro.AvroConverter"
    "key.converter.schema.registry.url"   = var.schema_registry_url
    "value.converter"                     = "io.confluent.connect.avro.AvroConverter"
    "value.converter.schema.registry.url" = var.schema_registry_url
    "producer.override.acks"              = "all"
    "errors.tolerance"                    = var.errors_tolerance
    "errors.deadletterqueue.topic.name"   = "pg.events.cdc.dlq"
    "errors.deadletterqueue.context.headers.enable" = "true"
    "snapshot.mode"                       = "initial"
  }
}
```

### Salidas

| Output | Descripción |
|---|---|
| `connector_name` | Nombre del conector Debezium registrado. |
| `output_topics` | Lista de tópicos CDC generados (uno por tabla monitoreada). |

---

## 7. Composición del Módulo Raíz

```hcl
# main.tf — Módulo raíz backbone-procesamiento

module "kafka_topics" {
  source                   = "./modules/kafka-topics"
  kafka_bootstrap_servers  = var.kafka_bootstrap_servers
  kafka_cluster_id         = var.kafka_cluster_id
  kafka_rest_endpoint      = var.kafka_rest_endpoint
  replication_factor       = 3
  min_insync_replicas      = 2
}

module "redis_credentials" {
  source               = "./modules/redis-credentials"
  vault_addr           = var.vault_addr
  redis_sentinel_addr  = var.redis_sentinel_addr
  redis_master_name    = var.redis_master_name
}

module "postgresql_extensions" {
  source                    = "./modules/postgresql-extensions"
  postgres_host             = var.postgres_host
  postgres_database         = var.postgres_database
  debezium_user_password    = data.vault_generic_secret.debezium.data["password"]
  app_writer_password       = data.vault_generic_secret.app_writer.data["password"]
  app_reader_password       = data.vault_generic_secret.app_reader.data["password"]
  depends_on               = [module.kafka_topics]
}

module "opensearch_templates" {
  source           = "./modules/opensearch-templates"
  opensearch_url   = var.opensearch_url
  country_codes    = var.country_codes
  ilm_delete_after = "730d"   # 24 meses (retención por defecto)
}

module "debezium_connect" {
  source                  = "./modules/debezium-connect"
  kafka_connect_url       = var.kafka_connect_url
  postgres_hostname       = var.postgres_host
  postgres_database       = var.postgres_database
  postgres_password       = data.vault_generic_secret.debezium.data["password"]
  schema_registry_url     = var.schema_registry_url
  depends_on              = [module.postgresql_extensions, module.kafka_topics]
}
```

---

## 8. Variables de Entrada del Módulo Raíz

| Variable | Tipo | Obligatoria | Descripción |
|---|---|---|---|
| `environment` | `string` | Sí | `staging` o `production`. |
| `kafka_bootstrap_servers` | `string` | Sí | Brokers Kafka. |
| `kafka_cluster_id` | `string` | Sí | ID del clúster Kafka (Confluent). |
| `kafka_rest_endpoint` | `string` | Sí | REST endpoint del clúster Kafka. |
| `postgres_host` | `string` | Sí | Host PostgreSQL. |
| `postgres_database` | `string` | No | Default: `antihurto`. |
| `redis_sentinel_addr` | `string` | Sí | Dirección Redis Sentinel. |
| `redis_master_name` | `string` | No | Default: `mymaster`. |
| `opensearch_url` | `string` | Sí | URL OpenSearch. |
| `kafka_connect_url` | `string` | Sí | URL Kafka Connect REST API. |
| `schema_registry_url` | `string` | Sí | URL Schema Registry. |
| `vault_addr` | `string` | Sí | URL Vault. |
| `country_codes` | `list(string)` | Sí | Lista de country codes activos (minúsculas). |

---

## 9. Salidas del Módulo Raíz

| Output | Descripción |
|---|---|
| `kafka_topic_names` | Map completo de nombres de tópicos creados. |
| `opensearch_template_name` | Nombre del index template OpenSearch. |
| `debezium_connector_name` | Nombre del conector Debezium registrado. |

---

## 10. Procedimiento de Aplicación

```bash
# 1. Inicializar Terraform
terraform init

# 2. Verificar el plan de cambios
terraform plan \
  -var-file="environments/production/terraform.tfvars" \
  -out=tfplan

# 3. Revisar el plan (nunca aplicar sin revisar)
terraform show tfplan

# 4. Aplicar los cambios
terraform apply tfplan

# 5. Verificar las salidas
terraform output
```

> **Importante:** los secretos (contraseñas de PostgreSQL, Redis) nunca se almacenan en archivos `.tfvars` bajo control de versiones. Se obtienen de Vault en tiempo de `plan`/`apply` mediante el provider `hashicorp/vault` configurado con AppRole o Kubernetes Auth Method.
