# ArgoCD Playground Notes

Running log of what I do, what I learn, and what breaks. Newest entries go at the bottom of each section.

## Decisions

- 2026-10-14: Local cluster with **kind** (not minikube, not EKS). Lightest option and fastest to wipe and recreate.
- 2026-10-14: EKS (Terraform-managed, separate repo/folder) is parked until ArgoCD is learned locally.
- 2026-10-14: Switchyard-on-Kubernetes parked for Week 2 (start from `muksyed1/switchyard-llm-routing-aws`).

## Progress

- [x] Docker installed and running (`docker version` shows Client and Server)
- [x] `kubectl` and `kind` installed (`brew install kind kubectl`)
- [x] kind cluster created (`kubectl get nodes` shows Ready)
- [x] Fix kubectl client version skew (client 1.32.2 vs cluster 1.37.0), see "Things that broke"
- [x] ArgoCD installed in the `argocd` namespace (all pods Running)
- [x] ArgoCD UI reachable at https://localhost:8080 and logged in
- [x] `base/` app renders (`kubectl kustomize base`)
- [x] `overlays/dev` renders (`kubectl kustomize overlays/dev`)
- [ ] `overlays/prod` renders (`kubectl kustomize overlays/prod`)
- [x] Repo pushed to GitHub
- [x] Dev Application created: auto-sync, shows Synced and Healthy
- [ ] Prod Application created: manual sync, OutOfSync until synced by hand
- [x] Change a replica count in git and ArgoCD applies it (1 -> 2 and 2 -> 3 both applied, but I clicked Sync both times)
- [ ] Prove auto-sync with no clicks: push a change, don't touch the UI, wait for the polling interval

## Command log

Commands I ran, with the date and what happened.

```
2026-10-14  brew install kind kubectl
            -> already installed (kind 0.33.0, Homebrew kubectl 1.37.1)
2026-10-14  docker version
            -> first "Cannot connect to the Docker daemon"; Docker Desktop wasn't running.
               After opening it: Docker Desktop 4.45.0, Engine 28.3.3
2026-10-14  (killed and restarted Firefox first: RAM was 85%, ~10 GB of it Firefox)
2026-10-14  kind create cluster --name argocd-playground
            -> node image kindest/node:v1.37.0, container argocd-playground-control-plane
2026-10-14  kubectl get nodes
            -> argocd-playground-control-plane   Ready   control-plane   v1.37.0
2026-10-14  kubectl version -> client 1.32.2 (Docker Desktop's copy), skew warning
2026-10-14  type -a kubectl; ls -l on both binaries; hash -r
            -> client 1.37.1, server 1.37.0, warning gone
2026-10-14  kubectl create namespace argocd; kubectl apply -n argocd -f .../install.yaml
            -> everything created except applicationsets CRD (annotation too long)
2026-10-14  kubectl apply --server-side --force-conflicts -n argocd -f .../install.yaml
2026-10-14  kubectl get crd
            -> applications, applicationsets, appprojects (all v1alpha1)
2026-10-14  kubectl get pods -n argocd
            -> 7/7 Running, 1/1 ready, 0 restarts (application-controller, applicationset-controller,
               dex-server, notifications-controller, redis, repo-server, server)
2026-10-14  kubectl port-forward svc/argocd-server -n argocd 8080:443   (leave running in its own tab)
2026-10-14  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d | pbcopy
            -> logged in at https://localhost:8080 as admin (first try failed: stray % copied)
2026-10-14  wrote base/deployments.yaml (nginx, pinned to nginx:1.27.2, not :latest) and base/kustomization.yaml
2026-10-14  tree showed overlays/ nested inside base/ (base/overlays/dev). Needs to move to the repo root
            so overlays/dev can reference ../../base
2026-10-14  moved overlays/ to the repo root; kubectl kustomize overlays/dev
            -> same Deployment as base plus namespace: dev (kustomize only prints, it never touches the cluster)
2026-10-14  git init -b main; git commit; git push -> github.com/muksyed1/argocd-playground (public), commit 6cbca54
2026-10-14  wrote argocd/dev-app.yaml. Fixed in review: syncPolicy "options" is not a field; "automated" must be a map
            (prune: true, selfHeal: true), not a list; syncOptions belongs inside syncPolicy as a list of strings
2026-10-14  kubectl apply --dry-run=server -f argocd/dev-app.yaml -> "created (server dry run)"
2026-10-14  kubectl apply -f argocd/dev-app.yaml; kubectl get application -n argocd
            -> nginx-dev   Synced   Healthy
            (note: kubectl get all -n argocd shows only ArgoCD's own pods; the app runs in -n dev)
2026-10-14  kubectl get pods -n dev -> nginx-67fbcd88dd-x8zws 1/1 Running. UI (ArgoCD v3.5.5) shows nginx-dev Healthy + Synced,
            path overlays/dev, destination in-cluster, namespace dev
2026-10-14  learned: `kubectl get all` only covers a fixed category of built-in types. Not shown: Applications (CRD),
            ConfigMaps, Secrets. Name the type: kubectl get applications,configmap,secret -n argocd
2026-10-14  kubectl kustomize base
            -> renders the Deployment, image nginx:1.27.2, no errors
2026-10-14  changed replicas 1 -> 2 in base/deployments.yaml; commit 4f2db06 pushed.
            Nothing happened for several minutes. kubectl get application nginx-dev -n argocd -o jsonpath='{.status.sync.revision}'
            showed b5a62e8 (one commit behind). Local render (kubectl kustomize overlays/dev | grep replicas) showed 2,
            so git and overlay were fine; ArgoCD just hadn't polled yet. Synced manually in the UI -> 2 pods.
2026-10-14  changed replicas to 3 (commit 9dbbd4d "Increased to 3 replicas"), pushed. ArgoCD showed 3 replicas within seconds.
            CORRECTION: the app's Events tab shows "admin initiated sync to main (9dbbd4d)", so I had clicked Sync.
            This was NOT a hands-off auto-sync test. Still to do: push a change and click nothing.
2026-10-14  scale test: kubectl scale deployment nginx -n dev --replicas=1 -> reverted to 3 within seconds (selfHeal).
            App Events tab: OutOfSync -> Synced. Deleting pods via the ArgoCD UI is recorded as "admin deleted resource /Pod ...";
            sync status stayed Synced for those (the ReplicaSet recreated them, not ArgoCD).
```

