# Helm — Pilar 4: Sincronización de Países

**Change:** `sincronizacion-paises`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Estructura de Charts

El Pilar 4 se organiza en tres charts Helm independientes:

```
helm/
├── canonical-vehicles-service/   # Canonical Vehicles Service
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── edge-distribution-service/    # Edge Distribution Service
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
└── country-adapter/               # Chart genérico para adaptadores de país
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
```

Existe también un chart umbrella `sincronizacion-paises` que agrega los tres:

```
helm/sincronizacion-paises/
├── Chart.yaml
└── values.yaml        # Overrides globales
```

---

## 2. Values del Canonical Vehicles Service

```yaml
# helm/canonical-vehicles-service/values.yaml
replicaCount: 2

image:
  repository: ghcr.io/antihurto/canonical-vehicles-service
  tag: "1.0.0"
  pullPolicy: IfNotPresent

config:
  kafkaBootstrapServers: "kafka:9092"
  schemaRegistryUrl: "http://schema-registry:8081"
  consumerGroupId: "canonical-vehicles-service"
  inputTopic: "stolen.vehicles.events"
  outputTopic: "stolen.vehicles.canonical"
  dlqTopic: "stolen.vehicles.events.dlq"
  redisHotListTtlSeconds: 2592000       # 30 días
  redisMaxRetries: 5
  dedupWindowSeconds: 86400             # 24 h
  bloomFilterNotifyEnabled: true
  # Lista separada por comas de country_codes registrados:
  validCountryCodes: "CO,VE,MX,AR"

secrets:
  # Referencia al secret de Kubernetes (gestionado por External Secrets Operator + Vault)
  databaseUrlSecretName: canonical-vehicles-db-url
  redisUrlSecretName: canonical-vehicles-redis-url

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8
  metrics:
    - type: External
      external:
        metric:
          name: kafka_consumer_group_lag
          selector:
            matchLabels:
              group: canonical-vehicles-service
              topic: stolen.vehicles.events
        target:
          type: Value
          value: "5000"

podDisruptionBudget:
  minAvailable: 1

service:
  port: 8080
  metricsPort: 9090

livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 5
```

---

## 3. Values del Edge Distribution Service

```yaml
# helm/edge-distribution-service/values.yaml
replicaCount: 2

image:
  repository: ghcr.io/antihurto/edge-distribution-service
  tag: "1.0.0"
  pullPolicy: IfNotPresent

config:
  kafkaBootstrapServers: "kafka:9092"
  consumerGroupId: "edge-distribution-service"
  inputTopic: "stolen.vehicles.canonical"
  minioBucket: "bloom-filters"
  fullSnapshotThresholdDays: 7
  bfRetentionVersions: 48
  # Países activos (uno por línea o separados por coma):
  activeCountries: "CO,VE,MX,AR"

secrets:
  databaseUrlSecretName: edge-distribution-db-url
  minioSecretName: edge-distribution-minio-credentials
  mqttSecretName: edge-distribution-mqtt-credentials

resources:
  requests:
    cpu: "500m"
    memory: "1Gi"    # El servicio mantiene BFs en memoria durante la generación
  limits:
    cpu: "2000m"
    memory: "4Gi"

podDisruptionBudget:
  minAvailable: 1

service:
  metricsPort: 9090
```

---

## 4. Values del Adaptador de País (Chart Genérico)

El chart `country-adapter` es un chart genérico parametrizable. Cada instancia se despliega con un `values-{cc}.yaml` específico.

```yaml
# helm/country-adapter/values.yaml (valores base)
replicaCount: 1   # Generalmente 1 instancia por adaptador (no escala horizontal)

image:
  repository: ghcr.io/antihurto/adapter-${COUNTRY_CODE_LOWER}
  tag: "1.0.0"
  pullPolicy: IfNotPresent

config:
  countryCode: ""           # Obligatorio: sobreescribir con el CC del país
  sourceSystem: ""          # Obligatorio: identificador del sistema fuente
  mode: "JDBC"              # CDC | JDBC | SOAP_REST | FILE_DROP
  checkpointStoreType: "REDIS"   # REDIS | POSTGRES | FILE
  pollingIntervalSeconds: 300    # Para modo JDBC y SOAP_REST (5 min por defecto)
  extensions_allowed: []         # Lista de campos adicionales autorizados

secrets:
  sourceCredentialsSecretName: ""   # Obligatorio: nombre del secret K8s para BD policial
  kafkaCredentialsSecretName: "kafka-producer-credentials"

# Para adaptadores on-prem: deshabilitar affinity y usar NodeSelector
# para fijar el pod en los nodos del territorio nacional
nodeSelector: {}
tolerations: []
affinity: {}

# Para adaptadores on-prem sin RBAC de K8s avanzado:
serviceAccount:
  create: false

resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"
```

