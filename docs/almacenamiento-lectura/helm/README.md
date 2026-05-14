# Helm Charts — Almacenamiento de Lectura

**Componente:** Almacenamiento de lectura — despliegue Kubernetes  
**Versión del documento:** 1.0  
**Referencia:** [postgresql-ha.md](../postgresql-ha.md) · [adr-redis-topology.md](../adr-redis-topology.md) · [terraform/README.md](../terraform/README.md)

---

## 1. Estructura de Charts

El almacenamiento de lectura se despliega mediante charts Helm estándar con values personalizados por entorno (on-prem vs. cloud, por país):

```
helm/
├── postgresql/          # bitnami/postgresql-ha (Patroni on-prem)
│   ├── values-base.yaml
│   ├── values-onprem.yaml
│   └── values-rds.yaml
├── opensearch/          # opensearch-project/opensearch
│   ├── values-base.yaml
│   └── values-cloud.yaml
├── clickhouse/          # altinity/clickhouse-operator
│   ├── values-base.yaml
│   └── values-ha.yaml
├── redis/               # bitnami/redis
│   ├── values-base.yaml
│   ├── values-sentinel.yaml
│   └── values-cluster.yaml
└── kafka-connect/       # debezium/kafka-connect (Debezium)
    ├── values-base.yaml
    └── connectors/
        └── debezium-vehicle-events.json
```

---

## 2. Values Configurables por Almacén

### 2.1 PostgreSQL

```yaml
# values-base.yaml (PostgreSQL)
global:
  postgresql:
    database: antihurto
    existingSecret: pg-credentials
    secretKeys:
      adminPasswordKey: postgres-password
      userPasswordKey: app-password

postgresql:
  replicaCount: 2
  resources:
    requests:
      memory: 8Gi
      cpu: "2"
    limits:
      memory: 16Gi
      cpu: "4"
  persistence:
    enabled: true
    size: 500Gi
    storageClass: nvme-ssd

postgis:
  enabled: true          # Instala postgis junto con postgresql

pgPartman:
  enabled: true          # Instala y configura pg_partman

pgcrypto:
  enabled: true          # Habilita pgcrypto para campos PII

initScripts:
  enabled: true
  scriptsConfigmap: pg-init-scripts  # ConfigMap con el DDL de postgresql-schema.md
```

```yaml
# values-onprem.yaml (Patroni)
mode: patroni
patroni:
  enabled: true
  etcd:
    endpoints:
      - etcd-0:2379
      - etcd-1:2379
      - etcd-2:2379
  parameters:
    wal_level: logical
    max_replication_slots: "5"
    max_wal_senders: "10"
    wal_keep_size: "1024"

haproxy:
  enabled: true
  service:
    type: ClusterIP
  ports:
    write: 5432
    read: 5433
```

```yaml
# values-rds.yaml (AWS RDS Multi-AZ — solo connection string)
mode: external
externalDatabase:
  host: antihurto.xxxx.us-east-1.rds.amazonaws.com
  port: 5432
  existingSecret: rds-credentials
  database: antihurto
```

### 2.2 OpenSearch

```yaml
# values-base.yaml (OpenSearch)
opensearch:
  version: "2.13.0"
  replicas: 3
  resources:
    requests:
      memory: 8Gi
      cpu: "2"
    limits:
      memory: 16Gi
      cpu: "4"
  persistence:
    enabled: true
    size: 1Ti
    storageClass: ssd

  javaOpts: "-Xmx4g -Xms4g"

  config:
    opensearch.yml: |
      cluster.name: antihurto-os
      network.host: 0.0.0.0
      plugins.security.disabled: false

  extraEnvs:
    - name: OPENSEARCH_JAVA_OPTS
      value: "-Xms4g -Xmx4g"

# Provisioning de índice template e ILM via init job
initJob:
  enabled: true
  image: curlimages/curl:latest
  configmaps:
    - name: os-init-scripts
      mountPath: /scripts
```

```yaml
# values-cloud.yaml (AWS OpenSearch Service)
mode: external
externalEndpoint:
  host: antihurto.us-east-1.es.amazonaws.com
  port: 443
  protocol: https
  existingSecret: opensearch-credentials
```

### 2.3 ClickHouse

```yaml
# values-base.yaml (ClickHouse Operator)
clickhouse:
  operator:
    enabled: true

  cluster:
    name: antihurto-ch
    layout:
      shardsCount: 3
      replicasCount: 2

  zookeeper:
    enabled: true
    replicas: 3

  resources:
    requests:
      memory: 16Gi
      cpu: "4"
    limits:
      memory: 32Gi
      cpu: "8"

  persistence:
    enabled: true
    size: 2Ti
    storageClass: nvme-ssd

  settings:
    max_concurrent_queries: 100
    max_memory_usage: "32000000000"
    background_pool_size: 16

# DDL se aplica via post-install hook
initJob:
  enabled: true
  scripts:
    - clickhouse-schema.sql
    - clickhouse-h3-views.sql
```

### 2.4 Redis

