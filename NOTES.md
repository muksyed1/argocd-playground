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
- [x] `overlays/prod` renders (`kubectl kustomize overlays/prod`): namespace prod, replicas 3 (dev stays at 2 from base)
- [x] Repo pushed to GitHub
- [x] Dev Application created: auto-sync, shows Synced and Healthy
- [x] Prod Application created: manual sync, OutOfSync until synced by hand (synced by hand in the UI -> Synced + Healthy)
- [x] Change a replica count in git and ArgoCD applies it (1 -> 2 and 2 -> 3 both applied, but I clicked Sync both times)
- [x] Prove auto-sync with no clicks: push a change, don't touch the UI, wait for the polling interval (replicas 3 -> 2, commit d7f72ed)
- [x] Helm: scaffold `charts/nginx`, trim it, `helm lint` and `helm template` pass (lint passes; confirm the render output)
- [x] Helm: `argocd/nginx-helm-dev-app.yaml` deployed to `helm-dev`, shows Synced and Healthy
- [x] Helm: change a value in git and watch ArgoCD apply it (replicaCount 2 -> 3 in values.yaml, 3 pods in helm-dev)
- [x] Helm prod: `charts/nginx/values-prod.yaml` and `argocd/nginx-helm-prod-app.yaml` (no `automated`, `helm.valueFiles: values-prod.yaml`, namespace helm-prod) applied; shows OutOfSync
- [x] Helm prod: synced by hand, 3 pods Running in `helm-prod` (all four Applications Synced + Healthy)
- [ ] Helm prod: trim `values-prod.yaml` down to only what differs (`replicaCount: 3`), push, confirm the app stays Synced

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

## Prod gate test (image bump)

- Checked the new tag exists before pushing: `docker manifest inspect nginx:1.31.7 > /dev/null && echo exists`.
- Changed the image in `base/deployments.yaml` from `nginx:1.27.2` to `nginx:1.31.7`, pushed, clicked nothing.
- **Dev (auto-sync)** rolled by itself: new ReplicaSet (`nginx-76d8bcb44`), pods replaced one at a time (new pod Running before the old one Terminating), no downtime.
- **Prod (manual)** showed OutOfSync and kept running 1.27.2. The Diff showed only the `image:` line changing (1.27.2 -> 1.31.7). Clicked Sync (left prune unchecked: it deletes objects removed from git, nothing to prune here) and prod rolled the same way: `kubectl get pods -n prod -w` showed 3 old pods (`nginx-67fbcd88dd-*`) replaced one at a time by 3 new ones (`nginx-76d8bcb44-*`). The pod-template hash is the same in dev and prod because the template is identical.
- Rolling update defaults seen in the Diff: `strategy: RollingUpdate`, `maxSurge: 25%`, `maxUnavailable: 25%`.
- A pod template change (image) creates a new ReplicaSet. A replicas change does not.

## Helm

Goal: deploy nginx a second way, with a Helm chart, as its own ArgoCD Application, and compare it with Kustomize.

**Setup**
- Helm 4.3.1 was already installed (`brew info helm`: installed on request, linked). It's a formula: `brew install helm`, not `--cask`.
- `helm create`, `helm lint` and `helm template` work the same as in Helm 3.
- ArgoCD bundles its own Helm in the repo-server, so the version on my laptop only matters for local previews.

**Helm vs Kustomize**
- Kustomize: plain YAML in `base/`, overlays patch it per environment. No templating.
- Helm: a chart is YAML with `{{ }}` placeholders plus `values.yaml` (the settings). Change the values per environment instead of patching.
- Helm also packages and versions charts, so public charts can be installed.
- ArgoCD runs `helm template` itself, so there is no Helm release in the cluster. `helm ls` shows nothing for an ArgoCD-managed chart; look in ArgoCD instead.

**Chart layout (from `helm create`)**
- `Chart.yaml`: chart name and version.
- `values.yaml`: the settings (image, replicas, service).
- `templates/`: manifests with `{{ .Values.something }}` placeholders (deployment, service, helpers).

