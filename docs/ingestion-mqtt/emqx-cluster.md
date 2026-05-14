# Clúster EMQX 5.x — Configuración

**Pilar:** 2 — Comunicación Segura y Eficiente  
**ADRs aplicados:** ADR-002 · ADR-010 · ADR-011  
**Referencia:** [Visión General](./overview.md) · [topic-schema.md](./topic-schema.md) · [security.md](./security.md) · [bridge.md](./bridge.md)  
**Última actualización:** 2026-05-13

---

## 1. Topología del Clúster

El clúster EMQX se despliega con **3 nodos** en Kubernetes distribuidos en anti-afinidad de zona (una instancia por AZ). Esta configuración garantiza tolerancia al fallo de un nodo completo (CR-04) y mantiene quórum para la gestión de sesiones distribuidas.

```
Nodo 1 (AZ-1): emqx-0   Nodo 2 (AZ-2): emqx-1   Nodo 3 (AZ-3): emqx-2
     ├── sesiones propias       ├── sesiones propias       ├── sesiones propias
     └── replicación Mnesia     └── replicación Mnesia     └── replicación Mnesia
```

**Parámetros de alta disponibilidad:**

| Parámetro | Valor | Justificación |
|---|---|---|
| Número de nodos | 3 | Quórum mínimo para decisiones distribuidas |
| Anti-afinidad | topologyKey: `topology.kubernetes.io/zone` | Un nodo por AZ, tolerancia a falla de AZ |
| `podAntiAffinity` | `requiredDuringSchedulingIgnoredDuringExecution` | Garantía fuerte, no preferencia |
| `PodDisruptionBudget.minAvailable` | 2 | Máximo 1 nodo caído durante mantenimiento |
| Estrategia de actualización | `RollingUpdate` con `maxUnavailable: 1` | Sin downtime en rollouts |
| Discovery | EMQX Operator (CRD) o `emqx.cluster.discovery_strategy: k8s` | Auto-descubrimiento vía API de Kubernetes |

---

## 2. Configuración mTLS

### 2.1 Parámetros TLS del Listener MQTT

El listener principal opera en el puerto 8883 (MQTT sobre TLS). La configuración mínima requerida:

```hocon
listeners.ssl.default {
  bind = "0.0.0.0:8883"
  ssl_options {
    # Versión mínima TLS 1.3 (ADR-010, CA-02)
    versions = ["tlsv1.3"]

    # Certificado del servidor (EMQX)
    certfile  = "/etc/emqx/certs/server.crt"
    keyfile   = "/etc/emqx/certs/server.key"
    cacertfile = "/etc/emqx/certs/ca-chain.crt"

    # mTLS obligatorio: se exige certificado de cliente
    verify = verify_peer
    fail_if_no_peer_cert = true

    # Suites de cifrado — solo suites TLS 1.3
    ciphers = [
      "TLS_AES_256_GCM_SHA384",
      "TLS_AES_128_GCM_SHA256",
      "TLS_CHACHA20_POLY1305_SHA256"
    ]

    # OCSP Stapling (para rendimiento; la verificación activa se hace en AuthZ)
    ocsp {
      enable_ocsp_stapling = true
      responder_url        = "http://vault.internal:8200/v1/pki/ocsp"
      refresh_http_timeout = "15s"
      refresh_interval     = "5m"
    }
  }
  # Tasa máxima de conexiones nuevas por segundo (protección contra tormentas)
  max_connections = 1000000
  limiter.connection.rate = "5000/s"
}
```

### 2.2 Suites de Cifrado

Solo se admiten suites TLS 1.3. No se habilita compatibilidad con TLS 1.2 ni TLS 1.1 para las conexiones de dispositivos. Si un dispositivo no puede negociar TLS 1.3, la conexión es rechazada.

### 2.3 Verificación de Certificado de Cliente

EMQX verifica el certificado del dispositivo contra la CA raíz almacenada en `cacertfile`. El `device_id` se extrae del campo CN del `Subject` o del primer SAN de tipo `DNS` o `URI` del certificado.

La verificación de revocación activa vía OCSP se configura en el plugin de autenticación (ver §2.4).

### 2.4 Integración OCSP con Vault

La verificación OCSP se realiza en dos niveles:

1. **OCSP Stapling (TLS handshake):** EMQX obtiene periódicamente la respuesta OCSP del servidor del certificado del servidor (no del cliente) y la incluye en el handshake TLS. Esto reduce la latencia del handshake inicial.

2. **Verificación OCSP del certificado de cliente (AuthN):** configurada como parte del plugin de autenticación TLS de EMQX:

