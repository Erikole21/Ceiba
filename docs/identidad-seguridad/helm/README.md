# Helm — Keycloak y Vault

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

---

## 1. Valores configurables de Keycloak

### 1.1 Valores principales

| Valor Helm | Descripción | Por defecto |
|---|---|---|
| `keycloak.image.repository` | Repositorio de la imagen Docker de Keycloak | `quay.io/keycloak/keycloak` |
| `keycloak.image.tag` | Tag de la imagen | `24.0.5` |
| `keycloak.replicas` | Número de réplicas en el Deployment | `3` |
| `keycloak.resources.requests.cpu` | CPU solicitada por pod | `500m` |
| `keycloak.resources.requests.memory` | Memoria solicitada por pod | `1Gi` |
| `keycloak.resources.limits.memory` | Límite de memoria por pod | `2Gi` |
| `keycloak.jvmHeap.initial` | JVM heap inicial (`-Xms`) | `512m` |
| `keycloak.jvmHeap.max` | JVM heap máximo (`-Xmx`) | `1536m` |
| `keycloak.domain` | Dominio público de Keycloak | `auth.sistema.com` |
| `keycloak.ingress.enabled` | Habilitar Ingress para Keycloak | `true` |
| `keycloak.ingress.tls.secretName` | Secret de TLS para el Ingress | `keycloak-tls` |
| `keycloak.ingress.annotations` | Anotaciones del Ingress (cert-manager, nginx, etc.) | `{}` |
| `keycloak.db.url` | JDBC URL del PostgreSQL backend | Inyectada por Vault Agent desde `secret/keycloak/db` |
| `keycloak.db.vaultPath` | Path en Vault para las credenciales de BD | `secret/keycloak/db` |
| `keycloak.countries` | Lista de países (realms) a provisionar | `["CO", "MX", "AR"]` |

### 1.2 Configuración de Vault Agent sidecar para Keycloak

El Vault Agent sidecar inyecta las credenciales de PostgreSQL en `/vault/secrets/keycloak-db` antes de que el pod de Keycloak arranque. Keycloak lee la JDBC URL y credenciales desde ese archivo, nunca desde variables de entorno.

```yaml
# Anotaciones del Deployment de Keycloak
podAnnotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "keycloak"
  vault.hashicorp.com/agent-inject-secret-keycloak-db: "database/creds/keycloak-app"
  vault.hashicorp.com/agent-inject-template-keycloak-db: |
    {{- with secret "database/creds/keycloak-app" -}}
    KC_DB_URL=jdbc:postgresql://keycloak-pg:5432/keycloak?sslmode=require
    KC_DB_USERNAME={{ .Data.username }}
    KC_DB_PASSWORD={{ .Data.password }}
    {{- end }}
```

---

## 2. Valores configurables de Vault

### 2.1 Valores principales

| Valor Helm | Descripción | Por defecto |
|---|---|---|
| `vault.image.repository` | Repositorio de la imagen Docker de Vault | `hashicorp/vault` |
| `vault.image.tag` | Tag de la imagen | `1.17.3` |
| `vault.raft.nodes` | Número de nodos Raft | `3` |
| `vault.raft.storageClass` | StorageClass para los PVCs de Raft | `standard-ssd` |
| `vault.raft.storageSize` | Tamaño del PVC por nodo | `10Gi` |
| `vault.unsealProvider` | Proveedor de auto-unseal (`awskms`, `gcpckms`, `azurekeyvault`, `pkcs11`) | `awskms` |
| `vault.unseal.awskms.region` | Región AWS del KMS key | `us-east-1` |
| `vault.unseal.awskms.kmsKeyId` | ARN o ID del KMS key | `""` |
| `vault.unseal.gcpckms.gcpProject` | Proyecto GCP | `""` |
| `vault.unseal.azurekeyvault.azureKeyVaultName` | Nombre del Azure Key Vault | `""` |
| `vault.pki.mounts` | Lista de países para los que se crean mounts PKI | `["co", "mx", "ar"]` |
| `vault.pki.rootTtl` | TTL de la root CA | `87600h` (10 años) |
| `vault.pki.intermediateTtl` | TTL de las CAs intermedias | `43800h` (5 años) |
| `vault.pki.deviceCertTtl` | TTL de los certificados de dispositivo | `2160h` (90 días) |
| `vault.ui.enabled` | Habilitar la UI web de Vault | `true` |
| `vault.ui.serviceType` | Tipo de servicio para la UI | `ClusterIP` |

