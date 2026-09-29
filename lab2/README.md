# Lab 2 — Web App + MongoDB

Deploy a test web app (mongo-express) that connects to MongoDB on a local kind cluster.

## What this creates

| Resource   | Name             | Notes                                              |
|------------|------------------|----------------------------------------------------|
| Secret     | `mongodb-secret` | MongoDB username / password / database             |
| Deployment | `mongodb`        | 1 replica, `mongo:7.0`                             |
| Service    | `mongodb`        | ClusterIP, port 27017                              |
| Deployment | `test-web`       | 2 replicas, `mongo-express` → connects to MongoDB  |
| Service    | `test-web`       | ClusterIP, port 80 → 8081                          |
| Ingress    | `test-web`       | host `test-web.test.local`, class `nginx`          |

## How the app connects

`test-web` uses the in-cluster DNS name `mongodb` and credentials from `mongodb-secret`:

```
mongodb://admin:changeme@mongodb:27017/
```

## Prerequisites

Same as lab1:

- kind cluster (`kind-dev`)
- ingress-nginx controller
- MetalLB (LoadBalancer EXTERNAL-IP)
- Wildcard DNS for `*.test.local` (dnsmasq)

## Apply

```bash
kubectl apply -f main.yml
```

Check status:

```bash
kubectl get deploy,svc,ingress,secret mongodb test-web mongodb-secret
kubectl get pods -l 'app in (mongodb,test-web)'
```

## Access

Open in a browser:

```
http://test-web.test.local
```

Or with curl:

```bash
curl -I http://test-web.test.local
```

You should see the mongo-express UI and be able to browse the `testdb` database.

## Cleanup

```bash
kubectl delete -f main.yml
```
