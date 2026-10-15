# argocd-playground

A small repo for learning ArgoCD and GitOps. It deploys nginx to a local [kind](https://kind.sigs.k8s.io/) cluster using Kustomize overlays, with ArgoCD keeping the cluster in sync with this repo.

## Layout

```
.
├── base/                # shared manifests (nginx Deployment)
│   ├── deployments.yaml
│   └── kustomization.yaml
├── overlays/
│   └── dev/             # dev: namespace "dev"
│       └── kustomization.yaml
├── argocd/
│   └── dev-app.yaml     # ArgoCD Application for dev (auto-sync)
├── NOTES.md             # running log: commands, problems, what I learned
└── CLAUDE.md            # ground rules for working with Claude Code
```

Planned: `overlays/prod` and `argocd/prod-app.yaml`, with manual sync for prod.

## How it works

- `base/` holds the manifests that are the same everywhere. Each overlay patches them for one environment.
- `argocd/dev-app.yaml` tells ArgoCD to watch `overlays/dev` on `main`, deploy it to the `dev` namespace, and sync automatically (`prune` and `selfHeal` on).
- Push a change to git and ArgoCD applies it. Change the cluster by hand and ArgoCD puts it back.

## Run it yourself

Needs Docker, `kubectl` and `kind`.

```
kind create cluster --name argocd-playground

kubectl create namespace argocd
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd
```

`--server-side` is needed because the ApplicationSet CRD is too large for a normal client-side apply.

Open the UI:

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Then go to https://localhost:8080 and log in as `admin`.

Preview what an overlay renders (this only prints, it doesn't deploy):

```
kubectl kustomize overlays/dev
```

Create the Application (the only manual apply):

```
kubectl apply -f argocd/dev-app.yaml
```

The Application points at the GitHub URL in `argocd/dev-app.yaml`. If you fork this repo, change `repoURL` to your own.

## Clean up

```
kind delete cluster --name argocd-playground
```
