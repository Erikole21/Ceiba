# ADR-PG-01 · Estrategia de Particionamiento PostgreSQL

**Estado:** Aceptado  
**Fecha:** 2026-05-13  
**Contexto:** Almacenamiento de lectura — capa PostgreSQL + PostGIS  
**Referencia:** ADR-011 (multi-tenant por `country_code`) · [postgresql-schema.md](./postgresql-schema.md)

---

## 1. Contexto

La tabla `vehicle_events` es el almacén operacional principal del sistema. Recibirá eventos de avistamiento de matrículas de todos los dispositivos desplegados, con un volumen estimado de:

- **100 K dispositivos** en fase LatAm: ~100 M eventos/mes (~1 000 placas/día/device).
- **1 M dispositivos** en escenario global: ~1 000 M eventos/mes.
- **Tamaño por fila:** ~1.2 KB incluyendo el punto PostGIS, metadatos y JSONB de extensiones.
- **Volumen estimado a 2 años (100 K devices):** ~2.4 TB sin compresión; ~1.0 TB con compresión de página PostgreSQL.

Sin particionamiento, las operaciones de mantenimiento (VACUUM, reindex, archivado) bloquearían la tabla completa. Además, el requisito ADR-011 exige aislamiento de datos por país.

Se evaluaron tres estrategias de particionamiento declarativas disponibles en PostgreSQL 16:

1. `PARTITION BY LIST (country_code)` — partición por discriminador de tenant.
2. `PARTITION BY RANGE (event_ts)` — partición por tiempo (mes o semana).
3. Particionamiento compuesto (LIST por país como primer nivel, RANGE por tiempo como segundo nivel).

---

## 2. Alternativas Evaluadas

### Alternativa A — PARTITION BY LIST (country_code) únicamente

**Ventajas:**
- Alineada directamente con ADR-011: cada país tiene su propia partición física.
- Las consultas con `WHERE country_code = 'CO'` eliminan todas las particiones de otros países (partition pruning).
- Permite asignar particiones de países con alta carga a tablespaces distintos (NVMe dedicado).
- Facilita el cumplimiento de soberanía de datos: en el futuro se puede mover la partición de un país a una instancia PostgreSQL separada en su región.
- Operaciones de mantenimiento (VACUUM, archivado) se pueden ejecutar por país sin afectar a los demás.

**Desventajas:**
- Sin partición por tiempo, las particiones de países activos crecen indefinidamente.
- El mantenimiento requiere herramientas adicionales (pg_partman) para sub-particionamiento temporal.

### Alternativa B — PARTITION BY RANGE (event_ts) únicamente

**Ventajas:**
- Facilita el archivado automático por antigüedad (drop de la partición del mes más antiguo).
- Consultas acotadas por fecha (`WHERE event_ts BETWEEN ...`) aprovechan partition pruning temporal.

**Desventajas:**
- No aísla datos por país: todos los países comparten la misma partición mensual.
- Las consultas `WHERE country_code = 'CO'` no se benefician de partition pruning.
- El aislamiento operacional por país requeriría filtros manuales, no aprovechando las capacidades declarativas de PostgreSQL.

### Alternativa C — Compuesto (LIST por país + RANGE por tiempo)

**Ventajas:**
- Aprovecha partition pruning en ambas dimensiones.
- Archivado por tiempo independiente por país.

**Desventajas:**
- PostgreSQL 16 soporta sub-particionamiento, pero el número de particiones hoja crece exponencialmente: N países × M meses = muchas particiones.
- Para 10 países y 24 meses de retención: 240 particiones hoja. Para 50 países: 1 200 particiones hoja. Impacto en el planning de queries y en el tiempo de conexión.
- pg_partman gestiona bien el tiempo pero combinar con LIST requiere scripts adicionales.
- Mayor complejidad operacional antes de validar el volumen real por país.

---

## 3. Decisión

**Se adopta Alternativa A: `PARTITION BY LIST (country_code)` como primer nivel.**