## Concepts I'm learning

- **kind** = Kubernetes IN Docker. Each node is a Docker container. Real, conformant cluster, not for production.
- **GitOps**: git is the source of truth. ArgoCD pulls from the repo and makes the cluster match.
- **Kustomize**: `base/` holds the shared manifests, `overlays/dev` and `overlays/prod` patch them per environment. Built into `kubectl`.
- **CRD** (CustomResourceDefinition): teaches Kubernetes a new object type beyond the built-ins. Not in my 2020 CKA notes. A controller in the cluster watches those objects and acts on them. Inspect with `kubectl get crd` and `kubectl explain application.spec`.
- **Application** (ArgoCD CRD): tells ArgoCD which repo, path, and cluster destination to sync. ArgoCD installs three CRDs: `applications`, `appprojects`, `applicationsets`.

- **What ArgoCD controls**: only the objects written in git (here the `nginx` Deployment). The ReplicaSet and Pods are created by Kubernetes controllers from the Deployment, so they are not in git and ArgoCD doesn't manage them directly.
- **Delete a pod vs. scale the Deployment** (tested in `dev`):
  - Delete a pod: the Deployment still says 3, matching git, so the app stays Synced. The ReplicaSet creates a replacement. ArgoCD does nothing. Events show only `replicaset/... SuccessfulCreate`.
  - `kubectl scale deployment nginx -n dev --replicas=1`: the Deployment now differs from git, so ArgoCD (selfHeal) sets it back to 3. Events show `deployment/nginx ScalingReplicaSet` 3 -> 1, then 1 -> 3.
