# CLAUDE.md

I'm learning ArgoCD. This repo is my playground.

## Rules

- Guide me and explain. Don't create or edit files or run commands unless I ask.
- Review what I write and point out problems rather than fixing them.
- Ask me to paste exact errors, and tell me where to look (which pod, log, UI panel, or command output).
- Use my notes template for anything that broke:

```markdown
## <short title>

**What broke:**

**How I found it:**

**What fixed it:**

**What I'd do differently:**
```

## Project context

- Repo name: `argocd-playground`
- Layout:
  - `base/`
  - `overlays/dev/`
  - `overlays/prod/`
  - `argocd/`
- Sync policy: dev is auto-sync, prod is manual sync.

## The 8-step order

<!-- TODO: paste the 8 steps from my last message, each with its "done when". -->

## Schedule note

Offered recruiter calls (12-5 PM) overlap my 1:30-4:30 deep-work block, so pick one fix.

## Parked

EKS (Terraform-managed, separate repo/folder) is parked until I've learned ArgoCD on a local kind cluster. Keep manifests free of kind-only assumptions so the move is easy.

The Switchyard-on-Kubernetes plan is parked for Week 2. The repo to start from is `muksyed1/switchyard-llm-routing-aws`.
