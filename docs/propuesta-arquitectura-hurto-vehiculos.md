# Propuesta de Arquitectura — Sistema para Combatir el Hurto de Vehículos

**Caso:** Evaluación técnica · Cargo: Coach Técnico — Ceiba
**Autor:** Erik Rodríguez
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## Índice

| # | Sección | Descripción |
|---|---|---|
| 1 | [Resumen y Pilares](propuesta/01-resumen.md) | Los seis pilares de la arquitectura y por qué cada uno responde a un requisito del enunciado |
| 2 | [Arquitectura y Diagramas](propuesta/02-arquitectura.md) | C4 L1/L2, agente de borde, hot path, cold path, read path, vista de despliegue |
| 3 | [Componentes Principales](propuesta/03-componentes.md) | Tablas de responsabilidad y tecnología para cada capa: borde, ingestión, backbone, países, almacenamiento, API/UI, identidad, observabilidad |
| 4 | [Decisiones de Diseño (ADRs)](propuesta/04-decisiones.md) | ADR-001 a ADR-012: contexto, decisión, alternativas y consecuencias |
| 5 | [Consideraciones de Calidad](propuesta/05-calidad.md) | Disponibilidad, escalabilidad, seguridad, rendimiento (SLOs), mantenibilidad, portabilidad, resiliencia operacional, cumplimiento |
| 6 | [Supuestos](propuesta/06-supuestos.md) | 16 supuestos del sistema: ANPR, conectividad, volumen, imágenes, reloj, GSM, instituciones, modelo canónico, retención, operación, provisioning, OTA, idioma, tiempo a producción, `upload_mode` |
| A | [Apéndices](propuesta/apendices.md) | Mapeo Requisito ↔ Decisión · Roadmap sugerido (3 fases) |

---

## Recursos transversales

| Recurso | Descripción |
|---|---|
| [Glosario](glosario.md) | Términos técnicos usados a lo largo de todos los pilares |

---

## Documentación detallada por pilar

Cada pilar cuenta con su propia carpeta de especificaciones:

| Pilar | Carpeta | Entrada |
|---|---|---|
| 1 — Edge Agent | [`docs/agente-borde/`](agente-borde/overview.md) | [overview.md](agente-borde/overview.md) |
| 2 — Ingestión MQTT | [`docs/ingestion-mqtt/`](ingestion-mqtt/overview.md) | [overview.md](ingestion-mqtt/overview.md) |
| 3 — Backbone y Procesamiento | [`docs/backbone-procesamiento/`](backbone-procesamiento/overview.md) | [overview.md](backbone-procesamiento/overview.md) |
| 4 — ACL por País | [`docs/sincronizacion-paises/`](sincronizacion-paises/overview.md) | [overview.md](sincronizacion-paises/overview.md) |
| 5 — Identidad y Seguridad | [`docs/identidad-seguridad/`](identidad-seguridad/overview.md) | [overview.md](identidad-seguridad/overview.md) |
| 6 — Almacenamiento de Lectura | [`docs/almacenamiento-lectura/`](almacenamiento-lectura/overview.md) | [overview.md](almacenamiento-lectura/overview.md) |
| 7 — API, Frontend y Analítica | [`docs/api-frontend-analitica/`](api-frontend-analitica/overview.md) | [overview.md](api-frontend-analitica/overview.md) |
