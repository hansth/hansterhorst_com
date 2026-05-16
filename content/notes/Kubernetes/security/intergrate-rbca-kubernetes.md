---
title: Implementing RBAC in Kubernetes
date: 2026-05-13
weight: 9
tags: [Security, RBAC, Tutorial]
---
# Implementing RBAC in Kubernetes

Role-Based Access Control (RBAC) is a fundamental security concept in Kubernetes that controls which authenticated users and processes are allowed to do in a Kubernetes cluster. This article covers practical implementation: creating users, assigning permissions through ClusterRoles and Roles, and verifying access controls.

---

## Creating a Super User Group

The following steps create a custom superuser configuration using a group called `superheroes-group`.

**Create a ClusterRole with unrestricted permissions**
```bash
kubectl create clusterrole superhero-clusterrole \
  --verb='*' --resource='*' \
  --dry-run=client -o yaml > superhero-clusterrole.yaml
```
- `--verb='*'`: All actions permitted.
- `--resource='*'`: Applies to all resources.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: superhero-clusterrole
rules:
- apiGroups:
  - ""
  resources:
  - '*'
  verbs:
  - '*'
```
This provides full access to perform all actions across all resources within the cluster.

**Create a ClusterRoleBinding for the `superheroes-group` group**
```bash
kubectl create clusterrolebinding superhero-clusterrolebinding \
  --clusterrole=superhero-clusterrole \
  --group=superheroes-group \
  --dry-run=client -o yaml > superhero-clusterrolebinding.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: superhero-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: superhero-clusterrole
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: superheroes-group
```

**Apply the manifests**
```bash
kubectl apply -f superhero-clusterrole.yaml
kubectl apply -f superhero-clusterrolebinding.yaml
```

**Verify the binding**
```bash
kubectl get clusterrolebindings | grep superhero-clusterrole
```

This shows the binding between the two resources.

```yaml
NAME                   ROLE         AGE
superhero-clusterrole  ClusterRole  1m
```

---

## Verifying Permissions with `kubectl auth can-i`

Kubernetes provides the `can-i` command to check user permissions.

**Check if the current admin user has full access**
```bash
kubectl auth can-i '*' '*'
# yes
```

**Verify that members of `superheroes-group` have full access**
```bash
kubectl auth can-i '*' '*' --as=batman --as-group=superheroes-group
kubectl auth can-i '*' '*' --as=superman --as-group=superheroes-group
kubectl auth can-i '*' '*' --as=wonder-woman --as-group=superheroes-group
```

All return `yes`. Individual user identity does not matter here. As long as a user belongs to `superheroes-group`, which is bound to `superhero-clusterrole` with wildcard permissions, full cluster access is inherited.

---

## Authenticating as a Specific User

To create a user named `batman` in a group called `superheroes-group`, the following steps produce a certificate with the subject `CN=batman, O=superheroes-group`:

1. **Generate a private key** for the user.
2. **Create a Certificate Signing Request (CSR)** using that private key, with the subject set to `CN=batman, O=superheroes-group`.
3. **Submit the CSR to Kubernetes** by creating a `CertificateSigningRequest` resource.
4. **An administrator approves the CSR** within the cluster.
5. **Retrieve the signed certificate** issued by the Kubernetes CA.
6. **Configure the `kubeconfig` file** with the signed certificate and private key as user credentials.

Use a manual process to authenticate as a specific user and execute commands under that user’s credentials.

### Step 1: Generate a Private Key

```bash
openssl genrsa -out batman.key 4096
```
This generates an RSA private key in 4096-bit format, which is highly secure.

### Step 2: Create a Certificate Signing Request

```bash
openssl req -new -key batman.key -out batman.csr \
  -subj "/CN=batman/O=superheroes-group" -sha256
