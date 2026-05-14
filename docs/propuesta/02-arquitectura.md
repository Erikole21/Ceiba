# 2. Arquitectura de la Solución

← [Índice](../propuesta-arquitectura-hurto-vehiculos.md)

---

## 2.1 Diagrama de Contexto (C4 Nivel 1)

```mermaid
graph TB
    subgraph Borde[Borde – calles de cada ciudad]
        DEV[Dispositivos de captura<br/>cámara + ANPR + GPS + GSM]
    end

    subgraph Sistemas_Externos[Sistemas externos por país]
        POL_DB[(BD Policial<br/>vehículos hurtados)]
        POL_IDP[Auth policial<br/>LDAP / DB / SOAP / REST]
    end

    subgraph Plataforma[Sistema Anti-Hurto de Vehículos]
        CORE[Plataforma Cloud<br/>Cloud-agnostic]
    end

    USR_POL[Oficial de policía<br/>búsqueda · alertas · recuperación]
    USR_ANAL[Analista<br/>tendencias · heatmaps · rutas]
    USR_ADMIN[Administrador<br/>dispositivos · vehículos · auditoría]

    DEV -- Telemetría MQTT/TLS + Imágenes pre-signed --> CORE
    CORE -- Sincroniza vehículos hurtados --> POL_DB
    CORE -- Federación OIDC --> POL_IDP

    USR_POL -- HTTPS + OIDC --> CORE
    USR_ANAL -- HTTPS + OIDC --> CORE
    USR_ADMIN -- HTTPS + OIDC --> CORE
```

**Descripción.** El sistema recibe eventos de placa desde miles a millones de dispositivos de borde, los correlaciona contra las bases policiales de cada país y expone consultas, alertas y analítica a tres tipos de usuarios. Todas las integraciones con sistemas externos ocurren a través de capas de adaptación para evitar acoplamiento.

---

## 2.2 Diagrama de Contenedores (C4 Nivel 2)

```mermaid
graph LR
    subgraph EDGE[Dispositivo de borde]
        AG[Agente<br/>Go binario]
        Q[(SQLite WAL<br/>cola persistente)]
        IMG[(Almacén local<br/>imágenes)]
        BF[Bloom filter<br/>placas hurtadas país]
        ANPR[ANPR<br/>componente local]
        AG --- Q
        AG --- IMG
        AG --- BF
        ANPR -.adapter SPI.-> AG
    end

    subgraph INGEST[Ingestión]
        MQTT[MQTT Broker Cluster<br/>EMQX]
        UPL[Upload Service<br/>presigned URLs]
        OBJ[(Object Storage<br/>S3-API / MinIO)]
    end

    subgraph BUS[Backbone de eventos]
        KAFKA[Kafka / Redpanda]
    end

    subgraph PROC[Procesamiento]
        ENR[Enrichment<br/>Geo / Calidad]
        MATCH[Matcher<br/>placas hurtadas]
        ALERT[Alert Service]
        DEDUP[Deduplicator]
    end

    subgraph DATA[Datos]
        PG[(PostgreSQL + PostGIS<br/>eventos consultables)]
        OS[(OpenSearch<br/>búsqueda placas)]
        CH[(ClickHouse<br/>analítica)]
        RDS[(Redis<br/>cache + sesiones)]
    end

    subgraph API[API y Frontend]
        GW[API Gateway<br/>REST + GraphQL]
        WEB[Web App<br/>React + MapLibre]
        WS[WebSocket<br/>alertas en vivo]
    end

    subgraph IDENT[Identidad]
        KC[Keycloak<br/>realm por país]
    end

    subgraph SYNC[Sincronización país]
        ACL1[Adapter País A]
        ACL2[Adapter País B]
        ACLn[Adapter País N]
    end

    AG -- MQTT/TLS --> MQTT
    AG -- HTTPS PUT --> OBJ
    AG -. solicita presign .-> UPL
    UPL --> OBJ
    MQTT --> KAFKA
    KAFKA --> DEDUP --> ENR --> MATCH --> ALERT
    ENR --> PG
    ENR --> OS
    PG -- CDC Debezium --> CH
    MATCH -- consulta --> RDS
    ALERT --> WS
    GW --> PG
    GW --> OS
    GW --> CH
    GW --> RDS
    WEB --> GW
    WEB --> WS
    GW -- valida JWT --> KC
    KC -- federa --> ACL_AUTH[LDAP/DB/SOAP/REST]
    ACL1 --> KAFKA
    ACL2 --> KAFKA
    ACLn --> KAFKA
    ACL1 -. polling/CDC .-> POL_A[(BD País A)]
    ACL2 -. polling/CDC .-> POL_B[(BD País B)]
    ACLn -. polling/CDC .-> POL_N[(BD País N)]
    KAFKA -- replica stolen list --> AG
```

