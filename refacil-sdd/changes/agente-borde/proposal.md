# proposal: agente-borde

## Objetivo

Los dispositivos de captura desplegados en vías públicas operan en condiciones de hardware muy limitado (2 cores, 1 GB RAM, 16 GB disco), con conectividad GSM intermitente y energía que puede fallar hasta 4 horas (batería de respaldo de solo 1 hora). Sin un agente autónomo con persistencia local, cualquier corte de energía o de red produce pérdida de evidencia de vehículos en tránsito, incluyendo potencialmente la captura de un vehículo hurtado. Este change especifica el agente Go que corre en cada dispositivo de borde como proceso único, resolviendo la resiliencia a fallos y la integración con el componente ANPR ya instalado.

## Alcance

**Incluye:**
- ANPR Adapter SPI: interfaz Go con implementaciones intercambiables (REST local, socket UNIX, watch file, named pipe) para consumir placas del componente ANPR existente sin modificarlo.
- Collector: normalización del evento capturado — placa + coordenadas GPS + timestamp dual (wall-clock + monotónico) + metadatos del dispositivo.
- Queue Manager: persistencia en SQLite con modo WAL y `synchronous=NORMAL` antes de intentar cualquier envío; orden FIFO con prioridad elevada para hits del Bloom filter.
- Image Store local: buffer rotativo en filesystem con política de retención de 7 días o 10 GB (lo que ocurra primero).
- Bloom Filter: implementación Go en memoria con persistencia en disco; tasa de falsos positivos ~1% para 1 M de placas con huella ~1.2 MB (n=1M, fp=1%, incluyendo overhead de estructura Go); actualización incremental vía MQTT retained topic.
- Uploader: transmisión de metadatos de eventos por MQTT QoS 1 con sesiones persistentes; imágenes subidas por HTTP PUT a URL pre-firmada (object storage S3-API).
- Health Beacon: publicación periódica de métricas de salud del dispositivo (CPU, RAM, disco, batería, cobertura GSM, drift NTP) vía MQTT.
- Config/OTA Manager: recepción de configuración remota y actualizaciones de binario firmadas con cosign vía MQTT retained topic.
- Watchdog: integración con `systemd WatchdogSec` para reinicio automático ante hangs o consumo anómalo.
- Restricciones de footprint: binario estático de ~30 MB; consumo sostenido < 5% CPU y < 50 MB RSS.

**Excluye:**
- La PKI de dispositivos (certificados mTLS): especificada en el change `pki-dispositivos` (ADR-010).
- El MQTT Broker (EMQX) y su configuración de ACL: responsabilidad del pilar de comunicación segura.
- El componente ANPR existente: se consume su interfaz sin modificarlo.
- El Upload Service cloud (proveedor de URLs pre-firmadas): responsabilidad del pilar de ingestión.
- El Bloom Filter Generator cloud (cálculo del filtro por país): responsabilidad del pilar de procesamiento.
- Interfaces de usuario, API Gateway y frontend.

## Justificación

La propuesta de arquitectura (ADR-001, ADR-002, ADR-009) establece el patrón edge-first con store-and-forward como pilar fundamental de confiabilidad. Sin este agente especificado y construido, el sistema carece de su punto de entrada de datos: ninguna detección de vehículo hurtado puede llegar a la nube. Los requisitos del caso de estudio exigen explícitamente "definir un mecanismo para acceder a la información en cada dispositivo" y "garantizar la recolección y sincronización de los datos al sistema central" bajo condiciones de energía y red adversas. La especificación detallada de este componente es el prerequisito bloqueante para avanzar con los pilares de ingestión y procesamiento.

## Restricciones

- El binario debe compilar como estático para Linux ARM64 y AMD64 sin dependencias de sistema externas.
- Debe operar dentro de los límites de hardware del dispositivo: 2 cores, 1 GB RAM, 16 GB disco.
- La cola SQLite debe sobrevivir a un corte abrupto de energía sin corrupción de datos (modo WAL + `synchronous=NORMAL`).
- La comunicación con la nube usa exclusivamente mTLS con certificado de dispositivo emitido por la PKI propia (ADR-010); el agente no acepta certificados no firmados por esa CA.
- Las actualizaciones OTA solo se aplican si la firma cosign es válida; ante firma inválida el agente mantiene el binario actual y reporta el error vía Health Beacon.
- El Bloom filter ocupa ~1.2 MB en memoria (n=1M placas, fp=1%, incluyendo overhead de estructura Go); bien dentro del presupuesto de 50 MB RSS del proceso sobre el hardware de referencia del caso de estudio (1 GB RAM).
- El agente no almacena rostros ni datos biométricos; solo placas, coordenadas y referencias a imágenes de vehículo (cumplimiento Ley 1581 y equivalentes).
- **Modo de captura por defecto (`upload_mode: stolen_only`):** el agente sube al cloud únicamente los eventos con hit en el Bloom filter. Esto minimiza banda GSM, volumen en cloud y exposición de datos de tránsito. El parámetro es configurable remotamente vía Config/OTA Manager (sin redeploy); el modo `all` —que sube todas las placas y usa el BF solo para priorización— requiere autorización institucional explícita del país operador.
