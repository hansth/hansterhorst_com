---
title: Projects
---

A collection of hands-on Kubernetes projects: CI/CD pipelines, GitOps deployments, and my Homelab. I share what I build, the problems I run into, and what I learn along the way.

---

## [Rancher ArgoCD Pipeline](https://gitlab.com/HansTH/rancher-argocd)

In this project, [Rancher](https://www.rancher.com/) provides a centralized control plane for managing multiple Kubernetes clusters. It enables registering, monitoring health, and upgrading clusters without requiring SSH access to each node. [ArgoCD](https://argo-cd.readthedocs.io/) enhances this setup by managing application deployments through a GitOps repository that serves as the single source of truth. Any changes pushed to this repository are automatically reflected in the registered clusters.

- **Rancher:** For multi-cluster management, simplifying cluster administration and upgrades
- **ArgoCD:** GitOps deployments without manual intervention
- **Kustomize:** For environment-specific adjustments to Kubernetes manifests
- **KSOPS:** Kustomize plugin for SOPS/age secret encryption in ArgoCD
- **Traefik IngressRoute:** HTTPS routing to internal services on the cluster
- **cert-manager:** For automatic TLS certificates via Let's Encrypt and Cloudflare DNS validation

---

## [Kubernetes HomeLab](https://gitlab.com/homelab7894101/k8s-homelab)

My Homelab projects to learn Kubernetes concepts through practical, hands-on tasks. The goal is to understand Kubernetes concepts by deploying and managing real applications on my multi-node cluster.

- **kubeadm:** For setting up an HA cluster with three control planes and one worker node
- **Cilium CNI:** As a kube-proxy replacement, including Cilium Gateway API
- **Longhorn:** For distributed block storage with replication across multiple nodes
- **Flux:** GitOps deployments without manual intervention
- **Kustomize:** For environment-specific adjustments to Kubernetes manifests
- **SOPS + age:** For managing encrypted secrets in GitOps workflows
- **Renovate:** CronJob for automated container image updates via Git
- **Prometheus + Grafana:** For monitoring and observability of cluster and workloads
- **HAProxy + UniFi:** For API server load balancing and network VLAN segmentation

---

## [Jenkins CI/CD Pipeline](https://gitlab.com/devops8614042)

This project groups several related projects focused on building tools, with the main project of setting up a K3s cluster with a fully automated CI/CD pipeline using Jenkins, Podman, and Sonatype Nexus.

- **Jenkins:** CI/CD pipelines with version control and automatic commits to Git
- **Podman:** For building container images from Jenkins without a Docker daemon
- **Sonatype Nexus:** My private registry for central storage and distribution of container images
- **Traefik IngressRoute:** HTTPS routing to internal services on the cluster
- **cert-manager:** For automatic TLS certificates via Let's Encrypt and Cloudflare DNS validation
