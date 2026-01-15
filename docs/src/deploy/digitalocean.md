# Deploy Rocketship Cloud (prod) on DigitalOcean Kubernetes

This is the current production runbook for Rocketship Cloud on DigitalOcean Kubernetes (DOKS), using Cloudflare for DNS + the web console.

All commands below assume you are running from the `rocketship-internal` repository root unless stated otherwise.

## Architecture / hostnames

- `app.rocketship.sh`: Cloudflare Pages (static UI) + Pages Functions proxy.
- `api.rocketship.sh`: Kubernetes ingress → `rocketship-controlplane` (HTTPS, DNS-only).
- `cli.rocketship.sh`: Kubernetes ingress → `rocketship-engine` (gRPC over TLS, DNS-only).

Why:
- Cloudflare’s standard HTTP proxy does **not** support gRPC for `cli.*` (keep DNS-only).
- The web console needs a stable browser origin (`app.*`) while still reaching controlplane endpoints; we proxy select paths via Pages Functions.

## Prerequisites

- A Cloudflare account (we delegate `rocketship.sh` DNS to Cloudflare).
- A DigitalOcean account + `doctl`.
- `kubectl`, `helm`.
- A GitHub OAuth App for the web/CLI SSO.
- Postmark (or equivalent) for transactional email (verification).
- Checkout layout (recommended): `rocketship` and `rocketship-internal` as sibling directories.

## 0) Cloudflare + Namecheap (DNS authority)

1. In Cloudflare, add the `rocketship.sh` domain and note the assigned nameservers (example: `amber.ns.cloudflare.com`, `thomas.ns.cloudflare.com`).
2. In Namecheap, set the domain’s nameservers to the Cloudflare nameservers.
3. In Cloudflare DNS, create (or update):
   - `api` **A** → the DOKS ingress LoadBalancer IP (DNS only / grey cloud)
   - `cli` **A** → the DOKS ingress LoadBalancer IP (DNS only / grey cloud)
4. Create the Cloudflare Pages project (step 8) and add `app.rocketship.sh` as a Pages custom domain.

Gotchas:
- Do **not** proxy `cli.rocketship.sh` (gRPC). Keep it DNS-only.
- Avoid proxying `api.rocketship.sh` while using cert-manager HTTP-01 unless you’re sure you want Cloudflare in the ACME path.

## 1) Create the DOKS cluster

Create a small cluster first (cost-sensitive):
- 2 nodes, 2vCPU/4GB each
- Enable automatic patch upgrades (recommended)

Then fetch kubeconfig:

```bash
doctl kubernetes cluster kubeconfig save <cluster-name>
kubectl config use-context do-<region>-<cluster-name>
```

## 2) Install ingress-nginx

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.14.1 \
  --namespace ingress-nginx --create-namespace
```

Get the external IP (you’ll use it for Cloudflare `api`/`cli` A records):

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

## 3) Create the `rocketship` namespace

```bash
kubectl create namespace rocketship
kubectl config set-context --current --namespace=rocketship
```

## 4) Install cert-manager + Let’s Encrypt issuer

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --version v1.19.2 \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

Create a prod ClusterIssuer (HTTP-01 via ingress-nginx):

```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: "<your-acme-email>"
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
YAML
```

## 5) Install Temporal (small + cheap defaults)

We currently run the Temporal Helm chart with:
- 1 replica of each service
- Cassandra (1 node)
- Elasticsearch (1 node) for visibility
- Prometheus/Grafana/Temporal Web disabled

```bash
helm repo add temporal https://go.temporal.io/helm-charts
helm repo update

helm upgrade --install temporal temporal/temporal \
  --version 0.73.1 \
  --namespace rocketship \
  -f deploy/prod/temporal-values.yaml \
  --wait --timeout 20m
```

Register the Temporal namespace Rocketship uses:

```bash
kubectl exec -n rocketship deploy/temporal-admintools -- \
  temporal operator namespace create --namespace default
```

Notes:
- Temporal does **not** strictly require Elasticsearch, but disabling it correctly requires changing the visibility config; we keep a small ES node for now.

## 6) Create Rocketship secrets (prod)

Create a folder for local secret material (never commit it):

```bash
mkdir -p .secrets-prod
chmod 700 .secrets-prod
```

Postgres credentials (for the bundled Bitnami Postgres subchart):

```bash
POSTGRES_PASSWORD="$(openssl rand -base64 24)"

kubectl create secret generic rocketship-postgres-auth -n rocketship \
  --from-literal=password="$POSTGRES_PASSWORD" \
  --from-literal=postgres-password="$POSTGRES_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -
```

If you are upgrading an existing install and want to keep the current Postgres password, reuse it instead of generating a new one:

```bash
POSTGRES_PASSWORD="$(
  kubectl -n rocketship get secret rocketship-postgresql -o jsonpath='{.data.password}' | base64 -d
)"
```

Controlplane DB URL (used by both controlplane and engine):

```bash
kubectl create secret generic rocketship-controlplane-db -n rocketship \
  --from-literal=DATABASE_URL="postgresql://rocketship:${POSTGRES_PASSWORD}@rocketship-postgresql:5432/rocketship?sslmode=disable" \
  --dry-run=client -o yaml | kubectl apply -f -
