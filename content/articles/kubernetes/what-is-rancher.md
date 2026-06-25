---
title: What is Rancher
date: 2026-05-12
draft: false
tags: ["Rancher", "Kubernetes"]
weight: 2
---

Rancher is an open-source Kubernetes management platform that simplifies deploying, managing, and securing Kubernetes clusters on-premises, in the cloud, or at the edge through a single web UI.

## Why Use Rancher?

Rancher's biggest advantage is its flexibility. You're not restricted to one provider; you can run Kubernetes on your hardware, in a data center, on a Raspberry Pi, or across clouds. Rancher offers full freedom. Here are the key benefits.

**Centralized multi-cluster management.** Managing multiple Kubernetes clusters across cloud, on-prem, or hybrid environments is complex, involving logins, workload tracking, and updates. Rancher provides a single control plane to manage all clusters, regardless of location, eliminating the need to switch between AWS, Azure, Google Cloud, and on-prem environments.

**Simplified security and access control.** Rancher centralizes user management, permissions, and access controls across clusters. It integrates with authentication providers like Active Directory, GitHub, LDAP, and SAML, enabling users to log in with existing credentials. Rancher also offers built-in Role-Based Access Control (RBAC) to define access at the cluster, workload, and namespace levels.

**Built-in monitoring and alerting.** Rancher simplifies managing Kubernetes at scale by including monitoring and alerting with Prometheus and Grafana. It provides real-time insights into cluster performance, helping you identify issues early.

**GitOps-powered deployment with Fleet.** Manually deploying applications across Kubernetes clusters is slow and error-prone. Rancher’s Fleet, its built-in GitOps system, allows defining deployment across clusters. Changes in your Git repo deploy automatically, enabling faster rollouts, fewer errors, and full automation.

**Cost control.** Using AWS EKS, Azure AKS, or Google GKE often incurs hidden costs such as data egress fees, loadbalancer costs, and per-node pricing. Rancher allows you to run your own clusters on bare metal, VMs, or private data centers without sacrificing performance, giving you control and reducing costs.

## The local cluster: Rancher manages itself

Under the hood, Rancher always runs on its own Kubernetes cluster, known as the **local cluster**. This works as follows:

- Rancher manages itself via this local cluster as a Kubernetes-native application running as a set of Pods.
- The `rancher-webhook` is a Pod that runs inside this local cluster.
- From this position, Rancher can import and manage external clusters, such as a separate K3s VM.

What the local cluster actually is depends on how Rancher was installed:

- In a **single-container setup**, the Rancher Docker container runs a built-in K3s instance, which becomes the local cluster.
- In a **production install**, Rancher is deployed via Helm on an existing K3s, RKE2, or RKE cluster, which becomes the local cluster. There's no K3s "inside" a container.

In both cases, Rancher is a Kubernetes-native application that manages other Kubernetes clusters from within its own cluster.

## The architecture: Rancher as the control plane

Rancher serves as the central control plane for all clusters you add to it. Schematically, it looks like this:

```
Rancher (control plane)
    └── local cluster (the cluster Rancher itself runs on)
    └── k3s-node1 (local VM)
    └── ec2-cluster (e.g. AWS)
    └── aws-eks-cluster (production)
```

From this single control plane, you get access to a wide range of management capabilities:

- Manage and monitor multiple clusters
- Run upgrades across all clusters
- Centrally manage RBAC (user permissions)
- Deploy applications through the UI

This is exactly the model used by companies like Shock Media to manage client-specific clusters from a single Rancher instance.

## Rancher works through the Kubernetes API

An important difference from cluster management is that Rancher doesn't use `kubeadm` commands. `kubeadm` is a one-time setup tool for bootstrapping a cluster. Rancher communicates continuously through the Kubernetes API to manage multiple clusters.

This difference shows up in three places:

**Importing existing clusters** (such as a separate K3s VM):

1. You install a Rancher agent on the cluster.
2. This agent establishes a connection back to Rancher.
3. From that point on, Rancher communicates with the cluster via the Kubernetes API.

**Creating new clusters** (for example, on AWS):

1. Rancher uses cloud provider APIs (such as AWS EC2).
2. Rancher provisions the nodes itself.
3. RKE2, Rancher's own Kubernetes distribution, is installed on those nodes.

**Upgrades:**

- Rancher sends API calls to the nodes to update Kubernetes components.
- This happens through Rancher's own upgrade controller, not `kubeadm`.

`kubeadm` is a one-time setup tool, while Rancher is a management control plane that keeps communicating with clusters via the API.

## What Rancher adds on top of Kubernetes

Rancher can provision Kubernetes from a hosted provider, provision compute nodes itself and install Kubernetes on them, or import existing Kubernetes clusters running anywhere.

On top of that, Rancher adds significant value in several areas:

- **Centralized authentication and RBAC**: Administrators control access across all clusters from a single place.
- **Monitoring and alerting**: Detailed visibility into clusters and their resources.
- **Logging**: Ships logs to external providers.
- **Application Catalog**: Direct integration with Helm.
- **CI/CD**: integrates with external CI/CD systems, such as ArgoCD, or uses the built-in Fleet to automatically deploy and upgrade workloads.

Altogether, this makes Rancher a complete management platform for Kubernetes, with tools to manage and run Kubernetes anywhere.

---
http://hansterhorst.com/articles/gitops/what-is-argocd/