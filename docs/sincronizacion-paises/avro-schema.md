# Esquema Avro del Evento Canónico

**Change:** `sincronizacion-paises`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Propósito

Este documento especifica el esquema Avro del evento canónico de vehículo hurtado, las reglas de compatibilidad en el Schema Registry, la política de versionado y el proceso de migración de adaptadores ante un cambio de versión major.

El esquema es la fuente de verdad para serialización/deserialización. Todo adaptador de país y todo consumidor de `stolen.vehicles.events` debe usar este esquema vía el Schema Registry.

---

## 2. Schema JSON (versión 1.0)

```json
{
  "type": "record",
  "name": "StolenVehicleEvent",
  "namespace": "com.antihurto.vehicles.canonical.v1",
  "doc": "Evento canónico de vehículo hurtado producido por un adaptador de país.",
  "fields": [
    {
      "name": "event_id",
      "type": "string",
      "doc": "UUID v4 único por evento. Permite idempotencia end-to-end."
    },
    {
      "name": "schema_version",
      "type": "string",
      "doc": "Versión del esquema canónico con la que fue producido el evento (e.g., '1.0')."
    },
    {
      "name": "country_code",
      "type": "string",
      "doc": "Código ISO 3166-1 alpha-2 del país. Discriminador de tenant."
    },
    {
      "name": "plate",
      "type": "string",
      "doc": "Matrícula normalizada a mayúsculas sin espacios ni guiones."
    },
    {
      "name": "status",
      "type": {
        "type": "enum",
        "name": "VehicleStatus",
        "symbols": ["STOLEN", "RECOVERED"],
        "doc": "Estado del registro. El adaptador siempre produce STOLEN; RECOVERED lo produce el incident-service."
      },
      "doc": "Estado actual del vehículo en el sistema canónico."
    },
    {
      "name": "stolen_date",
      "type": "string",
      "doc": "Fecha y hora del hurto en formato ISO 8601 UTC (YYYY-MM-DDTHH:mm:ssZ)."
    },
    {
      "name": "stolen_location",
      "type": "string",
      "doc": "Descripción textual del lugar del hurto. Máximo 500 caracteres."
    },
    {
      "name": "owner_id",
      "type": "string",
      "doc": "Identificador del propietario en el sistema del país. No se expone en Redis ni en BF."
    },
    {
      "name": "brand",
      "type": "string",
      "doc": "Marca del vehículo (e.g., TOYOTA, CHEVROLET)."
    },
    {
      "name": "class",
      "type": "string",
      "doc": "Clase del vehículo según clasificación policial del país (e.g., AUTOMOVIL, MOTOCICLETA)."
    },
    {
      "name": "line",
      "type": "string",
      "doc": "Línea o modelo de carrocería (e.g., COROLLA, SPARK)."
    },
    {
      "name": "color",
      "type": "string",
      "doc": "Color principal del vehículo."
    },
    {
      "name": "model",
      "type": "string",
      "doc": "Año modelo del vehículo como cadena de 4 dígitos (e.g., '2019')."
    },
    {
      "name": "source_system",
      "type": "string",
      "doc": "Identificador del sistema fuente (e.g., SIJIN_CO, CICPC_VE)."
    },
    {
      "name": "ingested_at",
      "type": "string",
      "doc": "Timestamp ISO 8601 UTC de cuando el adaptador produjo el mensaje."
    },
    {
      "name": "extensions",
      "type": {
        "type": "map",
        "values": "string"
      },
      "default": {},
      "doc": "Campos adicionales autorizados por el país. Clave: nombre del campo. Valor: representación string. No requiere cambio de versión al agregar nuevas claves."
    }
  ]
}
```

---

## 3. Registro en el Schema Registry

### 3.1 Subject de registro

El schema se registra bajo el subject:

```
stolen.vehicles.events-value
```

Convención: `{topic}-value` para el value, `{topic}-key` para la clave (la clave es un string simple `{country_code}:{plate}` y no requiere schema Avro por ahora).

### 3.2 Compatibilidad

El Schema Registry se configura con compatibilidad `BACKWARD` para el subject `stolen.vehicles.events-value`.

**BACKWARD** significa: los consumers que usan la versión anterior del schema pueden leer mensajes producidos con la versión nueva. Esto garantiza que al actualizar los productores (adaptadores), los consumers (Canonical Vehicles Service) no requieren actualización simultánea.

```bash
# Configurar compatibilidad (Confluent Schema Registry)
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "BACKWARD"}' \
  http://schema-registry:8081/config/stolen.vehicles.events-value

# Verificar
curl http://schema-registry:8081/config/stolen.vehicles.events-value
```

### 3.3 Registro inicial

```bash
# Registrar la versión 1.0 del schema
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data @canonical-vehicle-event-v1.avsc \
  http://schema-registry:8081/subjects/stolen.vehicles.events-value/versions
```

---

## 4. Política de Versionado

### 4.1 Versión minor (cambios compatibles)

Constituyen versión minor los siguientes cambios, permitidos bajo compatibilidad `BACKWARD`:

| Cambio | Ejemplo | Acción requerida |
|---|---|---|
| Agregar campo nuevo opcional (con `default`) | Agregar `recovery_date` con `default: null` | Registrar nueva versión en Schema Registry. Adaptadores existentes pueden ignorar el campo nuevo. |
| Agregar símbolo a un enum existente | Agregar `PENDING` a `VehicleStatus` | Registrar nueva versión. Consumers con schema anterior deben manejar el valor desconocido (recomendar ignorar o enrutar a DLQ). |
| Agregar clave nueva en `extensions` | Agregar `numero_motor` | No requiere cambio de schema. El mapa `extensions` acepta nuevas claves sin modificación. |
| Ampliar `doc` sin cambios funcionales | Clarificar descripción de un campo | Registrar nueva versión (cosmético). |