```
- `openssl req`: The certificate request utility in OpenSSL.
- `-new`: Generate a new certificate request.
- `-key batman.key`: Use this existing private key.
- `-out batman.csr`: Output the CSR to this file.
- `-subj`: Set certificate subject fields:
    - `CN=batman`: Common Name (typically the username or service name).
    - `O=superheroes-group`: Organization field.
- `-sha256`: Use SHA-256 hashing algorithm.

In Kubernetes RBAC, the `CN` becomes the **username**, and the `O` becomes the **group**, which creates a user `batman` in the `superheroes-group` for authorization.

### Step 3: Submit the CSR to Kubernetes

These steps are for the cluster's Certificate Authority to sign the request.

**Encode the CSR as base64**
```bash
CSR_DATA=$(cat batman.csr | base64 | tr -d '\n')
```

From the [Kubernetes documentation](https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/#create-k8s-certificatessigningrequest) uses the template to generate a YAML file.

**Create the CertificateSigningRequest manifest**
```bash
cat <<EOF > batman-csr-request.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: batman
spec:
  request: ${CSR_DATA}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF
```

**Apply the manifest**
```bash
kubectl apply -f batman-csr-request.yaml
```

**Check the status**
```bash
kubectl get csr batman
```

The CSR shows `Pending` until an administrator approves it.

### Step 4: Approve the CSR

The administrator has to approve it, and it remains available for the next 24 hours.

**Approve the request**
```bash
kubectl certificate approve batman
```
```bash
kubectl get csr batman
```

The CSR now shows `Approved,Issued`.

### Step 5: Retrieve the Signed Certificate

To retrieve the signed certificate, you can examine the CSR resource with JSON output:
```bash
kubectl get csr batman -o json
```
```json
{
    "apiVersion": "certificates.k8s.io/v1",
    "kind": "CertificateSigningRequest",
    "metadata": {...},
    "status": {
        "certificate": "LS0...=",
        "conditions": [
            {
                "lastTransitionTime": "2026-02-12T09:08:37Z",
                "lastUpdateTime": "2026-02-12T09:08:37Z",
                "message": "This CSR was approved by kubectl certificate approve.",
                "reason": "KubectlApprove",
                "status": "True",
                "type": "Approved"
            }
        ]
    }
}
```

The certificate data appears under `status.certificate` as a **base64-encoded** string.

**Extract it using JSON path**
```bash
kubectl get csr batman -o jsonpath='{.status.certificate}' | base64 -d > batman.crt
```

**Verify the certificate**
```bash
openssl x509 -in batman.crt -text -noout
```

This confirms that `CN=batman` and `O=superheroes-group` are in the certificate's subject.

### Step 6: Configure a Kubeconfig File

A `kubeconfig` file is required to authenticate as `batman`. Copy the existing one as a base:

**Copy the existing kubeconfig as a base**
```bash
cp ~/.kube/config batman-clustersuperheroes.config
```

**Remove existing user, context, and current-context**
```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl config unset users.default
KUBECONFIG=batman-clustersuperheroes.config kubectl config delete-context default
KUBECONFIG=batman-clustersuperheroes.config kubectl config unset current-context
```

**View the new `kubeconfig`**
```bash
cat batman-clustersuperheroes.config
```
The base file now contains only the cluster certificate authority details and API server information.

**Add the `batman` credentials**
```bash
KUBECONFIG=batman-clustersuperheroes.config \
  kubectl config set-credentials batman \
    --client-certificate=batman.crt \
    --client-key=batman.key \
    --embed-certs=true
```

**Create and activate the context**
```bash
KUBECONFIG=batman-clustersuperheroes.config \
  kubectl config set-context default \
    --cluster=kubernetes \
    --user=batman

KUBECONFIG=batman-clustersuperheroes.config \
  kubectl config use-context default
```

**Inspecting the new `kubeconfig`**
```bash
cat batman-clustersuperheroes.config
```
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0...
    server: https://192.xxx.xx.110:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: batman
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: batman
  user:
    client-certificate-data: LS0...=
    client-key-data: LS0...=
```

It shows the certificate and key data within the CA-signed certificate, all of which are associated with the `batman` user.

### Step 7: Testing

With `batman` configured as a cluster superuser, you can test the `kubeconfig` file:

