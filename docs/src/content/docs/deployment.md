---
title: Deploying on Nebari
description: GitOps deployment of the Langfuse pack with Argo CD, and the Secret you must create first.
---

:::caution[Create `langfuse-secrets` before the first sync]
Argo CD renders charts with `helm template`, which cannot do cluster lookups. Left at
`secrets.generate: true`, the chart writes **new random credentials on every sync**,
rotating the passwords out from under running PostgreSQL, Redis, ClickHouse, and MinIO.

The procedure is on [Secrets and GitOps](/secrets/#the-gitops-procedure). Do that first;
everything below assumes it.
:::

## Preparation

```bash
kubectl create namespace langfuse
kubectl label namespace langfuse nebari.dev/managed=true

kubectl -n langfuse create secret generic langfuse-secrets \
  --from-literal=salt=$(openssl rand -hex 16) \
  --from-literal=encryptionKey=$(openssl rand -hex 32) \
  --from-literal=nextauth-secret=$(openssl rand -hex 32) \
  --from-literal=postgres-password=$(openssl rand -hex 16) \
  --from-literal=redis-password=$(openssl rand -hex 16) \
  --from-literal=clickhouse-password=$(openssl rand -hex 16) \
  --from-literal=root-user=langfuse \
  --from-literal=root-password=$(openssl rand -hex 16)
```

Hex values only — these end up in datastore connection strings, where `/`, `+`, and `@`
need URL-encoding and produce parse errors that look like authentication failures.

## The Application

From
[`examples/argocd-app.yaml`](https://github.com/nebari-dev/langfuse-pack/blob/main/examples/argocd-app.yaml):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: langfuse
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://nebari-dev.github.io/helm-repository
    chart: nebari-langfuse
    targetRevision: 0.1.0
    helm:
      releaseName: langfuse
      valuesObject:
        nebariapp:
          enabled: true
          hostname: langfuse.example.com
          auth:
            enabled: true
          landingPage:
            enabled: true
        secrets:
          generate: false
        langfuse:
          langfuse:
            nextauth:
              url: https://langfuse.example.com     # must equal nebariapp.hostname
            auth:
              providers:
                keycloak:
                  issuer: "https://keycloak.example.com/realms/nebari"
  destination:
    server: https://kubernetes.default.svc
    namespace: langfuse
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    managedNamespaceMetadata:
      labels:
        nebari.dev/managed: "true"
```

Three values to change: the hostname (twice — `nebariapp.hostname` and
`nextauth.url` must match) and the Keycloak issuer.

Note `langfuse.langfuse` in that values block. It is not a typo — see
[Value nesting](/value-nesting/).

## What the parts do

**`releaseName: langfuse`** keeps the web Service named `langfuse-web`, which is what
`nebariapp.service.name` targets. Combined with the chart's
`langfuse.fullnameOverride: langfuse` pin, the name is stable.

**`managedNamespaceMetadata`** applies `nebari.dev/managed: "true"`. The operator ignores
`NebariApp` resources in unlabeled namespaces — silently, with no event. You already applied
the label by hand above; this keeps it if the namespace is ever recreated.

**`valuesObject`** rather than a `values` string means Argo CD parses the YAML, so a
structural error is caught at sync rather than becoming a silently-ignored value at the
wrong depth.

**`targetRevision: 0.1.0`** should stay pinned. A floating version would upgrade Langfuse
and run its schema migrations unannounced.

**`selfHeal: true`** reverts manual edits. Change values in git.

## Routing must stay enabled

The chart defaults `nebariapp.routing.enabled: true`. Do not remove the block.

Without `spec.routing`, the operator skips routing entirely — the `NebariApp` reports
`RoutingNotConfigured`, no HTTPRoute or certificate is created, and the hostname returns
404 while everything else looks healthy.

## Verifying a sync

```bash
kubectl -n argocd get application langfuse
kubectl -n langfuse get nebariapp,httproute,certificate
kubectl get namespace langfuse -o jsonpath='{.metadata.labels}'

# The Secret must be stable across syncs
kubectl -n langfuse get secret langfuse-secrets -o jsonpath='{.data.salt}' | sha256sum
```

Run that last command before and after a sync. A changed digest means `secrets.generate` is
still `true` and the credentials are rotating.

Then the app itself:

```bash
kubectl -n langfuse get pods
kubectl -n langfuse logs deploy/langfuse-web --tail=50
```

First sync takes several minutes — four datastores start, then Langfuse runs schema
migrations against PostgreSQL and ClickHouse before `langfuse-web` becomes ready.

## Argo CD may flag the Secret

The pre-created `langfuse-secrets` is not managed by the chart, so Argo CD can report it as
out of sync. Add an `ignoreDifferences` entry for the `Secret` resource if the noise
matters.

## Upgrading

Bump `targetRevision` and commit.

:::caution[Migrations are one-way]
A chart upgrade that bumps Langfuse runs schema migrations against both PostgreSQL and
ClickHouse. Rolling `targetRevision` back does not roll the schema back. Snapshot both
databases before a significant version bump — and read
[Datastores](/datastores/#clickhouse-is-single-node-here-deliberately) if you are running
clustered ClickHouse, where migrations are the known failure mode.
:::

After any values refactor, confirm `secrets.generate` is still `false`. Reintroducing `true`
rotates the credentials on the next sync.
