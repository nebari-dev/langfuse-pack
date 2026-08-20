---
title: Configuration reference
description: Every configuration surface for the nebari-langfuse pack - NebariApp values, Langfuse passthrough values, authentication, secrets, external datastores, and telemetry.
---

This document covers all configuration surfaces for the `nebari-langfuse` pack.

For installation instructions and quick-start examples, see `README.md`.

---

## Table of Contents

1. [NebariApp values reference](#nebariapp-values-reference)
2. [Langfuse passthrough values](#langfuse-passthrough-values)
3. [Authentication setup](#authentication-setup)
4. [Secrets](#secrets)
5. [External datastores](#external-datastores)
6. [Telemetry and security](#telemetry-and-security)
7. [OTel Collector export to Langfuse](#otel-collector-export-to-langfuse)

---

## NebariApp values reference

These values configure the NebariApp CRD resource and the Nebari-specific integration layer.
They have no effect when `nebariapp.enabled: false`.

| Value | Type | Default | Description |
|-------|------|---------|-------------|
| `nebariapp.enabled` | bool | `false` | Create a NebariApp resource. Set `true` on Nebari clusters. |
| `nebariapp.name` | string | `langfuse` | NebariApp resource name. Changing this also changes the OIDC secret name (`<name>-oidc-client`); update the `secretKeyRef` names in your auth values to match. |
| `nebariapp.hostname` | string | (required when enabled) | FQDN at which Langfuse is reachable, e.g. `langfuse.nebari.example.com`. Must match `langfuse.langfuse.nextauth.url`. |
| `nebariapp.service.name` | string | `langfuse-web` | Kubernetes Service name the NebariApp routes traffic to. Pinned by `langfuse.fullnameOverride`. |
| `nebariapp.service.port` | int | `3000` | Port on the target Service. |
| `nebariapp.auth.enabled` | bool | `false` | Enable Keycloak OAuth integration. Set `true` on Nebari to provision the OIDC client. |
| `nebariapp.auth.provider` | string | `keycloak` | OAuth provider. Only `keycloak` is supported. |
| `nebariapp.auth.provisionClient` | bool | `true` | Let the nebari-operator create and manage the Keycloak client. Set `false` if you manage the Keycloak client manually. |
| `nebariapp.auth.enforceAtGateway` | bool | `false` | Apply an Envoy SecurityPolicy to enforce auth at the gateway layer. Left `false` because Langfuse handles OAuth natively. |
| `nebariapp.auth.redirectURI` | string | `/api/auth/callback/keycloak` | OAuth redirect path registered on the Keycloak client. Must match Langfuse's NextAuth callback path. |
| `nebariapp.auth.scopes` | list | `[openid, profile, email]` | OIDC scopes requested. These are the minimum required for Langfuse user identity. |
| `nebariapp.auth.groups` | list | `[]` | (Optional) Restrict access to users in these Keycloak groups. |
| `nebariapp.landingPage.enabled` | bool | `false` | Show Langfuse on the Nebari landing page. |
| `nebariapp.landingPage.displayName` | string | `Langfuse` | Display name on the landing page tile. |
| `nebariapp.landingPage.description` | string | `LLM observability, tracing, and prompt management.` | Short description shown on the tile. |
| `nebariapp.landingPage.icon` | string | `grafana` | Icon identifier for the landing page tile. |
| `nebariapp.landingPage.category` | string | `Monitoring` | Landing page category grouping. |
| `nebariapp.landingPage.priority` | int | `100` | Sort order within the category (lower number = higher position). |
| `nebariapp.gateway` | string | `public` | Envoy Gateway to attach the HTTPRoute to. |

---

## Langfuse passthrough values

Values under `langfuse.*` are passed through to the upstream `langfuse/langfuse` chart
dependency. Because the dependency is named `langfuse` and the upstream chart also uses a
top-level `langfuse` app-config key, app configuration is double-nested:

```
langfuse.langfuse.*   ->  upstream chart's langfuse.* (app config)
langfuse.postgresql.* ->  upstream chart's postgresql.* (sub-chart)
langfuse.redis.*      ->  upstream chart's redis.*      (sub-chart)
langfuse.clickhouse.* ->  upstream chart's clickhouse.* (sub-chart)
langfuse.s3.*         ->  upstream chart's s3.*         (sub-chart)
langfuse.fullnameOverride -> upstream chart's fullnameOverride
```

### Commonly customized app-config values

| Value | Default | Description |
|-------|---------|-------------|
| `langfuse.langfuse.nextauth.url` | `http://localhost:3000` | The public URL of your Langfuse instance. Must equal `https://<nebariapp.hostname>` on Nebari. The upstream chart cannot template this from `nebariapp.hostname`, so set it explicitly. |
| `langfuse.langfuse.auth.disableUsernamePassword` | `true` | Set `false` to allow email/password login alongside SSO. Default `true` forces SSO-only on Nebari. |
| `langfuse.langfuse.auth.providers.keycloak.issuer` | `"https://REPLACE-ME/realms/nebari"` | Keycloak realm URL (the issuer, `AUTH_KEYCLOAK_ISSUER`). Must be set to your actual realm URL. See Authentication setup below. |
| `langfuse.langfuse.auth.providers.keycloak.allowAccountLinking` | `"true"` | Allow linking existing accounts to the Keycloak identity. |

### Datastore deploy and connection values

Each datastore can be bundled (default, dev) or external (recommended for production).

| Value | Default | Description |
|-------|---------|-------------|
| `langfuse.postgresql.deploy` | `true` | Deploy bundled PostgreSQL. Set `false` to use an external instance. |
| `langfuse.postgresql.host` | (unset) | External PostgreSQL hostname (required when `deploy: false`). |
| `langfuse.postgresql.auth.username` | `postgres` | Database user. |
| `langfuse.postgresql.auth.database` | `postgres_langfuse` | Database name. |
| `langfuse.postgresql.auth.existingSecret` | `langfuse-secrets` | Secret containing the password. |
| `langfuse.redis.deploy` | `true` | Deploy bundled Redis. Set `false` to use an external instance. |
| `langfuse.redis.host` | (unset) | External Redis hostname (required when `deploy: false`). |
| `langfuse.redis.auth.existingSecret` | `langfuse-secrets` | Secret containing the Redis password. |
| `langfuse.clickhouse.deploy` | `true` | Deploy bundled ClickHouse. Set `false` to use an external instance. |
| `langfuse.clickhouse.host` | (unset) | External ClickHouse hostname (required when `deploy: false`). |
| `langfuse.clickhouse.replicaCount` | `3` | Number of ClickHouse replicas. Reduce to `1` on small clusters. |
| `langfuse.clickhouse.auth.existingSecret` | `langfuse-secrets` | Secret containing the ClickHouse password. |
| `langfuse.s3.deploy` | `true` | Deploy bundled MinIO. Set `false` to use an external S3-compatible store. |
| `langfuse.s3.bucket` | `langfuse` | S3 bucket name. |

---

## Authentication setup

This pack uses Langfuse's app-native OAuth (NextAuth.js) with Keycloak as the provider.
The NebariApp CRD and nebari-operator handle Keycloak client provisioning.

### How the provider environment variables are set

The upstream chart translates the following values into environment variables on the
web and worker containers:

| Value path | Environment variable | Source |
|------------|----------------------|--------|
| `langfuse.langfuse.auth.providers.keycloak.clientId` | `AUTH_KEYCLOAK_ID` | `langfuse-oidc-client` Secret, key `client-id` (operator-provisioned) |
| `langfuse.langfuse.auth.providers.keycloak.clientSecret` | `AUTH_KEYCLOAK_SECRET` | `langfuse-oidc-client` Secret, key `client-secret` (operator-provisioned) |
| `langfuse.langfuse.auth.providers.keycloak.issuer` | `AUTH_KEYCLOAK_ISSUER` | **Must be set statically** in your values file |
| `langfuse.langfuse.auth.providers.keycloak.allowAccountLinking` | `AUTH_KEYCLOAK_ALLOW_ACCOUNT_LINKING` | Defaults to `"true"` |

The `clientId` and `clientSecret` are read automatically from the `langfuse-oidc-client` Secret
that the nebari-operator creates when `nebariapp.auth.provisionClient: true`.

The `issuer` cannot be read automatically because the operator does not reliably emit the
issuer URL into the OIDC Secret. Set it explicitly in your values file:

```yaml
langfuse:
  langfuse:
    auth:
      providers:
        keycloak:
          issuer: "https://keycloak.<your-domain>/realms/nebari"
```

### Redirect URI registration

When `nebariapp.auth.provisionClient: true`, the nebari-operator calls the Keycloak API to
register the redirect URI on the provisioned client. The operator reads `nebariapp.auth.redirectURI`
(default `/api/auth/callback/keycloak`) and registers exactly:

```
https://<nebariapp.hostname>/api/auth/callback/keycloak
http://<nebariapp.hostname>/api/auth/callback/keycloak
```

This matches Langfuse's NextAuth callback path. **No manual Keycloak configuration step is
needed** when `provisionClient: true`.

If you use `provisionClient: false` with a manually created Keycloak client, add the URI
`https://<hostname>/api/auth/callback/keycloak` to that client's Valid Redirect URIs in the
Keycloak admin console.

### Disabling username/password login

By default (`auth.disableUsernamePassword: true`), email/password login is disabled and users
must authenticate via Keycloak. For local development without SSO, set:

```yaml
langfuse:
  langfuse:
    auth:
      disableUsernamePassword: false
      providers: {}
```

---

## Secrets

### The `langfuse-secrets` Secret

When `secrets.generate: true` (the default), the pack generates a Kubernetes Secret named
`langfuse-secrets` on first install using Helm's `lookup` function with `randAlphaNum`. The
secret is annotated with `helm.sh/resource-policy: keep` so it is not deleted on
`helm uninstall` and is not rotated on `helm upgrade`.

| Key | Usage |
|-----|-------|
| `salt` | Langfuse password hashing salt |
| `encryptionKey` | 64-character hex key for encrypting stored integration credentials |
| `nextauth-secret` | NextAuth.js session signing key |
| `postgres-password` | PostgreSQL user password |
| `redis-password` | Redis auth password |
| `clickhouse-password` | ClickHouse user password |
| `root-user` | MinIO root username |
| `root-password` | MinIO root password |

### encryptionKey rotation warning

**Do not rotate `encryptionKey` unless you are prepared to re-enter all integrations.**
Langfuse uses this key to encrypt credentials stored in its database (e.g., Slack, cloud
provider keys). Rotating the key orphans all previously encrypted data - the credentials
will be unreadable and must be re-entered manually.

### GitOps / ArgoCD caveat

ArgoCD's `helm template` renderer runs without cluster access, so the `lookup`-based
secret generation path does not execute. The secret will not be created, and pods will
fail to start.

**Production and GitOps path:**

1. Set `secrets.generate: false` in your values or ArgoCD Application spec.
2. Create the `langfuse-secrets` Secret before the first ArgoCD sync:

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

3. To prevent ArgoCD from flagging the pre-created Secret as out-of-sync, add an
   `ignoreDifferences` entry to your ArgoCD Application:

   ```yaml
   ignoreDifferences:
     - group: ""
       kind: Secret
       name: langfuse-secrets
       jsonPointers:
         - /data
   ```

See `examples/argocd-app.yaml` for a full ArgoCD Application manifest.

---

## External datastores

For production deployments, disable the bundled `bitnamilegacy/*` images and configure
external managed services. See `examples/prod-external-datastores.yaml` for a complete
worked example.

Set `deploy: false` for each datastore you are replacing, then provide connection details:

```yaml
langfuse:
  postgresql:
    deploy: false
    host: <managed-postgres-host>
    port: 5432
    auth:
      username: langfuse
      database: langfuse
      existingSecret: <your-postgres-secret>
      secretKeys:
        userPasswordKey: password

  redis:
    deploy: false
    host: <managed-redis-host>
    port: 6379
    auth:
      existingSecret: <your-redis-secret>
      existingSecretPasswordKey: password

  clickhouse:
    deploy: false
    host: <managed-clickhouse-host>
    port: 9000
    auth:
      existingSecret: <your-clickhouse-secret>
      existingSecretKey: password

  s3:
    deploy: false
    bucket: <your-bucket-name>
    region: <aws-region>
    endpoint: ""          # leave empty for AWS S3; set for S3-compatible services
    accessKeyId:
      secretKeyRef:
        name: <your-s3-secret>
        key: access-key-id
    secretAccessKey:
      secretKeyRef:
        name: <your-s3-secret>
        key: secret-access-key
```

When all bundled datastores are disabled, the pack deploys only the Langfuse web and worker
containers.

---

## Telemetry and security

### Metrics

The upstream `langfuse/langfuse` chart does not expose a native Prometheus `/metrics`
endpoint on the web or worker containers, and ships no ServiceMonitor or PodMonitor.

This pack provides an opt-in `metrics.podMonitor.enabled` flag (default `false`) as a hook
for a future sidecar exporter. Do not enable it unless you have added a Prometheus-compatible
metrics exporter sidecar to the pods, as it will produce scrape errors against a non-existent
endpoint.

To enable the PodMonitor (requires Prometheus Operator CRDs installed):

```yaml
metrics:
  podMonitor:
    enabled: true
    interval: 30s
    path: /metrics
    port: http
```

This satisfies the Beta telemetry requirement with a documented justification for
default-off behavior.

### Structured logging

Langfuse web and worker containers emit JSON-structured logs to stdout and stderr by default.
No log-shipping sidecar is required - any standard cluster log aggregator (Fluentd, Vector,
Loki promtail, etc.) that reads pod stdout will capture structured logs automatically.

### Pod security context

The pack sets the following security context on Langfuse web and worker pods via the
upstream chart's `podSecurityContext` and `securityContext` passthrough values:

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  capabilities:
    drop: ["ALL"]
```

The bundled Bitnami sub-chart images (PostgreSQL, Redis, ClickHouse, MinIO) already run as
non-root by default.

---

## OTel Collector export to Langfuse

The Nebari foundational stack includes an OpenTelemetry Collector deployed as a DaemonSet
in the `monitoring` namespace. Its configuration is managed as a `helm.values:` block inside
an ArgoCD Application at `apps/opentelemetry-collector.yaml` in the
`nebari-infrastructure-core`-generated foundational-software GitOps repository. It is not
an OpenTelemetryCollector CR and not a standalone ConfigMap.

The collector already has a `traces` pipeline that receives OTLP telemetry and currently
exports only to `debug` (stdout). The steps below add Langfuse as an additional exporter for
that pipeline.

### Steps

**1. Create a Langfuse project and API key pair**

In the Langfuse UI (Projects - Settings - API Keys), create a new key pair. You will receive:
- A public key: `pk-lf-...`
- A secret key: `sk-lf-...`

**2. Compute the Basic auth header value**

```bash
echo -n "pk-lf-<your-public-key>:sk-lf-<your-secret-key>" | base64
```

Note: if your setup supports Kubernetes Secrets and the OTel Collector
[`headers_setter` extension](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/extension/headerssetterextension),
prefer storing the credentials in a Secret rather than committing the base64 value inline.

**3. Edit the ArgoCD Application in the foundational-software repo**

Open `apps/opentelemetry-collector.yaml` in your foundational-software GitOps repository
and add the `otlphttp/langfuse` exporter under `helm.values.config`, then append it to the
traces pipeline exporters. The resulting `helm.values:` block should look like:

```yaml
helm.values: |
  config:
    exporters:
      debug:
        verbosity: detailed
      otlp:
        endpoint: "localhost:4317"
        tls:
          insecure: true
      otlphttp/langfuse:
        endpoint: "https://<langfuse-host>/api/public/otel"
        headers:
          Authorization: "Basic <base64-public-key:secret-key>"
    service:
      pipelines:
        metrics:
          receivers: [otlp, prometheus]
          processors: [memory_limiter, batch]
          exporters: [debug]
        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug]
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug, otlphttp/langfuse]
```

Replace `<langfuse-host>` with your Langfuse hostname (e.g. `langfuse.nebari.example.com`)
and `<base64-public-key:secret-key>` with the value computed in step 2.

**4. Commit and push**

```bash
git add apps/opentelemetry-collector.yaml
git commit -m "add langfuse otlp exporter to traces pipeline"
git push
```

ArgoCD (`selfHeal: true`, `prune: true`) reconciles automatically and triggers a
`helm upgrade` of the `opentelemetry-collector` release in the `monitoring` namespace.
No manual `kubectl` or `helm` commands are needed.

**5. Verify**

- In Langfuse, open **Tracing** in your project.
- Within a minute or two of the ArgoCD sync completing, traces from workloads that
  instrument with OTLP should appear.
- If no traces appear, check the collector pod logs for errors from the
  `otlphttp/langfuse` exporter:
  ```bash
  kubectl logs -n monitoring -l app.kubernetes.io/name=opentelemetry-collector --tail=50
  ```
