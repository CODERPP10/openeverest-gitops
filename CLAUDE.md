# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This is a GitOps repository for deploying [OpenEverest](https://openeverest.io) (a cloud-native
database platform / Percona Everest fork) via ArgoCD onto a Kubernetes cluster. There is no
application source code here — the repo consists of vendored Helm charts, per-environment Helm
values, and ArgoCD `Application` manifests. ArgoCD (running in the `argocd` namespace) watches
this repo on `main` and reconciles the two Applications automatically (`syncPolicy.automated`).

## Repository layout

- `apps/` — ArgoCD `Application` CRs. These are the entry point: each points at a `charts/*` dir
  as the Helm source and a `values/*.yaml` file as Helm values.
  - `apps/everest-system.yaml` — deploys the `openeverest` chart (sync-wave `1`, installs first).
  - `apps/everest.yaml` — deploys the `everest-db-namespace` chart (sync-wave `2`, installs after).
  - Sync-wave ordering matters: `everest-system` (operator/server/OLM) must exist before
    `everest` (the DB namespace + operator subscriptions) can be reconciled.
- `charts/openeverest/` — vendored umbrella Helm chart for the core platform (operator, server,
  OLM, monitoring, CRDs, kube-state-metrics, etc., wired as chart dependencies in `Chart.yaml`).
  Installs into the `everest-system` namespace.
- `charts/everest-db-namespace/` — vendored sub-chart that provisions a database namespace
  (namespace, OperatorGroup, and OLM Subscriptions for the PXC/PSMDB/PostgreSQL operators).
  Installs into the `everest` namespace.
- `values/everest-system.yaml` / `values/everest.yaml` — the Helm values ArgoCD passes to each
  chart above. These are the actual per-environment configuration surface for this repo; the
  charts themselves are vendored upstream artifacts and should generally not be hand-edited.

## Working with this repo

- Charts under `charts/` are vendored (pulled from upstream OpenEverest/Percona Helm repos via
  `Chart.yaml` dependencies, `Chart.lock` pins exact subchart versions). Treat them as read-only
  unless intentionally re-vendoring/upgrading; prefer changing behavior through `values/*.yaml`.
- To change what gets deployed, edit `values/everest-system.yaml` (platform/operator/server/OLM/
  monitoring config) or `values/everest.yaml` (per-namespace DB operator toggles: `pxc`,
  `postgresql`, `psmdb`, `telemetry`, `cleanupOnUninstall`).
- To add a new ArgoCD-managed app, add a manifest under `apps/` with an appropriate
  `argocd.argoproj.io/sync-wave` annotation reflecting install-order dependencies, and a matching
  values file under `values/`.
- `repoURL`/`targetRevision` in `apps/*.yaml` point at `main` of this repo itself — pushing to
  `main` is what triggers ArgoCD to reconcile (this repo has no CI pipeline; validation happens
  via `helm template`/`helm lint` locally and then observing ArgoCD sync status in-cluster).

## Useful local commands

Render/validate a chart with a given values file before pushing (no CI does this for you):

```sh
helm dependency build charts/openeverest
helm template everest-system charts/openeverest -f values/everest-system.yaml
helm lint charts/openeverest -f values/everest-system.yaml

helm dependency build charts/everest-db-namespace
helm template everest charts/everest-db-namespace -f values/everest.yaml
helm lint charts/everest-db-namespace -f values/everest.yaml
```

`charts/openeverest/test/test-values.yaml` contains an alternate values file used for chart
testing/CI upstream — useful as a reference for exercising more of the chart's conditionals.
