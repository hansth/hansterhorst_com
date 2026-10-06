---
title: CV Hans ter Horst
Date: 2025-10-01
Draft: false
---

_[English version](/cv/)_

Ik ben Kubernetes-engineer met 4 jaar ervaring als full-stack developer. Ik heb de CKA, CKAD, KCSA en KCNA behaald en bereid me voor op de CKS om Kubestronaut te worden. Ik begon in 2025 met Kubernetes op één K3s-cluster. Sindsdien heb ik een homelab-cluster gebouwd met drie control-plane nodes en één worker node, en werk ik aan praktijkprojecten binnen het DevOps-programma van KubeCraft, met feedback van senior engineers.

Mijn achtergrond in Java en Angular geeft me inzicht in hoe applicaties van binnenuit werken en wat het betekent om developer te zijn. Dat maakt me een betere engineer en helpt me de beste oplossingen te vinden voor het deployen van applicaties op Kubernetes.

Ik zoek een rol als Kubernetes- of Platform Engineer waarin ik productie-ervaring kan opdoen en mijn hands-on kennis kan inzetten om jullie team te versterken.

---

## Persoonlijke gegevens

* **Naam**: Hans ter Horst
* **E-mail**: me@hansterhorst.com
* **Website**: [www.hansterhorst.com](http://www.hansterhorst.com/)
* **LinkedIn**: [linkedin.com/in/hansth](https://www.linkedin.com/in/hansth/)

---

## Certificaten

- [CKA](https://www.credly.com/badges/42b48583-e987-4d04-830c-37731b747bbb) — Certified Kubernetes Administrator
- [CKAD](https://www.credly.com/badges/dc72b82d-f473-491a-aba3-f879caa2485b) — Certified Kubernetes Application Developer
- [KCSA](https://www.credly.com/badges/9c20b140-72de-4593-af48-025689f22a24) — Kubernetes and Cloud Native Security Associate
- [KCNA](https://www.credly.com/badges/1d2b6127-d8d5-4b19-9839-b263337bc744) — Kubernetes and Cloud Native Associate
- [DevOps Practitioner](https://www.credly.com/badges/caa2887e-d3e3-40bc-8084-fba7920fa4f8) — TWN DevOps Bootcamp
- [AWS CCP](https://www.credly.com/badges/3e1d3967-0f7e-4a08-be25-f791fc13ab80) — AWS Certified Cloud Practitioner
- CKS — gepland eind 2026 _(Kubestronaut)_

---

## Projecten

**[DevOps CI/CD Pipeline](https://github.com/hansth/k8s-gitops)**

- **OpenAPI:** API-first ontwerp dat de Java-interface en de TypeScript-types genereert
- **Quarkus:** Backend REST API op Java 25 met Hibernate en Flyway
- **React:** Frontend in React 19 en TypeScript achter Nginx
- **Devcontainer + mise:** Reproduceerbare ontwikkelomgeving met vastgepinde toolversies, werkt op Linux en macOS
- **GitHub Actions:** Test- en Docker-workflows per component, met een coverage-eis van 80% en rapportage op elke pull request
- **Trivy:** Security-scan van de Docker-images op HIGH- en CRITICAL-kwetsbaarheden
- **Release Please:** Automatische versies, tags en CHANGELOGs op basis van Conventional Commits
- **GHCR:** Publiceert de container-images bij elke release-tag
- **k3d + Kustomize:** Lokale dev- en prod-clusters met gedeelde basismanifests en een overlay per omgeving
- **CloudNativePG:** Draait PostgreSQL in de cluster via een Kubernetes-operator
- **pre-commit:** Git hooks voor Commitizen, google-java-format, Checkstyle, oxlint en oxfmt

**[Multi-Tenant AKS Platform](https://gitlab.com/HansTH/k8s-azure)**

- **Terraform:** Provisioneert de AKS-cluster, Key Vault en Blob Storage, en genereert de GitOps-manifests per klant
- **FluxCD:** GitOps deployments zonder handmatige interventie, sync in zes fasen op afhankelijkheidsvolgorde
- **Kustomize:** Voor per-klant overlays op een gedeelde set basismanifests
- **Azure Key Vault:** Gecentraliseerde secrets, in de cluster gesynchroniseerd via de Secrets Store CSI-driver
- **CloudNativePG:** Beheert PostgreSQL per klant; back-up naar Blob Storage via de Barman Cloud-plugin
- **Traefik:** Ingress controller die extern verkeer naar de cluster routeert
- **cert-manager:** Voor automatische TLS-certificaten via Let's Encrypt

**[Rancher ArgoCD Pipeline](https://gitlab.com/HansTH/rancher-argocd)**

- **Rancher:** Voor multi-cluster management, wat clusterbeheer en upgrades vereenvoudigt
- **ArgoCD:** GitOps deployments zonder handmatige interventie
- **Kustomize:** Voor omgeving-specifieke aanpassingen van Kubernetes-manifests
- **KSOPS:** Kustomize-plugin voor SOPS/age secret encryption in ArgoCD
- **Traefik IngressRoute:** HTTPS-routing naar interne services op de cluster
- **cert-manager:** Voor automatische TLS-certificaten via Let's Encrypt en Cloudflare DNS-validatie
- **Terraform:** Provisioneert twee DigitalOcean Droplets (VM's)
- **Ansible:** Configureert de Droplets met k3s, cert-manager, Rancher en ArgoCD

**[Kubernetes HomeLab](https://gitlab.com/homelab7894101/k8s-homelab)**

- **kubeadm:** Voor het opzetten van een HA-cluster met drie control planes en één worker node
- **Cilium CNI:** Als kube-proxy vervanging, inclusief Cilium Gateway API
- **Longhorn:** Voor gedistribueerde block storage met replicatie over meerdere nodes
- **Flux:** GitOps deployments zonder handmatige interventie
- **Kustomize:** Voor omgeving-specifieke aanpassingen van Kubernetes-manifests
- **SOPS + age:** Voor het beheer van versleutelde secrets in GitOps-workflows
- **Renovate:** CronJob voor geautomatiseerde container image-updates via Git
- **Prometheus + Grafana:** Voor monitoring en observability van cluster en workloads
- **HAProxy + UniFi:** Voor load balancing van de API-server en VLAN-segmentatie van het netwerk

**[Jenkins CI/CD Pipeline](https://gitlab.com/devops8614042)**

- **Jenkins:** CI/CD-pipelines met versiebeheer en automatische commits naar Git
- **Podman:** Voor het bouwen van container-images vanuit Jenkins zonder Docker-daemon
- **Sonatype Nexus:** Mijn private registry voor centrale opslag en distributie van container-images
- **Traefik IngressRoute:** HTTPS-routing naar interne services op de cluster
- **cert-manager:** Voor automatische TLS-certificaten via Let's Encrypt en Cloudflare DNS-validatie

---

## Werkervaring

**Kubernetes Engineer (internship) — KubeCraft** | _2026 – heden_

- CI/CD-pipeline die applicatieontwikkeling van een devcontainer naar een Kubernetes-cluster brengt: GitHub Actions met een coverage-eis van 80%, Trivy-imagescans, releases naar GHCR met Release Please, en k3d dev- en prod-clusters
- Multi-tenant SaaS-platform op Azure AKS: Terraform provisioneert de cluster, Key Vault en Blob Storage, en FluxCD deployt per klant een eigen n8n-instantie
- Multi-cluster management met Rancher en ArgoCD op DigitalOcean, geprovisioneerd met Terraform en geconfigureerd met Ansible
- HA homelab-cluster gebouwd met kubeadm (drie control planes, één worker), met Cilium, Longhorn, FluxCD en Prometheus/Grafana

**Full-Stack Developer — Twict** | _2022 – 2026_

- i18n-vertaalfunctionaliteit geïmplementeerd voor internationale gebruikers
- Kosteneffectieve videostreaming-oplossing gebouwd voor een educatief platform
- Herbruikbare Angular-component voor internationale telefoonvalidatie ontwikkeld
- Bijgedragen aan codekwaliteit via refactoring en optimalisatie

**Procesoperator — Diverse bedrijven** | _2011 – 2022_

- Processen gemonitord en geoptimaliseerd, met focus op kwaliteit en betrouwbaarheid
- Sterke ervaring in teamwerk, probleemoplossing en werken onder druk

**Matchmove Artist — Escape Studios, Londen** | _2007_

- Gewerkt aan Walt Disney's The Chronicles of Narnia: Prince Caspian
- Verantwoordelijk voor het tracken van filmopnamen voor nauwkeurige VFX-integratie

**Grafische Industrie — Diverse bedrijven** | _1990 – 2011_

- Meer dan 20 jaar in diverse rollen, gespecialiseerd in drukproductie, opmaak en prepress
- Als shift-leader mensen begeleid en opgeleid en de planning over drie drukpersen beheerd

---

## Opleiding

**TWN DevOps Bootcamp** | **Online** | _2025_ Automatisering, CI/CD, cloudplatformen, Kubernetes en moderne DevOps-workflows.

**Full Stack Developer — Hogeschool Novi** | _2021 – 2022_ HBO-bootcamp in full-stack webontwikkeling, Java en React.

**Vapro B – Procesoperator — ROC Twente** | _2012 – 2015_ Beroepsopleiding procestechnologie en operaties.

**Maya Professional for VFX — Escape Studios, Londen** | _2007_ Opleiding in 3D-modellering, animatie en visuele effecten.

---

## Technische vaardigheden

- **Kubernetes & cloud-native:** Kubernetes, K3s, k3d, Rancher, Flux, ArgoCD, Helm, Kustomize, CloudNativePG, Prometheus, Grafana
- **DevOps & CI/CD:** Jenkins, GitHub Actions, GitLab CI/CD, Docker, Podman, Renovate, Release Please, Trivy
- **Infrastructure as Code:** Terraform, Ansible
- **Cloud:** Azure AKS, DigitalOcean, AWS
- **Registries & artifacts:** Sonatype Nexus, GHCR
- **Development:** Java, Spring Boot, Quarkus, Angular, React, TypeScript, JavaScript
- **Infrastructuur:** Linux, HAProxy, UniFi

---

## Naast techniek

Als ik niet aan het programmeren ben of met DevOps-tools experimenteer, ben ik buiten te vinden:

- Op de trails met mijn **mountainbike**
- Op **bikepacking-avonturen**
- Luisterend naar **muziek** en kijkend naar **films**
