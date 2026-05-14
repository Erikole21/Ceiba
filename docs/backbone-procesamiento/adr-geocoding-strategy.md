# ADR — Estrategia de Geocodificación Inversa en el Enrichment Service

**ID:** ADR-BP-01 (complementa ADR-008)  
**Estado:** Aprobado  
**Fecha:** 2026-05-13  
**Componente:** backbone-procesamiento → Enrichment Service  
**Autores:** Erik Rodríguez

---

## 1. Contexto

El Enrichment Service recibe eventos desde el tópico `vehicle.events.deduped`. Cada evento contiene coordenadas GPS (`lat`, `lon`) obtenidas por el agente de borde. El sistema debe traducir esas coordenadas a una dirección legible (`street`, `city`, `country`) para:

1. Mostrar la ubicación al oficial de policía en la UI de alertas.
2. Permitir búsquedas por zona geográfica en el `search-service`.
3. Enriquecer los registros de ClickHouse para analítica por ciudad/zona.

La geocodificación inversa puede realizarse de dos formas: **síncrona** (el Enrichment Service espera la respuesta antes de producir el evento enriquecido) o **asíncrona** (el Enrichment Service produce el evento con los campos de dirección pendientes y una segunda etapa los completa en un ciclo posterior).

---

## 2. Criterios de Evaluación

| Criterio | Peso | Descripción |
|---|---|---|
| **Latencia del hot path** | ALTA | La geocodificación no debe superar el presupuesto de 300 ms asignado al Enrichment Service en el SLO p95 < 2 s. |
| **Consistencia del payload** | ALTA | El Matcher Service necesita `country_code` verificado y la ubicación legible para el routing del Alert Service. Una geocodificación incompleta en el payload del hot path reduce la calidad operacional. |
| **Resiliencia ante fallo del servicio de geocodificación** | ALTA | La geocodificación no puede bloquear el pipeline. Un evento con `geocoding_status = "FAILED"` debe avanzar. |
| **Simplicidad de la topología** | MEDIA | Una segunda etapa asíncrona implica un nuevo tópico intermedio, un consumidor adicional y lógica de merge de estados. |
| **Costo operacional** | MEDIA | El dataset local (OpenStreetMap Nominatim offline o GeoNames) tiene un costo de almacenamiento (~3–10 GB por región) y una cadencia de actualización (mensual). Una API externa tiene costo por consulta y dependencia de internet. |
| **Privacidad** | MEDIA | Enviar coordenadas a una API externa implica exponer las ubicaciones de los avistamientos a un tercero. El dataset local elimina esta exposición. |

---

## 3. Opciones Evaluadas

### Opción A: Geocodificación síncrona con API externa (p. ej. Google Maps, HERE, Nominatim cloud)

**Descripción:** El Enrichment Service llama a una API REST de geocodificación inversa en tiempo de procesamiento del evento, espera la respuesta y agrega los campos al payload.

**Ventajas:**
- Sin costo de almacenamiento local.
- Cobertura global inmediata.

**Desventajas:**
- Dependencia de conectividad con el proveedor externo; un fallo de red bloquea o degrada el pipeline.
- Latencia variable (~50–400 ms en p95 según proveedor y red); difícil garantizar el presupuesto de 300 ms.
- Costo por consulta acumulable a millones de eventos/día.
- Expone coordenadas de avistamientos a un proveedor externo (implicaciones de privacidad y soberanía de datos).
- Lock-in con el proveedor si no hay abstracción hexagonal.

**Veredicto:** Descartada. Latencia impredecible y dependencia externa inaceptable para el SLO de seguridad pública.

---

### Opción B: Geocodificación asíncrona en dos etapas

**Descripción:** El Enrichment Service produce el evento con `geocoding_status = "PENDING"` y coordenadas sin resolver. Un servicio independiente (`geocoding-worker`) consume el tópico `vehicle.events.deduped`, resuelve las coordenadas y publica a un tópico intermedio. El Enrichment Service (segunda instancia) hace un join y produce el evento enriquecido final.

**Ventajas:**
- Desacopla la geocodificación del hot path principal.
- El Enrichment Service original no se bloquea.

**Desventajas:**
- Introduce latencia adicional de varias decenas de ms hasta varios segundos dependiendo del lag del `geocoding-worker`.
- Complejidad: un tópico adicional, lógica de join (KTable o join de streams con ventana), y la posibilidad de que el Matcher Service reciba eventos con `geocoding_status = "PENDING"`.
- El Alert Service no tendría la dirección legible en tiempo real, lo que degrada la experiencia del oficial.
- Mayor complejidad en la topología Kafka Streams.

**Veredicto:** Descartada. La latencia adicional y la complejidad de la topología no están justificadas cuando existe un dataset local de alta disponibilidad.

---

### Opción C: Geocodificación síncrona con dataset local + timeout estricto ✅ SELECCIONADA

