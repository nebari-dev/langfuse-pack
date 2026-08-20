# nebari-langfuse

A Nebari software pack that deploys [Langfuse](https://langfuse.com) - an open-source LLM
observability, tracing, evaluation, and prompt-management platform - on a Nebari cluster with
full Keycloak SSO integration.

This pack wraps the upstream `langfuse/langfuse` Helm chart **1.5.34** (Langfuse app version
**3.179.1**). Declared maturity level: **Beta**.

---

## Prerequisites

### Nebari deployment

- A running Nebari cluster managed by nebari-operator **v0.1.0-alpha.19 or later**
- cert-manager installed and able to issue certificates for your domain
- Envoy Gateway installed (the NebariApp CRD uses it for routing)
- Keycloak installed and accessible (the operator provisions an OIDC client)
- The target namespace labeled `nebari.dev/managed=true`:
  ```bash
  kubectl label namespace <namespace> nebari.dev/managed=true
  ```
- Helm 3

### Standalone deployment

Only a Kubernetes cluster and Helm 3 are required. No Keycloak or Envoy Gateway needed.

---

## Quick Start (Nebari)

Add the Nebari Helm repository and install the pack. The upstream `langfuse/langfuse` chart is
a transitive dependency - you only need to add the Nebari repo.

```bash
helm repo add nebari https://nebari-dev.github.io/helm-repository
helm repo update

helm install langfuse nebari/nebari-langfuse \
  -f examples/nebari-values.yaml \
  --set nebariapp.hostname=langfuse.<your-domain> \
  --set langfuse.langfuse.nextauth.url=https://langfuse.<your-domain> \
  --set langfuse.langfuse.auth.providers.keycloak.issuer=https://keycloak.<your-domain>/realms/nebari \
  --namespace <namespace>
```

Using release name `langfuse` is recommended. The pack pins
`langfuse.fullnameOverride: langfuse` so the web Service is always named `langfuse-web`.

See `examples/nebari-values.yaml` for a minimal values file to use with `-f`.

---

## Value Nesting

The upstream chart dependency is named `langfuse`, **and** the upstream chart also uses a
top-level `langfuse` key for app configuration. This creates double-nesting:

| What you are setting | Path from this wrapper |
|----------------------|------------------------|
| App config (auth, nextauth, secrets, ...) | `langfuse.langfuse.*` |
| PostgreSQL sub-chart | `langfuse.postgresql.*` |
| Redis sub-chart | `langfuse.redis.*` |
| ClickHouse sub-chart | `langfuse.clickhouse.*` |
| S3/MinIO sub-chart | `langfuse.s3.*` |
| Chart-level (fullnameOverride) | `langfuse.fullnameOverride` |

Example:

```yaml
langfuse:
  langfuse:              # double-nested: app config
    nextauth:
      url: https://langfuse.example.com
    auth:
      providers:
        keycloak:
          issuer: https://keycloak.example.com/realms/nebari
  postgresql:            # single-nested: sub-chart
    deploy: true
```

See the [configuration reference](https://packs.nebari.dev/langfuse-pack/configuration/)
(source: `docs/src/content/docs/configuration.md`) for the full values reference.

---

## Authentication

Langfuse uses app-native OAuth (NextAuth) with Keycloak as the provider.

**How it works:**

1. Set `nebariapp.auth.enabled: true` in your values file. The pack sets
   `nebariapp.auth.provisionClient: true` by default, which instructs the nebari-operator to
   create a Keycloak client automatically.
2. The operator registers the redirect URI
   `https://<hostname>/api/auth/callback/keycloak` (and the `http://` variant) on the
   provisioned Keycloak client. **No manual Keycloak step is needed** when
   `provisionClient: true`.
3. You **must** set `langfuse.langfuse.auth.providers.keycloak.issuer` to your Keycloak realm
   URL. The operator does not reliably emit the issuer URL into the OIDC secret, so this value
   cannot be read automatically:

   ```bash
   --set langfuse.langfuse.auth.providers.keycloak.issuer=https://keycloak.<your-domain>/realms/nebari
   ```

   Or in your values file:

   ```yaml
   langfuse:
     langfuse:
       auth:
         providers:
           keycloak:
             issuer: "https://keycloak.<your-domain>/realms/nebari"
   ```

4. The `clientId` and `clientSecret` are read automatically from the
   `langfuse-oidc-client` secret that the operator creates.

**Note on `enforceAtGateway`:** This is set to `false` by default. Langfuse handles OAuth
itself (app-native). The Envoy SecurityPolicy is not applied to the route.

If you use `provisionClient: false` with a manually created Keycloak client, you must
add `https://<hostname>/api/auth/callback/keycloak` to that client's Valid Redirect URIs
yourself.

---

## Secrets

On first install, the pack generates a Kubernetes Secret named `langfuse-secrets` containing:

| Key | Description |
|-----|-------------|
| `salt` | Langfuse password hashing salt |
| `encryptionKey` | 64-character hex key for encrypting integration credentials |
| `nextauth-secret` | NextAuth.js session signing key |
| `postgres-password` | PostgreSQL password |
| `redis-password` | Redis password |
| `clickhouse-password` | ClickHouse password |
| `root-user` | MinIO/S3 root username |
| `root-password` | MinIO/S3 root password |

The secret is persisted using `helm.sh/resource-policy: keep` and Helm lookup so it survives
`helm upgrade` without rotation.

**WARNING:** Rotating `encryptionKey` orphans any integration credentials already encrypted
with the old key. Do not rotate it unless you are prepared to re-enter all integrations.

### GitOps / ArgoCD caveat

ArgoCD's `helm template` rendering cannot perform cluster API lookups, so the auto-generation
path (`secrets.generate: true`) does not work in a pure GitOps setup. For production or GitOps:

1. Set `secrets.generate: false` in your values.
2. Create the `langfuse-secrets` Secret manually before the first sync:
   ```bash
   kubectl create secret generic langfuse-secrets \
     --from-literal=salt=$(openssl rand -hex 16) \
     --from-literal=encryptionKey=$(openssl rand -hex 32) \
     --from-literal=nextauth-secret=$(openssl rand -hex 32) \
     --from-literal=postgres-password=$(openssl rand -hex 16) \
     --from-literal=redis-password=$(openssl rand -hex 16) \
     --from-literal=clickhouse-password=$(openssl rand -hex 16) \
     --from-literal=root-user=langfuse \
     --from-literal=root-password=$(openssl rand -hex 16) \
     -n <namespace>
   ```
3. If needed, add an ArgoCD `ignoreDifferences` entry for the Secret resource so ArgoCD
   does not flag the pre-created secret as out-of-sync.

---

## Datastores

By default the pack deploys bundled `bitnamilegacy/*` images for PostgreSQL, Redis,
ClickHouse, and MinIO. These are suitable for development and demos.

**For production**, use external managed services. See `examples/prod-external-datastores.yaml`
for a complete example. Disable each bundled datastore with `deploy: false`:

```yaml
langfuse:
  postgresql:
    deploy: false
    host: <managed-postgres-host>
    port: 5432
    auth:
      username: langfuse
      database: langfuse
      existingSecret: <your-secret>
  redis:
    deploy: false
    host: <managed-redis-host>
  clickhouse:
    deploy: false
    host: <managed-clickhouse-host>
  s3:
    deploy: false
    bucket: <your-bucket>
```

---

## Standalone / Local

To run Langfuse without Nebari integration (email/password auth, bundled datastores):

```bash
helm repo add nebari https://nebari-dev.github.io/helm-repository
helm repo update

helm install langfuse nebari/nebari-langfuse \
  -f examples/standalone-values.yaml
```

Once deployed, forward the web port:

```bash
kubectl port-forward svc/langfuse-web 3000:3000
```

Then open `http://localhost:3000` and sign up with email/password.

---

## Known Limitations

- **Frozen bitnamilegacy images:** The bundled PostgreSQL, Redis, ClickHouse, and MinIO images
  are frozen `bitnamilegacy/*` variants pinned by upstream chart 1.5.34. They do not receive
  ongoing security patches. Use external managed datastores in production.

- **Generated secrets incompatible with ArgoCD GitOps:** The `secrets.generate: true`
  path uses a Helm cluster lookup that ArgoCD's `helm template` renderer cannot execute. For
  GitOps or production deployments, set `secrets.generate: false` and pre-create the
  `langfuse-secrets` Secret as described above.

- **Must set AUTH_KEYCLOAK_ISSUER manually:** The nebari-operator does not reliably emit the
  Keycloak issuer URL into the OIDC secret. You must set
  `langfuse.langfuse.auth.providers.keycloak.issuer` to your realm URL explicitly.

- **ClickHouse runs single-node by default:** This pack defaults to a single, non-clustered
  ClickHouse (`langfuse.clickhouse.clusterEnabled: false`, `replicaCount: 1`,
  `zookeeper.enabled: false`). The upstream chart defaults to a 3-replica cluster, but
  Langfuse runs its schema migrations over the load-balanced ClickHouse Service and the
  `schema_migrations` bookkeeping is not consistently replicated across replicas, so
  migrations get marked "dirty" and crashloop `langfuse-web`. Single-node avoids this and is
  far lighter. For HA, set `clusterEnabled: true`, `replicaCount: 3`, `zookeeper.enabled: true`
  (and accept the migration caveat), or use an external managed ClickHouse.

- **No native Prometheus metrics endpoint:** Langfuse does not expose a `/metrics` endpoint
  by default. The `metrics.podMonitor.enabled` opt-in (default `false`) is a hook for a future
  sidecar exporter. Enabling it without a metrics endpoint produces scrape errors.

---

## Troubleshooting

### NebariApp not reaching Ready

Check the NebariApp status and events:

```bash
kubectl describe nebariapp langfuse -n <namespace>
```

Common conditions and fixes:

- **`NamespaceNotOptedIn`**: The namespace is missing the required label.
  ```bash
  kubectl label namespace <namespace> nebari.dev/managed=true
  ```

- **`ServiceNotFound`**: The `langfuse-web` Service is missing. Check that the Helm release
  deployed successfully and that `langfuse.fullnameOverride: langfuse` is set (it is the
  default in this pack):
  ```bash
  kubectl get svc -n <namespace>
  helm status langfuse -n <namespace>
  ```

### Image pull failures

The bundled datastores use `bitnamilegacy/*` images from Docker Hub. If the cluster has
restricted egress or Docker Hub rate limits apply, pulls may fail:

```bash
kubectl get events -n <namespace> --field-selector reason=Failed
kubectl describe pod <pod-name> -n <namespace>
```

For restricted environments, mirror the images to an internal registry and override the image
repository in your values file, or switch to external datastores.

### Login / redirect mismatch (redirect_uri_mismatch)

This only applies when `nebariapp.auth.provisionClient: false`. If you created the Keycloak
client manually, ensure its Valid Redirect URIs include:

```
https://<hostname>/api/auth/callback/keycloak
```

When `provisionClient: true` (the default), the operator registers this URI automatically.

### Web pod crashloop: ENCRYPTION_KEY length error

If the `langfuse-web` pod crashloops with an error about `ENCRYPTION_KEY` being the wrong
length, the `langfuse-secrets` Secret is missing or has a malformed key. Verify:

```bash
kubectl get secret langfuse-secrets -n <namespace> -o jsonpath='{.data.encryptionKey}' | base64 -d | tr -d '\n' | wc -c
```

The decoded value must be exactly 64 hex characters (`wc -c` should print `64`). If missing,
check `secrets.generate` and recreate the secret if needed.

### ClickHouse not Ready on small clusters

ClickHouse requests significant CPU and memory. On small clusters (fewer than 3 schedulable
nodes), pods may stay Pending:

```bash
kubectl get pods -n <namespace> -l app.kubernetes.io/name=clickhouse
kubectl describe pod <clickhouse-pod> -n <namespace>
```

Reduce to a single replica for development:

```yaml
langfuse:
  clickhouse:
    replicaCount: 1
```

Or disable the bundled ClickHouse and point to an external instance.

---

## Telemetry

**Metrics:** Langfuse does not expose a native Prometheus `/metrics` endpoint. The pack
includes an opt-in `metrics.podMonitor.enabled` flag (default `false`) as a hook for a future
sidecar exporter. Enabling it requires Prometheus Operator CRDs to be installed. Do not enable
it without a corresponding metrics exporter.

**Logs:** Langfuse writes structured JSON logs to stdout/stderr on the web and worker
containers. These are captured by any standard cluster log shipper (Fluentd, Vector, etc.)
targeting pod stdout.

For sending traces from the Nebari OTel Collector to Langfuse, see
[configuration reference](https://packs.nebari.dev/langfuse-pack/configuration/#otel-collector-export-to-langfuse).

---

## Additional Documentation

- [Configuration reference](https://packs.nebari.dev/langfuse-pack/configuration/)
  (`docs/src/content/docs/configuration.md`) - Full values reference, auth wiring, secrets,
  external datastores, telemetry, and OTel Collector integration.
- `tests/e2e/README.md` - End-to-end test suite for this pack.
- `examples/nebari-values.yaml` - Minimal Nebari deployment values.
- `examples/standalone-values.yaml` - Standalone / local deployment values.
- `examples/prod-external-datastores.yaml` - Production values with external managed datastores.
- `examples/argocd-app.yaml` - ArgoCD Application manifest.

## Documentation

The docs site lives in [`docs/`](docs/) and is built with [Astro](https://astro.build) +
[Starlight](https://starlight.astro.build) using the shared `@nebari/starlight` theme. It
deploys to [packs.nebari.dev/langfuse-pack/](https://packs.nebari.dev/langfuse-pack/) on
every merge to `main`; pull requests that touch `docs/` get a preview URL posted as a
comment.

```bash
cd docs
npm ci
npm run dev     # dev server with hot reload at http://localhost:4321
npm run build   # static build into docs/dist/
npm test        # unit tests
```

Pages live in `docs/src/content/docs/` - each `.md` or `.mdx` file becomes a page, and the
sidebar is configured in `docs/astro.config.mjs`. See [`docs/README.md`](docs/README.md) for
details.
