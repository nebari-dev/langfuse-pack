---
title: Introduction
description: "Langfuse LLM observability: tracing, evals, prompt management, and metrics for debugging and improving LLM apps."
---

The Nebari Langfuse Pack deploys [Langfuse](https://langfuse.com/) — open-source LLM
observability, tracing, evaluation, and prompt management — on a
[Nebari](https://nebari.dev) cluster with Keycloak SSO, TLS, and routing handled by a
`NebariApp`.

Wraps the upstream `langfuse/langfuse` chart **1.5.34** (Langfuse **3.179.1**). Declared
maturity: **Alpha** (`level: alpha` in `pack-metadata.yaml`).

```
  browser ──► Envoy Gateway ──► langfuse-web :3000 ──┬─► PostgreSQL   metadata
                (routing            │                ├─► ClickHouse   traces & events
                 + TLS)             │                ├─► Redis        queue & cache
                              NextAuth OAuth         └─► S3/MinIO     large payloads
                                    │
                                Keycloak            langfuse-worker  async ingestion
```

Four datastores, which is the main thing to know before installing: Langfuse is not a
single-container application. All four are bundled by default for development, and all four
should be external in production. See [Datastores](/datastores/).

Authentication is app-native. Langfuse runs the OAuth flow itself through NextAuth, so
`enforceAtGateway` is `false` and the gateway only routes.

## Four things that will bite you

Each has a section below; they are collected here because all four are silent or
confusingly-reported failures.

- **Values are double-nested.** The dependency is named `langfuse` *and* the upstream chart
  has a top-level `langfuse` key, so app config lives at `langfuse.langfuse.*`. See
  [Value nesting](/value-nesting/).
- **Generated secrets do not work under Argo CD.** `helm template` cannot do cluster
  lookups, so every sync writes new random values and breaks datastore auth. Pre-create the
  Secret. See [Secrets and GitOps](/secrets/).
- **The Keycloak issuer must be set by hand.** The operator fills the OIDC secret's
  `issuer-url` only when `KEYCLOAK_EXTERNAL_URL` is set, and the chart's default is a literal
  `REPLACE-ME`. See [Getting started](/getting-started/).
- **`nebariapp.routing` must be present.** Omit it and the operator skips routing entirely —
  `RoutingNotConfigured`, and the hostname returns 404. The chart enables it by default;
  do not remove it.

## In this guide

- **[Getting started](/getting-started/)** — install on Nebari, with the three values you
  must supply
- **[Deploying on Nebari](/deployment/)** — GitOps with Argo CD, and the Secret you must create first
- **[Standalone deployment](/standalone/)** — no Nebari, email/password auth
- **[Local development](/local-development/)** — kind stack and the e2e suite

## Guides

- **[Value nesting](/value-nesting/)** — why `langfuse.langfuse.*`, and how to tell which
  depth a value belongs at
- **[Datastores](/datastores/)** — the four backends, and why ClickHouse is single-node here
- **[Secrets and GitOps](/secrets/)** — what is generated, what rotation costs, and the
  Argo CD path
- **[Troubleshooting](/troubleshooting/)** — the failures this pack actually produces

## Reference

- **[Configuration reference](/configuration/)** — every value: NebariApp, Langfuse
  passthrough, auth wiring, secrets, external datastores, telemetry, and OTel Collector
  export
- **[Release readiness](/release-readiness/)** — maturity-checklist status
