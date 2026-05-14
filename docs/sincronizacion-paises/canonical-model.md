# Modelo Canónico de Vehículo Hurtado

**Change:** `sincronizacion-paises`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Propósito

Este documento es la **especificación autoritativa** del modelo canónico de vehículo hurtado. Define los campos obligatorios, los campos de control del sistema, las reglas de validación y la política de extensiones por país. Todo adaptador de país y todo consumidor del tópico `stolen.vehicles.events` debe conformarse a esta especificación.

---

## 2. Campos Mandatorios (9 campos del caso de estudio)

Los nueve campos mandatorios son invariantes del contrato. Ningún adaptador puede omitirlos; ninguna versión del esquema puede eliminarlos ni cambiar su semántica sin una versión major (ver [`schema-evolution.md`](./schema-evolution.md)).

| # | Campo | Tipo | Descripción | Reglas de validación |
|---|---|---|---|---|
| 1 | `plate` | `string` | Matrícula del vehículo tal como aparece en el registro policial del país. | No nulo, no vacío. Longitud máxima 20 caracteres. El adaptador normaliza a mayúsculas sin espacios ni guiones (e.g., `ABC123`). |
| 2 | `stolen_date` | `string` (ISO 8601 UTC) | Fecha y hora en que se registró el hurto en la BD policial. | Formato `YYYY-MM-DDTHH:mm:ssZ`. No nulo. No puede ser futura (> now + 24h). |
| 3 | `stolen_location` | `string` | Descripción textual del lugar del hurto (dirección, barrio, ciudad, país). | No nulo, no vacío. Longitud máxima 500 caracteres. |
| 4 | `owner_id` | `string` | Identificador del propietario en el sistema del país (número de documento, pasaporte u equivalente). | No nulo. Longitud 1–50 caracteres. **No se publica directamente en Redis ni en el Bloom filter** — solo persiste en PostgreSQL bajo control de acceso. |
| 5 | `brand` | `string` | Marca del vehículo (e.g., `TOYOTA`, `CHEVROLET`). | No nulo, no vacío. Longitud máxima 100 caracteres. |
| 6 | `class` | `string` | Clase del vehículo según la clasificación policial del país (e.g., `AUTOMOVIL`, `MOTOCICLETA`, `CAMION`). | No nulo, no vacío. Longitud máxima 50 caracteres. |
| 7 | `line` | `string` | Línea o modelo de carrocería (e.g., `COROLLA`, `SPARK`). | No nulo, no vacío. Longitud máxima 100 caracteres. |
| 8 | `color` | `string` | Color principal del vehículo según el registro del hurto. | No nulo, no vacío. Longitud máxima 50 caracteres. |
| 9 | `model` | `string` | Año modelo del vehículo (e.g., `2019`). | No nulo. Formato: cadena de 4 dígitos representando el año. Rango válido: `1900`–`{año_actual + 1}`. |

---

## 3. Campos de Control del Sistema

Estos campos son generados o validados por el adaptador y el Canonical Vehicles Service. No provienen directamente de la BD policial, sino del sistema.

| Campo | Tipo | Descripción | Quién lo establece |
|---|---|---|---|
| `country_code` | `string` (ISO 3166-1 alpha-2) | Código ISO del país al que pertenece el registro. Actúa como **discriminador de tenant** en todos los almacenes. | Configuración del adaptador. Inmutable para cada instancia de adaptador. |
| `status` | `enum` | Estado del registro: `STOLEN` o `RECOVERED`. | Adaptador al ingerir (siempre `STOLEN`); `incident-service` al producir evento de recuperación. |
| `schema_version` | `string` | Versión del esquema canónico con el que fue producido el mensaje (e.g., `1.2`). | Adaptador al serializar; validado por el Schema Registry. |
| `event_id` | `string` (UUID v4) | Identificador único del evento. Permite idempotencia end-to-end. | Adaptador en cada publicación (nuevo UUID por publicación). |
| `source_system` | `string` | Identificador del sistema fuente tal como lo conoce el adaptador (e.g., `SIJIN_CO`, `CICPC_VE`). | Configuración del adaptador. |
| `ingested_at` | `string` (ISO 8601 UTC) | Timestamp de cuando el adaptador produjo el mensaje. | Adaptador al publicar. |

---

## 4. Campo `extensions` (JSONB)

El campo `extensions` es un objeto JSON abierto que permite a cada país transmitir campos adicionales no contemplados en el modelo canónico sin requerir una nueva versión mayor del esquema.

### 4.1 Política de uso

