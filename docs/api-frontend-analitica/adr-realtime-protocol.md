# ADR-RT-01 — Protocolo de Comunicación en Tiempo Real

**Estado:** Aceptado  
**Fecha:** 2026-05-13  
**Autores:** Equipo de arquitectura  
**Componente:** `api-frontend-analitica` — `alerts-service`

---

## Contexto

El sistema debe entregar alertas de vehículos hurtados a oficiales de policía conectados con una latencia end-to-end de **p95 < 2 s** desde que el Matcher Service confirma el match hasta que la notificación llega al navegador del oficial. El `alerts-service` actúa como el gateway de distribución de estas alertas.

Los requisitos operacionales son:

1. **Latencia p95 < 2 s end-to-end** (Kafka consumer → WebSocket delivery).
2. **Reconexión automática** en caso de caída del servidor o interrupción de red del cliente.
3. **Enrutamiento por zona:** cada oficial solo recibe alertas de su zona geográfica (`zone` claim del JWT). El servidor debe filtrar eficientemente.
4. **Escala multi-réplica en K8s:** múltiples instancias del `alerts-service` deben poder servir a todos los clientes sin pérdida de mensajes.
5. **Complejidad de implementación razonable** para el equipo de desarrollo.

---

## Opciones Evaluadas

### Opción A — WebSocket (Socket.IO)

**Descripción.** Protocolo full-duplex sobre TCP que mantiene una conexión persistente entre cliente y servidor. Socket.IO es la librería de más alto nivel sobre WebSocket que agrega rooms, reconnection automática, fallback a polling, y adapters para multi-servidor.

| Criterio | Evaluación |
|---|---|
| Latencia p95 < 2 s | Excelente. Entrega sub-100 ms después de que el mensaje llega al servidor. |
| Reconexión automática | Nativa en Socket.IO (exponential backoff, reconnection delay configurable). |
| Enrutamiento por zona | Socket.IO rooms: cada oficial se une a `room:CO:BOG-NORTE`; el servidor emite al room. |
| Multi-réplica K8s | Socket.IO Redis adapter (`@socket.io/redis-adapter`): sincroniza rooms y eventos entre réplicas. |
| Complejidad de implementación | Media. Socket.IO simplifica reconnection y rooms respecto a WS nativo. |
| Soporte de cliente | Excelente. Librería cliente oficial para React, Angular, etc. |

**Ventajas:**
- Full-duplex: el oficial puede enviar confirmación de recibo (ACK) sin abrir nueva conexión HTTP.
- Fallback automático a HTTP long polling si WebSocket es bloqueado por proxies corporativos.
- Socket.IO Redis adapter resuelve el problema de multi-réplica sin cambios en la lógica de negocio.
- Reconexión transparente con buffering de eventos no enviados durante la reconexión.

**Desventajas:**
- Las conexiones persistentes requieren sticky sessions o Redis adapter para K8s con HPA.
- Ligero overhead de protocolo Socket.IO vs. WS nativo (framing adicional).

---

### Opción B — Server-Sent Events (SSE)

**Descripción.** Protocolo unidireccional (servidor → cliente) sobre HTTP/1.1 o HTTP/2. El cliente abre una conexión y el servidor envía eventos como stream de texto.

| Criterio | Evaluación |
|---|---|
| Latencia p95 < 2 s | Buena. Sub-100 ms de entrega. Sin overhead de WebSocket handshake en reconexión. |
| Reconexión automática | Nativa en el estándar (el navegador reconecta automáticamente). Sin control sobre el delay. |
| Enrutamiento por zona | No nativo. Requiere implementación custom de suscripción por zona en el servidor. |
| Multi-réplica K8s | Más simple que WebSocket (HTTP stateless-like), pero requiere Redis pub/sub para fan-out entre réplicas. |
| Complejidad de implementación | Baja para casos simples; crece al agregar filtrado por zona. |
| Soporte de cliente | Nativo en navegadores (EventSource API). |

**Desventajas:**
- Unidireccional: no permite ACKs ni mensajes del oficial hacia el servidor sin abrir una nueva conexión HTTP.
- Limitación de conexiones por dominio en HTTP/1.1 (usualmente 6). Con HTTP/2 se multiplexan, pero complica el load balancing con sticky sessions.
- Sin mecanismo nativo de rooms/rooms filtradas por zona.

