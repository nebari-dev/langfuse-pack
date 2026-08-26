---
title: Release readiness checklist
description: Status of every Experimental, Alpha, and Beta item from the Nebari software pack maturity checklist as applied to the nebari-langfuse pack.
---

This document records the status of every `[E]` (Experimental), `[A]` (Alpha), and `[B]`
(Beta) item from the Nebari software pack maturity checklist
(`/tmp/nebari-software-pack-template/docs/release-readiness-checklist.md`) as applied to
the `nebari-langfuse` pack.

`[GA]` items are **out of scope** for this promotion and are not listed.

Items marked "covered by Task N" are satisfied by a later task in the implementation plan
and have not yet been verified at the time this document was authored (2026-06-10). They
will be confirmed at the Task 13 journey-verification gate before a formal promotion PR
is opened.

Items marked "EXTERNAL / human" cannot be completed inside this repo. They are listed
with justification and must be resolved as follow-up actions before or at promotion time.

---

## Ownership and Identity

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| Repo is created from the software pack template | `[E]` | N/A - effectively satisfied | The pack was not forked from a template repo (none existed at authoring time) but mirrors the template structure exactly: `chart/`, `examples/`, `tests/`, `docs/`, `dev/`, `.github/workflows/`, `CODEOWNERS`, `pack-metadata.yaml`, `LICENSE`. All template-required files are present. |
| `CODEOWNERS` names at least one accountable engineer | `[E]` | done | `CODEOWNERS` at repo root: `* @dcmcand` (this commit). |
| `pack-metadata.yaml` exists, validates against schema, declares level + owner + scope flags | `[E]` | done | `pack-metadata.yaml` at repo root. `level: alpha`, `owner: dcmcand`, `scope.standalone-supported: yes`. Validated with `check-jsonschema` against the published dashboard schema (2026-06-12). The schema repo is private, so unauthenticated CI falls back to a required-keys check kept in sync with the schema. |
| Pack is listed in `nebari-dev/software-pack-dashboard/tracked-packs.yaml` | `[E]` | done | Added via software-pack-dashboard#2 (merged 2026-06-12). |
| README explains what the pack does and who it is for | `[E]` | covered by Task 9 | `README.md` authored in Task 9. |
| `product_owner` field is populated in `pack-metadata.yaml` | `[B]` | done | `product_owner: dcmcand` in `pack-metadata.yaml` (this commit). |

---

## Installation

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| Installs cleanly on a fresh current-release NIC dev cluster following only the README instructions | `[A]` | covered by Task 13 | Journey 1 verification gate (live kind cluster). |
| Prerequisites documented (NIC version, cluster sizing, namespace labels, external dependencies) | `[A]` | covered by Task 9 | `README.md` Prerequisites section. |
| `helm lint` passes in CI | `[B]` | covered by Task 11 | `.github/workflows/lint.yaml` and `.github/workflows/test.yaml`. |
| `helm template` renders correctly with NebariApp enabled and disabled | `[B]` | covered by Task 11 | CI renders both paths; also exercised in Task 12 pre-cluster gate. |
| Schema validation passes (kubeconform or equivalent) in CI | `[B]` | covered by Task 11 | `test.yaml` workflow includes kubeconform against NebariApp CRD schema. |

---

## NebariApp Integration

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| `NebariApp` reaches Ready condition with all applicable sub-conditions healthy (RoutingReady, TLSReady, AuthReady) | `[A]` | covered by Task 13 | Journey 2 verification gate. |
| All configurable NebariApp fields used by the pack are documented in the values reference | `[A]` | covered by Task 9 | `docs/configuration.md` values reference section. |
| `nebariapp_integration` field in `pack-metadata.yaml` accurately reflects integration depth | `[A]` | done | `nebariapp_integration: full` - the pack provisions a Keycloak client and runs full app-native OAuth through the NebariApp CRD. |
| Auth-protected routes reject unauthenticated requests (auth enabled) | `[B]` | covered by Task 16 | Journey 11 - E2E black-box test (`tests/e2e/run.sh` assertion 3: `POST /api/public/otel/v1/traces` with no credentials returns 401). |
| Auth-protected routes allow authenticated users with correct group membership (auth enabled) | `[B]` | covered by Task 13 | Journey 3 verification gate (browser SSO walkthrough). |
| Health/readiness probes configured and verified | `[B]` | upstream chart + covered by Task 16 | Upstream `langfuse/langfuse` chart sets liveness probe `GET /api/public/health` and readiness probe `GET /api/public/ready` on the web container. E2E `run.sh` asserts health endpoint returns 200. |

