# cluster-applications

Source-of-truth for user-facing applications deployed via Argo CD.

Intentionally separate from infrastructure/addons ([`cluster-addons`](https://github.com/rayabueg/cluster-addons)) so app changes and infra changes have independent review/merge cycles.

| Repo | Owns |
|---|---|
| `cluster-addons` | Cluster infra — namespaces, CRDs, Envoy Gateway, cert-manager, Hubble, Istio, Argo CD config |
| `cluster-applications` | Argo CD `ApplicationSet` CRDs for team apps + app manifests (this repo) |

## Structure

```
cluster-applications/
├── bootstrap/
│   └── argocd/
│       └── root-app.yaml        # Root Application — apply once to seed Argo CD
├── apps/
│   └── <app-name>.yaml          # One ApplicationSet per app
└── apps-envs/
    └── <app-name>/              # Kubernetes manifests for the app
        ├── kustomization.yaml
        ├── deployment.yaml
        ├── service.yaml
        └── httproute.yaml
```

- **`apps/`** — ArgoCD picks up every `.yaml` here as an `ApplicationSet`.
  Each ApplicationSet's `spec.generators[].list.elements` controls which clusters the app runs on.
- **`apps-envs/`** — The actual Kubernetes manifests (Deployments, Services, HTTPRoutes) for each app.
  ApplicationSets in `apps/` point here as their source path.

## Current apps

| App | Path | URL |
|---|---|---|
| `demo-vite-ui` | `apps-envs/demo-vite-ui/` | `http://localhost:8080/vite/` |
| `mesh-demo` | `apps-envs/mesh-demo/` | `http://localhost:8080/mesh-demo` |
| `gateway-demos` | `apps-envs/gateway-demos/` | `/hello`, `/ui`, `/` (echo) |

## Adding a new app

1. Create `apps-envs/<app-name>/` with Kubernetes manifests and a `kustomization.yaml`.
2. Create `apps/<app-name>.yaml` as an `ApplicationSet` pointing at `apps-envs/<app-name>/`.
   Use `apps/demo-vite-ui.yaml` as a reference.
3. Open a PR.

## Adding a new cluster

Add an entry to each relevant `apps/<app-name>.yaml`:

```yaml
spec:
  generators:
    - list:
        elements:
          - cluster: k8s-lab
            server: https://kubernetes.default.svc
          - cluster: my-new-cluster           # add this
            server: https://<api-server-url>  # add this
```

## Bootstrap

```bash
export KUBECONFIG=~/.kube/lima-k8s-lab
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Commit message format

- `app(<name>): <summary>` — changes scoped to one app
- `cluster(<name>): <summary>` — cluster-level changes
- `bootstrap: <summary>` — root app or Argo CD bootstrap changes

## Submodule workflow

This repo is pinned as a submodule inside `k8s-lab`. After merging here, bump the parent pointer:

```bash
cd /path/to/k8s-lab
git add cluster-applications
git commit -m "chore: update cluster-applications submodule"
git push
```