```

JWT signing key (RSA):

```bash
openssl genrsa -out .secrets-prod/signing-key.pem 2048
kubectl create secret generic rocketship-controlplane-signing -n rocketship \
  --from-file=signing-key.pem=.secrets-prod/signing-key.pem \
  --dry-run=client -o yaml | kubectl apply -f -
```

Refresh-token HMAC key:

```bash
kubectl create secret generic rocketship-controlplane-secrets -n rocketship \
  --from-literal=ROCKETSHIP_CONTROLPLANE_REFRESH_KEY="$(openssl rand -base64 32)" \
  --dry-run=client -o yaml | kubectl apply -f -
```

GitHub OAuth client secret (the client ID is referenced in Helm values; the secret stays in Kubernetes):

```bash
kubectl create secret generic rocketship-github-oauth -n rocketship \
  --from-literal=ROCKETSHIP_GITHUB_CLIENT_SECRET="<github-oauth-client-secret>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

GitHub App (repo access + webhooks):

- Create a GitHub App (in the `rocketship-ai` org) with:
  - Setup URL: `https://app.rocketship.sh/github-app/callback`
  - Webhook URL: `https://api.rocketship.sh/github-app/webhook`
  - Generate a webhook secret
  - Generate a private key (download the `.pem`)

Notes:
- The setup URL must be on `app.rocketship.sh` so the controlplane can read the browser auth cookie during the install redirect.
- Current prod GitHub App: `rocketship-cloud` (App ID `2653029`), https://github.com/apps/rocketship-cloud

Then apply the secrets:

```bash
kubectl create secret generic rocketship-github-app -n rocketship \
  --from-literal=ROCKETSHIP_GITHUB_APP_ID="<github-app-id>" \
  --from-literal=ROCKETSHIP_GITHUB_APP_SLUG="<github-app-slug>" \
  --from-literal=ROCKETSHIP_GITHUB_APP_PRIVATE_KEY_PEM="$(cat /path/to/private-key.pem)" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic rocketship-github-webhook -n rocketship \
  --from-literal=ROCKETSHIP_GITHUB_WEBHOOK_SECRET="<github-webhook-secret>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Postmark:

```bash
kubectl create secret generic rocketship-postmark-secret -n rocketship \
  --from-literal=ROCKETSHIP_EMAIL_FROM="noreply@rocketship.sh" \
  --from-literal=ROCKETSHIP_POSTMARK_SERVER_TOKEN="<postmark-server-token>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

Worker token (engine → worker auth):

```bash
kubectl create secret generic rocketship-worker-token -n rocketship \
  --from-literal=token="$(openssl rand -base64 32)" \
  --dry-run=client -o yaml | kubectl apply -f -
```

## 7) Deploy Rocketship (engine + worker + controlplane)

The source-of-truth Helm chart is in the public repo: `rocketship/charts/rocketship`.

Deploy using the prod hostname model:

```bash
helm upgrade --install rocketship ../rocketship/charts/rocketship \
  --namespace rocketship \
  -f deploy/prod/rocketship-values.yaml \
  --wait
```

## 8) Cloudflare Pages (web console)

Create a Cloudflare Pages project connected to `rocketship-ai/rocketship`:
- Root directory: `web`
- Build command: `npm ci && npm run build`
- Output directory: `dist`
- Env var: `API_ORIGIN=https://api.rocketship.sh`

Then add custom domain `app.rocketship.sh`.

GitHub OAuth App settings (critical):
- Homepage URL: `https://rocketship.sh` (or `https://app.rocketship.sh`; callback URL matters most)
- Authorization callback URL: `https://app.rocketship.sh/callback`
- Enable Device Flow.

If you create a new OAuth App, update `deploy/prod/rocketship-values.yaml` (`controlplane.github.clientID`) and update the `rocketship-github-oauth` secret with the new client secret.

## 9) Verify prod is healthy

Kubernetes:

```bash
kubectl get pods -n rocketship
helm -n rocketship list
```

HTTP endpoints:

```bash
curl -fsS https://api.rocketship.sh/healthz
curl -fsS https://api.rocketship.sh/.well-known/jwks.json | head
```

gRPC endpoint (should fail on HTTP GET but prove TLS/route is live):

```bash
curl -sS https://cli.rocketship.sh/ | head
```

Web:
- Load `https://app.rocketship.sh/login`
- Complete GitHub OAuth and onboarding once.

CLI:

```bash
rocketship profile create prod grpcs://cli.rocketship.sh
rocketship profile use prod
rocketship login
rocketship status
rocketship run -af ../rocketship/.rocketship/simple-http.yaml
```

## 10) Updating prod

1. Pick the image tag(s) to deploy (release tag or git SHA).
2. Update `deploy/prod/rocketship-values.yaml`:
   - `controlplane.image.tag`
   - `engine.image.tag`
   - `worker.image.tag`
3. Deploy:

```bash
helm upgrade rocketship ../rocketship/charts/rocketship \
  --namespace rocketship \
  -f deploy/prod/rocketship-values.yaml \
  --wait
```

4. Watch rollouts:

```bash
kubectl -n rocketship rollout status deploy/rocketship-controlplane
kubectl -n rocketship rollout status deploy/rocketship-engine
kubectl -n rocketship rollout status deploy/rocketship-worker
```

## 11) Legacy cleanups (optional)

- If `auth.rocketship.sh` is still in Cloudflare DNS pointing at an old IP, delete it (we do not use this hostname in the current prod model).