```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl get nodes
KUBECONFIG=batman-clustersuperheroes.config kubectl run test-nginx --image=nginx
KUBECONFIG=batman-clustersuperheroes.config kubectl delete pod test-nginx
```

All commands succeed. `batman` has the same credentials access as the default cluster admin.

---

## Creating a Read-Only User

The following steps create a `watcher-clusterrole` with **view-only** access and a user `uatu` in the `watchers-group`.

**Create the ClusterRole**
```bash
kubectl create clusterrole watcher-clusterrole \
  --verb=list,get,watch \
  --resource='*' \
  --dry-run=client -o yaml > watcher-clusterrole.yaml
```
- `--verb`: This ClusterRole can only view resources
- `--resource`: Can view any resource

**Create the ClusterRoleBinding for `watchers-group`**
```bash
kubectl create clusterrolebinding watcher-clusterrolebinding \
  --clusterrole=watcher-clusterrole \
  --group=watchers-group \
  --dry-run=client -o yaml > watcher-clusterrolebinding.yaml
```

**Apply both manifests**
```bash
kubectl apply -f watcher-clusterrole.yaml
kubectl apply -f watcher-clusterrolebinding.yaml
```

**Verify permissions**
```bash
kubectl auth can-i '*' '*' --as=uatu --as-group=watchers-group
# no

kubectl auth can-i 'list' '*' --as=uatu --as-group=watchers-group
# yes
```
- `--as`: Assign to one object (user).
- `--as-group`: Assign a group.

### Certificate and Kubeconfig for uatu

Follow the same steps as for `batman`, with the `uatu` specific values.

**Generate a private key**
```bash
openssl genrsa -out uatu.key 4096
```

**Create a CSR**
```bash
openssl req -new -key uatu.key -out uatu.csr \
  -subj "/CN=uatu/O=watchers-group" -sha256
```

**Encode and submit the CSR**
```bash
CSR_DATA=$(cat uatu.csr | base64 | tr -d '\n')

cat <<EOF > uatu-csr-request.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: uatu
spec:
  request: ${CSR_DATA}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF

kubectl apply -f uatu-csr-request.yaml
```

**Approve and retrieve the certificate**
```bash
kubectl certificate approve uatu
kubectl get csr uatu -o jsonpath='{.status.certificate}' | base64 -d > uatu.crt
```

This writes the decoded, PEM-formatted certificate to `uatu.crt`.

**Configure the kubeconfig**
```bash
cp ~/.kube/config uatu-clusterwatchers.config
```

**Remove existing user**
```bash
KUBECONFIG=uatu-clusterwatchers.config kubectl config unset users.default
KUBECONFIG=uatu-clusterwatchers.config kubectl config delete-context default
KUBECONFIG=uatu-clusterwatchers.config kubectl config unset current-context
```

**Add the user's credentials**
```bash
KUBECONFIG=uatu-clusterwatchers.config \
  kubectl config set-credentials uatu \
    --client-certificate=uatu.crt \
    --client-key=uatu.key \
    --embed-certs=true
```

**Create and activate a context**
```bash
KUBECONFIG=uatu-clusterwatchers.config \
  kubectl config set-context default --cluster=kubernetes --user=uatu
```

**Set it as the active context**
```bash
KUBECONFIG=uatu-clusterwatchers.config kubectl config use-context default
```

**Test the read-only configuration**
```bash
KUBECONFIG=uatu-clusterwatchers.config kubectl get nodes
# success

KUBECONFIG=uatu-clusterwatchers.config kubectl run test-nginx --image=nginx
# Error from server (Forbidden): pods is forbidden: User "uatu" cannot create resource "pods"

KUBECONFIG=uatu-clusterwatchers.config kubectl get all -A
# success
```

---

## Creating User-Specific Permissions

The following steps create a `podmanager-clusterrole` bound directly to the user `deadpool`, without a group.

Repeat the same steps as in the previous sections:

