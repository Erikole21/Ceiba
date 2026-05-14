# Object Storage — Configuración

**Pilar:** 2 — Comunicación Segura y Eficiente  
**ADRs aplicados:** ADR-003 · ADR-005 · ADR-011  
**Referencia:** [Visión General](./overview.md) · [upload-service.md](./upload-service.md) · [security.md](./security.md)  
**Última actualización:** 2026-05-13

---

## 1. Estructura de Paths del Bucket

El sistema utiliza un único bucket lógico por país (o un bucket global con prefijos por país, según el proveedor). La estructura de paths dentro del bucket es la siguiente:

### 1.1 Imágenes Completas

```
images/{country_code}/{device_id}/{date}/{image_id}.jpg
```

**Ejemplo:**
```
images/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000.jpg
```

| Segmento | Descripción | Ejemplo |
|---|---|---|
| `images/` | Prefijo que distingue imágenes completas de thumbnails | — |
| `{country_code}` | Aislamiento de tenant por país (ADR-011) | `CO` |
| `{device_id}` | Identificador del dispositivo que capturó la imagen | `CO-BOG-DEV-00142` |
| `{date}` | Fecha UTC del evento (`YYYY-MM-DD`). Facilita lifecycle policy por fecha | `2026-05-13` |
| `{image_id}` | UUID v4 del `event_id` con extensión de formato | `550e8400...0000.jpg` |

### 1.2 Miniaturas (Thumbnails)

```
thumbnails/{country_code}/{device_id}/{date}/{image_id}-thumb.jpg
```

**Ejemplo:**
```
thumbnails/CO/CO-BOG-DEV-00142/2026-05-13/550e8400-e29b-41d4-a716-446655440000-thumb.jpg
```

Las miniaturas se generan por un proceso separado (Thumbnail Service, fuera del alcance de este pilar) tras la subida de la imagen completa. El path sigue la misma estructura con el prefijo `thumbnails/` y el sufijo `-thumb`.

### 1.3 Consideraciones de Prefijado Multi-tenant

El prefijo `{country_code}` implementa el aislamiento lógico de tenant en object storage (ADR-011). Las políticas de bucket, las claves KMS y las lifecycle rules pueden aplicarse por prefijo. En despliegues con requisitos de residencia de datos, cada país puede tener su propio bucket en la región correspondiente.

---

## 2. Lifecycle Policy

### 2.1 Política de Imágenes Completas

Las imágenes completas se eliminan automáticamente 180 días (6 meses) después de su creación (CA-11, Supuesto 10 de la propuesta).

**Regla S3-API:**

```xml
<Rule>
  <ID>expire-full-images</ID>
  <Status>Enabled</Status>
  <Filter>
    <Prefix>images/</Prefix>
  </Filter>
  <Expiration>
    <Days>180</Days>
  </Expiration>
</Rule>
```

**Equivalente en HCL Terraform (aws_s3_bucket_lifecycle_configuration):**

```hcl
rule {
  id     = "expire-full-images"
  status = "Enabled"

  filter {
    prefix = "images/"
  }

  expiration {
    days = 180
  }
}
```

### 2.2 Política de Miniaturas

Las miniaturas se conservan 730 días (24 meses) para soportar búsquedas históricas y dashboards analíticos (CA-11).

```xml
<Rule>
  <ID>expire-thumbnails</ID>
  <Status>Enabled</Status>
  <Filter>
    <Prefix>thumbnails/</Prefix>
  </Filter>
  <Expiration>
    <Days>730</Days>
  </Expiration>
</Rule>
```

### 2.3 Limpieza de Multipart Uploads Incompletos

Los multipart uploads incompletos (subidas fallidas a mitad de proceso) se eliminan después de 7 días para evitar almacenamiento huérfano:

```xml
<Rule>
  <ID>abort-incomplete-multipart</ID>
  <Status>Enabled</Status>
  <Filter>
    <Prefix></Prefix>
  </Filter>
  <AbortIncompleteMultipartUpload>
    <DaysAfterInitiation>7</DaysAfterInitiation>
  </AbortIncompleteMultipartUpload>
</Rule>
```

---

## 3. Configuración de Cifrado Server-Side (SSE-KMS)

### 3.1 Cifrado por Defecto del Bucket

Todo objeto almacenado en el bucket debe estar cifrado con SSE-KMS. El bucket se configura con cifrado por defecto obligatorio (CA-14):

**AWS S3:**

```json
{
  "Rules": [
    {
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/mrk-abc123"
      },
      "BucketKeyEnabled": true
    }
  ]
}
```

**MinIO (SSE-KMS con KES):**

```bash
# Configurar clave KMS en MinIO via KES (Key Encryption Service)
mc admin kms key create ceiba-evidence-key
mc encrypt set SSE-KMS ceiba-evidence-key ceiba-evidence-co/
```

El Upload Service incluye el header `x-amz-server-side-encryption: aws:kms` (o equivalente) en todas las operaciones de escritura al bucket. Si el header está ausente, la bucket policy lo deniega (ver §4).

### 3.2 Referencia a la Clave KMS

| Entorno | Tipo de clave | Referencia |
|---|---|---|
| AWS | AWS KMS CMK (Customer Managed Key) | ARN: `arn:aws:kms:{region}:{account}:key/{key-id}` |
| GCP | Cloud KMS | Recurso: `projects/{project}/locations/{region}/keyRings/{ring}/cryptoKeys/{key}` |
| Azure | Azure Key Vault | URI: `https://{vault}.vault.azure.net/keys/{key}/{version}` |
| On-prem (MinIO) | Vault Transit Engine via KES | Key name configurado en KES: `ceiba-evidence-key` |

