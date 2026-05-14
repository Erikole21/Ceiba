# proposal: ingestion-mqtt

## Objetivo

Los dispositivos de borde producen dos tipos de datos con naturalezas opuestas: metadatos de evento pequeños (~1 KB, frecuentes, críticos en el hot path) e imágenes grandes (50–500 KB, diferibles, evidencia legal). Ambos tipos deben viajar de forma confiable sobre el mismo enlace GSM intermitente sin comprometer la seguridad ni saturar el canal. Este change implementa el Pilar 2 (Comunicación segura y eficiente) de la arquitectura del Sistema Anti-Hurto de Vehículos: la capa de ingestión cloud que recibe telemetría de todos los dispositivos vía MQTT, acepta imágenes mediante upload directo a object storage vía URL pre-firmada, y puentea los eventos MQTT hacia Kafka como backbone para el procesamiento aguas abajo.

## Alcance

**Incluye:**
- Clúster EMQX 5.x (3+ nodos) con mTLS obligatorio, ACL estricta por `device_id`, sesiones persistentes con `session_expiry_interval` configurable y configuración de tópicos según el esquema acordado (ADR-002).
- Upload Service: microservicio que valida la identidad del dispositivo (certificado mTLS o JWT) y emite URLs pre-firmadas con TTL de 5 minutos para subida directa a object storage; incluye modo proxy-upload como fallback para dispositivos detrás de NAT/APN corporativa; arquitectura hexagonal con puerto/adaptador para el proveedor de object storage (ADR-005).
- Object Storage (MinIO por defecto, S3-API compatible): almacenamiento de imágenes completas y miniaturas con cifrado server-side (clave KMS), lifecycle policy (imágenes completas 6 meses, miniaturas 24 meses) y replicación Multi-AZ.
- MQTT→Kafka Bridge: componente que consume eventos del clúster EMQX y los publica en el tópico Kafka `vehicle.events.raw` preservando el orden por `device_id` mediante ruteo determinístico a partición Kafka.
- Especificación del esquema de tópicos MQTT y reglas ACL multi-tenant por `country_code` (ADR-011).
- Documentación operativa: Helm charts para Kubernetes, módulos Terraform, runbooks de escalado y rotación de certificados.

**Excluye:**
- La PKI de dispositivos (Vault, emisión y rotación de certificados mTLS): responsabilidad del change `identidad-seguridad` (ADR-010).
- El agente de borde (productor MQTT): especificado en el change `agente-borde` (aprobado).
- El backbone de procesamiento aguas abajo (consumidores de `vehicle.events.raw`): responsabilidad del change `backbone-procesamiento` (pendiente).
- El Bloom Filter Generator cloud (cálculo y distribución del filtro por país): responsabilidad del pilar de procesamiento.
- La API Gateway, Web App y capas de presentación.
- El sistema de identidad federada Keycloak para usuarios humanos: responsabilidad del change `identidad-seguridad`.

## Justificación

Sin la capa de ingestión, los datos capturados por el agente de borde no tienen destino cloud: los eventos MQTT no llegan a Kafka y las imágenes no tienen dónde almacenarse. El change `agente-borde` ya está aprobado y sus dispositivos quedarán bloqueados hasta que este pilar esté implementado. La separación entre metadatos por MQTT e imágenes por presigned URL (ADR-003) es la decisión arquitectural clave que permite optimizar el uso del canal GSM: los ~1 KB de metadatos viajan con baja latencia y garantía QoS 1, mientras que las imágenes (hasta 500 KB) se suben fuera de banda sin bloquear el broker. La implementación de este pilar es el paso desbloqueante para el change `backbone-procesamiento` y para la demostración end-to-end del sistema.

## Restricciones

- mTLS obligatorio en EMQX; ninguna conexión sin certificado de dispositivo válido emitido por la CA de Vault es aceptada (ADR-010).
- ACL estricta: un dispositivo solo puede publicar en `devices/{device_id}/*` y suscribirse a `devices/{device_id}/config`, `countries/{country_code}/bloom-filter` y `devices/{device_id}/ota`; la violación de ACL produce desconexión inmediata (ADR-002, ADR-011).
- El TTL de la URL pre-firmada es de exactamente 5 minutos; el Upload Service no emite URLs sin validación previa de identidad del dispositivo.
- El MQTT→Kafka Bridge debe garantizar orden por `device_id`: todos los eventos del mismo dispositivo deben llegar a la misma partición Kafka (clave de partición = `device_id`).
- El object storage debe usar cifrado server-side con clave gestionada por KMS; no se permite almacenamiento sin cifrar.
- El `session_expiry_interval` de EMQX debe ser finito y dimensionado para evitar agotamiento de memoria ante miles de dispositivos offline simultáneos.
- La solución se despliega en Kubernetes vía Helm; la infraestructura subyacente se aprovisiona con Terraform.
- Compatibilidad S3-API obligatoria: el adaptador de object storage debe funcionar con MinIO (on-prem), AWS S3, GCS y Azure Blob sin modificar el núcleo del Upload Service (ADR-005).
- CRL/OCSP: los certificados de dispositivo revocados deben ser rechazados en la siguiente reconexión; el broker consulta el endpoint OCSP de Vault (ADR-010).
