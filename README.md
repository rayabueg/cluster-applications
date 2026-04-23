# cluster-applications

This repo is the **source-of-truth for user-facing applications** deployed across clusters via Argo CD.

It is intentionally separate from infrastructure and addons, which live in [`cluster-addons`](https://github.com/rayabueg/gitops-lab):

| Repo | Owns |
|---|---|
| `cluster-addons` | Cluster infra — namespaces, CRDs, Envoy Gateway, cert-manager, DNS, Argo CD config |
| `cluster-applications` | Argo CD `Application` CRDs for team apps (this repo) |

This repo is included as a **git submodule** inside [`k8s-lab`](https://github.com/rayabueg/k8s-lab) at `cluster-applications/`.

## Structure

```
cluster-applications/
├── bootstrap/
│   └── argocd/
│       └── root-app.yaml   # Root Application pointing ArgoCD at apps/
└── apps/
    └── <app-name>.yaml     # One ApplicationSet per app — list generator controls which clusters run it
```

There is no `clusters/` directory. Which clusters an app runs on is declared inside
each ApplicationSet's `spec.generators[].list.elements` list — one entry per cluster.
This mirrors the pattern used in `sdp-cluster-applications`.

## Adding a new cluster

1. Register the cluster in ArgoCD (see ArgoCD docs for `argocd cluster add`).
2. For each app that should run on the new cluster, add an entry to that app's `spec.generators[].list.elements`:

```yaml
# apps/demo-vite-ui.yaml
spec:
  generators:
    - list:
        elements:
          - cluster: k8s-lab
            server: https://kubernetes.default.svc
          - cluster: my-new-cluster           # add this
            server: https://<api-server-url>  # add this
```

3. Open a PR — CI validates YAML and ApplicationSet schemas.

## Adding a new app

1. Create `apps/<app-name>.yaml` as an ArgoCD `ApplicationSet`.
   Use `apps/demo-vite-ui.yaml` as a reference — copy it, update `metadata.name`,
   `spec.template.spec.source`, and the `list.elements` for target clusters.

2. Open a PR — CI will yamllint and schema-validate `apps/`.

## Deploying

Changes merged to `main` are picked up automatically by Argo CD (`automated: {prune: true, selfHeal: true}`).

To apply manually (e.g. first-time bootstrap):

```bash
export KUBECONFIG=~/.kube/lima-k8s-lab
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Relationship to cluster-addons

The `Application` manifests in this repo point at `cluster-addons` as their source:

```
cluster-applications  →  Argo CD Application CRD  →  cluster-addons (k8s manifests)
```

For example, `clusters/k8s-lab/demo-vite-ui-application.yaml` tells Argo CD to sync
`clusters/k8s-lab/gateway/` from `cluster-addons`. The actual `Deployment`, `Service`, and
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
