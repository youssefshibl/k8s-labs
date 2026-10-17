# k8slabs

Local Kubernetes labs on kind (`kind-dev`) with MetalLB + nginx Ingress.

| Lab | Description |
|-----|-------------|
| [lab1](lab1/) | Test nginx Deployment, Service & Ingress |
| [lab2](lab2/) | Web app (mongo-express) connected to MongoDB |
| [lab3](lab3/) | Stateful MongoDB + TLS via cert-manager (`test-web.test.local`) |
| [lab4](lab4/) | Lab2 stack + Prometheus Operator (metrics for all pods) |

## Architecture overview

```mermaid
flowchart TB
  subgraph shared [Shared cluster]
    DNS["dnsmasq *.test.local"]
    LB["MetalLB"]
    IC["ingress-nginx"]
    CM["cert-manager + setup/ local CA"]
    Prom["setup/prometheus/"]
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
    L4Ing["Ingress test-web"]
    L4Web["mongo-express + mongodb<br/>same as lab2"]
    L4Ing --> L4Web
  end

  DNS --> LB --> IC
  IC --> L1Ing
  IC --> L2Ing
  IC --> L3Ing
  IC --> L4Ing
  CM -.-> L3Ing
  Prom -.->|metrics about| L4Web
```

## Setup (shared)

One-time / shared cluster pieces used by labs:

| Path | Purpose |
|------|---------|
| [setup/local-ca.yml](setup/local-ca.yml) | Local CA + `labs-ca-issuer` (cert-manager) — lab3, dashboard |
| [setup/prometheus/](setup/prometheus/) | Prometheus + Grafana — [README](setup/prometheus/README.md) (lab4) |
| [setup/dashboard/](setup/dashboard/) | Kubernetes Dashboard Ingress — [README](setup/dashboard/README.md) |

```bash
# after cert-manager is installed (lab3 TLS / dashboard TLS)
kubectl apply -f setup/local-ca.yml

# after prometheus-operator bundle is installed (lab4)
kubectl apply -f setup/prometheus/prometheus.yml
kubectl create configmap grafana-dashboards -n monitoring \
  --from-file=setup/prometheus/grafana/dashboards/ \
  --from-file=setup/prometheus/mongodb/dashboards/ \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f setup/prometheus/grafana/grafana.yml
# optional MongoDB metrics: setup/prometheus/mongodb/

# dashboard UI (after Dashboard + local CA are installed)
kubectl apply -f setup/dashboard/admin-user.yml
kubectl apply -f setup/dashboard/ingress.yml
```

See [setup/README.md](setup/README.md) for CA trust, Prometheus, and Dashboard details.

---

## Kubernetes Dashboard

Web UI at `https://dashboard.test.local` — full install and Ingress steps:

→ **[setup/dashboard/README.md](setup/dashboard/README.md)**
