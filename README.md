# Sistema Anti-Hurto de Vehículos — Propuesta de Arquitectura

**Evaluación técnica · Cargo: Coach Técnico — Ceiba**
**Autor:** Erik Rodríguez

---

## ¿De qué trata este proyecto?

Este repositorio contiene una **propuesta de arquitectura edge-to-cloud** para un sistema que combate el hurto de vehículos en Latinoamérica. El sistema captura eventos de tránsito en dispositivos físicos instalados en las calles, los sincroniza hacia la nube, los correlaciona contra bases de datos policiales heterogéneas de vehículos hurtados por país, y expone los resultados a usuarios policiales mediante alertas en tiempo real, mapas de trayectorias y herramientas de analítica.

El caso de estudio completo está en [`docs/Caso de Estudio1 - DETECCION VEHÍCULOS HURTADOS.pdf`](docs/Caso%20de%20Estudio1%20-%20DETECCION%20VEH%C3%8DCULOS%20HURTADOS.pdf).

---

## La propuesta en una línea

Una plataforma **portable, multi-tenant por país, tolerante a fallos de borde** que cierra el ciclo completo: *detección en la calle → alerta al oficial → localización → recuperación del vehículo → actualización de la BD policial*.

### Los seis pilares

| # | Pilar | Qué resuelve |
|---|-------|-------------|
| 1 | **Edge resiliente con store-and-forward** | Hardware limitado (2 cores, 1 GB RAM), GSM intermitente, cortes de energía hasta 4 h |
| 2 | **Comunicación segura y eficiente** | mTLS por dispositivo, MQTT sobre GSM, imágenes sin saturar el broker |
| 3 | **Cloud-agnostic por diseño** | Sin lock-in: mismo stack en AWS, GCP, Azure u on-prem |
| 4 | **Anti-Corruption Layer por país** | Cada BD policial tiene su propio formato y tecnología; el núcleo habla un modelo canónico |
| 5 | **Identidad federada con Keycloak** | LDAP, DB, SOAP/REST, SAML, OIDC — una sola entrada para todas las policías |
| 6 | **Hot path / Cold path separados** | Alertas en p95 < 2 s; analítica y heatmaps sin afectar el tiempo real |

---

## Cómo se construyó este documento

La propuesta no se escribió manualmente de corrido — se usó la metodología **SDD-AI** (`refacil-sdd-ai`), que estructura el trabajo en *changes* (unidades de documentación) que pasan por las fases `propose → apply → test → verify → review` antes de considerarse completos.

### Los 7 changes

| # | Change | Pilar | Documentos generados |
|---|--------|-------|---------------------|
| 1 | `agente-borde` | 1 · Edge resiliente | 13 docs en [`docs/agente-borde/`](docs/agente-borde/overview.md) |
| 2 | `ingestion-mqtt` | 2 · Comunicación segura | 12 docs en [`docs/ingestion-mqtt/`](docs/ingestion-mqtt/overview.md) |
| 3 | `backbone-procesamiento` | 6 · Hot/Cold path | 17 docs en [`docs/backbone-procesamiento/`](docs/backbone-procesamiento/overview.md) |
| 4 | `sincronizacion-paises` | 4 · ACL por país | 18 docs en [`docs/sincronizacion-paises/`](docs/sincronizacion-paises/overview.md) |
| 5 | `almacenamiento-lectura` | 6 · Hot/Cold path | 18 docs en [`docs/almacenamiento-lectura/`](docs/almacenamiento-lectura/overview.md) |
| 6 | `api-frontend-analitica` | 6 · Hot/Cold path | 24 docs en [`docs/api-frontend-analitica/`](docs/api-frontend-analitica/overview.md) |
| 7 | `identidad-seguridad` | 5 · Identidad federada | 21 docs en [`docs/identidad-seguridad/`](docs/identidad-seguridad/overview.md) |

> **Total: 123 documentos de especificación**, todos verificados con el framework 3D (Completeness / Correctness / Coherence) y revisados con el checklist de calidad del equipo.

### El flujo por change

```
/refacil:propose  →  revisión humana  →  /refacil:apply  →  /refacil:verify  →  /refacil:review
   (artefactos SDD)     (aprobación)      (implementa docs)   (3D + specs)       (checklist calidad)
```

Los artefactos intermedios (proposal, design, tasks, specs) viven en [`refacil-sdd/changes/`](refacil-sdd/changes/).

---

## Cómo navegar la documentación

### Propuesta de arquitectura (punto de entrada)

→ **[`docs/propuesta-arquitectura-hurto-vehiculos.md`](docs/propuesta-arquitectura-hurto-vehiculos.md)**

Índice de la propuesta con links a cada sección:
- [Resumen y pilares](docs/propuesta/01-resumen.md)
- [Arquitectura y diagramas](docs/propuesta/02-arquitectura.md)
- [Componentes principales](docs/propuesta/03-componentes.md)
- [Decisiones de diseño (ADRs)](docs/propuesta/04-decisiones.md)
- [Consideraciones de calidad](docs/propuesta/05-calidad.md)
- [Supuestos](docs/propuesta/06-supuestos.md)
- [Apéndices](docs/propuesta/apendices.md)

### Especificación detallada por pilar

| Pilar | Documento de entrada |
|-------|---------------------|
| Agente de borde | [`docs/agente-borde/overview.md`](docs/agente-borde/overview.md) |
| Ingestión MQTT | [`docs/ingestion-mqtt/overview.md`](docs/ingestion-mqtt/overview.md) |
| Backbone de procesamiento | [`docs/backbone-procesamiento/overview.md`](docs/backbone-procesamiento/overview.md) |
| Sincronización de países | [`docs/sincronizacion-paises/overview.md`](docs/sincronizacion-paises/overview.md) |
| Almacenamiento de lectura | [`docs/almacenamiento-lectura/overview.md`](docs/almacenamiento-lectura/overview.md) |
| API, frontend y analítica | [`docs/api-frontend-analitica/overview.md`](docs/api-frontend-analitica/overview.md) |
| Identidad y seguridad | [`docs/identidad-seguridad/overview.md`](docs/identidad-seguridad/overview.md) |

### Recursos adicionales

- [`docs/glosario.md`](docs/glosario.md) — términos y abreviaturas
- [`docs/plan-de-trabajo.md`](docs/plan-de-trabajo.md) — estado de cada change
- [`refacil-sdd/changes/`](refacil-sdd/changes/) — artefactos SDD por change
