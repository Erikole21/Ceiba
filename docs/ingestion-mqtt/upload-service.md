# Upload Service — Especificación

**Pilar:** 2 — Comunicación Segura y Eficiente  
**ADRs aplicados:** ADR-003 · ADR-005 · ADR-010 · ADR-011  
**Referencia:** [Visión General](./overview.md) · [object-storage.md](./object-storage.md) · [security.md](./security.md)  
**Upstream:** [`docs/agente-borde/uploader.md`](../agente-borde/uploader.md)  
**Última actualización:** 2026-05-13

---

## 1. Propósito

El Upload Service es el servicio cloud que habilita la subida de imágenes desde los dispositivos de borde al object storage, sin transitar por el broker MQTT. Implementa dos modos de subida (ADR-003):

1. **Presigned URL (modo directo):** el servicio emite una URL pre-firmada con TTL de 5 minutos; el dispositivo realiza el `PUT` directamente al object storage (CA-08, CA-09).
2. **Proxy-upload (modo indirecto):** para dispositivos detrás de NAT o APN restrictivo, el servicio actúa como proxy: recibe el binario y lo escribe al object storage en nombre del dispositivo (CA-10).

La identidad del solicitante se valida en ambos modos mediante el certificado mTLS del dispositivo (CN/SAN) o un JWT Bearer emitido por Vault.

---

## 2. API REST

### 2.1 Resumen de Endpoints

| Método | Ruta | Descripción | Auth requerida |
|---|---|---|---|
| `POST` | `/v1/presign` | Emite URL pre-firmada para PUT directo al object storage | mTLS cert o JWT |
| `POST` | `/v1/upload` | Proxy-upload: recibe imagen y la escribe al object storage | mTLS cert o JWT |
| `GET` | `/v1/health` | Health check del servicio | Ninguna (endpoint público) |

**Base URL:** `https://upload.ceiba-antihurto.io`  
**TLS:** TLS 1.3 mínimo. El endpoint acepta certificado de cliente (mTLS) pero no lo exige en el TLS handshake — la validación de identidad ocurre en la capa de aplicación con el header `Authorization` o con el certificado de cliente presentado.

---

### 2.2 POST /v1/presign

Emite una URL pre-firmada de tipo PUT para subida directa al object storage. Este endpoint es el canónico para la integración con el agente de borde (ver [`docs/agente-borde/uploader.md §4`](../agente-borde/uploader.md)).

**Request:**

```http
POST /v1/presign HTTP/1.1
Host: upload.ceiba-antihurto.io
Authorization: Bearer <device_jwt>
Content-Type: application/json

{
  "device_id":        "CO-BOG-DEV-00142",
  "event_id":         "550e8400-e29b-41d4-a716-446655440000",
  "content_type":     "image/jpeg",
  "file_size_bytes":  184320
}
```

**Parámetros del body:**

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `device_id` | string | Sí | Identificador del dispositivo. Debe coincidir con el CN/SAN del certificado o el claim `sub` del JWT. |
| `event_id` | string (UUID v4) | Sí | Identificador del evento al que pertenece la imagen. Usado para construir el path en el bucket. |
| `content_type` | string | Sí | MIME type de la imagen. Valores aceptados: `image/jpeg`, `image/png`. |
| `file_size_bytes` | integer | No | Tamaño declarado de la imagen en bytes. Usado para validación de límite (máx. 524288 = 512 KB). |

**Response 200 OK:**

```json
{
  "upload_url":  "https://minio.ceiba-antihurto.io/ceiba-evidence-co/images/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Signature=...",
  "image_uri":   "s3://ceiba-evidence-co/images/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000.jpg",
  "ttl_s":       300
}
```

| Campo | Descripción |
|---|---|
| `upload_url` | URL pre-firmada para HTTP PUT. Válida exactamente 300 segundos desde la emisión (CA-08, CR-01). |
| `image_uri` | URI canónica del objeto en el storage. Incluir en el payload MQTT `devices/{device_id}/image-uri`. |
| `ttl_s` | TTL de la URL en segundos. Siempre 300. |

**Códigos de error:**

| Código HTTP | Causa |
|---|---|
| `400 Bad Request` | Body JSON inválido, campos faltantes, `content_type` no soportado, `file_size_bytes` > 512 KB |
| `401 Unauthorized` | Sin credencial o credencial inválida (CR-02) |
| `403 Forbidden` | `device_id` en el body no coincide con la identidad validada |
| `503 Service Unavailable` | Object storage no disponible (CR-08); incluye header `Retry-After: 30` |

---

### 2.3 POST /v1/upload

Modo proxy: el servicio recibe el binario de la imagen y la escribe al object storage en nombre del dispositivo. Para dispositivos que no pueden alcanzar el object storage directamente (CA-10).

**Request (multipart/form-data):**

