---
title: What is ArgoCD
date: 2026-06-24
draft: false
tags: ["ArgoCD", "GitOps"]
weight: 2
---

This article discusses ArgoCD, a popular GitOps tool in DevOps. If unfamiliar with GitOps, learn about it first to better understand the rest of the article.

This is the first of two articles. Here, I'll cover what ArgoCD is, how it compares to traditional CI/CD tools, and the benefits GitOps brings to managing Kubernetes clusters. In the second article, I'll put this into practice with a hands-on project that deploys ArgoCD and sets up an automated, Kustomize-based CD pipeline for a real application.

## What is ArgoCD?

ArgoCD is a continuous delivery (CD) tool. To understand it, compare it with common tools like Jenkins or GitLab CI/CD. I'll explore whether ArgoCD is just another CD tool, what makes it unique, and whether it replaces Jenkins or GitLab CI/CD.

## CD Workflow Without ArgoCD

Suppose we have a microservices app in Kubernetes. When code changes, like new features or bugs, the CI pipeline (e.g., Jenkins) triggers automatically. It tests, builds the container image, and pushes it to a repository.

How does this new image get deployed to Kubernetes? The deployment YAML is updated with the new image tag and applied to the cluster. After pushing the image, Jenkins updates the YAML and uses `kubectl` to apply it.

This is how many projects are set up. However, there are several challenges with this approach.

First, you need to install and configure tools like `kubectl` or Helm on your build servers to access the Kubernetes cluster and make changes. You also need to set up cluster access for these tools, since `kubectl` requires credentials to connect. If you're using an AWS EKS cluster, you'll need to provide AWS credentials alongside Kubernetes credentials to Jenkins.

This is a security challenge because you're granting cluster credentials to external services. With 50 projects, each needs its own credentials to access specific resources. If you have 50 clusters, you'd need to configure this for each one.

The biggest challenge is visibility. After Jenkins deploys or configures Kubernetes, it lacks insight into the deployment status. Once `kubectl apply` runs, Jenkins doesn't know whether the application was created successfully, is healthy, or failed to start. Figuring that out requires additional follow-up steps.

The CD part of the pipeline can be made far more efficient when working with Kubernetes specifically, and that's exactly what ArgoCD was built for. Designed for Kubernetes and GitOps from the ground up, ArgoCD offers major advantages, as you'll see throughout this article.

## CD Workflow With ArgoCD

ArgoCD improves the CD process by reversing the flow. Instead of an external CI/CD tool pushing changes into the cluster, ArgoCD runs as an agent inside the cluster and uses a **pull** model: it pulls changes from Git and applies them itself.

Here's how the workflow operates once ArgoCD takes over from a traditional CD pipeline:

1. Install and configure ArgoCD within the cluster.
2. Point it at a designated Git repository and have it monitor that repository for updates.
3. When changes occur, ArgoCD automatically pulls and applies them to the cluster.

When developers commit code, the CI pipeline on Jenkins automatically builds, tests, and pushes the image, then updates the Kubernetes `deployment.yaml` manifest with the new image version.

> _**Note**_: Keep application source code and Kubernetes manifest files in separate repositories. This separation prevents changes to manifest files from triggering the entire CI pipeline.

The Jenkins CI pipeline updates `deployment.yaml` in a separate Git repository dedicated to Kubernetes manifests. When these manifests change, ArgoCD, monitoring the repo, detects the change, pulls it, and applies it to the cluster.

ArgoCD supports Kubernetes manifests in YAML, Helm charts, Kustomize, or other templates that produce Kubernetes YAML. The repository that ArgoCD tracks is called a **GitOps repository**.

Whether Jenkins updates the application configuration repository during an image version bump or DevOps engineers manually edit manifest files, ArgoCD automatically pulls and applies changes to the cluster.

This creates two pipelines:

- **CI pipeline**: Owned by developers and configured with tools such as Jenkins.
- **CD pipeline**: Owned by DevOps teams and configured with ArgoCD.

Together, this creates an automated, end-to-end **CI/CD pipeline** with clearly defined team responsibilities.

## Benefits of Using GitOps With ArgoCD

Now let's look at the benefits of using GitOps in your project, with ArgoCD as the CD pipeline.

### Git as the Single Source of Truth

The main benefit is that Kubernetes manifests are defined as code in a Git repository. Instead of individuals running `kubectl apply` or `helm install` from their own computers, everyone uses Git as the single interface for modifying the cluster.

An interesting scenario is when someone in the team runs a quick `kubectl apply` directly, instead of going through the usual flow of making code changes, committing, pushing, getting a review, and merging. What happens to that manual change once the Git configuration is synced again?

ArgoCD continuously compares the desired state defined in Git with the cluster's actual state. When a difference is detected, it syncs the cluster to match Git, restoring consistency. So if someone manually changes the cluster, ArgoCD detects the drift and reverts the change, syncing the cluster back to the state defined in Git. This keeps Git as the **single source of truth** and ensures full transparency of what's actually running in the cluster.