1. **Solo campos autorizados.** El operador del país debe listar explícitamente en la configuración del adaptador los campos que tienen autorización institucional para salir del territorio. Los demás campos de la BD policial no deben incluirse en `extensions` ni en ningún otro campo del evento.
2. **Sin PII no autorizada.** Los campos que contengan datos personales (nombre del propietario, número de documento adicional, dirección domiciliaria) requieren autorización explícita bajo las leyes de protección de datos del país y del RGPD/equivalente aplicable.
3. **Documentación obligatoria.** Cada país que use `extensions` debe documentar sus campos en su ficha de onboarding (ver [`country-onboarding-guide.md`](./country-onboarding-guide.md)).
4. **No afecta la versión del esquema.** El uso de nuevas claves en `extensions` no constituye un cambio de versión del esquema Avro. Es un mecanismo de escape controlado.
5. **Promoción a canónico.** Si un campo de `extensions` resulta ser de utilidad general para múltiples países, puede promoverse a campo canónico en una versión minor o major del esquema (ver [`schema-evolution.md`](./schema-evolution.md)).

### 4.2 Ejemplo

```json
{
  "plate": "ABC123",
  "country_code": "CO",
  "status": "STOLEN",
  "stolen_date": "2026-01-15T14:30:00Z",
  "stolen_location": "Cra 7 con Calle 32, Bogotá, Colombia",
  "owner_id": "1020304050",
  "brand": "CHEVROLET",
  "class": "AUTOMOVIL",
  "line": "SPARK",
  "color": "ROJO",
  "model": "2020",
  "schema_version": "1.0",
  "event_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "source_system": "SIJIN_CO",
  "ingested_at": "2026-01-15T14:31:05Z",
  "extensions": {
    "numero_motor": "4A1234567",
    "numero_chasis": "9BWZZZ377VT004251",
    "tipo_combustible": "GASOLINA"
  }
}
```

---

## 5. Reglas de Validación Completas

### 5.1 Validaciones en el adaptador (antes de publicar)

El adaptador debe validar antes de serializar y publicar:

- Todos los campos mandatorios presentes y no nulos.
- `country_code` coincide con el código configurado para la instancia del adaptador.
- `status` es `STOLEN` (el adaptador nunca produce `RECOVERED`; eso es responsabilidad del `incident-service`).
- `stolen_date` parseable como ISO 8601 UTC y no futura.
- `model` cuatro dígitos en rango válido.
- `schema_version` corresponde a la versión activa registrada en el Schema Registry.

Si falla cualquier validación, el adaptador:
1. Registra el error con nivel `WARN` incluyendo el campo fallido y el valor recibido.
2. Incrementa la métrica `adapter.validation.failures{country_code, field}`.
3. **No publica** el mensaje en Kafka.
4. Opcionalmente escribe el registro original en el almacén de archivado local para auditoría.

### 5.2 Validaciones en el Canonical Vehicles Service (al consumir)

El servicio revalida al consumir:

- Deserialización Avro exitosa contra el Schema Registry.
- `country_code` registrado en la configuración del sistema.
- Campos mandatorios presentes y con tipos correctos.
- `event_id` no procesado previamente (ventana de deduplicación configurable, valor de referencia 24 h).

Si falla la validación:
- El mensaje se redirige a `stolen.vehicles.events.dlq` con el código de error y el campo fallido.
- Se incrementa la métrica correspondiente.
- No se modifica PostgreSQL ni Redis.

---

## 6. Invariantes del Modelo Canónico

Las siguientes reglas son **invariantes** que no pueden violarse en ninguna versión del esquema:

1. `(country_code, plate)` es la clave de identidad de un vehículo. Un mismo vehículo puede tener registros en varios países solo si sus matrículas difieren (lo cual es el caso normal dado que las matrículas son nacionales).
2. El campo `owner_id` **nunca aparece** en la clave Redis (`stolen:{cc}:{plate}`), en el Bloom filter ni en ningún tópico downstream de analítica. Solo persiste en `canonical_vehicles` en PostgreSQL bajo control de acceso RBAC.
3. El campo `status` solo puede tener los valores `STOLEN` o `RECOVERED`. No existe un estado intermedio canónico.
4. El Bloom filter solo incluye el par `{country_code}:{plate}` de registros con `status = STOLEN`. Los registros `RECOVERED` deben ser eliminados del BF en el siguiente delta.

---

## 7. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [`avro-schema.md`](./avro-schema.md) | Serialización Avro de este modelo |
| [`kafka-topics.md`](./kafka-topics.md) | Tópicos donde viajan los eventos canónicos |
| [`postgresql-schema.md`](./postgresql-schema.md) | DDL de la tabla `canonical_vehicles` |
| [`country-adapter-framework.md`](./country-adapter-framework.md) | Responsabilidades del adaptador en la validación y mapeo |
| [`schema-evolution.md`](./schema-evolution.md) | Proceso de evolución de este modelo |
| [`data-sovereignty.md`](./data-sovereignty.md) | Política sobre qué campos pueden salir del territorio |
