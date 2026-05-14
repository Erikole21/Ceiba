# Edge Distribution Service

**Change:** `sincronizacion-paises`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Responsabilidad

El Edge Distribution Service mantiene los Bloom filters por país actualizados en los agentes de borde. Recibe notificaciones de cambio del Canonical Vehicles Service, regenera el BF del país afectado, calcula el delta respecto a la versión anterior y lo publica vía MQTT retained.

**Criterios de aceptación que implementa:** CA-09, CA-10, CA-11, CA-12; CR-05.

---

## 2. Puerto Hexagonal — `BloomFilterStorePort`

```mermaid
graph LR
    subgraph Dominio["Dominio — Edge Distribution Service"]
        UC_DELTA[GenerateBFDelta\nuse case]
        UC_FULL[GenerateFullSnapshot\nuse case]
        UC_QUERY[QueryBFVersion\nuse case]
    end

    subgraph Entrada["Puertos de entrada (primarios)"]
        CVS_NOTIFY[BloomFilterNotifyPort\nrecibir notificación de cambio]
        K_CONSUME[CanonicalEventsConsumerPort\nconsumir stolen.vehicles.canonical]
    end

    subgraph Salida["Puertos de salida (secundarios)"]
        BF_STORE[BloomFilterStorePort\nMinIO / S3 — versiones BF]
        PG_READ[VehiclesQueryPort\nPostgreSQL — leer placas STOLEN]
        MQTT_PUB[MQTTPublisherPort\npublicar en countries/{cc}/bloom-filter]
        METRICS[MetricsPort\nPrometheus]
    end

    CVS_NOTIFY --> UC_DELTA
    K_CONSUME --> UC_DELTA
    UC_DELTA --> BF_STORE
    UC_DELTA --> PG_READ
    UC_DELTA --> MQTT_PUB
    UC_DELTA --> UC_FULL
    UC_FULL --> BF_STORE
    UC_FULL --> PG_READ
    UC_FULL --> MQTT_PUB
    UC_QUERY --> BF_STORE
```

### 2.1 Definición del `BloomFilterStorePort`

```
interface BloomFilterStorePort {
    // Almacenar una versión del BF (serializada como bytes)
    store(country_code: string, version: string, bf_bytes: bytes): void

    // Obtener la versión más reciente
    getLatest(country_code: string): BFVersion | null

    // Obtener una versión específica
    get(country_code: string, version: string): BFVersion | null

    // Listar las últimas N versiones (para calcular deltas)
    listVersions(country_code: string, limit: int): []BFVersion

    // Eliminar versiones antiguas (retención)
    deleteOlderThan(country_code: string, cutoff_date: datetime): int
}

struct BFVersion {
    country_code: string
    version:      string    // ISO 8601 timestamp del momento de generación
    bf_bytes:     bytes     // BF serializado (bit array)
    checksum:     string    // SHA-256 del bf_bytes
    element_count: int      // Número de elementos en el BF
    created_at:   datetime
}
```

**Implementación de referencia:** MinIO / S3-compatible. Las versiones se almacenan como objetos en el bucket `bloom-filters`:

```
bloom-filters/
  CO/
    2026-01-15T08:00:00Z.bf.lz4
    2026-01-15T14:30:00Z.bf.lz4
    latest -> 2026-01-15T14:30:00Z.bf.lz4
  VE/
    ...
```

---

## 3. Algoritmo de Generación del BF

### 3.1 Construcción del BF para un país

```
procedure buildBF(country_code: string) -> BloomFilter:
    plates = VehiclesQueryPort.queryAllStolenPlates(country_code)
    bf = BloomFilter(
        size_bits = BF_SIZE_BITS,    // 8_388_608
        hash_funcs = BF_HASH_FUNCS   // 6
    )
    for plate in plates:
        bf.add("{country_code}:{plate}")
    return bf
```

Los elementos del BF son strings con la forma `{country_code}:{plate}` (e.g., `CO:ABC123`). Esto previene colisiones entre países en caso de que el agente de borde use un BF global en el futuro.

### 3.2 Cálculo del delta (XOR)

El delta se calcula como la operación XOR bit a bit entre los dos arrays de bits:

```
procedure calculateDelta(bf_prev: bytes, bf_new: bytes) -> bytes:
    assert len(bf_prev) == len(bf_new)  // Mismo tamaño fijo
    delta = []
    for i in range(len(bf_prev)):
        delta[i] = bf_prev[i] XOR bf_new[i]
    return delta
```

El agente de borde aplica el mismo XOR para obtener `bf_new` desde `bf_prev`:

```
bf_new = bf_prev XOR delta
```

