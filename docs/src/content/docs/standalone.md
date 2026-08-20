---
title: Standalone deployment
description: Running Langfuse without Nebari — email/password auth and bundled datastores.
---

With `nebariapp.enabled: false` — the chart's default — no `NebariApp` is rendered, so no
operator, gateway, cert-manager, or Keycloak is needed. Just Kubernetes and Helm 3.

## Install

```bash
helm repo add nebari https://nebari-dev.github.io/helm-repository
helm repo update

helm install langfuse nebari/nebari-langfuse \
  -f examples/standalone-values.yaml
```

[`examples/standalone-values.yaml`](https://github.com/nebari-dev/langfuse-pack/blob/main/examples/standalone-values.yaml)
is short:

```yaml
nebariapp:
  enabled: false

langfuse:
  langfuse:
    nextauth:
      url: http://localhost:3000
    auth:
      disableUsernamePassword: false   # allow email/password locally
      providers: {}                    # no SSO
```

Two things are flipped from the Nebari defaults: username/password login is re-enabled
(there is no Keycloak), and the providers map is emptied so NextAuth does not look for one.

## Access

```bash
kubectl port-forward svc/langfuse-web 3000:3000
```

Open `http://localhost:3000` and sign up with an email and password. The first account is
created through the normal sign-up flow.

:::caution[`nextauth.url` must match how you actually reach it]
NextAuth builds its callback URLs from this value. Port-forwarding to a different local
port, or exposing Langfuse through an Ingress, means changing it to match — otherwise login
redirects somewhere that does not exist.

```yaml
langfuse:
  langfuse:
    nextauth:
      url: https://langfuse.internal.example.com
```
:::

## Beyond port-forward

The upstream chart has an `ingress` block for the web service
(`langfuse.langfuse.ingress.*` — see [Value nesting](/value-nesting/)), or switch the
service type. Either way, update `nextauth.url` to the address users will actually type,
and put TLS in front of it: sessions and API keys travel over this connection.

## What standalone gives up

- **No SSO.** Accounts are local to Langfuse, with no central deprovisioning.
- **No TLS.** Terminate it yourself.
- **Bundled datastores.** Frozen `bitnamilegacy` images, fine for evaluation. See
  [Datastores](/datastores/).
- **Still four datastores.** Standalone does not mean lightweight — PostgreSQL, ClickHouse,
  Redis, and MinIO all still run, and ClickHouse in particular needs real CPU and memory.

## Generated secrets

`secrets.generate` defaults to `true`, and standalone installs are normally plain Helm, so
the lookup-based generation works as intended: values are created once and preserved across
upgrades.

If you later move this deployment under Argo CD, that changes — see
[Secrets and GitOps](/secrets/).

## Sending traces

Nothing about the client integration is Nebari-specific. Create a project in the UI,
generate API keys, and point the SDK at the port-forwarded address:

```python
from langfuse import Langfuse

langfuse = Langfuse(
    public_key="pk-lf-...",
    secret_key="sk-lf-...",
    host="http://localhost:3000",
)
```

From inside the cluster, use `http://langfuse-web.<namespace>.svc.cluster.local:3000`.

## Testing it end to end

The repository has a black-box e2e suite that works against any deployment:

```bash
BASE_URL=http://localhost:3000 MODE=standalone ./tests/e2e/run.sh
```

`run.sh` needs only `curl`. To build a throwaway cluster and run the whole thing:

```bash
./tests/e2e/kind-e2e.sh
```

See [Local development](/local-development/).
