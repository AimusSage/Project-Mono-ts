# My First Fleet Telemetry (Home/Enthusiast Edition)

A self-hosted stack that ingests car telemetry (e.g., OBD-II, GPS), stores it in a time-series database, and visualizes trips and metrics with Grafana—running on Kubernetes without full virtualization (k3d on Docker/WSL).  
Alternatively, I will also work on a fully virtualized version in the future.

> **Primary goals:** get back into the swing of things with something that's fun to work on for a while.  
> **Secondary goals:** clean architecture, GitOps friendliness, minimal vendor lock-in.

---

## Features

- **Ingestion API** (Node.js): accepts JSON via HTTP or MQTT (bridge), validates and enriches events.
- **Time-series storage**: InfluxDB v2 (default). Pluggable adapter pattern if you prefer TimescaleDB later.
- **Dashboards**: Grafana dashboards for live metrics and trip summaries.
- **Device options**: any OBD-II → phone app → HTTP/MQTT forwarder; or a small edge client (Raspberry Pi) that posts directly.
- **Cloud-agnostic**: designed to run on k3d (K8s-in-Docker), AKS, or any CNCF-compatible cluster.
- **Secrets & TLS ready**: K8s Secrets + optional cert-manager for real certificates.

---

## Architecture (high level)

```
[OBD-II dongle] -> [Phone/Edge client] -> (HTTP/MQTT)
                                       \
                                        v
                          +-----------------------------+
                          |        Ingestion API        |  Node.js
                          +--------------+--------------+
                                         |
                                         v
                                   [Queue/Buffer]      (optional Kafka/Redis; direct write by default)
                                         |
                                         v
                               +-------------------+
                               |    InfluxDB v2    |  time-series
                               +---------+---------+
                                         |
                                         v
                               +-------------------+
                               |      Grafana      |  dashboards & alerts
                               +-------------------+
```

Kubernetes resources:

- `Deployment`: api, grafana, (optional) mosquitto
- `StatefulSet`: influxdb (persistent storage)
- `Service`: cluster networking
- `Ingress`: external access via Traefik (k3d default)
- `Secret`/`ConfigMap`: credentials and configuration
- `PersistentVolumeClaim`: data durability

---

## Data model (events)

### Event schema (v1)

```json
{
  "schema_version": 1,
  "vin": "WAUZZZFY0R1234567",
  "trip_id": "b2bdc7f4-63af-4c6a-a9cb-0a9d4a0c9a0c",
  "seq": 154,
  "ts": "2025-11-04T12:00:00Z",
  "lat": 49.5891,
  "lon": 8.6527,
  "speed_kph": 82.4,
  "rpm": 2350,
  "coolant_c": 86.0,
  "throttle_pos": 0.32,
  "soc_pct": 87.5,
  "fuel_level_pct": 62.0
}
```

### Notes

- `vin` is the primary identity; you can also use `vehicle_id`.
- `trip_id` is a UUID generated on the device when a trip starts.
- `seq` increments per trip (1, 2, 3...) to ensure stable ordering and idempotency.
- `ts` is UTC ISO-8601; the API will reject clock-skew beyond a configurable window.
- The ingestion service tags measurements by VIN and groups events by trip (idle timeout threshold configurable).

---

## Offline uploads (store-and-forward)

Offline mode allows the device to buffer and upload complete trips later if no continuous connection is available.

### Device-side storage (SQLite)

Use two tables:

```sql
CREATE TABLE trips (
  trip_id TEXT PRIMARY KEY,
  vin TEXT NOT NULL,
  started_at TEXT NOT NULL,
  ended_at TEXT,
  meta_json TEXT
);

CREATE TABLE events (
  trip_id TEXT NOT NULL,
  seq INTEGER NOT NULL,
  ts TEXT NOT NULL,
  lat REAL NOT NULL,
  lon REAL NOT NULL,
  speed_kph REAL,
  rpm REAL,
  coolant_c REAL,
  throttle_pos REAL,
  soc_pct REAL,
  fuel_level_pct REAL,
  PRIMARY KEY (trip_id, seq)
);
```

### Chunking (NDJSON + gzip)

- Export chunks like `{trip_id}-{chunk_no}.ndjson.gz`, one JSON event per line (the schema above).
- Each trip has a manifest describing its chunks and metadata.

**Example manifest:**

