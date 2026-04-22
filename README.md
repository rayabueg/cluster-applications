# cluster-applications

This repo is the **source-of-truth for user-facing applications** deployed across clusters via Argo CD.

It is intentionally separate from infrastructure and addons, which live in [`gitops-lab`](https://github.com/rayabueg/gitops-lab):

| Repo | Owns |
|---|---|
| `gitops-lab` | Cluster infra — namespaces, CRDs, Envoy Gateway, cert-manager, DNS, Argo CD config |
| `cluster-applications` | Argo CD `Application` CRDs for team apps (this repo) |

This repo is included as a **git submodule** inside [`k8s-lab`](https://github.com/rayabueg/k8s-lab) at `cluster-applications/`.

## Structure

```
cluster-applications/
├── bootstrap/
│   └── argocd/
│       └── root-app.yaml                  # App-of-Apps — apply once to bootstrap
├── apps/
│   └── <app-name>/
│       └── application.yaml               # Canonical Argo CD Application definition
└── clusters/
    └── <cluster-name>/
        ├── kustomization.yaml             # Which apps run on this cluster
        └── <app-name>-application.yaml    # Per-cluster copy (possibly with overrides)
```

> **Why copy instead of `../../` references?**  
> Kustomize disallows path traversal outside the build root for security.  
> Each cluster directory contains its own copy of the Application manifests so  
> cluster-specific overrides (image tags, target namespaces) can be patched cleanly.

## Adding a new cluster

1. Create `clusters/<cluster-name>/kustomization.yaml` listing the app manifest files.
2. Copy the relevant `apps/<app>/application.yaml` files into the cluster directory and update `destination.server` / image tags as needed.
3. Apply the root app to that cluster:

```bash
export KUBECONFIG=~/.kube/<cluster-name>
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Adding a new app

1. Create the canonical definition at `apps/<app-name>/application.yaml`.
2. Copy it into each cluster directory where it should run:

```bash
cp apps/<app-name>/application.yaml clusters/k8s-lab/<app-name>-application.yaml
```

3. Add the file to the cluster's `kustomization.yaml`:

```yaml
# clusters/k8s-lab/kustomization.yaml
resources:
  - demo-vite-ui-application.yaml
  - <app-name>-application.yaml
```

4. Open a PR — CI will run `kustomize build clusters/<cluster>` to validate.

## Deploying

Changes merged to `main` are picked up automatically by Argo CD (`automated: {prune: true, selfHeal: true}`).

To apply manually (e.g. first-time bootstrap):

```bash
export KUBECONFIG=~/.kube/lima-k8s-lab
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Relationship to gitops-lab

The `Application` manifests in this repo point at `gitops-lab` as their source:

```
cluster-applications  →  Argo CD Application CRD  →  gitops-lab (k8s manifests)
```

For example, `clusters/k8s-lab/demo-vite-ui-application.yaml` tells Argo CD to sync
`clusters/k8s-lab/gateway/` from `gitops-lab`. The actual `Deployment`, `Service`, and
`HTTPRoute` for `demo-vite-ui` live there — only the pointer lives here.

## Contributing (submodule workflow)

This repo is pinned as a submodule inside `k8s-lab`. After merging changes here, bump
the parent pointer:

```bash
# 1) commit + push changes here
git add -A
git commit -m "app(my-app): add to k8s-lab"
git push

# 2) bump the parent submodule pointer
cd /path/to/k8s-lab
git add cluster-applications
git commit -m "submodule(cluster-applications): bump to $(cd cluster-applications && git rev-parse --short HEAD)"
git push
```

## Commit message format

- `app(<name>): <summary>` — changes scoped to one app
- `cluster(<name>): <summary>` — cluster-level changes
- `bootstrap: <summary>` — root app or ArgoCD bootstrap changes
