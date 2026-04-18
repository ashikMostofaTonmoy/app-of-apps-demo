# App of Apps Pattern — Online Boutique Demo

Source images: https://github.com/GoogleCloudPlatform/microservices-demo (v0.10.5)
Registry: `us-central1-docker.pkg.dev/google-samples/microservices-demo/` (public, no auth)

## Architecture

One root app → 5 child apps → 11 microservices → 1 namespace (`onlineboutique`)

```text
root-app (watches /apps)
├── frontend        → frontend (Go HTTP) + LoadBalancer service
├── cartservice     → cartservice (gRPC:7070) + redis-cart (TCP:6379)
├── checkoutservice → checkoutservice (gRPC:5050)
├── catalogservice  → productcatalogservice (gRPC:3550)
│                     currencyservice (gRPC:7000)
│                     recommendationservice (gRPC:8080)
└── orderservice    → emailservice (gRPC port 5000→8080)
                      paymentservice (gRPC:50051)
                      shippingservice (gRPC:50051)
                      adservice (gRPC:9555)
```

All services in same namespace — short DNS names resolve (`cartservice:7070`, not FQDN).

## Folder Structure

```text
app-of-apps-demo/
├── root-app.yaml                        # Deploy ONLY this to ArgoCD
├── apps/                                # Root watches this folder
│   ├── frontend-app.yaml
│   ├── cart-app.yaml
│   ├── checkout-app.yaml
│   ├── catalog-app.yaml
│   └── order-app.yaml
└── charts/
    ├── frontend/                        # frontend + 2 services (ClusterIP + LoadBalancer)
    ├── cartservice/                     # cartservice + redis-cart
    ├── checkoutservice/                 # checkoutservice only
    ├── catalogservice/                  # productcatalog + currency + recommendation
    └── orderservice/                    # email + payment + shipping + ad
```

## How to Deploy

### Step 1 — Update repo URL (4 files)

Replace `https://github.com/ashikMostofaTonmoy/app-of-apps-demo.git` in:
- `root-app.yaml`
- `apps/frontend-app.yaml`
- `apps/cart-app.yaml`
- `apps/checkout-app.yaml`
- `apps/catalog-app.yaml`
- `apps/order-app.yaml`

### Step 2 — Push to Git

```bash
git add argocd/app-of-apps-demo/
git commit -m "add app-of-apps online boutique demo"
git push
```

### Step 3 — Deploy the root app (one command!)

```bash
kubectl apply -f root-app.yaml -n argocd
```

### Step 4 — Watch ArgoCD deploy everything

```bash
# Watch apps appear in argocd
kubectl get applications -n argocd -w

# Watch pods come up
kubectl get pods -n onlineboutique -w
```

### Step 5 — Open the shop

```bash
# Get the LoadBalancer IP
kubectl get svc frontend-external -n onlineboutique

# Open in browser: http://<EXTERNAL-IP>
```

## Upgrade All Services (Demo: GitOps in action)

Edit `imageTag` in any `values.yaml` from `v0.10.5` to a newer version, commit, push.
ArgoCD detects the change and rolls out the update automatically.

```bash
# Example: upgrade catalog services
sed -i 's/v0.10.5/v0.10.6/' charts/catalogservice/values.yaml
git commit -am "upgrade catalog services to v0.10.6"
git push
# ArgoCD syncs within ~3 minutes (or force-sync in UI)
```

## Add a New Child App (Demo: App of Apps power)

1. Create `charts/newservice/` with Chart.yaml, values.yaml, templates/
2. Create `apps/newservice-app.yaml` pointing to that chart path
3. Push to Git — ArgoCD auto-discovers and deploys. No `kubectl apply` needed.

## Key Teaching Points

| Concept | Where to show |
| ------- | ------------- |
| App of Apps root | `root-app.yaml` — one file triggers everything |
| Child app grouping | `apps/*.yaml` — logical grouping (cart+redis together) |
| Helm values override | `charts/*/values.yaml` — change `imageTag` to show GitOps |
| Service DNS discovery | `checkoutservice/values.yaml` — `services.*` env vars |
| Port mapping surprise | `emailservice` — service port 5000 → container port 8080 |
| Security context | All deployments — `runAsNonRoot`, `readOnlyRootFilesystem` |
| emptyDir volume | `cartservice/templates/redis-deployment.yaml` — redis needs writable /data |
| LoadBalancer vs ClusterIP | `frontend/templates/service.yaml` — two services, one pod |
