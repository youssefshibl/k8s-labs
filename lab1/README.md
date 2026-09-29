# Lab 1 — Test Deployment, Service & Ingress

Simple nginx app on a local kind cluster with MetalLB + nginx Ingress.

## What this creates

| Resource   | Name       | Notes                                      |
|------------|------------|--------------------------------------------|
| Deployment | `test-app` | 2 replicas, `nginx:1.27-alpine`            |
| Service    | `test-app` | ClusterIP, port 80                         |
| Ingress    | `test-app` | host `test-app.test.local`, class `nginx`  |

## Prerequisites

- kind cluster (`kind-dev`)
- ingress-nginx controller
- MetalLB (LoadBalancer EXTERNAL-IP)
- Wildcard DNS for `*.test.local` (dnsmasq) — see below

## Apply

```bash
kubectl apply -f main.yml
```

Check status:

```bash
kubectl get deploy,svc,ingress test-app
kubectl get pods -l app=test-app
```

## Get LoadBalancer IP

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

Or only the IP:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

Example on this cluster: `172.20.255.200`

## Wildcard DNS (`*.test.local`)

`/etc/hosts` does **not** support wildcards. Use dnsmasq.

```bash
# install full package (dnsmasq-base alone has no systemd unit)
sudo apt update
sudo apt install dnsmasq

sudo tee /etc/dnsmasq.d/test-local.conf <<'EOF'
address=/test.local/172.20.255.200
listen-address=127.0.0.1
bind-interfaces
EOF

sudo systemctl enable --now dnsmasq
sudo systemctl restart dnsmasq

sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/test-local.conf <<'EOF'
[Resolve]
DNS=127.0.0.1
Domains=~test.local
EOF
sudo systemctl restart systemd-resolved
```

Verify:

```bash
dig +short test-app.test.local @127.0.0.1
# expect: 172.20.255.200
```

## Access

Open in a browser:

```
http://test-app.test.local
```

Or with curl:

```bash
curl -I http://test-app.test.local
```

## Cleanup

```bash
kubectl delete -f main.yml
```
