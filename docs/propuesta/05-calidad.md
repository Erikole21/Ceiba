# 5. Consideraciones de Calidad

← [Índice](../propuesta-arquitectura-hurto-vehiculos.md)

---

## 5.1 Disponibilidad

**Objetivo:** 99.9 % anual (≈ 8.7 h de indisponibilidad).

- Despliegue multi-AZ activo-activo en la región primaria.
- Réplica asíncrona a región DR con RPO ≤ 5 min y RTO ≤ 30 min.
- Postgres en HA (Patroni, RDS Multi-AZ o equivalente) con failover automático.
- Kafka cluster mínimo de 3 brokers, replication factor 3, `min.insync.replicas=2`.
- MQTT broker en cluster con session persistence.
- Servicios stateless con `PodDisruptionBudget` y `topologySpreadConstraints`.
- Object storage replicado entre AZs y opcionalmente cross-region.
- **Edge:** la arquitectura store-and-forward absorbe caídas del backend sin pérdida de evidencia (horas o días según retención de disco).

---

## 5.2 Escalabilidad

- **Horizontal en cómputo:** todos los servicios son stateless o particionables; HPA por CPU/RPS/queue lag.
- **Particionamiento de Kafka:** topics particionados por `device_id` (orden por device garantizado).
- **Sharding de ClickHouse** por país y mes. Sharding lógico de Postgres por país si una sola instancia no escala (Citus o múltiples instancias).
- **OpenSearch** con índices por país + tiempo (rolling indices).
- **Crecimiento previsto.** A 100 K devices con ~1 K eventos/día/device: ~1.2 K eventos/seg promedio, picos ~5 K/seg, manejables con un cluster Kafka modesto. A 1 M devices: re-particionamiento y crecimiento de cluster, sin cambios arquitectónicos.

---

## 5.3 Seguridad

- **Borde ↔ Nube.** mTLS con cert por dispositivo emitido por PKI propia; rotación cada 90 días; CRL/OCSP.
- **Identidad de usuarios.** OIDC vía Keycloak; MFA configurable por realm; políticas de password heredadas del IdP del país.
- **Autorización.** RBAC fino: oficial, supervisor, analista, administrador, auditor; permisos por país/ciudad.
- **Datos en tránsito.** TLS 1.2+ end-to-end; HSTS en frontend.
- **Datos en reposo.** Cifrado a nivel de volumen (LUKS / KMS-managed); cifrado a nivel de campo (pgcrypto) para PII sensible.
- **Pseudonimización.** El DNI se almacena hashed (HMAC con sal por país) en la capa analítica; sólo el dominio operativo retiene el valor en claro, bajo audit log.
- **Secrets.** Vault con auto-rotación; cero secretos en imágenes ni en variables de entorno en git.
- **Auditoría.** Todos los accesos a placas y eventos de vehículos quedan en un audit log inmutable (append-only, S3 Object Lock).
- **Supply chain.** Imágenes firmadas con cosign; SBOM con Syft; escaneo con Trivy en CI; gates en ArgoCD.
- **Borde.** Validación de firmas en OTA; modo "config locked" si watchdog detecta tampering.
- **Privacidad.** Cumplimiento Ley 1581 (Colombia) y equivalentes; minimización (no se almacenan rostros, sólo placas y vehículo); retención configurable por país.

---

## 5.4 Rendimiento

| Operación | SLO |
|---|---|
| Captura → recepción en cloud (red nominal) | p95 < 1 s, p99 < 3 s |
| Captura → alerta entregada a oficial (hot path) | p95 < 2 s, p99 < 5 s |
| Búsqueda por placa (UI) | p95 < 300 ms |
| Consulta de mapa con ≤ 10 K eventos | p95 < 800 ms |
| Dashboard agregado (rango 7 días, 1 ciudad) | p95 < 3 s |
| Sincronización inicial de Bloom filter en device | < 30 s sobre GSM 3G |

- Caches Redis para hot-list de vehículos y para búsquedas frecuentes.
- ClickHouse pre-agrega por hora/día/celda H3 mediante materialized views.
- CDN delante del frontend y de las imágenes thumbnails para reducir latencia geográfica.

---

## 5.5 Mantenibilidad y Evolución

- **12-factor apps** para todos los servicios.
- **Domain-Driven Design ligero:** límites por bounded context (devices, events, vehicles, identity, alerts, analytics).
- **Versionado de contratos** (schemas Avro + Schema Registry; OpenAPI para REST).
- **IaC total** con Terraform; clústeres reproducibles.
- **GitOps** vía ArgoCD; promociones por PR.
- **Observabilidad nativa:** todo servicio expone métricas Prometheus, logs JSON, trazas OTel.
- **Testing piramidal:** unit (host), integration (Testcontainers), contract (Pact), e2e mínimo, smoke en cada despliegue.

---

## 5.6 Portabilidad (Cloud-agnostic)

| Capacidad | Implementación de referencia | Alternativas portables |
|---|---|---|
| Cómputo | K8s (EKS/GKE/AKS) | RKE2, OpenShift, Kubespray |
| Object storage | S3-API | MinIO, Ceph RadosGW, GCS, Azure Blob |
| Database | PostgreSQL gestionado | Patroni, CrunchyData on-prem |
| Streaming | Kafka gestionado | Strimzi en K8s, Redpanda |
| Identidad | Keycloak | Keycloak (mismo) |
| Secretos | Vault | Vault (mismo) o KMS-cloud detrás de adapter |
| BI | Superset | Superset (mismo) |

> Estrategia: **preferir OSS portable**. Cuando se use un servicio gestionado, hacerlo siempre **detrás de un puerto/adapter**. La migración entre nubes pasa por reemplazar adapters, no por reescribir lógica de dominio.

---

## 5.7 Resiliencia Operacional

- **Caos engineering** periódico (Litmus/Chaos Mesh) en pre-prod: kill de pods, partición de red, lag de Kafka, latencia de S3.
- **Runbooks** versionados por incidente típico.
- **Backups** Postgres (PITR), Kafka (snapshots de topics críticos), ClickHouse, Keycloak realms.
- **DR drills** trimestrales.

---

## 5.8 Cumplimiento y Soberanía de Datos

- Capacidad de **residencia por país**: datos de Colombia en región sudamericana, etc.
- **Retención configurable** por país (eventos, imágenes, audit log).
- **Right to be forgotten** aplicable cuando un vehículo sale de la lista de hurtos (lifecycle automático).
- **Consentimiento institucional** documentado por país.