```hocon
authentication = [
  {
    mechanism = tls_cert
    backend    = built_in_database

    # Verificación OCSP del certificado de cliente
    check_peer_cert_via_ocsp = true
    ocsp_responder_urls      = ["http://vault.internal:8200/v1/pki/ocsp"]
    ocsp_request_timeout     = "5s"

    # Comportamiento cuando el responder OCSP no está disponible:
    # - "allow": permite la conexión (modo permisivo, aceptable para GSM intermitente)
    # - "deny": deniega la conexión (modo estricto)
    ocsp_unavailable_action  = "allow"

    # Caché de respuestas OCSP para reducir carga en Vault
    ocsp_response_cache_ttl  = "5m"

    # Extracción de device_id del certificado
    extract_subject_fields = {
      cn = "device_id"
    }
  }
]
```

**Comportamiento cuando OCSP no está disponible:** `ocsp_unavailable_action = "allow"` asegura que los dispositivos con certificados válidos no sean bloqueados por una indisponibilidad temporal del servidor OCSP (p. ej., durante un mantenimiento de Vault). Esta decisión prioriza la disponibilidad del servicio de borde sobre la verificación estricta de revocación en tiempo real. La CRL se usa como fallback (ver [security.md §3.2](./security.md)).

---

## 3. ACL por device_id y country_code

### 3.1 Plugin de Autorización

```hocon
authorization {
  sources = [
    {
      type   = built_in_database
      enable = true
    }
  ]
  # Acción por defecto cuando ninguna regla coincide
  no_match = deny
  # Registrar todas las violaciones de ACL
  deny_action = disconnect
}
```

### 3.2 Reglas ACL en AuthZ Plugin

Las reglas se configuran a nivel de template. El símbolo `${username}` se resuelve al `device_id` extraído del certificado. El símbolo `${clientid}` es el `client_id` MQTT (debe coincidir con `device_id`).

```hocon
# Reglas de publicación (PUBLISH) — dispositivo puede publicar solo en sus propios topics
{action = publish, topic = "devices/${username}/events",    permission = allow}
{action = publish, topic = "devices/${username}/health",    permission = allow}
{action = publish, topic = "devices/${username}/image-uri", permission = allow}

# Reglas de suscripción (SUBSCRIBE) — dispositivo puede suscribirse a sus topics de recepción
{action = subscribe, topic = "devices/${username}/config",           permission = allow}
{action = subscribe, topic = "devices/${username}/ota",              permission = allow}

# El country_code se extrae del prefijo del device_id (primeros 2 caracteres)
# EMQX 5.x permite funciones en rules; alternativamente se usa un campo custom del cert
{action = subscribe, topic = "countries/+/bloom-filter", permission = allow}
# Nota: la restricción por country_code específico se aplica vía regla adicional
# usando el campo SAN "country_code" del certificado si está disponible,
# o mediante el AuthZ plugin con verificación del prefijo del client_id.

# Denegar todo lo demás (implícito por no_match = deny)
```

**Restricción de country_code en bloom-filter:** si el certificado incluye un SAN personalizado `country_code`, se puede refinar la regla a `countries/${cert.country_code}/bloom-filter`. En ausencia de este SAN, la restricción se implementa extrayendo los dos primeros caracteres del `device_id` en la lógica de autorización o mediante un JWT emitido por Vault con el claim `country_code`.

### 3.3 Registro de Violaciones

Toda violación ACL (código de razón `0x87`) genera una entrada en el log de auditoría con:
- `timestamp`, `client_id`, `device_id`, `topic_attempted`, `action`, `reason: acl_violation`

---

## 4. Límites de Payload por Topic

Los límites se configuran a nivel de broker (límite global) y pueden refinarse por topic mediante el Rule Engine.

```hocon
mqtt {
  # Límite global de paquete (protección general)
  max_packet_size = 1MB

  # Session expiry — finito por restricción de memoria (CR-05)
  session_expiry_interval = "2h"

  # Keep-alive negociado con el cliente (ADR-002)
  keepalive_multiplier = 1.5  # timeout = keepalive * 1.5 = 90s para keepalive=60s

  # Mensajes máximos en vuelo por sesión (QoS 1)
  max_inflight = 32
}
```

**Límites por topic** (implementados en el Rule Engine con regla de descarte):

| Topic pattern | Límite configurado |
|---|---|
| `devices/+/events` | 4 KB |
| `devices/+/health` | 2 KB |
| `devices/+/image-uri` | 512 bytes |
| `devices/+/config` | 16 KB |
| `countries/+/bloom-filter` | 256 KB |
| `devices/+/ota` | 4 KB |

