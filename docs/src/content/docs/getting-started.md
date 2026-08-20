---
title: Getting started
description: Install the Langfuse pack on a Nebari cluster.
---

## Prerequisites

- A Nebari cluster with **nebari-operator v0.1.0-alpha.19 or later**
- cert-manager, able to issue certificates for your domain
- Envoy Gateway (the `NebariApp` routes through it)
- Keycloak, reachable — the operator provisions an OIDC client
- Helm 3
- The target namespace opted in:
  ```bash
  kubectl label namespace langfuse nebari.dev/managed=true
  ```

Enough cluster capacity for four datastores. ClickHouse in particular is not small — see
[Datastores](/datastores/).

Without a Nebari platform, go to [Standalone deployment](/standalone/).

## Install

```bash
helm repo add nebari https://nebari-dev.github.io/helm-repository
helm repo update

helm install langfuse nebari/nebari-langfuse \
  -f examples/nebari-values.yaml \
  --set nebariapp.hostname=langfuse.example.com \
  --set langfuse.langfuse.nextauth.url=https://langfuse.example.com \
  --set langfuse.langfuse.auth.providers.keycloak.issuer=https://keycloak.example.com/realms/nebari \
  --namespace langfuse
```

The upstream `langfuse/langfuse` chart comes in as a transitive dependency — only the Nebari
repo needs adding.

:::note[Use the release name `langfuse`]
The pack pins `langfuse.fullnameOverride: langfuse` so the web Service is always
`langfuse-web`, which is what `nebariapp.service.name` targets. A different release name
still works, but keep that pin in place.
:::

## The three values you must supply

None of these have usable defaults, and two of them fail in ways that do not point at
themselves.

| Value | Must be |
|---|---|
| `nebariapp.hostname` | your FQDN |
| `langfuse.langfuse.nextauth.url` | `https://<same hostname>` |
| `langfuse.langfuse.auth.providers.keycloak.issuer` | `https://<keycloak>/realms/<realm>` |

**`nextauth.url`** defaults to `http://localhost:3000`. Leave it and NextAuth builds
callback URLs pointing at localhost — the login round-trip fails after Keycloak, not
before, which makes it look like a Keycloak problem.

**`issuer`** defaults to the literal `https://REPLACE-ME/realms/nebari`. Langfuse requires
`AUTH_KEYCLOAK_ISSUER`, and the operator does not reliably write the issuer URL into the
OIDC secret, so it cannot be read automatically. Sub-chart values cannot be templated from
the parent, which is why the chart cannot derive it from `nebariapp.hostname` either.

`clientId` and `clientSecret` *are* automatic — they come from the `langfuse-oidc-client`
Secret the operator creates.

## What gets deployed

| Workload | Purpose |
|---|---|
| `langfuse-web` | UI and API, port `3000` |
| `langfuse-worker` | Async ingestion and processing |
| `langfuse-postgresql` | Metadata |
| `langfuse-clickhouse` | Traces, observations, scores |
| `langfuse-redis` | Queue and cache |
| `langfuse-minio` | Large payload storage |
| `langfuse-secrets` | Generated credentials |
| `langfuse` | `NebariApp` |

## Verify

```bash
kubectl -n langfuse get pods
kubectl -n langfuse get nebariapp,httproute,certificate

# Langfuse's own health endpoint — note the path
kubectl -n langfuse port-forward svc/langfuse-web 3000:3000 &
curl -sf http://localhost:3000/api/public/health && echo OK
kill %1
```

The health path is `/api/public/health`, not `/health`. The landing-page tile's health check
is configured for it; anything else you point at Langfuse should be too.

First start takes a while — Langfuse runs schema migrations against PostgreSQL and
ClickHouse before `langfuse-web` becomes ready.

## First login

Open `https://langfuse.example.com`. Keycloak takes the login, and Langfuse creates the
account on first sign-in.

`disableUsernamePassword: true` is the default here, so the email/password form is gone and
SSO is the only way in.

:::caution[Get SSO working before you disable the fallback]
With username/password disabled and a misconfigured issuer, there is no way into the UI at
all. If you are still wiring Keycloak, set `disableUsernamePassword: false` temporarily.
:::

## Next

- Understand the values layout before editing anything: [Value nesting](/value-nesting/)
- **Before** deploying through Argo CD: [Secrets and GitOps](/secrets/)
- Plan the production datastores: [Datastores](/datastores/)
- Every value in one place: [Configuration reference](/configuration/)
