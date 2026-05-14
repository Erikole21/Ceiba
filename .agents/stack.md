# Stack

## Stack propuesto (sistema a implementar)
| Capa | Tecnología |
|------|-----------|
| Agente borde | Go (binario único ~30 MB, SQLite WAL, Bloom filter) |
| Protocolo IoT | MQTT 5 / EMQX cluster, mTLS |
| Object storage | MinIO / S3-API |
| Backbone eventos | Kafka / Redpanda |
| BD operacional | PostgreSQL + PostGIS |
| Búsqueda | OpenSearch |
| Analítica | ClickHouse + Apache Superset |
| Caché / sesiones | Redis |
| Identidad | Keycloak (realm por país) |
| Frontend | React + MapLibre |
| Orquestación | Kubernetes |
| PKI | CA propia para certificados de dispositivo |

## Variables de entorno / configuración relevante
N/A — repositorio de documentación; no hay servicios desplegables.

## Integraciones externas (según propuesta)
- BDs policiales por país (heterogéneas: SOAP, REST, CDC, polling).
- Proveedores de identidad policiales (LDAP, BD relacional, SOAP/REST).
