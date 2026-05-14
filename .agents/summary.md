# Summary

Sistema Anti-Hurto de Vehículos — propuesta de arquitectura edge-to-cloud.
Evaluación técnica para el cargo Coach Técnico · Ceiba · Autor: Erik Rodríguez.

| Área | Detalle |
|------|---------|
| Tipo de repo | Documentación técnica (Markdown + PDF) — sin código ejecutable |
| Dominio | Detección y alerta de vehículos hurtados en tiempo real |
| Stack propuesto | Go (agente borde), React + MapLibre (web), MQTT/EMQX, Kafka, PostgreSQL+PostGIS, OpenSearch, ClickHouse, Redis, Keycloak, MinIO/S3, Kubernetes |
| Test command | N/A — repositorio de documentación |
| Build / deps | N/A |

### Always
- Mantén consistencia terminológica (p.ej. "agente de borde", "ACL por país", "hot path / cold path").
- Usa Mermaid para diagramas nuevos o actualizados.
- Escribe en español formal orientado a audiencia técnica senior.
- Preserva la numeración y estructura de secciones de la propuesta.
- Basa cualquier recomendación tecnológica en los requisitos del caso de estudio.

### Never
- No inventes código ejecutable ni comandos de build/test que no existen en el repo.
- No alteres los atributos de calidad objetivos (99.9 % uptime, p95 < 2 s, etc.) sin indicación del autor.

### Ask
- Si el usuario pide comparar stacks alternativos no contemplados en la propuesta.
- Si se deben agregar secciones que no figuran en la plantilla original.
