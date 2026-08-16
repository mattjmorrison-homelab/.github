# homelab-argocd

## TODO: upgrade to helm

## Goal

Bootstrap ArgoCD into the k3s cluster. ArgoCD becomes the single tool used to deploy everything else to the cluster going forward.

## Steps

### 1. Install ArgoCD

Apply the official ArgoCD install manifest to the cluster:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Wait for ArgoCD to be ready

```
kubectl wait --for=condition=available --timeout=120s deployment/argocd-server -n argocd
```

### 3. Access the ArgoCD UI

Forward the ArgoCD server port locally:

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then open <https://localhost:8080> in a browser.

### 4. Get the initial admin password

```
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

Login with username `admin` and the password above.

### 5. Done

ArgoCD is now running. The overall structure for deploying things to the cluster is:

- `homelab-argocd` (this repo) — run once, bootstraps ArgoCD into k3s
- `homelab-apps` — an "App of Apps" repo that ArgoCD watches; contains an ArgoCD `Application` manifest for each tool you want deployed
- `homelab-prometheus`, `homelab-grafana`, etc. — one repo per tool; each has its own deployment, tests, and lifecycle

To add a new tool: create a `homelab-<tool>` repo, then add an `Application` manifest pointing at it in `homelab-apps`. ArgoCD picks it up automatically.

## Upgrading ArgoCD

Re-run step 1 with a newer manifest URL (replace `stable` with a specific version tag). ArgoCD handles its own rolling update.
