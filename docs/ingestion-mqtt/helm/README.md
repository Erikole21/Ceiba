# Helm — Despliegue de la Capa de Ingestión en Kubernetes

**Pilar:** 2 — Comunicación Segura y Eficiente  
**ADR de referencia:** ADR-004  
**Referencia:** [Visión General](../overview.md) · [emqx-cluster.md](../emqx-cluster.md) · [operations.md](../operations.md)  
**Última actualización:** 2026-05-13

---

## 1. Dependencias de Charts Helm

La capa de ingestión se despliega con los siguientes charts:

| Componente | Chart | Repositorio | Versión recomendada |
|---|---|---|---|
| **EMQX Cluster** | `emqx/emqx-operator` (CRD) + `emqx/emqx` | `https://repos.emqx.io/charts` | EMQX Operator 2.x, EMQX 5.x |
| **Upload Service** | Chart custom (`ceiba/upload-service`) | Repositorio interno | Junto con el código del servicio |
| **MinIO** (on-prem) | `minio/minio` o `minio/operator` | `https://charts.min.io` | MinIO 5.x |
| **Cert-Manager** | `cert-manager/cert-manager` | `https://charts.jetstack.io` | 1.14+ |

### 1.1 Agregar Repositorios

```bash
helm repo add emqx https://repos.emqx.io/charts
helm repo add minio https://charts.min.io
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

---

## 2. Valores Configurables por Componente

### 2.1 EMQX Cluster

| Clave `values.yaml` | Descripción | Valor dev | Valor production |
|---|---|---|---|
| `replicaCount` | Número de nodos del clúster | `1` | `3` |
| `resources.requests.cpu` | CPU solicitada por nodo | `500m` | `2000m` |
| `resources.requests.memory` | Memoria solicitada por nodo | `1Gi` | `4Gi` |
| `resources.limits.cpu` | CPU máxima | `1000m` | `4000m` |
| `resources.limits.memory` | Memoria máxima | `2Gi` | `8Gi` |
| `emqxConfig.EMQX_MQTT__MAX_PACKET_SIZE` | Tamaño máximo de paquete MQTT | `"1MB"` | `"1MB"` |
| `emqxConfig.EMQX_MQTT__SESSION_EXPIRY_INTERVAL` | Expiración de sesión | `"10m"` | `"2h"` |
| `emqxConfig.EMQX_LISTENERS__SSL__DEFAULT__SSL_OPTIONS__VERSIONS` | Versiones TLS | `"tlsv1.3"` | `"tlsv1.3"` |
| `extraVolumeMounts` | Monta certificados de CA | Ver ejemplo | Ver ejemplo |
| `secretsName.tls` | Secret K8s con cert TLS del servidor | `emqx-tls-dev` | `emqx-tls-prod` |
| `secretsName.kafka` | Secret K8s con credenciales Kafka SASL | `kafka-sasl-dev` | `kafka-sasl-prod` |
| `kafkaBridge.enabled` | Habilita el Kafka Data Bridge | `true` | `true` |
| `kafkaBridge.bootstrapHosts` | Bootstrap servers de Kafka | `"kafka:9092"` | `"kafka-0:9093,kafka-1:9093,kafka-2:9093"` |
| `kafkaBridge.topic` | Topic Kafka destino | `"vehicle.events.raw"` | `"vehicle.events.raw"` |
| `kafkaBridge.requiredAcks` | Nivel de acknowledgement Kafka | `"leader_only"` | `"all_isr"` |
| `podAntiAffinity.enabled` | Anti-afinidad por zona | `false` | `true` |
| `podAntiAffinity.topologyKey` | Clave de topología | — | `"topology.kubernetes.io/zone"` |

### 2.2 Upload Service

| Clave `values.yaml` | Descripción | Valor dev | Valor production |
|---|---|---|---|
| `replicaCount` | Número de réplicas | `1` | `3` |
| `image.tag` | Versión de la imagen del servicio | `"dev-latest"` | `"1.2.0"` |
| `resources.requests.cpu` | CPU solicitada | `"200m"` | `"500m"` |
| `resources.requests.memory` | Memoria solicitada | `"256Mi"` | `"512Mi"` |
| `storage.backend` | Adaptador de storage | `"minio"` | `"s3"` (o `"minio"` on-prem) |
| `storage.bucketName` | Nombre del bucket | `"ceiba-evidence-dev"` | `"ceiba-evidence-co"` |
| `storage.kmsKeyId` | ID/ARN de la clave KMS | `"dev-key"` | `"arn:aws:kms:..."` |
| `vault.address` | URL de Vault para OCSP | `"http://vault-dev:8200"` | `"https://vault.internal:8200"` |
| `vault.pkiPath` | Path PKI en Vault | `"pki"` | `"pki"` |
| `presignTTLSeconds` | TTL de URLs pre-firmadas | `300` | `300` |
| `secretName.vaultToken` | Secret K8s con token Vault | `"vault-token-dev"` | `"vault-token-prod"` |
| `tls.enabled` | Habilita TLS en el servicio | `false` | `true` |
| `tls.secretName` | Secret K8s con cert TLS | — | `"upload-service-tls"` |

### 2.3 MinIO (On-prem)

| Clave `values.yaml` | Descripción | Valor dev | Valor production |
|---|---|---|---|
| `mode` | Modo de despliegue | `"standalone"` | `"distributed"` |
| `replicas` | Número de instancias (modo distributed) | `1` | `4` |
| `persistence.size` | Tamaño del volumen por instancia | `"10Gi"` | `"500Gi"` |
| `resources.requests.memory` | Memoria solicitada | `"512Mi"` | `"4Gi"` |
| `kes.enabled` | Habilita KES para SSE-KMS | `false` | `true` |
| `kes.vaultAddress` | URL de Vault para KES | — | `"https://vault.internal:8200"` |
| `lifecycleRules.fullImages.expireDays` | Expiración imágenes completas | `180` | `180` |
| `lifecycleRules.thumbnails.expireDays` | Expiración thumbnails | `730` | `730` |
| `auth.rootUser` | Usuario root de MinIO | `"minioadmin"` (Secret) | (Secret K8s) |
| `auth.rootPassword` | Contraseña root | (Secret) | (Secret K8s) |

---

## 3. Ejemplos de values.yaml

### 3.1 Entorno de Desarrollo (dev)

```yaml
# values-dev.yaml
emqx:
  replicaCount: 1
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "1000m"
      memory: "2Gi"
  emqxConfig:
    EMQX_MQTT__SESSION_EXPIRY_INTERVAL: "10m"
    EMQX_MQTT__MAX_PACKET_SIZE: "1MB"
    EMQX_LISTENERS__SSL__DEFAULT__BIND: "0.0.0.0:8883"
  kafkaBridge:
    enabled: true
    bootstrapHosts: "kafka.kafka.svc:9092"
    topic: "vehicle.events.raw"
    requiredAcks: "leader_only"
    saslUsername: ""   # se inyecta desde Secret
    saslPassword: ""   # se inyecta desde Secret
  podAntiAffinity:
    enabled: false