Dentro de cada partición por país, el archivado temporal se gestiona con `pg_partman` en modo **sub-partición opcional**, activable país por país cuando el volumen de esa partición supere el umbral de mantenimiento (500 GB o 6 meses de datos, lo que ocurra primero).

Esta decisión está alineada con **ADR-011**: el discriminador de tenant es `country_code` y es el filtro más frecuente en todas las consultas operacionales. El partition pruning por país ofrece el mayor beneficio inmediato con la menor complejidad operacional.

---

## 4. Justificación Técnica

Los patrones de acceso observados en el diseño del `search-service` y del `incident-service` incluyen:

```sql
-- Patrón 1: Búsqueda por matrícula + país (más frecuente)
WHERE country_code = 'CO' AND plate_normalized = 'ABC123'

-- Patrón 2: Trayectoria de vehículo + país
WHERE country_code = 'CO' AND plate_normalized = 'ABC123'
  AND event_ts BETWEEN '2026-01-01' AND '2026-03-31'

-- Patrón 3: Consulta geoespacial + país
WHERE country_code = 'CO'
  AND ST_DWithin(location, ST_MakePoint(-74.0, 4.7)::geography, 500)
```

En todos los patrones, `country_code` aparece como primer predicado. Con particionamiento LIST, PostgreSQL elimina en el planning todas las particiones que no corresponden al país consultado, reduciendo el espacio de búsqueda de forma significativa antes de evaluar índices secundarios.

---

## 5. Consecuencias

### Positivas

- Las consultas de `search-service` reducen el espacio de búsqueda al tamaño de la partición del país, no al total de la tabla.
- El mantenimiento (VACUUM, REINDEX) se puede planificar por país durante ventanas de baja carga local, sin afectar a países en horario de actividad.
- La soberanía de datos es operacionalmente ejecutable: detach de la partición de un país y re-attach en una instancia separada en su región.
- Los índices locales a cada partición son más pequeños y se reconstruyen más rápido.

### Negativas / Riesgos

- Las consultas sin `country_code` en el predicado hacen un sequential scan de todas las particiones. Se mitiga con una política de código: toda consulta al servicio de búsqueda debe incluir `country_code` (reforzado por el JWT del oficial, que incluye su `country_code`).
- Si se agregan muchos países pequeños, el número de particiones crece linealmente. Mitigación: una partición `OTHER` para países en piloto antes de crear su partición dedicada.

### Neutras

- La tabla `stolen_vehicles` **no** se particiona en este ADR: su cardinalidad es baja (tipicamente < 1 M registros por país) y sus accesos son por clave primaria `(country_code, plate_normalized)`. El índice B-tree existente es suficiente.
- Las tablas `incidents` y `audit_log` tampoco se particionan inicialmente; `pg_partman` por RANGE temporal se activará cuando `audit_log` supere los 200 M registros.

---

## 6. Plan de Evolución

| Fase | Criterio de activación | Acción |
|---|---|---|
| **Fase 1** (actual) | Volumen < 500 GB por partición de país | LIST por `country_code`; índices locales; pg_partman en modo monitoreo |
| **Fase 2** | Partición de un país supera 500 GB | Activar sub-particionamiento RANGE mensual en esa partición con pg_partman |
| **Fase 3** | Múltiples países con particiones grandes | Evaluar Citus o instancias separadas por país-región; los adapters hexagonales facilitan esta migración |

---

## 7. Referencias

- [postgresql-schema.md](./postgresql-schema.md) — DDL completo con la cláusula PARTITION.
- [data-lifecycle.md](./data-lifecycle.md) — Retención y archivado automatizado con pg_partman.
- [postgresql-ha.md](./postgresql-ha.md) — Configuración HA y parámetros de replicación.
- ADR-011 en [propuesta-arquitectura-hurto-vehiculos.md](../propuesta-arquitectura-hurto-vehiculos.md).
