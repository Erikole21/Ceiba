# Identidad Federada y Seguridad — Visión General

**Módulo:** `identidad-seguridad`
**Versión:** 1.0
**Última actualización:** 2026-05-13

Documentación de soporte: consultar los documentos de detalle enlazados en cada sección.

---

## 1. Objetivo del módulo

Este módulo define la capa de identidad federada y seguridad del Sistema Anti-Hurto de Vehículos. Sus responsabilidades son:

- **Autenticar** usuarios humanos (oficiales, supervisores, analistas, administradores, auditores) contra fuentes heterogéneas de identidad por país (LDAP, base de datos relacional, SOAP/REST legacy, SAML 2.0, OIDC).
- **Autenticar** dispositivos de borde mediante certificados X.509 emitidos por una PKI propia (mTLS).
- **Emitir** tokens JWT canónicos con los claims `country_code`, `role`, `zone` para el consumo uniforme por todos los servicios internos.
- **Aislar** identidades por país mediante el modelo realm-per-country de Keycloak (barrera técnica de ADR-011).
- **Gestionar** secretos de infraestructura con HashiCorp Vault y automatizar la rotación.

Decisiones de diseño aplicadas: **ADR-005** (puertos/adaptadores hexagonales para `DevicePKIPort` y `VaultUnsealPort`), **ADR-007** (Keycloak como broker), **ADR-011** (aislamiento multi-tenant por `country_code`).

---

## 2. Diagrama C4 Nivel 2 — Capa de Identidad y Seguridad

```mermaid
graph LR
    subgraph External["IdPs externos por país"]
        LDAP[LDAP / Active Directory]
        JDBCDB[(Base de datos relacional)]
        SOAP[Servicio SOAP/REST legacy]
        SAMLIDP[IdP SAML 2.0 / OIDC]
    end

    subgraph Identity["Capa de Identidad (Keycloak)"]
        KC["Keycloak Cluster\nrealm por país: CO, MX, AR, …"]
        SPI["User Storage SPI custom\n(SOAP/REST)"]
        KC_LDAP["Federation LDAP\n(User Storage SPI nativo)"]
        KC_JDBC["Federation JDBC\n(User Storage SPI nativo)"]
        KC_SAML["Identity Provider SAML/OIDC\n(Keycloak IdP broker)"]
        PGKC[(PostgreSQL\nKeycloak backend)]
    end

    subgraph PKI["Gestión de Secretos y PKI (Vault)"]
        VAULT["HashiCorp Vault\nHA – 3 nodos Raft"]
        PKI_ROOT["Root CA\n(pki-root)"]
        PKI_CO["Intermediate CA\npki-co"]
        PKI_MX["Intermediate CA\npki-mx"]
        PKI_KV["KV v2\nsecretos de infraestructura"]
        VA["Vault Agent\nsidecar"]
    end

    subgraph Gateway["Punto de entrada"]
        GW["API Gateway\n(Kong / Envoy)"]
    end

    subgraph Consumers["Consumidores de identidad"]
        EMQX["EMQX MQTT Broker\nmTLS device auth"]
        WEBAPP["Web App\nReact + MapLibre"]
        SERVICES["Servicios internos\nbackbone, almacenamiento, analítica"]
    end

    LDAP --> KC_LDAP --> KC
    JDBCDB --> KC_JDBC --> KC
    SOAP --> SPI --> KC
    SAMLIDP --> KC_SAML --> KC
    KC --- PGKC
    KC -- JWKS endpoint --> GW
    GW -- JWT claims\nX-Country-Code, X-Role, X-Zone --> SERVICES
    GW --> WEBAPP
    VAULT --- PKI_ROOT
    PKI_ROOT --> PKI_CO
    PKI_ROOT --> PKI_MX
    VAULT --- PKI_KV
    VAULT --- VA
    VA -- secretos inyectados --> SERVICES
    PKI_CO -- cert X.509 TTL 90d --> EMQX
    PKI_MX -- cert X.509 TTL 90d --> EMQX
    KC -- bind credentials\nLDAP, client secrets --> VAULT
```

---

## 3. Flujo de autenticación humana

El siguiente diagrama muestra el flujo de autenticación de un usuario humano desde la aplicación web hasta la emisión del JWT canónico.