---

### Opción C — Long Polling

**Descripción.** El cliente hace un request HTTP y el servidor lo mantiene abierto hasta que hay un mensaje o expira un timeout. Al recibir un mensaje o timeout, el cliente hace un nuevo request.

| Criterio | Evaluación |
|---|---|
| Latencia p95 < 2 s | Aceptable, pero depende del tiempo de reconexión (típicamente 0–500 ms de gap entre requests). |
| Reconexión automática | Manual. El cliente debe implementar la lógica de retry. |
| Enrutamiento por zona | Custom. Cada request lleva parámetros de zona; el servidor mantiene estado en Redis. |
| Multi-réplica K8s | Stateless HTTP: más fácil de escalar horizontalmente. |
| Complejidad de implementación | Alta. La gestión de timeouts, concurrencia y ordering de mensajes es compleja. |
| Overhead de red | Alto. Un nuevo TCP handshake + TLS handshake por polling. |

**Desventajas:**
- La latencia end-to-end puede superar los 2 s si hay muchos clientes y el servidor está ocupado procesando reconexiones.
- Overhead de red muy alto comparado con las otras opciones.
- Difícil de depurar y monitorear.

---

## Decisión

**Se adopta WebSocket con Socket.IO como protocolo de tiempo real para el `alerts-service`.**

**Justificación:**

1. Socket.IO es la única opción que cumple todos los requisitos: latencia < 2 s, reconexión automática, enrutamiento por zona via rooms, y escala multi-réplica via Redis adapter.
2. El fallback automático a long polling garantiza que oficiales en redes corporativas restrictivas puedan recibir alertas.
3. La bidireccionalidad permite implementar ACKs de entrega (el oficial confirma recepción), útil para métricas de `alerts_delivered`.
4. La integración con React a través del cliente oficial `socket.io-client` es directa y bien documentada.

---

## Consecuencias Operacionales

### Sticky Sessions en K8s

Con múltiples réplicas del `alerts-service`, los clientes WebSocket deben reconectarse al mismo pod o usar el Redis adapter para sincronización de rooms. Se recomiendan ambas estrategias en conjunto:

**Estrategia primaria — Socket.IO Redis Adapter:**
```yaml
# K8s deployment env
REDIS_HOST: redis-master.default.svc.cluster.local
REDIS_PORT: 6379
# En el código Node.js/NestJS:
# import { createAdapter } from "@socket.io/redis-adapter";
# const pubClient = createClient({ host: REDIS_HOST });
# const subClient = pubClient.duplicate();
# io.adapter(createAdapter(pubClient, subClient));
```

Con el Redis adapter, cualquier instancia puede emitir a cualquier room y el mensaje se distribuye a todas las réplicas. No se requieren sticky sessions.

**Estrategia secundaria — Sticky Sessions (defense in depth):**
Si el Redis adapter tiene latencia adicional inaceptable, se puede configurar sticky sessions via Kong:
```yaml
# KongIngress annotation
konghq.com/override: "alerts-service-upstream"
# Upstream config: hash_on: cookie, hash_on_cookie: io.sid
```

### Estado de Conexiones en Redis

El schema de estado en Redis se define en [`alerts-service.md §5`](./alerts-service.md). Cada conexión WebSocket activa tiene una clave:

```
ws:conn:{connection_id} → { country_code, zone, officer_id, connected_at }
TTL: 300s (renovado en cada heartbeat Socket.IO)
```

### Manejo de Reconexión

Al reconectar, el cliente solicita alertas perdidas vía REST:
```
GET /v1/alerts?since={last_received_at}&country=CO
```

Esto garantiza que no se pierdan alertas durante gaps de conectividad. Ver [`alerts-service.md §6`](./alerts-service.md).

---

## Referencias

- [alerts-service.md — Especificación completa](./alerts-service.md)
- [ADR-005 — Arquitectura Hexagonal](../propuesta-arquitectura-hurto-vehiculos.md#adr-005--arquitectura-hexagonal-para-servicios-sensibles-a-la-nube)
- Socket.IO Redis Adapter: https://socket.io/docs/v4/redis-adapter/
