# Apache Superset — Integración con ClickHouse

**Componente:** `api-frontend-analitica`  
**Versión del documento:** 1.0

---

## 1. Visión General

Apache Superset (OSS) se despliega como herramienta BI conectada a ClickHouse para los dashboards de analítica del perfil Analista. Proporciona:

- **Datasets** sobre las vistas materializadas de ClickHouse (`vehicle_events_h3_hourly`, `vehicle_events_daily`).
- **Row Level Security (RLS)** por `country_code` para garantizar el aislamiento multi-tenant.
- **Dashboards recomendados** para heatmaps, tendencias y zonas peligrosas.
- **Integración en la Web App** vía iframe embed o enlace SSO Keycloak.

---

## 2. Configuración K8s — Imagen OSS y Variables de Entorno

```yaml
# superset-values.yaml (Helm chart: apache/superset)
image:
  repository: apache/superset
  tag: "3.1.0"
  pullPolicy: IfNotPresent

# Replicas (stateless con metadata en PostgreSQL)
replicaCount: 2

# Base de datos de metadata (PostgreSQL dedicado)
postgresql:
  enabled: false  # usar PostgreSQL externo
  
extraEnv:
  SUPERSET_ENV: production
  SUPERSET_LOG_LEVEL: WARNING
  SQLALCHEMY_DATABASE_URI: "postgresql+psycopg2://superset:${SUPERSET_DB_PASS}@superset-pg.default.svc:5432/superset"

# SECRET_KEY: generado con `openssl rand -base64 42`, almacenado en K8s Secret
# NUNCA hardcoded en values.yaml
extraEnvFrom:
  - secretRef:
      name: superset-secrets

# Feature flags
configOverrides:
  my_override: |
    FEATURE_FLAGS = {
      "EMBEDDED_SUPERSET": True,          # permite embed vía iframe
      "ENABLE_TEMPLATE_PROCESSING": True,  # Jinja2 en SQL para filtros dinámicos
      "ALERT_REPORTS": True,              # alertas programadas por email
      "ROW_LEVEL_SECURITY": True,         # RLS por country_code
    }
    # Seguridad
    SESSION_COOKIE_SAMESITE = "Lax"
    SESSION_COOKIE_SECURE = True
    TALISMAN_ENABLED = True
    # Límites
    ROW_LIMIT = 50000
    SQL_MAX_ROW = 100000
```

**Secret K8s para Superset:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: superset-secrets
  namespace: anti-hurto
type: Opaque
stringData:
  SECRET_KEY: "<openssl rand -base64 42>"
  SUPERSET_DB_PASS: "<password del PostgreSQL de metadata>"
  CLICKHOUSE_PASSWORD: "<password de conexión ClickHouse>"
```

---

## 3. Conexión ClickHouse

### 3.1 SQLAlchemy URI

```
clickhouse+connect://superset_ro:<password>@clickhouse.default.svc.cluster.local:8123/anti_hurto?secure=false
```

**Alternativa con HTTP endpoint:**
```
clickhouse+http://superset_ro:<password>@clickhouse.default.svc.cluster.local:8123/anti_hurto
```

### 3.2 Configuración del Database en Superset

```json
{
  "engine": "clickhousedb",
  "configuration_method": "sqlalchemy_form",
  "sqlalchemy_uri": "clickhouse+connect://superset_ro:${CLICKHOUSE_PASSWORD}@clickhouse:8123/anti_hurto",
  "extra": {
    "engine_params": {
      "connect_args": {
        "connect_timeout": 10,
        "send_receive_timeout": 300
      }
    },
    "pool_size": 10,
    "max_overflow": 20,
    "pool_recycle": 300
  }
}
```

**Usuario de solo lectura en ClickHouse:**
```sql
CREATE USER superset_ro IDENTIFIED WITH plaintext_password BY '<password>';
GRANT SELECT ON anti_hurto.* TO superset_ro;
-- Solo acceso a tablas de lectura, sin acceso a tablas de sistema
```

---

## 4. Datasets sobre Vistas ClickHouse

| Dataset | Tabla/Vista ClickHouse | Descripción |
|---|---|---|
| `vehicle_events_h3_hourly` | `vehicle_events_h3_hourly` | Base del heatmap: celda H3, event_count, avg_confidence por hora y país. |
| `vehicle_events_daily` | `vehicle_events_daily` | Base de tendencias: hurtos por día/ciudad/país. |
| `stolen_vehicles_summary` | Vista SQL `stolen_vehicles_summary` | Resumen de vehículos hurtados por marca, color, modelo. |

**Vista para resumen de vehículos hurtados:**
```sql
-- Crear en ClickHouse
CREATE VIEW anti_hurto.stolen_vehicles_summary AS
SELECT
    country_code,
    brand,
    vehicle_class,
    color,
    toYear(theft_date) AS theft_year,
    count() AS theft_count
FROM anti_hurto.stolen_vehicles_daily  -- tabla proyectada desde PostgreSQL via CDC
GROUP BY country_code, brand, vehicle_class, color, theft_year;
```

---

## 5. Dashboards Recomendados

### 5.1 Dashboard: Heatmap de Zonas Peligrosas

**Objetivo:** visualizar la densidad de hurtos por celda H3 en un mapa interactivo.

**Charts:**
- **Deck.gl H3 Hexagon** (nativo en Superset con ClickHouse): `h3_index_8`, métrica `count()`, color por `event_count`.
- Filtros: `country_code`, `city`, rango de fechas.

**Dataset:** `vehicle_events_h3_hourly`

**SQL base:**
```sql
SELECT
    h3_index_8,
    sum(event_count) AS total_events,
    avg(avg_confidence) AS avg_confidence,
    any(lat_centroid) AS lat,
    any(lon_centroid) AS lon
