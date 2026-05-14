# Redis — Schema de Claves y Usos

**Componente:** Almacenamiento de lectura — capa Redis  
**Versión del documento:** 1.0  
**Referencia:** [adr-redis-topology.md](./adr-redis-topology.md) · [slo-observability.md](./slo-observability.md)

---

## 1. Propósito

Redis actúa como cache de baja latencia para los cuatro casos de uso del sistema:

1. **Hot-list de vehículos hurtados** — lookup en el hot path de alertas (SLO: GET < 1 ms p99).
2. **Cache de sesiones** — tokens de sesión Keycloak para el API Gateway.
3. **Rate-limiting** — contadores de ventana deslizante por API key y endpoint.
4. **Pub/sub de alertas** — distribución de alertas en tiempo real al WebSocket gateway.

---

## 2. Hot-List de Vehículos Hurtados

### 2.1 Esquema de Clave

```
stolen:{country_code}:{plate_normalized}
```

Donde:
- `country_code`: código ISO 3166-1 alpha-2 en **minúsculas** (ej. `co`, `mx`, `pe`).
- `plate_normalized`: matrícula normalizada en **mayúsculas**, sin guiones ni espacios (ej. `ABC123`).

**Ejemplo de clave:** `stolen:co:ABC123`

### 2.2 Valor

El valor es una cadena JSON con los metadatos mínimos del vehículo hurtado para presentar al oficial en la alerta:

```json
{
  "brand": "Toyota",
  "line": "Corolla",
  "model_year": 2020,
  "color": "Blanco",
  "stolen_at": "2026-03-15T10:00:00Z",
  "incident_id": "7a3f9c2e-...",
  "schema_version": 1
}
```

> El valor no incluye el DNI del propietario (dato PII sensible). El API Gateway lo recupera de PostgreSQL solo cuando el oficial tiene el scope `vehicles:pii_read` en su JWT.

### 2.3 TTL

```
TTL: 86400 segundos (24 horas)
```

El TTL de 24 h garantiza que la hot-list en Redis no se desincroniza con PostgreSQL más allá de un día. El `Canonical Vehicles Service` renueva el TTL con cada actualización del estado del vehículo.

### 2.4 Estrategia de Escritura y Lectura

```bash
# Escritura — SET con EX (TTL en segundos)
SET stolen:co:ABC123 '{"brand":"Toyota","line":"Corolla","model_year":2020,...}' EX 86400

# Lectura — GET simple (SLO: < 1 ms p99)
GET stolen:co:ABC123
# Responde: null (no hurtado) o el JSON del vehículo (hurtado)

# Invalidación — DEL cuando el vehículo es marcado como recuperado
DEL stolen:co:ABC123

# Renovación de TTL (al actualizar metadatos del vehículo)
SET stolen:co:ABC123 '{"brand":"Toyota",...,"stolen_at":"2026-03-15T10:00:00Z"}' EX 86400

# Verificar TTL actual de una clave
TTL stolen:co:ABC123

# Inspeccionar el patrón de claves de un país (uso diagnóstico, nunca en producción caliente)
SCAN 0 MATCH "stolen:co:*" COUNT 100
```

### 2.5 Sincronización con PostgreSQL

El `Canonical Vehicles Service` mantiene Redis sincronizado:

- **Al insertar un vehículo hurtado en PostgreSQL:** `SET stolen:{cc}:{plate} {json} EX 86400`.
- **Al cambiar estado a `recovered` o `cancelled`:** `DEL stolen:{cc}:{plate}`.
- **Proceso de recarga completa** (ej. después de un failover Redis sin persistencia):

```bash
# Script de recarga (ejecutar desde Canonical Vehicles Service)
# Selecciona todos los vehículos hurtados activos y los carga en Redis
SELECT country_code, plate_normalized, brand, line, model_year, color, stolen_at
FROM stolen_vehicles
WHERE status = 'stolen';
# Para cada fila: SET stolen:{cc}:{plate} {json} EX 86400
```

---

## 3. Cache de Sesiones

### 3.1 Esquema de Clave

```
session:{session_id}
```

Donde `session_id` es el `sid` del token Keycloak (JTI del refresh token).

