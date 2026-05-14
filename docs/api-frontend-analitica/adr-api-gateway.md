# ADR-GW-01 — Selección de API Gateway

**Estado:** Aceptado  
**Fecha:** 2026-05-13  
**Autores:** Equipo de arquitectura  
**Componente:** `api-frontend-analitica`

---

## Contexto

El sistema necesita un **punto único de entrada** para todos los microservicios de dominio (`search-service`, `alerts-service`, `incident-service`, `analytics-service`, `devices-service`, `vehicles-service`). Este componente debe cumplir los siguientes requisitos no negociables:

1. **Cloud-agnostic:** debe desplegarse en Kubernetes sobre cualquier proveedor (AWS EKS, GCP GKE, Azure AKS, RKE2 on-prem) sin dependencia de servicios gestionados del proveedor.
2. **Validación nativa de JWT con JWKS:** debe verificar tokens emitidos por Keycloak (RS256, JWKS endpoint) sin requerir implementación custom en cada microservicio.
3. **Rate-limiting con Redis:** ventana deslizante por tenant (`country_code`) y endpoint usando Redis como almacén compartido (para consistencia entre réplicas del gateway).
4. **Extensibilidad hexagonal:** que permita un `ApiGatewayPort` adapter para sustituir en tests o para equivalentes cloud-managed (AWS API GW, Azure API Management) detrás de la misma interfaz.
5. **Complejidad operacional razonable:** el equipo es un equipo SRE pequeño; se prefieren soluciones con documentación amplia y comunidad madura.

---

## Opciones Evaluadas

### Opción A — Kong OSS

**Descripción.** API Gateway open-source basado en NGINX/OpenResty con un model de plugins declarativo. Se despliega en K8s mediante el chart oficial o KIC (Kong Ingress Controller).

| Criterio | Evaluación |
|---|---|
| Cloud-agnostic | Excelente. Helm chart estable para cualquier distribución K8s. |
| JWT JWKS nativo | Plugin `jwt` + `openid-connect` soporta JWKS endpoint con cache configurable (TTL 5 min). |
| Rate-limiting Redis | Plugin `rate-limiting-advanced` con `strategy: redis` y ventana deslizante (`sliding`). Clave `ratelimit:{tenant}:{endpoint}:{window}`. |
| Extensibilidad hexagonal | Puede envolverse en un `ApiGatewayPort` adapter; las rutas se declaran como `KongIngress` CRDs. |
| Complejidad operacional | Media. KIC simplifica la configuración declarativa. Base de datos opcional (DB-less mode). |
| Licencia | Apache 2.0 (OSS). |

**Ventajas:**
- Plugin ecosystem maduro: JWT, rate-limiting, CORS, logging, OpenTelemetry.
- DB-less mode permite configuración como código (ConfigMap / CRD).
- Comunidad activa y documentación extensa.
- Plugins de autenticación ya probados en producción con Keycloak JWKS.

**Desventajas:**
- El plugin `openid-connect` en su versión completa requiere Kong Enterprise. El plugin `jwt` de OSS es suficiente para validación JWKS pero tiene menos configuraciones avanzadas.
- Overhead de NGINX/Lua en cada request (~2–5 ms típico, < 20 ms p95).

---

### Opción B — Envoy Proxy

**Descripción.** Proxy de alto rendimiento (C++), usado como data plane en service meshes (Istio). Puede usarse standalone como API Gateway con configuración xDS o estática.

| Criterio | Evaluación |
|---|---|
| Cloud-agnostic | Excelente. Contenedor estándar, Helm chart disponible. |
| JWT JWKS nativo | `envoy.filters.http.jwt_authn` soporta JWKS con `remote_jwks` y cache. |
| Rate-limiting Redis | Requiere integración con `envoy.filters.http.ratelimit` + servicio `ratelimit` separado (Lyft). Mayor complejidad de despliegue. |
| Extensibilidad hexagonal | Configurable via Envoy API; adapter posible pero xDS es complejo. |
| Complejidad operacional | Alta. La configuración xDS/YAML es verbosa y requiere expertos en Envoy. |
| Licencia | Apache 2.0. |