- **Events show what changed, not who**: `kubectl get events -n dev --sort-by=.lastTimestamp`. Identical repeated events are merged under the latest timestamp. To see ArgoCD's actions use the UI History tab or `kubectl logs -n argocd argocd-application-controller-0`.
- **`kubectl logs nginx -n dev` fails** (pods "nginx" not found): `nginx` is the Deployment, pods are named `nginx-<hash>-<id>`. Use `kubectl logs deploy/nginx -n dev`, or run `kubectl get pods -n dev` for the exact name. Those are nginx's own logs, not a record of who replaced a pod.

## CKA notes vs. current Kubernetes

My CKA notes are from 2020 (about Kubernetes 1.18). Things that have changed:

- `kubectl run --generator=...` is removed. `kubectl run` now only creates a Pod. Use `kubectl create deployment`.
- Bare `--dry-run` must be `--dry-run=client`.
- `kubectl create deployment` now supports `--replicas` (check `--help` on my version).
- Typo in my notes: Deployment YAML uses `specs:`, it must be `spec:`.
- Typo in my notes: `kubeconfig config use-context` should be `kubectl config use-context <context-name>`. For kind the context is `kind-argocd-playground`.
- Docker is no longer the in-cluster runtime (containerd since ~1.24). PodSecurityPolicy is gone, replaced by Pod Security.

## Things that broke

Copy this block for each problem.

### kubectl client is older than the cluster

**What broke:** My terminal's `kubectl version --client` showed v1.32.2, but the kind cluster is v1.37.0. Skew of 5 minor versions; only +/-1 is supported.

**How I found it:** `which -a kubectl` and `type -a kubectl` listed two binaries. `ls -l` showed `/opt/homebrew/bin/kubectl` -> Homebrew kubernetes-cli 1.37.1, and `/usr/local/bin/kubectl` -> Docker Desktop's bundled copy (1.32.2). Running each by full path confirmed which was which. Homebrew's was first on PATH, but my shell had cached the Docker one before the Homebrew link existed.

**What fixed it:** `hash -r` (clears zsh's command-location cache). `kubectl version` then showed client 1.37.1, server 1.37.0, no skew warning. Docker's symlink left alone since Docker Desktop manages it.

**What I'd do differently:** Run `hash -r` (or open a new terminal) after installing or relinking a CLI tool, and compare client vs server version right after creating a cluster.

### ArgoCD install: applicationsets CRD too long

**What broke:** `kubectl apply -n argocd -f .../install.yaml` created everything except one resource and ended with `The CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long: may not be more than 262144 bytes`.

**How I found it:** It was the last line of the apply output. Client-side `kubectl apply` stores the whole manifest in a `last-applied-configuration` annotation, and this CRD is too big for the 256 KB annotation limit.

**What fixed it:** Re-ran with server-side apply: `kubectl apply --server-side --force-conflicts -n argocd -f .../install.yaml`. `--force-conflicts` was needed because the other resources were already created client-side. `kubectl get crd` then showed all three argoproj CRDs (applicationsets created about a minute after the other two).

**What I'd do differently:** Use `--server-side` for the ArgoCD install from the start.

### ArgoCD login: "Invalid username or password"

**What broke:** The login page rejected `admin` plus the password from `argocd-initial-admin-secret`, even though the port-forward was working (terminal showed `Handling connection for 8080`).

**How I found it:** The error came from the server, so the network was fine and the credentials were wrong. The password field showed about 17 dots, but ArgoCD's generated password is 16 characters. The password prints with no trailing newline, so zsh adds a `%` at the end, and I had copied it along with the password.

**What fixed it:** Piped the decoded password into the clipboard instead of copying from the terminal: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d | pbcopy`, then pasted with Cmd+V.

**What I'd do differently:** Use `| pbcopy` for secrets from the start, and count the dots in a password field.

### Docker daemon not running

**What broke:** `docker version` printed the Client section, then "Cannot connect to the Docker daemon at unix:///Users/muksyed/.docker/run/docker.sock".

**How I found it:** The error names the Docker Desktop socket path, so the app just wasn't running.

**What fixed it:** Opened Docker Desktop and waited until `docker version` showed the Server section.

**What I'd do differently:** Start Docker Desktop before any kind command.

### <short title>

**What broke:**

**How I found it:**

**What fixed it:**

**What I'd do differently:**

## Open questions

-
