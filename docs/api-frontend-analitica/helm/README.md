# Helm — API / Frontend / Analítica

**Componente:** `api-frontend-analitica`  
**Versión del documento:** 1.0

---

## 1. Estructura de Subcharts

El chart principal `anti-hurto-api` contiene los siguientes subcharts (uno por microservicio + Superset + API Gateway):

```
charts/anti-hurto-api/
├── Chart.yaml
├── values.yaml                  # valores por defecto
├── values-cloud.yaml            # overrides para cloud (EKS/GKE/AKS)
├── values-onprem.yaml           # overrides para on-prem (RKE2 + MinIO)
├── charts/
│   ├── kong/                    # API Gateway (Kong OSS + KIC)
│   ├── search-service/          # search-service subchart
│   ├── alerts-service/          # alerts-service subchart
│   ├── incident-service/        # incident-service subchart
│   ├── analytics-service/       # analytics-service subchart
│   ├── devices-service/         # devices-service subchart
│   ├── vehicles-service/        # vehicles-service subchart
│   ├── webapp/                  # Web App React (Nginx + static assets)
│   └── superset/                # Apache Superset OSS
└── templates/
    ├── namespace.yaml
    ├── networkpolicies.yaml
    └── rbac.yaml
```

---

## 2. Valores Configurables Principales (`values.yaml`)

```yaml
global:
  # country_code del despliegue (ej. CO, MX, PE)
  countryCode: "CO"
  # Dominio base
  domain: "anti-hurto.internal"
  # Imagen registry
  imageRegistry: "registry.anti-hurto.internal"
  imageTag: "1.0.0"
  # Namespace K8s
  namespace: "anti-hurto"

# ─────────────────────────────────────────────────────────────────
# API Gateway (Kong OSS)
# ─────────────────────────────────────────────────────────────────
kong:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2000m"
      memory: "1Gi"
  # Keycloak JWKS (URLs de los realms por país)
  keycloak:
    jwksUrl: "https://keycloak.anti-hurto.internal/realms/co/protocol/openid-connect/certs"
    issuer: "https://keycloak.anti-hurto.internal/realms/co"
  # Redis para rate-limiting
  redis:
    host: "redis-master.default.svc.cluster.local"
    port: 6379
    database: 1
  # Credenciales en Secret K8s (no en values.yaml)
  existingSecret: "kong-secrets"

# ─────────────────────────────────────────────────────────────────
# search-service
# ─────────────────────────────────────────────────────────────────
searchService:
  enabled: true
  replicaCount: 3
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "512Mi"
  env:
    OPENSEARCH_URL: "https://opensearch.default.svc.cluster.local:9200"
    POSTGRES_HOST: "postgres-primary.default.svc.cluster.local"
    POSTGRES_DB: "anti_hurto"
    OBJECT_STORAGE_ENDPOINT: "https://minio.default.svc.cluster.local"
    OBJECT_STORAGE_BUCKET: "anti-hurto-thumbnails"
    PRESIGN_TTL_SECONDS: "300"
    CIRCUIT_BREAKER_THRESHOLD: "5"
    CIRCUIT_BREAKER_TIMEOUT: "30"
  existingSecret: "search-service-secrets"  # POSTGRES_PASSWORD, S3_SECRET_KEY

# ─────────────────────────────────────────────────────────────────
# alerts-service
# ─────────────────────────────────────────────────────────────────
alertsService:
  enabled: true
  replicaCount: 3
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "512Mi"
  env:
    KAFKA_BROKERS: "kafka-0.kafka.default.svc.cluster.local:9092,kafka-1.kafka.default.svc.cluster.local:9092,kafka-2.kafka.default.svc.cluster.local:9092"
    KAFKA_TOPIC_ALERTS: "vehicle.alerts"
    KAFKA_GROUP_ID: "alerts-service-group"
    REDIS_URL: "redis://redis-master.default.svc.cluster.local:6379/2"
    WS_HEARTBEAT_INTERVAL: "25000"
    WS_HEARTBEAT_TIMEOUT: "300000"
  existingSecret: "alerts-service-secrets"  # KAFKA_SASL_PASSWORD, REDIS_PASSWORD
  # Sticky sessions no requeridas con Redis adapter, pero recomendadas
  service:
    sessionAffinity: "ClientIP"

# ─────────────────────────────────────────────────────────────────
# incident-service
# ─────────────────────────────────────────────────────────────────
incidentService:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  env:
    POSTGRES_HOST: "postgres-primary.default.svc.cluster.local"
    POSTGRES_DB: "anti_hurto"
    KAFKA_BROKERS: "kafka-0.kafka.default.svc.cluster.local:9092"
    KAFKA_TOPIC_INCIDENTS: "incident.events"
    COUNTRY_SYNC_RETRY_INTERVAL_MINUTES: "5"
    COUNTRY_SYNC_MAX_RETRIES: "288"  # 24 horas
  existingSecret: "incident-service-secrets"

# ─────────────────────────────────────────────────────────────────
# analytics-service
# ─────────────────────────────────────────────────────────────────
analyticsService:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "512Mi"
  env:
    CLICKHOUSE_URL: "http://clickhouse.default.svc.cluster.local:8123"
    CLICKHOUSE_DATABASE: "anti_hurto"
    CLICKHOUSE_MAX_EXECUTION_TIME: "30"
    STALE_DATA_THRESHOLD_MINUTES: "60"
  existingSecret: "analytics-service-secrets"

# ─────────────────────────────────────────────────────────────────
# devices-service
# ─────────────────────────────────────────────────────────────────
devicesService:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"
  env:
    POSTGRES_HOST: "postgres-primary.default.svc.cluster.local"
    POSTGRES_DB: "anti_hurto"
    MQTT_BROKER_URL: "mqtts://emqx.default.svc.cluster.local:8883"
  existingSecret: "devices-service-secrets"

# ─────────────────────────────────────────────────────────────────
# vehicles-service
# ─────────────────────────────────────────────────────────────────
vehiclesService:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"
  env:
    POSTGRES_HOST: "postgres-primary.default.svc.cluster.local"
    POSTGRES_DB: "anti_hurto"
    REDIS_URL: "redis://redis-master.default.svc.cluster.local:6379/0"
    OWNER_DNI_HMAC_KEY_SECRET_NAME: "vehicles-hmac-key"
  existingSecret: "vehicles-service-secrets"

# ─────────────────────────────────────────────────────────────────
# Web App (React + Nginx)
# ─────────────────────────────────────────────────────────────────
webapp:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: "50m"
      memory: "64Mi"
    limits:
      cpu: "200m"
      memory: "128Mi"
  env:
    VITE_API_BASE_URL: "https://api.anti-hurto.internal/v1"
    VITE_ALERTS_WS_URL: "wss://api.anti-hurto.internal/alerts"
    VITE_KEYCLOAK_URL: "https://keycloak.anti-hurto.internal"
    VITE_KEYCLOAK_REALM: "co"
    # Tile provider MapLibre GL JS
    VITE_MAP_STYLE_URL: "https://tiles.openstreetmap.org/styles/basic/style.json"
    VITE_SUPERSET_EMBED_URL: "https://superset.anti-hurto.internal"
    VITE_ENV: "production"

# ─────────────────────────────────────────────────────────────────
# Apache Superset
# ─────────────────────────────────────────────────────────────────
superset:
  enabled: true
  replicaCount: 2
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2000m"
      memory: "4Gi"
  existingSecret: "superset-secrets"

# ─────────────────────────────────────────────────────────────────
# Ingress TLS (cert-manager + Vault PKI)
# ─────────────────────────────────────────────────────────────────
ingress:
  enabled: true
  className: "kong"
  annotations:
    cert-manager.io/cluster-issuer: "vault-pki"
  tls: true
  hosts:
    api: "api.anti-hurto.internal"
    webapp: "app.anti-hurto.internal"
    superset: "superset.anti-hurto.internal"
```

