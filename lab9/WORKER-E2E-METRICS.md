# Worker e2e metrics export

How `stream-worker` measures **stream enter → XACK** latency and exposes it on Prometheus `/metrics`.

Code lives in ConfigMap `stream-worker` → `worker.py` inside [`main.yml`](main.yml).

## Why the worker (not redis_exporter)

| Source | Has enter time? | Has ACK time? |
|--------|-----------------|---------------|
| Redis / `stream-redis-exporter` | Yes (stream id / message fields via XINFO) | **No** — `XACK` only clears the PEL |
| `stream-worker` | Yes (`produced_at` or stream id) | **Yes** — `time.time()` at ACK |

E2e latency must be observed in the app at ACK time.

## Endpoint

| | |
|--|--|
| Path | `/metrics` |
| Port | `9102` (`METRICS_PORT`) |
| Service | `stream-worker` ClusterIP |
| URL | `http://stream-worker.default.svc:9102/metrics` |
| Format | Prometheus text (`prometheus_client`) |

Started in `main()`:

```python
start_http_server(METRICS_PORT)  # default 9102 → GET /metrics
```

Kubernetes wiring ([`main.yml`](main.yml)):

- Container port `metrics` → `9102`
- Service port `9102` → targetPort `metrics`
- ServiceMonitor scrapes path `/metrics` every **15s**
- Readiness probe: `GET /metrics` on port `metrics`

### Smoke check

```bash
kubectl run curl-tmp --rm -i --restart=Never --image=curlimages/curl:8.5.0 -- \
  curl -s http://stream-worker.default.svc:9102/metrics | grep -E 'stream_message_e2e|stream_messages_acked'
```

## Exported series

| Metric | Type | Labels | Meaning |
|--------|------|--------|---------|
| `stream_message_e2e_latency_seconds` | Histogram | `stream`, `group` | Seconds from enter until `XACK` |
| `stream_messages_acked_total` | Counter | `stream`, `group` | Messages ACKed by this worker |

Histogram buckets (seconds): `0.5, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610`.

## Code walkthrough

### 1. Metric objects

```python
from prometheus_client import Counter, Histogram, start_http_server

E2E_LATENCY = Histogram(
    "stream_message_e2e_latency_seconds",
    "Seconds from message entering the Redis stream until ACK by this consumer group",
    labelnames=("stream", "group"),
    buckets=(0.5, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610),
)
ACKED = Counter(
    "stream_messages_acked_total",
    "Messages ACKed by this worker for a consumer group",
    labelnames=("stream", "group"),
)
```

### 2. Enter time (producer + worker)

Producer (`stream-api`) writes `produced_at` on **XADD**:

```python
fields = {
    "id": msg_id,
    "body": str(body),
    "produced_by": HOSTNAME,
    "produced_at": str(time.time()),
}
entry_id = r.xadd(STREAM, fields)
```

Worker resolves enter time:

```python
def message_enter_time(entry_id, fields):
    """When the message entered the stream (seconds since epoch)."""
    produced = fields.get("produced_at")
    if produced is not None:
        try:
            return float(produced)
        except (TypeError, ValueError):
            pass
    # Fallback: Redis stream IDs are "<unix_ms>-<seq>"
    try:
        return int(str(entry_id).split("-", 1)[0]) / 1000.0
    except (TypeError, ValueError):
        return None
```

### 3. Observe at ACK

```python
def observe_e2e(messages, acked_at):
    for entry_id, fields in messages:
        t0 = message_enter_time(entry_id, fields or {})
        if t0 is None:
            continue
        latency = acked_at - t0
        if latency < 0:
            continue
        E2E_LATENCY.labels(stream=STREAM, group=GROUP).observe(latency)
        ACKED.labels(stream=STREAM, group=GROUP).inc()
```

### 4. Batch process → sleep → XACK → observe

```python
def process_batch(messages):
    ids = [entry_id for entry_id, _fields in messages]
    time.sleep(PROCESS_SECONDS)          # simulate work (whole batch)
    acked_at = time.time()               # ACK clock — not stored in Redis
    if ids:
        r.xack(STREAM, GROUP, *ids)
    observe_e2e(messages, acked_at)
```

Flow:

```text
XREADGROUP (or XAUTOCLAIM)
    → sleep(PROCESS_SECONDS)
    → acked_at = now()
    → XACK
    → observe(acked_at − enter) per message
```

### 5. Main loop starts the HTTP server first

```python
def main():
    start_http_server(METRICS_PORT)
    # … ensure group, then loop: xreadgroup → process_batch …
```

## Example scrape output

After messages have been ACKed:

```text
# HELP stream_message_e2e_latency_seconds Seconds from message entering the Redis stream until ACK by this consumer group
# TYPE stream_message_e2e_latency_seconds histogram
stream_message_e2e_latency_seconds_bucket{stream="jobs",group="workers",le="0.5"} 0.0
stream_message_e2e_latency_seconds_bucket{stream="jobs",group="workers",le="3.0"} 12.0
...
stream_message_e2e_latency_seconds_sum{stream="jobs",group="workers"} 45.2
stream_message_e2e_latency_seconds_count{stream="jobs",group="workers"} 12.0

# HELP stream_messages_acked_total Messages ACKed by this worker for a consumer group
# TYPE stream_messages_acked_total counter
stream_messages_acked_total{stream="jobs",group="workers"} 12.0
```

## Grafana / PromQL

Dashboard panels use job `stream-worker`, e.g.:

```promql
# average e2e
sum(stream_message_e2e_latency_seconds_sum{job="stream-worker", stream=~"$stream", group=~"$group"})
  /
sum(stream_message_e2e_latency_seconds_count{job="stream-worker", stream=~"$stream", group=~"$group"})

# p95
histogram_quantile(0.95,
  sum by (le) (rate(stream_message_e2e_latency_seconds_bucket{job="stream-worker", stream=~"$stream", group=~"$group"}[5m]))
)
```

See [README.md](README.md) for full lab setup and the Redis Streams board.
