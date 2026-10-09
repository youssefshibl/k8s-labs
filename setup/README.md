# Labs setup — local CA

Shared cert-manager CA used by labs that need TLS (e.g. lab3). Apply once per cluster.

## Prerequisites

- kind cluster (`kind-dev`)
- **cert-manager** installed and ready

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s
```

## Apply

From the repo root:

```bash
kubectl apply -f setup/local-ca.yml
```

Check:

```bash
kubectl get clusterissuer selfsigned-bootstrap labs-ca-issuer
kubectl get certificate labs-ca -n cert-manager
kubectl get secret labs-ca-secret -n cert-manager
```

`labs-ca-issuer` should be `Ready=True`.

## Trust the CA on the host (browser / curl)

Without this step, HTTPS works at the Ingress but the browser shows a certificate error
(the site cert is signed by `k8slabs-ca`, not a public CA).

**Chrome / Chromium** (user NSS DB — no sudo):

```bash
kubectl get secret labs-ca-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/k8slabs-ca.crt

certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "k8slabs-ca" -i /tmp/k8slabs-ca.crt
```

Fully quit and reopen the browser, then open `https://test-web.test.local`.

**System trust** (curl / other tools — needs sudo):

```bash
kubectl get secret labs-ca-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d \
  | sudo tee /usr/local/share/ca-certificates/k8slabs-ca.crt >/dev/null

sudo update-ca-certificates
```

Verify TLS to the Ingress IP (avoids local DNS quirks):

```bash
curl -I --resolve test-web.test.local:443:172.20.255.200 https://test-web.test.local
```

## What this creates

| Resource      | Name                   | Notes                                      |
|---------------|------------------------|--------------------------------------------|
| ClusterIssuer | `selfsigned-bootstrap` | Issues the CA certificate once             |
| Certificate   | `labs-ca`              | CA cert in `cert-manager` namespace         |
| Secret        | `labs-ca-secret`       | CA cert + key                              |
| ClusterIssuer | `labs-ca-issuer`       | Signs Ingress / app certs for the labs     |

Labs reference the issuer with:

```yaml
cert-manager.io/cluster-issuer: labs-ca-issuer
```

## Cleanup

```bash
kubectl delete -f setup/local-ca.yml
sudo rm -f /usr/local/share/ca-certificates/k8slabs-ca.crt
sudo update-ca-certificates
```