**Create the ClusterRole**
```bash
kubectl create clusterrole podmanager-clusterrole \
  --verb=list,get,create,delete \
  --resource=pods \
  --dry-run=client -o yaml > podmanager-clusterrole.yaml
```

**Create the ClusterRoleBinding bound to the user `deadpool`**
```bash
kubectl create clusterrolebinding podmanager-clusterrolebinding \
  --clusterrole=podmanager-clusterrole \
  --user=deadpool \
  --dry-run=client -o yaml > podmanager-clusterrolebinding.yaml
```

**Apply the manifests**
```bash
kubectl apply -f podmanager-clusterrole.yaml
kubectl apply -f podmanager-clusterrolebinding.yaml
```

**Verify permissions**
```bash
kubectl auth can-i list '*' --as=deadpool
# no

kubectl auth can-i list pods --as=deadpool
# yes
```
- `list`: Reference `--verb` in the ClusterRole
- `pods`: Reference `--resource` in the ClusterRole

**Inspect the ClusterRoleBinding**
```bash
kubectl get clusterrolebindings podmanager-clusterrolebinding -o yaml
```
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {...}
  creationTimestamp: "2026-02-12T13:27:13Z"
  name: podmanager-clusterrolebinding
  resourceVersion: "6560766"
  uid: 103bf858-89d2-448b-a66a-141758189e80
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: podmanager-clusterrole
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: deadpool
```

The `subjects` section references `deadpool` as a `User`, not a group member.

### Certificate and Kubeconfig for deadpool

Follow the same steps as for `batman`, substituting the `deadpool`-specific values.

**Generate a private key**
```bash
openssl genrsa -out deadpool.key 4096
```

**Create a CSR (no group, so only CN is specified)**
```bash
openssl req -new -key deadpool.key -out deadpool.csr \
  -subj "/CN=deadpool" -sha256
```
In this command, only the Common Name (CN) is specified for the user; the Organization (`O`) is not defined because there is no group.

**Encode and submit the CSR**
```bash
CSR_DATA=$(cat deadpool.csr | base64 | tr -d '\n')

cat <<EOF > deadpool-csr-request.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: deadpool
spec:
  request: ${CSR_DATA}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF

kubectl apply -f deadpool-csr-request.yaml
```

**Approve and retrieve the certificate**
```bash
kubectl certificate approve deadpool
kubectl get csr deadpool -o jsonpath='{.status.certificate}' | base64 -d > deadpool.crt
```

**Configure the kubeconfig**
```bash
cp ~/.kube/config deadpool-podmanager.config
```

**Remove existing user**
```bash
KUBECONFIG=deadpool-podmanager.config kubectl config unset users.default
KUBECONFIG=deadpool-podmanager.config kubectl config delete-context default
KUBECONFIG=deadpool-podmanager.config kubectl config unset current-context
```

**Add the user's credentials**
```bash
KUBECONFIG=deadpool-podmanager.config \
  kubectl config set-credentials deadpool \
    --client-certificate=deadpool.crt \
    --client-key=deadpool.key \
    --embed-certs=true
```

**Set it as the active context**
```bash
KUBECONFIG=deadpool-podmanager.config \
  kubectl config set-context default --cluster=kubernetes --user=deadpool

KUBECONFIG=deadpool-podmanager.config kubectl config use-context default
```

**Test the pod manager configuration**
```bash
KUBECONFIG=deadpool-podmanager.config kubectl run test-nginx --image=nginx
# pod/test-nginx created

KUBECONFIG=deadpool-podmanager.config kubectl get pods
# NAME         READY   STATUS    RESTARTS   AGE
# test-nginx   1/1     Running   0          4s

KUBECONFIG=deadpool-podmanager.config kubectl get nodes
# Error from server (Forbidden): nodes is forbidden: User "deadpool" cannot list resource "nodes"
```

---

## Namespace-Scoped RBAC: Roles and RoleBindings

ClusterRoles and ClusterRoleBindings operate at the cluster level. Roles and RoleBindings are namespace-scoped equivalents.

| Scope        | Permissions Resource | Binding Resource   |
|--------------|----------------------|--------------------|
| Cluster-wide | ClusterRole          | ClusterRoleBinding |
| Namespace    | Role                 | RoleBinding        |

The following steps create a namespace `gryffindor` with a Role that grants pod management permissions to the `gryffindor-students` group, and a user `harry` in that group.

**Create the namespace**
```bash
kubectl create namespace gryffindor \
  --dry-run=client -o yaml > gryffindor-ns.yaml

