# Esquema de Topics MQTT

**Pilar:** 2 — Comunicación Segura y Eficiente  
**ADR gobernante:** ADR-002  
**Referencia:** [Visión General](./overview.md) · [EMQX Cluster](./emqx-cluster.md) · [Bridge](./bridge.md)  
**Última actualización:** 2026-05-13

---

## 1. Tabla Autoritativa de Topics

La siguiente tabla define los seis topics MQTT del sistema. Es el contrato compartido entre el agente de borde, el clúster EMQX y el bridge MQTT→Kafka. Cualquier cambio requiere versionado coordinado.

| # | Topic pattern | Dirección | Contenido | QoS | Retained | Payload máx. |
|---|---|---|---|---|---|---|
| 1 | `devices/{device_id}/events` | Device → Broker | Metadatos del evento de placa (matrícula, timestamp, GPS, confidence, image_uri) | 1 | false | 4 KB |
| 2 | `devices/{device_id}/health` | Device → Broker | Health beacon del dispositivo (CPU, RAM, disco, batería, GSM, drift NTP) | 1 | false | 2 KB |
| 3 | `devices/{device_id}/image-uri` | Device → Broker | URI de imagen ya almacenada en object storage | 1 | false | 512 bytes |
| 4 | `devices/{device_id}/config` | Broker → Device | Configuración remota del agente (TOML/JSON serializado) | 1 | **true** | 16 KB |
| 5 | `countries/{country_code}/bloom-filter` | Broker → Device | Delta del Bloom filter de placas hurtadas del país | 1 | **true** | 256 KB |
| 6 | `devices/{device_id}/ota` | Broker → Device | Notificación de actualización de firmware con URL firmada | 1 | **true** | 4 KB |

**Notas:**
- Los topics 1–3 son de publicación exclusiva del dispositivo (dirección Device→Broker).
- Los topics 4–6 son de suscripción exclusiva del dispositivo (dirección Broker→Device); la publicación en estos topics solo está autorizada para servicios de la plataforma con rol `platform-service`.
- El payload máximo por topic es aplicado como límite en la configuración EMQX (ver [emqx-cluster.md §4](./emqx-cluster.md)). Mensajes que excedan el límite reciben código de razón MQTT 5 `0x95 Packet Too Large` (CR-03).

---

## 2. Restricciones ACL por Topic

Las reglas de control de acceso se basan en el `device_id` y `country_code` extraídos del CN/SAN del certificado mTLS del dispositivo. Un dispositivo autenticado con `device_id = D` y `country_code = CC` recibe exactamente los siguientes permisos:

| Topic | Permiso PUBLISH | Permiso SUBSCRIBE |
|---|---|---|
| `devices/D/events` | Permitido | Denegado |
| `devices/D/health` | Permitido | Denegado |
| `devices/D/image-uri` | Permitido | Denegado |
| `devices/D/config` | Denegado | Permitido |
| `countries/CC/bloom-filter` | Denegado | Permitido |
| `devices/D/ota` | Denegado | Permitido |
| `devices/+/events` (cualquier otro `device_id`) | **Denegado** | Denegado |
| `devices/+/health` (cualquier otro `device_id`) | **Denegado** | Denegado |
| `devices/+/image-uri` (cualquier otro `device_id`) | **Denegado** | Denegado |
| Cualquier otro topic | Denegado | Denegado |

**Consecuencia de violación ACL:** el broker rechaza el `PUBLISH` con código de razón MQTT 5 `0x87 Not Authorized`, desconecta al cliente y registra la violación en el log de auditoría (CA-04). Ver [security.md §4](./security.md) para la especificación detallada de las reglas ACL en EMQX.

**Roles de plataforma:** los servicios internos (Config Manager, OTA Dispatcher, Edge Distribution Service) se autentican con certificados propios o JWT de servicio con claim `role: platform-service`. Solo estos clientes pueden publicar en los topics 4–6.

---

## 3. Reglas de Formato de Identificadores

### 3.1 device_id

Formato canónico:

```
{CC}-{CITY}-DEV-{NNNNN}
```

| Parte | Descripción | Ejemplo |
|---|---|---|
| `CC` | Código ISO 3166-1 alpha-2 del país. Dos letras mayúsculas. | `CO` |
| `CITY` | Código de ciudad de tres letras mayúsculas (definido en onboarding). | `BOG` |
| `DEV` | Literal fijo que identifica el tipo de entidad como dispositivo. | `DEV` |
| `NNNNN` | Número secuencial de cinco dígitos con ceros a la izquierda, único por país+ciudad. | `00142` |

**Validación:** expresión regular `^[A-Z]{2}-[A-Z]{3}-DEV-\d{5}$`

**Ejemplos válidos:** `CO-BOG-DEV-00142`, `MX-MEX-DEV-00001`, `PE-LIM-DEV-09999`

**Restricción:** el `device_id` en el topic MQTT **debe coincidir exactamente** con el campo CN o SAN del certificado del cliente. Una discrepancia resulta en rechazo ACL. El `client_id` MQTT debe ser igual al `device_id`.

### 3.2 country_code

Formato: código ISO 3166-1 alpha-2 en mayúsculas. Dos letras.

**Validación:** expresión regular `^[A-Z]{2}$`

**Ejemplos:** `CO` (Colombia), `MX` (México), `PE` (Perú), `AR` (Argentina), `BR` (Brasil)

**Restricción:** el `country_code` en el topic `countries/{country_code}/bloom-filter` debe corresponder al prefijo del `device_id` del dispositivo suscriptor. Un dispositivo `CO-BOG-DEV-00142` solo puede suscribirse a `countries/CO/bloom-filter`, no a `countries/MX/bloom-filter`.

