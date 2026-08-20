---
title: Secrets and GitOps
description: What the chart generates, what rotation costs, and why Argo CD needs a pre-created Secret.
---

## What is generated

With `secrets.generate: true` (the default), the chart creates a Secret named
`langfuse-secrets`:

| Key | Purpose |
|---|---|
| `salt` | Langfuse password hashing salt |
| `encryptionKey` | 64 hex characters — encrypts stored integration credentials |
| `nextauth-secret` | NextAuth session signing key |
| `postgres-password` | PostgreSQL |
| `redis-password` | Redis |
| `clickhouse-password` | ClickHouse |
| `root-user` | MinIO root user (`langfuse`) |
| `root-password` | MinIO root password |

The template reads any existing Secret with Helm's `lookup` and reuses each key it finds,
generating only what is missing. Combined with `helm.sh/resource-policy: keep`, values
survive `helm upgrade` and `helm uninstall` without rotating.

Both the Langfuse app and the four bundled datastore subcharts read from this one Secret, so
the credentials on both sides of every connection always agree.

## The Argo CD problem

:::caution[`secrets.generate: true` is incompatible with GitOps]
Argo CD's repo-server renders charts with `helm template`, which has no cluster access.
`lookup` returns nothing every time, so the chart generates **new random values on every
sync** — rotating the datastore passwords out from under running PostgreSQL, Redis,
ClickHouse, and MinIO instances that still hold the old ones.

The result is authentication failures across every datastore, with each component blaming a
different neighbour.
:::

## The GitOps procedure

**1. Namespace and label:**

```bash
kubectl create namespace langfuse
kubectl label namespace langfuse nebari.dev/managed=true
```

**2. Create the Secret once, before the first sync:**

```bash
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

:::note[Use hex, not base64 or random ASCII]
These passwords end up inside datastore connection strings. Characters like `/`, `+`, and
`@` need URL-encoding and produce parse failures that surface as authentication errors.
`openssl rand -hex` avoids the whole category.

`encryptionKey` must be exactly 64 hex characters — `openssl rand -hex 32` gives that.
:::

**3. Turn generation off:**

```yaml
secrets:
  generate: false
```

**4. Optionally, stop Argo CD flagging the pre-created Secret** as out of sync with an
`ignoreDifferences` entry for the `Secret` resource.

A production Helm install should do the same. Generation is a convenience for development,
not a secret-management strategy — for anything real, use sealed-secrets, external-secrets,
or your cloud's secret manager.

## Rotation

:::caution[Never rotate `encryptionKey` casually]
It encrypts every integration credential Langfuse has stored. Change it and those
credentials are unrecoverable — every integration must be re-entered by hand. There is no
migration path.
:::

The others are safer but not free:

| Key | Cost of rotating |
|---|---|
| `nextauth-secret` | Every active session is invalidated; users log in again. |
| `postgres-password`, `redis-password`, `clickhouse-password` | Must be changed in the datastore *and* the Secret, then both sides restarted. |
| `root-password` | Same, for MinIO. |
| `salt` | Affects password hashing; irrelevant with SSO-only auth. |
| `encryptionKey` | **Orphans all stored integration credentials.** |

Rotating a bundled datastore password means restarting the datastore and both Langfuse
workloads:

```bash
kubectl -n langfuse rollout restart statefulset/langfuse-postgresql
kubectl -n langfuse rollout restart deployment/langfuse-web deployment/langfuse-worker
```

## Verifying

```bash
kubectl -n langfuse get secret langfuse-secrets -o jsonpath='{.data}' | jq 'keys'
```

All eight keys should be present. The one with a hard length requirement:

```bash
kubectl -n langfuse get secret langfuse-secrets \
  -o jsonpath='{.data.encryptionKey}' | base64 -d | tr -d '\n' | wc -c
# 64
```

Anything other than 64 crash-loops `langfuse-web` with an `ENCRYPTION_KEY` length error.

Under Argo CD, confirm the Secret is stable across a sync:

```bash
kubectl -n langfuse get secret langfuse-secrets -o jsonpath='{.data.salt}' | sha256sum
# sync, then run again — the digest must not change
```

## Backing it up

The Secret is not derivable from anything else, and without it a restored database is
unreadable: `encryptionKey` decrypts the integration credentials, and the datastore
passwords are the only copies the bundled instances have.

```bash
kubectl -n langfuse get secret langfuse-secrets -o yaml > langfuse-secrets-backup.yaml
```

Store it wherever your other credentials live, not next to the database dump.