kubectl apply -f gryffindor-ns.yaml
```

**Create the Role**
```bash
kubectl create role gryffindor-role \
  --verb=list,get,create,delete \
  --resource=pods \
  --namespace=gryffindor \
  --dry-run=client -o yaml > gryffindor-role.yaml
```

**Create the RoleBinding for `gryffindor-students`**
```bash
kubectl create rolebinding gryffindor-rolebinding \
  --role=gryffindor-role \
  --group=gryffindor-students \
  --namespace=gryffindor \
  --dry-run=client -o yaml > gryffindor-rolebinding.yaml
```

**Apply the manifests**
```bash
kubectl apply -f gryffindor-role.yaml
kubectl apply -f gryffindor-rolebinding.yaml
```

**Verify namespace-scoped permissions**
```bash
kubectl auth can-i list pods --as=harry --as-group=gryffindor-students
# no (cluster-level check fails)

kubectl auth can-i list pods --as=harry --as-group=gryffindor-students -n gryffindor
# yes
```

**Check the RoleBinding in the Namespace**
```bash
kubectl -n gryffindor \
  get rolebindings gryffindor-rolebinding \
  -o yaml
```
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {...}
  creationTimestamp: "2026-02-12T14:15:21Z"
  name: gryffindor-rolebinding
  namespace: gryffindor
  resourceVersion: "6567987"
  uid: f55e44a2-8f1e-4ae7-b272-7e621829d0ce
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: gryffindor-role
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: gryffindor-students
```

This shows the `gryffindor-role` Role and `gryffindor-students` group.

### Certificate and Kubeconfig for harry

Follow the same steps as for `batman`, substituting the `harry`-specific values.

**Generate a private key**
```bash
openssl genrsa -out harry.key 4096
```

**Create a CSR**
```bash
openssl req -new -key harry.key -out harry.csr \
  -subj "/CN=harry/O=gryffindor-students" -sha256
```

**Encode and submit the CSR**
```bash
CSR_DATA=$(cat harry.csr | base64 | tr -d '\n')

cat <<EOF > harry-csr-request.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: harry
spec:
  request: ${CSR_DATA}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF
```

**Apply the manifest**
```bash
kubectl apply -f harry-csr-request.yaml
```

**Approve and retrieve the certificate**
```bash
kubectl certificate approve harry
kubectl get csr harry -o jsonpath='{.status.certificate}' | base64 -d > harry.crt
```

This writes the decoded, PEM-formatted certificate to `harry.crt`.

**Confirm the certificate details with OpenSSL**
```bash
openssl x509 -in harry.crt -text -noout
```

**Configure the kubeconfig**
```bash
cp ~/.kube/config harry-ns-gryffindor.config
```

**Remove existing user**
```bash
KUBECONFIG=harry-ns-gryffindor.config kubectl config unset users.default
KUBECONFIG=harry-ns-gryffindor.config kubectl config delete-context default
KUBECONFIG=harry-ns-gryffindor.config kubectl config unset current-context
```

**Add the user's credentials**
```bash
KUBECONFIG=harry-ns-gryffindor.config \
  kubectl config set-credentials harry \
    --client-certificate=harry.crt \
    --client-key=harry.key \
    --embed-certs=true
```

**Set it as the active context**
```bash
KUBECONFIG=harry-ns-gryffindor.config \
  kubectl config set-context default --cluster=kubernetes --user=harry

KUBECONFIG=harry-ns-gryffindor.config kubectl config use-context default
```

**Review the finished `kubeconfig`**
```bash
cat harry-ns-gryffindor.config
```
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0...
    server: https://192.xxx.xx.110:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: harry
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: harry
  user:
    client-certificate-data: LS0...
    client-key-data: LS0...