uploadService:
  replicaCount: 1
  image:
    tag: "dev-latest"
  storage:
    backend: "minio"
    bucketName: "ceiba-evidence-dev"
    endpoint: "http://minio.minio.svc:9000"
    kmsKeyId: "dev-key"
  vault:
    address: "http://vault.vault.svc:8200"
    pkiPath: "pki"
  presignTTLSeconds: 300
  tls:
    enabled: false

minio:
  mode: "standalone"
  replicas: 1
  persistence:
    size: "10Gi"
  kes:
    enabled: false
```

### 3.2 Entorno de Producción (production)

```yaml
# values-prod.yaml
emqx:
  replicaCount: 3
  resources:
    requests:
      cpu: "2000m"
      memory: "4Gi"
    limits:
      cpu: "4000m"
      memory: "8Gi"
  emqxConfig:
    EMQX_MQTT__SESSION_EXPIRY_INTERVAL: "2h"
    EMQX_MQTT__MAX_PACKET_SIZE: "1MB"
    EMQX_LISTENERS__SSL__DEFAULT__BIND: "0.0.0.0:8883"
    EMQX_LISTENERS__SSL__DEFAULT__LIMITER__CONNECTION__RATE: "5000/s"
  kafkaBridge:
    enabled: true
    bootstrapHosts: "kafka-0.kafka-headless.kafka.svc:9093,kafka-1.kafka-headless.kafka.svc:9093,kafka-2.kafka-headless.kafka.svc:9093"
    topic: "vehicle.events.raw"
    requiredAcks: "all_isr"
    tls:
      enabled: true
      cacertSecretName: "kafka-ca-cert"
    saslMechanism: "scram_sha_512"
    saslSecretName: "kafka-sasl-credentials"
  podAntiAffinity:
    enabled: true
    topologyKey: "topology.kubernetes.io/zone"
  pdb:
    minAvailable: 2
  secretsName:
    tls: "emqx-tls-prod"
    vaultCA: "vault-ca-cert"

