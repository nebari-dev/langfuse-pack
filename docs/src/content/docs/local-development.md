---
title: Local development
description: The kind stack in dev/, and the black-box e2e suite.
---

## Prerequisites

docker, kind, helm, kubectl, git.

## Two stacks

```bash
cd dev

make up             # kind + MetalLB + Envoy Gateway + cert-manager + Keycloak
                    # + nebari-operator + Langfuse with SSO
make up-standalone  # kind + Langfuse only, port-forward access
make down           # delete the cluster
```

`make up` deploys at `https://langfuse.nebari.local` by default (`HOSTNAME`), into cluster
`nebari-langfuse-dev` (`CLUSTER_NAME`), release `langfuse` (`RELEASE`).

:::note[This takes several minutes]
Langfuse pulls in ClickHouse, PostgreSQL, Valkey, and MinIO, then runs schema migrations on
first start. The Helm timeout is set to 10 minutes for a reason — a slow first run is normal,
not a hang.
:::

`make update-hosts` adds the NebariApp hostnames to `/etc/hosts` (via the operator's script;
it uses sudo). `make down` deletes the cluster.

## Watching it come up

```bash
kubectl get pods -w
kubectl logs -l app=web -f
kubectl get nebariapp,httproute,certificate
```

Ordering to expect: datastores first, then `langfuse-web` running migrations, then ready.
A `langfuse-web` restarting during that window is usually waiting on ClickHouse rather than
failing.

## Iterating on the chart

```bash
make up    # re-runs the helm install against the existing cluster
```

For a values change with no cluster at all:

```bash
helm template langfuse chart -f my-values.yaml \
  | grep -E "NEXTAUTH_URL|AUTH_KEYCLOAK_ISSUER" -A2
```

That is the fastest way to catch a [nesting mistake](/value-nesting/) — placeholders in the
output (`localhost:3000`, `REPLACE-ME`) mean the value did not land.

## Chart tests

The repository has helm-unittest suites and two shell checks:

```bash
helm dependency update chart/
helm plugin install https://github.com/helm-unittest/helm-unittest
helm unittest chart/                              # nebariapp, podmonitor, secrets

bash chart/tests/values-wiring.sh                 # cross-value wiring assertions
bash chart/tests/check-pack-metadata.sh pack-metadata.yaml
```

The unit tests cover exactly the templates most likely to break quietly — the `NebariApp`,
the generated Secret, and the opt-in PodMonitor. CI adds `kubeconform` over the rendered
manifests, and two render assertions worth knowing about: that the prod example really does
disable every bundled datastore, and that the bundled images are `bitnamilegacy` rather than
paid Bitnami or a floating `:latest`.

## End-to-end tests

`tests/e2e/` holds HTTP-only black-box tests that run against **any** deployed instance.
`run.sh` needs nothing but `curl`:

```bash
BASE_URL=https://langfuse.mycluster MODE=sso        ./tests/e2e/run.sh
BASE_URL=https://langfuse.mycluster MODE=standalone ./tests/e2e/run.sh
```

`MODE` selects the auth expectations — SSO-only versus email/password.

To run the whole thing locally against a throwaway cluster:

```bash
./tests/e2e/kind-e2e.sh
```

It creates a kind cluster named `nebari-langfuse-e2e`, installs the chart with
`tests/e2e/ci-values.yaml` (standalone, no operator), waits up to 12 minutes for
`helm install --wait`, port-forwards `svc/langfuse-web:3000`, polls the health endpoint,
runs `run.sh` in standalone mode, and tears the cluster down on exit — including on failure.

`tests/e2e/README.md` has the full detail.

## What local development cannot tell you

- **The Argo CD secret rotation.** It only appears under Argo CD; the dev stack uses Helm,
  where `lookup` works. Read [Secrets and GitOps](/secrets/) before deploying via GitOps.
- **Clustered ClickHouse migrations.** The dev stack runs the single-node default, which is
  precisely the configuration that avoids the problem.
- **Real TLS and a real Keycloak realm.** Certificates are local, and the dev realm's group
  and role names will not match yours.

## Docs site

```bash
cd docs
npm ci
npm run dev     # hot reload at http://localhost:4321
npm run build   # static build into docs/dist/
npm test        # unit tests
```

Pages live in `docs/src/content/docs/`; the sidebar is in `docs/astro.config.mjs`. Merges to
`main` publish to [packs.nebari.dev/langfuse-pack/](https://packs.nebari.dev/langfuse-pack/),
and pull requests touching `docs/` get a preview URL posted as a comment.
