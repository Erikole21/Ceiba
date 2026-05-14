# Backbone de Procesamiento — Despliegue con Helm

**Componente:** backbone-procesamiento  
**Versión del documento:** 1.0  
**Última actualización:** 2026-05-13

---

## 1. Estructura del Chart

El backbone de procesamiento se despliega como un **umbrella chart** (`backbone-procesamiento`) que orquesta los subchart de cada microservicio:

```
helm/
├── backbone-procesamiento/
│   ├── Chart.yaml
│   ├── values.yaml               # Values por defecto (ajustar por entorno)
│   ├── values-staging.yaml       # Overrides para staging
│   ├── values-production.yaml    # Overrides para producción
│   └── charts/
│       ├── deduplicator/
│       ├── enrichment-service/
│       ├── matcher-service/
│       ├── alert-service/
│       ├── websocket-gateway/
│       ├── incident-service/
│       └── debezium-connector/
```

---

## 2. Values Configurables por Microservicio

### 2.1 Values Globales (`values.yaml`)

```yaml
global:
  environment: production
  countryCode: ""                # Obligatorio si se despliega por país
  kafka:
    bootstrapServers: "kafka-broker-1:9092,kafka-broker-2:9092,kafka-broker-3:9092"
  schemaRegistry:
    url: "http://schema-registry:8081"
  postgres:
    host: "postgres-primary"
    port: 5432
    database: "antihurto"
    # password: inyectado via ExternalSecret / Vault Agent
  redis:
    sentinelAddr: "redis-sentinel:26379"
    masterName: "mymaster"
    # password: inyectado via ExternalSecret / Vault Agent
  opensearch:
    url: "http://opensearch-cluster:9200"
  image:
    registry: "registry.antihurto.io"
    pullPolicy: IfNotPresent
  metrics:
    enabled: true
    port: 9090
  tracing:
    enabled: true
    otlpEndpoint: "http://otel-collector:4317"
```

### 2.2 Deduplicator

```yaml
deduplicator:
  enabled: true
  replicaCount: 1          # 1 instancia por diseño de Kafka Streams
  image:
    repository: antihurto/deduplicator
    tag: "1.0.0"
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2000m"
      memory: "3Gi"
  kafkaStreams:
    appId: "deduplicator-v1"
    numStreamThreads: 4
    processingGuarantee: "exactly_once_v2"
  stateStore:
    storageClass: "fast-ssd"
    size: "10Gi"
  topics:
    inputTopic: "vehicle.events.raw"
    outputTopic: "vehicle.events.deduped"
    dlqTopic: "vehicle.events.raw.dlq"
  livenessProbe:
    initialDelaySeconds: 60
    periodSeconds: 30
  readinessProbe:
    initialDelaySeconds: 30
    periodSeconds: 10
```

### 2.3 Enrichment Service

```yaml
enrichmentService:
  enabled: true
  replicaCount: 3
  image:
    repository: antihurto/enrichment-service
    tag: "1.0.0"
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2000m"
      memory: "2Gi"
  geocoding:
    sidecarUrl: "http://localhost:8765"
    timeoutMs: 200
  plateRulesConfigPath: "/config/plate-rules.yaml"
  imageQuality:
    enabled: true
    timeoutMs: 500
  topics:
    inputTopic: "vehicle.events.deduped"
    outputTopic: "vehicle.events.enriched"
    dlqTopic: "vehicle.events.enriched.dlq"
  postgresMaxConnections: 20
  enrichmentVersion: 1
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 8
    targetCPUUtilizationPercentage: 70
```

### 2.4 Matcher Service

```yaml
matcherService:
  enabled: true
  replicaCount: 3
  image:
    repository: antihurto/matcher-service
    tag: "1.0.0"
  resources:
    requests:
      cpu: "250m"
      memory: "512Mi"
    limits:
      cpu: "1500m"
      memory: "2Gi"
  redis:
    timeoutMs: 50
    maxRetries: 0
  kafkaStreams:
    appId: "matcher-v1"
    stateDir: "/var/lib/matcher-streams"
  stateStore:
    storageClass: "fast-ssd"
    size: "5Gi"
  topics:
    inputTopic: "vehicle.events.enriched"
    outputTopic: "vehicle.alerts"
    stolenVehiclesTopic: "stolen.vehicles.events"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 6
    targetCPUUtilizationPercentage: 70
```

### 2.5 Alert Service

```yaml
alertService:
  enabled: true
  replicaCount: 3
  image:
    repository: antihurto/alert-service
    tag: "1.0.0"
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
  dedupWindowMinutes: 5
  alertExpiryHours: 24
  websocketGateway:
    grpcAddr: "websocket-gateway:9001"
    timeoutMs: 200
  topics:
    inputTopic: "vehicle.alerts"
    dlqTopic: "vehicle.alerts.dlq"
  postgresMaxConnections: 20
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 6
    targetCPUUtilizationPercentage: 70
```

### 2.6 WebSocket Gateway

```yaml
websocketGateway:
  enabled: true
  replicaCount: 3
  image:
    repository: antihurto/websocket-gateway
    tag: "1.0.0"
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
  keycloak:
    jwksRefreshIntervalSeconds: 300
  redis:
    sessionTtlSeconds: 300
  grpcServer:
    port: 9001
  websocketServer:
    port: 8080
    pingIntervalSeconds: 60
    connectionTimeoutSeconds: 120
  service:
    type: ClusterIP
    port: 8080
    annotations:
      nginx.ingress.kubernetes.io/affinity: "cookie"
      nginx.ingress.kubernetes.io/session-cookie-name: "ws-session"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 8
    targetCPUUtilizationPercentage: 60
```

