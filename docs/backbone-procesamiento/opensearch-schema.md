# Backbone de Procesamiento — OpenSearch Schema (Perspectiva de Indexación)

**Componente:** backbone-procesamiento — indexación en OpenSearch  
**Versión del documento:** 1.0

> **Nota de coordinación:** Este documento describe la perspectiva de indexación del `backbone-procesamiento` en OpenSearch. La **fuente de verdad del mapping, los aliases, la política ILM y las queries de ejemplo** está en:
>
> **[docs/almacenamiento-lectura/opensearch-schema.md](../almacenamiento-lectura/opensearch-schema.md)**  
> **[docs/almacenamiento-lectura/opensearch-ilm.md](../almacenamiento-lectura/opensearch-ilm.md)**
>
> Todo cambio de mapping debe realizarse en esos documentos y luego reflejarse aquí. No duplicar el JSON de mapping en este archivo.

---

## 1. Responsabilidades de Indexación del Backbone

El `Enrichment Service` indexa cada evento enriquecido en OpenSearch en paralelo a la escritura en PostgreSQL. No hay un servicio dedicado de indexación — el propio Enrichment Service actúa como productor de OpenSearch.

| Servicio | Índice destino | Operación |
|---|---|---|
| **Enrichment Service** | `vehicle-events-{country_code}-{YYYY-MM}` | INDEX (documento por evento) |

---

## 2. Nombre del Índice y Routing

El nombre del índice sigue el patrón:

```
vehicle-events-{country_code_lowercase}-{YYYY-MM}
```

Ejemplos: `vehicle-events-co-2026-05`, `vehicle-events-mx-2026-05`.

El `Enrichment Service` calcula el nombre del índice en tiempo de ejecución a partir del `country_code` y el mes del `event_ts`:

```typescript
// Cálculo del nombre del índice en el Enrichment Service
function getIndexName(countryCode: string, eventTs: Date): string {
  const year  = eventTs.getUTCFullYear();
  const month = String(eventTs.getUTCMonth() + 1).padStart(2, '0');
  return `vehicle-events-${countryCode.toLowerCase()}-${year}-${month}`;
}
```

---

## 3. Documento Indexado

El documento que el `Enrichment Service` envía a OpenSearch es un subconjunto del evento enriquecido. Los campos de imagen no se indexan (solo se almacenan en `_source`):

```json
{
  "event_id":           "550e8400-e29b-41d4-a716-446655440000",
  "country_code":       "CO",
  "device_id":          "dev-co-001",
  "plate_raw":          "ABC-123",
  "plate_normalized":   "ABC123",
  "event_ts":           "2026-05-13T14:30:00.000Z",
  "received_ts":        "2026-05-13T14:30:01.234Z",
  "confidence":         97.50,
  "location": {
    "lat": 4.7109,
    "lon": -74.0721
  },
  "image_uri":          "s3://antihurto-co/events/2026/05/13/550e8400.jpg",
  "thumbnail_uri":      "s3://antihurto-co/events/2026/05/13/550e8400_thumb.jpg",
  "clock_uncertain":    false,
  "image_unavailable":  false,
  "enrichment_version": 1
}
```

> El campo `location` usa el formato `{"lat": ..., "lon": ...}` que OpenSearch mapea automáticamente al tipo `geo_point` definido en la plantilla de índice.

---

## 4. Operación de Indexación

```typescript
// Pseudocódigo del Enrichment Service — indexación en OpenSearch
async function indexEvent(event: EnrichedEvent): Promise<void> {
  const indexName = getIndexName(event.countryCode, event.eventTs);

  await this.openSearchClient.index({
    index: indexName,
    id:    event.eventId,  // El event_id como ID del documento garantiza idempotencia
    body:  {
      event_id:           event.eventId,
      country_code:       event.countryCode,
      device_id:          event.deviceId,
      plate_raw:          event.plateRaw,
      plate_normalized:   event.plateNormalized,
      event_ts:           event.eventTs.toISOString(),
      received_ts:        new Date().toISOString(),
      confidence:         event.confidence,
      location:           { lat: event.lat, lon: event.lon },
      image_uri:          event.imageUri ?? null,
      thumbnail_uri:      event.thumbnailUri ?? null,
      clock_uncertain:    event.clockUncertain,
      image_unavailable:  event.imageUnavailable,
      enrichment_version: event.enrichmentVersion,
    },
    // refresh: 'false' — no esperar refresh; el documento tardará hasta 5 s en ser visible
  });
}
```

---

## 5. Manejo de Errores de Indexación

El `Enrichment Service` trata la indexación en OpenSearch como una operación **best-effort** con reintentos:

- **Fallo transitorio (timeout, 503):** Reintentar hasta 3 veces con backoff exponencial (500 ms, 1 s, 2 s).
- **Fallo persistente:** El evento sigue siendo procesado (se escribe en PostgreSQL). El fallo de OpenSearch no bloquea el pipeline principal. Se registra una métrica `opensearch_index_errors_total` y se envía una alerta si la tasa supera 0.1 %.
- **Recuperación:** Si un lote de eventos no se indexó, se puede re-indexar desde el tópico Kafka `vehicle.events.enriched` (siempre que el tópico tenga retención suficiente) o desde PostgreSQL usando un proceso de backfill.

---

## 6. Coordinación con Almacenamiento de Lectura

- **Consistencia:** PostgreSQL es la fuente de verdad. Si hay divergencia entre OpenSearch y PostgreSQL (evento en PostgreSQL pero no en OpenSearch), el `search-service` puede complementar la búsqueda con una query directa a PostgreSQL usando el índice `idx_ve_plate_country`.
- **Creación de índices:** Los índices nuevos se crean automáticamente por la plantilla `vehicle-events-template` (aprovisionada por Terraform). El `Enrichment Service` no necesita crear índices manualmente.
- **ILM:** Los índices son gestionados por la política ILM `vehicle-events-lifecycle`. El Enrichment Service no interviene en el ciclo de vida.

Ver el mapping completo, aliases, política ILM y queries de ejemplo en:
- [docs/almacenamiento-lectura/opensearch-schema.md](../almacenamiento-lectura/opensearch-schema.md)
- [docs/almacenamiento-lectura/opensearch-ilm.md](../almacenamiento-lectura/opensearch-ilm.md)
