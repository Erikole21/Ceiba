# ADR — Vault PKI Engine para Certificados de Dispositivo

**ID:** ADR-vault-pki
**Estado:** Aprobado
**Fecha:** 2026-05-13
**Relacionado con:** ADR-005 (Hexagonal), ADR-010 (PKI propia)

---

## Contexto

Cada dispositivo de borde requiere una identidad criptográfica fuerte para establecer conexiones mTLS con el broker EMQX. La PKI del sistema debe cumplir los siguientes requisitos:

- Emitir certificados por dispositivo con TTL corto (90 días).
- Permitir revocación inmediata de un dispositivo comprometido sin afectar al resto.
- Soportar bootstrap seguro de dispositivos nuevos (token de un solo uso).
- Permitir renovación automática desde el agente sin intervención del operador.
- Ser cloud-agnostic: no depender de un servicio PKI administrado por un proveedor cloud específico.
- Integrarse con la gestión de secretos de infraestructura (credenciales de servicios, bind credentials de LDAP).
- Ser operable con API programable para automatización completa vía IaC.

---

## Alternativas evaluadas

### Alternativa A — cfssl / step-ca

Herramientas de PKI open source dedicadas a la emisión de certificados.

**Ventajas:**

- Ligeras y simples de operar para casos de uso exclusivamente de PKI.
- `step-ca` soporta ACME y renovación automática.

**Desventajas:**

- No integran gestión de secretos: requieren una solución adicional para credenciales de infraestructura.
- Alta disponibilidad no nativa: requiere configuración adicional significativa.
- API menos madura para automatización completa en entornos multi-país.
- No soportan auto-unseal: ante un reinicio del servidor PKI, requieren intervención manual para desbloquear la CA.
- No cumplen el criterio de integración con secrets management.

**Veredicto:** descartada. Resuelve solo la PKI pero obliga a operar una solución adicional para secretos.

### Alternativa B — CA administrada por proveedor cloud (AWS ACM PCA, GCP CAS, Azure CA)

Servicio administrado de CA ofrecido por el proveedor cloud.

**Ventajas:**

- Cero operación de infraestructura PKI.
- SLA garantizado por el proveedor.

**Desventajas:**

- Lock-in directo con el proveedor cloud: viola ADR-005 y el requisito explícito de cloud-agnostic.
- No integra gestión de secretos de forma portable.
- El costo por certificado emitido puede ser significativo a escala (millones de certificados en rotaciones).
- Detrás de un puerto hexagonal (`DevicePKIPort`) el costo de adaptación existe pero no elimina el costo operativo de usar el servicio administrado.

**Veredicto:** descartada como implementación por defecto. Puede ser una implementación alternativa del puerto `DevicePKIPort` en despliegues que lo requieran explícitamente.

### Alternativa C — HashiCorp Vault PKI Engine (seleccionada)

Vault es una plataforma de secrets management con un PKI engine integrado que soporta emisión, renovación y revocación de certificados.

**Ventajas:**

- **Cloud-agnostic:** se despliega en Kubernetes con los mismos manifiestos en cualquier proveedor o on-prem.
- **Integración nativa con secrets management:** un solo sistema gestiona certificados de dispositivo y secretos de infraestructura (credenciales de BD, LDAP bind credentials, client secrets de Keycloak).
- **API REST completa:** toda operación (emisión, renovación, revocación, rotación de CRL) es automatizable vía API, CLI o Terraform.
- **Alta disponibilidad con Raft:** consenso nativo sin dependencias externas de coordinación.
- **Auto-unseal hexagonal:** soporta AWS KMS, GCP KMS, Azure Key Vault y HSM PKCS#11 como backends de unseal (véase [vault-ha-unseal.md](./vault-ha-unseal.md)).
- **Métodos de autenticación múltiples:** `token` (bootstrap), `cert` (renovación por mTLS), `approle` (servicios), `kubernetes` (pods).
- **Auditoría nativa:** audit log de todas las operaciones sobre secretos y certificados.
- **Detrás de `DevicePKIPort`:** el puerto hexagonal abstrae la implementación; si en el futuro un país requiere una CA cloud-managed, se implementa un adaptador del puerto sin cambiar los consumidores.

