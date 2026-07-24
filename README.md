# goldenpath-argocd

GitOps golden path for installing and self-managing ArgoCD on the `pimaster`
k3s cluster (control-plane / pipeline node).

## Layout

```
goldenpath-argocd/
├── bootstrap/          # Applied manually, once, with kubectl. Not managed by ArgoCD.
│   ├── argocd-install.yaml   # One-shot install of ArgoCD core (from upstream manifests)
│   └── root-app.yaml         # App-of-apps: hands control to apps/ after bootstrap
├── apps/
│   └── argocd/          # ArgoCD managing itself via the argo-cd Helm chart
│       ├── argocd.yaml
│       └── values.yaml
└── README.md
```

## Why bootstrap/ is separate from apps/

`apps/` is the desired-state tree that ArgoCD reconciles continuously —
everything in it is managed *by* ArgoCD, including ArgoCD's own Helm
release (self-management).

`bootstrap/` is the one-time, chicken-and-egg step: before ArgoCD exists in
the cluster, nothing can sync anything. Those two manifests are applied
directly with `kubectl`, exactly once, to get ArgoCD running and pointed at
this repo. After that, you should never need to touch `bootstrap/` again
unless you're rebuilding the cluster from scratch.

Keeping this split (rather than putting root-app.yaml under apps/) makes the
manual/automated boundary explicit and matches the standard "app of apps"
convention.

## First-time install

```bash
# 1. Bootstrap ArgoCD itself (manual, one-time)
kubectl apply -f bootstrap/argocd-install.yaml

# wait for it to come up
kubectl -n argocd rollout status deploy/argocd-server

# 2. Point ArgoCD at this repo — from here on, ArgoCD manages itself
#    and everything else added under apps/
kubectl apply -f bootstrap/root-app.yaml
```

After step 2, ArgoCD reconciles `apps/argocd/argocd.yaml`, recognizes it
owns its own running Helm release, and converges — no downtime, since the
running pods already match desired state.

**From this point on: never `kubectl apply` or `helm upgrade` ArgoCD
directly.** All changes go through a Git commit to `apps/argocd/values.yaml`.

## Escape hatch

Self-management can wedge itself if a bad config breaks ArgoCD's own pods.
Keep `kubectl` access as a manual fallback:

```bash
# roll back a bad commit
git revert <commit> && git push
# or patch directly if ArgoCD can't sync itself out of the hole
kubectl -n argocd edit deploy argocd-server
```

## Adding new apps

Drop a new folder under `apps/<name>/` with its own `Application` manifest
and `values.yaml`. The root-app's `directory.recurse: true` picks it up
automatically — no manual `kubectl apply` needed.
