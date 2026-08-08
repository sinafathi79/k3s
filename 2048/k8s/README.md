# 2048 on K3s

The classic [2048](https://github.com/gabrielecirulli/2048) game, containerized and deployed on K3s behind an NGINX Ingress Controller with a self-signed TLS certificate.

## Stack

- **K3s** — Traefik disabled (`--disable=traefik`)
- **F5 NGINX Ingress Controller** (`nginx/kubernetes-ingress`) — handles routing and TLS termination. Note: this is not the same project as the older `kubernetes/ingress-nginx`, which was retired in March 2026 and no longer receives security updates.
- **Self-signed certificate** — for HTTPS on a local/test domain
- Namespace: `game-2048`

## Project structure

```
.
├── Dockerfile
├── k8s/
│   ├── namespace.yaml    # the game-2048 namespace
│   ├── deployment.yaml   # 2 replicas of the game container
│   ├── service.yaml      # ClusterIP service in front of the pods
│   └── ingress.yaml      # host + TLS rule, routes 2048.local -> service
└── README.md
```

## Prerequisites

- A K3s cluster with Traefik disabled
- Helm 3.19+ on the K3s host
- Docker (or another OCI builder) to build the image

## Setup

### 1. Install the ingress controller

```bash
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress \
  --version 2.6.4 \
  --namespace nginx-ingress \
  --create-namespace
```

Verify:

```bash
kubectl get pods -n nginx-ingress
kubectl get ingressclasses   # expect "nginx" / nginx.org/ingress-controller
```

### 2. Create the namespace

```bash
kubectl apply -f k8s/namespace.yaml
```

### 3. Build the image and get it into the cluster

Single-node K3s (no registry needed):

```bash
docker build -t game-2048:latest .
docker save game-2048:latest -o game-2048.tar
sudo k3s ctr images import game-2048.tar
```

Multi-node cluster, or if you'd rather use a registry:

```bash
docker build -t <your-registry>/game-2048:latest .
docker push <your-registry>/game-2048:latest
```

If you go the registry route, update `image:` in `k8s/deployment.yaml` and set `imagePullPolicy: IfNotPresent`.

### 4. Generate the self-signed cert and TLS secret

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=2048.local/O=game2048" \
  -addext "subjectAltName=DNS:2048.local"

kubectl create secret tls game-2048-tls \
  --cert=tls.crt --key=tls.key \
  -n game-2048
```

> `tls.key`, `tls.crt`, and `game-2048.tar` are gitignored — never commit private key material to the repo.

### 5. Deploy

```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
kubectl rollout status deployment/game-2048 -n game-2048
```

### 6. Test

```bash
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
curl -kv --resolve 2048.local:443:$NODE_IP https://2048.local/
```

To play it in a browser, add to your hosts file:

```
<NODE_IP>  2048.local
```

Then visit `https://2048.local` (your browser will warn about the self-signed cert — that's expected).

## Configuration you'll likely want to change

| What | Where | Notes |
|---|---|---|
| Container port | `deployment.yaml` (`containerPort`), `service.yaml` (`targetPort`) | Must match whatever port your Dockerfile actually serves on |
| Host/domain | `ingress.yaml` (`spec.tls.hosts`, `spec.rules.host`) | Currently `2048.local` |
| Image source | `deployment.yaml` (`image`, `imagePullPolicy`) | Local import vs. registry, per step 3 above |
| Namespace | all four manifests | Currently `game-2048` |

## Troubleshooting

**502 Bad Gateway** — TLS/ingress is working, but the Service has no healthy pod to forward to. Check:
```bash
kubectl get pods -n game-2048 -o wide
kubectl get endpoints -n game-2048 game-2048   # empty = no ready pods
kubectl logs -n game-2048 -l app=game-2048
```
Usually caused by a container-port mismatch failing the readiness probe.

**Pods stuck `Pending`/`ImagePullBackOff`** — for local-import setups, confirm the image was imported on the same node the pod was scheduled to (`k3s ctr images ls | grep game-2048`), or push to a registry instead.

**Ingress has no address / 404** — confirm `kubectl get ingressclasses` shows `nginx`, and that `ingressClassName: nginx` in `ingress.yaml` matches.

**Cert warnings in browser** — expected with a self-signed cert; click through, or add `tls.crt` to your OS/browser trust store if you want to remove the warning.

## Cleanup

Remove just the app (leaves the ingress controller running, e.g. if you're hosting other services on it):

```bash
kubectl delete namespace game-2048
```

This deletes the namespace and everything in it — Deployment, Service, Ingress, and the `game-2048-tls` secret — in one shot.

Remove the NGINX Ingress Controller too, if you're tearing down the whole setup:

```bash
helm uninstall nginx-ingress -n nginx-ingress
kubectl delete namespace nginx-ingress
```

Remove local build artifacts (cert/key and the saved image tarball):

```bash
rm -f game-2048.tar tls.crt tls.key
```

Remove the imported image from K3s's containerd store (check the exact reference first, since `ctr` needs the full name):

```bash
sudo k3s ctr images ls | grep game-2048
sudo k3s ctr images rm docker.io/library/game-2048:latest
```