```json
{
  "schema_version": 1,
  "vin": "WAUZZZFY0R1234567",
  "trip_id": "b2bdc7f4-63af-4c6a-a9cb-0a9d4a0c9a0c",
  "started_at": "2025-11-04T11:53:12Z",
  "ended_at": "2025-11-04T12:21:44Z",
  "event_count": 3481,
  "chunks": [
    {
      "no": 1,
      "first_seq": 1,
      "last_seq": 900,
      "size_bytes": 214532,
      "sha256": "…"
    },
    {
      "no": 2,
      "first_seq": 901,
      "last_seq": 1800,
      "size_bytes": 209331,
      "sha256": "…"
    },
    {
      "no": 3,
      "first_seq": 1801,
      "last_seq": 3481,
      "size_bytes": 301223,
      "sha256": "…"
    }
  ]
}
```

---

## API endpoints

### Realtime ingestion

```http
POST /v1/ingest
Authorization: Bearer <api_key>
Content-Type: application/json

{ ...single event payload... }
```

If `trip_id` and `seq` are present, they’re stored directly.  
If not, the server assigns them dynamically based on VIN and time gaps.

### Offline uploads

```
POST /v1/trips
→ { "trip_id": "<uuid>" }

PUT /v1/trips/{trip_id}/chunks/{chunk_no}
Headers:
  Idempotency-Key: {trip_id}:{chunk_no}
  X-Chunk-SHA256: <hex>
  Content-Type: application/x-ndjson+gzip

POST /v1/trips/{trip_id}/complete
Content-Type: application/json
Body: manifest JSON (see above)
```

Each chunk upload is idempotent — retries are safe.

---

## InfluxDB mapping

- **Measurement:** `telemetry`
- **Timestamp:** `ts`
- **Tags:** `vin`, `trip_id`
- **Fields:** numeric values such as `speed_kph`, `rpm`, `coolant_c`, `throttle_pos`, `soc_pct`, `fuel_level_pct`
- **Importer:** sorts by `(trip_id, seq)` and skips duplicates before writing.

---

## Prerequisites

- **Windows 10/11** with **WSL2** and a Linux distro (e.g., Ubuntu)
- **Docker Desktop** using the **WSL2 backend**
- **kubectl** and **helm** available in WSL
- Optional: **Tailscale** or LAN access if you’ll manage the cluster remotely

---

## Quick start (k3d on WSL)

### 1) Create the cluster (no Hyper-V/VirtualBox required)

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

k3d cluster create fleet   --servers 1 --agents 2   -p "80:80@loadbalancer" -p "443:443@loadbalancer"

kubectl get nodes
```

### 2) Namespaces & base tooling

```bash
kubectl create ns fleet
kubectl create ns observability
```

### 3) InfluxDB v2

```bash
helm repo add influxdata https://helm.influxdata.com/
helm upgrade --install influxdb influxdata/influxdb2 -n fleet   --set persistence.storageClass=local-path   --set persistence.size=5Gi   --set adminUser.organization=fleet   --set adminUser.bucket=telemetry   --set adminUser.user=admin   --set adminUser.password=admin123   --set adminUser.token=dev-token-please-change
```

> After install, port-forward to initialize if needed:

```bash
kubectl -n fleet port-forward svc/influxdb-influxdb2 8086:80
# Visit http://localhost:8086 (first-run wizard), then ^C to stop
```

### 4) Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm upgrade --install grafana grafana/grafana -n observability   --set adminPassword=admin123   --set persistence.enabled=true   --set persistence.size=1Gi   --set service.type=ClusterIP
```

Expose via Ingress:

```yaml
# grafana-ing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: observability
spec:
  rules:
    - host: grafana.localtest.me
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: grafana
                port:
                  number: 80
```

```bash
kubectl apply -f grafana-ing.yaml
# Visit http://grafana.localtest.me
```

### 5) Ingestion API (Node.js)

Environment (K8s Secret):

```yaml
# fleet-api-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: fleet-api-secrets
  namespace: fleet
type: Opaque
stringData:
  INFLUX_URL: http://influxdb-influxdb2.fleet.svc.cluster.local
  INFLUX_TOKEN: dev-token-please-change
  INFLUX_ORG: fleet
  INFLUX_BUCKET: telemetry
  API_KEY: change-me
```

Deployment and Service:

```yaml
# fleet-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fleet-api
  namespace: fleet
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fleet-api
  template:
    metadata:
      labels:
        app: fleet-api
    spec:
      containers:
      - name: api
        image: ghcr.io/your-namespace/fleet-api:latest
        ports:
        - containerPort: 3000
        envFrom:
        - secretRef:
            name: fleet-api-secrets
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: fleet-api
  namespace: fleet
spec:
  selector:
    app: fleet-api
  ports:
  - name: http
    port: 80
    targetPort: 3000
```

