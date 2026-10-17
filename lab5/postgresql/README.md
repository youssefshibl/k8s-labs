# Lab 5 — PostgreSQL + postgres_exporter + Grafana dashboard

Part of [lab5](../README.md). Deploys PostgreSQL and scrapes DB metrics into Prometheus / Grafana.

Dashboard: [PostgreSQL Database (9628)](https://grafana.com/grafana/dashboards/9628-postgresql-database/) for [prometheuscommunity/postgres-exporter](https://github.com/prometheus-community/postgres_exporter) — converted to Grafana 11 panels.

## Files

| Path | Purpose |
|------|---------|
| [`postgres-exporter.yml`](postgres-exporter.yml) | PostgreSQL + Secret + exporter + ServiceMonitor |
| [`dashboards/postgresql-database.json`](dashboards/postgresql-database.json) | Grafana board 9628 |

## Apply

See [lab5/README.md](../README.md). Quick path from repo root:

```bash
kubectl apply -f lab5/postgresql/postgres-exporter.yml
kubectl create configmap grafana-dashboards-postgresql -n monitoring \
  --from-file=lab5/postgresql/dashboards/ \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring rollout restart deployment/grafana
```

## Cleanup

```bash
kubectl delete -f lab5/postgresql/postgres-exporter.yml --ignore-not-found
kubectl delete configmap grafana-dashboards-postgresql -n monitoring --ignore-not-found
kubectl -n monitoring rollout restart deployment/grafana
```