**Descripción.** Los dispositivos publican eventos por MQTT (telemetría liviana) y suben imágenes por HTTPS directo al object storage tras solicitar URL pre-firmada. Kafka actúa como backbone de eventos donde convergen ingestiones y sincronizaciones. Las proyecciones a Postgres, OpenSearch y ClickHouse desacoplan escritura de lectura (CQRS pragmático). La identidad se centraliza en Keycloak con federaciones por país. Cada base policial llega al sistema vía su propio Adapter (ACL).

---

## 2.3 Diagrama del Agente de Borde

```mermaid
graph TB
    subgraph Dispositivo[Dispositivo - 2 cores / 1 GB / 16 GB]
        CAM[Cámaras]
        ANPR[Componente ANPR<br/>local existente]
        GPS[GPS]

        subgraph Agent[Agente Anti-Hurto - binario Go ~30 MB]
            ADP[ANPR Adapter SPI<br/>REST/socket/file]
            COL[Collector<br/>normaliza evento]
            QUE[Queue Manager<br/>SQLite WAL]
            BF[Bloom Filter<br/>placas hurtadas]
            UPL[Uploader<br/>MQTT + HTTP]
            HEA[Health Beacon]
            CFG[Config & OTA]
            WAT[Watchdog]
        end

        BAT[Batería 1 h<br/>respaldo eléctrico]
        GSM[Módem GSM]
    end

    CAM --> ANPR
    GPS --> COL
    ANPR -. lectura placas .-> ADP
    ADP --> COL
    COL --> BF
    BF -- hit -> prioridad alta --> QUE
    COL --> QUE
    QUE --> UPL
    UPL -- MQTT QoS 1 --> CLOUD[(Nube)]
    UPL -- HTTP PUT imagen --> CLOUD
    CFG -. config remota .- CLOUD
    HEA -. heartbeat .- CLOUD
    WAT --- Agent
```

**Descripción.** El agente es un único proceso Go (footprint < 50 MB RSS, < 5 % CPU sostenido). Lee placas del ANPR vía un *adapter SPI*, construye un evento normalizado con GPS + timestamp dual, verifica contra el Bloom filter de placas hurtadas para priorizar el envío, persiste en SQLite (WAL) y reintenta hasta confirmación de entrega. Las imágenes se suben de forma diferida vía URL pre-firmada.

> **Especificación detallada:** [`docs/agente-borde/overview.md`](../agente-borde/overview.md)

---

## 2.4 Hot Path — Captura, Match y Alerta

```mermaid
sequenceDiagram
    autonumber
    participant ANPR as ANPR local
    participant AG as Agente Edge
    participant BF as Bloom Filter local
    participant MQTT as MQTT Broker
    participant K as Kafka
    participant M as Matcher
    participant R as Redis
    participant AL as Alert Service
    participant WS as WebSocket
    participant POL as App Policial

    ANPR->>AG: placa + confidence + timestamp
    AG->>BF: ¿placa en hot-list?
    BF-->>AG: posible match (FP rate ~1%)
    AG->>AG: priorizar evento, generar event_id idempotente
    AG->>MQTT: publish evento (QoS 1, retained=false)
    AG->>AG: solicitar presigned URL para imagen
    AG->>MQTT: publish image_uri tras subir a S3
    MQTT->>K: bridge a topic vehicle.events.raw
    K->>M: consume
    M->>R: GET stolen:{country}:{plate}
    R-->>M: hit confirmado
    M->>AL: trigger alert
    AL->>WS: push alerta a oficiales de la zona
    WS->>POL: notificación + ubicación + prueba visual
```

**Descripción.** Latencia objetivo desde captura hasta notificación al oficial: **p95 < 2 segundos** cuando hay conectividad nominal. El Bloom filter local elimina la mayor parte del tráfico irrelevante antes incluso de subir el evento. La confirmación de match se hace en cloud contra la lista canónica (Redis) para evitar falsos positivos.

---

## 2.5 Cold Path — Sincronización de Vehículos Hurtados por País

```mermaid
sequenceDiagram
    autonumber
    participant POL as BD Policial País X
    participant ACL as Adapter País X
    participant K as Kafka
    participant CN as Canonical Service
    participant PG as PostgreSQL
    participant R as Redis
    participant ED as Edge Distribution Service
    participant AG as Agentes del país

    Note over POL,ACL: Modos soportados:<br/>CDC (Debezium), polling SQL,<br/>SOAP/REST scheduled, file drop
    POL->>ACL: cambio detectado (insert/update/delete)
    ACL->>ACL: mapeo a modelo canónico v1
    ACL->>K: publish stolen.vehicles.events
    K->>CN: consume
    CN->>PG: upsert vehicle hurtado
    CN->>R: actualizar stolen:{country}:{plate}
    CN->>ED: notificar cambio
    ED->>ED: regenerar Bloom filter país (diff)
    ED->>AG: distribuir delta vía MQTT retained topic
    AG->>AG: aplicar delta al BF local
```

**Descripción.** Cada país tiene su propio adapter que mapea su esquema y tecnología a un modelo canónico versionado (9 campos obligatorios + `extensions` JSONB). Los Bloom filters se actualizan por diff en el borde, no por descarga completa.