Ingress:

```yaml
# fleet-api-ing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fleet-api
  namespace: fleet
spec:
  rules:
  - host: api.localtest.me
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fleet-api
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f fleet-api-secret.yaml
kubectl apply -f fleet-api.yaml
kubectl apply -f fleet-api-ing.yaml
```

Smoke test:

```bash
curl -X POST http://api.localtest.me/ingest \
  -H "Authorization: Bearer change-me" \
  -H "Content-Type: application/json" \
  -d '{"vin":"WAUZZZFY0R1234567","ts":"2025-11-04T12:00:00Z","lat":49.5891,"lon":8.6527,"speed_kph":80,"rpm":2200}'
```

### 6) Grafana data source & dashboards

* Add an **InfluxDB** data source in Grafana:

  * URL: `http://influxdb-influxdb2.fleet.svc.cluster.local`
  * Auth: Token (`dev-token-please-change`)
  * Org: `fleet`, Bucket: `telemetry`
* Import the provided dashboards from `./dashboards` (Trip Overview, Live Map, Engine Health).

---

## Local development loop (WSL friendly)

* **k3d** shares Docker with your host; images built in WSL are visible to the cluster.
* Recommended: **Skaffold** for fast inner loop.

`skaffold.yaml` (excerpt):

```yaml
apiVersion: skaffold/v4beta11
kind: Config
build:
  artifacts:
    - image: ghcr.io/your-namespace/fleet-api
      context: services/fleet-api
deploy:
  kubectl:
    manifests:
    - k8s/fleet-api-secret.yaml
    - k8s/fleet-api.yaml
    - k8s/fleet-api-ing.yaml
```

Run:

```bash
skaffold dev
```

---

## Edge/phone client options

* **HTTP forwarder** (simple): a small script/app reads OBD-II via ELM327 and posts JSON to `/ingest`.
* **MQTT**: deploy Mosquitto in-cluster and run a light bridge in the API that subscribes to `telemetry/<vin>`.
* **Security**: generate a per-device API key and rotate regularly; prefer VPN (Tailscale) for non-LAN posting.

---

## Scaling & reliability

* Use an **HPA**:

```bash
kubectl -n fleet autoscale deploy fleet-api --cpu-percent=50 --min=2 --max=10
```

* Enable **resource requests/limits** (already set in the example).
* Consider a **queue** (Redis, Kafka) if you expect bursty uploads or multiple vehicles.
* Set **retention policies** in InfluxDB to bound disk usage.

---

## Observability

* Install `kube-prometheus-stack` to monitor the cluster itself.
* Add Grafana alerts (Telegram, email) for coolant over-temp, low battery voltage, or high engine load.

---

## GitOps (optional but recommended)

* Install **Argo CD** in `gitops` namespace.
* Track `k8s/` manifests in this repo; PRs drive changes.
* Keep secrets in **SOPS**-encrypted files or use **External Secrets** with a KMS/KeyVault.

---

## Security considerations

* Do **not** expose InfluxDB and Grafana publicly without auth.
* Store all credentials in K8s Secrets (or External Secrets).
* Prefer **TLS** with cert-manager and a DNS solver (e.g., Cloudflare DNS-01) if you intend to access remotely.
* Consider VIN hashing if sharing dashboards publicly.

---

## Repository layout (suggested)

```
/services
  /fleet-api            # Node.js ingestion service
  /edge-clients         # optional scripts for Pi/phone
/k8s
  fleet-api.yaml
  fleet-api-ing.yaml
  fleet-api-secret.yaml
  influxdb-values.yaml
  grafana-ing.yaml
/dashboards
  trip-overview.json
  live-telemetry.json
/tools
  skaffold.yaml
  makefile
```

---

## Roadmap

- [ ] MQTT ingestion path (Mosquitto + API subscriber)
- [ ] Trip segmentation & summaries (daily Job)
- [ ] Geo reverse-lookup (OSM/Nominatim) for trip endpoints
- [ ] Exporter to Parquet for offline analytics
- [ ] Optional TimescaleDB backend
- [ ] Mobile-friendly dashboard UI (React)
- [ ] Alerting (coolant over-temp, battery low, harsh braking)

---

## License

MIT (or choose your preferred license). See `LICENSE`.

---

## Disclaimer

Use responsibly. Do not log personally identifiable data beyond what you consent to.  
Comply with local laws regarding telemetry and data sharing.
