# Lab 3 — Web App + Stateful MongoDB + TLS

Same stack as lab2 (mongo-express + MongoDB), but MongoDB runs as a **StatefulSet** with a persistent volume, and the Ingress gets a TLS cert from **cert-manager** for `test-web.test.local`.

## What this creates

| Resource     | Name             | Notes                                              |
|--------------|------------------|----------------------------------------------------|
| Secret       | `mongodb-secret` | MongoDB username / password / database             |
| Service      | `mongodb`        | Headless (`clusterIP: None`), port 27017           |
| StatefulSet  | `mongodb`        | 1 replica, `mongo:7.0`, PVC at `/data/db`          |
| PVC          | `mongodb-data-mongodb-0` | 1Gi, created by `volumeClaimTemplates`      |
| PVC          | `mongodb-backup` | 2Gi, separate volume for daily dumps               |
| CronJob      | `mongodb-backup` | `mongodump` every day at 02:00 → backup PVC        |
| Deployment   | `test-web`       | 2 replicas, `mongo-express` → connects to MongoDB  |
| Service      | `test-web`       | ClusterIP, port 80 → 8081                          |
| Ingress      | `test-web`       | host `test-web.test.local`, TLS via cert-manager   |
| Secret       | `test-web-tls`   | TLS cert/key signed by `labs-ca-issuer`            |

## Architecture

```mermaid
flowchart TB
  Browser["Browser / curl<br/>HTTPS"]
  IC["ingress-nginx"]
  Ing["Ingress<br/>test-web.test.local"]
  TLS["Secret<br/>test-web-tls"]
  CM["cert-manager<br/>labs-ca-issuer"]
  WebSvc["Service<br/>test-web :80"]
  Web["Deployment<br/>test-web ×2<br/>mongo-express"]
  Secret["Secret<br/>mongodb-secret"]
  DbSvc["Service headless<br/>mongodb :27017"]
  Db["StatefulSet<br/>mongodb-0"]
  DataPVC["PVC<br/>mongodb-data"]
  Cron["CronJob<br/>mongodb-backup<br/>daily 02:00"]
  BackupPVC["PVC<br/>mongodb-backup"]

  Browser --> IC --> Ing
  CM -.->|issues| TLS
  TLS -.->|TLS terminate| Ing
  Ing --> WebSvc --> Web
  Web -->|"mongodb://…@mongodb:27017/"| DbSvc --> Db
  Db --> DataPVC
  Secret -.->|credentials| Web
  Secret -.->|credentials| Db
  Secret -.->|credentials| Cron
  Cron -->|"mongodump"| DbSvc
  Cron --> BackupPVC
```

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
kubectl get secret,sts,svc,pvc,deploy,cronjob,ingress,certificate,clusterissuer
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

## Daily MongoDB backup

`mongodb-backup` CronJob runs **every day at 02:00** (`0 2 * * *`), dumps the database with `mongodump`, and writes timestamped folders to the **`mongodb-backup` PVC** (separate from the live data PVC).

After a run finishes, the Job pod goes to **Completed** (not deleted). That is normal — the process already exited. Jobs are removed automatically after `ttlSecondsAfterFinished: 120` (2 minutes).

Check the CronJob:

```bash
kubectl get cronjob mongodb-backup
kubectl get pvc mongodb-backup
```

Trigger a backup immediately (instead of waiting for 02:00):

```bash
kubectl create job --from=cronjob/mongodb-backup mongodb-backup-manual
kubectl wait --for=condition=complete job/mongodb-backup-manual --timeout=120s
kubectl logs job/mongodb-backup-manual
```

### Access / download backups

The dumps live only on the `mongodb-backup` PVC. Start a short-lived helper pod that mounts it, then copy files out with `kubectl cp`.

```bash
# 1) Helper pod with the backup volume mounted
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: mongodb-backup-access
spec:
  containers:
    - name: access
      image: busybox:1.36
      command: ["sleep", "3600"]
      volumeMounts:
        - name: backup
          mountPath: /backup
  volumes:
    - name: backup
      persistentVolumeClaim:
        claimName: mongodb-backup
  restartPolicy: Never
EOF

kubectl wait --for=condition=Ready pod/mongodb-backup-access --timeout=60s

# 2) List dumps
kubectl exec mongodb-backup-access -- ls -la /backup

# 3) Download one dump folder to your machine (change the folder name)
kubectl cp mongodb-backup-access:/backup/mongodb-20261009-120715 ./mongodb-20261009-120715

# Or pack everything into a tarball first, then copy
kubectl exec mongodb-backup-access -- tar -czf /tmp/mongodb-backups.tar.gz -C /backup .
kubectl cp mongodb-backup-access:/tmp/mongodb-backups.tar.gz ./mongodb-backups.tar.gz

# 4) Clean up the helper
kubectl delete pod mongodb-backup-access
```

Restore example (into the running MongoDB, from a local dump folder):

```bash
kubectl cp ./mongodb-20261009-120715 mongodb-0:/tmp/restore
kubectl exec -it mongodb-0 -- mongorestore \
  --username=admin --password=changeme --authenticationDatabase=admin \
  /tmp/restore
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
kubectl delete pvc mongodb-backup
```


Optional — remove cert-manager itself:

```bash
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```
