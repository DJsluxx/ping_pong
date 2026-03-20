# ping-pong

Ping-pong HTTP server deployed on Kubernetes (Kind) with Helm and CI/CD.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/) (for Helm-based deploy)

## 1. Create the Kind cluster

```bash
kind create cluster --name echo-pong --config kind-cluster.yaml
```

This creates a single-node cluster with ports 80/443 mapped to localhost for ingress.

## 2. Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Wait for it to be ready:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

## 3. Deploy (pick one)

### Option A: Raw manifests

```bash
kubectl apply -f k8s/
```

This creates the `ping-pong` namespace, secret, deployment (2 replicas), service, and ingress.

### Option B: Helm

```bash
helm install ping-pong helm/ping-pong --create-namespace -n ping-pong
```

To customize values:

```bash
helm install ping-pong helm/ping-pong \
  --create-namespace -n ping-pong \
  --set image.tag="7.0.0" \
  --set secretToken="your-token"
```

## 4. Configure DNS

Add to your hosts file (`/etc/hosts` or `C:\Windows\System32\drivers\etc\hosts`):

```
127.0.0.1 ping-pong.local
```

## 5. Verify

```bash
# health check (no auth)
curl http://ping-pong.local/health

# ping endpoint (auth required)
# Note that this secret is a placeholder, and in production it will be under a secret manager.
curl -H "Authorization: Bearer pp-auth-x346x23y6453yy" http://ping-pong.local/ping

# check pods
kubectl get pods -n ping-pong
```

## Teardown

```bash
# delete the app
kubectl delete -f k8s/
# or
helm uninstall ping-pong -n ping-pong

# delete the cluster
kind delete cluster --name echo-pong
```

## CI/CD

Pushing a `v*` tag triggers the release workflow:

1. **test** — build and run tests
2. **scan** — Trivy security scan on the Docker image
3. **docker** — push multi-arch image to GHCR
4. **binaries** — cross-compile for linux/darwin/windows
5. **release** — GitHub release + Helm chart version bump
6. **cleanup** — Stale GHCR images are cleaned weekly by the cleanup workflow.

## Project structure

```
echo-pong/
├── main.go                          # Go application entrypoint
├── go.mod                           # Go module definition
├── Dockerfile                       # Multi-stage Docker build
├── kind-cluster.yaml                # Kind cluster configuration
│
├── k8s/                             # Raw Kubernetes manifests
│   ├── namespace.yaml
│   ├── secret.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
│
├── helm/
│   └── ping-pong/                   # Helm chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           └── secret.yaml
│
└── .github/
    └── workflows/
        ├── release.yaml             # CI/CD pipeline - on tag creation (test → scan → build → release)
        ├── ci.yaml                  # CI/CD pipeline - on PR to main (test → scan → lint)
        └── cleanup.yaml             # Weekly GHCR image cleanup
```