La clave KMS se rota anualmente (o según política operacional del país). La rotación de la clave no requiere re-cifrar los objetos existentes en S3/MinIO; los objetos existentes siguen siendo accesibles con las versiones anteriores de la clave (envelope encryption).

---

## 4. Bucket Policy

La bucket policy deniega cualquier escritura que no incluya SSE-KMS y cualquier acceso público.

**AWS S3 Bucket Policy (JSON):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonKMSEncryption",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::ceiba-evidence-co/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    },
    {
      "Sid": "DenyPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::ceiba-evidence-co/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalType": "Anonymous"
        }
      }
    },
    {
      "Sid": "DenyHTTP",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::ceiba-evidence-co",
        "arn:aws:s3:::ceiba-evidence-co/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**Block Public Access (configuración de cuenta/bucket):**

```json
{
  "BlockPublicAcls": true,
  "IgnorePublicAcls": true,
  "BlockPublicPolicy": true,
  "RestrictPublicBuckets": true
}
```

---

## 5. Replicación Multi-AZ

### 5.1 Configuración de Replicación

El object storage se configura con replicación Multi-AZ para garantizar durabilidad (objetivo 99.999999999 % — "11 nueves").

| Proveedor | Mecanismo de replicación |
|---|---|
| **AWS S3** | S3 Standard (Multi-AZ por defecto); opcionalmente S3 Cross-Region Replication para DR |
| **MinIO** | Modo `MULTI_NODE_MULTI_DRIVE` con Erasure Coding distribuido entre AZs (paridad 4+4 para 8 nodos, o 2+2 para 4 nodos) |
| **GCS** | `MULTI_REGIONAL` o `DUAL_REGION` bucket location |
| **Azure Blob** | Zone-Redundant Storage (ZRS) o Geo-Zone-Redundant Storage (GZRS) |

### 5.2 Replicación Cross-Region (DR)

Para despliegues con requisito de DR (RPO ≤ 5 min, RTO ≤ 30 min según propuesta §5.1), se configura replicación asíncrona a una región secundaria:

```hcl
# AWS S3 Cross-Region Replication
resource "aws_s3_bucket_replication_configuration" "dr_replication" {
  bucket = aws_s3_bucket.evidence.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.evidence_dr.arn
      storage_class = "STANDARD"
      encryption_configuration {
        replica_kms_key_id = var.dr_kms_key_arn
      }
    }
  }
}
```

---

## 6. Tabla de Adaptadores S3-API

El Upload Service accede al object storage vía `ObjectStoragePort` (ADR-005). La siguiente tabla resume la compatibilidad por proveedor:

| Proveedor | Interfaz S3-API | Presigned URLs | SSE-KMS | Lifecycle policy | Notas |
|---|---|---|---|---|---|
| **MinIO** | Completa (API S3 v4) | Sí, idéntico a AWS | Sí (via KES o Vault Transit) | Sí | Recomendado para on-prem. Soporte completo de todas las features. |
| **AWS S3** | Referencia | Sí, nativo | Sí (AWS KMS CMK) | Sí | Implementación de referencia del adaptador. |
| **GCS** | Parcial (XML API S3) | Sí, con firma HMAC | Parcial (Cloud KMS via header `x-goog-encryption-kms-key-name`) | Sí (via GCS Object Lifecycle) | Verificar que el header `x-amz-server-side-encryption` sea traducido correctamente. El adaptador debe manejar la diferencia de headers KMS. |
| **Azure Blob** | Parcial (via NFS / emulación) | Sí (SAS tokens, TTL equivalente) | Sí (Azure Key Vault CMK) | Sí (Azure Blob Lifecycle Management) | El adaptador abstrae la diferencia de firma SAS vs. presigned URL. La compatibilidad S3 nativa de Azure es limitada; se recomienda Azurite para testing local. |

---

## 7. Controles de Privacidad

Alineado con la propuesta §5.3 y §5.8:

- **Solo imágenes de vehículos:** el object storage almacena imágenes de placas y vehículos. No se almacenan imágenes de rostros ni datos biométricos.
- **Retención máxima:** imágenes completas 180 días, thumbnails 730 días. La lifecycle policy garantiza la eliminación automática sin intervención manual.
- **Acceso por `country_code`:** las políticas IAM/RBAC del bucket restringen el acceso por prefijo de país. Un operador del país `CO` no puede acceder a objetos del prefijo `images/MX/`.
- **Residencia de datos:** en despliegues con requisito de residencia, el bucket del país se despliega en la región geográfica correspondiente. El módulo Terraform `object-storage` acepta el parámetro `region` por país.

---

## 8. Referencias Cruzadas

| Documento | Relación |
|---|---|
| [upload-service.md](./upload-service.md) | Servicio que escribe al bucket via ObjectStoragePort |
| [security.md](./security.md) | SSE-KMS y controles de privacidad en contexto de la arquitectura de seguridad |
| [terraform/README.md](./terraform/README.md) | Módulo Terraform `object-storage` que provisiona el bucket y las policies |
| [helm/README.md](./helm/README.md) | Valores de configuración del bucket para MinIO Operator |
| [`docs/almacenamiento-lectura/data-lifecycle.md`](../almacenamiento-lectura/data-lifecycle.md) | Ciclo de vida de datos en la plataforma (contexto ampliado) |