```mermaid
sequenceDiagram
    autonumber
    participant U as Usuario (oficial/supervisor/analista)
    participant WEB as Web App
    participant GW as API Gateway
    participant KC as Keycloak (realm país)
    participant IDP as IdP del país (LDAP/JDBC/SOAP/SAML)

    U->>WEB: accede a la aplicación
    WEB->>KC: redirige a /realms/{country}/protocol/openid-connect/auth\n(Authorization Code + PKCE)
    KC->>U: presenta pantalla de login
    U->>KC: credenciales (username + password [+ TOTP si está habilitado])
    KC->>IDP: valida credenciales contra fuente federada
    IDP-->>KC: respuesta (éxito / fallo)
    KC->>KC: mapea atributos externos a claims canónicos\n(country_code fijado por realm, role, zone)
    KC-->>WEB: authorization_code
    WEB->>KC: POST /token (code + code_verifier)
    KC-->>WEB: access_token (JWT RS256) + refresh_token
    WEB->>GW: GET /v1/... con Authorization: Bearer <access_token>
    GW->>KC: GET /realms/{country}/protocol/openid-connect/certs (JWKS)
    KC-->>GW: claves públicas JWK
    GW->>GW: valida firma RS256 + claims exp, iss, country_code
    GW-->>WEB: respuesta del servicio (HTTP 200)
```

---

## 4. Flujo de autenticación de dispositivo — Bootstrap inicial

```mermaid
sequenceDiagram
    autonumber
    participant OP as Operador
    participant VAULT as HashiCorp Vault
    participant AG as Agente de borde
    participant EMQX as EMQX Broker

    OP->>VAULT: vault token create -policy=bootstrap-co -use-limit=1
    VAULT-->>OP: bootstrap_token (un solo uso)
    OP->>AG: aprovisionamiento seguro del bootstrap_token
    AG->>VAULT: POST /v1/auth/token/login (bootstrap_token)
    VAULT-->>AG: vault_token temporal
    AG->>AG: genera RSA key pair + CSR (CN=device:{device_id}, OU=country:{country_code})
    AG->>VAULT: PUT /v1/pki-co/issue/device-role (CSR)
    VAULT-->>AG: certificado X.509 TTL 90d
    VAULT->>VAULT: invalida bootstrap_token (use-limit alcanzado)
    AG->>EMQX: CONNECT (mTLS con certificado emitido)
    EMQX->>EMQX: verifica cadena: cert → pki-co CA → pki-root CA
    EMQX-->>AG: CONNACK (conexión establecida)
```

---

## 5. Flujo de autenticación de dispositivo — Renovación de certificado

```mermaid
sequenceDiagram
    autonumber
    participant AG as Agente de borde
    participant VAULT as HashiCorp Vault
    participant EMQX as EMQX Broker

    Note over AG: 14 días antes de expiración del cert actual
    AG->>AG: genera nueva RSA key pair + CSR
    AG->>VAULT: POST /v1/auth/cert/login (cert vigente como credencial)
    VAULT->>VAULT: verifica cadena, comprueba que no está revocado
    VAULT-->>AG: vault_token de renovación
    AG->>VAULT: PUT /v1/pki-co/issue/device-role (CSR con nuevo key pair)
    VAULT-->>AG: nuevo certificado X.509 TTL 90d
    AG->>AG: reemplaza certificado anterior
    AG->>EMQX: reconexión mTLS con nuevo certificado
    EMQX-->>AG: CONNACK (conexión restablecida)
```

---

## 6. Tabla de componentes

