# Fleet Telemetry Architecture

This document provides a visual and conceptual overview of the Fleet Telemetry project – a self-hosted stack for ingesting, storing, and visualizing car telemetry data.

---


## 1. System Overview


```mermaid
flowchart LR
  subgraph Edge["Vehicle / Edge"]
    OBD[OBD-II Dongle]
    Phone[Phone / Pi Client]
    OBD --> Phone
  end

  subgraph Cluster["Kubernetes (k3d on Docker/WSL)"]
    direction LR
    Ingress[Traefik Ingress]
    API[(Ingestion API
Node.js)]
    Q[(Optional Queue
Redis/Kafka)]
    TS[(InfluxDB v2)]
    Graf[Grafana]
    Secrets[(K8s Secrets)]
    PVC[(PersistentVolumeClaim)]

    Ingress --> API
    API -->|write points| TS
    API -->|optional publish| Q
    Q -->|consumer| TS
    Graf --> TS
    TS --- PVC
    API --- Secrets
    Graf --- Secrets
  end

  Phone -->|HTTP/MQTT events| Ingress
```


---

## 2. Kubernetes Deployment View


```mermaid
flowchart TB
  subgraph nsFleet["namespace: fleet"]
    subgraph apiGroup["Deployment: fleet-api (replicas: 2)"]
      API1[(Pod: api-*)]
      API2[(Pod: api-*)]
    end
    SVC_API[(Service: fleet-api)]
    ING_API[[Ingress: api.localtest.me]]
    DB[(StatefulSet: influxdb)]
    SVC_DB[(Service: influxdb)]
    PVC_DB[(PVC: influxdb-data)]
    SECRETS1[(Secret: fleet-api-secrets)]

    ING_API --> SVC_API --> API1 & API2
    API1 --> SVC_DB
    API2 --> SVC_DB
    DB --- PVC_DB
    API1 --- SECRETS1
    API2 --- SECRETS1
  end

  subgraph nsObs["namespace: observability"]
    GRAF[(Deployment: grafana)]
    SVC_GRAF[(Service: grafana)]
    ING_GRAF[[Ingress: grafana.localtest.me]]
    SECRETS2[(Secret: grafana-admin)]
    ING_GRAF --> SVC_GRAF --> GRAF
    GRAF --- SECRETS2
  end

  classDef ns fill:#0b7285,stroke:#0b7285,color:#fff;
  class nsFleet,nsObs ns;
```


---

## 3. Ingestion Request Lifecycle


```mermaid
sequenceDiagram
  autonumber
  participant Client as Edge Client (Phone/Pi)
  participant Ingress as Traefik Ingress
  participant SVC as Service: fleet-api
  participant Pod as Pod: api-*
  participant TS as InfluxDB v2

  Client->>Ingress: POST /ingest (Bearer token)
  Ingress->>SVC: Route by host/path
  SVC->>Pod: Forward request
  Pod->>Pod: Validate & enrich payload
  Pod->>TS: Write line protocol / API v2
  TS-->>Pod: 204 No Content
  Pod-->>Client: 202 Accepted
```


---

## 4. Trip Segmentation State Machine


```mermaid
stateDiagram
  [*] --> Idle
  Idle --> Driving: first event above speed threshold
  Driving --> Driving: continuous events within gap < idleTimeout
  Driving --> Paused: gap >= idleTimeout and speed ~ 0
  Paused --> Driving: speed rises again within resumeWindow
  Paused --> EndTrip: resumeWindow elapsed
  EndTrip --> Idle: commit trip summary (distance, avg speed, extrema)
```


---

## 5. Data Lifecycle and Retention


```mermaid
flowchart LR
  Raw["`Raw measurements (second-level)`"] -->|Retention 7 days| Drop1((expire))
  Raw -->|Downsample job| Min1[1-min aggregates]
  Min1 -->|Retention 90 days| Drop2((expire))
  Min1 -->|Downsample job| Min5[5-min aggregates]
  Min5 -->|Retention 2 years| Archive[(archive/export)]
```


---

### Notes

* The architecture is modular. Each component (API, storage, dashboard) can be replaced or scaled independently.
* All services run inside Kubernetes namespaces (`fleet`, `observability`).
* Default Ingress: Traefik (provided by k3d).
* Default storage: InfluxDB v2 with PVC-based persistence.
* Optional components: Redis/Kafka for buffering, ArgoCD for GitOps, cert-manager for TLS.

---

### Next Steps

* Decide on the data ingestion strategy (HTTP-only, MQTT, or hybrid).
* Define telemetry schema (essential fields and optional sensors).
* Outline trip logic: idle thresholds, segmenting rules.
* Select the first dashboard metrics to visualize.
* Build incrementally; validate each module (API, DB, Grafana) before scaling up.
