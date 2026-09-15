# Dracolich on DigitalOcean — Deployment Runbook

Self-contained deployment guide for running the dracolich stack on **DigitalOcean Kubernetes (DOKS)**. Follow top-to-bottom on a fresh cluster.

For the Azure AKS runbook (kept for historical reference), see `DEPLOYMENT.md`.

---

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

---

## Prerequisites (one-time)

### Local tooling

```bash
brew install doctl kubernetes-cli helm
brew install kubectx mongosh   # optional but useful
```

### Accounts

| Account | Setup |
|---|---|
| DigitalOcean | cloud.digitalocean.com → API → Personal access tokens → Generate |
| Cloudflare | `dracolich.app` DNS managed there |
| GitHub PAT | Fine-grained, scoped to `dracolich-helm` repo, Contents: write |
| Docker Hub | For image pulls (no push needed locally) |
| Anthropic | API key for ai-api |

### Authenticate `doctl`

```bash
doctl auth init                # paste API token when prompted
doctl account get              # confirms you're logged in
```

### Env vars per session

```bash
export CLUSTER=dracolich-dev
export REGION=sfo3             # or nyc1 / tor1 / ams3 — pick nearest
export NS=dracolich-dev
```

---

## Step 1 — Provision the cluster

```bash
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

**Time:** ~4-5 min.

---

## Step 2 — Bootstrap cluster controllers

Install the three controllers via Helm:

```bash
# NGINX Ingress (provisions DO Regional LB with public IP)
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.externalTrafficPolicy=Local

# cert-manager (Let's Encrypt TLS)
helm upgrade --install cert-manager cert-manager \
  --repo https://charts.jetstack.io \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true

# ArgoCD (GitOps)
helm upgrade --install argocd argo-cd \
  --repo https://argoproj.github.io/argo-helm \
  --namespace argocd --create-namespace
```

Wait ~2 min for the LB to provision, then grab its IP:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

**Time:** ~5 min.

---

## Step 3 — Point DNS at the LB

In Cloudflare → `dracolich.app` zone → DNS:

| Type | Name | Value | Proxy |
|---|---|---|---|
| A | `dev` | `<LB IP from step 2>` | **DNS only** (gray cloud) |

The proxy must be off until Let's Encrypt issues the cert — HTTP-01 challenge won't survive proxied traffic.

Verify:

```bash
dig dev.dracolich.app +short   # returns the new LB IP
```

**Time:** DNS propagation ~1-2 min on Cloudflare.

---

## Step 4 — ClusterIssuer + namespace

```bash
kubectl create namespace dracolich-dev

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: laasilva@proton.me
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
EOF
```

---

## Step 5 — MongoDB

```bash
cd /Users/laasilva/Development/Personal/dracolich/dracolich-helm

# Generate root password — save it, you need it in step 7
PASS="$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)"
echo "MONGO PASSWORD: $PASS"

kubectl create secret generic mongodb-credentials \
  --namespace=dracolich-dev \
  --from-literal=MONGO_INITDB_ROOT_USERNAME=root \
  --from-literal=MONGO_INITDB_ROOT_PASSWORD="$PASS"

kubectl apply -f infrastructure/mongodb.yaml

# Wait for it to come up
kubectl get pods -n dracolich-dev -w        # ctrl-c when mongodb pod is Running 1/1
```

The manifest provisions a `mongo:7` Deployment with `strategy: Recreate`, a 5 Gi PVC on `do-block-storage`, and a `ClusterIP` Service on 27017. Auth env vars come from the `mongodb-credentials` Secret.

Connection URI (inside the cluster): `mongodb://root:<password>@mongodb.dracolich-dev.svc.cluster.local:27017/?authSource=admin`

**Time:** ~1 min.

---

## Step 6 — JWT keys

If you have PEMs backed up locally, upload those. Otherwise regenerate — invalidates any existing tokens, fine for dev:

```bash
mkdir -p /tmp/dracolich-keys && cd /tmp/dracolich-keys
openssl ecparam -genkey -name secp384r1 -noout -out ec-private.pem
openssl ec -in ec-private.pem -pubout -out ec-public.pem

# user-api needs both keys (signs + verifies)
kubectl create secret generic user-api-jwt-keys \
  --namespace=dracolich-dev \
  --from-file=ec-private.pem \
  --from-file=ec-public.pem

# deck-builder-api needs public only (verifies)
kubectl create secret generic deck-builder-api-jwt-keys \
  --namespace=dracolich-dev \
  --from-file=ec-public.pem
```

The Helm charts mount these secrets as volumes at `/etc/dracolich/keys/`, so the app reads them as files (not env-var strings).

---

## Step 7 — Per-service env secrets

Build the Mongo URI once:

