---
title: Value nesting
description: Why app config lives at langfuse.langfuse.* and how to tell which depth a value belongs at.
---

The chart dependency is named `langfuse`, **and** the upstream Langfuse chart uses a
top-level `langfuse` key for its own application config. Helm nests subchart values under the
dependency name, so from this wrapper the application config sits two levels down.

This is the single most common source of values that appear to be ignored.

## The rule

| What you are setting | Path from this wrapper |
|---|---|
| App config — auth, nextauth, secrets, security contexts | `langfuse.langfuse.*` |
| PostgreSQL subchart | `langfuse.postgresql.*` |
| Redis subchart | `langfuse.redis.*` |
| ClickHouse subchart | `langfuse.clickhouse.*` |
| S3 / MinIO subchart | `langfuse.s3.*` |
| Upstream chart-level keys | `langfuse.fullnameOverride`, `langfuse.global` |
| This wrapper's own values | `nebariapp.*`, `secrets.*`, `metrics.*`, `nameOverride` |

```yaml
langfuse:
  fullnameOverride: langfuse    # chart-level

  langfuse:                     # double-nested: app config
    nextauth:
      url: https://langfuse.example.com
    auth:
      providers:
        keycloak:
          issuer: https://keycloak.example.com/realms/nebari

  postgresql:                   # single-nested: subchart
    deploy: true
```

## Telling the depth from upstream docs

The Langfuse chart's own documentation is written from *its* root. Translate by prefixing
`langfuse.`:

| Upstream path | Here |
|---|---|
| `langfuse.nextauth.url` | `langfuse.langfuse.nextauth.url` |
| `postgresql.deploy` | `langfuse.postgresql.deploy` |
| `clickhouse.replicaCount` | `langfuse.clickhouse.replicaCount` |
| `fullnameOverride` | `langfuse.fullnameOverride` |

So a doubled `langfuse.langfuse` is correct precisely when the upstream path already starts
with `langfuse.`.

## Why it silently fails

Helm does not validate value paths. A value at the wrong depth is accepted, merged into the
values tree, and ignored — no warning, no error. The symptom is a default taking effect
where you set something else.

```bash
# What was actually passed
helm -n langfuse get values langfuse

# What was actually rendered into the pod
kubectl -n langfuse get deploy langfuse-web -o yaml | grep -A2 NEXTAUTH_URL
```

The second command is the real check. If the environment variable does not match what you
set, the depth is wrong.

## The one that hurts most

```yaml
# WRONG — silently ignored
langfuse:
  nextauth:
    url: https://langfuse.example.com

# RIGHT
langfuse:
  langfuse:
    nextauth:
      url: https://langfuse.example.com
```

At the wrong depth, `nextauth.url` keeps its default of `http://localhost:3000`. Langfuse
starts fine and the UI loads; the failure appears only when a user returns from Keycloak and
NextAuth redirects them to localhost. It reads as a Keycloak problem and is not.

## `--set` on the command line

Same paths, dot-separated:

```bash
--set langfuse.langfuse.nextauth.url=https://langfuse.example.com
--set langfuse.langfuse.auth.providers.keycloak.issuer=https://keycloak.example.com/realms/nebari
--set langfuse.postgresql.deploy=false
```

Long, and correct. A `--set langfuse.nextauth.url=...` that seems shorter is simply wrong.

## Why the chart cannot smooth this over

Two constraints:

- **The double nesting is structural.** Renaming the dependency alias would break
  compatibility with upstream documentation and with existing values files.
- **Subchart values cannot be templated.** Helm renders subchart values as data, so the
  wrapper cannot derive `langfuse.langfuse.nextauth.url` from `nebariapp.hostname`. That is
  also why the Keycloak `issuer` has to be supplied by hand.

Both are noted in `values.yaml` at the point where they matter.

## Checking a values file

Render before applying:

```bash
helm template langfuse chart -f my-values.yaml \
  | grep -E "NEXTAUTH_URL|AUTH_KEYCLOAK_ISSUER" -A2
```

Placeholders in the output — `localhost:3000`, `REPLACE-ME` — mean the value did not land.
