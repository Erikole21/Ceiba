# ADRs Locales del Agente de Borde

**Tipo:** Decisiones de Diseño Internas (Architecture Decision Records)  
**Alcance:** Decisiones específicas del agente de borde que complementan los ADRs de la propuesta principal  
**Referencia arquitectural:** [Visión General](./overview.md) · [Propuesta §4 — Decisiones de Diseño](../propuesta-arquitectura-hurto-vehiculos.md#4-decisiones-de-diseño)

---

## Introducción

Este documento registra las decisiones de diseño **internas** del agente de borde que no estaban completamente especificadas en los ADRs de la propuesta principal. Cada ADR local referencia y complementa el ADR de alto nivel correspondiente, añadiendo los parámetros concretos y las alternativas evaluadas a nivel de implementación.

---

## ADR-L-001 — SQLite WAL: `synchronous=NORMAL` vs `synchronous=FULL`

**Complementa:** [ADR-001 de la propuesta](../propuesta-arquitectura-hurto-vehiculos.md#adr-001--patrón-edge-first-con-store-and-forward)  
**Fecha de decisión:** 2024-05-07  
**Estado:** Aceptado

### Contexto

El Queue Manager usa SQLite en modo WAL para persistir eventos. El pragma `synchronous` controla la frecuencia de llamadas a `fsync` y tiene un impacto directo en el rendimiento de escritura y en la ventana de pérdida de datos ante cortes de energía.

Las opciones evaluadas fueron:

| Modo | Comportamiento | Latencia de escritura | Ventana de pérdida |
|---|---|---|---|
| `synchronous=FULL` | `fsync` en cada commit | ~5–15 ms por transacción | Prácticamente cero |
| `synchronous=NORMAL` | `fsync` solo en checkpoint WAL | ~0.5–2 ms por transacción | < 100 ms (ventana entre commits y checkpoint) |
| `synchronous=OFF` | Sin `fsync` | ~0.1 ms | Potencialmente grandes |

### Decisión

Se adopta `synchronous=NORMAL` en modo WAL.

### Razonamiento

1. **El `event_id` es idempotente.** En el caso extremo de que un evento se pierda en la ventana de < 100 ms (commit exitoso pero antes del checkpoint), el componente ANPR típicamente re-captura el vehículo en el siguiente ciclo de lectura, produciendo el mismo `event_id` (UUID v5 sobre `device_id + plate + monotonic_ns`). La pérdida es recuperable.

2. **El hardware de referencia es ARM embebido.** En dispositivos con almacenamiento flash eMMC o SD, las llamadas a `fsync` pueden tardar 5–50 ms dependiendo del controlador. Con `FULL`, a una tasa de 1 evento/s, el overhead sería de 5–50 ms por evento de forma sostenida, lo que podría saturar el pipeline de captura.

3. **El escenario más crítico no es el corte en mitad del commit.** Es el corte durante una ráfaga de eventos sin conectividad. En ese escenario, `NORMAL` pierde como máximo los eventos del último checkpoint pendiente, que es un período muy corto respecto a las horas de desconectividad que el sistema debe tolerar.

4. **El WAL en sí provee durabilidad suficiente.** Aunque `synchronous=NORMAL` no garantiza que el WAL esté en disco en cada commit, el SQLite WAL está diseñado para ser recuperable: la consistencia de la base de datos siempre se mantiene.

### Alternativas Descartadas

- **`synchronous=FULL`:** Latencia inaceptable en hardware embebido con almacenamiento lento. Penaliza la captura sostenida.
- **`synchronous=OFF`:** Inaceptable. La ventana de pérdida puede ser de segundos a minutos, incompatible con los requisitos de evidencia probatoria.

### Consecuencias

- Ventana teórica de pérdida de eventos: < 100 ms (prácticamente irrelevante operacionalmente).
- Se acepta el riesgo residual documentado.
- El cloud siempre deduplica por `event_id`; no hay problema con reenvíos si el evento se re-captura.

---

## ADR-L-002 — MQTT: Keep-Alive 60 s, Sesión Persistente, Tamaño Máximo de Payload

**Complementa:** [ADR-002 de la propuesta](../propuesta-arquitectura-hurto-vehiculos.md#adr-002--mqtt-5-como-protocolo-principal-de-telemetría)  
**Fecha de decisión:** 2024-05-07  
**Estado:** Aceptado

### Contexto

ADR-002 de la propuesta establece MQTT 5 con QoS 1, sesiones persistentes y keep-alive de 60 s. Este ADR local documenta el razonamiento detrás de cada valor concreto y las restricciones de tamaño de payload.

### Decisión 1: Keep-Alive de 60 Segundos

**Razonamiento:**

- En GSM/3G, mantener una conexión TCP activa consume un slot de radio incluso cuando no hay datos. Keep-alive demasiado frecuente (< 30 s) provoca paquetes `PINGREQ/PINGRESP` innecesarios que consumen cuota de datos GSM.
- Keep-alive demasiado largo (> 120 s) significa que una desconexión silenciosa (timeout de NAT, pérdida de señal sin TCP RST) no se detecta rápido, y el agente puede acumular mensajes creyendo que tiene sesión activa.
- 60 s es el estándar de facto en deployments IoT sobre GSM (compatibilidad con APNs privados y routers de operadores que tienen NAT timeout de 90 s típico).

**Configuración en el broker EMQX:** `zone.default.keepalive_backoff = 0.75`. Si el cliente no envía PINGREQ en `keepalive * 1.5 = 90 s`, el broker cierra la sesión.

### Decisión 2: Sesión Persistente (`CleanStart=false`)

**Razonamiento:**

- Con `CleanStart=false` (sesión persistente), el broker EMQX almacena los mensajes QoS 1 no entregados durante las desconexiones del dispositivo y los entrega al reconectar.
- Esto elimina la necesidad de que el agente repita `PUBLISH` de mensajes que ya llegaron al broker pero cuyo `PUBACK` se perdió por la desconexión. Sin sesión persistente, el agente tendría que re-publicar basándose en timeouts de PUBACK, lo que puede duplicar mensajes en el broker.
- Con sesión persistente, la deduplicación recae en el Deduplicator del cloud (por `event_id`), no en lógica compleja del agente.

**Consecuencia:** El broker debe tener capacidad de almacenamiento para las sesiones de todos los dispositivos. Para 100 K dispositivos con sesiones de hasta 4 h offline y ~1 K eventos/día, el almacenamiento por sesión es < 1 MB; el total es < 100 GB, manejable con EMQX cluster.

### Decisión 3: Tamaño Máximo de Payload MQTT

**Valores establecidos:**

| Payload | Tamaño máximo | Justificación |
|---|---|---|
| Evento de matrícula (`events`) | 2 KB | El payload JSON del `NormalizedEvent` nunca supera 1 KB en la práctica; 2 KB da margen para campos adicionales futuros |
| Health Beacon (`health`) | 512 bytes | El payload msgpack del Health Beacon es ~180–320 bytes; 512 bytes es el límite seguro |
| Config/OTA (`config`, `ota`) | 256 KB | El manifiesto OTA con URL larga y firma cosign puede alcanzar ~4 KB; 256 KB permite manifiestos extendidos sin cambio de protocolo |

**Configuración en el agente:**

```toml
[mqtt]
max_payload_bytes = 262144  # 256 KB (límite global por publicación)
```

**Razonamiento:** Los payloads de eventos y health son pequeños por diseño. El límite de 256 KB existe como guardia para el caso OTA. EMQX tiene por defecto un límite de 256 MB por mensaje; el límite del agente es mucho más conservador.

### Alternativas Descartadas

- **Keep-alive de 30 s:** Mayor consumo de datos GSM por PINGREQs frecuentes; sin beneficio operacional significativo.
- **`CleanStart=true`:** El agente debería re-publicar todos los mensajes sin PUBACK al reconectar, lo que duplica lógica y riesgo de mensajes duplicados.
- **Payload ilimitado:** Riesgo de acumulación de memoria en el broker ante dispositivos que envíen payloads anómalos; también puede saturar la cuota GSM si hay un bug en el serializador.

---

## ADR-L-003 — Bloom Filter: Parámetros Exactos y Delta vs Descarga Completa

**Complementa:** [ADR-009 de la propuesta](../propuesta-arquitectura-hurto-vehiculos.md#adr-009--bloom-filter-en-borde-para-hot-path-de-alertas)  
**Fecha de decisión:** 2024-05-07  
**Estado:** Aceptado

### Contexto

ADR-009 establece que el filtro tiene FP < 1% para 1 M placas. Este ADR local documenta los parámetros exactos (bit array ~1.14 MB, ~1.2 MB incluyendo overhead de estructura Go), la elección del algoritmo hash, y la decisión de usar delta vs descarga completa.

### Decisión 1: Parámetros Exactos del Bloom Filter

| Parámetro | Valor | Fórmula |
|---|---|---|
| n (capacidad) | 1 000 000 | Estimación conservadora máxima de placas hurtadas por país |
| fp (tasa de FP) | 0.01 (1%) | Objetivo del sistema |
| m (bits) | 9 585 059 (~1.14 MB) | `m = -n * ln(fp) / (ln 2)^2` |
| k (funciones hash) | 7 | `k = (m/n) * ln(2) ≈ 6.64` → redondea a 7 |
| Tamaño en disco | ~1.20 MB (cabecera 32 bytes + cuerpo + CRC32) | m/8 bytes + overhead |

**Razonamiento para n = 1 M:**

- La mayoría de países de LatAm tienen < 200 K vehículos hurtados activos en su lista (Colombia: ~150 K en 2023).
- Se usa 1 M como techo conservador para soportar países con listas más grandes sin rediseño.
- Si un país supera 1 M placas, el `m` aumentaría ~2x (a ~2.3 MB), aún dentro del presupuesto de memoria.

**Razonamiento para k = 7:**

- La fórmula da k = 6.64; redondear hacia arriba (7) reduce marginalmente la tasa de FP real por debajo del 1%.
- k = 7 implica 7 accesos al bit array por consulta. En hardware de 2 cores ARM, esto es < 1 μs por consulta.

### Decisión 2: Algoritmo Hash — MurmurHash3 con Doble Hashing

**Razonamiento:**

- MurmurHash3 (64-bit) es no-criptográfico, extremadamente rápido en ARM (< 10 ns por hash), y tiene excelente distribución para strings cortos como matrículas.
- La técnica de doble hashing de Kirsch-Mitzenmacher permite generar k posiciones con solo dos evaluaciones de hash, en lugar de k evaluaciones independientes. Esto reduce el costo de CPU por consulta de O(k) a O(2) evaluaciones hash + k multiplicaciones.
- No se usa SHA-256 ni ningún hash criptográfico: no es necesario para un Bloom filter; añadiría latencia innecesaria.

**Alternativas descartadas:**

- **xxHash:** Igualmente rápido, pero MurmurHash3 tiene más implementaciones auditadas en Go para Bloom filters.
- **FNV-1a:** Distribución inferior para strings cortos; mayor tasa de colisiones parciales.

### Decisión 3: Delta vs Descarga Completa

**Estrategia híbrida:**

- **Operación normal:** El cloud publica deltas incrementales en `countries/<cc>/bloom-filter/delta`. Un delta típico (10–50 placas nuevas) pesa < 2 KB.
- **Sincronización inicial (arranque sin filtro):** El agente se suscribe a `countries/<cc>/bloom-filter/full` y recibe el filtro completo comprimido (~1.25 MB con ZSTD).
- **Re-sincronización:** Si el agente recibe un delta con `base_generation` que no coincide con su generación actual (se perdió una actualización), se suscribe al topic `full` para resincronización.

**Razonamiento:**

- Los deltas son el mecanismo óptimo para el día a día: una placa nueva en la lista genera un delta de ~500 bytes vs 1.25 MB de la descarga completa. El ahorro es de 2500x en datos GSM para actualizaciones incrementales.
- La descarga completa solo ocurre: (a) al arrancar sin filtro local (o con filtro corrupto), y (b) cuando el agente se desconecta por más tiempo que la retención de deltas del cloud (configurable, default: 7 días). Esto minimiza los casos de descarga completa.
- La retención de deltas en el cloud durante 7 días garantiza que un dispositivo offline hasta 7 días puede recuperarse con deltas acumulados sin necesidad de descarga completa.

**Tiempo máximo de sincronización completa:** < 30 s sobre GSM 3G (análisis en [Bloom Filter §4.3](./bloom-filter.md#43-tiempo-máximo-de-sincronización-inicial)).

### Alternativas Descartadas

- **Solo descarga completa (sin deltas):** Innecesario. Descargar 1.25 MB cada vez que hay un cambio en la lista sería prohibitivo en términos de cuota GSM para listas que se actualizan múltiples veces al día.
- **CuckooFilter en lugar de BloomFilter:** Mayor eficiencia de espacio (~30% menos bits) y soporte de eliminación de elementos, pero mayor complejidad de implementación y sincronización. Para este caso de uso (listas de hurtados que casi nunca se reducen), el Bloom filter clásico es suficiente.
- **Lista completa en disco:** Posible técnicamente (< 10 MB para 1 M placas), pero la búsqueda es O(log n) vs O(1) del Bloom filter, y la sincronización incremental es más compleja.

---

## ADR-L-004 — `event_id` como UUID v5 (Idempotencia sin Coordinación)

**Complementa:** [ADR-001 de la propuesta](../propuesta-arquitectura-hurto-vehiculos.md#adr-001--patrón-edge-first-con-store-and-forward)  
**Fecha de decisión:** 2024-05-07  
**Estado:** Aceptado

### Contexto

El agente puede re-enviar el mismo evento múltiples veces ante fallos de red o reinicios del proceso. El cloud necesita deduplicar estos reenvíos. La clave de deduplicación debe ser generada por el agente sin coordinación con el cloud.

### Decisión

Usar UUID v5 (SHA-1 sobre namespace + nombre) como `event_id`, donde el nombre es `device_id + ":" + plate_normalizado + ":" + monotonic_ns`.

### Razonamiento

- **UUID v4 (aleatorio):** Si el agente se reinicia y re-procesa la misma lectura desde SQLite, el `event_id` ya está almacenado en la base. No hay problema. Pero si la lectura no se llegó a persistir (corte en mitad del commit), el re-procesamiento produciría un UUID v4 diferente → el cloud vería el mismo evento con dos IDs distintos → no puede deduplicar.
- **UUID v5 (determinista):** El mismo `(device_id, plate, monotonic_ns)` siempre produce el mismo UUID. Si el evento no se persistió, al re-procesarse (desde una nueva lectura del ANPR con el mismo `monotonic_ns` — caso prácticamente imposible al ser monotónico) se produciría el mismo `event_id`. Y si se persistió, se usa el `event_id` ya almacenado.
- El `monotonic_ns` como tercer componente garantiza que dos capturas de la misma placa por el mismo dispositivo en momentos distintos tengan `event_id` distintos.

### Alternativas Descartadas

- **UUID v4:** No determinista; no resuelve el caso de re-procesamiento sin persistencia previa.
- **Hash directo (SHA-256 del contenido del evento):** Equivalente al UUID v5 pero sin el estándar RFC 4122. Se prefiere UUID v5 por interoperabilidad con herramientas y bases de datos que validan el formato UUID.

---

## Referencias Cruzadas

| Documento | Relación |
|---|---|
| [Queue Manager](./queue-manager.md) | ADR-L-001: justificación de `synchronous=NORMAL` |
| [Uploader](./uploader.md) | ADR-L-002: keep-alive 60 s, sesión persistente, tamaño máximo de payload |
| [Bloom Filter](./bloom-filter.md) | ADR-L-003: parámetros exactos, delta vs descarga completa |
| [Collector](./collector.md) | ADR-L-004: algoritmo UUID v5 para `event_id` |
| [Propuesta §4](../propuesta-arquitectura-hurto-vehiculos.md#4-decisiones-de-diseño) | ADRs de alto nivel que estos ADRs locales complementan |
