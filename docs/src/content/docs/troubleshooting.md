---
title: Troubleshooting
description: The failures this pack actually produces, and the commands that tell them apart.
---

## First look

```bash
kubectl -n langfuse get pods
kubectl -n langfuse get nebariapp,httproute,certificate
kubectl -n langfuse logs deploy/langfuse-web --tail=100
kubectl -n langfuse logs deploy/langfuse-worker --tail=50
```

## A value I set had no effect

Almost always [value nesting](/value-nesting/). App config lives at `langfuse.langfuse.*`,
and Helm silently ignores values at the wrong depth.

```bash
helm -n langfuse get values langfuse
kubectl -n langfuse get deploy langfuse-web -o yaml | grep -A2 NEXTAUTH_URL
```

The second command is the real check — it shows what the pod received.

## NebariApp not Ready — or Ready but unreachable

```bash
kubectl -n langfuse describe nebariapp langfuse
```

| Condition | Reason | Fix |
|---|---|---|
| `Ready` | `NamespaceNotOptedIn` | `kubectl label namespace langfuse nebari.dev/managed=true` |
| `Ready` | `ServiceNotFound` | `langfuse-web` is missing — check the release deployed and `langfuse.fullnameOverride: langfuse` is still set |
| `RoutingReady` | `RoutingNotConfigured` | `nebariapp.routing` was removed; the operator skips routing entirely and the hostname 404s |

Read every condition, not the top-line status. `RoutingNotConfigured` lands on `RoutingReady`
and leaves `Ready` alone, so it presents as a `NebariApp` that looks healthy while the
hostname returns 404.

```bash
kubectl -n langfuse get svc
helm -n langfuse status langfuse
```

## Login fails after Keycloak

The user reaches Keycloak, authenticates, and the redirect back breaks. Two causes.

**`nextauth.url` still points at localhost.** The default is `http://localhost:3000`, and it
is set at the double-nested path, so it is easy to miss:

```bash
kubectl -n langfuse get deploy langfuse-web -o yaml | grep -A2 NEXTAUTH_URL
```

It must be `https://<nebariapp.hostname>`.

**The issuer is unset.** The chart's default is the literal
`https://REPLACE-ME/realms/nebari`. The OIDC secret's `issuer-url` key is populated only when
the operator has `KEYCLOAK_EXTERNAL_URL` set, so on most clusters there is nothing to derive
it from:

```bash
kubectl -n langfuse get deploy langfuse-web -o yaml | grep -A2 AUTH_KEYCLOAK_ISSUER
```

## redirect_uri_mismatch

Only applies with `nebariapp.auth.provisionClient: false`. A manually created Keycloak
client needs this in its Valid Redirect URIs:

```
https://<hostname>/api/auth/callback/keycloak
```

With `provisionClient: true` — the default — the operator registers it automatically.

## No way to log in at all

`disableUsernamePassword: true` is the default, so a broken SSO configuration leaves no
route into the UI. Temporarily:

```yaml
langfuse:
  langfuse:
    auth:
      disableUsernamePassword: false
```

Then fix the issuer or `nextauth.url` and turn it back off.

## langfuse-web crashloops: ENCRYPTION_KEY length

The key must be exactly 64 hex characters:

```bash
kubectl -n langfuse get secret langfuse-secrets \
  -o jsonpath='{.data.encryptionKey}' | base64 -d | tr -d '\n' | wc -c
# 64
```

Anything else means the Secret is missing or malformed — most often a hand-created Secret
built with `openssl rand -base64 32` (44 characters) instead of `-hex 32`. See
[Secrets and GitOps](/secrets/).

## Datastore authentication failures after an Argo CD sync

The signature symptom of generated secrets under GitOps: PostgreSQL, Redis, or ClickHouse
rejecting Langfuse's credentials, with each component appearing to blame a different
neighbour.

```bash
kubectl -n langfuse get secret langfuse-secrets -o jsonpath='{.data.salt}' | sha256sum
# sync, then run again — the digest must not change
```

A changed digest means `secrets.generate` is still `true`. Set it to `false` and pre-create
the Secret — [Secrets and GitOps](/secrets/#the-gitops-procedure).

## ClickHouse pods Pending

ClickHouse asks for real CPU and memory. On small clusters:

```bash
kubectl -n langfuse get pods -l app.kubernetes.io/name=clickhouse
kubectl -n langfuse describe pod <clickhouse-pod> | tail -20
```

`Insufficient cpu`/`memory` means the node cannot fit it. Lower the requests for
development, or use an external ClickHouse.

## langfuse-web crashloops after enabling clustered ClickHouse

The known failure mode this pack's defaults avoid. Langfuse runs migrations over the
load-balanced ClickHouse Service, and golang-migrate's `schema_migrations` bookkeeping is
not consistently replicated across replicas — migrations land on different nodes, get marked
*dirty*, and the web pod crash-loops.

Go back to `clusterEnabled: false`, `replicaCount: 1`, `zookeeper.enabled: false`, or use an
external managed ClickHouse. See
[Datastores](/datastores/#clickhouse-is-single-node-here-deliberately).

## Image pull failures

The bundled datastores pull `bitnamilegacy/*` from Docker Hub:

```bash
kubectl -n langfuse get events --field-selector reason=Failed
kubectl -n langfuse describe pod <pod-name>
```

Restricted egress or rate limiting. Mirror the images internally and override the
repository, or switch to external datastores.

## Everything is slow to start

Expected on first install. Four datastores come up, then Langfuse runs migrations against
both PostgreSQL and ClickHouse before `langfuse-web` becomes ready. The dev Makefile allows
ten minutes; the e2e script allows twelve.

```bash
kubectl -n langfuse logs deploy/langfuse-web -f
```

Migration progress is in that log. A restart during the window usually means it is waiting
on a datastore, not failing.

## PodMonitor produces scrape errors

Langfuse exposes no native Prometheus `/metrics` endpoint. `metrics.podMonitor.enabled` is a
hook for a future sidecar exporter, and enabling it without one produces exactly these
errors. Leave it `false` unless you have added an exporter.

## Gathering state for an issue

```bash
kubectl -n langfuse get all
kubectl -n langfuse describe nebariapp langfuse
kubectl -n langfuse logs deploy/langfuse-web --tail=200
kubectl -n langfuse logs deploy/langfuse-worker --tail=100
kubectl -n langfuse get events --sort-by=.lastTimestamp | tail -30
helm -n langfuse get values langfuse
```

Check the values output before sharing — it can contain hostnames and issuer URLs.
