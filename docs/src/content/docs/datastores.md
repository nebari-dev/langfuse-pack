---
title: Datastores
description: The four backends Langfuse needs, why ClickHouse is single-node here, and moving to managed services.
---

Langfuse needs four stateful backends. All are bundled by default; all should be external in
production.

| Backend | Holds | Losing it means |
|---|---|---|
| **PostgreSQL** | projects, users, prompts, API keys, config | everything except trace data |
| **ClickHouse** | traces, observations, scores | all observability history |
| **Redis** | ingestion queue, cache | in-flight events; the rest recovers |
| **S3 / MinIO** | large payloads — prompts and completions above the inline limit | trace bodies, while metadata survives |

Two databases rather than one because the workloads are different: PostgreSQL for
transactional metadata, ClickHouse for high-volume append-only analytics over traces.

## ClickHouse is single-node here, deliberately

```yaml
langfuse:
  clickhouse:
    clusterEnabled: false
    replicaCount: 1
    zookeeper:
      enabled: false
```

The upstream chart defaults to a 3-replica cluster with ZooKeeper. This pack overrides that.

Langfuse runs its schema migrations over the **load-balanced ClickHouse Service**, and
golang-migrate's `schema_migrations` bookkeeping is not consistently replicated across the
three nodes. Migrations land on different replicas, get marked *dirty*, and `langfuse-web`
crash-loops.

Single-node avoids that entire class of failure and is far lighter — no ZooKeeper, one pod
instead of four.

:::caution[HA ClickHouse carries the migration caveat]
`clusterEnabled: true`, `replicaCount: 3`, `zookeeper.enabled: true` gives you the upstream
topology and the migration problem with it. An external managed ClickHouse is the better
path to high availability.
:::

## The bundled images

All four use `bitnamilegacy/*` images pinned by upstream chart 1.5.34.

:::caution[These images are frozen and unpatched]
The `bitnamilegacy` mirrors exist because the original tags were removed from Docker Hub.
They receive no ongoing security updates. Fine for development and demos; not a base for a
production data store.
:::

They also pull from Docker Hub, so restricted egress or rate limiting shows up as image-pull
failures:

```bash
kubectl -n langfuse get events --field-selector reason=Failed
```

In a restricted environment, mirror them internally and override the repository, or go
external.

## Going external

`examples/prod-external-datastores.yaml` has a full example. The shape:

```yaml
langfuse:
  postgresql:
    deploy: false
    host: postgres.example.com
    port: 5432
    auth:
      username: langfuse
      database: langfuse
      existingSecret: langfuse-db-credentials
  redis:
    deploy: false
    host: redis.example.com
  clickhouse:
    deploy: false
    host: clickhouse.example.com
  s3:
    deploy: false
    bucket: my-langfuse-bucket
```

`deploy: false` removes the bundled subchart and switches the connection settings to the
host you supply. Each can be moved independently — a common intermediate step is external
PostgreSQL and ClickHouse with bundled Redis and MinIO.

Exact key names belong to the upstream chart; the
[configuration reference](/configuration/#external-datastores) has the full set.

## Sizing

The bundled defaults are development-scale. What actually drives capacity:

- **ClickHouse** grows with trace volume, and trace volume grows with LLM call volume. This
  is the one to plan for; it dominates storage on any active deployment.
- **PostgreSQL** stays small — projects, users, prompts, and keys are rows.
- **Redis** holds the ingestion queue. It grows when the worker falls behind and drains when
  it catches up; sustained growth means the worker is under-provisioned.
- **S3** grows with the fraction of payloads exceeding the inline limit, so it scales with
  prompt and completion size rather than call count.

On a small cluster — fewer than three schedulable nodes — ClickHouse pods commonly sit
`Pending` on CPU or memory:

```bash
kubectl -n langfuse get pods -l app.kubernetes.io/name=clickhouse
kubectl -n langfuse describe pod <clickhouse-pod>
```

## Backups

Nothing here backs anything up.

| Backend | Backup |
|---|---|
| PostgreSQL | `pg_dump`, or your managed service's snapshots |
| ClickHouse | `BACKUP TABLE`, or managed snapshots |
| Redis | none needed — it is a queue |
| S3 / MinIO | bucket replication or lifecycle policy |

:::caution[A database backup is useless without `langfuse-secrets`]
`encryptionKey` decrypts stored integration credentials, and the datastore passwords are the
only copies the bundled instances hold. Back the Secret up alongside the data. See
[Secrets and GitOps](/secrets/#backing-it-up).
:::

On a Nebari cluster with
[longhorn-backup-pack](https://packs.nebari.dev/longhorn-backup-pack/), the bundled
datastores' PVCs are covered by the cluster-wide schedule if they sit on the default
StorageClass. Confirm rather than assume:

```bash
kubectl -n longhorn-system get volumes.longhorn.io \
  -l recurring-job-group.longhorn.io/default=enabled
```

Volume snapshots of a running database are crash-consistent, not application-consistent.
Usually fine; a logical dump before a migration is the stronger guarantee.

## Credentials

All four bundled datastores read from `langfuse-secrets`, and so does Langfuse — one source
for both ends of every connection. That is why rotating a password there means restarting
the datastore *and* both Langfuse workloads. See [Secrets and GitOps](/secrets/#rotation).

External datastores use their own secrets, referenced per-datastore with `existingSecret`.
