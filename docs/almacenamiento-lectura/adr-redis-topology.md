# ADR-RD-01 · Topología Redis: Sentinel vs. Cluster

**Estado:** Aceptado  
**Fecha:** 2026-05-13  
**Contexto:** Almacenamiento de lectura — capa Redis  
**Referencia:** ADR-005 (hexagonal) · [redis-schema.md](./redis-schema.md) · [slo-observability.md](./slo-observability.md)

---

## 1. Contexto

Redis cumple cuatro roles en el sistema:

1. **Hot-list de vehículos hurtados:** clave `stolen:{country_code}:{plate_normalized}` con TTL 24 h. El Matcher Service la consulta en cada evento (SLO: GET < 1 ms p99).
2. **Cache de sesiones Keycloak:** tokens de sesión y metadata de usuario.
3. **Rate-limiting:** contadores de ventana deslizante por `(api_key, endpoint)`.
4. **Pub/sub de alertas:** canal por país para distribución de alertas en tiempo real al WebSocket gateway.

Se evaluaron dos topologías de Redis de alta disponibilidad:

1. **Redis Sentinel** (primario + réplicas + sentinels)
2. **Redis Cluster** (sharding automático + replicación por shard)

---

## 2. Alternativas Evaluadas

### Alternativa A — Redis Sentinel

**Arquitectura:**
```
┌─────────────────────────────────────────────────┐
│  Sentinel 1   │  Sentinel 2   │  Sentinel 3    │
└───────┬───────┴───────┬───────┴───────┬────────┘
        │               │               │
        └───────────────┼───────────────┘
                        │ monitorea + vota
              ┌─────────┴──────────┐
              │    Primario        │ ── escribe
              │    Redis           │
              └────────┬───────────┘
               ┌───────┴────────┐
      ┌────────┴────────┐  ┌────┴──────────┐
      │   Réplica 1     │  │   Réplica 2   │
      │   Redis         │  │   Redis       │
      └─────────────────┘  └───────────────┘
```

El cliente descubre la dirección del primario conectándose a cualquier Sentinel. Si el primario falla, Sentinel orquesta la elección de una réplica como nuevo primario (quórum de 2/3 sentinels).

**Ventajas:**
- **Simplicidad operacional:** tres nodos Sentinel + un primario + N réplicas. Menos piezas móviles que Cluster.
- **Compatibilidad:** todos los comandos Redis son compatibles; no hay restricciones por slot/shard.
- **Pub/sub sin restricciones:** el pub/sub funciona sobre el primario sin necesidad de redirect a shards.
- **Apropiado para el volumen actual:** la hot-list de vehículos hurtados de Colombia es ~500 K claves; incluso con 10 países grandes no superará los 5–10 M claves, lo que cabe cómodamente en un primario de 8–16 GB RAM.
- **Failover automático:** RTO < 30 s en condiciones normales (tiempo de elección Sentinel).

**Desventajas:**
- **Un solo escritor:** toda la escritura pasa por el primario. Si el throughput de escritura supera ~100 K ops/s, el primario puede convertirse en cuello de botella.
- **Escalado de lectura limitado:** las réplicas pueden servir lecturas, pero los clientes deben implementar lectura desde réplicas explícitamente (Redis Sentinel no redirige automáticamente).

### Alternativa B — Redis Cluster

**Arquitectura:**
```
  Shard 1              Shard 2              Shard 3
┌──────────┐         ┌──────────┐         ┌──────────┐
│ Primario │──repl──►│ Primario │──repl──►│ Primario │
│ Réplica  │         │ Réplica  │         │ Réplica  │
└──────────┘         └──────────┘         └──────────┘
  slots 0-5460     slots 5461-10922    slots 10923-16383
```

**Ventajas:**
- **Escalado horizontal transparente:** el cliente redirige automáticamente al shard correcto por hash slot.
- **Throughput casi ilimitado:** se añaden shards conforme crece la carga.
- **Sin SPOF de escritura:** cada shard tiene su propio primario.

**Desventajas:**
- **Restricciones de comandos:** comandos que operan sobre múltiples claves en distintos slots (`MGET`, `SUNIONSTORE`, transacciones multi-clave) requieren que todas las claves estén en el mismo slot (hash tags). Impacta al pub/sub de alertas por país si los canales y los clientes están en slots distintos.
- **Pub/sub en Cluster:** el pub/sub en Redis Cluster solo se distribuye a clientes conectados al mismo shard donde se suscribieron; los mensajes no se propagan automáticamente a todos los shards. Esto hace que el canal de alertas necesite ajustes (conexión directa al shard del canal o Pub/Sub con hash tags forzados).
- **Complejidad operacional:** migración de slots, rebalanceo, resharding manual o semi-automático.
- **Sobredimensionamiento:** para el volumen esperado en Fase 1 y Fase 2, Redis Cluster es excesivo y agrega complejidad sin beneficio real.