Esta operación es reversible: si se aplica dos veces se recupera el BF original. Por eso es seguro distribuir el delta — un agente que ya actualizó puede detectar el problema comparando el checksum.

### 3.3 Compresión

El delta y el full snapshot se comprimen con **LZ4** antes de publicar en MQTT:
- Un delta típico (pocas placas nuevas o eliminadas) puede comprimir de 1 MB a < 10 KB.
- Un full snapshot comprime de 1 MB a ~200–400 KB.

---

## 4. Flujo de Generación y Distribución

```mermaid
sequenceDiagram
    participant CVS as Canonical Vehicles Service
    participant EDS as Edge Distribution Service
    participant PG as PostgreSQL
    participant MINIO as MinIO (BloomFilterStorePort)
    participant MQTT as MQTT Broker
    participant AG as Agentes de Borde CO

    CVS->>EDS: notifyChange(country_code="CO")

    EDS->>MINIO: getLatest("CO")
    MINIO-->>EDS: BFVersion{version_prev, bf_bytes_prev}

    EDS->>PG: SELECT plate FROM canonical_vehicles\nWHERE country_code='CO' AND status='STOLEN'
    PG-->>EDS: stream de plates

    EDS->>EDS: buildBF("CO") — construir BF nuevo
    EDS->>EDS: calculateDelta(bf_prev, bf_new) — XOR bit a bit
    EDS->>EDS: compress(delta, LZ4)
    EDS->>EDS: sha256(bf_new_bytes) — checksum

    EDS->>MINIO: store("CO", version_new, bf_new_bytes)

    EDS->>MQTT: PUBLISH countries/CO/bloom-filter\n{type:DELTA, version_new, version_prev,\ndelta_bytes, checksum, bf_size_bits, bf_hash_funcs}
    Note over MQTT,AG: retained=true, QoS=1

    AG->>MQTT: SUBSCRIBE countries/CO/bloom-filter
    MQTT-->>AG: mensaje retained (inmediatamente al conectar)
    AG->>AG: verificar checksum
    AG->>AG: aplicar XOR sobre BF local
    AG->>AG: version_local = version_new
```

---

## 5. Política de Full Snapshot (CA-10, CR-05)

### 5.1 Criterios para emitir full snapshot

El servicio emite un full snapshot en lugar de un delta cuando:

1. **No hay versión previa:** primer BF generado para el país.
2. **Versión demasiado antigua:** la diferencia entre `version_prev` y `now` supera el umbral configurable (valor de referencia: 7 días = 604800 segundos).
3. **Cadena de deltas rota:** no existe la versión `version_prev` en el almacén (fue eliminada por la política de retención).
4. **Solicitud explícita:** el operador triggerea un full snapshot manualmente.

```
procedure shouldUseFullSnapshot(country_code: string, version_local: string) -> bool:
    latest = BloomFilterStorePort.getLatest(country_code)
    if version_local is null or latest is null:
        return true
    age_seconds = (now - parse(version_local)).total_seconds()
    if age_seconds > config.full_snapshot_threshold_seconds:  // 7 días
        return true
    prev_version = BloomFilterStorePort.get(country_code, version_local)
    if prev_version is null:
        return true
    return false
```

### 5.2 Payload de full snapshot

```json
{
  "type": "FULL_SNAPSHOT",
  "country_code": "CO",
  "version_new": "2026-01-15T14:30:00Z",
  "version_prev": null,
  "delta_bytes": "<base64 del BF completo comprimido LZ4>",
  "checksum_sha256": "abc123...",
  "bf_size_bits": 8388608,
  "bf_hash_funcs": 6,
  "element_count": 487532,
  "timestamp": "2026-01-15T14:31:00Z"
}
```

El campo `delta_bytes` contiene el BF completo (no un delta) cuando `type = FULL_SNAPSHOT`. El agente de borde debe reemplazar su BF local completamente al recibir este tipo de mensaje.

---

## 6. Tópico MQTT de Distribución

```
Topic:    countries/{country_code}/bloom-filter
QoS:      1 (at least once)
Retained: true
```

El mensaje retained garantiza que un agente de borde que se conecta (o reconecta) recibe automáticamente el BF más reciente sin solicitud adicional. Solo existe un mensaje retained por país en el broker.

**ACL en EMQX:**
- El Edge Distribution Service tiene permiso de PUBLISH en `countries/+/bloom-filter`.
- Los agentes de borde tienen permiso de SUBSCRIBE en `countries/{su_country_code}/bloom-filter` (solo su país).

Ver [`docs/ingestion-mqtt/topic-schema.md`](../ingestion-mqtt/topic-schema.md) para el esquema completo de topics MQTT.

---

## 7. Restricciones de Parámetros del BF

