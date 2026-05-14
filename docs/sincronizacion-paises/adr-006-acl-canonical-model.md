# ADR-006 — Anti-Corruption Layer por País con Modelo Canónico Versionado

**Estado:** Aprobado
**Fecha:** 2026-05-13
**Autores:** Erik Rodríguez
**Revisores:** Equipo de arquitectura
**Change asociado:** `sincronizacion-paises`

---

## Contexto

El Sistema Anti-Hurto de Vehículos debe integrar las bases de datos de vehículos hurtados de múltiples países de América Latina. Cada institución policial mantiene su propia base de datos con:

- **Tecnología heterogénea:** Oracle, SQL Server, PostgreSQL, MySQL, archivos CSV/XML, APIs SOAP heredadas, APIs REST modernas.
- **Esquemas propietarios:** Cada país tiene nombres de columna, tipos de datos y cardinalidades distintos para los mismos conceptos (matrícula, fecha del hurto, propietario, etc.).
- **Modos de entrega distintos:** Algunas BDs permiten CDC en tiempo real; otras solo permiten consultas periódicas; otras solo entregan archivos en un SFTP.
- **Restricciones de soberanía:** Algunas instituciones no pueden conectarse directamente al cloud; requieren que el conector opere on-prem dentro del territorio nacional.
- **Evolución independiente:** Los esquemas de las BDs policiales pueden cambiar sin aviso previo; el sistema no puede imponer un formato a las instituciones.

El requisito funcional del caso de estudio establece 9 campos mandatorios que el sistema debe mantener para cada vehículo hurtado, más la posibilidad de campos adicionales por país.

---

## Opciones Evaluadas

### Opción A — Adaptador genérico parametrizable (configuración YAML)

Un único microservicio "adaptador universal" que acepta una configuración YAML que describe el mapeo de campos, el tipo de fuente (CDC/JDBC/API/archivo), las credenciales y las reglas de transformación.

**Pros:**
- Un único repositorio y un único artefacto desplegable.
- Menos overhead operativo inicial.
- Nuevos países se incorporan solo editando YAML.

**Contras:**
- Los casos reales son demasiado heterogéneos para ser cubiertos por un mapa de configuración: conversiones de tipos complejas, lógica de desduplicación específica del país, autenticación propietaria, parseo de formatos XML heredados con namespaces no estándar.
- El YAML de configuración se convierte en un lenguaje de programación disfrazado (condicionales, loops, funciones de transformación), con menor expresividad y mayor superficie de bugs.
- Un fallo en el adaptador universal afecta a todos los países simultáneamente.
- La evolución de la lógica de un país requiere modificar el componente compartido, con riesgo de regresión en otros países.

**Veredicto:** Descartada.

### Opción B — Adaptador por país sin modelo canónico (mensajes directos al backbone)

Cada país tiene su propio microservicio adaptador, pero publica directamente en el backbone con su esquema propietario. El Canonical Vehicles Service del backbone entiende todos los esquemas.

**Pros:**
- Los adaptadores son más simples (no necesitan conocer el modelo canónico).
- Flexibilidad total en el payload.

**Contras:**
- El Canonical Vehicles Service debe conocer todos los esquemas propietarios de todos los países. Se convierte en el punto de acoplamiento con toda la heterogeneidad que se quería evitar.
- Evolución acoplada: un cambio en el esquema de la BD policial de Colombia requiere modificar el Canonical Vehicles Service, que es un componente del núcleo.
- Imposible versionar el contrato de forma centralizada.
- Violación directa del principio de Anti-Corruption Layer del Domain-Driven Design: el dominio del núcleo queda contaminado con los modelos de dominio externos.

**Veredicto:** Descartada.

### Opción C — Adaptador por país con modelo canónico versionado (DECISIÓN)

Cada país tiene su propio microservicio adaptador. El adaptador conoce el esquema propietario de su fuente y el modelo canónico del sistema. La conversión ocurre en el adaptador. El backbone del sistema solo ve el modelo canónico.

El modelo canónico está definido como schema Avro versionado en el Schema Registry, con reglas de compatibilidad `BACKWARD`.

**Pros:**
- El núcleo del sistema está completamente aislado de la heterogeneidad externa.
- Un cambio en la BD policial de Colombia solo afecta al adaptador de Colombia.
- El onboarding de un nuevo país es un proyecto delimitado que no modifica ningún componente del núcleo.
- El contrato del sistema se puede evolucionar de forma controlada y versionada.
- Los adaptadores pueden desplegarse on-prem cuando la soberanía de datos lo requiere.
- Un fallo en el adaptador de un país no afecta al procesamiento de otros países.