```yaml
# helm/country-adapter/values-co.yaml (Colombia — ejemplo)
config:
  countryCode: "CO"
  sourceSystem: "SIJIN_CO"
  mode: "JDBC"
  pollingIntervalSeconds: 300
  extensions_allowed:
    - numero_motor
    - numero_chasis

secrets:
  sourceCredentialsSecretName: "adapter-co-db-credentials"

resources:
  requests:
    cpu: "300m"
    memory: "384Mi"
```

---

## 5. Instalación

### 5.1 Prerrequisitos

```bash
# Verificar que los namespaces existen
kubectl get namespace antihurto-core antihurto-adapters

# Verificar que los secrets gestionados por External Secrets están presentes
kubectl get externalsecret -n antihurto-core
```

### 5.2 Instalación del chart umbrella (todos los componentes de núcleo)

```bash
# Añadir el repositorio (si aplica)
helm repo add antihurto https://helm.antihurto.internal
helm repo update

# Instalar el chart umbrella
helm install sincronizacion-paises antihurto/sincronizacion-paises \
  --namespace antihurto-core \
  --values values-production.yaml \
  --wait \
  --timeout 10m
```

### 5.3 Instalación de un adaptador de país

```bash
# Instalar el adaptador de Colombia
helm install adapter-co antihurto/country-adapter \
  --namespace antihurto-adapters \
  --values helm/country-adapter/values-co.yaml \
  --wait \
  --timeout 5m
```

---

## 6. Actualización

### 6.1 Actualización del Canonical Vehicles Service

```bash
# Actualizar la imagen a una nueva versión
helm upgrade canonical-vehicles-service antihurto/canonical-vehicles-service \
  --namespace antihurto-core \
  --set image.tag="1.1.0" \
  --reuse-values \
  --wait
```

### 6.2 Actualización de configuración de country_codes

Al agregar un nuevo país, se agrega el `country_code` a `validCountryCodes` y se hace un `helm upgrade` del Canonical Vehicles Service y del Edge Distribution Service.

```bash
helm upgrade canonical-vehicles-service antihurto/canonical-vehicles-service \
  --namespace antihurto-core \
  --set config.validCountryCodes="CO,VE,MX,AR,PE" \
  --reuse-values \
  --wait
```

---

## 7. Rollback

```bash
# Ver el historial de releases
helm history canonical-vehicles-service -n antihurto-core

# Rollback a la revisión anterior
helm rollback canonical-vehicles-service -n antihurto-core

# Rollback a una revisión específica
helm rollback canonical-vehicles-service 3 -n antihurto-core --wait
```

---

## 8. Notas sobre Despliegue On-Prem del Adaptador

Cuando un adaptador debe desplegarse on-prem dentro del territorio nacional (Nivel 1 de soberanía), el proceso difiere:

### 8.1 Restricciones on-prem

- El clúster Kubernetes on-prem puede ser RKE2 + Rancher o un K3s de un solo nodo si los recursos son limitados.
- El adaptador debe poder alcanzar el Kafka del cloud a través del túnel VPN.
- No se requiere acceso a Redis ni a PostgreSQL del sistema central desde el adaptador on-prem.

### 8.2 Instalación on-prem

```bash
# En el servidor on-prem del país (con kubeconfig configurado)
# El KUBECONFIG apunta al cluster K3s/RKE2 del país

# Instalar solo el adaptador (no el núcleo)
helm install adapter-co antihurto/country-adapter \
  --namespace antihurto-adapters \
  --values values-co-onprem.yaml \
  --set config.checkpointStoreType=FILE \
  --set config.checkpointFilePath=/data/checkpoints \
  --wait
```

```yaml
# values-co-onprem.yaml — overrides para despliegue on-prem
# El adaptador se conecta a Kafka en cloud a través de la VPN
config:
  kafkaBootstrapServers: "kafka.cloud.antihurto.internal:9092"  # Endpoint VPN
  schemaRegistryUrl: "http://schema-registry.cloud.antihurto.internal:8081"
  checkpointStoreType: FILE

# Usar un PersistentVolume local para el checkpoint
persistence:
  enabled: true
  storageClass: local-storage
  size: 1Gi
  mountPath: /data/checkpoints
```

### 8.3 Gestión de secretos on-prem

En entornos on-prem sin acceso a Vault del sistema central, los secretos se gestionan mediante:
- Vault Agent en modo on-prem (instancia local de Vault).
- O bien: Kubernetes Secrets creados manualmente con rotación periódica manual.

La opción de Vault local se recomienda para mayor seguridad.

---

## 9. Referencias

- [`canonical-vehicles-service.md`](../canonical-vehicles-service.md)
- [`edge-distribution-service.md`](../edge-distribution-service.md)
- [`country-adapter-framework.md`](../country-adapter-framework.md)
- [`country-onboarding-guide.md`](../country-onboarding-guide.md)
- [`terraform/README.md`](../terraform/README.md)
