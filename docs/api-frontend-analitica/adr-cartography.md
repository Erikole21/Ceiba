# ADR-CARTO-01 — Librería de Cartografía

**Estado:** Aceptado  
**Fecha:** 2026-05-13  
**Autores:** Equipo de arquitectura  
**Componente:** `api-frontend-analitica` — Web App

---

## Contexto

La Web App React necesita una librería de cartografía para:

1. **Mapa operacional del oficial:** renderizar la trayectoria de un vehículo hurtado como puntos en el mapa (eventos de avistamiento), con popups que muestran timestamp, device_id, thumbnail y coordenadas (Bogotá: ~4.71°N, 74.07°W).
2. **Heatmap de analítica:** renderizar celdas hexagonales H3 (resolución 8, ~0.74 km²) con densidad de hurtos por zona, powered by Kepler.gl.
3. **Alertas en vivo:** centrar el mapa automáticamente en la ubicación de la alerta recibida por WebSocket.

Los requisitos no negociables son:

1. **Cloud-agnostic:** no debe depender de servicios de tiles de proveedores cloud (Google Maps, Mapbox SaaS). Debe funcionar con tiles OSM auto-hosteados o CDN público.
2. **Vector layers para trayectorias de placas:** soporte de capas vectoriales (GeoJSON) para renderizar rutas y puntos.
3. **Compatibilidad con Kepler.gl:** Kepler.gl usa MapLibre GL JS o Mapbox GL JS como backend de renderizado. La librería elegida debe ser compatible.
4. **Licencia OSS permisiva:** compatible con distribución sin regalías ni restricciones comerciales.

---

## Opciones Evaluadas

### Opción A — MapLibre GL JS

**Descripción.** Fork open-source de Mapbox GL JS (antes del cambio de licencia en 2021). Renderizado de mapas vectoriales acelerado por WebGL. Comunidad activa mantenida por la Linux Foundation.

| Criterio | Evaluación |
|---|---|
| Cloud-agnostic OSM tiles | Excelente. Acepta cualquier tile server compatible con `xyz` o `tms`. Configuración: `style.sources.openmaptiles`. |
| Vector layers (trayectorias de placas) | Nativo. Soporte de capas `circle`, `line`, `fill`, `symbol` sobre fuentes GeoJSON. |
| Compatibilidad Kepler.gl | Excelente. Kepler.gl v3+ soporta MapLibre GL JS como deck.gl base map layer. |
| Licencia | BSD 3-Clause (OSS, permisiva). |
| Performance | WebGL: excelente para miles de puntos y capas vectoriales simultáneas. |
| Comunidad | Activa (Linux Foundation), actualizaciones frecuentes, amplia documentación. |

**Ventajas:**
- Drop-in replacement para Mapbox GL JS: APIs compatibles, migración minimal.
- Soporte nativo de PMTiles y MBTiles para tiles auto-hosteados.
- Plugins de la comunidad: `@maplibre/maplibre-gl-leaflet`, `maplibre-gl-draw`, etc.
- Kepler.gl tiene soporte oficial para MapLibre GL JS desde v3.0.

**Desventajas:**
- Requiere configurar un tile provider (CDN público de OSM o tiles auto-hosteados). No incluye tiles out-of-the-box.
- Algunas funciones avanzadas (terrain 3D, globe projection) son más nuevas y menos probadas que en Mapbox.

---

### Opción B — Leaflet

**Descripción.** Librería JavaScript liviana para mapas interactivos. La más popular históricamente para mapas de tiles raster.

| Criterio | Evaluación |
|---|---|
| Cloud-agnostic OSM tiles | Excelente. Acepta cualquier tile server XYZ. |
| Vector layers (trayectorias de placas) | Limitado. GeoJSON sobre tiles raster; no renderiza tiles vectoriales. Performance degradada con muchos puntos (> 10 K). |
| Compatibilidad Kepler.gl | No compatible. Kepler.gl requiere WebGL (Mapbox o MapLibre). |
| Licencia | BSD 2-Clause (OSS, permisiva). |
| Performance | Raster: limitado para grandes datasets de puntos. Sin aceleración WebGL. |

**Desventajas críticas:**
- Sin soporte WebGL: el renderizado de miles de eventos de avistamiento en una sola vista degrada significativamente.
- Incompatible con Kepler.gl: requeriría mantener dos librerías de mapas distintas (Leaflet para operacional, Kepler.gl con su propio base).
- No soporta Vector Tiles (MVT/PBF), limitando las opciones de tile providers modernos.

---

### Opción C — OpenLayers