**Desventajas:**

- Operación no trivial: requiere expertise en Vault (inicialización, Raft, unseal, renovación de root CA).
- La CA raíz debe protegerse operacionalmente (backup seguro de las shares de Shamir de emergencia).

**Veredicto:** seleccionada.

---

## Decisión

**Se adopta HashiCorp Vault PKI Engine como implementación del puerto hexagonal `DevicePKIPort`.**

La interfaz del puerto es:

```
DevicePKIPort:
  IssueCertificate(deviceID string, countryCode string, csrPEM []byte) (certPEM []byte, err error)
  RevokeCertificate(serialNumber string) error
  GetCRL(countryCode string) (crlPEM []byte, err error)
  RenewCertificate(deviceID string, csrPEM []byte, currentCertPEM []byte) (certPEM []byte, err error)
```

La jerarquía de CA es: `pki-root` (root CA, offline excepto para emisión de intermedias) → `pki-{country_code}` (CA intermedia por país, online, emite certificados de dispositivo).

---

## Criterios de selección cumplidos

| Criterio | Vault PKI |
|---|---|
| Cloud-agnostic | Sí — Kubernetes-native, sin dependencia de servicios cloud |
| Integración con secrets management | Sí — mismo sistema para PKI y secretos |
| API automatizable | Sí — REST API, CLI, Terraform provider |
| Alta disponibilidad con Raft | Sí — 3 nodos Raft, líder electo automáticamente |
| Auto-unseal portable | Sí — multiple backends detrás de `VaultUnsealPort` |
| Revocación granular por dispositivo | Sí — `vault pki revoke` + CRL/OCSP |
| Renovación automática desde agente | Sí — cert auth method + CSR |

---

## Consecuencias

### Plan de operación

1. **Inicialización (una vez):** `vault operator init` con 5 shares Shamir, threshold 3, para emergencia. Auto-unseal configurado con KMS seleccionado por entorno (véase [vault-ha-unseal.md](./vault-ha-unseal.md)).
2. **Root CA:** se monta como `pki-root` con TTL de 10 años. La clave privada de la root CA nunca sale de Vault. Se hacen backups cifrados de emergencia.
3. **CA intermedia por país:** se monta como `pki-{country_code}` (p.ej. `pki-co`, `pki-mx`) con TTL de 5 años, firmada por `pki-root`. Se crea una por cada país en el proceso de onboarding.
4. **Roles de emisión:** cada mount intermedio tiene un rol `device-role` con `allow_any_name=true`, `enforce_hostnames=false`, `key_type=rsa`, `key_bits=2048`, `ttl=90d`. Se usa `allow_any_name` porque el esquema de CN `device:{device_id}` contiene dos puntos y no es un hostname RFC 5280 válido; la restricción de emisión por país se impone mediante la política de Vault, no mediante `allowed_domains` (véase [vault-pki-device.md](./vault-pki-device.md) §5).
5. **CRL:** se publica automáticamente en `/v1/pki-{country_code}/crl` y se actualiza en cada revocación. EMQX descarga la CRL periódicamente.
6. **OCSP:** disponible en `/v1/pki-{country_code}/ocsp` para verificación en tiempo real.

### Riesgos aceptados

- Si todos los nodos de Vault fallan simultáneamente y el KMS de auto-unseal no está disponible, la emisión de certificados se interrumpe. Los certificados vigentes siguen siendo válidos en EMQX. Mitigación: runbook de emergency unseal con shares de Shamir documentado en [vault-ha-unseal.md](./vault-ha-unseal.md).
- La root CA offline debe protegerse con copia de seguridad cifrada fuera del sistema.

---

## Referencias

- [vault-pki-device.md](./vault-pki-device.md) — Especificación completa del proceso de PKI
- [vault-ha-unseal.md](./vault-ha-unseal.md) — Topología HA y auto-unseal hexagonal
- [mtls-device-policy.md](./mtls-device-policy.md) — Política de mTLS en EMQX
- [onboarding-nuevo-pais.md](./onboarding-nuevo-pais.md) — Creación de CA intermedia por país
