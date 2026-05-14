# OpenSearch — Schema del Índice de Eventos

**Componente:** Almacenamiento de lectura — capa OpenSearch  
**Versión del documento:** 1.0  
**Referencia:** [opensearch-ilm.md](./opensearch-ilm.md) · [overview.md](./overview.md)

> **Nota de coordinación:** Este documento es la **fuente de verdad** del índice OpenSearch. El documento [docs/backbone-procesamiento/opensearch-schema.md](../backbone-procesamiento/opensearch-schema.md) referencia este mapping para la perspectiva del indexador. Cualquier cambio de mapping debe reflejarse en ambos documentos.

---

## 1. Estrategia de Indexación

Los eventos de avistamiento se indexan en índices rodantes con el patrón de nombre:

```
vehicle-events-{country_code}-{YYYY-MM}
```

Ejemplos: `vehicle-events-co-2026-05`, `vehicle-events-mx-2026-05`.

Los alias por país (`vehicle-events-co`, `vehicle-events-mx`) apuntan a todos los índices del país para búsquedas que cruzan periodos.

Esta estrategia está alineada con ADR-011 (aislamiento por `country_code`) y facilita el ciclo de vida ILM (ver [opensearch-ilm.md](./opensearch-ilm.md)).

---

## 2. Mapping del Índice

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "event_id": {
        "type": "keyword",
        "doc_values": true
      },
      "country_code": {
        "type": "keyword"
      },
      "device_id": {
        "type": "keyword"
      },
      "plate_raw": {
        "type": "keyword"
      },
      "plate_normalized": {
        "type": "keyword",
        "fields": {
          "text": {
            "type": "text",
            "analyzer": "plate_analyzer"
          }
        }
      },
      "event_ts": {
        "type": "date",
        "format": "strict_date_optional_time_nanos"
      },
      "received_ts": {
        "type": "date",
        "format": "strict_date_optional_time_nanos"
      },
      "confidence": {
        "type": "float"
      },
      "location": {
        "type": "geo_point"
      },
      "image_uri": {
        "type": "keyword",
        "index": false,
        "doc_values": false
      },
      "thumbnail_uri": {
        "type": "keyword",
        "index": false,
        "doc_values": false
      },
      "clock_uncertain": {
        "type": "boolean"
      },
      "image_unavailable": {
        "type": "boolean"
      },
      "enrichment_version": {
        "type": "short"
      },
      "extensions": {
        "type": "object",
        "enabled": false
      }
    }
  },
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "5s",
    "analysis": {
      "analyzer": {
        "plate_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "plate_ngram"]
        }
      },
      "filter": {
        "plate_ngram": {
          "type": "ngram",
          "min_gram": 3,
          "max_gram": 7
        }
      }
    }
  }
}
```

### 2.1 Campo `plate_normalized` — Doble Mapping

El campo `plate_normalized` tiene mapping doble:
- **`keyword`**: para búsquedas exactas y agregaciones (term query).
- **`text` con analizador `plate_analyzer`**: para búsqueda fuzzy y parcial. El analizador aplica lowercase + asciifolding + ngram (3-7 caracteres), lo que permite encontrar `ABC123` buscando `abc`, `b12`, `123`, etc.

---

## 3. Aliases por País

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "vehicle-events-co-2026-05",
        "alias": "vehicle-events-co",
        "is_write_index": true
      }
    },
    {
      "add": {
        "index": "vehicle-events-co-2026-04",
        "alias": "vehicle-events-co"
      }
    },
    {
      "add": {
        "index": "vehicle-events-mx-2026-05",
        "alias": "vehicle-events-mx",
        "is_write_index": true
      }
    }
  ]
}
```

El alias `vehicle-events-co` abarca todos los índices del país Colombia. El `search-service` consulta siempre el alias, nunca el índice concreto.

---

## 4. Queries de Ejemplo

### 4.1 Búsqueda Exacta por Matrícula

```json
GET /vehicle-events-co/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "plate_normalized": "ABC123" } },
        { "range": { "event_ts": {
          "gte": "2026-01-01T00:00:00Z",
          "lte": "2026-05-31T23:59:59Z"
        }}}
      ]
    }
  },
  "sort": [{ "event_ts": "desc" }],
  "size": 50,
  "_source": ["event_id", "event_ts", "device_id", "confidence", "thumbnail_uri"]
}
```

### 4.2 Búsqueda Fuzzy por Matrícula