**Descripción.** Librería de mapas OSS con soporte extenso de proyecciones y formatos OGC (WMS, WFS, WMTS).

| Criterio | Evaluación |
|---|---|
| Cloud-agnostic OSM tiles | Excelente. Soporte de XYZ, WMTS, TMS. |
| Vector layers (trayectorias de placas) | Bueno. WebGL renderer disponible para grandes datasets. |
| Compatibilidad Kepler.gl | No compatible. Kepler.gl no soporta OpenLayers como base. |
| Licencia | BSD 2-Clause (OSS, permisiva). |
| Performance | WebGL disponible pero menos maduro que MapLibre. |

**Desventajas:**
- API más compleja que MapLibre o Leaflet para casos de uso básicos.
- Incompatible con Kepler.gl (mismo problema que Leaflet).
- Comunidad más pequeña en el contexto de aplicaciones web modernas (más popular en aplicaciones GIS desktop-like).

---

## Decisión

**Se adopta MapLibre GL JS + Kepler.gl como stack de cartografía.**

- **MapLibre GL JS:** para el mapa operacional (trayectorias, alertas, popups).
- **Kepler.gl:** para capas de analítica (heatmaps H3, rutas de escape). Kepler.gl usa MapLibre GL JS como su base de renderizado, eliminando la necesidad de dos librerías independientes.

**Justificación:**

1. MapLibre GL JS es la única opción plenamente compatible con Kepler.gl (el sistema usa Kepler.gl para los heatmaps H3 del analista).
2. La aceleración WebGL permite renderizar miles de puntos (eventos de avistamiento de un vehículo) sin degradación de performance.
3. La licencia BSD 3-Clause es la más permisiva de las evaluadas.
4. La configuración del tile provider en el Helm chart permite cambiar entre CDN público de OSM y tiles auto-hosteados sin cambiar código de la aplicación.

---

## Consecuencias

### Configuración del Tile Provider en Helm Chart

El tile provider se configura como variable de entorno en el frontend, inyectada por el Helm chart:

```yaml
# values.yaml
webapp:
  env:
    VITE_MAP_STYLE_URL: "https://tiles.openstreetmap.org/styles/basic/style.json"
    # Alternativa tiles auto-hosteados (on-prem):
    # VITE_MAP_STYLE_URL: "http://maptiler.internal/styles/basic/style.json"
    VITE_KEPLER_BASE_MAP: "maplibre"
```

En la aplicación:
```typescript
// src/modules/map/config.ts
export const MAP_STYLE_URL = import.meta.env.VITE_MAP_STYLE_URL;
export const MAPLIBRE_CONFIG = {
  style: MAP_STYLE_URL,
  center: [-74.0721, 4.7110], // Bogotá por defecto
  zoom: 11,
};
```

### Tile Providers Recomendados

| Escenario | Tile Provider | Configuración |
|---|---|---|
| Cloud (producción) | [OpenMapTiles CDN](https://openmaptiles.org/hosting/) | URL de estilo en `VITE_MAP_STYLE_URL` |
| Cloud (alternativa) | [Protomaps CDN](https://protomaps.com/) | PMTiles CDN URL |
| On-prem | [Martin tile server](https://martin.maplibre.org/) | Helm chart sub-dependencia |
| Desarrollo local | Demo style de MapLibre | `https://demotiles.maplibre.org/style.json` |

### Kepler.gl con MapLibre GL JS

```typescript
// src/modules/analytics/components/HeatmapLayer.tsx
import KeplerGl from '@kepler.gl/components';
import { KeplerGlSchema } from '@kepler.gl/schemas';

// Kepler.gl v3+ acepta mapStyle con provider 'maplibre'
const keplerConfig = {
  mapStyle: {
    styleType: 'maplibre',
    mapStyles: {
      maplibre: {
        id: 'maplibre',
        label: 'MapLibre OSM',
        url: MAP_STYLE_URL,
        icon: '/map-icon.png',
      }
    }
  }
};
```

---

## Referencias

- [webapp-architecture.md — Módulo de Mapa](./webapp-architecture.md)
- [ADR-005 — Arquitectura Hexagonal](../propuesta-arquitectura-hurto-vehiculos.md#adr-005--arquitectura-hexagonal-para-servicios-sensibles-a-la-nube)
- MapLibre GL JS: https://maplibre.org/maplibre-gl-js/docs/
- Kepler.gl MapLibre support: https://docs.kepler.gl/docs/user-guides/base-map-styles
- [helm/README.md — Configuración de tiles](./helm/README.md)
