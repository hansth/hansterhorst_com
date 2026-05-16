---
title: RBAC
date: 2026-05-13
weight: 8
tags: [Security, RBAC]
---
# Role-Based Access Control

Role-Based Access Control (RBAC) is the main mechanism Kubernetes uses to control access to cluster resources. Administrators define policies that specify which actions a user, group, or Service Account can perform on specific resources, such as Pods or Services.

---

## Users, Groups, and Service Accounts

Kubernetes does not manage user or group accounts directly. There is no stored user or group entity in Kubernetes. Instead, Kubernetes expects certificates issued by the cluster's Certificate Authority, with subject identifiers that map to users and groups.

To create a user named `dev-user` in a group called `dev-group`, a certificate is produced with the subject `CN=dev-user, O=dev-group`. Ownership of that certificate and its matching private key prove the user's identity. Kubernetes recognizes it as a valid user in a valid group without requiring any account creation.

### Certificate Signing Flow

1. Generate a private key for the user.
2. Create a Certificate Signing Request (CSR) with the subject set to `CN=<username>, O=<group>`.
3. Submit the CSR to Kubernetes by creating a `CertificateSigningRequest` resource.
4. An administrator approves the CSR.
5. Retrieve the signed certificate issued by the Kubernetes CA.
6. Configure the `kubeconfig` file with the signed certificate and private key as user credentials.

### Subjects in RBAC

Running `kubectl get clusterrolebindings -o wide` shows three types of subjects:

- **Users**: individuals or applications interacting with the cluster. Not managed by Kubernetes; represented by strings such as `dev-user` or `dev-user@example.com`.
- **Groups**: represent multiple users. Permissions granted to a group are inherited by all users in that group.
- **Service accounts**: used by applications running inside the cluster. Unlike users and groups, service accounts are Kubernetes objects managed natively, tied to a specific namespace.

---

## Roles

A Role defines a set of permissions within a Namespace. The `apiVersion` is `rbac.authorization.k8s.io/v1`, and the `kind` is `Role`.

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

- `apiGroups`: for core resources such as pods and ConfigMaps, use an empty string `""`. For other API groups, specify the group name explicitly.
- `resources`: the Kubernetes resource types to which the rule applies.
- `verbs`: the actions permitted on those resources.

A single Role can contain multiple rules.

**Apply manifest**
```bash
kubectl create -f developer-role.yaml
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
kubectl create -f dev-user-to-developer-binding.yaml
```

---

## Namespace Scope

Roles and RoleBindings are namespace-scoped resources. A RoleBinding grants access only within the namespace where it is created. Without a specified namespace, the binding applies to the `default` namespace.

To restrict access to a specific Namespace, add the `namespace` field under `metadata` in both the Role and RoleBinding:

```yaml
metadata:
  name: developer
  namespace: dev-namespace
```

---

## ClusterRoles and ClusterRoleBindings

Roles and RoleBindings are confined to a Namespace. For cluster-wide permissions, or permissions on non-namespaced resources such as nodes and persistent volumes, Kubernetes provides ClusterRoles and ClusterRoleBindings.

### ClusterRole

A ClusterRole is a cluster-wide resource that applies across all namespaces. The built-in `cluster-admin` ClusterRole grants unrestricted access to every resource and every verb:

```bash
kubectl describe clusterrole cluster-admin
```
```
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```

### ClusterRoleBinding

A ClusterRoleBinding connects a ClusterRole to its subjects. Without a ClusterRoleBinding, a ClusterRole is a definition of permissions with no one to exercise them.

**Example**: The `kubeadm:cluster-admins` group is bound to the `cluster-admin` ClusterRole:
```bash
kubectl get clusterrolebinding kubeadm:cluster-admins -o wide
```
```
NAME                     ROLE                        AGE   USERS   GROUPS
kubeadm:cluster-admins   ClusterRole/cluster-admin   28d           kubeadm:cluster-admins
```

Any user whose certificate contains `O=kubeadm:cluster-admins` inherits `cluster-admin` permissions without being listed individually. This is how group-based inheritance works in RBAC.

---

## Inspecting Roles and RoleBindings

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

**List ClusterRoleBindings with subject details**
```bash
kubectl get clusterrolebindings -o wide
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