# This fork: Flux on the local Floci cluster

This fork is synced by Flux in the local OpenChoreo cluster from
[openchoreo-poc](https://github.com/gmourahub/openchoreo-poc) (`local/flux.sh`). It follows
the [Flux tutorial](./README.md), with the differences below.

## How it syncs

| Kustomization | Path | Depends on |
|---|---|---|
| `flux-system` | `flux/` (Flux itself, the GitRepository, the Kustomizations) | |
| `namespaces` | `namespaces/` | |
| `platform-shared` | `platform-shared/` (ClusterWorkflowTemplates, ClusterResourceTypes) | |
| `oc-demo-platform` | `namespaces/default/platform/` | namespaces, platform-shared |
| `oc-demo-projects` | `namespaces/default/projects/` | oc-demo-platform |

Flux polls `main` every minute and re-applies every 5 minutes: a merged PR reaches the
cluster within about a minute, and a manual change in the cluster (console or kubectl) is
reverted at the next re-apply. Deleting a file deletes the resource (`prune: true`), except
the `default` namespace, which is marked `kustomize.toolkit.fluxcd.io/prune: disabled`.

Status: `kubectl get gitrepository,kustomization -n flux-system`. Force a sync:
`kubectl annotate gitrepository -n flux-system sample-gitops reconcile.fluxcd.io/requestedAt="$(date +%s)" --overwrite`.

## Upgrading Flux

Flux manages its own install from `flux/flux-system/gotk-components.yaml` (source-controller
and kustomize-controller only). To upgrade, regenerate that file with the new `flux` CLI and
merge the PR:

```bash
flux install --components=source-controller,kustomize-controller --export \
  > flux/flux-system/gotk-components.yaml
```

## Differences from the upstream sample

- Build workflows push images to Floci's ECR
  (`000000000000.dkr.ecr.us-east-1.localhost:4566`) instead of `host.k3d.internal:10082`,
  open PRs in this fork, and build with `--network host` (pod MTU 1450 vs. 1500 on podman's
  own bridge broke dependency downloads).
- The Doclet frontend component is `doclet-frontend`: the cluster already has a `frontend`
  component (doc-insights) in the `default` namespace.
- The Doclet postgres/nats ResourceReleaseBindings pin `resourceRelease` (required by
  OpenChoreo 1.3).
- `namespaces/default/projects/doc-insights/` holds the doc-insights app and
  `platform-shared/resource-types/` its AWS S3/SQS resource types. Its components have
  `autoDeploy: false`: releases and promotions happen by PR.

## Day-to-day

- **Build and release Doclet**: run the WorkflowRuns from the
  [tutorial, step 6](./README.md#step-6-build-and-deploy-the-doclet-application) with
  `componentName: doclet-frontend` for the frontend. Each run opens a PR here; merging it
  deploys to development.
- **Promote**: add a ReleaseBinding for the target environment (or run
  `bulk-gitops-release`) and merge.
- **Roll back**: point the ReleaseBinding's `releaseName` at an earlier ComponentRelease
  and merge.

The workflows read the GitHub token from the `gitops-token` and `git-token` secrets in Floci
Secrets Manager, as `{"git-token": "<PAT>"}` (`local/set-github-token.sh` in openchoreo-poc).