Los parámetros del BF son **constantes fijas** en el servicio (no configurables por país):

```go
const (
    BFSizeBits    = 8_388_608  // 1 048 576 bytes = 1 MB exacto
    BFHashFuncs   = 6
    BFMaxElements = 1_000_000
    // Tasa de FP esperada: ~0.84% con estos parámetros
)
```

Si un país tiene más de 1 000 000 de placas hurtadas activas, el servicio emite una alerta operacional `bf_capacity_warning` pero sigue funcionando. La tasa de FP aumentará proporcionalmente. En ese caso, se evaluará aumentar `BFSizeBits` en una versión futura del protocolo (cambio que requiere actualización coordinada del agente de borde).

---

## 8. Métricas Prometheus

| Métrica | Tipo | Descripción |
|---|---|---|
| `edge_distribution_bf_generated_total{country_code}` | Counter | Total de BFs generados (delta + full snapshot) |
| `edge_distribution_bf_full_snapshot_total{country_code}` | Counter | Total de full snapshots emitidos |
| `edge_distribution_bf_element_count{country_code}` | Gauge | Número de elementos en el BF actual de cada país |
| `edge_distribution_bf_generation_duration_seconds{country_code}` | Histogram | Duración de la generación del BF (incluyendo query a PG) |
| `edge_distribution_mqtt_publish_total{country_code, type}` | Counter | Publicaciones MQTT (DELTA / FULL_SNAPSHOT) |
| `edge_distribution_mqtt_publish_errors_total{country_code}` | Counter | Fallos al publicar en MQTT |
| `edge_distribution_bf_size_bytes{country_code}` | Gauge | Tamaño del BF serializado antes de compresión |
| `edge_distribution_bf_compressed_size_bytes{country_code}` | Gauge | Tamaño del BF tras compresión LZ4 |
| `edge_distribution_storage_store_errors_total{country_code}` | Counter | Fallos al persistir versión en MinIO |
| `edge_distribution_bf_capacity_ratio{country_code}` | Gauge | Ratio `element_count / BFMaxElements` (alerta si > 0.9) |

---

## 9. Configuración del Servicio

| Variable | Descripción | Valor de referencia |
|---|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | Bootstrap servers | `kafka:9092` |
| `KAFKA_INPUT_TOPIC` | Tópico de entrada | `stolen.vehicles.canonical` |
| `KAFKA_CONSUMER_GROUP_ID` | Consumer group | `edge-distribution-service` |
| `DATABASE_URL` | PostgreSQL (lectura) | `postgresql://user:pass@pg-replica:5432/antihurto` |
| `MINIO_ENDPOINT` | Endpoint MinIO/S3 | `minio:9000` |
| `MINIO_BUCKET` | Bucket de versiones BF | `bloom-filters` |
| `MQTT_BROKER_URL` | URL del broker MQTT | `mqtt://emqx:1883` |
| `MQTT_CLIENT_ID` | Client ID del publisher | `edge-distribution-service` |
| `FULL_SNAPSHOT_THRESHOLD_DAYS` | Umbral para full snapshot | `7` |
| `BF_RETENTION_VERSIONS` | Versiones a retener por país | `48` (≈ 2 días a 1 delta/hora) |

---

## 10. Relación con Otros Componentes

| Componente | Interacción |
|---|---|
| Canonical Vehicles Service | Notifica cambios vía `BloomFilterNotifyPort`; publica en `stolen.vehicles.canonical` |
| PostgreSQL | Lectura de `canonical_vehicles` para construir el BF; usa réplica de lectura |
| MinIO / S3 | Persistencia de versiones BF via `BloomFilterStorePort` |
| EMQX MQTT Broker | Publicación del BF vía `MQTTPublisherPort` |
| Agente de Borde | Consumer del BF; especificación en [`docs/agente-borde/bloom-filter.md`](../agente-borde/bloom-filter.md) |
| ingestion-mqtt | Define el esquema del topic MQTT; el EDS lo usa; ver [`docs/ingestion-mqtt/topic-schema.md`](../ingestion-mqtt/topic-schema.md) |

---

## 11. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [`adr-009-bloom-filter-edge.md`](./adr-009-bloom-filter-edge.md) | Decisión de diseño que fundamenta este servicio |
| [`canonical-vehicles-service.md`](./canonical-vehicles-service.md) | Fuente de notificaciones de cambio |
| [`postgresql-schema.md`](./postgresql-schema.md) | Fuente de datos para construir el BF |
| [`sla-freshness.md`](./sla-freshness.md) | SLA de tiempo entre cambio y BF en el borde |
| [`slo-observability.md`](./slo-observability.md) | Alertas sobre el BF y la distribución |
