# Operaciones de la Capa de Ingestión — Runbooks

**Pilar:** 2 — Comunicación Segura y Eficiente  
**ADRs de referencia:** ADR-002 · ADR-010  
**Referencia:** [Visión General](./overview.md) · [emqx-cluster.md](./emqx-cluster.md) · [bridge.md](./bridge.md) · [security.md](./security.md)  
**Última actualización:** 2026-05-13

---

## 1. Escalado Horizontal de EMQX

### 1.1 Métricas de Alerta para Escalado

Las siguientes métricas de Prometheus/Grafana disparan alertas de capacidad del clúster EMQX:

| Métrica | Alerta WARNING | Alerta CRITICAL | Acción |
|---|---|---|---|
| `emqx_connections_count` por nodo | > 70 % de `max_connections` | > 90 % de `max_connections` | Agregar nodo al clúster |
| `erlang_memory_bytes{type="total"}` por nodo | > 75 % de RAM disponible | > 90 % de RAM disponible | Agregar nodo o reducir `session_expiry_interval` |
| `emqx_messages_publish_rate` (mensajes/seg) | > 50.000 msg/s por nodo | > 80.000 msg/s por nodo | Agregar nodo al clúster |
| `emqx_sessions_count` por nodo | > 80 % del presupuesto calculado | > 95 % del presupuesto calculado | Agregar nodo o reducir expiración de sesiones |
| `emqx_cluster_nodes_running` | < 3 | < 2 | Alertar y recuperar nodo caído (ver §5) |

### 1.2 Procedimiento de Escalado (Adición de Nodo)

**Pre-condiciones:** el clúster tiene al menos 1 nodo en WARNING de capacidad durante ≥ 15 minutos.

```bash
# 1. Verificar estado actual del clúster
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl cluster status

# 2. Actualizar el StatefulSet con el número deseado de réplicas
kubectl scale statefulset emqx --replicas=4 -n mqtt

# 3. Verificar que el nuevo nodo se une al clúster (puede tardar 1-2 min)
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl cluster status

# 4. Verificar distribución de sesiones (las sesiones existentes NO se migran automáticamente)
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl broker stats | grep sessions

# 5. Actualizar el PodDisruptionBudget si es necesario
kubectl patch pdb emqx-pdb -n mqtt --type=json \
  -p='[{"op": "replace", "path": "/spec/minAvailable", "value": 3}]'
```

**Nota:** EMQX distribuye las *nuevas* conexiones al nodo adicional. Las sesiones existentes permanecen en sus nodos actuales hasta que los dispositivos se reconecten.

### 1.3 Anti-afinidad en Escalado

Al agregar nodos, verificar que el nuevo pod se despliegue en una AZ diferente a los existentes. Si las 3 AZs ya tienen un nodo, el cuarto nodo puede estar en cualquier AZ (la anti-afinidad fuerte se relaja a preferida para el cuarto nodo en adelante).

---

## 2. Gestión de Tormentas de Reconexiones (CR-07)

Una tormenta de reconexiones ocurre cuando miles de dispositivos recuperan conectividad GSM simultáneamente después de un corte masivo.

### 2.1 Detección

| Señal | Umbral de alerta |
|---|---|
| `rate(emqx_connections_count[1m])` > | 5.000 conexiones nuevas por minuto |
| `emqx_messages_publish_rate` | Pico > 3x la tasa media de los últimos 30 min |
| Latencia de PUBACK (Kafka bridge) | P95 > 1 s |

### 2.2 Mecanismos de Protección Existentes

**En el broker EMQX:**

```hocon
# Límite de conexiones nuevas por segundo
listeners.ssl.default.limiter.connection.rate = "5000/s"
```

Esto limita la tasa de nuevas conexiones TCP a 5.000 por segundo a nivel del listener. Los clientes que superan este límite son pausados (backpressure TCP).

**En el agente de borde:**

El agente implementa backoff exponencial de reconexión con jitter (ver [`docs/agente-borde/uploader.md §5.2`](../agente-borde/uploader.md)):

```
base_delay = 2s, max_delay = 120s, jitter_factor = 0.25
```

El jitter distribuye temporalmente los reintentos, reduciendo la amplitud del pico de reconexiones.

### 2.3 Procedimiento de Respuesta

```bash
# Paso 1: Verificar que el circuit breaker del listener está activo
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl listeners

# Paso 2: Si la tormenta es severa, reducir temporalmente la tasa de conexiones
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl conf update \
  "listeners.ssl.default.limiter.connection.rate" "2000/s"

# Paso 3: Monitorear la reducción del pico en Grafana
# Dashboard: EMQX → Connection Rate

# Paso 4: Una vez normalizado (< 30 min), restaurar la tasa
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl conf update \
  "listeners.ssl.default.limiter.connection.rate" "5000/s"

# Paso 5: Verificar que no haya sesiones bloqueadas o mensajes perdidos
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl broker stats
```