```bash
URI="mongodb://root:$PASS@mongodb.dracolich-dev.svc.cluster.local:27017/?authSource=admin"
```

### mtg-library-api

```bash
kubectl create secret generic mtg-library-secrets \
  --namespace=dracolich-dev \
  --from-literal=MONGODB_URI="$URI" \
  --from-literal=MONGODB_DATABASE=mtg-library \
  --from-literal=CORS_ALLOWED_ORIGINS=https://dev.dracolich.app
```

### user-api

```bash
kubectl create secret generic user-api-secrets \
  --namespace=dracolich-dev \
  --from-literal=MONGODB_URI="$URI" \
  --from-literal=MONGODB_DATABASE=dracolich-user \
  --from-literal=CORS_ALLOWED_ORIGINS=https://dev.dracolich.app \
  --from-literal=DRACOLICH_JWT_PRIVATE_KEY=/etc/dracolich/keys/ec-private.pem \
  --from-literal=DRACOLICH_JWT_PUBLIC_KEY=/etc/dracolich/keys/ec-public.pem
```

### ai-api

Grab the Anthropic key from the local `.env` (it's not committed to git):

```bash
ANTHROPIC_KEY="$(grep ANTHROPIC_API_KEY \
  /Users/laasilva/Development/Personal/dracolich/dracolich-ai-api/.env | cut -d= -f2)"

kubectl create secret generic ai-api-secrets \
  --namespace=dracolich-dev \
  --from-literal=MONGODB_URI="$URI" \
  --from-literal=MONGODB_DATABASE=dracolich-ai \
  --from-literal=ANTHROPIC_API_KEY="$ANTHROPIC_KEY" \
  --from-literal=MTG_LIBRARY_API_URL=http://mtg-library-api.dracolich-dev.svc.cluster.local:8080
```

### deck-builder-api

```bash
ANON_SECRET="$(openssl rand -base64 32)"

kubectl create secret generic deck-builder-api-secrets \
  --namespace=dracolich-dev \
  --from-literal=MONGODB_URI="$URI" \
  --from-literal=MONGODB_DATABASE=dracolich-deck-builder \
  --from-literal=CORS_ALLOWED_ORIGINS=https://dev.dracolich.app \
  --from-literal=DRACOLICH_JWT_PUBLIC_KEY=/etc/dracolich/keys/ec-public.pem \
  --from-literal=DRACOLICH_ANON_COOKIE_SECRET="$ANON_SECRET" \
  --from-literal=MTG_LIBRARY_API_URL=http://mtg-library-api.dracolich-dev.svc.cluster.local:8080 \
  --from-literal=AI_API_URL=http://ai-api.dracolich-dev.svc.cluster.local:8080
```

> **Env var names above are conservative best-guesses.** If a pod crashes on startup with `Could not resolve placeholder 'X'`, the correct var name is `X` — update that service's secret with the missing key. Spring's relaxed binding: property `foo.bar-baz` maps to env var `FOO_BAR_BAZ`. Idempotent update:
>
> ```bash
> kubectl create secret generic <name> \
>   --namespace=dracolich-dev \
>   --from-literal=KEY=value \
>   --dry-run=client -o yaml | kubectl apply -f -
>
> kubectl rollout restart deployment/<svc> -n dracolich-dev
> ```

---

## Step 8 — Wire up ArgoCD

```bash
cd /Users/laasilva/Development/Personal/dracolich/dracolich-helm

kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/applicationset.yaml
```

ArgoCD picks up the `ApplicationSet`, spawns 4 Applications (mtg-library-api, user-api, ai-api, deck-builder-api), and starts syncing them from `main`.

Watch progress:

```bash
kubectl get applications -n argocd -w      # ctrl-c when all show Synced/Healthy
```

### Access the ArgoCD UI (optional)

```bash
# Initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d ; echo

# Port-forward
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

Visit `https://localhost:8080`, log in as `admin`. Change the password via UI, then delete the bootstrap secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

**Time:** ~3 min for first sync (image pulls + pod starts).

---

## Step 9 — Wait for TLS cert

```bash
kubectl get certificate -n dracolich-dev -w
# ctrl-c when READY=True
```

Let's Encrypt HTTP-01 challenge runs automatically. If DNS hasn't propagated yet or Cloudflare is proxied, it'll retry — check with `kubectl describe certificate ... -n dracolich-dev`.

**Time:** ~2 min once DNS resolves correctly.

---

## Step 10 — Verify everything's live

```bash
# All pods Running 1/1?
kubectl get pods -n dracolich-dev

# Actuator health via the public URL
curl https://dev.dracolich.app/dracolich/mtg-library/api/v0/actuator/health
curl https://dev.dracolich.app/dracolich/user/api/v0/actuator/health
curl https://dev.dracolich.app/dracolich/mtg-deck-builder/api/v0/actuator/health

# ai-api has no ingress (cluster-internal only) — verify from within the cluster:
kubectl run curl-test -n dracolich-dev --rm -it --image=curlimages/curl -- \
  curl http://ai-api.dracolich-dev.svc.cluster.local:8080/dracolich/ai/api/v0/actuator/health
```

First `mtg-library-api` startup runs the Scryfall bulk sync (~10-15 min for 90k cards). Watch:

```bash
kubectl logs -n dracolich-dev -l app.kubernetes.io/name=mtg-library-api -f
```

Once the sync completes, restarts are fast — Mongo has the data.

---

## Operational Runbook

### Pause the cluster (save money overnight)

DOKS doesn't have `az aks stop`. To park:

```bash
POOL=$(doctl kubernetes cluster node-pool list $CLUSTER --format ID --no-header)

# Scale to zero (~1 min)
doctl kubernetes cluster node-pool update $CLUSTER $POOL --count 0

# Scale back up (~2-3 min to Ready)
doctl kubernetes cluster node-pool update $CLUSTER $POOL --count 1
```

**Cost while parked:** ~$13/mo (LB + PVC persist). Not free like Azure's ~$5, but close.

### Delete the cluster entirely

```bash
doctl kubernetes cluster delete $CLUSTER

# Detached LB + volume survive — delete separately for $0 cost:
doctl compute load-balancer list
doctl compute volume list
doctl compute load-balancer delete <id>
doctl compute volume delete <id>
```

### Get kubeconfig on a new machine

```bash
doctl kubernetes cluster kubeconfig save $CLUSTER
```

### Force ArgoCD sync

```bash
kubectl -n argocd patch app <app-name> --type merge -p '{"operation":{"sync":{}}}'

# Or all at once
for app in mtg-library-api user-api ai-api deck-builder-api; do
  kubectl -n argocd patch app $app --type merge -p '{"operation":{"sync":{}}}'
done
```

### Restart a pod (after secret update, etc.)

```bash
kubectl rollout restart deployment/<svc-name> -n dracolich-dev
```

### Update a Secret idempotently

```bash
kubectl create secret generic <name> \
  --namespace=dracolich-dev \
  --from-literal=KEY=value \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart deployment/<svc> -n dracolich-dev
```

### Recover the MongoDB password

```bash
kubectl get secret mongodb-credentials -n dracolich-dev \
  -o jsonpath='{.data.MONGO_INITDB_ROOT_PASSWORD}' | base64 -d ; echo
```

### Port-forward to MongoDB

```bash
kubectl port-forward -n dracolich-dev svc/mongodb 27017:27017
# Connect from mongosh: mongodb://root:<password>@localhost:27017/?authSource=admin
```

### Tail service logs

```bash
kubectl logs -n dracolich-dev -l app.kubernetes.io/name=<svc> -f --tail=100
```

### Backup MongoDB (manual — no auto-backups on self-hosted)

```bash
kubectl exec -n dracolich-dev deploy/mongodb -- \
  mongodump --archive --gzip \
    --uri="mongodb://root:$PASS@localhost:27017/?authSource=admin" \
  > "dracolich-$(date +%Y%m%d).archive.gz"
```

Restore:

```bash
kubectl exec -i -n dracolich-dev deploy/mongodb -- \
  mongorestore --archive --gzip --drop \
    --uri="mongodb://root:$PASS@localhost:27017/?authSource=admin" \
  < dracolich-YYYYMMDD.archive.gz
```

---

## Troubleshooting

### DO-specific

#### 1. Pod `Pending`, event: `Insufficient memory`

**Cause**: 4 GB node is tight when everything's up + Scryfall sync is running.

**Fix (short-term)**: temporarily scale to 2 nodes:

```bash
doctl kubernetes cluster node-pool update $CLUSTER $POOL --count 2
# ... wait for sync ...
doctl kubernetes cluster node-pool update $CLUSTER $POOL --count 1
```

**Fix (long-term)**: drop ArgoCD (~700 MB) or trim service memory limits.

#### 2. PVC `Pending`, event: `no storage class`

**Cause**: manifest still uses an Azure-era storage class value.

**Fix**: `infrastructure/mongodb.yaml` should say `storageClassName: do-block-storage`. Verify:

```bash
kubectl get storageclass
# do-block-storage should be marked (default)
```

#### 3. LB IP shows as `<pending>` for > 5 min

**Cause**: DO region temporarily out of LB capacity, or account quota.

**Fix**: check quota at cloud.digitalocean.com/account/limits, or try a different region.

### Provider-agnostic (still applies)

#### 4. Spring app fails to start: `Could not resolve placeholder 'X'`

**Cause**: An env var referenced in `application.yml` isn't in the pod's env, and has no default (CI doesn't include gitignored `application-dev.yml`).

**Fix**: add `X` to the service's Secret (see idempotent update pattern above), then rollout restart. Long-term fix in code: add empty default: `${X:}`.

#### 5. Spring app connects to `localhost:27017` instead of MongoDB

**Cause**: Spring Boot 4 + Spring Data MongoDB 5 expects `spring.mongodb.*` (NOT `spring.data.mongodb.*`). The latter is silently ignored.

**Fix in code**:

```yaml
spring:
  mongodb:               # NOT spring.data.mongodb
    uri: ${MONGODB_URI}
    database: ${MONGODB_DATABASE}
```

#### 6. Env var set in pod but Spring property unresolved

**Cause**: Spring relaxed binding — env var name must match property name with `.` and `-` replaced by `_`, uppercased. Property `dracolich.jwt.private-key` maps to env `DRACOLICH_JWT_PRIVATE_KEY`, NOT `JWT_PRIVATE_KEY_PATH`.

**Fix**: rename the env var, or reference it explicitly in `application.yml`: `${MY_RAW_VAR:default}`.

#### 7. Pod `CreateContainerConfigError`, "secret not found"

**Cause**: Secret created in the wrong namespace (usually `default` if `$NS` wasn't exported).

**Fix**:

```bash
kubectl get secrets --all-namespaces | grep <secret-name>
# Recreate with explicit --namespace=dracolich-dev
```

#### 8. `mtg-library-api` OOMKilled during Scryfall sync

**Cause**: Memory limit too low — the seeder buffers the ~400 MB Scryfall bulk file.

**Fix**: values already set `limits.memory: 1536Mi` and `JAVA_TOOL_OPTIONS: -XX:MaxRAMPercentage=70`. If still OOMing, scale to a bigger node temporarily.

#### 9. Browser: `ERR_CERT_AUTHORITY_INVALID` on `.app` domain

**Cause**: `.app` is HSTS-preloaded — browsers refuse any non-LE-issued cert. Usually means the cert hasn't been issued yet.

**Fix**: wait for `kubectl get certificate -n dracolich-dev` to show `READY=True`. Cloudflare must be gray-cloud until then.

---

## Cost breakdown

| Component | Running | Parked (nodes=0) |
|---|---|---|
| DOKS control plane | $0 | $0 |
| 1× `s-2vcpu-4gb` node | $24 | $0 |
| Regional Load Balancer | $12 | $12 |
| 5 GB block storage | $0.50 | $0.50 |
| Cloudflare DNS | $0 | $0 |
| Domain (`dracolich.app`) | ~$1/mo annualized | ~$1/mo annualized |
| **Total** | **~$37/mo** | **~$13/mo** |

Scale node pool to zero overnight → ~$13/mo. Delete the LB + PVC → $0/mo (but you rebuild them next time).

---

## What transferred from AKS

**Untouched, works as-is:**
- `lib/` library chart (Deployment, Service, Ingress templates)
- All `services/<svc>/values.yaml` + `values-dev.yaml`
- `argocd/project.yaml` + `argocd/applicationset.yaml`
- Cloudflare DNS records (just repoint the A record)
- cert-manager + Let's Encrypt config
- NGINX Ingress config
- GitHub Actions CI (Docker Hub push, semver bumping, cross-repo helm-bump)
- App code, secrets structure

**Had to change:**
- `infrastructure/mongodb.yaml` → `storageClassName: do-block-storage`, `storage: 5Gi`
- This doc replaces the Azure runbook for day-to-day work

**Lost in migration:**
- MongoDB data (Scryfall re-sync + lost dev decks/users)
- Old TLS cert (LE re-issues automatically)
- Any manual `kubectl edit` drift that wasn't committed to `dracolich-helm`

---

## Future hardening (deferred)

Same list as the Azure runbook — none of these are DO-specific:

- Cloudflare Access in front of `dev.dracolich.app` and `argocd.dracolich.app`
- External Secrets Operator + a secret store (DO doesn't have Key Vault; options are Doppler, 1Password Connect, or self-hosted Vault)
- NetworkPolicy to restrict ai-api ingress to deck-builder-api only
- Multi-replica services + PodDisruptionBudget
- HPA for traffic-driven scaling
- Automated MongoDB backups (cron job pushing to Spaces / S3)
- MongoDB replica set for HA
- Prometheus + Grafana + Loki
- Rate limiting at NGINX or app layer
- `application-prod.yml` profiles per service
- Production cluster — separate DOKS cluster, separate DNS zone (`dracolich.app` → prod, `dev.dracolich.app` → dev)