```http
POST /v1/upload HTTP/1.1
Host: upload.ceiba-antihurto.io
Authorization: Bearer <device_jwt>
Content-Type: multipart/form-data; boundary=----boundary123

------boundary123
Content-Disposition: form-data; name="metadata"
Content-Type: application/json

{
  "device_id":    "CO-BOG-DEV-00142",
  "event_id":     "550e8400-e29b-41d4-a716-446655440000",
  "content_type": "image/jpeg"
}
------boundary123
Content-Disposition: form-data; name="image"; filename="image.jpg"
Content-Type: image/jpeg

<binary image data>
------boundary123--
```

**Request alternativa (application/octet-stream con headers):**

```http
POST /v1/upload HTTP/1.1
Host: upload.ceiba-antihurto.io
Authorization: Bearer <device_jwt>
Content-Type: image/jpeg
X-Device-Id: CO-BOG-DEV-00142
X-Event-Id: 550e8400-e29b-41d4-a716-446655440000
Content-Length: 184320

<binary image data>
```

**Response 201 Created:**

```json
{
  "image_uri": "s3://ceiba-evidence-co/images/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000.jpg",
  "size_bytes": 184320
}
```

**Comportamiento interno del proxy-upload:**
1. El servicio recibe el binario via stream (no carga completo en memoria).
2. Genera el path del objeto: `images/{country_code}/{device_id}/{date}/{image_id}`.
3. Llama al `ObjectStoragePort.PutObject(path, stream, sse-kms)`.
4. Si el object storage no está disponible: retorna `503` con `Retry-After: 30` y no almacena datos parciales (CR-08).

**Códigos de error:** mismos que `/v1/presign` más `413 Payload Too Large` si la imagen supera 512 KB.

---

### 2.4 GET /v1/health

```http
GET /v1/health HTTP/1.1
Host: upload.ceiba-antihurto.io
```

**Response 200 OK:**

```json
{
  "status":    "ok",
  "storage":   "ok",
  "vault":     "ok",
  "timestamp": "2026-05-13T14:32:00Z"
}
```

**Response 503 Service Unavailable** (si algún dependency está caído):

```json
{
  "status":  "degraded",
  "storage": "unavailable",
  "vault":   "ok",
  "timestamp": "2026-05-13T14:32:00Z"
}
```

---

## 3. Flujo de Validación de Identidad

El servicio admite dos mecanismos de autenticación de dispositivo:

### 3.1 mTLS — Certificado de Dispositivo

El cliente presenta su certificado X.509 durante el TLS handshake. El servicio:

1. Verifica la cadena de confianza del certificado contra la CA raíz de Vault PKI del realm del país.
2. Extrae `device_id` del campo `Subject.CN` del certificado.
3. Extrae `country_code` de las 2 primeras letras del `device_id` o del SAN custom `country_code` si está presente.
4. Verifica que el `device_id` en el body/header coincida con el extraído del certificado.
5. Consulta OCSP (Vault PKI) para verificar revocación. Si OCSP no está disponible y el certificado no ha expirado, permite la operación (comportamiento consistente con emqx-cluster).

### 3.2 JWT Bearer

El cliente presenta un JWT emitido por Vault (Vault Auth — AppRole o similar) con claims:
- `sub`: `device_id` del dispositivo
- `country_code`: código ISO del país
- `exp`: timestamp de expiración

El servicio verifica la firma del JWT contra la clave pública de Vault (JWKS endpoint), valida `exp` y extrae los claims.

### 3.3 Rechazo sin Credencial

Si el request no presenta certificado mTLS ni JWT Bearer válido, el servicio retorna `401 Unauthorized` sin emitir ninguna URL y registra el intento fallido en el log de auditoría (CR-02).

---

## 4. Lógica de Generación de URL Pre-firmada

### 4.1 Construcción del Path del Objeto

```
images/{country_code}/{device_id}/{date}/{image_id}
```

| Segmento | Valor | Ejemplo |
|---|---|---|
| `images/` | Prefijo fijo del bucket | `images/` |
| `{country_code}` | Código ISO del país extraído del certificado/JWT | `CO` |
| `{device_id}` | Identificador del dispositivo | `CO-BOG-DEV-00142` |
| `{date}` | Fecha UTC del evento en formato `YYYY-MM-DD` | `2026-05-13` |
| `{image_id}` | UUID v4 del `event_id` con extensión `.jpg` o `.png` | `550e8400-e29b-41d4-a716-446655440000.jpg` |

**Path completo de ejemplo:**
```
images/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000.jpg
```

### 4.2 Parámetros de la URL Pre-firmada

| Parámetro | Valor |
|---|---|
| Método HTTP | PUT |
| TTL | 300 segundos exactos (CA-08) |
| Restricción de Content-Type | Debe coincidir con el declarado en el request |
| Server-side encryption | SSE-KMS obligatorio (el bucket rechaza PUTs sin este header) |