uploadService:
  replicaCount: 3
  image:
    tag: "1.2.0"
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
  storage:
    backend: "s3"
    bucketName: "ceiba-evidence-co"
    region: "us-east-1"
    kmsKeyId: "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
  vault:
    address: "https://vault.internal:8200"
    pkiPath: "pki"
    caCertSecretName: "vault-ca-cert"
  presignTTLSeconds: 300
  tls:
    enabled: true
    secretName: "upload-service-tls"
  secretName:
    vaultToken: "vault-token-prod"

minio:
  # Solo para despliegues on-prem; en cloud se usa el adaptador S3/GCS/Azure
  enabled: false
```

---

## 4. Instalación

### 4.1 Instalación Inicial (Entorno dev)

```bash
# 1. Crear el namespace
kubectl create namespace mqtt

# 2. Crear los Secrets necesarios (nunca en el values.yaml)
kubectl create secret generic kafka-sasl-credentials -n mqtt \
  --from-literal=username=emqx-bridge \
  --from-literal=password='<password>'

kubectl create secret generic emqx-tls-dev -n mqtt \
  --from-file=tls.crt=/path/to/emqx-server.crt \
  --from-file=tls.key=/path/to/emqx-server.key \
  --from-file=ca.crt=/path/to/ca-chain.crt

kubectl create secret generic vault-token-dev -n mqtt \
  --from-literal=token='<vault-token>'

# 3. Instalar el EMQX Operator (si no está instalado en el cluster)
helm install emqx-operator emqx/emqx-operator \
  --namespace emqx-operator-system \
  --create-namespace

# 4. Instalar la capa de ingestión
helm install ceiba-ingestion ./charts/ingestion-mqtt \
  --namespace mqtt \
  --values values-dev.yaml
```

### 4.2 Instalación en Producción

```bash
helm install ceiba-ingestion ./charts/ingestion-mqtt \
  --namespace mqtt \
  --values values-prod.yaml \
  --set emqx.secretsName.tls=emqx-tls-prod \
  --set uploadService.secretName.vaultToken=vault-token-prod
```

---

## 5. Actualización (Rolling Update)

```bash
# Actualizar a una nueva versión del chart o de los valores
helm upgrade ceiba-ingestion ./charts/ingestion-mqtt \
  --namespace mqtt \
  --values values-prod.yaml \
  --set uploadService.image.tag="1.3.0"

# Verificar el estado del rollout
kubectl rollout status deployment/upload-service -n mqtt
kubectl rollout status statefulset/emqx -n mqtt
```

**Nota para EMQX:** la actualización del StatefulSet de EMQX se realiza nodo por nodo (`maxUnavailable: 1`). Cada nodo se drena de conexiones antes de reiniciarse; los dispositivos se reconectan a los otros nodos. El proceso puede tardar varios minutos según el número de sesiones activas.

---

## 6. Rollback

```bash
# Ver historial de releases
helm history ceiba-ingestion -n mqtt

# Rollback a la revisión anterior
helm rollback ceiba-ingestion -n mqtt

# Rollback a una revisión específica
helm rollback ceiba-ingestion 3 -n mqtt

# Verificar el estado tras el rollback
helm status ceiba-ingestion -n mqtt
kubectl get pods -n mqtt
```

---

## 7. Verificación Post-Despliegue

```bash
# Verificar pods en ejecución
kubectl get pods -n mqtt

# Verificar que EMQX está en clúster
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl cluster status

# Verificar que el Kafka Data Bridge está activo
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl bridges status

# Health check del Upload Service
curl https://upload.ceiba-antihurto.io/v1/health

# Verificar métricas Prometheus
curl http://emqx-0.emqx-headless.mqtt.svc:18083/api/v5/prometheus/stats
```

---

## 8. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [emqx-cluster.md](../emqx-cluster.md) | Parámetros de configuración referenciados en los values |
| [upload-service.md](../upload-service.md) | Variables de configuración del Upload Service |
| [operations.md](../operations.md) | Procedimientos operacionales (escalado, rollback de emergencia) |
| [terraform/README.md](../terraform/README.md) | Provisión de infraestructura previa al despliegue Helm |