---

## 4. Ejemplos de Payload por Topic

### 4.1 Topic: `devices/{device_id}/events`

**Serialización:** JSON (Content-Type implícito por el topic)  
**Codificación:** UTF-8

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "device_id": "CO-BOG-DEV-00142",
  "country_code": "CO",
  "plate": "ABC123",
  "confidence": 0.97,
  "timestamp_wall": "2026-05-13T14:32:10.123Z",
  "timestamp_mono_ms": 8472913,
  "lat": 4.710989,
  "lon": -74.072092,
  "altitude_m": 2600,
  "heading_deg": 270,
  "speed_kmh": 45,
  "image_uri": "s3://ceiba-evidence-co/images/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000.jpg",
  "image_unavailable": false,
  "camera_id": "cam_front",
  "upload_mode": "stolen_only",
  "agent_version": "1.4.2"
}
```

**msgpack equivalente** (para optimizar ancho de banda GSM): el agente puede serializar el mismo objeto en msgpack usando las mismas claves de cadena. EMQX no diferencia la serialización a nivel de topic; el consumidor Kafka detecta el Content-Type via la propiedad MQTT 5 `Content-Type` si el agente la establece.

### 4.2 Topic: `devices/{device_id}/health`

**Serialización:** msgpack (binario compacto)  
**Esquema equivalente JSON para documentación:**

```json
{
  "device_id": "CO-BOG-DEV-00142",
  "timestamp": "2026-05-13T14:32:00Z",
  "cpu_pct": 12,
  "ram_used_mb": 312,
  "ram_total_mb": 1024,
  "disk_used_gb": 4.2,
  "disk_total_gb": 16,
  "battery_pct": 87,
  "battery_charging": false,
  "gsm_rssi_dbm": -85,
  "gsm_network": "3G",
  "ntp_drift_ms": 12,
  "queue_depth_events": 14,
  "queue_depth_images": 3,
  "cert_expires_at": "2026-07-20T00:00:00Z",
  "cert_expired": false,
  "uptime_s": 86400
}
```

### 4.3 Topic: `devices/{device_id}/image-uri`

**Serialización:** JSON

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "device_id": "CO-BOG-DEV-00142",
  "image_uri": "s3://ceiba-evidence-co/images/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000.jpg",
  "thumbnail_uri": "s3://ceiba-evidence-co/thumbnails/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000-thumb.jpg",
  "content_type": "image/jpeg",
  "size_bytes": 184320,
  "uploaded_at": "2026-05-13T14:32:15.456Z"
}
```

### 4.4 Topic: `devices/{device_id}/config`

**Serialización:** JSON  
**Retained:** true (el agente recibe la configuración vigente al suscribirse)

```json
{
  "version": "1.4.2",
  "upload_mode": "stolen_only",
  "bloom_filter_version": 142,
  "max_payload_bytes": 4096,
  "keep_alive_s": 60,
  "event_batch_size": 1,
  "image_max_kb": 500,
  "log_level": "info"
}
```

### 4.5 Topic: `countries/{country_code}/bloom-filter`

**Serialización:** binario (msgpack con delta del Bloom filter)  
**Retained:** true

```json
{
  "country_code": "CO",
  "version": 143,
  "delta_type": "add",
  "plates": ["XYZ789", "ABC456", "DEF123"],
  "full_hash": "sha256:abcdef1234567890...",
  "generated_at": "2026-05-13T14:00:00Z"
}
```

*Nota: el payload real es binario (msgpack). El esquema JSON se incluye solo como referencia.*

### 4.6 Topic: `devices/{device_id}/ota`

**Serialización:** JSON  
**Retained:** true (se limpia una vez el dispositivo confirma la actualización)

```json
{
  "version": "1.5.0",
  "download_url": "https://ota.ceiba-antihurto.io/agent/v1.5.0/agent-linux-arm64",
  "signature_url": "https://ota.ceiba-antihurto.io/agent/v1.5.0/agent-linux-arm64.sig",
  "sha256": "abc123def456...",
  "min_version": "1.3.0",
  "mandatory": false,
  "released_at": "2026-05-12T10:00:00Z"
}
```

---

## 5. Comportamiento de Backpressure en el Dispositivo

Cuando el broker EMQX rechaza un mensaje (por payload excesivo, rate limiting o indisponibilidad), el agente de borde aplica backoff exponencial según la especificación en [`docs/agente-borde/uploader.md §5.2`](../agente-borde/uploader.md). Los parámetros de reconexión no modifican el esquema de topics.

---

## 6. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [emqx-cluster.md](./emqx-cluster.md) | Límites de payload por topic configurados en EMQX; reglas ACL detalladas |
| [bridge.md](./bridge.md) | Reglas SQL del Rule Engine que filtran sobre estos topics |
| [security.md](./security.md) | Validación mTLS y extracción de `device_id` del certificado |
| [`docs/agente-borde/uploader.md`](../agente-borde/uploader.md) | Implementación cliente de publicación en topics 1–3 |
| [`docs/agente-borde/health-beacon.md`](../agente-borde/health-beacon.md) | Payload del topic `health` |
| [`docs/agente-borde/config-ota-manager.md`](../agente-borde/config-ota-manager.md) | Suscripción a topics `config` y `ota` |
| [`docs/agente-borde/bloom-filter.md`](../agente-borde/bloom-filter.md) | Suscripción al topic `bloom-filter` |
