---
title: API Groups
date: 2026-05-13
weight: 7
tags: [Security, API Groups]
---
# Kubernetes API Groups

Every operation performed against a cluster, whether through `kubectl` or directly via REST, passes through the `kube-apiserver`. The API is organized into groups based on purpose, each accessible via a distinct URL path.

---

## How the API Is Organized

| Path       | Purpose                                      |
|------------|----------------------------------------------|
| `/version` | Returns the cluster version                  |
| `/api`     | Core Kubernetes resources                    |
| `/apis`    | Named API groups                             |
| `/healthz` | Cluster health checks                        |
| `/metrics` | Cluster metrics                              |
| `/logs`    | Integration with third-party logging systems |

The two paths relevant to cluster functionality are `/api` and `/apis`, divided into the **core group** and the **named groups**.

---

## The Core Group (`/api`)

The core group contains the foundational Kubernetes resources, accessible at `/api/v1`. In documentation this is sometimes written as `v1` or `core`.

Resources in the core group:

- Namespaces
- Pods
- ReplicationControllers
- Events
- Endpoints
- Nodes
- Bindings
- PersistentVolumes and PersistentVolumeClaims
- ConfigMaps and Secrets
- Services

These resources pre-date the named group structure and use the version path directly without a named group prefix.

---

## The Named Groups (`/apis`)

All newer Kubernetes features are introduced under `/apis` rather than the core group. Each named group covers a distinct domain:

| Group                   | Example Resources                            |
|-------------------------|----------------------------------------------|
| `apps`                  | Deployments, ReplicaSets, StatefulSets       |
| `extensions`            | Deprecated features migrated to other groups |
| `networking.k8s.io`     | NetworkPolicies, Ingresses                   |
| `storage.k8s.io`        | StorageClasses, VolumeAttachments            |
| `authentication.k8s.io` | TokenReviews                                 |
| `authorization.k8s.io`  | SubjectAccessReviews                         |
| `certificates.k8s.io`   | CertificateSigningRequests                   |

The named groups are the **API groups** and the objects within them are the **resources**.

---

## Resources and Verbs

Each resource within an API group exposes a set of allowed operations called **verbs**. For a Deployment, the available verbs are:

- `list`: retrieve all Deployments.
- `get`: retrieve a specific Deployment.
- `create`: create a new Deployment.
- `update`: modify an existing Deployment.
- `patch`: partially update a Deployment.
- `delete`: remove a Deployment.
- `watch`: stream changes to a Deployment in real time.

Verbs are central to RBAC. A `Role` or `ClusterRole` grants a subject permission to perform specific verbs on specific resources within a specific API group.

---

## Discovering API Groups

**List all top-level API groups**
```bash
curl https://<master-ip>:6443/ \
  --cacert ca.crt --cert admin.crt --key admin.key
```

**List all resources within the named groups**
```bash
curl https://<master-ip>:6443/apis \
  --cacert ca.crt --cert admin.crt --key admin.key
```

**List API resources with group information**
```bash
kubectl api-resources -o wide
```

**Filter by group**
```bash
kubectl api-resources -o wide | grep apps
```

> **Note**: Direct `curl` requests to the API server require authentication. Certificate files must be passed as shown above. Endpoints such as `/version` are the exception and allow unauthenticated access.

---

## Using `kubectl proxy` for Local Access

`kubectl proxy` launches a local proxy on port `8001` that reads credentials from the kubeconfig file and forwards authenticated requests to the API server:

```bash
kubectl proxy
```

With the proxy running, credentials are no longer required in each request:

```bash
curl http://localhost:8001/
curl http://localhost:8001/apis
curl http://localhost:8001/api/v1/pods
```

---

## `kube-proxy` vs. `kubectl proxy`

| Tool            | Purpose                                                                                                                                          |
|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| `kube-proxy`    | A network component that runs on every node. Manages iptables or IPVS rules to enable connectivity between Pods and Services across the cluster. |
| `kubectl proxy` | A local HTTP proxy created by `kubectl`. Forwards requests to the `kube-apiserver` using credentials from the kubeconfig file.                   |

---

## Conclusion

The Kubernetes API is divided into the core group (`/api/v1`), containing foundational resources such as Pods, Nodes, and Services, and the named groups (`/apis`), where all newer features are introduced. Each resource within a group exposes a set of verbs representing the operations that can be performed on it. This structure is the foundation for RBAC, where access control policies are defined in terms of verbs on resources within API groups.