```yaml
# values-sentinel.yaml (Redis Sentinel)
architecture: replication

auth:
  enabled: true
  existingSecret: redis-credentials
  existingSecretPasswordKey: redis-password

sentinel:
  enabled: true
  quorum: 2
  downAfterMilliseconds: 5000
  failoverTimeout: 18000
  masterSet: antihurto-master

replica:
  replicaCount: 2
  resources:
    requests:
      memory: 4Gi
      cpu: "1"
    limits:
      memory: 8Gi
      cpu: "2"
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: kubernetes.io/hostname

persistence:
  enabled: true
  size: 20Gi
  storageClass: ssd

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
    interval: 15s

configmap: |
  save 900 1
  save 300 10
  appendonly yes
  appendfsync everysec
  no-appendfsync-on-rewrite yes
  maxmemory-policy allkeys-lru
```

---

## 3. Diferencias Cloud vs. On-Prem

| Aspecto | On-Prem (RKE2/Patroni) | Cloud Managed |
|---|---|---|
| PostgreSQL HA | Patroni + etcd + HAProxy (Helm) | RDS Multi-AZ / CloudSQL HA (values-rds.yaml) |
| OpenSearch | Chart Helm con StatefulSet | AWS OpenSearch Service (values-cloud.yaml) |
| ClickHouse | ClickHouse Operator + ZooKeeper | ClickHouse Cloud (conector externo) |
| Redis | bitnami/redis Sentinel (chart Helm) | ElastiCache Sentinel (values-cloud.yaml) |
| Storage class | nvme-ssd (Rook-Ceph / Longhorn) | gp3 (AWS) / premium-ssd (Azure) |
| Backups | pgBackRest + S3-compatible | RDS PITR automático |
| TLS entre pods | cert-manager + Vault PKI | ACM / Cloud Certificate Manager |

---

## 4. Instalación

### 4.1 Prerequisitos

```bash
# Añadir repositorios de charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add opensearch-project https://opensearch-project.github.io/helm-charts/
helm repo add altinity https://altinity.github.io/clickhouse-operator/
helm repo update
```

### 4.2 Orden de Instalación

El orden importa: PostgreSQL debe estar disponible antes de aplicar el schema y antes de configurar Debezium.

```bash
# 1. Namespace
kubectl create namespace antihurto-storage

# 2. Secrets
kubectl apply -f secrets/ -n antihurto-storage

# 3. PostgreSQL (primero, es fuente de verdad)
helm upgrade --install postgresql bitnami/postgresql-ha \
  -f helm/postgresql/values-base.yaml \
  -f helm/postgresql/values-onprem.yaml \
  -n antihurto-storage \
  --wait --timeout 10m

# 4. Redis (necesario para el Matcher Service)
helm upgrade --install redis bitnami/redis \
  -f helm/redis/values-base.yaml \
  -f helm/redis/values-sentinel.yaml \
  -n antihurto-storage \
  --wait --timeout 5m

# 5. OpenSearch
helm upgrade --install opensearch opensearch-project/opensearch \
  -f helm/opensearch/values-base.yaml \
  -n antihurto-storage \
  --wait --timeout 15m

# 6. ClickHouse (necesita ZooKeeper interno o externo)
helm upgrade --install clickhouse-operator altinity/clickhouse-operator \
  -n antihurto-storage \
  --wait

kubectl apply -f helm/clickhouse/clickhouse-cluster.yaml -n antihurto-storage

# 7. Kafka Connect + Debezium (después de PostgreSQL y Kafka)
helm upgrade --install kafka-connect bitnami/kafka-connect \
  -f helm/kafka-connect/values-base.yaml \
  -n antihurto-storage \
  --wait --timeout 5m

# 8. Registrar el conector Debezium
kubectl exec -n antihurto-storage deploy/kafka-connect -- \
  curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @/connectors/debezium-vehicle-events.json
```

### 4.3 Provisión de Extensiones PostgreSQL

```bash
# El init job aplica el DDL (extensiones + tablas) al completar la instalación de PostgreSQL
kubectl logs -n antihurto-storage job/postgresql-init -f

# Verificar que las extensiones están activas:
kubectl exec -n antihurto-storage postgresql-primary-0 -- \
  psql -U postgres -d antihurto -c "\dx"
# Debe mostrar: postgis, pg_partman, pgcrypto, btree_gist
```

---

## 5. Actualización

```bash
# Actualizar un chart (ej. Redis)
helm upgrade redis bitnami/redis \
  -f helm/redis/values-base.yaml \
  -f helm/redis/values-sentinel.yaml \
  -n antihurto-storage \
  --atomic --timeout 5m
# --atomic hace rollback automático si la actualización falla
```

---

## 6. Rollback

```bash
# Ver historial de releases
helm history redis -n antihurto-storage

# Rollback a la revisión anterior
helm rollback redis 2 -n antihurto-storage --wait

# Rollback a una revisión específica
helm rollback postgresql 1 -n antihurto-storage --wait --timeout 10m
```

---

## 7. Values por País (Multi-Tenant Regional)

Para países con requisito de residencia de datos en región específica:

```bash
# Desplegar almacenamiento dedicado para Colombia en región sa-east-1
helm upgrade --install postgresql-co bitnami/postgresql-ha \
  -f helm/postgresql/values-base.yaml \
  -f helm/postgresql/values-onprem.yaml \
  -f helm/postgresql/values-country-co.yaml \
  -n antihurto-co \
  --set "global.postgresql.database=antihurto_co"
```

```yaml
# values-country-co.yaml — override específico para Colombia
postgresql:
  nodeSelector:
    topology.kubernetes.io/region: sa-east-1
  tolerations:
    - key: "dedicated"
      value: "co-storage"
      effect: "NoSchedule"
  persistence:
    storageClass: nvme-ssd-sa-east-1
```