### 2.7 incident-service

```yaml
incidentService:
  enabled: true
  replicaCount: 2
  image:
    repository: antihurto/incident-service
    tag: "1.0.0"
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
  topics:
    stolenVehiclesTopic: "stolen.vehicles.events"
  postgresMaxConnections: 10
  apiServer:
    port: 8080
```

### 2.8 Debezium Connector

```yaml
debeziumConnector:
  enabled: true
  # Desplegado como Kafka Connect plugin, no como Deployment independiente
  connectorName: "debezium-pg-connector"
  kafkaConnect:
    url: "http://kafka-connect:8083"
  postgres:
    hostname: "postgres-primary"
    port: 5432
    database: "antihurto"
    slotName: "debezium_slot_events"
    publicationName: "debezium_publication"
    # username/password: inyectado via ExternalSecret
  tables:
    include:
      - "public.vehicle_events"
      - "public.stolen_vehicles"
      - "public.alerts"
      - "public.incidents"
  heartbeatIntervalMs: 10000
  outputTopic: "pg.events.cdc"
  dlqTopic: "pg.events.cdc.dlq"
```

---

## 3. Instalación

### 3.1 Requisitos Previos

Antes de instalar el umbrella chart, verificar que los siguientes recursos existen:

```bash
# Verificar que Kafka está operativo
kubectl get pods -n kafka

# Verificar que PostgreSQL está operativo
kubectl get pods -n postgres

# Verificar que Redis está operativo
kubectl get pods -n redis

# Verificar que OpenSearch está operativo
kubectl get pods -n opensearch

# Crear el namespace del backbone
kubectl create namespace backbone-procesamiento

# Crear los ExternalSecrets o Secrets con credenciales (via Vault Agent / ESO)
kubectl apply -f secrets/ -n backbone-procesamiento
```

### 3.2 Comando de Instalación

```bash
# Instalación en producción
helm upgrade --install backbone-procesamiento \
  ./helm/backbone-procesamiento \
  --namespace backbone-procesamiento \
  --values helm/backbone-procesamiento/values.yaml \
  --values helm/backbone-procesamiento/values-production.yaml \
  --set global.countryCode=CO \
  --timeout 10m \
  --wait

# Verificar el estado de la instalación
helm status backbone-procesamiento -n backbone-procesamiento
kubectl get pods -n backbone-procesamiento
```

### 3.3 Instalación en Staging

```bash
helm upgrade --install backbone-procesamiento \
  ./helm/backbone-procesamiento \
  --namespace backbone-procesamiento-staging \
  --values helm/backbone-procesamiento/values.yaml \
  --values helm/backbone-procesamiento/values-staging.yaml \
  --timeout 10m \
  --wait
```

---

## 4. Actualización (Rolling Update)

```bash
# Actualizar solo el Enrichment Service a una nueva imagen
helm upgrade backbone-procesamiento \
  ./helm/backbone-procesamiento \
  --namespace backbone-procesamiento \
  --reuse-values \
  --set enrichmentService.image.tag="1.1.0" \
  --timeout 10m \
  --wait

# Verificar que el rollout fue exitoso
kubectl rollout status deployment/enrichment-service -n backbone-procesamiento
```

**Estrategia de rolling update:** todos los Deployments usan `RollingUpdate` con `maxSurge=1` y `maxUnavailable=0` para garantizar cero downtime durante las actualizaciones.

---

## 5. Rollback

```bash
# Ver el historial de revisiones
helm history backbone-procesamiento -n backbone-procesamiento

# Rollback a la revisión anterior
helm rollback backbone-procesamiento -n backbone-procesamiento

# Rollback a una revisión específica
helm rollback backbone-procesamiento 3 -n backbone-procesamiento

# Verificar que el rollback fue exitoso
helm status backbone-procesamiento -n backbone-procesamiento
kubectl get pods -n backbone-procesamiento
```

> **Nota importante para el Deduplicator:** el rollback de la imagen del Deduplicator no deshace los cambios en el state store RocksDB ni en el changelog topic de Kafka Streams. Si la nueva versión introdujo una incompatibilidad en el schema del state store, se debe limpiar el PVC y reconstruir el estado desde el changelog topic antes de hacer rollback.

---

## 6. Verificación Post-Despliegue

```bash
# Verificar métricas expuestas por cada componente
kubectl port-forward deployment/deduplicator 9090:9090 -n backbone-procesamiento
curl http://localhost:9090/metrics | grep deduplicator_

# Verificar health endpoints
kubectl port-forward deployment/enrichment-service 8080:8080 -n backbone-procesamiento
curl http://localhost:8080/health/readiness

# Verificar consumer lag de todos los consumer groups
kafka-consumer-groups \
  --bootstrap-server kafka-broker-1:9092 \
  --describe \
  --group deduplicator-cg,enrichment-cg,matcher-cg,alert-service-cg
```

---

## 7. Consideraciones por Entorno

| Parámetro | Staging | Producción |
|---|---|---|
| `replicaCount` (todos) | 1 | 2–3 mínimo |
| `autoscaling.enabled` | false | true |
| `kafkaStreams.processingGuarantee` | `at_least_once` | `exactly_once_v2` |
| `resources.limits.memory` | 50 % del valor prod | Ver values-production.yaml |
| `global.tracing.enabled` | true | true |
| `debeziumConnector.enabled` | false (sin ClickHouse en staging) | true |