El número de versión interna del Schema Registry se incrementa automáticamente. El campo `schema_version` en el mensaje pasa de `1.0` a `1.1`, `1.2`, etc.

### 4.2 Versión major (cambios incompatibles)

Constituyen versión major los siguientes cambios:

| Cambio | Ejemplo | Por qué incompatible |
|---|---|---|
| Eliminar un campo mandatorio | Eliminar `owner_id` | Consumers que esperan el campo fallarían la deserialización. |
| Cambiar el tipo de un campo | `model` de `string` a `int` | Incompatible con consumers que usan el tipo anterior. |
| Renombrar un campo | `plate` → `license_plate` | El campo antiguo desaparecería; consumers lo leen como ausente. |
| Eliminar símbolo de un enum | Eliminar `RECOVERED` de `VehicleStatus` | Mensajes con ese valor no serían deserializables. |
| Cambiar el namespace | `v1` → `v2` | Rompe la identidad del record para todos los consumers. |

### 4.3 Proceso de migración ante versión major

Ver [`schema-evolution.md`](./schema-evolution.md) para el proceso completo. Resumen:

1. Crear el nuevo namespace `com.antihurto.vehicles.canonical.v2`.
2. Registrar el nuevo schema bajo `stolen.vehicles.events-v2-value`.
3. Migrar los adaptadores uno a uno para publicar en el nuevo tópico.
4. Mantener el tópico v1 activo durante el período de migración (mínimo 30 días).
5. Migrar el Canonical Vehicles Service para consumir ambos tópicos.
6. Deprecar el tópico v1 y dar aviso formal con 30 días de anticipación.
7. Eliminar el tópico v1 tras confirmación de todos los adaptadores migrados.

---

## 5. Uso en Productores (Adaptadores)

### 5.1 Dependencia (Java / Kafka Clients)

```xml
<dependency>
  <groupId>io.confluent</groupId>
  <artifactId>kafka-avro-serializer</artifactId>
  <version>7.6.0</version>
</dependency>
```

### 5.2 Configuración del producer

```properties
bootstrap.servers=kafka:9092
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=io.confluent.kafka.serializers.KafkaAvroSerializer
schema.registry.url=http://schema-registry:8081
auto.register.schemas=false
use.latest.version=true
```

`auto.register.schemas=false` es obligatorio en producción. El registro de nuevas versiones es un proceso controlado; no se permite que un adaptador registre esquemas de forma autónoma.

### 5.3 Configuración del producer (Go — goavro)

```go
// Usar goavro + franz-go o confluent-kafka-go con schema registry client
schemaRegistryClient := srclient.CreateSchemaRegistryClient("http://schema-registry:8081")
schema, err := schemaRegistryClient.GetLatestSchema("stolen.vehicles.events-value")
// ... serializar con goavro.NewCodec(schema.Schema())
```

---

## 6. Uso en Consumidores (Canonical Vehicles Service)

### 6.1 Configuración del consumer

```properties
bootstrap.servers=kafka:9092
group.id=canonical-vehicles-service
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=io.confluent.kafka.serializers.KafkaAvroDeserializer
schema.registry.url=http://schema-registry:8081
specific.avro.reader=true
```

### 6.2 Manejo de errores de deserialización

Si la deserialización falla (esquema desconocido, versión incompatible):
1. El consumer captura la excepción `SerializationException`.
2. Publica el mensaje crudo en `stolen.vehicles.events.dlq` con header `error_reason=SCHEMA_DESERIALIZATION_FAILURE`.
3. Continúa procesando el siguiente mensaje (no detiene el consumo).

---

## 7. Evolución del campo `extensions`

El campo `extensions` es un `map<string, string>`. Esta restricción (valores siempre string) es intencional:

- Simplifica la deserialización en todos los lenguajes.
- Evita anidamiento profundo que dificulta la inspección en Kafka UI.
- Los valores numéricos o booleanos se representan como strings y los adaptadores los convierten al deserializar.

Si un país necesita valores estructurados complejos (e.g., un array de imágenes adicionales), se evalúa promover el campo al esquema canónico como tipo Avro apropiado en la próxima versión minor.

---

## 8. Compatibilidad con Apicurio Registry

La especificación aplica tanto a Confluent Schema Registry como a Apicurio Registry (alternativa OSS cloud-agnostic). Las diferencias operacionales:

| Aspecto | Confluent Schema Registry | Apicurio Registry |
|---|---|---|
| Licencia | Community (BSL para algunas features) | Apache 2.0 |
| API de compatibilidad | `/compatibility/subjects/{subject}/versions` | `/apis/ccompat/v7/compatibility/subjects/{subject}/versions` |
| UI | Control Center (comercial) o AKHQ (OSS) | Apicurio Studio (incluido) |
| Autenticación | Basic / mTLS | Keycloak OIDC nativo |
| Despliegue on-prem | Requiere licencia para algunas features HA | Completamente OSS |

Para despliegues on-prem del adaptador se recomienda Apicurio por su licencia Apache 2.0 y su integración nativa con Keycloak.

---

## 9. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [`canonical-model.md`](./canonical-model.md) | Definición semántica de cada campo |
| [`kafka-topics.md`](./kafka-topics.md) | Tópicos donde se publica este schema |
| [`schema-evolution.md`](./schema-evolution.md) | Proceso completo de evolución de versiones |
| [`country-adapter-framework.md`](./country-adapter-framework.md) | Responsabilidades del adaptador al serializar |