Some teams need time to adjust to this workflow, and engineers may occasionally need to make a quick manual change before the corresponding code update is ready. ArgoCD can be configured to avoid automatically overriding manual cluster changes and send an alert when a drift is detected that needs to be reconciled.

By replacing `kubectl apply` commands with Git as the interface, every cluster change becomes documented and **version-controlled**. This provides a full history and audit trail and enables collaboration, where configuration changes can be reviewed and merged just like code.

### Easy Rollback

Another benefit of managing configuration as version-controlled files is the ability to easily roll back. ArgoCD applies changes pulled from Git, so if a new version breaks, you can `git revert` to the previous commit, and ArgoCD will automatically reapply that previous state to the cluster.

This is especially useful at scale. If you have multiple clusters syncing from the same Git repository, you don't need to manually revert each one with `kubectl delete` or `helm uninstall`. You declare the previous working state in Git, and every cluster syncs to it.

### Cluster Disaster Recovery

Disaster recovery also becomes much easier with this setup. If a cluster crashes, you can create a new one, point it at the Git repository containing the full configuration, and ArgoCD will restore the exact same state, no manual steps required, because everything is defined as code.

These benefits — a single source of truth, easy rollback, and disaster recovery — are general GitOps principles, not unique to ArgoCD. Other GitOps tools offer similar advantages. That said, some benefits are specific to ArgoCD itself.

## Kubernetes Access Control With Git and ArgoCD

If you don't want every team member making direct changes to a production cluster, this Git-based workflow lets you set access rules at the repository level instead. For example, DevOps or operations team members, including junior engineers, can propose changes via pull requests, while only senior engineers review and merge them.

This lets you manage cluster permissions through Git without creating separate Kubernetes roles and users for every team member with different access levels. Because of the pull model, engineers only need access to the Git repository, not direct access to the cluster.

This benefit also extends to non-human users, such as CI/CD tools like Jenkins. Since ArgoCD runs inside the cluster and applies changes, Jenkins and other external tools don't need direct cluster access at all. Cluster credentials no longer need to exist outside the cluster, which significantly improves security across all your Kubernetes clusters.

## ArgoCD as a Kubernetes Extension

Another advantage of ArgoCD is that it's more than just a tool deployed in the cluster. Other tools, including Jenkins, can also be deployed in a cluster. ArgoCD goes further by acting as an extension of the Kubernetes API itself, leveraging existing Kubernetes capabilities such as `etcd` for data storage and Kubernetes controllers for comparing current and desired state, rather than building all of this functionality from scratch.

The main benefit of this approach is the visibility that tools like Jenkins simply don't have. ArgoCD provides real-time updates on application status, allowing users to watch configuration files apply, Pods being created, and whether the application is healthy. If issues occur, it indicates failures and potential rollbacks.

## How ArgoCD is Configured

To configure ArgoCD, you deploy it into Kubernetes just like Prometheus or any other tool. Since ArgoCD is purpose-built for Kubernetes, it extends the Kubernetes API with Custom Resource Definitions (CRDs), letting you configure it using native Kubernetes YAML.

The main component is the **Application** CRD, a YAML manifest that specifies which Git repository should be synced with which Kubernetes cluster. This can be any Git repository and any cluster, including the one ArgoCD itself runs in, or an external cluster that ArgoCD manages. You can define multiple Application resources for different microservices, and group related applications using the **AppProject** CRD.

## Multiple Clusters With ArgoCD

Suppose you have three cluster replicas across different environments, and one runs ArgoCD to deploy changes to all of them. This lets administrators manage a single ArgoCD instance that controls the other clusters, whether that's three clusters or hundreds spread across other environments.

Now consider different environments (development, staging, production), each with its own cluster replicas. Here, you run one ArgoCD instance per environment, all of which reference a single shared configuration repository. Changes should be tested in development first, then in staging, and finally in production, not deployed everywhere at once.

There are two common ways to handle this. One option is to use separate Git branches for development, staging, and production, though this isn't ideal. A better approach is to use **overlays with Kustomize**, maintaining a shared set of base YAML files and overriding specific parts for each environment.

With this setup, each environment's CI pipeline updates only its corresponding overlay — the development pipeline updates the development overlay, the staging pipeline updates the staging overlay, and so on — streamlining promotion across environments.

## Does ArgoCD Replace Jenkins or GitLab CI/CD?

A common question is whether ArgoCD replaces tools like Jenkins or GitLab CI/CD. The answer is no, because you still need a CI pipeline to build and test code. ArgoCD replaces only the **CD** part of the Kubernetes pipeline.

> _**Note**_: ArgoCD isn't the only GitOps CD tool for Kubernetes. Several alternatives exist, including **Flux CD**, which also follows GitOps principles.

---
## Summary

The Git repository represents the desired state, and the Kubernetes cluster represents the actual state. ArgoCD sits between them, continuously reconciling to keep them in sync and updating the actual state to match the desired state when they differ.

In the next article, I'll put all of this into practice, installing ArgoCD, structuring a GitOps monorepo with Kustomize, and deploying a real application (linkding) with a fully automated sync, self-heal, and prune pipeline.

---