| Componente | Responsabilidad | Documento de detalle |
|---|---|---|
| **Keycloak Cluster** | Broker de identidad; un realm por país; emisión de JWT RS256; SSO | [keycloak-realm-model.md](./keycloak-realm-model.md) |
| **Federation LDAP** | User Storage SPI nativo de Keycloak — Active Directory / OpenLDAP | [federation-ldap.md](./federation-ldap.md) |
| **Federation JDBC** | User Storage SPI nativo — bases de datos SQL relacionales | [federation-jdbc.md](./federation-jdbc.md) |
| **SPI Custom SOAP/REST** | Adaptador de autenticación para endpoints legacy; caché TTL | [federation-spi-custom.md](./federation-spi-custom.md) |
| **Federation SAML/OIDC** | IdP broker de Keycloak para delegación a IdPs externos estándar | [federation-saml-oidc.md](./federation-saml-oidc.md) |
| **HashiCorp Vault PKI** | Emisión, renovación y revocación de certificados de dispositivo; CA hierarchy | [vault-pki-device.md](./vault-pki-device.md) |
| **Vault Secrets Engine** | KV v2, Database secrets, AppRole, Kubernetes auth | [vault-secrets-engine.md](./vault-secrets-engine.md) |
| **Vault HA (Raft)** | Alta disponibilidad con Raft consensus; auto-unseal hexagonal | [vault-ha-unseal.md](./vault-ha-unseal.md) |
| **JWT Claims Schema** | Contrato canónico de claims: tipos, formatos, restricciones | [jwt-claims-schema.md](./jwt-claims-schema.md) |
| **RBAC Model** | Cinco roles, permisos, scope geográfico, matriz de autorización | [rbac-model.md](./rbac-model.md) |
| **mTLS Device Policy** | Handshake TLS 1.3, ACL EMQX, CRL verification | [mtls-device-policy.md](./mtls-device-policy.md) |
| **Multitenancy Security** | Modelo de aislamiento técnico por realm y country_code | [multitenancy-security.md](./multitenancy-security.md) |
| **Audit Authentication** | Exportación de eventos de auditoría, retención, alertas | [audit-authentication.md](./audit-authentication.md) |
| **Onboarding Nuevo País** | Checklist ordenado para incorporar un país nuevo | [onboarding-nuevo-pais.md](./onboarding-nuevo-pais.md) |
| **Helm** | Valores configurables para Keycloak y Vault en Kubernetes | [helm/README.md](./helm/README.md) |
| **Terraform** | Módulos IaC para realms, PKI, secretos | [terraform/README.md](./terraform/README.md) |

---

## 7. Relaciones con otros módulos del sistema

| Módulo relacionado | Tipo de relación | Contrato |
|---|---|---|
| **`agente-borde`** | Upstream (consumidor de PKI) | Bootstrap token → primer cert X.509; renovación vía cert auth; TTL 90 días |
| **`ingestion-mqtt`** | Upstream (consumidor de mTLS config) | CA certs por país para EMQX; ACL por `device_id` extraído de CN |
| **`almacenamiento-lectura`** | Upstream (consumidor de JWT) | Claim `country_code` (ISO 3166-1 alpha-2) como discriminador de tenant en particionamiento y aliases |
| **`backbone-procesamiento`** | Upstream (consumidor de JWT) | Headers internos `X-Country-Code`, `X-Role`, `X-Zone` propagados por API Gateway |
| **`api-frontend-analitica`** | Downstream (spec de interfaz) | JWKS endpoint por realm; estructura JWT; flujo OIDC Authorization Code + PKCE |

---

## 8. ADRs aplicados en este módulo

| ADR | Decisión | Documento |
|---|---|---|
| **ADR-005** | Arquitectura hexagonal — `DevicePKIPort` y `VaultUnsealPort` como puertos para evitar lock-in | [adr-vault-pki.md](./adr-vault-pki.md), [vault-ha-unseal.md](./vault-ha-unseal.md) |
| **ADR-007** | Keycloak como broker de identidad con federación heterogénea | [keycloak-realm-model.md](./keycloak-realm-model.md) |
| **ADR-010** | PKI propia con certificados cortos (TTL 90 días) para dispositivos | [vault-pki-device.md](./vault-pki-device.md) |
| **ADR-011** (extendido por `adr-realm-por-pais.md`) | Multi-tenant por país con aislamiento de datos; realm por país como barrera técnica de aislamiento | [adr-realm-por-pais.md](./adr-realm-por-pais.md), [multitenancy-security.md](./multitenancy-security.md) |
| **ADR-vault-pki** | Vault PKI Engine detrás de `DevicePKIPort` | [adr-vault-pki.md](./adr-vault-pki.md) |
| **ADR-spi-cache** | Caché en memoria con TTL configurable en SPI custom | [adr-spi-cache.md](./adr-spi-cache.md) |