Los mensajes que superen estos límites son descartados por el Rule Engine con log de error. El cliente recibe `0x95 Packet Too Large` si el límite global del broker es excedido; para límites de topic el Rule Engine descarta y no hay PUBACK (el cliente reintentará hasta agotar retries).

---

## 5. Cálculo del session_expiry_interval

El `session_expiry_interval` determina cuánto tiempo EMQX retiene el estado de una sesión (suscripciones, mensajes QoS 1 pendientes) cuando el dispositivo está desconectado.

**Presupuesto de memoria por nodo:**

| Parámetro | Valor estimado |
|---|---|
| RAM por nodo EMQX | 4 GB (mínimo recomendado) |
| RAM reservada para OS y overhead | 1 GB |
| RAM disponible para sesiones | ~3 GB |
| Memoria por sesión activa (estimado) | ~10–15 KB |
| Número máximo de sesiones activas por nodo | ~200.000 |
| Número máximo de dispositivos por nodo | ~333.000 (con factor de distribución de 3 nodos) |

**Cálculo del intervalo:**

Para un despliegue inicial de 100.000 dispositivos distribuidos en 3 nodos (~33.333 por nodo), el presupuesto de memoria es holgado. El intervalo de expiración se fija en **2 horas** considerando:

- La especificación de edge menciona hasta 4 horas de tolerancia a corte de energía + GSM.
- Un `session_expiry_interval` de 2 horas cubre la mayoría de los cortes de GSM intermitente (escenario operacional típico).
- Si el dispositivo está offline más de 2 horas, EMQX libera la sesión y el agente inicia sesión nueva al reconectar; los eventos pendientes ya están almacenados localmente en SQLite.
- Para despliegues de 1 M de dispositivos, se revisará el cálculo y se reducirá el intervalo o se aumentará el hardware de los nodos.

```hocon
mqtt.session_expiry_interval = "2h"
```

**Alerta operacional:** si el número de sesiones activas supera el 80 % del presupuesto del nodo, se activa una alerta (ver [operations.md §1](./operations.md)).

---

## 6. Keep-alive y Parámetros de Sesión

```hocon
mqtt {
  keepalive_multiplier = 1.5
  # El agente configura keep_alive = 60s (ADR-002)
  # Timeout de EMQX = 60 * 1.5 = 90 s
}
```

El agente de borde establece `keep_alive_s = 60` en la conexión MQTT (ver [`docs/agente-borde/uploader.md §2.1`](../agente-borde/uploader.md)). El broker considera la sesión caída si no recibe ningún paquete en 90 segundos.

---

## 7. Configuración del Kafka Data Bridge en el Rule Engine

El Kafka Data Bridge se configura en EMQX como recurso de salida del Rule Engine. Ver la especificación operacional completa en [bridge.md](./bridge.md).

**Configuración del recurso Kafka:**

```hocon
bridges.kafka.vehicle_events_bridge {
  enable = true

  # Kafka cluster (SASL/TLS)
  bootstrap_hosts = "kafka-1:9093,kafka-2:9093,kafka-3:9093"
  connect_timeout = "5s"

  authentication {
    mechanism = scram_sha_512
    username  = "${KAFKA_SASL_USERNAME}"
    password  = "${KAFKA_SASL_PASSWORD}"
  }

  ssl {
    enable    = true
    verify    = verify_peer
    versions  = ["tlsv1.3"]
    cacertfile = "/etc/emqx/certs/kafka-ca.crt"
  }

  # Productor Kafka
  producer {
    topic             = "vehicle.events.raw"
    partition_strategy = key_dispatch
    message_key       = "${clientid}"        # == device_id
    message_value     = "${payload}"         # JSON del mensaje MQTT
    compression       = no_compression
    required_acks     = all_isr              # acks=all
    max_inflight      = 10
    buffer {
      mode            = memory
      per_partition_limit = "100MB"
      segment_bytes   = "10MB"
      memory_overload_protection = true
    }
  }
}
```

---

## 8. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [security.md](./security.md) | Flujo mTLS completo, OCSP, rotación de certificados |
| [topic-schema.md](./topic-schema.md) | Definición de topics y límites de payload |
| [bridge.md](./bridge.md) | Reglas SQL del Rule Engine y métricas del bridge |
| [operations.md](./operations.md) | Runbooks de escalado, tormentas de reconexión |
| [helm/README.md](./helm/README.md) | Valores Helm para el despliegue del clúster |
| [`docs/identidad-seguridad/vault-pki-device.md`](../identidad-seguridad/vault-pki-device.md) | Emisión y rotación de certificados de dispositivo |
| [`docs/identidad-seguridad/mtls-device-policy.md`](../identidad-seguridad/mtls-device-policy.md) | Política mTLS detallada |
