# ADR-CH-01 · Canal de Ingestión ClickHouse

**Estado:** Aceptado  
**Fecha:** 2026-05-13  
**Contexto:** Almacenamiento de lectura — capa ClickHouse  
**Referencia:** ADR-008 (CQRS pragmático) · [clickhouse-schema.md](./clickhouse-schema.md) · [debezium-cdc.md](./debezium-cdc.md)

---

## 1. Contexto

ClickHouse actúa como el almacén analítico del sistema: agrega millones de eventos por zona, hora y país para alimentar dashboards en Superset y mapas de calor H3. Necesita recibir los eventos enriquecidos de forma confiable, con un lag aceptable (< 5 s p99) y sin pérdida de datos.

Existen dos canales candidatos para llevar los eventos desde la capa de procesamiento hacia ClickHouse:

1. **Debezium CDC sobre WAL de PostgreSQL** (vía tópico Kafka `pg.events.cdc`)
2. **Consumidor directo del tópico Kafka `vehicle.events.enriched`** (producido por el Enrichment Service)

---

## 2. Alternativas Evaluadas

### Alternativa A — Debezium CDC sobre WAL PostgreSQL

En este flujo, el `Enrichment Service` escribe en PostgreSQL. Debezium captura los cambios del WAL (Write-Ahead Log) y produce el tópico `pg.events.cdc`. ClickHouse consume ese tópico a través de una Kafka Engine Table y una vista materializada.

```
Enrichment Service
      │
      ▼
  PostgreSQL (WAL)
      │ WAL stream
      ▼
  Debezium Connector
      │
      ▼
  Kafka: pg.events.cdc
      │
      ▼
  ClickHouse (Kafka Engine Table → MergeTree)
```

**Ventajas:**
- **PostgreSQL como fuente de verdad única:** ClickHouse refleja exactamente lo que está en PostgreSQL. Si hay un rollback o corrección en PostgreSQL, el CDC lo captura automáticamente (evento `delete` o `update`).
- **Desacoplamiento:** El `Enrichment Service` no necesita conocer la existencia de ClickHouse; escribe solo en PostgreSQL.
- **Soporte nativo de replay:** Si ClickHouse pierde datos (fallo, borrado accidental), se puede re-procesar desde una LSN específica del WAL, sin necesidad de re-procesar el pipeline de enriquecimiento.
- **Consistencia entre proyecciones:** La misma fila que aparece en PostgreSQL aparecerá en ClickHouse, garantizando que ambas proyecciones son coherentes.
- **Transformaciones en CDC:** Debezium soporta Single Message Transforms (SMT) para normalizar el payload antes de enviarlo a Kafka.

**Desventajas:**
- Lag adicional: PostgreSQL → Debezium → Kafka → ClickHouse agrega latencia comparado con una escritura directa. Estimado: 2–5 s en condiciones normales.
- Debezium es una pieza operacional adicional (JVM, Kafka Connect cluster).
- El payload CDC incluye campos de metadatos (`before`, `after`, `source.lsn`) que deben transformarse antes de insertar en ClickHouse.

### Alternativa B — Consumidor directo de `vehicle.events.enriched`

En este flujo, ClickHouse implementa su propia Kafka Engine Table sobre el tópico `vehicle.events.enriched` que ya produce el `Enrichment Service`.

```
Enrichment Service
      ├──► PostgreSQL
      ├──► OpenSearch
      └──► Kafka: vehicle.events.enriched
                      │
                      ▼
              ClickHouse (Kafka Engine Table → MergeTree)
```

**Ventajas:**
- Menor latencia: el evento llega a ClickHouse casi simultáneamente con PostgreSQL (< 1 s adicional).
- No requiere Debezium ni Kafka Connect; ClickHouse gestiona el consumo directamente.
- Payload más limpio: el evento enriquecido ya tiene el formato definitivo.

**Desventajas:**
- **Divergencia de proyecciones:** Si un evento falla al escribirse en PostgreSQL pero ya fue publicado en Kafka, ClickHouse tendría datos que PostgreSQL no tiene. La gestión de transaccionalidad entre escritura a PostgreSQL y publicación a Kafka requiere el patrón Outbox o transacciones Kafka.
- **Sin soporte nativo de correcciones:** Un update o delete en PostgreSQL (corrección de coordenadas, ajuste de confidence) no se refleja automáticamente en ClickHouse.
- **Acoplamiento implícito:** El `Enrichment Service` debe garantizar que el esquema de `vehicle.events.enriched` sea consumible por ClickHouse sin transformaciones intermedias.
- **Replay complejo:** Para reconstruir ClickHouse desde cero, habría que re-publicar todos los eventos desde el `Enrichment Service`, lo que requiere coordinar con el pipeline de procesamiento.

