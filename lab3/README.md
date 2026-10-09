# Lab 3 — Web App + Stateful MongoDB + TLS

Same stack as lab2 (mongo-express + MongoDB), but MongoDB runs as a **StatefulSet** with a persistent volume, and the Ingress gets a TLS cert from **cert-manager** for `test-web.test.local`.

## What this creates

| Resource     | Name             | Notes                                              |
|--------------|------------------|----------------------------------------------------|
| Secret       | `mongodb-secret` | MongoDB username / password / database             |
| Service      | `mongodb`        | Headless (`clusterIP: None`), port 27017           |
| StatefulSet  | `mongodb`        | 1 replica, `mongo:7.0`, PVC at `/data/db`          |
| PVC          | `mongodb-data-mongodb-0` | 1Gi, created by `volumeClaimTemplates`      |
| Deployment   | `test-web`       | 2 replicas, `mongo-express` → connects to MongoDB  |
| Service      | `test-web`       | ClusterIP, port 80 → 8081                          |
| Ingress      | `test-web`       | host `test-web.test.local`, TLS via cert-manager   |
| Secret       | `test-web-tls`   | TLS cert/key signed by `labs-ca-issuer`            |

## Lab2 vs Lab3

| | Lab2 | Lab3 |
|---|------|------|
| MongoDB workload | Deployment | StatefulSet |
| MongoDB Service | ClusterIP | Headless (`clusterIP: None`) |
| Data | ephemeral (lost on pod delete) | PVC at `/data/db` (persists) |
| TLS | none | cert-manager + Ingress TLS |

## How TLS works

1. Ingress asks cert-manager for a cert (`cert-manager.io/cluster-issuer: labs-ca-issuer`).
2. cert-manager creates Secret `test-web-tls` for host `test-web.test.local`, signed by the labs CA.
3. nginx Ingress terminates HTTPS using that secret (HTTP redirects to HTTPS).

Let's Encrypt cannot issue for `*.test.local`, so labs use the shared local CA from [`setup/`](../setup/). Trust that CA on the host for browser/curl without warnings.

## How the app connects

`test-web` uses the in-cluster DNS name `mongodb` and credentials from `mongodb-secret`:

```
mongodb://admin:changeme@mongodb:27017/
```

Stable pod DNS (StatefulSet): `mongodb-0.mongodb.default.svc.cluster.local`

## Prerequisites

Same as lab1:

- kind cluster (`kind-dev`)
- ingress-nginx controller
- MetalLB (LoadBalancer EXTERNAL-IP)
- Wildcard DNS for `*.test.local` (dnsmasq)
- A default StorageClass (kind provides one)
- **cert-manager** + labs local CA — see [`setup/`](../setup/)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s
kubectl apply -f ../setup/local-ca.yml
```

## Apply

```bash
kubectl apply -f main.yml
```

Check status:

```bash
kubectl get secret,sts,svc,pvc,deploy,ingress,certificate,clusterissuer
kubectl get pods -l 'app in (mongodb,test-web)'
kubectl describe certificate -A
```

The TLS Secret should appear shortly:

```bash
kubectl get secret test-web-tls
kubectl get certificate
```

## Verify persistence

1. Open https://test-web.test.local and create a collection / document in `testdb`.
2. Delete the MongoDB pod:

```bash
kubectl delete pod mongodb-0
```

3. Wait for it to come back, then reopen the UI — your data should still be there.

```bash
kubectl get pvc mongodb-data-mongodb-0
kubectl wait --for=condition=ready pod/mongodb-0 --timeout=120s
```

## Access

Open in a browser:

```
https://test-web.test.local
```

Or with curl:

```bash
curl -I https://test-web.test.local
```

## Cleanup

```bash
kubectl delete -f main.yml
```

Note: deleting the StatefulSet does **not** always remove PVCs created by `volumeClaimTemplates`. Remove leftover volume claims if needed:

```bash
kubectl delete pvc -l app=mongodb
```

Optional — remove cert-manager itself:

```bash
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```
