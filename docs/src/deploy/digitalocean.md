# Deploy Rocketship on DigitalOcean Kubernetes

This walkthrough recreates the production proof-of-concept we validated on DigitalOcean Kubernetes (DOKS). It covers standing up Temporal, terminating TLS through an NGINX ingress, wiring the CLI via
profiles, and provisioning the optional bundled Postgres dependency. The chart pulls Rocketship images directly from Docker Hub (`rocketshipai/...`), so you no longer need to maintain a separate registry
unless desired.

The steps assume you control public DNS for `cli.rocketship.sh`, `app.rocketship.sh`, and `auth.rocketship.sh` (or equivalent) and can issue a SAN certificate that covers all three hosts.

## Prerequisites

- DigitalOcean account with:
  - A Kubernetes cluster (2 × CPU-optimised nodes were used during validation)
- [`doctl`](https://docs.digitalocean.com/reference/doctl/how-to/install/) authenticated (`doctl auth init`)
- `kubectl` configured for the cluster (`doctl kubernetes cluster kubeconfig save <cluster-name>`)
- Docker CLI with [Buildx](https://docs.docker.com/build/install-buildx/) (if you plan to build custom images)
- Helm 3
- TLS assets
  - `certificate.crt` and `private.key` (ZeroSSL issues these; concatenate the intermediate bundle with the server cert if required)

All commands below run from the repository root.

## 1. Set Up Namespaces and Ingress Controller

```bash
kubectl create namespace rocketship
kubectl config set-context --current --namespace=rocketship

# Install ingress-nginx (DigitalOcean automatically provisions a Load Balancer)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.13.2 \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/do-loadbalancer-enable-proxy-protocol"="true"
```

The annotation enables PROXY protocol support on DigitalOcean’s load balancer, which keeps source IPs available in the ingress logs. Omit or adjust if you do not need it.

## 2. Install Temporal

```bash
helm repo add temporal https://go.temporal.io/helm-charts
helm repo update

helm install temporal temporal/temporal \
 --version 0.66.0 \
 --namespace rocketship \
 --set server.replicaCount=1 \
 --set cassandra.config.cluster_size=1 \
 --set elasticsearch.replicas=1 \
 --set prometheus.enabled=false \
 --set grafana.enabled=false \
 --wait --timeout 15m
```

Register the Temporal logical namespace the Rocketship worker will use:

```bash
kubectl exec -n rocketship deploy/temporal-admintools -- \
 temporal operator namespace create --namespace default
```

(Keep default unless you intend to manage multiple namespaces; update Helm values accordingly later.)

## 3. Create the TLS Secret

Issue a SAN certificate that covers cli.rocketship.sh, app.rocketship.sh, and auth.rocketship.sh (Let’s Encrypt or ZeroSSL work well). After you have the combined cert/key, update the secret:

```bash
# optional: remove the old secret if it exists
kubectl delete secret rocketship-cloud-tls -n rocketship 2>/dev/null || true
```

```bash
# create the secret with the new cert/key
kubectl create secret tls rocketship-cloud-tls \
 --namespace rocketship \
 --cert=/etc/letsencrypt/live/rocketship.sh/fullchain.pem \
 --key=/etc/letsencrypt/live/rocketship.sh/privkey.pem
```

## 4. Provision Postgres & Broker Secrets

Rocketship’s auth broker stores organisations, users, and refresh tokens in Postgres. You can either point at an existing database or enable the bundled Bitnami chart. The steps below assume you are using
the bundled chart.

### Postgres password (Bitnami expects this key name)

```bash
kubectl create secret generic rocketship-postgres-auth \
 --namespace rocketship \
 --from-literal=postgres-password='<strong-password>'
```

### DSN for the broker (targets the Bitnami service rocketship-postgresql:5432)

```bash
kubectl create secret generic rocketship-auth-broker-database \
 --namespace rocketship \
 --from-literal=DATABASE_URL='postgres://rocketship:<strong-password>@rocketship-postgresql:5432/rocketship?sslmode=disable'
```

### Refresh-token HMAC key (Base64-encoded)

```bash
kubectl create secret generic rocketship-auth-broker-secrets \
 --namespace rocketship \
 --from-literal=ROCKETSHIP_BROKER_REFRESH_KEY="$(openssl rand -base64 32)"
```

## 5. Signer & GitHub OAuth Secrets

### 1. Signing key for JWTs:

```bash
   openssl genrsa -out signing-key.pem 2048
   kubectl create secret generic rocketship-auth-broker-signing \
    --namespace rocketship \
    --from-file=signing-key.pem
```

### 2. GitHub OAuth application (enable Device Flow, set callback to https://cli.rocketship.sh/oauth2/callback). Store its secret:

```bash
   kubectl create secret generic rocketship-github-oauth \
    --namespace rocketship \
    --from-literal=ROCKETSHIP_GITHUB_CLIENT_SECRET='<github-client-secret>'
```

### 3. oauth2-proxy credentials (used by the web preset):

```bash
   COOKIE_SECRET=$(python - <<'PY'

import secrets, base64
print(base64.urlsafe_b64encode(secrets.token_bytes(32)).decode())
PY
)
kubectl create secret generic oauth2-proxy-credentials \
--namespace rocketship \
--from-literal=clientID='<github-client-id>' \
--from-literal=clientSecret='<github-client-secret>' \
--from-literal=cookieSecret="$COOKIE_SECRET"
```

## 6. Deploy the Rocketship Helm Chart

The chart pulls images from Docker Hub (`rocketshipai/...`) by default. Use the production + GitHub presets and enable the bundled Postgres:

```bash
helm upgrade --install rocketship charts/rocketship \
--namespace rocketship \
-f charts/rocketship/values-production.yaml \
-f charts/rocketship/values-github-cloud.yaml \
-f charts/rocketship/values-github-web.yaml \
--set auth.broker.github.clientID='<github-client-id>' \
--set postgres.enabled=true \
--set postgres.auth.existingSecret=rocketship-postgres-auth \
--set postgres.auth.existingSecretPasswordKey=postgres-password \
--wait
```

If you also run the self-hosted discovery presets, add the appropriate overlays (values-github-selfhost.yaml, values-oidc-web.yaml, etc.). Removing the postgres.enabled=true flag means no database will be
deployed; in that case, provide the DSN to an external Postgres instance in values-github-cloud.yaml.

Confirm the pods are healthy:

```bash
kubectl get pods -n rocketship
```

rocketship-engine, rocketship-worker, rocketship-auth-broker, and rocketship-web-oauth2-proxy should report READY 1/1. Temporal services may restart once while Cassandra and Elasticsearch initialise—that
is expected.

## 7. Configure the CLI & Web Login

1. CLI: ensure every developer has the rocketship binary from the latest release, then run:

```bash
   rocketship profile create cloud grpcs://cli.rocketship.sh
   rocketship profile use cloud
   rocketship login
   rocketship status
```

The CLI handles GitHub device flow, stores the refresh token securely, and auto-renews access tokens on each command.

2. Web UI: browse to https://app.rocketship.sh/ in an incognito session and authenticate via GitHub. Once approved, oauth2-proxy maintains the session (\_rocketship_auth cookie) while forwarding requests to the engine HTTP port.

First-time logins return a pending role. Call POST https://auth.rocketship.sh/api/orgs with the bearer token to create the first organisation/project, or accept an invitation from an existing admin before
running suites.

## 8. Bring Your Own IdP (Optional)

If your organisation mandates an internal IdP (Auth0, Okta, Azure AD, …), update values-oidc-web.yaml with the issuer, client IDs, and scopes, and deploy:

```bash
helm upgrade --install rocketship charts/rocketship \
 --namespace rocketship \
 -f charts/rocketship/values-production.yaml \
 -f charts/rocketship/values-oidc-web.yaml \
 --set postgres.enabled=true \
 --set postgres.auth.existingSecret=rocketship-postgres-auth \
 --set postgres.auth.existingSecretPasswordKey=postgres-password \
 --wait
```

Rocketship’s engine trusts whatever JWTs your IdP mints; the RBAC checks are the same.

### RBAC considerations

- Tokens carry organisation + project roles (read, write, owner, service_account). Engine interceptors reject RPCs that lack the necessary role.
- Pending users must create or join an organisation via the broker API before they can run suites.
- The broker persists all roles in Postgres, so every deployment (cloud or self-hosted) enforces the same claim checks.

## 9. Point DNS at the Load Balancer

Create A (or CNAME) records for cli.rocketship.sh, app.rocketship.sh, and auth.rocketship.sh pointing at the ingress load balancer IP. DigitalOcean DNS usually updates within a minute; public resolvers may
take longer.

## 10. Smoke Test the Endpoints

```bash
curl -v https://cli.rocketship.sh/healthz # returns 415 (gRPC)
curl -v https://auth.rocketship.sh/healthz # broker health
```

Use the CLI profile to run a simple suite:

```bash
rocketship profile list
rocketship login
rocketship run -f examples/simple-http/rocketship.yaml
```

If you see connection refused against 127.0.0.1:7700, confirm the profile points at grpcs://cli.rocketship.sh (port 443) and DNS has propagated.

## 11. Updating the Deployment

1. Tag a new release in GitHub (git tag v0.x.y && git push --tags).
2. The release pipeline builds/pushes Docker Hub images and opens a PR in the internal chart repo bumping image tags.
3. Merge the PR, then upgrade:

```bash
   helm upgrade rocketship charts/rocketship \
    --namespace rocketship \
    -f charts/rocketship/values-production.yaml \
    -f charts/rocketship/values-github-cloud.yaml \
    -f charts/rocketship/values-github-web.yaml \
    --wait
```

4. Monitor rollouts:

```bash
   kubectl rollout status deploy/rocketship-engine -n rocketship
   kubectl rollout status deploy/rocketship-worker -n rocketship
```

## 12. Troubleshooting Tips

- CrashLoopBackOff with exec format error → image pulled for wrong architecture; rebuild with --platform linux/amd64.
- Worker logs show Namespace <name> is not found → run the Temporal namespace registration step and ensure temporal.namespace matches.
- pending role on login → call POST /api/orgs or accept an invitation before running suites.
- grpc: received message larger than max → adjust ingress/proxy body size annotations if you push large suites.

With these steps you have a durable Rocketship installation bridging a managed Temporal stack, ingress TLS, GitHub SSO, and the CLI profile system—ready for teams to run suites from their laptops or CI
pipelines.