**Plan**
1. Scaffold: `helm create charts/nginx`, read `Chart.yaml`, `values.yaml`, `templates/deployment.yaml`.
2. Trim to a Deployment and a Service, settings in `values.yaml`.
3. Check locally (touches nothing): `helm lint charts/nginx` and `helm template nginx charts/nginx`.
4. Add `argocd/nginx-helm-app.yaml`: Application with `source.path: charts/nginx`, destination namespace `helm-dev` (a separate namespace so it doesn't fight the Kustomize `nginx` Deployment in `dev`).
5. Push, `kubectl apply -f` the Application, check it syncs.
6. Change a value in git and watch ArgoCD apply it.

**Rough learning estimate**
- First working chart deployed through ArgoCD: about one 1.5 to 2 hour session.
- Reading other people's charts comfortably (`if`, `range`, helpers): 2 to 3 sessions.
- Multi-environment values, subcharts, dependencies: a few more.

**Log**
- `helm create charts/nginx` failed: `Error: stat .../charts: no such file or directory`. `helm create` makes the chart folder but not its parent. Fix: `mkdir charts` first.
- Scaffold files: `Chart.yaml`, `values.yaml`, `templates/` (`_helpers.tpl`, `deployment.yaml`, `service.yaml`, `serviceaccount.yaml`, `hpa.yaml`, `ingress.yaml`, `httproute.yaml`, `NOTES.txt`, `tests/`), `charts/` (empty, for subcharts).
- Learned: `helm template nginx charts/nginx --set replicaCount=3 | grep -n replicas` -> `--set` applies to that one command only. Nothing stored, cluster untouched. Precedence: `values.yaml` < `-f file` < `--set`.
- Helm templates are Go templates (plus Sprig), not Jinja2 like Ansible: `{{ }}` for everything (values and logic), `{{- if }} ... {{ end }}`, `.Values.x` with a leading dot. `{{-` trims whitespace and must have no space between the braces and the dash.
- Converted my own manifest instead of trimming the scaffold: copied `base/deployments.yaml` to `charts/nginx/templates/deployments.yaml`, replaced `replicas` with `{{ .Values.replicaCount }}` and the image with `"{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"`. Values: `replicaCount: 2`, `image.repository: nginx`, `image.tag: "1.31.7"`. No `namespace:` in the template (ArgoCD sets it).
- `helm lint charts/nginx` -> 0 charts failed (only `[INFO] Chart.yaml: icon is recommended`).
- `helm template nginx charts/nginx` renders `replicas: 2` and `image: "nginx:1.31.7"`; `--set replicaCount=5` renders `replicas: 5`.
- The namespace is not in the chart or `values.yaml`. It comes from the Application's `destination.namespace` (`helm-dev`). Leaving it out keeps the chart reusable. Helm exposes it to templates as `{{ .Release.Namespace }}`.
- Wrote `argocd/nginx-helm-dev-app.yaml`. Review caught `metadata.name: nginx-dev`, a clash with the existing Kustomize Application (same name + namespace), which would have overwritten it and, with `prune: true`, could have deleted its pods. Renamed to `nginx-helm-dev`. A server dry run says `configured` for an existing name and `created` for a new one.
- Pushing doesn't create an Application. ArgoCD only watches paths that existing Applications point to, so each new Application is applied once with `kubectl apply -f`.
- `kubectl apply -f argocd/nginx-helm-dev-app.yaml` -> `nginx-helm-dev created`. Within seconds: three Applications (`nginx-dev`, `nginx-helm-dev`, `nginx-prod`) all Synced + Healthy; 2 pods Running in `helm-dev`. The ReplicaSet hash (`nginx-76d8bcb44`) matches the Kustomize pods because the pod template is identical.

### Helm prod Application: values-prod.yaml not found

**What broke:** `nginx-helm-prod` showed `Failed to load target state ... helm template . --name-template nginx-helm-prod --namespace helm-prod ... --values <path>/charts/nginx/values-prod.yaml ... Error: open .../values-prod.yaml: no such file or directory`.

**How I found it:** `git status` showed `argocd/nginx-helm-prod-app.yaml` and `charts/nginx/values-prod.yaml` as untracked. The file existed on my laptop (`ls` found it), but ArgoCD reads the GitHub copy, which didn't have it yet. The error message also prints the exact command ArgoCD runs: `helm template` with `--name-template` (the Application name), `--namespace` (the destination namespace) and `--values` (valueFiles, resolved relative to `source.path`).

**What fixed it:** Committed and pushed (adac27d), confirmed with `git ls-files charts/nginx`, then Refresh (Hard Refresh if needed) on the app. The error cleared and the app went OutOfSync, which is right for a manual-sync app.

**What I'd do differently:** Push before applying a new Application, and check `git ls-files` rather than `ls` (a file existing locally doesn't mean it's in the repo).

Also noticed: `values-prod.yaml` was a full copy of `values.yaml` (5,277 bytes). It should hold only the settings that differ (`replicaCount: 3`), or it silently pins every other value and prod stops following changes to `values.yaml`.

### Helm: `helm apply` doesn't exist

I ran `helm apply --dry-run=server -f argocd/nginx-helm-prod-app.yaml` -> `Error: unknown command "apply" for "helm"`. `apply` is a kubectl command, and the Application is a Kubernetes object (from the CRD), so use `kubectl apply`. Helm only lints and renders charts here; ArgoCD does the deploying.

### Helm: `helm lint` failed on NOTES.txt

**What broke:** `helm lint` errored: `nginx/templates/NOTES.txt:31:104 ... error calling include: template: no template "nginx.name" associated with template "gotpl"`.

**How I found it:** The error names the file and position. `NOTES.txt` calls `include "nginx.name"`, a helper defined in `templates/_helpers.tpl`, which I had deleted while trimming the chart.

**What fixed it:** Deleted `templates/NOTES.txt` (only help text printed by `helm install`; ArgoCD doesn't use it). Lint then passed.

**What I'd do differently:** Before deleting `_helpers.tpl`, run `grep -rn include charts/nginx/templates` to find templates that still depend on it.

## OpenShift (Developer Sandbox)

- Signed up at https://developers.redhat.com/developer-sandbox (first card, "OpenShift"). It first said "Your account needs manual approval", but the "ready to use" email arrived within minutes. Active for 30 days (ends around Nov 15, 2026), all data deleted at the end; Red Hat emails 3 days before. Log in at https://sandbox.redhat.com/. Calendar reminder set for Nov 14.
- It is a shared Red Hat OpenShift Service on AWS (ROSA) cluster. My project is `muksyed-dev` (OpenShift's name for a namespace) with ResourceQuotas, and project-level access only (not cluster-admin).
- No OpenShift GitOps (Argo CD) operator. Installed operators include Web Terminal, OpenShift Pipelines, Dev Spaces, Serverless, External Secrets, Ansible Automation Platform, OpenShift AI, Camel K, `gitops-primer` (Konveyor, not Argo CD). Software Catalog search "argocd" only found "RedHat Argo CD Agent - Agent Component", a Helm chart that connects managed clusters to a principal (hub-and-spoke), not the Argo CD server. For Argo CD on OpenShift the options are OpenShift Local (needs about 10.5 GB free RAM, I'd be admin) or OKD.
- The `>_` icon in the console opens a Web Terminal with `oc` preinstalled. The terminal runs as a pod in my project (`workspace...`); don't delete it. `oc whoami` -> muksyed, `oc project` -> muksyed-dev.

### OpenShift: stock nginx goes into CrashLoopBackOff

**What broke:** `oc create deployment nginx --image=nginx:1.31.7` -> pod went `Error`, then `CrashLoopBackOff`, 0/1 ready.

**How I found it:** `oc get pods`, then `oc logs deploy/nginx`: `[emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)`, plus `[warn] the "user" directive makes sense only if the master process runs with super-user privileges, ignored` and `can not modify /etc/nginx/conf.d/default.conf (read-only file system?)`. OpenShift runs containers as a random non-root user, and stock nginx wants root-owned directories and port 80.

**What fixed it:** `oc set image deployment/nginx nginx=nginxinc/nginx-unprivileged:1.31.7` (checked the tag first with `docker manifest inspect`). The unprivileged image runs as any user and listens on port 8080. The change created a new ReplicaSet and rolled out; the pod went `1/1 Running`.

**What I'd do differently:** On OpenShift, start with an image built for non-root (`nginxinc/nginx-unprivileged`) and expect port 8080. Also, `oc set image` is an imperative change that git doesn't know about; with ArgoCD self-heal it would be reverted, so the GitOps way is to change the image in the manifest and push.

### OpenShift: exposing nginx with a Service and a Route

- `oc expose deployment nginx --port=8080` creates a Service in front of the pods (8080 is the port `nginx-unprivileged` listens on).
- `oc expose service nginx` creates a Route from that Service; `oc get route` shows the hostname. A Route is OpenShift's way to publish a service on a public hostname, where plain Kubernetes would use an Ingress.
- Opened the hostname with `http://` in the browser and the "Welcome to nginx!" page loaded.
- Short names: `rc` = ReplicationController (the older predecessor of ReplicaSet; OpenShift's deprecated DeploymentConfig used them). `oc api-resources` lists the short names.
- `oc set image deployment/nginx nginx=<image>` is imperative: it edits the live Deployment, so git doesn't know about it, and ArgoCD self-heal would revert it. The GitOps way is to change the image in the manifest and push.

### OpenShift: from imperative commands to manifests in git

- The first files I saved in the web terminal (`~/openshift`) were `oc get ... -o yaml` dumps: `kind: List`, `uid`, `resourceVersion`, `creationTimestamp`, `status`, `namespace`, `host`. Not usable as manifests. My `service.yaml` dump even showed the wrong Service: `modelmesh-serving`, which the Sandbox's ModelMesh controller creates in my project (it was the "1 Service" on the project page). Don't touch it; name my own objects when deleting (`oc delete route,svc,deployment nginx`, never `--all`).
- `oc get` has no `--dry-run` (only commands that create or change things do).
- Wrote clean manifests by hand in `openshift/` in this repo: `deployment.yaml` (nginx-unprivileged:1.31.7), `service.yaml` (8080 -> 8080, selector `app: nginx`), `route.yaml` (`route.openshift.io/v1`, `to` Service `nginx`, `targetPort: 8080`; no namespace, no host, no status). Review caught: the Route was still a `kind: List` wrapper with `uid`/`resourceVersion`; a mis-indented `metadata:`; leftover `kind: List` at the bottom; `kind: Route` deleted by mistake.
- `oc apply` straight from GitHub (public repo, no login, a one-time fetch): `oc apply -f https://raw.githubusercontent.com/muksyed1/argocd-playground/main/openshift/<file>.yaml`. It is not GitOps: nothing watches the repo, so edits only take effect when I run `oc apply` again.
- Ran `oc delete route,svc,deployment nginx`, then applied the three files: each printed `created`. Deployment 1/1, Service `nginx` ClusterIP on 8080, Route admitted with the same generated hostname as before (OpenShift builds it from route name + project).
- Container-name caution: the manifest names the container `nginx-unprivileged`, but the imperative Deployment's container was `nginx`. Applying over an existing Deployment merges containers by name (could leave two), so I deleted the old objects first.

## Memory

- RAM hit 85-86% on the laptop. Early on the main user was Firefox (about 10 GB). Later the top process was Docker at 6.4 GB, even though `docker stats --no-stream` showed the kind container using only 1.64 GiB: the Docker Desktop VM holds extra memory as cache.
- Fix: Docker Desktop > Settings > Resources > Memory limit 7.9 GB -> 4 GB, Apply and restart. The kind cluster survived the restart (node Ready, 7 ArgoCD pods and 2 dev nginx pods Running, restart count 1 each).
- Decision: stay on kind. Moving to EKS is not needed for memory (and costs money); it stays parked.

## Concepts I'm learning

- **Auto-sync confirmed hands-off**: pushed d7f72ed (replicas 3 -> 2), clicked nothing. Several minutes later dev dropped to 2 pods by itself. When I checked mid-wait, `.status.sync.revision` was still the previous commit (0e9af0d) with `.status.reconciledAt` a few minutes old: ArgoCD polls on a timer (~3 min), so there is a delay. The UI's "Last Sync" is the last sync operation, not the last git check. Handy checks: `kubectl get application nginx-dev -n argocd -o jsonpath='{.status.sync.revision}{"\n"}{.status.reconciledAt}{"\n"}'` and `date -u`.

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

### Prod Application: "app path does not exist"

**What broke:** After `kubectl apply -f argocd/prod-app.yaml`, `nginx-prod` showed Sync `Unknown` and the error `Failed to load target state: failed to generate manifest for source 1 of 1: rpc error: code = Unknown desc = overlays/prod: app path does not exist`. Health showed Healthy only because nothing was deployed.

**How I found it:** The message names the path `overlays/prod`. ArgoCD reads the repo on GitHub, not my laptop folder, and I hadn't committed or pushed `overlays/prod` yet (git log showed the push only afterward as 90a6013).

**What fixed it:** `git add overlays/prod argocd/prod-app.yaml`, commit, push, then Refresh (Hard Refresh if needed) on `nginx-prod`. It went to OutOfSync; I clicked Sync and it became Synced + Healthy.

**What I'd do differently:** Push the overlay before applying the Application, and check `git status` and the repo on GitHub first.

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