```

**Test the namespace-scoped configuration**
```bash
KUBECONFIG=harry-ns-gryffindor.config \
  kubectl run test-nginx --image=nginx --namespace=gryffindor
# pod/test-nginx created

KUBECONFIG=harry-ns-gryffindor.config \
  kubectl get pods --namespace=gryffindor
# NAME         READY   STATUS    RESTARTS   AGE
# test-nginx   1/1     Running   0          40s

KUBECONFIG=harry-ns-gryffindor.config kubectl get nodes
# Error from server (Forbidden): nodes is forbidden: User "harry" cannot list resource "nodes"
```

---

## The Four RBAC API Objects

RBAC uses two pairs of resources, one pair for **namespace-scoped** access, one for **cluster-wide**:

| Object               | Scope        | Purpose                                                                   |
|----------------------|--------------|---------------------------------------------------------------------------|
| `Role`               | Namespace    | Defines permissions within a single namespace                             |
| `ClusterRole`        | Cluster-wide | Defines permissions across all namespaces or for cluster-scoped resources |
| `RoleBinding`        | Namespace    | Grants a Role or ClusterRole to subjects within a namespace               |
| `ClusterRoleBinding` | Cluster-wide | Grants a ClusterRole to subjects across the entire cluster                |

---

## Quick Reference: Commands Summary

| Step                               | Command                                                                                                                                                 |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| Create ClusterRole manifest        | `kubectl create clusterrole superhero-clusterrole --verb='*' --resource='*' --dry-run=client -o yaml`                                                   |
| Create ClusterRoleBinding manifest | `kubectl create clusterrolebinding superhero-clusterrolebinding --clusterrole=superhero-clusterrole --group=superheroes-group --dry-run=client -o yaml` |
| Check permissions (impersonation)  | `kubectl auth can-i '*' '*' --as=batman --as-group=superheroes-group`                                                                                   |
| Generate private key               | `openssl genrsa -out batman.key 4096`                                                                                                                   |
| Create CSR                         | `openssl req -new -key batman.key -out batman.csr -subj "/CN=batman/O=superheroes-group" -sha256`                                                       |
| Encode CSR to base64               | `CSR_DATA=$(cat batman.csr \| base64 \| tr -d '\n')`                                                                                                    |
| Apply CSR manifest                 | `kubectl apply -f batman-csr-request.yaml`                                                                                                              |
| Check CSR status                   | `kubectl get csr batman`                                                                                                                                |
| Approve CSR                        | `kubectl certificate approve batman`                                                                                                                    |
| Extract signed certificate         | `kubectl get csr batman -o jsonpath='{.status.certificate}' \| base64 -d > batman.crt`                                                                  |
| Verify certificate                 | `openssl x509 -in batman.crt -text -noout`                                                                                                              |
| Set credentials in kubeconfig      | `kubectl config set-credentials batman --client-certificate=batman.crt --client-key=batman.key --embed-certs=true`                                      |
| Test as user                       | `KUBECONFIG=batman-clustersuperheroes.config kubectl get nodes`                                                                                         |

---

## Conclusion

This article covered practical RBAC implementation in Kubernetes:

- Creating ClusterRoles and Roles.
- Binding them to users and groups through ClusterRoleBindings and RoleBindings.
- Generating user certificates.
- Configuring kubeconfig files.
- Verifying access controls at both cluster and namespace scope.

**Cleanup**
```bash
kubectl delete clusterrole \
  watcher-clusterrole \
  superhero-clusterrole \
  podmanager-clusterrole

kubectl delete clusterrolebindings \
  watcher-clusterrolebinding \
  podmanager-clusterrolebinding \
  superhero-clusterrolebinding

kubectl -n gryffindor delete role gryffindor-role
kubectl -n gryffindor delete rolebindings gryffindor-rolebinding
kubectl delete namespace gryffindor
kubectl delete csr batman uatu deadpool harry
```

---
