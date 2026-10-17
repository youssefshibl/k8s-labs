# Setup — Prometheus + Grafana

Shared Prometheus Operator stack: scrape metrics about **all pods** in the cluster, plus Grafana for dashboards. Used by [lab4](../../lab4/).

## Files

| Path | Purpose |
|------|---------|
| [`prometheus.yml`](prometheus.yml) | Prometheus + kube-state-metrics + ServiceMonitors + Ingress + metrics-demo |
| [`grafana/grafana.yml`](grafana/grafana.yml) | Grafana Deployment + datasource + Ingress |
| [`grafana/dashboards/`](grafana/dashboards/) | Dashboard JSON — Cluster Pods, Pods by Namespace, Workload Pods |
| [`mongodb/`](mongodb/) | MongoDB exporter + [Prod Overview dashboard](https://grafana.com/grafana/dashboards/25227-mongodb-prod-overview/) — see [mongodb/README.md](mongodb/README.md) |

## What this creates

| Resource       | Name                 | Notes                                                |
|----------------|----------------------|------------------------------------------------------|
| Namespace      | `monitoring`         | Prometheus + kube-state-metrics + Grafana            |
| Prometheus     | `prometheus`         | Scrapes all ServiceMonitors / PodMonitors + annotated pods |
| Service/Ingress| `prometheus`         | UI at `http://prometheus.test.local`                 |
| Deployment     | `kube-state-metrics` | Metrics **about every pod** (`kube_pod_*`)           |
| ServiceMonitor | `kube-state-metrics` | Tells Prometheus to scrape kube-state-metrics        |
| Deployment     | `metrics-demo`       | Tiny app with `/metrics` (annotation scrape demo)    |
| ServiceMonitor | `metrics-demo`       | Scrapes the demo app                                 |
| Deployment     | `grafana`            | Dashboards over Prometheus                           |
| Service/Ingress| `grafana`            | UI at `http://grafana.test.local` (admin / admin)    |
| Deployment     | `mongodb-exporter`   | Optional — MongoDB metrics (`mongodb/`)              |

## Architecture

```mermaid
flowchart LR
  Browser["Browser"]
  IC["ingress-nginx"]
  PromIng["Ingress<br/>prometheus.test.local"]
  GrafIng["Ingress<br/>grafana.test.local"]
  Prom["Prometheus<br/>monitoring"]
  Graf["Grafana"]
  KSM["kube-state-metrics"]
  Demo["metrics-demo<br/>/metrics"]
  AllPods["All cluster pods"]

  Browser --> IC
  IC --> PromIng --> Prom
  IC --> GrafIng --> Graf
  Graf -->|query| Prom
  Prom -->|ServiceMonitor| KSM
  KSM -.->|watches| AllPods
  Prom -->|annotation SD + ServiceMonitor| Demo
```

## How Prometheus gets data from all pods

1. **kube-state-metrics** watches the Kubernetes API and exports metrics for every pod (`kube_pod_info`, `kube_pod_status_phase`, `kube_pod_container_status_restarts_total`, …).
2. A **ServiceMonitor** points Prometheus at kube-state-metrics.
3. **Annotation scrape** (extra job in `additional-scrape-configs`): any pod with `prometheus.io/scrape: "true"` is scraped (see `metrics-demo`). Optional annotations: `prometheus.io/port`, `prometheus.io/path`.

## Prerequisites

- kind cluster (`kind-dev`)
- ingress-nginx controller
- MetalLB (LoadBalancer EXTERNAL-IP)
- Wildcard DNS for `*.test.local` (dnsmasq)

Plus the **Prometheus Operator** CRDs + controller (once per cluster):

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.78.2/bundle.yaml

kubectl wait --for=condition=Available deployment/prometheus-operator \
  -n default --timeout=120s
```

> The upstream `bundle.yaml` installs the operator into the `default` namespace. That is fine for local labs.

## Apply

From the repo root:

```bash
kubectl apply -f setup/prometheus/prometheus.yml

# Grafana — load JSON from grafana/dashboards/ (+ optional mongodb/) then deploy
kubectl create configmap grafana-dashboards -n monitoring \
  --from-file=setup/prometheus/grafana/dashboards/ \
  --from-file=setup/prometheus/mongodb/dashboards/ \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f setup/prometheus/grafana/grafana.yml

# Optional — MongoDB-specific metrics (needs lab2/lab4 MongoDB first)
kubectl apply -f setup/prometheus/mongodb/mongodb-exporter.yml
```

Add more boards by dropping `*.json` into `grafana/dashboards/` (or `mongodb/dashboards/`), then re-run the `kubectl create configmap …` command and `kubectl -n monitoring rollout restart deployment/grafana`.

MongoDB exporter details: [`mongodb/README.md`](mongodb/README.md).

Check status:

```bash
kubectl get prometheus,servicemonitor,podmonitor -A
kubectl get pods -n monitoring
kubectl get pods -l app=metrics-demo
```

Wait until Prometheus and Grafana are ready:

```bash
kubectl -n monitoring wait --for=condition=Available prometheus/prometheus --timeout=180s
kubectl -n monitoring get pods -l app.kubernetes.io/name=prometheus
kubectl -n monitoring wait --for=condition=Available deployment/grafana --timeout=120s
```

## Access

Prometheus:

```
http://prometheus.test.local
```

Grafana (login `admin` / `admin`):

```
http://grafana.test.local
```

Provisioned dashboards:

- **Cluster Pods** — overview of `kube_pod_*`
- **Pods by Namespace — Resources** — filter by namespace/pod; CPU, memory, requests/limits, network, filesystem, restarts
- **Workload Pods** — pick Deployment / StatefulSet / DaemonSet; lists its pods with CPU/memory/replicas
- **MongoDB — Prod Overview** — ops, latency, connections, WiredTiger, dbStats (needs `mongodb-exporter`)

Or build panels with the queries below.

### Useful queries (all pods)

In Prometheus → **Graph**:

```promql
# Every pod known to the API
kube_pod_info

# Pod phase (Running / Pending / …)
kube_pod_status_phase

# Restarts per container
kube_pod_container_status_restarts_total

# Demo app custom metric
version
```

### Targets

Open **Status → Targets** and confirm:

- `serviceMonitor/monitoring/kube-state-metrics` — UP
- `serviceMonitor/default/metrics-demo` — UP
- `serviceMonitor/default/mongodb-exporter` — UP (if MongoDB exporter applied)
- `kubernetes-pods` — UP for `metrics-demo` (annotation scrape)
- `kubelet-cadvisor` — UP (real CPU / memory / network / FS usage)

## Enable scraping on your own pods

Add annotations on the pod template:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

The container must serve Prometheus text format on that path/port.

## Cleanup

```bash
kubectl delete -f setup/prometheus/mongodb/mongodb-exporter.yml --ignore-not-found
kubectl delete configmap grafana-dashboards -n monitoring --ignore-not-found
kubectl delete -f setup/prometheus/grafana/grafana.yml
kubectl delete -f setup/prometheus/prometheus.yml
```

Optional — remove the Prometheus Operator itself:

```bash
kubectl delete -f \
  https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.78.2/bundle.yaml
```
