---
title: Role-Based and Cluster-Based RBAC
date: 2026-05-13
weight: 10
tags: [Security, RBAC]
---
# Role-Based and Cluster-Based RBAC

Role-Based Access Control (RBAC) is the primary mechanism Kubernetes uses to regulate access to cluster resources. Administrators define policies that specify which actions a user, group, or Service Account can perform on specific resources.

RBAC uses four core API objects organized into two pairs based on scope:

- Roles and RoleBindings operate within a single namespace
- ClusterRoles and ClusterRoleBindings apply across the entire cluster or to resources that exist outside any namespace.

| Object               | Scope        | Purpose                                                                   |
|----------------------|--------------|---------------------------------------------------------------------------|
| `Role`               | Namespace    | Defines permissions within a single namespace                             |
| `RoleBinding`        | Namespace    | Binds a Role to a user, group, or service account                         |
| `ClusterRole`        | Cluster-wide | Defines permissions across all namespaces or for cluster-scoped resources |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to a user, group, or service account                  |

---

## Namespaced vs. Cluster-Scoped Resources

All Kubernetes resources fall into one of two categories.

**Namespaced resources** are created in a specified namespace and managed within it:

- Pods, ReplicaSets, Jobs, Deployments
- Services, Secrets, ConfigMaps
- Roles, RoleBindings

**Cluster-scoped resources** exist outside any namespace and apply across the entire cluster:

- Nodes
- PersistentVolumes
- ClusterRoles, ClusterRoleBindings
- CertificateSigningRequests
- Namespaces themselves

To list all namespaced and cluster-scoped resources:

**Namespaced resources**
```bash
kubectl api-resources --namespaced=true
```

**Cluster-scoped resources**
```bash
kubectl api-resources --namespaced=false
```

---

## Roles

A Role defines a set of permissions within a namespace. The `apiVersion` is `rbac.authorization.k8s.io/v1` and the `kind` is `Role`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: dev-namespace
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "get", "create", "delete"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create"]
```

Each entry under `rules` has three fields:

- `apiGroups`: for core resources such as Pods and ConfigMaps, use an empty string `""`. For other API groups, specify the group name explicitly.
- `resources`: the Kubernetes resource types the rule applies to.
- `verbs`: the actions permitted on those resources.

A single Role can contain multiple rules.

**Apply manifest**
```bash
kubectl apply -f developer-role.yaml
```

**Create it imperatively**
```bash
kubectl create role developer \
  --verb=list,get,create,delete \
  --resource=pods \
  --namespace=dev-namespace
```

---

## RoleBindings

A RoleBinding links a subject (user, group, or Service Account) to a Role.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-to-developer-binding
  namespace: dev-namespace
subjects:
  - kind: User
    name: dev-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```
- `subjects`: defines who the binding applies to.
- `roleRef`: references the Role being granted.

**Apply manifest**
```bash
kubectl apply -f dev-user-to-developer-binding.yaml
```

**Create it imperatively**
```bash
kubectl create rolebinding dev-user-to-developer-binding \
  --role=developer \
  --user=dev-user \
  --namespace=dev-namespace
```

---

## Namespace Scope

Roles and RoleBindings are namespace-scoped. A RoleBinding grants access only within the namespace where it is created. Without a specified namespace, the binding applies to the `default` namespace.

To restrict access to a specific namespace, add the `namespace` field under `metadata` in both the Role and RoleBinding.

---

## ClusterRoles

Roles and RoleBindings handle authorization for namespaced resources. To authorize users against cluster-scoped resources such as nodes or persistent volumes, Kubernetes provides ClusterRoles and ClusterRoleBindings.

A ClusterRole is a cluster-wide resource that applies across all namespaces. The rules follow the same structure as a regular Role.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list", "get", "create", "delete"]
```

**Apply manifest**
```bash
kubectl apply -f cluster-admin-role.yaml
```

**Create it imperatively**
```bash
kubectl create clusterrole cluster-administrator \
  --verb=list,get,create,delete \
  --resource=nodes
```

### ClusterRoles for Namespaced Resources

A ClusterRole can also target namespaced resources. The difference from a regular Role is scope:

- A **Role** grants access to resources in a single namespace.
- A **ClusterRole** grants access to that resource type across all namespaces in the cluster.

### Default ClusterRoles

When a Kubernetes cluster is first initialized, it automatically creates a set of default ClusterRoles covering common administrative use cases:

```bash
kubectl get clusterroles
```

The built-in `cluster-admin` ClusterRole grants unrestricted access to every resource and every verb across the entire cluster.

---

## ClusterRoleBindings

A ClusterRoleBinding connects a ClusterRole to its subjects.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
  - kind: User
    name: cluster-admin-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```
- `subjects`: defines the user being authorized.
- `roleRef`: references the ClusterRole being granted.

**Apply manifest**
```bash
kubectl apply -f cluster-admin-role-binding.yaml
```

**Create it imperatively**
```bash
kubectl create clusterrolebinding cluster-admin-role-binding \
  --clusterrole=cluster-administrator \
  --user=cluster-admin-user
```

---

## Inspecting Roles and Bindings

**List all Roles in the current namespace**
```bash
kubectl get roles
```

**List all RoleBindings**
```bash
kubectl get rolebindings
```

**Inspect a specific Role**
```bash
kubectl describe role developer
```

**Inspect a specific RoleBinding**
```bash
kubectl describe rolebinding dev-user-to-developer-binding
```

## Inspecting ClusterRoles and ClusterRoleBindings

**List ClusterRoles**
```bash
kubectl get clusterroles
```

**List ClusterRoleBindings**
```bash
kubectl get clusterrolebindings
```

---

## Checking Permissions

**Check if the current user can perform an action**
```bash
kubectl auth can-i create deployments
kubectl auth can-i delete nodes
```

**Impersonate another user to verify their permissions**
```bash
kubectl auth can-i create deployments --as=dev-user
kubectl auth can-i create pods --as=dev-user
```

**Scope the check to a specific namespace**
```bash
kubectl auth can-i create pods --as=dev-user --namespace=dev-namespace
```

---

## Restricting Access to Specific Resource Instances

By default, a Role grants access to all instances of a resource type. Access can be restricted to specific named resources using the `resourceNames` field:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
    resourceNames: ["blue", "orange"]
```

Access to any pod not listed in `resourceNames` is denied, even if other pods exist in the namespace.

---

## Reference

| Concept              | Description                                                          |
|----------------------|----------------------------------------------------------------------|
| `Role`               | Defines a set of permissions within a namespace                      |
| `RoleBinding`        | Binds a Role to a user, group, or service account                    |
| `ClusterRole`        | Defines cluster-wide permissions, including non-namespaced resources |
| `ClusterRoleBinding` | Binds a ClusterRole to a user, group, or service account             |
| `apiGroups`          | Core group uses `""`, other groups specify the name                  |
| `verbs`              | Actions such as `get`, `list`, `create`, `delete`                    |
| `resourceNames`      | Restricts access to specific named resource instances                |
| `kubectl auth can-i` | Verifies permissions, supports `--as` for impersonation              |