---

## Documentation

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| README exists with: what the pack does, who it is for, deploy command | `[E]` | covered by Task 9 | `README.md`. |
| README includes prerequisites and a "Known Limitations" section | `[A]` | covered by Task 9 | `README.md` Prerequisites and Known Limitations sections. |
| Authentication setup is documented | `[B]` | covered by Task 9 | `docs/configuration.md` SSO wiring section; `README.md` auth model summary. |
| Troubleshooting section covers common failure modes | `[B]` | covered by Task 9 | `README.md` Troubleshooting section (NebariApp not Ready, image pull errors, login redirect mismatch, ENCRYPTION_KEY crash, ClickHouse resources). |
| Upstream chart values that users need to customize are documented | `[B]` | covered by Task 9 | `docs/configuration.md` values reference covers `langfuse.langfuse.*`, `langfuse.postgresql.*`, `langfuse.clickhouse.*`, `langfuse.redis.*`, `langfuse.s3.*`. |

---

## Examples

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| At least one example values file that deploys without modification (other than hostname) | `[A]` | done | `examples/nebari-values.yaml` (Task 7). |
| Example values file for full Nebari deployment (`nebari-values.yaml`) | `[B]` | done | `examples/nebari-values.yaml` (Task 7). |
| Example values file for standalone deployment (`standalone-values.yaml`) - `standalone-supported: yes` | `[B]` | done | `examples/standalone-values.yaml` (Task 7). |
| ArgoCD Application example that references the published Helm repo | `[B]` | done | `examples/argocd-app.yaml` (Task 7). |

---

## Telemetry

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| `ServiceMonitor` or `PodMonitor` exposed, or documented justification for not exposing metrics | `[B]` | covered by Task 15 | Task 15 adds an opt-in `podmonitor.yaml` template (default off, `metrics.podMonitor.enabled: false`). Justification for default-off: the upstream chart ships no native `/metrics` endpoint; enabling PodMonitor without a proper metrics endpoint produces scrape errors. Users running kube-state-metrics or the upstream Prometheus adapter can enable it. The justification is documented in `docs/configuration.md` telemetry section and `README.md`. |
| Application logs are written to stdout/stderr in a structured format (JSON preferred) | `[B]` | satisfied by upstream | Langfuse web and worker containers emit JSON-structured logs to stdout/stderr by default (no log-shipping sidecar needed). Documented in `README.md` telemetry section (Task 9). |

---

## Security

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| Containers do not run as root (or have documented justification if they must) | `[B]` | covered by Task 15 | Task 15 sets `runAsNonRoot: true` in the securityContext for the `langfuse-web` and `langfuse-worker` containers via the upstream chart's `podSecurityContext` passthrough. Bitnami sub-chart images already run non-root. Verified at Task 13. |
| No secrets hardcoded in templates or default values | `[B]` | done | All secrets are generated at runtime via `lookup`+`randAlphaNum` (no static values). The OIDC issuer placeholder is `REPLACE-ME` (not a real credential). Task 12 grep assertion confirms no inline secret values in rendered output. |
| OIDC scopes minimally scoped to what the app needs | `[B]` | done | Default scopes in `values.yaml`: `[openid, profile, email]` - the minimum required for Langfuse user identity (Task 4). |
| Upstream container images pinned to a specific tag or digest (never `latest`) | `[B]` | done | Langfuse `3.179.1`; all Bitnami sub-chart images are `bitnamilegacy/*` with pinned tags (frozen by upstream chart `langfuse 1.5.34`). Task 12 `no :latest` assertion. |

