# ADR — Decisión del Mecanismo de Bridge MQTT→Kafka

**ID:** ADR-bridge-001 (complementa ADR-002 de la propuesta principal)  
**Estado:** Aceptado  
**Fecha:** 2026-05-13  
**Autores:** Equipo de Plataforma  
**Referencia:** [Visión General](./overview.md) · [Bridge operacional](./bridge.md) · [ADR-002](../propuesta-arquitectura-hurto-vehiculos.md#adr-002--mqtt-5-como-protocolo-principal-de-telemetría)

---

## Contexto

La arquitectura de ingestión requiere mover eventos publicados por los dispositivos en el clúster EMQX 5.x hacia el topic Kafka `vehicle.events.raw`, manteniendo el orden por `device_id` como garantía crítica para el procesamiento downstream (CA-06).

El requisito de orden por dispositivo implica que todos los mensajes de un mismo `device_id` deben llegar a la misma partición Kafka. Esto se puede lograr configurando la partition key como `device_id` (== `client_id` MQTT == CN del certificado).

Se evaluaron dos opciones principales. Ambas son técnicamente viables; la elección se basa en criterios operacionales, de latencia y de mantenibilidad.

---

## Opciones Evaluadas

### Opción A: Kafka Connect + EMQX Rule Engine con EMQX Kafka Data Bridge (nativo)

EMQX 5.x incluye un Data Bridge nativo que permite, mediante el Rule Engine, seleccionar mensajes con SQL y publicarlos directamente en un topic Kafka sin ningún servicio intermedio. El Rule Engine actúa como capa de filtrado y transformación; el Data Bridge maneja la conexión al cluster Kafka con SASL/TLS.

**Características:**
- Configuración declarativa en la UI/API de EMQX o en archivos HOCON.
- Sin proceso adicional desplegado: el bridge corre dentro del nodo EMQX.
- Soporte nativo de partition key configurable (`${clientid}`).
- Buffer interno configurable para absorber picos de publicación cuando Kafka tiene latencia.
- Métricas de throughput, lag y errores expuestas vía API de EMQX y exportables a Prometheus.
- Backpressure: EMQX puede acumular mensajes QoS 1 en sesión si Kafka no confirma; configurable con `max_inflight` y `buffer_mode`.

### Opción B: Servicio Go Custom de Bridge

Un microservicio Go independiente que se suscribe a los topics relevantes del clúster EMQX vía MQTT y publica en Kafka usando el cliente Sarama o confluent-kafka-go. El servicio implementa lógica de particionamiento, manejo de errores y reintentos.

**Características:**
- Control total sobre lógica de filtrado, transformación y enrutamiento.
- Permite transformaciones complejas o enriquecimiento ligero antes de Kafka.
- Requiere despliegue, monitoreo, escalado y mantenimiento adicionales.
- Introduce una sesión MQTT adicional que consume recursos del broker.
- La lógica de gestión de errores y reintentos es responsabilidad del equipo.
- Latencia adicional por el ciclo MQTT→proceso→Kafka.

---

## Matriz de Criterios

| Criterio | Peso | Opción A (EMQX nativo) | Opción B (Go custom) |
|---|---|---|---|
| **Latencia P95** | Alto | Bajo (in-process, sin salto de red adicional) | Medio (salto MQTT→proceso→Kafka) |
| **Orden garantizado por device_id** | Alto | Sí (partition key configurable `${clientid}`) | Sí (implementable, requiere código) |
| **Mantenimiento de código** | Alto | Ninguno (configuración HOCON/YAML) | Alto (código Go + CI/CD + tests) |
| **Operación en producción** | Alto | Bajo (monitoreo integrado en EMQX dashboard) | Medio-Alto (proceso separado, propio observability) |
| **Backpressure ante Kafka caído** | Alto | Integrado (buffer + sesión EMQX) | Requiere implementación propia |
| **Costo de infraestructura** | Medio | Sin pods adicionales | 2–3 pods adicionales (HA) |
| **Flexibilidad de transformación** | Medio | Limitada (SQL simple en Rule Engine) | Alta (código arbitrario) |
| **Alineación con ADR-004 (K8s + OSS)** | Medio | Sí (configmap, sin binario custom) | Sí (contenedor K8s) |
| **Riesgo operacional inicial** | Alto | Bajo (probado en producción por comunidad EMQX) | Medio (código propio, curva de bugs) |
| **Consistencia de criterio ADR-005** | Alto | No aplica (no hay abstracción cloud aquí) | No aplica |

**Puntuación ponderada:** Opción A supera a Opción B en todos los criterios de alto peso.

---

## Decisión

**Se adopta la Opción A: EMQX Rule Engine con EMQX 5.x Kafka Data Bridge nativo.**

El Rule Engine de EMQX evalúa reglas SQL sobre los mensajes publicados y los enruta al Data Bridge configurado. El Data Bridge gestiona la conexión al cluster Kafka con SASL/TLS, la partition key `${clientid}` y la confirmación `acks=all`.

No se despliega ningún microservicio Go adicional para el bridge. Todo el comportamiento de bridge se declara en la configuración de EMQX.

---

## Justificación

1. **Latencia:** el bridge in-process de EMQX evita el ciclo de red MQTT→proceso Go→Kafka. En condiciones nominales esto reduce la latencia de bridge en ~5–15 ms por mensaje, relevante para el SLO hot-path de p95 < 2 s desde captura hasta alerta.

2. **Cero código adicional:** la configuración del Rule Engine es declarativa (SQL + HOCON). No se introduce código Go que requiera mantenimiento, tests de integración ni on-call separado.

3. **Backpressure nativo:** cuando Kafka tiene latencia o cae temporalmente (CR-06), EMQX acumula los mensajes QoS 1 en el buffer del Data Bridge y/o en la sesión MQTT del broker hasta que Kafka se recupere. No se requiere lógica custom de reintento.

4. **Operación integrada:** las métricas del Data Bridge (throughput, lag, errores de Kafka) se exponen junto con las métricas del broker EMQX en el mismo endpoint Prometheus/HTTP, simplificando el observability.

5. **Orden garantizado:** la partition key `${clientid}` (igual al `device_id`) asegura que todos los mensajes de un dispositivo van a la misma partición Kafka, cumpliendo CA-06 sin código adicional.

---

## Consecuencias

**Positivas:**
- Reducción de la superficie operacional: no hay pods adicionales del bridge que escalar, monitorear o actualizar.
- El canal MQTT→Kafka queda completamente definido en la configuración de EMQX, versionada en el repositorio (GitOps).
- El equipo de plataforma no necesita mantener código Go para el bridge.

**Negativas / Limitaciones:**
- La lógica de transformación está limitada al SQL del Rule Engine de EMQX. Si en el futuro se requiere enriquecimiento complejo antes de Kafka, se deberá evaluar si el Rule Engine alcanza o si se introduce un processor Kafka Streams downstream.
- El equipo debe tener familiaridad con la configuración HOCON de EMQX para operar el Rule Engine. Esta curva de aprendizaje es menor que mantener código custom pero no es cero.
- Si se migra a un broker MQTT distinto a EMQX (hipótesis muy poco probable dado ADR-002), el Rule Engine y el Data Bridge no son portables. Sin embargo, el contrato Kafka (`vehicle.events.raw`, partition key, serialización JSON) sí es portable.

**Criterios de revisión:** si el volumen de transformación requerida crece significativamente o si la capacidad del Rule Engine resulta insuficiente en producción, se revisará esta decisión en el contexto de una iteración dedicada a la evolución del pipeline de ingestión.

---

## Referencias

- [bridge.md](./bridge.md) — Especificación operacional detallada del bridge con esta decisión aplicada
- [emqx-cluster.md §5](./emqx-cluster.md) — Configuración del Kafka Data Bridge en el Rule Engine
- [operations.md §4](./operations.md) — Monitoreo del lag del bridge y alertas
- [ADR-002](../propuesta-arquitectura-hurto-vehiculos.md#adr-002--mqtt-5-como-protocolo-principal-de-telemetría) — Decisión de EMQX como broker MQTT
- [ADR-004](../propuesta-arquitectura-hurto-vehiculos.md#adr-004--kubernetes-como-plataforma-de-cómputo) — Kubernetes como plataforma de cómputo
