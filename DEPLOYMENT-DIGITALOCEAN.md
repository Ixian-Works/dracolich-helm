# Dracolich on DigitalOcean — Deployment Guide

Delta doc for running the dracolich stack on **DigitalOcean Kubernetes (DOKS)** instead of Azure AKS. Most of the runbook is identical because the Helm charts + ArgoCD config are provider-agnostic — this doc only covers what differs. For anything not mentioned here, see `DEPLOYMENT.md`.

## Target shape

Cheapest workable dev setup, ~**$37/mo** running 24/7:

| Component | SKU | $/mo |
|---|---|---|
| DOKS control plane | Free | 0 |
| Node pool | 1× `s-2vcpu-4gb` | 24 |
| Load Balancer | Regional | 12 |
| Block storage (MongoDB PVC) | 5 GB | 0.50 |
| Cloudflare DNS | Free | 0 |
| **Total** | | **~$37** |

**Trade-offs accepted vs the 2-node Azure plan:**
- Single node — no failover. Node reboot = brief outage. Fine for dev.
- ArgoCD + all four services + Mongo share one 4 GB node. Steady-state fits; Scryfall sync spikes to ~4 GB peak.
- No auto-stop equivalent to `az aks stop`. To pause: scale the node pool to zero (see "Pause the cluster" below).

## Prerequisites

```bash
brew install doctl kubernetes-cli helm
brew install kubectx mongosh   # optional

doctl auth init                # paste API token from cloud.digitalocean.com/account/api
```

Cloudflare account with `dracolich.app` DNS + a fine-grained GitHub PAT scoped to `dracolich-helm` are still needed (same as Azure).

## Env vars per session

```bash
export CLUSTER=dracolich-dev
export REGION=sfo3        # or nyc1 / tor1 / ams3 — pick nearest
export NS=dracolich-dev
```

## Stage 1 — DO provisioning

```bash
# ~4-5 min to spin up
doctl kubernetes cluster create $CLUSTER \
  --region $REGION \
  --version latest \
  --node-pool "name=default;size=s-2vcpu-4gb;count=1;auto-scale=false" \
  --wait

# kubeconfig context is added automatically
kubectl config current-context             # ends with -dracolich-dev
kubectl get nodes                           # expect 1 Ready node
```

`s-2vcpu-4gb`: 2 vCPU, 4 GB RAM, 80 GB SSD, ~$24/mo. Kubelet + system reserve ~500 MB, leaving ~3.5 GB usable.

## Stage 2 — Cluster bootstrap

Identical to `DEPLOYMENT.md` Stage 2. Run the same `helm upgrade --install` blocks for `ingress-nginx`, `cert-manager`, and `argo-cd`. Then apply the `ClusterIssuer` YAML and create the `dracolich-dev` namespace.

DO-specific note: the NGINX Ingress `Service` provisions a DO Regional Load Balancer (~2-3 min). Fetch its IP the usual way:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Update the Cloudflare `A` record for `dev` to that IP. Keep it **DNS-only** (gray cloud) until Cloudflare Access is set up — HTTP-01 challenge won't survive orange-cloud proxying.

## Stage 3 — MongoDB

Same as `DEPLOYMENT.md` Stage 3. Note that `infrastructure/mongodb.yaml` already declares `storageClassName: do-block-storage` (DO's default CSI class, auto-installed on every DOKS cluster) and `storage: 5Gi` (right-sized for the 90k-card Scryfall corpus).

```bash
PASS="$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)"
echo "Save this: $PASS"

kubectl create secret generic mongodb-credentials \
  --namespace=dracolich-dev \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=root \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD="$PASS"

kubectl apply -f infrastructure/mongodb.yaml
```

## Stages 4-5 — ArgoCD + services

Identical to `DEPLOYMENT.md`. `kubectl apply -f argocd/project.yaml` + `kubectl apply -f argocd/applicationset.yaml`, then recreate the per-service secrets (`mtg-library-secrets`, `user-api-jwt-keys`, `deck-builder-api-jwt-keys`, `ai-api` env, etc.) with the same commands.

## Operational Runbook (DO-specific)

### Pause the cluster (save money overnight)

DOKS doesn't have `az aks stop`, but you can scale the node pool to zero. Pods disappear, PVCs + LB + ArgoCD state persist:

```bash
# Get pool ID
POOL=$(doctl kubernetes cluster node-pool list $CLUSTER --format ID --no-header)

# Scale to zero (~1 min)
doctl kubernetes cluster node-pool update $CLUSTER $POOL --count 0

# Scale back up (~2-3 min to Ready)
doctl kubernetes cluster node-pool update $CLUSTER $POOL --count 1
```

While scaled to zero: ~$13/mo (LB + PVC persist). Not free like Azure's parked state (~$5/mo), but close.

### Delete the cluster entirely

```bash
doctl kubernetes cluster delete $CLUSTER
# Detached LB + volume survive — delete them separately if you want $0 cost:
doctl compute load-balancer list
doctl compute volume list
doctl compute load-balancer delete <id>
doctl compute volume delete <id>
```

### Get kubeconfig on a new machine

```bash
doctl kubernetes cluster kubeconfig save $CLUSTER
```

## Troubleshooting — DO edition

### 1. Pod stuck `Pending`, event: `Insufficient memory`

**Cause**: 4 GB node is tight when everything's up + Scryfall sync is running.

**Fix (short-term)**: temporarily scale to 2 nodes during sync, then back down:

```bash
doctl kubernetes cluster node-pool update $CLUSTER $POOL --count 2
# ... wait for sync ...
doctl kubernetes cluster node-pool update $CLUSTER $POOL --count 1
```

**Fix (long-term)**: drop ArgoCD (~700 MB) or trim service memory limits.

### 2. PVC `Pending`, event: `no storage class`

**Cause**: manifest still says `storageClassName: default` (old Azure value).

**Fix**: `infrastructure/mongodb.yaml` should use `do-block-storage`. Verify:

```bash
kubectl get storageclass
# do-block-storage should be marked (default)
```

### 3. LB IP shows as `<pending>` for > 5 min

**Cause**: DO region temporarily out of LB capacity (rare) or account quota.

**Fix**: check quota at cloud.digitalocean.com/account/limits, or try a different region.

### 4. Everything from Azure `DEPLOYMENT.md` troubleshooting items 5-16

All the Spring Boot / Jackson / cert / GitHub Actions gotchas are provider-agnostic and still apply. Cross-reference by number.

## What DID and DIDN'T transfer from AKS

**Untouched, works as-is:**
- `lib/` library chart (Deployment, Service, Ingress templates)
- All `services/<svc>/values.yaml` + `values-dev.yaml`
- `argocd/project.yaml` + `argocd/applicationset.yaml`
- Cloudflare DNS records (just repoint the A record)
- cert-manager + Let's Encrypt config (issues fresh certs on first pod start)
- NGINX Ingress config
- GitHub Actions CI (Docker Hub push, semver bumping, cross-repo helm-bump) — no changes
- All app code, all secrets structure

**Had to change:**
- `infrastructure/mongodb.yaml` → `storageClassName: do-block-storage`, `storage: 5Gi`
- This doc

**Lost in migration:**
- MongoDB data (needs `mongodump` from AKS before shutdown, or accept Scryfall re-sync + losing dev decks/users)
- TLS cert (LE re-issues automatically — no action)
- Any manual `kubectl edit` drift that wasn't committed to `dracolich-helm`