FROM vehicle_events_h3_hourly
WHERE country_code = '{{ country_code }}'
  AND event_hour >= '{{ from_date }}'
  AND event_hour <= '{{ to_date }}'
GROUP BY h3_index_8
ORDER BY total_events DESC
LIMIT 5000
```

### 5.2 Dashboard: Tendencias de Hurto

**Objetivo:** analizar la evolución temporal del número de hurtos.

**Charts:**
- **Line Chart:** hurtos por día (últimos 90 días).
- **Bar Chart:** hurtos por ciudad (top 10).
- **Scorecards:** total período actual vs. período anterior (variación %).

**Dataset:** `vehicle_events_daily`

### 5.3 Dashboard: Perfil del Vehículo Hurtado

**Objetivo:** entender qué tipo de vehículos son más frecuentemente hurtados.

**Charts:**
- **Pie Chart:** distribución por marca.
- **Bar Chart:** distribución por color.
- **Table:** top 20 modelos más hurtados.

**Dataset:** `stolen_vehicles_summary`

---

## 6. Row Level Security (RLS) por `country_code`

El RLS en Superset garantiza que un analista sólo vea datos de su `country_code` sin importar el dashboard al que acceda.

### 6.1 Configuración de RLS en Superset

```python
# En superset_config.py (o via UI Admin → Security → Row Level Security)
# Se crea una regla RLS por dataset:
#
# Dataset: vehicle_events_h3_hourly
# Filter: country_code = '{{ current_username_country }}'
# Roles afectados: Analyst (rol de Superset)
```

### 6.2 Mapeo Keycloak → Superset Role

Con SSO/Keycloak, el atributo `country_code` del JWT se mapea al filtro RLS:

```python
# custom_security_manager.py (hereda de SupersetSecurityManager)
class CustomSecurityManager(SupersetSecurityManager):
    def get_rls_filters(self, table):
        # Extraer country_code del token OIDC del usuario actual
        user = self.current_user
        country_code = user.extra_attributes.get('country_code', 'NONE')
        return [f"country_code = '{country_code}'"]
```

---

## 7. Integración en la Web App

### Opción A — Iframe Embed (recomendada para analistas)

Requiere activar `EMBEDDED_SUPERSET: True` en los feature flags.

```typescript
// src/modules/analytics/components/SupersetDashboard.tsx
const DASHBOARD_IDS = {
  heatmap: 'dashboard-heatmap-co',
  trends: 'dashboard-trends-co',
  profile: 'dashboard-vehicle-profile-co',
};

export const SupersetDashboard: React.FC<{ type: keyof typeof DASHBOARD_IDS }> = ({ type }) => {
  const supersetUrl = import.meta.env.VITE_SUPERSET_EMBED_URL;
  const embedUrl = `${supersetUrl}/embedded/${DASHBOARD_IDS[type]}?standalone=2`;
  
  return (
    <iframe
      src={embedUrl}
      title={`Dashboard ${type}`}
      style={{ width: '100%', height: 'calc(100vh - 64px)', border: 'none' }}
      sandbox="allow-scripts allow-same-origin allow-forms allow-popups"
    />
  );
};
```

### Opción B — SSO Keycloak Link (para acceso directo)

El analista puede acceder directamente a Superset mediante SSO. Superset se configura con el proveedor OIDC de Keycloak:

```python
# superset_config.py
from flask_appbuilder.security.manager import AUTH_OAUTH

AUTH_TYPE = AUTH_OAUTH
OAUTH_PROVIDERS = [
    {
        'name': 'keycloak',
        'icon': 'fa-key',
        'token_key': 'access_token',
        'remote_app': {
            'client_id': 'superset',
            'client_secret': '${KEYCLOAK_CLIENT_SECRET}',
            'api_base_url': 'https://keycloak.anti-hurto.internal/realms/co/protocol/openid-connect/',
            'server_metadata_url': 'https://keycloak.anti-hurto.internal/realms/co/.well-known/openid-configuration',
            'client_kwargs': {'scope': 'openid profile email'},
        },
    }
]
```

---

## 8. Guía de Despliegue K8s

### 8.1 Instalar con Helm

```bash
helm repo add superset https://apache.github.io/superset
helm repo update

# Instalar en namespace anti-hurto
helm install superset superset/superset \
  --namespace anti-hurto \
  --values superset-values.yaml \
  --set postgresql.enabled=false \
  --timeout 10m
```

### 8.2 Configurar Ingress TLS

```yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: kong
    cert-manager.io/cluster-issuer: vault-pki
  hosts:
    - host: superset.anti-hurto.internal
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: superset-tls
      hosts:
        - superset.anti-hurto.internal
```

### 8.3 Actualización

```bash
helm upgrade superset superset/superset \
  --namespace anti-hurto \
  --values superset-values.yaml \
  --reuse-values
```

### 8.4 Rollback

```bash
helm rollback superset <revision> --namespace anti-hurto
```

---

## 9. Referencias

- [webapp-architecture.md §8 — Embed](./webapp-architecture.md)
- [analytics-service.md](./analytics-service.md)
- [almacenamiento-lectura/clickhouse-h3-views.md](../almacenamiento-lectura/clickhouse-h3-views.md)
- [helm/README.md](./helm/README.md)
- Apache Superset Docs: https://superset.apache.org/docs/