**Contras:**
- Costo de mantenimiento por país: cada adaptador es un repositorio / módulo independiente.
- El adaptador debe conocer dos dominios: el esquema de la BD policial y el modelo canónico.
- Requiere disciplina en el proceso de onboarding para garantizar que los mapeos sean correctos.

**Veredicto:** Seleccionada.

---

## Decisión

**Se adopta la Opción C: adaptador dedicado por país con modelo canónico versionado.**

### Principios derivados de esta decisión

1. **Un adaptador, un país.** Cada adaptador es responsable exclusivamente de un país (un `country_code`). No existen adaptadores multipaís.
2. **El adaptador es la frontera del dominio externo.** Toda lógica relacionada con el esquema propietario de la BD policial reside en el adaptador y no se filtra al núcleo del sistema.
3. **El modelo canónico es inmutable para el backbone.** El núcleo del sistema (Canonical Vehicles Service, Matcher Service, Alert Service) solo conoce el modelo canónico. Nunca conoce los esquemas propietarios de las BDs policiales.
4. **Versionado estricto del contrato.** El schema Avro del modelo canónico se gestiona en el Schema Registry con compatibilidad `BACKWARD`. Cambios incompatibles requieren un proceso formal de migración.
5. **Soberanía de datos por topología.** El adaptador puede desplegarse on-prem dentro del territorio nacional. Solo los mensajes Avro del modelo canónico (ya normalizados) cruzan la frontera del territorio.
6. **Extensibilidad sin contaminación del núcleo.** Campos adicionales por país se transmiten en el campo `extensions: map<string,string>` del schema canónico, sin afectar la versión principal del esquema.

---

## Consecuencias

### Positivas

- El núcleo del sistema es estable e independiente de la evolución de las BDs policiales externas.
- El onboarding de un nuevo país requiere implementar un adaptador (semanas de trabajo) sin modificar ningún componente del núcleo.
- Los fallos de adaptadores están aislados: si el adaptador de Venezuela falla, Colombia sigue funcionando normalmente.
- La soberanía de datos es técnicamente garantizable: el adaptador on-prem solo expone el modelo canónico.
- El Schema Registry provee auditoría de todos los contratos publicados y su historial de versiones.

### Negativas y mitigaciones

| Consecuencia negativa | Mitigación |
|---|---|
| Mantenimiento de N adaptadores | El [`country-adapter-framework.md`](./country-adapter-framework.md) define una biblioteca base reutilizable que reduce el costo de cada adaptador al mapeo de campos y la configuración de conectividad. |
| Riesgo de mapeo incorrecto | El proceso de onboarding incluye una fase de validación con el contraparte policial antes del despliegue en producción (ver [`country-onboarding-guide.md`](./country-onboarding-guide.md)). |
| Evolución del modelo canónico requiere actualizar todos los adaptadores | La política de compatibilidad `BACKWARD` minimiza la necesidad de actualizaciones simultáneas. Los cambios minor se propagan de forma asíncrona. |

---

## Relación con Otros ADRs

| ADR | Relación |
|---|---|
| ADR-005 | Los adaptadores siguen la arquitectura hexagonal: la lógica de mapeo es el dominio; la conectividad con la BD policial y la publicación en Kafka son adaptadores de infraestructura. |
| ADR-009 | El Edge Distribution Service, que consume el resultado de este ADR para construir los Bloom filters, es habilitado por la existencia de un modelo canónico uniforme. |
| ADR-011 | El `country_code` del modelo canónico actúa como discriminador de tenant en todos los almacenes del sistema. |
| ADR-012 | Kafka actúa como la frontera durable entre el mundo de los adaptadores de país y el núcleo del sistema. |

---

## Referencias

- [`canonical-model.md`](./canonical-model.md) — Especificación del modelo canónico
- [`avro-schema.md`](./avro-schema.md) — Schema Avro y política de versionado
- [`country-adapter-framework.md`](./country-adapter-framework.md) — Marco de implementación de adaptadores
- [`data-sovereignty.md`](./data-sovereignty.md) — Política de soberanía de datos
- Evans, Eric. *Domain-Driven Design*, 2003 — Patrón Anti-Corruption Layer (Cap. 14)
- Propuesta de arquitectura, sección 4 (ADR-006 resumido): [`propuesta-arquitectura-hurto-vehiculos.md`](../propuesta-arquitectura-hurto-vehiculos.md#adr-006--anti-corruption-layer-por-país--modelo-canónico-versionado)
