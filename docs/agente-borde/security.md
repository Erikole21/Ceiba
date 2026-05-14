# Seguridad del Agente de Borde

**Subsistema:** Seguridad (transversal)  
**Responsabilidad:** Definir los controles de seguridad aplicados en el agente: identidad mTLS, integridad OTA, restricciones de filesystem, privacidad de datos y cumplimiento normativo  
**Referencia arquitectural:** [Visión General](./overview.md) · [Propuesta §5.3](../propuesta-arquitectura-hurto-vehiculos.md#53-seguridad) · [Propuesta ADR-010](../propuesta-arquitectura-hurto-vehiculos.md#adr-010--pki-propia-con-certificados-cortos-para-dispositivos)

---

## 1. Identidad mTLS del Dispositivo

Cada dispositivo tiene una identidad criptográfica única gestionada por la PKI de Vault (ADR-010).

### 1.1 Rutas de Sistema para Certificados

| Archivo | Contenido | Permisos |
|---|---|---|
| `/etc/agent/certs/device.crt` | Certificado X.509 del dispositivo (TTL 90 días) | `640` (agent:agent) |
| `/etc/agent/certs/device.key` | Clave privada RSA-2048 o ECDSA-P256 | `600` (agent:agent) |
| `/etc/agent/certs/ca.crt` | CA raíz de la PKI del proyecto | `644` (world-readable) |
| `/etc/agent/cosign.pub` | Clave pública cosign para verificación OTA | `644` |

**Provisioning inicial:** El fabricante instala un bootstrap token de un solo uso en `/etc/agent/bootstrap.token`. Al primer arranque, el agente llama a la API de Vault PKI para intercambiar el token por el certificado operativo (`device.crt` + `device.key`) y elimina el token inmediatamente.

**Rotación automática:** El agente compara la fecha de expiración de `device.crt` con el reloj del sistema. Si quedan menos de 15 días para el vencimiento, el agente solicita automáticamente un nuevo certificado a Vault PKI usando el certificado vigente para autenticar la solicitud (renovación). El proceso de renovación no interrumpe la operación del agente.

### 1.2 Referencia al Change `identidad-seguridad` / ADR-010

El protocolo completo de emisión, rotación y revocación de certificados de dispositivo está especificado en el change **`identidad-seguridad`** y en [ADR-010 de la propuesta principal](../propuesta-arquitectura-hurto-vehiculos.md#adr-010--pki-propia-con-certificados-cortos-para-dispositivos). Este documento se limita a las rutas de archivo y el comportamiento del agente ante certificados vencidos o revocados (CR-04).

### 1.3 Certificado Vencido o Revocado (CR-04)

Si el broker EMQX rechaza la conexión TLS por certificado inválido:

1. El [Uploader](./uploader.md) registra el error TLS con campo `"event": "mqtt_auth_failed"`.
2. La cola SQLite acumula eventos localmente.
3. El [Health Beacon](./health-beacon.md) incluye `a_cert: true` en su próximo payload.
4. **No se transmiten datos** hasta restaurar un certificado válido.
5. La restauración puede ocurrir por rotación automática (si el cert no fue revocado, solo vencido) o por intervención del operador via OTA que incluya el nuevo cert.

---

## 2. Restricciones de Escritura en Disco

El agente opera bajo el principio de mínimo privilegio en el filesystem:

| Directorio/Archivo | Tipo de acceso | Justificación |
|---|---|---|
| `/var/agent/` | Lectura y escritura (agent:agent) | Datos operacionales: SQLite, imágenes, Bloom filter, logs, auditoría |
| `/etc/agent/` | Solo lectura en operación normal | Configuración y certificados. Escritura solo durante provisioning y renovación de certs |
| `/usr/bin/agent` | Escritura atómica solo durante OTA | El Config/OTA Manager escribe el nuevo binario; en operación normal el binario es read-only |
| `/run/systemd/` | Escritura de socket `sd_notify` | Necesario para el Watchdog |
| Cualquier otro directorio | Sin acceso | `ProtectSystem=strict` en la unit de systemd |

La unit de systemd usa `ProtectSystem=strict` y `ReadWritePaths` explícitos, lo que garantiza que incluso si el agente es comprometido, el atacante no puede escribir fuera del árbol `/var/agent/`.

---

## 3. Verificación de Firma OTA

Cada binario descargado para actualización se verifica con cosign antes de reemplazar el binario activo (CA-09, CR-03). El flujo completo está en [Config/OTA Manager §4.3](./config-ota-manager.md#43-verificación-con-cosign).

**Controles en cadena:**

1. **Verificación de arquitectura:** El manifiesto OTA debe indicar el mismo `target_arch` que el binario activo.
2. **Verificación de hash SHA-256:** El hash del binario descargado debe coincidir con `sha256` del manifiesto.
3. **Verificación de firma cosign:** `cosign verify-blob` contra la clave pública embebida en el agente y distribuida en `/etc/agent/cosign.pub`.
4. Solo si los tres pasos son exitosos se procede al reemplazo atómico.

**Ante cualquier falla:** El archivo temporal se elimina, el binario activo no se toca, y se reporta `a_ota: true` en el Health Beacon.

---

## 4. Modo "Config Locked" ante Tampering

El agente activa el modo **"config locked"** ante indicios de manipulación física o lógica no autorizada:

### 4.1 Condiciones de Activación

| Condición | Detección |
|---|---|
| El archivo de configuración `/etc/agent/config.toml` fue modificado sin pasar por el Config/OTA Manager | Hash SHA-256 del archivo difiere del valor registrado al arrancar |
| El binario en ejecución difiere del hash almacenado en `/etc/agent/agent.sha256` | Verificación periódica (cada 5 minutos) |
| Tres intentos fallidos de aplicar OTA con firmas inválidas en menos de 1 hora | Contador en memoria |

### 4.2 Comportamiento en Modo Config Locked

1. El agente continúa capturando placas y persistiendo eventos en la cola SQLite (la captura no se detiene para no perder evidencia).
2. Se rechazan todos los mensajes de configuración y OTA entrantes.
3. El Health Beacon incluye `config_locked: true` en cada payload.
4. El log registra `"event": "config_locked_activated"` con la razón.
5. El modo se desactiva al reiniciar el proceso (por si fue una falsa alarma), pero si la condición persiste, se vuelve a activar al siguiente ciclo de verificación.

### 4.3 Notificación al Operador

El flag `config_locked: true` en el Health Beacon debe configurar una alerta de alta prioridad en el sistema de monitoreo cloud para que el equipo de operaciones investigue el dispositivo afectado.

---

## 5. Privacidad de Datos y Cumplimiento Normativo

### 5.1 Datos Almacenados en el Dispositivo

El agente almacena **únicamente** los siguientes datos:

| Dato | Dónde | Justificación |
|---|---|---|
| Número de matrícula (placa) | SQLite, payload JSON | Dato mínimo necesario para el propósito del sistema |
| Coordenadas GPS (lat, lon) | SQLite, payload JSON | Necesario para geolocalización del avistamiento |
| Timestamp de la captura | SQLite, payload JSON | Necesario para correlación temporal |
| Imagen del vehículo/matrícula | Filesystem `/var/agent/images/` | Evidencia probatoria; se sube al cloud y se rota localmente |
| Identificador del dispositivo | Configuración, payload JSON | Necesario para atribución del evento |
| Nivel de confianza del ANPR | SQLite, payload JSON | Metadato de calidad de la detección |

### 5.2 Datos Que NO Se Almacenan

Los siguientes datos **nunca** se almacenan ni transmiten por el agente (propuesta §6, supuesto 9):

| Dato | Razón de exclusión |
|---|---|
| Imágenes de rostros del conductor u ocupantes | Fuera de alcance; reconocimiento facial no es función del sistema |
| Datos biométricos de ningún tipo | Fuera de alcance y de cumplimiento normativo |
| Información personal del propietario del vehículo | El agente no tiene acceso a las bases policiales |
| DNI, nombre, dirección u otros datos personales | El agente solo captura evidencia de tránsito |
| Historial de navegación o datos del dispositivo móvil del conductor | No aplica al caso de uso |

### 5.3 Marco Normativo Aplicable

| Normativa | País | Aplicación al agente |
|---|---|---|
| Ley 1581 de 2012 (Protección de Datos Personales) | Colombia | Principio de finalidad (datos solo para combatir hurto), minimización (solo matrícula + coords + imagen vehículo), limitación de acceso (mTLS, ACL por país) |
| Ley Federal de Protección de Datos Personales en Posesión de los Particulares | México | Mismos principios de finalidad y minimización |
| Ley 29733 (Protección de Datos Personales) | Perú | Idem |
| GDPR (Reglamento General de Protección de Datos) | Unión Europea (si aplica) | Minimización, base legal (interés legítimo policial), retención limitada |

**Principio general:** La placa de un vehículo puede ser un dato personal (identifica indirectamente al propietario). Por esto, el agente aplica el principio de **minimización**: captura solo lo necesario para el propósito de detección de vehículos hurtados. El acceso a los datos capturados está restringido por mTLS, ACL por país en el broker MQTT, y RBAC en la plataforma cloud.

### 5.4 Retención Local

| Dato | Retención máxima local | Control |
|---|---|---|
| Eventos SQLite (`delivered`) | 24 horas (configurable) | Queue Manager limpieza periódica |
| Imágenes subidas (`uploaded/`) | 7 días o 10 GB (lo primero) | Image Store rotación |
| Imágenes no subidas (`pending/`) | Hasta capacidad de disco | Se eliminan solo cuando el disco está bajo |
| Log de auditoría de cambios de configuración | Sin rotación automática (máx. 10 MB, luego rota a `.1`) | Config/OTA Manager |

---

## 6. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [Uploader](./uploader.md) | Certificado mTLS para la sesión MQTT |
| [Config/OTA Manager](./config-ota-manager.md) | Verificación cosign de OTA; modo config locked |
| [Watchdog](./watchdog.md) | Reinicio ante tampering detectado |
| [Health Beacon](./health-beacon.md) | Reporta `a_cert`, `config_locked` |
| [Queue Manager](./queue-manager.md) | Retención de eventos; política de no-descarte de metadatos |
| [Image Store](./image-store.md) | Retención y rotación de imágenes; no almacena rostros |
| [ADRs Locales](./adr-local.md) | Decisiones de seguridad a nivel del agente |
| [Propuesta §5.3](../propuesta-arquitectura-hurto-vehiculos.md#53-seguridad) | Marco completo de seguridad de la plataforma |
| [Propuesta ADR-010](../propuesta-arquitectura-hurto-vehiculos.md#adr-010--pki-propia-con-certificados-cortos-para-dispositivos) | PKI propia con certificados cortos para dispositivos |
