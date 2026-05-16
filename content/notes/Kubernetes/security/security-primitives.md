---
title: Security Primitives
date: 2026-05-10
weight: 1
tags: [Security]
---
# Security Primitives

Kubernetes is the primary platform for hosting production-grade applications, making security a top priority. Its security layers include cluster setup, network policies, authentication, authorization, and workload hardening, all working together to reduce risk in dynamic, containerized environments.

---

## Security Layers

Security in Kubernetes operates across multiple layers, often described as the 4C's of Cloud Native Security:

- **Cloud**: the outermost layer: infrastructure security, including firewalls, IAM, encryption at rest, and physical security.
- **Cluster**: Kubernetes-level security: API server access, RBAC, admission controllers, etcd encryption, and TLS between components.
- **Container**: image and runtime security: trusted base images, vulnerability scanning, non-root execution, and dropping unnecessary capabilities.
- **Code**: the innermost layer: secure coding practices, dependency management, proper secrets handling, and TLS for service communication.

Each layer builds on the one beneath it. If an outer layer is compromised, the inner layers cannot be trusted, regardless of how well they are secured.

---

## Securing the Hosts

Before addressing Kubernetes-specific security, the physical or virtual hosts forming the cluster must be hardened:

- Root access is disabled.
- Password-based authentication is replaced with SSH key-based authentication only.
- Infrastructure-level security measures are applied.

If the underlying hosts are compromised, everything running on top of them is compromised as well.

---

## Kubernetes API Server as the First Line of Defense

The `kube-apiserver` sits at the center of all operations in Kubernetes. Every interaction, whether via `kubectl` or direct API calls, passes through it, making API server access control the first and most critical line of defense.

### Authentication: Who Can Access the Cluster

Authentication determines which users and systems are authorized to access the API server. Kubernetes supports several mechanisms:

- Certificates for cryptographic identity verification.
- External identity providers, such as LDAP, for integration with existing organizational directories.
- Service accounts for programmatic access by machines and internal workloads.

### Authorization: What Can They Do

Once a user or system is authenticated, authorization determines what actions they are permitted to perform. The primary mechanism is Role-Based Access Control (RBAC), in which users are assigned to groups that grant specific permissions. Kubernetes also supports Attribute-Based Access Control (ABAC), node authorizers, and webhooks.

---

## TLS Encryption for Internal Communication

All communication between cluster components: `etcd`, `kube-controller-manager`, `kube-scheduler`, `kube-apiserver`, and `kubelet`, is secured with TLS encryption.

---

## Network Policies for Pod-to-Pod Communication

By default, all pods in a Kubernetes cluster can communicate freely with each other. Network Policies provide fine-grained control over which pods can communicate with which others, and are an essential step toward a properly isolated cluster architecture.

---