---

## 3. Diferencias Cloud vs. On-Prem

### `values-cloud.yaml` (EKS / GKE / AKS)

```yaml
searchService:
  env:
    OBJECT_STORAGE_ENDPOINT: ""  # vacío: usa SDK nativo S3/GCS/Azure
    # El adapter detecta el proveedor vía AWS_REGION / GOOGLE_CLOUD_PROJECT / AZURE_*

webapp:
  env:
    VITE_MAP_STYLE_URL: "https://api.protomaps.com/tiles/v3/{z}/{x}/{y}.pbf?key=${PROTOMAPS_KEY}"
```

### `values-onprem.yaml` (RKE2 + MinIO + Vault PKI local)

```yaml
searchService:
  env:
    OBJECT_STORAGE_ENDPOINT: "https://minio.storage.svc.cluster.local"
    OBJECT_STORAGE_FORCE_PATH_STYLE: "true"  # MinIO requiere path-style

webapp:
  env:
    VITE_MAP_STYLE_URL: "http://martin.tiles.svc.cluster.local/style.json"
    # martin = tile server OSS auto-hosteado con datos OSM

ingress:
  annotations:
    cert-manager.io/cluster-issuer: "vault-pki-internal"  # PKI interna on-prem
```

---

## 4. CPU y Memoria Recomendados por Servicio

| Servicio | CPU Request | CPU Limit | Memoria Request | Memoria Limit | Replicas |
|---|---|---|---|---|---|
| Kong (API Gateway) | 500m | 2000m | 512Mi | 1Gi | 2 |
| search-service | 250m | 1000m | 256Mi | 512Mi | 3 |
| alerts-service | 250m | 1000m | 256Mi | 512Mi | 3 |
| incident-service | 250m | 500m | 256Mi | 512Mi | 2 |
| analytics-service | 250m | 1000m | 256Mi | 512Mi | 2 |
| devices-service | 100m | 500m | 128Mi | 256Mi | 2 |
| vehicles-service | 100m | 500m | 128Mi | 256Mi | 2 |
| Web App (Nginx) | 50m | 200m | 64Mi | 128Mi | 2 |
| Apache Superset | 500m | 2000m | 1Gi | 4Gi | 2 |

---

## 5. Instrucciones de Instalación / Actualización / Rollback

### Instalación inicial

```bash
# 1. Crear namespace y secrets (no en valores Helm)
kubectl create namespace anti-hurto

kubectl create secret generic search-service-secrets \
  --namespace anti-hurto \
  --from-literal=POSTGRES_PASSWORD=<pass> \
  --from-literal=S3_SECRET_KEY=<key>

# (Repetir para cada servicio)

# 2. Instalar el chart
helm install anti-hurto-api ./charts/anti-hurto-api \
  --namespace anti-hurto \
  --values values.yaml \
  --values values-cloud.yaml \
  --timeout 15m \
  --wait
```

### Actualización

```bash
helm upgrade anti-hurto-api ./charts/anti-hurto-api \
  --namespace anti-hurto \
  --values values.yaml \
  --values values-cloud.yaml \
  --timeout 15m \
  --wait
```

### Rollback

```bash
# Ver historial
helm history anti-hurto-api --namespace anti-hurto

# Rollback a revisión anterior
helm rollback anti-hurto-api <revision> --namespace anti-hurto --wait
```

---

## 6. Referencias

- [terraform/README.md](../terraform/README.md)
- [api-gateway.md](../api-gateway.md)
- [superset-integration.md](../superset-integration.md)
- [adr-cartography.md](../adr-cartography.md) — configuración de tiles en Helm