**Escalado reactivo:** si la tormenta supera la capacidad del rate limiter, escalar el clúster (ver §1.2) para distribuir la carga de conexiones.

---

## 3. Rotación de CA de Vault (Procedimiento Dual-CA)

La rotación de la CA de dispositivos es una operación de alto riesgo que requiere el procedimiento de doble CA para evitar interrupciones.

### 3.1 Pre-condiciones

- La CA actual tiene una antigüedad > 60 días (se rota antes del vencimiento de la CA).
- Se ha notificado al equipo de operaciones con al menos 48 horas de anticipación.
- Los runbooks de reversión están disponibles.

### 3.2 Procedimiento

**Fase 1 — Crear nueva CA en Vault (sin activar):**

```bash
# Crear nueva PKI mount para la CA nueva
vault secrets enable -path=pki_v2 pki
vault write pki_v2/root/generate/internal \
  common_name="Ceiba Anti-Hurto Device CA v2" \
  ttl="87600h"   # 10 años

# Exportar el certificado de la CA nueva
vault read -field=certificate pki_v2/root/ca/pem > /tmp/ca-v2.crt
```

**Fase 2 — Distribuir ambas CAs en EMQX:**

```bash
# Combinar CA actual (v1) y CA nueva (v2) en el bundle de confianza
cat /etc/emqx/certs/ca-chain.crt /tmp/ca-v2.crt > /tmp/ca-bundle.crt

# Actualizar el configmap con el bundle
kubectl create configmap emqx-ca-bundle -n mqtt \
  --from-file=ca-chain.crt=/tmp/ca-bundle.crt \
  --dry-run=client -o yaml | kubectl apply -f -

# Recargar TLS en EMQX sin reiniciar
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl tls update
```

En este punto, EMQX acepta certificados firmados por **ambas CAs** (v1 y v2). Los dispositivos existentes con certificados v1 siguen funcionando.

**Fase 3 — Emitir nuevos certificados desde CA v2:**

```bash
# Activar la CA v2 para emisión de nuevos certificados
vault write pki_v2/roles/device-cert \
  allowed_domains="ceiba-antihurto.io" \
  allow_subdomains=true \
  max_ttl="2160h"   # 90 días
```

Los dispositivos que roten su certificado en el período de transición recibirán certificados firmados por CA v2.

**Fase 4 — Revocar CA v1 tras completar la transición:**

Una vez que todos los dispositivos hayan rotado sus certificados (confirmado por monitoreo: ningún certificado v1 activo), retirar la CA v1 del bundle de confianza de EMQX:

```bash
# Actualizar el bundle para incluir solo CA v2
kubectl create configmap emqx-ca-bundle -n mqtt \
  --from-file=ca-chain.crt=/tmp/ca-v2.crt \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl exec -it emqx-0 -n mqtt -- emqx_ctl tls update
```

### 3.3 Monitoreo Durante la Rotación

```
# Métrica: dispositivos con certificados v1 activos
# Si desciende a 0, la transición está completa
emqx_connections_by_ca_version{version="v1"}
```

### 3.4 Reversión de Emergencia

Si la rotación causa problemas, restaurar el bundle CA anterior (v1 únicamente) y recargar TLS.

---

## 4. Monitoreo de Lag del Bridge Kafka (CA-15)

### 4.1 Métricas Clave

Las métricas del bridge se exponen en el endpoint Prometheus de EMQX. Ver [bridge.md §7](./bridge.md) para la lista completa.

**Alertas configuradas:**

| Alerta | Condición PromQL | Severidad | Acción |
|---|---|---|---|
| `BridgeKafkaLagHigh` | `emqx_bridges_kafka_queue_size > 10000` durante 5 min | WARNING | Verificar salud de Kafka; investigar causa |
| `BridgeKafkaLagCritical` | `emqx_bridges_kafka_queue_size > 50000` | CRITICAL | Iniciar procedimiento §4.2 |
| `BridgeKafkaDropped` | `increase(emqx_bridges_kafka_dropped[5m]) > 0` | CRITICAL | Iniciar procedimiento §4.2 |
| `BridgeLagSeconds` | El lag en segundos es `queue_size / throughput_rate` | WARNING si > 30s | Ver §4.2 |

**Cálculo del lag en segundos:**

```
lag_seconds = emqx_bridges_kafka_queue_size / emqx_bridges_kafka_sent_rate
```

**Umbral de alerta:** lag > 30 segundos indica que el bridge está procesando mensajes significativamente más lento de lo que se reciben.

