# Deploy Rocketship on DigitalOcean Kubernetes

This guide walks through deploying the Rocketship chart on DigitalOcean Kubernetes (DOKS). It assumes you are working from the `rocketship-internal` repository, which already contains production presets for the cloud environment. The chart pulls images from Docker Hub (`rocketshipai/...`), so you do not need a separate registry.

We will:

1. Install ingress-nginx and Temporal.
2. Create required TLS and application secrets.
3. Deploy the Rocketship chart (engine, worker, auth broker, optional web proxy, and bundled Postgres).
4. Verify the deployment and note the day-to-day maintenance steps.

## Prerequisites

- A DigitalOcean Kubernetes cluster (the reference setup uses a 2-node CPU-optimised pool).
- [`doctl`](https://docs.digitalocean.com/reference/doctl/how-to/install/) authenticated with your account.
- `kubectl` configured for the cluster (`doctl kubernetes cluster kubeconfig save <cluster-name>`).
- Helm 3.
- TLS certificate and key for the public endpoints (`cli.rocketship.sh`, `app.rocketship.sh`, `auth.rocketship.sh`).

All commands below run from the root of the `rocketship-internal` repository unless stated otherwise.

## 1. Install ingress-nginx

```bash
kubectl create namespace rocketship
kubectl config set-context --current --namespace=rocketship

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.13.2 \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/do-loadbalancer-enable-proxy-protocol"="true"
```

> The annotation keeps client IP addresses visible in the ingress logs. Remove it if you do not need PROXY protocol support.

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

Register the logical namespace the worker will use:

```bash
kubectl exec -n rocketship deploy/temporal-admintools -- \
  temporal operator namespace create --namespace default
```

## 3. TLS secret for ingress

```bash
kubectl delete secret rocketship-cloud-tls -n rocketship 2>/dev/null || true
kubectl create secret tls rocketship-cloud-tls \
  --namespace rocketship \
  --cert=/path/to/fullchain.pem \
  --key=/path/to/privkey.pem
```

## 4. Postgres and broker secrets

The chart bundles the Bitnami Postgres statefulset. For now we pin the legacy `bitnamilegacy/postgresql:16.3.0-debian-12-r23` image because Bitnami retired the original tag from Docker Hub. Replace it with a managed database or newer image as soon as you migrate.

Create the secrets the chart expects:

```bash
# PostgreSQL passwords (application user + superuser)
kubectl create secret generic rocketship-postgres-auth \
  --namespace rocketship \
  --from-literal=password='<postgres-password>' \
  --from-literal=postgres-password='<postgres-password>'

# Auth broker DSN (points at the statefulset service)
kubectl create secret generic rocketship-auth-broker-database \
  --namespace rocketship \
  --from-literal=DATABASE_URL='postgres://rocketship:<postgres-password>@rocketship-postgresql:5432/rocketship?sslmode=disable'

# Refresh-token HMAC key (32 bytes, Base64 encoded)
kubectl create secret generic rocketship-auth-broker-secrets \
  --namespace rocketship \
  --from-literal=ROCKETSHIP_BROKER_REFRESH_KEY="$(openssl rand -base64 32)"
```

## 5. Auth broker signing key and GitHub OAuth secrets

```bash
# RSA signing key for JWTs
openssl genrsa -out signing-key.pem 2048
kubectl create secret generic rocketship-auth-broker-signing \
  --namespace rocketship \
  --from-file=signing-key.pem

# GitHub OAuth (device flow). Enable Device Flow in the app settings.
kubectl create secret generic rocketship-github-oauth \
  --namespace rocketship \
  --from-literal=ROCKETSHIP_GITHUB_CLIENT_SECRET='<github-client-secret>'

# oauth2-proxy (web flows) – use the same GitHub App so UI and CLI share a client
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

> If you previously created the deprecated `rocketship-auth-broker-store` secret, you can delete it once the new deployment is healthy.

## 6. (Optional) Refresh chart dependencies

The repository already vendors the Bitnami Postgres chart (`charts/postgresql-15.5.20.tgz`). If you want to rebuild the tarball after changing the dependency, run:

```bash
helm dependency update charts/rocketship
```

## 7. Deploy Rocketship

The production presets already reference the secret names above and configure the bundled Postgres. Deploy with:

```bash
helm upgrade --install rocketship charts/rocketship \
  --namespace rocketship \
  -f charts/rocketship/values-production.yaml \
  -f charts/rocketship/values-github-cloud.yaml \
  -f charts/rocketship/values-github-web.yaml \
  --wait
```

If you prefer not to bake the GitHub Client ID into version control, remove it from `values-github-cloud.yaml` and supply it at install time via `--set auth.broker.github.clientID=<github-client-id>`.

### Secret recap

| Secret name                        | Purpose                                      |
| ---------------------------------- | -------------------------------------------- |
| `rocketship-postgres-auth`         | Postgres passwords (`password`, `postgres-password`) |
| `rocketship-auth-broker-database`  | Broker DSN (`DATABASE_URL`)                  |
| `rocketship-auth-broker-secrets`   | Refresh-token HMAC key                       |
| `rocketship-auth-broker-signing`   | RSA signing key                              |
| `rocketship-github-oauth`          | Device-flow GitHub OAuth client secret       |
| `oauth2-proxy-credentials`         | Web OAuth client ID/secret/cookie secret     |
| `rocketship-cloud-tls`             | TLS cert for ingress                         |

## 8. Verify pods

```bash
kubectl get pods -n rocketship
```

`rocketship-engine`, `rocketship-worker`, `rocketship-auth-broker`, `rocketship-web-oauth2-proxy`, and `rocketship-postgresql-0` should all report `READY 1/1` once the StatefulSet finishes initialising. Temporal components may restart once while Cassandra and Elasticsearch bootstrap.

## 9. CLI and web login

```bash
rocketship profile create cloud grpcs://cli.rocketship.sh
rocketship profile use cloud
rocketship login
rocketship status
```

Visit `https://app.rocketship.sh/` in a new browser session to confirm the oauth2-proxy round-trip. First-time logins receive a `pending` role; either create the first organisation via `POST https://auth.rocketship.sh/api/orgs` with your bearer token, or invite the user from another admin account before running suites.

## 10. Updating the deployment

1. Create a release tag in the public repository (`git tag vX.Y.Z && git push --tags`).
2. The release workflow publishes Docker Hub images and opens a PR in `rocketship-internal` bumping `charts/rocketship/values-production.yaml` to the new tag.
3. Merge the PR, then roll out the new version:
   ```bash
   helm upgrade rocketship charts/rocketship \
     --namespace rocketship \
     -f charts/rocketship/values-production.yaml \
     -f charts/rocketship/values-github-cloud.yaml \
     -f charts/rocketship/values-github-web.yaml \
     --wait
   ```
4. Monitor rollout status (`kubectl rollout status deploy/rocketship-engine -n rocketship`, etc.).

## 11. Troubleshooting

- **Auth broker stuck in `CrashLoopBackOff` with password errors** – ensure the `rocketship-postgres-auth` secret has both keys (`password`, `postgres-password`). If you deployed before wiring the secret, the StatefulSet may have persisted a random password; delete the `rocketship-postgresql` StatefulSet and its PVC to reinitialise with your credentials.
- **Postgres image pull failures** – the chart currently pins `bitnamilegacy/postgresql:16.3.0-debian-12-r23`. Update `postgresql.image.repository/tag` if you migrate to a managed database or a newer image.
- **Pending role after login** – call `POST /api/orgs` with your access token or accept an invitation before running suites.
- **Temporal namespace errors** – rerun the namespace registration step in section 2.

With these steps, the hosted chart provides a ready-made Rocketship environment backed by Temporal, ingress-nginx, GitHub SSO, and a Postgres database suitable for development or proof-of-concept use on DigitalOcean Kubernetes.