---

## 3. Decisión

**Se adopta Alternativa A: Redis Sentinel como topología inicial.**

La topología Sentinel cubre los cuatro casos de uso del sistema con menor complejidad operacional. El volumen de claves y el throughput esperado están dentro del rango que un solo primario Redis gestiona sin problemas. La restricción de pub/sub en Cluster hace de Sentinel la opción más compatible con el diseño del WebSocket gateway.

---

## 4. Justificación Técnica

**Estimación de capacidad para Fase 2 (100 K devices, 10 países):**

| Métrica | Estimación |
|---|---|
| Claves hot-list (10 países × 500 K placas) | ~5 M claves |
| Tamaño promedio por clave (`stolen:{cc}:{plate}`) | ~200 bytes |
| RAM total hot-list | ~1 GB |
| Sesiones Keycloak activas | ~50 K × 2 KB = ~100 MB |
| Contadores rate-limiting | ~100 K × 50 bytes = ~5 MB |
| **Total estimado** | **~1.2 GB** |

Un primario Redis con 8 GB RAM tiene margen amplio para Fase 2. El throughput esperado de lecturas es ~5 K GET/s (matching de eventos) — dentro del rango de un primario Redis en hardware moderno (> 100 K ops/s en operaciones simples).

**Criterio de migración a Redis Cluster:**
- Tamaño del dataset > 50 % de la RAM del primario (p. ej. > 30 GB).
- Throughput de escritura sostenido > 80 K ops/s.
- Necesidad de distribución geográfica de los shards por regulación de residencia de datos.

---

## 5. Consecuencias

### Positivas

- Despliegue inicial simple: 3 sentinels + 1 primario + 2 réplicas = 6 pods en Kubernetes.
- Failover automático < 30 s; el SLO de Redis (GET < 1 ms p99) se recupera sin intervención manual.
- El pub/sub de alertas funciona sin restricciones de hash slot.
- El Helm chart (`bitnami/redis`) soporta modo Sentinel directamente con valores simples.

### Negativas / Riesgos

- **Ventana de failover (CR-09):** con la configuración de referencia (`downAfterMilliseconds: 5000`, `failoverTimeout: 18000`), el RTO objetivo es < 10 s: Sentinel detecta el fallo en ~5 s y completa la elección en ~3–5 s adicionales (quórum de 2/3). Las escrituras en tránsito durante el failover se reintentan con backoff exponencial en el cliente (`maxRetries: 3`, `retryDelayMs: 200`). Las claves `stolen:{country_code}:{plate_normalized}` escritas en los 30 s previos al fallo son eventualmente consistentes tras la resincronización de la réplica promovida (replicación semi-síncrona de Redis).
- **Escalado de escritura limitado:** si el throughput de escritura supera el umbral, requiere migración a Cluster. Los puertos hexagonales (`StolenVehiclesPort`) facilitan este cambio sin afectar la lógica del Matcher Service.

### Configuración de quórum

Se requieren al menos 2 de 3 sentinels para declarar el primario como inaccesible y ordenar el failover. Con 3 sentinels desplegados en 3 nodos Kubernetes distintos (con `podAntiAffinity`), la pérdida de una AZ no impide el quórum.

---

## 6. Helm Values — Topología Sentinel

```yaml
# values-redis-sentinel.yaml (fragmento)
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
replica:
  replicaCount: 2
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: kubernetes.io/hostname
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

---

## 7. Plan de Evolución

| Fase | Criterio de activación | Acción |
|---|---|---|
| **Fase 1** | Dataset < 5 GB, throughput < 20 K ops/s | Sentinel (primario + 2 réplicas + 3 sentinels) |
| **Fase 2** | Dataset > 15 GB o throughput > 50 K ops/s | Evaluar Redis Cluster; migrar con adaptadores hexagonales; el cambio es transparente para los servicios |
| **Fase 3** | Residencia de datos por país con regulación estricta | Redis Cluster con shards en regiones específicas; o instancias Redis independientes por país |

---

## 8. Referencias

- [redis-schema.md](./redis-schema.md) — Claves, TTL, usos secundarios y puertos hexagonales.
- [slo-observability.md](./slo-observability.md) — Métricas de hitrate, latencia y tamaño del dataset Redis.
- [helm/README.md](./helm/README.md) — Values Helm para despliegue Sentinel.
- ADR-005 en [propuesta-arquitectura-hurto-vehiculos.md](../propuesta-arquitectura-hurto-vehiculos.md).