**TTL:** 3600 s (1 hora, alineado con `accessTokenLifespan` de Keycloak).

```bash
# El API Gateway almacena la sesión validada para evitar re-validar el JWT en cada request
SET session:abc123def EX 3600 '{"user_id":"...", "country_code":"CO", "roles":["officer"]}'

# Lookup en cada request del API Gateway (< 1 ms p99)
GET session:abc123def

# Invalidación al hacer logout
DEL session:abc123def
```

---

## 4. Rate-Limiting

### 4.1 Esquema de Clave

```
ratelimit:{api_key}:{endpoint}:{window_start_ts}
```

Donde `window_start_ts` es el timestamp en segundos del inicio de la ventana actual (ej. el segundo actual, o el minuto actual).

```bash
# Incrementar contador de la ventana actual (ventana de 60 s)
INCR ratelimit:key123:search_plate:1747147800
EXPIRE ratelimit:key123:search_plate:1747147800 120  # TTL = 2 ventanas

# Leer el contador actual
GET ratelimit:key123:search_plate:1747147800
# Si el valor > límite configurado (ej. 100 req/min): rechazar con HTTP 429

# Usar transacción atómica con MULTI/EXEC para evitar race conditions:
MULTI
INCR ratelimit:key123:search_plate:1747147800
EXPIRE ratelimit:key123:search_plate:1747147800 120
EXEC
```

---

## 5. Pub/Sub de Alertas

### 5.1 Canales por País

```
alerts:{country_code}
```

**Ejemplo:** `alerts:co`, `alerts:mx`.

```bash
# Publisher (Alert Service) — publica una alerta para oficiales de Colombia
PUBLISH alerts:co '{"type":"stolen_vehicle_match","plate":"ABC123","event_id":"...","location":{"lat":4.71,"lon":-74.07},"incident_id":"..."}'

# Subscriber (WebSocket Gateway) — se suscribe al canal del país que gestiona
SUBSCRIBE alerts:co

# Resultado al recibir el mensaje:
# 1) "message"
# 2) "alerts:co"
# 3) '{"type":"stolen_vehicle_match","plate":"ABC123",...}'

# Patrón alternativo (un subscriber para múltiples países)
PSUBSCRIBE alerts:*
```

### 5.2 Consideraciones de Pub/Sub

- El pub/sub en Redis es **fire-and-forget**: si el WebSocket Gateway está temporalmente caído, el mensaje se pierde.
- Para alertas críticas que no pueden perderse, se persiste también en la tabla `incidents` de PostgreSQL. El WebSocket Gateway puede recuperar alertas pendientes al reconectarse.
- La persistencia de alertas en PostgreSQL actúa como fallback; el pub/sub Redis es el canal de baja latencia (< 10 ms extremo a extremo en la misma región).

---

## 6. Puertos Hexagonales (ADR-005)

Siguiendo la arquitectura hexagonal, Redis no es accedido directamente por la lógica de dominio. Se accede a través de dos puertos:

### 6.1 `StolenVehiclesPort`

Abstrae el lookup de la hot-list de vehículos hurtados. El `Matcher Service` solo conoce este puerto.

```typescript
// Puerto (interfaz de dominio — agnóstica de Redis)
interface StolenVehiclesPort {
  isStolen(countryCode: string, plateNormalized: string): Promise<StolenVehicleInfo | null>;
  markStolen(countryCode: string, plateNormalized: string, info: StolenVehicleInfo): Promise<void>;
  markRecovered(countryCode: string, plateNormalized: string): Promise<void>;
}

// Adaptador Redis (implementación)
class RedisStolenVehiclesAdapter implements StolenVehiclesPort {
  async isStolen(cc: string, plate: string): Promise<StolenVehicleInfo | null> {
    const key = `stolen:${cc.toLowerCase()}:${plate.toUpperCase()}`;
    const value = await this.redis.get(key);
    return value ? JSON.parse(value) : null;
  }

  async markStolen(cc: string, plate: string, info: StolenVehicleInfo): Promise<void> {
    const key = `stolen:${cc.toLowerCase()}:${plate.toUpperCase()}`;
    await this.redis.set(key, JSON.stringify(info), 'EX', 86400);
  }

  async markRecovered(cc: string, plate: string): Promise<void> {
    const key = `stolen:${cc.toLowerCase()}:${plate.toUpperCase()}`;
    await this.redis.del(key);
  }
}
```

