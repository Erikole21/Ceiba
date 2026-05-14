# proposal: sincronizacion-paises

## Objetivo

Cada base de datos policial nacional expone sus registros de vehículos hurtados con tecnología, esquema y mecanismo de acceso propios. El núcleo del sistema no puede acoplarse a esos esquemas heterogéneos sin introducir deuda técnica severa y romper la regla de aislamiento multi-tenant por país. Adicionalmente, los dispositivos de borde requieren una copia local actualizada de placas hurtadas (Bloom filter) para operar sin conectividad, y esa copia debe mantenerse sincronizada a medida que las bases de datos policiales cambian.

Este change implementa el **Pilar 4 — Anti-Corruption Layer por país** de la arquitectura del Sistema Anti-Hurto de Vehículos: la capa que ingiere datos de vehículos hurtados desde cada BD policial nacional a través de adaptadores dedicados, mantiene una tabla canónica centralizada y distribuye actualizaciones del Bloom filter a los dispositivos de borde.

## Alcance

**Incluye:**
- Marco de adaptadores por país (`country-adapter-framework`) con cuatro modos de integración: CDC vía Debezium, JDBC polling, cliente SOAP/REST programado y file drop watcher (CSV/XML por filesystem o SFTP)
- Modelo canónico de vehículo hurtado con los 9 campos mandatorios del caso de estudio (`plate`, `stolen_date`, `stolen_location`, `owner_id`, `brand`, `class`, `line`, `color`, `model`), más `country_code`, `status`, `schema_version` y `extensions` (JSONB)
- Esquema Avro del evento canónico y su registro en Schema Registry con compatibilidad backward
- Canonical Vehicles Service: consumidor del tópico `stolen.vehicles.events`, validación de esquema vía Schema Registry, upsert en PostgreSQL (`canonical_vehicles`), actualización de lista roja en Redis y notificación al Edge Distribution Service
- Edge Distribution Service: mantenimiento de versiones del Bloom filter por país, cálculo de delta entre versiones, publicación en MQTT retained (`countries/{country_code}/bloom-filter`) y lógica de descarga completa ante BF desactualizado (> 7 días)
- Procesamiento del evento `RECOVERED` producido por el incident-service (`backbone-procesamiento`), incluyendo regeneración del BF para eliminar la placa de la lista activa
- Documentación del proceso repetible de incorporación de un nuevo país (onboarding guide)
- ADR-006 (ACL por país + modelo canónico versionado) y ADR-009 (Bloom filter en borde para hot-path) como documentos de decisión formales
- Especificación de SLAs de frescura por modo de integración
- Guías de despliegue Helm y módulos Terraform del subsistema

**Excluye:**
- Implementación de adaptadores para países concretos (solo el marco y el contrato de adaptador)
- Gestión de credenciales de los contenedores de adaptador (pertenece a `identidad-seguridad` con Vault)
- Esquema de la tabla `canonical_vehicles` para consultas analíticas (pertenece a `almacenamiento-lectura`)
- Lógica del Matcher Service que lee de Redis (pertenece a `backbone-procesamiento`)
- Frontend o API de consulta de la lista canónica

## Justificación

El caso de estudio establece explícitamente que "cada país podrá proveer datos adicionales del vehículo hurtado y de su propietario" y que la plataforma debe operar en múltiples países con bases de datos heterogéneas. Sin la capa de anti-corrupción, cualquier variación en el esquema de la BD de un país requeriría cambios en el núcleo del sistema, violando el principio de inversión de dependencias (ADR-005). Adicionalmente, los dispositivos de borde especificados en el change `agente-borde` esperan un protocolo de delta de Bloom filter vía MQTT retained; sin este change ese protocolo no tiene productor, dejando el Pilar 1 (Edge resiliente) funcional pero incompleto en su modo de actualización de lista roja.

## Restricciones

- El modelo canónico debe respetar exactamente los 9 campos del caso de estudio; su renombrado o eliminación invalida los contratos con `backbone-procesamiento` y `almacenamiento-lectura`
- La soberanía de datos impone que el adaptador de cada país puede ejecutarse on-premises dentro del territorio nacional; solo el evento canónico (no los datos crudos de la BD policial) se publica en Kafka
- Los parámetros del Bloom filter (tasa FP ≈ 1 %, < 1 MB para 1 M de placas) son fijos según el caso de estudio; cualquier cambio requiere regenerar el BF completo y notificar al change `agente-borde`
- La compatibilidad del esquema Avro debe ser backward; un campo nuevo obligatorio implica versión mayor y actualización coordinada de todos los adaptadores activos
- Los SLAs de frescura dependen del modo de integración: CDC < 1 s de latencia de replicación; JDBC polling ≤ ventana configurada (mín. 1 min, máx. 15 min); SOAP/REST y file drop ≤ período de ejecución del job programado; estos valores deben documentarse y acordarse por país antes del despliegue
- Existen dependencias de interfaz con `backbone-procesamiento` (aprobado), `ingestion-mqtt` (aprobado), `identidad-seguridad` (pendiente) y `almacenamiento-lectura` (pendiente); los contratos compartidos deben acordarse formalmente según `refacil-prereqs/BUS-CROSS-REPO.md`