**Desventajas:** la configuración rate-limiting requiere un proceso sidecar adicional (`ratelimit` service), incrementando la superficie operativa. La curva de aprendizaje para configuraciones no triviales es alta.

---

### Opción C — NGINX Ingress Controller

**Descripción.** Ingress Controller oficial de Kubernetes basado en NGINX. Ampliamente adoptado.

| Criterio | Evaluación |
|---|---|
| Cloud-agnostic | Excelente. Es el Ingress Controller más usado en K8s. |
| JWT JWKS nativo | Limitado en la versión OSS (`nginx-ingress`). Requiere módulo `auth_request` + sidecar Keycloak Gatekeeper o similar. |
| Rate-limiting Redis | No nativo con Redis. `limit_req` de NGINX es local por instancia, no distribuido entre réplicas. |
| Extensibilidad hexagonal | Menos extensible que Kong; configuración via annotations, no plugins. |
| Complejidad operacional | Baja para casos simples; compleja para casos avanzados (rate-limiting distribuido, JWT). |
| Licencia | Apache 2.0. |

**Desventajas:** la ausencia de rate-limiting distribuido con Redis nativo y la necesidad de sidecars para JWT complican significativamente la arquitectura para los requisitos del sistema.

---

## Decisión

**Se adopta Kong OSS (DB-less mode) como API Gateway.**

**Justificación:**

1. Es la única opción que cumple todos los requisitos sin componentes adicionales: JWT/JWKS con cache, rate-limiting Redis con ventana deslizante, routing por prefijo, plugins CORS y OpenTelemetry, todo en un único chart Helm.
2. El modo DB-less elimina la dependencia de una base de datos dedicada para el gateway, simplificando el despliegue.
3. La configuración declarativa como CRDs de Kubernetes (KIC) se integra con GitOps (ArgoCD) de forma natural.
4. La comunidad Kong tiene integraciones documentadas con Keycloak JWKS que acortan el tiempo de implementación.

### Adapter Hexagonal `ApiGatewayPort`

Para cumplir con ADR-005 (arquitectura hexagonal), se define el siguiente puerto:

```
ApiGatewayPort {
  + configureRoute(path: string, upstream: string, methods: []string): void
  + applyJWTPlugin(route: string, jwksEndpoint: string, cacheTTL: Duration): void
  + applyRateLimitPlugin(route: string, limit: int, window: Duration, key: string): void
  + applyPlugin(route: string, plugin: PluginConfig): void
}
```

Implementaciones:
- `KongAdapter` — usa la Admin API de Kong OSS (CRDs en K8s).
- `AWSApiGatewayAdapter` — usa AWS API Gateway REST API (para despliegues en AWS si se decide adoptar el servicio gestionado).
- `MockApiGatewayAdapter` — para tests de integración sin Kong real.

---

## Consecuencias

**Positivas:**
- Un único punto de entrada con autenticación, rate-limiting y routing sin código en los microservicios.
- Configuración como código en CRDs K8s, gestionada por GitOps.
- Sustituible por equivalentes cloud-managed vía el adapter `ApiGatewayPort`.

**Negativas / Riesgos:**
- Kong OSS no incluye el portal de desarrolladores (requiere Kong Enterprise). Se mitiga con Swagger UI hosteado por cada microservicio o con OpenAPI Spec versionada en el repositorio.
- El plugin `openid-connect` avanzado (session management, introspection) requiere Enterprise. Para el caso de uso actual (validación JWT stateless), el plugin `jwt` de OSS es suficiente.
- En escenarios de alta disponibilidad multi-réplica, se requiere que Redis sea accesible por todas las instancias del gateway para el rate-limiting distribuido (ver [api-gateway.md §4](./api-gateway.md)).

---

## Referencias

- [Configuración detallada del API Gateway](./api-gateway.md)
- [ADR-005 — Arquitectura Hexagonal](../propuesta-arquitectura-hurto-vehiculos.md#adr-005--arquitectura-hexagonal-para-servicios-sensibles-a-la-nube)
- [ADR-011 — Multi-tenant por País](../propuesta-arquitectura-hurto-vehiculos.md#adr-011--multi-tenant-por-país-con-aislamiento-de-datos)