### 6.2 `VehicleListPort`

Abstrae el acceso a la lista canónica de vehículos hurtados para sincronización y reporting.

```typescript
// Puerto (interfaz de dominio)
interface VehicleListPort {
  listStolenByCountry(countryCode: string, limit: number, offset: number): Promise<StolenVehicle[]>;
  countStolenByCountry(countryCode: string): Promise<number>;
  bulkLoad(vehicles: StolenVehicle[]): Promise<void>;
}

// En este caso el adaptador primario es PostgreSQL (vía VehicleListPostgresAdapter).
// Redis actúa como cache de escritura del adaptador, no como fuente de verdad.
```

---

## 7. Gestión de Fallos de Escritura en Redis (CR-07)

Los fallos de escritura en Redis (timeout de conexión, error de red, nodo caído antes del failover) **no interrumpen el pipeline de proyección principal**. El `Canonical Vehicles Service` aplica el patrón fire-and-forget con dead-letter para escrituras en la hot-list.

### 7.1 Comportamiento ante Fallo Redis

```
1. Canonical Vehicles Service intenta SET stolen:{cc}:{plate} ... EX 86400.
2. Redis retorna error (timeout o connection refused).
3. El servicio:
   a. Incrementa la métrica redis.write.failures{operation="stolen_list_set", country_code="CO"}.
   b. Encola el evento en el topic de reintento: vehicle.redis.retry (reintento con backoff).
   c. Continúa con la proyección a PostgreSQL y OpenSearch sin interrupción.
4. Un consumer dedicado del topic vehicle.redis.retry reintenta con backoff exponencial
   (1 s, 2 s, 4 s, máximo 3 reintentos). Si falla definitivamente, envía a dead-letter.
```

### 7.2 Métrica y Alerta

```promql
# Contador: redis.write.failures{operation="stolen_list_set"}
rate(redis_write_failures_total{operation="stolen_list_set"}[5m])

# Alerta si hay más de 5 fallos en 2 minutos:
# (ver slo-observability.md §2.6 para la definición de la métrica)
```

### 7.3 Garantía de Consistencia Post-Fallo

Cuando Redis se recupera del fallo, el proceso de recarga completa (§2.5) repopula la hot-list desde PostgreSQL. La ventana de inconsistencia (período entre el fallo de escritura y la recuperación) produce **falsos negativos temporales en el Matcher Service** (el Matcher no encuentra el vehículo en Redis pero el vehículo sí está hurtado). Esto es aceptable según el diseño del sistema: el Matcher tiene un fallback a Kafka Streams con Bloom filter (documentado en `backbone-procesamiento`) que actúa durante la ventana de inconsistencia de Redis.

---

## 8. Configuración de Persistencia Redis

Para sobrevivir reinicios del primario sin perder la hot-list:

```
# redis.conf — persistencia híbrida (AOF + RDB)
save 900 1          # RDB snapshot cada 15 min si hay al menos 1 cambio
save 300 10         # RDB snapshot cada 5 min si hay al menos 10 cambios
appendonly yes      # AOF habilitado
appendfsync everysec # fsync cada segundo (balance entre durabilidad y rendimiento)
no-appendfsync-on-rewrite yes
```

Con esta configuración, el RPO máximo ante fallo del primario sin réplica es de ~1 s (desde el último fsync del AOF). Con réplica síncrona activa, el RPO es 0.

---

## 8. Métricas Clave de Redis

```bash
# Hitrate de la hot-list (porcentaje de GETs que encuentran la clave)
redis-cli INFO stats | grep keyspace_hits
redis-cli INFO stats | grep keyspace_misses

# Número de claves por patrón (diagnóstico)
redis-cli DBSIZE

# Memoria usada
redis-cli INFO memory | grep used_memory_human

# Clientes conectados
redis-cli INFO clients | grep connected_clients
```

Alertas y dashboards completos en [slo-observability.md](./slo-observability.md).
