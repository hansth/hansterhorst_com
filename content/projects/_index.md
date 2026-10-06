---
title: Projects
---

A collection of hands-on Kubernetes projects: CI/CD pipelines, GitOps deployments, and my homelab. I share what I build, the problems I run into, and what I learn along the way.

---

## [DevOps CI/CD Pipeline](https://github.com/hansth/k8s-gitops)

This project is a CI/CD pipeline for a todo application that shows how code moves from a devcontainer to a Kubernetes cluster. An OpenAPI spec generates a Java interface for the Quarkus backend and TypeScript types for the React frontend. Every pull request is tested, checked for 80% coverage, built into a Docker image, and scanned with Trivy. Release Please creates versions, tags, and CHANGELOGs from Conventional Commits, with each tag publishing images to GHCR. Changes are first tested in a local dev cluster, pushed to GitHub, and then verified in a prod cluster after the release.

- **OpenAPI:** API-first design that generates the Java interface and the TypeScript types
- **Quarkus:** Backend REST API on Java 25 with Hibernate and Flyway
- **React:** Frontend in React 19 and TypeScript, served by Nginx
- **Devcontainer + mise:** Reproducible development environment with pinned tool versions, works on Linux and macOS
- **GitHub Actions:** Test and Docker workflows per component, with an 80% coverage gate and a report on every pull request
- **Trivy:** Security scan of the Docker images for HIGH and CRITICAL vulnerabilities
- **Release Please:** Automated versions, tags, and CHANGELOGs from Conventional Commits
- **GHCR:** Publishes the container images on every release tag
- **k3d + Kustomize:** Local dev and prod clusters with shared base manifests and an overlay per environment
- **CloudNativePG:** Runs PostgreSQL in the cluster through a Kubernetes operator
- **pre-commit:** Git hooks for Commitizen, google-java-format, Checkstyle, oxlint, and oxfmt

---

## [Multi-Tenant AKS Platform](https://gitlab.com/HansTH/k8s-azure)

This project is a multi-tenant SaaS platform on Azure AKS, provisioned with Terraform and deployed via a FluxCD GitOps pipeline. Each customer gets their own n8n instance. Terraform provisions the AKS cluster, Azure Key Vault, and Blob Storage, then bootstraps FluxCD. Flux then continuously reconciles what's declared in Git, with six Kustomizations applied in dependency order and polling every 5 minutes.

- **Terraform:** Provisions the AKS cluster, Key Vault, and Blob Storage, and generates each customer's GitOps manifests
- **FluxCD:** GitOps deployments without manual intervention, six-stage dependency-ordered sync
- **Kustomize:** For per-customer overlays on a shared base manifest set
- **Azure Key Vault:** Centralized secrets, synced into the cluster via the Secrets Store CSI driver
- **CloudNativePG:** Manages PostgreSQL per customer; Barman Cloud plugin backup to Blob Storage
- **Traefik:** Ingress controller routing external traffic into the cluster
- **cert-manager:** For automatic TLS certificates via Let's Encrypt

---

## [Rancher ArgoCD Pipeline](https://gitlab.com/HansTH/rancher-argocd)

In this project, [Rancher](https://www.rancher.com/) provides a centralized control plane for managing multiple Kubernetes clusters. It enables registering, monitoring health, and upgrading clusters without requiring SSH access to each node. [ArgoCD](https://argo-cd.readthedocs.io/) enhances this setup by managing application deployments through a GitOps repository that serves as the single source of truth. Any changes pushed to this repository are automatically reflected in the registered clusters. Terraform provisions the DigitalOcean Droplets and Ansible configures them.

- **Rancher:** For multi-cluster management, simplifying cluster administration and upgrades
- **ArgoCD:** GitOps deployments without manual intervention
- **Kustomize:** For environment-specific adjustments to Kubernetes manifests
- **KSOPS:** Kustomize plugin for SOPS/age secret encryption in ArgoCD
- **Traefik IngressRoute:** HTTPS routing to internal services on the cluster
- **cert-manager:** For automatic TLS certificates via Let's Encrypt and Cloudflare DNS validation
- **Terraform:** Provisions two DigitalOcean Droplets (VMs)
- **Ansible:** Configures the Droplets with k3s, cert-manager, Rancher, and ArgoCD

---

## [Kubernetes HomeLab](https://gitlab.com/homelab7894101/k8s-homelab)

My homelab projects to learn Kubernetes concepts through practical, hands-on tasks. The goal is to understand Kubernetes concepts by deploying and managing real applications on my multi-node cluster.

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

This project groups several related projects focused on building tools, with the main project being the setup of a K3s cluster with a fully automated CI/CD pipeline using Jenkins, Podman, and Sonatype Nexus.

- **Jenkins:** CI/CD pipelines with version control and automatic commits to Git
- **Podman:** For building container images from Jenkins without a Docker daemon
- **Sonatype Nexus:** My private registry for central storage and distribution of container images
- **Traefik IngressRoute:** HTTPS routing to internal services on the cluster
- **cert-manager:** For automatic TLS certificates via Let's Encrypt and Cloudflare DNS validation