### 4.2 Procedimiento de Respuesta a Lag Elevado del Bridge

```bash
# Paso 1: Verificar salud del clúster Kafka
kubectl exec -it kafka-0 -n kafka -- kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9092

# Paso 2: Verificar conectividad EMQX → Kafka
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl bridges status

# Paso 3: Verificar métricas del bridge
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl bridges metrics vehicle_events_bridge

# Paso 4: Si Kafka está sano pero el bridge tiene errores, reiniciar el bridge
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl bridges restart vehicle_events_bridge

# Paso 5: Verificar que el lag disminuye en los siguientes 5 minutos
# Si no disminuye, verificar topic Kafka y permisos SASL
kubectl exec -it kafka-0 -n kafka -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe --group emqx-bridge
```

---

## 5. Procedimientos de Recuperación

### 5.1 Recuperación ante Fallo de Nodo EMQX (CR-04)

**Síntoma:** un pod EMQX se cae o es eliminado; `emqx_cluster_nodes_running < 3`.

```bash
# Paso 1: Kubernetes reinicia automáticamente el pod (restart policy)
# Verificar estado:
kubectl get pods -n mqtt -w

# Paso 2: Verificar que el nodo recuperado se une al clúster
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl cluster status

# Paso 3: Los dispositivos conectados al nodo caído se reconectan
# automáticamente a los otros nodos (keep-alive 60s → timeout 90s).
# Verificar tasa de reconexiones:
rate(emqx_connections_count[5m])

# Paso 4: Si el nodo no se une al clúster automáticamente:
kubectl exec -it emqx-new -n mqtt -- emqx_ctl cluster join emqx@emqx-0.emqx-headless.mqtt.svc
```

**Impacto:** los dispositivos en el nodo caído se reconectan en máximo 90 segundos (1.5 × keep-alive). Las sesiones persistentes QoS 1 garantizan que los mensajes no se pierdan.

### 5.2 Recuperación ante Fallo del Bridge MQTT→Kafka (CR-06)

**Síntoma:** `emqx_bridges_kafka_failed` aumenta; el bridge está en estado `disconnected`.

```bash
# Paso 1: Verificar el estado del bridge
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl bridges status

# Paso 2: Verificar conectividad con Kafka
kubectl exec -it emqx-0 -n mqtt -- curl -s \
  http://kafka-0.kafka-headless.kafka.svc:9093 | head -5

# Paso 3: Reiniciar el bridge
kubectl exec -it emqx-0 -n mqtt -- emqx_ctl bridges restart vehicle_events_bridge

# Paso 4: Verificar que el buffer acumulado se drena
watch kubectl exec -it emqx-0 -n mqtt -- \
  emqx_ctl bridges metrics vehicle_events_bridge
```

**Comportamiento durante el fallo:** los mensajes QoS 1 permanecen en las sesiones EMQX de los dispositivos. EMQX no envía PUBACK a los dispositivos hasta que Kafka confirme la escritura. Los dispositivos mantienen los eventos en estado `in_flight` en su SQLite local.

### 5.3 Recuperación ante Fallo del Object Storage (CR-08)

**Síntoma:** el Upload Service retorna `503 Service Unavailable`; los dispositivos no pueden subir imágenes.

```bash
# Paso 1: Verificar salud del objeto storage
kubectl exec -it upload-service-xxx -n mqtt -- curl -s \
  http://minio-service.minio.svc:9000/minio/health/live

# Paso 2: Verificar health check del Upload Service
curl https://upload.ceiba-antihurto.io/v1/health

# Paso 3: Si MinIO está caído, verificar pods MinIO
kubectl get pods -n minio

# Paso 4: Si es un problema de red, verificar NetworkPolicy
kubectl describe networkpolicy -n mqtt

# Paso 5: Los dispositivos reciben 503 con Retry-After:30
# El agente de borde almacena las imágenes localmente hasta que
# el storage se recupere (Image Store local).
```

**Impacto:** los eventos MQTT con `image_unavailable: true` siguen publicándose en Kafka. Las imágenes se subirán cuando el storage se recupere. No se pierde telemetría de eventos.

---

## 6. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [emqx-cluster.md](./emqx-cluster.md) | Parámetros de configuración referenciados en los runbooks |
| [bridge.md §7](./bridge.md) | Lista completa de métricas del bridge |
| [security.md §2](./security.md) | Procedimiento de OCSP y CRL en contexto de seguridad |
| [helm/README.md](./helm/README.md) | Comandos Helm para actualización y rollback |
| [`docs/identidad-seguridad/vault-ha-unseal.md`](../identidad-seguridad/vault-ha-unseal.md) | Procedimiento de recuperación de Vault (referencia para rotación CA) |