---

## 3. Instalación

```bash
# Añadir el repositorio de HashiCorp
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Añadir el repositorio de Bitnami (para Keycloak)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Instalar Vault
helm install vault hashicorp/vault \
  --namespace identity \
  --create-namespace \
  -f values-vault-prod.yaml

# Instalar Keycloak (solo después de que Vault esté inicializado)
helm install keycloak bitnami/keycloak \
  --namespace identity \
  -f values-keycloak-prod.yaml
```

---

## 4. Actualización

### 4.1 Keycloak — rolling update

Keycloak soporta rolling updates sin downtime cuando hay múltiples réplicas:

```bash
# Actualizar Keycloak a una nueva versión
helm upgrade keycloak bitnami/keycloak \
  --namespace identity \
  -f values-keycloak-prod.yaml \
  --set keycloak.image.tag=25.0.0

# Verificar el rollout
kubectl rollout status deployment/keycloak -n identity
```

### 4.2 Vault — rolling restart con auto re-unseal

```bash
# Actualizar Vault a una nueva versión
helm upgrade vault hashicorp/vault \
  --namespace identity \
  -f values-vault-prod.yaml \
  --set vault.image.tag=1.18.0

# El StatefulSet de Vault se actualiza pod a pod
# Cada pod al reiniciar realiza auto-unseal automáticamente
# Verificar que todos los pods están unsealed
kubectl exec -n identity vault-0 -- vault status
kubectl exec -n identity vault-1 -- vault status
kubectl exec -n identity vault-2 -- vault status
```

---

## 5. Rollback

```bash
# Rollback de Keycloak a la versión anterior
helm rollback keycloak -n identity

# Rollback de Vault a la versión anterior
helm rollback vault -n identity

# Verificar el historial de releases
helm history keycloak -n identity
helm history vault -n identity
```

---

## 6. Bootstrap post-instalación

Después de la primera instalación, se debe realizar el bootstrap manual de Vault y la configuración inicial:

```bash
# Paso 1: Inicializar Vault (ejecutar UNA SOLA VEZ)
kubectl exec -n identity vault-0 -- vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json > vault-init.json
# GUARDAR vault-init.json en lugar seguro y eliminar localmente

# Paso 2: Verificar que Vault está activo
kubectl exec -n identity vault-0 -- vault status

# Paso 3: Montar el root CA PKI
kubectl exec -n identity vault-0 -- vault secrets enable -path=pki-root pki
kubectl exec -n identity vault-0 -- vault secrets tune -max-lease-ttl=87600h pki-root
# ... (ver vault-pki-device.md para los comandos completos)

# Paso 4: Montar las CAs intermedias por país
# Se hace en bucle para cada país en vault.pki.mounts
for country in co mx ar; do
  kubectl exec -n identity vault-0 -- vault secrets enable -path=pki-${country} pki
  # ... (ver vault-pki-device.md para los comandos completos)
done

# Paso 5: Montar el KV v2 para secretos
kubectl exec -n identity vault-0 -- vault secrets enable -version=2 -path=secret kv

# Paso 6: Configurar la autenticación Kubernetes
kubectl exec -n identity vault-0 -- vault auth enable kubernetes
```

---

## 7. Diferencias entre cloud y on-prem

| Aspecto | Cloud (AWS/GCP/Azure) | On-prem (Rancher/RKE2) |
|---|---|---|
| `vault.unsealProvider` | `awskms` / `gcpckms` / `azurekeyvault` | `pkcs11` (HSM físico) |
| `vault.raft.storageClass` | StorageClass gestionada (gp3, pd-ssd, Premium_LRS) | StorageClass de Longhorn o Rook-Ceph |
| PostgreSQL Keycloak | RDS / Cloud SQL / Aurora (managed) | Patroni on-prem con Crunchy Data |
| `keycloak.ingress.annotations` | Anotaciones específicas del LB cloud | Anotaciones de Nginx/HAProxy on-prem |
| DNS | Route53 / Cloud DNS / Azure DNS | DNS interno |
| Cert TLS del Ingress | cert-manager + Let's Encrypt o CA cloud | cert-manager + CA interna |
