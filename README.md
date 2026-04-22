# cluster-applications

This repo is the **source-of-truth for user-facing applications** deployed across clusters via Argo CD.

It is intentionally separate from infrastructure/addons (which live in `gitops-lab`).

## Structure

```
cluster-applications/
├── bootstrap/
│   └── argocd/
│       └── root-app.yaml       # App-of-Apps — apply once per cluster to bootstrap
├── apps/
│   └── <app-name>/
│       └── application.yaml    # Argo CD Application CRD for this app
└── clusters/
    └── <cluster-name>/
        └── kustomization.yaml  # Which apps run on this cluster
```

## Adding a new cluster

1. Create `clusters/<cluster-name>/kustomization.yaml` listing the apps to deploy.
2. Apply the root app to that cluster:

```bash
export KUBECONFIG=~/.kube/<cluster-name>
sed 's|<cluster>|<cluster-name>|g' bootstrap/argocd/root-app.yaml \
  | kubectl apply -f -
```

## Adding a new app

1. Create `apps/<app-name>/application.yaml` — an Argo CD `Application` pointing at
   the app's GitOps manifests (separate repo or sub-path).
2. Add it to the relevant cluster kustomization(s):

```yaml
# clusters/k8s-lab/kustomization.yaml
resources:
  - ../../apps/demo-vite-ui/application.yaml
  - ../../apps/<app-name>/application.yaml
```

3. Open a PR — CI will run `kustomize build clusters/<cluster>` to validate.

## Deploying

Changes merged to `main` are picked up automatically by Argo CD (`automated: {prune: true, selfHeal: true}`).

To apply manually (e.g. first-time bootstrap):

```bash
export KUBECONFIG=~/.kube/lima-k8s-lab
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Commit message format

- `app(<name>): <summary>` — changes scoped to one app
- `cluster(<name>): <summary>` — cluster-level changes
- `bootstrap: <summary>` — root app or ArgoCD bootstrap changes
