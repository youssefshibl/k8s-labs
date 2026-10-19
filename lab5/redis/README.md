# Lab 5 — Redis + redis_exporter + Grafana dashboard

Part of [lab5](../README.md). Deploys Redis and scrapes DB metrics into Prometheus / Grafana.

Dashboard: custom **Redis** board for [oliver006/redis_exporter](https://github.com/oliver006/redis_exporter) (inspired by [grafana.com/763](https://grafana.com/grafana/dashboards/763-redis-dashboard-for-prometheus-redis-exporter-1-x/)) — overview stats, hit ratio, memory/CPU, clients, keyspace, and command latency.

## Files

| Path | Purpose |
|------|---------|
| [`redis-exporter.yml`](redis-exporter.yml) | Redis + exporter + ServiceMonitor |
| [`dashboards/redis-dashboard.json`](dashboards/redis-dashboard.json) | Grafana board 763 |

## Apply

See [lab5/README.md](../README.md). Quick path from repo root:

```bash
kubectl apply -f lab5/redis/redis-exporter.yml
kubectl create configmap grafana-dashboards-redis -n monitoring \
  --from-file=lab5/redis/dashboards/ \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring rollout restart deployment/grafana
```

## Cleanup

```bash
kubectl delete -f lab5/redis/redis-exporter.yml --ignore-not-found
kubectl delete configmap grafana-dashboards-redis -n monitoring --ignore-not-found
kubectl -n monitoring rollout restart deployment/grafana
```
