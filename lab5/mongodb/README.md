# Lab 5 — MongoDB exporter + Grafana dashboard

Part of [lab5](../README.md). Scrapes MongoDB-specific metrics into Prometheus / Grafana.

Dashboard: [MongoDB — Prod Overview (25227)](https://grafana.com/grafana/dashboards/25227-mongodb-prod-overview/) for [gurjarchetan/mongodb-exporter](https://github.com/gurjarchetan/mongodb-exporter).

## Files

| Path | Purpose |
|------|---------|
| [`mongodb.yml`](mongodb.yml) | MongoDB Secret + Deployment + Service |
| [`mongodb-exporter.yml`](mongodb-exporter.yml) | Exporter Secret + Deployment + Service + ServiceMonitor |
| [`dashboards/mongodb-prod-overview.json`](dashboards/mongodb-prod-overview.json) | Grafana board 25227 |

## Apply

See [lab5/README.md](../README.md). Quick path from repo root:

```bash
kubectl apply -f lab5/mongodb/mongodb.yml
kubectl apply -f lab5/mongodb/mongodb-exporter.yml
kubectl create configmap grafana-dashboards-mongodb -n monitoring \
  --from-file=lab5/mongodb/dashboards/ \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring rollout restart deployment/grafana
```

## Cleanup

```bash
kubectl delete -f lab5/mongodb/mongodb-exporter.yml --ignore-not-found
kubectl delete -f lab5/mongodb/mongodb.yml --ignore-not-found
kubectl delete configmap grafana-dashboards-mongodb -n monitoring --ignore-not-found
kubectl -n monitoring rollout restart deployment/grafana
```
