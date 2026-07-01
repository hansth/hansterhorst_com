---
title: DevOps
---

This was my first real end-to-end project as a DevOps engineer — a series of hands-on portfolio pieces built after completing the [**TWN DevOps Bootcamp**](https://www.techworld-with-nana.com/devops-bootcamp). Starting from a blank VPS and ending with a working CI/CD pipeline, it covers the full stack I built and learned from scratch.

## Architecture

```
           ┌────────────────────────────────┐
           │            Ubuntu VPS          │
           │ UFW · Fail2ban · SSH Hardening │
           └───────────────┬────────────────┘
                           │
                           ▼
 ┌────────────────────────────────────────────────────┐
 │                    K3s Cluster                     │
 │                                                    │
 │ ┌────────────────────────────────────────────────┐ │
 │ │ Traefik Ingress · cert-manager · Let’s Encrypt │ │
 │ └───────┬────────────────────────────────┬───────┘ │
 │         │                                │         │
 │   jenkins.domain                    nexus.domain   │
 │         │                                │         │
 │         ▼                                ▼         │
 │ ┌───────────────┐               ┌────────────────┐ │
 │ │    Jenkins    │  push image   │      Nexus     │ │
 │ │ Podman Builds │──────────────>│ Image Registry │ │
 │ │               │               │                │ │
 │ └───────┬───────┘               └───────┬────────┘ │
 └─────────│───────────────────────────────│──────────┘
           │                               │
           ▼                               ▼
        ┌─────┐                         ┌─────┐
        │ Git │                         │ K8S │
        └─────┘                         └─────┘
```

## Projects
