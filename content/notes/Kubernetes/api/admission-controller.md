---
title: Admission Controllers
date: 2026-05-17
weigh: 1
tags: [Controllers]
---
# Admission Controllers

An Admission Controller is a component that intercepts requests to the Kubernetes API server after authentication and authorization, but before the resource is stored in `etcd`.

---

## The Request Flow

Every `kubectl` request passes through the `kube-apiserver`, where it is authenticated using certificates, authorized via RBAC, and then processed by Admission Controllers before the object is created and stored in `etcd`.

```
           ┌─────────────────────────────────────────────────────┐
           │ kube-apiserver                                      │
┌───────┐  │  ┌──────────────┐   ┌─────────────┐   ┌───────────┐ │  ┌───┐
│kubectl│──┼─▶│Authentication│──▶│Authorization│──▶│ Admission │─┼─▶│Pod│
└───────┘  │  │ certificates │   │    RBAC     │   │Controllers│ │  └───┘
           │  └──────────────┘   └─────────────┘   └───────────┘ │
           └─────────────────────────────────────────────────────┘
```

---

## The Limits of RBAC

RBAC determines which API operations a user is allowed to perform. However, there are scenarios where fine-grained control over incoming requests is required:

- Rejecting Pod creation requests that reference images from public registries, allowing only images from a specific internal registry.
- Enforce that the `latest` image tag is never used.
- Rejecting requests to run containers as the root user.
- Permitting only certain Linux capabilities.
- Ensuring that pod metadata always includes specific labels.

None of these can be configured with RBAC alone. Admission Controllers solve this gap.

---

## What Admission Controllers Do

Admission Controllers enforce security measures and control how a cluster is used. Beyond validating incoming requests, an Admission Controller can also mutate the request itself or perform additional backend operations before the object is created.

There are two types:

- **Validating**: inspect the request and accept or reject it.
- **Mutating**: modify the request before it is processed.

Some controllers do both.

---

## Built-in Admission Controllers

Kubernetes ships with a number of pre-built Admission Controllers:

- `AlwaysPullImages`: Ensures container images are always pulled fresh when a pod is created.
- `DefaultStorageClass`: Automatically adds the default storage class to PersistentVolumeClaims that do not specify one.
- `EventRateLimit`: Limits the number of requests the API server can handle at a time, protecting it from being flooded.
- `NamespaceLifecycle`: Rejects requests targeting non-existent namespaces and protects the `default`, `kube-system`, and `kube-public` namespaces from deletion.
- `NodeRestriction`: Limits what a kubelet can modify on its own node object.

---

## Managing Admission Controllers

### Viewing Enabled Controllers

To see which admission controllers are currently enabled, use the `kube-apiserver` command:

```bash
kube-apiserver -h | grep enable-admission-plugins
```

On a `kubeadm`-based cluster, run the command inside the `kube-apiserver` pod:

```bash
kubectl -n kube-system exec pod/kube-apiserver-k8s-master-1 -- \
  kube-apiserver -h | grep enable-admission-plugins
```

### Enabling a Controller

On a non-kubeadm setup, update the `kube-apiserver` service:

```bash
ExecStart=/usr/local/bin/kube-apiserver \
  --enable-admission-plugins=NodeRestriction,NamespaceLifecycle
```

On a kubeadm-based setup, update the flag in the `kube-apiserver` manifest:

```yaml
- --enable-admission-plugins=NodeRestriction,NamespaceLifecycle
```

The manifest is typically located at `/etc/kubernetes/manifests/kube-apiserver.yaml`.

### Disabling a Controller

Disable Admission Controller plugins, update the flag in the `kube-apiserver` manifest:

```yaml
- --disable-admission-plugins=NamespaceLifecycle
```

---

## Example: NamespaceAutoProvision

When creating a Pod in a Namespace that does not exist, the `kube-apiserver` throws an error that the Namespace does not exist.

Internally, the request is authenticated, then authorized, and then passes through the Admission Controllers.

The `NamespaceExists` Admission Controller checks whether the Namespace exists, and by default, creating a Pod in a namespace that does not exist produces an error:

```bash
kubectl run nginx --image=nginx --namespace=blue
# Error from server (NotFound): namespaces "blue" not found
```

With `NamespaceAutoProvision` enabled, the Admission Controller detects the missing Namespace, creates it automatically, and completes the Pod creation request.

> **Note**: `NamespaceAutoProvision` and `NamespaceExists` are deprecated. `NamespaceLifecycle` is their replacement and is enabled by default.

`NamespaceLifecycle` is enabled by default. Creating a Pod in a Namespace that does not exist produces an error.

The Controller also protects the `default`, `kube-system`, and `kube-public` namespaces from deletion:

```bash
kubectl delete namespace kube-system
# Error from server (Forbidden): namespaces "kube-system" is forbidden
```

---