---

## 3. Decisión

**Se adopta Alternativa A: Debezium CDC como canal primario de ingestión en ClickHouse.**

PostgreSQL es la fuente de verdad de los eventos operacionales. ClickHouse debe reflejar esa fuente de verdad, no una copia independiente. La capacidad de replay desde LSN específica del WAL y la consistencia automática entre proyecciones justifican el overhead operacional de Debezium.

La latencia adicional de 2–5 s es aceptable para el caso de uso analítico (dashboards, heatmaps), donde el SLO es p95 < 3 s para queries pero no hay SLO de ingesta en tiempo real.

---

## 4. Justificación Técnica

El flujo CDC completo con Debezium garantiza que:

1. Si el `Enrichment Service` hace rollback de una transacción en PostgreSQL, ese evento **nunca** llega a ClickHouse.
2. Si la coordenada de un evento se corrige en PostgreSQL (operación `UPDATE`), Debezium captura el evento `u` (update) con el `before` y el `after`, y ClickHouse puede aplicar la corrección mediante `ReplacingMergeTree` o una lógica de deduplicación basada en `(event_id, _version)`.
3. El payload CDC incluye `source.lsn` que permite auditar exactamente qué versión del WAL está representada en ClickHouse.

Detalle de configuración del conector (ver [debezium-cdc.md](./debezium-cdc.md)):
```json
{
  "plugin.name": "pgoutput",
  "slot.name": "debezium_slot_events",
  "publication.name": "debezium_pub_events",
  "table.include.list": "public.vehicle_events",
  "topic.prefix": "pg"
}
```

---

## 5. Consecuencias

### Positivas

- ClickHouse es un espejo fiel de PostgreSQL para la tabla `vehicle_events`.
- El replay de ClickHouse es posible sin intervención del pipeline de procesamiento.
- Las correcciones y borrados en PostgreSQL se propagan automáticamente.

### Negativas / Riesgos

- Debezium requiere su propio Kafka Connect cluster (o se despliega como conector en el cluster existente). Ver [helm/README.md](./helm/README.md) para el chart Helm.
- El slot de replicación (`debezium_slot_events`) debe monitorizarse: si Debezium se cae durante un tiempo prolongado, el slot retiene el WAL y puede agotar el disco del primario PostgreSQL. Alerta operacional configurada en [slo-observability.md](./slo-observability.md).
- El lag de 2–5 s es inherente al diseño; no es reducible sin cambiar a Alternativa B para casos de uso de tiempo real (el hot path no usa ClickHouse, usa Redis).

### Canal secundario (Alternativa B como fallback)

Si en el futuro se requiere un canal de menor latencia para un tipo de evento específico (ej. alertas de vehículos hurtados para conteos en tiempo real), se puede activar el consumo directo de `vehicle.events.enriched` para ese subset, manteniendo CDC para el grueso de los eventos. Esta combinación es soportada por ClickHouse con múltiples Kafka Engine Tables sobre el mismo topic.

---

## 6. Plan de Evolución

| Fase | Criterio | Acción |
|---|---|---|
| **Fase 1** | CDC estable, lag < 5 s | Solo CDC Debezium como canal primario |
| **Fase 2** | Lag CDC > 10 s en horas pico | Evaluar Kafka Engine Table directa en `vehicle.events.enriched` como canal paralelo para dashboards de tiempo real |
| **Fase 3** | Multi-región | Replicar el cluster Kafka entre regiones; ClickHouse regional consume el tópico local |

---

## 7. Referencias

- [debezium-cdc.md](./debezium-cdc.md) — Configuración detallada del conector Debezium.
- [clickhouse-schema.md](./clickhouse-schema.md) — Kafka Engine Table y vista materializada.
- [slo-observability.md](./slo-observability.md) — Alerta de lag CDC y tamaño de slot WAL.
- ADR-008 en [propuesta-arquitectura-hurto-vehiculos.md](../propuesta-arquitectura-hurto-vehiculos.md).
