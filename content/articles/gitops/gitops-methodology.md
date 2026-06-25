---
title: GitOps Methodology
date: 2026-06-24
draft: false
tags: ["GitOps", "ArgoCD", "FluxCD", "Kustomize"]
weight: 1
---

**GitOps** is a way to manage and automate infrastructure, applications, and operations by using **Git** as the **single source of truth**. It treats infrastructure configuration, application deployment, and operational procedures as code stored in Git repositories. Changes to systems are made by updating Git repositories, and those changes are then automatically applied to the live environment.

---

## What Is GitOps

> **GitOps is a set of best practices in which the entire code-delivery process is controlled via Git, including infrastructure and application configurations as code, with automation handling updates or rollbacks.**

The main idea is to use a Git repository to manage the system's desired state. Instead of executing commands directly against a cluster, such as running `kubectl apply` by hand, you modify configuration files in Git. Changes go through pull requests, are reviewed, and once merged, they automatically trigger system updates. You don't need direct access to the cluster; Git access is enough.

All configurations are written as files, typically in YAML, that describe how the infrastructure and applications should look. This covers both infrastructure components, such as monitoring tools and ingress controllers, and application details, such as deployments and services. These files are separate from the actual application source code, which lives in its own repository. The GitOps repository specifies which versions of those applications to run, not how to build them.

The automation is handled by a controller such as Flux CD or Argo CD. It constantly compares the desired state in Git with the current state of the production environment and automatically applies any differences it finds. If something goes wrong, you revert the Git commit, and the controller restores the cluster to its previous state.

---

## How GitOps Works

```
Developer                Git Repository      kubernetes Cluster
   │                           │                            │
   │──── edit & push YAML ────>│                            │
   │──── open pull request ───>│                            │
   │────── merge & push ──────>│                            │
   │                           │<───── reconcile loop ──────│
   │                           │────── apply manifest ─────>│
   │                           │                            │
   │                           │<───── drift detected ──────│
   │                           │──── re-apply manifest ────>│
   │                           │                            │
   │<-- alert (on failure) ----│                            │
```

Every change starts as a pull request. Once merged, the GitOps operator picks it up and applies it to the cluster. The operator also runs continuously: if someone makes a manual change on the cluster that deviates from what Git declares, the operator detects the drift and corrects it. If the operator can’t apply a change, it sends an alert so the issue can be investigated.

---

## The Reconciliation Loop

The GitOps operator runs inside the cluster and reconciles the repository on a continuous loop. When it detects a change, such as an image tag bumped from `1.0.0` to `1.1.0`, it pulls the updated manifest, applies it to the cluster, waits for the rollout to complete, and records the result.

```
┌────────────────────┐       watches         ┌──────────────────────────┐
│ Kubernetes Cluster │ ────────────────────> │     Git Repository       │
│  (with Flux CD)    │ <──────────────────── │  YAML / Kustomize / Helm │
└────────────────────┘   applies manifests   └──────────────────────────┘
```

No `kubectl` commands or cluster credentials need to leave the cluster. All a teammate needs is Git access.

---

## Why This Matters

1. **Collaboration & Audit Trail**  
   Pull requests become change requests. Every tweak, storage size, replica count, and new ConfigMap has a commit, author, and timestamp.
2. **Security**  
   Developers don't need direct access to the Kubernetes API server. Access control is as simple as Git repository permissions.
3. **Automation**  
   Tools like Renovate open pull requests when container images publish new tags. Merge the PR, and the operator updates the running workload; no extra CI/CD pipeline glue needed.
4. **Easy Rollbacks**  
   If a change is incorrect, revert the commit. The operator detects the revert and automatically restores the cluster to its previous state.

---

## Flux CD vs. Argo CD

Both controllers are production-ready and widely adopted. Flux is CLI-first with a minimal UI and relies on Kustomize, a declarative configuration tool for Kubernetes manifests. This keeps the workflow Git-centric and closer to the underlying mechanics. Argo CD has a full-featured UI that works well in team settings and supports Helm, Kustomize, and plain manifests, with Helm being the more common choice in practice.

|                       | Flux CD                            | Argo CD                                       |
|-----------------------|------------------------------------|-----------------------------------------------|
| **Interface**         | CLI-first; minimal UI              | Full-featured UI, well-suited for teams       |
| **Cloud integration** | Native to Azure AKS GitOps feature | Works everywhere                              |
| **Templating**        | Kustomize-oriented                 | Supports Helm, Kustomize, and plain manifests |

Both are worth learning. The concepts transfer directly between the two, so starting with either is a reasonable choice depending on your team's preferences and tooling.

---

## The Workflow in Action

You edit a manifest, commit, and push to the GitOps repository:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  #...
spec:
  replicas: 2
  selector:
    #...
  template:
    #...
    spec:
      #...
      containers:
      - image: nginx:1.1.0   # bumped tag
        #...
```

The GitOps operator detects the new commit, pulls the repository, and triggers a rollout. You can observe it directly:

```bash
kubectl get pods -n linkding
```

The Deployment creates a new ReplicaSet with `:1.36`. If something goes wrong, a revert is all it takes:

```bash
git revert <commit-sha>
git push
```

The operator reconciles again, and the workload is downgraded automatically.

---

## Best Practices

- **Everything Declarative**: Describe the desired state in manifest files. Imperative commands like `kubectl scale` have no place in a GitOps workflow.
- **Single Source of Truth**: The cluster pulls from Git; nothing external pushes manifests directly to it.
- **Automated PRs & Image Scans**: Renovate, Dependabot, or Flux's built-in image reflector keep workloads current without manual tracking.
- **Alerts on Divergence**: Wire operator events to Slack or email so failed syncs never go unnoticed.

---

## GitOps vs. Traditional DevOps

| Aspect           | Traditional DevOps                         | GitOps                           |
|------------------|--------------------------------------------|----------------------------------|
| Deployment       | Scripts and CI/CD pipelines push changes   | Pull-based from Git repositories |
| Configuration    | Often scattered across tools and pipelines | Centralized in Git               |
| Auditing         | Logs and monitoring systems                | Git commit history               |
| Rollback         | Often manual                               | Git revert and auto-sync         |
| Drift management | Typically reactive                         | Continuous reconciliation        |

---