**Descripción:** El Enrichment Service realiza la geocodificación inversa llamando a un proceso local (sidecar o biblioteca embebida) que resuelve las coordenadas contra un dataset geoespacial local (OpenStreetMap Nominatim offline o equivalente). Se aplica un timeout estricto de **200 ms**. Si el timeout se supera o el dataset no puede resolver las coordenadas:
- El evento avanza con `geocoding_status = "FAILED"` o `"INVALID_COORDINATES"`.
- Los campos `street`, `city` y `country` quedan en blanco.
- Se registra la métrica `enrichment_geocoding_failures_total`.

**Dataset local propuesto:** Nominatim offline con datos de OpenStreetMap por país (región América Latina, ~2 GB por continente). Actualización mensual vía proceso de mantenimiento programado. Como alternativa ligera: biblioteca `reverse-geocoder` en Python o `go-geocoder` en Go con dataset GeoNames (~100 MB global a nivel de ciudad).

**Ventajas:**
- Latencia predecible: p95 < 10 ms en dataset local para búsquedas por coordenada (lookup geoespacial por cuadrícula/tile).
- Sin dependencia de red externa; alta disponibilidad inherente.
- Sin exposición de coordenadas a terceros (privacidad y soberanía de datos).
- Sin costo por consulta.
- Simplicidad de la topología: no requiere tópicos intermedios adicionales.
- El timeout de 200 ms garantiza que el Enrichment Service no excede su presupuesto de 300 ms incluso en el peor caso.

**Desventajas:**
- Dataset local puede quedar desactualizado entre actualizaciones mensuales (nuevas calles, cambios de nombre).
- Costo de almacenamiento (~2–10 GB por dataset) y proceso de actualización programada.
- Implementación inicial del pipeline de actualización del dataset.

**Mitigación de desventajas:**
- Para el caso de uso (coordenadas de vehículos en vías de circulación), la granularidad a nivel de ciudad/barrio es suficiente; las calles nuevas tienen impacto mínimo en las primeras semanas.
- El proceso de actualización del dataset es un Kubernetes CronJob que descarga, valida e instala el nuevo dataset sin reinicio del Enrichment Service (hot reload).

---

## 4. Decisión

**Se adopta la Opción C: geocodificación síncrona con dataset local y timeout estricto de 200 ms.**

El Enrichment Service incluye un puerto hexagonal `GeocodingPort` con la siguiente interfaz:

```go
// GeocodingPort define el contrato de geocodificación inversa.
type GeocodingPort interface {
    // ReverseGeocode traduce coordenadas a dirección.
    // Si excede el timeout o falla, retorna un GeocodingResult con Status = "FAILED".
    ReverseGeocode(ctx context.Context, lat, lon float64) (GeocodingResult, error)
}

type GeocodingResult struct {
    Street  string
    City    string
    Country string
    Status  string // "OK" | "FAILED" | "INVALID_COORDINATES"
}
```

**Implementaciones del puerto:**
- `LocalNominatimAdapter` (producción): llama al sidecar Nominatim vía HTTP con timeout de 200 ms.
- `GeoNamesAdapter` (alternativa ligera): biblioteca Go embebida con dataset GeoNames en memoria.
- `MockGeocodingAdapter` (tests): retorna resultados predefinidos sin red.

---

## 5. Consecuencias

### Positivas
- El presupuesto de 300 ms del Enrichment Service es alcanzable con margen.
- El pipeline es resiliente a fallos de geocodificación: los eventos con `geocoding_status = "FAILED"` avanzan sin bloquear la cadena (CR-02).
- No hay dependencia de red externa en el hot path.
- El puerto hexagonal `GeocodingPort` permite reemplazar el adapter en el futuro (ej. API externa si se decide enviar los datos de coordenadas a un proveedor con un acuerdo de DPA).

### Negativas / Riesgos
- El sidecar Nominatim requiere recursos adicionales por pod del Enrichment Service (~1 GB RAM para el dataset en memoria).
- El proceso de actualización mensual del dataset debe probarse en staging antes de aplicarse en producción.
- En zonas rurales o de cobertura incompleta del dataset, la tasa de `geocoding_status = "FAILED"` puede ser mayor. Se recomienda monitorear la métrica `enrichment_geocoding_failures_total` por país.

### Alerta operacional asociada
Si `enrichment_geocoding_failures_total` supera el **5 % de los eventos procesados** en una ventana de 5 minutos, se activa la alerta `enrichment_geocoding_failure_rate_high`. Ver [slo-observability.md](./slo-observability.md).

---

## 6. Relación con ADR-008

ADR-008 establece la separación hot path / cold path y el patrón CQRS pragmático. Esta decisión complementa ADR-008 resolviendo un punto de extensión explícito: el enrichment del evento antes de su proyección a los stores de lectura. La geocodificación síncrona mantiene la simplicidad de la topología de Kafka Streams prevista en ADR-008 y no introduce nuevos tópicos intermedios fuera del modelo definido.

---

## 7. Revisión

Esta decisión se revisará si:
- La tasa de `geocoding_status = "FAILED"` supera el 10 % de forma sostenida en producción.
- El SLO p95 < 2 s se ve comprometido por la geocodificación en condiciones de carga extrema.
- Un requisito de cobertura geográfica fuera de América Latina (ej. Europa, Asia) hace inviable el mantenimiento del dataset local.
