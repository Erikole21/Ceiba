# Apéndices

← [Índice](../propuesta-arquitectura-hurto-vehiculos.md)

---

## Apéndice A — Mapeo Requisito ↔ Decisión

| Requisito del enunciado | Decisión / componente |
|---|---|
| Hardware limitado (2c/1GB/16GB) | Agente Go ligero (ADR-001), SQLite WAL, Bloom filter (ADR-009) |
| GSM intermitente | Store-and-forward (ADR-001), MQTT QoS 1 (ADR-002) |
| Batería 1 h / cortes hasta 4 h | Persistencia local antes de envío; reintento idempotente |
| ANPR ya existente | ANPR Adapter SPI con múltiples implementaciones |
| 9 campos canónicos + extra por país | Modelo canónico + `extensions: jsonb` (ADR-006) |
| Cambio de cloud / nube privada | Hexagonal (ADR-005), K8s (ADR-004), tabla §5.6 |
| Escalado dinámico | K8s + HPA, particionamiento Kafka (§5.2) |
| Alta disponibilidad | Multi-AZ + DR (§5.1) |
| Comunicación segura device↔sistema | mTLS + PKI (ADR-010), ACL en broker |
| Integración LDAP/DB/SOAP/REST policial | Keycloak broker + SPIs (ADR-007) |
| Buscar placa → eventos + mapa + prueba visual | search-service + OpenSearch + MapLibre + thumbnails en object storage (§2.6) |
| Facilitar la recuperación de vehículos | Alerta en tiempo real con ubicación exacta + incident-service (§2.6, §3.6) |
| Análisis de tendencias / vías peligrosas / rutas de escape | ClickHouse + H3 + Superset + Kepler.gl (§3.6) |
| Consolidar info de cada dispositivo | Pipeline Enrichment + proyecciones CQRS (ADR-008) |

---

## Apéndice B — Roadmap Sugerido (3 Fases)

**Fase 1 — Piloto.** Un país, 1 000 dispositivos. Edge agent + ingestión MQTT + S3-compat + Postgres + UI búsqueda y mapa básicos. Adapter país piloto. Keycloak con federación LDAP. PKI básica. Sin DR, sin analítica avanzada.

**Fase 2 — Producción Regional.** 2–4 países, hasta 100 K dispositivos. ClickHouse + Superset. Alertas en tiempo real. DR warm. Multi-tenant pulido. SPI custom para SOAP/REST. GitOps maduro. Observabilidad completa.

**Fase 3 — Expansión Global.** Multi-región, residencia por país. Modelos ML para predicción de zonas calientes. App móvil para oficiales en terreno. Federación interpaís opcional. Optimización de costos con tiering de imágenes.
