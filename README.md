# Kargo GitOps Demo

A demo repository for [Kargo](https://kargo.akuity.io) showcasing multi-stage GitOps promotion using the rendered-branch pattern. Built against Kargo v1.10.

## What this demos

- Automated promotion through a multi-stage pipeline (test → uat → prod)
- Three different promotion strategies: direct push, GitHub Actions dispatch, and pull request review
- Canary deployments via Argo Rollouts on one production stage
- Post-deployment verification using Argo Rollouts AnalysisTemplates

## Repository layout

```
guestbook/          Kustomize app — base + per-stage overlays
  base/             Deployment + Service
  stages/
    test/           Standard Kustomize overlay
    uat/            Standard Kustomize overlay
    prod-useast/    Standard Kustomize overlay
    prod-uswest/    Replaces Deployment with an Argo Rollouts Rollout (canary)

kargo/              Kargo resources, managed by ArgoCD
  project.yaml
  project-config.yaml
  warehouse.yaml
  stage.yaml
  promotiontasks.yaml
  analysis-template.yaml

apps/               ArgoCD ApplicationSet + Application (app-of-apps bootstrap)
.github/workflows/  Placeholder GHA workflow triggered by the uat promotion
```

## Pipeline

```
Warehouse (git main + image ghcr.io/dhpup/guestbook)
    |
    v
  test  ──── auto-promote ──── promote-standard
    |                          (renders manifests → stage/test branch → argocd sync)
    v
  uat   ──── auto-promote ──── promote-with-github-action
    |                          (renders + dispatches GHA workflow → argocd sync)
    |                          verification: pokemon-xp AnalysisTemplate
    |
    +──────────────────────────+
    v                          v
prod-useast                 prod-uswest
  auto-promote: false         auto-promote: true
  promote-with-pr             promote-standard
  (opens PR → merge gate)     (canary Rollout, manual pause at 50%)
```

## Promotion strategies

**`promote-standard`** (test, prod-uswest)
Clones main, renders Kustomize manifests for the stage, pushes to `stage/<stage>` branch, triggers ArgoCD sync.

**`promote-with-github-action`** (uat)
Same as standard, but also dispatches `example-github-action.yml` and waits for it to complete before triggering ArgoCD sync.

**`promote-with-pr`** (prod-useast)
Same render step, but pushes to an ephemeral branch and opens a pull request targeting `stage/prod-useast`. Promotion blocks until the PR is merged.

## Prerequisites

- Kargo v1.10+ ([install docs](https://docs.kargo.io/getting-started/installation))
- ArgoCD with the `kargo-demo` AppProject and a cluster registered as `mac1` (workload) and `kargo1` (Kargo control plane)
- Argo Rollouts installed on the workload cluster (for prod-uswest)
- A `githubtoken` Secret in the `guestbook` namespace containing a GitHub PAT with `repo` and `workflow` scopes (for the uat GHA dispatch)

## Setup

1. Fork this repo and update the `repoURL` references from `https://github.com/dhpup/kargo.git` to your fork across `kargo/promotiontasks.yaml`, `kargo/warehouse.yaml`, and `apps/guestbook.yaml`.

2. Apply the app-of-apps bootstrap to your ArgoCD instance:
   ```sh
   kubectl apply -f app-of-apps.yaml
   ```
   ArgoCD syncs the `apps/` directory, which creates the `guestbook` ApplicationSet and the `kargo-guestbook` Application. The latter deploys everything under `kargo/` to the Kargo control plane.

3. Create the GitHub token secret in the `guestbook` namespace:
   ```sh
   kubectl create secret generic githubtoken \
     --namespace guestbook \
     --from-literal=token=<your-pat>
   ```

4. Push a new image tag to `ghcr.io/<your-user>/guestbook` or commit to `main` — the Warehouse will detect it and freight will begin flowing through the pipeline.
