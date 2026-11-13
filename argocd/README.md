# GitOps with Argo CD (OpenChoreo on Floci)

This fork is synced by Argo CD instead of Flux. Argo CD runs in the local
OpenChoreo cluster from [openchoreo-poc](https://github.com/gmourahub/openchoreo-poc)
(`local/argocd.sh`), and its UI is at http://localhost:10083.

## How it syncs

`local/argocd.sh` creates a root Application that syncs `argocd/apps/`
(app-of-apps). Each file there is one Application, applied in sync-wave order,
the same order the Flux `dependsOn` chain gives:

| Wave | Application | Path |
|---|---|---|
| 0 | `namespaces` | `namespaces/` |
| 0 | `platform-shared` | `platform-shared/` (ClusterWorkflowTemplates, ClusterResourceTypes) |
| 1 | `oc-demo-platform` | `namespaces/default/platform/` |
| 2 | `oc-demo-projects` | `namespaces/default/projects/` |

All Applications use automated sync with prune and self-heal: a change merged to
`main` reaches the cluster within about a minute, and a manual change in the
cluster is reverted. Deleting a file deletes the resource.

## Differences from the upstream sample

- Build workflows push images to Floci's ECR
  (`000000000000.dkr.ecr.us-east-1.localhost:4566`) instead of
  `host.k3d.internal:10082`, and open PRs in this fork.
- The Doclet frontend component is `doclet-frontend`: the cluster already has a
  `frontend` component (doc-insights) in the `default` namespace.
- `namespaces/default/projects/doc-insights/` holds the doc-insights app
  (exported from the cluster), and `platform-shared/resource-types/` its
  AWS S3/SQS resource types. Its components have `autoDeploy: false`: releases
  and promotions happen by PR, not by the controller or the console.

## Day-to-day

- **Build and release Doclet**: run the WorkflowRuns from
  [the Flux tutorial, step 6](../flux/README.md#step-6-build-and-deploy-the-doclet-application)
  using `componentName: doclet-frontend` for the frontend. Each run opens a PR
  here; merging it deploys to development.
- **Promote**: add a ReleaseBinding for the target environment (or run
  `bulk-gitops-release`) and merge.
- **Roll back**: point the ReleaseBinding's `releaseName` at an earlier
  ComponentRelease and merge.

The workflows read the GitHub token from the `gitops-token` (and optionally
`git-token`) secret in Floci Secrets Manager, as `{"git-token": "<PAT>"}`.
