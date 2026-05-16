---
title: TLS Kubernetes
date: 2026-05-10
weight: 4
tags: [Security, TLS, Certificates]
---
# TLS Certificates in Kubernetes

All communication between nodes must be secure and encrypted. All interactions between services and their clients must also be secure. An administrator interacting with the cluster via `kubectl` or directly via the API must establish a secure TLS connection. The same applies to all communication between internal cluster components.

Two requirements follow from this:

- All services within the cluster must use server certificates to secure their endpoints.
- All clients must use client certificates to verify their identity when connecting to those services.

---

## Server Components

Server components expose HTTPS endpoints and require certificates to authenticate themselves to clients.

### kube-apiserver

The `kube-apiserver` exposes an HTTPS service used by other components and external users to manage the cluster. It is the central communication point for the entire cluster and requires a certificate and key pair: `apiserver.crt` and `apiserver.key`.

### etcd

The `etcd` server stores all cluster state and configuration data. It requires its own certificate and key pair: `etcd-server.crt` and `etcd-server.key`. Because etcd holds the entire cluster state, securing access to it is critical.

### kubelet

Each node runs a kubelet service that exposes an HTTPS API endpoint used by the `kube-apiserver` to interact with the node, schedule pods, and monitor node status. This requires a certificate and key pair: `kubelet.crt` and `kubelet.key`.

---

## Client Components

Client components connect to server components and must be authenticated using client certificates.

### Admin User

The administrator interacts with the `kube-apiserver` via `kubectl` or the REST API and requires a certificate and key pair to authenticate: `admin.crt` and `admin.key`.

### kube-scheduler

The `kube-scheduler` talks to the `kube-apiserver` to identify pods that require scheduling and to assign them to appropriate nodes. From the `kube-apiserver`'s perspective, the scheduler is a client and requires its own certificate pair: `scheduler.crt` and `scheduler.key`.

### kube-controller-manager

The `kube-controller-manager` runs the core control loops that regulate cluster state. It continuously accesses the `kube-apiserver` and requires a certificate pair: `controller-manager.crt` and `controller-manager.key`.

### kube-proxy

The `kube-proxy` runs on each node and maintains network rules for pod communication. It authenticates to the `kube-apiserver` and requires its own certificate pair: `kube-proxy.crt` and `kube-proxy.key`.

---

## Server-to-Server Communication

The `kube-apiserver` is the only component that communicates directly with `etcd`. From etcd's perspective, the `kube-apiserver` is a client and must authenticate. It can reuse its existing `apiserver.crt` and `apiserver.key`, or a dedicated pair can be generated specifically for authenticating to `etcd`.

The `kube-apiserver` also communicates with the `kubelet` on each node to schedule workloads and retrieve node and pod status. The same choice applies: reuse the existing certificate pair or generate a dedicated one.

---

## Grouping the Certificates

Certificates in the cluster fall into two groups:

- **Client certificates**: used by clients connecting to the `kube-apiserver`. This includes the admin user, `kube-scheduler`, `kube-controller-manager`, `kube-proxy`, and the `kube-apiserver` itself when authenticating to `etcd` or `kubelet`.
- **Server certificates**: used by `kube-apiserver`, `etcd`, and `kubelet` to secure their endpoints and authenticate to connecting clients.

---

## Certificate Authority

All certificates in the cluster must be signed by a Certificate Authority. Kubernetes requires at least one CA for the cluster. A single CA can sign all component certificates. Alternatively, a separate CA can be used specifically for `etcd`, in which case the `etcd` server certificate and all client certificates used to authenticate to `etcd` are signed by that dedicated CA rather than the cluster CA.

The CA has its own certificate and key pair: `ca.crt` and `ca.key`.

---

## Certificate Overview

| Component               | Certificate              | Key                      | Type   |
|-------------------------|--------------------------|--------------------------|--------|
| CA                      | `ca.crt`                 | `ca.key`                 | Root   |
| kube-apiserver          | `apiserver.crt`          | `apiserver.key`          | Server |
| etcd                    | `etcd-server.crt`        | `etcd-server.key`        | Server |
| kubelet                 | `kubelet.crt`            | `kubelet.key`            | Server |
| Admin user              | `admin.crt`              | `admin.key`              | Client |
| kube-scheduler          | `scheduler.crt`          | `scheduler.key`          | Client |
| kube-controller-manager | `controller-manager.crt` | `controller-manager.key` | Client |
| kube-proxy              | `kube-proxy.crt`         | `kube-proxy.key`         | Client |