La búsqueda fuzzy es útil cuando el ANPR tiene una lectura parcialmente incorrecta (ej. `AB0123` en lugar de `ABC123`).

```json
GET /vehicle-events-co/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match": {
            "plate_normalized.text": {
              "query": "AB0123",
              "fuzziness": "AUTO",
              "max_expansions": 50,
              "prefix_length": 2,
              "boost": 2.0
            }
          }
        },
        {
          "fuzzy": {
            "plate_normalized": {
              "value": "AB0123",
              "fuzziness": "AUTO",
              "prefix_length": 2,
              "max_expansions": 50
            }
          }
        }
      ],
      "minimum_should_match": 1,
      "filter": [
        { "range": { "event_ts": { "gte": "now-30d" }}}
      ]
    }
  },
  "sort": ["_score", { "event_ts": "desc" }],
  "size": 20
}
```

**Parámetros de fuzziness:**

| Parámetro | Valor | Descripción |
|---|---|---|
| `fuzziness` | `AUTO` | Distancia de edición automática: 0 para ≤ 2 chars, 1 para 3–5 chars, 2 para > 5 chars |
| `max_expansions` | `50` | Máximo de términos alternativos a considerar (limita el fan-out de la búsqueda) |
| `prefix_length` | `2` | Los primeros 2 caracteres deben coincidir exactamente (mejora precisión y rendimiento) |

### 4.3 Búsqueda Geoespacial — Avistamientos en Radio

```json
GET /vehicle-events-co/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "geo_distance": {
            "distance": "500m",
            "location": {
              "lat": 4.7109,
              "lon": -74.0721
            }
          }
        },
        { "range": { "event_ts": { "gte": "now-24h" }}}
      ]
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": { "lat": 4.7109, "lon": -74.0721 },
        "order": "asc",
        "unit": "m"
      }
    }
  ],
  "size": 100,
  "_source": ["event_id", "plate_normalized", "event_ts", "device_id", "confidence"]
}
```

### 4.4 Agregación — Conteo por Hora en los Últimos 7 Días

```json
GET /vehicle-events-co/_search
{
  "size": 0,
  "query": {
    "range": { "event_ts": { "gte": "now-7d" }}
  },
  "aggs": {
    "events_per_hour": {
      "date_histogram": {
        "field": "event_ts",
        "calendar_interval": "hour",
        "time_zone": "America/Bogota"
      },
      "aggs": {
        "unique_plates": {
          "cardinality": { "field": "plate_normalized" }
        }
      }
    }
  }
}
```

### 4.5 Multi-Country Search (para analistas con acceso a múltiples países)

```json
GET /vehicle-events-co,vehicle-events-mx/_search
{
  "query": {
    "bool": {
      "filter": [
        { "terms": { "country_code": ["CO", "MX"] }},
        { "term": { "plate_normalized": "ABC123" }}
      ]
    }
  }
}
```

---

## 5. Configuración de Permisos por Rol

```json
// Política OpenSearch — rol search_reader (usado por search-service)
{
  "cluster_permissions": ["cluster:monitor/health"],
  "index_permissions": [
    {
      "index_patterns": ["vehicle-events-*"],
      "allowed_actions": ["read", "search", "get"]
    }
  ]
}

// Política OpenSearch — rol search_writer (usado por Enrichment Service)
{
  "index_permissions": [
    {
      "index_patterns": ["vehicle-events-*"],
      "allowed_actions": ["create_index", "index", "bulk"]
    }
  ]
}
```

---

## 6. Notas de Rendimiento

- **Refresh interval de 5 s:** La latencia de indexación máxima antes de que un documento sea visible en búsquedas es de 5 s. Aceptable para el caso de uso (búsqueda no es tiempo real, usa OpenSearch para trayectoria histórica).
- **`_source` completo deshabilitado para campos grandes:** Los campos `image_uri` y `thumbnail_uri` tienen `index: false` y `doc_values: false`; se almacenan en `_source` pero no generan índice invertido ni columna de valores.
- **`dynamic: strict`:** Previene la creación accidental de campos no definidos en el mapping, que podrían causar explosión de mappings en el cluster.
- **Shards por índice:** 3 shards por índice mensual. Con 10 países y retención de 24 meses en fase hot/warm, el total es ~720 shards — dentro del límite recomendado de 1 shard/1 GB de RAM de heap por nodo.
