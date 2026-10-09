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
| ClusterIssuer| `selfsigned-issuer` | cert-manager self-signed issuer (local `.local`) |
| Ingress      | `test-web`       | host `test-web.test.local`, TLS via cert-manager   |
| Secret       | `test-web-tls`   | TLS cert/key created by cert-manager               |

## Lab2 vs Lab3

| | Lab2 | Lab3 |
|---|------|------|
| MongoDB workload | Deployment | StatefulSet |
| MongoDB Service | ClusterIP | Headless (`clusterIP: None`) |
| Data | ephemeral (lost on pod delete) | PVC at `/data/db` (persists) |
| TLS | none | cert-manager + Ingress TLS |

## How TLS works

1. Ingress asks cert-manager for a cert (`cert-manager.io/cluster-issuer: selfsigned-issuer`).
2. cert-manager creates Secret `test-web-tls` for host `test-web.test.local`.
3. nginx Ingress terminates HTTPS using that secret.

Self-signed is used because Let's Encrypt cannot issue for `*.test.local`. The browser will warn until you accept the cert.

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
- **cert-manager** (see below)

### Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```

Wait until ready:

```bash
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s
kubectl get pods -n cert-manager
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

Open in a browser (accept the self-signed warning):

```
https://test-web.test.local
```

Or with curl (skip verify for the lab cert):

```bash
curl -kI https://test-web.test.local
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