### 4.3 Expiración

La URL expira exactamente 300 segundos después de su emisión. Un PUT realizado después de la expiración recibe `403 Forbidden` del object storage (CR-01). El dispositivo debe solicitar una nueva URL al `/v1/presign`.

---

## 5. Puerto Hexagonal: ObjectStoragePort (ADR-005)

El Upload Service accede al object storage únicamente a través de la interfaz `ObjectStoragePort`. Esto aísla la lógica de negocio del proveedor de storage concreto.

### 5.1 Interfaz

```go
// ObjectStoragePort define las operaciones de object storage requeridas por el Upload Service.
type ObjectStoragePort interface {
    // GeneratePresignedPutURL genera una URL pre-firmada para PUT con TTL y path dados.
    GeneratePresignedPutURL(ctx context.Context, req PresignRequest) (PresignResponse, error)

    // PutObject sube un objeto al storage vía stream. Usado en modo proxy-upload.
    PutObject(ctx context.Context, req PutObjectRequest, body io.Reader) error

    // HealthCheck verifica la conectividad con el backend de storage.
    HealthCheck(ctx context.Context) error
}

type PresignRequest struct {
    BucketName  string
    ObjectKey   string        // path completo: images/{cc}/{device_id}/{date}/{id}.jpg
    ContentType string
    TTL         time.Duration // siempre 5 * time.Minute
    SSEKMSKeyID string        // ARN o alias de la clave KMS
}

type PresignResponse struct {
    URL      string
    ImageURI string        // URI canónica s3://bucket/key
    ExpiresAt time.Time
}

type PutObjectRequest struct {
    BucketName  string
    ObjectKey   string
    ContentType string
    SizeBytes   int64
    SSEKMSKeyID string
}
```

### 5.2 Implementaciones de Adaptador

| Adaptador | Proveedor | Notas de Compatibilidad |
|---|---|---|
| `MinIOAdapter` | MinIO (on-prem / cloud) | API S3 completa. Presigned URLs idénticas a AWS S3. SSE-KMS con Vault Transit Engine o KES. |
| `AWSS3Adapter` | AWS S3 | Implementación de referencia. Presigned URLs nativas v4. SSE-KMS con AWS KMS. |
| `GCSAdapter` | Google Cloud Storage | Interoperabilidad S3 via XML API habilitada. Presigned URLs con firma HMAC. SSE con Cloud KMS. Nota: la interoperabilidad S3 no soporta todas las características; verificar compatibilidad de SSE-KMS header. |
| `AzureBlobAdapter` | Azure Blob Storage | Interoperabilidad S3 via Azure Blob NFS o Blobfuse. SAS tokens como equivalente de presigned URLs. Cifrado server-side con Azure Key Vault. Nota: la firma de SAS difiere; el adaptador abstrae la diferencia de forma transparente para el servicio. |

El adaptador activo se selecciona vía variable de entorno `UPLOAD_SERVICE_STORAGE_BACKEND` (valores: `minio`, `s3`, `gcs`, `azure`).

---

## 6. Manejo de Errores y Códigos HTTP

| Escenario | Código HTTP | Header adicional |
|---|---|---|
| Request exitoso (presign) | `200 OK` | — |
| Request exitoso (proxy-upload) | `201 Created` | — |
| Body inválido o campo faltante | `400 Bad Request` | — |
| Sin credencial o credencial expirada | `401 Unauthorized` | `WWW-Authenticate: Bearer` |
| device_id del body no coincide con identidad | `403 Forbidden` | — |
| Certificado revocado | `403 Forbidden` | — |
| Imagen excede 512 KB | `413 Payload Too Large` | — |
| Content-Type no soportado | `415 Unsupported Media Type` | — |
| Object storage no disponible (CR-08) | `503 Service Unavailable` | `Retry-After: 30` |
| Error interno (bug, panic) | `500 Internal Server Error` | No expone detalle interno |

**Regla de privacidad de errores:** los mensajes de error retornados al cliente son genéricos. Los detalles internos (stack traces, nombres de infraestructura) solo aparecen en los logs estructurados internos.

---

## 7. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [object-storage.md](./object-storage.md) | Bucket, lifecycle policy, SSE-KMS — configuración del backend del ObjectStoragePort |
| [security.md](./security.md) | Flujo mTLS completo y validación OCSP aplicada en este servicio |
| [helm/README.md](./helm/README.md) | Despliegue Kubernetes del Upload Service |
| [terraform/README.md](./terraform/README.md) | Módulo Terraform `upload-service` |
| [`docs/agente-borde/uploader.md`](../agente-borde/uploader.md) | Cliente del servicio: request a `/v1/presign` y PUT al object storage |
| [`docs/identidad-seguridad/vault-pki-device.md`](../identidad-seguridad/vault-pki-device.md) | Emisión de certificados mTLS validados por este servicio |
