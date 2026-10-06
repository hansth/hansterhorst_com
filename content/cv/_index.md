---
title: CV Hans ter Horst
Date: 2025-10-01
Draft: false
---

_[Nederlandse versie](/cv-nl/)_

I’m a Kubernetes engineer with 4 years of full-stack development experience. I hold the CKA, CKAD, KCSA, and KCNA, and I’m preparing for the CKS to become a Kubestronaut. I started with Kubernetes in 2025 on a single K3s cluster. Since then, I’ve built a homelab cluster with three control-plane nodes and one worker node, and I’ve taken on real-world projects through KubeCraft’s DevOps program, getting feedback from senior engineers.

My background in Java and Angular helps me understand how applications work from the inside and what it means to be a developer. This makes me a better engineer and helps me find the best solutions for deploying applications on Kubernetes.

I’m looking for a Kubernetes or Platform Engineer role where I can build production experience and use my hands-on skills to contribute to your team.

---

## Personal Information

* **Name**: Hans ter Horst
* **Email**: me@hansterhorst.com
* **Website**: [www.hansterhorst.com](http://www.hansterhorst.com/)
* **LinkedIn**: [linkedin.com/in/hansth](https://www.linkedin.com/in/hansth/)

---

## Certifications

- [CKA](https://www.credly.com/badges/42b48583-e987-4d04-830c-37731b747bbb) — Certified Kubernetes Administrator
- [CKAD](https://www.credly.com/badges/dc72b82d-f473-491a-aba3-f879caa2485b) — Certified Kubernetes Application Developer
- [KCSA](https://www.credly.com/badges/9c20b140-72de-4593-af48-025689f22a24) — Kubernetes and Cloud Native Security Associate
- [KCNA](https://www.credly.com/badges/1d2b6127-d8d5-4b19-9839-b263337bc744) — Kubernetes and Cloud Native Associate
- [DevOps Practitioner](https://www.credly.com/badges/caa2887e-d3e3-40bc-8084-fba7920fa4f8) — TWN DevOps Bootcamp
- [AWS CCP](https://www.credly.com/badges/3e1d3967-0f7e-4a08-be25-f791fc13ab80) — AWS Certified Cloud Practitioner
- CKS — planned end of 2026 _(Kubestronaut)_

---

## Projects

**[DevOps CI/CD Pipeline](https://github.com/hansth/k8s-gitops)**

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

**[Multi-Tenant AKS Platform](https://gitlab.com/HansTH/k8s-azure)**

- **Terraform:** Provisions the AKS cluster, Key Vault, and Blob Storage, and generates each customer's GitOps manifests
- **FluxCD:** GitOps deployments without manual intervention, six-stage dependency-ordered sync
- **Kustomize:** For per-customer overlays on a shared base manifest set
- **Azure Key Vault:** Centralized secrets, synced into the cluster via the Secrets Store CSI driver
- **CloudNativePG:** Manages PostgreSQL per customer; Barman Cloud plugin backup to Blob Storage
- **Traefik:** Ingress controller routing external traffic into the cluster
- **cert-manager:** For automatic TLS certificates via Let's Encrypt

**[Rancher ArgoCD Pipeline](https://gitlab.com/HansTH/rancher-argocd)**

- **Rancher:** For multi-cluster management, simplifying cluster administration and upgrades
- **ArgoCD:** GitOps deployments without manual intervention
- **Kustomize:** For environment-specific adjustments to Kubernetes manifests
- **KSOPS:** Kustomize plugin for SOPS/age secret encryption in ArgoCD
- **Traefik IngressRoute:** HTTPS routing to internal services on the cluster
- **cert-manager:** For automatic TLS certificates via Let's Encrypt and Cloudflare DNS validation
- **Terraform:** Provisions two DigitalOcean Droplets (VMs)
- **Ansible:** Configures the Droplets with k3s, cert-manager, Rancher, and ArgoCD

**[Kubernetes HomeLab](https://gitlab.com/homelab7894101/k8s-homelab)**

- **kubeadm:** For setting up an HA cluster with three control planes and one worker node
- **Cilium CNI:** As a kube-proxy replacement, including Cilium Gateway API
- **Longhorn:** For distributed block storage with replication across multiple nodes
- **Flux:** GitOps deployments without manual intervention
- **Kustomize:** For environment-specific adjustments to Kubernetes manifests
- **SOPS + age:** For managing encrypted secrets in GitOps workflows
- **Renovate:** CronJob for automated container image updates via Git
- **Prometheus + Grafana:** For monitoring and observability of cluster and workloads
- **HAProxy + UniFi:** For API server load balancing and network VLAN segmentation

**[Jenkins CI/CD Pipeline](https://gitlab.com/devops8614042)**

- **Jenkins:** CI/CD pipelines with version control and automatic commits to Git
- **Podman:** For building container images from Jenkins without a Docker daemon
- **Sonatype Nexus:** My private registry for central storage and distribution of container images
- **Traefik IngressRoute:** HTTPS routing to internal services on the cluster
- **cert-manager:** For automatic TLS certificates via Let's Encrypt and Cloudflare DNS validation

---

## Work Experience

**Kubernetes Engineer (Internship) — KubeCraft** | _2026 – present_

- CI/CD pipeline that takes application development from a devcontainer to a Kubernetes cluster: GitHub Actions with an 80% coverage gate, Trivy image scans, Release Please releases to GHCR, and k3d dev and prod clusters
- Multi-tenant SaaS platform on Azure AKS: Terraform provisions the cluster, Key Vault, and Blob Storage, and FluxCD deploys an n8n instance per customer
- Multi-cluster management with Rancher and ArgoCD on DigitalOcean, provisioned with Terraform and configured with Ansible
- HA homelab cluster built with kubeadm (three control planes, one worker), running Cilium, Longhorn, FluxCD, and Prometheus/Grafana

**Full-Stack Developer — Twict** | _2022 – 2026_

- Implemented i18n translation functionality for international users
- Built a cost-effective video streaming solution for an educational platform
- Developed a reusable Angular component for international phone number validation
- Contributed to code quality improvements through refactoring and optimization

**Process Operator — Various companies** | _2011 – 2022_

- Monitored and optimized processes with a focus on quality control and operational reliability
- Strong experience in teamwork, problem-solving, and working under pressure

**Matchmove Artist — Escape Studios, London** | _2007_

- Worked on Walt Disney's The Chronicles of Narnia: Prince Caspian
- Responsible for tracking footage for accurate VFX integration

**Graphic Industry — Various companies** | _1990 – 2011_

- Over 20 years in various roles, specializing in print production, layout, and prepress
- As a shift-leader, I mentored and trained staff and managed the planning across three printing presses.

---

## Education

**TWN DevOps Bootcamp** | **Online** | _2025_ Automation, CI/CD, cloud platforms, Kubernetes, and modern DevOps workflows.

**Full Stack Developer — Hogeschool Novi** | _2021 – 2022_ Higher education bootcamp in full-stack web development, Java, and React.

**Vapro B – Process Operator — ROC Twente** | _2012 – 2015_ Vocational training in process technology and operations.

**Maya Professional for VFX — Escape Studios, London** | _2007_ Training in 3D modeling, animation, and visual effects.

---

## Technical Skills

- **Kubernetes & cloud-native:** Kubernetes, K3s, k3d, Rancher, Flux, ArgoCD, Helm, Kustomize, CloudNativePG, Prometheus, Grafana
- **DevOps & CI/CD:** Jenkins, GitHub Actions, GitLab CI/CD, Docker, Podman, Renovate, Release Please, Trivy
- **Infrastructure as Code:** Terraform, Ansible
- **Cloud:** Azure AKS, DigitalOcean, AWS
- **Registries & Artifacts:** Sonatype Nexus, GHCR
- **Development:** Java, Spring Boot, Quarkus, Angular, React, TypeScript, JavaScript
- **Infrastructure:** Linux, HAProxy, UniFi

---

## Beyond Tech

When I’m not coding or experimenting with DevOps tools, you’ll find me outdoors:

* Exploring trails on my **mountain bike**
* Going on **bikepacking adventures**
* Listening to **music** and watching **movies**
