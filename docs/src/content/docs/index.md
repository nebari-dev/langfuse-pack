---
title: Introduction
description: "Langfuse LLM observability: tracing, evals, prompt management, and metrics for debugging and improving LLM apps."
---

The Nebari Langfuse Pack deploys [Langfuse](https://langfuse.com/) on
[Nebari](https://nebari.dev) for LLM observability and tracing, with a
`NebariApp` custom resource for routing, TLS, and gateway authentication on a
Nebari cluster.

:::note[Documentation in progress]
This site is the scaffolding for the pack's documentation. Content is being
written; until it lands here, the
[repository README](https://github.com/nebari-dev/langfuse-pack#readme)
is the reference for installing the pack.
:::

## Reference

- [Configuration reference](/configuration/) — every configuration surface:
  NebariApp values, Langfuse passthrough values, authentication, secrets,
  external datastores, and telemetry.
- [Release readiness checklist](/release-readiness/) — maturity-checklist
  status for the pack.

## Contributing to these docs

Pages live in `docs/src/content/docs/`. See the
[docs README](https://github.com/nebari-dev/langfuse-pack/blob/main/docs/README.md)
for how to run the site locally and add a page.
