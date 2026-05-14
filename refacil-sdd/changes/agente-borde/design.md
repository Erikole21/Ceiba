# design: agente-borde

## Archivos a crear

| Ruta | Propósito |
|------|-----------|
| `docs/agente-borde/overview.md` | Descripción narrativa del agente: ciclo de vida, flujo de datos interno, restricciones de footprint |
| `docs/agente-borde/anpr-adapter-spi.md` | Especificación de la interfaz SPI y las cuatro implementaciones (REST local, socket UNIX, watch file, named pipe); contrato de datos de entrada |
| `docs/agente-borde/collector.md` | Especificación del Collector: esquema del evento normalizado, reglas de construcción del `event_id` idempotente, manejo de timestamp dual |
| `docs/agente-borde/queue-manager.md` | Especificación del Queue Manager: esquema de la tabla SQLite, política de prioridad FIFO + HIGH, parámetros WAL, recuperación tras corte de energía |
| `docs/agente-borde/image-store.md` | Especificación del Image Store local: estructura de directorios, política de rotación (7 días / 10 GB), estrategia de eliminación ordenada |
| `docs/agente-borde/bloom-filter.md` | Especificación del Bloom filter: parámetros (m, k, FP target), formato de serialización en disco, protocolo de delta vía MQTT retained, modo degradado |
| `docs/agente-borde/uploader.md` | Especificación del Uploader: topics MQTT, flujo de obtención de URL pre-firmada, manejo de PUBACK y reintentos, idempotencia en reenvío |
| `docs/agente-borde/health-beacon.md` | Especificación del Health Beacon: esquema del payload, campos obligatorios y opcionales, compresión, frecuencia, alertas embebidas |
| `docs/agente-borde/config-ota-manager.md` | Especificación del Config/OTA Manager: formato del manifiesto, verificación cosign, procedimiento de reemplazo atómico del binario, rollback |
| `docs/agente-borde/watchdog.md` | Especificación del Watchdog: configuración systemd (`WatchdogSec`, `Restart=on-failure`), criterios de reinicio, secuencia de recuperación de estado |
| `docs/agente-borde/security.md` | Especificación de seguridad del agente: uso del certificado mTLS (referencia al change `pki-dispositivos`), restricciones de escritura en disco, privacidad de datos |
| `docs/agente-borde/adr-local.md` | Decisiones de diseño específicas del agente que complementan ADR-001, ADR-002, ADR-009 de la propuesta principal |

## Archivos a modificar

| Ruta | Cambios |
|------|---------|
| `docs/propuesta-arquitectura-hurto-vehiculos.md` | Agregar referencias cruzadas a `docs/agente-borde/` en la sección 2.3 (Diagrama del Agente de Borde) y en la sección 3.1 (tabla de componentes de borde); no alterar diagramas Mermaid existentes ni atributos de calidad |

## Archivos fuera de alcance (doNotTouch)

- `refacil-sdd/` — artefactos SDD gestionados por la metodología
- `.claude/`, `.cursor/` — configuración de agentes
- `AGENTS.md` — índice del proyecto; solo se modifica en el change `setup` o por instrucción explícita del autor
- `package-lock.json` — no aplica (repo de documentación)
- `.agents/` — documentación de sesión; no se modifica en este change

## Patrones y convenciones detectados

- **Idioma:** español formal orientado a audiencia técnica senior (según regla `Always` en `.agents/summary.md`).
- **Diagramas:** Mermaid para todo diagrama nuevo o actualizado; la propuesta usa secuencia, C4 L1/L2 y flowchart.
- **Terminología canónica:** "agente de borde", "ACL por país", "hot path / cold path", "store-and-forward"; mantener consistencia con la propuesta.
- **Estructura de sección:** preservar numeración de la propuesta principal; los documentos nuevos en `docs/agente-borde/` usan encabezados H2/H3 sin numeración propia para no colisionar.
- **Referencias cruzadas ADR:** toda decisión técnica nueva que complemente un ADR existente debe mencionarlo explícitamente (ej. "Complementa ADR-001").
- **Privacidad:** explicitar en cada especificación que no se almacenan rostros ni datos biométricos; solo placas, coordenadas y referencias a imágenes de vehículo (Ley 1581 y equivalentes).

## Dependencias entre tareas

1. `docs/agente-borde/overview.md` debe redactarse primero — establece el glosario y el flujo de datos que referencian el resto de documentos.
2. `docs/agente-borde/anpr-adapter-spi.md` y `docs/agente-borde/collector.md` deben completarse antes de `docs/agente-borde/queue-manager.md`, ya que el esquema del evento normalizado (Collector) define el esquema de la tabla SQLite (Queue Manager).
3. `docs/agente-borde/bloom-filter.md` debe completarse antes de `docs/agente-borde/queue-manager.md` (sección de priorización) y antes de `docs/agente-borde/uploader.md` (sección de orden de transmisión).
4. `docs/agente-borde/security.md` puede redactarse en paralelo con los demás, pero debe publicarse después de que `overview.md` establezca los límites del componente.
5. La modificación de `docs/propuesta-arquitectura-hurto-vehiculos.md` es el último paso, una vez que todos los documentos de `docs/agente-borde/` están consolidados.

## Nota de validación cruzada entre repositorios

El Uploader y el Config/OTA Manager interactúan con contratos definidos fuera del agente:

- **Upload Service** (pilar de ingestión): el contrato de la URL pre-firmada (endpoint, método HTTP, headers requeridos, TTL) debe acordarse con el change que especifique ese servicio. Ver `refacil-prereqs/BUS-CROSS-REPO.md` para el proceso de acuerdo entre repositorios.
- **Bloom Filter Generator** (pilar de procesamiento): el formato del topic MQTT retained y la estructura del delta (formato binario, versión, checksum) deben acordarse con el change que especifique ese generador. Ver `refacil-prereqs/BUS-CROSS-REPO.md`.
- **PKI de dispositivos** (change `pki-dispositivos`, ADR-010): el agente asume que el certificado mTLS y la clave privada están disponibles en rutas de sistema configuradas; cualquier cambio en el mecanismo de provisioning debe notificarse a este change.