> **Especificación detallada:** [`docs/sincronizacion-paises/overview.md`](../sincronizacion-paises/overview.md)

---

## 2.6 Read Path — Búsqueda por Matrícula y Facilitar Recuperación

```mermaid
sequenceDiagram
    autonumber
    actor POL as Oficial de policía
    participant WEB as Web App (React + MapLibre)
    participant GW as API Gateway
    participant SS as search-service
    participant OS as OpenSearch
    participant PG as PostgreSQL + PostGIS
    participant OBJ as Object Storage
    participant INC as incident-service

    POL->>WEB: ingresa matrícula "ABC-123"
    WEB->>GW: GET /v1/plates/ABC-123/events?from=&to=&country=
    GW->>GW: valida JWT (Keycloak) + scope de país del oficial
    GW->>SS: buscar eventos por matrícula + filtros
    SS->>OS: query matrícula normalizada (fuzzy match + exacto)
    OS-->>SS: lista de event_ids ordenados por timestamp
    SS->>PG: SELECT eventos por event_id con coordenadas PostGIS
    PG-->>SS: eventos enriquecidos (timestamp, lat/lon, device_id,<br/>confidence, img_uri, país/ciudad)
    SS->>OBJ: genera presigned GET URLs para thumbnails (TTL 5 min)
    OBJ-->>SS: URLs firmadas
    SS-->>GW: lista paginada de eventos con coordenadas + thumbnail_url
    GW-->>WEB: respuesta paginada (JSON)
    WEB->>WEB: renderiza trayectoria en mapa (MapLibre GL)
    WEB->>WEB: panel lateral: fecha/hora · lugar · thumbnail por evento
    POL->>WEB: clic en evento → solicita imagen completa
    WEB->>OBJ: GET presigned URL imagen full (prueba visual)
    OBJ-->>POL: imagen completa del vehículo en circulación

    Note over POL,INC: Flujo de recuperación
    POL->>WEB: abre panel "Gestionar recuperación"
    WEB->>GW: POST /v1/incidents/{plate}/recovery-action
    GW->>INC: registrar acción (despachar unidad, marcar recuperado, cerrar caso)
    INC->>PG: persiste estado del incidente + audit log
    INC-->>WEB: confirmación + nuevo estado del caso
```

**Descripción.** El oficial busca una matrícula y obtiene la trayectoria completa del vehículo en el mapa — cada punto es un avistamiento con fecha/hora, lugar y prueba visual. La imagen completa se sirve directamente desde object storage vía URL pre-firmada de corta duración.

El flujo de recuperación está integrado: el oficial puede despachar una unidad al último punto conocido, escalar el incidente o marcar el vehículo como recuperado. Al marcarse recuperado, el `incident-service` notifica al `sincronizacion-paises` para actualizar el estado canónico. El sistema así cierra el ciclo: **detección → alerta → localización → recuperación → registro**.

---

## 2.7 Vista de Despliegue (Cloud-Agnostic sobre Kubernetes)

```mermaid
graph TB
    subgraph Region[Región Cloud A - Activo-Activo entre AZs]
        subgraph AZ1[AZ 1]
            K8S1[Kubernetes nodes]
            PGP[(Postgres primario)]
            CHN1[ClickHouse node]
        end
        subgraph AZ2[AZ 2]
            K8S2[Kubernetes nodes]
            PGR[(Postgres replica)]
            CHN2[ClickHouse node]
        end
        subgraph AZ3[AZ 3]
            K8S3[Kubernetes nodes]
            OBJ_R[(Object storage<br/>multi-AZ)]
            CHN3[ClickHouse node]
        end
        LB[Global LB + WAF]
        MQ[(MQTT cluster<br/>3 nodos)]
        KF[(Kafka cluster<br/>3 brokers)]
    end

    subgraph DR[Región Cloud B - DR Warm]
        K8SDR[Kubernetes nodes]
        PGD[(Postgres async replica)]
        OBJ_DR[(Object storage replicado)]
    end

    subgraph PRIV[Opción: Nube Privada]
        K8SP[K8s on-prem<br/>Rancher / RKE2]
        PGOP[Postgres HA]
        MINIO[(MinIO)]
        VAULT[Vault PKI]
    end

    DEV[Dispositivos de borde] --> LB
    LB --> K8S1
    LB --> K8S2
    LB --> K8S3
    PGP -- streaming repl --> PGR
    PGP -- async repl --> PGD
    OBJ_R -- replicación --> OBJ_DR

    note1[Mismos manifiestos Helm/Kustomize<br/>se despliegan en cualquier opción]
    note1 -.- K8S1
    note1 -.- K8SDR
    note1 -.- K8SP
```

**Descripción.** El despliegue base es multi-AZ activo-activo en una región primaria, con replicación asíncrona a una región DR. Toda la plataforma se describe con Helm/Kustomize + Terraform modular, lo que permite levantar la misma topología en AWS, Azure, GCP u on-prem sin reescribir la solución.