---

## Release Engineering

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| Chart is publishable to `nebari-dev.github.io/helm-repository` | `[B]` | done | Chart source added to `nebari-dev/helm-repository` `charts/nebari-langfuse/` (helm-repository#50, merged 2026-06-12); published to the central Helm index and `oci://quay.io/nebari/charts/nebari-langfuse`. |
| Release workflow is configured and at least one pre-1.0 release has been published | `[B]` | done | `nebari-langfuse-0.1.0` released in this repo (GitHub release + chart tgz) and published through the central helm-repository release workflow. |
| `appVersion` in `Chart.yaml` matches the upstream application version being wrapped | `[B]` | done | `chart/Chart.yaml` `appVersion: "3.179.1"` matches `langfuse/langfuse` chart `1.5.34` which deploys Langfuse `3.179.1` (Task 1). |

---

## Pre-sales Verification

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| Pre-sales engineer has run the demo end-to-end and signed off on the happy path | `[A]` | EXTERNAL - human sign-off | Required before Alpha promotion. Cannot be completed in this repo. Pre-sales engineer must run through journeys 1, 2, 3, 4 on the dev kind cluster. |
| Pre-sales engineer has confirmed the pack can be demoed without engineering on the call | `[B]` | EXTERNAL - human sign-off | Required for Beta promotion. Pre-sales rep must confirm. |
| `demo_notes` in `pack-metadata.yaml` reflects current known demo gotchas | `[B]` | done | `demo_notes` set in `pack-metadata.yaml` (this commit): SSO config requirement and bundled-datastore caveat. |

---

## Sign-off

| Item | Level | Status | Where / Notes |
|------|-------|--------|---------------|
| Pack owner approves the promotion PR | `[A]` | EXTERNAL - human | @dcmcand must approve the Alpha promotion PR. |
| Pre-sales rep approves the promotion PR | `[A]` | EXTERNAL - human | Pre-sales rep must approve the Alpha promotion PR. |
| Tech lead approves the promotion PR | `[B]` | EXTERNAL - human | Tech lead must approve the Alpha -> Beta promotion PR. |

---

## Summary of External / Human Items (cannot be completed in this repo)

1. **Listed in `software-pack-dashboard/tracked-packs.yaml`** - DONE
   (software-pack-dashboard#2, merged 2026-06-12).
2. **At least one pre-1.0 release actually published** - DONE: `nebari-langfuse-0.1.0`
   released in this repo and published via `nebari-dev/helm-repository`
   (helm-repository#50; Helm index + quay.io OCI verified installable).
3. **Pre-sales "demoable without engineering" sign-off** - OUTSTANDING: pre-sales engineer
   must run the demo end-to-end and sign off.
4. **Pre-sales rep + tech lead promotion-PR approvals** - OUTSTANDING: required reviewers
   per the maturity model promotion process table. The pack is published at `level: alpha`
   with these sign-offs accepted as outstanding by the pack owner.

## Summary of "Covered by Task N" Items

The following checklist items are satisfied by tasks not yet executed at the time this
document was authored. They will be confirmed at the Task 13 verification gate:

- Tasks 9: README (prerequisites, Known Limitations, auth setup, troubleshooting, telemetry)
- Task 11: CI (`helm lint`, `helm template` enabled/disabled, kubeconform, release workflow)
- Task 12: Pre-cluster render/unittest gate (no inline secrets, no `:latest`, `runAsNonRoot`)
- Task 13: Journey verification (installs cleanly, NebariApp Ready, auth works, probes)
- Task 15: securityContext (`runAsNonRoot`), PodMonitor opt-in template
- Task 16: E2E black-box tests (unauthenticated ingestion rejected = journey 11)
