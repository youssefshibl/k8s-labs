# k8slabs

Local Kubernetes labs on kind (`kind-dev`) with MetalLB + nginx Ingress.

| Lab | Description |
|-----|-------------|
| [lab1](lab1/) | Test nginx Deployment, Service & Ingress |
| [lab2](lab2/) | Web app (mongo-express) connected to MongoDB |
| [lab3](lab3/) | Stateful MongoDB + TLS via cert-manager (`test-web.test.local`) |
| [lab4](lab4/) | Prometheus + Grafana for Kubernetes (3 pod/workload dashboards) |
| [lab5](lab5/) | MongoDB & PostgreSQL exporters + Grafana DB dashboards |

## Architecture overview

```mermaid
flowchart TB
  subgraph shared [Shared cluster]
    DNS["dnsmasq *.test.local"]
    LB["MetalLB"]
    IC["ingress-nginx"]
    CM["cert-manager + setup/ local CA"]
  end

  subgraph lab1 [Lab 1]
    L1Ing["Ingress test-app"]
    L1Svc["Service test-app"]
    L1Dep["Deployment nginx"]
    L1Ing --> L1Svc --> L1Dep
  end

  subgraph lab2 [Lab 2]
    L2Ing["Ingress test-web"]
    L2Web["Service + Deployment<br/>mongo-express"]
    L2Db["Service + Deployment<br/>mongodb"]
    L2Ing --> L2Web --> L2Db
  end

  subgraph lab3 [Lab 3]
    L3Ing["Ingress test-web + TLS"]
    L3Web["Service + Deployment<br/>mongo-express"]
    L3Db["Headless Service + StatefulSet<br/>mongodb + data PVC"]
    L3Cron["CronJob backup → backup PVC"]
    L3Ing --> L3Web --> L3Db
    L3Cron --> L3Db
  end

  subgraph lab4 [Lab 4]
    Prom["Prometheus + Grafana<br/>kube-state-metrics"]
    Boards["3 dashboards<br/>cluster / namespace / workload"]
    Prom --> Boards
  end

  subgraph lab5 [Lab 5]
    L5Mongo["mongodb + mongodb-exporter"]
    L5PG["postgresql + postgres-exporter"]
    L5Mongo --> Prom
    L5PG --> Prom
  end

  DNS --> LB --> IC
  IC --> L1Ing
  IC --> L2Ing
  IC --> L3Ing
  CM -.-> L3Ing
```

## Setup (shared)

One-time / shared cluster pieces used by labs:

| Path | Purpose |
|------|---------|
| [setup/local-ca.yml](setup/local-ca.yml) | Local CA + `labs-ca-issuer` (cert-manager) — lab3, dashboard |
| [setup/dashboard/](setup/dashboard/) | Kubernetes Dashboard Ingress — [README](setup/dashboard/README.md) |

Prometheus + Grafana live in **[lab4](lab4/)** (not under `setup/`).

```bash
# after cert-manager is installed (lab3 TLS / dashboard TLS)
kubectl apply -f setup/local-ca.yml

# Prometheus + Grafana (lab4) — after prometheus-operator bundle is installed
kubectl apply -f lab4/prometheus.yml
kubectl create configmap grafana-dashboards -n monitoring \
  --from-file=lab4/grafana/dashboards/ \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f lab4/grafana/grafana.yml

# lab5 — MongoDB / PostgreSQL exporters + DB dashboards
# see lab5/README.md

# dashboard UI (after Dashboard + local CA are installed)
kubectl apply -f setup/dashboard/admin-user.yml
kubectl apply -f setup/dashboard/ingress.yml
```

See [setup/README.md](setup/README.md) for CA trust and Dashboard details.

---

## Kubernetes Dashboard

Web UI at `https://dashboard.test.local` — full install and Ingress steps:

→ **[setup/dashboard/README.md](setup/dashboard/README.md)**
